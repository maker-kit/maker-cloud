# Requirements Document

## Introduction

community 是 maker-cloud 三服务（user / community / collab）中的作品社区：对外负责案例库 / 作品分享。三服务各自独立、暂不互相依赖，只共享 user 签发的 user_id 与 JWT 验签契约，不共享数据库。community 的领域核是 Work（作品）。

本次 spec 的核心一句话：**立起「能发布作品、能浏览检索、能看详情、能删自己的作品」这条最短链路，并且这条链路以 MCP tool 形态对 agent 暴露。** 这是 community 作为作品社区能被使用与被 agent 接入的最小可用地基。

两条不可退让的底线贯穿全文：

- **AI 原生**：community 对外的一切能力必须以 MCP tool 暴露、能被 agent 接入。对外不先开 REST，而是先以 streamable HTTP MCP 门面呈现，复刻「壳薄核重」范式——门面只做协议编排，社区逻辑沉在核。
- **身份认 JWT、本地验签不回调**：community 不自管账号。发布者 / 操作者身份一律取自 user 服务签发的 JWT——community 用 user 公布的验签公钥在本地验签，从 JWT 取 user_id 认定身份，不回调 user 服务。user_id 来源唯一是 user 服务，community 只认不签发。

MVP 只做立住这条链路所必需的能力。点赞 / 收藏 / 评论、关注关系、全文搜索 / 标签推荐、二创溯源、审核 / 举报、排行榜、对象存储一律后置，并在文末挂明确翻盘条件。作品产物（如 .cavo 工程文件、缩略图）本次只存引用 / URL + 元信息，community 不做二进制存储。

## Glossary

- **community_db**: community 服务独占的 PostgreSQL 数据库。user / collab 不直接访问 community_db，community 也不访问其数据库。
- **Community_Service**: 本 spec 描述的服务整体（含 MCP 门面 + 作品核 + Repository）。
- **MCP_Facade**: Community_Service 对外的 streamable HTTP MCP 门面。对外能力只经由 MCP tool 暴露，门面只做协议编排（壳薄），不承载社区业务逻辑。
- **Work_Core**: 作品领域核（壳薄核重中的「核」）。承载作品发布、检索、详情、删除及归属校验等业务逻辑，不感知 MCP 协议与传输。
- **Repository**: 作品持久化的抽象接口。Work_Core 依赖 Repository 接口而非具体 PostgreSQL 实现（依赖倒置），持久化细节对核不可见。
- **Work**: 一件已发布作品的领域实体，含 work_id、发布者 user_id、作品元信息与产物引用。是 community 的领域核。
- **work_id**: 一件作品的全局唯一标识，由 Community_Service 在发布时签发，用于检索、查看详情与删除定位。
- **发布者 user_id**: 作品发布者的身份标识，取自发布请求所携带 JWT 中的 user_id，由 user 服务唯一签发，community 只认不签发。一件 Work 的发布者 user_id 在发布时写定，不可变更。
- **作品元信息 (Work_Metadata)**: 描述作品的非产物数据，MVP 指标题、描述等文本字段。
- **产物引用 (Artifact_Reference)**: 指向作品产物（如 .cavo 工程文件、缩略图）的 URL 与必要元信息。community 只存引用，不托管二进制本体。
- **JWT**: user 服务签发的无状态会话凭证，内含 user_id 及签发 / 过期声明，由 user 服务私钥签名。community 收到的请求凭此认定操作者身份。
- **JWT 本地验签 (Local_JWT_Verification)**: community 用 user 公布的验签公钥与契约在本地校验 JWT 的签名与有效期、并提取 user_id 的过程，全程不回调 user 服务。
- **验签公钥 (Verification_Public_Key)**: user 服务公布、供 community 本地验签 JWT 的公钥。经环境变量注入 community，不入库、不进仓库。
- **Downstream 身份契约**: user 公布的 JWT 签名算法与 user_id 声明字段约定，community 据此本地验签并提取发布者 / 操作者 user_id。

## Requirements

### Requirement 1: 以 MCP tool 形态对外暴露能力

**User Story:** 作为接入 community 的 agent，我想通过 MCP tool 使用 community 的全部对外能力，这样我不依赖任何私有 REST 协议即可完成作品相关操作。

