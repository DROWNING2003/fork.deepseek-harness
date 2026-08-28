# Agent Note: llm-deepseek OpenAI 方言

Status: implemented

[English](2026-08-15-llm-deepseek-openai-dialect.md) | 中文

## 问题

默认的 `llm-deepseek` 提供方被绑定在 DeepSeek 协议方言上。其序列化器总是发出 DeepSeek 扩展字段——`thinking` 请求字段、`reasoning_content` assistant 回传、`max` 推理强度——而这些字段会被纯 OpenAI chat-completions 端点（GPT 模型或 OpenAI 兼容 gateway）拒绝或忽略。因此把 `baseURL` 指向 `https://api.openai.com/v1` 会产生无法工作的请求，默认端点、凭据引用与模型目录也都是 DeepSeek 专属且没有开关。

## 决策

`Config` 新增 `dialect: 'deepseek' | 'openai'`（默认 `deepseek`）。`resolveAdapterOptions` 仍然是唯一的显式 resolve 步骤，现在按方言选取默认值：`apiKeyEnv`（`DEEPSEEK_API_KEY`／`OPENAI_API_KEY`）、`baseURL`（DeepSeek 公共 API／`https://api.openai.com/v1`，各自优先读取自己的受信环境变量）、以及建议模型目录（`deepseek-v4-flash`／`deepseek-v4-pro`，或 `gpt-4o`／`gpt-4o-mini`／`gpt-4.1`）。

序列化器按方言工作：OpenAI 方言绝不发送 `thinking`，绝不把 `reasoning_content` 回传进 assistant 历史，只接受 `off`／`high` 两种强度——`max` 会响亮失败，按请求抛出 `UNSUPPORTED_REASONING_EFFORT`，配置层面则在 resolve 时被拒绝。`dialect: openai` 下配置 `thinking` 会使插件加载失败。适配器的 `resolveModel` 在 OpenAI 方言下只公开 `off`／`high`，默认 `off`。DeepSeek 流必须发送 `[DONE]`；OpenAI 方言也会在收到合法 `finish_reason` 后接受 EOF，从而兼容省略该标记的 gateway，同时不会接受截断的流。分片工具调用中的后续空函数名不会覆盖之前的非空名称。嵌入 HTTP 200 SSE 响应的错误对象会被分类为提供方失败，不会被忽略直至 EOF。

settings 层会把缺失的 `models` 键规范化为 `[]`，使「空列表」与「未设置」无法区分；现在空列表或未设置的 `models` 都解析为当前方言的目录，而不是公布零个模型。

Models 页面编辑器（deepseek 家族）在自定义折叠区内提供方言选择器；切换它会同步替换 API 地址占位符、继承的建议模型行与凭据引用（`OPENAI_API_KEY`／`DEEPSEEK_API_KEY`），并在草稿中尊重尚未保存的方言切换——一次 Apply 即可写入方言并把密钥存入新引用。可配置提供方目录的显示名随方言变化（`OpenAI`／`DeepSeek`），变更时原地重新注册，与既有的重试策略重新注册并列。

## 备选方案

**新增独立的 `llm-openai` 提供方插件。** 已拒绝：用户要求改造默认提供方，且线协议本就 OpenAI 兼容——缺口是 DeepSeek 专属表面，不是传输层。

**保留 schema 默认值并为 openai 方言做特判。** 已拒绝：settings 层的解析值无法区分存储的 `models: []` 与 schema 默认值，因此按方言的默认值必须放在显式 resolve 步骤里。

**在 OpenAI 方言下把 `max` 映射为 `high`。** 已拒绝：静默替换会让用户选择的控制项与记录的请求意图不一致；应响亮失败。

## 后果

组合可以把 `deepseek-official` 路由指向任意 OpenAI 兼容的 chat-completions 端点并使用 OpenAI 默认值，或保持 DeepSeek 不变。两种方言下路由 id 都保持 `deepseek-official`；只有显示名、目录、默认值与线表面变化。空 `models` 列表不再表示「不公布任何模型」（settings 层无法区分地表达这一点）；需要空建议目录的部署不应使用本适配器的目录字段。文档更新：`packages/llm/llm-deepseek/README.md` 与 `.zh.md`，以及重新生成的 `docs/config-catalog.md` 与其镜像 `.zh.md`。
