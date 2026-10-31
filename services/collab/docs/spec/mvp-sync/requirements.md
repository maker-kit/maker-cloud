# Requirements Document

## Introduction

collab 是 maker-cloud 三服务（user / community / collab）中的用户态数据同步服务：对外负责把同一用户在 desktop / mobile / tablet 多端上的用户态数据（工程、设置）同步起来。三服务各自独立、暂不互相依赖，只共享 user 签发的 user_id 与 JWT 验签契约，不共享数据库。collab 的领域核是 Sync_Object（同步对象）与其版本。

本次 spec 的核心一句话：**立起「端能推变更、端能拉变更、端能列出自己的同步对象，并以单调版本号 + 乐观并发做冲突裁决」这条最短链路，并且这条链路以 MCP tool 形态对 agent 暴露。** 这是一个用户能把自己的工程与设置在多端之间对齐的最小可用地基。

三条不可退让的底线贯穿全文：

- **AI 原生**：collab 对外的一切能力必须以 MCP tool 暴露、能被 agent 接入。对外不先开 REST，而是先以 streamable HTTP MCP 门面呈现，复刻「壳薄核重」范式——门面只做协议编排，同步逻辑沉在核。
- **身份认 JWT、本地验签不回调**：collab 不自管账号。请求者身份一律取自 user 服务签发的 JWT——collab 用 user 公布的验签公钥在本地验签，从 JWT 取 user_id 认定身份，不回调 user 服务。user_id 来源唯一是 user 服务，collab 只认不签发。
- **collab 是用户数据同步，不是设备协议 / 设备通信**：collab 同步的是归属某 user_id 的用户态数据（工程、设置），不碰设备协议、不与设备通信——设备通信归 iot-kit。将来需要「多端即时感知他端变更」的实时推送时，**可复用 iot-kit 的传输层，但不并入 iot-kit，并自定用户数据的消息语义**；该能力属后置，MVP 不做。

MVP 只做立住这条链路所必需的能力。实时推送、服务端自动合并 / CRDT / OT、离线变更队列、字段级细粒度 diff、端在线状态 / presence 一律后置，并在文末挂明确翻盘条件。MVP 的冲突模型是「乐观并发 + 版本校验」：服务端只做版本裁决，不做任何自动合并，合并由端自行完成后重传。

## Glossary

