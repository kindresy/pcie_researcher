# MindShare Chapter 22–23：PCI-compatible registers与Expansion ROM OCR评估

日期：2026-08-23

## 1. 阅读范围

对`PCI.Express.System.Architecture.pdf` PDF pp.827–948附近逐页OCR，评估：

- Chapter 22 PCI Compatible Configuration Registers；
- Chapter 23 Expansion ROMs。

OCR cache：`/tmp/pcie1_arch_scan_cfg/page-0827..0849.txt`和`/tmp/pcie1_arch_scan_cfg2/page-0851..0948.txt`。Chapter 22长达约100页，细讲Type 0/1 header、BAR、bridge window、legacy capability、VPD等；它是很好的原理教材，但定量和字段定义必须回查PCIe 5.0 Ch.7。

## 2. Chapter 22给知识库带来的稳定结论

### 2.1 Function identity不只是Vendor/Device ID

OS通常结合Vendor ID、Device ID、Revision ID、Class Code、Subsystem Vendor ID和Subsystem ID选driver。Subsystem ID解决的是“同一颗第三方controller silicon被不同板卡厂商做成不同产品”的区分问题。这支持了知识库中driver matching不只能讲`vendor/device`两个数的结论。

### 2.2 Capability list是链表，不是固定数组

Status.Capabilities List表明前256 B内有legacy capability list，Capabilities Pointer指向第一个dword-aligned structure，各structure的Next Pointer继续串联。这和4 KiB扩展空间从`0x100`开始的Extended Capability header/next pointer不是同一条链。

教材的Capability ID表只是当时快照，不可以变成现代驱动里的硬编码长度或“必定按这个顺序出现”假设。

### 2.3 BAR sizing是在问decoder，bridge window是在为整棵子树开门

Type 0 BAR描述Function自己要哪几段I/O或Memory Space；Type 1 Memory/Prefetchable/I/O Base–Limit registers则决定bridge允许哪些下游地址范围穿过。因此一个Endpoint BAR即使已分配，上游bridge window没覆盖它时，Memory Request仍然走不到它。

Prefetchable属性不是“这是普通RAM”的别名，而是对read side effect、丢弃未消费prefetched data、write merging等语义的承诺。把有read side effect的control/status register标成Prefetchable，结果可不只是“慢一点”。

### 2.4 Command register的几个开关不要混为一谈

- I/O Space Enable控制I/O decoder；
- Memory Space Enable控制Memory BAR/ROM等Memory Space decoder；
- Bus Master Enable控制Function是否可作为Requester主动发送Memory/I/O Request；
- Interrupt Disable是legacy INTx的控制，不等于关闭MSI/MSI-X；
- SERR/Parity等名称保留了PCI历史痕迹，在PCIe中对应Poisoned TLP和error message/reporting行为，不是真的总线奇偶校验线。

### 2.5 Type 1的Primary/Secondary侧status是bridge边界视角

bridge拥有面向primary bus的Status和面向secondary bus的Secondary Status/Bridge Control语义。它们有大量为PCI/PCI-X兼容保留的bit，PCIe 5.0的主要错误诊断应联合Device Status、AER、Root Error Status以及DPC，不能只盯着老Status字段。

## 3. Chapter 23的有效知识

Chapter 23完整解释了ROM BAR sizing、临时分配地址、decoder enable、`0xAA55`签名、`PCIR`data structure、多code image和Code Type选择。它最有价值的硬件结论是：ROM是一个可选Memory Space target，读ROM仍是Memory Read Request/Completion with Data，不是一种神秘的“BIOS transaction”。

正文已新增 [Expansion ROM / Option ROM](../configuration/expansion_rom_and_option_rom.md)，并与Base Spec 5.0和Cadence交叉验证。

## 4. 必须加上的现代校正

### 4.1 不要把旧BIOS流程当成现代平台通则

`0xC0000–0xDFFFF`的shadow区、x86入口参数、实模式interrupt table hook和Open Firmware FCode是教材的历史平台语境。PCIe Base Spec 5.0把ROM contents的定义交给PCI Firmware Specification；当代UEFI/Secure Boot平台的image选择、authentication和execution policy不能由这本2003教材代替。

### 4.2 ROM BAR的低位已不只有Enable

PCIe 5.0在bits 3:1定义可选Expansion ROM Validation Status，bits 7:4定义Validation Details，可区分valid/trusted/untrusted/warning/failed/in-progress。2003年图把bits 10:1全部标成Reserved，已经过时。

### 4.3 Cadence当前配置根本没实现ROM BAR

Cadence User Guide §9.2.1.1.13和§9.8.1.1.13均把offset `0x30`标为Not Implemented/RO zero。因此教材的ROM流程是标准机制学习，不是当前core的可用feature。

### 4.4 正常探测不等于可以冒险执行

Vendor/Device/Class匹配、`0xAA55`和`PCIR`结构正常，只能说image格式看起来可识别，不能证明代码可信。对Option ROM的执行应纳入platform firmware和security policy。

## 5. Source Reference

- MindShare *PCI Express System Architecture*：Ch.22 printed pp.769–869左右，PDF pp.827–926左右；Ch.23 printed pp.871–892左右，PDF pp.927–948。
- PCI Express Base Specification 5.0 r1.0：§7.5.1 Type 0/1 common header；§7.5.1.2.1 BAR；§7.5.1.2.4 Expansion ROM BAR；§7.8 Extended Capabilities。
- Cadence PCIe Core User Guide：§9.2.1.1、§9.8.1.1 PF configuration header registers。
- 关联正文：[Capability discovery](../configuration/capability_discovery_and_safe_walking.md)、[BAR与Cadence地址转换](../addressing/bar_and_cadence_address_translation.md)、[Error/AER](../errors/error_model_aer_and_cadence.md)。
