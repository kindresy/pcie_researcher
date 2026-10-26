# FINAL COVERAGE AUDIT

> 2026-08-27新增的X6000六仓代码已另行执行完整审计：
> [X6000六仓 PCIe Final Coverage Audit](research_notes/x6000_multirepo_final_coverage_audit.md)。
> 本文以下内容是此前规范/Databook/教材证据集的审计，两份报告共同构成当前最终覆盖状态。

审计日期：2026-08-23

## 1. 审计结论先说

当前可读输入资料的 PCIe 相关内容已经完成有效扫描、主题深读、交叉验证和知识落盘。知识库现有 77 篇 Markdown、超过 9100 行正文，以及 12 张从原始资料裁出的教学图。全库本地链接检查为 0 个断链；MPS、Flow Control、MSI TLP 类型、Cadence AXI 方向和旧资料年代边界的专项术语检查没有发现未修正错误。

本轮没有发现一个“可由现有可读资料解决、对 PCIe 驱动/调试重要、却完全没有正文落点”的重大缺口。

这并不意味着缺失资料已经得到解决：受保护的 Training PPT 无法合法读取；当时目录中也没有 Linux source、SoC TRM、PHY Databook、CEM、DTS 或目标 driver。因此，**通用 PCIe 与当前 Cadence IP 知识体系已经收口，而目标芯片和目标 Linux commit 的精确调用链仍处于 Blocked 状态**。后续六仓代码补充解决了其中一部分，剩余边界以最新 `OPEN_QUESTIONS.md` 为准。

## 2. 原始资料反查

| 输入 | 文件级扫描 | PCIe章节评估 | 核心深读与映射 | 最终判断 |
|---|---:|---:|---:|---|
| PCIe Base Specification 5.0 r1.0，1299页 | 是 | Ch.1–10、Appendices/Annex全部评估 | Transaction、Data Link、LTSSM/Logical PHY、配置、PM、error、SR-IOV、ATS等已按主题深读 | 通过；Electrical定量实现只保留规范/PHY边界 |
| Cadence Gen4 x16 Dual-Mode User Guide，1480页 | 是 | Ch.1–11、App.A–J全部评估 | 架构、AXI、LTSSM、translation、uDMA、PM、error、register/programming/integration及capability链已反向审计 | 通过；OQ-010/OQ-011是手册内部冲突 |
| MindShare *PCI Express System Architecture*，1106页 | 是，完整OCR | Ch.1–24、App.A–E、Index全部评估 | 基础模型、事务、配置、LTSSM、power/hotplug、NTB/analyzer进入正文，并作PCIe 5.0校正 | 通过；2003-era数值/接口不冒充现代规则 |
| 王齐《PCI Express体系结构导读》，455页 | 是，完整OCR | Ch.1–15全部评估 | 地址域、TLP、FC/ordering、MSI、Capric EP、IOMMU/ATS/SR-IOV、旧Linux已分主题校正 | 通过；Linux 2.6.31路径只作历史对照 |
| PCI Express Basics & Background，45页 | 是 | 全部 | 用作直觉/图解辅助，规范结论回查Base Spec | 通过 |
| PCI SDMS，274页 | 是 | 全目录和用途 | 判定为旧PCI/SCSI安装指南，低相关，不制造虚假PCIe主题 | 通过 |
| PCI Express Training 8-2004.ppt | 是 | 否 | E-SafeNet/LOCK保护容器，常规PowerPoint/OLE工具不可读 | Blocked：需未加密原件或授权打开方式 |

逐章证据见 [原始资料逐章 Coverage Matrix](research_notes/source_chapter_coverage_matrix.md)，版本与页数见 [SOURCE_INDEX](SOURCE_INDEX.md)。

## 3. 最初 30 个问题反查

