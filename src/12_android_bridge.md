# 第十二章：Android 桥接集成 (Android Bridge)

ZeroClaw 提供了一个原生的 Android 集成方案，位于 `clients/android-bridge` 目录。该模块的核心目标是将 ZeroClaw 的 Rust 核心引擎（Agent 与 Gateway）直接嵌入到 Android 宿主应用中，使移动端设备能够在本地（局部或完全脱机，或直接连接 LLM Provider）运行高性能的 AI 代理。

## 1. 核心技术栈与架构

Android 桥接层并没有采用传统的 JNI 手写绑定，而是使用了 Mozilla 开发的 **UniFFI** 工具链。

- **UniFFI (`uniffi` crate)**: 通过 `uniffi::setup_scaffolding!()` 以及 `#[uniffi::export]` 宏，自动生成 Rust 到 Kotlin/JNI 的安全绑定层（Scaffolding 代码）。
- **隔离的 Tokio Runtime**: 在 Android 进程中，Rust 代码无法直接依赖宿主提供的线程池来执行异步操作。因此，源码中通过 `std::sync::OnceLock` 维护了一个全局唯一的 `tokio::runtime::Runtime` 实例，提供给 Gateway 和 Agent 使用。

## 2. 暴露的核心能力 (Kotlin API)

`clients/android-bridge/src/lib.rs` 的核心是暴露了一个统一控制器对象 `ZeroClawController`，供 Android 端调用，主要提供以下几方面的能力：

### 2.1 生命周期与网关控制
控制器暴露了 Agent 网关的生命周期方法，允许 Android App 随时启动或挂起 AI 服务，以最大化省电表现：
- **`start()` / `stop()`**: 控制底层的 ZeroClaw Gateway 的运行。
- **`get_status()`**: 随时查询守护状态，返回 `AgentStatus` 枚举（如 `Stopped`, `Starting`, `Running`, `Thinking`、`Error` 等）。

### 2.2 原生配置管理
Android 环境和标准 Linux 系统不同，没有标准的配置文件目录路径。因此控制器暴漏了环境配置传入能力：
- 结构体 `ZeroClawConfig`：允许 Kotlin 端传入应用专属内部存储沙盒路径（`data_dir`）、指定的提供商 (`provider`)、选用的模型 (`model`)、 `api_key` 以及系统级别提示词 (`system_prompt`)。
- **动态配置能力**: 提供了 `update_config()` 与 `get_config()` 和 `is_configured()` 方法，可以实现如 App 设置面板里实时更改提供商然后让底层引擎即可生效的设计。

### 2.3 消息与会话同步
跨越语言边界来传递聊天记录比较繁琐，Bridge 层提供了一套清晰的消息语义：
- **`send_message(content)`**: 从设备端投递自然语言指令到 Agent。该方法同步或者异步返回 `SendResult` 结果结构。
- **`get_messages()` / `clear_messages()`**: Android 端 UI 可以直接抓取被结构化的 `ChatMessage` (ID, Content, Role, Timestamp) 用于绘制气泡聊天流，不需要手动解析繁琐的 JSON。

## 3. 当前实现状态与未来展望

深入 `Cargo.toml` 和代码实现会发现，该桥接层目前**正处于 PoC（概念验证）或早期结构搭建阶段**：

1. **底层依赖未挂载**:
   在 Cargo 依赖中记录了 `# zeroclaw = { path = "../.." }`，目前处于注释掉的状态。
2. **逻辑仍在 Stub (存根) 阶段**:
   在 `ZeroClawController::start()` 的实现中，原有的 Gateway 启动逻辑暂时被 `TODO` 块包裹起来；`send_message()` 目前默认返回一个用于调试验证 UI 的 "Echo" 响应助理消息，而不是真正将指令扔给大语言模型调度器。

### **总结**

`clients/android-bridge` 描绘了 ZeroClaw **跨端嵌入**的蓝图：利用 Rust 的跨平台编译能力和极小的内存占用，以及 UniFFI 的现代化 FFI 方案，使 Android 开发者不再需要写一个云端服务器即可在智能手机本地运行拥有复杂调度与记忆能力的“胖客户端”数字生命管家组件。虽然当前该模块仅完成了数据模型与外层桩代码定义，但其整体设计方向已非常明确。
