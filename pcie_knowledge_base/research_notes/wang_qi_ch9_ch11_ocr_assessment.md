# 王齐第 9/11 章 OCR 交叉验证：Flow Control 与 Ordering

日期：2026-08-23

范围：扫描 PDF pp.254–276、291–305（printed Chapter 9/11 相应页面），中文 OCR 后与 PCIe Base Spec 5.0 §§2.4、2.5、2.6 对照。

## 值得吸收到正文的解释

### 1. Credit 是“接收缓存许可证”

教材先用 Upstream/Current/Downstream 的缓冲模型解释：发送端只有拿到下一跳反馈的空闲 buffer credit 才发包。这比直接背 InitFC/UpdateFC 字段更直观，也验证了知识库的 link-by-link 解释。

### 2. 六类 credit 对应六类真实队列

PH/PD、NPH/NPD、CplH/CplD 把 header/data 分开计量。一个 MWr 同时消耗 PH 与若干 PD；无数据 Completion 只消耗 CplH。可变 payload 是 header/data 分池的关键理由。

### 3. InitFC 是 Data Link Active 的前置条件

教材把 FC_INIT1/FC_INIT2 与 DL_Init/DL_Active 串起来：VC0 credit 初始化未完成，Transaction Layer 不能正常发 TLP。这很好地解释了“LTSSM 已 L0 但仍无法枚举”的中间断点。

### 4. Ordering 既保护 producer/consumer，也防 deadlock

教材用 data→flag/doorbell 模型说明 Posted 不能随意互相穿越，又用 Posted 必须能越过某些 NP traffic 说明 deadlock avoidance。这与 Base Spec Table 2-40 的 No/Yes 含义一致。

### 5. 不同 request 的 Completion 可乱序

同一 request 的 split CplD 需按地址推进，但不同 Tag/request 的 CplD 可交错/乱序；Requester 用 Tag 与 reordering buffer 恢复软件/AXI 所需顺序。Cadence §4.5.6.3.3 正好给出硬件实现。

### 6. Data与MSI跨TC时需要额外同步

教材在Capric卡实例中指出：若DMA Posted Writes用TC1而MSI MWr用TC0，不同TC无相对顺序保证，handler可能先运行却看到旧data。它建议使用同TC，或在MSI前对相同target/对应TC发zero-length Read并等Completion来flush先前Posted Writes。

这一点与Base Spec 5.0 §§2.2.5、6.1.4一致，已补入中断正文。需要说清：MSI本身的RO/No Snoop必须为0，同TC的早先Posted Writes不能被MSI超车；风险来自跨TC。

## 必须按新版 Spec 修正/加边界的地方

1. 教材某段把 Flow Control称为“端到端”，但它的实际例子和 Base Spec 都表明 PCIe credit 是 **每条 Link 两端、逐跳**；穿过 Switch 后下一跳重新计算。
2. 教材用“同一 VC 内有序”概括 Ordering 不够精确。Base Spec 5.0 规定 Table 2-40 规则作用于 **同一 TC**；不同 TC 没有 ordering requirement，即使多个 TC 映射到同一 VC。
3. “多数设备只有一个 VC/大多数不支持 RO”是 2010 年实现观察，不是协议永久规则，知识库不把它写成 normative statement。
4. 教材沿 PCI/PCI-X delayed transaction 讲大量历史规则，适合理解 deadlock来源，但不能直接映射成现代 PCIe TLP 名称。
5. Credit=0 的 Infinite 语义依 credit type/receiver角色和规范条件，不应简化成“0 都是不限量”。

## 对现有正文的审计结果

- `link/data_link_ack_replay_and_flow_control.md` 已正确写成 hop-by-hop，并区分六类 credit。
- `transactions/mps_mrrs_ordering_and_performance.md` 已正确写成 same-TC rules，并区分 RO/IDO/No Snoop。
- `debug/link_not_up.md` 已把 InitFC/DL_Up 放在 L0 与 Config TLP 之间。
- `interrupts/intx_msi_msix_and_cadence.md` 已新增data/MSI跨TC顺序和zero-length Read flush专节。

## Source Reference

- 王齐，《PCI Express 体系结构导读》，Chapter 9、11，2010 年第一版，扫描/OCR。
- PCIe Base Specification 5.0 r1.0, §§2.4–2.6。
- Cadence PCIe Core User Guide, §§4.3、4.5.4、4.5.6.3.3、4.5.6.8。
