# 前端白标改造（二开转售 · 与官方 WeKnora 拉开区分度）

> 私有部署备忘（`jack/custom`）。面向「基于 WeKnora 二开后卖给客户」：客户打开产品后，**第一眼不应认出这是官方 WeKnora 皮**。  
> 官方功能文档在 `website-docs/`；本文只谈**可见品牌层与视觉指纹**。

相关：

- 部署与远程库：[`README_JACK.md`](README_JACK.md)
- 改完 UI 后如何打前端镜像上 Ubuntu：[`frontend-docker-build-and-run.md`](frontend-docker-build-and-run.md)

---

## 0. 为什么「只换 Logo」不够

本地已起前端时（如 `http://localhost:5173`），未改造前典型指纹是：

| 一眼能认出来的点 | 现状（未改时） |
| --- | --- |
| 浏览器标签 | 标题 `WeKnora` |
| 侧栏顶栏 | `weknora.png` 字母 Logo「WEKNORA」 |
| 助手入口 | 「WeKnora 助手」 |
| 主色 | 微信绿系 `#07c05f`（按钮、选中态、强调） |
| 登录页 | 官方 Logo + 链到 `github.com/Tencent/WeKnora` / `weknora.weixin.qq.com` + 官方截图轮播 |
| 设置文案 | WeKnora CLI / Skill / Cloud 等大量产品名 |

客户或同行只要用过官方版，或对比过开源截图，**绿 + WEKNORA 字标 + 助手名** 三件套就能对上号。转售场景要把这套「视觉指纹」拆掉，而不是只换一张图。

**验收口诀（5 秒测试）**：把侧栏 Logo 遮住，给没接触过项目的人看登录页 + 知识库列表 5 秒——若仍有人说「这是 WeKnora / 腾讯那个」，改造未达标。

---

## 1. 目标、边界与合规

### 目标

1. **品牌可替换**：显示名、Logo、Favicon、助手名、空状态文案全部可换成客户/自有品牌。
2. **视觉有区分度**：主色、页面底、圆角/密度与官方默认绿皮明显不同。
3. **可跟官方升级**：尽量改资源、CSS 变量、i18n、少量 Vue 模板；少动路由与 API。

### 原则

- 优先改「用户看得见的层」；`WEKNORA_*` 环境变量、API path、容器内部名可后置（成本高、客户几乎看不见）。
- 定制落在 `jack/custom`，不要推 `upstream`。
- **保留**仓库 `LICENSE`、`THIRD_PARTY_NOTICES` 等法律文件；UI 白标 ≠ 删除开源声明。若向客户交付闭源商业壳，需自行确认与上游许可证的合规义务（本文不替代法务意见）。

### 不在本文范围

- 后端邮件模板、OpenAPI `title`、CLI 帮助文案（可另开清单）。
- 大面积重写 TDesign 页面结构（跟 `upstream/main` 合并会极痛）。

---

## 2. 品牌包（当前 `jack/custom` 已冻结）

本分支默认 Brand Kit（可按客户再 fork 覆盖）：

| 字段 | 当前取值 |
| --- | --- |
| 产品显示名 | **睿库** / **RuiBase** |
| Logo | `frontend/src/assets/img/brand-logo.svg`（青石蓝标 + RuiBase 字标） |
| Favicon | `frontend/public/favicon.svg` |
| 主色 | 方案 A 青石蓝 `#1B6B9A`（偏蓝；不是微信绿，也不是青绿 `#0F766E`） |
| 页底 / 侧栏 | `#F4F7FA` / `#F7F9FB` |
| 助手显示名 | **睿库助手**（i18n） |
| 主题 localStorage | `RuiBase_theme` / `RuiBase_${userId}_*`（兼容读旧 `PharmMind_*` / `WeKnora_*`） |
| 外链 | 登录页已去掉官方 GitHub / weknora.weixin.qq.com |
| 登录页示意 | 已移除官方 `screenshot-*` 轮播 |
| 云服务菜单 | 设置里默认隐藏 WeKnora Cloud 入口 |

给其他客户交付时，复制一份 Brand Kit 到 `docs/jack/brand/<客户名>.md`（勿提交密钥），再改 Logo / 色 / i18n 显示名即可。

**配色备选（若客户不要青石蓝）**：

