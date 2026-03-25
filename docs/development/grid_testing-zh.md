# 网格测试策略

网格测试（Grid tests）利用 pytest 的 parametrize 特性，在多个实现或配置上运行相同的测试逻辑。

## Case 类型模式

定义一个 `Case` 数据类，用于保存针对特定实现运行测试所需的所有信息：

```python
from collections.abc import Callable, Iterator
from contextlib import AbstractContextManager
from dataclasses import dataclass, field
from typing import Any, Generic

@dataclass
class Case(Generic[TopicT, MsgT]):
    name: str  # 用于 pytest 标识
    pubsub_context: Callable[[], AbstractContextManager[...]]  # 上下文管理器工厂
    topic_values: list[tuple[TopicT, MsgT]]  # 预生成的测试数据（始终为 3 对）
    tags: set[str] = field(default_factory=set)  # 用于过滤的能力标签

    def __iter__(self) -> Iterator[Any]:
        """使 Case 支持 pytest.parametrize 的解包操作。"""
        return iter((self.pubsub_context, self.topic_values))
```

## 能力标签

使用标签来标识每个实现支持的功能特性：

```python
testcases = [
    Case(
        name="lcm_typed",
        pubsub_context=lcm_typed_context,
        topic_values=[...],
        tags={"all", "glob", "regex"},  # LCM 支持所有匹配模式
    ),
    Case(
        name="shm_pickle",
        pubsub_context=shm_context,
        topic_values=[...],
        tags={"all"},  # SharedMemory 仅支持 subscribe_all
    ),
]
```

## 按能力过滤的测试列表

为每种能力构建单独的列表，配合 parametrize 使用：

```python
all_cases = [c for c in testcases if "all" in c.tags]
glob_cases = [c for c in testcases if "glob" in c.tags]
regex_cases = [c for c in testcases if "regex" in c.tags]
```

## 测试函数

在 parametrize 装饰器中使用过滤后的列表：

```python
@pytest.mark.parametrize("case", all_cases, ids=lambda c: c.name)
def test_subscribe_all(case: Case) -> None:
    with case.pubsub_context() as pubsub:
        # 使用 case.topic_values 的测试逻辑
        ...

@pytest.mark.parametrize("case", glob_cases, ids=lambda c: c.name)
def test_subscribe_glob(case: Case) -> None:
    if not glob_cases:
        pytest.skip("no implementations support glob")
    with case.pubsub_context() as pubsub:
        ...
```

## 上下文管理器

每个实现提供一个上下文管理器工厂：

```python
@contextmanager
def lcm_typed_context() -> Generator[LCM, None, None]:
    lcm = LCM()
    lcm.start()
    yield lcm
    lcm.stop()
```

## 测试数据指南

- 始终提供恰好 3 对 topic/value 数据以保持一致性
- 对于类型化的实现，每个 topic 使用不同类型以验证类型处理能力
- 对于字节类型的实现，使用简单可区分的字节字符串

```python
# 类型化测试数据 - 每个 topic 使用不同类型
typed_topic_values = [
    (Topic("/sensor/position", Vector3), Vector3(1, 2, 3)),
    (Topic("/sensor/orientation", Quaternion), Quaternion(0, 0, 0, 1)),
    (Topic("/robot/pose", Pose), Pose(...)),
]

# 字节测试数据
bytes_topic_values = [
    (Topic("/topic1"), b"msg1"),
    (Topic("/topic2"), b"msg2"),
    (Topic("/topic3"), b"msg3"),
]
```

## 示例

- `dimos/protocol/pubsub/test_spec.py` - 基本的发布/订阅操作
- `dimos/protocol/pubsub/test_subscribe_all.py` - 模式订阅
- `dimos/protocol/pubsub/benchmark/testdata.py` - 基准测试用例
