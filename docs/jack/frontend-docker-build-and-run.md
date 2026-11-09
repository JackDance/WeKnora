# 前端源码构建 Docker 镜像并在 Ubuntu 上运行

> 私有部署备忘（`jack/custom`）。在改完前端 UI 之后，如何用源码打出镜像、放到 Ubuntu，以及如何只重建前端容器。

相关：

- UI 去品牌化改什么：[`ui-rebrand.md`](ui-rebrand.md)
- 远程库 / 跳过本地 Postgres·Redis 等：[`README_JACK.md`](README_JACK.md)

---

## 1. 背景

官方 `docker-compose.yml` 里前端服务大致为：

```yaml
frontend:
  image: wechatopenai/weknora-ui:${WEKNORA_VERSION:-latest}
  build:
    context: ./frontend
```

`frontend/Dockerfile` 是多阶段构建：Node 编译 Vite 静态资源 → Nginx 托管。

- **静态资源与 CPU 架构无关**，但运行基础镜像要匹配服务器（Ubuntu 一般为 `linux/amd64`）。
- **只改前端时不必重打** `app` / `docreader`；Postgres / Redis 继续用现有远程实例。

---

## 2. 镜像命名建议

不要长期只用官方名 `wechatopenai/weknora-ui:latest`，否则以后 `docker compose pull` 可能把定制冲掉。

建议：

```text
jackdance/weknora-ui:custom
# 或带日期 / git short sha
jackdance/weknora-ui:20261103-3f91673
```

并在 compose 的 `frontend.image` 指到该 tag（见第 4 节）。

---

## 3. 构建镜像

### 3.1 方案 A：在 Ubuntu 服务器上构建（推荐）

本机磁盘紧、且目标机就是部署机时，在服务器构建最省事。

```bash
# 服务器上：代码已是含 UI 改动的 jack/custom
cd ~/projects/WeKnora
git checkout jack/custom
git pull origin jack/custom

# 国内可加 npm 镜像
export NPM_REGISTRY=https://registry.npmmirror.com

docker compose build frontend
# 或显式指定 tag：
docker build --platform linux/amd64 \
  --build-arg NPM_REGISTRY="${NPM_REGISTRY}" \
  --build-arg VITE_FRONTEND_COMMIT="$(git rev-parse --short HEAD)" \
  -t jackdance/weknora-ui:custom \
  -f frontend/Dockerfile frontend/
```

### 3.2 方案 B：本机 Mac 构建 amd64，再拷到服务器

Mac（尤其 arm64）必须指定平台：

```bash
cd ~/Desktop/Programs/WeKnora

NPM_REGISTRY=https://registry.npmmirror.com \
docker build --platform linux/amd64 \
  --build-arg NPM_REGISTRY=https://registry.npmmirror.com \
  --build-arg VITE_FRONTEND_COMMIT="$(git rev-parse --short HEAD)" \
  -t jackdance/weknora-ui:custom \
  -f frontend/Dockerfile frontend/

# 导出到外置盘（避免占满系统盘）
docker save -o /Volumes/Jack/docker_images/weknora-ui-custom-linux-amd64.tar \
  jackdance/weknora-ui:custom
```

**注意**：`docker build` 的层仍会进 Docker Desktop 虚拟盘；本机空间不够时优先用方案 A。

服务器加载：

```bash
docker load -i weknora-ui-custom-linux-amd64.tar
docker images | grep weknora-ui
```

---

## 4. 在 Ubuntu 上运行 / 替换前端容器

### 4.1 确认当前栈

```bash
cd ~/projects/WeKnora
docker compose ps
# 应仍是 frontend / app / docreader；不要加 --profile local-infra
```

### 4.2 更新代码（若在服务器上构建）

```bash
git fetch origin
git checkout jack/custom
git pull origin jack/custom
```

### 4.3 让 compose 使用自定义镜像

任选其一：

**方式 1**：改 `docker-compose.yml` 的 `frontend.image`（记在 `jack/custom`）：

```yaml
frontend:
  image: jackdance/weknora-ui:custom
```

**方式 2**：不改 compose 文件，构建后打成官方同名 tag（简单但不安全）：

```bash
docker tag jackdance/weknora-ui:custom wechatopenai/weknora-ui:latest
```

之后务必避免对 frontend 执行盲目 `docker compose pull`。

### 4.4 启动 / 重建前端

```bash
# 仅重建前端，不动 app
docker compose up -d --no-deps --force-recreate frontend

curl -I http://127.0.0.1:18780
# 浏览器强刷或清缓存后看 Logo / 标题
```

### 4.5 回滚

```bash
# 若仍留着官方层，可 tag 备份后再切回
docker tag wechatopenai/weknora-ui:latest jackdance/weknora-ui:backup
# 或重新 load 之前的 tar
docker compose up -d --no-deps --force-recreate frontend
```

---

## 5. 和现有部署的衔接

| 项 | 说明 |
| --- | --- |
| 端口 | 前端宿主机端口由 `.env` 的 `FRONTEND_PORT`（当前常用 `18780`） |
| 后端 | `APP_HOST=app`，前端 Nginx 反代到 app；只换 UI 镜像即可 |
| DB / Redis | 继续远程；服务器上用 `host.docker.internal`（见 [`README_JACK.md`](README_JACK.md)） |
| OSS / 模型 | 与前端镜像无关 |

---

## 6. 部署验收清单

- [ ] `docker images` 中前端为自有 tag（或确认不会被官方 pull 覆盖）
- [ ] `docker compose ps` 中 frontend 健康
- [ ] `http://服务器:18780` 能打开，且 UI 改动已生效（强刷缓存）
- [ ] 未误 `pull` 回官方 `wechatopenai/weknora-ui:latest` 盖掉定制

---

## 7. 后续可扩展（本文未展开）

- 单独 `docker-compose.override.yml` 只覆盖 `frontend.image`，少改主 compose。
- CI：在 `jack/custom` 推送时自动 `build --platform linux/amd64` 并传到私有镜像仓库，服务器只 `pull` 自有仓库。
