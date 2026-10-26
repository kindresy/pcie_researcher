# BAR Problem：设备有 BDF，但地址资源像失踪了一样

## 1. BAR 问题有三个不同阶段

1. **需求描述**：EP BAR aperture/type/32-64 bit 是否正确。
2. **资源分配**：枚举器 sizing 后是否找到合适 CPU/PCI window 并写回 BAR。
3. **地址落地**：Host outbound window 与 EP inbound BAR→AXI translation 是否一致。

即使前两步正确而第三步错误，`lspci` 仍可能显示一个看似正常的 BAR resource。配置空间中的地址分配结果不能证明控制器地址窗口和 EP 内部译码已经正确建立。

## 2. 快速判别表

| 现象 | 高概率范围 |
|---|---|
| BAR 显示 `[disabled]` / size 异常 | EP aperture/type 或 sizing mask |
| kernel 报 `can't assign` | bridge/host aperture 不足、对齐/32-bit限制 |
| BAR 有地址，`Mem-` | Command.MSE 未开启 |
| `Mem+` 但 read 全 1/UR | host window、TLP address、BAR hit/inbound translation |
| read timeout/CTO | EP local target hang、MRd Completion path |
| write 看似成功但寄存器不变 | MWr posted，可能被错误路由/丢弃；需 readback/status |

## 3. 按顺序核对

### A. 配置空间

- Header Type 与 BAR 数量；
- 32/64-bit、prefetchable；64-bit BAR 必须成对读写；
- sizing mask 计算出的 size 是否是 2 的幂并满足对齐；
- Command Memory Space Enable；
- Linux `resource` start/end/flags 与 `lspci -vv` 是否一致。

### B. 桥和 Host 地址窗口

- Root Port/上游 bridge Memory Base/Limit 是否覆盖 BAR 的 PCI bus address；
- `/proc/iomem` 中 CPU resource 是否被 host bridge/设备 claim；
- CPU physical address 与 PCI address 是否存在固定 offset；不要默认相等；
- Cadence RP outbound Memory region 是否覆盖 CPU MMIO range并翻译到分配后的 PCI BAR address。

### C. Endpoint 落地

- TLP address 是否命中正确 Function/BAR；
- Cadence EP BAR aperture 与 host sizing 结果是否一致；
- inbound PCIe→AXI translation 是否把 BAR offset 加到预期本地 base；
- 本地 AXI target 是否响应，读错误是否被转换为 CA。

## 4. 一个地址追踪模板

```text
driver: writel(0x12345678, bar0_va + 0x100)
  VA                         = __________
  CPU physical resource     = __________
  RP outbound region        = base/size __________
  PCIe MWr address          = __________
  EP Function/BAR hit       = __________
  BAR offset                = 0x100
  EP inbound AXI address    = __________
  final register            = __________
```

任一空格填不出来，就还没有资格说“PCIe 偶现”。

## 5. 安全检查法

先只读保存 `lspci -xxxx`、sysfs `resource`、桥窗口和 ATU。不要在 driver 已绑定时用 `setpci` 重写 BAR；Linux resource tree、page mapping 和设备 decode 会瞬间不一致。测试 MMIO 前确认 DMA/IRQ 已静默，优先读无副作用 ID/version register。

## 6. Source Reference

- PCIe Base Specification 5.0 r1.0, Configuration header/BAR and bridge window rules in Chapter 7。
- Cadence PCIe Core User Guide, §§4.5.5–4.5.6.6、§§9.5/9.11/9.17/9.23、§10.3.3。
- [BAR 与 Cadence 地址翻译](../addressing/bar_and_cadence_address_translation.md)。
