# 4. 大模型接入层与配置池 (Providers & Config)

ZeroClaw 的大模型（LLM）接入层采用了**基于 Trait 的插件化架构**。这使得无论是云端 API 还是本地大模型，都能以统一的接口（如流式返回、Function Calling）接入系统核心引擎。

相关源码位置：`src/providers/` 和配置层 `src/config/schema.rs`。

---

## 4.1 支持的 LLM 厂商概览

ZeroClaw 原生支持众多主流模型厂商（在 `src/providers/mod.rs` 中进行路由和工厂注册）：

*   **国际主流平台**:
    *   OpenAI (`openai`, `openai_codex`)
    *   Anthropic Claude (`anthropic`)
    *   Google Gemini (`gemini`)
    *   AWS Bedrock (`bedrock`)
    *   OpenRouter (`openrouter` - ZeroClaw 默认的路由提供商)
    *   GitHub Copilot (`copilot`)
*   **本地部署模型**:
    *   Ollama (`ollama`)
*   **国内大语言模型集成** (内置了完整的别名解析机制):
    *   阿里通义千问 (`qwen`, `dashscope`)
    *   智谱 GLM (`glm`, `zhipu`)
    *   月之暗面 Kimi (`moonshot`, `kimi`)
    *   MiniMax (`minimax`)
    *   字节豆包 (`doubao`, `ark`)
*   **通用协议兼容 (`compatible` 模块)**:
    *   只要提供并兼容 OpenAI `/v1/chat/completions` API 的任何推理框架（如 vLLM, LM Studio 等），都可以通过该模块无缝接入。

---

## 4.2 LLM 的灵活配置体系

ZeroClaw 采用了声明式配置以及 12Factor 环境变量双重支持。
以下是 ZeroClaw 接入层与配置系统的架构关系统系图：

```mermaid
graph TD
    subgraph core["ZeroClaw Core"]
        ConfigHandler[("配置读取模块 - src/config/schema.rs")]
        ProviderRouter{"模型路由分发 - src/providers/mod.rs"}
        AgentEngine("Agent 决策核心")
    end

    subgraph sources["Config Sources"]
        EnvVars["环境变量 - *_API_KEY"]
        TomlConfig["active_workspace.toml / config.toml"]
    end

    subgraph providers_group["Plugin Providers"]
        OpenAI("OpenAI Provider")
        Anthropic("Anthropic Provider")
        OpenRouter("OpenRouter Provider")
        Compatible("兼容泛用 Provider - vLLM/Ollama")
        OAuthPool("OpenAI OAuth 号池")
    end

    EnvVars -->|注入| ConfigHandler
    TomlConfig -->|反序列化| ConfigHandler

    AgentEngine -->|发起统一调用| ProviderRouter
    ConfigHandler -->|提供凭据与默认选项| ProviderRouter
    
    ProviderRouter -->|走标准 API| OpenAI
    ProviderRouter -->|走标准 API| Anthropic
    ProviderRouter -->|聚合多模型| OpenRouter
    ProviderRouter -->|自定义 BaseURL| Compatible
    ProviderRouter -->|走浏览器免签凭据| OAuthPool
```

### A. 配置文件 `config.toml`（推荐方式）
通常位于工作区根目录的 `active_workspace.toml` 或主目录下的 `~/.zeroclaw/config.toml`。

**全局基础配置示例：**
```toml
default_provider = "openrouter"
model = "anthropic/claude-3-5-sonnet"
api_key = "sk-xxxxxx" 
api_url = "https://openrouter.ai/api/v1" # 用于覆盖默认的 Base URL 或内网地址
default_temperature = 0.7
```

**多模型路由与自定义 Profile：**
通过 `model_providers` 块，可以为一个复杂的 Agent 系统定义成百上千个独立的接入配置（例如路由给专门的 Coding Agent 和普通的 Chat Agent 用不同的配置）：
```toml
[model_providers.my_local_llama]
name = "compatible" 
base_url = "http://127.0.0.1:11434/v1"
```

