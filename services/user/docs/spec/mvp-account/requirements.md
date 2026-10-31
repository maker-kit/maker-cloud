# Requirements Document

## Introduction

user 是 maker-cloud 三服务（user / community / collab）中的身份源头：对外负责注册、登录、账号基本信息，对内是另两个服务的 user_id 唯一签发者。三服务各自独立、暂不互相依赖，只共享 user_id 与 JWT 验签契约，不共享数据库。

本次 spec 的核心一句话：**立起「能注册、能登录、能拿账号信息、下游能验身份」这条最短链路，并且这条链路以 MCP tool 形态对 agent 暴露。** 这是 maker-cloud 身份体系能被其余服务依赖的最小可用地基。

两条不可退让的底线贯穿全文：

- **AI 原生**：user 对外的一切能力必须以 MCP tool 暴露、能被 agent 接入。对外不先开 REST，而是先以 streamable HTTP MCP 门面呈现，复刻既有「壳薄核重」范式——门面只做协议编排，账号逻辑沉在核。
- **无状态下游验签**：登录签发 JWT，community / collab 用 user 公布的公钥在本地验签，不回调 user 服务。user 是签发者，不是每次请求的在线校验点。

MVP 只做立住这条链路所必需的能力。第三方 OAuth、MFA、找回密码 / 邮箱验证、权限 / 角色、用户关系、历史审计一律后置，并在文末挂明确翻盘条件。

## Glossary

- **user_db**: user 服务独占的 PostgreSQL 数据库。community / collab 不直接访问 user_db。
- **User_Service**: 本 spec 描述的服务整体（含 MCP 门面 + 账号核 + Repository）。
- **MCP_Facade**: User_Service 对外的 streamable HTTP MCP 门面。对外能力只经由 MCP tool 暴露，门面只做协议编排（壳薄），不承载账号业务逻辑。
- **Account_Core**: 账号领域核（壳薄核重中的「核」）。承载注册、登录、账号信息读写、凭证签发等业务逻辑，不感知 MCP 协议与传输。
- **Repository**: 账号持久化的抽象接口。Account_Core 依赖 Repository 接口而非具体 PostgreSQL 实现（依赖倒置），持久化细节对核不可见。
- **Account**: 一个注册用户的账号实体，含 user_id、登录凭据、基本信息。
- **user_id**: 用户账号的全局唯一标识，由 User_Service 唯一签发。community / collab 只认 User_Service 签发的 user_id。
- **Credential**: 用户的登录凭据，MVP 指账号名 + 密码。密码以加盐哈希形式存于 user_db，不明文存储。
- **JWT**: User_Service 登录成功后签发的无状态会话凭证，内含 user_id 及签发/过期声明，由 User_Service 私钥签名。
- **Signing_Key**: 签发 JWT 的非对称密钥对。私钥仅 User_Service 持有并用于签名，经环境变量注入，不入库、不进仓库；公钥对外公布供下游验签。
- **Downstream_Service**: 依赖 user 身份的服务，MVP 指 community 与 collab。

## Requirements

### Requirement 1: 以 MCP tool 形态对外暴露能力

**User Story:** 作为接入 user 的 agent，我想通过 MCP tool 使用 user 的全部对外能力，这样我不依赖任何私有 REST 协议即可完成身份相关操作。

#### Acceptance Criteria

1. THE MCP_Facade SHALL 以 streamable HTTP MCP 门面形态对外提供服务。
2. THE MCP_Facade SHALL 将 User_Service 的每一项对外能力（注册、登录、账号信息读取、账号信息修改、验签契约获取）各暴露为一个 MCP tool。
3. WHEN MCP_Facade 收到一个 MCP tool 调用, THE MCP_Facade SHALL 将请求转交 Account_Core 处理并仅做协议层编排。
4. THE MCP_Facade SHALL NOT 承载注册、登录或账号信息的业务判定逻辑。
5. THE Account_Core SHALL 独立于 MCP 协议运行，使账号逻辑不感知传输形态。

### Requirement 2: 注册账号

**User Story:** 作为新用户，我想用账号名和密码注册，这样我能获得一个在 maker-cloud 全局唯一的身份。

#### Acceptance Criteria

1. WHEN 收到一个带合法账号名与密码的注册请求, THE Account_Core SHALL 创建一个 Account 并为其签发唯一的 user_id。
2. WHEN 创建 Account, THE Account_Core SHALL 将密码以加盐哈希形式经 Repository 存入 user_db。
3. THE Account_Core SHALL NOT 以明文形式存储密码。
4. IF 注册请求的账号名已存在于 user_db, THEN THE Account_Core SHALL 拒绝注册并返回账号名已存在的错误。
5. IF 注册请求缺少账号名或密码, THEN THE Account_Core SHALL 拒绝注册并返回字段缺失的错误。
6. THE Account_Core SHALL 保证所签发的 user_id 在 user_db 内全局唯一。

### Requirement 3: 登录并签发会话凭证

**User Story:** 作为已注册用户，我想用账号名和密码登录，这样我能拿到可被下游服务识别的会话凭证。

#### Acceptance Criteria

