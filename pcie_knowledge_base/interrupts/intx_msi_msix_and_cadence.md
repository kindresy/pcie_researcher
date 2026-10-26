# PCIe 中断：INTx 是“虚拟电线”，MSI 是一笔有特殊意义的内存写

## 1. 先把脑中的“中断脚”拔掉一会儿

做 UART、I2C 控制器驱动时，中断通常很好想象：外设把一根 IRQ 线拉高，中断控制器看到电平或边沿，CPU 跳进 handler。

PCIe 的设备可能隔着多级 Switch，而且一条 Link 只有高速差分 Lane，并没有给每个 Endpoint 单独铺一根中断线。因此 PCIe 把中断也装进包里，在链路内传输：

- **INTx** 用 Assert/Deassert Message TLP 传递四条传统中断线的逻辑状态；
- **MSI/MSI-X** 让设备向系统指定地址发一笔 Memory Write，请求中断服务。

MSI 的全称虽叫 *Message Signaled Interrupt*，但线上不是 PCIe `Msg` 类型 TLP，而是普通 `MWr`。名字在这里颇有迷惑性，像一家名叫“面馆”的米粉店。

## 2. 三种机制一眼比较

| 机制 | PCIe 链路上是什么 | 共享/向量 | Mask 状态 | 驱动工程上的含义 |
|---|---|---|---|---|
| INTx | `Assert_INTx` / `Deassert_INTx` Message TLP | 4 条虚拟线，可共享 | Command.Interrupt Disable | handler 要确认是否自己的中断；必须清源并 Deassert |
| MSI | 1 DW Memory Write | 每 Function 最多 32 个、按 2 的幂分配 | 可选 per-vector mask | 地址基本共享，设备用低位 data 区分 vector |
| MSI-X | 1 DW Memory Write | 每 Function 最多 2048 个 | 每 entry mask + Function Mask | 每个 vector 有独立 address/data，表放在 BAR 空间 |

Base Spec 5.0 §7.7.1 明确规定：能产生中断的 PCIe Function 必须实现 MSI 或 MSI-X，或两者都实现。INTx 是兼容机制，不是现代设备的首选。

## 3. INTx：没有物理线，仍然保留“电平状态”

### 3.1 为什么需要 Assert 和 Deassert 两种包

INTx 是 level-like 语义。设备中断条件出现时发 `Assert_INTA`，条件清除时发 `Deassert_INTA`；链路两端各自记住当前状态。这不是“每来一个包就计一次中断”的 edge counter。

Base Spec 定义八种无 payload 的 Message TLP：A/B/C/D 各有 Assert 与 Deassert。它们使用 local routing，沿每条 Link 逐跳维护虚拟线状态。Switch 会为每个 Downstream Port 分别跟踪四条线，再在 Upstream Port 合并；Bridge 还会根据下游 Device Number 做经典的 INTx swizzle。

因此：

```text
EP interrupt condition = 1
 → EP Upstream Port 发 Assert_INTA Message
 → Switch 记录该 Downstream Port 的 INTA=active，并向上游折叠/转发
 → Root Complex 记录虚拟线状态
 → 平台映射为本地 interrupt-controller resource
 → CPU IRQ handler

driver 清除 EP 中断源
 → EP condition = 0
 → Deassert_INTA Message
 → 各级撤销逻辑状态
```

Requester ID 在某条 Link 上标识的是发送该 Message 的 Upstream Port，不一定就是最初产生中断的 Function；这也是固件/OS 必须理解整棵 topology 和 swizzle 的原因。

来源：Base Spec 5.0 §2.2.8.1，printed/PDF pp.135–139。

### 3.2 Command.Interrupt Disable

Function 的 PCI Command Register 中 `Interrupt Disable` 置 1 后，INTx 被禁止；若虚拟线当时 active，还必须发送相应 Deassert。它不负责屏蔽 MSI/MSI-X。

Cadence EP 把 Function 的 `INTx_disable` 状态从 `FUNCTION_STATUS` sideband 给 client logic。client 在发 legacy interrupt 前必须检查它；还要通过 `INT_PENDING_STATUS` 提供 Configuration Status Register 中的 Interrupt Status。

### 3.3 Cadence 的 legacy 握手

Cadence 给 client logic 四个 `INTA_IN`～`INTD_IN` 输入。输入上升/下降分别触发 Assert/Deassert Message。每发完一次状态变化，core 用一拍 `INT_ACK` 告诉 client；同一 pin 再变化前必须等到 ACK，避免前一次消息尚未提交就覆盖状态。

这一个 `INT_ACK` 是 **Cadence 本地接口握手**，不要和 Data Link ACK DLLP、CPU 的 interrupt acknowledge 混为一谈。

来源：Cadence Core User Guide §§4.6.1、4.6.1.1，printed pp.272–274 / PDF pp.361–363。

