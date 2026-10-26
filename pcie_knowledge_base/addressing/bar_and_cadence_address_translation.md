# BAR 与 Cadence 地址翻译：三种地址别再挤一张身份证

## 1. BAR 解决的是“设备需要多大窗口、放在哪里”

Endpoint 内部有寄存器或存储资源，Host 需要通过 PCIe Memory Space 访问它们。但不同机器的地址布局不同，EP 不能任性地把自己钉死在 `0x8000_0000`。

BAR（Base Address Register）提供两部分信息：

- **硬件声明**：窗口是 Memory 还是 I/O、32/64-bit、是否 Prefetchable、需要多大空间；
- **系统分配**：firmware/OS 规划全局地址图后，把基址写回 BAR。

所以 BAR 像“房屋规格 + 最终门牌”，不是房屋里的那块 SRAM。

## 2. Type 0 BAR 的编码

Type 0 header 在 offset `0x10–0x24` 有六个 DWORD 位置。

### Memory BAR

- bit 0 = 0；
- bits 2:1：`00b` 为 32-bit，`10b` 为 64-bit；其余保留；
- bit 3：Prefetchable；
- 高位保存 base address；低属性位只读。

64-bit BAR 使用连续两个 DWORD：低 DWORD 含属性和地址低位，高 DWORD 含地址高 32 bit。因此“六个位置”不等于“必然有六个独立 BAR”。

### I/O BAR

- bit 0 = 1；
- 固定 32-bit I/O address；
- PCIe Endpoint 不应依赖 OS 为 I/O BAR 分配资源，现代设备控制空间优先用 Memory BAR。

### Prefetchable 到底在承诺什么

它不是“CPU 可以随便 cache”。规范核心条件是：读无副作用、读能返回请求的所有 byte、Host Bridge 合并写不会破坏语义。doorbell、clear-on-read status 这类寄存器通常不能轻率标成 prefetchable。

Base Spec 对标准 PCIe Endpoint 还要求：申请 Memory Space 的 BAR 除非包含 read side effect 或不能容忍 write merging，否则应置 Prefetchable；prefetchable BAR 必须支持 64-bit addressing。来源：§§1.3.2.2、7.5.1.2.1。

## 3. BAR sizing 为什么要写全 1

硬件只实现它关心的高地址位，窗口内 offset 对应的低地址位固定返回 0。软件在禁用相应 address decode 后：

1. 保存 BAR 原值；
2. 写全 1；
3. 读回；
4. 清掉属性位；
5. 对掩码按位取反再加 1，得到 power-of-two size；
6. 恢复原值，再按资源规划写入新基址。

例如 32-bit Memory BAR 读回：

```text
0xFFFF_0000  （低 4 bit 属性先清掉）
~0xFFFF_0000 + 1 = 0x0001_0000 = 64 KiB
```

窗口必须自然对齐到其大小。未实现 BAR 读回 0。64-bit BAR 要对连续两个 DWORD 一起 sizing。

Base Spec 的 implementation note 还强调 sizing 前通过 Command Register 禁用对应 decode，避免写全 1 的中间值被当成真实地址。来源：§7.5.1.2.1，printed/PDF pp.696–698。

## 4. 地址分配不只写 EP BAR

若 EP 在 Switch/Root Port 后面，配置软件还要为路径上的 Type 1 bridge 编程 Memory Base/Limit 或 Prefetchable Memory Base/Limit。只有目标地址同时命中每一级桥的下游 window，请求才会被逐级转发。

因此：

```text
EP BAR 配对了，但桥窗口没包住它 → Memory TLP 仍可能到不了 EP
```

这类 bug 看起来像“BAR 坏了”，实际是树上的路由门没开。

## 5. 三个地址域

先约定名字：

1. **CPU virtual address**：驱动 `pci_iomap()` 后拿到、交给 `readl/writel` 的 `__iomem` 地址。
2. **CPU physical / AXI address**：CPU/SoC interconnect 上用于访问 host bridge aperture 的地址。
3. **PCIe address**：Memory TLP header 中用于 fabric address routing 的地址。