- 墨蓝 + 琥珀：`#1E3A5F` + `#D97706`
- 钢青灰：`#334155` + 链接 `#0E7490`
- **避开**：`#07c05f` 官方绿、亮紫渐变 AI 皮

改主色后检查：主按钮、侧栏选中、Tab、链接、focus 环（均吃 `--td-brand-color-*`）。

---

## 3. 改造分层（按「被认出概率」排序）

前端：Vue 3 + Vite + TDesign，根目录 `frontend/`。

### L0 · 品牌壳（必做，否则 5 秒测试必挂）

| 项 | 位置 | 做什么 |
| --- | --- | --- |
| 浏览器标题 / SEO | `frontend/index.html` | `<title>`、`description`、`keywords` 改为自有产品；去掉「WeKnora / 微信对话开放平台」 |
| Favicon | `frontend/public/favicon.ico` | 换成自有图标 |
| 侧栏 Logo | `frontend/src/components/menu.vue`（`@/assets/img/weknora.png`） | 换资源或改路径；`alt` 改为产品名 |
| 品牌图文件 | `frontend/src/assets/img/weknora.png` | 直接覆盖或新增 `brand-logo.png` 并改引用 |
| 登录页 Logo + 外链 | `frontend/src/views/auth/Login.vue` | 现链 `github.com/Tencent/WeKnora`、`weknora.weixin.qq.com`；改为自有文档/官网或删除；Logo 换图 |
| 登录页轮播图 | `screenshot-1.svg` … `screenshot-4.svg` | 换成自有界面截图，或关闭/删除轮播区块 |
| 主题 localStorage key | `index.html` 等处的 `WeKnora_theme` | 改为自有 key（如 `AcmeKB_theme`），避免与官方同机抢主题 |

### L1 · 文案去官方名（强烈建议，转售必做）

界面文案走 i18n，中文主文件：

- `frontend/src/i18n/locales/zh-CN.ts`（当前约 **70+** 处含 `WeKnora`）
- 实际交付语言同步：`en-US.ts`、`zh-TW.ts`、`ja-JP.ts`、`ko-KR.ts`、`ru-RU.ts` 等

优先替换用户可见字符串（全局搜 `WeKnora` / `weknora`）：

| 暴露点 | 建议 |
| --- | --- |
| 「WeKnora 助手」及欢迎语 | 改为 Brand Kit 助手名 |
| 设置：WeKnora CLI / Skill / Cloud | 改名，或对客户版隐藏整块入口 |
| 系统信息「去 GitHub 提 Issue」 | `SystemInfo.vue`：改为你们的工单/文档，勿指官方仓 |
| 自动化 / 权限 / 嵌入说明里的产品名 | 统一替换 |
| 空状态、引导、搜索「产品文档」类文案 | 换成客户文档站或删除 |

**可后改**：测试用例、内部模块名、`weknora_pending_invite_token` 等 localStorage key（改 key 会丢一次本地状态）。

### L2 · 视觉体系（决定「像不像换皮绿版」）

| 项 | 位置 | 做什么 |
| --- | --- | --- |
| 品牌色 / 页面底 / 字体 | `frontend/src/assets/theme/theme.css` | 重写 `--td-brand-color-1`～`10`、`--td-bg-color-page`；字体可换成品牌指定栈（勿只改一两个色阶） |
| 亮色页底 | 默认偏 `#eee` 灰底 | 可改为更干净的近白或浅冷灰，改变「官方默认」观感 |
| 圆角与密度 | `theme.css` 中 radius / 组件覆盖 | 略增大或略收紧圆角，避免与官方截图像「同一套皮肤」 |
| 侧栏选中态 | 依赖 brand 色 | 主色换完后重点回归知识库列表选中、主按钮 |
| 设置「WeKnora Cloud」 | `Settings.vue` / `WeKnoraCloudSettings.vue` | 客户不用官方云则**隐藏菜单**；不必改协议实现 |

带 `-green` 的 SVG 图标（如 `*-green.svg`）若写死了绿色，主色大改后可能「按钮绿、图标还是旧绿」——抽查侧栏/知识库图标，必要时改为 `currentColor` 或重导出。

### L3 · 交付与嵌入（按客户合同选做）

