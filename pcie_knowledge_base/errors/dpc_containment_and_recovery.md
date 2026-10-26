# DPC：下游错误隔离与恢复

## 1. DPC 解决什么问题

DPC（Downstream Port Containment）是 Downstream Port 的可选规范能力。下游发生严重错误时，Port 主动阻止错误 traffic 继续污染上游，并把相关 Link 导向 Disabled。直觉上像机房某路电气故障后先跳闸隔离，不是在火花中坚持“服务可用性优先”。

DPC 与 AER 的分工：AER 记录/报告错误；DPC 执行 containment。两者经常一起工作，但“有 AER”不等于“实现 DPC”。

## 2. 触发后的硬件后果

```text
uncorrectable/fatal or configured trigger
 → Downstream Port DPC trigger/status/source capture
 → block/terminate downstream traffic
 → LTSSM directed to Disabled
 → outstanding requests按规则完成/终止
 → DPC interrupt/error signaling
 → software收集证据、reset/retrain/rescan/recover
```

因此 DPC 后看到 Link Disabled 是 containment 结果，不能只按普通 Link Not Up 重新训练并清光状态。应先读 DPC/AER status、source ID、header/prefix log。

## 3. 它和 Surprise Down/Hotplug 的区别

- Surprise Down：检测到 Link 非预期消失，是现象/错误类型。
- DPC：Port 主动隔离下游，是机制；可由多类错误触发。
- Hotplug remove：软件/用户有管理语义的设备移除流程。

一条日志可同时出现 AER fatal、DPC containment、Link down。时间顺序决定谁是根因、谁是保护动作。

## 4. Linux 恢复思路

1. freeze/quiesce 下游 driver I/O；
2. 保存 DPC/AER/Root Port config 与 topology；
3. 判断设备是否仍物理存在；
4. reset containment domain；
5. retrain Link；
6. 恢复/重新分配下游 configuration/resource；
7. driver slot reset/resume 或移除设备。

X6000 Linux 5.10.4 的通用 DPC service 位于 `drivers/pci/pcie/dpc.c`，由 Port service 框架分配 IRQ、读取 DPC Status/Source ID，并与 AER recovery 协作。外部 Host 是否启用 DPC native ownership 仍由其 firmware 和内核策略决定。无论由谁处理，都不能只清 DPC status 让中断消失，因为下游 outstanding DMA 和 driver state 仍可能冻结。

## 5. Cadence 证据边界

当前 Cadence 目录/寄存器搜索没有发现 DPC Extended Capability 或专用 DPC register block。手册错误章节会提及 DPC 语义，但这不足以证明本 build 实现 DPC。因此：

- 标准 DPC 协议知识：Confirmed by Base Spec；
- 当前 Cadence build 的 DPC capability：**Not evidenced / likely not implemented，需实际 capability dump 确认**；
- Cadence Link Down Reset 仍可能由其他 link-down/hot-reset 事件触发，不能把每次 `LINK_DOWN_RESET_OUT` 都称作 DPC。

## 6. Debug checklist

- DPC capability 是否真实存在、Trigger Enable/Reason/Status；
- 首个 AER error 与 severity；
- RP PIO/header/prefix log、Requester ID；
- Link 是错误先 down 还是 DPC 后被导向 Disabled；
- downstream outstanding NP/DMA 如何终止；
- reset 后 bridge bus/resource 与 IOMMU/IRQ 是否重建；
- 恢复失败时保留原状态，避免无限 clear-retrain loop。

## 7. Source Reference

- PCIe Base Specification 5.0 r1.0, §§2.9.3、6.2.10、DPC Extended Capability in Chapter 7。
- Cadence PCIe Core User Guide, §§4.7.2–4.7.3，用于区分 Cadence Link Down Reset；当前未发现 DPC implementation evidence。
