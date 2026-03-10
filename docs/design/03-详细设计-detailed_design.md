# 详细设计

> 文档编号：03-详细设计-detailed_design.md | 版本：1.5 | 日期：2026-03-04
> 依赖文档：overview.md（模块定义）, architecture.md（分层架构）

## 1. Domain 层实体设计

### 1.1 UserProfile

```csharp
public class UserProfile
{
    public Guid Id { get; private set; }             // = Supabase auth.users.id
    public string Username { get; private set; }
    public string? Nickname { get; private set; }
    public UserRole Role { get; private set; }        // User, Agent, Admin
    public Guid? AgentId { get; private set; }        // 上级代理 ID（仅 User 角色）
    public UserStatus Status { get; private set; }    // Active, Disabled
    public DateTime CreatedAt { get; private set; }
    public DateTime UpdatedAt { get; private set; }

    // 行为方法
    public void Disable();
    public void Enable();
    public void UpdateNickname(string nickname);
    public void AssignAgent(Guid agentId);
}
```

```csharp
public enum UserRole { User = 0, Agent = 1, Admin = 2 }
public enum UserStatus { Active = 0, Disabled = 1 }
```

### 1.2 Wallet

```csharp
public class Wallet
{
    public Guid Id { get; private set; }
    public Guid UserId { get; private set; }
    public decimal Balance { get; private set; }        // 必须 decimal，禁止 float/double
    public decimal FrozenBalance { get; private set; }  // 冻结余额（提现申请冻结）
    public int Version { get; private set; }             // CAS 乐观锁版本号
    public DateTime CreatedAt { get; private set; }
    public DateTime UpdatedAt { get; private set; }

    /// <summary>可用余额 = Balance - FrozenBalance</summary>
    public decimal AvailableBalance => Balance - FrozenBalance;

    /// <summary>入账：充值、游戏转出、退款、管理员调整（正）</summary>
    public WalletTransaction Credit(
        decimal amount,
        TransactionType type,
        string description,
        DateTime now,
        Guid? orderId = null);

    /// <summary>出账：提现、游戏转入、管理员调整（负）</summary>
    public WalletTransaction Debit(
        decimal amount,
        TransactionType type,
        string description,
        DateTime now,
        Guid? orderId = null);

    /// <summary>冻结余额（提现申请时调用）</summary>
    public void Freeze(decimal amount, DateTime now);

    /// <summary>解冻余额（提现拒绝/取消时调用）</summary>
    public void Unfreeze(decimal amount, DateTime now);

    /// <summary>确认冻结扣款（提现审批通过时调用，从 Balance 和 FrozenBalance 同时扣减）</summary>
    public WalletTransaction ConfirmFrozenDebit(
        decimal amount,
        TransactionType type,
        string description,
        DateTime now,
        Guid? orderId = null);
}
```

**Credit 方法逻辑**：
1. `WalletRules.ValidateAmount(amount)` — 金额必须 > 0
2. 记录 balanceBefore = Balance
3. Balance += amount
4. Version++
5. UpdatedAt = now（由调用方传入本地时间，Domain 层无外部依赖）
6. 返回新建的 WalletTransaction

**Debit 方法逻辑**：
1. `WalletRules.ValidateAmount(amount)` — 金额必须 > 0
2. `WalletRules.ValidateSufficientBalance(AvailableBalance, amount)` — **可用余额**充足
3. 记录 balanceBefore = Balance
4. Balance -= amount
5. Version++
6. UpdatedAt = now（由调用方传入本地时间）
7. 返回新建的 WalletTransaction（Amount 为负值）

**Freeze 方法逻辑**：
1. `WalletRules.ValidateAmount(amount)` — 金额必须 > 0
2. `WalletRules.ValidateSufficientBalance(AvailableBalance, amount)` — 可用余额充足
3. FrozenBalance += amount
4. Version++
5. UpdatedAt = now

**Unfreeze 方法逻辑**：
1. `WalletRules.ValidateAmount(amount)` — 金额必须 > 0
2. 验证 FrozenBalance >= amount — 冻结余额充足
3. FrozenBalance -= amount
4. Version++
5. UpdatedAt = now

**ConfirmFrozenDebit 方法逻辑**：
1. `WalletRules.ValidateAmount(amount)` — 金额必须 > 0
2. 验证 FrozenBalance >= amount — 冻结余额充足
3. 记录 balanceBefore = Balance
4. Balance -= amount
5. FrozenBalance -= amount
6. Version++
7. UpdatedAt = now
8. 返回新建的 WalletTransaction（Amount 为负值）

### 1.3 WalletTransaction

```csharp
public class WalletTransaction
{
    public Guid Id { get; private set; }
    public Guid WalletId { get; private set; }
    public TransactionType Type { get; private set; }
    public decimal Amount { get; private set; }          // 正 = 入账, 负 = 出账
    public decimal BalanceBefore { get; private set; }
    public decimal BalanceAfter { get; private set; }
    public string Description { get; private set; }
    public Guid? OrderId { get; private set; }
    public string? InternalOrderId { get; private set; }
    public DateTime CreatedAt { get; private set; }
    // 全部 private set — 创建后不可修改
}
```

```csharp
public enum TransactionType
{
    Deposit = 0,           // 充值
    Withdrawal = 1,        // 提现
    GameTransferIn = 2,    // 游戏转入（从平台扣款到厂商）
    GameTransferOut = 3,   // 游戏转出（从厂商回到平台）
    AdminAdjust = 4,       // 管理员调整
    Refund = 5             // 退款
}
```

### 1.4 GameOrder

```csharp
public class GameOrder
{
    public Guid Id { get; private set; }
    public Guid UserId { get; private set; }
    public string VendorCode { get; private set; }
    public string PlatType { get; private set; }             // 游戏子平台标识（如 "ag", "pg", "pp"）
    public string GameCode { get; private set; }
    public string InternalOrderId { get; private set; }      // UNIQUE 唯一索引（32 位字母数字，兼容厂商 orderId 格式）
    public string? ExternalOrderId { get; private set; }     // UNIQUE 唯一索引
    public decimal TransferInAmount { get; private set; }
    public decimal? TransferOutAmount { get; private set; }
    public OrderStatus Status { get; private set; }
    public string? GameUrl { get; private set; }
    public string? ErrorMessage { get; private set; }
    public string? CancelReason { get; private set; }        // 取消原因（与 ErrorMessage 语义不同）
    public DateTime CreatedAt { get; private set; }
    public DateTime UpdatedAt { get; private set; }

    // 状态流转方法（内部调用 OrderStateMachine.ValidateTransition）
    public void Confirm(string externalOrderId, string gameUrl, DateTime now);
    public void Settle(decimal transferOutAmount, DateTime now);
    public void Cancel(string? reason, DateTime now);
    public void Fail(string errorMessage, DateTime now);
}
```

**Cancel 触发场景**（从 `Confirmed` 状态触发）：
- **用户退出时一键回收金额为 0**：ExitGame 时 transferAll 回收金额为 0，TransferOutAmount = 0，直接 Cancel
- **管理员强制退出时厂商余额为 0**：ForceExit 时一键回收金额为 0，直接 Cancel

> **⚠️ 资金安全约束**：`Confirmed → Cancelled` 仅允许在厂商侧余额为 0 的前提下触发。管理员**不得**直接将有余额的 Confirmed 订单标记为 Cancelled（会导致资金丢失）。管理员若需取消 Confirmed 订单，必须先通过 `AdminForceExitGameAsync`（一键回收 TransferAll）完成资金回收后再自动流转。`AdminUpdateStatusAsync` 在 Confirmed 状态下仅接受 reason 备注，不允许直接变更为 Cancelled。

Cancel 时必须：
1. 设置 `CancelReason` 字段（必填）
2. 如果 Confirmed 订单有已转入金额但厂商余额为 0，不执行退款（资金已在厂商侧消耗）
3. 如果需要退款，走独立的 Refund 流程（管理员手动触发）

```csharp
public enum OrderStatus
{
    Pending = 0,
    Confirmed = 1,
    Settled = 2,
    Cancelled = 3,
    Failed = 4
}
```

### 1.5 OrderStateMachine

```csharp
public static class OrderStateMachine
{
    // 合法状态转换表
    private static readonly Dictionary<OrderStatus, HashSet<OrderStatus>> AllowedTransitions = new()
    {
        { OrderStatus.Pending,   new() { OrderStatus.Confirmed, OrderStatus.Failed } },
        { OrderStatus.Confirmed, new() { OrderStatus.Settled, OrderStatus.Cancelled } },
        { OrderStatus.Settled,   new() { } },    // 终态
        { OrderStatus.Cancelled, new() { } },    // 终态
        { OrderStatus.Failed,    new() { } }     // 终态
    };

    public static void ValidateTransition(OrderStatus from, OrderStatus to)
    {
        if (!AllowedTransitions.ContainsKey(from) || !AllowedTransitions[from].Contains(to))
            throw new InvalidStateTransitionException(from, to);
    }
}
```

### 1.6 WalletRules

```csharp
public static class WalletRules
{
    public static void ValidateAmount(decimal amount)
    {
        if (amount <= 0)
            throw new DomainException("Amount must be greater than zero.");
    }

    /// <summary>验证余额充足（传入 AvailableBalance 而非 Balance）</summary>
    public static void ValidateSufficientBalance(decimal availableBalance, decimal amount)
    {
        if (availableBalance < amount)
            throw new InsufficientBalanceException(availableBalance, amount);
    }

    /// <summary>验证冻结余额充足</summary>
    public static void ValidateSufficientFrozenBalance(decimal frozenBalance, decimal amount)
    {
        if (frozenBalance < amount)
            throw new DomainException($"Frozen balance {frozenBalance} is less than requested amount {amount}.");
    }
}

/// <summary>金额限额配置（Application 层使用，可通过 appsettings 配置）</summary>
public class AmountLimitOptions
{
    public decimal MinDepositAmount { get; set; } = 10m;            // 最小单笔充值
    public decimal MaxDepositAmount { get; set; } = 50_000m;        // 最大单笔充值
    public decimal MinWithdrawAmount { get; set; } = 10m;           // 最小单笔提现
    public decimal MaxWithdrawAmount { get; set; } = 50_000m;       // 最大单笔提现
    public decimal DailyWithdrawLimit { get; set; } = 100_000m;     // 每日累计提现上限
    public decimal MinGameTransferAmount { get; set; } = 1m;        // 最小游戏转入金额
}
```

### 1.7 VendorPlayerMapping

```csharp
public class VendorPlayerMapping
{
    public Guid Id { get; private set; }
    public Guid UserId { get; private set; }
    public string VendorCode { get; private set; }          // 厂商标识（如 "apibet"）
    public string VendorPlayerId { get; private set; }       // 厂商侧玩家 ID（5-11 位小写字母+数字）
    public string Currency { get; private set; }              // 厂商侧货币（如 "CNY"）
    public bool IsCreated { get; private set; }               // 是否已在厂商侧创建成功
    public DateTime CreatedAt { get; private set; }
    public DateTime UpdatedAt { get; private set; }

    public static VendorPlayerMapping Create(Guid userId, string vendorCode, string vendorPlayerId, string currency, DateTime now)
    {
        return new VendorPlayerMapping
        {
            Id = Guid.NewGuid(),
            UserId = userId,
            VendorCode = vendorCode,
            VendorPlayerId = vendorPlayerId,
            Currency = currency,
            IsCreated = false,
            CreatedAt = now,
            UpdatedAt = now
        };
    }

    public void MarkCreated(DateTime now)
    {
        IsCreated = true;
        UpdatedAt = now;
    }
}
```

**VendorPlayerId 生成规则**：
- 格式：`{sn前缀}{userId哈希}`，总长度 5-11 位，小写字母+数字
- 示例：sn="ab"，则生成 `ab` + SHA256(userId).Substring(0,8).ToLower() → `ab3f7e2a1c`
- 通过 `VendorPlayerMappings` 表的唯一索引保证不重复
- **碰撞处理**：如果唯一索引 `(VendorCode, VendorPlayerId)` 冲突（极低概率），截取更多哈希位（9 位、10 位...直至 11 位上限），最多重试 3 次。3 次后仍然冲突则抛出 `DomainException("VendorPlayerId generation failed after max retries")` 并触发告警

### 1.8 GameBetRecord

```csharp
public class GameBetRecord
{
    public Guid Id { get; private set; }
    public Guid UserId { get; private set; }
    public string VendorPlayerId { get; private set; }       // 厂商侧玩家 ID
    public string PlatType { get; private set; }              // 游戏平台（如 "ag", "pg"）
    public string Currency { get; private set; }
    public string GameType { get; private set; }              // 1=视讯, 2=老虎机, 3=彩票, 4=体育, 5=电竞, 6=捕猎, 7=棋牌
    public string GameName { get; private set; }
    public string Round { get; private set; }                 // 局号
    public string? Table { get; private set; }                // 桌号
    public string? Seat { get; private set; }                 // 座号
    public decimal BetAmount { get; private set; }            // 投注金额
    public decimal ValidAmount { get; private set; }          // 有效投注金额
    public decimal SettledAmount { get; private set; }        // 输赢金额
    public string? BetContent { get; private set; }           // 投注内容
    public int Status { get; private set; }                   // 0=未完成, 1=已完成, 2=已取消, 3=已撤单
    public string VendorBetOrderId { get; private set; }           // 厂商订单 ID（唯一索引，幂等用）
    public DateTime BetTime { get; private set; }             // 投注时间（厂商返回，本地时区）
    public DateTime LastUpdateTime { get; private set; }      // 最后更新时间（厂商返回，本地时区）
    public DateTime SyncedAt { get; private set; }            // 本地同步时间
}
```

### 1.9 DepositRequest / WithdrawRequest

