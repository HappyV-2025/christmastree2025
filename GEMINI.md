# GEMINI.md - Agent Instructional Context

> **注意**: 本文件是 AI 代理（Gemini）理解本项目架构、代码规范和修改逻辑的核心指南。在执行任何任务前，请优先参考此上下文。

## 1. 项目架构核心 (Core Architecture)

本项目是一个**单文件 (Monolithic)**、**程序化生成 (Procedural Generation)** 的 WebGL 应用。没有构建流程 (No Build Step)。

*   **核心文件**: `index.html` (包含所有 HTML, CSS, JS)。
*   **入口点**: `window.onload` -> `init()` -> `animate()`。
*   **渲染循环**: 基于 `AnimationSystem` 的组件化更新循环。

### 1.1 代码地图 (Code Map of `index.html`)
由于代码集中在一个文件中，理解区块分布至关重要：
1.  **HTML/CSS**: 顶部 (UI, HUD, Modal)。
2.  **Imports**: `Three.js` (CDN)。
3.  **Classes (逻辑类)**:
    *   `GiftParticleTrail` / `MagicTrail`: 粒子特效。
    *   `AudioEngine`: 音频合成 (BGM, SFX)。
    *   `CameraDirector`: 运镜逻辑。
    *   `IceTextManager`: 3D 文字生成。
    *   `MegaGiftManager`: 核心剧情物体。
    *   `DecorManager`: 实例化渲染资源管理。
    *   `HeroTreeResources`: 英雄树资源单例。
    *   `Train`: 火车物理与模型。
    *   `Fireworks` / `Snow`: 环境粒子。
    *   `SantaSleigh`: 圣诞老人 AI。
4.  **Global Functions**: `buildTrack` (轨道生成), `buildEnv` (环境生成), `createHeroTree` (英雄树)。
5.  **Systems**: `PhysicsSystem`, `LightingSystem`, `RenderingSystem`, `AnimationSystem`。
6.  **Init & Loop**: `init()`, `animate()`.

---

## 2. 状态管理与配置 (State & Config)

### 2.1 全局配置 (`CFG` / `CONFIG`)
*   **位置**: 代码中段，定义了 `const CFG = { ... }`。
*   **用途**: 控制物理（速度、重力）、视觉（Bloom强度、雾）、相机参数。
*   **修改规则**: 优先通过修改 `CFG` 对象的值来调整参数，而不是硬编码。

### 2.2 运行时状态 (`STATE`)
*   **位置**: `const STATE = { ... }`。
*   **关键属性**:
    *   `started`: (bool) 是否已点击开始。
    *   `viewMode`: (enum) `FOLLOW`, `FREE`, `FOCUS_GIFT`, `CELEBRATION`.
    *   `trainDir`: (1/-1) 火车行驶方向。
    *   `giftPhase`: (enum) 礼盒交互阶段。
    *   `christmasParty`: (bool) 是否处于圣诞狂欢模式。

---

## 3. 开发与修改规范 (Development Guidelines)

### 🔴 绝对禁止 (Strict Prohibitions)
1.  **禁止引入外部资源**: 不要使用 `.load('assets/...')`。所有纹理必须用 `<canvas>` 绘制，所有模型必须用 `THREE.Geometry` 组合，所有音频必须用 `AudioContext` 合成。
2.  **禁止破坏实例化**: 森林、枕木、雪人等大量重复物体 **必须** 使用 `DecorManager` 和 `InstancedMesh`。不要直接 `new THREE.Mesh` 添加到场景中（除非是主角物体）。
3.  **禁止硬编码颜色**: 尽量使用 `0xrrggbb` 十六进制格式，便于阅读。

### 🟢 推荐做法 (Best Practices)
1.  **添加新物体**:
    *   先在 `DecorManager` 或 `HeroTreeResources` 中定义共享的 Geometry 和 Material。
    *   在 `buildEnv` 中进行位置计算和放置。
2.  **修改视觉效果**:
    *   调整 Bloom: 修改 `index.html` 底部的 `bloomPass` 参数或 `CFG.bloomStr`。
    *   调整 ToneMapping: 修改 `renderer.toneMappingExposure`。
3.  **调试**:
    *   利用 `window.debugSetDate` 函数测试倒计时逻辑。
    *   检查控制台输出，`AudioEngine` 和 `AnimationSystem` 有详细的 log。

---

## 4. 关键算法速查 (Algorithm Cheatsheet)

*   **轨道高度对齐**: 使用 `window.getTerrainHeight(x, z)` 获取任意坐标的地面高度。**修改物体位置时务必调用此函数以防穿模。**
*   **相机避障**: `CameraDirector` 使用射线检测 (`checkDist`) 和 地形高度检测 (`getTerrainHeight + offset`) 来动态调整相机 Y 轴。
*   **雪花/粒子优化**: 粒子系统使用了 Vertex Shader 中的取模 (`mod`) 运算实现无限循环，**修改粒子运动轨迹需修改 Shader 字符串，而非 JS 代码**。
*   **防锯齿策略**: 项目使用 **MSAA (samples: 4)** 在 `WebGLRenderTarget` 中。不要轻易开启 `FXAA`，因为它曾导致高亮物体产生黑色块 Artifact。

---

## 5. 常见任务索引 (Common Tasks)

*   **任务**: 修改火车速度。
    *   **操作**: 修改 `CFG.maxSpeed`。
*   **任务**: 调整倒计时逻辑。
    *   **操作**: 修改 `initCelebrationCountdown` 函数。
*   **任务**: 修改 BGM 旋律。
    *   **操作**: 修改 `BGM_MELODY` 常量数组。
*   **任务**: 增加森林密度。
    *   **操作**: 修改 `buildEnv` 函数中的 `count` 变量。

---

## 6. 语言偏好
*   **中文 (Chinese)**: 与用户交流、Git Commit Message、注释请优先使用中文。
