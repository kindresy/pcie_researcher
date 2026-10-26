# X6000 PCIe Source Repository Index

采集日期：2026-08-27

这份索引固定本轮源码证据。网页上的 `x6000` 会继续移动，commit不会；后续文档一律引用这里的commit，而不是只写一个会漂移的分支名。

## 1. Repository baseline

| Repository | 研究分支 | 固定commit | 规模 | PCIe研究范围 |
|---|---|---|---:|---|
| `edge-driver` | `x6000` | `f27ec1192d2089bf80898677a20fb5ac714f6f8f` | 19 files | Host Linux加载的PCIe设备驱动：`pcie/edge.[ch]`；`pcie_cascade/`作为相关变体单独评估 |
| `linux` | `x6000` | `e74fd0e7746a834a5cbbbdbc1c36204f4a26a6a3` | 74,314 files | Linux 5.10.4；只读PCI core、Endpoint Framework、Edge/Cadence controller、相关DTS、DMA/MSI/AER/IOMMU路径 |
| `bootrom` | all remote branches | `master=9357f731414ebad5824661c93a38f58f076a7c56` | 355 files on master | 每个远端分支的BootROM PCIe init/download/second-boot差异 |
| `pcs` | `x6000` | `80d6faf2613d1b04e0166c9a258017a0b60588dc` | 415 files | `board/edge10/drivers/pcie/`、reset/system-control registers、main/boot handoff |
| `atf` | `x6000` | `98ac1eb5984ad69fc42860abbc9ed70346f240a8` | 4,896 files | `drivers/intellif/pcie/`、platform setup、IRQ relay、security/firewall/handoff |
| `edge10_tool` | `x6000` | `1daae3e45e22b87c1f847be19a3209a770bd321b` | 223 files（不含子模块） | `src/host/cases/pcie/`、Host下载编排与测试命令 |
| `edge10_tool/third_party/hal` | build实际切换到`x6000` | `aa005c36b477033c26324368d88c69b388b203a4` | 独立HAL仓库 | Host PCIe API、ioctl ABI、下载状态机；主仓gitlink为旧`8ed32a3b...`，但`build.sh`会忽略它并跟随同名分支 |

Remote base：内部 GitLab 的 `deepedge10/<repository>.git` 命名空间。

> 公开版说明：内网主机地址未写入知识库；仓库名、分支和固定 commit 保留，便于在具备内部访问权限的环境中回溯证据。

## 2. Initial source boundaries

### edge-driver

- `pcie/edge.c` 6658行、`pcie/edge.h` 856行：X6000 Host function driver主实现；
- `pcie_cascade/edge.c` 4820行、`edge.h` 654行：cascade变体；
- `ch9344/`与`usb/`不属于PCIe主线，只有出现共享ABI/测试依赖时才回看。

初扫已确认主驱动包含：probe/remove、BAR mapping、char device/ioctl/mmap、Cadence uDMA descriptor/queue、P2P region、MSI/MSI-X/INTx、notify shared-memory IRQ、AER/reset recovery、boot-stage magic和固件传输。

### linux x6000

Vendor PCIe实现集中在：

```text
drivers/pci/controller/pcie-edge*.{c,h}
drivers/pci/controller/pcie-cascade-edge*.{c,h}
drivers/pci/controller/edge-udma.c
drivers/pci/controller/pcie_irq_relay.c
drivers/pci/endpoint/functions/pci-epf-edge.c
arch/arm64/boot/dts/intellif/hsio/edge10_ultra_hsio_pcie_card.dtsi
arch/arm64/boot/dts/intellif/edge10_ultra/*pcie* / *prod*.dts
```

这棵5.10.4 vendor tree把X6000 Cadence实现直接放在`drivers/pci/controller/pcie-edge*`，
不是当前上游常见的`drivers/pci/controller/cadence/`目录布局，也不是DesignWare。
通用调用链限定在它实际调用到的`drivers/pci/`、Endpoint Framework、DMA/MSI/AER和
IOMMU文件，不把整个kernel纳入范围。

