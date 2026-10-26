# PCIe Researcher

这是一套面向芯片底层驱动工程师的 PCIe 学习与源码研究知识库。默认读者熟悉 C、MMIO、IRQ、DMA、Cache 和 AXI，但还没有系统学习 PCIe。

知识库从 PCIe 的拓扑、配置空间、BAR 和 TLP 开始，逐步进入 LTSSM、DMA、MSI/MSI-X、AER、IOMMU、Cadence 控制器实现，以及 X6000 的 BootROM、PCS、ATF、Device Linux、Host 驱动和测试工具。

建议从以下入口开始：

- [知识库首页](pcie_knowledge_base/README.md)
- [学习路线](pcie_knowledge_base/LEARNING_PATH.md)
- [知识地图](pcie_knowledge_base/KNOWLEDGE_MAP.md)
- [术语表](pcie_knowledge_base/GLOSSARY.md)
- [X6000综合PCIe架构](pcie_knowledge_base/code_studies/x6000_integrated_pcie_architecture.md)
- [研究进度](pcie_knowledge_base/PROGRESS.md)
- [开放问题](pcie_knowledge_base/OPEN_QUESTIONS.md)

## 资料边界

仓库保存的是经过整理和脱敏的知识文档，不包含内部源码镜像、原始 PCIe/Cadence PDF、受限培训资料或内网主机地址。关键结论尽量保留规范章节、PDF页码、源码仓名称、分支和固定 commit，方便具备相应资料权限的读者回溯证据。

当前研究基线和缺失资料见[资料索引](pcie_knowledge_base/SOURCE_INDEX.md)。静态分析结论不等于实机验证结果；需要硬件证据的问题集中维护在[开放问题](pcie_knowledge_base/OPEN_QUESTIONS.md)。
