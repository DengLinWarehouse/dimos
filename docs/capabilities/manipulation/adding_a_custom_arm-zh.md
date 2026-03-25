# 如何集成新的操控臂

本指南介绍如何将新的机器人臂集成到 DimOS 中，从编写硬件适配器到创建用于规划和控制的 Blueprint（蓝图）。

## 架构概览

DimOS 使用**基于 Protocol 的适配器模式**——不需要基类继承。你的适配器封装厂商 SDK 并暴露标准接口供系统其余部分使用：

```
┌──────────────────────────────────────────────────────────────┐
│              ManipulationModule（规划）                        │
│  - 使用 Drake 规划无碰撞轨迹                                  │
│  - 通过 RPC 将轨迹发送给协调器                                 │
└───────────────────────┬──────────────────────────────────────┘
                        │ RPC: 执行轨迹
┌───────────────────────▼──────────────────────────────────────┐
│              ControlCoordinator（100Hz 控制循环）              │
│  - 从所有适配器读取状态                                        │
│  - 运行任务（轨迹、伺服、速度）                                 │
│  - 按关节仲裁冲突（基于优先级）                                 │
│  - 将指令路由到正确的适配器                                     │
│  - 发布聚合的关节状态                                          │
└───────────────────────┬──────────────────────────────────────┘
                        │ 使用
┌───────────────────────▼──────────────────────────────────────┐
│              你的适配器（实现 Protocol）                        │
│  - 封装厂商 SDK（TCP/IP、CAN、串口等）                         │
│  - 在厂商单位和 SI 单位之间转换                                 │
│  - 处理连接生命周期                                            │
└──────────────────────────────────────────────────────────────┘
```

> 另请参阅：`dimos/hardware/manipulators/README.md` 了解快速参考。

## 前置条件

1. **厂商 SDK** — 你的机器人臂的 Python SDK（例如 `xarm-python-sdk`、`piper-sdk`）
2. **URDF/xacro** — 机器人描述文件（仅在需要运动规划时使用）
3. **连接信息** — IP 地址、CAN 端口、串口设备等

## 步骤 1：创建适配器

在 `dimos/hardware/manipulators/` 下为你的机械臂创建一个新目录：

```
dimos/hardware/manipulators/
├── spec.py              # ManipulatorAdapter Protocol（请勿修改）
├── registry.py          # 自动发现注册表（请勿修改）
├── mock/
├── xarm/
├── piper/
└── yourarm/             # ← 新目录
    ├── __init__.py
    └── adapter.py
```

### adapter.py — 完整骨架

以下是一个完整的带注解的适配器。通过封装你的厂商 SDK 调用来实现每个方法。所有跨越适配器边界的值**必须使用 SI 单位**。

| 物理量 | SI 单位 |
|--------|---------|
| 角度 | 弧度（radians） |
| 角速度 | rad/s |
| 力矩 | Nm |
| 位置 | 米（meters） |
| 力 | 牛顿（Newtons） |

