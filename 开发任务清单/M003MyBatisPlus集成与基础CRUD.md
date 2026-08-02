# M003: MyBatisPlus集成与基础CRUD

> **所属阶段**：第一阶段 - 基础架构
> **预计周期**：第 1-2 周
> **优先级**：P0
> **前置任务**：M002

## 一、任务目标
完成 MyBatis-Plus 框架集成（分页插件、乐观锁插件、自动填充），封装 `BaseEntity` 公共字段、统一返回结果 `Result<T>`、全局异常处理器 `GlobalExceptionHandler`，并针对 `material` 物料表完成一套示例 CRUD 接口，验证整个数据访问链路可用。

## 二、前置条件
- M002 已完成，数据库表已创建
- M001 工程骨架可编译

## 三、详细执行步骤
### 步骤 1：引入 MyBatis-Plus 依赖
在 `intellimes-common/pom.xml` 中引入：
- `com.baomidou:mybatis-plus-spring-boot3-starter:3.5.5`
- `com.baomidou:mybatis-plus-jsqlparser:3.5.5`（分页插件依赖）
- MySQL 驱动 `mysql-connector-j`

### 步骤 2：编写 MyBatisPlusConfig 配置类
在 `intellimes-common` 的 `com.intellimes.common.config` 下创建 `MyBatisPlusConfig`：
- `@Configuration` + `@MapperScan("com.intellimes.**.mapper")`
- 注册 `MybatisPlusInterceptor` Bean
- 添加 `PaginationInnerInterceptor(DbType.MYSQL)` 分页插件
- 添加 `OptimisticLockerInnerInterceptor` 乐观锁插件

### 步骤 3：编写 BaseEntity 公共实体
在 `com.intellimes.common.entity` 创建 `BaseEntity`：
- 字段：`Long id`（`@TableId(type=IdType.ASSIGN_ID)` 雪花 ID）
- `LocalDateTime createTime`（`@TableField(fill=FieldFill.INSERT)` + `@JsonFormat`）
- `LocalDateTime updateTime`（`@TableField(fill=FieldFill.INSERT_UPDATE)`）
- `Long createBy`
- `Integer deleted`（`@TableLogic` 逻辑删除，`@TableField(select=false)` 默认不查询）

### 步骤 4：编写 MetaObjectHandler 自动填充
创建 `MyMetaObjectHandler implements MetaObjectHandler`：
- `insertFill`：自动填充 createTime、updateTime、createBy（从 SecurityContext 获取当前用户）
- `updateFill`：自动填充 updateTime

### 步骤 5：编写统一返回结果 Result
在 `com.intellimes.common.result` 创建：
- `Result<T>`：`Integer code`、`String message`、`T data`、`long total`
- 静态方法 `success(T data)`、`success()`、`fail(String msg)`、`fail(Integer code, String msg)`
- `ResultCode` 枚举：200 成功、401 未授权、403 禁止访问、500 系统错误 等

### 步骤 6：编写全局异常处理器
在 `com.intellimes.common.exception` 创建：
- `BusinessException extends RuntimeException`（自定义业务异常，含 code、message）
- `GlobalExceptionHandler`：
  - `@ExceptionHandler(BusinessException.class)` 返回 `Result.fail`
  - `@ExceptionHandler(NotLoginException.class)` 返回 401
  - `@ExceptionHandler(MethodArgumentNotValidException.class)` 参数校验异常
  - `@ExceptionHandler(Exception.class)` 兜底返回 500 并打印日志

### 步骤 7：编写 Material 实体与 Mapper
在 `intellimes-wms` 中：
- `entity/Material extends BaseEntity`，字段：materialCode、materialName、spec、unit、category、status
- `mapper/MaterialMapper extends BaseMapper<Material>`
- 在 `src/main/resources/mapper/wms/MaterialMapper.xml` 编写自定义查询（按名称模糊分页）

### 步骤 8：编写 Material Service
- `MaterialService extends IService<Material>`
- `MaterialServiceImpl extends ServiceImpl<MaterialMapper, Material>`
- 方法：`pageQuery(MaterialQueryDTO dto)` 分页查询、`add`、`update`、`delete`（逻辑删除）、`getById`

### 步骤 9：编写 MaterialController
- `@RestController` + `@RequestMapping("/wms/material")`
- `POST /page` 分页、`POST /` 新增、`PUT /` 修改、`DELETE /{id}` 删除、`GET /{id}` 详情
- 返回 `Result<Page<MaterialVO>>`、`Result<Void>` 等

