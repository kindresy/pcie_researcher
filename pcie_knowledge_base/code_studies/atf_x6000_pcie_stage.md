# ATF/BL2 中的 PCIe：大包下载、片间枚举与中断接力

> 研究对象：`source_repos/atf`，分支 `x6000`，commit `98ac1eb5984ad69fc42860abbc9ed70346f240a8`（2026-03-20）。本文只讨论与 PCIe 有直接关系的 BL2 代码。

## 1. ATF 阶段的职责

X6000 的 ATF 仓库里，PCIe 有三份看似相似、实际职责完全不同的工作：

1. **继续对外 PCIe boot**：把 BootROM/PCS 的小包协议升级成 DDR 中的 512 KiB 批量下载通道，向 Host 报告 `DE12DE12`；
2. **建立内部 Chip-to-Chip PCIe**：在多 die 型号上，由 A8 充当 Root Complex（RC），亲自枚举 B0 Endpoint（EP），配置 BAR 和地址窗口；
3. **替 A0 转发中断**：A8 收到 B0 的 notify/event 后，读取 A0 的 MSI-X table，再向 Host 的 MSI doorbell 地址发一次 Memory Write。

换句话说，BL2 一手接快递、一手修片间高速路，还兼职中断“传话筒”。把这三条路径混在一起读，地址很快就会像意大利面一样缠住。

## 2. BL2 在启动链中的位置

```text
Host edge-driver / boot tool
        │  已枚举 A0 EP，PCIe Link 通常保持在线
        ▼
BootROM       DE10：最小 AP SRAM mailbox
        ▼
PCS           DE11：重画 APT，下载/分发 BL2
        ▼
ATF BL2       DE12：DDR 512 KiB bulk channel
        │             ├─ 下载 FIP / Linux
        │             ├─ 初始化 A8↔B0 C2C PCIe
        │             └─ 建立 IRQ relay
        ▼
Linux EP      DE13/DE14：EPC/EPF、uDMA、MSI-X 运行期服务
```

BL2 的 PCIe 源文件总计约 2,000 行，构建系统无条件把 `pcie.c`、`pcie_init.c`、`pcie_irq_relay.c`、`pcie_irq_misc.c` 和测试用 `c2c.c` 加入 BL2；`BOOT=pcie` 只负责定义 `PCIE_BOOT`，决定是否真的进入下载流程。

Source:

- `plat/intellif/edge10/platform.mk:78-91,246-263`
- `plat/intellif/edge10/edge10_bl2_setup.c:482-639`

## 3. 顶层调用链：谁在什么 die 上做什么

`bl2_platform_setup()` 在 DDR 初始化之后按产品、group、die 和 boot mode 分流：

```text
bl2_platform_setup()
  │
  ├─ LM/DS die1 ─→ c2c_ep_init(1) → c2c_ep_atu_init(1)
  ├─ LM/DS die7 ─→ c2c_rc_init(7) → c2c_rc_atu_init(7)
  │
  ├─ Group A die8 ─→ pcie_rc_init()
  │                   → c2c_rc_atu_init(8)
  │                   → irq_relay_init()
  │                   → irq_misc_init()
  │
  ├─ Group A die0 ─→ pcie_apt_remap_to_dma(g_apt_base[0])
  │                   → a0_pcie_set_inbound()
  │
  ├─ Group B die0 ─→ c2c_ep_atu_init(0)
  │                   → pcie_apt_remap_to_dma(g_apt_base[0])
  │                   → b0_pcie_set_inbound()
  │
  └─ die0 final/PCIe boot ─→ pcie_download_init()
                              → one or more pcie_download_main()
```

`pcie_rc_init()` 和 `c2c_rc_init()` 会完整操作 PMU、clock/reset、Torrent PHY、lane、Gen、LTSSM；而 A0 对外 EP 路径没有同等的重新训练动作。这延续了 PCS 的设计：**Host-facing Link 是启动阶段之间传下来的状态，C2C Link 才由 BL2 从头建立。**

