# Quest Teleop（Quest 遥操作）

通过 Meta Quest 3 VR 控制器进行遥操作。支持基于 WebXR（Web 扩展现实）的双手跟踪。

## 架构

```
Quest Browser  ──WebSocket──→  Embedded HTTPS Server  ──→  QuestTeleopModule
(WebXR poses + Joy)             (port 8443)                  (delta → PoseStamped)
```

## 运行方式

```bash
dimos run arm-teleop           # Basic arm teleop
dimos run arm-teleop-xarm6     # XArm6
dimos run arm-teleop-piper     # Piper
dimos run arm-teleop-dual      # Dual arm
```

在 Quest 浏览器中打开 `https://<host-ip>:8443/teleop`。接受证书，然后点击 Connect。

## 子类化

| 方法 | 用途 |
|--------|---------|
| `_handle_engage()` | 自定义启用/解除启用逻辑 |
| `_should_publish()` | 为发布增加额外条件 |
| `_get_output_pose()` | 自定义位姿计算 |
| `_publish_msg()` | 修改输出格式 |

`self._lock` 已经被持有——不要在重写方法中再次获取它。

## Joy 消息格式

**Axes**：thumbstick X、thumbstick Y、trigger（模拟量）、grip（模拟量）

**Buttons**：trigger、grip、touchpad、thumbstick、X/A、Y/B、menu

## 文件结构

```
quest/
├── quest_teleop_module.py   # Base module
├── quest_extensions.py      # ArmTeleop, TwistTeleop, VisualizingTeleop
├── quest_types.py           # QuestControllerState, Buttons
├── blueprints.py
└── web/static/index.html    # WebXR client
```
