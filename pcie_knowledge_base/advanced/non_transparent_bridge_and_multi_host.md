# Non-Transparent Bridge：两个Host各管各的，又要隔着PCIe说话

## 1. 先看普通透明桥

一个普通PCIe Switch Port在配置模型里是Type 1 PCI-to-PCI bridge。Root Complex的enumeration会穿过它，继续为后面的bus分配number，并为整棵下游树分配resource。对host来说，桥是“透明”的：它后面的Endpoint仍在我这棵PCI hierarchy里。

但如果桥后面不是一群普通Endpoint，而是另一台有自己CPU、DRAM、Root Complex和OS的完整主机，透明就成了麻烦：两个Host都想给同一片世界枚举、编bus number、分配地址，大家都是甲方，没人愿意当配件。

Non-Transparent Bridge（**NTB，非透明桥**）的做法是在两个主机域之间设一道明确边界。

## 2. 为什么叫“非透明”

每一侧的Host在边界处看到一个像Endpoint的Type 0 Function，而不是一个可继续下探的Type 1 bridge。Enumeration在这里停下：

```text
Host A hierarchy                         Host B hierarchy

CPU A ─ RC A ─ ... ─ [Type 0 face A] ║ [Type 0 face B] ─ ... ─ RC B ─ CPU B
                              NTB boundary

Host A不枚举Host B的bus/devices；Host B也不枚举Host A。
```

“非透明”不是说数据过不去，而是说**topology、configuration和address ownership不穿透**。数据要过去，必须走双方约定的window和通知机制。

## 3. 四个基本积木

### 3.1 Memory Window（MW）

本地Host给NTB Function的BAR分配地址，但访问该BAR的transaction会被NTB翻译到peer的一段内存或aperture。它就像一扇开在墙上的窗：窗框在A的PCI address space，窗外的景物却在B的address domain。

```text
Host A CPU writes NTB BAR aperture
  → A-side Memory Write TLP
  → NTB window match + address translation
  → B-side target address / backing buffer
```

Window不是“自动共享整个DRAM”。双方software必须协商size、alignment、translation target、buffer lifetime和access permission。

### 3.2 Doorbell

Doorbell是对peer的事件通知bit，通常可mask并发中断。常见用法是：A先把数据写进peer window，执行必要的ordering/barrier，然后敲doorbell告诉B“数据已放好”。

Doorbell自己不搬数据，就像快递员按门铃不等于把冰箱搬进了客厅。

### 3.3 Scratchpad / Mailbox

双方可见的小寄存器用来交换protocol version、queue size、buffer address、ready/owner/generation等控制信息。它容量小，适合握手，不适合当数据面。

### 3.4 Link/status与peer lifetime

NTB软件必须把link up/down、peer reboot、host failover和window失效当成一等公民。Peer重启后，旧的translation、buffer address和queue ownership可能已经全部过期；仅凭“Link又Up了”继续往旧地址DMA，是一种非常有创造力的数据破坏方式。

## 4. 一次跨Host发送的完整例子

假设B准备了1 MiB receive buffer，NTB把A侧BAR2的`0x1000_0000–0x100f_ffff`翻译到它。

1. B通过scratchpad发布protocol generation、buffer size和ready。
2. A验证alignment/size，建立本地BAR mapping和必要的NTB translation。
3. A向`BAR2 + 0x2000`写4 KiB payload；在Link上它是一个或多个Posted Memory Write TLP。
4. A做平台/驱动要求的write ordering/flush，避免doorbell超车。
5. A设peer doorbell bit 3。
6. B的NTB hardware产生中断，B driver清doorbell，根据queue metadata消费buffer。
7. B通过反向doorbell或completion queue返还ownership。

注意第4步：Memory Write是Posted，没有Completion帮A证明“B已经看到数据”。跨Host protocol必须自己设计producer/consumer ordering，不能把门铃当作数据完成回执。

## 5. Linux NTB框架映射

本机Ubuntu 6.8 headers提供`include/linux/ntb.h`，可以确认Linux将NTB分为两层：