```csharp
public class DepositRequest
{
    public Guid Id { get; private set; }
    public Guid UserId { get; private set; }
    public decimal Amount { get; private set; }
    public string? Proof { get; private set; }                  // 转账凭证 URL
    public DepositRequestStatus Status { get; private set; }    // Pending, Approved, Rejected
    public Guid? ReviewedBy { get; private set; }
    public string? ReviewNote { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime UpdatedAt { get; private set; }

    public void Approve(Guid adminId, DateTime now, string? note = null);
    public void Reject(Guid adminId, DateTime now, string? note = null);
}

public class WithdrawRequest
{
    public Guid Id { get; private set; }
    public Guid UserId { get; private set; }
    public decimal Amount { get; private set; }
    public string BankInfo { get; private set; }                  // JSON: 银行卡/提现信息
    public WithdrawRequestStatus Status { get; private set; }     // Pending, Approved, Rejected, Completed
    public Guid? ReviewedBy { get; private set; }
    public string? ReviewNote { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime UpdatedAt { get; private set; }

    public void Approve(Guid adminId, DateTime now, string? note = null);
    public void Reject(Guid adminId, DateTime now, string? note = null);
    public void Complete(DateTime now);   // 管理员确认提现已完成（线下打款后标记）
}

public enum DepositRequestStatus { Pending = 0, Approved = 1, Rejected = 2 }
public enum WithdrawRequestStatus { Pending = 0, Approved = 1, Rejected = 2, Completed = 3 }
public enum AdminActionLogLevel { Info = 0, Warning = 1, Error = 2 }
```

### 1.11 UserBankCard

```csharp
public class UserBankCard
{
    public Guid Id { get; private set; }
    public Guid UserId { get; private set; }
    public string BankName { get; private set; }          // 银行名称
    public string CardNumber { get; private set; }         // 银行卡号（脱敏存储：仅存后4位明文，完整卡号加密存储）
    public string CardHolderName { get; private set; }     // 持卡人姓名
    public string? BranchName { get; private set; }        // 开户支行（可选）
    public bool IsDefault { get; private set; }            // 是否默认卡
    public DateTime CreatedAt { get; private set; }
    public DateTime UpdatedAt { get; private set; }

    public static UserBankCard Create(Guid userId, string bankName, string cardNumber,
        string cardHolderName, string? branchName, bool isDefault, DateTime now);
    public void SetDefault(DateTime now);
    public void ClearDefault(DateTime now);
}
```

**银行卡管理规则**：
- 每个用户最多绑定 5 张银行卡
- 提现时可选择已绑定的银行卡
- 银行卡号在前端显示时脱敏（仅显示后 4 位）
- 绑定时需验证卡号格式（16-19 位数字）

### 1.12 AdminActionLog

```csharp
public class AdminActionLog
{
    public Guid Id { get; private set; }
    public Guid? AdminId { get; private set; }            // NULL = 系统自动操作
    public string Action { get; private set; }             // 操作类型（如 "AdminDeposit", "ForceExitGame", "BalanceInconsistency"）
    public string TargetType { get; private set; }         // 目标类型（如 "Wallet", "Order", "System"）
    public Guid TargetId { get; private set; }
    public object Details { get; private set; }            // JSONB 详情
    public AdminActionLogLevel Level { get; private set; } // 0=Info, 1=Warning, 2=Error
    public string? IdempotencyKey { get; private set; }    // 幂等键（前端生成 UUID）
    public DateTime CreatedAt { get; private set; }
}
```

### 1.13 VendorCallbackLog

```csharp
public class VendorCallbackLog
{
    public Guid Id { get; private set; }
    public string VendorCode { get; private set; }
    public Guid? OrderId { get; private set; }
    public object RawPayload { get; private set; }         // JSONB 原始请求体
    public string? Signature { get; private set; }
    public bool IsVerified { get; private set; }
    public string? ProcessResult { get; private set; }
    public string? ErrorMessage { get; private set; }
    public DateTime CreatedAt { get; private set; }
}
```

### 1.14 VendorQueryLog

```csharp
public class VendorQueryLog
{
    public Guid Id { get; private set; }
    public string VendorCode { get; private set; }
    public Guid OrderId { get; private set; }
    public object? QueryResult { get; private set; }       // JSONB 查询结果
    public int Attempt { get; private set; }               // 轮询次数
    public bool IsSuccess { get; private set; }
    public string? ErrorMessage { get; private set; }
    public DateTime CreatedAt { get; private set; }
}
```

**提现冻结流程**：
1. 用户提交提现申请 → `Wallet.Freeze(amount)` 冻结余额 → 创建 `WithdrawRequest(Pending)`
2. 管理员审批通过 → `Wallet.ConfirmFrozenDebit(amount)` 从冻结余额扣款 → `WithdrawRequest.Approve()`
3. 管理员审批拒绝 → `Wallet.Unfreeze(amount)` 解冻余额 → `WithdrawRequest.Reject()`
4. 管理员确认打款 → `WithdrawRequest.Complete()`

### 1.10 DepositRequestStateMachine / WithdrawRequestStateMachine（充值/提现状态机）

```csharp
public static class DepositRequestStateMachine
{
    private static readonly Dictionary<DepositRequestStatus, HashSet<DepositRequestStatus>> AllowedTransitions = new()
    {
        { DepositRequestStatus.Pending,  new() { DepositRequestStatus.Approved, DepositRequestStatus.Rejected } },
        { DepositRequestStatus.Approved, new() { } },    // 终态
        { DepositRequestStatus.Rejected, new() { } }     // 终态
    };

    public static void ValidateTransition(DepositRequestStatus from, DepositRequestStatus to)
    {
        if (!AllowedTransitions.ContainsKey(from) || !AllowedTransitions[from].Contains(to))
            throw new InvalidStateTransitionException($"DepositRequest: {from} → {to}");
    }
}

public static class WithdrawRequestStateMachine
{
    private static readonly Dictionary<WithdrawRequestStatus, HashSet<WithdrawRequestStatus>> AllowedTransitions = new()
    {
        { WithdrawRequestStatus.Pending,   new() { WithdrawRequestStatus.Approved, WithdrawRequestStatus.Rejected } },
        { WithdrawRequestStatus.Approved,  new() { WithdrawRequestStatus.Completed } },
        { WithdrawRequestStatus.Rejected,  new() { } },    // 终态
        { WithdrawRequestStatus.Completed, new() { } }     // 终态
    };

    public static void ValidateTransition(WithdrawRequestStatus from, WithdrawRequestStatus to)
    {
        if (!AllowedTransitions.ContainsKey(from) || !AllowedTransitions[from].Contains(to))
            throw new InvalidStateTransitionException($"WithdrawRequest: {from} → {to}");
    }
}
```

### 1.15 Domain 层接口

```csharp
public interface IWalletRepository
{
    Task<Wallet?> GetByUserIdAsync(Guid userId);
    Task<Wallet?> GetByIdAsync(Guid walletId);
    Task AddAsync(Wallet wallet);
    Task<bool> UpdateWithCasAsync(Wallet wallet, int expectedVersion);
    Task AddTransactionAsync(WalletTransaction transaction);
    Task<IReadOnlyList<WalletTransaction>> GetTransactionsAsync(Guid walletId, int page, int pageSize);
    Task<decimal> GetTransactionsSumAsync(Guid walletId);
}

public interface IUserProfileRepository
{
    Task<UserProfile?> GetByIdAsync(Guid userId);
    Task AddAsync(UserProfile profile);
    Task UpdateAsync(UserProfile profile);
    Task<IReadOnlyList<UserProfile>> GetFilteredAsync(string? filter, int page, int pageSize);
    Task<IReadOnlyList<UserProfile>> GetByAgentIdAsync(Guid agentId, int page, int pageSize);
}

public interface IOrderRepository
{
    Task<GameOrder?> GetByIdAsync(Guid orderId);
    Task<GameOrder?> GetByInternalOrderIdAsync(string internalOrderId);
    Task<GameOrder?> GetByExternalOrderIdAsync(string externalOrderId);
    Task AddAsync(GameOrder order);
    Task UpdateAsync(GameOrder order);
    Task<IReadOnlyList<GameOrder>> GetPendingOrdersAsync(TimeSpan olderThan);
    Task<IReadOnlyList<GameOrder>> GetConfirmedOrdersAsync(TimeSpan olderThan);
    Task<IReadOnlyList<GameOrder>> GetByUserIdAsync(Guid userId, int page, int pageSize);
    Task<bool> HasActiveOrderAsync(Guid userId);  // 单会话检查：是否有 Pending/Confirmed 订单
}

public interface IVendorPlayerMappingRepository
{
    Task<VendorPlayerMapping?> GetAsync(Guid userId, string vendorCode);
    Task AddAsync(VendorPlayerMapping mapping);
    Task UpdateAsync(VendorPlayerMapping mapping);
}

public interface IGameBetRecordRepository
{
    Task UpsertBatchAsync(IReadOnlyList<GameBetRecord> records);  // 按 VendorBetOrderId 幂等插入/更新
    Task<IReadOnlyList<GameBetRecord>> GetByUserIdAsync(Guid userId, int page, int pageSize);
    Task<IReadOnlyList<GameBetRecord>> GetPagedAsync(int page, int pageSize, string? platType = null, string? vendorPlayerId = null);
}

public interface IDepositRequestRepository
{
    Task<DepositRequest?> GetByIdAsync(Guid requestId);
    Task AddAsync(DepositRequest request);
    Task UpdateAsync(DepositRequest request);
    Task<IReadOnlyList<DepositRequest>> GetByStatusAsync(DepositRequestStatus status, int page, int pageSize);
    Task<IReadOnlyList<DepositRequest>> GetByUserIdAsync(Guid userId, int page, int pageSize);
}

public interface IWithdrawRequestRepository
{
    Task<WithdrawRequest?> GetByIdAsync(Guid requestId);
    Task AddAsync(WithdrawRequest request);
    Task UpdateAsync(WithdrawRequest request);
    Task<IReadOnlyList<WithdrawRequest>> GetByStatusAsync(WithdrawRequestStatus status, int page, int pageSize);
    Task<IReadOnlyList<WithdrawRequest>> GetByUserIdAsync(Guid userId, int page, int pageSize);
}

public interface IUserBankCardRepository
{
    Task<UserBankCard?> GetByIdAsync(Guid cardId);
    Task<IReadOnlyList<UserBankCard>> GetByUserIdAsync(Guid userId);
    Task AddAsync(UserBankCard card);
    Task UpdateAsync(UserBankCard card);
    Task DeleteAsync(Guid cardId);
    Task<int> GetCountByUserIdAsync(Guid userId);
}

public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct = default);
    Task BeginTransactionAsync();
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
}
```

```csharp
// 日志表 Repository 接口

public interface IVendorCallbackLogRepository
{
    Task AddAsync(VendorCallbackLog log);
    Task<IReadOnlyList<VendorCallbackLog>> GetByOrderIdAsync(Guid orderId);
    Task<IReadOnlyList<VendorCallbackLog>> GetPagedAsync(int page, int pageSize, string? vendorCode = null);
}

public interface IVendorQueryLogRepository
{
    Task AddAsync(VendorQueryLog log);
    Task<int> GetAttemptCountAsync(Guid orderId);
    Task<IReadOnlyList<VendorQueryLog>> GetByOrderIdAsync(Guid orderId);
    Task<IReadOnlyList<VendorQueryLog>> GetPagedAsync(int page, int pageSize);
}

public interface IAdminActionLogRepository
{
    Task AddAsync(AdminActionLog log);
    Task<AdminActionLog?> GetByIdempotencyKeyAsync(string idempotencyKey);
    Task<IReadOnlyList<AdminActionLog>> GetPagedAsync(int page, int pageSize, int? level = null);
    Task<IReadOnlyList<AdminActionLog>> GetByTargetAsync(string targetType, Guid targetId);
}
```

```csharp
// ITimeProvider 接口（Application 层获取当前本地时间，Domain 层通过参数注入）

public interface ITimeProvider
{
    /// <summary>获取当前本地时间（禁止 UTC）</summary>
    DateTime Now { get; }
}

// 默认实现
public class SystemTimeProvider : ITimeProvider
{
    public DateTime Now => DateTime.Now;
}
```

---

## 2. Application 层用例设计

### 2.1 钱包用例（IWalletService）

| 方法 | 参数 | 返回 | 说明 |
|------|------|------|------|
| GetBalanceAsync | userId | BalanceDto | 查询余额 |
| GetTransactionsAsync | userId, page, pageSize | PagedList\<TransactionDto\> | 查询流水 |
| AdminDepositAsync | userId, amount, adminId, description | TransactionDto | 管理员充值（CAS + 流水 + 日志） |
| AdminWithdrawAsync | userId, amount, adminId, description | TransactionDto | 管理员扣款 |
| AdminAdjustAsync | userId, amount, type, adminId, description, idempotencyKey | TransactionDto | 管理员调整（type=Credit/Debit，amount 必须为正数） |
| RequestDepositAsync | userId, amount, proof | DepositRequestDto | 用户充值申请（**金额限额校验**） |
| ApproveDepositAsync | requestId, adminId, note | void | 审批充值（自动入账） |
| RejectDepositAsync | requestId, adminId, note | void | 拒绝充值 |
| RequestWithdrawalAsync | userId, amount, bankInfo | WithdrawRequestDto | 用户提现申请（**金额限额校验 + 每日累计校验 + 冻结余额**） |
| ApproveWithdrawalAsync | requestId, adminId, note | void | 审批提现（**从冻结余额扣款**） |
| RejectWithdrawalAsync | requestId, adminId, note | void | 拒绝提现（**解冻余额**） |
| CompleteWithdrawalAsync | requestId, adminId | void | 确认提现完成（线下打款后） |
| ReconcileAsync | adminId | ReconcileResultDto | 对账校验 |

### 2.2 游戏订单用例（IOrderService）

| 方法 | 参数 | 返回 | 说明 |
|------|------|------|------|
| LaunchGameAsync | userId, vendorCode, platType, gameCode, gameType, amount | LaunchResultDto | **单会话检查** + 玩家映射 + 创建订单 + 扣款 + 转入 + 获取 URL |
| ExitGameAsync | orderId, userId | ExitResultDto | **一键回收（transferAll）** + 入账 + 结算 |

**DTO 定义**：

```csharp
public record LaunchResultDto(
    Guid OrderId,                   // 内部订单 ID
    string GameUrl,                 // 游戏启动 URL（新窗口打开）
    decimal TransferInAmount,       // 实际转入金额
    string VendorPlayerId           // 厂商侧玩家 ID
);

public record ExitResultDto(
    Guid OrderId,
    OrderStatus FinalStatus,        // Settled 或 Cancelled
    decimal TransferOutAmount,      // 一键回收金额（0 = Cancel）
    decimal NewBalance              // 回收后钱包余额
);
```
| HandleCallbackAsync | vendorCode, payload, signature, timestamp | void | 处理厂商回调（**当前厂商不使用，接口预留**） |
| PollPendingOrdersAsync | - | int (处理数量) | 轮询 Pending 订单（transferStatus 查询） |
| GetUserOrdersAsync | userId, page, pageSize | PagedList\<OrderDto\> | 查询用户订单 |
| GetOrderDetailAsync | orderId, userId | OrderDetailDto | 订单详情 |
| AdminUpdateStatusAsync | orderId, newStatus, adminId, reason | void | 管理员修改订单状态 |
| AdminForceExitGameAsync | orderId, adminId | ExitResultDto | 管理员强制退出用户游戏（触发转出流程 + AdminActionLog） |

