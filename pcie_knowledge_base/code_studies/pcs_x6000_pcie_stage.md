# PCS 中的 PCIe：Link 不拆，地址地图重画

> 研究对象：`source_repos/pcs`，分支 `x6000`，commit `80d6faf2613d1b04e0166c9a258017a0b60588dc`。

## 1. 一句话定位 PCS

BootROM 把 PCIe EP 拉起来并提供最小下载通道；PCS（平台控制/二级启动固件，运行在 STAR Cortex-M）接过这条仍在线的 Link，重画 BAR0 后面的 APT 地址地图，继续接收 Host 下载的 BL2/ATF 镜像，并负责多 die 分发、低功耗与部分实验性 SR-IOV/FLR 协调。

这里最重要的结论是：主 X6000 EP 路径中的 `pcie_ep_init()` **没有重新初始化 PHY，也没有重新 enable LTSSM**。它相信 BootROM 已经把公路修通，PCS 只换路牌。

## 2. 启动调用链

```text
main()
  → bootstrap_analyze()
  → devices_init()
      → memory/timer/UART/eFuse product identity
      → if final_boot or BOOTMODE_PCIE:
          pcie_ep_init()
              → preserve active Link
              → rewrite BAR0 inbound ATU
              → rewrite APT 0..8
              → status magic = DE11DE11 (PCS)
  → secure/firewall/QoS/multi-die setup
  → images_load()
      → PCIE_BOOT: pcie_download_main()
      → optional: repeat download and AXI-DMA copy to each slave die
  → images_go()
      → release A55 reset
  → FreeRTOS scheduler
```

`devices_init()` 的条件与 BootROM 类似：显式 PCIe boot 一定接管；某些 `final_boot` 的本地介质路径也会提前准备 EP。FPGA/EMU 构建则由 `PCIE_BOOT` 宏直接控制。

Source:

- `board/edge10/main/main.c:207-267`, `devices_init()`
- `board/edge10/main/main.c:320-520`, `main()`
- `board/edge10/main/images_load.c:39-114`

## 3. PCS 为什么不用重训 Link

PCS 的 EP 初始化函数只有约 80 行，主要写 BAR、ATU、APT、scratch 和 boot state；和同文件中长达 300 多行的 `pcie_rc_init()` 对比非常鲜明：

- `pcie_rc_init()` 会管理 PMU、HSIO clock/reset、Torrent PHY、lane/mode、PERST 和 Link Training；
- `pcie_ep_init()` 完全不做这些事。

因此可以作出强证据推断：

```text
BootROM EP Link/PHY state
          │ 保持
          ▼
PCS pcie_ep_init()
          │ 只改 address translation / boot state
          ▼
Host 无需重新枚举即可继续使用 BAR0 下载
```

这也解释了为什么 BootROM master 后来删除了 GO 前的 `pcie_ep_clear_ibatu()`：PCS 接管时会主动重写 ATU，不再要求 ROM 先清零。

## 4. PCS 如何重画 BAR0 地址地图

### 4.1 第一层：Cadence BAR inbound ATU

PCS 写：

```text
BAR0 inbound ATU = 0x0000003f_00000000
```

含义不是把 Host BAR 地址固定为 `0x3f...`。Host BAR 地址仍由 RC/BIOS/Linux 分配；这里设置的是 EP 内部 BAR hit 后产生的 AXI aperture base。

例如 Host 把 BAR0 分到 `0x80000000`，那么：

```text
Host store 0x80080000
  → Memory Write TLP, address 0x80080000
  → EP BAR0 hit, offset 0x80000
  → inbound ATU address 0x3f00080000
  → APT region 1
  → final local target 0x460000000
```

### 4.2 第二层：SoC APT

PCS 使用 9 个 APT region：

