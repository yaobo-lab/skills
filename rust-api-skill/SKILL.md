---
name: rust-api-skill
description: Rust Axum REST API 开发规范与模板 Skill。统一所有 Rust 后端 RESTful 项目的分层架构、依赖版本、编码规范和目录结构。当用户需要：(1) 创建新的 Rust 后端 REST API 项目，(2) 在现有 Rust Axum 项目中新增业务模块，(3) 编写/审查 Rust 后端代码是否符合分层规范，(4) 生成 Axum 项目的目录骨架或模板代码，(5) 查询 Rust API 项目的依赖版本、命名规范、错误处理方式，(6) 讨论或实施 Rust 后端的架构分层、数据流、状态管理等规范时使用此 Skill。触发关键词：Rust API、Axum、REST API、后端模板、分层架构、SeaORM、新增模块。
---

# Rust Axum REST API 开发规范 & 模板

所有 Rust 后端 RESTful 项目，统一使用该分层模板、依赖版本、编码规范、目录结构，强制统一收口。

## 一、分层架构与调用链

核心调用链：**web/api → services → repos**

严格单向依赖，禁止跨层逆向调用（repo → service 禁止反向依赖）。

## 二、固定目录结构

```
rust-api/
├── etc/
│   └── config.toml              # 全局 TOML 配置文件
├── migrations/                  # SeaORM 数据库迁移
├── src/
│   ├── web/
│   │   ├── api/                 # 接口路由层
│   │   └── middleware/          # 全局中间件
│   │   └── routes.rs            # 接口路由注册
│   │   └── swagger.rs           # Swagger 文档配置
│   │   └── mod.rs                # 接口路由层模块导出
│   ├── services/                # 业务逻辑层
│   ├── repos/                   # 数据持久化层
│   ├── entities/                # SeaORM 数据库实体
│   ├── dtos/                    # 请求/响应传输对象
│   ├── types/                   # 全局基础类型、配置、错误、状态
│   ├── utils/                   # 通用工具函数
│   ├── startup.rs               # 服务初始化入口
│   ├── main.rs                  # 程序启动入口
│   └── lib.rs                   # 模块统一导出
└── Cargo.toml                   # 锁定固定依赖版本
```

## 三、各层职责与强制约束

### 1. src/web/api — 接口路由层

**仅负责**：请求解析、参数接收、调用 Service、组装响应、Header/Cookie 设置、DTO 入参透出。

| 允许                                 | 禁止                         |
| ------------------------------------ | ---------------------------- |
| 注册 Axum 路由、path/query/json 提取 | 直接操作数据库、编写 SQL     |
| 注入 AppState、Extension 上下文      | 复杂业务逻辑、权限判断、事务 |
| 统一包装成功/失败响应                | 硬编码配置、密钥、常量       |
| 基础参数透传与转发                   | 直接读取全局静态变量         |

### 2. src/web/middleware — 中间件层

**仅负责**横切通用能力：鉴权、限流、CORS、请求ID、日志、请求大小限制、安全头。

| 允许                              | 禁止                 |
| --------------------------------- | -------------------- |
| JWT 身份校验、权限拦截            | 业务逻辑、数据库查询 |
| 跨域、压缩、追踪链路              | 全局静态变量窃取配置 |
| 通过 Extension 显式注入用户上下文 |                      |
| 统一捕获顶层异常                  |                      |

### 3. src/services — 业务逻辑层

**仅负责**：业务编排、权限校验、缓存协调、事务控制、外部三方调用、文件流程、复杂聚合逻辑。

| 允许                         | 禁止                               |
| ---------------------------- | ---------------------------------- |
| 多 Repo 组合调用、事务管理   | 依赖 HTTP 原生 Request/Response    |
| 业务权限、数据过滤、状态流转 | 编写路由、状态码、响应格式化       |
| 缓存读写、异步任务协调       | 直接操作数据库（必须依赖 Repo 层） |
| 三方 HTTP/消息队列/存储流程  |                                    |

### 4. src/repos — 数据持久化层

**仅负责**：纯数据持久化，SeaORM 数据库 CRUD、条件查询、联表、分页、事务封装。

| 允许                       | 禁止                       |
| -------------------------- | -------------------------- |
| 实体查询、新增、更新、删除 | 任何 HTTP 相关逻辑         |
| 复杂条件构造、分页、排序   | 业务判断、数据格式化、脱敏 |
| 数据库事务隔离             | 耦合前端响应结构           |

### 5. src/entities — 数据库实体

- 仅存放 `DeriveEntityModel` 数据库实体
- 字段与数据库结构严格对齐
- 统一使用：Uuid、chrono、json、decimal 规范类型
- 只做模型定义，不写业务方法

