# 原始资料逐章 Coverage Matrix

日期：2026-08-23

这份表回答一个很朴素、也很容易被“大总结”糊弄过去的问题：**原书每一章究竟处理到哪里了？**

状态定义：

- **D（Deep-read）**：已围绕核心主题逐节阅读，并与另一权威来源交叉；
- **E（Evaluated）**：已看目录、范围和代表性正文，判断与本项目的关系；
- **M（Mapped）**：结论已进入面向读者的知识文档；
- **B（Blocked）**：需要当前缺失资料才能确认，不用猜测填坑。

`D` 不表示逐字背诵。Specification 的正确用法是围绕行为规则、例外和实现问题反复查证，不是拿荧光笔把 1299 页都染黄——荧光笔很努力，工程问题不会因此消失。

## 1. PCI Express Base Specification 5.0 r1.0

| Chapter / 范围 | 状态 | 重点阅读或评估内容 | 已映射文档 |
|---|---|---|---|
| Ch.1 Introduction, pp.89–102 | D+M | topology、RC/RP/EP/Switch、discovery model、三层服务 | `fundamentals/overview_topology_and_layers.md` |
| Ch.2 Transaction Layer, pp.103–208 | D+M | TLP字段、routing、BE、RID/Tag、Message、Completion、ordering、VC/FC、ECRC/poison、CTO、DL_Down/DPC | `transactions/*`、`configuration/bdf_*`、`errors/*` |
| Ch.3 Data Link Layer, pp.209–244 | D+M | DL state、Feature Exchange、InitFC/Scaled FC、DLLP、LCRC/Sequence/Ack/Nak/replay | `link/data_link_ack_replay_and_flow_control.md` |
| Ch.4 Physical Logical Block, pp.245–438 | D/E+M | 分开评估2.5/5GT/s的8b/10b与8GT/s+的128b/130b；TS、width/speed、deskew、LTSSM、Recovery.Equalization、reset、margining | `link/link_lane_ltssm_and_training.md`；equalization模拟/PHY细节受缺失PHY DB限制 |
| Ch.5 Power Management, pp.439–494 | D+M | D-state、Link state、PME、ASPM、L1SS、L2/L3 Ready、aux power | `power/function_dstates_aspm_l1ss_and_pme.md` |
| Ch.6 System Architecture, pp.495–672 | D/E+M | MSI/INTx、AER/DPC、reset/FLR、Hot Plug、power budget、ACS/ARI、Atomic、DPA、TPH/LTR/OBFF、PASID/PTM/readiness | 多篇 `interrupts/`、`errors/`、`advanced/`、`power/`；低相关可选feature保留评估记录 |
| Ch.7 Software Init & Config, pp.673–1000 | D+M | ECAM/config TLP、Type0/1、BAR/bridge windows、Capabilities、MSI/AER/SR-IOV/ATS等寄存器 | `configuration/*`、`addressing/*`、`advanced/*` |
| Ch.8 Electrical Sub-Block, pp.1001–1098 | E | transmitter/receiver/channel/refclk合规范围已评估；驱动只使用其receiver detect、eye/equalization边界，不把电气测试模板改写成软件教程 | `link/link_lane_ltssm_and_training.md`、`debug/link_not_up.md`；板级定量结论B：缺CEM/PHY/原理图 |
| Ch.9 SR-IOV, pp.1099–1168 | D+M | PF/VF资源、VF BAR/RID、配置、error、interrupt、PM | `advanced/ari_sriov_pf_vf.md` |
| Ch.10 ATS, pp.1169–1210 | D+M | TA/ATC、Translated/Translation Request、invalidation、PRI、PASID与配置 | `advanced/iommu_ats_pasid_pri.md` |
| Appendices A–H, pp.1211–1276 | E/D | isochronous、symbol/scramble、request dependency、IDO、message code、protocol mux、FC/Ack latency；按所服务主题选择性深读 | Ordering、Message、Data Link文档 |
| Annex/ECN history/index | E | 用于版本差异和定位，不作为独立教学Topic | 本审计和各 Source Reference |

### 尚不能由 Base Spec 单独回答的事

Base Spec 能说“合规硬件必须做什么”，不能说本 Cadence build 选了哪些 optional capability，也不能告诉 Linux 哪个函数在这个 SoC 上写哪个私有寄存器。后两问必须分别看 configuration-specific Databook 和目标 kernel/SoC source。

