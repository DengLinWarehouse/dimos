# 高级流处理

> **前置知识：** 请先阅读 [ReactiveX 基础](/docs/usage/data_streams/reactivex.md) 了解 Observable 基础知识。

## 背压与硬件并行订阅者

在机器人领域，我们需要处理以自身速度产生数据的硬件——摄像头以 30fps 输出帧，无论你是否准备好。我们无法让摄像头减速。而且我们经常有多个消费者：一个模块需要每一帧用于录制，另一个运行缓慢的 ML（机器学习）推理，只需要最新帧。

**问题：** 快速的生产者可能压垮慢速的消费者，导致内存堆积或丢帧。我们可能有多个以不同速度运行的订阅者订阅同一硬件。


<details><summary>Pikchr</summary>

```pikchr fold output=assets/backpressure.svg
color = white
fill = none

Fast: box "Camera" "60 fps" rad 5px fit wid 130% ht 130%
arrow right 0.4in
Queue: box "queue" rad 5px fit wid 170% ht 170%
arrow right 0.4in
Slow: box "ML Model" "2 fps" rad 5px fit wid 130% ht 130%

text "items pile up!" at (Queue.x, Queue.y - 0.45in)
```

</details>

<!--Result:-->
![output](assets/backpressure.svg)


**解决方案：** `backpressure()`（背压）包装器通过以下方式处理此问题：

1. **共享源** - 摄像头只运行一次，所有订阅者共享数据流
2. **每个订阅者独立速度** - 快速订阅者获取每一帧，慢速订阅者在准备好时获取最新帧
3. **不阻塞** - 慢速订阅者不会阻塞源或其他订阅者

```python session=bp
import time
import reactivex as rx
from reactivex import operators as ops
from reactivex.scheduler import ThreadPoolScheduler
from dimos.utils.reactive import backpressure

# 这里需要这些脚手架代码。通常 DimOS 会自动处理。
scheduler = ThreadPoolScheduler(max_workers=4)

# 模拟快速数据源
source = rx.interval(0.05).pipe(ops.take(20))
safe = backpressure(source, scheduler=scheduler)

fast_results = []
slow_results = []

safe.subscribe(lambda x: fast_results.append(x))

def slow_handler(x):
    time.sleep(0.15)
    slow_results.append(x)

safe.subscribe(slow_handler)

time.sleep(1.5)
print(f"fast got {len(fast_results)} items: {fast_results[:5]}...")
print(f"slow got {len(slow_results)} items (skipped {len(fast_results) - len(slow_results)})")
scheduler.executor.shutdown(wait=True)
```

<!--Result:-->
```
fast got 20 items: [0, 1, 2, 3, 4]...
slow got 7 items (skipped 13)
```

### 工作原理


<details><summary>Pikchr</summary>

```pikchr fold output=assets/backpressure_solution.svg
color = white
fill = none
linewid = 0.3in

Source: box "Camera" "60 fps" rad 5px fit wid 170% ht 170%
arrow
Core: box "backpressure" rad 5px fit wid 170% ht 170%
arrow from Core.e right 0.3in then up 0.35in then right 0.3in
Fast: box "Fast Sub" rad 5px fit wid 170% ht 170%
arrow from Core.e right 0.3in then down 0.35in then right 0.3in
SlowPre: box "LATEST" rad 5px fit wid 170% ht 170%
arrow
Slow: box "Slow Sub" rad 5px fit wid 170% ht 170%
```

</details>

<!--Result:-->
![output](assets/backpressure_solution.svg)

`LATEST` 策略意味着：当慢速订阅者完成处理后，它获取最新的值，跳过在它处理期间到达的所有值。

### 在模块中的用法

大多数模块流提供经过背压处理的 observable。

```python session=bp
from dimos.core.module import Module
from dimos.core.stream import In
from dimos.msgs.sensor_msgs import Image

class MLModel(Module):
    color_image: In[Image]
    def start(self):
       # 无 reactivex，简单回调
       self.color_image.subscribe(...)
       # 经过背压处理
       self.color_image.observable().subscribe(...)
       # 未经背压处理 - 会导致队列堆积
       self.color_image.pure_observable().subscribe(...)


```

## 同步获取值

有时你不需要数据流，只想调用一个函数获取最新值。

