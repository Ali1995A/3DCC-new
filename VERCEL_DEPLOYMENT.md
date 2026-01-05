# Vercel 部署指南

## 项目概述
这是一个基于Three.js和MediaPipe的3D手势粒子交互系统，支持手势控制各种3D粒子形状。

## 已完成的准备工作

### ✅ Git 仓库设置
- ✅ 初始化了本地 Git 仓库
- ✅ 配置了远程仓库: https://github.com/Ali1995A/3DCC.git  
- ✅ 推送了所有代码到远程仓库

### ✅ Vercel 配置文件
- ✅ `package.json` - 项目配置文件
- ✅ `vercel.json` - Vercel 部署配置
- ✅ `.gitignore` - Git 忽略文件配置

### ✅ 项目结构优化
- ✅ `index.html` - 重命名并作为主页
- ✅ 完整的3D粒子系统代码
- ✅ MediaPipe 手势识别集成

## 🚀 Vercel 部署步骤

### 方法一：通过 Vercel Dashboard (推荐)

1. 访问 [Vercel Dashboard](https://vercel.com/dashboard)
2. 点击 "New Project"
3. 选择 "Import Git Repository"
4. 授权 GitHub 并选择 `Ali1995A/3DCC` 仓库
5. 配置项目设置:
   - **Project Name**: `3dcc-particle-system` (或自定义)
   - **Framework Preset**: Other
   - **Root Directory**: `/` (默认)
   - **Build Command**: 留空 (静态网站)
   - **Output Directory**: 留空
6. 点击 "Deploy"

### 方法二：通过 Vercel CLI

1. 安装 Vercel CLI:
   ```bash
   npm i -g vercel
   ```

2. 在项目根目录运行:
   ```bash
   vercel
   ```

3. 按提示操作:
   - 选择 "Set up and deploy"
   - 连接到 GitHub 账户
   - 选择 `Ali1995A/3DCC` 仓库
   - 按默认配置继续

## 📱 项目特性

### 3D 粒子系统
- 支持 6 种粒子形状: 爱心、花朵、土星、佛像、烟花、CC字母
- 10000 个粒子的高性能渲染
- 透视效果和雾效增强深度感

### 手势控制
- 基于 MediaPipe 的实时手势识别
- 手掌张合控制粒子缩放
- 支持 iOS Safari 的摄像头权限

### UI 控制台
- 形状切换按钮
- 粒子颜色选择器
- 手势状态监控
- 全屏模式支持

## 🌐 访问方式

部署成功后，您可以通过以下方式访问:
- 主域名: `https://your-project-name.vercel.app`
- GitHub Pages 备用: `https://ali1995a.github.io/3DCC/`

## 🔧 技术栈

- **前端**: 原生 HTML/CSS/JavaScript
- **3D 渲染**: Three.js r128
- **手势识别**: MediaPipe Hands
- **部署**: Vercel
- **版本控制**: Git + GitHub

## 📝 注意事项

1. **摄像头权限**: 需要用户授权才能使用手势控制
2. **HTTPS 要求**: 摄像头访问需要 HTTPS 环境
3. **移动端优化**: 已针对 iOS Safari 特别优化
4. **性能**: 建议在现代浏览器中运行以获得最佳体验

---

**准备就绪！** 您的3D手势粒子交互系统已经为Vercel部署做好了完整准备。