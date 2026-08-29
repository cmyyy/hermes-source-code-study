# 被打断时抢救半截回答：流式 Agent 的上下文一致性
> [← 返回笔记地图](../README.md#笔记地图) · 专题深挖


> 源码位置：`agent/conversation_loop.py` → `except InterruptedError:` 分支（`_interruptible_streaming_api_call` 的调用点）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

模型正在流式输出时用户打断（比如"不是这样，改成 X"），Hermes 在 `InterruptedError` 分支里**把已经流式输出的半截回答抢救出来**：从 `_current_streamed_assistant_text` 取出本次已传给用户的全部文本，剥掉 think 推理块、去空白，作为 assistant 消息 append 进消息历史并持久化；如果一个字都没来得及输出，则写入一条"还在等模型响应 X 秒"的占位消息。

## 为什么这么设计

- **不保存半截内容，下一回合模型会"失忆"**。流式输出是边生成边展示的，中断发生时这 200 字还没有 append 进消息历史。如果不抢救，下一回合模型看到的对话里**没有自己刚说过的内容**，用户基于半截内容提出的纠错就失去了参照——模型以为自己在回答一个全新问题。代码注释原话：Dropping it leaves history with no record of the half-finished reply on screen, so the next turn the model 'forgets' what it just said。**补救的核心动作是"把半截内容落进历史 + 持久化"**，让下一回合重载历史时看到"我上回合说到一半 + 用户打断并要求 X"，纠错才有依据。

- **工程原则：用户看到的，必须等于历史里记录的**。流式 UI 展示的内容与持久化历史不一致，是流式 Agent 最隐蔽的坑——界面上说了一半，历史里没记，多轮对话的上下文就此错位。这条原则比"保存半截回答"这个具体实现更值得记住，它直接指导了这里"取流式变量 → 进 messages → 持久化"的三步动作。

- **只救"用户屏幕上已看到的内容"，不救推理过程**。半截文本先过 `_strip_think_blocks` 剥掉 think 块——推理内容是模型的内心独白，不是对用户说过的话，不该污染历史；两分支里还有一句"一个字没输出"的兜底：给占位消息而不是空着，让用户明确知道"模型还在响应，只是被打断了"。

- **redirect 纠错场景特判跳过**。如果中断是为了纠错（/redirect 命令），只取消本次请求、保留纠错队列、让外层循环重建请求——此时**不保存半截内容**，因为纠错要基于全新请求，保留半截反而会污染纠错上下文。三种中断场景（循环边界干净退出 / API 流式中抢救半截 / redirect 重建请求）各司其职，互相补充。

- **半截内容直接当 final_response**。抢救出来的内容不仅进历史，还作为本次回合的最终答复返回——用户屏幕上看到的半截话就是"这一回合的答案"，两处保持一致；一字未出时，占位消息里的 `api_elapsed` 如实报告已等待秒数，让"模型其实响应了 X 秒"这个事实留在记录里，也避免用户面对一个空白的回合。

## 关键机制/流程

```
用户打断 → InterruptedError
  ├─ 是 /redirect 纠错 → 取消请求，保留纠错队列，外层重建（不保存半截）
  └─ 普通中断
       ├─ 有半截内容 → 剥 think 块 → append 进 messages → 作为 final_response → 持久化
       └─ 一字未出   → 占位消息（"还在等模型响应 X 秒"）→ 持久化
```

## 相关笔记

- hermes-loop-entry-phase（主循环入口与中断检查闸门）
- hermes-loop-counter-init（interrupted 标志的初始化）
- hermes-steer-mechanism（/steer：不打断的纠错，与 interrupt 对照）
- hermes-conversation-loop-run-conversation（主循环总纲）