```python
"""YourArm 适配器 - 实现 ManipulatorAdapter 协议。

SDK 单位：<在此描述你的 SDK 原生单位>
DimOS 单位：角度=弧度，距离=米，速度=rad/s
"""

from __future__ import annotations

import math
from typing import TYPE_CHECKING

# 导入你的厂商 SDK
from yourarm_sdk import YourArmSDK

if TYPE_CHECKING:
    from dimos.hardware.manipulators.registry import AdapterRegistry

from dimos.hardware.manipulators.spec import (
    ControlMode,
    JointLimits,
    ManipulatorInfo,
)

# 单位转换常量（如果你的 SDK 不使用 SI 单位）
MM_TO_M = 0.001
M_TO_MM = 1000.0


class YourArmAdapter:
    """YourArm 硬件适配器。

    通过鸭子类型实现 ManipulatorAdapter 协议。
    无需继承——只需匹配 spec.py 中的方法签名。
    """

    def __init__(self, address: str, dof: int = 6) -> None:
        """初始化适配器。

        Args:
            address: 连接地址（IP、CAN 端口、串口设备等）
            dof: 自由度。
        """
        if not address:
            raise ValueError("address is required for YourArmAdapter")
        self._address = address
        self._dof = dof
        self._sdk: YourArmSDK | None = None
        self._control_mode: ControlMode = ControlMode.POSITION

    # =========================================================================
    # 连接
    # =========================================================================

    def connect(self) -> bool:
        """连接到硬件。成功时返回 True。"""
        try:
            self._sdk = YourArmSDK(self._address)
            self._sdk.connect()
            # 验证连接是否成功
            if not self._sdk.is_alive():
                print(f"ERROR: Arm at {self._address} not reachable")
                return False
            return True
        except Exception as e:
            print(f"ERROR: Failed to connect to arm at {self._address}: {e}")
            return False

    def disconnect(self) -> None:
        """断开硬件连接。"""
        if self._sdk:
            self._sdk.disconnect()
            self._sdk = None

    def is_connected(self) -> bool:
        """检查是否已连接。"""
        return self._sdk is not None and self._sdk.is_alive()

    # =========================================================================
    # 信息
    # =========================================================================

    def get_info(self) -> ManipulatorInfo:
        """获取操控臂信息（厂商、型号、自由度）。"""
        return ManipulatorInfo(
            vendor="YourVendor",
            model="YourModel",
            dof=self._dof,
            firmware_version=None,  # 可选：如果可用，从 SDK 查询
            serial_number=None,     # 可选：如果可用，从 SDK 查询
        )

    def get_dof(self) -> int:
        """获取自由度。"""
        return self._dof

    def get_limits(self) -> JointLimits:
        """获取关节位置和速度限制（SI 单位）。

        可以硬编码已知限制，也可以从 SDK 查询。
        """
        return JointLimits(
            position_lower=[-math.pi] * self._dof,     # 弧度
            position_upper=[math.pi] * self._dof,       # 弧度
            velocity_max=[math.pi] * self._dof,          # rad/s
        )

    # =========================================================================
    # 控制模式
    # =========================================================================

    def set_control_mode(self, mode: ControlMode) -> bool:
        """设置控制模式。

        将 DimOS ControlMode 枚举值映射到你的 SDK 的模式代码。
        对于不支持的模式返回 False。
        """
        if not self._sdk:
            return False

        mode_map = {
            ControlMode.POSITION: 0,        # 你的 SDK 的位置模式代码
            ControlMode.SERVO_POSITION: 1,   # 高频伺服模式
            ControlMode.VELOCITY: 4,         # 速度模式
            # 添加其他支持的模式...
        }

        sdk_mode = mode_map.get(mode)
        if sdk_mode is None:
            return False  # 不支持的模式

        success = self._sdk.set_mode(sdk_mode)
        if success:
            self._control_mode = mode
        return success

    def get_control_mode(self) -> ControlMode:
        """获取当前控制模式。"""
        return self._control_mode

    # =========================================================================
    # 状态读取
    # =========================================================================

    def read_joint_positions(self) -> list[float]:
        """读取当前关节位置（弧度）。

        从 SDK 单位转换为弧度。
        """
        if not self._sdk:
            raise RuntimeError("Not connected")
        raw_positions = self._sdk.get_joint_positions()
        return [math.radians(p) for p in raw_positions[:self._dof]]

    def read_joint_velocities(self) -> list[float]:
        """读取当前关节速度（rad/s）。

        如果你的 SDK 不提供速度反馈，返回零值。
        协调器可以通过有限差分估算速度。
        """
        if not self._sdk:
            return [0.0] * self._dof
        # 如果 SDK 支持速度读取：
        # raw_velocities = self._sdk.get_joint_velocities()
        # return [math.radians(v) for v in raw_velocities[:self._dof]]
        return [0.0] * self._dof

    def read_joint_efforts(self) -> list[float]:
        """读取当前关节力矩（Nm）。

        如果你的 SDK 不提供力矩反馈，返回零值。
        """
        if not self._sdk:
            return [0.0] * self._dof
        # 如果 SDK 支持力矩读取：
        # return list(self._sdk.get_joint_torques()[:self._dof])
        return [0.0] * self._dof

    def read_state(self) -> dict[str, int]:
        """读取机器人状态（模式、状态码等）。"""
        if not self._sdk:
            return {"state": 0, "mode": 0}
        return {
            "state": self._sdk.get_state(),
            "mode": self._sdk.get_mode(),
        }

    def read_error(self) -> tuple[int, str]:
        """读取错误码和消息。(0, '') 表示无错误。"""
        if not self._sdk:
            return 0, ""
        code = self._sdk.get_error_code()
        if code == 0:
            return 0, ""
        return code, f"YourArm error {code}"

    # =========================================================================
    # 运动控制（关节空间）
    # =========================================================================

    def write_joint_positions(
        self,
        positions: list[float],
        velocity: float = 1.0,
    ) -> bool:
        """发送关节位置指令（弧度）。

        Args:
            positions: 目标位置（弧度）。
            velocity: 速度，为最大值的比例（0-1）。

        发送前需从弧度转换为 SDK 单位。
        """
        if not self._sdk:
            return False
        sdk_positions = [math.degrees(p) for p in positions]
        return self._sdk.set_joint_positions(sdk_positions)

    def write_joint_velocities(self, velocities: list[float]) -> bool:
        """发送关节速度指令（rad/s）。

        如果不支持速度控制，返回 False。
        """
        if not self._sdk:
            return False
        sdk_velocities = [math.degrees(v) for v in velocities]
        return self._sdk.set_joint_velocities(sdk_velocities)

    def write_stop(self) -> bool:
        """立即停止所有运动。"""
        if not self._sdk:
            return False
        return self._sdk.emergency_stop()

    # =========================================================================
    # 伺服控制
    # =========================================================================

    def write_enable(self, enable: bool) -> bool:
        """启用或禁用伺服电机。"""
        if not self._sdk:
            return False
        return self._sdk.enable_motors(enable)

    def read_enabled(self) -> bool:
        """检查伺服电机是否已启用。"""
        if not self._sdk:
            return False
        return self._sdk.motors_enabled()

    def write_clear_errors(self) -> bool:
        """清除错误状态。"""
        if not self._sdk:
            return False
        return self._sdk.clear_errors()

    # =========================================================================
    # 可选：笛卡尔控制
    # 如果你的机械臂不支持，返回 None/False。
    # =========================================================================

    def read_cartesian_position(self) -> dict[str, float] | None:
        """读取末端执行器位姿。

        返回包含以下键的字典：x, y, z（米），roll, pitch, yaw（弧度）。
        如果不支持返回 None。
        """
        return None  # 如果你的 SDK 支持，可以在此实现

    def write_cartesian_position(
        self,
        pose: dict[str, float],
        velocity: float = 1.0,
    ) -> bool:
        """发送末端执行器位姿指令。如果不支持返回 False。"""
        return False

    # =========================================================================
    # 可选：夹爪
    # =========================================================================

    def read_gripper_position(self) -> float | None:
        """读取夹爪位置（米）。如果没有夹爪返回 None。"""
        return None

    def write_gripper_position(self, position: float) -> bool:
        """发送夹爪位置指令（米）。如果没有夹爪返回 False。"""
        return False

    # =========================================================================
    # 可选：力/力矩传感器
    # =========================================================================

    def read_force_torque(self) -> list[float] | None:
        """读取力/力矩传感器数据 [fx, fy, fz, tx, ty, tz]。无传感器返回 None。"""
        return None


# ── 注册表钩子（自动发现所需）───────────────────
def register(registry: AdapterRegistry) -> None:
    """将此适配器注册到注册表中。"""
    registry.register("yourarm", YourArmAdapter)


__all__ = ["YourArmAdapter"]
```

