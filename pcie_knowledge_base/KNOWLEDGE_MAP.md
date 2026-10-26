# PCIe Knowledge Map

这张图不是目录装饰品，而是“为什么要先学 A 再学 B”的依赖图。`[x]` 表示已有可靠正文，`[~]` 表示当前资料只支持部分深入；真正未完成项统一维护在 `PROGRESS.md`，避免地图和进度表各过各的日子。

```text
[x] PCIe 世界观
 │
 ├─[x] 为什么它仍叫 PCI
 │    └─ 保留 load/store、Configuration Space、BAR、枚举和驱动模型
 │
 ├─[x] 点到点 Fabric / Hierarchy
 │    ├─[x] Root Complex (RC)
 │    │    └─[x] Root Port (RP)
 │    ├─[x] Endpoint (EP)
 │    ├─[x] Switch（逻辑上是一组 PCI-to-PCI Bridges）
 │    └─[x] Bus / Device / Function (BDF)
 │          └─[x] Requester ID / Completer ID / ID Routing
 │
 └─[x] 分层
      ├─[x] Transaction Layer
      │    ├─[x] TLP 类型与通用头
      │    ├─[x] Address / ID / Implicit Routing
      │    ├─[x] Posted / Non-Posted
      │    ├─[x] Memory / I/O / Configuration / Message
      │    ├─[x] Completion / Tag / IDs / Byte Enables
      │    ├─[x] Ordering / TC / VC
      │    └─[x] Credit-based Flow Control
      ├─[x] Data Link Layer
      │    ├─[x] Sequence Number / LCRC
      │    ├─[x] ACK / NAK DLLP
      │    └─[x] Replay Buffer / Link-local reliability
      └─[~] Physical Layer
           ├─[x] Link / Lane / speed / width
           ├─[~] PIPE / PHY / encoding / ordered sets（独立 PHY databook 缺失）
           ├─[x] LTSSM / TS1 / TS2
           ├─[~] Equalization coefficient（controller 已覆盖，PHY 实现缺失）
           │    └─[x] Secondary PCIe / Lane Margining / Physical Layer 16 GT/s 软件观测面
           └─[x] L0 / L0s / L1 / L2
```

```text
[x] “设备为什么会出现在 Linux 里”
 │
 ├─[x] Configuration access
 │    ├─ ECAM or controller-specific config aperture
 │    └─ CPU load/store → host bridge → Configuration TLP
 ├─[x] Enumeration
 │    ├─ Vendor ID 探测 Function 是否存在
 │    ├─ Type 0 / Type 1 Header
 │    ├─ Bridge Bus Number assignment
 │    ├─ Capability traversal
 │    └─ pci_dev / pci_bus object creation
 └─[x] Resource allocation
      ├─[x] BAR sizing probe
      ├─[x] Address window placement
      ├─[x] BAR programming
      ├─[x] Bridge aperture programming
      └─ Linux resource tree
```

```text
[x] “一次 CPU 写 EP 寄存器到底发生什么”

writel(value, __iomem_virtual)
  → CPU page table / I/O mapping
  → CPU physical MMIO address
  → RC outbound address window
  → PCIe Memory Write Request TLP（Posted）
  → Switch address routing（如果有）
  → EP BAR hit
  → EP inbound/target address translation
  → Cadence inbound AXI Master interface（core 发本地 AXI）
  → device AXI/APB register
```

依赖：Enumeration → BAR allocation → host bridge window → `pci_iomap()` → TLP → EP address decode。

已形成正文：

- [Configuration Space、ECAM 与配置请求路由](configuration/configuration_space_ecam_and_routing.md)
- [BDF、Requester ID 与层次路由](configuration/bdf_requester_id_and_hierarchy_routing.md)
- [Capability discovery 与安全遍历](configuration/capability_discovery_and_safe_walking.md)
- [Expansion ROM / Option ROM](configuration/expansion_rom_and_option_rom.md)
- [BAR 与 Cadence 地址翻译](addressing/bar_and_cadence_address_translation.md)

```text
[x] “一次 EP DMA 读 Host Memory 到底发生什么”

Linux dma_map_* / dma_alloc_*
  → DMA address（可能是 IOVA，不保证是 CPU physical）
  → driver 写入 EP DMA descriptor
  → EP client / Cadence outbound AXI Slave request
  → PCIe Memory Read Request TLP（Non-Posted，带 Requester ID + Tag）
  → RC / optional IOMMU translation and permission check
  → Host memory read
  → one or more Completion with Data TLPs
  → EP matches Tag and reassembles data
  → DMA completion / MSI-X
```

依赖：Bus Master Enable → address domain → DMA API → IOMMU → MRd/CplD → Ordering → interrupt。

