# 测试设计

> 文档编号：05-测试设计-testing_design.md | 版本：1.4 | 日期：2026-03-04
> 依赖文档：overview.md（模块定义）, detailed_design.md（接口与实体）

## 1. 测试策略总览

### 1.1 测试金字塔

```
            ┌───────┐
            │  E2E  │    ← 关键业务路径（Playwright）
          ┌─┴───────┴─┐
          │   集成     │  ← API + DB（WebApplicationFactory + Testcontainers）
        ┌─┴───────────┴─┐
        │     单元       │  ← Domain + Application（xUnit + Moq）
        └───────────────┘
```

### 1.2 覆盖率目标

| 层 | 目标 | 说明 |
|----|------|------|
| Domain | >= 90% | 核心业务规则，必须全面覆盖 |
| Application | >= 85% | 用例编排，Mock 外部依赖 |
| API | >= 80% | 端点测试，集成测试为主 |
| **总体 Backend** | **>= 85%** | **CI 门禁阈值** |
| **Frontend（组件/Store/Hook）** | **>= 80%** | **Vitest 覆盖率门禁阈值（vitest.config.ts 中配置）** |

### 1.3 工具链

| 工具 | 用途 |
|------|------|
| xUnit | 测试框架 |
| Moq | Mock 框架 |
| FluentAssertions | 断言库 |
| Coverlet | 覆盖率收集 |
| Microsoft.AspNetCore.Mvc.Testing | 集成测试（WebApplicationFactory） |
| Testcontainers.PostgreSql | 数据库集成测试 |
| Playwright | E2E 测试 |
| Bogus | 测试数据生成 |

### 1.4 TDD 工作流

每个功能必须遵循：

1. **RED** — 先写测试，运行确认失败
2. **GREEN** — 写最小实现，运行确认通过
3. **REFACTOR** — 重构代码，确保测试仍通过
4. **COMMIT** — 每个 RED-GREEN-REFACTOR 循环后提交

---

## 2. 单元测试场景

### 2.1 Wallet 实体

| 场景 | 输入 | 预期结果 |
|------|------|----------|
| Credit 正常金额 | amount=100, balance=500 | balance=600, Version+1, 返回 Transaction |
| Credit 金额为 0 | amount=0 | 抛出 DomainException |
| Credit 负数金额 | amount=-50 | 抛出 DomainException |
| Credit 极大金额 | amount=999999999999.99 | 正常处理（decimal 范围内） |
| Debit 正常（余额充足） | amount=100, balance=500 | balance=400, Version+1, 返回 Transaction |
| Debit 余额不足 | amount=600, balance=500 | 抛出 InsufficientBalanceException |
| Debit 金额等于余额 | amount=500, balance=500 | balance=0, 成功 |
| Debit 金额为 0 | amount=0 | 抛出 DomainException |
| Debit 负数金额 | amount=-50 | 抛出 DomainException |
| Transaction BalanceBefore/After 正确 | credit 100 from 500 | before=500, after=600 |
| Transaction Amount 符号正确 | credit=+, debit=- | Credit 返回正值, Debit 返回负值 |
| Debit 检查可用余额（非总余额） | balance=500, frozen=200, debit 400 | 抛出 InsufficientBalanceException（可用=300） |
| Debit 可用余额刚好够 | balance=500, frozen=200, debit 300 | 成功, balance=200 |
| Freeze 正常 | availableBalance=500, freeze 200 | frozenBalance=200, Version+1 |
| Freeze 余额不足 | availableBalance=100, freeze 200 | 抛出 InsufficientBalanceException |
| Unfreeze 正常 | frozenBalance=200, unfreeze 200 | frozenBalance=0, Version+1 |
| Unfreeze 超过冻结额 | frozenBalance=100, unfreeze 200 | 抛出 DomainException |
| ConfirmFrozenDebit 正常 | balance=500, frozen=200, confirm 200 | balance=300, frozen=0, 返回 Transaction |
| ConfirmFrozenDebit 冻结不足 | frozenBalance=100, confirm 200 | 抛出 DomainException |
| AvailableBalance 计算 | balance=1000, frozen=300 | availableBalance=700 |

**金额限额测试（AmountLimitOptions）**：

