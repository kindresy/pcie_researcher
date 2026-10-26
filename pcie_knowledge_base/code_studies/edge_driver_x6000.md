# edge-driver/x6000：Host 侧 X6000 PCIe 功能驱动源码导读

> 研究对象：`source_repos/edge-driver`，分支 `x6000`，提交
> `f27ec1192d2089bf80898677a20fb5ac714f6f8f`。
>
> 证据标签：**Confirmed** 表示源码直接支持；**Inferred** 表示需要与芯片侧
> Linux/固件继续交叉验证；**Risk** 是代码审计发现，不代表一定已经在产品中触发。

## 1. 先把角色叫对：它不是芯片内部的 EPC 驱动

这份仓库里的 `pcie/edge.c` 是运行在 **Host Linux** 上的 PCI function
driver。它匹配 `17cd:0100` 和 `17cd:2000`，Linux 枚举出 X6000 Endpoint 后，
PCI core 才会调用它的 `edge_probe()`。证据见 `edge.c:6470-6489`。

这里很容易绕晕，因为工程口语常把它叫“EP 驱动”：它确实在驱动一块 EP 卡，
但它本身并不实现 Endpoint Controller（EPC）。真正运行在 X6000 芯片内部、负责
配置 Endpoint Controller、BAR 和 inbound/outbound translation 的代码，在配套
Linux 仓库的 `drivers/pci/controller/pcie-edge*.c` 与
`drivers/pci/endpoint/functions/pci-epf-edge.c` 中。后续专题会把两端接起来。

```text
Host Linux                                      X6000
┌──────────────────────┐ PCIe Link ┌──────────────────────────┐
│ edge-driver/pcie     │◄──────────►│ EPC + EPF + firmware    │
│ pci_driver + /dev/*  │            │ BAR memory + uDMA       │
└──────────────────────┘            └──────────────────────────┘
          RC 一侧                              EP 一侧
```

## 2. 一句话架构

`edge.ko` 同时扮演四个角色：

1. 标准 PCI driver：启用设备、申请 BAR、映射 MMIO、配置 MSI/MSI-X、接入 AER；
2. 字符设备：把 boot mailbox、BAR MMIO、DMA、P2P、热复位等能力暴露给用户态；
3. X6000 uDMA 的“远程控制台”：Host 通过 BAR0 直接填写 EP 内部 DMA 描述符与寄存器；
4. 生命周期协调者：依据 BAR 中的 boot magic 判断 BootROM/PCS/BL2/U-Boot/Linux 阶段。

它不是一个薄薄的 `pci_iomap()` 包装层，而是把启动、数据搬运、中断、异常和拓扑
策略全塞在了一个 6658 行文件里。读它时按数据流拆解，比从第一行一路硬啃更有效。

## 3. 主要对象和所有权

### 3.1 `struct edge_dev`

每个枚举出的 PCI function 对应一个 `edge_dev`，关键成员见 `edge.h:470-516`：

| 成员 | 含义 | 谁创建/销毁 |
|---|---|---|
| `pdev` | Linux `struct pci_dev` | PCI core / `edge_probe()`引用 |
| `bar[6]` | BAR 的 Host 内核虚拟地址 | `edge_map_bars()` / `edge_unmap_bars()` |
| `entry[49]` | 1 个 DMA + 48 个通知 MSI-X vector | IRQ setup/teardown |
| `uengine[4]` | 四条 Cadence uDMA 软件队列 | probe/remove |
| `notify_irq_chan[2][24]` | 两组、每组 24 条通知通道 | exported notify API |
| `cdev` | 用户态 ioctl/mmap 入口 | probe/remove |
| `flr_brw` | 延迟到 remove 执行的 FLR 写请求 | ioctl/remove |
| `dma_pool` | 可选 coherent DMA block pool | module param 控制 |

### 3.2 `struct udma_engine` 与 `struct udma_xfer`

一个 engine 对应一个硬件 DMA channel。它维护描述符环的 `head/tail`、传输队列、
`running` 状态、spinlock 和 bottom-half work。一次 `udma_xfer` 保存 Host DMA 地址、
EP 本地地址、方向、SG mapping、描述符范围、waitqueue 和完成状态。

