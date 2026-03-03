# 3. Agent 核心引擎与流转架构 (Core Engine)

ZeroClaw 的核心执行图灵完备，基于 ReAct + Tool Calling (工具调用) 思想驱动，相关的实现收敛在 `src/agent/` 和底层的 `dispatcher.rs`、`loop_.rs`。

本章节将剖析其流转逻辑、状态管理以及并发控制。

---

## 3.1 核心构建与抽象 (`agent.rs` & `Builder`)

ZeroClaw 在构建 Agent 实例时提供了高度可组装的工业级抽象 (`AgentBuilder`)：
* 注入 `Provider`（底层 LLM 算力基座）。
* 挂载一组 `Box<dyn Tool>` 技能列表，并通过 `ToolDispatcher` 管理解析。
* 挂接 `Memory`（长短记忆驱动）及 `MemoryLoader`，以及安全 `Observer`（观察针脚）用于链路溯源。

`Agent` Struct 保存的不仅仅是静态配置，还通过 `history` 数组和 `trim_history` 滚动丢弃策略维护着当前局部的对话生命线 (Conversation Context)。

---

## 3.2 引擎心脏：递归执行流 (`loop_.rs`)

整个框架最核心的心脏是一段巨型的 `run_tool_call_loop` 循环体，它控制了用户提问、大模型推理、工具调用的核心流转链路。

### A. 智能调度与工具并发执行 (Parallel Tool Execution)
大模型在一次推理中经常会吐出多个独立指令（例如同时查询天气与读取日历）。`execute_tools_parallel` 会分析这些指令的读写依赖，在底座支持的情况下生成 Tokio 的并发任务 (JoinSet)，以达到纳秒级并行的异步工具执行效率。

### B. 限流与 Quota 熔断感知 (`quota_aware.rs`)
区别于实验性项目，ZeroClaw 加入了深度的企业级开销控制。核心引擎直接嵌合了配额检查：在进行昂贵的模型调用和耗时工具执行前，`check_quota_warning` 被触发，如果 API 余量耗尽或频控超时，引擎可以挂起或优雅降级（甚至触发 `switch_provider` 切模），确保流水线的“软着陆”。

### C. 长文本阻断自动续写 (MAX_TOKENS_CONTINUATION)
由于硬件约束或 API `max_tokens` 截断，LLM 输出代码或长文本极易发生“腰斩”。
ZeroClaw 引擎内部提供了一个确定的 `Token Continuation` 循环，当检测到输出异常截断时，引擎会自动构造类似 `"Previous response was truncated by token limit. Continue exactly from where you left off..."` 的拼接指令给模型追问，然后在接收端实施无缝的字符串拼接合并，向上层用户透明返回。

---

## 3.3 护栏：死循环哨兵 (`LoopDetector`)

当开放自由度的工具交给模型后，模型经常会陷入“调用工具失败 -> 重试同样的错误参数 -> 再次失败”的**死循环 (Infinite Loop)**。由于按 Token 计费，这可能几分钟内烧光配额。

核心引擎内嵌了一个状态机级别的哨兵：`LoopDetector` (`src/agent/loop_/detection.rs`)。
1. **调用指纹提取**: 将模型触发的具体工具和序列化参数抽取为签名。
2. **重复模式扫描**: 以小窗口匹配历史日志，识别出例如 `(A -> B -> A -> B)` 或重复重试单一错误的模式。
3. **熔断与干预**: 一旦命中阈值，立即进行拦截抛出异常，或在下一次 Prompt 输入强制追加警告 `"You are repeating the same failed tool calls. Please stop or change your approach."`。

---

## 3.4 下游分发：路由抽象 (`dispatcher.rs`)

执行引擎拿到 LLM 输出结果后，不再通过硬编码匹配。
通过 `ToolDispatcher` Trait，系统支持动态的适配拆包。目前主要的派发器支持了 `NativeToolDispatcher`（对原生大语言模型的 Function Calling 进行对接）和 `XmlToolDispatcher`（对不支持原生能力的模型，强制转换为特定格式的 XML / Markdown block 取参解析）。

> **总结**: ZeroClaw 的内核被设计成一台不知疲倦的状态机，兼收并蓄了内存管收、调用截断拼接、无限自环检测、并发工具流机制，是一套真正能承受工业界灰度乱象的坚固引擎。
