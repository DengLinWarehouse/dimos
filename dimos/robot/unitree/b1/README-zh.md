# Unitree B1 Dimensional Integration（Unitree B1 集成）

该模块为 Unitree B1 四足机器人提供基于 UDP 的控制，并通过 ROS `Twist` `cmd_vel` 接口与 DimOS 集成。

## 概览

系统由两个部分组成：
1. **服务端**：运行在 B1 内部计算机上的 C++ UDP 服务器
2. **客户端**：运行在外部机器上的 Python 控制模块

主要特性：
- 50Hz 持续 UDP 流发送
- 100ms 命令超时自动停车
- 标准 `Twist` 速度接口
- 紧急停止（`Space`/`Q` 键）
- `IDLE`/`STAND`/`WALK` 模式控制
- 可选 `pygame` 摇杆接口

## 服务端设置（B1 内部计算机）

### 前置条件

B1 机器人运行 Ubuntu，并需要满足以下要求：
- Unitree Legged SDK v3.8.3 for B1
- Boost（>= 1.71.0）
- CMake（>= 3.16.3）
- g++（>= 9.4.0）

### 第 1 步：连接到 B1 机器人

1. **连接到 B1 的 WiFi 接入点：**
   - SSID：`Unitree_B1_XXXXX`（其中 XXXXX 是你的机器人 ID）
   - 密码：`00000000`（8 个 0）

2. **通过 SSH 登录 B1：**
   ```bash
   ssh unitree@192.168.12.1
   # Default password: 123
   ```

### 第 2 步：构建 UDP 服务器

1. **将 `joystick_server_udp.cpp` 添加到 `CMakeLists.txt`：**
   ```bash
   # Edit the CMakeLists.txt in the unitree_legged_sdk_B1 directory
   vim CMakeLists.txt

   # Add this line with the other add_executable statements:
   add_executable(joystick_server example/joystick_server_udp.cpp)
   target_link_libraries(joystick_server ${EXTRA_LIBS})```

2. **构建服务端：**
   ```bash
   mkdir build
   cd build
   cmake ../
   make
   ```

### 第 3 步：运行 UDP 服务器

```bash
# Navigate to build directory
cd Unitree/sdk/unitree_legged_sdk_B1/build/
./joystick_server

# You should see:
# UDP Unitree B1 Joystick Control Server
# Communication level: HIGH-level
# Server port: 9090
# WARNING: Make sure the robot is standing on the ground.
# Press Enter to continue...
```

服务器随后会在 9090 端口监听 UDP 数据包，并控制 B1 机器人。

### 服务端安全特性

- **100ms 超时**：若 100ms 内未收到数据包，机器人会停止
- **数据包校验**：仅接受格式正确的 19 字节数据包
- **模式限制**：只有在 `WALK` 模式下才会应用速度命令
- **紧急停止**：模式 0（`IDLE`）会停止所有运动

## 客户端设置（外部机器）

### 前置条件

- Python 3.10+
- 已安装 DimOS 框架
- `pygame`（可选，用于摇杆控制）

### 第 1 步：安装依赖

```bash
# Install Dimensional
pip install -e .[cpu,sim]
```

### 第 2 步：连接到 B1 网络

1. **将你的机器连接到 B1 的 WiFi：**
   - SSID：`Unitree_B1_XXXXX`
   - 密码：`00000000`

2. **验证连接：**
   ```bash
   ping 192.168.12.1  # Should get responses
   ```

### 第 3 步：运行客户端

#### 使用摇杆控制（推荐用于测试）

```bash
python -m dimos.robot.unitree.b1.unitree_b1     --ip 192.168.12.1     --port 9090     --joystick
```

**摇杆控制说明：**
- `0/1/2` - 在 `IDLE`/`STAND`/`WALK` 模式间切换
- `WASD` - 前进/后退、左转/右转（仅在 `WALK` 模式）
- `JL` - 向左/向右平移（仅在 `WALK` 模式）
- `Space/Q` - 紧急停止（切换到 `IDLE`）
- `ESC` - 退出 `pygame` 窗口
- `Ctrl+C` - 退出程序

#### 测试模式（无需机器人）

```bash
python -m dimos.robot.unitree.b1.unitree_b1     --test     --joystick
```

该模式不会发送 UDP 数据包，而是直接打印命令，适合开发使用。

## 安全特性

### 客户端
- **命令新鲜度跟踪**：若 100ms 内没有新命令，将停止发送
- **紧急停止**：按 `Q` 或 `Space` 会立即切换到 `IDLE`
- **模式安全**：仅允许在 `WALK` 模式下移动
- **优雅关闭**：退出时发送停止命令

### 服务端
- **数据包超时**：100ms 内没有数据包时机器人停止
- **持续监控**：每次控制更新前都会检查超时
- **安全默认值**：启动时进入 `IDLE` 模式
- **数据包校验**：拒绝格式错误的数据包

## 架构

```
External Machine (Client)          B1 Robot (Server)
┌─────────────────────┐           ┌──────────────────┐
│ Joystick Module     │           │                  │
│ (pygame input)      │           │ joystick_server  │
│         ↓           │           │    _udp.cpp      │
│    Twist msg        │           │                  │
│         ↓           │  WiFi AP  │                  │
│ B1ConnectionModule  │◄─────────►│ UDP Port 9090    │
│ (Twist → B1Command) │ 192.168.  │                  │
│         ↓           │   12.1    │                  │
│  UDP packets 50Hz   │           │ Unitree SDK      │
└─────────────────────┘           └──────────────────┘
```

## 使用 Unitree B1 配置 ROS 导航栈

### 在板载硬件上配置外置无线 USB 网卡
这样做的原因是：板载硬件（mini PC、Jetson 等）既需要连接到 B1 的 WiFi AP 网络，以通过 UDP 发送 `cmd_vel` 消息，也需要连接到运行 Dimensional 的网络。

插入无线网卡：
```bash
nmcli device status
nmcli device wifi list ifname *DEVICE_NAME*
# Connect to b1 network
nmcli device wifi connect "Unitree_B1-251" password "00000000" ifname *DEVICE_NAME*
# Verify connection
nmcli connection show --active
```

### *TODO: 补充更多文档*

## 故障排查

### 无法连接到 B1
- 确认已连接到 B1 的 AP
- 检查 IP：应为 `192.168.12.1`
- 确认服务端正在运行：`ssh unitree@192.168.12.1`

### 机器人无响应
- 确认服务端输出中出现 “Client connected”
- 检查机器人是否处于 `WALK` 模式（按 `2`）
- 确认服务端输出中没有超时消息

### 超时问题
- 检查网络延迟：`ping 192.168.12.1`
- 确保保持 50Hz 发送频率
- 查找 “Command timeout” 消息

### 紧急情况
- 按 `Space` 或 `Q` 立即停止
- 使用 `Ctrl+C` 干净退出
- 机器人会在 100ms 无命令时自动停止

## 开发说明

- 数据包长度为 19 字节：4 个 float + `uint16` + `uint8`
- 坐标系：B1 使用不同约定，因此 `b1_command.py` 中存在取反处理
- LCM 话题：`/cmd_vel` 用于 `Twist`，`/b1/mode` 用于 `Int32` 模式切换

## 许可证

Copyright 2025 Dimensional Inc. Licensed under Apache License 2.0.