### B. 环境变量 (Env Vars)
支持各厂商的标准环境变量（例如 `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`）以及 ZeroClaw 统一的 `ZEROCLAW_API_KEY`，极大地简化了 Docker 镜像和容器云的无状态注入。

---

## 4.3 官方 OAuth 认证基础设施 (`src/auth/`)

> [!NOTE]
> 本节内容为**官方上游正式支持的功能**，对应 `src/auth/` 目录下的三个模块。

随着 ZeroClaw 的发展，官方在 `src/auth/` 目录下建立了一套完整的 OAuth 认证基础框架，目前支持两大主流平台（OpenAI 和 Google/Gemini）的无 API Key 订阅账号接入。

### 4.3.1 统一 OAuth 工具层 (`src/auth/oauth_common.rs`)

该模块是 OpenAI 和 Gemini 两套 OAuth 实现的共用底座，提供：

- **PKCE 状态生成 (`generate_pkce_state`)**: 生成符合 RFC7636 规范的 `code_verifier`、`code_challenge`（SHA-256）和随机 `state`，防止授权码拦截攻击。
- **URL 编解码工具**: `url_encode` / `url_decode`，用于构造授权 URL 参数。
- **URL 截断检测 (`detect_url_truncation`)**: 当用户手动复制回调 URL 时，智能判断 URL 是否被截断并给出提示，引导用户改为只复制授权码。

### 4.3.2 OpenAI OAuth 双模式授权 (`src/auth/openai_oauth.rs`)

支持两种授权模式，适应不同使用环境：

**模式一：Browser 回调流（桌面端推荐）**
1. 构造带 PKCE 的 OpenAI 授权 URL（`build_authorize_url`）
2. 自动打开浏览器，监听本地 `127.0.0.1:1455` 回调端口（`receive_loopback_code`）
3. 浏览器授权成功后拦截回调 code，换取 Token（`exchange_code_for_tokens`）

**模式二：Device Code 流（服务器/无头环境推荐）**
1. 从 `https://auth.openai.com/oauth/device/code` 获取 `user_code` 和 `device_code` (`start_device_code_flow`)
2. 在终端展示短码（如 `ABCD-1234`），引导用户在任意设备浏览器完成授权
3. 程序自动轮询 Token 端点，处理 `authorization_pending` / `slow_down` 状态（`poll_device_code_tokens`）

两种模式均支持 **Token 自动刷新** (`refresh_access_token`)，所有 Token 统一通过 `TokenSet` 结构体管理。

### 4.3.3 Gemini OAuth 双模式授权 (`src/auth/gemini_oauth.rs`)

与 OpenAI 流程对称，面向 Google Cloud Platform（`cloud-platform` scope）：

- **凭据配置**：通过 `GEMINI_OAUTH_CLIENT_ID` 和 `GEMINI_OAUTH_CLIENT_SECRET` 环境变量注入（与 Gemini CLI 共用同一套 Client 凭据）
- **回调端口**：`127.0.0.1:1456`（与 OpenAI 的 1455 错开，可同时运行互不干扰）
- **兜底双模式输入**：`receive_loopback_code` 函数内置了 `tokio::select!` 逻辑，同时监听**浏览器回调** 和 **stdin 手动输入**，若回调超时则自动降级为终端粘贴模式
- **Cloudflare 防护检测**：Device Code 流内置了对 403 + Cloudflare 响应的特殊检测，服务器环境触发时主动给出切换到 Browser 流的建议

