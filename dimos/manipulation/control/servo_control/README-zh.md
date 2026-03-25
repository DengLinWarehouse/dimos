# Cartesian Motion Controller（笛卡尔运动控制器）

面向机械臂的、与硬件无关的笛卡尔空间运动控制器。

## 概览

`CartesianMotionController` 通过以下方式提供闭环笛卡尔位姿跟踪：
1. **订阅**目标位姿（`PoseStamped`）
2. **计算**笛卡尔误差（位置 + 姿态）
3. **使用 PID（比例-积分-微分）控制**生成速度命令
4. **通过 IK（逆运动学）**转换到关节空间
5. **向硬件驱动发布**关节命令

## 架构

```
┌─────────────────────────────────────────────────────────────┐
│  TargetSetter (Interactive CLI)                             │
│  - User inputs target positions                             │
│  - Preserves orientation when left blank                    │
└───────────────────────┬─────────────────────────────────────┘
                        │ PoseStamped (/target_pose)
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  CartesianMotionController                                  │
│  - Computes FK (current pose)                               │
│  - Computes Cartesian error                                 │
│  - PID control → Cartesian velocity                         │
│  - Integrates velocity → next desired pose                  │
│  - Computes IK → target joint angles                        │
│  - Publishes current pose for feedback                      │
└──────────┬────────────────────────────────┬─────────────────┘
           │ JointCommand                   │ PoseStamped
           │                                │ (current_pose)
           ▼                                ▼
┌─────────────────────────────────┐  (back to TargetSetter
│  Hardware Driver (xArm, etc.)   │   for orientation preservation)
│  - 100Hz control loop           │
│  - Sends commands to robot      │
│  - Publishes JointState         │
└─────────────────────────────────┘
           │ JointState
           │ (feedback)
           ▼
  (back to controller)
```

## 关键特性

### ✓ 与硬件无关
- 可与**任何**实现 `ArmDriverSpec` 协议的机械臂驱动配合使用
- 只要求提供 `get_inverse_kinematics()` 和 `get_forward_kinematics()` 两个 RPC 方法
- 支持 xArm、Piper、UR、Franka 或自定义机械臂

### ✓ 基于 PID 的控制
- 位置（X、Y、Z）和姿态（roll、pitch、yaw）分别使用独立 PID
- 增益和速度限制均可配置
- 运动平滑、稳定，并具备阻尼效果

### ✓ 安全特性
- 可配置的位置/姿态误差限制
- 误差过大时自动执行紧急停止
- 命令超时检测
- 收敛性监控

### ✓ 灵活输入
- RPC 方法：`set_target_pose(position, orientation, frame_id)`
- 话题订阅：`target_pose`（`PoseStamped` 消息）
- 同时支持欧拉角和四元数

## 用法

### 基本示例

```python
from dimos.hardware.manipulators.xarm import XArmDriver, XArmDriverConfig
from dimos.manipulation.control import CartesianMotionController, CartesianMotionControllerConfig

# 1. Create hardware driver
arm_driver = XArmDriver(config=XArmDriverConfig(ip_address="192.168.1.235"))

# 2. Create Cartesian controller (hardware-agnostic!)
controller = CartesianMotionController(
    arm_driver=arm_driver,
    config=CartesianMotionControllerConfig(
        control_frequency=20.0,
        position_kp=1.0,
        max_linear_velocity=0.15,  # m/s
    )
)

# 3. Set up topic connections (shared memory)
from dimos.core.transport import pSHMTransport

transport_joint_state = pSHMTransport("joint_state")
transport_joint_cmd = pSHMTransport("joint_cmd")

arm_driver.joint_state.connection = transport_joint_state
controller.joint_state.connection = transport_joint_state
controller.joint_position_command.connection = transport_joint_cmd
arm_driver.joint_position_command.connection = transport_joint_cmd

# 4. Start modules
arm_driver.start()
controller.start()

# 5. Send Cartesian goal (move 10cm in X)
controller.set_target_pose(
    position=[0.3, 0.0, 0.5],  # xyz in meters
    orientation=[0, 0, 0],     # roll, pitch, yaw in radians
    frame_id="world"
)

# 6. Wait for convergence
while not controller.is_converged():
    time.sleep(0.1)

print("Target reached!")
```

### 使用四元数

```python
from dimos.msgs.geometry_msgs import Quaternion

# Create quaternion (identity rotation)
quat = Quaternion(x=0, y=0, z=0, w=1)

controller.set_target_pose(
    position=[0.4, 0.1, 0.6],
    orientation=[quat.x, quat.y, quat.z, quat.w],  # 4-element list
)
```

### 使用 PoseStamped 消息

