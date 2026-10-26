# X6000 Linux PCIe 栈：从 platform driver 到 EPC/EPF、BAR、MSI 与 RC

> 研究对象：`source_repos/linux` 分支 `x6000`，提交
> `e74fd0e7746a834a5cbbbdbc1c36204f4a26a6a3`，内核基线 5.10.4。
>
> 本文只覆盖 PCIe 直接相关的 vendor driver、Endpoint Framework、PCI core 接点、
> DTS 与中断/DMA辅助代码；没有把整棵 Linux 当百科全书抄一遍。

## 1. 这不是 DesignWare：主控制器来自 Cadence 架构

源码头、寄存器布局和 outbound descriptor 都表明 X6000 主 PCIe controller 是
Cadence 风格，而不是 Synopsys DesignWare。工程里虽然也有通用 DWC 驱动目录，
但 X6000 的直接调用链是：

```text
DTS intellif,edge-pcie-ep/host
  → pcie-edge-plat.c             SoC 时钟、复位、PHY、模式、lane、speed
  → pcie-edge.c                 Cadence ATU 与公共操作
  ├─→ pcie-edge-ep.c            Endpoint Controller (EPC)
  │    → Linux PCI Endpoint Framework
  │         → pci-epf-edge.c    Endpoint Function (EPF)
  └─→ pcie-edge-host.c          Root Complex / host bridge
       → Linux PCI core enumeration
```

主构建关系见 `drivers/pci/controller/Makefile:40-41`。cascade 是另一套硬件集成，
在后文单列，不能把它当作主代码的“备用名字”。

## 2. EPC、EPF、Host driver：三顶帽子别戴串了

| 组件 | 它代表什么 | 对外负责什么 |
|---|---|---|
| EPC driver | X6000 PCIe Endpoint Controller 硬件 | 配 config header、BAR、ATU、MSI、Link |
| EPF driver | Host 最终看到的一个 PCI Function | vendor/device/class、BAR需求、MSI-X表 |
| Host `edge.ko` | 外部服务器上的功能驱动 | 映射 BAR、控制 X6000 uDMA、提供用户 ABI |

Linux Endpoint Framework 把“控制器怎么编程”和“这个 Function 长什么样”分开。
这类似 SPI controller driver 与某个 SPI device driver 的分工，只是 PCIe 多了
Configuration Space、BAR 和 Link 等一大串上下文。

## 3. platform probe：Linux 接手控制器硬件

`edge_plat_pcie_probe()` 在 `pcie-edge-plat.c:377-538`。公共准备工作是解析：

- controller register、system-control、slave aperture；
- vendor/device/index；
- HS/P/AXI/PM clocks 与 AXI frequency；
- APB/PIPE/LINK/AXI/MGMT/MGMT_STICKY/PM resets；
- PHY；
- strap 推导出的 die/group ID。

### 3.1 EP 路径

```text
解析 DTS
 → PHY init + power_on
 → 开 clocks
 → deassert APB reset
 → deassert PIPE/LINK/AXI/MGMT/PM resets
 → runtime PM get
 → edge_pcie_ep_setup()
```

值得注意：EP 分支没有调用 `edge_pcie_ctrl_init()`，也没有在这里设置 RC/EP mode、
speed、lane count。**Inferred**：EP 的这些关键项已经由 BootROM/PCS/ATF 设置并保留，
Linux 选择“接管现状”而不是重做训练前配置。这个推断将在固件专题验证。

### 3.2 RC 路径

RC 路径会额外等待 PHY clock stable，调用 `edge_pcie_ctrl_init()` 设置 mode/speed/
lane，再去 `edge_pcie_host_setup()`。这说明同一 IP 在板内 cascade/root-port 场景下，
Linux 的所有权更完整。

Link training enable 是 system-control `MISC_CTRL_STS1.bit11`；Link up 判据是
`MISC_CTRL_STS2[1:0] == 3`，见 `pcie-edge-plat.c:38-68`。这个软件枚举值对应
“Data Link initialization complete”，不是简单的“PHY 有电”。

