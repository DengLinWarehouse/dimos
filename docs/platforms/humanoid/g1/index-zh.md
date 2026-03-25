# Unitree G1 — 快速入门

Unitree G1 是一款人形机器人平台，支持全身运动控制、手臂姿态控制和智能体能力——基本操作无需 ROS。

## 环境要求

- Unitree G1（原厂固件）
- Ubuntu 22.04/24.04 配备 CUDA GPU（推荐），或 macOS（实验性支持）
- Python 3.12
- ZED 相机（安装于胸部高度），用于感知蓝图
- ROS 2 用于导航（G1 导航栈使用 ROS nav）

## 安装

首先，为您的平台安装系统依赖：
- [Ubuntu](/docs/installation/ubuntu.md)
- [macOS](/docs/installation/osx.md)
- [Nix](/docs/installation/nix.md)

然后安装 DimOS：

```bash
uv venv --python "3.12"
source .venv/bin/activate
uv pip install 'dimos[base,unitree]'
```

## MuJoCo 仿真

没有硬件？从仿真开始：

```bash
uv pip install 'dimos[base,unitree,sim]'
dimos --simulation run unitree-g1-basic-sim
```

这将在 MuJoCo（物理仿真引擎）中运行 G1，使用原生 A*（A星）导航栈——相同的蓝图结构，模拟的机器人。在 [localhost:7779](http://localhost:7779) 打开指挥中心，并提供 Rerun 3D 可视化。

## 在您的 G1 上运行

```bash
export ROBOT_IP=<YOUR_G1_IP>
dimos run unitree-g1-basic
```

DimOS 通过 WebRTC 连接，启动 ROS 导航栈，并打开指挥中心。

### 运行模块

| 模块 | 功能说明 |
|--------|-------------|
| **G1Connection** | 与机器人的 WebRTC 连接——传输视频、里程计数据 |
| **Webcam** | ZED 相机采集（左目立体视觉，15 fps） |
| **VoxelGridMapper** | 使用列雕刻法构建 3D 体素地图（CUDA 加速） |
| **CostMapper** | 通过地形坡度分析将 3D 地图转换为 2D 代价地图 |
| **WavefrontFrontierExplorer** | 自主探索未建图区域 |
| **ROSNav** | ROS 2 导航集成，用于路径规划 |
| **RerunBridge** | 浏览器中的 3D 可视化 |
| **WebsocketVis** | 位于 localhost:7779 的指挥中心 |

### 发送目标

从指挥中心（[localhost:7779](http://localhost:7779)）：
- 在地图上点击设置导航目标
- 切换自主探索模式
- 监控机器人位姿、代价地图和规划路径

## 智能体控制

通过 LLM（大语言模型）智能体实现自然语言控制，智能体能理解物理空间并可指挥手臂姿态动作：

```bash
export OPENAI_API_KEY=<YOUR_KEY>
export ROBOT_IP=<YOUR_G1_IP>
dimos run unitree-g1-agentic
```

然后使用人类 CLI（命令行界面）：

```bash
humancli
> wave hello
> explore the room
> give me a high five
```

智能体订阅摄像头和空间记忆数据流，并可访问 G1 专属技能，包括手臂姿态动作和运动模式。

### 手臂姿态动作

G1 智能体可执行丰富的手臂姿态动作：

| 姿态动作 | 说明 |
|---------|-------------|
| Handshake | 用右手执行握手动作 |
| HighFive | 用右手击掌 |
| Hug | 双臂执行拥抱动作 |
| HighWave | 高举手臂挥手 |
| Clap | 双手鼓掌 |
| FaceWave | 在面部高度挥手 |
| LeftKiss | 用左手做飞吻动作 |
| ArmHeart | 双臂在头顶做心形动作 |
| RightHeart | 用右手做心形手势 |
| HandsUp | 双手举高 |
| RightHandUp | 仅举起右手 |
| Reject | 做拒绝或"不行"的手势 |
| CancelAction | 取消当前手臂动作并恢复中立姿态 |

### 运动模式

| 模式 | 说明 |
|------|-------------|
| WalkMode | 正常行走 |
| WalkControlWaist | 带腰部控制的行走 |
| RunMode | 跑步 |

## 键盘遥操作

通过基于 pygame 的摇杆进行直接键盘控制：

```bash
export ROBOT_IP=<YOUR_G1_IP>
dimos run unitree-g1-joystick
```

## 可用蓝图

| 蓝图 | 说明 |
|-----------|-------------|
| `unitree-g1-basic` | 连接 + ROS 导航 + 可视化 |
| `unitree-g1-basic-sim` | 仿真环境下的 A* 导航 |
| `unitree-g1` | 导航 + 感知 + 空间记忆 |
| `unitree-g1-sim` | 仿真环境下的感知 + 空间记忆 |
| `unitree-g1-agentic` | 完整栈：LLM 智能体 + G1 技能 |
| `unitree-g1-agentic-sim` | 仿真环境下的智能体栈 |
| `unitree-g1-full` | 智能体 + SHM 图像传输 + 键盘遥操作 |
| `unitree-g1-joystick` | 导航 + 键盘遥操作 |
| `unitree-g1-detection` | 导航 + YOLO 人体检测与追踪 |
| `unitree-g1-shm` | 导航 + 共享内存图像传输的感知 |
| `uintree-g1-primitive-no-nav` | 仅传感器 + 可视化（无导航，可作为自定义蓝图的基础） |

### 蓝图层级

蓝图采用增量组合方式：

```
primitive（传感器 + 可视化）
├── basic（+ 连接 + 导航）
│   ├── basic-sim（仿真连接 + A* 导航）
│   ├── joystick（+ 键盘遥操作）
│   └── detection（+ YOLO 人体追踪）
├── perceptive（+ 空间记忆 + 物体追踪）
│   ├── sim（仿真版本）
│   └── shm（+ 共享内存传输）
└── agentic（+ LLM 智能体 + G1 技能）
    ├── agentic-sim（仿真版本）
    └── full（+ SHM + 键盘遥操作）
```

## 深入了解

- [导航栈](/docs/capabilities/navigation/readme.md) — 路径规划和自主探索
- [可视化](/docs/usage/visualization.md) — Rerun、Foxglove、性能调优
- [数据流](/docs/usage/data_streams) — RxPY 流、背压机制、质量过滤
- [传输层](/docs/usage/transports/index.md) — LCM、SHM、DDS
- [蓝图](/docs/usage/blueprints.md) — 模块组合
- [智能体](/docs/capabilities/agents/readme.md) — LLM 智能体框架
