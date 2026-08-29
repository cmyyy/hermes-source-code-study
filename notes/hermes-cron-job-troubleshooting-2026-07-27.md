# Cron 定时任务排查实录：调度器活在进程里，扫描器会误伤
> [← 返回笔记地图](../README.md#笔记地图) · 故障排查与实战


> 源码位置：hermes cron 调度器（附着在网关主进程内）；注入扫描器 `_CRON_SKILL_ASSEMBLED_PATTERNS`
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

2026-07-27，每日 8 点推送的 cron job 没有在预定时间触发。排查后发现三个独立问题叠加：进程没在运行、skill 引用歧义、注入扫描器误报拦截。这不是一篇源码解剖，而是一次真实的故障排查实录——从中能看到 Hermes cron 的架构约束和安全机制的实际行为。

## 为什么这么设计（三个问题的根因与启示）

- **Cron 调度器附着在 Hermes 主进程中——进程不在，调度器不走 tick**。当天日志第一条记录出现在 18:36（用户启动会话时），早 8:00 附近没有任何日志。用户启动 Hermes 后，调度器检测到遗漏的 job 立即补跑，内容照常投递到飞书、无损失，只是有时间延迟。修复：`hermes gateway install` 在 Windows 启动文件夹创建开机自启项（无需管理员权限；若要登录前即运行需 `--system`）。**启示：进程内调度器的存活依赖宿主进程，开机自启是 cron 可靠性的前提条件。**

- **Skill 引用必须用完整分类路径**。`obsidian` 这个名字同时匹配了多个来源：正确的 `note-taking/obsidian/SKILL.md`、同目录下 `note-taking/SKILL.md`（frontmatter 声明了 `name: obsidian` 造成重复）、以及内置的 obsidian skill。cron job 改用完整路径 `note-taking/obsidian` 后歧义消除。**启示：模糊名解析在多来源场景下必然出问题，引用要显式化。**

- **注入扫描器会误伤合法内容**。cron 调度器把 user prompt + skill 内容 + 系统上下文汇编成完整 prompt 后，会跑一遍注入扫描（防 prompt injection）。某 skill 里的 `Do NOT tell the user`（原文措辞）命中了 `deception_hide` 威胁模式——正则 `r'do\s+not\s+tell\s+the\s+user'` + `re.IGNORECASE`，整个 job 被拦截。改成 `Do NOT simply reply "it should appear shortly"` 后语义不变、不再触发模式。**启示：基于正则的注入扫描器存在误报面，安全拦截的代价是整任务失败——被扫描的内容（包括 skill 文本）措辞要避开敏感模式。**

- **`deliver=all` 在 Windows 上无法解析投递目标**，改为 `origin`（投递回当前飞书会话）解决。属于平台相关的配置解析问题。

## 关键机制/流程

故障链路总结：Hermes 未运行导致 cron 缺触发（环境配置，开机自启修复）→ skill 名称歧义导致 llm-wiki job 失败（配置错误，完整路径修复）→ 注入扫描器误报 `deception_hide` 拦截 job（措辞修正）。修复后的经验：cron 调度器依赖网关进程存活；skill 引用一律 `category/skill-name` 完整路径；cron 注入扫描器对合法系统指令存在误报，需要源码级别调整；进程不在时补跑的 job 内容无损失但有延迟。

## 相关笔记

- obsidian-llm-wiki-setup（搭建记录与 cron 配置）
- hermes-cron-troubleshooting（cron 故障诊断技能）
