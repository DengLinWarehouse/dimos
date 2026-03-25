
# 代码块

**所有代码块必须是可执行的。**
不要编写示意性/伪代码块。
如果你要展示 API 使用模式，请创建一个能实际运行的最小工作示例。这确保了文档随代码库演进始终保持正确。

编写完 Markdown 文件中的代码块后，可以通过执行以下命令来运行它：
`md-babel-py run document.md`

关于此工具的更多信息请参阅 [codeblocks](/docs/agents/docs/codeblocks.md)


# 代码或文档链接

添加文档链接后运行：

`doclinks document.md`

### 代码文件引用
```markdown
See [`service/spec.py`](/dimos/protocol/service/spec.py) for the implementation.
```

运行 doclinks 后，变为：
```markdown
See [`service/spec.py`](/dimos/protocol/service/spec.py) for the implementation.
```

### 符号自动链接
在同一行提及符号可自动链接到其所在行号：
```markdown
The `Configurable` class is defined in [`service/spec.py`](/dimos/protocol/service/spec.py#L22).
```

变为：
```markdown
The `Configurable` class is defined in [`service/spec.py`](/dimos/protocol/service/spec.py#L22).
```
### 文档间引用
使用 `.md` 作为链接目标：
```markdown
See [Configuration](/docs/usage/configuration.md) for more details.
```

变为：
```markdown
See [Configuration](/docs/usage/configuration.md) for more details.
```

关于此功能的更多信息请参阅 [doclinks](/docs/agents/docs/doclinks.md)


# Pikchr

[Pikchr](https://pikchr.org/) 是来自 SQLite 的图表语言。适用于流程图和架构图。

**重要：** 始终将 Pikchr 代码块包裹在 `<details>` 标签中，使源码在 GitHub 上默认折叠。渲染后的 SVG 在折叠区域外保持可见。代码块（Python 等）不应折叠——它们是用来阅读的。

## 基本语法

<details>
<summary>图表源码</summary>

```pikchr fold output=assets/pikchr_basic.svg
color = white
fill = none

A: box "Step 1" rad 5px fit wid 170% ht 170%
arrow right 0.3in
B: box "Step 2" rad 5px fit wid 170% ht 170%
arrow right 0.3in
C: box "Step 3" rad 5px fit wid 170% ht 170%
```

</details>

<!--Result:-->
![output](assets/pikchr_basic.svg)

## 框体尺寸

使用 `fit` 配合百分比缩放来自动调整带内边距的框体大小：

<details>
<summary>图表源码</summary>

```pikchr fold output=assets/pikchr_sizing.svg
color = white
fill = none

# fit wid 170% ht 170% = 自动调整大小 + 内边距
A: box "short" rad 5px fit wid 170% ht 170%
arrow right 0.3in
B: box ".subscribe()" rad 5px fit wid 170% ht 170%
arrow right 0.3in
C: box "two lines" "of text" rad 5px fit wid 170% ht 170%
```

</details>

<!--Result:-->
![output](assets/pikchr_sizing.svg)

`fit wid 170% ht 170%` 模式的含义是：根据文本自动调整大小，然后将宽度缩放 170%，高度缩放 170%。

对于显式尺寸设置（需要一致的框体大小时）：

<details>
<summary>图表源码</summary>

```pikchr fold output=assets/pikchr_explicit.svg
color = white
fill = none

A: box "Step 1" rad 5px fit wid 170% ht 170%
arrow right 0.3in
B: box "Step 2" rad 5px fit wid 170% ht 170%
```

</details>

<!--Result:-->
![output](assets/pikchr_explicit.svg)

## 常用设置

始终以以下设置开头：

```
color = white    # 文本颜色
fill = none      # 透明框体填充
```

## 分支路径

<details>
<summary>图表源码</summary>

```pikchr fold output=assets/pikchr_branch.svg
color = white
fill = none

A: box "Input" rad 5px fit wid 170% ht 170%
arrow
B: box "Process" rad 5px fit wid 170% ht 170%

# 向上分支
arrow from B.e right 0.3in then up 0.35in then right 0.3in
C: box "Path A" rad 5px fit wid 170% ht 170%

# 向下分支
arrow from B.e right 0.3in then down 0.35in then right 0.3in
D: box "Path B" rad 5px fit wid 170% ht 170%
```

</details>

<!--Result:-->
![output](assets/pikchr_branch.svg)

**提示：** 对于树形/层级图表，建议使用从左到右的布局（根节点在左，子节点向右展开）。这种布局更易于阅读，避免了垂直堆叠带来的不便。

## 添加标签

<details>
<summary>图表源码</summary>

```pikchr fold output=assets/pikchr_labels.svg
color = white
fill = none

A: box "Box" rad 5px fit wid 170% ht 170%
text "label below" at (A.x, A.y - 0.4in)
```

</details>

<!--Result:-->
![output](assets/pikchr_labels.svg)

## 参考

| 元素 | 语法 |
|---------|--------|
| 矩形框 | `box "text" rad 5px wid Xin ht Yin` |
| 箭头 | `arrow right 0.3in` |
| 椭圆 | `oval "text" wid Xin ht Yin` |
| 文本 | `text "label" at (X, Y)` |
| 命名点 | `A: box ...` 然后引用 `A.e`、`A.n`、`A.x`、`A.y` |

完整文档请参阅 [pikchr.org/home/doc/trunk/doc/userman.md](https://pikchr.org/home/doc/trunk/doc/userman.md)。
