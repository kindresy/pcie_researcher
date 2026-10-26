# BootROM PCIe 全分支差异矩阵

> 本文比较 `source_repos/bootrom` 的 21 个远端分支。判断依据是分支 tip 的实际 PCIe 源码 blob、构建条件和调用点；分支名只用于定位，不能单独证明功能差异。

## 1. 怎么读这份矩阵

这里有两类“差异”必须分开：

- **PCIe 实现差异**：初始化流程、BAR、ATU、LTSSM、下载协议真的不同；
- **仓库历史差异**：分支因为 USB、D2D、WDI 或验证报告不同而产生不同 tree hash，但 PCIe 核心文件完全相同。

例如 `multi_die-err-test` 的简单文件名扫描得到 73 个 PCIe 命中文件，绝大多数只是 T32 coverage XML；它的三个 PCIe 核心源文件与 master blob 完全相同。把 73 当成“这个分支有巨量 PCIe 功能”就会被目录结构骗得很惨。

## 2. 三代实现

### 第一代：Cadence 参考驱动 API（2021）

代表：`boot_form_nand`、`v0.86r`。

特点：

- 仓库保留 `pcie_core_driver.c`、`pcie_obj_if.c` 和大量 Cadence interface/header；
- BootROM 用 `PCIE_OBJ` API 探测 EP/RP strap、初始化 driver、打印 LTSSM、等待 L0、设置 inbound translation；
- 配置参数使用早期三段格式，EEPROM/NOR 与 eFuse 的布局不完全相同；
- 显式打开 SR-IOV 和 ARI wrapper bit；
- Host helper driver 最早仍绑定示例 ID `10ee:8014`。

这是“功能齐全但对 ROM 偏重”的实现。

### 第二代：直接寄存器、紧凑 ROM 流程（2021–2022）

代表：`v0.95*` → `v1.0r_pre*` → `tapeout_asic` → `20-*` → `pcie_init` → `master`。

特点：

- 移除 20K+ 行 Cadence 生成 driver，直接写 wrapper、LM、AXI wrapper 和 PHY 寄存器；
- LTSSM/L0 判断逐渐由 core state API 简化为 wrapper link status；
- 配置包从早期 `pcie_init_parameters()` 演进到带 checksum、三 stage offset 的 `pcie_load_firmware()`；
- 加入 Hot Reset 分流、external init、PLL bypass、ready magic、双 eFuse block 备份、GPIO93 时序与更严格 delay；
- Host helper driver 从 `pcie_boot` 重命名为 `edge_boot`，PCI ID 固定为 `17cd:0100`。

### 第三代：mkfw 多产品外部固件（2023–2026）

代表：`pcie-mkfw-p2p`、`pcie-mkfw-x2000`、`x6000_pcie_boot`、`pcie-mkfw`。

特点：

- 基础 BootROM 主流程基本不动，变化集中在可抽取的 `pcie_main_ex.c`；
- 从 X2000/X5000 扩展到 X6000 多产品形态；
- 外部固件直接设置 Device ID、BAR 编码、MPS、ordering、NP outstanding、APT、eFuse 可访问窗口；
- 后期加入版本/SN/reset history、多 die boot state，以及 FLR/MSI-X 中断实验。

## 3. 21 个分支逐一结论

