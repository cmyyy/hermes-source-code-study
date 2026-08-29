# 工具调用消息的"合法结构"：配对、顺序、无孤儿、无空名
> [← 返回笔记地图](../README.md#笔记地图) · 专题深挖


> 源码位置：OpenAI 兼容工具调用协议；Hermes 在 `agent/conversation_loop.py` 的消毒层与安全网中维护
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

工具调用在消息协议里有严格的形状要求：**assistant 消息声明 tool_calls（每个含唯一 id + function.name + JSON 字符串 arguments），其后必须紧跟同 id 的 tool 结果消息（role="tool"、tool_call_id 指向那个 id）**。调用-结果一一配对、顺序相邻、无孤儿、无空名，这就是合法结构。模型经常生成不满足它的消息（重复 id、参数 JSON 损坏、空工具名、半截调用），Hermes 在发送前用消毒层和安全网把消息修复成合法形状，否则严格 provider 会 400 拒收。

## 为什么这么设计

**四要素缺一不可**。① 配对：tool 结果的 `tool_call_id` 必须等于 assistant 声明里的 `id`，一一对应；② 顺序：assistant（带 tool_calls）必须紧跟 tool 结果，中间不能插入其他消息；③ 无孤儿：只有声明没有结果（有调用没结果）、只有结果没有声明（孤儿结果）都不行；④ 无空名：`function.name` 不能是空字符串——空名会被 provider 静默丢弃，导致结果变孤儿。

**三类修复各有讲究**。补 stub（`make_tool_result_message` 生成占位内容）保住"调用-结果"配对；删孤儿防止多余消息干扰后续配对；空名改成哨兵名 `"invalid_tool_call"`——**不删**，因为删了合成结果就无配对、反制信号丢失；改成哨兵名能保住 "tool name was empty" 的反馈，让弱模型自我纠正（#47967）。

**角色交替是底层约束**。tool 之后只能跟 assistant 或下一轮 user，修复时不能插入新 user 消息——这也是 /steer、空响应 nudge 都选择"搭便车"而非"插消息"的原因：任何恢复路径都必须维护消息顺序的合法性。

**为什么 provider 这么严格**。工具调用是"模型说了算、系统照着做"的异步契约：系统必须能凭 tool_call_id 把结果精确送还给对应的调用——id 对不上、顺序乱了、多了孤儿，系统就不知道结果该归谁，严格 provider 直接 400 拒收，宽松的则静默丢弃（空名场景），静默丢弃比报错更糟：模型以为自己调了工具，实际什么都没发生。所以 Hermes 把"合法结构"当成协议底线，在消毒层和安全网里各修一遍。

**为什么 arguments 是 JSON 字符串**。它是模型生成的，必须走"字符串"这个通用载体（模型输出即文本），语义在字符串内部；因此任何对参数的读写都要先 parse——这也解释了为什么需要规范化（按键排序重排字节）和参数强转（coerce）两道工序，它们都建立在"arguments 是字符串"这个事实上。多工具并行时每个 id 必须唯一，结果消息按 id 归位、顺序无所谓，靠的是 id 配对而非位置配对。

## 关键机制/流程

一个完整合法的工具回合是四条消息：user 提问 → assistant 声明 tool_calls（含 id / type / function.name / arguments）→ tool 结果（tool_call_id 指向声明 id）→ assistant 最终回答。多工具调用时，一条 assistant 消息可声明多个 tool_calls，结果消息按 id 配对依次排列（顺序无所谓，id 配对就行）。

```json
{"role": "user", "content": "北京今天天气怎么样？"}
{"role": "assistant", "content": null, "tool_calls": [{"id": "call_abc123", "type": "function", "function": {"name": "get_weather", "arguments": "{\"city\": \"北京\"}"}}]}
{"role": "tool", "tool_call_id": "call_abc123", "name": "get_weather", "content": "北京今天 25°C，晴"}
{"role": "assistant", "content": "北京今天 25°C，晴天，适合出门。"}
```

## 相关笔记

- hermes-conversation-loop-run-conversation.md
- hermes-message-sanitize.md
- hermes-api-messages-build.md
- hermes-tool-execution-finalize-phase.md
- hermes-request-assembly-extras.md
