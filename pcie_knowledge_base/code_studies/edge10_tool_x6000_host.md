# edge10_tool x6000：从命令行一路追到 BAR、DMA 与 Hot Reset

> 主仓：`source_repos/edge10_tool`，分支 `x6000`，commit `1daae3e45e22b87c1f847be19a3209a770bd321b`（2026-08-20）。
>
> 实际构建使用的 HAL：`third_party/hal` 分支 `x6000`，commit `aa005c36b477033c26324368d88c69b388b203a4`（2026-08-20）。本文聚焦 Linux Host 侧 PCIe 工具，不把 USB 和设备本地外设测试混进来。

## 1. 先说结论：它不是一个“小测试程序”

`edge10_tool` 的 Host PCIe 部分接近两万行。它同时是：

- 设备发现和稳定编号工具；
- BootROM→PCS→BL2→Linux 的固件下载客户端；
- `edge.ko` 用户态 HAL 和 ABI exerciser；
- BAR/MMIO、uDMA、P2P、共享内存和带宽压测工具；
- Hot Reset、remove/rescan 和 AER blocklist 运维工具；
- 多 die/group 状态、温度、异常、串口日志等板卡诊断前端。

它不只是单项测试程序，更接近 Host 侧的 PCIe 控制台：既包含面向使用者的测试命令，也暴露了不少寄存器、DMA 和调试入口。第一次阅读时应先按数据路径分类，而不是从命令列表顺序阅读。

## 2. 源码分层

```text
edge10_tool CLI
  ├─ src/host/cases/download/download_host.c
  │      多设备/多卡下载编排、hot reset、线程与日志
  │
  └─ src/host/cases/pcie/pcie_cases/*.c
         scan / bario / xferstress / hotreset / debug / ...
                    │
                    ▼
third_party/hal x6000
  ├─ host/communication/pcie/hal_pcie.c
  │      open/xfer/mmap/boot/DMA-pool/bandwidth
  ├─ host/dcmi/hal_dcmi.c
  │      boot state/card info/hot reset/exception/remote command
  └─ download/
         stage machine、BUT协议、固件清单、512 KiB bulk path
                    │
                    ▼
/dev/<slot>_<96-bit-uid>  +  /dev/edge_smbus
                    │ ioctl / mmap
                    ▼
Host edge-driver pcie/edge.c
                    │
                    ▼
BAR MMIO / Cadence uDMA / MSI-X / PCI core / bridge reset
```

### 一个容易踩的版本坑

主仓 gitlink 固定的是旧 HAL commit `8ed32a3b...`，其 `hal_pcie_open()` 只有两个参数，和工具 x6000 源码不兼容。但 `build.sh:87-106` 会取得当前工具分支名，然后在 HAL 子模块中执行：

```text
git checkout x6000
git pull origin x6000
```

因此真实构建基线不是 gitlink commit，而是拉取时刻的 HAL `x6000` tip。本次研究把当日 tip 固定为 `aa005c36...`。这也是一个可复现性风险：半年后用同一个 tool commit 构建，HAL 可能已经变了。

Source:

- `.gitmodules`
- `build.sh:87-147`
- `CMakeLists.txt:73-189`

## 3. PCIe 命令地图

`pcie_cases.h` 注册了 27 个子命令。对 PCIe 学习最有价值的分组如下：

| 类别 | 命令 | 主要硬件/内核路径 |
|---|---|---|
| 发现 | `scan`, `slot_info`, `probe`, `check` | sysfs、PCI topology、config space、devprop |
| BAR | `bario`, `rreg`, `vcom`, `debug`, `ip` | `READ/WRITE_BAR` ioctl → `readl/writel` |
| 数据 | `xferstress`, `filecopy`, `shm`, `mmapdp` | uDMA、SG、DMA pool、BAR2 mmap、P2P |
| 监控 | `bwmon`, `debug`, `exception` | DMA counters/queue、exception FIFO/AER |
| 复位 | `hotreset`, `scan -r` | sysfs remove、Bridge Control、rescan |
| 板级 | `eeprom_x6000`, `temp`, `power`, `led`, `nor` | switch/config/BAR 后面的板级管理 |
| 生成 | `mkfw`, `genscripts`, `tmux` | firmware packaging、测试脚本与串口日志 |

