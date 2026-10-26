# Source Index

更新日期：2026-08-27（Asia/Shanghai）

## 证据边界

最初输入为下列7个资料文件；2026-08-27又补入六个X6000工程仓库及工具HAL。
资料与代码均已完成文件级扫描和PCIe范围评估；章节/主题级进度见
[PROGRESS.md](PROGRESS.md)，代码固定版本和规模见
[X6000代码仓来源索引](code_studies/SOURCE_REPOSITORY_INDEX.md)。

| 文件 | 类型 / 权威性 | 版本 / 日期 | 页数 / 规模 | 主要内容 | 与当前项目关系 / 可用定位方式 |
|---|---|---|---:|---|---|
| `PCI_Express_Base_Specification_Revision5.0r1.0.pdf` | PCI-SIG Base Specification；第一优先级协议权威来源 | Revision 5.0 Version 1.0，2019-05-22 | 1299 页；10.6 MiB | PCIe 体系结构、Transaction/Data Link/Physical Layer、配置、电源、错误、Capabilities 与高级特性 | 定义标准行为；文本可搜索；printed page 与 PDF page 基本一致，可按 section/page/table/figure 回溯 |
| `Cadence_PCIE_userdoc.pdf` | Cadence PCIe Core User Guide；第一优先级控制器 Databook | 发布 2019-02-20；文件修改 2021-07-14；文档未给出独立 revision 字段 | 1480 页；6.9 MiB | 特定配置的 Gen4 x16 Dual-Mode PCIe core；4 个 Link、AXI、PIPE、协议层、DMA、多功能、功耗、寄存器、编程与集成 | 当前唯一控制器实现资料；文本可搜索；正文 printed page = PDF page - 89（已用多个章节起点验证） |
| `PCI.Express.System.Architecture.pdf` | MindShare 体系结构教材；辅助解释来源 | Ravi Budruk、Don Anderson、Tom Shanley；Second Printing，2003-12；ISBN 0-321-15630-7 | 1106 页；13.6 MiB | 从 PCI 演进到 PCIe，事务、路由、配置、链路、电源、错误、热插拔等 | 适合教学和例子，但年代早于当前 Spec；内嵌字体映射损坏，`pdftotext` 为乱码，需逐页 OCR/截图；按 printed page + PDF page 定位 |
| `PCI_Express体系结构导读_王齐.pdf` | 中文体系结构与 Linux 教材；辅助解释和历史源码线索 | 王齐，机械工业出版社，2010-03 第 1 版第 1 次印刷；ISBN 978-7-111-29822-9 | 455 页；66.4 MiB | PCI/PCIe、地址域、TLP、链路、MSI/MSI-X、Ordering、IOMMU/ATS/SR-IOV、实例卡及旧版 Linux PCI 初始化 | 很贴近目标读者，但为扫描 PDF，需 `chi_sim+eng` OCR；Linux 代码年代较老，必须与目标内核版本重新验证 |
| `PCI_Express_Basics_Background.pdf` | PCI-SIG Technology Seminar / Synopsys 培训；辅助教学来源 | Richard Solomon，2014 | 45 页；1.9 MiB | PCIe 背景、配置、MSI/MSI-X、split transaction、拓扑、PIO/DMA、TLP/DLLP、Flow Control、ACK/NAK | 图示直观，适合作为初学者桥梁；不是规范替代品；文本可搜索，按 slide/PDF page 定位 |
| `PCI SDMS.pdf` | LSI Logic PCI Storage Device Management System 用户指南；外围历史资料 | SDMS 4.0 User's Guide，Version 12，2001-09；DB15-000099-04 | 274 页；1.6 MiB | PCI SCSI 主机适配器在旧 Windows/DOS/UNIX/Linux 上的驱动安装与配置 | 不是 PCIe 规范或控制器手册；可用于历史 PCI 设备/驱动背景，核心知识低优先级；PDF 禁止复制但当前工具可提取文本 |
| `PCI Express Training 8-2004.ppt` | 名称看似培训幻灯片，实际为受保护容器 | 文件名日期 2004-08；内部 revision 未知 | 2.7 MiB；幻灯片数未知 | 内容未知 | 文件头为 `E-SafeNet`/`LOCK`，不是标准 OLE PowerPoint；LibreOffice、7-Zip、strings 均无法恢复内容。状态：**Unknown / 需要原始未加密文件或授权解密方式** |

## X6000 工程代码

| 仓库 | 分支/版本 | PCIe研究内容 | 结果入口 |
|---|---|---|---|
| edge-driver | x6000 `f27ec119...` | 外部Host `edge.ko`、BAR/uDMA/MSI/AER/ioctl | `code_studies/edge_driver_x6000.md` |
| linux | x6000 `e74fd0e7...`，5.10.4 | Cadence platform/EPC/EPF/RC、PCI core接点、最终DTS | `code_studies/linux_x6000_pcie_stack.md` |
| bootrom | 全21远端分支 | PHY/Link/BAR/APT/下载与分支演进 | BootROM主流程与分支矩阵 |
| pcs | x6000 `80d6faf2...` | DE11、APT、多die、PM/VF | `code_studies/pcs_x6000_pcie_stage.md` |
| atf | x6000 `98ac1eb5...` | DE12 bulk、C2C、MSI relay、error/handoff | `code_studies/atf_x6000_pcie_stage.md` |
| edge10_tool + HAL | x6000 `1daae3e4...` + `aa005c36...` | Host boot/BAR/DMA/P2P/debug与ABI | `code_studies/edge10_tool_x6000_host.md` |

## 当前资料所覆盖的 PCIe 版本

- 标准基线：PCI Express Base Specification 5.0 r1.0。
- 当前 Cadence 实现：遵从 PCIe 1.1、2.1、3.0 和 4.0 r1.0；手册描述的是 Gen4 x16 Dual-Mode 配置，不是 Gen5 控制器。
- MindShare 教材：2003 年早期 PCIe 时代，适合解释兼容软件模型与基本事务，但不能作为 Gen4/Gen5 新特性的最终依据。
- 王齐教材：2010 年，包含当时的 ATS/SR-IOV 与 Linux 实现视角；任何版本敏感结论都要回到 PCIe 5.0 Spec 和实际内核代码。

## 明确缺失的资料

- PCIe CEM Specification；
- Cadence PHY Databook / PIPE PHY 集成手册；
- 完整目标 SoC TRM、时钟/复位/电源域寄存器说明（源码/DTS已补入实现视图）；
- SoC 地址映射与 IOMMU/SMMU 手册；
- RTL配置清单与当前芯片IP build ID（工程代码已提供寄存器header，但不能替代RTL配置）；
- 板级原理图、参考时钟/PERST#/CLKREQ# 连接与示波器/协议分析仪记录。

这些缺口不妨碍当前代码静态研究收口，但会限制实机/量产结论。相关问题统一维护在
[OPEN_QUESTIONS.md](OPEN_QUESTIONS.md)。

## 扫描与提取说明

- 可搜索 PDF 已用 `pdfinfo`、`pdftotext -layout` 和章节关键词建立文本索引。
- 扫描件使用 `pdftoppm` + Tesseract `chi_sim+eng` 分批 OCR；OCR 只用于检索和辅助理解，关键数字仍需回看原页。
- 图、表、字体映射或 OCR 存疑时，必须查看原始 PDF 页面图像。
- 不把文件名当证据：例如 `PCI SDMS.pdf` 的“PCI”并不意味着它是 PCIe 资料，`.ppt` 扩展名也没有让受保护容器突然变成 PowerPoint。
