# Hermes Agent 核心功能

> 深度解析 Hermes Agent 的核心能力与实现原理

## 1. 智能体对话循环（Agent Loop）

### 1.1 ReAct 模式实现

Hermes Agent 的核心是一个 **ReAct（Reasoning + Acting）** 循环，模型在推理与工具调用之间交替：

```
用户输入 → 模型推理 → 工具调用 → 观察结果 → 模型推理 → ... → 最终回复
```

**关键实现细节：**

- **同步循环**：`AIAgent.run_conversation()` 采用同步设计，避免异步复杂性
- **迭代预算**：`max_iterations`（默认 90）和 `iteration_budget.remaining` 双重限制
- **推理提取**：支持多种推理格式 — `reasoning_content`、`reasoning` 字段、`reasoning_details`
- **自动重试**：API 错误时通过 `tenacity` 实现指数退避重试
- **上下文压缩**：当消息超出模型上下文窗口时自动压缩

### 1.2 双引擎架构

| 引擎 | 类 | 用途 |
|------|-----|------|
| **完整引擎** | `AIAgent` (run_agent.py) | CLI/Gateway 的主对话循环，包含会话管理、记忆、上下文文件 |
| **轻量引擎** | `HermesAgentLoop` (agent/) | 子智能体和批处理，最小依赖，专注工具调用 |

### 1.3 多提供商支持

通过 `litellm` 和原生 SDK 适配器，支持无缝切换 LLM 提供商：

- **OpenRouter**：200+ 模型统一接口
- **Anthropic**：原生 SDK，支持提示词缓存
- **自定义端点**：任何 OpenAI 兼容 API
- **运行时切换**：`hermes model` 即时更换模型，无需代码修改
- **自动回退**：主模型失败时切换到备用模型

---

## 2. 工具系统（30+ 工具）

### 2.1 工具注册表架构

集中式注册表 (`tools/registry.py`) 是工具系统的核心：

```python
registry.register(
    name="tool_name",           # 工具名称
    toolset="toolset_name",     # 所属工具集
    schema={...},               # OpenAI 函数调用格式的 Schema
    handler=handler_fn,         # 执行函数
    check_fn=check_fn,          # 运行时可用性检查
    requires_env=["API_KEY"],   # 必需的环境变量
)
```

**工具生命周期：**
1. **发现**：`model_tools._discover_tools()` 导入所有工具模块
2. **注册**：每个模块在导入时调用 `registry.register()`
3. **过滤**：根据工具集配置和可用性检查筛选
4. **Schema 生成**：生成 OpenAI 兼容的 JSON Schema
5. **分发**：`handle_function_call()` 路由到对应 handler
6. **结果处理**：所有 handler 返回 JSON 字符串

### 2.2 核心工具详解

#### 终端工具（Terminal）

```
terminal(command, background=false, check_interval=null)
```

- 支持 6 种后端（local、docker、ssh、modal、daytona、singularity）
- 后台执行模式 + 定期状态检查
- 进程注册表管理后台进程
- 与 mini-swe-agent 深度集成

#### 文件工具（File Tools）

```
read_file(path, offset=null, limit=null)  # 读取文件
write_file(path, content)                  # 写入文件
patch(path, changes, flags)                # 精确补丁
search_files(pattern, path)                # 文件搜索
```

- 支持偏移量和限制的大文件读取
- 补丁模式支持精确的查找替换
- 搜索支持 glob 和正则表达式

#### 浏览器工具（Browser）

```
browser(action, url=null, selector=null, text=null)
```

- 基于 Playwright 的浏览器自动化
- 页面截图 + 视觉分析
- 表单填写、点击、导航
- Browserbase 云端浏览器支持

#### 网页工具（Web Tools）

```
web_search(query, num_results=5)     # 网页搜索
web_crawl(url)                        # 网页爬取
web_extract(url, schema)              # 结构化提取
```

- Firecrawl API 驱动
- 自动 HTML 到 Markdown 转换
- 结构化数据提取

#### 代码执行工具（Code Execution）

```
execute_code(language, code)
```

- 沙箱化代码执行
- 支持 Python 和其他语言
- RPC 方式调用工具，零上下文成本

#### 委派工具（Delegate）

```
delegate(task, tools=null, model=null)
```

- 生成隔离子智能体
- 并行工作流支持
- 独立工具集和模型选择

#### 记忆工具（Memory）

```
memory(action, content=null, key=null)
```

- 持久化存储在 `~/.hermes/memory.md`
- 智能体主动记忆（周期性提醒）
- 结构化键值存储

#### MCP 工具

```
mcp(server, method, params)
```

- Model Context Protocol 客户端
- 连接任意 MCP 服务器
- 动态工具发现和调用

---

## 3. 技能系统（Skill System）

### 3.1 技能的本质

技能是**结构化的提示词模板**，让智能体获得特定领域的专业能力：

