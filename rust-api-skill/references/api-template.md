# API Handler 模板

接口路由层标准写法。

```rust
// src/web/api/user_api.rs

use axum::{
    extract::{Path, Query, State},
    Json,
};
use validator::Validate;
use crate::dtos::user_dto::{CreateUserReq, UpdateUserReq, ListUserReq, UserResp, UserListResp};
use crate::services::user_service::user_service;
use crate::types::app_state::AppState;
use crate::types::error::AppError;

/// 创建用户
pub async fn create_user(
    State(state): State<AppState>,
    Json(req): Json<CreateUserReq>,
) -> Result<Json<ApiResult<UserResp>>, AppError> {
    req.validate()?;
    let resp = user_service::create(&state, &req).await?;
    Ok(Json(ApiResult::success(resp)))
}

/// 用户列表
pub async fn list_users(
    State(state): State<AppState>,
    Query(req): Query<ListUserReq>,
) -> Result<Json<ApiResult<UserListResp>>, AppError> {
    let resp = user_service::list(
        &state,
        req.page.unwrap_or(1),
        req.page_size.unwrap_or(20),
        req.keyword,
    ).await?;
    Ok(Json(ApiResult::success(resp)))
}

/// 查询用户详情
pub async fn get_user(
    State(state): State<AppState>,
    Path(id): Path<Uuid>,
) -> Result<Json<ApiResult<UserResp>>, AppError> {
    let resp = user_service::get(&state, id).await?;
    Ok(Json(ApiResult::success(resp)))
}

/// 更新用户
pub async fn update_user(
    State(state): State<AppState>,
    Path(id): Path<Uuid>,
    Json(req): Json<UpdateUserReq>,
) -> Result<Json<ApiResult<UserResp>>, AppError> {
    req.validate()?;
    let resp = user_service::update(&state, id, &req).await?;
    Ok(Json(ApiResult::success(resp)))
}

/// 删除用户
pub async fn delete_user(
    State(state): State<AppState>,
    Path(id): Path<Uuid>,
) -> Result<Json<ApiResult<()>>, AppError> {
    user_service::delete(&state, id).await?;
    Ok(Json(ApiResult::success(())))
}


```

## 约束

- 仅负责请求解析、参数接收、调用 Service、组装响应
- 入参使用 `Json` / `Path` / `Query` 提取
- 统一通过 `State(state)` 注入 AppState
- 统一包装 `ApiResult<T>` 响应
- **禁止**直接操作数据库、编写业务逻辑、硬编码配置
