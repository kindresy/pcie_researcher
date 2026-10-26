# 王齐第 4–5 章 OCR 评估：从 PCIe 世界观走到地址域

日期：2026-08-23

本轮逐页 OCR 阅读：

- 第 4 章，printed pp.101–139 / PDF pp.113–151；
- 第 5 章，printed pp.140–154 / PDF pp.152–166。

这两章的价值不在于替代 PCIe 5.0 规范，而在于回答驱动工程师最早会问的两类问题：**这棵树到底长什么样，以及 CPU 眼里的地址为什么不能和 PCIe 地址画等号。**

## 1. 第 4 章：仍然好用的体系结构直觉

### 1.1 点到点、双单工、Lane 与协议分层

教材从并行共享 PCI 总线切换到 point-to-point serial Link，再解释每条 Lane 有独立发送/接收差分对，能自然说明为什么 PCIe 需要 Switch、routing、packet 和 flow control。它把 Transaction/Data Link/Physical Layer 类比为硬件实现的网络协议栈，这个比喻对初学者仍然有效。

但数字属于 2010 年：书中把当时尚未正式落地的 Gen3 描述成“4 GHz”，只能当历史预测。现行速率、编码和有效吞吐以 Base Spec 5.0 与 [`link/link_lane_ltssm_and_training.md`](../link/link_lane_ltssm_and_training.md) 为准。

### 1.2 RC、Switch、EP 与端口方向

教材用拓扑实例解释：

- RC 把 CPU/memory subsystem 接入 PCIe hierarchy；
- Switch 的 Upstream Port 面向 RC，Downstream Port 面向 EP/下级 Switch；
- Ingress/Egress 取决于某一笔 packet 的流向，不是端口焊死的第二个名字；
- 软件把 Switch 端口建模成 PCI-to-PCI bridge，从而复用 Bus Number 与 bridge window 模型。

其中“每扩展一条 Link 就产生一个 Bus Number”适合建立初始直觉，但不能当硬件定律。更准确的说法是：配置软件给 bridge secondary side 分配 Bus Number，多个 Function 可以共享同一 Bus；编号是 enumeration 的软件结果，不是 Lane 训练完后 PHY 自动吐出来的纪念品。

Crosslink、MR-IOV 等段落保留为历史/多主机背景；面向当前开发的 NTB 路径已独立整理到 [`advanced/non_transparent_bridge_and_multi_host.md`](../advanced/non_transparent_bridge_and_multi_host.md)。

### 1.3 Reset 的价值与边界

第 4.1.5 节区分 Cold/Warm/Hot Reset 与 FLR，并强调 FLR 只针对一个 Function，不让整条 Link 跟着“陪跑”。稳定原则已经进入 [`controller/cadence_reset_and_recovery_matrix.md`](../controller/cadence_reset_and_recovery_matrix.md)。

旧书对具体保存字段的罗列不能直接套用到 Cadence build；当前 controller 的 Link Down、Hot Reset、FLR reset domain 必须看 Databook，而且手册内部还存在 OQ-011 冲突。

## 2. Capability 部分发现的两处重要年代差异

### 2.1 PCI Express Capability 不是可有可无

书中说许多 PCIe 设备可能不存在 PCI Express Capability。对真正的 PCIe device Function，这不符合 PCIe 5.0：该 capability 是 required，Status.Capabilities List 也必须为 1。可选的是部分寄存器组或其他 capability，不是 ID `0x10` 的整座房子。

### 2.2 Extended Capability 的空链编码不能混用

书中第 4.3.3 节把“Configuration Space 没有 Extended Capability”写成 ID=`ffffh`。PCIe 5.0 §§7.6.1–7.6.2 的规则是：

- Function Configuration Space 在 `0x100` 用 ID=`0000h`、version=`0`、next=`000h` 表示空链；
- RCRB 的空链才用 ID=`ffffh`、next=`000h`；
- 最后一个真实 capability 只把 next 设为 0，自己的 ID/version 仍然有效。

这两项已经补入 [`configuration/capability_discovery_and_safe_walking.md`](../configuration/capability_discovery_and_safe_walking.md)。这也是为什么教材必须和规范交叉：老书不是坏书，只是偶尔会把钥匙插进隔壁房门。

