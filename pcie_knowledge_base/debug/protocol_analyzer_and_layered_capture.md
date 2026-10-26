# PCIe Protocol Analyzer：分层捕获与证据对齐

## 1. 为什么PCIe不能像I2C那样直接看线

I2C/SPI调试时，时钟、片选、地址和data在少量独立导线上，人眼对着logic analyzer就能大致讲出故事。PCIe把clock recovery、scrambling/encoding、ordered set、DLLP和TLP都塞进高速差分串行bitstream，两个方向还在各自独立的Tx/Rx pair上同时工作。

普通scope可以告诉你信号“长得像不像话”，但不会自动告诉你：这一段是TS1，那一段是Ack DLLP，再后面是对`01:00.0 BAR0+0x100`的Memory Write。Protocol analyzer的价值是把原始物理信号恢复成按层可检索的事件。

## 2. 先写“我要证明什么”，再写trigger

一次有用的capture不是“抓所有东西，回去再说”。buffer总有限，满屏SKP/TS/FC update会让关键事件跳出capture window。建议先填这张小表：

| 问题 | 需要的trigger | 必须保留的前后文 |
|---|---|---|
| Link为什么训练失败 | LTSSM state change、TS1/TS2 field异常、Recovery loop | receiver detect到失败后数个training attempt，两个方向 |
| Enumeration卡在某BDF | 目标Bus/Device/Function的Config Request | Type 1路由、最后Type 0、Completion status/timeout |
| BAR write没效果 | 目标PCI address+request type+direction | 请求header/data、bridge/EP返回的error message |
| Memory Read CTO | Requester ID+Tag+address的MRd | 全部Cpl/CplD fragments、AER/error Message、Link recovery |
| DMA数据错 | RID/PASID+DMA address range | RO/IDO/No Snoop、Byte Enable/Length、doorbell与data的相对时间 |
| MSI没进CPU | MSI target address/data的MWr | EP发送时间、RC/IOMMU/MSI-controller软件证据 |

一条trigger最少要说清：**Link/方向、层、packet type、identity/address、时间关系**。只写“触发TLP”基本等于请analyzer帮忙制造大数据。

## 3. 两个方向必须对在同一条时间线上

PCIe是dual-simplex：Request可能在一个方向，Completion/Ack/Error Message在反方向。比如排查Memory Read：

```text
downstream: MRd  RID=01:00.0 Tag=0x2a Addr=...
                  │
                  ├─ 中间可能经历Ack/replay/Recovery
                  │
upstream:   CplD Completer=... Requester=01:00.0 Tag=0x2a
```

如果两个方向没有common time base，你可能看见了request和completion，却回答不了latency、timeout和recovery是谁先发生。

Upstream/Downstream按topology定义：向Root Complex是upstream，离Root Complex更远是downstream。不要被probe厂商的A/B端或Tx/Rx命名带偏，capture前先做一笔已知transaction验证方向标签。

## 4. 按层解码，不要跳层定罪

### Physical/LTSSM

看receiver detection、Electrical Idle、TS1/TS2、lane number、width/speed协商、equalization和Recovery。Link没到L0时，后面讨论BAR/MSI没有意义。

### Data Link

看sequence number、LCRC、Ack/Nak、Replay Timer/REPLAY_NUM、InitFC/UpdateFC。某TLP在同一Link上出现两次可能是正常replay，不能立即判成Requester发了两笔业务transaction。

### Transaction

看Fmt/Type、TC/Attr、Requester/Completer ID、Tag、Length/Byte Enable、address、Completion Status和payload。Memory Write没有Completion；想证明“write已生效”，还要结合device-visible status/readback或上层协议。

## 5. Scope、protocol analyzer和logic analyzer各自管哪段

| 工具 | 最擅长 | 不能单独证明 |
|---|---|---|
| 高带宽scope/VNA/TDR | eye/jitter/loss/reflection/crosstalk/channel | 某个BDF的Config Completion为什么UR |
| PCIe protocol analyzer/exerciser | ordered set、DLLP/TLP、双向trigger、故障注入 | SoC内部AXI address翻译最后命中了哪个寄存器 |
| SoC logic analyzer/trace | PIPE/controller/client/AXI内部状态 | 板级差分信号的真实eye margin |
| Linux log/config dump | OS看到的topology/resource/capability/error | 被mask且未进software的硬件event |

最快的debug往往是对齐多个时间源：在driver中打sequence marker，保留controller local timestamp/status，analyzer抓同一笔RID/Tag/address，scope仅在证据指向SI时上场。

## 6. 当年教材的图还能看，数字别照抄

MindShare Appendix A用Gen1 2.5 GT/s、8b/10b和早期analyzer UI演示TS1/TS2、SKP与InitFC，“需要双向、分层、精确trigger”的方法论仍然完全有用。但其2.5 Gb/s、某个probe connector、差分走线20 mil等数字不是Gen4/Gen5 compliance限值。

现代Gen3+使用128b/130b、equalization和更严格channel测量；板级要求必须回查目标PCIe generation的Base/CEM/electrical spec、PHY Databook和board stack-up/simulation。

## 7. Source Reference

- MindShare *PCI Express System Architecture*：Appendix A，printed pp.961–987左右 / PDF pp.1017–1043，dual-simplex capture、layered decode、trigger/exerciser和Gen1 SI示例。
- PCI Express Base Specification 5.0 r1.0：Ch.2–4 packet/layer/link encoding；Ch.6 error/training rules。
- 相关手册：[Link Not Up](link_not_up.md)、[Completion Timeout](completion_timeout.md)、[AER Debug](aer_debug.md)、[Data Link ACK/Replay/FC](../link/data_link_ack_replay_and_flow_control.md)。
