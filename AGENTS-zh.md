# Dimensional AGENTS.md

## 什么是 DimOS

面向通用机器人（generalist robotics）的智能体操作系统。`Modules` 通过 LCM、ROS2、DDS 或其他传输上的类型化流进行通信。`Blueprints` 将模块组合为可运行的机器人栈。`Skills` 让智能体具备执行诸如 `grab()`、`follow_object()` 或 `jump()` 之类真实硬件动作的能力。

---

## 快速开始

```bash
# Install
uv sync --all-extras --no-extra dds

# List all runnable blueprints
dimos list

# --- Go2 quadruped ---
dimos --replay run unitree-go2                  # perception + mapping, replay data
dimos --replay run unitree-go2 --daemon         # same, backgrounded
dimos --replay run unitree-go2-agentic          # + LLM agent (GPT-4o) + skills
dimos --replay run unitree-go2-agentic-mcp      # + McpServer + McpClient (MCP tools live)
dimos run unitree-go2-agentic --robot-ip 192.168.123.161  # real Go2 hardware

# --- G1 humanoid ---
dimos --simulation run unitree-g1-agentic-sim   # G1 in MuJoCo sim + agent + skills
dimos run unitree-g1-agentic --robot-ip 192.168.123.161   # real G1 hardware

# --- Inspect & control ---
dimos status
dimos log              # last 50 lines, human-readable
dimos log -f           # follow/tail in real time
dimos agent-send "say hello"
dimos stop             # graceful SIGTERM → SIGKILL
dimos restart          # stop + re-run with same original args
```

### Blueprint 快速参考

| Blueprint | 机器人 | 硬件 | Agent | MCP server | 说明 |
|-----------|-------|----------|-------|------------|-------|
| `unitree-go2-agentic-mcp` | Go2 | real | 通过 McpClient | ✓ | **唯一一个默认启用 McpServer 的 blueprint** |
| `unitree-g1-agentic-sim` | G1 | sim | GPT-4o（G1 prompt） | — | 完整智能体仿真，无需真实机器人 |
| `xarm-perception-agent` | xArm | real | GPT-4o | — | 操作 + 感知 + 智能体 |
| `xarm7-trajectory-sim` | xArm7 | sim | — | — | 轨迹规划仿真 |
| `arm-teleop-xarm7` | xArm7 | real | — | — | Quest VR 遥操作 |
| `dual-xarm6-planner` | xArm6×2 | real | — | — | 双臂运动规划器 |

完整列表请运行 `dimos list`。

---

## 你可用的工具（MCP）

**只有 blueprint 中包含 `McpServer` 时，MCP 才能工作。** 目前唯一自带该能力的 blueprint 是 `unitree-go2-agentic-mcp`。其他所有 agentic blueprint 都使用进程内 `Agent` 模块，**不会**暴露 MCP 端点。

```bash
# Start the MCP-enabled blueprint first:
dimos --replay run unitree-go2-agentic-mcp --daemon

# Then use MCP tools:
dimos mcp list-tools                                              # all available skills as JSON
dimos mcp call move --arg x=0.5 --arg duration=2.0               # call by key=value args
dimos mcp call move --json-args '{"x": 0.5, "duration": 2.0}'    # call by JSON
dimos mcp status      # PID, module list, skill list
dimos mcp modules     # module → skills mapping

# Send a message to the running agent (works without McpServer too):
dimos agent-send "walk forward 2 meters then wave"
```

MCP 服务器运行在 `http://localhost:9990/mcp`（`GlobalConfig.mcp_port`）。

### 向 blueprint 添加 McpServer

请同时使用 **`McpServer`** 和 **`mcp_client()`** ——不要与 `agent()` 混用。

```python
from dimos.agents.mcp.mcp_client import mcp_client
from dimos.agents.mcp.mcp_server import McpServer

unitree_go2_agentic_mcp = autoconnect(
    unitree_go2_spatial,   # robot stack
    McpServer.blueprint(), # HTTP MCP server — exposes all @skill methods on port 9990
    mcp_client(),          # LLM agent — fetches tools from McpServer
    _common_agentic,       # skill containers
)
```

参考：`dimos/robot/unitree/go2/blueprints/agentic/unitree_go2_agentic_mcp.py`

