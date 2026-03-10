# CI/CD 设计

> 文档编号：06-CI-CD设计-ci_cd_design.md | 版本：1.3 | 日期：2026-03-03
> 依赖文档：architecture.md（技术栈）, testing_design.md（测试策略）

## 1. 分支策略

```
main (保护分支 — 禁止直接 push)
  ├── feature/xxx       # 功能分支
  ├── fix/xxx           # 修复分支
  └── release/x.x.x    # 发布分支
```

**规则**：
- `main` 为保护分支，禁止直接 push
- 所有变更必须通过 PR 合并
- PR 必须通过全部 CI 检查才能合并
- 测试失败禁止合并
- 建议使用 Squash Merge

---

## 2. CI 流水线（GitHub Actions）

### 2.1 Backend CI

**触发条件**：PR → main / push → feature/* / fix/*

**前置条件**：项目根目录必须包含 `.editorconfig` 文件（步骤 8 `dotnet format` 依赖此文件定义代码风格规范）

```yaml
# 流水线步骤
1. Checkout 代码
2. Setup .NET 10
3. dotnet restore
4. dotnet build --no-restore --configuration Release
5. dotnet test --no-build --configuration Release
     --collect:"XPlat Code Coverage"
     --results-directory ./coverage
6. 覆盖率检查（阈值 85%，低于则失败）
7. dotnet ef migrations script --idempotent --output /dev/null（验证 Migration 文件完整性）
8. dotnet format --verify-no-changes（代码格式检查，依赖 .editorconfig）
9. dotnet list package --vulnerable --include-transitive（依赖漏洞扫描，High/Critical 阻断）
10. 上传覆盖率报告（artifact）
```

### 2.2 Frontend CI（User Portal）

**触发条件**：PR → main / push → feature/* / fix/*（Frontend/user-portal/** 变更时）

**前置条件**：项目中必须包含 `vitest.config.ts`（步骤 6 `npm run test` 依赖此文件）、`src/test/setup.ts`（测试环境初始化）、`src/test/mocks/handlers.ts`（MSW 配置）

```yaml
# 流水线步骤
1. Checkout 代码
2. Setup Node.js 20+
3. cd Frontend/user-portal
4. npm ci
5. npm run lint
6. npm run test（Vitest 组件/Store 测试，依赖 vitest.config.ts）
7. npm audit --audit-level=high（依赖漏洞扫描，High/Critical 阻断）
8. npm run build
```

### 2.3 Frontend CI（Admin/Agent Portal）

**触发条件**：同上，路径匹配 Frontend/admin-portal/**

**前置条件**：同 §2.2（vitest.config.ts + test setup + MSW handlers）

```yaml
# 流水线步骤（同 user-portal）
1. Checkout 代码
2. Setup Node.js 20+
3. cd Frontend/admin-portal
4. npm ci
5. npm run lint
6. npm run test（Vitest 组件/Store 测试，依赖 vitest.config.ts）
7. npm audit --audit-level=high（依赖漏洞扫描）
8. npm run build
```

### 2.4 E2E CI

**触发条件**：PR → main（仅当 Backend/ 或 Frontend/ 有变更时）

```yaml
# 流水线步骤
1. Checkout 代码
2. Setup .NET 10 + Node.js 20+
3. 启动 PostgreSQL（Testcontainers 或 service container）
4. 启动 Backend（dotnet run）
5. 启动 User Portal（npm run dev）
6. 启动 Admin Portal（npm run dev）
7. npx playwright install --with-deps
8. npx playwright test
9. 上传测试报告 + 截图 + 视频（artifact）
```

---

## 3. 质量门禁

| 检查项 | 阈值 | 阻断 PR |
|--------|------|---------|
| Backend 编译 | 通过 | 是 |
| Backend 测试 | 全部通过 | 是 |
| Backend 覆盖率 | >= 85% | 是 |
| Backend 代码格式 | dotnet format 通过 | 是 |
| Backend 依赖安全 | dotnet list package --vulnerable 无 High/Critical | 是 |
| Backend Migration | ef migrations 脚本生成成功 | 是 |
| Frontend (user) lint | 通过 | 是 |
| Frontend (user) test | Vitest 通过 | 是 |
| Frontend (user) 覆盖率 | >= 80% | 是 |
| Frontend (user) 依赖安全 | npm audit 无 High/Critical | 是 |
| Frontend (user) build | 通过 | 是 |
| Frontend (admin) lint | 通过 | 是 |
| Frontend (admin) test | Vitest 通过 | 是 |
| Frontend (admin) 覆盖率 | >= 80% | 是 |
| Frontend (admin) 依赖安全 | npm audit 无 High/Critical | 是 |
| Frontend (admin) build | 通过 | 是 |
| E2E 测试 | 关键路径通过 | 是 |

---

## 4. 环境策略

| 环境 | 触发 | 数据库 | 用途 |
|------|------|--------|------|
| CI | PR / push | Testcontainers（自动创建/销毁） | 自动化测试 |
| Staging | merge → main | Supabase Staging 项目 | 预发布验证 |
| Production | 手动触发 / git tag | Supabase Production 项目 | 生产部署 |

### 4.1 环境变量管理

```
# 每个环境独立配置
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_ANON_KEY=xxx
SUPABASE_SERVICE_ROLE_KEY=xxx
DATABASE_CONNECTION_STRING=Host=xxx;Database=xxx;...
JWT_SECRET=xxx
VENDOR_API_KEY=xxx
VENDOR_API_SECRET=xxx
VENDOR_CALLBACK_SECRET=xxx
```

- CI 环境：GitHub Actions Secrets
- Staging / Production：服务器环境变量或 Secret Manager
- 各环境密钥完全隔离

---

## 5. 部署流程

### 5.1 Staging 自动部署

```
merge PR → main
  → CI 全部通过
  → 自动部署到 Staging 服务器
    1. SSH 连接 Staging
    2. git pull main
    3. Backend: dotnet publish → 重启服务
    4. Frontend: npm run build → 重启 Next.js
    5. 运行数据库 Migration（dotnet ef database update）
  → 冒烟测试（可选）
```

### 5.2 Production 手动部署

```
创建 git tag (v1.x.x)
  → CI 全部通过
  → 人工确认（GitHub Environment Protection Rules）
  → 部署到 Production
    1. 备份数据库（pg_dump）
    2. 运行 Migration
    3. 部署 Backend + Frontend
    4. 健康检查（/api/health）
    5. 冒烟测试
  → 成功: 通知团队
  → 失败: 自动/手动回滚
```

### 5.3 回滚策略

- 保留最近 3 个版本的部署产物
- Backend: 切换到上一个版本的发布目录 + 重启
- Frontend: 切换到上一个 build 产物 + 重启
- 数据库: Migration Down 方法回滚（需验证安全性）
- 紧急回滚: 直接还原上一个 git tag 的部署

---

## 6. 健康检查

### 6.1 Backend 健康检查端点

```
GET /api/health
```

返回：
- 数据库连接状态
- Supabase Auth 可达
- SignalR Hub 状态

### 6.2 部署后检查清单

- [ ] `GET /api/health` 返回 200 且所有检查项 Healthy
- [ ] 验证数据库连接（health check 自动检测）
- [ ] 前端页面可访问（user-portal + admin-portal）
- [ ] 登录流程正常（Supabase Auth）
- [ ] SignalR 连接正常（WebSocket 握手成功）
- [ ] 厂商 API 可达（可选）
- [ ] Migration 已应用到最新版本

---

## 7. 测试项目结构说明

```
Backend/tests/
├── GamePlatform.Domain.Tests/          # Domain 层纯单元测试（无 DB）
├── GamePlatform.Application.Tests/     # Application 层单元测试（Mock 依赖）
└── GamePlatform.Integration.Tests/     # 集成测试（WebApplicationFactory + Testcontainers）
                                        # 包含 API 端点测试、数据库事务测试、业务流程测试
```

> 注：架构文档中提到的 `GamePlatform.API.Tests/` 已合并到 `GamePlatform.Integration.Tests/`，因为 API 端点测试依赖完整的服务启动环境（`WebApplicationFactory`），本质是集成测试。

---

## 8. 约束引用

- 技术架构 → 参见 `02-技术架构设计-architecture.md`
- 测试策略 → 参见 `05-测试设计-testing_design.md`
- 模块定义 → 参见 `01-概要功能设计-overview.md`
