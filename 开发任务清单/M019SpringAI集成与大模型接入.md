# M019: SpringAI集成与大模型接入

> **所属阶段**：第四阶段 - AI 知识库问答
> **预计周期**：第 7 周
> **优先级**：P0
> **前置任务**：M018

## 一、任务目标
集成 Spring AI 框架，对接 DeepSeek 或通义千问大模型 API，实现基础对话接口（非流式），为后续知识库 RAG 问答、SSE 流式响应奠定 AI 能力基础。

## 二、前置条件
- M005 已完成，application.yml 可配置
- 已申请大模型 API Key（DeepSeek 或阿里云通义千问）
- M001 中 intellimes-ai 模块已引入 spring-ai 依赖

## 三、详细执行步骤
### 步骤 1：引入 Spring AI 依赖
在 `intellimes-ai/pom.xml` 添加：
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```
DeepSeek 兼容 OpenAI 协议，可直接用 openai starter 配置 base-url 指向 DeepSeek。

### 步骤 2：配置大模型连接
在 `application-dev.yml`：
```yaml
spring:
  ai:
    openai:
      api-key: ${DEEPSEEK_API_KEY:sk-xxx}
      base-url: https://api.deepseek.com
      chat:
        options:
          model: deepseek-chat
          temperature: 0.7
          max-tokens: 2000
```
通义千问方案：
```yaml
spring:
  ai:
    openai:
      api-key: ${DASHSCOPE_API_KEY:sk-xxx}
      base-url: https://dashscope.aliyuncs.com/compatible-mode/v1
      chat:
        options:
          model: qwen-plus
```

### 步骤 3：编写 ChatService 基础对话
在 `com.intellimes.ai.service`：
```java
@Service
public class ChatService {
    private final ChatClient chatClient;

    public ChatService(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("你是 IntelliMES 智能制造助手的 AI 助理，专注于 MES/WMS/AI 知识库领域问题解答。")
            .build();
    }

    public String chat(String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .call()
            .content();
    }
}
```

### 步骤 4：编写对话 Controller
`ChatController`：
- `POST /ai/chat` body `{"message":"xxx"}` 返回 `{"answer":"xxx"}`
- `POST /ai/chat/withHistory` 带历史对话（M024 实现）

### 步骤 5：编写 Prompt 模板
在 `src/main/resources/prompts/` 创建：
- `system.st` 系统提示词（定义助手角色、回答范围、语言风格）
- `rag-prompt.st` RAG 检索增强提示模板（M022 使用）

### 步骤 6：配置 ChatClient 参数
- `temperature` 0.7 平衡创造性与准确性
- `maxTokens` 2000 控制响应长度
- 可选配置 `topP`、`frequencyPenalty` 等

### 步骤 7：编写异常处理
- API Key 无效：捕获并返回友好提示
- 调用超时：设 30s 超时，超时返回"AI 服务繁忙"
- 限流：429 状态码返回"请求过于频繁"

### 步骤 8：联调验证
- Postman `POST /ai/chat` body `{"message":"什么是 MES 系统？"}`
- 返回大模型生成的回答
- 测试中文、英文、专业问题
- 验证 System Prompt 生效（回答聚焦智能制造领域）

### 步骤 9：成本与限流控制
- 记录每次调用的 token 用量到日志
- 可选：Redis 限流（每用户每分钟 10 次）
- 监控 API 调用费用

## 四、涉及文件清单
| 文件路径 | 说明 |
|---------|------|
| `intellimes-ai/pom.xml` | 引入 spring-ai 依赖 |
| `intellimes-ai/src/main/java/com/intellimes/ai/service/ChatService.java` | 对话服务 |
| `intellimes-ai/src/main/java/com/intellimes/ai/controller/ChatController.java` | 对话控制器 |
| `intellimes-ai/src/main/java/com/intellimes/ai/config/AIConfig.java` | ChatClient 配置 |
| `intellimes-ai/src/main/resources/prompts/system.st` | 系统提示词 |
| `intellimes-ai/src/main/resources/prompts/rag-prompt.st` | RAG 提示模板 |
| `intellimes-system/src/main/resources/application-dev.yml` | 大模型连接配置 |

## 五、预期产出
- Spring AI 框架集成完成
- 大模型 API（DeepSeek/通义）对接成功
- 基础对话接口 `/ai/chat` 可用
- System Prompt 定义助手角色
- 为 RAG 知识库问答与 SSE 流式响应奠定基础

## 六、验证方式
- `POST /ai/chat` 返回大模型回答
- 中文问题回答流畅
- System Prompt 生效（回答聚焦智能制造）
- API Key 错误时返回友好提示而非堆栈
- 日志记录 token 用量

## 七、技术要点与注意事项
- **Spring AI 版本**：Spring AI 仍在快速迭代，1.0.0-M1 与 Spring Boot 3.2.x 兼容，需关注版本匹配；API 可能有变化，参考官方文档。
- **DeepSeek 兼容 OpenAI 协议**：DeepSeek API 完全兼容 OpenAI 接口规范，用 `spring-ai-openai-spring-boot-starter` 配 `base-url` 即可，无需单独 SDK，降低学习成本。
- **API Key 安全**：通过环境变量 `${DEEPSEEK_API_KEY}` 注入，不硬编码到 yml；生产环境用配置中心或 K8s Secret 管理。
- **System Prompt 设计**：明确定义助手角色（智能制造 AI 助理）、回答范围（MES/WMS/AI 知识库）、语言风格（专业简洁）、边界（不回答无关问题），提升回答质量。
- **temperature 参数**：知识库问答建议 0.3-0.5（准确优先），创意对话 0.7-1.0（多样优先），RAG 场景用低温度。
- **token 成本**：大模型按 token 计费，maxTokens 控制单次响应上限；System Prompt 与历史对话都会消耗 input token，需控制上下文长度。
- **超时与重试**：大模型响应较慢（几秒到几十秒），HTTP 客户端超时设 30s；网络抖动可重试但避免无限重试放大费用。

---

## ✅ 任务完成状态

| 项目 | 内容 |
|------|------|
| 是否完成 | ☐ 未完成 |
| 完成时间 | 　　　　 |
| 执行人 | 　　　　 |
| 验收结果 | 　　　　 |
| 备注 | 　　　　 |
