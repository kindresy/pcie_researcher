# X6000 PCIe 统一地址地图：Host BAR、Cadence aperture、APT 与多 die

> 这是一份“拿地址来查”的工程文档。第一次阅读仍建议先看
> [BAR 与地址翻译原理](../addressing/bar_and_cadence_address_translation.md)和
> [启动交接](x6000_boot_handoff_and_ownership.md)。

## 1. 同一笔访问里至少有四种地址

假设 Host Linux 把 X6000 BAR0 分配到 `0x80000000`，进程通过驱动映射后执行：

```c
writel(0x12345678, bar0 + 0x400018);
```

这条语句里同时存在：

| 地址域 | 示例 | 谁理解它 |
|---|---:|---|
| Host CPU virtual address | `bar0 + 0x400018` | Host进程/内核页表 |
| Host physical PCI resource | `0x80400018` | Host RC/bridge window |
| PCIe TLP address | 通常仍为 `0x80400018` | RC、Switch、EP BAR comparator |
| EP BAR-relative offset | `0x400018` | X6000 Function BAR decode |
| Cadence inbound aperture | `0x3f00400018` | Endpoint controller |
| 最终 SoC AXI address | runtime为 `0x58400018` | APT、AXI-config register target |

它们数值偶尔相同只是平台配置结果，不是 PCIe 的保证。尤其不要把 `pci_iomap()` 返回的
Host virtual address叫“PCIe地址”，也不要把 `dma_map_sg()` 返回的 DMA address 默认
当Host物理地址；IOMMU一加入，后者通常就是IOVA。

## 2. X6000 inbound 是两级翻译

```mermaid
flowchart LR
    A[Host BAR address] --> B[EP Function BAR hit]
    B --> C[BAR-relative offset]
    C --> D[Cadence inbound base<br/>BAR0 0x3f... / BAR2 0x3d...]
    D --> E[SoC APT region match]
    E --> F[local AXI target]
```

第一层回答“这是哪个 Function 的哪个 BAR”；第二层回答“这个 BAR offset 最终接到
芯片内部哪块 SRAM、寄存器或远端窗口”。BAR 是门牌，APT 是门后的走廊布局。

## 3. 启动公共 mailbox

BootROM、PCS、BL2 都把 Host 可见的 BAR0 低 512 KiB 指向 AP SRAM `0x60000000`，
因此下面的低位协议能跨阶段复用：

| BAR0 offset | EP local | 语义 |
|---:|---:|---|
| `0x7fa00` | `0x6007fa00` | H2D ready |
| `0x7fa40` | `0x6007fa40` | DE10/DE11/DE12/DE14 stage magic |
| `0x7fa80` | `0x6007fa80` | D2H ready |
| `0x7fb00` | `0x6007fb00` | H2D packet buffer |
| `0x7fd80` | `0x6007fd80` | D2H packet buffer |

Host 写 packet buffer 和 ready flag 都是 Posted Memory Write；Host 轮询 ready 则是
Memory Read，需要 Completion with Data。软件握手弥补了 Memory Write 没有协议级
Completion 这一点。

## 4. PCS 阶段 BAR0 地图

PCS 把 BAR0 inbound base 设为 `0x3f00000000`，再配置九条 APT：

| Region | BAR0 offset | 最终 SoC 地址 | size | 用途 |
|---:|---:|---:|---:|---|
| 0 | `0x000000` | `0x60000000` | 512 KiB | AP SRAM/mailbox |
| 1 | `0x080000` | `0x460000000` | 512 KiB | die/片上窗口 |
| 2 | `0x100000` | `0x1c60000000` | 512 KiB | 多die窗口 |
| 3 | `0x180000` | `0x2060000000` | 512 KiB | 多die窗口 |
| 4 | `0x200000` | `0x5a200000` | 4 KiB | PCIe system control |
| 5 | `0x201000` | `0x5a216000` | 8 KiB | PHY/aux control |
| 6 | `0x203000` | `0xa7400000` | 4 KiB | eFuse |
| 7 | `0x760000` | `0x240000000` | 128 KiB | 共享/保留窗口 |
| 8 | `0x780000` | `0x242000000` | 512 KiB | 共享/保留窗口 |

例如 Host 写 `BAR0+0x80000`：BAR命中后先成为 `0x3f00080000`，再被region 1
翻到 `0x460000000`。

## 5. Device Linux runtime Ultra 地图

