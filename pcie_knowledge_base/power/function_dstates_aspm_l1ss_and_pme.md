# PCIe Power Management：D-state 管 Function，L-state 管 Link

本篇聚焦Function D-state、Link ASPM/L1SS与PME。Function在D0内部按标准功耗预算分档的机制见 [Dynamic Power Allocation (DPA)](dynamic_power_allocation_dpa.md)。

## 1. 先拆坐标轴

PCIe power management 至少有三套状态：

| 状态族 | 对象 | 例子 | 谁主要控制 |
|---|---|---|---|
| D-state | PCI Function | D0、D1、D2、D3hot、D3cold | software 经 PM Capability，外加平台 power |
| L-state | 一条相邻 Link | L0、L0s、L1/L1.0/L1.1/L1.2、L2/L3 Ready | LTSSM/DLL 协议 + ASPM/PCI-PM |
| System state | 整个平台 | S0、sleep 等 | ACPI/平台规范，非 Base Spec 本章完整定义 |

一个 EP 可有多个 Function，但只有一条上游 Link。把某 Function 写到 D3hot，不等于“LTSSM 寄存器被写成 L1”；硬件会综合同一组件所有 Function 状态、在途 traffic 和两端协议，决定 Link 怎样过渡。

## 2. Function D-state

- **D0**：正常工作、最高功能和功耗；D0 uninitialized 与 fully initialized 在配置语义上还可区分。
- **D1/D2**：可选的中间省电状态，保留更多 context、恢复快于 D3。
- **D3hot**：主电仍在、Function 大部分不可操作；Configuration Space 的有限字段仍可访问。
- **D3cold**：Function 无主电，配置访问不可用；回归需要平台恢复 power，通常相当于更彻底的重新初始化。

每个 Function 的 PM Capability（传统 capability）声明支持状态、PME 能力，并提供 PMCSR：PowerState、PME Enable、PME Status 等。软件在改变 D-state 前必须满足设备/driver quiesce 要求；不能把正在 DMA 的设备直接塞进 D3hot，然后惊讶 descriptor 只搬了一半。

来源：Base Spec 5.0 §§5.1、5.3.1、7.5.2，printed/PDF pp.439、444–449、712–716。

## 3. Link L-state

### L0

正常 active，TLP/DLLP 都可传。

### L0s

低退出延迟、较浅的 standby；TX/RX 方向可非对称。main power、refclk、PLL 保持，适合短空闲。不是所有 refclock 架构都支持。

### L1 / L1.0

更深、更高 exit latency；整个 Link 双向进入，TLP/DLLP 停止。ASPM 可在双方仍为 D0 时自治进入；PCI-PM 则可因下游所有 Function 离开 D0 而进入。

### L1.1 / L1.2

可选的 L1 PM Substates：

- L1.1 维持 Link common-mode，能进一步关 clock；
- L1.2 连 common-mode 也不要求维持，省电更深、恢复要求更严格；
- 退出由 `CLKREQ#` assertion 触发。

### L2/L3 Ready、L2、L3

L2/L3 Ready 是移除 main power/refclock 前的握手 staging point。若保留 Vaux/唤醒逻辑，power off 后抽象为 L2；完全无 power 为 L3。不要把它们简单等同于某个一直运行的 LTSSM 状态——main power 都没了，状态机通常也下班了。

来源：Base Spec §§5.1–5.2，printed/PDF pp.439–443。

## 4. ASPM 和 PCI-PM 是两条进入路径

### ASPM（Active State Power Management）

双方 Function 仍可处于 D0。software 只配置 policy/capability；Link 空闲后 hardware 根据协商 autonomously 进入 L0s/L1，traffic 出现又退出。它追求 runtime idle power。

### PCI-PM software-compatible path

软件把 Function 写入 D1/D2/D3hot，组件据此请求 Link L1；系统进一步 power down 时，Root Complex 用 PME_Turn_Off / PME_TO_Ack 与所有下游协调，再进入 L2/L3 Ready。

所以：“ASPM 开了”不代表 device 进入 D3；“runtime suspend 到 D3hot”也不等于必然使用 ASPM L1.2。

## 5. L1 entry/exit 在协议上发生什么

以 ASPM L1 为例：

```text
Link idle
 → Downstream side 计时并暂时阻止新 TLP
 → PM_Active_State_Request_L1 DLLP
 → Upstream side 检查是否可进入
    ├─ PM_Active_State_Nak Message：拒绝，恢复 L0
    └─ Ack + electrical idle sequence
 → LTSSM L1.Entry → L1.Idle

任一侧出现 traffic/explicit exit
 → electrical idle exit / FTS or Recovery
 → L0
```

进入低功耗前必须处理 ordering、outstanding/receive buffer 和 wake latency。省电不是随手关 clock；协议花了很大篇幅确保两端不会一个睡了、一个还在往门缝塞 Completion。

## 6. PME：唤醒和事件通知是两步

PME（Power Management Event）用于设备请求系统恢复/处理事件：

1. 若主 power/refclock 已关，先通过 WAKE#/Beacon 等 form-factor/platform mechanism 让资源重新可用；
2. Link 恢复后，Function 发送 `PM_PME` Message TLP 到 Root Complex；
3. Root Port/RC 记录 PME source 并通知 system software；
4. software 读取/清 PMCSR.PME Status，恢复 Function。

`PME_Turn_Off` 则是 RC 向下游广播“准备断主电”，`PME_TO_Ack` 是上游 Port 汇聚后的确认。它们和 PM_PME 名字相近、方向/目的却不同。

## 7. Cadence 实现