Source:

- `plat/intellif/edge10/edge10_bl2_setup.c:533-639`
- `drivers/intellif/pcie/pcie_init.c:797-1143,1145-1322`

## 4. 下载通道：mailbox 门铃留在 SRAM，货物搬到 DDR

### 4.1 地址演进

BL2 仍使用 AP SRAM 中的两个 ready flag：

| 名称 | 地址 | 谁写 1 | 含义 |
|---|---:|---|---|
| H2D ready | `AP_SRAM + 0x7fa00` | Host | Host 已准备好一包数据 |
| D2H ready | `AP_SRAM + 0x7fa80` | Device | Device 已准备好响应 |

但是普通消息和 bulk data 已迁往 DDR：

| 区域 | 地址 | 用途 |
|---|---:|---|
| H2D message | `APSS_S34_DDR_BASE + 0x40000000` | CMD/响应协议输入 |
| D2H message | `APSS_S34_DDR_BASE + 0x40010000` | Device 响应 |
| H2D bulk buffer | `APSS_S34_DDR_BASE + 0xa0000000` | 512 KiB 固件块 |

这是一个经典的“doorbell + payload”设计：小旗子放在永远容易访问的 SRAM，重货放进容量和带宽都更合适的 DDR。

### 4.2 一包固件如何落到目标 DDR

Host 在 bulk buffer 首部写一个 32-bit 长度；第一包随后携带 `sb_header`。BL2 的路径是：

```text
Host BAR write / DMA
  → H2D bulk DDR buffer
  → Host sets H2D ready = 1
  → BL2 flush_dcache_range(512 KiB)
  → first packet: parse sb_header
  → destination = header.load_addr + 0x200000000
  → CPU memcpy unaligned prefix
  → SoC AXI DMA copies aligned remainder
  → BL2 clears H2D ready
  → repeat until received >= header.data_size
  → validate magic + CRC16
  → send R_RCV_DONE(download_cnt)
```

这里的 `+ 0x200000000` 是设备 SoC 地址域中的 DDR 基址修正，并非 PCIe Spec 定义的固定值。Host 传入镜像协议约定的 `load_addr`；BL2 再把它换算到自身可寻址的 DDR 空间。

`download_cnt` 还承担隐含的协议顺序：代码注释把 0–4 解释成 `fip-bh-1`、`fip-bh-7`、`fip-bh-8`、`fip-bh-0`、`boot_diex-bh.img`。Host 与固件因此不只有“包格式 ABI”，还有“第几轮下载是什么镜像”的时序 ABI。

Source:

- `drivers/intellif/pcie/pcie.c:15-21,41-83,86-182`
- `plat/intellif/edge10/edge10_multi_die.c:234-268`

## 5. `DE12DE12` 到底表示什么

`pcie_download_init()` 只做两件事：

```text
APT_REMAP_CTRL = 1
0x6007fa40 = 0xDE12DE12
```

因此 `DE12` 的准确含义应写成：**BL2 已进入 PCIe 下载阶段，并确认当前 APT 可用。** 它不等价于“所有内部 C2C Link 都成功”或“Linux 已经就绪”。非 die0 也会把 `DE12` 写到 `BOOT_STATE_MAGIC_ADDR + 4 * die_id`，形成多 die 状态数组。

对 Host 来说，这个 magic 是阶段同步点；对 Debug 来说，它还是一把很实用的尺：

- 看不到 `DE10`：先查 ROM/Link/枚举；
- 有 `DE10` 无 `DE11`：查 PCS 下载与跳转；
- 有 `DE11` 无 `DE12`：查 BL2 镜像、DDR、PCS→A55 释放；
- 有 `DE12` 但 bulk 卡住：查运行期 APT、DDR buffer、cache 和 ready flag。

Source:

