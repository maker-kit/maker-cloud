# Implementation Plan: mvp-sync（collab 服务）

## Overview

按 design 的三层单向依赖（facade → syncobj → repository/verifier 抽象 ← pgrepo + EdDSAVerifier）增量落地，组装点唯一落在 `cmd/collab-service/main.go`。实现语言 **Go 1.27+**，依赖按 design「依赖清单」精确 pin（`=x.y.z`，不开 caret）。

与姊妹服务 user / community 同构（壳薄核重三层），本服务三处关键差异写死进任务：
- **领域包名为 `syncobj`，不叫 `sync`**——`sync` 是 Go 标准库包名，同名会与 `import "sync"` 冲突、给核内并发原语埋坑。核代码全部落在 `package syncobj`。
- **只验签不签发**：`TokenVerifier` 接口只有 `Verify`、无 `Sign`；验签公钥经环境变量注入、启动 fail fast 加载进 verifier。不存密码、无 argon2；`golang.org/x/crypto` 仅用于 Ed25519 公钥（PEM）处理。
- **领域本质是「带版本的乐观并发写入」**：版本裁决下沉到 Repository 的一条带 `WHERE version = base_version AND owner_user_id = $` 的条件 UPDATE（compare-and-swap），核不做「先 SELECT version 再 UPDATE」的跨语句读改——这是消灭并发丢更新的根，本计划重点守护。

执行顺序遵循测试驱动：先接口/模型/错误集 → 核逻辑 + 对应 PBT（内存 Repository + 测试 TokenVerifier + 假 Clock，内存 Repository 的 CompareAndSwap 以原子语义建模，不依赖 docker，必须真绿）→ pgx 实现 + 容器化集成测试（无 docker 时如实标「待验证」，不影响核 PBT 全绿）→ 门面 3 个 tool → 配置 fail fast → 组装 + 端到端 + 静态依赖硬门 → build+test 全绿收口。

五道架构硬门贯穿本计划：
- **静态依赖 / 零依赖硬门**：`internal/syncobj`（核，排除 `pgrepo` 子包）不得 import 任何 MCP 包（`modelcontextprotocol/go-sdk`）、不得 import `jackc/pgx`、不得 import user/community 另两服务；并静态断言 `TokenVerifier` 只有 `Verify` 无 `Sign`、领域包名为 `syncobj` 无与标准库 `sync` 的 import 冲突。用 `go list -deps` / `go/packages` 自动断言 import 图，随 `go test` 跑（Task 11.1）。
- **12 条 Correctness Properties 用 rapid 真跑**：每条 P1–P12 一个 property-based test（`rapid.Check` + rapid 生成器），最少 100 次迭代，注释带 `// Feature: mvp-sync, Property {n}: {text}` 标签，核的 PBT 在内存 Repository + 测试 TokenVerifier + 假 Clock 上必须全绿。
- **并发丢更新硬门（本服务最关键）**：Property 8——内存 Repository 的 CompareAndSwap 以原子语义建模（每对象互斥量/atomic 复刻数据库单行 UPDATE 的行级原子性），两 goroutine + WaitGroup 汇合断言恰一成功一冲突、最终 version 只 +1、丢更新不发生，内存建模的并发 PBT 必须真绿（Task 6.1 / 6.2）。
- **pgx 集成测试硬门**：针对容器化 PostgreSQL（CompareAndSwap 条件 UPDATE + RETURNING、CAS 命中 +1 / 0 行冲突 / 0 行无权限、Create 主键冲突→`ErrAlreadyExists`、两连接并发 CompareAndSwap 复核 SQL 层原子性）；无 docker 时如实标「待验证」，核 PBT 不受影响必须全绿（Task 8.2）。
- **收口全绿硬门**：`go build` + `go test` 全绿、`migrations` 建 `sync_objects` 表（含 version CAS 的条件 UPDATE 语句、owner 索引）、配置 fail fast（缺验签公钥/DB 凭据/公网未配 TLS 则启动失败）（Task 12）。

## Tasks

