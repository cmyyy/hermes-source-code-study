# 模型的"工具菜单"是怎么来的：注册、过滤与三层验货

> 源码位置：`tools/registry.py`（登记簿）→ `model_tools.py`（收集/过滤/coerce）→ `tools/file_tools.py`（schema 示例）→ `hermes_cli/tools_config.py`（平台配置）
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

模型看到的工具列表不是全量发的：每个工具的手写 schema 是"说明文档"，经"注册 → 按 toolset 过滤 → 可用性检查"三步后，才变成 API 请求里的 tools 参数；模型返回参数后还有"coerce 强转 → schema 校验 → handler 归一化"三层验货，保证 handler 拿到能干活的值。

## 为什么这么设计

**手写 schema 与 handler 配对注册**。每个工具源码末尾调 `registry.register(name, toolset, schema, handler, check_fn)`——"说明文档"（模型看的）和"干活函数"（执行的）在这里登记配对，登记簿（`tools/registry.py`）统一保管。schema 描述必须准确：模型靠 description 决定"这题调不调这个工具"，写不清楚它就乱点或不敢点。check_fn 控制暴露：依赖没装（如无浏览器）的工具 schema 根本不出现在请求里——模型看不到 = 不会幻觉调用。

**toolset 四层启用链，层层收窄**。一个工具最终出不出现在模型面前要过四层：运行时显式 `enabled_toolsets`（AIAgent 构造参数，优先级最高，典型用途是**收窄**——cron 只给 web/file、delegate 按任务给、kanban worker 只给 kanban 工具集）> 平台默认（`hermes tools` 向导写入 `platform_toolsets`，CLI 全量、Telegram/Discord 基础集）> 全局禁用（config 的 `agent.disabled_toolsets`）> check_fn 运行时可用性。收窄既是成本控制（子代理只给必要工具集，输入 token 大减）也是安全（禁用 = 模型看不到 = 不会调用）。收集结果还有缓存（`_tool_defs_cache`，key 含 toolset、config 指纹、registry 代际）。

**为什么是"注册"而不是"自动收集"**。schema 是开发者手写的说明文档，质量由人保证——模型对工具的理解完全来自 description，自动生成的描述往往空洞或误导（AGENTS.md 专门强调这一点）。注册制还顺带把"说明文档、执行函数、可用性检查、展示表情"绑定成一个工具单元，后续无论过滤、校验、审计都按这个单元走。

**参数三层验货**。LLM 返回的参数是字符串化的 JSON，模型常把数字写成 `"42"`、布尔写成 `"true"`、数组写成 `"[...]"`，不处理 handler 一接就崩。第一层 `coerce_tool_args` 按 schema 声明类型强转（`"42"` → 42、JSON 字符串解析成 list/object、union 类型依次尝试、nullable 的 `"null"` → None），还带两个容错：裸标量自动包数组（schema 要数组但模型给单个字符串——开权模型 DeepSeek/Qwen/GLM 常见，避免好调用莫名失败）和强转失败保留原值（不硬转出垃圾，让下一层兜底）；第二层 schema 校验（required 字段缺没缺、类型对不对），coerce 尽力修、修不好就报错，防脏参数进 handler；第三层 handler 内部归一化（工具自己的默认值/容错兜底）。另有一个细节：含非法字符的参数名（如 Cloudflare 的 `issue_class~neq`）发送前被改成合规名（`~` → `_`），模型看到/返回的都是改名后的名字，coerce 前用 `unrename_tool_args` 映射回原始名再查 schema——改名规则是纯函数，还原表独立计算永远一致，不会错位。

## 关键机制/流程

出站：registry.register 注册 → `get_tool_definitions()` 按启用 toolset 收集（有显式 enabled 只用这些，否则全量，再减 disabled）→ check_fn 过滤 → schema 列表进 API 请求的 tools 参数。入站：模型返回参数 → coerce_tool_args 按 schema 强转 → registry 分发时 schema 校验 → handler 内部归一化 → 执行。

## 相关笔记

- hermes-conversation-loop-run-conversation.md
- hermes-tool-call-message-structure.md
- hermes-tool-execution-finalize-phase.md
