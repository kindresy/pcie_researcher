# MindShare Chapter 5–10 OCR 深读与现行规范校正

日期：2026-08-23

## 1. 范围和结论先行

来源：MindShare *PCI Express System Architecture*，Second Printing，2003。原 PDF 字体映射损坏，本轮对 PDF pp.267–449 逐页渲染并 OCR，覆盖：

- Chapter 5 ACK/NAK Protocol；
- Chapter 6 QoS / Traffic Class / Virtual Channel / Arbitration；
- Chapter 7 Flow Control；
- Chapter 8 Transaction Ordering；
- Chapter 9 Interrupts；
- Chapter 10 Error Detection and Handling。

这六章的“为什么”讲得相当好，尤其适合建立直觉；但它描述的是 PCIe 1.x 时代。知识库吸收它的因果叙事，不照搬 8b/10b framing、早期 credit 宽度、旧式平台中断路由和能力边界。现代结论统一回查 Base Specification 5.0 r1.0 与当前 Cadence Gen4 配置。

## 2. Chapter 5：ACK/NAK 是逐跳可靠投递，不是事务响应

书中把发送端和接收端的概念状态完整拆开：

```text
发送端                         接收端
NEXT_TRANSMIT_SEQ              NEXT_RCV_SEQ
Retry/Replay Buffer            LCRC + Sequence check
REPLAY_TIMER / REPLAY_NUM      AckNak latency timer
        ───── TLP ─────►
        ◄── Ack/Nak DLLP ──
```

至今仍成立的要点：

1. 好 TLP 可由一条累计 Ack 一并确认，不必一包一 Ack；
2. Nak 或 replay timer timeout 触发未确认 TLP 重放；
3. replay 期间先阻止新的 Transaction Layer TLP 混入；
4. Ack/Nak 仍必须在 replay 期间处理，较新的 Ack 可以覆盖较早的 Nak；
5. 2-bit `REPLAY_NUM` rollover 触发 Link retrain。

Base Spec 5.0 §§3.6.2–3.6.3 仍保留这些规则。一个容易忽略的细节是：**收到 Nak 本身不是 reported error**；协议本来就是用它修复偶发传输错误。Replay Timer Timeout、REPLAY_NUM Rollover、Bad TLP 等才进入错误状态统计。

本章的 cut-through/nullified TLP 用 `EDB` 和 inverted LCRC 描述，属于 8b/10b framing 语境。Gen3+ 的线路编码和 framing 不能照抄这些 symbol，但“Switch 已经开始转发、尾部才发现坏包，因此必须让下游丢弃，并由上游 Link replay”这个硬件问题仍然真实。

对应正文已补强：[Data Link ACK/Replay/Flow Control](../link/data_link_ack_replay_and_flow_control.md)。

## 3. Chapter 6：TC 是端到端标签，VC 是每一跳的资源选择

书中最有价值的一句话是：TC 随 TLP 穿过整个 fabric 保持不变，VC 却可以每经过一条 Link 重新映射。这避免把两者误当成同一个字段。

```text
TLP.TC = 3（端到端不变）

EP ── Link A: TC3→VC1 ── Switch ── Link B: TC3→VC2 ── RP
                         每一跳独立 buffer/credit/arbitration
```

每个启用的 VC 有独立 receive buffer 和 Flow Control credit；不同 VC 之间没有 ordering guarantee。VC arbitration 在同一 Port 的多个 VC 之间选谁发，Port arbitration 则解决多个 ingress 都想争同一 egress 的问题。全路径的软件配置必须一致，否则 QoS 不是“稍微不准”，而是 TC 在某一跳根本没有可用映射。

年代边界：教材强调 isochronous service interval 和摄像机/磁带机案例，这是早期 PCIe 对 QoS 的主要宣传语境，不代表今天的 OS 会自动替普通设备建立带宽保证。Base Spec 5.0 §2.5 仍定义 TC/VC，但 servicing policy 是平台软件/实现问题。当前 Cadence instance 的 configuration-specific 表没有选中额外 VC Capability，Local Error Status 还明确提供“访问未实现 VC Capability”的错误位；因此本项目不能把通用架构图中的 VC/PTM 方框当成已实例化功能。

详见 [TC/VC/QoS/PTM/ACS](../advanced/tc_vc_qos_ptm_and_acs.md)。

## 4. Chapter 7：Credit 是累计账本，不是一把 token

本章用 conceptual counters 解释发送判定：远端累计广告的 `Credit Limit`，减去本端累计 `Credits Consumed`，必须足以容纳待发 TLP。接收端 buffer 被上层取走后，推进累计 allocated credit，再用 UpdateFC DLLP 告诉对端。

六类资源依然是最实用的 Debug 心智模型：

| 类别 | Header | Data |
|---|---|---|
| Posted | PH | PD |
| Non-Posted | NPH | NPD |
| Completion | CplH | CplD |

VC0 的 FC initialization 必须由硬件完成，否则 Configuration TLP 自己都过不去；额外 VC 则要在软件启用后分别初始化。credit 为零导致的是该类 TLP不能跨当前 Link，不等于 LTSSM 一定掉出 L0。

必须校正的旧数值是：书中按早期字段宽度给出的 header/data 最大 credit 不能当作 Gen4 实际 buffer 上限。PCIe 5.0 §3.4.2引入 Scaled Flow Control；16.0 GT/s 及以上 Port 必须支持它。infinite advertisement 和 scaled field 还带来更细的初始化/更新规则。

