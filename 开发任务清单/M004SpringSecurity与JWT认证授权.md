# M004: SpringSecurity与JWT认证授权

> **所属阶段**：第一阶段 - 基础架构
> **预计周期**：第 2 周
> **优先级**：P0
> **前置任务**：M003

## 一、任务目标
集成 Spring Security + JWT 实现用户认证与授权：完成登录接口签发 Token、Token 刷新机制、RBAC 用户角色权限校验，并将 Token 黑名单存入 Redis 支持登出失效，为所有业务接口提供统一的安全基座。

## 二、前置条件
- M003 已完成，sys_user/sys_role/sys_permission 等表已建好
- 默认 admin 用户已存在（密码 BCrypt 加密）
- Redis 可连接

## 三、详细执行步骤
### 步骤 1：引入依赖
在 `intellimes-system/pom.xml` 添加：
- `spring-boot-starter-security`
- `io.jsonwebtoken:jjwt-api/jjwt-impl/jjwt-jackson:0.12.x`
- `intellimes-common`（已含 Redis 配置）

### 步骤 2：编写 SecurityConfig 配置类
在 `com.intellimes.system.config` 创建 `SecurityConfig`：
- `@EnableWebSecurity` + `@EnableMethodSecurity`
- 关闭 CSRF（前后端分离不需要）
- 配置 `sessionCreationPolicy=STATELESS` 无状态
- 放行 `/auth/login`、`/auth/register`、`/swagger-ui/**`、`/v3/api-docs/**`、`/static/**`
- 其他请求 `anyRequest().authenticated()`
- 添加 `JwtAuthenticationFilter` 在 `UsernamePasswordAuthenticationFilter` 之前
- 暴露 `BCryptPasswordEncoder` 与 `AuthenticationManager` Bean

### 步骤 3：编写 JwtUtils 工具类
在 `com.intellimes.system.security` 创建 `JwtUtils`：
- 配置 `secret` 与 `expiration`(默认 2 小时)、`refreshExpiration`(7 天)
- `generateToken(String username, List<String> roles)` 生成访问 Token
- `generateRefreshToken(String username)` 生成刷新 Token
- `parseToken(token)` 解析 Claims
- `validateToken(token)` 校验签名与过期
- `getUsernameFromToken(token)`

### 步骤 4：编写 JwtAuthenticationFilter
- 继承 `OncePerRequestFilter`
- 从 Header `Authorization: Bearer xxx` 提取 Token
- 校验 Token 且不在 Redis 黑名单
- 解析用户名，从 DB 加载用户权限，构造 `UsernamePasswordAuthenticationToken` 放入 `SecurityContextHolder`

### 步骤 5：实现 UserDetailsService
- `CustomUserDetailsService implements UserDetailsService`
- `loadUserByUsername`：查询 sys_user + 关联 sys_role + sys_permission，构造 `LoginUserDetails`（含 userId、username、password、authorities）

### 步骤 6：编写登录/登出/刷新接口
在 `AuthController`：
- `POST /auth/login`：校验用户名密码 → 生成 accessToken 与 refreshToken → 返回 `LoginVO`
- `POST /auth/logout`：将当前 Token 写入 Redis 黑名单 `jwt:blacklist:{token}` 设过期时间为 Token 剩余有效期
- `POST /auth/refresh`：传 refreshToken，校验后签发新的 accessToken
- `GET /auth/info`：获取当前登录用户信息与权限菜单

### 步骤 7：Token 黑名单 Redis 操作
在 `TokenService`：
- `addToBlacklist(token)`：`redisTemplate.opsForValue().set("jwt:blacklist:"+token, "1", remainingTime, SECONDS)`
- `isBlacklisted(token)`：`redisTemplate.hasKey("jwt:blacklist:"+token)`
- 登录时可选 `jwt:user:{userId}` 缓存最新 Token 实现单点登录控制（参考计划书 5.3 Token 缓存场景）

### 步骤 8：接口权限注解
- 在 Controller 方法上用 `@PreAuthorize("@ss.hasPerm('system:user:add')")` 控制按钮权限
- 编写 `PermissionService`（Bean 名 `ss`），`hasPerm(String perm)` 从 SecurityContext 取权限比对

