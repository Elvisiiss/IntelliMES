# M021: Elasticsearch向量检索集成

> **所属阶段**：第四阶段 - AI 知识库问答
> **预计周期**：第 7 周
> **优先级**：P0
> **前置任务**：M020

## 一、任务目标
将 knowledge_doc 中的文档分块（chunking），调用 Embedding 模型向量化，存入 Elasticsearch `knowledge_base` 索引的 `dense_vector` 字段（参考计划书 5.2 knowledge_base 索引设计），实现基于向量的语义检索能力，为 RAG 与混合检索奠定基础。

## 二、前置条件
- M020 已完成，knowledge_doc 表有文档数据
- M005 已完成，ES 客户端与 `knowledge_base` 索引已初始化（dense_vector dims 1024）
- 已申请 Embedding 模型 API（如 bge-large-zh 或通义 text-embedding）

## 三、详细执行步骤
### 步骤 1：配置 Embedding 模型
在 `intellimes-ai` 引入 Spring AI Embedding 依赖：
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```
在 `application-dev.yml` 配置 Embedding：
```yaml
spring:
  ai:
    openai:
      embedding:
        options:
          model: bge-large-zh-v1.5  # 或 text-embedding-v2
```

### 步骤 2：编写文档分块器 TextChunker
`TextChunker`：
- `chunk(String text, int chunkSize, int overlap) -> List<String>`
- 按字符长度分块（如 chunkSize=500，overlap=50）
- 优先按段落/句子边界切分，避免破坏语义
- 返回分块列表

### 步骤 3：编写 EmbeddingService
`EmbeddingService`：
- `embed(String text) -> float[]`：调用 `EmbeddingModel.embed(text)` 返回向量
- `embedBatch(List<String> texts) -> List<float[]>`：批量向量化（提升效率）
- 缓存：相同文本向量存 Redis（计划书 5.3 知识库问答缓存场景），避免重复调用

### 步骤 4：编写 ES 索引文档服务
`KnowledgeBaseIndexService`：
- `indexDocument(KnowledgeDoc doc)`：
  - 调用 TextChunker 分块
  - 调用 EmbeddingService 批量向量化
  - 构造 ES 文档：`{doc_id, title, content_chunk, embedding, file_type, create_time}`
  - 批量写入 ES `knowledge_base` 索引
  - 更新 knowledge_doc.status=1 已索引，es_index_id 记录
- `deleteDocument(String docId)`：按 doc_id 删除 ES 中所有分块
- `reindex(Long docId)`：重新索引（先删后建）

### 步骤 5：编写向量检索服务
`VectorSearchService`：
- `search(String query, int topK) -> List<SearchResult>`
  - 调用 EmbeddingService 将 query 向量化
  - 用 ES `knn` 查询检索 topK 最相似分块
  - 返回 `SearchResult(chunk, score, docId, title)`
- ES 查询示例：
```json
{
  "knn": {
    "field": "embedding",
    "query_vector": [...],
    "k": 5,
    "num_candidates": 50
  }
}
```

### 步骤 6：触发文档索引
在 `KnowledgeDocService` 中：
- 上传文档解析后自动调用 `KnowledgeBaseIndexService.indexDocument`
- 或提供手动索引接口 `POST /ai/knowledge/index/{id}`
- 索引失败时 status=2，记录错误

### 步骤 7：编写检索接口
`VectorSearchController`：
- `POST /ai/search/vector` body `{"query":"xxx","topK":5}` 返回相似分块列表
- `POST /ai/knowledge/index/{id}` 手动触发索引
- `POST /ai/knowledge/reindexAll` 重建所有索引（管理员）

### 步骤 8：编写前端检索测试页
`src/views/ai/search-test.vue`：
- 输入框输入查询
- 返回相似分块列表（标题、内容片段、相似度分数）
- 用于验证向量检索效果

### 步骤 9：联调验证
- 上传一份"SOP 操作手册.txt"
- 触发索引：分块、向量化、写入 ES
- 查 knowledge_doc.status=1
- 查 ES `knowledge_base` 索引有分块文档
- 调用 `/ai/search/vector` 查询"操作步骤是什么"返回相关分块

## 四、涉及文件清单
| 文件路径 | 说明 |
|---------|------|
| `intellimes-ai/src/main/java/com/intellimes/ai/service/TextChunker.java` | 文档分块器 |
| `intellimes-ai/src/main/java/com/intellimes/ai/service/EmbeddingService.java` | 向量化服务 |
| `intellimes-ai/src/main/java/com/intellimes/ai/service/KnowledgeBaseIndexService.java` | ES 索引服务 |
| `intellimes-ai/src/main/java/com/intellimes/ai/service/VectorSearchService.java` | 向量检索服务 |
| `intellimes-ai/src/main/java/com/intellimes/ai/controller/VectorSearchController.java` | 检索控制器 |
| `intellimes-ai/src/main/java/com/intellimes/ai/vo/SearchResult.java` | 检索结果对象 |
| `intellimes-ai/src/main/resources/application-dev.yml` | Embedding 配置 |
| `intellimes-frontend/src/api/ai/search.ts` | 检索 API |
| `intellimes-frontend/src/views/ai/search-test.vue` | 检索测试页 |

## 五、预期产出
- 文档分块 + 向量化 + ES 存储完整链路
- 向量语义检索接口可用
- knowledge_doc 状态流转：待处理→已索引
- ES knowledge_base 索引存储文档分块与向量
- 简历亮点："Elasticsearch 向量检索集成，文档向量化存储"

## 六、验证方式
- 上传文档索引后 ES `knowledge_base/_count` 返回分块数
- 向量检索返回语义相关分块（即使关键词不完全匹配）
- 相似度分数合理（0-1 之间，越接近 1 越相似）
- 删除文档后 ES 对应分块同步删除
- knowledge_doc.status 正确流转

## 七、技术要点与注意事项
- **knowledge_base 索引设计（计划书 5.2）**：`embedding` 字段类型 `dense_vector`，`dims` 必须与 Embedding 模型输出维度一致（bge-large-zh 为 1024，text-embedding-v2 为 1536），建索引时确定无法修改。
- **分块策略**：chunkSize 500 字 + overlap 50 字是经验值，平衡检索粒度与上下文完整性；过小丢失上下文，过大检索不精准。优先按段落/句子切分避免破坏语义。
- **批量向量化**：单条调用 Embedding API 效率低，用 `embedBatch` 批量处理（一次 10-50 条），降低 API 调用次数与费用。
- **向量缓存（计划书 5.3）**：相同文本的向量存 Redis，Key `embedding:md5:{textMd5}`，避免重复文档重复向量化。
- **ES knn 查询**：`num_candidates` 设为 topK 的 10 倍左右（如 topK=5, num_candidates=50），平衡召回率与性能；过大影响性能，过小漏召回。
- **索引异步化**：向量化与 ES 写入耗时长，建议异步执行（`@Async` 或 MQ），上传接口立即返回，索引完成后更新 status。
- **Embedding 模型选择**：bge-large-zh-v1.5 中文效果好且开源可本地部署；通义 text-embedding-v2 API 调用方便；按成本与效果权衡。

---

## ✅ 任务完成状态

| 项目 | 内容 |
|------|------|
| 是否完成 | ☐ 未完成 |
| 完成时间 | 　　　　 |
| 执行人 | 　　　　 |
| 验收结果 | 　　　　 |
| 备注 | 　　　　 |
