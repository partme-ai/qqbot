# OpenClaw-QQBot 知识库 RAG 集成文档

> **将 `@partme.ai/openclaw-knowledge` 独立知识库引擎集成到 `openclaw-qqbot` 渠道插件中**，为 QQ 机器人提供完整的 RAG 能力：文档自动索引、语义检索、多租户隔离、AI 主动 CRUD 操作。

---

**术语速记：**
- **RAG（检索增强生成）**：对话时从向量库检索相关知识，注入 AI 上下文以提升回答准确性
- **Embedding**：将文本转为语义向量的过程
- **VectorStore**：存储和检索向量的数据库引擎
- **Chunk**：文档切分后的文本块
- **Namespace**：知识库命名空间，用于多租户数据隔离

---

## 1. 概述

### 1.1 什么是 `@partme.ai/openclaw-knowledge`

`@partme.ai/openclaw-knowledge` 是 OpenClaw 生态中的**独立知识库 RAG 引擎插件**，提供以下核心能力：

| 能力 | 说明 |
|------|------|
| 🔍 **混合检索** | 向量语义检索 + FTS5 关键词全文检索的加权融合 |
| 📄 **文档索引** | 自动切分、嵌入、存储 `.md/.txt/.csv/.json` 等文本文件 |
| 🧠 **AI 主动 CRUD** | 4 个 Tool（add/query/update/delete）让 AI 自主操作知识库 |
| 🔒 **多租户隔离** | 基于 `{accountId}:{mode}` 命名空间的严格数据隔离 |
| ⚡ **零依赖起步** | 内置 ZVec 纯 JS 向量引擎，无需外部数据库即可运行 |
| 🔗 **可选增强** | Reranker 重排序、Tokenizer 上下文截断、DocParser 文档解析 |
| 🎯 **Intent Gate** | 规则/关键词门控，只在需要时触发检索，节省 Token |

### 1.2 与 openclaw-qqbot 的关系

`openclaw-qqbot` 是一个**渠道插件**（Channel Plugin），负责 QQ 消息的收发与事件处理。

知识库 RAG 能力由 `@partme.ai/openclaw-knowledge` 作为 **npm 依赖**引入，在 `onRegister` 中完成两件事：

1. **调用 `registerKnowledgeHooks`** — 注册 `before_prompt_build` hook，实现**对话时自动检索注入**
2. **注册 4 个知识库 Tool** — 让 AI 在对话中主动读写知识库

```
┌───────────────────────────────────────────────────┐
│               openclaw-qqbot                        │
│  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ Bot / Agent   │  │   @partme.ai/            │  │
│  │ 消息处理      │  │   openclaw-knowledge     │  │
│  │              │  │                          │  │
│  │ 用户消息 →   │  │  registerKnowledgeHooks │  │
│  │ 发送回复     │  │  createKnowledgeAddTool  │  │
│  │              │  │  createKnowledgeQueryTool│  │
│  │              │  │  createKnowledgeUpdateTool│  │
│  │              │  │  createKnowledgeDeleteTool│  │
│  └──────────────┘  └──────────────────────────┘  │
└───────────────────────────────────────────────────┘
```

---

## 2. 安装

### 2.1 添加 npm 依赖

在 `openclaw-qqbot` 项目中安装：

```bash
npm install @partme.ai/openclaw-knowledge
# 或
pnpm add @partme.ai/openclaw-knowledge
# 或
yarn add @partme.ai/openclaw-knowledge
```

### 2.2 验证安装

```bash
# 确认依赖已添加
npm ls @partme.ai/openclaw-knowledge

# 输出示例
# openclaw-qqbot@2026.3.24-beta /path/to/openclaw-qqbot
# └── @partme.ai/openclaw-knowledge@0.1.0
```

---

## 3. 集成方式

### 3.1 package.json 依赖声明

```json
{
  "name": "@mocrane/qqbot",
  "version": "2026.3.24-beta",
  "dependencies": {
    "@partme.ai/openclaw-knowledge": "^0.1.0",
    "zod": "^4.3.6"
    // ... 其他已有依赖
  }
}
```

### 3.2 onRegister 集成代码

在插件的入口文件（如 `index.ts`）中的 `onRegister` 函数内完成集成：

