# Manipulation Planning Stack（操作规划栈）

用于机械臂的运动规划。采用后端无关设计，并提供 Drake 实现。

## 快速开始

```bash
# 1. Verify manipulation dependencies load correctly (standalone, no hardware):
dimos run xarm6-planner-only

# 2. Keyboard teleop with mock arm (single command):
dimos run keyboard-teleop-xarm7

# 3. Interactive RPC client (plan, preview, execute from Python):
dimos run xarm7-planner-coordinator                                    # terminal 1
python -i -m dimos.manipulation.planning.examples.manipulation_client  # terminal 2
```

在交互式客户端中：
```python
commands()              # List available commands
joints()                # Get current joint positions
plan([0.1] * 7)         # Plan to target
preview()               # Preview in Meshcat (url() for link)
execute()               # Execute via coordinator
```

## 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    ManipulationModule                       │
│         (RPC interface, state machine, multi-robot)         │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│              Backend-Agnostic Components                    │
│  ┌──────────────────┐  ┌─────────────────────────────┐     │
│  │ RRTConnectPlanner│  │ JacobianIK                  │     │
│  │ (rrt_planner.py) │  │ (iterative & differential) │     │
│  └──────────────────┘  └─────────────────────────────┘     │
│              Uses only WorldSpec interface                  │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                    WorldSpec Protocol                       │
│  Context management, collision checking, FK, Jacobian       │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│               Backend-Specific Implementations              │
│  ┌──────────────────┐  ┌─────────────────────────────┐     │
│  │ DrakeWorld       │  │ DrakeOptimizationIK         │     │
│  │ (physics/viz)    │  │ (nonlinear IK)              │     │
│  └──────────────────┘  └─────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

## 使用 ManipulationModule

```python
from pathlib import Path
from dimos.manipulation import ManipulationModule
from dimos.manipulation.planning.spec import RobotModelConfig

config = RobotModelConfig(
    name="xarm7",
    urdf_path=Path("/path/to/xarm7.urdf"),
    base_pose=PoseStamped(position=Vector3(), orientation=Quaternion()),
    joint_names=["joint1", "joint2", "joint3", "joint4", "joint5", "joint6", "joint7"],
    end_effector_link="link7",
    base_link="link_base",
    joint_name_mapping={"arm_joint1": "joint1", ...},  # coordinator <-> URDF
    coordinator_task_name="traj_arm",
)

module = ManipulationModule(
    robots=[config],
    planning_timeout=10.0,
    enable_viz=True,
    planner_name="rrt_connect",           # Only option
    kinematics_name="drake_optimization", # Or "jacobian"
)
module.start()
module.plan_to_joints([0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7])
module.execute()  # Sends to coordinator
```

## RobotModelConfig 字段

| 字段 | 说明 |
|-------|-------------|
| `name` | 机器人标识符 |
| `urdf_path` | URDF/XACRO 文件路径 |
| `base_pose` | 世界坐标系中机器人底座的 `PoseStamped` |
| `joint_names` | URDF 中的关节名称 |
| `end_effector_link` | 末端执行器链接名 |
| `base_link` | 底座链接名 |
| `max_velocity` | 最大关节速度（rad/s） |
| `max_acceleration` | 最大加速度（rad/s²） |
| `joint_name_mapping` | Coordinator → URDF 名称映射 |
| `coordinator_task_name` | 执行 RPC 使用的任务名 |
| `package_paths` | 网格资源使用的 ROS package 路径 |
| `xacro_args` | Xacro 参数（例如 `{"dof": "7"}`） |

## 组件

### 规划器（后端无关）

| 规划器 | 说明 |
|---------|-------------|
| `RRTConnectPlanner` | 双向 RRT-Connect（快速、可靠） |

### IK 求解器

| 求解器 | 类型 | 说明 |
|--------|------|-------------|
| `JacobianIK` | 后端无关 | 迭代式阻尼最小二乘 |
| `DrakeOptimizationIK` | Drake 专用 | 完整的非线性优化 |

### World 后端

| 后端 | 说明 |
|---------|-------------|
| `DrakeWorld` | 带 Meshcat 可视化的 Drake 物理世界 |

## Blueprints

| Blueprint | 说明 |
|-----------|-------------|
| `xarm6_planner_only` | XArm 6-DOF 独立运行（无 coordinator） |
| `xarm7-planner-coordinator` | 带 coordinator 的 XArm 7-DOF |
| `dual-xarm6-planner` | 双臂 XArm 6-DOF |

## 目录结构

```
planning/
├── spec.py                  # Protocols (WorldSpec, KinematicsSpec, PlannerSpec)
├── factory.py               # create_world, create_kinematics, create_planner
├── world/
│   └── drake_world.py       # DrakeWorld implementation
├── kinematics/
│   ├── jacobian_ik.py       # Backend-agnostic Jacobian IK
│   └── drake_optimization_ik.py  # Drake nonlinear IK
├── planners/
│   └── rrt_planner.py       # RRTConnectPlanner
├── monitor/                 # WorldMonitor (live state sync)
├── trajectory_generator/    # Time-parameterized trajectories
└── examples/
    └── manipulation_client.py    # Interactive RPC client (python -i)
```

## 障碍物类型

| 类型 | 尺寸 |
|------|------------|
| `BOX` | (width, height, depth) |
| `SPHERE` | (radius,) |
| `CYLINDER` | (radius, height) |
| `MESH` | mesh_path |

## 支持的机器人

| 机器人 | DOF |
|-------|-----|
| `piper` | 6 |
| `xarm6` | 6 |
| `xarm7` | 7 |

## 测试

```bash
# Unit tests (fast, no Drake)
pytest dimos/manipulation/test_manipulation_unit.py -v

# Integration tests (requires Drake)
pytest dimos/e2e_tests/test_manipulation_module.py -v
```