### 关键实现注意事项

- **不支持的功能** — 读取操作返回 `None`，写入操作返回 `False`。对于可选功能，切勿抛出异常。
- **速度/力矩反馈** — 如果你的 SDK 不提供这些数据，返回零值。协调器会妥善处理这种情况。
- **延迟导入 SDK** — 如果厂商 SDK 是可选依赖，可以在 `connect()` 内部导入而非在模块级别导入（参见 Piper 适配器的实现方式）：
  ```python
  def connect(self) -> bool:
      try:
          from yourarm_sdk import YourArmSDK
          self._sdk = YourArmSDK(self._address)
          ...
      except ImportError:
          print("ERROR: yourarm-sdk not installed. Run: pip install yourarm-sdk")
          return False
  ```

## 步骤 2：创建包文件

### \_\_init\_\_.py

```python
"""YourArm 操控臂硬件适配器。

用法：
    >>> from dimos.hardware.manipulators.yourarm import YourArmAdapter
    >>> adapter = YourArmAdapter(address="192.168.1.100", dof=6)
    >>> adapter.connect()
    >>> positions = adapter.read_joint_positions()
"""

from dimos.hardware.manipulators.yourarm.adapter import YourArmAdapter

__all__ = ["YourArmAdapter"]
```

### 自动发现机制

