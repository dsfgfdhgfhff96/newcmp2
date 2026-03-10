# 游戏聚合平台 — MVP 实施计划

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.
>
> **Git 工作流（强制）：**
> - 每个阶段在独立分支开发（命名：`phase-N-中文描述`）
> - 每个任务完成后立即 commit（中文 commit message）
> - 阶段全部任务完成后：运行所有测试确认通过 → 创建 PR（标题：`feat: 阶段N - 描述`）→ **暂停并等待 review 确认后再继续下一阶段**
> - PR 合并确认后：`git pull master` → 创建下一阶段分支

**目标：** 构建 MVP 游戏聚合平台，采用 Transfer Wallet 模型、中心钱包 CAS 并发控制、单厂商接入，三端（用户/代理/管理）。

**架构：** .NET 10 清洁架构（API/Application/Domain/Infrastructure），EF Core 连接 Supabase PostgreSQL。两个 Next.js 15 应用：user-portal（响应式）和 admin-portal（管理+代理）。Supabase Auth 认证，SignalR 实时推送。

**技术栈：** .NET 10, ASP.NET Core, EF Core, Npgsql, PostgreSQL (Supabase), Next.js 15, React 19, TypeScript 5, Tailwind CSS 4, shadcn/ui, xUnit, Moq, FluentAssertions, Playwright

**设计文档：** 详见 `docs/design/`：
- `01-概要功能设计-overview.md` — 模块列表、优先级、风险点
- `02-技术架构设计-architecture.md` — 分层架构、目录结构、安全设计
- `03-详细设计-detailed_design.md` — 实体、接口、状态机、数据库 DDL
- `04-UI-UX设计-uiux_design.md` — 页面结构、交互流程
- `05-测试设计-testing_design.md` — 测试场景、覆盖率目标
- `06-CI-CD设计-ci_cd_design.md` — 流水线、质量门禁

---

## 阶段 1：项目脚手架 + Domain 层

### 任务 1：创建后端解决方案结构

**文件：**
- 创建：`Backend/GamePlatform.sln`
- 创建：`Backend/src/GamePlatform.API/GamePlatform.API.csproj`
- 创建：`Backend/src/GamePlatform.Application/GamePlatform.Application.csproj`
- 创建：`Backend/src/GamePlatform.Domain/GamePlatform.Domain.csproj`
- 创建：`Backend/src/GamePlatform.Infrastructure/GamePlatform.Infrastructure.csproj`
- 创建：`Backend/tests/GamePlatform.Domain.Tests/GamePlatform.Domain.Tests.csproj`
- 创建：`Backend/tests/GamePlatform.Application.Tests/GamePlatform.Application.Tests.csproj`
- 创建：`Backend/tests/GamePlatform.Integration.Tests/GamePlatform.Integration.Tests.csproj`
- 创建：`Backend/.editorconfig`（代码格式规范，CI `dotnet format --verify-no-changes` 依赖此文件）

**步骤 1：** 创建 .NET 10 解决方案，含 4 个源码项目 + 3 个测试项目（Domain.Tests, Application.Tests, Integration.Tests）
**步骤 2：** 配置项目引用：
  - API → Application
  - Application → Domain
  - Infrastructure → Domain
  - API → Infrastructure（用于 DI 注册）
  - Domain.Tests → Domain
  - Application.Tests → Application
  - Integration.Tests → API
**步骤 3：** 添加 NuGet 包（EF Core, Npgsql, xUnit, Moq, FluentAssertions, Coverlet）
**步骤 4：** 创建 `.editorconfig` — 配置 C# 代码风格规范（缩进、命名规则、using 排序等），确保 `dotnet format` 可用（参见 `06-CI-CD设计-ci_cd_design.md` §2.1 步骤 5）
**步骤 5：** 运行 `dotnet build` — 验证编译通过
**步骤 6：** 运行 `dotnet format --verify-no-changes` — 验证格式规范生效
**步骤 7：** 提交：`chore: scaffold backend solution with clean architecture and editorconfig`

### 任务 2：创建前端项目

**文件：**
- 创建：`Frontend/user-portal/`（Next.js 15 应用）
- 创建：`Frontend/admin-portal/`（Next.js 15 应用）
- 创建：`Frontend/user-portal/vitest.config.ts`（Vitest 配置）
- 创建：`Frontend/admin-portal/vitest.config.ts`（Vitest 配置）
- 创建：`Frontend/user-portal/src/test/setup.ts`（测试环境初始化）
- 创建：`Frontend/admin-portal/src/test/setup.ts`（测试环境初始化）
- 创建：`Frontend/user-portal/src/test/mocks/handlers.ts`（MSW 请求拦截）
- 创建：`Frontend/admin-portal/src/test/mocks/handlers.ts`（MSW 请求拦截）

**步骤 1：** 创建 user-portal：`npx create-next-app@latest user-portal --typescript --tailwind --app --src-dir`
**步骤 2：** 创建 admin-portal：同上命令
**步骤 3：** 安装共享依赖：`@supabase/ssr`, `@microsoft/signalr`, `zustand`, `zod`, `react-hook-form`
**步骤 4：** 安装测试依赖：`vitest`, `@testing-library/react`, `@testing-library/jest-dom`, `@testing-library/user-event`, `jsdom`, `msw`
**步骤 5：** 在两个项目中安装 shadcn/ui
**步骤 6：** 配置 Vitest — 创建 `vitest.config.ts`（environment: jsdom, setupFiles, coverage 阈值 80%）、`src/test/setup.ts`（@testing-library/jest-dom 导入）、`src/test/mocks/handlers.ts`（MSW 基础配置）（参见 `05-测试设计-testing_design.md` §8）
**步骤 7：** 在两个项目的 `package.json` 中添加 `"test": "vitest run"` 和 `"test:coverage": "vitest run --coverage"` 脚本（CI 依赖 `npm run test`，参见 `06-CI-CD设计-ci_cd_design.md` §2.2-§2.3）
**步骤 8：** 运行 `npm run build` — 验证编译通过
**步骤 9：** 运行 `npm run test` — 验证测试框架可用（空运行通过）
**步骤 10：** 提交：`chore: scaffold frontend projects with Vitest, MSW, and testing infrastructure`

### 任务 3：Domain — Wallet + WalletTransaction + WalletRules

**文件：**
- 测试：`Backend/tests/GamePlatform.Domain.Tests/Entities/WalletTests.cs`
- 测试：`Backend/tests/GamePlatform.Domain.Tests/Rules/WalletRulesTests.cs`
- 创建：`Backend/src/GamePlatform.Domain/Entities/Wallet.cs`
- 创建：`Backend/src/GamePlatform.Domain/Entities/WalletTransaction.cs`
- 创建：`Backend/src/GamePlatform.Domain/Enums/TransactionType.cs`
- 创建：`Backend/src/GamePlatform.Domain/Rules/WalletRules.cs`
- 创建：`Backend/src/GamePlatform.Domain/Exceptions/InsufficientBalanceException.cs`
- 创建：`Backend/src/GamePlatform.Domain/Exceptions/DomainException.cs`

**步骤 1：** 编写失败测试（参见 `05-测试设计-testing_design.md` §2.1 + §2.7 所有场景）
  - 包含 Freeze/Unfreeze/ConfirmFrozenDebit 和 AvailableBalance 计算
  - 包含金额限额测试（AmountLimitOptions 配置验证）
  - Credit/Debit 方法接受 `DateTime now` 参数（Domain 层无外部时间依赖）
