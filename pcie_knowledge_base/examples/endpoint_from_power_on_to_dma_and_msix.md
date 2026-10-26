# 一个 Endpoint 的一天：从 PERST# 到 DMA 完成和 MSI-X

这是一台虚构但协议上自洽的设备。我们用同一组 BDF、BAR 和地址走完整条链，免得每章换一套数字，读到最后像接手了三块不同的板子。

## 0. 场景和地址表

```text
Root Port                 00:01.0
Endpoint                  01:00.0
EP Vendor/Device ID       17cd:0100（仅作例子）
EP BAR0 requirement       64 KiB, 32-bit, non-prefetchable
EP BAR2 requirement       64 KiB, MSI-X table/device SRAM

Host ECAM CPU base        0x4000_0000
Host CPU MMIO window      0x4_8000_0000 ...
Window maps to PCI addr   0x8000_0000 ...
BAR0 PCI / CPU physical   0x8000_0000 / 0x4_8000_0000
BAR2 PCI / CPU physical   0x8001_0000 / 0x4_8001_0000
```

这里故意让 CPU physical 与 PCI address 不相等。现实平台可能恰好相等，但把“可能相等”背成“按定义相等”，迟早会被 host bridge window 教育。

## 1. 上电：先有电气世界，再有 BDF

最初还没有 `01:00.0` 这个身份：refclk/power/reset/PHY 按平台时序准备；EP 配好 Configuration Space 模板与 BAR aperture；Cadence RP 在 training disabled 时写有效的 Gen3/4 preset、LM 与必要 region；两端 LTSSM 再从 Detect 经 Polling/Configuration 进入 L0，Data Link 对 VC0 完成 InitFC。

此时只说明 Link 能传包。EP 还没被 OS 命名、BAR 还没分配、驱动也没 probe。Link Up 像楼通电了，不代表每家公司都领到了门牌和钥匙。

## 2. 枚举：`01:00.0` 怎样出现

系统先给 Root Port Type 1 header 写：

```text
Primary Bus=0, Secondary Bus=1, Subordinate Bus=1
```

随后访问 EP offset 0。ECAM CPU 地址为：

```text
0x4000_0000 + (1 << 20) + (0 << 15) + (0 << 12) + 0
= 0x4010_0000
```

```text
CPU ECAM read
  → host bridge config path
  → Type 0 Configuration Read, target 01:00.0 offset 0
  → EP config decode
  → CplD 返回 Vendor/Device ID
  → Linux 创建/填充 struct pci_dev
```

若 Vendor ID 返回 `0xffff`，通常表示 Function 不存在或访问没成功；不是 EP 很有创意地申请了全 1 Vendor ID。

### BAR size probing

Linux/固件关闭相关 decode、保存原值，向 BAR0 写全 1 再读回。对 64 KiB 32-bit Memory BAR：

```text
readback mask   0xffff_0000（忽略 flags）
size            ~(0xffff_0000) + 1 = 0x1_0000
```

系统把 BAR0 放在 PCI address `0x8000_0000`，BAR2 放在 `0x8001_0000`，同时确保 Root Port Memory Base/Limit 覆盖它们。Cadence EP 此后可用 Function+BAR hit，再把 offset 翻到本地 AXI target。

## 3. Linux probe：API 各铺哪一段路

```c
pci_enable_device(pdev);
pci_request_regions(pdev, "demo");
regs = pci_iomap(pdev, 0, 0);
dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));
pci_set_master(pdev);
nvec = pci_alloc_irq_vectors(pdev, 1, 8,
                             PCI_IRQ_MSIX | PCI_IRQ_MSI);
request_irq(pci_irq_vector(pdev, 0), demo_irq, 0,
            "demo", priv);
```

- enable 使 Function/address decode 可用，精确内部链依目标 kernel；
- request regions 声明驱动独占 BAR；
- iomap 返回 CPU 可用的 `__iomem` virtual mapping；
- DMA mask 描述设备寻址能力，set master 设置 BME；
- IRQ API 让 PCI/MSI core 分配并编程消息，不由驱动手拧 capability。

本例 `pci_iomap(BAR0)` 最终对应 CPU resource `0x4_8000_0000` 一带，不是把 PCI address `0x8000_0000` 直接当普通指针。

## 4. 一次 `writel()`：Host 敲 EP doorbell

```c
writel(0x12345678, regs + 0x100);
```

```text
CPU __iomem VA
 → CPU physical 0x4_8000_0100
 → RC outbound window
 → PCI address 0x8000_0100
 → 3DW MWr, Length=1 DW, BE=1111/0000
 → EP BAR0 hit, offset 0x100
 → Cadence inbound AXI Master write
 → device doorbell register
```

Requester ID 由 RC requester/path 生成，不是目标 EP 的 BDF。MWr 是 Posted Request，没有正常 Completion；Data Link ACK 也只证明下一跳收到，不能证明业务状态机已经处理 doorbell。

## 5. Linux 准备 Host DMA buffer

```c
ring_cpu = dma_alloc_coherent(&pdev->dev, 4096,
                              &ring_dma, GFP_KERNEL);
```

假设结果是：

```text
CPU virtual            ffff....          // 只给 CPU 解引用
DMA address / IOVA     0x4000_0000       // 写进 EP descriptor
IOMMU mapping          0x4000_0000 → PA 0x12_3400_0000
```

驱动写 `ring_dma`、length、ownership，执行相应 barrier，最后敲 doorbell。若把 `ring_cpu` 填进 descriptor，EP 会对内核虚拟地址发 TLP；硬件不会因为这个指针长得很像地址就自动理解 Linux 页表。

## 6. EP DMA Read：从 Host 取 256 B

EP `01:00.0` 发一笔 aligned MRd：