### PCS / ATF

PCS主路径：`board/edge10/drivers/pcie/{pcie_init,pcie_fw,pcie_pm,vf}.*`。

ATF主路径：`drivers/intellif/pcie/{pcie_init,pcie,c2c,pcie_irq_relay,pcie_irq_misc}.*`，并追到 `plat/intellif/edge10/` 与对应firewall/address-map headers。

### edge10_tool

Host cases包括 BAR I/O、bandwidth monitor、hot reset、mkfw、NOR、remote command、register read、scan、DMA/stress、shared memory与debug。Device cases包含 DMA、exception、LTSSM、firmware/version等对端逻辑。二者必须与 `edge-driver` ioctl/header逐项对照。

实际构建前 `build.sh` 会把 HAL checkout/pull 到工具当前分支名，因此本轮以 HAL `x6000 @ aa005c36...` 为证据，而不是子模块gitlink指向的2023年通用版本。该行为已经作为构建可复现性风险记录。

## 3. BootROM remote branches

| Branch | Tip commit | 初始PCIe源码形态 |
|---|---|---|
| `20-spi_master-pll_bypss` | `4a0f3c41...` | compact `pcie_main*` + `edge_boot` |
| `20-spi_master-pll_bypss-2` | `83f33146...` | compact `pcie_main*` + `edge_boot` |
| `22-wdi` | `83f33146...` | 与上一分支同一commit |
| `boot_form_nand` | `ff302f87...` | generated Cadence core driver/API + `pcie_boot` |
| `eda_d2d` | `3c0992f2...` | compact `pcie_main` + `pcie_boot` |
| `master` | `9357f731...` | compact `pcie_main*` + `edge_boot` |
| `multi_die-err-test` | `2d8e9df7...` | compact实现 + coverage artifacts + `edge_boot` |
| `pcie-mkfw` | `573da310...` | compact实现 + `edge_boot` |
| `pcie-mkfw-p2p` | `d66e76b4...` | compact实现 + `edge_boot` |
| `pcie-mkfw-x2000` | `14684a49...` | compact实现 + `edge_boot` |
| `pcie_init` | `172b5ff0...` | compact实现 + `edge_boot` |
| `pcie_second_boot` | `5f873d23...` | compact实现 + `edge_boot` |
| `tapeout_asic` | `bbb44b7a...` | compact `pcie_main` + `edge_boot` |
| `test_usb` | `ae298084...` | compact `pcie_main` + `pcie_boot` |
| `v0.86r` | `323d509c...` | generated Cadence core driver/API + `pcie_boot` |
| `v0.95r` | `a5f4ce6f...` | compact `pcie_main` + `pcie_boot` |
| `v0.95r_patch1` | `27ea124b...` | compact `pcie_main` + `pcie_boot` |
| `v0.95r_pre` | `a396175a...` | compact `pcie_main` + `pcie_boot` |
| `v1.0r_pre2` | `65ad539d...` | compact `pcie_main` + `pcie_boot` |
| `v1.0r_pre3` | `a4f3d54c...` | compact `pcie_main` + `pcie_boot` |
| `x6000_pcie_boot` | `b20e4816...` | compact `pcie_main*` + `edge_boot` |

“源码形态”这里只是路径级inventory，不代表功能已经比较完成。真正的分支结论必须继续比较函数、常量、寄存器写序和build flags；相同路径名不保证相同行为。

## 4. Evidence labels

- **Confirmed**：源码行为能由具体语句和权威Spec/Databook直接支持；
- **Inferred**：跨文件/跨阶段推导，文档必须写推导链；
- **Conflict**：仓库、分支、header或Databook互相冲突；
- **Unknown**：缺硬件资料、runtime dump或调用者，不能靠函数名补剧情。
