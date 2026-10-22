---
name: cap-index
description: Inspect, rebuild, or troubleshoot the capability routing index. Use when the user asks which skills or MCP tools are available, when a newly installed skill/MCP server is not being suggested, or when capability routes look stale or missing.
---

# Cap Index

cap-index 扫描本机已安装的 Skills 与 MCP servers，生成意图路由表，并在每次用户提交 prompt 时只注入命中的 2–5 条路由。

## 何时使用

- 用户问「有哪些 skill / MCP 可用」「为什么没调用某个工具」
- 刚安装了新的 skill、插件或 MCP server，需要刷新索引
- 怀疑路由表过期、MCP 工具列表不完整
- 想查看或调整能力优先级（编辑 `~/.cap-index/qoder/overrides.json`）

## 命令

运行时由插件在会话启动时同步到 `~/.cap-index/runtime/`：

```bash
# 查看所有条目与 MCP 状态
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" list --client qoder

# 重新枚举 MCP 并重建索引
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" build --client qoder --refresh

# 检查索引是否过期
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" check --client qoder
```

Windows PowerShell 中把 `$HOME` 换成 `$env:USERPROFILE`。

## 数据

- 索引：`~/.cap-index/qoder/capabilities.json`
- 路由表：`~/.cap-index/qoder/capability-index.md`
- 用户覆盖：`~/.cap-index/qoder/overrides.json`（可选，覆盖默认触发词与优先级）
- MCP 缓存：`~/.cap-index/qoder/mcp-cache.json`

不要输出 MCP server 的 env、headers 或任何凭据；索引只保存工具名、用途与状态。
