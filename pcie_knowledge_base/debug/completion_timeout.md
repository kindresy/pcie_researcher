# Completion Timeout：请求发出后没有按时收到完整 Completion

## 1. 症状的准确含义

CTO 只针对需要 Completion 的 Non-Posted Request。timer 在 request transmitted 时启动；多条 split CplD 必须全部收齐才完成。Memory Write 没有普通 Completion，因此不会以“等不到 MWr Completion”形成 CTO。

## 2. 逐段追踪

```text
Requester request queue/tag
 → outbound address translation/routing
 → each Link Ack/replay
 → Completer decode/BAR hit
 → local AXI/target response
 → Cpl/CplD generation
 → return routing
 → Requester ID+Tag match
 → completion reassembly
```

每一段都可能让最终表现成为 CTO。

## 3. 检查清单

1. AER CTO bit 在谁的 Function 上？它就是 Requester，不要先去错误设备找 timer。
2. Header Log 是否支持/记录原 Request；解 address、Requester ID、Tag、Length、BE。
3. Request TLP 是否真的出 Link；Data Link Ack 只能证明下一跳收到。
4. 地址是否穿过正确 bridge window/ATU，最终 BAR/function 是否命中。
5. Completer 本地 AXI/APB 是否 hang；正常失败应尽量返回 CA/UR，而不是不回应。
6. 是否生成 Completion；Completer ID、Requester ID、Tag、byte count 是否正确。
7. return path 是否完整；是否被当 Unexpected Completion 丢弃。
8. Link Down/FLR/reset 是否清掉 outstanding context；Cadence 会把相关 AXI 请求以 error 终止。
9. Completion Timeout Value 是否被异常缩短；不要用关闭 CTO 掩盖硬件死锁。

## 4. Cadence 观察点

- split completion table entry/tag 与 timeout status；
- AER Uncorrectable CTO bit；
- `INTERRUPT_SIDEBAND_SIGNALS[21]`；
- Completion RX FIFO、Unexpected Completion、Requester/TC/Attr/address mismatch；
- AXI slave `RRESP/BRESP`；
- Link Down Indication、FLR in progress；
- Local Management Completion Timeout Limit registers（实现细节需结合该 Link register block）。

## 5. Source Reference

- Base Spec 5.0 §2.8，printed/PDF p.204；§6.2；§7.8.4。
- Cadence §4.5.6.3.1、Appendix B、Completion Timeout Limit register descriptions。
