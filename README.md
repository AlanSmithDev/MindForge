# Super Agent — 企业级 AI Agent 框架

> 基于 Spring AI Alibaba Agent Framework 构建的企业级 AI Agent 解决方案，支持 RAG 知识问答、对话管理、记忆系统、工具调用等完整 Agent 能力。

---

## 📋 项目简介

**Super Agent** 是一个企业级的 AI Agent 框架项目，演示了如何使用 **Spring AI Alibaba Agent Framework** 构建具备完整 Agent 能力的业务系统。项目包含：

- 🤖 **对话业务模块** — 支持知识问答、任务执行、多轮对话的 Agent 服务
- 📚 **知识库管理** — 基于 RAG（检索增强生成）的知识文档检索与问答
- 🧠 **记忆系统** — 会话级与长期记忆管理，支持上下文恢复与断点续聊
- 🔧 **工具调用** — 支持 MCP（Model Context Protocol）工具集成与调用
- 🎯 **示例模块** — 包含 Spring AI Alibaba、RAG、MCP、Memory 等多个示例

---

## 🏗️ 项目架构

```
super-agent-master/
├── ai-example/                          # 示例模块
│   ├── ai-example-spring-ai-alibaba/   # Spring AI Alibaba 基础示例
│   ├── ai-example-rag/                  # RAG 知识问答示例
│   ├── ai-example-mcp/                  # MCP 工具调用示例
│   └── ai-example-memory/               # Memory 记忆系统示例
├── super-agent-business/                # 业务模块
│   └── super-agent-business-chat/       # 对话业务核心模块
├── super-agent-common/                  # 公共模块
├── super-agent-id-generator-framework/ # ID 生成框架（雪花算法）
├── super-agent-redisson-framework/     # Redis 分布式锁框架
├── vue/                                 # Vue3 前端（对话 UI）
└── 需要的例子演示/                       # 示例文档（产品手册、管理制度等）
```

---

## 🔧 技术栈

### 后端

| 技术 | 版本 | 说明 |
|------|------|------|
| **Spring Boot** | 3.x | 基础框架 |
| **Spring AI Alibaba** | latest | 阿里云 AI Agent 框架 |
| **Spring AI OpenAI** | latest | OpenAI 协议兼容（硅基流动） |
| **React Agent** | - | ReAct 推理 Agent |
| **MyBatis-Plus** | 3.x | ORM 框架 |
| **Redisson** | - | Redis 分布式锁 |

### 中间件

| 组件 | 用途 |
|------|------|
| **Redis** | 会话缓存、分布式锁、消息队列 |
| **Kafka** | 事件驱动、异步消息 |
| **MinIO** | 对象存储（文档、附件） |
| **Elasticsearch** | 全文检索、RAG 知识库 |
| **Neo4j** | 知识图谱、关系推理 |
| **Apache Tika** | 文档解析（PDF、Word 等） |

### 前端

| 技术 | 版本 | 说明 |
|------|------|------|
| **Vue** | 3.4+ | 渐进式框架 |
| **Vue Router** | 4.x | 路由管理 |
| **Vite** | 6.x | 构建工具 |
| **Heroicons** | 2.x | 图标库 |
| **Marked** | 15.x | Markdown 解析 |
| **Highlight.js** | 11.x | 代码高亮 |

---

## 📦 模块详解

### 1. `super-agent-business-chat` — 对话业务核心

核心业务模块，提供完整的 Agent 对话能力：

**核心组件：**

| 类名 | 功能 |
|------|------|
| `BusinessChatService` | 对话服务入口，支持流式响应、会话管理、检查点恢复 |
| `RagChatExecutor` | RAG 知识问答执行器，负责知识检索与答案生成 |
| `RagRetrievalEngine` | 双通道混合检索引擎（向量 + 关键词） |
| `ConversationMemoryService` | 记忆服务，支持短期记忆与长期记忆 |
| `CheckpointManager` | 会话检查点管理，支持断点续聊 |

**核心特性：**

- ✅ 流式响应（Server-Sent Events）
- ✅ 会话级断点续聊（Checkpoint）
- ✅ 双通道 RAG 检索（Vector + BM25）
- ✅ 多阶段追踪（Retrieval → Rerank → Assembly → Generation）
- ✅ 会话记忆与上下文恢复
- ✅ Redis 分布式锁（防并发）

### 2. `ai-example-spring-ai-alibaba` — 框架基础示例

演示 Spring AI Alibaba Agent Framework 的基础用法：

**示例代码：**

