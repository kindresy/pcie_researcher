# Message、AtomicOp 与 TLP Prefix：Memory Read/Write 之外还有什么包

## 1. Message TLP：把“事件”也放进分组网络

PCIe 没有给所有控制事件拉专线，而是定义 Message Request。它可按不同 routing code：向 Root Complex、按地址、按 ID、本地终止、广播等方式传播；Message Code 再说明语义。

常见例子：

- INTx Assert/Deassert；
- PM_PME / PME_TO_Ack；
- ERR_COR/ERR_NONFATAL/ERR_FATAL；
- Vendor_Defined Message；
- ATS Invalidation、PRI Page Request 等扩展协议消息。

Message 可以有或没有 data，绝不能看到“消息”二字就当作 MSI：**MSI/MSI-X 是 Memory Write，不是 Message TLP。**

Cadence outbound region 可把 AXI aperture 定义成 Message TLP。Chapter 10 的实现要求 Message region 至少 128 KiB，因为 routing、Message Code、with/without-data 等信息从 AXI address bit 携带；其余 header 由 descriptor/translation register 提供。

## 2. AtomicOp：把 read-modify-write 变成一笔事务

AtomicOp 对目标 Memory Space 位置执行不可分割 read-modify-write，包括 FetchAdd、Swap、CAS。它是带 data 的 Non-Posted Memory transaction，必须返回 AtomicOp Completion（带原值或状态）。

完整路径必须每一环都支持：Requester capability/enable、所有 routing element、Completer 对应 operand size。软件还要检查 Device Capabilities 2 / Device Control 2；不支持的正确形成请求通常应返回 UR，而不是偷偷退化成普通 read+write。

AtomicOp 的价值是跨 PCIe fabric 做共享计数器或同步，而不需要软件用两笔可被插队的事务。但它不自动解决 cache coherency、地址 permission 或 endian；ATS translated address 还必须具备适当权限。

**当前 Cadence 配置明确写明 `AtomicOps are not supported in this configuration`。** 因此本项目不能因为 Base Spec 有 AtomicOp 就在该 controller 上使用。它仍值得认识，因为收到 AtomicOp 后的 UR/AER、capability bit 和协议分析都可能遇到。

## 3. TLP Prefix：在标准 header 前加上下文

Prefix 位于 TLP header 前。两大类：

- Local TLP Prefix：只在特定本地处理范围内有意义；
- End-End TLP Prefix：由 Requester 到最终 Completer 保留/解释。

本资料中最重要的用途是 PASID Prefix：给 Requester ID 再加 process address-space identity，并携带 Execute/Privileged 等语义。TPH 也可携带 Steering Tag hint，让平台把处理/缓存位置靠近预期 CPU 或资源。

Prefix 会改变 analyzer 解码位置、AER header log 解释和最大 header 开销。AER 支持时还可能记录 TLP Prefix Log；只保存四个 Header Log DWord，遇到 PASID fault 时可能少掉最关键的“这是哪个进程”。

## 3.1 历史包袱：MRdLk/CplDLk/Unlock不是给原生PCIe EP做原子操作的

PCIe 5.0仍保留Locked Transaction sequence，但目的是让Host CPU的旧software能访问PCI/PCI-X legacy target，不是鼓励现代Endpoint把它当成AtomicOp的另一种写法。

```text
Host CPU locked RMW
  → Root Complex发MRdLk
  → Switch/PCIe-to-PCI bridge锁住相关TC0/VC0 path
  ← CplDLk返回旧值并确认lock
  → 普通MWr写回修改值
  → Unlock Message释放path
```

关键限制：

- 只有Root Complex可发起Locked Request；
- MRdLk/CplDLk/Unlock只能用TC0，而lock影响相关VC0 path；
- 一个hierarchy domain同时只能尝试一条locked sequence；
- 不管读是否成功，尝试过的sequence都要用Unlock结束；
- 原生PCIe Endpoint不支持lock，收到MRdLk必须当作UR；
- 新software不应对I/O device使用导致locked access的CPU instruction，因为它可造成严重性能退化甚至deadlock。

它和AtomicOp的根本差别是：AtomicOp是Completer针对一个memory location执行一笔不可分割RMW transaction；legacy lock则在fabric中锁住一条path，再跑read–modify–write序列。一个是手术刀，一个更像为了换一个灯泡先封路。

当前Cadence interface对各Link的AXI `AWLOCK/ARLOCK`明确标注为不支持并忽略；inbound message table能识别Unlock Message，但手册没有给出一套可供client发起完整MRdLk sequence的编程机制。这与“原生EP不应发起lock”一致。

## 4. TH、TPH 与“Hint”

TPH（Transaction Processing Hints）提供 Steering Tag 等 hint，帮助平台优化 transaction processing location。它不是 routing address，也不是硬保证。Cadence 本 build 暴露 PF/VF TPH Requester capability、enable/mode outputs 和 outbound sideband字段；实际启用仍需 Host capability/policy。

Hint 被忽略不应破坏功能正确性。若“不按 TPH 走就数据错”，设计已经把性能提示偷偷升级成协议依赖。

## 5. Debug 解码顺序

```text
先看 Prefix Type/数量
 → 再定位真正 TLP header
 → Fmt/Type 判断 Memory/Config/Message/Atomic/Completion
 → routing/ID/address
 → Attr/TC/Tag/Length
 → payload/ECRC
```

- Message 不到：查 routing code，不只查 address。
- AtomicOp UR：查 requester enable、每跳 routing capability、completer operand support；当前 Cadence build预期不支持。
- PASID/IOMMU fault：保存 Prefix Log、RID、PASID、address type/permission。
- TPH 无效果：先确认它只是 hint，再查 requester enable/ST mode/table和平台支持。

## 6. Source Reference

- PCIe Base Specification 5.0 r1.0, §§2.2.8、2.2.10、6.5、6.15。
- PCIe Base Specification 5.0 r1.0, PASID/TPH Extended Capabilities in Chapter 7/10。
- Cadence PCIe Core User Guide, §§4.5.1 message、4.5.6.6.5 message、Chapter 10.3.2 Message TLP；§4.5.2 note on AtomicOps。
- Cadence PCIe Core User Guide, PF/VF TPH and TLP Prefix Log registers in Chapter 9。
- MindShare *PCI Express System Architecture*, Appendix E, printed pp.1033–1042 / PDF pp.1089–1098，legacy locked transaction sequence；以Base Spec 5.0 §6.5校正。
