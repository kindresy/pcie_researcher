# X6000 六仓 PCIe Final Coverage Audit

审计日期：2026-08-27

结论：**当前可访问的六套工程代码已完成有效扫描、PCIe范围评估、核心路径深读、
跨仓映射和文档化；没有发现尚可由现有源码解决的重大主题缺口。** 余项需要产品
rootfs、外部Host运行环境、寄存器/TRM/PHY资料或实机证据，已全部登记在
`OPEN_QUESTIONS.md`。

## 1. 审计证据基线

| 仓库 | 固定版本 | 审计结果 |
|---|---|---|
| edge-driver | x6000 `f27ec1192d20...` | Host function driver主PCIe目录深读；cascade做差异级评估 |
| linux | x6000 `e74fd0e7746a...`，5.10.4 | PCI core接点、Cadence platform/EPC/EPF/RC、Endpoint core、最终产品DTS、MSI/AER/DMA/IOMMU配置已评估 |
| bootrom | 21个远端分支，master `9357f731414e...` | 每个tip逐分支归类，三代实现、mkfw演进、下载ABI与差异已形成矩阵 |
| pcs | x6000 `80d6faf2613d...` | Link继承、BAR/APT、DE11、下载、多die、PM/VF骨架已深读 |
| atf | x6000 `98ac1eb5984a...` | DE12 bulk、C2C RC/EP、APT、MSI relay、error/firewall、handoff已深读 |
| edge10_tool | x6000 `1daae3e45e22...` | Host命令、boot、BAR、DMA/P2P、shm、reset/debug已深读 |
| edge10_tool HAL | x6000 `aa005c36b477...` | 实际构建分支而非旧gitlink；ioctl/mmap/boot ABI已与driver对照 |

BootROM `git branch -r`（排除HEAD）实测为21个，和分支矩阵逐项相等。所有commit已用
本地`git rev-parse HEAD`复核。

## 2. 逐仓独立消化检查

### edge-driver

- probe/remove、BAR request/iomap、BME、DMA mask、MSI/MSI-X、AER已覆盖；
- char device ioctl/mmap与HAL二进制ABI已覆盖；
- uDMA descriptor、SG/coherent pool、queue/ISR/P2P已追到TLP方向；
- 49-vector和双group notify已与EP Linux/ATF relay交叉验证；
- 静态风险没有混进“功能说明”，单独标注为Risk。

### Linux

- 已确认是Cadence风格vendor实现，不是DesignWare；
- platform→EPC/EPF或RC→PCI core边界已覆盖；
- 公共Ultra DTSI与最终A/B、LM/DS产品DTS覆盖语义已区分；
- 最终DTS只更新选定APT，EPC `set_bar()`不清全表，固件保留态已写入交接模型；
- 目标defconfig关闭IOMMU已确认，外部Host IOMMU另行保留；
- configfs创建/start脚本不在六仓，已列OQ-012。

### BootROM

- 21分支全部有单独结论，不以master代表全仓；
- generated API、compact direct-register、mkfw三代实现已分开；
- PHY/Link/BAR/APT/mailbox/GO行为已追踪；
- pcie-mkfw的产品BAR、MPS与FLR/MSI-X实验已评估；
- 无法由tip静态代码解决的helper ABI与MPS冲突进入OQ-015/016。

### PCS

- 明确不是重新枚举Host-facing Link，而是继承并改写地址图；
- 九条下载APT、DE11 mailbox、多die AXI DMA分发已闭环；
- local AXI DMA与PCIe uDMA明确分开；
- PM/ASPM、Hot Reset BAR修补、SMBus group与VF骨架已评估；
- 源码自身VF IRQ TODO保留为实现状态，不写成已量产能力。

### ATF

- 512 KiB bulk协议已与Host ioctl 110对齐；
- A8对B0的Type 0 config、BAR sizing/assignment、MSE/BME和outbound已覆盖；
- B0→A8 capture→A0 outbound→Host MSI-X relay已闭环；
- firewall、misc/FIQ、hwspin与错误路径已审计；
- 长度/对齐/DMA失败继续等风险已记录；未知bit和hwspin语义进入OQ-017/018。

### edge10_tool/HAL

- 命令注册→HAL→ioctl/mmap→driver完整链已覆盖；
- DE10/11/12/14四阶段下载和多group顺序已与固件对端验证；
- xferstress九模式、bwmon、shm、P2P、hotreset的硬件含义已解释；
- 动态checkout HAL造成的构建不可复现风险已固定到实际HAL commit；
- 非对齐mmap、产品识别、group id等静态问题已记录。

## 3. 跨仓主题覆盖

