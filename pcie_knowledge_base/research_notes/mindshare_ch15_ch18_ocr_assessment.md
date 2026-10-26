# MindShare Chapter 15–18：Power、Hot Plug 与 Add-in Card 评估

日期：2026-08-23

## 1. 阅读范围

逐页OCR评估：

- Chapter 15 Power Budgeting；
- Chapter 16 Power Management；
- Chapter 17 Hot Plug；
- Chapter 18 Add-in Cards and Connectors。

约PDF pp.613–759。前三章的角色/时序直觉可大量保留；Chapter 18引用CEM 1.0a，只能作为板级信号入门，所有pin、电流、时序和form factor结论需由当前CEM/板级资料重新确认。

## 2. Chapter 15：报告需求、分配系统预算、下发slot limit

教材把firmware、OS power manager、port、device driver和adapter职责画在一张图中，恰好能解释为什么功率信息不是“读一个W数就完成”：

- Function用Power Budgeting Capability报告不同D-state/工况/rail需求；
- firmware可用System Allocated说明已经计入预算；
- Downstream Port用Slot Power Limit Value/Scale规定slot额度；
- `Set_Slot_Power_Limit` Message把额度送到adapter；
- driver/client真正限制设备功能。

这些机制在PCIe 5.0 §§6.8–6.9仍存在。当前Cadence PF0还实际选择了`0x160` Power Budgeting Capability，并实现RP message/EP capture。由此新增[Slot Power Limit与Power Budgeting](../power/slot_power_limit_and_power_budgeting.md)。

## 3. Chapter 16：最大的价值是把三种状态维度分开

书中清楚区分Function D-state、Link power state和system ACPI state，并解释：ASPM由hardware在software enable/policy之后根据空闲进入；software把所有Functions置non-D0则会触发PCI-PM Link路径；L2/L3 Ready还需要PME_Turn_Off/PME_TO_Ack握手，避免切电时丢掉in-band PME。

这些直觉仍成立，且与Cadence Chapter 8的sideband/PM_CLK/L23 handshake吻合。

年代校正：

- Windows 98/XP、WDM miniport和ACPI 2.0调用关系只作历史例子，不映射到Linux；
- 早期ASPM capability组合、L0s“required”等陈述不能替代PCIe 5.0；
- 书中没有L1.1/L1.2、LTR、OBFF和现代Clock PM完整模型；
- Exit latency的路径预算不能只靠Endpoint Acceptable Latency，需要现代LTR/CLKREQ#/L1SS参数共同判断。

对应正文：[D-state/ASPM/L1SS/PME](../power/function_dstates_aspm_l1ss_and_pme.md)与新增[LTR/OBFF](../power/ltr_and_obff_latency_power_hints.md)。

## 4. Chapter 17：orderly hot-remove叙事很好，surprise remove结论过时

书中安全移除顺序非常适合作为driver checklist：request→OS验证→driver quiesce→disable Link→off power→indicator允许拔卡→释放resource。插入则上电/clock/PERST#→training/InitFC→enumeration/resource→bind driver。

但其“PRSNT触点先断，因此surprise removal有足够时间让软件关电”的说法不能保留。PCIe 5.0 §6.7.6把async removal定义为OS可能没有预警的移除/containment disable，并说明CTO、Bad TLP/DLLP、Surprise Down等是预期副作用。PCIe 5.0还明确deprecated HPS用于async hot-plug，推荐DPC。

当前Cadence build没有DPC Capability，所以平台能否可靠支持async removal不能只看它有Hot Plug sideband。新增[Hot Plug与Async Removal](../power/hotplug_orderly_and_async_removal.md)，记录了这一实现缺口。

## 5. Chapter 18：知道有哪些线，但别拿2003年数值验今天的板

仍有用的架构认识：

- Lane有PET/PER差分对，connector把x1/x4/x8/x16做成机械长度；
- PRSNT1#/PRSNT2#是out-of-band card presence，不等于PHY receiver detect；
- PERST#承载平台Fundamental Reset；
- REFCLK架构、SSC与common/separate clock选择会影响controller/PHY配置；
- WAKE#、3.3Vaux支持掉主电后的唤醒；
- SMBus/JTAG是辅助管理/测试接口，不在TLP数据路径。

必须标作历史的内容：CEM 1.0a pin table、电源电流、25/60W等级、100ms PERST示例、SIOM/NEWCARD等当年“under development”的form factor。当前workspace没有CEM、原理图和PHY Databook，所以知识库只用它建立检查分类，不输出所谓当前合规阈值。

## 6. Coverage结果

| Chapter | 状态 | 落点 |
|---|---|---|
| 15 | Deep-read + mapped | Power Budget/Slot Limit专题 |
| 16 | Deep-read + mapped | PM/L1SS与LTR/OBFF专题 |
| 17 | Deep-read + mapped | Hot Plug专题，完成HPS/DPC年代校正 |
| 18 | Evaluated | board signal分类；现代定量结论被缺失CEM/PHY/原理图阻塞 |

## 7. Source Reference

- MindShare *PCI Express System Architecture*，Chapters 15–18，约PDF pp.613–759。
- PCI Express Base Specification 5.0 r1.0：Ch.5、§§6.7–6.9、6.18–6.19。
- Cadence PCIe Core User Guide：Chapter 8、Message handling、Power Budget/LTR registers、Hot Plug sidebands与Appendix I。
