# MindShare Appendices A–E OCR评估

日期：2026-08-23

## 1. 阅读范围

对`PCI.Express.System.Architecture.pdf` PDF pp.1015–1106逐页OCR，评估Appendix A–E与Index。OCR cache：`/tmp/pcie1_arch_scan_cfg2/page-1015..1106.{jpg,txt}`。

| 附录 | 处理级别 | 知识库影响 |
|---|---|---|
| A Test, Debug and Verification | D+M | 新增protocol analyzer分层/双向/trigger手册；剔除Gen1专有定量值 |
| B Markets & Applications | E | 评估历史市场预测；无新的规范/寄存器结论 |
| C Intelligent Adapters & Multi-Host | D+M | 新增NTB/multi-host正文，并映射Linux 6.8 NTB API surface |
| D Class Codes | E | 2003/PCI 2.3-era code表已评估；不复制过时数值表，现场应查当前PCI Code and ID Assignment |
| E Locked Transaction Series | D+M | 并入Message/Atomic/Prefix正文，用Base Spec 5.0 §6.5校正 |
| Index | E | 用于反向主题漏项检索，不当作独立章节 |

## 2. Appendix A：方法论很稳，仪器和电气数字已经过时

仍有价值的结论：PCIe差分串行比特流不能用普通低速logic analyzer解释；必须同时抓两个dual-simplex方向，将Physical ordered set、DLLP和TLP分层解码，再用方向+packet type+address/ID+event sequence设置trigger。Exerciser可用于注入错误/边界条件，不只是被动抓包。

需要封存的数量：教材只有Gen1 2.5 GT/s/8b/10b、早期Catalyst analyzer和probe、以及某个时代的trace length/via/jitter示例。它们只能解释原理，不是Gen4/Gen5 CEM/PHY/board compliance limit。

## 3. Appendix B：历史背景，不当规范

本章预测PCIe会进入desktop/server/storage/communications/embedded市场，并用northbridge/southbridge、2.5 GHz和“5 GHz being considered”的语境解释低pin count、差分串行、分层packet、QoS和PCI software compatibility优势。它能说明设计动机，不能用来解码当代SoC topology、speed或market requirement。

## 4. Appendix C：NTB是一条新知识分支

附录解释了为什么multi-host/failover/intelligent adapter不能让两个Host同时透明enumerate对方。NTB向每侧呈现Type 0 boundary，再用BAR aperture/translation、mailbox、doorbell和shared memory建立显式通信。

已形成 [Non-Transparent Bridge与multi-host](../advanced/non_transparent_bridge_and_multi_host.md)，并核对Ubuntu 6.8 `include/linux/ntb.h`的memory window、doorbell、scratchpad/message和link operations。Cadence guide未暴露NTB feature，不把uDMA Doorbell误当peer doorbell。

## 5. Appendix D：Class Code是driver matching hint，旧表不是永久真理

Class/Subclass/Programming Interface可让software在没有vendor-specific driver时识别通用类型，也用于选class driver或判断bridge/storage/network/display等大类。但Appendix D复制的是PCI 2.3时代assignment，已经缺少后来设备类型。本知识库只保留`Base Class/Subclass/ProgIF`的分层意义，数值应查当前PCI-SIG PCI Code and ID Assignment。

## 6. Appendix E：Lock在PCIe 5.0仍在，但只为legacy负责收尾

本章的MRdLk→CplDLk→MWr→Unlock主线与Base Spec 5.0 §6.5一致：只有RC代Host CPU发起，只用TC0/VC0，Switch/PCIe-to-PCI bridge锁住相关path，原生PCIe Endpoint收到MRdLk应返UR。新software不应对I/O target发locked access，它是legacy PCI兼容机制，不是AtomicOp替代品。

已并入 [Message、AtomicOp与TLP Prefix](../transactions/message_atomic_and_tlp_prefix.md)。

## 7. Source Reference

- MindShare *PCI Express System Architecture*：Appendix A printed pp.961–987；B pp.989–997；C pp.999–1017；D pp.1019–1031；E pp.1033–1042；Index pp.1043–1049。
- PCI Express Base Specification 5.0 r1.0：§6.5 Locked Transactions；Ch.2–4 layered packet/link behavior；Ch.7 configuration fields。
- Local Ubuntu 6.8 generated headers/Kconfig：`include/linux/ntb.h`、`drivers/ntb/Kconfig`、`drivers/ntb/test/Kconfig`。
