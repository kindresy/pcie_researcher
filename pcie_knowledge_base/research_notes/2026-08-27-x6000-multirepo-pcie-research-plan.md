# X6000 Multi-Repository PCIe Research Plan

> **For agentic workers:** execute inline and keep every checkpoint on disk. Do not depend on chat context; do not edit production source repositories.

**Goal:** 分别吃透 X6000 的 BootROM、PCS、ATF、Endpoint Linux PCIe driver、对应 Linux PCI subsystem 和 Host 测试工具，再把它们连接为一套可追溯的启动、控制、数据、IRQ、DMA、错误与恢复架构。

**Architecture:** 源码仓库以只读研究副本放在 `source_repos/`；知识产物继续进入 `pcie_knowledge_base/code_studies/`。每个仓库先独立建立文件/符号/寄存器/生命周期模型，BootROM额外以所有远端分支为审计单位，最后才做跨仓库mapping，避免一开始就把相似名字误认成同一职责。

**Evidence:** 每条关键结论记录 repository、branch、commit、file、symbol/line；协议结论回查本地 PCIe 5.0 Spec，controller字段回查 Cadence User Guide。使用 Confirmed/Inferred/Conflict/Unknown 标签。

## Global Constraints

- 只研究各仓库 PCIe 相关代码；Linux tree 不做无关子系统通读。
- `edge-driver`、`linux`、`pcs`、`atf`、`edge10_tool` 以 `x6000` 为主线。
- `bootrom` 枚举全部远端分支，并逐分支说明 PCIe 部分的特点和差异。
- 不修改生产源码，不创建commit，不把某分支的行为外推到其它分支。
- 文档保持初学者可读，但必须追到真实函数、寄存器和跨阶段handoff。
- 长任务状态持续写入 `X6000_RESEARCH_PROGRESS.md`、`KNOWLEDGE_MAP.md` 与 `OPEN_QUESTIONS.md`。

---

## Task 1: Source acquisition and immutable evidence index

**Produces:** 仓库路径、remote、branch、commit、分支列表、源码规模与PCIe候选文件清单。

- [ ] 验证六个 GitLab remote 可访问。
- [ ] 将各仓库克隆到 `source_repos/<repo>`，切到指定分支；BootROM获取全部remote refs。
- [ ] 保存每个仓库 `HEAD`、remote URL、最近提交和BootROM分支表。
- [ ] 用 `rg --files` 和内容关键词建立PCIe source inventory。

## Task 2: edge-driver x6000 — Endpoint Linux driver

**Produces:** `code_studies/edge_driver_x6000.md`。

- [ ] 盘点Kconfig/Makefile、probe/remove、file operations、ioctl、mmap、DMA、IRQ、BAR和reset符号。
- [ ] 重建 module load → probe → BAR/config → buffer/queue → IRQ/DMA → remove/error路径。
- [ ] 把共享结构、协议常量和ABI映射到Host tool。
- [ ] 把每个controller register访问映射到Cadence章节或标Unknown。

## Task 3: linux x6000 — only PCIe-related kernel code

**Produces:** `code_studies/linux_x6000_pcie_core_and_controller.md`。

- [ ] 盘点 `drivers/pci/`、controller/Cadence、endpoint、相关DMA/IOMMU/IRQ和headers。
- [ ] 追枚举、`pci_enable_device()`、resource、iomap、MSI/MSI-X、AER、EPC/EPF精确调用链。
- [ ] 识别vendor patch与upstream边界。
- [ ] 建立 Linux API → config/MMIO/DMA/IRQ → Cadence硬件链。

## Task 4: bootrom — all-branch PCIe comparison

**Produces:** `code_studies/bootrom_pcie_all_branches.md`。

- [ ] 枚举所有remote branches与commit关系。
- [ ] 对每个分支检索PCIe/controller/PHY/reset/strap/BAR/ATU/DMA关键词和文件。
- [ ] 选公共基线，做PCIe相关path/symbol/constant差异，而非全仓库噪声diff。
- [ ] 分支逐项说明角色、启动阶段、寄存器序列、capability/BAR配置及特殊限制。

## Task 5: PCS x6000 and ATF x6000

**Produces:** `code_studies/pcs_x6000_pcie.md` 与 `code_studies/atf_x6000_pcie.md`。

- [ ] 分别确定它们位于启动链的哪个阶段、入口和handoff数据结构。
- [ ] 追PCIe clock/reset/PHY/controller/security/memory-window相关代码。
- [ ] 明确与BootROM及Linux之间哪些状态被继承、重写或假设已完成。
- [ ] 记录EL/安全状态、电源管理或runtime service对PCIe的影响。

## Task 6: edge10_tool x6000 — Host-side test tool

**Produces:** `code_studies/edge10_tool_x6000.md`。

- [ ] 盘点命令入口、设备发现、open/ioctl/mmap/read/write、DMA和性能测试代码。
- [ ] 追每个测试命令到EP driver共享ABI、BAR offset、descriptor/doorbell和完成条件。
- [ ] 区分Host CPU MMIO、Host buffer DMA、设备DMA和MSI/MSI-X观测。
- [ ] 形成“一个命令最终生成什么PCIe transaction”的worked examples。

## Task 7: Cross-repository architecture and learning route

**Produces:** `code_studies/x6000_integrated_pcie_architecture.md`、`X6000_LEARNING_PATH.md` 与更新后的 `KNOWLEDGE_MAP.md`。

- [ ] 建立 BootROM → PCS → ATF → Linux EP driver 的启动/状态ownership时间线。
- [ ] 建立 Host tool → syscall/ABI → EP driver → controller → Link → Host memory 的控制和数据流。
- [ ] 建立Config、BAR/MMIO、DMA、MSI-X、error/reset五条端到端sequence。
- [ ] 建立共享headers/constants/register ownership矩阵和跨仓库依赖图。
- [ ] 给出按源码依赖排序的阅读路线及每阶段验收问题。

## Task 8: Final coverage audit

**Produces:** `research_notes/x6000_source_coverage_audit.md`。

- [ ] 每个PCIe候选源码文件归属一个topic或明确判定无关。
- [ ] 每个关键寄存器访问有原因、时序、reset属性和Databook locator或Unknown。
- [ ] 每个BootROM分支都有PCIe差异结论，包括“无差异”的证据。
- [ ] 检查跨仓库ABI/地址/常量是否一致，记录Conflict。
- [ ] 重跑Markdown链接、TODO、术语和AXI方向检查，并更新进度。

