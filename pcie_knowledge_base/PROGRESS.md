# Research Progress

最后更新：2026-08-27

状态标记：`[x]` 已有效完成；`[~]` 已扫描/部分深读；`[ ]` 未完成；`[!]` 被当前资料阻塞。

## Sources

- [x] PCIe Base Specification 5.0 r1.0：Ch.1–10与Appendices/Annex已逐章评估；Transaction、Data Link、Logical PHY/LTSSM、配置、PM、error、SR-IOV、ATS等核心主题已按依赖深读并映射；Electrical定量实现保留PHY/CEM资料边界。
- [x] Cadence PCIe Core User Guide：Ch.1–11与App.A–J已逐章评估；架构、AXI、LTSSM、translation、uDMA、PM/error、Chapter 9/10/11及Appendix H已完成主题深读和反向审计；手册冲突保留OQ-010/OQ-011。
- [x] PCI Express System Architecture：1106 页已按 Chapter 1–24、Appendix A–E 与 Index 完成 OCR/章节评估；核心章节已深读并完成 PCIe 5.0 年代校正，旧 CEM、Class Code 与市场资料明确保留历史边界。
- [x] PCI Express 体系结构导读：455 页已完成目录、Chapter 1–15及书末页的OCR/章节评估；已把PCI legacy、地址域、TLP/Link、Capric FPGA、虚拟化和Linux 2.6内容分别做现代规范/Databook/Linux职责校正。
- [x] PCI Express Basics & Background：45 页文本已完整提取并评估，将作为辅助图解来源。
- [x] PCI SDMS：已评估全部目录与用途；判定为旧 PCI/SCSI 驱动安装指南，非核心 PCIe 资料，保留作低优先级历史参考。
- [!] PCI Express Training 8-2004.ppt：E-SafeNet/LOCK 保护容器，当前不可读；需要未加密原件或合法解密方法。
- [x] X6000 Linux 5.10.4 `x6000 @ e74fd0e7...`：已限定PCIe范围深读Cadence platform/EPC/EPF/RC、Endpoint core、DTS、MSI/AER与相关DMA路径。
- [x] X6000 Host edge-driver、BootROM 21分支、PCS、ATF、edge10_tool/HAL：已完成独立专题与跨仓集成，固定commit见代码仓来源索引。
- [!] PHY Databook / 完整 SoC TRM / CEM：当前仍未提供。

## Knowledge Base Foundation

- [x] `SOURCE_INDEX.md`
- [x] `README.md`
- [x] `PROGRESS.md`
- [x] `KNOWLEDGE_MAP.md` 第一版
- [x] `OPEN_QUESTIONS.md`
- [x] `LEARNING_PATH.md` 第一版
- [x] `GLOSSARY.md` 第一版
- [x] `COMMON_MISCONCEPTIONS.md` 第一版
- [x] `SPEC_DATABOOK_MAPPING.md` 第一版
- [x] 原始资料逐章 Coverage Matrix：Base Spec、Cadence、MindShare、王齐教材均已建立全章评估表；受保护PPT与缺失Linux/SoC/PHY资料明确标为Blocked
- [x] 全局写作要求：自然、成熟、严谨的工程交流口吻；修辞克制，解释充分；图示需有教学价值和来源

## Topics

