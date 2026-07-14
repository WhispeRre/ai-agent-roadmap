<div align="center">

<img src="readme_img.png" width="900" alt="AI Agent Roadmap">

# AI Agent Roadmap

**一份面向 AI Agent 学习、实践与长期复盘的个人学习手册。**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Built with mdBook](https://img.shields.io/badge/built%20with-mdBook-2f6f73)](https://rust-lang.github.io/mdBook/)
[![GitHub Pages](https://img.shields.io/badge/deploy-GitHub%20Pages-4460aa)](https://liuzeyu03.github.io/ai-agent-roadmap/)

[在线阅读（中文）](https://liuzeyu03.github.io/ai-agent-roadmap/zh/) ·
[Read English](https://liuzeyu03.github.io/ai-agent-roadmap/en/) ·
[English README](README.md)

</div>

---

## 项目定位

AI Agent Roadmap 是我整理的一份 Agent 学习手册与实践路线图。它把 LLM 基础、工具调用、记忆系统、RAG、规划与推理、LangGraph、MCP、多 Agent 协作、评估、安全、部署、Agentic RL 等内容放进一个可持续维护的 mdBook 站点里。

我希望这个仓库解决的问题不是“资料越多越好”，而是“学习路径更清楚”：先理解概念，再做小项目，再理解生产级 Agent 系统如何评估、加固和上线。

## 推荐学习路径

| 目标 | 推荐路线 |
| ---- | -------- |
| 快速理解 Agent 是什么 | Agent 基础 -> LLM 基础 -> 工具调用 -> ReAct -> Hello Agent |
| 构建一个能用的 Agent 应用 | 工具层 -> 记忆系统 -> 规划推理 -> RAG -> 上下文工程 -> LangGraph |
| 面向生产环境 | Harness Engineering -> 评估与优化 -> 安全与可靠性 -> 部署与观测 |
| 跟进研究脉络 | ReAct -> Reflexion -> 记忆论文 -> RAG 论文 -> Agentic RL |
| 边做边学 | 环境搭建 -> 工具调用 -> RAG QA -> LangGraph 工作流 -> 编程/数据/多模态 Agent |

## 内容包括

- 中英文双语 mdBook 阅读结构。
- LLM、RAG、Memory、Tool Use、Planning、Frameworks、Protocols、Evaluation、Security、Deployment、Agentic RL 等主题章节。
- 用图示、动画、公式和代码解释复杂概念，方便反复查阅。
- GitHub Pages 自动部署工作流。
- 为长时间阅读重新整理过的页面样式与导航体验。

## 本地预览

安装 `mdbook` 和 `mdbook-katex` 后，可以构建中英文两个版本：

```bash
mdbook build
cp book.toml book.toml.zh_bak
cp book-en.toml book.toml
mdbook build
mv book.toml.zh_bak book.toml
cp root-index.html ./book/index.html
```

快速本地预览：

```bash
./serve.sh
```

## 部署

仓库里已经包含 GitHub Actions 工作流：`.github/workflows/deploy.yml`。推送到 `main` 分支后，在 GitHub Pages 设置里选择 GitHub Actions 作为发布来源，即可自动构建并发布站点。

预期访问地址：

```text
https://liuzeyu03.github.io/ai-agent-roadmap/
```

如果你的 GitHub 用户名不是 `liuzeyu03`，请同步修改 `README.md`、`README_ZH.md`、`book.toml`、`book-en.toml` 和 `root-index.html` 中的链接。

## 项目说明

这是一个个人学习与整理项目。我会用它持续梳理 Agent 领域的概念地图、工程实践、论文脉络和项目经验，让它成为一个可以长期回看的技术手册。

本仓库遵循 MIT License。项目中包含基于开源资料整理和二次维护的内容，必要的版权与许可声明保留在 `LICENSE` 文件中。

## 致谢

本项目基于 MIT License 发布的开源 AI Agent 学习资料继续整理与维护。原始版权声明已按许可要求保留在 `LICENSE` 中；本仓库侧重个人学习路线整理、阅读体验优化、部署配置和持续维护。
