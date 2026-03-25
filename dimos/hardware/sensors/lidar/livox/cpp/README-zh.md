# Livox Mid-360 Native Module（C++ 原生模块）

Livox Mid-360 LiDAR 的原生 C++ 驱动。直接通过 LCM 发布 PointCloud2 和 IMU 数据，绕过 Python 以实现最低延迟。

## 构建

### Nix（推荐）

```bash
cd dimos/hardware/sensors/lidar/livox/cpp
nix build .#mid360_native
```

二进制文件位于 `result/bin/mid360_native`。

仅构建 Livox SDK2 库：

```bash
nix build .#livox-sdk2
```

### 原生构建（CMake）

依赖项：
- CMake >= 3.14
- [LCM](https://lcm-proj.github.io/)（`pacman -S lcm` 或从源码构建）
- [Livox SDK2](https://github.com/Livox-SDK/Livox-SDK2) 安装到 `/usr/local`

手动安装 Livox SDK2：

```bash
cd ~/src
git clone https://github.com/Livox-SDK/Livox-SDK2.git
cd Livox-SDK2 && mkdir build && cd build
cmake .. && make -j$(nproc)
sudo make install
```

然后构建：

```bash
cd dimos/hardware/sensors/lidar/livox/cpp
cmake -B build
cmake --build build -j$(nproc)
cmake --install build
```

二进制文件位于 `result/bin/mid360_native`（与 nix 相同位置）。

CMake 在首次配置时会自动获取 [dimos-lcm](https://github.com/dimensionalOS/dimos-lcm) 的 C++ 消息头文件。

## 网络设置

Mid-360 通过 USB 以太网通信。配置网络接口：

```bash
sudo nmcli con add type ethernet ifname usbeth0 con-name livox-mid360 \
    ipv4.addresses 192.168.1.5/24 ipv4.method manual
sudo nmcli con up livox-mid360
```

此配置在重启后仍然有效。雷达默认地址为 `192.168.1.155`。

## 使用方法

通常由 `Mid360` 通过 NativeModule 框架启动：

```python
from dimos.hardware.sensors.lidar.livox.module import Mid360
from dimos.core.blueprints import autoconnect

autoconnect(
    Mid360.blueprint(host_ip="192.168.1.5"),
    SomeConsumer.blueprint(),
).build().loop()
```

### 手动调用（用于调试）

```bash
./result/bin/mid360_native \
    --pointcloud '/pointcloud#sensor_msgs.PointCloud2' \
    --imu '/imu#sensor_msgs.Imu' \
    --host_ip 192.168.1.5 \
    --lidar_ip 192.168.1.155 \
    --frequency 10
```

Topic 字符串必须包含 `#type` 后缀——这是 dimos 订阅者使用的实际 LCM 通道名称。

在另一个终端中查看数据：

完整可视化：
```sh
rerun-bridge
```

查看 LCM 流量：
```sh
lcm-spy
```

## 文件概览

| 文件 | 描述 |
|---------------------------|----------------------------------------------------------|
| `main.cpp`                | Livox SDK2 回调、帧累积、LCM 发布 |
| `dimos_native_module.hpp` | 可复用头文件，用于解析 NativeModule CLI 参数 |
| `flake.nix`               | Nix flake，用于可重复构建 |
| `CMakeLists.txt`          | 构建配置，自动获取 dimos-lcm 头文件 |
| `../module.py`            | Python NativeModule 包装器（`Mid360`） |
