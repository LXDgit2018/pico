# `uv run pico "inspect the test failures and propose a fix"` 完整执行流程

## 第一层：命令启动

**`uv run pico`** 做的事情：

1. `uv` 读取 `pyproject.toml`，找到 `[project.scripts]` 中的 `pico = "pico.cli:main"`
2. 用 Python 执行 `pico/cli.py` 的 `main()` 函数
3. 将 `"inspect the test failures and propose a fix"` 作为位置参数 `prompt` 传入

## 第二层：CLI 参数解析 (`pico/cli.py`)

`main()` 调用 `build_arg_parser()` 解析命令行：

```
prompt  →  ["inspect", "the", "test", "failures", "and", "propose", "a", "fix"]
           → 拼接为 "inspect the test failures and propose a fix"
--provider  → 默认 "openai"
--model     → 默认值（取决于 provider）
--approval  → 默认 "ask"（每次工具调用需人工确认）
--max-steps → 默认 6
--cwd       → 当前目录
```

然后调用 `build_agent(args)` 组装 agent：

1. **`WorkspaceContext.build()`** — 运行 `git rev-parse --show-toplevel`、`git branch`、`git status --short`、`git log --oneline -5`，收集 README、AGENTS.md、pyproject.toml 等文件内容，构建工作区快照
2. **`load_project_env()`** — 加载 `.env` 文件中的环境变量（API key 等）
3. **`_build_model_client()`** — 根据 `--provider` 创建对应的模型客户端（`OpenAICompatibleModelClient` / `AnthropicCompatibleModelClient` / `OllamaModelClient`）
4. **`SessionStore`** — 初始化会话持久化存储（`.pico/sessions/`）
5. **`Pico(...)`** — 创建核心 runtime 实例

因为传了 prompt，所以走**一次性模式**（非 REPL），直接调用 `agent.ask(prompt)`。

## 第三层：Pico Agent 初始化 (`pico/runtime.py`)

`Pico.__init__()` 做下面几件事：

1. **`build_prefix()`** — 构建系统提示词（"工作手册"），包含：
   - Agent 身份定义和规则
   - 全部工具列表（`list_files`、`read_file`、`search`、`run_shell`、`write_file`、`patch_file`、`delegate`）及其 JSON 调用格式、使用示例
   - 工作区事实：git 分支、状态、最近 commit、README 摘要

2. **`build_tools()`** — 创建工具注册表，每个工具有风险等级（`read_only`/`write`）、参数校验函数、执行函数

3. **`LayeredMemory`** — 初始化双层记忆系统（情景记忆 + 持久记忆）

4. **`ContextManager`** — 初始化上下文管理器（负责 prompt 组装和预算裁剪，总预算 12000 字符）

## 第四层：主循环 (`Pico.ask()`)

这是核心执行引擎，最多迭代 `max_steps`（默认 6）次。每次迭代：

### 第 1 步：组装 prompt

`ContextManager.build()` 按以下顺序拼出一个约 12K 字符的 prompt：

```
{prefix}           ← 系统提示词（工具定义、工作区信息）
{memory}           ← 当前任务摘要、最近访问文件、情景笔记
{relevant_memory}  ← 与当前查询相关的持久记忆
{history}          ← 对话历史（含预算裁剪）
Current user request:
inspect the test failures and propose a fix
```

如果总长度超出 12000 字符，按优先级依次裁剪 `relevant_memory` → `history` → `memory` → `prefix`。

### 第 2 步：调用模型

`model_client.complete(prompt)` 通过纯 `urllib.request` 发 HTTP POST：

- **OpenAI 兼容**：POST `{base_url}/responses`，body 含 `model`、`input`、`max_output_tokens`
- **Anthropic 兼容**：POST `{base_url}/messages`，body 含 `model`、`messages[].role/content`、`max_tokens`
- 自动重试 HTTP 5xx 错误最多 3 次

### 第 3 步：解析模型输出

`Pico.parse()` 使用正则匹配：

- 匹配 `<tool>{"name":"...","args":{...}}</tool>` → 识别为工具调用
- 匹配 `<final>...</final>` → 识别为最终答案
- 都不匹配 → 将原始文本当作最终答案

### 第 4 步：执行工具（如果模型返回工具调用）

`run_tool(name, args)` 执行：

