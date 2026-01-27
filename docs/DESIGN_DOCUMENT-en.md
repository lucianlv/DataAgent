# DataAgent Project Design Document

[中文](./DESIGN_DOCUMENT.md) | English

> This document provides a detailed record of the product design and technical implementation details of the DataAgent project.

---

## Table of Contents

- [1. Product Overview](#1-product-overview)
- [2. Core Product Features](#2-core-product-features)
- [3. Technical Architecture Design](#3-technical-architecture-design)
- [4. StateGraph Workflow Engine](#4-stategraph-workflow-engine)
- [5. Frontend Architecture Design](#5-frontend-architecture-design)
- [6. Data Model Design](#6-data-model-design)
- [7. Key Technical Implementations](#7-key-technical-implementations)
- [8. Deployment Architecture](#8-deployment-architecture)

---

## 1. Product Overview

### 1.1 Product Positioning

**DataAgent** is an enterprise-grade intelligent data analysis Agent built on **Spring AI Alibaba Graph**. It goes beyond traditional Text-to-SQL tools, evolving into an AI-powered data analyst capable of executing **Python deep analysis** and generating **multi-dimensional chart reports**.

### 1.2 Core Value

| Value Dimension | Description |
|:---|:---|
| **Lower Barriers** | Non-technical users can obtain data insights through natural language |
| **Improve Efficiency** | Automate data analysis workflows, reduce repetitive work |
| **Enhanced Capabilities** | Combine Python for advanced analysis that traditional BI tools cannot achieve |
| **Knowledge Accumulation** | Accumulate and reuse business knowledge through RAG |

### 1.3 Technical Highlights

- **Fully Compatible with OpenAI API Specifications**: Supports integration with mainstream LLMs like Qwen, Deepseek
- **Flexible Vector Database Integration**: Pluggable design, adaptable to any vector storage
- **Native MCP Protocol Support**: Can serve as an MCP server integrated into Claude Desktop and similar tools
- **Enterprise-grade Features**: Complete API Key management, permission control, multi-tenant support

---

## 2. Core Product Features

### 2.1 Intelligent Data Analysis (Text-to-SQL)

**Description**: Converts users' natural language queries into executable SQL statements.

**Core Capabilities**:
- Natural language understanding and intent recognition
- Complex multi-table query support (JOIN, aggregation, subqueries)
- Multi-turn conversation context understanding
- SQL semantic consistency verification

**Use Case**:
```
User: "Query the top 10 products with highest sales in the past 30 days"
System: Automatically generates SQL, executes, and returns results
```

### 2.2 Python Deep Analysis

**Description**: Automatically generates and executes Python code based on analysis requirements for advanced data analysis.

**Core Capabilities**:
- Statistical analysis (mean, variance, correlation analysis)
- Trend prediction (time series analysis, regression prediction)
- Machine learning models (classification, clustering, anomaly detection)
- Data visualization generation

**Execution Environments**:
| Type | Description | Use Case |
|:---|:---|:---|
| Docker Executor | Containerized isolated execution | Production (Recommended) |
| Local Executor | Local Python environment | Development/Testing |
| AI Simulation | AI simulates execution results | Demo/Testing |

### 2.3 Intelligent Report Generation

**Description**: Automatically summarizes analysis results into reports with visualization charts.

**Output Formats**:
- HTML format (with interactive ECharts)
- Markdown format (suitable for documentation integration)

**Chart Types**:
- Bar Chart
- Line Chart
- Pie Chart
- More custom charts

### 2.4 Human Feedback Mechanism (Human-in-the-loop)

**Description**: Allows user intervention and adjustment during the analysis plan generation phase.

**Workflow**:
```
Plan Generation → Pause waiting for user feedback → Approve/Reject
                                                         │
                                           Approve ────→ Continue execution
                                           Reject ─────→ Re-plan
```

**Technical Implementation**:
- Request parameter `humanFeedback=true` to enable
- `CompiledGraph` uses `interruptBefore(HUMAN_FEEDBACK_NODE)` for pausing
- Resume execution via `threadId`

### 2.5 RAG Retrieval Enhancement

**Description**: Improves SQL generation accuracy through semantic retrieval of business knowledge base.

**Knowledge Types**:
| Type | Description |
|:---|:---|
| Business Knowledge | General business terms, calculation rules |
| Agent Knowledge | Domain knowledge specific to an agent |
| Semantic Model | Business semantic descriptions of tables and fields |

**Retrieval Strategies**:
- Vector retrieval (semantic similarity)
- Hybrid retrieval (vector + keyword)
- Dynamic filtering (by agent, knowledge type)

### 2.6 Multi-Model Orchestration

**Description**: Supports runtime dynamic switching between different LLM and Embedding models.

**Architecture Design**:
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

**Supported Features**:
- Hot-swap models without restart
- Only one model active per type at a time
- Fully compatible with OpenAI API specifications

### 2.7 MCP Server

**Description**: Compliant with Model Context Protocol, provides external tool capabilities.

**Available Tools**:

| Tool Name | Function |
|:---|:---|
| `nl2SqlToolCallback` | Natural language to SQL |
| `listAgentsToolCallback` | Query agent list |

**Configuration**:
```yaml
spring:
  ai:
    mcp:
      server:
        sse-endpoint: /sse  # Default endpoint
```

### 2.8 API Key Management

**Description**: Complete API Key lifecycle management with fine-grained permission control.

**Management Capabilities**:
- Generate API Key
- Reset API Key
- Enable/Disable
- Delete

**Usage**:
```bash
curl -X POST "http://localhost:8065/api/..." \
  -H "X-API-Key: <your_api_key>"
```

---

## 3. Technical Architecture Design

### 3.1 Overall Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Clients Layer                              │
│   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐        │
│   │  Frontend UI   │  │  Admin Console │  │   MCP Client   │        │
│   │   (Vue 3)      │  │                │  │ (Claude, etc.) │        │
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
│  │     │  16 Processing Nodes + 12 Dispatchers             │      │  │
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
   │(Biz Data)  │          │ (Metadata) │          │(Vector DB) │
   └────────────┘          └────────────┘          └────────────┘
          │
          │
   ┌────────────┐
   │  Python    │
   │  Runtime   │
   │(Docker/Local)│
   └────────────┘
```

### 3.2 Technology Stack Details

| Layer | Technology | Version |
|:---|:---|:---|
| **Backend Framework** | Spring Boot | 3.4.8+ |
| **AI Framework** | Spring AI | 1.1.0 |
| **AI Extension** | Spring AI Alibaba | 1.1.0.0 |
| **Programming Language** | Java | 17+ |
| **Frontend Framework** | Vue 3 + TypeScript | - |
| **Build Tool** | Maven | - |
| **ORM Framework** | MyBatis | 3.0.4 |
| **Connection Pool** | Druid | 1.2.22 |
| **Database** | MySQL | 5.7+ |
| **Containerization** | Docker | - |
| **Chart Library** | ECharts | - |

### 3.3 Module Structure

```
DataAgent/
├── data-agent-management/          # Backend core module
│   └── src/main/java/com/alibaba/cloud/ai/dataagent/
│       ├── controller/             # REST API controllers
│       ├── service/                # Business service layer
│       │   ├── agent/              # Agent services
│       │   ├── graph/              # Graph execution services
│       │   ├── llm/                # LLM services
│       │   ├── mcp/                # MCP services
│       │   ├── semantic/           # Semantic model services
│       │   └── vectorstore/        # Vector store services
│       ├── workflow/               # Workflow engine
│       │   ├── node/               # Processing nodes (16)
│       │   └── dispatcher/         # Dispatchers (12)
│       ├── mapper/                 # MyBatis Mappers
│       ├── entity/                 # Data entities
│       ├── dto/                    # Data transfer objects
│       ├── vo/                     # View objects
│       ├── config/                 # Configuration classes
│       ├── strategy/               # Strategy pattern implementations
│       ├── splitter/               # Text splitters
│       └── util/                   # Utility classes
│
├── data-agent-frontend/            # Frontend module
│   └── src/
│       ├── components/             # Vue components
│       ├── views/                  # Page views
│       ├── services/               # API services
│       ├── router/                 # Router configuration
│       └── styles/                 # Style files
│
├── docs/                           # Documentation
├── docker-file/                    # Docker configuration
└── CI/                             # CI/CD configuration
```

---

## 4. StateGraph Workflow Engine

### 4.1 Node Overview

The system uses a StateGraph state-machine driven workflow engine with **16 core processing nodes**:

| # | Node Name | Type | Description |
|:---:|:---|:---|:---|
| 1 | `IntentRecognitionNode` | Input | Recognize user intent, determine if data analysis is needed |
| 2 | `EvidenceRecallNode` | Retrieval | Recall relevant business knowledge from knowledge base |
| 3 | `QueryEnhanceNode` | Retrieval | Rewrite and optimize user query |
| 4 | `SchemaRecallNode` | Retrieval | Retrieve relevant table and field schemas |
| 5 | `TableRelationNode` | Retrieval | Analyze table relationships |
| 6 | `FeasibilityAssessmentNode` | Planning | Assess feasibility of analysis request |
| 7 | `PlannerNode` | Planning | Generate analysis execution plan |
| 8 | `PlanExecutorNode` | Planning | Orchestrate and execute analysis steps |
| 9 | `HumanFeedbackNode` | Feedback | Wait for user confirmation or adjustment |
| 10 | `SqlGenerateNode` | SQL Exec | Generate SQL statements |
| 11 | `SemanticConsistencyNode` | SQL Exec | Verify SQL semantic consistency |
| 12 | `SqlExecuteNode` | SQL Exec | Execute SQL and get results |
| 13 | `PythonGenerateNode` | Python Exec | Generate Python code for analysis |
| 14 | `PythonExecuteNode` | Python Exec | Execute code in isolated environment |
| 15 | `PythonAnalyzeNode` | Python Exec | Process Python execution results |
| 16 | `ReportGeneratorNode` | Output | Generate visualization report |

### 4.2 Dispatchers

Each key node has a dispatcher responsible for flow control and error handling:

| Dispatcher | Associated Node | Responsibility |
|:---|:---|:---|
| `IntentRecognitionDispatcher` | IntentRecognitionNode | Intent routing decisions |
| `SchemaRecallDispatcher` | SchemaRecallNode | Schema recall control |
| `QueryEnhanceDispatcher` | QueryEnhanceNode | Query enhancement control |
| `TableRelationDispatcher` | TableRelationNode | Table relation analysis retry |
| `FeasibilityAssessmentDispatcher` | FeasibilityAssessmentNode | Feasibility judgment |
| `PlanExecutorDispatcher` | PlanExecutorNode | Step selection and orchestration |
| `HumanFeedbackDispatcher` | HumanFeedbackNode | Feedback processing routing |
| `SqlGenerateDispatcher` | SqlGenerateNode | SQL generation control |
| `SemanticConsistenceDispatcher` | SemanticConsistencyNode | Semantic validation control |
| `SQLExecutorDispatcher` | SqlExecuteNode | SQL execution and retry |
| `PythonExecutorDispatcher` | PythonExecuteNode | Python execution control |

### 4.3 Execution Flow Diagram

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
              Need Analysis              No Need
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
                   │              │Retry
           ┌───────┴───────┐      │
           │               │      │
        Success          Failed───┘
           │
           ▼
   ┌───────────────────┐
   │ FeasibilityAssess │
   └─────────┬─────────┘
             │
     ┌───────┴───────┐
     │               │
  Feasible      Not Feasible
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
      │       │Invalid/Rejected
      ▼       │
 ┌──────────┐ │
 │ Human    │─┤
 │ Feedback │ │
 └────┬─────┘ │
      │       │
  Approve     │
      ▼       │
┌───────────────────────────────────────────┐
│           Step Selection Loop              │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │         SQL Step Branch              │  │
│  │  SqlGenerate → SemanticCheck        │  │
│  │       ↓                             │  │
│  │  SqlExecute ← Retry(Failed)         │  │
│  │       ↓                             │  │
│  │  Store SQL Result                   │  │
│  └─────────────────────────────────────┘  │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │       Python Step Branch             │  │
│  │  PythonGenerate → PythonExecute     │  │
│  │       ↓            ↑                │  │
│  │  PythonAnalyze  ←Retry(Failed)      │  │
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

## 5. Frontend Architecture Design

### 5.1 Directory Structure

```
data-agent-frontend/src/
├── App.vue                         # Root component
├── main.js                         # Entry point
│
├── components/                     # Component directory
│   ├── agent/                      # Agent configuration components
│   │   ├── AccessApi.vue           # API Key management UI
│   │   ├── AgentKnowledgeConfig.vue    # Agent knowledge config
│   │   ├── BaseSetting.vue         # Basic settings
│   │   ├── BatchImportDialog.vue   # Batch import dialog
│   │   ├── BusinessKnowledgeConfig.vue # Business knowledge config
│   │   ├── DataSourceConfig.vue    # Data source config
│   │   ├── PresetsConfig.vue       # Preset questions config
│   │   ├── PromptConfig.vue        # Prompt configuration
│   │   └── SemanticsConfig.vue     # Semantic model config
│   │
│   └── run/                        # Runtime components
│       ├── ChartComponent.vue      # Generic chart component
│       ├── ChatSessionSidebar.vue  # Session sidebar
│       ├── HumanFeedback.vue       # Human feedback component
│       ├── PresetQuestions.vue     # Preset questions display
│       ├── ResultSetDisplay.vue    # Result set table display
│       │
│       ├── charts/                 # Chart implementations
│       │   ├── BaseChart.ts        # Chart base class
│       │   ├── ChartFactory.ts     # Chart factory
│       │   ├── BarChart.ts         # Bar chart
│       │   ├── LineChart.ts        # Line chart
│       │   └── PieChart.ts         # Pie chart
│       │
│       └── markdown/               # Markdown rendering
│           ├── index.ts            # Export entry
│           ├── markdown-plugin-echarts.ts  # ECharts plugin
│           ├── markdown-plugin-highlight.ts # Code highlighting
│           └── MarkdownAgentContainer.vue  # Container component
│
├── views/                          # Page views
│   ├── AgentCreate.vue             # Create agent
│   ├── AgentDetail.vue             # Agent detail/edit
│   ├── AgentList.vue               # Agent list
│   ├── AgentRun.vue                # Agent run interface
│   ├── ModelConfig.vue             # Model configuration
│   └── NotFound.vue                # 404 page
│
├── services/                       # API service layer
│   ├── agent.ts                    # Agent CRUD
│   ├── agentDatasource.ts          # Agent data sources
│   ├── agentKnowledge.ts           # Agent knowledge
│   ├── businessKnowledge.ts        # Business knowledge
│   ├── chat.ts                     # Chat/Session
│   ├── common.ts                   # Common utilities
│   ├── datasource.ts               # Data source management
│   ├── fileUpload.ts               # File upload
│   ├── graph.ts                    # Graph execution (SSE)
│   ├── logicalRelation.ts          # Logical foreign keys
│   ├── modelConfig.ts              # Model configuration
│   ├── presetQuestion.ts           # Preset questions
│   ├── resultSet.ts                # Result sets
│   ├── semanticModel.ts            # Semantic models
│   └── sessionStateManager.ts      # Session state management
│
├── router/                         # Router configuration
│   ├── index.js                    # Router instance
│   └── routes.js                   # Route definitions
│
├── layouts/                        # Layout components
│   └── BaseLayout.vue              # Base layout
│
└── styles/                         # Global styles
    └── global.css
```

### 5.2 Core Page Functions

| Page | Route | Function |
|:---|:---|:---|
| AgentList | `/agents` | Agent list display, search, status management |
| AgentCreate | `/agents/create` | Create new agent |
| AgentDetail | `/agents/:id` | Agent detail editing (multi-tab config) |
| AgentRun | `/agents/:id/run` | Agent conversation run interface |
| ModelConfig | `/models` | LLM/Embedding model configuration management |

### 5.3 SSE Streaming Communication

Frontend receives real-time streaming responses via Server-Sent Events (SSE):

```typescript
// services/graph.ts example
export function streamSearch(agentId: string, query: string) {
  const eventSource = new EventSource(
    `/api/graph/stream?agentId=${agentId}&query=${encodeURIComponent(query)}`
  );
  
  eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    // Handle different content types based on TextType
    switch (data.textType) {
      case 'SQL':
        // Render SQL code block
        break;
      case 'HTML':
        // Render HTML report
        break;
      case 'MARKDOWN':
        // Render Markdown
        break;
    }
  };
}
```

---

## 6. Data Model Design

### 6.1 Core Tables

#### agent
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

#### datasource
```sql
CREATE TABLE datasource (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL,
  type VARCHAR(20) NOT NULL,  -- mysql/postgresql/...
  host VARCHAR(255) NOT NULL,
  port INT NOT NULL,
  database_name VARCHAR(100) NOT NULL,
  username VARCHAR(100),
  password VARCHAR(255),  -- encrypted
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### agent_knowledge
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

#### business_knowledge
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

#### semantic_model
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

#### logical_relation
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

#### model_config
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

#### user_prompt_config
```sql
CREATE TABLE user_prompt_config (
  id INT PRIMARY KEY AUTO_INCREMENT,
  agent_id INT,  -- NULL for global config
  type VARCHAR(50) NOT NULL,  -- report-generator/planner/sql-generator/...
  content TEXT,
  priority INT DEFAULT 0,
  display_order INT DEFAULT 0,
  enabled BOOLEAN DEFAULT TRUE
);
```

### 6.2 Entity Relationship Diagram

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
│business_knowledge│ (Global knowledge, not tied to specific agent)
└─────────────────┘

┌─────────────────┐
│  model_config   │ (Global model configuration)
└─────────────────┘

┌───────────────────┐
│ user_prompt_config│ (Can be tied to agent_id or global)
└───────────────────┘
```

---

## 7. Key Technical Implementations

### 7.1 Streaming Output Mechanism

**Technical Solution**: Spring WebFlux + Server-Sent Events (SSE)

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

**Text Type Markers**:
```java
public enum TextType {
    TEXT,       // Plain text
    SQL,        // SQL code
    JSON,       // JSON data
    HTML,       // HTML content
    MARKDOWN,   // Markdown content
    ERROR       // Error message
}
```

### 7.2 Multi-turn Conversation Context

**Implementation**: `MultiTurnContextManager`

**Core Functions**:
- Maintain conversation history per session
- Inject history into subsequent request prompts
- Support configurable maximum history turns

```java
public class MultiTurnContextManager {
    
    private final Map<String, List<TurnRecord>> contextStore = new ConcurrentHashMap<>();
    
    public void beginTurn(String sessionId, String query) {
        // Begin new turn
    }
    
    public String buildContext(String sessionId) {
        // Build context with history
    }
    
    public void finishTurn(String sessionId, String result) {
        // Finish current turn, record result
    }
}
```

### 7.3 Python Code Execution

**Architecture Design**:

```
CodePoolExecutorService
        │
        ├── DockerCodeExecutor (Recommended)
        │       └── Uses docker-java library
        │       └── Image: continuumio/anaconda3:latest
        │
        ├── LocalCodeExecutor
        │       └── Local Python process
        │
        └── AiSimulationExecutor
                └── AI simulates execution results
```

**Configuration Example**:
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

### 7.4 RAG Retrieval Enhancement

**Retrieval Flow**:

```
User Question
    │
    ▼
┌───────────────┐
│ LLM Query     │ ← Combined with multi-turn context
│   Rewrite     │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ Build Filter  │ ← Filter by agent_id, type
│  Conditions   │
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
         Retrieved Documents
```

### 7.5 Logical Foreign Key Processing

**Background**: Production databases often lack physical foreign keys, causing LLMs to fail in inferring table relationships.

**Solution**:

1. User configures logical foreign keys in frontend
2. Schema recall automatically loads logical foreign keys
3. Filter foreign keys related to recalled tables
4. Merge physical and logical foreign keys
5. Provide complete schema for SQL generation

```java
private List<String> getLogicalForeignKeys(Integer agentId, List<Document> tableDocuments) {
    // 1. Get data source
    // 2. Extract recalled table names
    // 3. Query logical foreign keys
    // 4. Filter relevant foreign keys
    // 5. Format and return
    return formattedForeignKeys;
}
```

---

## 8. Deployment Architecture

### 8.1 Environment Requirements

| Component | Version | Notes |
|:---|:---|:---|
| JDK | 17+ | Required |
| MySQL | 5.7+ | Stores metadata |
| Node.js | 16+ | Frontend build |
| Docker | latest | Python execution environment (Optional) |

### 8.2 Deployment Methods

#### Method 1: Local Development

```bash
# 1. Import database
mysql -u root -p < data-agent-management/src/main/resources/sql/schema.sql

# 2. Start backend
cd data-agent-management
./mvnw spring-boot:run

# 3. Start frontend
cd data-agent-frontend
npm install && npm run dev

# 4. Access system
open http://localhost:3000
```

#### Method 2: Docker Compose

Refer to configuration files in `docker-file/` directory.

### 8.3 Configuration Parameters

```yaml
spring:
  ai:
    alibaba:
      data-agent:
        # LLM service type
        llm-service-type: STREAM  # STREAM or BLOCK
        
        # Multi-turn conversation config
        multi-turn:
          enabled: true
          max-history: 10
          context-window: 4096
        
        # Vector store config
        vector-store:
          enable-hybrid-search: true
          similarity-threshold: 0.7
          top-k: 10
        
        # Code executor config
        code-executor:
          type: docker
          docker:
            image: continuumio/anaconda3:latest
            timeout: 300000
        
        # Plan executor config
        plan-executor:
          max-retry: 3
          timeout: 600000
```

---

## Appendix

### A. Related Documents

| Document | Description |
|:---|:---|
| [Quick Start](QUICK_START-en.md) | Installation and configuration guide |
| [Architecture Design](ARCHITECTURE-en.md) | Detailed system architecture |
| [Developer Guide](DEVELOPER_GUIDE-en.md) | Contribution guide and code standards |
| [Advanced Features](ADVANCED_FEATURES-en.md) | API invocation and MCP configuration |
| [Knowledge Configuration Best Practices](KNOWLEDGE_USAGE-en.md) | Knowledge base configuration guide |

### B. Version Information

- **Document Version**: 1.0
- **Last Updated**: 2026-01-27
- **Applicable Version**: DataAgent 1.0.0-SNAPSHOT

---

<div align="center">
    Made with ❤️ by Spring AI Alibaba DataAgent Team
</div>