```typescript
import type { PluginApi } from 'openclaw/plugin-sdk';
import {
  registerKnowledgeHooks,
  createKnowledgeAddTool,
  createKnowledgeQueryTool,
  createKnowledgeUpdateTool,
  createKnowledgeDeleteTool,
} from '@partme.ai/openclaw-knowledge';

export function onRegister(api: PluginApi) {
  // ── 1. 注册知识库 hooks（自动检索注入） ──
  // 第二个参数 "channels.qqbot.knowledge" 指定配置读取路径
  registerKnowledgeHooks(api, 'channels.qqbot.knowledge');

  // ── 2. 注册 4 个知识库 CRUD Tool ──
  api.registerTool(createKnowledgeAddTool,    { name: 'knowledge_add' });
  api.registerTool(createKnowledgeQueryTool,  { name: 'knowledge_query' });
  api.registerTool(createKnowledgeUpdateTool, { name: 'knowledge_update' });
  api.registerTool(createKnowledgeDeleteTool, { name: 'knowledge_delete' });

  // ── 3.（可选）在 knowledge.enabled 时注入使用指引 ──
  const knowledgeEnabled = !!(api.config as any)?.channels?.qqbot?.knowledge?.enabled;
  if (knowledgeEnabled) {
    api.on('before_prompt_build', (_event, ctx) => {
      if (ctx.channelId !== 'qqbot') return;
      return {
        systemPrompt: [
          '【知识库 Tool】你拥有 4 个知识库工具：',
          '- knowledge_add    → 写入（store_text / store_file / store_summary）',
          '- knowledge_query  → 检索（vector / keyword / hybrid）',
          '- knowledge_update → 更新（按 sourceId 替换内容）',
          '- knowledge_delete → 删除（按 sourceId 删除 / 清空命名空间）',
          '用户说"记住这个""搜索知识库""更新资料""删除 XX"时主动调用。',
        ].join('\n'),
      };
    });
  }
}
```

### 3.3 集成要点

| 要点 | 说明 |
|------|------|
| **Tool 名称** | 注册为 `knowledge_*`（**不是** `qqbot_knowledge_*`），保持与 `@partme.ai/openclaw-knowledge` 命名一致 |
| **配置路径** | `registerKnowledgeHooks(api, 'channels.qqbot.knowledge')` 告诉引擎从 `channels.qqbot.knowledge` 读取配置 |
| **纯加法** | 不修改原有 Bot/Agent 消息流的任何逻辑 |
| **错误安全** | Hook 内部 try/catch 兜底，失败时静默返回 `undefined`，不影响主流程 |
| **通道过滤** | Hook 内部按 `ctx.channelId === 'qqbot'` 过滤，不影响其他通道 |
| **命名空间** | 自动按 `{accountId}:{mode}` 隔离（如 `default:bot`、`acme_corp:agent`） |

---

## 4. 配置说明

知识库 RAG 的所有配置项位于 **`channels.qqbot.knowledge.*`** 路径下。

### 4.1 最小配置

```json
{
  "channels": {
    "qqbot": {
      "knowledge": {
        "enabled": true,
        "embedding": {
          "model": "text-embedding-3-small"
        }
      }
    }
  }
}
```

> `embedding.baseUrl` 和 `apiKey` 未填写时自动复用 LLM 侧的 OpenAI 兼容配置。

### 4.2 完整配置项

```json
{
  "channels": {
    "qqbot": {
      "knowledge": {
        "enabled": true,

        "embedding": {
          "provider": "openai",
          "baseUrl": "https://api.openai.com/v1",
          "apiKey": "***",
          "model": "text-embedding-3-small",
          "dimensions": 1536
        },

        "store": {
          "provider": "zvec",
          "dbPath": "/data/knowledge/qqbot.db",
          "sources": {
            "docIds": ["doc-001", "doc-002"],
            "docDirs": ["/path/to/docs"],
            "urls": ["https://example.com/docs/api.md"]
          }
        },

        "retrieval": {
          "strategy": "hybrid",
          "topK": 5,
          "minScore": 0.3,
          "keywordBoost": true
        },

        "injection": {
          "position": "system",
          "template": "以下是相关知识库内容：\n\n{context}\n\n请基于以上内容回答用户问题。如果知识库中没有相关信息，请如实告知。",
          "maxChunks": 5,
          "maxTokens": 2048
        },

        "tokenizer": {
          "provider": "tiktoken"
        },

        "reranker": {
          "provider": "jina",
          "apiKey": "***"
        },

        "parser": {
          "provider": "ollama",
          "baseUrl": "http://localhost:11434"
        },

        "moderation": {
          "enabled": false,
          "rejectionMessage": "抱歉，我无法回答这个问题。"
        }
      }
    }
  }
}
```

