# Enumeration Failure：Link 已经亮了，Linux 为什么还看不见设备

## 1. 先把“枚举失败”切成四段

```text
Physical L0 / DL_Up
  → RP 能生成正确 Config Request
  → 请求路由到目标 Function，并返回 Completion
  → PCI core 扫描、分配资源并创建 pci_dev
```

`lspci` 没有 BDF，只说明最后没得到一个可用的 `pci_dev`。它不能区分没发请求、请求地址错、EP 回 CRS，还是桥的 bus number 把路封了。

## 2. 最小探针：Vendor ID read

枚举器通常从目标 BDF offset `0x00` 读 Vendor/Device ID。观察结果：

| 结果 | 协议含义 | 下一步 |
|---|---|---|
| 合法 Vendor ID | Function 存在，继续读 header/capability | 查后续资源或 capability access |
| 全 1 / master abort 风格值 | 没有 Function 或 host 把失败折算成全 1 | 查 config TLP/Completion |
| CRS | Function 尚未准备好 | 等待/重试，查 EP `CONFIG_ENABLE` |
| UR | BDF/offset/type/routing 不受支持 | 查 Type 0/1 与 bus routing |
| CTO | Request 发出但没有完整 Completion | 转 [Completion Timeout](completion_timeout.md) |

Cadence EP 的 `LINK{N}_CONFIG_ENABLE=0` 时会对 Configuration Request 返回 CRS。这是“我还在准备”，不是“我不存在”。固件完成 PF identity、BAR/capability 等初始化后应置 1。

若Root Port启用CRS Software Visibility，读取Vendor ID时Host可能看到保留值`0001h`，表示设备用CRS回答、尚未准备好；它不同于常见的`ffffh`不存在/访问失败。Base Spec 5.0 §6.6仍要求常规reset后给设备最多1.0 s成功响应，且对支持高于5.0 GT/s的端口，首次100 ms等待与Link training完成时点相关。

## 3. 从 RP 向下逐层检查

1. Root Port 自己是否出现在 Linux，`LnkSta` 是否 Link up。
2. Primary/Secondary/Subordinate Bus Number 是否覆盖目标 bus。
3. Cadence outbound Config region 是否存在，descriptor 是否为 Config TLP。
4. BDF 编码采用 register method 还是 AXI-address method；两者 region 最小分别为 4 KiB 和 256 MiB。
5. 下游直连 EP 应使用 Type 0；跨桥继续向下路由使用 Type 1。
6. 抓 Config Read：Bus/Device/Function、register number 是否正确。
7. EP 是否命中 Function并返回 CplD；Completion 的 Requester ID/Tag 是否能匹配。

## 4. Linux 侧分诊

```bash
lspci -D -t
lspci -vv -s <root-port-bdf>
dmesg -T | grep -iE 'pci|pcie|link|resource|aer'
```

- Root Port 不存在：先查 platform host driver/host bridge 注册。
- Root Port 存在、Link down：转 [Link Not Up](link_not_up.md)。
- Link up、下游 bus 不存在：查 bus number/config access。
- BDF 存在但 driver 不 probe：这已不是枚举失败，查 ID table、modalias、driver binding 和 probe return。
- BDF 存在但 BAR 未分配：转 [BAR Problem](bar_problem.md)。

## 5. 典型误区

- 用 BAR MMIO read 测枚举：BAR 尚未分配时这条路本来就没通。
- 只看 LTSSM=L0：Data Link credit/config routing 仍可能坏。
- 把 CRS 当 UR：CRS 允许软件重试，语义完全不同。
- 在桥后设备使用错误 Config Type：BDF 数字对了也到不了。

## 6. 必留证据

拓扑树、Root Port config dump、bus-number registers、Cadence Config region 四元组（AXI base/size、PCI target/BDF、descriptor type）、首个失败 Config TLP 与 Completion、EP `CONFIG_ENABLE`、kernel 首次扫描日志。

## 7. Source Reference

- PCIe Base Specification 5.0 r1.0, §§2.2.7–2.2.9、2.3、7.5。
- Cadence PCIe Core User Guide, `CONFIG_ENABLE` signal；§§10.3.1–10.3.2。
- [Configuration Space、ECAM 与路由](../configuration/configuration_space_ecam_and_routing.md)。