描述符有 11 个 dword：system/local address、external/PCIe address、attributes、
24-bit length/control、status 和 next pointer。代码会在 `transfer_set_desc()` 中把：

```text
sys_addr = X6000 本地 AXI 地址
ext_addr = Host DMA address / IOVA
size     = 最多 0xFFFFFF bytes
last     = INT_EN；非末尾 = CONTINUE_LL
```

写进描述符，见 `edge.c:2580-2607`。这正是理解整条 DMA 路径的钥匙。

## 4. probe：Linux 发现设备以后发生什么

`edge_probe()` 的真实顺序见 `edge.c:5847-5966`：

```text
pci_enable_device()
  → pci_set_master()
  → 可选 dma_pool
  → pci_request_regions()
  → pci_iomap()/pci_iomap_wc() 所有有效 BAR
  → 设置 64-bit DMA mask
  → 开启 AER reporting
  → 创建 4 个 uDMA engine
  → 创建字符设备与 dma_stats sysfs
  → 安装异常事件 INTx
  → 启动 boot-stage timer/work
  → 标记 online
  → 调整上游链路/ACS/Relaxed Ordering 策略
```

### 4.1 这些 PCI API 最终改变了什么

- `pci_enable_device()` 让 PCI core 恢复电源并启用可用资源解码；它不是“让 Link
  训练一次”。具体 5.10.4 调用链将在 Linux 专题追踪。
- `pci_set_master()` 设置 Command Register 的 Bus Master Enable，使该 Function
  能发起访问 Host memory 的 Memory Read/Write Request。这一步对 EP uDMA 必需。
- `pci_request_regions()` 是资源所有权声明，避免另一个驱动同时占用相同 BAR；它不
  产生 PCIe TLP。
- `pci_iomap()` 返回 Host CPU 可访问的内核虚拟 MMIO 地址。之后一次 `writel()`
  才会经 RC outbound window 变成 Memory Write TLP。
- BAR2/BAR4 使用 `pci_iomap_wc()`，适合批量写内存窗口；其余 BAR 使用普通映射，
  见 `edge.c:5670-5697`。

### 4.2 probe 的几个实现约束

- **Confirmed**：驱动要求拓扑至少深到 `pdev->bus->parent->parent->parent`，并直接把
  它交给 `pcie_bus_configure_settings()`（`edge.c:5955`）。这不是通用 PCI driver
  可以假定的拓扑。
- **Confirmed**：X6000 会沿上游清 ACS Request/Completion Redirect、Egress Control，
  并关闭 Relaxed Ordering；另有接口关闭 switch 的 ACS Source Validation，见
  `edge.c:1729-1831`。这有利于特定 P2P 路径，但会削弱标准 PCIe 隔离边界。
- **Risk**：`dma_set_mask(..., 64-bit)` 的返回值未检查（`edge.c:5901`）；普通路径也
  未显式调用 `dma_set_coherent_mask()`。若平台不能提供 64-bit streaming DMA，
  后续描述符中的地址可能不可达。

## 5. boot magic：同一个 BAR，跨越五个软件世界

驱动在 BAR 偏移 `0x7fa40` 读取启动状态：

| 阶段 | magic |
|---|---:|
| BootROM | `0xde10de10` |
| PCS | `0xde11de11` |
| BL2/ATF | `0xde12de12` |
| U-Boot | `0xde13de13` |
| Linux | `0xde14de14` |

定义见 `edge.h` 的 boot-state 常量。Host driver 因此可以在 Linux 尚未起来时，通过
同一 PCIe function 给 BootROM/PCS/BL2 发送镜像或控制消息。换句话说：PCIe Link
和 BAR 可访问，不等于 Endpoint OS 已经启动。

`edge_event_timer_cb()` 每 500 ms 轮询 magic，最多 600 次；看到 Linux magic 后，
work 才正式配置数据 IRQ、把 MRRS 调到 1024 B，并发送 `EDGE_DEVICE_ON` notifier，
见 `edge.c:5756-5841`。这是“枚举成功”和“业务可用”之间的第二道状态机。

