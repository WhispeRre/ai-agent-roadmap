# 从零开始学 Agent - 样式总览

本文档整理了当前 mdBook 项目中使用的所有样式文件，按加载顺序和层级分类。

---

## 📋 样式加载顺序

构建后的 HTML 页面按以下顺序加载 CSS：

| 序号 | 文件 | 路径 | 类型 | 说明 |
|------|------|------|------|------|
| 1 | `variables.css` | `book/css/` | mdBook 内置 | 主题色变量定义（Light / Coal / Navy / Ayu / Rust） |
| 2 | `general.css` | `book/css/` | mdBook 内置 | 基础排版、字体、内容区布局 |
| 3 | `chrome.css` | `book/css/` | mdBook 内置 | 侧边栏、菜单栏、搜索框、导航、主题切换弹窗 |
| 4 | `font-awesome.css` | `book/FontAwesome/css/` | mdBook 内置 | 图标字体（GitHub 图标等） |
| 5 | `fonts.css` | `book/fonts/` | mdBook 内置 | 字体定义（Open Sans + Source Code Pro） |
| 6 | `highlight.css` | `book/` | mdBook 内置 | 代码语法高亮（亮色主题） |
| 7 | `tomorrow-night.css` | `book/` | mdBook 内置 | 代码语法高亮（暗色主题） |
| 8 | `ayu-highlight.css` | `book/` | mdBook 内置 | 代码语法高亮（Ayu 主题） |
| 9 | `custom.css` | `theme/` | ✅ 自定义 | 覆盖 mdBook 默认样式的自定义文件 |
| 10 | `head.hbs` 内联样式 | `theme/` | ✅ 自定义 | 交互增强组件的内联 CSS + JS |
| 11 | KaTeX CSS | CDN | 外部 | 数学公式渲染（`katex@0.16.4`） |

> **优先级**：后加载的样式优先级更高，`custom.css` 可覆盖所有 mdBook 内置样式。

---

## 一、mdBook 内置样式

> ⚠️ 以下文件由 `mdbook build` 自动生成，**不应直接修改**。如需调整，应在 `theme/custom.css` 中覆盖。

### 1.1 `variables.css` — 主题色变量定义

**路径**：`book/css/variables.css` (332 行)

定义了全局 CSS 变量和 6 套主题配色：

```css
/* 全局变量 */
:root {
    --sidebar-target-width: 300px;
    --sidebar-width: min(var(--sidebar-target-width), 80vw);
    --sidebar-resize-indicator-width: 8px;
    --page-padding: 15px;
    --content-max-width: 750px;
    --menu-bar-height: 50px;
    --mono-font: "Source Code Pro", Consolas, "Ubuntu Mono", Menlo, "DejaVu Sans Mono", monospace;
    --code-font-size: 0.875em;
}
```

**主题配色一览**：

| 主题 | `--bg` | `--fg` | `--links` | `--sidebar-bg` | 色彩模式 |
|------|--------|--------|-----------|-----------------|----------|
| `.light` | `hsl(0, 0%, 100%)` (白) | `hsl(0, 0%, 0%)` (黑) | `#20609f` | `#fafafa` | light |
| `.coal` | `hsl(200, 7%, 8%)` (深灰) | `#98a3ad` | `#2b79a2` | `#292c2f` | dark |
| `.navy` | `hsl(226, 23%, 11%)` (深蓝) | `#bcbdd0` | `#2b79a2` | `#282d3f` | dark |
| `.ayu` | `hsl(210, 25%, 8%)` (深黑) | `#c5c5c5` | `#0096cf` | `#14191f` | dark |
| `.rust` | `hsl(60, 9%, 87%)` (暖灰) | `#262625` | `#2b79a2` | `#3b2e2a` | — |