## 2. Cadence PCIe Core User Guide

| Chapter / Appendix | 状态 | 项目结论 | 已映射文档 |
|---|---|---|---|
| Ch.1 Preface | E | 文档约定、适用配置和读者边界 | `SOURCE_INDEX.md` |
| Ch.2 Overview | D+M | Gen4 x16 Dual-Mode、4 Link cores、AXI/PIPE、optional features | `controller/cadence_core_architecture.md` |
| Ch.3 Signal Descriptions | D/E+M | reset/clock/strap、AXI/APB、PIPE、client/message/error/PM sideband；按数据路径审计 | controller、link、power、interrupt各专题 |
| Ch.4 Device Operation | D+M | Transaction/Data Link/Physical、inbound/outbound AXI、config、BAR/translation、errors、SRIS/margining、multifunction | 核心controller/transaction/addressing文档 |
| Ch.5 PCIe–AXI DMA | D+M | descriptor、channel、interrupt/error/residue、reset/recovery | `controller/cadence_udma_programming.md`、`dma/*` |
| Ch.6 Multifurcation | D+M | 五种静态lane map与独立Link实例 | `controller/cadence_multifurcation_and_link_instances.md` |
| Ch.7 End-to-End Protection | E/M | parity/ECC/protection作为内部错误源纳入error路径；没有把安全宣传扩展成未证实功能 | `errors/error_model_aer_and_cadence.md` |
| Ch.8 Power Management | D+M | PM sideband、L1SS、CLKREQ/PM_CLK、L23 handshake | `power/function_dstates_aspm_l1ss_and_pme.md` |
| Ch.9 Register Specification | D/E+M | 已按PF/VF/RP Config、LM、AXI Config、DMA分块；关键路径字段深读，非PCIe/保留字段不机械抄表 | `controller/cadence_register_space_and_programming_phases.md` 及各专题 |
| Ch.10 Programming Guide | D+M | config/memory/message region实例、访问时序、寄存器组合 | address/controller专题和worked example |
| Ch.11 Integration Guide | D+M | reset/clock/PHY初始化、buffer/backpressure、client integration | reset、completion-buffer、link文档 |
| App.A Debug Bus | D/M | LTSSM/protocol/replay/credit观察信号按故障层映射 | `debug/link_not_up.md`等runbook |
| App.B Error Handling | D+M | error source、status/mask/sideband/AXI response | `errors/error_model_aer_and_cadence.md` |
| App.C LTSSM Encoding | D+M | 状态编码与history register | `link/link_lane_ltssm_and_training.md` |
| App.D Skew | E/M | 作为lane deskew/板级问题边界 | link/debug文档 |
| App.E PIPE Equalization | D/E+M | controller↔PHY coefficient/preset消息接口已评估；PHY内部算法B | link文档 |
| App.F Post-delivery Config | D+M | DPA等字段默认值/客户定制；发现DPA默认表不满足规范单调性 | `power/dynamic_power_allocation_dpa.md` |
| App.G Debug Topics | D+M | LTSSM history、completion timeout、典型诊断 | `debug/*` |
| App.H Capability Linked List | D+M | generic图与H.1 build表严格区分；据此判定VC/PTM/ACS/DPC/Atomic支持情况 | capability、advanced、mapping文档 |
| App.I Hot Plug | E/M | controller surface纳入reset/error/power边界 | power/error/controller reset文档 |
| App.J FAQ | E | 已检索地址/AXI/config/reset/DMA相关问答并作为交叉检查 | 分散进入专题；不单独照录FAQ |

### 当前仍需清理的 Databook 问题

- OQ-010：uDMA interrupt enable/mask reset语义互相冲突；
- OQ-011：Link Down Reset 是否复位 Local Management寄存器，§4.7.2与§11.1.2.3冲突；
- PHY内部训练算法、SoC strap wiring、真实地址图不在这本core guide内。

详见 [OPEN_QUESTIONS.md](../OPEN_QUESTIONS.md)。

## 3. MindShare *PCI Express System Architecture*