### 步骤 10：测试验证
启动应用，用 Postman 调用：
- `POST /wms/material` 新增一条物料
- `GET /wms/material/page?name=螺丝` 分页查询
- `PUT /wms/material` 修改
- `DELETE /wms/material/1` 后再查询确认 deleted=1 不显示

## 四、涉及文件清单
| 文件路径 | 说明 |
|---------|------|
| `intellimes-common/src/main/java/com/intellimes/common/config/MyBatisPlusConfig.java` | MP 配置（分页+乐观锁） |
| `intellimes-common/src/main/java/com/intellimes/common/entity/BaseEntity.java` | 公共实体 |
| `intellimes-common/src/main/java/com/intellimes/common/handler/MyMetaObjectHandler.java` | 自动填充处理器 |
| `intellimes-common/src/main/java/com/intellimes/common/result/Result.java` | 统一返回结果 |
| `intellimes-common/src/main/java/com/intellimes/common/result/ResultCode.java` | 返回码枚举 |
| `intellimes-common/src/main/java/com/intellimes/common/exception/GlobalExceptionHandler.java` | 全局异常处理 |
| `intellimes-common/src/main/java/com/intellimes/common/exception/BusinessException.java` | 业务异常 |
| `intellimes-wms/src/main/java/com/intellimes/wms/entity/Material.java` | 物料实体 |
| `intellimes-wms/src/main/java/com/intellimes/wms/mapper/MaterialMapper.java` | Mapper |
| `intellimes-wms/src/main/java/com/intellimes/wms/service/MaterialService.java` | 服务接口 |
| `intellimes-wms/src/main/java/com/intellimes/wms/service/impl/MaterialServiceImpl.java` | 服务实现 |
| `intellimes-wms/src/main/java/com/intellimes/wms/controller/MaterialController.java` | 控制器 |
| `intellimes-wms/src/main/resources/mapper/wms/MaterialMapper.xml` | 自定义 SQL |

## 五、预期产出
- MyBatis-Plus 完整集成，分页与乐观锁插件生效
- 一套可复用的 BaseEntity、Result、GlobalExceptionHandler 工具
- Material 表的完整 CRUD 接口（增删改查、分页、模糊搜索）
- 后续所有业务表可直接套用此模式快速开发

## 六、验证方式
- Postman 调用 `POST /wms/material` 返回 `Result{code:200}`
- 分页接口返回 `total`、`records`，分页参数生效
- 删除后再查询，记录消失但数据库 `deleted` 字段为 1（逻辑删除生效）
- 故意触发参数校验异常，返回 400 + 友好提示（全局异常生效）
- 控制台不打印 SQL 时配置 `logging.level.com.intellimes.**.mapper=debug` 可见 SQL

## 七、技术要点与注意事项
- **Spring Boot 3 适配**：必须用 `mybatis-plus-spring-boot3-starter`，旧版 `mybatis-plus-boot-boot-starter` 不兼容 Spring Boot 3.x。
- **逻辑删除**：`@TableLogic` 配合 `application.yml` 中 `mybatis-plus.global-config.db-config.logic-delete-field=deleted`、`logic-delete-value=1`、`logic-not-delete-value=0`，所有查询自动过滤已删除记录。
- **乐观锁**：`@Version` 注解的字段在 `updateById` 时会自动 `WHERE version=?` 并 `version+1`，是 M011 库存扣减并发安全的关键，必须验证生效。
- **自动填充**：`MetaObjectHandler` 中获取当前用户需注意线程上下文，未登录场景（如登录接口本身）要做空判断，否则 NPE。
- **雪花 ID**：`IdType.ASSIGN_ID` 生成 19 位 Long，前端 JS 精度会丢失，需配置 Jackson `Long -> String` 序列化（`@JsonSerialize(using=ToStringSerializer.class)`）或全局配置。
- **分页插件**：必须 `PaginationInnerInterceptor(DbType.MYSQL)` 指定数据库类型，否则不同数据库方言分页 SQL 错误。

---

## ✅ 任务完成状态

| 项目 | 内容 |
|------|------|
| 是否完成 | ☐ 未完成 |
| 完成时间 | 　　　　 |
| 执行人 | 　　　　 |
| 验收结果 | 　　　　 |
| 备注 | 　　　　 |
