# TC、VC、QoS、PTM 与 ACS：同样是“高级能力”，解决的却不是一件事

这些缩写经常集中出现在 Extended Capability 列表中，但它们解决的问题并不相同。先明确各自的职责：

- **TC/VC** 解决不同流量如何分类、隔离队列并获得不同服务的问题；
- **PTM** 解决不同 PCIe 组件怎样建立精确时间关系的问题；
- **ACS** 解决一个 TLP 能否按原路径直达、必须重定向，还是应该拦截的问题；
- 当前这份 Cadence Gen4 x16 Dual-Mode 配置有完整的 TC0/VC0 基线能力，但**没有选中可配置 VC Capability、PTM Capability 或 ACS Capability**。

最后一句很重要。控制器 IP 的通用框图里画过某项 capability，不等于交付到手的硬件实例一定把它综合进去了。硅片不会因为软件“非常希望它支持”就临时长出几个寄存器。

## 1. Traffic Class：贴在 TLP 上的服务标签

Traffic Class（**TC，流量类别**）是 TLP header 中的 3-bit 字段，因此有 `TC0`～`TC7` 八种标签。可以先把它理解为快递箱上的服务标签：它不改变箱子里装了什么，也不负责决定最终地址，但沿途每个服务点可以依据标签采用不同的排队和调度策略。

规范给出的核心语义是：

- TC 标签随 TLP 端到端穿过 fabric，正常转发时不被中途改写；
- 同一个 TC 构成一个 ordering domain；不同 TC 之间没有规范保证的相对顺序；
- 所有设备都必须支持 `TC0`；额外 TC 是可选能力；
- Completion 必须沿用原 Request 的 TC；
- INTx、错误、PM、LTR、PTM 等若干 Message 被规范要求使用 `TC0`。

这意味着 TC 不是 Linux 进程优先级，也不是 PHY 的 Lane 优先级。它只是协议包上的分类信息。想让 `TC5` 真比 `TC0` 更快，路径上的 Requester、每个 Switch Port、Root Port 和相关软件策略都要配合；只改发送端一个字段，通常只能得到一份很有个性的配置错误。

### 一个小例子

假设 Endpoint 同时产生两类流量：

```text
控制寄存器读写：TC0
实时采样数据：  TC3
```

如果 TC0 和 TC3 最终都映射到 VC0，它们仍共享 VC0 的队列/credit，只是调度器可能依据 TC 做有限区分。若平台把 TC3 映射到另一条真正独立的 VC，采样数据才可能拥有独立的 Flow Control 资源，避免被 VC0 中堵住的包“堵在同一条收费站车道里”。

## 2. Virtual Channel：不是虚拟 Lane，而是独立逻辑流资源

Virtual Channel（**VC，虚拟通道**）不是把一条物理 Lane 软件切成几段。VC 是在同一条 Link 上复用的逻辑流，通常具有独立的队列、buffer、控制逻辑和 Flow Control credit。

VC 要解决的经典问题是 **head-of-line blocking**：某类流量因为下游没有 credit 而停住时，不应该必然拖住所有无关流量。

规范关系可以压成下面这张小图：

```text
TLP.TC label                         Link 上的物理传输
   TC0 ───────┐                    ┌─ VC0 queues + FC credits ─┐
   TC1 ───────┼─ TC→VC mapping ───┤                          ├─ multiplex → Lane(s)
   TC2 ───────┘                    └─ VC1 queues + FC credits ─┘
   TC3 ────────────────────────────────▲
```

关键规则：

- 每个受支持的 TC 必须映射到一个 VC；
- `TC0 → VC0` 固定，而且所有设备必须实现默认 `VC0`；
- 一个 VC 可以承载多个 TC；同一条 Link 上的一个 TC 不能同时映射到多个 VC；
- 一个 Port 最多支持 1～8 个 VC；VC0 以外的 VC 是可选的；
- 每个 VC 都有独立 Flow Control credit；VC0 在 Link 初始化时由硬件自动完成 InitFC；额外 VC 由软件在链路两端协调 enable 后再独立初始化。

因此，TC 与 VC 的分工是“标签”和“资源”。TC 告诉沿途“我属于哪类”；VC 提供真正可独立阻塞、独立计 credit 的运输通道。

### 与 Ordering 的关系

Base Spec 的普通 ordering 规则以**同一 TC**为主要边界。不同 TC 没有相对顺序保证；不同 VC 当然也不应依赖相对顺序。因为同一 TC 不能在同一 Link 上拆到多个 VC，所以不会出现“规范要求同 TC 有序，硬件却把它随机分流到两个 VC”的自我矛盾。

另外，Flow Control credit 是**逐跳**维护的。VC 能隔离的是当前 Link 两端的资源；它不是一张从 EP 直通 Host Memory 的端到端车票。相关基础见 [Data Link ACK、Replay 与 Flow Control](../link/data_link_ack_replay_and_flow_control.md)。

## 3. QoS 到底来自哪里

PCIe 的 Quality of Service（QoS）不是一个单独的“高优先级开关”，而是多层机制的组合：