每套主题定义了以下变量族：
- 背景/前景：`--bg`, `--fg`
- 侧边栏：`--sidebar-bg`, `--sidebar-fg`, `--sidebar-active`, `--sidebar-spacer`
- 链接：`--links`
- 引用块：`--quote-bg`, `--quote-border`
- 表格：`--table-border-color`, `--table-header-bg`, `--table-alternate-bg`
- 搜索栏：`--searchbar-bg`, `--searchbar-fg`, `--searchbar-border-color`
- 代码复制按钮：`--copy-button-filter`, `--copy-button-filter-hover`
- 其他：`--icons`, `--icons-hover`, `--theme-popup-bg`, `--overlay-bg`

---

### 1.2 `general.css` — 基础排版布局

**路径**：`book/css/general.css` (281 行)

负责基础文档结构和排版的样式：

| 功能模块 | 关键规则 | 说明 |
|----------|----------|------|
| 根元素 | `font-size: 62.5%` | 1rem = 10px 基准 |
| 字体 | `font-family: "Open Sans", sans-serif` | 正文字体 |
| 内容区 | `.content main { max-width: var(--content-max-width) }` | 内容最大宽度 |
| 段落 | `.content p { line-height: 1.45em }` | 行高 |
| 表格 | `table { border-collapse: collapse }` | 基础表格样式 + 斑马纹 |
| 引用块 | `blockquote { background-color: var(--quote-bg) }` | 引用块底色 |
| 警告框 | `.warning { border-inline-start: 2px solid var(--warning-border) }` | 带 ⓘ 图标 |
| 键盘标签 | `kbd { background-color: var(--table-border-color) }` | 键盘快捷键样式 |
| 脚注 | `.footnote-definition` | 脚注定义样式 + 高亮动画 |
| 目录分篇标题 | `.chapter li.part-title { font-weight: bold }` | 侧边栏分篇标题 |
| 工具提示 | `.tooltiptext` | 复制按钮提示文字 |

---

### 1.3 `chrome.css` — UI 组件

**路径**：`book/css/chrome.css` (726 行)

控制所有 UI 交互组件：

| 功能模块 | 说明 |
|----------|------|
| **菜单栏** (`#menu-bar`) | 顶部导航栏，支持 sticky 定位 |
| **侧边栏** (`.sidebar`) | 固定定位，支持拖拽调整宽度、滚动、折叠动画 |
| **导航按钮** (`.nav-chapters`) | 两侧翻页按钮 + 底部移动端导航 |
| **搜索** (`#searchbar`, `#searchresults`) | 搜索栏、搜索结果列表、加载 spinner |
| **主题选择器** (`.theme-popup`) | 主题切换弹窗 |
| **帮助弹窗** (`#mdbook-help-popup`) | 键盘快捷键帮助 |
| **代码块按钮** (`pre > .buttons`) | 复制按钮（含 SVG 图标）+ hover 可见 |
| **内联代码** (`:not(pre) > .hljs`) | 行内代码片段样式 |
| **搜索高亮** (`mark`) | 搜索关键词高亮 + 渐隐动画 |
| **RTL 支持** | 全套从右到左布局适配 |
| **响应式** | 620px/1080px/1380px 多断点适配 |

---

### 1.4 `fonts.css` — 字体定义

**路径**：`book/fonts/fonts.css` (101 行)

定义了两套字体，均使用 `woff2` 格式本地加载：

| 字体 | 字重 | 用途 |
|------|------|------|
| **Open Sans** | 300 (Light), 400 (Regular), 600 (SemiBold), 700 (Bold), 800 (ExtraBold) | 正文字体，含正体和斜体变体 |
| **Source Code Pro** | 500 (Medium) | 代码等宽字体 |

> 许可证：Open Sans 使用 Apache License 2.0，Source Code Pro 使用 OFL。

---

## 二、自定义样式

> ✅ 以下文件是项目自行定制的样式，修改样式时应优先编辑这些文件。

### 2.1 `custom.css` — 主样式覆盖

**路径**：`theme/custom.css` (556 行)
**配置**：通过 `book.toml` 的 `additional-css = ["theme/custom.css"]` 加载

#### 样式模块清单