### 2.3 用户管理用例

| 方法 | 参数 | 返回 | 说明 |
|------|------|------|------|
| SyncProfileAsync | supabaseUserId, username | UserProfileDto | 注册后同步 Profile（**首次同步时自动创建空钱包**） |
| GetProfileAsync | userId | UserProfileDto | 获取用户资料 |
| UpdateProfileAsync | userId, dto | UserProfileDto | 更新资料 |
| ListUsersAsync | filter, page, pageSize | PagedList\<UserDto\> | 管理员查询用户 |
| DisableUserAsync | userId, adminId | void | 禁用用户 |
| EnableUserAsync | userId, adminId | void | 启用用户 |
| GetAgentUsersAsync | agentId, page, pageSize | PagedList\<UserDto\> | 代理查看下级（**MVP 仅支持直接下级**，即 `UserProfile.AgentId = agentId`；不递归子代理的下级） |

### 2.4 用户管理接口（IUserService）

```csharp
public interface IUserService
{
    Task<UserProfileDto> SyncProfileAsync(Guid supabaseUserId, string username);
    Task<UserProfileDto> GetProfileAsync(Guid userId);
    Task<UserProfileDto> UpdateProfileAsync(Guid userId, UpdateProfileDto dto);
    Task<PagedList<UserDto>> ListUsersAsync(string? filter, int page, int pageSize);
    Task DisableUserAsync(Guid userId, Guid adminId);
    Task EnableUserAsync(Guid userId, Guid adminId);
    Task<PagedList<UserDto>> GetAgentUsersAsync(Guid agentId, int page, int pageSize);
}
```

### 2.5 银行卡管理接口（IUserBankCardService）

```csharp
public interface IUserBankCardService
{
    Task<IReadOnlyList<UserBankCardDto>> GetUserCardsAsync(Guid userId);
    Task<UserBankCardDto> AddCardAsync(Guid userId, AddBankCardDto dto);
    Task DeleteCardAsync(Guid userId, Guid cardId);
    Task SetDefaultCardAsync(Guid userId, Guid cardId);
}
```

---

## 3. API 端点设计

### 3.0 统一响应格式

所有平台 API 端点使用统一的响应信封格式：

```csharp
// 成功响应
public record ApiResponse<T>(
    bool Success,        // true
    T? Data,             // 业务数据
    string? Error,       // null
    object? Meta         // 分页元数据（可选）
);

// 失败响应
public record ApiErrorResponse(
    bool Success,        // false
    object? Data,        // null
    string Error,        // 用户友好错误信息
    string? ErrorCode    // 机器可读错误码（可选）
);

// 分页元数据
public record PaginationMeta(
    int Page,
    int PageSize,
    int TotalCount,
    int TotalPages
);
```

**错误码约定**（ErrorCode 字段）：

| ErrorCode | HTTP | 场景 |
|-----------|------|------|
| `ACTIVE_ORDER_EXISTS` | 400 | 已有进行中的游戏订单 |
| `INSUFFICIENT_BALANCE` | 400 | 钱包余额不足 |
| `INVALID_AMOUNT` | 400 | 金额不在限额范围内 |
| `DAILY_LIMIT_EXCEEDED` | 400 | 超过每日提现限额 |
| `INVALID_STATE_TRANSITION` | 400 | 非法状态流转 |
| `CONCURRENCY_CONFLICT` | 409 | CAS 版本冲突（客户端应重试） |
| `VENDOR_UNAVAILABLE` | 503 | 厂商 API 不可达或限频 |
| `VENDOR_ERROR` | 502 | 厂商返回业务错误 |
| `FORBIDDEN` | 403 | 无权限（代理归属验证失败等） |

**Controller 层统一处理**：通过 `ApiResponseFilter`（ActionFilter）统一包装成功结果；通过全局异常中间件统一包装错误。Controller 方法只需 `return Ok(data)` 或抛出业务异常。

### 3.1 认证 API

| 方法 | 路径 | 角色 | 说明 |
|------|------|------|------|
| POST | /api/auth/sync-profile | Authenticated | 登录后同步 UserProfile |
| GET | /api/auth/me | Authenticated | 获取当前用户信息 |

> 注册/登录通过 Supabase Auth 前端 SDK 完成，不经过后端。

### 3.2 钱包 API（User）

| 方法 | 路径 | 角色 | 说明 |
|------|------|------|------|
| GET | /api/wallet/balance | User | 查询余额 |
| GET | /api/wallet/transactions?page=&pageSize= | User | 查询流水 |
| POST | /api/wallet/deposit-request | User | 充值申请 |
| GET | /api/wallet/deposit-requests?page=&pageSize= | User | 查看自己的充值申请记录 |
| POST | /api/wallet/withdraw-request | User | 提现申请 |
| GET | /api/wallet/withdraw-requests?page=&pageSize= | User | 查看自己的提现申请记录 |
| GET | /api/wallet/bank-cards | User | 查看已绑定的银行卡列表 |
| POST | /api/wallet/bank-cards | User | 绑定新银行卡（最多 5 张） |
| DELETE | /api/wallet/bank-cards/{id} | User | 解绑银行卡 |
| PUT | /api/wallet/bank-cards/{id}/default | User | 设置默认银行卡 |

### 3.3 游戏 API（User）

| 方法 | 路径 | 角色 | 说明 |
|------|------|------|------|
| GET | /api/games?platType= | User | 游戏列表（按平台查询，数据源：IVendorAdapter.GetGameListAsync + IMemoryCache） |
| GET | /api/games/platforms | User | 支持的游戏平台列表（platType 列表） |
| POST | /api/games/launch | User | 启动游戏（玩家映射 + 转入 + 获取 URL） |
| POST | /api/games/exit | User | 退出游戏（一键回收） |
| GET | /api/games/demo | User | 试玩游戏 URL（可选） |
| GET | /api/games/bet-records?page=&pageSize= | User | 个人投注记录 |

### 3.4 订单 API（User）

| 方法 | 路径 | 角色 | 说明 |
|------|------|------|------|
| GET | /api/orders?page=&pageSize= | User | 订单列表 |
| GET | /api/orders/{id} | User | 订单详情 |

### 3.5 厂商回调 API（当前厂商不使用，接口预留）

| 方法 | 路径 | 角色 | 说明 |
|------|------|------|------|
| POST | /api/vendor/{vendorCode}/callback | Vendor（签名验证） | 厂商回调入口（**当前接入厂商无回调机制，此端点预留**） |

### 3.6 管理 API（Admin）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/admin/dashboard/stats | 仪表盘统计（总用户数、今日充值/提现/订单数、待审核数、今日新增） |
| GET | /api/admin/users?filter=&page=&pageSize= | 用户列表 |
| GET | /api/admin/users/{id} | 用户详情 |
| POST | /api/admin/users/{id}/suspend | 禁用用户 |
| POST | /api/admin/users/{id}/activate | 启用用户 |
| GET | /api/admin/deposits?page=&pageSize= | 充值申请列表 |
| GET | /api/admin/withdrawals?page=&pageSize= | 提现申请列表 |
| GET | /api/admin/orders?filter=&page=&pageSize= | 所有订单 |
| GET | /api/admin/orders/{id} | 订单详情 |
| POST | /api/admin/orders/{id}/force-exit | 强制退出游戏（仅 Confirmed 订单，触发转出流程） |
| GET | /api/admin/wallets/{userId} | 用户钱包信息 |
| GET | /api/admin/wallets/{userId}/transactions?page=&pageSize= | 用户流水 |
| POST | /api/admin/wallets/{userId}/adjust | 人工调整余额（必须生成流水） |
| POST | /api/admin/deposit-requests/{id}/approve | 审批充值 |
| POST | /api/admin/deposit-requests/{id}/reject | 拒绝充值 |
| POST | /api/admin/withdraw-requests/{id}/approve | 审批提现 |
| POST | /api/admin/withdraw-requests/{id}/reject | 拒绝提现 |
| POST | /api/admin/withdraw-requests/{id}/complete | 确认提现完成（线下打款后标记） |
| GET | /api/admin/bet-records?page=&pageSize=&userId=&platType= | 全局投注记录 |
| GET | /api/admin/logs/vendor-query?page=&pageSize= | 厂商查询日志 |
| GET | /api/admin/logs/admin-action?page=&pageSize= | 管理员操作日志 |
| GET | /api/admin/vendor/balance | 查询商户在厂商的余额 |
| POST | /api/admin/reconciliation/trigger | 触发对账 |
| GET | /api/admin/alerts?page=&pageSize= | 告警列表 |

> **设计约定**：
> - 用户禁用/启用使用 POST（执行操作语义），路径为 `suspend`/`activate`
> - 充值/提现列表查询路径简写为 `deposits`/`withdrawals`
> - 审批/拒绝操作统一使用 POST（执行动作语义）

### 3.7 代理 API（Agent）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/agent/dashboard/stats | 代理仪表盘统计（下级用户数、活跃订单数、今日充值/提现/盈亏） |
| GET | /api/agent/users?page=&pageSize= | 下级用户列表 |
| GET | /api/agent/users/{id} | 下级用户详情 |
| GET | /api/agent/users/{id}/balance | 下级余额 |
| GET | /api/agent/users/{id}/orders | 下级订单 |
| GET | /api/agent/users/{id}/bet-records?page=&pageSize= | 单个下级投注记录 |
| GET | /api/agent/bet-records?page=&pageSize=&platType= | 所有下级投注记录汇总 |
| GET | /api/agent/reports/summary?from=&to= | 汇总报表 |

**Agent 授权规则**：
- 所有 Agent 端点要求 `[Authorize(Roles = "Agent")]`
- **下级归属验证**：每个请求中涉及的 `{id}` 参数（userId）必须满足 `UserProfile.AgentId == 当前 Agent 的 UserId`
- 不满足归属关系的请求返回 `403 Forbidden`
- 汇总报表 `GET /api/agent/reports/summary` 仅统计当前 Agent 的下级数据，SQL 中强制 `WHERE AgentId = @agentId`
- 归属验证建议通过 `AgentAuthorizationFilter` 实现，避免在每个 Controller 重复判断

---

## 4. 关键业务流程

### 4.1 游戏启动流程（LaunchGame）

```
User → Frontend: 选择游戏, 指定转入金额
Frontend → Backend: POST /api/games/launch { vendorCode, platType, gameCode, gameType, amount }
Backend (OrderService):

  ── 前置检查（事务外）──
  0. 检查用户是否有进行中的游戏（IOrderRepository.HasActiveOrderAsync）
     └── 有 Pending/Confirmed 订单 → 拒绝，返回错误 "已有进行中的游戏，请先退出"
  0b. 检查/创建厂商玩家账号
     └── 查询 VendorPlayerMappings，不存在则生成 VendorPlayerId
     └── 如果 IsCreated=false，调用 IVendorAdapter.CreatePlayerAsync
     └── 创建成功后标记 IsCreated=true（10002 账号已存在 → 也标记为成功）

  ── 事务 1（原子操作：扣款 + 创建订单）──
  1. 生成 InternalOrderId（32 位字母数字，GUID 去连字符取前 32 位）
  2. 创建 GameOrder (Status=Pending, TransferInAmount=amount, PlatType=platType)
  3. 钱包 Debit(amount, GameTransferIn) — CAS 更新
  4. 保存 WalletTransaction
  5. 提交事务 1
  ── 事务 1 结束 ──

  ── 事务外（厂商 API 调用，不在数据库事务内）──
  6. 调用 IVendorAdapter.TransferInAsync(vendorPlayerId, platType, amount, internalOrderId)
     ├── 返回码 10000 → 成功
     ├── 返回码 10005 → 失败
     └── 其他返回码 → 调用 IVendorAdapter.QueryTransferStatusAsync 轮询确认
         └── status=0(pending) → 等待重试（见 §9.5 TransferStatusRetryOptions：最多 3 次，间隔 5s）
         └── status=1(success) → 成功
         └── status=2(failed) → 失败

  ── 事务 2（根据厂商结果更新订单状态）──
     ├── 成功:
     │   7a. 调用 IVendorAdapter.GetGameUrlAsync(vendorPlayerId, platType, gameCode, gameType, ingress, returnUrl)
     │   8a. GameOrder.Confirm(externalOrderId, gameUrl)
     │   9a. 提交事务 2
     │   → 返回 { gameUrl, orderId }
     └── 失败:
         7b. GameOrder.Fail(errorMessage)
         8b. 钱包 Credit(amount, Refund) — 退款
         9b. 保存订单 + 流水
         10b. 提交事务 2
         → 返回错误
  ── 事务 2 结束 ──

Frontend: 检测设备类型（PC/移动端）→ 新窗口打开 gameUrl
```

**InternalOrderId 格式**：GUID 去连字符后取前 32 位，全小写字母+数字，符合厂商 orderId 要求（32 位字母数字组合）。生成位置：**Application 层** `OrderService.LaunchGameAsync` 内部，在事务 1 之前调用 `Guid.NewGuid().ToString("N")`。

**ingress 参数**：由前端根据 `navigator.userAgent` 或 CSS media query 判断，传递 `device1`（PC）或 `device2`（移动端）。

**returnUrl 参数**：设置为平台前端的游戏退出页面 URL，用户在厂商页面退出时自动跳转回来。

**事务边界说明**：厂商 API 调用不在数据库事务内，避免长事务。如果事务 1 成功但厂商调用前崩溃，PendingOrderRecoveryService 会自动拾取 Pending 订单进行恢复。

### 4.2 游戏退出流程（ExitGame — 一键回收模式）

