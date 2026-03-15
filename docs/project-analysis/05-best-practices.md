# Hermes Agent 最佳实践

> 开发、部署与使用 Hermes Agent 的最佳实践指南

## 1. 开发最佳实践

### 1.1 工具开发规范

#### 创建新工具的标准流程

新增工具需要修改 **3 个文件**：

**步骤 1：创建工具文件** `tools/your_tool.py`

```python
import json
import os
from tools.registry import registry


def check_requirements() -> bool:
    """运行时可用性检查 — 缺少依赖时优雅降级"""
    return bool(os.getenv("EXAMPLE_API_KEY"))


def example_tool(param: str, task_id: str = None) -> str:
    """所有 handler 必须返回 JSON 字符串"""
    try:
        result = do_work(param)
        return json.dumps({"success": True, "data": result})
    except Exception as e:
        return json.dumps({"error": str(e)})


registry.register(
    name="example_tool",
    toolset="example",
    schema={
        "name": "example_tool",
        "description": "清晰描述工具用途",
        "parameters": {
            "type": "object",
            "properties": {
                "param": {"type": "string", "description": "参数描述"}
            },
            "required": ["param"],
        },
    },
    handler=lambda args, **kw: example_tool(
        param=args.get("param", ""),
        task_id=kw.get("task_id"),
    ),
    check_fn=check_requirements,
    requires_env=["EXAMPLE_API_KEY"],
)
```

**步骤 2：注册到工具发现** — 在 `model_tools.py` 的 `_discover_tools()` 列表中添加导入

**步骤 3：添加到工具集** — 在 `toolsets.py` 中将工具添加到对应工具集

#### 工具开发要点

| 要点 | 说明 |
|------|------|
| 返回 JSON 字符串 | 所有 handler 必须返回 `json.dumps()` 的结果 |
| 优雅降级 | `check_fn` 在缺少依赖时返回 `False`，而非抛异常 |
| 环境变量声明 | `requires_env` 列出所有必需的环境变量 |
| Schema 规范 | 使用 OpenAI 函数调用格式的 JSON Schema |
| 错误处理 | 捕获异常并返回结构化错误信息 |

### 1.2 提示词缓存保护

**核心原则：绝不在对话中途修改缓存内容**

```
✅ 正确做法：
- 系统提示词在会话开始时构建，之后不变
- 工具集在会话开始时确定，之后不变
- 技能指令作为用户消息注入（不影响系统缓存）

❌ 错误做法：
- 对话中途重新加载记忆
- 对话中途切换工具集
- 对话中途重建系统提示词
```

缓存失效会导致 API 成本大幅增加。唯一允许修改上下文的时机是**上下文压缩**。

### 1.3 测试编写规范

#### 必须遵循的测试原则

```python
# 1. 使用隔离 fixture — 绝不写入 ~/.hermes/
@pytest.fixture(autouse=True)
def _isolate_hermes_home(tmp_path, monkeypatch):
    fake_home = tmp_path / "hermes_test"
    fake_home.mkdir()
    monkeypatch.setenv("HERMES_HOME", str(fake_home))

# 2. Mock 外部 API — 不依赖真实服务
@patch("tools.web_tools.firecrawl_search")
def test_web_search(mock_search):
    mock_search.return_value = [{"title": "test", "url": "..."}]
    result = web_search("query")
    assert "test" in result

# 3. 设置超时 — 防止测试卡死
@pytest.fixture(autouse=True)
def _enforce_test_timeout():
    signal.alarm(30)
    yield
    signal.alarm(0)

# 4. 使用 tmp_path — 不硬编码文件路径
def test_session_db(tmp_path):
    db = SessionDB(db_path=str(tmp_path / "test.db"))
    # ...
```

#### 测试分层策略

| 层级 | 标记 | 运行时机 | 范围 |
|------|------|---------|------|
| 单元测试 | 无标记 | 每次提交 | Mock 所有外部依赖 |
| 集成测试 | `@pytest.mark.integration` | 手动触发 | 需要真实 API |
| 基准测试 | 在 `environments/` | 手动触发 | 端到端评估 |

### 1.4 配置管理规范

#### 添加新配置项

