# 坐标变换

## 问题：每个设备都从自己的视角测量

假设你的机器人配备了一个 RGB-D 相机——一种同时捕捉彩色图像和深度（每个像素的距离）的相机。这在机器人领域很常见：Intel RealSense、Microsoft Kinect 及类似传感器。

相机在像素 (320, 240) 处发现了一个咖啡杯，深度传感器显示距离为 1.2 米。你想让机器人手臂去抓取它——但手臂不理解像素或相机相对距离。它需要在自己的工作空间中的坐标："移动到距离我基座 (0.8, 0.3, 0.1) 米的位置。"

要将相机测量值转换为手臂坐标，你需要知道：
- 相机的内参（焦距、传感器尺寸）以将像素转换为 3D 方向
- 深度值以获得相对于相机的完整 3D 位置
- 相机相对于手臂的安装位置和角度

这个转换链——（像素 + 深度）→ 相机坐标系中的 3D 点 → 机器人坐标——就是**坐标变换（transforms）**所处理的。

<details>
<summary>图表源码</summary>

```pikchr fold output=assets/transforms_tree.svg
color = white
fill = none

# Root (left side)
W: box "world" rad 5px fit wid 170% ht 170%
arrow right 0.4in
RB: box "robot_base" rad 5px fit wid 170% ht 170%

# Camera branch (top)
arrow from RB.e right 0.3in then up 0.4in then right 0.3in
CL: box "camera_link" rad 5px fit wid 170% ht 170%
arrow right 0.4in
CO: box "camera_optical" rad 5px fit wid 170% ht 170%
text "mug here" small italic at (CO.s.x, CO.s.y - 0.25in)

# Arm branch (bottom)
arrow from RB.e right 0.3in then down 0.4in then right 0.3in
AB: box "arm_base" rad 5px fit wid 170% ht 170%
arrow right 0.4in
GR: box "gripper" rad 5px fit wid 170% ht 170%
text "target here" small italic at (GR.s.x, GR.s.y - 0.25in)
```

</details>

<!--Result:-->
![output](assets/transforms_tree.svg)


树中的每个箭头都是一个变换。要获取杯子在夹爪坐标系中的位置，需要通过共同父节点链接变换：camera → robot_base → arm → gripper。

## 什么是坐标系？

**坐标系（coordinate frame）**简单来说就是一个观察视角——一个原点和一组坐标轴（X、Y、Z），用于测量位置和方向。

可以类比给人指路：
- **GPS** 显示你在北纬 37.7749°，西经 122.4194°
- **咖啡店平面图**显示"5 号桌在入口前方 3 米处"
- 你的**朋友**说"我在你左边两桌"

这些都描述了同一物理空间中的位置，但参考点不同。每个都是一个坐标系。

在机器人中：
- **相机**以像素或相对于镜头的米为单位测量
- **LiDAR** 从自身安装点测量距离
- **机器人手臂**以基座或末端执行器位置为参考
- **世界**有一个固定的坐标系，所有物体都在其中

每个传感器、关节和参考点都有自己的坐标系。

## Transform 类