在 EP DMA 方向还会出现 **DMA address/IOVA**，它是设备应发出的地址，可能经 IOMMU 翻译到 Host physical memory。

这些地址“可能数值相等”，但概念永远不同。相等是平台配置结果，不是 PCIe 宇宙常数。

## 6. Cadence：PCIe inbound 怎样变成 AXI

Cadence 把从 Link 进入本地 AXI 的路径叫 inbound。

### EP mode

收到 Memory/I/O TLP 后，core 根据 **Function Number + BAR match** 选择对应 translation registers。每 Function 有 7 组 BAR translation registers（BAR0–BAR6，包含 Expansion ROM BAR 的实现语境需后续寄存器核对）。

转换逻辑：高位来自该 BAR 的 inbound PCIe-to-AXI translation register，低位来自 incoming PCIe address；传递多少低位由 BAR aperture 决定。

示意：

```text
EP BAR0 assigned PCIe range: 0x8000_0000–0x8000_FFFF
BAR0 inbound AXI base:       0x4001_0000

Incoming PCIe MWr address:   0x8000_0120
BAR0 hit, offset:            0x120
Generated AXI address:       0x4001_0120
```

实际寄存器 base/offset 和 BAR aperture encoding 要以 Chapter 9 对应 Link/Function block 为准。

### Root Port（RP）模式：Core 工作在 Host/RC 侧

这里沿用 Cadence 手册的 **RP mode**：它描述这一个 Dual-Mode Core 在 PCIe hierarchy
中表现为 Root Port；从 SoC/Host controller 的系统视角看，也常称为 **RC mode**。
RP 是 RC 向外提供的具体端口，不能因此把 RP 和整个 Root Complex 视为同义词。

RP 有 BAR0/BAR1 translation set，另有 BAR7 作为 no-match path；当 BAR matching disabled 时，HAL2AXI 会指示 BAR value 7。Translation register 的 `addr0[5:0] + 1` 决定从 incoming PCIe address 透传到 AXI address 的低位数量，手册要求最小编程值 7，因此最小粒度是 256 B。

来源：Cadence §§4.5.5.1–.2，printed pp.246–247 / PDF pp.335–336。

## 7. Cadence：AXI outbound 怎样变成 PCIe TLP

Cadence 提供两种方式。

### 7.1 Static / Region-based

最多 32 个 active regions。每个 region 有三组状态：

- **AXI Region Base Address**：匹配 AXI address 并规定 region size；
- **Outbound AXI-to-PCIe Address Translation**：给出 PCIe address 高位和低位透传位数；
- **Outbound PCIe Descriptor**：给出 transaction type、Requester ID/Function、TC、Attr、ATS、Poison、ECRC、PASID 等 TLP 静态信息。

Region 从 0 向上匹配，选择第一个命中的 region；overlap 不受支持。无 region 命中时 AXI 返回 DECERR。

对 Memory/I/O：`addr0[5:0] + 1` 个低位从 AXI address 透传到 PCIe address，高位来自 translation register。最低值 7，对应至少透传低 8 bit。

一个典型例子：

```text
AXI region base:      0x5000_0000
region aperture:      64 KiB（概念示例）
PCIe translated base: 0x8000_0000

AXI access:           0x5000_0120
PCIe TLP address:     0x8000_0120
```

Cadence Chapter 10 用 `axi_addr0[5:0]=19` 表达 1 MiB Memory region，验证了 aperture 为 `2^(N+1)` bytes，同时低 `N+1` address bits透传。配置 region 使用 `N=11` 对应 4 KiB；AXI address携带完整 BDF 的方法使用 `N=27`，因此 aperture 至少 256 MiB。**证据：Confirmed by programming examples。**

### 7.2 Dynamic / Sideband-descriptor-based

客户端直接在 AXI address/USER sideband 提供 translated PCIe address 和 TLP descriptor，绕过 region translation。它适合 TLP 信息快速动态变化、来不及反复 APB 重编 region 的场景。

这个接口能力很强，也意味着普通 SoC host driver 未必使用它；必须看实际 integration 和 driver，不能因为手册里有就默认 Linux 一定走这条路。