| 场景 | 输入 | 预期结果 |
|------|------|----------|
| 充值金额低于最小限额 | amount=5, min=10 | 拒绝 |
| 充值金额超过最大限额 | amount=60000, max=50000 | 拒绝 |
| 充值金额在限额内 | amount=100, min=10, max=50000 | 通过 |
| 提现金额低于最小限额 | amount=5, min=10 | 拒绝 |
| 提现金额超过最大限额 | amount=60000, max=50000 | 拒绝 |
| 每日累计提现超限 | 当日已提现 90000, 申请 20000, limit=100000 | 拒绝 |
| 每日累计提现未超限 | 当日已提现 80000, 申请 10000, limit=100000 | 通过 |
| 游戏转入金额低于最小限额 | amount=0.5, min=1 | 拒绝 |

### 2.2 OrderStateMachine

**合法路径（全部必须通过）**：

| 起始状态 | 目标状态 | 预期 |
|----------|----------|------|
| Pending | Confirmed | 允许 |
| Pending | Failed | 允许 |
| Confirmed | Settled | 允许 |
| Confirmed | Cancelled | 允许 |

**非法路径（全部必须抛出 InvalidStateTransitionException）**：

| 起始状态 | 目标状态 | 预期 |
|----------|----------|------|
| Pending | Settled | 异常（跳跃） |
| Pending | Cancelled | 异常（跳跃） |
| Confirmed | Pending | 异常（回退） |
| Confirmed | Failed | 异常（非法） |
| Settled | Pending | 异常（终态） |
| Settled | Confirmed | 异常（终态） |
| Settled | Cancelled | 异常（终态） |
| Settled | Failed | 异常（终态） |
| Cancelled | * (任意) | 异常（终态） |
| Failed | * (任意) | 异常（终态） |

### 2.3 GameOrder 实体

| 场景 | 预期 |
|------|------|
| 创建订单（正常参数） | 状态 Pending, InternalOrderId 非空 |
| Confirm（设置 ExternalOrderId + GameUrl） | 状态变为 Confirmed |
| Settle（设置 TransferOutAmount） | 状态变为 Settled |
| Cancel（正常） | 状态变为 Cancelled |
| Cancel 设置 CancelReason | CancelReason 正确记录 |
| Fail（设置 ErrorMessage） | 状态变为 Failed |
| Settle 时 TransferOutAmount <= 0 | 抛出 DomainException |
| Confirm 时 ExternalOrderId 为空 | 抛出 DomainException |
| Confirm 时 GameUrl 为空 | 抛出 DomainException |

**单会话限制测试**：

| 场景 | 预期 |
|------|------|
| 用户已有 Confirmed 订单，再次 LaunchGame | 拒绝，抛出 DomainException "已有进行中的游戏" |
| 用户已有 Pending 订单，再次 LaunchGame | 拒绝，抛出 DomainException |
| 用户无活跃订单（全部 Settled/Cancelled/Failed） | 允许 LaunchGame |

### 2.4 VendorPlayerMapping

| 场景 | 预期 |
|------|------|
| Create（正常参数） | VendorPlayerId 格式正确（5-11 位小写字母+数字），IsCreated=false |
| MarkCreated | IsCreated=true，UpdatedAt 更新 |
| VendorPlayerId 唯一性 | 同一 VendorCode 下不同用户的 VendorPlayerId 不重复 |
| VendorPlayerId 格式 | 长度 5-11 位，仅小写字母+数字 |

### 2.5 UserProfile 实体

| 场景 | 预期 |
|------|------|
| Disable Active 用户 | 状态变为 Disabled |
| Enable Disabled 用户 | 状态变为 Active |
| Disable 已 Disabled 用户 | 幂等（无异常） |
| Enable 已 Active 用户 | 幂等（无异常） |
| AssignAgent（有效代理 ID） | AgentId 更新 |
| AssignAgent（自己的 ID） | 抛出 DomainException |
| UpdateNickname（正常） | Nickname 更新 |
| UpdateNickname（空字符串） | 抛出 DomainException |

### 2.6 DepositRequest / WithdrawRequest

| 场景 | 预期 |
|------|------|
| Approve Pending 请求 | 状态变为 Approved, ReviewedBy 设置 |
| Reject Pending 请求 | 状态变为 Rejected |
| Approve 已 Approved 请求 | 抛出异常（防止重复审批） |
| Approve 已 Rejected 请求 | 抛出异常 |
| 创建时金额 <= 0 | 抛出 DomainException |
| Complete Approved 提现请求 | 状态变为 Completed |
| Complete Pending 提现请求 | 抛出异常（必须先 Approve） |
| Complete Rejected 提现请求 | 抛出异常 |
| Complete 已 Completed 提现请求 | 抛出异常（防止重复完成） |

### 2.7 UserBankCard 实体

