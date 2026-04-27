# Router 模板

全局路由聚合层标准写法

```rust
// src/web/routes.rs
 
use crate::web::api::{self, *};
use crate::web::middleware::*;
use crate::web::nextcloud::*;
use crate::{
    types::{self, config::AppConfig, state::AppState},
    web::routes,
    web::swagger,
};
use axum::http::HeaderValue;
use axum::http::header::HeaderName;
use axum::{
    Router,
    extract::DefaultBodyLimit,
    routing::{any, delete, get, post, put},
};
use std::sync::Arc;
use tower_http::compression::CompressionLayer;
use tower_http::compression::predicate::{NotForContentType, Predicate, SizeAbove};
use tower_http::cors::{Any, CorsLayer};
use tower_http::set_header::SetResponseHeaderLayer;
use tower_http::trace::TraceLayer;

/// 聚合全部业务路由
pub fn routes(app_state: Arc<AppState>, config: &AppConfig) -> Router<AppState> {
  let mut app =  Router::new()
        .route("/health", get(health))
        .nest("/api", api_routes(&app_state))
        .merge(swagger_router());

    if config.is_debug_run_mode() {
        log::info!("HTTP request tracing enabled because run_mod=debug");
        app = app.layer(TraceLayer::new_for_http());
    }

    if config.swagger.enabled {
        log::info!("Swagger UI enabled at /swagger");
        app = app.merge(swagger::swagger_routes());
    }
    with_csp_and_compression(app, &config)
}
/// 健康检查
async fn health() -> &'static str {
    "ok"
}

/// 聚合 API 路由
fn api_routes(app_state: &Arc<AppState>) -> Router<AppState> {
    Router::new()
        .merge(user_router(app_state))
        // .merge(order_router())
        // .merge(auth_router())
}


/// 路由注册
  fn user_router(app_state: &Arc<AppState>) -> Router<AppState> {
    Router::new()
        .route("/api/users", post(user_api::create_user).get(user_api::list_users))
        .route("/api/users/{id}", get(user_api::get_user).put(user_api::update_user).delete(user_api::delete_user))
}


fn with_csp_and_compression(
    mut app: Router<Arc<AppState>>,
    config: &types::config::AppConfig,
) -> Router<Arc<AppState>> {
    // 限制客户端发来的 HTTP request body 最大不能超过 BODY_LIMIT
    #[cfg(target_pointer_width = "64")]
    const BODY_LIMIT: usize = 10 * 1024 * 1024 * 1024; // 10 GB
    #[cfg(target_pointer_width = "32")]
    const BODY_LIMIT: usize = 1024 * 1024 * 1024; // 1 GB
    app = app.layer(DefaultBodyLimit::max(BODY_LIMIT));

    // HTTP 服务启用“有条件的响应压缩”
    // 只压缩大于 256 字节、且不是 gRPC / 图片 / SSE / 压缩包 / PDF / 音视频 等类型的响应。”
    // 只有响应体大于 256 字节才压缩。太小的内容压缩收益不大，反而增加 CPU 开销
    let predicate = SizeAbove::new(256)
        //gRPC 响应不压缩
        .and(NotForContentType::GRPC)
        // 图片不压缩。因为 jpg/png/webp 这类通常本身已经压缩过了
        .and(NotForContentType::IMAGES)
        // SSE 响应不压缩
        .and(NotForContentType::SSE)
        // 常见的不可压缩类型
        .and(NotForContentType::const_new("application/octet-stream"))
        .and(NotForContentType::const_new("application/zip"))
        .and(NotForContentType::const_new("application/gzip"))
        .and(NotForContentType::const_new("application/x-tar"))
        .and(NotForContentType::const_new("application/pdf"))
        .and(NotForContentType::const_new("video/"))
        .and(NotForContentType::const_new("audio/"));
    app = app.layer(CompressionLayer::new().compress_when(predicate));
    app = apply_cors_config(app, &config.web.cors);
    let csp = if config.swagger.enabled {
        "default-src 'self'; \
             script-src 'self' https://cdn.jsdelivr.net; \
             style-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; \
             img-src 'self' data: blob:; \
             media-src 'self' blob:; \
             connect-src 'self'; \
             font-src 'self' data: https://cdn.jsdelivr.net; \
             frame-src * blob:; \
             frame-ancestors 'none'; \
             base-uri 'self'; \
             form-action 'self'"
    } else {
        "default-src 'self'; \
             script-src 'self'; \
             style-src 'self' 'unsafe-inline'; \
             img-src 'self' data: blob:; \
             media-src 'self' blob:; \
             connect-src 'self'; \
             font-src 'self' data:; \
             frame-src * blob:; \
             frame-ancestors 'none'; \
             base-uri 'self'; \
             form-action 'self'"
    };

    app = app
        .layer(SetResponseHeaderLayer::overriding(
            HeaderName::from_static("content-security-policy"),             
            HeaderValue::from_str(csp).expect("valid CSP header"),
        ))
        .layer(SetResponseHeaderLayer::overriding(
            HeaderName::from_static("x-content-type-options"),
            HeaderValue::from_static("nosniff"),
        )) 
        .layer(SetResponseHeaderLayer::overriding(
            HeaderName::from_static("x-frame-options"),
            HeaderValue::from_static("DENY"),
        ))
        
        .layer(SetResponseHeaderLayer::overriding(
            HeaderName::from_static("referrer-policy"),
            HeaderValue::from_static("strict-origin-when-cross-origin"),
        )) 
        .layer(SetResponseHeaderLayer::overriding(
            HeaderName::from_static("permissions-policy"),
            HeaderValue::from_static("camera=(), microphone=(), geolocation=()"),
        ));
    app
}



fn apply_cors_config(
    app: Router<Arc<AppState>>,
    cors: &types::config::CorsConfig,
) -> Router<Arc<AppState>> {
    if !cors.enabled {
        return app;
    }

    let mut layer = CorsLayer::new();

    let methods = cors
        .parsed_methods()
        .expect("invalid [web.cors].allow_methods config");
    layer = layer.allow_methods(methods);

    let headers = cors
        .parsed_headers()
        .expect("invalid [web.cors].allow_headers config");
    layer = layer.allow_headers(headers);

    if cors.allow_credentials {
        layer = layer.allow_credentials(true);
    }

    if cors.allow_origins.iter().any(|origin| origin.trim() == "*") {
        layer = layer.allow_origin(Any);
    } else {
        let origins = cors
            .parsed_origins()
            .expect("invalid [web.cors].allow_origins config");
        layer = layer.allow_origin(origins);
    }

    app.layer(layer)
}
```

## 约束

- 仅负责聚合各业务模块 Router、统一前缀、挂载公共路由
- 单个业务模块路由定义放在 `src/web/api/*_api.rs`，此处只做 `.merge(...)` 或 `.nest(...)`
- 公共前缀统一使用 `/api`，如需版本化统一改为 `/api/v1`
- 健康检查、Swagger、静态资源等非业务入口可在此集中注册
- 返回类型统一为 `Router<AppState>`，由 `startup.rs` 继续挂载中间件、注入 state、组装顶层应用
- 需要区分公开接口和鉴权接口时，拆分为 `public_routes()` 与 `protected_routes()`，再由顶层统一合并
- **禁止**在此编写业务逻辑、数据库访问、参数校验、权限判断
- **禁止**在此重复定义 handler，handler 必须复用 `api-template.md` 中的模块函数
- **禁止**硬编码配置、密钥或环境差异路径
