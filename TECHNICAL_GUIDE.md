# 3DCC 手势粒子交互系统：技术原理与交互说明

本仓库是一个“单页静态站点”（`index.html` 即应用本体），用 **Three.js** 渲染 3D 粒子，用 **MediaPipe Hands** 做单手手势追踪，将手势映射为粒子“缩放 + 旋转”的实时控制，并提供 UI 控制台用于切换造型、配色与性能档位。

---

## 1. 技术栈与整体结构

- **渲染**：Three.js r128（CDN 引入），使用 `THREE.Points + BufferGeometry + PointsMaterial` 实现高粒子数渲染。
- **手势追踪**：MediaPipe Hands（CDN 引入），直接用 `<video>` 帧喂给 `hands.send({ image })`，避免 `camera_utils` 二次 `getUserMedia` 造成重复授权弹窗。
- **运行形态**：纯静态文件，可直接用 Vercel 静态部署；摄像头能力需要 **HTTPS**。

> 代码组织上所有逻辑都在 `index.html` 的 `<script>` 内，主要分为：环境/性能自适配 → 启动流程 → 摄像头初始化 → MediaPipe → Three.js 初始化 → UI 事件 → 动画循环 → 错误提示。

---

## 2. 渲染原理：Three.js 粒子系统

### 2.1 场景与渲染器配置

- `scene = new THREE.Scene()`，高/中档开启 `FogExp2`（雾效用于提升空间层次并隐藏远处粒子）。
- `camera = new THREE.PerspectiveCamera(60, aspect, 0.1, 1000)`，相机位置 `z=30, y=2`，形成更强的透视深度感。
- `renderer = new THREE.WebGLRenderer({ antialias, alpha, powerPreference: "high-performance" })`：
  - `powerPreference: "high-performance"` 倾向选择高性能 GPU 路径（浏览器仍可能忽略）。
  - `renderer.setPixelRatio(min(devicePixelRatio, quality.maxPixelRatio))` 对 DPR 做上限钳制，避免高分屏渲染开销暴涨。

### 2.2 数据结构：BufferGeometry 与两套位置缓冲

粒子系统由：

- `geometry = new THREE.BufferGeometry()`
- `position` attribute：`Float32Array(MAX_PARTICLES * 3)`
- 可选 `color` attribute：仅在“彩虹色模式”启用，用 `vertexColors` 进行逐点上色

并维护两套“目标位置”来源：

- `basePositions`：每个粒子的“造型基准点”（未缩放的形状坐标）。
- `positions`（`geometry.attributes.position.array`）：当前显示位置（会平滑逼近目标）。

动画中每个粒子的目标点按手势缩放得到：

```
target = base * handScaleFactor
position += (target - position) * TRANSITION_SPEED
```

这种“指数平滑”能在粒子数较多时保持连续的形变过渡，避免瞬时跳变造成眩晕或卡顿尖峰。

### 2.3 造型生成：算法采样而非模型加载

造型由 `getPointOnShape(type, idx, total)` 以随机采样的方式生成点云（例：心形公式、土星球体+环、彩虹拱形+云、两个字母 C 的 270° 弧等）。

为避免频繁重算造成卡顿，仓库使用了“惰性补齐”策略：

- `generatedCountForShape` 记录当前造型已生成的点数
- `ensureShapeGenerated(count)` 只对新增部分补算点位
- 切换质量档位导致粒子数变化时，按需补齐到新的 `activeParticleCount`

### 2.4 动画循环：渲染节流与粒子更新节流分离

主循环 `animate()` 使用 `requestAnimationFrame`，但会根据档位做两类节流：

- `quality.renderIntervalMs`：控制“渲染帧率”
- `quality.particleUpdateIntervalMs`：控制“粒子位置更新频率”

低端机/高负载场景下，降低更新频率通常比降低粒子数更容易保持流畅与稳定（CPU 端循环压力更可控）。

### 2.5 材质策略：AdditiveBlending + 可选雾/深度测试

`PointsMaterial` 关键配置：

- `blending: THREE.AdditiveBlending`：叠加发光效果
- `depthWrite: false`：避免写深度导致的排序/遮挡问题与额外开销
- `depthTest`、`fog` 随质量档位开关
- `sizeAttenuation: true`：点大小随深度衰减（近大远小）

彩虹色模式下：

- 设置 `material.vertexColors = true`
- 同时把 `material.color` 设为白色（因为 vertexColors 会与 material.color 相乘，避免颜色被“染暗/变少”）