- `drivers/intellif/pcie/pcie.h:5-6`
- `drivers/intellif/pcie/pcie.c:86-91`
- `plat/intellif/edge10/edge10_bl2_setup.c:592-610`

## 6. APT 的 BL2 增量：把 uDMA 控制面接进来

BL2 没有完全推倒 PCS 的 APT 表。Die0 final boot 时，它额外配置 region 9：

```text
APT input : apt_base + 0x00400000
SoC target: 0x58400000
size      : 0x00210000
```

该范围覆盖 PCIe address translation/uDMA 相关寄存器。它是启动通道从 CPU 小包复制升级到 DMA 传输的硬件基础，也能解释为什么后续 Linux runtime DTS 中仍有 `0x58400000` 一带的窗口。

注意这里至少存在三种地址：

1. Host 为 BAR 分配的 PCI address；
2. Cadence inbound ATU 产生的 `apt_base + offset`；
3. APT remap 后的 SoC target `0x58400000 + offset`。

把 `pci_iomap()` 返回的虚拟地址直接和 `0x58400000` 比较，是跨了两个地址域，结论通常会很精彩，也通常会错。

Source:

- `drivers/intellif/pcie/pcie_init.c:209-227,418-438,603-607`
- `plat/intellif/edge10/edge10_bl2_setup.c:570-586`

## 7. C2C PCIe：芯片内部也有一套 RC/EP 世界

### 7.1 基本角色

多 die 路径使用同一 Cadence 控制器模型构成内部 PCIe：

```text
Group A A8 (RC) ───── PCIe/C2C ───── Group B B0 (EP)
   bus 0                               bus 1, devfn 0

LM/DS 某些拓扑：die7 (RC) ───── die1 (EP)
```

通用窗口约定：

| 角色 | PCI bus base | 本地 CPU aperture base |
|---|---:|---:|
| EP outbound | `0x2d00000000` | `0x100000000` |
| RC outbound | `0x2c00000000` | `0x100000000` |

`g_apt_base[]` 则按位置选择 `0x3f00000000`、`0x3e00000000`、`0x3d00000000`……作为 inbound aperture。这里的“bus address”和“CPU address”恰好都很大，更要靠名字而不是肉眼猜地址域。

### 7.2 A8 真的做了一次最小枚举

`a8_pcie_set_outbound()` 是一段非常适合学习 Enumeration 的裸机例子：

1. 把 RC outbound region 0 配成 bus 1、devfn 0 的 Configuration Type 0 窗口；
2. 通过这个窗口读 Vendor ID / Device ID；
3. 把 Command Register 的 I/O Space、Memory Space、Bus Master 三个位一起置 1（`| 0x7`）；
4. 向 BAR0 写 `0xffffffff`，读回 mask，打印 size；
5. 给 BAR0/BAR2/BAR4 写入固定 bus address；
6. 配置 RC 的 outbound Memory regions，让本地 CPU aperture 对应这些 BAR bus address。

它没有 Linux PCI core 那样完整的桥扫描和资源冲突解决，因为拓扑是板级固定的一对一链路；但 Config Read/Write、BAR sizing、BAR assignment、MSE/BME 的本质完全一样。

### 7.3 A8 给 B0 分配的窗口

| Region | A8 CPU aperture | B0 PCI address / BAR | 大小 |
|---:|---:|---:|---:|
| 0 | `0x21fff00000` | config `0x2dfff00000` | 4 KiB |
| 1 | `0x21fc000000` | BAR0 `0x2dfc000000` | 16 MiB |
| 2 | `0x21f8000000` | BAR2 `0x2df8000000` | 64 MiB |
| 3 | `0x2100000000` | BAR4 `0x2e00000000` | 2 GiB |
| 4 | `0x2180000000` | continuation/window | 1 GiB |

A0 的 BAR1 又被指向 `0x21fc000000`，即 A8 访问 B0 BAR0 的 CPU aperture；B0 BAR2 inbound 则落到 `g_apt_base[2] = 0x3d00000000`，注释称为 B0 DDR `mmz-pcie`。这形成一条跨层地址路径：

