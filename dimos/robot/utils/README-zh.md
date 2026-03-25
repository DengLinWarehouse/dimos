# Robot Utils（机器人工具）

## RobotDebugger

`RobotDebugger` 提供了一种通过 Python shell 调试正在运行机器人的方式。

要求：

```bash
pip install rpyc
```

### 用法

1. **添加到你的机器人应用中：**
   ```python
   from dimos.robot.utils.robot_debugger import RobotDebugger

   # In your robot application's context manager or main loop:
   with RobotDebugger(robot):
       # Your robot code here
       pass

   # Or better, with an exit stack.
   exit_stack.enter_context(RobotDebugger(robot))
   ```

2. **以启用调试的方式启动机器人：**
   ```bash
   ROBOT_DEBUGGER=true python your_robot_script.py
   ```

3. **打开 Python shell：**
   ```bash
   ./bin/robot-debugger
    >>> robot.explore()
    True
   ```