```mermaid
graph TD
    Start([zeroclaw auth login]) --> Choice{环境类型?}
    
    Choice -->|有浏览器 - 桌面端| BrowserFlow["Browser 回调流\n监听 localhost:1455/1456"]
    Choice -->|无浏览器 - 服务器/SSH| DeviceFlow["Device Code 流\n终端展示短码等待用户授权"]
    
    BrowserFlow --> PKCE["生成 PKCE State\noauth_common.rs"]
    DeviceFlow --> Poll["循环轮询 Token 端点\nauthorization_pending → 继续等待\nslow_down → 延迟增加"]
    
    PKCE --> Callback["浏览器授权后\n拦截回调 code"]
    Callback --> Exchange["换取 TokenSet\naccess_token + refresh_token"]
    Poll --> Exchange

    Exchange --> Store["保存至 profiles\n统一 Token 管理"]
```

---

## 4.4 深挖扩展：OpenAI OAuth 多号池与高可用 Failover（⚠️ Fork 自研扩展）

> [!IMPORTANT]
> **本节内容来自本项目的 Fork 自研扩展，并非 ZeroClaw 官方上游功能。**
>
> 官方上游 (`src/auth/openai_oauth.rs`) 仅实现了标准的 **PKCE + Browser/Device Code 单账户授权流程**，用于获取并存储 Token 供 CLI 本地使用，没有多账号号池、没有 Chacha20 自定义加密存储和 Failover 自动切换功能。
>
> 本节描述的以下能力均为 Fork 中新增/修改的实现：
> - `src/providers/openai_oauth.rs`（完整的 OAuth Provider 接入）
> - `src/oauth/store.rs`（多号 Chacha20 加密存储池）
> - `advance_cursor()` Failover 切换逻辑
>
> **官方上游支持计划**：从现有 GitHub Issue/PR 来看，官方目前尚无公开的多账号号池路线图。Gemini OAuth 已于近期被合并到 `src/auth/` 中（含 `gemini_oauth.rs` 与统一 OAuth 工具类 `oauth_common.rs`），说明官方在持续完善 OAuth 基础框架，但 "Provider 层的多账号 Failover" 属于更高阶的应用层能力，短期内是否会官方化尚无公开讨论。

在 Fork 版的 `src/providers/` 目录下，扩展实现了一个生产级、企业级的高可用接入层：**`openai_oauth.rs`**。
这一扩展解决了：在没有购买纯 API Key 时，如何利用官方 ChatGPT Plus/Pro/Free 账号的 OAuth 鉴权，实现稳定、高可用的 API 调用，并自带多号防风控切换功能。

### 4.4.1 OAuth 授权与多号录入 (`src/oauth_cli.rs` & `src/oauth/openai_login.rs`)

执行命令 `zeroclaw oauth login --provider openai` 的底层微观流程：
1.  **启动本地回调服务**：建立短暂的 HTTP Server (默认监听 `127.0.0.1:1455`)。
2.  **拉起浏览器**：在桌面端自动调起浏览器访问 OpenAI 授权端点。
3.  **拦截 Token**：用户同意授权后，浏览器重定向回本地，换取完整 OAuth 载荷（`access_token`, `refresh_token` 等）。
4.  **安全入库**：通过 `store.add_openai_account(...)` 将提取的账号存入配置池文件中。

### 4.4.2 凭据的多号池存储引擎 (`src/oauth/store.rs`)

*   **存储位置**: 位于 `~/.zeroclaw/oauth/openai_accounts.toml`。
*   **追加模式 (Append)**：每次调用 `login` 都会在 TOML 的 Array 中追加一个账号对象。构成了多号的弹药库。
*   **AES256 加密静息态 (SecretStore)**：最亮眼的设计。ZeroClaw **绝对不会**将 OAuth Token 明文存放在磁盘上。由于这属于超级敏感凭证，系统会在将其序列化落盘前，使用核心包封装的基于 Chacha20Poly1305 算法的 `SecretStore` 加密，确保哪怕配置被拷贝也无法劫持 Token。

### 4.4.3 Agent 运行时的全自动轮转与智能 Failover (`src/providers/openai_oauth.rs`)

当配置文件指定 `default_provider = "openai-oauth"` 或命令行传入指定接入方时，多号池的强大能力将在这一刻展现。
以下是这个自带中间件能力的 Provider 如何在发包时进行错误拦截与重试的流程图：

