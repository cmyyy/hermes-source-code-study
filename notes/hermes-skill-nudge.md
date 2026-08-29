# skill nudge：怎么让 Agent 自己想起"该沉淀经验了"
> [← 返回笔记地图](../README.md#笔记地图) · 故障排查与实战


> 源码位置：`agent/conversation_loop.py`（计数）→ `agent/turn_finalizer.py`（检查与触发）→ `agent/tool_executor.py`（清零）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

Hermes 的卖点之一是"Agent 能自己沉淀 skill"，但模型不会主动想起来"该创建 skill 了"。skill nudge 是一个计数器：每轮工具迭代 +1，攒够阈值（默认 10）就在回合结束后台提醒一次——"这轮任务有没有值得沉淀成 skill 的经验"。**用行为引导代替硬性要求**，软提醒推动 agent 养成"干完活沉淀 skill"的习惯。

## 为什么这么设计

**计数器 + 阈值做软提醒**。模型没有"自我反思该沉淀了"的主动性，但它的行为会暴露信号：干了很多轮工具迭代（≥10）还没碰过 skill_manage，大概率有可沉淀经验，值得提醒；刚用过 skill_manage 说明已经在管 skill，计数器清零重新数（注释原话 "Counter resets whenever skill_manage is actually used"）。提醒本身是**后台异步**的——回复送达之后才 spawn 后台审查，不打断用户任务。

**检查放在回合收尾而非主循环中间**。计数器在主循环每轮迭代时递增，但判定放在 `turn_finalizer` 的回合收尾处——一个回合结束时才评估"这一轮整体值不值得沉淀"，时机正好对应后台 review 的粒度。

**开关与作用域设计**。`_skill_nudge_interval` 默认 10，由 config 的 `skills.creation_nudge_interval` 驱动，设 0 即关闭；后台 review fork 会强制关闭（review 自己不能再触发 review）。工具集里没有 skill_manage 时也不计数——计数是为了提醒用它，没有它就没意义。

**为什么是"轮次"而不是"时间"**。工具迭代次数是工作量的真实信号：一轮迭代 = 一次 API 调用 + 一批工具执行，攒到 10 轮说明这轮任务已经足够复杂、大概率产生了可复用的经验；而墙钟时间与工作量无关。计数放在 `_iters_since_skill` 这个 agent 属性上，随 agent 生命周期存在——它是"跨回合保留、用后归零"的典型状态，与循环计数器是同一类管理问题。

**提醒是引导不是强制**。整个机制不阻塞主流程：计数只是一行增量，检查放在回合收尾，审查 spawn 成后台任务——用户看到的是回复先送达、skill 沉淀建议随后悄悄出现。这背后是 Hermes 的产品哲学：**skill 不是用户手动创建的，是 agent 被引导着自己沉淀的**——curator 技能生命周期（创建 → 使用 → 修正 → 淘汰）由 nudge 这件小事启动。

**nudge 与 review 的分工**。nudge 只负责"什么时候提醒"——一个计数器加一个阈值判断，逻辑极简；"这轮经验到底值不值得沉淀、怎么沉淀"的质量判断完全交给后台 review 流程。触发机制与审查逻辑分离，各自可以独立演进。

## 关键机制/流程

闭环四步：计数（主循环每轮迭代 `_iters_since_skill += 1`）→ 检查（回合收尾 finalizer 判断是否 ≥ 阈值，达标置 `_should_review_skills` 并清零重新数）→ 后台审查（`_spawn_background_review(review_skills=True)` 异步评估这轮有没有值得沉淀的经验）→ 清零（agent 真正用了 skill_manage 时归零）。

整套机制零配置可用（默认值即开），也允许高级用户通过 config 调阈值或关闭；配置为 0 则完全静默，适合不需要 skill 自动沉淀的受限场景。

## 相关笔记

- hermes-conversation-loop-run-conversation.md
- hermes-loop-counter-init.md
- hermes-loop-entry-phase.md
