# Cadence Reset 与恢复矩阵：区分不同 Reset 的作用范围

## 1. 先按作用域分 reset

| 事件/输入 | 作用域 | Link 是否受影响 | 关键保存/清除行为 |
|---|---|---|---|
| cold/conventional reset chain | core/PHY/system | 重新训练 | 依 reset domain 初始化 |
| Link Down Reset | 整条 Link core | Link 已 down/disabled | core state/timer/config非sticky清；AXI translation保留；LM是否保留存在手册冲突 OQ-011 |
| Hot Reset | 整条 Link | 会导致 Link Down Reset | 数据面响应同 Link Down |
| FLR | 单个 PF/VF Function | Link 不受影响 | 该 Function config reset、BME清；LM/AXI translation保留 |
| `MGMT_RESET_N` | management/config domain | 不直接等同 Link reset | LM与config恢复默认，sticky例外 |
| `MGMT_STICKY_RESET_N` | sticky config bits | 不直接等同 Link reset | 清 Spec 指定 sticky bits |

恢复代码若只有一个 `init_all()`，很可能不是过度勤劳就是漏配；先知道谁被 reset 了，才知道该重建谁。

## 2. Link Down / Hot Reset 数据面行为

Cadence 内部生成 Link Down Reset，并置 AXI Wrapper `Link Down Indication`：

- application outbound AXI write 被 bridge consume，但返回 `SLVERR`；
- outbound AXI read 返回 zero data + `SLVERR`；
- `LINK_DOWN_RESET_OUT` 通知 client 清自己的 FIFO/state；
- AXI Address Translation registers 保留；Local Management 是否保留存在 §4.7.2 与 §11.1.2.3 的直接冲突；
- PCIe Configuration 除 sticky 字段外 reset。

只有看到 `LINK_DOWN_RESET_OUT` falling edge 后，固件才应清 Link Down Indication 并恢复有效 traffic。过早清相当于状态机还在拆舞台时催下一场开演。

Hot Reset 最终落到同一 Link Down Reset 行为。

## 3. FLR：只重置一个 Function

Host 向 Device Control 的 FLR bit 发 Config Write；controller 先返回 Completion，再开始 FLR：

1. assert 对应 `FLR_IN_PROGRESS`/`VF_FLR_IN_PROGRESS`；
2. client 停止该 Function 新 outbound write、reset function-owned queue/context；
3. 进行中的 write 正常完成；affected Function 的 read 返回 AXI `SLVERR`；
4. client 必须在 100 ms 内 pulse `FLR_DONE`；
5. FLR 期间 Config/Memory/IO 默认可被静默丢弃；LM `Enable CRS under FLR` 可让 Config Request 回 CRS；
6. Host 重新写 BME 后，该 Function 才能恢复 outbound traffic。

FLR 不清 Local Management，也不清 AXI Address Translation。共享这些资源的多 Function 设计必须做 ownership/refcount，不能因为 PF0 FLR 把 PF1 正在用的全局窗口拆掉。

## 4. 恢复清单不能混用

### Link Down 后

- quiesce 所有该 Link Function/queue；
- 收集 LTSSM/AER/uDMA status 后再 W1C；
- 等 reset output 释放，重训 Link；
- 清 Link Down Indication；
- 配置空间需重新枚举/恢复，BME/MSE重新确认；
- translation 可校验但不必无脑重写；关键 LM 必须 readback，必要时重写；
- uDMA 根据 residue/descriptor status 决定重试。

### FLR 后

- 只处理目标 PF/VF；
- 等 client reset completion；
- 重建 function queue/MSI-X table ownership/device-private state；
- 等 Host 重配 Command/BAR/capability/BME；
- 不重训 Link，不破坏其他 Function。

### Hot Reset 后

按 Link Down Reset 处理，并额外记录 Hot Reset 来源/链路训练历史。

## 5. Linux 恢复映射

Linux Function reset、AER error recovery、hotplug/remove 和 host-controller Link recovery 是不同入口。共同原则是：先阻止新 I/O，再 drain/abort outstanding，再按 reset scope 重建 resource/IRQ/DMA。X6000 Linux 5.10.4 已提供 PCI core、AER/DPC 和 `pcie-edge*.c`；但源码中尚未以 Cadence Databook 的字段名闭合“由谁清 Link Down Indication”这一动作，仍需用目标寄存器 offset、wrapper 定义和故障注入 trace 对照，因此该点保留 Unknown。

## 6. Source Reference

- PCIe Base Specification 5.0 r1.0, §6.6 reset、§7.5.3.4 FLR。
- Cadence PCIe Core User Guide, §§4.7.1–4.7.4, printed pp.276–278 / PDF pp.365–367。
- Cadence PCIe Core User Guide, Chapter 11 reset operation, especially §11.1.2。

## 7. 手册冲突：OQ-011

§4.7.2 说 Local Management Registers 在 Link Down Reset 中不被清除、无需重编；§11.1.2.3 说除 sticky configuration 外，其余 configuration 与 Local Management 都 reset。两者无法同时成立。

安全实现策略是：保留一份关键 LM shadow，Link Down 后 readback 验证，若与期望不符则在 training/traffic允许的阶段重写；同时通过 RTL simulation、errata 或实机写入非默认哨兵值再触发 Link Down 来确认当前 build。不要用 destructive bit、W1C status 或会改变 Link 的字段做哨兵。
