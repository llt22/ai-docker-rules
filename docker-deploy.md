# Docker 部署配置生成规则

> 将此文件内容追加到你的 AI 编码工具规则文件中（CLAUDE.md / .cursorrules / .windsurfrules / AGENTS.md / copilot-instructions.md），AI 即可遵循这套规则为项目生成 Docker 部署配置。本文件是纯 Markdown 规则文档，不依赖任何插件或扩展机制。

---

## 触发条件

当用户要求"部署"、"Docker"、"容器化"、"写 docker-compose"、"Dockerfile"、"上线"、"服务器部署"时，遵循以下规则。

---

## 适用范围

本规则适用于 Docker Compose 单机部署，典型场景包括 VPS、单台云服务器、小团队内部系统、个人项目、MVP、低到中等流量 Web 服务。

本规则不覆盖 Kubernetes、Swarm、蓝绿发布、滚动发布、多实例高可用、跨主机编排、服务网格、自动扩缩容。如果用户明确要求这些场景，必须先说明本规则不适用，再按对应平台的最佳实践处理。

默认目标是生成可以在本机和服务器用同一个 `./deploy.sh` 执行的部署配置。

本规则假设单仓库对应单个可部署服务。如果项目是 Monorepo（一个仓库包含多个独立部署的服务，如 frontend + backend），必须先询问用户要为哪个服务生成部署配置，或是否需要为每个服务分别生成独立的 `Dockerfile` 和 `docker-compose.yml`。

---

## AI 执行协议

执行时必须按以下顺序工作，不得跳过阶段：

1. **分析项目** — 读取项目文件，识别技术栈、启动命令、构建命令、数据库、迁移工具、现有 Docker 配置、运行时依赖。
2. **做出决策** — 判断是否需要数据库服务、migration 服务、额外依赖服务、开发 override 文件。
3. **生成或修改文件** — 优先修改现有配置；没有配置时再新建标准文件。
4. **执行自检** — 按本文的自检清单逐项核对，不通过就继续修正。
5. **交付结果** — 使用本文的最终回复格式说明生成文件、部署命令、环境变量和注意事项。

如果信息充分，可以直接生成配置，不必为了形式确认。只有在关键事实无法判断时才向用户提问，例如无法确定启动命令、数据库类型、迁移命令、对外端口。

---

## 决策树

### 是否使用数据库

如果项目没有数据库连接、ORM、迁移文件或持久化存储需求，则按无数据库项目处理，不生成 db / migration 服务。

如果项目使用 PostgreSQL、MySQL、MongoDB 等数据库，则必须生成 db 服务、named volume、healthcheck、migration 服务和有数据库版本的 `deploy.sh`。

如果项目只使用 Redis 作为缓存或队列，不要把 Redis 当作 migration 目标；只生成 Redis 服务和 healthcheck，不生成 migration 服务，除非项目另有数据库迁移工具。

如果项目同时使用多个运行时依赖（如 PostgreSQL + Redis + RabbitMQ），`deploy.sh` 中必须一次性启动所有依赖服务，例如 `docker compose up -d db redis rabbitmq`，而不是只写 `docker compose up -d db`。

### 是否已有 Docker 配置

如果已有 `Dockerfile`、`docker-compose.yml`、`.dockerignore`、`deploy.sh`，必须优先增量修改，保留项目已有的有效设置。

如果现有配置违反本规则，例如迁移写在 app 启动命令里、数据库 bind mount 到项目目录、硬编码密码，必须修正并在最终回复中说明。

如果没有现有配置，按本文模板生成完整文件。

### 是否需要询问用户

信息充分时直接生成配置。

信息不足但可以从项目惯例推断时，先采用合理默认值，并在最终回复中标明假设。

信息不足且会影响部署正确性时，先问用户，不要猜测。关键问题包括应用监听端口、生产启动命令、迁移命令、数据库类型、必须暴露的服务端口。

---

## 阶段一：项目分析

在生成任何配置之前，必须先理解项目：

1. **识别技术栈** — 检查 package.json / requirements.txt / pyproject.toml / go.mod / pom.xml / build.gradle / Cargo.toml / Gemfile 等，确定语言版本、运行时、构建方式
2. **识别数据库** — 检查 ORM 配置、数据库连接代码、已有迁移文件，确定数据库类型
3. **识别迁移工具** — Prisma / Knex / TypeORM / Drizzle / Alembic / Django ORM / Flyway / golang-migrate 等，确定迁移命令
4. **识别现有配置** — 检查是否已有 Dockerfile / docker-compose.yml / .env / .dockerignore，已有时优先修改而非重写
5. **识别额外服务** — Redis / RabbitMQ / Elasticsearch / Nginx 等

如果识别结果存在关键不确定项，必须先向用户确认；否则可以直接进入下一阶段。