```python
from dimos.msgs.geometry_msgs import PoseStamped

# Create target pose
target = PoseStamped(
    frame_id="world",
    position=[0.3, 0.2, 0.5],
    orientation=[0, 0, 0, 1]  # quaternion
)

# Option 1: Via RPC
controller.set_target_pose(
    position=list(target.position),
    orientation=list(target.orientation)
)

# Option 2: Via topic (if connected)
controller.target_pose.publish(target)
```

### 使用 TargetSetter 工具

`TargetSetter` 是一个交互式 CLI 工具，可方便地手动向控制器发送目标位姿。它为测试和遥操作提供了用户友好的界面。

**关键特性：**
- **交互式终端界面**：提示输入 x、y、z 坐标
- **姿态保持**：姿态留空时会自动使用当前姿态
- **实时反馈**：订阅控制器当前位姿
- **工作流简单**：输入坐标后按 Enter 即可

**设置方式：**

```python
# Terminal 1: Start the controller (as shown in Basic Example above)
arm_driver = XArmDriver(config=XArmDriverConfig(ip_address="192.168.1.235"))
controller = CartesianMotionController(arm_driver=arm_driver)

# Set up LCM transports for target_pose and current_pose
from dimos.core.transport import LCMTransport
controller.target_pose.connection = LCMTransport("/target_pose", PoseStamped)
controller.current_pose.connection = LCMTransport("/xarm/current_pose", PoseStamped)

arm_driver.start()
controller.start()

# Terminal 2: Run the target setter
python -m dimos.manipulation.control.target_setter
```

**使用示例：**

```
================================================================================
Interactive Target Setter
================================================================================
Mode: WORLD FRAME (absolute coordinates)

Enter target coordinates (Ctrl+C to quit)
================================================================================

--------------------------------------------------------------------------------

Enter target position (in meters):
  x (m): 0.3
  y (m): 0.0
  z (m): 0.5

Enter orientation (in degrees, leave blank to preserve current orientation):
  roll (°):
  pitch (°):
  yaw (°):

✓ Published target (preserving current orientation):
  Position: x=0.3000m, y=0.0000m, z=0.5000m
  Orientation: roll=0.0°, pitch=0.0°, yaw=0.0°
```

**工作原理：**

1. **TargetSetter** 订阅控制器发布的 `/xarm/current_pose`
2. 用户以米为单位输入目标位置（x、y、z）
3. 用户可以选择性地以角度输入姿态（roll、pitch、yaw）
4. 如果姿态留空（0、0、0），TargetSetter 会使用控制器提供的当前姿态
5. TargetSetter 将目标位姿发布到 `/target_pose` 话题
6. **CartesianMotionController** 接收目标并对其进行跟踪

**优点：**

- **无需姿态数学计算**：无需关心四元数即可移动位置
- **测试安全**：可在发送前手动验证每次移动
- **迭代迅速**：交互式测试不同位置
- **教学友好**：可以实时观察控制器响应

## 配置

```python
@dataclass
class CartesianMotionControllerConfig:
    # Control loop
    control_frequency: float = 20.0  # Hz (recommend 10-50Hz)
    command_timeout: float = 1.0     # seconds

    # PID gains (position)
    position_kp: float = 1.0   # m/s per meter of error
    position_ki: float = 0.0   # Integral gain
    position_kd: float = 0.1   # Derivative gain (damping)

    # PID gains (orientation)
    orientation_kp: float = 2.0   # rad/s per radian of error
    orientation_ki: float = 0.0
    orientation_kd: float = 0.2

    # Safety limits
    max_linear_velocity: float = 0.2   # m/s
    max_angular_velocity: float = 1.0  # rad/s
    max_position_error: float = 0.5    # m (emergency stop threshold)
    max_orientation_error: float = 1.57  # rad (~90°)

    # Convergence
    position_tolerance: float = 0.001  # m (1mm)
    orientation_tolerance: float = 0.01  # rad (~0.57°)

    # Control mode
    velocity_control_mode: bool = True  # Use velocity-based control
```

## 硬件抽象

该控制器使用 **Protocol pattern（协议模式）** 实现硬件抽象：

```python
# spec.py
class ArmDriverSpec(Protocol):
    # Required RPC methods
    def get_inverse_kinematics(self, pose: list[float]) -> tuple[int, list[float] | None]: ...
    def get_forward_kinematics(self, angles: list[float]) -> tuple[int, list[float] | None]: ...

    # Required topics
    joint_state: Out[JointState]
    robot_state: Out[RobotState]
    joint_position_command: In[JointCommand]
```

**任何实现该协议的驱动都可以与控制器配合工作！**

### 添加新的机械臂

1. 实现 `ArmDriverSpec` 协议：
   ```python
   class MyArmDriver(Module):
       @rpc
       def get_inverse_kinematics(self, pose: list[float]) -> tuple[int, list[float] | None]:
           # Your IK implementation
           return (0, joint_angles)

       @rpc
       def get_forward_kinematics(self, angles: list[float]) -> tuple[int, list[float] | None]:
           # Your FK implementation
           return (0, tcp_pose)
   ```

