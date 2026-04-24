# AppError 模板

统一错误处理标准写法。

```rust
// src/types/error.rs

use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use axum::Json;
use serde::Serialize;

#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("数据库错误: {0}")]
    DbError(String),

    #[error("参数校验失败: {0}")]
    ValidationError(String),

    #[error("鉴权失败: {0}")]
    AuthError(String),

    #[error("权限不足: {0}")]
    Forbidden(String),

    #[error("资源不存在: {0}")]
    NotFound(String),

    #[error("业务错误: {0}")]
    BizError(String),

    #[error("内部错误: {0}")]
    Internal(String),
}

#[derive(Serialize)]
struct ErrorResponse {
    code: i32,
    message: String,
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code, message) = match &self {
            AppError::DbError(msg) => (StatusCode::INTERNAL_SERVER_ERROR, 50001, msg.clone()),
            AppError::ValidationError(msg) => (StatusCode::BAD_REQUEST, 40001, msg.clone()),
            AppError::AuthError(msg) => (StatusCode::UNAUTHORIZED, 40101, msg.clone()),
            AppError::Forbidden(msg) => (StatusCode::FORBIDDEN, 40301, msg.clone()),
            AppError::NotFound(msg) => (StatusCode::NOT_FOUND, 40401, msg.clone()),
            AppError::BizError(msg) => (StatusCode::BAD_REQUEST, 40002, msg.clone()),
            AppError::Internal(msg) => (StatusCode::INTERNAL_SERVER_ERROR, 50000, msg.clone()),
        };

        (status, Json(ErrorResponse { code, message })).into_response()
    }
}

/// garde 校验错误自动转换
impl From<garde::Errors> for AppError {
    fn from(errs: garde::Errors) -> Self {
        AppError::ValidationError(errs.to_string())
    }
}
```

## 统一成功响应包装

```rust
// src/types/response.rs

use serde::Serialize;

#[derive(Serialize)]
pub struct ApiResult<T: Serialize> {
    pub code: i32,
    pub message: String,
    pub data: Option<T>,
}

impl<T: Serialize> ApiResult<T> {
    pub fn success(data: T) -> Self {
        Self {
            code: 0,
            message: "ok".to_string(),
            data: Some(data),
        }
    }

    pub fn error(code: i32, message: String) -> ApiResult<()> {
        ApiResult {
            code,
            message,
            data: None,
        }
    }
}
```

## 约束

- 全局唯一错误类型 `AppError`
- 实现 `IntoResponse`，统一标准化 JSON 输出
- 数据库、参数、鉴权、业务错误统一枚举分支
- 业务层直接返回 `Err(AppError::XXX)`，无需手动处理状态码
