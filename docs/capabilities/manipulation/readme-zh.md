# 操控

机器人操控臂的运动规划与遥操作。使用 Drake 进行物理仿真，Meshcat 进行 3D 可视化。

## 快速开始

### 键盘遥操作（单命令）

每个 Blueprint（蓝图）会启动完整的功能栈——键盘 UI、模拟控制器、IK（逆运动学）求解器和 Drake 可视化：

```bash
dimos run keyboard-teleop-piper   # Piper 6-DOF
dimos run keyboard-teleop-xarm6   # XArm6 6-DOF
dimos run keyboard-teleop-xarm7   # XArm7 7-DOF
```

打开终端中输出的 Meshcat URL（默认 `http://localhost:7000`）即可查看机器人。

键盘控制：

| 按键 | 动作 |
|------|------|
| W/S | +X/-X（前进/后退） |
| A/D | -Y/+Y（左移/右移） |
| Q/E | +Z/-Z（上升/下降） |
| R/F | +Roll/-Roll（横滚） |
| T/G | +Pitch/-Pitch（俯仰） |
| Y/H | +Yaw/-Yaw（偏航） |
| SPACE | 重置到初始姿态 |
| ESC | 退出 |

### 运动规划（两个终端）

```bash
# 终端 1：模拟协调器
dimos run coordinator-mock

# 终端 2：带 Drake 可视化的规划器
dimos run xarm7-planner-coordinator
```

然后使用 IPython 客户端：

```bash
python -m dimos.manipulation.planning.examples.manipulation_client
```

```python
joints()                # 获取当前关节角度
plan([0.1] * 7)         # 规划到目标位置
preview()               # 在 Meshcat 中预览
execute()               # 通过协调器执行
```

### 感知 + 智能体

```bash
# 终端 1：使用真实 xarm7 的协调器
dimos run coordinator-xarm7

# 终端 2：感知 + 操控 + LLM 智能体
dimos run xarm-perception-agent
```

## 架构

```
KeyboardTeleopModule ──→ ControlCoordinator ──→ ManipulationModule
  (pygame UI)              (100Hz tick loop)      (Drake + Meshcat)
       │                        │                       │
  PoseStamped            CartesianIK task         RRT planner
  commands               (Pinocchio IK)           JacobianIK
                              │                   DrakeWorld
                         JointState ────────────→ (visualization)
```

- **KeyboardTeleopModule** — Pygame UI，发布笛卡尔位姿指令
- **ControlCoordinator** — 100Hz 控制循环，支持模拟或真实硬件适配器
- **ManipulationModule** — Drake 物理引擎、Meshcat 可视化、RRT 运动规划、障碍物管理

## 蓝图

| 蓝图 | 描述 |
|------|------|
| `keyboard-teleop-piper` | Piper 6-DOF 键盘遥操作，带 Drake 可视化 |
| `keyboard-teleop-xarm6` | XArm6 6-DOF 键盘遥操作，带 Drake 可视化 |
| `keyboard-teleop-xarm7` | XArm7 7-DOF 键盘遥操作，带 Drake 可视化 |
| `xarm6-planner-only` | XArm6 独立规划器（无协调器） |
| `xarm7-planner-coordinator` | XArm7 规划器，集成协调器 |
| `dual-xarm6-planner` | 双 XArm6 规划 |
| `xarm-perception` | XArm7 + RealSense 摄像头感知 |
| `xarm-perception-agent` | XArm7 感知 + LLM 智能体 |

## 支持的机器人

| 机器人 | 自由度 | 遥操作 | 规划 | 感知 |
|--------|--------|--------|------|------|
| Piper | 6 | Y | Y | — |
| XArm6 | 6 | Y | Y | — |
| XArm7 | 7 | Y | Y | Y |

## 添加自定义机械臂

[指南在此](/docs/capabilities/manipulation/adding_a_custom_arm.md)

## 关键文件

| 文件 | 描述 |
|------|------|
| [`manipulation_module.py`](/dimos/manipulation/manipulation_module.py) | 主模块（RPC 接口、状态机） |
| [`manipulation/blueprints.py`](/dimos/manipulation/blueprints.py) | 规划器和感知蓝图 |
| [`robot/manipulators/piper/blueprints.py`](/dimos/robot/manipulators/piper/blueprints.py) | Piper 键盘遥操作蓝图 |
| [`robot/manipulators/xarm/blueprints.py`](/dimos/robot/manipulators/xarm/blueprints.py) | XArm 键盘遥操作蓝图 |
| [`teleop/keyboard/keyboard_teleop_module.py`](/dimos/teleop/keyboard/keyboard_teleop_module.py) | 键盘遥操作模块 |
| [`planning/world/drake_world.py`](/dimos/manipulation/planning/world/drake_world.py) | Drake 物理后端 |
| [`planning/planners/rrt_planner.py`](/dimos/manipulation/planning/planners/rrt_planner.py) | RRT-Connect 运动规划器 |
