# 智能客服 Agent 项目学习指南

本文档结合《简历项目1：智能客服Agent.pdf》与当前代码库，说明如何系统学习本项目：从整体架构到各模块对应代码位置，再到可动手的实践顺序。

---

## 一、文档与代码的对应关系概览

| 文档中的概念 | 代码中的位置 | 说明 |
|-------------|-------------|------|
| **项目入口 / HTTP 接口** | `llm_backend/main.py` | `/api/langgraph/query` 接收用户 query，调用 `graph.astream` |
| **主图 / Router 意图识别** | `llm_backend/app/lg_agent/lg_builder.py` | `analyze_and_route_query` + `route_query`，五类意图 |
| **意图类型定义** | `llm_backend/app/lg_agent/lg_states.py` | `Router` 的 `type`: general / additional / graphrag / image / file |
| **Router 提示词** | `llm_backend/app/lg_agent/lg_prompts.py` | `ROUTER_SYSTEM_PROMPT` 等 |
| **安全护栏 (Guardrails)** | `lg_builder.py` 的 `get_additional_info` + `multi_tool` 里的 guardrails | 经营范围 + Neo4j Schema 动态注入 |
| **Planner 子任务分解** | `kg_sub_graph/.../planner/` + `multi_tool.py` | `create_planner_node`，Map-Reduce 前一步 |
| **并行 Tools (Text2Cypher / 预定义 Cypher / GraphRAG)** | `kg_sub_graph/.../workflows/multi_agent/multi_tool.py` | guardrails → planner → tool_selection → 各 tool → summarize → final_answer |
| **Text2Cypher 生成与校验** | `kg_sub_graph/.../components/text2cypher/` | generation / validation / correction / execution |
| **预定义 Cypher** | `kg_sub_graph/.../predefined_cypher/` + `cypher_dict` | 人工 Cypher 模板字典 |
| **GraphRAG 查询** | `kg_sub_graph/.../components/customer_tools` + `app/graphrag/` | 非结构化检索、社区/实体检索 |
| **幻觉检测** | `lg_builder.py` 的 `check_hallucinations` + `lg_prompts.py` 的 `CHECK_HALLUCINATIONS` | 当前主图中未在每条链路强制调用，可扩展 |

---

## 二、推荐学习路径（按顺序）

### 第 1 步：跑通请求链路（1 天）

1. **环境与依赖**
   - 阅读并执行：`requirements.txt`（FastAPI、LangGraph、langchain-deepseek、langchain_neo4j、graphrag 等）。
   - 配置 `.env`：DeepSeek API Key、Neo4j 连接、GraphRAG/向量服务等（若有）。

2. **入口与主图**
   - 打开 `llm_backend/main.py`：
     - 找到 `POST /api/langgraph/query`，看如何从 `query`/`conversation_id` 构造 `InputState` 和 `thread_config`。
     - 理解 `graph.astream(input_state, stream_mode="messages", config=thread_config)` 的用途（主图流式执行）。
   - 打开 `llm_backend/app/lg_agent/lg_builder.py`：
     - 从 `StateGraph(AgentState, input=InputState)` 和 `builder.add_node` / `add_edge` / `add_conditional_edges` 画出「主图」：  
       START → 意图识别 → 按类型分支 → 各处理节点 → END。
   - 对照文档：**Multi-Agent 主流程** = 意图识别 → 安全护栏（部分分支）→ Planner（仅 graphrag）→ 并行 tools → 汇总。

**目标**：能本地或通过前端发一条消息，看到日志里「意图类型」和最终回复，并能在代码里指出「这句话是在哪一步被分类、哪一步被处理的」。

---

### 第 2 步：意图识别（Router）（1–2 天）

1. **文档回顾**
   - 文档：五类意图（general / additional / graphrag / image / file）、Prompt-template + DeepSeek、结构化 JSON 输出、few-shot 与难负样本迭代（F1 0.82→0.93）。

2. **代码精读**
   - `lg_states.py`：`Router` 的 `type` 和 `logic` 字段。
   - `lg_builder.py`：
     - `analyze_and_route_query`：如何拼 `ROUTER_SYSTEM_PROMPT` + `state.messages`，调用 `model.with_structured_output(Router)`。
     - `route_query`：根据 `state.router["type"]` 和是否带图，返回下一节点名（如 `create_research_plan`、`get_additional_info` 等）。
   - `lg_prompts.py`：`ROUTER_SYSTEM_PROMPT` 的完整内容，对应文档里的「分类标准 + 五类描述」。