`debug.c` 一份就有 4,249 行，覆盖 Link state、lastword、DMA descriptor、switch、异常 FIFO、NoC 和温度。它适合作为“有现象时查某一项”的诊断资料，不适合拿来当 PCIe 入门第一章。

Source:

- `src/host/cases/pcie/pcie_cases/pcie_cases.h:1-68`
- `src/host/cases/pcie/pcie_cases/`

## 4. `scan`：Linux 已经枚举完，工具再建立稳定身份

### 4.1 它不负责 PCI Enumeration

Linux PCI core 先发现 `17cd:0100/2000`，调用 edge-driver probe；驱动映射 BAR、读取 slot key 和 96-bit unique ID，然后创建：

```text
/sys/class/edge/<8-hex-slot>_<24-hex-uid>
/dev/<8-hex-slot>_<24-hex-uid>
```

`scan` 扫的是 `/sys/class/edge`，不是直接发 Configuration Read TLP。因此工具层的“scan”准确含义是：**在内核已枚举并绑定驱动之后，发现 edge 设备并建立用户态编号。**

它开头会向 `/sys/bus/pci/rescan` 写 1；真正做 bus walk、读 Vendor ID 和分配资源的是 Linux PCI core。

### 4.2 为什么还要 `devprop.ini`

BDF 会因为插槽、桥枚举顺序、remove/rescan 改变；`device_id=0` 又希望保持稳定。工具把三种身份拼在一起：

| 身份 | 来源 | 稳定性/用途 |
|---|---|---|
| BDF / slot key | PCI topology | 本次枚举的物理位置 |
| 96-bit unique ID | EP AP SRAM/eFuse映射 | 识别同一颗芯片 |
| device ID | `devprop.ini` | 给用户和自动化脚本的短编号 |

`scan -f` 生成配置，`scan -c` 用 unique ID 将历史条目与当前节点配对，再用 ioctl 6 把 device ID 写回驱动。工具也记录 Host-side parent slot，给多卡和 hot reset 使用。

### 4.3 AER blocklist

`/var/intellif/Blocklist` 的 `[aer] aer=1` 会启用过滤：匹配其中 UID 的节点不进入本次设备列表。全 `f` UID 也直接跳过。它不是让 PCI core 停止枚举设备，而是让用户态管理层不再使用已知异常设备。

Source:

- Tool `scan.c:425-630,1828-2208,2216-2345`
- Driver `edge.c:5528-5654,5916-5961`

## 5. 用户态 ABI 总表

HAL 与 edge-driver 用 ioctl magic `'E'`。核心编号在两边一致：

| nr | ioctl | 用户动作 | 驱动/硬件结果 |
|---:|---|---|---|
| 0 | UDMA D2H | Device→Host DMA | EP 读本地、向 Host 发 Memory Write TLP |
| 1 | UDMA H2D | Host→Device DMA | EP 向 Host 发 Memory Read，接收 CplD |
| 2/3 | MMIO D2H/H2D | CPU copy 数据 | Host CPU 通过 BAR load/store |
| 4/5 | BOOT READ/WRITE | ROM/PCS 小包 | AP SRAM mailbox + ready flag |
| 6 | SET DEVICE ID | 稳定编号 | 更新驱动中的逻辑 ID |
| 7 | GET CHANNEL STATUS | 查询 DMA 队列 | running/tasks/bytes |
| 8 | GET EP FLAGS | scratch flags | reset/upgrade 状态 |
| 9 | GET BOOT STATE | 查询 group/die | 解析 DE10–DE14 magic |
| 10 | GET PCIE INFO | 查拓扑与链路 | BDF、parent、speed、width |
| 11/12 | BOOT READ/WRITE PRO | BL2+消息 | DDR message buffer |
| 98/99 | P2P OB/IB | Peer DMA | 一个 EP 直接访问另一 PCI function/window |
| 100/101 | READ/WRITE BAR | 32-bit BAR访问 | `readl/writel` 到映射后的 BAR |
| 102/103 | ALLOC/FREE BLOCK MEM | DMA pool | coherent block + user mmap |
| 104/105 | COPY TO/FROM BLOCK | user↔coherent block | CPU copy，非PCIe传输 |
| 106/107 | BW MON START/STOP | 统计区间 | 汇总各 uDMA channel bytes/time |
| 110 | BOOT UDMA H2D | BL2 bulk download | uDMA 到 512 KiB DDR buffer |
| 120 | GET DMA POOL INFO | 查询模块参数 | pool enable/block size/count |

