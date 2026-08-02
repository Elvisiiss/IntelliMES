# M022: 混合检索BM25向量RRF融合

> **所属阶段**：第四阶段 - AI 知识库问答
> **预计周期**：第 7 周
> **优先级**：P0
> **前置任务**：M021

## 一、任务目标
实现混合检索：BM25 全文检索（关键词匹配）+ 向量检索（语义匹配），通过 RRF（Reciprocal Rank Fusion）算法融合排序，提升知识库召回准确率，避免单一检索的局限（参考计划书 5.2 + 简历亮点 5），为 RAG 提供高质量上下文。

## 二、前置条件
- M021 已完成，向量检索可用，ES knowledge_base 索引有数据
- knowledge_base 索引 content 字段为 text 类型支持 BM25
- 已了解 RRF 算法原理

## 三、详细执行步骤
### 步骤 1：编写 BM25 全文检索服务
`BM25SearchService`：
- `search(String query, int topK) -> List<SearchResult>`
- ES 查询：
```json
{
  "query": {
    "match": {
      "content_chunk": "查询关键词"
    }
  },
  "size": 5
}
```
- 返回带 `_score` 的结果列表

### 步骤 2：编写 RRF 融合算法
`RRFMerger`：
- `merge(List<SearchResult> bm25Results, List<SearchResult> vectorResults, int topK) -> List<SearchResult>`
- RRF 公式：`score(d) = Σ 1/(k + rank_i(d))`，k 通常取 60
- 对每个文档，BM25 排名 rank1 + 向量排名 rank2，计算 `1/(60+rank1) + 1/(60+rank2)`
- 按融合分数降序，取 topK

示例：
```
文档A：BM25 排名 1，向量排名 3 → 1/61 + 1/63 = 0.0164 + 0.0159 = 0.0323
文档B：BM25 排名 2，向量排名 2 → 1/62 + 1/62 = 0.0161 + 0.0161 = 0.0322
文档C：BM25 排名 3，向量排名 1 → 1/63 + 1/61 = 0.0159 + 0.0164 = 0.0323
```

### 步骤 3：编写混合检索服务
`HybridSearchService`：
- `search(String query, int topK) -> List<SearchResult>`
- 并行调用 BM25SearchService 与 VectorSearchService（各取 topK * 2 候选）
- 调用 RRFMerger 融合排序，取 topK 返回
- 可配置权重：`bm25Weight * rrfScore + vectorWeight * rrfScore`

### 步骤 4：编写 RAG 提示词模板
在 `src/main/resources/prompts/rag-prompt.st`：
```
你是 IntelliMES 智能制造助手。请根据以下知识库上下文回答用户问题。
如果上下文中没有相关信息，请说明"知识库中未找到相关信息"，不要编造。

【知识库上下文】
{context}

【用户问题】
{question}

【回答】
```

### 步骤 5：编写 RAG 问答服务
`RagChatService`：
- `chat(String question) -> String`
- 调用 HybridSearchService 检索 topK=3 相关分块
- 拼接上下文 context = 分块1 + 分块2 + 分块3
- 用 PromptTemplate 填充 rag-prompt.st
- 调用 ChatService 生成回答
- 返回回答（M023 改为流式）

### 步骤 6：编写混合检索接口
`HybridSearchController`：
- `POST /ai/search/hybrid` body `{"query":"xxx","topK":5}` 返回融合结果
- `POST /ai/rag/chat` body `{"question":"xxx"}` 返回 RAG 回答（非流式，M023 改流式）

### 步骤 7：编写检索对比测试
- 准备测试问题集（10 个智能制造相关问题）
- 分别用 BM25、向量、混合检索
- 人工评估召回准确率，输出对比报告
- 验证 RRF 融合优于单一检索

### 步骤 8：前端检索测试页增强
在 `src/views/ai/search-test.vue`：
- 增加检索方式切换：BM25 / 向量 / 混合
- 展示融合分数与各检索原始排名
- 直观对比三种检索效果

### 步骤 9：联调验证
- 上传多份文档（MES 概念、WMS 流程、设备维护手册）
- 索引后测试：
  - 查询"什么是制造执行系统"：BM25 命中关键词，向量命中语义
  - 查询"如何盘点库存"：向量理解"盘点"="清点"，BM25 可能漏
  - 混合检索结果综合两者优势
- 调用 `/ai/rag/chat` 问"工单完工后会发生什么"，回答引用知识库上下文

## 四、涉及文件清单
| 文件路径 | 说明 |
|---------|------|
| `intellimes-ai/src/main/java/com/intellimes/ai/service/BM25SearchService.java` | BM25 检索服务 |
| `intellimes-ai/src/main/java/com/intellimes/ai/service/RRFMerger.java` | RRF 融合算法 |
| `intellimes-ai/src/main/java/com/intellimes/ai/service/HybridSearchService.java` | 混合检索服务 |
| `intellimes-ai/src/main/java/com/intellimes/ai/service/RagChatService.java` | RAG 问答服务 |
| `intellimes-ai/src/main/java/com/intellimes/ai/controller/HybridSearchController.java` | 混合检索控制器 |
| `intellimes-ai/src/main/resources/prompts/rag-prompt.st` | RAG 提示模板 |
| `intellimes-frontend/src/views/ai/search-test.vue` | 检索测试页（增强） |

## 五、预期产出
- BM25 + 向量 + RRF 混合检索完整实现
- RAG 问答接口（基于知识库上下文回答）
- 检索方式可切换对比
- 简历亮点："BM25 全文检索 + 向量检索 + RRF 融合排序，提升知识召回准确率"

## 六、验证方式
- BM25 检索返回关键词匹配结果
- 向量检索返回语义匹配结果
- 混合检索融合两者，排序更合理
- RAG 回答引用知识库上下文，未命中时如实说明
- 对比报告显示混合检索准确率高于单一检索
- 测试问题召回率提升明显

## 七、技术要点与注意事项
- **BM25 vs 向量检索互补**：BM25 擅长关键词精确匹配（如物料编码、工单号），向量擅长语义理解（如"盘点"="清点"）；混合检索综合两者优势，是 RAG 最佳实践（参考计划书 5.2）。
- **RRF 算法优势**：无需归一化不同检索的分数（BM25 与向量分数尺度不同），仅用排名融合，简单有效；k=60 是经验值，平衡头部与尾部文档权重。
- **候选数扩展**：BM25 与向量各取 topK*2 候选再融合，避免单一检索漏掉的相关文档在融合时丢失。
- **RAG Prompt 设计**：明确告知大模型"基于上下文回答，无信息时说明"，避免幻觉（编造）；上下文按相关度排序拼接，最相关的放前面。
- **上下文长度控制**：topK=3，每个分块 500 字，上下文约 1500 字，加上问题与系统提示不超过模型 context window；过长会稀释重点并增加费用。
- **检索质量评估**：准备标注好的测试集（问题-期望文档），用召回率（Recall@K）与准确率（Precision@K）量化评估，避免主观判断。
- **权重调优**：可根据场景调整 BM25 与向量权重，如专业术语场景提高 BM25 权重，自然语言场景提高向量权重。

---

## ✅ 任务完成状态

| 项目 | 内容 |
|------|------|
| 是否完成 | ☐ 未完成 |
| 完成时间 | 　　　　 |
| 执行人 | 　　　　 |
| 验收结果 | 　　　　 |
| 备注 | 　　　　 |
