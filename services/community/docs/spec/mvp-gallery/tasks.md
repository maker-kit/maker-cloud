# Implementation Plan: mvp-gallery（community 服务）

## Overview

按 design 的三层单向依赖（facade → work → repository/verifier 抽象 ← pgrepo + EdDSAVerifier）增量落地，组装点唯一落在 `cmd/community-service/main.go`。实现语言 **Go 1.27+**，依赖按 design「依赖清单」精确 pin（`=x.y.z`，不开 caret）。

与姊妹服务 user 同构（壳薄核重三层），本服务两处关键差异写死进任务：
- **只验签不签发**：`TokenVerifier` 接口只有 `Verify`、无 `Sign`；验签公钥经环境变量注入、启动 fail fast 加载进 verifier。
- **不存密码**：无 `PasswordHasher`、无 argon2；`golang.org/x/crypto` 仅用于 Ed25519 公钥（PEM）处理。

执行顺序遵循测试驱动：先接口/模型/错误集 → 核逻辑 + 对应 PBT（内存 Repository + 测试 TokenVerifier + 假 Clock/确定 IDGenerator，不依赖 docker，必须真绿）→ pgx 实现 + 容器化集成测试（无 docker 时如实标「待验证」，不影响核 PBT 全绿）→ 门面 4 个 tool → 配置 fail fast → 组装 + 端到端 + 静态依赖硬门 → build+test 全绿收口。

四道架构硬门贯穿本计划：
- **静态依赖硬门**：`internal/work`（核）不得 import 任何 MCP 包、不得 import pgx、不得 import user/collab 另两服务；用 `go list -deps` / `go/packages` 自动断言 import 图，随 `go test` 跑（Task 11.1）。
- **13 条 Correctness Properties 用 rapid 真跑**：每条 P1–P13 一个 property-based test（`rapid.Check` + rapid 生成器），最少 100 次迭代，注释带 `// Feature: mvp-gallery, Property {n}: {text}` 标签，核的 PBT 在内存 Repository + 测试 TokenVerifier 上必须全绿；特别守护「删除归属」「work_id 唯一」「发布者 user_id 不可变」「坏 JWT 无副作用」「列表分页不重不漏」。
- **pgx 集成测试硬门**：针对容器化 PostgreSQL；无 docker 时如实标「待验证」，核 PBT 不受影响必须全绿（Task 8.2）。
- **收口全绿硬门**：`go build` + `go test` 全绿、`migrations` 建 `works` 表、配置 fail fast（缺验签公钥/DB 凭据/公网未配 TLS 则启动失败）（Task 12）。

## Tasks

- [ ] 1. 初始化 module、依赖锁定与核契约骨架
  - [ ] 1.1 创建 Go module 与精确锁定依赖
    - 在 `services/community/` 创建 `go.mod`，module 名 `github.com/xt-line/maker-kit/maker-cloud/services/community`，`go 1.27`
    - 精确 pin（`=x.y.z`，不开 caret）：`github.com/jackc/pgx/v5 v5.7.2`、`github.com/golang-jwt/jwt/v5 v5.2.1`、`golang.org/x/crypto v0.31.0`、`github.com/google/uuid v1.6.0`、`github.com/modelcontextprotocol/go-sdk v1.8.0`、`pgregory.net/rapid v1.3.0`（test-only）
    - **不引入任何密码哈希库（无 argon2）**——community 不存密码
    - 建立 design「目录结构」的空包骨架：`cmd/community-service/`、`internal/facade/`、`internal/work/`、`internal/work/pgrepo/`、`internal/config/`、`migrations/`
    - _Requirements: 7.1, 7.2_

  - [ ] 1.2 定义核的领域模型与错误集
    - `internal/work/model.go`：`PublishInput`（Title/Description/Artifacts，无 publisher user_id 字段——身份只来自 JWT）、`PublishResult{WorkID}`、`ListQuery`（PublisherUserID *string/Limit/Offset，无全文搜索参数）、`ListResult{Items []WorkSummary}`、`WorkSummary`、`WorkDetail`、`WorkRecord`、`ArtifactRef`（URL/Kind/MimeType，只引用无二进制字段）、`Claims{UserID}`
    - `internal/work/errors.go`：`ErrInvalidToken`、`ErrFieldMissing`、`ErrForbidden`、`ErrWorkNotFound`、`ErrInvalidPagination`（稳定、可被下游/测试断言）
    - _Requirements: 3.3, 4.5, 5.1_

  - [ ] 1.3 定义核依赖的替换点接口与 Work 接口
    - `internal/work/repository.go`：`Repository` 接口（Create/List/FindByID/Delete；FindByID/Delete 不存在→`ErrWorkNotFound`；Delete 只按 work_id 删、不承载归属逻辑的契约注释）
    - `internal/work/verifier.go`：`TokenVerifier` 接口（**只 `Verify(token) (Claims, error)`，无 `Sign`**）与 `Clock`、`IDGenerator` 接口
    - `internal/work/work.go`：`Work` 接口（Publish/List/Get/Delete）
    - _Requirements: 1.5, 2.5, 7.1_