- [ ] 1. 初始化 module、依赖锁定与核契约骨架
  - [ ] 1.1 创建 Go module 与精确锁定依赖
    - 在 `services/collab/` 创建 `go.mod`，module 名 `github.com/xt-line/maker-kit/maker-cloud/services/collab`，`go 1.27`
    - 精确 pin（`=x.y.z`，不开 caret）：`github.com/jackc/pgx/v5 v5.7.2`、`github.com/golang-jwt/jwt/v5 v5.2.1`、`golang.org/x/crypto v0.31.0`、`github.com/google/uuid v1.6.0`、`github.com/modelcontextprotocol/go-sdk v1.8.0`、`pgregory.net/rapid v1.3.0`（test-only）
    - **不引入任何密码哈希库（无 argon2）**——collab 不存密码；`golang.org/x/crypto` 仅用于 Ed25519 公钥处理
    - 建立 design「目录结构」的空包骨架：`cmd/collab-service/`、`internal/facade/`、`internal/syncobj/`（**包名避开标准库 `sync`**）、`internal/syncobj/pgrepo/`、`internal/config/`、`migrations/`
    - _Requirements: 8.2, 8.3_

  - [ ] 1.2 定义核的领域模型与错误集
    - `internal/syncobj/model.go`：`PushInput`（SyncObjectID/BaseVersion/Data，无归属 user_id 字段——身份只来自 JWT）、`PushResult`（SyncObjectID/Version/Created）、`PullInput`（SyncObjectID/KnownVersion *int64）、`PullResult`（SyncObjectID/Data/Version/UpToDate）、`ListResult{Items []SyncObjectSummary}`、`SyncObjectSummary`（SyncObjectID/Version）、`SyncObjectRecord`（含 OwnerUserID/Data/Version/CreatedAt/UpdatedAt）、`CASOutcome`（CASApplied/CASConflict/CASNotAccessible）、`CASResult`（Outcome/NewVersion）、`Claims{UserID}`
    - `internal/syncobj/errors.go`：`ErrInvalidToken`、`ErrFieldMissing`、`ErrVersionConflict`（**携带服务端当前 version**，唯一「错误值带数据」的错误）、`ErrNotAccessible`（归属不符或对象不存在归一，堵存在性侧信道）、`ErrAlreadyExists`（主键冲突/新建撞已存在 id）（稳定、可被下游/测试断言）
    - _Requirements: 3.4, 4.1, 5.3, 7.2_

  - [ ] 1.3 定义核依赖的替换点接口与 Sync 接口
    - `internal/syncobj/repository.go`：`Repository` 接口（`Create`/`CompareAndSwap`/`FindOwned`/`ListByOwner`）；契约注释写死：`Create` 主键冲突→`ErrAlreadyExists`；`CompareAndSwap` 返回 `CASResult`（1 行→`CASApplied`+新 version、0 行且归属相符→`CASConflict`+当前 version、0 行且不存在/归属不符→`CASNotAccessible`），**归属校验并入同一条 WHERE，核不做跨语句读改**；`FindOwned` 不存在或归属不符统一→`ErrNotAccessible`；`ListByOwner` 只按 owner_user_id 查
    - `internal/syncobj/verifier.go`：`TokenVerifier` 接口（**只 `Verify(token) (Claims, error)`，无 `Sign`**）与 `Clock` 接口
    - `internal/syncobj/sync_core.go`：`Sync` 接口（`Push`/`Pull`/`List`，三方法入参均含 raw JWT）
    - _Requirements: 1.5, 2.5, 8.1_

- [ ] 2. 实现替换点的内存实现（供核 PBT 用，不依赖 I/O）
  - [ ] 2.1 实现内存 Repository（CompareAndSwap 以原子语义建模）
    - `internal/syncobj/memrepo`（test helper 包或 `_test` 文件）：以 map[sync_object_id]SyncObjectRecord 实现四方法
    - **关键**：`CompareAndSwap` 必须以**原子语义**建模——用每对象互斥量或 `sync/atomic`（属内存实现细节，不污染核）把「比较 base_version + 写入 + version+1」包成不可分操作，复刻数据库单行 UPDATE 的行级原子性，使并发属性（Property 8）在无数据库下可验
    - `Create` 对已存在 id 返回 `ErrAlreadyExists`；`FindOwned` 不存在/归属不符统一返回 `ErrNotAccessible`；`ListByOwner` 按 owner 过滤
    - _Requirements: 8.1_

  - [ ] 2.2 实现测试 TokenVerifier 与假 Clock
    - 测试 TokenVerifier：持测试私钥签 JWT 的配套 helper + 核持对应公钥的 `Verify` 实现（EdDSA），支持制造「篡改/过期/格式非法」的坏 JWT
    - 假 `Clock`：可设定「当前时刻」并快进，供 Property 3 控过期
    - _Requirements: 8.1_

