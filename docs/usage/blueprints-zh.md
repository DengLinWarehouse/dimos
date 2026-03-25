# 蓝图

蓝图（`_BlueprintAtom`）是用于描述如何初始化一个 `Module` 的指令。

通常你不会只运行一个模块，因此多个蓝图会通过 `Blueprint` 一起管理。

你可以从一个模块（比如 `ConnectionModule`）创建一个 `Blueprint`：

```python session=blueprint-ex1
from dimos.core.blueprints import Blueprint
from dimos.core.core import rpc
from dimos.core.module import Module

class ConnectionModule(Module):
    def __init__(self, arg1, arg2, kwarg='value') -> None:
        super().__init__()

blueprint = Blueprint.create(ConnectionModule, 'arg1', 'arg2', kwarg='value')
```

同样的效果可以用更简洁的方式实现：

```python session=blueprint-ex1
connection = ConnectionModule.blueprint
```

然后你可以这样创建蓝图：

```python session=blueprint-ex1
blueprint = connection('arg1', 'arg2', kwarg='value')
```

## 链接蓝图

你可以使用 `autoconnect` 将多个蓝图链接在一起：

```python session=blueprint-ex1
from dimos.core.blueprints import autoconnect

class Module1(Module):
    def __init__(self, arg1) -> None:
        super().__init__()

class Module2(Module):
    ...

class Module3(Module):
    ...

module1 = Module1.blueprint
module2 = Module2.blueprint
module3 = Module3.blueprint

blueprint = autoconnect(
    module1(),
    module2(),
    module3(),
)
```

`blueprint` 本身也是一个 `Blueprint`，所以你可以继续将它与其他模块链接：

```python session=blueprint-ex1
class Module4(Module):
    ...

class Module5(Module):
    ...

module4 = Module4.blueprint
module5 = Module5.blueprint

expanded_blueprint = autoconnect(
    blueprint,
    module4(),
    module5(),
)
```

蓝图是冻结的数据类，`autoconnect()` 总是构造一个展开后的新蓝图，因此你无需担心修改一个蓝图会影响到另一个。

### 重复模块处理

如果同一个模块在 `autoconnect` 中出现多次，**后面的蓝图会覆盖前面的**：

```python session=blueprint-ex1
blueprint = autoconnect(
    module1(arg1=1),
    module2(),
    module1(arg1=2),  # 使用这个，第一个被丢弃
)
```

这样你就可以从一个蓝图"继承"，同时覆盖需要修改的部分。

## 传输层如何链接

假设你有以下代码：

```python session=blueprint-ex1
from functools import partial

from dimos.core.blueprints import Blueprint, autoconnect
from dimos.core.core import rpc
from dimos.core.module import Module
from dimos.core.stream import Out, In
from dimos.msgs.sensor_msgs import Image

class ModuleA(Module):
    image: Out[Image]
    start_explore: Out[bool]

class ModuleB(Module):
    image: In[Image]
    begin_explore: In[bool]

module_a = partial(Blueprint.create, ModuleA)
module_b = partial(Blueprint.create, ModuleB)

autoconnect(module_a(), module_b())
```

连接基于 `(属性名, 对象类型)` 进行匹配。在这个例子中，`('image', Image)` 会在两个模块之间建立连接，但 `begin_explore` 不会链接到 `start_explore`。

## Topic 名称

默认情况下，属性名被用于生成 topic 名称。因此对于 `image`，topic 将是 `/image`。

属性名仅在唯一时使用。如果两个模块有相同的属性名但类型不同，那么两者都会获得一个随机 topic，例如 `/SGVsbG8sIFdvcmxkI`。

如果你不喜欢这个名称，可以按照下一节的方式进行覆盖。

## 使用哪种传输协议？

默认情况下，如果对象支持 `lcm_encode`，则使用 `LCMTransport`。如果不支持，则使用 `pLCMTransport`（即"pickled LCM"，基于 pickle 序列化的 LCM）。

你可以使用 `transports` 方法覆盖传输协议。它返回一个设置了覆盖项的新蓝图。

```python session=blueprint-ex1
from dimos.core.transport import pSHMTransport, pLCMTransport

base_blueprint = autoconnect(
    module1(arg1=1),
    module2(),
)
expanded_blueprint = autoconnect(
    base_blueprint,
    module4(),
    module5(),
)
base_blueprint = base_blueprint.transports({
    ("image", Image): pSHMTransport(
        "/go2/color_image", default_capacity=1920 * 1080 * 3,  # 1920x1080 frame x 3 (RGB) x uint8
    ),
    ("start_explore", bool): pLCMTransport("/start_explore"),
})
```

