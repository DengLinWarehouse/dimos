# Teleop Stack（遥操作栈）

DimOS 的遥操作模块。支持 Meta Quest 3 VR 控制器和手机运动传感器。

## 架构

```
Quest/Phone Browser
    │
    │  LCM-encoded binary via WebSocket
    ▼
Embedded FastAPI Server (HTTPS)
    │
    │  Fingerprint-based message dispatch
    ▼
TeleopModule (Quest or Phone)
    │  Frame transforms + pose/twist computation
    ▼
PoseStamped / TwistStamped / Buttons outputs
```

每个遥操作模块都会内嵌一个 `RobotWebInterface`（FastAPI + uvicorn），它负责：
- 在 `/teleop` 提供遥操作 Web 应用
- 在 `/ws` 接受 WebSocket 连接
- 处理 HTTPS 所需的 SSL 证书生成（移动端传感器 API 需要）

## 模块

### QuestTeleopModule
基础 Quest 遥操作模块。通过 WebSocket 获取控制器数据，计算输出位姿并发布。默认启用方式：按住主按钮（X/A）。可通过子类化进行自定义。

### ArmTeleopModule
基于切换的启用方式——按一次主按钮启用，再按一次解除启用。

### TwistTeleopModule
输出 `TwistStamped`（线速度 + 角速度），而不是 `PoseStamped`。

### VisualizingTeleopModule
增加用于调试的 Rerun 可视化。继承自 `ArmTeleopModule`（切换式启用）。

### PhoneTeleopModule
基础手机遥操作模块。接收手机运动传感器提供的姿态和陀螺仪数据，并根据姿态增量计算速度命令。

### SimplePhoneTeleop
将输出限制为移动底盘相关轴（`linear.x`、`linear.y`、`angular.z`），并以 `Twist` 形式发布。

## 子类化

`QuestTeleopModule` 被设计为可扩展模块。可以重写以下方法：

| 方法 | 用途 |
|--------|---------|
| `_handle_engage()` | 自定义启用/解除启用逻辑 |
| `_should_publish()` | 增加发布条件 |
| `_get_output_pose()` | 自定义位姿计算 |
| `_publish_msg()` | 修改输出格式 |
| `_publish_button_state()` | 修改按钮输出 |

### 子类规则

- **不要在重写方法中获取 `self._lock`。** 控制循环已经持有它。
  可以直接访问 `self._controllers`、`self._current_poses`、`self._is_engaged` 等属性。
- **保持重写逻辑足够快**——这些方法会在 `control_loop_hz` 的控制循环中运行。

## 文件结构

```
teleop/
├── quest/
│   ├── quest_teleop_module.py   # Base Quest teleop module
│   ├── quest_extensions.py      # ArmTeleop, TwistTeleop, VisualizingTeleop
│   ├── quest_types.py           # QuestControllerState, Buttons
│   └── web/
│       └── static/index.html    # WebXR client
├── phone/
│   ├── phone_teleop_module.py   # Base Phone teleop module
│   ├── phone_extensions.py      # SimplePhoneTeleop
│   ├── blueprints.py            # Pre-wired configurations
│   └── web/
│       └── static/index.html    # Mobile sensor web app
├── utils/
│   ├── teleop_transforms.py     # WebXR → robot frame math
│   └── teleop_visualization.py  # Rerun visualization helpers
└── blueprints.py                # Module blueprints for easy instantiation
```

## 快速开始

```bash
dimos run arm-teleop            # Quest arm teleop
dimos run phone-go2-teleop      # Phone → Go2
```

在设备上打开 `https://<host-ip>:<port>/teleop`。接受自签名证书。
- Quest：端口 8443
- Phone：端口 8444
