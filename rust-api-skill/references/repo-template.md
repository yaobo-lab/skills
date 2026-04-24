# Repo 模板

数据持久化层标准写法。

```rust
// src/repos/user_repo.rs

use sea_orm::{DatabaseConnection, EntityTrait, ActiveModelTrait, Set, Condition, QueryFilter, PaginatorTrait, QueryOrder};
use crate::entities::user;
use crate::types::error::AppError;

pub struct UserRepo;

impl UserRepo {
    /// 根据ID查询
    pub async fn find_by_id(
        db: &DatabaseConnection,
        id: Uuid,
    ) -> Result<Option<user::Model>, AppError> {
        let result = user::Entity::find_by_id(id)
            .one(db)
            .await
            .map_err(|e| AppError::DbError(e.to_string()))?;
        Ok(result)
    }

    /// 分页查询
    pub async fn find_paginated(
        db: &DatabaseConnection,
        page: u64,
        page_size: u64,
        keyword: Option<String>,
    ) -> Result<(Vec<user::Model>, u64), AppError> {
        let mut condition = Condition::all();
        if let Some(kw) = keyword {
            condition = condition
                .add(user::Column::Username.contains(&kw))
                .add(user::Column::Email.contains(&kw));
        }

        let paginator = user::Entity::find()
            .filter(condition)
            .order_by_desc(user::Column::CreatedAt)
            .paginate(db, page_size);

        let total = paginator.num_items().await
            .map_err(|e| AppError::DbError(e.to_string()))?;
        let items = paginator.fetch_page(page.saturating_sub(1)).await
            .map_err(|e| AppError::DbError(e.to_string()))?;

        Ok((items, total))
    }

    /// 新增
    pub async fn create(
        db: &DatabaseConnection,
        active_model: user::ActiveModel,
    ) -> Result<user::Model, AppError> {
        let result = active_model.insert(db).await
            .map_err(|e| AppError::DbError(e.to_string()))?;
        Ok(result)
    }

    /// 更新
    pub async fn update(
        db: &DatabaseConnection,
        active_model: user::ActiveModel,
    ) -> Result<user::Model, AppError> {
        let result = active_model.update(db).await
            .map_err(|e| AppError::DbError(e.to_string()))?;
        Ok(result)
    }

    /// 删除
    pub async fn delete_by_id(
        db: &DatabaseConnection,
        id: Uuid,
    ) -> Result<(), AppError> {
        user::Entity::delete_by_id(id)
            .exec(db)
            .await
            .map_err(|e| AppError::DbError(e.to_string()))?;
        Ok(())
    }
}
```

## 约束

- 仅负责 SeaORM CRUD、条件查询、联表、分页、事务封装
- 所有数据库错误统一转为 `AppError::DbError`
- **禁止**任何 HTTP 相关逻辑、业务判断、数据格式化、脱敏
- **禁止**耦合前端响应结构
