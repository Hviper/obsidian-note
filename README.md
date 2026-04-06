# Obsidian Note 📝

> 个人知识库与技术沉淀仓库，基于 Obsidian 构建

## 📋 项目简介

这是一个技术知识管理与学习笔记仓库，主要用于记录和整理技术学习过程中的重要内容、最佳实践和深度思考。仓库采用 Obsidian 双向链接笔记系统，支持知识图谱可视化。

## 🗂️ 仓库结构

```
obsidian-note/
├── AI-每日资讯/           # AI 领域每日资讯跟踪
├── company-work/          # 公司工作相关技术分析
│   ├── optimal-dataq-backend-架构分析.md
│   └── omm-setup-源码分析报告.md
├── python/                # Python 相关技术笔记
│   ├── Clash Cli探索_obsidian.md
│   └── 打包成pip.md
├── skills生命周期/        # Agent Skills 生命周期研究
│   ├── OpenSpec-Skills生命周期.md
│   ├── Superpowers生命周期.md
│   └── autoresearch-Skills生命周期.md
├── temporal工作流/        # Temporal 分布式工作流引擎
│   ├── Temporal工作流框架完整指南.md
│   └── Activity幂等性实践指南.md
├── 好用skills/            # Claude Code Skills 实践
│   ├── PUA Skill 完整图谱 - 11 Skills 底层逻辑与使用场景.md
│   └── crawl.md
└── 随性记/                # 随手记录与想法
    └── 2026-03-23.md
```

## 🎯 核心内容模块

### 1. Agent Skills 生命周期研究

深入研究 Claude Code Skills 的生命周期管理与最佳实践：

- **OpenSpec 生命周期**: 用户主导的"工具箱"模型，显式调用，手动流转
- **Superpowers 生命周期**: AI 托管的"流水线"模型，自动流转，REQUIRED SUB-SKILL
- **PUA Skills**: 提高Agent积极性的skill系统，通过大厂PUA rhetoric驱动AI穷尽一切方案解决问题

### 2. Temporal 分布式工作流引擎

Temporal 是一个开源的分布式工作流编排平台，用于构建可靠的长时运行应用程序。

**核心概念**：
- **Workflow**: 编排层，定义业务逻辑流程（项目经理）
- **Activity**: 执行层，执行具体业务操作（工人）
- **Signal/Query**: 动态控制和查询状态（遥控器/仪表盘）

**关键要点**：
- Activity 最重要的是**幂等性**（不是原子性）
- Workflow 必须保证**确定性执行**
- 支持 Heartbeat 机制处理长时运行任务

### 3. 公司项目架构分析

包含企业级项目的架构分析与源码解读：

- **optimal-dataq-backend**: DDD 严格分层架构，ArchUnit 强制执行架构规则
- **omm-setup**: 源码分析与设计模式研究

### 4. Claude Code Skills 实践

收集和整理 Claude Code 的高效使用 Skills：

- **PUA Skill 11 Skills 图谱**: P10/P9/P8/P7 角色层级、味道切换、方法论智能路由
- **crawl**: Web 爬虫相关技能

### 5. AI 每日资讯

持续跟踪 AI 领域的最新动态和技术趋势。

## 🔧 技术栈

- **笔记工具**: Obsidian (双向链接 + 知识图谱)
- **版本控制**: Git + GitHub
- **编程语言**: Python, Java, TypeScript
- **工作流引擎**: Temporal
- **架构模式**: DDD (领域驱动设计)
- **Agent Skills**: Claude Code + Custom Skills

## 🎨 Obsidian 特性应用

本仓库充分利用 Obsidian 的核心特性：

### 双向链接 (Wikilinks)

```markdown
[[Temporal工作流框架完整指南]] - 快速跳转到相关笔记
[[PUA Skill 完整图谱]] - 建立 Skills 之间的关联
```

### 标签系统

```markdown
#temporal #workflow #distributed-systems
#pua #agent-skill #methodology
#ddd #architecture
```

### 元数据 Frontmatter

```yaml
---
title: 文章标题
date: 2026-04-05
tags:
  - tag1
  - tag2
aliases:
  - 别名
---
```

### Callouts 突出重点

```markdown
> [!tip] 核心洞察
> 关键信息摘要

> [!warning] 注意事项
> 需要特别关注的要点

> [!danger] 重要警告
> 可能导致问题的关键点
```

## 📚 学习路径推荐

### 新手入门

1. 阅读 `Temporal工作流框架完整指南.md` 了解分布式工作流
2. 学习 `OpenSpec-Skills生命周期.md` 理解 Agent Skills 的基础概念
3. 查看 `PUA Skill 完整图谱.md` 掌握 Skills 的实战应用

### 进阶学习

1. 深入研究 `optimal-dataq-backend-架构分析.md` 学习 DDD 架构设计
2. 实践 Temporal Activity 幂等性设计模式
3. 探索 Claude Code Skills 的高级用法

### 专家之路

1. 对比研究 OpenSpec vs Superpowers 两种生命周期哲学
2. 设计和实现自定义 Skills
3. 输出最佳实践文档并贡献社区

## 🔗 相关资源

- [Obsidian 官网](https://obsidian.md/)
- [Temporal 官方文档](https://docs.temporal.io/)
- [Claude Code](https://claude.ai/code)
- [DDD 领域驱动设计](https://martinfowler.com/bliki/DomainDrivenDesign.html)

## 📝 使用建议

1. **克隆仓库**到本地 Obsidian 库目录
2. **使用 Obsidian 打开**仓库根目录
3. **开启核心插件**:
   - 双向链接
   - 知识图谱
   - 标签面板
   - 模板
4. **推荐社区插件**:
   - Dataview
   - Templater
   - Calendar
   - Graph Analysis

## 🤝 贡献指南

这是一个个人知识库，暂不接受外部贡献。但欢迎参考本仓库的内容组织方式来构建你自己的知识库。

## 📄 License

MIT License - 自由使用和分享

---

**最后更新**: 2026-04-06
**维护者**: dongchang.he
**仓库地址**: https://github.com/Hviper/obsidian-note

---

> [!quote] 知识管理的本质
> "知识管理的目标不是存储信息，而是建立连接。"
> — Obsidian 理念