注意：`expanded_blueprint` 不会获得传输层的覆盖设置，因为它是从 `base_blueprint` 的初始值创建的，而非第二个。

## 重映射连接

有时你需要重命名一个连接以匹配其他模块的期望。你可以使用 `remappings` 来重命名模块连接：

```python session=blueprint-ex2
from dimos.core.blueprints import autoconnect
from dimos.core.core import rpc
from dimos.core.module import Module
from dimos.core.stream import Out, In
from dimos.msgs.sensor_msgs import Image

class ConnectionModule(Module):
    color_image: Out[Image]  # 输出到 'color_image'

class ProcessingModule(Module):
    rgb_image: In[Image]  # 期望在 'rgb_image' 上接收输入

# 不使用重映射的话，这两者不会自动连接
# 使用重映射后，color_image 被重命名为 rgb_image
blueprint = (
    autoconnect(
        ConnectionModule.blueprint(),
        ProcessingModule.blueprint(),
    )
    .remappings([
        (ConnectionModule, 'color_image', 'rgb_image'),
    ])
)
```

重映射后：
- `ConnectionModule` 的 `color_image` 输出被视为 `rgb_image`
- 它会自动连接到任何具有 `Image` 类型 `rgb_image` 输入的模块
- Topic 名称变为 `/rgb_image` 而非 `/color_image`

如果你想覆盖 topic，仍然需要手动操作：

```python session=blueprint-ex2
from dimos.core.transport import LCMTransport
blueprint.remappings([
    (ConnectionModule, 'color_image', 'rgb_image'),
]).transports({
    ("rgb_image", Image): LCMTransport("/custom/rgb/image", Image),
})
```

## 覆盖全局配置

每个模块可以选择性地在 `__init__` 中接受一个 `cfg` 选项作为全局配置。例如：

```python session=blueprint-ex3
from dimos.core.core import rpc
from dimos.core.module import Module
from dimos.core.global_config import GlobalConfig

class ModuleA(Module):

    def __init__(self, cfg: GlobalConfig | None = None):
        self._global_config: GlobalConfig = cfg
        ...
```

配置通常从 .env 文件或环境变量中获取。但你可以为特定蓝图显式覆盖配置值：

```python session=blueprint-ex3
blueprint = ModuleA.blueprint().global_config(n_workers=8)
```

## 调用其他模块的方法

假设你有以下代码：

```python session=blueprint-ex3
from dimos.core.core import rpc
from dimos.core.module import Module

class Drone(Module):

    @rpc
    def get_time(self) -> str:
        ...

class HelperModule(Module):
    def set_alarm_clock(self) -> None:
        ...
```

你想在 `ModuleB.request_the_time` 中调用 `ModuleA.get_time`。

为此，你可以请求一个模块引用。

```python session=blueprint-ex3
from dimos.core.core import rpc
from dimos.core.module import Module

class HelperModule(Module):
    drone_module: Drone

    def set_alarm_clock(self) -> None:
        print(self.drone_module.get_time_rpc())
```

但如果我们希望 `HelperModule` 不仅仅适用于 `Drone` 呢？为此我们可以使用 spec（规范）。

```python session=blueprint-ex3
from dimos.spec.utils import Spec
from typing import Protocol

class Drone(Module):
    def get_time(self) -> str:
        return "1:00 PM"

class Car(Module):
    def get_time(self) -> str:
        return "2:00 PM"

# 你的 Spec
class AnyModuleWithGetTime(Spec, Protocol):
    def get_time(self) -> str: ...

class ModuleB(Module):
    device: AnyModuleWithGetTime

    def request_the_time(self) -> None:
        # autoconnect() 会自动找到任何具有 get_time() 方法的模块
        print(self.device.get_time())
```

## 定义技能

技能（Skills）是 `Module` 上使用 `@skill` 装饰器的方法。智能体在启动时会自动发现所有已启动模块的技能。

```python session=blueprint-ex4
from dimos.core.core import rpc
from dimos.core.module import Module
from dimos.agents.annotation import skill
from dimos.core.global_config import GlobalConfig

class SomeSkill(Module):

    @skill
    def some_skill(self) -> str:
        """供 LLM 使用的技能描述。"""
        return "result"
```

## 构建

要构建一个蓝图，只需调用：

```python session=blueprint-ex4
module_coordinator = SomeSkill.blueprint().build(global_config=GlobalConfig())
```

这将返回一个 `ModuleCoordinator` 实例，用于管理所有已部署的模块。

### 运行与关闭

你可以阻塞线程直到退出：

```python session=blueprint-ex4
module_coordinator.loop()
```

这将等待 Ctrl+C，然后自动停止所有模块并清理资源。