| 场景 | 预期 |
|------|------|
| Create（正常参数） | 银行卡创建成功，IsDefault=false |
| Create 卡号长度 < 16 位 | 抛出 DomainException |
| Create 卡号长度 > 19 位 | 抛出 DomainException |
| Create 持卡人为空 | 抛出 DomainException |
| Create 银行名称为空 | 抛出 DomainException |
| SetDefault(true) | IsDefault=true, UpdatedAt 更新 |
| SetDefault(false) | IsDefault=false |
| 同一用户绑定第 6 张卡 | 抛出 DomainException "最多绑定 5 张银行卡" |

### 2.8 DepositRequestStateMachine / WithdrawRequestStateMachine

**DepositRequest 合法路径**：

| 起始状态 | 目标状态 | 预期 |
|----------|----------|------|
| Pending | Approved | 允许 |
| Pending | Rejected | 允许 |

**DepositRequest 非法路径**（全部抛异常）：

| 起始状态 | 目标状态 | 预期 |
|----------|----------|------|
| Approved | Pending | 异常（终态） |
| Approved | Rejected | 异常（终态） |
| Rejected | Pending | 异常（终态） |
| Rejected | Approved | 异常（终态） |

**WithdrawRequest 合法路径**：

| 起始状态 | 目标状态 | 预期 |
|----------|----------|------|
| Pending | Approved | 允许 |
| Pending | Rejected | 允许 |
| Approved | Completed | 允许 |

**WithdrawRequest 非法路径**（全部抛异常）：

| 起始状态 | 目标状态 | 预期 |
|----------|----------|------|
| Pending | Completed | 异常（跳跃） |
| Approved | Pending | 异常（回退） |
| Approved | Rejected | 异常（非法） |
| Rejected | * (任意) | 异常（终态） |
| Completed | * (任意) | 异常（终态） |

### 2.8 WalletRules

| 场景 | 预期 |
|------|------|
| ValidateAmount(100) | 通过 |
| ValidateAmount(0) | 抛出异常 |
| ValidateAmount(-1) | 抛出异常 |
| ValidateSufficientBalance(500, 400) | 通过 |
| ValidateSufficientBalance(500, 500) | 通过 |
| ValidateSufficientBalance(500, 501) | 抛出 InsufficientBalanceException |

---

## 3. Application 层单元测试

### 3.1 WalletService

| 场景 | 预期 |
|------|------|
| AdminDeposit 正常 | Wallet.Credit + WalletTransaction + AdminActionLog |
| AdminDeposit 用户不存在 | 抛出 NotFoundException |
| AdminDeposit CAS 版本冲突 | 抛出 ConcurrencyConflictException |
| AdminWithdraw 余额不足 | 抛出 InsufficientBalanceException |
| ApproveDeposit 正常 | 更新 Request + Wallet.Credit + Transaction + Log |
| ApproveDeposit 已审批（幂等） | 返回成功，不重复充值 |
| ApproveDeposit 已拒绝 | 抛出异常 |
| RequestDeposit 金额 <= 0 | 抛出验证错误 |
| RequestWithdrawal 正常 | 钱包 Freeze(amount) + 创建 WithdrawRequest(Pending) |
| RequestWithdrawal 可用余额不足 | 抛出 InsufficientBalanceException（不冻结） |
| ApproveWithdrawal 正常 | ConfirmFrozenDebit + 更新 Request + AdminActionLog |
| ApproveWithdrawal 已审批（幂等） | 返回成功，不重复扣款 |
| RejectWithdrawal 正常 | Unfreeze(amount) + 更新 Request + AdminActionLog |
| RejectWithdrawal 已拒绝 | 抛出异常 |
| CompleteWithdrawal 正常 | 更新 Request 状态为 Completed + AdminActionLog |
| CompleteWithdrawal 请求非 Approved | 抛出异常 |
| ReconcileAsync | 对比 Balance 和 SUM(Transactions)，不一致时报告 |

### 3.2 OrderService

