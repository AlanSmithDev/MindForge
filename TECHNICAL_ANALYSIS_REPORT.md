# Super Agent Master - 技术分析报告

> 扫描时间: 2026-06-25
> 扫描范围: 完整项目源码、配置文件、SQL建表脚本

---

## 一、项目整体结构

### 1.1 模块划分

```
super-agent-master/
├── pom.xml                           # 根POM（版本管理、模块聚合）
├── ai-example/                       # 示例模块
├── super-agent-business/             # 业务模块（核心）
│   └── super-agent-business-chat/    # 聊天业务核心实现
├── super-agent-common/               # 公共模块
├── super-agent-id-generator-framework/ # 分布式ID生成器
├── super-agent-redisson-framework/   # Redisson分布式锁框架
└── sql/                              # 数据库建表脚本
    ├── Mysql/                        # MySQL建表脚本
    └── PostgresSql/                  # PostgreSQL建表脚本（PGVector）
```

### 1.2 核心包结构

```
org.javaup.ai
├── chatagent/                        # 聊天Agent核心包
│   ├── config/                       # Agent配置（ChatAgentConfiguration等）
│   ├── controller/                   # SSE聊天控制器
│   ├── data/                         # 数据实体（Dialogue, Exchange, MemorySummary等）
│   ├── dto/                          # 请求DTO
│   ├── mapper/                       # MyBatis-Plus Mapper
│   ├── model/                        # 业务模型（SearchReference, ExecutionMode等）
│   │   ├── es/                       # ES索引模型
│   │   ├── graph/                    # 图数据模型
│   │   ├── route/                    # 路由模型
│   │   └── trace/                    # 追踪模型
│   ├── rag/                          # RAG检索引擎核心
│   │   ├── channel/                  # 检索通道（Vector/Keyword）
│   │   ├── executor/                 # 执行器（6种模式）
│   │   ├── model/                    # RAG模型
│   │   └── service/                  # 编排服务
│   ├── service/                      # 业务服务（BusinessChatService等）
│   ├── support/                      # 支撑组件（StreamEventWriter等）
│   ├── tool/                         # Agent工具（TavilySearchTool等）
│   └── vo/                           # 视图对象
├── manage/                           # 文档管理核心包
│   ├── config/                       # 管理配置
│   ├── controller/                   # 管理控制器
│   ├── data/                         # 文档相关实体
│   ├── dto/                          # 请求DTO
│   ├── mapper/                       # MyBatis-Plus Mapper
│   ├── model/                        # 业务模型
│   ├── mq/                           # Kafka消息（Producer/Consumer/Message）
│   ├── service/                      # 服务接口+实现
│   │   ├── impl/                     # 服务实现
│   │   └── keyword/                  # 关键词检索网关
│   ├── support/                      # 支撑组件（文档结构解析等）
│   └── vo/                           # 视图对象
├── auth/                             # 管理端认证
└── prompt/                           # Prompt模板管理
```

---

## 二、技术栈版本

### 2.1 核心框架

| 技术 | 版本 | 说明 |
|------|------|------|
| Java | 17 | 运行环境 |
| Spring Boot | 3.5.6 | 核心框架 |
| Spring AI | 1.1.0 (BOM) | AI框架 |
| Spring AI Alibaba | 1.1.2.0 (BOM) | 阿里云AI扩展 |
| MyBatis-Plus | 3.5.7 | ORM框架 |
| MyBatis Spring Boot | 3.0.5 | MyBatis集成 |

### 2.2 Spring AI Alibaba 关键依赖

| 依赖 | 说明 |
|------|------|
| `spring-ai-alibaba-agent-framework` | Agent框架（ReactAgent、Checkpoint、Tool） |
| `spring-ai-starter-model-openai` | OpenAI兼容协议模型（DashScope代理） |
| `spring-ai-rag` | RAG支持 |

### 2.3 中间件与存储