| Region | BAR0 offset | APT input range | 最终 SoC 地址 | 大小 | 用途推断 |
|---:|---:|---:|---:|---:|---|
| 0 | `0x000000` | `3f00000000–3f0007ffff` | `0x60000000` | 512 KiB | AP SRAM、mailbox、boot state |
| 1 | `0x080000` | `3f00080000–3f000fffff` | `0x460000000` | 512 KiB | 另一 die/片上窗口 |
| 2 | `0x100000` | `3f00100000–3f0017ffff` | `0x1c60000000` | 512 KiB | 多 die 窗口 |
| 3 | `0x180000` | `3f00180000–3f001fffff` | `0x2060000000` | 512 KiB | 多 die 窗口 |
| 4 | `0x200000` | `3f00200000–3f00200fff` | `0x5a200000` | 4 KiB | PCIe system control |
| 5 | `0x201000` | `3f00201000–3f00202fff` | `0x5a216000` | 8 KiB | PCIe auxiliary/system registers |
| 6 | `0x203000` | `3f00203000–3f00203fff` | `0xa7400000` | 4 KiB | eFuse |
| 7 | `0x760000` | `3f00760000–3f0077ffff` | `0x240000000` | 128 KiB | 保留/共享内存窗口 |
| 8 | `0x780000` | `3f00780000–3f007fffff` | `0x242000000` | 512 KiB | 保留/共享内存窗口 |

随后写 `APT_REMAP_CTRL = 1` 启用重映射。

代码注释说“clear region 7~31”，实际循环是 `9..31`。这不是 off-by-two：region 7/8 刚好紧接着被配置，实际意图显然是保留 0–8、清 9–31；注释过时了。

Source:

- `board/edge10/drivers/pcie/pcie_init.c:470-561`

## 5. 启动阶段地址地图为何会变化三次

把三个仓库对起来，可以看见地址所有权的完整演进：

### BootROM 外部固件

先用以 `0x60000000` 为起点的临时 aperture，使 AP SRAM、PCIe system control 和 eFuse 可访问。目标是“Host 能下载，ROM 能活下来”。

### PCS

把 BAR0 inbound base 固定到 `0x3f00000000`，配置上表 0–8。目标是“继续下载 BL2，并覆盖多 die/板级启动窗口”。

### Linux EPC/EPF

Linux `edge10_ultra_hsio_pcie_card.dtsi` 再把 APT 改成运行期布局：

- AP SRAM mailbox；
- PCIe system/ATU/uDMA register windows；
- 多 die controller windows；
- BAR2 大块数据窗口。

特别注意，Linux runtime APT 与 PCS APT **不是一一相同**。Linux 会复用 region 编号并覆盖内容。例如 Linux region 1 把 BAR0+`0x80000` 指向 `0x5a200000`，而 PCS 同一 offset/region 指向 `0x460000000`。

所以 Debug 某个 BAR offset 时必须先问一句：“现在 CPU 跑到哪个启动阶段了？”同一个 `BAR0+0x80000` 在 PCS 和 Linux 眼里可能是两栋完全不同的楼。

## 6. Boot state 是 Host 的阶段指示灯

PCS 写：

```text
0x6007fa40 = 0xDE11DE11
```

与整个启动协议对应：

| 阶段 | magic | 含义 |
|---|---:|---|
| BootROM | `DE10DE10` | ROM PCIe mailbox ready |
| PCS | `DE11DE11` | PCS 已接管并重配 APT |
| 后续固件 | `DE12...DE14` | 由 edge-driver 的 boot 状态机继续识别 |

Host `edge-driver` 正是轮询这些 magic 来判断什么时候可以发送下一阶段镜像。它不是装饰性版本号，而是跨 PCIe 的分布式启动状态机。

PCS 还会：

- 清 `IPCM_NET_IP_STATUS_ADDR`；
- 清 PCIe scratch register 2（X6000 large model 用于保存 devnum）；
- 非 Die0 把自己的 PCS magic 通过 SoC AXI DMA复制到 Die0 的状态数组。

## 7. PCS 如何继续下载 BL2/ATF

PCS 保留 BootROM 的四个 mailbox 地址：

```text
H2D ready  0x6007fa00
D2H ready  0x6007fa80
H2D buffer 0x6007fb00
D2H buffer 0x6007fd80
```

`pcie_download_main()` 不再在 GO 时直接函数跳转：

1. 清 `download_done`；
2. 轮询 H2D ready；
3. 复用同一 CMD/DATA datastream parser；
4. 收到 GO 后，protocol 层只设 `download_done = 1`；
5. 检查缓存的 `sb_header.magic_num`；
6. 返回镜像 data size；
7. `images_load()` 完成多 die 分发；
8. `images_go()` 统一释放 A55 reset。

这比 BootROM 的“GO 立即跳函数指针”更像正规的多核启动管理器。PCS 必须先保证其他 die 的镜像和启动握手完成，不能让 Die0 收到 GO 就独自冲出去。

