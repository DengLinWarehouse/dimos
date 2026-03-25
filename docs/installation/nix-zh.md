# Nix 安装（使用 Nix 管理 dimos 时必需）

你需要先安装 [nix](https://nixos.org/) 并启用 [flakes](https://nixos.wiki/wiki/Flakes)。

建议参考[官方安装文档](https://nixos.org/download/)，以下是快速安装步骤：

```sh
# 安装 Nix https://nixos.org/download/
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install
. /nix/var/nix/profiles/default/etc/profile.d/nix-daemon.sh

# 确保启用了 nix-flakes
mkdir -p "$HOME/.config/nix"; echo "experimental-features = nix-command flakes" >> "$HOME/.config/nix/nix.conf"
```

# 将 DimOS 作为库使用

```sh
mkdir myproject && cd myproject

# 拉取 flake（在仓库外部运行 nix develop 时需要）
wget https://raw.githubusercontent.com/dimensionalOS/dimos/refs/heads/main/flake.nix
wget https://raw.githubusercontent.com/dimensionalOS/dimos/refs/heads/main/flake.lock

# 进入 nix 开发 shell（提供系统依赖）
nix develop

python3 -m venv .venv
source .venv/bin/activate

# 安装所有内容（根据你的使用场景，可能不需要所有 extras，
# 请参考对应的平台指南）
pip install "dimos[misc,sim,visualization,agents,web,perception,unitree,manipulation,cpu,dev]"
```

# 开发 DimOS

```sh
# 允许按需获取大文件（而不是立即拉取所有文件）
export GIT_LFS_SKIP_SMUDGE=1
git clone -b dev https://github.com/dimensionalOS/dimos.git
cd dimos

# 进入 nix 开发 shell（提供系统依赖）
nix develop

python3 -m venv .venv
source .venv/bin/activate

pip install -e ".[misc,sim,visualization,agents,web,perception,unitree,manipulation,cpu,dev]"

# 类型检查
mypy dimos

# 测试（大约需要一分钟运行）
pytest dimos
```
