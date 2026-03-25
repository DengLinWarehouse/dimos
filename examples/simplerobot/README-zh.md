# SimpleRobot

一个用于测试和开发的最小化虚拟机器人。它实现了与真实机器人相同的部分 LCM 接口，非常适合测试第三方集成（参见 `examples/language-interop/`）或尝试 dimos Module（模块）模式。

## 接口

| 话题       | 类型          | 方向   | 描述                                    |
|------------|---------------|--------|-----------------------------------------|
| `/cmd_vel` | `Twist`       | 订阅   | 速度指令（linear.x、angular.z）         |
| `/odom`    | `PoseStamped` | 发布   | 以 30Hz 频率发布当前位姿                |

真实物理机器人通常以 `TransformStamped` 形式在 TF 树中发布多个位姿关系，而 SimpleRobot 为简化起见直接发布 `PoseStamped`。

详细信息请参阅 [坐标变换](/docs/usage/transforms.md)

## 使用方法

```bash
# 带 pygame 可视化界面
python examples/simplerobot/simplerobot.py

# 无头模式
python examples/simplerobot/simplerobot.py --headless

# 运行自测试演示
python examples/simplerobot/simplerobot.py --headless --selftest
```

在另一个终端中使用 `lcmspy` 查看消息。按 `q` 或 `Esc` 退出可视化界面。

## 发送指令

从任何具有 LCM 绑定的语言中，向 `/cmd_vel` 发布 `Twist` 消息：

```python
from dimos.core.transport import LCMTransport
from dimos.msgs.geometry_msgs import Twist

transport = LCMTransport("/cmd_vel", Twist)
transport.publish(Twist(linear=(0.5, 0, 0), angular=(0, 0, 0.3)))
```

C++、TypeScript 和 Lua 示例请参见 `examples/language-interop/`。

## 物理模型

SimpleRobot 使用二维独轮车模型（unicycle model）：
- `linear.x` 控制前进/后退
- `angular.z` 控制左转/右转
- 指令在 0.5 秒后超时（如果没有新指令，机器人将停止）