## 4. MSI：为什么写内存就能让 CPU 收到 IRQ

### 4.1 配置阶段

每个支持 MSI 的 Function 在传统 Capability List 中提供 MSI Capability（ID `05h`）。系统软件枚举它、分配 interrupt vector，然后写入：

- Message Address（PCIe Endpoint 必须支持 64-bit address layout）；
- Message Data；
- Multiple Message Enable；
- 可选的 Mask Bits；
- 最后置 MSI Enable。

设备不能自己猜地址/data，也不应把它们写死。它们由平台决定，可能编码本地 APIC 目标、interrupt-remapping 索引或其他实现信息。

Base Spec 的措辞还区分 **system configuration software** 与 device driver 对 MSI control bits 的权限。Linux 驱动应使用 PCI/MSI core API，让 core 完成 vector 分配和 capability 编程，不要自己拿 `pci_write_config_word()` 生拧 MSI Enable。

### 4.2 触发阶段

设备要触发 MSI 时，把配置好的 Message Data 写到 Message Address：

```text
EP interrupt source
 → 选择 MSI vector
 → 读取 MSI Message Address/Data
 → 生成 Length=1 DW、First BE=1111b 的 Posted Memory Write
 → PCIe Link / Switch
 → RC 接收这笔特殊地址的写
 → host interrupt controller / remapper 解码 address+data
 → CPU vector
 → Linux generic IRQ layer
 → driver handler
```

MSI write 的协议性质仍是 Posted MWr，所以正常情况下没有 Completion。不能因为“中断很重要”就期待远端返回 CplD。它依赖链路可靠传输、平台中断架构及驱动的设备级状态处理。

### 4.3 多 vector 怎样编码

MSI 的 Multiple Message Capable 表示设备最多请求 `1/2/4/8/16/32` 个 vector；Multiple Message Enable 表示系统实际给了多少。设备只能修改 Message Data 的相应低位来选择已分配 vector。例如 MME=`010b` 表示 4 vectors，设备可改变 data bit `[1:0]`。

这和 MSI-X 的“一项一个完整 address/data”不同。

来源：Base Spec 5.0 §§7.7.1–7.7.1.5，printed/PDF pp.773–779。

### 4.4 MSI到了，DMA数据就一定已经可见吗

先给稳定结论：MSI/MSI-X MWr的Relaxed Ordering和No Snoop必须为0。对于在**同一TC**中、由同一Function早于MSI发出的Posted Write，ordering规则禁止MSI超车；system必须保证handler能看见那些先到的update。驱动不需要为这个基本同步性再额外读一次设备寄存器。

但MSI/MSI-X不被强制使用TC0。如果Function的DMA data用TC1，MSI用TC0，不同TC之间没有默认相对顺序保证：

```text
EP发送顺序： DMA MWr(TC1) → MSI MWr(TC0)
RC观测顺序： MSI可能先到 → handler读到旧数据
```

规范要求使用多TC的device自己明确维持data/interrupt synchronization。一种方法是在发MSI前，针对每个额外data TC对同一target发zero-length Memory Read，等CplD回来后再发MSI；这样可flush该TC的早先Posted Writes。更常见的简化是让相关data和MSI使用同一TC。

Zero-length Read不是`Length=0`的非法TLP；它是`Length=1 DW`且First DW BE=`0000b`的Memory Read，Completer仍回1 DW data（内容未规定）。flush只覆盖与该Read相同TC且相同target的早先Posted Writes。

这个角落案例来自王齐教材的Capric卡实例，并已与Base Spec 5.0 §§2.2.5、6.1.4交叉验证。它也解释了为什么“中断来了但数据偶尔还旧”可能是TC/ordering设计问题，不只是cache bug。

## 5. MSI-X：把每个向量的投递信息放进表里

### 5.1 三块结构不要混

MSI-X 涉及三处数据：

1. **MSI-X Capability** 在 Configuration Space，ID `11h`，包含 Enable、Function Mask、Table/PBA 的 BIR 与 offset；
2. **MSI-X Table** 位于某个 BAR 指向的 Memory Space；每 entry 16 B：lower address、upper address、data、vector control；
3. **PBA**（Pending Bit Array）也位于 BAR Memory Space，记录被 mask 时挂起的 vector。

`BIR` 指示使用哪个 BAR，真实地址为：

```text
Table host address = BAR[BIR] assigned base + Table Offset
entry[k]            = Table base + k × 16
```

Table Size 字段编码 `N-1`，所以读到 3 代表 4 entries，不是 3。每 entry 的 Mask Bit 复位值为 1；软件通常保持 mask，写完 address/data 后再解除，避免“桌子还没摆好，服务员先把菜泼出去了”。

### 5.2 谁访问这张表