---

## 仓库结构

```
dimos/
├── core/                    # Module system, blueprints, workers, transports
│   ├── module.py            # Module base class, In/Out streams, @rpc, @skill
│   ├── blueprints.py        # Blueprint composition (autoconnect)
│   ├── global_config.py     # GlobalConfig (env vars, CLI flags, .env)
│   └── run_registry.py      # Per-run tracking + log paths
├── robot/
│   ├── cli/dimos.py         # CLI entry point (typer)
│   ├── all_blueprints.py    # Auto-generated blueprint registry (DO NOT EDIT MANUALLY)
│   ├── unitree/             # Unitree robot implementations (Go2, G1, B1)
│   │   ├── unitree_skill_container.py  # Go2 @skill methods
│   │   ├── go2/             # Go2 blueprints and connection
│   │   └── g1/              # G1 blueprints, connection, sim, skills
│   └── drone/               # Drone implementations (MAVLink + DJI)
│       ├── connection_module.py        # MAVLink connection
│       ├── camera_module.py            # DJI video stream
│       ├── drone_tracking_module.py    # Visual object tracking
│       └── drone_visual_servoing_controller.py  # Visual servoing
├── agents/
│   ├── agent.py             # Agent module (LangGraph-based)
│   ├── system_prompt.py     # Default Go2 system prompt
│   ├── annotation.py        # @skill decorator
│   ├── mcp/                 # McpServer, McpClient, McpAdapter
│   └── skills/              # NavigationSkillContainer, SpeakSkill, etc.
├── navigation/              # Path planning, frontier exploration
├── perception/              # Object detection, tracking, memory
├── visualization/rerun/     # Rerun bridge
├── msgs/                    # Message types (geometry_msgs, sensor_msgs, nav_msgs)
└── utils/                   # Logging, data loading, CLI tools
docs/
├── usage/modules.md         # ← Module system deep dive
├── usage/blueprints.md      # Blueprint composition guide
├── usage/configuration.md   # GlobalConfig + Configurable pattern
├── development/testing.md   # Fast/slow tests, pytest usage
├── development/dimos_run.md # CLI usage, adding blueprints
└── agents/                  # Agent system documentation
```

---

## 架构

### Modules

自治子系统。通过 `In[T]`/`Out[T]` 类型化流进行通信，并运行在 `forkserver` worker 进程中。

```python
from dimos.core.module import Module
from dimos.core.stream import In, Out
from dimos.core.core import rpc
from dimos.msgs.sensor_msgs import Image

class MyModule(Module):
    color_image: In[Image]
    processed: Out[Image]

    @rpc
    def start(self) -> None:
        super().start()
        self.color_image.subscribe(self._process)

    def _process(self, img: Image) -> None:
        self.processed.publish(do_something(img))
```

### Blueprints

使用 `autoconnect()` 组合模块。流会按照 `(name, type)` 自动匹配连接。

```python
from dimos.core.blueprints import autoconnect

my_blueprint = autoconnect(module_a(), module_b(), module_c())
```

如果想直接从 Python 运行一个 blueprint：

```python
# build() deploys all modules into forkserver workers and wires streams
# loop() blocks the main thread until stopped (Ctrl-C or SIGTERM)
autoconnect(module_a(), module_b(), module_c()).build().loop()
```

将其作为模块级变量暴露出来，`dimos run` 才能发现它。然后运行 `pytest dimos/robot/test_all_blueprints_generation.py` 将其加入注册表。

### GlobalConfig

单例配置。值的覆盖顺序为：defaults → `.env` → 环境变量 → blueprint → CLI flags。环境变量统一以 `DIMOS_` 为前缀。关键字段包括：`robot_ip`、`simulation`、`replay`、`viewer`、`n_workers`、`mcp_port`。

### Transports

- **LCMTransport**：默认传输。基于组播 UDP。
- **SHMTransport/pSHMTransport**：共享内存——适用于图像和点云。
- **pLCMTransport**：Pickled LCM——适用于复杂 Python 对象。
- **ROSTransport**：ROS 话题桥接——可与 ROS 节点互操作（`dimos/core/transport.py`）。
- **DDSTransport**：DDS 发布/订阅——在 `DDS_AVAILABLE` 为真时可用；安装方式为 `uv sync --extra dds`（`dimos/protocol/pubsub/impl/ddspubsub.py`）。