- [ ] 3. 实现 EdDSA TokenVerifier（真实现，只验签）
  - [ ] 3.1 实现 EdDSAVerifier
    - `internal/syncobj/verifier.go`：`NewEdDSAVerifier(pemPublicKey, userIDClaim)`——用 `golang.org/x/crypto`/`crypto/ed25519` 解析 PEM 公钥（缺失/非法在构造期 fail fast），`Verify` 用 `golang-jwt/jwt/v5` 按契约（algorithm=EdDSA、user_id_claim=`sub`）验签 + exp 校验，失败/过期/格式非法→`ErrInvalidToken`，成功取 `sub` 作 `Claims.UserID`
    - **只验签路径，无任何 `Sign` 方法**；全程不回调 user
    - _Requirements: 2.1, 2.2, 2.3, 2.4_

  - [ ]* 3.2 为 EdDSAVerifier 写单元测试
    - 有效 JWT 验签通过并取回正确 user_id；篡改/过期/格式非法→`ErrInvalidToken`；构造期缺失/非法 PEM 公钥 fail fast
    - _Requirements: 2.1, 2.3_

- [ ] 4. 实现 Sync_Core 的验签与门面委派属性
  - [ ] 4.1 实现 Sync_Core 构造与验签前置骨架
    - `internal/syncobj/sync_core.go`：`NewSyncCore(repo, verifier, clk)` 构造；三方法统一前置「`TokenVerifier.Verify(token)`（失败→`ErrInvalidToken`，**不触达持久化**）→从 claims 取 `sub` 作请求者 user_id」的编排骨架
    - _Requirements: 2.1, 2.2, 2.3_

  - [ ]* 4.2 Property 2：验签取值正确
    - **Property 2: 用 user 私钥对任一 user_id 签发的有效 JWT，Sync_Core 本地验签后提取的请求者 user_id 等于签入该 JWT 的 user_id（全程不回调 user）**
    - **Validates: Requirements 2.1** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代，标签 `// Feature: mvp-sync, Property 2: ...`
    - _Requirements: 2.1_

  - [ ]* 4.3 Property 3：坏 JWT 被拒且无副作用
    - **Property 3: 被篡改/已过期/格式非法的 JWT，推送、拉取、列出都返回凭证无效错误，且不改变 collab_db 中任何 Sync_Object（不新增、不改版本、不改数据）**
    - **Validates: Requirements 2.3** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代，用假 Clock 制造过期
    - _Requirements: 2.3_

