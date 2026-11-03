# 本地改动与 Docker 启动

这份说明只覆盖相对上游仓库、在本机做过的改动，以及现在怎么把服务拉起来。密钥、密码、AccessKey 都在 `.env` 里，不要写进这份文档，也不要提交 `.env`。

## Git 管理

当前仓库已接上 Git，基于 fork [`JackDance/WeKnora`](https://github.com/JackDance/WeKnora) 的 `0.9.0`（`main`），定制在分支 `jack/custom`。

| remote | 地址 |
| --- | --- |
| `origin` | `https://github.com/JackDance/WeKnora.git`（自己的 fork） |
| `upstream` | `https://github.com/Tencent/WeKnora.git`（官方） |

同步官方最新代码（在 `jack/custom` 上）：

```bash
git fetch upstream
git merge upstream/main
# 冲突多半在 docker-compose.yml，保住挂载 builtin_models、去掉本地库 depends_on、local-infra 三处即可
```

推送到自己的 fork：

```bash
git push -u origin jack/custom
```

本机 Mac 连远程库用公网 IP；若 WeKnora 和 Postgres/Redis 部署在**同一台**服务器，容器内请把 `.env` 的 `DB_HOST` / `REDIS_ADDR` 改成 `host.docker.internal`（公网 IP 在容器里常会超时）。

## 相对原仓库改了什么

没有改业务源码。改的是启动编排和配置。

### 1. `docker-compose.yml`

- `app` 挂载了 `./config/builtin_models.yaml:/app/config/builtin_models.yaml:ro`。启动时会按这个文件注册默认对话、向量、重排模型，`${变量}` 从容器环境变量替换。
- 当前连的 PostgreSQL 和 Redis 是远程服务（`8.159.128.87`），不是本机 Docker 里的 `WeKnora-postgres`、`WeKnora-redis`。因此 `app` 不再等待这两个本地容器的启动健康检查，`depends_on` 里只留了 `docreader`。
- `postgres`、`redis` 加了 profile `local-infra`。默认 `docker compose up -d` 不会启动这两个本地容器。

改回本地 Postgres、Redis 时，只加 profile 还不够：`app` 仍然不会等它们就绪。需要同时做三件事，然后重新启动：

1. 把 `.env` 里的 `DB_HOST`、`REDIS_ADDR` 改回本地容器地址（Compose 网络内分别是 `postgres` 和 `redis:6379`），不要继续指向远程 IP。
2. 删掉 `postgres`、`redis` 上的 `profiles: [local-infra]`，让默认启动带上它们。
3. 在 `app.depends_on` 里重新打开启动校验：

```yaml
depends_on:
  docreader:
    condition: service_healthy
  postgres:
    condition: service_healthy
  redis:
    condition: service_started
```

`postgres` 已有 `pg_isready` 健康检查，所以用 `service_healthy`。`redis` 目前没有 healthcheck，用 `service_started`，表示容器已启动即可。改完后在仓库根目录执行 `docker compose up -d`。

### 2. `config/builtin_models.yaml`

新建文件，三个默认模型，名字和地址都读 `.env`：

| 类型 | 环境变量 | 说明 |
| --- | --- | --- |
| KnowledgeQA | `LLM_*` | `provider=openai`，`LLM_BASE_URL` 必须带 `/v1`，应用会再拼 `/chat/completions` |
| Embedding | `EMBEDDING_*` | `provider=aliyun`，向量维度写成字面量 `1024` |
| Rerank | `RERANK_*` | `provider=aliyun`，`RERANK_BASE_URL` 用百炼原生完整路径，不要拼在 `/compatible-mode/v1` 后面 |

从 `.env` 删掉某条模型后，数据库里已经写入的模型不会自动删除。

### 3. `.env`

从 `.env.example` 复制后改过这些项：

- 宿主机端口：前端 `FRONTEND_PORT=18780`，后端 `APP_PORT=18781`。容器内后端仍是 `8080`，`APP_BACKEND_PORT` 保持默认，不要改成宿主机端口。
- PostgreSQL：`8.159.128.87:5433`，库名 `WeKnora`。密码里的 `@` 在 URL 里要写成 `%40`，写进 `.env` 时用原文。
- Redis：`8.159.128.87:6379`，无用户名。`REDIS_DB=2`，避开已被其他应用占用的逻辑库 `0`。
- 对象存储：`STORAGE_TYPE=oss`，地域 `cn-shanghai`，桶 `weknora-20261016`，前缀 `weknora/`。桶名 `weknora` 已被其他账号占用，不能用。
- `JWT_SECRET`、`SYSTEM_AES_KEY`、`SYSTEM_SIGNING_KEY` 已生成。`SYSTEM_AES_KEY` 不要再换，已加密的数据会解不开。
- Langfuse 的 `LANGFUSE_PUBLIC_KEY`、`LANGFUSE_SECRET_KEY`、`LANGFUSE_HOST` 已注释。两项密钥同时非空时会自动开启追踪。
- `SSRF_WHITELIST=8.159.128.87`。对话模型地址是 IP，不加白名单会被拒绝。

Ollama、OIDC、飞书、联网搜索、MCP、浏览器扩展、`APP_EXTERNAL_URL` 都还是空的，当前启动不需要。

## 远程 PostgreSQL 上做过的事

库在 Docker 容器 `d69f94cc909e` 里，PostgreSQL 18，宿主机端口 `5433`。扩展只在数据库 `WeKnora` 里创建，没有动同实例上的其他库。

- 安装 `postgresql-18-pgvector`，然后 `CREATE EXTENSION vector`。
- 安装 ParadeDB `pg_search` 0.26.2 的 deb，把 `shared_preload_libraries` 设为 `pg_search`，重启该 Postgres 容器后再 `CREATE EXTENSION pg_search`。

迁移跑完后 `schema_migrations` 应为 version `118`、`dirty=false`。

## Docker 启动

在仓库根目录：

```bash
docker compose pull
docker compose up -d
```

默认会起 `frontend`、`app`、`docreader`。PostgreSQL 和 Redis 走远程地址，本地这两个容器不会起，`app` 也不会做它们的启动健康检查。

改过 `.env` 或 `config/builtin_models.yaml` 后，环境变量是在创建容器时写入的，需要重建 `app`：

```bash
docker compose up -d --force-recreate --no-deps app
```

## 访问地址

- 前端：http://localhost:18780
- 后端健康检查：http://localhost:18781/health

## 改配置后仍异常时

- 对话报 direct IP address access is not allowed：确认 `SSRF_WHITELIST` 含模型 IP，并重建 `app`。
- 对话报 connection interrupted，日志里是 `invalid character '<'`：`LLM_BASE_URL` 少了 `/v1`，网关返回的是 HTML。
- 上传文件 403，错误码 `0003-00001403`：桶不属于当前账号。使用 `weknora-20261016`，并重建 `app`。
- 日志里 `model_catalog_configs does not exist`：迁移停在 version 2。先确认 `vector` 和 `pg_search` 都已在 `WeKnora` 库里启用，再重启 `app`。
- 日志里 `browser_devices` record not found：没有配对本地浏览器扩展，接口仍是成功，可以忽略。