Host 通过 BAR MMIO 写 MSI-X Table。对 EP 来说，这些是普通 inbound Memory Write TLP，经 BAR hit 和 inbound address translation 落到 client memory。

触发 vector `k` 时，EP client 读取该 entry：

- 若 entry 或 Function 被 mask，置 PBA bit `k`；
- 否则按 entry 的完整 address/data 发 1 DW MWr；
- mask 清除后，如果对应 pending 仍有效，应重新投递并清理 pending 状态。

MSI-X data 是 32 bit，并且设备不能像 MSI 那样自行修改低位。

来源：Base Spec 5.0 §§7.7.2–7.7.2.9，printed/PDF pp.781–789；Cadence §§4.5.6.6.7、4.6.1，printed pp.269–270、273–274 / PDF pp.358–359、362–363。

## 6. Cadence 实现：标准 Capability 只是“菜单”，client logic 还得下厨

对这份 Cadence Dual-Mode core，EP 的 MSI/MSI-X generation 不是简单拉一根 `msi_req` 就全部完成。手册给出的路径是：

```text
Host config write
 → Cadence Function MSI/MSI-X Capability registers
 → MSI_ENABLE / MSI_MASK / MSIX_ENABLE sideband 可见

EP client interrupt
 → client 读 capability 或 MSI-X table 的 address/data/mask
 → 选择/编程 AXI outbound region
    AXI Region Base
    AXI-to-PCIe Address Translation = Message Address
    PCIe Descriptor = Memory Write
 → client 对该 AXI region 发 write
 → Cadence core 转换为 PCIe MWr TLP
```

MSI data 是 `MSI Data Register + vector number`；MSI-X 使用 entry 中的完整 vector data。

Cadence 还提供 MSI mask-change、mask set/clear、MSI-X Function Mask change等 sideband events，方便 client 发现“某个 pending vector 现在可以补发”。MSI Pending Status 有两种实现模式：APB 读写内部状态，或通过 `MSI_PENDING_STATUS_IN` 直接提供，模式由 Local Management `Debug Mux Control 2[9]` 选择。

在 RP 模式，core 把收到的 MSI/MSI-X 当普通 inbound Memory Write，经 AXI Master interface 交给本地系统。手册说软件负责监视 MSI 区域并 service interrupt；具体 SoC 往往还会接 MSI controller/interrupt remapper，但当前资料没有 SoC TRM，因此其地址命中后怎样进入 CPU interrupt controller 标为 **Unknown / NEED_SOC_TRM**。

来源：Cadence §§4.5.6.6.6–.7、4.6.1–.2，printed pp.268–276 / PDF pp.357–365。

## 7. Linux 驱动应该怎样看它

当前目录未包含 Linux kernel tree，所以下列是稳定 API/架构关系，不冒充某个版本的精确调用链：

```c
int nvec;

nvec = pci_alloc_irq_vectors(pdev, 1, wanted,
			     PCI_IRQ_MSIX | PCI_IRQ_MSI);
if (nvec < 0)
	return nvec;

ret = request_irq(pci_irq_vector(pdev, 0), handler, 0,
		  "device-name", priv);
```

概念上，PCI core/MSI subsystem 会：

1. 检查 MSI/MSI-X capability 与平台 irq-domain 支持；
2. 分配 Linux IRQ 和底层 interrupt vector；
3. 得到平台指定的 message address/data；
4. 对 MSI 写 capability registers，或对 MSI-X 映射 BAR table 后逐 entry 编程；
5. 处理 mask、Enable 以及 INTx/MSI/MSI-X 互斥；
6. `pci_irq_vector()` 返回给驱动可传给 `request_irq()` 的 Linux IRQ number。

`request_irq()` 本身不会教 EP 怎样生成 MSI。驱动通常还要写设备 BAR 寄存器，启用具体业务中断源；设备硬件/client firmware 才会在事件发生时发出 MSI write。

精确函数调用链、irq_domain 层级、x86/ARM64 MSI domain 差异，需要补充目标 kernel commit 与 SoC interrupt-controller driver 后再标 Confirmed。

## 8. 一次真实 MSI-X 的完整例子

假设：

```text
EP                         01:00.0
BAR2 host-assigned base    0x9000_0000
MSI-X Table BIR            2
Table Offset               0x2000
vector                     5
entry[5] host address      0x9000_2050
host programmed address    0x0000_0000_FEE0_1000  （仅作说明）
host programmed data       0x0000_0045
Mask                       0
```

枚举/启用时 Host 对 `0x9000_2050` 附近的写，在线上首先是 **RC→EP 的 BAR Memory Write**，用于建立 table entry。

业务事件发生时方向反过来：

```text
EP client 读 entry[5]
 → Cadence outbound AXI write
 → address region 翻译成 0xFEE0_1000
 → EP→RC MWr, Length=1 DW, BE=1111, payload=0x45
 → RC/interrupt remapper 消费
 → Linux IRQ handler
```

