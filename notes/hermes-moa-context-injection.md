# MoA 圆桌会议：把参考模型的答案拼进用户消息
> [← 返回笔记地图](../README.md#笔记地图) · 关键机制


> 源码位置：`agent/conversation_loop.py` → MoA 聚合上下文注入段；实现见 `agent/moa_loop.py` → `aggregate_moa_context()`
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

MoA（Mixture of Agents，多模型协作）回合在发聚合请求前，先让所有参考模型对同一问题各答一遍，把答案汇总成"参考资料"，拼到用户消息末尾——聚合模型带着"用户问题 + 多家参考答案"给出最终回答。一句话：**开会之前，先让每个参会者提交自己的意见**。

## 为什么这么设计

- **MoA ≠ MoE，这是应用层的协作而非架构层的路由**。MoA 是应用层设计：参考模型各答一遍 + 聚合模型综合，花钱但灵活——参考模型和聚合模型可以任意换 provider。MoE（混合专家）是模型内部的专家路由。Hermes 把 MoA 做成应用层特性，意味着用户可以用便宜的参考模型凑意见、用强模型做综合，按需组合。

- **并行 fan-out 省时间，且中断可提前中止**。`aggregate_moa_context` 内部用 `_run_references_parallel` 并行跑所有参考模型，而不是串行等一个接一个；同时把 `agent` 引用传进去——用户中断时能提前中止参考模型的 fan-out，不浪费 token 等超时。参考模型的 `max_tokens` 单独限制（`reference_max_tokens`），**聚合器永不限 max_tokens**（#53580）：给聚合器也限流会截断长综合，参考模型限流则只影响草案质量。参考模型超时 `reference_timeout` 未显式设置时继承 `auxiliary.moa_reference.timeout` 配置——**回合级临时参数优先，缺省回落到全局配置**，和聚合失败降级的思路一致：能拿到配置就用，拿不到就按默认走。

- **搭便车注入，保持角色交替**。汇总文本不新开消息，而是**追加到最后一条 user 消息的 content 末尾**——纯文本直接拼接，多模态（图+文）则追加一个 text 块。原因和 /steer 一样：新开 user 消息会破坏角色交替，严格 provider 会拒绝。参考模型的输出在拼接前还过了两道保护：`flatten_message_text` 只取可见文本（防止 base64 图片泄进聚合器 prompt），`moa.privacy_filter: full` 时对参考输出脱敏。

- **失败降级不崩溃，是整段设计的基调**。参考模型全挂 → 聚合器直接不调用（避免零建议还烧 token 等超时）；部分挂 → 按 `degraded_reference_policy`（默认 "loud" 显式报错）生成降级说明随上下文给聚合器；聚合过程本身抛异常 → 只记 warning、**降级为普通单模型回答**。注释原话：Failures are returned as model-specific notes instead of aborting the normal agent loop; the main model can still act with partial context。**多模型协作是增强，不是主路径的依赖**——参考模型全挂，用户的问题依然能答。

- **非 MoA 回合零开销**。`if moa_config:` 保证普通回合跳过整段；`from agent.moa_loop import ...` 是函数内延迟导入，非 MoA 回合连模块都不加载。**增强功能的成本只在被使用时发生**——这是给主循环加可选特性的标准姿势。

## 关键机制/流程

```
用户问题
  → aggregate_moa_context（并行跑所有参考模型）
  → 过滤 disabled 模型、成功/失败分类、可选隐私脱敏
  → 拼成段落：Reference 1 — deepseek: ... / Reference 2 — kimi: ...
  → 追加到最后一条 user 消息末尾（多模态追加 text 块）
  → 聚合模型带着"用户问题 + 参考答案"综合出最终回答
```

## 相关笔记

- hermes-loop-entry-phase（MoA 魔数前缀解码：回合入口认出圆桌请求）
- hermes-steer-mechanism（同为搭便车注入保角色交替）
- hermes-api-messages-build（api_messages 组装，本段在其后追加）
- hermes-conversation-loop-run-conversation（主循环总纲）