- [x] PCIe Overview / 为什么比 I2C、SPI 复杂：已用“兼容软件模型 + packet fabric + OS资源管理”解释复杂度来源。
- [x] Topology / RC / RP / EP / Switch：角色、Port、Hierarchy、Switch虚拟bridge与逐跳/端到端边界已覆盖。
- [x] PCIe Layers / Link / Lane / speed / width：分层、速率/宽度和训练主线已形成独立文档。
- [x] BDF / Requester ID / hierarchy routing：已覆盖 RID/Tag、Type0/1、ARI、IOMMU/error identity 与 Linux devfn。
- [x] Configuration Space / Type 0 / Type 1：已形成配置/ECAM/路由、header与capability正文；字段细节按所属专题解释而非复制整表。
- [x] Capability / Extended Capability / ECAM：已覆盖两类链、safe walker、path prerequisite、Cadence fixed map 与 Linux API。
- [x] Enumeration：协议/硬件/Linux 架构链与 Debug 已完成；精确 Linux 扫描实现被 source 缺失阻塞。
- [x] BAR sizing / resource allocation：BAR sizing、bridge/host window、Cadence两端translation与 Debug 已完成；Linux allocator实现被源码边界阻塞。
- [x] Expansion ROM / Option ROM：已覆盖ROM BAR sizing/enable、Memory Read TLP路径、image与trust policy年代校正；确认当前Cadence PF未实现offset 0x30。
- [x] CPU / PCI / DMA address domains：已分清CPU virtual/physical、PCI address、DMA address/IOVA与Cadence inbound/outbound translation。
- [x] Cadence RP/EP address translation：已深读 4.5.5/4.5.6.6、Chapter 9 block 和 Chapter 10 Config/Memory/Message region 实例。
- [x] TLP format / routing / Byte Enable：Memory/Config/Message/Completion/Atomic/Prefix 主线已覆盖；当前 Cadence build AtomicOps 明确不支持。
- [x] Memory Read / Memory Write / Completion：已形成完整 transaction 与 Cadence AXI 转换路径。
- [x] Posted / Non-Posted / Tag / IDs：核心语义、worked example 与 outstanding 映射已完成。
- [x] MPS / MRRS / RCB / Ordering / Flow Control：已区分四类约束，覆盖 RO/IDO/No Snoop、Cadence AXI reordering 与性能排查。
- [x] TC / VC / QoS / PTM / ACS：已完成协议用途、硬件行为和 Linux surface；确认当前 Cadence build 未选中额外 VC、PTM、ACS Capability。
- [x] Data Link / ACK-NAK / Replay：已连接 DLLP、逐跳可靠性、Cadence replay buffer 和 flow-control init。
- [x] LTSSM / equalization / retrain：完整建链主线、Cadence encoding/debug/reset、Secondary PCIe/16GT/s状态与controller↔PIPE coefficient握手已完成；PHY内部analog/校准实现因缺PHY Databook单列Blocked。
- [x] INTx / MSI / MSI-X：已完成协议、Cadence EP/RP 路径、Linux API 边界和 debug 文档。
- [x] DMA / Bus Master / SG / coherency：已完成概念分层、Linux DMA API、Cadence uDMA descriptor/register/ISR/Link Down recovery；平台 cache 架构差异保留边界。
- [x] IOMMU / ATS / PASID / PRI：已完成地址身份、ATC/TA/invalidation/page request、Cadence surface 与 Linux边界。
- [x] Error / AER / DPC / CTO / UR / CA / Poison / Link Down：协议分类、containment、传播路径、AER/日志、Cadence AXI/sideband 和分层 debug 已完成。
- [x] Power / D-state / ASPM / L1SS / PME：已分清状态维度、协议路径、Cadence sideband/timing 和 Linux driver 边界。
- [x] Dynamic Power Allocation：已覆盖D0内power substate、Cadence client握手与PF/VF边界；发现默认allocation table不满足规范单调性，明确要求integrator定制。
- [x] Power Budgeting / Slot Power Limit：已分清设备报告需求、slot下发额度与DPA；覆盖Cadence PF0 capability、RP MsgD、EP captured fields及默认模板风险。
- [x] LTR / OBFF：已覆盖双向功耗hint、snoop/no-snoop service latency、hierarchy聚合、Cadence LTR生成/L1.2联动与OBFF client边界。
- [x] Hot Plug / async removal：已覆盖orderly quiesce、slot event/control、Cadence sideband/W1C IRQ和PCIe 5.0推荐DPC而弃用HPS的年代校正。
- [x] SR-IOV / ARI / FLR：ARI/PF/VF/VF BAR 及 FLR/Link Down/Hot Reset 作用域与恢复矩阵已完成。
- [x] NTB / multi-host：已覆盖transparent/non-transparent enumeration边界、Memory Window/translation、doorbell/scratchpad、Linux NTB API surface；确认当前Cadence guide未提供NTB feature。
- [x] Linux PCI core architecture：通用对象/枚举/resource语义已建立，并用X6000 5.10.4追到目标Cadence host/EPC/EPF接点。
- [x] Linux driver probe/remove and API call chains：X6000 Host `edge.ko`的enable/regions/iomap/BME/DMA/MSI/AER/ioctl路径已完成；通用正文继续保留跨版本边界。
- [x] Linux PCIe debug tools：已解释 lspci/setpci/sysfs/dmesg/iomem/IOMMU 数据来源和安全边界。
- [x] Protocol analyzer / exerciser：已形成双向同时线、分层解码、按RID/Tag/address/event触发与scope/SoC trace证据分工手册。
- [x] 分层 Debug runbooks：Link、枚举、BAR、MMIO、DMA、MSI、CTO、AER 已分别形成从现象到 TLP/AXI/Linux 证据链。

## Controller Deep Dive

- [x] Cadence core configuration：Gen4 x16 Dual-Mode；四逻辑 Link core、五种 16-Lane map；256-bit AXI；64-bit AXI address。
- [x] Cadence internal layers：AXI Application / Transaction / Link / Physical / PIPE已映射到buffer、translation、replay、LTSSM和数据路径。
- [x] Reset/clock/PHY/controller initialization sequence：Cadence §11.1.2 顺序已整理并连接 LTSSM。
- [x] RP vs EP mode selection and core configuration：mode strap、PF/Root Port config可见性和每Link独立模式已确认；目标SoC strap wiring另列资料边界。
- [x] Configuration/local-management register spaces：APB 全局译码、PF/VF/RP Config、LM、AXI Config、DMA 的访问者与作用已建立索引。
- [x] AXI Master/Slave data paths：已按 Cadence core 观察方向统一命名——inbound 为 core AXI Master，outbound 为 core AXI Slave。
- [x] PNP and Completion queues / outstanding tracking：已覆盖 two-stage Completion buffer、64 outstanding/576 worst-case CplD/128 Stage-1 capacity、ordering/AXI backpressure 与限流表。
- [x] Address translation / BAR aperture behavior：已整理 RP/EP inbound、region/sideband outbound，并用 Chapter 10 Config/Memory/Message programming example 交叉验证。
- [x] MSI/MSI-X/INTx hardware path
- [x] Integrated PCIe-AXI DMA：架构、descriptor、能力发现、channel 寄存器顺序、ISR/error/residue 与 Link Down 恢复已完成；interrupt enable 手册矛盾记 OQ-010。
- [x] LTSSM control/status and debug signals：状态编码、history shift register、debug mux 与逐状态排障已完成。
- [x] Error status/mask/W1C/recovery registers：AER与Cadence行为、四Link block、sticky/W1C、mask/severity、AXI/sideband与恢复顺序已系统解释。
- [x] Power-management registers and sideband signals：PM Capability、Link Control、L1SS/CLKREQ/PM_CLK/client exit 与 L23 handshake 已覆盖。
- [x] Bifurcation / multifurcation：已确认 wrapper/独立 core、`LANE_MAP_MODE=0..4` 精确 Lane 分配、PHY/clock 前置条件。
- [x] Register programming sequence inventory：已覆盖 strap/reset前、training前、address path、training、enumeration后、uDMA 与分类 reset recovery。

