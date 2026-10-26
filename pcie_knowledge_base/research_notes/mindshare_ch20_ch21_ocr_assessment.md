# MindShare Chapter 19–21：Configuration / Enumeration OCR评估

日期：2026-08-23

## 1. 阅读范围

对 `PCI.Express.System.Architecture.pdf` PDF pp.769–826附近逐页OCR，深读：

- Chapter 19 Configuration Overview；
- Chapter 20 Configuration Mechanisms；
- Chapter 21 PCI Express Enumeration。

OCR cache：`/tmp/pcie1_arch_scan_cfg/page-*.txt`。教材为2003年早期PCIe资料，配置基本模型仍很有价值，但时序、multi-root和新能力必须与Base Spec 5.0校正。

## 2. 教材讲清楚了什么

### 2.0 上电时主机并不知道树上挂了什么

Chapter 19从一个很有用的反常识开始：初始时配置软件只知道host bridge和根bus的存在，尚不知道Root Port、Switch、Endpoint的数量和连接关系。Enumeration不是读一张设备自己上报的“拓扑表”，而是从已知入口开始，用Configuration Read逐层探测并为bridge分配bus number。

Chapter 19还把三件容易混在一起的事拆开了：

- 每个Function各自实现4 KiB Configuration Space，前256 B是PCI-compatible header/capability区，后3840 B是Extended Configuration Space；
- 配置软件运行在主机CPU上，Root Complex代它在PCIe fabric中生成Configuration Request；
- 设备的存在性通常由Vendor ID读取探测，Header Type再告诉软件这是Type 0 Function还是Type 1 bridge。

这个“主机发问，设备回答”模型非常重要：Endpoint不会在Link Up后自己跳出来向Linux报到。

### 2.1 CF8/CFC与ECAM最终都只是CPU侧入口

旧x86 PCI-compatible mechanism先写`0xCF8`选择BDF/register，再读写`0xCFC`；Enhanced Configuration mechanism把每个Function 4 KiB空间放进memory-mapped aperture：

```text
base + bus[27:20] + device[19:15] + function[14:12] + offset[11:0]
```

一个传统segment覆盖256 buses、32 devices、8 functions、每Function 4 KiB，因此窗口是256 MiB。两种CPU入口最后都由Root Complex转成Configuration Request；它们不是两套EP配置空间。

现代系统还可有多个PCI segment/domain，每个segment各有bus 0–255及ECAM描述。教材的单个256 MiB图不要外推成“全系统最多一个ECAM窗口”。

### 2.2 Type 1沿桥传播，到目标bus转Type 0

教材逐Hop演示：target bus还在更深处时，bridge按Secondary–Subordinate aperture转发Type 1；当target bus等于某bridge的Secondary Bus时，最后一跳转换为Type 0，由该bus上的Device/Function decode。

这条因果链解释两个常见现象：

- bridge bus number没配好时，EP即使Link Up也收不到Config Request；
- Type 0/Type 1不是“Endpoint header/Bridge header”的另一种叫法，而是Configuration TLP routing类型。

### 2.3 Enumeration用Vendor ID探存在，再递归建树

Chapter 21的例子按Device 0–31读取Function 0 Vendor ID：`0xffff`表示Function 0不存在，且传统设备要求若有Function则Function 0必须存在；有效ID后再看Header Type/multifunction和PCIe Device/Port Type。

遇到bridge就写Primary/Secondary/Subordinate并进入下游bus，完成该分支后回退。这个depth-first故事很好地解释了为什么Bus Number是枚举结果，不是EP出厂序列号。

教材称早期spec要求depth-first。Base Spec 5.0主要规范配置路由和组件行为，并不适合据此断言某个现代OS的精确扫描函数/顺序；Linux实现仍需目标source tree确认。知识库保留“递归建立hierarchy”的稳定概念，不伪造版本调用链。

### 2.4 一笔Config Write还把身份告诉Function

