# Cap Index（Qoder 插件）

在会话开始注入一份紧凑的能力目录；需要细节时用 `find` 检索，任务开始前用 `decide` 获取建议或候选，产物生成后用 `verify` 检查格式。默认不再对每个 prompt 打分或强制某个能力。

## 安装

需要 Node.js 18+。

```bash
qodercli plugins install <本目录>
```

本地 zip 文件不受支持：请先解压，再安装解压出的插件目录。

安装后重启会话。首次会话会在后台建立索引；`SessionStart` 同步读取已生成的目录，索引缺失时只输出一行 bootstrap，不阻塞会话。

## 组成

- `hooks/hooks.json`
  - `SessionStart`（同步）：`hook session-start` 读取 `~/.cap-index/qoder/capability-catalog.md`，通过 `hookSpecificOutput.additionalContext` 注入能力目录
  - `SessionStart`（async）：`build --if-stale --quiet --sync-runtime` 保持索引和运行时新鲜
  - 默认不再注册 `UserPromptSubmit` 逐 prompt 路由
- `skills/cap-index/SKILL.md`：目录、检索与重建说明
- `commands/cap-index.md`：`/cap-index [list|find <need>|decide <task>|verify <path>|catalog|build|check]`
- `cap-index.bundle.mjs` + `defaults/overrides.json`：单文件运行时与出厂覆盖

## 能力目录与检索

- 目录按文档、调研、工程、架构、网站、个人事务、桌面、扩展、安全分组；每条只放名称和一句用途。
- 状态异常的 MCP 会标注 `[不可用]` / `[缓存]`；advisory 条目标注 `[建议]`。
- `find` 是检索工具，不是意图判定：返回 top5 候选、用途、适用/不适用与调用目标；是否使用由 agent 结合任务判断。
- `decide` 读取可移植 `defaults/policy.json` + 本机 `capability-roles.json`：强信号给出 native-first / tool / blocked、质量门槛与回退；未识别时返回 `unclear` + 候选，不冒充 native 结论。
- `verify` 检查 `.pptx`、PDF、图片、SVG、HTML 或站点目录的格式；失败时按 decide 的回退切到能力。
- 中文按 bigram/trigram，英文按词级；字段加权 BM25，`notFor` 命中时降权并提示，不硬排除。
- 默认保持沉默；旧的逐 prompt 分数路由仅在 `CAP_INDEX_LEGACY_ROUTE=1` 时启用。

## 数据

- `~/.cap-index/qoder/capability-catalog.md`：会话目录
- `~/.cap-index/qoder/capabilities.json`：能力条目（schema 3）
- `~/.cap-index/qoder/capability-roles.json`：本机能力到类别的映射（本地生成，不上传）
- `~/.cap-index/qoder/mcp-cache.json`：MCP 工具缓存
- `~/.cap-index/runtime/`：插件与 npm CLI 共用的运行时
- 第三方描述清洗后入库；不保存 MCP 的 env / headers / 凭据

## 诊断

```bash
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" catalog --client qoder
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" find "合并扫描PDF" --client qoder
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" list --client qoder
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" eval-find --fixture test/fixtures/capabilities.json --cases <cases.json>
```

## 回滚

- `CAP_INDEX_DISABLE=1`：所有 hook no-op
- `CAP_INDEX_CATALOG=0`：关闭目录注入
- `CAP_INDEX_LEGACY_ROUTE=1`：恢复旧的逐 prompt 分数路由
- 旧版插件目录保留，可切回 0.4.0

## v0.6.1 变更

- `decide` 改为分句判定（否定作用域 + 动作动词），未识别时返回 `unclear` + 候选，不下 native 结论。
- 强信号仍给 tool / native-first / blocked；质量门槛和回退不变。
- 自动/ hook 同步不再降级 runtime；显式 `build --sync-runtime` 仍可回滚。

## v0.6.0 变更

- 新增 `decide`：任务开始前给出 native-first / tool / blocked、质量门槛与回退。
- 新增本机 `capability-roles.json`：能力类别映射；可在 `overrides.json` 的 `roles` 覆盖。
- 新增 `verify`：检查 PPTX/PDF/图片/SVG/HTML/站点格式。
- 策略默认“质量优先、时间其次”，token 暂不作为硬限制。
- 保留 `find`/`catalog`/`eval-find`；旧路由仅作诊断。

## 卸载

```bash
qodercli plugins uninstall cap-index
```

数据目录会保留，可按需删除 `~/.cap-index/qoder`。
