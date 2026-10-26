# Cadence Chapter 9 反向寄存器覆盖审计

日期：2026-08-23

## 1. 这轮审计在找什么

Cadence Chapter 9 从 PDF p.411 一直铺到 p.1357。它很长，但“长”主要来自四种机械展开：同一组 capability 对 PF/VF/RP 展开、同一套 region 对 32 个 outbound window 展开、同一套 BAR translation 对多个 Function 展开，以及 LINK0–3 的重复。

所以本轮不按页数制造完成感，而是反过来问：

> 每一类寄存器背后的硬件职责，在知识库里有没有解释；软件什么时候读写；写错会变成哪种 TLP、AXI response 或 Link 症状？

状态标记：`D` 有独立正文，`M` 已与 Spec/Linux职责连接，`O` 主要是观测点，`B` 缺目标 SoC/Linux/PHY 证据。

## 2. 六个顶层 register block

| Chapter 9 block | 硬件职责 | 覆盖 | 结论 |
|---|---|---:|---|
| PF Configuration | EP 对 Host 暴露的 identity、Command/BAR、Capability/Extended Capability | D+M | Appendix H selected chain 已逐项反查；不能用 generic capability 图代替实际 build/readback |
| VF Configuration | 每个 VF 的独立 config view | D+M | 已解释 VF 与 PF capability/BAR management 的不对称；Function 1–8 的表是重复实例，不是八种新协议 |
| Root Port Configuration | Type 1 header、bus/window、Link/Slot/Root/AER | D+M | 已连到 host bridge、枚举、resource window、AER/PM；精确 Linux offset writer 被源码缺失阻塞 |
| Local Management | Cadence 私有 link/TL/DL/PHY、error、PM、debug 和 build control | D+M+O | 关键状态机与调试观测面已覆盖；需要目标 PHY 文档的 coefficient/analog 语义保留 B |
| AXI Configuration | outbound region descriptor/translation 与 inbound BAR→AXI translation | D+M | Region 0–31、Function/BAR 的重复表按结构消化；已用 Chapter 10 实例验证 Config/Memory/Message 路径 |
| PCIe DMA | 四 channel、descriptor pointer/attribute、IRQ/ECC/capability | D+M | 已形成独立 uDMA 编程与故障文档；Interrupt Enable/Disable 描述冲突保留 OQ-010 |

## 3. Local Management：哪些寄存器值得驱动工程师记住

### 3.1 配置/动作寄存器

| 寄存器族 | 控制的硬件 | 软件阶段 | 配错时更像什么 |
|---|---|---|---|
| Physical Layer Configuration、Linkwidth Control、SRIS Control | generation/width、training辅助策略、独立参考时钟 | training前 | Detect/Polling/Recovery循环，或只降速/降宽 |
| Gen3/Gen4 preset、Equalization Control | 8/16 GT/s equalization初值和状态机策略 | training前；retrain前按需 | Gen1/2能起、Gen3/4失败是典型气味；默认 `1111b` preset 无效，不能偷懒 |
| Completion Timeout Limit | controller内部 completion watchdog 范围 | training前/初始化 | 合法慢请求被误报 timeout，或坏请求长时间挂死 |
| Receive Credit Limit / Update Interval / Scaled FC | 本端广告 buffer 容量和 UpdateFC节奏 | Link enable前 | FC init失败、吞吐停顿或 Flow Control Protocol Error |
| PF/VF/RC BAR Configuration | BAR type、aperture、Resizable BAR选择 | EP identity配置期 | BAR sizing读回异常、Host分配后仍不命中 |
| L0s/L1 delay、PM_CLK、PME timeout | ASPM/PM handshake实现定时 | policy确定后、training前 | L0s/L1抖动、PME/L23 handshake timeout |
| Local Error/Interrupt Mask、Link Down Indication Control | 私有错误上报、W1C与Link Down握手 | 初始化与恢复 | 错误风暴、事件永远不再来，或过早恢复AXI traffic |
| AXI/Core Feature Control | integration/build相关行为开关 | reset/初始化限定阶段 | APB/AXI边界行为改变；没有目标 integration guide 时不能凭名字猜 bit |

这些寄存器并非全部由 Linux PCI core 配置。标准 Configuration Space 通常由 PCI core或Function driver管理；Local Management 更可能属于 boot firmware 或 SoC-specific host/EPC driver。workspace 没有目标 glue code，所以这里只记录职责映射，不推测具体函数名。

### 3.2 观测寄存器：它们是故障定位的仪表盘

