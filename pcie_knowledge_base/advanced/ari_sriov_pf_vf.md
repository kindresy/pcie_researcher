# ARI 与 SR-IOV：一条 Link 上长出很多“轻量 Function”

## 1. Multi-Function 先天只有 8 个 Function 编号

传统 Routing ID 是：

```text
[ Bus 8 bit ][ Device 5 bit ][ Function 3 bit ]
```

同一个 Device Number 下只有 Function 0–7。ARI（Alternative Routing-ID Interpretation）把低 8 bit 改解释为一个 8-bit Function Number：

```text
[ Bus 8 bit ][ Function 8 bit ]
```

同一个 ARI Device 因而最多 256 Functions，Device Number 概念在该段被消除。上游紧邻的 Downstream Port 必须支持并由 software enable ARI Forwarding，否则 extended Function 无法正确发现。

来源：Base Spec §6.13，printed/PDF pp.590–592。

## 2. SR-IOV 为什么出现

传统虚拟化让 hypervisor 截获 guest 的每次 MMIO/DMA/interrupt，成本很高。SR-IOV 让一颗物理 PCIe Device 暴露：

- **PF（Physical Function）**：完整、可信管理功能，含 SR-IOV Extended Capability；
- **VF（Virtual Function）**：轻量 Function，保留直接 data movement 所需的 queue/BAR/DMA/MSI-X 资源，可直通给 VM/System Image。

PF driver/virtualization manager 分配 device-internal resources、创建 VF；VF 在 Linux 看起来仍是独立 `pci_dev`，有唯一 Requester ID，可绑定 VF driver/VFIO、经过 IOMMU 隔离。

## 3. VF 不是纯软件影子

VF 至少要有真正可区分的：

- Configuration Space view；
- Routing/Requester ID；
- VF BAR aperture 中属于自己的 slice；
- queue/context；
- DMA/IOMMU identity；
- interrupt vector；
- FLR/reset边界。

SR-IOV 只标准化 discover/configure 外壳，内部 queue、scheduler、bandwidth/QoS、PF↔VF mailbox 仍由 device-specific design 决定。

## 4. VF BAR 怎样分配

同一 PF 的一组 VF 共享一套 VF BAR register 描述。software 做 size probing 并为 **全部 enabled VFs 的连续 aperture** 分配资源；第一个 VF 的 base 加上每 VF aperture stride，得到后续 VF 的 space。

这不是每个 VF 都有六个完全独立、随意分配的实体 BAR register。VF MSE 也在 SR-IOV control 中统一控制相关 VF Memory Space。

## 5. 创建 VF 的概念流程

```text
enumerate PF
 → discover SR-IOV capability: TotalVFs/InitialVFs/offset/stride/VF Device ID/page size
 → reserve/assign VF BAR resource
 → configure NumVFs + VF MSE + VF Enable
 → Device instantiates VF config/routing identity
 → PCI core discovers/adds VF pci_dev
 → IOMMU group/domain + MSI-X
 → bind VF driver or assign to VM
```

关闭时要先从 guests/drivers 收回 VF、停止 DMA/IRQ，再 clear enable；不是简单把 sysfs number 写 0 就可以无视业务 ownership。

## 6. Cadence 配置

Cadence core guide：

- legacy Routing ID 支持最多 8 Functions，ARI 最多 256；
- `LINK*_ARI_ENABLE` static input 选择解释；
- `LINK*_SR_IOV_ENABLE` 使能 SR-IOV，且要求 ARI enable；
- 提供 PF/VF configuration register sets、VF BAR configuration、Function power/FLR/MSI/MSI-X/ATS sidebands；
- signal widths 显示部分 Link 配置为每 PF 关联最多 8 个 VF 的接口集合，但每 Link/parameter 精确数量必须按 configuration tables/registers核对。

static inputs 必须在 MGMT reset 释放前稳定，因为会影响 capability linked-list initialization。若 SR-IOV 逻辑未在硬件构建阶段实现，或未在初始化阶段接入 capability 链，运行期软件无法仅靠写寄存器补出这项能力。

来源：Cadence §§2.1、3.x configuration signals、Chapter 9 PF/VF register sets。

## 7. Linux 边界

X6000 Linux 5.10.4 的 `struct pci_driver` 包含 PF driver 的 `sriov_configure` 等 callbacks；`struct pci_dev` 在相关配置下通过 `sriov/physfn` 关联 PF/VF，通用 SR-IOV 管理实现位于 `drivers/pci/iov.c`。

典型管理接口 `/sys/bus/pci/devices/.../sriov_numvfs` 属于 PCI core/SR-IOV framework。通用 enable 流程和 VF resource 管理可以从 `drivers/pci/iov.c` 追踪；但 X6000 实际硬件 build 是否启用 SR-IOV，以及外部 Host 的 IOMMU/VFIO 接线，仍需 capability dump、Host kernel 和部署配置确认。

## 8. 安全和 debug

- 每个 VF 是否有独立 Requester ID/IOMMU mapping；
- PF reset/link reset 是否连带所有 VF；
- VF FLR 是否真正只清该 VF queue/context；
- VF BAR slice/stride 是否正确，越界是否能碰相邻 VF；
- MSI-X table/vector 是否按 VF 隔离；
- PF mailbox 输入是否验证 VF identity/length；
- ARI forwarding 只在确认下游是 ARI Device 后打开，错误 enable 可能产生 function alias。

## 9. Source Reference

- PCI Express Base Specification 5.0 r1.0, §6.13, printed/PDF pp.590–592。
- PCI Express Base Specification 5.0 r1.0, Chapter 9, printed/PDF pp.1099–1168。
- Cadence PCIe Core User Guide, §§2.1、3.x、Chapter 9 PF/VF register blocks。
- X6000 Linux 5.10.4：`include/linux/pci.h`、`drivers/pci/iov.c`。
