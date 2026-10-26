# Feature Coverage Audit 2

日期：2026-08-23

## 本轮新增/收口

- Message TLP：routing 与 Cadence outbound Message aperture。
- AtomicOp：协议主线，并确认当前 Cadence configuration 明确不支持。
- TLP Prefix/TPH：PASID/Steering Tag 与 AER Prefix Log 关系。
- DPC：标准 containment/recovery，并明确 Cadence build 缺实现证据。
- Resizable BAR：已并入BAR专题，覆盖supported-size、quiesce、resource重新分配、Cadence PF enable/aperture联动。
- TC/VC/QoS：已独立解释；确认当前 Cadence configuration 只有 TC0/VC0 基线，不实现可配置 VC Capability Registers。
- PTM：已完成标准机制和 Linux API surface；Cadence 通用 capability 图虽画出 `0xA20`，但当前 configuration-specific tables 未选中。
- ACS：已完成隔离/redirect 主线；当前 Cadence configuration 未暴露 ACS Extended Capability。

## 已覆盖但不单独扩文档

- ECRC：已在 AER/Data Link 文档区分 LCRC 与 end-to-end ECRC。
- ACS：已在高级专题补齐 source validation、translation blocking 与 P2P redirect 主线。
- TPH：与 Prefix/optional transaction 合并。

## 后续完成项与剩余边界

- DPA已独立成文，并发现Cadence默认allocation table违反规范单调性，必须由integrator定制；
- Power Budgeting/Slot Power Limit已独立成文，并识别Cadence 240 W默认记录为需定制模板；
- Lane Margining已在Link/Controller编程中覆盖；controller command/PIPE边界可确认，PHY内部算法仍被缺失PHY Databook阻塞。

## 资料年代边界

Base Spec 为 5.0，Cadence core 为 Gen4；后于 PCIe 5.0 或手册发布日期的 DOE、IDE/CMA 等不能从当前资料推定支持。本知识库不会为了“看起来新”从互联网拼入无项目证据的功能。

## 本轮重要审计方法

Cadence Appendix H Figure H.1 是 IP 的通用 capability linked-list implementation，不能单独证明当前实例选中了某能力。必须继续检查 H.1 的 configuration-specific tables，并与 Chapter 9 register block 及 tied-off status 交叉验证。VC、PTM 和 ACS 正是通过这一步从“图中出现”收敛为“当前配置未选中”。