| 中间件 | 版本 | 用途 |
|--------|------|------|
| Redisson | 3.32.0 | 分布式锁/Redis客户端 |
| MinIO | 8.5.17 | 对象存储（文档存储） |
| Apache Tika | 3.2.3 | 文档解析 |
| Elasticsearch (Spring Data) | Spring Boot管理 | 关键词检索/导航索引/路由索引 |
| Neo4j Java Driver | 项目内管理 | 文档结构图谱 |
| PostgreSQL JDBC | Spring Boot管理 | PGVector向量存储 |
| Spring Kafka | Spring Boot管理 | 异步消息 |
| Hutool | 5.8.25 | 工具库 |
| FastJSON | 项目内管理 | JSON序列化 |
| Baidu UidGenerator | 项目内管理 | 分布式ID生成 |

### 2.4 模型服务

| 模型 | 提供商 | 用途 |
|------|--------|------|
| qwen-plus-latest | DashScope (阿里云) | 聊天/Agent推理 |
| text-embedding-v4 | DashScope (阿里云) | 文本向量化 |
| BGE-reranker-v2-m3 | SiliconFlow | 检索重排序（可选） |

---

## 三、核心业务架构总览

### 3.1 整体请求处理流程

```
用户请求 → BusinessChatController.stream()
  → BusinessChatService.openConversationStream()
    → buildLaunchPlan() → 解析请求参数
    → claimConversationLease() → Redis分布式锁
    → bootstrapConversation() → 创建Exchange记录
    → bindClientChannel() → SSE绑定
      → activateGeneration() → 异步启动生成
        → buildConversationExecution()
          → prepareExecutionPlan() → ChatPreparationOrchestrator 5步决策
          → ConversationExecutor.execute() → 6种执行模式
            → SSE流式输出
```

### 3.2 ChatPreparationOrchestrator 五步决策链

```
Step 1: summarizeHistory → 装载会话记忆（滑动窗口+摘要压缩）
Step 2: buildPlanningHistory + buildAnswerHistoryContext → 构建历史上下文
Step 3: chatQueryRewriteService.rewrite() → 问题改写+子问题拆分
Step 4: knowledgeRouteService.route() → AUTO_DOCUMENT模式三层知识路由
Step 5: documentQuestionRouter.route() → 执行模式判定
```

---

## 四、6种ExecutionMode执行器详解

### 4.1 CLARIFICATION（澄清模式）
- **触发条件**: 知识路由置信度 < 0.55，或文档歧义
- **执行器**: `ClarificationExecutor`
- **输出**: 候选文档列表 + 澄清问题 + 推荐理由

### 4.2 RETRIEVAL（混合检索模式）
- **触发条件**: 普通文档问答问题
- **执行器**: `RagChatExecutor`
- **核心流程**:
  1. `RagRetrievalEngine` 多子问题并行检索
  2. 双通道（Vector + Keyword）并行召回
  3. 证据闸门过滤
  4. RRF融合排序
  5. 父块提升
  6. PromptBudget预算控制
  7. LLM流式生成

### 4.3 GRAPH_ONLY（纯图模式）
- **触发条件**: 章节查询/目录展开类问题
- **执行器**: `GraphOnlyExecutor`
- **操作**: SECTION_ADJACENCY_LOOKUP / CHILD_SECTION_DESCEND

### 4.4 GRAPH_THEN_EVIDENCE（先图后证据模式）
- **触发条件**: 编号项/步骤查询
- **执行器**: `GraphThenEvidenceExecutor`
- **流程**: 图定位 → 证据提取 → 结构化答案

### 4.5 REACT_AGENT（Agent模式）
- **触发条件**: OPEN_CHAT模式，或闲聊/开放式问题
- **执行器**: `ReactAgentExecutor`
- **工具**: Tavily联网搜索
- **输出**: 流式文本 + 引用来源

### 4.6 CONVERSATION（对话模式）
- 已注册但尚未深度实现

---

## 五、双通道混合检索体系

### 5.1 VectorRetrievalChannel（向量检索通道）

