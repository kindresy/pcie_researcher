# MPS、MRRS 与 Ordering：TLP 大小、顺序与性能

## 1. 四个经常被混成一个旋钮的概念

| 概念 | 限制/决定什么 | 作用位置 |
|---|---|---|
| MPS | 一个**带数据 TLP**最多带多少 payload | transmitter 和 receiver 都必须遵守 |
| MRRS | 一个 Function 作为 Requester 时，一笔 Read Request 最多请求多少数据 | 只限制它发出的 read request |
| RCB | Completer 可在哪里切分一次 Memory Read 的返回数据 | Completion 切包边界 |
| Flow Control credit | 当前下一跳是否有 buffer 接收某类 header/data | 每条 Link、每 VC、逐跳运行时资源 |

MPS 与 MRRS 的编码范围都是 128、256、512、1024、2048、4096 bytes，但长得像不等于作用相同。

## 2. MPS：限制单个 TLP 的数据负载

MPS（Max Payload Size）适用于带 data payload 的 TLP，例如 Memory Write 和 Completion with Data。它**不直接限制 Memory Read Request 的 Length**，因为 MRd header 只是描述“请给我多少”，请求本身没有那些读数据。

规则是双向的：

- transmitter 不得生成 payload 大于自身 Device Control.MPS 的 TLP；
- receiver 收到 payload 大于自身 MPS 的 TLP，必须判为 Malformed TLP。

系统软件若把 MPS 从默认 128 B 调大，必须保证这笔 TLP 路径上的每个 System Element 都支持该大小。路径中任何 Switch/Port 只支持 256 B，Endpoint 发 512 B 就不是“可能慢一点”，而是协议错误。

### 例子：写 600 bytes

若 MPS=256 B，一次 600 B DMA write 至少拆为 payload `256 + 256 + 88` 的三个 MWr TLP（还要遵守 4 KiB boundary、alignment/BE 等规则）。若 MPS=128 B，则至少五个。更大的 MPS 可减少 header/LCRC/framing 开销，但也会：

- 增加单个 packet 占用 Link 的时间和其他流的等待延迟；
- 要求更多 receive/replay buffer；
- 一次 replay 重发更多数据；
- 受整条 hierarchy 的最小能力约束。

所以“拉到 4096”不是性能调优，是愿望调优。

## 3. MRRS：限制单个 Memory Read Request 请求的数据量

MRRS（Max Read Request Size）只限制 Function 作为 Requester 时生成的 Read Request size。默认编码 `010b` 是 512 B；实现可以只支持 128 B read 并把字段做成只读。

### 例子：EP DMA 读 Host 1024 bytes

假设 MRRS=512 B，DMA engine 至少发两个 MRd；若 MRRS=128 B，至少八个。较大 MRRS 通常减少 request header 和仲裁次数，但不保证 Host 用一个 CplD 返回：Completer 仍受它的 MPS、RCB、4 KiB 边界及内部策略影响，可能把 512 B read 拆成多个 CplD。

较大 MRRS 还会占用更多 outstanding/completion buffer，并可能影响不同 Requester 的公平性。Spec 明确把 MRRS 描述为 QoS/bandwidth allocation 的控制手段之一。

## 4. RCB：为什么一个 MRd 会回来好几个 CplD

Read Completion Boundary 对 Root Complex 是 64 B 或 128 B，并在 Link Control 中报告；其他 System Element 是 128 B。跨越自然对齐 RCB 的 read 可被拆成多个 Completion，每个 CplD 又不得超过 Completer 的 MPS。

例如从地址 `0x1040` 读 256 B，RCB=128 B：返回可能在 `0x1080`、`0x1100` 等边界切分。Requester 用 Requester ID + Tag 匹配同一 request，再依据 Byte Count/Lower Address 收齐。MRRS 不是 Completion payload size 的别名。

## 5. Flow Control：配置允许发大包，不代表此刻有 credit

Data Link 为 Posted、Non-Posted、Completion 分别维护 Header/Data credit。发送端只有拿到足够对应 credit 才可发送；接收端消费 buffer 后用 UpdateFC DLLP 返还额度。

关键关系：

- MPS 决定最大 data packet，因此影响 receiver buffer/credit sizing；
- MRRS 增大可能带来更多/更大的 Completion 数据占用；
- credit 是每跳的，TLP 穿过 Switch 后下一条 Link 重新受那一跳 credit 约束；
- posted traffic 不能因为 NP queue 堵住而永远无法前进，ordering/credit 规则共同防止 deadlock。

性能差时要同时看 payload efficiency、outstanding depth、credit starvation、replay rate和应用 burst；只看 negotiated Gen4 x8 像只看公路限速牌，不看收费站排队。

## 6. Ordering 的直觉：doorbell 不能超过它指向的数据

典型 producer/consumer：

```text
1. EP/CPU 写 descriptor/data       （Posted MWr）
2. 再写 doorbell                   （Posted MWr）
3. device/peer 看到 doorbell 后读数据
```

默认同一 Traffic Class 下，后发的 Posted Request 不得越过先发的 Posted Request。这保证 doorbell 不会先于 descriptor 对对端可见。类似地，后发 read 默认不能越过先前 posted write；driver 常用一次 readback 把 posted write 的可见性推进到需要的点。

但 ordering table 不是“所有 TLP 全局排成一列”：

- 规则在单个 TC 内适用，不同 TC 之间没有 ordering requirement；
- 某些“必须能通过”是为了避免 protocol deadlock；
- RO（Relaxed Ordering）和 IDO（ID-Based Ordering）允许特定 passing；
- Switch/bridge、Completer 与 AXI 还要把 PCIe 规则映射到各自队列/ID。

## 7. RO、IDO、No Snoop 别串台

