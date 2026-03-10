# 技术架构设计

> 文档编号：02-技术架构设计-architecture.md | 版本：1.2 | 日期：2026-03-03
> 依赖文档：overview.md（模块定义）, detailed_design.md（接口细节）

## 1. 技术栈

### 1.1 Backend

| 技术 | 版本 | 用途 |
|------|------|------|
| .NET | 10 | 运行时 |
| ASP.NET Core | 10 | Web API + SignalR |
| EF Core | 10 | ORM + Migration |
| Npgsql | latest | PostgreSQL 驱动 |
| xUnit | latest | 测试框架 |
| Moq | latest | Mock 框架 |
| FluentAssertions | latest | 断言库 |
| Coverlet | latest | 覆盖率收集 |
| Serilog | latest | 结构化日志（JSON 输出） |
| Serilog.Sinks.Console | latest | 控制台日志输出 |
| Serilog.Sinks.File | latest | 文件日志输出 |

### 1.2 Frontend

| 技术 | 版本 | 用途 |
|------|------|------|
| Next.js | 15 (App Router) | 前端框架 |
| React | 19 | UI 库 |
| TypeScript | 5.x | 类型安全 |
| Tailwind CSS | 4.x | 样式框架 |
| shadcn/ui | latest | 组件库 |
| @supabase/ssr | latest | Supabase Auth |
| @microsoft/signalr | latest | 实时通信 |
| Zustand | latest | 状态管理 |
| React Hook Form + Zod | latest | 表单 + 验证 |
| Playwright | latest | E2E 测试 |

### 1.3 基础设施

| 技术 | 用途 |
|------|------|
| Supabase | PostgreSQL + Auth + Storage（凭证文件存储） |
| GitHub Actions | CI/CD |
| Nginx | 反向代理 |

---

## 2. 系统架构总览

```
┌──────────────────────────────────────────────────────────────┐
│                        Client Layer                           │
│                                                               │
│  ┌───────────────────┐    ┌───────────────────────────────┐  │
│  │  User Portal       │    │  Admin / Agent Portal          │  │
│  │  (Next.js 15)      │    │  (Next.js 15)                  │  │
│  │  Responsive        │    │  Desktop-first                 │  │
│  │  user.domain.com   │    │  admin.domain.com              │  │
│  └─────────┬─────────┘    └──────────────┬────────────────┘  │
│            │         Supabase Auth        │                   │
│            └──────────────┬───────────────┘                   │
└───────────────────────────┼───────────────────────────────────┘
                            │ HTTPS + JWT Bearer
┌───────────────────────────┼───────────────────────────────────┐
│                           │  Backend (.NET 10)                 │
│                           │                                    │
│  ┌────────────────────────┴──────────────────────────────┐   │
│  │  API Layer (Controllers)                               │   │
│  │  - REST endpoints + JWT validation                     │   │
│  │  - Request/Response DTO mapping                        │   │
│  │  - Rate Limiting + CORS                                │   │
│  ├────────────────────────────────────────────────────────┤   │
│  │  Application Layer (UseCases)                          │   │
│  │  - Business orchestration                              │   │
│  │  - DTO conversion                                      │   │
│  │  - Transaction coordination                            │   │
│  ├────────────────────────────────────────────────────────┤   │
│  │  Domain Layer (Entities + Rules)                       │   │
│  │  - Pure logic, zero external dependencies              │   │
│  │  - State machines + Business rules                     │   │
│  │  - Repository interfaces (abstractions only)           │   │
│  ├────────────────────────────────────────────────────────┤   │
│  │  Infrastructure Layer                                  │   │
│  │  - EF Core DbContext + Repositories                    │   │
│  │  - Vendor API adapter (MD5 auth, rate limiting)        │   │
│  │  - Background services (recovery, polling, reconcile,  │   │
│  │    bet record sync)                                    │   │
│  │  - SignalR hub implementation                          │   │
│  └────────────────────────────────────────────────────────┘   │
└───────────────────────────┬───────────────────────────────────┘
                            │ EF Core / Npgsql
┌───────────────────────────┼───────────────────────────────────┐
│                           │  Supabase                          │
│  ┌────────────────────────┴──────────────────────────────┐   │
│  │  PostgreSQL                                            │   │
│  │  - auth.users (Supabase managed)                       │   │
│  │  - public.* (EF Core managed via Migrations)           │   │
│  └───────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│  External: Game Vendor Aggregation API (70+ sub-platforms)     │
│  - Create Player / Transfer In / Transfer Out / TransferAll   │
│  - Get Game URL / Query Balance / Transfer Status              │
│  - Get Realtime/History Bet Records / Merchant Balance         │
│  - Auth: MD5(random+sn+secretKey) in Headers                  │
│  - 纯轮询模式（无回调/Webhook）                                  │
└───────────────────────────────────────────────────────────────┘
```

