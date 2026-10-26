# PCIe Spec ↔ Cadence Databook ↔ Linux Mapping

这张表维护三种语言之间的翻译：规范概念、Cadence 硬件实现、软件动作。workspace 缺 Linux source；本机 Ubuntu 6.8 headers 只能确认公开对象/API，不能冒充特定版本实现调用链。

证据等级：**C** Confirmed，**I** Inferred，**U** Unknown/待验证。

| PCIe 概念 | Base Spec 5.0 | Cadence 模块 / 文档 | 关键硬件落点 | Linux 入口 / 最终硬件行为 | 证据 |
|---|---|---|---|---|---|
| Link / Lane | §1.2 pp.90–91；Chapter 4 | §§2.1–2.2；§4.2 | 每 Lane PIPE、deskew/scramble、frame codec | host driver 启用 clock/reset/training；LTSSM 硬件协商 width/speed | C at architecture level |
| RC / RP / EP | §§1.3.1–1.3.3 pp.92–95 | §2.1 mode options；§4.1 AXI roles | Dual-Mode strap/config、AXI target/master | host bridge 或 endpoint controller 初始化；目标 glue driver 缺失 | C/U |
| Transaction Layer | Chapter 2；§1.5.1 | §§4.4–4.5 | PNP RX FIFO、Completion RX FIFO、TX scheduler、Flow Control | MMIO/config/DMA 请求最终形成 TLP | C |
| Data Link reliability | Chapter 3；§1.5.2 | §4.3 | Sequence/LCRC、ACK/NAK/FC DLLP、Replay Buffer | 硬件自治；AER/local status/debug bus 可观测 | C |
| Physical Layer / LTSSM | Chapter 4；§1.5.3 | §4.2；Appendix A/C/G | LTSSM、training RX SM、PIPE、16-state history建议 | host/client控制 training/retrain，hardware 交换 TS1/TS2 | C/U target driver |
| Bifurcation / multiple Root Ports | topology/link rules | §§4.9、6；`LANE_MAP_MODE` | 16 PHY Lane 静态分给 1–4 个独立 core；各自训练/寄存器 | platform driver 按活动 Link 初始化并各建 host bridge/port；目标 glue 缺失 | C core / U SoC |
| Config vs private register space | Chapter 7 config model | Table 9.1；§§9.1–9.24 | APB 译码 PF/VF/RP Config、LM、AXI Config、DMA | host driver写私有空间；PCI core/Function driver经 config ops 写标准空间 | C boundary / U target offsets |
| Configuration Space | Chapter 7；§1.4 | §§4.5.5–.6；Chapter 9 | 每 Function config、RP Type0/1 generation | Linux `pci_ops` → aperture/ECAM → Config TLP | C/U target ops |
| Enumeration | §1.4 + Chapters 6/7 | RP config request path、bus routing | Config Request/Cpl、bridge bus/window | PCI core 扫 BDF、递归 bus、分 resource；精确源码缺失 | C protocol / U implementation |
| BAR | §7.5.1.2；§1.3.2.2 | §§4.5.5–.6 + PF/VF BAR config | Function+BAR inbound hit、RC BAR/no-match、translation | sizing/allocation；`pci_iomap()` → CPU MMIO → MWr/MRd | C |
| Expansion ROM / Option ROM | §7.5.1.2.4；§7.8.5；§9.3.4.1.15 | §9.2.1.1.13、§9.8.1.1.13 | 当前PF offset 0x30为Not Implemented/RO zero；通用request编码不等于ROM BAR存在 | firmware/PCI core对ROM单独建模；非`pci_iomap(BAR0)` | C absent in build / U firmware policy |
| Memory Write | Chapter 2 | §§4.5.2、4.5.6 | inbound AXI Master / outbound AXI Slave Posted path | `writel()` 或 DMA write → MWr；无普通 Cpl | C |
| Memory Read / Completion | §§2.2.9、2.3 | §§4.5.1、4.5.6 | PCIe Tag↔AXI ID、64-entry split completion table | `readl()`/DMA read → MRd → CplD | C |
| Flow Control Credits | §§2.6、3.4 | §§4.3–4.3.1 | P/NP/Cpl H/D credit、InitFC/UpdateFC、Scaled FC | hardware autonomous；status/limits用于性能 debug | C |
| MSI/MSI-X | §§6.1.4、7.7.1–.2 | §§4.5.6.6.6–.7、4.6 | capability/table + client AXI outbound MWr | `pci_alloc_irq_vectors` → address/data → EP MWr → IRQ domain | C/U target irq domain |
| DMA | Chapter 2 transaction；device-specific engine | Chapter 5 uDMA | descriptor/SRAM/AXI Slave/MRd/MWr/CplD | DMA API 给 `dma_addr_t`；client program engine + BME | C controller / U target driver |
| Error / AER | §§2.8、6.2、7.8.4 | §§4.5、4.6.2；Appendix B | status/mask/severity/header、AXI error、sideband | Root AER service → driver recovery callbacks | C/U target kernel |
| ASPM / Link PM | Chapter 5 | Chapter 8 | LTSSM、CLKREQ、PM_CLK、L1SS/PHY handshake | PCI PM/ASPM policy + driver quiesce/resume | C/U target policy |
| SRIS | relevant ECN / Chapter 4 references | §4.2.1 pp.206–207 | SKP OS scheduling、SRIS control、Link Cap 2 | firmware/driver 必须在 training 前配置；Linux ownership待定 | C/U |
| Lane Margining | PCIe 4.0 capability/rules | §4.2.2，config cap offset 0x920 | controller command checks + PIPE 4.4.1 PHY messages | diagnostic software requests margin while L0 | C/U |
| Secondary PCIe / Physical Layer 16 GT/s | §§4.2.6、7.7.3、7.9.20 | PF0/RP caps `0x300`/`0x9c0`，Chapter 9，Appendix H | per-Lane error、8/16 GT/s preset/hint、equalization phase与retimer parity status | Link diagnostics/retrain/equalization request；PHY内部调参仍需PHY DB | C controller / U PHY |
| ARI / SR-IOV | §6.13；Chapter 9 | §2.1；PF/VF regs + static enables | legacy 8 / ARI 256 IDs、VF BAR/FLR/MSI-X | PF enables VFs；unique RID feeds IOMMU/VFIO | C/U build config |
| IOMMU / ATS / PASID / PRI | Chapter 10；§7.8.8 | ATS/PASID/PRI regs, descriptors, sidebands | TA/ATC/invalidation/page request + RID/PASID | DMA API/IOMMU；optional SVA stack | C protocol/surface, U SoC TA |
| TC / VC / QoS | §§2.2.6.6、2.5、6.3、7.9.1–.2 | Appendix H；LM Error Status `Unmapped TC` | 当前配置仅 TC0/VC0；VC capability 未选中、status tied 0 | hierarchy-wide TC→VC/arb policy；当前实例无额外 VC 可配置 | C unsupported optional VC |
| PTM | §§2.2.8.10、6.22、7.9.16 | Appendix H generic `0xA20`，但 H.1 current lists 无 PTM | 当前配置没有 PTM capability/register block | Linux `pci_enable_ptm()` surface；本实例不可用 | C unsupported in build |
| ACS | §§6.12、7.7.8 | 零散 conditional fields；Appendix H current lists 无 ACS | 当前配置没有 ACS Extended Capability | Linux ACS/IOMMU-group policy依平台；本实例不可用 | C unsupported in build / U platform |
| NTB / multi-host | 非Base Spec通用NTB header；基础TLP/BAR/ordering仍适用 | 全手册未找到NTB/non-transparent feature | 当前guide仅RP/EP；uDMA Doorbell≠NTB peer doorbell | Linux `ntb_dev_ops`：MW/translation、doorbell、scratchpad/message、link | C absent / U vendor platform |
| DPA | §§6.16、7.9.12 | PF0 config `0x1C0–0x1D4`、`DPA_INTERRUPT`、Appendix F.4 | client按Control调整D0内功耗并回写Status；VF由PF代管 | 平台power policy surface；目标Linux实现缺失 | C build support / U software owner |
| Power Budget / Slot Limit | §§2.2.8.5、6.8–6.9、7.8.1 | PF0 Power Budget cap `0x160–0x16c`、RP message generator、EP captured limit、Appendix F.3/I | PF报告工况功耗；RP以TC0 MsgD下发slot额度；EP捕获后client执行 | firmware/hotplug/driver共同policy；目标平台实现缺失 | C core / U platform policy |
| LTR / OBFF | §§2.2.8.8–.9、6.18–6.19 | PF0 LTR cap `0x1b8`、LM `0x21c/0x220`、RP L1.2 threshold、Message interface、`OBFF_ENABLE` | EP上报snoop/no-snoop容忍延迟；RP下发中央资源active/idle hint | hierarchy-wide PM policy；device据hint batch DMA/IRQ | C core / U Linux policy |
| Hot Plug / async removal | §§6.7、2.9、6.2 | per-Link slot sidebands、Slot Cap/Control/Status、`HOT_PLUG_INTERRUPT_OUT`、Appendix I | RP控制slot外部逻辑并捕获sticky event；当前build无DPC capability | hotplug service增删hierarchy；driver quiesce/remove/error cleanup | C core / U target pciehp/platform |