| 观测面 | 能回答的问题 | 不能单独证明什么 |
|---|---|---|
| TX/RX TLP count、payload DWord count | Link上是否真的有事务、流量方向与粗粒度payload量 | 某个TLP的Header、Tag或最终业务正确性 |
| RX/TX Credit Limit、Scaled FC状态 | 对端广告了多少credit、是否卡在某类P/NP/Cpl资源 | buffer为什么没释放；还要看ordering/Completion/AXI backpressure |
| DL Gen1–4 Receive DLLP count | Data Link活动是否存在、不同速率代际的接收计数 | DLLP内容与CRC是否正确 |
| Negotiated Lane Map、Receive FTS count、PHY STATUS | lane映射、训练序列和PHY侧状态是否合理 | 模拟眼图、抖动根因；那是PHY Databook/示波器的地盘 |
| LTSSM history / Debug Mux | 状态机刚才从哪里退回、内部选择信号是什么 | 板级信号完整性根因 |
| LCRC Error Count、AER Header/TLP Prefix log | 是否发生Link传输错误、首个失败transaction长什么样 | 只凭累计计数不能断定当前一次业务失败由它造成 |
| ECC counters（core/AXI/PNP/uDMA RAM） | controller内部RAM是否有correctable/uncorrectable事件 | PCIe线缆一定有问题；这可能是SoC RAM integration |
| MSI mask set/clear、MSI-X Function Mask变化 | Host何时改变mask，client是否应补发pending vector | MSI Memory Write是否到达CPU interrupt remapper |
| Lane Margining / Equalization Monitor | 每Lane margin command、error count、phase/retimer parity | 真实电压/时间裕量的绝对物理意义，仍需PHY定义和校准边界 |

一个实用原则是：**先读 sticky/status/log，再清 W1C，最后触发恢复。** 先清再截图，就像先擦黑板再问是谁写错了公式，现场会非常整洁，也非常没用。

## 4. AXI Configuration：把 400 多项压缩成两个可推理的模板

### 4.1 Outbound region 模板

每个 region 的核心不是编号，而是四件事：

```text
AXI region base/size
  + AXI→PCIe address translation
  + PCIe descriptor（Memory/I/O/Config/Message与属性）
  + enable/匹配优先级
      → 本地 AXI access 变成哪一种 outbound TLP
```

Region 0–31 的表只是这套模板展开。驱动真正要审的是 overlap/alignment、64-bit地址写入顺序、descriptor type、Requester属性以及启用前是否已经写完整。

### 4.2 Inbound BAR 模板

EP inbound register 按 Function×BAR 展开：

```text
Host分配给BAR的PCI address
  → TLP address命中Function/BAR
  → 对应 inbound translation 加/替换地址高低位
  → core AXI Master在本地SoC发起访问
```

因此 `Function 8 BAR 5` 并不是特殊事务；它只是 VF7 的一个实例。知识库按“Function identity + BAR hit + translation”解释一次，读者就能推导其它实例。逐项抄 100 多个 offset 只会让树更茂盛，森林反而不见了。

## 5. 反向检查结果与剩余边界

- Chapter 9 的所有**顶层寄存器块**均有知识入口；
- Local Management 的动作、状态、计数、日志、mask 和 equalization/margining 观测面均已评估；
- AXI Region、Function/BAR、Link replication 被判定为结构化重复，并非遗漏；
- Appendix H capability selected list 已另做逐项审计；
- 未发现一个“当前 build 已选中、对驱动/调试关键、却完全没有知识落点”的新模块；
- 仍为 `B/Unknown` 的是实际 SoC APB physical base、strap/readback、PHY coefficient/analog定义，以及目标 Linux symbol→offset调用链；这些无法从本 User Guide 单独推出。

## 6. Source Reference

- Cadence *PCIe Core User Guide*, Chapter 9 opening/Table 9.1，printed pp.322–323 / PDF pp.411–412。
- §9.1 PF Configuration、§9.2 VF Configuration、§9.3 Root Port Configuration，printed pp.324–569 / PDF pp.413–658。
- §9.4 Local Management，printed pp.570–668 / PDF pp.659–757。
- §9.5 AXI Configuration，printed pp.669–791 / PDF pp.758–880。
- §9.6 PCIe DMA，printed pp.792–804 / PDF pp.881–893。
- §§9.7–9.24，LINK1–3 replicated blocks，printed pp.804–1268 / PDF pp.893–1357。
- Chapter 10 programming examples，printed pp.1269–1283 / PDF pp.1358–1372。