```python
# 1. 在 hermes_cli/config.py 的 DEFAULT_CONFIG 中添加默认值
DEFAULT_CONFIG = {
    # ... 现有配置
    "new_feature": {
        "enabled": False,
        "option": "default_value",
    },
}

# 2. 递增 _config_version 触发迁移
_config_version = 6  # 从 5 增加到 6

# 3. 如果需要环境变量，添加到 OPTIONAL_ENV_VARS
OPTIONAL_ENV_VARS = {
    "NEW_API_KEY": {
        "description": "API key for new feature",
        "prompt": "New Feature API Key",
        "url": "https://example.com",
        "password": True,
        "category": "tool",
    },
}
```

#### 配置加载的三套系统

| 加载器 | 使用场景 | 位置 |
|--------|---------|------|
| `load_cli_config()` | CLI 模式 | `cli.py` |
| `load_config()` | setup/tools 命令 | `hermes_cli/config.py` |
| 直接 YAML 加载 | Gateway 模式 | `gateway/run.py` |

**注意**：修改配置逻辑时，确保三套系统保持一致。

---

## 2. 部署最佳实践

### 2.1 生产环境部署

#### VPS 部署清单

```bash
# 1. 系统准备
sudo apt update && sudo apt install -y git python3.11 nodejs npm

# 2. 安装 Hermes Agent
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc

# 3. 配置环境变量
cp ~/.hermes/.env.example ~/.hermes/.env
# 编辑 .env 填入 API 密钥

# 4. 运行设置向导
hermes setup

# 5. 使用 systemd 管理网关进程
sudo tee /etc/systemd/system/hermes-gateway.service << 'EOF'
[Unit]
Description=Hermes Agent Gateway
After=network.target

[Service]
Type=simple
User=hermes
WorkingDirectory=/home/hermes
ExecStart=/home/hermes/.local/bin/hermes gateway
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable --now hermes-gateway
```

#### 安全加固

```bash
# 1. 非 root 用户运行
useradd -m -s /bin/bash hermes
su - hermes

# 2. 限制文件权限
chmod 600 ~/.hermes/.env
chmod 700 ~/.hermes/

# 3. 使用 Docker 后端隔离执行
# config.yaml
terminal:
  backend: docker
  docker:
    image: "python:3.11-slim"
    network: none           # 禁用网络（按需启用）
    read_only: true         # 只读文件系统
    memory: "512m"          # 内存限制

# 4. 启用命令审批
# 危险命令自动触发审批流程（默认启用）
```

### 2.2 高可用部署

```yaml
# docker-compose.yml
version: '3.8'
services:
  hermes-gateway:
    build: .
    restart: always
    env_file: .env
    volumes:
      - hermes-data:/home/hermes/.hermes
    healthcheck:
      test: ["CMD", "hermes", "doctor"]
      interval: 30s
      timeout: 10s
      retries: 3

  # 可选：Honcho 记忆服务
  honcho:
    image: plasticlabs/honcho:latest
    restart: always
    env_file: .env.honcho

volumes:
  hermes-data:
```

### 2.3 成本优化

| 策略 | 方法 | 节省幅度 |
|------|------|---------|
| 提示词缓存 | 使用 Anthropic + 保持缓存不失效 | 50-90% |
| 模型选择 | 简单任务用 Haiku，复杂任务用 Sonnet | 60-80% |
| 子智能体优化 | 委派任务给更小的模型 | 40-60% |
| Serverless 后端 | Modal/Daytona 按需计费 | 70-90% 空闲时 |
| 上下文压缩 | 自动压缩长对话 | 30-50% |

---

## 3. 使用最佳实践

### 3.1 对话技巧

#### 高效提示词

```
✅ 好的提示：
"重构 src/auth.py 中的 login() 函数，使用 JWT 替代 session，
保持向后兼容，添加单元测试"

❌ 差的提示：
"改一下登录"
```

**有效提示的要素：**
1. **具体文件/函数**：指明操作目标
2. **明确目标**：说明要达到的效果
3. **约束条件**：向后兼容、性能要求等
4. **验证标准**：需要测试、需要通过 lint 等

#### 多步任务拆分

```
# 对于复杂任务，分步骤指示更有效

Step 1: "先帮我分析 src/ 目录下所有 Python 文件的依赖关系"
Step 2: "基于分析结果，制定重构计划"
Step 3: "执行重构计划的第一阶段"
Step 4: "运行测试确认没有回归"
```

### 3.2 记忆管理

#### 什么应该存入记忆

