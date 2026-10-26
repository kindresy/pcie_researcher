# Glossary

这份术语表给的是“驱动工程师真正该想到什么”，不只展开英文全称。随着主题研究会持续补充字段宽度、版本和来源。

| 缩写 / 术语 | 英文全称 | 对驱动工程师意味着什么 |
|---|---|---|
| PCIe | PCI Express | 保留 PCI 软件模型、但在线路上使用点到点串行分组协议的高速 I/O 互连 |
| RC | Root Complex | I/O hierarchy 的根，把 CPU/Host Memory 子系统连接到 PCIe fabric；可包含多个 Root Port |
| RP | Root Port | RC 向外暴露的一条 PCIe hierarchy 入口；配置上表现为桥型 Function，而不是 RC 的同义词 |
| EP | Endpoint | PCIe 叶子设备中的 Function，可作为 Requester 发请求，也可作为 Completer 响应请求 |
| RCiEP | Root Complex Integrated Endpoint | 集成在 RC 内、没有外部 Link 的 Endpoint；部分 Link/ASPM 规则不适用 |
| Switch | PCIe Switch | 在多个点到点 Link 间路由包；配置模型上像一组虚拟 PCI-to-PCI Bridges |
| Hierarchy | PCIe Hierarchy | 从一个 RC interface/Root Port 向下展开的一棵设备树 |
| BDF | Bus / Device / Function | 软件给 Function 的路由身份，例如 `01:00.0`；Bus 8 bit、Device 5 bit、Function 3 bit（传统解释） |
| RID | Routing ID | 标识 PCIe Function 的 16-bit 路由身份；根据上下文成为 Requester ID 或 Completer ID，传统解释来自 BDF，ARI 下改为 Bus+8-bit Function |
| Requester ID | Requester Identifier | Request 中标明发起 Function 的 RID；与 Tag 组成 Transaction ID，IOMMU/AER/Completion 都会使用 |
| Completer ID | Completer Identifier | Completion 中报告完成请求一方的身份 |
| Configuration Space | Configuration Space | 每个 Function 的标准化控制/状态空间；PCIe Function 最多 4 KiB，不是设备业务 BAR 空间 |
| ECAM | Enhanced Configuration Access Mechanism | 把 BDF+register offset 线性映射成 CPU MMIO 地址的标准配置访问机制 |
| BAR | Base Address Register | Function 申报地址窗口类型/大小并接受系统分配基址的配置寄存器；它不是那块存储本体 |
| Expansion ROM / Option ROM | Expansion ROM | Function可选向host暴露的启动firmware image；用独立ROM BAR临时映射成Memory Space target，不是BAR0或设备内部firmware的必然同义词 |
| RCRB | Root Complex Register Block | RC内部、由firmware描述base的4 KiB MMIO寄存块，可承载VC等Extended Capability；它不是ECAM或可枚举EP |
| DSN | Device Serial Number | 可选的64-bit设备序列号Extended Capability；适合识别/asset tracking，但不能单独作为安全信任根 |
| Capability | Capability Structure | 配置空间内以链表暴露的可选能力，如 PM、MSI、MSI-X、PCIe Capability |
| Extended Capability | PCI Express Extended Capability | 4 KiB 扩展配置空间中的能力链，如 AER、ATS、SR-IOV |
| TLP | Transaction Layer Packet | Memory/Config/I/O/Message Request 与 Completion 在线路上的事务层包 |
| DLLP | Data Link Layer Packet | 只在相邻 Link 两端交换的链路管理包，如 ACK/NAK 和 Flow Control update |
| LTSSM | Link Training and Status State Machine | 硬件状态机，负责检测对端、训练 Lane、协商宽度/速率并维护 Link；不是 Linux 线程 |
| Link | PCI Express Link | 两个相邻 Port 间的双单工通信通道，由一条或多条 Lane 聚合而成 |
| Lane | PCI Express Lane | 一组 Tx 差分对加一组 Rx 差分对；x4 表示四条 Lane，并非四倍时钟 |
| PHY | Physical Layer / PHY | 完成电气发送接收、SerDes 等物理功能；Controller 常通过 PIPE 与独立 PHY 对接 |
| PIPE | PHY Interface for PCI Express | PCIe controller MAC/逻辑与 PHY 之间的并行接口标准；它不是线上协议 |
| Completion | Completion TLP | 对 Non-Posted Request 的响应，可带数据（CplD）或仅带状态（Cpl） |
| Cpl / CplD | Completion / Completion with Data | 不带/带数据的 Completion；Memory Read 正常返回 CplD，Configuration Write正常返回不带数据的Cpl |
| Posted Request | Posted Request | Requester 发出后协议上不等待 Completion；典型例子是 Memory Write |
| Non-Posted Request | Non-Posted Request | 必须由 Completer 返回 Completion；典型例子是 Memory Read 与 Configuration Read |
| Tag | Tag | Requester 为 outstanding Non-Posted Request 分配的事务编号，用来把返回 Completion 对回原请求 |
| TC | Traffic Class | TLP 的端到端流量分类标签，用于 QoS/映射 VC；不是 Linux task priority |
| VC | Virtual Channel | Link 上具有独立 flow-control 资源的虚拟通道，减少不同流量间的 head-of-line blocking |
| QoS | Quality of Service | 通过 TC 分类、TC→VC mapping、独立 credit 和端口调度形成的服务策略；不是一个“高优先级”总开关 |
| Flow Control Credit | Flow Control Credit | 每条 Link 上接收端通告的 buffer 容量配额；发送端有足够 header/data credit 才能发对应 TLP，不是跨 Switch 的端到端额度 |
| DLF | Data Link Feature | 相邻 Port 在 Data Link 初始化期间交换可选功能支持的机制；当前资料重点是 Scaled Flow Control |
| SFC | Scaled Flow Control | 为高速 Link 扩大可表达 credit 范围的 Data Link 功能；scale/limit必须在训练和FC初始化规则内使用 |
| MPS | Max Payload Size | 单个 TLP data payload 的上限；影响效率、缓冲需求与 fabric 兼容性 |
| MRRS | Max Read Request Size | Function 允许自己发出的 Memory Read Request 最大长度 |
| RCB | Read Completion Boundary | Completer可切分一次 Memory Read 返回数据的自然对齐边界；它不是 MRRS 或 MPS |
| RO | Relaxed Ordering | TLP 属性，允许特定默认顺序约束放宽；只能用于不存在生产者/消费者依赖的事务 |
| IDO | ID-Based Ordering | 以 Requester/Completer ID（某些场景含 PASID）区分 ordering domain 的属性 |
| No Snoop | No Snoop Attribute | 表示不期望硬件强制 cache coherency；不改变 transaction ordering |
| MSE | Memory Space Enable | PCI Command bit，控制 Function 是否响应其 Memory BAR address decode |
| BME | Bus Master Enable | PCI Command bit，允许 Function 作为 Requester 主动发 Memory/I/O Request；DMA 常需它 |
| INTx | Legacy INTx Interrupt | 兼容 PCI 的 level-triggered 中断语义；PCIe Link 上通常用 Message 表达，不是贯穿 fabric 的独立引脚 |
| MSI | Message Signaled Interrupt | 设备向系统指定地址发特定数据的一次 Memory Write，用“写消息”代替物理中断线 |
| MSI-X | MSI-X | MSI 的扩展机制，vector 更多且每个 vector 有独立 address/data/mask 表项 |
| PBA | Pending Bit Array | MSI-X位于BAR空间的pending位图；vector被mask时设备用它记录尚待投递的中断 |
| DMA | Direct Memory Access | PCIe EP 作为 bus master/Requester 主动访问 Host Memory；数据仍通过 PCIe TLP 传输 |
| DMA address | DMA Address | 设备应使用的地址；可能是 IOVA，不保证等于 CPU physical address，更不是 CPU virtual address |
| IOMMU | I/O Memory Management Unit | 位于设备 DMA 请求与系统内存之间，翻译 IOVA 并执行隔离/权限检查 |
| CRS | Configuration Request Retry Status | Function 暂未准备好处理配置请求时返回的 Completion status，要求 Host 按规则重试 |
| UR | Unsupported Request | 请求类型、地址或目标不受支持；Non-Posted 请求通常以 UR Completion结束 |
| CA | Completer Abort | Completer接受到请求但无法正常完成，返回 CA Completion status |
| CTO | Completion Timeout | Requester 发出需要 Completion 的请求后，规定时间内未收齐响应 |
| AER | Advanced Error Reporting | PCIe 扩展错误记录与上报能力，区分 correctable/non-fatal/fatal 并保留 header/prefix log 等诊断信息 |
| DPC | Downstream Port Containment | Downstream Port 检测严重错误后隔离下游并阻止错误扩散的可选能力 |
| PTM | Precision Time Measurement | 通过逐 Link Message dialog 测量传播延迟并建立本地时间与 Master Time 的关系；不负责事务 ordering |
| TPH | TLP Processing Hints | Requester给平台的处理/目标局部性hint；它不是路由命令，也不保证性能提升 |
| VSEC | Vendor-Specific Extended Capability | 厂商自定义扩展能力；walker能识别header，但字段语义必须查对应vendor/build文档，不能只认固定offset |
| ACS | Access Control Services | 在 RC/Switch/MFD 中验证、阻断或重定向 TLP 路径的可选访问控制能力，对 P2P/IOMMU 隔离很重要 |
| ASPM | Active State Power Management | 由链路两端协商/配置的主动链路省电机制，涉及 L0s/L1；不同于 Function D-state |
| DPA | Dynamic Power Allocation | 在Function仍处于D0时由软件选择功耗预算substate；不是D1/D2或ASPM |
| Power Budgeting | Power Budgeting Capability | Function按D-state/工况/电源轨向系统报告功耗需求，供firmware/OS做平台预算；不直接限流 |
| Slot Power Limit | Slot Power Limit | Downstream Port通过MsgD告诉adapter当前slot允许的最大功率；Controller传递额度，device仍需真正遵守 |
| LTR | Latency Tolerance Reporting | EP向RC报告Memory traffic可容忍的service latency，帮助平台决定中央资源能睡多深；不是延迟测量值或CTO |
| OBFF | Optimized Buffer Flush/Fill | 平台提示EP何时集中DMA/IRQ的增量唤醒成本较低；只是优化hint，不禁止窗口外traffic |
| PME | Power Management Event | Function 请求唤醒/报告电源事件的机制 |
| Hot Plug | PCIe Hot Plug | 由Downstream Port的Slot Capability/Control/Status和外部电源/检测逻辑支持运行中插拔；orderly与async removal恢复条件完全不同 |
| HPS | Hot-Plug Surprise | 旧式async removal处理机制；PCIe 5.0已不推荐其用于async hot-plug，优先采用DPC containment |
| ATS | Address Translation Services | 允许设备请求并缓存 IOMMU 地址转换结果，减少每次 DMA 的翻译开销 |
| PASID | Process Address Space ID | 给事务附加进程地址空间身份，让设备共享多个进程虚拟地址上下文 |
| PRI | Page Request Interface | 设备遇到缺页/权限问题时向系统请求页面服务的机制 |
| SR-IOV | Single Root I/O Virtualization | 一个 Physical Function 暴露多个轻量 Virtual Functions，便于直接分配给虚拟机/容器工作负载 |
| ARI | Alternative Routing-ID Interpretation | 扩展 Function 编号解释，使一个 Device 下可表达更多 Functions；会改变传统 BDF 直觉 |
| FLR | Function Level Reset | 只复位一个 Function 的标准机制，尽量不影响同设备其他 Functions |
| PF / VF | Physical / Virtual Function | SR-IOV 中负责管理的完整 PF 与轻量、可独立分配的 VF |
| Multifurcation | Multifurcation | 把一组物理 Lane 静态分给多条独立 Link；不是 Link training 自动降宽 |
| NTB | Non-Transparent Bridge | 在两个独立Host/PCI hierarchy间终止彼此enumeration，再用Memory Window、地址翻译、doorbell和scratchpad建立通信的桥 |
| Memory Window (NTB) | NTB Memory Window | 把本地BAR aperture的访问翻译到peer address domain/backing buffer；两侧地址不需相同 |
| Doorbell (NTB) | NTB Doorbell | 用bit/中断通知peer发生事件；不搬数据，也不自动保证Posted Write已被peer消费 |
| LM | Local Management | Cadence 私有本地管理寄存器空间，控制/观察 core 状态，远端 Config TLP 不可见 |
| W1C | Write 1 to Clear | 向状态位写1清除；对这类寄存器做普通read-modify-write可能顺手清掉邻近事件，debug前应先保存日志 |
| ATU / address translation | Address Translation Unit | 在 CPU/AXI 与 PCIe address domain 间匹配窗口并重写地址/TLP属性；具体厂商名称不一定叫 iATU |
| L0/L0s/L1/L2 | Link Power States | LTSSM/链路电源状态；L0 正常传输，L0s/L1 逐步省电，L2 接近关闭；不能和 D0/D3 混为一谈 |

## 主要来源

- PCIe Base Specification 5.0 r1.0, Sections 1.2–1.5, printed/PDF pp.90–102。
- Cadence PCIe Core User Guide, Sections 2.1, 4.1, printed pp.6, 202–204 / PDF pp.95, 291–293。
- 后续术语会随对应主题文档加入更精确的章节定位。
