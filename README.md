# Hunter Agent 文档中心

> OFW 信贷智能助手矩阵系统

---

## 🎯 项目概述

Hunter 是专为 OFW（海外菲律宾劳工）信贷业务设计的 AI Agent 矩阵系统，旨在通过智能化手段提升获客效率、风控能力和贷后管理。

### 核心价值

| 维度 | 传统模式 | Hunter Agent 模式 |
|------|---------|-------------------|
| **获客成本** | $4.5/人 | $1-2/人 |
| **审批效率** | 3-5 天 | 1 天内 |
| **风控准确率** | 人工判断 | AI 预审 + 人工复核 |
| **用户体验** | 被动等待 | 7×24 小时智能服务 |

---

## 📚 文档导航

### 战略规划

- **[Hunter 重新定义 v2.0](docs/strategy/hunter-redefined-v2.0.md)** - 产品定位与战略方向
- **[顶层设计](docs/strategy/top-level-design.md)** - 系统架构与业务闭环

### 架构设计

- **[产品架构 v1.2](docs/architecture/hunter-architecture-v1.2.md)** - Agent 矩阵架构
- **[数据系统 v1.0](docs/data-system/hunter-data-system-v1.0.md)** - 数据模型与指标体系

### 产品需求

- **[Agent 管理系统 v1.0](docs/prd/hunter-agent-management-v1.0.md)** - 核心功能需求
- **[Agent 开发计划](docs/prd/hunter-agent-dev-plan-v1.0.md)** - 开发路线图
- **[Agent 测试用例](docs/prd/hunter-agent-test-cases-v1.0.md)** - 测试规范

---

## 🚀 快速开始

### 原型演示

访问 [在线原型](https://leojunhan.github.io/ofw-docs/) 查看交互演示。

### 文档结构

```
hunter-docs/
├── index.html          # Docsify 入口
├── README.md           # 首页
├── _sidebar.md         # 侧边栏导航
└── docs/
    ├── strategy/       # 战略规划
    ├── architecture/   # 架构设计
    ├── prd/            # 产品需求
    ├── data-system/    # 数据系统
    └── review/         # 评审记录
```

---

## 📊 Agent 矩阵概览

| Agent | 环节 | 职责 | 优先级 |
|-------|------|------|--------|
| SEO 内容 Agent | 获客 | 内容引流 | Phase 1 |
| FB 社群 Agent | 获客 | 社群监听互动 | Phase 1 |
| 风控预审 Agent | 风控 | 反欺诈 + 评分 | Phase 1 |
| 贷后管理 Agent | 贷后 | 还款提醒 + 复借 | Phase 1 |
| 数据回流 Agent | 闭环 | 数据优化模型 | Phase 1 |

---

## 🔗 相关资源

- [原型演示](https://leojunhan.github.io/ofw-docs/)
- [GitHub 仓库](https://github.com/leojunhan/hunter-docs)
- [问题反馈](https://github.com/leojunhan/hunter-docs/issues)

---

*最后更新: 2024-04-17*