| 模块 | 关键选择器 | 说明 |
|------|-----------|------|
| **字体与排版** | `.content { font-size: 1.6rem; line-height: 1.9 }` | 加大正文字号，行高 1.9 |
| **段落间距** | `.content p { margin-bottom: 1.2em }` | 中文段落间距优化 |
| **内容宽度** | `:root { --content-max-width: 900px }` | 覆盖默认 750px → 900px |
| **标题美化** | `h1` 蓝色下划线 + 左侧竖条装饰，`h2` 灰色下划线，`h3` 蓝色字 | 多级标题视觉层次 |
| **引用块** | `blockquote` 蓝色左边框 + 圆角 | 替换默认的上下边框样式 |
| **代码块** | `pre` 圆角 8px + 边框 + 阴影 | 更精致的代码块 |
| **内联代码** | `p > code` 蓝底 + 粉色字 + 边框 | 区分代码块和行内代码 |
| **表格** | `thead` 蓝底白字 + 圆角 + 阴影 + 斑马纹 + hover 效果 | 全面美化表格 |
| **侧边栏** | 字号加大，hover 蓝底，active 高亮，分篇标题分隔线 | 侧边栏增强 |
| **有序列表** | 三级编号：①蓝色圆形序号 → ②小写字母方形标签 → ③罗马数字 | 自定义计数器 |
| **图片** | 居中 + 圆角 + 阴影 | 图片美化 |
| **链接** | 去下划线 + 底部蓝色边框 + hover 变深 | 链接样式 |
| **分隔线** | 渐变透明线 | `hr` 美化 |
| **粗体** | `strong { color: var(--links) }` | 加粗字蓝色高亮 |
| **选中文本** | `::selection` 蓝色半透明背景 | 文本选中效果 |
| **滚动条** | 6px 窄滚动条 + 灰色圆角 | WebKit 滚动条美化 |
| **翻页按钮** | 隐藏两侧按钮 `.nav-wide-wrapper { display: none }` | 仅保留底部导航 |
| **底部导航** | 圆角 + 边框 + hover 效果 | 底部翻页导航增强 |
| **页面动画** | `fadeInUp 0.4s` | 页面加载渐入动画 |
| **暗色主题适配** | `.navy / .coal / .ayu` 前缀 | 表格/代码块/引用/图片/分隔线的暗色适配 |
| **Mermaid** | `.mermaid` 居中 + 背景 + 圆角 | 流程图容器美化 |
| **脚注** | `.footnote-definition` 虚线上边框 | 脚注样式 |
| **打印** | `@media print` 隐藏侧边栏/导航 | 打印优化 |
| **移动端** | `@media (max-width: 768px)` | 字号/表格/代码块/图片/列表响应式适配 |

#### 完整代码