- **collab_db**: collab 服务独占的 PostgreSQL 数据库。user / community 不直接访问 collab_db，collab 也不访问其数据库。
- **Collab_Service**: 本 spec 描述的服务整体（含 MCP 门面 + 同步核 + Repository）。
- **MCP_Facade**: Collab_Service 对外的 streamable HTTP MCP 门面。对外能力只经由 MCP tool 暴露，门面只做协议编排（壳薄），不承载同步业务逻辑。
- **Sync_Core**: 同步领域核（壳薄核重中的「核」）。承载推送变更、拉取变更、列出同步对象、版本裁决与数据隔离校验等业务逻辑，不感知 MCP 协议与传输。
- **Repository**: 同步数据持久化的抽象接口。Sync_Core 依赖 Repository 接口而非具体 PostgreSQL 实现（依赖倒置），持久化细节对核不可见。
- **Sync_Object**: 一份可同步的用户态数据单元（一份工程 / 一组设置），归属某 user_id，有自己的全局唯一标识与当前数据、当前版本号。是 collab 的领域核。
- **sync_object_id**: 一个 Sync_Object 的全局唯一标识，用于推送 / 拉取 / 列表定位。
- **归属 user_id**: 一个 Sync_Object 所属用户的身份标识，取自请求所携带 JWT 中的 user_id，由 user 服务唯一签发，collab 只认不签发。一个 Sync_Object 的归属 user_id 在创建时写定。
- **version**: 一个 Sync_Object 的单调递增版本号。每个 Sync_Object 各持一个 version，每次成功写入后 version 加一。
- **base_version**: 推送变更时端随请求带上的「本次变更所基于的版本号」，即端在本地最后一次读到该 Sync_Object 时的 version。服务端据此与当前 version 做乐观并发校验。
- **乐观并发版本校验 (Optimistic_Version_Check)**: 服务端接受写入的裁决规则——仅当请求的 base_version 等于该 Sync_Object 的服务端当前 version 时接受写入并把 version 加一；否则判为版本冲突，拒绝写入。
- **版本冲突 (Version_Conflict)**: base_version 不等于服务端当前 version 的情形。服务端拒绝本次写入、回传服务端最新 version，不自动合并、不覆盖；由端自行拉取、在端上合并后以新的 base_version 重传。
- **数据隔离 (User_Data_Isolation)**: 一个 user_id 只能推送 / 拉取 / 列出自己名下的 Sync_Object，不能访问他人名下的 Sync_Object。服务端以 JWT 中的 user_id 与 Sync_Object 的归属 user_id 比对执行隔离。
- **JWT**: user 服务签发的无状态会话凭证，内含 user_id 及签发 / 过期声明，由 user 服务私钥签名。collab 收到的请求凭此认定请求者身份。
- **JWT 本地验签 (Local_JWT_Verification)**: collab 用 user 公布的验签公钥与契约在本地校验 JWT 的签名与有效期、并提取 user_id 的过程，全程不回调 user 服务。
- **验签公钥 (Verification_Public_Key)**: user 服务公布、供 collab 本地验签 JWT 的公钥。经环境变量注入 collab，不入库、不进仓库。
- **Downstream 身份契约**: user 公布的 JWT 签名算法与 user_id 声明字段约定，collab 据此本地验签并提取请求者 user_id。
- **iot-kit**: maker-kit 中负责设备协议与设备通信的能力。collab 不碰设备协议；将来的实时推送可复用其传输层，但不并入 iot-kit。

## Requirements

### Requirement 1: 以 MCP tool 形态对外暴露能力

**User Story:** 作为接入 collab 的 agent，我想通过 MCP tool 使用 collab 的全部对外能力，这样我不依赖任何私有 REST 协议即可完成同步相关操作。

#### Acceptance Criteria

1. THE MCP_Facade SHALL 以 streamable HTTP MCP 门面形态对外提供服务。
2. THE MCP_Facade SHALL 将 Collab_Service 的每一项对外能力（推送变更、拉取变更、列出同步对象）各暴露为一个 MCP tool。
3. WHEN MCP_Facade 收到一个 MCP tool 调用, THE MCP_Facade SHALL 将请求转交 Sync_Core 处理并仅做协议层编排。
4. THE MCP_Facade SHALL NOT 承载推送、拉取或列表的业务判定逻辑。
5. THE Sync_Core SHALL 独立于 MCP 协议运行，使同步逻辑不感知传输形态。

### Requirement 2: 本地验签 JWT 认定请求者身份

**User Story:** 作为 collab 服务，我想用 user 公布的公钥在本地验证请求携带的 JWT 并取出 user_id，这样我无需回调 user 服务即可认定请求者身份。

#### Acceptance Criteria

1. WHEN 收到一个携带 JWT 的请求, THE Sync_Core SHALL 以验签公钥按 Downstream 身份契约在本地验证 JWT 的签名与有效期，并从 JWT 中提取 user_id 作为请求者身份。
2. WHILE Sync_Core 验证一枚 JWT, THE 验证过程 SHALL 仅依赖 user 公布的验签公钥与身份契约，不回调 user 服务。
3. IF 请求携带的 JWT 验签不通过或已过期, THEN THE Sync_Core SHALL 拒绝请求并返回凭证无效的错误。
4. THE Collab_Service SHALL 经环境变量获取验签公钥，不从 collab_db 或仓库读取验签公钥。
5. THE Collab_Service SHALL NOT 自行签发 user_id，请求者身份一律取自 user 服务签发的 JWT。

### Requirement 3: 推送变更（上传）

