---
description: Show or refresh the cap-index capability routing index
argument-hint: [list|build|check]
---

# /cap-index

运行 cap-index 并简要汇报结果。

1. 从 `$ARGUMENTS` 读取动作，默认 `list`。
2. 执行：`node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" <动作> --client qoder`。如果动作是 `build`，追加 `--refresh`。
3. 用简洁的列表或结论回复；只展示能力名、用途与 MCP 状态，不要展示凭据或配置文件内容。

如果运行时不存在，先运行 `/cap-index` 所在插件的会话启动钩子或直接执行 `node "${QODER_PLUGIN_ROOT}/cap-index.bundle.mjs" build --client qoder`。