---

## 3. 后端分层架构

### 3.1 目录结构

```
Backend/
├── GamePlatform.sln
├── src/
│   ├── GamePlatform.API/                    # ASP.NET Core Web API
│   │   ├── Controllers/
│   │   │   ├── AuthController.cs
│   │   │   ├── WalletController.cs
│   │   │   ├── GameController.cs
│   │   │   ├── OrderController.cs
│   │   │   └── Admin/
│   │   │       ├── UserManagementController.cs
│   │   │       ├── WalletManagementController.cs
│   │   │       ├── OrderManagementController.cs
│   │   │       ├── DepositRequestController.cs
│   │   │       ├── WithdrawRequestController.cs
│   │   │       └── LogController.cs
│   │   ├── Controllers/Agent/
│   │   │   ├── AgentUserController.cs
│   │   │   └── AgentReportController.cs
│   │   ├── Controllers/Vendor/
│   │   │   └── VendorCallbackController.cs
│   │   ├── Middleware/
│   │   │   ├── ExceptionHandlingMiddleware.cs
│   │   │   └── RequestLoggingMiddleware.cs
│   │   ├── Filters/
│   │   │   └── IdempotencyFilter.cs
│   │   ├── Hubs/
│   │   │   └── NotificationHub.cs
│   │   └── Program.cs
│   │
│   ├── GamePlatform.Application/            # 业务编排层
│   │   ├── UseCases/
│   │   │   ├── Auth/
│   │   │   ├── Wallet/
│   │   │   ├── Game/
│   │   │   ├── Order/
│   │   │   ├── Admin/
│   │   │   └── Agent/
│   │   ├── DTOs/
│   │   │   ├── Request/
│   │   │   └── Response/
│   │   ├── Interfaces/
│   │   │   ├── IWalletService.cs
│   │   │   ├── IOrderService.cs
│   │   │   ├── IGameService.cs
│   │   │   ├── IVendorAdapter.cs
│   │   │   ├── INotificationService.cs
│   │   │   ├── IAlertService.cs
│   │   │   └── ITimeProvider.cs
│   │   └── Mappers/
│   │
│   ├── GamePlatform.Domain/                 # 纯领域层
│   │   ├── Entities/
│   │   │   ├── UserProfile.cs
│   │   │   ├── Wallet.cs
│   │   │   ├── WalletTransaction.cs
│   │   │   ├── GameOrder.cs
│   │   │   ├── VendorPlayerMapping.cs
│   │   │   ├── GameBetRecord.cs
│   │   │   ├── DepositRequest.cs
│   │   │   └── WithdrawRequest.cs
│   │   ├── Enums/
│   │   │   ├── UserRole.cs
│   │   │   ├── UserStatus.cs
│   │   │   ├── OrderStatus.cs
│   │   │   ├── TransactionType.cs
│   │   │   ├── DepositRequestStatus.cs
│   │   │   └── WithdrawRequestStatus.cs
│   │   ├── Rules/
│   │   │   ├── OrderStateMachine.cs
│   │   │   └── WalletRules.cs
│   │   ├── Exceptions/
│   │   │   ├── InsufficientBalanceException.cs
│   │   │   ├── InvalidStateTransitionException.cs
│   │   │   ├── ConcurrencyConflictException.cs
│   │   │   └── DomainException.cs
│   │   └── Interfaces/
│   │       ├── IWalletRepository.cs
│   │       ├── IOrderRepository.cs
│   │       ├── IUserProfileRepository.cs
│   │       ├── IVendorPlayerMappingRepository.cs
│   │       ├── IGameBetRecordRepository.cs
│   │       └── IUnitOfWork.cs
│   │
│   └── GamePlatform.Infrastructure/         # 基础设施层
│       ├── Persistence/
│       │   ├── AppDbContext.cs
│       │   ├── Configurations/
│       │   │   ├── UserProfileConfiguration.cs
│       │   │   ├── WalletConfiguration.cs
│       │   │   ├── WalletTransactionConfiguration.cs
│       │   │   ├── GameOrderConfiguration.cs
│       │   │   └── ...
│       │   ├── Migrations/
│       │   └── Repositories/
│       │       ├── WalletRepository.cs
│       │       ├── OrderRepository.cs
│       │       └── UserProfileRepository.cs
│       ├── Vendors/
│       │   ├── BaseVendorAdapter.cs
│       │   ├── ApiBetAdapter.cs
│       │   ├── VendorSignatureHelper.cs
│       │   └── VendorRateLimiter.cs
│       ├── BackgroundServices/
│       │   ├── PendingOrderRecoveryService.cs
│       │   ├── BalanceReconciliationService.cs
│       │   ├── OrderPollingService.cs
│       │   └── BetRecordSyncService.cs
│       └── Notifications/
│           └── SignalRNotificationService.cs
│
└── tests/
    ├── GamePlatform.Domain.Tests/            # Domain 层纯单元测试
    ├── GamePlatform.Application.Tests/       # Application 层单元测试
    └── GamePlatform.Integration.Tests/       # 集成测试（API + DB + 业务流程）
```

