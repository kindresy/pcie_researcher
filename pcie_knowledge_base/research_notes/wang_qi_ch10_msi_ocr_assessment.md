# 王齐第10章 OCR评估：MSI/MSI-X的稳定原理与平台差异

日期：2026-08-23

## 1. 阅读范围

结合扫描PDF pp.275起的Chapter 10、已OCR的MSI Capability页，以及Chapter 15 PDF pp.429–452的Linux中断实例交叉评估。本章的PowerPC MPIC和x86 FSB Interrupt Message属于2010年平台实现；协议结论以Base Spec 5.0和当前Cadence guide为准。

## 2. 教材讲对了的主线

- MSI/MSI-X在PCIe Link上是向Message Address写Message Data的Memory Write TLP，不是Message TLP；
- INTx保留level-like semantics，PCIe-to-PCI bridge用Assert/Deassert_INTx Message传递状态，不能把legacy level简单转成一次MSI edge；
- MSI Capability可有32/64-bit address layout与可选per-vector mask，MSI最多32 vectors；
- MSI-X把每个vector独立address/data/mask放在BAR中的Table，PBA记录masked pending，可扩展到2048 vectors；
- Message Address/Data的真实编码由host interrupt architecture决定，Endpoint不应猜测或写死。

这些已进入 [INTx、MSI、MSI-X与Cadence](../interrupts/intx_msi_msix_and_cadence.md)。

## 3. 需要校正的两个细节

### 3.1 一个Function可以同时实现MSI和MSI-X Capability

教材说“通常仅包含一种结构”，作为当年实现观察可以理解，却不是normative restriction。Base Spec 5.0 §7.7明确允许一个Function同时具有MSI和MSI-X Capability，只是software同一时刻只能enable一种。

### 3.2 “PCIe设备必须支持MSI/MSI-X”有条件

准确说法是：**能够生成interrupt的PCIe device Function**必须实现MSI、MSI-X或两者；完全不请求interrupt resource的Function不能被这句话强行推导出MSI Capability。如果PCIe Endpoint实现MSI，现代Spec还要求支持64-bit Message Address layout。

## 4. 平台实现不要泛化

教材的PowerPC MPIC和x86 FSB Interrupt Message很适合说明“同一笔MSI MWr到了RC后，会由不同平台用不同方式解码”。但现代x86/ARM64还可以有interrupt remapping/IOMMU、hierarchical irq-domain、ITS等实现。

当前Cadence RP证据链只能追到：收到MSI Memory Write→通过core AXI Master交给SoC。后面如何命中MSI controller/remapper再进CPU，必须要目标SoC TRM/kernel source，不能用教材的MPIC/FSB图代替。

## 5. Linux版本边界

Chapter 15的`pci_enable_msi()`/`pci_enable_msix()`和Linux 2.6.31内部调用链只是历史证据。当前驱动使用`pci_alloc_irq_vectors()`/`pci_irq_vector()`等统一API，并让PCI/MSI core编程capability/table与平台irq-domain。本地只有Ubuntu 6.8 public headers，所以精确内部调用链仍保留为资料边界。

## 6. Source Reference

- 王齐，《PCI Express体系结构导读》，Ch.10 printed pp.263–278 / 约PDF pp.275–290；Ch.15 printed pp.417–440 / PDF pp.429–452。
- PCI Express Base Specification 5.0 r1.0：§6.1 Interrupts；§7.7.1 MSI；§7.7.2 MSI-X。
- Cadence PCIe Core User Guide：§4.6 Interrupt Interface；§4.5.6.6.6–.7 outbound MSI/MSI-X。