`dimos/hardware/manipulators/registry.py` 中的 `AdapterRegistry` 在导入时自动发现你的适配器：

1. 它遍历 `dimos/hardware/manipulators/` 下的所有子包
2. 对于每个子包，尝试导入 `<subpackage>.adapter`
3. 如果该模块有 `register()` 函数，则调用它

这意味着**无需手动注册**——只需在 `adapter.py` 中包含 `register()` 函数即可。

你可以验证自动发现是否正常工作：

```python
from dimos.hardware.manipulators.registry import adapter_registry
print(adapter_registry.available())  # 应包含 "yourarm"
```

## 步骤 3：创建机器人目录和蓝图

DimOS 中的每个机器人在 `dimos/robot/` 下都有自己的目录。在这里你可以定义机械臂的所有蓝图——协调器、规划、感知等。这遵循与 Unitree 机器人（`dimos/robot/unitree/`）相同的模式。

### 3a. 创建机器人目录

```
dimos/robot/
├── unitree/                 # Unitree 机器人（参考示例）
│   ├── go2/
│   │   └── blueprints/
│   └── g1/
│       └── blueprints/
└── yourarm/                 # ← 你的机器人的新目录
    ├── __init__.py
    └── blueprints.py
```

### 3b. 定义蓝图

创建 `dimos/robot/yourarm/blueprints.py`，包含你的协调器和（可选的）规划蓝图：

```python
"""YourArm 机器人蓝图。

用法：
    # 通过 CLI 运行：
    dimos run coordinator-yourarm          # 启动带真实硬件的协调器
    dimos run yourarm-planner              # 启动规划器（可选，用于运动规划）

    # 或通过编程方式：
    from dimos.robot.yourarm.blueprints import coordinator_yourarm
    coordinator = coordinator_yourarm.build()
    coordinator.loop()
"""

from __future__ import annotations

from pathlib import Path

from dimos.control.components import HardwareComponent, HardwareType, make_joints
from dimos.control.coordinator import TaskConfig, control_coordinator
from dimos.core.transport import LCMTransport
from dimos.msgs.sensor_msgs import JointState

# =============================================================================
# 协调器蓝图
# =============================================================================

# YourArm (6-DOF) — 真实硬件
coordinator_yourarm = control_coordinator(
    tick_rate=100.0,                    # 控制循环频率（Hz）
    publish_joint_state=True,           # 发布聚合的关节状态
    joint_state_frame_id="coordinator",
    hardware=[
        HardwareComponent(
            hardware_id="arm",                        # 此硬件的唯一 ID
            hardware_type=HardwareType.MANIPULATOR,
            joints=make_joints("arm", 6),             # 创建 ["arm_joint1", ..., "arm_joint6"]
            adapter_type="yourarm",                   # 必须与注册表名称匹配
            address="192.168.1.100",                  # 传递给适配器 __init__
            auto_enable=True,                         # 启动时自动启用伺服电机
        ),
    ],
    tasks=[
        TaskConfig(
            name="traj_arm",                          # 任务名称（由 ManipulationModule RPC 使用）
            type="trajectory",                        # 轨迹执行任务
            joint_names=[f"arm_joint{i+1}" for i in range(6)],
            priority=10,                              # 更高优先级赢得仲裁
        ),
    ],
).transports(
    {
        ("joint_state", JointState): LCMTransport("/coordinator/joint_state", JointState),
    }
)


```

### 蓝图字段参考

| 字段 | 描述 |
|------|------|
| `hardware_id` | 此硬件组件的唯一名称，用于路由指令。 |
| `adapter_type` | 在 `adapter_registry` 中注册的名称（例如 `"yourarm"`）。 |
| `address` | 传递给适配器 `__init__` 的 `address` 参数的连接信息。 |
| `joints` | 关节名称列表。`make_joints("arm", 6)` 创建 `["arm_joint1", ..., "arm_joint6"]`。 |
| `auto_enable` | 如果为 `True`，协调器启动时自动启用伺服电机。 |
| `task.name` | ManipulationModule 通过 RPC 调用轨迹执行时使用的名称。 |
| `task.type` | 任务类型：`"trajectory"`、`"servo"`、`"velocity"` 或 `"cartesian_ik"`。 |
| `task.priority` | 按关节仲裁的优先级。数字越大优先级越高。 |