同一个 MSI-X 功能因此至少涉及两类方向相反的 Memory Write。只说“MSI-X 表在 BAR 里”而不分方向，很容易把自己绕进 Link Training 的 Recovery 状态——人脑版的。

## 9. 常见误解

- **“MSI 是 PCIe Message TLP。”** 错。MSI/MSI-X 使用 Memory Write Request；INTx Assert/Deassert 才是 Message TLP。
- **“MSI 写到普通 DRAM，CPU 轮询那块内存。”** 不准确。address/data 由平台中断架构解释，通常命中中断控制/重映射机制；Cadence RP 侧是否再由 SoC 专用 MSI block 截获，需要 SoC TRM。
- **“MSI 有 Completion，所以比 INTx 可靠。”** 错。MSI 是 Posted MWr，没有普通 Completion。
- **“MSI-X Table 在配置空间。”** 错。Capability 在配置空间，Table/PBA 在 BAR Memory Space。
- **“开了 MSI 后 INTx 还能一起报。”** Spec 禁止。MSI 或 MSI-X enable 后，Function 不得用 INTx 请求服务。
- **“Linux IRQ number 就是 MSI data。”** 错。Linux IRQ 是软件抽象；底层 vector/message data 由 irq domain/平台映射。

## 10. Debug：中断不来时从哪头剥洋葱

### Host 配置

1. `lspci -vv -s 01:00.0`：MSI/MSI-X capability 是否存在、Enabled、vector 数、Table/PBA BIR/offset 是否合理。
2. Command.Intx Disable 与 MSI/MSI-X Enable 是否符合预期。
3. MSI-X entry 是否仍 masked；Table BAR 是否分配并能访问。

### EP/client

1. 业务 interrupt status 是否真的置位、enable/mask 是否正确。
2. `MSI_ENABLE/MSIX_ENABLE` sideband 是否看到 Host 配置结果。
3. client 是否使用枚举后最新的 address/data，尤其 reset/link-down 后是否失效。
4. Cadence outbound region 是否命中，BME 是否置位、Function 是否 D0 且不在 FLR。
5. masked 时 PBA/pending status 是否置位，解除 mask 后是否补发。

### 链路与 Host

1. analyzer 是否看到 1 DW MWr，address/data/Requester ID 是否正确。
2. RC inbound AXI 是否收到写；SoC MSI controller/remapper 是否命中。
3. Linux `/proc/interrupts` 计数是否增加；若增加而 handler 没完成，查 handler 注册与共享数据。
4. handler 进入但中断风暴，查设备源是否清除、写回是否 flush、INTx 是否 Deassert。

## 11. Evidence 与边界

- **Confirmed**：INTx 线上格式、状态跟踪、swizzle、Command.Interrupt Disable；Base Spec §2.2.8.1。
- **Confirmed**：MSI/MSI-X 是 1 DW Memory Write；capability/table/PBA 字段和 mask 规则；Base Spec §§7.7.1–7.7.2。
- **Confirmed**：Cadence EP client 经 AXI outbound region 生成 MSI/MSI-X，RP 经 AXI Master 接收；Cadence §§4.5.6.6.6–.7、4.6。
- **Inferred**：一个具体 SoC 会在 Cadence AXI Master 之后接专用 MSI controller 或 interrupt remapper。这是常见集成，但不能从当前 core-only databook 确认。
- **Unknown**：目标 Linux 版本的精确 PCI/MSI/irq-domain 调用链，以及目标 SoC 的 MSI address decode。需要 kernel source、SoC TRM 和 interrupt-controller 文档。

## 12. Source Reference

- PCI Express Base Specification Revision 5.0 Version 1.0, §2.2.8.1, printed/PDF pp.135–139。
- PCI Express Base Specification Revision 5.0 Version 1.0, §§7.7.1–7.7.1.6, printed/PDF pp.773–781。
- PCI Express Base Specification Revision 5.0 Version 1.0, §§7.7.2–7.7.2.9, printed/PDF pp.781–789。
- Cadence PCIe Core User Guide, §§4.5.6.6.6–4.5.6.6.7, printed pp.268–270 / PDF pp.357–359。
- Cadence PCIe Core User Guide, §§4.6–4.6.2, printed pp.272–276 / PDF pp.361–365。
- 王齐《PCI Express 体系结构导读》第 15 章，printed pp.417–440 / PDF pp.429–452：Linux 2.6.31-era INTx routing、MSI message compose 与 MSI-X driver 示例；仅作历史实现对照，现代驱动使用 `pci_alloc_irq_vectors()`。
- 王齐第11章printed p.296 / PDF p.308：Capric卡的多TC data/MSI顺序与zero-length Read flush例子；以Base Spec 5.0 §§2.2.5、6.1.4校正。
