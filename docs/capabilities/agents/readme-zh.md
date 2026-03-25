# 智能体

LLM（大语言模型）智能体作为原生 DimOS 模块运行。它们订阅摄像头、LiDAR（激光雷达）、里程计和空间记忆数据流，并通过技能控制机器人。

## 架构

```
Human Input ──→ Agent ──→ Skill Calls ──→ Robot
  (text/voice)     │         (RPC)
                   │
          subscribes to streams:
          color_image, odom, spatial_memory
```

**Agent**（`dimos/agents/agent.py`）是一个 `Module`，包含：
- `human_input: In[str]`：接收来自 `humancli`、`WebInput` 或 `agent-send` 的文本
- `agent: Out[BaseMessage]`：发布智能体响应（文本、工具调用、图像）
- `agent_idle: Out[bool]`：在智能体等待输入时发出信号

智能体使用 LangGraph 配合可配置的 LLM。默认模型为 `gpt-4o`，需要提供 `OPENAI_API_KEY` 环境变量。启动时，它通过 RPC 自动发现所有已部署模块中带有 `@skill` 注解的方法，并将其作为 LangChain 工具暴露。

## 技能

技能是任何 `Module` 上使用 `@skill` 装饰器标注的方法。智能体在启动时会自动发现它们。

```python
from dimos.agents.annotation import skill
from dimos.core.module import Module

class MySkillContainer(Module):
    @skill
    def wave_hello(self) -> str:
        """向最近的人挥手。"""
        # ... 机器人控制逻辑 ...
        return "Waving!"
```

**规则：**
- 参数必须是可 JSON 序列化的基本类型（`str`、`int`、`float`、`bool`、`list`、`dict`）。
- 文档字符串会成为 LLM 看到的工具描述，请清晰编写以便智能体获得足够的上下文信息。
- 函数必须返回字符串或图像，智能体将据此决定下一步操作。

### 内置技能

| 技能 | 模块 | 描述 |
|------|------|------|
| `relative_move(forward, left, degrees)` | `UnitreeSkillContainer` | 相对当前位置移动机器人 |
| `execute_sport_command(command_name)` | `UnitreeSkillContainer` | Unitree 运动指令（坐下、站立、翻转等） |
| `wait(seconds)` | `UnitreeSkillContainer` | 暂停执行 |
| `observe()` | `GO2Connection` | 捕获并返回当前摄像头画面 |
| `navigate_with_text(query)` | `NavigationSkillContainer` | 通过文本描述导航到指定位置 |
| `tag_location(name)` | `NavigationSkillContainer` | 标记当前位置以便后续调用 |
| `stop_navigation()` | `NavigationSkillContainer` | 取消当前导航目标 |
| `follow_person(query)` | `PersonFollowSkill` | 视觉伺服跟随指定描述的人 |
| `stop_following()` | `PersonFollowSkill` | 停止跟随 |
| `speak(text)` | `SpeakSkill` | 通过机器人扬声器进行语音合成 |
| `where_am_i()` | `GoogleMapsSkillContainer` | 通过 GPS 获取当前街道/区域 |
| `get_gps_position_for_queries(queries)` | `GoogleMapsSkillContainer` | 查询 GPS 坐标 |
| `set_gps_travel_points(points)` | `GPSNavSkill` | 通过 GPS 航点导航 |
| `map_query(query)` | `OsmSkill` | 使用 VLM（视觉语言模型）搜索 OpenStreetMap |

## MCP

DimOS 还提供了 MCP（模型上下文协议）实现。它用两个模块替代了 `Agent`：`McpServer` 和 `McpClient`。

* `McpServer` 将带有 `@skill` 注解的方法作为 MCP 工具暴露。任何外部客户端都可以连接到服务器使用 MCP 工具。
* `McpClient` 拥有一个 LangGraph LLM，用于调用来自 `McpServer` 的 MCP 工具。

CLI 访问：

```bash
dimos mcp list-tools                                # 列出可用技能
dimos mcp call relative_move --arg forward=0.5      # 调用技能
dimos mcp status                                    # 服务器状态
```

## 输入方式

| 方式 | 工作原理 |
|------|----------|
| `humancli` | 独立终端——输入消息，查看响应 |
| `dimos agent-send "text"` | 通过 LCM 发送的一次性 CLI 命令 |
| `WebInput` | localhost:7779 上的 Web 界面，支持可选的 Whisper STT（语音转文字） |

## 模型

| 配置 | 模型 | 备注 |
|------|------|------|
| 默认 | `gpt-4o` | 最佳质量，需要 `OPENAI_API_KEY` |
| `ollama:llama3.1` | 本地 Ollama | 需要运行 `ollama serve` |
| 自定义 | 任何兼容 LangChain 的模型 | 通过 `AgentConfig(model="...")` 设置 |
