# Implementation Plan: mvp-account（user 服务）

## Overview

按 design 的三层单向依赖（facade → account → repository 抽象 ← pgrepo）增量落地，组装点唯一落在 `cmd/user-service/main.go`。实现语言 **Go 1.27+**，依赖按 design「依赖清单」精确 pin（`=x.y.z`，不开 caret）。

执行顺序遵循测试驱动：先接口/模型/错误集 → 核逻辑 + 对应 PBT（内存 Repository/Hasher/Signer + 假 Clock，不依赖 docker，必须真绿）→ pgx 实现 + 容器化集成测试（无 docker 时如实标「待验证」，不影响核 PBT 全绿）→ 门面 5 个 tool → 配置 fail fast → 组装 + 端到端 + 静态依赖硬门 → build+test 全绿收口。

三道架构硬门贯穿本计划：
- **静态依赖硬门**：`internal/account`（核）不得 import 任何 MCP 包、不得 import pgx、不得 import community/collab；自动化可跑（Task 11.1）。
- **14 条 Correctness Properties 用 rapid 真跑**：每条 P1–P14 一个 property-based test（`rapid.Check` + rapid 生成器），最少 100 次迭代，注释带 `// Feature: mvp-account, Property {n}: {text}` 标签，核的 PBT 在内存实现上必须全绿。
- **收口全绿硬门**：`go build` + `go test` 全绿、建表迁移就绪、配置 fail fast（Task 12）。

## Tasks

- [ ] 1. 初始化 module、依赖锁定与核契约骨架
  - [x] 1.1 创建 Go module 与精确锁定依赖
    - 在 `services/user/` 创建 `go.mod`，module 名 `github.com/xt-line/maker-kit/maker-cloud/services/user`，`go 1.27`
    - 精确 pin（`=x.y.z`，不开 caret）：`github.com/jackc/pgx/v5 v5.7.2`、`github.com/golang-jwt/jwt/v5 v5.2.1`、`golang.org/x/crypto v0.31.0`、`github.com/google/uuid v1.6.0`、`github.com/modelcontextprotocol/go-sdk v1.8.0`、`pgregory.net/rapid v1.3.0`（test-only）
    - 建立 design「目录结构」的空包骨架：`cmd/user-service/`、`internal/facade/`、`internal/account/`、`internal/account/pgrepo/`、`internal/config/`、`migrations/`
    - _Requirements: 6.2_

  - [~] 1.2 定义核的领域模型与错误集
    - `internal/account/model.go`：`AccountRecord`、`Profile`（无 password/password_hash 字段）、`ProfilePatch`（`DisplayName *string`）、`Claims`（sub/iss/iat/exp）、`Contract`、`RegisterResult`、`LoginResult`
    - `internal/account/errors.go`：`ErrFieldMissing`、`ErrNameTaken`、`ErrInvalidCredentials`、`ErrInvalidToken`、`ErrNotFound`（稳定、可被下游/测试断言）
    - _Requirements: 2.6, 4.4, 3.5_

  - [~] 1.3 定义核依赖的三个替换点接口与 Account 接口
    - `internal/account/repository.go`：`Repository` 接口（Create/FindByName/FindByID/UpdateProfile，不存在→ErrNotFound，唯一冲突→ErrNameTaken 契约注释）
    - `internal/account/hasher.go`：`PasswordHasher` 接口（Hash/Verify）
    - `internal/account/signer.go`：`TokenSigner` 接口（Sign/Verify/Contract）与 `Clock` 接口
    - `internal/account/account.go`：`Account` 接口（Register/Login/GetProfile/UpdateProfile/VerificationContract）
    - _Requirements: 1.5, 6.1_

- [ ] 2. 实现替换点的内存实现（供核 PBT 用，不依赖 I/O）
  - [~] 2.1 实现内存 Repository
    - `internal/account/memrepo`（test helper 包或 `_test` 文件）：以 map 实现四方法，`Create` 对重复 name 返回 `ErrNameTaken`，`FindByName/FindByID` 不存在返回 `ErrNotFound`，`UpdateProfile` 应用 patch 并持久化
    - _Requirements: 6.1_

  - [~] 2.2 实现内存 Hasher 与假 Clock
    - 内存 `PasswordHasher`：低成本可逆编码（如前缀 + 明文派生串）满足「不等于明文、不含明文子串」契约，供 PBT 压低成本
    - 假 `Clock`：可设定「当前时刻」，支持快进，供 Property 8/12 控过期
    - _Requirements: 6.1_

