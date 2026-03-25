# 系统依赖安装（Ubuntu 22.04 或 24.04）

```sh
sudo apt-get update
sudo apt-get install -y curl g++ portaudio19-dev git-lfs libturbojpeg python3-dev pre-commit

# 安装 uv
curl -LsSf https://astral.sh/uv/install.sh | sh && export PATH="$HOME/.local/bin:$PATH"
```

# 将 DimOS 作为库使用

```sh
mkdir myproject && cd myproject

uv venv --python 3.12
source .venv/bin/activate

# 安装所有内容（根据你的使用场景，可能不需要所有 extras，
# 请参考对应的平台指南）
uv pip install 'dimos[misc,sim,visualization,agents,web,perception,unitree,manipulation,cpu,dev]'
```

# 开发 DimOS

```sh
# 允许按需获取大文件（而不是立即拉取所有文件）
export GIT_LFS_SKIP_SMUDGE=1
git clone -b dev https://github.com/dimensionalOS/dimos.git
cd dimos

uv sync --all-extras --no-extra dds

# 类型检查
uv run mypy dimos

# 测试（大约需要一分钟运行）
uv run pytest dimos
```
