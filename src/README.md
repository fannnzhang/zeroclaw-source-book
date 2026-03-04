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

5. **[⚙️ 核心执行引擎层 (Core Agent Engine)](05_core_agent_engine.md)**
   > 探究 Agent 核心状态机的运转机制，深入 `src/agent/` 的决策循环与上下文组装过程。

6. **[💾 全量记忆体与检索引擎 (Memory & RAG)](06_memory_and_rag.md)**
   > 探究 `src/memory/` 和 `src/rag/`。解密极致精简架构下的短期与长期记忆管理（SQLite / Postgres 支持），以及对文本向量化的原生支持。

7. **[💬 多端通道互联与网关通信 (Channels & Gateway)](07_channels_and_gateway.md)**
   > 探究 `src/channels/` 和 `src/gateway/`。从统一的 `Channel` Trait 到工业级的 Axum 网关，解析限流、幂等、流式响应等生产级能力。

8. **[🛠️ 工具链与智能外设 (Tools & Hardware IoT)](08_tools_and_hardware.md)**
   > 探究 `src/tools/`。梳理内置工具的注册机制以及硬件/IoT 设备的接入扩展。

9. **[⏰ 时空调度、系统驻留与高可用保障 (Scheduling & Reliability)](09_scheduling_and_reliability.md)**
   > 探究 `src/daemon/` 和调度子系统。解析定时任务、守护进程驻留与高可用容错机制。

10. **[🛡️ 绝对安全防御与生存经济学 (Security, Approval & Economic)](10_security_approval_and_economic.md)**
    > 探究安全鉴权链路、高危操作人工审批机制，以及 Token 配额经济与成本过载防护。

11. **[🧩 插件体系架构 (Plugin Architecture)](11_plugin_architecture.md)**
    > 探究 `src/plugins/` 目录。解密基于 WebAssembly 的插件沙盒加载、动态注册与扩展机制。

12. **[📱 Android 桥接集成 (Android Bridge)](12_android_bridge.md)**
    > 探究 `clients/android-bridge/`。解析通过 Mozilla UniFFI 实现的 Rust 原生智能体与 Kotlin/Android 的跨端无缝集成。

---

*“Zero overhead. Zero compromise. Deploy anywhere. Swap anything.”*
开始你的源码解析之旅吧！ 🦀
