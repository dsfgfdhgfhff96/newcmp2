# 游戏聚合平台

MVP游戏聚合平台项目 - Transfer Wallet模型

## 技术栈

- **后端**: .NET 9, ASP.NET Core, EF Core, PostgreSQL (Supabase)
- **前端**: Next.js 15, React 19, TypeScript 5, Tailwind CSS 4, shadcn/ui
- **认证**: Supabase Auth
- **实时通信**: SignalR

## 项目结构

```
├── Backend/                 # .NET 后端
│   ├── src/
│   │   ├── GamePlatform.API/           # HTTP API 层
│   │   ├── GamePlatform.Application/   # 业务逻辑层
│   │   ├── GamePlatform.Domain/        # 领域模型层
│   │   └── GamePlatform.Infrastructure/# 基础设施层
│   └── tests/                          # 测试项目
└── Frontend/                # Next.js 前端
    ├── user-portal/        # 用户端
    └── admin-portal/       # 管理端
```

## 开发进度

- [x] 阶段1: 项目脚手架 + Domain层
- [ ] 阶段2: Infrastructure + 数据库
- [ ] 阶段3: Application层用例
- [ ] 阶段4: API层 + 前端核心页面

## 核心约束

1. **资金安全** - 所有金额操作必须原子性
2. **数据一致性** - CAS乐观锁版本控制
3. **幂等控制** - 重复请求不产生副作用
4. **可审计性** - 所有资金/状态变更有日志
