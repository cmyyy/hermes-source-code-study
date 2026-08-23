# 工具执行与收尾：安全执行，干净收尾

> 源码位置：`agent/conversation_loop.py` → `run_conversation()` 工具执行与收尾段
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

前半程保证"请求发得出去、响应拿得回来"，到这里模型已经说了"我要用某某工具"。本阶段像收到外卖订单后的处理流程：核对订单（工具名、参数）→ 有问题的挑出来退回 → 确认无误才执行 → 结果记到账上 → 给用户最终回答、收尾。核心问题：**怎么安全地执行；执行完了，怎么干净地收尾**。

## 为什么这么设计

**工具验证两关，别让一颗老鼠屎坏了一锅汤**。第一关验名字：`_uniquify_tool_call_ids` 修重复 id（否则两个调用的结果记在同一个 id 下互相冲掉）、`_repair_tool_call` 纠正拼错；混合批次（一批调用里有好有坏）只退回坏的、好的照常执行——一个坏调用不让整轮作废（大上下文模型容易批量生成，一个错就全废很浪费）。第二关验参数 JSON：明显被截断（不以 `}`/`]` 结尾）直接拒绝执行（执行了也是错的）；其他损坏 3 次重试后注入一条工具错误结果让模型看到"参数坏了"自我纠正——注入的是 tool 消息不是 user 消息，保持角色交替。

**执行前先记账后动手**。破坏性工具（改文件、删东西）执行前先把"我要做这件事"持久化进数据库——万一 Hermes 中途崩溃/重启，resume 时能看到"这个工具已执行过"，不重复执行也不丢状态。无效调用先补错误结果再从执行集剔除（provider 要求每个调用都有对应结果消息）。仅 execute_code 的回合退还迭代预算——便宜的事不占大额配额。

**空响应 7 级处理，按成本从低到高**。模型返回空内容的原因很多（连接断、模型弱、纯思考、真空白），不能一刀切：已流出的内容先用（连接断了也不浪费）→ 复用全 housekeeping 回合的记录 → post-tool nudge 推模型一把（弱模型 #9400 常见）→ 纯思考无正文则记入历史继续 → 真空白重试 3 次 → 持续空白切 fallback → 终态存哨兵 `_empty_terminal_sentinel`。哨兵防的是**重放死循环**：空响应终态会往历史里 append 一条 `assistant("(empty)")`，下回合加载时弱模型可能被"空先例"带偏再次输出空……循环烧钱；哨兵标记这条不是真回答，脚手架清理时 pop 掉、不进持久历史，模型下回合看不到 "(empty)"，循环被打断。

**改了代码不能直接停（verify-on-stop 门）**。agent 编辑了代码必须强制续一轮让模型验证自己的改动（`build_verify_on_stop_nudge`）——改完不验证就停，可能交付一个坏的改动；已写的回答先暂存（#65919），万一验证轮预算耗尽还能兜底（#61631）。脚手架标记 `_verification_stop_synthetic` 可剥，验证完不留痕迹。

**趁没满先瘦身（proactive tool-result prune）**。大窗口模型离 50% 压缩阈值很远，但上下文已堆了不少没用的块头：旧工具输出压成一行话（几百行 → `[terminal] ran npm test -> exit 0, 47 lines`）、完全相同的工具结果只留一份、超大参数截短——纯规则、不用 LLM、免费。但有个"最小收益"门槛：清理后省得不够多就干脆不动——因为清理会打断缓存（上下文变了前缀缓存要重算），为小收益打断缓存不划算。

**外层兜底按异常来源分派**。主循环抛异常时按 traceback 模块判断：只经过本地处理模块（本地 bug）→ 不重试（每次都会同样失败，只烧预算）；进过 API 调用模块 → 走重试。所有路径（正常/中断/失败/空响应）最终都汇到 `finalize_turn` 统一出口——像所有顾客在同一个收银台结账。

## 关键机制/流程

响应归一化（三家 API 形状统一 + content 转纯字符串）→ 展示回答/转发子代理思考 → Codex incomplete 续写（≤3 次）→ 验证两关（名字 → 参数 JSON）→ 安全审查（delegate 上限、去重、housekeeping 分类决定是否静音）→ 先记账后执行 → `_execute_tool_calls` + guardrail 护栏 + 退预算 → post-tool 压缩（用真值 `last_prompt_tokens`，无真值退回估算）+ 主动瘦身 → 空响应 7 级处理 → 正常收尾（dropped tool-call 重试、脚手架清理、verify-on-stop 门、kanban 收尾）→ finalize_turn。

## 相关笔记

- hermes-conversation-loop-run-conversation.md
- hermes-error-recovery-phase.md
- hermes-tool-call-message-structure.md
- hermes-pre-api-pressure-check.md
- hermes-interrupt-partial-response-save.md
- hermes-message-sanitize.md
- hermes-compression-per-turn-state-reset.md
