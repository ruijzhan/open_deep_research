# Open Deep Research 学习计划（面向具备 Python 与 LangChain/LangGraph 基础的初学者）

> 目标：通过系统阅读与动手实践，深入理解本项目的 LangChain/LangGraph 实现思路与工程化细节，能够本地运行、调参、扩展工具与评估效果。

---

## 基础准备与运行
- [x] 安装与环境准备（建议使用 uv）
  - [x] 创建虚拟环境并安装依赖
    ```bash
    uv venv
    source .venv/bin/activate   # Windows: .venv\Scripts\activate
    uv sync
    ```
  - [x] 复制并填写环境变量（查看 `/.env.example`）
    - [x] `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / `TAVILY_API_KEY` 等
    - [x] 若仅本地试跑，可先填写 OpenAI + Tavily 最小组合
    ```bash
    cp .env.example .env
    vi .env
    ```
    - [x] 配置使用第三方 OpenAI 兼容 API
      
- [x] 启动 LangGraph 本地开发服务（参见 `README.md` 与 `langgraph.json`）
  - [x] 使用 LangGraph CLI 启动
    ```bash
    uvx --refresh --from "langgraph-cli[inmem]" --with-editable . --python 3.11 langgraph dev --allow-blocking
    ```
  - [x] 打开 Studio UI 并确认能与图交互
    - API: http://127.0.0.1:2024
    - Studio: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024

---

## 快速熟悉代码结构（鸟瞰）
- [ ] 阅读 `README.md`，明确项目定位、依赖与评估方式
- [ ] 浏览目录与关键文件：
  - [ ] `src/open_deep_research/deep_researcher.py`（核心 LangGraph 图与节点/子图定义）
  - [ ] `src/open_deep_research/configuration.py`（可配置参数，含搜索/模型/MCP 配置）
  - [ ] `src/open_deep_research/state.py`（`MessagesState`、工具结构化输出、状态定义与 reducer）
  - [ ] `src/open_deep_research/prompts.py`（澄清、研究、压缩、终稿等系统提示词模板）
  - [ ] `src/open_deep_research/utils.py`（工具加载、Tavily 搜索、MCP、token 限制处理等）
  - [ ] `src/security/auth.py`（与 LangGraph Studio/OAP 的鉴权集成）
  - [ ] `tests/` & `examples/`（评估脚本与成品范例）

---

## 由浅入深阅读代码（建议顺序）
- [ ] 配置系统 `configuration.py`
  - [ ] 了解 `Configuration` 字段含义与默认值（模型、搜索 API、并发、迭代上限等）
  - [ ] 理解 `from_runnable_config()` 如何将 `RunnableConfig`/环境变量注入配置
  - [ ] 掌握 `SearchAPI` 与 `MCPConfig` 的用途
- [ ] 状态与结构化输出 `state.py`
  - [ ] 阅读 `ConductResearch`、`ResearchComplete`、`ClarifyWithUser`、`ResearchQuestion` 等结构体
  - [ ] 理解 `AgentState`/`SupervisorState`/`ResearcherState` 的字段及 reducer（如 `override_reducer`）
  - [ ] 明确消息流采用 `MessagesState` & `MessageLikeRepresentation`
- [ ] 提示词模板 `prompts.py`
  - [ ] `clarify_with_user_instructions`、`lead_researcher_prompt`、`research_system_prompt`
  - [ ] `compress_research_system_prompt` 与 `final_report_generation_prompt` 的产物格式与引用规范
- [ ] 工具与通用逻辑 `utils.py`
  - [ ] 搜索工具：`tavily_search()` → `tavily_search_async()` → `summarize_webpage()` 的并发/超时/摘要流程
  - [ ] 反思工具：`think_tool()`（研究过程中的中间反思）
  - [ ] 工具集合：`get_all_tools()` 如何基于 `Configuration.search_api` 聚合 Tavily/OpenAI/Anthropic/MCP 工具
  - [ ] MCP：`load_mcp_tools()`、鉴权封装 `wrap_mcp_authenticate_tool()`、token 持久化 `get_tokens()/set_tokens()`
  - [ ] Token 限制处理：`is_token_limit_exceeded()` 与 `MODEL_TOKEN_LIMITS`/`get_model_token_limit()`
- [ ] 主图与子图 `deep_researcher.py`
  - [ ] 主流程节点：
    - [ ] `clarify_with_user()` → 判定是否需要二次澄清
    - [ ] `write_research_brief()` → 生成 `ResearchQuestion` 并初始化 `supervisor_messages`
    - [ ] `research_supervisor`（子图）：`supervisor()` + `supervisor_tools()`
      - [ ] 工具调用：`ConductResearch`、`ResearchComplete`、`think_tool`
      - [ ] 并发额度 `max_concurrent_research_units` 与 `research_iterations` 上限
    - [ ] `final_report_generation()` → 汇总所有 `notes` 生成终稿
  - [ ] 研究者子图：
    - [ ] `researcher()` → 绑定工具（搜索/MCP/think）并产出 `tool_calls`
    - [ ] `researcher_tools()` → 执行并行工具调用，控制 `max_react_tool_calls` 与 `ResearchComplete`
    - [ ] `compress_research()` → 对工具产出进行“清洗但不总结”的压缩
  - [ ] 图构建：`StateGraph` 的 `add_node()`/`add_edge()`/`compile()` 用法与输入输出 `Schema`

---

## 上手运行与观察（Studio 交互）
- [ ] 在 Studio 的 `Manage Assistants` 中选择 `Deep Researcher`（来自 `langgraph.json`）
- [ ] 在 `messages` 输入框发起一个中文研究请求，观察：
  - [ ] 是否进入澄清步骤（`clarify_with_user()`）
  - [ ] `research_supervisor` 的工具调用轨迹（`think_tool`→`ConductResearch`）
  - [ ] `researcher_subgraph` 的搜索调用与 `compress_research()` 的产物格式
  - [ ] 终稿 `final_report_generation()` 的结构与引用规范（Markdown + Sources）
- [ ] 调整配置并复跑（通过 Studio UI configurable 字段）：
  - [ ] 切换 `search_api`：`tavily` → `openai`/`anthropic`（无 Tavily 时的行为与限制）
  - [ ] 调整 `max_concurrent_research_units`、`max_researcher_iterations` 的影响
  - [ ] 提高/降低 `*_model_max_tokens` 观察 token 限制重试逻辑是否触发

---

## 阅读与运行评估脚本（理解如何打分与比较）
- [ ] 快速阅读 `tests/`：
  - [ ] `tests/evaluators.py` 与 `tests/prompts.py`（六大维度：总体质量、相关性、结构、正确性、扎实度、完整性）
  - [ ] `tests/run_evaluate.py`（使用 `LangSmith` 的批量评估，如何注入 `Configuration`）
  - [ ] `tests/supervisor_parallel_evaluation.py`（检查监督者是否使用了期望的并行度）
  - [ ] `tests/extract_langsmith_data.py`（如何从 LangSmith 导出 `jsonl` 供 Deep Research Bench 提交）
- [ ] 预备运行（注意成本与 API Key）：
  - [ ] 在 `.env` 配置 `LANGSMITH_API_KEY` 并按需修改 `dataset_name`/模型
  - [ ] 先小规模跑少量样本确认链路通畅，再跑全量

---

## 动手改造小练习（循序渐进）
- [ ] 练习 1：自定义搜索工具
  - [ ] 在 `utils.py` 中增加或切换搜索后端（如 DuckDuckGo/Exa 已在依赖内）
  - [ ] 在 `get_search_tool()`/`get_all_tools()` 挂接并通过 Studio 验证
- [ ] 练习 2：修改提示词以强化“反思+停手”
  - [ ] 在 `prompts.py` 中微调 `research_system_prompt` 与 `lead_researcher_prompt` 的“何时停止继续搜索”规则
  - [ ] 对比终稿长度/引用质量变化
- [ ] 练习 3：压缩策略微调
  - [ ] 在 `compress_research()` 增加更严格的“保留来源与段落”要求，观察是否更好地支撑终稿
- [ ] 练习 4：加入 MCP 工具（可选）
  - [ ] 配置 `Configuration.mcp_config`（带/不带鉴权）
  - [ ] 确认 `wrap_mcp_authenticate_tool()` 的报错友好性与 `think_tool` 配合
- [ ] 练习 5：限制与健壮性
  - [ ] 人为构造长上下文触发 `is_token_limit_exceeded()`，验证回退策略（删至最近 AI）与终稿截断重试

---

## 深入理解与对照学习（可选）
- [ ] 阅读 `src/legacy/legacy.md` 概览旧实现思路
  - [ ] 对比 `legacy/graph.py`（工作流/计划-执行）与 `legacy/multi_agent.py`（多研究者并行）
  - [ ] 体会新实现（`deep_researcher.py`）的简化/性能/可维护性取舍

---

## 阶段目标（建议 3–5 天节奏）
- [ ] 第 1 天：搭建环境、读配置与状态、在 Studio 成功跑通一次
- [ ] 第 2 天：精读 `deep_researcher.py` 主图/子图；画一张状态与边的流程图
- [ ] 第 3 天：完成“练习 1/2”，对比改动前后输出
- [ ] 第 4 天：阅读评估脚本，选择 5–10 条样本做小批量评估
- [ ] 第 5 天：尝试 MCP 或 token 极限/错误路径处理，形成自己的“最佳实践”笔记

---

## 检查清单（交付物）
- [ ] 一份你自己的流程图（节点：`clarify_with_user` → `write_research_brief` → `research_supervisor` → `final_report_generation`；子图 `researcher` 的循环与工具调用）
- [ ] 至少两处小改动的 PR 草稿（搜索/提示词/压缩策略任一）
- [ ] 一次最小评估跑通（含实验链接或本地记录）
- [ ] 一份“如何在中文问题下保持中文终稿”的经验总结（关注 `final_report_generation_prompt` 的语言一致性要求）

---

## 参考定位（文件与函数）
- [ ] 主入口与图：`src/open_deep_research/deep_researcher.py: deep_researcher_builder / deep_researcher`
- [ ] 配置：`src/open_deep_research/configuration.py: Configuration`
- [ ] 状态：`src/open_deep_research/state.py: AgentState / SupervisorState / ResearcherState`
- [ ] 提示词：`src/open_deep_research/prompts.py`
- [ ] 工具：`src/open_deep_research/utils.py: tavily_search / think_tool / get_all_tools / load_mcp_tools`
- [ ] 鉴权：`src/security/auth.py: auth`
- [ ] 部署/绑定：`langgraph.json`
- [ ] 评估：`tests/run_evaluate.py`、`tests/evaluators.py`、`tests/supervisor_parallel_evaluation.py`
