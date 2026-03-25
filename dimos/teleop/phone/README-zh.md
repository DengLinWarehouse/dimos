# Phone Teleop（手机遥操作）

通过智能手机运动传感器进行遥操作。倾斜即可驱动。

## 架构

```
Phone Browser  ──WebSocket──→  Embedded HTTPS Server  ──→  PhoneTeleopModule
(sensors + button)              (port 8444)                  (delta → velocity)
```

## 运行方式

```bash
dimos run phone-go2-teleop     # Go2
dimos run simple-phone-teleop  # Generic ground robot
```

在手机上打开 `https://<host-ip>:8444/teleop`。接受证书、允许传感器权限、连接后按住即可驾驶。

## 子类化

| 方法 | 用途 |
|--------|---------|
| `_handle_engage()` | 自定义启用/解除启用逻辑 |
| `_publish_msg()` | 修改输出格式 |

`self._lock` 已经被持有——不要在重写方法中再次获取它。

## 文件结构

```
phone/
├── phone_teleop_module.py   # Base module
├── phone_extensions.py      # SimplePhoneTeleop
├── blueprints.py
└── web/static/index.html    # Mobile web app
```