- **存储**: PostgreSQL + PGVector扩展
- **向量模型**: text-embedding-v4 (DashScope)
- **检索方式**: PGVector `<=>` 余弦距离算子
- **SQL模板**: `1 - (embedding <=> CAST(? AS vector)) AS similarity_score`
- **TopK**: 8 (配置项 `vector-top-k`)
- **过滤**: 支持 documentId/taskId/sectionPath/structureNodeId/canonicalPath/itemIndex 多维度过滤

### 5.2 KeywordRetrievalChannel（关键词检索通道）

- **存储**: Elasticsearch（主）+ PostgreSQL LIKE fallback（备）
- **ES查询策略**:
  - matchPhrase: sectionPath(boost=8), chunkText(boost=5), documentName(boost=4)
  - multiMatch(BestFields): sectionPath^6, documentName^4, knowledgeScopeName^3, chunkText
  - 支持业务分类、文档标签、文档名称、章节路径、上下文提示等多维度加权
- **PG fallback关键词提取**:
  - 英文: 正则提取 `[a-z0-9._-]{2,}`
  - 中文: 正则提取 `[\p{IsHan}]{2,}` + 去噪词 + 分词 + N-gram滑动窗口
  - 最多8个关键词项，权重递减（6→1），章节路径额外+2权重
- **TopK**: 8 (配置项 `keyword-top-k`)

### 5.3 RRF融合排序（RagRetrievalEngine）

```
RRF分数 = 1 / (RRF_K + rank + 1)
其中 RRF_K = 60
```

- 双通道结果按各自排名计算RRF分数
- 按RRF分数降序排列
- 去重（同一chunk_id合并分数）

### 5.4 证据闸门（Evidence Gate）

| 通道 | 闸门规则 | 配置项 |
|------|----------|--------|
| Vector | minVectorSimilarity = 0.45 | `vector-min-similarity` |
| Keyword | 相对分数floor = 0.35 | `keyword-relative-score-floor` |

- 向量通道: 相似度低于阈值直接丢弃
- 关键词通道: 分数低于(最高分 × floor)的丢弃

### 5.5 父块提升（Parent Block Elevation）

- 命中子块(chunk)后，查询对应的父块(SuperAgentDocumentParentBlock)
- 父块分数聚合: `bestChildScore × (1 + supportWeight + multiChannelWeight)`
  - supportWeight: min(0.36, supportCount × 0.12)，支持子块越多越加分
  - multiChannelWeight: 双通道命中 +0.10
- 父块内容 = 完整父块文本 + 命中子片段摘要

### 5.6 重排序（Rerank）

- 可选启用，使用外部Rerank服务
- 模型: SiliconFlow BGE-reranker-v2-m3
- Final TopK截断: 5 (配置项 `final-top-k`)

---

## 六、PromptBudget 证据预算机制

### 6.1 双层预算

| 预算层级 | 默认值 | 配置项 |
|----------|--------|--------|
| 总预算 | 5200字符 | `total-evidence-max-chars` |
| 单子问题预算 | 2200字符 | `per-sub-question-evidence-max-chars` |

### 6.2 消耗逻辑

- 每条证据同时检查总预算和子问题预算
- 超出任一预算的证据记录到 `omittedReferenceDetails`
- 引用去重: 通过uniqueKey避免同一证据重复注入

---

## 七、DocumentQuestionRouter 文档路由

### 7.1 三种执行模式判定

| 模式 | 触发条件 | 检测方式 |
|------|----------|----------|
| GRAPH_ONLY | 相邻章节查询(SECTION_ADJACENCY_LOOKUP)、目录展开(CHILD_SECTION_DESCEND) | 正则+关键词本地规则 |
| GRAPH_THEN_EVIDENCE | 编号项查询(ITEM_REFERENCE) | 正则+关键词本地规则 |
| RETRIEVAL | 普通文档问题 | 默认fallback |

### 7.2 意图检测

- **本地规则**: 正则匹配 + 关键词集（ADJACENCY_HINTS、OUTLINE_HINTS、ITEM_HINTS等）
- **LLM兜底**: 本地规则不确定时调用LLM进行JSON分类输出
- **置信度阈值**: GRAPH_ONLY_INTENT_CONFIDENCE_THRESHOLD = 0.75

