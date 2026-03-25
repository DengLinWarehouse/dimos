# DimOS 的 ROS Docker 集成

本目录包含 Docker 配置文件，用于在同一容器中运行 DimOS 与 ROS 自主导航栈，使两套系统能够互通。

## 前置条件

1. **安装支持 `docker compose` 的 Docker（容器平台）**。请参阅 [Docker 官方安装指南](https://docs.docker.com/engine/install/)。
2. **安装 NVIDIA GPU 驱动**。请参阅 [NVIDIA 驱动安装](https://www.nvidia.com/download/index.aspx)。
3. **安装 NVIDIA Container Toolkit（容器工具包）**。请参阅 [安装指南](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)。

## 自动化快速开始

以下为乐观概览；完整步骤请使用下文命令。

**构建 Docker 镜像：**

```bash
cd docker/navigation
./build.sh --humble    # 为 ROS 2 Humble 构建
./build.sh --jazzy     # 为 ROS 2 Jazzy 构建
```

将执行：
- 克隆 ros-navigation-autonomy-stack 仓库
- 构建同时包含 arise_slam 与 FASTLIO2 的 Docker 镜像
- 为 ROS 与 DimOS 配置运行环境

生成的镜像名为 `dimos_autonomy_stack:{distro}`（例如 `humble`、`jazzy`）。  
运行时通过 `--localization arise_slam` 或 `--localization fastlio` 选择 SLAM 方式。

注意：构建耗时较长，镜像体积约 24 GB。

**运行仿真以验证可用：**

请与构建时使用相同的 ROS 发行版标志：

```bash
./start.sh --simulation --image humble  # 若使用 --humble 构建
# 或
./start.sh --simulation --image jazzy   # 若使用 --jazzy 构建
```

<details>
<summary><h2>手动构建</h2></summary>

进入 docker 目录并克隆 ROS 导航栈（分支需与 ROS 发行版一致）。

```bash
cd docker/navigation
git clone -b humble git@github.com:dimensionalOS/ros-navigation-autonomy-stack.git
# 或
git clone -b jazzy git@github.com:dimensionalOS/ros-navigation-autonomy-stack.git
```

下载 [麦克纳姆轮平台的 Unity 环境模型](https://drive.google.com/drive/folders/1G1JYkccvoSlxyySuTlPfvmrWoJUO8oSs?usp=sharing)，并解压到 `unity_models`。

或者从 LFS 解压 `office_building_1`：

```bash
tar -xf ../../data/.lfs/office_building_1.tar.gz
mv office_building_1 unity_models
```

然后返回仓库根目录（从 docker/navigation）并构建镜像：

```bash
cd ../..  # 回到 dimos 根目录
ROS_DISTRO=humble docker compose -f docker/navigation/docker-compose.yml build
# 或
ROS_DISTRO=jazzy docker compose -f docker/navigation/docker-compose.yml build
```

</details>

## 真实硬件

### 配置 WiFi

请[阅读此文](https://github.com/dimensionalOS/ros-navigation-autonomy-stack/tree/jazzy?tab=readme-ov-file#transmitting-data-over-wifi)了解如何通过 WiFi 传输数据及相关配置。

### 配置 Livox 激光雷达

容器启动时会根据环境变量（LIDAR_COMPUTER_IP 与 LIDAR_IP）自动生成 MID360_config.json 文件。

### 复制环境模板
```bash
cp .env.hardware .env
```

### 编辑 `.env` 文件

主要配置参数：

```bash
# 机器人配置
ROBOT_CONFIG_PATH=unitree/unitree_go2  # 机器人类型（mechanum_drive、unitree/unitree_go2、unitree/unitree_g1）

# 激光雷达配置
LIDAR_INTERFACE=eth0              # 以太网接口（可用 ip link show 查看）
LIDAR_COMPUTER_IP=192.168.1.5    # 雷达子网内本机 IP
LIDAR_GATEWAY=192.168.1.1        # 雷达子网网关 IP
LIDAR_IP=192.168.1.1xx           # xx = 雷达二维码序列号末两位
ROBOT_IP=                        # 本地网络中机器人 IP（若使用 WebRTC 连接）

# Unitree G1 EDU 专用配置
# Unitree G1 EDU 专用配置
LIDAR_COMPUTER_IP=192.168.123.5
LIDAR_GATEWAY=192.168.123.1
LIDAR_IP=192.168.123.120
ROBOT_IP=192.168.12.1  # WebRTC 本地 AP 模式（可选，需额外 WiFi 网卡）
```

### 启动导航栈

#### 自动启动并带路径规划器（Route Planner）

```bash
# arise_slam（默认）
./start.sh --hardware --route-planner
./start.sh --hardware --route-planner --rviz

# FASTLIO2
./start.sh --hardware --localization fastlio --route-planner
./start.sh --hardware --localization fastlio --route-planner --rviz

# Jazzy 镜像
./start.sh --hardware --image jazzy --route-planner

# 开发模式（挂载 src 便于改配置）
./start.sh --hardware --dev
```

[Foxglove Studio](https://foxglove.dev/download) 为默认可视化工具，适合远程操作：通过 SSH 将端口转发到机器人 mini PC，并在该机上执行命令：

```bash
ssh -L 8765:localhost:8765 user@robot-ip
```

随后在本地机器上：
1. 打开 Foxglove，连接到 `ws://localhost:8765`
2. 从 `dimos/assets/foxglove_dashboards/Overwatch.json` 导入布局（Layout 菜单 → Import）
3. 在 3D 面板中点击放置目标位姿（与 RViz 类似）。“Autonomy ON” 指示灯应为绿色，到达目标后会显示 “Goal Reached”。

<details>
<summary><h4>手动启动</h4></summary>

启动容器并保持运行。请与构建时使用相同的 ROS 发行版标志：

```bash
./start.sh --hardware --image humble  # 若使用 --humble 构建
# 或
./start.sh --hardware --image jazzy   # 若使用 --jazzy 构建
```

默认不会自动跑任何进程，需通过 `exec` 在容器内执行命令。

在另一个终端进入容器：

```bash
docker exec -it dimos_hardware_container bash
```

##### 在容器内

要运行完整导航栈，需同时运行带 connection 模块的 dimensional Python runfile 与导航栈。

###### Dimensional Python + Connection 模块

Unitree G1：
```bash
dimos run unitree-g1
ROBOT_IP=XX.X.X.XXX dimos run unitree-g1 # 若 .env 中未设置 ROBOT_IP
```

###### 导航栈

```bash
cd /ros2_ws/src/ros-navigation-autonomy-stack
./system_real_robot_with_route_planner.sh
```

此时可在 RVIZ 中点击 “Goalpoint” 放置目标点/位姿。机器人将导航至该点，并运行局部与全局规划器以进行动态避障。

</details>
