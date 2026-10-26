# 正文文章索引

这份索引用于快速判断“某篇文章解决什么问题、适合在什么时候读”。它只收录面向读者的正文和源码专题，不把研究过程记录混入学习路线。原始资料清单见 [SOURCE_INDEX](SOURCE_INDEX.md)，知识依赖关系见 [KNOWLEDGE_MAP](KNOWLEDGE_MAP.md)。

## 基础、配置与地址

- [PCIe 世界观：拓扑、角色与三层协议](fundamentals/overview_topology_and_layers.md)：从低速外设开发者熟悉的视角解释 Link、Lane、RC、RP、EP、Switch、BDF 和三层协议。
- [Configuration Space、ECAM 与配置请求路由](configuration/configuration_space_ecam_and_routing.md)：解释设备尚无 BAR 地址时 Host 如何发现它，以及 Type 0/Type 1、ECAM 和配置 TLP 怎样配合。
- [BDF、Requester ID 与层次路由](configuration/bdf_requester_id_and_hierarchy_routing.md)：区分软件地址、TLP 身份和 Completion 匹配所使用的 BDF、Requester ID、Completer ID 与 Tag。
- [Capability 的发现与安全遍历](configuration/capability_discovery_and_safe_walking.md)：说明 conventional/extended capability 两类链表、异常链保护和“能力可见不等于可以启用”。
- [Expansion ROM / Option ROM](configuration/expansion_rom_and_option_rom.md)：介绍 ROM BAR sizing、临时 decode、镜像信任和 Linux ROM resource 路径，并说明当前 Cadence build 的实现边界。
- [BAR 与 Cadence 地址翻译](addressing/bar_and_cadence_address_translation.md)：把 BAR sizing、资源分配、CPU/PCIe/DMA 地址域，以及 Cadence inbound/outbound translation 串成一条路径。

## Transaction 与 Data Link

- [Memory TLP 与 Completion](transactions/memory_tlp_and_completion.md)：逐字段解释 Memory Read/Write、Byte Enable、Requester ID、Tag、Cpl/CplD 和错误返回。
- [MPS、MRRS 与 Ordering](transactions/mps_mrrs_ordering_and_performance.md)：区分 TLP payload 上限、读请求大小、Completion 切分、流控和顺序规则，并给出性能排查顺序。
- [Message、AtomicOp 与 TLP Prefix](transactions/message_atomic_and_tlp_prefix.md)：覆盖 Message routing、AtomicOp、legacy locked transaction、PASID Prefix 和 TPH，并注明当前 Cadence 不支持 AtomicOp。
- [Data Link Layer：确认、重放与流量控制](link/data_link_ack_replay_and_flow_control.md)：解释 Sequence Number、LCRC、Ack/Nak、Replay Buffer、InitFC/UpdateFC 以及 credit 的逐跳性质。

## Link、训练与控制器

- [Link、Lane 与 LTSSM](link/link_lane_ltssm_and_training.md)：从 Detect 到 L0/Recovery 讲完整建链过程，并映射 Cadence 状态编码、历史记录和 Gen3/Gen4 equalization 观测面。
- [Cadence PCIe Core 整体架构](controller/cadence_core_architecture.md)：介绍当前 Gen4 x16 Dual-Mode 配置、内部模块、AXI/PIPE 接口和软硬件职责边界。
- [Cadence 多链路与 Multifurcation](controller/cadence_multifurcation_and_link_instances.md)：解释 16 条 Lane 如何映射到多个独立 Link core，以及 strap、PHY、clock 和 reset 的前置条件。
- [Cadence 寄存器空间与编程阶段](controller/cadence_register_space_and_programming_phases.md)：区分 Function/RP Configuration、Local Management、AXI Configuration 和 uDMA 空间，并按 reset、training、enumeration 阶段安排配置。
- [Cadence Completion Buffer 与反压](controller/cadence_completion_buffers_and_backpressure.md)：分析 split Completion、AXI 重组、two-stage buffer、ordering block 和 NP outstanding 限制。
- [Cadence uDMA 编程](controller/cadence_udma_programming.md)：从 capability、descriptor、channel 启停、IRQ 到 Link Down residue，说明内置 uDMA 的完整生命周期。
- [Cadence Reset 与恢复矩阵](controller/cadence_reset_and_recovery_matrix.md)：比较 Fundamental Reset、Hot Reset、Link Down Reset 和 FLR 的作用域、保留状态及恢复责任。

