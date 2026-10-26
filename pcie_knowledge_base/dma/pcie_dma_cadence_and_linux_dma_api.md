# PCIe DMA、Cadence uDMA 与 Linux DMA API：区分协议、引擎和地址映射

## 1. 先拆开三个“DMA”

大家都叫 DMA，职责却不一样：

1. **PCIe bus mastering / transaction ability**：Endpoint 能作为 Requester 发 Memory Read/Write TLP。
2. **设备或控制器里的 DMA engine**：读取 descriptor、切分传输、搬数据、记录状态并发中断。
3. **Linux DMA API**：为设备准备可访问的内存和 DMA address，处理 cache/IOMMU/架构差异。

PCIe 规范主要规定第 1 层的线上行为。Cadence Chapter 5 描述一个可选集成的 Universal DMA Engine。Linux 的 `dma_alloc_coherent()`、`dma_map_single()` 则不替设备搬数据，它们负责让“设备要访问哪块内存、用哪个地址”这件事在当前平台上成立。

可以把三者的职责简化为：Linux DMA API 建立设备可用的地址映射和一致性语义，DMA engine 执行数据搬运，PCIe TLP 则承载跨 PCIe Link 的读写请求和返回数据。这种类比只用于区分职责，具体地址和时序仍以后文定义为准。

## 2. 普通 SoC DMA 与 PCIe DMA 的共同点和差别

共同点：

- 都有 source/destination、length、descriptor、channel 和 completion interrupt；
- 都可能做 scatter/gather；
- 都要处理 cache coherency、address width、alignment 和 error。

差别主要在互连边界：

- SoC DMA 常在本地 AXI fabric 上读写两个本地地址端点。
- PCIe EP DMA 的一端经过 PCIe fabric，读写被封装为 MWr/MRd/CplD；存在 MPS、MRRS、4 KiB boundary、Tag、Completion Timeout、AER、Link Down 等协议约束。
- EP 看到的 Host buffer 地址通常是 DMA address/IOVA，未必是 CPU physical address。

“PCIe controller 会不会自带 DMA”是 IP 实现问题，不是 PCIe 标准的必选答案。当前 Cadence 配置确实有 PCIe-AXI integrated uDMA，但另一个 EP 完全可能把 DMA 放在 application logic 里，只借 controller 发 TLP。

## 3. Endpoint 为什么有资格主动访问 Host Memory

EP 要发 Memory Request，至少需要：

- 作为 Requester/bus master 的硬件路径；
- 对应 Function 的 PCI Command Register `Bus Master Enable`（BME）置位；
- 有效的 Host DMA address 与访问权限；
- Function 处于允许发请求的电源/复位状态。

Cadence §4.5.6 明确要求：EP 通过 AXI Slave interface 发 MRd/MWr 前，BME 必须为 1、Function 必须 D0-Active、且不处于 FLR。X6000 Linux 5.10.4 中，`pci_set_master()` 调用 `__pci_set_master(dev, true)` 更新 Command.Bus Master Enable，再调用 architecture hook `pcibios_set_master()`；它只打开发起事务的权限，并不启动设备内部 DMA engine。

## 4. DMA address 为什么不是随手拿个指针

驱动常同时拿到两个地址：

```c
void *cpu_addr;
dma_addr_t dma_addr;
```

- `cpu_addr` 给 CPU 解引用；
- `dma_addr` 写入设备 descriptor/register，让设备放进 PCIe Memory Request address。

概念路径：

```text
EP TLP uses DMA address / IOVA
 → RC ingress
 → optional IOMMU / interrupt-remapping domain
 → Host physical memory
 → memory controller / cache-coherency fabric
```

如果没有 IOMMU，DMA address 也不应被程序员想当然地写成 CPU virtual 或 physical address；DMA offset、host bridge aperture 和架构限制仍可能存在。API 的意义就是把平台差异收在统一契约里。

## 5. `dma_alloc_coherent()` 与 PCIe DMA 的关系

通用 Linux 语义（目标版本源码待补）：

```c
cpu_addr = dma_alloc_coherent(dev, size, &dma_addr, GFP_KERNEL);
```

它分配一块 CPU 与设备可以并发观察、无需常规 streaming sync 的 coherent DMA buffer，并返回：

- CPU 访问用的 virtual address；
- 设备访问用的 DMA address。

典型用途是 descriptor ring、doorbell-adjacent metadata 或控制结构。驱动把 `dma_addr` 写入 EP，EP 的 DMA engine 才能向正确 Host address 发 TLP。