- hardware driver实现`struct ntb_dev_ops`；
- client driver通过`struct ntb_client_ops` probe/remove，并注册context callback接收link、doorbell和message event。

`ntb_dev_ops`的接口几乎就是上面四块积木的直译：

| Linux operation族 | 硬件含义 |
|---|---|
| `link_is_up/enable/disable` | Link/lifetime |
| `mw_count/get_align/set_trans/clear_trans` | Memory Window与translation |
| `db_read/set/clear/*mask*` | Doorbell status、trigger、mask |
| `spad_count/read/write` | Scratchpad握手 |
| `msg_count/read/peer_msg_write` | 额外message/mailbox通道 |

Kconfig还显示`NTB_TRANSPORT`可在hardware API上提供queue-pair transport，而`NTB_PINGPONG`、`NTB_TOOL`、`NTB_PERF`和`NTB_MSI_TEST`分别用来做基本doorbell/scratchpad测试、debugfs操作、raw window性能和MSI forwarding测试。

但workspace只有generated headers/Kconfig，没有完整`drivers/ntb/` C source，因此本文不伪造某个hardware driver的调用链。

## 6. 当前Cadence core能直接做NTB吗

当前`Cadence_PCIE_userdoc.pdf`没有找到Non-Transparent Bridge/NTB配置模式、peer memory window、NTB scratchpad/mailbox或NTB doorbell block。手册里的uDMA `Doorbell (Go)`只是启动本地DMA channel的控制，**不是NTB peer doorbell**。

因此现有证据只能得出：

- Cadence这份Gen4 Dual-Mode guide描述RP/EP，未暴露NTB feature；
- 如果产品要在两个SoC间实现类NTB通信，可能需要Switch/NTB IP，或在EP client logic中自己组合BAR window、translation、mailbox和MSI；
- 自己组合“看起来像NTB”的机制不代表自动兼容Linux NTB subsystem，还要实现对应hardware driver/semantics。

## 7. Debug时把问题分成两个世界

| 现象 | 先查什么 |
|---|---|
| 本地根本看不到NTB Function | 本地Link/enumeration/configuration，不要先查peer buffer |
| Function可见，MW读写UR/CTO | BAR resource、bridge window、NTB translation和peer backing memory lifetime |
| 数据到了，peer没中断 | doorbell valid/mask/vector/clear语义，不要重新调BAR size |
| doorbell先到，payload还是旧的 | Posted Write ordering、write-combining/barrier/flush和peer cache coherency |
| peer reboot后随机破坏 | generation/handshake、旧MW translation和buffer ownership是否彻底失效 |

## 8. 常见误解

- **“NTB是普通Switch的另一个名字。”** 不对。透明Switch延伸同一hierarchy；NTB终止枚举并隔离两个domain。
- **“两侧看到的BAR地址应该一样。”** 不对。两个Host独立分配resource，地址通过NTB window翻译。
- **“Doorbell等于DMA completion。”** 不对。它是软件协议通知，ordering和ownership必须另行保证。
- **“PCIe Base Spec定义了一个通用NTB configuration header。”** 不对。NTB历史上是厂商/平台机制；通用概念稳定，寄存器、window和doorbell细节依hardware。

## 9. Source Reference

- MindShare *PCI Express System Architecture*：Appendix C，printed pp.999–1017左右，约PDF pp.1055–1073；Type 0 enumeration boundary、BAR translation window、mailbox/doorbell和multi-host/failover例子。
- Local Ubuntu 6.8 generated kernel headers：`include/linux/ntb.h`，`struct ntb_dev_ops`、client/context callbacks；`drivers/ntb/Kconfig`与`drivers/ntb/test/Kconfig`。
- Cadence PCIe Core User Guide：全手册关键词审计未发现NTB/non-transparent/multi-host feature；uDMA Doorbell属于DMA channel本地控制。
- 相关文档：[Topology与分层](../fundamentals/overview_topology_and_layers.md)、[BAR与地址翻译](../addressing/bar_and_cadence_address_translation.md)、[DMA与Linux DMA API](../dma/pcie_dma_cadence_and_linux_dma_api.md)。
