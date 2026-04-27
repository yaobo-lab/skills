# Service 模板

业务逻辑层标准写法。

```rust
// src/services/user_service.rs

use crate::dtos::user_dto::{CreateUserReq, UpdateUserReq, UserResp, UserListResp};
use crate::repos::user_repo;
use crate::types::app_state::AppState;
use crate::types::error::AppError;
use crate::utils::password;
 
/// 创建用户
pub async fn create(
    state: &AppState,
    req: &CreateUserReq,
) -> Result<UserResp, AppError> {
    // 业务校验：邮箱唯一性
    if let Some(_) = user_repo::find_by_email(&state.db, &req.email).await? {
        return Err(AppError::BizError("邮箱已存在".to_string()));
    }

    // 密码哈希
    let hash = password::hash(&req.password)
        .map_err(|e| AppError::BizError(e.to_string()))?;

    // 构建实体并持久化
    let active_model = user::ActiveModel {
        id: Set(Uuid::new_v4()),
        username: Set(req.username.clone()),
        email: Set(req.email.clone()),
        password_hash: Set(hash),
        status: Set(1),
        created_at: Set(chrono::Utc::now().naive_utc()),
        updated_at: Set(chrono::Utc::now().naive_utc()),
    };

    let model = user_repo::create(&state.db, active_model).await?;
    Ok(model.into())
}

/// 分页查询
pub async fn list(
    state: &AppState,
    page: u64,
    page_size: u64,
    keyword: Option<String>,
) -> Result<UserListResp, AppError> {
    let (items, total) = user_repo::find_paginated(&state.db, page, page_size, keyword).await?;
    Ok(UserListResp {
        list: items.into_iter().map(|m| m.into()).collect(),
        total,
        page,
        page_size,
    })
}

/// 更新用户
pub async fn update(
    state: &AppState,
    id: Uuid,
    req: &UpdateUserReq,
) -> Result<UserResp, AppError> {
    let existing = user_repo::find_by_id(&state.db, id)
        .await?
        .ok_or(AppError::NotFound("用户不存在".to_string()))?;

    let mut active_model: user::ActiveModel = existing.into();
    if let Some(ref username) = req.username {
        active_model.username = Set(username.clone());
    }
    if let Some(ref email) = req.email {
        active_model.email = Set(email.clone());
    }
    active_model.updated_at = Set(chrono::Utc::now().naive_utc());

    let model = user_repo::update(&state.db, active_model).await?;
    Ok(model.into())
}

/// 删除用户
pub async fn delete(state: &AppState, id: Uuid) -> Result<(), AppError> {
    user_repo::find_by_id(&state.db, id)
        .await?
        .ok_or(AppError::NotFound("用户不存在".to_string()))?;
    user_repo::delete_by_id(&state.db, id).await
}
 
```

## 约束

- 业务编排、权限校验、缓存协调、事务控制、外部三方调用
- **禁止**依赖 HTTP 原生 Request/Response
- **禁止**编写路由、状态码、响应格式化
- **禁止**直接操作数据库（必须依赖 Repo 层）
- 通过 `&AppState` 获取依赖，不使用全局静态变量
