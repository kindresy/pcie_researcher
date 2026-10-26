# Memory TLP 与 Completion：写请求、读请求及其返回路径

## 1. Transaction Layer 在解决什么

PCIe 允许很多请求同时在途，目标响应时间也各不相同。它不采用“发一个请求就霸住总线等结果”的老办法，而使用 pipelined full split-transaction protocol：Request 先走，Completion 之后独立返回。

这带来较高的并发能力，也要求 TLP 显式携带足够的信息：谁发起请求、目标在哪里、请求多长、哪些 byte 有效、返回对应哪个请求，以及事务是否成功。

Base Spec 5.0 §2.1 把 Transaction Layer 的核心职责概括为 TLP 构造/处理、ordering、Traffic Class/VC 与 credit-based flow control。来源：printed/PDF pp.103–104。

## 2. 四种地址空间、四类事务

| Address Space | Request | 基本用途 | 正常 Completion？ |
|---|---|---|---|
| Memory | Read / Write / AtomicOp | BAR MMIO、Host Memory、DMA | Read/Atomic 需要；Write 不需要 |
| I/O | Read / Write | legacy I/O port | 都需要 |
| Configuration | Read / Write | Function 配置空间 | 都需要 |
| Message | Msg / MsgD | 中断、电源、错误、厂商消息等 in-band event | 不以普通 Completion 应答 |

来源：Base Spec §§2.1.1–2.1.2、2.2.9。

## 3. TLP 的组成

一条 TLP 从前到后可能包含：

```text
[optional TLP Prefix(es)]
[3 DW or 4 DW Header]
[optional Data Payload]
[optional 1 DW TLP Digest / ECRC]
```

所有 TLP header 最先出现 Fmt 和 Type：

- **Fmt** 决定 3DW/4DW、是否带 data；
- **Type** 决定 Memory/Config/Completion/Message 等类型和路由语义。

常用组合：

| TLP | Fmt | Type | Header / Data |
|---|---:|---:|---|
| 32-bit Memory Read | `000b` | `00000b` | 3DW，无 data |
| 64-bit Memory Read | `001b` | `00000b` | 4DW，无 data |
| 32-bit Memory Write | `010b` | `00000b` | 3DW，有 data |
| 64-bit Memory Write | `011b` | `00000b` | 4DW，有 data |
| Completion without Data | `000b` | `01010b` | 3DW，无 data |
| Completion with Data | `010b` | `01010b` | 3DW，有 data |

来源：Base Spec Tables 2-2/2-3，printed/PDF pp.108–110。

## 4. Memory Request header 中真正要盯的字段

### Common fields

- **Length[9:0]**：以 DW（4 B）为单位；编码 0 对有 payload/引用 data 的请求表示 1024 DW。
- **TC**：Traffic Class。
- **Attr**：含 Relaxed Ordering、No Snoop、IDO 等语义。
- **TD**：尾部是否带 ECRC digest。
- **EP**：Error Poisoned，不是 Endpoint 的缩写；同一个缩写换了片场就换了角色。

### Request-specific fields

- **Requester ID**：发请求的 Function 身份，传统格式就是 Bus/Device/Function。
- **Tag**：给 Non-Posted outstanding request 编号，Completion 原样带回。
- **First DW BE / Last DW BE**：第一个和最后一个 DW 哪些 byte 有效。
- **Address**：32-bit request 用 3DW header，64-bit request 用 4DW header。

### Byte Enable 规则

对 Length=1 DW：

- `Last DW BE` 必须为 `0000b`；
- `First DW BE` 可非连续，例如 `0101b`；
- bit 0 对应最低地址 byte，bit 3 对应该 DW 的最高地址 byte。

对多 DW request，首尾有效 byte 通常要与中间完整 DW 组成连续范围；QW-aligned、Length=2 DW 有特别许可。详见 Base Spec §2.2.5，printed/PDF pp.117–119。

## 5. Memory Write：Posted Request

### 5.1 为什么没有正常 Completion

Memory Write 是 Posted：Requester 把 header + payload 发出去，协议正常路径不要求 Completer 回一条 Completion。这降低了延迟和返回流量，很适合 doorbell、descriptor/data write。

但“没有 Completion”不等于“没有任何可靠性”：

- Data Link ACK/NAK 仍保证每一跳 TLP 传输；
- malformed/poison/error 仍可能进入错误上报；
- 设备业务协议可以另设完成状态或 interrupt。