## 阶段二：生成部署配置

### 必须生成的文件

| 文件 | 用途 |
|------|------|
| `Dockerfile` | 应用镜像构建 |
| `docker-compose.yml` | 服务编排（app + db + migration + 其他依赖） |
| `.env.example` | 环境变量模板（提交到仓库） |
| `.dockerignore` | 排除不必要文件 |
| `deploy.sh` | 统一部署脚本（生成后自动添加可执行权限） |

> **无数据库项目**：如果项目不使用数据库（纯 API 代理、静态站点、无状态微服务等），跳过 db 和 migration 服务，deploy.sh 中也不执行迁移命令。

### 可选生成的文件

| 文件 | 条件 |
|------|------|
| `docker-compose.override.yml` | 开发环境需要热重载、源码挂载时 |

## 阶段三：核心原则

生成配置时，以下规则必须遵守。违反任何一条都需要停下来修正。

### 3.1 迁移容器独立

- 数据库迁移用单独的 `migration` service，不写进主应用的 CMD / ENTRYPOINT
- migration 服务与 app 服务使用同一镜像 / 构建上下文
- migration 服务不设置 `restart` 策略（one-off 执行）

### 3.2 迁移幂等

- 迁移命令可以安全重复执行，首次部署和升级用同一命令
- 使用迁移工具自身的版本管理机制（migration history table）
- 种子数据使用幂等写法（upsert / ON CONFLICT DO NOTHING）

### 3.3 命令统一

- 本机和服务器执行完全相同的 `deploy.sh`
- 通过 `.env` 文件区分环境差异，不在脚本中判断环境

### 3.4 数据持久化

- 数据库数据必须用 named volume
- 禁止 bind mount 到项目目录存储数据库文件

### 3.5 环境变量外置

- 所有可变配置通过 `.env` 注入
- 禁止硬编码端口、密码、主机名
- `.env.example` 提交到仓库，`.env` 加入 `.gitignore`

### 3.6 健康检查

- 数据库等依赖服务必须配置 `healthcheck`
- app 通过 `depends_on` + `condition: service_healthy` 等待依赖就绪

### 3.7 镜像优化

- 有编译步骤的项目使用 multi-stage build
- 基础镜像指定具体版本，禁止使用 `latest`
- 优先选用 alpine / slim 变体
- 使用非 root 用户运行应用

### 3.8 配置一致性

- `docker-compose.yml` 中的服务名必须和 `.env.example` 中的主机名一致，例如 `DB_HOST=db`
- `DATABASE_URL` 必须和拆分变量 `DB_HOST`、`DB_PORT`、`DB_NAME`、`DB_USER`、`DB_PASSWORD` 表达同一个连接目标
- app、migration 必须使用同一个镜像或同一个构建上下文
- migration 命令必须使用项目实际迁移工具，不得使用占位命令交付
- `deploy.sh` 中启动的依赖服务名必须存在于 `docker-compose.yml`

### 3.9 迁移安全

- 迁移命令必须是生产部署命令，例如 Prisma 使用 `prisma migrate deploy`，不能使用开发命令 `prisma migrate dev`
- 破坏性迁移必须在最终回复中提示风险，例如删除字段、重命名字段、修改字段含义、批量重写数据
- 不要用应用启动命令隐式触发迁移
- 不要用 `sleep` 等待数据库，必须使用 healthcheck 和 `depends_on.condition`

## 阶段四：deploy.sh 规范

`deploy.sh` 是唯一需要执行的部署入口。本机和服务器使用同一个脚本，通过是否存在数据库迁移区分脚本结构。

### 有数据库项目

应用必须在迁移完成后再启动，避免新版本应用先于 schema 变更对外服务。

```bash
#!/bin/bash
set -e

echo "构建镜像..."
docker compose build

echo "启动依赖服务..."
docker compose up -d db

echo "执行数据库迁移..."
docker compose run --rm migration

echo "启动应用服务..."
docker compose up -d app

echo "部署完成"
```

### 无数据库项目

无数据库项目不生成 db / migration 服务，也不执行迁移命令。

```bash
#!/bin/bash
set -e

echo "构建镜像 & 启动服务..."
docker compose up -d --build

echo "部署完成"
```

规则：
- 首次部署和后续升级执行同一个 `deploy.sh`
- 脚本必须幂等，重复执行不产生副作用
- 使用 `docker compose`（V2）而非 `docker-compose`（V1）
- 不包含环境判断逻辑
- 启动依赖服务时列出 db / redis / rabbitmq 等所有运行时依赖，不包含 app 和 migration
- 有数据库项目必须先执行 migration，再启动 app
- migration 失败时脚本会因 `set -e` 自动终止，app 不会启动，这是安全的预期行为；用户修复迁移问题后重新执行 `./deploy.sh` 即可
- 生成后执行 `chmod +x deploy.sh` 添加可执行权限

