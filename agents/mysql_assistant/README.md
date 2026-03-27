# mysql_assistant

`mysql_assistant` 是一个基于 LangChain `ChatOpenAI` 的 MySQL 工具调用式智能体。对话模型实例来自项目根目录的 `lib/langchain_model.py`（`chat_open_ai`）。

它会在回答问题时按需调用 MySQL 工具，而不是一次性猜 SQL：

- 先探索数据库、表和字段
- 再根据上下文决定是否执行查询
- 在交互模式下保留会话历史，支持连续追问

## 核心实现

- `main.py`：启动入口，委托 `chat_cli.main`
- `chat_cli.py`：合并 agent 与根目录环境变量、装配 `MySQLOps` 与 `MySQLAssistant`、CLI 会话循环；LLM 使用 `lib.langchain_model.chat_open_ai`
- `mysql_assistant.py`：维护 tool use 循环、上下文历史和系统提示
- `tools.py`：注册 `list_databases`、`list_tables`、`get_table_schema`、`run_sql`
- `mysql_ops.py`：负责 MySQL 连接、元数据查询、SQL 安全校验和执行
- 根目录 `lib/langchain_model.py`：预置 `ChatOpenAI`（导入时从**项目根** `.env` 与进程环境读取 OpenAI 相关变量）

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

- **OpenAI 对话模型**：由 `lib/langchain_model` 在**导入时**从项目根 `.env` 与进程环境读取；`agents/mysql_assistant/.env` **不会**参与合并模型相关变量。
- **MySQL 与运行策略**：由 `chat_cli.py` 调用 `load_env_config(项目根, agent 目录)`，合并根目录与 agent 的 `.env`（进程环境优先）。

参考示例：

- 根配置：`.env.example`
- agent 配置：`agents/mysql_assistant/.env.example`

### 项目根 `.env`（OpenAI）

常用变量（与 `lib/langchain_model` 中 `ChatOpenAI` 一致）：

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

## 变量优先级（MySQL 与 agent 侧开关）

对 `load_env_config(项目根, agents/mysql_assistant)` 合并出的配置，同名变量覆盖顺序为：

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
- 依赖包含 `langchain`、`langchain-openai`、`pymysql`、`python-dotenv` 等（见项目根 `requirements.txt`）