| 分支 | tip | PCIe 代码世代 | PCIe 特点与结论 |
|---|---|---|---|
| `20-spi_master-pll_bypss` | `4a0f3c41` | 第二代中期 | 与下面两个分支的 PCIe blobs 完全相同。已有三阶段新配置包、external init、PLL bypass、ready magic；仍保留 `pcie_ep_clear_ibatu()`，无双 eFuse backup/GPIO93 新时序。分支主题是 SPI/PLL，不是独立 PCIe 方案。 |
| `20-spi_master-pll_bypss-2` | `83f33146` | 第二代中期 | PCIe 内容 hash 与上一分支完全相同。 |
| `22-wdi` | `83f33146` | 第二代中期 | 与 `20-spi_master-pll_bypss-2` 不只是 PCIe 相同，tip commit 本身也相同；它是同一提交的另一个远端名字。 |
| `boot_form_nand` | `ff302f87` | 第一代 | 和 `v0.86r` 使用相同 `pcie_main.c`：Cadence object API、LTSSM 文本解码、`setupIbEPAddrTranslation()`、早期参数布局、SR-IOV/ARI wrapper 设置。Host helper driver 绑定 `10ee:8014`，明显带参考设计血统。 |
| `eda_d2d` | `3c0992f2` | 第二代早期 | PCIe main 与 `v1.0r_pre3` 相同：已经移除生成 driver，仍用早期参数解析；不再设置 SR-IOV/ARI；Host helper 已改为 `17cd:0100`。分支主题 D2D，对 PCIe 更像一个版本快照。 |
| `master` | `9357f731` | 第二代成熟 | ROM 内建实现的基准：external init、三阶段配置包、checksum/白名单、冷热复位、PLL bypass、双 eFuse block、GPIO93、ready magic、无超时等待 Link、BAR0→AP SRAM mailbox。GO 时不再清 inbound ATU。 |
| `multi_die-err-test` | `2d8e9df7` | 与 master 相同 | `pcie_main.c`、`pcie_main_ex.c`、`edge_boot.c` blob 均与 master 完全一致。额外的 PCIe 文件命中来自 coverage 报告；PCIe 本身没有独立实现。 |
| `pcie-mkfw` | `573da310` | 第三代最新 | 当前最丰富的外部 PCIe firmware：X2000/X5000/X6000/VS/LM/DS，Group A/B BAR，X6000 APT，MPS/order/NP tuning，版本/SN/reset history，多 die 状态；tip 还含 FLR IRQ 36、MSI-X table/PBA 与向量表实验。主线研究 X6000 应优先看这个分支，但不能把 `IRQ_TEST` 自动当成量产设计。 |
| `pcie-mkfw-p2p` | `d66e76b4` | 第三代早期 | 以 X2000/X5000 为主；增加 P2P 支持，并新增 BAR0 inbound ATU，使 BootROM 阶段可经 BAR0 访问 eFuse。它确立了后来“AP SRAM aperture + APT 到 system/eFuse”的方向。 |
| `pcie-mkfw-x2000` | `14684a49` | 第三代中期 | 加入 X6000A/X6000B 条件代码，但分支目标仍以 X2000 BAR size optimization 为名；为各芯片写不同 BAR/区域配置。它是 `pcie-mkfw` 继续扩展的主要祖先之一。 |
| `pcie_init` | `172b5ff0` | 第二代成熟前夜 | 和 master 只差少量硬件时序：master 后来拆分 PMU set 写并加入多个 1–20 μs delay，同时删除 `pcie_ep_clear_ibatu()`。tip 提交主题 `pcs-pcie.bin`，说明它用于生成/交付 PCS 所需 PCIe 外部镜像，是研究 BootROM→PCS 交接的重要快照。 |
| `pcie_second_boot` | `5f873d23` | 第二代中期 | 支持通过 mailbox 下载并 GO 到二级镜像；在 `process_command(GO)` 中仅当当前介质为 PCIe 时先调用 `pcie_ep_clear_ibatu()`。这明确规定下一阶段接管前清掉 ROM 的临时 BAR0 inbound 映射。master 后来取消该清理，交接契约发生变化。 |
| `tapeout_asic` | `bbb44b7a` | 第二代早中期 | 已采用新 `pcie_load_firmware()`，但没有 ready magic、external init、PLL bypass、双 eFuse backup/GPIO93；Link 仍有超时返回；保留 clear inbound ATU。它接近 ASIC tapeout 时的一次较保守快照。 |
| `test_usb` | `ae298084` | 第二代早期 | PCIe 主体虽已使用新配置包，但仍缺后来的 external init/ready magic 等；Host helper 是 `pcie_boot.c` + `17cd:0100`。分支重点是 USB，PCIe 只是当时公共主线状态。 |
| `v0.86r` | `323d509c` | 第一代 | PCIe main 与 `boot_form_nand` 完全相同，包含整套 Cadence generated API；是当前远端可见的最早一代实现代表。 |
| `v0.95r` | `a5f4ce6f` | 第二代早期 | generated driver 已从主调用路径移除，直接寄存器初始化；使用早期参数格式；显式配置 Gen/lane、SR-IOV/ARI；用 LTSSM state 判断 L0；保留 clear inbound ATU。 |
| `v0.95r_patch1` | `27ea124b` | 第二代早期修订 | `pcie_main.c` 与 `v1.0r_pre2` blob 相同；相较 v0.95 系列已取消 SR-IOV/ARI 显式设置，反映 wrapper 默认策略调整。仍是早期参数布局与有超时 Link 等待。 |
| `v0.95r_pre` | `a396175a` | 第二代早期 | 与 `v0.95r` 同一阶段但 blob 不同；保留 SR-IOV/ARI 设置和早期参数格式，可用于定位 2021-09 初到 release 的寄存器更新。 |
| `v1.0r_pre2` | `65ad539d` | 第二代早期修订 | PCIe main 与 `v0.95r_patch1` 完全相同；分支主要更新寄存器头版本，PCIe 行为没有形成新的独立族。 |
| `v1.0r_pre3` | `a4f3d54c` | 第二代早期修订 | PCIe main 与 `eda_d2d` 完全相同，Host helper 也同为 `17cd:0100` 版本。分支主题是 interrupt register 更新，不是新的 PCIe boot 协议。 |
| `x6000_pcie_boot` | `b20e4816` | 第三代过渡 | 名字写着 X6000，但 tip 的 `edge10/Makefile` 只识别 `BOARD=X2000/X5000`，`pcie_main_ex.c` 也没有 `IPU_X6000*` 条件。也就是说这个 tip **并不能单凭 `BOARD=X6000` 构建出专属配置**；它更像把 X6000 相关提交带到独立分支的过渡点。完整可构建的 X6000 选择出现在后来的 `pcie-mkfw`。 |

