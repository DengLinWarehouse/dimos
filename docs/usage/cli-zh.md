# CLI 参考

`dimos` CLI 管理 DimOS 机器人栈的完整生命周期 — 启动、停止、检查和交互。

## 全局选项

每个 [`GlobalConfig`](/docs/usage/configuration.md) 字段都可以作为 CLI 标志使用。标志会覆盖环境变量、`.env` 和蓝图默认值。

```bash
dimos [GLOBAL OPTIONS] COMMAND [ARGS]
```

| 标志 | 类型 | 默认值 | 描述 |
|------|------|---------|-------------|
| `--robot-ip` | TEXT | `None` | 机器人 IP 地址 |
| `--robot-ips` | TEXT | `None` | 多个机器人 IP |
| `--simulation` / `--no-simulation` | bool | `False` | 启用 MuJoCo 仿真 |
| `--replay` / `--no-replay` | bool | `False` | 使用录制的回放数据 |
| `--replay-dir` | TEXT | `go2_sf_office` | 回放数据集目录名 |
| `--new-memory` / `--no-new-memory` | bool | `False` | 启动时清除持久化记忆 |
| `--viewer` | `rerun\|rerun-web\|rerun-connect\|foxglove\|none` | `rerun` | 可视化后端 |
| `--n-workers` | INT | `2` | forkserver 工作进程数 |
| `--memory-limit` | TEXT | `auto` | Rerun 查看器内存限制 |
| `--mcp-port` | INT | `9990` | MCP 服务器端口 |
| `--mcp-host` | TEXT | `0.0.0.0` | MCP 服务器绑定地址 |
| `--dtop` / `--no-dtop` | bool | `False` | 启用实时资源监控叠加层 |
| `--obstacle-avoidance` / `--no-obstacle-avoidance` | bool | `True` | 启用避障功能 |
| `--detection-model` | `qwen\|moondream` | `moondream` | 目标检测使用的视觉模型 |
| `--robot-model` | TEXT | `None` | 机器人型号标识符 |
| `--robot-width` | FLOAT | `0.3` | 机器人宽度（米） |
| `--robot-rotation-diameter` | FLOAT | `0.6` | 机器人旋转直径（米） |
| `--planner-strategy` | `simple\|mixed` | `simple` | 导航规划器策略 |
| `--planner-robot-speed` | FLOAT | `None` | 规划器机器人速度覆盖值 |
| `--mujoco-camera-position` | TEXT | `None` | MuJoCo 相机位置 |
| `--mujoco-room` | TEXT | `None` | MuJoCo 房间模型 |
| `--mujoco-room-from-occupancy` | TEXT | `None` | 从占据栅格图生成房间 |
| `--mujoco-global-costmap-from-occupancy` | TEXT | `None` | 从占据栅格图生成代价地图 |
| `--mujoco-global-map-from-pointcloud` | TEXT | `None` | 从点云生成地图 |
| `--mujoco-start-pos` | TEXT | `-1.0, 1.0` | MuJoCo 机器人起始位置 |
| `--mujoco-steps-per-frame` | INT | `7` | MuJoCo 每帧仿真步数 |

### 配置优先级

值按以下顺序级联（后者覆盖前者）：

1. `GlobalConfig` 默认值 → `simulation = False`
2. `.env` 文件 → `DIMOS_SIMULATION=true`
3. 环境变量 → `export DIMOS_SIMULATION=true`
4. 蓝图定义 → `.global_config(simulation=True)`
5. CLI 标志 → `dimos --simulation run ...`

环境变量和 `.env` 的值必须以 `DIMOS_` 为前缀。

---

## 命令

### `dimos run`

启动一个机器人蓝图。

```bash
dimos run <blueprint> [<blueprint> ...] [--daemon] [--disable <module> ...]
```

| 选项 | 描述 |
|--------|-------------|
| `--daemon`, `-d` | 在后台运行（双 fork，健康检查，写入运行注册表） |
| `--disable` | 要从蓝图中排除的模块类名 |

```bash
# 前台运行（Ctrl-C 停止）
dimos run unitree-go2

# 后台运行（立即返回）
dimos run unitree-go2-agentic --daemon

# 使用 Rerun 查看器进行回放
dimos --replay --viewer rerun run unitree-go2

# 连接真实机器人
dimos run unitree-go2-agentic --robot-ip 192.168.123.161

# 动态组合模块
dimos run unitree-go2 keyboard-teleop

# 禁用特定模块
dimos run unitree-go2-agentic --disable OsmSkill WebInput
```

使用 `--daemon` 时，进程会：
1. 构建并启动所有模块（前台执行 — 可以看到错误信息）
2. 运行健康检查（轮询 worker PID）
3. Fork 到后台，写入运行注册表条目
4. 打印运行 ID、PID、日志路径和 MCP 端点

#### 添加新蓝图

定义一个模块级的 `Blueprint` 变量，然后在 `all_blueprints.py` 中注册：

```bash
pytest dimos/robot/test_all_blueprints_generation.py
```

这会自动生成注册表。有关组合细节，请参阅[蓝图](/docs/usage/blueprints.md)。

### `dimos status`

显示正在运行的 DimOS 实例。

```bash
dimos status
```

读取运行注册表，验证 PID 是否存活，并显示：运行 ID、PID、蓝图名称、运行时间、日志路径和 MCP 端口。

