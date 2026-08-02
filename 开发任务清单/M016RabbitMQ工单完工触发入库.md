# M016: RabbitMQ工单完工触发入库

> **所属阶段**：第三阶段 - MES 生产模块
> **预计周期**：第 6 周
> **优先级**：P0
> **前置任务**：M015

## 一、任务目标
使用 RabbitMQ 实现工单完工后异步触发 WMS 自动创建成品入库单：工单完工时发送消息到 `mes.topic`（routing key `order.completed`），WMS 监听 `mes.order.completed.queue` 后自动创建入库单（成品入库），实现 MES 与 WMS 模块解耦（参考计划书 5.1 RabbitMQ 应用场景表）。

## 二、前置条件
- M015 已完成，工单完工逻辑可用
- M005 已完成，RabbitMQ Exchange `mes.topic` 与 Queue `mes.order.completed.queue` 已声明
- M008 入库单创建逻辑可用

## 三、详细执行步骤
### 步骤 1：定义工单完工消息体
在 `intellimes-common` 创建 `OrderCompletedMessage`：
- orderId、orderNo、productName、productMaterialId、quantity、completedQuantity、workshopId、completedTime
- 实现 Serializable

### 步骤 2：改造工单完工逻辑发送消息
在 `ProductionOrderServiceImpl.complete` 方法：
- 完工状态流转后，构造 `OrderCompletedMessage`
- 调用 `MQProducer.send(MQConstants.MES_TOPIC_EXCHANGE, "order.completed", message)`
- 发送失败时记录日志但不回滚事务（消息最终一致性，可用本地消息表补偿）
- 或使用 RabbitMQ confirm 机制确认发送成功

### 步骤 3：编写 WMS 消息消费者
在 `intellimes-wms` 创建 `OrderCompletedConsumer`：
```java
@RabbitListener(queues = MQConstants.MES_ORDER_COMPLETED_QUEUE)
public void handleOrderCompleted(OrderCompletedMessage msg, Channel channel, Message message) {
    try {
        // 1. 幂等校验：查入库单 source_type=ORDER, source_id=orderId 是否已存在
        // 2. 构造入库单：仓库=成品仓（默认或工单车间对应仓），明细=工单产品物料 completedQuantity 个
        // 3. 调用 InboundOrderService.createOrder + audit + complete 自动完成入库
        // 4. channel.basicAck
    } catch (Exception e) {
        // 5. channel.basicNack，决定是否重入队列
    }
}
```

### 步骤 4：实现幂等性保障
- 入库单 source_type=ORDER, source_id=orderId 唯一索引
- 消费前先查询是否已处理，避免消息重复消费导致重复入库
- 或用 Redis `SETNX intellimes:msg:processed:{msgId}` 标记已处理

### 步骤 5：配置成品仓映射
- 系统配置表或 warehouse 表加 type 字段标识成品仓
- 工单完工时根据 workshopId 找对应车间的成品仓
- 或简化为全局默认成品仓配置

### 步骤 6：编写消息发送日志
- 创建 `mq_message_log` 表记录发送的消息（id, msg_id, exchange, routing_key, content, status, retry_count, create_time）
- 发送时插入，消费成功更新状态，便于补偿重发

### 步骤 7：联调验证
- 创建工单 PO20260801001 产品电机A 数量 100
- 派工、报工至 completedQuantity=100
- 完工：工单 status=2
- 查 RabbitMQ 管理界面 mes.topic 有消息投递
- 查 WMS 入库单表自动生成一条 source_type=ORDER, source_id=工单ID 的入库单
- 查 inventory 成品仓有电机A 库存 100

### 步骤 8：异常场景测试
- 故意停掉 WMS 模块，工单完工后消息进入队列等待
- 启动 WMS 后消费者自动拉取消息处理
- 重复消费测试：手动重新投递消息，幂等校验阻止重复入库

## 四、涉及文件清单
| 文件路径 | 说明 |
|---------|------|
| `intellimes-common/src/main/java/com/intellimes/common/mq/dto/OrderCompletedMessage.java` | 消息体 |
| `intellimes-mes/src/main/java/com/intellimes/mes/service/impl/ProductionOrderServiceImpl.java` | 改造完工发送消息 |
| `intellimes-wms/src/main/java/com/intellimes/wms/mq/OrderCompletedConsumer.java` | WMS 消费者 |
| `intellimes-wms/src/main/java/com/intellimes/wms/service/InboundOrderService.java` | 补充自动入库方法 |
| `intellimes-common/src/main/java/com/intellimes/common/mq/MQProducer.java` | 生产者（确认机制） |
| `intellimes-common/src/main/java/com/intellimes/common/mq/MQConstants.java` | 常量（补充队列名） |

## 五、预期产出
- 工单完工自动触发成品入库，无需人工干预
- MES 与 WMS 通过消息解耦，互不阻塞
- 幂等性保障防重复入库
- 简历亮点："RabbitMQ 实现 MES 与 WMS 模块解耦，工单完工异步触发入库"

## 六、验证方式
- 工单完工后 5 秒内 WMS 入库单自动生成
- 入库单 source_type=ORDER, source_id=工单ID
- 成品仓库存自动增加
- 停掉 WMS 后工单仍可完工（消息入队等待）
- 重复消费不产生重复入库单（幂等生效）
- RabbitMQ 管理界面可见消息消费速率

## 七、技术要点与注意事项
- **Exchange 与 Routing Key（计划书 5.1）**：使用 `mes.topic` Topic Exchange + routing key `order.completed`，便于后续扩展 `order.*` 订阅其他场景（如工单创建通知）。
- **异步解耦价值**：MES 工单完工不需等待 WMS 入库完成，提升响应速度；WMS 故障时消息入队等待，不影响 MES 业务，提升系统健壮性。
- **幂等性必备**：RabbitMQ 默认 at-least-once 投递，消息可能重复，必须用 source_id 唯一索引或 Redis SETNX 保证重复消费不产生副作用。
- **手动 ACK**：消费成功 `basicAck`，失败 `basicNack(deliveryTag, false, true)` 重入队列或死信队列；不能自动 ACK 否则异常时消息丢失。
- **消息发送可靠性**：开启 `publisher-confirm-type: correlated` 确认消息到达 Broker，失败时记录日志并补偿；关键业务可用本地消息表 + 定时补偿。
- **事务边界**：消息发送应在工单完工事务提交后（用 `TransactionSynchronizationManager.registerSynchronization` 的 afterCommit），避免事务回滚但消息已发送。
- **死信队列**：配置 DLX 死信交换机，多次消费失败的消息进入死信队列人工处理，避免无限重试。

---

## ✅ 任务完成状态

| 项目 | 内容 |
|------|------|
| 是否完成 | ☐ 未完成 |
| 完成时间 | 　　　　 |
| 执行人 | 　　　　 |
| 验收结果 | 　　　　 |
| 备注 | 　　　　 |