## 中断、DMA 与高级地址能力

- [INTx、MSI、MSI-X 与 Cadence](interrupts/intx_msi_msix_and_cadence.md)：解释虚拟 INTx Message 和 MSI/MSI-X Memory Write，并连接 Cadence EP/RP、Linux vector 与 X6000 IRQ domain。
- [PCIe DMA、Cadence uDMA 与 Linux DMA API](dma/pcie_dma_cadence_and_linux_dma_api.md)：区分总线主设备、搬运引擎和 DMA 地址映射，覆盖 coherent、streaming、SG、IOMMU 与 cache/order。
- [IOMMU、ATS、PASID、PRI](advanced/iommu_ats_pasid_pri.md)：从 Requester ID 和 IOVA 出发，解释地址翻译缓存、进程地址空间标识、缺页请求和 invalidation。
- [ARI 与 SR-IOV](advanced/ari_sriov_pf_vf.md)：介绍扩展 Function 编号、PF/VF、VF BAR、FLR、Linux SR-IOV 管理和隔离检查。
- [TC、VC、QoS、PTM 与 ACS](advanced/tc_vc_qos_ptm_and_acs.md)：区分流量标签、逻辑资源、时间测量和访问控制，并核对当前 Cadence build 实际选中的能力。
- [Non-Transparent Bridge 与多 Host](advanced/non_transparent_bridge_and_multi_host.md)：说明两个独立枚举域如何通过 Memory Window、translation、doorbell 和 scratchpad 通信，以及当前 X6000 的证据边界。

## 错误、电源与热插拔

- [PCIe 错误模型与 AER](errors/error_model_aer_and_cadence.md)：从 Completion Status、ERR_* Message、Poison、AER log 追到 Cadence AXI/sideband 和 Linux recovery。
- [DPC 下游错误隔离](errors/dpc_containment_and_recovery.md)：解释 DPC 如何在 Downstream Port 阻断流量、记录根因并与 AER 恢复协作。
- [Function D-state、ASPM、L1SS 与 PME](power/function_dstates_aspm_l1ss_and_pme.md)：区分 Function 和 Link 状态，说明进入/退出、CLKREQ#、唤醒和 Linux suspend/resume 边界。
- [Dynamic Power Allocation](power/dynamic_power_allocation_dpa.md)：解释 D0 内部功耗 substate、Cadence client 握手和默认 power table 的集成风险。
- [Slot Power Limit 与 Power Budgeting](power/slot_power_limit_and_power_budgeting.md)：区分设备功耗需求报告、插槽额度和 DPA，覆盖标准 Message 与 Cadence 捕获字段。
- [LTR 与 OBFF](power/ltr_and_obff_latency_power_hints.md)：说明设备延迟需求如何向上聚合，以及平台如何提示 Endpoint 集中访问中央资源。
- [PCIe Hot Plug 与异步移除](power/hotplug_orderly_and_async_removal.md)：比较有序 quiesce、Surprise Down、DPC/HPS containment 和 Linux pciehp 的证据边界。

## Linux 驱动

- [Linux PCI Core](linux/pci_core_enumeration_and_driver_model.md)：解释 `pci_host_bridge`、`pci_bus`、`pci_dev`、枚举、资源分配和 driver binding，并落到 X6000 `pci_ops`。
- [Linux PCI Function Driver](linux/function_driver_probe_mmio_and_lifecycle.md)：沿 `probe()` 解释 enable、region claim、BAR mapping、BME、DMA、IRQ 和 remove，追到 X6000 Linux 5.10.4 实现。
- [Linux PCI Host Controller Driver](linux/host_controller_driver_and_cadence_boundary.md)：区分 Host、Function、EPC、EPF 四类驱动，说明 X6000 RC/EP platform glue 如何接入通用 PCI/Endpoint Framework。

## X6000 多仓库源码专题

