# mysql_assistant

`mysql_assistant` 是基于 LangChain `ChatOpenAI` 的 MySQL 工具调用式 agent。

它不会直接猜 SQL，而是优先通过工具探索数据库信息后再查询，适合做结构化问答和排查。

## 功能特点

- 先查库/表/字段，再决定是否执行 SQL
- 交互模式保存上下文，支持连续追问
- 默认只读，避免误执行写操作
- 支持 `INCLUDE_TABLES` 做表级访问范围限制

## 快速开始

1. 在项目根目录安装依赖：

```bash
pip install -r requirements.txt
```

2. 配置环境变量：

```bash
cp .env.example .env
cp agents/mysql_assistant/.env.example agents/mysql_assistant/.env
```

3. 启动：

```bash
python agents/mysql_assistant/main.py
```

单次提问：

```bash
python agents/mysql_assistant/main.py "当前实例有哪些数据库"
```

也可通过统一入口运行：

```bash
python main.py
```

交互命令：

- `exit` / `quit` / `q`：退出
- `clear` / `reset`：清空会话历史

## 核心文件

- `main.py`：入口，转发到 `chat_cli.main`
- `chat_cli.py`：加载配置、初始化 `MySQLOps` 与 `MySQLAssistant`、执行 CLI 循环
- `mysql_assistant.py`：工具调用循环与会话历史管理
- `tools.py`：工具注册
- `mysql_ops.py`：MySQL 连接、元数据查询、SQL 安全策略

## 环境变量

### 模型配置（项目根 `.env`）

由 `lib/langchain_model.py` 在导入时读取：

- `OPENAI_API_KEY`
- `OPENAI_BASE_URL`（可选）
- `OPENAI_MODEL`（可选）

`agents/mysql_assistant/.env` 不会覆盖模型变量。

### Agent 配置（`agents/mysql_assistant/.env`）

- `MYSQL_HOST`（默认 `127.0.0.1`）
- `MYSQL_PORT`（默认 `3306`）
- `MYSQL_USER`（默认 `root`）
- `MYSQL_PASSWORD`（默认空）
- `MYSQL_DATABASE`（可选）
- `MYSQL_CHARSET`（默认 `utf8mb4`）
- `MYSQL_CONNECT_TIMEOUT`（默认 `30`）
- `MYSQL_READ_TIMEOUT`（默认 `30`）
- `MYSQL_WRITE_TIMEOUT`（默认 `30`）
- `ALLOW_WRITE`（默认 `false`）
- `INCLUDE_TABLES`（可选，逗号分隔）
- `PRINT_MODEL_OUTPUT`（默认 `false`）

说明：

- `MYSQL_DATABASE` 可不填，agent 会从 `information_schema` 探索
- `INCLUDE_TABLES` 支持 `table_name` 或 `db_name.table_name`
- `.env.example` 里可能示例为 `PRINT_MODEL_OUTPUT=true`，仅用于调试演示，代码默认值仍为 `false`

### 变量优先级

`load_env_config(project_root, agent_dir)` 的覆盖顺序：

1. 进程环境变量
2. `agents/mysql_assistant/.env`
3. 项目根 `.env`

## 可用工具

- `list_databases`
- `list_tables(database_name)`
- `get_table_schema(database_name, table_name)`
- `run_sql(sql)`

## SQL 策略

- 仅允许单条 SQL
- 只读模式下只允许 `SELECT` / `WITH`
- 检测到危险关键字会拒绝执行
- 若需写操作，必须显式设置 `ALLOW_WRITE=true`

## 示例问题

- `当前实例有哪些数据库`
- `analytics 库里有哪些表`
- `再看 users 表结构`
- `统计 sales.orders 最近 30 天订单数`
