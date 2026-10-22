# Cap Index Author & Maintainer Guide / 作者与维护者指南

Applies to Cap Index 0.3.0. 适用于 Cap Index 0.3.0。

This guide has two parts. 本文档分为两部分：

- **Part A — Plugin authors**: declare how your skills participate in routing. 插件作者：声明你的 skill 如何参与路由。
- **Part B — Maintainers**: release, package, distribute, and troubleshoot. 维护者：发布、打包、分发与排障。

---

## Part A — Plugin Authors / 插件作者接入

### A1. What you can declare / 可以声明什么

Cap Index reads an optional `cap-index.json` at your plugin root. It controls only skills from that plugin. MCP servers are always `directive`.

Cap Index 会读取插件根目录下可选的 `cap-index.json`，只影响该插件提供的 skill；MCP server 固定为 `directive`。

### A2. File location / 文件位置

```text
my-plugin/
├── .qoder-plugin/plugin.json     # or .claude-plugin/plugin.json
├── cap-index.json                # optional routing declaration
└── skills/
    └── my-skill/SKILL.md
```

The file must sit next to the `.qoder-plugin/` or `.claude-plugin/` directory. It is read only when that directory contains `plugin.json`.

该文件必须与 `.qoder-plugin/` 或 `.claude-plugin/` 目录同级；只有该目录中存在 `plugin.json` 时才会被读取。

### A3. Format / 格式

```json
{
  "routing": {
    "*": "directive",
    "my-guidelines": "advisory",
    "my-command-only-skill": "off"
  }
}
```

Keys are skill names from `SKILL.md` frontmatter (`name`), not namespaced full names. `*` is the fallback for skills without an explicit entry.

键名是 `SKILL.md` frontmatter 中的 `name`，不是带命名空间的完整名称；`*` 是未单独声明 skill 的兜底值。

A bare mapping without the `routing` wrapper is also accepted:

也兼容不写 `routing` 包装层的直接映射：

```json
{
  "*": "directive"
}
```

### A4. Modes / 路由模式

| Mode | Behavior | Typical use |
| --- | --- | --- |
| `directive` | Can be marked `[必须]`; participates in the default delegation rule. | MCP-backed capabilities, task skills, experts. |
| `advisory` | Never marked `[必须]` and never triggers delegation alone. Shown as `[建议]` when a directive route also matches. | Guidelines, conventions, review hints. |
| `off` | Excluded from routing and from the static routing table. | Slash-command-only or intentionally hidden skills. |

| 模式 | 行为 | 适用场景 |
| --- | --- | --- |
| `directive` | 可被标为 `[必须]`，参与默认委派规则。 | MCP 能力、任务型 skill、专家能力。 |
| `advisory` | 永不被标 `[必须]`，不会单独触发委派；有 directive 命中时显示 `[建议]`。 | 行为准则、编码规范、评审建议。 |
| `off` | 不参与路由，也不写入静态路由表。 | 仅斜杠命令、刻意隐藏的 skill。 |

### A5. Precedence / 优先级

Highest to lowest:

1. `~/.cap-index/qoder/overrides.json` → `entries.<id>.mode`
2. Your plugin's `cap-index.json` declaration
3. Automatic classification from the skill description
4. Default: `directive`

从高到低：

1. 用户覆盖 `~/.cap-index/qoder/overrides.json` 中的 `mode`
2. 插件根目录 `cap-index.json` 的声明
3. 根据 skill 描述自动分类
4. 默认 `directive`

A user override always wins, so do not rely on declarations to force a mode for every user. 用户覆盖始终优先，不能假设你的声明会覆盖用户选择。

### A6. Automatic classification / 自动分类

When you do not declare a mode, Cap Index classifies from the description:

- Matches guidelines, best practices, conventions, principles, checklist, 行为准则, 编码规范, 最佳实践, 风格指南, 编码指南 → `advisory`
- Declares that it is only invoked through a slash command or explicit user action (`only via slash`, `仅通过命令`, `需要显式调用` 等) → `off`
- Otherwise → `directive`

