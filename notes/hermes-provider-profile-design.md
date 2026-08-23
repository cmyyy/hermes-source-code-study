# ProviderProfile：把 provider 的差异关进声明式的笼子

> 源码位置：`providers/base.py` → `ProviderProfile`（dataclass）；实战子类见 `plugins/model-providers/deepseek/__init__.py` → `DeepSeekProfile`
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

`ProviderProfile` 是 Hermes **声明式 provider 配置**的核心类：把接入一个 AI 推理服务所需的一切——认证方式、端点、默认参数、行为差异——统一放进一个 dataclass。它的职责单一：**描述 provider 长什么样**，不负责构建 client、不负责认证轮换、不处理流式响应（这些归 AIAgent）。支持 20+ provider 而不把核心代码变成 if-else 泥潭，靠的就是这个抽象。

## 为什么这么设计

- **声明式而非命令式：行为描述与逻辑执行分离**。文件头的 docstring 是设计宣言：Provider profiles are DECLARATIVE — they describe the provider's behavior. They do NOT own client construction, credential rotation, or streaming。每个 provider 实例是一个可复用的配置对象，AIAgent 读它来驱动通用逻辑。**新增一个 provider = 新增一个配置对象 + 可选钩子重写，主流程零改动**——这是"把差异数据化"的经典做法，也是 Hermes 能一天接入一个新平台的底气。

- **base_url 与 models_url 分离，两个端点可以不同域**。推理端点和模型目录端点大多数 provider 同域（OpenAI 都在 /v1 下），但 relay 服务可能模型目录在别的域名，有些模型目录是公开接口无需认证、推理接口要 API key。**分开配置让"不需要认证的公开端点"和"需要认证的推理端点"各管各的**；models_url 为空时 fallback 到 `{base_url}/models`。

- **OMIT_TEMPERATURE sentinel：三个语义用一个字段表达**。`fixed_temperature` 有三种取值：`None`（用调用方默认）、具体数值（固定温度）、`OMIT_TEMPERATURE`（完全不发 temperature 参数）。`None` 无法同时表示"用默认"和"不要发"，所以用 `object()` sentinel 区分——Kimi 要求请求里完全不出现 temperature，由服务端自己管理。**当"缺省"和"显式不发"是两种语义时，需要 sentinel 而不是复用 None**，这是 Python API 设计里容易被忽略的细节。

- **钩子插槽：把差异留给子类重写，主流程不动**。五个钩子覆盖 provider 的典型差异点：`prepare_messages`（发送前改消息，如 Anthropic 格式转换）、`build_extra_body`（往 extra_body 加参数）、`build_api_kwargs_extras`（最复杂，双路返回）、`get_max_tokens`（按模型返回不同上限，relay 类 provider 需要）、`fetch_models`（拉模型列表，唯一发 HTTP 的方法，用 `hermes-cli/x.x.x` 自定义 UA 规避 WAF——默认的 Python-urllib UA 会被某些 WAF 当爬虫直接 403）。**主流程到点自动调用钩子，子类重写来定制行为**，这是模板方法模式的典型应用，每个钩子的默认实现都是无害透传，子类只管自己关心的那一个。

- **build_api_kwargs_extras 双路返回，因为 reasoning 参数在不同 provider 的请求里位置不同**。OpenAI 把 `reasoning_effort` 放顶层 api_kwargs，OpenRouter 要求 `{reasoning: {...}}` 放 extra_body（httpx 展开到请求体的嵌套字段），Kimi 又放顶层。统一往一处塞必然出错，所以钩子返回 `(extra_body_additions, top_level_kwargs)` 两路，transport 层合并 `body = {**api_kwargs, **extra_body}`。**协议差异不是 bug，是现实，抽象的任务是给现实留出表达空间**。

## 关键机制/流程

DeepSeekProfile 是钩子的最佳实战：DeepSeek V4 服务端默认开 thinking，若请求没显式声明 thinking 状态，回放带 `reasoning_content` 的历史时服务端返回 400（reasoning_content must be passed back）。解法是每次请求显式声明：`extra_body.thinking = {"type": "enabled"|"disabled"}` + 顶层 `reasoning_effort`（low/medium/high/max，effort 映射 xhigh/max → max，不传则用服务端默认）。模型判断用 `_model_supports_thinking`：`startswith("deepseek-v")` 且非 V3 → 支持——**用前缀匹配而非枚举版本号，未来出 V5、V6 自动生效不用改代码**。实例化时 `default_aux_model="deepseek-chat"` 还印证了另一个设计：主模型用贵的 V4，压缩等辅助任务用便宜的 V3。

钩子调用顺序：codex 字段清理 → `prepare_messages` → developer 角色交换 → 构建请求 → `build_extra_body` / `build_api_kwargs_extras` → `get_max_tokens` → 发出。

## 相关笔记

- hermes-credential-pool-turn-reset（认证池：同一 provider 换 key，与 fallback 换 provider 分层）
- hermes-request-assembly-extras（请求组装：钩子产出的参数如何进入请求）
- hermes-conversation-loop-run-conversation（主循环总纲）