| 场景 | 预期 |
|------|------|
| LaunchGame 正常流程 | 玩家映射 + 创建订单 + Debit + 厂商 TransferIn + Confirm |
| LaunchGame 玩家首次使用 | 自动创建 VendorPlayerMapping + CreatePlayer |
| LaunchGame 厂商 CreatePlayer 返回 10002（已存在） | 标记 IsCreated=true，继续流程 |
| LaunchGame 余额不足 | 抛出异常，无订单创建 |
| LaunchGame 厂商转入失败（10005） | 订单 Fail + Refund Credit |
| LaunchGame 厂商转入返回非 10000/10005 | 调用 transferStatus 轮询确认 |
| LaunchGame transferStatus 返回 pending→success | 订单 Confirm |
| LaunchGame transferStatus 返回 pending→failed | 订单 Fail + Refund Credit |
| LaunchGame 厂商获取 URL 失败 | 订单 Fail + Refund Credit |
| ExitGame 正常 | 一键回收 TransferAll + Credit + Settle |
| ExitGame 订单非 Confirmed 状态 | 抛出异常 |
| ExitGame 一键回收失败 | 保持 Confirmed 状态，返回错误 |
| ExitGame 一键回收金额为 0 | 订单 Cancel（非 Settle），不执行退款 |
| AdminUpdateStatus Cancel Confirmed 订单 | 订单 Cancel + AdminActionLog 记录 |
| HandleCallback 正常回调（**预留测试**） | 验签 + 状态更新 + 结算 |
| HandleCallback 签名无效（**预留测试**） | 记录日志（IsVerified=false），返回 401 |
| HandleCallback 重复回调幂等（**预留测试**） | 订单已终态，跳过，返回 200 |
| PollPendingOrders 正常 | 查询过期 Pending 订单，通过 transferStatus 恢复 |
| PollPendingOrders transferStatus 返回 success | Confirm 订单 |
| PollPendingOrders transferStatus 返回 failed | Fail + Refund |
| PollPendingOrders transferStatus 返回 pending | 等待下次轮询 |
| PollPendingOrders 订单不存在（10013） | Fail + Refund |
| PollPendingOrders 轮询次数 < MaxRetryCount | 继续轮询，Attempt+1，记录 VendorQueryLog |
| PollPendingOrders 轮询次数 = MaxRetryCount | 停止轮询，触发 PollingMaxRetryExceeded 告警 |
| PollPendingOrders 已超过 MaxRetryCount | 跳过该订单，不再轮询 |
| LaunchGame 已有活跃订单（单会话限制） | 抛出异常 "已有进行中的游戏" |
| LaunchGame 金额低于最小游戏转入限额 | 抛出异常 |
| AdminForceExitGame 正常 | 一键回收 + 钱包 Credit + Settle + AdminActionLog |
| AdminForceExitGame 回收金额为 0 | Cancel + AdminActionLog |
| AdminForceExitGame 非 Confirmed 订单 | 抛出异常 |
| AdminForceExitGame 一键回收 API 失败 | 返回错误，保持 Confirmed |

### 3.3 UserService / GameService

**UserService**：

| 场景 | 预期 |
|------|------|
| SyncProfile 新用户 | 创建 UserProfile + 自动创建 Wallet(Balance=0) |
| SyncProfile 已存在用户 | 返回现有 Profile，不重复创建 |
| SyncProfile 已存在用户（幂等） | 并发调用不产生重复记录 |
| DisableUser 正常 | UserProfile.Status=Disabled + AdminActionLog |
| EnableUser 正常 | UserProfile.Status=Active + AdminActionLog |
| GetAgentUsers（Agent 查自己下级） | 返回 AgentId 匹配的用户列表 |
| GetAgentUsers（Agent 查非自己下级） | 返回空列表或 403 |

**GameService**：

| 场景 | 预期 |
|------|------|
| GetGameList 首次请求 | 调用 IVendorAdapter.GetGameListAsync，缓存结果 |
| GetGameList 缓存命中 | 返回缓存，不调用厂商 API |
| GetGameList 缓存过期（10 分钟后） | 重新调用厂商 API |
| GetGameList 厂商 API 失败 | 返回空列表或抛出异常 |
| GetGameList 按分类筛选 | 返回匹配分类的游戏 |

### 3.4 UserBankCardService

| 场景 | 预期 |
|------|------|
| AddBankCard 正常 | 创建银行卡记录 |
| AddBankCard 用户已有 5 张卡 | 抛出 DomainException |
| AddBankCard 设为默认 | 新卡 IsDefault=true，旧默认卡 IsDefault=false |
| DeleteBankCard 正常 | 删除银行卡记录 |
| DeleteBankCard 非本人银行卡 | 抛出 NotFoundException 或 403 |
| SetDefaultCard 正常 | 目标卡 IsDefault=true，其余卡 IsDefault=false |
| GetUserBankCards | 返回用户所有银行卡列表 |

### 3.5 BetRecordSyncService

