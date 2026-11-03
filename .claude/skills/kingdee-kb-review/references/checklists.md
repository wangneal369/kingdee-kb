# 分 Pass 检查清单

每项后括号为常见级别；实际级别按触发后果判断。带 ⚑ 的条目来自本仓库历史回归，优先检查。

## Pass 1：架构与分层

- [ ] `commands/*.rs` 只做参数解析、取 State、加锁、错误归一，业务逻辑在 `services/`（P2）
- [ ] 新 service 文件已在 `services/<domain>/mod.rs` 声明，且在 `services/mod.rs` 有 `pub use` 再导出（编译失败则 P0）
- [ ] 新 store 已加入 `AppState` 并在 `AppState::new` 中初始化并调用 `ensure_table()?`（P1）
- [ ] 页面/组件没有裸 `invoke`，统一走 `src/lib/*-commands.ts`（P2；现存 `export_report`/`save_attachment_as` 为存量）
- [ ] 跨页面状态放 `contexts/`，复用逻辑放 `hooks/` 或 `src/lib/utils.ts`，没有在组件间复制粘贴（P2）
- [ ] 改动属于新能力/跨模块重构/破坏性变更/安全策略时有 OpenSpec 提案（P2）

## Pass 2：前后端契约

- [ ] ⚑ 新增/改名的 `#[tauri::command]` 已加入 `lib.rs` 的 `generate_handler![]`（漏注册编译不报错，运行时 command not found —— P0；见 318fb24）
- [ ] 删除/改名命令时，前端封装、调用方、`e2e/mocks/tauri-mock.ts` 同步更新（P1）
- [ ] TS 接口字段与 Rust `Serialize` 输出一致（字段名大小写、`Option` ↔ `?`/`| null`、`i64` ↔ `number`、枚举序列化格式、`#[serde(rename…)]`）（P1）
- [ ] `invoke` 参数 key 为 camelCase，与 Rust 参数名 snake_case 对应（P1）
- [ ] 返回类型从 `Result<T, String>` 改为 `AppResult<T>`（或反之）时，前端错误处理仍正确（`parseAppError` 能解析）（P1）
- [ ] 新增 `AppError` 变体时 `code()`、`user_message()` 已补，前端 `app-error.ts` 注释约定的 code 一致（P1）
- [ ] 事件名（`emit`/`listen`）两端一致，payload 结构一致（P1）
- [ ] e2e mock 为新命令提供默认返回值（P2）

## Pass 3：业务正确性与数据

- [ ] 所有业务 SQL / 查询带 `project_id` 过滤；按 id 查询时也校验归属项目（跨项目泄露 P0）
- [ ] 候选内容只写 `content_candidate`，正式 `content` 只经 `approve_wiki_page` 提升；`version`/`candidate_version` 不变量不被破坏（P0）
- [ ] 删除 wiki 页不级联删除 documents / chunks / vectors / BM25（P0）
- [ ] 多步写操作在事务 / SAVEPOINT 内，失败时回滚，不留半成品（P1）
- [ ] 长事务不阻塞高频写表（参考 `build_knowledge_graph` 的两阶段拆分）（P2）
- [ ] `ensure_table()` DDL 幂等（`IF NOT EXISTS`），改表结构直接改 DDL，不新增迁移函数（P2）
- [ ] 缓存键使用 `project_id + source_identity + sha256` 三元组；内容变化后缓存失效（P1）
- [ ] 知识编译提示词仍要求至少 1 个 `[[slug]]`（否则图谱 0 边）（P1）
- [ ] `kb_compilation_enabled` 关闭时，新逻辑不依赖 wiki / 图谱数据（P1）
- [ ] 边界：空项目、`currentProjectId == null`、空列表、超长文本、中文/特殊字符路径、重复导入（P1/P2）
- [ ] ⚑ LLM 上下文/历史截断使用配置的 `context_window`，不写死（见 54cbedc）（P1）
- [ ] 删除语义：涉及外部资源删除时走 `deletion_outbox` 补偿机制（P1）

## Pass 4：并发、异步与错误处理

