# Stage 5 Review：Cadence Register / Programming / Integration

日期：2026-08-23

## Review 目标

本轮不是把 Chapter 9 的 900 多页寄存器换一种排版复制，而是检查每个硬件职责是否已经回答四个问题：

1. 寄存器属于哪个访问空间，谁能碰到；
2. 它对应什么协议/硬件状态机；
3. cold boot、training、enumeration、runtime、recovery 中何时使用；
4. reset、W1C、clock shutoff、reserved access 会造成什么软件后果。

## 1. Chapter 9 register block coverage

| Register block | 当前知识落点 | Review 结果 |
|---|---|---|
| PF/VF Configuration | configuration、BAR、MSI/AER、SR-IOV/ATS 专题 | 已覆盖标准空间与本地 APB双入口；optional capability 以 H.1 当前列表为准 |
| Root Port Configuration | enumeration/host controller/error/power | 已覆盖 Type 1、bus/window、Root/AER/Link 状态；目标 glue offset 缺资料 |
| Local Management | controller/link/error/power/debug | 已覆盖 LTSSM、credit、BAR aperture、error/mask、PM/equalization 等代表性状态机 |
| AXI Configuration | BAR/address translation/transaction | 已覆盖 region base/size、translation、descriptor、Config/Memory/I/O/Message TLP |
| PCIe DMA | uDMA 专题 | 已覆盖 capability discovery、descriptor、channel、ISR/error/recovery；OQ-010 保留中断 enable 语义冲突 |
| LINK0–3 replication | multifurcation/register-space 专题 | 已覆盖每 Link 独立 core/APB 与 9.1–9.24 对应关系 |

结论：寄存器空间的职责与关键路径已覆盖；没有必要逐个复制 register table。后续若有目标驱动，应补“symbol → offset → write timing → reset domain”，而不是再造一本无代码引用的寄存器手册。

## 2. Chapter 10 programming-guide coverage

- Gen3/Gen4 RP preset 必须在 training 前写有效值：已进入 register programming 和 LTSSM 文档；
- outbound Config 两种 BDF 编码、4 KiB/256 MiB aperture：已进入 address translation 文档；
- Memory/I/O/Message descriptor 与 region：已进入 address/TLP专题；
- Endpoint autonomous speed/width management：已进入 link 文档；
- SR-IOV VF number/BAR/System Page Size：已进入 ARI/SR-IOV专题；
- Lane Margining 参数、ready、error log：已进入 link/error边界。

Chapter 10 的示例已经全部按主题评估，没有发现尚未归档的核心编程流程。

## 3. Chapter 11 integration-guide 新发现

### 3.1 Completion buffer 不是“64 entries 所以高枕无忧”

§11.4 显示 64 笔 MRRS=512 B、非 64 B 对齐的 read 可产生 576 个 split CplD，而 Stage 1 只能保存 128 个。已新增 [Completion buffers 与 backpressure](../controller/cadence_completion_buffers_and_backpressure.md)，解释 two-stage buffer、ordering/AXI backpressure 与 outstanding limit。

### 3.2 Configuration Snoop 不能替代完整 Capability

§11.3 允许客户逻辑观察 Config Read/Write，并在严格的一到两个 cycle 内提供扩展 read data。但手册明确说不能把完整 capability structure 搬到 core 外部，因为 capability 控制字段还要扇出到内部各层执行规范行为。

工程结论：Configuration Snoop 适合少量 vendor-specific register，不是“软件返回任意 config data 就算硬件支持 ATS/PTM/ACS”的后门。

### 3.3 RAM integration 是 SoC integrator 的硬约束

§11.2 给出 PNP、Replay、Completion、split table、AXI reorder、uDMA RAM 的 depth/width 和 read latency。RTL/SoC integration 必须满足 read-after-write、registered interface 与跨 clock 约束。Linux Function driver通常不直接配置这些 RAM，但错误集成会以 parity/ECC、replay、overflow、timeout 等形式冒到软件层。

## 4. 发现一处必须保留的手册冲突

Cadence §4.7.2 明确说 Link Down Reset **不清 Local Management Registers**，其值保持且无需重编；§11.1.2.3 却说除 sticky configuration 外，“all other configuration registers and Local Management registers are reset”。两处对同一事件给出相反结论。

因此早期文档中“LM一定保留”的绝对表述已纠正为 Conflict：

- AXI Address Translation 保留在 §4.7.2 有直接、一致证据；
- Configuration 非 sticky reset 两处一致；
- Local Management 是否保留记为 OQ-011；
- 在没有 errata/RTL/实机验证时，恢复代码应 readback 校验关键 LM，并在安全时重写，而不能依赖任一段文字下注。

## 5. Access/reset attribute review

- Chapter 9 地址为 DWord；本地 APB reserved read=0/write ignored，Link config reserved access=Cpl/UR；
- core clock shutoff 时 APB read 可能返回 dummy data，write可能无效，是否 PSLVERR 由 feature bit 决定；
- status 常见 `R/WOCLR`/W1C，读取日志后再清，不能用普通 read-modify-write 模板误清邻位；
- strap/capability选择必须在 `RESET_N`、`MGMT_RESET_N`、`MGMT_STICKY_RESET_N` release 前稳定；
- LM programming 在 reset release 后、`LINK_TRAINING_ENABLE=0` 时完成；
- Link Down Indication 必须等 `LINK_DOWN_RESET_OUT` falling edge 后清，再恢复 AXI traffic。

## 6. Stage 5 结论

Stage 5 在 **Cadence core architecture / programming / representative register semantics** 层面完成。仍未关闭的项目都是证据边界，而非遗漏阅读：

- OQ-003：实际 RTL build ID/configuration report；
- OQ-010：uDMA interrupt enable 手册矛盾；
- OQ-011：Link Down Reset 是否清 LM 的手册矛盾；
- OQ-002/OQ-004：SoC wrapper/PHY/physical address/clock/reset glue；
- 目标 kernel/DTS 缺失导致 Linux symbol↔offset mapping unavailable。

## 7. Source Reference

- Cadence PCIe Core User Guide Chapter 9 opening/Table 9.1，printed pp.322–323 / PDF pp.411–412。
- Chapter 10，printed pp.1269–1283 / PDF pp.1358–1372。
- §§11.1–11.4，printed pp.1284–1317 / PDF pp.1373–1406。
- §4.7.2 printed pp.276–277 / PDF pp.365–366 与 §11.1.2.3 printed p.1290 / PDF p.1379：LM reset conflict。