未声明模式时，会根据描述自动分类：

- 命中准则/规范/最佳实践等关键词 → `advisory`
- 声明仅通过斜杠命令或显式调用 → `off`
- 其他 → `directive`

`off` is checked before `advisory`. If both patterns appear, the skill is hidden. 若两类模式同时命中，`off` 优先。

### A7. Skill families / 技能家族

Skills under the same namespace are grouped into one routing entry. The family mode comes from its router skill: `explore`, a skill whose name equals the namespace, then `building/build/overview/router`, then the first skill.

同一命名空间下的 skill 会聚合为一条家族路由。家族模式取路由器 skill：`explore` → 与命名空间同名的 skill → `building/build/overview/router` → 第一个 skill。

To make a whole family advisory, declare the router skill's name, for example:

要让整个家族变为 advisory，声明路由器 skill 的名称，例如：

```json
{
  "routing": {
    "explore": "advisory"
  }
}
```

Declaring every member does not change the family mode; only the router entry is classified. 逐个声明成员不会改变家族模式，只有路由器条目参与分类。

### A8. Verify / 验证

```bash
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" build --client qoder --refresh
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" list --client qoder
```

In `list`, your declaration appears as `adv/author` (advisory) or `off`. Directive routes keep the normal `on`/MCP status. To simulate a prompt:

在 `list` 中，声明生效时会显示 `adv/author` 或 `off`；directive 显示正常的 `on`/MCP 状态。模拟一次 prompt：

```bash
printf '%s' '{"prompt":"帮我写代码，同时调研一下"}' \
  | node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" hook user-prompt-submit --client qoder
```

Windows PowerShell: replace `$HOME` with `$env:USERPROFILE` and pipe with `'{"prompt":"..."}' | node ...`.

Expected behavior / 预期行为：

- A high-confidence directive route is tagged `[必须]` and a delegation header is injected. 高置信 directive 会标 `[必须]` 并注入委派规则。
- Advisory routes are tagged `[建议]` only when a directive route also matches; advisory-only matches use the suggestion header without a delegation rule. 只有同时存在 directive 命中时 advisory 才标 `[建议]`；仅 advisory 命中时使用建议式提示，不含委派规则。
- `off` routes never appear and are not written to `capability-index.md`. `off` 不会出现在路由中，也不会写入静态路由表。

`~/.cap-index/qoder/capabilities.json` also records `mode` and `modeReason` for each entry. `modeReason: "author"` confirms your declaration was applied.

`~/.cap-index/qoder/capabilities.json` 会记录每条路由的 `mode` 和 `modeReason`；`author` 表示你的声明已生效。

### A9. FAQ / 常见问题

**Declaration has no effect / 声明不生效**

- Confirm `cap-index.json` is at the plugin root, next to `.qoder-plugin/`.
- Confirm the key matches `SKILL.md` `name` exactly.
- Run `build --refresh`; list reads the rebuilt index.
- Check whether a user override sets `mode` for the same entry.

- 确认 `cap-index.json` 在插件根目录，与 `.qoder-plugin/` 同级。
- 确认键名与 `SKILL.md` 的 `name` 完全一致。
- 运行 `build --refresh` 重建索引。
- 检查用户覆盖是否对同一条目设置了 `mode`。

**Family mode changed unexpectedly / 家族模式变化**

Declare the router skill (`explore`), not an individual member. 声明路由器 skill（如 `explore`），而不是单个成员。

**Why is my MCP server not affected? / 为什么 MCP 不受影响**

MCP entries are always `directive`; use user overrides if a specific user needs to hide one. MCP 条目固定为 `directive`；个别用户如需隐藏，可用用户覆盖。

**Can I put secrets in cap-index.json? / 能放凭据吗**

No. This file is meant to be committed and shared. 不要放任何凭据；该文件会随插件一起分发。

---

## Part B — Maintainers / 维护者发布指南

### B1. Versioning / 版本

