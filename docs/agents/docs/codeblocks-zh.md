# 可执行代码块

我们使用 [md-babel-py](https://github.com/leshy/md-babel-py/) 来执行 Markdown 中的代码块并插入结果。

## 黄金法则

**所有代码块必须是可执行的。** 不要编写示意性/伪代码块。如果你要展示 API 使用模式，请创建一个能实际运行的最小工作示例。这确保了文档随代码库演进始终保持正确。

## 运行

```sh skip
md-babel-py run document.md          # 就地编辑
md-babel-py run document.md --stdout # 预览输出到 stdout
md-babel-py run document.md --dry-run # 显示将要运行的内容
```

## 支持的语言

Python、Shell (sh)、Node.js，以及可视化工具：Matplotlib、Graphviz、Pikchr、Asymptote、OpenSCAD、Diagon。

## 代码块标志

在语言标识符后添加标志：

| 标志 | 效果 |
|------|--------|
| `session=NAME` | 在具有相同会话名称的代码块之间共享状态 |
| `output=path.png` | 将输出写入文件而非内联显示 |
| `no-result` | 执行但不插入结果 |
| `skip` | 不执行此代码块 |
| `expected-error` | 预期此代码块会失败 |

## 示例

# md-babel-py

在 Markdown 文件中执行代码块并插入结果。

![Demo](assets/screencast.gif)

**使用场景：**
- 自动保持文档示例与代码同步
- 验证文档中的代码片段确实可以运行
- 从 Markdown 中的代码生成图表和图形
- 可执行文档的文学编程

## 语言

### Shell

```sh
echo "cwd: $(pwd)"
```

<!--Result:-->
```
cwd: /work
```

### Python

```python session=example
a = "hello world"
print(a)
```

<!--Result:-->
```
hello world
```

会话（Session）在代码块之间保持状态：

```python session=example
print(a, "again")
```

<!--Result:-->
```
hello world again
```

### Node.js

```node
console.log("Hello from Node.js");
console.log(`Node version: ${process.version}`);
```

<!--Result:-->
```
Hello from Node.js
Node version: v22.21.1
```

### Matplotlib

```python output=assets/matplotlib-demo.svg
import matplotlib.pyplot as plt
import numpy as np
plt.style.use('dark_background')
x = np.linspace(0, 4 * np.pi, 200)
plt.figure(figsize=(8, 4))
plt.plot(x, np.sin(x), label='sin(x)', linewidth=2)
plt.plot(x, np.cos(x), label='cos(x)', linewidth=2)
plt.xlabel('x')
plt.ylabel('y')
plt.legend()
plt.grid(alpha=0.3)
plt.savefig('{output}', transparent=True)
```

<!--Result:-->
![output](assets/matplotlib-demo.svg)

### Pikchr

SQLite 的图表语言：

<details>
<summary>图表源码</summary>

```pikchr fold output=assets/pikchr-demo.svg
color = white
fill = none
linewid = 0.4in

# 输入文件
In: file "README.md" fit
arrow

# 处理
Parse: box "Parse" rad 5px fit
arrow
Exec: box "Execute" rad 5px fit

# 分发到各语言
arrow from Exec.e right 0.3in then up 0.4in then right 0.3in
Sh: oval "Shell" fit
arrow from Exec.e right 0.3in then right 0.3in
Node: oval "Node" fit
arrow from Exec.e right 0.3in then down 0.4in then right 0.3in
Py: oval "Python" fit

# 合并回来
X: dot at (Py.e.x + 0.3in, Node.e.y) invisible
line from Sh.e right until even with X then down to X
line from Node.e to X
line from Py.e right until even with X then up to X
Out: file "README.md" fit with .w at (X.x + 0.3in, X.y)
arrow from X to Out.w
```

</details>

<!--Result:-->
![output](assets/pikchr-demo.svg)

### Asymptote

矢量图形：

```asymptote output=assets/histogram.svg
import graph;
import stats;

size(400,200,IgnoreAspect);
defaultpen(white);

int n=10000;
real[] a=new real[n];
for(int i=0; i < n; ++i) a[i]=Gaussrand();

draw(graph(Gaussian,min(a),max(a)),orange);

int N=bins(a);

histogram(a,min(a),max(a),N,normalize=true,low=0,rgb(0.4,0.6,0.8),rgb(0.2,0.4,0.6),bars=true);

xaxis("$x$",BottomTop,LeftTicks,p=white);
yaxis("$dP/dx$",LeftRight,RightTicks(trailingzero),p=white);
```

<!--Result:-->
![output](assets/histogram.svg)

### Graphviz

```dot output=assets/graph.svg
A -> B -> C
A -> C
```

<!--Result:-->
![output](assets/graph.svg)

### OpenSCAD

```openscad output=assets/cube-sphere.png
cube([10, 10, 10]);
sphere(r=7);
```

<!--Result:-->
![output](assets/cube-sphere.png)

### Diagon

ASCII 艺术图表：

```diagon mode=Math
1 + 1/2 + sum(i,0,10)
```

<!--Result:-->
```
        10
        ___
    1   ╲
1 + ─ + ╱   i
    2   ‾‾‾
         0
```

```diagon mode=GraphDAG
A -> B -> C
A -> C
```

<!--Result:-->
```
┌───┐
│A  │
└┬─┬┘
 │┌▽┐
 ││B│
 │└┬┘
┌▽─▽┐
│C  │
└───┘
```

## 安装

### Nix（推荐）

```sh skip
# 直接从 GitHub 运行
nix run github:leshy/md-babel-py -- run README.md --stdout

# 或克隆后本地运行
nix run . -- run README.md --stdout
```

### Docker

```sh skip
# 从 Docker Hub 拉取
docker run -v $(pwd):/work lesh/md-babel-py:main run /work/README.md --stdout

# 或通过 Nix 本地构建
nix build .#docker     # 构建 tarball 到 ./result
docker load < result   # 从 tarball 加载镜像
docker run -v $(pwd):/work md-babel-py:latest run /work/file.md --stdout
```

### pipx

```sh skip
pipx install md-babel-py
# 或：uv pip install md-babel-py
md-babel-py run README.md --stdout
```

如果不使用 Nix 或 Docker，各语言执行器需要相应的系统依赖：

| 语言 | 系统包 |
|-----------|-----------------------------|
| python | python3 |
| node | nodejs |
| dot | graphviz |
| asymptote | asymptote, texlive, dvisvgm |
| pikchr | pikchr |
| openscad | openscad, xvfb, imagemagick |
| diagon | diagon |

```sh skip
# Arch Linux
sudo pacman -S python nodejs graphviz asymptote texlive-basic openscad xorg-server-xvfb imagemagick

# Debian/Ubuntu
sudo apt-get install python3 nodejs graphviz asymptote texlive xvfb imagemagick openscad
```

注意：pikchr 和 diagon 可能需要从源码编译。使用 Docker 或 Nix 可获得完整的执行器支持。

## 用法

```sh skip
# 就地编辑文件
md-babel-py run document.md

# 输出到单独文件
md-babel-py run document.md --output result.md

# 输出到 stdout
md-babel-py run document.md --stdout

# 仅运行指定语言
md-babel-py run document.md --lang python,sh

# 试运行 - 显示将要执行的内容
md-babel-py run document.md --dry-run
```