- [ ] 3. 实现 argon2id Hasher 与 EdDSA Signer（真实现）
  - [~] 3.1 实现 argon2id PasswordHasher
    - `internal/account/hasher.go`：用 `golang.org/x/crypto/argon2` 实现 Hash（输出自带盐与参数的 PHC 编码串）与 Verify，参数基线 memory=64MB/iterations=3/parallelism=1/saltLen=16/keyLen=32（可配）
    - _Requirements: 2.2, 2.3_

  - [~] 3.2 实现 EdDSA TokenSigner 与 Contract
    - `internal/account/signer.go`：用 `crypto/ed25519` + `golang-jwt/jwt/v5` 实现 Sign（claims sub/iss/iat/exp，EdDSA 私钥签名）、Verify（验签 + 过期校验，失败→ErrInvalidToken）、Contract（algorithm=EdDSA、public_key=PEM、user_id_claim=sub）
    - _Requirements: 3.2, 5.1, 5.2_

  - [ ]* 3.3 为 argon2id Hasher 与 EdDSA Signer 写单元测试
    - Hash/Verify 往返、PHC 串格式；Sign/Verify 往返、篡改/过期拒绝、Contract 返回合法 PEM 公钥
    - _Requirements: 2.2, 3.2, 5.1_

- [ ] 4. 实现 Account_Core 的注册逻辑 + 注册类属性
  - [~] 4.1 实现 Register
    - `internal/account/account.go`：校验 name/password 非空白（空白→`ErrFieldMissing`）；`PasswordHasher.Hash`；核内生成 UUID v4 作 user_id；`Repository.Create`，撞唯一→`ErrNameTaken`；返回 `RegisterResult{UserID}`
    - `NewAccountCore(repo, hasher, signer, clk)` 构造
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6_

  - [ ]* 4.2 Property 2：注册后可按名查回且 user_id 一致
    - **Property 2: 注册后按名查询查回一条记录，返回 user_id 与查回记录 user_id 相等**
    - **Validates: Requirements 2.1** ｜ rapid，≥100 迭代，标签 `// Feature: mvp-account, Property 2: ...`
    - _Requirements: 2.1_

  - [ ]* 4.3 Property 3：密码哈希往返正确
    - **Property 3: 原密码 Verify 为真、任一不同密码 Verify 为假**
    - **Validates: Requirements 2.2** ｜ rapid，≥100 迭代
    - _Requirements: 2.2_

  - [ ]* 4.4 Property 4：不以明文存储密码
    - **Property 4: 存储的 password_hash 不等于明文、也不含明文子串**
    - **Validates: Requirements 2.3** ｜ rapid，≥100 迭代
    - _Requirements: 2.3_

  - [ ]* 4.5 Property 5：同名注册被拒且不产生重复
    - **Property 5: 二次注册返回账号名已存在错误，库中该名始终只有一条记录**
    - **Validates: Requirements 2.4** ｜ rapid，≥100 迭代
    - _Requirements: 2.4_

  - [ ]* 4.6 Property 6：user_id 全局唯一
    - **Property 6: 注册序列所有成功产出的 user_id 两两互不相同**
    - **Validates: Requirements 2.6, 5.3** ｜ rapid，≥100 迭代
    - _Requirements: 2.6, 5.3_

  - [ ]* 4.7 Property 7：缺字段注册被拒
    - **Property 7: 空/全空白 name 或 password 返回字段缺失错误，库中不新增记录**
    - **Validates: Requirements 2.5** ｜ rapid，≥100 迭代
    - _Requirements: 2.5_

- [ ] 5. 实现 Account_Core 的登录逻辑 + 登录类属性
  - [~] 5.1 实现 Login
    - `FindByName` 不存在→`ErrInvalidCredentials`（不提前返回「账号不存在」）；`PasswordHasher.Verify` 不匹配→同一 `ErrInvalidCredentials`；匹配→构造 claims（sub=user_id, iat=clk.Now, exp=iat+TTL）→`TokenSigner.Sign`；返回 `LoginResult{UserID, Token}`
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

  - [ ]* 5.2 Property 8：登录往返
    - **Property 8: 正确凭据登录返回与注册相同 user_id 及一枚 JWT，JWT 验签通过、sub 等于 user_id、exp 晚于签发时刻**
    - **Validates: Requirements 3.1, 3.2** ｜ rapid，≥100 迭代，用假 Clock 控时
    - _Requirements: 3.1, 3.2_

  - [ ]* 5.3 Property 9：凭据无效不可区分
    - **Property 9: 不存在账号名登录 与 存在但密码错登录 返回同一凭据无效错误（类型与对外信息一致）**
    - **Validates: Requirements 3.3, 3.4, 3.5** ｜ rapid，≥100 迭代
    - _Requirements: 3.3, 3.4, 3.5_