### Relaxed Ordering

RO 表示该 transaction 不需要默认的强 ordering，可允许例如后发 posted request 越过先发 posted request。只有数据和 doorbell 确实无依赖时才能用；否则性能优化会升级为很难复现的数据损坏。

Configuration、I/O、MSI/MSI-X Memory Request 等不允许使用 RO（相关 Attr bit 必须清零）。

### ID-Based Ordering

IDO 可按 Requester/Completer ID，某些情况下再结合 PASID，区分互不依赖的 ordering domain。Requester/Completer 只有相应 Device Control 2 enable 后才可设置。不能简单地把它概括为“相同 ID 保序、不同 ID 可以任意通过”，具体 passing 仍由 Table 2-40 的条目约束。

### No Snoop

No Snoop 是 cache coherency 属性，**不改变 transaction ordering**。Function 只有能保证目标地址不在需要硬件一致性维护的 cache 中时才可设置。把它当成“Relaxed Ordering 加强版”会把 cache bug 和 ordering bug打包寄给未来的自己。

也不要照搬早期 FSB 教材里的优化配方：“软件先 flush 一次 cache，再把所有大块 DMA 的 No Snoop 置 1”。这是否安全取决于平台的 coherency domain、目标 memory type、DMA mapping 属性、CPU 后续访问以及 IOMMU/Root Complex policy；一次 flush 不是永久结界。Linux Function driver 应遵守 DMA API 和平台提供的属性，不因 descriptor 很大就私自改 TLP No Snoop。王齐教材 §6.1.3 的 FSB 例子只能说明这个 bit 为什么可能节省 snoop traffic，不能证明它在当前 SoC 上可用。

## 8. Cadence 怎样把 PCIe ordering 映射到 AXI

### Inbound：Link → core AXI Master

Cadence 对 posted write 使用相同 AXI AWID，保证本地 AXI 子系统按序完成。在后续 Non-Posted read/write 前，core 等待先前 posted write 的 `BVALID` 返回，落实 PCIe producer/consumer ordering。

core 可有最多 64 个 inbound outstanding AXI read，并保存 PCIe Tag↔AXI ID。不同 AXI ID 的 read data 可乱序返回，core 再生成对应 Completion。

### Outbound：client → core AXI Slave → Link

write path 为 cut-through，TLP 按 AXI Slave 接受顺序发出。read 请求映射为唯一 PCIe Tag；如果多个 AXI read 使用相同 ARID，而 Completion 乱序回来，Cadence reordering RAM 会先恢复 AXI 对同 ID 要求的顺序。若 ARID 不同，数据可按 Completion 到达次序分别返回。

Cadence region descriptor/sideband 可携带 No Snoop、RO、IDO。能写这些 bit 不表示应该默认全开；client 是发起语义的那一方，必须知道 transaction 是否存在依赖。

## 9. Linux 驱动能做什么

X6000 Linux 5.10.4 公开并实现以下 API：

```c
int pcie_get_readrq(struct pci_dev *dev);
int pcie_set_readrq(struct pci_dev *dev, int rq);
int pcie_get_mps(struct pci_dev *dev);
int pcie_set_mps(struct pci_dev *dev, int mps);
```

但 Function driver 不应在不知道 hierarchy 策略时擅自把 MPS 调大。MPS 是路径协同配置；MRRS 是本 Function requester 上限，也仍受控制器 outstanding/buffer 和工作负载影响。任何调参都应记录调前后：MPS/MRRS、平均 TLP payload、CplD 数、credit stall、replay、吞吐、尾延迟和 error。

软件内存屏障与 PCIe ordering 也不是互相替代：barrier 约束 CPU/compiler/架构的访问发出顺序，PCIe ordering 约束 transaction 在 fabric 中的 passing；完整正确性需要两边都满足，并遵循 device register 的 flush/readback 规则。

## 10. 性能排查顺序

1. `LnkSta` 实际 speed/width 与 replay/error rate；
2. DMA burst/SG 是否能形成足够大的 payload；
3. MPS、MRRS、RCB 与实际 TLP/CplD 分布；
4. outstanding Tag/AXI ID 深度是否吃满；
5. P/NP/Cpl header/data credit 是否长期为零；
6. RO/IDO 是否安全且真正解除了阻塞；
7. ASPM exit、IOMMU、cache sync 和软件 doorbell batching。

只记录“带宽低”不足以定位问题。性能测试应同时保留 transaction size histogram、stall reason、replay/error 和实际 Link speed/width。

## 11. Source Reference

- PCIe Base Specification 5.0 r1.0, §2.2.2, printed/PDF pp.109–110：payload/MPS rules。
- PCIe Base Specification 5.0 r1.0, §2.2.6.4–.5, pp.125–126：RO/IDO/No Snoop。
- 王齐《PCI Express体系结构导读》，§6.1.3，printed pp.159–160 / PDF pp.171–172（历史 FSB 优化例及其平台边界）。
- PCIe Base Specification 5.0 r1.0, §2.3.1.1, pp.168–170：Completion split/RCB。
- PCIe Base Specification 5.0 r1.0, §2.4, pp.177–183：ordering rules。
- PCIe Base Specification 5.0 r1.0, §2.6, pp.193–203：Flow Control。
- PCIe Base Specification 5.0 r1.0, §7.5.3.4, printed/PDF pp.725–729：Device Control MPS/MRRS/attribute enable。
- Cadence PCIe Core User Guide, §§4.5.2.7、4.5.4、4.5.6.3.3、4.5.6.4.4、4.5.6.8。
- X6000 Linux 5.10.4 `include/linux/pci.h` 与 `drivers/pci/pci.c`：MPS/MRRS API 及实现。
