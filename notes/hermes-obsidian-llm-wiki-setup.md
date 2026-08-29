# Hermes + Obsidian：把知识库做成 AI 的第二大脑
> [← 返回笔记地图](../README.md#笔记地图) · 故障排查与实战


> 源码位置：vault 目录 `D:\llmwiki\llm-wiki` + skills 组合（llm-wiki / obsidian / personal-knowledge-mgmt）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

从零搭建 Hermes 与 Obsidian 联动的 llm-wiki 知识库（基于 Karpathy 的三层知识库理念）：Hermes 不只是"问答助手"，还是这座知识库的**作者兼图书管理员**——它主动写笔记、主动整理、主动建立链接，人类负责定规则。本文记录整个搭建方案：目录结构、技能组合、cron 自动化与环境变量注入。

## 为什么这么设计

- **Karpathy 三层架构：不可变原料 / AI 自由创作 / 人类定规则**。层 1 `raw/`（文章、论文、转录）写入后不编辑，是知识库的"原材料库"；层 2 `entities/`、`concepts/`、`comparisons/`、`queries/` 由 AI 拥有，自由创建、更新、交叉引用——实体页（人/公司/产品/模型）与概念页是知识图谱的节点；层 3 `SCHEMA.md` 是规则层，**人类编辑、AI 遵守**（命名规范、标签分类、frontmatter 模板）。核心原则一句话：**AI 既是作者也是图书管理员，主动写、主动整理，而非人类写好 AI 再去搜**——把"检索式知识管理"升级成"生成式知识管理"。

- **能力分层：架构规范、文件读写、综合编排三个技能各司其职**。`llm-wiki`（官方）提供三层架构的 SCHEMA 规范；`obsidian`（官方）负责 vault 的文件读写搜索——这两者是"基础设施"；`personal-knowledge-mgmt`（本地）是综合编排层：对话归档、cron 自动化、全库分析都在这层。**底层能力与编排逻辑分离**，换 vault 或换编排策略都不用动对方。本系列笔记本身就是这套体系的产物：源码解剖进 raw/，研读笔记沉淀成独立作品集。

- **cron 自动化让知识库自己整理自己**。两个定时任务形成闭环：`llm-wiki-daily-scan`（每天 08:00）扫描全库 → 找孤立笔记 → 建议链接 → 更新 index.md（目录页）；`llm-wiki-inbox-triage`（每天 18:00）扫描 Inbox/ → 分类 → 加 frontmatter → 归档并加 wikilink。**收集（Inbox）与整理（daily-scan）分开时段**，灵感随时丢进 Inbox，整理统一在固定时间批量做——像真实的图书管理员一样有工作节奏。

- **环境变量注入而非硬编码路径**。`OBSIDIAN_VAULT_PATH`、`WIKI_PATH` 通过 `~/.hermes/.env` 注入，vault 路径变动只改配置不改技能；搜索服务商选 Tavily（AI agent 专用搜索引擎），`TAVILY_API_KEY` 同样走环境变量。**配置与逻辑分离**，这套方案换台机器、换个 vault 路径都能复用。

- **Obsidian 是 Markdown 文件系统，AI 和人类在同一层协作**。vault 就是普通文件夹，笔记是纯 Markdown——AI 用 skill 读写，人类用 Obsidian 客户端浏览，两边没有格式壁垒，整个 vault 还能纳入 git 版本管理。**知识库的载体越普通，AI 的参与度才能越高**：如果选私有格式数据库，AI 就永远只能隔着 API 操作，做不到"既是作者也是图书管理员"。周边生态也顺带受益：VS Code 的 Comment Translate 插件能直接翻译 vault 里的注释，说明这套体系对普通工具链完全开放。

## 关键机制/流程

目录结构按"层"组织：`SCHEMA.md`（规则）→ `raw/`（层 1 不可变）→ `entities/` + `concepts/` + `comparisons/` + `queries/`（层 2 AI 拥有）→ 加上 `Inbox/`（待分类入口）、`Daily/`（日记）、`Projects/`、`Meetings/`、`Refs/`（灵感与剪藏）等辅助区。配套的 Hermes 管理命令：`hermes skills list` / `hermes cron list` / `hermes cron update <id> --schedule "..."` / `hermes prompt-size`（看上下文用量）/ `hermes insights`（会话统计）。

## 相关笔记

- llm-wiki（Karpathy 三层知识库架构）
- personal-knowledge-mgmt（Obsidian 集成工作流）
- hermes-source-study（本系列总览）
