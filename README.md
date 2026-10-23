# Cap Index（Qoder 插件）

按 prompt 动态注入能力路由：把当前请求与本机已安装的 Skills、MCP 工具做匹配，只注入真正相关的 2–5 条，避免每次会话都加载完整工具清单。

## 安装

需要 Node.js 18+。

```bash
qodercli plugins install <本目录>
# 或从 Release zip 安装
qodercli plugins install https://github.com/Screamity/cap-index/releases/download/v0.4.0/cap-index-0.4.0.zip
```

本地 zip 文件不受支持：请先解压，再安装解压出的插件目录。

本地验证：

```bash
qodercli plugins validate <本目录>
qodercli --plugin-dir <本目录> -p "帮我做个PPT"
```

安装后重启会话。首次会话会在后台建立索引；之后的新 prompt 会开始注入路由。

## 组成

- `hooks/hooks.json`
  - `SessionStart`（async）：索引过期时后台重建，并把运行时同步到 `~/.cap-index/runtime/`
  - `UserPromptSubmit`：读取索引，匹配当前 prompt，通过 `hookSpecificOutput.additionalContext` 注入 top-k 路由
- `skills/cap-index/SKILL.md`：排查与重建索引的说明
- `commands/cap-index.md`：`/cap-index [list|build|check]`
- `cap-index.bundle.mjs` + `defaults/overrides.json`：单文件运行时与出厂覆盖（默认为空，路由从本机已安装能力自动生成）

## 匹配与诊断

- 触发词从明确的意图标记提取（`Use when…`、`适用于…`、`当…时`），通用词和长句碎片会被过滤。
- 英文按词级匹配并允许少量间隔词；内置中英别名组提供软触发词，用于同义词召回，但不参与 `[必须]` 计分。
- 关键词分强 / 弱两层；只有弱关键词命中时不注入，除非条目设置 `trust: "high"` 或策略为 `always`。
- `notFor`（`Do not use for…` / `不适用于…`）是硬排除；阈值只统计可注入候选。
- 第三方描述会先清洗：移除控制/零宽字符、HTML/script、代码块与 URL；路由行跳过祈使句和注入话术，没有安全摘要时只保留名称。
- 单条路由行最多 150 字，单次注入最多 600 字 / 约 240 tokens；超限优先丢弃最长的建议，不丢必须。
- 用 `explain` 查看一条 prompt 的完整判定链：

```bash
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" explain --client qoder "帮我做个 PPT"
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" explain --client qoder "帮我做个 PPT" --json
```

## 数据

- `~/.cap-index/qoder/`：索引、MCP 缓存、可选用户覆盖
- `~/.cap-index/runtime/`：插件与 npm CLI 共用的运行时
- 第三方描述清洗后入库；不保存 MCP 的 env / headers / 凭据

## 定制

在 `~/.cap-index/qoder/overrides.json` 中覆盖触发词与优先级，然后运行：

```bash
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" build --client qoder --refresh
```

`[必须]` 默认只在最高分条目同时满足分数、领先幅度与证据门槛时出现，最多一条。可在用户覆盖中调整：

```json
{
  "version": 1,
  "routing": { "must": "trusted-only" }
}
```

`must` 可取 `trusted-only`（默认）、`always`、`never`；环境变量 `CAP_INDEX_MUST` 优先。

## v0.4.0 变更

- 新增内置中英同义词 / 别名层，作为软触发词参与召回，不参与 `[必须]` 计分。
- 英文触发词支持词级弹性匹配，允许少量间隔词；MCP 工具增加“动作 + 对象”关键词。
- 新增具体度抑制与 2.4 阈值上限，降低同义命中带来的重复路由。
- 增加代码评审文档类负向守卫，并补充固定集 / 盲测 / 85 项 selftest。
- 已知边界：`slides` 与 `Presentations` 等同一输出形态的替代能力仍可能同时注入；`[必须]` 的盲测标注口径仍在评估中。

## 卸载

```bash
qodercli plugins uninstall cap-index
```

数据目录会保留，可按需删除 `~/.cap-index/qoder`。
