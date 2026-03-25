# 配置

Dimos 提供了一个 `Configurable` 基类。参见 [`service/spec.py`](/dimos/protocol/service/spec.py#L22)。

它允许使用 dataclass 来指定每个模块的配置结构和默认值。

```python
from dimos.protocol.service import Configurable
from rich import print
from dataclasses import dataclass

@dataclass
class Config():
    x: int = 3
    hello: str = "world"

class MyClass(Configurable):
    default_config = Config
    config: Config
    def __init__(self, **kwargs) -> None:
        super().__init__(**kwargs)

myclass1 = MyClass()
print(myclass1.config)

# 可以轻松覆盖
myclass2 = MyClass(hello="override")
print(myclass2.config)

# 对于未指定的键会抛出错误
try:
    myclass3 = MyClass(something="else")
except TypeError as e:
    print(f"Error: {e}")


```

<!--Result:-->
```
Config(x=3, hello='world')
Config(x=3, hello='override')
Error: Config.__init__() got an unexpected keyword argument 'something'
```

# 可配置模块

[模块](/docs/usage/modules.md)继承自 `Configurable`，因此以上所有内容均适用。模块配置应继承自 `ModuleConfig`（[`core/module.py`](/dimos/core/module.py#L40)），其中包含所有模块的共享配置，如传输协议、frame ID 等。

```python
from dataclasses import dataclass
from dimos.core.core import rpc
from dimos.core.module import Module, ModuleConfig
from dimos.core.stream import In, Out
from rich import print

@dataclass
class Config(ModuleConfig):
    frame_id: str = "world"
    publish_interval: float = 0
    voxel_size: float = 0.05
    device: str = "CUDA:0"

class MyModule(Module):
    default_config = Config
    config: Config

    def __init__(self, **kwargs) -> None:
        super().__init__(**kwargs)
        print(self.config)


myModule = MyModule(frame_id="frame_id_override", device="CPU")

# 在生产环境中，请使用 dimos.deploy()：
# myModule = dimos.deploy(MyModule, frame_id="frame_id_override")


```

<!--Result:-->
```
Config(
    rpc_transport=<class 'dimos.protocol.rpc.pubsubrpc.LCMRPC'>,
    tf_transport=<class 'dimos.protocol.tf.tf.LCMTF'>,
    frame_id_prefix=None,
    frame_id='frame_id_override',
    publish_interval=0,
    voxel_size=0.05,
    device='CPU'
)
```