教材的“Memorize Your Identity”与Base Spec 5.0交叉验证一致：Function从每笔完成的Type 0 Config Write捕获Bus/Device Number，用于Requester/Completer ID；hotplug renumber后，software必须再次向每个Function完成Config Write以刷新身份。

首次Type 0 Config Write前，Function没有可供Completion返回的有效Requester ID，因此不得发Non-Posted Request。ARI只捕获Bus Number，SR-IOV还有跨Bus VF规则。

## 3. 需要校正的地方

### 3.1 教材有一处posted/non-posted编辑错误

Chapter 21在解释首次ID assignment前限制时，写成不允许发“non-posted requests (memory write requests and message requests)”。括号里的Memory Write/普通Message恰恰是Posted类型，与句首矛盾。

Base Spec 5.0 §2.2.6.2的准确规则是首次Config Write前不得发**Non-Posted Requests**，原因是返回Completion需要有效Requester ID。不能把教材括号错误复制进正文。

### 3.2 CRS时序已比2003模型更细

教材给出reset后至少100 ms再配置、1.0 s内设备须能成功响应，以及CRS重试。Base Spec 5.0仍保留100 ms/1.0 s主线，但增加关键条件：

- 对支持>5.0 GT/s的Downstream Port，100 ms从Link training完成后计算；
- 若使用Readiness Notifications，等待可缩短；
- software最好只在CRS Software Visibility enable时采用100 ms尝试，否则可能造成processor stall/platform timeout；
- CRS SV enable时，读Vendor ID可向Host呈现保留值`0001h`，其他配置访问仍由RC重试。

Debug时`Vendor ID=0001h`与`ffffh`语义不同：前者可表示CRS software-visible“设备还没准备好”，后者通常是不存在/失败。

### 3.3 multi-root例子不能替代现代PCI segment模型

教材用bus 0与bus 255方向描述多个Root Complex的例子，适合展示host bridge如何选择目标bus，却不是现代ACPI/DT/Linux PCI domain的完整模型。segment/domain不进入16-bit Requester ID；跨segment的host routing和资源由平台描述。

### 3.4 Enumeration不等于resource allocation已经结束

Chapter 21重点是bus/device发现。真实系统还要probe BAR size、放置resource、编程bridge Memory/I/O windows、启用MSE、遍历capability并绑定driver。只看到BDF出现不能判定MMIO/DMA/MSI已经可用。

### 3.5 2003年的Extended Capability清单只是当时快照

Chapter 19只列出AER、VC、Device Serial Number和Power Budgeting等当时已定义能力。PCIe 5.0已有ACS、ARI、ATS、SR-IOV、LTR、Resizable BAR、DPC、PASID、L1 PM Substates等更长的Extended Capability列表。教材的图适合讲“前256 B+后3840 B”布局，不能拿来判定现代硬件应支持哪些能力。

## 4. 对正文的影响

- ECAM公式、Type 1→Type 0逐Hop路由和BDF capture得到独立教材交叉证据；
- enumeration debug补充`Vendor ID=0001h`的CRS SV含义；
- 不采用教材中Memory Write/Message被误列为Non-Posted的错误；
- 不把2003 multi-root与depth-first叙述冒充现代Linux精确实现；
- worked example中的ECAM地址和bus递归获得支持。

## 5. Source Reference

- MindShare *PCI Express System Architecture*：Ch.19 printed pp.713–720，约PDF pp.769–776；Ch.20 printed pp.723–742，约PDF pp.779–798；Ch.21 printed pp.743–768左右，约PDF pp.799–826。
- PCI Express Base Specification 5.0：§2.2.6.2 Requester ID capture；§2.2.9 Completer ID；§2.3.1 CRS；§6.6 reset/readiness。
- 相关正文：
  - [Configuration Space、ECAM与路由](../configuration/configuration_space_ecam_and_routing.md)
  - [BDF、Requester ID与层次路由](../configuration/bdf_requester_id_and_hierarchy_routing.md)
  - [Enumeration failure](../debug/enumeration_failure.md)
