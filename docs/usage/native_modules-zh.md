# 原生模块

阅读本文档前，需要先了解 dimos 的[模块](/docs/usage/modules.md)和[蓝图](/docs/usage/blueprints.md)。

原生模块（Native modules）允许你将**任何可执行文件**包装为一流的 DimOS 模块，前提是它能使用 LCM 通信。

Python 负责蓝图布线、生命周期管理和日志记录。原生二进制文件负责实际计算，直接在 LCM 上进行发布和订阅。

Python 模块**不会接触发布/订阅数据**。它只是通过 CLI 参数将配置和要使用的 LCM topic 传递给你的可执行文件。

关于如何使用 LCM 与 dimos 其余部分通信，请阅读我们的 [LCM 介绍](/docs/usage/lcm.md)。

## 定义原生模块

Python 端的原生模块只是一个 **config** 数据类和一个指定发布/订阅 I/O 的 **module** 类的定义。

config 数据类和发布/订阅 topic 都会被转换为 CLI 参数，在模块启动后传递给你的可执行文件。

```python no-result session=nativemodule
from dataclasses import dataclass
from dimos.core.stream import Out
from dimos.core.transport import LCMTransport
from dimos.core.native_module import NativeModule, NativeModuleConfig
from dimos.msgs.sensor_msgs.PointCloud2 import PointCloud2
from dimos.msgs.sensor_msgs.Imu import Imu
import time

@dataclass(kw_only=True)
class MyLidarConfig(NativeModuleConfig):
    executable: str = "./build/my_lidar"
    host_ip: str = "192.168.1.5"
    frequency: float = 10.0

class MyLidar(NativeModule):
    default_config = MyLidarConfig
    pointcloud: Out[PointCloud2]
    imu: Out[Imu]


```

就这样。`MyLidar` 是一个完整的 DimOS 模块。你可以将它与 `autoconnect`、蓝图、传输覆盖和 spec 一起使用。启动此模块后，你的 `./build/my_lidar` 将使用特定的 CLI 参数被调用。


## 工作原理

当调用 `start()` 时，NativeModule 会：

1. 如果可执行文件不存在且设置了 `build_command`，则**构建可执行文件**。
2. 从每个声明端口上蓝图分配的传输中**收集 topic**。
3. **构建命令行**：`<executable> --<port> <topic> ... --<config_field> <value> ...`
4. 使用 `Popen` **启动子进程**，管道化 stdout/stderr。
5. **启动看门狗**线程，如果进程崩溃则调用 `stop()`。

对于上面的示例，启动的命令如下：

```sh
./build/my_lidar \
    --pointcloud '/pointcloud#sensor_msgs.PointCloud2' \
    --imu '/imu#sensor_msgs.Imu' \
    --host_ip 192.168.1.5 \
    --frequency 10.0
```

```python ansi=false session=nativemodule skip
mylidar = MyLidar()
mylidar.pointcloud.transport = LCMTransport("/lidar", PointCloud2)
mylidar.imu.transport = LCMTransport("/imu", Imu)
mylidar.start()
```

<!--Result:-->
```
2026-02-14T11:22:12.123963Z [info     ] Starting native process   [dimos/core/native_module.py] cmd='./build/my_lidar --pointcloud /lidar#sensor_msgs.PointCloud2 --imu /imu#sensor_msgs.Imu --host_ip 192.168.1.5 --frequency 10.0' cwd=/home/lesh/coding/dimos/docs/usage/build
```

Topic 字符串使用 `/<name>#<msg_type>` 格式，这是 Python `LCMTransport` 订阅者使用的 LCM 通道名称。原生二进制文件在这些确切的通道上发布消息。

当调用 `stop()` 时，进程收到 SIGTERM。如果在 `shutdown_timeout` 秒（默认 10 秒）内没有退出，则发送 SIGKILL。

## 配置

`NativeModuleConfig` 在 `ModuleConfig` 基础上扩展了子进程相关字段：