- [ ] 5. 实现 Sync_Core 的推送逻辑（新建 vs 更新分路 + 版本 CAS）+ 推送类属性
  - [ ] 5.1 实现 Push（字段校验 → 新建/更新分路）
    - `internal/syncobj/sync_core.go`：验签取 user_id 后校验必填——`sync_object_id` 非空且为合法 UUID（用 `github.com/google/uuid` 校验）、`base_version` 存在、`data` 非空；缺任一→`ErrFieldMissing`（**不触达持久化**）
    - 分路（由 `base_version == 0` 区分，口径写死）：
      - `base_version == 0`（新建语义）：组 `SyncObjectRecord`（owner=请求者 user_id、version=初始值 0、data=新数据）→ `Repository.Create`；成功→`{SyncObjectID, Version: 初始, Created: true}`；撞主键冲突 `ErrAlreadyExists`→译为版本冲突（`ErrVersionConflict`，回传该对象当前 version，端应改走拉取后更新）
      - `base_version > 0`（更新语义）：调 `Repository.CompareAndSwap(id, userID, baseVersion, data)`，一条条件 UPDATE 同时做归属校验 + 版本校验 + 写入 + version+1；`CASApplied`→`{SyncObjectID, Version: 新值, Created: false}`；`CASConflict`→`ErrVersionConflict`（回传当前 version，数据/version 不变）；`CASNotAccessible`→`ErrNotAccessible`
    - 检查顺序写死：验签→字段校验→归属校验+版本 CAS（归属先于版本，归属失败与对象不存在归一 `ErrNotAccessible`）
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 4.1, 4.2, 4.3, 4.4, 7.1, 7.2_

  - [ ]* 5.2 Property 4：缺字段推送被拒且无副作用
    - **Property 4: 缺少 sync_object_id、base_version 或新数据的推送输入（携带有效 JWT），推送返回字段缺失错误，且 collab_db 中不新增对象、不改动任何既有对象的数据与 version**
    - **Validates: Requirements 3.4** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代
    - _Requirements: 3.4_

  - [ ]* 5.3 Property 5：版本匹配则写入且版本单调 +1
    - **Property 5: 归属请求者的既有 Sync_Object 与等于其当前 version 的 base_version，推送成功写入新数据、返回最新 version 恰等于原 version+1，且该对象 version 历史严格单调递增、仅在每次成功写入时 +1（从不跳变、从不回退）**
    - **Validates: Requirements 3.1, 3.2, 3.5** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代
    - _Requirements: 3.1, 3.2, 3.5_

  - [ ]* 5.4 Property 6：新建往返且归属写定
    - **Property 6: 不存在的 sync_object_id、base_version=0 的新建推送与任一 user_id 签发的有效 JWT，推送创建新 Sync_Object 并以初始 version 保存；随后以同一 JWT 拉取该对象取回与推送一致的数据，其归属 user_id 等于该 JWT 的 user_id**
    - **Validates: Requirements 3.3** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代
    - _Requirements: 3.3_

  - [ ]* 5.5 Property 7：版本冲突拒绝、不覆盖、不合并、回传最新
    - **Property 7: 归属请求者的既有 Sync_Object 与不等于其当前 version 的 base_version，推送判为版本冲突：返回冲突错误并回传服务端当前最新 version、collab_db 中该对象的数据与 version 均保持不变（既不被新数据覆盖，也不与新数据做任何自动合并）**
    - **Validates: Requirements 4.1, 4.2, 4.3, 4.4** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

- [ ] 6. 并发丢更新硬门（本服务最关键）
  - [ ] 6.1 确保内存 Repository 的 CompareAndSwap 原子建模可承载并发
    - 复核 Task 2.1 的内存 `CompareAndSwap`：每对象互斥量/atomic 把「比较 base_version + 写入 + version+1」包成不可分操作，使两 goroutine 并发携带同一 base_version 对同一对象时行为与真库单行 UPDATE 一致（恰一成功一落空）
    - _Requirements: 3.5, 4.1_

  - [ ]* 6.2 Property 8：并发同 base_version 推送恰一个成功、丢更新不发生
    - **Property 8: 归属请求者的既有 Sync_Object，两次推送携带相同 base_version（均等于当前 version）并发到达时，恰有一次返回成功（version+1）、另一次返回版本冲突（回传已 +1 的最新 version），且对象最终 version 恰为原 version+1（绝不 +2、绝不发生「后写覆盖先写」的丢更新）**
    - **Validates: Requirements 3.5, 4.1** ｜ rapid（`rapid.Check` + rapid 生成器，rapid 只负责生成输入），≥100 迭代；并发断言逻辑不变——每轮用两 goroutine + `WaitGroup` 汇合后断言恰一成功一冲突、最终 version 只 +1（守护本服务最关键安全不变量，内存建模的并发 PBT 必须真绿）
    - _Requirements: 3.5, 4.1_

