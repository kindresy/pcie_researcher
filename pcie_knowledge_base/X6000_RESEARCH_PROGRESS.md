# X6000 PCIe Multi-Repository Research Progress

最后更新：2026-08-27

状态：`[x]`完成，`[~]`进行中，`[ ]`未开始，`[!]`受证据/访问阻塞。

## Source acquisition

- [x] edge-driver `x6000` @ `f27ec119...`
- [x] linux `x6000` @ `e74fd0e7...`（仅PCIe相关范围）
- [x] bootrom全21个远端分支，`master` @ `9357f731...`
- [x] pcs `x6000` @ `80d6faf2...`
- [x] atf `x6000` @ `98ac1eb5...`
- [x] edge10_tool `x6000` @ `1daae3e4...`

## Per-repository studies

- [x] edge-driver Host侧EP功能驱动（主`pcie/`；cascade已完成差异级评估）
- [x] Linux X6000 controller/EPC/EPF/RC/Endpoint core、最终产品DTS、MSI与relay；设备defconfig关闭IOMMU已确认
- [x] BootROM all-branch PCIe comparison（21个分支、三代实现、tip源码聚类、mkfw演进）
- [x] PCS PCIe boot stage
- [x] ATF PCIe boot stage
- [x] Host edge10_tool（含构建实际使用的HAL/x6000）

## Cross-repository studies

- [x] Boot-stage ownership/handoff
- [x] Host tool ↔ EP driver ABI
- [x] Config/BAR/MMIO path
- [x] DMA path
- [x] MSI/MSI-X path
- [x] Link/error/reset recovery path
- [x] Integrated architecture view
- [x] Source learning path
- [x] Final source coverage audit

## Exact next starting point

当前六仓静态研究已收口。新资料到来时从`OPEN_QUESTIONS.md`对应ID继续：优先顺序为
产品rootfs/configfs、外部Host IOMMU/runtime dump、PCIe wrapper/PHY/TRM与实机故障注入。

## Completed reports

- `code_studies/edge_driver_x6000.md`：Host驱动角色、probe/remove、boot mailbox、MMIO、uDMA、notify IRQ、MSI/AER、cascade差异和源码审计风险。
- `code_studies/linux_x6000_pcie_stack.md`：X6000 Cadence EPC/EPF/RC、DTS APT、MSI-X、notify、跨die relay与Host闭环。
- `code_studies/bootrom_pcie_boot_flow.md`：BootROM上电/PHY/LTSSM、三阶段配置、BAR0 mailbox、下载协议、X6000 mkfw与Linux交接假设。
- `code_studies/bootrom_pcie_branch_matrix.md`：21个远端分支逐一结论、三代实现、blob聚类、mkfw时间线与静态审计。
- `code_studies/pcs_x6000_pcie_stage.md`：PCS保持Link、重画BAR0/APT、DE11下载、多die分发、PM/ASPM与实验性VF/FLR路径。
- `code_studies/atf_x6000_pcie_stage.md`：BL2 bulk下载、DE12、C2C RC最小枚举、跨die APT、MSI-X relay和错误路径。
- `code_studies/edge10_tool_x6000_host.md`：Host命令面、HAL实际版本、ioctl ABI、四阶段下载、BAR/MMIO、DMA/P2P、Hot Reset与静态审计。
- `code_studies/x6000_boot_handoff_and_ownership.md`：四启动阶段的Link/BAR/APT/uDMA/MSI/ready flag所有权交接。
- `code_studies/x6000_address_map.md`：PCS、Linux runtime、C2C的统一地址域与transaction实例。
- `code_studies/x6000_integrated_pcie_architecture.md`：Host→A0→A8/B0总体架构、MMIO/DMA/MSI-X/error路径和七阶段源码路线。
- `research_notes/x6000_multirepo_final_coverage_audit.md`：六仓、21分支、跨仓主题、地址修正、链接与开放问题的最终审计。
