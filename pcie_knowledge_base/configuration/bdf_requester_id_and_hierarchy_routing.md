# BDF、Requester ID 与层次路由：地址牌是谁贴的，包又靠什么回家

## 1. BDF 是软件名字，也是协议身份的来源

传统 BDF：

```text
0000:01:00.0
^^^^ segment/domain
     ^^ bus
        ^^ device
           ^ function
```

一个 PCIe hierarchy 内，legacy Routing ID 是 16 bit：Bus[7:0]、Device[4:0]、Function[2:0]。Linux 常把 Device+Function 合成 8-bit `devfn`：

```c
PCI_DEVFN(slot, func) = (slot << 3) | func;
PCI_SLOT(devfn); PCI_FUNC(devfn);
```

segment/domain 不在 16-bit PCIe Requester ID 里；它由 Host/软件在多个 hierarchy/segment 之间消歧。

## 2. Bus/Device Number 不是 EP 烧死的序列号

Bus number 和 bridge 的 Secondary/Subordinate range由枚举软件分配，热插拔/重配置时可以变化。直连非 ARI Endpoint 在该 Link 逻辑 bus 上通常是 Device 0；Function 0–7 表示同一 device 的多个 Function。

Endpoint 会从完成的 Type 0 Configuration Write 中捕获分配给自己的 Bus/Device number，随后把它们放入自己发出的 Requester ID/Completer ID。于是：

```text
Host 枚举并给 EP 分配 01:00.x
  → EP 捕获 Bus=1, Device=0
  → EP DMA MRd 的 Requester ID=01:00.function
  → Completion 用该 Requester ID 路由回来
  → IOMMU/ACS/AER 也用 RID 识别来源
```

BDF 不是只给 `lspci` 打印；它进入 DMA isolation、Completion matching、error attribution 和虚拟化安全边界。

## 3. Requester ID、Completer ID、Tag 的分工

- Requester ID：谁发起 request；全 hierarchy 内唯一标识 Function。
- Tag：该 Requester 当前是哪一笔 Non-Posted transaction。
- Requester ID + Tag：Transaction ID，用于把 Completion 匹配回请求。
- Completer ID：谁完成了 request；用于报告和 ordering 等语义。

Completion header 会复制原 Requester ID 和 Tag。Switch 用 Completion 中的 Requester ID 做 ID routing，把它送回原 Requester；不是靠读数据的地址反向猜路。

## 4. Config 路由为何需要 Type 0/Type 1

Type 1 在桥树中寻找目标 bus：

- target bus 落在 Downstream Port 的 Secondary..Subordinate 范围，向下转发；
- target bus 正好等于 Secondary，出端口时转换 Type 0；
- Type 0 到达目标所在 Link，由本地 Function decode。

Memory TLP 则主要按 address 和 bridge Base/Limit window 路由。BDF 分配正确不代表 BAR window 正确，这是两个平行控制面。

## 5. ARI 怎样把 8 个 Function 扩到 256 个

ARI 把传统 `Device[4:0]+Function[2:0]` 的 8 bit 重新解释为 Function[7:0]，Device number 隐含为 0：

```text
non-ARI RID = Bus[7:0] : Device[4:0] : Function[2:0]
ARI RID     = Bus[7:0] : Function[7:0]
```

因此 ARI Extended Function >7 只有在 Downstream Port/软件启用并理解 ARI forwarding 后才能访问。Linux 常规 `devfn` 恰好仍是 8 bit，但其 bit 解释随 ARI 改变；不要盲目对 ARI `devfn` 使用 `PCI_SLOT/PCI_FUNC` 后当成传统 BDF 语义。

## 6. 一个 DMA/IOMMU 例子

PF `01:00.0` 与 VF（ARI Function 9）都向 IOVA `0x8000_0000` 发 MRd。即使地址相同，IOMMU 可用不同 RID（必要时再加 PASID）选择不同 translation domain。若 device 错填 Requester ID：

- Completion 可能回错 Function或成为 Unexpected Completion；
- IOMMU mapping/permission 选错；
- ACS Source Validation 报错；
- AER Error Source Identification 指向错误设备。

所以“DMA 地址对”只是三分之一，RID 和权限上下文同样重要。

## 7. Linux 对象

`struct pci_dev` 保存所在 `pci_bus` 与 `devfn`；`pci_domain_nr(bus)` 给 segment，`pci_name(pdev)` 生成软件可读地址。driver 日志应打印完整 domain:BDF，尤其多 Root Complex 系统中不要省 domain。

## 8. Debug

- Config 首次成功但 DMA IOMMU fault：比对 TLP Requester ID 与 Linux BDF/IOMMU group。
- 桥后所有设备消失：查 bus-number range，不要逐个重写 EP BAR。
- 只有 Function >7 不见：查 ARI capability/forwarding、Next Function Number 和软件扫描。
- Completion Unexpected：同时核对 Requester ID、Tag、Function reset/renumber 时是否仍有 outstanding。
- SR-IOV VF fault 归到 PF：查 VF RID allocation/stride、descriptor Requester ID 字段。

## 9. Source Reference

- PCIe Base Specification 5.0 r1.0, §§2.2.4.2、2.2.6.2–.3、2.3.2、7.3.1。
- PCIe Base Specification 5.0 r1.0, §6.13 ARI；§7.2 ECAM。
- Cadence PCIe Core User Guide, §§10.3.1、10.5.1。
- Ubuntu 6.8 headers, `include/uapi/linux/pci.h:31–33`、`include/linux/pci.h` public declarations。

