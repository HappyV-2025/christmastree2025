以下是《Christmas Express: Definitive Edition》的**深度技术规格说明书**

### 1. 核心数学模型与物理引擎规格 (Mathematical & Physics Specifications)

#### 1.1 轨道生成算法 (Procedural Track Generation)
*   **基础曲线方程**: 采用极坐标系下的变体**三叶草曲线 (Trifolium)**。
    *   公式逻辑: $r = BaseR + (Amp \times RandomScale \times \sin(3\theta))$。
    *   **参数细节**:
        *   `BaseR` (基准半径): 120 单位。
        *   `Amp` (振幅): 50 单位。
        *   **非对称随机化**: 轨道被分为三个扇区 ($0\sim2, 2\sim4, >4$ 弧度)，每个扇区应用独立的随机缩放系数 ($0.8 \sim 1.2$)，确保轨道形状每次运行都不规则，非完美对称。
*   **铁轨几何构建**:
    *   **双轨生成**: 不使用简单的 `ExtrudeGeometry` (会导致翻转问题)，而是计算切线 ($T$) 和副法线 ($B = T \times Up$)。
    *   **平行线计算**: $Point_{Left/Right} = Point_{Center} \pm (B \times \text{轨距})$，轨距设为 2.2。
    *   **平滑处理**: 使用 `CatmullRomCurve3` 对计算出的离散点进行插值，细分度 `divisions = 400`。

#### 1.2 火车动力学 (Train Dynamics)
*   **运动模型**:
    *   位置计算: 基于曲线进度 `prog` ($0 \sim 1$) 的归一化参数。
    *   **速度积分**: `prog += speed * dt * 0.2 * direction`。
    *   **加减速曲线**:
        *   加速: 线性增长 `speed += 0.005`。
        *   减速: 摩擦力衰减模型 `speed *= 0.98` (指数衰减)。
*   **车厢拖曳物理**:
    *   **间距算法**: 每节车厢具有固定的相位偏移 `offset = (index + 1) * 0.009`。
    *   **双向兼容**: 偏移量计算考虑 `STATE.trainDir`，确保无论是顺时针还是逆时针，车厢都在车头后方。
*   **轮组运动学**:
    *   角速度 $\omega$ 与线速度 $v$ 挂钩: `rotation.x -= dist * 150 * 2.0`。

#### 1.3 抛射物弹道 (Projectile Ballistics)
*   **礼盒投掷**:
    *   **初始向量**: $V_0 = V_{side} + V_{up} + V_{forward}$。
    *   **向量合成**:
        *   $V_{side}$: 沿车厢局部 X 轴 (Right向量) * 随机力度 ($8 \sim 16$)。
        *   $V_{up}$: 沿车厢局部 Y 轴 * 随机力度 ($5 \sim 10$)。
        *   $V_{forward}$: 沿车厢局部 Z 轴 * 随机扰动 (模拟空气湍流)。
    *   **重力积分**: $V_y = V_y - 25 \times dt$。
    *   **碰撞检测**: 简单的地平面检测 ($y \le 1$)，落地后重置旋转速度。

---

### 2. 图形渲染管线与着色器规格 (Graphics & Shader Specifications)

#### 2.1 材质系统 (Material System)
*   **物理材质 (PBR)**: 全面使用 `MeshStandardMaterial`。
    *   **车漆**: `roughness: 0.4`, `metalness: 0.2` (模拟烤漆)。
    *   **金属部件**: `roughness: 0.3`, `metalness: 0.8` (模拟金色/黄铜)。
    *   **发光部件**: 利用 `emissive` 属性实现自发光（车窗、车灯、LED）。
*   **实例化渲染 (Instancing)**:
    *   **应用对象**: 森林树木 (Cone)、枕木 (Box)、雪人 (Sphere)、路灯杆、地面礼盒。
    *   **动态更新**: 烟雾粒子使用 `InstancedMesh` 的 `DynamicDrawUsage`，每帧更新矩阵。
    *   **性能指标**: 场景中 3000+ 物体仅占用 <20 个 Draw Call。

