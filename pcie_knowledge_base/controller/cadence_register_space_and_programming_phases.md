# Cadence 寄存器空间：APB 后的四类寄存器区域

## 1. 为什么先画地图，不能直接抄寄存器表

Cadence Chapter 9 超过 900 个 printed pages，因为 LINK0/LINK1 的功能丰富，PF、VF、Root Port、Local Management、AXI translation 和 uDMA 又各有一套寄存器。逐项抄表只会得到一本更难搜索的手册。

真正有用的问题是：

> 软件这次访问的是 PCIe 标准配置状态，还是 Cadence 私有控制状态？它会不会被链路另一端看到？地址单位是 byte 还是 DWord？写错以后是 UR、APB error，还是悄悄什么也没发生？

## 2. 入口：每个 Link 都有本地 APB 管理接口

Cadence 为每个 `LINK{N}` 暴露 32-bit APB v1.0 管理接口。本地处理器可以借它访问：

- PF/VF 或 Root Port Configuration Register Set；
- Local Management registers；
- AXI Configuration registers；
- PCIe DMA registers。

这条 APB 是 **SoC 内部管理路径**，不是 PCIe Link。一次 APB read 不会自动变成 Configuration Read TLP；它只是本地固件从侧门查看 core 内部寄存器。另一方面，远端发来的 Configuration Request 只能到协议允许暴露的 Function Configuration Space，进不了 Cadence Local Management 空间。

## 3. APB 全局地址选择

Cadence Table 9.1 给出本地管理总线的一级译码。寄存器表中的地址是 **DWord address**，而 APB 使用 `ADDR[11:2]` 选择 DWord；换算 byte offset 时通常要乘 4。

| APB 选择字段 | EP/RP 条件 | 选中的空间 | 谁能访问 / 用来干什么 |
|---|---|---|---|
| `ADDR[22]=0, ADDR[20]=0` | EP | PF0 或 VF0–7 Configuration | Link Configuration TLP 可访问相应 Function；本地 APB 也可访问 |
| `ADDR[22]=0, ADDR[21]=0, ADDR[20]=0` | RP | Root Port Configuration，正常写权限 | 只从本地 APB 访问 |
| `ADDR[22]=0, ADDR[21]=1, ADDR[20]=0` | RP | Root Port Configuration，部分通常 RO 字段可写 | 固件初始化模板字段的特殊窗口 |
| `ADDR[22]=0, ADDR[20]=1` | EP/RP | Local Management | Cadence 私有 operational/debug/status/mask |
| `ADDR[22]=1, ADDR[21]=0` | EP/RP | AXI Configuration | outbound/inbound region、descriptor、BAR 到 AXI translation |
| `ADDR[22]=1, ADDR[21]=1` | EP/RP | PCIe DMA | uDMA channel、descriptor pointer、status/interrupt/ECC |

PF/VF Configuration 选择中，`ADDR[19:12]` 选择 Function：本配置 PF0 使用 0，VF0–7 使用 1–8；其余 9–255 保留。章节正文也把同一规则写成 `MGMT_ADDR[18]=0`、`[17:10]=Function`、`[9:0]=register`，这是去掉 APB byte-address 低两位后的 DWord 视图。两套位号不要硬拼成两个不同寄存器，它们是在不同表示层级描述同一译码。

## 4. 四类空间分别代表什么

### 4.1 Configuration Register Set：标准配置身份与控制字段

EP 模式下，每个 PF/VF 有自己的 4 KiB Configuration Space，包含：

- Vendor/Device ID、Class Code、Command/Status、BAR；
- PM、MSI、MSI-X、PCI Express Capability；
- AER、ARI、SR-IOV、ATS/PASID/PRI 等由 build 支持的 Extended Capability。

它有两条访问路径：

```text
远端 RC 发 Configuration Request TLP → Function decode → Config register → Completion
本地固件 APB access              → Function select → 同一 Config register
```

这解释了为什么 EP 固件能在 Link up 前写 Vendor ID、BAR aperture 或 capability 参数，而 Host 枚举后又能通过配置请求读取它们。

RP 模式下使用 Root Port Configuration Register Set。它长得像 Type 1 header，含 Primary/Secondary/Subordinate Bus Number、Memory Base/Limit、Link/Slot/Root/AER 等寄存器，但手册明确说它不能由 Link 上的 Configuration Request 直接访问，只能由本地管理总线操作。

### 4.2 Local Management：控制器内部管理状态

Local Management 在 EP/RP 两种模式都存在，且不暴露给远端。它控制或记录的是实现状态，例如：

- LTSSM/link width/speed/retrain；
- local error status/mask、debug mux、LTSSM history；
- power management handshake 与 clock shutoff；
- BAR aperture/type、VF BAR 配置；
- equalization/margining 参数；
- core feature、polarity inversion 等实现特性。