来源：Cadence §§4.5.6.6–4.5.6.6.5，printed pp.255–262 / PDF pp.344–351；Chapter 9 §9.5 registers 从 printed p.669 起。

## 8. Resizable BAR：大小也可以由软件协商

普通 BAR sizing 只读取当前固定 aperture；Resizable BAR Extended Capability 则列出硬件可接受的多个 size，系统软件选择最合适值写入 Control.BAR Size。

resize 不是在线拉伸内存窗口：

1. quiesce Function，停止相关 MMIO/DMA；
2. 清 Command.Memory Space Enable；
3. 写 Resizable BAR Control 的 size；
4. 此时原 BAR 内容未定义，重新分配并写 BAR base/bridge window；
5. 恢复 MSE 和 driver 数据面。

只有 64-bit BAR 才能宣告 4 GiB 及以上 size。软件通常选择资源布局可容纳的最大受支持 size；更大 aperture 可能改善设备可直接暴露的资源范围，也会挤占其他 MMIO address space。

Cadence PF BAR Configuration Register 的 bit 31 可为对应 BAR 启用 Resizable BAR capability；启用后 aperture 由 Resizable BAR Control 的 BAR Size 控制，未启用时仍由静态 BAR aperture 字段控制。这是 RTL build + local firmware + Host resource allocator 三方共同完成的能力，不能由 Function driver 在设备运行中随手切换。

来源：Base Spec §7.8.6, printed/PDF pp.862–866；Cadence PF BAR Configuration / Resizable BAR registers, Chapter 9。

## 9. Bus Master Enable 是 DMA 前置条件

Cadence §4.5.6 明确要求，EP 只有在对应 Function 的 PCI Command Register **Bus Master Enable** 置位时才能通过 AXI Slave interface 发 Memory Read/Write；Function 还必须处于 D0-Active，且不在 FLR 中。

这提供了非常具体的软件↔硬件映射：

```text
Linux pci_set_master()
 → Configuration Write 修改 Command.BME（调用链待内核源码确认）
 → Cadence FUNCTION_STATUS 反映 BME
 → client logic 才可以合法提交 AXI outbound request
 → core 生成 Memory TLP
```

## 10. Debug 路径

### CPU 访问 EP BAR 失败

1. `lspci` 检查 BAR 是否有资源、Command.MSE 是否置位。
2. 检查 Host/RC outbound CPU/AXI-to-PCIe region 是否覆盖 CPU physical access。
3. 看协议分析：是否发出正确 PCIe address 的 MWr/MRd。
4. 检查桥 Memory/Prefetchable window。
5. EP 侧检查 Function/BAR hit 和 inbound translation。
6. 看 Cadence inbound AXI Master 是否产生期望 AXI address/response。

### EP DMA 请求一发就 AXI DECERR

优先检查 region 是否匹配、region overlap/priority、base alignment 与 size，而不是先怪 PCIe Link。若 AXI request 都没成功进入 TLP generation，链路当然“清白得有点无聊”。

### MRd 发出后 AXI 返回 SLVERR/DECERR

Cadence §4.5.6.2 列出的原因包括 Poisoned Completion、UR/CA/CRS、byte count 错误、Requester ID/TC/Attr 不匹配、starting address 错误、Completion Timeout 或 FLR。应结合 Split Completion Table、AER/header log 与 AXI response 定位。

## 11. Source Reference

- PCIe Base Specification 5.0 r1.0, §7.5.1.2.1, printed/PDF pp.696–699。
- PCIe Base Specification 5.0 r1.0, §7.5.1.3.3–.9, pp.705–708。
- PCIe Base Specification 5.0 r1.0, §7.8.6, pp.862–866。
- Cadence PCIe Core User Guide, §§4.5.5–4.5.6.2, printed pp.246–249 / PDF pp.335–338。
- Cadence PCIe Core User Guide, §§4.5.6.6–.6.5, printed pp.255–262 / PDF pp.344–351。
- Cadence PCIe Core User Guide, §9.5, printed p.669 onward / PDF p.758 onward；Chapter 10 programming examples。