- [ ] 7. 实现 Sync_Core 的拉取、列表与数据隔离 + 对应属性
  - [ ] 7.1 实现 Pull 与 List
    - `Pull(token, in)`：验签取 user_id→`Repository.FindOwned(id, userID)`（数据隔离一次完成），不存在/归属不符→`ErrNotAccessible`；取回对象返回 `{Data, Version}`，携带 `KnownVersion` 且 `KnownVersion >= Version`→`UpToDate: true`，否则返回最新数据 `UpToDate: false`
    - `List(token)`：验签取 user_id→`Repository.ListByOwner(userID)`，只查 owner_user_id=请求者 user_id 的对象，返回每项 `{SyncObjectID, Version}`；核内无任何不带 owner 过滤的列表路径
    - _Requirements: 5.1, 5.2, 5.3, 6.1, 6.2, 7.1, 7.3, 7.4_

  - [ ]* 7.2 Property 9：数据隔离——非属主被拒且数据不变
    - **Property 9: 归属某 user_id 的 Sync_Object 与任一不等于其归属的请求者 JWT，该请求者对该对象的推送与拉取都被拒（返回无权限/不存在，不区分对象是否存在），且 collab_db 中该对象的数据与 version 保持不变**
    - **Validates: Requirements 7.1, 7.2, 7.4** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代（守护数据隔离与存在性侧信道不泄露）
    - _Requirements: 7.1, 7.2, 7.4_

  - [ ]* 7.3 Property 10：列表只含自己的对象且版本如实
    - **Property 10: 多归属的 Sync_Object 集合与任一请求者 JWT，列表返回的每一项其归属 user_id 都等于该 JWT 的 user_id（绝不含他人对象），该请求者名下的对象不被漏掉，且每项 version 等于该对象服务端当前 version**
    - **Validates: Requirements 6.1, 6.2, 7.3** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代
    - _Requirements: 6.1, 6.2, 7.3_

  - [ ]* 7.4 Property 11：拉取返回最新数据与版本并如实判定端是否最新
    - **Property 11: 归属请求者的既有 Sync_Object，拉取返回该对象服务端当前数据与 version；端已知 version 低于服务端 version 时返回最新数据且标记端未最新，端已知 version 不低于服务端 version 时标记端已最新**
    - **Validates: Requirements 5.1, 5.2** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代
    - _Requirements: 5.1, 5.2_

  - [ ]* 7.5 Property 12：拉取不存在被拒
    - **Property 12: 归属请求者但在 collab_db 中不存在的 sync_object_id，拉取返回同步对象不存在错误**
    - **Validates: Requirements 5.3** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代
    - _Requirements: 5.3_

- [ ] 8. 实现 Repository 的 pgx 实现 + 容器化集成测试
  - [ ] 8.1 实现 PgSyncRepository
    - `internal/syncobj/pgrepo/pg_repository.go`：用 `jackc/pgx/v5`（pgxpool）实现 `syncobj.Repository` 四方法
    - **CompareAndSwap**：一条 `UPDATE sync_objects SET data=$4, version=version+1, updated_at=now() WHERE sync_object_id=$1 AND owner_user_id=$2 AND version=$3 RETURNING version`——RETURNING 非空→`CASApplied`（version 已 +1）；RETURNING 空→以 `FindOwned` 探查成因：存在且归属相符→`CASConflict`（回传当前 version），否则→`CASNotAccessible`
    - `Create` 捕获 sync_object_id 主键唯一冲突→`syncobj.ErrAlreadyExists`（并发双新建兜底）；`FindOwned` 命中空结果（不存在/归属不符）统一→`syncobj.ErrNotAccessible`；`ListByOwner` 只查 owner_user_id
    - **pgx 只出现在本包**
    - _Requirements: 8.1, 8.2, 8.3_

  - [ ]* 8.2 针对容器化 PostgreSQL 的集成测试（含并发丢更新真库复核）
    - Create/CompareAndSwap/FindOwned/ListByOwner 往返：CAS 命中 +1、CAS 版本不匹配 0 行→`CASConflict`、归属不符 0 行→`CASNotAccessible`、Create 主键冲突→`ErrAlreadyExists` 的映射
    - **并发丢更新真库复核**：开两条连接携带同一 base_version 对同一行并发 `CompareAndSwap`，断言恰一条影响 1 行、另一条影响 0 行，行最终 version 只 +1——在真实 PG 上复核 Property 8 的 SQL 层原子性
    - 无 docker 环境时用 build tag / skip 如实标注「待验证」，不影响核 PBT 全绿
    - _Requirements: 8.2, 8.3_

