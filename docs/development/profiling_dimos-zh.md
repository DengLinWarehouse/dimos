# 性能分析 dimos

你可以使用 py-spy 对特定的 blueprint（蓝图）进行性能分析：

```bash
uv run py-spy record --format speedscope --subprocesses -o profile.speedscope.json -- python -m dimos.robot.cli.dimos run unitree-go2-agentic
```

分析完成后按 `Ctrl+C` 停止。程序会生成一个 `profile.speedscope.json` 文件，你可以将其上传到 [speedscope.app](https://www.speedscope.app/) 进行可视化查看。