## 6. boot mailbox：最朴素、也最可靠的搬运方式

普通 mailbox 使用：

```text
H2D ready  0x7fa00       D2H ready  0x7fa80
H2D buffer 0x7fb00       D2H buffer 0x7fd80
```

Host 写数据时等待 H2D ready 为 0，把 payload `memcpy_toio()` 到 BAR buffer，执行
写屏障，再把 ready 写为 1；Host 读数据则等待 D2H ready 为 1，读出 buffer，再清
零 ready。这里的 ready 是共享内存握手，不是 PCIe DLLP credit。

从 TLP 视角看，Host 的 payload 和 ready 都是 Posted Memory Write；读 ready 和
payload 是 Non-Posted Memory Read，需要 EP 返回 Completion with Data。软件用
ready 的所有权翻转弥补了 Posted Write 没有 Completion 这一事实。

“pro” mailbox 把大缓冲区放到 `0x760000/0x770000`。具体由哪个 boot stage 初始化
这些地址，要在 BootROM、PCS、ATF 三份源码里反向核对。

## 7. MMIO 传输：它和 DMA 不是一回事

`mmio_xfer()` 先读取 EP 的 inbound translation 寄存器，算出目标落在 BAR2 还是
BAR4 的哪个 offset，然后执行 `copy_to_user()`/`copy_from_user()` 与 `__iomem`
之间的拷贝（`edge.c:3445-3550`）。完整硬件路径是：

```text
用户态 buffer
  → Host CPU load/store
  → Host 内核 BAR 虚拟地址
  → Host physical MMIO resource
  → Root Port outbound translation
  → PCIe Memory Read/Write TLP
  → X6000 BAR hit
  → EP inbound translation
  → X6000 AXI/DDR/寄存器
```

这里 CPU 在搬数据，所以它是 MMIO copy。名字里即使带 `xfer`，也不能叫 PCIe DMA。

扩展地址 ioctl 会临时改写 inbound aperture region 5，访问目标后恢复默认窗口；
驱动只用 `edev->lock` 串行化本设备内的这类修改。它相当于“把一扇小窗临时挪到
另一堵墙上”，窗口移动期间其他访问者若绕过同一把锁，可能看到错误地址。

## 8. uDMA：一行 ioctl 如何变成 PCIe transaction

### 8.1 H2D 与 D2H 的方向

源码中 `dir == 1` 表示 outbound，也就是 X6000 → Host（D2H）；`dir == 0` 表示
Host → X6000（H2D）。因此 DMA API 方向分别为：

| 业务方向 | DMA API direction | EP 发出的 PCIe TLP |
|---|---|---|
| H2D | `DMA_TO_DEVICE` | EP Memory Read Host memory，收到 CplD 后写本地 AXI |
| D2H | `DMA_FROM_DEVICE` | EP 读本地 AXI，向 Host 发 Posted Memory Write |

“H2D 用 Memory Read”初看有点反直觉，原因是方向是按数据说的，而 Request 是由
EP 发起的：EP 想把 Host 数据拉进来，自然要读 Host memory。

### 8.2 用户 buffer 到描述符

SG 路径依次执行：

```text
get_user_pages_fast()
  → 为每页建立 scatterlist
  → dma_map_sg(&pdev->dev, ...)
  → 得到 EP 可使用的 DMA address/IOVA
  → 合并相邻 DMA segment
  → 建 Cadence linked-list descriptor
  → 把 sys_addr/ext_addr 写进 BAR0 内描述符区
  → wmb()
  → 写 channel control/KICK
```

关键代码为 `edge.c:1906-2001`、`2372-2442`、`2520-2577`、
`2580-2673`。`dma_map_sg()` 返回的不是“保证等于 Host 物理地址”的数；开启 IOMMU
时它通常是 IOVA。对 EP 来说它就是 PCIe 请求中的目标地址。

### 8.3 描述符为什么位于 BAR，而 bus address 又是 `0x60000000`

