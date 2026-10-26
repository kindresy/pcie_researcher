# Linux PCI Host Controller Driver：把 SoC 窗口包装成通用 PCI 世界

## 1. Function driver 和 Host driver 别认错工种

| Driver | 管什么 | 典型输入 |
|---|---|---|
| Host controller driver | Cadence RP、PHY、clock/reset、ECAM/ATU、MSI/INTx、host windows | Device Tree/ACPI + SoC registers |
| Function driver | 枚举出来的一张网卡/NVMe/自研 EP Function | `struct pci_dev` + BAR/DMA/IRQ |
| Endpoint controller driver | SoC 在 EP 模式时管理 EPC hardware | EP framework + controller registers |
| Endpoint function driver | 在 EP SoC 上实现一个可被 Host 枚举的功能 | EPC/EPF APIs + BAR/MSI/DMA |

当前资料只有 Cadence core guide，没有 SoC glue 或 Linux controller source，因此下面给的是应建立的接口映射和待确认点。

## 2. Host 初始化主线

```text
platform probe
 → acquire clocks/resets/PHY/regmaps/ranges/dma-ranges/IRQs
 → configure controller for Host/RC operation
      (the Cadence core presents a Root Port)
 → hold LINK_TRAINING_ENABLE=0
 → initialize PHY and Cadence Local Management/config defaults
 → program RC BAR/no-match and outbound config/I/O/memory regions
 → release training enable, wait LTSSM L0 + DL_Up
 → allocate/register pci_host_bridge
      ops = config read/write
      windows = CPU↔PCI resource apertures
      dma_ranges = PCI requester↔CPU memory mapping
      map_irq / MSI domain
 → scan/assign/add root bus
```

哪些步骤由 bootloader/firmware 预先完成、Linux 是 preserve 还是 reinitialize，必须看目标 platform driver。

## 3. `ranges` 与 `dma-ranges` 方向相反

- host bridge **ranges/windows**：CPU 要访问 PCI I/O/Memory Space，host outbound 怎样翻译；
- **dma-ranges**：PCI requester 发起 DMA 时，PCI bus address 怎样到 CPU physical domain。

两者都叫 address translation，却服务相反方向。再叠 IOMMU 后，EP descriptor 中 DMA address 还可能是 IOVA。

## 4. Config access 要特别小心共享 ATU

若不是纯 ECAM，而是每次 config access：

```text
program target bus/dev/function + Type0/Type1 region
 → access small CPU aperture
 → restore/reuse region
```

那么 `pci_ops.read/write` 必须处理 serialization、absent device/timeout、root bus self config、byte enables 与 region restore。并发扫描或 AER/PM config access 若抢同一个 region，会制造非常难看的幽灵 BDF。

## 5. Cadence EP mode 对应 Linux Endpoint Framework

概念映射：

```text
pci_epc driver
 → set_bar/clear_bar：配置 Function BAR aperture/inbound target
 → map_addr/unmap_addr：配置 outbound AXI→PCIe region
 → raise_irq：按 INTx/MSI/MSI-X 机制生成中断
 → start/stop：控制 LTSSM/link

pci_epf driver
 → 定义 vendor/device/class/capability
 → 分配 BAR backing memory
 → 实现业务 register/DMA/test function
```

Cadence 当前手册显示 MSI/MSI-X 需要 client logic 构造 AXI MWr，因此 `raise_irq` 的具体 EPC 实现可能要读 capability/table 并编 region，或由 SoC wrapper 提供快捷路径；目标代码缺失，标 **Unknown**。

## 6. Source Reference

- Ubuntu 6.8 `include/linux/pci.h`：`struct pci_host_bridge`/`pci_bus`。
- Cadence Chapters 4、8–11：RP/EP、address translation、reset/link、interrupt。
- Linux EPC/EPF 精确接口：当前 workspace 未提供 `include/linux/pci-epc.h`、`pci-epf.h` 和实现，待补 kernel tree。