---

## CLI 参考

### 全局标志

每个 `GlobalConfig` 字段都对应一个 CLI 标志：`--robot-ip`、`--simulation/--no-simulation`、`--replay/--no-replay`、`--viewer {rerun|rerun-web|foxglove|none}`、`--mcp-port`、`--n-workers` 等。CLI 标志会覆盖 `.env` 和环境变量。

### 核心命令

| 命令 | 说明 |
|---------|-------------|
| `dimos run <blueprint> [--daemon]` | 启动一个 blueprint |
| `dimos status` | 显示当前运行实例（run ID、PID、blueprint、运行时长、日志路径） |
| `dimos stop [--force]` | 先发 `SIGTERM`，5 秒后再发 `SIGKILL`；`--force` 表示立即 `SIGKILL` |
| `dimos restart [--force]` | 停止后使用原始参数重新执行 |
| `dimos list` | 列出所有非 demo blueprint |
| `dimos show-config` | 打印解析后的 `GlobalConfig` 值 |
| `dimos log [-f] [-n N] [--json] [-r <run-id>]` | 查看单次运行日志 |
| `dimos mcp list-tools / call / status / modules` | MCP 工具（要求 blueprint 中包含 `McpServer`） |
| `dimos agent-send "<text>"` | 通过 LCM 向运行中的 agent 发送文本 |
| `dimos lcmspy / agentspy / humancli / top` | 调试/诊断工具 |
| `dimos topic echo <topic> / send <topic> <expr>` | LCM 话题收发 |
| `dimos rerun-bridge` | 独立启动 Rerun 可视化 |

日志文件：`~/.local/state/dimos/logs/<run-id>/main.jsonl`
运行注册表：`~/.local/state/dimos/runs/<run-id>.json`

---

## Agent 系统

### `@skill` 装饰器

定义在 `dimos/agents/annotation.py` 中。它会设置 `__rpc__ = True` 和 `__skill__ = True`。

- 仅使用 `@rpc`：可通过 RPC 调用，但不会暴露给 LLM
- 使用 `@skill`：隐含 `@rpc`，并将该方法作为工具暴露给 LLM。**不要同时叠加两个装饰器。**

#### Schema 生成规则

| 规则 | 违反后的结果 |
|------|------------------------------|
| **必须有 docstring** | 启动时抛出 `ValueError` —— 模块注册失败，所有技能都会消失 |
| **每个参数都要有类型注解** | 缺少注解 → schema 中没有 `"type"` —— LLM 无法获得类型信息 |
| **返回 `str`** | 返回 `None` → agent 只会听到 “It has started. You will be updated later.” |
| **`description` 中会原样包含完整 docstring** | 请保持 `Args:` 段简洁——它会出现在每次工具调用提示中 |

支持的参数类型：`str`、`int`、`float`、`bool`、`list[str]`、`list[float]`。避免复杂嵌套类型。

#### 最小正确 skill 示例

```python
from dimos.agents.annotation import skill
from dimos.core.core import rpc
from dimos.core.module import Module

class MySkillContainer(Module):
    @rpc
    def start(self) -> None:
        super().start()

    @rpc
    def stop(self) -> None:
        super().stop()

    @skill
    def move(self, x: float, duration: float = 2.0) -> str:
        """Move the robot forward or backward.

        Args:
            x: Forward velocity in m/s. Positive = forward, negative = backward.
            duration: How long to move in seconds.
        """
        return f"Moving at {x} m/s for {duration}s"

my_skill_container = MySkillContainer.blueprint
```

### System Prompts

| 机器人 | 文件 | 变量 |
|-------|------|----------|
| Go2（默认） | `dimos/agents/system_prompt.py` | `SYSTEM_PROMPT` |
| G1 humanoid | `dimos/robot/unitree/g1/system_prompt.py` | `G1_SYSTEM_PROMPT` |

传入机器人专用 prompt：`agent(system_prompt=G1_SYSTEM_PROMPT)`。Agent 默认使用 Go2 prompt——如果 prompt 不匹配，会导致技能幻觉。

### RPC Wiring