完整 worked example：[一个 Endpoint 的一天：上电、枚举、MMIO、DMA 与 MSI-X](examples/endpoint_from_power_on_to_dma_and_msix.md)。

```text
MPS（单个 data TLP 上限）
  ↔ receiver/replay buffer 与 Flow Control data credit
MRRS（单个 Read Request 上限）
  → RCB/MPS 切分多个 CplD
  → Tag/outstanding/reordering buffer
Ordering（同 TC 的 passing 约束）
  ↔ RO/IDO；No Snoop 只管 cache coherency
```

已形成正文：[MPS、MRRS、Ordering 与性能](transactions/mps_mrrs_ordering_and_performance.md)。

其他 TLP/错误边界：

- [Message、AtomicOp 与 TLP Prefix](transactions/message_atomic_and_tlp_prefix.md)
- [DPC containment 与恢复](errors/dpc_containment_and_recovery.md)

```text
TC：TLP 上的端到端服务标签
  → 每个 Port 执行 TC→VC mapping
  → VC：独立 queue/buffer/Flow Control credit
  → arbitration 与逐跳资源形成 QoS

PTM：逐 Link Request/Response timestamp
  → 建立 local time ↔ PTM Master Time

ACS：按 RID/AT/P2P policy 检查 TLP
  → normal route / block / redirect to RC Translation Agent
```

已形成正文：[TC、VC、QoS、PTM 与 ACS](advanced/tc_vc_qos_ptm_and_acs.md)。当前 Cadence build 的 configuration-specific tables 未选中额外 VC、PTM 或 ACS Capability；这与通用 capability 图的“可选表面”必须分开。

Multi-host在topology上又分出一条支线：

```text
Transparent Bridge → 同一Host继续enumeration下游tree
Non-Transparent Bridge → 两个Host各自停在Type 0 face
                       → Memory Window + translation + doorbell/scratchpad
```

已形成正文：[Non-Transparent Bridge与multi-host](advanced/non_transparent_bridge_and_multi_host.md)。当前Cadence guide没有NTB feature，uDMA Doorbell也不是peer doorbell。

```text
[x] Interrupt
 ├─ INTx：传统语义，在 PCIe 上通常是 Message，不是四根专用线贯穿链路
 ├─ MSI：设备发一笔规范化 Memory Write
 ├─ MSI-X：每 vector 独立 address/data/mask table
 └─ Linux：vector allocation → capability programming → irq domain/remapping → handler

[x] Error propagation
 TLP/link fault
  → layer-specific detection
  → status bit / error message
  → Root Port / AER record
  → controller IRQ
  → Linux AER service / driver recovery callbacks

[x] Power
 Function D-state  ≠  Link LTSSM state
  ├─ D0/D1/D2/D3hot/D3cold
  ├─ ASPM L0s/L1
  ├─ L1 Substates
  ├─ PME wakeup
  ├─ Power Budgeting：Function报告需求
  ├─ Slot Power Limit：Port下发adapter额度
  ├─ DPA：D0内部动态功耗substate
  ├─ LTR：EP上报可容忍service latency
  ├─ OBFF：平台提示低增量功耗工作窗口
  └─ Hot Plug
       ├─ orderly：quiesce→disable→power off→remove
       └─ async：DL_Down/error→DPC/HPS containment→software cleanup
```

## Cadence 实现映射（第一版）

```text
AXI Application Layer
 ├─ AXI Master Read/Write：core 把链路来的 inbound request 发到本地 AXI target
 ├─ AXI Slave Read/Write：本地 client 访问 core，主动生成 outbound PCIe request
 └─ Interrupt Interface
        ↓↑
Transaction Layer
 ├─ Flow Control
 ├─ PNP Receive FIFO RAM
 ├─ Completion Receive FIFO RAM
 └─ 最多跟踪 64 个 master-side Non-Posted transactions
        ↓↑
Link Layer
 ├─ Link State
 ├─ LCRC check/generate
 └─ Replay Buffer RAM
        ↓↑
Physical Layer
 ├─ LTSSM
 ├─ lane de-skew / scramble / framing
 └─ PIPE Interface → external PHY

Config Registers / Local Management Registers 横向控制各层。
```

来源：Cadence 4.1，printed pp.202–204 / PDF pp.291–293；Figure 4.1。

已形成控制器正文：

