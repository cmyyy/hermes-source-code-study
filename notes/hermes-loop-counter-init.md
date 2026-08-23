# 16 个计数器：主循环开跑前的仪表盘初始化

> 源码位置：`agent/conversation_loop.py` → `run_conversation()` 循环入口前的初始化块
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

主循环（while 迭代）开跑前，一口气初始化 16 个局部变量——计数器、状态标志、恢复用的临时仓库，代码注释称之为 "pure locals consumed by the loop below"。它们是一个回合的"仪表盘"：迭代计数、截断恢复进度、压缩尝试次数、验证门扣留的候选回答、MoA 预准备请求、回合退出原因，全部在循环开始前定好初值，主循环按需读写。

## 为什么这么设计

- **全是函数局部变量，而不是 agent 对象属性——天然 per-turn，不需要归零**。这是这份代码最值得讲的设计决策。Hermes 的 gateway 会**跨回合复用同一个 agent 对象**，所以放在对象上的状态（如压缩三标志、过程旁白去重集合）必须在回合边界显式归零，否则上一回合的残留会污染下一回合。而这 16 个变量是 `run_conversation` 的局部变量，函数每回合新建，**从机制上不可能泄漏**——"能用局部变量表达的状态就不放对象上，从设计上消灭一类 bug，而不是靠归零防守"，这是比"记得归零"更高一层的工程实践。

- **compression_attempts 是多道闸共享的计数器**。pre-API 压力闸、413/溢出恢复、post-tool 压缩闸共用同一个计数器和同一个上限（`max_compression_attempts`，默认 3，由配置 `compression.max_attempts` 驱动）。如果每道闸各压 3 次，一个回合可能压缩十几次，既烧钱又拖慢——**共享上限把"总压缩预算"锁死**，任何一道闸用了额度，其他闸就少一次机会。`max_compression_attempts` 用 `getattr` 兜底：老 pickle / 最小 stub 缺属性时回退到 3，保持旧行为。

- **_preflight_compression_blocked 是唯一不从零初始化的变量**。它的值在回合前奏（build_turn_context）里就算好了：如果预检已经证明"压缩无效"（压完仍超阈值），循环内的 pre-API 压缩检查直接跳过防空转。**它是输入，不是状态**——区分"这一回合要消费的状态"和"这一回合带进来的事实"，是读懂这段初始化的钥匙。

- **诊断变量 _turn_exit_reason 每个退出路径都赋值**。正常回答、预算耗尽、用户中断、空响应……每个 break/return 路径都写一个退出原因（如 `interrupted_by_user`、`budget_exhausted`、`text_response(finish_reason=stop)`），finalize 阶段记入日志——排查"这回合为什么没正常回答"时，它是第一手线索，而不是靠猜。

## 关键机制/流程

六组职责：

1. 主循环状态：api_call_count（回合级 API 调用次数，日志 "API call #3/10" 和请求 ID 靠它）/ final_response（循环出口变量）/ interrupted / failed
2. 截断恢复：length_continue_retries（finish_reason=length 续写，上限 4 次）/ truncated_tool_call_retries（tool_call JSON 截断重试，上限 4 次且每次 max_tokens ×2）/ truncated_response_parts（截断时已输出文本的"临时仓库"，续写完成后拼接，用户不丢已听到的部分）/ codex_ack_continuations（防 Codex 模型只回"好的，我继续"不干活，上限 2 次）
3. 压缩控制：compression_attempts / max_compression_attempts / _last_preflight_pressure（上次压缩前压力量了多少，判断压缩有无进展）/ _preflight_compression_blocked
4. 验证门：_pending_verification_response（verify-on-stop 门扣留的最佳回答，预算耗尽时它就是最好的用户可见结果）/ _pending_verification_response_previewed
5. MoA 专用：pending_moa_prepared_request（pre-API 压缩后要 rebase 到新 transcript 的预准备请求，防 advisors 再跑一遍）
6. 诊断：_turn_exit_reason

## 相关笔记

- hermes-loop-entry-phase（回合启动阶段总览，本段是其一部分）
- hermes-compression-per-turn-state-reset（agent 属性必须显式归零的对照案例）
- hermes-interrupt-partial-response-save（interrupted 标志的消费场景）
- hermes-build-turn-context（_preflight_compression_blocked 的产出方）
