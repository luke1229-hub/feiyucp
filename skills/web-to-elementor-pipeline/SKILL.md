---
name: web-to-elementor-pipeline
description: "Verified end-to-end pipeline for batch-replicating user-supplied HTML files (from any local folder, attached directly in chat or given by absolute path) into hand-editable Elementor V4 pages on a WordPress site: parse the received HTML, resolve images from external URLs / companion _files / data URIs, upload them into the media library via Elementor MCP tokens, build native atomic elements with three-device responsive breakpoints written inline, publish, and verify with real browser measurements. Use when replicating pages, switching the target WordPress site, or auditing responsive output."
---

# 网页 → Elementor 可编辑响应式页面 · 流水线手册

> 本手册全部来自 2026-11-23 在 `testdecor` 上的**实跑验证**，不是理论推演。凡标「实测」的结论都有状态码、像素值或截图佐证；标「坑」的是踩过并已定位根因的陷阱。

## 0. 这套流水线解决什么

用户一次给几个网页 → 每个网页变成一个独立的新 WordPress 页面 → 图片进媒体库 → 页面从第一版就适配电脑/平板/手机 → 页面在 Elementor 编辑器里能逐层点开改。

**五条铁律**
1. 源网页由用户提供，不自己去公网找。
2. **不挑存放目录**：用户下载/保存到自己任意目录都行，直接把 `.html` 发进对话或给完整路径即可，绝不要求挪进 `captured/` 等指定目录。
3. **布局忠实原版（用户硬要求，最高优先级）**：内容可以差一点，但布局必须尽量和原版一致——分栏、区块顺序、图片尺寸/比例、卡片列数、字号层级都贴合原版。用户要在原版式样基础上手改，布局一变就没法改。任何布局决策来自真实渲染+像素测量，不靠猜。见 `references/layout-fidelity.md`。
4. 响应式在建页时一次写入，不留「事后调整」步骤。
5. 页面必须是 Elementor 原生可编辑元素，不是纯代码页、不是 3.x JSON 盲写。

## 1. 输入通道（第一步，已换过两次方案）

原计划用 `vercel-labs/agent-browser` 无头抓取，实测在目标站被 Vercel Security Checkpoint 拦截（「代码 21」），无头/有头/复用真实 Chrome profile 全部失败；内置浏览器与 WebFetch 能过验证，但只给精简 DOM 或 markdown 正文，**导不出完整 HTML+CSS+图片**。

因此第一步改为「用户保存网页 → 把文件直接给我」，且**用户爱存哪存哪**：

| 通道 | 适用 | 产出 | 结论 |
|---|---|---|---|
| **用户把下载的 `.html` 直接发进对话**（或给出任意本地绝对路径） | **默认且推荐**，含受验证墙保护的站点 | 完整 HTML；图片按来源分别取 | ✅ 一个文件即可开工、绕开验证墙，图片按来源自检；**不要求任何指定目录** |
| 用户 `Ctrl+S`「网页，全部」→ `.html` + 同名 `_files` | 图片实体最齐的场景 | 完整 HTML、CSS、图片、JS | ✅ 三样齐全；`_files` 一起发来或给路径即可，仍不用挪目录 |
| 内置浏览器 / WebFetch | 无防护站点、快速取文案 | 精简 DOM / markdown | ⚠️ 拿不到 CSS 与图片 URL，不能单独支撑建页 |
| `curl` / Node 直连 | 公网普通站点 | 原始 HTML | ⚠️ 沙箱代理常拦成 HTTP 000 / 超时 |
| agent-browser 自动化 | 无 Vercel 类防护的站点 | `get html` / `get styles` | ❌ 对目标站不可用，别硬扛 |

**给用户的话术（别说「丢进 captured/」这类黑话，也别指定目录）**：
> 网页你想下载/保存到哪个目录都行，桌面、下载文件夹、D 盘都可以，不用管我。存好之后直接把那个 `.html` 文件在这条对话里发给我（或者把完整路径告诉我），我就能接着做。一次发几个都行。如果保存时还多出来一个同名的 `_files` 文件夹，一起发我或告诉我路径（图片实体在里面）；只发 HTML 也行，我先自检，真缺图片实体时才找你补要，不会让你为此整理目录。

**Chrome 保存方式**（用户问才说）：打开网页 → `Ctrl+S` → 保存类型选「网页，全部」或「网页，仅 HTML」→ 得到 `.html`（前者另有同名 `_files`）。

### 收到文件后的输入自检（关键：保证「只发 HTML 也能接着做」）

定位用户发来的 `.html` —— 对话附件，或任意本地绝对路径（不要求用户整理、移动、复制到任何指定目录）。然后扫描 `<img src>` / `srcset` / CSS `url()`，判断图片来源并分别处理：

