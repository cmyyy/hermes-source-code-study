# 请求组装：把内部消息变成"模型收得下、不浪费钱"的请求
> [← 返回笔记地图](../README.md#笔记地图) · 六阶段核心 · [← 上一篇](hermes-loop-entry-phase.md) · [下一篇 →](hermes-request-execution-phase.md)


> 源码位置：`agent/conversation_loop.py` → `run_conversation()` 请求组装段
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

Hermes 内部维护的消息是"富形态"——带推理字段、注入标记、记账 sidecar；而模型 API 只认"贫形态"的标准字段。请求组装段负责在每次调用前把内部消息消毒、转成 API 格式、注入各类上下文，再估算 token 压力，核心目标是发出一个**模型收得下、且不浪费钱**的请求。

## 为什么这么设计

**守护前缀字节稳定（省 90% 成本的设计灵魂）**。LLM 前缀缓存按字节匹配，历史消息的字节一变，缓存全部失效。因此组装时：system prompt 每会话只构建一次、逐字重放（缓存锚点）；历史消息通过 sidecar 重放"当年真实发送的字节"，绝不用当前内存版本；工具参数 JSON 被解析后按键排序重排（`json.dumps(sort_keys=True)`），同一语义无论模型生成多少次，字节完全一致。Hermes 还在最后一步给 Anthropic 打 `cache_control` 断点——缓存读取只要 0.1 倍价格。

**不信任任何上游，发请求前最后把关一次**。消息历史可能混入两类脏数据：工具参数 JSON 损坏、角色交替违规。不带修直接发，provider 要么 400 拒收，要么**静默返回空内容**（比报错更糟，会触发空响应重试死循环）。所以组装链开头有消毒层：参数损坏改成合法空 `{}` 并给对应工具结果打损坏标记（模型下轮能看到、有机会自我纠正）；角色交替违规则合并相邻 assistant、丢弃孤儿 tool 消息。修复时还同步修正数据库写入游标 `_last_flushed_db_idx`，否则回合结束持久化会跳过消息链（#44837）。

**所有注入都不碰 system prompt**。MoA 聚合参考答案、prefill 示范消息、插件上下文全部注入 user 消息或独立消息条目，system prompt 只留给 Hermes 内部——**稳定的东西放缓存区，可变的东西放数据区**。插件内容每轮可能不同，塞进 system 就破坏了缓存锚点。

**pre-API 压力检查用"实时但有噪声"的估算，不用"准但滞后"的真值**。API 返回的真实 token 数只在调用时产生，工具结果 append 后的暴涨看不见（真实事故：Codex 271k/272k 失败）。所以每次发请求前重新估算完整请求——工具 schema 必须算进去（50+ 工具 = 20-30K token，不算会低估）——超阈值先压缩再发；压缩后压力没降就标记 `_preflight_compression_blocked` 停止主动压缩，但保留 provider 报错驱动的被动压缩兜底。

**当前消息与历史消息，组装策略完全不同**。当前轮的 user 消息有"新料"（这轮的记忆 prefetch、插件上下文），可以现拼现注入；历史消息没有新料，只能通过 sidecar 重放当年发送的字节——sidecar 在回合前奏塞一次（`turn_context.py` 记录"要发的字节"），之后每次构建出站副本时 pop 掉：当前消息贴进 content、历史消息重放。为什么历史必须重放？"交卷后不能再改答案"——前缀缓存按字节匹配，历史字节变了缓存全失效，改了就要全部重判。

## 关键机制/流程

一条组装链：消息消毒 → 构建 api_messages（剥离内部字段、重放 sidecar 字节）→ system prompt 置顶 → MoA 上下文聚合注入 → prefill 示范插入 → context engine 插件钩子（RAG 检索的落点）→ 孤儿 tool 清理安全网 → 消息规范化 → surrogate 字符清理 → Anthropic 缓存打标（必须最后跑，避免破坏字节稳定性）→ MoA 预跑 advisors → token 估算 → Ollama 上下文预检 → pre-API 压缩压力检查。

## 相关笔记

- hermes-conversation-loop-run-conversation.md
- hermes-message-sanitize.md
- hermes-api-messages-build.md
- hermes-moa-context-injection.md
- hermes-prefill-messages.md
- hermes-context-engine-selection.md
- hermes-tool-call-message-structure.md
- hermes-message-normalization.md
- hermes-anthropic-cache-control.md
- hermes-pre-api-pressure-check.md
