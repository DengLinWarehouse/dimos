# 传感器存储与回放

将传感器流录制到磁盘并按原始时序回放。适用于测试、调试和创建可复现的数据集。

## 快速入门

### 录制

```python skip
from dimos.utils.testing.replay import TimedSensorStorage

# 创建存储（数据目录中的子目录）
storage = TimedSensorStorage("my_recording")

# 从流中保存帧
camera_stream.subscribe(storage.save_one)

# 或手动保存
storage.save(frame1, frame2, frame3)
```

### 回放

```python skip
from dimos.utils.testing.replay import TimedSensorReplay

# 加载录制数据
replay = TimedSensorReplay("my_recording")

# 按原始速度迭代
for frame in replay.iterate_realtime():
    process(frame)

# 或作为 Observable 流
replay.stream(speed=1.0).subscribe(process)
```

## TimedSensorStorage

将传感器数据连同时间戳存储为 pickle 文件。每一帧保存为 `000.pickle`、`001.pickle` 等。

```python skip
from dimos.utils.testing.replay import TimedSensorStorage

storage = TimedSensorStorage("lidar_capture")

# 保存单帧
storage.save_one(lidar_msg)  # 返回帧计数

# 保存多帧
storage.save(frame1, frame2, frame3)

# 订阅流
lidar_stream.subscribe(storage.save_one)

# 或通过管道传递（发出帧计数）
lidar_stream.pipe(
    ops.flat_map(storage.save_stream)
).subscribe()
```

**存储位置：** 文件保存在给定名称下的数据目录中。该目录不能已包含 pickle 文件（防止意外覆盖）。

**存储内容：** 默认情况下，如果帧具有 `.raw_msg` 属性，则序列化该属性而非完整对象。你可以通过 `autocast` 参数自定义：

```python skip
# 自定义序列化
storage = TimedSensorStorage(
    "custom_capture",
    autocast=lambda frame: frame.to_dict()
)
```

## TimedSensorReplay

回放存储的传感器数据，支持带时间戳的迭代和跳转。

### 基本迭代

```python skip
from dimos.utils.testing.replay import TimedSensorReplay

replay = TimedSensorReplay("lidar_capture")

# 迭代所有帧（忽略时序）
for frame in replay.iterate():
    process(frame)

# 带时间戳迭代
for ts, frame in replay.iterate_ts():
    print(f"Frame at {ts}: {frame}")

# 带相对时间戳迭代（从起始算起）
for relative_ts, frame in replay.iterate_duration():
    print(f"At {relative_ts:.2f}s: {frame}")
```

### 实时回放

```python skip
# 按原始速度播放（在帧之间阻塞）
for frame in replay.iterate_realtime():
    process(frame)

# 2 倍速播放
for frame in replay.iterate_realtime(speed=2.0):
    process(frame)

# 半速播放
for frame in replay.iterate_realtime(speed=0.5):
    process(frame)
```

### 跳转与切片

```python skip
# 从录制的第 10 秒开始
for ts, frame in replay.iterate_ts(seek=10.0):
    process(frame)

# 从第 10 秒开始只播放 5 秒
for ts, frame in replay.iterate_ts(seek=10.0, duration=5.0):
    process(frame)

# 永久循环
for frame in replay.iterate(loop=True):
    process(frame)
```

### 查找特定帧

```python skip
# 查找最接近绝对时间戳的帧
frame = replay.find_closest(1704067200.0)

# 查找最接近相对时间的帧（从起始算起 30 秒）
frame = replay.find_closest_seek(30.0)

# 带容差（如果 0.1 秒内无匹配则返回 None）
frame = replay.find_closest(timestamp, tolerance=0.1)
```

### Observable 流

`.stream()` 方法返回一个按原始时序发出帧的 Observable：

```python skip
# 按原始速度流式播放
replay.stream(speed=1.0).subscribe(process)

# 2 倍速带跳转的流式播放
replay.stream(
    speed=2.0,
    seek=10.0,      # 从第 10 秒开始
    duration=30.0,  # 播放 30 秒
    loop=True       # 永久循环
).subscribe(process)
```

## 用法：用于测试的桩连接

一个常见的模式是创建基于回放的连接桩，用于无硬件测试。出自 [`robot/unitree/go2/connection.py`](/dimos/robot/unitree/go2/connection.py#L83)：

这种方式比较原始。我们希望编写一个更高层的 API 来录制任意模块的完整输入输出，但目前仍在开发中。


```python skip
class ReplayConnection(UnitreeWebRTCConnection):
    dir_name = "go2_sf_office"

    def __init__(self, **kwargs) -> None:
        get_data(self.dir_name)
        self.replay_config = {
            "loop": kwargs.get("loop"),
            "seek": kwargs.get("seek"),
            "duration": kwargs.get("duration"),
        }

    def lidar_stream(self):
        lidar_store = TimedSensorReplay(f"{self.dir_name}/lidar")
        return lidar_store.stream(**self.replay_config)

    def video_stream(self):
        video_store = TimedSensorReplay(f"{self.dir_name}/video")
        return video_store.stream(**self.replay_config)
```

这允许对录制的数据运行完整的感知管道：

```python skip
# 使用回放连接代替真实硬件
connection = ReplayConnection(loop=True, seek=5.0)
robot = GO2Connection(connection=connection)
```

## 数据格式

每个 pickle 文件包含一个元组 `(timestamp, data)`：

- **timestamp**：帧被捕获时的 Unix 时间戳（float）
- **data**：传感器数据（如果提供了 `autocast` 则为其返回结果）

文件按顺序编号：`000.pickle`、`001.pickle` 等。

录制数据存储在 `data/` 目录中。有关数据存储的工作方式（包括大数据集的 Git LFS 处理），请参见[数据加载](/docs/development/large_file_management.md)。

## API 参考

### TimedSensorStorage

| 方法 | 描述 |
|------------------------------|------------------------------------------|
| `save_one(frame)` | 保存单帧，返回帧计数 |
| `save(*frames)` | 保存多帧 |
| `save_stream(observable)` | 通过管道将 observable 传入存储 |
| `consume_stream(observable)` | 订阅并保存，不返回结果 |

### TimedSensorReplay

| 方法 | 描述 |
|--------------------------------------------------|---------------------------------------|
| `iterate(loop=False)` | 迭代帧（不考虑时序） |
| `iterate_ts(seek, duration, loop)` | 带绝对时间戳迭代 |
| `iterate_duration(...)` | 带相对时间戳迭代 |
| `iterate_realtime(speed, ...)` | 阻塞式迭代以匹配原始时序 |
| `stream(speed, seek, duration, loop)` | 带原始时序的 Observable |
| `find_closest(timestamp, tolerance)` | 按绝对时间戳查找帧 |
| `find_closest_seek(relative_seconds, tolerance)` | 按相对时间查找帧 |
| `first()` | 获取第一帧 |
| `first_timestamp()` | 获取第一个时间戳 |
| `load(name)` | 按名称/索引加载特定帧 |
