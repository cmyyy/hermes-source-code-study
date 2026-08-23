# prefill 消息：塞进对话开头的 few-shot 示范

> 源码位置：`agent/conversation_loop.py` → 请求组装阶段的 prefill 插入段；配置来源 `cli.py` → `_load_prefill_messages()`（`prefill_messages_file`）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

prefill messages 是**预先塞进对话历史的"示范消息"**（few-shot 示例）：每次 API 调用时，在 system prompt 之后、真实历史之前插入若干条来自配置文件的示例消息，给模型示范"该按什么格式/风格回答"。它们是临时的（API-call-time-only），**绝不入库**——只存在于发给 provider 的消息里，不写会话历史。

## 为什么这么设计

- **位置敏感：紧跟 system、先于历史，保证是"示范"而非"对话的一部分"**。插入逻辑先检查位置 0 是不是 system 消息，是则从位置 1 开始插入（每条 `pfm.copy()` 副本，原件不动）。最终顺序：`[system prompt] → [prefill 1..N] → [真实历史...] → [当前用户消息]`。模型看到的是"先给我看了几个标准答案的样子，再让我看真实对话"——**示范放在最前面，引导效力最强，又不会和真实历史混在一起被当成事实**。

- **few-shot priming 比纯指令更有效**。代码注释直白地写着用途：Ephemeral prefill messages (few-shot priming, never persisted)。让模型按特定 JSON 格式输出，给 2-3 个格式范例比在 system 里写"请按 JSON 输出"可靠得多；模仿语气风格、理解任务类型同理。**与其描述"要什么"，不如展示"长什么样"**——这是与模型沟通的基本原则，也是 prefill 存在的根本理由。

- **API-call-time-only 是刻意的边界**。prefill 只存在于发给 API 的消息里，不写会话历史/轨迹——它们是"示范道具"，不是"发生过的事实"。这条边界让 prefill 可以随时改、随时删而不污染长期记忆。与之对照，`ephemeral_system_prompt` 也是临时内容，但它是**拼进 system 文本**；prefill 是**独立消息条目**（可多条、有 role），两者是"改一段话"与"插几段话"的区别。需要带角色（如 assistant 示例）的示范只能用 prefill 表达。

- **prompt cache 视角：prefill 是请求前缀的一部分**。它插在 system 之后，处在缓存前缀内——内容会话内不变则缓存命中，变了则前缀 miss。**任何会动的注入都要考虑对前缀稳定性的影响**，这是 Hermes 全代码库反复出现的同一根弦：省钱的机制（前缀缓存）与引导的机制（prefill）在字节层面是同一笔账。

- **配置文件驱动，改示例不用改代码**。prefill 内容来自 `prefill_messages_file` 指向的 JSON 文件（消息数组），CLI 启动时加载——换格式示例、调风格示范，编辑 JSON 即可，不碰源码。**引导内容属于"内容"而非"代码"**，用配置文件承载，既降低使用门槛，也避免把示例写死进程序里；示例通常只需 2-3 条，体积小，对前缀的影响可控。这类"临时注入、用完即弃"的机制在 Agent 系统里很常见，它们的共同点是把**引导**与**事实**严格分开——prefill 是引导，历史是事实，混在一起迟早出事。

## 关键机制/流程

```
配置：prefill_messages_file → JSON 文件（消息数组，role/content 列表）
加载：CLI 启动时读入 AIAgent 构造参数（agent.prefill_messages）
插入：每次 API 调用组装消息时
  → 位置 0 是 system？→ 从 1 开始；否则从 0 开始
  → 逐条插入副本（pfm.copy()）
最终：[system] → [prefill...] → [真实历史] → [当前用户消息]
```

## 相关笔记

- hermes-api-messages-build（api_messages 组装，本段在其后插入）
- hermes-message-normalization（请求前缀的字节稳定）
- hermes-moa-context-injection（同为"往消息里注入内容"的不同手法）
- hermes-conversation-loop-run-conversation（主循环总纲）
