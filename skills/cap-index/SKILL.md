---
name: cap-index
description: Capability catalog, on-demand search, native-vs-tool advice, and artifact verification for installed skills and MCP servers. Use before starting a task when unsure whether native tools or an installed capability is the better path, or when a newly installed capability is not visible.
---

# Cap Index

cap-index 扫描本机已安装的 Skills 与 MCP servers，在会话开始时注入一份紧凑的能力目录；需要细节时用 `find` 检索，任务开始前用 `decide` 获取建议或候选，产物生成后用 `verify` 检查格式。

默认不再对每个 prompt 推送分数路由或“必须”项。目标是：**先保证质量，再选择耗时/token 更低的路径**；强信号才下结论，未识别时给候选。

## 何时使用

- 任务开始前，不确定应该用 native 还是已安装能力
- 不确定某个任务应该用哪个已安装 skill / MCP
- 刚安装了新的 skill、插件或 MCP server，需要刷新索引
- 需要检查 `.pptx`、PDF、图片、SVG、HTML 或站点是否真的符合交付格式
- 用户问「有哪些 skill / MCP 可用」「为什么没调用某个工具」

## 命令

运行时由插件在会话启动时同步到 `~/.cap-index/runtime/`：

```bash
# 任务开始前：native / 工具建议、质量门槛与回退
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" decide "做一张封面图" --client qoder

# 按“想要的结果”检索候选能力（只读，不注入）
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" find "把季度复盘做成可编辑的PPT" --client qoder

# 产物校验
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" verify "output/deck.pptx" --class presentation --client qoder

# 查看会话级能力目录
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" catalog --client qoder

# 查看所有条目与 MCP 状态
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" list --client qoder

# 重新枚举 MCP 并重建索引（同时生成本地 roles）
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" build --client qoder --refresh

# 检查索引是否过期
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" check --client qoder
```

Windows PowerShell 中把 `$HOME` 换成 `$env:USERPROFILE`。

## 决策方式

- `decide` 使用**可移植的类别策略**（`defaults/policy.json`）+ **本机能力映射**（`capability-roles.json`）。
- 类别策略不绑定具体工具 ID；本机映射把用户实际安装的能力归入类别。
- 默认顺序：质量门槛 → 时间/成本；token 暂不作为硬限制。
- 强信号（明确能力名、明确产物或操作）才给出 `tool` / `native-first` / `blocked` 结论；未识别到明确信号时返回 `unclear` + 最多 3 个候选能力（`check`），由 agent 结合任务判断，不要当成 native 结论。
- `native-first` 表示先用 native，产物 `verify` 失败再切工具；`tool` 表示优先工具；`blocked` 表示缺少必要能力或权限。

## 数据

- 索引：`~/.cap-index/qoder/capabilities.json`
- 本机能力类别：`~/.cap-index/qoder/capability-roles.json`
- 会话目录：`~/.cap-index/qoder/capability-catalog.md`
- 策略：插件内 `defaults/policy.json`（可移植）；用户可在 `overrides.json` 的 `policy`/`roles` 中覆盖
- 路由表（legacy 诊断）：`~/.cap-index/qoder/capability-index.md`
- MCP 缓存：`~/.cap-index/qoder/mcp-cache.json`

`find` 是检索工具，`decide` 是任务前的建议，`verify` 是产物检查；都不是强制指令。不要输出 MCP server 的 env、headers 或任何凭据；索引只保存工具名、用途与状态。
