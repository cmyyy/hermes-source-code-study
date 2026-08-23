# 发请求前的压力检查：用估算抓住工具结果带来的暴涨

> 源码位置：`agent/conversation_loop.py` → pre-API 压缩压力检查段（压缩四道闸的第二道）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

每次发请求前，用本地估算把"当前这轮组装好的完整请求"（消息 + 工具 schema）算一遍 token 压力，超阈值就先压缩再发——**不等 provider 报错**。它是压缩体系四道闸中的第二道，也是唯一卡在"组装完、未发出"这个时间点的检查。

## 为什么这么设计

- **"实时且准确"的 token 数不存在——真值必然滞后**。这是整段设计的核心认知。两个数据源各有缺陷：真值（`last_prompt_tokens`，API 响应里 provider 报告的实际 token 数）准确，但**只在 API 调用时产生**，两次调用之间没有任何更新机制；估算（`estimate_*_rough`）随时能算，但有噪声（#36718 高估）。**工具结果 append 之后，没有任何 API 会报告新的数字**——想要实时只能本地估算。真值滞后不是设计选择，是数据来源决定的。

- **三处检查分工，覆盖不同时间点**。回合开始的 preflight（估算，此时还没执行工具）、每轮迭代的 pre-API（估算，看到刚 append 的工具结果）、工具执行后的 post-tool（真值，只看到上一轮 API 报告的数字）。推演一下就明白为什么 pre-API 必不可少：迭代 1 估算 100K 放行 → 调 API → 模型要工具 → 执行 → 结果 append（消息变 150K）→ 迭代 2 重新估算 150K 超阈值 → 压缩。**post-tool 用"准但旧"的真值看不到这轮暴涨，pre-API 用"实时但糙"的估算才能看见**——真实事故是 271k/272k 的 Codex 失败，暴涨没被任何检查看到。两者互补，缺一不可。

- **七道门槛防误压**。触发压缩前要依次过：压缩开启、有历史（len(messages) > 1）、共享上限内（与溢出处理器共用 `compression_attempts`）、未被"压缩无效"标记、估算可信（#36718 噪声保护）、不在压缩失败冷却期、确实超阈值（`should_compress` 的阈值计算还预留了输出空间——压缩后还要留够模型把回答写完的余量，避免压完立刻又超；另含摘要 LLM 冷却与防抖 #11529）。**压缩是有成本的（烧 token 重写历史），宁可少压不可乱压**。

- **进展判断防空转**。压过一次压力没降多少 → `_preflight_compression_blocked = True`，停止主动压缩——压缩无效就不要再空转烧钱。但 provider 真正证明放不下时（413/溢出错误），**错误处理器仍可压缩**：更强的信号兜底，层层递进。配合 `_last_preflight_pressure` 记录"上次压缩前压力量了多少"，压缩有无进展一目了然。

- **post-tool 只看 prompt_tokens 不算 completion**（#12026）。推理模型（GLM-5.1 / QwQ / R1）的 completion 被推理内容撑爆，把 completion 也算进去会过早压缩；真值缺失时（0/-1）fallback 到估算（#2153 / #14695）。**"准但旧"和"实时但糙"各管一段**，是这个检查体系的总原则。

## 关键机制/流程

```
每轮迭代组装完请求、未发出
  → 七道门槛全过？
      是 → compression_attempts += 1 → 压缩 → 重新估算
            → 有进展？重来 / 无进展？置 blocked 停主动压缩
      否 → 直接发出
  → 仍超阈值？留给 provider 报错，错误处理器兜底压缩
```

## 相关笔记

- hermes-compression-per-turn-state-reset（压缩状态 per-turn 归零）
- hermes-loop-entry-phase（压缩四道闸的完整视野）
- hermes-message-normalization（token 记账与字节稳定）
- hermes-conversation-loop-run-conversation（主循环总纲）