### 2.3 MPS 不是 Link 自动“协商”完就不用管

教材已经正确区分 `Max_Payload_Size Supported` 与 Device Control 中的实际 MPS，但用“链路两端协商”容易让读者联想到 LTSSM 自动谈判。实际上 configuration software 要为 hierarchy 选择并编程合法 MPS；硬件只报告能力并执行限制。详见 [`transactions/mps_mrrs_ordering_and_performance.md`](../transactions/mps_mrrs_ordering_and_performance.md)。

## 3. 第 5 章：Montevina 旧平台，为什么今天仍值得读

### 3.1 最有价值的是“域”，不是老芯片寄存器名

教材把 2008 年前后的 Intel Montevina 平台拆成：

```text
CPU memory domain
  ├── DRAM-visible ranges
  ├── chipset/internal MMIO
  └── PCI MMIO aperture
          ↓ host bridge decode/translation
      PCI bus address domain
```

同一个数值在两个域里碰巧相等，不代表它们是同一种地址；CPU 访问 PCI aperture 与设备 DMA 访问 host memory 更是相反方向。这条原则直接映射到：

- [`addressing/bar_and_cadence_address_translation.md`](../addressing/bar_and_cadence_address_translation.md)；
- [`dma/pcie_dma_cadence_and_linux_dma_api.md`](../dma/pcie_dma_cadence_and_linux_dma_api.md)；
- [`advanced/iommu_ats_pasid_pri.md`](../advanced/iommu_ats_pasid_pri.md)。

教材还用 x86 的 memory hole/reclaim 说明 DRAM physical organization 与 CPU physical map 也未必一一对应。这是理解 IOMMU/IOVA 前很好的热身：地址相等只是一种实现选择，绝不是 API contract。

### 3.2 ECAM 的计算仍然稳定

Montevina 用 256 MiB PCIEXBAR 映射一个 8-bit Bus Number segment：

```text
ECAM offset = bus << 20 | device << 15 | function << 12 | register
```

每个 Function 占 4 KiB；某个 Device/Function 不存在时，对应洞也照样占 address geometry。CPU 对 ECAM 的 load/store 经 host controller 变成 Configuration Request，Type 0/Type 1 取决于目标位置与 controller routing。

这条协议直觉仍有效，但“ECAM 固定在 `0xe0000000–0xefffffff`”只是该平台实例，绝不是 PCIe 标准地址。现代 firmware 用 ACPI MCFG、Device Tree ranges 或平台机制描述真实窗口。

### 3.3 旧 Linux 调用链只作考古样本

第 5.3.2 节展示 2010 年 x86 的 `raw_pci_read/write`、`raw_pci_ops` 与 `pci_mmcfg_*`。它能说明 legacy CF8/CFC 与 ECAM backend 的选择思想，却不能作为当前 Linux 6.x 调用链证据：目录结构、锁、ops 层和架构 glue 已演化，而 workspace 没有完整 kernel C source。

因此知识库只吸收“PCI core 最终调用 host config-access backend”这一稳定架构，精确函数/代码行继续标为资料阻塞，不拿十五年前的函数名给今天的 kernel 戴工牌。

## 4. Coverage 结论

| 范围 | 状态 | 去向 |
|---|---|---|
| Ch.4 Link/Layer/Topology | D+M | fundamentals、link、configuration |
| Ch.4 reset | D+M | controller reset matrix |
| Ch.4 PM/PCIe/Extended Capability | D+M | power、capability walker；新增两项规范校正 |
| Ch.4 VC arbitration | D+M | TC/VC/QoS 与 Flow Control 文档 |
| Ch.5 Montevina registers/layout | E | 作为特定历史平台实例，不复制寄存器表 |
| Ch.5 address domains/reclaim | D+M | addressing、DMA、IOMMU |
| Ch.5 ECAM | D+M | configuration/ECAM；旧 Linux 实现保留年代边界 |

## 5. Source Reference

- 王齐，《PCI Express体系结构导读》，Chapters 4–5，2010 年第一版，扫描 PDF pp.113–166。
- PCI Express Base Specification 5.0 r1.0, §§7.5.1.2、7.5.3、7.6.1–7.6.3。
- Cadence *PCIe Core User Guide*, reset、configuration、address translation相关章节。
