# mysql_assistant_re_act

`mysql_assistant_re_act` 是基于 LangChain `create_agent` 的 MySQL ReAct agent，模型来自根目录 `lib/langchain_model.py` 中的 `chat_anthropic`。

该实现基于 LangGraph runtime，让模型在循环中自主决定何时调用工具、何时继续推理、何时输出最终答案。

## 功能特点

- ReAct 方式多轮工具调用
- 会话模式保留完整 `messages` 历史
- 工具异常自动包装为 `ToolMessage`，便于模型自我修正后继续执行
- 默认只读，降低误操作风险

## 快速开始

1. 在项目根目录安装依赖：

```bash
pip install -r requirements.txt
```

2. 配置环境变量：

```bash
cp .env.example .env
cp agents/mysql_assistant_re_act/.env.example agents/mysql_assistant_re_act/.env
```

3. 启动：

```bash
python agents/mysql_assistant_re_act/main.py
```

单次提问：

```bash
python agents/mysql_assistant_re_act/main.py "当前实例有哪些数据库"
```

交互命令：

- `exit` / `quit` / `q`：退出
- `clear` / `reset`：清空会话消息

## 核心文件

- `main.py`：初始化 `create_agent`、系统提示、CLI 会话循环
- `tools.py`：工具注册
- `mysql_ops.py`：MySQL 连接与 SQL 策略控制

## 系统提示中的关键约束

- 优先探索数据库结构，再执行 SQL
- 禁止 `USE 数据库名`
- 每次只允许执行一条 SQL
- 只读模式下仅允许 `SELECT` / `WITH`
- 最终答复使用中文

## 环境变量

### 模型配置（项目根 `.env`）

由 `lib/langchain_model.py` 在导入时读取：

- `ANTHROPIC_API_KEY`
- `ANTHROPIC_BASE_URL`（可选）
- `ANTHROPIC_API_MODEL`

说明：

- 当前 agent 使用 `ChatAnthropic`
- 配置中启用了 `thinking`，应选择可兼容该能力的模型/网关
- `agents/mysql_assistant_re_act/.env` 不会覆盖模型变量

### Agent 配置（`agents/mysql_assistant_re_act/.env`）

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

- `INCLUDE_TABLES` 支持 `table_name` 与 `db_name.table_name`
- `PRINT_MODEL_OUTPUT=true` 会打印推理、工具调用和工具结果
- `MYSQL_DATABASE` 可不填，agent 可自行探索 `information_schema`

### 变量优先级

`load_env_config(project_root, agent_dir)` 的覆盖顺序：

1. 进程环境变量
2. `agents/mysql_assistant_re_act/.env`
3. 项目根 `.env`

## 可用工具

- `list_databases`
- `list_tables(database_name)`
- `get_table_schema(database_name, table_name)`
- `run_sql(sql)`

## SQL 策略

- 仅允许单条 SQL
- 只读模式下拒绝写操作和危险关键字
- 若需写操作，必须显式设置 `ALLOW_WRITE=true`

## 示例问题

- `当前实例有哪些数据库`
- `analytics 库里有哪些表`
- `再看 users 表结构`
- `统计这个表最近 7 天新增数据`
