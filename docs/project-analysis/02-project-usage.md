# Hermes Agent 项目使用指南

> 从安装到高级用法的完整指南

## 1. 安装

### 1.1 一键安装（推荐）

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

支持 Linux、macOS 和 WSL2。安装器自动处理 Python、Node.js、依赖安装和 `hermes` 命令注册。

安装后：
```bash
source ~/.bashrc    # 重新加载 shell
hermes              # 开始对话
```

### 1.2 开发环境安装

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
git submodule update --init mini-swe-agent  # 必需

# 安装 uv 包管理器
curl -LsSf https://astral.sh/uv/install.sh | sh

# 创建虚拟环境并安装
uv venv .venv --python 3.11
source .venv/bin/activate
uv pip install -e ".[all,dev]"
uv pip install -e "./mini-swe-agent"

# 验证安装
python -m pytest tests/ -q
```

### 1.3 可选组件

```bash
# 强化学习训练支持
git submodule update --init tinker-atropos
uv pip install -e "./tinker-atropos"

# 仅安装特定功能组
uv pip install -e ".[messaging]"    # 消息平台支持
uv pip install -e ".[voice]"        # 语音交互
uv pip install -e ".[mcp]"          # MCP 协议
uv pip install -e ".[modal]"        # Modal Serverless
```

---

## 2. 基础命令

### 2.1 CLI 命令总览

```bash
hermes              # 启动交互式对话
hermes model        # 选择 LLM 提供商和模型
hermes tools        # 配置启用的工具
hermes config set   # 设置单个配置项
hermes gateway      # 启动消息网关（Telegram、Discord 等）
hermes setup        # 运行完整的设置向导
hermes update       # 更新到最新版本
hermes doctor       # 诊断问题
hermes claw migrate # 从 OpenClaw 迁移
```

### 2.2 首次设置

```bash
hermes setup
```

设置向导将引导你完成：
1. **LLM 提供商选择** — OpenRouter、Anthropic、Nous Portal 等
2. **API 密钥配置** — 安全存储在 `~/.hermes/.env`
3. **模型选择** — 从可用模型列表中选择
4. **工具配置** — 启用/禁用各工具集
5. **消息平台配置**（可选） — Telegram Bot Token 等

---

## 3. 交互式 CLI 使用

### 3.1 基本对话

```
hermes

╭─ Hermes Agent ─────────────────────╮
│ Welcome! How can I help you today? │
╰────────────────────────────────────╯

You: 帮我分析这个项目的代码结构

(✿◠‿◠) thinking...

╭─ Hermes ──────────────────────────╮
│ 这个项目采用了分层架构...          │
╰───────────────────────────────────╯
```

### 3.2 CLI 快捷键与功能

| 功能 | 操作 |
|------|------|
| 多行输入 | 直接粘贴多行文本 |
| 中断并重新指示 | Ctrl+C |
| 退出 | `/exit` 或 Ctrl+D |
| 命令自动补全 | 输入 `/` 后 Tab |
| 查看历史 | 上/下方向键 |

### 3.3 斜杠命令

```bash
/help              # 显示可用命令
/model <name>      # 切换模型
/tools             # 查看/切换工具
/sessions          # 浏览历史会话
/search <query>    # 搜索历史会话
/memory            # 查看/管理记忆
/skills            # 浏览/安装技能
/skin <name>       # 切换主题皮肤
/clear             # 清除当前对话
/exit              # 退出
```

---

## 4. 配置管理

### 4.1 配置文件位置

```
~/.hermes/
├── config.yaml      # 主配置文件
├── .env             # API 密钥（敏感信息）
├── memory.md        # 持久化记忆
├── sessions.db      # 会话数据库（SQLite）
├── skills/          # 技能文件
└── skins/           # 自定义主题
```

### 4.2 核心配置项（config.yaml）

```yaml
# 模型配置
model: "anthropic/claude-sonnet-4-20250514"
agent:
  max_turns: 90

# 终端后端
terminal:
  backend: local          # local, docker, ssh, modal, daytona, singularity
  # Docker 后端示例
  # backend: docker
  # docker:
  #   image: "python:3.11"
  #   volumes: [".:/workspace"]