## 4. DTS 是地址地图，不只是开关

X6000/Edge10 Ultra 的公共 EP 节点在
`arch/arm64/boot/dts/intellif/hsio/edge10_ultra_hsio_pcie_card.dtsi:41-125`；
但 die0 产品 DTS 又在 `&pcie0` 中**整项覆盖** `apt-ranges`。公共 DTSI 是硬件
能力/模板，最终 DTB 的运行期列表必须读具体
`edge10_ultra_{a,b}_die0_a55_prod{,_lm,_ds}_unconverted.dts`。只读 include 文件会
得到一张看起来很完整、实际没有原样交给驱动的地图。

Ultra 节点给出：

- controller：`0x58000000`, size 8 MiB；
- system control：`0x5a200000`, size 4 KiB；
- EPC slave aperture：`0x1_00000000`, size 4 GiB；
- vendor/device：`17cd:0100`；
- 8 lanes；
- 32 outbound regions、32 APT regions；
- 一个 DMA IRQ、local/misc/PM/error，24 个普通 captured MSI IRQ，以及 thermal、
  link-info、LED mode、DDR DFS 四个专用 captured MSI IRQ。

### 4.1 `apt-ranges` 怎么读

每个条目含：`bar_no | apt_region_idx`、BAR offset、SoC CPU/AXI address、size。
以普通（非 LM/DS）A 组 die0 最终 DTS 为例：

| 编码 | BAR/region | BAR offset | EP AXI address | size |
|---|---|---:|---:|---:|
| `0x9` | BAR0 / region 9 | `0x400000` | `0x58400000` ATU+uDMA连续块 | `0x210000` |
| `0xD` | BAR0 / region 13 | `0x800000` | `0x2a0000000` group 0 | 2 MiB |
| `0xE` | BAR0 / region 14 | `0xa00000` | `0x6a0000000` group 1 | 2 MiB |
| `0xF` | BAR0 / region 15 | `0xc00000` | `0x1ea0000000` group 2 | 2 MiB |
| `0x10` | BAR0 / region 16 | `0xe00000` | `0x22a0000000` group 3 | 2 MiB |
| `0x2001C` | BAR2 / region 28 | `0x0000000` | `0x2b0000000` group 0 | 16 MiB |
| `0x2001D` | BAR2 / region 29 | `0x1000000` | `0x6b0000000` group 1 | 16 MiB |
| `0x2001E` | BAR2 / region 30 | `0x2000000` | `0x1eb0000000` group 2 | 16 MiB |
| `0x2001F` | BAR2 / region 31 | `0x3000000` | `0x22b0000000` group 3 | 16 MiB |
| `0x4000A` | BAR4 / region 10 | `0` | `0x21f8000000` | 64 MiB |

解析器在 `pcie-edge.c:219-255`，高位 `[18:16]` 取 BAR number，低 5 bit 取
APT region index。这解释了为什么 `0x2001C` 是 BAR2 + region28。

APT 是 SoC wrapper 的 **二级 inbound remap**：Cadence 的 BAR inbound register 先
把 Host 分配的 BAR 地址命中到 `0x3f/3d/3b...` aperture；system-control APT 再按
BAR 内 offset 把请求送到 controller register、DDR 或其他 AXI 地址。

region 9 闭合 Host 驱动的连续本地控制块：

```text
BAR0 + 0x400000 → 0x58400000  AXI configuration/ATU起点
BAR0 + 0x600000 → 0x58600000  Cadence uDMA起点
```

Host `REG_AXI_CFG_ADDR` 数值是 `0x800000`，它命中 region 13 的 group 0 窗口，
用于Host侧远程/outbound编程；不能写成“本地 `0x58400000` ATU”。

最终DTS没有重新列出AP SRAM region 0。EPC `set_bar()`只写命中的条目，不先清空
32个APT，因此boot mailbox等窗口依赖BootROM/PCS/BL2保留。这是跨阶段硬件状态ABI，
不是DTS默认映射。LM/DS与B组还会缩减或更换条目，详见统一地址地图。

