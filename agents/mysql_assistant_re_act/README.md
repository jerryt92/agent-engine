# mysql_assistant_re_act

`mysql_assistant_re_act` 是一个基于 LangChain `create_agent` 的 MySQL ReAct 智能体。

它直接使用 LangGraph 驱动的 agent runtime，让模型在循环中自主决定何时调用工具、何时继续推理、何时给出最终答案。

## 核心实现

- `main.py`：启动入口、创建 `create_agent`、维护 CLI 会话
- `tools.py`：注册 `list_databases`、`list_tables`、`get_table_schema`、`run_sql`
- `mysql_ops.py`：负责 MySQL 连接、元数据查询、SQL 安全校验和执行

当前 agent 的系统提示会约束模型：

- 先探索库、表、字段，再执行 SQL
- 禁止使用 `USE 数据库名`
- 一次只允许执行一条 SQL
- 只读模式下仅允许 `SELECT` / `WITH`
- 最终答复使用中文

另外，这个 agent 还注册了工具错误中间件：如果工具抛出异常，会把错误包装成 `ToolMessage` 回传给模型，方便模型修正参数后继续尝试。

## 入口

直接启动：

```bash
python agents/mysql_assistant_re_act/main.py
```

单次提问：

```bash
python agents/mysql_assistant_re_act/main.py "当前实例有哪些数据库"
```

交互模式下：

- 输入 `exit`、`quit` 或 `q` 退出
- 输入 `clear` 或 `reset` 清空当前会话消息

## 环境变量

这个 agent 使用“根 `.env` + agent `.env`”两层配置。

- 项目根目录 `.env`：Anthropic 模型配置
- `agents/mysql_assistant_re_act/.env`：数据库和运行策略配置

参考示例：

- 根配置：`.env.example`
- agent 配置：`agents/mysql_assistant_re_act/.env.example`

### 根 `.env`

常用模型变量：

- `ANTHROPIC_API_KEY`
- `ANTHROPIC_BASE_URL`（可选）
- `ANTHROPIC_API_MODEL`

说明：

- 当前实现使用 `ChatAnthropic`
- 代码里启用了 `thinking` 模式，因此应选择支持该能力的 Anthropic 兼容模型

### `agents/mysql_assistant_re_act/.env`

数据库相关变量：

- `MYSQL_HOST`
- `MYSQL_PORT`
- `MYSQL_USER`
- `MYSQL_PASSWORD`
- `MYSQL_DATABASE`（可选）
- `MYSQL_CHARSET`（可选，默认 `utf8mb4`）
- `MYSQL_CONNECT_TIMEOUT`（可选，默认 `30`）
- `MYSQL_READ_TIMEOUT`（可选，默认 `30`）
- `MYSQL_WRITE_TIMEOUT`（可选，默认 `30`）
- `ALLOW_WRITE`（可选，默认 `false`）
- `INCLUDE_TABLES`（可选）
- `PRINT_MODEL_OUTPUT`（可选，默认 `false`）

示例：

```bash
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD='123456'
# MYSQL_DATABASE=your_database_name
MYSQL_CHARSET=utf8mb4
MYSQL_CONNECT_TIMEOUT=30
MYSQL_READ_TIMEOUT=30
MYSQL_WRITE_TIMEOUT=30
ALLOW_WRITE=false
INCLUDE_TABLES=
PRINT_MODEL_OUTPUT=false
```

说明：

- `INCLUDE_TABLES` 支持 `table_name` 和 `db_name.table_name`
- `PRINT_MODEL_OUTPUT=true` 时，会打印推理片段、工具调用和工具结果
- 不配置 `MYSQL_DATABASE` 也可以运行，agent 会自行探索 `information_schema`

## 可用工具

- `list_databases`：列出当前实例中的业务数据库
- `list_tables(database_name)`：列出指定数据库中的表
- `get_table_schema(database_name, table_name)`：查看表结构
- `run_sql(sql)`：执行单条 SQL

## 使用示例

CLI 在交互模式下会维护完整 `messages` 历史，因此可以连续追问，例如：

- `当前实例有哪些数据库`
- `analytics 库里有哪些表`
- `再看 users 表结构`
- `统计这个表最近 7 天新增数据`

## 注意事项

- 默认只允许单条 `SELECT` / `WITH` 查询，避免误执行增删改
- 如果要放开写操作，需要显式设置 `ALLOW_WRITE=true`
- 为了减少歧义，SQL 中尽量显式使用 `db_name.table_name`
- 当前依赖包含 `langchain`、`langchain-anthropic`、`pymysql`、`python-dotenv`