- [ ] 9. 实现 MCP_Facade（壳薄，零业务判定）
  - [ ] 9.1 实现 MCPServer 与 3 个 tool handler
    - `internal/facade/server.go`：`MCPServer{core syncobj.Sync}`、`NewMCPServer`、`Serve(ctx, TransportConfig)`（`modelcontextprotocol/go-sdk` 起 streamable HTTP MCP server，TLS 开关来自 Config）
    - `internal/facade/tools.go`：注册 `sync.push`/`sync.pull`/`sync.list` 三个 tool，每个 handler 仅「解参→调 core→译结果」；三个 tool 均把 raw JWT 原样透传给核，门面自身不验签、不比对归属、不比对 version、不判定对象存在；核错误映射为 MCP isError，保留冲突回传的 version，不泄露内部细节（堆栈/SQL/JWT 内部字段）
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

  - [ ]* 9.2 门面单元/属性测试
    - **Property 1: 合法 MCP tool 调用后恰好调用 Sync_Core 对应方法一次、入参原样透传、门面不做业务判定（不验签、不比对归属、不比对版本、不判定对象存在）**
    - **Validates: Requirements 1.3, 1.4** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代，用 spy Core
    - tool 注册表恰含 `sync.push`/`sync.pull`/`sync.list` 三个约定 tool（1.2）
    - _Requirements: 1.2, 1.3, 1.4_

- [ ] 10. 实现配置加载（fail fast）与建表迁移
  - [ ] 10.1 实现 Config 加载与 fail fast
    - `internal/config/config.go`：从环境变量读验签公钥（PEM）、DB 连接凭据、TLS 开关
    - fail fast：缺验签公钥 / 缺 DB 凭据 / 公网未配 TLS → 启动期报错退出；dev/内网允许经配置关闭 TLS 强制；安全差异只在配置、不分叉代码；验签公钥不从 collab_db 或仓库读取
    - _Requirements: 2.4, 9.1, 9.2, 9.3, 9.4_

  - [ ] 10.2 编写 sync_objects 表迁移
    - `migrations/0001_sync_objects.sql`：按 design schema 建 `sync_objects` 表（sync_object_id UUID PK、owner_user_id TEXT NOT NULL、data BYTEA NOT NULL、version BIGINT NOT NULL、created_at/updated_at TIMESTAMPTZ NOT NULL DEFAULT now()），建 `idx_sync_objects_owner`；DDL 文件附版本 CAS 的条件 UPDATE 语句（`UPDATE ... WHERE sync_object_id AND owner_user_id AND version=base_version RETURNING version`）作参考/校验
    - _Requirements: 3.3, 3.5, 7.1, 8.3_

  - [ ]* 10.3 Config 单元测试
    - 缺验签公钥/DB 凭据→启动失败；公网未配 TLS→启动失败；dev 明文可启动
    - _Requirements: 2.4, 9.1, 9.2, 9.3, 9.4_

- [ ] 11. 组装、端到端与静态依赖硬门
  - [ ] 11.1 静态依赖 / 零依赖硬门（自动化）
    - 脚本/测试断言 import 图：`internal/syncobj`（核，排除 `pgrepo` 子包）不得 import 任何 MCP 包（`modelcontextprotocol/go-sdk`）、不得 import `jackc/pgx`、不得 import user/community 另两服务
    - 补充结构断言：`TokenVerifier` 无 `Sign` 方法（只验签不签发，2.5）；核无 user_id 生成逻辑（只认 JWT）；**领域包名为 `syncobj`、核内无与标准库 `sync` 的 import 冲突**（命名坑的静态守护）；`data` 为 BYTEA 且核不解析 data 内容（无字段级 diff / 自动合并路径，4.3）
    - 用 `go list -deps` / `go/packages` 解析 import 图并断言，作为可自动跑的硬门（`go test` 一并执行）
    - _Requirements: 1.5, 2.5, 4.3, 8.1_

  - [ ] 11.2 组装 main.go（唯一组装点）
    - `cmd/collab-service/main.go`：装配 Config → pgxpool → `NewPgSyncRepository` → `NewEdDSAVerifier`(公钥来自环境变量) → `NewSyncCore(repo, verifier, realClock)` → `NewMCPServer(core)` → `Serve`；依赖倒置的注入点只在此
    - _Requirements: 1.1, 1.5, 2.4, 8.1, 8.3, 9.4_

  - [ ]* 11.3 端到端集成测试（容器化）
    - 起门面（MCP over HTTP）→ `sync.push` 新建（base_version=0）→ `sync.pull` 取回 → `sync.push` 带正确 base_version 更新（version+1）→ `sync.push` 带过期 base_version（冲突、回传最新 version、数据不变）→ `sync.list` 确认名下对象与版本；并验非属主 `sync.push`/`sync.pull` 被拒（`ErrNotAccessible`），贯通整链与数据隔离
    - 无 docker 环境时如实标「待验证」，不影响核 PBT 全绿
    - _Requirements: 1.1, 1.3, 3.1, 4.1, 5.1, 6.1, 7.2_

