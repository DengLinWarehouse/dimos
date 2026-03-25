
# DimOS 模块

模块是机器人上的子系统，它们自主运行，并使用标准化消息与其他子系统通信。

一些模块的例子：

- 摄像头（输出图像）
- 导航（输入地图和目标，输出路径）
- 检测（接收图像和视觉模型如 YOLO，输出检测结果流）

以下是一个控制机器人的结构示例。黑色方块代表模块，彩色线条代表连接和消息类型。如果现在看不懂也没关系，看完本文档后你就会理解了。

```python output=assets/go2_nav.svg
from dimos.core.introspection import to_svg
from dimos.robot.unitree_webrtc.unitree_go2_blueprints import nav
to_svg(nav, "assets/go2_nav.svg")
```

<!--Result:-->
![output](assets/go2_nav.svg)

## 摄像头模块

让我们从一个简单的摄像头模块开始，学习如何构建上面类似的结构。

```python session=camera_module_demo output=assets/camera_module.svg
from dimos.hardware.sensors.camera.module import CameraModule
from dimos.core.introspection import to_svg
to_svg(CameraModule.module_info(), "assets/camera_module.svg")
```

<!--Result:-->
![output](assets/camera_module.svg)

我们也可以通过 `.io()` 调用快速将模块 I/O 打印到控制台。后续我们将使用这种方式。

```python session=camera_module_demo ansi=false
print(CameraModule.io())
```

<!--Result:-->
```
┌┴─────────────┐
│ CameraModule │
└┬─────────────┘
 ├─ color_image: Image
 ├─ camera_info: CameraInfo
 │
 ├─ RPC start()
 ├─ RPC stop()
 │
 ├─ Skill take_a_picture
```

我们可以看到摄像头模块输出两个流：

- `color_image`，类型为 [sensor_msgs.Image](https://docs.ros.org/en/melodic/api/sensor_msgs/html/msg/Image.html)
- `camera_info`，类型为 [sensor_msgs.CameraInfo](https://docs.ros.org/en/melodic/api/sensor_msgs/html/msg/CameraInfo.html)

它提供两个 RPC 调用：`start()` 和 `stop()`（生命周期方法）。

它还暴露了一个智能体[技能](/docs/usage/blueprints.md#defining-skills) `take_a_picture`（更多关于技能的内容请参阅蓝图指南）。

我们可以启动这个模块并实时查看其流的输出（这将使用你的摄像头）。

```python session=camera_module_demo ansi=false
import time

camera = CameraModule()
camera.start()
# 现在这个模块在主循环的线程中运行。我们可以观察其输出。

print(camera.color_image)

camera.color_image.subscribe(print)
time.sleep(0.5)
camera.stop()
```

<!--Result:-->
```
Out color_image[Image] @ CameraModule
Image(shape=(480, 640, 3), format=RGB, dtype=uint8, dev=cpu, ts=2025-12-31 15:54:16)
Image(shape=(480, 640, 3), format=RGB, dtype=uint8, dev=cpu, ts=2025-12-31 15:54:16)
Image(shape=(480, 640, 3), format=RGB, dtype=uint8, dev=cpu, ts=2025-12-31 15:54:17)
Image(shape=(480, 640, 3), format=RGB, dtype=uint8, dev=cpu, ts=2025-12-31 15:54:17)
Image(shape=(480, 640, 3), format=RGB, dtype=uint8, dev=cpu, ts=2025-12-31 15:54:17)
Image(shape=(480, 640, 3), format=RGB, dtype=uint8, dev=cpu, ts=2025-12-31 15:54:17)
Image(shape=(480, 640, 3), format=RGB, dtype=uint8, dev=cpu, ts=2025-12-31 15:54:17)
Image(shape=(480, 640, 3), format=RGB, dtype=uint8, dev=cpu, ts=2025-12-31 15:54:17)
Image(shape=(480, 640, 3), format=RGB, dtype=uint8, dev=cpu, ts=2025-12-31 15:54:17)
Image(shape=(480, 640, 3), format=RGB, dtype=uint8, dev=cpu, ts=2025-12-31 15:54:17)
```


## 连接模块

让我们加载一个标准的 2D 检测器模块并将其连接到摄像头。

```python ansi=false session=detection_module
from dimos.perception.detection.module2D import Detection2DModule, Config
print(Detection2DModule.io())
```

<!--Result:-->
```
 ├─ image: Image
┌┴──────────────────┐
│ Detection2DModule │
└┬──────────────────┘
 ├─ detections: Detection2DArray
 ├─ annotations: ImageAnnotations
 ├─ detected_image_0: Image
 ├─ detected_image_1: Image
 ├─ detected_image_2: Image
 │
 ├─ RPC set_transport(stream_name: str, transport: Transport) -> bool
 ├─ RPC start() -> None
 ├─ RPC stop() -> None
```

<!-- TODO: add easy way to print config -->

看起来检测器只需要一个图像输入，并输出某种检测和标注消息。让我们将它连接到摄像头。

```python ansi=false
import time
from dimos.perception.detection.module2D import Detection2DModule, Config
from dimos.hardware.sensors.camera.module import CameraModule

camera = CameraModule()
detector = Detection2DModule()

detector.image.connect(camera.color_image)

camera.start()
detector.start()

detector.detections.subscribe(print)
time.sleep(3)
detector.stop()
camera.stop()
```

<!--Result:-->
```
Detection(Person(1))
Detection(Person(1))
Detection(Person(1))
Detection(Person(1))
```

## 分布式执行

随着我们构建的模块结构越来越复杂，我们很快就需要利用机器上的所有核心（Python 单进程无法做到这一点），甚至可能需要将模块分布到多台机器或互联网上。

为此，我们使用 `dimos.core` 和 DimOS 传输协议。

定义消息交换协议和消息类型还使我们能够用更快的语言编写模块。

## 蓝图

蓝图是预定义的互连模块结构。你可以将蓝图或模块包含在新的蓝图中。

一个基本的 Unitree Go2 蓝图如我们之前所见。

```python  session=blueprints output=assets/go2_agentic.svg
from dimos.core.introspection import to_svg
from dimos.robot.unitree_webrtc.unitree_go2_blueprints import agentic

to_svg(agentic, "assets/go2_agentic.svg")
```

<!--Result:-->
![output](assets/go2_agentic.svg)


要了解更多关于蓝图使用方法的信息，请参阅[蓝图](/docs/usage/blueprints.md)。
