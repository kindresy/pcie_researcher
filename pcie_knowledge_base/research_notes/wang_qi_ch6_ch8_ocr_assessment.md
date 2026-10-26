# 王齐第 6–8 章 OCR 评估：TLP、可靠传输、LTSSM 与电源

日期：2026-08-23

逐页 OCR 阅读范围：

- Ch.6 Transaction Layer：printed pp.155–184 / PDF pp.167–196；
- Ch.7 Data Link / Physical Layer：printed pp.185–210 / PDF pp.197–222；
- Ch.8 Link Training / Power：printed pp.211–242 / PDF pp.223–254。

这三章正好组成一笔 PCIe transaction 的“纵切面”：上面是软件最关心的请求/完成，中间负责这一个 Link 的可靠交付，下面负责让一堆高速差分线先真正成为 Link。读懂这条纵线，比背三张 header 表更值钱。

## 1. Chapter 6：Transaction Layer

### 1.1 稳定、已吸收的主线

教材对以下概念的初学者解释仍然有效：

- 3DW/4DW header、Fmt/Type、TC、Attr、Length、Requester ID、Tag、Byte Enable；
- address routing、ID routing 与 Message implicit routing；
- Memory Write 为 Posted，Memory Read 用 Request + CplD；
- Configuration/IO Write 虽然携带数据，仍是 Non-Posted，需要 Completion；
- 一个 Memory Read Request 可以由多个 CplD 返回，MPS、MRRS、RCB 分别约束不同对象；
- AtomicOp 返回旧值，不是 legacy MRdLk/Unlock；
- TPH/Steering Tag 是 performance hint，不改变访问权限。

对应正文已经分别进入：

- [`transactions/memory_tlp_and_completion.md`](../transactions/memory_tlp_and_completion.md)；
- [`transactions/mps_mrrs_ordering_and_performance.md`](../transactions/mps_mrrs_ordering_and_performance.md)；
- [`transactions/message_atomic_and_tlp_prefix.md`](../transactions/message_atomic_and_tlp_prefix.md)；
- [`configuration/bdf_requester_id_and_hierarchy_routing.md`](../configuration/bdf_requester_id_and_hierarchy_routing.md)。

### 1.2 三个不能照抄的点

#### “除了 MWr 之外都 Non-Posted”过于粗糙

Message Request 也属于 Posted Request；AtomicOp 则是 Non-Posted。判断是否要 Completion 要看 transaction type，不要从“有没有 data payload”反推。最典型反例就是 MSI：它是带 data 的 Memory Write，成功路径没有 Completion。

#### AT 不是随便做 CPU↔PCI 地址转换的开关

AT 字段描述 ATS 语境下地址是 Untranslated、Translation Request 还是 Translated。普通 Function driver 不能因为系统有 IOMMU 就自行把 AT 改成 Translated；translated address 必须来自受信 Translation Agent/ATS protocol，并接受 invalidation。详见 [`advanced/iommu_ats_pasid_pri.md`](../advanced/iommu_ats_pasid_pri.md)。

#### No Snoop 的 FSB 优化例不能移植成通用 Linux 配方

教材建议对大 buffer 先 software cache flush 再置 No Snoop，这是特定早期 FSB/coherency 假设下的性能讨论。现代系统是否安全取决于 DMA mapping、memory attribute、coherency fabric 与平台 policy；一次 flush 不是“从此免维护”。这项边界已补入 MPS/ordering 正文。

### 1.3 MPS、MRRS、RCB 的年代校正

教材正确说明：

- MPS 限制带 data TLP，包括 MWr 与 CplD；
- MRRS 限制 Requester 单个 Memory Read Request 的请求量；
- RCB 约束 Completion boundary，不能拿来当 payload size。

但 MPS 不是 LTSSM 自动“协商”的字段，而是 software 根据 hierarchy 能力与 policy 编程。书中“多数 x86 RC 是 128 B、PowerPC 多为 256 B”只是当年平台观察，不是协议默认，更不能拿来预测当前 Cadence + SoC。

### 1.4 AtomicOp/TPH 与当前 Cadence build

书写作时 AtomicOp/TPH 都很新，甚至写道当时尚无支持 TPH/ST 的设备。今天这句话只剩历史价值：Cadence Appendix H 的 PF/VF/RP capability 链包含 TPH Requester surface；但实际 client 是否生成有意义的 hint 仍需 SoC/RTL证据。

反方向也一样：PCIe 5.0 定义 AtomicOps，不等于当前 IP build 支持。Cadence 手册明确把这份 configuration 的 AtomicOp 标成 unsupported，不能因为规范有 Type code 就让 driver 开始发包。

## 2. Chapter 7：Data Link 与早期 Physical Layer