**步骤 2：** 运行 `dotnet test` — 验证测试失败（RED）
**步骤 3：** 实现 Wallet 实体（Credit/Debit/Freeze/Unfreeze/ConfirmFrozenDebit）、WalletRules、WalletTransaction、AmountLimitOptions、异常类
**步骤 4：** 运行 `dotnet test` — 验证测试通过（GREEN）
**步骤 5：** 按需重构
**步骤 6：** 提交：`feat: add Wallet domain entity with CAS versioning, frozen balance, amount limits, and WalletRules`

### 任务 4：Domain — GameOrder + OrderStateMachine

**文件：**
- 测试：`Backend/tests/GamePlatform.Domain.Tests/Entities/GameOrderTests.cs`
- 测试：`Backend/tests/GamePlatform.Domain.Tests/Rules/OrderStateMachineTests.cs`
- 创建：`Backend/src/GamePlatform.Domain/Entities/GameOrder.cs`
- 创建：`Backend/src/GamePlatform.Domain/Enums/OrderStatus.cs`
- 创建：`Backend/src/GamePlatform.Domain/Rules/OrderStateMachine.cs`
- 创建：`Backend/src/GamePlatform.Domain/Exceptions/InvalidStateTransitionException.cs`

**步骤 1：** 编写所有状态流转的失败测试（参见 `05-测试设计-testing_design.md` §2.2 + §2.3）
  - GameOrder 包含 CancelReason 字段（与 ErrorMessage 分离）
  - 所有状态流转方法接受 `DateTime now` 参数
**步骤 2：** 运行测试 — 验证失败（RED）
**步骤 3：** 实现 GameOrder 实体和状态流转、OrderStateMachine
**步骤 4：** 运行测试 — 验证通过（GREEN）
**步骤 5：** 提交：`feat: add GameOrder entity with state machine validation`

### 任务 5：Domain — UserProfile + DepositRequest + WithdrawRequest + VendorPlayerMapping

**文件：**
- 测试：`Backend/tests/GamePlatform.Domain.Tests/Entities/UserProfileTests.cs`
- 测试：`Backend/tests/GamePlatform.Domain.Tests/Entities/DepositRequestTests.cs`
- 测试：`Backend/tests/GamePlatform.Domain.Tests/Entities/WithdrawRequestTests.cs`
- 测试：`Backend/tests/GamePlatform.Domain.Tests/Entities/VendorPlayerMappingTests.cs`
- 测试：`Backend/tests/GamePlatform.Domain.Tests/Entities/UserBankCardTests.cs`
- 创建：`Backend/src/GamePlatform.Domain/Entities/UserProfile.cs`
- 创建：`Backend/src/GamePlatform.Domain/Entities/DepositRequest.cs`
- 创建：`Backend/src/GamePlatform.Domain/Entities/WithdrawRequest.cs`
- 创建：`Backend/src/GamePlatform.Domain/Entities/VendorPlayerMapping.cs`
- 创建：`Backend/src/GamePlatform.Domain/Entities/GameBetRecord.cs`
- 创建：`Backend/src/GamePlatform.Domain/Entities/UserBankCard.cs`
- 创建：`Backend/src/GamePlatform.Domain/Enums/UserRole.cs`
- 创建：`Backend/src/GamePlatform.Domain/Enums/UserStatus.cs`
- 创建：`Backend/src/GamePlatform.Domain/Enums/RequestStatus.cs`

**步骤 1：** 编写失败测试（参见 `05-测试设计-testing_design.md` §2.4 VendorPlayerMapping + §2.5 UserProfile + §2.6 DepositRequest/WithdrawRequest + §2.7 UserBankCard + §2.8 StateMachine）
  - VendorPlayerMapping：Create（格式校验 5-11 位小写+数字）、MarkCreated、唯一性
  - UserBankCard：Create（卡号 16-19 位校验、必填字段）、SetDefault、最多 5 张限制
  - 包含 DepositRequestStateMachine 和 WithdrawRequestStateMachine 测试
  - WithdrawRequest 状态流转需验证冻结/解冻时序
**步骤 2：** 运行测试 — 验证失败（RED）
**步骤 3：** 实现实体 + VendorPlayerMapping + GameBetRecord + UserBankCard + StateMachines
**步骤 4：** 运行测试 — 验证通过（GREEN）
**步骤 5：** 提交：`feat: add UserProfile, DepositRequest, WithdrawRequest, VendorPlayerMapping, UserBankCard domain entities`

### 任务 6：Domain — Repository 接口 + ITimeProvider

**文件：**
- 创建：`Backend/src/GamePlatform.Domain/Interfaces/IWalletRepository.cs`
- 创建：`Backend/src/GamePlatform.Domain/Interfaces/IOrderRepository.cs`
- 创建：`Backend/src/GamePlatform.Domain/Interfaces/IUserProfileRepository.cs`
- 创建：`Backend/src/GamePlatform.Domain/Interfaces/IDepositRequestRepository.cs`
- 创建：`Backend/src/GamePlatform.Domain/Interfaces/IWithdrawRequestRepository.cs`
- 创建：`Backend/src/GamePlatform.Domain/Interfaces/IVendorPlayerMappingRepository.cs`
- 创建：`Backend/src/GamePlatform.Domain/Interfaces/IGameBetRecordRepository.cs`
- 创建：`Backend/src/GamePlatform.Domain/Interfaces/IUserBankCardRepository.cs`
- 创建：`Backend/src/GamePlatform.Domain/Interfaces/IUnitOfWork.cs`
- 创建：`Backend/src/GamePlatform.Application/Interfaces/IVendorCallbackLogRepository.cs`
- 创建：`Backend/src/GamePlatform.Application/Interfaces/IVendorQueryLogRepository.cs`
- 创建：`Backend/src/GamePlatform.Application/Interfaces/IAdminActionLogRepository.cs`
- 创建：`Backend/src/GamePlatform.Application/Interfaces/ITimeProvider.cs`

**步骤 1：** 按 `03-详细设计-detailed_design.md` §1.15 定义 Domain 接口
  - IOrderRepository 包含 HasActiveOrderAsync（单会话检查）和 GetConfirmedOrdersAsync
  - IUserProfileRepository 包含 GetByAgentIdAsync
  - IVendorPlayerMappingRepository 包含 GetByUserAndVendorAsync、UpsertAsync
  - IGameBetRecordRepository 包含 UpsertBatchAsync（幂等批量写入）
  - IUserBankCardRepository 包含 GetByUserIdAsync、CountByUserIdAsync
**步骤 2：** 定义日志 Repository 接口（VendorCallbackLog, VendorQueryLog, AdminActionLog）
**步骤 3：** 定义 ITimeProvider 接口 + SystemTimeProvider 默认实现
**步骤 4：** 运行 `dotnet build` — 验证编译通过
**步骤 5：** 提交：`feat: add domain repository interfaces, log repositories, and ITimeProvider`

---

## 阶段 2：Infrastructure + 数据库

### 任务 7：EF Core DbContext + 实体配置

**文件：**
- 创建：`Backend/src/GamePlatform.Infrastructure/Persistence/AppDbContext.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/Persistence/Configurations/`（每个实体一个配置文件）