```text
应用识别流量
  → Requester 写入 TC 标签
  → 每个 Port 的 TC→VC mapping
  → VC 独立 buffer / credit
  → Port arbitration（如 WRR，具体能力依实现）
  → 下一个 Link 重复上述过程
```

这也解释了为什么只在 Endpoint 把 TLP 设为 `TC7` 并不能自动获得低延迟。中间 Switch 若只实现 VC0，或者所有 TC 都映射进 VC0，真正的资源隔离就不存在。QoS 必须按完整 hierarchy 设计、配置并验证。

性能调优时还要把它与 MPS/MRRS 分开：MPS/MRRS 改变 TLP/request 粒度，TC/VC 改变分类、排序边界和共享资源方式。两者都会影响延迟，却不是同一个旋钮。参见 [MPS、MRRS、Ordering 与性能](../transactions/mps_mrrs_ordering_and_performance.md)。

## 4. 当前 Cadence 配置：只有基线 TC0/VC0

Cadence Appendix H Figure H.1 是 IP 的 capability 链实现示意，图中为 Virtual Channel Capability 预留了 `0x4C0`。但同一附录随后明确提醒：部分 capability 可能未在当前配置中选中；H.1 的 PF0、VF 和 RP **configuration-specific capability list 都没有列出 VC Capability**。

更直接的证据来自 Local Management Error Status：`Unmapped TC` 状态位被 tied to 0，原因原文明确写为当前 Controller configuration 不实现 Virtual Channel Capability Registers。这个结论的工程含义是：

- 当前实例仍必须拥有标准要求的 `TC0/VC0`；
- 软件看不到可配置 VC Extended Capability，不能建立额外 VC；
- 不要照着通用 Figure H.1 的 `0x4C0` 偏移硬读写并期待寄存器存在；
- 对当前项目进行带宽/延迟优化时，应先研究 VC0 内部调度、MPS/MRRS、outstanding、credit 与 AXI 行为，而不是设计一个硬件并不存在的多 VC 方案。

这是一个很典型的阅读方法：**通用架构图说明 IP 家族能做什么，configuration-specific table 才说明这颗实例实际带了什么。**

## 5. PTM：在 PCIe hierarchy 中测出“我们的时钟差多少”

Precision Time Measurement（**PTM，精确时间测量**）让不同 PCIe 组件建立本地时间与 PTM Master Time 的关系，从而协调跨设备事件。它不是 TLP ordering，也不是把 PTP 数据包换了一个名字。

在一条 Link 上：

```text
Upstream Port / PTM Requester                  Downstream Port / PTM Responder
        t1: 发 PTM Request   ──────────────────────────────► t2: 收到
        t4: 收 Response      ◄────────────────────────────── t3: 发 Response/ResponseD
```

Requester 观察往返时间 `t4 - t1`，Responder 提供其内部处理时间 `t3 - t2`。在链路双向传播延迟近似对称的前提下，去掉处理时间后再除以二，可以估计单向 Link delay，并把本地时钟关联到 PTM Master Time。

协议上，PTM 使用 `PTM Request`、`PTM Response` 和带数据的 `PTM ResponseD` Message TLP；一个匹配的 request/response 称为一次 PTM dialog。Capability 角色包括：

- **PTM Root**：提供 PTM Master Time；
- **PTM Responder**：响应当前 Link 上游端发来的请求；
- **PTM Requester**：消费测量结果并维护本地 PTM context。

软件必须沿 hierarchy 有顺序地 enable：只有相关 Downstream Port 已经 enable，关联 Upstream Port 才能 enable。收到 PTM Message 的 Port 若不支持 PTM，或 Downstream Port 尚未 enable，应按 Unsupported Request 处理。

X6000 Linux 5.10.4 的 `struct pci_dev` 保存 `ptm_cap`、`ptm_root`、`ptm_enabled` 和 granularity；`drivers/pci/pcie/ptm.c` 实现 `pci_enable_ptm()` 等调用。该源码能说明 Linux 通用 PTM 管理方式，但 X6000 实际硬件 build 是否暴露 PTM Capability，仍要以 capability dump 和 Cadence configuration 为准。

### 当前 Cadence 实例的结论

通用 Figure H.1 为 PTM Capability 画出了 `0xA20`，但 configuration-specific PF0/VF/RP capability tables 没有 PTM；Chapter 9 也没有找到 PTM Capability register block。因此对当前实例应判定为：**PTM 未选中，而不是“偏移已知所以支持”。**

手册某些 error-mask 字段也缩写为 `PTM`，但那里是 **Poisoned TLP Mask**，与 Precision Time Measurement 无关。缩写复用是手册给读者准备的小香蕉皮，看到 `PTM` 必须先看字段上下文。

## 6. ACS：给 P2P 和隔离路径设置检查站

Access Control Services（**ACS，访问控制服务**）在 RC、Switch 和多 Function Device 的若干位置设置控制点，决定 TLP 是正常转发、阻断，还是强制重定向。它对 IOMMU/VFIO 隔离尤其重要：即使两个设备之间物理上可以 P2P，平台也可能要求请求先回到 Root Complex 的 Translation Agent 检查权限。

