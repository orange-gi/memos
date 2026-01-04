# Memos 项目介绍（Why / What / How）

## 背景与目标
Memos 是一个**自托管**的知识管理/备忘录平台。它以 “Memo” 为核心资源，提供创建、检索、组织与分享等能力，并围绕认证鉴权、附件、过滤与快捷方式、实例配置、插件扩展等构建完整系统。

本项目采用**契约优先**（Protocol Buffers）与**清晰分层**（API 层 / 数据层 / 前端交互层 / 插件与后台任务）的方式，确保 API 稳定、可扩展、易演进。

---

## Why（为什么做）
- **自托管与数据可控**：将知识资产保留在用户/组织可控的环境中，满足隐私、合规与备份需求。
- **统一入口的知识沉淀**：以 Memo 为最小单元，支持附件、标签/链接/任务解析、关联与反应等能力，让记录与回溯更高效。
- **面向扩展与集成**：通过明确的 API 契约与插件体系，便于接入对象存储、Webhook、邮件等外部能力，同时保持核心稳定。

---

## What（是什么）
从“对外提供的能力”视角，系统主要包含以下部分：

### 核心资源与能力
- **Memo**：创建、查询、过滤、排序、关联（关系/附件/反应等）。
- **User / Identity**：用户、注册与统计、身份提供方（IDP）相关能力、权限控制。
- **Attachment / Storage**：附件上传与访问；可选接入 S3 等对象存储。
- **Instance / Setting**：实例信息、站点级配置、版本与迁移状态。
- **Shortcut / Filter**：保存筛选条件、快捷访问。

### API 形态（双协议）
- **Connect RPC**：面向浏览器/前端的类型安全 RPC（Connect 协议）。
- **gRPC-Gateway（REST）**：提供 `/api/v1/*` 风格 HTTP/JSON 接口，便于外部工具与集成。

### 运行与部署形态
- **单体服务**：一个 HTTP Server 承载 API、健康检查与静态资源。
- **后台任务（runner）**：承载异步或周期性工作，避免阻塞在线请求。
- **插件（plugin）**：承载可选能力（Webhook、邮件、Markdown、HTTP 获取、S3 存储等）。

---

## How（怎么实现）
### 1) 契约优先：Proto 定义系统对外契约
- Proto 定义位于：`proto/api/v1/*.proto`
- 生成与校验：`proto/buf.yaml`、`proto/buf.gen.yaml`、`proto/buf.lock`
- 生成产物：
  - Go：`proto/gen/api/v1/`（后端实现与路由依赖）
  - TypeScript：`web/src/types/proto/`（前端类型与客户端依赖）

这使得“服务接口、请求/响应结构、资源命名”成为稳定边界，减少前后端与外部集成的协作成本。

### 2) 后端：API 编排 + 认证鉴权 + 数据访问
后端代码主要位于 `server/` 与 `store/`：

- **HTTP Server 与路由**
  - `server/server.go`：HTTP Server 初始化、健康检查、runner 启动等。
  - `server/router/api/v1/`：API v1 服务实现、Connect 与 Gateway 注册、公共接口白名单等。

- **认证鉴权**
  - `server/auth/`：JWT/PAT 等认证逻辑与上下文注入。
  - `server/router/api/v1/connect_interceptors.go`：Connect 拦截器链（元数据、日志、恢复、认证等）。
  - `server/router/api/v1/acl_config.go`：公共接口白名单（无需认证的入口）。

- **数据层与迁移**
  - `store/`：统一 `Driver` 接口与缓存包装，屏蔽 SQLite/MySQL/Postgres 差异。
  - `store/migrator.go` 与 `store/migration/**`：版本化迁移与 LATEST 基线脚本。

### 3) 前端：React SPA + React Query 管理服务端状态
前端代码位于 `web/`：
- `web/src/hooks/`：基于 React Query 的数据获取、缓存与更新（服务端状态）。
- `web/src/contexts/`：认证态、视图偏好、过滤器等客户端状态。
- `web/src/lib/connect.ts`：Connect 客户端配置与调用入口。

