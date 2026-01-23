# 架构与设计

## 1. 总体架构
- 模块：导入/解析 → 索引构建 → 检索 → 生成 → 引用组装
- 数据流：文档 → chunk → 索引 → query →候选→ rerank → answer + citations

## 2. 模块边界
- 文档导入与解析
- 索引层（向量 + 关键词）
- 检索与 rerank
- 生成与引用
- API 与 UI（如果有）

## 3. 数据与接口
- 输入：文件/目录、查询文本
- 输出：答案、引用、置信度
- 元数据：文档来源、页码/段落、时间

## 4. 关键决策记录（ADR）
- 使用 LangChain4j
- 向量库选型（待定）
- 混检策略（待定）

## 5. 风险
- 引用漂移（chunk 位置变化）
- 文档更新后的索引一致性

## 本阶段完成标准
- 架构图/数据流图完成
- 模块职责清晰
- 关键技术选型初步确定
## 6. 技术选型（推荐组合）

### 6.1 基础栈（MVP 优先）
- **LangChain4j（Java）**：作为编排层，统一文档导入、向量化、检索与生成流程。
- **PostgreSQL**：统一承载结构化元数据与检索索引。
- **向量检索：pgvector**：在 Postgres 内完成向量检索，减少组件数量。
- **关键词检索：PostgreSQL Full-Text Search**（`tsvector/tsquery` + GIN）
  - 说明：不是严格 BM25，但可作为 MVP 的“BM25-ish”实现。
- **外部 LLM API**：生成回答与（可选）轻量 rerank。

### 6.2 为什么这样选
- **部署简单**：只引入一个数据库组件，降低自部署复杂度。
- **可演进**：后续可平滑升级到 ES/OpenSearch 或独立向量库。
- **一致性强**：向量与关键词结果在同一存储层聚合，易做混合检索。

### 6.3 MVP 级技术边界（明确不做）
- 不引入独立搜索引擎（ES/OpenSearch）
- 不引入多向量库并存
- 不做复杂多策略 rerank

### 6.4 扩展路径（阶段性升级）
- **M2：增量索引**
  - 文档版本 + chunk hash 做变更检测与增量更新
- **M2：rerank**
  - 先用轻量模型或 API rerank
- **M3：评测体系**
  - 评价检索与生成效果，形成回归基线

### 6.5 最小闭环的数据流（对应选型）
1. 文档导入 → 解析 → 切分 chunk（含元数据）
2. Embedding 写入 `pgvector` 表
3. 关键词向量写入 `tsvector` 字段
4. 查询时：keyword FTS 召回 + vector 相似度召回
5. 结果合并（score 融合）→ LLM 生成 → 组装引用
## 7. 数据表设计（推荐方案 A）

### 7.1 表结构概要
- `documents`：文档级元数据与版本
- `chunks`：文本切片与位置信息
- `chunk_embeddings`：向量数据（与文本解耦）
- `chunk_fts`：全文检索索引数据

### 7.2 字段建议（草案）

**documents**
- `id` (PK)
- `source` / `title` / `mime_type`
- `version` / `checksum`
- `tenant_id`
- `created_at` / `updated_at`

**chunks**
- `id` (PK)
- `document_id` (FK)
- `chunk_index`
- `text`
- `start_offset` / `end_offset`
- `page`
- `hash`
- `metadata` (jsonb)

**chunk_embeddings**
- `chunk_id` (PK/FK)
- `embedding` (vector)
- `model_id`
- `dim`
- `created_at`

**chunk_fts**
- `chunk_id` (PK/FK)
- `tsv` (tsvector)

### 7.3 索引建议
- `chunk_fts.tsv` 建 GIN 索引
- `chunk_embeddings.embedding` 建向量索引（pgvector）
- `chunks.document_id` 建 BTREE 索引