## Deep Dive Mapping

- [x] Spec ↔ Databook Mapping：协议、配置、事务、Link、PM/error、高级capability及Cadence register/module已建立系统对应，并经Appendix H/Chapter 9反向审计。
- [x] Cadence Appendix H capability反向审计：PF0/VF/RP selected链逐项映射；补齐Secondary PCIe/16 GT/s equalization寄存器，generic-only VC/PTM/NPEM等明确不宣称当前build支持。
- [x] Cadence Chapter 9反向寄存器审计：PF/VF/RP Config、LM、AXI translation、uDMA六类顶层block均已反查；重复region/Function/BAR/Link按模板消化，补齐计数器、credit、ECC、mask与equalization观测面。
- [x] Databook ↔ Linux Mapping：已落到X6000 Cadence platform/EPC/EPF、Ultra DTS APT与Host edge-driver远程uDMA编程。
- [x] Linux API call chains：目标5.10.4源码、Endpoint Framework和Host driver主链已研究；未被实际调用的全内核分支不机械展开。
- [x] CPU MMIO → outbound TLP → EP BAR → AXI/APB transaction
- [~] EP DMA → Cadence uDMA/outbound → Memory TLP → RC/IOMMU → host memory：X6000 descriptor、DMA API和TLP已闭环；设备Linux明确关闭IOMMU，实际外部Host IOMMU/域配置待runtime证据。
- [x] Endpoint interrupt → MSI-X Memory Write → Host IRQ：A0直接49-vector路径和B0→A8→A0 relay均已闭环；实际irq-remapper型号不是理解该项目ABI的必要前提。
- [x] Link/error event → controller status → IRQ → AER/recovery：X6000 controller和Host `edge.ko` recovery主链已覆盖，并记录静态风险。

## Review Gates

- [x] Stage 1：世界观/拓扑/分层 Review：BDF/RID、Link/Data Link/LTSSM和角色/层次依赖已闭合；修正 Glossary RID 定义。
- [x] Stage 2：配置/枚举/BAR Review：完成因果链、地址域、TLP、Cadence与Linux边界复核；见 research note。
- [x] Stage 3：事务/地址/DMA/中断 Review：完成 AXI 方向、posted/completion、地址域、DMA/IRQ 与 reset recovery 跨文档审计。
- [x] Stage 4：Link/Error/Power Review：三条主线已完成首轮交叉 review；equalization coefficient 与平台 PM policy 作为资料边界保留。
- [x] Stage 5：Cadence register/programming Review：Chapter 9/10/11 职责、访问、时序、reset、buffer/backpressure 已审计；新增 OQ-011 手册冲突。
- [x] Stage 6：Linux call-chain Review：X6000 5.10.4与Host edge-driver补齐目标implementation；部署configfs和实机IOMMU保留为明确开放问题。
- [x] 全库结构/链接/术语审计：77篇Markdown本地链接无断链；MPS、Flow Control、MSI TLP类型、AXI方向与年代边界风险模式复核通过；导航补齐Linux/DMA/IRQ/Link/Error与Chapter 9观测入口。
- [x] FINAL COVERAGE AUDIT：已完成输入资料、30个目标问题、Databook feature/register、软件→硬件链、结构/图示、链接/术语及24项DONE条件反查；当前证据集无重大遗漏，外部缺失资料保持Blocked。
- [x] X6000六仓 FINAL COVERAGE AUDIT：21个BootROM分支、Linux/driver/PCS/ATF/tool/HAL、最终产品DTS覆盖、启动交接、MMIO/DMA/MSI/error与源码路线已审计；0断链，外部证据缺口已进入OPEN_QUESTIONS。

## 下一步准确起点

1. 当前可读资料与X6000六仓已收口；新资料到来时从`OPEN_QUESTIONS.md`准确续跑。
2. 最高优先级新输入：部署rootfs/configfs脚本、完整SoC/PHY资料、实机config/APT/IOMMU/AER dump和Cadence errata。