1. WHEN 收到账号名与密码与 user_db 中某 Account 匹配的登录请求, THE Account_Core SHALL 返回该 Account 的 user_id 与一枚 JWT。
2. WHEN 签发 JWT, THE Account_Core SHALL 在 JWT 中写入该 Account 的 user_id 及过期时间声明，并以 Signing_Key 私钥签名。
3. IF 登录请求的账号名不存在, THEN THE Account_Core SHALL 拒绝登录并返回凭据无效的错误。
4. IF 登录请求的密码经加盐哈希比对不匹配, THEN THE Account_Core SHALL 拒绝登录并返回凭据无效的错误。
5. THE Account_Core SHALL 使账号名不存在与密码不匹配两种情形返回同一类凭据无效错误，不区分二者。

### Requirement 4: 读取与修改账号基本信息

**User Story:** 作为已登录用户，我想读取并修改我的账号基本信息，这样我能维护自己的资料。

#### Acceptance Criteria

1. WHEN 收到携带有效 JWT 的账号信息读取请求, THE Account_Core SHALL 返回该 JWT 中 user_id 对应 Account 的基本信息。
2. WHEN 收到携带有效 JWT 的账号信息修改请求, THE Account_Core SHALL 更新该 JWT 中 user_id 对应 Account 的基本信息并经 Repository 持久化到 user_db。
3. IF 账号信息请求携带的 JWT 验签不通过或已过期, THEN THE Account_Core SHALL 拒绝请求并返回凭证无效的错误。
4. THE Account_Core SHALL NOT 在账号基本信息中返回密码或密码哈希。

### Requirement 5: 对下游公布验签契约

**User Story:** 作为 community / collab 服务，我想本地验证 user 签发的 JWT 与 user_id，这样我无需每次回调 user 服务即可确认请求者身份。

#### Acceptance Criteria

1. THE User_Service SHALL 提供获取 JWT 验签公钥的能力，供 Downstream_Service 本地验签使用。
2. THE User_Service SHALL 以明确契约公布 JWT 的签名算法与 user_id 声明字段，使 Downstream_Service 能据此本地验签。
3. THE User_Service SHALL 是 user_id 的唯一签发者。
4. WHILE Downstream_Service 验证一枚 JWT, THE 验证过程 SHALL 仅依赖 User_Service 公布的公钥与契约，不回调 User_Service。
5. THE Account_Core SHALL 经环境变量获取 Signing_Key 私钥，不从 user_db 或仓库读取私钥。

### Requirement 6: 持久化与依赖倒置

**User Story:** 作为 user 服务的维护者，我想账号核只依赖持久化抽象，这样数据库实现可替换、核可独立测试。

#### Acceptance Criteria

1. THE Account_Core SHALL 经 Repository 接口访问持久化，不直接依赖具体数据库驱动。
2. THE Repository 的 PostgreSQL 实现 SHALL 使用 jackc/pgx 驱动访问 user_db。
3. THE User_Service SHALL 将所有账号数据存入 user 服务独占的 user_db，不与 community 或 collab 共享数据库。

### Requirement 7: 传输安全与配置收敛

**User Story:** 作为运维者，我想安全差异只在配置里切换，这样公网强制加密而内网开发不被阻塞，且代码不因环境分叉。

#### Acceptance Criteria

1. WHERE User_Service 对公网提供服务, THE User_Service SHALL 强制 TLS 传输。
2. WHERE User_Service 运行于 dev 或内网环境, THE User_Service SHALL 允许经配置关闭 TLS 强制。
3. THE User_Service SHALL 将 dev / 内网与公网的传输安全差异收敛在配置，不通过修改代码切换。
4. THE User_Service SHALL 经环境变量注入 Signing_Key 与数据库连接凭据，不将其写入仓库。

## MVP 范围与后置项

### 本次 MVP 做（核心链路）

- Requirement 1：MCP tool 门面（对外唯一形态）。
- Requirement 2：注册。
- Requirement 3：登录 + 签发 user_id 与 JWT。
- Requirement 4：账号基本信息读 / 改。
- Requirement 5：下游验签契约（公钥 + 签名算法 + user_id 声明字段）。
- Requirement 6 / 7：持久化依赖倒置、传输安全与配置收敛（支撑上述链路立住的工程底线）。

### 本次不做（后置，含翻盘条件）

- **第三方 OAuth 登录**——翻盘条件：出现需要以外部身份（如 GitHub / Google）直接登录 maker-cloud 的真实接入方。
- **多因素认证 (MFA)**——翻盘条件：账号面临需要二次因素的安全要求，或接入方要求高保障等级登录。
- **找回密码 / 邮箱验证**——翻盘条件：引入真实邮箱 / 手机等可达的验证信道，且出现用户自助找回的真实诉求。
- **权限 / 角色体系**——翻盘条件：出现「同一账号需按角色区分可做什么」的真实业务，而非当前的「有效身份即可」。
- **用户关系（关注 / 好友）**——翻盘条件：community / collab 明确需要 user 承载关系数据，而非在各自服务内自管。
- **历史审计**——翻盘条件：出现合规或排障对账号操作留痕的硬要求。

后置项均不在本次设计与实现范围内，引入任一项须先满足其翻盘条件并重开 spec。
