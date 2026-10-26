# MindShare Chapter 24：Express-specific Configuration Registers OCR评估

日期：2026-08-23

## 1. 阅读范围

对`PCI.Express.System.Architecture.pdf` PDF pp.949–1014附近逐页OCR，深读Chapter 24：

- PCI Express Capability；
- AER Extended Capability；
- Virtual Channel Extended Capability；
- Device Serial Number；
- Power Budgeting；
- Root Complex Register Block（RCRB）。

OCR cache：`/tmp/pcie1_arch_scan_cfg2/page-0949..1014.txt`。这是2003年对PCIe 1.x-era configuration surface的详细快照，不是PCIe 5.0 capability全表。

## 2. 仍然非常有用的主线

### 2.1 PCI Express Capability把一个Function的角色、负载和Link暴露给软件

Capability header的Device/Port Type让software区分Endpoint、Root Port、Switch Upstream/Downstream Port、RCiEP等角色。后续Device registers描述MPS、Tag、error reporting等Function行为；Link registers描述speed/width/ASPM/training；Slot registers描述hotplug和indicator；Root registers汇聚PME/error。

“同一个Capability里字段是否有效依赖Device/Port Type”是一条稳定软件规则。不能对Endpoint硬解码Root Status，也不能看到Slot register offset就假定这是slot-implemented Downstream Port。

### 2.2 supported咎enabled是两层

Device Capabilities中的Max Payload Size Supported是硬件上限，Device Control.MPS是软件实际选择。Link Capabilities中有Maximum Link Speed/Width，Link Status中是当前协商值。这种“capability上限≠当前运行状态”的阅读方法，已用于MPS/MRRS、ASPM、MSI-X等知识库正文。

### 2.3 AER不是一个“总错误bit”

教材把Correctable Status/Mask、Uncorrectable Status/Mask/Severity、First Error Pointer、ECRC control、Header Log和Root Error registers连成了一条证据链：

```text
error source latches status
  → mask决定是否向上报
  → severity决定Non-Fatal/Fatal
  → First Error Pointer/Header Log保留关键上下文
  → ERR_COR/ERR_NONFATAL/ERR_FATAL Message
  → Root Port Root Error Status/Source ID
  → platform interrupt/software recovery
```

这正是 [Error model、AER与Cadence](../errors/error_model_aer_and_cadence.md) 中使用的主线。

### 2.4 VC Capability是配置TC→VC与arbitration的表面

Port VC Capability、VC Resource Capability/Control/Status和arbitration table将“这个Port有几个VC、支持哪种仲裁、哪些TC映射到该VC、是否完成negotiation”暴露给software。详见 [TC、VC、QoS、PTM与ACS](../advanced/tc_vc_qos_ptm_and_acs.md)。

### 2.5 RCRB是RC内部逻辑的4 KiB MMIO寄存块

RCRB不是一个可枚举Endpoint Function，也不是ECAM。它是firmware向OS提供base address的Root Complex特有MMIO register block，可承载VC等Extended Capability，且访问应避免跨dword/misaligned/locked操作，除非software确知该RC支持。

PCIe 5.0仍定义RCRB，不能因为它在2003年书里就把它判成过时。现代规范还定义RCRB Header Extended Capability（ID `0x000A`）来提供Vendor/Device ID、capabilities/control和association信息。

## 3. 必须纠正的年代差异

### 3.1 PCI Express Capability已经不止教材画的那一页

PCIe 5.0的version 2 Capability包含Device Capabilities/Control/Status 2、Link Capabilities/Control/Status 2和Slot Capabilities/Control/Status 2等扩展。教材只给出初代structure，其中的8-bit Extended Tag也不是Tag扩展的终点：PCIe 4.0后还有10-bit Tag Requester/Completer能力和enable。

### 3.2 Extended Capability绝不只有四种

教材只列AER(`0001h`)、VC(`0002h`)、DSN(`0003h`)和Power Budgeting(`0004h`)。PCIe 5.0 Ch.7还包含ACS、ARI、ATS、SR-IOV、LTR、Resizable BAR、DPA、TPH、L1 PM Substates、PTM、DPC、PASID、PRI等。正确软件行为是对未知ID继续走next pointer，而不是报错停止。

### 3.3 AER bit也随版本扩展

2003年的AER图已有Receiver Error、Bad TLP/DLLP、Replay Rollover/Timeout、Poison、Flow Control Protocol Error、Completion Timeout/Abort、Unexpected Completion、Receiver Overflow、Malformed TLP、ECRC和UR。后续规范又增加Surprise Down、ACS Violation、Uncorrectable Internal Error、MC Blocked TLP、AtomicOp Egress Blocked、TLP Prefix Blocked等条目与新log/control。用旧bit map解码Gen4/Gen5 dump可能会把真正的错误当成Reserved。

### 3.4 DSN是识别信息，不是信任根

Device Serial Number Capability是可选的64-bit只读标识。它可用于asset/device identification，但仅凭DSN无法证明设备或firmware未被伪造；安全身份需要额外authentication/attestation机制。

## 4. 当前Cadence build的映射

- PCIe Capability/AER有实际PF/VF/RP register map，但需按对应Function type和configuration-specific capability list使用；
- PF0实现Power Budgeting，其默认power data是需要integrator改写的template，不是真实板卡功耗；
- 当前configuration-specific tables没有额外VC Capability，只能依赖TC0/VC0基线；
- 手册未提供目标SoC firmware的RCRB地址描述，不能从通用PCIe定义反推出实际MMIO base。

## 5. Source Reference

- MindShare *PCI Express System Architecture*：Ch.24 printed pp.893–959左右，约PDF pp.949–1014。
- PCI Express Base Specification 5.0 r1.0：§7.2.3 RCRB；§7.5.3 PCI Express Capability；§7.6–7.9 Extended Capabilities；§7.9.7 RCRB Header。
- Cadence PCIe Core User Guide：Chapter 9 configuration register blocks；Appendix F；Appendix H configuration-specific capability lists。
- 关联正文：[Capability discovery](../configuration/capability_discovery_and_safe_walking.md)、[MPS/MRRS](../transactions/mps_mrrs_ordering_and_performance.md)、[Power Budgeting](../power/slot_power_limit_and_power_budgeting.md)、[TC/VC/QoS](../advanced/tc_vc_qos_ptm_and_acs.md)。