| 场景 | 预期 |
|------|------|
| 同步实时记录（正常） | 调用 GetRealtimeRecordsAsync，Upsert 到 GameBetRecords |
| 同步记录（幂等，重复 VendorBetOrderId） | 已存在记录更新，不重复插入 |
| 同步记录（分页，total > pageSize） | 自动翻页获取全部记录 |
| 同步记录（VendorPlayerId 找不到对应用户） | 跳过该记录，记录日志 |
| 同步记录（厂商 API 失败） | 记录日志，下次重试 |
| 同步记录（限频，10009） | 等待后重试 |

### 3.5 VendorAdapter 签名与错误码

| 场景 | 预期 |
|------|------|
| 签名计算 MD5(random+sn+secretKey) | 32 位小写 MD5，与厂商验证一致 |
| random 生成 | 16-32 位小写字母+数字 |
| 错误码 10002 处理（CreatePlayer 已存在） | 视为成功，MarkCreated |
| 错误码 10005 处理（转账失败） | 明确失败，触发退款 |
| 错误码 10009 处理（限频） | 等待后重试 + 告警 |
| 错误码 10012 处理（orderId 重复） | 查询 transferStatus 确认 |
| 错误码 10014 处理（额度不足） | 商户余额告警 |
| 非明确结果转账（非 10000/10005） | 调用 transferStatus 确认 |
| 客户端限流（balanceAll 3次/分钟） | 超限时排队等待 |

### 3.6 AlertService

| 场景 | 预期 |
|------|------|
| RaiseAlert BalanceInconsistency | 创建 AdminActionLog(Level=Error, Action="BalanceInconsistency") + SignalR 推送 admins 组 |
| RaiseAlert DuplicateSettlement | 创建 AdminActionLog(Level=Error) + SignalR 推送 |
| GetAlerts 分页查询 | 仅返回 Level=Error 的记录，按时间降序 |

### 3.7 Mock 策略

| 依赖 | Mock 方式 |
|------|----------|
| IWalletRepository | Moq — 模拟 CAS 成功/失败 |
| IOrderRepository | Moq — 返回预设订单 |
| IVendorAdapter | Moq — 模拟厂商 API 成功/失败/超时/限频(10009)/非明确结果 |
| IVendorPlayerMappingRepository | Moq — 模拟玩家映射 |
| IGameBetRecordRepository | Moq — 模拟投注记录存储 |
| IUserBankCardRepository | Moq — 模拟银行卡 CRUD |
| INotificationService | Moq — 验证通知被发送 |
| IUnitOfWork | Moq — 验证事务提交/回滚 |
| ITimeProvider | Moq — 控制时间 |

---

## 4. 集成测试场景

### 4.1 基础设施

- 使用 `WebApplicationFactory<Program>` 启动测试服务器
- 使用 `Testcontainers.PostgreSql` 启动 PostgreSQL 容器
- 每个测试类独立数据库 schema 或事务回滚隔离

### 4.2 API 端点测试

| 场景 | 预期 |
|------|------|
| GET /api/wallet/balance（有效 Token） | 200 + 余额数据 |
| GET /api/wallet/balance（无 Token） | 401 Unauthorized |
| GET /api/wallet/balance（非 User 角色） | 403 Forbidden |
| POST /api/games/launch（正常参数，含 platType/gameType） | 200 + gameUrl |
| POST /api/games/launch（首次用户，自动创建玩家映射） | 200 + gameUrl + VendorPlayerMapping 已创建 |
| POST /api/games/launch（余额不足） | 400 + 错误信息 |
| POST /api/games/launch（无效 gameCode） | 400 |
| POST /api/vendor/{code}/callback（有效签名） | 200 |
| POST /api/vendor/{code}/callback（无效签名） | 401 |
| POST /api/admin/wallets/{id}/deposit（Admin 角色） | 200 |
| POST /api/admin/wallets/{id}/deposit（User 角色） | 403 |
| GET /api/agent/users（Agent 查自己的下级） | 200 + 仅下级数据 |
| GET /api/agent/users（Agent 查别人的下级） | 403 或空 |
| GET /api/agent/users/{id}（Agent 查非自己下级的用户详情） | 403 |
| GET /api/agent/reports/summary（Agent 统计） | 200 + 仅自己下级数据 |
| POST /api/games/launch（已有活跃订单，单会话） | 400 + "已有进行中的游戏" |
| POST /api/games/launch（金额超出限额） | 400 + "金额超出限额" |
| POST /api/admin/orders/{id}/force-exit（Confirmed 订单） | 200 + 一键回收结果 |
| POST /api/admin/orders/{id}/force-exit（非 Confirmed 订单） | 400 |
| GET /api/games/bet-records（User） | 200 + 个人投注记录 |
| GET /api/admin/bet-records（Admin） | 200 + 全局投注记录 |
| GET /api/agent/users/{id}/bet-records（Agent 下级） | 200 + 下级投注记录 |
| GET /api/admin/vendor/balance（Admin） | 200 + 商户余额信息 |
| POST /api/wallet/deposit-request（金额低于最小限额） | 400 |
| POST /api/wallet/withdraw-request（每日累计超限） | 400 |
| GET /api/wallet/deposit-requests（User） | 200 + 个人充值申请列表 |
| GET /api/wallet/withdraw-requests（User） | 200 + 个人提现申请列表 |
| GET /api/wallet/bank-cards（User） | 200 + 用户银行卡列表 |
| POST /api/wallet/bank-cards（正常参数） | 201 + 新银行卡 |
| POST /api/wallet/bank-cards（已有 5 张卡） | 400 + 超出数量限制 |
| DELETE /api/wallet/bank-cards/{id}（本人银行卡） | 204 |
| DELETE /api/wallet/bank-cards/{id}（非本人银行卡） | 403 |
| PUT /api/wallet/bank-cards/{id}/default | 200 |
| GET /api/admin/dashboard/stats（Admin） | 200 + 仪表盘统计数据 |
| GET /api/admin/dashboard/stats（非 Admin） | 403 |
| GET /api/agent/dashboard/stats（Agent） | 200 + 代理仪表盘统计 |
| GET /api/agent/bet-records（Agent） | 200 + 下级投注记录汇总 |

