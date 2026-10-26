# 全库正文复审台账（2026-10-26）

这次复审不是重新写一遍摘要，而是检查知识库中的结论能否回到原始资料，表达是否准确，以及文章是否符合当前的语言规范。

## 复审依据

- PCI Express Base Specification Revision 5.0, Version 1.0
- Cadence PCIe Controller User Guide（工程内 `Cadence_PCIE_userdoc.pdf`）
- Linux PCI 子系统及 X6000 相关源码；源码结论以仓库、分支和路径为边界
- 工程根目录 `AGENTS.md` 中的“语言与表达风格”规范

Base Spec 和 Databook 分开核对：前者用于判断标准协议语义，后者用于判断 Cadence 控制器的接口方向、寄存器行为和实现限制。只有在两部分各自成立后，才建立 Spec、Databook、X6000 集成和 Linux 代码之间的映射。

## 范围

正文文章共 55 篇，分布在以下目录：

- `fundamentals/`、`configuration/`、`addressing/`、`transactions/`
- `link/`、`interrupts/`、`dma/`、`errors/`、`power/`、`advanced/`
- `controller/`、`linux/`、`code_studies/`、`debug/`、`examples/`

根目录的 12 份导航、映射和状态文档单独复审。`research_notes/` 是证据和研究过程记录，不按教程正文的可读性标准逐句改写，但其中被正文采用的结论仍需核验。

## 检查方法

每篇正文至少检查四项：

1. 关键协议结论能否由 Base Spec 支持；
2. Cadence 专有结论是否与 Databook 的接口、寄存器或状态机描述一致；
3. Linux 和 X6000 代码结论是否注明适用仓库、分支或证据边界；
4. 是否存在翻译腔、口号式标题、刻意比喻、过度拟人或缺少适用条件的绝对表述。

检索只能用于定位，不能代替阅读上下文。涉及方向性、时序、错误归属和软件职责的结论，需要回看原文前后段落。

## 已发现问题

| ID | 文档 | 问题 | 处理状态 |
|---|---|---|---|
| R-001 | `code_studies/bootrom_pcie_boot_flow.md` | 将 MPS 写成 RC 枚举后的“协商”；MPS 实际由配置软件依据层级能力与策略编程，不是链路自动协商 | 已修正 |
| R-002 | `debug/linux_pcie_debug_tools.md` | 缺少明确的 Source Reference，命令输出与配置空间、资源树、内核日志之间的来源关系需要补足 | 已补充 |
| R-003 | 多篇正文 | 仍有“玄学”“偷偷”“魔法数字”“一句话定位”等偏刻意表达 | 已逐篇收敛 |
| R-004 | Linux、controller、power、advanced 等多篇正文 | 仍称 workspace 没有目标 Linux source；后续加入的 X6000 Linux 5.10.4 已可闭合多条调用链 | 已按目标源码更新 |
| R-005 | `debug/memory_access_failure.md` | 把所有 Memory Read 返回都写成 CplD；成功读返回 CplD，UR/CA 等失败通常是不带数据的 Completion | 已修正 |
| R-006 | `CONFLICTS_AND_ERRATA.md` CE-001 | 最终产品 DTS 覆盖问题已经在四篇 X6000 正文修正，但冲突账本仍标 Document Fix Pending | 已改为 Resolved |

## 当前进度

- [x] 递归盘点 Markdown 文档并区分正文、导航文档和研究记录
- [x] 将 Base Spec 与 Cadence Databook 转为可检索文本
- [x] 完成全库第一轮语言风险和关键技术词扫描
- [x] 复审基础、配置、地址与事务正文
- [x] 复审链路、中断、DMA、错误和电源正文
- [x] 复审 Cadence 控制器正文
- [x] 复审 Linux、X6000 多仓库代码研究和完整实例
- [x] 复审 Debug 正文
- [x] 复审根目录导航、映射和状态文档
- [x] 执行链接、代码围栏、来源引用和遗留措辞检查

## 复审结论

- 55 篇正文均已检查来源段、关键协议风险词、Linux/硬件证据边界和语言风格；缺少来源段的 Linux Debug 工具文档已经补齐。
- MPS、MSI/MSI-X TLP 类型、Memory Write Completion、Flow Control credit 作用域、DMA address 和 Cadence AXI 方向完成专项反查。
- 早期文章中“只有 Linux headers、缺少目标源码”的表述已按 X6000 Linux 5.10.4 更新；外部 Host 环境仍与设备侧内核严格区分。
- 新增 `ARTICLE_INDEX.md`，为所有 55 篇正文提供用途概述和入口。
- 全库 94 份 Markdown、13698 行；本地 Markdown/image 链接 0 断链，代码围栏 0 个不配对，正文风格风险词检查 0 命中，内部 IP 文本 0 命中。

## 下一处准确起点

现有资料范围内的本轮复审已经完成。下一轮应从 `OPEN_QUESTIONS.md` 获取新证据：优先补产品 rootfs/configfs 脚本、PHY Databook、完整 SoC TRM、外部 Host 内核和实机 config/APT/IOMMU/AER dump。收到资料后，先重查与其直接关联的文章，再更新映射和覆盖审计，不需要从头重写知识库。