### 4.3 配置项详解

#### `embedding` — Embedding 配置

| 字段 | 类型 | 必需 | 默认值 | 说明 |
|------|------|------|--------|------|
| `provider` | string | 否 | `"openai"` | 仅支持 OpenAI 兼容 API |
| `baseUrl` | string | 否 | 复用 LLM 配置 | Embedding API 端点 |
| `apiKey` | string | 否 | 复用 LLM 配置 | API 密钥 |
| `model` | string | 否 | `"text-embedding-3-small"` | 嵌入模型 |
| `dimensions` | number | 否 | 模型默认 | 输出向量维度 |

> **推荐模型**：`text-embedding-3-small`（性能/成本平衡）或 `text-embedding-3-large`（更高精度）。
>
> **重要**：切换模型后若 `dimensions` 变化，旧索引数据需删除重建。

#### `store` — 向量存储配置

| 字段 | 类型 | 必需 | 默认值 | 说明 |
|------|------|------|--------|------|
| `provider` | string | 否 | `"zvec"` | 存储引擎 |
| `dbPath` | string | 否 | 自动生成 | 持久化数据库路径 |
| `sources` | object | 否 | `{}` | 文档来源配置 |

**支持的存储引擎**：

| 引擎 | 场景 | 依赖 |
|------|------|------|
| `zvec` | 开发调试 | 零依赖（纯 JS 内存引擎，重启丢失） |
| `sqlite-vec` | 轻量生产 | `better-sqlite3` |
| `redis` | 高速缓存 | Redis 服务 + `redis` |
| `pinecone` | 云端生产 | Pinecone 账号 |
| `chroma` | 本地生产 | Chroma 服务 |
| `qdrant` | 高性能 | Qdrant 服务 |
| `milvus` | 大规模分布式 | Milvus 服务 |
| `pgvector` | 已有 PostgreSQL | `pg` + pgvector 扩展 |

#### `retrieval` — 检索配置

| 字段 | 类型 | 必需 | 默认值 | 说明 |
|------|------|------|--------|------|
| `strategy` | string | 否 | `"hybrid"` | `vector` / `keyword` / `hybrid` |
| `topK` | number | 否 | `5` | 返回结果数 |
| `minScore` | number | 否 | `0.0` | 最低相似度阈值（0-1） |
| `keywordBoost` | boolean | 否 | `true` | 是否启用 BM25 关键词增强 |

**策略选择**：
- `vector`：纯语义检索，适合同义词/意图匹配
- `keyword`：精确匹配，适合产品编号/专有名词
- `hybrid`：加权融合（默认 0.7 向量 + 0.3 关键词），适合大多数场景

#### `injection` — 上下文注入配置

| 字段 | 类型 | 必需 | 默认值 | 说明 |
|------|------|------|--------|------|
| `position` | string | 否 | `"system"` | `system`（系统提示）或 `user`（用户提示） |
| `template` | string | 否 | 见下 | 上下文格式化模板 |
| `maxChunks` | number | 否 | `5` | 注入的最大块数 |
| `maxTokens` | number | 否 | `2048` | 注入的最大 token 数 |

**默认模板**：
```
以下是相关知识库内容，请据此回答用户问题：

{context}
```

> `{context}` 会被替换为检索到的文本块。`position: "user"` 适合需要用户确认上下文的场景。

#### `tokenizer` — 分词计数（可选）

| 字段 | 类型 | 必需 | 默认值 | 说明 |
|------|------|------|--------|------|
| `provider` | string | 否 | 未配置（不启用） | `zhipu` / `tiktoken` |

> 配置后，注入上下文的文本会被截断到 `injection.maxTokens` 以内，确保不超出 LLM 上下文窗口。

#### `reranker` — 重排序（可选）

| 字段 | 类型 | 必需 | 默认值 | 说明 |
|------|------|------|--------|------|
| `provider` | string | 否 | 未配置（不启用） | `zhipu` / `jina` / `ollama` |
| `apiKey` | string | 是(云端) | - | 云端服务必需 |

> 配置后，检索结果会经过重排序再注入上下文，显著提升相关性（尤其 `topK` > 10 时）。

