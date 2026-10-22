# Cap Index（Qoder 插件）

按 prompt 动态注入能力路由：把当前请求与本机已安装的 Skills、MCP 工具做匹配，只注入真正相关的 2–5 条，避免每次会话都加载完整工具清单。

## 安装

需要 Node.js 18+。

```bash
qodercli plugins install <本目录>
# 或先打包成 zip 再安装
qodercli plugins install ./cap-index-0.3.0.zip
# 发布后可从 HTTPS 直接安装
qodercli plugins install https://github.com/<owner>/<repo>/releases/download/v0.3.0/cap-index-0.3.0.zip
```

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
- `cap-index.bundle.mjs` + `defaults/overrides.json`：单文件运行时与空默认词表；每条路由由安装者本机的 skills / MCP 自动生成
- `docs/AUTHORING.md`：插件作者接入与维护者发布指南（中英双语）

## 路由分级

每次 prompt 只注入与当前请求相关的能力，并按命中强度分级：

- 命中门槛：得分 ≥ `max(1.8, 最高分 × 0.45)`，最多注入 `--top-k` 条（默认 3）。
- **高置信**（directive 条目中得分 ≥ 最高 directive 分的 0.8）：标注 `[必须]`，并注入「默认委派原则」——专家能力与原生能力重叠时默认委派给专家能力，仅在不可用、调用失败或成本明显更高时才自行处理。
- **中置信**（其余命中）：标注 `[建议]`，仅供参考，不相关可忽略。
- 若没有任何 directive 命中，则只输出建议式提示，不附加强制规则。

## 路由模式

每个条目有 `mode`，决定它如何参与路由：

- `directive`（默认）：可被标 `[必须]`，参与默认委派规则。适合 MCP、可执行技能和任务型专家能力。
- `advisory`：只以 `[建议]` 出现，永不单独触发委派规则。适合行为准则、编码规范、评审建议等“随时适用但不该强制”的能力。
- `off`：不参与路由，也不写入静态路由表。

构建时会自动分类，用户无需逐条配置：

- 描述匹配 `guidelines / best practices / conventions / principles / checklist / 行为准则 / 编码规范 / 最佳实践` 等 → `advisory`
- 描述声明“仅斜杠命令 / 显式调用”（`only via slash`、`仅通过命令` 等）→ `off`
- 插件根目录提供 `cap-index.json` 可精确声明，优先级高于自动分类：
  ```json
  { "routing": { "skill-name": "advisory", "*": "directive" } }
  ```
- `~/.cap-index/qoder/overrides.json` 中的 `mode` 优先级最高

## 数据

- `~/.cap-index/qoder/`：索引、MCP 缓存、可选用户覆盖
- `~/.cap-index/runtime/`：插件与 npm CLI 共用的运行时
- 不保存 MCP 的 env / headers / 凭据

## 定制

在 `~/.cap-index/qoder/overrides.json` 中覆盖触发词与优先级，然后运行：

```bash
node "$HOME/.cap-index/runtime/cap-index.bundle.mjs" build --client qoder --refresh
```

## 0.3.0 变更

- 新增 `directive` / `advisory` / `off` 路由模式、自动分类与 `cap-index.json` 作者声明
- 补全 MIT 许可证与市场元数据
- 改进 MCP 降级日志：明确缓存回退、不可达影响和重试命令
- 新增 `docs/AUTHORING.md` 中英双语作者与维护者文档

## 卸载

```bash
qodercli plugins uninstall cap-index
```

数据目录会保留，可按需删除 `~/.cap-index/qoder`。