- [ ] 2. 实现替换点的内存实现（供核 PBT 用，不依赖 I/O）
  - [ ] 2.1 实现内存 Repository
    - `internal/work/memrepo`（test helper 包或 `_test` 文件）：以 map[work_id]WorkRecord 实现四方法，`Create` 落库，`List` 按 PublisherUserID 过滤 + 按 created_at 稳定排序 + limit/offset 分页，`FindByID`/`Delete` 不存在返回 `ErrWorkNotFound`
    - _Requirements: 7.1_

  - [ ] 2.2 实现测试 TokenVerifier、假 Clock 与确定 IDGenerator
    - 测试 TokenVerifier：持测试私钥签 JWT 的配套 helper + 核持对应公钥的 `Verify` 实现（EdDSA），支持制造「篡改/过期/格式非法」的坏 JWT
    - 假 `Clock`：可设定「当前时刻」并快进，供 Property 3 控过期
    - 确定 `IDGenerator`：可注入确定序列以精确验 Property 6 的 work_id 唯一性
    - _Requirements: 7.1_

- [ ] 3. 实现 EdDSA TokenVerifier（真实现，只验签）
  - [ ] 3.1 实现 EdDSAVerifier
    - `internal/work/verifier.go`：`NewEdDSAVerifier(pemPublicKey, userIDClaim)`——用 `golang.org/x/crypto`/`crypto/ed25519` 解析 PEM 公钥（缺失/非法在构造期 fail fast），`Verify` 用 `golang-jwt/jwt/v5` 按契约（algorithm=EdDSA、user_id_claim=`sub`）验签 + exp 校验，失败/过期/格式非法→`ErrInvalidToken`，成功取 `sub` 作 `Claims.UserID`
    - **只验签路径，无任何 `Sign` 方法**
    - _Requirements: 2.1, 2.2, 2.3_

  - [ ]* 3.2 为 EdDSAVerifier 写单元测试
    - 有效 JWT 验签通过并取回正确 user_id；篡改/过期/格式非法→`ErrInvalidToken`；构造期缺失/非法 PEM 公钥 fail fast
    - _Requirements: 2.1, 2.3_

- [ ] 4. 实现 Work_Core 的发布逻辑 + 发布类属性
  - [ ] 4.1 实现 Publish
    - `internal/work/work.go`：`NewWorkCore(repo, verifier, clk, idgen)` 构造；`Publish(token, in)` 内先 `TokenVerifier.Verify(token)`（失败→`ErrInvalidToken`，不落库），取 `sub` 作发布者 user_id；校验 Title 非空白、Artifacts 非空（缺→`ErrFieldMissing`，不落库）；经 `IDGenerator` 本地生成 UUID v4 作 work_id、`Clock` 取 created_at；组 `WorkRecord`（发布者 user_id 写定）→ `Repository.Create`；返回 `{work_id}`
    - 产物引用只存 URL + 元信息，不存二进制本体
    - _Requirements: 2.1, 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7_

  - [ ]* 4.2 Property 2：验签取值正确
    - **Property 2: 用 user 私钥对任一 user_id 签发的有效 JWT，核本地验签后提取的操作者 user_id 等于签入该 JWT 的 user_id（全程不回调 user）**
    - **Validates: Requirements 2.1** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代，标签 `// Feature: mvp-gallery, Property 2: ...`
    - _Requirements: 2.1_

  - [ ]* 4.3 Property 3：坏 JWT 被拒且无副作用
    - **Property 3: 被篡改/已过期/格式非法的 JWT，发布与删除都返回凭证无效错误，且不改变 community_db 中任何作品数据（不新增、不删除）**
    - **Validates: Requirements 2.3** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代，用假 Clock 制造过期
    - _Requirements: 2.3_

  - [ ]* 4.4 Property 4：发布往返
    - **Property 4: 合法元信息与产物引用、任一 user_id 的有效 JWT，发布成功后按返回 work_id 查详情查回一件作品，其标题/描述/产物引用与输入一致，发布者 user_id 等于该 JWT 的 user_id**
    - **Validates: Requirements 3.1, 3.2, 5.1** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代
    - _Requirements: 3.1, 3.2, 5.1_

  - [ ]* 4.5 Property 5：缺必填字段被拒且无副作用
    - **Property 5: 标题空/全空白、或产物引用为空的发布输入（携带有效 JWT），发布返回字段缺失错误，且 community_db 中不新增任何作品**
    - **Validates: Requirements 3.4, 3.5** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代
    - _Requirements: 3.4, 3.5_

  - [ ]* 4.6 Property 6：work_id 全局唯一
    - **Property 6: 任一发布序列，所有发布成功所签发的 work_id 两两互不相同**
    - **Validates: Requirements 3.6** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代，用确定 IDGenerator 验唯一性
    - _Requirements: 3.6_

  - [ ]* 4.7 Property 7：发布者 user_id 不可变
    - **Property 7: 已发布作品在其后任意次数详情读取与列表检索后，发布者 user_id 始终等于发布时 JWT 的 user_id（核不提供修改发布者的路径）**
    - **Validates: Requirements 3.7** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代
    - _Requirements: 3.7_