### 7.3 章节解析

按优先级四级fallback:
1. sectionCode 精确匹配
2. NavigationIndex（ES导航索引）
3. LocalStructure（MySQL结构节点表）
4. GraphService（Neo4j图查询）

---

## 八、KnowledgeRouteService 三层知识路由

### 8.1 三层路由

```
Scope（范围）→ Topic（主题）→ Document（文档）
```

### 8.2 评分公式

```
总分 = semanticMainScore × 50    （向量语义相似度 - 0.2阈值）
     + lexicalAssist              （ES词法分数 × 1.6）
     + keywordEntityAssist        （实体词命中 +6/个）
     + scopeBoost                 （匹配TopScope +15）
     + topicRelationBoost         （主题-文档关系分 × 20）
```

### 8.3 置信度计算

```
confidence = topScore / (topScore + secondScore + 5)
```

### 8.4 路由状态

| 状态 | 条件 | 候选数 |
|------|------|--------|
| SUCCESS | confidence >= 0.55 | 3个 |
| LOW_CONFIDENCE | confidence < 0.55 | 5个 |
| FAILED | 无候选 | 0个 |

### 8.5 影子路由

- DOCUMENT模式（用户指定文档）下异步记录路由结果
- 用于评估自动路由准确性

---

## 九、会话记忆管理

### 9.1 滑动窗口 + 摘要压缩

| 参数 | 默认值 | 说明 |
|------|--------|------|
| keepRecentTurns | 4 | 保留最近4轮原文 |
| compressionBatchTurns | 6 | 每6轮触发一次压缩 |
| recentTranscriptMaxChars | 2200 | 最近窗口最大字符数 |
| summaryMaxChars | 1400 | 摘要最大字符数 |

### 9.2 ConversationSummaryPayload 摘要结构

```
{
  "summary": "长期摘要文本",
  "conversationGoal": "对话目标",
  "stableFacts": "稳定事实",
  "userPreferences": "用户偏好",
  "resolvedPoints": "已解决要点",
  "pendingQuestions": "待解决问题",
  "retrievalHints": "检索提示"
}
```

### 9.3 增量压缩

- LLM调用 `mergeSummaryPayload` 合并新批次到现有摘要
- 失败时fallback到规则压缩

---

## 十、文档管理全链路

### 10.1 文档处理流程

```
文件上传 → MinIO存储
    ↓
Kafka消息（parse-route topic）
    ↓
Tika解析 → 文本清洗 → 结构提取 → 节点入库
    ↓
策略推荐 → 策略确认 → 切块执行
    ↓
Kafka消息（index-build topic）
    ↓
向量化（PGVector） + ES索引 + Neo4j图投影
    ↓
索引构建完成
```

### 10.2 各阶段详解

#### 阶段1: 文件上传
- 上传到MinIO对象存储
- 路径: `{prefix}/{documentId}/{timestamp}-{originalFileName}`
- 创建文档记录（parse_status=待解析）

#### 阶段2: 内容解析
- Kafka消费者触发 `DocumentAsyncProcessService.handleParseRoute()`
- 使用Apache Tika提取文本（支持PDF/DOC/DOCX/TXT/MD/HTML）
- 文本清洗: 统一换行符、去除控制字符、压缩空行
- 结构提取: `DocumentStructureNodeExtractor` 提取章节/步骤/列表项
- 解析后文本上传回MinIO
- 结构节点写入MySQL（super_agent_document_structure_node）
- 同步导航ES索引 + Neo4j图投影

#### 阶段3: 策略推荐
- 根据文档结构化程度和质量推荐分块策略
- 策略类型: 基于文档结构切块、递归分块、语义分块、LLM智能切块
- 策略角色: 主策略、优化策略、兜底策略、增强策略
- 流水线类型: PARENT（父块）和 CHILD（子块）两条流水线

