# Vercel 部署指南

## 项目概述
这是一个基于 Three.js 和 MediaPipe Hands 的 3D 手势粒子交互系统，支持多种粒子造型（含彩虹）与颜色模式（含赤橙黄绿青蓝紫的彩虹色）。

## 仓库与部署
- GitHub 仓库: `https://github.com/Ali1995A/3DCC-new.git`
- 部署平台: Vercel（静态站点）

## Vercel Dashboard 部署（推荐）
1. 打开 `https://vercel.com/dashboard`
2. New Project → Import Git Repository
3. 选择仓库 `Ali1995A/3DCC-new`
4. Framework Preset 选 `Other`
5. 其余保持默认（静态站点，无需额外构建步骤）→ Deploy

## Vercel CLI 部署（可选）
1. 安装：`npm i -g vercel`
2. 在项目根目录运行：`vercel`
3. 按提示绑定仓库并部署

## 注意事项
- 摄像头访问需要 HTTPS（Vercel 默认提供）。
- 微信内置浏览器对摄像头/全屏能力可能有限制：项目已做降级与提示。
