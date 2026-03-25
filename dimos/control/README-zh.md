# Control Coordinator（控制协调器）

用于多臂机器人的集中式控制系统，支持逐关节仲裁。

## 架构

```
┌─────────────────────────────────────────────────────────────┐
│                   ControlCoordinator                        │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    TickLoop (100Hz)                  │   │
│  │                                                      │   │
│  │   READ ──► COMPUTE ──► ARBITRATE ──► ROUTE ──► WRITE │   │
│  └──────────────────────────────────────────────────────┘   │
│         │           │           │              │            │
│         ▼           ▼           ▼              ▼            │
│    ┌─────────┐  ┌───────┐  ┌─────────┐   ┌──────────┐       │
│    │Connected│  │ Tasks │  │Priority │   │ Adapters │       │
│    │Hardware │  │       │  │ Winners │   │          │       │
│    └─────────┘  └───────┘  └─────────┘   └──────────┘       │
└─────────────────────────────────────────────────────────────┘
```

## 快速开始

```bash
# 终端 1：运行协调器
dimos run coordinator-mock          # 单个 7 自由度模拟臂
dimos run coordinator-dual-mock     # 双臂（7+6 自由度）
dimos run coordinator-piper-xarm    # 真实硬件

# 终端 2：通过 CLI 控制
python -m dimos.manipulation.control.coordinator_client
```

## 核心概念

### Tick Loop（控制循环）
以 100Hz 运行的单一确定性循环：
1. **Read（读取）** - 从所有硬件获取关节位置
2. **Compute（计算）** - 每个任务计算期望输出
3. **Arbitrate（仲裁）** - 逐关节，最高优先级获胜
4. **Route（路由）** - 按硬件分组命令
5. **Write（写入）** - 向适配器发送命令

### Tasks（控制器）
Tasks 是由协调器调用的被动控制器：

```python
class MyController:
    def claim(self) -> ResourceClaim:
        return ResourceClaim(joints={"joint1", "joint2"}, priority=10)

    def compute(self, state: CoordinatorState) -> JointCommandOutput:
        # 在此编写你的控制律（PID、阻抗控制等）
        return JointCommandOutput(
            joint_names=["joint1", "joint2"],
            positions=[0.5, 0.3],
            mode=ControlMode.POSITION,
        )
```

### 优先级与仲裁
更高优先级始终获胜。仲裁在每个 tick 中进行：

```
traj_arm（优先级=10）希望 joint1 = 0.5
safety  （优先级=100）希望 joint1 = 0.0
                              ↓
                    safety 获胜，traj_arm 被抢占
```

### Preemption（抢占）
当一个任务的关节被更高优先级抢占时，它会收到通知：

```python
def on_preempted(self, by_task: str, joints: frozenset[str]) -> None:
    self._state = TrajectoryState.PREEMPTED
```

## 文件结构

```
dimos/control/
├── coordinator.py       # 模块 + RPC 接口
├── tick_loop.py         # 100Hz 控制循环
├── task.py              # ControlTask 协议 + 类型
├── hardware_interface.py # ConnectedHardware 包装器
├── components.py        # HardwareComponent 配置 + 类型别名
├── blueprints.py        # 预配置设置
└── tasks/
    └── trajectory_task.py  # 关节轨迹控制器
```

## 配置

```python
from dimos.control import control_coordinator, HardwareComponent, TaskConfig

my_robot = control_coordinator(
    tick_rate=100.0,
    hardware=[
        HardwareComponent(
            hardware_id="left_arm",
            hardware_type=HardwareType.MANIPULATOR,
            joints=make_joints("left_arm", 7),
            adapter_type="xarm",
            address="192.168.1.100",
        ),
        HardwareComponent(
            hardware_id="right_arm",
            hardware_type=HardwareType.MANIPULATOR,
            joints=make_joints("right_arm", 6),
            adapter_type="piper",
            address="can0",
        ),
    ],
    tasks=[
        TaskConfig(name="traj_left", type="trajectory", joint_names=[...], priority=10),
        TaskConfig(name="traj_right", type="trajectory", joint_names=[...], priority=10),
        TaskConfig(name="safety", type="trajectory", joint_names=[...], priority=100),
    ],
)
```

## RPC 方法

| 方法 | 描述 |
|--------|-------------|
| `list_hardware()` | 列出硬件 ID |
| `list_joints()` | 列出所有关节名称 |
| `list_tasks()` | 列出任务名称 |
| `get_joint_positions()` | 获取当前位置 |
| `execute_trajectory(task, traj)` | 执行轨迹 |
| `get_trajectory_status(task)` | 获取任务状态 |
| `cancel_trajectory(task)` | 取消活动轨迹 |

## 控制模式

Tasks 以以下三种模式之一输出命令：

| 模式 | 输出 | 使用场景 |
|------|--------|----------|
| POSITION | `q` | 轨迹跟踪 |
| VELOCITY | `q_dot` | 摇杆遥操作 |
| TORQUE | `tau` | 力控制、阻抗控制 |

## 编写自定义 Task

```python
from dimos.control.task import ControlTask, ResourceClaim, JointCommandOutput, ControlMode

class PIDController:
    def __init__(self, joints: list[str], priority: int = 10):
        self._name = "pid_controller"
        self._claim = ResourceClaim(joints=frozenset(joints), priority=priority)
        self._joints = joints
        self.Kp, self.Ki, self.Kd = 10.0, 0.1, 1.0
        self._integral = [0.0] * len(joints)
        self._last_error = [0.0] * len(joints)
        self.target = [0.0] * len(joints)

    @property
    def name(self) -> str:
        return self._name

    def claim(self) -> ResourceClaim:
        return self._claim

    def is_active(self) -> bool:
        return True

    def compute(self, state) -> JointCommandOutput:
        positions = [state.joints.joint_positions[j] for j in self._joints]
        error = [t - p for t, p in zip(self.target, positions)]

        # PID 控制
        self._integral = [i + e * state.dt for i, e in zip(self._integral, error)]
        derivative = [(e - le) / state.dt for e, le in zip(error, self._last_error)]
        output = [self.Kp*e + self.Ki*i + self.Kd*d
                  for e, i, d in zip(error, self._integral, derivative)]
        self._last_error = error

        return JointCommandOutput(
            joint_names=self._joints,
            positions=output,
            mode=ControlMode.POSITION,
        )

    def on_preempted(self, by_task: str, joints: frozenset[str]) -> None:
        pass  # 处理抢占
```

## 关节状态输出

协调器发布一条聚合的 `JointState` 消息，包含所有关节：

```python
JointState(
    name=["left_arm_joint1", ..., "right_arm_joint1", ...],  # 所有关节
    position=[...],
    velocity=[...],
    effort=[...],
)
```

通过以下主题订阅：`/coordinator/joint_state`
