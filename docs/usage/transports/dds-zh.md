# 在 Ubuntu 上安装 DDS 传输库

`dds` 扩展通过 [Eclipse Cyclone DDS](https://cyclonedds.io/docs/cyclonedds-python/latest/) 提供 DDS（Data Distribution Service，数据分发服务）传输支持。在构建 Python 包之前需要先安装系统库。

```bash
# 安装 CycloneDDS 开发库
sudo apt install cyclonedds-dev

# 创建兼容性目录结构
#（必需，因为 Ubuntu 的 multiarch 布局与 CMake 预期的布局不匹配）
sudo mkdir -p /opt/cyclonedds/{lib,bin,include}
sudo ln -sf /usr/lib/x86_64-linux-gnu/libddsc.so* /opt/cyclonedds/lib/
sudo ln -sf /usr/lib/x86_64-linux-gnu/libcycloneddsidl.so* /opt/cyclonedds/lib/
sudo ln -sf /usr/bin/idlc /opt/cyclonedds/bin/
sudo ln -sf /usr/bin/ddsperf /opt/cyclonedds/bin/
sudo ln -sf /usr/include/dds /opt/cyclonedds/include/

# 使用 dds 扩展安装
CYCLONEDDS_HOME=/opt/cyclonedds uv pip install -e '.[dds]'
```

要安装所有扩展（包括 DDS）：

```bash
CYCLONEDDS_HOME=/opt/cyclonedds uv sync --extra dds
```