### 多 die 下载

Die0 对每个 slave die 重复进入下载流程，得到 BL2 长度后，用 SoC AXI DMA：

```text
Die0 BL2_LOAD_ADDR
  → AXI DMA channel
  → MULTI_DIE_ADDR(slave_die_id, BL2_LOAD_ADDR)
```

最后再下载 Die0 自己的 BL2。Host 工具侧因此可能看到多轮相同协议会话；不能把第二次 ready 当成第一次传输“重试失败”。

Source:

- `board/edge10/drivers/pcie/pcie_init.c:566-645`
- `board/edge10/drivers/protol/protol_processing.c:170-239,406-418,591-599`
- `board/edge10/main/images_load.c:39-114`
- `board/edge10/main/images_go.c:1108-1116`

## 8. Hot Reset 与 BAR 资源重分配

对特定 X200/card type，PCS 会在 Hot Reset 后补 BAR2/BAR4 大小，并检查 `REG_PF_BAR_CFG1 == 0x505` 是否回到默认值。如果是：

1. 在 scratch register 1 设置 bit 1；
2. 注释明确说后续 `hal_pcie_open()` 需要调用 `hal_pcie_reconnect`；
3. Host 侧重新分配 BAR resource。

这个注释揭示了硬件到软件的完整因果链：

```text
Hot Reset
  → EP BAR size register 回默认
  → 原 Host BAR allocation 不再匹配
  → PCS scratch 标记“需要 reconnect”
  → 后续软件读取标记
  → Host 重新枚举/分配 BAR
```

`PCIE_CASCADE` 构建还会写固定 BAR 编码 `0x85f185d1` 并使用相同 scratch 标志。

## 9. PCS 里还有一个 RC 初始化器，但当前主路径没用

`pcie_rc_init(rc_mode)` 能把 x8 controller 0 与 x2 controller 1/2 配成 RC，完成：

- lane 6/7 分给 USB；
- PIPE mux 选择；
- controller 0 x4 Gen3；controller 1/2 x2 Gen3；
- RC mode select；
- PERST 输出；
- 多控制器 Link Training 和等待 Link Up。

但 `main.c` 中唯一直接调用被注释为 `//pcie_rc_init(1);`。因此它应归类为 cascade/验证/备用能力，而不是当前 X6000 Host-facing EP 启动的必经路径。

## 10. PCIe 低功耗由 PCS 长期参与

PCS 并不会在 A55 启动后消失；它进入 FreeRTOS scheduler，继续作为平台控制固件运行。

`pcie_pm.c` 提供：

- 开/关 ASPM L0s；
- 开/关 ASPM L1；
- 配置 entry timeout、advertised exit latency；
- STAR IRQ 37 接收 PCIe PM 状态；
- L0→L1 且 Host 允许整芯片掉电时，用 mailbox 8 通知 A55；
- L1 exit 时唤醒 A55/释放 semaphore；
- L0→Recovery 时记录并清中断。

这里可以看出 PCIe Link state 和 SoC power state 的区别：LTSSM 进入 L1 只是链路低功耗；PCS 还要检查 Host 的 `allow_chip_power_off` 协议位，才敢进一步关 SoC。不能把“PCIe L1”直接翻译成“芯片已掉电”。

这些能力由 PSM 消息/调试配置按 board type 开启，不是默认无条件启用。

## 11. 多 group 管理路径也借用了 PCIe uDMA

`main/smbus_group.c` 解决的是 Group A 管理 Group B 地址空间。它的路径比较绕：

```text
PCS on Group A
  → local AXI DMA 访问 Group B 的远程 AXI window
  → 把 Cadence uDMA descriptor 写进 Group B AP SRAM
  → 通过远程 MMIO 启动 Group B PCIe uDMA channel 0
  → Group B PCIe uDMA 在其 AXI address 与 PCI address 间搬运
  → PCS 轮询远程 uDMA interrupt status
```

这不是 Host Linux 驱动的普通 DMA 路径，而是板内/组间管理通道。源码为 VS、DS、普通 X6000 定义不同的远端 PCIe base 和 PCI buffer address；当前文件顶部直接 `#define X6000`，VS/DS 宏未启用。