#### `parser` — 文档解析（可选）

| 字段 | 类型 | 必需 | 默认值 | 说明 |
|------|------|------|--------|------|
| `provider` | string | 否 | 未配置（不启用） | `zhipu` / `ollama` |

> 配置后，PDF/图片等非纯文本文件会被自动解析为 Markdown 再索引。

### 4.4 多账户配置（多租户场景）

不同的 QQ Bot 账号（account）可以拥有独立的知识库，配置继承全局值，account 级进行深度覆盖。

```json
{
  "channels": {
    "qqbot": {
      "knowledge": {
        "enabled": true,
        "embedding": { "model": "text-embedding-3-small" },
        "store": {
          "provider": "sqlite-vec",
          "dbPath": "/data/knowledge/default.db"
        },
        "retrieval": { "topK": 5 }
      },
      "accounts": {
        "acme_corp": {
          "knowledge": {
            "store": { "dbPath": "/data/knowledge/acme.db" },
            "retrieval": { "topK": 10, "minScore": 0.5 }
          }
        },
        "globex": {
          "knowledge": {
            "store": {
              "provider": "pinecone",
              "pineconeApiKey": "xxx",
              "pineconeEnvironment": "us-east-1",
              "pineconeIndexName": "globex-knowledge"
            }
          }
        }
      }
    }
  }
}
```

**合并规则**：
- `enabled`：继承全局，account 级不可覆盖
- `embedding/store/retrieval/injection/moderation`：深度合并，account 级字段覆盖全局同名字段
- `store.sources`：完全替换（不合并）

---

## 5. 注册的 Tool 说明

通过 `@partme.ai/openclaw-knowledge` 注册的 4 个 Tool 名称统一为 `knowledge_*`：

| Tool | 名称 | 功能 | 关键参数 |
|------|------|------|----------|
| 写入 | `knowledge_add` | 存储文本/文件/对话总结 | `action`（store_text/store_file/store_summary）|
| 检索 | `knowledge_query` | 语义/关键词/混合检索 | `query`、`strategy`、`topK`、`minScore` |
| 更新 | `knowledge_update` | 按 sourceId 替换内容 | `sourceId`、`updateType`、新内容 |
| 删除 | `knowledge_delete` | 按 sourceId 删除/清空命名空间 | `action`（delete_by_source/clear）|

> **注意**：Tool 名称为 `knowledge_*`（而非 `qqbot_knowledge_*`），与 `@partme.ai/openclaw-knowledge` 的导出命名保持一致，便于在跨渠道插件中复用同一套知识库 Tool。

### 典型使用场景

| 用户指令 | AI 行为 | 调用的 Tool |
|----------|---------|------------|
| "记住这个：XX 项目的 API Key 是 sk-xxx" | 将文字内容存入当前对话知识库 | `knowledge_add`（store_text） |
| "帮我把这个文件保存到知识库" | 校验文件后索引到知识库 | `knowledge_add`（store_file） |
| "总结今天的对话" | 组合主题+总结内容后存入 | `knowledge_add`（store_summary） |
| "之前存的报销流程在哪？帮我查一下" | 显式检索知识库 | `knowledge_query` |
| "上个月存的服务器配置有更新" | 按 sourceId 替换 | `knowledge_update` |
| "把报销流程那条删除" | 按 sourceId 删除 | `knowledge_delete`（delete_by_source） |

---

## 6. 快速验证

### 6.1 确认集成生效

```bash
# 查看插件日志，确认 knowledge hooks 和 tools 已加载
openclaw logs | grep -i knowledge

# 预期输出
# [Knowledge] registerKnowledgeHooks: configPath=channels.qqbot.knowledge
# [Knowledge] Registered tool: knowledge_add
# [Knowledge] Registered tool: knowledge_query
# [Knowledge] Registered tool: knowledge_update
# [Knowledge] Registered tool: knowledge_delete
```

### 6.2 验证配置读取

```bash
# 检查配置是否正确读取
openclaw config get channels.qqbot.knowledge.enabled
# → true

openclaw config get channels.qqbot.knowledge.embedding.model
# → text-embedding-3-small
```

### 6.3 端到端验证

1. 向 QQ Bot 发送一条包含关键信息的消息，如"记住这个：我的工号是 2024001"
2. AI 应调用 `knowledge_add` 工具存储该信息
3. 再发送"我的工号是什么？"
4. AI 应通过 Hook 自动检索知识库，正确回答"2024001"