ABI 结构体也已对照：`xfer_task`、`boot_rw`、`block_mem`、`bw_info`、`group_boot_state` 的字段顺序与当前 edge-driver x6000 匹配。这里不能只比较字段名字；ioctl 穿过内核边界的是二进制 layout，类型大小、padding 和 32/64-bit ABI 都是契约的一部分。

Source:

- HAL `host/communication/pcie/hal_pcie.c:17-95`
- HAL `host/dcmi/hal_dcmi.c:73-93`
- Driver `pcie/edge.h:583-680,779-832`
- Driver `pcie/edge.c:5192-5322`

## 6. 完整 PCIe boot 下载路线

### 6.1 顶层编排

`edge10_tool download` 为每个 device 建一个线程。X6000 PCIe 若允许 Hot Reset，会先并发 reset 所选设备，再将 `is_reboot` 清掉，随后每个 device 运行：

```text
hal_download()
  → download PCS
  → download BL2 images
  → download FIP images
  → download boot image
  → optional: Linux 后下载刷机 image + command group
```

双 group 板卡需要协同 reset。工具明确阻止用户只用 `-d/-n` 重启半张 X6000 卡，除非加 `-r` 或改用 card selection；因为只复位 A 或 B 会让 C2C 另一端看到 Link Down，留下半活不活的系统。

### 6.2 阶段状态机

下载库每进入一层都轮询 boot state：

| 等待状态 | magic | 接收方 | 下载内容 | 传输方式 |
|---|---:|---|---|---|
| BOOTROM | `DE10` | BootROM | `pcs-bh.bin` | 小包 mailbox |
| PCS | `DE11` | PCS | A/B group、die 1/7/8/0 的 BL2 | 小包 mailbox |
| BL2 | `DE12` | ATF BL2 | FIP、boot image | 512 KiB uDMA bulk |
| LINUX | `DE14` | Linux service | 刷机大镜像/命令 | runtime channel |

`hal_pcie_boot_send/recv()` 先 ioctl GET_BOOT_STATE：早于 BL2 使用 4/5，BL2 以后使用 11/12。这是控制消息通道。FIP 和 boot payload 则明确走 ioctl 110。

### 6.3 512 KiB bulk 如何与 ATF 对上

X6000 固定：

```text
packet buffer = 512 KiB
first 4 bytes = this packet payload length
payload max   = 512 KiB - 4
device target = 0x2_00000000 + 0xa0000000
channel mask  = group A: bit0, group B: bit2
```

Host 路径：

```text
read(firmware, buf + 4)
 → *(u32 *)buf = len
 → ioctl BOOT_UDMA_H2D_XFER
 → edge-driver pins/maps Host buffer
 → programs Cadence uDMA
 → EP issues Memory Read TLPs to Host DMA address
 → data arrives in BL2 H2D bulk DDR buffer
 → Host-side driver signals ready/completion
```

ATF `pcie_download_main()` 正好从该 DDR buffer 读长度和第一包 `sb_header`，用设备 AXI DMA落到镜像目标地址；所有包发完后才返回一次 CRC/magic response。工具也只在 bulk 循环结束后收一次 `R_RCV_DONE`，两边协议完全吻合。

### 6.4 多 group 顺序

Group A 先完成 PCS/BL2 并进入 BL2；随后工具为 Group B 下载 PCS→BL2→FIP，再回到 Group A 下载 FIP。boot image 则为 A/B 两组并行线程。

VS 型号 FIP 清单包含 die 1/7/8/0；LM/DS 读取 product magic 后缩减为各 group 的 die0 FIP。这个动态选择与 ATF 中不同 product 的 C2C/多 die分支相呼应。

Source:

- Tool `src/host/cases/download/download_host.c:320-382,680-855`
- HAL `download/api/hal_download.c:150-310`
- HAL `download/download/firmware.c:31-304`
- HAL `download/download/init.c:356-430`
- HAL `download/download/core.c:55-202`
- HAL `download/comm/comm.c:137-163`
- HAL `host/communication/pcie/hal_pcie.c:1244-1337`

## 7. `bario`：一句 read 最终是什么 TLP

示例：

```bash
edge10_tool pcie bario -d 0 r 0 0x7fa40
```

调用链：

```text
bario
 → hal_pcie_open(device0, groupA)
 → ioctl EDGE_IOCTL_READ_BAR {barno=0, addr=0x7fa40}
 → edge-driver validates range
 → readl(edev->bar[0] + 0x7fa40)
 → Host CPU MMIO load
 → Root Port emits Memory Read Request TLP
 → EP BAR0 hit
 → inbound ATU + APT → AP SRAM 0x6007fa40
 → Completion with Data returns DE1x magic
```

写操作则是 Posted Memory Write TLP，一般没有 Completion；CPU 的 `writel()` 返回不表示 EP 内部目标模块已经完成所有副作用。

`bario` 支持 BAR0/1/2/4，读写粒度固定 32 bit；dump 只是循环发很多单次 ioctl，不是一次向量化 BAR read。大量 dump 因而会包含 syscall 和单笔 PCIe read completion 的开销，不适合当带宽测试。

Source:

- `src/host/cases/pcie/pcie_cases/bario.c:20-183`
- Driver `pcie/edge.c:3400-3452,5258-5263`

## 8. `xferstress`：九种 mode 其实是三类数据路径

### 8.1 模式表

| mode | 名称 | Host memory | 发起者/方向 | 是否校验 |
|---:|---|---|---|---|
| 0 | DMA round-trip | malloc/SG或block pool | H2D + D2H | 是 |
| 1 | MMIO round-trip | BAR2 mmap | Host CPU memcpy | 是 |
| 2 | inbound block DMA | coherent block | H2D only | 否 |
| 3 | outbound block DMA | coherent block | D2H only | 否 |
| 4 | inbound SG DMA | user pages | H2D only | 否 |
| 5 | outbound SG DMA | user pages | D2H only | 否 |
| 6 | MMIO read probe | BAR2 mmap | Host CPU read | 否 |
| 7 | P2P inbound DMA | peer PCI address | peer→device | 否 |
| 8 | P2P outbound DMA | peer PCI address | device→peer | 否 |

项目把 H2D 称为 inbound、D2H 称为 outbound，是站在 Endpoint/uDMA 的视角：数据进入 EP 是 inbound；EP 向外写 Host 是 outbound。

### 8.2 SG DMA 与 block DMA

SG 路径把普通用户 buffer 传给驱动。驱动 `get_user_pages_fast()`、`dma_map_sg()` 后取得 DMA address/IOVA，构建 uDMA descriptor。

Block 路径先通过 ioctl 102 分配 coherent DMA block，返回：

- kernel virtual address（opaque cookie）；
- Host DMA/physical address；
- 用户 `mmap()` 后的 virtual address。

测试中的 `COPY_TO_BLOCK_MEM` / `COPY_FROM_BLOCK_MEM` 只是用户 buffer 与 coherent block 的 CPU copy。真正跨 PCIe 的动作仍是后续 UDMA ioctl。

### 8.3 MMIO mmap 不是把任意 SoC 地址直接映给进程

用户传入的是 SoC target address，如 `0x2b0000000`。HAL 先识别 product magic：

- DS：固定 16 MiB window；Group B 在 BAR2 再偏移 16 MiB；
- LM：Group B 在 BAR2 偏移 64 MiB；
- VS：读取运行期 APT region 12/28/29/30/31，根据 target remap 反算 BAR2 offset。

然后 `mmap(fd, offset)`；edge-driver 的 `mmap` 再把这个 file offset 变成 BAR2 PCI resource PFN。最终进程拿到的是 Host virtual address，CPU 对它 `memcpy` 会产生 PCIe Memory Write/Read TLP，并不是“CPU 可以直接寻址设备物理地址”。

### 8.4 一个性能读数到底包含什么