### 7.4 查询与增量
- 查询时：FTS 召回 + 向量召回并行，再融合评分
- 增量：通过 `documents.version` + `chunks.hash` 判断变更，按 chunk 更新向量与 FTS

### 7.5 选择理由
- 内容 / 向量 / FTS 解耦，便于扩展与重建索引
- 在不引入 ES 的前提下，支持 MVP 的混合检索闭环
## 8. API 设计（REST + SSE）

### 8.1 端点清单（MVP）

**导入**
- `POST /v1/documents:import`：上传文件或目录列表，返回任务 id
- `GET /v1/documents:import/{id}`：查询导入状态

**检索问答**
- `POST /v1/query`：非流式回答（答案 + 引用）
- `POST /v1/query:stream`：SSE 流式回答
  - 事件示例：`answer.delta`、`citations.final`

**索引管理**
- `POST /v1/index:build`：全量建索引
- `POST /v1/index:refresh`：增量索引
- `GET /v1/index:status`：索引版本与状态

**系统**
- `GET /v1/health`
- `GET /v1/metrics`（可选）

### 8.2 请求与响应（要点）
- 请求通用字段：`tenant_id`、`source`（可选标签）
- 响应通用字段：`trace_id`、`elapsed_ms`
- 引用结构：`citations[]` 含 `document_id`、`chunk_id`、`page`、`score`

### 8.3 SSE 事件建议
- `answer.delta`：逐段答案增量
- `citations.final`：最终引用列表
- `error`：错误信息
## 9. 混合检索融合策略（线性可解释）

### 9.1 召回与融合
- 两路召回：FTS（关键词）与向量检索各取 TopK
- 归一化：
  - FTS：`ts_rank` 或 `ts_rank_cd` 归一化到 0–1
  - 向量：`1 - distance` 或 cosine 相似度归一化到 0–1
- 融合公式：`score = α * score_vector + (1-α) * score_fts`
  - 默认 `α=0.6`（语义优先）

### 9.2 排序与截断
- 合并去重后按 `score` 排序
- 取 TopN 进入生成阶段

### 9.3 可解释性
- 保留 `score_vector`、`score_fts` 与 `score_final`
- MVP 阶段以 TopN 截断为主，少用硬阈值
## 10. 增量索引任务流与状态机（事件 + 定时回退）

### 10.1 触发策略
- **事件驱动优先**：文件变更（新增/修改/删除）触发入队
- **定时回退**：周期性扫描（如每日）修复漏触发与一致性问题

### 10.2 任务流（M2 目标）
1. **变更检测**：对文件计算 `checksum` 与 `version`
2. **入队**：生成索引任务（document_id + version）
3. **解析与切分**：生成 chunk + hash
4. **差异计算**：对比旧 chunk hash，确定新增/更新/删除
5. **索引更新**：
   - 新增/更新：写入向量与 FTS
   - 删除：标记失效或物理删除
6. **完成与审计**：写入任务日志与统计指标

### 10.3 状态机（索引任务）
- `PENDING` → `RUNNING` → `SUCCEEDED`
- `RUNNING` → `FAILED`（可重试）
- `FAILED` → `RETRYING` → `SUCCEEDED`
- `SUCCEEDED` → `VERIFIED`（可选一致性校验）

### 10.4 幂等与一致性
- 按 `document_id + version` 保证幂等
- chunk 层以 `hash` 为准，避免重复写入

### 10.5 观测与指标
- 任务耗时、失败率
- 每次增量影响的 chunk 数
- 索引版本与更新时间
## 11. 接口字段与示例（JSON 为主）

### 11.1 文档导入

**请求** `POST /v1/documents:import`
```json
{
  "tenant_id": "t1",
  "source": "local",
  "paths": ["/data/docs"],
  "options": {
    "recursive": true,
    "mime_whitelist": ["application/pdf", "text/markdown", "text/plain"]
  }
}
```

**响应**
```json
{
  "import_id": "imp_123",
  "status": "PENDING",
  "queued_at": "2026-01-23T08:00:00Z"
}
```

