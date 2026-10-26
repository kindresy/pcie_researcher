# 王齐第 12 章 OCR 评估：Capric FPGA Endpoint 从硬件到旧 Linux 驱动

日期：2026-08-23

阅读范围：printed pp.297–343 / PDF pp.309–355，逐页 OCR。

这是整本教材里与本知识库目标最贴近的一章：它不满足于说“设备可以 DMA”，而是让一个 Xilinx Virtex-5 LogiCORE Endpoint 真正组织 MRd/MWr、维护 Tag、接收 CplD，再接到 Linux driver。它的 Linux 是 2.6.31 年代，但硬件问题一点也没有因此变得客气。

## 1. 示例硬件做了什么

Capric 卡由 PCIe LogiCORE、TLP 发送/接收逻辑、256 B BAR0 register block、DMA control 和片内 SRAM 组成：

```text
Host写BAR0 address/size/start
 → FPGA DMA state machine
 → MWr：SRAM → Host Memory
 → MRd/CplD：Host Memory → SRAM
 → MSI通知完成
```

它只支持 32-bit DMA address、一次最多 `0x7ff` B、一个 active DMA operation 和一个 MSI vector。简陋反而适合教学：所有延时、Tag、buffer 和 interrupt race 都没地方躲。

## 2. DMA Write 为什么比 DMA Read 简单

### Device → Host：MWr

设备按 MPS 与 4 KiB boundary 把 `[address, length]` 切成一个或多个 MWr。MWr 是 Posted；硬件发完 packet 后不能因为“没有 Completion”就把 protocol error handling 和 data visibility 当作不存在。

### Host → Device：MRd + CplD

设备还必须：

1. 按 MRRS/4 KiB boundary 发多个 MRd；
2. 给每个 outstanding request 分配未占用 Tag；
3. 为最坏情况 Completion header/data 预留资源；
4. 用 Requester ID + Tag 拒绝 Unexpected Completion；
5. 检查 Completion Status、Byte Count、Lower Address 与 Length；
6. 接受不同 Tag 的 CplD 交错返回，把 payload 放回正确位置；
7. 收齐一个 request 的所有 CplD 后才能释放 Tag；最后一个 request 收齐才宣布整笔 DMA 完成。

教材用“256 个 512 B MRd、每个 CplD 只有 64 B payload”算出 header/data buffer 压力，直观展示了 outstanding 越多，吞吐潜力和硬件欠账一起增加。这个思路已经补入 [`examples/endpoint_from_power_on_to_dma_and_msix.md`](../examples/endpoint_from_power_on_to_dma_and_msix.md)，并与 Cadence two-stage Completion buffer 对照。

## 3. BAR register protocol 也需要设计

Capric 的 address、size、start、done、mask、pending、W1C register 揭示一个经常被 PCIe 术语掩住的事实：PCIe 只负责把 MMIO/DMA/interrupt 送到设备，**不会替你设计设备业务协议**。

一个稳健 register interface 还要明确：

- address 高低位写入顺序与 commit point；
- start 是 pulse、W1S 还是 level；busy 时再写如何处理；
- done/error 的 W1C 与 interrupt mask/pending 语义；
- reset/FLR/Link Down 如何终止 active DMA；
- MMIO write ordering 与启动前 barrier；
- 64-bit DMA、SG/ring、timeout/cancel 能力。

Cadence uDMA 文档提供了更现代的 descriptor/channel/error surface；Capric 则适合解释为什么这些寄存器会存在。

## 4. Linux 部分：稳定职责与退休函数分开看

### 4.1 仍稳定的 probe 因果链

教材依次使用：driver ID match → probe → `pci_enable_device()` → DMA mask → `pci_set_master()` → request BAR → resource start/length → ioremap → MSI → request IRQ。这条职责顺序仍有教学价值：