```
✅ 适合记忆的内容：
- 用户偏好（编码风格、框架偏好）
- 项目关键信息（架构决策、部署流程）
- 常用命令和配置
- 团队约定和规范

❌ 不适合记忆的内容：
- 临时性的调试信息
- 一次性的任务细节
- 敏感凭证（使用 .env 存储）
```

#### 记忆组织建议

```markdown
<!-- ~/.hermes/memory.md 建议结构 -->

## 用户偏好
- 编码语言偏好：Python, TypeScript
- 测试框架：pytest
- 代码风格：Black 格式化, 4 空格缩进

## 项目信息
### project-alpha
- 技术栈：FastAPI + PostgreSQL + React
- 部署方式：Docker Compose on AWS
- CI/CD：GitHub Actions

## 常用命令
- 部署 staging：`./deploy.sh staging`
- 运行测试：`pytest tests/ -q -n auto`
```

### 3.3 技能使用策略

#### 技能选择矩阵

| 任务类型 | 推荐技能 | 优先级 |
|---------|---------|--------|
| 代码审查 | `code-review` | 高 |
| GitHub PR | `github/pr-workflow` | 高 |
| 调试问题 | `systematic-debugging` | 高 |
| 学术研究 | `arxiv` | 中 |
| 文档生成 | `paper-writing` | 中 |
| 数据分析 | `jupyter-live-kernel` | 中 |
| 项目管理 | `linear`, `notion` | 低 |

#### 技能开发建议

```markdown
<!-- 自定义技能模板 -->
---
name: my-deploy-skill
description: 标准化部署流程
tags: [devops, deploy]
conditions:
  - "deploy"
  - "部署"
---

# 部署流程

当用户请求部署时，按以下步骤执行：

## 前置检查
1. 确认目标环境（staging/production）
2. 检查 Git 状态（无未提交更改）
3. 运行测试套件

## 部署步骤
1. 构建 Docker 镜像
2. 推送到 Registry
3. 更新 K8s 部署
4. 验证健康检查

## 回滚计划
如果部署失败，执行 `kubectl rollout undo`
```

### 3.4 安全最佳实践

#### API 密钥管理

```bash
# ✅ 正确：使用 .env 文件
echo "OPENROUTER_API_KEY=sk-or-..." >> ~/.hermes/.env
chmod 600 ~/.hermes/.env

# ❌ 错误：在配置文件中硬编码
# config.yaml 中不应包含密钥
```

#### 命令执行安全

```
# 1. 始终启用命令审批（默认已启用）
# 2. 生产环境使用 Docker 后端隔离
# 3. 限制 SSH 后端的 sudoers 权限
# 4. 网关设备配对防止未授权访问
```

#### 数据安全

```
# 1. 定期备份 sessions.db
cp ~/.hermes/sessions.db ~/.hermes/sessions.db.bak

# 2. 记忆中不存储敏感信息
# 3. 日志中脱敏处理（agent/redact.py）
# 4. 技能文件不包含凭证
```

---

## 4. 性能最佳实践

### 4.1 上下文窗口管理

```
# 长对话策略
1. 使用子智能体委派独立子任务（零上下文成本）
2. 复杂任务拆分为多个会话
3. 利用上下文文件（AGENTS.md）提供持久上下文
4. 信任自动上下文压缩机制
```

### 4.2 工具集精简

```yaml
# config.yaml — 按需启用工具集
agent:
  enabled_toolsets:
    - terminal
    - file
    # 仅在需要时启用以下工具集
    # - browser
    # - web
    # - delegate
```

**原则**：启用的工具越少，模型的工具选择准确率越高，响应速度越快。

### 4.3 批量处理优化

```yaml
# datagen-config-examples/optimized_batch.yaml
workers: 8                    # 根据 API 限流调整
max_concurrent_requests: 16   # 并发 API 请求上限
per_trajectory_timeout: 300   # 单轨迹超时（秒）
checkpoint_interval: 10       # 每 10 条保存检查点
```

### 4.4 终端后端性能对比

| 后端 | 启动时间 | 执行速度 | 持久化 | 推荐场景 |
|------|---------|---------|--------|---------|
| local | 即时 | 最快 | 永久 | 开发调试 |
| docker | 1-5s | 快 | 容器内 | CI/CD |
| ssh | 1-2s | 取决于网络 | 永久 | 远程服务器 |
| modal | 10-30s | 取决于实例 | 临时 | GPU 任务 |
| daytona | 5-15s | 中等 | 休眠恢复 | Serverless |

