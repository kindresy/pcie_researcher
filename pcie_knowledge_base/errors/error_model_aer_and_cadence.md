# PCIe 错误模型与 AER：从一颗坏 bit 追到 Linux recovery callback

## 1. “PCIe error”不是一种错误

PCIe 把多个可靠性层叠起来，因此错误也按发生位置和影响范围分层：

```text
Physical:    receiver/framing/block alignment/equalization
Data Link:   bad LCRC/DLLP、replay timeout/rollover、flow-control protocol
Transaction: UR、CA、Unexpected Completion、Malformed、Poison、Completion Timeout
Port/Link:   Surprise Down、DPC
Controller:  FIFO overflow、ECC、internal parity、AXI response
```

先问“哪一层、谁检测、影响哪笔 transaction”，比先问“是不是 AER”更有效。AER 主要是统一的记录/通知能力，不是神奇修复按钮。

## 2. Correctable、Non-Fatal、Fatal

| 分类 | 协议含义 | 典型例子 | 常见恢复粒度 |
|---|---|---|---|
| Correctable | 硬件恢复，无信息丢失；也含部分 advisory 情形 | Receiver Error、Bad TLP 后 replay | 记录/计数，通常无需 driver recovery |
| Uncorrectable Non-Fatal | 某笔 transaction 不可靠，但 Link 可继续工作 | UR、CA、Completion Timeout（默认常为 non-fatal） | 终止/重试业务，可能 Function recovery |
| Uncorrectable Fatal | Link/相关硬件已不可信 | Receiver Overflow、严重 protocol/internal error（依 severity） | containment、reset、重建 hierarchy |

Fatal/Non-Fatal 是 PCIe severity，最终平台动作仍可由 firmware/OS policy 决定。AER 的 Uncorrectable Error Severity register 允许逐 error 选择 fatal/non-fatal；baseline error reporting 没有这个细粒度。

Correctable 也不等于“可以永远不管”。偶发 LCRC 被 replay 是硬件成功修复；每秒几千个 Correctable Error 则像汽车稳定控制灯一直闪——车还能走，但路或车肯定不太对。

来源：Base Spec 5.0 §§6.2.1–6.2.2，printed/PDF pp.505–507。

## 3. 三种互补的错误传播方式

### 3.1 Completion Status：告诉原 Requester 这笔请求失败

对 Non-Posted Request，Completer 可返回：

- **UR**（Unsupported Request）：请求类型/地址/属性不受支持，或没有合适 Completer；
- **CA**（Completer Abort）：Completer 接受到了请求，但内部原因无法完成；
- **CRS**：Configuration Request Retry Status，Function 暂未准备好；
- **SC**：Successful Completion，不是错误。

Requester 用 Requester ID + Tag 关联原请求。Completion Status 是 transaction-local 的结果，不等同于已经向 Root Complex 上报 error event。

### 3.2 ERR_COR / ERR_NONFATAL / ERR_FATAL：向 Root Complex 报事件

这是路由到 Root Complex 的 Message TLP。Root Port 根据 Requester ID 记录 source，把它转成平台事件/中断。AER mask、Device Control reporting enable、Root Error Command 等共同决定是否发送/是否中断。

同一错误可能由不同观察者以不同级别报告。Role-Based Error Reporting 还定义 Advisory Non-Fatal：例如探测不存在 Function 得到 UR，本身可能是软件正常 probing；Completer 若再发一个会导致系统恐慌的 ERR_NONFATAL，反而过度热情。因此某些 detector 用 ERR_COR advisory，或不额外发 message。

### 3.3 Poison：数据继续走，但标记“不可信”

TLP header 的 `EP` bit 是 Error Poisoned。它允许系统把带错误的数据沿 transaction 路径转发，由最终消费者处理/报告。Poison 不是 Endpoint 的缩写，也不意味着 packet 的 LCRC 一定错；LCRC 保护“传输是否改变”，Poison 表示“payload 在更早处就已知不可靠”。

来源：Base Spec §§6.2.3–6.2.3.3，printed/PDF pp.507–513。

## 4. 几个驱动最常遇到的错误

### 4.1 Unsupported Request（UR）

直觉：收件地址/业务类型合法地到达某处，但那里说“我不支持这单”。

常见原因：

- 访问未实现 BAR/offset 或桥 window/routing 配错；
- Function 不支持某 TLP/type/AtomicOp；
- 配置访问不存在的 BDF；
- Memory Space Enable/目标 decode 条件不满足；
- Link down/DPC 边界上的 request 被 Port 以 UR 终止。

对需 Completion 的 request 返回 Cpl(UR)；对 Posted request 无 Completion，只能记录/上报 UR。

