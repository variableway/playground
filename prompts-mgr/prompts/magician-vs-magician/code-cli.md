# Code CLI配置指南


目前已经有很多的code cli工具，请列出目前最常见的几种，同时说明如何遇到不让国内用户使用的情况，如何用国内的LLM 模型API key替换的方法？

## 常见的 Code CLI 工具（简要）

- OpenAI CLI (`openai`)：官方命令行，用来管理 keys、调用 Chat/Completions、文件上传等。很多本地脚本和第三方工具默认基于 OpenAI 的接口。
- Hugging Face Inference / Hub (`huggingface-cli`)：用于上传/管理模型与调用 Inference API（适合 HF 模型与 HF 托管推理）。
- Replicate CLI (`replicate`)：调用第三方模型托管与推理服务，常见于模型快速测试/部署场景。
- Ollama（本地模型管理与运行）：把模型拉到本地运行，提供本地 HTTP 接口，适合离线或低延迟场景。
- GPT4All / llama.cpp 工具链：本地/轻量化推理工具，适合完全离线或边缘设备。
- GitHub Copilot / Copilot CLI（或通过 `gh` + copilot 插件）：更多偏向 IDE/编辑器集成，但在某些工作流中会有命令行桥接。

以上工具里，`openai` / `huggingface` / `replicate` 比较常见于云端 API 调用；`ollama`、`gpt4all`、`llama.cpp` 则偏向本地运行。

## 国内用户常遇到的限制与应对策略

- 阻断/不可用的服务：一些国外 LLM 服务（例如某些 OpenAI API 端点或第三方托管）在国内可能被限制访问，或者在注册/支付上有地域限制。
- 慢/高延迟：跨境网络导致请求延迟高、丢包或超时。
- 支付与发票问题：信用卡、企业结算、税务合规等导致无法直接长期使用国外服务。

应对策略（概览）：
- 使用国内云厂商或国内 LLM 服务商的 API（例如 Hugging Face 在中国的合作、阿里/百度/腾讯/讯飞 和若干创业公司提供的 LLM 服务）
- 使用替代的、OpenAI 兼容的终端（例如某些厂商提供 OpenAI-compatible endpoints，直接替换 base URL 与 key）
- 使用本地模型（Ollama、gpt4all、llama.cpp）以避免跨境访问
- 在无法直接替换的情况下，搭建一个小型代理/网关（proxy）把本地/国内请求转换为目标 CLI/工具所期望的格式（注意安全与合规）

## 通用：如何把 "OpenAI-style" 请求换成国内 LLM API key（步骤 + 示例）

大多数项目/脚本对接 OpenAI 有两个常见假设：
1) 使用 `OPENAI_API_KEY` 环境变量传密钥；
2) 请求遵循 OpenAI 的 REST 结构（/v1/chat/completions 等）。

替换思路：
1. 找到使用的 CLI/库如何读取 key（常见环境变量：`OPENAI_API_KEY`, `OPENAI_API_BASE`, `HF_TOKEN`, `REPLICATE_API_TOKEN`，或工具自有配置文件）。
2. 若目标国内服务兼容 OpenAI 接口（少数厂商/私有网关会提供兼容层），只需替换 `OPENAI_API_BASE` 和 `OPENAI_API_KEY` 即可。
3. 若不兼容，需要根据目标厂商 API 文档调整请求 body/headers，例如参数名、路径、鉴权方式、返回格式等。
4. 测试并回退：先用 curl 或 Postman 验证请求，再改 CLI/脚本配置。

下面给出几个实战模版：

1) OpenAI-style CLI -> OpenAI-compatible endpoint（最简单）

设置环境变量替换 base 与 key：

```bash
export OPENAI_API_BASE="https://api.your-domestic-provider.com/v1"
export OPENAI_API_KEY="sk-domestic-xxxxxxxx"

# 然后运行依赖 openai 环境变量的 CLI/脚本（示例）
openai chat completions.create -m gpt-4o-mini -M '{"role":"user","content":"你好"}'
```

注意：不是所有国内厂商都支持完全兼容的路径与参数，如果 CLI 抛出 404/400，说明接口不兼容，需要按下述方式定制。

2) Hugging Face Inference API（示例）

Hugging Face 使用自己的 header (`Authorization: Bearer $HF_API_TOKEN`) 与路径：

