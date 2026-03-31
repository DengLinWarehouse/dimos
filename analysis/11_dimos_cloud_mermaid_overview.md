# DimOS 云端化方案 Mermaid 总览

## 1. 文档目标

本文档用 Mermaid 图把 DimOS 云端化方案完整表达出来。

原则：

- 图完整
- 逻辑清晰
- 文字简洁
- 适合汇报和总览

## 2. 方案一句话

> 用户或业务系统决定机器人需要什么能力，云端负责管理和分发目标状态，机器人本地 Loader 负责拉取、校验、缓存、启动、健康检查和回滚，DimOS Runtime 负责真正运行机器人系统。

## 3. 总体架构图

```mermaid
flowchart TB
    subgraph Cloud[云端控制层]
        CP[Cloud Control Plane]
        DS[Device Service]
        CS[Config Service]
        RS[Release Service]
        PS[Package Registry / OCI Registry]
        OS[Object Storage]
        AS[Audit Service]
        RLS[Rollout Service]
    end

    subgraph Edge[机器人本地控制层]
        Loader[Robot Loader Agent]
        Cache[Local Cache]
        State[Loader StateStore]
        Health[Health Checker]
        Rollback[Rollback Manager]
    end

    subgraph Runtime[本地运行时层]
        DRuntime[DimOS Runtime]
        BP[Blueprint]
        MC[ModuleCoordinator]
        Mods[Modules]
    end

    subgraph HW[机器人硬件层]
        Sensors[传感器]
        Actuators[执行器]
        Robot[机器人本体]
    end

    CP --> DS
    CP --> CS
    CP --> RS
    CP --> RLS
    CP --> AS
    RS --> PS
    RS --> OS
    CS --> RS

    Loader --> CS
    Loader --> RS
    Loader --> PS
    Loader --> OS
    Loader --> AS
    Loader --> Cache
    Loader --> State
    Loader --> Health
    Loader --> Rollback
    Loader --> DRuntime

    DRuntime --> BP
    BP --> MC
    MC --> Mods

    Mods --> Sensors
    Mods --> Actuators
    Sensors --> Robot
    Actuators --> Robot
```

说明：

- 云端不做实时控制
- Loader 是本地部署控制器
- Runtime 是本地真实运行时

## 4. 核心对象关系图

```mermaid
flowchart TD
    Device[Device\n设备]
    Manifest[Manifest\n运行声明]
    Release[Release\n发布版本]
    PackageRef[PackageRef\n模块发布包 / 资源引用]
    Deploy[DeploymentRecord\n设备部署记录]
    LoaderState[LoaderState\n本地状态]
    Audit[AuditEvent\n审计事件]
    Rollout[RolloutPlan\n灰度发布计划]

    Release --> Manifest
    Release --> PackageRef
    Release --> Rollout
    Device --> Deploy
    Release --> Deploy
    Device --> LoaderState
    Manifest --> LoaderState
    PackageRef --> LoaderState
    Deploy --> Audit
    Rollout --> Audit
    LoaderState --> Audit
```

说明：

- `Manifest` 定义怎么运行
- `Release` 定义发布哪个版本
- `LoaderState` 保证本地可恢复
- `DeploymentRecord` 记录设备级结果

## 5. 配置与发布包模型图

```mermaid
flowchart LR
    Manifest[Manifest]
    Entry[Entrypoint\nblueprint / remappings / disabled_modules]
    GC[GlobalConfig\nrobot_ip / viewer / simulation / replay]
    Mods[modules]
    Assets[assets]
    HC[healthcheck]
    RB[rollback]
    SEC[security]

    Manifest --> Entry
    Manifest --> GC
    Manifest --> Mods
    Manifest --> Assets
    Manifest --> HC
    Manifest --> RB
    Manifest --> SEC

    Mods --> OCI[OCI Image]
    Mods --> Wheel[Python wheel]
    Assets --> Blob[Model / Map / URDF / Data Blob]
```

说明：

- Manifest 是云端与本地之间的核心契约
- 发布包和资源都通过 Manifest 引用