| # | 读者问题 | 主要答案入口 | 结果 |
|---:|---|---|---:|
| 1–2 | PCIe是什么、为何比I2C/SPI复杂 | [世界观/拓扑/分层](fundamentals/overview_topology_and_layers.md) | 通过 |
| 3–4 | 软件看到的设备怎样出现、EP从上电到Linux发现经历什么 | [配置/ECAM](configuration/configuration_space_ecam_and_routing.md)、[PCI core/枚举](linux/pci_core_enumeration_and_driver_model.md)、[Endpoint的一天](examples/endpoint_from_power_on_to_dma_and_msix.md) | 通过；精确kernel内部函数链B |
| 5 | Link怎样建立 | [Link/Lane/LTSSM](link/link_lane_ltssm_and_training.md) | 通过；PHY analog实现B |
| 6–7 | Enumeration、Configuration Space | configuration三篇正文 | 通过 |
| 8–9 | BAR是什么、谁分配地址 | [BAR与地址翻译](addressing/bar_and_cadence_address_translation.md) | 通过 |
| 10–11 | RC访问EP、EP访问Host Memory | BAR/translation、DMA、worked example | 通过 |
| 12–13 | Memory Read/Write怎样成为TLP、Completion是什么 | [Memory TLP与Completion](transactions/memory_tlp_and_completion.md) | 通过 |
| 14 | PCIe DMA与普通SoC DMA差别 | [PCIe DMA、Cadence uDMA、Linux DMA API](dma/pcie_dma_cadence_and_linux_dma_api.md) | 通过 |
| 15 | MSI/MSI-X为何不用物理线 | [INTx/MSI/MSI-X](interrupts/intx_msi_msix_and_cadence.md) | 通过 |
| 16–17 | Linux PCI core做什么、driver怎样probe | [PCI core](linux/pci_core_enumeration_and_driver_model.md)、[Function driver](linux/function_driver_probe_mmio_and_lifecycle.md) | 架构/API通过；目标commit内部链B |
| 18 | `pci_enable_device()`做什么 | Function driver §2 | API与硬件职责通过；版本精确内部链B |
| 19 | `pci_request_regions()`做什么 | Function driver §4 | 通过 |
| 20 | `pci_iomap()`返回什么 | Function driver §5、MMIO worked example | 通过 |
| 21–22 | `dma_alloc_coherent()`、`dma_map_single()`与PCIe DMA | DMA §5–6 | 通过；平台cache/IOMMU实现B |
| 23 | IOMMU在数据路径哪里 | [IOMMU/ATS/PASID/PRI](advanced/iommu_ats_pasid_pri.md) | 通过；目标SMMU拓扑B |
| 24–25 | error怎样传播、AER怎样工作 | [错误模型/AER/Cadence](errors/error_model_aer_and_cadence.md)、[AER Debug](debug/aer_debug.md) | 通过 |
| 26 | Link Down软件看到什么 | reset/recovery、AER、hotplug文档 | 通过；目标IRQ glue B |
| 27–28 | Completion Timeout、Unsupported Request | [CTO runbook](debug/completion_timeout.md)、错误模型 | 通过 |
| 29 | Linux如何debug PCIe | [Linux debug工具](debug/linux_pcie_debug_tools.md)和8篇分层runbook | 通过 |
| 30 | 如何结合Databook写真实驱动 | [Host controller边界](linux/host_controller_driver_and_cadence_boundary.md)、[编程阶段](controller/cadence_register_space_and_programming_phases.md)、[Endpoint的一天](examples/endpoint_from_power_on_to_dma_and_msix.md) | 通用方法/Cadence IP通过；目标SoC代码B |

`B` 表示 Blocked by missing input，不表示前面的概念答案缺失。比如 `pci_iomap()` 返回CPU虚拟I/O mapping这件事已经能确认；“目标6.x内核究竟经过哪三个static helper、哪个arch hook”没有目标源码就不能编。

## 4. Databook feature 与关键寄存器反查

### 4.1 已完成的反查

- Appendix H 的 PF0、VF、RP selected capability逐项进入专题；generic-only VC/PTM/NPEM等没有被误报为当前build支持；
- Chapter 9 已按 PF/VF/RP Configuration、Local Management、AXI Configuration、PCIe DMA 六类职责反查；
- Region 0–31、Function×BAR、LINK0–3 的机械展开按模板消化，不把重复offset冒充新知识；
- LTSSM、credit、TLP count、LCRC/ECC、AER logs、MSI mask变化、lane margin/equalization monitor均有调试落点；
- Chapter 10 Config/Memory/Message region示例已用来验证translation文档；
- Chapter 11 reset、RAM integration、two-stage Completion buffer/backpressure进入controller正文。

详细表见 [Appendix H Capability Audit](research_notes/cadence_appendix_h_reverse_capability_audit.md) 和 [Chapter 9 Register Audit](research_notes/cadence_chapter9_reverse_register_audit.md)。

### 4.2 保留的冲突与边界

- OQ-010：uDMA interrupt enable/reset/`R/WOCLR`描述冲突；
- OQ-011：Link Down Reset是否保留LM寄存器，§4.7.2与§11.1.2.3冲突；
- OQ-009：手册configuration-specific表仍不能代替实际SoC config dump；
- PHY coefficient的controller侧握手已覆盖，analog算法/寄存器/校准必须等PHY Databook；
- 实际APB physical base、clock/reset/strap/IRQ/IOMMU路径必须等SoC TRM、DTS和driver。

## 5. 软件动作到硬件行为反查

