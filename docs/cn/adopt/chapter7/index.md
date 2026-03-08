# 第七章 多模型与成本优化

OpenClaw 不绑定任何单一 LLM 提供商。你可以同时配置 Claude、GPT、本地 Ollama 等多个模型，根据任务复杂度智能路由，在保证质量的同时大幅降低 API 费用。

## 1. 支持的模型

### 1.1 云端模型

| 提供商 | 模型 | 特点 | 适合任务 |
|--------|------|------|---------|
| Anthropic | `anthropic/claude-opus-4-6` | 最强推理，200K 上下文 | 复杂分析、架构设计 |
| Anthropic | `anthropic/claude-sonnet-4-6` | 均衡性能 | 日常编码、文档生成 |
| Anthropic | `anthropic/claude-haiku-4-5` | 速度快、成本低 | 简单查询、格式转换 |
| OpenAI | `openai/gpt-5` | 强推理能力 | 数学、逻辑推理 |
| OpenAI | `openai/gpt-4o` | 多模态支持 | 图片理解、语音处理 |
| Google | `google/gemini-3-pro` | 1M 上下文 | 大文件分析 |

> **模型标识格式**：OpenClaw 统一使用 `provider/model-name` 格式标识模型。目前支持 12 个官方提供商：Ollama、OpenAI、Anthropic、OpenRouter、Amazon Bedrock、Vercel AI Gateway、Moonshot AI、MiniMax、OpenCode Zen、GLM、Z.AI、Synthetic。

### 1.2 本地模型（Ollama）

| 模型 | 参数量 | 最低显存 | 特点 |
|------|--------|---------|------|
| Llama 3.3 | 70B | 48GB | 开源最强通用模型 |
| Qwen 2.5 | 72B | 48GB | 中文能力突出 |
| DeepSeek V3 | 671B (MoE) | 24GB | 编码能力强 |
| Phi-4 | 14B | 8GB | 轻量级推理 |

## 2. 配置多模型

### 2.1 添加模型提供商

```bash
# 配置 Anthropic（默认）
openclaw config set llm.providers.anthropic.apiKey "sk-ant-xxxxx"

# 添加 OpenAI
openclaw config set llm.providers.openai.apiKey "sk-xxxxx"

# 添加本地 Ollama
openclaw config set llm.providers.ollama.baseUrl "http://localhost:11434"
```

### 2.2 设置默认模型

```bash
# 主模型：用于大多数任务
openclaw config set llm.default "claude-sonnet-4-6"

# 回退模型：主模型不可用时使用
openclaw config set llm.fallback "gpt-4o"
```

### 2.3 配置文件示例

```json
// openclaw.json
{
  "llm": {
    "default": "anthropic/claude-sonnet-4-6",
    "fallback": "openai/gpt-4o",
    "providers": {
      "anthropic": {
        "apiKey": "sk-ant-xxxxx",
        "models": ["claude-opus-4-6", "claude-sonnet-4-6", "claude-haiku-4-5"]
      },
      "openai": {
        "apiKey": "sk-xxxxx",
        "models": ["gpt-5", "gpt-4o"]
      },
      "ollama": {
        "baseUrl": "http://localhost:11434",
        "models": ["qwen2.5:72b", "deepseek-v3"]
      }
    }
  }
}
```

> **注意**：OpenClaw 的配置文件为 `openclaw.json`（JSON 格式），不是 YAML。

## 3. 模型路由策略

### 3.1 基于任务复杂度

OpenClaw 可以根据任务自动选择合适的模型：

```json
{
  "llm": {
    "routing": {
      "simple": "anthropic/claude-haiku-4-5",
      "standard": "anthropic/claude-sonnet-4-6",
      "complex": "anthropic/claude-opus-4-6"
    }
  }
}
```

运行时也可以通过命令快速切换模型：

```
/model fast
```

### 3.2 基于技能类型

不同技能可以指定不同的模型：

```json
{
  "skills": {
    "weather": {
      "model": "anthropic/claude-haiku-4-5"
    },
    "code-reviewer": {
      "model": "anthropic/claude-opus-4-6"
    },
    "translator": {
      "model": "openai/gpt-4o"
    }
  }
}
```

## 4. 本地模型部署（Ollama）

### 4.1 安装 Ollama

```bash
# Linux
curl -fsSL https://ollama.ai/install.sh | sh

# macOS
brew install ollama
```

### 4.2 下载模型

```bash
# 下载 Qwen 2.5（推荐中文场景）
ollama pull qwen2.5:72b

# 下载 DeepSeek V3（推荐编码场景）
ollama pull deepseek-v3

# 轻量级模型（适合低配置机器）
ollama pull phi4:14b
```

### 4.3 配置 OpenClaw 使用本地模型

```bash
openclaw config set llm.providers.ollama.baseUrl "http://localhost:11434"
openclaw config set llm.default "ollama/qwen2.5:72b"
```

### 4.4 混合模式

推荐的混合部署：本地模型处理日常任务（零成本），复杂任务切换到云端 API：

```json
{
  "llm": {
    "default": "ollama/qwen2.5:72b",
    "fallback": "anthropic/claude-sonnet-4-6",
    "routing": {
      "simple": "ollama/phi4:14b",
      "complex": "anthropic/claude-opus-4-6"
    }
  }
}
```

> **Ollama 自动发现**：OpenClaw 会自动查询 Ollama 的 `/api/tags` 和 `/api/show` 接口，发现本地已安装的模型，无需手动逐一配置。

## 5. 成本监控与优化

### 5.1 查看 Token 消耗

```bash
# 查看今日消耗
openclaw usage today

# 查看本月统计
openclaw usage month

# 按技能查看消耗
openclaw usage --by-skill
```

### 5.2 设置预算上限

```json
{
  "llm": {
    "budget": {
      "daily": 5.00,
      "monthly": 100.00,
      "alert_at": 80
    }
  }
}
```

### 5.3 成本优化技巧

**使用缓存**：相同查询不重复调用 API。OpenClaw 内置了语义缓存，相似问题会复用之前的结果。

**精简 Prompt**：过长的系统提示词会增加每次调用的 Token 数。定期审查技能的提示词，去除冗余内容。

**选择合适的模型**：不要所有任务都用最贵的模型。天气查询用 Haiku（$0.25/M tokens）就够了，不需要 Opus（$15/M tokens），价格差 60 倍。

**减少活跃技能**：每个活跃技能的说明都会加入上下文，增加 Token 消耗。只保留常用技能。

### 5.4 成本对比

| 使用模式 | 日均调用 | 月估算费用 |
|---------|---------|-----------|
| 全部 Opus | 100 次 | ~$150-300 |
| 全部 Sonnet | 100 次 | ~$30-60 |
| 混合路由 | 100 次 | ~$15-30 |
| 本地 + 云端回退 | 100 次 | ~$5-10 |

## 6. 常见问题

**模型切换延迟**：不同模型的首次调用可能有冷启动延迟。Ollama 本地模型需要先加载到显存，首次响应会慢一些。

**本地模型质量不够**：对于复杂任务，本地模型确实不如 Claude Opus。建议设置自动回退：本地模型处理失败时自动切换到云端。

**API 限流**：Anthropic 和 OpenAI 都有 RPM（每分钟请求数）限制。如果遇到 429 错误，减少并发任务数或升级 API tier。

---

**下一步**：[第八章 个人助理系统](/cn/adopt/chapter8/)