- mode 0 SG：包含 page pin/map、descriptor 和 DMA completion 软件开销；
- block DMA：计时区间排除了 user↔coherent block copy，因此更接近 PCIe/uDMA 本体；
- MMIO：包含 CPU memcpy、PCIe request/completion 和映射属性影响；
- `bwmon`：驱动按 uDMA channel 的 bytes/timestamp 统计，最适合观察 DMA engine；不统计普通 BAR memcpy。

拿 mode 1 的数字直接和 mode 3 比，像拿快递员跑一趟的时间和货车在高速上的速度比——都叫“传输”，测的不是同一段。

Source:

- `src/host/cases/pcie/pcie_cases/xferstress.c:1-480,1019-1819`
- HAL `host/communication/pcie/hal_pcie.c:838-878,950-1218,1376-1474`
- Driver `pcie/edge.c:2280-3320,4256-4408,5332-5408`

## 9. `shm`：共享内存之上自己造一个 ring

`shm -w` 将 BAR2 的 16 MiB VMM 窗口映进进程，每个线程拿 512 KiB，头 16 byte放 read/write pointer，后面按 64-byte entry 写循环队列。

这说明两个层次要分开：

1. PCIe/Bar mmap 只提供“双方可访问的一块地址空间”；
2. read pointer、write pointer、满/空判断是应用协议，不是 PCIe flow control credit。

代码没有显式 memory barrier，也没有对 shared pointer 使用原子/volatile 语义；在 x86 Host 上常常能跑，在更弱内存序 CPU 或真实跨设备并发下需要特别验证。PCIe ordering、CPU cacheability 和编译器重排是三件不同的事。

Source:

- `src/host/cases/pcie/pcie_cases/shm.c:1-241`

## 10. `bwmon`：队列状态不等于 Link 带宽

`bwmon start` 清/记 uDMA 统计起点；`stop` 读取每个 channel 的 tx/rx bytes 和 timestamp，并查询 running、queued task count、queued bytes。

它显示的是 **uDMA effective throughput**，不是 PHY raw rate：

- 不含 128b/130b 编码等物理层细节的直接测量；
- 不含普通 CPU BAR MMIO；
- 包含 descriptor、TLP header、read completion、flow control、Host memory/IOMMU 等综合影响。

如果 Link 是 Gen3 x8，理论线速只是天花板；`bwmon` 是货物实际过收费站的速度。

Source:

- `src/host/cases/pcie/pcie_cases/bwmon.c:1-160`
- Driver `pcie/edge.c:4409-4450`

## 11. Hot Reset：不是“往 EP 某寄存器写个 reset”

单 device hot reset 流程：

```text
检查没有其他 edge/notifier handle
 → /dev/edge_smbus 全局/总线锁
 → 读取 upstream bridge Bridge Control (offset 0x3e)
 → 清各 die boot state，保留 dynamic-IP scratch bits
 → sysfs remove EP
 → Bridge Control.Secondary Bus Reset = 1
 → wait
 → Secondary Bus Reset = 0
 → parent/global rescan，最多重试
 → 等字符节点重建
 → 恢复 logical device_id
 → bus unlock
```

Secondary Bus Reset 是桥下游层级的 PCIe reset 机制。设备会丢失配置状态，Linux 也会销毁并重建 `pci_dev`/edge char device；旧 fd 和旧 BAR mapping 不能继续当作有效对象。

工具在 reset 前把 slave die boot state 预置为 ROM magic，并清 scratch 中除 dynamic IP 外的位，是为了让 reset 后固件状态机与 Host 重新同步，而不只是让 LTSSM重新训练。

Source:

- `src/host/cases/pcie/pcie_cases/hotreset.c:1-98`
- HAL `host/dcmi/hal_dcmi.c:320-690`

## 12. Debug 工具如何分层使用

推荐把工具命令按故障层次使用：

```text
看不到设备
  → lspci / dmesg / scan -l
  → Link/BDF/driver binding

设备存在但阶段卡住
  → bario 读 0x7fa40 / debug boot state
  → DE10/11/12/14 定位 ROM/PCS/BL2/Linux

BAR访问失败
  → bario 小范围 32-bit read
  → 对照当前阶段 APT，不要先跑大段 dump

DMA失败
  → xferstress 单线程固定小长度
  → bwmon + channel status + debug DMA descriptor
  → 再增加 SG、线程和 group

C2C/多die异常
  → debug linkstates / exception FIFO
  → 区分 Host-facing Link 和 A8↔B0 C2C Link

恢复测试
  → hotreset 单设备/整卡
  → 同时观察 remove、rescan、boot magic 与节点重建
```

