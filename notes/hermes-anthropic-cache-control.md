# Anthropic 提示词缓存打标：让多轮对话便宜 90% 的显式断点

> 源码位置：agent/conversation_loop.py → `apply_anthropic_cache_control`（消息处理链的最后一步）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

Anthropic 的 prompt caching 是**显式断点缓存**：在请求里给 system 或消息打上 `cache_control: {"type": "ephemeral"}` 标记，断点之前的内容的 KV 状态会被缓存在 Anthropic 服务器端，TTL（5 分钟，可续期到 1 小时）内发送相同前缀时直接命中缓存。缓存读取价格只有正常的 0.1 倍（便宜 90%），首字延迟（TTFT）快 2-5 倍。

和 OpenAI 的关键区别：**OpenAI 自动缓存不用管，Anthropic 必须显式标记"缓存哪段"**。Agent 场景命中率天然极高——system prompt 和工具集几乎不变，只有对话历史在增长，前缀大部分字节是稳定的。

## 为什么这么设计

- **打多个断点，覆盖"几乎不变"的大块**：静态 system 前缀（如有独立静态部分）、完整 system prompt、最后两条消息——长 system 和几十个工具 schema 是最大的固定块，一个断点覆盖一块，让缓存收益最大化。

- **必须在消息处理链的最后跑**：打标会把 content 从纯字符串改写成 `[{"type": "text", ...}]` 的块结构。如果提前跑，后续的空白规范化就匹配不上 `isinstance(content, str)` 的判断，字节稳定性被破坏。**排最后不是随意选择，是结构约束**——它改写数据形态，任何依赖字符串形态的步骤都得在它之前。

- **Hermes 本地干的事是"保证前缀字节稳定"**：system prompt 每会话只建一次、消息 JSON 键重排固定、历史消息用 sidecar 重放当年发送的字节、空白规范化……这一整套工程手段，最终都服务于同一个 cache_control 断点——**断点前的字节永远一致 = 缓存永远命中 = 便宜 90%**。缓存实体在 Anthropic 云端（按 API key 隔离），Hermes 本地没有任何缓存文件，它只"告诉服务器该缓存哪段"。

- **按配置开关**：`_use_prompt_caching` 由 config 控制，TTL 可选 5 分钟或 1 小时——把"是否用缓存"留给用户决定，代码只负责打标打得对。

## 关键机制/流程

完整链路：Hermes 本地保证前缀字节稳定（system 只建一次、JSON 重排、sidecar 重放）→ 消息处理链最后打 cache_control 断点（静态 system 前缀 + 完整 system + 尾部消息）→ Anthropic 服务器把断点前内容存入云端缓存 → 下次相同前缀直接命中（0.1x 价格 + 更快）。类比：图书馆借书——第一次借（write，1.25x 略贵）管理员把书放前台，5 分钟内再借同一本（hit）直接前台拿，不用去书架找。

## 相关笔记

- api-messages-build（sidecar 重放，字节稳定的手段之一）
- message-normalization（消息规范化）
- conversation-loop-run-conversation（主循环）
- agent-init-structure（初始化时自动开启缓存）