```
User → Frontend: 退出游戏
Frontend → Backend: POST /api/games/exit { orderId }
Backend (OrderService):
  1. 查找 GameOrder (状态必须为 Confirmed, **且 UserId == 当前用户**)
  2. 获取 VendorPlayerMapping → vendorPlayerId, currency
  3. 调用 IVendorAdapter.TransferAllAsync(vendorPlayerId, currency) — 一键回收所有平台余额
     ├── 成功 (balanceAll > 0):
     │   4a. transferOutAmount = balanceAll
     │   5a. 钱包 Credit(transferOutAmount, GameTransferOut) — CAS 更新
     │   6a. 保存 WalletTransaction
     │   7a. GameOrder.Settle(transferOutAmount)
     │   8a. 保存订单
     │   9a. SignalR: BalanceUpdated
     │   → 返回 { newBalance, transferOutAmount }
     ├── 成功但回收金额 = 0:
     │   4b. GameOrder.Cancel("一键回收金额为0，资金已在厂商侧消耗")
     │   5b. 保存订单
     │   → 返回 { newBalance (不变), transferOutAmount: 0, status: Cancelled }
     └── 失败:
         4c. 记录 VendorQueryLog
         → 返回错误 (订单保持 Confirmed, 等待重试或人工处理)
```

**一键回收优势**：`/api/server/transferAll` 自动回收玩家在所有子平台（AG/PG/PP 等）的余额，避免遗漏某个平台的资金。超时设置必须 > 60 秒。

**注意**：部分游戏平台不支持游戏进行中的额度转换，用户必须先结束当前游戏牌局/回合才能成功回收。

### 4.3 厂商回调处理流程（HandleCallback）— 当前厂商不使用，接口预留

> **注意**：当前接入的厂商 API 无回调/Webhook 机制，全部通过轮询模式获取状态。以下流程仅为 IVendorAdapter 抽象层预留，未来接入支持回调的厂商时启用。

```
Vendor → Backend: POST /api/vendor/{vendorCode}/callback { payload, signature }
Backend (OrderService):
  1. 存储原始 payload → VendorCallbackLogs
  2. IVendorAdapter.VerifyCallbackSignature(payload, signature, timestamp)
     └── 失败: 记录日志(IsVerified=false), IAlertService.RaiseAlert(CallbackSignatureFailure), 返回 401
  3. IVendorAdapter.ParseCallback(payload) → { externalOrderId, status, amount }
  4. 查找 GameOrder by ExternalOrderId
     └── 未找到: 记录日志, 返回 200 (防止厂商重发)
  5. 幂等检查: 如果订单已是终态 → 跳过, 返回 200
  6. OrderStateMachine.ValidateTransition(currentStatus, newStatus)
  7. 如果需要结算 (→ Settled):
     7a. 钱包 Credit(amount, GameTransferOut)
     7b. 保存 WalletTransaction
  8. 更新 GameOrder 状态
  9. 保存所有变更 (事务)
  10. SignalR: OrderStatusChanged + BalanceUpdated
  → 返回 200
```

### 4.4 人工充值流程

```
User → Frontend: 提交充值申请 (amount + proof)
Frontend → Backend: POST /api/wallet/deposit-request
Backend:
  1. 创建 DepositRequest (Status=Pending)
  → 等待管理员审核

Admin → Admin Portal: 查看充值申请列表
Admin Portal → Backend: PUT /api/admin/deposit-requests/{id}/approve
Backend:
  1. DepositRequest.Approve(adminId, note)
  2. 钱包 Credit(amount, Deposit) — CAS 更新
  3. 保存 WalletTransaction
  4. 记录 AdminActionLog
  5. SignalR: BalanceUpdated (通知用户)
  → 返回成功
```

---

## 5. 数据库表结构

### 5.1 UserProfiles

```sql
CREATE TABLE "UserProfiles" (
    "Id"         UUID PRIMARY KEY,                          -- = auth.users.id
    "Username"   VARCHAR(50) NOT NULL UNIQUE,
    "Nickname"   VARCHAR(100),
    "Role"       SMALLINT NOT NULL DEFAULT 0,               -- 0=User, 1=Agent, 2=Admin
    "AgentId"    UUID REFERENCES "UserProfiles"("Id"),
    "Status"     SMALLINT NOT NULL DEFAULT 0,               -- 0=Active, 1=Disabled
    "CreatedAt"  TIMESTAMP NOT NULL DEFAULT NOW(),
    "UpdatedAt"  TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX "IX_UserProfiles_AgentId" ON "UserProfiles"("AgentId");
CREATE INDEX "IX_UserProfiles_Role" ON "UserProfiles"("Role");
CREATE INDEX "IX_UserProfiles_Status" ON "UserProfiles"("Status");
```

### 5.2 Wallets

```sql
CREATE TABLE "Wallets" (
    "Id"             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "UserId"         UUID NOT NULL UNIQUE REFERENCES "UserProfiles"("Id"),
    "Balance"        DECIMAL(18,2) NOT NULL DEFAULT 0.00,
    "FrozenBalance"  DECIMAL(18,2) NOT NULL DEFAULT 0.00,
    "Version"        INTEGER NOT NULL DEFAULT 0,
    "CreatedAt"      TIMESTAMP NOT NULL DEFAULT NOW(),
    "UpdatedAt"      TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT "CK_Wallets_Balance_NonNegative" CHECK ("Balance" >= 0),
    CONSTRAINT "CK_Wallets_FrozenBalance_NonNegative" CHECK ("FrozenBalance" >= 0),
    CONSTRAINT "CK_Wallets_FrozenBalance_LTE_Balance" CHECK ("FrozenBalance" <= "Balance")
);
```

### 5.3 WalletTransactions

```sql
CREATE TABLE "WalletTransactions" (
    "Id"              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "WalletId"        UUID NOT NULL REFERENCES "Wallets"("Id"),
    "Type"            SMALLINT NOT NULL,
    "Amount"          DECIMAL(18,2) NOT NULL,
    "BalanceBefore"   DECIMAL(18,2) NOT NULL,
    "BalanceAfter"    DECIMAL(18,2) NOT NULL,
    "Description"     VARCHAR(500) NOT NULL,
    "OrderId"         UUID,
    "InternalOrderId" VARCHAR(64),
    "CreatedAt"       TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX "IX_WalletTransactions_WalletId" ON "WalletTransactions"("WalletId");
CREATE INDEX "IX_WalletTransactions_OrderId" ON "WalletTransactions"("OrderId");
CREATE INDEX "IX_WalletTransactions_CreatedAt" ON "WalletTransactions"("CreatedAt");
-- 幂等索引：同一订单同一类型的流水不得重复（InternalOrderId 可为 NULL，仅对非 NULL 生效）
CREATE UNIQUE INDEX "IX_WalletTransactions_Idempotent"
    ON "WalletTransactions"("WalletId", "InternalOrderId", "Type")
    WHERE "InternalOrderId" IS NOT NULL;
```

### 5.4 GameOrders

```sql
CREATE TABLE "GameOrders" (
    "Id"                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "UserId"            UUID NOT NULL REFERENCES "UserProfiles"("Id"),
    "VendorCode"        VARCHAR(50) NOT NULL,
    "PlatType"          VARCHAR(20) NOT NULL,                         -- 游戏子平台（如 "ag", "pg", "pp"）
    "GameCode"          VARCHAR(50) NOT NULL,
    "InternalOrderId"   VARCHAR(32) NOT NULL UNIQUE,                  -- 32 位字母数字（GUID 去连字符）
    "ExternalOrderId"   VARCHAR(64) UNIQUE,
    "TransferInAmount"  DECIMAL(18,2) NOT NULL,
    "TransferOutAmount" DECIMAL(18,2),
    "Status"            SMALLINT NOT NULL DEFAULT 0,
    "GameUrl"           VARCHAR(2000),
    "ErrorMessage"      VARCHAR(1000),
    "CancelReason"      VARCHAR(1000),
    "CreatedAt"         TIMESTAMP NOT NULL DEFAULT NOW(),
    "UpdatedAt"         TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX "IX_GameOrders_UserId" ON "GameOrders"("UserId");
CREATE INDEX "IX_GameOrders_Status" ON "GameOrders"("Status");
CREATE INDEX "IX_GameOrders_VendorCode" ON "GameOrders"("VendorCode");
CREATE INDEX "IX_GameOrders_CreatedAt" ON "GameOrders"("CreatedAt");
```

### 5.5 DepositRequests

```sql
CREATE TABLE "DepositRequests" (
    "Id"          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "UserId"      UUID NOT NULL REFERENCES "UserProfiles"("Id"),
    "Amount"      DECIMAL(18,2) NOT NULL,
    "Proof"       VARCHAR(2000),
    "Status"      SMALLINT NOT NULL DEFAULT 0,
    "ReviewedBy"  UUID REFERENCES "UserProfiles"("Id"),
    "ReviewNote"  VARCHAR(500),
    "CreatedAt"   TIMESTAMP NOT NULL DEFAULT NOW(),
    "UpdatedAt"   TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX "IX_DepositRequests_UserId" ON "DepositRequests"("UserId");
CREATE INDEX "IX_DepositRequests_Status" ON "DepositRequests"("Status");
```

### 5.6 WithdrawRequests

```sql
CREATE TABLE "WithdrawRequests" (
    "Id"          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "UserId"      UUID NOT NULL REFERENCES "UserProfiles"("Id"),
    "Amount"      DECIMAL(18,2) NOT NULL,
    "BankInfo"    JSONB NOT NULL,
    "Status"      SMALLINT NOT NULL DEFAULT 0,
    "ReviewedBy"  UUID REFERENCES "UserProfiles"("Id"),
    "ReviewNote"  VARCHAR(500),
    "CreatedAt"   TIMESTAMP NOT NULL DEFAULT NOW(),
    "UpdatedAt"   TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX "IX_WithdrawRequests_UserId" ON "WithdrawRequests"("UserId");
CREATE INDEX "IX_WithdrawRequests_Status" ON "WithdrawRequests"("Status");
```

### 5.7 VendorCallbackLogs

```sql
CREATE TABLE "VendorCallbackLogs" (
    "Id"            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "VendorCode"    VARCHAR(50) NOT NULL,
    "OrderId"       UUID REFERENCES "GameOrders"("Id"),
    "RawPayload"    JSONB NOT NULL,
    "Signature"     VARCHAR(500),
    "IsVerified"    BOOLEAN NOT NULL DEFAULT FALSE,
    "ProcessResult" VARCHAR(50),
    "ErrorMessage"  VARCHAR(1000),
    "CreatedAt"     TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX "IX_VendorCallbackLogs_OrderId" ON "VendorCallbackLogs"("OrderId");
CREATE INDEX "IX_VendorCallbackLogs_CreatedAt" ON "VendorCallbackLogs"("CreatedAt");
CREATE INDEX "IX_VendorCallbackLogs_VendorCode" ON "VendorCallbackLogs"("VendorCode");
```

### 5.8 VendorQueryLogs

```sql
CREATE TABLE "VendorQueryLogs" (
    "Id"           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "VendorCode"   VARCHAR(50) NOT NULL,
    "OrderId"      UUID NOT NULL REFERENCES "GameOrders"("Id"),
    "QueryResult"  JSONB,
    "Attempt"      INTEGER NOT NULL DEFAULT 1,
    "IsSuccess"    BOOLEAN NOT NULL DEFAULT FALSE,
    "ErrorMessage" VARCHAR(1000),
    "CreatedAt"    TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX "IX_VendorQueryLogs_OrderId" ON "VendorQueryLogs"("OrderId");
CREATE INDEX "IX_VendorQueryLogs_CreatedAt" ON "VendorQueryLogs"("CreatedAt");
```

### 5.9 AdminActionLogs

```sql
CREATE TABLE "AdminActionLogs" (
    "Id"              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "AdminId"         UUID REFERENCES "UserProfiles"("Id"),    -- NULL = 系统自动操作
    "Action"          VARCHAR(100) NOT NULL,
    "TargetType"      VARCHAR(50) NOT NULL,
    "TargetId"        UUID NOT NULL,
    "Details"         JSONB NOT NULL,
    "Level"           SMALLINT NOT NULL DEFAULT 0,     -- 0=Info, 1=Warning, 2=Error
    "IdempotencyKey"  VARCHAR(64),                     -- 管理员操作幂等键（前端生成 UUID）
    "CreatedAt"       TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX "IX_AdminActionLogs_AdminId" ON "AdminActionLogs"("AdminId");
CREATE INDEX "IX_AdminActionLogs_TargetType_TargetId" ON "AdminActionLogs"("TargetType", "TargetId");
CREATE INDEX "IX_AdminActionLogs_Level" ON "AdminActionLogs"("Level");
CREATE INDEX "IX_AdminActionLogs_CreatedAt" ON "AdminActionLogs"("CreatedAt");
CREATE UNIQUE INDEX "IX_AdminActionLogs_IdempotencyKey"
    ON "AdminActionLogs"("IdempotencyKey") WHERE "IdempotencyKey" IS NOT NULL;
```

### 5.10 VendorPlayerMappings

```sql
CREATE TABLE "VendorPlayerMappings" (
    "Id"              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "UserId"          UUID NOT NULL REFERENCES "UserProfiles"("Id"),
    "VendorCode"      VARCHAR(50) NOT NULL,
    "VendorPlayerId"  VARCHAR(20) NOT NULL,                 -- 厂商侧玩家 ID（5-11 位小写字母+数字）
    "Currency"        VARCHAR(10) NOT NULL DEFAULT 'CNY',
    "IsCreated"       BOOLEAN NOT NULL DEFAULT FALSE,       -- 是否已在厂商侧创建成功
    "CreatedAt"       TIMESTAMP NOT NULL DEFAULT NOW(),
    "UpdatedAt"       TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT "UQ_VendorPlayerMappings_User_Vendor" UNIQUE ("UserId", "VendorCode"),
    CONSTRAINT "UQ_VendorPlayerMappings_VendorPlayerId" UNIQUE ("VendorCode", "VendorPlayerId")
);
```

### 5.11 GameBetRecords