公共能力模板是`hsio/edge10_ultra_hsio_pcie_card.dtsi`，但运行期权威来源是最终
`edge10_ultra/*die0*prod*.dts`对`&pcie0`的覆盖。Device Tree同名property不是数组
追加；后者会替换公共DTSI的整个`apt-ranges`。

### 5.1 普通A组die0最终布局

| Region | BAR0 offset | local AXI target | size | Host侧用途 |
|---:|---:|---:|---:|---|
| 9 | `0x400000` | `0x58400000` | `0x210000` | AXI config + uDMA连续块 |
| 13 | `0x800000` | `0x2a0000000` | 2 MiB | group 0控制窗 |
| 14 | `0xa00000` | `0x6a0000000` | 2 MiB | group 1控制窗 |
| 15 | `0xc00000` | `0x1ea0000000` | 2 MiB | group 2控制窗 |
| 16 | `0xe00000` | `0x22a0000000` | 2 MiB | group 3控制窗 |

Host `edge.ko` 中的这些关键常量可以追溯到明确的地址窗口：

```text
BAR0+0x400000 → local 0x58400000 AXI config/ATU
BAR0+0x600000 → local 0x58600000 uDMA
BAR0+0x800000 → group 0 window（Host REG_AXI_CFG_ADDR使用的远端/outbound视图）
```

### 5.2 BAR2：四个大数据窗口

| Region | BAR2 offset | local AXI target | size |
|---:|---:|---:|---:|
| 28 | `0x0000000` | `0x2b0000000` | 16 MiB |
| 29 | `0x1000000` | `0x6b0000000` | 16 MiB |
| 30 | `0x2000000` | `0x1eb0000000` | 16 MiB |
| 31 | `0x3000000` | `0x22b0000000` | 16 MiB |

Host进程 `mmap()` BAR2 后执行 `memcpy()`，是 Host CPU 发起 MMIO Read/Write TLP；
它与 uDMA 无关。`xferstress` mode 1/6 正是在测这条路径。

普通A组还用BAR4/region10把64 MiB映到`0x21f8000000`。普通B组的BAR2首region
是12而非28；LM只保留group0/BAR2/BAR4必要窗口，DS也有自己的缩减表。

| 最终DTB | BAR0 group窗 | BAR2 | BAR4 |
|---|---|---|---|
| 普通 A | region13–16，四组 | region28–31，四个16 MiB | region10→`0x21f8000000`, 64 MiB |
| 普通 B | region13–16，四组 | 首窗region12，其余29–31 | 未列 |
| LM A | 只有region13/group0 | region28/group0 | region10→`0x21f8000000`, 64 MiB |
| LM B | 只有region13/group0 | region12/group0 | 未列 |
| DS A | region13/group0 | region28→group0、region29→A8 16 MiB | regions10–12，三段`0x15000000`大窗 |
| DS B | region13/group0 | region12/group0 | 未列 |

以上六种都保留region9的`BAR0+0x400000 → 0x58400000, size 0x210000`，所以
本地AXI-config/uDMA控制块是共同底座；产品差异主要落在大窗和远端group可见性。

最终DTS没有重列mailbox region0，而EPC `set_bar()`也不会预先清空APT，所以AP SRAM
等低offset依赖固件阶段留下的region。最终地址图是“Linux写入的条目 + 固件有意保留
的条目”，不是只把某个DTS表打印出来就结束。

不同产品（DS/LM/VS）对group B或动态APT的选择仍有额外策略：DS通常按16 MiB，LM
可偏到64 MiB，VS会读取运行期region target反算offset。工具必须先识别产品，不能把
BAR2 offset 0当万能钥匙。

## 6. Host BAR resource 与EP local target怎样闭环

以下以Host给BAR0分到`0x80000000`为例。

### 6.1 读 boot magic

```text
edge10_tool bario read 0x7fa40
 → ioctl READ_BAR
 → edge.ko readl(host_bar0_va + 0x7fa40)
 → Host Memory Read TLP address 0x8007fa40
 → EP BAR0 hit, offset 0x7fa40
 → aperture 0x3f0007fa40
 → APT region0 → AP SRAM 0x6007fa40
 → CplD returns DE12DE12（例如BL2阶段）
```

### 6.2 启动 uDMA channel

```text
edge.ko writel(BIT(ch), BAR0+0x600xxx)
 → Host Memory Write TLP
 → EP BAR0 region9（offset 0x400000起）
 → local 0x586xxxxx uDMA register
 → Cadence uDMA读取descriptor并发起outbound PCIe request
```

这是一条很有意思的“请求产生请求”路径：Host先发一笔MMIO Memory Write去按uDMA
门铃，随后EP uDMA作为Requester再向Host memory发出Memory Read或Memory Write。

