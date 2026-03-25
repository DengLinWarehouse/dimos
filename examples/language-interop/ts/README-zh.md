# TypeScript 机器人控制示例

订阅 `/odom` 话题并向 `/cmd_vel` 发布速度指令。

## 命令行示例

```bash
deno task start
```

## Web 示例

基于浏览器的控制界面，通过 WebSocket 桥接：

```bash
cd web
deno run --allow-net --allow-read --unstable-net server.ts
```

在浏览器中打开 http://localhost:8080。

功能特性：
- 实时位姿显示
- 方向键 / WASD 键控制
- 点击按钮发送 twist 指令

浏览器通过 [esm.sh](https://esm.sh) 导入 `@dimos/msgs`，直接编码/解码 LCM 数据包——服务器仅在 WebSocket 和 UDP 多播之间转发原始二进制数据。

## 依赖项

TypeScript 互操作的主要文档：

- [@dimos/lcm](https://jsr.io/@dimos/lcm)
- [@dimos/msgs](https://jsr.io/@dimos/msgs)
