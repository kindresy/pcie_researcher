# Open Questions

这里只记录“现有证据尚不能可靠回答”的问题。能够从现有资料解决的问题应在研究到对应章节时关闭，不能长期保留为未核实状态。

| ID | 状态 | 问题 | 当前判断 / 为什么不能确认 | 需要什么证据 | 下一步 |
|---|---|---|---|---|---|
| OQ-012 | Open | 谁在产品rootfs中创建、bind并start `pci_epf_edge`，实际广告多少个MSI-X vector？ | 六仓已确认EPC/EPF实现和Host需要49 vector，但未找到configfs部署脚本 | rootfs/部署仓、启动日志、EP config dump | 查configfs脚本并把参数加入跨仓ABI表 |
| OQ-013 | Open | 外部Host运行`edge.ko`时实际IOMMU/IOVA策略是什么？ | BL2无SMMU，X6000设备Linux defconfig也关闭IOMMU；但外部服务器kernel/平台不在六仓内 | Host boot log、`iommu_group`、IOMMU domain/debug dump | 实机记录A0 Requester ID→Host domain→IOVA→PA路径 |
| OQ-015 | Open | X6000产品rootfs实际配哪代BootROM helper driver/utility ABI？ | BootROM tip的`edge_boot.c`只有read/write，仓内utility却调用ioctl 2/3，二者不能直接配对 | rootfs、部署包、历史driver commit | 固定量产helper版本并补ABI兼容矩阵 |
| OQ-016 | Open | `pcie-mkfw`当前X6000B MPS究竟是512 B还是1024 B？ | 2025 commit message称512，tip写标准编码3通常代表1024；可能存在后续回改或非标准寄存器语义 | 构建产物反汇编、PF Device Control readback、controller register定义 | 比较固件写值与Host `lspci -vv` MPS |
| OQ-017 | Open | BL2 IRQ relay硬件spinlock是否为read-to-acquire？ | `__hwspin_lock()`只轮询bit0为0而不显式写1；缺寄存器语义无法判断互斥是否成立 | hwspinlock寄存器说明或双核并发trace | 实机争用测试并核对owner/readback |
| OQ-018 | Open | ATF `pcie_irq_misc` status bit1对应哪种精确硬件错误？ | 当前代码只把它当致命连接中止事件，寄存器bit没有名称；不能硬套成Surprise Down或CTO | X6000 PCIe wrapper寄存器手册、故障注入trace | 分别注入Link Down/CTO/UR并记录status bit |
| OQ-003 | Blocked | 哪个 Cadence IP revision/RTL build 与实际芯片一致？ | 手册发布日期和特定配置已知，但没有 IP build ID 或 SoC 配置清单 | IP release note、RTL configuration report 或芯片文档 | 与手册 2.2 参数逐项核对 |
| OQ-004 | Blocked | 独立 PHY 的寄存器、校准、equalization 和 margining 实现是什么？ | Cadence guide 只描述 PIPE 边界及 controller 行为 | PHY Databook/寄存器说明 | 建立 Controller↔PIPE↔PHY 映射 |
| OQ-005 | Blocked | `PCI Express Training 8-2004.ppt` 的实际内容是什么？ | 文件由 E-SafeNet 保护，非标准 PPT，当前工具无法读取 | 未加密 PPT/PDF 或授权打开方式 | 收到可读原件后重新索引 |
| OQ-009 | Open | 实际 SoC build 对 ATS/PASID/PRI/SR-IOV 最终启用了哪些 capability？ | manual 已证明 RTL surface/register/sideband 支持；最终 capability list 受 configurator/static strap/初始化影响 | 实芯片 config dump、configuration report、SoC TRM | 用实际 capability dump 建 build-specific matrix |
| OQ-010 | Open | Cadence uDMA interrupt enable 的 reset/写语义以哪处为准？ | §5.5.1 说 reset 后全 disabled；§9.6.1.22 表格同时出现 reset=1、`R/WOCLR` 和“write 1 enables”，表述互相牵扯 | RTL simulation、release errata 或实芯片 readback/IRQ test | 初始化时显式 mask/clear/enable并验证，收到硬件证据后关闭问题 |
| OQ-011 | Open | Link Down Reset 是否保留 Local Management Registers？ | §4.7.2 明确说 LM 保留且无需重编；§11.1.2.3 明确说 LM reset，手册内部冲突 | Cadence errata/RTL，或实机对安全非默认 LM 字段做 Link Down 前后 readback | 恢复代码先按不保证保留设计；验证当前 build 后关闭问题 |

## 已解决记录

| ID | 结论 | 证据 |
|---|---|---|
| R-001 | Base Spec 版本是 PCIe 5.0 r1.0 | Base Spec 封面，PDF p.1 |
| R-002 | Cadence guide 描述特定 Gen4 x16 Dual-Mode core，兼容 PCIe 1.1/2.1/3.0/4.0 r1.0 | Cadence 2.1，printed p.6 / PDF p.95 |
| R-003 | 中文教材为王齐 2010 年第一版，并含 Linux PCI 初始化、设备驱动、DMA/IOMMU/ATS/SR-IOV 章节 | 扫描 PDF pp.3–11 OCR；版权页和目录 |
| R-004 | Cadence inbound 由 Function+BAR/RC BAR/no-match aperture 命中；outbound 由最多 32 static regions 或 sideband descriptor 翻译，no-hit 返回 AXI DECERR | Cadence §§4.5.5–4.5.6.6；`addressing/bar_and_cadence_address_translation.md` |
| R-005 | Cadence manual 定义 ATS/PASID/PRI/SR-IOV capability/register/sideband surface，但实际 SoC enable 仍需 build/config dump | Cadence Chapters 2、3、4、9；advanced 文档 |
| R-006 | x16/x8/x4/x4 是四个逻辑 Link core 的最大宽度；`LANE_MAP_MODE=0..4` 把共计 16 条 PHY Lane 配成 x16、x8+x8、两种 x8+x4+x4 或 x4+x4+x4+x4 | Cadence `LANE_MAP_MODE` signal、§§4.9、6；`controller/cadence_multifurcation_and_link_instances.md` |
| R-008 | 每 Link APB 后分为 Function/Root Port Configuration、Local Management、AXI Configuration 和 PCIe DMA 四类空间；远端 Configuration TLP 只能进入 Function 配置空间 | Cadence Table 9.1、§§9.1–9.24；`controller/cadence_register_space_and_programming_phases.md` |
| R-009 | Link Down/Hot Reset 与 FLR 作用域不同：前者影响整 Link，后者只影响 PF/VF；AXI translation 在两者中保留，FLR 还会清目标 Function BME。Link Down 的 LM 保存行为转入 OQ-011 | Cadence §§4.7.1–4.7.4、Chapter 11；reset matrix 文档 |
| R-010 | 目标Linux为5.10.4 `x6000 @ e74fd0e7...`，X6000主PCIe实现为Cadence风格platform/EPC/EPF/RC，不是DesignWare | `code_studies/linux_x6000_pcie_stack.md`、代码仓来源索引 |
| R-011 | X6000 SoC集成的Host-facing BAR/APT、A0/A8/B0 C2C、uDMA/MSI relay和四阶段交接已由BootROM/PCS/ATF/Linux/Host六仓交叉闭合 | `code_studies/x6000_integrated_pcie_architecture.md`及其来源 |
| R-012 | OQ-001已由用户提供的Linux tree解决；原OQ-002部分解决到源码可见的clock/reset/DTS/APT/C2C层，PHY模拟细节与完整TRM保留在OQ-004 | 固定commit与六仓专题 |
