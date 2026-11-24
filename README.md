# WorkBuddy 专家包合集（feiyucp）

本仓库同时收录 **2 个** 专家包，一眼看全：

| 专家 | 名称 | 位置 |
|---|---|---|
| 页匠 | 网页复刻26年11月23日版本 | 本仓库根目录（下方即完整说明） |
| 博客文章复刻 | 博客文章复刻26年11月24日版本 | [`blog-to-elementor-20261124/`](./blog-to-elementor-20261124/) |

---

# 专家一：页匠 · 网页复刻26年11月23日版本（web-to-elementor）

把你给的网页批量变成 WordPress 后台里**真人可以用 Elementor 手工编辑**的新页面：图片下载后上传媒体库，页面从第一版起就同时适配电脑、平板、手机三种设备。

## 类型

Agent 型（单个 AI 专家）· 分类 `02-Engineering`

## 能做什么

- **一次给几个网页都行**：每个网页生成一个独立新页面，不覆盖已有页面。
- **文件存哪都行**：网页下载/保存到你自己的任意目录（桌面、下载文件夹、D 盘……），直接把 `.html` 文件发给我或告诉我完整路径即可，不用挪到任何指定文件夹。
- **图片本地化**：从你发来的 HTML 里解析图片，外链直接下载、随附 `_files` 直接取用，上传目标站媒体库，页面用媒体库 ID 引用，不留外站链接。
- **真人可编辑**：用 Elementor 原生元素（`e-flexbox` / `e-heading` / `e-paragraph` / `e-image` 等）搭建，编辑器里逐层可点选、可改字改图，不是纯代码页。
- **响应式一次到位**：建页时就把桌面 / 平板 / 手机三档断点写进样式，发布即三端可用，不另留「再调响应式」这一步。
- **目标站点可换**：默认 `testdecor`，你给新地址就按新站做，流程不变。
- **真实证据验收**：前台 200、三端横向溢出 0、外站图片残留 0、结构树可编辑，逐项实测后才算完成。

## 怎么用（三步）

1. 打开 WorkBuddy 左侧栏 **专家** → **我的专家** → 找到「页匠 / 网页复刻26年11月23日版本」→ 开始对话。
2. **把网页交给我**：网页你想下载/保存到哪个目录都行，不用管我、不用挪文件夹。真实 Chrome 打开网页 → `Ctrl+S` → 保存类型选「网页，全部」或「网页，仅 HTML」→ 得到 `.html` 文件（前者还会多一个同名 `_files` 文件夹）→ **直接把这个 `.html` 文件在对话里发给我**（或把完整路径告诉我）即可。一次发几个都行；`_files` 一起发来或告知路径更好，没有也不影响我先开工。
3. 说一句需求，例如：「我把 3 个网页的 HTML 发给你了，复刻成 testdecor 上的新页面，图片传媒体库，直接做好三端响应式」。我会先列清单跟你确认，再逐页建、逐页验，最后给汇总表（页面 ID / 前台链接 / 后台编辑链接 / 三端实测结果）。

> 如果目标站有反爬验证（例如 Vercel Security Checkpoint），自动抓取会被拦，用第 2 步手动保存 HTML 再发我即可——这是已验证可用的通道。若 HTML 里图片是相对路径、且同名 `_files` 的图片实体确实没给到，我会一次性跟你补要，但不会让你为此整理目录。

## 试试这样问我

- 我把几个网页的 HTML 文件发给你，复刻成 testdecor 里可手工编辑的 Elementor 页面，图片传媒体库，直接做好三端响应式
- 把这几个网页做成新页面，目标站点换成我给你的新地址，同样一次做成响应式
- 检查我后台这批新页面的响应式，确认手机、平板、桌面三端都不横向溢出

## 随附资料

| 文件 | 内容 |
|---|---|
| `skills/web-to-elementor-pipeline/SKILL.md` | 完整流水线 SOP |
| `references/responsive-recipe.md` | 三档响应式参数配方（实测数值，可直接复制） |
| `references/troubleshooting.md` | 踩坑手册（抓取通道、代理、headless 504px、overflow-x 假阴性等） |
| `references/target-site-config.md` | 目标站点配置与切换 |
| `templates/measure-iframe.html` | 三端同源 iframe 实测页 |
| `templates/batch-intake.md` | 批量任务清单模板 |
| `scripts/verify_front.py` | 前台图片本地化与可访问性核验 |

## 边界

本地测试站可作一比一还原底稿；**正式对外上线前**请替换成自有或已获授权的文字与图片。

## 头像

头像位于 `avatars/expert.png`（ImageGen 生成）。可手动替换：PNG/JPG，512×512，≤500KB。

## 安装位置

```
C:\Users\Administrator\.workbuddy\plugins\marketplaces\my-experts\plugins\web-to-elementor\
```

注册后出现在「专家 → 我的专家」。分享打包：

```bash
python3 scripts/package_expert.py <expert-dir>
```

---

# 专家二：博客文章复刻26年11月24日版本（blog-to-elementor）

把博客文章的网页 HTML 复刻成 WordPress 后台里**真人可以用 Elementor 手工编辑**的一篇**文章（post，不是页面）**：还原标题、**摘要（excerpt）**、发布日期、分类、标签、特色图片；图片上传媒体库；从第一版起就适配电脑 / 平板 / 手机三端。

- **专家包目录**：[`blog-to-elementor-20261124/`](./blog-to-elementor-20261124/)
- **完整说明**：[README.md](./blog-to-elementor-20261124/README.md)
- **与页匠的区别**：页匠产出的是**页面（page）**，本专家产出的是**文章（post）**，并额外还原摘要 / 日期 / 分类 / 标签 / 特色图。
- **2026-11-24 更新**：新增摘要（excerpt）必写流程——从源网页 `description` / `og:description` 取纯文本写入 WordPress `excerpt`，发布后回读复核；编辑器右侧不再空着 `Add an excerpt...`。

## 怎么用

WorkBuddy 左侧栏 **专家** → **我的专家** → 找到「博客文章复刻26年11月24日版本」→ 把博客文章的 `.html`（存任意目录即可）发进对话，说一句需求即可。