```mermaid
sequenceDiagram
    participant Agent as ZeroClaw Agent
    participant Provider as OpenAiOAuthProvider
    participant Pool as 本地加密号池 - OAuthStore
    participant OpenAI as OpenAI API 端点

    Agent->>Provider: 发起对话请求 Chat Completion
    activate Provider
    
    Provider->>Pool: 提取当前 Index 的 Account
    
    alt Token 即将过期 - Expires in < 30s
        Provider->>OpenAI: 使用 Refresh Token 静默换发
        OpenAI-->>Provider: 返回新 Access Token
        Provider->>Pool: 重新 Chacha20 加密落盘
    end

    loop Failover 重试循环 - Cursor Advance
        Provider->>OpenAI: 携带 Access Token 发送请求 Body
        
        alt 正常返回 HTTP 200
            OpenAI-->>Provider: 返回流式/静态回答
            Provider-->>Agent: 处理完毕！
            
        else 触发 401 Unauthorized
            OpenAI--xProvider: HTTP 401
            Note over Provider: 拦截错误！强制刷新 Token 并原地再重试一次
            
        else 触发限流或欠费 HTTP 429 / 50X / Quota Exceeded
            OpenAI--xProvider: HTTP 429 Too Many Requests
            Note over Provider: 触发 advance_cursor - 标记当前账号风控休眠，指针移向号池下一个有效账号
            Provider->>Pool: 获取下一个接力 Account
            Note over Provider: 回到循环头部，用下一个账号重发 Request
        end
    end
    
    deactivate Provider
```

1.  **解密与装载 (Pooling)**：从 `store.load_openai_runtime_accounts()` 加载并解密所有状态为启用的 OAuth 号，构建运行时的 `Vec<oauth::OpenAiOAuthRuntimeAccount>` 池。
2.  **静默无感续期 (Auto-refresh Token)**：发包前，如果检测到当前使用的 `access_token` 将在 30 秒内过期，底层会静默地向 OpenAI 申请换发并发起落盘更新。对业务层代码 100% 透明。
3.  **401 抢救机制**：如果是突发的 HTTP 401，拦截错误、强制刷新 Token、然后拿着合法的 Request 帮用户**重发一遍**。
4.  **智能限流容灾 (Failover / cursor advance)**：
    *   如果遭遇 HTTP 429 Too Many Requests, 502 Bad Gateway，或者 Body 包含 `insufficient_quota` (配额耗尽)。
    *   在 `openai_oauth.rs` 中，有一个经典的函数 `advance_cursor`。程序会自动将当前请求的轮询索引 `index + 1`。
    *   **无缝将出错的请求投递给池子里的下一个 OpenAI 账号**，直到成功或者全部耗尽才会向外抛出 Err。

> **总结**: ZeroClaw 的接入层并非仅仅完成了“发一个 HTTP Post 请求”这么简单。它对于特定厂商的封装达到了企业级中间件的标准，原生的限流退避、自动容灾、安全加密存储使得它可以胜任长时间运行的强健生产级 Agent。

### 4.4.4 号池的底层数据结构与加密模型

在 `oauth/store.rs` 中，定义了这个高度安全的“号池”的数据结构。文件落盘时，它的标准面貌是一个包含了**顶层版本号**和**账号数组 (Array of Tables)** 的 TOML 文件。

#### A. TOML 文件的结构

```rust
struct OAuthStoreFile {
    version: u32,                                // 版本号，当前硬编码为 1
    openai_accounts: Vec<StoredOpenAiOAuthAccount>,  // 一个包含多账号的数组 (号池)
}
```

#### B. 数据安全层级切分

一个账号 (`StoredOpenAiOAuthAccount`) 在序列化时，严格区分了**明文**和**密文**：