**Coherent 不等于不需要 ordering。** CPU 写 descriptor 后再写 doorbell，仍要遵守 DMA API/architecture 要求的 memory barrier；设备写 data 后再发 MSI，也要满足 PCIe ordering 与设备协议。Cache 一致性解决“彼此看见什么”，barrier/ordering 解决“先看见什么”。

## 6. `dma_map_single()` 为什么返回新地址

通用 Linux streaming mapping 语义（目标版本源码待补）：

```c
dma_addr = dma_map_single(dev, cpu_buf, len, DMA_TO_DEVICE);
```

它把已有 CPU buffer 映射为设备可用 DMA address，可能执行：

- 建立 IOMMU mapping；
- 做 architecture-specific cache maintenance；
- 检查 DMA mask/addressability；
- 在受限平台使用 bounce buffer。

方向非常重要：

- `DMA_TO_DEVICE`：CPU 准备数据，设备读取——在 PCIe 上通常对应 EP 发 MRd/CplD。
- `DMA_FROM_DEVICE`：设备写 Host buffer——通常对应 EP 发 MWr。
- `DMA_BIDIRECTIONAL`：两边都可能读写，但不能把它当成懒得思考方向的万能胶。

映射失败必须检查，完成后要按 API 生命周期 unmap；非一致性架构上，重复共享 buffer 还可能需要 `dma_sync_*()`。

## 7. Cadence Universal DMA Engine 架构

Cadence 把两侧内存命名为：

- **System Memory**：本地 AXI infrastructure 可访问的内存；
- **External Memory**：PCIe link partner 一侧的远端内存。

uDMA 最多 8 channels，每 channel 读取本地 System Memory 中的 linked-list descriptors。描述符包含：

- AXI/System address 和 AXI attributes；
- PCIe/External address 和 TLP attributes；
- transfer length、direction/control；
- system/external status、channel status；
- next descriptor pointer。

每 channel 通过 APB control registers 编程 list head、Go/status 与 interrupt。Inbound/outbound 各有共享 dual-port SRAM，分 partition 给 channel，既用于乱序重排，也作为 bus width gearbox、clock-domain crossing 和 alignment buffer。

来源：Cadence Chapter 5 §§5.1–5.4，printed pp.281–289 / PDF pp.370–378。

## 8. Cadence 所说的 inbound / outbound

命名以本地 Cadence controller 为观察点：

### Inbound transfer：External PCIe → local AXI

```text
uDMA descriptor
 → 发 PCIe Memory Read Request 到 remote/external address
 → bridge 按 MRRS、MPS、4 KiB boundary 进一步切分
 → 多个 Tag / 多个 CplD 可能交错返回
 → dual-port RAM 按原传输顺序重排
 → AXI Master write 到 local System Memory
```

同一个 PCIe MRd 的多个 CplD 必须按地址顺序返回，但不同 MRd 的 Completion 可以交错。uDMA buffer 的一个核心职责就是把这种合法乱序整理成 AXI 侧可消费的顺序。

### Outbound transfer：local AXI → External PCIe

```text
AXI Master read local System Memory
 → 数据先完整进入 dual-port RAM
 → PCIe controller 发 Memory Write TLP
 → channel 处理 descriptor/status
 → 最后一个 descriptor 完成后中断 local firmware
```

Cadence 特意等完整 write data 进入 RAM 后再向 PCIe controller 开始 write，因为 controller 要求一旦 TLP transfer 开始，write data 必须持续可用。

注意这里的 inbound/outbound 与“Host 视角”可能相反。文档讨论方向时应永远把端点写出来，光说“做一次 inbound DMA”容易让会议多开半小时。

## 9. 切包不是 DMA channel 一个人的活

Cadence inbound example 中，channel 可以先把大 transfer 分成最多 4 KiB 的片段，bridge 再按：

- PCIe 4 KiB address boundary；
- MRRS（Memory Read Request 最大请求长度）；
- MPS（Completion/Write payload 上限相关）；
- AXI burst boundary/length；

继续切分。

这说明应用请求的“传 1 MiB”不会成为一条巨型 TLP。descriptor-level transfer、PCIe Request、Completion 与 AXI burst 是不同粒度。

## 10. Firmware 编程模型

### 初始化

1. 读取 `CAPVS`，确认 hardware version/capability；手册算法对应 1.0。
2. 读取 `CONF`，得到 descriptor layout、channel 数、RAM partition 数/大小。
3. 按需启用 interrupt；mask reset 后默认全部禁用。

