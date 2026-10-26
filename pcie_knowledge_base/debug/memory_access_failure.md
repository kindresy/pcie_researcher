# Memory Access Failure：BAR 映射后仍无法正常访问

## 1. 读失败和写失败不是同一种病

- Memory Read 是 Non-Posted：成功时通过一个或多个 CplD 返回数据；UR/CA 等失败状态通常通过不带数据的 Completion 返回，完全没有返回则可能触发 CTO。
- Memory Write 是 Posted：没有普通 Completion；CPU 写完成只代表请求被接收进发送路径，不代表 EP 寄存器已改变。

因此调试 MWr 必须找 EP BAR-hit/local AXI write 或做受控 readback，不能等一个协议上不存在的 Completion。

## 2. 完整路径

```text
CPU VA → page table/device mapping → CPU physical MMIO
 → host bridge outbound decode/translation → MWr/MRd TLP
 → Link/Data Link → EP Function+BAR hit
 → inbound PCIe→AXI translation → local AXI/APB target
 → MRd only: AXI read data → CplD → requester → CPU load
```

## 3. 先判断失败类型

| 表现 | 重点检查 |
|---|---|
| CPU synchronous abort | host MMIO aperture/SError policy/Link Down |
| read 全 1 | config/resource失效、UR折算、设备移除 |
| read 卡住后 timeout | MRd/CplD 路径、EP local AXI hang |
| AER UR | address/type/Function/BAR 不支持 |
| AER CA | EP 已接请求但本地完成失败 |
| write 无效果且无错误 | posted write 路由、MSE、BAR hit、byte enable、ordering/readback |
| 偶发旧值 | posted write drain、device-specific flush、并发/doorbell语义 |

## 4. 软件侧核对

- 使用 `pci_iomap()`/`devm_pci_iomap*()` 得到的 `__iomem` VA，不把 BAR bus address 强转指针；
- 用 `readl/writel` 等 MMIO accessor，不用普通 pointer load/store；
- offset、register width、endianness、alignment 正确；
- `pci_request_regions` 成功，Command.MSE=1；
- 设备未处在 D3hot/D3cold，Link 未进入不可访问状态；
- remove/error recovery 与当前访问没有竞态。

## 5. Cadence 观察点

- RP outbound region hit；no-hit 会对本地 AXI 返回 DECERR；
- TLP descriptor 是 Memory 而不是 Config/IO/Message；
- EP BAR hit、Function number、address translation；
- inbound core AXI Master `AW/ARADDR`、`WSTRB`、`BRESP/RRESP`；
- local target 的 SLVERR/DECERR 会导致读请求 CA 等错误处理；
- Link Down indication会终止 outstanding request。

## 6. Byte Enable 小陷阱

未对齐或窄访问会通过 First/Last DW Byte Enable 表达有效 byte。EP AXI `WSTRB` 应与其对应。若寄存器只接受 32-bit full write，而软件用了 `writeb()`，Link 和 BAR 可能都完全正常，最后只是外设礼貌地没照办。

## 7. 最小复现实验

1. 选无副作用 scratch/version register；
2. 固定 speed/width，禁掉会干扰复现的 runtime PM；
3. 保存 BAR/resource/ATU；
4. 单次 read 或 write+readback；
5. 同时记录 CPU address、TLP header、EP AXI address/response、AER；
6. 若只在高代际失败，再转 Link replay/equalization，而不是先改 BAR。

## 8. Source Reference

- PCIe Base Specification 5.0 r1.0, §§2.2–2.3、2.8。
- Cadence PCIe Core User Guide, §§4.5.2、4.5.5–4.5.6。
- [Memory TLP 与 Completion](../transactions/memory_tlp_and_completion.md)。