```rust
struct StoredOpenAiOAuthAccount {
    // ---- 账号标识 (明文) ----
    id: String,              // UUID，例如: 550e8400...
    name: String,            // 用户易读别名，通常是从 JWT 提取出的 Email
    enabled: bool,           // 开关，如果设为 false，Agent 轮询时会跳过
    authorize_url: String,   // OAuth 授权跳转 URL
    token_url: String,       // 换取/刷新 Token 的 API URL
    client_id: String,       // 执行 OAuth 的客户端 ID
    scope: String,           // OAuth 权限范围

    // ---- 核心敏感凭据 (🌟 全部经过 AES-GCM/Chacha20 密文加密) ----
    client_secret: Option<String>,  
    id_token: Option<String>,       
    access_token: String,           // (密文) 最核心的调用凭证！
    refresh_token: Option<String>,  // (密文) 用于后台无感续期的令牌！
    openai_api_key: Option<String>, 

    // ---- 时间戳与元数据 (明文) ----
    expires_at: Option<String>, // access_token 过期时间 (RFC3339)
    created_at: String,         
    updated_at: String,         
}
```

#### C. 落盘后的真实文件样貌

正因为底层极其关注安全 (Secure by design)，如果你用文本编辑器打开 `~/.zeroclaw/oauth/openai_accounts.toml`，你看到的**绝对不是**裸露的 Token：

```toml
version = 1

[[openai_accounts]]
id = "9bd11026-64c8-47e0-b6e2-57fa2cd2f7c0"
name = "my-pro-account@gmail.com"
authorize_url = "https://auth.openai.com/oauth/authorize"
token_url = "https://auth.openai.com/oauth/token"
client_id = "app_EMoamEEZ73f0CkXaXp7hrann"
scope = "openid profile email offline_access"

# 注意：核心 Token 被 chacha20poly1305 加密成了乱码！
# 即便文件泄露，没有本机的衍生硬件环境密钥也无法解密！
access_token = "M4bXjQ3T1jVp0G2g...[极长的乱码密文]..."
refresh_token = "Q8zL3P4xR2mA9bC...[极长的乱码密文]..."

expires_at = "2026-03-01T22:30:15Z"
created_at = "2026-02-28T22:25:00Z"
updated_at = "2026-02-28T22:25:00Z"
enabled = true

# 多次执行 login，下方就会继续出现多组 [[openai_accounts]]
```

---

### 4.4.5 延伸对照：官方 OpenAI Codex CLI 的解法 (Keyring) 🔑

为了更深入理解系统级编程中的凭据存储艺术，我们特意查阅了同期的 OpenAI 官方开源项目 —— **Codex CLI** (`codex-rs`) 是如何处理这段逻辑的（见 Codex 源码 `rmcp-client/src/oauth.rs`）。

Codex 采取了另一种业界标准的做法：**基于操作系统的 Keyring（凭据管理器）**。

1.  **存储策略三连 (`OAuthCredentialsStoreMode`)**：
    Codex 定义了三种模式：`Auto` (默认), `File`, `Keyring`。
2.  **默认走硬件 / OS 级密保 (`KeyringStore`)**：
    在 `Auto` 模式下，Codex 会默认尝试调用底层操作系统的 Keychain 机制（通过 Rust 的 `keyring` crate）：
    *   **macOS**: macOS Keychain (钥匙串)
    *   **Windows**: Windows Credential Manager (凭据管理器)
    *   **Linux**: DBus-based Secret Service 或 Linux kernel keyutils
3.  **降级策略 (Fallback to File)**：
    只有当操作系统的 Keyring 服务不可用、或者因为环境问题（如 Docker 容器、无头服务器）报错时，Codex 才会降级将 Token 写到文件 `~/.codex/.credentials.json` 中。
    值得注意的是，Codex 的文件降级存储方案是**明文 (Plain text)**的，因此它通过代码在 Unix 系统下强制把文件权限设定为 `0o600`（仅当前用户可读写）来做基础防范：
    ```rust
    // Codex 源码片段
    let perms = fs::Permissions::from_mode(0o600);
    fs::set_permissions(&path, perms)?;
    ```

