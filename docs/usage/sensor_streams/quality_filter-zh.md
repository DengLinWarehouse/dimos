# 基于质量的流过滤

在处理传感器流时，你通常想降低频率同时保留最高质量的数据。对于像图像这样无法求均值或合并的离散数据，`quality_barrier` 不会盲目丢帧，而是在每个时间窗口内选择质量最高的项。

## 问题

摄像头输出 30fps，但你的 ML 模型只需要 2fps。简单的方法：

- **`sample(0.5)`** - 取恰好落在间隔点上的帧
- **`throttle_first(0.5)`** - 取第一帧，忽略其余的

两者都忽略了质量。你可能得到一帧模糊的图像，而明明有清晰的帧可用。

## 解决方案：`quality_barrier`

```python session=qb
import reactivex as rx
from reactivex import operators as ops
from dimos.utils.reactive import quality_barrier

# 带质量分数的模拟传感器数据
data = [
    {"id": 1, "quality": 0.3},
    {"id": 2, "quality": 0.9},  # 第一个窗口中最佳
    {"id": 3, "quality": 0.5},
    {"id": 4, "quality": 0.2},
    {"id": 5, "quality": 0.8},  # 第二个窗口中最佳
    {"id": 6, "quality": 0.4},
]

source = rx.of(*data)

# 每个窗口选择最高质量项（每秒 2 项 = 0.5 秒窗口）
result = source.pipe(
    quality_barrier(lambda x: x["quality"], target_frequency=2.0),
    ops.to_list(),
).run()

print("Selected:", [r["id"] for r in result])
print("Qualities:", [r["quality"] for r in result])
```

<!--Result:-->
```
Selected: [2]
Qualities: [0.9]
```

## 图像清晰度过滤

对于摄像头流，我们提供了 `sharpness_barrier`，它使用图像的清晰度分数。

让我们使用 Unitree Go2 机器人的真实摄像头数据来演示。我们使用[传感器存储与回放](/docs/usage/sensor_streams/storage_replay.md)工具包，它提供对录制的机器人数据的访问：

```python session=qb
from dimos.utils.testing import TimedSensorReplay
from dimos.msgs.sensor_msgs.Image import Image, sharpness_barrier

# 加载录制的 Go2 摄像头帧
video_replay = TimedSensorReplay("go2_sf_office/video")

# 使用 stream() 配合 seek 跳过空白帧，speed=10x 加速采集
input_frames = video_replay.stream(seek=5.0, duration=1.4, speed=10.0).pipe(
    ops.to_list()
).run()

def show_frames(frames):
   for i, frame in enumerate(frames[:10]):
      print(f"  Frame {i}: {frame.sharpness:.3f}")

print(f"Loaded {len(input_frames)} frames from Go2 camera")
print(f"Frame resolution: {input_frames[0].width}x{input_frames[0].height}")
print("Sharpness scores:")
show_frames(input_frames)
```

<!--Result:-->
```
Loaded 20 frames from Go2 camera
Frame resolution: 1280x720
Sharpness scores:
  Frame 0: 0.351
  Frame 1: 0.227
  Frame 2: 0.223
  Frame 3: 0.267
  Frame 4: 0.295
  Frame 5: 0.307
  Frame 6: 0.328
  Frame 7: 0.348
  Frame 8: 0.346
  Frame 9: 0.322
```

使用 `sharpness_barrier` 选择最清晰的帧：

```python session=qb
# 从录制的帧创建流

sharp_frames = video_replay.stream(seek=5.0, duration=1.5, speed=1.0).pipe(
    sharpness_barrier(2.0),
    ops.to_list()
).run()

print(f"Output: {len(sharp_frames)} frame(s) (selected sharpest per window)")
show_frames(sharp_frames)
```

<!--Result:-->
```
Output: 3 frame(s) (selected sharpest per window)
  Frame 0: 0.351
  Frame 1: 0.352
  Frame 2: 0.360
```

<details>
<summary>可视化辅助工具</summary>

```python session=qb fold no-result
import matplotlib
import matplotlib.pyplot as plt
import math

def plot_mosaic(frames, selected, path, cols=5):
    matplotlib.use('Agg')
    rows = math.ceil(len(frames) / cols)
    aspect = frames[0].width / frames[0].height
    fig_w, fig_h = 12, 12 * rows / (cols * aspect)

    fig, axes = plt.subplots(rows, cols, figsize=(fig_w, fig_h))
    fig.patch.set_facecolor('black')
    for i, ax in enumerate(axes.flat):
        if i < len(frames):
            ax.imshow(frames[i].data)
            for spine in ax.spines.values():
                spine.set_color('lime' if frames[i] in selected else 'black')
                spine.set_linewidth(4 if frames[i] in selected else 0)
            ax.set_xticks([]); ax.set_yticks([])
        else:
            ax.axis('off')
    plt.subplots_adjust(wspace=0.02, hspace=0.02, left=0, right=1, top=1, bottom=0)
    plt.savefig(path, facecolor='black', dpi=100, bbox_inches='tight', pad_inches=0)
    plt.close()

def plot_sharpness(frames, selected, path):
    matplotlib.use('svg')
    plt.style.use('dark_background')
    sharpness = [f.sharpness for f in frames]
    selected_idx = [i for i, f in enumerate(frames) if f in selected]

    plt.figure(figsize=(10, 3))
    plt.plot(sharpness, 'o-', label='All frames', color='#b5e4f4', alpha=0.7)
    for i, idx in enumerate(selected_idx):
        plt.axvline(x=idx, color='lime', linestyle='--', label='Selected' if i == 0 else None)
    plt.xlabel('Frame'); plt.ylabel('Sharpness')
    plt.xticks(range(len(sharpness)))
    plt.legend(); plt.grid(alpha=0.3); plt.tight_layout()
    plt.savefig(path, transparent=True)
    plt.close()
```