---

## 3. 手势识别原理：MediaPipe Hands 到控制量映射

### 3.1 摄像头采集：iOS/微信兼容策略

初始化摄像头使用：

- `navigator.mediaDevices.getUserMedia({ video: { facingMode: 'user', width/height: ideal(...) }, audio: false })`
- `videoElement.srcObject = stream`
- `<video playsinline webkit-playsinline muted autoplay>`：保证移动端内联播放，避免强制全屏或播放失败

兼容性细节：

- 通过 `waitForVideoReady(video, timeout)` 监听 `loadedmetadata/loadeddata/canplay/playing` 多事件兜底，解决 iOS/微信环境下事件不触发导致的“永远等不到可用帧”的问题。
- 只要拿到 `stream` 就立刻隐藏“权限引导遮罩”，避免某些浏览器在授权完成后没有触发预期事件导致 UI 卡住。

### 3.2 MediaPipe 执行循环：主动喂帧 + 节流 + 防重入

MediaPipe Hands 配置（移动端优先）：

- `maxNumHands: 1`（单手）
- `modelComplexity: 0`（Lite 模型）
- `minDetectionConfidence / minTrackingConfidence: 0.5`

执行方式：

- 不使用 `camera_utils` 的 `Camera` 封装，避免它内部再次触发 `getUserMedia`（常见于重复权限弹窗/流冲突）。
- 使用单独的 `requestAnimationFrame(loop)`：
  - `quality.handFrameIntervalMs` 控制喂帧频率（高档 0 = 不限；中/低档为 66/100ms）
  - `handBusy` 标志位防止上一帧 `hands.send` 未完成又并发发送，避免队列堆积导致延迟上升

### 3.3 手势到控制量：缩放 + 旋转（含抗抖）

当检测到 `results.multiHandLandmarks[0]` 时：

1) **缩放（张合度）**：以“拇指尖(4) 到 食指尖(8)”距离（pinchDist）除以“手腕(0) 到 食指根部(5)”距离（palmSize）得到归一化的 `openRatio`，再映射到：

- `minScale = 0.3` ~ `maxScale = 2.5`

2) **旋转（手掌中心偏移）**：取 `[0,5,9,13,17]` 的平均作为掌心中心 `(cx, cy)`，映射到屏幕中心偏移 `dx, dy ∈ [-1,1]`：

- `targetHandRotY = dx * HAND_ROTATE_MAX_Y`（左右 → 绕 Y）
- `targetHandRotX = dy * HAND_ROTATE_MAX_X`（上下 → 绕 X）
- `HAND_DEADZONE` 对小幅抖动归零，降低手部微颤造成的画面抖动

3) **平滑**：在 `animate()` 中对 `targetHandScale/RotX/RotY` 做 lerp：

- 缩放使用 `HAND_SENSITIVITY`
- 旋转在“检测到手”与“未检测/未启用”时采用不同 lerp 系数，使回正更自然

当未检测到手时，系统会把目标缩放与旋转回到默认值（等价于进入“自动展示”状态）。

---

## 4. 交互模式：从“启动”到“沉浸体验”

### 4.1 启动页（Start Screen）

浏览器对摄像头与媒体播放通常要求**用户手势触发**，因此页面默认显示启动页：

- 点击“开启沉浸式体验”：
  1. 应用当前性能档位（`applyQuality`）
  2. 显示权限引导遮罩（提示点系统弹窗“允许”）
  3. 尝试初始化摄像头与 MediaPipe
  4. 初始化 Three.js 并进入 `animate()`

若摄像头不可用/被拒绝，则进入 **演示模式**：继续渲染粒子与自动旋转，但 `handTrackingEnabled=false`，不再依赖手势输入。

### 4.2 UI 控制台（右上角面板）

面板默认隐藏，交互逻辑：

- 点击画布（canvas）切换面板显示/隐藏
- 为避免移动端 `touchstart` 与 `click/pointer` 重复触发，代码做了：
  - `PointerEvent` 可用时仅监听 `pointerdown`
  - 否则退化为 `touchstart + click`
  - 且加 250ms 的节流避免连点误触
- 点击面板内部会 `stopPropagation()`，防止误触切换

面板内容：

- **造型选择**：切换 `currentShape` 并触发 `generateShape`
- **颜色模式**：单色 / 彩虹色（彩虹色启用 `vertexColors`）
- **颜色选择器**：仅在单色模式可用
- **性能模式**：auto/high/medium/low（影响粒子数、DPR、雾效、深度测试、渲染/手势/粒子更新节流、摄像头分辨率）
- **全屏**：调用 Fullscreen API；在微信内置浏览器中禁用并提示