### 7.1 L0s/L1

Cadence 会依据各 Function/Root Port Link Control 的 ASPM enable 和 TX idle 自动进入。ASPM L1 entry timer 位于 Local Management，单位 16 ns；字段为 0 会禁止 ASPM L1 transition。

Client 可用 `CLIENT_REQ_L1_EXIT`：

- 若已在 L1，触发退出；
- 若还没进入，保持 assertion 可阻止进入，给即将发生的 local traffic/低延迟任务让路。

### 7.2 L1 PM Substates 与 CLKREQ#

Cadence 提供：

- `L1_PM_SUBSTATE_OUT` 当前子状态；
- open-drain 语义的 `CLKREQ_OUT_N` 与共享总线采样 `CLKREQ_IN_N`；
- `PHY_ENT_L1_X / PHY_ACK_L1_X` 与 PHY 协调 clock/common-mode；
- 独立、始终工作的 `PM_CLK`，因为 L1.x 中 `CORE_CLK` 可以关闭；
- `CLIENT_REQ_EXIT_L1_SUBSTATE` 供本地 client 唤醒/阻止深睡。

两端都 deassert CLKREQ# 才能关 refclock。任一侧拉低共享 CLKREQ#，clock generator 开启，controller/PHY 按 T_POWER_ON、common-mode restore 等 timing 返回 L1.0，再退出 L1。

Cadence 特别提醒：`T_POWER_ON` 或 `Common_Mode_Restore_Time` 配得比真实 PHY 小，可能让一端向尚未上电的远端驱动，最终 link down 回 Detect。这是一类“看起来像 SI，实际是 power timing 配错”的经典问题。

### 7.3 Register access 也可能把 Link 叫醒

默认在 L1 substate 访问 controller register 会自动触发退出，回 L0 后完成访问，再可能睡回去。debug bit 可禁止此行为；此时因 CORE_CLK 不可用，APB 对合法地址也会 error。轮询一个调试寄存器因此可能持续阻止深度省电——观测行为改变被观测对象，硬件世界自己的量子笑话。

### 7.4 L2/L3 entry

Cadence 收到 PME_Turn_Off 后可交给 client。若仍有 PF 在 D0，controller 不会擅自自动 Ack；client 必须确保所有 PCIe traffic quiesced，再发 PME_TO_Ack。之后 client assert `REQ_PM_TRANSITION_L23_READY` 直到 LTSSM 进入 L2，外部 power logic 才能真正关 controller。

来源：Cadence Chapter 8，printed pp.307–320 / PDF pp.396–409。

## 8. Linux driver 的职责边界

稳定的概念层：

- PCI core/PM framework 保存配置、选择 D-state、设置 wake/PME，并协调 platform runtime/system sleep；
- driver 的 runtime/system suspend callback 必须停新请求、关业务 IRQ、quiesce DMA、保存 device-specific context；
- resume 恢复 power/config/BAR/device context，再重开 queue/interrupt；
- ASPM policy 通常由 PCI core/platform 控制，普通 function driver 不应随意改 Link Control 影响整条 Link；
- `pci_set_power_state()` 只处理 PCI Function 的标准状态转换，不替驱动停止硬件 DMA。

现有X6000 Linux tree与PCS `pcie_pm.c`已补齐目标controller/固件侧的ASPM、clock、
reset和PM入口，见`code_studies/pcs_x6000_pcie_stage.md`与Linux专题。外部Host平台的
ACPI/BIOS policy和实际L1SS enable仍不在六仓内，不能从设备侧源码反推。

## 9. Debug

### 性能正常，启用省电后间歇超时

1. 禁用 ASPM/L1SS 做 A/B，不把它当最终修复；
2. 查 Link Control 两端 enable 是否一致、advertised exit latency；
3. 查 L1 substate history、CLKREQ#、PM_CLK、PHY handshake；
4. 校验 T_POWER_ON/common-mode timing 与实际 PHY/refclock generator；
5. 查 outstanding request/Completion 是否在进入前正确处理；
6. 查 client 是否在发 AXI/TLP 前 assert exit request；
7. 看是否频繁 L1→Recovery→Detect，而不是正常 L1→L0。

### runtime resume 后 BAR/DMA 不工作

1. Function 是 D3hot 还是 D3cold，配置 context 是否丢失；
2. Command MSE/BME、BAR、MSI/MSI-X 是否恢复；
3. IOMMU/DMA mappings 与 descriptor 是否仍有效；
4. Cadence Link Down Indication/FLR/reset 是否同时发生；
5. device-specific clocks/resets/FIFO 是否由 driver/platform 恢复。

## 10. Source Reference

功耗“状态”之外，平台还要处理设备需求报告、slot额度、D0内动态分配和central-resource latency hint，分别见[Slot Power Limit与Power Budgeting](slot_power_limit_and_power_budgeting.md)、[Dynamic Power Allocation](dynamic_power_allocation_dpa.md)和[LTR与OBFF](ltr_and_obff_latency_power_hints.md)。

- PCI Express Base Specification 5.0 r1.0, Chapter 5, printed/PDF pp.439–494。
- PCI Express Base Specification 5.0 r1.0, §7.5.2, printed/PDF pp.712–716。
- Cadence PCIe Core User Guide, Chapter 8, printed pp.307–320 / PDF pp.396–409。
- Cadence PCIe Core User Guide, L1 PM Substates signals, printed pp.52–59 / PDF pp.141–148。
- Cadence PCIe Core User Guide, §11.7.2.3, printed pp.1324–1325 / PDF pp.1413–1414。