## 步骤 4：添加 URDF 和规划集成（可选）

如果你需要运动规划（通过 Drake 生成无碰撞轨迹），则需要 URDF 和规划蓝图。将这些添加到你机器人自己的 `blueprints.py` 中。

### 4a. 添加 URDF

将你的 URDF/xacro 文件放在 LFS 数据目录下，以便通过 `LfsPath` 解析。`LfsPath` 是 `Path` 的子类，在首次访问时延迟下载 LFS 数据——这避免了在加载蓝图模块时进行导入时下载。

```python
from dimos.utils.data import LfsPath
from dimos.manipulation.manipulation_module import manipulation_module
from dimos.manipulation.planning.spec import RobotModelConfig
from dimos.msgs.geometry_msgs import PoseStamped, Quaternion, Vector3

# LfsPath 在路径被实际访问时才触发下载
_YOURARM_URDF_PATH = LfsPath("yourarm_description/urdf/yourarm.urdf")
_YOURARM_PACKAGE_PATH = LfsPath("yourarm_description")


def _make_base_pose(x=0.0, y=0.0, z=0.0) -> PoseStamped:
    return PoseStamped(
        position=Vector3(x=x, y=y, z=z),
        orientation=Quaternion(0.0, 0.0, 0.0, 1.0),
    )
```

### 4b. 创建机器人模型配置辅助函数

```python
def _make_yourarm_config(
    name: str = "arm",
    y_offset: float = 0.0,
    joint_prefix: str = "",
    coordinator_task: str | None = None,
) -> RobotModelConfig:
    """创建 YourArm 机器人规划配置。

    Args:
        name: Drake 规划世界中的机器人名称。
        y_offset: 多臂设置的 Y 轴偏移。
        joint_prefix: 映射到协调器命名空间的关节名称前缀。
        coordinator_task: 通过 RPC 执行轨迹的协调器任务名称。
    """
    # 这些必须与你的 URDF 中的关节名称匹配
    joint_names = ["joint1", "joint2", "joint3", "joint4", "joint5", "joint6"]
    joint_mapping = {f"{joint_prefix}{j}": j for j in joint_names} if joint_prefix else {}

    return RobotModelConfig(
        name=name,
        urdf_path=_YOURARM_URDF_PATH,
        base_pose=_make_base_pose(y=y_offset),
        joint_names=joint_names,
        end_effector_link="link6",      # URDF 运动链中的最后一个连杆
        base_link="base_link",          # URDF 的根连杆
        package_paths={"yourarm_description": _YOURARM_PACKAGE_PATH},
        xacro_args={},                  # 使用 .xacro 文件时的 xacro 参数
        collision_exclusion_pairs=[],   # 可以接触的连杆对（例如夹爪手指）
        auto_convert_meshes=True,       # 为 Drake 转换 DAE/STL 网格
        max_velocity=1.0,               # 最大速度缩放系数
        max_acceleration=2.0,           # 最大加速度缩放系数
        joint_name_mapping=joint_mapping,
        coordinator_task_name=coordinator_task,
    )
```

### 4c. 创建规划蓝图

将以下内容添加到你的 `dimos/robot/yourarm/blueprints.py` 中，与协调器蓝图放在一起：

```python
# =============================================================================
# 规划器蓝图（需要 URDF）
# =============================================================================

yourarm_planner = manipulation_module(
    robots=[_make_yourarm_config("arm", joint_prefix="arm_", coordinator_task="traj_arm")],
    planning_timeout=10.0,
    enable_viz=True,
).transports(
    {
        ("joint_state", JointState): LCMTransport("/coordinator/joint_state", JointState),
    }
)
```

### 关键配置字段

