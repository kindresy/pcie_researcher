# 王齐第 1–3 章 OCR 评估：PCI 祖先留下了什么，又退休了什么

日期：2026-08-23

逐页 OCR 阅读范围：printed pp.3–98 / PDF pp.15–110；另评估本篇导言 PDF pp.13–14。

这三章几乎都在讲 PCI，不是 PCIe。把它们全删掉会很省页数，却会丢掉 PCIe 最特别的一半：**传输硬件推倒重来，Configuration/BAR/BDF/driver 软件模型却刻意继承。** 正确读法是把“祖传软件契约”和“已经退休的并行总线时序”分开。

## 1. Chapter 1：shared PCI bus 是 PCIe 复杂度的反衬

### 1.1 值得保留的祖先概念

- Host Bridge 分隔 CPU/memory domain 与 PCI bus domain；
- BDF/Configuration Space/BAR 建立可发现、可分配的软件模型；
- device 可以成为 bus master，主动 DMA；
- Memory Write 可 Posted，read/non-posted transaction 需要结果；
- INTx 需要 routing/共享处理，MSI后来用写事务摆脱物理中断线；
- PCI-X Split Transaction 已出现 request 与 later response 分离的思路。

这些概念解释了 PCIe 为什么对 Linux 看起来“像 PCI”：不是巧合，而是兼容性目标。

### 1.2 已退休的部分

PCI 的 AD/CBE#/FRAME#/IRDY#/TRDY#、central arbitration、turnaround cycle、retry/disconnect/delayed transaction 是共享并行总线协议。PCIe 没有把这些信号偷偷塞进 SerDes；它使用 TLP、Link-local Flow Control、Tag 和 Completion。

因此：

- PCI Posted buffer 的故事能帮助理解“发起者何时可以继续”，但不能替代 PCIe ordering/Completion 规则；
- PCI-X Split Completion 是很好的历史台阶，但 PCIe Completion header/status/routing 必须重新学；
- PCI INTx swizzle/wire routing 只解释 legacy 背景，PCIe Link 上的 INTx 是 Assert/Deassert Message，现代 driver 首选 MSI/MSI-X。

主线已映射到 [`fundamentals/overview_topology_and_layers.md`](../fundamentals/overview_topology_and_layers.md)、[`transactions/memory_tlp_and_completion.md`](../transactions/memory_tlp_and_completion.md) 与 interrupt 文档。

## 2. Chapter 2：地址域与配置模型是真正的核心

### 2.1 地址不是因为数值相等就成了同一种地址

教材用 PowerPC Host Bridge windows 和 x86 chipset 对比：

```text
CPU/memory-domain address
  → host outbound decode/translation
  → PCI bus address（访问设备BAR）

PCI device DMA address
  → host inbound decode/translation
  → memory-domain/DRAM address
```

这与当前 Cadence core 的方向正好能接起来，但命名观察者不同：Host/SoC 通常把 CPU→PCI 叫 outbound；Cadence core 看到 Link→local 时自己是 AXI Master，local→Link 时 client 访问 core AXI Slave。方向必须写全观察者，单独一个 `inbound` 很容易让两位工程师同时说对、同时吵起来。

这一原则已进入 [`addressing/bar_and_cadence_address_translation.md`](../addressing/bar_and_cadence_address_translation.md) 与 DMA/IOMMU专题。

### 2.2 Configuration/Enumeration：外壳延续，运输工具变化

旧 PCI 的 Type 0/Type 1 Configuration Cycle、bridge Primary/Secondary/Subordinate Bus Number、Type 0/1 header、BAR sizing 和 bridge window 是 PCIe software model 的直接祖先。

但具体访问路径已经变化：PCI 的 IDSEL/并行信号不是 PCIe Configuration TLP；x86 CF8/CFC 也只是一种 legacy host mechanism。PCIe 系统常用 ECAM load/store，经 Host Controller 变成 Type 0/1 Configuration Request。

“Bus Number 由配置软件探索/分配，不是设备焊死”这一点仍完全值得保留。

### 2.3 Non-Transparent Bridge

