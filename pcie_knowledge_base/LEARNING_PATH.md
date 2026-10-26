# Learning Path

下面的路线不是考试大纲，而是尽量避免“每个字都认识，连起来完全不知道在说什么”的阅读顺序。

## Stage 1：建立 PCIe 世界观

必须理解：PCIe 是保留 PCI 软件模型的点到点串行分组互连；RC、RP、EP、Switch、Link、Lane 和三层协议各自负责什么。

先读：

- [PCIe 世界观：拓扑、角色与三层协议](fundamentals/overview_topology_and_layers.md)
- [术语表](GLOSSARY.md) 中 RC、RP、EP、Link、Lane、TLP、DLLP、LTSSM

学完应能回答：PCIe 为什么比 SPI 复杂？RC 和 Root Port 是不是同一个东西？x4 是四倍频率还是四条 Lane？

## Stage 2：设备如何被发现——Configuration 与 Enumeration

必须理解：BDF、Configuration Space、Type 0/1 Header、ECAM、Capability 链，以及枚举怎样构建 hierarchy。

先读：

- [Configuration Space、ECAM 与配置路由](configuration/configuration_space_ecam_and_routing.md)
- [BDF、Requester ID 与层次路由](configuration/bdf_requester_id_and_hierarchy_routing.md)
- [Capability discovery 与安全遍历](configuration/capability_discovery_and_safe_walking.md)

学完应能回答：设备还没 BAR 地址时，Linux 如何访问它？Bus Number 是谁分配的？Vendor ID 读回全 1 通常意味着什么？

选读：[Expansion ROM / Option ROM](configuration/expansion_rom_and_option_rom.md)。它是一个很好的综合例子：Configuration register描述decoder，firmware临时分配resource，CPU读最终变成Memory Read TLP，现代平台还要叠加trust policy。

## Stage 3：BAR 与 MMIO

必须理解：BAR 是地址窗口声明，不是“那块内存本身”；size probing、resource allocation、bridge window、CPU address 与 PCI address 的区别。

先读：[BAR 与 Cadence 地址翻译](addressing/bar_and_cadence_address_translation.md)。

学完应能从 `pci_resource_start()`、`pci_iomap()`、`writel()` 一路讲到 EP BAR hit 和 AXI/APB 寄存器。

## Stage 4：TLP 与 Completion

必须理解：Memory、I/O、Configuration、Message 四类事务；3DW/4DW header、Requester/Completer ID、Tag、Length、Byte Enable；Posted 与 Non-Posted。

先读：

- [Memory TLP 与 Completion](transactions/memory_tlp_and_completion.md)
- [MPS、MRRS、Ordering 与性能](transactions/mps_mrrs_ordering_and_performance.md)
- [Message、AtomicOp 与 TLP Prefix](transactions/message_atomic_and_tlp_prefix.md)

学完应能手算一个 4-byte Memory Write 的关键头字段，并解释为什么 Memory Write 没有正常 Completion、Memory Read 却必须等 CplD。

## Stage 5：Linux PCI Driver

必须理解：`pci_driver` 如何匹配 `pci_dev`；probe/remove 生命周期；enable、regions、iomap、bus mastering、DMA mask、IRQ vector 与 PM API 各自在保护什么资源或改变什么硬件状态。

先读：

- [Linux PCI core、枚举与 driver model](linux/pci_core_enumeration_and_driver_model.md)
- [Function driver：probe、MMIO 与生命周期](linux/function_driver_probe_mmio_and_lifecycle.md)
- 如果你写的是 RC/EPC glue：[Host controller driver 与 Cadence 边界](linux/host_controller_driver_and_cadence_boundary.md)

学完应能解释 `pci_enable_device()` 不是一句神秘咒语，以及为什么 `pci_set_master()` 通常是另一件事。

通用章节先解释稳定语义；X6000 的 Linux 5.10.4 精确实现链见
[X6000 Linux PCIe栈](code_studies/linux_x6000_pcie_stack.md)与
[Host edge-driver](code_studies/edge_driver_x6000.md)。

## Stage 6：DMA 与地址域

必须理解：EP 作为 Requester 主动读写 Host Memory；CPU physical、PCI bus address、DMA address/IOVA 的区别；coherent vs streaming mapping；SG、Cache、IOMMU。

先读：[PCIe DMA、Cadence uDMA 与 Linux DMA API](dma/pcie_dma_cadence_and_linux_dma_api.md)。随后用 [Endpoint 完整实例](examples/endpoint_from_power_on_to_dma_and_msix.md) 把 descriptor、MRd/CplD、MWr 和 completion IRQ 串起来。

学完应能解释为什么驱动必须把 DMA address 写进设备 descriptor，而不是随手塞一个内核虚拟地址——后者通常是把 bug 做得很有自信。

## Stage 7：Interrupt

必须理解：INTx 的兼容语义、MSI/MSI-X 的 Memory Write 本质、vector/mask/table、Linux IRQ domain 与 interrupt remapping。

先读：[INTx、MSI/MSI-X 与 Cadence](interrupts/intx_msi_msix_and_cadence.md)，再用 [MSI/MSI-X Failure](debug/msi_failure.md) 检查自己是否真能沿整条投递路径排障。

学完应能画出 EP 触发 MSI-X 到 Linux handler 被调用的完整路径。

