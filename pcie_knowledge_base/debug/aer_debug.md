# AER Debug：先留现场，再清 W1C

## 1. 推荐采集顺序

1. 记录时间、BDF、Port topology、是否伴随 Link reset/down；
2. 读 Root Error Status 与 Error Source ID；
3. 读出错 Function/Port 的 Uncorrectable/Correctable Status；
4. 保存 Mask、Severity、Device Control reporting enable；
5. 保存 First Error Pointer、Header Log/TLP Prefix Log；
6. 保存 Cadence Local Error Status、sideband、LTSSM history、AXI response；
7. 关联 kernel log 与业务 transaction；
8. 最后按 RW1C/W1C 语义清状态，并确认是否立即复发。

## 2. 解码不要只翻译 bit 名

为每个 bit 回答：

- 谁检测：Requester、Completer、中间 Receiver、Port 还是 controller RAM？
- 哪一层：PHY、Data Link、Transaction、Link state、internal？
- packet 是否被丢弃、replay、poison-forward、返回 UR/CA？
- AXI client 看见 `OKAY`、`SLVERR` 还是根本没有 transaction？
- error message 是否被 mask、severity 是什么、本地 sideband 是否另有通知？
- 恢复对象是单 transaction、Function、Link 还是整棵 hierarchy？

## 3. 常见组合

- `Receiver Error + Bad TLP + Replay`：优先查该 Link 信号/PHY；后两者可能是上层后果。
- `UR + Header address 落在 BAR 外`：查 BAR/window/translation/MSE。
- `CA + EP AXI SLVERR`：查 Completer 内部目标；PCIe path 可能是健康的。
- `Unexpected Completion` 随后 `CTO`：completion context/tag/routing 不匹配。
- `Surprise Down + 多个 CTO`：Link loss 是根因，CTO 是在途请求后果。
- `Correctable 持续洪泛`：虽然能 replay，也应统计 rate、lane、speed；降速/降宽做 SI 对照。

## 4. 清除陷阱

- Status 常为写 1 清，普通 read-modify-write 可能把同时为 1 的旧 bit 全清掉；严格按 register 语义写目标 mask。
- Mask 不会清既有 Status，也不修复产生错误的硬件。
- 先清 Function AER、后清 Root，还是相反，取决于事件仍否活跃与平台 ISR 流程；操作前保存两端快照。
- 如果错误条件持续存在，清完立即置回是线索，不是“寄存器清不掉”。

## 5. Source Reference

- Base Spec 5.0 §§6.2.3–6.2.7、7.8.4。
- Cadence Appendix B、§4.6.2、Appendix A/C。