| 软件动作 | 已闭合到的硬件行为 | 仍缺证据的位置 |
|---|---|---|
| config read | `pci_ops`概念入口→ECAM/Config aperture→Type0/1 CfgRd→CplD | 目标host driver和锁/ATU复用实现 |
| `writel(bar + off)` | CPU VA→physical MMIO→RC outbound window→MWr→EP BAR hit→Cadence AXI Master→寄存器 | 目标SoC页表/host window physical map |
| EP DMA read | DMA address→descriptor→Cadence outbound AXI Slave→MRd→IOMMU/Host memory→split CplD/Tag重组 | 目标SMMU与cache/coherency互连 |
| EP DMA write + MSI-X | data MWr与ordering→MSI-X address/data MWr→RC MSI target→IRQ domain→handler | 目标interrupt remapper/MSI controller源码 |
| Link/error事件 | PHY/DL/TL detector→LM/AER sticky/log→Message/Root Status→IRQ→Linux recovery架构 | 目标service driver精确调用链 |

这里也复核了三个常见“看起来像闭环其实不是”的坑：MWr没有正常Completion；跨TC的data MWr与MSI不自动保序；zero-length Read flush只对同TC/同target的适用路径负责。

## 6. 可理解性、结构和图示检查

- 首页、Learning Path、Knowledge Map、Glossary、Common Misconceptions均存在并互链；
- Learning Path已给11个阶段的阅读入口和“学完能回答什么”；
- 核心文档采用直觉→场景→协议→Cadence硬件→Linux→transaction/debug顺序；
- 重要例子固定BDF、BAR、Host地址、Tag和方向，不只停留在名词解释；
- 12张原始资料截图覆盖topology、Type0/Type1 header、LTSSM、receiver detect、Cadence层次、multifurcation、Completion buffer和capability chain；图注区分来源与PDF页；
- 行文以准确、自然和易于理解为主；必要的轻松表达不能替代字段语义、适用条件和证据说明。

## 7. 自动化/机械检查结果

执行于2026-08-23：

- Markdown文件：77；总行数超过9100；
- 知识库全部文件：88；目录大小约3.1 MiB；
- 本地Markdown/image相对链接：0个失效；
- 风险词检查：未发现正文把MPS写成LTSSM自动协商、FC credit写成端到端额度、MSI写成Message TLP，或把Cadence inbound/outbound AXI Master/Slave写反；
- `TODO/NEED/Unknown`反查：均为有说明的资料边界或OQ-010冲突，没有匿名TODO；
- 核心topic文档均含来源/证据定位，OCR资料的版本敏感结论已回查Base Spec/Cadence或明确标作历史。

## 8. 24项 DONE 条件裁决

| 条件组 | 裁决 |
|---|---|
| 1–8 输入扫描、核心深读、Spec/Databook独立消化与双向mapping | 对所有可读输入通过；受保护PPT为明确Blocked，不冒充已读 |
| 9–17 软件流程、Linux主线、DMA、MSI-X、枚举、BAR/translation、LTSSM、error/debug | 通用协议与Cadence层通过；目标Linux/SoC精确实现受缺资料阻塞 |
| 18–20 Knowledge Map、Learning Path、Glossary | 通过，并已在本轮复核导航与新增Gen4观测面 |
| 21 PROGRESS无未处理重要任务 | 无可由现有资料继续解决的重要项；剩余均标Blocked/Open且列出所需证据 |
| 22 OPEN_QUESTIONS无现有资料可解的重要问题 | 通过；OQ-009/010/011需要实芯片/errata/RTL，OQ-001–005需要新输入 |
| 23 执行FINAL COVERAGE AUDIT | 通过，即本文 |
| 24 Audit无重大遗漏 | 对当前证据集通过；发现的都是外部资料缺口，不是知识库断链 |

## 9. 新资料到来后的准确续跑点

这套库对**当前证据集**完成收口，但仍设计为可继续生长：

1. 收到目标 Linux tree/commit：从 OQ-001 开始，追 `pci_enable_device()`、resource allocator、MSI/AER、host/EPC driver的精确symbol/line调用链；
2. 收到 SoC TRM/DTS：从 OQ-002 开始，把APB/ECAM/outbound window/IRQ/SMMU physical map落到当前芯片；
3. 收到PHY Databook：从 OQ-004 开始，补reset/calibration、preset/coefficient、margining单位和板级debug；
4. 收到实芯片dump/errata：关闭 OQ-009/010/011，建立actual-build capability/reset/IRQ matrix；
5. 收到未加密Training材料：补做来源索引与年代校正，不因为它名字里有Training就默认它比5.0 Spec更权威。
