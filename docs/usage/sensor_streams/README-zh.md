# 传感器数据流

Dimos 使用响应式流（RxPY）来处理传感器数据。这种方式天然适合机器人领域——多个传感器以不同频率异步发送数据，而下游处理器的速度可能跟不上数据源。

## 指南

| 指南 | 描述 |
|----------------------------------------------|---------------------------------------------------------------|
| [ReactiveX 基础](/docs/usage/sensor_streams/reactivex.md) | Observable、订阅和 disposable |
| [高级数据流](/docs/usage/sensor_streams/advanced_streams.md) | 背压处理、并行订阅者、同步获取器 |
| [基于质量的过滤](/docs/usage/sensor_streams/quality_filter.md) | 在数据流降采样时选择最高质量的帧 |
| [时间对齐](/docs/usage/sensor_streams/temporal_alignment.md) | 按时间戳匹配多个传感器的消息 |
| [存储与回放](/docs/usage/sensor_streams/storage_replay.md) | 将传感器流录制到磁盘并以原始时序回放 |

## 快速示例

```python
from reactivex import operators as ops
from dimos.utils.reactive import backpressure
from dimos.types.timestamped import align_timestamped
from dimos.msgs.sensor_msgs.Image import sharpness_barrier

# 相机 30fps，LiDAR 10Hz
camera_stream = camera.observable()
lidar_stream = lidar.observable()

# 流水线：过滤模糊帧 -> 与 LiDAR 对齐 -> 处理慢消费者
processed = (
    camera_stream.pipe(
        sharpness_barrier(10.0),  # 每 100ms 窗口保留最清晰的帧（10Hz）
    )
)

aligned = align_timestamped(
    backpressure(processed),     # 相机作为主流
    lidar_stream,                # LiDAR 作为副流
    match_tolerance=0.1,
)

aligned.subscribe(lambda pair: process_frame_with_pointcloud(*pair))
```