- BAR register 中保存的是 PCI-side assigned address；Linux resource 是 CPU-side host resource；
- ioremap 后得到 CPU virtual I/O mapping，不是 DMA address；
- Bus Master Enable 必须单独允许设备主动发 Memory Request；
- `pci_map_single()`/今天的 `dma_map_single()` 返回给设备填写 descriptor 的 DMA address。

### 4.2 不能复制粘贴的 2.6.31 细节

- `pci_enable_msi()`/单一 `pdev->irq` 应改用 `pci_alloc_irq_vectors()` + `pci_irq_vector()` 等现代接口；
- `pci_map_single()` compatibility wrapper、`pci_dma_*` 常量和书中 arch implementation 路径均已演化；
- 通用 driver 优先 `pci_iomap()`/managed resource helpers，不自己假定 `ioremap()` 属性；
- `interruptible_sleep_on()` 有经典 lost-wakeup race，教材自己也指出它很糟；现代代码应用 condition + wait-event/completion，并在置条件/唤醒间保证锁与 ordering；
- 书中精确 `pci_register_driver → ... → local_pci_probe` 调用链只能作为当年考古，不能填到缺失的 Linux 6.x source 证据栏。

### 4.3 DMA direction 名字要站在设备视角

Capric 所称“DMA 写”是 device 写 Host，所以 map direction 是 `DMA_FROM_DEVICE`；“DMA 读”是 device 从 Host 读，所以是 `DMA_TO_DEVICE`。方向不是看 driver 函数叫 `read()` 还是 `write()`，也不是看 CPU 心情。

非一致性平台还必须遵守 map/sync/unmap ownership。教材对未对齐 buffer 首尾 cache line 的讨论非常有价值：粗暴 invalidate 可能丢掉同一 cache line 中不属于 DMA buffer 的脏数据。正确做法是遵守 DMA API 的对齐和 ownership contract，而不是在 driver 里手搓一组“看起来差不多”的 cache 指令。

## 5. 性能分析：最值得继承的不是旧数字

教材分解了四类成本：

- TLP header/LCRC/framing 对 payload ratio 的影响；
- MRd request + 多个 CplD 比 MWr 多出的往返和 header；
- ACK/NAK、replay、FC update；
- host↔EP MMIO Read 的 request/completion round trip。

其中一处需要明确勘误：PCIe Flow Control 是 **逐 Link/hop-by-hop**，不是教材 p.337 所说的端到端机制。端到端的是 Request/Completion 等 transaction 语义。

书中 Gen1/2、100 MHz/64-bit FPGA interface、固定 D0 延时和 payload ratio 公式只是模型。今天仍成立的工程结论是：

- 增大 useful work batch 可以摊薄 doorbell/interrupt latency；
- descriptor ring 与多个 outstanding Tag 用流水掩盖 round-trip；
- ISR 若再做 MMIO Read，会把一次高延时 MRd/CplD 串到 critical path；
- MSI-X 多 vector 或 host-memory completion record 可减少这类读；
- MPS/MRRS 不是越大越好，还受 buffer/backpressure/latency/workload约束。

## 6. Coverage 结论

| 范围 | 状态 | 映射 |
|---|---|---|
| Endpoint BAR/DMA/MSI architecture | D+M | worked example、DMA、interrupt |
| MWr splitting | D+M | memory TLP/MPS |
| MRd Tag/CplD reassembly/buffering | D+M | transaction、Cadence Completion buffer |
| Linux probe/MMIO/DMA/IRQ | D+M（年代校正） | Linux function driver、DMA API |
| cache maintenance/alignment | D+M | DMA文档 |
| throughput/latency model | D+M | performance原则；旧平台数字不外推 |

## 7. Source Reference

- 王齐，《PCI Express体系结构导读》，Chapter 12，2010 年第一版，扫描 PDF pp.309–355。
- PCI Express Base Specification 5.0 r1.0, Chapters 2–3、§6.1.4。
- Cadence *PCIe Core User Guide*, Chapters 5、10、§11.4。
