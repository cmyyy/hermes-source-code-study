# 入口转发器：run_agent.py 为什么只是个"跳板"
> [← 返回笔记地图](../README.md#笔记地图) · 专题深挖


> 源码位置：`run_agent.py` → `run_conversation()`（转发器）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

Hermes 的 Agent 主循环真正实现在 `agent/conversation_loop.py` 的 `run_conversation()`，而 `run_agent.py` 里的同名函数是个**转发器（forwarder）**：入口不干活，只负责跳转，以及会话上下文和记账的初始化与清理。外部调用方（cli.py、gateway、子代理）看到的入口签名不变，内部实现却可以独立演进。

## 为什么这么设计

**转发器模式（门面/防腐层）**。run_agent.py 约 1.2 万行，把主循环实现下沉到独立模块后，入口只留薄壳：调用方不用改代码，实现代码可独立测试、独立演进。注释第一行 "Forwarder — see `agent.conversation_loop.run_conversation`" 即此意。接口稳定、实现可换，是经典的门面模式。

**函数内延迟导入**。`from agent.conversation_loop import run_conversation` 写在函数体里而非文件顶部——run_agent.py 被 import 时不用连带加载 conversation_loop 的依赖树，加快启动、避免循环依赖。

**ContextVar + token 管理会话身份**。会话 ID 和 SQLite 记账句柄通过 `set_conversation_context` / `set_accounting_context` 塞进 Python ContextVar（线程/协程隔离的全局变量），本轮内所有 LLM 调用（主循环、压缩、视觉、web_extract、session_search、MoA、子代理 fork）都能读到"我是哪个会话"——Nous Portal 据此打 `conversation=<root>` 标签，token 用量记账到 session_model_usage（#23270 修复辅助调用 token 在分析面板不可见的问题）。set 返回 token，finally 里 `reset(token)` 精确还原调用前的值，防止记账状态泄漏到下一个会话或复用线程。

**with + finally 双保险**。`with bind_subagent_parent(self), scoped_runtime_main({})` 管资源生命周期（标记当前 agent 为父代理供子代理回溯、注入运行时主配置作用域，退出时自动清理）；finally 管上下文还原。异步环境下这是防状态串扰的标准做法。

**为什么是 ContextVar 而不是普通全局变量**。异步环境下主循环、压缩、子代理 fork 可能跑在不同线程/协程里，普通全局变量无法区分"这是哪个会话的请求"；ContextVar 天然线程/协程隔离，每个执行上下文读到自己的值。token 复位是配套的关键：gateway 复用 agent 对象跑多轮会话，若退出时不 reset，上一场的会话标签和记账句柄会泄漏给下一场——这正是"finally 精确还原"要防的状态串扰。

**薄壳入口的额外收益**。转发器让主循环实现可以整体替换而不动调用方（后续重构、A/B 实现都可行），也让测试可以方便地 mock 掉真实实现；`moa_config` 这类新参数只需在入口透传，不需要各调用方逐一适配——接口稳定、演进自由，这正是"防腐层"的价值。

**如果没有这层转发**。各调用方就得直接 import `conversation_loop`——既把庞大的依赖树拖进每个入口模块（启动变慢），也让"会话身份怎么初始化"散落到各处，稍有遗漏就会串会话。转发器把"初始化 → 跳转 → 清理"收敛成唯一入口，是典型的职责单一化。

## 关键机制/流程

延迟导入 → 发布会话/记账上下文（ContextVar）→ with 包装（子代理父绑定 + 运行时作用域）→ 参数原样透传调用真实实现 → finally reset token 还原上下文。

整个过程刻意不碰任何业务逻辑——入口的职责边界就是"上下文就位 + 跳转 + 清理"三件事。

## 相关笔记

- hermes-conversation-loop-run-conversation.md
- hermes-build-turn-context.md
- hermes-agent-init-refactoring.md
- hermes-loop-entry-phase.md
