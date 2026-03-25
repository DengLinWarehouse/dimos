编写或编辑 Markdown 文档时，使用 `doclinks` 工具来解析文件引用。

完整文档（如需要）：[`utils/docs/doclinks.md`](/dimos/utils/docs/doclinks.md)

## 语法

<!-- doclinks-ignore-start -->
| 模式 | 示例 |
|-------------|-----------------------------------------------------|
| 代码文件 | `[`service/spec.py`]()` → 解析路径 |
| 带符号 | `Configurable` 在 `[`spec.py`]()` → 添加 `#L<行号>` |
| 文档链接 | `[Configuration](.md)` → 解析到文档 |
<!-- doclinks-ignore-end -->

## 用法

```bash
doclinks docs/guide.md   # 单个文件
doclinks docs/           # 目录
doclinks --dry-run ...   # 仅预览
```