### 3.2 层间依赖规则

```
API ──→ Application ──→ Domain ←── Infrastructure
                                       │
                          implements Domain.Interfaces
```

- **Domain 层**：零外部依赖，只定义接口（IWalletRepository 等）
- **Application 层**：依赖 Domain，通过接口使用 Infrastructure
- **Infrastructure 层**：实现 Domain 接口，依赖 EF Core、Vendor SDK 等
- **API 层**：依赖 Application，仅负责 HTTP 协议转换

**禁止**：Controller 写业务逻辑 | 跨层直接访问数据库 | 直接修改余额字段

---

## 4. 前端项目结构

### 4.1 两个独立项目

```
Frontend/
├── user-portal/                      # 用户端（响应式布局）
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx              # 首页 / 游戏大厅
│   │   │   ├── (auth)/
│   │   │   │   ├── login/page.tsx
│   │   │   │   └── register/page.tsx
│   │   │   ├── wallet/
│   │   │   │   ├── page.tsx          # 余额 + 近期流水
│   │   │   │   ├── deposit/page.tsx
│   │   │   │   ├── withdraw/page.tsx
│   │   │   │   └── transactions/page.tsx
│   │   │   ├── games/
│   │   │   │   ├── page.tsx          # 游戏列表
│   │   │   │   └── [gameId]/
│   │   │   │       └── launch/page.tsx
│   │   │   ├── orders/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [orderId]/page.tsx
│   │   │   └── profile/page.tsx
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── lib/
│   │   │   ├── supabase/
│   │   │   ├── api/
│   │   │   └── signalr/
│   │   ├── stores/
│   │   └── types/
│   ├── next.config.ts
│   ├── tailwind.config.ts
│   └── package.json
│
└── admin-portal/                      # 管理端 + 代理端
    ├── src/
    │   ├── app/
    │   │   ├── layout.tsx
    │   │   ├── (auth)/login/page.tsx
    │   │   ├── admin/
    │   │   │   ├── layout.tsx
    │   │   │   ├── dashboard/page.tsx
    │   │   │   ├── users/
    │   │   │   ├── wallets/
    │   │   │   ├── orders/
    │   │   │   ├── deposit-requests/
    │   │   │   ├── withdraw-requests/
    │   │   │   ├── logs/
    │   │   │   ├── alerts/
    │   │   │   └── reconciliation/
    │   │   └── agent/
    │   │       ├── layout.tsx
    │   │       ├── dashboard/page.tsx
    │   │       ├── users/
    │   │       ├── orders/
    │   │       └── reports/
    │   ├── components/
    │   ├── hooks/
    │   ├── lib/
    │   └── stores/
    └── package.json
```

---

## 5. 认证授权流程

```
┌──────────┐    ┌──────────────┐    ┌───────────┐    ┌──────────────┐
│  Client   │───>│ Supabase     │───>│ JWT Token │───>│ .NET Backend │
│  (Next.js)│<───│ Auth         │<───│ (access + │    │ Validate JWT │
│           │    │              │    │  refresh)  │    │ + Role Check │
└──────────┘    └──────────────┘    └───────────┘    └──────────────┘
```