值得注意：PCS 控制的是 **远端 A8/Group B 的 uDMA 寄存器**，descriptor 的 `sys_addr` 是 Group B AXI 地址，`ext_addr` 是该 PCIe 视角的 PCI address。理解这一层后，才不会把本地 AXI DMA和 PCIe uDMA混成同一个搬运器。

## 12. SR-IOV / VF / FLR 代码目前是未接通的实验骨架

`vf.c` 包含：

- 为 VF0–VF3 设置 MSI-X outbound region；
- 按 4-die/8-die topology 切换 master chip ID，读取每个 die 的 MSI-X table；
- PCIe uDMA interrupt handler 框架；
- SR-IOV/FLR IRQ handler。

但静态调用图显示：

- `vf_int_init()` 没有调用者；
- `pcie_flr_init()` 在 `main.c` 中被注释；
- DMA IRQ handler 中“raise mailbox to VF A55”仍是 TODO。

因此不能说 PCS 已经完整支持 SR-IOV。更准确的说法是：寄存器编程思路和中断入口已经存在，但当前 x6000 tip 的启动路径没有把它们接通。

还有一处疑似明确 bug：FLR handler 读取 `SOFTWARE_RESET_FLAG` 后，如果值为 INIT，却把 `SOFTWARE_RESET_RUNNING` 写到 `PCIEX8_SRIOV_INT_CLR_ADDR`，看起来本应写回 `SOFTWARE_RESET_FLAG`。需结合设计意图验证。

## 13. Linux 接管时发生什么

Linux 平台 driver 的 EP probe 初始化资源但不重新执行完整 controller mode/speed/lane 流程，和 PCS/BootROM 的所有权模型一致。

随后 Linux EPC/EPF 会：

1. 根据最终A/B、LM/DS产品DTS更新选定runtime APT，未列region依赖固件保留；
2. 设置 BAR backing 与 outbound region；
3. 建立 MSI-X table/PBA；
4. 启动 function/link API；
5. 把 boot magic 推进到 Linux 阶段。

由此形成一条连续但多次换配置的 Link：

```text
BootROM: Link + tiny boot map
   ↓ no retrain
PCS: boot/multi-die map
   ↓ A55 reset release
ATF: DE12 bulk、secure boot、C2C enumeration与IRQ relay（已验证）
   ↓
Linux: runtime BAR/uDMA/notify map
```

## 14. 静态审计与 Debug 风险

### 14.1 切换 APT 的瞬间需要协议静默

PCS 直接重写已在线 BAR0 后的映射。若 Host 同时访问刚被改写的 offset，transaction 可能落到旧目标或新目标。代码本身没有显式握手包围 APT 更新，实际安全依赖阶段 magic/Host 工具不并发访问。

### 14.2 下载循环无超时

`pcie_download_main()` 会永久等 H2D ready。对于 PCIe boot 这是合理策略，但诊断必须结合 boot magic 和 Link status，否则看不出是 Host 未发送、BAR 映射错误还是协议卡死。

### 14.3 `pcie_slave_receive()` 的 `buff` 拷贝路径与 download fast path 不同

普通 protocol adapter 会把 mailbox memcpy 到调用者 buffer；`pcie_download_main()` 直接把 mailbox 当 packet buffer。维护协议时要同时检查两条路径，避免只修其中一处长度验证。

### 14.4 RC 初始化错误不阻断启动

即使 `pcie_rc_init()` 中某条 Link timeout，函数在多个等待点只是打印并继续，最后返回 0；只有 PHY clock unstable 返回 -1。调用者若未来启用该函数，不能把返回 0 等同于所有 RC Link Up。

## 15. Source Reference

- PCS `80d6faf2613d1b04e0166c9a258017a0b60588dc`
  - `board/edge10/drivers/pcie/pcie_init.c/.h`
  - `board/edge10/drivers/pcie/pcie_pm.c/.h`
  - `board/edge10/drivers/pcie/pcie_fw.c/.h`
  - `board/edge10/drivers/pcie/vf.c/.h`
  - `board/edge10/drivers/protol/protol_processing.c`
  - `board/edge10/main/main.c`
  - `board/edge10/main/images_load.c`
  - `board/edge10/main/images_go.c`
  - `board/edge10/main/smbus_group.c`
  - `Makefile`
- BootROM 对照：[bootrom_pcie_boot_flow.md](bootrom_pcie_boot_flow.md)
- Linux 对照：[linux_x6000_pcie_stack.md](linux_x6000_pcie_stack.md)
