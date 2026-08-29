# 过程旁白去重：一个 set 的回合边界纪律
> [← 返回笔记地图](../README.md#笔记地图) · 专题深挖


> 源码位置：agent/conversation_loop.py → run_conversation（每回合开头重置）；run_agent.py → `_interim_text_was_delivered` / `_record_delivered_interim_text` / `_fire_streamed_codex_commentary`
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

Agent 一轮对话里，模型可能分多次返回内容：续写、工具调用前的旁白、Codex commentary。这些中间内容通过 `interim_assistant_callback` 实时流式推送给用户看。问题是：**同一文本片段可能在同一个回合内被多次检测到**（provider 续写、多次 normalize_response、流式重连/重试后重发），不记录就会重复刷屏——用户会看到两遍"正在做 X…"。

`_delivered_interim_texts` 就是去重账本：一个 set，发过一次的文本记进去，再次遇到跳过。每回合开头把它置空。

## 为什么这么设计

- **去重范围 = 整个回合，但绝不跨回合**。注释原文点明了两条边界：去重"跨越同一用户回合内的所有 provider 续写和工具调用"（回合内共用一个账本），但"不能抑制下一回合的同一句话"——下一回合用户问同样的问题、模型说同样的开头，那是正常回答，不该被当成重复跳过。所以每回合开头 `= set()` 重置：**上一回合的记录清掉，本回合重新记账**。

- **规范化后查重，防"同话不同形"**。存进 set 的是规范化后的文本（`_normalize_interim_visible_text`：strip think 块、flatten 多模态内容）——保证"同一句话换了格式"也能被识别为重复，而不是靠字节完全一致才算重。

- **使用路径自带完整纪律**：`_fire_streamed_codex_commentary` 发送前先查重（发过就 return），发完立刻记录。查 → 发 → 记三步锁死，中间没有漏记的窗口。

- **它是"状态边界归零"模式的又一个实例**。与压缩三标志重置、认证池刷新计数归零是**同一个设计模式的多次应用**：gateway 跨回合复用 agent 对象，所有 per-turn 状态必须在回合边界显式归零。不清零的后果：压缩状态残留会把未压缩回合误判成压缩过、transcript 被错误重写；去重集合残留则会把本回合的正常回答当成上一回合的重复跳过——**同样的根因，同样的解法**。

## 关键机制/流程

回合开始：`agent._delivered_interim_texts = set()` → 回合中：每次要推送过程旁白，先 `_interim_text_was_delivered(text)` 查重（发过跳过）→ `cb(visible, already_streamed=False)` 推送给 UI → `_record_delivered_interim_text(visible)` 记账。Java 类比：WebSocket 推送的消息去重窗口——同一事件 ID 只推一次，窗口过期后可重推；或者请求处理中的"已发送通知集合"——同一次请求内不重复通知，不同请求各自独立。

## 相关笔记

- compression-per-turn-state-reset（压缩三标志重置，同一模式）
- credential-pool-turn-reset（认证池归零，同一模式）
- loop-counter-init（循环计数器，对比局部变量与属性）
- conversation-loop-run-conversation（主循环全解）
