# doclinks

一个 Markdown 链接解析器，可自动为文档中的代码引用补全正确的文件路径。

## 它做什么

编写文档时，你可以使用如下占位链接：

<!-- doclinks-ignore-start -->
```markdown
See [`service/spec.py`]() for the implementation.
```
<!-- doclinks-ignore-end -->

运行 `doclinks` 后，这些链接会被解析为实际路径：

<!-- doclinks-ignore-start -->
```markdown
See [`service/spec.py`](/dimos/protocol/service/spec.py) for the implementation.
```
<!-- doclinks-ignore-end -->

## 功能特性

<!-- doclinks-ignore-start -->
- **代码文件链接**：`[`filename.py`]()` 会解析为该文件的路径
- **符号行号链接**：如果同一行中还出现了另一个反引号包裹的术语，它会在文件中定位该符号并添加 `#L<line>`：
  ```markdown
  See `Configurable` in [`config.py`]()
  → [`config.py`](/path/config.py#L42)
  ```
- **文档到文档的链接**：`[Modules](.md)` 会解析为 `modules.md` 或 `modules/index.md`
<!-- doclinks-ignore-end -->
- **多种链接模式**：支持 absolute、relative 或 GitHub URL
- **监视模式**：文件变更后自动重新处理
- **忽略区域**：跳过包含 `<!-- doclinks-ignore-start/end -->` 注释的区块

## 用法

```bash
# Process a single file
doclinks docs/guide.md

# Process a directory recursively
doclinks docs/

# Relative links (from doc location)
doclinks --link-mode relative docs/

# GitHub links
doclinks --link-mode github   --github-url https://github.com/org/repo docs/

# Dry run (preview changes)
doclinks --dry-run docs/

# CI check (exit 1 if changes needed)
doclinks --check docs/

# Watch mode (auto-update on changes)
doclinks --watch docs/
```

## 选项

| 选项 | 说明 |
|--------------------|-------------------------------------------------|
| `--root PATH` | 仓库根目录（默认：自动检测 git 根目录） |
| `--link-mode MODE` | `absolute`（默认）、`relative` 或 `github` |
| `--github-url URL` | GitHub 基础 URL（`github` 模式必需） |
| `--github-ref REF` | GitHub 链接所用的分支/ref（默认：`main`） |
| `--dry-run` | 显示变更但不修改文件 |
| `--check` | 如果需要变更则报错退出（用于 CI） |
| `--watch` | 监听变更并重新处理 |

## 链接模式

<!-- doclinks-ignore-start -->
| 模式 | 说明 |
|----------------------|------------------------------------------------|
| `[`file.py`]()` | 代码文件引用（空链接或任意链接都可） |
| `[`path/file.py`]()` | 带部分路径的代码文件引用，用于消除歧义 |
| `[`file.py`](#L42)` | 保留现有行号片段 |
| `[Doc Name](.md)` | 文档到文档链接（按名称解析） |
<!-- doclinks-ignore-end -->

## 解析方式

该工具会为仓库中的所有文件建立索引。对于 `/dimos/protocol/service/spec.py`，它会创建以下查找条目：

- `spec.py`
- `service/spec.py`
- `protocol/service/spec.py`
- `dimos/protocol/service/spec.py`

当多个文件同名时，请使用更长的路径。