## 7. EP outbound：DMA地址不是BAR地址

当EP访问Host memory时，方向反过来：

```text
Host dma_map_sg()/dma_alloc_coherent
 → Host DMA address / IOVA
 → 写进uDMA descriptor ext_addr
 → Cadence outbound region把EP CPU aperture映到该PCI/DMA address
 → uDMA AXI request
 → PCIe Memory Read/Write TLP
 → optional Host IOMMU translation
 → Host RAM
```

这里不经过EP BAR。BAR服务于“Host访问EP”；outbound ATU服务于“EP主动访问外面”。
两者都叫地址翻译，却是相反方向。

## 8. A8 RC → B0 EP 的C2C地址图

BL2对固定一对一C2C链路执行最小枚举：Config Type 0探测、BAR sizing、固定地址分配、
置MSE/BME，再建立以下A8 CPU→B0 PCI窗口：

| outbound region | A8 CPU aperture | B0 PCI target | size |
|---:|---:|---:|---:|
| 0 | `0x21fff00000` | config `0x2dfff00000` | 4 KiB |
| 1 | `0x21fc000000` | BAR0 `0x2dfc000000` | 16 MiB |
| 2 | `0x21f8000000` | BAR2 `0x2df8000000` | 64 MiB |
| 3 | `0x2100000000` | BAR4 `0x2e00000000` | 2 GiB |
| 4 | `0x2180000000` | continuation/window | 1 GiB |

A0 BAR1还能把Host访问接到A8的`0x21fc000000`，于是形成：

```text
Host Memory Request → A0 BAR1 → A8 CPU aperture
 → A8 RC outbound Memory TLP → B0 BAR0
 → B0 inbound aperture/APT → B0 local target
```

这是一条PCIe-over-PCIe的桥接路径。每一跳的地址都合法，但绝不属于同一地址域。

## 9. C2C APT的功能分区

BL2先清32个region，再按产品/位置组合：

| Region | offset/范围 | target类别 | 典型用途 |
|---:|---:|---|---|
| 0–3 | `0x0–0x1fffff` | 四个die AP SRAM | mailbox/boot state |
| 9 | `0x00400000` | `0x58400000`, size `0x210000` | ATU/uDMA control |
| 11 | `0x00380000` | `0xa6009000` | CoreSight timestamp |
| 14–17 | `0x0ec00000...` | 各die `0xa8000000` | AISS小窗 |
| 18–21 | `0x0f000000...` | 各die `0xb0800000` | AISS窗口 |
| 22–25 | `0x10000000...` | 各die `0x50000000` | 64 MiB设备窗 |
| 26–29 | `0x20000000...` | 多die高地址 | 128 MiB窗 |
| 30–31 | `0x40000000/0x80000000` | 邻接device/group | 1 GiB窗 |

这些region号是X6000 SoC集成协议，不是PCIe Base Spec规定。Spec只要求BAR decode、
TLP路由和Completion等线上语义，并不知道芯片内部还有一位叫APT的门卫。

## 10. Debug时怎样避免地址域事故

1. 先记录当前阶段 magic；没有阶段，offset就没有完整语义。
2. 用 `lspci -vv` / sysfs确认Host BAR resource，不要把示例`0x80000000`当实机值。
3. 区分BAR-relative offset、EP aperture与final AXI target，三列分别记录。
4. BAR读全1先查Link/Completion/译码，不要把`0xffffffff`直接当有效寄存器值。
5. 修改动态APT前锁住所有访问者，并在更新前后建立协议静默/屏障。
6. DMA失败时另记Host DMA address/IOVA；它不在BAR表里。
7. 多die失败要写清A0/A8/B0和group，不要只写一句“PCIe地址错了”。

## 11. Source Reference

- PCS `board/edge10/drivers/pcie/pcie_init.c`：BAR0 inbound与APT 0..8。
- ATF `drivers/intellif/pcie/pcie_init.{c,h}`：A8/B0 config、BAR、outbound和C2C APT。
- Linux公共Ultra DTSI及`edge10_ultra/*die0*prod*.dts`最终覆盖：runtime APT；
  `drivers/pci/controller/pcie-edge{,-ep}.c`：解析和编程。
- Host edge-driver `pcie/edge.c`、`pcie/edge.h`：BAR mapping、MMIO、outbound与uDMA。
- Host tool HAL `host/communication/pcie/hal_pcie.c`：BAR2 mmap与产品窗口选择。