Host CPU 通过 BAR0 虚拟地址写描述符；X6000 uDMA 从芯片本地 AXI 地址读取同一块
存储。两个观察者面对的是两个地址空间，因此“Host 写描述符的地址”和“DMA next
pointer 使用的地址”不同完全正常。后续将用 EPC inbound window 初始化代码确认
这一块存储的确切映射。

### 8.4 完成与中断

vector 0 用于 DMA。ISR 读 uDMA common interrupt status：bit 0..3 是 channel done，
bit 8..11 是 channel error，W1C 后调度对应 engine work，见 `edge.c:4732-4800`。
bottom half 检查描述符 status、推进 tail、唤醒同步 ioctl，并启动队列中的下一笔。

boot uDMA 不能假定正常 MSI 已可用，因此 `transfer_submit()` 对 `is_boot` 传输先
poll channel 状态，再进入共同完成逻辑（`edge.c:2854-2887`）。

## 9. 通知中断：共享内存是信，MSI 是门铃

每组有 24 个物理 channel，每个 channel 的共享区能保存 16 条 message：

```c
struct pcie_notify_irq_message {
        u8  die_id;
        u8  chip_id;
        u64 event_id;
};
```

H2D queue 从 BAR `0x40100 + chan*0x800` 开始；D2H queue 接在后面的 24 个 slot。
Host 发送时写共享内存、`wmb()`，再向 BAR offset `0x40000` 写 channel id。后者是
门铃写。EP 收到后处理共享内存；反方向则由 EP 写 D2H queue 并发 MSI/MSI-X。

49 个 vector 的分工是：

```text
vector 0      DMA
vector 1..24  group 0 notify channel 0..23
vector 25..48 group 1 notify channel 0..23
```

Host ISR 根据 IRQ 反推 group/channel，排空 D2H 环并调用 blocking notifier chain，
见 `edge.c:892-988`。这里把 payload 放共享内存而不是 MSI message data，是因为 MSI
本质上只是一笔很小的 Memory Write 门铃，承载复杂事件既不方便也不可扩展。

队列满时 `mode != 0` 返回 `-EBUSY`；`mode == 0` 则覆盖旧消息。源码已经用中文注释
明确提醒 bitmap 模式可能丢数据。这是业务队列策略，不是 PCIe flow control。

## 10. MSI/MSI-X 的实现特点

驱动优先要求 49 个 MSI-X vector。`pci_enable_msix_range(..., 1, 49)` 即使成功返回
少于 49，也会被驱动主动禁用；MSI 路径则调用 `pci_alloc_irq_vectors_affinity()`，
见 `edge.c:5016-5058`。

**Risk：MSI vector 编号使用不符合现代 PCI API。** 内核 5.10 的 API 文档明确要求
用 `pci_irq_vector(pdev, nr)` 取得 Linux IRQ；当前代码却对 MSI 使用
`pdev->irq + i`（`edge.c:686-703, 922-929, 4994-5007`）。IRQ 编号不保证连续，
因此 MSI notify 在某些 irqdomain/platform 上可能请求错误 IRQ。MSI-X 路径使用
`entry[i].vector`，不存在这个特定问题。

**Risk：MSI 部分分配未校验数量。** 申请范围是 1..49，只要返回非负数就设置
`msi_enabled=1`，后续却可能使用到 vector 48。若只分到少量 vector，会越过实际
分配范围。

在当前 `DMA_INT_BRIDGE == 0` 路径，setup 向 EP 的 MSI generator control 写 1。
备用 bridge 路径会读取 Host 分配的 MSI address/data，在 EP outbound ATU region 0
建立对应窗口，再把地址和 data 写进 MSI generator。它非常直观地证明：MSI 最终
就是 EP 向 Host 指定地址发起的一笔 Memory Write。

## 11. AER、reset 与 remove

`pci_error_handlers` 覆盖 error_detected、slot_reset、resume 和 reset prepare/done，
见 `edge.c:6333-6463`：