# 显示设置
display:
  skin: default           # default, ares, mono, slate
  background_process_notifications: all  # all, result, error, off

# Git Worktree
git_worktree:
  enabled: false
  base_dir: null

# 语音设置
voice:
  transcription_provider: local  # local, openai
```

### 4.3 环境变量（.env）

```bash
# LLM 提供商
OPENROUTER_API_KEY=sk-or-...
ANTHROPIC_API_KEY=sk-ant-...
NOUS_API_KEY=...

# 工具 API
FIRECRAWL_API_KEY=...          # 网页搜索与爬取
FAL_KEY=...                    # 图像生成
BROWSERBASE_API_KEY=...        # 浏览器自动化

# 消息平台
TELEGRAM_BOT_TOKEN=...
DISCORD_BOT_TOKEN=...
SLACK_BOT_TOKEN=...

# 记忆系统
HONCHO_API_KEY=...

# 终端后端
TERMINAL_BACKEND=local
MESSAGING_CWD=/home/user       # 消息模式的工作目录
```

---

## 5. 消息网关使用

### 5.1 启动网关

```bash
hermes gateway
```

网关进程同时服务所有已配置的消息平台。

### 5.2 Telegram 配置

1. 向 @BotFather 创建 Bot，获取 Token
2. 设置环境变量：`TELEGRAM_BOT_TOKEN=your_token`
3. 启动网关：`hermes gateway`
4. 向 Bot 发送消息开始对话

### 5.3 Discord 配置

1. 在 Discord Developer Portal 创建应用
2. 创建 Bot 并获取 Token
3. 设置环境变量：`DISCORD_BOT_TOKEN=your_token`
4. 启动网关

### 5.4 多平台特性

| 特性 | 描述 |
|------|------|
| 跨平台会话连续性 | 在 Telegram 开始对话，在 Discord 继续 |
| 语音备忘录转录 | 发送语音消息自动转文字 |
| 文件处理 | 发送文件、图片由智能体处理 |
| 安全设备配对 | DM 配对防止未授权访问 |
| 命令审批 | 危险操作需用户确认 |

---

## 6. 定时任务（Cron）

### 6.1 创建定时任务

在对话中自然语言描述：

```
You: 每天早上 9 点帮我检查 GitHub 仓库的新 issues 并总结
```

或使用斜杠命令：

```
/cron "0 9 * * *" 检查 GitHub 仓库的新 issues 并总结到 Telegram
```

### 6.2 定时任务管理

```
/cron list          # 列出所有定时任务
/cron delete <id>   # 删除指定任务
```

### 6.3 调度格式

| 格式 | 示例 |
|------|------|
| Cron 表达式 | `0 9 * * *`（每天 9:00） |
| 间隔 | `every 30m`（每 30 分钟） |
| 一次性 | `once 2024-12-25T09:00:00` |
| 自然语言 | `every day at 9am` |

---

## 7. 技能系统

### 7.1 浏览与安装技能

```
/skills                    # 浏览可用技能
/skills search github      # 搜索 GitHub 相关技能
/skills install <name>     # 安装技能
```

### 7.2 内置技能分类

| 分类 | 技能数 | 示例 |
|------|--------|------|
| 自主 AI 智能体 | 5+ | Claude Code、Codex 集成 |
| 软件开发 | 8+ | 代码审查、TDD、调试 |
| 研究 | 6+ | arXiv、论文写作、领域情报 |
| 生产力 | 8+ | Google Workspace、Notion、OCR |
| GitHub | 7+ | PR 工作流、代码审查、仓库管理 |
| MLOps | 5+ | 训练、推理、评估 |
| 媒体 | 3+ | YouTube、音乐 |
| 智能家居 | 1+ | Philips Hue 控制 |
| 创意 | 3+ | ASCII 艺术、Excalidraw |

### 7.3 自定义技能

智能体在完成复杂任务后会自动创建技能，也可手动创建：

```markdown
<!-- ~/.hermes/skills/my-skill/SKILL.md -->
---
name: my-custom-skill
description: 自定义技能描述
tags: [custom, example]
---

# 技能指令

