# 1. 🚀 入门与生命周期 (Entrypoint & Lifecycle)

在深入复杂的源码之前，我们需要先理清 ZeroClaw 体系中最常出现的三个核心术语：**Onboard (引导装载)**、**Daemon (守护进程)** 和 **Gateway (网关)**。

理解了这三个概念，你就能明白这个 Agent 操作系统是如何从无到有“苏醒”并持续提供服务的。

---

## 1.1 `zeroclaw onboard` (交互式开箱引导)

> **源码位置**: `src/onboard/mod.rs` 和 `src/onboard/wizard.rs`

在你第一次安装或运行 ZeroClaw 时，如果你没有任何配置文件，系统或者官方文档最先让你运行的命令一定是 `zeroclaw onboard`。

### 它是做什么的？
`onboard` 在英文中是“登机、入职引导”的意思。在 ZeroClaw 中，它是一个**基于命令行的交互式安装向导 (Interactive Setup Wizard)**。
它**不是**一个长期运行的后台服务，而是一次性的配置构建工具。

### 功能亮点：
1. **傻瓜式配置生成**：通过在终端里一问一答（比如：“你想使用哪个 LLM 供应商？”“你的 API Key 是什么？”“你要接入飞书还是 Telegram？”），帮你自动在 `~/.zeroclaw/config.toml` 生成合格的配置。
2. **免手写排错**：直接规避了新手手写 TOML 文件容易出现的格式错误或少漏字段的问题。
3. **数据迁移**：代码中甚至包含 `run_wizard_with_migration`，它会在引导时自动检测旧版系统（如原版的 OpenClaw）并在用户同意下无缝迁移旧数据。

**当你执行完 `zeroclaw onboard`，你的电脑里就拥有了一个配置好的零号初始大脑，随时准备启动。**

---

## 1.2 Daemon (总栈守护进程)

> **源码位置**: `src/daemon/mod.rs`
> **启动命令**: `zeroclaw start` (底层调用 daemon 的 `run` 函数)

当一切配置就绪，你需要让机器“活”起来。此时，你启动的就是 **Daemon（守护进程）**。
在计算机科学中，Daemon 特指在后台无人值守运行的、监控并维持系统运转的常驻程序。

### 它到底守护着什么？
在 ZeroClaw 中，Daemon 不是一个单一的业务，而是整个 Agent 宇宙的**最高指挥官 (Supervisor)**。当你调用 `src/daemon/mod.rs` 的 `run` 函数时，它会疯狂地拉起并用线程（`tokio::spawn`）**守护**以下几个核心子系统：

1. **Gateway**：启动 HTTP / Webhook / WebSockets 监听服务（见下文）。
2. **Channels**：拉起各种长连接 IM 平台监听器（比如扫码连着 WhatsApp、Telegram Bot、Discord 的 WebSocket 连接）。
3. **Heartbeat**：开启心跳引擎，定时清理失效线程或探活。
4. **Scheduler (Cron)**：开启定时任务引擎，定期抛出提醒或者运行自检。

### 它的超能力：进程抢救 (Supervisor/Failover)
Daemon 最大的作用是**不死性**。
如果你仔细看 `src/daemon/mod.rs` 里面的 `spawn_component_supervisor` 函数，你会发现它被设计成了：如果 Gateway 或者某个 Channel 崩溃了，Daemon 主进程不会死掉，而是会记录一个 Error，稍等一秒钟（带指数退避的 Backoff），然后**原地把它重新拉起来**！

> **所以：Daemon 就是那个随电脑开机启动（或手动用 `zeroclaw start` 启动），并在后台死死盯着各个子系统，谁死了就救活谁的大管家。**

---

## 1.3 Gateway (流量收发网关)

> **源码位置**: `src/gateway/mod.rs` 和 `src/gateway/api.rs`

