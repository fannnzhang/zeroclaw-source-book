# 9. 时空调度、系统驻留与高可用保障 (Scheduling, Daemon & Reliability)

对于一般的命令行大模型工具来说，执行完毕就退出了。但 ZeroClaw 的定位是“赛博管家”与“自治智能体”。它需要跨越时间的维度，在后台做到 24 小时高可用。

这些特性集中实现在了 `src/cron/`（定时调度）、`src/daemon/`（系统守护）、`src/health/`（健康流检）以及 `src/heartbeat/`（心跳自愈）四大模块中。

---

## 9.1 定时引擎 (`src/cron/scheduler.rs`)

由于 ZeroClaw 要求极致轻量脱离第三方依赖库（不再运行体积庞大的 Kubernetes 或独立的 Redis/Celery 队列等），其内置了一套强健的 Cron 调度引擎。

它能够解析标准的 Linux 5域 Cron 表达式（如 `0 9 * * 1-5` 周一至周五早上 9 点）。
不仅如此，这套引擎专门针对 AI 场景做了改良：

1. **Job 路由类型**:
   * **Agent 任务**: 将指令（如“抓取今天早上的 HN 新闻汇总成全天报表”）定点无缝传递给 Core Loop 进行 LLM 推理。
   * **Shell 任务**: 在特定的高权环境下执行 `backup.sh` 脚本，同时将 Stdout 的反馈喂给大模型作为上下文储备。
2. **主动投递反馈 (Deliver)**:
   不同于传统的定时脚本输出抛到 `/dev/null` 里。在 Cron 中配置了 `Deliver Target` 后，一旦某周一早上的抓取简报生成完毕，ZeroClaw 会立马通过 `src/channels/` 中的 Telegram / 钉钉 等 IM 发送到相关人群聊。
3. **安全熔断超时 (`SHELL_JOB_TIMEOUT_SECS`)**:
   每个后台任务都被隔离在独立的 `tokio` 任务组，并带有硬超时的 120秒 截断机制，以防御大模型产生的异常挂死脚本导致整机耗电/CPU卡死。

---

## 9.2 Daemon 守护与防死监工 (`src/daemon/mod.rs`)

`zeroclaw daemon` 命令是启动全功能系统的入口。在 `src/daemon/mod.rs` 中，不负责发包也不负责处理 AI，它充当的是一个极其严格的 **系统管理员 / Supervisor (进程管理器)**。

```rust
// 摘自 mod.rs 下的重启背压控制
fn spawn_component_supervisor<F, Fut>(
    name: &'static str,
    initial_backoff_secs: u64,
    max_backoff_secs: u64,
    mut run_component: F,
) -> JoinHandle<()>
```

### 退避/崩毁容忍 (Backoff Restart):
当内部网关 (Gateway)、或者通过 WebSocket 连入外界的 Discord 连接意外被截断抛出崩溃（Panic / anyhow Error）时，这套 Supervisor 会立刻捕获崩溃，不让进程退出，并在记录日志后执行带有**指数退避 (Exponential Backoff)** 策略的重启。

---

## 9.3 自然流的挂机与唤醒：Heartbeat.md (`src/heartbeat/`)

这是 ZeroClaw 中非常浪漫的一个设定。普通的系统探活是用 HTTP Ping/Pong 进行监控的。而 ZeroClaw 采用的是：**人类自然语言的 `HEARTBEAT.md`**。

在 `src/heartbeat/engine.rs` 中，实现了一个每 N 分钟触发一次的自醒（Tick）引擎。

这个引擎会走到 Agent 当前的工作区下找到 `HEARTBEAT.md`，它仅仅只是去读取 Markdown 中的 `* ` 或 `- ` 无序列表。
如果里面写着：
```markdown
- 请时刻关注推特上的 @elonmusk 是否发了新推币
- 去抓取 error.log ，看是否有超过 3 条系统报错，如果有就总结通知我
```

**运行流：**
Agent 的 `tick` 事件读到这行字，触发一轮 Core Engine 唤醒，调用对应的 Tool（Web 搜索或 Log 查阅），判定完毕后自动陷入沉睡。无需任何配置、JSON编写或代码接入，就达成了真正意义上的“人类语言设定 Agent 的生命周期目标”。

---

## 9.4 全链路自动会诊系统 (`src/health/`)

当如此多复杂的环境（Token 会过期、IM 连接会断流、端口会被占用、物理硬件会松动）组合在一起，运维该如何定位问题？

`src/health/mod.rs` 提供了一个 `Doctor` 大前端探针汇总能力。
如果在 CLI 执行 `zeroclaw doctor`，系统会自顶向下穿透调用每个 Trait 中约定的 `health_check()` 函数。如果有任何配置文件的证书存在问题、或是 SQLite 出现了损坏，这套代码都将用友好的红色报告在控制台上向人类告警。

> **下一章预告：** 
> 系统现在能够 24 小时运行并呼风唤雨（软硬双杀），此时，一个最严峻的命题登场了——**绝对安全性控制与云配额消耗管控**。最后一章我们将解析：**Chapter 10. Security, Approval & Economic**。