只是不能拿一条普通 Completion 来证明 MWr 已经在最终寄存器生效。

### 5.2 具体例子：CPU 写 EP BAR0+0x100

假设：

```text
EP:                    01:00.0
BAR0 assigned:         0x8000_0000–0x8000_FFFF
CPU operation:         writel(0x12345678, bar0 + 0x100)
PCIe target address:   0x8000_0100
```

对应 32-bit Memory Write Request 的关键字段：

| 字段 | 值 | 原因 |
|---|---|---|
| Fmt / Type | `010b / 00000b` | 3DW Memory Write with data |
| Length | 1 DW | 写 4 B |
| Address | `0x8000_0100` | BAR 基址 + offset；header 保存 DW-aligned address |
| First DW BE | `1111b` | 四个 byte 全有效 |
| Last DW BE | `0000b` | Length=1 时必须为 0 |
| Requester ID | RC 生成请求所使用的身份 | 它不是目标 EP 的 BDF；精确值由 RC 实现/路径决定 |
| Tag | header 中存在，但不用于等待普通 MWr Completion | Posted request 没有对应完成配对 |
| Payload | `0x12345678` | 低地址 byte 先进入数据布局；具体链路序列还经过物理层编码 |

完整硬件路径：

```text
writel
 → CPU virtual __iomem
 → CPU physical MMIO aperture
 → RC outbound AXI/host window
 → 3DW MWr TLP, PCIe address 0x8000_0100
 → Link Layer: Sequence Number + LCRC，ACK/NAK/replay
 → EP BAR0 hit
 → Cadence EP inbound PCIe-to-AXI translation
 → AXI AW/W，WSTRB 对应 First/Last BE
 → device register
```

Cadence §4.5.2 说明 inbound write 的有效 byte 会映射到 256-bit AXI `WSTRB[31:0]`。对于 Posted Write，即使本地 AXI `BRESP` 报错，core 也忽略 response type，不返回 Completion；Non-Posted Write 才会根据 AXI response 形成成功或 CA Completion。来源：printed pp.233–235 / PDF pp.322–324。

这也是 Posted Write 调试困难的根源之一：Requester 端看不到普通 completion status，需要 EP 本地 AXI/error/AER 观察点帮忙。

## 6. Memory Read：Non-Posted Request + Completion with Data

### 6.1 请求

仍读 `BAR0 + 0x100`，执行 `readl()`：

| 字段 | 示例 |
|---|---|
| Fmt / Type | `000b / 00000b`（3DW MRd，无 payload） |
| Length | 1 DW |
| Address | `0x8000_0100` |
| First DW BE | `1111b` |
| Last DW BE | `0000b` |
| Requester ID | RC requester identity |
| Tag | RC 从空闲 outstanding entry 分配，例如 `0x15` |

Requester 发出后不能忘掉这笔事：它要保存 Requester ID、Tag、address/length、TC/Attr 等上下文，并启动 Completion Timeout 相关管理。

### 6.2 EP 内部

EP 收到 MRd，BAR 命中并翻译为 AXI read。Cadence §4.5.1 说明：

- core 可同时向 AXI 发最多 64 个 outstanding read request；
- 每个 AXI read 使用唯一 `ARID`；
- core 保存 incoming PCIe Tag ↔ AXI ARID 映射；
- client 按某 ARID 返回 data 后，core 查回 PCIe Tag 并形成 CplD。

这是一条非常漂亮的“协议 ID 翻译”链：

```text
PCIe Requester ID + Tag
       ↕ mapping table
AXI ARID
```

若 AXI `RRESP` 为 SLVERR/DECERR，Cadence core 发送 Completion Status = Completer Abort，而不是把错误 AXI data 当成功返回。

### 6.3 Completion header

Completion 总是 ID-routed、使用 3DW header。关键字段：

- **Completer ID**：谁完成请求；本例通常是 `01:00.0`，传统编码值 `0x0100`。
- **Completion Status**：`000 SC`、`001 UR`、`010 CRS`、`100 CA`。
- **Byte Count**：包括本 Completion 在内，完成原请求还剩多少 byte。
- **Requester ID + Tag**：复制原请求，Requester 用它找到 outstanding entry。
- **Lower Address**：本 Completion 返回的第一个有效 byte 的低 7 bit。
- **Length**：这条 Completion 实际携带的 DW 数。

对本例的一条成功 CplD：

