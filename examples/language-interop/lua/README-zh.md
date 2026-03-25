# Lua 机器人控制示例

订阅机器人里程计数据并使用 LCM（Lightweight Communications and Marshalling，轻量级通信与编组）发布 twist 指令。

## 前置条件

- Lua 5.4
- LuaSocket（`sudo luarocks install luasocket`）
- 系统依赖：`glib`、`cmake`

## 设置

```bash
./setup.sh
```

此脚本将执行以下操作：
1. 克隆并构建官方 [LCM](https://github.com/lcm-proj/lcm) Lua 绑定
2. 克隆 [dimos-lcm](https://github.com/dimensionalOS/dimos-lcm) 消息定义

## 运行

```bash
lua main.lua
```

## 输出示例

```
Robot control started
Subscribing to /odom, publishing to /cmd_vel
Press Ctrl+C to stop.

[pose] x=15.29 y=9.62 z=0.00 | qw=0.57
[twist] linear=0.50 angular=0.00
[pose] x=15.28 y=9.63 z=0.00 | qw=0.57
...
```