#### 阶段4: 索引构建
- Kafka消费者触发 `DocumentAsyncProcessService.handleIndexBuild()`
- 按策略执行切块流水线
- 构建 ParentBlock + Chunk 实体
- 向量化: `DefaultDocumentVectorGateway` 批量embedding（每批10条）
- ES索引: `ElasticsearchDocumentKeywordSearchGateway` 批量写入
- Neo4j图投影: `Neo4jDocumentStructureGraphProjectionService`

---

## 十一、底层存储网关详解

### 11.1 DefaultDocumentVectorGateway（PGVector向量网关）

- **功能**: 文档切块向量化+存储
- **批处理**: 每批10条chunk（`EMBEDDING_BATCH_SIZE_LIMIT = 10`）
- **SQL**: INSERT ... ON CONFLICT (id) DO UPDATE（Upsert语义）
- **向量格式**: PGVector数组字面量 `[0.1,0.2,...]`
- **元数据**: JSON格式存储 documentId/taskId/sectionPath/structureNodeId等
- **状态更新**: 向量化完成后标记 `vectorStatus = VECTOR_SUCCESS`

### 11.2 ElasticsearchDocumentKeywordSearchGateway（ES关键词网关）

- **功能**: 文档切块ES索引+关键词检索
- **索引**: 单索引存储，chunkId作为文档ID
- **索引字段**: chunkId, documentId, taskId, parentBlockId, chunkNo, documentName, sectionPath, structureNodeId, canonicalPath, itemIndex, knowledgeScopeCode/Name, businessCategory, documentTags, chunkText
- **检索**: Bool Query组合（filter精确匹配 + should加权匹配）
- **批量写入**: Bulk API, Refresh.WaitFor

### 11.3 Neo4jDocumentStructureGraphService（Neo4j图服务）

- **图模型**: `(Document)-[:HAS_SECTION]->(Section)-[:HAS_CHILD]->(Section)`
- **关系**: NEXT_SIBLING（兄弟关系）、HAS_ITEM（条目关系）
- **核心查询**:
  - `findSectionById`: 按节点ID查询
  - `findSectionByCode`: 按章节编码查询（如"1.2"）
  - `findSectionByTitle`: 按标题查询（normalized匹配）
  - `findSectionByCanonicalPath`: 按稳定路径查询
  - `findBestSection`: 按topic+facet综合打分匹配
  - `listChildren`: 查询子节点
  - `previousSibling/nextSibling`: 兄弟节点查询
  - `findItemByIndex`: 按条目序号查询
- **条件启用**: `@ConditionalOnBean(name = "documentManageNeo4jDriver")`

### 11.4 DocumentKnowledgeServiceImpl（知识检索服务）

- **向量检索**: PGVector原生SQL `1 - (embedding <=> CAST(? AS vector))`
- **关键词检索**: 优先ES网关，fallback到PG LIKE查询
- **父块提升**: `elevateToParentBlocks()` 方法
- **关键词提取**: 英文+中文混合提取，N-gram滑动窗口，去噪词过滤
- **PG fallback评分**: `CASE WHEN LOWER(chunk_text) LIKE ? THEN weight ELSE 0 END`

---

## 十二、Tavily联网搜索工具

### 12.1 配置

| 参数 | 说明 |
|------|------|
| API端点 | POST /search |
| 认证方式 | Bearer Token |
| 默认topic | general |
| 搜索深度 | advanced |
| 最大结果数 | 5 |
| 超时 | 30秒 |

### 12.2 时间敏感查询

- `TimeSensitiveQueryHelper.buildEffectiveSearchQuery()` 注入当前日期
- 确保搜索结果时效性

### 12.3 SSE状态推送

- 搜索过程中通过 `SinkEmitHelper` 推送"正在联网搜索"等状态信息

### 12.4 Agent限制

| 限制类型 | 配置 |
|----------|------|
| ModelCallLimitHook | runLimit=8, threadLimit=40 |
| ToolCallLimitHook (tavily_search) | runLimit=6, threadLimit=30 |
| ToolRetryInterceptor | maxRetries=2 |
| 并行工具执行 | maxParallelTools=4 |

