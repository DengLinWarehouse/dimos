# Temporal Memory（时间记忆）

这是一个面向实时或回放视频流的 Video RAG（检索增强生成）流水线，可构建以实体为中心的记忆。VLM（Vision Language Model，视觉语言模型）会在滑动窗口中提取证据、跨时间跟踪实体、维护滚动摘要，并将关系持久化到 SQLite 图数据库中，以便在查询时提供上下文。

## 架构

```
color_image (In[Image])
    │
    ▼
FrameWindowAccumulator   ← fps, window_s, stride_s, max_frames_per_window
    │
    ▼
WindowAnalyzer           ← VLM calls: caption + entities + relations
    │
    ├──▶ TemporalState   ← rolling summary, entity roster
    ├──▶ EntityGraphDB   ← persistent SQLite graph (memory/temporal/)
    └──▶ JSONL log       ← per-run raw VLM output (logs/<run>/temporal_memory/)
    └──▶ JSONL dump      ← persistent raw dump (memory/temporal_memory/)
```

共分为五个组件，每个组件都可以独立测试：

| 组件 | 职责 |
|---|---|
| `TemporalMemory(Module)` | 轻量级编排器、RxPY 管线、生命周期管理 |
| `FrameWindowAccumulator` | 有界帧缓冲、滑动窗口提取 |
| `WindowAnalyzer` | 无状态 VLM 调用（窗口分析、摘要、距离估计） |
| `TemporalState` | 线程安全状态：实体清单、滚动摘要、计数器 |
| `EntityGraphDB` | SQLite 持久化：实体、关系、距离 |

## 快速开始

```bash
# With a real robot or simulation
export OPENAI_API_KEY=...
dimos --simulation run unitree-go2-temporal-memory

# With replay data
dimos --replay run unitree-go2-temporal-memory

# Chat with the agent (queries temporal memory)
humancli
```

独立的 `temporal-memory` 组件已在 `all_blueprints.py` 中注册，因此可以与任意相机源组合使用：

```python
from dimos.core.blueprints import autoconnect
from dimos.perception.experimental.temporal_memory import temporal_memory

bp = autoconnect(your_camera_blueprint, temporal_memory())
```

## 配置

所有 VLM 调用频率相关参数都通过 `TemporalMemoryConfig` 暴露，因此你可以在不改动代码的情况下调节成本、延迟与准确率：

```python
from dimos.perception.experimental.temporal_memory import TemporalMemory, TemporalMemoryConfig

config = TemporalMemoryConfig(
    # Frame processing
    fps=1.0,                    # Target frame sampling rate (Hz)
    window_s=5.0,               # Window duration (seconds)
    stride_s=5.0,               # Stride between windows (seconds)
    max_frames_per_window=3,    # Max frames sent to VLM per window
    max_buffer_frames=100,      # Ring buffer capacity

    # VLM call frequencies
    summary_interval_s=30.0,    # Rolling summary update interval
    enable_distance_estimation=True,  # Background distance VLM calls
    max_distance_pairs=5,       # Max entity pairs per distance call
    stale_scene_threshold=5.0,  # Seconds before scene considered stale

    # VLM parameters
    max_tokens=900,             # Max tokens per VLM response
    temperature=0.2,            # VLM temperature

    # Storage
    db_dir=None,                # Persistent DB dir (default: ~/.local/state/dimos/temporal_memory/)
    new_memory=False,           # Clear persistent DB on start

    # Visualization
    visualize=True,             # Rerun entity graph (GraphNodes + GraphEdges)

    # CLIP filtering
    use_clip_filtering=True,    # Filter duplicate/static frames via CLIP
    clip_model="ViT-B/32",     # CLIP model name

    # Graph context
    max_relations_per_entity=10,  # Max relations returned per entity query
    nearby_distance_meters=5.0,   # Threshold for "nearby" in distance queries
)

bp = TemporalMemory.blueprint(config=config)
```

如果没有显式传入 VLM，则会根据 `OPENAI_API_KEY` 自动创建一个实例。

## 存储

有两类输出，彼此不重叠：

| 输出 | 位置 | 生命周期 | 内容 |
|---|---|---|---|
| JSONL 日志 | `logs/<run>/temporal_memory/temporal_memory.jsonl` | 单次运行 | 原始 VLM 文本 + 解析后的 JSON（可直接 grep） |
| JSONL 导出 | `~/.local/state/dimos/temporal_memory/temporal_memory.jsonl` | 持久化 | 跨所有运行累计的原始 VLM 输出 |
| SQLite DB | `~/.local/state/dimos/temporal_memory/entity_graph.db` | 持久化 | 实体、关系、距离 |

- **JSONL** 会逐字保存每次 VLM 响应（`raw_response` 字段）以及解析后的结构化数据。智能体可以直接对自然语言文本执行 grep。
- **SQLite DB** 会跨运行持续保留。传入 `new_memory=True` 可在启动时清空。
- 可通过设置 `db_dir=` 覆盖持久化数据库的位置。
- 启动时会记录这两个路径，便于你在日志中找到它们。

## VLM 调用预算

在默认设置下（`window_s=5, stride_s=5, summary_interval_s=30`）：

- **窗口分析：** 每 5 秒 1 次调用 = 12 次/分钟
- **滚动摘要：** 每 30 秒 1 次调用 = 2 次/分钟
- **距离估计：** 约每个窗口 1 次调用（若启用）= 12 次/分钟
- **稳态总计：** 约 26 次 VLM 调用/分钟

可通过增大 `stride_s` 和 `summary_interval_s`，或禁用距离估计（`enable_distance_estimation=False`）来降低成本。

## 测试

```bash
# Unit tests (29 tests, mocked VLMs, no API key needed)
DISPLAY=:99 python -m pytest dimos/perception/experimental/temporal_memory/test_temporal_memory_module.py -v -c /dev/null

# Integration test with real VLM
export OPENAI_API_KEY=...
DISPLAY=:99 python -m pytest dimos/perception/experimental/temporal_memory/test_temporal_memory_module.py -v -c /dev/null -k "integration" --slow
```