| 字段 | 示例 |
|---|---|
| Fmt/Type | `001b/00000b`，4DW MRd |
| Address | `0x0000_0000_4000_0000` |
| Length | 64 DW = 256 B |
| First/Last BE | `1111b / 1111b` |
| Requester ID | `01:00.0` → `0x0100` |
| Tag / TC | `0x2a` / `0` |

```text
EP DMA/local AXI requester
 → Cadence outbound AXI Slave region
 → MRd, RID=0x0100, Tag=0x2a, IOVA=0x4000_0000
 → Root Port / IOMMU permission+translation
 → Host PA 0x12_3400_0000
 → memory/cache subsystem
```

假设 Completion payload 上限和 RCB 都为 128 B，这笔请求返回两个 CplD。两条都复制 Requester ID、Tag、TC；第一条 Byte Count=256，第二条 Byte Count=128。Cadence 用 Split Completion Table 匹配、检查并重组成本地 AXI read response。

若 IOMMU 没映射，平台会记录 fault；EP可能收到 UR/CA 或最终 CTO，具体映射由 RC/IOMMU 实现决定。只盯 EP descriptor 会错过真正的门卫。

## 7. EP DMA Write 与 ordering

设备把采样数据写到 `0x4000_0100`：

```text
local SRAM → DMA → Cadence outbound AXI Slave
 → one or more MWr（受 MPS/4 KiB boundary约束）
 → RC/IOMMU → Host memory
```

MWr 没有 CplD。设备在数据写后发 MSI-X 时，仍须靠 ordering、barrier 和 descriptor ownership 保证 Host 不会先处理“完成中断”，后看到半份旧数据。Coherent memory 不能替代 ordering。

## 8. MSI-X：最后一笔 4 B 写通知 CPU

Host 已在 BAR2 MSI-X entry 0 写入一组平台决定的值：

```text
Message Address    0x0000_0000_fee0_1000（纯示例）
Message Data       0x0000_0045
Vector Mask        0
```

设备完成 DMA 后发 4DW MWr，Length=1 DW，address=`0xfee0_1000`，payload=`0x45`，Requester ID=`0x0100`：

```text
EP interrupt condition
 → MSI-X entry address/data/mask
 → Cadence outbound MWr
 → Root interrupt controller/remapper
 → Linux IRQ number → demo_irq()
 → driver 读 status，回收 DMA buffer
```

MSI-X 仍是 Posted MWr，不返回普通 Completion。Linux IRQ number 也不等于 Message Data；前者是内核抽象，后者是平台写入设备的硬件消息。

## 9. 从“能搬一次”升级到“像个真正设备”

上面的单笔例子适合看清 packet，却不是高性能 endpoint 的终点。王齐教材第 12 章那块 Capric FPGA 卡恰好展示了一个很诚实的反面教材：每次由 Host 写地址/长度/start，设备只做一笔 DMA，完成后发一个 MSI，ISR 再跨 PCIe 读 status。能跑，但 Link latency 每轮都来收过路费。

真实设计通常继续做四件事：

1. 用 host-memory descriptor ring 一次提交多笔工作，doorbell 只通知 producer index；
2. 同时保留多个 MRd Tag，让读延时被其他 outstanding request 覆盖；
3. 按 Requester ID + Tag 接收并归位可能交错返回的 CplD，直到 Byte Count/Lower Address 表明各 request 收齐再释放 Tag；
4. 用多个 MSI-X vector 或 DMA completion record 表达完成原因，尽量避免 ISR 为了“你叫我干吗”再做一次高延时 MMIO Read。

这里有一笔不能省的硬件账：多发 outstanding MRd 意味着接收端必须为最坏情况下的 Completion header/data、Tag context 和 backpressure 准备资源。Tag 数量不是免费的性能旋钮；Cadence 的 two-stage Completion buffer 和限流规则正是在付这笔账。

来源：王齐《PCI Express体系结构导读》Chapter 12，尤其 §§12.2.2、12.4；Cadence §11.4 Completion buffer integration。

## 10. 如果中途 Link Down

- LTSSM 离开 L0，链路不能继续送 TLP；
- Cadence uDMA descriptor 得到 Link Down/error；
- AXI bridge 的 Link Down Indication 使新 outbound access 返回 SLVERR；
- Function 非 sticky config 可能 reset，MSE/BME/MSI-X enable 要重查；
- AXI translation 明确保留；LM 是否保留存在手册冲突 OQ-011；
- Linux 可能进入 AER/hotplug/controller recovery，精确链需目标 kernel。

恢复应是 quiesce → 收集错误 → 识别 reset scope → retrain/re-enumerate或restore → 重建 DMA/IRQ ownership → 恢复流量，不是“再敲一次 doorbell 碰碰运气”。

## 11. 延伸阅读与来源

- PCI Express Base Specification 5.0：§§2.2–2.3、6.1.4、7.5、7.7。
- Cadence PCIe Core User Guide：§§4.5–4.7、Chapter 5、9、10、§11.4。
- 王齐《PCI Express体系结构导读》：Chapter 12 Capric FPGA endpoint实例（用作实现直觉和年代校正，不照搬Linux 2.6代码）。
- [Configuration Space、ECAM 与路由](../configuration/configuration_space_ecam_and_routing.md)
- [BAR 与 Cadence 地址翻译](../addressing/bar_and_cadence_address_translation.md)
- [Memory TLP 与 Completion](../transactions/memory_tlp_and_completion.md)
- [PCIe DMA 与 Linux DMA API](../dma/pcie_dma_cadence_and_linux_dma_api.md)
- [INTx、MSI 与 MSI-X](../interrupts/intx_msi_msix_and_cadence.md)
- [Cadence Reset 与恢复矩阵](../controller/cadence_reset_and_recovery_matrix.md)
