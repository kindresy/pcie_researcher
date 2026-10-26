# Slot Power Limit 与 Power Budgeting：设备想吃多少，插槽最多给多少

## 1. 先把三个相似名字拆开

| 机制 | 回答的问题 | 信息方向 |
|---|---|---|
| Power Budgeting Capability | Function在某种D-state/工况/电源轨大约需要多少功率？ | 设备报告，软件读取 |
| Slot Power Limit | 当前slot允许adapter最多消耗多少功率？ | Downstream Port通过寄存器和Message告诉Upstream Port |
| DPA | Function留在D0时，软件选择哪个动态功耗substate？ | Host写Function DPA Control，client执行 |

一句人话版：Power Budgeting像设备递菜单，Slot Power Limit像餐厅告诉你本桌最高消费额度，DPA则是在已经开席后切“省电/性能套餐”。它们互相关联，但绝不是同一个寄存器换了三套名字。

## 2. Power Budgeting Capability 报告什么

Power Budgeting是Extended Capability ID `0004h`。软件写Data Select索引，再从Data Register读一条记录。每条记录描述：

- Base Power + Data Scale；
- PM State 与 PM Sub-State；
- operation condition/type；
- power rail。

设备至少按规范要求提供适用的D0信息；可以再报告D1、D2、D3hot、sustained/maximum等不同条件。`System Allocated`表示firmware已经把该Function功耗计入系统预算：若置位，后来的power manager不应再把Capability里报告的功耗重复相加。它是“别算两遍”的标记，不是“已经给设备上电”。

Power Budgeting主要服务hot-plug和平台热设计。它不会自动降频、关Lane或限制DMA；是否允许设备启用、是否要求其降低功能，由平台policy与device driver共同落实。

## 3. Slot Power Limit 怎样跨Link送到adapter

在支持slot的Root Port/Switch Downstream Port中，firmware/software配置Slot Capabilities里的Value和Scale。Port随后发送一个`Set_Slot_Power_Limit` MsgD TLP，带1 DW payload、使用TC0、向对端Upstream Port路由。

```text
firmware / hot-plug power manager
  → Downstream Port Slot Power Limit Value + Scale
  → Set_Slot_Power_Limit MsgD（TC0）
  → Link
  → adapter Upstream Port捕获Value + Scale
  → device driver/client限制设备最大功耗
```

Value与Scale组合成瓦数；现代规范还为250/275/300 W保留特殊编码。软件不能编一个低于对应form factor最低要求的limit，也不能把这个字段当作实际实时功耗计量器。

从non-DL_Up变为DL_Up时，Downstream Port通常自动重发limit；`Auto Slot Power Limit Disable`可抑制自动发送，由software显式管理。这个重发很合理：Link刚重新建立，对端之前捕获的context未必还在。

## 4. Cadence 当前配置怎样实现

### EP / PF 报告功耗

H.1明确列出PF0 Power Budgeting Capability位于`0x160`：

- `0x160` Capability Header；
- `0x164` Data Select；
- `0x168` Data Register；
- `0x16c` Capability/System Allocated。

该core为每个PF提供四个索引：

| Index | Cadence定义 |
|---:|---|
| 0 | D0 maximum |
| 1 | D0 sustained |
| 2 | D3hot |
| 3 | D1 maximum |

这些register对Host表现为标准read-only/selector字段，但integrator可从Local Management/APB侧为每个PF定制内容。VF不得实现Power Budgeting Extended Capability，它的预算由PF/平台总体管理。

Appendix F.3给的四条默认记录都以Base Power `0xF0`、Scale `0`开头，即报告240 W量级；手册明确要求customer按应用修改。对于多数实际endpoint，这显然应当被当成**交付模板值**，不是芯片测出来的真实功耗。把默认值原样交给OS，等于设备在入职表上把期望工资写成了模板示例。

### RP 发送、EP捕获limit

Cadence在RP模式下写Slot Power Limit Value/Scale会生成`Set_Slot_Power_Limit` Message。EP模式收到该Message后，core把payload存进Device Capabilities的Captured Slot Power Limit Value/Scale；client也可在Message Interface观察它。

这形成一条清晰映射：

```text
RP config write
 → Cadence message generator
 → Set_Slot_Power_Limit MsgD
 → EP message decoder
 → Captured Slot Power Limit fields
 → EP client/driver policy
```

Controller负责传递和捕获额度，不会替device client自动把功耗压到该值以下。

## 5. Linux/固件边界

通用Linux/firmware视角中，power budget、slot hot-plug与device runtime PM分属协作链。Function driver最重要的责任不是随手改slot limit，而是：

- 在硬件真正启动高功耗功能前，确认平台允许的资源；
- 对功率不足给出可诊断失败或降级，而不是继续拉电流碰运气；
- suspend/resume、hot reset或Link重建后重新确认捕获的limit/context；
- 不把D-state、ASPM、DPA和slot power limit混作一个“power level”。

workspace没有目标kernel tree、ACPI/DT和hot-plug controller driver，所以具体policy owner与调用链保持Unknown。公开register语义能够确认，项目平台如何做预算不能凭空补齐。

## 6. Debug清单

1. Capability链中是否真的有Power Budgeting `ID=0004h`；
2. 遍历Data Select时记录原始DWORD，不要只打印“240W”；同时解码state/type/rail；
3. `System Allocated`是否导致软件重复或漏算；
4. Downstream Port的Slot Implemented、Value、Scale和auto-send policy；
5. analyzer是否看到TC0 `Set_Slot_Power_Limit` MsgD；
6. EP Captured Value/Scale是否更新；
7. client是否真的依据limit限制功能，而不只是把字段收进抽屉。

## 7. Source Reference

- PCI Express Base Specification 5.0 r1.0：§2.2.8.5、§§6.8–6.9、§7.8.1。
- Cadence PCIe Core User Guide：§4.5.3 Message handling；§§9.1.1.1.65–.68；Appendix F.3；Appendix H.1；Appendix I Slot Capabilities implementation。
- MindShare *PCI Express System Architecture*：Chapter 15，printed pp.557–566 / 约PDF pp.613–622；用作software/firmware/device职责直觉，字段以PCIe 5.0为准。
