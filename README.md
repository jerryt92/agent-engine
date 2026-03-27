# Agent Engine

这是一个基于 Python 的多 Agent 运行器，支持 CLI 与 Web 两种方式启动 agent。

项目根目录职责：

- 自动扫描 `agents/*` 下可运行的 agent
- 在 CLI 中展示列表并启动选中的 agent
- 在 Web 页面展示 agent 卡片，并通过浏览器终端透传对应 `main.py`

## 当前内置 Agent

- `mysql_assistant`：基于 `ChatOpenAI` 的 MySQL 工具调用式助手
- `mysql_assistant_re_act`：基于 `create_agent + ChatAnthropic` 的 ReAct 风格 MySQL 助手

每个 agent 的详细说明见对应目录的 `README.md`。

## 快速开始

1. 安装依赖

```bash
pip install -r requirements.txt
```

2. 配置环境变量

```bash
cp .env.example .env
```

3. 启动 CLI 运行器

```bash
python main.py
```

4. 或启动 Web 外壳

```bash
python main_web.py
```

默认地址：`http://127.0.0.1:8000`

## 目录结构

```text
.
├── main.py
├── main_web.py
├── web/
│   ├── index.html
│   ├── terminal.html
│   └── static/
├── lib/
│   ├── agent_registry.py
│   ├── agent_runtime.py
│   ├── env_loader.py
│   └── langchain_model.py
├── agents/
│   ├── mysql_assistant/
│   │   ├── info.json
│   │   ├── README.md
│   │   ├── .env.example
│   │   └── main.py
│   └── mysql_assistant_re_act/
│       ├── info.json
│       ├── README.md
│       ├── .env.example
│       └── main.py
├── .env.example
└── requirements.txt
```

## 环境变量与优先级

根目录 `.env` 主要放模型相关配置，agent 目录 `.env` 主要放该 agent 的 MySQL/运行开关配置。

### 根目录 `.env`（模型相关）

- OpenAI：`OPENAI_API_KEY`、`OPENAI_BASE_URL`、`OPENAI_MODEL`
- Anthropic：`ANTHROPIC_API_KEY`、`ANTHROPIC_BASE_URL`、`ANTHROPIC_API_MODEL`

### 优先级（agent 运行时合并）

对通过 `load_env_config(project_root, agent_dir)` 合并的变量，同名覆盖顺序为：

1. 进程环境变量
2. `agents/<agent>/.env`
3. 项目根目录 `.env`

### 特别说明（模型初始化）

`lib/langchain_model.py` 在导入时读取的是 `load_env_config(project_root)`，即模型变量只受「进程环境 + 根 `.env`」影响，不会被 agent 目录 `.env` 覆盖。

## 运行方式

### CLI 统一入口

```bash
python main.py
```

运行后会列出已发现的 agent，输入编号即可启动。

### 直接运行指定 Agent

```bash
python agents/mysql_assistant/main.py
# 或
python agents/mysql_assistant_re_act/main.py
```

### Web 模式

```bash
python main_web.py --host 127.0.0.1 --port 8000
```

Web 页面支持：

- 首页展示 agent 列表
- 在线编辑根目录 `.env`
- 进入 agent 终端页，与 `main.py` 实时交互
- 在线编辑该 agent 的 `.env`

## 新增 Agent 规范

新增 `agents/demo_agent` 时，最少包含：

```text
agents/
└── demo_agent/
    ├── info.json
    ├── README.md
    └── main.py
```

`info.json` 示例：

```json
{
  "agent_id": "demo_agent",
  "name": "Demo Agent",
  "description": "这里填写 agent 的简要说明。"
}
```

约定：

- `agent_id` 建议与目录名一致
- `name` 用于 CLI 和 Web 展示
- `description` 用于列表和卡片描述

## Agent 文档入口

- `agents/mysql_assistant/README.md`
- `agents/mysql_assistant_re_act/README.md`
