# M017: 工单与WMS联动领料出库

> **所属阶段**：第三阶段 - MES 生产模块
> **预计周期**：第 6 周
> **优先级**：P0
> **前置任务**：M016

## 一、任务目标
实现工单领料申请→自动生成 WMS 出库单的联动流程：工单派工后工人发起领料申请，系统根据 BOM（产品物料清单）自动生成出库单，库存扣减与报工解耦（参考计划书 5.1 inventory.deduct 场景），完成 MES 工单与 WMS 出库的数据贯通。

## 二、前置条件
- M016 已完成，MES-WMS 消息链路可用
- M009 出库单流程可用
- M014 工单管理可用
- M007 物料管理可用

## 三、详细执行步骤
### 步骤 1：设计 BOM 物料清单表
创建 `bom` 表：
- id、productMaterialId（成品物料ID）、componentMaterialId（组件物料ID）、quantity（单耗）、unit
- 示例：电机A（M003）需要 螺丝 M001 × 10、钢板 M002 × 2

### 步骤 2：编写 BOM 管理 CRUD
- `entity/Bom`、`BomMapper`、`BomService`、`BomController`
- `POST /mes/bom` 维护 BOM
- `GET /mes/bom/listByProduct/{productMaterialId}` 查询成品的 BOM 列表

### 步骤 3：编写领料申请实体与服务
- `entity/MaterialRequisition`（orderId、status(0待审核1已审核2已出库3已取消)、totalQuantity）extends BaseEntity
- `entity/MaterialRequisitionDetail`（requisitionId、materialId、quantity、locationId）
- `MaterialRequisitionService`：
  - `apply(RequisitionDTO)`：根据工单产品查 BOM，按工单数量 × 单耗自动计算各组件领料数量，生成领料申请单 status=0
  - `audit(id)`：0→1 已审核，发送消息到 `wms.topic` routing key `inventory.deduct`
  - `complete(id)`：1→2 已出库，WMS 出库完成后回调更新

### 步骤 4：定义领料消息体
`MaterialDeductMessage`：
- requisitionId、orderId、仓库ID、明细列表（materialId、locationId、quantity）

### 步骤 5：发送领料消息到 wms.topic
领料审核后调用：
- `MQProducer.send(MQConstants.WMS_TOPIC_EXCHANGE, "inventory.deduct", message)`

### 步骤 6：编写 WMS 领料消费者
在 `intellimes-wms` 创建 `MaterialDeductConsumer`：
- 监听 `wms.inventory.deduct.queue`
- 接收消息后自动创建出库单（source_type=ORDER, source_id=工单ID）
- 自动审核（锁定库存）+ 完成（扣减库存）
- 写 inventory_log change_type=2，source_type=ORDER
- 回调通知 MES 领料完成（可选，发消息到 mes.topic routing key requisition.completed）

### 步骤 7：编写领料 Controller
`MaterialRequisitionController`：
- `POST /mes/requisition/apply` 申请领料（传 orderId，自动算 BOM）
- `PUT /mes/requisition/audit/{id}` 审核
- `GET /mes/requisition/listByOrder/{orderId}` 按工单查询领料记录
- `POST /mes/requisition/page` 分页

### 步骤 8：编写前端页面
- `src/views/mes/requisition/index.vue`：领料申请列表
- `src/views/mes/requisition/apply.vue`：申请页面
  - 选择工单（下拉显示生产中工单）
  - 自动展示 BOM 计算结果（物料、单耗、需领数量，可手动调整）
  - 选择仓库（默认原材料仓）
  - 提交申请
- 工单详情页（M014 detail.vue）增加"领料记录"tab

### 步骤 9：联调验证
- 工单 PO20260801001 产品电机A 数量 100
- BOM：电机A 需要 螺丝 × 10、钢板 × 2
- 领料申请：自动计算 螺丝 1000、钢板 200
- 审核：消息发送到 wms.topic
- WMS 消费：自动创建出库单，扣减库存
- 查 inventory：螺丝减少 1000，钢板减少 200
- 查 inventory_log：source_type=ORDER 记录

### 步骤 10：库存不足场景
- 故意将螺丝库存设为 500（需 1000）
- 领料审核后 WMS 出库单创建，但完成时库存不足
- 出库单状态停留在已审核（锁定 500），异常重试或人工干预

## 四、涉及文件清单
| 文件路径 | 说明 |
|---------|------|
| `intellimes-mes/src/main/java/com/intellimes/mes/entity/Bom.java` | BOM 实体 |
| `intellimes-mes/src/main/java/com/intellimes/mes/controller/BomController.java` | BOM 控制器 |
| `intellimes-mes/src/main/java/com/intellimes/mes/entity/MaterialRequisition.java` | 领料申请实体 |
| `intellimes-mes/src/main/java/com/intellimes/mes/service/MaterialRequisitionService.java` | 领料服务 |
| `intellimes-mes/src/main/java/com/intellimes/mes/controller/MaterialRequisitionController.java` | 领料控制器 |
| `intellimes-common/src/main/java/com/intellimes/common/mq/dto/MaterialDeductMessage.java` | 消息体 |
| `intellimes-wms/src/main/java/com/intellimes/wms/mq/MaterialDeductConsumer.java` | WMS 消费者 |
| `intellimes-frontend/src/api/mes/requisition.ts` | 领料 API |
| `intellimes-frontend/src/views/mes/requisition/index.vue` | 领料列表 |
| `intellimes-frontend/src/views/mes/requisition/apply.vue` | 领料申请页 |

## 五、预期产出
- BOM 物料清单管理
- 工单领料申请（基于 BOM 自动计算）
- 领料审核→消息驱动 WMS 自动出库
- MES 工单与 WMS 出库数据贯通
- 简历亮点："库存扣减与报工解耦，RabbitMQ 异步扣减库存"

## 六、验证方式
- 领料申请自动按 BOM 计算组件数量
- 审核后 WMS 出库单自动生成并扣减库存
- inventory_log 有 source_type=ORDER 记录
- 库存不足时出库单停留在已审核状态
- 工单详情页可查看领料记录
- 消息驱动链路：MES 发送 → WMS 消费 → 库存扣减

## 七、技术要点与注意事项
- **BOM 是联动核心**：BOM（Bill of Materials）定义产品与组件的物料关系，是工单领料数量的计算依据，单耗 × 工单数量 = 领料数量。
- **消息驱动解耦（计划书 5.1）**：领料审核后发消息到 `wms.topic` routing key `inventory.deduct`，WMS 异步扣减库存，MES 不阻塞等待，提升响应速度。库存扣减与报工解耦，避免报工时同步扣减库存影响报工体验。
- **库存不足处理**：WMS 消费时发现库存不足，出库单停留在已审核（已锁定可用部分），通过死信队列或人工干预处理，避免消息无限重试。
- **幂等性**：领料申请 ID 作为 source_id 唯一标识，WMS 消费前查是否已生成出库单，防止重复消费。
- **BOM 维护成本**：实际生产 BOM 复杂多变，本项目简化为单层 BOM（成品→组件），多层 BOM 需递归展开。
- **领料数量调整**：BOM 计算的数量允许工人手动调整（如损耗多领 10%），但需记录调整原因便于审计。
- **回调通知**：WMS 出库完成后可选发消息到 `mes.topic` routing key `requisition.completed` 通知 MES 更新领料状态，形成完整闭环。

---

## ✅ 任务完成状态

| 项目 | 内容 |
|------|------|
| 是否完成 | ☐ 未完成 |
| 完成时间 | 　　　　 |
| 执行人 | 　　　　 |
| 验收结果 | 　　　　 |
| 备注 | 　　　　 |
