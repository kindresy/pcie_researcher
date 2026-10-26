# MindShare Chapter 2–4 OCR 深读与年代校正

日期：2026-08-23

## 1. 阅读范围与方法

来源：`PCI.Express.System.Architecture.pdf`，Ravi Budruk / Don Anderson / Tom Shanley，Second Printing，2003。

原 PDF 的字体映射损坏，普通 `pdftotext` 的正文大多成为 mojibake。本轮对 PDF pp.119–260 附近逐页渲染并 OCR，深入评估：

- Chapter 2 Architecture Overview；
- Chapter 3 Address Spaces & Transaction Routing；
- Chapter 4 Packet-Based Transactions 的 TLP/Request/Completion 主体。

OCR cache 位于 `/tmp/pcie1_arch_scan100/page-*.txt`，只作为检索辅助；关键结论仍以页面图像和 Base Spec 5.0 交叉验证。

## 2. 最值得保留的教学直觉

### 2.1 Request 与 Completion 是两趟独立的车

Chapter 2 用 CPU→EP Memory Read、EP→System Memory Read、CPU→EP Memory Write 展开 split transaction：Request 依据 address 下行，CplD 复制 Requester ID/Tag 后按 ID 回到原 Requester。这个叙事非常适合初学者，也与 Base Spec 5.0 主线一致。

特别有用的是它同时展示 Endpoint 既可当 Completer，也可主动发 MRd 做 Requester。这样能直接拆掉“EP只能被Host访问”的误解。

### 2.2 Data Link ACK 与事务 Completion 不在同一层

教材把一次 MRd/CplD 各自经过 Sequence Number/LCRC、ACK/NAK/Replay 的过程完整走了两遍：MRd TLP 在每一跳得到链路 ACK，后来返回的 CplD 自己又是另一条要被 ACK 的 TLP。

直觉可以写成：

```text
MRd TLP:   Requester ──逐跳 ACK──► Completer
CplD TLP:  Requester ◄──逐跳 ACK── Completer

Completion 是事务响应；ACK 是每条 Link 的可靠投递确认。
```

这与现有 [Memory TLP 与 Completion](../transactions/memory_tlp_and_completion.md) 和 [Data Link](../link/data_link_ack_replay_and_flow_control.md) 一致。

### 2.3 路由的三种决定

Chapter 3 把 multi-port component 收到 TLP 后的动作概括为：

1. 自己就是目标，consume；
2. 目标在其他 Port，forward；
3. 既不是自己也不在可达窗口，reject/UR。

它又把 TLP routing 分为 address、ID、implicit 三类，并反复强调 DLLP/Ordered Set 只在本 Link 消费，不跨 Switch forward。这种“先问包能否跨 Hop，再问用什么字段选 egress”的顺序很清楚。

### 2.4 Switch 的两步检查

教材对 Switch address routing 的描述很适合拿来 Debug：

```text
先检查 address 是否命中 Switch 自己的 BAR/internal target
  否 → 检查每个 downstream virtual bridge 的 Memory/Prefetchable/I/O Base-Limit
```

ID routing 类似：先看是不是自己的 BDF，再用 Secondary–Subordinate Bus Number range 选 downstream Port。它解释了为什么只给 EP 写 BAR 不够，沿途 bridge window/bus aperture 也必须覆盖。

### 2.5 Transaction Descriptor 不是某一段连续 bit

Chapter 4 把 Requester ID+Tag、TC 和 Attr 作为共同描述 transaction 的字段集合。字段不一定在 header 中相邻，却共同决定匹配、ordering 和服务类别。这个表达有助于读者理解 Cadence 为什么要为 outstanding request 保存一整组 context，而不是只存一个 Tag。

## 3. 与当前知识库交叉验证后的修正

### 3.1 2003 年的 Physical Layer 不能套在 Gen4 上