```css
/* ============================================
   从零开始学 Agent - 自定义样式
   ============================================ */

/* ---------- 字体与排版 ---------- */
:root {
  --content-max-width: 900px;
  --sidebar-width: 300px;
}

.content {
  font-size: 1.6rem;
  line-height: 1.9;
  letter-spacing: 0.015em;
}

/* 中文优化：段落间距 */
.content p {
  margin-bottom: 1.2em;
}

/* 侧边栏部分标题（分篇链接）加粗加大 */
.sidebar .chapter > li > a {
  font-size: 1.1rem;
}

/* ---------- 标题美化 ---------- */
.content h1 {
  border-bottom: 3px solid var(--links);
  padding-bottom: 0.4em;
  margin-top: 1.5em;
}

.content h2 {
  border-bottom: 1px solid #e0e0e0;
  padding-bottom: 0.3em;
  margin-top: 2em;
  margin-bottom: 0.8em;
}

.content h3 {
  margin-top: 1.6em;
  margin-bottom: 0.5em;
  color: var(--links);
}

/* ---------- 引用块 / Quote 美化 ---------- */
.content blockquote {
  border-left: 4px solid var(--links);
  background: rgba(var(--links-rgb, 55, 131, 235), 0.06);
  padding: 0.8em 1.2em;
  margin: 1.2em 0;
  border-radius: 0 8px 8px 0;
  font-style: normal;
}

.content blockquote p {
  margin-bottom: 0.3em;
}

/* ---------- 代码块优化 ---------- */
.content pre {
  border-radius: 8px;
  padding: 1em 1.2em;
  border: 1px solid #e0e0e0;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.04);
}

.content code {
  font-size: 0.95em;
  padding: 0.15em 0.4em;
  border-radius: 4px;
}

/* ---------- 表格美化 ---------- */
.content table {
  border-collapse: collapse;
  width: 100%;
  margin: 1.2em 0;
  font-size: 1.15rem;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.06);
}

.content table thead {
  background: var(--links);
  color: #fff;
}

.content table th {
  padding: 0.7em 1em;
  font-weight: 600;
  text-align: left;
  border: none;
}

.content table td {
  padding: 0.6em 1em;
  border-bottom: 1px solid #eee;
}

.content table tbody tr:hover {
  background: rgba(var(--links-rgb, 55, 131, 235), 0.04);
}

.content table tbody tr:last-child td {
  border-bottom: none;
}

/* ---------- 侧边栏优化 ---------- */
.sidebar .sidebar-scrollbox {
  padding-top: 1em;
  font-size: 1.1rem;
}

.sidebar .chapter li a {
  padding: 0.35em 1em;
  border-radius: 4px;
  transition: background 0.15s ease;
}

.sidebar .chapter li a:hover {
  background: rgba(var(--links-rgb, 55, 131, 235), 0.08);
}

.sidebar .chapter li.active > a {
  font-weight: 600;
  color: var(--links);
  background: rgba(var(--links-rgb, 55, 131, 235), 0.1);
  border-radius: 4px;
}

/* 分篇标题样式（SUMMARY 中的 --- 分隔符） */
.sidebar .chapter li.part-title {
  font-size: 0.95em;
  font-weight: 700;
  letter-spacing: 0.05em;
  color: var(--links);
  padding: 1em 0 0.3em 0.8em;
  margin-top: 0.6em;
  border-top: 1px solid rgba(128, 128, 128, 0.15);
}

.sidebar .chapter li.part-title:first-child {
  border-top: none;
  margin-top: 0;
}

/* ---------- 复选框列表美化 ---------- */
.content li > input[type="checkbox"] {
  margin-right: 0.4em;
}

/* ---------- 学习目标列表 ---------- */
.content ul li {
  margin-bottom: 0.3em;
}

/* ---------- 分隔线 ---------- */
.content hr {
  border: none;
  height: 1px;
  background: linear-gradient(to right, transparent, #ccc, transparent);
  margin: 2.5em 0;
}

/* ---------- 图片居中 & 阴影 ---------- */
.content img {
  display: block;
  margin: 1.5em auto;
  max-width: 100%;
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

/* ---------- 链接样式 ---------- */
.content a {
  text-decoration: none;
  border-bottom: 1px solid rgba(var(--links-rgb, 55, 131, 235), 0.3);
  transition: border-color 0.2s ease;
}

.content a:hover {
  border-bottom-color: var(--links);
}

/* ---------- 底部导航栏优化 ---------- */
.nav-wrapper {
  margin-top: 3em;
}

.nav-chapters {
  padding: 0.8em 1.2em;
  border-radius: 8px;
  transition: background 0.15s ease, box-shadow 0.15s ease;
}

.nav-chapters:hover {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

/* ---------- 隐藏两侧翻页按钮（保留底部导航） ---------- */
.nav-wide-wrapper {
  display: none !important;
}

/* ---------- 暗色主题适配 ---------- */
.navy .content table thead,
.coal .content table thead,
.ayu .content table thead {
  background: var(--links);
}

.navy .content pre,
.coal .content pre,
.ayu .content pre {
  border-color: #444;
}

.navy .content table td,
.coal .content table td,
.ayu .content table td {
  border-bottom-color: #444;
}

.navy .content blockquote,
.coal .content blockquote,
.ayu .content blockquote {
  background: rgba(255, 255, 255, 0.04);
}

.navy .content h2,
.coal .content h2,
.ayu .content h2 {
  border-bottom-color: #555;
}

/* 暗色主题下 SVG/图片：保持浅色背景，添加圆角和柔和边框 */
.navy .content img,
.coal .content img,
.ayu .content img {
  background: #f8f9fa;
  padding: 8px;
  border-radius: 10px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.3);
}

/* 暗色主题下分隔线颜色适配 */
.navy .content hr,
.coal .content hr,
.ayu .content hr {
  background: linear-gradient(to right, transparent, #555, transparent);
}

/* ---------- Mermaid 图表美化 ---------- */
.content .mermaid {
  text-align: center;
  margin: 1.5em 0;
  padding: 1em;
  background: rgba(0, 0, 0, 0.02);
  border-radius: 8px;
  overflow-x: auto;
}

.navy .content .mermaid,
.coal .content .mermaid,
.ayu .content .mermaid {
  background: rgba(255, 255, 255, 0.04);
}

/* ---------- 内联代码优化（区别于代码块） ---------- */
.content p > code,
.content li > code,
.content td > code,
.content h2 > code,
.content h3 > code {
  background: rgba(var(--links-rgb, 55, 131, 235), 0.08);
  color: #d63384;
  font-weight: 500;
  border: 1px solid rgba(var(--links-rgb, 55, 131, 235), 0.12);
}

.navy .content p > code,
.navy .content li > code,
.navy .content td > code,
.coal .content p > code,
.coal .content li > code,
.coal .content td > code,
.ayu .content p > code,
.ayu .content li > code,
.ayu .content td > code {
  background: rgba(255, 255, 255, 0.08);
  color: #ff79c6;
  border-color: rgba(255, 255, 255, 0.1);
}

/* ---------- 有序列表美化 ---------- */
.content ol {
  counter-reset: custom-counter;
  list-style: none;
  padding-left: 0;
}

.content ol > li {
  counter-increment: custom-counter;
  position: relative;
  padding-left: 2.2em;
  margin-bottom: 0.6em;
}

.content ol > li::before {
  content: counter(custom-counter);
  position: absolute;
  left: 0;
  top: 0.1em;
  width: 1.6em;
  height: 1.6em;
  border-radius: 50%;
  background: var(--links, #3783eb);
  color: #fff;
  font-size: 0.8em;
  font-weight: 700;
  display: flex;
  align-items: center;
  justify-content: center;
  line-height: 1;
}

/* 嵌套有序列表（二级）*/
.content ol ol {
  counter-reset: nested-counter;
  margin-top: 0.4em;
  margin-bottom: 0.2em;
}

.content ol ol > li {
  counter-increment: nested-counter;
  padding-left: 2em;
  margin-bottom: 0.3em;
}

.content ol ol > li::before {
  content: counter(nested-counter, lower-alpha) ")";
  width: auto;
  height: auto;
  border-radius: 3px;
  background: rgba(var(--links-rgb, 55, 131, 235), 0.1);
  color: var(--links, #3783eb);
  font-size: 0.78em;
  font-weight: 600;
  padding: 0.1em 0.4em;
}

/* 嵌套有序列表（三级）*/
.content ol ol ol {
  counter-reset: deep-counter;
}

.content ol ol ol > li {
  counter-increment: deep-counter;
  padding-left: 1.8em;
}

.content ol ol ol > li::before {
  content: counter(deep-counter, lower-roman) ".";
  background: none;
  color: var(--fg);
  font-size: 0.8em;
  font-weight: 500;
  padding: 0;
  opacity: 0.7;
}

/* ---------- 关键词高亮 ---------- */
.content strong {
  color: var(--links, #3783eb);
  font-weight: 700;
}

.navy .content strong,
.coal .content strong,
.ayu .content strong {
  color: #5ba3f5;
}

/* ---------- 脚注样式 ---------- */
.content .footnote-definition {
  font-size: 0.9em;
  padding: 0.5em 0;
  border-top: 1px dashed #e0e0e0;
  margin-top: 0.5em;
}

/* ---------- 章节标题装饰 ---------- */
.content h1::before {
  content: '';
  display: inline-block;
  width: 5px;
  height: 1em;
  background: var(--links, #3783eb);
  margin-right: 0.5em;
  border-radius: 3px;
  vertical-align: middle;
}

/* ---------- 页面容器过渡动画 ---------- */
.content {
  animation: fadeInUp 0.4s ease-out;
}

@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(12px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* ---------- 底部导航栏增强 ---------- */
.nav-chapters {
  display: flex;
  align-items: center;
  gap: 0.5em;
  border: 1px solid rgba(var(--links-rgb, 55, 131, 235), 0.15);
  background: rgba(var(--links-rgb, 55, 131, 235), 0.03);
}

.nav-chapters:hover {
  background: rgba(var(--links-rgb, 55, 131, 235), 0.08);
  border-color: rgba(var(--links-rgb, 55, 131, 235), 0.3);
}

/* ---------- 选中文本高亮 ---------- */
::selection {
  background: rgba(var(--links-rgb, 55, 131, 235), 0.25);
  color: inherit;
}

/* ---------- 表格偶数行斑马纹 ---------- */
.content table tbody tr:nth-child(even) {
  background: rgba(0, 0, 0, 0.02);
}

.navy .content table tbody tr:nth-child(even),
.coal .content table tbody tr:nth-child(even),
.ayu .content table tbody tr:nth-child(even) {
  background: rgba(255, 255, 255, 0.02);
}

/* ---------- 滚动条美化 ---------- */
::-webkit-scrollbar {
  width: 6px;
  height: 6px;
}

::-webkit-scrollbar-thumb {
  background: rgba(128, 128, 128, 0.3);
  border-radius: 3px;
}

::-webkit-scrollbar-thumb:hover {
  background: rgba(128, 128, 128, 0.5);
}

/* ---------- 打印优化 ---------- */
@media print {
  .sidebar,
  .nav-wrapper,
  .menu-bar {
    display: none !important;
  }
  .content {
    max-width: 100% !important;
  }
}

/* ---------- 移动端响应式 ---------- */
@media (max-width: 768px) {
  .content { font-size: 1.4rem; line-height: 1.75; }
  .content h1 { font-size: 1.6rem; }
  .content h2 { font-size: 1.35rem; }
  .content h3 { font-size: 1.15rem; }
  .content table { display: block; overflow-x: auto; font-size: 0.95rem; }
  .content pre { font-size: 0.82em; padding: 0.8em; }
  .content code { font-size: 0.88em; }
  .content img { margin: 1em auto; border-radius: 4px; }
  .content blockquote { padding: 0.5em 0.8em; margin: 0.8em 0; }
  .content ol > li { padding-left: 2em; }
  .content ol > li::before { width: 1.4em; height: 1.4em; font-size: 0.75em; }
  #back-to-top { bottom: 1.2rem; right: 1.2rem; width: 36px; height: 36px; font-size: 1rem; }
}
```