| 项 | 说明 |
| --- | --- |
| 前端镜像名 | 勿长期用 `wechatopenai/weknora-ui:latest`；见 [`frontend-docker-build-and-run.md`](frontend-docker-build-and-run.md) |
| `frontend/public/weknora-widget.js` | 对外嵌页时改名/改注释中的产品标识 |
| `embed.html` | 嵌入场景标题与 branding |
| 容器显示名 | `WeKnora-frontend` 仅 `docker ps` 可见，可改 compose |

### 不建议一上来就改

- 路由 path、API 前缀、`WEKNORA_` 环境变量名。
- 为「不像」而整页重做信息架构（维护与升级成本暴涨）。
- 删除许可证与第三方声明文件。

---

## 4. 推荐实施顺序（一条流水线）

1. **冻结 Brand Kit**（名、色、Logo、助手名、外链）。
2. 换 `favicon`、`weknora.png`（或新路径）、`index.html` 标题与 SEO。
3. 改 `Login.vue` 外链与轮播；改 `menu.vue` Logo。
4. 在 `zh-CN.ts`（及交付语言）批量替换可见 `WeKnora`；隐藏不需要的 Cloud/CLI/Skill 入口。
5. 重做 `theme.css` 色阶 + 页底；抽查 `-green` 图标。
6. 本地验收：

   ```bash
   cd frontend && npm ci && npm run dev
   ```

   必看：登录页 → 侧栏 → 知识库列表（`/platform/knowledge-bases`）→ 助手抽屉标题 → 浏览器 Tab。
7. 按 [`frontend-docker-build-and-run.md`](frontend-docker-build-and-run.md) 打 **linux/amd64** 自有 tag 镜像交付。
8. 提交到 `jack/custom`（或按客户分 branch / 目录的 brand overlay）。

### 后续增强（减少多客户硬编码）

- 用 `VITE_APP_NAME` / `VITE_ASSISTANT_NAME` / `VITE_BRAND_COLOR` 等注入显示名与主色，构建时按客户打不同包（需小改 `index.html`、i18n 入口与 theme）。多客户交付时优先做这一步，避免在 `zh-CN.ts` 里来回改死字符串。

---

## 5. 验收清单（转售交付用）

### 5 秒视觉

- [ ] 浏览器 Tab 标题不是 `WeKnora`
- [ ] Favicon 已换
- [ ] 侧栏 Logo 无官方「WEKNORA」字标
- [ ] 主色不再是 `#07c05f` 同系默认绿（或已明显偏离）
- [ ] 登录页无官方 Logo、无 `Tencent/WeKnora` / `weknora.weixin.qq.com` 产品外链
- [ ] 登录页无官方 `screenshot-*.svg` 示意（或已换成自有图）

### 文案

- [ ] 助手入口名称不是「WeKnora 助手」
- [ ] 中文界面用户可见文案无「WeKnora」（刻意保留的合规声明除外）
- [ ] 设置里无「去官方 GitHub 提 Issue」；Cloud/CLI/Skill 已按客户需求隐藏或改名

### 交付

- [ ] 前端镜像为自有仓库/tag，不是会被官方 `pull` 覆盖的名字
- [ ] 做过一次「遮住 Logo 的 5 秒测试」

---

## 6. 跟官方升级时的冲突点

```bash
git fetch upstream
git merge upstream/main
```

冲突高发：`theme.css`、i18n、`Login.vue`、`menu.vue`、Logo / screenshot 资源、`index.html`。合并后必须重新跑第 5 节清单，不要假设「代码能编过 = 品牌还在」。

---

## 7. 本地对照（改造前后）

未改前可在本机开发服快速对照指纹：

- 前端开发：`http://localhost:5173`（`make dev-frontend`）
- 知识库：`/platform/knowledge-bases`
- 登录页：`/auth/login`

改造后应用**同一路径**再走一遍第 5 节；不要只看 Docker 旧镜像。

---

## 8. 与「仅私有部署自用」的差异

| | 自用私有化 | 二开转售客户 |
| --- | --- | --- |
| 换 Logo / 标题 | 建议 | **必须** |
| 去文案里的 WeKnora | 建议 | **必须** |
| 换主色体系 | 可选 | **强烈建议**（否则像换皮） |
| 藏 Cloud / 官方 GitHub | 按需 | **建议默认藏** |
| 多客户 Brand Kit / `VITE_*` | 通常不需要 | **建议尽早做** |