当用户请求 XXX 时，执行以下步骤：
1. ...
2. ...
```

---

## 8. 编程式使用（API 调用）

### 8.1 基本用法

```python
from run_agent import AIAgent

agent = AIAgent(
    model="anthropic/claude-sonnet-4-20250514",
    max_iterations=10,
    quiet_mode=True,
)

# 简单对话
response = agent.chat("解释什么是递归")
print(response)

# 完整对话（返回详细结果）
result = agent.run_conversation(
    user_message="帮我写一个 Python 排序函数",
    system_message="你是一个 Python 专家",
)
print(result["final_response"])
print(result["messages"])  # 完整对话历史
```

### 8.2 自定义工具集

```python
agent = AIAgent(
    model="anthropic/claude-sonnet-4-20250514",
    enabled_toolsets=["terminal", "file"],  # 仅启用终端和文件工具
    disabled_toolsets=["browser"],          # 禁用浏览器
    skip_context_files=True,
    skip_memory=True,
)
```

### 8.3 会话管理

```python
from hermes_state import SessionDB

db = SessionDB(db_path="path/to/sessions.db")

# 创建会话
db.create_session("session-1", source="api", model="anthropic/claude-sonnet-4-20250514")

# 追加消息
db.append_message("session-1", role="user", content="Hello")

# 搜索历史
results = db.search_messages("docker", source_filter=["telegram"])

# Token 统计
db.update_token_counts("session-1", input_tokens=100, output_tokens=50)
```

---

## 9. 批量处理与训练

### 9.1 批量轨迹生成

```bash
python batch_runner.py \
    --config datagen-config-examples/web_research.yaml \
    --workers 4 \
    --output ./trajectories/
```

### 9.2 轨迹压缩

```bash
python trajectory_compressor.py \
    --config datagen-config-examples/trajectory_compression.yaml \
    --input ./trajectories/ \
    --output ./compressed/
```

### 9.3 RL 训练

```bash
python rl_cli.py \
    --environment hermes_base \
    --model anthropic/claude-sonnet-4-20250514 \
    --episodes 100
```

---

## 10. 终端后端切换

### 10.1 配置后端

```yaml
# config.yaml
terminal:
  backend: docker
  docker:
    image: "python:3.11-slim"
    volumes:
      - "/home/user/projects:/workspace"
    memory: "4g"
```

### 10.2 后端对比

| 后端 | 成本 | 持久化 | GPU | 适用场景 |
|------|------|--------|-----|---------|
| local | 免费 | 永久 | 本机 | 开发调试 |
| docker | 免费 | 容器内 | 可映射 | 隔离执行 |
| ssh | VPS 费用 | 永久 | 远程 | 远程服务器 |
| modal | 按需计费 | 临时 | A100 等 | GPU 训练 |
| daytona | 按需计费 | 休眠恢复 | 无 | Serverless 开发 |
| singularity | HPC 费用 | 共享存储 | 集群 GPU | 科研计算 |

---

## 11. 主题与自定义

### 11.1 内置主题

```bash
/skin default    # 经典金色/可爱风
/skin ares       # 深红/青铜战神主题
/skin mono       # 灰度极简
/skin slate      # 冷蓝开发者主题
```

### 11.2 自定义主题

创建 `~/.hermes/skins/mytheme.yaml`：

```yaml
name: mytheme
description: 我的自定义主题
colors:
  banner_border: "#FF6600"
  banner_title: "#FFFFFF"
spinner:
  thinking_verbs: ["思考中", "分析中", "推理中"]
branding:
  agent_name: "My Agent"
  prompt_symbol: ">"
```

激活：`/skin mytheme`

---

## 12. 故障排查

### 12.1 常用诊断

```bash
hermes doctor         # 自动诊断
hermes config show    # 查看当前配置
```

### 12.2 常见问题

| 问题 | 解决方案 |
|------|---------|
| API 密钥无效 | 检查 `~/.hermes/.env` 中的密钥 |
| 模型不可用 | 运行 `hermes model` 重新选择 |
| 工具不工作 | 运行 `hermes tools` 检查启用状态 |
| 网关连接失败 | 检查 Bot Token 和网络连接 |
| 安装失败 | 确保 Python 3.11+ 和 git 已安装 |
