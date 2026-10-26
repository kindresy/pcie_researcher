# Expansion ROM / Option ROM：设备为什么还带着一小块“开机说明书”

## 1. 它解决什么问题

有些PCIe设备在OS driver启动前就必须工作。比如，机器想从一张NVMe或RAID卡启动，firmware至少要知道如何识别它、如何读启动介质。Expansion ROM（常被叫做Option ROM）就是Function可选提供的设备本地firmware image。

它不是BAR0后面“偷藏的一段RAM”，而是一个独立、可选的Memory Space decoder，由Type 0 header offset `0x30`的Expansion ROM Base Address Register描述。Type 1 header中对应位置是`0x38`。

## 2. 直觉：先借个地址，读完就还

系统firmware或配置软件的典型动作是：

```text
发现Function
  ↓
探测ROM decoder需要多大窗口
  ↓
临时分配一段不冲突的Memory Space
  ↓
同时打开Command.MSE和ROM Enable
  ↓
CPU读取该窗口 → Memory Read TLP → Function的ROM decoder
  ↓
检查image格式/可信性，必要时复制到系统内存
  ↓
关闭ROM Enable，恢复临时改写的resource状态
```

所以ROM“在卡上”，但CPU并不是通过SPI私下找它；它在启用时是PCIe Memory Space的一个目标。

## 3. ROM BAR和普通BAR有什么不同

PCIe Base Specification 5.0的Type 0 Expansion ROM Base Address Register中：

- bit 0是`Expansion ROM Enable`，默认0；
- bits 3:1可选报告Expansion ROM Validation Status；
- bits 7:4可选报告implementation-specific Validation Details；
- bits 10:8保留；
- bits 31:11是32-bit ROM base address的高21 bit。

它和普通Memory BAR一样通过“写全1、读回硬连线0”暴露size/alignment，最大请求空间为16 MiB。但它多了一个明确的Enable bit，并且只有当下面两个条件同时满足时才能命中：

```text
Command.Memory Space Enable == 1
              AND
Expansion ROM Enable == 1
```

`MSE=0`的优先级更高。只开ROM Enable而忘了MSE，读到的常常是一声沉默——然后调试者开始怀疑人生。

### 3.1 一个128 KiB sizing例子

旧教材给出了一个很直观的例子：软件在保持bit 0为0的前提下对base field写全1，读回`0xfffe0000`。最低可写地址位对应`0x20000`，因此decoder要求128 KiB，base也必须128 KiB对齐。

工程上不要在Function仍在decode Memory Space时随手对BAR做sizing。Base Spec要求先禁用相应address decode，保存原值，sizing后恢复，最后再重新开decode。

## 4. 一次ROM读最终变成什么

假设host把128 KiB ROM临时放在PCIe address `0x9000_0000`，然后CPU读取image的`0x40`偏移：

```text
firmware/OS read
  ↓ CPU virtual/physical mapping
host MMIO address
  ↓ Root Complex outbound translation
Memory Read Request, address = 0x9000_0040
  ↓ PCIe hierarchy
Function ROM decoder hit
  ↓ local ROM/flash read path
Completion with Data
  ↓ match Requester ID + Tag
Root Complex returns bytes to CPU
```

这是Non-Posted Memory Read，所以需要Completion with Data。ROM BAR本身不会导致一种新TLP type；它只是让Function的address decoder把这笔Memory Request导向ROM数据源。

## 5. PCIe 5.0比2003教材多了一道安全问题

2003年教材的主线是“找到ROM、检查`0xAA55`和`PCIR`、选择匹配Vendor/Device/Class/Code Type的image、shadow、执行初始化代码”。它还详细描述了旧PC把image复制到`0xC0000–0xDFFFF`、调用x86初始化入口和hook实模式中断链的做法。后者是很有价值的历史背景，却不是现代UEFI/PCIe 5.0平台的通用实现要求。PCIe Base Spec 5.0对ROM内容格式直接引用独立的PCI Firmware Specification，不应用2003教材的旧BIOS流程替代它。

现代PCIe还定义了可选的hardware validation状态：未支持、验证中、有效、有效且可信、无效、有效但不可信，以及带warning的结果。如果Fundamental Reset解除1分钟后仍显示“验证中”，software可假定它不会完成。

