# 第一章 十分钟上手 OpenClaw

这一章会带你完成 OpenClaw 的安装，从环境准备到运行配置向导，整个过程不超过 10 分钟。

## 1. 系统要求

- **操作系统**：macOS、Linux、或 Windows（推荐 WSL2）
- **Node.js**：22 或更高版本
- **内存**：至少 1GB，推荐 4GB
- **端口**：18789 需要可用

检查 Node.js 版本：

```bash
node --version
```

如果版本低于 22，需要先升级 Node.js。

## 2. 安装 Node.js

### Windows 用户

**方案一：使用一键安装脚本（推荐）**

打开 PowerShell（管理员模式），运行：

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

这个脚本会自动安装 Node.js 和 OpenClaw。如果遇到权限错误，先运行：

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

**方案二：手动安装**

1. 下载 nvm-windows：访问 [github.com/coreybutler/nvm-windows/releases](https://github.com/coreybutler/nvm-windows/releases) 下载 `nvm-setup.exe`
2. 安装后重新打开 PowerShell（管理员模式）
3. 安装 Node.js 22：

```powershell
nvm install 22
nvm use 22
```

### macOS 用户

使用 Homebrew 安装：

```bash
brew install node@22
```

### Linux 用户

使用 nvm（推荐）：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc
nvm install 22
nvm use 22
```

## 3. 安装 OpenClaw

全局安装 OpenClaw CLI：

```bash
npm install -g openclaw@latest
```

验证安装：

```bash
openclaw --version
```

看到版本号说明安装成功。

## 4. 运行配置向导

安装完成后，运行配置向导：

```bash
openclaw onboard --install-daemon
```

这个命令会启动交互式配置向导，并安装后台守护进程。

### 4.1 选择模型提供商

向导首先会询问你使用哪个 AI 模型：

```
◇  Model/auth provider
│  ● Anthropic (Claude)
│  ○ OpenAI (GPT)
│  ○ DeepSeek（国内推荐）
│  ○ OpenRouter（聚合多家模型）
```

**国内用户推荐**：
- **DeepSeek**：国内服务商，支持支付宝充值，注册即用
- **OpenRouter**：一个 Key 访问多家模型，支持多种支付方式

**国际用户**：选择 Anthropic 或 OpenAI，需要国际信用卡。

### 4.2 输入 API Key

根据选择的提供商输入 API Key：

```
◇  DeepSeek API key
│  （粘贴你的 API Key）
```

获取 API Key：

- DeepSeek: https://platform.deepseek.com
- OpenRouter: https://openrouter.ai
- Anthropic: https://console.anthropic.com
- OpenAI: https://platform.openai.com

### 4.3 配置聊天渠道（可选）

向导会询问是否配置 Slack、Telegram 等聊天渠道。如果暂时不需要，可以选择跳过，后续通过 `openclaw configure` 添加。

### 4.4 配置技能（可选）

向导会显示可用的技能列表，询问是否安装。建议先跳过，等熟悉基本操作后再安装。

配置完成后，向导会自动启动 Gateway 守护进程。

## 5. 验证安装

检查 Gateway 状态：

```bash
openclaw status
```

看到 `Gateway service: running` 说明安装成功。

打开 Web 控制面板：

```bash
openclaw dashboard
```

浏览器会自动打开 `http://localhost:18789`，你可以在这里和 OpenClaw 对话。

## 6. 第一次对话

在控制面板输入：

```
帮我列出当前目录的文件
```

如果 OpenClaw 能正确执行命令并返回结果，说明一切正常。