| 字段 | 类型 | 默认值 | 描述 |
|--------------------|------------------|---------------|-------------------------------------------------------------|
| `executable`       | `str`            | *（必填）*  | 原生二进制文件路径（如果设置了 `cwd` 则相对于 `cwd`） |
| `build_command`    | `str \| None`    | `None`        | 可执行文件缺失时运行的 shell 命令（自动构建） |
| `cwd`              | `str \| None`    | `None`        | 构建和运行时的工作目录。相对路径基于定义模块的 Python 文件所在目录解析 |
| `extra_args`       | `list[str]`      | `[]`          | 附加在自动生成参数之后的额外 CLI 参数 |
| `extra_env`        | `dict[str, str]` | `{}`          | 子进程的额外环境变量 |
| `shutdown_timeout` | `float`          | `10.0`        | 发送 SIGTERM 后等待多少秒再发送 SIGKILL |
| `log_format`       | `LogFormat`      | `TEXT`        | 如何解析子进程输出（`TEXT` 或 `JSON`） |
| `cli_exclude`      | `frozenset[str]` | `frozenset()` | 生成 CLI 参数时要跳过的配置字段 |

### 自动 CLI 参数生成

你添加到 config 子类的任何字段都会自动变为 `--name value` CLI 参数。`NativeModuleConfig` 本身的字段（如 `executable`、`extra_args`、`cwd`）**不会**被传递 — 它们仅用于 Python 端的编排。

```python skip

class LogFormat(enum.Enum):
    TEXT = "text"
    JSON = "json"

@dataclass(kw_only=True)
class MyConfig(NativeModuleConfig):
    executable: str = "./build/my_module" # 可执行文件的相对或绝对路径
    host_ip: str = "192.168.1.5"     # 变为 --host_ip 192.168.1.5
    frequency: float = 10.0           # 变为 --frequency 10.0
    enable_imu: bool = True           # 变为 --enable_imu true
    filters: list[str] = field(default_factory=lambda: ["a", "b"])  # 变为 --filters a,b
```

- `None` 值会被跳过。
- 布尔值转为小写（`true`/`false`）。
- 列表以逗号连接。

### 排除字段

如果一个配置字段不应该成为 CLI 参数，将其添加到 `cli_exclude`：

```python skip
@dataclass(kw_only=True)
class FastLio2Config(NativeModuleConfig):
    executable: str = "./build/fastlio2"
    config: str = "mid360.yaml"                          # 人类友好的名称
    config_path: str | None = None                       # 解析后的绝对路径
    cli_exclude: frozenset[str] = frozenset({"config"})  # 只传递 config_path

    def __post_init__(self) -> None:
        if self.config_path is None:
            self.config_path = str(Path(self.config).resolve())
```

## 与蓝图配合使用

原生模块与 `autoconnect` 的使用方式和 Python 模块完全一样：

```python skip
from dimos.core.blueprints import autoconnect

class PointCloudConsumer(Module):
    pointcloud: In[PointCloud2]
    imu: In[Imu]

autoconnect(
    MyLidar.blueprint(host_ip="192.168.1.10"),
    PointCloudConsumer.blueprint(),
).build().loop()
```

`autoconnect` 按 `(名称, 类型)` 匹配端口，分配 LCM topic，并将它们作为 CLI 参数传递给原生二进制文件。你可以照常覆盖传输层：

```python skip
blueprint = autoconnect(
    MyLidar.blueprint(),
    PointCloudConsumer.blueprint(),
).transports({
    ("pointcloud", PointCloud2): LCMTransport("/my/custom/lidar", PointCloud2),
})
```

## 日志

NativeModule 将子进程的 stdout 和 stderr 通过 structlog 进行管道化：

- **stdout** 以 `info` 级别记录。
- **stderr** 以 `warning` 级别记录。

### JSON 日志格式

如果你的原生二进制文件输出结构化 JSON 行，设置 `log_format=LogFormat.JSON`：

