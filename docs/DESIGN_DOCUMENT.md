# DataAgent 项目设计文档

中文 | [English](./DESIGN_DOCUMENT-en.md)

> 本文档详细记录 DataAgent 项目的产品设计和技术实现细节。

---

## 目录

- [一、产品概述](#一产品概述)
- [二、核心产品功能](#二核心产品功能)
- [三、技术架构设计](#三技术架构设计)
- [四、StateGraph 工作流引擎](#四stategraph-工作流引擎)
- [五、前端架构设计](#五前端架构设计)
- [六、数据模型设计](#六数据模型设计)
- [七、关键技术实现](#七关键技术实现)
- [八、部署架构](#八部署架构)

---

## 一、产品概述

### 1.1 产品定位

**DataAgent** 是一个基于 **Spring AI Alibaba Graph** 构建的企业级智能数据分析 Agent。它超越了传统的 Text-to-SQL 工具，进化为一个能够执行 **Python 深度分析**、生成 **多维度图表报告** 的 AI 智能数据分析师。

### 1.2 核心价值

| 价值维度 | 描述 |
|:---|:---|
| **降低门槛** | 非技术人员也能通过自然语言获取数据洞察 |
| **提升效率** | 自动化数据分析流程，减少重复工作 |
| **增强能力** | 结合 Python 实现传统 BI 工具难以完成的高级分析 |
| **知识沉淀** | 通过 RAG 实现业务知识的积累和复用 |

### 1.3 技术特色

- **全面兼容 OpenAI 接口规范**：支持接入 Qwen、Deepseek 等主流大模型
- **灵活挂载向量数据库**：可插拔设计，适配任意向量存储
- **原生 MCP 协议支持**：可作为 MCP 服务器集成到 Claude Desktop 等工具
- **企业级特性**：完善的 API Key 管理、权限控制、多租户支持

---

## 二、核心产品功能

### 2.1 智能数据分析 (Text-to-SQL)

**功能描述**：将用户的自然语言查询转换为可执行的 SQL 语句。

**核心能力**：
- 自然语言理解与意图识别
- 多表复杂查询支持（JOIN、聚合、子查询）
- 多轮对话上下文理解
- SQL 语义一致性校验

**使用场景**：
```
用户: "查询过去30天销售额最高的10个产品"
系统: 自动生成 SQL 并执行，返回结果
```

### 2.2 Python 深度分析

**功能描述**：根据分析需求自动生成并执行 Python 代码，实现高级数据分析。

**核心能力**：
- 统计分析（均值、方差、相关性分析）
- 趋势预测（时间序列分析、回归预测）
- 机器学习模型（分类、聚类、异常检测）
- 数据可视化生成

**执行环境**：
| 类型 | 说明 | 适用场景 |
|:---|:---|:---|
| Docker Executor | 容器化隔离执行 | 生产环境（推荐） |
| Local Executor | 本地 Python 环境 | 开发测试 |
| AI Simulation | AI 模拟执行结果 | 演示测试 |

### 2.3 智能报告生成

**功能描述**：将分析结果自动汇总为包含可视化图表的报告。

**输出格式**：
- HTML 格式（包含交互式 ECharts 图表）
- Markdown 格式（适合文档集成）

**图表类型**：
- 柱状图 (BarChart)
- 折线图 (LineChart)
- 饼图 (PieChart)
- 更多自定义图表

### 2.4 人工反馈机制 (Human-in-the-loop)

**功能描述**：在分析计划生成阶段允许用户进行干预和调整。

**工作流程**：
```
计划生成 → 暂停等待用户反馈 → 同意/拒绝
                                  │
                        同意 ────→ 继续执行
                        拒绝 ────→ 重新规划
```

**技术实现**：
- 请求参数 `humanFeedback=true` 启用
- `CompiledGraph` 使用 `interruptBefore(HUMAN_FEEDBACK_NODE)` 实现暂停
- 通过 `threadId` 恢复执行

### 2.5 RAG 检索增强

**功能描述**：通过语义检索业务知识库，提升 SQL 生成的准确性。

**知识类型**：
| 类型 | 说明 |
|:---|:---|
| 业务知识 | 通用的业务术语、计算规则 |
| 智能体知识 | 特定智能体的领域知识 |
| 语义模型 | 表和字段的业务语义描述 |

**检索策略**：
- 向量检索（语义相似度）
- 混合检索（向量 + 关键词）
- 动态过滤（按智能体、知识类型）

### 2.6 多模型调度

**功能描述**：支持运行时动态切换不同的 LLM 和 Embedding 模型。

**架构设计**：
```
ModelConfigController
        │
        ▼
ModelConfigOpsService
        │
        ▼
AiModelRegistry ──→ DynamicModelFactory ──→ OpenAiApi
        │
        ▼
   Chat Model / Embedding Model
```

**支持特性**：
- 热切换模型无需重启
- 同一时间每类模型仅一个激活
- 全面兼容 OpenAI 接口规范

### 2.7 MCP 服务器

**功能描述**：遵循 Model Context Protocol 协议，对外提供工具能力。

**可用工具**：

| 工具名称 | 功能 |
|:---|:---|
| `nl2SqlToolCallback` | 自然语言转 SQL |
| `listAgentsToolCallback` | 查询智能体列表 |

**配置方式**：
```yaml
spring:
  ai:
    mcp:
      server:
        sse-endpoint: /sse  # 默认端点
```

### 2.8 API Key 管理

**功能描述**：完善的 API Key 生命周期管理，支持细粒度权限控制。

**管理能力**：
- 生成 API Key
- 重置 API Key
- 启用/禁用
- 删除

**调用方式**：
```bash
curl -X POST "http://localhost:8065/api/..." \
  -H "X-API-Key: <your_api_key>"
```

---

## 三、技术架构设计

### 3.1 总体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Clients 层                                 │
│   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐        │
│   │  Frontend UI   │  │  Admin Console │  │   MCP Client   │        │
│   │   (Vue 3)      │  │                │  │ (Claude等)     │        │
│   └────────────────┘  └────────────────┘  └────────────────┘        │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    ▼                           ▼
             ┌────────────┐              ┌────────────┐
             │  REST API  │              │ SSE Stream │
             └────────────┘              └────────────┘
                    │                           │
                    └─────────────┬─────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 data-agent-management (Spring Boot)                  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                      Controller Layer                          │  │
│  │  GraphController │ AgentController │ ModelConfigController    │  │
│  │  PromptConfigController │ DatasourceController │ ...          │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                       Service Layer                            │  │
│  │  GraphServiceImpl    │ LlmService       │ AgentVectorStoreService │
│  │  MultiTurnContextManager │ CodePoolExecutorService │ ...      │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                   StateGraph Workflow Engine                   │  │
│  │     ┌──────────────────────────────────────────────────┐      │  │
│  │     │  16个处理节点 + 12个调度器 (Dispatcher)            │      │  │
│  │     └──────────────────────────────────────────────────┘      │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                        Data Layer                              │  │
│  │            Mapper (MyBatis) │ Entity │ DTO/VO                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          ▼                       ▼                       ▼
   ┌────────────┐          ┌────────────┐          ┌────────────┐
   │ Business   │          │ Management │          │  Vector    │
   │    DB      │          │     DB     │          │   Store    │
   │ (业务数据) │          │  (元数据)   │          │ (向量库)   │
   └────────────┘          └────────────┘          └────────────┘
          │
          │
   ┌────────────┐
   │  Python    │
   │  Runtime   │
   │(Docker/本地)│
   └────────────┘
```

### 3.2 技术栈详情

| 层级 | 技术选型 | 版本 |
|:---|:---|:---|
| **后端框架** | Spring Boot | 3.4.8+ |
| **AI 框架** | Spring AI | 1.1.0 |
| **AI 扩展** | Spring AI Alibaba | 1.1.0.0 |
| **编程语言** | Java | 17+ |
| **前端框架** | Vue 3 + TypeScript | - |
| **构建工具** | Maven | - |
| **ORM 框架** | MyBatis | 3.0.4 |
| **连接池** | Druid | 1.2.22 |
| **数据库** | MySQL | 5.7+ |
| **容器化** | Docker | - |
| **图表库** | ECharts | - |

### 3.3 模块结构

```
DataAgent/
├── data-agent-management/          # 后端核心模块
│   └── src/main/java/com/alibaba/cloud/ai/dataagent/
│       ├── controller/             # REST API 控制器
│       ├── service/                # 业务服务层
│       │   ├── agent/              # 智能体服务
│       │   ├── graph/              # 图执行服务
│       │   ├── llm/                # LLM 服务
│       │   ├── mcp/                # MCP 服务
│       │   ├── semantic/           # 语义模型服务
│       │   └── vectorstore/        # 向量存储服务
│       ├── workflow/               # 工作流引擎
│       │   ├── node/               # 处理节点 (16个)
│       │   └── dispatcher/         # 调度器 (12个)
│       ├── mapper/                 # MyBatis Mapper
│       ├── entity/                 # 数据实体
│       ├── dto/                    # 数据传输对象
│       ├── vo/                     # 视图对象
│       ├── config/                 # 配置类
│       ├── strategy/               # 策略模式实现
│       ├── splitter/               # 文本分割器
│       └── util/                   # 工具类
│
├── data-agent-frontend/            # 前端模块
│   └── src/
│       ├── components/             # Vue 组件
│       ├── views/                  # 页面视图
│       ├── services/               # API 服务
│       ├── router/                 # 路由配置
│       └── styles/                 # 样式文件
│
├── docs/                           # 文档
├── docker-file/                    # Docker 配置
└── CI/                             # CI/CD 配置
```

---

## 四、StateGraph 工作流引擎

### 4.1 节点概览

系统采用 StateGraph 状态图驱动的工作流引擎，包含 **16 个核心处理节点**：

| 序号 | 节点名称 | 类型 | 功能描述 |
|:---:|:---|:---|:---|
| 1 | `IntentRecognitionNode` | 输入处理 | 识别用户意图，判断是否需要数据分析 |
| 2 | `EvidenceRecallNode` | 检索增强 | 从知识库召回相关业务知识 |
| 3 | `QueryEnhanceNode` | 检索增强 | 重写优化用户查询 |
| 4 | `SchemaRecallNode` | 检索增强 | 检索相关数据表和字段 Schema |
| 5 | `TableRelationNode` | 检索增强 | 分析表间关联关系 |
| 6 | `FeasibilityAssessmentNode` | 规划 | 评估分析请求的可行性 |
| 7 | `PlannerNode` | 规划 | 生成分析执行计划 |
| 8 | `PlanExecutorNode` | 规划 | 调度执行分析步骤 |
| 9 | `HumanFeedbackNode` | 反馈 | 等待用户确认或调整 |
| 10 | `SqlGenerateNode` | SQL执行 | 生成 SQL 语句 |
| 11 | `SemanticConsistencyNode` | SQL执行 | 验证 SQL 语义一致性 |
| 12 | `SqlExecuteNode` | SQL执行 | 执行 SQL 并获取结果 |
| 13 | `PythonGenerateNode` | Python执行 | 生成分析用 Python 代码 |
| 14 | `PythonExecuteNode` | Python执行 | 在隔离环境执行代码 |
| 15 | `PythonAnalyzeNode` | Python执行 | 处理 Python 执行结果 |
| 16 | `ReportGeneratorNode` | 输出 | 生成可视化分析报告 |

### 4.2 节点详细实现解析

#### 4.2.1 IntentRecognitionNode（意图识别节点）

**文件位置**: `workflow/node/IntentRecognitionNode.java`

**核心职责**: 识别用户输入是闲聊还是数据分析请求

**实现逻辑**:
```java
// 1. 获取用户输入和多轮对话上下文
String userInput = StateUtil.getStringValue(state, INPUT_KEY);
String multiTurn = StateUtil.getStringValue(state, MULTI_TURN_CONTEXT, "(无)");

// 2. 构建意图识别提示词
String prompt = PromptHelper.buildIntentRecognitionPrompt(multiTurn, userInput);

// 3. 调用 LLM 进行意图识别
Flux<ChatResponse> responseFlux = llmService.callUser(prompt);

// 4. 解析 LLM 返回的 JSON 结果
IntentRecognitionOutputDTO intentRecognitionOutput = 
    jsonParseUtil.tryConvertToObject(result, IntentRecognitionOutputDTO.class);
```

**输出结构** (`IntentRecognitionOutputDTO`):
- `needAnalysis`: boolean - 是否需要数据分析
- `responseToUser`: String - 如果是闲聊，直接返回的回复内容

---

#### 4.2.2 EvidenceRecallNode（证据召回节点）

**文件位置**: `workflow/node/EvidenceRecallNode.java`

**核心职责**: 从向量知识库召回与用户问题相关的业务知识

**实现逻辑**:
```java
// 1. 查询重写 - 让 LLM 生成独立的检索查询
String prompt = PromptHelper.buildEvidenceQueryRewritePrompt(multiTurn, question);
// 注：不扩展为多个子查询，避免 LLM 无法理解个性化业务术语时引入噪音

// 2. 从向量库检索两类知识文档
// - 业务知识 (business_term)
List<Document> businessTermDocuments = vectorStoreService
    .getDocumentsForAgent(agentId, standaloneQuery, BUSINESS_TERM);
// - 智能体知识 (agent_knowledge) 
List<Document> agentKnowledgeDocuments = vectorStoreService
    .getDocumentsForAgent(agentId, standaloneQuery, AGENT_KNOWLEDGE);

// 3. 构建格式化的证据内容
// 格式: 1. [来源: xxx] Q: xxx A: xxx
String evidence = buildFormattedEvidenceContent(...);
```

**知识类型处理**:
| 知识类型 | 格式 |
|:---|:---|
| FAQ/QA | `[来源: 标题] Q: 问题 A: 答案` |
| DOCUMENT | `[来源: 标题-文件名] 内容片段` |

---

#### 4.2.3 QueryEnhanceNode（查询增强节点）

**文件位置**: `workflow/node/QueryEnhanceNode.java`

**核心职责**: 根据 evidence 信息将业务术语翻译为数据库可理解的查询

**实现逻辑**:
```java
// 1. 获取用户输入、证据和多轮上下文
String userInput = StateUtil.getStringValue(state, INPUT_KEY);
String evidence = StateUtil.getStringValue(state, EVIDENCE);
String multiTurn = StateUtil.getStringValue(state, MULTI_TURN_CONTEXT);

// 2. 构建查询增强提示词
String prompt = PromptHelper.buildQueryEnhancePrompt(multiTurn, userInput, evidence);

// 3. 调用 LLM 进行查询增强
Flux<ChatResponse> responseFlux = llmService.callUser(prompt);

// 4. 解析输出
QueryEnhanceOutputDTO queryEnhanceOutputDTO = 
    jsonParseUtil.tryConvertToObject(enhanceResult, QueryEnhanceOutputDTO.class);
```

**输出结构** (`QueryEnhanceOutputDTO`):
- `canonicalQuery`: String - 增强后的规范化查询（后续节点使用此查询）

**设计说明**: 此节点不需要提取关键词，因为混合检索时 ES 等库会自行分词并计算相关性。

---

#### 4.2.4 SchemaRecallNode（Schema召回节点）

**文件位置**: `workflow/node/SchemaRecallNode.java`

**核心职责**: 根据增强后的查询检索相关的数据表和字段信息

**实现逻辑**:
```java
// 1. 获取增强后的查询
QueryEnhanceOutputDTO queryEnhanceOutputDTO = 
    StateUtil.getObjectValue(state, QUERY_ENHANCE_NODE_OUTPUT);
String input = queryEnhanceOutputDTO.getCanonicalQuery();

// 2. 召回相关表文档（通过向量检索）
List<Document> tableDocuments = schemaService.getTableDocumentsForAgent(agentId, input);

// 3. 提取表名
List<String> recalledTableNames = extractTableName(tableDocuments);

// 4. 根据表名获取对应的字段文档
List<Document> columnDocuments = schemaService.getColumnDocumentsByTableName(agentId, recalledTableNames);
```

**错误处理**:
如果未检索到相关数据表，会返回友好提示:
- 数据源尚未初始化
- 提问与数据库表结构无关
- Embedding 模型更换后需重新初始化

---

#### 4.2.5 TableRelationNode（表关系节点）

**文件位置**: `workflow/node/TableRelationNode.java`

**核心职责**: 分析表间关联关系，合并物理外键和逻辑外键

**实现逻辑**:
```java
// 1. 获取逻辑外键（用户配置的虚拟外键）
List<String> logicalForeignKeys = getLogicalForeignKeys(agentId, tableDocuments);

// 2. 构建初始 Schema（包含物理外键）
SchemaDTO initialSchema = buildInitialSchema(agentId, columnDocuments, 
    tableDocuments, agentDbConfig, logicalForeignKeys);

// 3. 合并逻辑外键到 Schema
if (logicalForeignKeys != null && !logicalForeignKeys.isEmpty()) {
    List<String> existingForeignKeys = schemaDTO.getForeignKeys();
    List<String> allForeignKeys = new ArrayList<>(existingForeignKeys);
    allForeignKeys.addAll(logicalForeignKeys);
    schemaDTO.setForeignKeys(allForeignKeys);
}

// 4. 调用 LLM 进行精细选择（fineSelect）
Flux<ChatResponse> schemaFlux = nl2SqlService.fineSelect(
    schemaDTO, input, evidence, schemaAdvice, agentDbConfig, dtoConsumer);

// 5. 获取语义模型并生成 prompt
List<SemanticModel> semanticModels = semanticModelService
    .getByAgentIdAndTableNames(agentId, tableNames);
String semanticModelPrompt = buildSemanticModelPrompt(semanticModels);
```

**逻辑外键格式**: `source_table.source_column=target_table.target_column`

---

#### 4.2.6 FeasibilityAssessmentNode（可行性评估节点）

**文件位置**: `workflow/node/FeasibilityAssessmentNode.java`

**核心职责**: 评估用户需求是否可以通过当前 Schema 和知识实现

**实现逻辑**:
```java
// 1. 获取必要信息
String canonicalQuery = StateUtil.getCanonicalQuery(state);
SchemaDTO recalledSchema = StateUtil.getObjectValue(state, TABLE_RELATION_OUTPUT);
String evidence = StateUtil.getStringValue(state, EVIDENCE);
String multiTurn = StateUtil.getStringValue(state, MULTI_TURN_CONTEXT);

// 2. 构建可行性评估提示词
String prompt = PromptHelper.buildFeasibilityAssessmentPrompt(
    canonicalQuery, recalledSchema, evidence, multiTurn);

// 3. 调用 LLM 评估
Flux<ChatResponse> responseFlux = llmService.callUser(prompt);

// 4. 返回评估结果（通过/不通过）
return Map.of(FEASIBILITY_ASSESSMENT_NODE_OUTPUT, assessmentResult);
```

**评估维度**:
- 数据分析请求 → 继续执行
- 需要澄清 → 返回澄清问题
- 闲聊/无法处理 → 结束流程

---

#### 4.2.7 PlannerNode（计划生成节点）

**文件位置**: `workflow/node/PlannerNode.java`

**核心职责**: 生成分析执行计划（JSON 格式）

**实现逻辑**:
```java
// 1. 检查是否为纯 NL2SQL 模式
Boolean onlyNl2sql = state.value(IS_ONLY_NL2SQL, false);
if (onlyNl2sql) {
    return Flux.just(Plan.nl2SqlPlan());  // 返回预定义的简单计划
}

// 2. 获取上下文信息
String canonicalQuery = StateUtil.getCanonicalQuery(state);
String semanticModel = state.value(GENEGRATED_SEMANTIC_MODEL_PROMPT);
SchemaDTO schemaDTO = StateUtil.getObjectValue(state, TABLE_RELATION_OUTPUT);
String schemaStr = PromptHelper.buildMixMacSqlDbPrompt(schemaDTO, true);
String evidence = StateUtil.getStringValue(state, EVIDENCE);

// 3. 检查是否为重新规划（用户拒绝后）
String validationError = StateUtil.getStringValue(state, PLAN_VALIDATION_ERROR);
if (validationError != null) {
    // 构建包含用户反馈的 prompt
    userPrompt = String.format("User rejected previous plan with feedback: \"%s\"...", 
        validationError);
}

// 4. 生成计划
BeanOutputConverter<Plan> beanOutputConverter = new BeanOutputConverter<>(Plan.class);
Map<String, Object> params = Map.of(
    "user_question", userPrompt,
    "schema", schemaStr,
    "evidence", evidence,
    "semantic_model", semanticModel,
    "format", beanOutputConverter.getFormat()
);
String plannerPrompt = PromptConstant.getPlannerPromptTemplate().render(params);
return llmService.callUser(plannerPrompt);
```

**输出结构** (`Plan`):
```json
{
  "thoughtProcess": "分析思考过程...",
  "executionPlan": [
    {
      "step": 1,
      "toolToUse": "sql_generate_node",
      "toolParameters": {
        "instruction": "查询用户订单数据...",
        "sqlQuery": null
      }
    },
    {
      "step": 2,
      "toolToUse": "report_generator_node",
      "toolParameters": {
        "summaryAndRecommendations": "生成报告..."
      }
    }
  ]
}
```

---

#### 4.2.8 PlanExecutorNode（计划执行节点）

**文件位置**: `workflow/node/PlanExecutorNode.java`

**核心职责**: 验证计划有效性，决定下一个执行节点

**实现逻辑**:
```java
// 1. 验证计划格式
Plan plan = PlanProcessUtil.getPlan(state);

// 2. 验证执行计划结构
if (!validateExecutionPlanStructure(plan)) {
    return buildValidationResult(state, false, "计划为空或没有执行步骤");
}

// 3. 验证每个执行步骤
for (ExecutionStep step : plan.getExecutionPlan()) {
    String validationResult = validateExecutionStep(step);
    if (validationResult != null) {
        return buildValidationResult(state, false, validationResult);
    }
}

// 4. 检查是否启用人工审核
Boolean humanReviewEnabled = state.value(HUMAN_REVIEW_ENABLED, false);
if (Boolean.TRUE.equals(humanReviewEnabled)) {
    return Map.of(PLAN_NEXT_NODE, HUMAN_FEEDBACK_NODE);
}

// 5. 确定下一个执行节点
ExecutionStep executionStep = executionPlan.get(currentStep - 1);
String toolToUse = executionStep.getToolToUse();
return determineNextNode(toolToUse);
```

**支持的节点类型**:
- `sql_generate_node` - SQL 生成
- `python_generate_node` - Python 生成
- `report_generator_node` - 报告生成

---

#### 4.2.9 HumanFeedbackNode（人工反馈节点）

**文件位置**: `workflow/node/HumanFeedbackNode.java`

**核心职责**: 处理用户对计划的审批或拒绝

**实现逻辑**:
```java
// 1. 检查最大修复次数（防止无限循环）
int repairCount = StateUtil.getObjectValue(state, PLAN_REPAIR_COUNT, Integer.class, 0);
if (repairCount >= 3) {
    return Map.of("human_next_node", "END");  // 超过3次，结束流程
}

// 2. 获取用户反馈数据
Map<String, Object> feedbackData = StateUtil.getObjectValue(state, HUMAN_FEEDBACK_DATA);
if (feedbackData.isEmpty()) {
    return Map.of("human_next_node", "WAIT_FOR_FEEDBACK");  // 等待反馈
}

// 3. 处理反馈结果
boolean approved = (Boolean) feedbackData.getOrDefault("feedback", true);

if (approved) {
    // 同意 → 继续执行
    updated.put("human_next_node", PLAN_EXECUTOR_NODE);
    updated.put(HUMAN_REVIEW_ENABLED, false);
} else {
    // 拒绝 → 重新规划
    updated.put("human_next_node", PLANNER_NODE);
    updated.put(PLAN_REPAIR_COUNT, repairCount + 1);
    updated.put(PLAN_VALIDATION_ERROR, feedbackContent);  // 保存用户反馈内容
}
```

---

#### 4.2.10 SqlGenerateNode（SQL生成节点）

**文件位置**: `workflow/node/SqlGenerateNode.java`

**核心职责**: 根据计划步骤生成 SQL 语句

**实现逻辑**:
```java
// 1. 检查是否达到最大重试次数
int count = state.value(SQL_GENERATE_COUNT, 0);
if (count >= properties.getMaxSqlRetryCount()) {
    // 超限，返回错误信息
    return Map.of(SQL_GENERATE_OUTPUT, StateGraph.END);
}

// 2. 获取当前步骤的 SQL 任务要求
String promptForSql = getCurrentExecutionStepInstruction(state);

// 3. 根据重试原因选择不同的处理方式
SqlRetryDto retryDto = StateUtil.getObjectValue(state, SQL_REGENERATE_REASON);

if (retryDto.sqlExecuteFail()) {
    // SQL 执行失败，带上错误信息重新生成
    sqlFlux = handleRetryGenerateSql(state, originalSql, errorMsg, promptForSql);
} else if (retryDto.semanticFail()) {
    // 语义一致性校验失败，带上校验结果重新生成
    sqlFlux = handleRetryGenerateSql(state, originalSql, errorMsg, promptForSql);
} else {
    // 首次生成
    sqlFlux = handleGenerateSql(state, promptForSql);
}

// 4. 调用 Nl2SqlService 生成 SQL
SqlGenerationDTO sqlGenerationDTO = SqlGenerationDTO.builder()
    .evidence(evidence)
    .query(userQuery)
    .schemaDTO(schemaDTO)
    .sql(originalSql)  // 重试时带上原SQL
    .exceptionMessage(errorMsg)  // 重试时带上错误信息
    .dialect(dialect)
    .build();
return nl2SqlService.generateSql(sqlGenerationDTO);
```

**重试机制**:
- 最大重试次数由 `DataAgentProperties.maxSqlRetryCount` 配置
- 每次重试会将错误信息反馈给 LLM

---

#### 4.2.11 SemanticConsistencyNode（语义一致性节点）

**文件位置**: `workflow/node/SemanticConsistencyNode.java`

**核心职责**: 验证生成的 SQL 是否与用户意图语义一致

**实现逻辑**:
```java
// 1. 获取必要信息
String sql = StateUtil.getStringValue(state, SQL_GENERATE_OUTPUT);
String userQuery = StateUtil.getCanonicalQuery(state);
SchemaDTO schemaDTO = StateUtil.getObjectValue(state, TABLE_RELATION_OUTPUT);
String evidence = StateUtil.getStringValue(state, EVIDENCE);

// 2. 构建语义一致性校验 DTO
SemanticConsistencyDTO dto = SemanticConsistencyDTO.builder()
    .dialect(dialect)
    .sql(sql)
    .executionDescription(getCurrentExecutionStepInstruction(state))
    .schemaInfo(buildMixMacSqlDbPrompt(schemaDTO, true))
    .userQuery(userQuery)
    .evidence(evidence)
    .build();

// 3. 调用 LLM 进行语义一致性校验
Flux<ChatResponse> validationResultFlux = 
    nl2SqlService.performSemanticConsistency(dto);

// 4. 解析校验结果
boolean isPassed = !validationResult.startsWith("不通过");
if (!isPassed) {
    // 校验不通过，触发 SQL 重新生成
    return Map.of(SQL_REGENERATE_REASON, SqlRetryDto.semantic(validationResult));
}
```

**校验逻辑**: LLM 返回 "通过" 或 "不通过：原因..."

---

#### 4.2.12 SqlExecuteNode（SQL执行节点）

**文件位置**: `workflow/node/SqlExecuteNode.java`

**核心职责**: 执行 SQL 查询并处理结果

**实现逻辑**:
```java
// 1. 获取 SQL 和数据库配置
String sqlQuery = StateUtil.getStringValue(state, SQL_GENERATE_OUTPUT);
DbConfigBO dbConfig = databaseUtil.getAgentDbConfig(agentId);
Accessor dbAccessor = databaseUtil.getAgentAccessor(agentId);

// 2. 执行 SQL 查询
DbQueryParameter dbQueryParameter = new DbQueryParameter();
dbQueryParameter.setSql(sqlQuery);
dbQueryParameter.setSchema(dbConfig.getSchema());

ResultSetBO resultSetBO = dbAccessor.executeSqlAndReturnObject(dbConfig, dbQueryParameter);

// 3. 调用 LLM 获取图表配置（可选）
if (properties.isEnableSqlResultChart()) {
    DisplayStyleBO displayStyleBO = enrichResultSetWithChartConfig(state, resultSetBO);
    resultBO.setDisplayStyle(displayStyleBO);
}

// 4. 更新步骤结果
Map<String, String> updatedResults = PlanProcessUtil.addStepResult(
    existingResults, currentStep, strResultSetJson);

// 5. 错误处理 - 触发 SQL 重新生成
catch (Exception e) {
    result.put(SQL_REGENERATE_REASON, SqlRetryDto.sqlExecute(errorMessage));
}
```

**图表配置**: 通过 LLM 分析查询结果，自动推荐合适的图表类型（柱状图/折线图/饼图/表格）

---

#### 4.2.13 PythonGenerateNode（Python生成节点）

**文件位置**: `workflow/node/PythonGenerateNode.java`

**核心职责**: 根据分析需求生成 Python 代码

**实现逻辑**:
```java
// 1. 获取上下文信息
SchemaDTO schemaDTO = StateUtil.getObjectValue(state, TABLE_RELATION_OUTPUT);
List<Map<String, String>> sqlResults = StateUtil.getListValue(state, SQL_RESULT_LIST_MEMORY);
boolean codeRunSuccess = StateUtil.getObjectValue(state, PYTHON_IS_SUCCESS);
int triesCount = StateUtil.getObjectValue(state, PYTHON_TRIES_COUNT, 0);

// 2. 如果上次执行失败，构建包含错误信息的 prompt
if (!codeRunSuccess) {
    String lastCode = StateUtil.getStringValue(state, PYTHON_GENERATE_NODE_OUTPUT);
    String lastError = StateUtil.getStringValue(state, PYTHON_EXECUTE_NODE_OUTPUT);
    userPrompt += String.format("""
        上次尝试生成的Python代码运行失败，请你重新生成符合要求的Python代码。
        【上次生成代码】
        ```python
        %s
        ```
        【运行错误信息】
        ```
        %s
        ```
        """, lastCode, lastError);
}

// 3. 加载 Python 代码生成模板
String systemPrompt = PromptConstant.getPythonGeneratorPromptTemplate().render(Map.of(
    "python_memory", codeExecutorProperties.getLimitMemory(),
    "python_timeout", codeExecutorProperties.getCodeTimeout(),
    "database_schema", objectMapper.writeValueAsString(schemaDTO),
    "sample_input", objectMapper.writeValueAsString(sqlResults.stream().limit(5).toList()),
    "plan_description", objectMapper.writeValueAsString(toolParameters)
));

// 4. 调用 LLM 生成 Python 代码
Flux<ChatResponse> pythonGenerateFlux = llmService.call(systemPrompt, userPrompt);
```

---

#### 4.2.14 PythonExecuteNode（Python执行节点）

**文件位置**: `workflow/node/PythonExecuteNode.java`

**核心职责**: 在隔离环境中执行 Python 代码

**实现逻辑**:
```java
// 1. 获取 Python 代码和 SQL 结果数据
String pythonCode = StateUtil.getStringValue(state, PYTHON_GENERATE_NODE_OUTPUT);
List<Map<String, String>> sqlResults = StateUtil.getListValue(state, SQL_RESULT_LIST_MEMORY);

// 2. 检查重试次数
int triesCount = StateUtil.getObjectValue(state, PYTHON_TRIES_COUNT, 0);

// 3. 构建任务请求
CodePoolExecutorService.TaskRequest taskRequest = new TaskRequest(
    pythonCode,
    objectMapper.writeValueAsString(sqlResults),
    null
);

// 4. 执行 Python 代码
CodePoolExecutorService.TaskResponse taskResponse = codePoolExecutor.runTask(taskRequest);

// 5. 处理执行结果
if (!taskResponse.isSuccess()) {
    // 检查是否超过最大重试次数
    if (triesCount >= codeExecutorProperties.getPythonMaxTriesCount()) {
        // 启动降级兜底逻辑
        return Map.of(PYTHON_FALLBACK_MODE, true);
    }
    throw new RuntimeException(errorMsg);  // 触发重试
}

// 6. 处理 Unicode 转义（Python输出的JSON可能有Unicode形式）
String stdout = taskResponse.stdOut();
Object value = jsonParseUtil.tryConvertToObject(stdout, Object.class);
if (value != null) {
    stdout = objectMapper.writeValueAsString(value);
}
```

**降级策略**: 超过最大重试次数后，进入降级模式，允许流程继续执行

---

#### 4.2.15 PythonAnalyzeNode（Python分析节点）

**文件位置**: `workflow/node/PythonAnalyzeNode.java`

**核心职责**: 对 Python 代码的运行结果进行总结分析

**实现逻辑**:
```java
// 1. 获取上下文信息
String userQuery = StateUtil.getCanonicalQuery(state);
String pythonOutput = StateUtil.getStringValue(state, PYTHON_EXECUTE_NODE_OUTPUT);
int currentStep = PlanProcessUtil.getCurrentStepNumber(state);

// 2. 检查是否进入降级模式
boolean isFallbackMode = StateUtil.getObjectValue(state, PYTHON_FALLBACK_MODE, false);
if (isFallbackMode) {
    // 返回固定提示信息
    String fallbackMessage = "Python 高级分析功能暂时不可用，出现错误";
    return Map.of(SQL_EXECUTE_NODE_OUTPUT, updatedSqlResult, PLAN_CURRENT_STEP, currentStep + 1);
}

// 3. 构建分析提示词
String systemPrompt = PromptConstant.getPythonAnalyzePromptTemplate().render(Map.of(
    "python_output", pythonOutput,
    "user_query", userQuery
));

// 4. 调用 LLM 分析结果
Flux<ChatResponse> pythonAnalyzeFlux = llmService.callSystem(systemPrompt);

// 5. 将分析结果存入步骤结果
Map<String, String> updatedSqlResult = PlanProcessUtil.addStepResult(
    sqlExecuteResult, currentStep, aiResponse);
```

---

#### 4.2.16 ReportGeneratorNode（报告生成节点）

**文件位置**: `workflow/node/ReportGeneratorNode.java`

**核心职责**: 生成包含图表的综合分析报告

**实现逻辑**:
```java
// 1. 获取执行计划和结果
String plannerNodeOutput = StateUtil.getStringValue(state, PLANNER_NODE_OUTPUT);
Plan plan = converter.convert(plannerNodeOutput);
HashMap<String, String> executionResults = StateUtil.getObjectValue(
    state, SQL_EXECUTE_NODE_OUTPUT);

// 2. 获取当前步骤的摘要建议
ExecutionStep executionStep = getCurrentExecutionStep(plan, currentStep);
String summaryAndRecommendations = executionStep.getToolParameters()
    .getSummaryAndRecommendations();

// 3. 构建报告内容
String userRequirementsAndPlan = buildUserRequirementsAndPlan(userInput, plan);
String analysisStepsAndData = buildAnalysisStepsAndData(plan, executionResults);

// 4. 获取优化配置（支持按智能体个性化配置）
List<UserPromptConfig> optimizationConfigs = 
    promptConfigService.getOptimizationConfigs("report-generator", agentId);

// 5. 生成最终报告
String reportPrompt = PromptHelper.buildReportGeneratorPromptWithOptimization(
    userRequirementsAndPlan,
    analysisStepsAndData,
    summaryAndRecommendations,
    optimizationConfigs
);
return llmService.callUser(reportPrompt);
```

**报告内容结构**:
```markdown
## 用户原始需求
...

## 执行计划概述
**思考过程**: ...

## 详细执行步骤
### 步骤 1: 步骤编号 1
**工具**: sql_generate_node
**参数描述**: ...

## 数据执行结果
### step_1
**执行SQL**: ```sql ... ```
**执行结果**: ```json ... ```
```

### 4.3 调度器 (Dispatcher)

每个关键节点配备调度器，负责流程控制和错误处理：

| 调度器 | 关联节点 | 职责 |
|:---|:---|:---|
| `IntentRecognitionDispatcher` | IntentRecognitionNode | 意图路由决策 |
| `SchemaRecallDispatcher` | SchemaRecallNode | Schema 召回控制 |
| `QueryEnhanceDispatcher` | QueryEnhanceNode | 查询增强控制 |
| `TableRelationDispatcher` | TableRelationNode | 表关系分析重试 |
| `FeasibilityAssessmentDispatcher` | FeasibilityAssessmentNode | 可行性判断 |
| `PlanExecutorDispatcher` | PlanExecutorNode | 步骤选择与调度 |
| `HumanFeedbackDispatcher` | HumanFeedbackNode | 反馈处理路由 |
| `SqlGenerateDispatcher` | SqlGenerateNode | SQL 生成控制 |
| `SemanticConsistenceDispatcher` | SemanticConsistencyNode | 语义校验控制 |
| `SQLExecutorDispatcher` | SqlExecuteNode | SQL 执行与重试 |
| `PythonExecutorDispatcher` | PythonExecuteNode | Python 执行控制 |

#### 调度器核心逻辑示例

**IntentRecognitionDispatcher** - 意图路由:
```java
// 根据意图识别结果决定下一节点
if (intentOutput.getNeedAnalysis()) {
    return EVIDENCE_RECALL_NODE;  // 需要分析，进入证据召回
} else {
    return END;  // 闲聊，直接结束
}
```

**PlanExecutorDispatcher** - 计划执行路由:
```java
// 根据计划验证状态和下一节点类型路由
Boolean validationStatus = state.value(PLAN_VALIDATION_STATUS);
if (!validationStatus) {
    return PLANNER_NODE;  // 验证失败，重新规划
}
String nextNode = state.value(PLAN_NEXT_NODE);
return nextNode;  // sql_generate_node / python_generate_node / report_generator_node
```

**SQLExecutorDispatcher** - SQL 执行重试:
```java
// 检查是否需要重试
SqlRetryDto retryDto = state.value(SQL_REGENERATE_REASON);
if (retryDto.needRetry()) {
    return SQL_GENERATE_NODE;  // 重新生成 SQL
}
return PLAN_EXECUTOR_NODE;  // 继续执行下一步骤
```

**PythonExecutorDispatcher** - Python 执行重试:
```java
// 检查执行是否成功
Boolean isSuccess = state.value(PYTHON_IS_SUCCESS);
if (!isSuccess) {
    return PYTHON_GENERATE_NODE;  // 重新生成 Python 代码
}
return PYTHON_ANALYZE_NODE;  // 进入结果分析
```

### 4.4 执行流程图

```
                              ┌─────────┐
                              │  Start  │
                              └────┬────┘
                                   │
                                   ▼
                         ┌─────────────────┐
                         │ BuildMultiTurn  │
                         │    Context      │
                         └────────┬────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │   IntentRecognitionNode  │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                 需要分析                   不需要
                    │                         │
                    ▼                         ▼
          ┌─────────────────┐              ┌─────┐
          │ EvidenceRecall  │              │ End │
          └────────┬────────┘              └─────┘
                   │
                   ▼
          ┌─────────────────┐
          │  QueryEnhance   │
          └────────┬────────┘
                   │
                   ▼
          ┌─────────────────┐
          │  SchemaRecall   │
          └────────┬────────┘
                   │
                   ▼
          ┌─────────────────┐
          │ TableRelation   │◄────┐
          └────────┬────────┘     │
                   │              │重试
           ┌───────┴───────┐      │
           │               │      │
          成功            失败────┘
           │
           ▼
   ┌───────────────────┐
   │ FeasibilityAssess │
   └─────────┬─────────┘
             │
     ┌───────┴───────┐
     │               │
    可行           不可行
     │               │
     ▼               ▼
┌─────────┐       ┌─────┐
│ Planner │◄──┐   │ End │
└────┬────┘   │   └─────┘
     │        │
     ▼        │
┌────────────┐│
│PlanExecutor││
└─────┬──────┘│
      │       │计划无效/被拒绝
      ▼       │
 ┌──────────┐ │
 │ Human    │─┤
 │ Feedback │ │
 └────┬─────┘ │
      │       │
  同意│       │
      ▼       │
┌───────────────────────────────────────────┐
│           Step Selection Loop              │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │         SQL Step Branch              │  │
│  │  SqlGenerate → SemanticCheck        │  │
│  │       ↓                             │  │
│  │  SqlExecute ← 重试(失败)            │  │
│  │       ↓                             │  │
│  │  Store SQL Result                   │  │
│  └─────────────────────────────────────┘  │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │       Python Step Branch             │  │
│  │  PythonGenerate → PythonExecute     │  │
│  │       ↓            ↑                │  │
│  │  PythonAnalyze  ←重试(失败)         │  │
│  │       ↓                             │  │
│  │  Store Analysis Result              │  │
│  └─────────────────────────────────────┘  │
│                                           │
└───────────────────────┬───────────────────┘
                        │
                        ▼
               ┌─────────────────┐
               │ ReportGenerator │
               └────────┬────────┘
                        │
                        ▼
                    ┌─────┐
                    │ End │
                    └─────┘
```

---

## 五、前端架构设计

### 5.1 目录结构

```
data-agent-frontend/src/
├── App.vue                         # 应用根组件
├── main.js                         # 应用入口
│
├── components/                     # 组件目录
│   ├── agent/                      # 智能体配置组件
│   │   ├── AccessApi.vue           # API Key 管理界面
│   │   ├── AgentKnowledgeConfig.vue    # 智能体知识配置
│   │   ├── BaseSetting.vue         # 基础设置
│   │   ├── BatchImportDialog.vue   # 批量导入对话框
│   │   ├── BusinessKnowledgeConfig.vue # 业务知识配置
│   │   ├── DataSourceConfig.vue    # 数据源配置
│   │   ├── PresetsConfig.vue       # 预设问题配置
│   │   ├── PromptConfig.vue        # Prompt 配置
│   │   └── SemanticsConfig.vue     # 语义模型配置
│   │
│   └── run/                        # 运行时组件
│       ├── ChartComponent.vue      # 通用图表组件
│       ├── ChatSessionSidebar.vue  # 会话侧边栏
│       ├── HumanFeedback.vue       # 人工反馈组件
│       ├── PresetQuestions.vue     # 预设问题展示
│       ├── ResultSetDisplay.vue    # 结果集表格展示
│       │
│       ├── charts/                 # 图表实现
│       │   ├── BaseChart.ts        # 图表基类
│       │   ├── ChartFactory.ts     # 图表工厂
│       │   ├── BarChart.ts         # 柱状图
│       │   ├── LineChart.ts        # 折线图
│       │   └── PieChart.ts         # 饼图
│       │
│       └── markdown/               # Markdown 渲染
│           ├── index.ts            # 导出入口
│           ├── markdown-plugin-echarts.ts  # ECharts 插件
│           ├── markdown-plugin-highlight.ts # 代码高亮
│           └── MarkdownAgentContainer.vue  # 容器组件
│
├── views/                          # 页面视图
│   ├── AgentCreate.vue             # 创建智能体
│   ├── AgentDetail.vue             # 智能体详情/编辑
│   ├── AgentList.vue               # 智能体列表
│   ├── AgentRun.vue                # 智能体运行界面
│   ├── ModelConfig.vue             # 模型配置管理
│   └── NotFound.vue                # 404 页面
│
├── services/                       # API 服务层
│   ├── agent.ts                    # 智能体 CRUD
│   ├── agentDatasource.ts          # 智能体数据源
│   ├── agentKnowledge.ts           # 智能体知识
│   ├── businessKnowledge.ts        # 业务知识
│   ├── chat.ts                     # 聊天/会话
│   ├── common.ts                   # 通用工具
│   ├── datasource.ts               # 数据源管理
│   ├── fileUpload.ts               # 文件上传
│   ├── graph.ts                    # 图执行 (SSE)
│   ├── logicalRelation.ts          # 逻辑外键
│   ├── modelConfig.ts              # 模型配置
│   ├── presetQuestion.ts           # 预设问题
│   ├── resultSet.ts                # 结果集
│   ├── semanticModel.ts            # 语义模型
│   └── sessionStateManager.ts      # 会话状态管理
│
├── router/                         # 路由配置
│   ├── index.js                    # 路由实例
│   └── routes.js                   # 路由定义
│
├── layouts/                        # 布局组件
│   └── BaseLayout.vue              # 基础布局
│
└── styles/                         # 全局样式
    └── global.css
```

### 5.2 核心页面功能

| 页面 | 路由 | 功能 |
|:---|:---|:---|
| AgentList | `/agents` | 智能体列表展示、搜索、状态管理 |
| AgentCreate | `/agents/create` | 创建新智能体 |
| AgentDetail | `/agents/:id` | 智能体详情编辑（多 Tab 配置） |
| AgentRun | `/agents/:id/run` | 智能体对话运行界面 |
| ModelConfig | `/models` | LLM/Embedding 模型配置管理 |

### 5.3 SSE 流式通信

前端通过 Server-Sent Events (SSE) 接收实时流式响应：

```typescript
// services/graph.ts 示例
export function streamSearch(agentId: string, query: string) {
  const eventSource = new EventSource(
    `/api/graph/stream?agentId=${agentId}&query=${encodeURIComponent(query)}`
  );
  
  eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    // 根据 TextType 处理不同内容类型
    switch (data.textType) {
      case 'SQL':
        // 渲染 SQL 代码块
        break;
      case 'HTML':
        // 渲染 HTML 报告
        break;
      case 'MARKDOWN':
        // 渲染 Markdown
        break;
    }
  };
}
```

---

## 六、数据模型设计

### 6.1 核心数据表

#### agent (智能体)
```sql
CREATE TABLE agent (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL,
  description VARCHAR(500),
  status VARCHAR(20) DEFAULT 'draft',  -- draft/published/offline
  human_review_enabled BOOLEAN DEFAULT FALSE,
  api_key VARCHAR(64),
  api_key_enabled BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

#### datasource (数据源)
```sql
CREATE TABLE datasource (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL,
  type VARCHAR(20) NOT NULL,  -- mysql/postgresql/...
  host VARCHAR(255) NOT NULL,
  port INT NOT NULL,
  database_name VARCHAR(100) NOT NULL,
  username VARCHAR(100),
  password VARCHAR(255),  -- 加密存储
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### agent_knowledge (智能体知识)
```sql
CREATE TABLE agent_knowledge (
  id INT PRIMARY KEY AUTO_INCREMENT,
  agent_id INT NOT NULL,
  title VARCHAR(200) NOT NULL,
  content TEXT,
  type VARCHAR(50),
  enabled BOOLEAN DEFAULT TRUE,
  FOREIGN KEY (agent_id) REFERENCES agent(id)
);
```

#### business_knowledge (业务知识)
```sql
CREATE TABLE business_knowledge (
  id INT PRIMARY KEY AUTO_INCREMENT,
  title VARCHAR(200) NOT NULL,
  content TEXT,
  category VARCHAR(100),
  enabled BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### semantic_model (语义模型)
```sql
CREATE TABLE semantic_model (
  id INT PRIMARY KEY AUTO_INCREMENT,
  datasource_id INT NOT NULL,
  table_name VARCHAR(100) NOT NULL,
  column_name VARCHAR(100),
  business_name VARCHAR(200),
  description VARCHAR(500),
  FOREIGN KEY (datasource_id) REFERENCES datasource(id)
);
```

#### logical_relation (逻辑外键)
```sql
CREATE TABLE logical_relation (
  id INT PRIMARY KEY AUTO_INCREMENT,
  datasource_id INT NOT NULL,
  source_table_name VARCHAR(100),
  source_column_name VARCHAR(100),
  target_table_name VARCHAR(100),
  target_column_name VARCHAR(100),
  relation_type VARCHAR(20),  -- 1:1, 1:N, N:1
  description VARCHAR(500),
  FOREIGN KEY (datasource_id) REFERENCES datasource(id)
);
```

#### model_config (模型配置)
```sql
CREATE TABLE model_config (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL,
  type VARCHAR(20) NOT NULL,  -- chat/embedding
  provider VARCHAR(50),
  model_name VARCHAR(100),
  api_key VARCHAR(255),
  base_url VARCHAR(255),
  is_active BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### user_prompt_config (Prompt 配置)
```sql
CREATE TABLE user_prompt_config (
  id INT PRIMARY KEY AUTO_INCREMENT,
  agent_id INT,  -- NULL 表示全局配置
  type VARCHAR(50) NOT NULL,  -- report-generator/planner/sql-generator/...
  content TEXT,
  priority INT DEFAULT 0,
  display_order INT DEFAULT 0,
  enabled BOOLEAN DEFAULT TRUE
);
```

### 6.2 实体关系图

```
┌─────────────┐     1:N     ┌──────────────────┐
│   agent     │─────────────│ agent_knowledge  │
└─────────────┘             └──────────────────┘
       │
       │ N:1
       ▼
┌─────────────┐     1:N     ┌──────────────────┐
│ datasource  │─────────────│ semantic_model   │
└─────────────┘             └──────────────────┘
       │
       │ 1:N
       ▼
┌─────────────────┐
│ logical_relation│
└─────────────────┘

┌─────────────────┐
│business_knowledge│ (全局知识，不关联特定智能体)
└─────────────────┘

┌─────────────────┐
│  model_config   │ (全局模型配置)
└─────────────────┘

┌───────────────────┐
│ user_prompt_config│ (可关联 agent_id 或全局)
└───────────────────┘
```

---

## 七、关键技术实现

### 7.1 流式输出机制

**技术方案**：Spring WebFlux + Server-Sent Events (SSE)

```java
// GraphController.java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> streamSearch(
    @RequestParam String agentId,
    @RequestParam String query,
    @RequestParam(required = false) Boolean humanFeedback) {
    
    return graphService.graphStreamProcess(agentId, query, humanFeedback)
        .map(chunk -> ServerSentEvent.builder(chunk).build());
}
```

**文本类型标记**：
```java
public enum TextType {
    TEXT,       // 普通文本
    SQL,        // SQL 代码
    JSON,       // JSON 数据
    HTML,       // HTML 内容
    MARKDOWN,   // Markdown 内容
    ERROR       // 错误信息
}
```

### 7.2 多轮对话上下文

**实现类**：`MultiTurnContextManager`

**核心功能**：
- 维护每个会话的对话历史
- 将历史注入后续请求的 Prompt
- 支持配置最大历史轮数

```java
public class MultiTurnContextManager {
    
    private final Map<String, List<TurnRecord>> contextStore = new ConcurrentHashMap<>();
    
    public void beginTurn(String sessionId, String query) {
        // 开始新一轮对话
    }
    
    public String buildContext(String sessionId) {
        // 构建包含历史的上下文
    }
    
    public void finishTurn(String sessionId, String result) {
        // 完成当前轮，记录结果
    }
}
```

### 7.3 Python 代码执行

**架构设计**：

```
CodePoolExecutorService
        │
        ├── DockerCodeExecutor (推荐)
        │       └── 使用 docker-java 库
        │       └── 镜像: continuumio/anaconda3:latest
        │
        ├── LocalCodeExecutor
        │       └── 本地 Python 进程
        │
        └── AiSimulationExecutor
                └── AI 模拟执行结果
```

**配置示例**：
```yaml
spring:
  ai:
    alibaba:
      data-agent:
        code-executor:
          type: docker
          docker:
            image: continuumio/anaconda3:latest
            timeout: 300000
            memory-limit: 512m
            cpu-limit: 1.0
```

### 7.4 RAG 检索增强

**检索流程**：

```
用户问题
    │
    ▼
┌───────────────┐
│ LLM 查询重写  │ ← 结合多轮上下文
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ 构建过滤条件  │ ← 按 agent_id, type 过滤
└───────┬───────┘
        │
        ▼
┌───────────────────────────────────┐
│       HybridRetrievalStrategy      │
│  ┌─────────────┐  ┌─────────────┐ │
│  │ VectorSearch│  │KeywordSearch│ │
│  └──────┬──────┘  └──────┬──────┘ │
│         │                 │        │
│         └────────┬────────┘        │
│                  ▼                 │
│          FusionStrategy            │
└─────────────────┬─────────────────┘
                  │
                  ▼
           检索结果文档
```

### 7.5 逻辑外键处理

**问题背景**：生产数据库常无物理外键，导致 LLM 无法推断表关系。

**解决方案**：

1. 用户在前端配置逻辑外键
2. Schema 召回时自动加载逻辑外键
3. 过滤与召回表相关的外键
4. 合并物理外键和逻辑外键
5. 提供完整 Schema 给 SQL 生成

```java
private List<String> getLogicalForeignKeys(Integer agentId, List<Document> tableDocuments) {
    // 1. 获取数据源
    // 2. 提取召回表名
    // 3. 查询逻辑外键
    // 4. 过滤相关外键
    // 5. 格式化返回
    return formattedForeignKeys;
}
```

---

## 八、部署架构

### 8.1 环境要求

| 组件 | 版本要求 | 说明 |
|:---|:---|:---|
| JDK | 17+ | 必须 |
| MySQL | 5.7+ | 存储元数据 |
| Node.js | 16+ | 前端构建 |
| Docker | latest | Python 执行环境（可选） |

### 8.2 部署方式

#### 方式一：本地开发部署

```bash
# 1. 导入数据库
mysql -u root -p < data-agent-management/src/main/resources/sql/schema.sql

# 2. 启动后端
cd data-agent-management
./mvnw spring-boot:run

# 3. 启动前端
cd data-agent-frontend
npm install && npm run dev

# 4. 访问系统
open http://localhost:3000
```

#### 方式二：Docker Compose 部署

参考 `docker-file/` 目录下的配置文件。

### 8.3 配置参数

```yaml
spring:
  ai:
    alibaba:
      data-agent:
        # LLM 服务类型
        llm-service-type: STREAM  # STREAM 或 BLOCK
        
        # 多轮对话配置
        multi-turn:
          enabled: true
          max-history: 10
          context-window: 4096
        
        # 向量存储配置
        vector-store:
          enable-hybrid-search: true
          similarity-threshold: 0.7
          top-k: 10
        
        # 代码执行器配置
        code-executor:
          type: docker
          docker:
            image: continuumio/anaconda3:latest
            timeout: 300000
        
        # 计划执行配置
        plan-executor:
          max-retry: 3
          timeout: 600000
```

---

## 附录

### A. 相关文档

| 文档 | 说明 |
|:---|:---|
| [快速开始](QUICK_START.md) | 安装配置指南 |
| [架构设计](ARCHITECTURE.md) | 系统架构详解 |
| [开发者指南](DEVELOPER_GUIDE.md) | 贡献指南和代码规范 |
| [高级功能](ADVANCED_FEATURES.md) | API 调用和 MCP 配置 |
| [知识配置最佳实践](KNOWLEDGE_USAGE.md) | 知识库配置指南 |

### B. 版本信息

- **文档版本**: 1.0
- **最后更新**: 2026-01-27
- **适用版本**: DataAgent 1.0.0-SNAPSHOT

---

<div align="center">
    Made with ❤️ by Spring AI Alibaba DataAgent Team
</div>