---

## 十三、ReactAgent配置

### 13.1 ChatAgentConfiguration

- **Checkpoint持久化**: `MysqlSaver`（MySQL存储）
- **Checkpoint创建策略**: `CREATE_IF_NOT_EXISTS`
- **工具注册**: `TavilySearchTool` 作为 `FunctionToolCallback`
- **Hook系统**:
  - `ModelCallLimitHook`: 限制模型调用次数
  - `ToolCallLimitHook`: 限制工具调用次数
  - `ToolRetryInterceptor`: 工具调用重试
- **系统Prompt**: 从 `PromptTemplateService` 动态加载

### 13.2 ChatCheckpointManager

- **存储**: MySQL（GRAPH_THREAD + GRAPH_CHECKPOINT 表）
- **操作**: get/list/clearThread
- **清除会话**: 按threadName级联删除checkpoint和thread

---

## 十四、SSE流式响应实现

### 14.1 架构

```
BusinessChatController
  ↓ Flux<String> (text/event-stream)
BusinessChatService
  ↓ Sinks.Many<String> (unicast, backpressure buffer)
StreamEventWriter → JSON事件序列化
SinkEmitHelper → 线程安全发射
```

### 14.2 事件类型

| 类型 | 说明 | 数据结构 |
|------|------|----------|
| text | 回答文本片段 | {type, content, timestamp, conversationId, exchangeId} |
| thinking | 思考过程 | 同上 |
| status | 状态更新 | 同上 |
| error | 错误信息 | 同上 |
| reference | 引用来源 | {type, content: [...], count, timestamp} |
| recommend | 推荐追问 | {type, content: [...], count, timestamp} |

### 14.3 关键实现

- `Sinks.many().unicast().onBackpressureBuffer()`: 创建单播Sink
- `SinkEmitHelper.emitNext()`: 同步发射，处理CANCELLED/TERMINATED/ZERO_SUBSCRIBER
- `StreamEventWriter`: JSON序列化，统一事件格式
- `StreamEventMetadata`: 携带conversationId和exchangeId

---

## 十五、Kafka消息机制

### 15.1 主题配置

| 主题 | 配置项 | 用途 |
|------|--------|------|
| parse-topic | super-agent-document-parse-route | 文档解析路由 |
| index-topic | super-agent-document-index-build | 索引构建 |

### 15.2 消息流程

```
上传文档 → DocumentKafkaProducer.sendParseRoute()
  → Kafka parse-topic
  → DocumentKafkaConsumer.consumeParseRoute()
  → DocumentAsyncProcessService.handleParseRoute()

策略确认 → DocumentKafkaProducer.sendIndexBuild()
  → Kafka index-topic
  → DocumentKafkaConsumer.consumeIndexBuild()
  → DocumentAsyncProcessService.handleIndexBuild()
```

### 15.3 消息体

- `DocumentParseRouteMessage`: {documentId, taskId}
- `DocumentIndexBuildMessage`: {documentId, taskId, planId}

---

## 十六、MinIO对象存储

### 16.1 存储策略

| 内容类型 | 路径格式 | 说明 |
|----------|----------|------|
| 原始文件 | `{prefix}/{documentId}/{timestamp}-{originalFileName}` | 用户上传的原文件 |
| 解析文本 | `{parsedTextPrefix}/{documentId}.txt` | Tika解析后的纯文本 |

### 16.2 操作

- **上传**: 自动确保Bucket存在
- **下载**: 按objectName下载字节
- **删除**: 批量按objectName列表删除
- **URL构建**: `{endpoint}/{bucketName}/{objectName}`

---

## 十七、数据库设计

### 17.1 MySQL 核心表（15张）