**User Story:** 作为一个在某端编辑了工程或设置的用户，我想把本端的变更推送到云端，这样我的其他端能同步到这份最新数据。

#### Acceptance Criteria

1. WHEN 收到一个携带有效 JWT、sync_object_id、base_version 与新数据的推送请求, THE Sync_Core SHALL 以 base_version 与该 Sync_Object 的服务端当前 version 做乐观并发版本校验。
2. WHILE 推送请求的 base_version 等于该 Sync_Object 的服务端当前 version, THE Sync_Core SHALL 经 Repository 将新数据写入 collab_db、把该 Sync_Object 的 version 加一，并返回写入后的最新 version。
3. WHERE 推送请求所指 sync_object_id 在 collab_db 中尚不存在且请求者有权创建, THE Sync_Core SHALL 创建一个新的 Sync_Object、将 JWT 中的 user_id 记为其归属 user_id，并以初始 version 保存新数据。
4. IF 推送请求缺少 sync_object_id、base_version 或新数据, THEN THE Sync_Core SHALL 拒绝推送并返回字段缺失的错误。
5. THE Sync_Core SHALL 保证每个 Sync_Object 的 version 单调递增，且仅在成功写入后加一。

### Requirement 4: 版本冲突裁决（乐观并发、服务端不自动合并）

**User Story:** 作为用户，我想在我推送的变更基于过期版本时被明确拒绝并拿到服务端最新版本，这样我能在端上合并后再重传，而不会用旧数据覆盖云端。

#### Acceptance Criteria

1. IF 推送请求的 base_version 不等于该 Sync_Object 的服务端当前 version, THEN THE Sync_Core SHALL 判为版本冲突、拒绝本次写入，并回传该 Sync_Object 的服务端最新 version。
2. WHILE 处于版本冲突, THE Sync_Core SHALL NOT 以本次推送的新数据覆盖 collab_db 中该 Sync_Object 的当前数据。
3. THE Sync_Core SHALL NOT 对发生版本冲突的两份数据执行任何自动合并。
4. THE Sync_Core SHALL 将冲突后的合并职责交由端完成：端据回传的最新 version 拉取最新数据、在端上合并后以新的 base_version 重新推送。

### Requirement 5: 拉取变更（下载）

**User Story:** 作为用户，我想按 sync_object_id 从云端拉取某个同步对象的最新数据与版本号，这样我的本端能对齐到云端最新状态。

#### Acceptance Criteria

1. WHEN 收到一个携带有效 JWT 与 sync_object_id 的拉取请求, THE Sync_Core SHALL 经 Repository 从 collab_db 返回该 Sync_Object 的最新数据与当前 version。
2. WHERE 拉取请求携带端已知的 version, THE Sync_Core SHALL 据以判定端是否已持有最新数据，并在服务端 version 高于端已知 version 时返回最新数据与当前 version。
3. IF 请求的 sync_object_id 在 collab_db 中不存在, THEN THE Sync_Core SHALL 返回同步对象不存在的错误。

### Requirement 6: 列出用户的同步对象

**User Story:** 作为用户，我想列出我名下的所有同步对象及其版本号，这样我能了解自己有哪些数据可同步、各自处于哪个版本。

#### Acceptance Criteria

1. WHEN 收到一个携带有效 JWT 的列表请求, THE Sync_Core SHALL 经 Repository 从 collab_db 返回该 JWT 中 user_id 名下所有 Sync_Object 的清单，每项含 sync_object_id 与当前 version。
2. THE Sync_Core SHALL 仅在列表中返回请求者 user_id 名下的 Sync_Object，不返回他人名下的 Sync_Object。

### Requirement 7: 数据隔离（按 user_id 隔离）

**User Story:** 作为用户，我想确保只有我能推送 / 拉取 / 列出我名下的同步对象，这样他人无法访问或改动我的用户态数据。

#### Acceptance Criteria

