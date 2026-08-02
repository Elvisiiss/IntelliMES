# M023: SSE流式响应与AI助手界面

> **所属阶段**：第四阶段 - AI 知识库问答
> **预计周期**：第 7 周
> **优先级**：P0
> **前置任务**：M022

## 一、任务目标
实现后端 SSE（Server-Sent Events）流式响应接口，让大模型回答逐字输出，提升用户体验；前端开发 AI 问答助手侧边栏对话框，支持实时显示流式回答、停止生成、重新生成，对标 ChatGPT 交互体验。

## 二、前置条件
- M022 已完成，RAG 问答可用（非流式）
- M019 ChatService 已集成 Spring AI ChatClient
- 前端工程可运行

## 三、详细执行步骤
### 步骤 1：编写后端 SSE 流式接口
在 `ChatController` 添加：
```java
@PostMapping(value = "/ai/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter chatStream(@RequestBody ChatRequest req) {
    SseEmitter emitter = new SseEmitter(120000L); // 2分钟超时
    // 异步调用 ChatClient 流式 API
    chatService.chatStream(req.getMessage(), emitter);
    return emitter;
}
```

### 步骤 2：改造 ChatService 支持流式
```java
public void chatStream(String message, SseEmitter emitter) {
    chatClient.prompt()
        .user(message)
        .stream()
        .content()
        .subscribe(
            chunk -> {
                try {
                    emitter.send(SseEmitter.event().data(chunk));
                } catch (IOException e) {
                    emitter.completeWithError(e);
                }
            },
            error -> emitter.completeWithError(error),
            () -> emitter.complete()
        );
}
```

### 步骤 3：编写 RAG 流式问答
在 `RagChatService` 添加 `chatStream(question, emitter)`：
- 先同步检索知识库上下文（HybridSearchService）
- 构造 RAG Prompt
- 调用 ChatClient 流式 API 输出
- 起始事件发送引用来源（检索到的文档列表）

### 步骤 4：处理 SSE 连接管理
- 超时设置：`SseEmitter(120000L)` 2 分钟
- 异常处理：`onError`、`onTimeout`、`onCompletion` 回调
- 客户端断开：检测并停止生成，避免浪费 token
- 并发控制：可选用 `Thread.sleep` 模拟逐字效果（如真实流式可用直接转发）

### 步骤 5：编写前端 SSE 客户端
在 `src/utils/sse.ts` 封装 EventSource 或 fetch + ReadableStream：
```typescript
export function chatStream(message: string, onMessage: (chunk: string) => void, onDone: () => void) {
    const eventSource = new EventSource(`/api/ai/chat/stream?message=${encodeURIComponent(message)}`);
    eventSource.onmessage = (e) => onMessage(e.data);
    eventSource.onerror = () => { eventSource.close(); onDone(); };
    return eventSource; // 用于停止
}
```
或用 fetch + ReadableStream（支持 POST body）：
```typescript
const response = await fetch('/api/ai/chat/stream', {method:'POST', body: JSON.stringify({message})});
const reader = response.body.getReader();
// 循环读取 chunk
```

### 步骤 6：编写 AI 助手侧边栏组件
在 `src/components/AiAssistant/`：
- `index.vue`：侧边栏容器（右侧抽屉，可折叠）
- `ChatWindow.vue`：对话窗口
  - 消息列表（用户消息右对齐，AI 消息左对齐，支持 Markdown 渲染）
  - 输入框 + 发送按钮
  - 停止生成按钮（流式中显示）
  - 引用来源展示（RAG 检索的文档列表）
- `MessageItem.vue`：单条消息组件（支持 Markdown、代码高亮）

### 步骤 7：集成 Markdown 渲染
- 安装 `markdown-it` 与 `highlight.js`
- AI 回答支持 Markdown 格式（标题、列表、代码块、表格）
- 代码块语法高亮