#### 对话相关

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| super_agent_chat_dialogue | 对话归档主表 | dialogue_code, dialogue_stage, chat_mode, selected_document_id |
| super_agent_chat_exchange | 对话轮次归档表 | dialogue_code, user_prompt, reply_content, reasoning_note_list(JSON), source_snapshot_list(JSON), followup_suggestion_list(JSON), tool_trace_list(JSON), exchange_state, first_token_latency_ms, total_latency_ms |
| super_agent_chat_memory_summary | 长期记忆摘要表 | dialogue_code(UNIQUE), covered_exchange_id, compression_count, summary_text, summary_json(JSON) |
| super_agent_chat_exchange_trace_stage | 执行阶段轨迹表 | exchange_id, trace_id, stage_code, stage_order, stage_level, execution_mode, duration_ms, snapshot_json(JSON) |
| super_agent_chat_retrieval_result | 检索结果快照表 | exchange_id, trace_id, sub_question_index, channel_type, original_score, rrf_score, rerank_score, gate_passed, is_elevated, is_selected |
| super_agent_chat_channel_execution | 通道执行详情表 | exchange_id, channel_type, execution_state, duration_ms, recalled_count, accepted_count, final_selected_count, avg_score |
| super_agent_chat_stage_benchmark | 阶段性能基准表 | stage_code, execution_mode, p50/p90/p99_duration_ms, avg/max/min_duration_ms, sample_count |

#### 文档管理相关

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| super_agent_document | 文档表 | document_name, file_type, parse_status(1-4), strategy_status(1-4), index_status(1-4), knowledge_scope_code, structure_node_count |
| super_agent_document_strategy_plan | 策略方案表 | document_id, plan_version, plan_source, strategy_snapshot, recommend_reason |
| super_agent_document_strategy_step | 策略步骤表 | plan_id, step_no, pipeline_type(PARENT/CHILD), strategy_type(1-4), strategy_role(1-4), execute_status |
| super_agent_document_task | 文档任务表 | document_id, plan_id, task_type(1解析/2索引), task_status, current_stage(1-8), trigger_source, retry_count |
| super_agent_document_task_log | 任务日志表 | task_id, stage_type, event_type, log_level, operator_type, content, detail_json |
| super_agent_document_structure_node | 结构节点表 | document_id, parse_task_id, node_no, node_type(1-4), parent_node_id, prev/next_sibling_node_id, depth, node_code, title, canonical_path |
| super_agent_document_parent_block | 父块表 | document_id, task_id, parent_no, section_path, structure_node_id, canonical_path, item_index, parent_text, child_count, start/end_chunk_no |
| super_agent_document_chunk | 子块表 | document_id, task_id, parent_block_id, chunk_no, section_path, chunk_text, vector_status(1-4), vector_store_type, vector_id |

#### 知识路由相关

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| super_agent_knowledge_scope_node | 知识范围表 | scope_code(UNIQUE), scope_name, parent_scope_code, aliases, examples(JSON) |
| super_agent_knowledge_topic_node | 知识主题表 | topic_code(UNIQUE), scope_code, execution_preference, answer_shape |
| super_agent_document_profile | 文档画像表 | document_id(UNIQUE), document_type, core_topics(JSON), graph_friendly, supports_graph_outline |
| super_agent_topic_document_relation | 主题文档关联表 | topic_code+document_id(UNIQUE), relation_score, relation_source |
| super_agent_knowledge_route_trace | 路由追踪表 | conversation_id, exchange_id, mode(shadow/auto), top_scopes/topics/documents(JSON), confidence |

#### Agent状态持久化

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| GRAPH_THREAD | Graph线程表 | thread_id, thread_name(UNIQUE), is_released |
| GRAPH_CHECKPOINT | Checkpoint表 | checkpoint_id, thread_id, node_id, next_node_id, state_data(JSON) |

### 17.2 PostgreSQL 表（1张）

| 表名 | 说明 | 核心字段 |
|------|------|----------|
| super_agent_document_embedding | 文档向量表 | id(复用MySQL chunk ID), embedding(VECTOR类型), metadata_json(JSONB), embedding_model |

- 启用 `CREATE EXTENSION IF NOT EXISTS vector`
- 索引: document_id, task_id, plan_id, parent_block_id, status
- 备注: 当前使用未固定维度的VECTOR类型，后续可固定为VECTOR(1024/1536)并添加HNSW索引

