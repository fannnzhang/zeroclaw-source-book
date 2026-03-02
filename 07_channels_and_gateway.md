# 7. 多端通道互联与网关通信 (Channels & Gateway)

Agent 只有大脑（核心执行引擎）和记忆是不够的，它需要感知外部世界并作出交互。
在 ZeroClaw 中，负责与人类或其他系统交互的嘴巴和耳朵被抽象为 `channels` 模块，而负责统一管理这些管道生命周期的则是基于 Tokio 构建的高级异步 `gateway` 模块。

---

## 7.1 通用通信管道抽象: `Channel` Trait (`src/channels/traits.rs`)

由于市面上有数百种 IM（即时通讯）和社群软件，它们的 API 千奇百怪。ZeroClaw 采用了适配器模式，用这一个 Trait 统御了所有的外部输入源（Telegram, Discord, Slack, 飞书/Lark, 微信等）：

```rust
#[async_trait]
pub trait Channel: Send + Sync {
    // ---- 核心收发机制 ----
    async fn send(&self, message: &SendMessage) -> anyhow::Result<()>;
    async fn listen(&self, tx: tokio::sync::mpsc::Sender<ChannelMessage>) -> anyhow::Result<()>;

    // ---- 用户体验 (UX) 增强机制 ----
    async fn start_typing(&self, recipient: &str) -> anyhow::Result<()>;
    async fn stop_typing(&self, recipient: &str) -> anyhow::Result<()>;

    // ---- 流式响应机制 (Draft Updates) ----
    fn supports_draft_updates(&self) -> bool; // 该平台是否支持修改已发送的消息？
    async fn send_draft(&self, message: &SendMessage) -> anyhow::Result<Option<String>>;
    async fn update_draft(&self, recipient: &str, message_id: &str, text: &str) -> anyhow::Result<Option<String>>;
    
    // ---- 审批交互机制 ----
    async fn send_approval_prompt(...) -> anyhow::Result<()>;
}
```

### 重点架构巧思：
1. **统一流式截取 (`supports_draft_updates`)**:
   大模型的回答是一个字一个字蹦出来的（Server-Sent Events）。对于支持此特性的平台（如 Telegram），Agent 会持续调用 `update_draft` 用新获取的 Token 去**修改 (Edit)** 刚刚发出的临时消息，实现打字机效果。对于不支持的平台（如邮件），此机制会平滑降级为等待全量生成再发送。
2. **免端侧审批 (`send_approval_prompt`)**:
   这是 ZeroClaw 的一大亮点。当 Agent 准备执行高危动作（如 `rm -rf`或转账）时，它会通过当前通讯的 Channel 下发卡片或特定命令（如 `/approve-allow [id]`），管理员只要在手机上点击一下，指令就会飞回核心引擎并放行阻塞的线程。

---

## 7.2 异步神经中枢：网关层设计 (`src/gateway/mod.rs`)

外部平台的 Webhook 事件是如何流入 Agent 的？系统提供了一个极其强健的、基于 `Axum` 框架编写的 HTTP/WebSocket 网关微服务。

它具备了非常硬核的生产级特性，而非仅仅是一个简单的 API 挂载器。

```mermaid
graph TD
    Client[Telegram / Webhook / Web App] -->|HTTP POST / WSS| Gateway((ZeroClaw Gateway))
    
    subgraph Gateway Core (src/gateway/)
        RateLimiter{"限流防火墙<br/>(SlidingWindowRateLimiter)"}
        IdempotencyStore{"幂等去重器<br/>(IdempotencyStore)"}
        PayloadValidation[恶意载荷拦截<br/>Content-Length 阻断]
        
        Gateway --> PayloadValidation
        PayloadValidation --> RateLimiter
        RateLimiter --> IdempotencyStore
        IdempotencyStore --> Routes[Axum 路由分发器]
    end
    
    subgraph Event Bus (MPSC Channels)
        Routes -->|tokio::sync::mpsc::Sender| Core[Agent 核心执行引擎]
    end
```

### 7.2.1 工业级的微服务网关防御面
ZeroClaw 能够以安全裸漏在公网，归功于 `mod.rs` 内置的这些防御：

1. **内存耗尽防御 (OOM Block)**:
   使用 `tower_http::limit::RequestBodyLimitLayer`，直接在四层拦截大于 64KB 的异常请求，彻底杜绝恶意构造巨型 JSON 把运行内存在单实例上打满的漏洞。
2. **滑动窗口限流器 (Sliding Window Rate Limiter)**:
   自带了一个非常精巧的内存级流控算法，清理僵尸 IP，如果某人在极短时间内狂点发送或恶意构造攻击包，会被直接返回 429 Too Many Requests。
3. **极强幂等墙 (IdempotencyStore)**:
   这对于即时通讯机器人开发**至关重要**。很多平台的 Webhook（特别是企业微信和 WhatsApp）如果在 3 秒内未收到 200 OK，会疯狂开始重试。由于 Agent 大脑思考可能要十几秒，网关的 `IdempotencyStore` 会记录消息指纹，保证再多的重发砸过来也会被吃掉，保证 LLM 的 Token 配额**绝对不会被平行触发的幽灵请求双倍扣除**。

### 7.2.2 WebSocket 与 Server-Sent Events (SSE)
除了被动的 Webhook，网关还暴露了 `ws.rs` 和 `sse.rs`。这主要为了支撑 ZeroClaw 未来的前端 Web UI 管理面板（或提供长连接以便网页直接与 Agent 大脑互动）。它们共享了通道投递事件槽，让架构极其优雅统一。
