# Cadence uDMA 编程：从 descriptor 到 Done/Error 中断

## 1. 这块 DMA 搬的是哪两边

Cadence integrated uDMA 在本地 **System Memory（AXI）** 与 Link partner 的 **External Memory（PCIe）** 之间搬数据。手册的方向命名是：

- inbound：External/PCIe → local System/AXI，先发 MRd，收 CplD，再写本地 AXI；
- outbound：local System/AXI → External/PCIe，先读本地 AXI，再发 MWr。

uDMA 本身对 descriptor 中大部分 PCIe/AXI attribute 只是转发，不会替固件判断地址是否合法。特别是该版本明确说：PASID、ATS 只支持 AXI data path 生成的 traffic，**不支持 uDMA 生成的 traffic**。

## 2. 初始化顺序

1. 读 `CAPVS @0xf8`，确认 uDMA version；手册 §5.5 算法面向 version 1.0，但本配置 register reset 字段显示 minor/major 的排列需按 bit 定义解码。
2. 读 `CONF @0xfc`：channel 数、RAM partition 数/size、system/external address/attribute width。
3. 本配置 `num_channels=4`、`num_partitions=4`、`partition_size=6`，但 RTL 家族最多描述 8 channel；软件必须读能力，不能按目录里出现 Channel 7 就宣布硬件有八条。
4. 清旧 interrupt status，再通过 enable/disable 寄存器选择 Done/Error IRQ。
5. 分配本地 descriptor memory 和数据 buffer，满足 DMA address width/coherency。

**NEED_VERIFY：** §5.5.1 写“interrupt mask reset 后全部 disabled”，而 §9.6.1.22 的 Enable Register 表格给 reset `1'b1` 且属性标作 `R/WOCLR`，文字又说 write 1 enables。这三者语义看起来不完全一致。驱动应读回/仿真验证实际 mask state，不能只信表中某一列。

## 3. descriptor 主体

Command descriptor 位于 local System Memory，包含：

- AXI base address 与 AXI attributes；
- PCIe base address 与 TLP attributes（Requester ID、TC、RO/IDO/No Snoop 等）；
- transfer length；
- inbound/outbound、continue、interrupt 等 control；
- system/external/channel status；
- next descriptor pointer。

最后一项 `next=0` 且 `continue=0`。中间项指向下一 descriptor 并设置 continue。descriptor 的 status 由硬件回写，所以它既是命令也是回执；cache ownership 和 write ordering 必须正确。

ECC 版本要求两端地址 4-byte aligned 且 length 是 4 的倍数。

## 4. 三种 transfer model

- Bulk：每项从一个连续 source 区域搬到一个连续 destination 区域，大传输拆成 linked list。
- Gather：多段 source 预取到一个 partition，再写入单一 destination。
- Scatter：单一 source 预取后，分写多个 destination。

Scatter/Gather 总数据受一个 RAM partition 限制。手册文本中 OCR 把最大 descriptor length 显示成 `224`，结合 descriptor 的 24-bit length 字段和同章“0 表示 2^24 bytes”，这里应读作 **2^24 bytes**，不是 224 bytes。

## 5. 启动一个 channel

以 Channel 0 为例，寄存器 byte offset：

| offset | 寄存器 | 动作 |
|---:|---|---|
| `0x0` | Control | bit1 `ob_not_ib` 选方向；bit0 `go` 启动 |
| `0x4` | Start Pointer Low | descriptor 首地址低 32 bit |
| `0x8` | Start Pointer High | 高 32 bit（address width>32 时） |
| `0xc` | Attribute Low | fetch/status-writeback AXI attribute |
| `0x10` | Attribute High | attribute width>32 时使用 |

每 channel stride 为 `0x14`，本配置只应使用能力寄存器报告存在的 0–3。

正确的发布顺序：

```text
填完 descriptor/list
 → DMA coherent visibility / required barrier
 → 写 attr
 → 写 start pointer high/low（按硬件要求）
 → 最后写 direction + go
```

§5.5.7 明确要求：启动后，除 linked-list append 规定的动作外，直到 Done/Error 前固件不得访问该 channel 寄存器。

## 6. completion 与 ISR

`PCIe DMA Interrupt @0xa0` 的 bits 0–7 是 channel Done，8–15 是 Error，sticky、write-one-to-clear。ISR 流程：

1. 读 status 并与 enabled mask 相交；
2. 对每个 channel 扫 descriptor status；
3. 遇到 channel status=0 表示该 descriptor 尚未尝试，可停止；
4. 区分 external early completion、system early completion、parity、invalid descriptor、overflow、underflow、not-empty；
5. 把本次已处理的原 status value 写回 W1C；
6. 完成 buffer ownership/cache sync，再通知上层。

只看 Done/Error bit 不足以判断搬了多少；失败前的若干 descriptor 可能已成功。

## 7. Link Down 恢复

正在运行时 Link Down，uDMA 会把 PCIe status 更新为 `LINK_DOWN (status E)`、丢弃 descriptor chain 并报 Error IRQ。固件要：

1. 扫所有 channel，不能只看第一个报错 channel；
2. 等 `LINK_DOWN_RESET_OUT` falling edge；
3. 清 AXI Wrapper Link Down Indication；
4. 重新确认 Link/Data Link、Function BME 和 address mapping；
5. 从业务定义允许的位置重建/restart descriptor chain。

不要原样重放已经部分完成的链，否则“恢复”可能变成重复写数据。

## 8. Linux 集成边界

若把它做成 Linux DMA engine provider，底层 driver 仍需负责 descriptor allocation、register/IRQ、reset/error residue；DMAengine client API 不会自动理解 PCIe External Memory address、Requester ID 或 Cadence status E。当前 workspace 没有该 uDMA driver，具体 `dma_device`/channel glue 为 Unknown。

## 9. Source Reference

- Cadence PCIe Core User Guide, Chapter 5, printed pp.281–298 / PDF pp.370–387。
- Cadence PCIe Core User Guide, §4.7.1, printed p.276 / PDF p.365。
- Cadence PCIe Core User Guide, §§9.6、9.12、9.18、9.24；LINK0 registers printed pp.792–803 / PDF pp.881–892。

