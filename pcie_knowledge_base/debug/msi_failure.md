# MSI / MSI-X Failure：从设备 Memory Write 到 CPU IRQ 的排查

## 1. 中断路径不是一根线

```text
device event
 → EP interrupt logic selects vector
 → MSI address/data or MSI-X table entry
 → outbound PCIe Memory Write TLP
 → RC / interrupt remapper / MSI controller
 → Linux irq_domain/vector
 → ISR
```

MSI 本质是 Posted Memory Write，没有独立的物理 `MSI#` 信号线，也没有普通 Completion。排障应沿着这笔写事务检查地址、数据和路由，而不是按传统线中断的路径寻找引脚电平。

## 2. 先判定断在哪边

```bash
lspci -vv -s <bdf>
ls /sys/bus/pci/devices/<bdf>/msi_irqs/
grep -E '<driver|irq>' /proc/interrupts
```

- capability `Enable-`：Host 尚未启用，先查 `pci_alloc_irq_vectors*()` 返回值和 fallback。
- `Enable+` 但计数不增：查 EP 是否发 MWr、address/data 是否正确。
- remapper 收到但 Linux 计数不增：查 irq-domain/vector/affinity/mask。
- ISR 进入一次后不再来：查 device source ack、vector mask/PBA、handler return。

## 3. MSI 检查

- MSI Enable、Multiple Message Enable；设备不能发送超出分配范围的 vector；
- 32/64-bit Message Address 与 Message Data 是否由 Host 正确写入 capability；
- per-vector mask/pending（若实现）；
- Cadence `MSI_ENABLE`/allocation sideband 是否反映同一配置状态；
- 事件到 vector number 的设备内部映射；
- outbound MWr 的 address/data/Requester ID。

## 4. MSI-X 检查

- MSI-X Enable=1，Function Mask=0；
- Table BIR 指向哪个 BAR，table offset/entry stride 正确；
- 目标 entry 的 Vector Control Mask=0；
- address/data 读取自 table 的正确 entry；
- masked 时事件是否置 PBA，unmask 后是否补发；
- BAR/MSE 有效，否则 Host 连 table 都配置不好；
- table/PBA 所在区域不得被 device 普通寄存器逻辑误覆盖。

Cadence 暴露 `MSIX_ENABLE`、`MSIX_MASK` 和 PF/VF 对应 sideband，也有 Function Mask set/clear 状态寄存器。它们适合区分“Host 根本没打开”与“打开了但事件生成逻辑没动作”。

## 5. INTx/MSI/MSI-X 切换陷阱

Linux 可能因 vector 分配失败回退。driver 必须检查 API 返回的实际 vector 数和模式，使用 `pci_irq_vector()` 获取 Linux IRQ，不能把 PCIe vector index 当 Linux IRQ number。启用 MSI/MSI-X 后还要正确处理 INTx disable/设备内部旧中断状态，避免双报或静默。

## 6. 用 TLP 一锤定音

若 protocol analyzer 能看到目标 MWr：

- address/data 错：Host capability programming、table fetch 或 EP vector选择问题；
- address/data 对且到达 RC：问题进入 interrupt remapping/MSI controller/Linux；
- 根本没有 MWr：查 device event、enable/mask、Function reset、Cadence outbound path；
- Link 有 replay/error：先修链路可靠性，中断只是受害者。

MSI 是 Memory Write，但是否受 Command.BME gate 要以具体 Function/controller 实现和规范规则核对；不要用“开 BME 就好了”替代证据。Cadence 对 outbound initiator 的 gate/status 应与实际 TLP 一起观察。

## 7. 必留证据

capability 原始 bytes、MSI-X table/PBA snapshot（注意敏感地址）、API 分配 vector 数、Linux IRQ mapping、device source/mask、Cadence enable/mask sideband、MWr header、remapper fault/IRQ count。

## 8. Source Reference

- PCIe Base Specification 5.0 r1.0, §§6.1.4、7.7.1–7.7.2。
- Cadence PCIe Core User Guide, §§4.5.6.6.6–4.5.6.6.7、4.6、MSI/MSI-X signal/register descriptions。
- [INTx、MSI、MSI-X 与 Cadence](../interrupts/intx_msi_msix_and_cadence.md)。
