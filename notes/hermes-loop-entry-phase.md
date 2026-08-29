# 回合启动：问模型之前，先把所有准备做对
> [← 返回笔记地图](../README.md#笔记地图) · 六阶段核心 · [← 上一篇](conversation-loop-run-conversation.md) · [下一篇 →](hermes-request-assembly-extras.md)


> 源码位置：`agent/conversation_loop.py` → `run_conversation()` 入口与循环骨架（函数签名、回合前奏、主循环顶部）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

一个回合（turn）从函数入口到真正"问模型问题"之前，有一段庞大的准备代码，解决一个核心问题：**把准备工作做对，不然要么上一回合的状态串到这一回合，要么循环停不下来**。它分三段：函数签名（回合的契约）、回合前奏（一次性准备）、循环骨架（主循环入口 + 每轮迭代开头的闸门与准备）。

## 为什么这么设计

- **gateway 跨回合复用 agent 对象，所以 per-turn 状态必须显式归零**。回合结束对象不销毁、下一回合接着用——压缩状态若不清，回合 A 压缩过、回合 B 没压缩，gateway 读到残留标志会把 B 的正常对话误判成"压缩过"→ 错误重写（数据损坏级 bug）。所以回合前奏第一件事是清掉压缩三标志（Hermes 压缩默认 in-place，`compression.in_place: True`，#38763）。这与 Java 线程池复用线程时 ThreadLocal 必须 remove 是同一个道理：**跨复用边界的对象，per-turn 状态要自己负责清理**。

- **用户消息维护"工作版"和"纯净版"两个版本**。`user_message` 是工作版——剥离孤立代理字符（surrogate，会崩 json.dumps 的 Unicode 半零件，坏粘贴密钥是持久 UnicodeEncodeError 的主因，#6843）、可能被注入内容；`original_user_message` 是纯净版——同样剥离 surrogate 但**绝不注入**，供存档、记忆检索和日志使用。**要操作的和要存档的分开**，避免存档里混入运行期污染。

- **循环顶部的"闸门"与"准备"严格分离**。每轮迭代开头做五件事，但只有两件是真闸门（判断"这轮还让不让我干活"，有 break 退出点）：用户中断检查（发了新消息 → interrupted → break）和预算消费（扣不动 → budget_exhausted break）。另外三件是纯准备工作：把用户中途的纠错（/redirect）拼进消息、每迭代一次 checkpoint 快照、迭代计数。**闸门管控制流，准备管工作流**——类比过安检（不合格拦下）与登机后放行李（已确定能飞）。三层循环也就从这里展开：外层 while（回合迭代）→ 中层 retry（单次 API 调用）→ 内层恢复分派。

- **预算检查是双保险，不是冗余**。循环条件读 `remaining > 0` 是快速门卫（非原子检查），循环内 `consume()` 是原子占用（IterationBudget 带 threading.Lock）。两处之间不是原子操作——其他线程可能抢走最后一个名额；而且"宽限调用"（`_budget_grace_call`）专门在预算为 0 时放行。**所以第二次检查是兜底**：一处管"能不能进"，一处管"进去了有没有名额"。

- **/steer 用"搭便车注入"保持角色交替**。用户运行中递纸条（/steer 注意别改配置文件），Hermes 不插一条新 user 消息——那会破坏 tool→assistant→user 的严格交替，被严格 provider 拒绝。而是把纸条（带 `[User steer: ...]` marker）**追加到最后一条 tool 消息的 content 末尾**：角色没变、内容多几个字、交替完好。**新内容不新开消息，附在已有消息上**，是处理"运行中用户输入"的优雅方案。第一轮还没有 tool 消息可贴时，把纸条放回 pending，等工具执行完再贴（`apply_pending_steer_to_tool_results`）。

- **MoA 请求用魔数前缀解码，消息体与配置分离**。MoA（多模型圆桌）回合的请求和普通请求长得一样，只是 user_message 里藏了一个 `__HERMES_MOA_TURN_V1__` + base64(JSON) 前缀。入口负责认出它、拆开：解出配置就把消息体还原、配置存好；解不出就当作普通消息。注意 MoA ≠ MoE：MoA 是应用层的多模型协作（各答一遍再综合，花钱但灵活），MoE 是模型内部的专家路由。默认配置指向两个 provider 的凭据，没配 key 会 loud 报错——提醒用户这是要花钱的功能。

## 关键机制/流程

```
入口（9 个参数定义回合边界：user_message / system_message / conversation_history /
     task_id / stream_callback / persist_user_message / moa_config 等）
  → 解码 MoA 魔数（认出圆桌请求）
  → 压缩状态归零（清上一回合的账）
  → build_turn_context（打包 12 个字段：输入 / 身份 / 定位 / 旁路四组）
  → 过程旁白去重集合重置（防刷屏，去重范围=整个回合，不跨回合抑制）
  → 16 个循环计数器初始化（全是局部变量，天然 per-turn）
  → 认证池 + 回合用量归零（防 401 死循环空转，#26080）
  → Codex App Server 旁路检查（IDE 集成模式整个循环跳过）
  → while 循环：redirect drain → checkpoint → 中断检查（闸门）→ 计数 → 预算消费（闸门）
```

回合前奏里还有两个细节值得留意：`turn_id` 是 `session_id:task_id:random` 三层嵌套，看到 turn_id 就知道属于哪个会话、哪个任务；step_callback 只在"有人听"时才发（CLI 没有钩子系统时整个 if 零成本跳过），它是进度心跳而非执行审计，工具执行的细节由 tool_progress_callback 实时上报。

## 相关笔记

- hermes-loop-counter-init（16 个计数器的分组详解）
- hermes-build-turn-context（回合前奏 12 字段的产出方）
- hermes-compression-per-turn-state-reset（压缩状态归零）
- hermes-steer-mechanism（/steer 递纸条机制）
- hermes-step-callback-explained（进度广播）
- hermes-conversation-loop-run-conversation（主循环总纲）