- [ ] `std::sync::Mutex`/`RwLock` guard 没有跨 `.await` 存活（P1；死锁或 `Send` 编译错误）
- [ ] 多把锁的获取顺序一致，不在持有一把锁时去拿另一把可能反向持有的锁（P1）
- [ ] `tokio::sync::Mutex` 字段（`risk_control_store`、`skill_manager`）用 `.lock().await`，不在同步上下文 `blocking_lock`（P1）
- [ ] 后台任务用 `spawn_monitored`，且内部 `match` Result 并把状态复位为失败，不会让前端永远「进行中」（P1）
- [ ] Agent 会话支持取消：长循环检查 `cancel_flags` 中的 `AtomicBool`（P2）
- [ ] 生产代码无 `unwrap()` / `expect()` / `panic!` / 下标越界风险（测试除外）（P1）
- [ ] 吞错处必须 `warn!`/`error!` 记录：禁止 `Err(_) => return None`、`let _ = fallible()` 无日志（P1）
- [ ] 锁中毒处理：`.map_err(|e| e.to_string())?`，不 `unwrap`（P2）
- [ ] 前端异步：组件卸载后不再 setState、竞态请求取最新结果（P2）
- [ ] ⚑ 流式事件缓冲（rAF / event-buffer）不丢最后一批数据，状态引用不陈旧（见 26c01fb）（P1）

## Pass 5：安全与隐私

- [ ] 密钥 / token / secret 只存 `keyring`（service `kingdee-kb`），不落明文 JSON、不进 SQLite 明文列（P0）
- [ ] 多凭据写入有失败回滚并 `warn!`（参考 `AsrConfigStore::save_tencent`）（P1）
- [ ] 日志、错误信息、前端返回值不包含完整密钥、Authorization 头或用户原文敏感数据；外发 LLM 前经脱敏器（`desensitizer`）处理（P0/P1）
- [ ] 错误详情截断（`truncate_for_user`）后再给前端（P2）
- [ ] 文件操作：路径来自用户/前端时防目录穿越，不越出数据目录；写临时文件后清理（P1）
- [ ] 外部进程（ffmpeg、PowerShell Visio COM）参数不拼接未转义的用户输入（P0）
- [ ] `src-tauri/capabilities/` 新增权限遵循最小权限（P1）
- [ ] 前端：无未消毒的 `dangerouslySetInnerHTML`；`react-markdown` 不开启原始 HTML；`target="_blank"` 带 `rel="noopener noreferrer"`（P1）
- [ ] 未擅自修改配置文件、清空凭据或改为 `${ENV}` 占位（P1，违反用户规则）

## Pass 6：前端质量

- [ ] Hooks 只在顶层调用；`useEffect`/`useCallback`/`useMemo` 依赖完整（P1/P2）
- [ ] State 不可变更新；不在渲染期间 setState（P1）
- [ ] 列表 `key` 稳定唯一，不用数组索引（P2）
- [ ] ⚑ Context 默认值用工厂函数，不共享可变单例（见 90c3a04）（P1）
- [ ] ⚑ 跨页面切换需保留的状态（如对话历史）放 Context，而非页面局部 state（见 5856a46）（P1）
- [ ] 新页面 `React.lazy()` 注册；`Home` 保持静态 import（P2）
- [ ] 加载态（`Loader2`）、空态、错误态齐全；`LLM_INVALID_KEY` 走 `showLlmKeyError`，其余 `toast.error(formatAppError(e))`（P2）
- [ ] 不在 props 中内联创建新对象/数组/函数导致子组件重复渲染（大列表、图谱场景 P2）
- [ ] a11y：`img` 有 `alt`，可点击非交互元素改用 `<button>`（P3）
- [ ] 使用 `@/` 别名，不出现 `../../` 跨目录导入（P3）
- [ ] 生产代码无调试 `console.log`（P3）

## Pass 7：规范、性能与测试

- [ ] 日志用 `tracing`，无 `println!` / `dbg!`（P2）
- [ ] 无 `#![allow(dead_code)]` 全局抑制；局部 `#[allow(dead_code)]` 有注释（P2）
- [ ] 无死代码、未使用导入、注释掉的代码块（P3）
- [ ] 组合/配对计算有上限截断（`MAX_CO_OCCURRENCE_GROUP_SIZE` 类）；全量图/列表查询有 limit（P2）
- [ ] 循环内无重复 SQL 查询（N+1）、无重复加载模型/大文件（P2）
- [ ] 新增业务逻辑有测试：Rust 模块内 `#[cfg(test)]`、前端同目录 `*.test.tsx`；修 bug 有回归测试（P2）
- [ ] 文档注释为中文；公开函数可能失败时有 `# Errors`（P3）
- [ ] 用户可见行为变化同步 README / CHANGELOG / `docs/`（P3）
- [ ] 新依赖已说明理由并经用户确认（P2）
- [ ] 提交信息（若已有）符合英文 Conventional Commits 三段式，且不提及 AI 工具/内部目录（P3）