如果你在处理循环中周期性地执行此操作，使用实际的 reactivex 管道会让你的代码更清晰、更安全。所以建议优先查看我们的 [reactivex 快速指南](/docs/usage/data_streams/reactivex.md) 和 [官方文档](https://rxpy.readthedocs.io/)。

（TODO 我们应该让这个示例真正可执行）

```python skip
    self.color_image.observable().pipe(
        # 从流中每 200ms 取最佳图像，
        # 确保我们用最高质量的帧进行检测
        quality_barrier(lambda x: x["quality"], target_frequency=0.2),

        # 将 Image 转换为 Person 检测结果
        ops.map(detect_person),

        # 将 Detection2D 转换为指向检测方向的 Twist
        ops.map(detection2d_to_twist),

        # 每 50ms 发出最新值，使我们的控制循环以 20hz 运行
        # 尽管检测运行在 200ms
        ops.sample(0.05),
    ).subscribe(self.twist.publish) # 将 Twist 从模块发送出去
```


如果你仍然想切换到同步获取，我们提供两种方法：`getter_hot()` 和 `getter_cold()`

| | `getter_hot()` | `getter_cold()` |
|------------------|--------------------------------|----------------------------------|
| **订阅方式** | 后台保持活跃 | 每次调用重新订阅 |
| **读取速度** | 即时（值已缓存） | 较慢（等待值到达） |
| **资源占用** | 保持连接打开 | 每次调用打开/关闭 |
| **使用场景** | 频繁读取，需要最新值 | 偶尔读取，节省资源 |

<details>
<summary>diagram source</summary>

```pikchr fold output=assets/getter_hot_cold.svg
color = white
fill = none

H_Title: box "getter_hot()" rad 5px fit wid 170% ht 170%

Sub: box "subscribe" rad 5px fit wid 170% ht 170% with .n at H_Title.s + (0, -0.5in)
arrow from H_Title.s to Sub.n
arrow right from Sub.e
Cache: box "Cache" rad 5px fit wid 170% ht 170%

# blocking box around subscribe->cache (one-time setup)
Blk0: box dashed color 0x5c9ff0 with .nw at Sub.nw + (-0.1in, 0.25in) wid (Cache.e.x - Sub.w.x + 0.2in) ht 0.7in rad 5px
text "blocking" italic with .n at Blk0.n + (0, -0.05in)

arrow right from Cache.e
Getter: box "getter" rad 5px fit wid 170% ht 170%

arrow from Getter.e right 0.3in then down 0.25in then right 0.2in
G1: box invis "call()" color 0x8cbdf2 fit wid 150%
arrow right 0.4in from G1.e
box invis "instant" fit wid 150%

arrow from Getter.e right 0.3in then down 0.7in then right 0.2in
G2: box invis "call()" color 0x8cbdf2 fit wid 150%
arrow right 0.4in from G2.e
box invis "instant" fit wid 150%

text "always subscribed" italic with .n at Blk0.s + (0, -0.1in)


# === getter_cold section ===
C_Title: box "getter_cold()" rad 5px fit wid 170% ht 170% with .nw at H_Title.sw + (0, -1.6in)

arrow down 0.3in from C_Title.s
ColdGetter: box "getter" rad 5px fit wid 170% ht 170%

# Branch to first call
arrow from ColdGetter.e right 0.3in then down 0.3in then right 0.2in
Cold1: box invis "call()" color 0x8cbdf2 fit wid 150%
arrow right 0.4in from Cold1.e
Sub1: box invis "subscribe" fit wid 150%
arrow right 0.4in from Sub1.e
Wait1: box invis "wait" fit wid 150%
arrow right 0.4in from Wait1.e
Val1: box invis "value" fit wid 150%
arrow right 0.4in from Val1.e
Disp1: box invis "dispose  " fit wid 150%

# blocking box around first row
Blk1: box dashed color 0x5c9ff0 with .nw at Cold1.nw + (-0.1in, 0.25in) wid (Disp1.e.x - Cold1.w.x + 0.2in) ht 0.7in rad 5px
text "blocking" italic with .n at Blk1.n + (0, -0.05in)

# Branch to second call
arrow from ColdGetter.e right 0.3in then down 1.2in then right 0.2in
Cold2: box invis "call()" color 0x8cbdf2 fit wid 150%
arrow right 0.4in from Cold2.e
Sub2: box invis "subscribe" fit wid 150%
arrow right 0.4in from Sub2.e
Wait2: box invis "wait" fit wid 150%
arrow right 0.4in from Wait2.e
Val2: box invis "value" fit wid 150%
arrow right 0.4in from Val2.e
Disp2: box invis "dispose  " fit wid 150%

# blocking box around second row
Blk2: box dashed color 0x5c9ff0 with .nw at Cold2.nw + (-0.1in, 0.25in) wid (Disp2.e.x - Cold2.w.x + 0.2in) ht 0.7in rad 5px
text "blocking" italic with .n at Blk2.n + (0, -0.05in)
```

</details>

<!--Result:-->
![output](assets/getter_hot_cold.svg)


**优先使用 `getter_cold()`**，前提是你可以承受等待且预热开销不大。它更简单（无需清理），也不会占用资源。只有当你需要即时读取或源的启动开销很大时，才使用 `getter_hot()`。

### `getter_hot()` - 后台订阅，即时读取

立即订阅并在后台持续更新。每次调用即时返回缓存的最新值。

```python session=sync
import time
import reactivex as rx
from reactivex import operators as ops
from dimos.utils.reactive import getter_hot

source = rx.interval(0.1).pipe(ops.take(10))

get_val = getter_hot(source, timeout=5.0) # 阻塞直到收到第一条消息，超时 5 秒
# 也可以不阻塞（但 get_val() 可能返回 None）
# get_val = getter_hot(source, nonblocking=True)

print("first call:", get_val())  # 即时 - 值已经存在
time.sleep(0.35)
print("after 350ms:", get_val())  # 即时 - 返回缓存的最新值
time.sleep(0.35)
print("after 700ms:", get_val())

get_val.dispose()  # 别忘了清理！
```

<!--Result:-->
```
first call: 0
after 350ms: 3
after 700ms: 6
```

### `getter_cold()` - 每次调用重新订阅

每次调用创建新的订阅，等待一个值，然后清理。较慢但不占用资源：

```python session=sync
from dimos.utils.reactive import getter_cold

source = rx.of(0, 1, 2, 3, 4)
get_val = getter_cold(source, timeout=5.0)

# 每次调用创建新订阅，获取第一个值
print("call 1:", get_val())  # 订阅，获取 0，dispose
print("call 2:", get_val())  # 再次订阅，获取 0，dispose
print("call 3:", get_val())  # 再次订阅，获取 0，dispose
```

<!--Result:-->
```
call 1: 0
call 2: 0
call 3: 0
```
