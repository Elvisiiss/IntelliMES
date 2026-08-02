# M002: 数据库设计与初始化SQL

> **所属阶段**：第一阶段 - 基础架构
> **预计周期**：第 1 周
> **优先级**：P0
> **前置任务**：M001

## 一、任务目标
完成 IntelliMES 全平台数据库设计，输出完整 DDL（含字段类型、注释、索引）与初始化数据 SQL。覆盖 MES 核心表（production_order、work_report）、WMS 核心表（inventory、inventory_log）、AI 知识库表（knowledge_doc），并补充 RBAC 系统表、仓库库区货位、物料、车间产线工序、设备等基础数据表，最终能一键初始化出一个结构完整的 `intellimes` 数据库。

## 二、前置条件
- M001 已完成，工程骨架存在
- 本机 MySQL 8.0 可连接（root 用户或专用 intellimes 用户）
- 已阅读 IntelliMES.md 第四章「数据库核心表设计」

## 三、详细执行步骤
### 步骤 1：创建 SQL 目录
在启动模块下创建目录 `intellimes-system/src/main/resources/db/`，用于存放所有 SQL 脚本。

### 步骤 2：编写建库脚本 schema.sql
创建 `db/schema.sql`：
- `CREATE DATABASE IF NOT EXISTS intellimes DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;`
- `USE intellimes;`

### 步骤 3：编写 RBAC 系统表 DDL
在 `db/01_system.sql` 中创建：
- `sys_user`（id, username, password, nickname, email, phone, status, dept_id, create_time, update_time, create_by, deleted）索引：uk_username
- `sys_role`（id, role_code, role_name, status, remark, create_time, deleted）
- `sys_permission`（id, parent_id, perm_code, perm_name, type(1菜单2按钮), path, component, icon, sort, status）
- `sys_user_role`（user_id, role_id）联合主键
- `sys_role_permission`（role_id, permission_id）联合主键

### 步骤 4：编写 WMS 基础与库存表 DDL
在 `db/02_wms.sql` 中创建：
- `warehouse`（id, warehouse_code, warehouse_name, address, status）
- `warehouse_area`（id, warehouse_id, area_code, area_name, type）
- `storage_location`（id, area_id, warehouse_id, location_code, location_name, capacity, status）
- `material`（id, material_code, material_name, spec, unit, category, status）
- `inventory`（id, material_id, warehouse_id, location_id, quantity DECIMAL(18,2), locked_quantity DECIMAL(18,2), version INT DEFAULT 0）索引：uk_material_location
- `inventory_log`（id, material_id, change_type, change_quantity, before_quantity, after_quantity, source_type, source_id, create_time, create_by）索引：idx_material_id, idx_create_time

### 步骤 5：编写入库/出库单表 DDL
在 `db/02_wms.sql` 续写：
- `inbound_order`（id, order_no, warehouse_id, source_type, source_id, status(0待审核1已审核2已完成3已取消), total_quantity, create_time, create_by）
- `inbound_order_detail`（id, order_id, material_id, location_id, quantity）
- `outbound_order`（id, order_no, warehouse_id, source_type, source_id, status, total_quantity, create_time, create_by）
- `outbound_order_detail`（id, order_id, material_id, location_id, quantity）
- `move_order`（移库单：id, order_no, from_location_id, to_location_id, status）
- `stocktake_order`（盘点单：id, order_no, warehouse_id, status, diff_quantity）

### 步骤 6：编写 MES 核心表 DDL
在 `db/03_mes.sql` 中创建：
- `workshop`（id, workshop_code, workshop_name, status）
- `production_line`（id, workshop_id, line_code, line_name, status）
- `process`（id, process_code, process_name, sequence, status）
- `production_order`（id, order_no, product_name, product_material_id, quantity, completed_quantity, status(0待派工1生产中2已完工3已关闭), priority, plan_start_time, plan_end_time, workshop_id, line_id, create_by, create_time）索引：uk_order_no
- `work_report`（id, order_id, worker_id, process_id, report_quantity, qualified_quantity, scrap_quantity, report_time, remark）
- `equipment`（id, equipment_code, equipment_name, workshop_id, status(0停机1运行2故障), location）