- [代码仓来源索引](code_studies/SOURCE_REPOSITORY_INDEX.md)：固定 Linux、edge-driver、BootROM、PCS、ATF、edge10_tool/HAL 的仓库、分支、commit 和证据等级。
- [BootROM PCIe 启动流程](code_studies/bootrom_pcie_boot_flow.md)：分析上电、PHY/controller 初始化、BAR0 mailbox、临时 APT、分支特有配置和下一阶段移交。
- [BootROM PCIe 分支矩阵](code_studies/bootrom_pcie_branch_matrix.md)：比较 21 个远端分支的 PCIe 源码、构建条件、MPS/地址映射和实验特性。
- [PCS PCIe 阶段](code_studies/pcs_x6000_pcie_stage.md)：说明 PCS 如何保持 Link、重写 APT、管理 mailbox、多 die 启动、功耗和 uDMA。
- [ATF/BL2 PCIe 阶段](code_studies/atf_x6000_pcie_stage.md)：覆盖大镜像下载、C2C 枚举、uDMA、MSI-X relay 和静态审计风险。
- [X6000 Linux PCIe 栈](code_studies/linux_x6000_pcie_stack.md)：分析 Cadence platform、RC、EPC、EPF、DTS、BAR/APT、outbound ATU 和中断调用链。
- [Host edge-driver](code_studies/edge_driver_x6000.md)：按 probe、boot magic、mailbox、MMIO、uDMA、MSI-X、AER 和 ioctl 路径拆解 Host 功能驱动。
- [edge10_tool x6000](code_studies/edge10_tool_x6000_host.md)：把 scan、bario、xferstress、shared memory、bandwidth、hot reset 等命令追到 HAL、ioctl 和硬件事务。
- [X6000 统一地址地图](code_studies/x6000_address_map.md)：集中记录各启动阶段与产品 DTS 下的 BAR offset、APT region、AXI target、outbound/DMA address。
- [X6000 启动交接与所有权](code_studies/x6000_boot_handoff_and_ownership.md)：用时间线和状态所有权表说明 BootROM、PCS、BL2、Linux 在同一 Link 上如何继承和覆盖硬件状态。
- [X6000 综合 PCIe 架构](code_studies/x6000_integrated_pcie_architecture.md)：汇总 Host、A0/A8/B0、C2C、MMIO、H2D/D2H DMA、MSI-X 和跨仓 ABI，并给出七阶段源码阅读路线。

## Debug 手册与完整实例

- [Link Not Up](debug/link_not_up.md)：根据 LTSSM 停留状态检查电源、reset、refclk、PHY、TS1/TS2、width/speed 和 Data Link 初始化。
- [Enumeration Failure](debug/enumeration_failure.md)：从 Vendor ID 配置读出发，逐层检查 config aperture、Type 0/1、bus number 和 Completion。
- [BAR Problem](debug/bar_problem.md)：区分 BAR sizing、Host resource、bridge window、RC outbound、EP BAR hit 和本地 APT。
- [Memory Access Failure](debug/memory_access_failure.md)：用 MWr/MRd、Byte Enable、Completion 和 AXI response 定位 BAR 映射后仍无法访问的问题。
- [DMA Failure](debug/dma_failure.md)：按 engine、BME/D-state、DMA address/IOMMU、TLP/Completion、cache/order 分层排查。
- [MSI / MSI-X Failure](debug/msi_failure.md)：从设备状态、table/mask、MSI Memory Write、RC decode、IRQ domain 一直查到 handler。
- [Completion Timeout](debug/completion_timeout.md)：区分请求未发出、目标未响应、Completion 丢失/不匹配和超时配置问题。
- [AER Debug](debug/aer_debug.md)：说明先保存 Header Log、Source ID 和上下文，再处理 W1C、containment 与 recovery。
- [Linux PCIe Debug 工具](debug/linux_pcie_debug_tools.md)：解释 `lspci`、`setpci`、sysfs、dmesg、debugfs 和 `/proc/iomem` 的数据来源与安全边界。
- [Protocol Analyzer 分层捕获](debug/protocol_analyzer_and_layered_capture.md)：设计双向 trigger、关联 RID/Tag/address，并把 analyzer、controller trace 和示波器放在各自适合的层次。
- [Endpoint 完整工作流程](examples/endpoint_from_power_on_to_dma_and_msix.md)：用固定 BDF、BAR、Host buffer 和 Tag，把上电、枚举、`writel()`、DMA Read/Write、MSI-X 与错误恢复串成一个实例。