## 13. 静态审计发现

以下风险来自源码阅读，尚未声明已上板复现：

### 13.1 HAL 子模块不可复现

构建脚本忽略 gitlink固定版本、跟随同名远端分支 tip。HAL ABI 与 tool/driver强耦合，建议发布物记录 tool、HAL、edge-driver 三个 commit。

### 13.2 `hal_pcie_mmap()` 对未知 product magic 仍继续

DS/LM/VS 都不匹配时只打印 error，`offset` 保持 0，随后仍 mmap BAR2 offset 0。更安全的行为应立即返回错误，否则错误产品身份会悄悄变成“映到了一个合法但错误的窗口”。

### 13.3 非页对齐 mmap 返回地址处理不完整

HAL 把 file offset 向下页对齐并扩展长度，但把 `mmap()` 返回的 page base 直接交给调用者，没有加 `page_off`。当前压测地址通常页对齐，所以问题被藏住；任意非对齐 `start` 会访问目标之前的地址。`munmap()` 还用 `start` 的低位而非反算 file offset 的低位，依赖 APT 映射保持同页偏移。

### 13.4 `xferstress -g` 接受 2

帮助只定义 group 0/1，校验却是 `group_id > 2`，因此 2 会通过。HAL 中许多地方把任意非零值当 Group B，但 `xfer_task.group_id` 又原样传到驱动，存在错误 group 或数组索引风险。

### 13.5 P2P mode 的 peer address 可能未初始化

mode 7/8 未强制要求 `-e`，局部变量 `peer_addr` 没有默认值。漏参数时可能把随机栈值当 peer PCI address。

### 13.6 `shm` 缺少明确的跨设备内存序约束

ring pointer 与 payload 之间没有 compiler/CPU barrier；仅靠普通 C load/store 不能完整表达设备共享队列的发布/消费顺序。

## 14. 读源码的推荐路线

1. `pcie_cases.h`：先知道工具能做什么；
2. `bario.c`：最短的 BAR→TLP例子；
3. HAL `hal_pcie.c`：理解 ioctl 与 mmap 适配层；
4. edge-driver `edge_ioctl()`：把 API 连接到内核实现；
5. `xferstress.c`：比较 MMIO、SG、coherent block、P2P；
6. download `firmware.c` + `core.c`：理解跨阶段下载；
7. BootROM/PCS/ATF 对端代码：闭合 ready flag、APT 和 bulk buffer；
8. `scan.c` 与 `hal_dcmi.c`：理解设备身份、reset和运维；
9. 最后按具体问题选读 `debug.c`，不必从头顺序阅读四千行寄存器操作。

## 15. Source Reference

- `src/host/cases/pcie/pcie_cases/pcie_cases.h`：命令注册表。
- `src/host/cases/pcie/pcie_cases/scan.c`：sysfs发现、devprop、AER blocklist、P2P和hot reset编排。
- `src/host/cases/pcie/pcie_cases/bario.c`：直接 BAR ioctl。
- `src/host/cases/pcie/pcie_cases/xferstress.c`：DMA/MMIO/P2P九种模式。
- `src/host/cases/pcie/pcie_cases/shm.c`：BAR2 mmap共享队列。
- `src/host/cases/pcie/pcie_cases/bwmon.c`：uDMA吞吐与队列状态。
- `src/host/cases/download/download_host.c`：多设备下载和reset线程。
- HAL `host/communication/pcie/hal_pcie.c`：Host PCIe API→ioctl/mmap。
- HAL `host/dcmi/hal_dcmi.c`：拓扑、boot state、exception与Hot Reset。
- HAL `download/download/{init,firmware,core}.c`：X6000固件清单、阶段与bulk协议。
- HAL `download/comm/{comm,boot_stage}.c`：PCIe transport和状态轮询。
- Driver `pcie/edge.[ch]`：所有用户 ABI 的内核实现。
