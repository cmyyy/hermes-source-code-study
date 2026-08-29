# api_messages 组装：内部世界与外部 API 之间的翻译层
> [← 返回笔记地图](../README.md#笔记地图) · 专题深挖


> 源码位置：agent/conversation_loop.py → run_conversation（消息转换段）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

把内部消息列表 `messages` 逐条转换成"可以发给 AI 的干净副本"`api_messages`。这是内部世界与外部 API 之间的翻译层：内部消息是"富形态"——带着推理内容、注入标记、展示元数据、记账 sidecar；而 API 只认标准 schema。这段代码负责抹平差异：**删掉 API 不认识的，改写 API 需要的，全程不动原件**。

## 为什么这么设计

- **副本原则：原件永不动**。每条消息先 `copy()` 再处理。内部消息是存档历史，任何转换操作都在副本上进行——转换出错最坏影响一次请求，不会污染会话历史。

- **富→贫的定向转换**。内部字段按用途处理：`reasoning`（推理轨迹，内部存储用）改名成 API 认识的 `reasoning_content`；`finish_reason`、`_thinking_prefill`、`display_kind`/`display_metadata`（展示用时间线元数据）直接删除——严格 provider 收到不认识字段会拒收。Codex Responses 特有的字段（call_id、response_item_id）在非 Codex 模式下全部剥离，否则 Mistral/Fireworks 这类严格 API 会拒绝请求。

- **当前消息注入 vs 历史消息重放，两种改写策略**。当前轮的 user 消息有"新料"可加（这轮的记忆 prefetch、插件上下文注入），替换成注入后的发送版；历史消息没有新料，只能重放当年实际发送的字节——这个字节存在 `api_content` sidecar 里。**重放的动机是前缀缓存**：AI 侧缓存按字节匹配，历史消息的字节变了，缓存全部失效，等于每轮都全价重算。

- **"塞一次、pop 无数次"的 sidecar 生命周期**。回合开始时 build_turn_context 组装好发送版并贴上 `api_content`（同一行写入数据库持久化）；主循环每轮迭代 pop 出站副本的 sidecar——当前消息贴进 content，历史消息重放 content，用完把记录本身摘掉。sidecar 是"记账单"，告诉组装段"当年发的是这些字节"。

## 关键机制/流程

对每条消息的五步处理：① 复制（原件不动）→ ② 剥离内部字段（api_content 暂存、display 元数据删除）→ ③ 按需改写 content（当前轮注入、历史轮重放）→ ④ 推理字段处理（reasoning → reasoning_content，删原字段）→ ⑤ 工具调用字段消毒（严格 API 剥离 Codex 特有字段，MoA 模式解析聚合模型名）→ append 到 api_messages。产出的列表是发请求前的最终形态，之后依次是 system prompt 组装、prefill 注入、context engine 选择。

## 相关笔记

- build-turn-context（sidecar 的产生地）
- message-sanitize（内容修复；本段是格式转换，两者分工不同）
- prefill-messages（本段之后的插入）
- context-engine-selection（本段之后的选择/替换）
- anthropic-cache-control（重放字节的最终受益者）