---

### 2.2 `head.hbs` — 内联样式 + 交互脚本

**路径**：`theme/head.hbs` (272 行)

注入到 `<head>` 中，包含内联 `<style>` 和 `<script>` 两部分。

#### 内联样式部分

| 组件 | 选择器 | 说明 |
|------|--------|------|
| **阅读进度条** | `#reading-progress` | 固定顶部渐变蓝色进度条（3px 高） |
| **回到顶部按钮** | `#back-to-top` | 右下角圆形按钮，滚动超过 400px 渐入显示 |
| **代码语言标签** | `.code-lang-tag` | 代码块右上角显示语言名（半透明背景） |
| **Admonition 提示框** | `.admonition-tip/warning/danger/info` | 基于 blockquote + emoji 的四种提示框 |
| **标题锚点链接** | `.header-anchor` | hover 时显示锚点链接 |
| **阅读时间** | `.reading-time` | h1 标题下方的圆角标签 |
| **暗色适配** | `.navy / .coal / .ayu` 前缀 | 所有组件的暗色主题变体 |

#### 内联样式代码

```css
#reading-progress {
  position: fixed;
  top: 0;
  left: 0;
  width: 0%;
  height: 3px;
  background: linear-gradient(90deg, #3783eb, #00d4ff);
  z-index: 10000;
  transition: width 0.1s ease-out;
  box-shadow: 0 0 8px rgba(55, 131, 235, 0.4);
}

#back-to-top {
  position: fixed;
  bottom: 2rem;
  right: 2rem;
  width: 42px;
  height: 42px;
  border-radius: 50%;
  background: var(--links, #3783eb);
  color: #fff;
  border: none;
  cursor: pointer;
  font-size: 1.2rem;
  display: flex;
  align-items: center;
  justify-content: center;
  opacity: 0;
  transform: translateY(20px);
  transition: opacity 0.3s ease, transform 0.3s ease, background 0.2s ease;
  z-index: 999;
  box-shadow: 0 2px 12px rgba(0,0,0,0.15);
}

.code-lang-tag {
  position: absolute;
  top: 0;
  right: 0;
  padding: 2px 10px;
  font-size: 0.7rem;
  font-weight: 600;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  color: rgba(255,255,255,0.85);
  background: rgba(0,0,0,0.25);
  border-radius: 0 8px 0 6px;
  pointer-events: none;
}

/* Admonition 四种类型 */
.content blockquote.admonition-tip    { border-left-color: #10b981; background: rgba(16, 185, 129, 0.06); }
.content blockquote.admonition-warning { border-left-color: #f59e0b; background: rgba(245, 158, 11, 0.06); }
.content blockquote.admonition-danger  { border-left-color: #ef4444; background: rgba(239, 68, 68, 0.06); }
.content blockquote.admonition-info    { border-left-color: #3b82f6; background: rgba(59, 130, 246, 0.06); }

.reading-time {
  display: inline-block;
  margin-top: 0.5em;
  padding: 0.3em 0.8em;
  font-size: 0.85rem;
  color: #666;
  background: rgba(0,0,0,0.04);
  border-radius: 20px;
  font-weight: 500;
}
```

