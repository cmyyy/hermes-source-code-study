# Hermes 主循环：run_conversation 全解

> 源码位置：`agent/conversation_loop.py` → `run_conversation()`
> 研读版本：hermes-agent v0.19.0（commit `d71033a40`）
> 配套：入口转发器见 `hermes-run-conversation-forwarder.md`

## 这个函数是什么

**Hermes 每个 Agent 回合（turn）的完整生命周期**——用户说一句话，从"理解请求"到"给出回答"的全部过程，都在这一个函数里。

## 它为什么有 5800 行

因为真实世界的 Agent 不是"把消息发给模型、拿回回答"这么简单。拆开看，代码里混着**三层逻辑**：

```
第一层 主流程：调 LLM、处理工具调用（用户看得见的"正常流程"）
第二层 容错恢复：重试、换模型、压缩历史、等限流（占超过一半代码！）
第三层 旁路钩子：插件、MoA 多模型协作、代码验证、kanban 任务（扩展能力）
```

**读的时候分清这三层，就不会被淹没。** 这也是"生产级 Agent 和玩具 demo 的分水岭"——玩具 demo 只有第一层，生产系统 2/3 的代码在回答"万一出错了怎么办"。

## 六个阶段（一个回合的骨架）

```
A 回合启动与迭代准备 → B 请求组装 → C 请求执行 → D 响应处理
  →（要工具就执行工具，回到 C）→ F 工具执行与收尾
        ↖ 出错了进 E 错误恢复，再回 C
```

| 阶段 | 一句话 | 关键设计决策 |
|------|--------|-------------|
| **A 回合启动** | 开会前的全部准备：解码特殊请求（MoA）、清空上一场状态、准备上下文 | 状态归零（gateway 复用 agent 对象，不清会串场） |
| **B 请求组装** | 把问题整理成模型看得懂的格式：消毒、组装、注入上下文、估算字数 | 守护 prompt cache（字节一致=便宜 90%）和角色交替（tool→assistant→user 顺序） |
| **C 请求执行** | 真正发给模型，等回答 | 默认流式（非流式会因 SSE 假连接挂起） |
| **D 响应处理** | 检查回答：格式对吗、被拒了吗、说到一半断了吗 | 按 finish_reason 分派（内容拒绝不重试，重试=烧钱复现） |
| **E 错误恢复** ★ | 出错了怎么办（占 2/3 代码，面试重点） | 按错误类型恢复：401 刷新凭据、429 等限流、溢出压缩历史 |
| **F 工具执行与收尾** | 模型要调工具就执行，回答完就存档 | 执行前先持久化（防工具跑了但没存档）、空响应 7 级处理阶梯 |

## 五个最值得讲的设计点

### 1. 三层循环（对应"回合边界、调用边界、错误类型"）

```
外层 while：整个回合（最多 N 次迭代 + 预算 + 宽限）
中层 while：单次 API 调用重试（网络抖动、临时 429）
内层分派：按 _retry 标志决定怎么恢复（换模型？压缩？等退避？）
```

**为什么好**：每个"失败"发生在哪一层，就在哪一层处理——不会因为一次网络抖动就把整个回合推倒重来。

### 2. 压缩的四道闸（上下文超了怎么办）

```
① 回合开始 preflight（prologue 检查）
② 发请求前压力检查（实时估算，超阈值先压缩）
③ 工具执行后压缩（拿到真实 token 数再判断）
④ 溢出错误触发的恢复压缩（兜底）
```

**为什么好**：不等到"爆了"才处理——四道闸在不同时机拦截，每道都比上一道更接近真实情况。

### 3. prompt cache 守护（省钱的设计灵魂）

Hermes 的 AGENTS.md 第一条就是前缀稳定。体现在：
- api_messages 重放历史字节（缓存命中靠字节一致）
- 消息规范化：arguments JSON 排序重排（保证字节稳定）
- Anthropic cache_control 最后打标（显式断点，0.1x 价格）
- system prompt 每会话只建一次（缓存锚点）

**为什么好**：缓存命中省 ~90% 成本——这是"把工程细节当设计原则"的典型。

### 4. 角色交替是硬约束（维护协议正确性）

模型 API 要求消息按 tool→assistant→user 严格交替。Hermes 所有恢复路径都在维护它：
- 空响应 → nudge（提示模型继续）
- 工具 JSON 损坏 → 注入 tool error 消息（保持交替）
- steer 纠错 → 追加到 tool 消息末尾（不破坏顺序）

**为什么好**：破坏交替会被严格 provider 拒绝——这是"协议正确性"的系统级守护。

### 5. 引用真实 issue（证明是深度研读）

笔记里的问题编号都是真实的：`#38763`（in-place 压缩）、`#44837`（修复交替）、`#62625`（静默溢出）、`#32421`（流式内容过滤）……遇到"为什么这么设计"，去 GitHub 搜对应 issue 能看到完整讨论。

## 怎么继续深入

- **阶段细节**：`hermes-loop-entry-phase.md`（A）｜`hermes-request-assembly-extras.md`（B）｜`hermes-request-execution-phase.md`（C）｜`hermes-response-handling-phase.md`（D）｜`hermes-error-recovery-phase.md`（E）｜`hermes-tool-execution-finalize-phase.md`（F）
- **专题深挖**：`hermes-build-turn-context.md`（上下文怎么组装）｜`hermes-message-normalization.md`（字节稳定）｜`hermes-moa-context-injection.md`（多模型协作）｜`hermes-anthropic-cache-control.md`（缓存打标）
- **概念**：`moa-mixture-of-agents.md` ｜ `message-role.md` ｜ `prompt-cache-prefix-stability.md`
