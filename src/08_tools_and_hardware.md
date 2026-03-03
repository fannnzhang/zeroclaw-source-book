# 8. 工具链与智能外设 (Tools & Hardware IoT)

ZeroClaw 之所以能够自称为真实的“具身智能 (Embodied AI) 引擎”，最大的底气来源于它极其庞大且安全的泛用工具箱库（`src/tools/`），以及深植底层的物理硬件控制总线（`src/hardware/` 和 `src/peripherals/`）。

---

## 8.1 超级工具箱库的设计 (`src/tools/`)

打开 `src/tools/mod.rs`，你会发现一幅非常壮观的宏大景象。ZeroClaw 内部一共预置了超过 **70 款实用的系统级及应用级 Tool**。

### A. Trait 与统一执行协议
每个工具都必须实现标准的 `Tool` Trait：

```rust
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> serde_json::Value; // JSON Schema 给大模型理解参数结构

    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult>;
}
```
LLM 输出 JSON，反序列化进 `args`，经过执行变为 `ToolResult`，再打包成 `ToolMessage` 喂给下一轮模型 Prompt。

### B. 按领域分类的预置 Tool
这 70 多个工具涵盖了：
*   **Web 浏览域**: `BrowserTool` (大名鼎鼎的 MCP Browser / Playwright 驱动层), `WebFetchTool`, `WebSearchTool`。
*   **研发构建域**: `ShellTool` (执行 Bash 脚本), `FileEditTool`, `FileReadTool`, `GitOperationsTool`。
*   **文件解析域**: `PdfReadTool`, `PptxReadTool`, `DocxReadTool`, `XlsxReadTool` (本地免 API 的纯底层解析)。
*   **Agent 编排与记忆流**: `SubAgentSpawnTool`, `TaskPlanTool`, `MemoryStoreTool`, `MemoryRecallTool`。

---

## 8.2 沙箱与安全拦截 (`SecurityPolicy`)

Agent 的不可控性意味着，如果它错误理解了你的意思直接执行 `rm -rf /` 会带来灭顶之灾。
在把工具抛给执行层面之前，它必须经过 `SecurityPolicy` 与 `ApprovalManager` 的严格审查。

一旦涉及到系统突变的 Tool（如 `ShellTool`、`FileWriteTool`），ZeroClaw 在构建这些工具实例的时候会将 `Arc<SecurityPolicy>` 注入。这些 Tool 在真正发生内部执行动作时，会向上发起安全询问，或触发我们第 7 章讲过的网关动态审批交互。

---

## 8.3 IoT 边缘计算与软硬结合 (`hardware/` & `peripherals/`)

这是 ZeroClaw 在所有 AI 框架中最独树一帜的重底层架构能力。目前市面上的框架几乎全为云上的纯软调用，而 ZeroClaw 将触手伸向了集成电路世界。

由于 ZeroClaw 的引擎能在一台 10 美金的单板计算机（如含有不到百兆内存的 Linux 设备）甚至未来的极低功耗板上运行，它天然适合作为 IoT 网关大脑。

### A. 硬件发现引擎 (`src/hardware/discover.rs`)
系统底层封装了 `discover_hardware()`，通过读取主机底层的 USB 总线或串口，能够一键自动嗅探插入的调试探针或设备（如 ST-Link、J-Link 甚至是纯串口转换芯片）。

### B. 外设挂载器 (`src/peripherals/mod.rs`)
当配置挂载了一块诸如 STM32 Nucleo 或 Arduino Uno 或 Raspberry Pi 时，`peripherals` 模块会在初始化时，将与硬件建立的底层通讯信道**动态转化为 Agent Tool**：

1. **GPIO 电平翻转**: 衍生出 `HardwareGpioTool`，AI 可以知道自己身上哪些针脚是可用的。
2. **寄存器查阅**: `HardwareMemoryMapTool`，配合上一章提到的 RAG，让 AI 读取针脚映射表。
3. **Firmware 烧录**: `usb_flash_tool` 甚至赋予了 Agent 重新为下位机编译并刷写固件的能力。

> **终极闭环场景：**
> “读取这颗传感器的指标，若低于 20度，将 5号引脚电平置高开启加热器”。
> 这套命令不再需要程序员写 Python 胶水硬编码。大模型结合 `src/rag/` 中读取到的该型号元件的说明书，自我规划出了调用的外设针脚并通过 `execute()` 实施，这一连串的魔力全部归功于以上架构的彻底打通。
