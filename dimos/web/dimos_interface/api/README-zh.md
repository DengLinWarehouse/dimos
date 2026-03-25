# Unitree API Server（Unitree API 服务器）

这是一个最小化的 FastAPI 服务器实现，为终端前端提供 API 端点。

## 快速开始

```bash
# Navigate to the api directory
cd api

# Install minimal requirements
pip install -r requirements.txt

# Run the server
python unitree_server.py
```

服务器将在 `http://0.0.0.0:5555` 上启动。

## 与前端集成

1. 按照上面的说明启动 API 服务器
2. 在另一个终端中，从项目根目录运行前端：
   ```bash
   cd ..  # Navigate to root directory (if you're in api/)
   yarn dev
   ```
3. 在终端界面中使用 `unitree` 命令：
   - `unitree status` - 检查 API 状态
   - `unitree command <text>` - 向 API 发送命令

## 与 DIMOS Agents 集成

更多信息请参见 DimOS 文档。

```python
from dimos.agents_deprecated.agent import OpenAIAgent
from dimos.robot.unitree.unitree_go2 import UnitreeGo2
from dimos.robot.unitree.unitree_skills import MyUnitreeSkills
from dimos.web.robot_web_interface import RobotWebInterface

robot_ip = os.getenv("ROBOT_IP")

# Initialize robot
logger.info("Initializing Unitree Robot")
robot = UnitreeGo2(ip=robot_ip,
                    connection_method=connection_method,
                    output_dir=output_dir)

# Set up video stream
logger.info("Starting video stream")
video_stream = robot.get_ros_video_stream()

# Create FastAPI server with video stream
logger.info("Initializing FastAPI server")
streams = {"unitree_video": video_stream}
web_interface = RobotWebInterface(port=5555, **streams)

# Initialize agent with robot skills
skills_instance = MyUnitreeSkills(robot=robot)

agent = OpenAIAgent(
    dev_name="UnitreeQueryPerceptionAgent",
    input_query_stream=web_interface.query_stream,
    output_dir=output_dir,
    skills=skills_instance,
)

web_interface.run()
```

## API 端点

- **GET /unitree/status**：检查 Unitree API 的状态
- **POST /unitree/command**：向 Unitree API 发送命令

## 工作原理

前端和后端是彼此独立的两个应用：

1. Svelte 前端通过 Vite 运行在 3000 端口
2. FastAPI 后端运行在 5555 端口
3. Vite 的开发服务器会将 `/unitree/*` 请求代理到 FastAPI 服务器
4. 终端界面中的 `unitree` 命令会向这些端点发送请求

这种架构允许前端和后端独立开发和运行。
