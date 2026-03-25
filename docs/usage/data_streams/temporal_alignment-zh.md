# 时间消息对齐

机器人拥有多个以不同速率和延迟发射数据的传感器。摄像头可能以 30fps 运行，而激光雷达以 10Hz 扫描，且每个传感器有不同的处理延迟。对于将 2D 检测投影到 3D 点云这类感知任务，我们需要根据时间戳匹配来自不同数据流的数据。

`align_timestamped` 通过缓冲消息并在时间容差范围内匹配来解决此问题。

<details><summary>Pikchr</summary>

```pikchr fold output=assets/alignment_overview.svg
color = white
fill = none

Cam: box "Camera" "30 fps" rad 5px fit wid 170% ht 170%
arrow from Cam.e right 0.4in then down 0.35in then right 0.4in
Align: box "align_timestamped" rad 5px fit wid 170% ht 170%

Lidar: box "Lidar" "10 Hz" rad 5px fit wid 170% ht 170% with .s at (Cam.s.x, Cam.s.y - 0.7in)
arrow from Lidar.e right 0.4in then up 0.35in then right 0.4in

arrow from Align.e right 0.4in
Out: box "(image, pointcloud)" rad 5px fit wid 170% ht 170%
```

</details>

<!--Result:-->
![output](assets/alignment_overview.svg)


## 基本用法

下面我们设置 Unitree Go2 机器人的真实摄像头和激光雷达数据的回放。如果你感兴趣可以查看。

<details>
<summary>数据流设置</summary>

你可以在[传感器存储](/docs/usage/data_streams/storage_replay.md)和 [LFS 数据存储](/docs/development/large_file_management.md)中了解更多信息。

```python session=align no-result
from reactivex import Subject
from dimos.utils.testing import TimedSensorReplay
from dimos.types.timestamped import Timestamped, align_timestamped
from reactivex import operators as ops
import reactivex as rx

# 加载录制的 Go2 传感器数据流
video_replay = TimedSensorReplay("go2_sf_office/video")
lidar_replay = TimedSensorReplay("go2_sf_office/lidar")

# 这里有点技巧。我们找到第一个视频帧的时间戳，然后加 2 秒。
seek_ts = video_replay.first_timestamp() + 2

# 用于收集流中数据项的列表
video_frames = []
lidar_scans = []

# 这里使用 from_timestamp=... 而不是 seek=...，因为 seek 按录制时间戳跳转，
# from_timestamp 按实际消息时间戳匹配。
# 传感器数据可能延迟到达，但具有正确的采集时间戳
video_stream = video_replay.stream(from_timestamp=seek_ts, duration=2.0).pipe(
    ops.do_action(lambda x: video_frames.append(x))
)

lidar_stream = lidar_replay.stream(from_timestamp=seek_ts, duration=2.0).pipe(
    ops.do_action(lambda x: lidar_scans.append(x))
)

```


</details>

