# ReactiveX (RxPY) 快速参考

RxPY 提供可组合的异步数据流。本指南专注于此代码库中的常见模式。

## 快速入门：使用 Observable

给定一个返回 `Observable`（可观察对象）的函数，以下是使用方法：

```python session=rx
import reactivex as rx
from reactivex import operators as ops

# 创建一个发出 0,1,2,3,4 的 observable
source = rx.of(0, 1, 2, 3, 4)

# 订阅并打印每个值
received = []
source.subscribe(lambda x: received.append(x))
print("received:", received)
```

<!--Result:-->
```
received: [0, 1, 2, 3, 4]
```

## `.pipe()` 模式

使用 `.pipe()` 链式调用操作符：

```python session=rx
# 转换值：乘以 2，然后过滤大于 4 的值
result = []

# 构建另一个 observable。它是被动的，直到调用 `subscribe` 才会执行。
observable = source.pipe(
    ops.map(lambda x: x * 2),
    ops.filter(lambda x: x > 4),
)

observable.subscribe(lambda x: result.append(x))

print("transformed:", result)
```

<!--Result:-->
```
transformed: [6, 8]
```

## 常用操作符

### 转换：`map`

```python session=rx
rx.of(1, 2, 3).pipe(
    ops.map(lambda x: f"item_{x}")
).subscribe(print)
```

<!--Result:-->
```
item_1
item_2
item_3
<reactivex.disposable.disposable.Disposable object at 0x7fcedec40b90>
```

### 过滤：`filter`

```python session=rx
rx.of(1, 2, 3, 4, 5).pipe(
    ops.filter(lambda x: x % 2 == 0)
).subscribe(print)
```

<!--Result:-->
```
2
4
<reactivex.disposable.disposable.Disposable object at 0x7fcedec40c50>
```

### 限制发射数量：`take`

```python session=rx
rx.of(1, 2, 3, 4, 5).pipe(
    ops.take(3)
).subscribe(print)
```

<!--Result:-->
```
1
2
3
<reactivex.disposable.disposable.Disposable object at 0x7fcedec40a40>
```

### 展平嵌套的 observable：`flat_map`

```python session=rx
# 对每个输入，发出多个值
rx.of(1, 2).pipe(
    ops.flat_map(lambda x: rx.of(x, x * 10, x * 100))
).subscribe(print)
```

<!--Result:-->
```
1
10
100
2
20
200
<reactivex.disposable.disposable.Disposable object at 0x7fcedec41a60>
```

## 速率限制

### `sample(interval)` - 每 N 秒发出最新值

在每个时间间隔取最近的值。适用于需要最新数据的连续流。

```python session=rx
# 使用阻塞式的 .run() 来正确收集结果
results = rx.interval(0.05).pipe(
    ops.take(10),
    ops.sample(0.2),
    ops.to_list(),
).run()
print("sample() got:", results)
```

<!--Result:-->
```
sample() got: [2, 6, 9]
```

### `throttle_first(interval)` - 先发出第一个，然后阻塞 N 秒

取第一个值，然后在间隔时间内忽略后续值。适用于用户输入去抖。

```python session=rx
results = rx.interval(0.05).pipe(
    ops.take(10),
    ops.throttle_first(0.15),
    ops.to_list(),
).run()
print("throttle_first() got:", results)
```

<!--Result:-->
```
throttle_first() got: [0, 3, 6, 9]
```

### `sample` 和 `throttle_first` 的区别

```python session=rx
# sample：在每个间隔时间点取最新值
# throttle_first：取第一个值然后阻塞

# 以每 50ms 快速发出 (0,1,2,3,4,5,6,7,8,9)：
# sample(0.2s)          -> 在 200ms、400ms 时间点取值 -> [2, 6, 9]
# throttle_first(0.15s) -> 取 0，阻塞，然后取 3，阻塞，然后取 6... -> [0,3,6,9]
print("sample: latest value at each tick")
print("throttle_first: first value, then block")
```

<!--Result:-->
```
sample: latest value at each tick
throttle_first: first value, then block
```


## 什么是 Observable？

Observable 类似于列表，但它不是一次性持有所有值，而是随时间推移逐个产生值。

| | 列表 | 迭代器 | Observable |
|-------------|----------------------|----------------------|------------------|
| **值** | 当前全部存在 | 按需生成 | 随时间到达 |
| **控制** | 你来拉取（`for x in`） | 你来拉取（`next()`） | 推送给你 |
| **大小** | 有限 | 可以无限 | 可以无限 |
| **异步** | 否 | 是（使用 asyncio） | 是 |
| **取消** | 不适用 | 停止调用 `next()` | `.dispose()` |

