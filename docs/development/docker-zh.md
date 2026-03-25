# Docker 镜像

Dimos 使用并行的 Docker 镜像层级结构来支持 ROS 和非 ROS 构建，让你可以根据使用场景选择合适的环境。

## 镜像层级结构

<details><summary>Pikchr</summary>

```pikchr fold output=assets/docker-hierarchy.svg
color = white
fill = none

# Base images
U1: box "ubuntu:22.04" rad 5px fit wid 170% ht 170%
U2: box "ubuntu:22.04" rad 5px fit wid 170% ht 170% at (U1.x + 2.5in, U1.y)

# Labels
text "Non-ROS Track" at (U1.x, U1.y + 0.5in)
text "ROS Track" at (U2.x, U2.y + 0.5in)

# Non-ROS track
arrow from U1.s down 0.4in
P: box "python" rad 5px fit wid 170% ht 170%
arrow from P.s down 0.4in
D: box "dev" rad 5px fit wid 170% ht 170%

# ROS track
arrow from U2.s down 0.4in
R: box "ros" rad 5px fit wid 170% ht 170%
arrow from R.s down 0.4in
RP: box "ros-python" rad 5px fit wid 170% ht 170%
arrow from RP.s down 0.4in
RD: box "ros-dev" rad 5px fit wid 170% ht 170%

# Cross-reference: same dockerfiles reused
line dashed from P.e right 0.3in then down until even with RP then right to RP.w
line dashed from D.e right 0.3in then down until even with RD then right to RD.w
text "same dockerfiles" at (D.e.x + 1.2in, D.e.y + 0.4in)
```

</details>

<!--Result:-->
![output](assets/docker-hierarchy.svg)


## 镜像列表

所有镜像均发布至 `ghcr.io/dimensionalos/`。

| 镜像 | 基础镜像 | 用途 |
|--------------|-----------------------------|----------------------------------------------------|
| `python`     | ubuntu:22.04                | 核心 dimos，包含 Python 依赖，不含 ROS |
| `dev`        | python                      | 开发环境（编辑器、git、pre-commit） |
| `ros`        | ubuntu:22.04                | ROS2 Humble（ROS 2 的一个长期支持版本），包含导航相关包 |
| `ros-python` | ros                         | ROS + dimos Python 依赖 |
| `ros-dev`    | ros-python                  | 完整的 ROS 开发环境 |

## 标签

镜像标签基于 git 分支命名：

| 分支 | 标签 |
|------------------|-------------------------------------------------|
| `main`           | `latest`                                        |
| `dev`            | `dev`                                           |
| feature branches（功能分支） | 经过规范化处理的分支名（例如 `feature_foo_bar`） |

## 各镜像适用场景

### 非 ROS 路线（`python` → `dev`）

```sh skip
docker run -it ghcr.io/dimensionalos/dev:latest bash
```

### ROS 路线（`ros` → `ros-python` → `ros-dev`）

当你需要 ROS2 集成时使用：
- 通过 ROS topic（话题）控制机器人硬件
- 集成导航栈
- 组件之间通过 ROS 消息传递
- 运行 ROS 测试（`pytest -m ros`）

```sh skip
docker run -it ghcr.io/dimensionalos/ros-dev:latest bash
```

## 本地开发

### 在本地构建镜像

使用辅助脚本：

```sh skip
./bin/dockerbuild python    # Build python image
./bin/dockerbuild dev       # Build dev image
./bin/dockerbuild ros       # Build ros image
```

## CI/CD 流水线

[`.github/workflows/docker.yml`](/.github/workflows/docker.yml) 中的工作流负责：

1. **变更检测** - 仅在相关文件发生变更时才重新构建镜像
2. **并行构建** - ROS 和非 ROS 路线独立构建
3. **级联重建** - 基础镜像变更时自动触发下游镜像重建
4. **执行测试** - 在新构建的镜像中运行测试

### 触发路径

| 镜像 | 触发变更的文件路径 |
|----------|------------------------------------------------------|
| `ros`    | `docker/ros/**`、工作流文件                      |
| `python` | `docker/python/**`、`pyproject.toml`、工作流文件 |
| `dev`    | `docker/dev/**`                                      |

### 测试任务

镜像构建完成后，测试并行运行：

| 任务 | 使用镜像 | 执行命令 |
|-------------------------|---------|---------------------------|
| `run-tests`             | dev     | `pytest`                  |
| `run-ros-tests`         | ros-dev | `pytest && pytest -m ros` |
| `run-heavy-tests`       | dev     | `pytest -m heavy`         |
| `run-lcm-tests`         | dev     | `pytest -m lcm`           |
| `run-integration-tests` | dev     | `pytest -m integration`   |
| `run-mypy`              | ros-dev | `mypy dimos`              |

## Dockerfile 结构

### 通用模式

所有 Dockerfile 都接受 `FROM_IMAGE` 构建参数以提供灵活性：

```dockerfile skip
ARG FROM_IMAGE=ubuntu:22.04
FROM ${FROM_IMAGE}
```

这样同一个 Dockerfile（例如 `python`）可以基于不同的基础镜像进行构建。

### Python 包安装

镜像使用 [uv](https://github.com/astral-sh/uv) 实现快速依赖安装：

```dockerfile skip
ENV UV_SYSTEM_PYTHON=1
RUN curl -LsSf https://astral.sh/uv/install.sh | sh
RUN uv pip install '.[misc,cpu,sim,drone,unitree,web,perception,visualization]'
```

### 开发镜像特性

dev 镜像（[`docker/dev/Dockerfile`](/docker/dev/Dockerfile)）额外包含：
- Git、git-lfs、pre-commit
- 编辑器（nano、vim）
- tmux 及自定义配置
- Node.js（通过 nvm 安装）
- 带版本信息的自定义 bash 提示符
- 自动加载 ROS 环境的入口脚本
