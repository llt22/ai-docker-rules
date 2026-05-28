# DockSkill

一份可复制到 AI 编码工具上下文中的 Markdown 规则文件，让 AI 遵循 Docker Compose 部署最佳实践，为常见 Web 项目生成一致、幂等、可迁移的单机部署配置。

DockSkill 不是插件或扩展包，只是一个文档项目。你把规则内容放进项目的 AI 规则文件后，再让 AI 生成 Docker 部署配置。

它面向 VPS / 单机服务器 / 小团队项目的 Docker Compose 部署，不试图覆盖 Kubernetes、蓝绿发布、滚动发布、多实例高可用等复杂场景。

## 解决什么问题

- 本机跑通了，服务器部署出问题
- 首次部署和后续更新流程不一致
- 数据库迁移混在应用启动流程里，不可控
- 每个项目的 Docker 配置风格不统一
- AI 生成的 Dockerfile、Compose、`.env`、迁移命令互相不匹配

## 核心理念

1. **本机 = 服务器** — 同一个 `deploy.sh`，同一个 `docker-compose.yml`
2. **首次 = 升级** — 迁移命令幂等，不区分初始化和更新
3. **迁移容器独立** — 数据库迁移不混入应用启动流程
4. **技术栈无关** — Node / Python / Go / Java / Ruby 均适用

## 使用方法

### 快速使用

```bash
# 下载规则文件到当前项目
curl -fsSL https://raw.githubusercontent.com/llt22/ai-docker-rules/main/docker-deploy.md -o docker-deploy-rules.md
```

然后将内容追加到你的 AI 编码工具规则文件中，或直接复制粘贴到对应文件：

| AI 工具 | 规则文件 | 操作 |
|---------|---------|------|
| Claude Code | `CLAUDE.md` | `cat docker-deploy-rules.md >> CLAUDE.md` |
| Cursor | `.cursorrules` | `cat docker-deploy-rules.md >> .cursorrules` |
| Windsurf | `.windsurfrules` | `cat docker-deploy-rules.md >> .windsurfrules` |
| GitHub Copilot | `.github/copilot-instructions.md` | `mkdir -p .github && cat docker-deploy-rules.md >> .github/copilot-instructions.md` |
| OpenAI Codex | `AGENTS.md` | `cat docker-deploy-rules.md >> AGENTS.md` |

之后，当你让 AI "帮我写 Docker 部署配置"、"容器化这个项目" 时，它会自动遵循规则生成配置。

## 规则包含什么

- **AI 执行协议**：分析 → 决策 → 生成 → 自检 → 交付
- **决策树**：有/无数据库、有/无现有 Docker 配置、单服务/Monorepo、信息充分/不足分别处理
- **9 条核心原则**：迁移容器独立、迁移幂等、命令统一、数据持久化、环境变量外置、健康检查、镜像优化、配置一致性、迁移安全
- **一致性约束**：`Dockerfile`、`docker-compose.yml`、`.env.example`、`deploy.sh`、迁移命令必须互相匹配
- **反例清单**：明确禁止 AI 常见错误写法
- **自检清单**：生成配置后逐项验证
- **技术栈模板**：Node.js / Python / Go / Java / Ruby 的 Dockerfile + 迁移命令
- **数据库模板**：PostgreSQL / MySQL / MongoDB / Redis

## 部署效果

加入规则后，带数据库项目的部署流程统一为：

```bash
# 首次部署
cp .env.example .env
# 编辑 .env 填入实际值
./deploy.sh

# 后续升级（同一命令）
./deploy.sh
```

无数据库项目会跳过数据库和迁移服务，仍然使用同一个 `./deploy.sh` 完成构建与启动。

如果迁移失败，脚本会自动终止，app 不会启动。修复迁移问题后重新执行 `./deploy.sh` 即可。

## 支持的技术栈

| 语言 | 框架 | 迁移工具 |
|------|------|----------|
| Node.js / TypeScript | Express, NestJS, Fastify... | Prisma, Knex, TypeORM, Drizzle, Sequelize |
| Python | Django, Flask, FastAPI... | Alembic, Django ORM, Flask-Migrate |
| Go | Gin, Echo, Fiber... | golang-migrate, goose, atlas |
| Java | Spring Boot | Flyway, Liquibase |
| Ruby | Rails | ActiveRecord |

## 支持的数据库

PostgreSQL, MySQL, MongoDB, Redis（作为缓存/队列服务）

## 不适用场景

- Kubernetes / Docker Swarm 编排
- 蓝绿发布、滚动发布
- 多实例高可用、自动扩缩容
- 跨主机编排、服务网格

遇到以上场景时，AI 会先说明本规则不适用，再按对应平台的最佳实践处理。

## License

MIT
