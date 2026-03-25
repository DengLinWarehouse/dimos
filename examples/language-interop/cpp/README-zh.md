# C++ 机器人控制示例

订阅 `/odom` 话题并向 `/cmd_vel` 发布速度指令。

## 构建

```bash
mkdir build && cd build
cmake ..
make
./robot_control
```

## 依赖项

- [lcm](https://lcm-proj.github.io/) - 通过包管理器安装
- 消息头文件从 [dimos-lcm](https://github.com/dimensionalOS/dimos-lcm) 自动获取
