# step_callback：Agent 每走一步的"进度广播"

> 源码位置：`agent/conversation_loop.py`（触发）→ `gateway/run.py` → `_step_callback_sync`（消费方）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

Agent 每走一步（每轮迭代），就对钩子系统喊一嗓子："我现在到第 N 步了，上一步我用了这些工具"。gateway 里装的钩子插件拿这个信号做进度播报、工具审计、风控。就像外卖 App 每到一个节点推送"骑手已取餐""骑手距您 500 米"——只有装了"跟踪骑手"的 App（钩子插件）才会收到推送。

## 为什么这么设计

**只在有人听时才发**。CLI 模式没有钩子系统，step_callback 是 None，整个逻辑直接跳过、零成本——因为倒着翻聊天记录找工具清单是有开销的，没人听就不白干。gateway 模式也只有装了钩子插件时才挂载（`loaded_hooks` 非空才给 agent 赋值回调）。

**每轮迭代喊一次，报告"上一批"工具**。回调挂在循环顶部、新一轮 API 调用之前，所以报告的永远是上一轮那批工具的执行情况（名字 + 参数 + 结果）。想看每个工具的实时细节？那是 tool_progress_callback（tool.start / tool.complete 事件）的活——step_callback 只管粗粒度心跳，两套互补。已知边界：工具执行后被 guardrail 拦截或中断时，最后一批工具会漏报（break 了，没有下一轮来报告），设计上可接受——它是进度心跳，不是执行审计。

**同步线程 → asyncio 事件循环的桥接**。agent 主循环是同步的，gateway 钩子是 asyncio 的：`_step_callback_sync` 经 `safe_schedule_threadsafe` 把同步调用投进事件循环异步执行，不阻塞 agent 循环；钩子挂了也只记 debug 日志（try/except 包裹），绝不影响主流程——**观测是旁路，不能是主路径的绊脚石**。

**三层触发门槛**：装了钩子（CLI 永远不触发）→ `_run_still_current()` 校验本轮运行未被 /new 或新消息抢占（不发过期信号）→ 每轮迭代调用一次。

**事件载荷的向后兼容设计**。`agent:step` 事件同时携带 `tool_names`（工具名列表，兼容老钩子）和 `tools`（含 name/arguments/result 的完整信息，给新钩子）——演进时老插件不破、新插件能用，这是钩子生态里很典型的兼容策略。

**一个具体的时间线**：迭代 1 模型调 search_files + read_file 并执行 → 迭代 2 开始前广播"第 2 步，上一步用了 [search_files, read_file]" → 迭代 2 调 write_file → 迭代 3 开始前广播"第 3 步，上一步用了 [write_file]"——广播永远滞后一步，但每批工具都不漏（除非那轮被 guardrail 或中断打断）。

**与其它钩子的分工**：`agent:step` 是每轮迭代的轻量心跳；`pre_llm_call` / `post_api_request` 是每次 API 调用的请求/响应细节（含 token 用量）；`event_callback` 管压缩、会话切换等生命周期事件——框架只定时机，行为完全由插件决定。

**回调签名为什么是两个参数**。`(iteration, prev_tools)`——迭代号回答"走到哪了"，工具清单回答"刚才干了什么"，刚好构成一句人话播报；不传消息全文、不传会话内部对象，让消费方（包括未来的第三方钩子）拿到的永远是稳定、轻量的数据，不会耦合 Hermes 内部结构。

## 关键机制/流程

主循环每轮迭代开始 → 倒序翻聊天记录找最近一条带 tool_calls 的 assistant 消息 → 连带捞取对应的 tool 结果 → 组装 `prev_tools` 清单（name / result / arguments）→ 回调（iteration, prev_tools）→ gateway 侧归一化成工具名列表 → 以 `agent:step` 事件投递（tool_names 兼容老钩子 + tools 完整信息）。

## 相关笔记

- hermes-conversation-loop-run-conversation.md
- hermes-loop-counter-init.md
- hermes-interrupt-partial-response-save.md
- hermes-loop-entry-phase.md
