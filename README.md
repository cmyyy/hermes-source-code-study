# Hermes Agent 源码研读笔记

> 深度研读 [Hermes Agent](https://github.com/NousResearch/hermes-agent)（22 万+ star 的开源 AI Agent 框架）核心源码，沉淀为面向读者的设计解读。

## 为什么读 Hermes

Hermes 是一个**生产级 Agent 框架**：它的价值不在"能调 LLM"，而在**把 Agent 从 demo 变成可靠系统**的系统性设计——错误恢复占了主循环一半以上代码、压缩四道闸、prompt cache 守护、角色交替硬约束。读它的源码，比读任何 Agent 教程都更接近"生产级 Agent 到底怎么设计"。

## 研读方法

1. **以 run_conversation 为主轴**（约 6675 行的 god function，一个 Agent 回合的完整生命周期）
2. **六阶段拆解**：回合启动 → 请求组装 → 请求执行 → 响应处理 → 错误恢复 → 工具执行与收尾
3. **分层读**：主流程 / 容错恢复 / 旁路钩子——三层逻辑混在一起，分清就不迷路
4. **锚点稳定**：笔记用「函数名 + 关键符号」定位（如 `_last_compaction_in_place`），不用行号——符号跨版本稳定

## 研读版本

| 项 | 值 |
|---|---|
| 框架 | Hermes Agent |
| 版本 | v0.19.0 |
| Commit | `d71033a40` |
| 主文件 | `agent/conversation_loop.py` |
| 回合主循环函数 | `run_conversation()`（约 6675 行；文件共 8436 行） |
| 研读时间 | 2026-07 ~ 2026-08 |

> 笔记中的函数名/符号基于上述版本；跨版本定位请按函数名搜索，勿依赖行号。

## 架构总览

[![Hermes 架构图](assets/hermes-agent-architecture.svg)](assets/hermes-agent-architecture.html)
（浏览器打开架构图，README 内嵌为 SVG 预览）

```
入口（CLI/Gateway/飞书）
  → agent_init.py（组装 agent）
    → run_conversation()（Agent 回合主循环：约 6675 行）
      → chat_completion_helpers.py（LLM 调用：流式/非流式、认证池、fallback）
        → 模型 Provider（DeepSeek/OpenAI/Anthropic/Ollama/Bedrock）
      ↖ 工具执行回环（execute_tool）
  → 扩展：插件钩子 / MoA / context engine
  → 状态：SQLite 会话 / 上下文压缩 / prompt cache
```

## 笔记地图

### 核心：主循环六阶段

| 笔记 | 覆盖 |
|---|---|
| [conversation-loop-run-conversation](notes/conversation-loop-run-conversation.md) | run_conversation 全解（总览） |
| hermes-loop-entry-phase | 阶段 A：回合启动与迭代准备 |
| hermes-request-assembly-extras | 阶段 B：请求组装 |
| hermes-request-execution-phase | 阶段 C：请求执行 |
| hermes-response-handling-phase | 阶段 D：响应处理 |
| hermes-error-recovery-phase | 阶段 E：错误恢复（面试重点） |
| hermes-tool-execution-finalize-phase | 阶段 F：工具执行与收尾 |

### 关键机制

| 笔记 | 主题 |
|---|---|
| hermes-compression-per-turn-state-reset | 压缩状态管理（gateway 复用 agent 对象） |
| hermes-moa-context-injection | MoA 多模型协作 |
| hermes-message-normalization | 消息规范化（prompt cache 字节稳定） |
| hermes-anthropic-cache-control | Anthropic 缓存打标 |
| hermes-build-turn-context | 回合上下文组装（12 字段） |
| hermes-credential-pool-turn-reset | 认证池（多 key 轮换） |
| hermes-message-sanitize | 消息消毒 |
| hermes-prefill-messages | prefill 示范注入 |
| hermes-tool-schema-registry | 工具 schema 注册 |
| hermes-context-engine-selection | context engine 注入钩子 |

### 设计细节

（其余 20+ 篇见 notes/ 目录，每篇讲一个具体机制的"解决什么问题/怎么设计/为什么好"）

## 亮点（可引用的问题编号）

笔记中引用的 issue 均为真实存在，是深度研读的证据：
- `#38763` in-place 压缩 · `#44837` 角色交替修复 · `#62625` 静默溢出 · `#32421` 流式内容过滤 · `#26080` 认证池 401 死循环

## 许可证

笔记为个人研读产出，基于 MIT 协议的开源项目 Hermes Agent；如需引用请注明来源。