与迭代器的关键区别：使用 Observable 时，**你无法控制值何时到达**。摄像头以 30fps 输出帧，无论你是否准备好。迭代器则会等你调用 `next()`。

**Observable 是惰性的。** Observable 只是对要执行的工作的描述——在你调用 `.subscribe()` 之前它什么都不做。调用时它才"启动"并开始产生值。

这意味着你可以构建复杂的管道，传递它们，在有人订阅之前什么都不会发生。

**Observable 可以告诉你三件事：**

1. **"这是一个值"**（`on_next`）- 新值到达
2. **"出错了"**（`on_error`）- 发生错误，流停止
3. **"我完成了"**（`on_completed`）- 不会再有更多值

**基本模式：**

```
observable.subscribe(what_to_do_with_each_value)
```

就是这样。你创建或接收一个 Observable，然后订阅以开始接收值。

当你订阅时，数据通过管道流动：

<details>
<summary>diagram source</summary>

```pikchr fold output=assets/observable_flow.svg
color = white
fill = none

Obs: box "observable" rad 5px fit wid 170% ht 170%
arrow right 0.3in
Pipe: box ".pipe(ops)" rad 5px fit wid 170% ht 170%
arrow right 0.3in
Sub: box ".subscribe()" rad 5px fit wid 170% ht 170%
arrow right 0.3in
Handler: box "callback" rad 5px fit wid 170% ht 170%
```

</details>

<!--Result:-->
![output](assets/observable_flow.svg)


**关键特性：Observable 是惰性的。** 在你调用 `.subscribe()` 之前什么都不会发生。这意味着你可以构建复杂的管道而不执行任何工作，准备好后再启动数据流。

以下是包含所有三个回调的完整 subscribe 签名：

```python session=rx
rx.of(1, 2, 3).subscribe(
    on_next=lambda x: print(f"value: {x}"),
    on_error=lambda e: print(f"error: {e}"),
    on_completed=lambda: print("done")
)
```

<!--Result:-->
```
value: 1
value: 2
value: 3
done
<reactivex.disposable.disposable.Disposable object at 0x7fcedec42d20>
```

## Disposable：取消订阅

当你订阅时，会得到一个 `Disposable`（可处置对象）。这是你的"取消按钮"：

```python session=rx
import reactivex as rx

source = rx.interval(0.1)  # 每 100ms 发出 0, 1, 2, ... 永不停止
subscription = source.subscribe(lambda x: print(x))

# 之后，当你完成时：
subscription.dispose()  # 停止接收值，清理资源
print("disposed")
```

<!--Result:-->
```
disposed
```

**为什么这很重要？**

- Observable 可以是无限的（传感器数据流、websocket、定时器）
- 不进行 dispose，你会泄漏内存并永远持续处理值
- dispose 还会清理 Observable 打开的所有资源（连接、文件句柄等）

**经验法则：** 每次订阅时都保存 disposable，因为你需要在某个时刻通过调用 `disposable.dispose()` 来取消订阅。

**在 dimos 模块中：** 每个 `Module` 都有一个 `self._disposables`（一个 `CompositeDisposable`），在模块关闭时自动处置所有内容：

```python session=rx
import time
from dimos.core.module import Module

class MyModule(Module):
    def start(self):
        source = rx.interval(0.05)
        self._disposables.add(source.subscribe(lambda x: print(f"got {x}")))

module = MyModule()
module.start()
time.sleep(0.25)

# 取消所有订阅
module.stop()
```

<!--Result:-->
```
got 0
got 1
got 2
got 3
got 4
```

## 创建 Observable

API 中有两种常见的回调模式。请使用对应的辅助函数：

| 模式 | 示例 | 辅助函数 |
|---------|---------|--------|
| 使用同一回调进行注册/注销 | `sensor.register(cb)` / `sensor.unregister(cb)` | `callback_to_observable` |
| 订阅返回取消订阅函数 | `unsub = pubsub.subscribe(cb)` | `to_observable` |

### 从注册/注销 API 创建

当 API 有分别的 register 和 unregister 函数且接受相同的回调引用时，使用 `callback_to_observable`：