```sql
CREATE TABLE "GameBetRecords" (
    "Id"              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "UserId"          UUID NOT NULL REFERENCES "UserProfiles"("Id"),
    "VendorPlayerId"  VARCHAR(20) NOT NULL,
    "PlatType"        VARCHAR(20) NOT NULL,
    "Currency"        VARCHAR(10) NOT NULL,
    "GameType"        VARCHAR(10) NOT NULL,             -- 1=视讯, 2=老虎机, 3=彩票, 4=体育, 5=电竞, 6=捕猎, 7=棋牌
    "GameName"        VARCHAR(200) NOT NULL,
    "Round"           VARCHAR(100) NOT NULL,
    "Table"           VARCHAR(100),
    "Seat"            VARCHAR(50),
    "BetAmount"       DECIMAL(18,4) NOT NULL,             -- 4 位小数（兼容厂商高精度金额）
    "ValidAmount"     DECIMAL(18,4) NOT NULL,
    "SettledAmount"   DECIMAL(18,4) NOT NULL,            -- 输赢金额（正=赢，负=输）
    "BetContent"      TEXT,
    "Status"          SMALLINT NOT NULL DEFAULT 0,       -- 0=未完成, 1=已完成, 2=已取消, 3=已撤单
    "VendorBetOrderId"     VARCHAR(100) NOT NULL,             -- 厂商订单 ID（唯一索引，幂等用）
    "BetTime"         TIMESTAMP NOT NULL,                -- 投注时间
    "LastUpdateTime"  TIMESTAMP NOT NULL,                -- 厂商最后更新时间
    "SyncedAt"        TIMESTAMP NOT NULL DEFAULT NOW()   -- 本地同步时间
);

CREATE UNIQUE INDEX "IX_GameBetRecords_VendorBetOrderId" ON "GameBetRecords"("VendorBetOrderId");
CREATE INDEX "IX_GameBetRecords_UserId" ON "GameBetRecords"("UserId");
CREATE INDEX "IX_GameBetRecords_PlatType" ON "GameBetRecords"("PlatType");
CREATE INDEX "IX_GameBetRecords_BetTime" ON "GameBetRecords"("BetTime");
CREATE INDEX "IX_GameBetRecords_VendorPlayerId" ON "GameBetRecords"("VendorPlayerId");
```

### 5.12 UserBankCards

```sql
CREATE TABLE "UserBankCards" (
    "Id"              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "UserId"          UUID NOT NULL REFERENCES "UserProfiles"("Id"),
    "BankName"        VARCHAR(100) NOT NULL,
    "CardNumber"      VARCHAR(30) NOT NULL,               -- 完整卡号（加密存储）
    "CardNumberLast4" VARCHAR(4) NOT NULL,                 -- 后4位明文（前端展示用）
    "CardHolderName"  VARCHAR(50) NOT NULL,
    "BranchName"      VARCHAR(200),
    "IsDefault"       BOOLEAN NOT NULL DEFAULT FALSE,
    "CreatedAt"       TIMESTAMP NOT NULL DEFAULT NOW(),
    "UpdatedAt"       TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX "IX_UserBankCards_UserId" ON "UserBankCards"("UserId");
```

### 5.13 日志表物理删除防护

以下表**禁止物理删除**（CLAUDE.md 强制要求）：
- `WalletTransactions`
- `VendorCallbackLogs`
- `VendorQueryLogs`
- `AdminActionLogs`

**防护措施**（通过 Supabase RLS 或 PostgreSQL 触发器实现）：

```sql
-- 方式 1：REVOKE DELETE 权限（推荐，在 Supabase 中对应用角色执行）
REVOKE DELETE ON "WalletTransactions" FROM authenticated;
REVOKE DELETE ON "VendorCallbackLogs" FROM authenticated;
REVOKE DELETE ON "VendorQueryLogs" FROM authenticated;
REVOKE DELETE ON "AdminActionLogs" FROM authenticated;

-- 方式 2：防删除触发器（备选）
CREATE OR REPLACE FUNCTION prevent_delete() RETURNS trigger AS $$
BEGIN
    RAISE EXCEPTION 'Physical deletion is prohibited on this table';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_delete_wallet_transactions
    BEFORE DELETE ON "WalletTransactions" FOR EACH ROW EXECUTE FUNCTION prevent_delete();
CREATE TRIGGER no_delete_vendor_callback_logs
    BEFORE DELETE ON "VendorCallbackLogs" FOR EACH ROW EXECUTE FUNCTION prevent_delete();
CREATE TRIGGER no_delete_vendor_query_logs
    BEFORE DELETE ON "VendorQueryLogs" FOR EACH ROW EXECUTE FUNCTION prevent_delete();
CREATE TRIGGER no_delete_admin_action_logs
    BEFORE DELETE ON "AdminActionLogs" FOR EACH ROW EXECUTE FUNCTION prevent_delete();
```

EF Core 的 `AppDbContext` 也应覆写 `SaveChangesAsync`，拦截对日志实体的 `EntityState.Deleted` 操作并抛出异常。

---

## 6. IVendorAdapter 接口设计

```csharp
public interface IVendorAdapter
{
    /// 厂商标识码
    string VendorCode { get; }

    // ═══ 玩家管理 ═══

    /// 在厂商侧创建玩家账号
    Task<VendorResult> CreatePlayerAsync(string vendorPlayerId, string platType, string currency);

    // ═══ 余额查询 ═══

    /// 查询用户在指定平台的余额
    Task<VendorBalanceResult> QueryBalanceAsync(string vendorPlayerId, string platType, string currency);

    /// 一键查询用户在所有平台的余额（限频：3 次/分钟/玩家）
    Task<VendorBalanceAllResult> QueryBalanceAllAsync(string vendorPlayerId, string currency);

    // ═══ 额度转换 ═══

    /// 转入资金到厂商指定平台（type=1）
    Task<TransferResult> TransferInAsync(string vendorPlayerId, string platType, string currency, decimal amount, string orderId);

    /// 转出资金从厂商指定平台（type=2）
    Task<TransferResult> TransferOutAsync(string vendorPlayerId, string platType, string currency, decimal amount, string orderId);

    /// 一键回收所有平台余额（限频：2 次/分钟/玩家，超时 > 60s）
    Task<TransferAllResult> TransferAllAsync(string vendorPlayerId, string currency);

    /// 查询转账状态（转账返回非 10000/10005 时必须调用）
    Task<TransferStatusResult> QueryTransferStatusAsync(string vendorPlayerId, string currency, string orderId);

    // ═══ 游戏启动 ═══

    /// 获取游戏启动 URL
    Task<GameLaunchResult> GetGameUrlAsync(string vendorPlayerId, string platType, string currency,
        string gameType, string? gameCode, string ingress, string? returnUrl, string? lang = null);

    /// 获取试玩游戏 URL
    Task<GameLaunchResult> GetDemoUrlAsync(string platType, string currency,
        string gameType, string? gameCode, string ingress, string? returnUrl, string? lang = null);

    // ═══ 游戏列表 ═══

    /// 获取指定平台的游戏列表（限频：30 次/小时）
    Task<IReadOnlyList<GameInfo>> GetGameListAsync(string platType);

    // ═══ 投注记录 ═══

    /// 获取实时投注记录（最近 10 分钟，限频：1 次/分钟）
    Task<PagedVendorResult<GameBetRecordData>> GetRealtimeRecordsAsync(string currency, int pageNo = 1, int pageSize = 200);

    /// 获取历史投注记录（限频：5 次/小时，时间范围 ≤ 6 小时）
    Task<PagedVendorResult<GameBetRecordData>> GetHistoryRecordsAsync(string currency, DateTime startTime, DateTime endTime, int pageNo = 1, int pageSize = 200);

    // ═══ 商户接口 ═══

    /// 查询商户在厂商的余额
    Task<MerchantBalanceResult> QueryMerchantBalanceAsync(string currency);

    // ═══ 回调处理（预留，当前厂商不使用）═══

    /// 验证回调签名（当前厂商无回调机制，预留接口）
    bool VerifyCallbackSignature(string payload, string signature, string timestamp);

    /// 解析回调数据（当前厂商无回调机制，预留接口）
    VendorCallbackData ParseCallback(string payload);
}

// ═══ 返回值类型 ═══

public record VendorResult(bool Success, int Code, string? ErrorMessage);

public record VendorBalanceResult(bool Success, decimal Balance, string? ErrorMessage);

public record VendorBalanceAllResult(bool Success, Dictionary<string, decimal?> PlatformBalances, string? ErrorMessage);

public record TransferResult(bool Success, int Code, string? ExternalOrderId, string? ErrorMessage)
{
    /// 是否需要通过 QueryTransferStatusAsync 确认（非 10000/10005）
    public bool NeedsStatusCheck => Code != 10000 && Code != 10005;
}

public record TransferAllResult(bool Success, decimal TotalAmount, Dictionary<string, decimal> PlatformAmounts, string? ErrorMessage);

public record TransferStatusResult(bool Success, int Status, decimal Amount, decimal AfterBalance, string? ErrorMessage)
{
    /// 转账状态：0=处理中, 1=成功, 2=失败
    public bool IsPending => Status == 0;
    public bool IsSuccess => Status == 1;
    public bool IsFailed => Status == 2;
}

public record GameLaunchResult(bool Success, string? GameUrl, string? ErrorMessage);

public record GameInfo(string PlatType, string GameCode, string GameType, string GameName,
    string Ingress, Dictionary<string, string>? LocalizedNames, string? ThumbnailUrl);

public record GameBetRecordData(string PlayerId, string PlatType, string Currency, string GameType,
    string GameName, string Round, string? Table, string? Seat, decimal BetAmount, decimal ValidAmount,
    decimal SettledAmount, string? BetContent, int Status, string VendorBetOrderId,
    DateTime BetTime, DateTime LastUpdateTime);

public record PagedVendorResult<T>(bool Success, int Total, int PageNo, int PageSize,
    IReadOnlyList<T> Items, string? ErrorMessage);

public record MerchantBalanceResult(bool Success, decimal Balance, decimal CostRatio,
    Dictionary<string, decimal>? PlatformRatios, string? ErrorMessage);

public record VendorCallbackData(string ExternalOrderId, string EventType, decimal? Amount, string RawStatus);
```

### 6.1 IVendorAdapterFactory（厂商适配器工厂）

```csharp
public interface IVendorAdapterFactory
{
    /// 根据厂商标识码获取对应的适配器实例
    IVendorAdapter GetAdapter(string vendorCode);

    /// 获取所有已注册的厂商标识码
    IReadOnlyList<string> GetRegisteredVendorCodes();
}
```

**实现方式**：
```csharp
public class VendorAdapterFactory : IVendorAdapterFactory
{
    private readonly Dictionary<string, IVendorAdapter> _adapters;

    public VendorAdapterFactory(IEnumerable<IVendorAdapter> adapters)
    {
        _adapters = adapters.ToDictionary(a => a.VendorCode, StringComparer.OrdinalIgnoreCase);
    }

    public IVendorAdapter GetAdapter(string vendorCode)
    {
        if (!_adapters.TryGetValue(vendorCode, out var adapter))
            throw new DomainException($"Unknown vendor: {vendorCode}");
        return adapter;
    }

    public IReadOnlyList<string> GetRegisteredVendorCodes() => _adapters.Keys.ToList();
}
```

**DI 注册**：
```csharp
builder.Services.AddSingleton<IVendorAdapter, VendorXAdapter>();
// 未来新增厂商只需追加注册
builder.Services.AddSingleton<IVendorAdapterFactory, VendorAdapterFactory>();
```

### 6.2 游戏列表数据源与缓存策略

MVP 阶段不设独立 Games 表，游戏列表数据来源为 `IVendorAdapter.GetGameListAsync()`。

**缓存策略**：
- 使用 `IMemoryCache`，缓存 key = `games:{vendorCode}`
- 缓存有效期：**10 分钟**（`SlidingExpiration`）
- 首次请求或缓存过期时调用厂商 API 刷新
- 管理员可通过 `POST /api/admin/games/refresh-cache` 手动刷新（可选，P2）

**Application 层接口**：

```csharp
public interface IGameService
{
    Task<IReadOnlyList<GameInfoDto>> GetGameListAsync(string platType, string? category = null);
    Task<IReadOnlyList<string>> GetSupportedPlatTypesAsync();
    Task<GameInfoDto?> GetDemoUrlAsync(string platType, string gameCode, string gameType, string ingress);
    Task<PagedList<GameBetRecordDto>> GetBetRecordsAsync(Guid userId, int page, int pageSize);
    Task<PagedList<GameBetRecordDto>> GetAllBetRecordsAsync(int page, int pageSize, string? platType = null);
}
```

---

## 7. SignalR Hub 设计

```csharp
public class NotificationHub : Hub
{
    // 客户端监听的事件:
    // "BalanceUpdated"           → { newBalance: decimal, transactionType: string }
    // "OrderStatusChanged"       → { orderId: Guid, newStatus: string }
    // "DepositRequestUpdated"    → { requestId: Guid, status: string }
    // "WithdrawRequestUpdated"   → { requestId: Guid, status: string }
    // "SystemNotification"       → { message: string, level: string }
}
```

**连接认证**：JWT Bearer Token，Hub 建立连接时验证。
**消息分发**：按 userId 分组（`Groups.AddToGroupAsync(Context.ConnectionId, userId)`），仅推送给目标用户。
**管理员通道**：Admin 角色加入 "admins" 组，接收告警通知。

### 7.1 INotificationService 接口

```csharp
public interface INotificationService
{
    /// 通知用户余额变动（充值/扣款/游戏转入转出/调整）
    Task NotifyBalanceUpdatedAsync(Guid userId, decimal newBalance, string transactionType);

    /// 通知用户订单状态变更
    Task NotifyOrderStatusChangedAsync(Guid userId, Guid orderId, string newStatus);

    /// 通知用户充值申请状态变更（Approved / Rejected）
    Task NotifyDepositRequestUpdatedAsync(Guid userId, Guid requestId, string status);

    /// 通知用户提现申请状态变更（Approved / Rejected / Completed）
    Task NotifyWithdrawRequestUpdatedAsync(Guid userId, Guid requestId, string status);

    /// 系统通知（告警推送给管理员、维护公告等）
    Task NotifySystemAsync(Guid userId, string message, string level = "Info");

    /// 向所有管理员推送告警（发送到 "admins" 组）
    Task NotifyAdminsAlertAsync(string alertType, string message, object? details = null);
}
```

**实现**：`SignalRNotificationService`（Infrastructure 层），通过 `IHubContext<NotificationHub>` 调用 Hub 的 `SendAsync` 方法。

---

## 8. CAS 更新实现（Infrastructure 层）

