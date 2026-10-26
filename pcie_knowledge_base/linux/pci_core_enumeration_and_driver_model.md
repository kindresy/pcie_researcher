# Linux PCI Core：设备枚举、资源分配与驱动绑定

## 1. PCI Core 在 driver probe 之前做了什么

普通 PCI function driver 不负责训练 Link、遍历所有 BDF、给 BAR 分地址。Host controller driver 与 PCI core 先完成：

```text
Firmware/DT/ACPI 描述 Host Bridge
 → host controller clocks/reset/PHY/LTSSM
 → register pci_host_bridge + config access ops + resource windows
 → scan root bus
 → Config Read Vendor ID 探测 Device/Function
 → 读取 Type 0/1 header、class、capabilities
 → 遇 Bridge：分配/读取 bus numbers，递归扫描 subordinate bus
 → size/claim/assign BAR 与 bridge windows
 → 建立 pci_bus / pci_dev 对象树并注册 device model
 → PCI driver id_table 匹配
 → 调 driver probe(pdev, id)
```

所以 probe 收到 `struct pci_dev` 时，设备早已“被发现”；driver 的任务是把这个已枚举 Function 变成可工作的业务设备。

## 2. 三个核心对象

### `struct pci_host_bridge`

代表 CPU/SoC 进入一棵 PCI hierarchy 的根。X6000 Linux 5.10.4 的定义包含：

- root `pci_bus`；
- `pci_ops` configuration access；
- domain/starting bus number；
- host bridge resource windows 与 `dma_ranges`；
- INTx swizzle/map hook；
- MSI domain 与 native AER/PME/DPC/hotplug ownership 标志。

它正是“Cadence RP controller + SoC integration”交给通用 PCI core 的软件边界。Cadence core 只生成 Config TLP/Memory TLP；host driver 还要把 ECAM/ATU/IRQ/domain/resource 描述包装成这些 Linux 对象。

### `struct pci_bus`

代表一个逻辑 Bus：parent/children、该 bus 上的 devices、上游 bridge `self`、bus-number resource、可路由 address resources、configuration `ops`。

### `struct pci_dev`

代表一个 PCI Function，而不是整颗物理芯片。它保存 BDF/devfn、vendor/device/class、capability offsets、BAR `resource[]`、当前 power/error/enable 状态、MSI-X mapping、ATS/PASID/SR-IOV 信息等。

一颗 multi-function EP 会有多个 `pci_dev`；它们共享物理 Link，却有独立 Configuration Space 和 driver binding。

来源：X6000 Linux 5.10.4 `include/linux/pci.h` 中的 `struct pci_host_bridge`、`struct pci_bus` 和 `struct pci_dev`；通用实现位于 `drivers/pci/`。本机 Ubuntu 6.8 header 仅用于对照较新接口，不能替代目标分支源码。

## 3. Configuration access 怎样下到硬件

PCI core 调 `bus->ops` 的 read/write。常见 host 实现有两种：

- **ECAM**：按 segment/bus/device/function/register 算 MMIO address，CPU load/store 由 host bridge 转为 Config Request；
- **controller-specific**：先编程 Cadence outbound config region/target BDF，再访问 aperture，由 core 生成 Type 0/1 Config TLP。

无论 CPU 端长什么样，线上规则仍是：到 secondary bus 转 Type 1→Type 0，Endpoint 以 Type 0 命中 Function，返回 CplD/Completion。

```text
pci_bus_read_config_dword(bus, devfn, where)
 → pci_ops.read
 → ECAM load 或 Cadence config aperture sequence
 → Configuration Read TLP
 → Completion with Data
 → u32 返回 PCI core
```

X6000 RC 路径已经提供：`drivers/pci/controller/pcie-edge-host.c` 的 `edge_pcie_host_ops` 实现配置访问，`edge_pcie_host_setup()` 设置 `bridge->ops` 后调用 `pci_host_probe()`。该驱动把名为 `cfg` 的 resource 映射为配置 aperture，并在 `edge_pcie_host_init_address_translation()` 中初始化 Cadence 地址转换。精确的 BDF 编码和访问锁见同文件的 `edge_pcie_host_rd_other_conf()` / `edge_pcie_host_wr_other_conf()`，不再是 Unknown。