| 图片来源形态 | 处理方式 |
|---|---|
| 绝对外链 `http(s)://...` | 直接下载到本地（代理/超时按第 7 节处理） |
| 相对路径 `xxx_files/foo.png` | 在该 HTML **同目录**的同名 `_files` 里找实体；用户已一并发来或给了路径就直接读。**必须先做 HTML 实体解码**（`html.unescape`）：src 里的 `&` 存的是 `&amp;`，不解码会 0 命中（2026-11-23 实测：不解码 0/8，解码后 8/8） |
| `data:` / base64 内联 | 直接解码落盘 |
| 用户发来 **zip 包**（整页文件夹打包） | 解压后按上述规则处理；已实测整链路可行：打包 7.4 MB → 解压 52 文件 → 8/8 图片实体全命中 |
| 上述相对路径**且 `_files` 实体确实拿不到** | 一次性请用户把那个 `_files` 文件夹发来或告知路径；**不得**要求用户为此整理目录或挪文件，也不把它设为前置条件 |

原则：**能自己解决的自己解决，只在真缺实体时问一次**。一个 `.html` 即可启动流程并完成来源判定；Chrome「网页，全部」保存的 HTML 常见形态是图片全为相对路径（2026-11-23 实测某页 8 张图全是），此时补要 `_files` 是取图片实体，不是重新设置门槛，也绝不要求用户挪动或整理目录。

**验收**：本机 Chrome 打开 `file://` 能完整渲染；HTML 里有完整 `<h1>`/`<p>`；图片实体可取（外链可下载、`_files` 有真 PNG、或 data URI 可解码，不是占位图）。

## 2. 解析网页

- 先做第 1 节的输入自检，图片来源判定完成后再解析 DOM。
- **先还原原版真实布局（强制，尤其复杂页面）**：把 HTML 去掉脚本做成静态副本、图片指向已上传的本站附件、放站点根用 `http://` 打开并截图；注入测量脚本读每个标题/图片/区块的真实像素尺寸（内容宽度、分栏、字号、图片固定尺寸、卡片列数、区块顺序）。这是建页的唯一布局依据。**完整方法与 NatureTough 实测尺寸基线见 `references/layout-fidelity.md`。**
- 只取正文主体：`<main>` 或文章容器，**排除**页眉、页脚、全局联系表单、导航、Cookie 提示。
- 按 DOM 顺序记录：面包屑、主标题、副标题、引言、正文段落、图片、正文段落……直到结尾。
- 记录版式基线（今天实测的 SPC Wall Panels 页）：
  - 内容区 `max-width: 900px` 居中
  - H1 `38px` `#1a1a1a`，`font-weight:700`，`line-height:1.25`
  - 副标题 `16px`，正文 `15px` `#4a4a4a`，`line-height:2`
  - 图片 `16/9`，底色 `#f0ebe3`
  - 面包屑 `13px` `#888888`
- 文案按源语言逐字保留（通常英文），不改、不翻译、不润色。**但与布局冲突时优先保布局**——不为塞文案改动原版骨架。
- **图片尺寸用实测固定值**（如色卡 89×89、配件图 182×182、主图 520×520），不要一概 `width:100%` 撑满；分栏方式、卡片列数、区块顺序严格对齐测量数据。这是"图片太大突兀 / 布局失真"的根治办法。

## 3. 图片上传媒体库（第二步）

用 Elementor MCP，不用外部下载技能：

1. `elementor/request-media-upload` 为每张图申请一次性 token。
2. `curl --noproxy '*' -X POST` 把本地 PNG 传到
   `http://<站点>/index.php?rest_route=/elementor/v1/mcp-media-upload`
   带上 `X-Elementor-Mcp-Upload-Token` / `Filename` / `Title` 头。
3. 确认 **HTTP 201**，记录返回的附件 ID。实测得到 ID 30–33，URL 形如
   `http://testdecor/wp-content/uploads/2026/11/spc-bathroom-1.png`。
4. WordPress 会自动生成 `1024x683` / `768x512` / `300x200` / `150x150` 等缩略图，前台 `srcset` 自动生效。

**环境坑**
- Windows 的 `curl` 是 `C:\windows\system32\curl`，**不认** Git Bash 的 `/e/...` 路径，传 Windows 路径。
- 本地站点必须加 `--noproxy '*'`，否则被沙箱代理拦成 HTTP 000。
- 非 HTTPS 本地站若提示 Application Password 不可用，在 `wp-config.php` 加 `define('WP_ENVIRONMENT_TYPE','local')`。

## 4. 建页：原生元素 + 响应式一次写入（第三步，核心）

用 `elementor/build-composition`，**不要**用 `html2elementor`（产出 3.x 容器 JSON，本站 V4 Atomic Editor 兼容性未验证）。