- [ ] 6. 实现 Account_Core 的信息读改与验签契约 + 对应属性
  - [~] 6.1 实现 GetProfile / UpdateProfile / VerificationContract
    - 核内先 `TokenSigner.Verify(token)`（验签 + 过期，失败→`ErrInvalidToken`），从 claims 取 sub 作 user_id
    - GetProfile：`FindByID`→组 `Profile`（剔除 password_hash）
    - UpdateProfile：`Repository.UpdateProfile(userID, patch)` 持久化后返回更新后 `Profile`
    - VerificationContract：返回 `TokenSigner.Contract()`
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 5.1, 5.2, 5.4_

  - [ ]* 6.2 Property 10：Profile 读取映射正确
    - **Property 10: 用 JWT 读取返回该账号自身信息（user_id 与账号名与注册一致）**
    - **Validates: Requirements 4.1** ｜ rapid，≥100 迭代
    - _Requirements: 4.1_

  - [ ]* 6.3 Property 11：信息修改写-读往返
    - **Property 11: 先改后读得到修改后的值，且该值已持久化到库中**
    - **Validates: Requirements 4.2** ｜ rapid，≥100 迭代
    - _Requirements: 4.2_

  - [ ]* 6.4 Property 12：坏 token 被拒且无副作用
    - **Property 12: 被篡改/已过期/格式非法的 JWT，读与改都返回凭证无效错误，且不改变库中任何数据**
    - **Validates: Requirements 4.3** ｜ rapid，≥100 迭代，用假 Clock 制造过期
    - _Requirements: 4.3_

  - [ ]* 6.5 Property 13：Profile 不泄露密码
    - **Property 13: Profile 结果不含密码或密码哈希（结构无该字段、序列化结果不含哈希子串）**
    - **Validates: Requirements 4.4** ｜ rapid，≥100 迭代
    - _Requirements: 4.4_

  - [ ]* 6.6 Property 14：验签契约自洽往返
    - **Property 14: 用 VerificationContract 公布的公钥与算法独立本地验签通过，从 user_id_claim 取出值等于登录账号 user_id（无需回调）**
    - **Validates: Requirements 5.2, 5.4** ｜ rapid，≥100 迭代
    - _Requirements: 5.2, 5.4_

- [ ] 7. 核收口检查点
  - [~] 7.1 Checkpoint - 核的 PBT 全绿
    - 运行 Property 2–14 的 property-based test（均 ≥100 迭代、内存实现、不依赖 docker），确保全绿；Ensure all tests pass, ask the user if questions arise.

- [ ] 8. 实现 Repository 的 pgx 实现 + 容器化集成测试
  - [~] 8.1 实现 PgAccountRepository
    - `internal/account/pgrepo/pg_repository.go`：用 `jackc/pgx/v5`（pgxpool）实现 `account.Repository` 四方法；捕获 name UNIQUE 冲突译为 `account.ErrNameTaken`；FindByName/FindByID 无行→`account.ErrNotFound`；pgx 只出现在本包
    - _Requirements: 6.1, 6.2_

  - [ ]* 8.2 针对容器化 PostgreSQL 的集成测试
    - Create/FindByName/FindByID/UpdateProfile 往返 + name UNIQUE 冲突→`ErrNameTaken` 映射
    - 无 docker 环境时用 build tag / skip 如实标注「待验证」，不影响核 PBT 全绿
    - _Requirements: 6.1, 6.2, 6.3_

- [ ] 9. 实现 MCP_Facade（壳薄，零业务判定）
  - [~] 9.1 实现 MCPServer 与 5 个 tool handler
    - `internal/facade/server.go`：`MCPServer{core account.Account}`、`NewMCPServer`、`Serve(ctx, TransportConfig)`（`modelcontextprotocol/go-sdk` 起 streamable HTTP MCP server）
    - `internal/facade/tools.go`：注册 `account.register`/`account.login`/`account.get_profile`/`account.update_profile`/`account.verification_contract` 五个 tool，每个 handler 仅「解参→调 core→译结果」，核错误映射为 MCP isError，不泄露内部细节，无业务分支
    - _Requirements: 1.1, 1.2, 1.3, 1.4_

  - [ ]* 9.2 门面单元/属性测试
    - **Property 1: 合法 MCP tool 调用后恰好调用 Core 对应方法一次、入参原样透传、门面不做业务判定**
    - **Validates: Requirements 1.3** ｜ rapid，≥100 迭代，用 spy Core
    - tool 注册表恰含 5 个约定 tool（1.2）
    - _Requirements: 1.2, 1.3_