#### 2.2 自定义着色器 (Custom Shaders)
*   **魔法尾迹 (Magic Trail)**:
    *   **渲染模式**: `Points` + `ShaderMaterial`。
    *   **深度衰减**: 在 Vertex Shader 中计算 `gl_PointSize = size * (Scale / -mvPosition.z)`，实现透视近大远小。
    *   **混合模式**: `NormalBlending`，纹理 Alpha 通道阈值剔除 (`discard`)。
*   **GPU 雪花系统 (Snow System)**:
    *   **顶点动画**: 完全在 Vertex Shader 中计算位置。
    *   **循环逻辑**: `pos.y = mod(position.y - speed * uTime, height)`，实现无限下落。
    *   **风力扰动**: `pos.x += sin(uTime * freq + y) * amp`，模拟飘雪摆动。

#### 2.3 后处理特效 (Post-Processing)
*   **管线**: `EffectComposer` -> `RenderPass` -> `UnrealBloomPass`。
*   **Bloom 参数**:
    *   `strength`: 0.7 (中等强度泛光)。
    *   `radius`: 0.5 (光晕扩散范围)。
    *   `threshold`: 0.02 (低阈值，确保大部分高亮物体都能产生辉光)。
    *   **Tone Mapping**: `ACESFilmicToneMapping`，曝光度 `1.3`，提供电影级色调。

---

### 3. 程序化几何体生成 (Procedural Geometry Generation)

#### 3.1 车顶积雪算法 (Snow Blanket Algorithm)
*   **基础网格**: 高细分圆柱体 (`radialSegments: 64`, `heightSegments: 40`)。
*   **噪声重构**:
    *   遍历顶点，将规则圆柱坐标 $(r, \theta, y)$ 映射为不规则积雪。
    *   **边缘噪声**: 使用多层正弦波叠加 ($\sin(freq1) + \sin(freq2)$) 模拟积雪边缘的垂落感。
    *   **不对称性**: 左右两侧 (`phaseL`, `phaseR`) 使用不同的随机相位，避免镜像对称。
    *   **厚度衰减**: 顶部最厚，边缘按 $\cos$ 曲线衰减变薄。

#### 3.2 英雄树构建 (Hero Tree Architecture)
*   **层级结构**: 5 层圆锥体，半径/高度按比例递减。
*   **电线生成 (Wire Generation)**:
    *   **路径**: 沿树体表面的螺旋线，但在半径上增加随机扰动 ($r \times 1.05$) 模拟电线松弛下垂。
    *   **钻孔逻辑**: 每层螺旋结束后，路径点强制收缩至树干中心，再延伸至下一层，模拟电线缠绕逻辑。
*   **装饰分布**:
    *   基于层级半径的泊松盘采样变体，确保装饰物不重叠。
    *   物体类型概率: 40% 彩球, 30% 礼盒, 30% 拐杖糖。

---

### 4. 音频合成架构 (Audio Synthesis Architecture)

#### 4.1 核心引擎 (`AudioEngine`)
*   **无采样设计**: 0% 外部音频文件，100% 实时合成。
*   **混音链路**: `Source -> Gain(Layer) -> MasterGain -> Destination`。
*   **噪音发生器**: 预计算 2秒 的 `AudioBuffer`，混合白噪音和简单的低通滤波（模拟粉红噪音），作为蒸汽和爆炸的基础素材。