```text
Host → A0 BAR1
     → A0 inbound address 0x21fc000000
     → A8 RC outbound region 1
     → C2C Memory TLP to B0 BAR0
     → B0 local APT / target
```

Source:

- `drivers/intellif/pcie/pcie_init.h:8-31`
- `drivers/intellif/pcie/pcie_init.c:209-227,271-383,441-528,691-792`

## 8. C2C inbound APT 地图

RC/EP 初始化会先清空 32 个 APT region，再按产品/位置组合以下窗口：

| Region | aperture offset | target | 大小 | 意图 |
|---:|---:|---:|---:|---|
| 0–3 | `0x0–0x1fffff` | 四个 die 的 `0x60000000` AP SRAM | 每段 512 KiB | mailbox / boot state |
| 9 | `0x00400000` | `0x58400000` | `0x210000` | PCIe ATU/uDMA control |
| 11 | `0x00380000` | `0xa6009000` | 4 KiB | 右侧 die CoreSight timestamp |
| 14–17 | `0x0ec00000...` | 各 die `0xa8000000` | 每段 1 MiB | AISS 小窗口 |
| 18–21 | `0x0f000000...` | 各 die `0xb0800000` | 每段 4 MiB | AISS 窗口 |
| 22–25 | `0x10000000...` | 各 die `0x50000000` | 每段 64 MiB | 大块设备窗口 |
| 26–29 | `0x20000000...` | 多 die 高地址 | 每段 128 MiB | 大块设备窗口 |
| 30–31 | `0x40000000/0x80000000` | 邻接 device/group | 每段 1 GiB | 跨 device/group |

这些不是 PCIe 标准规定的 BAR layout，而是 X6000 SoC integration policy。Spec 只规定 TLP、BAR 和地址 decode 行为；APT region 编号、`g_apt_base` 和多 die target 全是本项目硬件/固件约定。

Source:

- `drivers/intellif/pcie/pcie_init.c:576-667,691-792`

## 9. MSI-X relay：一次中断为什么会变成两次 Memory Write

### 9.1 直觉

B0 不是 Host 直接枚举到的那个函数，不能天然把自己的内部通知送到 Host 为 A0 分配的 MSI-X vector。于是 A8 做中继：先“接住”B0 的写，再代表 A0 向 Host doorbell 写一次。

### 9.2 完整路径

```text
B0 event/notify
  → C2C Memory Write
  → A8 MSI capture address: 0x3f001c0000
  → A8 GIC IRQ_PCIE0_MSIX_1..24
  → plat_bl2_fiq_handle()
  → irq_relay_notify_isr()
      ├─ clear A8 captured MSI bit
      ├─ locate A0 MSI-X table through A0 controller + 0xb4
      ├─ read vector[notify + 25] {msg_addr,msg_data,mask}
      ├─ lazily program A0 outbound region 0 around msg_addr
      └─ writel(msg_data, outbound + low_20_bits(msg_addr))
  → A0 emits PCIe Memory Write TLP
  → Host MSI/MSI-X interrupt controller
  → edge-driver notify ISR
```

这正好解释了“MSI 为什么不需要一根物理中断线”：最终动作仍是一笔地址+数据写，只不过 Host 把这类写解释为 interrupt message。

代码把 notify vector 加上 `EDGE_IRQ_MAX_NUM`（25）后索引 A0 MSI-X table。这与 Host edge-driver 的 49-vector 布局对得上：前面是 DMA/基础 vector，后 24 个是 notify vector。event 路径不直接构造 MSI-X write，而是脉冲 A0 `MISC_CTRL_STS4.bit2`，让 A0 再产生事件。

### 9.3 中断注册链

BL2 把 PCIe MSI-X 0–23、INTA 和 MISC IRQ 都声明成 GICv2 Group 0 secure interrupts。统一 FIQ handler 按 IRQ 号分派：

