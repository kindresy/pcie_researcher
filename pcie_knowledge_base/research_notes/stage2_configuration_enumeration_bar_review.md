# Stage 2 Review：Configuration、Enumeration 与 BAR

日期：2026-08-23

## Review 结论

本阶段核心因果链已闭合：

```text
Link/DL up
 → Host config aperture/ECAM
 → BDF + Type 1/Type 0 routing
 → Vendor ID 发现 Function
 → Type 0/1 header + capability walker
 → BAR sizing
 → hierarchy resource/bridge window allocation
 → RP outbound Memory translation
 → EP BAR hit + inbound AXI translation
 → Linux resource/pci_iomap
```

## 完整性

- 已覆盖 Config Space vs BAR Space、Type 0/1、ECAM address formula、BDF/RID/Tag、ARI、capability/extended capability、安全 walker、CRS/UR/CTO。
- 已覆盖 BAR type、32/64-bit、prefetchable、sizing、bridge window、CPU/PCI/AXI 三种地址和 Cadence translation。
- 已补 enumeration/BAR/MMIO 独立 Debug runbook。
- Linux 精确 resource allocator/scan 函数调用链仍因缺 kernel source blocked；文档明确保留架构级路径，没有用旧教材替代现代源码。

## 正确性复核

- ECAM 是 CPU MMIO 外观但在线路上生成 Non-Posted Configuration Request；与 Memory BAR access 区分明确。
- BAR 是地址需求/decoder，不是 RAM 本体；`pci_iomap()` 返回 CPU virtual `__iomem` 地址，不是 PCI bus address。
- Requester ID+Tag 构成 transaction identity，segment/domain 不在线上 16-bit RID 内。
- ARI 把传统 Device+Function 8 bit 重解释为 8-bit Function，未把它描述成增加 Bus Number。
- Cadence local APB config access 与远端 Config TLP 是两条入口；LM/AXI/DMA 私有空间不会被远端遍历到。

## 初学者可理解性

正文均从“身份证/分房/地址牌”直觉切入，再落到 TLP、bridge window、AXI translation 和 Linux API。每个抽象概念至少有一个数值或路径例子。

## 尚存边界

1. 目标 SoC ECAM/APB physical base、host aperture 和 `ranges/dma-ranges` 未提供。
2. 目标 Linux `pci_scan_*`、resource assignment、Cadence `pci_ops` 精确实现未提供。
3. 实际 build capability dump/offset/enable 仍需硬件或 configuration report（OQ-009）。

这些是输入资料缺失，不是可从现有资料继续推导的空白；已在 `OPEN_QUESTIONS.md` 标为 Blocked/Open。

