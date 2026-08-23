# 把 tool_calls 的 JSON 重排成标准字节：prompt cache 的省钱密码

> 源码位置：`agent/conversation_loop.py` → 请求组装阶段的 tool_calls 规范化循环（`json.loads` + `sort_keys` 重排）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

每次发请求前，遍历所有带 tool_calls 的 assistant 消息，把每个工具调用的 `arguments`（一串 JSON 字符串）做"解析 → 重排 → 再序列化"：用紧凑分隔符（`separators=(",", ":")`）和键排序（`sort_keys=True`）重新输出。**目标是：同一份 JSON，无论经过多少次序列化、无论模型当初怎么生成，字节完全一致。**

## 为什么这么设计

- **JSON 的"语义"和"字节"是两回事，而缓存只认字节**。`{"b": 2, "a": 1}` 和 `{"a": 1, "b": 2}` 语义相同，但字节不同。模型每次生成 arguments，键序、空格都可能变。如果不重排：同一轮对话里每次生成的字节不同，跨回合重放时历史字节和当时发的不一致——**KV cache 按字节匹配，前缀一漂移，缓存就失效**，每次都是全价推理。

- **规范化让"发出字节 = 重排字节 = 历史存储字节"三合一**。先 `json.loads` 解析成语义等价的数据结构，再统一格式重新序列化——无论原始格式是缩进的、乱序的还是带空格的，重排后都收敛到同一个标准字节串。这样跨回合重放不会漂移，本地推理服务器（llama.cpp / vLLM / Ollama）能复用 KV cache，云端命中率也提高。**prompt cache 命中能省约 90% 成本，这是把工程细节当设计原则的典型**——Hermes 的 AGENTS.md 第一条就是前缀稳定。

- **成功路径函数式重建，失败路径交给修复器**。解析成功的条目用字典展开重建（`{**tc, "function": {...}}`），保留 id/type 等字段、只换 arguments，且**不原地改原列表**——新列表建好后整体替换；`json.loads` 失败（模型生成了坏 JSON）则交给 `_repair_tool_call_arguments` 原地修复。**成功路径不污染原数据，失败路径有专职兜底**，两条路各司其职。

- **与前后环节分工明确：消毒管非法、安全网管结构、规范化管格式**。消息消毒（⑬）把损坏 JSON 修成 `{}`；安全网（⑲）修孤儿/缺失/空名的结构问题；本段只处理**合法但格式不统一**的 JSON，把它重排成标准字节。三者都是"让发出的字节既合法又稳定"这条防线上的不同关卡，规范化失败时还会回调消毒的修复器——环节之间是接力而非重复。

- **防御检查与不改原列表，都是"不信任输入"的体现**。只处理 dict + 含 function 的合法条目，畸形条目跳过留给安全网兜底；新列表整体替换而非原地修改，避免半路改坏正在被其他逻辑引用的消息结构。**每一层只修自己职责内的东西，修不动就留给下一层**——这是防御式编程在长管道里的正确姿势，也是这段代码只有十几行却能稳稳插在 5800 行主循环里的原因。

## 关键机制/流程

```
遍历 api_messages
  → 无 tool_calls 的跳过
  → 逐条校验（dict + 含 function 才处理，畸形条目跳过留给安全网）
  → json.loads 解析 arguments
       ├─ 成功 → sort_keys + 紧凑分隔符重新序列化 → 重建新 dict
       └─ 失败 → _repair_tool_call_arguments 原地修复
  → 新列表整体替换 am["tool_calls"]
```

## 相关笔记

- hermes-message-sanitize（消息消毒：管非法 JSON）
- hermes-tool-call-message-structure（工具调用消息的合法结构）
- hermes-anthropic-cache-control（Anthropic cache_control 打标，规范化后的显式断点）
- hermes-prefill-messages（同样影响请求前缀字节的注入机制）
