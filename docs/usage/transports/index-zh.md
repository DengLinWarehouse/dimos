# 传输层

传输层将**模块流**跨**进程边界**和/或**网络**进行连接。

* **模块（Module）**：一个运行中的组件（如相机、建图、导航）。
* **流（Stream）**：模块拥有的单向消息流（一个广播者 → 多个接收者）。
* **Topic**：传输层或发布/订阅后端使用的名称/标识符。
* **消息（Message）**：流上承载的数据负载（通常是 `dimos.msgs.*`，但也可以是字节/图像/点云等）。

图中的每条边是一个**传输流**（可能使用不同协议）。每个节点是一个**模块**：

![go2_nav](../assets/go2_nav.svg)

## 传输层保证什么（以及不保证什么）

模块**不需要**知道或关心数据*如何*传输。它们只需：

* 发送消息（广播）
* 订阅消息（接收）

传输层负责传递的具体机制（IPC、套接字、Redis、ROS 2 等）。

**重要提示：**传递语义取决于后端：

* 有些是**尽力而为**的（如 UDP 组播 / LCM）：可能发生丢包。
* 有些可以做到**可靠传递**（如基于 TCP、Redis、某些 DDS 配置）但可能增加延迟/背压。

因此：API 的使用方式是统一的，但需要根据任务选择语义匹配的后端。

---

## 基准测试

我们发布/订阅后端的性能概览：

```sh skip
python -m pytest -svm tool -k "not bytes" dimos/protocol/pubsub/benchmark/test_benchmark.py
```

![Benchmark results](../assets/pubsub_benchmark.png)

---

## 抽象层

<details><summary>Pikchr</summary>

```pikchr output=../assets/abstraction_layers.svg fold
color = white
fill = none
linewid = 0.5in
boxwid = 1.0in
boxht = 0.4in

# Boxes with labels
B: box "Blueprints" rad 10px
arrow
M: box "Modules" rad 5px
arrow
T: box "Transports" rad 5px
arrow
P: box "PubSub" rad 5px

# Descriptions below
text "robot configs" at B.s + (0.1, -0.2in)
text "camera, nav" at M.s + (0, -0.2in)
text "LCM, SHM, ROS" at T.s + (0, -0.2in)
text "pub/sub API" at P.s + (0, -0.2in)
```

</details>

<!--Result:-->
![output](../assets/abstraction_layers.svg)

我们将自顶向下介绍这些层。

---

## 在蓝图中使用传输层

参见[蓝图](/docs/usage/blueprints.md)了解蓝图 API。

来自 [`unitree/go2/blueprints/__init__.py`](/dimos/robot/unitree/go2/blueprints/__init__.py)。