### 步骤 9：登录测试
- Postman `POST /auth/login` body `{"username":"admin","password":"123456"}`
- 返回 token，复制后调用 `GET /auth/info` Header 携带 `Authorization: Bearer xxx`
- 不带 Token 访问 `/wms/material/page` 应返回 401
- 调用 `/auth/logout` 后原 Token 再访问应返回 401（黑名单生效）

## 四、涉及文件清单
| 文件路径 | 说明 |
|---------|------|
| `intellimes-system/src/main/java/com/intellimes/system/config/SecurityConfig.java` | Security 配置 |
| `intellimes-system/src/main/java/com/intellimes/system/security/JwtUtils.java` | JWT 工具 |
| `intellimes-system/src/main/java/com/intellimes/system/security/JwtAuthenticationFilter.java` | Token 过滤器 |
| `intellimes-system/src/main/java/com/intellimes/system/security/CustomUserDetailsService.java` | 用户详情服务 |
| `intellimes-system/src/main/java/com/intellimes/system/security/LoginUserDetails.java` | 登录用户对象 |
| `intellimes-system/src/main/java/com/intellimes/system/service/TokenService.java` | Token 黑名单服务 |
| `intellimes-system/src/main/java/com/intellimes/system/service/AuthService.java` | 认证服务 |
| `intellimes-system/src/main/java/com/intellimes/system/controller/AuthController.java` | 登录/登出/刷新 |
| `intellimes-system/src/main/java/com/intellimes/system/service/PermissionService.java` | 权限校验 Bean |
| `intellimes-system/src/main/java/com/intellimes/system/dto/LoginDTO.java` | 登录入参 |
| `intellimes-system/src/main/java/com/intellimes/system/vo/LoginVO.java` | 登录返回 |

## 五、预期产出
- 完整的 Spring Security + JWT 认证授权链路
- 登录接口签发双 Token（accessToken + refreshToken）
- 登出后 Token 即时失效（Redis 黑名单）
- 基于权限码的接口级与方法级控制
- M006 前端登录页可直接对接

## 六、验证方式
- `POST /auth/login` 正确账密返回 token，错误账密返回 401 + 提示
- 携带 Token 访问受保护接口返回 200，不携带返回 401
- 登出后再用原 Token 访问返回 401（黑名单生效）
- `@PreAuthorize` 注解的方法，无权限用户访问返回 403
- Redis 中能查到 `jwt:blacklist:{token}` 键

## 七、技术要点与注意事项
- **Spring Boot 3 适配**：`spring-boot-starter-security` 已包含，但 `WebSecurityConfigurerAdapter` 已废弃，需用 `SecurityFilterChain` Bean 链式配置。
- **无状态会话**：`SessionCreationPolicy.STATELESS` 不创建 HttpSession，完全依赖 Token，符合前后端分离架构。
- **Token 黑名单（计划书 5.3）**：JWT 本身无状态无法主动失效，登出/修改密码场景必须配合 Redis 黑名单，Key 设为 `jwt:blacklist:{token}` 并设置与 Token 相同的过期时间，避免无限增长。
- **密钥安全**：`jwt.secret` 至少 32 字符，生产环境必须通过环境变量注入，不能硬编码到 yml。
- **BCrypt 加盐**：同一密码每次 BCrypt 结果不同，校验时用 `passwordEncoder.matches(rawPwd, encodedPwd)`。
- **权限注解**：`@EnableMethodSecurity` 开启后 `@PreAuthorize` 才生效；自定义 Bean 用 `@PreAuthorize("@ss.hasPerm('xxx')")` 调用，注意 Bean 名与方法名匹配。
- **Token 续期**：refreshToken 设计避免用户频繁登录，但 refreshToken 也应存 Redis 便于失效控制。

---

## ✅ 任务完成状态

| 项目 | 内容 |
|------|------|
| 是否完成 | ☐ 未完成 |
| 完成时间 | 　　　　 |
| 执行人 | 　　　　 |
| 验收结果 | 　　　　 |
| 备注 | 　　　　 |
