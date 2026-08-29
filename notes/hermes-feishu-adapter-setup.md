# 把 Hermes 接上飞书：消息平台适配器的配置与设计
> [← 返回笔记地图](../README.md#笔记地图) · 故障排查与实战


> 源码位置：`plugins/platforms/feishu/`（飞书适配器）→ `hermes gateway run` 网关入口
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

Hermes 不是只活在终端里的 CLI——它通过一套**消息平台网关**（gateway）接入 20+ 聊天平台，用户在哪聊天，它就在哪出现。本文记录从零把 Hermes 接上飞书（Feishu/Lark）的完整配置流程，并借此讲清楚"平台适配器"这一层是怎么设计的：为什么加一个新平台不用动 Agent 核心。

## 为什么这么设计

- **平台适配器插件化，Agent 核心零侵入**。国内平台（钉钉、飞书、企业微信、微信、QQ、元宝）在 `plugins/platforms/` 下按插件组织，海外平台（Telegram、Discord、Slack 等）在 `gateway/platforms/` 下。Agent 的对话循环不关心消息来自哪个平台，只面对统一的"消息进出"接口——新平台 = 新适配器，核心逻辑一行不改。这也是飞书能成为功能最完整的适配器（语音/图片/文件/流式输出/表情反应全支持）而不用等官方改架构的原因。

- **配置全部走环境变量，凭证不进代码库**。App ID、App Secret 通过 `hermes config env-path` 指向的 `.env` 文件注入，`FEISHU_DOMAIN` 一个变量切换国内（feishu）与海外（lark）两套域名。**凭证与代码分离**是消息平台接入的第一原则——仓库可以公开，密钥永不落地。

- **安全与便利用两个开关分层**。开发阶段 `FEISHU_ALLOW_ALL_USERS=true` 放行所有人；上线前换成 `FEISHU_ALLOWED_USERS` 白名单（逗号分隔用户 ID）。**默认拒绝、显式放行**的思路，避免"调试完忘了关"直接裸奔。`FEISHU_HOME_CHANNEL` 则指定 cron/webhook 默认推送的群，让定时任务有确定的出口。

- **常驻服务而非按需拉起**。飞书是事件回调驱动（订阅 `im.message.receive_v1` 事件，平台把消息推给机器人），所以机器人必须有一个常驻进程收事件。网关支持 `hermes gateway run`（前台调试）和 `hermes gateway install`（装成 systemd/launchd 后台服务）两种形态——**开发与生产用同一套代码，只是托管方式不同**，这也是 `hermes gateway setup` 交互式向导存在的意义：把"该配什么"固化成流程。

- **适配器做"能力映射"，Agent 只表达意图**。飞书适配器能收发语音/图片/文件、支持流式输出、表情反应、卡片交互按钮、打字指示器——但每个平台的能力不一样，Telegram 有的飞书未必有。网关层不假设平台能力，由适配器把 Agent 的"回复文本 + 附件"翻译成各平台自己的消息形态，再处理各平台的回调事件（收消息、点按钮、加表情）。**平台差异在适配层消化，Agent 核心永远只说通用语言**，这就是能同时接 20+ 平台而不互相打架的原因。

## 关键机制/流程

配置一条龙：开放平台创建企业自建应用 → 拿 App ID（`cli_` 开头）/ App Secret → 开权限（`im:message` 收发消息、`im:resource` 下载图片/文件/语音、`contact:user.employee_id:readonly` 读用户信息）→ 订阅 `im.message.receive_v1` 事件 → 写入 Hermes 环境变量 → 发布应用版本（审批通过后才能在联系人里搜到机器人）→ 启动网关。

语音消息需要额外配 STT 引擎把语音转文字：本地 Whisper（`faster-whisper`，免费无 key）、Groq Whisper（免费额度）或 OpenAI Whisper（付费），用 `hermes config set stt.provider <local|groq|openai>` 切换，`stt.enabled true` 启用后重启网关。

## 相关笔记

- hermes-feishu-markdown-issue（飞书适配器的 markdown 渲染取舍）
- hermes-provider-profile-design（ProviderProfile：provider 差异如何声明式建模）
- hermes-source-study（本系列总览）