- [ ] 12. 最终收口检查点（全绿硬门）
  - [ ] 12.1 Checkpoint - go build + go test 全绿
    - `go build ./...` 通过；`go test ./...` 全绿（核 Property 2–12 + Property 1 均 ≥100 迭代真绿、Property 8 并发 PBT 真绿、静态依赖硬门通过）；无 docker 的集成测试如实标注「待验证」而非失败
    - Ensure all tests pass, ask the user if questions arise.

## Notes

- 标 `*` 的子任务为测试类（单元/属性/集成），可为更快 MVP 跳过；核心实现子任务不标 `*`、必须实现。
- 每个任务引用其实现/验证的需求编号（Requirements X.Y）；P1–P12 属性任务显式引用 design 对应 Property 编号与所验证需求。
- 五道硬门：静态依赖/零依赖（11.1）、12 条 Property rapid ≥100 迭代全绿（4.2–4.3、5.2–5.5、6.2、7.2–7.5、9.2）、并发丢更新 Property 8 内存建模真绿（6.2）、pgx 容器化集成含并发真库复核（8.2）、build+test 全绿收口（12.1）。
- 核的 PBT 跑在内存 Repository（CompareAndSwap 以原子语义建模）+ 测试 TokenVerifier（测试私钥签 JWT、核持对应公钥验签）+ 假 Clock 上，不依赖 docker、必须真绿；pgx/端到端集成测试针对容器化 PostgreSQL，无 docker 时如实标「待验证」。
- collab 领域本质：带版本的乐观并发写入——版本 CAS 下沉 SQL 一条条件 UPDATE（`WHERE version=base AND owner=$`），命中 +1 / 0 行冲突或无权限；核不做跨语句读改；新建（base_version==0）走 Create 主键兜底、更新（>0）走 CompareAndSwap。
- collab 只验签不签发：`TokenVerifier` 只有 `Verify`、无 `Sign`；不存密码、无 argon2；data 存 BYTEA 不解析/不合并；sync_object_id 由端携带、核只校验合法 UUID；版本冲突回传服务端最新 version。领域包名 `syncobj` 避开标准库 `sync`。
- 仅覆盖 MVP 范围；实时推送 / 服务端自动合并（CRDT/OT）/ 离线变更队列 / 字段级 diff / presence 等后置项不在本计划内，且不把 collab 并入 iot-kit。
- 需求全覆盖：Req1→Task 9/11.1/11.2；Req2→Task 3.1/4.1/10.1/11.1；Req3→Task 5；Req4→Task 5；Req5→Task 7.1；Req6→Task 7.1；Req7→Task 5.1/7.1/11.1；Req8→Task 1.3/8/11.2；Req9→Task 10.1/11.2。Property 全覆盖：P1→9.2，P2–P3→4.2–4.3，P4–P7→5.2–5.5，P8→6.2，P9–P12→7.2–7.5。

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2"] },
    { "id": 2, "tasks": ["1.3"] },
    { "id": 3, "tasks": ["2.1", "2.2", "3.1", "10.1", "10.2"] },
    { "id": 4, "tasks": ["3.2", "4.1", "6.1", "8.1", "10.3"] },
    { "id": 5, "tasks": ["4.2", "4.3", "5.1", "8.2"] },
    { "id": 6, "tasks": ["5.2", "5.3", "5.4", "5.5", "6.2", "7.1"] },
    { "id": 7, "tasks": ["7.2", "7.3", "7.4", "7.5", "9.1"] },
    { "id": 8, "tasks": ["9.2", "11.1", "11.2"] },
    { "id": 9, "tasks": ["11.3"] }
  ]
}
```