2. 与控制器一起使用：
   ```python
   my_driver = MyArmDriver()
   controller = CartesianMotionController(arm_driver=my_driver)
   ```

**就这些！无需修改控制器本身。**

## RPC 方法

### 控制方法

```python
@rpc
def set_target_pose(
    position: list[float],           # [x, y, z] in meters
    orientation: list[float],         # [qx, qy, qz, qw] or [roll, pitch, yaw]
    frame_id: str = "world"
) -> None
```

```python
@rpc
def clear_target() -> None
```

### 查询方法

```python
@rpc
def get_current_pose() -> Optional[Pose]
```

```python
@rpc
def is_converged() -> bool
```

## 话题

### 输入（订阅）

| 话题 | 类型 | 说明 |
|-------|------|-------------|
| `joint_state` | `JointState` | 当前关节位置/速度（来自驱动） |
| `robot_state` | `RobotState` | 机器人状态（来自驱动） |
| `target_pose` | `PoseStamped` | 目标 TCP 位姿（来自规划器） |

### 输出（发布）

| 话题 | 类型 | 说明 |
|-------|------|-------------|
| `joint_position_command` | `JointCommand` | 目标关节角（发送到驱动） |
| `cartesian_velocity` | `Twist` | 调试用：笛卡尔速度命令 |
| `current_pose` | `PoseStamped` | 当前 TCP 位姿（供 TargetSetter 和其他工具使用） |

## 控制算法

```
1. Read current joint state from driver
2. Compute FK: joint angles → TCP pose
3. Compute error: e = target_pose - current_pose
4. PID control: velocity = PID(e, dt)
5. Integrate: next_pose = current_pose + velocity * dt
6. Compute IK: next_pose → target_joints
7. Publish target_joints to driver
```

### 为什么这样有效

- **外环（笛卡尔）**：以 10-50Hz 运行，负责计算 IK
- **内环（关节）**：驱动以 100Hz 运行，负责平滑执行
- **解耦**：将高层规划与底层控制分离

## 调参指南

### 保守（安全）
```python
config = CartesianMotionControllerConfig(
    control_frequency=10.0,
    position_kp=0.5,
    max_linear_velocity=0.1,  # Slow!
)
```

### 中等（推荐）
```python
config = CartesianMotionControllerConfig(
    control_frequency=20.0,
    position_kp=1.0,
    position_kd=0.1,
    max_linear_velocity=0.15,
)
```

### 激进（更快）
```python
config = CartesianMotionControllerConfig(
    control_frequency=50.0,
    position_kp=2.0,
    position_kd=0.2,
    max_linear_velocity=0.3,
)
```

### 提示

- **增大 Kp**：响应更快，但可能出现振荡
- **增大 Kd**：阻尼更强，运动更平滑
- **增大 Ki**：消除稳态误差（通常不需要）
- **降低频率**：CPU 负载更低，运动更平滑
- **提高频率**：响应更快，精度更高

## 扩展

### 下一步（Phase 2+）

1. **轨迹跟踪**：增加航点跟踪
   ```python
   controller.follow_trajectory(waypoints: list[Pose], duration: float)
   ```

2. **避障**：与规划系统集成
   ```python
   controller.set_collision_checker(checker: CollisionChecker)
   ```

3. **阻抗控制**：增加力/力矩反馈
   ```python
   controller.set_impedance(stiffness: float, damping: float)
   ```

4. **视觉伺服**：与感知模块集成
   ```python
   controller.track_object(object_id: int)
   ```

## 故障排查

### 控制器不运动
- 检查 `arm_driver` 是否已启动并正在发布 `joint_state`
- 确认话题连接是否已建立
- 检查机器人是否处于正确模式（xArm 需要 servo mode）

### 振荡 / 不稳定
- 降低 `position_kp` 或 `orientation_kp`
- 增大 `position_kd` 或 `orientation_kd`
- 降低 `control_frequency`

### IK 失败
- 目标位姿可能不可达
- 检查关节限制
- 确认目标位姿在工作空间内
- 检查是否避开奇异位形

### 无法收敛
- 增大 `position_tolerance` / `orientation_tolerance`
- 检查工作空间限制
- 增大 `max_linear_velocity`

## 文件

```
dimos/manipulation/control/
├── __init__.py                          # Module exports
├── cartesian_motion_controller.py       # Main controller
├── target_setter.py                     # Interactive target pose publisher
├── example_cartesian_control.py         # Usage example
└── README.md                            # This file
```

## 相关模块

- [xarm_driver.py](../../hardware/manipulators/xarm/xarm_driver.py) - xArm 的硬件驱动
- [spec.py](../../hardware/manipulators/xarm/spec.py) - 协议规范
- [simple_controller.py](../../utils/simple_controller.py) - PID 实现

## 许可证

Copyright 2025 Dimensional Inc. - Apache 2.0 License
