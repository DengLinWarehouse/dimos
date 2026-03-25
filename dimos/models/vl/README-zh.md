# Vision Language Models（视觉语言模型）

这里提供了用于处理图像和文本查询的视觉语言模型实现。

## QwenVL Model

`QwenVlModel` 类提供了对阿里巴巴 Qwen2.5-VL 模型的访问能力，可用于视觉-语言任务。

### 示例用法

```python
from dimos.models.vl.qwen import QwenVlModel
from dimos.msgs.sensor_msgs.Image import Image

# Initialize the model (requires ALIBABA_API_KEY environment variable)
model = QwenVlModel()

image = Image.from_file("path/to/your/image.jpg")

response = model.query(image.data, "What do you see in this image?")
print(response)
```

## Moondream Hosted Model

`MoondreamHostedVlModel` 类提供了对托管版 Moondream API（应用程序编程接口）的访问能力，可用于快速视觉语言任务。

**前置条件：**

在使用该模型之前，必须先导出你的 API key：
```bash
export MOONDREAM_API_KEY="your_api_key_here"
```

### 能力

该模型支持四种工作模式：

1. **Caption**：生成图像描述。
2. **Query**：对图像提出自然语言问题。
3. **Detect**：查找特定对象的边界框。
4. **Point**：定位特定对象的中心点。

### 示例用法

```python
from dimos.models.vl.moondream_hosted import MoondreamHostedVlModel
from dimos.msgs.sensor_msgs import Image

model = MoondreamHostedVlModel()
image = Image.from_file("path/to/image.jpg")

# 1. Caption
print(f"Caption: {model.caption(image)}")

# 2. Query
print(f"Answer: {model.query(image, 'Is there a person in the image?')}")

# 3. Detect (returns ImageDetections2D)
detections = model.query_detections(image, "person")
for det in detections.detections:
    print(f"Found person at {det.bbox}")

# 4. Point (returns list of (x, y) coordinates)
points = model.point(image, "person")
print(f"Person centers: {points}")
```
