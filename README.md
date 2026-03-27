# Langchain Playground

这是一个基于 Python 的多 Agent 运行 playground。

根目录只负责两件事：

- 扫描 `agents/` 下可运行的 agent
- 展示列表并启动你选择的 agent

每个 agent 的具体能力、依赖、环境变量和使用方式，都写在各自目录下的 `README.md` 中。

## 当前包含的 Agent

- `mysql_assistant`：基于 `ChatOpenAI`（`lib/langchain_model.py` 预配置）的工具调用式 MySQL 助手
- `mysql_assistant_re_act`：基于 LangChain `create_agent` + `ChatAnthropic`（`lib/langchain_model.py` 预配置）的 ReAct 风格 MySQL 助手

## 目录结构

```text
.
├── main.py
├── lib/
│   ├── env_loader.py      # 合并根与 agent 的 .env（进程环境优先）
│   └── langchain_model.py # 预置 ChatOpenAI / ChatAnthropic（仅从根 .env + 进程环境读模型变量）
├── agents/
│   ├── mysql_assistant/
│   │   ├── README.md
│   │   ├── .env.example
│   │   ├── main.py
│   │   ├── chat_cli.py
│   │   ├── mysql_assistant.py
│   │   ├── mysql_ops.py
│   │   └── tools.py
│   └── mysql_assistant_re_act/
│       ├── README.md
│       ├── .env.example
│       ├── main.py
│       ├── mysql_ops.py
│       └── tools.py
├── .env.example
└── requirements.txt
```

## 运行约定

- 根目录 `main.py` 是统一入口
- `agents/` 下每个直接子目录都可以视为一个候选 agent
- 只有包含 `main.py` 的 agent 目录才会被自动发现
- 运行器会列出全部 agent，并通过子进程启动对应入口文件

## 安装依赖

```bash
pip install -r requirements.txt
```

## 环境变量

项目根目录 `.env` 用来放公共模型配置（供 `lib/langchain_model.py` 在导入时读取）；agent 自己的 `.env` 用来放该 agent 的专属配置（例如 MySQL、运行开关等，由各 agent 在运行时通过 `load_env_config(project_root, agent_dir)` 合并）。

参考示例：

- 根配置：`.env.example`
- Agent 配置：各 agent 目录下的 `.env.example`

当前根 `.env.example` 里包含两套模型变量（可按需填写其一或全部）：

- `OPENAI_API_KEY`、`OPENAI_BASE_URL`、`OPENAI_MODEL`
- `ANTHROPIC_API_KEY`、`ANTHROPIC_BASE_URL`、`ANTHROPIC_API_MODEL`

## 变量优先级

对通过 `load_env_config(project_root, agent_dir)` 合并的配置（例如各 agent 里的 MySQL、运行开关），同名变量覆盖顺序如下：

1. 进程环境变量
2. agent 目录下的 `.env`
3. 项目根目录 `.env`

也就是说，agent 的本地配置会覆盖根配置。

**说明**：`lib/langchain_model.py` 在导入时只调用 `load_env_config(project_root)`，模型 API 相关变量以**项目根 `.env` 与进程环境**为准，不受 agent 目录 `.env` 覆盖。

## 运行方式

启动统一运行器：

```bash
python main.py
```

运行后会看到类似下面的列表：

```text
已注册的 agents：
1. mysql_assistant
2. mysql_assistant_re_act
```

输入编号后，运行器会启动对应 agent。

如果你已经确定要运行哪个 agent，也可以直接执行该 agent 目录下的 `main.py`。

## 新增 Agent

新增 agent 时，遵循下面的约定：

1. 在 `agents/` 下创建新子目录，例如 `agents/demo_agent`
2. 在该目录中创建 `main.py`
3. 在该目录中创建 `README.md`
4. 如有专属配置，补充 `.env.example`

最小结构示例：

```text
agents/
└── demo_agent/
    ├── README.md
    └── main.py
```

## Agent 文档

- `agents/mysql_assistant/README.md`
- `agents/mysql_assistant_re_act/README.md`
