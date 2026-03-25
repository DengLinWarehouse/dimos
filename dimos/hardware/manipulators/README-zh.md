# Manipulator Drivers（机械臂驱动）

本模块提供机械臂驱动：仅协议定义，支持可注入的适配器。

## 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                      Driver (Module)                        │
│  - 拥有线程管理（控制循环、监控循环）                            │
│  - 发布 joint_state、robot_state                             │
│  - 订阅 joint_position_command、joint_velocity_cmd           │
│  - 暴露 RPC 方法（move_joint、enable_servos 等）              │
└─────────────────────┬───────────────────────────────────────┘
                      │ uses
┌─────────────────────▼───────────────────────────────────────┐
│              Adapter（实现 Protocol）                         │
│  - 处理 SDK 通信                                              │
│  - 单位转换（弧度 ↔ 厂商单位）                                 │
│  - 可替换：XArmAdapter、PiperAdapter、MockAdapter             │
└─────────────────────────────────────────────────────────────┘
```

## 主要优势

- **可测试**：注入 `MockAdapter` 即可在无硬件环境下进行单元测试
- **灵活**：每条臂独立控制自己的线程/时序
- **简洁**：无需 ABC 继承——只需实现 Protocol
- **类型安全**：通过 `ManipulatorAdapter` Protocol 实现完整的类型检查

## 目录结构

```
manipulators/
├── spec.py              # ManipulatorAdapter Protocol + 共享类型
├── registry.py          # 适配器注册表，支持自动发现
├── mock/
│   └── adapter.py       # MockAdapter，用于测试
├── xarm/
│   ├── adapter.py       # XArmAdapter（SDK 包装器）
└── piper/
    ├── adapter.py       # PiperAdapter（SDK 包装器）
```

## 快速开始

### 直接使用驱动

```python
from dimos.hardware.manipulators.xarm import XArm

arm = XArm(ip="192.168.1.185", dof=6)
arm.start()
arm.enable_servos()
arm.move_joint([0, 0, 0, 0, 0, 0])
arm.stop()
```

### 使用 Blueprints

```python
from dimos.hardware.manipulators.xarm.blueprints import xarm_trajectory

coordinator = xarm_trajectory.build()
coordinator.loop()
```

### 无硬件测试

```python
from dimos.hardware.manipulators.mock import MockAdapter
from dimos.hardware.manipulators.xarm import XArm

arm = XArm(adapter=MockAdapter(dof=6))
arm.start()  # 无需硬件！
arm.move_joint([0.1, 0.2, 0.3, 0.4, 0.5, 0.6])
```

## 添加新的机械臂

1. **创建适配器**（`adapter.py`）：

```python
class MyArmAdapter:  # 无需继承——只需匹配 Protocol
    def __init__(self, ip: str = "192.168.1.100", dof: int = 6) -> None:
        self._ip = ip
        self._dof = dof

    def connect(self) -> bool: ...
    def disconnect(self) -> None: ...
    def read_joint_positions(self) -> list[float]: ...
    def write_joint_positions(self, positions: list[float], velocity: float = 1.0) -> bool: ...
    # ... 实现其他 Protocol 方法
```

2. **创建驱动**（`arm.py`）：

```python
from dimos.core.core import rpc
from dimos.core.module import Module, ModuleConfig
from dimos.core.stream import In, Out
from .adapter import MyArmAdapter

class MyArm(Module[MyArmConfig]):
    joint_state: Out[JointState]
    robot_state: Out[RobotState]
    joint_position_command: In[JointCommand]

    def __init__(self, adapter=None, **kwargs):
        super().__init__(**kwargs)
        self.adapter = adapter or MyArmAdapter(
            ip=self.config.ip,
            dof=self.config.dof,
        )
        # ... 设置控制循环
```

3. **创建 blueprints**（`blueprints.py`），用于常见配置。

## ManipulatorAdapter Protocol

所有适配器必须实现以下核心方法：

| 类别 | 方法 |
|----------|---------|
| 连接 | `connect()`、`disconnect()`、`is_connected()` |
| 信息 | `get_info()`、`get_dof()`、`get_limits()` |
| 状态 | `read_joint_positions()`、`read_joint_velocities()`、`read_joint_efforts()` |
| 运动 | `write_joint_positions()`、`write_joint_velocities()`、`write_stop()` |
| 伺服 | `write_enable()`、`read_enabled()`、`write_clear_errors()` |
| 模式 | `set_control_mode()`、`get_control_mode()` |

可选方法（不支持时返回 `None`/`False`）：
- `read_cartesian_position()`、`write_cartesian_position()`
- `read_gripper_position()`、`write_gripper_position()`
- `read_force_torque()`

## 单位约定

所有适配器均转换为/从 SI 单位：

| 物理量 | 单位 |
|----------|------|
| 角度 | 弧度（radians） |
| 角速度 | rad/s |
| 力矩 | Nm |
| 位置 | 米（meters） |
| 力 | 牛顿（Newtons） |