- normal：声明可恢复；
- frozen：设备 offline、disable，要求 slot reset；
- permanent failure：断开；
- slot reset：重新 enable memory、set master、restore/save config state，再 online；
- resume：清 non-fatal AER status。

驱动只在当前 function 的 AER Uncorrectable Status 非零时写 `/var/intellif/Blocklist`，
用于区分“本设备真正报错”和“上游广播/传播过来的错误”。

`edge_device_offline()` 清 online bit 后，无超时地等待四个 transfer list 变空，见
`edge.c:5733-5748`。**Risk**：如果 DMA 已冻结且 completion 永不回来，AER recovery
或 remove 可能永久卡在这里。

remove 最多等用户打开实例约 10 秒，随后拆 IRQ、字符设备、engine、BAR 和 PCI
resource。一个特殊 ioctl 写 `BAR+0x80758 = 0x10000` 不立即执行，而保存到
`flr_brw`，在 remove 中设置 `FLR_ACTIVE`（`edge.c:6047-6053`）。这说明 reset
协议与驱动解绑顺序有硬件耦合，后续要与 EP controller 的 SR-IOV/FLR 中断寄存器
交叉验证。

## 12. `pcie` 与 `pcie_cascade` 不是简单复制

仓库还有 `pcie_cascade/`，但它不是 X6000 主数据面的同名备份：

| 维度 | `pcie/` x6000 | `pcie_cascade/` |
|---|---|---|
| 代码规模 | 6658 + 856 行 | 4820 + 654 行 |
| IRQ vectors | 49 | 25 |
| 通知组 | 2×24 | 1×24 |
| DMA mask | 主业务 ch3，P2P ch0..2 | ch0..2 |
| controller base | BAR+`0x400000` | BAR+`0x100000` |
| uDMA base | controller+`0x200000` | controller+`0x600000` |
| product handling | X6000 LM/VS/DS、双 group、P2P、exception、DMA pool | cascade position/event/GPIO shared memory |
| boot state ABI | 多 die/group 结构 | 单个 enum boot state |

因此二者必须作为不同硬件集成方案阅读。cascade 代码新增从共享 SRAM/efuse 判断
`0xA5..0xA8` 位置和 cascade card 的路径，并有专门的 cascade event INTx；X6000
主目录则加入双 group、P2P outbound window、异常 FIFO、boot uDMA、产品 magic、
DMA pool 和更复杂的 AER 策略。

## 13. 已确认的健壮性/安全风险

以下是源码审计结果，目标是帮助维护者定位边界，不是泛泛挑刺：

1. **ioctl ABI 编码依赖指针宽度。** 大量 `_IOW/_IOR` 的第三参数写成
   `struct foo *` 而不是 `struct foo`（`edge.h:780-831`），命令号编码的是指针大小。
   32-bit compat 用户态与 64-bit kernel 会不一致；tool 侧复制同样宏只能保证同
   架构“碰巧对上”。
2. **命令序号复用。** nr 112/113 同时给 temperature/inspur 与 ext read/write；
   完整 ioctl 值因 direction/size 不同而不同，但维护和 compat dispatch 易混淆。
3. **boot uDMA 临时 buffer 使用错误变量。** 分配到 `k_host_virt` 后却检查
   `dma_task.host_virt`，又对原用户指针做 `virt_to_phys()`，见
   `edge.c:4687-4717`。这是 Confirmed code bug；后续赋回 `host_virt` 后的 DMA map
   可能掩盖部分影响，但错误检查仍可能导致空分配继续执行。
4. **DMA mapping error 未检查。** `dma_map_single()` 后未调用
   `dma_mapping_error()`；`dma_map_sg()` 返回 0 也没有转成正常错误，而后续直接取
   `sgl[0]` 并建表（`edge.c:2478-2577`）。
5. **用户页失败会触发 BUG。** `sgm_get_user_pages()` 的正常错误在调用者变成
   `BUG_ON(rc < 0)`。恶意或普通坏地址都可能把 recoverable ioctl error 放大成
   kernel crash。
6. **描述符环 full/empty 歧义。** `head == tail` 返回全部 1024 项可用，而写满
   1024 项又会让 head 回到 tail，见 `edge.c:2609-2669`。没有保留空槽或 generation
   bit 时，排队传输存在覆盖未完成描述符的可能。
