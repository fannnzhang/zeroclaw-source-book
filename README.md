# ZeroClaw 源码学习指南 (Source Code Study Guide)

欢迎来到 ZeroClaw 源码学习指南！这是一个由读者梳理的非官方、但极具深度的源码级剖析文档集。

由于 ZeroClaw 采用 100% Rust 原生实现，且主打极低开销（<5MB 内存）和极高模块化（Trait 插件驱动），它的源码对于学习现代 Rust 系统编程、Agent 架构设计具有极高的参考价值。

## 📖 目录 (Table of Contents)

本指南按照系统架构的层级，自底向上/自顶向下将庞大的工程拆分为以下几个核心模块，你可以点击链接进入对应的深入解析：

1. **[🚀 入门与生命周期 (Entrypoint & Lifecycle)](01_entrypoint_and_lifecycle.md)**
   > 探究 `src/onboard/mod.rs`、`src/daemon/mod.rs` 和 `src/gateway/mod.rs`。从零配置的引导程序 (Onboard)、不死常驻服务 (Daemon)，到内收外发的流量收发中枢 (Gateway)。

2. **[⚙️ 配置中心与架构 (Configuration & State)](02_configuration.md)**
   > 探究 `src/config/schema.rs`。一览多达 4 大板块的巨型配置树，解析全局默认模型、Agent 记忆挂载以及强大的底层热插拔 (Hot-Reloading) 机制。

   * **[📖 附录：用户级配置实战指南](02_5_configuration_guide.md)**
     > 一份针对普通用户的实战指南：如何在 `config.toml` 中配置大模型、多号池、IM 频道接入以及搜索爬虫配置等。

3. **[🧠 Agent 核心引擎 (Agent Core Engine)](03_agent_core_engine.md)**
   > 探究 `src/agent/` 和 `src/runtime/`。解密智能体的核心决策循环（思考 -> 规划 -> 执行），以及目标（Goal）与标准操作程序（SOP）的调度机制。

4. **[🔌 大模型接入层与配置池 (Providers & Config)](04_llm_providers.md)**
   > 探究 `src/providers/`。深度剖析基于 Trait 的 LLM 抽象，特别是独特的 **OpenAI OAuth 原生支持与多号池高可用 (Failover) 轮转机制**。

5. **[🛠️ 动作执行与工具箱沙箱 (Tools & Skills WASM)](05_tools_and_skills.md)**
   > 探究 `src/tools/` 和 `src/skillforge/`。梳理内置工具的注册机制，以及如何在受限的 WASM 环境下安全地动态加载第三方 Skill。

6. **[💬 交互通道与网关 (Channels & Gateway)](06_channels_gateway.md)**
   > 探究 `src/channels/` 和 `src/gateway/`。ZeroClaw 作为服务器运行时，如何通过灵活的 Channel 机制无缝接入如 Discord、Telegram、Matrix 甚至提供原生的 HTTP API。

7. **[💾 记忆持久化与基础设施 (Memory & RAG)](07_memory_infrastructure.md)**
   > 探究 `src/memory/` 和 `src/rag/`。解密极致精简架构下的短期与长期记忆管理（SQLite / Postgres 支持），以及对文本向量化的原生支持。

8. **[🛡️ 安全机制与内网穿透 (Security & Tunnels)](08_security_tunnels.md)**
   > 探究 `src/security/` 和 `src/tunnel/`。重点分析基于 Tailscale 的内网穿透能力，以及从底层保障 Agent 运行和网络暴露安全的鉴权链路。

---

*“Zero overhead. Zero compromise. Deploy anywhere. Swap anything.”*
开始你的源码解析之旅吧！ 🦀
