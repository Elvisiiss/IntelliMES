# M011: Redis分布式锁库存扣减

> **所属阶段**：第二阶段 - WMS 仓储模块
> **预计周期**：第 4 周
> **优先级**：P0
> **前置任务**：M010

## 一、任务目标
使用 Redisson RLock 分布式锁对库存扣减（出库完成、入库增加）进行并发控制，配合 inventory 表 version 乐观锁形成双重保障，彻底防止高并发场景下的库存超卖问题，并编写并发测试用例验证线程安全（参考计划书 4.2 version 字段 + 5.3 Redis 分布式锁场景）。

## 二、前置条件
- M005 已完成 RedissonDistributedLock 工具类
- M009 已完成出库扣减逻辑（已有乐观锁）
- M010 库存查询可用，便于观察结果

## 三、详细执行步骤
### 步骤 1：分析超卖风险场景
梳理超卖场景：
- 多个出库单同时完成，并发扣减同一货位库存
- 出库完成与移库操作并发
- 工单领料（M017）与手动出库并发
仅靠 DB 乐观锁：高并发下大量 update 失败重试，性能差且用户体验差。引入分布式锁串行化扣减。

### 步骤 2：定义库存锁 Key 规则
在 `InventoryService` 中定义锁 Key：
- `intellimes:lock:inventory:{materialId}:{locationId}`
- 粒度到物料+货位，不同货位可并行扣减，提升并发度

### 步骤 3：改造 InventoryService.deduct 方法
```java
public void deduct(Long materialId, Long locationId, BigDecimal quantity) {
    String lockKey = "intellimes:lock:inventory:" + materialId + ":" + locationId;
    distributedLock.executeWithLock(lockKey, 5, 10, TimeUnit.SECONDS, () -> {
        // 1. 查询当前库存
        Inventory inv = inventoryMapper.selectByMaterialAndLocation(materialId, locationId);
        BigDecimal available = inv.getQuantity().subtract(inv.getLockedQuantity());
        if (available.compareTo(quantity) < 0) {
            throw new BusinessException("库存不足，可用：" + available);
        }
        // 2. 乐观锁扣减（双重保障）
        int rows = inventoryMapper.deductWithVersion(inv.getId(), quantity, inv.getVersion());
        if (rows == 0) {
            throw new BusinessException("库存扣减失败，请重试");
        }
        // 3. 写流水
        inventoryLogService.save(buildLog(inv, quantity, 2));
    });
}
```

### 步骤 4：改造 lock / unlock / increase 方法
同样用分布式锁包裹：
- `lock`：锁内查询 available，校验，UPDATE locked_quantity
- `unlock`：锁内 UPDATE locked_quantity -= ?
- `increase`：锁内 upsert + 写流水

### 步骤 5：编写并发测试用例
在 `src/test/java/com/intellimes/wms/InventoryConcurrencyTest.java`：
- 用 `CountDownLatch` + 线程池模拟 50 个线程同时扣减同一货位库存
- 库存初始 100，每个线程扣减 3，预期成功 33 个，失败 17 个
- 验证最终库存 quantity = 100 - 33*3 = 1
- 验证 inventory_log 记录数 = 33 条

### 步骤 6：对比测试
- 不加分布式锁（仅乐观锁）跑同样并发用例：观察失败重试次数、耗时
- 加分布式锁跑：观察串行化效果、总耗时
- 输出对比报告：加锁后失败率降低、数据一致性保证

### 步骤 7：编写压测脚本
提供 JMeter 或 ab 脚本：
- 并发 100 个出库完成请求
- 监控 Redis 锁等待时间、DB 慢查询
- 输出 TPS、平均响应时间、错误率

### 步骤 8：添加监控日志
在 `RedissonDistributedLock` 中：
- 记录获取锁等待时间、持锁时间
- 锁获取失败告警日志
- 便于排查死锁与性能瓶颈

## 四、涉及文件清单
| 文件路径 | 说明 |
|---------|------|
| `intellimes-wms/src/main/java/com/intellimes/wms/service/impl/InventoryServiceImpl.java` | 改造为分布式锁+乐观锁双重保障 |
| `intellimes-wms/src/main/java/com/intellimes/wms/mapper/InventoryMapper.java` | 补充 deductWithVersion 方法 |
| `intellimes-wms/src/main/resources/mapper/wms/InventoryMapper.xml` | 乐观锁扣减 SQL |
| `intellimes-wms/src/test/java/com/intellimes/wms/InventoryConcurrencyTest.java` | 并发测试 |
| `intellimes-common/src/main/java/com/intellimes/common/lock/RedissonDistributedLock.java` | 补充监控日志 |
| `intellimes-wms/src/test/resources/jmeter/inventory-stress-test.jmx` | JMeter 压测脚本（可选） |

## 五、预期产出
- 库存扣减/锁定/释放/增加全部加分布式锁
- 分布式锁 + 乐观锁双重保障防超卖
- 并发测试用例验证线程安全
- 对比报告证明加锁方案有效性
- 简历亮点："使用 Redisson 分布式锁解决 WMS 库存超卖问题"

## 六、验证方式
- 50 线程并发扣减测试通过：成功数 + 失败数 = 50，最终库存正确
- inventory_log 流水数 = 成功扣减数
- 关闭分布式锁（注释代码）跑同样用例：出现超卖（库存负数）或大量失败
- 压测 100 并发无死锁、无数据不一致
- Redis 中可观察到 `intellimes:lock:inventory:*` 锁 Key（短暂存在）

## 七、技术要点与注意事项
- **锁粒度选择**：锁 Key 到 `materialId:locationId` 而非全局锁，不同物料/货位可并行，最大化并发度；过粗（全局锁）性能差，过细（无锁）超卖。
- **waitTime 与 leaseTime（计划书 5.3）**：`tryLock(5s, 10s, SECONDS)` 等待 5s 获取锁，持锁最多 10s 自动释放，避免业务异常死锁。leaseTime 必须大于业务最长执行时间。
- **双重保障必要性**：分布式锁防并发，乐观锁兜底（防止锁失效或主从切换丢锁场景）。任一失效另一保障数据安全。
- **锁与事务顺序**：先获取分布式锁，再开启事务，事务提交后再释放锁。顺序反了会导致锁释放后事务未提交，其他线程读到旧数据。
- **锁超时降级**：获取锁超时返回业务异常"系统繁忙请重试"，而非无限等待阻塞线程。
- **看门狗慎用**：Redisson 默认 30s 看门狗续期，业务超长时锁可能被续到很久，建议显式设 leaseTime 更可控。
- **流水一致性**：扣减与流水在同一事务内，分布式锁释放前事务已提交，保证流水与库存一致。

---

## ✅ 任务完成状态

| 项目 | 内容 |
|------|------|
| 是否完成 | ☐ 未完成 |
| 完成时间 | 　　　　 |
| 执行人 | 　　　　 |
| 验收结果 | 　　　　 |
| 备注 | 　　　　 |