## 5. EP Controller 初始化

`edge_pcie_ep_setup()` 见 `pcie-edge-ep.c:1208-1472`：

1. 解析 outbound/APT region 数量和 `apt-ranges`；
2. 暂时只 enable Function 0；
3. `devm_pci_epc_create()` 注册 EPC 与 `edge_pcie_epc_ops`；
4. `pci_epc_mem_init()` 管理 4 GiB slave aperture；
5. 从中分配 1 GiB 虚拟/物理窗口专供 MSI/INTx outbound；
6. 保留 outbound region 0 给中断；
7. 映射四颗 die 的 PCIe/link status 与 B0 SRAM；
8. 请求 thermal/misc/LED/DFS/DMA/link 和 24 个 notify capture IRQ；
9. 配置 MSI capture address `BAR0 aperture + 0x40000`；
10. enable uDMA IRQ 与 MSI capture；
11. 发布全局 `g_epc` 供业务模块发通知。

这里的 `pci_epc_mem_init()` 管的是“EP 本地可用于 outbound 的 CPU aperture”，不是
给 Host 分配普通 RAM。`pci_epc_mem_alloc_addr(..., SZ_1G)` 返回一段 EP CPU 可写的
地址；当 outbound region 0 被重定向到 Host MSI address 后，对它 `writel()` 才会
跨 PCIe 变成 MSI Memory Write。

## 6. EPF：决定 Host 枚举出来的 Function 长什么样

`pci-epf-edge.c` 只有 263 行，却卡在整条链最关键的位置。

### 6.1 bind

`pci_epf_edge_bind()`：

- 查询 EPC capability；
- 把所有 BAR 标为 controller 要求的 64-bit；
- MSI-X table 指定为 BAR0 + `0x2c000`；
- BAR0 local physical address 指向 AP SRAM `0x60000000`，映射 512 KiB；
- 调用 core init 写 header、BAR、MSI 和 MSI-X capability。

`EDGE_MSIX_TABLE_BIR_PHYS_ADDR` 名字略怪，它其实是 BAR0 后端 local physical
address，不是 Host 的 BIR。BIR 是 MSI-X capability 里的 BAR Indicator Register
字段，这里明确选 BAR0。

### 6.2 配置空间字段来自哪里

EPF 默认 header 的 vendor/device 是 `PCI_ANY_ID`；configfs 可在 bind 前写
`vendorid/deviceid/msi_interrupts/msix_interrupts`。随后：

```text
configfs 参数
 → pci_epf_edge_core_init()
 → pci_epc_write_header()/set_bar()/set_msi()/set_msix()
 → EPC ops
 → edge_pcie_ep_fn_write*()
 → Cadence Function 0 configuration registers
 → Host Configuration Read TLP 的返回内容
```

Linux core wrapper 位于 `pci-epc-core.c:245-323, 410-464`。它负责锁和参数校验，
真正写硬件的是 `pcie-edge-ep.c:636-888`。

### 6.3 谁创建 EPF、谁按下 start

主 X6000 `pcie-edge-ep.c` 只创建 EPC，没有像 cascade driver 那样在内核里直接
`pci_epf_create("pci_epf_edge")`。标准路径应是 configfs：创建 function、写 ID/
vector 数、link 到 controller，再写 controller `start=1`。现有六仓源码中尚未找到
执行这些 configfs 操作的启动脚本，故标记 **Unknown / NEED_VERIFY**：它可能存在于
rootfs/部署包而非这六个仓库。

这是重要证据缺口，因为 Host driver 要求 49 个 MSI-X vector，而 EPF 的
`msix_interrupts` 必须由外部配置成至少 49 才能兑现。不能仅凭 Host 数组大小认定
EP 一定广告了 49 个。

## 7. BAR 的完整两级翻译

### 7.1 BAR setup

EPC `set_bar()` 只处理 BAR0/2/4，并分别设定 Cadence inbound target：