### 建链与启动

1. 在 System Memory 构造 Bulk / Scatter / Gather descriptor list。
2. 非末尾 descriptor 的 `continue=1`、next pointer 指向下一项；末尾 next=0、continue=0。
3. 编程 channel attributes 与 start pointer。
4. 写 Go/doorbell。
5. channel 运行时，除 linked-list append 规则允许的寄存器外，firmware 不应访问该 channel registers。

### 完成与错误

channel 置 done/error interrupt。Firmware 应扫描 descriptor status，直到第一个未尝试项，并区分：

- external/PCIe early completion；
- system/AXI early completion；
- internal data-integrity error；
- invalid descriptor；
- buffer overflow/underflow/not-empty；
- descriptor fetch/status writeback failure。

ISR 读 `PCIe DMA Interrupt Register`，处理所有置位 channel，再把同一值写回清除——这是典型 W1C 使用模式，不能用随意的 read-modify-write 清状态。

来源：Cadence §5.5，printed pp.289–298 / PDF pp.378–387。

## 11. DMA descriptor 的 ownership 与 ordering

Cadence 支持运行中 append Bulk linked list，但 firmware 可能在 hardware 读 descriptor 时更新 continue/next pointer。手册用 `go`、start pointer 和 next pointer 组合定义一致性处理，并假设 hardware 能按 firmware 写寄存器的顺序观察它们。

落到 Linux/CPU 上仍要问：

- descriptor memory 是否 coherent？
- 更新 descriptor 后、敲 Go 前需要哪个 DMA write barrier？
- hardware 写 status 后，CPU 在 ISR 中读取前需要怎样同步？
- descriptor 与 payload 的 ownership 在哪个状态切换？

Cadence 手册给状态机契约，Linux DMA API/architecture 给 cache 与 barrier 契约；少看任何一半，都可能得到“逻辑完全正确，偶尔就是不工作”的经典作品。

## 12. 一个 Linux EP device driver 的概念流程

以下为版本无关骨架，不是已确认源码调用链：

```text
probe
 → pci_enable_device()
 → set DMA mask / coherent mask
 → pci_request_regions()
 → pci_iomap() control BAR
 → pci_set_master()          // BME
 → allocate/map descriptor and data buffers
 → write DMA addresses into device descriptors/registers
 → allocate MSI/MSI-X vectors
 → publish descriptor ownership + barrier
 → write doorbell

device
 → fetch descriptor via MRd/CplD
 → transfer data via MWr or MRd/CplD
 → write completion/status
 → send MSI/MSI-X MWr

ISR/NAPI/thread
 → read/ack device status
 → sync/unmap streaming buffers as required
 → return buffer ownership
```

真正实现时要追踪 kernel version、device DMA mask、IOMMU domain、interrupt model 和 Cadence uDMA 是否实际综合/使用。

## 13. Debug 清单

### 请求根本没出 Link

- Command.BME、D0/FLR 状态；
- descriptor DMA address 与 ownership；
- channel Go/mask/status；
- Cadence AXI outbound region/DECERR；
- Tag/Split Completion Table 是否满。

### MRd 发出但不完成

- IOMMU mapping/permission 与 DMA direction；
- 4 KiB boundary/MRRS；
- UR/CA/Poison/CTO；
- Requester ID + Tag/Completion fields；
- Link Down/FLR 是否终止请求。

### 数据错但状态成功

- CPU/Device cache sync 与 barrier；
- descriptor address/length/endian；
- Completion interleaving/reassembly；
- AXI width/alignment/WSTRB；
- MSI 是否越过 data write（TC/VC/ordering）。

## 14. Source Reference

- Cadence PCIe Core User Guide, §4.5.6, printed pp.248–271 / PDF pp.337–360。
- Cadence PCIe Core User Guide, Chapter 5, printed pp.281–298 / PDF pp.370–387。
- PCIe Base Specification 5.0 r1.0, Chapter 2 request/completion/ordering rules；DMA engine 本身不是 Base Spec 定义的统一模块。
- `PCI_Express体系结构导读_王齐.pdf`, Chapter 12, especially printed pp.297–343（OCR 起始 PDF p.308）；作为实例与历史 Linux 线索，API 版本需重新验证。
- X6000 Linux 5.10.4 DMA API：`include/linux/dma-mapping.h`、`kernel/dma/mapping.c` 和 `Documentation/core-api/dma-api*.rst`；外部 Host 的 IOMMU backend 与运行期 IOVA 策略仍需 Host 内核和实机日志确认。