- [ ] 5. 实现 Work_Core 的检索与详情逻辑 + 对应属性
  - [ ] 5.1 实现 List 与 Get
    - `List(q)`（无需 JWT）：校验分页参数合法（limit>0 且 ≤ 上限、offset≥0），非法→`ErrInvalidPagination`；经 `Repository.List` 按 PublisherUserID 过滤 + created_at 稳定排序 + limit/offset 分页，返回摘要列表；无全文搜索路径
    - `Get(workID)`（无需 JWT）：`Repository.FindByID`，不存在→`ErrWorkNotFound`，否则返回 `WorkDetail`（元信息 + 产物引用 + 发布者 user_id）
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 5.1, 5.2_

  - [ ]* 5.2 Property 8：发布者过滤只返回该发布者的作品
    - **Property 8: 多发布者作品集合与任一发布者 user_id 过滤值，带该过滤值的列表检索返回的每一项发布者 user_id 都等于该值，且该发布者作品在合法分页范围内不被漏掉**
    - **Validates: Requirements 4.1, 4.2** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代
    - _Requirements: 4.1, 4.2_

  - [ ]* 5.3 Property 9：分页不重不漏
    - **Property 9: 任一作品集合与任一合法分页参数，单页返回条数不超过 limit、结果顺序稳定，稳定顺序下逐页拼接恰好不重不漏覆盖符合条件的全集**
    - **Validates: Requirements 4.3** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代
    - _Requirements: 4.3_

  - [ ]* 5.4 Property 10：非法分页被拒
    - **Property 10: 非法分页参数（负 limit、负 offset、limit 超上限等），列表检索返回分页参数非法错误且不返回任何作品数据**
    - **Validates: Requirements 4.4** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代
    - _Requirements: 4.4_

  - [ ]* 5.5 Property 11：详情不存在被拒
    - **Property 11: community_db 中不存在的 work_id，查看详情返回作品不存在错误**
    - **Validates: Requirements 5.2** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代
    - _Requirements: 5.2_

- [ ] 6. 实现 Work_Core 的删除逻辑（归属校验前置、路径唯一）+ 删除类属性
  - [ ] 6.1 实现 Delete
    - `Delete(token, workID)`：先 `TokenVerifier.Verify(token)`（失败→`ErrInvalidToken`，不删），取 `sub` 作操作者 user_id；`Repository.FindByID`，不存在→`ErrWorkNotFound`（不删）；**归属比对**——操作者 user_id 与查回作品发布者 user_id 逐字符比对：相等→`Repository.Delete(workID)` 删除并返回成功；不等→`ErrForbidden`，不调 `Repository.Delete`，作品原样保留
    - 删除路径唯一且归属校验前置：核内无任何跳过归属比对的删除分支，不提供绕过归属校验的删除路径
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

  - [ ]* 6.2 Property 12：删除归属
    - **Property 12: 已发布作品——操作者 JWT user_id 等于发布者 user_id 时删除成功且随后查详情返回不存在；不等于时删除返回无删除权限错误且该作品仍可查看详情（不被删除）**
    - **Validates: Requirements 6.1, 6.2, 6.3, 6.5** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代（守护最关键安全不变量）
    - _Requirements: 6.1, 6.2, 6.3, 6.5_

  - [ ]* 6.3 Property 13：删除不存在被拒
    - **Property 13: community_db 中不存在的 work_id（携带有效 JWT），删除返回作品不存在错误**
    - **Validates: Requirements 6.4** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代
    - _Requirements: 6.4_