示例：将几个流从默认的 `LCMTransport` 重新绑定到 `ROSTransport`（定义在 [`transport.py`](/dimos/core/transport.py#L226)），以便在 **rviz2** 中可视化。

```python skip
nav = autoconnect(
    basic,
    voxel_mapper(voxel_size=0.1),
    cost_mapper(),
    replanning_a_star_planner(),
    wavefront_frontier_explorer(),
).global_config(n_workers=6, robot_model="unitree_go2")

ros = nav.transports(
    {
        ("lidar", PointCloud2): ROSTransport("lidar", PointCloud2),
        ("global_map", PointCloud2): ROSTransport("global_map", PointCloud2),
        ("odom", PoseStamped): ROSTransport("odom", PoseStamped),
        ("color_image", Image): ROSTransport("color_image", Image),
    }
)
```

---

## 在模块中使用传输层

模块上的每个**流**都可以使用不同的传输层。在**启动模块之前**设置流的 `.transport`。

```python ansi=false
import time

from dimos.core.module import Module
from dimos.core.stream import In
from dimos.core.transport import LCMTransport
from dimos.hardware.sensors.camera.module import CameraModule
from dimos.msgs.sensor_msgs import Image
from dimos.core.module_coordinator import ModuleCoordinator


class ImageListener(Module):
    image: In[Image]

    def start(self):
        super().start()
        self.image.subscribe(lambda img: print(f"Received: {img.shape}"))


if __name__ == "__main__":
    # 启动本地集群并将模块部署到独立进程
    dimos = ModuleCoordinator()
    dimos.start()

    camera = dimos.deploy(CameraModule, frequency=2.0)
    listener = dimos.deploy(ImageListener)

    # 选择流的传输层（示例：LCM 类型化通道）
    camera.color_image.transport = LCMTransport("/camera/rgb", Image)

    # 将监听器输入连接到相机输出
    listener.image.connect(camera.color_image)

    dimos.start_all_modules()

    time.sleep(2)
    dimos.stop()
```

<!--Result:-->

```
Initialized dimos local cluster with 2 workers, memory limit: auto
2026-01-24T13:17:50.190559Z [info     ] Deploying module.                                            [dimos/core/__init__.py] module=CameraModule
2026-01-24T13:17:50.218466Z [info     ] Deployed module.                                             [dimos/core/__init__.py] module=CameraModule worker_id=1
2026-01-24T13:17:50.229474Z [info     ] Deploying module.                                            [dimos/core/__init__.py] module=ImageListener
2026-01-24T13:17:50.250199Z [info     ] Deployed module.                                             [dimos/core/__init__.py] module=ImageListener worker_id=0
Received: (480, 640, 3)
Received: (480, 640, 3)
Received: (480, 640, 3)
```

参见[模块](/docs/usage/modules.md)了解更多模块架构信息。

---

## 检查 LCM 流量（CLI）

`lcmspy` 显示 topic 频率/带宽统计：

![lcmspy](../assets/lcmspy.png)

`dimos topic echo /topic` 监听类型化通道如 `/topic#pkg.Msg` 并自动解码：

```sh skip
Listening on /camera/rgb (inferring from typed LCM channels like '/camera/rgb#pkg.Msg')... (Ctrl+C to stop)
Image(shape=(480, 640, 3), format=RGB, dtype=uint8, dev=cpu, ts=2026-01-24 20:28:59)
```

---

## 实现传输层

在流层面，传输层通过继承 `Transport`（参见 [`core/stream.py`](/dimos/core/stream.py#L83)）并实现以下方法来实现：

* `broadcast(...)`
* `subscribe(...)`

`Transport.__init__` 的参数可以是任何对你的后端有意义的内容：

* `(ip, port)`
* 共享内存段名称
* 文件系统路径
* Redis 通道

编码是实现细节，但我们建议尽可能使用 LCM 兼容的消息类型。

### 编码辅助工具

许多消息类型提供 `lcm_encode` / `lcm_decode` 用于紧凑的、语言无关的二进制编码（通常比 pickle 更快）。详情参见 [LCM](/docs/usage/lcm.md)。

---

## PubSub 传输层

虽然传输层可以是任何形式（TCP 连接、Unix 套接字），但目前我们所有的传输后端都实现了 `PubSub` 接口。

* `publish(topic, message)`
* `subscribe(topic, callback) -> unsubscribe`

```python
from dimos.protocol.pubsub.spec import PubSub
import inspect

print(inspect.getsource(PubSub.publish))
print(inspect.getsource(PubSub.subscribe))
```

<!--Result:-->
```python
    @abstractmethod
    def publish(self, topic: TopicT, message: MsgT) -> None:
        """Publish a message to a topic."""
        ...

    @abstractmethod
    def subscribe(
        self, topic: TopicT, callback: Callable[[MsgT, TopicT], None]
    ) -> Callable[[], None]:
        """Subscribe to a topic with a callback. returns unsubscribe function"""
        ...
```

Topic/消息类型灵活多样：字节、JSON 或我们兼容 ROS 的 [LCM](/docs/usage/lcm.md) 类型。我们还提供基于 pickle 的传输层用于任意 Python 对象。

### LCM（UDP 组播）

LCM 使用 UDP 组播。在机器人局域网上非常快，但它是**尽力而为**的（可能丢包）。
在本地发送时，它会自动配置系统，使其比 ROS、DDS 等更常见的协议更健壮、更快。

```python
from dimos.protocol.pubsub.lcmpubsub import LCM, Topic
from dimos.msgs.geometry_msgs import Vector3

lcm = LCM()
lcm.start()

received = []
topic = Topic("/robot/velocity", Vector3)

lcm.subscribe(topic, lambda msg, t: received.append(msg))
lcm.publish(topic, Vector3(1.0, 0.0, 0.5))

import time
time.sleep(0.1)

print(f"Received velocity: x={received[0].x}, y={received[0].y}, z={received[0].z}")
lcm.stop()
```

<!--Result:-->
```
Received velocity: x=1.0, y=0.0, z=0.5
```

### 共享内存（IPC）

共享内存性能最高，但仅适用于**同一台机器**。

```python
from dimos.protocol.pubsub.shmpubsub import PickleSharedMemory

shm = PickleSharedMemory(prefer="cpu")
shm.start()

received = []
shm.subscribe("test/topic", lambda msg, topic: received.append(msg))
shm.publish("test/topic", {"data": [1, 2, 3]})

import time
time.sleep(0.1)

print(f"Received: {received}")
shm.stop()
```

<!--Result:-->
```
Received: [{'data': [1, 2, 3]}]
```

### DDS 传输层

用于网络通信，DDS 使用 Data Distribution Service（数据分发服务）协议：

```python session=dds_demo ansi=false
from dataclasses import dataclass
from cyclonedds.idl import IdlStruct

from dimos.protocol.pubsub.impl.ddspubsub import DDS, Topic

@dataclass
class SensorReading(IdlStruct):
    value: float

dds = DDS()
dds.start()

received = []
sensor_topic = Topic(name="sensors/temperature", data_type=SensorReading)

dds.subscribe(sensor_topic, lambda msg, t: received.append(msg))
dds.publish(sensor_topic, SensorReading(value=22.5))

import time
time.sleep(0.1)

print(f"Received: {received}")
dds.stop()
```

<!--Result:-->
```
Received: [SensorReading(value=22.5)]
```

---

## 最简传输层实现：`Memory`

最简单的示例后端是 `Memory`（单进程）。实现新的发布/订阅后端时，可以从这里开始。

```python
from dimos.protocol.pubsub.memory import Memory

bus = Memory()
received = []

unsubscribe = bus.subscribe("sensor/data", lambda msg, topic: received.append(msg))

bus.publish("sensor/data", {"temperature": 22.5})
bus.publish("sensor/data", {"temperature": 23.0})

print(f"Received {len(received)} messages:")
for msg in received:
    print(f"  {msg}")

unsubscribe()
```

<!--Result:-->
```
Received 2 messages:
  {'temperature': 22.5}
  {'temperature': 23.0}
```

完整源码参见 [`memory.py`](/dimos/protocol/pubsub/impl/memory.py)。

---

## 编码/解码 mixin

传输层通常需要在发送前序列化消息，在接收后反序列化。

[`pubsub/spec.py`](/dimos/protocol/pubsub/spec.py#L95) 中的 `PubSubEncoderMixin` 提供了一种简洁的方式为任何发布/订阅实现添加编码/解码功能。

### 可用 mixin

| Mixin | 编码方式 | 使用场景 |
|----------------------|-----------------|------------------------------------|
| `PickleEncoderMixin` | Python pickle   | 任意 Python 对象，仅限 Python |
| `LCMEncoderMixin`    | LCM 二进制      | 跨语言（C/C++/Python/Go/…） |
| `JpegEncoderMixin`   | JPEG 压缩       | 图像数据，减少带宽 |

`LCMEncoderMixin` 特别有用：你可以将 LCM 消息定义与*任何*传输层一起使用（不仅限于 UDP 组播）。详情参见 [LCM](/docs/usage/lcm.md)。

### 创建自定义 mixin

```python session=jsonencoder no-result
from dimos.protocol.pubsub.spec import PubSubEncoderMixin
import json

class JsonEncoderMixin(PubSubEncoderMixin[str, dict, bytes]):
    def encode(self, msg: dict, topic: str) -> bytes:
        return json.dumps(msg).encode("utf-8")

    def decode(self, msg: bytes, topic: str) -> dict:
        return json.loads(msg.decode("utf-8"))
```

通过多重继承与发布/订阅实现组合：

```python session=jsonencoder no-result
from dimos.protocol.pubsub.memory import Memory

class MyJsonPubSub(JsonEncoderMixin, Memory):
    pass
```

更换序列化方式只需更改 mixin：

```python session=jsonencoder no-result
from dimos.protocol.pubsub.spec import PickleEncoderMixin

class MyPicklePubSub(PickleEncoderMixin, Memory):
    pass
```

---

## 测试与基准测试

### 规范测试

参见 [`pubsub/test_spec.py`](/dimos/protocol/pubsub/test_spec.py) 了解你的新后端应该通过的网格测试。

### 基准测试

将你的后端添加到基准测试中进行对比：

```sh skip
python -m pytest -svm tool -k "not bytes" dimos/protocol/pubsub/benchmark/test_benchmark.py
```

---

# 可用传输层

| 传输层 | 使用场景 | 跨进程 | 跨网络 | 备注 |
|----------------|-------------------------------------|---------------|---------|--------------------------------------|
| `Memory`       | 仅限测试，单进程 | 否 | 否 | 最简参考实现 |
| `SharedMemory` | 同一机器上的多进程 | 是 | 否 | 最高吞吐量（IPC） |
| `LCM`          | 机器人局域网广播（UDP 组播） | 是 | 是 | 尽力而为；局域网可能丢包 |
| `Redis`        | 通过 Redis 服务器的网络发布/订阅 | 是 | 是 | 中心化代理；增加一跳 |
| `ROS`          | ROS 2 topic 通信 | 是 | 是 | 可与 RViz/ROS 工具集成 |
| `DDS`          | 不依赖 ROS 的 Cyclone DDS（开发中） | 是 | 是 | 开发中 |
