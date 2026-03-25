# FAST-LIO2 Native Module（C++ 原生模块）

使用 FAST-LIO2 进行实时 LiDAR SLAM，集成 Livox Mid-360 驱动。
将 Livox SDK2 直接绑定到 FAST-LIO-NON-ROS：SDK 回调将
CustomMsg/Imu 馈送到 FastLio，执行 EKF-LOAM SLAM。配准后的
世界坐标系点云和里程计通过 LCM 发布。

## 构建

### Nix（推荐）

```bash
cd dimos/hardware/sensors/lidar/fastlio2/cpp
nix build .#fastlio2_native
```

二进制文件位于 `result/bin/fastlio2_native`。

该 flake 会自动从 livox 子 flake 拉取 Livox SDK2，并从 GitHub 拉取
[FAST-LIO-NON-ROS](https://github.com/leshy/FAST-LIO-NON-ROS)。

### 原生构建（CMake）

依赖项：
- CMake >= 3.14
- [LCM](https://lcm-proj.github.io/)（`pacman -S lcm` 或从源码构建）
- [Livox SDK2](https://github.com/Livox-SDK/Livox-SDK2) 安装到 `/usr/local`
- Eigen3、PCL（common、filters）、yaml-cpp、Boost、OpenMP
- [FAST-LIO-NON-ROS](https://github.com/leshy/FAST-LIO-NON-ROS) 本地检出

```bash
cd dimos/hardware/sensors/lidar/fastlio2/cpp
cmake -B build -DFASTLIO_DIR=$HOME/coding/FAST-LIO-NON-ROS
cmake --build build -j$(nproc)
cmake --install build
```

二进制文件位于 `result/bin/fastlio2_native`（与 nix 相同位置）。

如果省略 `-DFASTLIO_DIR`，CMake 会自动从 GitHub 获取 FAST-LIO-NON-ROS。

## 网络设置

Mid-360 通过 USB 以太网通信。配置网络接口：

```bash
sudo nmcli con add type ethernet ifname usbeth0 con-name livox-mid360 \
    ipv4.addresses 192.168.1.5/24 ipv4.method manual
sudo nmcli con up livox-mid360
```

此配置在重启后仍然有效。雷达默认地址为 `192.168.1.155`。

## 使用方法

通常由 `FastLio2` 通过 NativeModule 框架启动：

```python
from dimos.hardware.sensors.lidar.fastlio2.module import FastLio2
from dimos.core.blueprints import autoconnect

autoconnect(
    FastLio2.blueprint(host_ip="192.168.1.5"),
    SomeConsumer.blueprint(),
).build().loop()
```

### 手动调用（用于调试）

```bash
./result/bin/fastlio2_native \
    --lidar '/pointcloud#sensor_msgs.PointCloud2' \
    --odometry '/odometry#nav_msgs.Odometry' \
    --host_ip 192.168.1.5 \
    --lidar_ip 192.168.1.155 \
    --config_path ../config/mid360.yaml
```

Topic 字符串必须包含 `#type` 后缀——这是 dimos 订阅者使用的实际 LCM 通道名称。

完整可视化：
```sh
rerun-bridge
```

查看 LCM 流量：
```sh
lcm-spy
```

## 配置

FAST-LIO2 配置文件位于 `config/` 目录。YAML 配置文件控制滤波器参数、EKF 调参和点云处理设置。

## 文件概览

| 文件 | 描述 |
|---------------------------|--------------------------------------------------------------|
| `main.cpp`                | Livox SDK2 + FAST-LIO2 集成、EKF SLAM、LCM 发布 |
| `cloud_filter.hpp`        | 点云滤波（距离滤波、体素降采样） |
| `voxel_map.hpp`           | 全局体素地图累积 |
| `dimos_native_module.hpp` | 可复用头文件，用于解析 NativeModule CLI 参数 |
| `config/`                 | FAST-LIO2 YAML 配置文件 |
| `flake.nix`               | Nix flake，用于可重复构建 |
| `CMakeLists.txt`          | 构建配置，自动获取 dimos-lcm 头文件 |
| `../module.py`            | Python NativeModule 包装器（`FastLio2`） |