---

## 5. 针对不同硬件与浏览器的控制细节

### 5.1 自动档位选择（Auto Quality）

`resolveInitialTier()` 的策略要点：

- URL 参数强制：`?quality=high|medium|low`
- 系统偏好：`prefers-reduced-motion: reduce` → `low`
- 微信内置浏览器：`low`（权限/性能/全屏限制更常见）
- iPad（含 iPadOS UA 伪装）：`low`
- iPhone 等 iOS：`medium`（平衡体验与热量/掉帧风险）
- 其他默认：`high`

### 5.2 性能档位影响面（Quality Presets）

每个档位同时调控“GPU/画质”和“CPU/算法吞吐”：

- `particleCount`：3800 / 6500 / 10000
- `maxPixelRatio`：1 / 1.5 / 2
- `antialias`：仅 high 开启
- `fog`：低档关闭
- `depthTest`：中低档常关闭（叠加粒子本身更适合关闭深度测试）
- `renderIntervalMs`：低档 33ms（≈30fps），中档 16ms（≈60fps 上限但会抑制波动），高档不限
- `handFrameIntervalMs`：低档 100ms，中档 66ms，高档不限
- `particleUpdateIntervalMs`：与渲染类似，低档降低粒子更新频率以减轻 CPU 循环
- `video`：320×240 / 480×360 / 640×480（越低越省电、省 CPU/GPU 带宽）

### 5.3 iOS / iPadOS 的关键点

- `playsinline` + `webkit-playsinline` 是 iOS 内联视频与 WebGL 同屏的关键。
- `waitForVideoReady` 多事件兜底，处理 iOS/微信里 `loadedmetadata` 不触发或触发顺序异常。
- DPR 钳制能显著降低 iPhone 高分屏在粒子大量叠加时的 fill-rate 压力。

### 5.4 微信内置浏览器的关键点

- 摄像头权限与全屏能力更容易被限制；代码会提示“建议右上角用 Safari 打开后重试”。
- 全屏按钮在微信环境下会被禁用（避免“看似可点但失败”的交互挫败）。

### 5.5 输入事件与滚动/回弹控制

为保证“沉浸式画布交互”：

- `touch-action: none`、`overscroll-behavior: none`：禁用默认手势滚动/回弹
- `user-select: none`：避免拖拽选择文本
- 对 `pointerdown/touchstart` 使用 `{ passive: true }`：减少浏览器对滚动阻塞的警告与额外成本

---

## 6. 常用调试与扩展点

### 6.1 URL 参数

- `?quality=high|medium|low`：强制性能档位（覆盖 Auto 判断）

### 6.2 常见要调的常量（都在 `index.html` 顶部脚本区）

- `MAX_PARTICLES` / `QUALITY_PRESETS.*.particleCount`：粒子数量与档位策略
- `TRANSITION_SPEED`：造型变形速度（越大越“快/硬”，越小越“慢/柔”）
- `HAND_SENSITIVITY`、`HAND_DEADZONE`、`HAND_ROTATE_MAX_X/Y`：手势手感（灵敏度/抗抖/旋转范围）

### 6.3 形状扩展

新增一个造型通常只需要：

1. 在 HTML 面板增加一个带 `data-shape="xxx"` 的按钮
2. 在 `getPointOnShape` 的 `switch` 内新增 `case 'xxx': ...`

建议保持“随机采样、单位成本小”的策略，否则在低端机上会明显影响首帧与切换响应。

---

## 7. 部署与安全相关（Vercel）

本仓库附带 `vercel.json`，主要用于：

- 强制一些通用安全响应头（`X-Frame-Options: DENY` 等）
- 对 `index.html` 禁止强缓存（便于快速发布更新）
- 提供 `/3dcc -> /index.html` 的重定向入口

---

## 8. 常见问题（FAQ）

- **摄像头不弹权限/一直黑屏**：确认使用 HTTPS；iOS/微信环境建议用系统浏览器打开；检查系统权限是否禁用浏览器摄像头。
- **点击全屏无效**：微信内置浏览器通常不支持/限制 Fullscreen API，属于预期降级。
- **帧率低/发热**：切到“低”或“中”；或用 `?quality=low` 强制低档并降低视频分辨率与喂帧频率。

