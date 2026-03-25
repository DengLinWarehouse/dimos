# WebSocket Visualization Module（WebSocket 可视化模块）

`WebsocketVisModule` 提供实时数据，用于在 Foxglove 中对机器人进行可视化和控制（参见 `dimos/web/command-center-extension/README.md`）。

## 概览

可视化：

- 机器人位置和朝向
- 导航路径
- 代价地图

控制：

- 设置导航目标
- 设置 GPS 位置目标
- 键盘遥操作（WASD）
- 触发探索

## 它提供的内容

### 输入（订阅的话题）
- `robot_pose` (`PoseStamped`)：机器人当前的位置和朝向
- `gps_location` (`LatLon`)：机器人的 GPS 坐标
- `path` (`Path`)：规划得到的导航路径
- `global_costmap` (`OccupancyGrid`)：用于可视化的全局代价地图

### 输出（发布的话题）
- `click_goal` (`PoseStamped`)：用户在 Web 界面点击设置的目标位置
- `gps_goal` (`LatLon`)：通过界面设置的 GPS 目标坐标
- `explore_cmd` (`Bool`)：启动自主探索的命令
- `stop_explore_cmd` (`Bool`)：停止探索的命令
- `movecmd` (`Twist`)：来自界面的直接移动命令
- `movecmd_stamped` (`TwistStamped`)：带时间戳的移动命令

## 如何使用

### 基本用法

```python
from dimos.web.websocket_vis.websocket_vis_module import WebsocketVisModule
from dimos.core.transport import LCMTransport, pLCMTransport

# Deploy the WebSocket visualization module
websocket_vis = dimos.deploy(WebsocketVisModule, port=7779)

# Receive control from the Foxglove plugin.
websocket_vis.click_goal.transport = LCMTransport("/goal_request", PoseStamped)
websocket_vis.explore_cmd.transport = LCMTransport("/explore_cmd", Bool)
websocket_vis.stop_explore_cmd.transport = LCMTransport("/stop_explore_cmd", Bool)
websocket_vis.movecmd.transport = LCMTransport("/cmd_vel", Twist)
websocket_vis.gps_goal.transport = pLCMTransport("/gps_goal")

# Send visualization data to the Foxglove plugin.
websocket_vis.robot_pose.connect(connection.odom)
websocket_vis.path.connect(global_planner.path)
websocket_vis.global_costmap.connect(mapper.global_costmap)
websocket_vis.gps_location.connect(connection.gps_location)

# Start the module
websocket_vis.start()
```

### 访问界面

关于如何在 Foxglove 中添加 command-center 插件，请参见 `dimos/web/command-center-extension/README.md`。