数据流通常通过 `In` 输入从实际机器人传入你的模块。[`detection/module3D.py`](/dimos/perception/detection/module3D.py#L11) 是一个很好的示例。

假设我们已经有了数据流。让我们进行对齐。

```python session=align
# 将视频（主流）与激光雷达（辅助流）对齐
# match_tolerance：匹配的最大时间差（秒）
# buffer_size：保留消息等待匹配的时长（秒）
aligned_pairs = align_timestamped(
    video_stream,
    lidar_stream,
    match_tolerance=0.025,  # 25ms 容差
    buffer_size=5.0, # 等待匹配的时长
).pipe(ops.to_list()).run()

print(f"Video: {len(video_frames)} frames, Lidar: {len(lidar_scans)} scans")
print(f"Aligned pairs: {len(aligned_pairs)} out of {len(video_frames)} video frames")

# 显示一个匹配对
if aligned_pairs:
    img, pc = aligned_pairs[0]
    dt = abs(img.ts - pc.ts)
    print(f"\nFirst matched pair: Δ{dt*1000:.1f}ms")
```

<!--Result:-->
```
Video: 29 frames, Lidar: 15 scans
Aligned pairs: 11 out of 29 video frames

First matched pair: Δ11.3ms
```

<details>
<summary>可视化辅助函数</summary>

```python session=align fold no-result
import matplotlib
import matplotlib.pyplot as plt

def plot_alignment_timeline(video_frames, lidar_scans, aligned_pairs, path):
    """单一时间轴：视频在轴上方，激光雷达在轴下方，绿线表示匹配。"""
    matplotlib.use('Agg')
    plt.style.use('dark_background')

    # 获取基准时间戳用于计算相对时间（帧具有 .ts 属性）
    base_ts = video_frames[0].ts
    video_ts = [f.ts - base_ts for f in video_frames]
    lidar_ts = [s.ts - base_ts for s in lidar_scans]

    # 查找已匹配的时间戳
    matched_video_ts = set(img.ts for img, _ in aligned_pairs)
    matched_lidar_ts = set(pc.ts for _, pc in aligned_pairs)

    fig, ax = plt.subplots(figsize=(12, 2.5))

    # 视频标记在轴上方 (y=0.3) - 圆形，匹配时为青色
    for frame in video_frames:
        rel_ts = frame.ts - base_ts
        matched = frame.ts in matched_video_ts
        ax.plot(rel_ts, 0.3, 'o', color='cyan' if matched else '#688', markersize=8)

    # 激光雷达标记在轴下方 (y=-0.3) - 方形，匹配时为橙色
    for scan in lidar_scans:
        rel_ts = scan.ts - base_ts
        matched = scan.ts in matched_lidar_ts
        ax.plot(rel_ts, -0.3, 's', color='orange' if matched else '#a86', markersize=8)

    # 绿线连接匹配对
    for img, pc in aligned_pairs:
        img_rel = img.ts - base_ts
        pc_rel = pc.ts - base_ts
        ax.plot([img_rel, pc_rel], [0.3, -0.3], '-', color='lime', alpha=0.6, linewidth=1)

    # 坐标轴样式
    ax.axhline(y=0, color='white', linewidth=0.5, alpha=0.3)
    ax.set_xlim(-0.1, max(video_ts + lidar_ts) + 0.1)
    ax.set_ylim(-0.6, 0.6)
    ax.set_xlabel('Time (s)')
    ax.set_yticks([0.3, -0.3])
    ax.set_yticklabels(['Video', 'Lidar'])
    ax.set_title(f'{len(aligned_pairs)} matched from {len(video_frames)} video + {len(lidar_scans)} lidar')
    plt.tight_layout()
    plt.savefig(path, transparent=True)
    plt.close()
```

</details>

```python session=align output=assets/alignment_timeline.png
plot_alignment_timeline(video_frames, lidar_scans, aligned_pairs, '{output}')
```

<!--Result:-->
![output](assets/alignment_timeline.png)

如果我们放宽匹配容差，可能会有多对匹配到同一个激光雷达帧。

```python session=align
aligned_pairs = align_timestamped(
    video_stream,
    lidar_stream,
    match_tolerance=0.05,  # 50ms 容差
    buffer_size=5.0, # 等待匹配的时长
).pipe(ops.to_list()).run()

print(f"Video: {len(video_frames)} frames, Lidar: {len(lidar_scans)} scans")
print(f"Aligned pairs: {len(aligned_pairs)} out of {len(video_frames)} video frames")
```

<!--Result:-->
```
Video: 58 frames, Lidar: 30 scans
Aligned pairs: 23 out of 58 video frames
```


```python session=align output=assets/alignment_timeline2.png
plot_alignment_timeline(video_frames, lidar_scans, aligned_pairs, '{output}')
```

<!--Result:-->
![output](assets/alignment_timeline2.png)

## 将帧对齐与质量过滤结合

更多关于[质量过滤的信息](/docs/usage/data_streams/quality_filter.md)。

```python session=align
from dimos.msgs.sensor_msgs.Image import Image, sharpness_barrier

# 用于收集流中数据项的列表
video_frames = []
lidar_scans = []

video_stream = video_replay.stream(from_timestamp=seek_ts, duration=2.0).pipe(
    sharpness_barrier(3.0),
    ops.do_action(lambda x: video_frames.append(x))
)

lidar_stream = lidar_replay.stream(from_timestamp=seek_ts, duration=2.0).pipe(
    ops.do_action(lambda x: lidar_scans.append(x))
)

aligned_pairs = align_timestamped(
    video_stream,
    lidar_stream,
    match_tolerance=0.025,  # 25ms 容差
    buffer_size=5.0, # 等待匹配的时长
).pipe(ops.to_list()).run()

print(f"Video: {len(video_frames)} frames, Lidar: {len(lidar_scans)} scans")
print(f"Aligned pairs: {len(aligned_pairs)} out of {len(video_frames)} video frames")

```

<!--Result:-->
```
Video: 6 frames, Lidar: 15 scans
Aligned pairs: 1 out of 6 video frames
```

```python session=align output=assets/alignment_timeline3.png
plot_alignment_timeline(video_frames, lidar_scans, aligned_pairs, '{output}')
```

<!--Result:-->
![output](assets/alignment_timeline3.png)

我们筛选条件非常严格，但数据质量很高。在此窗口中选择了最佳帧以及最接近的激光雷达匹配。

## 工作原理

主流（第一个参数）驱动输出。当主流消息到达时：

1. **立即匹配**：如果缓冲区中已存在匹配的辅助流消息，立即发出
2. **延迟匹配**：如果缺少辅助流消息，缓冲主流消息并等待

当辅助流消息到达时：
1. 添加到缓冲区供未来的主流消息匹配
2. 检查已缓冲的主流消息——如果此消息完成了匹配，则发出

<details>
<summary>diagram source</summary>

```pikchr fold output=assets/alignment_flow.svg
color = white
fill = none
linewid = 0.35in

Primary: box "Primary" "arrives" rad 5px fit wid 170% ht 170%
arrow
Check: box "Check" "secondaries" rad 5px fit wid 170% ht 170%

arrow from Check.e right 0.35in then up 0.4in then right 0.35in
Emit: box "Emit" "match" rad 5px fit wid 170% ht 170%
text "all found" at (Emit.w.x - 0.4in, Emit.w.y + 0.15in)

arrow from Check.e right 0.35in then down 0.4in then right 0.35in
Buffer: box "Buffer" "primary" rad 5px fit wid 170% ht 170%
text "waiting..." at (Buffer.w.x - 0.4in, Buffer.w.y - 0.15in)
```

</details>

<!--Result:-->
![output](assets/alignment_flow.svg)

## 参数

| 参数 | 类型 | 默认值 | 描述 |
|--------------------------|--------------------|----------|------------------------------------------------|
| `primary_observable` | `Observable[T]` | 必填 | 驱动输出时序的主流 |
| `*secondary_observables` | `Observable[S]...` | 必填 | 一个或多个需要对齐的辅助流 |
| `match_tolerance` | `float` | 0.1 | 匹配的最大时间差（秒） |
| `buffer_size` | `float` | 1.0 | 缓冲未匹配消息的时长（秒） |



## 在模块中的用法

每个模块的 `In` 端口都暴露了一个 `.observable()` 方法，返回经过背压处理的传入消息流。这使得对齐多个传感器的输入变得简单。

出自 [`detection/module3D.py`](/dimos/perception/detection/module3D.py)，将 2D 检测投影到 3D 点云：

```python skip
class Detection3DModule(Detection2DModule):
    color_image: In[Image]
    pointcloud: In[PointCloud2]

    def start(self):
        # 将 2D 检测与点云数据对齐
        self.detection_stream_3d = align_timestamped(
            backpressure(self.detection_stream_2d()),
            self.pointcloud.observable(),
            match_tolerance=0.25,
            buffer_size=20.0,
        ).pipe(ops.map(detection2d_to_3d))
```

2D 检测流（摄像头 + ML 模型）是主流，与激光雷达的原始点云数据匹配。较长的 `buffer_size=20.0` 考虑了 ML 推理时间的不确定性。