```csharp
// WalletRepository.UpdateWithCasAsync
public async Task<bool> UpdateWithCasAsync(Wallet wallet, int expectedVersion)
{
    var sql = @"
        UPDATE ""Wallets""
        SET ""Balance"" = @Balance,
            ""FrozenBalance"" = @FrozenBalance,
            ""Version"" = @Version,
            ""UpdatedAt"" = @UpdatedAt
        WHERE ""Id"" = @Id
          AND ""Version"" = @ExpectedVersion";

    var affected = await _context.Database.ExecuteSqlRawAsync(sql,
        new NpgsqlParameter("@Balance", wallet.Balance),
        new NpgsqlParameter("@FrozenBalance", wallet.FrozenBalance),
        new NpgsqlParameter("@Version", wallet.Version),
        new NpgsqlParameter("@UpdatedAt", wallet.UpdatedAt),
        new NpgsqlParameter("@Id", wallet.Id),
        new NpgsqlParameter("@ExpectedVersion", expectedVersion));

    return affected > 0;
}
```

调用方必须检查返回值，`false` 表示版本冲突，需重试或拒绝。

---

## 9. 告警模块设计

### 9.1 IAlertService 接口

```csharp
public interface IAlertService
{
    /// 发送告警（记录到 AdminActionLogs + SignalR 推送管理员）
    Task RaiseAlertAsync(AlertType type, string message, object? details = null);

    /// 查询告警列表（查询 AdminActionLogs 中 Level=Error 的记录）
    Task<PagedList<AlertDto>> GetAlertsAsync(int page, int pageSize);
}
```

```csharp
public enum AlertType
{
    BalanceInconsistency,       // 余额不一致
    DuplicateSettlement,        // 重复结算
    CallbackSignatureFailure,   // 回调签名验证失败（预留，当前厂商不使用）
    PollingMaxRetryExceeded,    // 轮询超过最大重试次数
    TransferStatusTimeout,      // 转账状态查询超时（transferStatus 多次 pending）
    MerchantBalanceLow,         // 商户余额不足（可能导致转入失败）
    VendorRateLimitExceeded,    // 厂商 API 限频（收到 10009）
    CasConflictFrequencyHigh,  // CAS 冲突频率异常
    ConfirmedOrderTimeout      // Confirmed 订单超时强制回收（超过 ConfirmedMaxLifetimeHours）
}
```

### 9.2 告警与 AdminActionLogs 关系

告警**不设独立表**，统一存储到 `AdminActionLogs`：
- `Level = 2 (Error)` 的记录即为告警
- `Action` 字段存储 `AlertType` 枚举名称（如 `"BalanceInconsistency"`）
- `TargetType` = `"System"` | `"Wallet"` | `"Order"` | `"VendorCallback"`
- `Details` 存储告警详情 JSON
- `AdminId` 在自动告警场景设为 `NULL`（AdminActionLogs.AdminId 已改为可选外键）

`GET /api/admin/alerts` 实际查询 `AdminActionLogs WHERE Level = 2 ORDER BY CreatedAt DESC`。

### 9.3 告警触发配置

| 告警类型 | 触发条件 | 配置参数 |
|----------|----------|----------|
| 余额不一致 | 对账任务发现 `Wallet.Balance ≠ SUM(WalletTransactions)` | 对账任务间隔：**每 4 小时** |
| 重复结算 | 同一订单被处理多次（CAS 检测到） | 自动检测，无阈值 |
| 回调签名失败 | 回调验签不通过（预留，当前厂商无回调） | 自动检测，每次失败触发 |
| 轮询超过最大重试 | 单订单轮询次数 > **MaxPollRetryCount** | **MaxPollRetryCount = 10** |
| 转账状态超时 | transferStatus 连续 3 次返回 pending | 自动检测 |
| 商户余额不足 | 商户余额低于阈值 | **MerchantBalanceThreshold = 1000** |
| 厂商 API 限频 | 收到错误码 10009 | 自动检测，每次触发 |
| CAS 冲突频率异常 | **1 分钟内**同一钱包 CAS 失败次数 > **CasConflictThreshold** | **CasConflictThreshold = 5** |
| Confirmed 订单超时 | Confirmed 订单 CreatedAt > **ConfirmedMaxLifetimeHours** | **ConfirmedMaxLifetimeHours = 24** |

### 9.4 轮询配置参数

```csharp
public class PollingOptions
{
    public int MaxRetryCount { get; set; } = 10;           // 单订单最大轮询次数
    public int IntervalSeconds { get; set; } = 30;          // 轮询间隔（秒）
    public int PendingThresholdMinutes { get; set; } = 5;   // Pending 超过此时间才轮询
    public int ConfirmedMaxLifetimeHours { get; set; } = 24; // Confirmed 订单最大生存时间（小时），超时后强制回收
}
```

- 超过 `MaxRetryCount` 的订单标记为需要人工处理，不再自动轮询
- 失败记录到 `VendorQueryLogs`，触发 `PollingMaxRetryExceeded` 告警

### 9.5 TransferStatus 即时重试配置

TransferIn 调用后，若返回码不是 10000（成功）或 10005（失败），需立即通过 `QueryTransferStatusAsync` 确认结果。此重试与后台 PollingOptions 不同，是同步阻塞式的即时重试：

```csharp
public class TransferStatusRetryOptions
{
    public int MaxRetryCount { get; set; } = 3;             // 最大重试次数
    public int IntervalMs { get; set; } = 5000;             // 重试间隔（毫秒）
}
```

- 重试逻辑在 `OrderService.LaunchGameAsync` 内部（§4.1 步骤 6）
- 若 3 次重试后 transferStatus 仍返回 `status=0`（pending）→ 订单保持 `Pending`，由 `PendingOrderRecoveryService` 后续处理
- 重试期间 HTTP 请求保持等待（总耗时上限 ≈ 3×5s = 15s + API 调用时间）
- 区别于 `PollingOptions`：PollingOptions 用于后台定时任务，IntervalSeconds=30，MaxRetryCount=10

---

## 10. 后台服务设计（BackgroundService）

### 10.1 BalanceReconciliationService（对账任务）

```
执行间隔: 每 4 小时
执行逻辑:
  1. 获取所有 Active 状态用户的 Wallet 列表（分批，每批 100）
  2. 对每个 Wallet:
     a. 查询 Wallet.Balance
     b. 查询 SUM(WalletTransactions.Amount) WHERE WalletId = wallet.Id
     c. 比较两者是否一致
     d. 不一致 → IAlertService.RaiseAlert(BalanceInconsistency, details)
  3. 记录对账结果到 AdminActionLog（Level=Info 或 Error）
  4. 全部完成后记录总体结果
```

**幂等保证**：对账是只读操作 + 告警写入，不修改业务数据，天然幂等。

### 10.2 PendingOrderRecoveryService（异常恢复）

```
执行间隔: 每 2 分钟
执行逻辑:
  1. 查询 GameOrders WHERE Status = Pending AND UpdatedAt < NOW() - 5分钟
  2. 对每个 Pending 订单:
     a. 获取 VendorPlayerMapping → vendorPlayerId, currency
     b. 调用 IVendorAdapter.QueryTransferStatusAsync(vendorPlayerId, currency, internalOrderId)
     c. 根据 transferStatus 结果:
        ├── status=1(成功) → 获取游戏 URL → Confirm 订单
        ├── status=2(失败) → 订单 Fail + 钱包 Refund
        ├── status=0(处理中) → 等待下次轮询
        └── 订单不存在(10013) → 订单 Fail + 钱包 Refund
     d. 每步都在事务内完成
  3. 记录处理结果到 VendorQueryLog
```

**幂等保证**：处理前检查订单当前状态，已非 Pending 则跳过。

### 10.3 OrderPollingService（轮询 Confirmed 订单）

```
执行间隔: 每 30 秒（PollingOptions.IntervalSeconds）
执行逻辑:
  1. 查询 GameOrders WHERE Status = Confirmed
     AND UpdatedAt < NOW() - PendingThresholdMinutes
  2. 超时检测：如果 CreatedAt < NOW() - ConfirmedMaxLifetimeHours（默认 24h）
     → 强制执行 TransferAll 回收 + Settle/Cancel + 记录 AdminActionLog（系统自动操作，AdminId=null）
     → 触发 IAlertService.RaiseAlert(ConfirmedOrderTimeout)
  3. 对每个订单:
     a. 查询 VendorQueryLogs 中该订单的 Attempt 次数
     b. 如果 Attempt >= MaxRetryCount → 跳过（已触发告警）
     c. 获取 VendorPlayerMapping → vendorPlayerId, currency
     d. 调用 IVendorAdapter.QueryBalanceAsync(vendorPlayerId, platType, currency) 查询余额
     e. 如果余额 = 0 或 null（游戏已结束）→ 触发一键回收 TransferAllAsync
        ├── 回收金额 > 0 → Settle
        └── 回收金额 = 0 → Cancel
     f. 如果余额 > 0（游戏进行中）→ 等待下次轮询
     g. 记录 VendorQueryLog（Attempt+1）
     h. 如果 Attempt = MaxRetryCount → IAlertService.RaiseAlert(PollingMaxRetryExceeded)
```

### 10.4 BetRecordSyncService（投注记录同步）

```
执行间隔: 每 2 分钟
执行逻辑:
  1. 调用 IVendorAdapter.GetRealtimeRecordsAsync(currency) — 获取最近 10 分钟投注记录
  2. 对每条记录:
     a. 根据 playerId 查找 VendorPlayerMapping → UserId
     b. 构建 GameBetRecord 实体
  3. 批量 Upsert 到 GameBetRecords 表（按 VendorBetOrderId 幂等）
  4. 记录同步结果（成功数、跳过数、失败数）
```

**分页处理**：如果 total > pageSize，自动翻页获取全部记录。
**历史补全**：管理员可手动触发 GetHistoryRecordsAsync 按时间段补全历史数据（P2）。

---

## 11. 厂商配置设计

```csharp
public class VendorOptions
{
    public Dictionary<string, VendorConfig> Vendors { get; set; } = new();
}

public class VendorConfig
{
    public string VendorCode { get; set; }           // 厂商标识（如 "apibet"）
    public string ApiBaseUrl { get; set; }            // API 基础地址（如 "https://ap.api-bet.net"）
    public string Sn { get; set; }                    // 商户前缀（Merchant prefix，从厂商后台获取）
    public string SecretKey { get; set; }             // 签名密钥（用于 MD5(random+sn+secretKey)）
    public string Currency { get; set; } = "CNY";     // 默认货币
    public string[] SupportedPlatTypes { get; set; }  // 启用的子平台列表（如 ["ag","pg","pp","cq9"]）
    public int TimeoutSeconds { get; set; } = 65;     // API 调用超时（转账 API 要求 > 60 秒）
    public int TransferTimeoutSeconds { get; set; } = 65;  // 转账专用超时
    public string? CallbackSecret { get; set; }       // 回调签名秘钥（预留，当前厂商不使用）
    public string[]? AllowedCallbackIps { get; set; } // 回调 IP 白名单（预留）
}
```

**配置来源**：
- 开发环境：`appsettings.Development.json`（仅结构，不含真实密钥）
- Staging / Production：环境变量或 Secret Manager
- 环境变量命名规范：`VENDOR_{VendorCode}_{Key}`，如 `VENDOR_APIBET_SN`, `VENDOR_APIBET_SECRET_KEY`

**注册方式**：
```csharp
builder.Services.Configure<VendorOptions>(builder.Configuration.GetSection("Vendors"));
```

---

## 12. CAS 重试策略

Application 层对 CAS 版本冲突的重试策略：

```csharp
public class CasRetryOptions
{
    public int MaxRetryCount { get; set; } = 3;            // 最大重试次数
    public int BaseDelayMs { get; set; } = 50;              // 基础延迟（毫秒）
    public int MaxDelayMs { get; set; } = 200;              // 最大延迟（毫秒）
}
```

**重试逻辑**：
1. 调用 `UpdateWithCasAsync` → 返回 `false`（版本冲突）
2. 重新从数据库加载 Wallet 最新状态
3. 重新执行 Domain 层的 Credit/Debit 操作
4. 重试 `UpdateWithCasAsync`
5. 重复最多 `MaxRetryCount` 次，每次随机延迟 `BaseDelayMs ~ MaxDelayMs`
6. 超过重试次数 → 抛出 `ConcurrencyConflictException`

**回调相关备注**：当前接入的厂商 API 无回调/Webhook 机制，`VendorConfig.CallbackSecret` 和 `AllowedCallbackIps` 均为预留字段。未来接入支持回调的厂商时，在回调处理流程中启用签名验证和 IP 白名单校验。

**IP 白名单备注**：厂商要求我方服务器 IP 加入其白名单（出站 IP），否则返回 10403。部署时需向厂商提供服务器出站 IP。

---

## 13. 数据库迁移规范

- 所有结构变更必须通过 `dotnet ef migrations add <MigrationName>` 生成
- 每个 Migration 必须包含 `Up()` 和 `Down()` 方法（可回滚）
- 禁止在 Migration 中删除含数据的列或表（先标记废弃 `[Obsolete]`，下个版本再清理）
- Migration 文件必须纳入 Git 版本控制
- 生产部署前需审核 Migration 内容（PR Review 阶段）
- 回滚策略：通过 `dotnet ef database update <PreviousMigration>` 执行 Down 方法

---

## 14. 凭证文件存储方案

### 14.1 Supabase Storage

充值凭证（转账截图）使用 **Supabase Storage** 存储：

- Bucket: `deposit-proofs`（私有，需认证访问）
- 文件命名: `{userId}/{requestId}.{ext}`（如 `abc123/def456.jpg`）
- 最大文件大小: 5MB
- 允许格式: `image/jpeg`, `image/png`

**上传流程**：
1. 前端通过 Supabase Storage SDK 直接上传到 Storage
2. 上传成功后获取文件路径
3. 将文件路径提交到后端 API（`POST /api/wallet/deposit-request`）
4. 后端存储路径到 `DepositRequest.Proof` 字段

**RLS 规则**：
- 用户只能上传到自己的目录（`{userId}/`）
- 用户只能读取自己的文件
- 管理员可读取所有文件（审核用）

---

## 15. 分页查询规范

所有分页查询的默认值和限制：

| 参数 | 默认值 | 最小值 | 最大值 |
|------|--------|--------|--------|
| page | 1 | 1 | - |
| pageSize | 20 | 1 | 100 |

