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