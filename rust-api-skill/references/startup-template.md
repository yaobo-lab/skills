# Router 模板

全局路由聚合层标准写法

```rust
// src/startup.rs

use crate::{
    types::{self, config::AppConfig, state::AppState},
    web::{self, routes, swagger},
};
use anyhow::anyhow;
use axum::Router;
use socket2::{Domain, Protocol, Socket, TcpKeepalive, Type};
use std::net::ToSocketAddrs;
use std::sync::Arc;
use std::time::Duration;
use toolkit_rs::AppResult;
use toolkit_rs::logger;
use tower_http::trace::TraceLayer;
use types::db::{create_database_pools, run_startup_migrations_if_enabled};
use types::state::AppStateBuilder;
 
async fn build_router() -> (Router<Arc<AppState>>, Arc<AppState>) {
    let config = AppConfig::from_default_file();

    logger::setup(config.log.clone()).expect("init logger failed");

    if config.auth.cookie_same_site() == "None" && !config.auth.cookie_secure {
        log::warn!(
            "auth.cookie_same_site=None but auth.cookie_secure=false; many browsers will reject cross-site cookies unless Secure is enabled over HTTPS"
        );
    }

    let pools = match create_database_pools(&config).await {
        Ok(p) => {
            log::info!("MySQL database pools initialized successfully");
            p
        }
        Err(e) => {
            panic!(
                "FATAL: enable_auth=true but database connection failed: {}. \
                     Refusing to start without authentication.",
                e
            );
        }
    };

    if let Err(e) = run_startup_migrations_if_enabled(&config, &pools.maintenance).await {
        panic!(
            "FATAL: database migrations were enabled but failed to run: {}. \
                 Refusing to start with a partially migrated schema.",
            e
        );
    }


    let mut app = routes::routes();
    if config.is_debug_run_mode() {
        log::info!("HTTP request tracing enabled because run_mod=debug");
        app = app.layer(TraceLayer::new_for_http());
    }

    if config.swagger.enabled {
        log::info!("Swagger UI enabled at /swagger");
        app = app.merge(swagger::swagger_routes());
    }

    let app = routes::with_csp_and_compression(app, &config);
    (app, app_state)
}

pub async fn start() -> AppResult {
    let (app, app_state) = build_router().await;
    let config = &app_state.config;
    let mut addrs = (config.web.host.clone(), config.web.port).to_socket_addrs()?;
    let addr = addrs
        .next()
        .ok_or_else(|| anyhow!("Failed to resolve bind address for {}", config.web.host))?;

    log::info!("Starting app server on {}", config.base_url());

    let domain = if addr.is_ipv4() {
        Domain::IPV4
    } else {
        Domain::IPV6
    };
    let socket = Socket::new(domain, Type::STREAM, Some(Protocol::TCP))?;
    socket.set_reuse_address(true)?;
    #[cfg(not(windows))]
    socket.set_reuse_port(true)?;
    socket.set_tcp_nodelay(true)?;
    socket.set_keepalive(true)?;
    socket.set_tcp_keepalive(
        &TcpKeepalive::new()
            .with_time(Duration::from_secs(60))
            .with_interval(Duration::from_secs(10)),
    )?;
    socket.set_nonblocking(true)?;
    socket.bind(&addr.into())?;
    socket.listen(2048)?;
    let listener = tokio::net::TcpListener::from_std(socket.into())?;
    let app = app.with_state(app_state);
    axum::serve(listener, app).await?;
    log::info!("Server shutdown completed");
    Ok(())
}

```

## 约束

 