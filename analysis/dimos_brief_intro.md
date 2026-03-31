# DimOS 简介

## 1. DimOS 是什么

DimOS 是一个面向机器人的通用运行时平台，可以把传感器、控制、导航、感知、Agent 和技能系统装配成一个可运行的机器人栈。

它不是单一机器人项目，而是一个可组合的机器人操作系统，当前覆盖：

- 四足机器狗
- 人形机器人
- 机械臂
- 无人机

关键实现入口：

- `Module`：功能单元，见 `dimos/core/module.py`
- `Blueprint`：装配单元，见 `dimos/core/blueprints.py`
- `dimos` CLI：运行入口，见 `dimos/robot/cli/dimos.py`

## 2. DimOS 有什么用

DimOS 主要用来快速搭建和运行机器人应用，而不是从零手写整套机器人中间层。

它解决的核心问题是：

- 把不同机器人硬件接进同一套运行框架
- 把感知、建图、导航、控制等能力按需组合
- 把机器人技能暴露给 Agent 或 MCP
- 支持真机、回放、仿真三种运行方式

典型用途：

- 跑一个 Go2 的导航和建图系统
- 给机器人接入自然语言 Agent
- 给技能开放 MCP 接口，供外部系统调用
- 运行机械臂规划、抓取和操作流程

## 3. 主要工作逻辑

DimOS 的主逻辑是“先装配，再运行”。

```mermaid
flowchart LR
    A["CLI: dimos run <blueprint>"] --> B["按名称加载 Blueprint"]
    B --> C["autoconnect 组合多个模块"]
    C --> D["Blueprint.build()"]
    D --> E["启动 Worker 和模块"]
    E --> F["连接 Stream / RPC / ModuleRef"]
    F --> G["统一 start()"]
    G --> H["系统持续运行"]
```

具体来说：

1. 用户执行 `dimos run unitree-go2-agentic-mcp`
2. 系统从蓝图注册表里找到对应蓝图
3. 蓝图把多个 `Module` 组合在一起
4. `build()` 把这些模块部署到 worker 进程
5. 系统自动连好数据流、RPC 和依赖注入
6. 所有模块开始工作，形成完整机器人系统

对应代码：

- CLI 入口：`dimos/robot/cli/dimos.py:112`
- 蓝图构建：`dimos/core/blueprints.py:474`
- 自动装配：`dimos/core/blueprints.py:502`
- 运行协调：`dimos/core/module_coordinator.py:35`

## 4. 为什么要这样做

因为机器人系统天然是“多能力协同”的，不适合写成一个巨大单体。

DimOS 这样设计的好处是：

- 模块化：每个模块只负责一个功能，便于替换和复用
- 可组合：同一套感知或 Agent 能力可以复用到不同机器人
- 易扩展：新增硬件、新技能、新蓝图时，不需要重写整套系统
- 易运行：同一条运行链可以切换真机、回放和仿真
- 易接入智能体：技能可直接暴露给 Agent 或 MCP

一句话总结：

> DimOS 的目标，是把“机器人能力开发”从一次性工程，变成可装配、可复用、可被 Agent 调用的系统工程。