Chapter 2.5 用 Intel 21555 说明两个 Host domain 不能共享一套透明 enumeration/地址 ownership，因此需要各自看到 Type 0 face、translation window 和 mailbox/doorbell。具体芯片寄存器已过时，问题模型已进入 [`advanced/non_transparent_bridge_and_multi_host.md`](../advanced/non_transparent_bridge_and_multi_host.md)。

## 3. Chapter 3：BAR、DMA 与 Cache

### 3.1 BAR sizing/resource allocation 的软件模型仍在

教材展示向 BAR 写全 1、读取硬件 mask 推导 aperture，再由 firmware/OS 分配地址并编程 bridge Base/Limit。这条算法的 PCIe版本已独立说明；实现时还要处理 64-bit BAR、decode disable、EA/Resizable BAR、bridge window 和 Host address translation。

### 3.2 Combining/Merging 为什么能解释 Prefetchable

PCI bridge 可以在不破坏语义时合并连续 write、合并同一 DW 的 byte write；但对 doorbell、FIFO、clear-on-read/status register，这种优化可能改变设备行为。

这正是 Prefetchable BAR 承诺的直觉来源：读不能有副作用，write combining/merge 不能破坏语义。`Prefetchable` 不是“给 CPU 随便 cache 的许可证”。对应结论见 [`addressing/bar_and_cadence_address_translation.md`](../addressing/bar_and_cadence_address_translation.md)。

### 3.3 DMA/cache coherence 是平台问题，不是 PCIe 自动包办

教材花大量篇幅讲 MESI/MOESI、snoop、可/不可 cache memory、未对齐 cache line 与 software prefetch。对驱动工程师最有价值的结论是：

- device DMA 与 CPU cache 可能观察到不同副本；
- coherent/non-coherent 取决于 SoC interconnect、Host Bridge、memory attribute 与 mapping；
- DMA direction 站在 device 视角；
- 未对齐 buffer 的首尾 cache line 不能粗暴 invalidate；
- Linux driver 应依赖 DMA API 的 map/sync/unmap ownership，不直接照抄 PowerPC/x86 cache instruction。

旧 PCI 的 MRL/MRM、bridge prefetch buffer、FSB snoop 与特定 CPU预取算法只作背景。PCIe MRd/MWr 的 MPS/MRRS/RCB/No Snoop 是另一套线上规则。

## 4. 三章给 PCIe 初学者的最佳阅读方式

推荐只带着三个问题读这部分：

1. PCIe 为了兼容，保留了哪些 software objects？
2. Host Bridge 两边为什么必须区分地址域？
3. device DMA 与 CPU cache/ordering 的责任由谁承担？

如果读到 FRAME# 时开始背波形，说明已经走偏；除非正在维护一块真 PCI 板，否则它只需要帮你理解 PCIe 为什么选择 packetized split transaction。

## 5. Coverage 结论

| 范围 | 状态 | 去向 |
|---|---|---|
| Ch.1 PCI signals/timing/arbitration | E | 历史比较，不映射成PCIe硬件行为 |
| Ch.1 Posted/Non-Posted/PCI-X split/INTx | D+M | overview、transaction、interrupt |
| Ch.2 address domains/Host Bridge | D+M | addressing、DMA、controller direction |
| Ch.2 config/header/enumeration/BAR | D+M | configuration、BAR |
| Ch.2 NTB | D+M | multi-host正文；21555寄存器只评估 |
| Ch.3 BAR/data path/DMA | D+M | BAR、worked example、DMA |
| Ch.3 cache/coherency/alignment | D+M | DMA API 与 No Snoop边界 |
| Ch.3 CPU/PCI prefetch details | E/M | 只吸收Prefetchable/side-effect直觉 |

## 6. Source Reference

- 王齐，《PCI Express体系结构导读》，Chapters 1–3，2010 年第一版，扫描 PDF pp.13–110。
- PCI Express Base Specification 5.0 r1.0, Chapter 1、§§2.2–2.3、Chapter 7。
- Cadence *PCIe Core User Guide*, §§4.5–4.6、Chapters 5、9–10。