## 4. Linux enumeration 与协议 enumeration 的接缝

| Linux 动作 | 最终 PCIe 行为 |
|---|---|
| probe Vendor ID | Config Read offset `0x00`；全 1/失败通常表示不存在 |
| 读 Header Type/Class | Config Read Type 0/1 fields |
| 扫 capability list | 追 Configuration Space pointer 链 |
| BAR sizing | disable/decode-safe 流程中写全 1、读 mask、恢复 |
| 给 BAR 分 resource | 在 host/bridge window 中选择地址，写 BAR |
| 配 bridge bus numbers | 写 Type 1 Primary/Secondary/Subordinate |
| 配 bridge windows | 写 Type 1 I/O/Memory/Prefetchable ranges |
| register `pci_dev` | 纯软件对象，此后 driver core 才匹配 |

Bus Number 和 BAR address 多数不是 Endpoint 出厂固化的。Endpoint 提供 size/type/功能，firmware/Linux 做 hierarchy-wide allocation。

## 5. Driver binding

`struct pci_driver` 至少含：

- `id_table`：vendor/device/subvendor/subdevice/class 等 match；
- `probe` / `remove`；
- PM callbacks；
- `err_handler`；
- 可选 SR-IOV callbacks。

Header 注释确认：`pci_register_driver()` 会对已存在且未被占用的匹配设备调用 probe，之后热插入的设备也会触发；probe 在 process context，可睡眠。成功返回 0 表示 driver 接管该 Function。

注册 driver 不会“触发 PCIe 枚举”，它只是把一个匹配规则加入 driver model。枚举与 binding 是两阶段。

## 6. Resource tree 与 `pci_request_regions()` 的作用

枚举得到的 BAR 范围进入 `pdev->resource[]` 和全局 resource tree。它同时表达：

- start/end/flags；
- 是 I/O 还是 Memory、prefetchable 等；
- 此地址范围已由哪个 device/driver claim。

request region 的意义是执行资源所有权冲突检测，避免两个驱动同时使用同一段 MMIO。它通常不重新给 BAR 分配地址，也不创建 CPU 虚拟映射。

## 7. Evidence boundary

- **Confirmed**：结构、API 和通用调用链，来自 X6000 Linux 5.10.4（commit `e74fd0e7746a`）的 `include/linux/pci.h`、`drivers/pci/probe.c`、`drivers/pci/pci.c` 和 `drivers/pci/pci-driver.c`。
- **Confirmed**：PCIe config/BAR/bus-number 协议，来自 Base Spec 5.0 与 Cadence manual。
- **Confirmed X6000 integration**：`pcie-edge-plat.c`、`pcie-edge-host.c` 和 `pcie-edge.c` 将 Cadence access/window、IRQ domain 与 `pci_host_bridge/pci_ops` 连接起来，并由 `pci_host_probe()` 进入通用枚举流程。
- **仍需实机确认**：具体产品最终 DTB、firmware 是否预先保留部分窗口，以及外部 Host 所用内核不属于这棵 X6000 设备侧源码树。

## 8. Source Reference

- X6000 Linux 5.10.4：`include/linux/pci.h`、`drivers/pci/probe.c`、`drivers/pci/pci.c`、`drivers/pci/pci-driver.c`。
- X6000 Host controller：`drivers/pci/controller/pcie-edge-plat.c`、`pcie-edge-host.c`、`pcie-edge.c`。
- PCIe Base Specification 5.0, Configuration/Type 0/Type 1/BAR sections。
- Cadence Core User Guide §§4.5.5–4.5.6 与 Chapter 9/10 address/config register blocks。
- 王齐《PCI Express 体系结构导读》第 14 章，printed pp.365–416 / PDF pp.377–428：历史 Linux x86/PowerPC 扫描、resource assignment 与 host `ranges`/inbound/outbound 实例；API 属 Linux 2.6-era，只作架构演变证据。