```python session=create
import reactivex as rx
from reactivex import operators as ops
from dimos.utils.reactive import callback_to_observable

class MockSensor:
    def __init__(self):
        self._callbacks = []
    def register(self, cb):
        self._callbacks.append(cb)
    def unregister(self, cb):
        self._callbacks.remove(cb)
    def emit(self, value):
        for cb in self._callbacks:
            cb(value)

sensor = MockSensor()

obs = callback_to_observable(
    start=sensor.register,
    stop=sensor.unregister
)

received = []
sub = obs.subscribe(lambda x: received.append(x))

sensor.emit("reading_1")
sensor.emit("reading_2")
print("received:", received)

sub.dispose()
print("callbacks after dispose:", len(sensor._callbacks))
```

<!--Result:-->
```
received: ['reading_1', 'reading_2']
callbacks after dispose: 0
```

### 从订阅返回取消函数的 API 创建

当 subscribe 函数返回一个取消订阅的可调用对象时，使用 `to_observable`：

```python session=create
from dimos.utils.reactive import to_observable

class MockPubSub:
    def __init__(self):
        self._callbacks = []
    def subscribe(self, cb):
        self._callbacks.append(cb)
        return lambda: self._callbacks.remove(cb)  # 返回取消订阅函数
    def publish(self, value):
        for cb in self._callbacks:
            cb(value)

pubsub = MockPubSub()

obs = to_observable(pubsub.subscribe)

received = []
sub = obs.subscribe(lambda x: received.append(x))

pubsub.publish("msg_1")
pubsub.publish("msg_2")
print("received:", received)

sub.dispose()
print("callbacks after dispose:", len(pubsub._callbacks))
```

<!--Result:-->
```
received: ['msg_1', 'msg_2']
callbacks after dispose: 0
```

### 使用 `rx.create` 从零创建

```python session=create
from reactivex.disposable import Disposable

def custom_subscribe(observer, scheduler=None):
    observer.on_next("first")
    observer.on_next("second")
    observer.on_completed()
    return Disposable(lambda: print("cleaned up"))

obs = rx.create(custom_subscribe)

results = []
obs.subscribe(
    on_next=lambda x: results.append(x),
    on_completed=lambda: results.append("DONE")
)
print("results:", results)
```

<!--Result:-->
```
cleaned up
results: ['first', 'second', 'DONE']
```

## CompositeDisposable

如前所述，我们可以在完成后 dispose 订阅以防止泄漏：

```python session=dispose
import time
import reactivex as rx
from reactivex import operators as ops

source = rx.interval(0.1).pipe(ops.take(100))
received = []

subscription = source.subscribe(lambda x: received.append(x))
time.sleep(0.25)
subscription.dispose()
time.sleep(0.2)

print(f"received {len(received)} items before dispose")
```

<!--Result:-->
```
received 2 items before dispose
```

对于多个订阅，使用 `CompositeDisposable`（组合可处置对象）：

```python session=dispose
from reactivex.disposable import CompositeDisposable

disposables = CompositeDisposable()

s1 = rx.of(1,2,3).subscribe(lambda x: None)
s2 = rx.of(4,5,6).subscribe(lambda x: None)

disposables.add(s1)
disposables.add(s2)

print("subscriptions:", len(disposables))
disposables.dispose()
print("after dispose:", disposables.is_disposed)
```

<!--Result:-->
```
subscriptions: 2
after dispose: True
```

## 参考

| 操作符 | 用途 | 示例 |
|-----------------------|------------------------------------------|---------------------------------------|
| `map(fn)` | 转换每个值 | `ops.map(lambda x: x * 2)` |
| `filter(pred)` | 保留匹配谓词的值 | `ops.filter(lambda x: x > 0)` |
| `take(n)` | 取前 n 个值 | `ops.take(10)` |
| `first()` | 只取第一个值 | `ops.first()` |
| `sample(sec)` | 每个间隔发出最新值 | `ops.sample(0.5)` |
| `throttle_first(sec)` | 发出第一个，然后阻塞一段时间 | `ops.throttle_first(0.5)` |
| `flat_map(fn)` | 映射 + 展平嵌套的 observable | `ops.flat_map(lambda x: rx.of(x, x))` |
| `observe_on(sched)` | 切换调度器 | `ops.observe_on(pool_scheduler)` |
| `replay(n)` | 为延迟订阅者缓存最后 n 个值 | `ops.replay(buffer_size=1)` |
| `timeout(sec)` | 超时内无值则报错 | `ops.timeout(5.0)` |

更多完整的操作符参考请参见 [RxPY 文档](https://rxpy.readthedocs.io/)。
