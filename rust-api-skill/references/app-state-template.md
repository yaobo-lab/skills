# AppState 模板

全局状态管理标准写法。

```rust
// src/types/app_state.rs

use sea_orm::DatabaseConnection;
use moka::future::Cache;
use std::sync::Arc;
use crate::types::config::AppConfig;
use moka_cache::MokaCache;

#[derive(Clone)]
pub struct AppState {
    pub db: DatabaseConnection,
    pub config: Arc<AppConfig>,
    pub cache: Arc<MokaCache>,
}
```

## 约束

- 全局资源（配置、DB连接池、缓存）全部收拢至 `AppState`
- `AppState` 必须实现 `Clone`（Axum State 要求）
- 配置使用 `Arc<AppConfig>` 共享引用
- **禁止**使用 `lazy_static` / `once_cell` 全局静态变量
- 各层构造函数统一传入 `&AppState` 完成依赖注入