#### 4.2 具体音色合成表
| 音效 | 构成组件 | 调制方式 |
| :--- | :--- | :--- |
| **BGM (Jingle Bells)** | 4层振荡器 (Sine基频, Sine泛音, Triangle敲击, Detuned Sine氛围) | 快速 Attack (0.01s), 指数 Decay |
| **汽笛 (Whistle)** | 双 Sawtooth 振荡器 (349Hz F4 + 415Hz G#4) | 频率微升 (Attack), 低通滤波 (1000Hz) |
| **行驶声 (Chug)** | 噪音 Buffer | BiquadFilter 低通滤波频率随速度线性增加 |
| **铃铛 (Bell)** | 4个 Sine 振荡器 (非整数倍频: 1x, 1.5x, 2x, 2.5x) | 极短 Attack, 长 Decay, 模拟金属共振 |
| **烟花 (Firework)** | 噪音 (Texture) + Sine (Kick) | **StereoPanner** (声像定位), BiquadFilter (Q值控制闷响度) |

---

### 5. 智能行为与状态机 (AI & State Machines)

#### 5.1 圣诞老人 AI
*   **状态机**: `Idle` -> `Approach` (贝塞尔曲线入场) -> `Sync` (PID控制器模拟速度同步) -> `Drop` (触发投掷) -> `Depart` (加速离场)。
*   **礼物制导系统 (Guided Missile Logic)**:
    *   **锁定**: 随机选择一节车厢 (`targetCar`)。
    *   **轨迹**: 线性插值 (`lerp`) + 正弦波垂直偏移 (`sin(t * PI)`) 模拟抛物线。
    *   **吸附**: 飞行进度 $t=1$ 时，将礼物对象从 `Scene` 移除，添加到 `Car` 的子节点，并根据车顶曲率计算最终的 Position 和 Rotation (法线对齐)。

#### 5.2 无人机摄像机系统
*   **覆盖算法**:
    *   将轨道分为 6 个扇区，每个扇区对应一个最佳观测点。
    *   **迟滞比较**: 计算火车当前的归一化位置 (`prog`) 与无人机扇区的相对偏移。
*   **平滑切换**:
    *   检测到越界时，不立即切镜，而是启动 `switching` 状态。
    *   使用 `smoothStep(t)` 函数在 `CurrentPos` 和 `NextPos` 之间插值，时长 1.0秒。

---

### 6. 内存管理与性能优化 (Memory & Performance)

#### 6.1 享元模式 (Flyweight Pattern)
*   **几何体复用**: 全局单例 `DecorManager` 和 `HeroTreeResources`。所有相同的物体（如几百个雪人）共用同一个 `SphereGeometry` 和 `MeshStandardMaterial` 内存引用。
*   **工厂模式**: `GiftFactory` 创建礼盒时，复用材质和几何体，仅在需要独立颜色控制时才克隆材质 (`material.clone()`)。

#### 6.2 对象池 (Object Pooling)
*   **烟花粒子池**: 预创建 15 个爆炸发射器 (`Points` 对象)。
    *   **复用逻辑**: 爆炸结束不销毁对象，而是标记 `available = true` 并隐藏。下次爆炸直接从池中取出重置属性。
    *   **零 GC**: 运行过程中无几何体创建/销毁，完全避免垃圾回收造成的掉帧。

#### 6.3 预分配变量
*   在 `animate` 循环外预先定义 `Vector3`, `Quaternion`, `Color` 等临时变量（如 `_v2`, `_target`），避免在每一帧的渲染循环中 `new THREE.Vector3()`，显著降低内存抖动。

---

### 7. 数据结构与配置常量

*   **DRONE 配置**:
    ```javascript
    const DRONE = {
        count: 6,           // 摄像机机位数量
        coverage: 0.18,     // 单机位覆盖范围 (略大于 1/6 以提供重叠区)
        switchDur: 1.0,     // 切换过渡时间
        smoothSpeed: 0.08   // 跟随阻尼系数
    };
    ```
*   **物理配置 (`CFG`)**:
    *   `carriageGap`: 0.009 (车厢间距，归一化单位)。
    *   `wheelMult`: 2.0 (视觉轮转速度倍率，非物理)。

### 8. 总结
该代码不仅仅是一个简单的 3D 场景，它实际上包含了一个**完整的迷你游戏引擎**架构：
1.  **ECS (Entity Component System) 雏形**: 将渲染(`Mesh`)、物理(`update`)、音频(`AudioEngine`)解耦。
2.  **程序化内容生成 (PCG)**: 从纹理到模型到音频的全流程式生成。
3.  **高级渲染技术**: Bloom、Instancing、Custom Shaders。
