# Cadence Appendix H 反向 Capability Audit

日期：2026-08-23

目的不是再抄一遍 capability 菜单，而是从 Databook 的 configuration-specific 链表反查：每个被选中的硬件 surface 是否有解释去处，generic 图里出现但本 build 未选中的项目有没有被误宣称支持。

![Cadence PF capability linked list 原图](../assets/source_figures/cadence-capability-chain-pdf1458.png)

> 图源：Cadence *PCIe Core User Guide*, Appendix H, Figure H.1，PDF p.1458 / 文档页1369。Figure H.1 是可裁剪的 generic chain；最终判断必须看 Tables H.1–H.4。

## 1. PF0 selected capability

| Capability / offset | 当前处理 | 读者入口 |
|---|---|---|
| PM `0x80` | D+M | D-state/PME |
| MSI `0x90`、MSI-X `0xb0` | D+M | interrupt专题 |
| PCIe `0xc0` | D+M | capability、MPS/MRRS、Link、error/power |
| AER `0x100` | D+M | error/AER |
| ARI `0x140` | D+M；受 `ARI_ENABLE` strap | ARI/SR-IOV |
| DSN `0x150` | E/M | Glossary与MindShare评估；识别信息不是trust root |
| Power Budget `0x160` | D+M | slot/power budgeting |
| Resizable BAR `0x180` | D+M；LM bit可旁路 | BAR/addressing |
| LTR `0x1b8` | D+M | LTR/OBFF |
| DPA `0x1c0` | D+M | Dynamic Power Allocation |
| SR-IOV `0x200` | D+M；受 `SR_IOV_ENABLE` strap | PF/VF/ARI |
| TPH Requester `0x274` | D+M | TLP Prefix/TPH |
| Secondary PCIe `0x300` | D+M；Gen3/4时存在 | Link/equalization，已在本轮补寄存器落点 |
| VSEC `0x400` | E/M | capability safe walker；内容必须按Cadence/vendor文档解释 |
| PASID `0x440` | D+M | IOMMU/ATS/PASID/PRI |
| ATS `0x5c0` | D+M | 同上 |
| ATS Page Request `0x640` | D+M | 同上 |
| L1 PM Substates `0x900` | D+M | ASPM/L1SS |
| Data Link Feature `0x910` | D+M；Gen4时存在 | DL Feature Exchange/Scaled FC |
| Lane Margining `0x920` | D+M；Gen4时存在 | Link与debug/PHY边界 |
| Physical Layer 16 GT/s `0x9c0` | D+M；Gen4时存在 | equalization phase/retimer parity/preset |

`VSEC` 不值得单独写一篇“神秘寄存器”文档：Extended Capability header 只告诉 walker 它是 vendor-specific，具体语义必须来自本 vendor/build 的寄存器说明。通用 Linux driver 应按 ID/version/length安全发现，不凭 `0x400` 硬编码所有 Cadence 世代。

## 2. VF 与 PF 不对称

Tables H.2/H.3 的 VF0/VFn 只列：PM、MSI、MSI-X、PCIe、AER、conditional ARI、TPH、ATS。

因此：

- VF 有独立 config view，但不是 PF capability 的缩小复印件；
- PF0 的 PASID/PRI/LTR/DPA/SR-IOV management structure 不自动出现在 VF；
- 本表只证明该 guide configuration 的 surface，实际 VF enable/数量/strap 仍需目标 build dump。

精确角色矩阵已进入 ATS/PASID/PRI 正文。

## 3. RP selected capability

Table H.4 列出 PM、MSI/MSI-X、PCIe、AER、DSN、conditional Resizable BAR、TPH、Secondary PCIe、ATS、ATS Page Request、L1SS、Data Link Feature、Lane Margining、Physical Layer 16 GT/s。

这说明同一个 dual-mode core 切到 RP 后仍暴露大量 Port/translation/link capability，但没有 PF0 的 PASID、DPA、Power Budget reporting 和 SR-IOV管理结构。Slot Power Limit Message/Hot Plug 是另一条 RP职责，不能因为 RP 表里没 Power Budget Capability 就说它不管slot功率。

## 4. Generic图出现、H.1表未选中的项目

Figure H.1 还画了 Virtual Channel `0x4c0`、Multi-Function PCIe `0x480`、PTM `0xa20`、NPEM `0xa30` 等节点；Tables H.1–H.4 没把它们列入本 configuration。

处理规则：

- **VC/MFVC**：协议概念已评估，但当前 build 不宣称额外 VC capability；VC0/TC基础仍存在；
- **PTM**：协议与 Linux surface已评估，当前 build不宣称 capability；
- **NPEM**（Native PCIe Enclosure Management）：仅记录为可选 enclosure management capability；当前输入没有被选中实现，也没有 chassis/LED/client资料，不独立成文；
- generic pointer 图不能压过 configuration-specific table。星号、strap旁路和 dynamic next pointer 都在提醒你：先 dump 真实链，再谈功能。

## 5. Audit 结论与剩余边界

- Appendix H selected capability 已逐项进入正文或明确评估为低相关/未选中；
- 新补 gap：Secondary PCIe + Physical Layer 16 GT/s 的 software-visible equalization/lane-error寄存器；
- 没有发现应单独成文却完全缺失的当前-build capability；
- OQ-009 仍不能关闭：这本 guide 的“configuration-specific”不等于用户目标 SoC 的最终 RTL configuration/strap/readback。需要真实 config dump 或 SoC integration report。

## 6. Source Reference

- Cadence *PCIe Core User Guide*, Appendix H, Figure H.1 and Tables H.1–H.4, PDF pp.1458–1462 / 文档页1369–1373。
- Cadence Chapter 9 corresponding capability register blocks。
- PCI Express Base Specification 5.0 r1.0, Chapters 7、9、10。