**对比总结：**
*   **Codex CLI**: 优先依赖操作系统的 Keychain，极其标准化。但在无状态服务器容器内部署时，通常会降级到**明文存储的 json 文件**，有文件被拖库窃取的风险。
*   **ZeroClaw**: 取消了对臃肿的 OS Keychain API 的依赖，采用**自带的 AES (Chacha20Poly1305) 跨平台加密**。无论是在 Mac 桌面还是 Docker Linux 容器里，落盘的永远是用机器环境熵派生出 Key 的乱码密文。这种 `SecretStore` 机制让它的全平台安全性表现得更一致，体积也更轻巧。

---

### 4.4.6 高阶技巧：拥抱开源生态 —— 将官方 Codex `auth.json` 迁移至 ZeroClaw 🚀

> [!WARNING]
> **本节迁移方案基于 Fork 版的 `OAuthStore`（TOML 格式）。官方上游已更新为 `auth-profiles.json`（JSON + `enc2:` 加密），写入目标和方式已不同。Token 字段映射仍然有效，但写入层需适配新格式。**

Codex 会将凭据以 `auth.json`（明文格式）落盘至 `~/.codex/auth.json`。核心 Token 字段两端完全兼容，只是写入目标格式已变化。

我们看一下 Codex 源码中定义的文件结构：

```rust
pub struct AuthDotJson {
    pub auth_mode: Option<String>,
    pub openai_api_key: Option<String>,
    pub tokens: Option<TokenData>,
}

pub struct TokenData {
    pub id_token: IdTokenInfo,   // 含 email 和 raw_jwt
    pub access_token: String,
    pub refresh_token: String,
    pub account_id: Option<String>,
}
```

#### 字段映射（两端完全兼容）

| Codex `TokenData` | ZeroClaw `TokenSet` / `AuthProfile` |
|---|---|
| `access_token` | `token_set.access_token` |
| `refresh_token` | `token_set.refresh_token` |
| `id_token.raw_jwt` | `token_set.id_token` |
| `account_id` | `profile.account_id` |
| `id_token.email` | `profile.profile_name` |

#### 新旧写入目标对比

| | Fork 版（旧） | 官方上游（当前） |
|---|---|---|
| **目标文件** | `openai_accounts.toml` | `auth-profiles.json` |
| **写入方式** | Append TOML Array | `AuthProfilesStore::upsert_profile` |
| **加密格式** | Fork 自定义 Chacha20 | `enc2:` 前缀（SecretStore） |
| **多账号** | ✅ 多条记录 | ❌ 每个 provider 一个 active profile |

#### 迁移实现思路

**面向官方上游**（`auth-profiles.json`）：

1. 探测并读取 `~/.codex/auth.json`
2. 提取 `tokens.access_token`、`refresh_token`、`id_token`、`account_id`
3. 构造 `AuthProfile::new_oauth("openai-codex", email, TokenSet { ... })`
4. 调用 `AuthProfilesStore::upsert_profile(profile, true)` — 内部自动以 `enc2:` 格式加密落盘

**面向 Fork 版**（`openai_accounts.toml`）：

1. 提取 Token 后用 `SecretStore` 加密
2. 常量字段（`client_id`、`authorize_url`、`scope`）硬编码写入
3. 动态字段（`id`、`expires_at`）用 `Uuid::new_v4()` / `Utc::now()` 重建
4. Append 到 `openai_accounts.toml`

为什么可以对齐？因为两端 Token 均来自 **OpenAI Auth 服务器**，是标准 RFC 6749 OAuth Token。有了 `refresh_token`，系统在后续请求时可自动刷新 `access_token`，无需关心初始 `expires_at` 精度。

最终效果：用一行命令 `zeroclaw auth import --from-codex`（需实现），跳过浏览器重新授权，直接复用 Codex 明文凭据驱动 ZeroClaw Agent。
