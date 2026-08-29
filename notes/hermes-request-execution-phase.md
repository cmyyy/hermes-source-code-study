# 请求执行：怎么把请求安全发出去、顺利拿回来
> [← 返回笔记地图](../README.md#笔记地图) · 六阶段核心 · [← 上一篇](hermes-request-assembly-extras.md) · [下一篇 →](hermes-response-handling-phase.md)


> 源码位置：`agent/conversation_loop.py` → `run_conversation()` 请求执行段
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

请求组装完成后进入执行段：显示等待动画、把请求参数打包好交给中间件和插件过目、决定流式还是非流式、真正调用 API、计时收尾。它是中层 retry 循环（`while retry_count < max_retries`）的主体——单次 API 调用失败后的重试与恢复也在这层展开。

## 为什么这么设计

**限流时干脆不发（前置守卫思维）**。Nous Portal 被限流时，每次尝试（含 SDK 级重试）都烧 RPH 配额，发一次就加深限流坑——限流状态下重试是负收益。所以发请求前先查 `nous_rate_limit_remaining()`：有 fallback 就切换并重设重试计数，没有就返回限流提示。守卫自身的异常要吞掉，绝不让它打断 agent 循环。

**默认流式，即使没有任何消费者**。这不是性能优化而是可靠性设计：流式路径才有细粒度健康检查（90 秒 stale-stream 检测、60 秒读超时），非流式路径遇到 provider 只发 SSE 心跳包、迟迟不吐内容时会**无限挂起**——子代理等静默调用方可能永远等下去。只有明确的排除项才走非流式：provider 标记不支持流式、Copilot ACP 子进程协议、MoA 无流式消费者、测试 Mock。

**请求参数打包成 kwargs 字典而非显式参数**。provider 参数高度动态：中间件要整体改写 payload（保留 original_payload 供对比/回滚）、插件钩子要加字段、MoA 要塞私有对象（在最后、所有可能序列化的环节之后注入，防止被 dump 或序列化）。字典是唯一能同时容纳这些改写的容器。

**处理"响应返回瞬间用户纠错"的竞争**。响应与 redirect 的竞争用标志位协调：`_redirect_crossed_response` 为真说明响应是上一秒的答案、用户已说"不对"——直接丢弃过期响应，从纠错消息重建（clear_interrupt 保留 redirect + restart_with_redirected_messages），不浪费下一轮。

**本段是三层循环的中间层**。整个回合是三层 while：外层管回合迭代（最多 N 次 + 迭代预算 + 宽限）、中层管单次 API 调用的重试（本段）、内层按 `_retry` 标志分派恢复方式（换模型？压缩？退避？）。每个"失败"发生在哪一层就在哪一层处理——一次网络抖动不该把整个回合推倒重来。retry 循环初始化时把 `api_request_id`（turn_id:api:调用次数）定好，日志与追踪全程贯穿。

**等待反馈也要分层**。非 quiet 模式打印请求摘要（第几次调用、消息数、token 数），quiet 模式走动画——TUI 用 thinking_callback（prompt_toolkit widget），纯终端用原始 KawaiiSpinner；但只要有流式消费者或不该起 spinner 就不用原始动画，避免动画刷新和流式文本显示互相打架。调用期间用 `_model_request_active` 标记请求进行中，供 redirect 跨线程竞争检测。

## 关键机制/流程

thinking spinner（非 quiet 打印请求摘要 / quiet 走动画，有流式输出时不起动画避免打架）→ retry 循环初始化（`_retry` = TurnRetryState 状态机）→ Nous 限流守卫 → 构建 kwargs + LLM 请求中间件 + pre_api_request 插件钩子 → 流式/非流式决策 → `_interruptible_*_api_call` 执行调用 → 计时、停 spinner、安静收尾（把屏幕让给更有价值的响应内容）。

## 相关笔记

- hermes-conversation-loop-run-conversation.md
- hermes-pre-api-pressure-check.md
- hermes-response-handling-phase.md
- hermes-interrupt-partial-response-save.md