## 阶段五：自检

生成完所有文件后，逐项验证：

### Dockerfile

- [ ] 基础镜像指定了具体版本（非 latest）
- [ ] 依赖安装在源码拷贝之前（利用缓存分层）
- [ ] 有编译步骤时使用了多阶段构建
- [ ] 最终镜像不包含开发依赖和构建工具
- [ ] 使用非 root 用户运行应用
- [ ] CMD / ENTRYPOINT 中没有混入迁移逻辑

### docker-compose.yml

- [ ] 定义了独立的 migration 服务
- [ ] migration 服务没有 restart 策略
- [ ] migration 服务与 app 服务使用同一镜像 / 构建上下文
- [ ] 数据库服务配置了 healthcheck
- [ ] app 服务通过 depends_on + condition 等待运行时依赖就绪
- [ ] app 服务通过 `deploy.sh` 保证在 migration 成功后启动
- [ ] 数据库使用 named volume 持久化
- [ ] 所有服务使用 env_file 而非硬编码环境变量
- [ ] 使用自定义 network
- [ ] app、migration 使用同一个镜像或同一个构建上下文
- [ ] deploy.sh 中引用的服务名都存在

### .env.example

- [ ] 包含所有必需的环境变量
- [ ] 密码类变量使用占位值（如 `changeme`）
- [ ] DB_HOST 指向 compose 中的服务名（如 `db`）而非 `localhost`
- [ ] 不包含真实密钥、token、生产密码
- [ ] DATABASE_URL 与拆分数据库变量保持一致

### deploy.sh

- [ ] 第一行有 `set -e`
- [ ] 使用 `docker compose`（V2）
- [ ] 有数据库项目命令顺序：构建镜像 → 启动依赖 → 迁移 → 启动应用
- [ ] 无数据库项目不包含 migration 命令
- [ ] 可重复执行无副作用

### .dockerignore

- [ ] 排除了依赖目录（node_modules / venv / target）
- [ ] 排除了 .git 和 .env

### 一致性

- [ ] 本机和服务器执行 `./deploy.sh` 命令完全一致
- [ ] 首次部署（空数据库）和后续升级执行同一命令
- [ ] README 或最终回复中的部署命令与生成的脚本一致
- [ ] 对所有推断、假设、破坏性迁移风险已在最终回复中明确说明
- [ ] 有数据库项目不可直接 `docker compose up -d`，必须通过 deploy.sh 控制启动顺序

**全部通过后才能向用户报告完成。**

## 禁止事项

- ❌ 在 Dockerfile 的 RUN 中执行数据库迁移
- ❌ 在 app 容器的 CMD / ENTRYPOINT 中混入迁移逻辑
- ❌ 使用 `sh -c "migrate && start"` 作为 app 启动命令
- ❌ 使用 `sleep 5` / `sleep 10` 等方式等待数据库
- ❌ 硬编码端口、密码、主机名等环境相关值
- ❌ 使用 `latest` 标签作为基础镜像
- ❌ 省略 `.dockerignore`
- ❌ 数据库使用 bind mount 代替 named volume
- ❌ migration 服务设置 restart 策略
- ❌ 把 `.env` 复制进镜像或提交到仓库
- ❌ 在 `.env.example` 中写真实密钥
- ❌ 使用开发迁移命令作为生产迁移命令

## 常见错误反例

### 错误：在 app 启动时执行迁移

```yaml
services:
  app:
    command: sh -c "npm run migrate && npm run start"
```

正确做法：迁移放到独立 `migration` service，由 `deploy.sh` 在 app 启动前执行。

### 错误：在 Dockerfile 构建阶段执行迁移

```dockerfile
RUN npx prisma migrate deploy
```

正确做法：镜像构建不能依赖运行时数据库，迁移只能在部署阶段执行。

### 错误：用 sleep 等待数据库

```bash
sleep 10
docker compose run --rm migration
```

正确做法：数据库服务配置 healthcheck，migration 通过 `depends_on.condition: service_healthy` 等待数据库就绪。

### 错误：数据库数据 bind mount 到项目目录

```yaml
volumes:
  - ./postgres:/var/lib/postgresql/data
```

正确做法：使用 named volume，例如 `postgres_data:/var/lib/postgresql/data`。

---

## 最终回复格式

生成或修改配置后，必须用以下格式回复用户：

```markdown
已生成 Docker Compose 部署配置。

修改文件：
- Dockerfile：应用镜像构建
- docker-compose.yml：服务编排
- .env.example：环境变量模板
- .dockerignore：构建上下文排除规则
- deploy.sh：统一部署入口

部署方式：
1. cp .env.example .env
2. 编辑 .env 填入真实值
3. ./deploy.sh

后续升级：
1. ./deploy.sh

需要确认或填写的变量：
- APP_PORT
- DATABASE_URL
- DB_PASSWORD

自检结果：
- Dockerfile：通过
- docker-compose.yml：通过
- deploy.sh：通过
- 环境变量一致性：通过

注意事项：
- 如果存在破坏性数据库迁移，先备份数据库再部署。
```