- [Cadence Core 架构](controller/cadence_core_architecture.md)
- [Cadence Multifurcation 与 Link 实例](controller/cadence_multifurcation_and_link_instances.md)
- [Cadence 寄存器空间与编程阶段](controller/cadence_register_space_and_programming_phases.md)
- [Cadence uDMA 编程](controller/cadence_udma_programming.md)
- [Cadence Reset 与恢复矩阵](controller/cadence_reset_and_recovery_matrix.md)
- [Cadence Completion Buffer 与 backpressure](controller/cadence_completion_buffers_and_backpressure.md)

```text
16 PHY/PIPE Lanes
  → static LANE_MAP_MODE
  → 1～4 个独立 LINK{N} core
  → 每 Link 独立 LTSSM/Data Link/Transaction
  → 每 Link APB
       ├─ PF/VF or Root Port Configuration
       ├─ Local Management
       ├─ AXI Configuration
       └─ PCIe DMA
```

## Link bring-up dependency

```text
Power / refclk / reset / PHY initialization
  → LTSSM Detect：receiver termination
  → Polling：TS1/TS2、lock、polarity
  → Configuration：Link/Lane number、width、de-skew
  → Physical L0 at Gen1
  → optional Recovery.Speed/Equalization → higher generation
  → Data Link Feature + InitFC
  → DL_Up
  → Configuration transaction
  → Enumeration
```

已形成正文：

- [Link、Lane 与 LTSSM](link/link_lane_ltssm_and_training.md)
- [Data Link ACK、Replay 与 Flow Control](link/data_link_ack_replay_and_flow_control.md)
- [Link Not Up 分层排障](debug/link_not_up.md)

Cadence Chapter 9 的计数器、credit、LCRC/ECC、mask-change 和 equalization monitor 已按“动作寄存器/观测寄存器”反向归档；见 [Chapter 9反向寄存器审计](research_notes/cadence_chapter9_reverse_register_audit.md)。这些寄存器可以帮助判断故障接近哪一层，但无法替代 PHY 眼图或其他模拟链路测量。

## Interrupt delivery dependency

```text
Device interrupt condition
  ├─ legacy compatibility → INTx virtual-wire state → Assert/Deassert Message TLP
  └─ modern path
       ├─ MSI Capability address/data
       └─ MSI-X BAR Table entry + PBA
            ↓
       EP outbound address translation
            ↓
       Posted Memory Write TLP
            ↓
       RC / platform MSI controller / interrupt remapper
            ↓
       Linux IRQ domain → request_irq handler
```

已形成正文：[INTx、MSI、MSI-X 与 Cadence 实现](interrupts/intx_msi_msix_and_cadence.md)。

目标相关断点：Cadence RP AXI Master 之后的 MSI controller/remapper、Linux irq-domain 实现需要 SoC TRM 与 kernel tree。

## Error propagation dependency

```text
PHY / DLL / Transaction / internal detector
  → local handling: replay / discard / poison / UR / CA / SLVERR
  → AER Status (sticky W1C)
       ├─ Mask：是否 log/report
       └─ Severity：Non-Fatal / Fatal
  → optional ERR_COR / ERR_NONFATAL / ERR_FATAL Message
  → Root Port Root Error Status + Source ID + platform IRQ
  → Linux AER service
  → driver error recovery callbacks / reset / disconnect
```

已形成正文：

- [PCIe 错误模型、AER 与 Cadence](errors/error_model_aer_and_cadence.md)
- [Completion Timeout 排障](debug/completion_timeout.md)
- [AER Debug](debug/aer_debug.md)
- [Enumeration Failure](debug/enumeration_failure.md)
- [BAR Problem](debug/bar_problem.md)
- [Memory Access Failure](debug/memory_access_failure.md)
- [DMA Failure](debug/dma_failure.md)
- [MSI/MSI-X Failure](debug/msi_failure.md)
- [Protocol Analyzer与分层抓包](debug/protocol_analyzer_and_layered_capture.md)

## Power-state dependency

```text
Function D-state (PMCSR)
  ├─ D0：仍可由 ASPM 让空闲 Link 进入 L0s/L1/L1SS
  └─ all downstream Functions non-D0：PCI-PM path 请求 Link L1

Link L1.0
  → both sides deassert CLKREQ#
  → L1.1 / L1.2 + clock/common-mode power reduction
  → traffic or explicit request asserts CLKREQ#
  → PHY timing/handshake → L1.0 → L0

System power-off
  → PME_Turn_Off / PME_TO_Ack
  → L2/L3 Ready
  → main power removal → L2(Vaux) or L3(no power)
```

已形成正文：[D-state、ASPM、L1SS 与 PME](power/function_dstates_aspm_l1ss_and_pme.md)。

D0内部还有一条独立功耗控制链：

```text
DPA capability power table
  → Host选择Substate Control
  → Cadence DPA_INTERRUPT
  → EP client实施降频/关单元等策略
  → local firmware更新Substate Status
```