结构形状（今天实测的 11 元素树）：
```xml
<e-flexbox configuration-id="页面主体">
  <e-div-block configuration-id="内容容器">
    <e-paragraph configuration-id="面包屑"></e-paragraph>
    <e-heading configuration-id="主标题"></e-heading>
    <e-paragraph configuration-id="副标题"></e-paragraph>
    <e-paragraph configuration-id="引言"></e-paragraph>
    <e-image configuration-id="图片1"></e-image>
    <e-paragraph configuration-id="正文A"></e-paragraph>
    <e-image configuration-id="图片2"></e-image>
    <e-paragraph configuration-id="正文B"></e-paragraph>
    <e-image configuration-id="图片3"></e-image>
    <e-paragraph configuration-id="结论"></e-paragraph>
  </e-div-block>
</e-flexbox>
```
- 每个标签都要有唯一 `configuration-id`，且它会成为编辑器结构面板里的层名，用人类可读的中文命名。
- 图片配置：`{"image":{"src":{"id":30},"size":"full"}}` —— 用**媒体库 ID**，不要 URL。
- 断点写法用 Elementor 原生 `@media(--tablet)` / `@media(--mobile)`，**不要**写 `@media (max-width: 768px)` 像素查询。
- 建完后 `elementor/get-page-structure` 复核树完整、`staged` 状态、图片 ID 正确。

后续若需微调，用 `elementor/manage-elements` 批量 update，`style_apply_mode: "replace"` 整段重写样式。**写入后是暂存草稿，必须 `elementor/publish-document` 才生效。**

完整三档参数见 `references/responsive-recipe.md`。

## 5. 发布与验收（第四步）

`elementor/publish-document` → 拿前台 URL、后台 `wp-admin/post.php?post=<ID>&action=elementor`、预览链接。

每个页面至少验：
| 检查 | 合格标准 |
|---|---|
| 前台 HTTP | 200 |
| 图片来源 | 全部 `wp-content/uploads`；外站 CDN 残留数 = 0 |
| 结构树 | 原生 `e-` 元素，`staged:false` |
| 三端横向溢出 | 1440 / 768 / 390 全部 `0px` |
| 三端实测值 | 字号、行高、图片尺寸符合配方 |
| 截图 | 文字不被硬截断、图片不裁切变形 |

批量任务输出汇总表：页面 ID / 标题 / 前台 URL / 后台编辑 URL / 图片数 / 三端溢出结果。

## 6. 换目标站点

默认 `testdecor`（`http://testdecor`，MCP 连接 `testdecor-elementor`）。用户给新地址时：
1. 确认新站的 WordPress 地址与 Elementor MCP 连接可用（先 `elementor/list-posts` 探活）。
2. 新连接的建立方式见 `references/target-site-config.md`。
3. 其余 Phase 1–5 完全不变，响应式照常一次写入。
4. 不把站点地址、页面 ID、附件 ID 写死进流程；每次从工具返回值取。

## 7. 验证工具与陷阱

- 三端实测用同源 iframe 测量页：`templates/measure-iframe.html`（放到目标站根目录，用 Chrome headless `--dump-dom` 读 `RESULT_JSON`）。
- 前台快速核验：`scripts/verify_front.py`。
- 陷阱全清单见 `references/troubleshooting.md`，尤其：
  - Chrome headless `--window-size=390` 被钳到 **504px** CSS 视口，却按 390px 画布截图 → 假「文字被切断」。真手机截图要用 390px iframe + ≥600px 窗口截容器页再裁左 390px。
  - 站点 `body` 常有 `overflow-x:clip` → `documentElement.scrollWidth` 假阴性。要遍历元素比 `scrollWidth > clientWidth` 或 `getBoundingClientRect().right > 视口`。
  - 中文固定链接曾把 `?page_id=8` 301 到 404：把 `wp_options.permalink_structure` 置空可解。
  - **只发 HTML 时的图片实体**：Chrome「网页，全部」会把 `<img src>` 改写成指向同名 `_files` 的相对路径。先判来源再决定要不要补要文件夹，别默认要求用户重新保存或整理目录；「网页，仅 HTML」通常保留外链，直接下载即可。

## 8. 版权边界

本地测试站作还原底稿没问题；**正式对外上线前**必须换成自有或已获授权的文字与图片，并主动提醒用户。

## References

- `references/layout-fidelity.md` — **布局忠实原版铁律 + 真实渲染测量还原方法 + 实测尺寸基线（最高优先级，动手前必读）**
- `references/responsive-recipe.md` — 三档响应式参数配方（实测数值 + 可直接复制的 style 字符串）
- `references/troubleshooting.md` — 踩坑手册（通道、代理、容器、断点、固定链接、兼容性）
- `references/target-site-config.md` — 目标站点配置与切换
- `templates/measure-iframe.html` — 三端同源 iframe 实测页
- `templates/batch-intake.md` — 批量任务清单模板
- `scripts/verify_front.py` — 前台图片本地化与可访问性核验