#### Acceptance Criteria

1. THE MCP_Facade SHALL 以 streamable HTTP MCP 门面形态对外提供服务。
2. THE MCP_Facade SHALL 将 Community_Service 的每一项对外能力（发布作品、检索作品列表、查看作品详情、删除作品）各暴露为一个 MCP tool。
3. WHEN MCP_Facade 收到一个 MCP tool 调用, THE MCP_Facade SHALL 将请求转交 Work_Core 处理并仅做协议层编排。
4. THE MCP_Facade SHALL NOT 承载发布、检索、详情或删除的业务判定逻辑。
5. THE Work_Core SHALL 独立于 MCP 协议运行，使社区逻辑不感知传输形态。

### Requirement 2: 本地验签 JWT 认定操作者身份

**User Story:** 作为 community 服务，我想用 user 公布的公钥在本地验证请求携带的 JWT 并取出 user_id，这样我无需回调 user 服务即可认定操作者身份。

#### Acceptance Criteria

1. WHEN 收到一个携带 JWT 的请求, THE Work_Core SHALL 以验签公钥按 Downstream 身份契约在本地验证 JWT 的签名与有效期，并从 JWT 中提取 user_id 作为操作者身份。
2. WHILE Work_Core 验证一枚 JWT, THE 验证过程 SHALL 仅依赖 user 公布的验签公钥与身份契约，不回调 user 服务。
3. IF 请求携带的 JWT 验签不通过或已过期, THEN THE Work_Core SHALL 拒绝请求并返回凭证无效的错误。
4. THE Community_Service SHALL 经环境变量获取验签公钥，不从 community_db 或仓库读取验签公钥。
5. THE Community_Service SHALL NOT 自行签发 user_id，发布者 / 操作者身份一律取自 user 服务签发的 JWT。

### Requirement 3: 发布作品

**User Story:** 作为已登录的创作者，我想携带作品元信息与产物引用发布一件作品，这样我的作品能进入社区被他人浏览。

#### Acceptance Criteria

1. WHEN 收到一个携带有效 JWT、作品元信息与产物引用的发布请求, THE Work_Core SHALL 创建一件 Work、为其签发唯一 work_id，并将 JWT 中的 user_id 记为该 Work 的发布者 user_id。
2. WHEN 创建 Work, THE Work_Core SHALL 经 Repository 将作品元信息、产物引用与发布者 user_id 存入 community_db。
3. THE Work_Core SHALL 仅存储产物引用（URL 与必要元信息），不存储产物的二进制本体。
4. IF 发布请求缺少标题, THEN THE Work_Core SHALL 拒绝发布并返回字段缺失的错误。
5. IF 发布请求缺少产物引用, THEN THE Work_Core SHALL 拒绝发布并返回字段缺失的错误。
6. THE Work_Core SHALL 保证所签发的 work_id 在 community_db 内全局唯一。
7. THE Work_Core SHALL 在 Work 发布后保持其发布者 user_id 不可变更。

### Requirement 4: 浏览与检索作品列表

**User Story:** 作为社区访客，我想按简单条件分页浏览作品列表，这样我能在社区中找到感兴趣的作品。

#### Acceptance Criteria

1. WHEN 收到一个作品列表检索请求, THE Work_Core SHALL 经 Repository 从 community_db 返回符合条件的 Work 列表及其摘要信息。
2. WHERE 检索请求携带发布者 user_id 过滤条件, THE Work_Core SHALL 仅返回该发布者 user_id 发布的 Work。
3. WHEN 检索请求携带分页参数, THE Work_Core SHALL 按分页参数返回对应页的 Work 列表。
4. IF 检索请求的分页参数超出合法范围, THEN THE Work_Core SHALL 拒绝请求并返回分页参数非法的错误。
5. THE Work_Core SHALL NOT 在列表检索中提供全文搜索能力。

### Requirement 5: 查看作品详情

**User Story:** 作为社区访客，我想按 work_id 查看一件作品的完整详情，这样我能了解作品的元信息与产物引用。

#### Acceptance Criteria

1. WHEN 收到一个携带 work_id 的作品详情请求, THE Work_Core SHALL 经 Repository 从 community_db 返回该 Work 的作品元信息、产物引用与发布者 user_id。
2. IF 请求的 work_id 在 community_db 中不存在, THEN THE Work_Core SHALL 返回作品不存在的错误。