**步骤 1：** 实现 AppDbContext，包含所有 DbSet<T> 属性
**步骤 2：** 配置每个实体（表名、列类型、索引、约束）
  - 参见 `03-详细设计-detailed_design.md` §5 的完整 DDL
  - 所有金额字段使用 `DECIMAL(18,2)`
  - InternalOrderId (VARCHAR 32) 和 ExternalOrderId 配置 UNIQUE 索引
  - 配置 CHECK 约束：Balance >= 0, FrozenBalance >= 0, FrozenBalance <= Balance
  - Wallets 表包含 FrozenBalance 字段
  - GameOrders 表包含 CancelReason、PlatType 字段
  - VendorPlayerMappings 表：UNIQUE(UserId,VendorCode), UNIQUE(VendorCode,VendorPlayerId)
  - GameBetRecords 表：VendorBetOrderId UNIQUE INDEX
  - UserBankCards 表：UNIQUE(UserId,CardNumber)
  - AdminActionLogs.AdminId 为可选外键（NULL = 系统自动操作）
**步骤 3：** 运行 `dotnet build` — 验证编译通过
**步骤 4：** 提交：`feat: add EF Core DbContext with entity configurations`

### 任务 8：初始数据库迁移

**文件：**
- 创建：`Backend/src/GamePlatform.Infrastructure/Persistence/Migrations/`（自动生成）

**步骤 1：** 配置连接字符串（开发环境 Supabase）
**步骤 2：** 运行 `dotnet ef migrations add InitialCreate`
**步骤 3：** 审查生成的迁移 — 验证与 `03-详细设计-detailed_design.md` DDL 一致
**步骤 4：** 运行 `dotnet ef database update` — 验证表创建成功
**步骤 5：** 提交：`feat: add initial database migration`

### 任务 9：Repository 实现

**文件：**
- 测试：`Backend/tests/GamePlatform.Integration.Tests/Repositories/WalletRepositoryTests.cs`
- 测试：`Backend/tests/GamePlatform.Integration.Tests/Repositories/OrderRepositoryTests.cs`
- 测试：`Backend/tests/GamePlatform.Integration.Tests/Repositories/VendorPlayerMappingRepositoryTests.cs`
- 测试：`Backend/tests/GamePlatform.Integration.Tests/Repositories/GameBetRecordRepositoryTests.cs`
- 创建：`Backend/tests/GamePlatform.Integration.Tests/Fixtures/DatabaseFixture.cs`（共享测试数据库）
- 创建：`Backend/src/GamePlatform.Infrastructure/Persistence/Repositories/WalletRepository.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/Persistence/Repositories/OrderRepository.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/Persistence/Repositories/UserProfileRepository.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/Persistence/Repositories/VendorPlayerMappingRepository.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/Persistence/Repositories/GameBetRecordRepository.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/Persistence/Repositories/UserBankCardRepository.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/Persistence/Repositories/UnitOfWork.cs`

**步骤 1：** 创建 `DatabaseFixture` — 集成测试共享的测试数据库上下文（使用独立测试数据库或事务回滚隔离）
**步骤 2：** 编写 WalletRepository 集成测试（RED）
  - CAS 更新成功（版本号匹配）
  - CAS 更新失败（版本号冲突，返回 0 行）
  - 并发 CAS 更新（多线程竞争，仅 1 个成功）
  - GetByUserIdAsync 正常查询
**步骤 3：** 编写 OrderRepository 集成测试（RED）
  - HasActiveOrderAsync（有/无活跃订单）
  - GetConfirmedOrdersAsync（筛选 Confirmed 状态）
  - InternalOrderId 唯一约束违反
**步骤 4：** 编写 VendorPlayerMappingRepository 集成测试（RED）
  - UpsertAsync（新建 + 更新）
  - 唯一约束 (UserId,VendorCode) 和 (VendorCode,VendorPlayerId)
**步骤 5：** 编写 GameBetRecordRepository 集成测试（RED）
  - UpsertBatchAsync 幂等性（重复写入不报错，数据不重复）
**步骤 6：** 运行 `dotnet test` — 验证测试失败（RED）
**步骤 7：** 实现 WalletRepository（重点：UpdateWithCasAsync — 参见 `03-详细设计-detailed_design.md` §8）
**步骤 8：** 实现 OrderRepository
**步骤 9：** 实现 UserProfileRepository
**步骤 10：** 实现 VendorPlayerMappingRepository（GetByUserAndVendorAsync, UpsertAsync）
**步骤 11：** 实现 GameBetRecordRepository（UpsertBatchAsync — 幂等批量写入）
**步骤 12：** 实现 UnitOfWork
**步骤 13：** 运行 `dotnet test` — 验证所有集成测试通过（GREEN）
**步骤 14：** 运行 `dotnet format --verify-no-changes` — 验证代码格式
**步骤 15：** 提交：`feat: add repository implementations with CAS wallet update and integration tests`

---

## 阶段 3：Application 层 — 钱包

### 任务 10：WalletService + Application 接口

**文件：**
- 测试：`Backend/tests/GamePlatform.Application.Tests/Wallet/WalletServiceTests.cs`
- 创建：`Backend/src/GamePlatform.Application/Interfaces/IWalletService.cs`
- 创建：`Backend/src/GamePlatform.Application/UseCases/Wallet/WalletService.cs`
- 创建：`Backend/src/GamePlatform.Application/DTOs/`（相关 DTO）

**步骤 1：** 编写 WalletService 失败测试（参见 `05-测试设计-testing_design.md` §3.1）
  - AdminDeposit、AdminWithdraw、CAS 冲突、余额校验、对账
**步骤 2：** 运行测试 — 验证失败（RED）
**步骤 3：** 实现 WalletService 的所有用例
**步骤 4：** 运行测试 — 验证通过（GREEN）
**步骤 5：** 提交：`feat: add WalletService with admin deposit/withdraw and reconciliation`

### 任务 11：充值/提现申请处理

**文件：**
- 测试：`Backend/tests/GamePlatform.Application.Tests/Wallet/DepositRequestServiceTests.cs`
- 测试：`Backend/tests/GamePlatform.Application.Tests/Wallet/WithdrawRequestServiceTests.cs`
- 修改：`Backend/src/GamePlatform.Application/UseCases/Wallet/WalletService.cs`

**步骤 1：** 编写 RequestDeposit、ApproveDeposit、RejectDeposit 失败测试（包含金额限额校验测试）
**步骤 2：** 编写 RequestWithdrawal（含冻结 + 金额限额 + 每日累计限额 + 银行卡选择）、ApproveWithdrawal（ConfirmFrozenDebit）、RejectWithdrawal（Unfreeze）失败测试
**步骤 3：** 运行测试 — 验证失败（RED）
**步骤 4：** 实现 WalletService 中的申请处理（提现流程：申请→冻结→审批→扣款或解冻）
**步骤 5：** 运行测试 — 验证通过（GREEN）
**步骤 6：** 提交：`feat: add deposit/withdraw request approval flow with balance freezing`

---

## 阶段 4：Application 层 — 订单 + 厂商

### 任务 12：IVendorAdapter + Factory + ApiBet 实现

**文件：**
- 测试：`Backend/tests/GamePlatform.Application.Tests/Vendors/VendorSignatureHelperTests.cs`
- 测试：`Backend/tests/GamePlatform.Application.Tests/Vendors/ApiBetAdapterTests.cs`
- 测试：`Backend/tests/GamePlatform.Application.Tests/Vendors/VendorRateLimiterTests.cs`
- 创建：`Backend/src/GamePlatform.Application/Interfaces/IVendorAdapter.cs`
- 创建：`Backend/src/GamePlatform.Application/Interfaces/IVendorAdapterFactory.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/Vendors/VendorAdapterFactory.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/Vendors/ApiBetAdapter.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/Vendors/VendorSignatureHelper.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/Vendors/VendorRateLimiter.cs`
- 创建 DTO：`VendorResult`, `VendorBalanceResult`, `VendorBalanceAllResult`, `TransferResult`, `TransferAllResult`, `TransferStatusResult`, `GameLaunchResult`, `GameBetRecordData`, `PagedVendorResult<T>`, `MerchantBalanceResult`, `VendorConfig`

