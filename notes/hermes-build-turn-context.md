# build_turn_context：把"每回合一次的会前准备"从主循环里抽出来
> [← 返回笔记地图](../README.md#笔记地图) · 关键机制


> 源码位置：agent/turn_context.py → `build_turn_context()`（返回 `TurnContext` 对象）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

主循环 run_conversation 里，每个回合开始前都有一堆"setup"要做：stdio 守护、重试计数清零、用户消息清洗、todo/nudge 注入、system prompt 恢复或构建、preflight 压缩检查、pre_llm_call 插件钩子、外部记忆 prefetch、崩溃韧性持久化。`build_turn_context` 把这一整串从主循环里抽出来，集中执行，最后返回一个 `TurnContext` 对象——它只携带**主循环接下来要读的局部变量**，run_conversation 解包后就是循环的输入。这是 god-function 分解的典型手法：**setup 与 loop 解耦，结果显式传递**。

TurnContext 的 12 个字段可以分成四组：4 个输入类（user_message、original_user_message、messages、conversation_history）+ 3 个身份类（active_system_prompt、effective_task_id、turn_id）+ 1 个定位类（current_turn_user_idx）+ 4 个旁路状态类（should_review_memory、plugin_user_context、ext_prefetch_cache、preflight_compression_blocked）。

## 为什么这么设计

- **消息双轨制：工作版 vs 存档版**。`user_message` 是加工后的工作版——剥掉孤立代理字符后，还可能被注入记忆 prefetch、插件上下文、/steer 备注；`original_user_message` 永远不加任何注入，专供写 transcript、存记忆、做检索查询。**存档不能掺假**——记录进记忆库的必须是用户原话。

- **conversation_history 是"防重复写库"的开场种子**。`/resume` 从数据库加载消息时，这些消息是新的 dict 对象、没有持久化标记，如果不告诉 flush"它们是历史"，就会被当成新消息重复写库。conversation_history 记录"回合开始时已存在的消息集合"，flush 时给这些消息批量盖上 `_DB_PERSISTED_MARKER` 跳过。它与标记的分工：**种子管开场已有的，账本管过程新增的**（issue #860）。以前用位置切片判断，但消息修复会删/并消息让列表变短、切片失效，已交付的回答永远进不了库（issue #46053）——所以改成按身份标记。压缩后旧基线失效，重置为 None 重新记账。

- **回调函数当参数传，打破循环依赖**。turn_context 模块不能 import conversation_loop（会循环依赖），所以 restore_or_build_system_prompt、install_safe_stdio、sanitize_surrogates 这些函数由调用方显式传入。依赖倒置：**框架只认接口，不认实现来源**。

- **入口剥离孤立代理字符**。Unicode 代理区（U+D800-DFFF）的单个"半成品"码点（emoji 截断、富文本粘贴产生）会崩 json.dumps、污染存储，在入口直接删除而非替换。连 API key 里的非 ASCII 都可能被清——坏粘贴的密钥是持久 UnicodeEncodeError 的主因（issue #6843）。

- **预取只跑一次，结果跨迭代复用**。`ext_prefetch_cache` 是外部记忆（honcho/mem0 等）针对本轮问题的预取结果，注释明确 reused across loop iterations——查询一次，循环每次构造请求都复用。

## 关键机制/流程

12 字段记忆口诀：4 输入 + 3 身份 + 1 定位 + 4 旁路。`preflight_compression_blocked` 标记"预检压缩已尝试且无进展"，主循环读到就不再空转重试；`current_turn_user_idx` 定位本轮 user 消息，压缩后索引失效需重锚。

## 相关笔记

- conversation-loop-run-conversation（主循环，调用点）
- loop-counter-init（循环计数器初始化）
- api-messages-build（sidecar 的产生与消费）
- agent-init-refactoring（_ra 延迟引用，同类依赖问题解法）