1. 前端通过 Supabase Auth SDK 登录（邮箱/密码）
2. 获取 JWT access token + refresh token
3. 前端请求后端 API 时携带 `Authorization: Bearer <token>`
4. 后端验证 JWT 签名（Supabase JWT Secret）
5. 从 JWT claims 中提取 userId
6. 从 UserProfile 表查询角色，进行授权

**角色模型**：
- UserProfile 表存储 Role 字段（User / Agent / Admin）
- Supabase Auth `user_metadata` 可存储角色快照
- 后端最终以数据库 UserProfile.Role 为权威来源

---

## 6. 通信模式

### 6.1 同步请求（HTTPS REST）

- 前端 → 后端：所有 CRUD 操作
- 后端 → 厂商 API：转入/转出/查询

### 6.2 异步推送（SignalR WebSocket）

- 后端 → 前端：
  - 余额变动通知
  - 订单状态更新
  - 充值/提现审核结果
  - 系统公告

### 6.3 回调（HTTPS POST）— 预留

> 当前接入的厂商 API（聚合 API）不提供回调/Webhook 机制。接口层预留回调处理能力，待未来对接支持回调的厂商时启用。

- 厂商 → 后端：
  - 游戏结果回调（预留）
  - 转账状态回调（预留）
  - 必须签名验证（预留）

### 6.4 后台轮询与定时任务（BackgroundService）

- **PendingOrderRecoveryService** — 恢复卡在 Pending 状态的订单（通过 transferStatus 查询确认）
- **OrderPollingService** — 轮询 Confirmed 状态订单（用户掉线/未主动退出场景，使用 QueryBalance + TransferAll）
- **BetRecordSyncService** — 每 2 分钟同步厂商实时投注记录到 GameBetRecords 表
- **BalanceReconciliationService** — 定时对账（Wallet.Balance vs SUM(WalletTransactions)）
- 所有定时任务必须幂等，重复执行不产生副作用

---

## 7. 数据库设计概要

### 7.1 Schema 分离

- `auth.*` — Supabase Auth 管理（不直接操作）
- `public.*` — 业务表（EF Core Migration 管理）

### 7.2 核心表

| 表 | 说明 | 关键约束 |
|----|------|----------|
| UserProfiles | 用户扩展信息 | FK → auth.users, Role 字段 |
| Wallets | 用户钱包 | 一用户一钱包, Version CAS, Balance >= 0, FrozenBalance（提现冻结） |
| WalletTransactions | 钱包流水 | 不可删除, FK → Wallets |
| GameOrders | 游戏订单 | InternalOrderId UNIQUE(VARCHAR 32), ExternalOrderId UNIQUE, PlatType |
| VendorPlayerMappings | 玩家映射 | UNIQUE(UserId,VendorCode), UNIQUE(VendorCode,VendorPlayerId) |
| GameBetRecords | 投注记录 | VendorBetOrderId UNIQUE, FK → UserProfiles |
| DepositRequests | 充值申请 | FK → UserProfiles |
| WithdrawRequests | 提现申请 | FK → UserProfiles |
| VendorCallbackLogs | 回调日志 | 不可删除, 存储原始 payload |
| VendorQueryLogs | 轮询日志 | 不可删除 |
| AdminActionLogs | 操作日志 | 不可删除, Level 字段 |

详细表结构 → 参见 `03-详细设计-detailed_design.md`

### 7.3 迁移规范

- 所有结构变更通过 EF Core Migration 管理
- 禁止手动修改生产数据库结构
- Migration 必须可回滚（Down 方法）
- 详细规范 → 参见 `03-详细设计-detailed_design.md` 第 13 节

---

## 8. 安全架构

### 8.1 传输安全

- 所有通信强制 HTTPS
- JWT Token 有效期：access 15min, refresh 7d
- SignalR 连接需 JWT 认证

### 8.2 API 安全

- JWT 验证 + 角色授权（`[Authorize(Roles = "Admin")]`）
- Rate Limiting（per-endpoint 配置）
- CORS 仅允许已知域名
- 请求体大小限制

### 8.3 厂商 API 通信安全

**出站请求签名（当前使用）**：
- MD5 签名：`sign = MD5(random + sn + secretKey).ToLower()`
- 请求头：`random`（16-32 位随机字符）+ `sn`（商户前缀）+ `sign`（MD5 签名）
- 出站 IP 需加入厂商白名单（厂商端配置）

**回调安全（预留，当前厂商不使用）**：
- HMAC 签名验证（预留）
- 时间戳校验（预留）
- IP 白名单（预留）
- 原始 payload 全量存档到 VendorCallbackLogs