#### JavaScript 交互脚本

| 功能 | 触发方式 | 说明 |
|------|----------|------|
| **阅读进度条** | `scroll` 事件（requestAnimationFrame 节流） | 计算 scrollTop / (scrollHeight - innerHeight) |
| **回到顶部按钮** | `scroll` 事件 | 滚动 > 400px 显示，点击 smooth scroll |
| **代码语言标签** | `DOMContentLoaded` | 遍历 `pre code`，解析 `language-*` 类名，映射为显示名 |
| **Admonition 分类** | `DOMContentLoaded` | 遍历 `blockquote`，根据首段 emoji 自动分类 |
| **阅读时间估计** | `DOMContentLoaded` | 按中文 500 字/分钟计算，插入到 h1 后方 |

**Admonition Emoji 映射规则**：

| Emoji | 类型 | 颜色 |
|-------|------|------|
| 💡 🔑 ✨ | `admonition-tip` | 绿色 `#10b981` |
| ⚠️ 🔶 | `admonition-warning` | 橙色 `#f59e0b` |
| 🚨 ❌ ⛔ | `admonition-danger` | 红色 `#ef4444` |
| 📖 ℹ️ 📝 📌 🔗 | `admonition-info` | 蓝色 `#3b82f6` |

**代码语言映射表**：