这不等于“PCIe帮你安全执行了ROM”。完整的trust policy、签名格式和是否允许执行仍是platform firmware/security policy的事。Validation Details也明确是implementation-specific。

PCIe 5.0还允许用Enhanced Allocation（EA）Capability中`BEI=8`的entry来表示Expansion ROM。同一Function不能同时用EA entry和传统ROM BAR机制。

## 6. 当前Cadence core的结论

当前`Cadence_PCIE_userdoc.pdf`的PF configuration header对offset `0x30`明确写着`Expansion ROM Base Address Register: Not Implemented`，32 bit均为只读0。手册的另一组Function header描述也给出相同结论。

因此，对这份core配置：

- host对offset `0x30`做ROM sizing不会得到可分配的ROM resource；
- 不能因为Cadence手册的某些request type编码中出现`Expansion ROM Access`，就推断PF header已经实现ROM BAR；
- 若某SoC产品需要Option ROM，必须查该SoC的integration/RTL feature selection和额外firmware path，不能只凭通用PCIe行为补齐。

这也是“Spec允许”和“当前硬件实现”最典型的差别之一。

## 7. Linux驱动工程师需要关心吗

普通Function driver的`probe()`不应把Option ROM当成常规MMIO BAR来映射和运行。ROM的resource discovery、必要时的临时decode和firmware policy属于PCI core/platform firmware边界；它也不会被`pci_iomap(pdev, 0, ...)`当成BAR0返回。

Linux通常会把ROM作为单独的PCI resource建模，sysfs也可在policy允许时提供ROM访问入口。但当前输入没有Linux source tree，因此这里不伪造某个kernel version的`pci_enable_rom()`内部调用链。未来拿到目标kernel后，应继续追踪：ROM resource如何从config header创建、enable/disable如何保护MSE和共享decoder，以及sysfs read如何与architecture host bridge交互。

## 8. Debug清单

读不到Option ROM时，按这个顺序查：

1. Function真的实现了ROM BAR吗？写全1前先保存状态并禁用decode。
2. 分配的base是否按size对齐，且不与其他resource冲突？
3. `Command.MSE`和`Expansion ROM Enable`是否同时为1？
4. bridge Memory window和RC outbound window是否覆盖该地址？
5. 是没有Completion、返回UR，还是返回了内容但image signature/validation失败？这三种故障不在同一层。
6. 如果decoder与普通BAR共享，读ROM时是否还有driver在访问那个BAR？Base Spec明确要求device-independent software此时不要通过共享decoder的其他BAR访问Function。

## 9. 常见误解

- **“Option ROM就是Endpoint的firmware本体。”** 不一定。它是host可见的Expansion ROM image，可以和设备内部执行firmware分开。
- **“ROM是只读的，所以ROM BAR也全是只读。”** 内容可能只读，但base address和Enable是系统配置机制的一部分。
- **“看到offset 0x30就肯定有ROM。”** 不对。每个Type 0 header都有这个architectural位置，实现可以全为0；当前Cadence就是这样。
- **“验证通过就一定可执行。”** 不对。Validation Status提供硬件检查信息，最终执行policy仍属于platform。

## 10. Source Reference

- PCI Express Base Specification 5.0 r1.0：§7.5.1.2.4，Type 0 Expansion ROM Base Address Register；§7.5.1.3.12，Type 1对应定义；§7.8.5，Enhanced Allocation；§9.3.4.1.15，SR-IOV PF/VF变化。
- Cadence PCIe Core User Guide：§9.2.1.1.13，p.452；§9.8.1.1.13，p.922，均标记Not Implemented。
- MindShare *PCI Express System Architecture*：Ch.22 printed pp.783–785，ROM BAR和128 KiB sizing例子；Ch.23 printed pp.871–892左右，ROM detection、multiple image、shadowing与旧BIOS/Open Firmware流程。
- 相关文档：[BAR与Cadence地址转换](../addressing/bar_and_cadence_address_translation.md)、[Configuration Space、ECAM与路由](configuration_space_ecam_and_routing.md)。
