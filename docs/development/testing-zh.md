# 测试

进行开发时，你应该安装所有依赖项，以便测试能够正常访问它们。

```bash
uv sync --all-extras --no-extra dds
```

## 测试类型

通常，根据测试目标的不同，有以下几种类型的测试：

| 类型 | 描述 | Mock（模拟） | 速度 |
|------|-------------|---------|-------|
| Unit（单元测试） | 测试一小段独立的代码 | 模拟所有外部系统 | 非常快 |
| Integration（集成测试） | 测试多个代码单元之间的集成 | 模拟大部分外部系统 | 有快有慢 |
| Functional（功能测试） | 测试某个特定的期望功能 | 模拟部分外部系统 | 有快有慢 |
| End-to-end（端到端测试） | 从用户视角测试整个系统 | 不模拟 | 非常慢 |

关于单元测试、集成测试和功能测试的界限划分，常有争议且很少有实际意义。

与其在测试分类上浪费时间，不如按照测试的使用方式来区分：

| 测试分组 | 运行时机 | 典型用法 |
|------------|-------------|---------------|
| **快速测试** | 每次代码修改后 | 通常配合文件系统监听器使用，每次保存文件时自动重新运行测试 |
| **慢速测试** | 隔一段时间运行一次以确保没有引入问题 | 可能每次提交都运行，但至少在发布 PR 之前必须运行 |

循环运行测试的目的是获得即时反馈。循环越快，越容易定位问题，因为问题来源就是你刚刚修改的那一小段代码。

在 DimOS 中，慢速测试使用 `@pytest.mark.slow` 标记，其余均为快速测试。

## 使用方法

### 快速测试

运行快速测试：

```bash
./bin/pytest-fast
```

等同于：

```bash
pytest dimos
```

`pyproject.toml` 中默认的 `addopts` 包含一个 `-m` 过滤器，排除了 `slow`/`mujoco`/`tool` 标记。因此直接运行 `pytest dimos` 只会执行快速测试。

### 慢速测试

运行慢速测试：

```bash
./bin/pytest-slow
```

（这只是 `pytest -m 'not (tool or mujoco)' dimos` 的快捷方式。即同时运行快速测试和慢速测试，但不运行 `tool` 或 `mujoco` 标记的测试。）

在编写或调试特定的慢速测试时，可以手动覆盖 `-m` 参数来运行它：

```bash
pytest -m slow dimos/path/to/test_something.py
```

## 编写测试

测试文件放在被测代码的旁边。如果你有 `dimos/core/pubsub.py`，其测试文件应放在 `dimos/core/test_pubsub.py`。

编写测试时，你可能想限制只运行当前正在编写的测试：

```bash
pytest -sv dimos/core/test_my_code.py
```

### Fixture（测试夹具）

pytest 的 fixture 对于确保测试失败不影响其他测试非常有用。

当有需要在测试结束时清理的资源（断开连接、关闭、删除临时文件等）时，你应该使用 fixture。

简单示例代码：

```python
@pytest.fixture
def arm():
    arm = RobotArm(device="/dev/ttyUSB0")
    arm.connect()
    yield arm
    arm.disconnect()

def test_arm_moves_to_position(arm):
    arm.move_to(x=0.5, y=0.3, z=0.1)
    assert arm.position == (0.5, 0.3, 0.1)
```

`yield` 是关键：`yield` 之前的部分是 setup（初始化），之后的部分是 teardown（清理）。即使测试失败，teardown 也会执行，确保测试之间不会泄漏资源。

### Mock（模拟）

使用 `mocker` fixture 比直接使用 `unittest.mock` 更方便。它会在测试结束时自动撤销所有 patch，无需使用 `with` 块。

Patch 一个方法：

```python
def test_uses_cached_position(mocker):
    mocker.patch("dimos.hardware.RobotArm.get_position", return_value=(0.0, 0.0, 0.0))
    arm = RobotArm()
    assert arm.get_position() == (0.0, 0.0, 0.0)
```

`mocker` 还提供了其他实用功能，例如 `mocker.MagicMock()` 用于创建模拟对象。

## 常用 pytest 选项

| 选项 | 描述 |
|--------|-------------|
| `-s` | 显示标准输出/标准错误输出 |
| `-v` | 更详细的测试名称 |
| `-x` | 第一个测试失败时立即停止 |
| `-k foo` | 只运行名称匹配 `foo` 的测试 |
| `--lf` | 仅重新运行上次失败的测试 |
| `--pdb` | 测试失败时进入调试器 |
| `--tb=short` | 使用更简短的回溯信息 |
| `--durations=0` | 测量每个测试的执行速度 |

## Marker（标记）

目前使用中的标记如下：

* `slow`：用于标记执行时间超过 1 秒的测试。
* `tool`：需要人工交互的测试。不推荐使用，请尽量避免。
* `mujoco`：使用 MuJoCo 的测试。这些测试非常慢，且目前在 CI 中无法运行。

如果需要跳过某个测试，请使用以下标记之一，或添加新的标记：

* `skipif_in_ci`：无法在 GitHub Actions 中运行的测试
* `skipif_no_openai`：需要环境变量中设置 `OPENAI_API_KEY` 的测试
* `skipif_no_alibaba`：需要环境变量中设置 `ALIBABA_API_KEY` 的测试
