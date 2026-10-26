# 王齐教材第 14/15 章 OCR 评估：哪些仍值得学，哪些已经退休

## 范围

已对扫描 PDF pp.374–455 批量 OCR，并逐页评估 Chapter 14（Linux PCI 初始化）与 Chapter 15（Linux PCI 中断）。printed page 与 PDF page 约相差 12。

## 仍然有效的架构洞见

- enumeration 与 driver registration 是不同阶段；
- host bridge 必须描述 bus range、config access、CPU↔PCI `ranges`；
- controller inbound/outbound windows 是非 x86 SoC 的核心；
- `pci_enable_device`、BME、resource allocation 各有不同职责；
- INTx 会经过 bridge swizzle，系统软件需要 routing information；
- MSI/MSI-X 最终由 architecture/platform compose address/data；Linux IRQ number 不等于硬件 vector/data；
- MSI-X 可把 RX/TX/other 分给不同 handler，减少共享状态判别。

## 明确过时的实现细节

- 代码基于 Linux 2.6.30/2.6.31，现代 MSI API、IRQ domains、host bridge/device model 已重构；
- `pci_enable_msi()`、`pci_enable_msix()` 等旧 API 已被统一 vector-allocation API 取代；
- `pdev->irq` 连续区间、多 MSI 数量限制等不能外推到现代 kernel；
- x86 ACPI/IO-APIC 和 PowerPC `hose` 调用链只代表当时特定 architecture；
- 书中使用的 PCIe 版本早于当前 Base Spec 5.0，advanced capability 行为必须以当前 Spec 为准。

## 可追溯 locator

- Chapter 14：printed pp.365–416 / PDF pp.377–428。
- `pci_enable_device` historical chain：printed p.419 / PDF p.431。
- PowerPC DTS `ranges`、ATMU 与 MSI inbound example：printed pp.412–416 / PDF pp.424–428。
- Chapter 15：printed pp.417–440 / PDF pp.429–452。
- MSI/MSI-X 2.6.31 code：printed pp.432–440 / PDF pp.444–452。

OCR 原始文本位于运行期 `/tmp/pcie1_cn_ch14_txt/`，不是知识库永久依赖；关键结论已转写并给出 PDF locator。

