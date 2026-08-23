# 认证池与回合级状态归零：多把钥匙轮着用，还不怕死循环

> 源码位置：agent/conversation_loop.py → run_conversation（每回合开头）；实现 agent/credential_pool.py
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

**认证池（credential pool）= 同一 provider 的多套凭据（API key / OAuth token）的集合，按优先级排队、自动轮换使用。** 场景：给某个 provider 配了多个账号的 key，或一个账号多个 OAuth token。单 key 有速率限制（429）或额度（billing）上限，用完就废；认证池解决"多把钥匙轮着用"：`select()` 按优先级和可用性选当前凭据，`mark_exhausted_and_rotate()` 在限流/超额度时标记冷却并换下一个，`try_refresh_current()` 在 OAuth 过期时用 refresh token 换新的。Java 类比就是连接池（HikariCP）——只是管的是"钥匙"不是"连接"。

回合开头有两行归零代码，分别处理两个隐患：认证池刷新计数账本清空、`_last_turn_usage` 置 None。

## 为什么这么设计

- **刷新计数防"无限成功"死循环（issue #26080）**。如果上游 401 是永久性的（账号被 ban、token 被 revoke），`try_refresh_current()` 会"永远成功"——拿到新 token，但新 token 也永远 401。单凭据池（single-entry pool）遇到这种情况就是刷新 → 重试 → 再刷新 → 再 401 的无限空转。解法：记录同一凭据的**连续成功刷新次数**，有上限，超过后不再死磕，让 fallback 链接管（换 provider）。注释原文点明了这个场景。

- **`_last_turn_usage` 置 None 防陈旧数据**。这个字段存"本回合最后一次成功 API 响应的 token 用量"，回合结束转发给 context engine 的 `on_turn_complete()` 观察钩子。置 None 的意义：本回合**从未成功到达响应**（早期失败/中断）时，钩子收到 None 而不是上一回合残留的旧用量——**防 context engine 把上一回合的数据当这一回合的**。

- **"局部变量 vs 属性"的边界纪律**。这两行属于"必须放对象上"的状态（跨函数共享：context engine 钩子、错误处理路径都要读），所以采用"属性 + 回合边界显式归零"的防守写法。对比同段代码里 16 个局部变量：**能局部表达就用局部变量（机制上天然 per-turn），必须跨函数共享才放对象上并显式归零**。这是 Hermes 反复出现的"状态边界归零"模式的两个具体实例之一。

## 关键机制/流程

认证池核心概念：`PooledCredential`（一条凭据：id/provider/token/优先级/状态/冷却截止）、`CredentialPool`（池本身：线程安全锁、按 priority 排序、活动租约）、exhaustion cooldown（凭据被打爆后进入冷却期，冷却期内不选）、pool strategy（不同 provider 的轮换策略可不同）。

## 相关笔记

- compression-per-turn-state-reset（压缩三标志重置，同一模式）
- interim-text-dedup（过程旁白去重，同一模式）
- loop-counter-init（循环计数器，对比局部变量与属性）
- context-engine-selection（on_turn_complete 的用量消费方）
- error-recovery-phase（凭据旋转的恢复场景）