| 字段 | 示例值 |
|---|---|
| Fmt / Type | `010b / 01010b` |
| Length | 1 DW |
| Completion Status | SC |
| Byte Count | 4 |
| Lower Address | `0x00`（`0x80000100 & 0x7f`） |
| Completer ID | `01:00.0` → `0x0100` |
| Requester ID / Tag | 原请求的值 / `0x15` |
| Payload | 目标寄存器的 4 B 数据 |

来源：Base Spec §2.2.9、Figure 2-38、Table 2-35，printed/PDF pp.153–156。

## 7. 一个 MRd 可以对应多个 CplD

大读请求可能被拆成多个 Completion。Requester 用同一 Transaction ID 收集它们，并检查：

- 数据地址递增；
- Byte Count 逐步减少；
- Lower Address/Length 正确；
- 所有成功 Completion payload 总和满足请求；
- 不违反 RCB（Read Completion Boundary）切分规则。

RCB 对 Root Complex 可以是 64 B 或 128 B，其他 system element 为 128 B。它约束允许的 Completion 边界，不等于 MPS 或 MRRS。

Cadence 也会在 inbound TLP length 超过最大 AXI burst 时拆成多条 AXI read，使用同一 ARID 保序，并随本地 read data 返回生成多个 split completions。

## 8. 三种“完成/确认”别串台

| 机制 | 范围 | 它证明什么 | 它不证明什么 |
|---|---|---|---|
| Data Link ACK | 相邻一条 Link | 带 Sequence/LCRC 的 TLP 已被下一跳正确接收 | 最终 Completer 已执行业务 |
| PCIe Completion | Requester ↔ Completer 事务端到端 | Non-Posted Request 的成功/错误和可能的数据 | Posted MWr 的普通业务完成 |
| AXI RRESP/BRESP | Cadence core ↔ 本地 AXI client | SoC 内部 AXI transaction response | 单独不能代表远端 PCIe 请求全过程 |

Cadence core 正是在边界处翻译语义：AXI read error → Cpl CA；远端 UR/CA/CTO 等 → AXI SLVERR/DECERR。把这三层分开，错误路径才不会被揉成一句“总线超时”。

## 9. Completion error 怎样回到 AXI

Cadence §4.5.6.1–.2 的 outbound requester path 有 64-entry Split Completion Table、6-bit Tag。收到 Completion 后，core 检查 Tag 对应 entry，并验证 Requester ID、TC、Attr、byte count、starting address 等。

可导致 AXI error response 的事件包括：

- Poisoned Completion；
- Completion Status 为 UR、CA 或 CRS；
- incorrect byte count；
- Tag 命中但 Requester ID/TC/Attr 不匹配；
- starting address error；
- Completion Timeout；
- 目标 Function 被 FLR。

如果 Completion 的 Transaction ID 根本匹配不到 outstanding request，Spec 称为 **Unexpected Completion**，接收者应丢弃并报告相应错误；原 Requester 还可能最终触发 Completion Timeout。来源：Base Spec §2.3.2，printed/PDF p.175 起。

## 10. Debug 方法

### Memory Write 没效果

1. 确认 MSE、BAR/resource 与 host outbound window。
2. 抓 TLP：地址、Length、First/Last BE、payload 是否正确。
3. 查 EP BAR/function hit 与 inbound translation。
4. 查 AXI AW/W/WSTRB/BRESP；Requester 不应等待规范并未要求返回的 Completion。

### Memory Read 超时

1. 请求是否真的发出，Tag 是否耗尽？
2. EP 是否收到并 BAR hit？
3. AXI AR 请求/返回是否完成？
4. 是否生成 CplD/UR/CA？
5. 返回路径 Requester ID + Tag 是否正确？
6. 是否被 Switch/RC 当 Unexpected Completion 丢掉？
7. Completion Timeout status/AER header log 记录了什么？

## 11. Source Reference

- PCIe Base Specification 5.0 r1.0, §§2.1–2.2.1, pp.103–110。
- PCIe Base Specification 5.0 r1.0, §2.2.5, pp.117–119。
- PCIe Base Specification 5.0 r1.0, §2.2.7, pp.127–133。
- PCIe Base Specification 5.0 r1.0, §2.2.9, pp.153–156。
- PCIe Base Specification 5.0 r1.0, §§2.3.1.1–2.3.2, pp.169–176。
- Cadence PCIe Core User Guide, §§4.5.1–4.5.2, printed pp.229–236 / PDF pp.318–325。
- Cadence PCIe Core User Guide, §§4.5.6.1–.2, printed p.248 / PDF p.337。