---

## 十八、线程池配置

| 线程池 | 核心线程 | 最大线程 | 队列容量 | 用途 |
|--------|----------|----------|----------|------|
| chatRagExecutorService | 4 | 8 | 256 | RAG检索并行 |
| chatMemorySummaryExecutorService | 1 | 2 | 32 | 记忆摘要压缩 |
| chatPostProcessExecutorService | 1 | 2 | 64 | 后处理 |

---

## 十九、关键配置项汇总

### 19.1 RAG编排

```yaml
app.chat.rag:
  vector-top-k: 8
  keyword-top-k: 8
  final-top-k: 5
  vector-min-similarity: 0.45
  keyword-relative-score-floor: 0.35
  keyword-channel-enabled: true
  rrf-k: 60
  channel-timeout-ms: 5000
  sub-question-timeout-ms: 12000
  total-evidence-max-chars: 5200
  per-sub-question-evidence-max-chars: 2200
```

### 19.2 会话记忆

```yaml
app.chat.memory:
  keep-recent-turns: 4
  compression-batch-turns: 6
  recent-transcript-max-chars: 2200
  summary-max-chars: 1400
```

### 19.3 Tavily搜索

```yaml
app.tavily:
  api-key: ${TAVILY_API_KEY}
  base-url: https://api.tavily.com
  default-topic: general
  search-depth: advanced
  max-results: 5
  timeout-seconds: 30
```

### 19.4 模型配置

```yaml
spring.ai.openai:
  base-url: https://dashscope.aliyuncs.com/compatible-mode/v1
  api-key: ${DASHSCOPE_API_KEY}
  chat.options.model: qwen-plus-latest
  embedding.options.model: text-embedding-v4
```

### 19.5 文档管理

```yaml
app.manage:
  minio:
    endpoint: ${MINIO_ENDPOINT}
    bucket-name: super-agent-documents
    object-prefix: documents
    parsed-text-prefix: parsed-texts
  kafka:
    parse-topic: super-agent-document-parse-route
    index-topic: super-agent-document-index-build
    group-id: super-agent-document
  chunk:
    recursive-char-limit: 800
    recursive-overlap: 120
    semantic-min-chars: 700
    semantic-max-chars: 240
  pgvector:
    host: ${PGVECTOR_HOST}
    port: ${PGVECTOR_PORT}
    database: super_agent_pgvector
  elasticsearch:
    enabled: true
    index-name: super_agent_document_keyword
    navigation-index-name: super_agent_document_navigation
    route-index-name: super_agent_knowledge_route
  neo4j:
    uri: ${NEO4J_URI}
    database: neo4j
```

---

## 二十、架构亮点总结

1. **5步决策链**: ChatPreparationOrchestrator 将复杂的对话决策分解为5个有序步骤，职责清晰
2. **双通道混合检索**: Vector语义 + Keyword词法互补，PGVector + Elasticsearch双引擎
3. **RRF融合排序**: 无需分数归一化即可融合异构排序结果
4. **证据闸门**: 双层过滤机制确保检索质量
5. **父子分块**: 检索命中子块后提升到父块，平衡检索精度与上下文完整性
6. **PromptBudget**: 双层字符预算精确控制Prompt证据注入量
7. **三层知识路由**: Scope→Topic→Document递进式路由，语义+词法混合打分
8. **滑动窗口+摘要压缩**: 平衡对话记忆的完整性与Token消耗
9. **Kafka异步文档处理**: 解耦上传→解析→索引全链路
10. **全链路可观测**: Exchange→TraceStage→RetrievalResult→ChannelExecution四级追踪
11. **MySQL Checkpoint持久化**: ReactAgent状态可恢复
12. **SSE流式响应**: Reactor Flux + Sinks.Many 实现高效服务端推送
13. **多存储协同**: MySQL(业务数据) + PGVector(向量) + ES(关键词+导航+路由) + Neo4j(结构图) + MinIO(文件) + Redis(分布式锁)
