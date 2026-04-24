# DTO 模板

请求/响应传输对象标准写法。

## 请求 DTO

```rust
// src/dtos/user_dto.rs

use garde::Validate;
use serde::Deserialize;
use utoipa::ToSchema;

#[derive(Debug, Deserialize, Validate, ToSchema)]
pub struct CreateUserReq {
    #[garde(length(min = 2, max = 50))]
    pub username: String,

    #[garde(email)]
    pub email: String,

    #[garde(length(min = 8, max = 128))]
    pub password: String,
}

#[derive(Debug, Deserialize, Validate, ToSchema)]
pub struct UpdateUserReq {
    #[garde(length(min = 2, max = 50))]
    pub username: Option<String>,

    #[garde(email)]
    pub email: Option<String>,
}

#[derive(Debug, Deserialize, ToSchema)]
pub struct ListUserReq {
    pub page: Option<u64>,
    pub page_size: Option<u64>,
    pub keyword: Option<String>,
}
```

## 响应 DTO

```rust
#[derive(Debug, Serialize, ToSchema)]
pub struct UserResp {
    pub id: Uuid,
    pub username: String,
    pub email: String,
    pub status: i16,
    pub created_at: String,
    pub updated_at: String,
}

#[derive(Debug, Serialize, ToSchema)]
pub struct UserListResp {
    pub list: Vec<UserResp>,
    pub total: u64,
    pub page: u64,
    pub page_size: u64,
}
```

## 约束

- 入参 DTO 集成 `garde` 校验注解
- 出参 DTO 精简脱敏，屏蔽 `password_hash` 等敏感字段
- 命名规范：`XxxReq` / `XxxResp`
- 纯序列化结构体，无复杂业务逻辑
- 统一添加 `ToSchema` 以支持 Swagger 文档