### 11.2 导入状态

**请求** `GET /v1/documents:import/{id}`

**响应**
```json
{
  "import_id": "imp_123",
  "status": "SUCCEEDED",
  "counts": {"documents": 120, "chunks": 3540},
  "elapsed_ms": 18234
}
```

### 11.3 检索问答（非流式）

**请求** `POST /v1/query`
```json
{
  "tenant_id": "t1",
  "query": "这个项目的MVP范围是什么？",
  "top_k": 8
}
```

**响应**
```json
{
  "answer": "MVP包含文档导入、混合检索和带引用回答...",
  "citations": [
    {
      "document_id": "doc_9",
      "chunk_id": "chk_77",
      "page": 3,
      "score": 0.82
    }
  ],
  "trace_id": "tr_abc",
  "elapsed_ms": 2450
}
```

### 11.4 检索问答（SSE 流式）

**请求** `POST /v1/query:stream`
```json
{
  "tenant_id": "t1",
  "query": "给出最小闭环的流程",
  "top_k": 8
}
```

**事件（示意）**
```json
{"event":"answer.delta","data":"最小闭环包含..."}
```
```json
{"event":"citations.final","data":[{"document_id":"doc_2","chunk_id":"chk_11","page":1,"score":0.79}]}
```

### 11.5 索引管理

**请求** `POST /v1/index:build`
```json
{
  "tenant_id": "t1",
  "mode": "full"
}
```

**响应**
```json
{
  "index_job_id": "idx_901",
  "status": "RUNNING"
}
```
## 12. 引用生成与校验规范（简洁版）

### 12.1 引用输出结构
- 每条引用包含：`document_id`、`title`（可选）、`page` 或 `section`、`score`
- 不暴露内部 `chunk_id` 与 offset（可在调试模式输出）

### 12.2 引用生成流程
1. 从最终 TopN chunk 中提取来源信息
2. 对同一文档的相邻 chunk 做合并（减少重复）
3. 生成引用列表并按贡献度排序

### 12.3 引用校验
- 引用必须与回答内容显式对应（若无法对应，降级为“无引用回答”）
- 当引用来自同一段落，优先合并并保留最高分

### 12.4 UI/输出建议
- 仅显示文档标题 + 页码/段落
- 提供“展开详情”开关用于调试
## 13. LLM 调用策略与成本控制（稳定一致）

### 13.1 模型策略
- 固定单一模型与版本（MVP 阶段不做动态切换）
- 固定温度与 top_p，确保输出可复现
- 仅在必要时切换模型（需记录版本变更）

### 13.2 调用约束
- 统一的 `LLMProvider` 抽象层
- 强制超时、重试与退避策略
- 记录 `trace_id` 贯通检索与生成

### 13.3 成本控制（不牺牲一致性）
- 限制上下文长度（TopN + 压缩）
- 对引用做合并去重，减少无效 tokens
- 记录每次请求 token 与成本指标

### 13.4 失败与降级
- LLM 失败时可返回“仅检索结果 + 引用”
- 错误信息标准化，避免影响上层调用
## 14. 错误码与异常处理规范

### 14.1 结构
- HTTP 状态码 + 业务错误码
- 错误响应统一结构：
```json
{
  "error": {
    "code": "E_DOC_PARSE_FAILED",
    "message": "Document parse failed",
    "trace_id": "tr_abc"
  }
}
```

### 14.2 常见错误码（示例）
- `E_DOC_PARSE_FAILED`：文档解析失败
- `E_IMPORT_PATH_INVALID`：导入路径无效
- `E_INDEX_BUILD_FAILED`：索引构建失败
- `E_QUERY_INVALID`：查询参数错误
- `E_LLM_TIMEOUT`：LLM 超时
- `E_LLM_UNAVAILABLE`：LLM 服务不可用