```text
BAR0 → 0x3f00000000
BAR2 → 0x3d00000000
BAR4 → 0x3b00000000
```

然后遍历 DTS `apt-ranges`，配置：

```text
aperture + BAR offset .. limit  → EP AXI remap address
```

见 `pcie-edge-ep.c:667-726`。Host对BAR0+`0x400000`的写落到`0x58400000`；
Host driver的controller基址确为`0x400000`。其`REG_UDMA_ADDR = base + 0x200000`
进一步落到`0x58600000`。但`REG_AXI_CFG_ADDR = base + 0x400000`即BAR0+`0x800000`，
命中group窗口；三者必须分别解释。

### 7.2 一个具体访问

假设 Host 给 BAR0 分配 `0x80000000`，执行：

```c
writel(v, host_bar0 + 0x600018);
```

路径为：

```text
Host CPU VA
 → Host PA 0x80600018
 → RC 产生 Memory Write TLP(address 0x80600018)
 → X6000 Function0 BAR0 hit，得到 offset 0x600018
 → Cadence inbound target 0x3f00600018
 → APT region9 remap为 0x58600018
 → Cadence uDMA register
```

最后那个offset位于uDMA block。同一个`writel()`走了两级地址
翻译，难怪 PCIe 比 SPI 的“寄存器基址+offset”更会套娃。

### 7.3 Linux 保留 BAR 的含义

group 0 把 BAR1 标 reserved，group 1 把 BAR4 标 reserved，注释写明“不在 Linux
中用 set_bar 重配固件留下的 APT”。见 `pcie-edge-ep.c:1411-1416`。这进一步证明
部分 BAR/地址窗口跨 boot stage 保留，Linux 不是唯一 owner。

**Risk**：`edge_pcie_epc_features` 是全局 static，却在不同 EPC probe 中按 group
修改 `reserved_bar`。多 controller 并行 probe 时，后一个 group 会改变前一个
EPC `get_features()` 的结果。这是共享可变 capability 的竞争/串扰风险。

## 8. outbound ATU：EP 如何访问 Host memory

`edge_pcie_ep_map_addr()` 从 region 1 起找空闲 region（region 0 留给 IRQ），调用
`edge_pcie_set_outbound_region()` 把：

```text
EP CPU/AXI address window → Host PCI/DMA address window
```

映射起来。公共函数会写 PCI target、TLP type、function number 与 AXI source base，
见 `pcie-edge.c:88-171`。EP mode 不 hardcode bus/device，controller 使用训练/枚举时
捕获的 BDF，只显式填 function number。

Host `edge.ko` 的 uDMA 没直接调用 Linux EPC `map_addr()`：它通过 BAR0远程写相同
controller register。也就是说，Host 驱动与 EP Linux 都能成为 ATU programmer，
因此 region ownership 和硬件锁非常重要；部分 relay 代码确实使用硬件 spinlock。

## 9. MSI/MSI-X：从 table entry 到一笔 Memory Write

### 9.1 Capability

EPF 请求 MSI-X 时，EPC 把 QSIZE 写成 `interrupts-1`，table 指到 BAR0+0x2c000，
PBA 紧跟 table 后，见 `pcie-edge-ep.c:863-888`。

Host 枚举和启用 MSI-X 后，会把每个 entry 的 `msg_addr/msg_data/vector_ctrl` 写进
BAR0 SRAM。EP `edge_pcie_ep_send_msix_irq()`：

1. 检查 Function 的 MSI-X Enable；
2. 从 config space 读 BIR/table offset；
3. 从 BAR 后端 SRAM 读目标 entry；
4. 检查 per-vector mask；
5. 把 outbound region 0 映到 `msg_addr` 所在的 1 MiB；
6. 向 `msg_addr` offset 写 `msg_data`。

第 6 步在 PCIe 链路上就是 MSI-X Memory Write TLP。源码证据为
`pcie-edge-ep.c:995-1074`。

### 9.2 DMA 与 notify vector 对齐

