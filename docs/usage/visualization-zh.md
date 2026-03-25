# 可视化后端

Dimos 支持三种可视化后端：Rerun（Web 或原生桌面版）和 Foxglove。

## 快速入门

通过 CLI 选择查看器（推荐方式）：

```bash
# Rerun 原生查看器（默认）- dimos-viewer，内置遥操作 + 点击导航
dimos run unitree-go2

# 显式选择查看器模式：
dimos --viewer rerun run unitree-go2
dimos --viewer rerun-web run unitree-go2
dimos --viewer foxglove run unitree-go2
```

替代方式（环境变量）：

```bash
# Rerun 原生查看器（默认）- dimos-viewer，内置遥操作 + 点击导航
VIEWER=rerun dimos run unitree-go2

# Rerun Web 查看器 - 浏览器仪表盘 + 遥操作，访问 http://localhost:7779
VIEWER=rerun-web dimos run unitree-go2

# Foxglove - 使用 Foxglove Studio 替代 Rerun
VIEWER=foxglove dimos run unitree-go2
```

## 查看器模式说明

### Rerun 原生模式（`rerun`）— 默认

**你将获得：**
- [dimos-viewer](https://github.com/dimensionalOS/dimos-viewer)，Dimensional 定制版的 Rerun，内置键盘遥操作和点击导航功能
- 原生桌面应用程序（自动打开）
- 处理大型地图/高分辨率时性能更好
- 无需浏览器或 Web 服务器

---

### Rerun Web 模式（`rerun-web`）

**你将获得：**
- 基于浏览器的仪表盘，访问 http://localhost:7779
- Rerun 3D 查看器 + 命令中心侧栏合为一页
- 通过 Web UI 进行遥操作控制和目标设定
- 支持无头模式运行（无需显示器）

---

### Foxglove 模式（`foxglove`）

**你将获得：**
- Foxglove 桥接，位于 ws://localhost:8765
- 不使用 Rerun（节省资源）
- 处理大型地图/高分辨率时性能更好
- 打开布局文件：`assets/foxglove_dashboards/old/foxglove_unitree_lcm_dashboard.json`

---

## 在自定义蓝图中渲染

要在自己的蓝图中启用 rerun，只需包含 `RerunBridgeModule`：

```python
from dimos.visualization.rerun.bridge import RerunBridgeModule
from dimos.hardware.sensors.camera.module import CameraModule
from dimos.protocol.pubsub.impl.lcmpubsub import LCM

camera_demo = autoconnect(
    CameraModule.blueprint(),
    RerunBridgeModule.blueprint(
        viewer_mode="native", # native（桌面）、web（浏览器）、none（无头）
    ),
)

if __name__ == "__main__":
    camera_demo.build().loop()
```

每个 LCM 流（如 `CameraModule` 输出的 `color_image`）如果使用的数据类型（如 `Image`）有 `.to_rerun` 方法，就会使用 LCM topic 作为 rerun 实体路径进行渲染（`rr.log`）。换言之：要渲染某些内容，只需将其记录到流中，它就会自动在 rerun 中可用。

## 性能调优

### 症状：地图更新缓慢

如果你注意到：
- 机器人看起来在"空白区域行走"
- 代价地图更新落后于机器人
- 可视化卡顿或冻结

这通常发生在低端硬件（NUC、较旧的笔记本电脑）处理大型地图时。

### 增大体素尺寸

编辑 [`dimos/robot/unitree/go2/blueprints/__init__.py`](/dimos/robot/unitree/go2/blueprints/__init__.py) 第 82 行：

```python
# 修改前（高精度，大型地图时较慢）
voxel_mapper(voxel_size=0.05),  # 5cm 体素

# 修改后（低精度，快 8 倍）
voxel_mapper(voxel_size=0.1),   # 10cm 体素
```

**权衡：**
- 更大的体素 = 更少的体素数量 = 更快的更新
- 但地图细节会稍有降低

---

## 在 `dev` 上使用 Rerun（以及 TF/实体的注意事项）

`dev` 上的 Rerun 是**模块驱动的**：模块决定记录什么内容，`Blueprint.build()` 设置共享查看器和默认布局。