**步骤 1：** 定义 IVendorAdapter 接口（参见 `03-详细设计-detailed_design.md` §6）
  - 包含：CreatePlayerAsync, QueryBalanceAsync, QueryBalanceAllAsync, TransferInAsync, TransferOutAsync, TransferAllAsync, QueryTransferStatusAsync, GetGameUrlAsync, GetDemoUrlAsync, GetGameListAsync, GetRealtimeRecordsAsync, GetHistoryRecordsAsync, QueryMerchantBalanceAsync
  - TransferResult 包含 NeedsStatusCheck 属性（非 10000/10005 时为 true）
  - 预留：VerifyCallbackSignature, ParseCallback
**步骤 2：** 定义 IVendorAdapterFactory 接口（参见 `03-详细设计-detailed_design.md` §6.1）
**步骤 3：** 编写 VendorSignatureHelper 失败测试（RED）（参见 `05-测试设计-testing_design.md` §3.5）
  - 正确签名生成验证（已知输入→已知输出）
  - 空/null 参数处理
**步骤 4：** 编写 ApiBetAdapter 失败测试（RED）（使用 MockHttpMessageHandler 模拟 HTTP）
  - 正常响应解析（status=10000）
  - 错误码处理（10001 参数错误、10002 玩家已存在、10003 玩家不存在、10013 订单不存在等，参见 §21）
  - 网络超时/异常处理
  - NeedsStatusCheck 属性在非 10000/10005 时为 true
**步骤 5：** 编写 VendorRateLimiter 失败测试（RED）
  - 限流阈值内请求通过
  - 超过阈值拒绝
  - 过期后恢复
**步骤 6：** 运行 `dotnet test` — 验证测试失败（RED）
**步骤 7：** 实现 VendorSignatureHelper — MD5(random+sn+secretKey) 签名计算（参见 §20）
**步骤 8：** 实现 VendorRateLimiter — SemaphoreSlim + MemoryCache 客户端限流（参见 §22）
**步骤 9：** 实现 ApiBetAdapter（完整厂商 API 对接，含 21 个错误码处理，参见 §21）
**步骤 10：** 实现 VendorAdapterFactory（字典解析器，根据 vendorCode 获取适配器）
**步骤 11：** 运行 `dotnet test` — 验证所有测试通过（GREEN）
**步骤 12：** 运行 `dotnet format --verify-no-changes` — 验证代码格式
**步骤 13：** 提交：`feat: add IVendorAdapter, ApiBetAdapter with MD5 auth, rate limiting, error code handling`

### 任务 13：OrderService — LaunchGame + ExitGame

**文件：**
- 测试：`Backend/tests/GamePlatform.Application.Tests/Order/OrderServiceTests.cs`
- 创建：`Backend/src/GamePlatform.Application/Interfaces/IOrderService.cs`
- 创建：`Backend/src/GamePlatform.Application/UseCases/Order/OrderService.cs`

**步骤 1：** 编写 LaunchGame 场景失败测试（参见 `05-测试设计-testing_design.md` §3.2）
  - 正常流程：玩家映射 + 创建订单 + Debit + TransferIn + Confirm
  - **玩家首次使用**：自动创建 VendorPlayerMapping + CreatePlayerAsync
  - **CreatePlayer 返回 10002（已存在）**：标记 IsCreated=true，继续流程
  - **转入返回非 10000/10005**：调用 transferStatus 轮询确认
  - **单会话限制**：已有 Pending/Confirmed 订单时拒绝
  - **金额限额**：低于最小游戏转入金额时拒绝
  - 注意事务边界：事务 1（扣款+创建订单）→ 厂商 API → 事务 2（更新状态）
  - InternalOrderId 格式：32 位字母数字（GUID 去掉连字符）
**步骤 2：** 运行测试 — 验证失败（RED）
**步骤 3：** 实现 LaunchGameAsync（参见 `03-详细设计-detailed_design.md` §4.1 含玩家映射和 transferStatus）
**步骤 4：** 运行测试 — 验证通过（GREEN）
**步骤 5：** 编写 ExitGame 场景失败测试（一键回收 TransferAllAsync）
  - 正常流程：TransferAll → Credit → Settle
  - 回收金额为 0：Cancel（非 Settle）
  - 一键回收失败：保持 Confirmed
**步骤 6：** 实现 ExitGameAsync（参见 `03-详细设计-detailed_design.md` §4.2 一键回收流程）
**步骤 7：** 运行测试 — 验证通过
**步骤 8：** 编写 AdminForceExitGameAsync 失败测试（一键回收 + AdminActionLog）
**步骤 9：** 实现 AdminForceExitGameAsync（参见 `03-详细设计-detailed_design.md` §18）
**步骤 10：** 运行测试 — 验证通过
**步骤 11：** 提交：`feat: add OrderService with player mapping, transferStatus, one-click recovery`

### 任务 14：OrderService — 轮询 + 回调（预留）+ GameService

**文件：**
- 测试：`Backend/tests/GamePlatform.Application.Tests/Order/PollPendingOrdersTests.cs`
- 测试：`Backend/tests/GamePlatform.Application.Tests/Game/GameServiceTests.cs`
- 修改：`Backend/src/GamePlatform.Application/UseCases/Order/OrderService.cs`
- 创建：`Backend/src/GamePlatform.Application/UseCases/Game/GameService.cs`

**步骤 1：** 编写 PollPendingOrders 失败测试（RED）（参见 `05-测试设计-testing_design.md` §3.2 轮询部分）
  - transferStatus 返回 success → Confirm
  - transferStatus 返回 failed → Fail + 退款
  - transferStatus 返回 pending → 等待下次轮询
  - 订单不存在（10013）→ Fail + 退款
  - 轮询超过 MaxRetryCount → 告警
**步骤 2：** 运行 `dotnet test` — 验证测试失败（RED）
**步骤 3：** 实现 PollPendingOrdersAsync（基于 transferStatus 的恢复机制）
**步骤 4：** 运行 `dotnet test` — 验证 PollPendingOrders 测试通过（GREEN）
**步骤 5：** 编写 HandleCallbackAsync 预留实现（标记为当前厂商不使用，参见 §4.3）
**步骤 6：** 编写 GameService 失败测试（RED）（参见 `05-测试设计-testing_design.md` §3.3）
  - GetGameList（缓存、分平台）、GetSupportedPlatTypes、GetDemoUrl
  - GetBetRecords、GetAllBetRecords（分页）
**步骤 7：** 运行 `dotnet test` — 验证测试失败（RED）
**步骤 8：** 实现 GameService
**步骤 9：** 运行 `dotnet test` — 验证所有测试通过（GREEN）
**步骤 10：** 运行 `dotnet format --verify-no-changes` — 验证代码格式
**步骤 11：** 提交：`feat: add pending order polling, callback stub, and GameService`

---

## 阶段 5：API 层

### 任务 15：JWT 认证 + 中间件配置

**文件：**
- 测试：`Backend/tests/GamePlatform.Application.Tests/Middleware/ExceptionHandlingMiddlewareTests.cs`
- 修改：`Backend/src/GamePlatform.API/Program.cs`
- 创建：`Backend/src/GamePlatform.API/Middleware/ExceptionHandlingMiddleware.cs`