**实现方式**：
- Application 层定义 `PagingOptions` 类，统一校验和默认值
- API 层使用 `[FromQuery]` 绑定，Application 层统一 clamp

```csharp
public class PagingOptions
{
    private const int DefaultPageSize = 20;
    private const int MaxPageSize = 100;

    public int Page { get; init; } = 1;
    public int PageSize { get; init; } = DefaultPageSize;

    public int ValidatedPage => Math.Max(1, Page);
    public int ValidatedPageSize => Math.Clamp(PageSize, 1, MaxPageSize);
    public int Skip => (ValidatedPage - 1) * ValidatedPageSize;
}
```

---

## 16. 健康检查端点

### 16.1 端点定义

```
GET /api/health        → 200 OK / 503 Service Unavailable
```

**检查项目**：
| 检查 | 说明 | 失败影响 |
|------|------|----------|
| Database | PostgreSQL 连接可达 | 503 |
| Supabase Auth | Auth API 可达（可选） | 降级（Warning） |
| SignalR | Hub 可用 | 降级（Warning） |

**响应格式**：
```json
{
  "status": "Healthy",
  "checks": {
    "database": { "status": "Healthy", "responseTimeMs": 12 },
    "supabaseAuth": { "status": "Healthy", "responseTimeMs": 45 },
    "signalr": { "status": "Healthy" }
  },
  "version": "1.0.0",
  "timestamp": "2026-03-03T14:30:00"
}
```

**实现方式**：使用 ASP.NET Core `HealthChecks` 中间件。

---

## 17. 管理员操作幂等控制

管理员手动调整（AdminAdjust）等操作缺少 InternalOrderId，需要独立的幂等保护：

**方案**：
- 管理员操作请求必须携带 `idempotencyKey`（前端生成 UUID）
- Application 层在执行前检查 `AdminActionLogs` 中是否已存在相同 key
- 如果存在，返回上次操作结果（幂等）

```csharp
// AdminAdjustRequest DTO
public record AdminAdjustRequest(
    decimal Amount,
    string Description,
    string IdempotencyKey    // 前端生成的唯一键，防止重复提交
);
```

**API 层**：
- 可使用 `IdempotencyFilter`（检查 `AdminActionLogs.Details` 中的 idempotencyKey）
- 或在 `AdminActionLogs` 表增加 `IdempotencyKey` 列（UNIQUE，可选 NULL）

---

## 18. 管理员强制退出游戏流程

```
Admin → Admin Portal: 选择 Confirmed 状态订单 → 点击"强制退出"
Admin Portal → Backend: POST /api/admin/orders/{id}/force-exit
Backend (OrderService.AdminForceExitGameAsync):
  1. 查找 GameOrder（状态必须为 Confirmed）
  2. 获取 VendorPlayerMapping → vendorPlayerId, currency
  3. 调用 IVendorAdapter.TransferAllAsync(vendorPlayerId, currency) — 一键回收
     ├── 成功 (totalAmount > 0):
     │   4a. 钱包 Credit(totalAmount, GameTransferOut) — CAS 更新
     │   5a. GameOrder.Settle(totalAmount)
     │   6a. 记录 AdminActionLog（Action="ForceExitGame", AdminId=当前管理员）
     │   7a. SignalR: BalanceUpdated + OrderStatusChanged（通知用户）
     ├── 成功但回收金额 = 0:
     │   4b. GameOrder.Cancel("管理员强制退出，一键回收金额为0")
     │   5b. 记录 AdminActionLog
     └── 失败:
         4c. 返回错误，订单保持 Confirmed
         5c. 记录 AdminActionLog（Level=Error）
```

---

## 19. Wallet 自动创建

用户首次通过 `POST /api/auth/sync-profile` 同步 Profile 时，自动创建空钱包：

```
SyncProfileAsync:
  1. 查询 UserProfile by supabaseUserId
  2. 如果不存在：
     a. 创建 UserProfile（Role=User, Status=Active）
     b. 创建 Wallet（Balance=0, FrozenBalance=0, Version=0）
     c. 保存（同一事务）
  3. 如果已存在：
     a. 返回现有 Profile
```

---

## 20. 厂商 API 签名计算

当前接入的厂商使用 HTTP Header 签名认证，非回调签名：

```csharp
// 签名计算方式
// sign = MD5(random + sn + secretKey)，32 位小写
// random = 16-32 位小写字母数字随机字符串

public class VendorSignatureHelper
{
    public static (string sign, string random) ComputeSign(string sn, string secretKey)
    {
        var random = GenerateRandom(16); // 16-32 位小写字母+数字
        var raw = random + sn + secretKey;
        var sign = MD5Hash(raw); // 32 位小写 MD5
        return (sign, random);
    }
}
```

**HTTP 请求 Header**：
| Header | 说明 |
|--------|------|
| sign | MD5(random + sn + secretKey)，32 位小写 |
| random | 16-32 位小写字母数字随机字符串 |
| sn | 商户前缀（从厂商后台获取） |
| Content-Type | application/json |

---

## 21. 厂商 API 错误码映射

| 厂商错误码 | 含义 | 平台处理策略 |
|-----------|------|-------------|
| 10000 | 成功 | 正常处理 |
| 10001 | 请求失败 | 记录日志，返回错误 |
| 10002 | 账号已存在 | CreatePlayer 时视为成功（幂等） |
| 10003 | 账号不存在 | 触发 CreatePlayer 后重试 |
| 10004 | 账号格式错误 | 记录日志 + 告警（VendorPlayerId 生成逻辑有误） |
| 10005 | 额度转换失败 | 转账明确失败，执行退款流程 |
| 10006 | 转换金额错误 | 校验金额格式后重试（需 string, 2 位小数, ≥1） |
| 10009 | 接口请求频繁 | 等待后重试 + 触发 VendorRateLimitExceeded 告警 |
| 10011 | 订单号不符合要求 | 校验 orderId 格式（32 位字母数字） |
| 10012 | 订单号重复 | 幂等处理：查询 transferStatus 确认结果 |
| 10013 | 订单号不存在 | transferStatus 查询失败，视为转账不存在 |
| 10014 | 额度不足 | 商户余额不足告警 + 通知管理员充值 |
| 10403 | IP 限制访问 | 服务器 IP 未加白名单，联系厂商 |
| 10404 | 签名验证失败 | 检查 sn/secretKey 配置 |
| 10405 | 缺少参数 | 检查请求参数完整性 |
| 10407 | 游戏平台错误 | 检查 platType 是否在支持列表内 |
| 10408 | 游戏类型错误 | 检查 gameType 值 |
| 10409 | 转换类型错误 | 检查 type 值（1=转入, 2=转出） |
| 其他 | 未知错误 | 对于转账操作：必须调用 transferStatus 确认 |

---

## 22. 厂商 API 限流策略

| 接口 | 限频规则 | 平台处理方式 |
|------|---------|-------------|
| `/api/server/balanceAll` | 每玩家每分钟 ≤ 3 次 | 内存计数器 + 排队等待 |
| `/api/server/transferAll` | 每玩家每分钟 ≤ 2 次 | 内存计数器 + 排队等待 |
| `/api/server/transfer` | 无明确限频，超时 > 60s | 配置 65s 超时 |
| `/api/server/recordAll` | 每分钟 ≤ 1 次（不含分页） | BetRecordSyncService 间隔 2 分钟 |
| `/api/server/recordHistory` | 每小时 ≤ 5 次，间隔 ≥ 1 分钟 | 仅管理员手动触发时使用 |
| `/api/server/gameCode` | 每小时 ≤ 30 次 | IMemoryCache 缓存 10 分钟 |

**实现方案**：使用 `SemaphoreSlim` + `MemoryCache` 在 Infrastructure 层的 VendorAdapter 内实现客户端限流。收到 10009 错误码时，等待 10 秒后重试（最多 2 次）。

---

## 23. Dashboard 统计 DTO

### 23.1 管理端仪表盘

```csharp
public record DashboardStatsDto(
    int TotalUsers,                    // 总用户数
    int TodayOrders,                   // 今日订单数
    decimal TodayDepositTotal,         // 今日充值总额
    decimal TodayWithdrawTotal,        // 今日提现总额
    int PendingDepositRequests,        // 待审核充值申请数
    int PendingWithdrawRequests,       // 待审核提现申请数
    int TodayNewUsers                  // 今日新注册用户数
);
```

### 23.2 代理端仪表盘

```csharp
public record AgentDashboardStatsDto(
    int SubUserCount,                  // 下级用户数
    int ActiveOrders,                  // 活跃订单数
    decimal TodayDepositTotal,         // 下级今日充值总额
    decimal TodayWithdrawTotal         // 下级今日提现总额
);
```

---

## 24. 代理端报表 DTO

```csharp
public record AgentReportSummaryDto(
    DateTime From,                                      // 统计起始时间
    DateTime To,                                        // 统计结束时间
    int TotalUsers,                                     // 下级用户总数
    int ActiveUsers,                                    // 期间活跃用户数
    decimal TotalDeposit,                               // 期间充值总额
    decimal TotalWithdraw,                              // 期间提现总额
    decimal TotalBetAmount,                             // 期间投注总额
    decimal TotalValidBetAmount,                        // 期间有效投注总额
    decimal TotalWinLoss,                               // 期间盈亏总额
    int TotalOrders,                                    // 期间订单总数
    IReadOnlyList<PlatformSummary> PlatformBreakdown    // 按平台分组汇总
);

public record PlatformSummary(
    string PlatType,           // 平台标识（如 "ag", "pg"）
    decimal BetAmount,         // 该平台投注总额
    decimal ValidBetAmount,    // 该平台有效投注总额
    decimal WinLoss,           // 该平台盈亏
    int OrderCount             // 该平台订单数
);
```

---

## 25. 前端 ErrorCode → 中文提示映射

后端 API 返回的 `ErrorCode` 字段，前端统一映射为用户友好的中文提示：

| ErrorCode | HTTP | 中文提示 |
|-----------|------|----------|
| `ACTIVE_ORDER_EXISTS` | 400 | 已有进行中的游戏，请先退出当前游戏 |
| `INSUFFICIENT_BALANCE` | 400 | 余额不足，请先充值 |
| `INVALID_AMOUNT` | 400 | 金额不在允许范围内 |
| `DAILY_LIMIT_EXCEEDED` | 400 | 超过每日提现限额 |
| `INVALID_STATE_TRANSITION` | 400 | 操作不允许（当前状态不支持此操作） |
| `CONCURRENCY_CONFLICT` | 409 | 操作冲突，请重试 |
| `VENDOR_UNAVAILABLE` | 503 | 游戏服务暂不可用，请稍后重试 |
| `VENDOR_ERROR` | 502 | 游戏服务异常，请联系客服 |
| `FORBIDDEN` | 403 | 无权限执行此操作 |
| `MAX_BANK_CARDS_EXCEEDED` | 400 | 最多绑定 5 张银行卡 |
| `INVALID_CARD_NUMBER` | 400 | 银行卡号格式不正确 |
| `DEPOSIT_REQUEST_ALREADY_REVIEWED` | 400 | 该充值申请已审核 |
| `WITHDRAW_REQUEST_ALREADY_REVIEWED` | 400 | 该提现申请已审核 |

**前端实现方式**：

```typescript
// lib/api/error-messages.ts
const ERROR_MESSAGES: Record<string, string> = {
  ACTIVE_ORDER_EXISTS: '已有进行中的游戏，请先退出当前游戏',
  INSUFFICIENT_BALANCE: '余额不足，请先充值',
  // ...其他映射
};

export function getErrorMessage(errorCode?: string, fallback?: string): string {
  if (errorCode && ERROR_MESSAGES[errorCode]) {
    return ERROR_MESSAGES[errorCode];
  }
  return fallback ?? '操作失败，请稍后重试';
}
```

---

## 26. 前端 API 客户端设计

前端统一的 API 调用封装，处理认证、错误拦截、响应解析：

```typescript
// lib/api/client.ts
import { createClient } from '@supabase/ssr';

const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL;

interface ApiResponse<T> {
  success: boolean;
  data: T | null;
  error: string | null;
  errorCode?: string;
  meta?: PaginationMeta;
}

interface PaginationMeta {
  page: number;
  pageSize: number;
  totalCount: number;
  totalPages: number;
}

class ApiClient {
  private async getToken(): Promise<string | null> {
    const supabase = createClient(/* config */);
    const { data } = await supabase.auth.getSession();
    return data.session?.access_token ?? null;
  }

  async request<T>(path: string, options?: RequestInit): Promise<ApiResponse<T>> {
    const token = await this.getToken();
    const response = await fetch(`${API_BASE_URL}${path}`, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...(token ? { Authorization: `Bearer ${token}` } : {}),
        ...options?.headers,
      },
    });

    // 401: Token 过期，自动跳转登录
    if (response.status === 401) {
      window.location.href = '/login';
      throw new Error('登录已过期');
    }

    return response.json() as Promise<ApiResponse<T>>;
  }

  // GET / POST / PUT / DELETE 便捷方法
  async get<T>(path: string): Promise<ApiResponse<T>>;
  async post<T>(path: string, body: unknown): Promise<ApiResponse<T>>;
  async put<T>(path: string, body: unknown): Promise<ApiResponse<T>>;
  async del<T>(path: string): Promise<ApiResponse<T>>;
}

export const apiClient = new ApiClient();
```

**错误处理流程**：
1. API 调用 → 获取 `ApiResponse`
2. 检查 `response.success` → `false` 时调用 `getErrorMessage(response.errorCode)` 获取中文提示
3. 通过 Toast 组件展示错误信息
4. 409 (CONCURRENCY_CONFLICT) 可自动重试一次

---

## 27. 约束引用

- 模块功能定义 → 参见 `01-概要功能设计-overview.md`
- 技术架构和分层 → 参见 `02-技术架构设计-architecture.md`
- 页面交互设计 → 参见 `04-UI-UX设计-uiux_design.md`
- 测试场景设计 → 参见 `05-测试设计-testing_design.md`
- CI/CD 流程 → 参见 `06-CI-CD设计-ci_cd_design.md`

---

## 28. API 契约详细定义

> **命名规则**：后端 C# record 使用 PascalCase，.NET JSON 序列化配置为 camelCase，
> 前端收到的 JSON 字段名为 camelCase。本节同时给出后端 DTO 和前端 TypeScript 接口定义，
> 作为前后端统一契约。

### 28.0 通用响应格式