### 4.2 Completer Abort（CA）

直觉：Completer 认领了这笔请求，但内部执行失败，主动“弃单”。Cadence inbound Memory Read 若本地 AXI `RRESP=SLVERR/DECERR`，会返回 Cpl(CA)。这条映射把 PCIe 错误精确落到 SoC 内部 AXI target。

### 4.3 Unexpected Completion

收到的 Completion 找不到匹配的 outstanding Requester ID + Tag，或上下文不匹配。接收端丢弃并记录；真正等待它的 requester 可能随后 Completion Timeout。常查 tag reuse、reset/FLR/link flap、错误 routing、重复/迟到 completion。

### 4.4 Completion Timeout（CTO）

Requester 发了需 Completion 的 request，直到配置的 timeout interval 仍没收齐。大 MRd 只有部分 CplD 返回也未完成。CTO 是 **Requester Function** 的错误，而不是天然属于 Completer。

可能根因从“request 根本没出去”、远端 BAR/AXI hang、completion 返回丢失，到 async removal；timeout 值不是典型 latency 指标，不能拿它做性能规格。

来源：Base Spec §2.8，printed/PDF p.204。

### 4.5 Malformed TLP

header/packet 违反格式或字段组合规则，例如非法 Length/Fmt/Type/TC、应保留字段不合规等。它是结构性协议错误，不等同于 LCRC 错。Cadence 对 malformed TLP 不交 AXI，丢弃并置 AER/Local Error status。

### 4.6 Surprise Down / Link Down

DL_Active 在没有被 Secondary Bus Reset、Link Disable、正常 power-off/hotplug policy 等预期条件解释时突然进入 DL_Inactive，可报告 Surprise Down。此时在途 Non-Posted request 可能 CTO，Posted request 可能静默丢失或伴随 Port-side UR reporting；症状会像烟雾一样向上层扩散。

## 5. AER Extended Capability 是什么

AER（Advanced Error Reporting）位于 Extended Configuration Space。关键 register：

| Offset | Register | 核心作用 |
|---:|---|---|
| `+0x04` | Uncorrectable Error Status | 哪类 uncorrectable 已发生，RW1C |
| `+0x08` | Uncorrectable Error Mask | 屏蔽该类向 RC 报告/log，不等于阻止检测 |
| `+0x0C` | Uncorrectable Error Severity | 0=Non-Fatal，1=Fatal |
| `+0x10` | Correctable Error Status | Receiver/Bad TLP/Bad DLLP/replay/advisory 等，RW1C |
| `+0x14` | Correctable Error Mask | 逐项 reporting mask |
| `+0x18` | Capabilities and Control | First Error Pointer、ECRC、multiple-header 等 |
| `+0x1C..+0x28` | Header Log | 与 first uncorrectable error 相关的最多 16 B TLP header |
| Root only `+0x2C` | Root Error Command | correctable/nonfatal/fatal interrupt enable |
| Root only `+0x30` | Root Error Status | 收到的 ERR_* 与 multiple error 状态 |
| Root only `+0x34` | Error Source ID | 首个 correctable/uncorrectable Requester ID |

Uncorrectable bits包括 Data Link Protocol、Surprise Down、Poison、Flow Control、CTO、CA、Unexpected Completion、Receiver Overflow、Malformed、ECRC、UR、ACS Violation、Internal Error 等。Correctable bits包括 Receiver Error、Bad TLP、Bad DLLP、Replay Number Rollover、Replay Timer Timeout、Advisory Non-Fatal、Corrected Internal Error。

### Status、Mask、Severity 别互换

- **Status**：发生过；多数是 sticky RW1C/W1C。软件写 1 清对应 bit，写 0 不动。
- **Mask**：是否进一步 log/report；被 mask 的错误通常仍影响 status。
- **Severity**：若是 uncorrectable，按 fatal 还是 non-fatal 报。
- **Enable**：Device/Root 层还各有 class-level enable；单项没 mask 不代表一定会产生平台 IRQ。

清 AER 时应先采集 status、First Error Pointer、Header Log、source、controller local status，再按要求 W1C。上来先写全 1，虽然屏幕清爽，却相当于案发现场先拖地。

来源：Base Spec §§7.8.4.1–7.8.4.11，printed/PDF pp.842–856。

## 6. Header Log 怎样帮你还原 TLP

Header Log 保存 4 DW。按 spec 的 DWORD 读法，字段布局与 TLP header 图一致，但每个 DW 内的 byte 展示要留意 endian。解码顺序：

1. 由 Fmt/Type 判断 TLP 类型和 3DW/4DW；
2. 看 Length、TC、Attr、EP；
3. request 看 Requester ID、Tag、BE、address；completion 看 Completer/Requester ID、Tag、status；
4. 用 First Error Pointer 确认 header 对应哪个 uncorrectable status；
5. 再联系 Port 方向判断是 inbound 还是 outbound。

