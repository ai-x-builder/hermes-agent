# Hermes Agent 项目架构

> 深度分析 Hermes Agent 的系统架构、模块设计与技术栈

## 1. 项目概览

Hermes Agent 是由 [Nous Research](https://nousresearch.com) 构建的**自我进化 AI 智能体框架**。它不仅仅是一个 CLI 聊天工具，而是一个具备闭环学习能力、多平台通信、可调度自动化、子智能体委派以及研究级训练支持的完整智能体系统。

**技术栈核心：**
- **主语言：** Python 3.11+（占比 95%+）
- **辅助语言：** TypeScript/JavaScript（文档站 + 浏览器工具）
- **构建系统：** setuptools + pyproject.toml
- **包管理：** uv (Astral)
- **测试框架：** pytest + pytest-asyncio + pytest-xdist
- **文档框架：** Docusaurus 3.x

---

## 2. 顶层目录结构

```
hermes-agent/
├── run_agent.py              # AIAgent 类 — 核心对话循环（入口点）
├── cli.py                    # HermesCLI — 交互式终端界面
├── model_tools.py            # 工具发现、注册与分发
├── toolsets.py               # 工具集定义与管理
├── hermes_state.py           # SessionDB — SQLite 会话存储 + FTS5 搜索
├── hermes_constants.py       # 全局常量
├── hermes_time.py            # 时间工具
├── utils.py                  # 通用工具函数
├── batch_runner.py           # 批量轨迹生成
├── trajectory_compressor.py  # 轨迹压缩（用于训练）
├── mini_swe_runner.py        # Mini-SWE 智能体运行器
├── rl_cli.py                 # 强化学习训练 CLI
├── toolset_distributions.py  # 工具集分布逻辑
│
├── agent/                    # 智能体内核模块
├── tools/                    # 30+ 工具实现
├── gateway/                  # 多平台消息网关
├── hermes_cli/               # CLI 子命令与配置
├── cron/                     # 定时任务调度
├── honcho_integration/       # Honcho 记忆集成
├── acp_adapter/              # Agent Client Protocol 适配器
├── skills/                   # 84+ 技能目录
├── optional-skills/          # 17 可选技能
├── mini-swe-agent/           # 终端执行后端（git 子模块）
├── tinker-atropos/           # RL 训练集成（git 子模块）
├── plans/                    # 智能体规划模块
├── environments/             # 基准评测环境
├── tests/                    # 205+ 测试文件
├── website/                  # Docusaurus 文档站
├── scripts/                  # 实用脚本
└── assets/                   # 静态资源
```

---

## 3. 核心架构分层

Hermes Agent 采用**分层架构**设计，每层职责清晰：

```
┌─────────────────────────────────────────────────────┐
│                  用户接入层                           │
│   CLI (cli.py)  │  Gateway (gateway/)  │  ACP       │
├─────────────────────────────────────────────────────┤
│                  智能体引擎层                         │
│   AIAgent (run_agent.py) + HermesAgentLoop (agent/) │
├─────────────────────────────────────────────────────┤
│                  工具执行层                           │
│   Registry (tools/registry.py) + 30+ Tool Modules   │
├─────────────────────────────────────────────────────┤
│                  状态持久化层                         │
│   SessionDB (hermes_state.py) + Memory + Skills     │
├─────────────────────────────────────────────────────┤
│                  基础设施层                           │
│   Terminal Backends │ LLM Providers │ External APIs  │
└─────────────────────────────────────────────────────┘
```

### 3.1 用户接入层

负责接收用户输入，呈现输出结果。三个入口：

| 入口 | 模块 | 用途 |
|------|------|------|
| **CLI** | `cli.py` + `hermes_cli/` | 交互式终端，支持多行编辑、自动补全、流式输出 |
| **Gateway** | `gateway/` | 多平台消息适配器（Telegram、Discord、Slack 等） |
| **ACP** | `acp_adapter/` | IDE 集成协议（VS Code、Zed、JetBrains） |

### 3.2 智能体引擎层

核心对话循环，实现 ReAct 模式的工具调用链：

```python
# 简化的核心循环 (run_agent.py)
while api_call_count < max_iterations:
    response = client.chat.completions.create(
        model=model, messages=messages, tools=tool_schemas
    )
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args)
            messages.append(tool_result_message(result))
    else:
        return response.content  # 最终回复
```

- **AIAgent** (`run_agent.py`): 完整对话管理，包括上下文构建、提供商切换、会话持久化
- **HermesAgentLoop** (`agent/`): 轻量级智能体循环，用于子智能体和批处理
- **PromptBuilder** (`agent/prompt_builder.py`): 系统提示词组装，含安全检测
- **ContextCompressor** (`agent/context_compressor.py`): 自动上下文压缩

### 3.3 工具执行层

集中式工具注册表模式：

```
tools/registry.py  ← 无外部依赖，所有工具模块导入它
       ↑
tools/*.py          ← 每个工具文件在导入时调用 registry.register()
       ↑
model_tools.py      ← 导入 registry，触发工具发现
       ↑
run_agent.py / cli.py / batch_runner.py
```

**关键工具模块：**

| 工具 | 文件 | 功能 |
|------|------|------|
| terminal | `terminal_tool.py` | 命令执行（6 种后端） |
| browser | `browser_tool.py` | 浏览器自动化 + 截图 |
| file_tools | `file_tools.py` | 文件读写/搜索/补丁 |
| web_tools | `web_tools.py` | 网页搜索/爬取/提取 |
| memory | `memory_tool.py` | 持久化记忆管理 |
| delegate | `delegate_tool.py` | 子智能体委派 |
| mcp | `mcp_tool.py` | Model Context Protocol 集成 |
| code_execution | `code_execution_tool.py` | 沙箱代码执行 |
| skills | `skills_tool.py` + `skill_manager_tool.py` | 技能系统 |

### 3.4 状态持久化层

- **SessionDB** (`hermes_state.py`): SQLite 数据库，FTS5 全文搜索
  - 会话生命周期管理（创建、更新、结束）
  - 消息存储与检索
  - Token 统计与成本追踪
  - 父子会话关系
- **Memory**: 持久化记忆存储在 `~/.hermes/memory.md`
- **Skills**: 技能文件存储在 `~/.hermes/skills/`
- **Config**: YAML 配置文件 `~/.hermes/config.yaml`

### 3.5 基础设施层

**终端后端（6 种）：**

| 后端 | 适用场景 |
|------|---------|
| Local | 本地开发 |
| Docker | 容器隔离 |
| SSH | 远程服务器 |
| Modal | Serverless GPU |
| Daytona | Serverless 持久化 |
| Singularity | HPC 集群 |

**LLM 提供商：**
- OpenRouter（200+ 模型）
- Anthropic（Claude 系列）
- Nous Portal
- z.ai/GLM、Kimi/Moonshot、MiniMax
- 自定义 OpenAI 兼容端点

---

## 4. 关键设计模式

### 4.1 注册表模式（Tool Registry）

所有工具通过 `registry.register()` 在导入时自注册，实现松耦合：

```python
# tools/your_tool.py
from tools.registry import registry

registry.register(
    name="example_tool",
    toolset="example",
    schema={...},          # OpenAI 函数调用格式
    handler=lambda args, **kw: ...,
    check_fn=check_requirements,  # 运行时可用性检查
    requires_env=["API_KEY"],
)
```

### 4.2 适配器模式（Gateway Platforms）

每个消息平台有独立的适配器，统一的消息接口：

```
gateway/
├── run.py           # 中央调度器
├── session.py       # 会话管理
├── delivery.py      # 消息路由与格式化
├── config.py        # 平台配置
├── pairing.py       # 安全设备配对
└── platforms/
    ├── telegram.py
    ├── discord.py
    ├── slack.py
    ├── whatsapp.py
    ├── signal.py
    └── homeassistant.py
```

### 4.3 回调模式（CLI Callbacks）

CLI 通过回调函数与智能体引擎解耦：

```python
# hermes_cli/callbacks.py
callbacks = {
    "clarify": clarify_callback,     # 追问用户
    "sudo": sudo_callback,           # 权限提升
    "approval": approval_callback,   # 危险命令审批
}
```

### 4.4 提示词缓存（Prompt Caching）

Anthropic 提示词缓存策略，显著降低成本：
- 系统提示词在会话中不可变
- 工具集在会话中不可变
- 仅在上下文压缩时重建缓存

---

## 5. 数据流架构

### 5.1 CLI 模式数据流

```
用户输入 → HermesCLI.process_input()
    → 命令? → process_command()
    → 对话? → AIAgent.run_conversation()
        → PromptBuilder 构建系统提示词
        → LLM API 调用
        → 工具调用循环
            → model_tools.handle_function_call()
            → tools/registry.dispatch()
            → 工具执行 → 结果追加到消息
        → 最终回复 → Rich 格式化输出
```

### 5.2 Gateway 模式数据流

```
平台消息 → Platform Adapter (telegram/discord/...)
    → gateway/run.py 消息分发
    → SessionStore 获取/创建会话
    → AIAgent.run_conversation()
    → 结果通过 delivery.py 路由回平台
```

---

## 6. 入口点与命令

```toml
# pyproject.toml 定义的三个入口点
[project.scripts]
hermes = "hermes_cli.main:main"        # 主 CLI（hermes 命令）
hermes-agent = "run_agent:main"        # 智能体运行器
hermes-acp = "acp_adapter.entry:main"  # ACP 协议服务器
```

---

## 7. 依赖架构

### 核心依赖（47 个）

| 类别 | 依赖包 | 用途 |
|------|--------|------|
| LLM | openai, anthropic, litellm | LLM API 客户端 |
| CLI/UI | rich, prompt_toolkit, typer | 终端界面 |
| HTTP | httpx, requests | 网络请求 |
| 配置 | pydantic, pyyaml, python-dotenv | 数据验证与配置 |
| 模板 | jinja2 | 提示词模板 |
| 重试 | tenacity | 指数退避重试 |

### 可选依赖组（15 个）

| 组名 | 用途 | 关键依赖 |
|------|------|---------|
| messaging | 消息平台 | python-telegram-bot, discord.py, slack-sdk |
| modal | Serverless GPU | modal |
| daytona | Serverless VM | daytona-sdk |
| voice | 语音交互 | sounddevice, numpy |
| mcp | MCP 协议 | mcp |
| honcho | AI 记忆 | honcho-ai |
| rl | 强化学习 | atropos, tinker, wandb |
| all | 全部安装 | — |

---

## 8. 安全架构

### 8.1 命令审批系统

```python
# tools/approval.py — 危险命令检测
dangerous_patterns = [
    "rm -rf", "dd if=", "mkfs",
    "DROP TABLE", "DELETE FROM",
    "> /dev/sda", ...
]
```

### 8.2 提示注入防护

`agent/prompt_builder.py` 内置多种注入检测：
- "ignore previous instructions" 模式
- HTML 注释注入
- 不可见 Unicode 字符
- API 密钥外泄尝试（`curl $API_KEY`）
- 系统提示覆盖尝试

### 8.3 安全设备配对

`gateway/pairing.py` 实现 DM 配对机制，防止未授权访问。

---

## 9. 测试架构

```
tests/                    # 205+ 测试文件, ~3000 测试用例
├── agent/                # 智能体内核测试
├── gateway/              # 65+ 网关平台测试
├── tools/                # 60+ 工具测试
├── hermes_cli/           # 25+ CLI 命令测试
├── cron/                 # 定时任务测试
├── acp/                  # ACP 协议测试
├── honcho_integration/   # 记忆系统测试
├── integration/          # 端到端集成测试
├── skills/               # 技能系统测试
└── fakes/                # Mock 对象与 Fixtures
```

**关键测试设施：**
- `_isolate_hermes_home` 自动 fixture — 隔离 `~/.hermes/` 到临时目录
- Mock LLM 服务器 — 模拟完整的 API 响应
- 30 秒超时强制执行
- pytest-xdist 并行执行

---

## 10. 架构总结

Hermes Agent 的架构体现了以下核心设计理念：

1. **模块化**: 工具注册表、平台适配器、终端后端均可独立扩展
2. **松耦合**: 层间通过回调和消息传递交互，避免硬依赖
3. **可配置性**: 从 LLM 提供商到终端后端，几乎所有组件可通过配置切换
4. **安全第一**: 多层安全防护，从命令审批到提示注入检测
5. **自我进化**: 技能系统 + 记忆系统构成闭环学习能力
6. **多入口**: CLI、消息网关、ACP 三个入口覆盖不同使用场景
