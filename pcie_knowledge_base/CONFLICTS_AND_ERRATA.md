# PCIe 资料冲突与文档勘误账本

这份文件专门收容那些“两个证据都摆在桌上，却讲了不同故事”的问题。

它不是 TODO 垃圾桶，也不是把判断责任推给读者。每项冲突都必须给出当前最佳裁决；
证据仍不足时，明确采用哪一个保守模型继续研究，并说明未来怎样验证。研究过程中不因
这类冲突打断用户，除非继续操作会造成不可逆的外部影响。

## 记录规则

每项至少维护：

- 冲突双方及精确来源；
- 冲突属于 Spec/Databook、源码/注释、分支、构建产物还是本文勘误；
- 当前裁决及证据等级：**Confirmed / Inferred / Conflict / Unknown**；
- 影响哪些知识文档、代码路径和调试结论；
- 能否由当前资料解决；若不能，需要什么实机或额外资料；
- 状态：`Open`、`Provisional`、`Resolved`、`Document Fix Pending`。

处理优先级通常是：

```text
最终构建产物 / 最终DTB / 实机行为
  > 产品级DTS与当前分支实际源码
  > 公共DTSI / 配置模板
  > 代码注释、commit message和旧分支
```

PCIe Spec 与芯片实现手册的关系不能简单套这条顺序：Spec 决定线上合规语义，Databook
决定当前 IP 如何实现；实现限制可以比 Spec 更窄，却不能把不合规行为说成标准要求。

## CE-001：公共 Ultra DTSI 与最终产品 DTS 的 `apt-ranges` 不同

- 状态：`Document Fix Pending`
- 严重度：高
- 类型：最终 DTS 覆盖公共 DTSI；已有知识文档勘误

### 冲突证据

公共模板：

`source_repos/linux/arch/arm64/boot/dts/intellif/hsio/edge10_ultra_hsio_pcie_card.dtsi`

其中定义 BAR0 region 0–6/11–14/30 和 BAR2 region 21–24 等 16 条映射。例如：

```text
BAR0+0x300000 → 0x58400000, 1 MiB
BAR0+0x400000 → 0x58600000, 1 MiB
```

最终产品 DTS，例如：

`source_repos/linux/arch/arm64/boot/dts/intellif/edge10_ultra/edge10_ultra_a_die0_a55_prod_unconverted.dts`

在 `&pcie0` 中重新定义整个 `apt-ranges` property：

```text
region 9: BAR0+0x400000 → 0x58400000, size 0x210000
region 13..16: BAR0+0x800000..0xe00000 → four group windows
region 28..31: BAR2 four × 16 MiB
region 10: BAR4 → A8/B0 window（部分产品/位置）
```

DTS property 覆盖不是数组追加，因此最终 DTB 采用产品 DTS 的 property，而不是把两张
表拼接起来。

### 当前裁决

**Confirmed**：分析运行期地址图必须以具体产品最终 DTS/DTB 为准；公共 DTSI 只能
说明模板和默认能力。

同时，Linux EPC `edge_pcie_ep_set_bar()` 只写 `apt-ranges` 中列出的 region，并未先
清空全部 APT。因此未列出的 mailbox/AP SRAM 等 region 可能继续继承 PCS/BL2 的硬件
状态。这一点目前为 **Inferred**，需要最终 DTB、启动日志和 APT register dump 验证。

### 影响与勘误

下列文档若把公共 DTSI 16 条表直接称作“X6000 最终 runtime 地址图”，属于待修正文：

- `code_studies/linux_x6000_pcie_stack.md`
- `code_studies/x6000_address_map.md`
- `code_studies/x6000_boot_handoff_and_ownership.md`
- `code_studies/x6000_integrated_pcie_architecture.md`

修正模型应写成“固件保留 region + 产品 DTS 选择性覆盖”，并分开列普通、LM、DS、
Group A/B 的最终 property。

## CE-002：Host 驱动常量名称与实际 BAR/APT 目标容易产生误读

- 状态：`Provisional`
- 严重度：高
- 类型：源码命名、地址路径与文档解释冲突

### 冲突证据

Host edge-driver：

`source_repos/edge-driver/pcie/edge.h`

```c
#define PCIE_CONTROLLER_X8_BASE 0x400000
#define REG_UDMA_ADDR (PCIE_CONTROLLER_X8_BASE + 0x200000)     /* 0x600000 */
#define REG_AXI_CFG_ADDR (PCIE_CONTROLLER_X8_BASE + 0x400000) /* 0x800000 */
```

最终产品 DTS region 9 使 `BAR0+0x400000..0x60ffff` 线性映射到
`0x58400000..0x5860ffff`，因此 `BAR0+0x600000` 正好到达本地 uDMA。