3. **与文档的差异与可做扩展**
   - 文档提到：image/file 用规则识别、logprobs 置信度、难负样本挖掘与 few-shot 迭代。当前代码里是「纯 LLM 结构化输出」，没有 logprobs、规则分支和难负样本 pipeline。  
   - 学习时可思考：若你要做「难负样本挖掘」，应在哪里接日志、哪里算置信度、哪里更新 few-shot 列表（例如在 `ROUTER_SYSTEM_PROMPT` 的示例部分）。

**目标**：能解释「用户这句话为什么被分到 graphrag-query」，并知道若要把 F1 做到 0.93 级别，需要在现有 Router 上增加哪些模块（置信度、规则、few-shot 管理）。

---

### 第 3 步：安全护栏（Guardrails）（1 天）

1. **文档回顾**
   - 经营范围 + Neo4j Schema 动态注入；输出 continue/end；误放比例压到一定比例以下。

2. **代码精读**
   - **主图侧**（additional 分支）：`lg_builder.py` 的 `get_additional_info`：
     - 用 `retrieve_and_parse_schema_from_graph_for_prompts(neo4j_graph)` 取 Schema。
     - 拼 `GUARDRAILS_SYSTEM_PROMPT` + 经营范围描述 + Schema，`AdditionalGuardrailsOutput(decision="end"|"continue")`，决定是否拒答。
   - **子图侧**（graphrag 分支）：`multi_tool.py` 的 `guardrails` 节点 + `guardrails_conditional_edge`：
     - 先过 guardrails 再进 planner；未通过则走默认回复或 final_answer。
   - 工具函数：`kg_sub_graph/.../utils/utils.py` 的 `retrieve_and_parse_schema_from_graph_for_prompts`（或同名模块），对应文档「get_schema() + 清理 + 注入占位符」。

**目标**：能说出「安全护栏在哪两个地方被调用」「Schema 从哪里来、如何塞进 prompt」，以及若要加「只加载局部 Schema」优化，应改哪里。

---

### 第 4 步：Planner 与并行 Tools（2–3 天）

1. **文档回顾**
   - 复杂问题拆成子任务；Map-Reduce 并行；结构化走 Text2Cypher/预定义 Cypher，非结构化走 GraphRAG。

2. **代码精读**
   - `multi_tool.py`：
     - 节点顺序：START → guardrails → planner → …（tool_selection / cypher_query / predefined_cypher / customer_tools）→ summarize → final_answer。
     - `edges.py`：`map_reduce_planner_to_tool_selection` 等，理解「每个子任务如何映射到工具选择、如何并行」。
   - Planner 实现：`kg_sub_graph/.../components/planner/node.py`（及同目录），看输入/输出状态（如 steps / question）和如何调用 LLM 做任务分解。
   - 工具选择：`tool_selection` 节点 + `tool_schemas`（cypher_query、predefined_cypher、microsoft_graphrag_query），对应文档「动态选择 Cypher vs GraphRAG」。

**目标**：能画出「从用户一句 graphrag-query 到 planner 输出多个 step，再到每个 step 选哪个 tool、最后 summarize」的流程图，并能在代码里指出 Map 与 Reduce 的边界。

---

### 第 5 步：Text2Cypher（2–3 天）

1. **文档回顾**
   - 生成（Few-shot + 模板 / neo4j-graphrag）+ 校验（EXPLAIN、关系方向、权限、Schema、LLM 辅助）+ 纠错重试；正确率 88% 等。

2. **代码精读**
   - 目录：`kg_sub_graph/.../components/text2cypher/`。
     - `generation/`：如何根据 question + schema + 示例生成 Cypher（prompts、node）。
     - `validation/`：语法（EXPLAIN）、关系方向、权限、Schema、LLM 校验（validators、prompts、node）。
     - `correction/`：校验失败后的修正逻辑（prompts、node）。
     - `execution/`：执行 Cypher 并返回结果。
   - Cypher 示例检索：`retrievers/cypher_examples/`（如 Northwind），对应文档「Cypher Dictionary + 相似示例检索」。
   - 预定义 Cypher：`predefined_cypher/` 和 `cypher_dict`，对应文档「80 条人工 + 20+ LLM 补充」。

