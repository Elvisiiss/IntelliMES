# M027: DockerCompose编排与部署

> **所属阶段**：第五阶段 - 联调测试与部署
> **预计周期**：第 8 周
> **优先级**：P0
> **前置任务**：M026

## 一、任务目标
编写 docker-compose.yml 编排文件，统一管理 MySQL、Redis、Elasticsearch、RabbitMQ、后端 Spring Boot 服务、前端 Nginx 服务，实现 `docker-compose up` 一键启动整个 IntelliMES 平台，完成生产环境容器化部署能力。

## 二、前置条件
- M026 已完成测试，应用可打包
- 本机已安装 Docker 与 Docker Compose
- 已阅读计划书"交付标准：Docker Compose 一键启动"

## 三、详细执行步骤
### 步骤 1：编写后端 Dockerfile
在 `intellimes-backend/Dockerfile`：
```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY intellimes-system/target/intellimes-system.jar app.jar
ENV TZ=Asia/Shanghai
ENV JAVA_OPTS="-Xms512m -Xmx1024m -Duser.timezone=Asia/Shanghai"
EXPOSE 8080
ENTRYPOINT ["sh","-c","java $JAVA_OPTS -jar app.jar --spring.profiles.active=prod"]
```

### 步骤 2：编写前端 Dockerfile
在 `intellimes-frontend/Dockerfile`：
```dockerfile
# 构建阶段
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
# 运行阶段
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

### 步骤 3：编写前端 nginx.conf
```nginx
server {
    listen 80;
    server_name localhost;
    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
    location /api/ {
        proxy_pass http://backend:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 步骤 4：编写 docker-compose.yml
在项目根目录 `docker-compose.yml`：
```yaml
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:intellimes123}
      MYSQL_DATABASE: intellimes
    ports: ["3306:3306"]
    volumes:
      - mysql_data:/var/lib/mysql
      - ./intellimes-backend/intellimes-system/src/main/resources/db:/docker-entrypoint-initdb.d
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_general_ci

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    volumes: [redis_data:/data]

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports: ["9200:9200"]
    volumes: [es_data:/usr/share/elasticsearch/data]

  rabbitmq:
    image: rabbitmq:3-management
    ports: ["5672:5672","15672:15672"]
    volumes: [rabbitmq_data:/var/lib/rabbitmq]

  backend:
    build: ./intellimes-backend
    ports: ["8080:8080"]
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/intellimes
      - SPRING_DATA_REDIS_HOST=redis
      - SPRING_ELASTICSEARCH_URIS=http://elasticsearch:9200
      - SPRING_RABBITMQ_HOST=rabbitmq
      - DEEPSEEK_API_KEY=${DEEPSEEK_API_KEY}
    depends_on: [mysql, redis, elasticsearch, rabbitmq]

  frontend:
    build: ./intellimes-frontend
    ports: ["80:80"]
    depends_on: [backend]

volumes:
  mysql_data:
  redis_data:
  es_data:
  rabbitmq_data:
```

### 步骤 5：编写 application-prod.yml
在 `intellimes-system/src/main/resources/application-prod.yml`：
- 所有连接用环境变量：`${SPRING_DATASOURCE_URL}`、`${SPRING_DATA_REDIS_HOST}` 等
- 关闭 SQL 日志打印
- 开启生产级日志级别（INFO）
- 配置跨域允许前端域名

### 步骤 6：编写 .env 文件模板
在项目根目录 `.env.example`：
```
MYSQL_ROOT_PASSWORD=intellimes123
DEEPSEEK_API_KEY=sk-xxx
```
实际 `.env` 不提交到 git（加入 .gitignore）。

### 步骤 7：编写启动脚本
`start.sh`（Linux/Mac）与 `start.bat`（Windows）：
```bash
#!/bin/bash
# 构建后端
cd intellimes-backend && mvn clean package -DskipTests && cd ..
# 启动所有服务
docker-compose up -d --build
# 等待后端就绪
echo "等待后端启动..."
until curl -s http://localhost:8080/actuator/health | grep UP; do
  sleep 5
done
echo "IntelliMES 启动完成：http://localhost"
```

### 步骤 8：编写停止与清理脚本
`stop.sh`：
```bash
docker-compose down
```
`reset.sh`（含数据清理）：
```bash
docker-compose down -v
```

### 步骤 9：本地验证一键启动
- 执行 `start.sh` 或 `start.bat`
- 等待所有容器启动（首次拉镜像较慢）
- 访问 http://localhost 验证前端
- 访问 http://localhost:8080/actuator/health 验证后端
- admin/123456 登录验证业务可用
- 访问 http://localhost:15672 验证 RabbitMQ 管理界面

### 步骤 10：处理启动顺序与依赖
- `depends_on` 只保证容器启动顺序，不保证服务就绪
- 后端用 Spring Boot 重试机制或 `wait-for-it.sh` 脚本等待 MySQL 就绪
- ES 启动较慢，后端连接失败重试配置

## 四、涉及文件清单
| 文件路径 | 说明 |
|---------|------|
| `intellimes-backend/Dockerfile` | 后端镜像构建 |
| `intellimes-frontend/Dockerfile` | 前端镜像构建 |
| `intellimes-frontend/nginx.conf` | Nginx 配置 |
| `docker-compose.yml` | 编排文件 |
| `intellimes-system/src/main/resources/application-prod.yml` | 生产环境配置 |
| `.env.example` | 环境变量模板 |
| `.gitignore` | 补充忽略 .env |
| `start.sh` / `start.bat` | 一键启动脚本 |
| `stop.sh` / `stop.bat` | 停止脚本 |
| `reset.sh` | 重置（含数据清理） |

## 五、预期产出
- 完整的 Docker Compose 编排文件（6 个服务）
- 前后端 Dockerfile
- 一键启动/停止脚本
- `docker-compose up` 即可启动整个平台
- 简历亮点："Docker Compose 一键启动，容器化部署"

## 六、验证方式
- `docker-compose up -d --build` 全部容器启动成功
- `docker-compose ps` 显示 6 个服务状态 Up
- http://localhost 显示前端登录页
- admin/123456 登录后业务正常
- http://localhost:8080/actuator/health 返回 UP
- http://localhost:15672 RabbitMQ 管理界面可访问
- 数据持久化：重启容器后数据仍在（volumes 挂载）

## 七、技术要点与注意事项
- **一键启动是交付标准（计划书交付标准）**：docker-compose up 即可拉起全部服务，无需手动安装配置中间件，是项目交付的核心要求，也是面试展示亮点。
- **多阶段构建**：前端 Dockerfile 用多阶段构建（node 编译 + nginx 运行），最终镜像仅含 nginx 与静态文件，体积小、安全。
- **服务依赖与就绪**：`depends_on` 只保证启动顺序，MySQL/ES 启动需时间，后端应用需配置连接重试或用 `wait-for-it.sh` 等待端口可用，避免启动失败。
- **数据持久化**：MySQL、Redis、ES、RabbitMQ 数据目录挂载到 volume，容器重建后数据不丢失；生产环境建议挂载到宿主机目录或用云存储。
- **环境变量注入**：敏感信息（密码、API Key）通过 `.env` 文件或环境变量注入，不硬编码到镜像；`.env` 加入 `.gitignore` 不提交。
- **资源限制**：ES 占内存大，`ES_JAVA_OPTS=-Xms512m -Xmx512m` 限制堆内存；生产环境根据服务器配置调整，避免 OOM。
- **网络隔离**：docker-compose 默认创建 bridge 网络，服务间用服务名通信（如 `mysql:3306`）；前端 Nginx 通过 `backend:8080` 反向代理后端。
- **日志管理**：容器日志用 `docker-compose logs -f backend` 查看；生产环境用 ELK 或 Loki 集中收集，避免单机日志丢失。
- **镜像优化**：用 alpine 基础镜像减小体积；`.dockerignore` 排除 node_modules、target 等无用文件，加速构建。

---

## ✅ 任务完成状态

| 项目 | 内容 |
|------|------|
| 是否完成 | ☐ 未完成 |
| 完成时间 | 　　　　 |
| 执行人 | 　　　　 |
| 验收结果 | 　　　　 |
| 备注 | 　　　　 |
