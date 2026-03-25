# 数据加载

[`get_data`](/dimos/utils/data.py) 函数提供对测试数据和模型文件的访问，并自动处理 Git LFS 下载。

## 基本用法

```python
from dimos.utils.data import get_data

# 获取数据文件/目录的路径
data_path = get_data("cafe.jpg")
print(f"Path: {data_path}")
print(f"Exists: {data_path.exists()}")
```

<!--Result:-->
```
Path: /home/lesh/coding/dimos/data/cafe.jpg
Exists: True
```

## 工作原理

<details><summary>Pikchr</summary>

```pikchr fold output=assets/get_data_flow.svg
color = white
fill = none

A: box "get_data(name)" rad 5px fit wid 170% ht 170%
arrow right 0.4in
B: box "Check" "data/{name}" rad 5px fit wid 170% ht 170%

# Branch: exists
arrow from B.e right 0.3in then up 0.4in then right 0.3in
C: box "Return path" rad 5px fit wid 170% ht 170%

# Branch: missing
arrow from B.e right 0.3in then down 0.4in then right 0.3in
D: box "Pull LFS" rad 5px fit wid 170% ht 170%
arrow right 0.3in
E: box "Decompress" rad 5px fit wid 170% ht 170%
arrow right 0.3in
F: box "Return path" rad 5px fit wid 170% ht 170%
```

</details>

<!--Result:-->
![output](assets/get_data_flow.svg)

1. 检查 `data/{name}` 是否已在本地存在
2. 若不存在，从 Git LFS 拉取 `.tar.gz` 归档文件
3. 将归档文件解压至 `data/`
4. 返回解压后文件/目录的 `Path` 路径

## 常见使用模式

### 加载图片

```python
from dimos.utils.data import get_data
from dimos.msgs.sensor_msgs import Image

image = Image.from_file(get_data("cafe.jpg"))
print(f"Image shape: {image.data.shape}")
```

<!--Result:-->
```
Image shape: (771, 1024, 3)
```

### 加载模型检查点

```python
from dimos.utils.data import get_data

model_dir = get_data("models_yolo")
checkpoint = model_dir / "yolo11n.pt"
print(f"Checkpoint: {checkpoint.name} ({checkpoint.stat().st_size // 1024}KB)")
```

<!--Result:-->
```
Checkpoint: yolo11n.pt (5482KB)
```

### 加载录制数据用于回放

```python
from dimos.utils.data import get_data
from dimos.utils.testing.replay import TimedSensorReplay

data_dir = get_data("unitree_office_walk")
replay = TimedSensorReplay(data_dir / "lidar")
print(f"Replay {replay} loaded from: {data_dir.name}")
print(replay.find_closest_seek(1))
```

<!--Result:-->
```
Replay <dimos.utils.testing.replay.TimedSensorReplay object at 0x7fdc24c708f0> loaded from: unitree_office_walk
{'type': 'msg', 'topic': 'rt/utlidar/voxel_map_compressed', 'data': {'stamp': 1751591000.0, 'frame_id': 'odom', 'resolution': 0.05, 'src_size': 77824, 'origin': [-3.625, -3.275, -0.575], 'width': [128, 128, 38], 'data': {'points': array([[ 2.725, -1.025, -0.575],
       [ 2.525, -0.275, -0.575],
       [ 2.575, -0.275, -0.575],
       ...,
       [ 2.675, -0.525,  0.775],
       [ 2.375,  1.175,  0.775],
       [ 2.325,  1.225,  0.775]], shape=(22730, 3))}}}
```

### 加载点云数据

```python
from dimos.utils.data import get_data
from dimos.mapping.pointclouds.util import read_pointcloud

pointcloud = read_pointcloud(get_data("apartment") / "sum.ply")
print(f"Loaded pointcloud with {len(pointcloud.points)} points")
```

<!--Result:-->
```
Loaded pointcloud with 63672 points
```

## 数据目录结构

数据文件存放在仓库根目录的 `data/` 下。大文件以 `.tar.gz` 归档形式存储在 `data/.lfs/` 中，由 Git LFS 追踪管理。

<details><summary>结构图</summary>

```diagon fold mode=Tree
data/
  cafe.jpg
  apartment/
    sum.ply
  .lfs/
    cafe.jpg.tar.gz
    apartment.tar.gz
```

</details>

<!--Result:-->
```
data/
 ├──cafe.jpg
 ├──apartment/
 │   └──sum.ply
 └──.lfs/
     ├──cafe.jpg.tar.gz
     └──apartment.tar.gz
```


## 添加新数据

### 小文件（< 1MB）

直接提交到 `data/`：

```sh skip
cp my_image.jpg data/

# 2. 压缩并上传至 LFS
./bin/lfs_push

git add data/.lfs/my_image.jpg.tar.gz

git commit -m "Add test image"
```

### 大文件或目录

使用 LFS 工作流：

```sh skip
# 1. 将数据复制到 data/
cp -r my_dataset/ data/

# 2. 压缩并上传至 LFS
./bin/lfs_push

git add data/.lfs/my_dataset.tar.gz

# 3. 提交 .tar.gz 引用
git commit -m "Add my_dataset test data"
```

[`lfs_push`](/bin/lfs_push) 脚本的工作流程：
1. 将 `data/my_dataset/` 压缩为 `data/.lfs/my_dataset.tar.gz`
2. 上传至 Git LFS
3. 暂存压缩文件

pre-commit 钩子（[`bin/hooks/lfs_check`](/bin/hooks/lfs_check#L26)）会阻止提交：如果 `data/` 中存在未压缩的目录且在 `data/.lfs/` 中没有对应的 `.tar.gz` 文件。

## 路径解析

运行位置不同时的路径解析方式：
- **Git 仓库中**：使用 `{repo}/data/`
- **已安装的包**：克隆仓库到用户数据目录：
  - Linux：`~/.local/share/dimos/repo/data/`
  - macOS：`~/Library/Application Support/dimos/repo/data/`
  - 回退路径：`/tmp/dimos/repo/data/`