### 8.4 数据安全

- 密钥通过环境变量 / Secret Manager 管理
- 日志脱敏（不记录完整卡号、密码等）
- 生产/测试环境完全隔离（不同数据库、不同密钥）
- 回调签名密钥分厂商管理

### 8.5 安全扫描

- CI 流水线中集成 `dotnet list package --vulnerable`（依赖漏洞检测）
- CI 流水线中集成 `npm audit`（前端依赖漏洞检测）
- 发现高危漏洞（High/Critical）时阻断 PR 合并

### 8.6 健康检查

- `GET /api/health` — 检查数据库连接、Supabase Auth 可达性、SignalR Hub 状态
- 部署后自动调用，失败时阻止流量切换
- 详细设计 → 参见 `03-详细设计-detailed_design.md` §17

### 8.7 业务安全

- 幂等控制（唯一索引 + 状态检查）
- CAS 乐观锁（防止并发余额错误）
- 事务完整性（原子操作，无部分成功）
- 审计日志（所有变更可追溯）

---

## 9. 部署架构

### 9.1 MVP 部署方案

```
┌────────────────────────────────────────────────┐
│              Hosting (VPS / Cloud)               │
│                                                  │
│  ┌──────────────┐    ┌────────────────────────┐ │
│  │ User Portal   │    │ Admin/Agent Portal      │ │
│  │ (Next.js)     │    │ (Next.js)               │ │
│  │ :3000         │    │ :3001                   │ │
│  └──────────────┘    └────────────────────────┘ │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │ .NET Backend API                          │   │
│  │ :5000 (HTTP) / :5001 (HTTPS)              │   │
│  │ + SignalR Hub (/hubs/notification)         │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │ Nginx (Reverse Proxy + TLS)               │   │
│  │ :443                                      │   │
│  │ user.domain.com   → localhost:3000        │   │
│  │ admin.domain.com  → localhost:3001        │   │
│  │ api.domain.com    → localhost:5000        │   │
│  └──────────────────────────────────────────┘   │
└────────────────────────────────────────────────┘
                    │
                    │ Internet
                    │
┌───────────────────┴──────────────────────────────┐
│  Supabase Cloud                                    │
│  - PostgreSQL (managed)                            │
│  - Auth (managed)                                  │
└──────────────────────────────────────────────────┘
```

### 9.2 环境隔离

| 环境 | 用途 | 数据库 | 密钥 |
|------|------|--------|------|
| Development | 本地开发 | Supabase 开发项目 | 开发密钥 |
| Staging | 测试验证 | Supabase Staging 项目 | 测试密钥 |
| Production | 生产 | Supabase Production 项目 | 生产密钥 |

---

## 10. 日志架构

### 10.1 结构化日志（Serilog）

- 使用 **Serilog** 作为日志框架，输出 JSON 结构化日志
- 日志级别：`Debug` / `Information` / `Warning` / `Error` / `Fatal`
- **Console Sink**：开发环境，人类可读格式
- **File Sink**：生产环境，JSON 格式，按日滚动
- 资金相关操作（Credit/Debit/CAS 更新）必须记录 `Information` 级别日志
- 异常和告警记录 `Error` 级别日志
- **日志脱敏**：不记录完整卡号、密码、JWT Token 等敏感信息

### 10.2 日志上下文

每条日志自动包含：
- `UserId` — 当前请求用户
- `RequestId` — 请求跟踪 ID（`HttpContext.TraceIdentifier`）
- `Endpoint` — 请求路径
- `Timestamp` — 本地时间

---

## 11. API 版本策略

MVP 阶段所有端点不加版本前缀（`/api/...`）。

**后续版本规划**：
- 采用 URL Path Versioning：`/api/v1/...`、`/api/v2/...`
- 使用 `Asp.Versioning.Http` NuGet 包
- 新版本发布后，旧版本保留至少一个版本周期
- MVP 阶段不需要实现，但代码结构预留 versioning 扩展点

---

## 12. 约束引用

- 模块定义和优先级 → 参见 `01-概要功能设计-overview.md`
- 详细接口和数据模型 → 参见 `03-详细设计-detailed_design.md`
- 页面结构和交互 → 参见 `04-UI-UX设计-uiux_design.md`
- 测试策略 → 参见 `05-测试设计-testing_design.md`
- CI/CD 流程 → 参见 `06-CI-CD设计-ci_cd_design.md`
