# OpenClaw QQBot 知识库 RAG 集成指南

## 概述

本文档描述如何在 **OpenClaw QQBot 渠道插件** 中集成 `@partme.ai/openclaw-knowledge` 知识库 RAG 模块。

集成后，QQBot 用户可以：

- 通过 AI 对话向 Bot **添加知识**（文档/URL/文本）
- 通过 AI 对话向 Bot **查询知识**（基于 RAG 检索的问答）
- 通过 AI 对话**更新/删除**已有知识条目
- 支持 **Bot 级隔离**和 **Agent 级隔离**的数据沙箱

## 前置条件

- `@partme.ai/openclaw-knowledge` 已发布或本地 link
- OpenClaw QQBot 插件版本 >= 0.1.0
- 配置文件 `openclaw.plugin.json` 中已有 `channels.qqbot.knowledge` 配置段

## 集成步骤

### 1. 安装依赖

```bash
# 在工作空间根目录
pnpm add @partme.ai/openclaw-knowledge --filter @mocrane/qqbot
```

### 2. 注册知识库模块

在 `openclaw-qqbot/src/index.ts` 中：

```typescript
import {
  registerKnowledgeHooks,
  createKnowledgeAddTool,
  createKnowledgeQueryTool,
  createKnowledgeUpdateTool,
  createKnowledgeDeleteTool,
} from "@partme.ai/openclaw-knowledge";
```

在 `onRegister()` 回调中：

```typescript
async onRegister(config, { registerHook, registerTool }) {
  const knowledge = registerKnowledgeHooks(config, {
    configPath: "channels.qqbot.knowledge",
  });

  registerHook("before_prompt_build", knowledge.beforePromptBuild);

  registerTool(createKnowledgeAddTool(knowledge));
  registerTool(createKnowledgeQueryTool(knowledge));

  // 可选：需要更新/删除能力时也注册
  registerTool(createKnowledgeUpdateTool(knowledge));
  registerTool(createKnowledgeDeleteTool(knowledge));
}
```

### 3. 配置项

在 `openclaw.plugin.json` 的 `channels.qqbot` 下添加：

```json
{
  "channels": {
    "qqbot": {
      "knowledge": {
        "enabled": true,
        "vector_store": {
          "type": "zvec",
          "zvec": { "table": "knowledge_vectors" }
        },
        "embedding": {
          "provider": "openai",
          "model": "text-embedding-3-small"
        },
        "chunker": {
          "chunk_size": 512,
          "chunk_overlap": 64
        },
        "retrieval": {
          "top_k": 5,
          "min_score": 0.7
        },
        "intent_gate": {
          "mode": "rule"
        }
      }
    }
  }
}
```

### 4. Bot 级 / Agent 级隔离

- **Bot 级隔离**：`namespace = accountId`
- **Agent 级隔离**：`namespace = accountId:agentId`

`registerKnowledgeHooks` 自动从会话上下文中提取隔离标识。

## 可用工具

| 工具名 | 功能 | 必需 |
|--------|------|------|
| `knowledge_add` | 添加知识（URL/文件/文本） | ✅ |
| `knowledge_query` | 基于语义检索的知识问答 | ✅ |
| `knowledge_update` | 更新知识条目 | ❌ |
| `knowledge_delete` | 删除知识条目 | ❌ |

## 数据流

```
用户消息
  ↓ before_prompt_build Hook
  ↓ IntentGate（Rule 模式 ≈0ms 过滤）
知识检索请求？
  ├── 否 → 正常 LLM 对话
  └── 是 → Embedding → ZVec 检索 → Reranker（可选）
         ↓
      检索结果注入 System Prompt → LLM 生成回答
```

## 验证

向 QQ Bot 发送：

```
帮我记住：公司的午休时间是 12:00-13:30
```

确认后问：

```
公司的午休时间是什么时候？
```

应基于知识返回正确回答。

## 注意事项

- `before_prompt_build` Hook 只能注册一次
- `IntentGate` 默认 `rule` 模式约 0ms，不影响正常对话速度
- 建议在 Agent Prompt 中引导 AI 区分"添加"和"查询"场景