如果项目无数据库，最终回复中不得出现 migration 步骤。

如果存在假设，必须增加”推断与假设”小节，区分”从项目文件推断的结论”和”无法确认的假设”。

---

## 附录：各技术栈模板参考

### Dockerfile — Node.js / TypeScript（多阶段构建）

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS production

WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=builder /app/dist ./dist

USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Dockerfile — Python

```dockerfile
FROM python:3.12-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim AS production

WORKDIR /app
COPY --from=builder /install /usr/local
COPY . .

RUN useradd -m appuser
USER appuser
EXPOSE 8000
CMD ["gunicorn", "app:app", "--bind", "0.0.0.0:8000"]
```

### Dockerfile — Go

```dockerfile
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM alpine:3.19 AS production

WORKDIR /app
COPY --from=builder /app/server .

RUN adduser -D appuser
USER appuser
EXPOSE 8080
CMD ["./server"]
```

### Dockerfile — Java / Spring Boot

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /app
COPY gradlew settings.gradle build.gradle ./
COPY gradle ./gradle
RUN ./gradlew dependencies --no-daemon
COPY . .
RUN ./gradlew bootJar --no-daemon

FROM eclipse-temurin:21-jre-alpine AS production

WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar

RUN adduser -D appuser
USER appuser
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

### 迁移命令速查

| 技术栈 | 工具 | 命令 |
|--------|------|------|
| Node.js | Prisma | `npx prisma migrate deploy` |
| Node.js | Knex | `npx knex migrate:latest` |
| Node.js | TypeORM | `npx typeorm migration:run -d dist/data-source.js` |
| Node.js | Drizzle | `npx drizzle-kit migrate` |
| Python | Alembic | `alembic upgrade head` |
| Python | Django | `python manage.py migrate` |
| Go | golang-migrate | `migrate -path ./migrations -database $DATABASE_URL up` |
| Go | goose | `goose -dir ./migrations up` |
| Java | Flyway | `flyway migrate` |
| Java | Liquibase | `liquibase update` |
| Ruby | Rails | `bundle exec rake db:migrate` |

### docker-compose.yml 通用模板

模板中的 `app:local` 必须替换为项目相关镜像名，例如 `my-project:local`。app 和 migration 必须引用同一个镜像，避免迁移容器和应用容器运行不同版本代码。

> **注意**：生产环境必须通过 `deploy.sh` 控制启动顺序（先迁移后启动 app），不要直接执行 `docker compose up -d`，否则 app 可能在 migration 未完成时就启动。

```yaml
services:
  app:
    build: .
    image: app:local
    env_file: .env
    ports:
      - "${APP_PORT:-3000}:3000"
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - app-network

  db:
    image: postgres:16-alpine
    env_file: .env
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - app-network

  migration:
    image: app:local
    env_file: .env
    command: ["npm", "run", "migrate"]
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
```

### 数据库 healthcheck 变体

**MySQL：**
```yaml
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  interval: 5s
  timeout: 5s
  retries: 5
```

**MongoDB：**
```yaml
healthcheck:
  test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
  interval: 5s
  timeout: 5s
  retries: 5
```

**Redis：**
```yaml
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 5s
  timeout: 5s
  retries: 5
```

### .env.example 通用模板

```env
# Application
APP_PORT=3000
# 按技术栈选择环境变量名：NODE_ENV / FLASK_ENV / GIN_MODE / RAILS_ENV 等
APP_ENV=production

# Database
DB_HOST=db
DB_PORT=5432
DB_NAME=app
DB_USER=postgres
DB_PASSWORD=changeme
DATABASE_URL=postgresql://postgres:changeme@db:5432/app

# PostgreSQL container（值必须与上方保持一致）
POSTGRES_DB=app
POSTGRES_USER=postgres
POSTGRES_PASSWORD=changeme
```

### .dockerignore 通用模板

```
node_modules
.git
.env
dist
build
*.log
.vscode
.idea
__pycache__
*.pyc
venv
.venv
target
.DS_Store
```

### deploy.sh 模板（有数据库）

```bash
#!/bin/bash
set -e

echo "构建镜像..."
docker compose build

echo "启动依赖服务..."
docker compose up -d db

echo "执行数据库迁移..."
docker compose run --rm migration

echo "启动应用服务..."
docker compose up -d app

echo "部署完成"
```

### deploy.sh 模板（无数据库）

```bash
#!/bin/bash
set -e

echo "构建镜像 & 启动服务..."
docker compose up -d --build

echo "部署完成"
```