```bash
export HF_API_TOKEN="hf_xxx"

curl -X POST "https://api-inference.huggingface.co/models/<model-id>" \
	-H "Authorization: Bearer $HF_API_TOKEN" \
	-H "Content-Type: application/json" \
	-d '{"inputs":"请把下面代码重构为更可读的形式: ..."}'
```

3) Replicate（示例）

```bash
export REPLICATE_API_TOKEN="replicate_xxx"

curl -X POST "https://api.replicate.com/v1/predictions" \
	-H "Authorization: Token $REPLICATE_API_TOKEN" \
	-H "Content-Type: application/json" \
	-d '{"version":"<version-id>","input": {"prompt": "写一个单元测试"}}'
```

4) Hugging Face / Replicate 等返回格式可能和 OpenAI 不同：
- OpenAI 返回的是 `choices[].message.content`；HF/Replicate 可能直接把文本放在 `generated_text` 或不同字段，脚本中需要做字段映射。

5) 国内厂商（通用模式）

很多国内厂商会采用下面两类鉴权：
- Header Bearer（Authorization: Bearer <API_KEY>）
- Query string token (?access_token=<token>) 或自有 header 名称（例如 `X-API-KEY`）

示例（伪通用 curl，替换为目标 provider 文档中的 endpoint 与字段）:

```bash
export PROVIDER_KEY="your_domestic_key"

curl -X POST "https://api.domestic-llm.example.com/v1/chat/completions" \
	-H "Authorization: Bearer $PROVIDER_KEY" \
	-H "Content-Type: application/json" \
	-d '{"model":"chat-model-1","messages":[{"role":"user","content":"把这段 JS 代码改成更清晰"}]}'
```

如果 provider 不使用 OpenAI 相同的参数名（比如把 `messages` 改成 `input` 或 `prompt`），需要把客户端/脚本中发送请求的地方做对应变更或增加一层转换代理。

## 如果 CLI 本身不允许替换 base URL 或环境变量怎么办？

选项：
1. 在本地写一个小型请求代理（把 OpenAI 风格的请求接收后转换为目标 provider 的格式），然后把 `OPENAI_API_BASE` 指向本地代理地址；注意保护密钥与访问控制。
2. 改造脚本/使用 SDK：用对应厂商的 SDK/HTTP 请求代码替代原来直接调用 OpenAI 的部分。
3. 使用本地模型（Ollama / gpt4all / llama.cpp 等），避免网络与鉴权问题。

示例：最小代理示意（Node.js + express，仅做演示，不可直接用于生产）：

```js
// ...existing code...
// 伪代码：接收 OpenAI 风格请求，转换并转发到国内 provider
const express = require('express');
const app = express();
app.use(express.json());
app.post('/v1/chat/completions', async (req, res) => {
	const payload = req.body; // OpenAI 风格
	// build domestic provider payload
	const providerBody = {
		model: 'chat-model-1',
		input: payload.messages.map(m => m.content).join('\n')
	};
	// forward request to provider with provider key
	// ...fetch to provider endpoint...
	res.json({/* normalized response to mimic OpenAI */});
});
app.listen(3000);
```

## 测试与回归

- 先在终端用 `curl` 验证国内 provider 的接口与鉴权。
- 把脚本/CLI 指向新 endpoint，运行单元/集成级 quick smoke 测试。
- 注意返回格式差异（文本字段、usage 字段）并做必要的转换。

## 小结（checklist）

1. 确认目标 provider 支持的鉴权方式（Bearer / token / query）。
2. 尝试是否能直接替换 `OPENAI_API_BASE` + `OPENAI_API_KEY`（最快）。
3. 如果不兼容，使用 provider SDK 或写一个轻量代理来做字段/参数映射。
4. 在 PR 描述中说明你替换了哪个 endpoint、使用了哪个 key、并附上测试样例。

如果你想，我可以把上面的代理示例扩展成一个可运行的小仓库（Node 或 Python 版本），并生成一个具体的 `curl` 测试脚本。告诉我你偏好的国内模型提供商（例如：百度文心、阿里盘古、讯飞、智谱/ChatGLM、阿尔法/权威厂商或私有 Ollama 实例），我会把示例对齐到该厂商的真实请求格式。