### 4) 异步与扩展：runner 与 plugin 将可选能力隔离
- `server/runner/`：后台任务（如 memo payload 处理、S3 预签名等）。
- `plugin/`：可选扩展能力（调度、邮件、Webhook、Markdown、HTTP 获取、S3 存储等）。

这种拆分保证在线请求链路更“短更稳”，同时让扩展能力可以独立演进。

---

## 系统边界（System Boundary）
从“系统内/系统外”划分系统责任范围：

### 系统内（本项目负责）
- 对外 API：Connect RPC 与 REST Gateway
- 认证鉴权、请求拦截器链、业务服务实现（API v1）
- 数据访问与缓存、数据库迁移
- 前端 SPA 静态资源（可由后端托管或独立部署）
- 插件框架与内置插件、后台 runner

### 系统外（依赖或被集成）
- **客户端**：浏览器前端、CLI、第三方系统（通过 REST/Connect 接入）
- **数据库**：SQLite / MySQL / PostgreSQL（持久化边界）
- **对象存储（可选）**：S3 兼容服务（附件存储边界）
- **邮件（可选）**：SMTP 服务（邮件插件边界）
- **Webhook 接收方（可选）**：外部 HTTP 服务（事件分发边界）
- **运行环境**：文件系统、网络、反向代理、证书、监控等

---

## 模块边界（Module Boundary）
按“职责 + 输入/输出”划分主要模块，并明确推荐的依赖方向。

### 1) `proto/`：契约边界（Contract）
- **职责**：定义对外 API、消息结构与资源命名；约束兼容性。
- **输入**：领域需求与资源抽象。
- **输出**：生成的 Go/TS 代码与 OpenAPI（如有）。

### 2) `server/router/api/v1/`：应用服务边界（Application Service）
- **职责**：用例编排（鉴权、校验、调用 store、组装响应、返回标准错误）。
- **输入**：RPC/HTTP 请求（经 Connect/Gateway 适配进入统一实现）。
- **输出**：RPC/REST 响应或结构化错误（如 NotFound/PermissionDenied 等）。

### 3) `server/auth/`：安全边界（Security）
- **职责**：认证与身份解析；将“当前用户是谁”注入上下文供业务使用。
- **原则**：业务模块不直接解析 token；通过上下文获取已认证主体。

### 4) `store/`：数据访问边界（Data Access）
- **职责**：统一 `Driver` 接口与缓存；屏蔽不同数据库差异；负责迁移与一致性相关约束。
- **输入**：应用层的查询/创建/更新意图（Find/Create/Update/Delete 等）。
- **输出**：领域对象与错误；不泄漏底层 SQL 细节给应用层。

### 5) `plugin/`：扩展边界（Extension）
- **职责**：承载可选、可替换的外部集成能力。
- **原则**：通过明确接口/配置接入，避免插件反向侵入核心业务逻辑。

### 6) `server/runner/`：异步边界（Async/Jobs）
- **职责**：异步/周期任务；与在线请求隔离，便于重试与可观测。
- **原则**：失败不应影响主请求可用性；需要清晰的任务边界与日志。

### 7) `web/`：交互边界（UI）
- **职责**：用户交互体验；通过 API 获取/提交数据；管理客户端状态与路由。
- **原则**：前端不直连数据库；仅通过 API 契约与后端交互。

---

## 典型请求链路（端到端）
以“前端读取/更新 Memo”为例：
1. 用户在 `web/` 触发操作（页面/组件）。
2. 前端通过 Connect 客户端调用 `memos.api.v1.*` RPC。
3. 后端 Connect 拦截器链执行：元数据 → 日志 → 恢复 → 认证（`server/auth`）→ 注入上下文。
4. 业务服务（`server/router/api/v1/*_service.go`）完成校验与用例编排，调用 `store/`。
5. `store/` 通过对应数据库驱动访问 SQLite/MySQL/Postgres，并返回结果。
6. 后端组装响应返回；前端 React Query 更新缓存并触发 UI 渲染。