| Chapters | 状态 | 处理方式 |
|---|---|---|
| Ch.1 | E | 历史/PCI背景，非当前规范依据 |
| Ch.2–4 | D+M | architecture、routing、TLP、Request/Completion；已作PCIe 5.0校正 |
| Ch.5–10 | D+M | ACK/replay、QoS/VC、FC、ordering、interrupt、error；已作PCIe 5.0校正 |
| Ch.11–14 | D+M | Physical、electrical、reset、LTSSM；已严格区分Gen1 8b/10b与Gen3/4 128b/130b/equalization |
| Ch.15–17 | D+M | power budget、power、hotplug；新增Power Budget、LTR/OBFF与async removal专题 |
| Ch.18 | E | CEM 1.0a add-in/connector已评估；现代定量结论受缺失当前CEM/board资料阻塞 |
| Ch.19 | D+M | topology unknown at startup、4 KiB per-Function config space、host-originated discovery；已与Ch.20/21合并评估 |
| Ch.20–21 | D+M | config mechanism、enumeration、BDF capture、CRS；已作现代校正 |
| Ch.22–23 | D+M | Type 0/1 header、BAR/bridge windows、legacy capability、Expansion ROM；已作PCIe 5.0/firmware年代校正 |
| Ch.24 | D+M | Express Capability/AER/VC/DSN/Power Budget/RCRB；已作PCIe 5.0 capability/version年代校正 |
| Appendix A | D+M | analyzer/exerciser、dual-simplex trigger、layered capture；已剔除Gen1 tool/SI定量外推 |
| Appendix B | E | 2003 markets/applications历史评估，无新规范结论 |
| Appendix C | D+M | NTB/multi-host、BAR translation、mailbox/doorbell；已新增Linux NTB映射正文 |
| Appendix D | E | PCI 2.3-era Class Code表已评估；数值须查当前PCI-SIG assignment |
| Appendix E | D+M | MRdLk/CplDLk/Unlock legacy lock；已与Base Spec 5.0 §6.5交叉并入transaction正文 |
| Index | E | 已用于反向主题漏项检索 |

详见同目录的 `mindshare_ch*_ocr_assessment.md`。

## 4. 其他输入

### 王齐《PCI Express体系结构导读》

| Chapters | 状态 | 处理方式 |
|---|---|---|
| Ch.1–3 | D/E+M | PCI legacy shared-bus背景、address domain、Host Bridge、config/BAR/NTB、DMA/cache/prefetch；并行时序只评估，稳定软件模型已映射 |
| Ch.4–5 | D+M | PCIe topology/layer/capability与Montevina address domain/ECAM；修正PCIe Capability required、Extended空链编码和旧Linux路径 |
| Ch.6–8 | D+M | TLP/routing/completion、Data Link/replay、Gen1/2 Physical、LTSSM/power；严格补Gen3+与PCIe 5.0边界 |
| Ch.9 | D+M | Credit Flow Control；与Base Spec Scaled FC及Cadence buffer/credit交叉 |
| Ch.10 | D+M | MSI/MSI-X机制；平台地址/旧APIC与Linux 2.6接口作年代校正 |
| Ch.11 | D+M | Ordering/deadlock；新增跨TC data→MSI显式同步与zero-length Read flush |
| Ch.12 | D+M | Capric FPGA Endpoint、TLP拆分/Tag/CplD重组、旧Linux driver、cache与性能；进入worked example |
| Ch.13 | D+M | IOMMU/ATS/SR-IOV/MR-IOV；纠正ATS安全归因与VF config/BAR模型 |
| Ch.14–15 | D/E+M | Linux 2.6.31 PCI初始化/driver；职责模型吸收，版本敏感调用链不冒充现代源码 |

逐章评估见同目录 `wang_qi_*.md`。

| Source | 状态 | 说明 |
|---|---|---|
| PCI Express Basics & Background | E（全45页） | 已完整提取；图解直觉由核心文档吸收，规范结论回查Base Spec |
| PCI SDMS | E（全目录/用途） | 旧PCI/SCSI安装指南，低相关，不伪装成PCIe controller资料 |
| 受保护Training PPT | B | 文件是E-SafeNet/LOCK容器；没有合法可读内容可审计 |
| Linux/SoC/PHY/CEM | B（缺输入） | 明确的项目资料缺口，不以其他平台源码偷梁换柱 |

## 5. 下一次更新条件

四份核心来源的逐章矩阵已收口。最终 Coverage Audit 还会反向从所有 Databook feature、关键寄存器和 Linux API 检查是否存在断链；Blocked 输入不以猜测填补。
