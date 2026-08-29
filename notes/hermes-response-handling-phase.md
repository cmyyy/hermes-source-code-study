# 响应处理：模型给的结果能不能信、能不能用
> [← 返回笔记地图](../README.md#笔记地图) · 六阶段核心 · [← 上一篇](hermes-request-execution-phase.md) · [下一篇 →](hermes-error-recovery-phase.md)


> 源码位置：`agent/conversation_loop.py` → `run_conversation()` 响应处理段
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

模型返回响应后，先验货（响应形状对不对）、翻译（把各家结束原因统一说法）、分类处理（拒绝/截断/空响应各有各的恢复路径）、记账（用量/成本/持久化）。一句话：**验货 → 翻译 → 分类处理 → 记账**。

## 为什么这么设计

**先验货再动手**。按 api_mode 分派四家 transport 的 validate_response（codex_responses / anthropic_messages / bedrock_converse / chat_completions）。Codex 的 `status=failed/cancelled` 视为终态失败，标记 response_invalid 后路由进恢复链（错误钩子 → 急切 fallback → 退避），而不是让错误冒泡到循环外走错恢复路径。空/畸形响应是限流常见症状，所以**急切 fallback**——立即切备用模型而非带退避重试主 provider（主 provider 在重试窗口内不会恢复，等退避是浪费时间）。

**确定性拒绝绝不重试**。模型返回内容安全拒绝（Anthropic refusal、finish_reason=content_filter、Bedrock guardrail）时，同样 prompt 一定再拒——重试 = 烧钱复现必然结果（注释原话：不处理会被误判成"限流"走重试循环）。只切一次 fallback（不同模型可能不拒），否则终态返回结构化拒绝结果。注意同源不同处理：完整拒绝（内容为空）不重试；**流式中途被内容过滤器掐断**（#32421，已有部分输出）则 best-effort 续写——续写生成的是过滤器没拦过的新内容，有胜率。

**length 截断按"截断的性质"分四路**，不能一刀切：① thinking 预算耗尽（有 think 标签、think 块后无任何正文）→ 续写无意义，直接给针对性错误（"降低 reasoning effort 或增大 max_tokens"），省 3 次 API 调用；② 文本截断 → 已输出部分 append 进历史，最多 4 次续写；③ 工具调用截断 → 最多 4 次重试且每次 max_tokens ×2（2^retries，封顶 32768），坏响应不 append 进消息，直接从当前状态重发；④ 都不是 → 回滚到最后完整 assistant 回合，首条消息就截断则无法恢复（标记 failed）。

**记账要归一化、要防双计**。三家 API 的 usage 形状经 `normalize_usage` 归一；MoA 把 advisor fan-out 用量折进回合（否则整个 advisor 花费不可见）；`update_from_response` 把真实用量写进压缩器——这是后续 post-tool 压缩判断的"真值"来源；SQLite 写入用绝对总量覆盖 per-call 增量，同一回合多次写不会重复计数。

**中断时抢救半截回答**。流式输出中被用户打断，如果不把已输出的文本存进历史，模型下回合"失忆"——看不到自己刚说的话，对"改成 X"这类纠错的理解就会跑偏。所以 `_current_streamed_assistant_text` 剥掉 think 块后 append 进消息并持久化；但 redirect 纠错场景不保存半截（会污染纠错上下文），只取消本次请求、保留纠错队列。

**退避重试也要保持"活人"状态**。次数没用完时指数退避（5 秒基、120 秒上限，随机抖动防惊群），但等待不是死等：0.2 秒切片 sleep 让用户随时能中断，每 30 秒 touch activity 防止 gateway 的失活监控把会话杀掉。记账侧还有个细节：无 usage 时压缩后等待真实用量判定，用空 dict 消费 pending 判定，避免"预检延迟"永久锁死；成本估算时 MoA 用聚合器真实模型计价（虚拟 preset 名 "moa" 没有定价条目），advisor 成本按各自模型价追加。

## 关键机制/流程

validate_response 验货 → 无效走恢复链（错误钩子 → 急切 fallback → 人类可读错误提示 → 退避重试/终态）→ finish_reason 归一化映射（Ollama/GLM 的可疑 stop 重分类为 length，防止"假报完成"吞掉后半段回答）→ content_filter 拒绝不重试 → length 截断四路分派 → token 记账（喂压缩器）→ 会话统计/成本/持久化 → 成功收尾，break 退出中层 retry 循环。

## 相关笔记

- hermes-conversation-loop-run-conversation.md
- hermes-request-execution-phase.md
- hermes-pre-api-pressure-check.md
- hermes-credential-pool-turn-reset.md
- hermes-interrupt-partial-response-save.md
- hermes-error-recovery-phase.md