### 4.3 数据库事务测试

| 场景 | 预期 |
|------|------|
| 扣款 + 创建订单在同一事务 | 同时成功或同时回滚 |
| 入账 + 更新订单状态在同一事务 | 同时成功或同时回滚 |
| CAS 版本冲突时事务回滚 | UPDATE 影响行数 = 0，整个事务回滚 |
| 余额一致性校验 | Wallet.Balance = SUM(WalletTransactions.Amount) |
| 冻结余额一致性 | Wallet.FrozenBalance = SUM(Pending WithdrawRequests.Amount) |
| LaunchGame 事务 1 成功后厂商失败 | 订单 Fail + Refund Credit，Balance 恢复 |
| LaunchGame 事务 1 成功后崩溃 | PendingOrderRecovery 自动拾取处理 |

### 4.4 业务流程集成测试

| 场景 | 验证点 |
|------|--------|
| 完整游戏流程（玩家创建 → 转入 → 一键回收 → 结算） | 订单 Settled，余额正确，流水完整，VendorPlayerMapping 存在 |
| 充值审核流程（申请 → 审批 → 入账） | 余额增加，流水存在，AdminActionLog 存在 |
| 提现审核流程（申请 → 冻结 → 审批 → 扣款 → 完成） | 余额冻结 → 扣款，流水存在，最终状态 Completed |
| 提现拒绝流程（申请 → 冻结 → 拒绝 → 解冻） | 余额解冻恢复，FrozenBalance 归零 |
| 回调处理流程（**预留，当前厂商不使用**） | 签名验证，状态正确，余额正确 |
| 订单恢复流程（Pending 超时 → 自动恢复） | Pending 订单被处理 |
| 管理员强制退出流程（force-exit → 转出 → Settle/Cancel） | 余额正确，订单终态，AdminActionLog 存在 |
| 单会话限制流程（有活跃订单 → 再次 LaunchGame） | 第二次被拒绝，第一个订单不受影响 |

---

## 5. E2E 测试场景

### 5.1 用户端关键路径

| 场景 | 步骤 |
|------|------|
| 用户注册登录 | 打开注册页 → 填写信息 → 注册 → 自动登录 → 看到首页 |
| 查看钱包 | 登录 → 点击钱包 → 看到余额和流水 |
| 充值申请 | 登录 → 钱包 → 充值 → 填写金额 → 上传凭证 → 提交 → 确认提示 |
| 游戏启动 | 登录 → 游戏大厅 → 选择平台（AG/PG/PP 等）→ 选择游戏 → 输入金额 → 进入 → 验证新窗口打开 |
| 查看投注记录 | 登录 → 投注记录页 → 验证有数据 → 验证筛选（按平台/时间） |
| 查看订单 | 登录 → 订单列表 → 验证有数据 → 点击查看详情 |
| 银行卡管理 | 登录 → 钱包 → 银行卡管理 → 添加银行卡 → 设为默认 → 删除银行卡 |
| 提现选择银行卡 | 登录 → 钱包 → 提现 → 选择银行卡 → 填写金额 → 提交 |

