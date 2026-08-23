# 错误恢复：生产级 Agent 与玩具 demo 的分水岭

> 源码位置：agent/conversation_loop.py → run_conversation（错误恢复段）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

错误恢复占了整个主循环约三分之一的代码，只回答一个问题：**出错了怎么处理才不浪费钱、不死循环、不误判**。玩具 demo 只会 try/except 重试三遍；生产系统三分之一的工作量在给每个错误类型设计正确的恢复路径。

它与前置的响应处理是同一个中层 retry 循环的两个失败入口：一个管"响应回来了但内容无效"（HTTP 200，形状不对/被拒/截断），一个管"请求直接抛异常"（网络/401/413/429）。两者共用重试计数和退避，判断依据不同：**一个看响应内容，一个看异常类型**。

## 为什么这么设计（三个贯穿原则）

- **错误分类先行**。`classify_api_error` 把异常分成四类：可重试 / 该压缩 / 该换凭据 / 该切 fallback，后续分支都以它为判断依据。不分类就盲目重试是玩具 demo 的做法——把"该换凭据"当"可重试"只会反复烧预算。
- **确定性错误不重试**。内容安全拒绝、本地 bug，重试只会同样失败，直接终态或切 fallback，不浪费一次请求。
- **恢复动作 one-shot 防重入**。刷新凭据、改请求、重建客户端都有副作用，无限重试会死循环或烧钱。每种恢复（13+ 个 `_retry.xxx_attempted` 标志）**只允许触发一次**，不行就交给下一层（fallback / 终态）。

## 关键机制/流程

- **通用异常处理**：UnicodeEncodeError（剥离孤立代理字符，最多两轮，连 API key 里的非 ASCII 都清，issue #6843）、图片拒绝（剥图转纯文本）、Bedrock 流失败（切原生 Converse API），然后才做错误分类。
- **定向恢复**：billing+Nous 刷新付费凭据、429 旋转凭据池、图片过大缩小、多模态内容被拒降级文本（issue #27344）、各家 401 各自刷新、thinking 签名失效剥 reasoning、llama.cpp grammar 拒绝剥 pattern……均带防重入标志。
- **尊重用户显式选择**：`compression.enabled: false` 时溢出错误不得绕过设置偷偷压缩（issue #30749）；output-cap 错误不涉及压缩，照常重试。
- **溢出错误分两类**：`"Prompt too long"`（输入超窗）→ 减 context_length + 压缩；`"max_tokens too large"`（输入+输出超窗）→ **只减输出预算、不动 context_length——压缩无用还会死循环，fail-fast 才对**（issue #55546）。压缩重启必须重锚 current_turn_user_idx（索引已失效）。
- **限流分级**：429/billing 立即切 fallback（主 provider 短时间不会恢复）；传输错误先重试 1 次再切；退避尊重 Retry-After 头、封顶 600 秒（Anthropic Tier1 重置约 171s，上限太短会提前重试，issue #26293）。Nous 场景用 429 的 `x-ratelimit-*` 头区分真限流（记入跨会话断路器）与上游容量不足（普通重试）。
- **压缩救不了就明说**：GitHub Models 免费层硬顶 8K token，提示不兼容，避免 3 次无效压缩循环；压缩锁被占是"临时让路"不是"耗尽"（issue #69870）；错误回合不写库，防会话增长循环。
- **重试耗尽三步收尾**：重建主客户端一次（stale 连接池/TCP reset，仅一次）→ 再试 fallback → 终态（failure_reason 供 kanban worker 区分、按类型行动指南、错误摘要，防 Cloudflare 60KB 页面泄漏成 31 条消息）。
- **统一分派**：所有恢复路径都通过 `_retry.xxx` 标志表达"怎么重启"，末尾统一消费——redirect（退预算减计数）、压缩（退预算+重锚索引）、fallback 回滚、length 续写（max_tokens 按 2^retries 提升、封顶 32768）。

## 相关笔记

- response-handling-phase（前置阶段：响应内容无效的处理）
- pre-api-pressure-check（压缩四道闸）
- credential-pool-turn-reset（凭据旋转的池实现）
- interrupt-partial-response-save（退避中的中断响应）
- conversation-loop-run-conversation（主循环全解）