| 语言标识 | 显示名 | 语言标识 | 显示名 |
|----------|--------|----------|--------|
| `python` / `py` | Python | `json` | JSON |
| `javascript` / `js` | JS | `yaml` / `yml` | YAML |
| `typescript` / `ts` | TS | `toml` | TOML |
| `bash` | Bash | `sql` | SQL |
| `sh` / `shell` / `zsh` | Shell | `html` | HTML |
| `rust` | Rust | `css` | CSS |
| `go` | Go | `dockerfile` / `docker` | Docker |
| `java` | Java | `markdown` / `md` | MD |
| `xml` | XML | `graphql` | GraphQL |
| `mermaid` | Mermaid | 其他 | 大写原名 |

---

## 三、外部样式

### 3.1 KaTeX — 数学公式渲染

- **来源**：`https://cdn.jsdelivr.net/npm/katex@0.16.4/dist/katex.min.css`
- **加载方式**：CDN `<link>` 标签
- **用途**：渲染 LaTeX 数学公式
- **配置**：通过 `book.toml` 中 `[preprocessor.katex]` 启用

### 3.2 Font Awesome — 图标字体

- **来源**：mdBook 自带，本地加载 `book/FontAwesome/css/font-awesome.css`
- **用途**：菜单栏图标（如 GitHub 图标 `fa-github`）