### 14.3 处理原则
- 所有错误必须返回 `trace_id`
- 对客户端可修复错误给出明确 message
- 服务器内部错误不暴露细节
## 15. 已确认的关键决策（MVP）

- 部署形态：单机（内网）
- 数据来源：内网文件目录
- 权限模型：多租户（tenant_id 隔离）
- 接口安全：内网无认证
- UI 形态：简易 Web UI（后续独立前端）
- 文档格式：PDF / MD / TXT + DOCX
- 切分策略：段落优先 + 长段再切
- Embedding：外部 API
- Rerank：MVP 不做
- 观测：基本日志
## 16. 目录导入与解析规范（含 OCR）

### 16.1 支持类型
- PDF / DOCX / MD / TXT
- PDF 若为图像型，走 OCR

### 16.2 编码与清洗
- 文本统一转 UTF-8
- 去除控制字符与重复空白
- 保留段落与分页标记

### 16.3 PDF 解析策略
- 优先解析文本层
- 无文本层则 OCR
- 记录页码与段落位置

### 16.4 DOCX/MD/TXT
- DOCX 保留段落层级
- MD 保留标题层级与段落
- TXT 按空行分段

### 16.5 失败处理
- 失败文件记录在导入报告
- OCR 失败不阻断任务（标记为 failed）
## 17. 架构图与流程图（文字版）

### 17.1 架构模块
- 导入与解析 → 切分 → 索引（向量/FTS） → 检索融合 → 生成与引用 → API/UI

### 17.2 查询流程
1. 接收 query
2. FTS 与向量并行召回
3. 线性融合与去重
4. TopN 送入 LLM
5. 生成答案并组装引用
6. 返回响应（SSE/JSON）

### 17.3 索引流程
1. 导入目录与文件解析
2. OCR（如需）
3. 生成 chunk + metadata
4. 写入向量与 FTS
5. 记录索引版本
## 18. 数据库建表 SQL 草案（PostgreSQL）

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE IF NOT EXISTS documents (
  id TEXT PRIMARY KEY,
  source TEXT,
  title TEXT,
  mime_type TEXT,
  version TEXT,
  checksum TEXT,
  tenant_id TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE IF NOT EXISTS chunks (
  id TEXT PRIMARY KEY,
  document_id TEXT REFERENCES documents(id) ON DELETE CASCADE,
  chunk_index INT,
  text TEXT,
  start_offset INT,
  end_offset INT,
  page INT,
  hash TEXT,
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE IF NOT EXISTS chunk_embeddings (
  chunk_id TEXT PRIMARY KEY REFERENCES chunks(id) ON DELETE CASCADE,
  embedding VECTOR(1536),
  model_id TEXT,
  dim INT,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE IF NOT EXISTS chunk_fts (
  chunk_id TEXT PRIMARY KEY REFERENCES chunks(id) ON DELETE CASCADE,
  tsv TSVECTOR
);

CREATE INDEX IF NOT EXISTS idx_chunks_document_id ON chunks(document_id);
CREATE INDEX IF NOT EXISTS idx_chunk_fts_tsv ON chunk_fts USING GIN(tsv);
-- 向量索引示例（根据 pgvector 版本选择）
-- CREATE INDEX idx_chunk_embeddings_vec ON chunk_embeddings USING ivfflat (embedding vector_cosine_ops);
```
## 19. OCR 方案与限制（外部 API）

### 19.1 方案选择
- 使用外部 OCR API 处理图像型 PDF
- 优先保证接入简单与稳定性

### 19.2 调用策略
- 仅在检测不到文本层时触发 OCR
- 按页拆分调用，支持并行但限制并发
- 超时与重试策略明确

### 19.3 成本与性能
- OCR 成本按页计费，需记录页数
- OCR 结果缓存，避免重复调用

### 19.4 失败处理
- OCR 失败标记为 failed，不阻断导入任务
- 在报告中显示 OCR 失败原因与文件清单