```
skills/
├── software-development/
│   ├── code-review/
│   │   ├── SKILL.md          # 技能指令
│   │   └── DESCRIPTION.md    # 元数据描述
│   ├── tdd/
│   └── systematic-debugging/
├── research/
│   ├── arxiv/
│   └── paper-writing/
└── ...
```

### 3.2 技能分类（84+ 技能）

| 分类 | 数量 | 代表技能 |
|------|------|---------|
| 软件开发 | 8+ | 代码审查、TDD、系统化调试、子智能体开发 |
| 研究 | 6+ | arXiv 论文、学术写作、领域情报 |
| 生产力 | 8+ | Google Workspace、Notion、OCR、PowerPoint |
| GitHub | 7+ | PR 工作流、代码审查、仓库管理 |
| MLOps | 5+ | GRPO 训练、推理、评估、向量数据库 |
| 自主 AI | 5+ | Claude Code、Codex、OpenCode 集成 |
| 媒体 | 3+ | YouTube 分析、音乐 |
| 创意 | 3+ | ASCII 艺术、Excalidraw 绘图 |
| 智能家居 | 1+ | Philips Hue 控制 |
| 游戏 | 2+ | Minecraft、Pokemon |

### 3.3 自我进化机制

```
任务完成 → 智能体评估复杂度 → 自动创建技能
                                    ↓
后续类似任务 → 加载技能 → 执行 → 评估效果 → 改进技能
```