## 已确认的关键桥接结论

### 1. TLP 分层不是概念图里的空气

Cadence Figure 4.1 直接对应 Base Spec §1.5：Transaction Layer 的 PNP/Completion/Flow Control、Link Layer 的 LCRC/Replay、Physical Layer 的 LTSSM/PIPE 都有明确硬件模块。后续寄存器意义必须挂回这些状态机，而不是孤立解释 bit。

### 2. Cadence 的 AXI Master/Slave 名称以 core 为观察者

- Link → local device：core 发 AXI transaction，所以走 **AXI Master**，典型为 Host 访问 EP BAR。
- local client → Link：client 访问 core，所以进入 **AXI Slave**，典型为 EP DMA Host Memory 或 RP 发配置/内存请求。

PCIe target/requester role 与 AXI master/slave role不是一套命名，不能把 `master` 直接当“向 PCIe 发请求”。完整 region 规则见地址翻译正文。

### 3. Completion matching 有可见硬件资源上限

Cadence §4.1 表明 Application interface 保存最多 64 个 master-side Non-Posted transactions 的状态以匹配 Completion。Base Spec 定义 Tag/Transaction ID 和 Completion 规则；两者构成“协议抽象 → outstanding table/FIFO”的直接映射。

## Linux 调用链证据边界

```text
pci_enable_device()
  → power/resource/Command register path
  → architecture/host config ops
  → RC configuration transaction

pci_iomap()
  → struct resource
  → CPU virtual I/O mapping
  → host bridge outbound window
  → Memory TLP

dma_map_single()
  → DMA mapping/IOMMU
  → DMA address
  → EP descriptor
  → EP client/optional uDMA发起本地访问
  → Cadence controller outbound AXI Slave
  → Memory TLP
```

协议与公开 API 语义已建立；内部 symbol 只在王齐教材中有 Linux 2.6-era 历史例子，不能外推到现代 6.8。补入目标 Linux tree 后要记录具体 symbol、line、commit 和 architecture-specific 分支。
