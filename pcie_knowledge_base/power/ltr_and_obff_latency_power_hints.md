# LTR 与 OBFF：设备延迟需求与平台活动提示

## 1. 两种方向相反的功耗提示

PCIe平台想让内存控制器、互连和CPU多睡一会儿，却不能把对延迟敏感的网卡/存储饿到timeout。LTR与OBFF从两边交换信息：

| 机制 | 方向 | 说的是什么 |
|---|---|---|
| LTR | Endpoint → Root Complex | “我的Memory traffic最多容忍这段service latency” |
| OBFF | Root Complex → Endpoint | “现在中央资源本来就醒着，集中做DMA/IRQ的增量功耗较低” |

它们都是power/performance policy输入，不是traffic许可协议。LTR不是Completion Timeout，OBFF也不是Bus Master Enable。

## 2. LTR 报的是容忍上限，不是测量结果

Latency Tolerance Reporting把snoop和no-snoop traffic分开报告。每个16-bit字段包含：

- Requirement：这一类traffic是否有要求；
- 10-bit Value；
- 3-bit Scale：1 ns、32 ns、1024 ns……逐级放大。

Requirement=1且Value=0表示“任何delay都会影响我，请给最佳service”；Requirement=0表示没有这一类要求，此时Value/Scale忽略。LTR并不是设备用秒表测出RC延迟，而是设备根据buffer深度、实时deadline和功能需求声明可接受上限。

规范定义的service latency尤其值得驱动工程师记住：

- Non-Posted Request：从request最后部分发出，到第一个Completion开始返回；
- Posted traffic被credit卡住：从耗尽credit的前一包发完，到能解锁下一包的FC DLLP开始返回；
- 若使用Clock PM，还要把设备拉`CLKREQ#`到有效clock的时间算进平台总预算。

所以LTR会间接影响平台是否敢进入更深省电状态，却不直接命令Link进入L1，也不修改MRRS/MPS/Completion Timeout。

## 3. LTR 是整条hierarchy的承诺

软件只有在Root Complex与路径上所有Switch都支持LTR时才可给Endpoint enable。推荐从靠近Root Port的组件向下启用；disabled/unsupported Downstream Port收到LTR Message要报UR。

Multi-Function Device把各Function要求合并，取最严格（最小）的snoop/no-snoop值。Switch还要扣除自身引入的latency后再向上报告。DL_Down会使旧值失效；Function转到non-D0或LTR被disable时，若此前报告过有效requirement，还需按规则发一条Requirement全清的更新，别让平台永远以为一个已经睡着的设备仍在催单。

Root Complex并非绝对必须满足请求值，但规范强烈建议最坏service latency不要超过它。LTR是协商信息，不是硬实时合同。

## 4. Cadence LTR 的完整硬件路径

当前PF0 configuration-specific capability list明确包含：

- `0x1b8` LTR Extended Capability Header，ID `0018h`；
- `0x1bc` Max Snoop / Max No-Snoop Latency，约束Function最多能请求多严格/宽松的值。

EP client从Local Management编程：

- `0x21c` Snoop/No-Snoop Value、Scale与Requirement；
- `0x220` minimum interval与`Send LTR Message`。

`Send`在消息真正发出前保持1，client必须等它清零再改下一笔；默认minimum interval `0xFA µs = 250 µs`，与“500 µs内通常不超过两条”的规范建议相配。Link可能正在L0s/L1，编interval还需包含exit latency。

Cadence还可以在LTR enable变化或所有Functions离开D0时自动发送清除requirement的更新，避免client漏做状态收尾。

```text
EP client latency policy
 → LM 0x21c / 0x220
 → Cadence LTR Message generator
 → upstream Message TLP
 → RP Cadence receiver
 → 捕获/聚合latency
 → 与L1.2 threshold比较
 → platform power policy
```

Cadence RP收到LTR后会内部捕获，并与L1 PM Substates Control 1中的LTR L1.2 threshold比较。于是“设备能等多久”会参与“平台是否敢进L1.2”的决策，这正是协议语义落到真实state machine的地方。

## 5. OBFF：平台给Endpoint的批处理提示

Optimized Buffer Flush/Fill告诉Endpoint：CPU、memory或central interconnect正因其他工作处于active window，此时发bus-master DMA或interrupt的额外唤醒成本较低。

Message模式有三种code：

- `CPU Active`：所有device action窗口；
- `OBFF`：system memory path适合bus-master read/write；
- `Idle`：平台正处低功耗空闲。

**它只是hint。** Function在BME/IRQ已允许时仍可在窗口外发traffic；只是最好buffer/batch到合适窗口。Endpoint也不能假定窗口至少持续多久。否则一个晚到的DMA burst会让“省电优化”变成新的数据丢失条件。

OBFF可经WAKE#或Message传递。Message只向downstream、point-to-point routing；Variation A在Link处于Tx_L0s/L1时丢弃通知，Variation B会先唤到L0再发送。平台必须确认整条拓扑能把所选形式送到Endpoint，才可enable。

## 6. Cadence OBFF 实现

Cadence PCIe Capability的Device Capabilities 2报告OBFF支持，Device Control 2选择disabled、Message Variation A/B或WAKE#。`LINKx_OBFF_ENABLE[1:0]`把软件选择暴露给client。

RP生成/转发OBFF；EP收到Message时：

- Message A/B enabled：转发到client Message Interface，header stripe中的code位于`[123:120]`；
- 未enable：按UR处理；
- client依据CPU Active/OBFF/Idle决定是否flush/fill/batch，但Controller不会替业务DMA engine自动排队。

这条边界非常重要：Capability只说明“通信机制存在”，设备固件/driver若从不根据hint改变行为，功能合规却不会省下一滴电。

## 7. Linux/驱动应该怎样使用这个认知

普通Function driver通常不应自行开启整条hierarchy的LTR/OBFF；PCI core、firmware和platform PM policy需要先确认路径能力。device-specific driver/firmware更关心：

- 当前工作负载变化时，是否需要更新LTR requirement；
- 更新为更严格值要提前于相关traffic；放宽值应跟在旧需求最后一笔之后；
- non-D0、FLR、DL_Down/Up后是否重建/清除报告；
- 收到OBFF窗口时只优化batch，不改变正确性和deadline；
- DMA completion/interrupt ordering仍按普通PCIe与DMA API规则处理。

X6000 Linux 5.10.4 可用于核对通用 PCI capability 解析与 PM 框架，但外部 Host 的 firmware、平台电源策略和 central resource 实现不在当前源码范围内，因此 LTR/OBFF 的实际 policy owner 与运行期配置仍保持 Unknown。

## 8. Debug

- LTR Message收到UR：检查路径每一跳support/enable以及启用顺序；
- L1.2后DMA尾延迟异常：比较LTR值、CLKREQ#有效时间、L1 exit与RC service latency总和；
- LTR更新不发：检查`0x220.Send`是否仍pending、minimum interval、DL_Up与enable；
- 多Function值异常：检查是否取最小要求并在Function状态变化后重新conglomerate；
- OBFF无功耗收益：先证明EP client真的消费了hint并batch traffic；
- OBFF报UR：核对Device Control 2选择的Message/WAKE#形式和Cadence `OBFF_ENABLE`。

## 9. Source Reference

- PCI Express Base Specification 5.0 r1.0：§§2.2.8.8–.9、6.18–6.19、相关Device Capabilities/Control 2与LTR Extended Capability。
- Cadence PCIe Core User Guide：Message Interface/Message handling；PF0 LTR `0x1b8–0x1bc`；LM `0x21c–0x220`；`LINKx_OBFF_ENABLE`；Appendix H.1。
