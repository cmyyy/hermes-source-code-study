# 发请求前的最后一道防御带：消息消毒两层修复
> [← 返回笔记地图](../README.md#笔记地图) · 关键机制


> 源码位置：`agent/conversation_loop.py` → 发请求前的 sanitize 调用点；实现见 `agent/agent_runtime_helpers.py` → `sanitize_tool_call_arguments()`、`repair_message_sequence_with_cursor()`
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

"消毒"（sanitize）不是杀病毒，是**清洗数据**：发请求前把消息历史里两类脏数据修好，防止 provider 拒收或返回空响应。第一层修工具调用的参数 JSON（模型生成的 arguments 不是合法 JSON）；第二层修角色交替违规（消息顺序不符合 user/assistant/tool 交替规则）。

## 为什么这么设计

- **它是"防御带"，不是主防线**。正常情况下，前面的恢复路径（空响应脚手架剥离等）已经挡住了大部分脏形状。但**外部来源会直接喂坏历史**：gateway 多队列重放、会话恢复、cron 任务、宿主代码显式传入的 conversation_history——这些入口无法控制历史是怎么生成的。代码注释说得很清楚：This runs right before the API call as a defensive belt。所以 API 调用前做一道兜底检查，**不信任任何上游，发请求前最后把关一次**，是典型的防御式编程。

- **坏 JSON 会招来 400，修成 `{}` 至少请求能过**。模型偶尔生成残缺 JSON（截断、格式错误），带坏参数发出去 provider 直接 400 拒收。`sanitize_tool_call_arguments` 把 None / 空串 / 非法 JSON 一律改成 `{}`（合法空参数），并给对应的 tool 结果消息打上损坏标记（marker）——**请求能过，模型还能在下一轮看到"这参数坏了，我修成了空"，形成自我纠正的闭环**。

- **角色违规比报错更可怕：provider 会静默返回空内容**。连续两条 assistant、孤儿 tool 消息（tool_call_id 对不上任何前面的调用）、连续两条 user，这些违规序列在多数 provider 上**不报错，而是静默返回空内容**——空内容又触发空响应重试循环，形成死循环。代码注释原话：Violations cause silent empty responses on most providers, which triggers the empty-retry loop。所以修复规则是：连续 assistant 合并（tool_calls 并集 + 内容拼接）、孤儿 tool 丢弃、连续 user 换行合并（不丢用户输入）。

- **修复必须同步数据库写入游标**。`repair_message_sequence_with_cursor` 比裸函数多干一件事：同步修复持久化游标（`_last_flushed_db_idx`）。repair 删/并消息让列表变短，游标还指着旧位置的话，回合结束持久化会跳过 assistant/tool 链（#44837）——**修了内存里的数据，就得修指向数据的指针**，否则修复本身制造新 bug。这也是为什么循环里用的是"带游标版本"而不是裸的修复函数。

- **消毒在前、规范化在后，顺序就是流水线**。消毒先保证"合法"（能过 provider 的校验），随后的规范化再保证"稳定"（字节一致喂缓存）——先修对，再修齐。消毒修复时打上的损坏标记也会留在历史里，让模型的自我纠正闭环跨回合生效：这轮修掉一个坏参数，模型下轮就能看到"上次那个参数坏了"。

## 关键机制/流程

```
发请求前
  ├─ 第一层：sanitize_tool_call_arguments
  │    扫描所有 assistant 消息的 tool_calls
  │    arguments 为 None / 空串 / 非法 JSON → 改成 "{}" + 打损坏标记
  └─ 第二层：repair_message_sequence_with_cursor
        连续 assistant → 合并（tool_calls 并集 + 内容拼接）
        孤儿 tool 消息 → 丢弃
        连续 user → 换行合并
        → 同步 _last_flushed_db_idx 游标
```

## 相关笔记

- hermes-message-normalization（合法 JSON 重排成标准字节，消毒的下游）
- hermes-tool-call-message-structure（工具调用消息的合法结构基准）
- hermes-api-messages-build（api_messages 组装，消毒之后的格式转换层）
- hermes-steer-mechanism（/steer 搭便车注入，同为维护角色交替）