1. WHEN 处理一个针对已存在 Sync_Object 的推送或拉取请求, THE Sync_Core SHALL 以 JWT 中的 user_id 与该 Sync_Object 的归属 user_id 比对以判定访问权。
2. IF JWT 中的 user_id 不等于目标 Sync_Object 的归属 user_id, THEN THE Sync_Core SHALL 拒绝访问并返回无访问权限的错误。
3. THE Sync_Core SHALL 将列表请求的返回范围限定为请求者 user_id 名下的 Sync_Object。
4. THE Sync_Core SHALL NOT 提供绕过数据隔离校验访问他人 Sync_Object 的路径。

### Requirement 8: 持久化与依赖倒置

**User Story:** 作为 collab 服务的维护者，我想同步核只依赖持久化抽象，这样数据库实现可替换、核可独立测试。

#### Acceptance Criteria

1. THE Sync_Core SHALL 经 Repository 接口访问持久化，不直接依赖具体数据库驱动。
2. THE Repository 的 PostgreSQL 实现 SHALL 使用 jackc/pgx 驱动访问 collab_db。
3. THE Collab_Service SHALL 将所有同步数据存入 collab 服务独占的 collab_db，不与 user 或 community 共享数据库。

### Requirement 9: 传输安全与配置收敛

**User Story:** 作为运维者，我想安全差异只在配置里切换，这样公网强制加密而内网开发不被阻塞，且代码不因环境分叉。

#### Acceptance Criteria

1. WHERE Collab_Service 对公网提供服务, THE Collab_Service SHALL 强制 TLS 传输。
2. WHERE Collab_Service 运行于 dev 或内网环境, THE Collab_Service SHALL 允许经配置关闭 TLS 强制。
3. THE Collab_Service SHALL 将 dev / 内网与公网的传输安全差异收敛在配置，不通过修改代码切换。
4. THE Collab_Service SHALL 经环境变量注入验签公钥的获取方式与数据库连接凭据，不将其写入仓库。

## MVP 范围与后置项

### 本次 MVP 做（核心链路）

- Requirement 1：MCP tool 门面（对外唯一形态）。
- Requirement 2：JWT 本地验签认定请求者身份（下游身份契约落地，不回调 user）。
- Requirement 3：推送变更（带 base_version 的乐观并发写入，匹配则 +1）。
- Requirement 4：版本冲突裁决（不匹配则拒绝、回传最新 version、不自动合并、不覆盖，合并交端）。
- Requirement 5：拉取变更（返回最新数据与 version）。
- Requirement 6：列出用户的同步对象（清单含 sync_object_id + version）。
- Requirement 7：数据隔离（按 user_id 隔离，JWT user_id 与归属不符则拒绝）。
- Requirement 8 / 9：持久化依赖倒置、传输安全与配置收敛（支撑上述链路立住的工程底线）。

### 本次不做（后置，含翻盘条件）

- **实时推送（多端即时感知他端变更）**——复用 iot-kit 的传输层、但不并入 iot-kit、自定用户数据的消息语义；翻盘条件：出现多端需要即时感知他端变更的真实诉求（当前靠端主动拉取已足够）。
- **服务端自动合并 / CRDT / OT**——翻盘条件：出现「版本冲突让端自行合并重传」无法满足的协同诉求（如多端需同时写入同一对象且不可丢变更）。
- **离线变更队列**——翻盘条件：出现端长时间离线后需有序回放多笔变更的真实诉求。
- **字段级细粒度 diff**——翻盘条件：整对象版本的推拉在数据体量或冲突粒度上不可接受，出现需按字段级 diff 同步的真实诉求（MVP 按整对象版本）。
- **端在线状态 / presence**——翻盘条件：出现需要呈现「用户在哪些端在线 / 活跃」的真实诉求。

后置项均不在本次设计与实现范围内，引入任一项须先满足其翻盘条件并重开 spec。特别地，实时推送即便落地，也只复用 iot-kit 传输层、以 collab 自定的用户数据消息语义承载，不把 collab 并入 iot-kit、不让 collab 承担设备协议 / 设备通信职责。