若要调用其他模块上的方法，请先声明一个 `Spec` Protocol，并使用它为某个属性添加类型标注。blueprint 会在构建时注入匹配模块——全程保持完整类型检查，不依赖字符串；如果找不到匹配项，会在构建阶段失败，而不是运行时失败。

```python
# my_module_spec.py
from typing import Protocol
from dimos.spec.utils import Spec

class NavigatorSpec(Spec, Protocol):
    def set_goal(self, goal: PoseStamped) -> bool: ...
    def cancel_goal(self) -> bool: ...

# my_skill_container.py
class MySkillContainer(Module):
    _navigator: NavigatorSpec   # injected by blueprint at build time

    @skill
    def go_to(self, x: float, y: float) -> str:
        """Navigate to a position."""
        self._navigator.set_goal(make_pose(x, y))
        return "Navigating"
```

如果有多个模块匹配该 spec，请使用 `.remappings()` 消除歧义。源码位置：`dimos/spec/utils.py`、`dimos/core/blueprints.py`。

**Legacy**：现有部分 skill container 仍使用 `rpc_calls: list[str]` + `get_rpc_calls("ClassName.method")`。这种方式依然可用，但连线失败是静默的，只会在运行时暴露问题。新代码不要这样做。

### 添加新 Skill

1. 选择合适的 container（机器人专用，或 `dimos/agents/skills/` 下的通用容器）。
2. 添加 `@skill`，并确保所有参数都带有必需的 docstring 和类型注解。
3. 如果需要调用其他模块的 RPC，请使用 Spec 模式。
4. 返回一个描述性的 `str`。
5. 更新 system prompt——将新技能加入 `# AVAILABLE SKILLS` 段。
6. 以 `my_container = MySkillContainer.blueprint` 的形式暴露，并将其加入 agentic blueprint。

---

## 测试

```bash
# Fast tests (default)
uv run pytest

# Include slow tests (CI)
./bin/pytest-slow

# Single file
uv run pytest dimos/core/test_blueprints.py -v

# Mypy
uv run mypy dimos/
```

`uv run pytest` 默认排除 `slow`、`tool` 和 `mujoco` 标记。CI（`./bin/pytest-slow`）会包含 `slow`，但仍排除 `tool` 和 `mujoco`。详情见 `docs/development/testing.md`。

---

## Pre-commit 与代码风格

`pre-commit` 会在 `git commit` 时运行。包含 ruff format/check、许可证头检查、LFS 检查等。

**提交前务必激活虚拟环境：** `source .venv/bin/activate`

代码风格规则：
- `import` 放在文件顶部。除非存在循环依赖，否则不要内联导入。
- HTTP 请使用 `requests`（不要用 `urllib`）。JSON 值类型请使用 `Any`（不要用 `object`）。
- 手动测试脚本请使用 `demo_` 前缀，以避免被 pytest 自动收集。
- 不要硬编码端口或 URL——请使用 `GlobalConfig` 常量。
- 必须写类型注解。Mypy 使用严格模式。

---

## `all_blueprints.py` 是自动生成的

`dimos/robot/all_blueprints.py` 由 `test_all_blueprints_generation.py` 自动生成。新增或重命名 blueprint 后，请运行：

```bash
pytest dimos/robot/test_all_blueprints_generation.py
```

CI 会断言该文件处于最新状态——如果过期，CI 会失败。

---

## Git 工作流

- 分支前缀：`feat/`、`fix/`、`refactor/`、`docs/`、`test/`、`chore/`、`perf/`
- **PR 目标分支是 `dev`** ——不要直接推送到 `main` 或 `dev`
- **不要强推**，除非是在解决 rebase 冲突之后
- **尽量减少 push 次数** ——每次 push 都会触发 CI（在自托管 runner 上约 1 小时）。请先在本地攒好提交，再一次性推送。

---

## 延伸阅读

- 模块系统：`docs/usage/modules.md`
- Blueprints：`docs/usage/blueprints.md`
- 可视化：`docs/usage/visualization.md`
- 配置：`docs/usage/configuration.md`
- 测试：`docs/development/testing.md`
- CLI / dimos run：`docs/development/dimos_run.md`
- LFS 数据：`docs/development/large_file_management.md`
- Agent 系统：`docs/agents/`