Base Spec 5.0 定义的控制包括：

- Source Validation：检查上行 Request 的 Requester ID 是否属于该下游 bus aperture；
- Translation Blocking：阻止不被允许的 translated Request；
- P2P Request / Completion Redirect：把 P2P 流量重定向到上游；
- Upstream Forwarding；
- P2P Egress Control；
- Direct Translated P2P，以及后续版本定义的更多访问控制项。

ACS 是 optional Extended Capability，Capability ID 为 `0x000D`，且硬件控制默认 disabled，由 ACS-aware software enable。Linux 6.8 header 中可见 `pci_dev.acs_cap`、`pci_request_acs()`、`pci_acs_enabled()`，以及 Source Validation、Translation Blocking、Request/Completion Redirect 等寄存器 bit 定义。

### 当前 Cadence 实例的结论

Cadence 手册只有“当支持 ACS 时才相关”的 ARI ACS Function Group 字段，以及若干内部 ACS 宽度信号名字；Appendix H 当前配置列表没有 ACS Capability，Chapter 9 也没有对应 ACS Extended Capability register block。因此不能把这些零散字段当成当前实例支持 ACS 的证据：**本资料所描述的当前配置未暴露 ACS Capability。**

这会影响虚拟化安全结论：没有 ACS 的 topology 可能仍能工作，但不能仅凭软件愿望把某些 P2P 路径视为硬件隔离。最终 IOMMU group/VFIO 判断还依赖目标 Root Complex、Switch 和 SoC；这些资料目前未提供。

## 7. 现场验证：别只信文档，也别只信一个偏移

拿到真实板卡后，正确的检查顺序是：

1. 用 `lspci -vv -s <BDF>` 查看内核解析出的 VC、PTM、ACS 能力；
2. 用 `lspci -xxxx` 或安全的 config-space reader 遍历 Extended Capability next pointer；
3. 确认 capability ID，而不是直接读取手册通用图中的固定偏移；
4. 对多 Hop topology，在每一段 Port 上分别检查能力和 enable 状态；
5. 若硬件与 configuration-specific table 不一致，记录 RTL/IP configuration、chip revision、strap 与固件修改，不能笼统写成“Cadence 支持/不支持”。

Capability walker 的防环、对齐和边界规则见 [Capability discovery 与安全遍历](../configuration/capability_discovery_and_safe_walking.md)。

## 8. 常见误解

**“VC 就是虚拟 Lane。”** 不是。Lane 是 Physical Layer 的串行物理通道；VC 是拥有独立 Flow Control 资源的逻辑流。

**“TC 数字越大，优先级越高。”** 规范把 TC 定义为服务类别标签，不定义简单的数值越大越快规则。实际服务策略由平台配置和每个 Port 的 arbitration 决定。

**“设备支持 TC 字段，就支持 8 个 VC。”** 所有 TLP header 都有 TC 字段，但基线设备可以只产生 TC0、只实现 VC0。

**“PTM 能保证 DMA 完成顺序。”** PTM建立时间关系；事务顺序由 ordering、RO/IDO、barrier 和设备语义管理。

**“有 IOMMU 就不需要 ACS。”** IOMMU只检查经过它的请求。若拓扑允许设备间请求绕过 Translation Agent 直达，ACS redirect/blocking 才是把路径纳入隔离策略的重要控制点。

## 9. Source Reference

- PCI Express Base Specification Revision 5.0 Version 1.0：
  - §2.2.6.6，Traffic Class；
  - §2.5，Virtual Channel Mechanism；
  - §2.6、§3.4，per-VC Flow Control；
  - §6.3，VC configuration；§7.9.1/§7.9.2，VC/MFVC Extended Capability；
  - §2.2.8.10、§6.22、§7.9.16，PTM Message、机制与 Extended Capability；
  - §6.12、§7.7.8，ACS 行为与 Extended Capability。
- Cadence PCIe Core User Guide：
  - Appendix H Figure H.1，通用 capability linked-list implementation，printed p.1369 / PDF p.1458；
  - Appendix H.1 Tables H.1–H.4，当前配置 PF/VF/RP 实际 capability list，printed pp.1370–1372 / PDF pp.1459–1461；
  - Chapter 9 Local Error Status `Unmapped TC`：当前配置未实现 Virtual Channel Capability Registers，代表位置 printed pp.594、1046；
  - 手册 glossary 中 PTM = Precision Time Measurement；error-mask 字段中的 PTM = Poisoned TLP Mask，必须按上下文区分。
- X6000 Linux 5.10.4：
  - `include/linux/pci.h`：`struct pci_dev` 的 PTM/ACS 元数据与 `pci_enable_ptm()`、`pci_request_acs()` 等 API declaration；
  - `drivers/pci/pcie/ptm.c` 与 `drivers/pci/quirks.c`：PTM 管理和 ACS 相关策略/quirk；
  - `include/uapi/linux/pci_regs.h`：VC/MFVC/PTM/ACS Extended Capability IDs 和 PTM/ACS register bit definitions。