`Transform` 类位于 [`geometry_msgs/Transform.py`](/dimos/msgs/geometry_msgs/Transform.py#L21)，表示一个空间变换，包含：

- `frame_id` - 父坐标系名称
- `child_frame_id` - 子坐标系名称
- `translation` - `Vector3` (x, y, z) 平移偏移
- `rotation` - `Quaternion` (x, y, z, w) 旋转方向
- `ts` - 用于时间查询的时间戳

```python
from dimos.msgs.geometry_msgs import Transform, Vector3, Quaternion

# 相机在基座前方 0.5m、上方 0.3m，无旋转
camera_transform = Transform(
    translation=Vector3(0.5, 0.0, 0.3),
    rotation=Quaternion(0.0, 0.0, 0.0, 1.0),  # Identity rotation
    frame_id="base_link",
    child_frame_id="camera_link",
)
print(camera_transform)
```

<!--Result:-->
```
base_link -> camera_link
  Translation: → Vector Vector([0.5 0.  0.3])
  Rotation: Quaternion(0.000000, 0.000000, 0.000000, 1.000000)
```


### 变换运算

变换可以组合和求逆：

```python
from dimos.msgs.geometry_msgs import Transform, Vector3, Quaternion

# 创建两个变换
t1 = Transform(
    translation=Vector3(1.0, 0.0, 0.0),
    rotation=Quaternion(0.0, 0.0, 0.0, 1.0),
    frame_id="base_link",
    child_frame_id="camera_link",
)
t2 = Transform(
    translation=Vector3(0.0, 0.5, 0.0),
    rotation=Quaternion(0.0, 0.0, 0.0, 1.0),
    frame_id="camera_link",
    child_frame_id="end_effector",
)

# 组合：base_link -> camera -> end_effector
t3 = t1 + t2
print(f"Composed: {t3.frame_id} -> {t3.child_frame_id}")
print(f"Translation: ({t3.translation.x}, {t3.translation.y}, {t3.translation.z})")

# 求逆：如果 t 从 A -> B，则 -t 从 B -> A
t_inverse = -t1
print(f"Inverse: {t_inverse.frame_id} -> {t_inverse.child_frame_id}")
```

<!--Result:-->
```
Composed: base_link -> end_effector
Translation: (1.0, 0.5, 0.0)
Inverse: camera_link -> base_link
```


### 转换为矩阵形式

用于与 NumPy 或 OpenCV 等库集成：

```python
from dimos.msgs.geometry_msgs import Transform, Vector3, Quaternion

t = Transform(
    translation=Vector3(1.0, 2.0, 3.0),
    rotation=Quaternion(0.0, 0.0, 0.0, 1.0),
)
matrix = t.to_matrix()
print("4x4 transformation matrix:")
print(matrix)
```

<!--Result:-->
```
4x4 transformation matrix:
[[1. 0. 0. 1.]
 [0. 1. 0. 2.]
 [0. 0. 1. 3.]
 [0. 0. 0. 1.]]
```



## 模块中的 Frame ID

DimOS 中的模块会自动获得 `frame_id` 属性。这由 [`core/module.py`](/dimos/core/module.py#L78) 中的两个配置选项控制：

- `frame_id` - 基础坐标系名称（默认为类名）
- `frame_id_prefix` - 可选的命名空间前缀

```python
from dimos.core.module import Module, ModuleConfig
from dataclasses import dataclass

@dataclass
class MyModuleConfig(ModuleConfig):
    frame_id: str = "sensor_link"
    frame_id_prefix: str | None = None

class MySensorModule(Module[MyModuleConfig]):
    default_config = MyModuleConfig

# 使用默认配置：
sensor = MySensorModule()
print(f"Default frame_id: {sensor.frame_id}")

# 使用前缀（多机器人场景下很有用）：
sensor2 = MySensorModule(frame_id_prefix="robot1")
print(f"With prefix: {sensor2.frame_id}")
```

<!--Result:-->
```
Default frame_id: sensor_link
With prefix: robot1/sensor_link
```


## TF 服务

每个模块都可以通过 `self.tf` 访问坐标变换服务，它能够：

- **发布**变换到系统
- **查询**任意两个坐标系之间的变换
- **缓存**历史变换用于时间查询

TF 服务实现在 [`tf.py`](/dimos/protocol/tf/tf.py) 中，在首次访问时延迟初始化。

### 多模块变换示例

此示例展示了多个模块如何发布和接收变换。三个模块协同工作：

1. **RobotBaseModule** - 发布 `world -> base_link`（机器人在世界坐标系中的位置）
2. **CameraModule** - 发布 `base_link -> camera_link`（相机安装位置）和 `camera_link -> camera_optical`（光学坐标系约定）
3. **PerceptionModule** - 查询任意坐标系之间的变换

```python ansi=false
import time
import reactivex as rx
from reactivex import operators as ops
from dimos.core.core import rpc
from dimos.core.module import Module
from dimos.msgs.geometry_msgs import Quaternion, Transform, Vector3
from dimos.core.module_coordinator import ModuleCoordinator

class RobotBaseModule(Module):
    """以 10Hz 频率发布机器人在世界坐标系中的位置。"""
    def __init__(self, **kwargs: object) -> None:
        super().__init__(**kwargs)

    @rpc
    def start(self) -> None:
        super().start()

        def publish_pose(_):
            robot_pose = Transform(
                translation=Vector3(2.5, 3.0, 0.0),
                rotation=Quaternion(0.0, 0.0, 0.0, 1.0),
                frame_id="world",
                child_frame_id="base_link",
                ts=time.time(),
            )
            self.tf.publish(robot_pose)

        self._disposables.add(
            rx.interval(0.1).subscribe(publish_pose)
        )

class CameraModule(Module):
    """以 10Hz 频率发布相机变换。"""
    @rpc
    def start(self) -> None:
        super().start()

        def publish_transforms(_):
            camera_mount = Transform(
                translation=Vector3(1.0, 0.0, 0.3),
                rotation=Quaternion(0.0, 0.0, 0.0, 1.0),
                frame_id="base_link",
                child_frame_id="camera_link",
                ts=time.time(),
            )
            optical_frame = Transform(
                translation=Vector3(0.0, 0.0, 0.0),
                rotation=Quaternion(-0.5, 0.5, -0.5, 0.5),
                frame_id="camera_link",
                child_frame_id="camera_optical",
                ts=time.time(),
            )
            self.tf.publish(camera_mount, optical_frame)

        self._disposables.add(
            rx.interval(0.1).subscribe(publish_transforms)
        )


class PerceptionModule(Module):
    """接收变换并执行查询。"""

    def start(self) -> None:
        # 这只是初始化变换系统。
        # 首次访问该属性会为此模块启用变换系统。
        # 变换查询通常在实际模块的快速循环中发生。
        _ = self.tf

    @rpc
    def lookup(self) -> None:

        # 打印缓冲区中的变换信息
        print(self.tf)

        direct = self.tf.get("world", "base_link")
        print(f"Direct: robot is at ({direct.translation.x}, {direct.translation.y})m in world\n")

        # 链式查询 - 自动组合 world -> base -> camera -> optical
        chained = self.tf.get("world", "camera_optical")
        print(f"Chained: {chained}\n")

        # 逆向查询 - 自动反转方向
        inverse = self.tf.get("camera_optical", "world")
        print(f"Inverse: {inverse}\n")

        print("Transform tree:")
        print(self.tf.graph())


if __name__ == "__main__":
    dimos = ModuleCoordinator()
    dimos.start()

    robot = dimos.deploy(RobotBaseModule)
    camera = dimos.deploy(CameraModule)
    perception = dimos.deploy(PerceptionModule)

    dimos.start_all_modules()

    time.sleep(1.0)

    perception.lookup()

    dimos.stop()

```

<!--Result:-->
```
Initialized dimos local cluster with 3 workers, memory limit: auto
2025-12-29T12:47:01.433394Z [info     ] Deployed module.                                             [dimos/core/__init__.py] module=RobotBaseModule worker_id=1
2025-12-29T12:47:01.603269Z [info     ] Deployed module.                                             [dimos/core/__init__.py] module=CameraModule worker_id=0
2025-12-29T12:47:01.698970Z [info     ] Deployed module.                                             [dimos/core/__init__.py] module=PerceptionModule worker_id=2
LCMTF(3 buffers):
  TBuffer(world -> base_link, 10 msgs, 0.90s [2025-12-29 20:47:01 - 2025-12-29 20:47:02])
  TBuffer(base_link -> camera_link, 9 msgs, 0.80s [2025-12-29 20:47:01 - 2025-12-29 20:47:02])
  TBuffer(camera_link -> camera_optical, 9 msgs, 0.80s [2025-12-29 20:47:01 - 2025-12-29 20:47:02])
Direct: robot is at (2.5, 3.0)m in world

Chained: world -> camera_optical
  Translation: → Vector Vector([3.5 3.  0.3])
  Rotation: Quaternion(-0.500000, 0.500000, -0.500000, 0.500000)

Inverse: camera_optical -> world
  Translation: → Vector Vector([ 3.   0.3 -3.5])
  Rotation: Quaternion(0.500000, -0.500000, 0.500000, 0.500000)

Transform tree:
┌─────┐
│world│
└┬────┘
┌▽────────┐
│base_link│
└┬────────┘
┌▽──────────┐
│camera_link│
└┬──────────┘
┌▽─────────────┐
│camera_optical│
└──────────────┘
```


你也可以在下一个终端中运行 `foxglove-studio-bridge`（由 DimOS 提供的二进制文件，应在你的 Python 环境中）和 `foxglove-studio` 来以 3D 方式查看这些变换。（TODO：我们需要为 rerun 更新此内容）

![transforms](assets/transforms.png)

关键要点：

- **自动广播**：`self.tf.publish()` 通过 LCM 向所有模块广播
- **链式查询**：TF 自动找到坐标系树中的路径
- **逆向查询**：可以在任意方向请求变换
- **时间缓冲**：变换带有时间戳并被缓冲（默认 10 秒），用于传感器融合

上面示例的变换树，展示了每个变换由哪个模块发布：

<details>
<summary>图表源码</summary>

```pikchr fold output=assets/transforms_modules.svg
color = white
fill = none

# Frame boxes
W: box "world" rad 5px fit wid 170% ht 170%
A1: arrow right 0.4in
BL: box "base_link" rad 5px fit wid 170% ht 170%
A2: arrow right 0.4in
CL: box "camera_link" rad 5px fit wid 170% ht 170%
A3: arrow right 0.4in
CO: box "camera_optical" rad 5px fit wid 170% ht 170%

# RobotBaseModule box - encompasses world->base_link
box width (BL.e.x - W.w.x + 0.15in) height 0.7in \
    at ((W.w.x + BL.e.x)/2, W.y - 0.05in) \
    rad 10px color 0x6699cc fill none
text "RobotBaseModule" italic at ((W.x + BL.x)/2, W.n.y + 0.25in)

# CameraModule box - encompasses camera_link->camera_optical (starts after base_link)
box width (CO.e.x - BL.e.x + 0.1in) height 0.7in \
    at ((BL.e.x + CO.e.x)/2, CL.y + 0.05in) \
    rad 10px color 0xcc9966 fill none
text "CameraModule" italic at ((CL.x + CO.x)/2, CL.s.y - 0.25in)
```


</details>

<!--Result:-->
![output](assets/transforms_modules.svg)


# 内部机制

## 变换缓冲区

模块上的 `self.tf` 是一个变换缓冲区。这是一个独立的类，维护变换的时间缓冲（默认 10 秒），允许查询过去时间戳的变换，你可以直接使用它：

```python
from dimos.protocol.tf import TF
from dimos.msgs.geometry_msgs import Transform, Vector3, Quaternion
import time

tf = TF(autostart=False)

# 在不同时间模拟变换
for i in range(5):
    t = Transform(
        translation=Vector3(float(i), 0.0, 0.0),
        rotation=Quaternion(0.0, 0.0, 0.0, 1.0),
        frame_id="base_link",
        child_frame_id="camera_link",
        ts=time.time() + i * 0.1,
    )
    tf.receive_transform(t)

# 查询最新变换
result = tf.get("base_link", "camera_link")
print(f"Latest transform: x={result.translation.x}")
print(f"Buffer has {len(tf.buffers)} transform pair(s)")
print(tf)
```

<!--Result:-->
```
Latest transform: x=4.0
Buffer has 1 transform pair(s)
LCMTF(1 buffers):
  TBuffer(base_link -> camera_link, 5 msgs, 0.40s [2025-12-29 18:19:18 - 2025-12-29 18:19:18])
```


这对于传感器融合至关重要——你需要知道图像被捕捉时相机的位置，而不是当前位置。


## 延伸阅读

关于坐标变换和坐标系的可视化介绍：
- [Coordinate Transforms (YouTube)](https://www.youtube.com/watch?v=NGPn9nvLPmg)

关于数学基础，ROS 文档提供了详细的背景知识：

- [ROS tf2 概念](http://wiki.ros.org/tf2)
- [ROS REP 103 - 标准单位和坐标约定](https://www.ros.org/reps/rep-0103.html)
- [ROS REP 105 - 移动平台坐标系](https://www.ros.org/reps/rep-0105.html)

另请参阅：
- [模块](/docs/usage/modules.md) - 了解模块系统
- [配置](/docs/usage/configuration.md) - 模块配置模式
