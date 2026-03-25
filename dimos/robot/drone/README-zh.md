# DimOS Drone Module（DimOS 无人机模块）

通过 RosettaDrone（DJI SDK 到 MAVLink 的桥接应用）集成 DJI 无人机，支持视觉伺服、自主跟踪和 LLM（Large Language Model，大语言模型）智能体控制。

## 快速开始

```bash
# Replay mode (no hardware needed)
dimos --replay run drone-basic

# Agentic mode with replay
dimos --replay run drone-agentic

# Real drone — indoor (velocity-based odometry)
dimos run drone-basic

# Real drone — outdoor (GPS-based odometry)
dimos run drone-basic --set outdoor=true

# Agentic with LLM control
dimos run drone-agentic
```

若要与智能体交互，请在另一个终端中运行 `dimos humancli`。

## Blueprints

### `drone-basic`
连接 + 相机 + 可视化，是基础层。

| 模块 | 用途 |
|--------|---------|
| `DroneConnectionModule` | MAVLink 通信、运动技能 |
| `DroneCameraModule` | 相机内参、图像处理 |
| `WebsocketVisModule` | 基于 Web 的可视化 |
| `RerunBridgeModule` / `FoxgloveBridge` | 3D 查看器（由 `--viewer` 选择） |

**室内 vs 室外：** 默认情况下，无人机会使用速度积分进行里程计估计（室内模式）。对于带 GPS 的室外飞行，请设置 `outdoor=true`——这会切换为仅基于 GPS 的定位方式，在开阔环境中更可靠，但在近距离机动时精度较低。

### `drone-agentic`
在 `drone-basic` 之上继续组合，增加自主能力：

| 模块 | 用途 |
|--------|---------|
| `DroneTrackingModule` | 视觉伺服与目标跟踪 |
| `GoogleMapsSkillContainer` | 基于 GPS 的导航技能 |
| `OsmSkill` | OpenStreetMap 查询 |
| `Agent` | LLM 智能体（默认：GPT-4o） |
| `WebInput` | 面向人工命令的 Web/CLI 接口 |

## 安装

### Python（随 DimOS 提供）
```bash
pip install -e ".[drone]"
```

### 系统依赖
```bash
# GStreamer for video streaming
sudo apt-get install -y gstreamer1.0-tools gstreamer1.0-plugins-base     gstreamer1.0-plugins-good gstreamer1.0-plugins-bad     gstreamer1.0-libav python3-gi python3-gi-cairo

# LCM for communication
sudo apt-get install liblcm-dev
```

### 环境变量
```bash
# Required for agentic blueprint
export OPENAI_API_KEY=sk-...

# Optional
export GOOGLE_MAPS_API_KEY=...  # For GoogleMapsSkillContainer
```

## RosettaDrone 设置（关键）

RosettaDrone 是一个 Android 应用，可将 DJI SDK 桥接到 MAVLink 协议。没有它，无人机就无法与 DimOS 通信。

### 方案 1：预编译 APK
1. 下载最新版本：https://github.com/RosettaDrone/rosettadrone/releases
2. 安装到连接了 DJI 遥控器的 Android 设备上
3. 在应用中配置：
   - MAVLink Target IP：你的计算机 IP
   - MAVLink Port：14550
   - Video Port：5600
   - 启用视频流

### 方案 2：从源码构建

#### 前置条件
- Android Studio
- DJI Developer Account：https://developer.dji.com/
- Git

#### 构建步骤
```bash
# Clone repository
git clone https://github.com/RosettaDrone/rosettadrone.git
cd rosettadrone

# Build with Gradle
./gradlew assembleRelease

# APK will be in: app/build/outputs/apk/release/
```

#### 配置 DJI API Key
1. 在 https://developer.dji.com/user/apps 注册应用
   - 包名：`sq.rogue.rosettadrone`
2. 将 key 添加到 `app/src/main/AndroidManifest.xml`：
```xml
<meta-data
    android:name="com.dji.sdk.API_KEY"
    android:value="YOUR_API_KEY_HERE" />
```

#### 安装 APK
```bash
adb install -r app/build/outputs/apk/release/rosettadrone-release.apk
```

### 硬件连接
```
DJI Drone ← Wireless → DJI Controller ← USB → Android Device ← WiFi → DimOS Computer
```

1. 通过 USB 将 Android 设备连接到 DJI 遥控器
2. 启动 RosettaDrone 应用
3. 等待显示 “DJI Connected” 状态
4. 确认应用中显示 “MAVLink Active”

## 架构

### 模块结构
```
dimos/robot/drone/
├── blueprints/
│   ├── basic/drone_basic.py              # Base blueprint (connection + camera + vis)
│   └── agentic/drone_agentic.py          # Agentic blueprint (composes on basic)
├── connection_module.py                   # MAVLink communication & skills
├── camera_module.py                       # Camera processing & intrinsics
├── drone_tracking_module.py               # Visual servoing & object tracking
├── drone_visual_servoing_controller.py    # PID-based visual servoing
├── mavlink_connection.py                  # Low-level MAVLink protocol
└── dji_video_stream.py                    # GStreamer video capture + replay
```

### 通信流
```
DJI Drone → RosettaDrone → MAVLink UDP → connection_module → LCM Topics
                         → Video UDP   → dji_video_stream → tracking_module
```