所有端点返回统一信封（详见 §3.0）：

```typescript
// 前端收到的 JSON 结构
interface ApiResponse<T> {
  success: boolean;
  data: T | null;
  error: string | null;
  errorCode?: string;         // 仅错误时
  meta?: PaginationMeta;      // 仅分页端点
}

interface PaginationMeta {
  page: number;               // 当前页码（从 1 开始）
  pageSize: number;           // 每页条数
  totalCount: number;         // 总记录数
  totalPages: number;         // 总页数
}
```

**前端分页数据提取约定**：

```typescript
// 前端应从 ApiResponse 中提取为内部 PagedResult 格式
interface PagedResult<T> {
  items: T[];                 // = response.data（分页端点中 data 为数组）
  total: number;              // = response.meta.totalCount
  page: number;               // = response.meta.page
  limit: number;              // = response.meta.pageSize
}
```

### 28.1 认证 API

#### POST /api/auth/sync-profile

同步 Supabase 用户到平台 UserProfile。

- **请求**：无 body（从 JWT token 中提取用户信息）
- **响应 data**：`UserProfileDto`

#### GET /api/auth/me

获取当前登录用户信息。

- **请求**：无参数
- **响应 data**：`UserProfileDto`

```typescript
interface UserProfile {
  id: string;                 // Guid
  username: string;
  displayName: string;
  role: string;               // "User" | "Agent" | "Admin"
  status: string;             // "Active" | "Disabled"
  agentId: string | null;     // 上级代理 ID
  createdAt: string;          // ISO 8601
}
```

### 28.2 钱包 API（User）

#### GET /api/wallet/balance

- **响应 data**：`BalanceDto`

```typescript
interface BalanceDto {
  userId: string;
  balance: number;
  frozenBalance: number;
  availableBalance: number;   // balance - frozenBalance
}
```

#### GET /api/wallet/transactions?page=&pageSize=

- **查询参数**：`page`（int, 默认 1）、`pageSize`（int, 默认 20）
- **响应 data**：`TransactionDto[]`（分页，含 meta）

```typescript
interface TransactionDto {
  id: string;
  walletId: string;
  type: string;               // TransactionType 枚举的字符串表示
  amount: number;
  balanceBefore: number;
  balanceAfter: number;
  description: string;
  createdAt: string;
}
```

#### POST /api/wallet/deposit-request

用户提交充值申请。

- **请求 body**：

```typescript
interface CreateDepositRequest {
  amount: number;
  description?: string;       // 充值备注
  proofImageUrl?: string;     // 凭证图片 URL
}
```

- **响应 data**：`DepositRequestDto`

#### GET /api/wallet/deposit-requests?page=&pageSize=

- **响应 data**：`DepositRequestDto[]`（分页，含 meta）

```typescript
interface DepositRequestDto {
  id: string;
  userId: string;
  amount: number;
  status: string;             // "Pending" | "Approved" | "Rejected"
  description: string | null;
  proofImageUrl: string | null;
  adminNote: string | null;
  requestedAt: string;        // 申请时间
  processedAt: string | null; // 审核时间
}
```

#### POST /api/wallet/withdraw-request

用户提交提现申请。

- **请求 body**：

```typescript
interface CreateWithdrawRequest {
  amount: number;
  bankCardId: string;         // 银行卡 ID
}
```

- **响应 data**：`WithdrawRequestDto`

#### GET /api/wallet/withdraw-requests?page=&pageSize=

- **响应 data**：`WithdrawRequestDto[]`（分页，含 meta）

```typescript
interface WithdrawRequestDto {
  id: string;
  userId: string;
  amount: number;
  bankCardId: string;         // 关联银行卡 ID
  status: string;             // "Pending" | "Approved" | "Rejected" | "Completed"
  adminNote: string | null;
  requestedAt: string;        // 申请时间
  processedAt: string | null; // 审核时间
}
```

#### GET /api/wallet/bank-cards

- **响应 data**：`BankCardDto[]`

```typescript
interface BankCardDto {
  id: string;
  userId: string;
  bankName: string;
  cardNumberLast4: string;    // 仅后4位
  cardHolderName: string;
  branchName: string | null;
  isDefault: boolean;
  createdAt: string;
}
```

#### POST /api/wallet/bank-cards

- **请求 body**：

```typescript
interface AddBankCardRequest {
  bankName: string;
  cardNumber: string;         // 完整卡号（后端仅存储后4位）
  cardHolderName: string;
  branchName?: string;
}
```

- **响应 data**：`BankCardDto`

#### DELETE /api/wallet/bank-cards/{id}

- **响应**：成功返回 `{ success: true, data: null }`

#### PUT /api/wallet/bank-cards/{id}/default

- **响应**：成功返回 `{ success: true, data: null }`

### 28.3 游戏 API（User）

#### GET /api/games?platType=

游戏列表（按平台查询）。

- **查询参数**：`platType`（string, 可选）
- **响应 data**：`GameInfo[]`

```typescript
interface GameInfo {
  platType: string;           // 平台标识（如 "ag", "pg"）
  gameCode: string;
  gameType: string;
  gameName: string;
  ingress: string;
  localizedNames: Record<string, string> | null;
  thumbnailUrl: string | null;
}
```

#### GET /api/games/platforms

支持的游戏平台列表。

- **响应 data**：`string[]`（platType 列表，如 `["ag", "pg", "bbin"]`）

#### POST /api/games/launch

启动游戏。

- **请求 body**：

```typescript
interface LaunchGameRequest {
  vendorCode: string;
  platType: string;
  gameCode: string;
  gameType: string;
  amount: number;             // 转入金额
  ingress: string;            // "device1"(PC) 或 "device2"(移动端)
  returnUrl?: string;         // 退出游戏后跳转地址
}
```

- **响应 data**：`LaunchResultDto`

```typescript
interface LaunchResultDto {
  orderId: string;
  gameUrl: string;
  transferInAmount: number;
  vendorPlayerId: string;
}
```

#### POST /api/games/exit

退出游戏（一键回收）。

- **请求 body**：`{ orderId: string }`
- **响应 data**：`ExitResultDto`

```typescript
interface ExitResultDto {
  orderId: string;
  finalStatus: string;        // OrderStatus 枚举
  transferOutAmount: number;
  newBalance: number;
}
```

#### GET /api/games/bet-records?page=&pageSize=

个人投注记录。

- **响应 data**：`BetRecordDto[]`（分页，含 meta，字段同 §28.5 全局投注记录）

### 28.4 订单 API（User）

#### GET /api/orders?page=&pageSize=

- **响应 data**：`OrderDto[]`（分页，含 meta）

#### GET /api/orders/{id}

- **响应 data**：`OrderDetailDto`

```typescript
interface OrderDto {
  id: string;
  userId: string;
  vendorCode: string;
  platType: string;
  gameCode: string;
  internalOrderId: string;
  externalOrderId: string | null;
  transferInAmount: number;       // 转入金额
  transferOutAmount: number | null; // 转出金额（结算后）
  status: string;                 // "Pending" | "Confirmed" | "Settled" | "Cancelled" | "Failed"
  gameUrl: string | null;
  errorMessage: string | null;
  cancelReason: string | null;
  createdAt: string;
  updatedAt: string;
}
```

> `OrderDetailDto` 与 `OrderDto` 字段相同。

### 28.5 管理 API（Admin）

#### GET /api/admin/dashboard/stats

- **响应 data**：`DashboardStatsDto`（定义见 §23.1）

#### GET /api/admin/users?filter=&page=&pageSize=

- **查询参数**：`filter`（string, 用户名/ID模糊搜索）、`page`、`pageSize`
- **响应 data**：`UserProfileDto[]`（分页，含 meta）

#### GET /api/admin/users/{id}

- **响应 data**：`UserProfileDto`

#### POST /api/admin/users/{id}/suspend

禁用用户。

- **响应**：`{ success: true, data: null }`

#### POST /api/admin/users/{id}/activate

启用用户。

- **响应**：`{ success: true, data: null }`

#### GET /api/admin/deposits?page=&pageSize=

充值申请列表。

- **响应 data**：`DepositRequestDto[]`（分页，含 meta）

#### GET /api/admin/withdrawals?page=&pageSize=

提现申请列表。

- **响应 data**：`WithdrawRequestDto[]`（分页，含 meta）

#### GET /api/admin/orders?filter=&page=&pageSize=

- **响应 data**：`OrderDto[]`（分页，含 meta）

#### POST /api/admin/deposit-requests/{id}/approve

审批充值申请。

- **响应**：`{ success: true, data: null }`

#### POST /api/admin/deposit-requests/{id}/reject

拒绝充值申请。

- **请求 body**：`{ reason: string }`
- **响应**：`{ success: true, data: null }`

#### POST /api/admin/withdraw-requests/{id}/approve

审批提现申请。

- **响应**：`{ success: true, data: null }`

#### POST /api/admin/withdraw-requests/{id}/reject

拒绝提现申请。

- **请求 body**：`{ reason: string }`
- **响应**：`{ success: true, data: null }`

#### GET /api/admin/wallets/{userId}

用户钱包信息。

- **响应 data**：`BalanceDto`

#### GET /api/admin/wallets/{userId}/transactions?page=&pageSize=

- **响应 data**：`TransactionDto[]`（分页，含 meta）

#### POST /api/admin/wallets/{userId}/adjust

人工调整余额。

- **请求 body**：

```typescript
interface AdjustWalletRequest {
  amount: number;
  type: "Deposit" | "Withdraw";
  reason: string;
}
```

- **响应**：`{ success: true, data: null }`

#### POST /api/admin/orders/{id}/force-exit

强制退出游戏（仅 Confirmed 状态订单）。

- **响应 data**：`ExitResultDto`

#### POST /api/admin/reconciliation/trigger

触发对账。

- **响应 data**：`ReconcileResultDto`

```typescript
interface ReconcileResultDto {
  totalWallets: number;
  inconsistentCount: number;
  inconsistencies: WalletInconsistency[];
}

interface WalletInconsistency {
  walletId: string;
  userId: string;
  walletBalance: number;
  transactionSum: number;
  difference: number;
}
```

#### GET /api/admin/vendor/balance

查询商户在厂商的余额。

- **响应 data**：`VendorBalanceDto[]`

```typescript
interface VendorBalanceDto {
  vendorCode: string;
  balance: number;
  currency: string;
  updatedAt: string;
}
```

#### GET /api/admin/bet-records?page=&pageSize=&userId=&platType=

全局投注记录。

- **查询参数**：`page`、`pageSize`、`userId`（可选）、`platType`（可选）
- **响应 data**：`BetRecordDto[]`（分页，含 meta）

```typescript
interface BetRecordDto {
  id: string;
  userId: string;
  vendorPlayerId: string;
  platType: string;
  vendorBetOrderId: string;
  betAmount: number;
  validBetAmount: number;
  winLossAmount: number;
  betTime: string;
}
```

#### GET /api/admin/logs/vendor-query?page=&pageSize=

厂商查询日志。

- **响应 data**：`VendorQueryLogDto[]`（分页，含 meta）

```typescript
interface VendorQueryLogDto {
  id: string;
  orderId: string;
  vendorCode: string;
  attempt: number;
  queryResult: string | null;
  isSuccess: boolean;
  errorMessage: string | null;
  createdAt: string;
}
```

#### GET /api/admin/logs/admin-action?page=&pageSize=

管理员操作日志。

- **响应 data**：`AdminActionLogDto[]`（分页，含 meta）

```typescript
interface AdminActionLogDto {
  id: string;
  adminId: string | null;
  action: string;
  targetType: string;
  targetId: string;
  details: string | null;
  level: string;              // "Info" | "Warning" | "Error"
  createdAt: string;
}
```

### 28.6 代理 API（Agent）

#### GET /api/agent/dashboard/stats

- **响应 data**：`AgentDashboardStatsDto`（定义见 §23.2）

### 28.7 JWT 认证设计

**Supabase Auth JWT 验证要求**：

| 配置项 | 值 |
|--------|-----|
| 签名算法 | ES256（ECDSA） |
| 验证方式 | OIDC Discovery（JWKS 公钥自动获取） |
| Authority | `{SUPABASE_URL}/auth/v1` |
| MetadataAddress | `{SUPABASE_URL}/auth/v1/.well-known/openid-configuration` |
| Audience | `authenticated` |
| Issuer | `{SUPABASE_URL}/auth/v1` |

**角色提取规则**：

Supabase JWT 的顶层 `role` claim 固定为 `"authenticated"`，应用角色存储在 `app_metadata.role` 中。
后端需在 JWT 验证成功后（`OnTokenValidated` 事件），从 `app_metadata` JSON 中解析 `role` 字段，
添加为标准 `role` claim，供 `[Authorize(Roles = "...")]` 使用。

角色值统一为小写：`"user"` | `"agent"` | `"admin"`。

---

## 29. 字段命名规范

### 29.1 命名转换规则

| 层级 | 命名风格 | 示例 |
|------|---------|------|
| 后端 C# record | PascalCase | `TransferInAmount` |
| 前端 JSON | camelCase | `transferInAmount` |
| 查询参数 | camelCase | `pageSize`, `platType`, `filter` |

转换关系：后端 PascalCase → JSON 序列化自动转 camelCase，无例外。

### 29.2 易混淆字段对照

| DTO | 字段 | 说明 |
|-----|------|------|
| `UserProfileDto` | `displayName` | 用户显示名（非 email、非 nickname） |
| `OrderDto` | `transferInAmount` | 转入金额（非 amount） |
| `DepositRequestDto` / `WithdrawRequestDto` | `requestedAt` | 申请时间（非 createdAt） |
| `WithdrawRequestDto` | `bankCardId` | 关联银行卡 ID（非卡号、非银行名） |
| 查询参数 | `filter` | 模糊搜索参数（非 search） |
| 查询参数 | `pageSize` | 每页条数（非 limit） |
| `PaginationMeta` | `totalCount` | 总记录数（非 total） |

---

## 30. 设计文档维护规范

1. **新增 API 端点时**，必须同时在 §28 中添加完整的请求/响应字段定义
2. **DTO 字段变更时**，必须同步更新 §28 对应的 TypeScript 接口定义
3. **前端开发时**，必须参照 §28 的 TypeScript 接口定义
4. **字段命名规则**：后端 PascalCase → 前端 camelCase，无例外
5. **路径变更时**，必须更新 §3.x 的端点表格