## 6. 发布包存储映射图

```mermaid
flowchart TD
    Package[模块发布包 / 资源]

    Package --> OCIImg[OCI 镜像]
    Package --> WheelPkg[Python wheel 包]
    Package --> DataBlob[模型 / 数据 Blob]

    OCIImg --> OCIReg[OCI Registry]
    WheelPkg --> ObjStore[对象存储 或 Package Registry]
    DataBlob --> ObjStore

    Meta[元数据] --> DB[PostgreSQL]
    AuditLog[审计 / 日志] --> LogStore[Loki / Elasticsearch / ClickHouse / S3]
```

说明：

- 元数据和实际发布包分层存储
- Runtime 不直接访问云端仓库，由 Loader 统一准备

## 7. 端到端发布主流程

```mermaid
sequenceDiagram
    participant User as 用户 / 业务系统
    participant CP as Cloud Control Plane
    participant Store as Config / Package Storage
    participant Loader as Robot Loader
    participant Cache as Local Cache
    participant RT as DimOS Runtime
    participant Audit as Audit Service

    User->>CP: 确定目标能力 / 目标配置
    CP->>Store: 写入 Manifest 与发布包引用
    CP->>Audit: 记录发布动作
    Loader->>CP: 查询当前设备目标 Release
    CP-->>Loader: 返回 Manifest + PackageRef
    Loader->>Store: 拉取缺失发布包与资源
    Store-->>Loader: 返回镜像 / wheel / Blob
    Loader->>Cache: 写入本地缓存
    Loader->>Loader: 校验结构、版本、digest、兼容性
    Loader->>RT: 启动或重启 DimOS
    RT-->>Loader: 返回启动结果
    Loader->>Loader: 健康检查
    alt 通过
        Loader->>Loader: 标记 Stable
        Loader->>Audit: 上报 stable
    else 失败
        Loader->>Loader: 触发回滚
        Loader->>Audit: 上报 rollback
    end
```

说明：

- 目标状态来源于用户 / 业务系统
- 云端负责托管和分发
- 本地负责真实执行和恢复

## 8. 本地 Loader 状态机

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Polling
    Polling --> Validating: 发现新 Manifest / Release
    Validating --> Downloading: 校验通过
    Validating --> Failed: 校验失败
    Downloading --> Staging: 下载完成
    Downloading --> Failed: 下载失败
    Staging --> Launching: 本地准备完成
    Launching --> HealthChecking: Runtime 已启动
    Launching --> Rollback: 启动失败
    HealthChecking --> Stable: 健康检查通过
    HealthChecking --> Rollback: 健康检查失败
    Rollback --> Stable: 回滚成功
    Rollback --> Failed: 回滚失败
    Stable --> Polling: 持续轮询
    Failed --> Polling: 等待下次重试
```

说明：

- 没通过校验不能运行
- 没通过健康检查不能标记 Stable
- 回滚必须优先依赖本地缓存

## 9. 本地执行边界图

```mermaid
flowchart LR
    Cloud[云端控制面] --> Loader[本地 Loader]
    Loader --> Runtime[DimOS Runtime]
    Runtime --> Modules[Module 集合]
    Modules --> Hardware[机器人硬件]

    Cloud -. 不直接控制 .-> Hardware
    Cloud -. 不直接下载到 Runtime .-> Runtime
    Runtime -. 不直接访问云端仓库 .-> Cloud
```

说明：

- 云端不能越过 Loader 直接控制运行时
- Runtime 不能承担供应链职责

## 10. 云端控制面分层图

```mermaid
flowchart TB
    Portal[管理门户 / API]
    ConfigCenter[配置中心]
    ReleaseCenter[发布中心]
    RolloutCenter[灰度发布中心]
    AuditCenter[审计中心]
    DeviceCenter[设备中心]

    Portal --> ConfigCenter
    Portal --> ReleaseCenter
    Portal --> RolloutCenter
    Portal --> AuditCenter
    Portal --> DeviceCenter

    ConfigCenter --> ManifestRepo[Manifest Repository]
    ReleaseCenter --> ReleaseRepo[Release Repository]
    RolloutCenter --> RolloutRepo[Rollout Repository]
    AuditCenter --> AuditRepo[Audit Repository]
    DeviceCenter --> DeviceRepo[Device Repository]