- MSI-X 0–23 → `irq_relay_notify_isr()`；
- INTA → `irq_relay_event_isr()`；
- MISC → `irq_misc_isr()`。

初始化时 `irq_relay_init()` 配置 A8 的 MSI capture 地址与 enable；MSI-X table 和 A0 outbound region 则第一次中断到达后 lazy setup，因为 Host 何时填完 MSI-X table 并不由 BL2 控制。

Source:

- `plat/intellif/edge10/edge10_bl2_setup.c:75-154,419-459,559-566`
- `drivers/intellif/pcie/pcie_irq_relay.c:1-237`

## 10. MISC error 与 firewall

`irq_misc_init()` 写 `MISC_INT_MSK = 0xfffd`，只留下 bit 1。该位触发后 ISR：

1. 读取并清全部 MISC status；
2. 写五个固定 firewall 寄存器；
3. 上报 `EXCEPTION_MOD_PCIE / EXCEPTION_TYPE_BL2 / EMERG / ECONNABORTED`。

代码没有给 bit 1 命名，因此目前只能确认“它被当成致命 PCIe 连接中止事件”，不能从现有源码可靠声称它就是标准 AER Surprise Down 或 Completion Timeout。PCIe controller register manual 是消掉这个 `NEED_VERIFY` 的正确资料。

工程上还要区分两套 firewall：通用 `edge10_firewall_init()` 在平台 setup 中被注释掉；PCIe MISC ISR 里的固定寄存器写是错误发生后的专项动作，不等于系统启动时全面启用安全防火墙。

Source:

- `drivers/intellif/pcie/pcie_irq_misc.c:1-49`
- `plat/intellif/edge10/edge10_bl2_setup.c:499-504`

## 11. SMMU/IOMMU 在这一阶段出现了吗

在 X6000 platform 和 `drivers/intellif/pcie/` 中没有 SMMU/IOMMU 初始化或 stream table 配置。仓库确实带通用 Arm SMMUv3 头文件，但没有证据表明本平台 BL2 使用它。

因此BL2的AXI DMA和PCIe地址翻译应按“物理SoC地址 + Cadence ATU/APT”理解，
不能把IOMMU映射倒灌到BL2。后续核对已确认X6000设备Linux defconfig关闭
`CONFIG_IOMMU_SUPPORT`；但外部Host的`dma_map_*()`是否返回IOVA，仍由Host平台决定。

## 12. 静态审计发现：值得上板验证的风险

以下是源码证据支持的风险，不代表已经在硬件上复现：

### 12.1 超长包只打印，不拒绝

`pcie_download_main()` 检测 `len > 512 KiB payload capacity` 后只打印日志，仍继续 memcpy 和 AXI DMA。恶意或损坏长度可能越界读取 bulk buffer。

### 12.2 对齐前缀可能大于短包

`unaligned_len` 取到下一个 16-byte 边界的距离，随后直接计算 `sg_len = len - unaligned_len`。若极短包的 `len < unaligned_len`，无符号下溢会把 DMA 长度变成很大的数。

### 12.3 DMA 失败后协议仍可能返回成功

`axidma_memcpy_sg()` 失败只打印 ERROR，循环仍累计 `download_len`，最后可能仅凭目标区内容计算 CRC。CRC 通常会兜底，但错误语义不直接，定位时容易只看到 Host 的 CRC error。

### 12.4 Link timeout 返回值被忽略

`pcie_rc_init()` 最后调用 `c2c_wait_for_linkup(0)`，不检查返回值并始终返回 0；顶层也不检查 `pcie_rc_init()`。结果可能是“日志打印 Fail，但继续配置 ATU/IRQ relay”。

### 12.5 硬件 spinlock 的 acquire 语义不明确

`__hwspin_lock()` 只轮询 bit 0 是否为 0，没有显式写 1；unlock 写 0。若该硬件寄存器是 read-to-acquire，这可能正确；若不是，就没有真正互斥。现有代码和缺失的 spinlock register spec 无法定论，标记 `NEED_VERIFY`。

