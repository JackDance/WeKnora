# 前端去 WeKnora 品牌化（UI 改动）

> 私有部署备忘（`jack/custom`）。产品官方文档在 `website-docs/`；本文只覆盖「看起来不像官方 WeKnora」的前端 UI 改动范围。

相关：

- 同目录部署与运维总览：[`README_JACK.md`](README_JACK.md)
- 改完 UI 后如何用源码构建镜像并在 Ubuntu 上跑起来：[`frontend-docker-build-and-run.md`](frontend-docker-build-and-run.md)

---

## 1. 目标与原则

**目标**：登录页、侧栏、浏览器标签等第一眼不再像官方 WeKnora；配色与文案更接近自有产品。

**原则**：

- 优先改「看得见的品牌层」，少动业务逻辑与路由。
- 定制打在 `jack/custom`，跟 `upstream/main` 时冲突多半在资源与文案文件。
- 保留仓库根目录 `LICENSE` / `THIRD_PARTY_NOTICES` 等法律文件；UI 去品牌不等于删开源声明。
- 环境变量前缀 `WEKNORA_*`、容器名、API 路径可先不动（改成本高，用户几乎看不见）。

---

## 2. 建议改动清单（按收益排序）

前端是 Vue 3 + Vite + TDesign，目录在 `frontend/`。

### P0：一看就知道是 WeKnora 的地方（必改）

| 项 | 位置 | 做什么 |
| --- | --- | --- |
| 浏览器标题 / SEO | `frontend/index.html` | `<title>`、`description`、`keywords` 改成自有产品名；去掉「WeKnora / 微信对话开放平台」表述 |
| Favicon | `frontend/public/favicon.ico` | 换成自有图标 |
| 登录页 Logo + GitHub 链 | `frontend/src/views/auth/Login.vue` | 现链到 `github.com/Tencent/WeKnora`，图是 `@/assets/img/weknora.png`；改成自有图，外链改成你们的文档/官网或去掉 |
| 侧栏 Logo | `frontend/src/components/menu.vue` | 同样引用 `weknora.png`，替换即可 |
| 品牌图资源 | `frontend/src/assets/img/weknora.png` | 直接换文件，或改引用路径指向新文件 |
| 登录页产品截图 | `screenshot-1.svg` … `screenshot-4.svg` | 官方产品示意，建议换成你们自己的界面图或拿掉轮播 |

### P1：文案里的「WeKnora」（强烈建议）

界面文案走 i18n，中文主文件：

- `frontend/src/i18n/locales/zh-CN.ts`
- 若开英文 / 日文等：同步改 `en-US.ts`、`ja-JP.ts`、`ko-KR.ts`、`zh-TW.ts`、`ru-RU.ts`

典型暴露点（可全文搜 `WeKnora`）：

- 助手名：「WeKnora 助手」→ 自有助手名
- 设置页：「WeKnora Cloud / CLI / Skill」等标题与说明
- 系统信息里「去 GitHub 提 Issue」链到官方仓库（`SystemInfo.vue`）
- 欢迎语、空状态、引导文案中的产品名

**做法**：全局搜 `WeKnora` / `weknora`，用户可见字符串优先改；代码标识符、测试用例、内部 env 名可后改。

### P2：视觉体系（让版式也不像默认绿配色）

| 项 | 位置 | 做什么 |
| --- | --- | --- |
| 品牌色 / 圆角 / 字体 | `frontend/src/assets/theme/theme.css` | 官方主色偏绿（`--td-brand-color-*` 约 `#07c05f`）。改成你们的主色阶，并视情况调 `--td-bg-color-page`、圆角、字体 |
| 主题 localStorage key | `frontend/index.html` 等处的 `WeKnora_theme` | 可改成自有 key，避免和官方同机共存时抢主题；改后旧用户主题会丢一次 |
| 设置里「WeKnora Cloud」入口 | `Settings.vue` / `WeKnoraCloudSettings.vue` | 不用云服务可隐藏菜单项；不建议大改组件内部协议 |

### P3：可选 / 低优先级

- 容器显示名 `WeKnora-frontend`（`docker-compose.yml`）：仅 `docker ps` 可见。
- 镜像名 `wechatopenai/weknora-ui`：部署时建议换成自有 tag，避免被官方 `pull` 盖掉（见 [`frontend-docker-build-and-run.md`](frontend-docker-build-and-run.md)）。
- `frontend/public/weknora-widget.js`：嵌入式 widget，若对外嵌页才需要改。
- 后端返回的产品名 / 邮件模板：属后端，本文不展开。

### 不建议一上来就改

- 路由 path、API 前缀、`WEKNORA_` 环境变量名（升级成本高）。
- 大面积重写 TDesign 组件结构（跟官方合并极痛）。
- 删除 `LICENSE` 或第三方声明。

---

## 3. 推荐实施顺序

1. 定产品显示名、主色、Logo、Favicon。
2. 换 `weknora.png`、favicon、`index.html` 标题。
3. 改登录页外链与侧栏 Logo。
4. 在 `zh-CN.ts`（及实际会用的语言）替换可见「WeKnora」。
5. 调 `theme.css` 品牌色。
6. 本地 `cd frontend && npm ci && npm run dev` 目测登录页 + 侧栏 + 标签页。
7. 再按 [`frontend-docker-build-and-run.md`](frontend-docker-build-and-run.md) 打 **linux/amd64** 前端镜像并部署到 Ubuntu。

改动提交到 `jack/custom`，不要直接推官方 `upstream`。

---

## 4. UI 验收清单

- [ ] 浏览器标签标题不是「WeKnora」
- [ ] Favicon 已换
- [ ] 登录页无官方 Logo / 无链到 `Tencent/WeKnora`
- [ ] 侧栏 Logo 为自有图
- [ ] 主色不再是默认绿（若做了 P2）
- [ ] 中文界面可见文案无「WeKnora」（或仅保留你刻意保留的）

---

## 5. 跟官方升级时的冲突点

```bash
git fetch upstream
git merge upstream/main
```

冲突常见于：`theme.css`、i18n、`Login.vue`、`menu.vue`、Logo 资源。合并后重新走一遍第 4 节验收。

---

## 6. 后续可扩展（本文未展开）

- 用 `VITE_*` 注入产品名，减少硬编码（需小改构建与少量源码）。