这类 bit 往往不是 PCIe Spec 定义的 wire-visible 字段，而是 Cadence 为实现协议状态机提供的固件旋钮或观察窗。解释它时必须反向连接到协议概念，不能只写“bit 3 enables feature”。

### 4.3 AXI Configuration：地址与 TLP 类型转换

这个空间配置：

- AXI region base/size；
- outbound AXI→PCIe address translation；
- outbound PCIe descriptor（Memory/IO/Config/Message、Requester 属性等）；
- EP/RP inbound PCIe→AXI translation；
- Function/BAR 到本地 AXI aperture 的映射。

Cadence Chapter 10 给出一个非常重要的事实：同一片 AXI aperture 最终产生什么 TLP，不只由地址决定，还由 region descriptor 决定。访问一个被标成 Config 的 region 会生成 Configuration TLP；标成 Memory 的 region 才生成 Memory Request。

### 4.4 PCIe DMA：内置 uDMA 的控制和状态

这里是 Cadence integrated uDMA 的 channel control、start pointer、status、interrupt 与能力寄存器。它不等于 Linux 通用 DMA API，也不等于所有 Endpoint 的 DMA engine；只有实际产品选择并驱动这块 Cadence uDMA 时才会使用。

## 5. 一个具体例子：RP 发起配置读时经过哪些区域

假设本地 CPU 要读取 `01:00.0` 的 Vendor ID：

1. 固件先经 APB 进入 **AXI Configuration** 空间，建立一个 outbound region，descriptor 类型为 Configuration。
2. BDF 可采用两种编码法：
   - Method 1：Bus/Device/Function 写入 region translation register，AXI 低 12 bit 只携带配置寄存器 offset；region 最小 4 KiB。
   - Method 2：AXI address `[27:12]` 直接携带 BDF，低 12 bit 携带 offset；region 至少 256 MiB。
3. CPU/本地 AXI master 对这个 outbound aperture 发 read。
4. core 的 outbound **AXI Slave** 接收 read，region decode 后生成 Configuration Read Request TLP。
5. EP 返回 Completion with Data，core 再用 AXI read response 把 Vendor ID 交还本地 CPU。

注意，步骤 1 是通过 APB 写控制器寄存器；步骤 3 才是触发 PCIe transaction 的 AXI aperture read。两者若混在一张地址表里，debug 时很容易误用 `setpci` 查找 Cadence region descriptor；但该 descriptor 位于控制器私有寄存器空间，并不属于 PCIe Configuration Space。

## 6. 初始化不应按寄存器地址排序，而应按硬件依赖排序

推荐的编程阶段如下；具体 SoC clock/reset/PHY glue 当前缺资料，因此这里只给 Cadence core 级顺序：

### Phase A：静态模式和 reset 前条件

- 确定 `LANE_MAP_MODE`、RP/EP `MODE_SELECT`、generation/width 等 strap；
- 配好 PHY/refclk/PIPE grouping；
- 保持 Link training 未使能。

### Phase B：Link training 之前必须完成的协议参数

- RP Gen3/Gen4 equalization preset。Cadence 默认值 `1111b` 在 Spec 中无效；未初始化会导致 Gen3/Gen4 training 失败；
- 如启用 Lane Margining，先按 PHY 数据手册写 Local Management 参数；
- PF/Root Port identity、capability、BAR aperture 和实现特性；
- 必要的 error mask、power/clock policy。

### Phase C：地址数据面

- RP outbound Config aperture，供后续枚举；
- RP outbound Memory/IO windows；
- EP inbound BAR→AXI translation；
- 若需要，uDMA descriptor memory 和 channel。

### Phase D：释放 reset 并开始训练

- 遵循 reset/clock 顺序；
- 使能 `LINK_TRAINING_ENABLE`；
- 观察 LTSSM、negotiated speed/width 与 local error。

### Phase E：枚举后再打开 Function 数据通路

- Host 分配 bus number/BAR/resource；
- Function driver 请求资源并 map BAR；
- 设置 Memory Space Enable；
- 只有需要 Endpoint 主动发请求时才设置 Bus Master Enable；
- 最后启用 DMA 与 MSI/MSI-X。

在 Phase B 直接打开 Bus Master Enable 不会促进 Link training，只会让尚未准备好的设备提前获得发起事务的权限。BME 应在 DMA 路径、地址映射和错误处理已经就绪后再启用。

## 7. Reset 与寄存器保存范围

Cadence 的 reset 并非一个按钮清空所有东西：

