# Unitree Go2 — 快速入门

Unitree Go2 是 DimOS 的主要参考平台。支持完整的自主导航、建图和智能体控制——无需 ROS。

## 环境要求

- Unitree Go2 Pro 或 Air（原厂固件 1.1.7+，无需越狱）
- Ubuntu 22.04/24.04 配备 CUDA GPU（推荐），或 macOS（实验性支持）
- Python 3.12

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

## 试用——无需硬件

```bash
# 回放已录制的 Go2 导航会话
# 首次运行会从 LFS 下载约 2.4 GB 的激光雷达/视频数据
dimos --replay run unitree-go2
```

在 [localhost:7779](http://localhost:7779) 打开指挥中心，提供 Rerun 3D 可视化——实时观看 Go2 在办公室中建图和导航。

## 在您的 Go2 上运行

### 飞行前检查

1. 机器人可达且延迟低于 10ms，0% 丢包率
```bash
ping $ROBOT_IP
```

2. 内置避障功能已开启。（DimOS 负责路径规划，但机载避障功能在狭窄区域提供额外安全保障）

3. 如果视频与激光雷达/机器人位置不同步，请与 NTP 服务器同步时钟

```bash
sudo ntpdate pool.ntp.org
```
或
```bash
sudo sntp -sS pool.ntp.org
```

### 准备运行 DimOS

```bash
export ROBOT_IP=<YOUR_GO2_IP>
dimos run unitree-go2
```

就这么简单。DimOS 通过 WebRTC 连接（无需越狱），启动完整导航栈，并在浏览器中打开指挥中心。

### 运行模块

| 模块 | 功能说明 |
|--------|-------------|
| **GO2Connection** | 与机器人的 WebRTC 连接——传输激光雷达、视频、里程计数据 |
| **VoxelGridMapper** | 使用列雕刻法构建 3D 体素地图（CUDA 加速） |
| **CostMapper** | 通过地形坡度分析将 3D 地图转换为 2D 代价地图 |
| **ReplanningAStarPlanner** | 带动态重规划的持续 A* 路径规划 |
| **WavefrontFrontierExplorer** | 自主探索未建图区域 |
| **RerunBridge** | 浏览器中的 3D 可视化 |
| **WebsocketVis** | 位于 localhost:7779 的指挥中心 |

### 发送目标

从指挥中心（[localhost:7779](http://localhost:7779)）：
- 在地图上点击设置导航目标
- 切换自主探索模式
- 监控机器人位姿、代价地图和规划路径

## MuJoCo 仿真

```bash
uv pip install 'dimos[base,unitree,sim]'
dimos --simulation run unitree-go2
```

在 MuJoCo 中运行完整导航栈——相同的代码，模拟的机器人。

## 智能体控制

通过 LLM（大语言模型）智能体实现自然语言控制，智能体能理解物理空间：

```bash
export OPENAI_API_KEY=<YOUR_KEY>
export ROBOT_IP=<YOUR_GO2_IP>
dimos run unitree-go2-agentic
```

然后使用人类 CLI（命令行界面）与智能体对话：

```bash
humancli
> explore the space
```

智能体订阅摄像头、激光雷达和空间记忆数据流——它能看到机器人所看到的一切。

## 可用蓝图

| 蓝图 | 说明 |
|-----------|-------------|
| `unitree-go2-basic` | 连接 + 可视化（无导航） |
| `unitree-go2` | 完整导航栈 |
| `unitree-go2-agentic` | 导航 + LLM 智能体 |
| `unitree-go2-agentic-ollama` | 使用本地 Ollama 模型的智能体 |
| `unitree-go2-agentic-mcp` | 具有 MCP 工具访问能力的智能体 |
| `unitree-go2-spatial` | 导航 + 空间记忆 |
| `unitree-go2-detection` | 导航 + 物体检测 |
| `unitree-go2-ros` | ROS 2 桥接模式 |

## 深入了解

- [导航栈](/docs/capabilities/navigation/native/index.md) — 列雕刻体素建图、代价地图生成、A* 规划
- [可视化](/docs/usage/visualization.md) — Rerun、Foxglove、性能调优
- [数据流](/docs/usage/data_streams) — RxPY 流、背压机制、质量过滤
- [传输层](/docs/usage/transports/index.md) — LCM、SHM、DDS
- [蓝图](/docs/usage/blueprints.md) — 模块组合