| 字段 | 描述 |
|------|------|
| `urdf_path` | `.urdf` 或 `.xacro` 文件的路径 |
| `joint_names` | 受控关节的有序列表（必须与 URDF 匹配） |
| `end_effector_link` | 用作 IK 末端执行器的连杆 |
| `base_link` | 机器人模型的根连杆 |
| `package_paths` | 将 `package://` URI 映射到文件系统路径（用于 xacro） |
| `joint_name_mapping` | 将协调器名称（例如 `"arm_joint1"`）映射到 URDF 名称（例如 `"joint1"`） |
| `coordinator_task_name` | 必须与协调器蓝图中的 `TaskConfig.name` 匹配 |
| `collision_exclusion_pairs` | 可以合法接触的 `(link_a, link_b)` 元组列表（例如夹爪手指） |

## 步骤 5：注册蓝图

`dimos/robot/all_blueprints.py` 中的蓝图注册表是通过扫描代码库中的蓝图声明**自动生成**的。添加蓝图后：

1. 运行生成测试以更新注册表：
   ```bash
   pytest dimos/robot/test_all_blueprints_generation.py
   ```
3. 现在你可以通过 CLI 运行你的机械臂：
   ```bash
   dimos run coordinator-yourarm
   dimos run yourarm-planner        # 如果你添加了规划蓝图
   ```

## 步骤 6：测试

### 验证适配器注册

```python
from dimos.hardware.manipulators.registry import adapter_registry

# 检查你的适配器是否出现
assert "yourarm" in adapter_registry.available()

# 通过注册表创建实例（与协调器使用的路径相同）
adapter = adapter_registry.create("yourarm", address="192.168.1.100", dof=6)
```

### 使用 mock 进行单元测试

你可以使用 `unittest.mock` 在没有硬件的情况下测试协调器逻辑：

```python
import pytest
from unittest.mock import MagicMock
from dimos.hardware.manipulators.spec import ManipulatorAdapter

@pytest.fixture
def mock_adapter():
    adapter = MagicMock(spec=ManipulatorAdapter)
    adapter.get_dof.return_value = 6
    adapter.read_joint_positions.return_value = [0.0] * 6
    adapter.read_joint_velocities.return_value = [0.0] * 6
    adapter.read_joint_efforts.return_value = [0.0] * 6
    adapter.write_joint_positions.return_value = True
    adapter.read_enabled.return_value = True
    adapter.is_connected.return_value = True
    return adapter

def test_read_positions(mock_adapter):
    assert mock_adapter.read_joint_positions() == [0.0] * 6

def test_write_positions(mock_adapter):
    target = [0.1, 0.2, 0.3, 0.4, 0.5, 0.6]
    assert mock_adapter.write_joint_positions(target) is True
```

### 与协调器进行集成测试

```python
from dimos.control.blueprints import coordinator_mock

# 使用模拟硬件构建并启动协调器
coordinator = coordinator_mock.build()
coordinator.start()

# 你的适配器通过相同的协调器接口进行测试
# 只需在蓝图中将 adapter_type="mock" 换成 adapter_type="yourarm"
```

### 独立测试真实适配器

```python
from dimos.hardware.manipulators.yourarm import YourArmAdapter

adapter = YourArmAdapter(address="192.168.1.100", dof=6)
assert adapter.connect() is True
assert adapter.is_connected() is True

# 读取状态
positions = adapter.read_joint_positions()
assert len(positions) == 6
print(f"Joint positions (rad): {positions}")

# 启用并移动
adapter.write_enable(True)
adapter.write_joint_positions([0.0] * 6)

# 清理
adapter.write_stop()
adapter.disconnect()
```

## 快速参考清单

需要创建的文件：

- [ ] `dimos/hardware/manipulators/yourarm/__init__.py`
- [ ] `dimos/hardware/manipulators/yourarm/adapter.py`（实现 Protocol + `register()`）
- [ ] `dimos/robot/yourarm/__init__.py`
- [ ] `dimos/robot/yourarm/blueprints.py`（协调器 + 规划蓝图）

需要修改的文件：

- [ ] `pyproject.toml` — 将厂商 SDK 添加到可选依赖中*（如适用）*

验证：

- [ ] `adapter_registry.available()` 包含 `"yourarm"`
- [ ] `pytest dimos/robot/test_all_blueprints_generation.py` 通过（重新生成 `all_blueprints.py`）
- [ ] `dimos run coordinator-yourarm` 成功启动
