# Common Misconceptions

PCIe 有很多“听起来差不多”的东西。下面先把高频坑插上警示牌；每条会在对应主题完成后补足字段和调用链。

## BAR 是一块 memory 吗？

不是。BAR 是 Configuration Space 中描述地址窗口属性、大小并接受系统分配基址的寄存器。窗口背后可以接寄存器、SRAM、doorbell、frame buffer 或设备内部地址翻译逻辑。把门牌号当房子本体，装修时很容易敲到邻居家。

## PCIe address 等于 CPU physical address 吗？

不保证。Host bridge 可以把一段 CPU physical MMIO window 翻译到另一段 PCI bus address。是否相等由平台地址图和 outbound translation 决定。

## DMA address 等于 physical address 吗？

不保证。它是设备应该发出的地址；在 IOMMU 存在时通常是 IOVA。即使没有 IOMMU，架构 DMA offset、bridge window 等也可能让它不同于 CPU physical address。

## `pci_iomap()` 返回的是 PCIe 地址吗？

不是。它返回 CPU 可用的 `__iomem` 虚拟地址。CPU 访问该虚拟地址后，经页表/I/O mapping、host bridge window，最终才可能变成某个 PCIe address 的 Memory TLP。

## Memory Write TLP 有 Completion 吗？

正常 Memory Write 是 Posted Request，没有正常 Completion。Data Link ACK 也不是 Completion，它只确认相邻 Link 正确收到包。若要确认写入在协议或设备层生效，驱动通常需要设备定义的状态、doorbell/queue 语义，或一次具有 ordering 意义的 read；具体方法必须依据设备协议，不能脱离适用条件泛化。

## Memory Read 为什么需要 Completion？

Memory Read 是 Non-Posted Request。Requester 发 MRd 后保留 Tag/outstanding state，Completer 用一个或多个 CplD 返回数据。请求与返回是 split transaction，不会长期占住共享总线等回应。

## PCIe DMA 是 PCIe Controller 替 Endpoint 搬数据吗？

不一定。PCIe 标准定义的是设备作为 Requester 发 Memory TLP 的能力，具体 DMA engine 可以在 Endpoint application logic、Cadence 集成 DMA block 或其他本地模块中。PCIe controller 负责把请求变成合规 TLP，并处理 Completion/flow control/link；谁读 descriptor、切块和报完成是实现问题。

## MSI 是一根新硬件中断线吗？

不是。MSI 的本质是设备向系统配置的 Message Address 写入 Message Data，即一笔 Memory Write。系统的中断控制器或重映射单元把这笔写解释为中断，因此不需要为每个中断事件增加一根独立物理信号线。

## MSI到了，之前的DMA数据就一定可见吗？

同一Function、同一TC、默认ordering的早先Posted Writes不能被后发MSI MWr超车。但如果data和MSI使用不同TC，两个TC之间没有默认顺序保证；device必须使用同TC或zero-length Read等明确同步手段。“中断来了但buffer还旧”不一定是cache bug，也可能是跨TC ordering设计错误。

## PCIe 上的 INTx 还是四根贯穿链路的线吗？

不是。PCIe 保留传统 INTx 的软件/level semantics，但在 Link 上用 Message 传递 assert/deassert 事件。设备封装可能还有本地 sideband interrupt interface，那是 controller 与 application logic 的接口，不等于 PCIe fabric 上的物理 INTx 线。

## LTSSM 属于软件状态机吗？

不是。LTSSM 是 Physical Layer 硬件状态机。软件可以读状态、开关 training 或请求 retrain，但 Detect/Polling/Configuration 等训练过程由链路两端硬件完成。

## Enumeration 是 Endpoint 自己完成的吗？

不是。EP 暴露 Configuration Space 并响应 Configuration Requests；firmware/OS 从 RC 出发扫描 topology、分配 Bus Number 和资源、写 BAR/bridge window，并创建软件设备对象。

## Bus Number 是硬件固定的吗？

通常不是。Bus Number 是枚举层次时写入桥寄存器并用于 ID routing 的拓扑编号。设备换槽或桥层次变化后 BDF 可以改变，驱动不应把某个 BDF 当永久身份。

## Root Complex 和 Root Port 是同一个东西吗？

不是。RC 是连接 CPU/memory subsystem 与 I/O hierarchy 的根；一个 RC 可以有多个 Root Port。RP 是其中向下游 hierarchy 暴露 Link 的端口/桥型 Function。

## Endpoint 只能被 Host 访问，不能主动发请求吗？