- `MGMT_RESET_N` 重置 local management 与 configuration registers，但 PCIe 定义为 sticky 的字段例外；
- `MGMT_STICKY_RESET_N` 再负责 sticky bits；
- Link Down Reset 不会清除 AXI Address Translation registers，§4.7.2 明确说它们无需重编。

Local Management 是否保留存在手册内部冲突：§4.7.2 说 LM 不清并保持编程值，§11.1.2.3 却说 Link Down Reset 会重置 LM。没有 errata/RTL/实机证据前，恢复代码应 readback 校验关键 LM 并准备重写；详见 reset matrix 的 OQ-011。这里不能靠挑一句更顺眼的来决定硅片行为。

这直接影响恢复策略：Link Down 后不能假设所有翻译表都归零，也不能假设所有配置状态都保留。恢复代码应该按 reset domain 建 checklist，而不是整段复制 cold boot 初始化。

## 8. 访问失败如何表现

| 访问 | 目标 | 保留/非法地址的行为 |
|---|---|---|
| Link Configuration Request | Function Config Space | core 返回 Completion，status 为 UR |
| 本地 APB read | 保留地址 | 返回全 0 |
| 本地 APB write | 保留地址 | 不修改寄存器 |
| core clock 已关闭时 APB access | 取决于 low-power/control 配置 | 可等待 clock 恢复、产生 `PSLVERR`/dummy data 或给出 `reg_access_clk_shutoff` 指示 |

因此，读到 0 不一定表示寄存器内容确实为 0，也可能是地址选择错误或 core clock 已关闭。驱动在读取关键 ID/status 前，应先确认 APB error 和 clock indication。

## 9. 四个 Link register block 的重复关系

Chapter 9 的顶层结构是：

| Link | PF | VF | Root Port | Local Mgmt | AXI Config | DMA |
|---|---|---|---|---|---|---|
| LINK0 | §9.1 | §9.2 | §9.3 | §9.4 | §9.5 | §9.6 |
| LINK1 | §9.7 | §9.8 | §9.9 | §9.10 | §9.11 | §9.12 |
| LINK2 | §9.13 | §9.14 | §9.15 | §9.16 | §9.17 | §9.18 |
| LINK3 | §9.19 | §9.20 | §9.21 | §9.22 | §9.23 | §9.24 |

LINK2/LINK3 的 LM/AXI/DMA 小节非常短，并不表示这些 Link 没有硬件；它们沿用相应寄存器定义，章节重点只再次给出 build-specific PF/VF/RP map。实际软件仍需要每 Link 的独立 APB base/wrapper 选择，具体 SoC physical address 未提供。

## 10. Linux 对应关系

```text
SoC platform/host controller driver
  ├─ ioremap controller APB register resource
  ├─ 写 Local Management / AXI Config / DMA
  ├─ 提供 pci_ops config read/write
  └─ 建 pci_host_bridge 后触发枚举

PCI core / Function driver
  ├─ 通过 pci_ops 访问标准 Configuration Space
  ├─ 管理 BAR resource、Command、MSI/AER/PM capability
  └─ 不应直接依赖 Cadence 私有 LM 寄存器
```

X6000 Linux tree 已经补齐目标 kernel、DTS 和 SoC glue。`pcie-edge-plat.c` 负责平台资源、PHY、clock/reset 和 RC/EP 分流；`pcie-edge-host.c` 负责 Root Port、配置访问、地址转换与 IRQ domain；`pcie-edge-ep.c` 负责 EPC 的 BAR/APT、outbound mapping 和中断。具体产品运行期使用哪些 `apt-ranges`，必须继续以最终 DTS/DTB 为准，不能只看公共 DTSI。

## 11. Source Reference

- Cadence PCIe Core User Guide, §§3.37–3.40, printed pp.169–173 / PDF pp.258–262：每 Link APB interface。
- Cadence PCIe Core User Guide, §4.8.1, printed pp.278–279 / PDF pp.367–368：core clock shutoff 时 APB 行为。
- Cadence PCIe Core User Guide, Chapter 9 opening and Table 9.1, printed pp.322–323 / PDF pp.411–412：全局 register-space map。
- Cadence PCIe Core User Guide, §§9.1–9.24, printed pp.324–1268 / PDF pp.413–1357：四 Link register blocks。
- Cadence PCIe Core User Guide, §§10.1–10.3, printed pp.1269–1277 / PDF pp.1358–1366：equalization 与 outbound programming examples。
- Cadence PCIe Core User Guide, §11.1.2, reset sequence；§4.7.2, Link Down Reset preservation。
- X6000 Linux 5.10.4：`drivers/pci/controller/pcie-edge-{plat,host,ep}.c`、`pcie-edge.c` 以及 `arch/arm64/boot/dts/intellif/` 下的产品 DTS。