---

## 7. 运维命令

| 命令 | 用途 | 示例 |
|------|------|------|
| 检查集成状态 | 确认 knowledge 模块已加载 | `openclaw plugins list` |
| 查看配置 | 检查知识库配置项 | `openclaw config get channels.qqbot.knowledge` |
| 查看日志 | 检索知识库相关日志 | `openclaw logs \| grep KNOWLEDGE` |
| 清除缓存 | 重启后重建 VectorStore 缓存 | 重启 OpenClaw Gateway |

---

## 8. 生产建议

### 8.1 持久化存储

开发阶段使用 ZVec（内存存储，重启后数据丢失）。生产环境务必切换到持久化后端：

```json
{
  "store": {
    "provider": "sqlite-vec",
    "dbPath": "/var/lib/openclaw/knowledge.db"
  }
}
```

> SQLite-Vec 依赖 `better-sqlite3` 原生模块，在 `@partme.ai/openclaw-knowledge` 中已声明为依赖。

### 8.2 Embedding API 安全

- 不使用复用 LLM 配置时，在 Embedding 配置中单独指定 API Key
- 使用独立的 Embedding 模型账户，便于成本跟踪和限流
- 避免在日志中打印完整 API Key

### 8.3 多租户数据路径规划

```bash
/var/lib/openclaw/knowledge/
├── default/
│   ├── bot.db      # 默认 Bot 知识库
│   └── agent.db    # 默认 Agent 知识库
├── acme_corp/
│   ├── bot.db      # Acme 公司 Bot 知识库
│   └── agent.db    # Acme 公司 Agent 知识库
└── globex/
    └── bot.db      # Globex 公司 Bot 知识库
```

### 8.4 性能调优参数

| 参数 | 建议值 | 说明 |
|------|--------|------|
| `retrieval.topK` | 3-5 | 检索返回结果数 |
| `retrieval.minScore` | 0.3-0.5 | 相似度阈值，过低引入噪声，过高丢失召回 |
| `injection.maxTokens` | 1024-2048 | 注入上下文的 token 上限 |
| `tokenizer.provider` | `"tiktoken"` | 启用后自动截断上下文，防止超窗口 |

### 8.5 依赖管理

```bash
# 确保 @partme.ai/openclaw-knowledge 版本锁定
npm install @partme.ai/openclaw-knowledge@0.1.0 --save-exact

# 生产环境安装
npm ci --production
```

---

## 9. 故障排除

### 9.1 知识库未生效

**症状**：对话中没有知识库上下文注入。

**排查步骤**：
```bash
# 1. 确认 knowledge 配置已启用
openclaw config get channels.qqbot.knowledge.enabled

# 2. 确认 embedding 配置正确
openclaw config get channels.qqbot.knowledge.embedding

# 3. 查看日志中是否有知识库相关输出
openclaw logs | grep KNOWLEDGE
```

**常见原因**：
- `enabled: false`（未启用）
- Embedding API Key 无效或网络不可达
- 向量维度不匹配（模型切换后未重建索引）
- `minScore` 设置过高，没有命中结果

### 9.2 Tool 未注册

**症状**：AI 无法调用 `knowledge_*` 工具。

**排查步骤**：
```bash
# 确认 @partme.ai/openclaw-knowledge 已安装
npm ls @partme.ai/openclaw-knowledge

# 查看插件注册日志
openclaw logs | grep "registerTool"
```

### 9.3 文件上传后未索引

**症状**：用户上传文件后，AI 无法检索相关内容。

**排查**：
- 确认文件格式为 `.md/.txt/.csv/.json`（未配置 parser 时仅支持纯文本格式）
- 检查 Embedding API 是否正常
- 查看日志中的 `[KNOWLEDGE]` 索引记录

---

## 10. 参考

| 资源 | 链接 |
|------|------|
| `@partme.ai/openclaw-knowledge` 源码 | [GitHub](https://github.com/partme-ai/openclaw-knowledge) |
| OpenClaw 插件 SDK | [文档](https://docs.openclaw.ai) |
| 知识库 RAG 架构设计 | [架构文档](./OpenClaw-QQBot-Knowledge-RAG-Architecture_CN.md) |
| 安装与配置指南 | [使用指南](./OpenClaw-QQBot-Knowledge-RAG-Guide_CN.md) |