### 5.2 管理端关键路径

| 场景 | 步骤 |
|------|------|
| 用户管理 | 管理员登录 → 用户列表 → 搜索用户 → 查看详情 → 禁用/启用 |
| 充值审核 | 管理员登录 → 充值审核 → 查看申请 → 通过 → 验证用户余额变动 |
| 手动调整 | 管理员登录 → 钱包管理 → 选择用户 → 调整金额 → 验证流水 |
| 查看日志 | 管理员登录 → 各日志页面 → 验证数据加载 |
| 查看投注记录 | 管理员登录 → 投注记录页 → 验证全局投注数据 → 按用户/平台筛选 |
| 查看商户余额 | 管理员登录 → 商户余额页 → 验证厂商余额显示 |

### 5.3 完整业务链路测试

**链路 A：用户充值审批 + 游戏 + 管理员干预（对照 CLAUDE.md 7 步）**

```
1. [CLAUDE.md 步骤1] user1 登录
2. [CLAUDE.md 步骤2] user1 提交充值申请 ¥1,000（上传凭证）
3. 管理员登录，审批充值申请 → 通过
4. 验证 user1 余额 = ¥1,000
5. [CLAUDE.md 步骤3] user1 选择平台（如 AG）→ 选择游戏 → 进入游戏（转入 ¥500）→ 验证游戏 URL 返回
6. 验证 user1 余额 = ¥500
7. 验证 VendorPlayerMapping 已创建
8. user1 退出游戏 → 一键回收（TransferAll）→ 返回 ¥600
9. [CLAUDE.md 步骤5] 验证 user1 最终余额 = ¥1,100（资金最终一致）
10. 验证订单状态 = Settled
11. 验证 WalletTransactions 流水完整（3 条：Deposit +1000、GameTransferIn -500、GameTransferOut +600）
12. [CLAUDE.md 步骤6] 管理员手动调整 user1 余额 +100（AdminAdjust）→ 验证余额 = ¥1,200
13. [CLAUDE.md 步骤6] 管理员修改一个 Confirmed 订单状态为 Cancelled → 验证状态变更
14. [CLAUDE.md 步骤7] 验证所有操作产生 AdminActionLog 审计日志
15. 验证 WalletTransactions 新增 AdminAdjust 流水
```

**链路 B：Pending 订单恢复验证**

```
1. 管理员为 user2 手动充值 ¥500
2. user2 进入游戏（转入 ¥200）→ 模拟转账返回非明确结果（非 10000/10005）→ 订单停留 Pending
3. 等待后台恢复任务（PendingOrderRecoveryService）自动拾取
4. 恢复任务调用 transferStatus 查询 → 确认转账成功 → 订单 Confirmed
5. 验证 VendorQueryLogs 记录存在
6. user2 退出游戏 → 一键回收 → 订单 Settled
```

**链路 C：投注记录同步验证**

```
1. user3 进入游戏并完成若干投注
2. 等待 BetRecordSyncService 定时同步（2 分钟间隔）
3. 验证 GameBetRecords 表有数据
4. user3 访问投注记录页 → 验证数据正确
5. 管理员查看全局投注记录 → 验证包含 user3 的投注数据
```

---

## 6. 特殊场景测试

### 6.1 幂等性测试

| 场景 | 测试方法 | 预期 |
|------|----------|------|
| 重复回调（相同 ExternalOrderId）（**预留**） | 发送两次相同回调 | 第二次跳过，余额只增加一次 |
| 重复 transferStatus 查询触发结算 | 并发两次 transferStatus 返回 success | 只执行一次结算（CAS 保证） |
| 重复一键回收同一订单 | 并发两次 ExitGame | 第一次成功，第二次订单已非 Confirmed 拒绝 |
| 重复 ExternalOrderId 创建 | 数据库 INSERT 相同 ExternalOrderId | 唯一索引拒绝 |
| 重复 InternalOrderId（orderId 32 字符） | 数据库 INSERT 相同 InternalOrderId | 唯一索引拒绝 |
| 重复审批同一充值申请 | 连续两次 Approve | 第一次成功，第二次幂等返回/异常 |

### 6.2 并发测试