但 `BAR0+0x800000` 在产品 DTS 中命中 group window（例如 Group A 的
`0x2a0000000`），并非公共 DTSI 所暗示的本地 controller region。Host驱动在这里用
`struct atu_regs` 编程 outbound region，说明它访问的是经过group/远端窗口暴露的
controller address，而不是简单的`0x58400000 + offset`。

### 当前裁决

- `BAR0+0x600000 → local 0x58600000 uDMA`：**Confirmed**。
- `BAR0+0x800000` 的精确 A0/A8/Group owner：当前为 **Inferred**；必须结合产品最终
  DTS、`A8_OUTBOUND_BAR0_ADDR_BASE`、group ID 与实机 APT dump描述，不能仅凭宏名写成
  “本地 ATU”。

### 文档约束

以后写地址路径时同时列出：BAR offset、命中的 APT region、最终 AXI target、执行
该访问的 group/controller。禁止只依据 `REG_AXI_CFG_ADDR` 名字猜硬件归属。

## CE-003：Cadence uDMA interrupt enable 的 reset/写语义互相冲突

- 状态：`Open`
- 严重度：中
- 类型：Databook 内部冲突

Cadence §5.5.1 表述 reset 后中断全部 disabled；§9.6.1.22 表格同时出现 reset=1、
`R/WOCLR` 与“write 1 enables”等相互牵扯的描述。

当前裁决：初始化和恢复代码不得依赖 reset 默认值，应显式 mask、clear、enable并做
readback/IRQ测试。最终需要 Cadence errata、RTL simulation 或实芯片验证。

关联：`OPEN_QUESTIONS.md` OQ-010。

## CE-004：Link Down Reset 是否保留 Local Management Registers

- 状态：`Open`
- 严重度：中
- 类型：Databook 内部冲突

- Cadence §4.7.2：Local Management Registers 保留、无需重编；
- Cadence §11.1.2.3：Local Management Registers 被 reset。

当前裁决：恢复代码按“不保证保留”设计；在当前 RTL build 上选择安全的非默认 LM
字段做 Link Down 前后 readback，取得实机证据后再缩短恢复路径。

关联：`OPEN_QUESTIONS.md` OQ-011。

## CE-005：BootROM `pcie-mkfw` 的 MPS 演进记录与 tip 源码不一致

- 状态：`Open`
- 严重度：中
- 类型：commit message / 注释与当前源码冲突

2024 记录曾为 X6000B 设置 MPS=1024；2025 commit message 又称“all X6000 MPS=512”，
但当前 tip 写入编码仍为 `3`，若对应标准 Device Control MPS 编码就是 1024 B。

当前裁决：不依据 commit message 宣称量产 MPS。需要核对寄存器定义、最终构建产物
反汇编和 Host `lspci -vv` negotiated MPS。

关联：`code_studies/bootrom_pcie_branch_matrix.md`。

## CE-006：edge10_tool 主仓 gitlink 与实际构建使用的 HAL 版本不同

- 状态：`Resolved as Build Behavior`
- 严重度：高
- 类型：源码版本/构建可复现性冲突

- 主仓 `x6000 @ 1daae3e4...` 的 HAL gitlink 指向旧 `8ed32a3b...`；
- `build.sh` 会把 HAL 动态 checkout/pull 到工具当前同名分支；
- 本轮实际 `HAL x6000 @ aa005c36...` 才与当前 ioctl ABI 对照。

当前裁决：研究和发布记录必须同时固定主仓与 HAL 两个 commit。若要可复现构建，应避免
构建过程静默跟随远端可移动分支，或至少把解析后的 HAL commit写入产物 metadata。

## CE-007：BootROM utility ioctl 与同仓 `edge_boot.c` file operations 不匹配

- 状态：`Provisional`
- 严重度：中
- 类型：同仓不同代 Host 工具/驱动 ABI 冲突

`bootrom_utility_tool/pcie.c` 调用 ioctl 2/3；当前 tip `edge_boot.c` 的
`file_operations` 只有 `.read/.write`，没有 `.unlocked_ioctl`。

当前裁决：两者不是可直接配对运行的同代 ABI。需要按分支/发布包寻找实际配套 Host
driver，不能把同仓路径邻近误当版本兼容证据。

关联：`code_studies/bootrom_pcie_boot_flow.md`。

## 新冲突登记模板

```markdown
## CE-NNN：标题

- 状态：Open / Provisional / Resolved / Document Fix Pending
- 严重度：高 / 中 / 低
- 类型：Spec↔Databook / 代码↔注释 / 分支 / 构建 / 文档勘误

### 冲突证据

### 当前裁决

### 影响范围

### 验证与关闭条件
```