## Stage 8：Link、PHY 与 LTSSM

必须理解：Detect、Polling、Configuration、L0、Recovery、Disabled、Hot Reset 等状态；TS1/TS2、lane number、width/speed negotiation、equalization；Controller 与 PHY 的 PIPE 边界。

先读：[Link、Lane 与 LTSSM](link/link_lane_ltssm_and_training.md) → [Data Link ACK、Replay 与 Flow Control](link/data_link_ack_replay_and_flow_control.md) → [Link Not Up](debug/link_not_up.md)。

学完应能从“Link not up”判断应该看参考时钟、PERST#、receiver detect、training ordered set、equalization 还是协议层，而不是第一反应就重装驱动。

## Stage 9：Error Handling 与 Debug

必须理解：Completion Timeout、Unsupported Request、Completer Abort、Poisoned/Malformed TLP、AER、Surprise Down，以及错误如何从硬件传播到 Linux recovery。

先读：[错误模型、AER 与 Cadence](errors/error_model_aer_and_cadence.md)，再按现象进入 `debug/`。不要一看到 AER 就先清状态位；Header Log 还没拍照，案发现场就没了。

学完应能按层定位 enumeration failure、BAR access failure、DMA failure、MSI failure 与 Link Down。

工具进阶：[Protocol Analyzer与分层抓包](debug/protocol_analyzer_and_layered_capture.md)。学完应能把一个问题改写成“在哪条Link、哪个方向、用哪个RID/Tag/address和什么前置事件触发”，而不是只说“帮我抓个PCIe包”。

建议从 [Linux PCIe Debug 工具](debug/linux_pcie_debug_tools.md) 进入对应现象 runbook；严重下游错误另读 [DPC containment 与恢复](errors/dpc_containment_and_recovery.md)。

## Stage 10：Power 与 Advanced PCIe

必须理解：Function D-state 与 Link state 的区别；ASPM、L1SS、PME；再进入 TC/VC/QoS、PTM、ACS、ARI、SR-IOV、FLR、ATS、PASID、PRI。

学完应能回答这些高级能力解决了哪个真实问题、硬件路径在哪里、IOMMU/虚拟化软件为什么需要它，而不只是背缩写。

进阶入口：[TC、VC、QoS、PTM 与 ACS](advanced/tc_vc_qos_ptm_and_acs.md)、[ARI/SR-IOV/PF/VF](advanced/ari_sriov_pf_vf.md)、[IOMMU/ATS/PASID/PRI](advanced/iommu_ats_pasid_pri.md)。

## Stage 11：Cadence Core 与真实芯片

必须理解：Cadence inbound AXI Master / outbound AXI Slave（以 core 为观察者）、PNP/Completion FIFO、Replay Buffer、LTSSM、PIPE、本地管理/配置寄存器、DMA、多链路配置；再叠加目标 SoC 的 clock/reset/PHY/address map 和 Linux glue driver。

学完应能依据 Databook 重建：复位 → 静态配置 → PHY → LTSSM → 枚举 → BAR/translation → DMA/MSI → error recovery 的完整编程序列。

推荐按 [Core 架构](controller/cadence_core_architecture.md) → [Multifurcation](controller/cadence_multifurcation_and_link_instances.md) → [寄存器空间/编程阶段](controller/cadence_register_space_and_programming_phases.md) → [uDMA](controller/cadence_udma_programming.md) → [Reset/恢复矩阵](controller/cadence_reset_and_recovery_matrix.md) 阅读。

做真实 bring-up 时，再把 [Chapter 9反向寄存器审计](research_notes/cadence_chapter9_reverse_register_audit.md) 当作仪表盘索引：TLP/credit/LCRC/ECC/equalization计数用于定位层级，不能代替 protocol analyzer、PHY测量或目标 Linux源码。

## Stage 12：X6000 六仓实战

这一阶段把前十一阶段的通用知识落到一张真实多die PCIe卡上。

推荐顺序：

1. [综合架构](code_studies/x6000_integrated_pcie_architecture.md)：先知道所有参与者；
2. [BootROM主流程](code_studies/bootrom_pcie_boot_flow.md)与
   [21分支矩阵](code_studies/bootrom_pcie_branch_matrix.md)；
3. [PCS](code_studies/pcs_x6000_pcie_stage.md)和
   [ATF BL2](code_studies/atf_x6000_pcie_stage.md)；
4. [设备Linux Cadence栈](code_studies/linux_x6000_pcie_stack.md)；
5. [Host edge-driver](code_studies/edge_driver_x6000.md)；
6. [edge10_tool/HAL](code_studies/edge10_tool_x6000_host.md)；
7. 回到[启动交接](code_studies/x6000_boot_handoff_and_ownership.md)和
   [统一地址地图](code_studies/x6000_address_map.md)做闭环。

学完应能回答：

- A0如何从上电走到Host `edge_probe()`？
- 为什么Host驱动可以编程设备里的uDMA？
- 同一个`BAR0+0x80000`为何在PCS与Linux指向不同目标？
- H2D为什么产生Memory Read和CplD？
- B0通知为什么要经过A8捕获与A0 MSI-X relay？
- Link Down、APT错配、IOMMU拒绝和MSI-X错配分别会留下什么证据？
