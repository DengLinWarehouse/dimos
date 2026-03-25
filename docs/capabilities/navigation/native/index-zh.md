# Go2 非 ROS 导航

<img src="assets/noros_nav.gif" width="100%">

Go2 导航栈完全不依赖 ROS 运行。它使用**柱状裁剪体素地图（column-carving voxel map）**策略：每一帧新的 LiDAR 数据会完整替换全局地图中对应的区域，确保地图始终反映最新的观测结果。

## 数据流

<details>
<summary>图表源码</summary>

```pikchr fold output=assets/go2nav_dataflow.svg
color = white
fill = none

Go2: box "Go2" rad 5px fit wid 170% ht 170%
arrow right 0.5in
Vox: box "VoxelGridMapper" rad 5px fit wid 170% ht 170%
arrow right 0.5in
Cost: box "CostMapper" rad 5px fit wid 170% ht 170%
arrow right 0.5in
Nav: box "Navigation" rad 5px fit wid 170% ht 170%

M1: dot at 1/2 way between Go2.e and Vox.w invisible
text "PointCloud2" italic at (M1.x, Go2.n.y + 0.15in)

M2: dot at 1/2 way between Vox.e and Cost.w invisible
text "PointCloud2" italic at (M2.x, Vox.n.y + 0.15in)

M3: dot at 1/2 way between Cost.e and Nav.w invisible
text "OccupancyGrid" italic at (M3.x, Cost.n.y + 0.15in)

arrow dashed from Nav.s down 0.3in then left until even with Go2.s then to Go2.s
M4: dot at 1/2 way between Go2.s and Nav.s invisible
text "Twist" italic at (M4.x, Nav.s.y - 0.45in)
```

</details>

<!--Result:-->
![output](assets/go2nav_dataflow.svg)
## 流水线步骤

### 1. LiDAR 帧 — [`GO2Connection`](/dimos/robot/unitree/go2/connection.py)

我们不直接连接 LiDAR——而是使用 Unitree 的 WebRTC 客户端（通过 [legion 的 webrtc 驱动](https://github.com/legion1581/unitree_webrtc_connect)），它传输的是经过大量预处理的 5cm 体素网格，而非原始点云数据。这使我们能够开箱即用地支持原装未越狱的 Go2 Air 和 Pro 型号。

![LiDAR 帧](assets/1-lidar.png)

### 2. 全局体素地图 — [`VoxelGridMapper`](/dimos/mapping/voxels.py)

[`VoxelGridMapper`](/dimos/mapping/voxels.py) 使用 Open3D 的 `VoxelBlockGrid`（由哈希表支持）维护一个稀疏 3D 占用栅格。默认情况下每个体素是一个 5cm 的立方体。

体素哈希表提供 O(1) 的插入/删除/查找性能，因此即使有数百万个体素也能高效运行。栅格默认在 **CUDA** 上运行以提高速度，并支持 CPU 回退。

每个传入的 LiDAR 帧通过柱状裁剪拼接到全局地图中。我们将收到的 LiDAR 帧空间中所有之前已映射的体素视为过时数据，通过擦除覆盖区域内的整个 Z 列来保证：

- 不会出现来自之前扫描的幽灵障碍物
- 动态物体（人、门）会被自动清除
- 最新观测始终优先

我们没有正式的回环检测和稳定的里程计，我们信任 Go2 里程计报告的数据，虽然这些数据出人意料地稳定但最终会产生漂移。你可以可靠地建图并导航通过非常大的空间（在我们的测试中达到 500 平方米），但无法沿着街道导航到超市。

#### 配置

| 参数 | 默认值 | 描述 |
|------|--------|------|
| `voxel_size` | 0.05 | 体素立方体尺寸（米） |
| `block_count` | 2,000,000 | 哈希表中最大体素数 |
| `device` | `CUDA:0` | 计算设备（`CUDA:0` 或 `CPU:0`） |
| `carve_columns` | `true` | 启用柱状裁剪（禁用则为仅追加建图） |
| `publish_interval` | 0 | 地图发布间隔秒数（0 = 每帧发布） |

![全局地图](assets/2-globalmap.png)

### 3. 全局代价地图 — [`CostMapper`](/dimos/mapping/costmapper.py)

[`CostMapper`](/dimos/mapping/costmapper.py) 将 3D 体素地图转换为 2D 占用栅格。默认算法（`height_cost`）映射 Z 轴变化率，并进行一些平滑处理。

算法设置在 [`occupancy.py`](/dimos/mapping/pointclouds/occupancy.py) 中，可按机器人进行配置。

#### 配置

```python skip
@dataclass(frozen=True)
class HeightCostConfig(OccupancyConfig):
    """基于高度代价的占用配置（地形坡度分析）。"""
    can_pass_under: float = 0.6
    can_climb: float = 0.15
    ignore_noise: float = 0.05
    smoothing: float = 1.0
```

| 代价值 | 含义 |
|--------|------|
| 0 | 平坦，易于通行 |
| 50 | 中等坡度（Go2 每格约 7.5cm 高差） |
| 100 | 陡峭或不可通行（Go2 每格 ≥15cm 高差） |
| -1 | 未知（无观测数据） |

![全局代价地图](assets/3-globalcostmap.png)

### 4. 导航代价地图 — [`ReplanningAStarPlanner`](/dimos/navigation/replanning_a_star/module.py)

规划器会处理地形梯度并计算自身算法相关的代价地图，优先选择安全的自由路径，同时在必要时愿意激进地穿过狭窄空间。

我们在持续循环中运行规划器，因此它会动态响应遇到的障碍物。

![带路径的导航代价地图](assets/4-navcostmap.png)

### 5. 所有层叠加

所有可视化层一起显示。

![所有层](assets/5-all.png)

## 蓝图组合

导航栈在 [`unitree_go2`](/dimos/robot/unitree/go2/blueprints/__init__.py) 蓝图中组合：

```python fold output=assets/go2_blueprint.svg
from dimos.core.blueprints import autoconnect
from dimos.core.introspection import to_svg
from dimos.mapping.costmapper import cost_mapper
from dimos.mapping.voxels import voxel_mapper
from dimos.navigation.frontier_exploration import wavefront_frontier_explorer
from dimos.navigation.replanning_a_star.module import replanning_a_star_planner
from dimos.robot.unitree.go2.blueprints.basic.unitree_go2_basic import unitree_go2_basic

unitree_go2 = autoconnect(
    unitree_go2_basic,                    # 机器人连接 + 可视化
    voxel_mapper(voxel_size=0.05),        # 3D 体素建图
    cost_mapper(),                        # 2D 代价地图生成
    replanning_a_star_planner(),          # 路径规划
    wavefront_frontier_explorer(),        # 探索
).global_config(n_workers=6, robot_model="unitree_go2")

to_svg(unitree_go2, "assets/go2_blueprint.svg")
```

<!--Result:-->
![output](assets/go2_blueprint.svg)