如果 Daemon 是人体的大脑和骨骼，那么 **Gateway (网关)** 就是人的嘴巴和耳朵，它是 Agent 接触外界的唯一互联网出入口。

### 它是做什么的？
Gateway 是一个基于 `Axum` 框架构建的高性能 **HTTP / WebSocket 服务器**（默认监听你在 `onboard` 时设置的端口资源，通常是 1455 等本地端口）。

### 它承担的三大核心任务：
1. **接管所有远端 Webhook**：当你的 Telegram、QQ、飞书收到消息时，腾讯/国外的服务器会把消息 POST 到你的机器上。Gateway 里的 `/qq`, `/telegram`, `/whatsapp` 等路由，就是专门张开嘴巴接收这些报文的地方。
2. **代理 OpenAI 兼容接口**：这是极具杀伤力的一个特性！Gateway 自己实现了 `/v1/chat/completions`。这意味着你可以把 ZeroClaw 的网关地址配置给任何第三方开源 UI (比如 Chatbox, NextChat)，这个 UI 会以为自己在和 OpenAI 对话，但实际上这个流量打到了 Gateway 里，被 Gateway 导向了背后的 **Agent 决策核心**！
3. **Web 可视化面板 (Dashboard)**：Gateway 也是一个 Web 服务器，它提供了 `/api/status`, `/api/config` 等接口，并在根路由 `/` 渲染了一个漂亮的 Web 界面，让你在浏览器里监控机器人的健康度、调用成本（Cost Tracker）和历史对话（SSE 实时流推）。

### 为什么在家里运行，外网还能访问 Gateway？
你可能注意到了代码里有这段极为关键的网关安全策略：
```rust
let tunnel = crate::tunnel::create_tunnel(&config.tunnel)?;
```
ZeroClaw 的 Gateway 天生支持 **穿透 (Tunneling)**。如果你的 Gateway 运行在自己家没公网 IP 的 Mac 电脑上，休眠会断网。但在运行期间，Gateway 可以自动挂载 `Tailscale` 或 `Ngrok` 隧道，生成一个诸如 `https://my-agent.ngrok.io` 的公网地址，让外部平台的数据能顺利打入内网。

> **💡 高阶实务：家用 Mac 秒变私有云节点与 OpenAI 兼容网关**
> 
> 一旦底层的隧道打通，这不仅意味着你能接收聊天工具的消息，更意味着**彻底脱离了 IM 工具的束缚**。你的这台闲置 Mac 原地飞升，变成了一台暴露在公网上的“云服务器”：
> 1. **直连 Web 面板**：在外面用手机浏览器打开穿透地址，就能直接在网页端和自家的 Agent 聊天、改配置、看状态。
> 2. **万物皆可套壳**：因为 Gateway 实现了 `/v1/chat/completions` (OpenAI 协议)，你可以把这个公网地址填入任何第三方开源 UI (如 Chatbox, NextChat) 或第三方 App 的 `API URL` 中。你的手机 App 会以为在连 OpenAI 官方接口，但流量和计算全跑在你家书房的 Mac 上！
> 3. **极强安全防线**：正因为如此强大，源码中才会有严密的防护——如果你在公网暴露了端口却没有配置隧道或显式开启放行，Gateway 会在启动时直接 `panic` 报错。一旦暴露到公网，调用者必须提供 `Bearer Token`（即 Pairing Code 鉴权）才能访问这层“私有云”算力。

---

## 总结：三者的协作流

1. 我什么都不会 ➡️ 运行 `zeroclaw onboard` (交互问答帮你写配置)
2. 我想让机器人上班 ➡️ 运行 `zeroclaw start` (启动 **Daemon**)
3. **Daemon** 开始工作 ➡️ 它在后台拉起了 IM 监听，同时拉起并守护着 **Gateway**。
4. 别人给你发消息 ➡️ 平台服务器将消息投递到你公网的 **Gateway** 端口 ➡️ Gateway 将消息丢给 Agent 大脑处理。
