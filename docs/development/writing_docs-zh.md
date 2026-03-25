# 编写文档

1. 文档存放位置：
    - 如果文档仅面向 dimos 的贡献者（如本文档），请放在 `docs/development` 目录下
    - 其他文档放在 `docs/usage` 目录下
2. 运行 `bin/gen-diagrams` 来生成图表的 SVG 文件。我们使用 [mermaid](https://mermaid.js.org/intro/)（无需额外生成步骤）和 [pikchr](https://pikchr.org/home/doc/trunk/doc/userman.md) 作为图表语言。
3. 使用 [md-babel-py](https://github.com/leshy/md-babel-py/)（`md-babel-py run thing.md`）来确保你的代码示例能够正常运行。