- [ ] 10. 实现配置加载（fail fast）与建表迁移
  - [~] 10.1 实现 Config 加载与 fail fast
    - `internal/config/config.go`：从环境变量读 `USER_SIGNING_PRIVATE_KEY`（PEM）/可选 `USER_SIGNING_PUBLIC_KEY`、DB 连接凭据、TLS 开关
    - fail fast：缺私钥 / 缺 DB 凭据 / 公网未配 TLS → 启动期报错退出；dev/内网允许经配置关闭 TLS 强制；安全差异只在配置、不分叉代码
    - _Requirements: 5.5, 7.1, 7.2, 7.3, 7.4_

  - [~] 10.2 编写 accounts 表迁移
    - `migrations/0001_accounts.sql`：按 design schema 建 `accounts` 表（user_id UUID PK、name TEXT NOT NULL UNIQUE、password_hash TEXT NOT NULL、display_name TEXT NOT NULL DEFAULT ''、created_at/updated_at TIMESTAMPTZ DEFAULT now()）
    - _Requirements: 2.4, 2.6, 4.2_

  - [ ]* 10.3 Config 单元测试
    - 缺私钥/DB 凭据→启动失败；公网未配 TLS→启动失败；dev 明文可启动
    - _Requirements: 5.5, 7.1, 7.2, 7.3, 7.4_

- [ ] 11. 组装、端到端与静态依赖硬门
  - [~] 11.1 静态依赖硬门（自动化）
    - 脚本/测试断言 import 图：`internal/account`（含子逻辑，排除 pgrepo 子包）不得 import 任何 MCP 包（`modelcontextprotocol/go-sdk`）、不得 import `jackc/pgx`、不得 import community/collab
    - 用 `go list -deps` / `go/packages` 解析 import 图并断言，作为可自动跑的硬门（`go test` 一并执行）
    - _Requirements: 1.5, 6.1_

  - [~] 11.2 组装 main.go（唯一组装点）
    - `cmd/user-service/main.go`：装配 Config → pgxpool → `NewPgAccountRepository` → `NewAccountCore(repo, argon2idHasher, eddsaSigner, realClock)` → `NewMCPServer(core)` → `Serve`；依赖倒置的注入点只在此
    - _Requirements: 1.1, 1.5, 5.5, 6.1, 6.2, 7.4_

  - [ ]* 11.3 端到端集成测试（容器化）
    - 起门面（MCP over HTTP）→ `account.register` → `account.login` → `account.get_profile` → 用 `account.verification_contract` 公布的公钥离线验签，贯通整链
    - 无 docker 环境时如实标「待验证」，不影响核 PBT 全绿
    - _Requirements: 1.1, 1.3, 2.1, 3.1, 4.1, 5.2, 5.4_

- [ ] 12. 最终收口检查点（全绿硬门）
  - [~] 12.1 Checkpoint - go build + go test 全绿
    - `go build ./...` 通过；`go test ./...` 全绿（核 Property 2–14 + Property 1 均 ≥100 迭代真绿、静态依赖硬门通过）；无 docker 的集成测试如实标注「待验证」而非失败
    - Ensure all tests pass, ask the user if questions arise.

## Notes

- 标 `*` 的子任务为测试类（单元/属性/集成），可为更快 MVP 跳过；核心实现子任务不标 `*`、必须实现。
- 每个任务引用其实现/验证的需求编号（Requirements X.Y）；P1–P14 属性任务显式引用 design 对应 Property 编号与所验证需求。
- 三道硬门：静态依赖（11.1）、14 条 Property rapid ≥100 迭代全绿（4.2–6.6、9.2）、build+test 全绿收口（12.1）。
- 核的 PBT 跑在内存 Repository + 内存 Hasher + 内存 TokenSigner + 假 Clock 上，不依赖 docker、必须真绿；pgx/端到端集成测试针对容器化 PostgreSQL，无 docker 时如实标「待验证」。
- 仅覆盖 MVP 范围；OAuth / MFA / 找回密码 / 角色 / 关系 / 审计等后置项不在本计划内。
- 需求全覆盖：Req1→Task 9/11.2；Req2→Task 4；Req3→Task 5；Req4→Task 6；Req5→Task 3.2/6.1/10.1/11.2；Req6→Task 1.3/8；Req7→Task 10.1/11.2。Property 全覆盖：P1→9.2，P2–P7→4.2–4.7，P8–P9→5.2–5.3，P10–P14→6.2–6.6。

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2"] },
    { "id": 2, "tasks": ["1.3"] },
    { "id": 3, "tasks": ["2.1", "2.2", "3.1", "3.2", "10.1", "10.2"] },
    { "id": 4, "tasks": ["3.3", "4.1", "8.1", "10.3"] },
    { "id": 5, "tasks": ["4.2", "4.3", "4.4", "4.5", "4.6", "4.7", "5.1", "8.2"] },
    { "id": 6, "tasks": ["5.2", "5.3", "6.1"] },
    { "id": 7, "tasks": ["6.2", "6.3", "6.4", "6.5", "6.6", "9.1"] },
    { "id": 8, "tasks": ["9.2", "11.1", "11.2"] },
    { "id": 9, "tasks": ["11.3"] }
  ]
}
```
