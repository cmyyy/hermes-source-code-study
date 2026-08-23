# 压缩状态回合级归零：三行代码防一场数据损坏

> 源码位置：agent/conversation_loop.py → run_conversation（每回合开头）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

上下文超窗压缩时，Hermes 有两种模式：

1. **Rotation（会话轮换，旧模式）**：压缩后换一个新 session_id，旧会话归档。gateway 检测到"session_id 变了"就知道发生过分流，从而重置 transcript 处理。
2. **In-place（就地压缩，issue #38763，当前默认）**：压缩后 session_id 不变，直接在同一个会话里把历史软归档（active=0）并原子插入摘要。gateway 光靠"id 变了"检测不到——所以需要另一个信号告诉它"其实发生了压缩"：`_last_compaction_in_place` 就是那个"轮换无关的压缩信号"。

每回合开头有三行代码，把上一回合残留的压缩状态标志全部归零。看起来不起眼，缺了它会造成数据损坏级的 bug。

## 为什么这么设计

- **in-place 成为默认，因为 rotation 是一整簇 bug**。代码注释直接称之为 "session-rotation bug cluster"：换 id 导致会话链断裂（一个对话变两个会话，恢复要跨 id 追踪 parent 链）、网关误判（手动 /new、/resume、rewind 也变 id，靠 id 变化判断分流必然误判）、会话命名被重新编号（历史锚点丢失）、压缩锁按 session_id 键控导致锁语义漂移、transcript 要重建。**in-place 在同一 id 内软归档，会话不换 id、命名不变、锁语义稳定**，代价只是多一个"压缩发生"的信号让 gateway 感知。

- **gateway 跨回合复用 agent 对象，per-turn 状态必须显式归零**。agent 对象缓存在 gateway 进程里，回合结束不销毁，下回合接着用；而压缩状态是 per-turn 的。事故链：回合 A 发生 in-place 压缩（标志被写入 True）→ 回合 B 完全没压缩 → 标志没被清掉 → gateway 读到 True → 误判"B 也压缩了" → 一份**正常的、未压缩的 transcript 被当成压缩过的重写**（history_offset=0 + 重写 JSONL）。三行归零保证：压缩真的发生时值会被重新写对，没发生时保持"干净"。

- **三标志分工：运行级粗信号 + 尝试级细信号**。`_last_compaction_in_place` 是运行级信号（最近一次压缩是否 in-place）；`_last_compression_attempt_recorded` 和 `_last_compression_attempt_in_place` 是"本次尝试"级别的细粒度信号（尝试是否已记录、具体是 in-place 还是 rotation）。读取方 `conversation_history_after_compression()` 优先读细粒度，回退读粗信号——两层信号互相兜底。

## 关键机制/流程

工程素养点：**跨回合复用的对象，所有 per-turn 状态必须在回合边界显式归零**——等价于 Java 线程池复用线程时 ThreadLocal 必须 remove 的坑。这是 Hermes 里反复出现的同一个模式（见相关笔记）。

## 相关笔记

- credential-pool-turn-reset（认证池刷新计数归零，同一模式）
- interim-text-dedup（过程旁白去重，同一模式）
- loop-counter-init（循环计数器初始化，对比局部变量与属性）
- context-engine-selection（另一条上下文管理机制）
- pre-api-pressure-check（压缩四道闸之一）