### 12.6 MSI-X table offset 缺少范围校验

代码只检查 table offset 非零，随后直接把 `APSRAM_BASE + offset` 当表地址，也没有验证 `notify_irq + 25` 是否落在 Host 实际分配的 table entries 内。当前固定 49-vector ABI 让它“按约定工作”，但 ABI 变化时风险很高。

## 13. 上板 Debug 顺序

### 下载停在 DE11

1. 检查 BL2 是否真正运行、DDR 初始化是否完成；
2. 读 `0x6007fa40`，确认是否写 `DE12DE12`；
3. 读 APT remap enable 和 region 9；
4. 检查 Host 是否仍能访问 AP SRAM ready flag；
5. 检查 bulk DDR aperture 是否与当前阶段 BAR layout 一致。

### C2C Link 不通

1. 先分清是对外 A0 Link 还是 A8↔B0 C2C Link；
2. 看 PHY reference clock stable 等待是否超时；
3. 看 mode select 是否为 RC/EP、lane count 与 Gen 是否匹配；
4. 读 Link Status；
5. 在 A8 config aperture 读 B0 Vendor ID——全 1 通常意味着 Config Completion 没回来；
6. 再检查 BAR 和 outbound region，而不是一上来就怪 DMA。

### Host 收不到 notify

1. 看 A8 capture status 和对应 GIC IRQ 是否到达；
2. 确认 A0 controller `0xb4` 的 MSI-X table offset 非零；
3. 读取目标 entry 的 mask bit、message address/data；
4. 检查 A0 outbound region 0 是否以 vector 0 message address 的 1 MiB 对齐地址配置；
5. 确认目标 vector 是 `notify + 25`，与 Host 49-vector 分配一致；
6. 最后才查 Host `/proc/interrupts` 和 edge-driver ISR 计数。

## 14. 与 Linux 的交接边界

BL2 建立的是启动期可用世界，并非 Linux 最终资源模型：

- BL2 用固定地址、固定拓扑做 C2C 最小枚举；Linux 以后可能重新配置 controller/ATU/APT；
- BL2 的 IRQ relay 在 secure Group 0 FIQ 中运行；Linux 有自己的 irq relay 实现，需要确认切换时谁 mask、谁清 pending；
- BL2 未配置 SMMU；Linux DMA API 不能继承这一假设；
- Host-facing A0 MSI-X table 内容来自 Host，BL2 和 Linux 都可能消费它，所以 table layout 是跨固件/内核 ABI。

后续综合文档必须把交接写成“状态所有权表”，至少包含 Link、BAR config、APT、outbound region、MSI capture、GIC routing、uDMA 和 ready flag。只画一条从 ROM 到 Linux 的箭头还不够——箭头很礼貌，寄存器可没那么礼貌。

## 15. Source Reference

- `drivers/intellif/pcie/pcie.c`：BL2 mailbox/bulk download、CRC 与 `DE12`。
- `drivers/intellif/pcie/pcie.h`：BL2 boot magic。
- `drivers/intellif/pcie/pcie_init.c`：Cadence outbound/inbound、APT、C2C RC/EP、PHY/Link。
- `drivers/intellif/pcie/pcie_init.h`：A8↔B0 固定 config/BAR 地址窗口。
- `drivers/intellif/pcie/pcie_irq_relay.c`：MSI capture、A0 MSI-X table 和 outbound replay。
- `drivers/intellif/pcie/pcie_irq_misc.c`：MISC fatal path 与 firewall。
- `plat/intellif/edge10/edge10_bl2_setup.c`：按 group/die/boot mode 的真实调用链、GIC FIQ 分派。
- `plat/intellif/edge10/edge10_multi_die.c`：多 die 下载及 AXI DMA 分发。
- `plat/intellif/edge10/platform.mk`：构建条件与源文件集合。