- **自动技能创建**：复杂任务完成后，智能体主动创建可复用技能
- **技能自我改进**：使用过程中根据效果反馈优化技能
- **技能共享**：兼容 [agentskills.io](https://agentskills.io) 开放标准

### 3.4 技能激活方式

1. **斜杠命令**：`/skills` 浏览和安装
2. **自动加载**：匹配上下文自动激活
3. **条件触发**：基于 SKILL.md 中定义的条件
4. **注入方式**：作为用户消息注入（保留提示词缓存）

---

## 4. 记忆与学习系统

### 4.1 多层记忆架构

```
┌──────────────────────────────────────┐
│         长期记忆（Memory）            │
│    ~/.hermes/memory.md               │
│    结构化知识、用户偏好、项目信息       │
├──────────────────────────────────────┤
│         会话记忆（Session Search）    │
│    SQLite FTS5 全文搜索              │
│    跨会话搜索 + LLM 摘要             │
├──────────────────────────────────────┤
│         用户建模（Honcho）            │
│    辩证推理式用户画像                 │
│    跨会话身份连续性                   │
├──────────────────────────────────────┤
│         技能记忆（Skills）            │
│    程序化记忆                        │
│    自动创建与改进                     │
└──────────────────────────────────────┘
```

### 4.2 记忆管理

- **主动记忆提醒**：智能体周期性被提醒持久化重要信息
- **FTS5 搜索**：SQLite 全文搜索引擎，支持特殊字符（C++、引号等）
- **跨平台连续性**：记忆在 CLI、Telegram、Discord 等平台间共享
- **LLM 摘要**：搜索结果自动经 LLM 摘要处理

### 4.3 Honcho 集成

- **AI 原生用户建模**：通过辩证推理构建用户画像
- **会话映射**：跨会话身份追踪
- **上下文感知**：基于用户历史提供个性化回复
- **会话策略**：per-session 或 per-project 粒度

---

## 5. 多平台消息网关

### 5.1 统一网关架构

```
                    ┌─────────────┐
  Telegram ────────→│             │
  Discord ─────────→│   Gateway   │────→ AIAgent
  Slack ───────────→│   (run.py)  │
  WhatsApp ────────→│             │
  Signal ──────────→│             │
  Email ───────────→│             │
  Home Assistant ──→│             │
                    └─────────────┘
```

### 5.2 支持平台详情

| 平台 | 功能 |
|------|------|
| **Telegram** | 完整 Bot API、文档/媒体、格式化、语音转录 |
| **Discord** | 斜杠命令、Opus 音频、Bot 过滤、媒体元数据 |
| **Slack** | Webhook/Bot API、消息线程 |
| **WhatsApp** | WhatsApp Business API |
| **Signal** | Signal CLI 集成 |
| **Email** | SMTP/IMAP 支持 |
| **Home Assistant** | 智能家居集成 |

### 5.3 高级特性

- **会话连续性**：跨平台保持对话上下文
- **话题隔离**：不同话题独立会话
- **中断处理**：支持中断当前任务
- **设备配对**：安全的 DM 配对机制
- **命令审批**：危险操作审批流程
- **后台通知**：长时间任务完成后主动通知

---

## 6. 定时任务调度（Cron）

### 6.1 调度系统架构

```python
# cron/scheduler.py
class CronScheduler:
    def run(self):
        while True:
            due_jobs = self.get_due_jobs()
            for job in due_jobs:
                result = self.execute_job(job)
                self.deliver_result(result, job.platform)
```

### 6.2 调度格式支持

| 格式 | 示例 | 说明 |
|------|------|------|
| Cron 表达式 | `0 9 * * 1-5` | 工作日每天 9:00 |
| 间隔 | `every 30m` | 每 30 分钟 |
| 一次性 | `once 2024-12-25T09:00` | 指定时间执行一次 |
| ISO 时间戳 | `2024-12-25T09:00:00+08:00` | 精确时间 |

### 6.3 结果交付

任务结果可以交付到任意已配置的消息平台：
- Telegram 消息
- Discord 频道
- Slack 通道
- Email 邮件

---

## 7. 子智能体委派

### 7.1 委派机制

```python
# tools/delegate_tool.py
delegate(
    task="分析这 100 个文件的代码质量",
    tools=["file", "terminal"],
    model="anthropic/claude-haiku-4-5-20251001",
)
```

### 7.2 特性

- **隔离执行**：独立的上下文和工具集
- **并行工作流**：多个子智能体并行执行
- **模型选择**：可为子任务选择更高效的模型
- **零上下文成本**：子智能体不污染主对话上下文
- **Git Worktree**：可选的 Git 工作树隔离

---

## 8. 安全系统

### 8.1 多层安全防护

```
┌─ 第一层：提示注入检测 ─────────────────────┐
│  agent/prompt_builder.py                    │
│  检测 15+ 种注入模式                        │
├─ 第二层：命令审批 ─────────────────────────┤
│  tools/approval.py                          │
│  危险命令需用户确认                          │
├─ 第三层：设备配对 ─────────────────────────┤
│  gateway/pairing.py                         │
│  防止未授权消息平台访问                      │
├─ 第四层：容器隔离 ─────────────────────────┤
│  Docker/Daytona/Singularity 后端            │
│  执行环境与宿主机隔离                        │
└─────────────────────────────────────────────┘
```

### 8.2 提示注入检测模式

- `ignore previous instructions` — 指令覆盖
- `system prompt override` — 系统提示覆盖
- HTML 注释注入 (`<!-- -->`)
- 不可见 Unicode 字符
- `display:none` 隐藏内容
- API 密钥外泄 (`curl $API_KEY`)
- `cat ~/.env` 敏感文件读取
- `translate to bash and execute` 执行绕过

### 8.3 命令审批

```python
# 自动检测的危险命令模式
dangerous_patterns = [
    "rm -rf", "dd if=", "mkfs",
    "DROP TABLE", "DELETE FROM",
    "> /dev/sda",
    "chmod 777", "chown root",
    # ...
]
```

---

## 9. 上下文管理

### 9.1 上下文文件系统

项目级上下文文件影响每次对话：

```
project/
├── AGENTS.md         # 智能体开发指南
├── .hermes/
│   └── context.md    # 项目上下文
└── ...
```

### 9.2 上下文压缩

当对话超出模型上下文窗口时自动触发：

```python
# agent/context_compressor.py
class ContextCompressor:
    def compress(self, messages, target_tokens):
        # 保留首尾消息
        # LLM 摘要中间消息
        # 保持对话连贯性
```

### 9.3 提示词缓存

Anthropic 提示词缓存策略：
- 系统提示词在会话中保持不变（缓存命中）
- 工具 Schema 在会话中保持不变（缓存命中）
- 仅在压缩时重建（最小化缓存失效）
- 技能指令作为用户消息注入（不影响系统缓存）

---

## 10. 研究与训练能力

### 10.1 批量轨迹生成

```yaml
# datagen-config-examples/web_research.yaml
environment: web-research
toolsets: [terminal, web, file]
workers: 4
model: anthropic/claude-sonnet-4-20250514
batch_size: 100
output_dir: ./trajectories/
```

### 10.2 轨迹压缩

为训练下一代工具调用模型而设计：
- 目标 Token 预算（8K-32K）
- 保护关键 Turn（首/尾、系统/用户/助手）
- LLM 摘要式压缩
- 多工作者并行处理

### 10.3 RL 环境

```
environments/
├── hermes_base_env.py      # 基础环境类
├── agentic_opd_env.py      # OpenAI SWE 环境
├── web_research_env.py     # 网页研究基准
├── terminal_test/          # 终端测试环境
├── terminalbench_2/        # 终端基准 v2
├── tblite/                 # 轻量基准
└── yc_bench/               # YC 基准套件
```

### 10.4 Atropos 集成

- GRPO（Group Relative Policy Optimization）训练
- Wandb 实验追踪
- 可插拔的奖励函数
- 多环境并行训练

---

## 11. 功能特性矩阵

| 功能 | CLI | Telegram | Discord | Slack | API |
|------|:---:|:--------:|:-------:|:-----:|:---:|
| 文本对话 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 工具调用 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 文件处理 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 语音交互 | ✅ | ✅ | ✅ | ❌ | ❌ |
| 技能系统 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 记忆系统 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 定时任务 | ❌ | ✅ | ✅ | ✅ | ❌ |
| 子智能体 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 命令审批 | ✅ | ✅ | ✅ | ✅ | ❌ |
| 主题皮肤 | ✅ | ❌ | ❌ | ❌ | ❌ |
| 会话搜索 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 批量处理 | ❌ | ❌ | ❌ | ❌ | ✅ |
