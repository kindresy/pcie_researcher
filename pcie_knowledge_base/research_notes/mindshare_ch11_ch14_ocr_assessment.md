# MindShare Chapter 11–14：Physical、Reset、LTSSM 的可用直觉与年代边界

日期：2026-08-23

## 1. 阅读范围

本轮逐页 OCR 评估 MindShare *PCI Express System Architecture*：

- Chapter 11 Physical Layer Logic；
- Chapter 12 Electrical Physical Layer；
- Chapter 13 System Reset；
- Chapter 14 Link Initialization & Training。

对应约 PDF pp.451–609。这里尤其不能“见图就抄”：全书出版于 2003 年，Chapter 11/12 的包围、K-code、comma、de-emphasis和眼图数值主要属于 Gen1 2.5 GT/s。它能解释物理层为什么需要那些模块，却不能取代 PCIe 5.0 Chapter 4/8、CEM 或目标 PHY Databook。

## 2. 仍然很好用的 Physical Layer 流水线

去掉只属于8b/10b的细节后，教材给出的硬件分解仍有教学价值：

```text
TX packet stream
 → 多Lane striping
 → 每Lane scrambling / encoding
 → serializer / differential driver
 → channel
 → receiver / CDR / deserializer
 → block/symbol alignment
 → elastic buffer + clock compensation
 → Lane-to-Lane de-skew
 → decoding / descrambling / unstriping
 → packet stream
```

这解释了几类常见现象：

- 有电气波形不等于已经获得 bit/block lock；
- 每条Lane都能收，不等于多Lane合起来不会因skew失败；
- SKP/elastic buffer解决的是独立参考时钟的频差，不是协议Flow Control；
- polarity inversion可由每Lane接收端处理，而Lane reversal涉及整条Link的lane编号关系。

### 8b/10b 和 128b/130b 必须分桌吃饭

教材的 COM/K28.5、STP/END/EDB symbol、10-bit symbol lock只适用于2.5/5.0 GT/s encoding。PCIe 5.0 §4.2.2规定8.0 GT/s及以上使用128b/130b block、Sync Header与framing token，且还有precoding/equalization规则。共同的架构问题是lock、framing、scrambling、deskew；具体在线格式完全不同。

因此 analyzer/PIPE debug 时必须先问当前速率。拿Gen1的K-code表去找Gen4 TLP，和拿UART start bit去解以太网差不多：都很认真，但方向已经离站了。

## 3. Receiver Detect：TS1之前先判断对面有没有终端

Chapter 12 Figure 12-4解释了Detect.Active的电气直觉：发送端改变共模条件，通过AC耦合电容与远端receiver termination形成的不同RC响应，判断Lane另一头是否存在接收器。这项动作发生在TS1交换前，由PHY执行。

原图已裁入 [Link Not Up runbook](../debug/link_not_up.md)。它适合说明“为什么termination/AC coupling/power会让LTSSM卡在Detect”，但图中的电容、阻抗、时序数值是旧规范语境，不能直接作为Gen4验板标准。

眼图的定性解释仍成立：水平闭合常与jitter相关，垂直闭合常与noise/attenuation相关；de-emphasis/equalization用于补偿channel loss。定量mask、test load、preset/coefficient和compliance方法必须回到目标速率的现代规范。

## 4. Reset：三种名字背后是不同的作用域与传递方式

教材区分：

- Cold Fundamental Reset：加电引起；
- Warm Fundamental Reset：不切主电源，由平台硬件触发；
- Hot Reset：在已配置Link上用TS1的Hot Reset bit向下游传播。

PERST#是平台/形态规范常用的Fundamental Reset sideband，但Base Spec并不把所有平台具体时序都定义在一个地方。Hot Reset是in-band protocol reset：通常由Downstream Port发起，只向downstream传播；Switch收到后还要传播到其下游Ports。

现代驱动还必须把FLR单独摆出来。FLR只重置一个Function，不等同于Fundamental Reset或整条Link Hot Reset。Cadence又把MAIN/MGMT/MGMT_STICKY/AXI/APB/PM/PIPE reset分成多个domain，所以“reset一下”在review里不算完整句子，至少要说清触发源、scope、哪些register/translation/outstanding state被清。

正文：[Cadence Reset 与恢复矩阵](../controller/cadence_reset_and_recovery_matrix.md)。

## 5. LTSSM：旧书保留主干，现代高速率增加关键楼层

教材给出的主干仍然正确：

```text
Detect
 → Polling（lock、polarity、TS1/TS2）
 → Configuration（Link/Lane number、width、deskew）
 → L0
```

它还用多个例子说明了Link Number、Lane Number和Lane reversal的协商。最实用的硬件直觉是：

- Detect只决定哪些Lane看见receiver；
- Polling让Lane能够可靠识别训练序列；
- Configuration才把候选Lane编成一条具有width/number的Link；
- Configuration.Idle之后才能进入正常L0；
- Data Link的Feature Exchange/InitFC还在后面，所以Physical L0不保证Config TLP已经可用。

### 必须纠正的高速率叙事

2003 年章节把Gen2/更高速率写成未来能力，并说初始训练在2.5 Gb/s完成。对现代Gen3/Gen4，正确叙事是：两端先以2.5 GT/s建立可用基线，然后根据能力与Target Link Speed经`Recovery.Speed`切速；8.0 GT/s及以上还进入`Recovery.Equalization`完成Phase 0–3的preset/coefficient过程。于是：

- 启动过程中短暂进入Recovery不是故障；
- 反复在Recovery.Equalization/Speed循环才值得追SI、preset、PHY status；
- 限速Gen1稳定只能证明基线链路工作，不能证明Gen4 channel合格；
- 旧书“Recovery不用于初始training”的表述不适用于现代多速率bring-up。

正文：[Link、Lane与LTSSM](../link/link_lane_ltssm_and_training.md)。

## 6. L0s/L1/L2的旧模型怎样使用

教材对L0s的单向性、L1双方协调、L2接近断电的直觉仍可用，但它早于L1 PM Substates。现代系统必须再加入L1.1/L1.2、CLKREQ#、Common Mode Restore、Power On和T_POWER_ON等参数。也不能把Link state与Function D-state混成一列：Function可处D0而Link因空闲进入ASPM L1，软件也可因D3hot把Link推向更深状态。

正文：[Function D-state、ASPM、L1SS与PME](../power/function_dstates_aspm_l1ss_and_pme.md)。

## 7. 对Debug文档的实际影响

- `link_not_up.md`新增Receiver Detect原始图和现代数值边界；
- LTSSM分诊继续使用Cadence当前state encoding，不使用旧书的编号；
- 物理层故障先按Detect / lock / TS / width / equalization拆段；
- 所有2003年K-code、电气限值与timer只作历史解释，不进入当前Gen4检查阈值。

## 8. Source Reference

- MindShare *PCI Express System Architecture*，Chapters 11–14，约 PDF pp.451–609；Figure 12-4为PDF p.516。
- PCI Express Base Specification Revision 5.0 Version 1.0：§§4.2.1–4.2.6、6.6、Chapter 8。
- Cadence PCIe Core User Guide：§§4.2、11.1.2、Appendices A/C/E/G。
