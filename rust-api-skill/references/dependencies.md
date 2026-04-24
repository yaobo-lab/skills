# 固定依赖版本（Cargo.toml）

依赖版本锁定，不可随意修改。

```toml

# basic
chrono = { version = "0.4.44", features = ["serde"] }
serde = { version = "1.0.228", features = ["derive"] }
serde_json = "1.0.149"
uuid = { version = "1.23.0", features = ["v4", "serde"] }

# Cache
moka-cache = "1.1.2" 

# config
config = { version = "0.15", default-features = false, features = ["toml"] }

# HTTP
axum = { version = "0.8.8", features = ["multipart", "http1", "http2", "tokio", "macros"] }
tower = "0.5.3"
tower-http = { version = "0.6.8", features = ["fs", "compression-gzip", "compression-br", "trace", "cors", "add-extension", "request-id", "set-header", "limit"] }
socket2 = { version = "0.6.3", features = ["all"] }
validator = { version = "0.20.0", features = ["derive"] }
headers = "0.4.1"


# HTTP client
reqwest = { version = "0.13.2", default-features = false, features = ["json", "form", "rustls"] }

# Database
sea-orm = { version = "1.1.2", default-features = false, features = ["macros", "sqlx-mysql", "runtime-tokio-rustls", "with-chrono", "with-json", "with-rust_decimal", "with-uuid"] }
sea-orm-migration = { version = "1.1.2", features = ["runtime-tokio-rustls", "sqlx-mysql"] }

# Tokio
tokio = { version = "1.49.0", features = ["rt-multi-thread", "macros", "io-util", "net", "time", "sync", "fs"] }
tokio-util = { version = "0.7.18", features = ["io", "codec", "compat"] }

# Logging 
toolkit-rs = "1.0.24"
anyhow = "1"
thiserror = "2.0.18"
log = "0.4.29"

# JWT
jsonwebtoken = { version = "10.3.0", features = ["rust_crypto"] }

# swagger
utoipa = { version = "5.4.0", features = ["chrono", "uuid"] }
utoipa-swagger-ui = { version = "9.0.2", features = ["axum"] }
utoipa-axum = "0.2.0"

# Async trait
async-trait = "0.1.89"

# Validation library
garde = { version = "0.22.0", features = ["full"] }
```

 