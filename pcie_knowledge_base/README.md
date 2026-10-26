# PCIe：从“这根线是什么”到能写驱动、查链路

这是一套面向芯片底层驱动工程师的 PCIe 学习知识库。默认读者会写 C，认识 MMIO、IRQ、DMA、Cache 和 AXI，但还没有把 PCIe 的术语、协议、控制器硬件与 Linux 串成一条完整因果链。

这里不准备把 1299 页的规范压成一份“十分钟速通”。PCIe 也不太配合这种愿望——它把总线、网络、分层协议、资源管理和操作系统揉在了一起。知识库采用下面的主线：

> 先建立直觉，再看协议定义；先弄清一次动作为什么发生，再看寄存器和 Linux API；最后把一行 `writel()` 一路追到链路上的 TLP。

## 建议从哪里开始

1. 先读 [学习路线](LEARNING_PATH.md)，知道哪些概念是后面内容的地基。
2. 需要按问题查找文章时，使用 [正文文章索引](ARTICLE_INDEX.md)。
3. 阅读 [PCIe 世界观：拓扑、角色与三层协议](fundamentals/overview_topology_and_layers.md)。
4. 遇到陌生缩写时随时查 [术语表](GLOSSARY.md)，先确认术语含义，再继续阅读调用链或寄存器说明。
5. 进入配置、枚举、BAR 与地址翻译，理解 Linux 为什么能“看到”设备。
6. 再学习 TLP、Completion、DMA、MSI/MSI-X、LTSSM、错误与电源管理。
7. 最后进入 `linux/` 和 `controller/`，把软件调用链映射到真实硬件模块。
8. 若目标是 X6000，进入 [X6000 综合 PCIe 架构](code_studies/x6000_integrated_pcie_architecture.md)，
   再按其中的七阶段源码路线阅读六个仓库。

## 目录各自负责什么

| 目录 | 主要问题 |
|---|---|
| `fundamentals/` | PCIe 是什么，RC/RP/EP、拓扑、BDF、分层、Link/Lane |
| `configuration/` | 配置空间、Capability、ECAM、枚举、Expansion ROM和资源分配 |
| `addressing/` | BAR、CPU/PCI/DMA 地址、inbound/outbound translation |
| `transactions/` | TLP、读写、Completion、Tag、Byte Enable、Ordering、Flow Control |
| `link/` | PHY、LTSSM、链路训练、速率/宽度协商与恢复 |
| `interrupts/` | INTx、MSI、MSI-X 及 Linux 中断框架 |
| `dma/` | Bus Master、DMA API、IOMMU、Cache 一致性、SG |
| `errors/` | AER、UR、CA、CTO、Poison、Link Down 与恢复 |
| `power/` | D-state、ASPM、L1SS、PME、DPA、slot功耗预算与Hot Plug |
| `linux/` | PCI core、枚举、驱动生命周期和关键 API 调用链 |
| `controller/` | Cadence PCIe 核心架构、寄存器、AXI/PIPE、RP/EP 实现 |
| `debug/` | 从现象到分层定位、寄存器、日志、TLP/波形的调试手册，以及protocol analyzer的双向/trigger方法 |
| `advanced/` | ATS、PASID、PRI、SR-IOV、ARI、NTB 等进阶特性 |
| `examples/` | 带具体 BDF、地址和 TLP 字段的端到端事务例子 |
| `code_studies/` | X6000 六仓源码、BootROM分支、启动交接、地址图与综合架构 |

目录已经按完整知识体系预留，但空目录不算完成；实际覆盖状态以 [PROGRESS.md](PROGRESS.md) 为准。

想先看整套机制怎样串起来，可以直接读 [Endpoint 完整工作流程：从 PERST# 到 DMA 完成和 MSI-X](examples/endpoint_from_power_on_to_dma_and_msix.md)，再回到各专题补原理。

## Spec、Databook 与 Linux 怎么配合读

三类资料分别回答不同问题：

```text
PCIe Base Spec：线上必须遵守什么规则？
        ↓
Cadence Core User Guide：这个控制器怎样实现规则？暴露哪些接口和寄存器？
        ↓
Linux：系统何时、为何、通过什么调用链配置并使用这些能力？
```

不要拿 Databook 代替 Spec：控制器手册会省略它认为读者已经知道的标准规则。也不要拿 Spec 代替 Databook：规范不会告诉你某颗核心的 AXI 接口、复位顺序或本地管理寄存器。Linux 则是第三块拼图，它解释“谁在什么时候动了这些硬件”。三者的逐项对应维护在 [SPEC_DATABOOK_MAPPING.md](SPEC_DATABOOK_MAPPING.md)。

知识库最初只有通用资料，现已补入 X6000 Linux 5.10.4、Host edge-driver、BootROM、
PCS、ATF 和 edge10_tool/HAL。通用 Linux 章节仍保留“跨版本 API 语义”的视角；目标
实现则在 `code_studies/` 中按固定 commit 追到 Cadence platform/EPC/EPF、BAR/APT、
uDMA、MSI-X 与 AER。仍缺的 PHY/TRM/部署脚本和实机 dump 统一列在
[OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)，不会拿推测填空。

## 写作与证据约定

- 重要概念第一次出现时按“直觉 → 场景 → 规范 → 硬件 → 软件 → Linux”展开。
- 每个关键结论尽量给出本地资料路径及 section、printed page 或 PDF page。
- **Confirmed** 表示被权威资料直接支持；**Inferred** 表示有明确推理链；**Conflict** 表示资料冲突；**Unknown** 表示证据不足。
- printed page 是文档印刷页码，PDF page 是阅读器中的页序号；两者不同就同时写。
- 配图只在真正能降低理解成本时出现。原始资料截图放在 `assets/source_figures/`，文件名包含来源 PDF 页；图注必须说明来源。
- 行文采用自然、成熟的工程交流口吻。必要术语保留准确含义，并在首次出现时解释；能够直接说明的内容不使用夸张比喻或刻意制造笑点。技术结论同时说明原理、适用条件和证据边界。

## 当前资料边界

本知识库以目录内的 PCIe 5.0 Base Spec、Cadence Gen4 x16 Dual-Mode Core User Guide、
两本体系结构教材和培训材料为规范/控制器基础，并加入六套 X6000 工程代码。文件资料
盘点见 [SOURCE_INDEX.md](SOURCE_INDEX.md)，代码版本见
[X6000代码仓来源索引](code_studies/SOURCE_REPOSITORY_INDEX.md)。仍未提供 CEM、独立
PHY Databook 和完整 SoC TRM；这些边界不会被源码里的寄存器宏“假装解决”。

## X6000 快速入口

- [综合 PCIe 架构与源码阅读路线](code_studies/x6000_integrated_pcie_architecture.md)
- [BootROM→PCS→BL2→Linux 交接与所有权](code_studies/x6000_boot_handoff_and_ownership.md)
- [统一地址地图](code_studies/x6000_address_map.md)
- [六仓研究进度](X6000_RESEARCH_PROGRESS.md)
