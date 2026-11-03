---
name: kingdee-kb-dev
description: KingdeeKB（顾问工作台）项目开发指南。在本仓库新增/修改 Tauri 命令、Rust service/store、React 页面/组件、IPC 封装、知识编译链路、发版时使用。覆盖架构分层、编码约定、新增功能的端到端步骤、验证命令与常见坑。
---

# KingdeeKB 开发 Skill

面向金蝶 ERP 实施顾问的本地优先（local-first）AI 知识库桌面应用。
Tauri v2（Rust 后端）+ React 19 / TypeScript / Tailwind v4 前端，数据落本地 SQLite。

## 0. 开工前

1. 判断任务级别（见根目录 `CLAUDE.md`）：新能力 / 跨模块重构 / 破坏性变更 / 安全策略 → 先走 `OpenSpec/changes/<change-id>/` 提案，用户审批后再写代码；bug 修复、小改可直接做。
2. 写代码前对照 `OpenSpec/coding-standards.md`（Rust 规则 ID 详见 `OpenSpec/rust-guidelines/safe-guides/`）。
3. 领域概念与关键链路见 `OpenSpec/project.md`。

**文档与代码不一致时以代码为准**，已知漂移：
- README 写 `pnpm`，实际 `tauri.conf.json` 与 CI 均用 **npm**（仓库只有 `package-lock.json`）。
- `CLAUDE.md` 提到 `kit.Logger`（zap）是其他项目遗留；本项目后端日志用 `tracing`。
- `OpenSpec/roadmap.md` 内容为 HexLM 项目遗留，不代表本项目计划。
- `OpenSpec/coding-standards.md` 写「JS 字符串用单引号」，但 Biome 配置是 **双引号 + 按需分号**，以 Biome 为准。
- 根目录 `skills/` 是**应用内置的业务技能**（随安装包分发），不是开发用 skill，别往里放开发文档。

## 1. 架构速览

```
前端  pages/ → contexts/ + hooks/ → lib/*-commands.ts ──invoke──┐
后端  commands/<domain>.rs → services/<domain>/*.rs → Store(rusqlite) ←┘
      全局状态：app_state.rs::AppState（Arc<Mutex|RwLock<_>>）
      命令注册：lib.rs 的 tauri::generate_handler![...]
```

| 位置 | 职责 |
|---|---|
| `src-tauri/src/commands/<domain>.rs` | 参数解析、取 `State<'_, AppState>`、加锁、错误归一；**不写业务逻辑** |
| `src-tauri/src/services/<domain>/` | 领域逻辑：agent / knowledge / project / risk / skill / media / security / verification / common / harness |
| `src-tauri/src/services/mod.rs` | `pub use` 再导出子模块，保持 `crate::services::xxx` 扁平路径 |
| `src-tauri/src/app_state.rs` | `AppState` 字段 + 初始化（每个 store 自己 `Connection` + `ensure_table()`） |
| `src-tauri/src/error.rs` | `AppError` / `AppResult<T>`，序列化为 `{code, message, provider_id?}` |
| `src-tauri/resources/prompts/` | 系统提示词与 recipe 模板（`.md`） |
| `src-tauri/capabilities/` | Tauri 权限声明（最小权限） |
| `src/lib/tauri-commands.ts` 等 `*-commands.ts` | 所有 IPC 封装 + 与 Rust 结构体对应的 TS 类型 |
| `src/lib/app-error.ts` | `parseAppError` / `formatAppError`；code 必须与 `AppError::code()` 一致 |
| `src/contexts/` | 跨页面状态（Project / Agent / KbCompilation / Outline / AsrConfig / Audio / AppError） |
| `e2e/mocks/tauri-mock.ts` | Playwright 的 `invoke` mock 表 |

核心业务链路：原始资料 → `file_extractor::extract_text` → Step1 文档分析（`analysis_cache`）→ Step2 LLM 知识编译写 `content_candidate` → 人工 `approve_wiki_page` → 正式 `content` → `build_knowledge_graph`（wikilink 1.0 / tag 0.6 / source 0.4 / co_citation 0.3）。检索：fastembed 向量（usearch）+ BM25（tantivy + jieba）+ RRF + rerank。

## 2. 硬性约定

### Rust
- **项目隔离**：所有业务查询 / 写入必须带 `project_id` 过滤。
- **错误**：command 边界返回 `Result<T, String>`（存量主流）或 `AppResult<T>`（新代码推荐，前端可按 `code` 分支）。内部用 `anyhow::Result` / `AppResult` + `?`。生产代码禁止 `unwrap()` / `expect()` / `panic!`（测试除外）。锁错误统一 `.map_err(|e| e.to_string())?`。
- **LLM 401/Key 失效**：用 `AppError::classify_llm_error(provider_id, &err)`，前端据 `LLM_INVALID_KEY` 弹配置框而非 toast。
- **吞错必须留痕**：`Err(e) => { warn!("描述: {e}"); return None }`，禁止静默 `Err(_) =>`。
- **日志**：只用 `tracing::{info,warn,error,debug}!`，禁止 `println!` / `dbg!`。面向用户的错误文案用中文。
- **锁**：不得持 `std::sync::Mutex` guard 跨 `.await`；进入异步前 clone 出所需 store / 数据，或用块作用域提前释放 guard。`risk_control_store`、`skill_manager` 是 `tokio::sync::Mutex`，用 `.lock().await`。
- **后台长任务**：用 `services::common::spawn_safe::spawn_monitored(name, Some(&app), fut)`，不要裸 `tauri::async_runtime::spawn`；内部仍需 `match` Result 并更新状态。前端轮询状态命令（如 `get_kb_recompile_status`）或监听事件。
- **数据库**：store 在 `ensure_table()` 里 `CREATE TABLE IF NOT EXISTS` 幂等建表；**项目未发布，单一 schema，不写迁移函数**（改表结构直接改 DDL）。多步写操作用事务 / SAVEPOINT。O(n²) 组合要有上限截断（参考 `MAX_CO_OCCURRENCE_GROUP_SIZE`）。
- **业务不变量**：候选内容不可直接覆盖正式 `content`，必须走 approve；删 wiki 页不级联删 documents / chunks / vectors。
- **凭据**：密钥 / token 一律 `keyring`（service = `kingdee-kb`），禁止明文写 JSON；多项凭据写入要考虑失败回滚（参考 `AsrConfigStore::save_tencent`）。
- **dead code**：禁止 `#![allow(dead_code)]` 全局抑制，只在具体项上加并注释原因。
- 文档注释用中文 `///`，公开函数写 `# Errors` 段落（存量风格）。