### 步骤 8：全局悬浮入口
在 `src/layout/components/Navbar.vue`：
- 添加 AI 助手图标按钮
- 点击打开侧边栏抽屉
- 全局任意页面可呼出 AI 助手

### 步骤 9：联调验证
- 点击导航栏 AI 助手按钮，侧边栏展开
- 输入"什么是 MES"，发送
- AI 回答逐字流式显示
- 点停止生成，立即中断
- RAG 问答显示引用来源
- Markdown 格式正确渲染（代码块高亮）

### 步骤 10：异常处理
- 网络断开：提示"连接中断，请重试"
- 超时：提示"AI 响应超时"
- 空回答：提示"未生成回答，请重新提问"

## 四、涉及文件清单
| 文件路径 | 说明 |
|---------|------|
| `intellimes-ai/src/main/java/com/intellimes/ai/controller/ChatController.java` | 添加流式接口 |
| `intellimes-ai/src/main/java/com/intellimes/ai/service/ChatService.java` | 添加 chatStream |
| `intellimes-ai/src/main/java/com/intellimes/ai/service/RagChatService.java` | 添加 RAG 流式 |
| `intellimes-frontend/src/utils/sse.ts` | SSE 客户端封装 |
| `intellimes-frontend/src/components/AiAssistant/index.vue` | 侧边栏容器 |
| `intellimes-frontend/src/components/AiAssistant/ChatWindow.vue` | 对话窗口 |
| `intellimes-frontend/src/components/AiAssistant/MessageItem.vue` | 消息组件 |
| `intellimes-frontend/src/layout/components/Navbar.vue` | 添加 AI 入口 |

## 五、预期产出
- 后端 SSE 流式响应接口
- 前端 AI 助手侧边栏（全局可呼出）
- 流式逐字输出体验（对标 ChatGPT）
- Markdown 渲染与代码高亮
- 停止生成功能
- RAG 问答显示引用来源

## 六、验证方式
- 发送消息后 AI 回答逐字显示（非一次性返回）
- 停止生成按钮可中断流式
- Markdown 格式正确（列表、代码块、表格）
- 代码块语法高亮
- RAG 问答底部显示引用文档
- 侧边栏可折叠，全局任意页面可用
- 网络断开有友好提示

## 七、技术要点与注意事项
- **SSE vs WebSocket**：SSE 单向（服务器→客户端）适合流式输出，实现简单（HTTP 协议）；WebSocket 双向但复杂。AI 流式回答只需服务器推送，SSE 是最佳选择。
- **SseEmitter 超时**：默认 30s，大模型回答可能较长，需设 120s+；超时后连接断开，前端提示重试。
- **异步订阅**：Spring AI 的 `stream().content()` 返回 Flux，用 `subscribe` 异步消费 chunk，避免阻塞 Servlet 线程。
- **客户端断开检测**：SseEmitter 的 `onCompletion`/`onTimeout` 回调可感知客户端断开，应取消 Flux 订阅停止生成，避免浪费 token。
- **fetch vs EventSource**：EventSource 只支持 GET，且不能传 body；POST + fetch + ReadableStream 更灵活，推荐用 fetch 方案。
- **Markdown 渲染**：用 `markdown-it` 解析 Markdown，`highlight.js` 代码高亮；流式输出时需增量渲染（每次 chunk 追加后重新解析当前完整回答）。
- **引用来源透明**：RAG 问答应展示引用的文档片段，让用户验证回答来源，增强可信度，也是 RAG 系统的核心价值。
- **XSS 防护**：渲染 Markdown 前过滤 XSS（用 DOMPurify），避免恶意脚本注入。

---

## ✅ 任务完成状态

| 项目 | 内容 |
|------|------|
| 是否完成 | ☐ 未完成 |
| 完成时间 | 　　　　 |
| 执行人 | 　　　　 |
| 验收结果 | 　　　　 |
| 备注 | 　　　　 |