### 步骤 7：编写 AI 知识库表 DDL
在 `db/04_ai.sql` 中创建：
- `knowledge_doc`（id, title, content LONGTEXT, file_type, file_path, status(0待处理1已索引2失败), es_index_id, create_time, create_by）
- `chat_history`（id, session_id, role(user/assistant), content, create_time）
- `chat_session`（id, session_id, title, user_id, create_time）

### 步骤 8：编写初始化数据 data.sql
在 `db/05_init_data.sql` 中插入：
- 角色：admin、operator、warehouse_keeper
- 权限：菜单树（系统管理、WMS、MES、AI 各模块根菜单与子菜单）
- admin 角色赋全部权限
- 默认用户：admin/123456（密码 BCrypt 加密）、operator/123456、keeper/123456
- 示例仓库：WH01 中心仓
- 示例库区/货位：WH01-A1-A1-01
- 示例物料：M001 螺丝 M6、M002 钢板 1m×1m、M003 成品电机

### 步骤 9：执行 SQL 验证
用 Navicat 或命令行依次执行 schema.sql → 01~04 DDL → 05_init_data.sql，确认全部表创建成功、初始数据可查询。

## 四、涉及文件清单
| 文件路径 | 说明 |
|---------|------|
| `intellimes-system/src/main/resources/db/schema.sql` | 建库脚本 |
| `intellimes-system/src/main/resources/db/01_system.sql` | RBAC 系统表 DDL |
| `intellimes-system/src/main/resources/db/02_wms.sql` | WMS 表 DDL（含库存/出入库/移库/盘点） |
| `intellimes-system/src/main/resources/db/03_mes.sql` | MES 表 DDL |
| `intellimes-system/src/main/resources/db/04_ai.sql` | AI 知识库表 DDL |
| `intellimes-system/src/main/resources/db/05_init_data.sql` | 初始化数据 |

## 五、预期产出
- 一个 utf8mb4 字符集的 `intellimes` 数据库
- 20+ 张表，含完整字段类型、注释、索引
- 3 个默认用户、3 个角色、完整权限菜单树
- 示例仓库/库区/货位/物料数据

## 六、验证方式
- 执行 `SHOW TABLES;` 能列出全部业务表
- `SELECT * FROM sys_user WHERE username='admin';` 返回默认管理员
- `SELECT * FROM material;` 返回 3 条示例物料
- 所有表均含 `create_time`、`update_time`、`deleted` 公共字段（除纯关联表）
- `inventory` 表 `version` 字段默认值为 0

## 七、技术要点与注意事项
- **字符集统一 utf8mb4**：工业场景涉及中文物料名、工序名，utf8mb4 支持 emoji 与生僻字。
- **公共字段**：参考 M003 BaseEntity，所有业务表加 `id(BIGINT AUTO_INCREMENT)`、`create_time`、`update_time`、`create_by`、`deleted(TINYINT 默认 0)`。
- **inventory 乐观锁**：`version INT DEFAULT 0` 字段配合 MyBatis-Plus `@Version` 注解，是 M011 防超卖的双重保障（参考计划书 4.2 + 5.3）。
- **索引设计**：`inventory` 表 `uk_material_location(material_id, location_id)` 唯一索引保证同一物料同一货位只有一条记录；`inventory_log` 按时间和物料建索引便于流水查询。
- **DECIMAL(18,2)**：工业数量可能为小数（如 1.5kg），不用 FLOAT/DOUBLE 避免精度丢失。
- **状态字段 TINYINT**：与 Java 枚举对应，注释里写清每个值含义，避免魔法数字。
- **DDL 顺序**：先建无外键依赖的表（warehouse/material），再建依赖表（warehouse_area/inventory），避免外键报错。

---

## ✅ 任务完成状态

| 项目 | 内容 |
|------|------|
| 是否完成 | ☐ 未完成 |
| 完成时间 | 　　　　 |
| 执行人 | 　　　　 |
| 验收结果 | 　　　　 |
| 备注 | 　　　　 |