**步骤 1：** 编写 ExceptionHandlingMiddleware 失败测试（RED）
  - DomainException → 400 + ApiErrorResponse（ErrorCode 映射）
  - InsufficientBalanceException → 400 + INSUFFICIENT_BALANCE
  - InvalidStateTransitionException → 400 + INVALID_STATE_TRANSITION
  - 未处理异常 → 500 + 通用错误（不泄露内部信息）
**步骤 2：** 运行 `dotnet test` — 验证测试失败（RED）
**步骤 3：** 实现全局异常处理中间件（统一 ApiResponse/ApiErrorResponse 格式，参见 `03-详细设计-detailed_design.md` §3.0）
**步骤 4：** 运行 `dotnet test` — 验证中间件测试通过（GREEN）
**步骤 5：** 在 Program.cs 配置 Supabase JWT 验证
**步骤 6：** 添加角色授权策略（User, Agent, Admin）
**步骤 7：** 配置 CORS、速率限制
**步骤 8：** 配置 Serilog 结构化日志（Console + File sinks，JSON 输出）
**步骤 9：** 在 DI 中注册 ITimeProvider（SystemTimeProvider）
**步骤 10：** 运行 `dotnet build` — 验证编译通过
**步骤 11：** 运行 `dotnet format --verify-no-changes` — 验证代码格式
**步骤 12：** 提交：`feat: add JWT auth, exception handling middleware with TDD, CORS, Serilog logging`

### 任务 16：Wallet + Game + Order 控制器

**文件：**
- 测试：`Backend/tests/GamePlatform.Integration.Tests/WalletControllerTests.cs`
- 测试：`Backend/tests/GamePlatform.Integration.Tests/GameControllerTests.cs`
- 创建：`Backend/src/GamePlatform.API/Controllers/WalletController.cs`
- 创建：`Backend/src/GamePlatform.API/Controllers/GameController.cs`
- 创建：`Backend/src/GamePlatform.API/Controllers/OrderController.cs`
- 创建：`Backend/src/GamePlatform.API/Controllers/AuthController.cs`

**步骤 1：** 编写关键端点的集成测试（参见 `05-测试设计-testing_design.md` §4.2，包含银行卡 CRUD、充值/提现申请查询端点）
**步骤 2：** 实现控制器（薄层 — 委托给 Application 服务，含银行卡管理端点）
**步骤 3：** 运行集成测试 — 验证通过
**步骤 4：** 提交：`feat: add user-facing API controllers (wallet, bank-cards, game, order, auth)`

### 任务 17：Admin + Agent + 厂商回调控制器

**文件：**
- 测试：`Backend/tests/GamePlatform.Integration.Tests/AdminControllerTests.cs`
- 创建：`Backend/src/GamePlatform.API/Controllers/Admin/*.cs`（所有管理控制器）
- 创建：`Backend/src/GamePlatform.API/Controllers/Agent/*.cs`
- 创建：`Backend/src/GamePlatform.API/Controllers/Vendor/VendorCallbackController.cs`

**步骤 1：** 编写管理端点 + 回调的集成测试（参见 `05-测试设计-testing_design.md` §4.2，包含 dashboard stats 端点）
**步骤 2：** 实现管理控制器（包含 force-exit、bet-records、vendor/balance、dashboard/stats 端点）
**步骤 3：** 实现代理控制器（包含 AgentAuthorizationFilter 归属验证、bet-records、dashboard/stats 端点）
**步骤 4：** 实现厂商回调控制器（预留，当前厂商不使用）
**步骤 5：** 运行所有测试
**步骤 6：** 提交：`feat: add admin, agent, and vendor callback API controllers`

---

## 阶段 6：SignalR + 后台服务

### 任务 18：SignalR NotificationHub

**文件：**
- 测试：`Backend/tests/GamePlatform.Application.Tests/Notifications/SignalRNotificationServiceTests.cs`
- 创建：`Backend/src/GamePlatform.API/Hubs/NotificationHub.cs`
- 创建：`Backend/src/GamePlatform.Application/Interfaces/INotificationService.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/Notifications/SignalRNotificationService.cs`

**步骤 1：** 定义 INotificationService 接口（参见 `03-详细设计-detailed_design.md` §7.1）
**步骤 2：** 编写 SignalRNotificationService 失败测试（RED）（Mock IHubContext）
  - NotifyBalanceUpdatedAsync — 验证向指定用户组发送正确消息
  - NotifyOrderStatusChangedAsync — 验证消息格式
  - NotifyAdminsAlertAsync — 验证向管理员组广播
**步骤 3：** 运行 `dotnet test` — 验证测试失败（RED）
**步骤 4：** 实现 NotificationHub（JWT 认证，参见 `03-详细设计-detailed_design.md` §7）
**步骤 5：** 实现 SignalRNotificationService
**步骤 6：** 运行 `dotnet test` — 验证测试通过（GREEN）
**步骤 7：** 在 WalletService 和 OrderService 中接入通知
**步骤 8：** 运行 `dotnet test` — 验证所有现有测试仍然通过
**步骤 9：** 运行 `dotnet format --verify-no-changes` — 验证代码格式
**步骤 10：** 提交：`feat: add SignalR notification hub and service with TDD`

### 任务 19：后台服务

**文件：**
- 测试：`Backend/tests/GamePlatform.Application.Tests/BackgroundServices/PendingOrderRecoveryServiceTests.cs`
- 测试：`Backend/tests/GamePlatform.Application.Tests/BackgroundServices/BalanceReconciliationServiceTests.cs`
- 测试：`Backend/tests/GamePlatform.Application.Tests/BackgroundServices/OrderPollingServiceTests.cs`
- 测试：`Backend/tests/GamePlatform.Application.Tests/BackgroundServices/BetRecordSyncServiceTests.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/BackgroundServices/PendingOrderRecoveryService.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/BackgroundServices/BalanceReconciliationService.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/BackgroundServices/OrderPollingService.cs`
- 创建：`Backend/src/GamePlatform.Infrastructure/BackgroundServices/BetRecordSyncService.cs`

**步骤 1：** 编写 PendingOrderRecoveryService 失败测试（RED）（参见 `05-测试设计-testing_design.md` §3.2 轮询部分）
  - Pending 订单被拾取并恢复
  - 恢复成功 → Confirmed
  - 恢复失败 → Failed + 退款
  - 超过最大重试 → 告警
  - 幂等：重复执行不产生副作用
**步骤 2：** 编写 BalanceReconciliationService 失败测试（RED）
  - 余额一致 → 无告警
  - 余额不一致 → 触发告警 + 记录 AdminActionLog
**步骤 3：** 编写 OrderPollingService 失败测试（RED）（参见 §10.3）
  - Confirmed 订单被轮询
  - 厂商余额为 0 → Cancel
  - 厂商余额 > 0 → TransferAll + Settle
  - 轮询失败 → 保持 Confirmed + 记录日志
**步骤 4：** 编写 BetRecordSyncService 失败测试（RED）（参见 `05-测试设计-testing_design.md` §3.4）
  - 正常同步 → 调用 UpsertBatchAsync
  - 空结果 → 不报错
  - 厂商 API 失败 → 记录日志，不中断
