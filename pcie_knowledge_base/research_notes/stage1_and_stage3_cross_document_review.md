# Stage 1 / Stage 3 Cross-document Review

日期：2026-08-23

## Stage 1：世界观、拓扑、分层

复核链：RC≠RP、EP同时可做 Requester/Completer、Switch是多桥配置模型、Link/Lane/PHY/PIPE、Transaction/Data Link/Physical职责、BDF/hierarchy。

结果：核心术语首次出现均有直觉和软件/硬件落点；BDF/RID 独立文档补齐后依赖链已闭合。Glossary 中原把 RID 直接展开成 Requester ID，现已纠正为 Routing ID，并单列 Requester ID。

## Stage 3：Transaction、Address、DMA、Interrupt

### 方向一致性

统一采用 Cadence core 观察点：

```text
PCIe inbound  → core AXI Master → local target
local outbound → core AXI Slave → PCIe Request
```

发现 `PROGRESS.md` 一处旧写法“EP DMA→AXI master”，已纠正为 outbound AXI Slave。uDMA 自己访问 local System Memory 时是 AXI Master，但它把 PCIe request交给 controller 时仍进入 controller outbound path；文档已分别注明观察者。

### Transaction 语义

- MWr/MSI 是 Posted、没有普通 Completion；MRd/Config/Atomic 需要 Completion。
- Data Link ACK 是逐跳可靠性，不是 transaction completion。
- MPS/MRRS/RCB/credit 已分离；RO/IDO 与 No Snoop 已分离。
- Requester ID+Tag、split completion与 Cadence/AXI reordering 对应完整。

### 地址域

- CPU VA/physical、PCI bus address、EP local AXI、DMA address/IOVA 无跨文档混用。
- BAR sizing/allocation、RP outbound、EP inbound 和 Resizable BAR 顺序完整。

### DMA/IRQ

- Linux DMA API 与 DMA engine/PCIe bus mastering 三层分清。
- Cadence uDMA descriptor/register/ISR/partial completion/reset recovery已覆盖。
- MSI-X capability/table/PBA→MWr→remapper→Linux IRQ 路径完整；未把 PCIe vector index 当 Linux IRQ。

## Evidence boundary

目标 SoC IOMMU/MSI remapper、DMA coherency和 Linux glue driver 未提供；相应路径在 RC/AXI boundary 后标为 Unknown。此边界不会阻止通用协议/控制器阶段通过，但会阻止最终项目特定调用链宣称完成。