7. **timeout 不推进 tail。** 源码明确注释掉 tail update（`edge.c:2821-2845`），
   反复 timeout 可能逐步耗尽/混淆描述符环。
8. **raw BAR ioctl 缺少可见的范围校验。** `barno` 和 offset 可由用户提供，safe
   helper 只检查 BAR 指针/部分 group 状态，不验证 resource length。设备节点权限
   因而是重要安全边界。
9. **block memory copy 信任用户提供的 kernel virtual address。** alloc 返回
   `virt` 给用户，copy ioctl 再把用户传回的 `block_mem_virt` 直接作为内核指针，
   未按当前 file 的 allocation list 做归属/范围验证。这可能形成任意内核地址读写，
   必须结合 `/dev` 权限立即审计。
10. **open 错误路径泄漏。** 分配 `file_priv` 后若 BAR0 不可用或 IRQ setup 失败，
    `edge_open()` 直接返回而未释放对象，见 `edge.c:5061-5105`。

## 14. 推荐源码阅读路线

1. `edge.h:171-280, 470-637, 780-831`：先认地址、对象和 ABI；
2. `edge.c:5847-6103`：probe/remove 骨架；
3. `edge.c:3562-4045`：boot stage 与 mailbox；
4. `edge.c:3445-3560`：CPU MMIO path；
5. `edge.c:1833-3340`：页固定、DMA mapping、描述符、queue/completion；
6. `edge.c:4732-5058`：DMA IRQ 与 MSI/MSI-X；
7. `edge.c:459-988`：notify shared-memory + doorbell；
8. `edge.c:1664-1831`：P2P、ACS 与 ordering；
9. `edge.c:6333-6463`：AER recovery；
10. 最后读 ioctl switch 和辅助功能，避免一开始淹没在业务命令里。

## 15. 跨仓接口验证结果

| 本驱动观察 | 跨仓结论 | 状态 |
|---|---|---|
| BAR0 controller/uDMA窗口 | BL2 region9 + 最终产品DTS共同验证 | Confirmed |
| `0x7fa40` DE10–DE14 | BootROM/PCS/ATF/Linux/tool写读双方闭合 | Confirmed |
| mailbox ready/buffer | 三阶段固件与HAL逐地址吻合 | Confirmed |
| descriptor local `0x60000000` | AP SRAM后端与Host BAR0描述符视图闭合 | Confirmed |
| 49个MSI-X布局 | Host与BL2 relay索引吻合；EPF部署参数未找到 | Partial / OQ-012 |
| notify H2D doorbell `0x40000` | EP Linux capture/notify handler闭合 | Confirmed |
| uDMA interrupt写语义 | 源码用法确认，Databook自身冲突仍在 | OQ-010 |
| FLR_ACTIVE `0x80758` | BootROM实验、PCS VF骨架、Host延迟写已关联 | Partial / 实机验证 |
| group动态aperture | A0/A8/B0及A/B、LM/DS最终DTS已映射 | Confirmed |

完整路径见[综合架构](x6000_integrated_pcie_architecture.md)和
[统一地址地图](x6000_address_map.md)。

## 16. Source Reference

- Host driver：`source_repos/edge-driver/pcie/edge.c`、`edge.h`、`README.md`、`Makefile`
- cascade variant：`source_repos/edge-driver/pcie_cascade/edge.c`、`edge.h`
- 对应 Linux API 语义：`source_repos/linux/drivers/pci/msi.c:1173-1282`
- PCIe 概念背景：知识库 `linux/function_driver_probe_mmio_and_lifecycle.md`、
  `dma/pcie_dma_cadence_and_linux_dma_api.md`、`interrupts/intx_msi_msix_and_cadence.md`、
  `errors/error_model_aer_and_cadence.md`

当前没有运行硬件测试或加载 module；本文是静态源码研究。跨仓静态映射已完成，
runtime行为仍以实机config/APT/IOMMU/AER dump为最终裁判。