### `dimos stop`

停止正在运行的 DimOS 实例。

```bash
dimos stop [--force]
```

| 选项 | 描述 |
|--------|-------------|
| `--force`, `-f` | 立即发送 SIGKILL（跳过优雅的 SIGTERM） |

默认行为：SIGTERM → 等待 5 秒 → SIGKILL。同时清理运行注册表条目。

### `dimos restart`

使用原始参数重启正在运行的实例。

```bash
dimos restart [--force]
```

| 选项 | 描述 |
|--------|-------------|
| `--force`, `-f` | 重启前强制终止 |

从运行注册表中读取保存的 CLI 参数，停止当前实例，然后使用相同参数重新运行。

### `dimos log`

查看 DimOS 运行日志。

```bash
dimos log [OPTIONS]
```

| 选项 | 描述 |
|--------|-------------|
| `--follow`, `-f` | 持续跟踪日志输出（类似 `tail -f`） |
| `--lines`, `-n` | 显示的行数（默认：50） |
| `--all`, `-a` | 显示完整日志 |
| `--json` | 原始 JSONL 输出（可通过管道传递给 `jq`） |
| `--run`, `-r` | 指定运行 ID（默认为最近一次） |

```bash
dimos log                    # 最近 50 行，人类可读格式
dimos log -f                 # 实时跟踪
dimos log -n 100             # 最近 100 行
dimos log --json | jq .event # 原始 JSONL，提取事件
dimos log -r 20260306-143022-unitree-go2  # 指定运行
```

所有进程（主进程 + worker）写入同一个 `main.jsonl`。按模块过滤：

```bash
dimos log --json | jq 'select(.logger | contains("RerunBridge"))'
```

### `dimos list`

列出所有可用的蓝图。

```bash
dimos list
```

### `dimos show-config`

打印解析后的 GlobalConfig 值及其来源。

```bash
dimos show-config
```

---

## 智能体与 MCP 命令

### `dimos agent-send`

通过 LCM 向正在运行的智能体发送文本消息。

```bash
dimos agent-send "walk forward 2 meters"
```

适用于任何智能体蓝图 — 不需要 MCP。直接发布到 `/human_input` LCM topic。

### `dimos mcp`

与正在运行的 MCP 服务器交互。**需要包含 `McpServer` 的蓝图** — 例如 `unitree-go2-agentic-mcp`。MCP 服务器默认运行在 `http://localhost:9990/mcp`（可通过 `--mcp-port` / `--mcp-host` 覆盖）。

要在蓝图中添加 MCP，需同时包含 `McpServer`（将技能暴露为 HTTP 工具）和 `mcp_client()`（从服务器获取工具的 LLM 智能体）：

```python
from dimos.agents.mcp.mcp_client import mcp_client
from dimos.agents.mcp.mcp_server import McpServer

my_mcp_blueprint = autoconnect(
    my_robot_stack,
    McpServer.blueprint(),
    mcp_client(),
    my_skill_containers,
)
```

#### `dimos mcp list-tools`

列出 MCP 服务器暴露的所有可用技能。

```bash
dimos mcp list-tools
```

返回包含工具名称、描述和参数 schema 的 JSON。

#### `dimos mcp call`

按名称调用一个技能。

```bash
dimos mcp call <tool_name> [--arg key=value ...] [--json-args '{}']
```

| 选项 | 描述 |
|--------|-------------|
| `--arg`, `-a` | 以 `key=value` 对传递参数（可重复） |
| `--json-args`, `-j` | 以 JSON 字符串传递参数 |

```bash
dimos mcp call relative_move --arg forward=0.5
dimos mcp call relative_move --json-args '{"forward": 2.0, "left": 0, "degrees": 0}'
dimos mcp call observe
dimos mcp call land
```

#### `dimos mcp status`

显示 MCP 服务器状态 — PID、运行时间、已部署模块、技能数量。

```bash
dimos mcp status
```

#### `dimos mcp modules`

列出已部署的模块及其技能。

```bash
dimos mcp modules
```

---

## 独立工具

以下工具作为独立入口点安装，可以直接运行，无需 `dimos` 前缀。

### `humancli`

用于向运行中的智能体发送消息的交互式终端。

```bash
humancli
```

### `lcmspy`

实时监控 LCM 消息。

```bash
lcmspy
```

### `agentspy`

监控智能体消息和工具调用。

```bash
agentspy
```

### `dtop`

实时资源监控 TUI — CPU、内存和进程统计。也可以在运行时通过 `--dtop` 启用：

```bash
dimos --dtop run unitree-go2
```

或独立运行：

```bash
dtop
```

### `rerun-bridge`

以独立进程启动 Rerun 可视化桥接（在蓝图之外）。

```bash
rerun-bridge
```

也可以通过 `dimos rerun-bridge` 运行。

---

## 文件位置

| 路径 | 内容 |
|------|----------|
| `~/.local/state/dimos/runs/<run-id>.json` | 运行注册表（PID、蓝图、参数、端口）。被 `status`/`stop`/`restart` 使用。进程退出时自动清理。 |
| `~/.local/state/dimos/logs/<run-id>/main.jsonl` | 结构化日志（主进程 + 所有 worker） |
| `.env` | 本地配置覆盖（`DIMOS_ROBOT_IP=192.168.123.161`） |