- `pci_epc_raise_irq(... MSI-X, 1)`：MSI-X entry 0，Host `edge.ko` vector 0，DMA；
- `chan_id + 2`：entry 1..24，Host vector 1..24，notify group 0；
- group 1 特殊路径通过 B0/A8 relay 转发到 table 后半段，Host vector 25..48。

这解释了代码里看似多余的 `+1+1`：EPC API 的 interrupt number 从 1 起，而 vector
0 已经留给 DMA。

### 9.3 DMA completion 的 flush read

DMA ISR 在向 Host 发 MSI 前读取 `B0_X8_APB`，注释说用它 flush groupB DMA write，
见 `pcie-edge-ep.c:1182-1204`。目的不是取这个寄存器的值，而是建立 posted writes
的可见性顺序：先确保数据写到达，再敲完成门铃，否则 Host 可能“先收到中断，后
看到旧数据”。具体 ordering 保证需要再与 Databook 核对，当前标为 **Inferred**。

## 10. notify 双向协议闭环

EP 与 Host 使用相同 `notify_irq_shm` ABI、24 channel、16-slot queue 和地址常量。

### Host → EP

```text
Host 写 BAR0 shared queue
 → wmb
 → Host 写 BAR0+0x40000，data=channel
 → X6000 MSI capture 把该 Memory Write 转成 GIC IRQ 5+channel
 → edge_notify_irq_isr()
 → 排空 h2d_shm
 → blocking notifier
```

注意这里叫“MSI capture”，但线上的 transaction 是 Host 发给 EP BAR 的普通 Memory
Write；X6000 wrapper 把特定地址写捕获成本地中断。它不是标准的“EP 给 RC 发 MSI”
方向。

### EP → Host

```text
EP 写 d2h_shm
 → wmb
 → pci_epc_raise_irq(MSI-X, channel+2)
 → outbound Memory Write 到 Host MSI-X msg_addr
 → Host vector ISR
 → Host 排空 d2h_shm
```

对称代码在 `pcie-edge-ep.c:285-635` 与 Host `edge.c:459-988`。

## 11. Link start 与 Linux magic 的精确顺序

用户/启动脚本写 configfs controller `start=1`：

```text
pci_epc_start()
 → edge_pcie_ep_start()
 → enable Function bitmap
 → system-control LTSSM enable
 → 把 0xde14de14 写入 BAR0 backend + 0x7fa40
```

代码见 `pci-ep-cfs.c:44-72`、`pci-epc-core.c:161-177`、
`pcie-edge-ep.c:1098-1128`。实现先发起 Link start，随即写 magic，并不等待 Host
完成 Enumeration。Host 只有已经能访问 BAR 后才看得到 magic，因此从 Host 视角
仍然是可靠的业务阶段标记，但它不等价于“Host driver probe 已结束”。

## 12. X6000 内部 RC / cascade host 路径

同一 platform driver 能匹配 `intellif,edge-pcie-host`。`edge_pcie_host_setup()`：

1. 映射 4 KiB config aperture；
2. assert PERST# 约 101 ms，再 deassert 后等 100 ms；
3. enable LTSSM，最多约 1 秒等待 Link；
4. 初始化 Root Port Type 1 configuration header；
5. region 0 专门用于 config access；
6. 其余 outbound regions 映射 DTS IO/MEM windows；
7. 根据 `dma-ranges` 配置 inbound BAR/no-BAR match；
8. 建 INTx 与 MSI irqdomain；
9. `pci_host_probe()` 触发标准 Linux enumeration。

### 12.1 Configuration Read 的硬件路径

PCI core 扫描某个 BDF 时调用 `pci_ops.map_bus()`：

```text
Linux scan BDF
 → edge_pci_map_bus(bus, devfn, where)
 → 动态写 outbound region0 的 bus/devfn
 → 次级总线用 Config Type 0，更深总线用 Type 1
 → generic_config_read() 访问 cfg_base
 → Cadence 产生 Config Read TLP
 → 对端 Completion with Data
```

见 `pcie-edge-host.c:80-132`。这就是“Linux 软件枚举”落到 Configuration TLP 的
明确连接点。