| 类 | 功能 |
|----|------|
| `SimpleAgentController` | 最简 Agent 示例 |
| `OrderAssistantController` | 订单助手（工具调用示例） |
| `OrderTools` | 订单查询、退款等工具定义 |
| `SensitiveWordInterceptor` | 敏感词拦截器 |

**关键能力：**

- React Agent 推理
- Tool 定义与调用
- 会话级记忆（Memory）
- 日志钩子（LoggingHook）

### 3. `ai-example-rag` — RAG 知识问答示例

演示 RAG（检索增强生成）的完整流程：

- 文档上传与解析
- 向量化存储（Embedding）
- 相似度检索
- 提示词组装与答案生成

### 4. `ai-example-mcp` — MCP 工具调用示例

演示 MCP（Model Context Protocol）工具的集成：

- 外部工具注册
- 工具调用与结果处理
- 多工具编排

### 5. `super-agent-id-generator-framework` — ID 生成框架

基于雪花算法的分布式 ID 生成器：

| 类 | 功能 |
|----|------|
| `SnowflakeIdGenerator` | 雪花算法实现 |
| `CachedUidGenerator` | 缓冲区优化版本 |
| `BufferedUidProvider` | Ring Buffer 缓冲 |
| `RedisWorkerIdAssigner` | Redis 分配 Worker ID |

### 6. `vue/` — 对话前端

Vue3 + Vite 构建的对话 UI：

```
vue/
├── src/
│   ├── components/   # 组件（ChatMessage、SearchBox 等）
│   ├── views/        # 页面
│   ├── router/       # 路由配置
│   └── App.vue       # 根组件
└── package.json
```

---

## 🚀 快速开始

### 前置条件

- JDK 17+
- Maven 3.8+
- Node.js 18+（前端）
- Redis 7.x
- Elasticsearch 8.x
- MinIO / Kafka / Neo4j（可选）

### 后端启动

```bash
# 1. 进入项目目录
cd E:\code\resume\super-agent-master

# 2. 编译打包
mvn clean install -DskipTests

# 3. 启动业务模块（示例）
cd super-agent-business/super-agent-business-chat
mvn spring-boot:run
```

### 前端启动

```bash
# 1. 进入前端目录
cd vue

# 2. 安装依赖
npm install

# 3. 启动开发服务器
npm run dev

# 4. 构建生产版本
npm run build
```

---

## 📖 示例文档

`需要的例子演示/` 目录包含企业级知识库示例文档：

| 文档 | 说明 |
|------|------|
| XX-200智能网关产品手册.pdf | 产品手册示例 |
| 客户数据分级与访问控制管理制度.md | 管理制度模板 |
| 差旅与费用报销管理办法.md | 财务制度模板 |
| 星联智服全渠道客服平台上线与运营管理手册.md | 运营文档模板 |
| 核心业务系统故障应急响应预案.md | 应急预案模板 |
| 澄星智能新员工入职培训手册.md | 培训文档模板 |
| 生产环境发布与回滚操作规范.md | 运维规范模板 |

这些文档可用于测试 RAG 知识库的导入、检索与问答能力。

---

## 🎯 核心流程

### RAG 知识问答流程

```
用户提问
    ↓
┌─────────────────────────────────────┐
│  ChatPreparationOrchestrator         │
│  （对话准备编排器）                    │
│  • 意图识别                           │
│  • 执行计划生成                       │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  RagRetrievalEngine                  │
│  （双通道检索引擎）                    │
│  • 向量相似度检索                     │
│  • BM25 关键词检索                    │
│  • 混合排序与去重                     │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  RagPromptAssemblyService            │
│  （提示词组装服务）                    │
│  • 引用注入                           │
│  • 上下文拼接                         │
│  • Prompt 模板渲染                   │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  ObservedChatModelService            │
│  （模型调用服务）                      │
│  • 流式响应                           │
│  • 阶段追踪                           │
│  • 错误重试                           │
└─────────────────────────────────────┘
    ↓
流式返回答案
```

---

## 🔑 配置说明

### application.yml（关键配置）

```yaml
spring:
  ai:
    alibaba:
      agent:
        enabled: true
    openai:
      base-url: https://api.siliconflow.cn/v1  # 硅基流动
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: deepseek-ai/DeepSeek-V3
          
# 向量存储
spring:
  elasticsearch:
    uris: http://localhost:9200
    
# 对象存储
minio:
  endpoint: http://localhost:9000
  access-key: ${MINIO_ACCESS_KEY}
  secret-key: ${MINIO_SECRET_KEY}
```



---

## 📝 许可证

本项目仅供学习参考。

---

**最后更新**: 2026-06-26
