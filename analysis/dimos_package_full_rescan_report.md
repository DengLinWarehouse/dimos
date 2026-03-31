# DimOS 代码包全量重扫报告

扫描时间：2026-03-31  
扫描范围：`D:\DevelopmentProject\ROBOOT\dimos\dimos`

## 1. 扫描结论摘要

本次重新扫描确认：`dimos/` 代码包不是单一机器人项目，也不是单一 Agent 项目，而是一套以 `Module + Blueprint + ModuleCoordinator` 为运行核心的通用机器人运行时平台。

它当前在源码层面同时承载了：

- 多机器人平台适配
- 感知 / 地图 / 导航
- 机械臂规划与操控
- Agent / Skill / MCP
- Web / 可视化 / 遥操作
- 多种本地通信与传输机制

从源码事实看，当前包内主运行链仍然稳定收敛为：

```text
dimos CLI
-> blueprint registry
-> autoconnect(...)
-> Blueprint.build()
-> ModuleCoordinator.start()
-> deploy_parallel(...)
-> connect_streams / connect_rpc_methods / connect_module_refs
-> start_all_modules()
-> loop()
```

对应核心文件：

- `dimos/robot/cli/dimos.py`
- `dimos/robot/get_all_blueprints.py`
- `dimos/robot/all_blueprints.py`
- `dimos/core/blueprints.py`
- `dimos/core/module_coordinator.py`

## 2. 包规模事实

### 2.1 文件规模

- `dimos/` 下当前共有 `976` 个文件

### 2.2 一级子系统分布

按 `dimos/` 一级路径统计，当前规模较大的子系统如下：

| 子系统 | 文件数 | 初步职责判断 |
| --- | ---: | --- |
| `robot/` | 120 | 机器人平台接入、蓝图、CLI、运行注册 |
| `msgs/` | 81 | 消息模型与类型化数据契约 |
| `web/` | 76 | Web 接口、可视化、前端交互 |
| `perception/` | 75 | 检测、跟踪、空间/时间感知 |
| `hardware/` | 68 | 摄像头、雷达、执行器等硬件接入 |
| `agents/` | 56 | Agent、MCP、技能容器 |
| `manipulation/` | 55 | 机械臂规划、抓取、操作流程 |
| `core/` | 53 | Module/Blueprint/Coordinator 内核 |
| `utils/` | 52 | 工具、日志、数据辅助 |
| `protocol/` | 50 | RPC / PubSub / TF 协议实现 |
| `mapping/` | 47 | 体素图、代价地图、地图相关能力 |
| `simulation/` | 27 | 仿真环境与仿真蓝图 |
| `navigation/` | 26 | A* 重规划、前沿探索等 |
| `stream/` | 25 | 流式组件与背压机制 |
| `teleop/` | 23 | Quest / Phone / Keyboard 遥操作 |

结论：

- `robot/ + core/ + agents/ + manipulation/ + perception/ + mapping/ + navigation` 一起构成真正的产品主干
- `web/` 和 `teleop/` 的体量也不小，说明该仓库不是只面向离线算法，而是面向可交互运行系统

## 3. 核心入口与注册机制

### 3.1 CLI 入口

`pyproject.toml` 中明确注册：

- `dimos = "dimos.robot.cli.dimos:main"`

说明 `dimos` 命令仍然是整个系统的统一启动入口。

### 3.2 运行命令入口

`dimos/robot/cli/dimos.py` 中的 `run()` 负责：

- 读取 CLI 覆盖配置
- 更新 `GlobalConfig`
- 清理 stale run registry
- 解析 blueprint 名称
- 构建 blueprint
- 启动运行时
- 写入本地 run registry

这意味着当前运行入口仍是：

- 命令式启动
- 本地运行
- 本地进程生命周期管理

### 3.3 蓝图与模块注册规模

`dimos/robot/all_blueprints.py` 当前实际注册：

- `76` 个蓝图
- `55` 个可按名称拉起的模块

这进一步证明：

- DimOS 是“蓝图库 + 模块库 + 运行时”
- 而不是单个应用程序

## 4. 运行时内核重新核对

### 4.1 Module

`dimos/core/module.py`

确认点：