- [ ] 7. 核收口检查点
  - [ ] 7.1 Checkpoint - 核的 PBT 全绿
    - 运行 Property 2–13 的 property-based test（均 ≥100 迭代、内存 Repository + 测试 TokenVerifier + 假 Clock/确定 IDGenerator、不依赖 docker），确保全绿；Ensure all tests pass, ask the user if questions arise.

- [ ] 8. 实现 Repository 的 pgx 实现 + 容器化集成测试
  - [ ] 8.1 实现 PgWorkRepository
    - `internal/work/pgrepo/pg_repository.go`：用 `jackc/pgx/v5`（pgxpool）实现 `work.Repository` 四方法；artifacts 读写为 JSONB；work_id 唯一性靠 `works` 表 PRIMARY KEY 兜底；FindByID/Delete 命中空结果→`work.ErrWorkNotFound`；**pgx 只出现在本包**
    - _Requirements: 7.1, 7.2, 7.3_

  - [ ]* 8.2 针对容器化 PostgreSQL 的集成测试
    - Create/List/FindByID/Delete 往返（含 FindByID/Delete 命中空结果→`ErrWorkNotFound` 映射、List 过滤+分页）
    - 无 docker 环境时用 build tag / skip 如实标注「待验证」，不影响核 PBT 全绿
    - _Requirements: 7.2, 7.3_

- [ ] 9. 实现 MCP_Facade（壳薄，零业务判定）
  - [ ] 9.1 实现 MCPServer 与 4 个 tool handler
    - `internal/facade/server.go`：`MCPServer{core work.Work}`、`NewMCPServer`、`Serve(ctx, TransportConfig)`（`modelcontextprotocol/go-sdk` 起 streamable HTTP MCP server，TLS 开关来自 Config）
    - `internal/facade/tools.go`：注册 `work.publish`/`work.list`/`work.get`/`work.delete` 四个 tool，每个 handler 仅「解参→调 core→译结果」；`work.publish`/`work.delete` 把 raw JWT 原样透传给核，门面自身不验签、不比对发布者、不判定作品存在；核错误映射为 MCP isError，不泄露内部细节（堆栈/SQL/JWT 内部字段）
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

  - [ ]* 9.2 门面单元/属性测试
    - **Property 1: 合法 MCP tool 调用后恰好调用 Core 对应方法一次、入参原样透传、门面不做业务判定（不验签、不比对发布者、不判定作品存在）**
    - **Validates: Requirements 1.3** ｜ rapid（`rapid.Check` + rapid 生成器），≥100 迭代，用 spy Core
    - tool 注册表恰含 `work.publish`/`work.list`/`work.get`/`work.delete` 四个约定 tool（1.2）
    - _Requirements: 1.2, 1.3_

- [ ] 10. 实现配置加载（fail fast）与建表迁移
  - [ ] 10.1 实现 Config 加载与 fail fast
    - `internal/config/config.go`：从环境变量读验签公钥（PEM）、DB 连接凭据、TLS 开关
    - fail fast：缺验签公钥 / 缺 DB 凭据 / 公网未配 TLS → 启动期报错退出；dev/内网允许经配置关闭 TLS 强制；安全差异只在配置、不分叉代码；验签公钥不从 community_db 或仓库读取
    - _Requirements: 2.4, 8.1, 8.2, 8.3, 8.4_

  - [ ] 10.2 编写 works 表迁移
    - `migrations/0001_works.sql`：按 design schema 建 `works` 表（work_id UUID PK、publisher_user_id TEXT NOT NULL、title TEXT NOT NULL、description TEXT NOT NULL DEFAULT ''、artifacts JSONB NOT NULL、created_at TIMESTAMPTZ NOT NULL DEFAULT now()），建 `idx_works_publisher`、`idx_works_created_at`
    - _Requirements: 3.2, 3.3, 3.6, 4.2_

  - [ ]* 10.3 Config 单元测试
    - 缺验签公钥/DB 凭据→启动失败；公网未配 TLS→启动失败；dev 明文可启动
    - _Requirements: 2.4, 8.1, 8.2, 8.3, 8.4_