**步骤 5：** 运行 `dotnet test` — 验证测试失败（RED）
**步骤 6：** 实现 PendingOrderRecoveryService — 使用 transferStatus 查询恢复 Pending 订单（参见 `03-详细设计-detailed_design.md` §10.2）
**步骤 7：** 实现 BalanceReconciliationService（比对 Balance vs SUM，参见 §10.1）
**步骤 8：** 实现 OrderPollingService — 使用 QueryBalance + TransferAll 轮询 Confirmed 订单（参见 §10.3）
**步骤 9：** 实现 BetRecordSyncService — 每 2 分钟同步实时投注记录（参见 §10.4）
**步骤 10：** 运行 `dotnet test` — 验证所有测试通过（GREEN）
**步骤 11：** 运行 `dotnet format --verify-no-changes` — 验证代码格式
**步骤 12：** 提交：`feat: add background services with TDD for recovery, reconciliation, polling, bet record sync`

---

## 阶段 7：前端 — 用户端

### 任务 20：认证页面 + Supabase 集成

**文件：**
- 测试：`Frontend/user-portal/src/app/(auth)/__tests__/login.test.tsx`
- 测试：`Frontend/user-portal/src/app/(auth)/__tests__/register.test.tsx`
- 创建：`Frontend/user-portal/src/lib/supabase/client.ts`
- 创建：`Frontend/user-portal/src/lib/supabase/server.ts`
- 创建：`Frontend/user-portal/src/app/(auth)/login/page.tsx`
- 创建：`Frontend/user-portal/src/app/(auth)/register/page.tsx`
- 创建：`Frontend/user-portal/src/app/layout.tsx`

**步骤 1：** 配置 Supabase 客户端（浏览器 + 服务器端）
**步骤 2：** 编写登录页面失败测试（RED）（参见 `05-测试设计-testing_design.md` §8.3）
  - 渲染登录表单（用户名 + 密码 + 提交按钮）
  - 空字段提交显示验证错误
  - 登录成功后跳转
  - 登录失败显示错误信息（MSW 模拟 Supabase 返回错误）
**步骤 3：** 编写注册页面失败测试（RED）
  - 渲染注册表单
  - 密码不一致显示错误
  - 注册成功后跳转到登录
**步骤 4：** 运行 `npm run test` — 验证测试失败（RED）
**步骤 5：** 实现登录页面（含表单验证）
**步骤 6：** 实现注册页面
**步骤 7：** 添加认证中间件（未登录用户重定向）
**步骤 8：** 运行 `npm run test` — 验证测试通过（GREEN）
**步骤 9：** 验证登录后跳转到首页
**步骤 10：** 提交：`feat: add user portal auth pages with Supabase integration and tests`

### 任务 21：游戏大厅 + 启动

**文件：**
- 测试：`Frontend/user-portal/src/components/__tests__/GameCard.test.tsx`
- 测试：`Frontend/user-portal/src/components/__tests__/GameGrid.test.tsx`
- 测试：`Frontend/user-portal/src/app/games/__tests__/launch.test.tsx`
- 创建：`Frontend/user-portal/src/app/page.tsx`（游戏大厅）
- 创建：`Frontend/user-portal/src/app/games/[gameId]/launch/page.tsx`
- 创建：`Frontend/user-portal/src/components/GameCard.tsx`
- 创建：`Frontend/user-portal/src/components/GameGrid.tsx`
- 创建：`Frontend/user-portal/src/lib/api/games.ts`

**步骤 1：** 编写 GameCard/GameGrid 组件失败测试（RED）
  - GameCard 渲染游戏名称、图标、平台标签
  - GameGrid 按平台筛选显示正确数量
**步骤 2：** 编写游戏启动页面失败测试（RED）
  - 金额输入 + 快捷金额按钮
  - 金额限额校验（低于最小值显示错误）
  - 单会话限制错误处理（MSW 模拟 ACTIVE_ORDER_EXISTS 错误）
**步骤 3：** 运行 `npm run test` — 验证测试失败（RED）
**步骤 4：** 实现游戏大厅页面（响应式网格 + 平台选择器：全部/AG/PG/PP/CQ9 等）
**步骤 5：** 实现游戏启动页面（金额输入 + 快捷金额 + 金额限额校验 + 设备类型自动检测 ingress）
**步骤 6：** 实现"游戏进行中"状态页面（退出游戏按钮 + 窗口关闭检测）
**步骤 7：** 对接后端 API（包含单会话限制错误处理）
**步骤 8：** 处理新窗口打开游戏 URL
**步骤 9：** 运行 `npm run test` — 验证测试通过（GREEN）
**步骤 10：** 提交：`feat: add game lobby with platform selector, launch, and in-progress state pages with tests`

### 任务 22：钱包页面

**文件：**
- 测试：`Frontend/user-portal/src/stores/__tests__/useWalletStore.test.ts`
- 测试：`Frontend/user-portal/src/app/wallet/__tests__/deposit.test.tsx`
- 测试：`Frontend/user-portal/src/app/wallet/__tests__/withdraw.test.tsx`
- 测试：`Frontend/user-portal/src/components/__tests__/BankCardList.test.tsx`
- 测试：`Frontend/user-portal/src/components/__tests__/AddBankCardDialog.test.tsx`
- 创建：`Frontend/user-portal/src/app/wallet/page.tsx`
- 创建：`Frontend/user-portal/src/app/wallet/deposit/page.tsx`
- 创建：`Frontend/user-portal/src/app/wallet/withdraw/page.tsx`
- 创建：`Frontend/user-portal/src/app/wallet/transactions/page.tsx`
- 创建：`Frontend/user-portal/src/app/wallet/bank-cards/page.tsx`
- 创建：`Frontend/user-portal/src/components/BankCardList.tsx`
- 创建：`Frontend/user-portal/src/components/BankCardItem.tsx`
- 创建：`Frontend/user-portal/src/components/AddBankCardDialog.tsx`
- 创建：`Frontend/user-portal/src/lib/api/wallet.ts`
- 创建：`Frontend/user-portal/src/lib/api/bank-cards.ts`
- 创建：`Frontend/user-portal/src/stores/useWalletStore.ts`

**步骤 1：** 编写 useWalletStore 失败测试（RED）
  - 初始余额状态
  - fetchBalance 更新余额
  - SignalR 余额推送更新 store
**步骤 2：** 编写充值/提现页面失败测试（RED）（参见 `05-测试设计-testing_design.md` §8.3）
  - 充值：金额输入 + 限额校验 + 提交成功
  - 提现：金额限额 + 每日累计提示 + 银行卡选择 + 余额不足错误
**步骤 3：** 运行 `npm run test` — 验证测试失败（RED）
**步骤 4：** 实现钱包概览（余额 + Tab：流水记录 / 申请记录）
**步骤 5：** 实现充值申请页面（金额 + 图片上传 + 金额限额前端校验）
**步骤 6：** 实现提现申请页面（金额限额 + 每日累计提示 + 银行卡选择）
**步骤 7：** 实现银行卡管理页面（列表 + 添加/删除/设为默认，参见 `04-UI-UX设计-uiux_design.md`）
**步骤 8：** 实现流水记录分页列表
**步骤 9：** 实现申请记录 Tab（显示充值/提现申请状态：待审核/已通过/已拒绝/已完成）
**步骤 10：** 运行 `npm run test` — 验证测试通过（GREEN）
**步骤 11：** 提交：`feat: add wallet pages with bank-card management and store tests`

### 任务 23：订单 + 个人资料 + SignalR