- `Module` 仍然是最小功能单元
- 基于 `In[T] / Out[T]` 进行类型化流连接
- `@rpc` 方法自动纳入 RPC 暴露集合
- 模块具备独立事件循环、RPC、TF、流订阅生命周期

结论：

- Module 不是简单类封装，而是一个带通信、生命周期和远程调用能力的运行单元

### 4.2 Blueprint

`dimos/core/blueprints.py`

确认点：

- `Blueprint.create()` 将模块类型转为可装配原子
- `autoconnect()` 负责合并多个蓝图
- `build()` 内部完成：
  - `global_config` 合并
  - requirement / configurator 检查
  - `ModuleCoordinator.start()`
  - `deploy_parallel(...)`
  - `connect_streams(...)`
  - `connect_rpc_methods(...)`
  - `connect_module_refs(...)`
  - `start_all_modules()`

结论：

- Blueprint 是系统装配器，不是业务模块
- autoconnect 的关键价值是“按声明自动连线并消除重复模块”

### 4.3 ModuleCoordinator

`dimos/core/module_coordinator.py`

确认点：

- 通过 `WorkerManager` 管理 worker 进程
- 支持 `deploy_parallel(...)`
- 支持并行 `start_all_modules()`
- 提供 `health_check()`、`stop()`、`loop()` 等生命周期控制

结论：

- ModuleCoordinator 是本地运行时调度核心
- 当前运行模型仍然是“多 worker + 多 ModuleProxy”的进程级编排

### 4.4 GlobalConfig

`dimos/core/global_config.py`

确认点：

- 仍由 `BaseSettings` 驱动
- 支持 `robot_ip`、`simulation`、`replay`、`viewer`、`n_workers`、`mcp_port` 等全局字段
- `unitree_connection_type` 由 `replay/simulation` 自动派生

结论：

- GlobalConfig 依旧是全局运行参数的统一入口
- 本地运行模式切换仍然高度依赖它

## 5. 当前真实通信机制

`dimos/core/transport.py` 重新确认了当前支持的主要传输形态：

- `LCMTransport`
- `pLCMTransport`
- `SHMTransport`
- `pSHMTransport`
- `ROSTransport`
- `DDSTransport`（按依赖条件启用）

结论：

- DimOS 当前不是只依赖一种总线
- 它本质上支持“按消息类型和场景切换底层传输”
- LCM / 共享内存仍是默认核心通信面
- ROS / DDS 是扩展互操作面

## 6. 平台适配层重新识别

`dimos/robot/` 下当前一级目录为：

- `cli`
- `drone`
- `manipulators`
- `unitree`
- `unitree_webrtc`
- `utils`

结论：

- 机器人平台适配仍然以 `unitree + drone + manipulators` 三大族群为主
- `unitree_webrtc` 独立存在，说明真机连接链路在代码结构上已被明确抽离

## 7. Agent / MCP 层重新核对

### 7.1 Agent

`dimos/agents/agent.py`

确认点：

- `Agent` 仍然是一个 `Module`
- 通过 `human_input` 输入消息
- 在 `on_system_modules()` 阶段收集系统中全部 skill
- 使用 LangChain / LangGraph 创建 agent

结论：

- Agent 不是外挂脚本，而是系统内标准模块
- 它依赖“从运行中的模块集合中抽取 skills”

### 7.2 MCP Server

`dimos/agents/mcp/mcp_server.py`

确认点：

- `McpServer` 是独立 `Module`
- 基于 FastAPI + Uvicorn 暴露 `/mcp`
- 在 `on_system_modules()` 中收集全部 skills 并映射为 RPC 调用
- 自带 `server_status`、`list_modules`、`agent_send` 等 introspection / control skill

结论：

- MCP 能力并不是全局默认存在，而是一个可装配模块
- 只有明确装配 `McpServer` 的蓝图，才具备外部 MCP 端点

## 8. 三条代表性业务主链重新确认

### 8.1 Go2 agentic MCP 主链

关键文件：