书中大量使用 K-code、Start/End symbol、8b/10b encoder 的统一描述，只适用于 Gen1/Gen2。Gen3/4 使用 128b/130b block encoding、sync header、ordered-set block 等不同 framing。分层思想仍可用，线路格式必须以 Base Spec 5.0 Chapter 4 和 Cadence Gen4 实现为准。

教材使用 Physical Layer Packet（PLP）概括 Ordered Set 等物理层流量，但这不是现代驱动资料中最常用的术语。知识库继续直接使用 Ordered Set、TS1/TS2、EIOS/EIEOS/SKP 等具体名称，避免引入没有必要的中间术语。

### 3.2 Type 0 write 捕获 Bus/Device Number 今天仍是规范要求

教材说 Function 从完成的 Type 0 Configuration Write 中捕获自己的 Bus/Device Number。Base Spec 5.0 §2.2.6.2仍明确要求：Function必须从**每一笔**由它完成的Type 0 Config Write重新捕获这些编号，用于后续Requester ID；§2.2.9也对Completer ID作同类要求。首次Config Write之前，Function不得发需要Completion的Non-Posted Request，因为还没有可供Completion正确返回的有效Requester ID。

现代差异不是“这个机制过时了”，而是ARI设备只需捕获Bus Number，Device Number不再按传统字段解释；SR-IOV跨Bus VF还有额外规则。现有[BDF专题](../configuration/bdf_requester_id_and_hierarchy_routing.md)的表述得到此次复核确认。

### 3.3 Ordering 规则不能继续称为“PCI strong ordering”二选一

教材用 PCI/PCI-X strong vs Relaxed Ordering 解释早期规则，很有历史价值，但 Base Spec 5.0 的 passing matrix、同 TC边界、IDO、No Snoop 与不同 TLP类型更精细。正文以 §2.4 为准；特别是 No Snoop 不改变 ordering，RO 也不是“所有包随便超车”。

### 3.4 早期 header/能力宽度不是今天的全貌

教材的 Requester ID+8-bit Tag、传统 BDF 与基础 capability 是 PCIe 1.x 世界的主视图。现代资料还需考虑 Extended Tag、10-bit Tag、ARI、PASID、TLP Prefix、ATS/PRI/SR-IOV 等。它讲的基本 header 仍正确，但不能据此推断当前最大 outstanding identity space。

### 3.5 Root Complex Requester ID 是实现/路径语义

教材例子简单称 Memory Read request 带“root complex requester ID”。教学上足够，但驱动分析不能随手把 CPU MMIO Requester ID 固定写成 `00:00.0`。RC-generated request 的具体 identity 与 port/implementation相关；现有 worked example 保留这个边界。

## 4. 对现有知识库的影响

- 现有 topology/layers、TLP/Completion、BDF/routing 和 BAR/bridge window 主线与教材一致；
- 新增 [一个 Endpoint 的一天](../examples/endpoint_from_power_on_to_dma_and_msix.md)，吸收其逐跳讲故事方式，但使用 PCIe 5.0/Cadence 当前证据；
- 不引入教材的 PLP 作为核心 glossary 项；
- 保留并强化Type 0 Config Write捕获BDF的规范机制，同时补ARI/SR-IOV例外；Ordering则不采用教材的早期绝对表述；
- 后续 OCR Chapter 20/21 可重点复核 configuration mechanism/enumeration 的具体算法，Chapter 5–8 可与 ACK/FC/ordering 文档交叉。

## 5. Source Reference

- MindShare *PCI Express System Architecture*：
  - Chapter 2，printed pp.55–104，约 PDF pp.115–162；
  - Chapter 3，printed pp.105–154，约 PDF pp.163–212；
  - Chapter 4，printed p.155 起，约 PDF p.213 起；
  - OCR page locators：PDF pp.119–260。
- PCI Express Base Specification Revision 5.0 Version 1.0：§§1.5、2.2–2.5、Chapters 3–4。
- Cadence PCIe Core User Guide：§§4.1–4.5，作为层次与 requester/completer buffer 的实现交叉证据。
