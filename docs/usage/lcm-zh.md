# LCM 消息

DimOS 使用 [LCM（Lightweight Communications and Marshalling，轻量级通信与序列化）](https://github.com/lcm-proj/lcm) 在本地机器上进行进程间通信（类似于 ROS 使用 DDS 的方式）。LCM 是一种简单的 [UDP 组播](https://lcm-proj.github.io/lcm/content/udp-multicast-protocol.html#lcm-udp-multicast-protocol-description)发布/订阅协议，具有简洁的[消息定义语言](https://lcm-proj.github.io/lcm/content/lcm-type-ref.html#lcm-type-specification-language)。

LCM 项目为许多语言提供了发布/订阅客户端和代码生成器。对我们来说，LCM 的强大之处在于其消息定义格式——多语言类将自身编码为紧凑的二进制格式。这意味着 LCM 消息可以通过任何传输层（WebSocket、SSH、共享内存等）在不同编程语言之间发送。

我们的消息类型移植自 ROS（结构兼容，以便在需要时能够轻松与 ROS 通信）。
托管消息定义和自动生成器的仓库在 [dimos-lcm](https://github.com/dimensionalOS/dimos-lcm/)。

我们的 LCM 实现在本地通信方面[显著优于 ROS](/docs/usage/transports/index.md#benchmarks)。

## 支持的语言

除 Python 外，我们还提供了以下语言的 LCM 集成示例：
- [**C++**](/examples/language-interop/cpp/README.md)
- [**TypeScript**](/examples/language-interop/ts/README.md)
- [**Lua**](/examples/language-interop/lua/README.md)

位于 [/examples/language-interop/](/examples/language-interop/) 目录下。

已生成类型（但尚无示例）的语言：
[**C#**](https://github.com/dimensionalOS/dimos-lcm/tree/main/generated/csharp) 和 [**Java**](https://github.com/dimensionalOS/dimos-lcm/tree/main/generated/java)

### 原生模块

由于 LCM 的高可移植性，我们可以轻松运行用[第三方语言](/docs/usage/native_modules.md)编写的 dimos [模块](/docs/usage/modules.md)。

## dimos-lcm 包

`dimos-lcm` 包提供了与 [ROS 消息定义](https://docs.ros.org/en/melodic/api/sensor_msgs/html/index.html)对应的基础消息类型：

```python session=lcm_demo ansi=false
from dimos_lcm.geometry_msgs import Vector3 as LCMVector3
from dimos_lcm.sensor_msgs.PointCloud2 import PointCloud2 as LCMPointCloud2

# LCM 消息可以编码为二进制
msg = LCMVector3()
msg.x, msg.y, msg.z = 1.0, 2.0, 3.0

binary = msg.lcm_encode()
print(f"Encoded to {len(binary)} bytes: {binary.hex()}")

# 也可以解码回来
decoded = LCMVector3.lcm_decode(binary)
print(f"Decoded: x={decoded.x}, y={decoded.y}, z={decoded.z}")
```

<!--Result:-->
```
Encoded to 24 bytes: 000000000000f03f00000000000000400000000000000840
Decoded: x=1.0, y=2.0, z=3.0
```

## Dimos 消息覆盖层

Dimos 对基础 LCM 类型进行了子类化，添加了 Python 友好的特性，同时保持二进制兼容性。例如，`dimos.msgs.geometry_msgs.Vector3` 在 LCM 基类上扩展了：

- 多种构造函数重载（从元组、numpy 数组等构造）
- 数学运算（`+`、`-`、`*`、`/`、点积、叉积）
- 转换为 numpy、四元数等

```python session=lcm_demo ansi=false
from dimos.msgs.geometry_msgs import Vector3

# 丰富的构造函数
v1 = Vector3(1, 2, 3)
v2 = Vector3([4, 5, 6])
v3 = Vector3(v1)  # 复制

# 数学运算
print(f"v1 + v2 = {(v1 + v2).to_tuple()}")
print(f"v1 dot v2 = {v1.dot(v2)}")
print(f"v1 x v2 = {v1.cross(v2).to_tuple()}")
print(f"|v1| = {v1.length():.3f}")

# 仍然可以编码为 LCM 二进制
binary = v1.lcm_encode()
print(f"LCM encoded: {len(binary)} bytes")
```

<!--Result:-->
```
v1 + v2 = (5.0, 7.0, 9.0)
v1 dot v2 = 32.0
v1 x v2 = (-3.0, 6.0, -3.0)
|v1| = 3.742
LCM encoded: 24 bytes
```

## PointCloud2 与 Open3D

一个更复杂的例子是 `PointCloud2`，它封装了 Open3D 点云，同时保持 LCM 二进制兼容性：

```python session=lcm_demo ansi=false
import numpy as np
from dimos.msgs.sensor_msgs import PointCloud2

# 从 numpy 创建
points = np.random.rand(100, 3).astype(np.float32)
pc = PointCloud2.from_numpy(points, frame_id="camera")

print(f"PointCloud: {len(pc)} points, frame={pc.frame_id}")
print(f"Center: {pc.center}")

# 以 Open3D 形式访问（用于可视化、处理）
o3d_cloud = pc.pointcloud
print(f"Open3D type: {type(o3d_cloud).__name__}")

# 编码为 LCM 二进制（用于传输）
binary = pc.lcm_encode()
print(f"LCM encoded: {len(binary)} bytes")

# 解码回来
pc2 = PointCloud2.lcm_decode(binary)
print(f"Decoded: {len(pc2)} points")
```

<!--Result:-->
```
PointCloud: 100 points, frame=camera
Center: ↗ Vector (Vector([0.49166839, 0.50896413, 0.48393918]))
Open3D type: PointCloud
LCM encoded: 1716 bytes
Decoded: 100 points
```

## 传输无关性

由于 LCM 消息编码为字节，你可以通过任何传输层使用它们：

```python session=lcm_demo ansi=false
from dimos.msgs.geometry_msgs import Vector3
from dimos.protocol.pubsub.memory import Memory
from dimos.protocol.pubsub.shmpubsub import PickleSharedMemory

# 同一消息可用于任何传输层
msg = Vector3(1, 2, 3)

# 内存传输（同一进程）
memory = Memory()
received = []
memory.subscribe("velocity", lambda m, t: received.append(m))
memory.publish("velocity", msg)
print(f"Memory transport: received {received[0]}")

# LCM 二进制也可以通过任何面向字节的通道原始发送
binary = msg.lcm_encode()
# 通过 WebSocket、Redis、TCP、文件等发送
decoded = Vector3.lcm_decode(binary)
print(f"Raw binary transport: decoded {decoded}")
```

<!--Result:-->
```
Memory transport: received ↗ Vector (Vector([1. 2. 3.]))
Raw binary transport: decoded ↗ Vector (Vector([1. 2. 3.]))
```

## 可用消息类型

Dimos 为常用消息类型提供了覆盖层：

| 包 | 消息 |
|---------|----------|
| `geometry_msgs` | `Vector3`, `Quaternion`, `Pose`, `Twist`, `Transform` |
| `sensor_msgs` | `Image`, `PointCloud2`, `CameraInfo`, `LaserScan` |
| `nav_msgs` | `Odometry`, `Path`, `OccupancyGrid` |
| `vision_msgs` | `Detection2D`, `Detection3D`, `BoundingBox2D` |

基础 LCM 类型（不含 Dimos 扩展）可通过 `dimos_lcm.*` 访问。

## 创建自定义消息类型

创建新消息类型的步骤：

1. 以 `.lcm` 格式定义 LCM 消息（或使用现有的 `dimos_lcm` 基类）
2. 创建一个继承 LCM 类型的 Python 覆盖类
3. 如果需要自定义序列化，添加 `lcm_encode()` 和 `lcm_decode()` 方法

示例请参见 [`PointCloud2.py`](/dimos/msgs/sensor_msgs/PointCloud2.py) 和 [`Vector3.py`](/dimos/msgs/geometry_msgs/Vector3.py)。