- `dimos/robot/unitree/go2/blueprints/smart/unitree_go2.py`
- `dimos/robot/unitree/go2/blueprints/smart/unitree_go2_spatial.py`
- `dimos/robot/unitree/go2/blueprints/agentic/_common_agentic.py`
- `dimos/robot/unitree/go2/blueprints/agentic/unitree_go2_agentic_mcp.py`

源码链条：

```text
unitree_go2_basic
-> unitree_go2
   + voxel_mapper
   + cost_mapper
   + replanning_a_star_planner
   + wavefront_frontier_explorer
-> unitree_go2_spatial
   + spatial_memory
   + PerceiveLoopSkill
-> _common_agentic
   + navigation_skill
   + person_follow_skill
   + unitree_skills
   + web_input
   + speak_skill
-> unitree_go2_agentic_mcp
   + McpServer
   + mcp_client
```

结论：

- Go2 主线当前仍是仓库最完整、最能代表 DimOS 架构价值的一条链
- 它覆盖了基础连接、导航建图、空间记忆、技能系统、MCP 外部暴露

### 8.2 无人机 agentic 主链

关键文件：

- `dimos/robot/drone/blueprints/agentic/drone_agentic.py`

源码链条：

```text
drone_basic
-> DroneTrackingModule
-> GoogleMapsSkillContainer
-> OsmSkill
-> agent(system_prompt=..., model="gpt-4o")
-> web_input
-> remappings(video_input <- video, cmd_vel <- movecmd_twist)
```

结论：

- 无人机链路说明 DimOS 不局限于地面机器人
- 它复用了同一套 `autoconnect + remappings + skill + agent` 机制

### 8.3 机械臂规划 / 操控主链

关键文件：

- `dimos/manipulation/blueprints.py`

确认点：

- 该文件不是单一蓝图，而是一个完整机械臂蓝图工厂
- 内部包含：
  - XArm6 / XArm7 / Piper 等机器人模型配置
  - planner-only
  - dual-arm planner
  - perception + manipulation 组合
  - coordinator 集成

结论：

- manipulation 不是仓库边角功能，而是第二条重要业务线
- 机械臂链路的复杂度已经达到独立产品级别

## 9. 当前最重要的系统特征

基于这次重新扫描，可以把 DimOS 当前代码包总结成 6 个关键特征：

1. 它是本地运行时平台，不是默认云端系统
2. 它以 Blueprint 为系统装配单位，而不是以单模块或单服务为单位
3. 它同时支持移动机器人、无人机、机械臂三类主要机器人形态
4. 它把 Agent / MCP 作为可插拔能力层，而不是强耦合内核
5. 它已经具备比较成熟的多传输机制与多进程运行基础
6. 它当前最强的演示主线仍然是 Go2 agentic / MCP 栈

## 10. 本次重扫后的最新判断

如果只基于 `dimos/` 包本身做一句话判断，那么当前最准确的表述是：

> DimOS 是一个以 `Module + Blueprint + 多进程运行时` 为核心的通用机器人运行平台，已经实际承载了多平台机器人接入、导航建图、感知记忆、机械臂操控、Agent/MCP 与遥操作可视化等完整能力族群。

如果按“当前最值得继续深入分析的源码主线”排序，建议优先级如下：

1. Go2 agentic MCP 运行主链
2. Blueprint / ModuleCoordinator / Stream / RPC 内核装配链
3. Manipulation 规划与 coordinator 协作链
4. Drone agentic 适配链

## 11. 关键证据索引

- CLI 入口：`dimos/robot/cli/dimos.py`
- 蓝图注册：`dimos/robot/all_blueprints.py`
- 名称解析：`dimos/robot/get_all_blueprints.py`
- Blueprint 内核：`dimos/core/blueprints.py`
- Module 内核：`dimos/core/module.py`
- 运行协调器：`dimos/core/module_coordinator.py`
- 全局配置：`dimos/core/global_config.py`
- 传输层：`dimos/core/transport.py`
- Go2 主链：`dimos/robot/unitree/go2/blueprints/...`
- Drone 主链：`dimos/robot/drone/blueprints/agentic/drone_agentic.py`
- Manipulation 主链：`dimos/manipulation/blueprints.py`
- Agent：`dimos/agents/agent.py`
- MCP Server：`dimos/agents/mcp/mcp_server.py`
