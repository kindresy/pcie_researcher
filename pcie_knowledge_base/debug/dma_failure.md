# DMA Failure：从描述符到 Completion 的分层排查

## 1. 先确认在调哪一个 DMA

“PCIe DMA”可能是 Endpoint 自己的业务 DMA engine，也可能是 Cadence integrated uDMA。PCIe core 本身不会看到一块 buffer 就主动搬运；软件必须把 **DMA address** 写进正确 engine 的 descriptor。

## 2. 方向先说清

从 EP 观察：

- EP 写 Host：DMA engine → outbound AXI Slave → PCIe MWr，Posted；
- EP 读 Host：DMA engine → outbound AXI Slave → MRd → CplD，Non-Posted。

若手册把 inbound/outbound 以别的观察点命名，画箭头，不和方位词争论。

## 3. 四关检查法

### 第 1 层：engine 是否真正启动

- descriptor ownership/valid、next pointer、length、direction；
- channel enable/start/doorbell；
- channel busy/done/error、interrupt mask/status；
- descriptor/payload 对齐与边界要求。

### 第 2 层：设备是否具备发起事务的条件

- PCI Command Bus Master Enable=1；Cadence 明确要求 EP 只有 BME=1 才能发 MRd/MWr；
- Function 未处在 FLR/reset；Cadence FLR 会清 BME，Host 重新 Config Write 后才能恢复 outbound traffic；
- Link/Data Link up，outbound AXI region/descriptor 正确。

### 第 3 层：DMA address 是否对设备有效

- buffer 来自 `dma_alloc_coherent()`，或 streaming mapping 已成功；
- descriptor 写的是 `dma_addr_t`，不是 CPU virtual/physical；
- `dma_set_mask_and_coherent()` 与 engine address width 匹配；
- IOMMU mapping/domain/permission 与 Requester ID/PASID 匹配；
- SG 每段使用 `sg_dma_address/len`，不是原始 page 字段；
- streaming DMA 的 sync/ownership 方向正确。

### 第 4 层：协议返回是否完整

- EP→Host write：看 MWr 是否到 RC/IOMMU/memory，IOMMU fault 是否报 RID/IOVA；
- EP→Host read：看 MRd Tag、CplD byte count/lower address、所有 split Completion 是否收齐；
- UR/CA/CTO/Poison/Unexpected Completion；
- Cadence AXI response、completion table 与 channel error。

## 4. 症状到根因

| 症状 | 先查 |
|---|---|
| channel 一直 idle | descriptor/start/clock/reset |
| 一启动就 IOMMU fault | DMA address、RID/PASID、map direction/permission |
| Host→EP/EP→Host 单向失败 | direction descriptor、inbound/outbound translation |
| 小包好、大包失败 | boundary、SG、MPS/MRRS、split completion、length encoding |
| 数据旧/脏 | cache ownership、sync API、barrier、descriptor visibility |
| done 不来但数据已到 | interrupt mask/MSI path，而非数据路径 |
| FLR 后永远不再发包 | BME 被清、descriptor/channel状态未重建 |

## 5. 一份能复盘的日志

应记录 buffer CPU VA、`dma_addr_t`、长度和方向、descriptor 原始 DWords、channel state、BME、Function RID、IOMMU fault、首个 MRd/MWr header、Completion/AER，以及最终的 cache sync。只打印 `dma timeout` 只能说明操作未完成，无法判断失败发生在哪一层。

## 6. Source Reference

- PCIe Base Specification 5.0 r1.0, Memory Request/Completion、Command.BME、error rules。
- Cadence PCIe Core User Guide, Chapter 5；§4.5.6；§4.7.4；§§9.6/9.12/9.18/9.24。
- [PCIe DMA、Cadence uDMA 与 Linux DMA API](../dma/pcie_dma_cadence_and_linux_dma_api.md)。