### 6. src/dtos — 传输对象

- 入参 DTO：集成 `garde` / `validator` 校验
- 出参 DTO：精简脱敏，屏蔽敏感字段
- 命名规范：`XxxReq` / `XxxResp`
- 纯序列化结构体，无复杂业务逻辑

### 7. src/types — 全局基础类型

- `config.rs`：配置映射结构体，与 `etc/config.toml` 双向同步
- `app_state.rs`：全局 AppState，统一持有 db / config / cache
- `error.rs`：全局统一错误枚举，实现 `IntoResponse`
- 全局枚举、基础自定义类型、全局常量

### 8. src/utils — 通用工具

- 无状态通用工具函数
- 密码哈希、JWT 工具、时间处理、加解密、字符串、文件辅助
- 纯静态方法，不依赖 AppState

### 9. startup.rs — 初始化入口

全局唯一初始化入口，按序执行：

1. 加载 `etc/config.toml` 配置
2. 初始化日志组件
3. 构建 AppState（连接池、缓存、配置）
4. 统一挂载所有中间件
5. 聚合注册全部业务路由
6. 组装顶层 Router 实例

## 四、新增业务模块标准流程

严格按顺序开发：

1. **entities** — 编写数据库实体
2. **migrations** — 编写迁移脚本
3. **dtos** — 定义请求/响应结构体 + 参数校验
4. **repos** — 实现数据层 CRUD 方法
5. **services** — 实现业务逻辑与流程编排
6. **web/api** — 编写 Handler 与路由
7. **startup.rs** — 统一挂载路由
8. **middleware** — 如需鉴权，接入权限中间件

## 五、状态与依赖注入规范

1. 全局资源（配置、DB连接池、缓存）全部收拢至 `AppState`
2. **禁止**使用 `lazy_static` / `once_cell` 全局静态变量
3. 用户身份、上下文信息：使用 `Extension<T>` 显式传递
4. 各层构造函数统一传入 `&AppState` 完成依赖注入

## 六、统一错误处理

1. 全局唯一错误类型：`types::error::AppError`
2. 实现 `IntoResponse`，统一标准化 JSON 输出
3. 数据库、参数、鉴权、业务错误统一枚举分支
4. 业务层直接返回 `Err(AppError::XXX)`，无需手动处理状态码

## 七、配置文件规范

1. 配置文件固定路径：`etc/config.toml`
2. 配置结构体存放：`src/types/config.rs`
3. 新增配置项：先修改结构体 → 同步补充 config.toml 默认值
4. **全程仅使用 TOML，禁止 YAML**
5. **禁止代码硬编码**任何环境配置、密钥、地址

## 八、代码命名规范

| 对象        | 规范                           | 示例                         |
| ----------- | ------------------------------ | ---------------------------- |
| 文件/模块   | 蛇形命名                       | `user_service.rs`            |
| 结构体/枚举 | 大驼峰                         | `UserService` / `AppError`   |
| 函数/方法   | 蛇形命名                       | `get_by_id`                  |
| DTO         | 请求 `XxxReq` / 响应 `XxxResp` | `CreateUserReq` / `UserResp` |
| 路由路径    | 小写+短横线                    | `/api/user-profile`          |

## 九、模板复用规则

1. 新建项目直接拷贝完整模板
2. 清空示例业务代码，保留基础骨架与分层
3. **禁止**删减目录、破坏分层依赖关系
4. **禁止**跨层逆向调用：repo → service 禁止反向依赖
5. 基础中间件、错误处理、AppState、配置加载**禁止**随意改造

## 十、固定依赖版本

依赖版本锁定，不可随意修改。新建项目或新增依赖时参见：

- **详细依赖清单**：参见 [references/dependencies.md](references/dependencies.md)

## 十一、代码模板

新增业务模块时的骨架代码模板：

- **Entity 模板**：参见 [references/entity-template.md](references/entity-template.md)
- **DTO 模板**：参见 [references/dto-template.md](references/dto-template.md)
- **Repo 模板**：参见 [references/repo-template.md](references/repo-template.md)
- **Service 模板**：参见 [references/service-template.md](references/service-template.md)
- **API Handler 模板**：参见 [references/api-template.md](references/api-template.md)
- **API Router 模板**：参见 [references/router-template.md](references/router-template.md)
- **AppState 模板**：参见 [references/app-state-template.md](references/app-state-template.md)
- **AppError 模板**：参见 [references/error-template.md](references/error-template.md)
- **startup.rs 模板**：参见 [references/startup-template.md](references/startup-template.md)