### LCM 话题
- `/video` — 相机帧（`sensor_msgs.Image`）
- `/odom` — 位置和朝向（`geometry_msgs.PoseStamped`）
- `/movecmd_twist` — 速度命令（`geometry_msgs.Twist`）
- `/gps_location` — GPS 坐标（`LatLon`）
- `/gps_goal` — GPS 导航目标（`LatLon`）
- `/tracking_status` — 跟踪模块状态
- `/follow_object_cmd` — 目标跟踪命令
- `/color_image` — 处理后的相机图像
- `/camera_info` — 相机内参
- `/camera_pose` — 世界坐标系中的相机位姿

## 视觉伺服与跟踪

### 目标跟踪
```python
# Track specific object
result = drone.tracking.track_object("red flag", duration=60)

# Track nearest/most prominent object
result = drone.tracking.track_object(None, duration=60)

# Stop tracking
drone.tracking.stop_tracking()
```

### PID 调参
```python
# Indoor (gentle, precise)
x_pid_params=(0.001, 0.0, 0.0001, (-0.5, 0.5), None, 30)

# Outdoor (aggressive, wind-resistant)
x_pid_params=(0.003, 0.0001, 0.0002, (-1.0, 1.0), None, 10)
```

参数含义：`(Kp, Ki, Kd, (min_output, max_output), integral_limit, deadband_pixels)`

### 视觉伺服流程
1. Qwen 模型检测目标 → 边界框
2. 在 bbox 上初始化 CSRT 跟踪器
3. PID 控制器根据像素误差计算速度
4. 通过 LCM 流发送速度命令
5. 连接模块将其转换为 MAVLink 命令

## 可用技能

所有技能都通过 `DroneConnectionModule` 上的 `@skill` 装饰器暴露给 LLM 智能体：

### 运动与控制
- `move(x, y, z, duration)` — 按速度移动（m/s）
- `takeoff(altitude)` — 起飞到指定高度
- `land()` — 在当前位置降落
- `arm()` / `disarm()` — 电机解锁/上锁
- `set_mode(mode)` — 设置飞行模式（`GUIDED`、`LOITER` 等）
- `fly_to(lat, lon, alt)` — 飞往指定 GPS 坐标

### 感知
- `observe()` — 获取当前相机帧
- `follow_object(description, duration)` — 使用视觉伺服跟随目标
- `is_flying_to_target()` — 检查是否正在飞往 GPS 目标

## 回放模式

回放数据包括：
- **2,148 帧视频**（640×360 RGB，约 71 秒，30fps）
- **4,098 帧 MAVLink 遥测数据**（约 136 秒）

这些数据以 `TimedSensorStorage` `pickle` 文件形式存储在 `data/drone/` 中，并会在首次使用时自动下载。

```bash
# Basic replay
dimos --replay run drone-basic

# Agentic replay (requires OPENAI_API_KEY)
dimos --replay run drone-agentic
```

## 可视化

### Rerun Viewer（推荐）
```bash
dimos --viewer rerun run drone-basic
```

分屏布局，包含相机画面和 3D 世界视图。内置静态无人机机体可视化和 LCM 传输集成。

### Foxglove Studio
```bash
dimos --viewer foxglove run drone-basic
```

将 Foxglove Studio 连接到 `ws://localhost:8765` 后可查看：
- 带跟踪叠加层的实时视频
- 3D 无人机位置
- 遥测曲线
- 变换树

### Web 可视化
始终可通过 `WebsocketVisModule` 在 `http://localhost:7779` 访问。

## 测试

```bash
# Unit tests
pytest -s dimos/robot/drone/

# Replay integration test
dimos --replay run drone-basic
```

## 故障排查

### 没有 MAVLink 连接
- 检查 Android 设备和电脑是否在同一网络
- 确认 RosettaDrone 中配置的 IP 地址与电脑一致
- 使用 `nc -lu 14550` 测试（应能看到数据）
- 检查防火墙：`sudo ufw allow 14550/udp`

### 没有视频流
- 在 RosettaDrone 设置中启用视频
- 使用 `nc -lu 5600` 测试（应能看到数据）
- 确认已安装 GStreamer：`gst-launch-1.0 --version`

### 跟踪问题
- 提高光照条件以获得更好的检测效果
- 根据环境调整 PID 增益
- 检查跟踪模块中的 `max_lost_frames`

### 智能体无响应
- 确认已设置 `OPENAI_API_KEY`
- 运行 `dimos humancli` 发送命令
- 检查日志中的 `on_system_modules` 错误

### 运动方向错误
- 不要修改坐标转换逻辑
- 使用 `pytest test_drone.py::test_ned_to_ros_coordinate_conversion` 验证
- 检查相机朝向假设

## 网络端口

| 端口 | 协议 | 用途 |
|------|----------|---------|
| 14550 | UDP | MAVLink 命令/遥测 |
| 5600 | UDP | 视频流 |
| 7779 | WebSocket | DimOS Web 可视化 |
| 8765 | WebSocket | Foxglove bridge |
| 7667 | UDP | LCM 消息传输 |

## 坐标系
- **MAVLink/NED**：X=North，Y=East，Z=Down
- **ROS/DimOS**：X=Forward，Y=Left，Z=Up
- 内部会自动完成坐标转换

## 修改 PID 控制
- 增大 Kp 可获得更快响应
- 加入 Ki 可消除稳态误差
- 增大 Kd 可增加阻尼
- 调整限制值可改变最大速度

## 安全说明
- 请始终先在模拟器中或拆下桨叶后进行测试
- 初始时使用保守的 PID 增益
- 在室外飞行中实现地理围栏
- 持续监控电池电压
- 始终准备好手动接管