| 场景 | 测试方法 | 预期 |
|------|----------|------|
| 10 次并发结算同一订单 | Task.WhenAll 10 个线程 | 仅 1 次成功，其余 CAS 冲突 |
| 并发扣款（余额刚好够一次） | 余额 100，10 线程各扣 100 | 仅 1 次成功 |
| 并发充值同一钱包 | 10 线程各充 100 | 全部成功（CAS 重试），最终余额 +1000 |
| 并发提现申请（余额仅够一次） | 余额 100，5 线程各申请 100 | 仅 1 次成功冻结，其余可用余额不足 |

### 6.3 异常恢复测试

| 场景 | 测试方法 | 预期 |
|------|----------|------|
| 转入成功但订单更新前崩溃 | 创建 Pending 订单 + 模拟中断 | 恢复任务查询厂商状态并更新 |
| 转出成功但入账前崩溃 | 模拟转出后中断 | 恢复任务完成入账 |
| 恢复任务重复执行 | 连续运行两次恢复 | 幂等，无重复操作 |

### 6.4 安全测试

| 场景 | 测试方法 | 预期 |
|------|----------|------|
| 伪造回调签名（**预留，当前厂商不使用**） | 发送错误签名的回调 | 401 + VendorCallbackLog(IsVerified=false) |
| 过期时间戳回调（**预留**） | 时间戳超过 5 分钟 | 拒绝 |
| 厂商 API 签名篡改 | 请求头 sign 字段被修改 | 厂商返回 10404（签名校验失败） |
| 越权访问管理端 API | User Token 调用 /api/admin/* | 403 |
| 越权查看其他用户钱包 | User A 查 User B 的余额 | 403 或 404 |
| SQL 注入尝试 | 在搜索/筛选参数注入 SQL | 参数化查询防护，无异常 |

---

## 7. 测试数据策略

| 环境 | 数据源 | 隔离方式 |
|------|--------|----------|
| 单元测试 | 内存构造 + Bogus | 无依赖 |
| 集成测试 | Testcontainers PostgreSQL | 每测试事务回滚 或 独立 schema |
| E2E 测试 | 独立测试 Supabase 项目 | 每次运行前 seed 数据 |

### 7.1 测试数据 Seed

E2E 测试前准备的标准数据集：
- 1 个 Admin 用户
- 1 个 Agent 用户
- 3 个 User 用户（其中 2 个属于 Agent）
- 每个 User 有钱包（余额不同）
- 预置若干订单（不同状态）
- 预置充值/提现申请（不同状态）
- 每个 User 有 VendorPlayerMapping（已创建状态）
- 预置若干 GameBetRecords（不同平台）
- 每个 User 有 1-2 张 UserBankCard（其中 1 张为默认）

---

## 8. 前端测试策略

### 8.1 工具链

| 工具 | 用途 |
|------|------|
| Vitest | 测试运行器（兼容 Jest API，速度更快） |
| React Testing Library | 组件测试（用户视角） |
| MSW (Mock Service Worker) | API Mock |
| Playwright | E2E 测试（见 §5） |

### 8.2 测试层次

| 层 | 覆盖范围 | 示例 |
|----|----------|------|
| Store 测试 | Zustand Store 逻辑 | useWalletStore: 余额更新、SignalR 事件处理 |
| 组件测试 | 独立组件渲染和交互 | AmountInput: 格式化、最小/最大值校验 |
| Hook 测试 | 自定义 Hook 逻辑 | useSignalR: 连接、断线重连、消息处理 |
| 页面测试 | 页面渲染和 API 集成 | 游戏大厅: 加载、空状态、错误状态 |

### 8.3 关键前端测试场景

| 场景 | 预期 |
|------|------|
| AmountInput 输入非数字 | 过滤非法字符 |
| AmountInput 超出限额 | 显示错误提示 |
| GameCard 点击"进入" | 调用正确 API |
| WalletSummary SignalR 更新 | 余额实时更新 |
| 登录 Token 过期 | 自动跳转登录页 |
| 游戏进行中检测窗口关闭 | 自动触发退出 API |
| BankCardList 加载 | 正确渲染银行卡列表 |
| AddBankCardDialog 提交 | 验证表单校验（卡号长度、必填字段） |
| BankCardItem 设为默认 | 调用正确 API，UI 更新 |
| BankCardItem 删除确认 | 显示确认弹窗，确认后删除 |
| 提现页银行卡选择 | 无银行卡时提示先添加 |

---

## 9. 约束引用

- 模块功能 → 参见 `01-概要功能设计-overview.md`
- 接口定义 → 参见 `03-详细设计-detailed_design.md`
- CI 集成 → 参见 `06-CI-CD设计-ci_cd_design.md`
