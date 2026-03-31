# DimOS Manifest JSON Schema 方案设计

## 1. 目标

本文只讨论 `Manifest JSON Schema` 的方案设计，不涉及具体实现代码。

目标是定义一份稳定的数据契约，使云端发布系统、机器人侧 Loader、以及 DimOS 本地运行时能够围绕同一份 Manifest 协同工作。

## 2. 设计目标

Manifest Schema 应满足以下要求：

- 可校验：能够在下发前和加载前做严格校验
- 可复现：同一份 Manifest 应导向相同部署结果
- 可审计：必须保留版本、来源、时间和发布标识
- 可约束：必须能表达目标设备、环境和兼容性边界
- 可回滚：必须能描述上一个稳定版本和回滚策略
- 可扩展：未来新增字段时不破坏已有协议

## 3. 顶层结构

建议顶层包含以下对象：

- `manifest_version`
- `release_id`
- `created_at`
- `target`
- `runtime`
- `entrypoint`
- `global_config`
- `modules`
- `assets`
- `healthcheck`
- `rollback`
- `security`
- `metadata`

## 4. 推荐 Schema 骨架

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.company/dimos/manifest-1.0.schema.json",
  "title": "DimOS Release Manifest",
  "type": "object",
  "required": [
    "manifest_version",
    "release_id",
    "created_at",
    "target",
    "runtime",
    "entrypoint",
    "global_config",
    "modules",
    "healthcheck",
    "rollback",
    "security"
  ],
  "properties": {
    "manifest_version": { "type": "string" },
    "release_id": { "type": "string" },
    "created_at": { "type": "string", "format": "date-time" },
    "target": { "$ref": "#/$defs/target" },
    "runtime": { "$ref": "#/$defs/runtime" },
    "entrypoint": { "$ref": "#/$defs/entrypoint" },
    "global_config": { "type": "object" },
    "modules": {
      "type": "array",
      "minItems": 1,
      "items": { "$ref": "#/$defs/moduleArtifact" }
    },
    "assets": {
      "type": "array",
      "default": [],
      "items": { "$ref": "#/$defs/assetArtifact" }
    },
    "healthcheck": { "$ref": "#/$defs/healthcheck" },
    "rollback": { "$ref": "#/$defs/rollback" },
    "security": { "$ref": "#/$defs/security" },
    "metadata": {
      "type": "object",
      "default": {}
    }
  }
}
```

## 5. 字段设计建议

### 5.1 `manifest_version`

用途：标识 Schema 版本。  
建议：语义化版本，例如 `1.0`。

约束：

- 必填
- 由 Loader 决定是否支持该版本
- 不应和 `dimos_version` 混用

### 5.2 `release_id`

用途：标识一次完整发布。  
示例：`go2-prod-2026.03.31.001`

约束：

- 必填
- 全局唯一
- 用于审计、回滚、追踪

### 5.3 `created_at`

用途：记录 Manifest 生成时间。

约束：

- 必填
- 必须是 RFC3339 / ISO8601 时间字符串

### 5.4 `target`

用途：定义这份 Manifest 面向哪些设备。

建议字段：

- `robot_type`
- `device_id`
- `arch`
- `os`
- `labels`

示例：

```json
{
  "robot_type": "unitree_go2",
  "device_id": "go2-001",
  "arch": "aarch64",
  "os": "ubuntu22.04",
  "labels": ["prod", "cn-shanghai"]
}
```

约束：

- `robot_type` 建议必填
- `device_id` 可选，支持定向发布
- `arch` 与 `os` 用于兼容性校验

### 5.5 `runtime`

用途：约束本地运行环境。

建议字段：

- `dimos_min`
- `dimos_max`
- `python`
- `gpu_required`
- `transport_required`

约束：

- DimOS 版本必须落在允许范围内
- `gpu_required=true` 时，设备必须具备对应能力

### 5.6 `entrypoint`

用途：描述系统从哪个蓝图起装配。

建议字段：

- `blueprint`
- `disabled_modules`
- `remappings`

约束：

- `blueprint` 必填
- `disabled_modules` 默认为空数组
- `remappings` 必须满足目标模块存在

### 5.7 `global_config`

用途：承接运行时全局配置。

说明：

- 本对象可映射到 DimOS `GlobalConfig`
- 建议只允许显式白名单字段

建议字段举例：

- `robot_ip`
- `viewer`
- `simulation`
- `replay`
- `n_workers`
- `mcp_port`

约束：

- 不建议把敏感 secret 放进这里
- 布尔、整型、字符串类型要与本地配置模型兼容

### 5.8 `modules`

用途：定义需要分发和装配的模块发布包。

建议每项字段：

- `name`
- `type`
- `source`
- `version`
- `digest` 或 `sha256`
- `required`
- `config`
- `dependencies`

示例：

```json
{
  "name": "perception.object_tracking",
  "type": "docker",
  "source": "registry.company/dimos/object-tracking:1.3.4",
  "version": "1.3.4",
  "digest": "sha256:abc",
  "required": true,
  "dependencies": [],
  "config": {}
}
```

约束：

- `name` 必填
- `type` 只允许 `docker`, `wheel`
- `source` 必填
- 必须存在版本和校验字段
- `required=false` 的模块允许失败但不影响整体启动

### 5.9 `assets`

用途：定义非代码资源。

建议每项字段：

- `name`
- `type`
- `uri`
- `version`
- `sha256`
- `mount_to`

约束：

- `type` 建议固定为 `blob`
- `uri` 必填
- 必须可校验
- 目标挂载路径必须明确

### 5.10 `healthcheck`

用途：定义“启动成功”的判定规则。

建议字段：

- `startup_timeout_sec`
- `required_modules`
- `required_streams`
- `required_rpc`
- `custom_checks`

约束：

- `startup_timeout_sec` 必须大于 0
- 关键模块和关键 RPC 不得为空时缺失

### 5.11 `rollback`

用途：定义回滚策略。

建议字段：

- `previous_stable_release`
- `allow_auto_rollback`
- `max_failures`

约束：

- `allow_auto_rollback=true` 时，应有稳定版本可回退
- 该对象只描述策略，不直接保存执行结果

### 5.12 `security`

用途：定义校验和安全边界。

建议字段：

- `manifest_signature`
- `trusted_registries`
- `secrets_profile`

约束：

- `manifest_signature` 必填
- `trusted_registries` 至少一个
- secret 内容不能内嵌在 Manifest 中

### 5.13 `metadata`

用途：留给扩展字段。

建议内容：

- 发布人
- 发布说明
- 环境标签
- 工单号

约束：

- 不参与核心部署逻辑
- 可作为审计辅助信息

## 6. 建议的 `$defs` 子结构

建议至少定义以下子结构：

- `target`
- `runtime`
- `entrypoint`
- `moduleArtifact`
- `assetArtifact`
- `healthcheck`
- `rollback`
- `security`

这样可以：

- 便于维护
- 便于复用
- 便于以后拆分多版本 schema

## 7. 推荐校验规则

除了类型校验，建议加入业务规则校验。

### 7.1 静态校验

- 顶层必填字段必须存在
- 数组字段类型必须正确
- 时间格式必须正确
- 枚举值必须合法

### 7.2 兼容性校验

- `target.arch` 与设备架构一致
- `runtime.dimos_min/max` 与本地版本兼容
- `gpu_required=true` 时设备需具备 GPU

### 7.3 发布包校验

- `modules[].digest/sha256` 必须存在
- `assets[].sha256` 必须存在
- 所有 `source/uri` 必须在白名单范围内

### 7.4 启动前校验

- `entrypoint.blueprint` 在本地可解析
- `disabled_modules` 指向有效模块名
- `remappings` 结构合法
- `healthcheck.required_rpc` 对应的 RPC 具备可检查性

## 8. 推荐的演进策略

Manifest Schema 建议按版本演进。

推荐规则：

- 新增字段尽量向后兼容
- 删除字段需要升级 `manifest_version`
- Loader 应支持至少两个相邻主版本
- 不兼容变更必须明确拒绝加载

## 9. 推荐的错误分类

Schema 校验失败时，建议把错误分为三类：

- `SchemaError`
  - 结构不合法
- `CompatibilityError`
  - 本地环境不兼容
- `ArtifactError`
  - 发布包信息不完整或不可信

这样便于：

- 本地决策是否回滚
- 云端统计失败原因
- 运维快速定位问题

## 10. 示例：精简版可用 Manifest

```json
{
  "manifest_version": "1.0",
  "release_id": "go2-prod-2026.03.31.001",
  "created_at": "2026-03-31T10:00:00Z",
  "target": {
    "robot_type": "unitree_go2",
    "arch": "aarch64",
    "os": "ubuntu22.04"
  },
  "runtime": {
    "dimos_min": "0.0.11",
    "dimos_max": "0.0.x",
    "python": "3.12"
  },
  "entrypoint": {
    "blueprint": "unitree-go2-agentic-mcp",
    "disabled_modules": [],
    "remappings": []
  },
  "global_config": {
    "robot_ip": "192.168.123.161",
    "viewer": "rerun",
    "n_workers": 8
  },
  "modules": [
    {
      "name": "perception.object_tracking",
      "type": "docker",
      "source": "registry.company/dimos/object-tracking:1.3.4",
      "version": "1.3.4",
      "digest": "sha256:abc",
      "required": true,
      "config": {}
    }
  ],
  "healthcheck": {
    "startup_timeout_sec": 60,
    "required_modules": ["Agent"],
    "required_streams": [],
    "required_rpc": []
  },
  "rollback": {
    "previous_stable_release": "go2-prod-2026.03.20.002",
    "allow_auto_rollback": true
  },
  "security": {
    "manifest_signature": "base64-signature",
    "trusted_registries": ["registry.company"],
    "secrets_profile": "go2-prod"
  }
}
```

## 11. 结论

Manifest JSON Schema 设计本质上是：

- 定义一次发布的标准结构
- 约束哪些字段必须存在
- 规定机器人侧在加载前如何校验
- 保证云端发布与本地运行之间的契约稳定

它属于方案设计，不是实现代码，但会直接决定后续 Loader、发布系统和回滚机制能否稳定落地。