---

## 四、配置入口 `book.toml`

```toml
[output.html]
default-theme = "light"           # 默认亮色主题
preferred-dark-theme = "navy"     # 暗色偏好使用 Navy 主题
additional-css = ["theme/custom.css"]  # 加载自定义样式

[output.html.print]
enable = false                    # 禁用打印页
```

---

## 五、样式架构图

```
┌─────────────────────────────────────────────────────────┐
│                     浏览器渲染                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ① mdBook 内置基础层                                     │
│  ┌──────────────┬──────────────┬──────────────┐         │
│  │ variables.css │ general.css  │  chrome.css  │         │
│  │ (主题变量)     │ (基础排版)    │ (UI组件)     │         │
│  └──────────────┴──────────────┴──────────────┘         │
│                         ↓                                │
│  ② 字体层                                                │
│  ┌──────────────────┬──────────────────┐                │
│  │   fonts.css       │ font-awesome.css │                │
│  │ (Open Sans +      │ (图标字体)        │                │
│  │  Source Code Pro)  │                  │                │
│  └──────────────────┴──────────────────┘                │
│                         ↓                                │
│  ③ 代码高亮层                                             │
│  ┌──────────────┬─────────────────┬─────────────────┐   │
│  │ highlight.css │tomorrow-night.css│ayu-highlight.css│   │
│  │ (亮色高亮)     │ (暗色高亮)       │ (Ayu高亮)       │   │
│  └──────────────┴─────────────────┴─────────────────┘   │
│                         ↓                                │
│  ④ 自定义覆盖层 (最高优先级)                               │
│  ┌──────────────────────────────────────────┐           │
│  │  custom.css (排版/表格/列表/主题适配等)     │           │
│  │  head.hbs 内联 (进度条/回顶/语言标签等)     │           │
│  └──────────────────────────────────────────┘           │
│                         ↓                                │
│  ⑤ 外部 CDN                                              │
│  ┌──────────────────────────────────────────┐           │
│  │  KaTeX CSS (数学公式渲染)                   │           │
│  └──────────────────────────────────────────┘           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 六、修改指南

| 需求 | 修改文件 | 说明 |
|------|----------|------|
| 调整颜色/排版/布局 | `theme/custom.css` | 添加或覆盖 CSS 规则 |
| 添加交互组件 | `theme/head.hbs` | 添加内联 `<style>` 和 `<script>` |
| 修改主题色变量 | `theme/custom.css` 中覆盖 `:root` 变量 | 不要直接修改 `book/css/variables.css` |
| 添加新字体 | `theme/head.hbs` 中添加 `<link>` 或 `@font-face` | 或在 `custom.css` 中定义 |
| 修改 mdBook 默认行为 | `theme/custom.css` 中使用更高优先级选择器覆盖 | 不要修改 `book/css/` 下文件 |

> 💡 **核心原则**：`book/` 目录下的文件由 `mdbook build` 自动生成，任何手动修改都会被覆盖。所有自定义样式应放在 `theme/` 目录下。
