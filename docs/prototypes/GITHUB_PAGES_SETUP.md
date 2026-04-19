# 🚀 GitHub Pages 部署指南

> Hunter Agent 管理模块原型已上传，需要手动启用 GitHub Pages

---

## ✅ 已完成

- [x] 原型文件已推送到 `main` 分支
- [x] 文件路径：`prototypes/*.html`
- [x] 提交记录：`feat: Hunter Agent 管理模块 - 原型设计与 PRD 文档（v1.0）`

---

## 🔧 需要手动操作（3 分钟）

### 步骤 1：登录 GitHub

访问：https://github.com/login

---

### 步骤 2：进入 Pages 设置

访问：https://github.com/Leojunhan/ofw-docs/settings/pages

---

### 步骤 3：配置 GitHub Pages

1. **Source**: 选择 `Deploy from a branch`

2. **Branch**: 选择 `main`

3. **Folder**: 选择 `/ (root)` 或输入 `prototypes/`
   
   **推荐方案 A**（简单）：
   - Branch: `main`
   - Folder: `/ (root)`
   - 访问地址：`https://leojunhan.github.io/ofw-docs/prototypes/dashboard.html`

   **推荐方案 B**（干净）：
   - Branch: `main`
   - Folder: `prototypes/`
   - 访问地址：`https://leojunhan.github.io/ofw-docs/`

4. 点击 **Save**

---

### 步骤 4：等待部署（1-2 分钟）

保存后，GitHub 会自动部署：

```
✅ Your site is live at https://leojunhan.github.io/ofw-docs/
```

---

## 🌐 访问地址

### 方案 A：根目录部署

```
https://leojunhan.github.io/ofw-docs/prototypes/dashboard.html
https://leojunhan.github.io/ofw-docs/prototypes/agents.html
https://leojunhan.github.io/ofw-docs/prototypes/workflow-editor.html
https://leojunhan.github.io/ofw-docs/prototypes/execution-detail.html
```

### 方案 B：prototypes 子目录部署

```
https://leojunhan.github.io/ofw-docs/dashboard.html
https://leojunhan.github.io/ofw-docs/agents.html
https://leojunhan.github.io/ofw-docs/workflow-editor.html
https://leojunhan.github.io/ofw-docs/execution-detail.html
```

---

## 📋 验证部署

### 方式 1：访问 Dashboard

```bash
# 在浏览器中打开
open https://leojunhan.github.io/ofw-docs/prototypes/dashboard.html
```

### 方式 2：检查部署状态

访问：https://github.com/Leojunhan/ofw-docs/deployments

查看最新的 GitHub Pages 部署状态。

---

## ⚠️ 常见问题

### 404 错误

**原因：** GitHub Pages 未启用或部署中

**解决：**
1. 确认已启用 GitHub Pages
2. 等待 1-2 分钟部署完成
3. 检查 Branch 和 Folder 配置是否正确

### 页面空白

**原因：** CDN 资源加载失败（国内网络问题）

**解决：**
1. Tailwind CSS、Font Awesome、Chart.js 使用 CDN
2. 可能需要科学上网
3. 或者下载 CDN 资源到本地

### 缓存问题

**解决：**
```bash
# 强制刷新浏览器缓存
Cmd + Shift + R (Mac)
Ctrl + Shift + R (Windows)
```

---

## 🔄 更新原型

```bash
# 修改原型文件后
cd /app/working/workspaces/RAKkDm

# 提交更改
git add prototypes/
git commit -m "feat: 更新 XXX 页面"
git push origin main

# GitHub Pages 会自动重新部署（1-2 分钟）
```

---

## 📊 部署架构图

```
本地开发 → Git 推送 → GitHub 仓库 → GitHub Pages → 全球 CDN
   ↓
prototypes/     ↓        main          自动部署      加速访问
*.html          ↓                    (1-2 分钟)
```

---

## 🎯 下一步

1. ✅ 启用 GitHub Pages（手动操作）
2. ✅ 验证部署成功
3. ✅ 分享评审链接给团队
4. ✅ 组织评审会议

---

## 📞 需要帮助？

如果遇到问题：
1. 检查 GitHub Pages 设置：https://github.com/Leojunhan/ofw-docs/settings/pages
2. 查看部署日志：https://github.com/Leojunhan/ofw-docs/deployments
3. 参考官方文档：https://docs.github.com/pages

---

**创建时间：** 2026-04-16  
**仓库：** https://github.com/Leojunhan/ofw-docs  
**分支：** `main`  
**状态：** ⏳ 等待启用 GitHub Pages