已形成正文：[Dynamic Power Allocation (DPA)](power/dynamic_power_allocation_dpa.md)。当前Cadence PF0实际选中DPA，但默认allocation array是必须由integrator替换的占位值。

slot/hot-plug还有另一条“报告需求与下发额度”的链：

```text
PF Power Budgeting Capability：报告D-state/工况/rail功耗
  ↕ firmware / power manager
Downstream Port Slot Power Value+Scale
  → Set_Slot_Power_Limit MsgD
  → EP Captured Slot Power fields
  → device client执行限制
```

详见：[Slot Power Limit 与 Power Budgeting](power/slot_power_limit_and_power_budgeting.md)。

与central-resource省电相关的双向hint见：[LTR 与 OBFF](power/ltr_and_obff_latency_power_hints.md)。

运行中插拔与突然消失的不同恢复链见：[Hot Plug与Async Removal](power/hotplug_orderly_and_async_removal.md)。

## Linux object and driver dependency

```text
SoC host controller driver
  → pci_host_bridge (pci_ops + windows + dma_ranges + IRQ domains)
  → pci_bus hierarchy
  → enumeration creates pci_dev per Function
  → pci_driver id_table match
  → probe
       → enable Function decode
       → claim + map BAR
       → set DMA mask/BME + queues
       → allocate MSI/MSI-X + request_irq
       → enable device-specific traffic
```

已形成正文：

- [Linux PCI Core、枚举与 driver model](linux/pci_core_enumeration_and_driver_model.md)
- [Function driver probe、MMIO 与生命周期](linux/function_driver_probe_mmio_and_lifecycle.md)
- [Host controller driver 与 Cadence 边界](linux/host_controller_driver_and_cadence_boundary.md)
- [Linux PCIe Debug 工具](debug/linux_pcie_debug_tools.md)

## Virtualized DMA identity dependency

```text
Requester ID (PF/VF identity)
  → IOMMU domain: IOVA → Host PA + permissions
  → optional PASID: one Function, many process address spaces
  → optional ATS: Function asks TA, caches translation in ATC
  → optional PRI: absent page triggers Page Request/Response

SR-IOV PF
  → enables multiple lightweight VFs
  → unique Routing/Requester IDs + VF BAR slices + MSI-X + FLR
  → IOMMU/VFIO assigns each VF to a System Image
```

已形成正文：

- [IOMMU、ATS、PASID 与 PRI](advanced/iommu_ats_pasid_pri.md)
- [ARI 与 SR-IOV PF/VF](advanced/ari_sriov_pf_vf.md)
- [TC、VC、QoS、PTM 与 ACS](advanced/tc_vc_qos_ptm_and_acs.md)

## 后续扩展规则

每完成一个主题，必须检查：它依赖哪些节点、又解锁哪些节点；是否同时解释 Spec、Cadence 硬件和 Linux；是否能落到一次真实 transaction。出现断链时先补依赖，不按 PDF 页码硬冲。

## X6000 跨仓实现地图

```text
Host server
 ├─ edge10_tool → HAL → ioctl/mmap
 ├─ edge.ko → pci_dev/BAR/DMA API/49 MSI-X/AER
 └─ Root Port
       ↓ Host-facing PCIe Link
A0 Endpoint
 ├─ BootROM：PHY/初始Link/BAR/临时APT/DE10
 ├─ PCS：保留Link、重写APT、DE11、多die分发
 ├─ ATF BL2：DE12、512 KiB bulk、C2C最小枚举、IRQ relay
 └─ Linux Cadence EPC/EPF：Ultra runtime APT/uDMA/MSI-X/DE14服务
       ├─ BAR0 → SRAM/system/controller/ATU/uDMA/four groups
       ├─ BAR2 → four × 16 MiB data windows
       └─ A8 RC → internal C2C Link → B0 EP
                                      → A8 capture → A0 MSI-X → Host
```

端到端依赖：

```text
Boot firmware establishes Function + Link
 → Host enumeration assigns BAR and MSI-X
 → edge.ko maps BAR and publishes userspace ABI
 → tool selects protocol using DE10..DE14
 → BAR MMIO programs Endpoint-local uDMA
 → uDMA uses Host DMA address/IOVA
 → Memory Read/CplD or Posted Memory Write
 → MSI-X vector0/notify relay reports completion
```

导航：

- [X6000综合架构](code_studies/x6000_integrated_pcie_architecture.md)
- [启动交接与所有权](code_studies/x6000_boot_handoff_and_ownership.md)
- [统一地址地图](code_studies/x6000_address_map.md)
- 六仓独立专题均位于 `code_studies/`。