**文件：**
- 测试：`Frontend/user-portal/src/lib/signalr/__tests__/client.test.ts`
- 测试：`Frontend/user-portal/src/stores/__tests__/useNotificationStore.test.ts`
- 创建：`Frontend/user-portal/src/app/orders/page.tsx`
- 创建：`Frontend/user-portal/src/app/orders/[orderId]/page.tsx`
- 创建：`Frontend/user-portal/src/app/profile/page.tsx`
- 创建：`Frontend/user-portal/src/lib/signalr/client.ts`
- 创建：`Frontend/user-portal/src/stores/useNotificationStore.ts`

**步骤 1：** 编写 SignalR client 失败测试（RED）
  - 连接建立 + 自动重连
  - 接收余额更新事件 → 更新 store
  - 接收通知事件 → 更新 notification store
**步骤 2：** 编写 useNotificationStore 失败测试（RED）
  - 添加通知 + 标记已读 + 清除
**步骤 3：** 运行 `npm run test` — 验证测试失败（RED）
**步骤 4：** 实现订单列表 + 详情页面
**步骤 5：** 实现投注记录页 `/bet-records`（分页 + 按平台/时间筛选）
**步骤 6：** 实现个人资料页面（含密码修改入口，调用 Supabase Auth updateUser）
**步骤 7：** 集成 SignalR 实时余额更新
**步骤 8：** 添加 Toast 通知（余额变动提醒）
**步骤 9：** 运行 `npm run test` — 验证测试通过（GREEN）
**步骤 10：** 提交：`feat: add order pages, bet records, profile, and SignalR integration with tests`

---

## 阶段 8：前端 — 管理/代理端

### 任务 24：管理端认证 + 布局

**文件：**
- 测试：`Frontend/admin-portal/src/components/__tests__/Sidebar.test.tsx`
- 测试：`Frontend/admin-portal/src/app/(auth)/__tests__/login.test.tsx`
- 创建：`Frontend/admin-portal/src/app/(auth)/login/page.tsx`
- 创建：`Frontend/admin-portal/src/app/admin/layout.tsx`（侧边栏导航）
- 创建：`Frontend/admin-portal/src/app/admin/dashboard/page.tsx`
- 创建：`Frontend/admin-portal/src/components/Sidebar.tsx`

**步骤 1：** 编写 Sidebar 组件失败测试（RED）
  - 渲染所有导航项（用户管理、钱包管理、订单管理等）
  - 当前路由高亮
  - Admin 和 Agent 角色显示不同菜单
**步骤 2：** 编写管理端登录页面失败测试（RED）
  - 非 Admin/Agent 角色登录被拒绝
**步骤 3：** 运行 `npm run test` — 验证测试失败（RED）
**步骤 4：** 实现管理端登录
**步骤 5：** 实现侧边栏布局和导航（参见 `04-UI-UX设计-uiux_design.md` §3.2）
**步骤 6：** 实现仪表盘（统计数据 + 待处理项）
**步骤 7：** 运行 `npm run test` — 验证测试通过（GREEN）
**步骤 8：** 提交：`feat: add admin portal auth, layout, and dashboard with tests`

### 任务 25：管理端 — 用户 + 钱包管理

**文件：**
- 测试：`Frontend/admin-portal/src/components/__tests__/AdjustWalletDialog.test.tsx`
- 创建：`Frontend/admin-portal/src/app/admin/users/page.tsx`
- 创建：`Frontend/admin-portal/src/app/admin/users/[id]/page.tsx`
- 创建：`Frontend/admin-portal/src/app/admin/wallets/page.tsx`
- 创建：`Frontend/admin-portal/src/app/admin/wallets/[userId]/page.tsx`
- 创建：`Frontend/admin-portal/src/components/AdjustWalletDialog.tsx`

**步骤 1：** 编写 AdjustWalletDialog 失败测试（RED）（参见 `05-测试设计-testing_design.md` §8.3）
  - 渲染金额输入 + 原因输入 + 类型选择（充值/扣款）
  - 金额为 0 或负数时提交按钮禁用
  - 提交成功后关闭弹窗 + 刷新列表
  - 提交失败显示错误信息
**步骤 2：** 运行 `npm run test` — 验证测试失败（RED）
**步骤 3：** 实现用户管理（列表 + 搜索 + 启用/禁用）
**步骤 4：** 实现钱包管理 + 手动调整对话框
**步骤 5：** 运行 `npm run test` — 验证测试通过（GREEN）
**步骤 6：** 提交：`feat: add admin user and wallet management pages with dialog tests`

### 任务 26：管理端 — 订单 + 申请 + 日志

**文件：**
- 测试：`Frontend/admin-portal/src/app/admin/deposit-requests/__tests__/page.test.tsx`
- 测试：`Frontend/admin-portal/src/app/admin/withdraw-requests/__tests__/page.test.tsx`
- 创建：`Frontend/admin-portal/src/app/admin/orders/` 相关页面
- 创建：`Frontend/admin-portal/src/app/admin/deposit-requests/` 相关页面
- 创建：`Frontend/admin-portal/src/app/admin/withdraw-requests/` 相关页面
- 创建：`Frontend/admin-portal/src/app/admin/logs/` 相关页面
- 创建：`Frontend/admin-portal/src/app/admin/alerts/page.tsx`
- 创建：`Frontend/admin-portal/src/app/admin/reconciliation/page.tsx`

**步骤 1：** 编写充值/提现申请审核页面失败测试（RED）
  - 待审核列表渲染
  - 审批按钮点击 → 确认弹窗
  - 审批成功后刷新列表 + 状态更新
  - 拒绝 + 原因输入
**步骤 2：** 运行 `npm run test` — 验证测试失败（RED）
**步骤 3：** 实现订单管理（列表 + 详情 + 状态变更 + 强制退出）
**步骤 4：** 实现充值/提现申请审核页面
**步骤 5：** 实现日志查看器（轮询日志、操作日志；回调日志预留）
**步骤 6：** 实现告警页面 + 对账触发
**步骤 7：** 实现投注记录页 `/admin/bet-records`（全局投注数据 + 按用户/平台筛选）
**步骤 8：** 实现商户余额页 `/admin/vendor/balance`（查看厂商余额）
**步骤 9：** 运行 `npm run test` — 验证测试通过（GREEN）
**步骤 10：** 提交：`feat: add admin order management, request review, bet records, vendor balance, and log pages with tests`

### 任务 27：代理端

**文件：**
- 测试：`Frontend/admin-portal/src/app/agent/__tests__/dashboard.test.tsx`
- 测试：`Frontend/admin-portal/src/app/agent/users/__tests__/page.test.tsx`
- 创建：`Frontend/admin-portal/src/app/agent/layout.tsx`
- 创建：`Frontend/admin-portal/src/app/agent/dashboard/page.tsx`
- 创建：`Frontend/admin-portal/src/app/agent/users/` 相关页面
- 创建：`Frontend/admin-portal/src/app/agent/orders/page.tsx`
- 创建：`Frontend/admin-portal/src/app/agent/reports/page.tsx`

**步骤 1：** 编写代理端仪表盘失败测试（RED）
  - 渲染下级用户数、活跃订单数等统计
  - 仅显示归属于该代理的数据
**步骤 2：** 编写下级用户列表失败测试（RED）
  - 列表仅包含归属用户（Agent 归属验证）
  - 搜索 + 分页