- [ ] 11. 组装、端到端与静态依赖硬门
  - [ ] 11.1 静态依赖硬门（自动化）
    - 脚本/测试断言 import 图：`internal/work`（核，排除 `pgrepo` 子包）不得 import 任何 MCP 包（`modelcontextprotocol/go-sdk`）、不得 import `jackc/pgx`、不得 import user/collab 另两服务
    - 补充结构断言：`TokenVerifier` 无 `Sign` 方法（只验签不签发，2.5）；核无 user_id 生成逻辑；`ArtifactRef`/artifacts 无二进制字段（3.3）；`ListQuery` 无全文搜索参数（4.5）
    - 用 `go list -deps` / `go/packages` 解析 import 图并断言，作为可自动跑的硬门（`go test` 一并执行）
    - _Requirements: 1.5, 2.5, 3.3, 4.5, 7.1_

  - [ ] 11.2 组装 main.go（唯一组装点）
    - `cmd/community-service/main.go`：装配 Config → pgxpool → `NewPgWorkRepository` → `NewEdDSAVerifier`(公钥来自环境变量) → `NewWorkCore(repo, verifier, realClock, uuidIDGen)` → `NewMCPServer(core)` → `Serve`；依赖倒置的注入点只在此
    - _Requirements: 1.1, 1.5, 2.4, 7.1, 7.3, 8.4_

  - [ ]* 11.3 端到端集成测试（容器化）
    - 起门面（MCP over HTTP）→ `work.publish` → `work.list` → `work.get` → 发布者 `work.delete` → 再 `work.get` 确认消失；并验非发布者 `work.delete` 被拒且作品仍可查看，贯通整链与归属校验
    - 无 docker 环境时如实标「待验证」，不影响核 PBT 全绿
    - _Requirements: 1.1, 1.3, 3.1, 4.1, 5.1, 6.2, 6.3_

- [ ] 12. 最终收口检查点（全绿硬门）
  - [ ] 12.1 Checkpoint - go build + go test 全绿
    - `go build ./...` 通过；`go test ./...` 全绿（核 Property 2–13 + Property 1 均 ≥100 迭代真绿、静态依赖硬门通过）；无 docker 的集成测试如实标注「待验证」而非失败
    - Ensure all tests pass, ask the user if questions arise.

## Notes

- 标 `*` 的子任务为测试类（单元/属性/集成），可为更快 MVP 跳过；核心实现子任务不标 `*`、必须实现。
- 每个任务引用其实现/验证的需求编号（Requirements X.Y）；P1–P13 属性任务显式引用 design 对应 Property 编号与所验证需求。
- 四道硬门：静态依赖（11.1）、13 条 Property rapid ≥100 迭代全绿（4.2–4.7、5.2–5.5、6.2–6.3、9.2）、pgx 容器化集成（8.2）、build+test 全绿收口（12.1）。
- 核的 PBT 跑在内存 Repository + 测试 TokenVerifier（测试私钥签 JWT、核持对应公钥验签）+ 假 Clock + 确定 IDGenerator 上，不依赖 docker、必须真绿；pgx/端到端集成测试针对容器化 PostgreSQL，无 docker 时如实标「待验证」。
- community 只验签不签发：`TokenVerifier` 只有 `Verify`、无 `Sign`；不存密码、无 argon2；产物只存引用/URL + 元信息不存二进制；列表只基础过滤/分页无全文搜索。
- 仅覆盖 MVP 范围；点赞/收藏/评论、关注关系、全文搜索/标签推荐、二创/fork、审核/举报、排行榜、对象存储等后置项不在本计划内。
- 需求全覆盖：Req1→Task 9/11.1/11.2；Req2→Task 3.1/4.1/10.1/11.1；Req3→Task 4；Req4→Task 5；Req5→Task 5.1；Req6→Task 6；Req7→Task 1.3/8/11.2；Req8→Task 10.1/11.2。Property 全覆盖：P1→9.2，P2–P7→4.2–4.7，P8–P11→5.2–5.5，P12–P13→6.2–6.3。

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2"] },
    { "id": 2, "tasks": ["1.3"] },
    { "id": 3, "tasks": ["2.1", "2.2", "3.1", "10.1", "10.2"] },
    { "id": 4, "tasks": ["3.2", "4.1", "8.1", "10.3"] },
    { "id": 5, "tasks": ["4.2", "4.3", "4.4", "4.5", "4.6", "4.7", "5.1", "8.2"] },
    { "id": 6, "tasks": ["5.2", "5.3", "5.4", "5.5", "6.1"] },
    { "id": 7, "tasks": ["6.2", "6.3", "9.1"] },
    { "id": 8, "tasks": ["9.2", "11.1", "11.2"] },
    { "id": 9, "tasks": ["11.3"] }
  ]
}
```