**目标**：能说明「一条用户问题从进入 cypher_query 到最终执行，经过哪几层校验、失败时如何重试」，以及 88% 正确率的 bad case 在代码里对应哪些校验或生成环节。

---

### 第 6 步：GraphRAG（1–2 天）

1. **文档回顾**
   - Basic / Local / Global / Drift 检索；实体、社区、上下文图谱；离线索引 + 在线查询。

2. **代码精读**
   - 子图里的 GraphRAG 调用：`customer_tools`（`create_graphrag_query_node`），参数、输入输出格式。
   - 若项目内嵌了 GraphRAG 索引/查询实现：`app/graphrag/` 下与 index/query 相关的模块（如索引构建、检索入口），对应文档的「实体抽取、社区、embedding、四种 search」。

**目标**：能区分「当前代码里哪些查询走 Neo4j Cypher、哪些走 GraphRAG」，以及若你要加 Local/Global 策略选择，应在哪一层改（tool_selection 或 GraphRAG 封装）。

---

### 第 7 步：幻觉检测与后续优化（1 天）

1. **文档回顾**
   - 知识溯源、数值一致性、实体存在性、小模型裁判；漏检率等指标。

2. **代码精读**
   - `lg_builder.py` 的 `check_hallucinations`：输入 `state.documents` 与 `state.messages[-1]`，使用 `CHECK_HALLUCINATIONS` 与 `GradeHallucinations`。
   - 主图里是否在每条回复路径都调用了 `check_hallucinations`（当前可能未全链路接入），若未接入，思考应加在哪个节点之后（例如在 `create_research_plan` 返回前或 final_answer 前）。

**目标**：能说出「若要在当前项目里做到文档级别的幻觉拦截，需要在哪几个分支后插入幻觉检测节点、state 里需要传递哪些 documents」。

---

## 三、按文档章节快速索引代码

- **意图识别**  
  → `lg_builder.py`（`analyze_and_route_query`, `route_query`）、`lg_states.py`（Router）、`lg_prompts.py`（ROUTER_SYSTEM_PROMPT）。

- **安全护栏**  
  → `lg_builder.py`（`get_additional_info` 的 guardrails）、`multi_tool.py`（guardrails 节点）、`guardrails/node.py`、Schema 工具函数。

- **Planner**  
  → `multi_tool.py`、`planner/node.py`、`edges.py`（map_reduce 等）。

- **并行 tools**  
  → `multi_tool.py`（cypher_query、predefined_cypher、customer_tools）、`kg_tools_list.py`（tool schema 定义）。

- **Text2Cypher**  
  → `text2cypher/`（generation、validation、correction、execution）、`retrievers/cypher_examples/`、`predefined_cypher/`。

- **GraphRAG**  
  → `customer_tools`、`app/graphrag/`。

- **幻觉检测**  
  → `lg_builder.py`（`check_hallucinations`）、`lg_prompts.py`（CHECK_HALLUCINATIONS）、`lg_states.py`（GradeHallucinations）。

- **Neo4j Schema 注入**  
  → `retrieve_and_parse_schema_from_graph_for_prompts` 及 guardrails 的 prompt 拼接。

---

## 四、实践建议

1. **先主图后子图**：把主图的「意图 → 分支 → 节点」跑通并画出来，再进入 `create_research_plan` 内部的 multi_tool 子图。
2. **用日志跟一条请求**：从 `/api/langgraph/query` 跟到 `analyze_and_route_query` 的返回，再跟到 `create_research_plan` 里 planner → tool_selection → 某 tool → summarize，便于理解文档里「单条复杂询问 4.8s→2.1s」的并行化发生在哪里。
3. **对照文档做「差距列表」**：例如 logprobs、难负样本、Schema 缓存与局部加载、幻觉全链路等，在代码里哪些已实现、哪些可作你的扩展作业。
4. **指标与评估**：文档中的 F1、Cypher 正确率、幻觉率等，当前仓库未必有现成评估脚本；学习时可在本地用少量样本复现「意图评估」「Cypher 校验」的离线流程，便于面试时讲清楚。

按上述顺序把「文档—代码」一一对应走一遍，你就能既理解简历项目的设计，又能在代码层面讲清实现与可优化点。