**步骤 3：** 运行 `npm run test` — 验证测试失败（RED）
**步骤 4：** 实现代理端布局（简化版侧边栏）
**步骤 5：** 实现下级用户列表 + 详情
**步骤 6：** 实现下级订单列表
**步骤 7：** 实现下级投注记录页 `/agent/bet-records`
**步骤 8：** 实现汇总报表页面
**步骤 9：** 运行 `npm run test` — 验证测试通过（GREEN）
**步骤 10：** 提交：`feat: add agent portal with user/order/bet-record views, reports, and tests`

---

## 阶段 9：特殊测试 + E2E

### 任务 28：幂等性 + 并发测试

**文件：**
- 创建：`Backend/tests/GamePlatform.Integration.Tests/Idempotency/`
- 创建：`Backend/tests/GamePlatform.Integration.Tests/Concurrency/`

**步骤 1：** 编写幂等性测试（参见 `05-测试设计-testing_design.md` §6.1）
**步骤 2：** 编写并发测试（使用 Task.WhenAll，参见 `05-测试设计-testing_design.md` §6.2）
**步骤 3：** 编写恢复测试（参见 `05-测试设计-testing_design.md` §6.3）
**步骤 4：** 运行所有测试
**步骤 5：** 提交：`test: add idempotency, concurrency, and recovery integration tests`

### 任务 29：E2E 测试（Playwright）

**文件：**
- 创建：`e2e/playwright.config.ts`
- 创建：`e2e/seed/seed-test-data.ts`（测试种子数据脚本）
- 创建：`e2e/seed/test-users.json`（测试用户配置）
- 创建：`e2e/tests/user-registration.spec.ts`
- 创建：`e2e/tests/game-flow.spec.ts`
- 创建：`e2e/tests/admin-management.spec.ts`
- 创建：`e2e/tests/full-business-flow.spec.ts`

**步骤 1：** 配置 Playwright（baseURL、超时、截图策略）
**步骤 2：** 创建 E2E 测试种子数据（参见 `05-测试设计-testing_design.md` §7.1）
  - 测试用户：普通用户（有余额）、代理用户、管理员用户
  - 测试钱包：预设余额 10000
  - 每个 User 预设 1-2 张 UserBankCard（1 张为默认）
  - Supabase Auth 预创建测试账号
  - `globalSetup` 中执行种子数据初始化
**步骤 3：** 编写用户注册 + 登录测试
**步骤 4：** 编写游戏启动流程测试（含平台选择 + 玩家映射 + 一键回收）
**步骤 5：** 编写银行卡管理 + 提现选卡流程测试
**步骤 6：** 编写管理端测试（含投注记录 + 商户余额页面）
**步骤 7：** 编写完整业务链路测试（参见 `05-测试设计-testing_design.md` §5.3 链路 A/B/C）
  - 链路 A：充值 → 游戏 → 一键回收 → 管理员干预
  - 链路 B：Pending 订单 → transferStatus 恢复
  - 链路 C：投注记录同步验证
**步骤 8：** 运行 `npx playwright test`
**步骤 9：** 提交：`test: add Playwright E2E tests with seed data for critical business flows`

---

## 阶段 10：CI/CD + 最终验证

### 任务 30：GitHub Actions CI

**文件：**
- 创建：`.github/workflows/backend-ci.yml`
- 创建：`.github/workflows/frontend-ci.yml`
- 创建：`.github/workflows/e2e-ci.yml`

**步骤 1：** 实现后端 CI（参见 `06-CI-CD设计-ci_cd_design.md` §2.1，包含 `dotnet list package --vulnerable` 安全扫描）
**步骤 2：** 实现前端 CI（参见 `06-CI-CD设计-ci_cd_design.md` §2.2 + §2.3，包含 `npm audit` + Vitest）
**步骤 3：** 实现 E2E CI（参见 `06-CI-CD设计-ci_cd_design.md` §2.4）
**步骤 4：** 配置 `main` 分支保护规则
**步骤 5：** 推送并验证 CI 运行
**步骤 6：** 提交：`ci: add GitHub Actions workflows for backend, frontend, and E2E`

### 任务 31：覆盖率检查 + 最终验证

**步骤 1：** 运行 `dotnet test --collect:"XPlat Code Coverage"` — 验证 >= 85%
**步骤 2：** 运行 `dotnet format --verify-no-changes` — 验证代码格式一致
**步骤 3：** 运行两个前端项目 `npm run test:coverage` — 验证前端覆盖率 >= 80%
**步骤 4：** 运行两个前端项目 `npm run build` — 验证编译通过
**步骤 5：** 运行 `npm audit` — 验证无高危漏洞
**步骤 6：** 运行 `dotnet list package --vulnerable` — 验证无已知漏洞
**步骤 7：** 运行 E2E 测试 — 验证关键路径通过
**步骤 8：** 检查所有质量门禁（参见 `06-CI-CD设计-ci_cd_design.md` §3）
**步骤 9：** 最终提交：`chore: verify MVP quality gates pass`

---

## 实施说明

### 任务依赖关系

```
任务 1-2：脚手架搭建（可并行）
任务 3-6：Domain 实体（顺序执行 — 每个依赖前一个）
任务 7-9：Infrastructure（顺序执行 — 依赖 Domain）
任务 10-11：钱包服务（依赖 Infrastructure）
任务 12-14：订单服务（依赖钱包服务）
任务 15-17：API 层（依赖 Application 层）
任务 18-19：SignalR + 后台服务（依赖 API）
任务 20-23：用户端（可与任务 24-27 并行，依赖 API）
任务 24-27：管理端（可与任务 20-23 并行，依赖 API）
任务 28-29：特殊测试 + E2E（依赖前后端）
任务 30-31：CI/CD + 验证（最后执行）
```

### 可并行化机会

- 任务 1（后端脚手架）+ 任务 2（前端脚手架）
- 任务 20-23（用户端）+ 任务 24-27（管理端）
- 任务 28（后端特殊测试）+ 任务 29（E2E 测试）

### 每个任务的完成定义（DoD）

每个任务提交前**必须满足以下全部条件**，否则不允许提交：

**后端任务（任务 1, 3-19, 28）：**
- [ ] `dotnet build` — 编译通过，无警告
- [ ] `dotnet test` — 所有测试通过
- [ ] `dotnet format --verify-no-changes` — 代码格式一致
- [ ] 无硬编码密钥/连接字符串
- [ ] 测试先于实现（RED → GREEN → REFACTOR）
- [ ] 新代码首行包含 `// AUTO-GENERATED BY CLAUDE` 标识

**前端任务（任务 2, 20-27）：**
- [ ] `npm run build` — 编译通过
- [ ] `npm run test` — 所有测试通过
- [ ] `npm run lint` — Lint 通过
- [ ] 无硬编码 API URL 或密钥
- [ ] 测试先于实现（RED → GREEN）
- [ ] 新文件首行包含 `// AUTO-GENERATED BY CLAUDE` 标识

**E2E 任务（任务 29）：**
- [ ] 种子数据可重复执行（幂等）
- [ ] `npx playwright test` — 所有测试通过
- [ ] 截图/视频 artifacts 配置正确

**CI 任务（任务 30）：**
- [ ] CI pipeline 在 GitHub Actions 上运行成功
- [ ] 质量门禁与 `06-CI-CD设计-ci_cd_design.md` §3 一致

### 关键检查点

每个阶段完成后验证：
- [ ] 所有测试通过
- [ ] 覆盖率达标（后端 ≥85%，前端 ≥80%）
- [ ] 代码编译无警告
- [ ] `dotnet format` / `npm run lint` 格式通过
- [ ] 无硬编码密钥
- [ ] 创建 PR 并**等待 review 确认**后再继续下一阶段
