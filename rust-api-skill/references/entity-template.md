# Entity 模板

SeaORM 数据库实体标准写法。

```rust
// src/entities/user.rs

use sea_orm::entity::prelude::*;
use serde::{Deserialize, Serialize};

#[derive(Clone, Debug, PartialEq, DeriveEntityModel, Serialize, Deserialize)]
#[sea_orm(table_name = "user")]
pub struct Model {
    #[sea_orm(primary_key, auto_increment = false)]
    pub id: Uuid,
    pub username: String,
    pub email: String,
    pub password_hash: String,
    pub status: i16,
    pub created_at: ChronoDateTime,
    pub updated_at: ChronoDateTime,
}

#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {}

impl ActiveModelBehavior for ActiveModel {}
```

## 约束

- 字段与数据库结构严格对齐
- 统一使用 `Uuid`、`chrono::NaiveDateTime`、`serde_json::Value`、`rust_decimal::Decimal` 规范类型
- 只做模型定义，不写业务方法
- 主键统一使用 `Uuid`，`auto_increment = false`
