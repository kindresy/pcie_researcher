# Linux PCI Function Driver：从 probe 到一笔真实 `writel()`

## 1. 一个现代 driver 的骨架

```c
static int demo_probe(struct pci_dev *pdev,
		      const struct pci_device_id *id)
{
	int ret;

	ret = pcim_enable_device(pdev);
	if (ret)
		return ret;

	ret = pcim_request_region(pdev, 0, "demo-bar0");
	if (ret)
		return ret;

	pci_set_master(pdev);

	/* 选择合适 DMA mask 后，再分配/映射 DMA buffer。 */
	/* 映射 BAR，初始化硬件，分配 IRQ，最后开放业务。 */
	return 0;
}

static const struct pci_device_id demo_ids[] = {
	{ PCI_DEVICE(0x1234, 0x5678) },
	{ }
};
MODULE_DEVICE_TABLE(pci, demo_ids);

static struct pci_driver demo_driver = {
	.name = "demo",
	.id_table = demo_ids,
	.probe = demo_probe,
	.remove = demo_remove,
};
module_pci_driver(demo_driver);
```

这里只展示生命周期顺序，不把错误清理、DMA/IRQ 全塞进一屏代码。真实驱动应优先使用目标 kernel 支持的 managed helper，并逐项检查返回值。

## 2. `pci_enable_device()` 到底在启用什么

调用前：

- `pci_dev` 已由 PCI core 枚举；
- BAR resource 通常已分配/记录；
- Function 可能不在 D0；
- PCI Command 的 I/O Space / Memory Space decode 可能没开。

公共语义是让设备回到可操作 power state，并按已分配 resources 启用需要的 I/O/Memory decode；它有引用计数，header 中 `enable_cnt` 可证实这一点。

它**不会自动**：

- 设置 Bus Master Enable；那是 `pci_set_master()`；
- claim BAR；那是 `pci_request_regions()`；
- ioremap BAR；那是 `pci_iomap()`/managed mapping；
- 启用设备内部 DMA queue/interrupt source。

现代 6.x 精确内部调用链需要 `drivers/pci/pci.c`，当前只有 header，故不写伪行号。

## 3. `pci_set_master()` 与 Bus Master Enable

它设置 PCI Command.Bus Master Enable（BME），允许 Function 主动发起 Memory/IO transaction。对 Endpoint DMA 来说这是协议闸门：BME=0 时 client 不得正常向 Host Memory 发 MRd/MWr。

它不启动某个 DMA engine。driver 仍需准备 buffer/descriptor，写 device-specific DMA registers/doorbell。

## 4. `pci_request_regions()`

它把该 Function 的 BAR resources claim 给 driver，进行资源冲突检查。它不：

- 发送 BAR sizing transaction；枚举早已做过；
- 改写 BAR address；
- 返回 CPU pointer；
- 控制 EP inbound ATU。

失败常表示资源已被占用、无效或另一个 driver/firmware owner 没释放，不能硬着头皮继续映射。

## 5. `pci_iomap()` 返回什么

`pci_iomap(pdev, bar, maxlen)` 根据 BAR resource 类型创建适合 architecture 的 I/O mapping，返回 `void __iomem *` **CPU virtual I/O cookie/address**。

它不是：

- BAR 中保存的 PCI bus address；
- CPU physical address；
- DMA address；
- Endpoint 内部 AXI/APB address。

完整路径：

```text
void __iomem *bar0
 → CPU I/O virtual mapping
 → host CPU physical MMIO resource
 → Host Bridge outbound aperture
 → PCIe address = assigned BAR base + offset
 → Memory Read/Write TLP
 → EP Function/BAR hit
 → Cadence inbound PCIe-to-AXI translation
 → device register/SRAM
```

本机 `include/asm-generic/pci_iomap.h` 注释明确称其为 “Create a virtual mapping cookie for a PCI BAR”。

## 6. `readl()` / `writel()` 的同步边界

`writel(value, bar + off)` 让 CPU 执行 MMIO store，最终通常成为 Posted MWr；没有 PCIe Completion。体系结构 accessor 还处理必要 endian/compiler/I/O ordering，驱动不应把 `__iomem` 转普通 pointer 直接解引用。

若必须确认 posted write 已越过某些 bridge/device buffer，常用同一设备安全寄存器的 read-back 触发 Non-Posted MRd/CplD 形成 ordering/flush 点；但寄存器必须允许读、并符合设备手册，不能盲读 write-only doorbell。

## 7. DMA API 与 BAR MMIO 是两条地址路径

```text
BAR MMIO:
CPU → RC → EP register

EP DMA:
EP requester → RC/IOMMU → Host buffer
```

`dma_alloc_coherent(&pdev->dev, ..., &dma_handle, ...)`：返回 CPU pointer + device 可用 DMA address；`dma_map_single()` 把现有 CPU buffer 映射为 DMA address，并建立 cache/IOMMU 语义。descriptor 必须写 `dma_handle`，不能写 `virt_to_phys()` 猜出来的地址。

## 8. IRQ 顺序

推荐先：

1. 清设备 pending status；
2. 分配 MSI/MSI-X vectors；
3. `request_irq()` 注册 handler；
4. 最后打开 device-specific interrupt enable。

remove/suspend 则反向：先阻止新业务/关设备源并同步 handler，再释放 vector。否则 device 可能在 handler 拆掉后还发最后一笔 MSI write。

## 9. remove 和错误路径

非 managed resource 的释放通常按获取逆序：

```text
stop queues / block new I/O
 → disable device interrupt source
 → synchronize/free IRQ
 → stop DMA and wait ownership returned
 → unmap/free DMA buffers
 → iounmap BAR
 → release regions
 → clear bus master / disable device as appropriate
```

热拔时 MMIO 可能已不可达；remove 不能假设每次 read 都会回来。error recovery callback 与 ordinary remove 的硬件可用性边界也不同。

## 10. Source Reference 与证据等级

- **Confirmed declarations/comments**：Ubuntu 6.8 headers `include/linux/pci.h` 与 `include/asm-generic/pci_iomap.h`。
- **Confirmed hardware behavior**：Base Spec 5.0 Command/BAR/Memory transaction；Cadence address translation。
- **Unknown exact Linux implementation**：缺 `drivers/pci/*.c` 和目标 architecture/kernel commit。
- **Historical cross-check**：王齐《PCI Express 体系结构导读》第 14 章使用约 Linux 2.6.31 的代码展示 `__pci_enable_device_flags → do_pci_enable_device → pcibios_enable_device` 和 x86/PowerPC host initialization。该调用链只帮助理解职责分层，不视为现代 6.8 证据；printed pp.365–416 / PDF pp.377–428。
