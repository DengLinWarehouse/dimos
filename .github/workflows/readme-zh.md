# 工作流（workflows）总体结构

Docker.yml 会检测相关文件变更并重建所需镜像。  
当前镜像存在依赖链：ros → python → dev（未来可能变为树形并可分叉）。

在 dev 镜像之上运行测试。  
dev 镜像也是开发者通过 devcontainers 在 IDE 中使用的环境。  
https://code.visualstudio.com/docs/devcontainers/containers

# 登录 GitHub 容器仓库

创建个人访问令牌（classic，非 fine-grained）  
https://github.com/settings/tokens

添加权限：
- `read:packages`：下载容器镜像并读取其元数据。

以及可选：

- `write:packages`：下载与上传容器镜像，并读写其元数据。
- `delete:packages`：删除容器镜像。

更多信息见 https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry

通过以下方式登录 docker：

```bash
echo TOKEN | docker login ghcr.io -u GITHUB_USER --password-stdin
```

拉取 dev 镜像（dev 分支）

```bash
docker pull ghcr.io/dimensionalos/dev:dev
```

拉取 dev 镜像（master）

```bash
docker pull ghcr.io/dimensionalos/dev:latest
```

# 待办

当前在保证 Docker 镜像构建顺序正确与跳过不必要重建两方面存在问题。

（需要为构建任务设置 job 依赖，使上层镜像等待下层镜像构建完成，例如 py 等待 ros。）  
默认情况下若父任务被跳过，子任务也会被跳过，除非其条件中包含 `always()`。

问题是一旦在条件中加入 `always()`，似乎无论同条件内其他检查如何，任务都会运行。  
因此目前无法跳过 python（及更上层）的构建。需要再评审。

我认为可能需要用 Python 编写自建构建调度器，由它调用用于构建镜像的 GitHub workflows。
