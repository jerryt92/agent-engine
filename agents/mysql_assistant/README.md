# mysql_assistant

`mysql_assistant` 是一个基于 `ChatOpenAI` 的 MySQL 工具调用式智能体。

它会在回答问题时按需调用 MySQL 工具，而不是一次性猜 SQL：

- 先探索数据库、表和字段
- 再根据上下文决定是否执行查询
- 在交互模式下保留会话历史，支持连续追问

## 核心实现

- `main.py`：启动入口
- `chat_cli.py`：读取配置、创建模型和 CLI 会话循环
- `mysql_assistant.py`：维护 tool use 循环、上下文历史和系统提示
- `tools.py`：注册 `list_databases`、`list_tables`、`get_table_schema`、`run_sql`
- `mysql_ops.py`：负责 MySQL 连接、元数据查询、SQL 安全校验和执行

## 入口

直接启动：

```bash
python agents/mysql_assistant/main.py
```

单次提问：

```bash
python agents/mysql_assistant/main.py "当前实例有哪些数据库"
```

也可以通过项目根运行器启动：

```bash
python main.py
```

交互模式下：

- 输入 `exit`、`quit` 或 `q` 退出
- 输入 `clear` 或 `reset` 清空当前会话历史

## 环境变量

这个 agent 使用“根 `.env` + agent `.env`”两层配置。

- 项目根目录 `.env`：模型配置
- `agents/mysql_assistant/.env`：数据库和运行策略配置

参考示例：

- 根配置：`.env.example`
- agent 配置：`agents/mysql_assistant/.env.example`

### 根 `.env`

常用模型变量：

- `OPENAI_API_KEY`
- `OPENAI_BASE_URL`（可选）
- `OPENAI_MODEL`（可选，默认 `gpt-4o-mini`）

### `agents/mysql_assistant/.env`

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

- 不配置 `MYSQL_DATABASE` 也可以运行，agent 会通过 `information_schema` 自行探索
- `INCLUDE_TABLES` 支持 `table_name` 和 `db_name.table_name` 两种写法
- `PRINT_MODEL_OUTPUT=true` 时，会打印模型输出、工具参数和工具结果，方便调试

## 变量优先级

同名变量的覆盖顺序如下：

1. 进程环境变量
2. `agents/mysql_assistant/.env`
3. 项目根目录 `.env`

## 可用工具

- `list_databases`：列出当前实例中的业务数据库
- `list_tables(database_name)`：列出指定数据库中的表
- `get_table_schema(database_name, table_name)`：查看表结构
- `run_sql(sql)`：执行单条 SQL

## 使用示例

示例提问：

- `当前实例有哪些数据库`
- `analytics 库里有哪些表`
- `再看 users 表结构`
- `统计 sales.orders 最近 30 天订单数`

## 行为约束

- 默认只允许单条 `SELECT` / `WITH` 查询
- 只读模式下会拒绝写操作和危险关键字
- 如果要放开写操作，需要显式设置 `ALLOW_WRITE=true`
- 为了减少歧义，执行 SQL 时应尽量显式使用 `db_name.table_name`
