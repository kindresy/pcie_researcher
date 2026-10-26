# 王齐第 13 章 OCR 评估：IOMMU、ATS、SR-IOV 与早期 MR-IOV

日期：2026-08-23

阅读范围：printed pp.344–362 / PDF pp.356–374，逐页 OCR。

这章写于 PCIe 2.1 左右，优点是把“设备也像一个需要页表的进程”讲得很直观；缺点是当时 PASID/PRI/SVA 生态尚未长齐，几处安全归因与 VF 配置空间描述也不能照搬。

## 1. IOMMU：最稳定的直觉

教材用两个 VM Domain、两个 EP 和相同的 DMA address 举例：

```text
EP1: RID1 + IOVA 0x1000_0000 → HPA 0x1000_0000
EP2: RID2 + IOVA 0x1000_0000 → HPA 0x5000_0000
```

同一地址数值因 Requester identity/domain 不同，可以得到不同 translation 与 permission。IOTLB 缓存 translation，page table/context 建立和失效由 VMM/OS/IOMMU driver 管理。这条模型今天仍然成立，也解释了：

- `dma_addr_t` 为什么不能当 CPU physical address；
- 每个 VF 为什么必须有唯一 Requester ID；
- 32-bit DMA mask 的设备如何通过 32-bit IOVA 访问 4 GiB 以上 HPA；
- IOMMU 同时提供 translation 和 isolation，不只是虚拟机加速器。

书中 Intel VT-d Root/Context Entry、AMD Device Table/page walk 是当年平台实例。其精确格式、page size 与 cache invalidation 必须读目标 IOMMU spec/driver，当前 workspace 没有这些输入，不移植旧表。

## 2. ATS：机制主线正确，安全解释要重画边界

教材已经覆盖：

- TA/ATPT、device ATC；
- AT=`00b` Untranslated、`01b` Translation Request、`10b` Translated；
- Translation Completion 返回 translated range/permission；
- TA 修改 mapping 后发送 Invalidation Request，设备清 ATC 并回 Completion。

这些内容已与 PCIe 5.0 Chapter 10 交叉进入 [`advanced/iommu_ats_pasid_pri.md`](../advanced/iommu_ats_pasid_pri.md)。现代版本还必须加入 PASID、PRI、context binding、page request 和更完整 invalidation/ordering 规则。

### 2.1 ATS 不是 P2P 隔离开关

教材把 ATS 的安全价值解释为防止恶意 VM 把 DMA address 指向另一个 EP BAR。这个风险场景真实，但阻断它依赖：

```text
Requester identity + IOMMU permission
        + hierarchy routing / ACS P2P control
        + OS device ownership
```

Untranslated Request 本来就不应绕过授权；某笔 P2P transaction 是否经过 TA 又取决于 topology/ACS。ATS 缓存的是 TA 授予的 translation，不是把不可信设备升级成可信设备的魔法。该校正已经进入 ATS 主文。

### 2.2 “当时没有设备”不是今天的能力结论

2010 年书中“尚无支持 ATS 的设备”只是时代照片。当前 Cadence build 的 Appendix H 明确列出 PF0 ATS/PASID/ATS-PR、VF ATS、RP ATS/ATS-PR capability surface；实际 SoC 是否启用及 client/ATC/IOMMU 是否完整仍是 OQ-009。

## 3. SR-IOV：PF/VF 直觉正确，配置空间描述需纠正

稳定部分是：PF 管理 device-wide resource，VF 是可单独枚举/绑定/IOMMU隔离的轻量 Function，减少 hypervisor 模拟 data path 的成本。

教材说“与 PF 对应的 VF 共享该配置空间，每个 VF 有独立 BAR”。更准确的现代说法是：

- 每个 VF 有自己的 Configuration Space view、Routing/Requester ID；
- PF 的 SR-IOV Extended Capability 中有一组 **VF BAR registers**，software 用它为所有 VF 统一 sizing/分配连续 aperture；
- 每个 VF 获得该 aperture 中自己的 slice，不是每个 VF 都有六个可独立任意分配的实体 VF BAR register；
- VF queue/context/MSI-X/FLR 必须在设备内部真正隔离。

这些边界见 [`advanced/ari_sriov_pf_vf.md`](../advanced/ari_sriov_pf_vf.md)。书中没有系统展开 ARI、VF FLR、现代 Linux SR-IOV/VFIO/IOMMU group，不用旧图补猜。

## 4. MR-IOV：保留体系结构启发，不宣称当前 Cadence 支持

本章后半介绍 Multi-Root Aware RP/Switch/Device 与 Virtual Hierarchy，目标是让多个 Root/System Image 共享同一 fabric，同时保持独立 hierarchy/address domain。它对理解 multi-host 的问题空间仍有价值：enumeration ownership、地址 translation、peer isolation、doorbell/mailbox、reset/error scope 都会比单 Root 更难。

但是当前 Cadence user guide 没有 MR-IOV/MRA capability 或 programming model，workspace 也没有对应 switch/device databook。知识库因此只把它映射为历史背景；可落地的多主机正文使用 [`advanced/non_transparent_bridge_and_multi_host.md`](../advanced/non_transparent_bridge_and_multi_host.md)，不把“规范里画过”写成“芯片里已经有”。

## 5. Coverage 结论

| 范围 | 状态 | 去向 |
|---|---|---|
| IOMMU/RID/domain/IOTLB | D+M | DMA/ATS文档 |
| Intel VT-d/AMD IOMMU具体表格式 | E | 历史平台实例；目标spec/driver缺失 |
| ATS translation/invalidation | D+M | ATS正文；加入PCIe 5.0校正 |
| PASID/PRI | 教材未覆盖，Base Spec D+M | ATS/PASID/PRI正文 |
| SR-IOV PF/VF | D+M | ARI/SR-IOV正文；纠正VF config/BAR |
| MR-IOV/MRA | E/M | multi-host历史背景，当前controller不宣称支持 |

## 6. Source Reference

- 王齐，《PCI Express体系结构导读》，Chapter 13，2010 年第一版，扫描 PDF pp.356–374。
- PCI Express Base Specification 5.0 r1.0, Chapters 9–10。
- Cadence *PCIe Core User Guide*, ATS/PASID/PRI/SR-IOV signals/registers and Appendix H。
