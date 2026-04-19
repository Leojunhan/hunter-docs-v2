# 原型演示系统

## 在线原型

访问 [原型演示站点](https://leojunhan.github.io/ofw-docs/) 查看交互式原型。

---

## Apple 风格原型页面

基于 Apple Human Interface Guidelines 设计的高保真原型：

| 页面 | 说明 | 链接 |
|------|------|------|
| **Dashboard** | 数据概览仪表板 | [查看](https://leojunhan.github.io/ofw-docs/prototypes/apple-style/dashboard.html) |
| **Agent 管理** | Agent 列表与详情 | [查看](https://leojunhan.github.io/ofw-docs/prototypes/apple-style/agents.html) |
| **Skill 管理** | Skill 列表与详情 | [查看](https://leojunhan.github.io/ofw-docs/prototypes/apple-style/skills.html) |
| **执行记录** | 执行日志列表 | [查看](https://leojunhan.github.io/ofw-docs/prototypes/apple-style/executions.html) |
| **系统设置** | 系统配置页面 | [查看](https://leojunhan.github.io/ofw-docs/prototypes/apple-style/settings.html) |

---

## 设计规范

原型遵循 **Apple Human Interface Guidelines**：

### 颜色系统
- `systemBlue` (#0071e3) - 主要操作
- `systemGreen` (#34c759) - 成功/在线
- `systemRed` (#ff3b30) - 错误/离线
- `systemOrange` (#ff9500) - 警告/处理中

### 字体层级
- 标题: SF Pro Display, 34px/28px/22px
- 正文: SF Pro Text, 17px/15px/13px

### 间距规范
- 标准 8px 边距
- 圆角 12px (卡片), 8px (按钮)

### 动画效果
- Spring 动画 (duration: 0.3s)
- Material 模糊效果 (backdrop-filter)

---

## 统一设计原则

### 卡片封面字段统一

| 卡片类型 | 封面字段（最多 4 个） |
|---------|-------------------|
| Agent 卡片 | 名称、状态、描述、最后运行时间 |
| Skill 卡片 | 名称、状态、描述、使用次数 |

### 操作面板统一

所有操作统一放在详情抽屉底部：
- 编辑、删除、启动/停止、查看详情

---

## 相关文档

- [原型优化总结](OPTIMIZATION_SUMMARY.md)
- [GitHub Pages 部署指南](GITHUB_PAGES_DEPLOYMENT.md)
- [GitHub Pages 设置指南](GITHUB_PAGES_SETUP.md)

---

*最后更新: 2024-04-17*