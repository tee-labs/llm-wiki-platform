# LLM Wiki Platform — 环境搭建指南

> 从零开始搭建完整的前后端运行环境，包括数据库、后端、前端。

---

## 目录

1. [环境要求](#1-环境要求)
2. [数据库搭建](#2-数据库搭建)
3. [配置参数说明](#3-配置参数说明)
4. [Docker Compose 一键部署（推荐）](#4-docker-compose-一键部署推荐)
5. [本地开发环境搭建](#5-本地开发环境搭建)
6. [验证服务](#6-验证服务)
7. [自定义扩展说明](#7-自定义扩展说明)
8. [常见问题排查](#8-常见问题排查)

---

## 1. 环境要求

### 最低要求（Docker 部署方式）

| 组件 | 版本 | 说明 |
|------|------|------|
| Docker | 20.10+ | 容器引擎 |
| Docker Compose | v2.0+ | 编排工具（`docker compose` 命令） |
| 内存 | 4GB+ | 推荐 8GB |
| 磁盘 | 10GB+ | 镜像 + 数据 |

### 本地开发额外要求

| 组件 | 版本 | 说明 |
|------|------|------|
| Java | 17 (Temurin/Oracle) | 后端运行 |
| Maven | 3.9+ | 后端构建 |
| Node.js | 18+ | 前端开发 |
| npm | 9+ | 前端包管理 |

---

## 2. 数据库搭建

本项目使用 **PostgreSQL + pgvector**（向量扩展），用于存储知识图谱的向量数据。

### 2.1 Docker 方式（推荐）

Docker Compose 已内置 PostgreSQL + pgvector，使用 `ankane/pgvector` 镜像，无需手动安装扩展：

```bash
# 仅启动数据库
docker compose up -d postgres

# 等待健康检查通过（约 10-15 秒）
docker compose ps
```

数据库默认配置：

| 参数 | 值 |
|------|-----|
| 数据库名 | `llmwiki` |
| 用户名 | `llmwiki` |
| 密码 | `llmwiki` |
| 端口 | `5432` |
| 数据卷 | `pgdata`（持久化） |

### 2.2 手动安装 PostgreSQL + pgvector

如果不想用 Docker，可以手动安装：

**Ubuntu/Debian:**

```bash
# 安装 PostgreSQL 16
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt update
sudo apt install postgresql-16

# 安装 pgvector 扩展
sudo apt install postgresql-16-pgvector

# 启动服务
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

**macOS (Homebrew):**

```bash
brew install postgresql@16
brew install pgvector
brew services start postgresql@16
```

**创建数据库和扩展:**

```sql
-- 登录 PostgreSQL
sudo -u postgres psql

-- 创建数据库和用户
CREATE USER llmwiki WITH PASSWORD 'llmwiki';
CREATE DATABASE llmwiki OWNER llmwiki;

-- 启用 pgvector 扩展
\c llmwiki
CREATE EXTENSION IF NOT EXISTS vector;

-- 验证
\dx
```

### 2.3 数据库结构说明

首次启动时，Flyway 会自动执行以下迁移脚本：

| 版本 | 文件 | 说明 |
|------|------|------|
| V1 | `V1__init_schema.sql` | 初始化所有表 + pgvector 扩展 |
| V2 | `V2__add_page_sources.sql` | 页面来源关联 |
| V3 | `V3__dead_letter_queue.sql` | 死信队列（失败重试） |
| V4 | `V4__maintenance_report_log.sql` | 维护报告日志 |
| V5 | `V5__audit_log.sql` | 审计日志 |
| V6 | `V6__prd_compliance_fixes.sql` | PRD 合规修复 |
| V7 | `V7__entity_sub_type.sql` | 实体子类型 |
| V8 | `V8__create_entity_examples.sql` | 实体示例表 |

核心表：

| 表名 | 用途 |
|------|------|
| `wiki_sources` | Wiki 数据源配置 |
| `raw_documents` | 从数据源同步的原始文档 |
| `sync_logs` | 同步日志 |
| `processing_log` | 处理流水线日志 |
| `kg_nodes` | 知识图谱节点 |
| `kg_edges` | 知识图谱边 |
| `kg_vectors` | 节点向量（pgvector `vector(1536)`） |
| `pages` | 生成的页面 |
| `page_links` | 页面交叉链接 |
| `page_tags` | 页面标签 |
| `approval_queue` | 审批队列 |
| `system_config` | 系统配置 |

---

## 3. 配置参数说明

### 3.1 后端环境变量

所有参数通过 `application.yml` 中的 `${ENV_VAR:default}` 语法注入。

#### 数据库连接

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `DB_HOST` | `localhost` | PostgreSQL 主机地址（Docker 内用 `postgres`） |
| `DB_USERNAME` | `llmwiki` | 数据库用户名 |
| `DB_PASSWORD` | `llmwiki` | 数据库密码 |

#### AI API 配置（核心）

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `AI_API_BASE_URL` | `http://localhost:8000/v1` | OpenAI 兼容 API 地址 |
| `AI_API_KEY` | `sk-local` | API 密钥 |
| `AI_MODEL` | `gpt-4o-mini` | 对话/评分模型 |
| `EMBEDDING_MODEL` | `text-embedding-ada-002` | 向量嵌入模型 |
| `EMBEDDING_DIMENSION` | `1536` | 向量维度（需与模型匹配） |

> **重要**: AI API 必须兼容 OpenAI 接口格式（`/v1/chat/completions`、`/v1/embeddings`）。支持本地部署的 Ollama、vLLM、LM Studio 等。

#### 安全配置

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `JWT_SECRET` | `llm-wiki-platform-default-jwt-secret-key-256bits` | JWT 签名密钥（**生产环境必须修改**） |
| `ENCRYPTION_KEY` | `llm-wiki-default-key-32chars-long!` | 数据加密密钥（**生产环境必须修改**） |

#### 流水线配置（application.yml 内）

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `pipeline.thread-pool-size` | `4` | 并行处理线程数 |
| `pipeline.retry-max-attempts` | `3` | 失败重试次数 |
| `pipeline.retry-delay-ms` | `1000` | 重试间隔（毫秒） |
| `sync.enabled` | `true` | 是否启用定时同步 |
| `sync.default-cron` | `0 0 */6 * * *` | 同步周期（每 6 小时） |
| `llm.wiki.dedup.similarity-threshold` | `0.85` | 去重相似度阈值 |
| `llm.wiki.dedup.warn-threshold` | `0.70` | 去重警告阈值 |

### 3.2 前端配置

前端通过 Vite 代理连接后端，无需单独配置环境变量。

| 环境 | API 地址 | 说明 |
|------|----------|------|
| 开发模式 | `http://localhost:8080/api` | Vite proxy 转发 |
| Docker 生产 | `http://llm-wiki:8080/api` | nginx 反向代理 |

---

## 4. Docker Compose 一键部署（推荐）

这是最简单的方式，一条命令启动所有服务。

### 4.1 克隆项目

```bash
git clone <repository-url>
cd llm-wiki-platform
```

### 4.2 配置 AI API

编辑 `docker-compose.yml`，修改 `llm-wiki` 服务的环境变量：

```yaml
services:
  llm-wiki:
    environment:
      DB_HOST: postgres
      DB_USERNAME: llmwiki
      DB_PASSWORD: llmwiki
      AI_API_BASE_URL: http://your-llm-server/v1    # ← 改这里
      AI_API_KEY: your-api-key                        # ← 改这里
      AI_MODEL: gpt-4o-mini                           # ← 按需修改
```

**常见 AI 后端配置示例：**

| 后端 | AI_API_BASE_URL | AI_MODEL |
|------|-----------------|----------|
| OpenAI 官方 | `https://api.openai.com/v1` | `gpt-4o-mini` |
| Ollama 本地 | `http://host.docker.internal:11434/v1` | `llama3` |
| vLLM 本地 | `http://host.docker.internal:8000/v1` | `Qwen2-7B` |
| DeepSeek | `https://api.deepseek.com/v1` | `deepseek-chat` |
| 智谱 GLM | `https://open.bigmodel.cn/api/paas/v4` | `glm-4-flash` |

### 4.3 启动所有服务

```bash
# 构建镜像并启动（首次运行）
docker compose up -d --build

# 后续启动（不重新构建）
docker compose up -d
```

启动顺序：`postgres` → 健康检查通过 → `llm-wiki`（后端）→ `frontend`（前端）

### 4.4 查看状态和日志

```bash
# 查看所有服务状态
docker compose ps

# 查看后端日志
docker compose logs -f llm-wiki

# 查看数据库日志
docker compose logs -f postgres

# 查看前端日志
docker compose logs -f frontend
```

### 4.5 访问服务

| 服务 | 地址 | 说明 |
|------|------|------|
| 前端 | http://localhost:3000 | Web 界面 |
| 后端 API | http://localhost:8080/api | REST API |
| 数据库 | localhost:5432 | PostgreSQL |

### 4.6 停止和清理

```bash
# 停止服务（保留数据）
docker compose down

# 停止并删除数据卷（⚠️ 删除所有数据）
docker compose down -v
```

---

## 5. 本地开发环境搭建

适合需要调试代码、修改功能的场景。

### 5.1 启动数据库

```bash
# 仅用 Docker 启动 PostgreSQL
docker compose up -d postgres
```

### 5.2 启动后端

```bash
# 进入后端目录
cd backend

# 构建（跳过测试加速）
mvn clean package -DskipTests

# 运行
mvn spring-boot:run -pl llm-wiki-web
```

或者使用本地配置创建 `application-dev.yml`：

```yaml
# backend/llm-wiki-web/src/main/resources/application-dev.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/llmwiki
    username: llmwiki
    password: llmwiki

ai:
  api:
    base-url: http://localhost:11434/v1   # 本地 Ollama
    key: ollama
  model: llama3

logging:
  level:
    com.llmwiki: DEBUG
```

```bash
# 使用 dev profile 启动
mvn spring-boot:run -pl llm-wiki-web -Dspring-boot.run.profiles=dev
```

后端启动后监听 `http://localhost:8080`。

### 5.3 启动前端

```bash
# 进入前端目录
cd frontend

# 安装依赖
npm install

# 启动开发服务器
npm run dev
```

前端开发服务器监听 `http://localhost:3000`，API 请求自动代理到 `localhost:8080`。

### 5.4 构建前端生产版本

```bash
cd frontend
npm run build
# 产物在 dist/ 目录
```

---

## 6. 验证服务

### 6.1 检查后端健康

```bash
curl http://localhost:8080/actuator/health
# 返回: {"status":"UP"}
```

### 6.2 检查数据库连接

```bash
# 进入数据库容器
docker exec -it llmwiki-postgres psql -U llmwiki -d llmwiki

# 检查表是否创建
\dt

# 检查 pgvector 扩展
\dx

# 检查种子数据
SELECT * FROM system_config;
```

### 6.3 检查前端

浏览器访问 http://localhost:3000，应看到 Ant Design 风格的界面。

### 6.4 测试 API

```bash
# 注册
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'

# 登录获取 token
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'
```

---

## 7. 自定义扩展说明

### 7.1 添加新的 Wiki 数据源

1. 实现 `WikiSourceAdapter` 接口（位于 `llm-wiki-adapter` 模块）
2. 在 `wiki_sources` 表中注册新源：

```sql
INSERT INTO wiki_sources (name, base_url, adapter_class, enabled)
VALUES ('My Wiki', 'https://mywiki.example.com', 'com.llmwiki.adapter.wiki.MyWikiAdapter', true);
```

### 7.2 更换 AI 模型

修改 `docker-compose.yml` 或 `application.yml`：

```yaml
ai:
  api:
    base-url: https://your-new-api/v1
    key: your-key
  model: your-model

embedding:
  model: your-embedding-model
  dimension: 1536   # 必须与模型输出维度一致
```

> ⚠️ 修改 `EMBEDDING_DIMENSION` 后，需要清空 `kg_vectors` 表并重新处理所有文档。

### 7.3 调整评分阈值

通过系统配置表动态调整（无需重启）：

```sql
-- 修改最低评分阈值（0-10）
UPDATE system_config SET config_value = '6.0' WHERE config_key = 'scoring.threshold';

-- 修改评分权重
UPDATE system_config SET config_value = 'information_density:0.3,entity_richness:0.25,knowledge_independence:0.2,structure_integrity:0.15,timeliness:0.1'
WHERE config_key = 'scoring.weights';
```

### 7.4 修改同步频率

```yaml
sync:
  default-cron: "0 */2 * * * *"   # 每 2 小时
```

### 7.5 生产环境安全加固

**必须修改以下配置：**

```yaml
jwt:
  secret: <生成一个 256-bit 随机字符串>

encryption:
  key: <生成一个 32 字符随机密钥>
```

生成随机密钥：

```bash
# JWT Secret (32 bytes = 256 bits)
openssl rand -base64 32

# Encryption Key (32 chars)
openssl rand -hex 16
```

**其他建议：**
- 修改数据库默认密码
- 使用 `.env` 文件管理敏感配置（不要提交到 Git）
- 配置 HTTPS（在 nginx 或前置反向代理上）
- 限制数据库端口不对外暴露

### 7.6 数据库备份与恢复

```bash
# 备份
docker exec llmwiki-postgres pg_dump -U llmwiki llmwiki > backup.sql

# 恢复
docker exec -i llmwiki-postgres psql -U llmwiki llmwiki < backup.sql
```

---

## 8. 常见问题排查

### 数据库连接失败

```
Cause: Connection refused
```

- 确认 PostgreSQL 容器已启动：`docker compose ps`
- 检查 `DB_HOST` 是否正确（Docker 内用服务名 `postgres`，宿主机用 `localhost`）
- 等待健康检查通过后再启动后端

### pgvector 扩展未安装

```
ERROR: type "vector" does not exist
```

- Docker 方式：确认使用的是 `ankane/pgvector` 镜像
- 手动安装：`CREATE EXTENSION IF NOT EXISTS vector;`

### AI API 调用失败

```
401 Unauthorized / Connection refused
```

- 检查 `AI_API_BASE_URL` 是否可访问
- 检查 `AI_API_KEY` 是否正确
- Docker 内访问宿主机服务用 `host.docker.internal`（Linux 需额外配置）

### 前端页面空白

- 检查 nginx 配置中 `proxy_pass` 是否指向正确的后端服务名
- 查看浏览器控制台网络请求是否 404

### Flyway 迁移失败

```
Flyway validation failed
```

- 不要手动修改已执行的迁移文件
- 如需修改，创建新的 `V{N}__description.sql` 文件
- 紧急修复：`docker exec -it llmwiki-postgres psql -U llmwiki -d llmwiki -c "DELETE FROM flyway_schema_history WHERE version='X';"`

### 端口冲突

如果 3000/8080/5432 端口已被占用：

```yaml
# docker-compose.yml 修改端口映射
services:
  postgres:
    ports:
      - "5433:5432"    # 改为 5433
  llm-wiki:
    ports:
      - "8081:8080"    # 改为 8081
  frontend:
    ports:
      - "3001:80"      # 改为 3001
```

---

## 项目结构速查

```
llm-wiki-platform/
├── backend/                        # Java 后端（Maven 多模块）
│   ├── llm-wiki-common/           # 公共枚举、DTO（零依赖）
│   ├── llm-wiki-adapter/          # AI API 客户端、Wiki 适配器
│   ├── llm-wiki-domain/           # JPA 实体、Repository
│   ├── llm-wiki-service/          # 业务逻辑
│   └── llm-wiki-web/              # Controller、配置、启动类
│       └── src/main/resources/
│           ├── application.yml    # 主配置
│           └── db/migration/      # Flyway 迁移脚本
├── frontend/                       # React 前端
│   ├── src/
│   │   ├── pages/                 # 页面组件
│   │   ├── components/            # 布局组件
│   │   └── api.ts                 # Axios API 客户端
│   ├── nginx.conf                 # 生产环境 nginx 配置
│   └── vite.config.ts             # Vite 开发配置
├── docker-compose.yml              # 编排配置
├── Dockerfile                      # 后端镜像构建
└── frontend/Dockerfile             # 前端镜像构建
```