Cadence Completion 接收路径又提醒了一层：Link credit 只保护“对端宣告的接收能力”，不自动解决 controller 内 Stage-1 FIFO 到 AXI reorder RAM 的局部背压。见 [Cadence Completion Buffer](../controller/cadence_completion_buffers_and_backpressure.md)。

## 5. Chapter 8：Producer/Consumer 是 Ordering 最好的入门故事

书中的经典场景仍然值得保留：Producer 先把 data 写到内存，再写 flag/doorbell；Consumer 看见 flag 后才读 data。Ordering rule 的任务，是让 doorbell 不要把它指向的数据甩在身后。

但“strong ordering vs relaxed ordering”只能当入口，不能当完整规则。PCIe 5.0 §2.4 使用 passing table，并加入 ID-Based Ordering、PASID 参与的 ordering domain 等现代条件。仍需牢记：

- ordering rules 按同一 TC讨论；不同 TC没有 ordering guarantee；
- RO 只在发起者知道两笔事务没有依赖时才可设置；
- No Snoop 是 cache coherency 属性，不改变 ordering；
- 某些 Completion 必须越过 Posted/NP traffic 是为了避免 deadlock，不是实现偷偷乱序。

正文：[MPS/MRRS/Ordering](../transactions/mps_mrrs_ordering_and_performance.md)。

## 6. Chapter 9：MSI 是 Memory Write，顺序选择也是协议的一部分

本章把 MSI 的 address/data 编程和触发过程讲得很具体：软件先在 Capability 中分配 address/data，设备事件发生后自行发一笔 DWORD Memory Write。这个模型至今有效。

更值得保留的是“data write 与 MSI 使用不同 TC/VC”的反例：如果 MSI 走高优先级 VC，可能先于真正的数据到达 CPU，ISR 醒来却读到旧内容。因此数据和通知必须选择能维持所需 ordering 的属性/TC，并同时满足 CPU/cache/DMA API 的同步规则。MSI 并不会因为名字里有 Interrupt 就免疫普通 Memory Write 的 ordering、Flow Control 和 integrity 规则；Base Spec 5.0 §2.2.8 直接说 MSI/MSI-X 与普通 MWr 在这些方面不可区分。

旧平台里 IOAPIC、固定 IRQ 和 chipset 的具体拓扑只作历史背景。现代 Linux 的 irq domain、interrupt remapping 以及平台 MSI controller 不能从这本 2003 年教材推导，且 workspace 当前没有相应内核源码/SoC TRM。正文保持证据边界：[INTx/MSI/MSI-X](../interrupts/intx_msi_msix_and_cadence.md)。

## 7. Chapter 10：先找“谁最先发现”，再看“如何上报”

书中按层划分错误源的思路仍很适合 Debug：

```text
Physical：receiver/framing/encoding
Data Link：Bad TLP、Bad DLLP、Replay Timeout/Rollover
Transaction：Malformed、UR、Completion Timeout、Poison/Completer Abort
        ↓
本地 status/mask/severity
        ↓
ERR_COR / ERR_NONFATAL / ERR_FATAL Message
        ↓
Root Port / AER / 软件恢复
```

关键原则是错误的“首次发现者”与最终日志打印者可能隔着多跳。Correctable 也不等于可以无视：大量 replay 最终会吞掉带宽，甚至演化为 Link retrain/down。

年代校正：本章基于早期 PCIe error model，现代 AER 增加了更完整的 status/mask/severity、Header Log、Root Error Command/Status，PCIe 5.0还要结合 DPC、Advisory Non-Fatal、Surprise Down 等机制阅读。旧书中把 poisoned TLP与兼容 `Parity Error Response/SERR#` 的关系作为主叙事，也不能替代现代 AER 路径。

正文：[Error/AER/Cadence](../errors/error_model_aer_and_cadence.md) 与 [DPC](../errors/dpc_containment_and_recovery.md)。

## 8. 覆盖判断

| Chapter | 有效阅读 | 吸收到知识库 | 仍需额外资料 |
|---|---|---|---|
| 5 ACK/NAK | 完成 | Data Link文档已补累计Ack/replay细节 | Gen3+ framing以Base Spec/Cadence为准 |
| 6 QoS/TC/VC | 完成 | TC/VC专题 | 当前SoC policy无资料 |
| 7 Flow Control | 完成 | Data Link、MPS/MRRS、Completion buffer | controller内部计数器可见性继续寄存器审计 |
| 8 Ordering | 完成 | Ordering专题 | 目标CPU/AXI一致性规则缺TRM |
| 9 Interrupts | 完成 | Interrupt专题及DMA通知顺序 | 目标irq remapper/kernel tree缺失 |
| 10 Errors | 完成 | Error/AER/DPC及runbooks | 目标平台IRQ/recovery调用链缺失 |

## 9. Source Reference

- MindShare *PCI Express System Architecture*：Chapters 5–10，约 PDF pp.267–449；OCR cache `/tmp/pcie1_arch_scan_core2/page-0267..0449.txt`。
- PCI Express Base Specification Revision 5.0 Version 1.0：§§2.2.6、2.2.8、2.4–2.6、3.4、3.6、6.1、6.2、7.7、7.9。
- Cadence PCIe Core User Guide：§§4.2–4.6、11.4 与 Appendix H configuration-specific capability tables。