### 12.2 RC MSI domain

RC 分配一页作为 MSI target，wrapper 最多捕获 64 个 vector；compose callback 把
target address 与 hwirq data 写进下游 Function MSI/MSI-X table/capability。收到写
后，wrapper 触发独立 GIC line，chained handler 清 capture status 并转进 Linux
MSI domain。见 `pcie-edge-host.c:527-761`。

## 13. 多 die IRQ relay

真正由 `controller/Makefile:38` 无条件构建的是 `pcie_irq_relay.c`。它是 X6000
A8/B0 跨 die 中断桥，不是外部 Host 的普通 PCI driver：

```text
A8 local captured notify IRQ
 → relay ISR 清 A8 capture bit
 → 读取 A0/B0 AP SRAM 中 Host 已写好的 MSI-X table
 → 用 A0 controller outbound region0 映射 Host msg_addr
 → 写 msg_data
 → 外部 Host 收到 group1 vector 25..48
```

它还把 group1 event INTx 转发到 A0 system-control。A0 outbound region0 同时会被
EP MSI 发送逻辑使用，所以 relay 用硬件 spinlock 126 试图协调。ATF 里也有同名
relay 实现，说明这个职责跨启动阶段延续；最终所有权将在 ATF 专题确认。

仓库另有 `pcie-edge-irq-relay.c` PCI function 版，但当前 Makefile 没有构建它；
本文将它归类为历史/替代实现，而非 x6000 默认运行路径。

## 14. 与 Host `edge.ko` 的对照表

| Host 侧观察 | X6000 Linux 侧证据 | 状态 |
|---|---|---|
| `BAR0+0x400000` 是本地AXI config/ATU起点 | 最终产品DTS region9映到`0x58400000` | Confirmed |
| `BAR0+0x600000` 是本地uDMA起点 | Host常量与region9线性偏移共同映到`0x58600000` | Confirmed |
| `BAR0+0x800000` 是本地ATU | 实际命中产品group窗口；Host用它编程远端/outbound视图 | 错误理解 |
| descriptor local base `0x60000000` | BAR0 backend/AP SRAM 正是 `0x60000000` | Confirmed |
| notify doorbell `0x40000` | wrapper MSI-capture target | Confirmed |
| notify SHM `0x40100 + n*0x800` | EP 完全同构结构与 offset | Confirmed |
| vector0 是 DMA | EP DMA ISR raise interrupt 1 | Confirmed |
| vector1..24 是 group0 notify | EP raise channel+2 | Confirmed |
| vector25..48 是 group1 notify | `pcie_irq_relay` table index `25+chan` | Confirmed |
| Linux magic `0xde14de14` | EPC start 写 BAR0+`0x7fa40` | Confirmed |
| 读 B0 APB flush D2H | EP DMA ISR 明确这样做 | Confirmed |
| Host 远程改 outbound ATU | BAR0 APT 暴露 controller AT block | Confirmed |
| 一定有 49 MSI-X | 配置值来自未找到的 configfs启动脚本 | NEED_VERIFY |

### 14.1 Device侧IOMMU结论

X6000目标defconfig（包括`edge10_a55_defconfig`、emu、FPGA、cascade、EVB等）都明确
写着`# CONFIG_IOMMU_SUPPORT is not set`，Host-facing EP节点也没有`iommus`属性。
因此在这份设备Linux构建中，本地AXI/uDMA路径不经过Linux管理的Device-side IOMMU。

这不等于外部Host没有IOMMU：`edge.ko`运行在哪台服务器、Host kernel如何配置VT-d/
SMMU不属于这棵设备内核。Host `dma_map_sg()`返回值仍必须按DMA address/可能的IOVA
使用，不能因Device defconfig关闭IOMMU就把它当成“必然Host物理地址”。

## 15. 静态审计发现