错误。EP 可作为 Completer 响应 BAR 访问，也可作为 Requester/Bus Master 主动 DMA Host Memory。Cadence 的 AXI 名称以 **core** 为观察者：inbound 时 core 是 AXI Master，outbound 时 client 访问 core 的 AXI Slave；两条路径不能只凭 `master` 这个词猜 PCIe 方向。

## Link Up 就等于设备已经可用吗？

远远不够。Link Up 通常只表示 LTSSM 到达可传输状态。配置访问、枚举、BAR/resource、Command.MSE/BME、DMA mapping、MSI 和功能驱动都还可能失败。它像“网线通了”，不是“应用已经登录并跑完业务”。

## Function power state 与 Link power state 是一回事吗？

不是。D0/D1/D2/D3 描述 Function 电源状态；L0/L0s/L1/L2 描述 Link 状态。二者有关联，但属于不同状态机和控制面。

## `pci_enable_device()` 会自动允许 DMA 吗？

不能这样概括。是否设置 Bus Master Enable、资源与电源状态如何处理，需要沿目标 Linux 版本源码确认。驱动通常显式调用 `pci_set_master()` 允许设备发 Memory/I/O Request；当前目录没有内核源码，完整结论先保持待验证。

## MPS 和 MRRS 都是“最大包长”吗？

不是。MPS 限制带数据 TLP 的 payload；MRRS 限制 Function 一笔 Memory Read Request 最多请求多少数据。一个 MRRS=512 B 的 MRd 仍可能因 RCB/MPS 被拆成多个 CplD。

## Flow Control Credit 是端到端额度吗？

不是。Credit 在每条 Link 两端逐跳维护；Switch 收到 TLP 后，在下一条 Link 上重新受那一跳的 P/NP/Cpl header/data credit 约束。它不会从 EP 带着一张“Host 全程通行证”穿过所有 Switch。

## No Snoop 会让 TLP 更容易乱序吗？

不会。No Snoop 是 cache coherency 属性，规范明确说它不改变 transaction ordering。需要讨论 passing 时看 RO/IDO 和同一 TC 的 ordering rules。

## VC 是把物理 Lane 虚拟成多个通道吗？

不是。Lane 是 Physical Layer 的收发差分对；VC 是复用在同一 Link 上、拥有独立 queue/buffer 与 Flow Control credit 的逻辑流。TC 是包上的分类标签，VC 才是隔离阻塞的资源。当前 Cadence 实例只有标准基线 TC0/VC0，没有选中额外 VC Capability。

## TC 数字越大，优先级就越高吗？

规范没有定义这种朴素的数字优先级。TC0～TC7 是服务类别标签，真实 QoS 取决于整条 hierarchy 的 TC→VC mapping、Port arbitration 和可用资源。只在一个 EP 上写 TC7，不会召唤一条 VIP Lane。

## Capability 总图画了一个方框，当前芯片就一定支持吗？

不一定。可配置 IP 手册常同时描述“家族可实现能力”和“当前 build 实际选中能力”。本 Cadence 手册的通用图画了 VC/PTM 偏移，但 configuration-specific tables 没列出它们，VC error status 还明确 tied off。判断必须交叉看 capability 链、实例表、寄存器块和实机 config dump。

## `LANE_MAP_MODE` 是 Link training 自动选出来的吗？

不是。Cadence 把它定义为静态 strap，在训练前先决定 16 条 PHY Lane 分给哪些独立 Link core；每条 Link 随后才在自己获得的最大宽度内训练/降宽。

## Link Down、Hot Reset 和 FLR 可以用同一恢复流程吗？

不应。Link Down/Hot Reset 影响整条 Link；FLR 只针对一个 PF/VF 且不重训 Link。Cadence 明确说 AXI translation 在这些路径下保留，FLR 会清目标 Function BME；但 Link Down 是否保留 LM，§4.7.2 与 §11.1.2.3 互相冲突，记为 OQ-011。恢复必须按 reset scope 和 readback 证据重建状态。

## 找到 Capability 就能直接置 Enable 吗？

不行。Capability 只证明 Function 暴露这套结构；ARI、ATS、MSI-X、SR-IOV 等还需要上游 path、平台/IOMMU/IRQ资源和 Linux subsystem 共同准备。背着内核直接改 enable bit，通常会得到一套“硬件开了、软件不知道”的分裂人格。

## Source Reference

- PCIe Base Specification 5.0 r1.0, §§1.2–1.5, Chapter 2, §1.3.2.2。
- Cadence PCIe Core User Guide, §4.1, printed pp.202–204 / PDF pp.291–293。
- 具体 BAR、DMA、MSI、LTSSM 与 Linux API 结论会在对应独立文档中添加精确 section/page/code locator。