---

## 5. 故障排查最佳实践

### 5.1 诊断流程

```bash
# Step 1: 自动诊断
hermes doctor

# Step 2: 检查配置
hermes config show

# Step 3: 检查日志
# 查看最近的会话
hermes sessions

# Step 4: 验证 API 连接
hermes model  # 测试模型可用性
```

### 5.2 常见问题速查

| 症状 | 可能原因 | 解决方案 |
|------|---------|---------|
| "API key invalid" | 密钥过期或格式错误 | 检查 `~/.hermes/.env` |
| 模型响应为空 | 上下文溢出 | 减少 `max_turns` 或启用压缩 |
| 工具调用失败 | 缺少依赖或 API 密钥 | 运行 `hermes tools` 检查 |
| 网关连接断开 | Bot Token 无效或网络问题 | 检查 Token 和网络 |
| 内存占用过高 | 长会话未压缩 | 重启会话或减少上下文 |
| Spinner 显示乱码 | 终端不支持 Unicode | 切换到 `mono` 皮肤 |

### 5.3 日志与调试

```bash
# 启用调试模式
export HERMES_DEBUG=1

# 保存轨迹（用于问题复现）
# config.yaml
agent:
  save_trajectories: true
  trajectory_dir: ~/.hermes/trajectories/
```

---

## 6. 架构扩展最佳实践

### 6.1 添加新消息平台

```python
# gateway/platforms/new_platform.py

class NewPlatformAdapter:
    """遵循现有适配器模式"""

    async def start(self):
        """初始化连接"""

    async def on_message(self, message):
        """处理收到的消息"""
        # 1. 解析平台特定格式
        # 2. 转换为统一消息格式
        # 3. 调用网关消息分发

    async def send_message(self, chat_id, content):
        """发送消息到平台"""
        # 1. 将统一格式转换为平台特定格式
        # 2. 处理媒体附件
        # 3. 发送并处理错误
```

### 6.2 添加新终端后端

```python
# tools/environments/new_backend.py

class NewBackendTerminal:
    """遵循终端后端接口"""

    def execute(self, command: str, timeout: int = None) -> str:
        """执行命令并返回输出"""

    def is_available(self) -> bool:
        """检查后端可用性"""

    def cleanup(self):
        """清理资源"""
```

### 6.3 MCP 服务器集成

```json
// ~/.hermes/mcp.json
{
  "servers": {
    "my-server": {
      "command": "npx",
      "args": ["my-mcp-server"],
      "env": {
        "API_KEY": "${MY_API_KEY}"
      }
    }
  }
}
```

---

## 7. 团队协作最佳实践

### 7.1 项目上下文文件

```markdown
<!-- AGENTS.md — 每个项目根目录 -->
# 项目名称 - 开发指南

## 技术栈
- Python 3.11 + FastAPI
- PostgreSQL 15
- React 19 + TypeScript

## 编码规范
- 使用 Black 格式化
- 类型注解必须
- 测试覆盖率 > 80%

## 部署流程
- staging: git push → CI → 自动部署
- production: PR 合并 → 手动触发
```

### 7.2 共享技能管理

```bash
# 团队共享技能仓库
git clone https://github.com/team/hermes-skills.git ~/.hermes/skills/team/

# 定期同步
hermes skills sync
```

### 7.3 多用户网关配置

```yaml
# config.yaml
gateway:
  platforms:
    telegram:
      allowed_users: [user_id_1, user_id_2]
      require_pairing: true
    slack:
      allowed_channels: ["#dev-ops", "#engineering"]
      require_mention: true
```

---

## 8. 总结：黄金法则

1. **最小工具集原则**：只启用需要的工具，提高准确率和速度
2. **缓存保护原则**：绝不在对话中途修改缓存内容
3. **隔离执行原则**：生产环境使用 Docker/Daytona 后端
4. **渐进式记忆原则**：记忆有价值的长期信息，不存储临时细节
5. **分步指示原则**：复杂任务拆分为明确的步骤
6. **安全第一原则**：密钥存 .env，启用命令审批，限制权限
7. **测试覆盖原则**：新功能必须包含测试，使用隔离 fixture
8. **成本意识原则**：合理选择模型，利用缓存，使用子智能体
