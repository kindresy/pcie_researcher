# DPA：Function 在 D0 状态下的动态功耗档位

## 1. 它解决什么问题

传统 PCI PM 的 D0/D1/D2/D3 表达 Function 整体电源状态。很多设备在 D0 工作时也有很宽的功耗范围：加速卡可以全速跑，也可以降低频率、并发或启用单元，同时仍正常响应配置与业务。

Dynamic Power Allocation（**DPA，动态功率分配**）为这种场景定义标准控制面：软件选择一个 D0 内部 power substate，设备把实际功耗限制到该档声明的瓦数以内。

```text
PCI-PM D-state       Function整体电源状态
ASPM/L1SS            Link空闲省电状态
DPA substate         Function在D0内的功耗预算
DVFS/clock gating    设备实现预算的可能手段
```

DPA不是ASPM或Link speed。规范只定义预算、控制和状态；设备究竟降频还是关闭计算单元属于实现。

## 2. 最多32个逐级降低的预算档

- `Substate 0` 是reset/default状态，表示Function可能消耗的**最大功率**；
- 后续substate的allocation必须小于或等于前一档；
- 软件可在任意档间直接跳转；
- 所有substate中都必须响应Configuration access；
- DPA只适用于D0，与D1–D3和PCI-PM Capability独立。

例如 25/20/12/5 W 四档。设备任何时刻都不能超过 `Substate Status` 所表示的预算。高→低转换时，Status在完成前保持高档；低→高时，Status必须在增加功耗前先报告高档。这样平台不会低估设备当前可能吃掉的电。

## 3. Capability字段

DPA Extended Capability ID=`0x0016`，包含：

- `Substate_Max`：最大档号，即数量减一；
- Power Allocation Scale：8-bit allocation乘 `10/1/0.1/0.01 W`；
- 两组Transition Latency Value和单位（1/10/100 ms）；
- 每档选择latency 0或1的indicator；
- Power Allocation Array；
- `Substate Control`：Host请求的目标档；
- `Substate Status`：设备当前承诺的档；
- `Substate Control Enabled`：Control是否仍驱动转换。

Control Enabled在conventional reset/FLR后由硬件置1；Host通过RW1C写1只能清掉，不能直接置回。清零后，对Control的写不再触发DPA转换，便于更高层软件协商机制接管。

软件可在一次转换未结束时再次改目标。设备完成动作后若Status与最新Control不同，必须继续转向最新目标，不能假设软件会耐心等上一笔。

## 4. Cadence当前配置的路径

Cadence Appendix H.1明确把DPA列入当前PF0 capability list，offset=`0x1C0`。它不是VC/PTM那种“通用图里有、当前实例没选”的能力。

| Offset | Register | 作用 |
|---:|---|---|
| `0x1c0` | DPA Header | ID/next pointer |
| `0x1c4` | DPA Capability | 8档、scale、latency |
| `0x1c8` | Latency Indicator | 每档选latency 0/1 |
| `0x1cc` | Control + Status | Host目标与client状态 |
| `0x1d0–0x1d4` | Allocation Array | 8档瓦数编码 |

Host写Control后，controller向client输出一次 `DPA_INTERRUPT` pulse：

```text
Host Config Write: target substate
  → Cadence DPA_INTERRUPT
  → EP client经LM读目标
  → client实施clock/power/performance动作
  → 完成后经LM更新Substate Status
```

core 只提供标准寄存器与通知握手，不会替 application logic 自动降频。若 client 忽略状态变化通知，设备虽然暴露了 DPA Capability，却不会真正执行相应的功耗转换。

VF不能实现DPA Capability；VF power allocation由PF的DPA管理。当前Cadence VF capability list也没有DPA，与Base Spec一致。

## 5. 默认值是一个必须处理的陷阱

当前Cadence defaults显示：

- `Substate_Max=7`，共8档；
- scale=0，即乘10 W；
- allocation reset依次为 `0,1,2,3,4,5,6,7`。

直接解释就是 `0,10,20,...70 W`：substate 0反而最小，后续逐步增大，与Base Spec“substate 0最大、后续不得增加”相反。Appendix F.4也明确要求customers按application修改默认DPA值。

这组数是配置占位值，不是可直接发布的合规power table。IP/SoC integrator必须提供：

- 合规且单调不增的allocation；
- 来自真实characterization的transition latency；
- 与slot/board/thermal budget一致的substate 0；
- 能兑现每档上限的client firmware。

只把数组倒序也不够；瓦数和时间都必须来自实际硬件测量。

## 6. Linux边界

X6000 Linux 5.10.4 的 `include/uapi/linux/pci_regs.h` 可确认 capability ID、offset 和 Substate_Max mask；完整 `drivers/pci/` 中也未发现通用 `pci_set_dpa_substate()` 一类的 DPA policy API。因此可以确认该内核没有通用 DPA 管理调用链；若产品使用 DPA，需要继续查平台或设备专用代码，而不能假定 PCI core 会自动选择 substate。

合理边界是platform power/thermal policy选择预算，device firmware执行转换。设备私有DVFS策略必须与DPA ownership协调；不能一边Status声称10 W，一边私有daemon把全部engine拉满。

## 7. Debug

Control写了但Status不变：检查Control Enabled、`DPA_INTERRUPT`、client目标读取、LM status回写和latency。

平台功耗超过声明：核对allocation×scale，观察升/降功耗时Status报告顺序，检查多Function合计、DPA与私有DVFS控制权，并确认Cadence默认占位表已经被替换。

常见误解：

- DPA substate不是D1/D2，只存在于D0内部；
- Cadence core不会自动关clock，client必须执行；
- Capability表通常只读，但Host可写Control选择一档；
- 当前8档默认值不可直接使用。

## 8. Source Reference

- PCI Express Base Specification 5.0：§6.16 pp.604–606；§7.9.12 pp.941–946；§9.3.7.12 VF规则。
- Cadence PCIe Core User Guide：§§9.1.1.1.85–90，printed pp.388–391 / PDF pp.477–480；Appendix F.4 Table F.5；Appendix H.1 Table H.1，printed p.1370 / PDF p.1459。
- X6000 Linux 5.10.4 `include/uapi/linux/pci_regs.h`：DPA ID/basic fields；`drivers/pci/` 全树未发现通用 DPA policy API。