### 前端
- 页面不裸 `invoke`，统一走 `lib/*-commands.ts`；命令名 `snake_case`，封装函数 camelCase，参数对象 key 用 camelCase（Tauri 自动转 snake_case）。
- TS 类型字段与 Rust serde 输出一致（多为 snake_case 字段名，如 `chunk_id`），新增结构体时两边同步。
- 错误展示：`formatAppError(e)`；Key 失效走 `AppErrorContext`。
- 函数组件 + Hooks；`@/` 别名导入；严格 TS，避免 `any`；列表 `key` 不用索引。
- 新页面用 `React.lazy()` 注册到 `App.tsx`；**`Home` 保持静态 import**。
- 两处以上复用的工具函数放 `src/lib/utils.ts`；Context 默认值用工厂函数（`createDefaultSlot()`），不共享可变单例。
- UI：Tailwind，主色 `#1A6BD8` 系列，图标 `lucide-react` 具名导入，Toast 用 `useToast()`，加载态 `Loader2`，空态要有占位。
- 格式：Biome（2 空格、行宽 100、双引号、按需分号），提交前 `npm run lint:fix`。

## 3. 常见任务步骤

详细模板见 [references/recipes.md](references/recipes.md)。速查：

**新增一个 Tauri 命令（最常见，也最容易漏）**
1. `services/<domain>/xxx.rs` 写逻辑（新文件需在 `services/<domain>/mod.rs` 声明，并在 `services/mod.rs` 加 `pub use`）。
2. `commands/<domain>.rs` 加 `#[tauri::command]` 薄封装。
3. **`lib.rs` 的 `generate_handler![]` 注册**（历史上漏注册导致线上回归，见提交 318fb24）。
4. `src/lib/*-commands.ts` 加 TS 类型 + 封装函数。
5. `e2e/mocks/tauri-mock.ts` 的 `MOCK_RESPONSES` 补默认返回值。
6. 页面 / context 调用，错误用 `formatAppError`。

**新增一张表 / store**：store 结构 `new(conn)` + `ensure_table()` → `AppState` 加字段（`Arc<Mutex<_>>`）并在 `AppState::new` 中初始化调用 `ensure_table()?` → 所有 SQL 带 `project_id`。

**改提示词**：`src-tauri/resources/prompts/*.md`；知识编译提示词需保证至少产出 1 个 `[[slug]]`，否则图谱为 0 边。

## 4. 验证

```bash
npm run typecheck                                   # TS 类型
npm run lint                                        # Biome
npm test                                            # Vitest（src/**/*.test.tsx，与源码同目录）
npm run test:e2e                                    # Playwright（mock invoke，不需要后端）
cargo clippy --manifest-path src-tauri/Cargo.toml --all-targets -- -D warnings
cargo test --manifest-path src-tauri/Cargo.toml     # 单测多为模块内 #[cfg(test)]；集成测试在 src-tauri/tests/
npm run tauri:dev                                   # 真实运行（端口 1420）
```

- 只改前端：typecheck + lint + 相关 vitest；交互改动需在 `tauri:dev` 或 e2e 中走关键路径。
- 改 Rust：至少 `cargo check`/clippy + 相关模块测试；Rust 全量编译较慢，可只跑 `cargo test --manifest-path src-tauri/Cargo.toml <模块名>`。
- 图谱 / canvas / 可视化改动需真实界面验证并看控制台错误。
- 没跑过的验证不得写成已验证。

## 5. 发版

- 版本号两处同步：`package.json` 的 `version` 与 `src-tauri/Cargo.toml` 的 `version`；提交 `chore: release x.y.z`。
- 推送 `v*` tag 触发 `.github/workflows/build.yml`（Win / macOS ARM / macOS x64 / Linux），Windows 另出 portable zip，release notes 由 `scripts/generate-release-notes.py` 生成。
- 用户可见变更补 `CHANGELOG.md`（中文，按 ✨/🐛/🔒/♻️/📝 分类）。
- commit / tag / push 均需用户明确指令。

## 6. 提交信息

英文 Conventional Commits + 三段式正文（What changed / Impact / Validation），type ∈ feat/fix/refactor/chore/docs/test/style/perf。正文不得提及 AI 工具、内部规划目录或本地参考仓库。
