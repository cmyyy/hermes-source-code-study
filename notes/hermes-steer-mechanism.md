# /steer：不打断 Agent 的"递纸条"机制

> 源码位置：`agent/conversation_loop.py`（pre-API drain）→ `agent/agent_runtime_helpers.py` → `apply_pending_steer_to_tool_results()`；`agent/agent_init.py`（_pending_steer 初始化）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

Agent 正在跑长任务（读文件、执行命令），你发现方向偏了但不想让它从头来——`/steer` 把一句话注入下一次工具结果里，模型下一轮迭代就能看到，但**当前这轮工具照常执行完**。不打断，只引导。

## 为什么这么设计

**与 interrupt 的本质区别**。interrupt 是"抢方向盘"：设 `_interrupt_requested` 立即取消当前工具批，中断后重新开始；steer 是"旁边递纸条"：不设中断标志，等当前工具批自然执行完，再把文本追加到最后一条 tool 消息的内容末尾，模型下一轮迭代看到提示自动调整。一个用于彻底停止/换方向，一个用于方向微调。

**搭便车注入，不新插 user 消息**。为什么不插一条新的 user 消息？会破坏角色交替——tool 与 user 之间不能插消息，严格 provider 会拒收。纸条"贴"在现有 tool 消息内容末尾（`existing + marker`）：角色没变、内容多几个字，消息结构始终合法。这是 Agent 系统处理"运行中用户输入"的优雅方案——新内容不新开消息，而是附在已有消息上，既传递信息又保持协议正确。

**marker 包装**。纸条不是裸文本，而是包了一层标记（`format_steer_marker`，如 `[User steer: 注意别改配置文件]`），让模型能区分"工具的结果"和"用户中途递的纸条"——避免把用户指示误当成工具输出。

**没有工具消息可贴就先等着**。首轮迭代还没有 tool 输出时，把纸条放回 `_pending_steer`（存储带锁防并发），等工具执行完再由 `apply_pending_steer_to_tool_results` 补贴——没有 tool 输出可搭便车时，宁可等也不破坏消息结构。

**"不设中断标志"是设计的核心**。一旦设了 `_interrupt_requested`，会触发一整套中断保存/恢复路径（半截内容抢救、redirect 特判等）——steer 刻意绕开这套机制：当前工具批照常执行、结果照常落库，唯一的副作用是下轮迭代的 tool 结果末尾多了一行提示。对会话状态零扰动，因此也零风险。

**两种 drain 时机互补**。pre-API drain 保证"只要上一轮执行过工具，这轮就一定能看到纸条"；工具执行后的 drain 兜住"首轮还没有 tool 消息"的场景——两处都用"追加到已有 tool 消息"这同一个手法，与 MoA 聚合上下文"拼 user 消息末尾"是同一类搭便车注入，说明这是 Hermes 处理"运行中附加信息"的统一套路。

**典型用法**：agent 正在批量改文件，你发现它顺手动了配置文件——发 `/steer 注意别改配置文件`，当前这批文件改完，下一轮迭代它就收敛方向；方向错得离谱才用 interrupt 全部重来。两者也可以组合：先 interrupt 打断，再在新回合里用 steer 给一句引导。

## 关键机制/流程

存纸条（`steer()` 写入 `_pending_steer`）→ 找时机（主循环 pre-API drain 取出）→ 贴上去（倒序找最后一条 tool 消息，内容末尾追加 marker）→ 模型看到（下一轮 API 调用读到提示并调整）。首轮无 tool 消息则放回 pending，工具执行后补贴；配套的工具执行后 drain 走同一套贴纸条逻辑。

## 相关笔记

- hermes-conversation-loop-run-conversation.md
- hermes-message-sanitize.md
- hermes-moa-context-injection.md
- hermes-interrupt-partial-response-save.md
- hermes-loop-entry-phase.md
