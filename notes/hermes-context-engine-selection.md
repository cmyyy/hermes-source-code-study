# context engine：请求发出前的"上下文选择插槽"
> [← 返回笔记地图](../README.md#笔记地图) · 关键机制


> 源码位置：agent/conversation_loop.py → `_apply_context_engine_selection` / `_notify_context_engine_turn_complete`；插件发现 plugins/context_engine/
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

context engine 是一个**每次发请求前、允许外部插件决定"该给 AI 看什么上下文"的插槽**。默认没人用（no-op），但可以装引擎进去——比如装个"文档检索引擎"，它就变成：每次用户提问，引擎先去知识库搜相关材料塞进这次请求，AI 带着材料回答。员工问"公司上个月的退款政策是什么？"，默认的 AI 只有训练数据的通用知识答不上来；装了检索引擎后，AI 能引用公司内部文档给出准确回答。

引擎能做的三类事：**检索**（外部知识库/文档检索后塞进 prompt，即 RAG，最常用）、**话题路由**（按话题切换对应领域上下文）、**角色切换**（不同会话/分支用不同上下文设定）。

## 为什么这么设计

- **与压缩是两条互补机制**。压缩是被动的：历史太长、超窗阈值才触发，只能"减"（摘要/丢弃）；context engine 是主动的：每回合都跑，可能"加"内容（检索注入）。类比：压缩是"会议室装不下时把旧材料归档"，context engine 是"每次开会前主动准备材料"。两者方向相反、目的不同，互不替代。

- **框架只约定接口，不约束行为**。Hermes 只约定两件事：输入是完整的 api_messages（顺序已定好），输出是合法的非空 dict 列表就整体采用。返回的列表"长什么样"（检索块放哪、拼到 user 消息末尾还是作为独立消息插入）完全由引擎作者决定——对比内置注入（记忆 prefetch、MoA、prefill）行为固定写死，context engine 是自由扩展点。

- **fail-open：引擎出错绝不毁掉请求**。异常 → warning + 原样返回；返回 None / 空列表 / 非 dict 列表 → 原样返回。其中空列表必须显式检查——`all([])` 是 True，不查空的话坏引擎返回 [] 会把合法请求替换成空消息列表，下游救不回来。

- **浅拷贝保护存档**。传给引擎的是浅拷贝，引擎原地改输入只改到拷贝上——**持久化 transcript 永远安全**，引擎怎么折腾都污染不了历史。

- **默认 no-op 零开销**。引擎还是内置 no-op 基类的默认实现时直接短路返回，每请求零成本。这体现 Hermes 的架构哲学：**核心是窄腰、能力在边缘**——context engine 是插件扩展点，默认不占资源不跑逻辑，需要时通过 config（`context.engine`）装一个。个人用户不需要检索公司文档，就不该为这个插槽付任何代价。

- **成对的生命周期钩子**。`select_context`（请求前选择）↔ `on_turn_complete`（回合后消化）：每次用户回合 = 引擎先选 → agent 跑 → 引擎再消化。on_turn_complete 让引擎 ingest/index/summarize 刚完成的回合，本回合的 token 用量（`_last_turn_usage`）就喂给它。

## 关键机制/流程

调用点位于消息组装链的最后——api_messages 的最终顺序是 [system] → [prefill] → [历史] → [当前 user 消息]，context engine 是**整个列表的最后把关**：传入当前 user 消息作只读参考，返回值重新赋给 api_messages，引擎可以整体换掉请求列表。之后才进入孤儿清理、规范化、token 估算。

## 相关笔记

- api-messages-build（组装段，本段在其之后选择/替换）
- credential-pool-turn-reset（_last_turn_usage 的产生与归零）
- compression-per-turn-state-reset（对比对象：被动压缩）
- agent-init-structure（初始化时按配置选择引擎）