Use semantic versioning. Bump both the plugin manifest and the runtime constant:

使用语义化版本，同时更新插件清单和运行时常量：

- `.qoder-plugin/plugin.json` → `version`
- `cap-index.bundle.mjs` → `VERSION`
- `~/.cap-index/runtime/cap-index.bundle.mjs` → synced by `build --sync-runtime`
- README / this guide / release notes

Before official publication, fill in the author metadata; it is intentionally left empty:

正式发布前补上作者信息（当前有意留空）：

```json
"author": { "name": "<your name>", "email": "<your email>" },
"homepage": "https://github.com/<owner>/<repo>",
"repository": "https://github.com/<owner>/<repo>"
```

Keep runtime-only fields in `plugin.json`. Listing fields such as `category`, `tags`, `logo`, `interface`, and `descriptionZh` belong to marketplace metadata; the current runtime ignores them in `plugin.json` and emits validation warnings.

`plugin.json` 只保留运行时字段。`category`、`tags`、`logo`、`interface`、`descriptionZh` 等属于市场展示元数据；当前运行时会忽略它们，并在校验时给出警告。

### B2. Build and sync / 构建与同步

```bash
node cap-index.bundle.mjs build --client qoder --refresh --sync-runtime
```

This rebuilds the index, forces live MCP discovery, and copies the runtime + defaults to `~/.cap-index/runtime/`.

该命令会重建索引、强制重新探测 MCP，并把运行时和默认配置同步到 `~/.cap-index/runtime/`。

### B3. Validate / 校验

```bash
qodercli plugins validate <plugin-dir>
qodercli --plugin-dir <plugin-dir> -p "帮我做个PPT"
node cap-index.bundle.mjs selftest --client qoder
node cap-index.bundle.mjs check --client qoder
```

Expected: validation reports no warnings, selftest passes, and `check` reports the index is current.

预期：校验无警告、自检全部通过、`check` 显示索引最新。

### B4. Package / 打包

The zip should contain a top-level plugin folder, for example:

zip 应包含一个顶层插件目录，例如：

```text
cap-index/
├── .qoder-plugin/plugin.json
├── cap-index.bundle.mjs
├── defaults/overrides.json
├── hooks/hooks.json
├── skills/cap-index/SKILL.md
├── commands/cap-index.md
├── docs/AUTHORING.md
├── assets/logo.svg
├── README.md
└── LICENSE
```

Do not include `node_modules`, local indexes, logs, or credentials. The shipped `defaults/overrides.json` must stay empty (`entries`, `groups`, `extraEntries`); every route is generated from the installer's own skills and MCP servers. Never ship entries derived from your local environment. 不要包含 `node_modules`、本地索引、日志或凭据。发布包中的 `defaults/overrides.json` 必须保持为空（`entries`、`groups`、`extraEntries`）；所有路由由安装者本机的 skills 与 MCP 自动生成，不要把作者本机的条目带进发布包。

Windows PowerShell:

```powershell
cd <parent-of-plugin>
Compress-Archive -Path .\cap-index -DestinationPath .\cap-index-0.3.0.zip
Get-FileHash .\cap-index-0.3.0.zip -Algorithm SHA256
```

Validate the extracted package, not only the source directory. 校验解压后的包，而不只是源码目录。

### B5. Distribution / 分发

**Option 1 — Direct HTTPS zip / 直接 HTTPS 安装**

```bash
qodercli plugins install https://github.com/<owner>/<repo>/releases/download/v0.3.0/cap-index-0.3.0.zip
```

**Option 2 — Self-hosted marketplace / 自建市场**

Create a `marketplace.json` that points to the release asset:

创建指向 release 产物的 `marketplace.json`：

```json
{
  "name": "cap-index-marketplace",
  "owner": { "name": "<your name>" },
  "plugins": [
    {
      "name": "cap-index",
      "description": "Per-prompt capability routing for skills and MCP servers.",
      "version": "0.3.0",
      "author": { "name": "<your name>" },
      "source": {
        "source": "archive",
        "url": "https://github.com/<owner>/<repo>/releases/download/v0.3.0/cap-index-0.3.0.zip",
        "sha256": "<sha256-of-zip>"
      }
    }
  ]
}
```