| 必查主题 | 结果 | 主入口 |
|---|---|---|
| 上电→枚举→probe | Pass | BootROM专题、Linux专题、综合架构§4 |
| 启动阶段交接 | Pass | `x6000_boot_handoff_and_ownership.md` |
| Configuration/BAR/APT | Pass | Linux专题、`x6000_address_map.md` |
| Host MMIO→TLP→EP AXI | Pass | 地址地图§6、综合架构§5 |
| H2D/D2H DMA | Pass | edge-driver专题、综合架构§6–7 |
| SG/coherent/MMIO/P2P区别 | Pass | tool专题、综合架构§8 |
| MSI/MSI-X/notify | Pass | Linux/Host/ATF专题、综合架构§9 |
| A0/A8/B0 topology | Pass | ATF专题、综合架构总图 |
| DE10–DE14 boot ABI | Pass | 四固件/Host专题、交接文档 |
| Link/Hot Reset/FLR/AER | Pass | 各仓专题、综合架构§11 |
| Device/Host IOMMU边界 | Pass with external boundary | Linux专题§14.1、OQ-013 |
| BootROM全分支差异 | Pass | `bootrom_pcie_branch_matrix.md` |
| 产品A/B、LM/DS地址差异 | Pass | 地址地图§5 |
| 源码学习路线 | Pass | 综合架构§13、`LEARNING_PATH.md` Stage 12 |

## 4. “软件动作最终变成什么”反查

审计没有接受只停在API名的解释：

- `pci_iomap()`→Host VA→BAR resource→Memory TLP→BAR hit→APT→AXI；
- `dma_map_sg()`→Host DMA address/IOVA→descriptor `ext_addr`→MRd/CplD或MWr；
- `WRITE_BAR` ioctl→`writel()`→Posted MWr→uDMA register/doorbell；
- MSI-X table→EP outbound region→message-address MWr→Host IRQ；
- B0 notify→C2C MWr→A8 capture/FIQ→A0 relay MWr→Host vector；
- Config Read→A8 config outbound→B0 Type 0 request→Completion→BAR sizing/assignment。

每条链都同时说明Requester、地址域、是否有Completion和最终硬件target。

## 5. 审计中发现并修正的问题

### 5.1 公共DTSI被误当成最终地址图

初稿读取了`edge10_ultra_hsio_pcie_card.dtsi`的模板APT。审计发现六个die0产品DTS
会整项覆盖`apt-ranges`。已全部改为最终DTB视角，并新增A/B、LM/DS差异表。

### 5.2 Host controller/uDMA/REG_AXI offset混淆

最终结论：

```text
BAR0+0x400000 → region9 → local 0x58400000 AXI config起点
BAR0+0x600000 → 同一region9 → local 0x58600000 uDMA起点
BAR0+0x800000 → 产品group窗口；Host REG_AXI_CFG_ADDR经此访问远端/outbound视图
```

受影响的Linux专题、地址图、交接和综合架构已同步回修。

### 5.3 Linux不是从零重画APT

最终产品DTS未列mailbox region0，而EPC `set_bar()`也未清空全部region。已将模型改为
“Linux更新项 + 固件保留项”，并在所有权矩阵中明确。

### 5.4 Device与Host IOMMU混为一谈

设备Linux目标defconfig明确关闭IOMMU；外部Host仍可能启用IOMMU。文档已分开两端，
OQ-013只保留外部Host实机问题。

## 6. 文档与状态完整性检查

机械检查结果：

- 知识库Markdown：写审计前91篇，本文件加入后92篇；
- `code_studies/`：11篇、约3,700行；
- 本地Markdown链接：0处断链；
- Markdown fenced code block：0篇不平衡；
- BootROM远端分支：21/21进入矩阵；
- 六主仓与HAL commit：全部可复现；
- `README`、`KNOWLEDGE_MAP`、`LEARNING_PATH`、`PROGRESS`、`OPEN_QUESTIONS`已同步。

## 7. 剩余问题是否会阻止当前研究收口

不会阻止“当前代码静态研究完成”，但会阻止更强的实机/量产结论：

- OQ-012：EPF configfs部署与实际MSI-X count；
- OQ-013：外部Host IOMMU domain/IOVA；
- OQ-015：量产Boot helper ABI；
- OQ-016：MPS实写值；
- OQ-017/018：hwspin和misc error bit寄存器语义；
- OQ-003/004/009/010/011：RTL build、PHY、capability与Cadence errata类既有边界。

这些问题都无法通过继续搜索当前六仓可靠解决。每项写明了所需证据和下一动作，符合
“不确定内容不伪造”的要求。

## 8. 最终判定

本轮用户新增代码研究目标满足：分别研究六仓、BootROM全分支差异、Host工具重点、
跨仓启动路线、完整地址/DMA/MSI/error链、综合架构视图、源码学习路线、可恢复状态。

知识库不会宣称实机验证已完成，也不会宣称缺失的PHY/TRM/rootfs已被源码替代；但在
当前可读证据范围内，没有尚未落盘的重要PCIe主题。后续新资料到来时，从
`OPEN_QUESTIONS.md`相应ID和`X6000_RESEARCH_PROGRESS.md`继续即可。