## 4. 核心 blob 聚类证据

下表列出分支 tip 中关键文件的 Git blob 前 10 位。相同 blob 意味着文件逐字节相同。

| 分支组 | `pcie_main.c` | `pcie_main_ex.c` | Host helper |
|---|---|---|---|
| `20-spi_master-pll_bypss`, `-2`, `22-wdi` | `a5f62aab28` | `3da2677600` | `edge_boot.c ac9e5a3fae` |
| `boot_form_nand`, `v0.86r` | `52fe90575c` | 不存在 | `pcie_boot.c fb5370fb38` |
| `eda_d2d`, `v1.0r_pre3` | `ddb011d9dc` | 不存在 | `pcie_boot.c 121c62aff3` |
| `v0.95r_patch1`, `v1.0r_pre2` | `d085788a7d` | 不存在 | `pcie_boot.c fb5370fb38` |
| `master`, `multi_die-err-test` | `fb9dc75e43` | `78337eb2ec` | `edge_boot.c 5f7b1598d6` |

其余分支的 `pcie_main.c` blob 独立，但仍可按上文三代架构理解。

## 5. 关键行为演进表

缩写：CAPI=Cadence generated API；OldParam=早期参数格式；NewFW=带三 stage offset/checksum 的新格式；Magic=写 `0x6007fa40` ready magic；Ext=从 NOR 加载可执行 PCIe firmware；ClearIB=GO 前/接口中可清 BAR0 inbound ATU。

| 分支族 | CAPI | OldParam | NewFW | Magic | Ext | ClearIB | SR-IOV/ARI |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `boot_form_nand` / `v0.86r` | ✓ | ✓ |  |  |  | ✓ | ✓ |
| `v0.95r` / `v0.95r_pre` |  | ✓ |  |  |  | ✓ | ✓ |
| `v0.95r_patch1` / `v1.0r_pre2` |  | ✓ |  |  |  | ✓ |  |
| `eda_d2d` / `v1.0r_pre3` |  | ✓ |  |  |  | ✓ |  |
| `tapeout_asic` / `test_usb` |  |  | ✓ |  |  | ✓ |  |
| `20-*` / `22-wdi` |  |  | ✓ | ✓ | ✓ | ✓ |  |
| `pcie_init` |  |  | ✓ | ✓ | ✓ | ✓ |  |
| `master` / `multi_die` |  |  | ✓ | ✓ | ✓ |  |  |
| mkfw 系列 | 基础 ROM 同 master；主要差异位于 external executable firmware |

## 6. mkfw 分支的真正演进

根据 PCIe 路径的提交历史：

```text
2023-06  external EP init
2023-09  X2000 firmware
2023-10  BootROM APT remap
2023-11  BAR0 inbound ATU
2023-12  disable relaxed ordering
2024-01  BAR0 non-prefetchable, fw V1.0.1
2024-04  NP outstanding tuning for MRRS=512
2024-06  P2P + eFuse-visible BAR0 inbound
2024-07  initial X6000 configure
2024-09  X6000A/B BAR layouts, B BAR4, B MPS=1024
2024-11  X6000 video structuring
2025-01  merge X6000 group A/B binary, LM support
2025-03  X6000 DS
2025-04  X6000 MPS=512 revision, DS BAR merge
2025-08  X6000/DS ordering and NP outstanding
2025-09  VS/DS merge and firmware update
2025-10  firmware version copied to AP SRAM
2025-11  BootROM interrupt response
2026-06  virtualization/FLR experiment
```

