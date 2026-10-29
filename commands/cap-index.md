---
description: Show, search, decide, or verify cap-index capabilities and artifacts
argument-hint: [list|find <need>|decide <task>|verify <path>|catalog|build|check]
---

# /cap-index

运行 cap-index 并简要汇报结果。

1. 从 `$ARGUMENTS` 读取动作，默认 `list`。
2. 如果动作以 `find ` 开头，执行：
   `node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" find "<需求>" --client qoder`
   这是只读检索，返回候选能力；是否使用由当前任务决定。
3. 如果动作以 `decide ` 开头，执行：
   `node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" decide "<任务>" --client qoder`
   它会在任务开始前给出 native/工具建议或候选，以及质量门槛和回退；强信号才下结论，未识别时返回候选，不冒充 native 判断。
4. 如果动作以 `verify ` 开头，执行：
   `node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" verify "<产物路径>" --client qoder`
   用于检查 `.pptx`、PDF、图片、SVG、HTML 或站点目录是否真的符合格式。
5. 否则执行 `node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" <动作> --client qoder`。
   `build` 时追加 `--refresh`；`catalog` 用于查看会话级能力目录。
6. 用简洁的列表或结论回复；只展示能力名、用途、建议与 MCP 状态，不要展示凭据或配置文件内容。

如果运行时不存在，先运行 `/cap-index` 所在插件的会话启动钩子或直接执行 `node "${QODER_PLUGIN_ROOT}/cap-index.bundle.mjs" build --client qoder`。