Users then run:

用户随后执行：

```bash
qodercli plugins marketplace add https://raw.githubusercontent.com/<owner>/<repo>/main/marketplace.json
qodercli plugins install cap-index
```

**Option 3 — Official Qoder marketplace / 官方市场**

Submit through Qoder's marketplace channel. It requires review and typically asks for listing copy, logo, screenshots, license, support contact, and a privacy/data statement. The CLI has no publish command.

通过 Qoder 官方市场渠道提交，通常需要审核、展示文案、图标、截图、许可证、支持联系方式以及隐私/数据说明。CLI 本身没有发布命令。

Public requirements in all cases / 通用要求：

- Public, stable HTTPS URL. 公开且稳定的 HTTPS 地址。
- Version in `plugin.json` matches the release tag and marketplace entry. 清单版本与 release/marketplace 条目一致。
- Node.js 18+ on the user machine. 用户机器需要 Node.js 18+。
- No credentials, absolute paths, or private endpoints. 不含凭据、绝对路径或私有端点。
- Publish the SHA256 with the release for supply-chain verification. 发布 SHA256 以便校验。

### B6. Release checklist / 发布检查清单

- [ ] Version bumped in manifest and bundle. 清单与运行时版本已更新。
- [ ] Author/homepage/repository filled before official publication. 正式发布前补全作者与仓库信息。
- [ ] `selftest` passes. 自检通过。
- [ ] `plugins validate` reports no warnings. 插件校验无警告。
- [ ] `build --refresh --sync-runtime` succeeds. 构建与运行时同步成功。
- [ ] `check` reports the index is current. 索引检查为最新。
- [ ] MCP degradation logs are understandable. MCP 降级日志可读。
- [ ] README and this guide updated. README 与本文档已更新。
- [ ] Zip contains the top-level plugin folder and no secrets. 压缩包结构正确且无凭据。
- [ ] Extracted zip validates. 解压后的包校验通过。
- [ ] SHA256 recorded and uploaded. 已记录并上传 SHA256。
- [ ] GitHub release asset published. Release 附件已发布。
- [ ] Marketplace metadata updated. 市场元数据已更新。

### B7. Logs and troubleshooting / 日志与排障

Default degradation logs are designed for end users:

默认降级日志面向普通用户：

- `stale-cache`: connection failed, using the cached tool list, with the number of tools and the `/cap-index build` retry hint. 连接失败，已回退缓存，并提示重试命令。
- `unreachable`: server skipped, does not affect other routes, with the retry hint. 已跳过该服务，不影响其他路由，并提示重试。
- `check`: lists concrete problems and the rebuild command. 列出具体问题并提示重建命令。

For diagnostics, set `CAP_INDEX_DEBUG=1`; hook failures, background-build failures, and the main error handler will print a redacted message and stack trace.

诊断时设置 `CAP_INDEX_DEBUG=1`，钩子失败、后台构建失败和主流程异常会输出脱敏后的错误与堆栈。

Data locations / 数据位置：

- `~/.cap-index/qoder/`: index, MCP cache, optional overrides
- `~/.cap-index/runtime/`: shared runtime
- MCP env/headers/credentials are not persisted. 不保存 MCP 凭据。

To rebuild from scratch, delete `~/.cap-index/qoder/` and run `build --refresh`. 需要完全重建时，删除该目录后重新构建。

### B8. Security / 安全

- Cap Index stores tool names, purposes, and status only. 只保存工具名、用途与状态。
- Error messages pass through redaction. 错误信息会脱敏。
- Do not paste MCP credentials into issues or docs. 不要把 MCP 凭据贴到 issue 或文档中。
- Review the bundle before publishing; it is a single self-contained file. 发布前审查打包后的单文件运行时。
