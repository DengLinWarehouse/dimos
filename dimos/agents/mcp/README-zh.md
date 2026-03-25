# DimOS MCP 服务器

通过 Model Context Protocol（模型上下文协议）将 DimOS 机器人技能暴露给 Claude Code。

## 设置

```bash
uv sync --extra base --extra unitree
```

添加到 Claude Code（一条命令）

```bash
claude mcp add --transport http --scope project dimos http://localhost:9990/mcp
```

验证是否添加成功：

```bash
claude mcp list
```

## MCP Inspector

如果你想手动检查服务器，可以使用 MCP Inspector。

安装：

```bash
npx -y @modelcontextprotocol/inspector
```

它会打开一个浏览器窗口。

将 **Transport Type** 改为 "Streamable HTTP"，将 **URL** 改为 `http://localhost:9990/mcp`，将 **Connection Type** 改为 "Direct"，然后点击 "Connect"。

## 使用方法

**终端 1** - 启动 DimOS：
```bash
uv run dimos run unitree-go2-agentic-mcp
```

**Claude Code** - 使用机器人技能：
```
> move forward 1 meter
> go to the kitchen
> tag this location as "desk"
```

## 工作原理

1. Blueprint 中的 `McpServer` 在端口 9990 上启动一个 FastAPI 服务器
2. Claude Code 直接连接到 `http://localhost:9990/mcp`
3. 技能作为 MCP 工具暴露（例如 `relative_move`、`navigate_with_text`）