```python skip
@dataclass(kw_only=True)
class MyConfig(NativeModuleConfig):
    executable: str = "./build/my_module"
    log_format: LogFormat = LogFormat.JSON
```

模块会将每行解析为 JSON，并将键值对传入 structlog。`event` 键成为日志消息：

```json
{"event": "sensor initialized", "device": "/dev/ttyUSB0", "baud": 115200}
```

格式错误的行会回退到纯文本日志。

## 编写 C++ 端

在 [`dimos/hardware/sensors/lidar/common/dimos_native_module.hpp`](/dimos/hardware/sensors/lidar/common/dimos_native_module.hpp) 提供了一个仅头文件的辅助工具：

```cpp
#include "dimos_native_module.hpp"
#include "sensor_msgs/PointCloud2.hpp"

int main(int argc, char** argv) {
    dimos::NativeModule mod(argc, argv);

    // 获取声明端口的 LCM 通道
    std::string pc_topic = mod.topic("pointcloud");

    // 获取配置值
    float freq = mod.arg_float("frequency", 10.0);
    std::string ip = mod.arg("host_ip", "192.168.1.5");

    // 设置 LCM 发布者并在 pc_topic 上发布...
}
```

该辅助工具提供：

| 方法 | 描述 |
|---------------------------|----------------------------------------------------------------|
| `topic(port)`             | 获取端口的完整 LCM 通道字符串（`/topic#msg_type`） |
| `arg(key, default)`       | 获取字符串配置值 |
| `arg_float(key, default)` | 获取浮点配置值 |
| `arg_int(key, default)`   | 获取整数配置值 |
| `has(key)`                | 检查端口/参数是否已提供 |

它还包含 `make_header()` 和 `time_from_seconds()` 用于构建与 ROS 兼容的带时间戳消息。

## 示例

关于语言互操作示例（从 C++、TypeScript、Lua 订阅 DimOS topic），请参阅 [/examples/language-interop/](/examples/language-interop/README.md)。

### Livox Mid-360 模块

Livox Mid-360 LiDAR 驱动是一个完整的示例，位于 [`dimos/hardware/sensors/lidar/livox/module.py`](/dimos/hardware/sensors/lidar/livox/module.py)：

```python skip
from dimos.core.stream import Out
from dimos.core.native_module import NativeModule, NativeModuleConfig
from dimos.msgs.sensor_msgs.PointCloud2 import PointCloud2
from dimos.msgs.sensor_msgs.Imu import Imu
from dimos.spec import perception

@dataclass(kw_only=True)
class Mid360Config(NativeModuleConfig):
    cwd: str | None = "cpp"
    executable: str = "result/bin/mid360_native"
    build_command: str | None = "nix build .#mid360_native"
    host_ip: str = "192.168.1.5"
    lidar_ip: str = "192.168.1.155"
    frequency: float = 10.0
    enable_imu: bool = True
    frame_id: str = "lidar_link"
    # ... SDK 端口配置

class Mid360(NativeModule, perception.Lidar, perception.IMU):
    default_config = Mid360Config
    lidar: Out[PointCloud2]
    imu: Out[Imu]
```

用法：

```python skip
from dimos.hardware.sensors.lidar.livox.module import Mid360

autoconnect(
    Mid360.blueprint(host_ip="192.168.1.5"),
    SomeConsumer.blueprint(),
)
```

## 自动构建

如果模块配置中设置了 `build_command`，且在调用 `start()` 时可执行文件不存在，NativeModule 会自动运行构建命令。
构建输出通过 structlog 管道化（stdout 为 `info` 级别，stderr 为 `warning` 级别）。

```python skip
@dataclass(kw_only=True)
class MyLidarConfig(NativeModuleConfig):
    cwd: str | None = "cpp"
    executable: str = "result/bin/my_lidar"
    build_command: str | None = "nix build .#my_lidar"
```

`cwd` 同时用于构建命令和运行时子进程。相对路径基于定义模块的 Python 文件所在目录解析。

如果可执行文件已经存在，构建步骤将被完全跳过。