### 2.1 Data Link 部分仍然非常能打

本章把 replay path 讲成一条清楚的因果链：

```text
Transaction Layer TLP
 → sequence number + LCRC
 → replay buffer保留副本
 → 对端校验
 → ACK释放旧副本 / NAK触发重放
```

它还解释 ACK/NAK 是逐 Link 的可靠性，不是端到端 Completion；DLLP 也服务于 flow-control 与 power-management。对应现代规范校正和 Cadence replay buffer 映射见 [`link/data_link_ack_replay_and_flow_control.md`](../link/data_link_ack_replay_and_flow_control.md)。

老书使用 `DL_Inactive/DL_Init/DL_Active`、`DL_Down/DL_Up` 讲上下层握手，适合作为状态边界直觉；具体状态、Feature Exchange、Scaled Flow Control 和 timeout 必须以 PCIe 5.0 Chapter 3 为准。

### 2.2 Physical Layer 只覆盖 Gen1/Gen2

书中的 symbol、K-code、comma、8b/10b、2.5/5 GT/s framing 是早期链路的有效知识，但不能外推到 Gen3/4/5：后者使用 128b/130b、ordered-set/block framing，并增加 equalization 等训练过程。

因此本章 Physical 部分状态记为 **E/D+M**：Gen1/2 机制进入年代分层说明；现代训练以 Base Spec Chapter 4、Cadence PIPE/equalization interface 为主。缺 PHY Databook 时，不编造 CTLE/DFE/SerDes 内部算法。

## 3. Chapter 8：LTSSM 与 Power

### 3.1 LTSSM 主骨架稳定

Detect → Polling → Configuration → L0，以及 L0 → Recovery → L0 的因果主线仍然成立。教材逐状态解释了：

- Receiver Detection；
- TS1/TS2、Link Number/Lane Number；
- width/lane negotiation 与 lane reversal；
- speed change、retrain 与 Recovery；
- Disabled、Loopback、Hot Reset；
- Electrical Idle 与低功耗退出。

这部分已经与 Cadence LTSSM encoding/history/debug bus 交叉进入 [`link/link_lane_ltssm_and_training.md`](../link/link_lane_ltssm_and_training.md) 和 [`debug/link_not_up.md`](../debug/link_not_up.md)。

### 3.2 缺失的是 Gen3+ Equalization，不是“小补丁”

书中 Recovery 只围绕 2.5/5 GT/s、bit/symbol lock 与 speed change 展开；它早于 8/16/32 GT/s 的 phase/preset/coefficient equalization。调 Gen4 Link 卡在 Recovery.Equalization 时，不能拿这章状态图硬套，否则会像拿 UART 波特率表修 SerDes 眼图——态度认真，工具不对。

当前知识库已覆盖 equalization 的 controller/PIPE 边界，但 PHY 模拟内部细节继续标为资料阻塞。

### 3.3 D-state 与 Link state 必须分开

教材分别讨论 Function 的 D0/D1/D2/D3hot/D3cold 与 Link 的 L0/L0s/L1/L2/L3 Ready，这条分法是对的：

- D-state 描述 Function 电源/上下文；
- Link state 描述一段物理 Link；
- ASPM 是 Link policy，PCI PM 是 software-visible Function power management；
- 二者会互相约束，但不一一等价。

书中的 Beacon、L2、ASPM 支持性与精确迁移条件带有 PCIe 2.x 年代背景。现代 L1 Substates、CLKREQ#、LTR、OBFF、DPC/Hot Plug policy 等应读当前 power 专题，而不是把 2010 状态表当平台 firmware contract。

## 4. Coverage 结论

| 范围 | 状态 | 结论去向 |
|---|---|---|
| Ch.6 TLP format/routing/request/completion | D+M | transactions、configuration |
| Ch.6 Atomic/TPH | D+M | advanced transaction；Cadence build差异明确 |
| Ch.6 MPS/MRRS/RCB | D+M | performance文档；修正“自动协商”表述 |
| Ch.7 Data Link/ACK/NAK/replay | D+M | Data Link专题与Cadence buffer |
| Ch.7 8b/10b Physical | E/D+M | 仅作为Gen1/2层，禁止外推Gen3+ |
| Ch.8 LTSSM core states | D+M | Link与debug文档 |
| Ch.8 ASPM/PCI PM | D+M | power专题；加入L1SS/平台年代校正 |

## 5. Source Reference

- 王齐，《PCI Express体系结构导读》，Chapters 6–8，2010 年第一版，扫描 PDF pp.167–254。
- PCI Express Base Specification 5.0 r1.0, Chapters 2–5。
- Cadence *PCIe Core User Guide*, Chapters 4、8、Appendices C/E。