并非每个 error 都必有可用 header；masked error 不进入 Header Log，多个错误还受 first-error/multiple-header 规则影响。CTO 记录的是原 Request header 也需要 capability 宣告支持。

## 7. Cadence：协议错误怎样落到 AXI、寄存器和 sideband

Cadence Appendix B 很有工程价值，因为它不是只列 bit，而是同时列 **AXI response / AER status / client action / local interrupt / severity output**。

典型映射：

| 事件 | AXI 行为 | 记录/输出 |
|---|---|---|
| inbound MRd 本地 AXI error | core 回 Cpl(CA) | CA/target-abort 相关状态 |
| outbound MRd 收 Cpl(UR/CA) | AXI Slave 返回 `SLVERR` | AER UR/CA；client 忽略 read data |
| Completion Timeout / targeted FLR | AXI `SLVERR` | AER CTO + sideband bit 21 |
| Unexpected Completion | 不送 AXI | AER Unexpected + Local Error + sideband 19 |
| Malformed TLP | 丢弃，不送 AXI | AER Malformed + sideband 18 |
| Bad TLP/DLLP LCRC | replay/无 AXI 影响 | Correctable Bad TLP/DLLP |
| Replay Timeout/Rollover | 无直接 AXI 影响 | Correctable + Local Error + sideband 15/16 |
| FIFO overflow | 造成 packet/drop | Receiver Overflow；client 需 reset |
| Link Down | AXI bridge indication 后访问 `SLVERR` | `LINK_DOWN_RESET_OUT`；FW 清状态后重启 |

`INTERRUPT_SIDEBAND_SIGNALS` 已被相应 Local Management mask 内部过滤，customer 可 OR 选定 bits 形成本地 IRQ。`FATAL_ERROR_OUT`、`NON_FATAL_ERROR_OUT`、`CORRECTABLE_ERROR_OUT` 则受 AER severity/mask 影响。不要把 sideband error IRQ 和线上 ERR_* Message 当成同一根信号：一个通知 SoC 本地 client，一个沿 PCIe 向 RC 报告。

来源：Cadence Appendix B，printed pp.1332–1337 / PDF pp.1421–1426；§4.6.2，printed pp.274–276。

## 8. Linux AER 的位置

概念路径：

```text
device/Port detects error
 → local AER status/header log
 → optional ERR_* Message upstream
 → Root Port Root Error Status/source + platform IRQ/MSI
 → Linux PCIe port service / AER service
 → print severity/source/TLP header
 → decide recovery scope
 → pci_driver.err_handler callbacks
```

支持 PCI error recovery 的 driver 通常提供 `struct pci_error_handlers`，概念阶段包括 `error_detected()`、可选 `mmio_enabled()`、`slot_reset()`、`resume()`；channel state 和各 driver vote 决定是否 reset/recover/disconnect。

现已补入X6000 Linux 5.10.4和Host `edge.ko`，目标driver的AER callbacks、DMA quiesce、
slot reset/resume主链见`code_studies/edge_driver_x6000.md`；X6000 controller错误IRQ见
`code_studies/linux_x6000_pcie_stack.md`。不同kernel版本和非X6000 DPC/AER内部锁序
仍需以对应源码为准。

## 9. Error pollution：别让一个根因生出一串假线索

同一坏 packet 可能在 Physical、Data Link、Transaction 多层都看起来有错。Spec 定义 precedence/抑制原则，优先报告更靠近根因或更严重的错误。Cadence 的优先级从 Internal Error、Receiver Overflow、Flow Control、Malformed、ECRC，到 UR/CA/Unexpected、Poison。

所以 log 中“只看到 Malformed”不证明没有更底层波形问题；反过来，一次 Receiver Error 后出现 replay 属于同一恢复故事，不要分别开十张 bug 单。

## 10. Source Reference

- PCI Express Base Specification 5.0 r1.0, §2.8–2.9, printed/PDF pp.204–207。
- PCI Express Base Specification 5.0 r1.0, §6.2, printed/PDF pp.505–533。
- PCI Express Base Specification 5.0 r1.0, §7.8.4, printed/PDF pp.841–858。
- Cadence PCIe Core User Guide, §§4.5.1.1.2、4.5.2.3、4.5.6.3.1、4.5.6.4.1。
- Cadence PCIe Core User Guide, §4.6.2, printed pp.274–276 / PDF pp.363–365。
- Cadence PCIe Core User Guide, Appendix B, printed pp.1332–1337 / PDF pp.1421–1426。
