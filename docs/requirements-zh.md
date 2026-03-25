# 系统要求

## 硬件

| 组件 | 最低配置 | 推荐配置 |
|-----------|---------|-------------|
| GPU | NVIDIA RTX 3000+（8 GB 显存） | RTX 4070+（12 GB+ 显存） |
| CPU | 8 核 Intel / AMD | 12+ 核 |
| 内存 | 16 GB | 32 GB+ |
| 磁盘 | 10 GB SSD | 25 GB+ SSD |
| 操作系统 | Ubuntu 22.04, macOS 12.6+ | Ubuntu 24.04 |

> GPU 对于基本的机器人控制是可选的。感知、VLM（视觉语言模型）和 AI 功能则需要 GPU。

## 已测试配置

| 配置 | GPU | CPU | 内存 | 状态 |
|--------|-----|-----|-----|--------|
| 开发工作站 | RTX 4090 (24 GB) | i9-13900K | 64 GB | ✅ 主力开发 |
| 中端配置 | RTX 4070 (12 GB) | i7-12700 | 32 GB | ✅ 已测试 |
| 笔记本 | RTX 4060 Mobile (8 GB) | i7-13700H | 16 GB | ✅ 已测试 |
| 无头服务器 | 无 GPU | Xeon | 32 GB | ✅ 仅控制 |
| Jetson AGX Orin | Orin (32 GB 共享) | ARM A78AE | 32 GB | ✅ 已测试 |
| Jetson Orin Nano | Orin (8 GB 共享) | ARM A78AE | 8 GB | 🟧 实验性 |

## 依赖层级

直接 `pip install dimos` 安装 **core**（核心）层级。通过 extras 添加更多功能。

```bash
pip install dimos                           # 仅核心
pip install 'dimos[base,unitree]'             # 完整栈 + Unitree
pip install 'dimos[base,unitree,sim]'         # + MuJoCo 仿真
pip install 'dimos[base,unitree,drone]'       # + 无人机支持
pip install 'dimos[base,unitree,manipulation]' # + 机械臂控制
```

| Extra | 添加功能 | 关键包 | 需要 GPU？ |
|-------|-------------|--------------|------|
| *（core）* | 传输层、数据流、CLI、蓝图、占用地图 | dimos-lcm, numpy, scipy, opencv, open3d, numba, Pinocchio, typer, textual | 否 |
| `agents` | LLM 智能体、语音、工具调用 | langchain, openai, whisper, anthropic | 否 |
| `perception` | 物体检测、VLM、追踪 | ultralytics, transformers, moondream | **是** |
| `visualization` | Rerun 查看器 + 桥接 | rerun-sdk, dimos-viewer | 否 |
| `web` | FastAPI Web 界面、音频 | fastapi, uvicorn, ffmpeg-python | 否 |
| `sim` | MuJoCo 仿真 | mujoco, playground, pygame | 否 |
| `unitree` | Unitree Go2 / G1 支持 | unitree-webrtc-connect | 否 |
| `drone` | DJI Tello / MAVLink 无人机 | pymavlink | 否 |
| `manipulation` | 机械臂规划 + 控制 | Drake, piper-sdk, xarm-sdk | 否 |
| `cuda` | GPU 加速 | cupy, onnxruntime-gpu, xformers | **是** |
| `cpu` | CPU 推理后端 | onnxruntime, ctransformers | 否 |
| `misc` | 额外模型、嵌入向量、硬件 SDK | cerebras, edgetam, sentence-transformers, tiktoken | 视情况 |
| `docker` | Docker 辅助模块的最小集 | dimos-lcm, numpy, opencv-headless, rerun-sdk | 否 |
| `base` | 全功能集（agents + web + perception + viz + sim） | 以上所有 | **是** |
| `dev` | 代码检查、测试、类型存根 | ruff, mypy, pytest, pre-commit | 否 |
| `psql` | PostgreSQL 存储 | psycopg2 | 否 |
| `dds` | DDS 传输层（CycloneDDS） | dev + cyclonedds | 否 |

## 无头/服务器环境

如果在无头 Ubuntu 服务器（无显示器）上运行，需安装 OpenGL 库以满足可视化依赖：

```bash
sudo apt-get install -y libgl1 libegl1
```

Nix 用户（`nix develop`）无需此步骤——flake 已提供 `libGL`、`libGLU` 和 `mesa`。