</details>

可视化哪些帧被选中（绿色边框 = 该窗口中选为最清晰的帧）：

```python session=qb output=assets/frame_mosaic.jpg
plot_mosaic(input_frames, sharp_frames, '{output}')
```

<!--Result:-->
![output](assets/frame_mosaic.jpg)

```python session=qb output=assets/sharpness_graph.svg
plot_sharpness(input_frames, sharp_frames, '{output}')
```

<!--Result:-->
![output](assets/sharpness_graph.svg)

让我们请求更高的频率。

```python session=qb
sharp_frames = video_replay.stream(seek=5.0, duration=1.5, speed=1.0).pipe(
    sharpness_barrier(4.0),
    ops.to_list()
).run()

print(f"Output: {len(sharp_frames)} frame(s) (selected sharpest per window)")
show_frames(sharp_frames)
```

<!--Result:-->
```
Output: 6 frame(s) (selected sharpest per window)
  Frame 0: 0.351
  Frame 1: 0.348
  Frame 2: 0.346
  Frame 3: 0.352
  Frame 4: 0.360
  Frame 5: 0.329
```

```python session=qb output=assets/frame_mosaic2.jpg
plot_mosaic(input_frames, sharp_frames, '{output}')
```

<!--Result:-->
![output](assets/frame_mosaic2.jpg)


```python session=qb output=assets/sharpness_graph2.svg
plot_sharpness(input_frames, sharp_frames, '{output}')
```

<!--Result:-->
![output](assets/sharpness_graph2.svg)

可以看到系统正在尝试在请求的频率和可用质量之间取得平衡。

### 在相机模块中的用法

以下是在实际相机模块中的使用方式：

```python skip
from dimos.core.module import Module

class CameraModule(Module):
    frequency: float = 2.0  # 目标输出频率
    @rpc
    def start(self) -> None:
        stream = self.hardware.image_stream()

        if self.config.frequency > 0:
            stream = stream.pipe(sharpness_barrier(self.config.frequency))

        self._disposables.add(
            stream.subscribe(self.color_image.publish),
        )

```

### 清晰度的计算方式

清晰度分数（0.0 到 1.0）使用 Sobel（索贝尔）边缘检测计算：

来自 [`Image.py`](/dimos/msgs/sensor_msgs/Image.py)

```python session=qb
import cv2

# 获取一帧并展示计算过程
img = input_frames[10]
gray = img.to_grayscale()

# Sobel 梯度 - 使用 .data 获取底层的 numpy 数组
sx = cv2.Sobel(gray.data, cv2.CV_32F, 1, 0, ksize=5)
sy = cv2.Sobel(gray.data, cv2.CV_32F, 0, 1, ksize=5)
magnitude = cv2.magnitude(sx, sy)

print(f"Mean gradient magnitude: {magnitude.mean():.2f}")
print(f"Normalized sharpness:    {img.sharpness:.3f}")
```

<!--Result:-->
```
Mean gradient magnitude: 230.00
Normalized sharpness:    0.332
```

## 自定义质量函数

你可以将 `quality_barrier` 与任何质量指标一起使用：

```python session=qb
# 示例：按 "confidence" 字段选择
detections = [
    {"name": "cat", "confidence": 0.7},
    {"name": "dog", "confidence": 0.95},  # 最佳
    {"name": "bird", "confidence": 0.6},
]

result = rx.of(*detections).pipe(
    quality_barrier(lambda d: d["confidence"], target_frequency=2.0),
    ops.to_list(),
).run()

print(f"Selected: {result[0]['name']} (conf: {result[0]['confidence']})")
```

<!--Result:-->
```
Selected: dog (conf: 0.95)
```

## API 参考

### `quality_barrier(quality_func, target_frequency)`

RxPY 管道操作符，在每个时间窗口内选择质量最高的项。

| 参数                | 类型                   | 描述                                                  |
|--------------------|------------------------|------------------------------------------------------|
| `quality_func`     | `Callable[[T], float]` | 返回每个项的质量分数的函数                               |
| `target_frequency` | `float`                | 输出频率，单位为 Hz（例如 2.0 表示每秒 2 项）             |

**返回值：** 用于 `.pipe()` 的管道操作符

### `sharpness_barrier(target_frequency)`

图像的便捷包装器，使用 `image.sharpness` 作为质量函数。

| 参数                | 类型    | 描述                |
|--------------------|---------|---------------------|
| `target_frequency` | `float` | 输出频率，单位为 Hz  |

**返回值：** 用于 `.pipe()` 的管道操作符