### Requirement 6: 删除自己的作品并校验归属

**User Story:** 作为作品发布者，我想删除我发布的作品，并确保他人无法删除我的作品，这样作品的删除权只归属于其发布者。

#### Acceptance Criteria

1. WHEN 收到一个携带有效 JWT 与 work_id 的删除请求, THE Work_Core SHALL 以 JWT 中的 user_id 与该 Work 的发布者 user_id 比对以判定归属。
2. WHILE JWT 中的 user_id 等于该 Work 的发布者 user_id, THE Work_Core SHALL 经 Repository 从 community_db 删除该 Work。
3. IF JWT 中的 user_id 不等于该 Work 的发布者 user_id, THEN THE Work_Core SHALL 拒绝删除并返回无删除权限的错误。
4. IF 删除请求的 work_id 在 community_db 中不存在, THEN THE Work_Core SHALL 返回作品不存在的错误。
5. THE Work_Core SHALL 仅在归属校验通过后执行删除，不提供绕过归属校验的删除路径。

### Requirement 7: 持久化与依赖倒置

**User Story:** 作为 community 服务的维护者，我想作品核只依赖持久化抽象，这样数据库实现可替换、核可独立测试。

#### Acceptance Criteria

1. THE Work_Core SHALL 经 Repository 接口访问持久化，不直接依赖具体数据库驱动。
2. THE Repository 的 PostgreSQL 实现 SHALL 使用 jackc/pgx 驱动访问 community_db。
3. THE Community_Service SHALL 将所有作品数据存入 community 服务独占的 community_db，不与 user 或 collab 共享数据库。

### Requirement 8: 传输安全与配置收敛

**User Story:** 作为运维者，我想安全差异只在配置里切换，这样公网强制加密而内网开发不被阻塞，且代码不因环境分叉。

#### Acceptance Criteria

1. WHERE Community_Service 对公网提供服务, THE Community_Service SHALL 强制 TLS 传输。
2. WHERE Community_Service 运行于 dev 或内网环境, THE Community_Service SHALL 允许经配置关闭 TLS 强制。
3. THE Community_Service SHALL 将 dev / 内网与公网的传输安全差异收敛在配置，不通过修改代码切换。
4. THE Community_Service SHALL 经环境变量注入验签公钥的获取方式与数据库连接凭据，不将其写入仓库。

## MVP 范围与后置项

### 本次 MVP 做（核心链路）

- Requirement 1：MCP tool 门面（对外唯一形态）。
- Requirement 2：JWT 本地验签认定操作者身份（下游身份契约落地，不回调 user）。
- Requirement 3：发布作品（元信息 + 产物引用，记录发布者 user_id）。
- Requirement 4：浏览 / 检索作品列表（基础列表 + 简单过滤 / 分页，不做全文搜索）。
- Requirement 5：查看单个作品详情。
- Requirement 6：删除自己的作品（归属校验——只有 JWT 中 user_id 等于发布者 user_id 才能删）。
- Requirement 7 / 8：持久化依赖倒置、传输安全与配置收敛（支撑上述链路立住的工程底线）。

### 本次不做（后置，含翻盘条件）

- **点赞 / 收藏 / 评论等互动**——翻盘条件：出现社区需要承载用户对作品的互动表达的真实诉求。
- **关注关系**——翻盘条件：出现「按创作者订阅 / 关注其作品动态」的真实业务，且确认关系数据归属 community。
- **全文搜索 / 标签推荐**——翻盘条件：基础列表过滤无法满足查找需求，出现需按内容全文或标签智能检索的真实诉求。
- **二创 / fork 溯源**——翻盘条件：出现需要记录作品间派生关系并溯源的真实业务。
- **审核 / 举报**——翻盘条件：出现内容合规或社区治理对作品准入 / 下架留痕的硬要求。
- **排行榜**——翻盘条件：出现需按热度 / 质量对作品排序呈现的真实运营诉求。
- **对象存储（二进制产物托管）**——翻盘条件：出现 community 需直接托管二进制产物（而非仅存引用 / URL）的真实诉求。

后置项均不在本次设计与实现范围内，引入任一项须先满足其翻盘条件并重开 spec。