```

说明：

- 配置、发布、灰度、审计、设备管理建议分层建设

## 11. 灰度发布与熔断流程图

```mermaid
flowchart TD
    A[创建 Release] --> B[创建 RolloutPlan]
    B --> C[Canary 批次]
    C --> D[统计成功率 / 失败率 / 回滚率]
    D --> E{是否通过阈值}
    E -- Yes --> F[Pilot 批次]
    E -- No --> G[暂停或熔断]
    F --> H[Staged 批次]
    H --> I[Full 批次]
    I --> J[发布完成]
    G --> K[人工分析或批量回滚]
```

说明：

- 灰度是阶段 6 能力，不应早于基础闭环建设

## 12. 审计事件流图

```mermaid
flowchart LR
    ManifestCreated[manifest.created]
    ReleaseCreated[release.created]
    ReleaseActivated[release.activated]
    DeployStarted[deployment.started]
    DeploySucceeded[deployment.succeeded]
    DeployFailed[deployment.failed]
    RollbackStarted[rollback.started]
    RollbackSucceeded[rollback.succeeded]
    RolloutPaused[rollout.paused]
    RolloutAborted[rollout.aborted]
    RolloutCompleted[rollout.completed]

    ManifestCreated --> AuditStore[Audit Store]
    ReleaseCreated --> AuditStore
    ReleaseActivated --> AuditStore
    DeployStarted --> AuditStore
    DeploySucceeded --> AuditStore
    DeployFailed --> AuditStore
    RollbackStarted --> AuditStore
    RollbackSucceeded --> AuditStore
    RolloutPaused --> AuditStore
    RolloutAborted --> AuditStore
    RolloutCompleted --> AuditStore
```

说明：

- 审计需要覆盖配置、发布、部署、回滚、灰度全过程

## 13. 分阶段实施路线图

```mermaid
flowchart LR
    P1[阶段1\n数据契约与术语统一]
    P2[阶段2\n云端配置中心]
    P3[阶段3\n本地 Loader 最小闭环]
    P4[阶段4\n模块发布包分发]
    P5[阶段5\n发布 / 回滚控制面]
    P6[阶段6\n灰度发布与运维审计]

    P1 --> P2
    P2 --> P3
    P3 --> P4
    P4 --> P5
    P5 --> P6
```

说明：

- 正确顺序是先配置，再本地闭环，再分发，再控制面，最后灰度与审计

## 14. MVP 最小落地路径

```mermaid
flowchart TD
    A[定义 Manifest] --> B[建设配置中心]
    B --> C[本地 Loader 拉取配置]
    C --> D[本地启动 DimOS]
    D --> E[健康检查]
    E --> F{成功?}
    F -- Yes --> G[标记 Stable]
    F -- No --> H[回滚旧版本]
```

说明：

- 这是最关键的第一条闭环
- 先打通这一条，再扩展发布包和控制面

## 15. 最终目标图景

```mermaid
flowchart TB
    LocalRuntime[可本地运行]
    ConfigManaged[可云端配置管理]
    PackageManaged[可发布包管理]
    ReleaseManaged[可统一发布管理]
    Rollbackable[可本地自动回滚]
    Auditable[可全过程审计]
    Scalable[可灰度与规模化运维]

    LocalRuntime --> ConfigManaged
    ConfigManaged --> PackageManaged
    PackageManaged --> ReleaseManaged
    ReleaseManaged --> Rollbackable
    Rollbackable --> Auditable
    Auditable --> Scalable
```

一句话总结：

> ##### DimOS 云端化不是把机器人运行搬到云端，而是把云端管理能力叠加到本地可靠运行之上，形成“可配置、可发布、可回滚、可审计、可灰度”的机器人运行平台。


