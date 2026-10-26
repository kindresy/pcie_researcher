# Linux PCIe Debug 工具：每个命令到底在读哪一层

## 1. `lspci` 不是直接“问设备名字”

`lspci` 主要从 Linux PCI sysfs/proc/config access 获取 `pci_dev` 和 Configuration Space，再用本地 ID database 把 vendor/device number 翻成名字。

常用：

```bash
lspci -D
lspci -t
lspci -nn -s 0000:01:00.0
lspci -vv -s 0000:01:00.0
lspci -xxxx -s 0000:01:00.0
```

- `-D` 保留 domain，避免多 segment 系统 BDF 重名；
- `-t` 看 bus/bridge hierarchy，适合检查 Secondary/Subordinate bus 范围；
- `-nn` 同时显示名字和原始 vendor/device ID；
- `-vv` 解 capability、BAR resource、Command/Status、Link/MSI/AER；
- `-xxxx` dump 4 KB config space，需要权限且对某些故障设备读取可能有副作用/超时风险。

### `lspci -vv` 关键字段回硬件

| 输出 | 来源 | 意义 |
|---|---|---|
| `Control: I/O+ Mem+ BusMaster+` | PCI Command | I/O/MSE/BME decode/initiator gate |
| `Region 0: Memory at ... [size=]` | BAR + Linux resource | 枚举后分配地址，不是 EP 内部 SRAM 地址 |
| `LnkCap` | PCIe Link Capabilities | Port 最大能力 |
| `LnkSta Speed/Width` | Link Status | 实际 negotiated 值；低于 capability 可能 downtrain |
| `MSI/MSI-X: Enable+` | Capability control | Host 已配置这种中断机制 |
| `AER` status/mask/severity | Extended Capability | 错误发生/屏蔽/严重度 |

## 2. `setpci`：Configuration Space 手术刀

```bash
setpci -s 0000:01:00.0 COMMAND
setpci -s 0000:01:00.0 CAP_EXP+12.w
setpci -s 0000:01:00.0 ECAP_AER+04.l
```

它通过 Linux config access 读写配置字段，最终可能产生 Config Read/Write TLP。写操作会真实改变 MSE/BME/ASPM/AER W1C 等硬件状态；先保存旧值和完整拓扑，并确认 field width/语义。

不要用 read-modify-write 想当然处理 RW1C register。也不要在 Linux driver 正管理设备时偷偷改 MSI/BAR/Command，kernel 的软件状态不会自动知道你在背后动过家具。

## 3. `dmesg` / kernel log

关注：

```bash
dmesg -T | grep -iE 'pci|pcie|aer|dpc|iommu|dm[a|r]|link'
```

它能串起 host probe、Link training、resource assignment、AER/DPC、driver probe、IOMMU fault。时间顺序很重要：一串 CTO 之前若先出现 Surprise Down，根因通常在 Link，而不是十个 Function 同时变坏。

日志是软件观测，不保证包含被 mask 的硬件 error，也不替代 controller local status/logic analyzer。

## 4. sysfs：Linux 对 PCI 树的对象化视图

```text
/sys/bus/pci/devices/0000:01:00.0/
  config        4 KB configuration access
  resource      每个 BAR/resource 的 start/end/flags
  resource0     BAR0 的 mmap interface
  enable        device enable reference/interface
  driver        当前绑定 driver symlink
  iommu_group   IOMMU isolation group
  irq           legacy/current IRQ view
  msi_irqs/     分配的 MSI/MSI-X Linux IRQs
  power/        runtime PM state/policy
  reset         若支持，Function reset interface
```

`resource` 中是 Linux CPU resource view；`config` 是 config bytes；`resource0` 是 BAR MMIO aperture。三者地址域不同，不要用一个 `dd` 命令把它们当同一文件。

## 5. `/proc/iomem` 与 IOMMU

`/proc/iomem` 显示 CPU physical resource tree，可检查 Host Bridge aperture 与 BAR CPU range 是否被 claim。它不显示 EP descriptor 中的 DMA address/IOVA translation。

IOMMU fault log 要看：Requester ID/BDF、IOVA、read/write permission、PASID（若有）。出现 fault 时对照 driver 给 device 的 `dma_addr_t`，不要只打印 CPU virtual/physical。

## 6. debugfs 与 trace

`/sys/kernel/debug/pci/` 并非所有 kernel/config 都有固定内容；具体节点由内核版本和 driver 决定。先 `mount -t debugfs none /sys/kernel/debug`（若系统 policy 允许），再只读盘点。

动态调试/tracepoint 可帮助观察 PCI core、PM、AER、IRQ/DMA，但名称随版本变化。目标 kernel source/config 未提供，本知识库不编造固定节点。

## 7. MMIO 访问工具的安全边界

`devmem`、直接 mmap `resourceN`、自写 BAR poke 工具可能：

- 触发 doorbell、清 W1C status、启动 DMA；
- 与正式 driver 并发访问；
- 对已 Link Down 设备造成 CPU external abort/timeout；
- 绕过 register ordering/locking。

先 unbind driver 也不自动让设备安全：DMA/interrupt 可能仍活跃。应有 device-specific quiesce 流程。

## 8. 现象到工具

| 现象 | 第一组证据 |
|---|---|
| 完全无 BDF | Root Port `lspci -vv`、LTSSM history、config TLP、host log |
| BDF 有但 probe 失败 | `dmesg`、driver bind、Command/BAR/resource、probe return |
| BAR read 全 1/timeout | MSE、resource/ATU、TLP UR/CA/CTO、EP BAR hit |
| DMA fault | BME、DMA mask/address、IOMMU fault、descriptor、MRd/MWr trace |
| MSI 不来 | MSI-X table/mask、`msi_irqs`、MWr address/data、`/proc/interrupts` |
| 性能低 | LnkSta speed/width、MPS/MRRS、replay/error rate、ASPM exit、credits |

## 9. Evidence boundary

这些接口是稳定的 Linux PCI 调试概念；具体文件权限、debugfs 节点、`lspci` 版本输出受目标发行版/kernel/pciutils 影响。当前环境有 kernel headers，但没有目标板 kernel source/runtime hardware，命令执行结果不能替目标系统证据。