这里有一个看似矛盾的版本轨迹：2024 曾为 X6000B 设置 MPS=1024，2025 commit message 又说“all X6000 MPS=512”，但当前 tip 源码写入的编码仍为 `3`（按 PCIe Device Control MPS 编码对应 1024 B）。这可能是后续又改回、注释/commit message 不同步，或寄存器并非标准 Device Control 语义。当前结论标为 `NEED_VERIFY`，需要对照该 controller PF config register 定义与构建产物反汇编。

## 7. 哪个分支最值得读

若目标是理解代码演进，建议顺序：

1. `v0.86r`：看 Cadence API 原型和完整 LTSSM 名称；
2. `v0.95r`：看为何改成直接寄存器；
3. `pcie_second_boot`：理解 GO 与 ATU 交接；
4. `master`：理解成熟的 ROM 固定流程；
5. `pcie-mkfw-p2p`：理解 Boot 阶段 APT/eFuse 映射的起点；
6. `pcie-mkfw`：理解 X6000 产品化配置。

若目标只是当前 X6000 开发，不必按 21 个分支逐个通读全部相同代码。先读 `master` + `pcie-mkfw`，再用本矩阵定位历史原因，更有效率。

## 8. 静态审计发现

### 8.1 Host utility 与 helper driver ABI 不匹配

`bootrom_utility_tool/pcie.c` 用 ioctl 2/3；tip `edge_boot.c` 只实现 `read/write/mmap`。需要寻找部署时真实配对版本。

### 8.2 Host helper 的错误码检查写错变量

代码调用 `pci_request_regions(pdev, DRV_NAME);` 后没有接收返回值，却检查旧的 `rc`。资源申请失败可能继续执行，或报告错误路径不可信。

### 8.3 mailbox 没有长度边界

Host helper 按 userspace `count` 直接复制到 BAR0 mailbox；BootROM 又信任协议长度字段。两端都缺少与固定 buffer 容量的严格交叉检查。这在可信工厂下载环境可能被接受，但不应当作通用安全接口。

### 8.4 Link 等待没有超时机制的设计代价

master 永久等待 Link 适合“Host 一定会来”的 PCIe boot，却会让硬件故障表现为永远卡住。量产诊断需要独立 watchdog/reset-source 证据，否则“Host 没插”和“PHY 坏了”看起来完全一样。

### 8.5 最新 `IRQ_TEST` 名副其实

`pcie-mkfw` tip 把 `IRQ_TEST` 直接定义在源码里，使用硬编码 VTOR、stack、IRQ 36、MSI-X table/PBA 地址。它需要和启动入口、链接脚本、实际芯片 revision 一起验证，不能孤立复用。

## 9. 跨仓研究后的问题收口

- 已解决：PCS保留Host-facing Link并重写APT 0..8；BL2增量加入region9/uDMA。
- 已解决：AP SRAM/mailbox从BootROM跨PCS/BL2保留，Linux最终DTS只更新选定region。
- 仍开放：产品rootfs究竟配哪代boot helper/ABI（OQ-015）。
- 仍开放：`pcie-mkfw` MPS注释/commit message/写值冲突（OQ-016）。
- 仍需量产构建或实机确认：最新FLR/MSI-X `IRQ_TEST`是否实际启用。

所有可由六仓静态源码解决的问题已经关闭，余项同步到`OPEN_QUESTIONS.md`。

## 10. Source Reference

- 分支与 tip：`git for-each-ref refs/remotes/origin`，采集日期 2026-08-27
- blob 比较：`git rev-parse origin/<branch>:<path>`
- PCIe 历史：`git log --all -- edge10/device/pcie pcie_boot_driver`
- 每个分支源文件：
  - `origin/<branch>:edge10/device/pcie/pcie_main.c`
  - `origin/<branch>:edge10/device/pcie/pcie_main_ex.c`（若存在）
  - `origin/<branch>:pcie_boot_driver/{pcie_boot,edge_boot}.c`
  - `origin/<branch>:edge10/main.c`
  - `origin/<branch>:edge10/device/protol/protol_processing.c`
- 深入流程：[bootrom_pcie_boot_flow.md](bootrom_pcie_boot_flow.md)
