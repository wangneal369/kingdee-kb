# KingdeeKB 开发模板

以下模板均取自仓库现有写法，复制后按领域改名。

## 1. 新增 Tauri 命令（端到端）

### 1.1 service（`src-tauri/src/services/<domain>/foo.rs`）

```rust
//! Foo 领域服务（模块级中文说明）

use rusqlite::{params, Connection};
use serde::Serialize;
use tracing::warn;

#[derive(Debug, Serialize)]
pub struct FooItem {
    pub id: i64,
    pub name: String,
}

pub struct FooStore {
    db: Connection,
}

impl FooStore {
    pub fn new(db: Connection) -> Self {
        let _ = db.busy_timeout(std::time::Duration::from_secs(5));
        Self { db }
    }

    /// 创建 foo 表（幂等）。单一 schema，不写迁移。
    pub fn ensure_table(&self) -> Result<(), String> {
        self.db
            .execute_batch(
                "CREATE TABLE IF NOT EXISTS foo (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    project_id INTEGER NOT NULL REFERENCES projects(id),
                    name TEXT NOT NULL
                );
                CREATE INDEX IF NOT EXISTS idx_foo_project ON foo(project_id);",
            )
            .map_err(|e| format!("创建 foo 表失败: {}", e))
    }

    /// 列出项目下的 foo（必须带 project_id）。
    pub fn list(&self, project_id: i64) -> Result<Vec<FooItem>, String> {
        let mut stmt = self
            .db
            .prepare("SELECT id, name FROM foo WHERE project_id = ?1 ORDER BY id")
            .map_err(|e| e.to_string())?;
        let rows = stmt
            .query_map(params![project_id], |r| {
                Ok(FooItem { id: r.get(0)?, name: r.get(1)? })
            })
            .map_err(|e| e.to_string())?;
        rows.collect::<Result<Vec<_>, _>>().map_err(|e| e.to_string())
    }
}
```

注册模块：
- `services/<domain>/mod.rs` 加 `pub mod foo;`
- `services/mod.rs` 加 `pub use <domain>::foo;`

### 1.2 AppState（`src-tauri/src/app_state.rs`）

```rust
/// Foo 存储（foo 表）
pub foo_store: Arc<Mutex<FooStore>>,
```

`AppState::new` 中（与 `graph_store` 同样写法）：

```rust
let foo_store = {
    let conn = Connection::open(&db_path).map_err(|e| format!("打开 foo 数据库失败: {}", e))?;
    let store = FooStore::new(conn);
    store.ensure_table()?;
    Arc::new(Mutex::new(store))
};
```

### 1.3 command（`src-tauri/src/commands/<domain>.rs`）

```rust
/// 列出项目下的 foo。
#[tauri::command]
pub async fn list_foo(
    state: State<'_, AppState>,
    project_id: i64,
) -> Result<Vec<FooItem>, String> {
    let store = state
        .foo_store
        .lock()
        .map_err(|e: std::sync::PoisonError<_>| e.to_string())?;
    store.list(project_id)
}
```

如需让前端按错误码分支（如 LLM Key 失效），返回 `AppResult<T>` 并构造 `AppError::classify_llm_error(provider_id, &err)` / `AppError::InvalidArgument(..)` 等。

### 1.4 注册（`src-tauri/src/lib.rs`）

在 `tauri::generate_handler![ ... ]` 对应领域分组下追加：

```rust
commands::<domain>::list_foo,
```

**漏这一步不会编译报错**，只会在运行时报 `command list_foo not found`。

### 1.5 前端封装（`src/lib/tauri-commands.ts` 或领域 `*-commands.ts`）

```ts
/** Foo 条目（对应 Rust FooItem） */
export interface FooItem {
  id: number
  name: string
}

/** 列出项目下的 foo */
export async function listFoo(projectId: number): Promise<FooItem[]> {
  return invoke("list_foo", { projectId })
}
```

### 1.6 e2e mock（`e2e/mocks/tauri-mock.ts`）

```ts
list_foo: [],
```

### 1.7 页面调用

```tsx
import { useToast } from "@/components/Toast"
import { useAppError } from "@/contexts/AppErrorContext"
import { useProject } from "@/contexts/ProjectContext"
import { formatAppError, parseAppError } from "@/lib/app-error"
import { listFoo } from "@/lib/tauri-commands"

const { currentProjectId } = useProject()
const toast = useToast()
const { showLlmKeyError } = useAppError()

if (currentProjectId == null) return // 未选项目时显示空态，不发请求
try {
  const items = await listFoo(currentProjectId)
} catch (err) {
  const parsed = parseAppError(err)
  if (parsed?.code === "LLM_INVALID_KEY") showLlmKeyError(parsed)
  else toast.error(`加载失败: ${formatAppError(err)}`)
}
```

## 2. 新增页面

1. `src/pages/Foo.tsx`（默认导出函数组件）。
2. `src/App.tsx`：`const Foo = lazy(() => import("./pages/Foo"))` + `<Route path="foo" element={<Foo />} />`。
3. 侧边栏入口：`src/components/Layout.tsx` 的 `navItems`。
4. e2e：`e2e/*.spec.ts` 增加导航/关键路径用例。

## 3. 后台长任务

```rust
use crate::services::common::spawn_safe::spawn_monitored;

// 进入 async 前把需要的东西 clone 出来，不持 std Mutex guard 跨 await
let store = state.wiki_pages.clone();
let status = state.kb_recompile_status.clone();

spawn_monitored("foo_rebuild", Some(&app), async move {
    match do_work(store).await {
        Ok(n) => tracing::info!("foo 重建完成: {n}"),
        Err(e) => {
            tracing::error!("foo 重建失败: {e}");
            if let Ok(mut s) = status.lock() { /* 标记失败，避免前端永远显示进行中 */ }
        }
    }
});
```

前端：轮询状态命令或 `listen("<event>")`。现有事件名：`agent-event`、`ingestion-progress`、`document-imported`、`outline:changed`、`task:failed`、`meeting-sync`、`video_progress`、`app:setup_status`、`contract-scope-progress`、`spotlight-toggle`。

## 4. 持锁跨 await 的正确写法

```rust
// ❌ guard 活到 await 之后
let store = state.wiki_pages.lock().map_err(|e| e.to_string())?;
let text = llm.complete(prompt).await?;
store.save(&text)?;

// ✅ 块作用域内读完即释放，await 后再重新加锁写入
let page = {
    let store = state.wiki_pages.lock().map_err(|e| e.to_string())?;
    store.get(project_id, &slug)?
};
let text = llm.complete(prompt).await?;
{
    let store = state.wiki_pages.lock().map_err(|e| e.to_string())?;
    store.save(project_id, &slug, &text)?;
}
```

## 5. Vitest 组件测试

与源同目录 `Foo.test.tsx`，环境 happy-dom，`globals: false`（需显式 `import { describe, it, expect, vi } from "vitest"`）。参考 `src/components/wiki/GraphStatsBanner.test.tsx`。