1. **EP notify affinity 使用未分配 cpumask。** `pcie-edge-ep.c:1440-1454` 声明
   `cpumask_var_t mask` 后直接 `cpumask_clear()`，没有 `alloc_cpumask_var()`；这是
   Confirmed bug，可能破坏内存或崩溃。Host controller 同类代码正确分配了 mask。
2. **全局 `g_epc` 只容纳一个 EPC。** 多 controller/group probe 时最后者覆盖前者，
   exported notify API 的 `device_id` 又没有用于选择 EPC，存在路由错误风险。
3. **全局 mutable `edge_pcie_epc_features`。** 不同 group 的 reserved BAR 会互相
   覆盖，见前文。
4. **BAR clear API 可能空指针。** EPF remove 调用
   `pci_epc_clear_bar(epc, fn, NULL)`，而 core `pci_epc_clear_bar()` 在校验中先访问
   `epf_bar->barno`；这是明确的 NULL dereference 路径（`pci-epf-edge.c:219-224` 对
   `pci-epc-core.c:385-399`）。
5. **EPF bind 的 `epc_features` 未空值防护。** 虽然本 EPC 总会返回 feature，通用
   健壮性不足。
6. **EP setup 多处早退泄漏。** 多个 raw `ioremap()` 或 IRQ 失败后直接 return，
   没有沿统一 unwind 释放此前映射/IRQ。
7. **system command 是高风险控制面。** PCIe misc event 可触发
   `call_usermodehelper("/bin/sh -c <共享内存命令>")`，虽然校验 magic/长度/加和，
   但不是密码学认证；任何能写对应 PCIe共享区并触发中断的主体可能取得 EP root
   命令执行。必须把 PCIe拓扑、BAR权限和运维工具访问当安全边界。
8. **relay 假设 Linux IRQ 连续。** `notify_irq = irq - notify_irq_base` 与 DTS/GIC
   mapping 绑定，在一般 irqdomain 上不保证成立。
9. **代码版本本身提示需复核。** 当前仓库最后一次相关提交主题是
   “修复bug时，引入新bug”；上述发现因此不仅是风格问题，应在硬件回归前优先核查。

## 16. 推荐阅读顺序

1. 公共Ultra DTSI后再读具体`edge10_ultra_*die0*prod*.dts`的`&pcie0`覆盖；
2. `pcie-edge.h:26-345, 391-532`：寄存器和对象；
3. `pcie-edge-plat.c:38-132, 377-523`：时钟复位与 RC/EP分叉；
4. `pcie-edge-ep.c:1208-1465`：EPC建立；
5. `pci-epf-edge.c` + `pci-ep-cfs.c:44-124`：Function与启动；
6. `pcie-edge-ep.c:636-1128`：header/BAR/outbound/IRQ/start；
7. `pcie-edge-ep.c:285-635`：notify；
8. `pcie-edge-host.c:80-132, 387-453, 789-926`：RC enumeration；
9. `pcie_irq_relay.c`：最后再看多 die relay，否则很容易迷路。

## 17. Source Reference

- X6000 controller：`drivers/pci/controller/pcie-edge.[ch]`
- SoC glue：`drivers/pci/controller/pcie-edge-plat.c`
- EPC：`drivers/pci/controller/pcie-edge-ep.c`
- RC：`drivers/pci/controller/pcie-edge-host.c`
- EPF：`drivers/pci/endpoint/functions/pci-epf-edge.c`
- Endpoint core/configfs：`drivers/pci/endpoint/pci-epc-core.c`、
  `pci-epf-core.c`、`pci-ep-cfs.c`、`pci-epc-mem.c`
- IRQ relay：`drivers/pci/controller/pcie_irq_relay.c`
- DTS：公共`hsio/edge10_ultra_hsio_pcie_card.dtsi`与最终
  `edge10_ultra/edge10_ultra_*die0_a55_prod*_unconverted.dts`
- Host 对照：`code_studies/edge_driver_x6000.md`

本文是静态源码研究，尚未启动 kernel、configfs 或硬件 Link。所有“当前运行配置”
结论必须以最终 DTB、kernel config、rootfs启动脚本和板上寄存器 dump 再验证。
