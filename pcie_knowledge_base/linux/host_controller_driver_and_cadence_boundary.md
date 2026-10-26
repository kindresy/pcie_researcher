# Linux PCI Host Controller Driver：把 SoC 窗口包装成通用 PCI 世界

## 1. Function driver 与 Host controller driver 的职责边界

| Driver | 管什么 | 典型输入 |
|---|---|---|
| Host controller driver | Cadence RP、PHY、clock/reset、ECAM/ATU、MSI/INTx、host windows | Device Tree/ACPI + SoC registers |
| Function driver | 枚举出来的一张网卡/NVMe/自研 EP Function | `struct pci_dev` + BAR/DMA/IRQ |
| Endpoint controller driver | SoC 在 EP 模式时管理 EPC hardware | EP framework + controller registers |
| Endpoint function driver | 在 EP SoC 上实现一个可被 Host 枚举的功能 | EPC/EPF APIs + BAR/MSI/DMA |

当前工程已经包含 X6000 Linux 5.10.4 的 SoC glue、RC、EPC 和 EPF 源码。下面先解释通用职责，再把它们落到 `pcie-edge-plat.c`、`pcie-edge-host.c`、`pcie-edge-ep.c` 和 Endpoint Framework；更完整的目标代码分析见 [`linux_x6000_pcie_stack.md`](../code_studies/linux_x6000_pcie_stack.md)。

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

在 X6000 RC 路径中，`edge_plat_pcie_probe()` 完成 PHY、clock、APB reset 和 controller 初始化，然后调用 `edge_pcie_host_setup()`。后者映射配置 aperture、控制 PERST#、启动并等待 Link、初始化地址转换和 INTx/MSI domain，最后设置 `bridge->ops = &edge_pcie_host_ops` 并调用 `pci_host_probe()`。firmware 是否预先留下额外状态仍要结合最终产品 DTB 和实机寄存器 dump 判断。

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

那么 `pci_ops.read/write` 必须处理 serialization、absent device/timeout、root bus self config、byte enables 与 region restore。若并发的枚举、AER 或 PM 配置访问争用同一个 region，目标 BDF 和返回数据可能不再对应，因此共享窗口必须受到锁和恢复逻辑保护。

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

X6000 的调用链已经可以确认：EPF 通过 `pci_epc_raise_irq()` 进入 `edge_pcie_ep_raise_irq()`；后者按 `PCI_EPC_IRQ_LEGACY/MSI/MSIX` 分派到 X6000 controller 实现。具体 MSI-X table、远端 relay 和 vector 约定见 `pcie-edge-ep.c` 及多仓库专题。实际产品通过 configfs 广告多少个 vector 仍缺部署脚本，因此数量保留为 `NEED_VERIFY`。

## 6. Source Reference

- X6000 Linux 5.10.4 `include/linux/pci.h`：`struct pci_host_bridge`/`pci_bus`。
- X6000 RC：`drivers/pci/controller/pcie-edge-plat.c`、`pcie-edge-host.c`、`pcie-edge.c`。
- X6000 EP：`drivers/pci/controller/pcie-edge-ep.c`、`drivers/pci/endpoint/pci-epc-core.c`、`drivers/pci/endpoint/functions/pci-epf-edge.c`。
- Cadence Chapters 4、8–11：RP/EP、address translation、reset/link、interrupt。
- 目标源码固定为 `x6000 @ e74fd0e7746a`；外部 Host 服务器所用内核、IOMMU 和 MSI remapper 不在该仓库证据范围内。