1. **参数验证** — 路径逃逸检查（不允许 `../`）、文件存在性校验、行范围校验、timeout 范围（1-120s）
2. **重复调用检测** — 如果连续两次调用完全相同的工具+参数，拒绝执行
3. **审批** — 根据 `--approval` 策略：
   - `ask`：打印工具调用详情，等待用户输入 `y/n/s`（yes/no/skip）
   - `auto`：自动批准
   - `never`：拒绝所有高风险工具
4. **执行** — 例如 `run_shell` 通过 `subprocess.run(shell=True)` 运行命令（环境变量被白名单限制）
5. **快照 diff** — 高风险工具执行前后对比工作区文件变化
6. **更新记忆** — 将访问的文件加入 `recent_files`，创建文件摘要，添加情景笔记
7. **创建检查点** — 保存当前状态快照

### 第 5 步：循环继续

工具执行结果追加到 `history`，进入下一轮迭代。模型看到工具输出后：

- 可能继续调用更多工具（读文件、搜索代码、再次运行测试）
- 或者输出 `<final>...</final>` 结束

## 第五层：针对这个具体 prompt，模型实际会做什么

对于 "inspect the test failures and propose a fix"，典型行为是：

| 迭代 | 工具调用 | 目的 |
|------|---------|------|
| 1 | `run_shell`: `uv run --with pytest python -m pytest -q` | 运行测试，获取失败列表 |
| 2 | `read_file`: 失败的测试文件 | 阅读测试代码 |
| 3 | `read_file`: 被测源文件 | 阅读实现代码 |
| 4 | `search`: 搜索相关函数/类 | 定位问题根因 |
| 5 | `<final>` | 总结失败原因并提出修复方案 |

## 第六层：结束和持久化

当模型返回 `<final>`（或达到 max_steps 上限）：

1. 记录 assistant 消息到 history
2. `TaskState.finish_success()` 标记任务完成
3. 调用 `promote_durable_memory()` — 如果对话包含"记住/保存/记录"等关键词，将关键信息持久化到 `.pico/memory/MEMORY.md`
4. 写入运行工件到 `.pico/runs/{run_id}/`：
   - `task_state.json` — 步骤数、尝试次数、停止原因
   - `trace.jsonl` — 每一步的详细事件日志
   - `report.json` — 最终答案摘要
5. 会话保存到 `.pico/sessions/{session_id}.json`

## 第七层：返回 CLI

`ask()` 返回最终答案字符串。`main()` 将其打印到终端，然后 `sys.exit(0)`。

## 数据流总览

```
命令行参数
  → cli.py:main() 解析
    → build_agent() 组装 agent
      → Pico.ask("inspect the test failures and propose a fix")
        → 循环：
          ContextManager.build() 拼 prompt
          → ModelClient.complete() 调大模型
            → Pico.parse() 解析响应
              → run_tool() 执行工具（如需审批则等待用户输入）
                → 更新 memory / 创建 checkpoint
        → 返回最终答案
  → print() 输出
  → 写入 .pico/runs/ 和 .pico/sessions/
```

## 关键文件索引

| 文件 | 关键符号 |
|------|----------|
| `pyproject.toml` | 入口点 `pico.cli:main` |
| `pico/__main__.py` | `__main__.main()` |
| `pico/cli.py` | `main()`、`build_agent()`、`build_arg_parser()`、`_build_model_client()` |
| `pico/runtime.py` | `class Pico`：`ask()`、`run_tool()`、`build_prefix()`、`parse()`、`approve()` |
| `pico/models.py` | `OllamaModelClient`、`OpenAICompatibleModelClient`、`AnthropicCompatibleModelClient` |
| `pico/tools.py` | `tool_run_shell()`、`tool_read_file()`、`tool_write_file()`、`tool_patch_file()`、`tool_search()`、`tool_delegate()` |
| `pico/context_manager.py` | `ContextManager.build()` — prompt 组装和预算控制 |
| `pico/memory.py` | `LayeredMemory`、`DurableMemoryStore` |
| `pico/workspace.py` | `WorkspaceContext.build()`、`fingerprint()`、`text()` |
| `pico/task_state.py` | `TaskState` — 任务进度追踪 |
| `pico/run_store.py` | `RunStore` — 运行工件持久化 |
| `pico/config.py` | `load_project_env()`、环境变量管理 |
