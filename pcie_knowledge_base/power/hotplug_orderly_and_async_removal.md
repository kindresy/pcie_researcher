# PCIe Hot Plug 与异步移除：有序下线和突然断开

## 1. 两种“设备不见了”必须先分开

### Orderly removal

用户先按Attention Button或通过软件请求移除，OS有机会：

1. 阻止新I/O；
2. quiesce DMA与中断；
3. 完成/终止outstanding requests；
4. unbind driver、释放资源；
5. disable Link；
6. 关闭slot power并用指示灯告诉用户可以拔卡。

### Async removal

卡、线缆或 Downstream Link 在 OS 同意前已经消失。此时在途 MRd 可能无法收到 Completion，Posted MWr 是否到达也可能无法判断；Requester 会看到 CTO，Port 可能记录 Receiver Error、Bad TLP、Bad DLLP 或 Surprise Down。软件只能在事件发生后进行 containment 与恢复，无法补做移除前本应完成的 quiesce。

早期教材称presence pins会先断开，给software“足够时间”在其他触点断开前关电。现代Base Spec 5.0 §6.7.6对async removal的定义恰恰强调：hierarchy和OS不一定提前知道，必须按已经发生的异步事件处理。那句2003年的乐观描述不能作为驱动安全保证。

## 2. Hot Plug 为什么属于 Downstream Port

每个可热插拔slot由相邻Root Port或Switch Downstream Port控制。PCIe点到点拓扑使每个slot有独立Port，不需要像并行PCI那样为整条共享bus加一套隔离开关。

标准toolbox包括：

- Presence Detect；
- Attention Button；
- Power Fault；
- Manually-operated Retention Latch（MRL）sensor；
- power controller；
- Attention/Power indicator；
- electromechanical interlock；
- Slot Capabilities/Control/Status；
- Command Completed与Data Link Layer State Changed event。

不是所有系统都装齐这些零件。Capability bit说明有什么，Control bit发命令，Status bit保存发生过什么。看到`Slot Implemented=1`也不等于一定有Attention Button、电源开关和MRL。

## 3. 一次安全拔卡的协议/硬件链

```text
Attention Button / software request
 → Slot Status event + hot-plug IRQ
 → OS验证请求
 → Function driver停止queue、DMA、IRQ
 → PCI core解除driver与resource使用
 → Downstream Port Link Disable
 → Slot Control关闭power / indicator
 → Command Completed + 状态确认
 → 用户得到“可以拔”指示
```

写Slot Control后不能立刻假定继电器已经动作。Hot-plug command可能异步完成；支持Command Completed的实现用status/interrupt确认。现代规范要求单条command相关动作不超过1秒；power-off之后，software还要等待规定时间再执行依赖电源已移除的动作。

安全插入基本反向：检测presence、上电、refclk、释放PERST#，等待LTSSM与DL_Up，再枚举/分配资源/bind driver。**Presence=1不是可以读Vendor ID的同义词**；中间还有power、reset、training和Data Link initialization。

## 4. Event 与 interrupt 为什么是sticky + level思维

Slot Status里的Attention Button、Power Fault、MRL Changed、Presence Changed、Command Completed、DLL State Changed各有status，Slot Control有对应enable，另有Hot-Plug Interrupt Enable总开关。

status是pending事件，通常RW1C。INTx模式只要“master enable + 某status与enable同时为1”就保持assert；MSI/MSI-X则在该逻辑从false变true时发送消息。ISR应：

1. snapshot所有status；
2. 处理/排队work；
3. 对确认过的bit写1清除；
4. 再读一次，处理清除期间新到的事件。

只清irq controller不清Slot Status，会让level interrupt像不肯下班的同事一样再次出现；一把写全1又可能抹掉尚未记录的新事件。

## 5. Async removal 的 containment

现代规范允许DPC或Hot-Plug Surprise（HPS）支持async removal，但明确把HPS用于async hot-plug标为deprecated，并推荐DPC。原因不是喜新厌旧：DPC能在Downstream Port干净停止TLP stream，给containment error recovery一个明确边界；HPS路径下，Poisoned MWr、CTO等Uncorrectable error可能被忽略、异步处理或导致系统崩溃，driver很难判断哪些高级操作成功。

典型证据链：

```text
physical disappearance / containment disable
 → DL_Active→DL_Down
 → optional Surprise Down / DPC trigger
 → pending MRd becomes CTO or aborted
 → Port/Root AER + hot-plug event
 → OS停止访问removed subtree
 → driver error/remove path清DMA/IRQ/resource
```

当前Cadence configuration-specific capability list没有DPC。通用core有Hot-Plug接口，但Slot Capabilities中的Hot-Plug Capable/Surprise字段默认为0，必须由integrator按真实slot能力配置。因此本项目不能承诺async removal已有规范推荐的DPC containment；这需要SoC build、board wiring与OS策略共同确认。

## 6. Cadence Hot-Plug 硬件边界

每个Link有独立sideband：

- 输入：`PRSNT_N`、`ATTENTION_BUTTON_N`、`POWER_FAULT_N`、`MRL_SENSOR_N`、`EMI_STATUS`、`COMMAND_COMPLETED`；
- 输出：`COMMAND_CHANGED`、Attention/Power indicator control、power/interlock control、`HOT_PLUG_INTERRUPT_OUT`。

Controller同步异步输入、更新Slot Status、把Slot Control变化送给外部slot control logic。外部逻辑真正驱动电源、LED、锁，并用`COMMAND_COMPLETED`回应。Cadence并没有在IP里藏一只替你切12V的继电器。

`HOT_PLUG_INTERRUPT_OUT`保持assert直到software对Slot Status写1清除。Cadence Appendix I明确把Hot-Plug Capability限制在Downstream Ports；Endpoint Function不会因为自己插在slot里就实现这套slot controller。

## 7. Linux 边界与driver责任

通用Linux架构中，Root Port hot-plug service负责slot事件、增删PCI hierarchy；Function driver的remove/error path负责设备特有的queue、DMA、IRQ和context。即使OS能从对象树移除`pci_dev`，它也无法撤销已被设备发出、但状态不明的外部副作用。

Function driver应保证：

- remove/suspend/error路径可阻止新doorbell和DMA descriptor；
- 中断handler/workqueue在BAR unmap前同步退出；
- streaming DMA mappings按API收尾；
- MMIO read返回all-ones或异常后不无限重试；
- async removal后不等待永远不会到的device completion；
- recovery与正常remove可区分“设备还活着能quiesce”和“只能本地清理”。

X6000 Linux 5.10.4 中的通用 hot-plug 路径可在 `drivers/pci/hotplug/pciehp_*` 和 `drivers/pci/pcie/portdrv_*` 中确认；DPC 路径见 `drivers/pci/pcie/dpc.c`。不过，外部 Host 是否启用 pciehp/DPC、firmware 是否保留控制权，以及实际 irq-domain 拓扑仍取决于 Host 平台，当前保持 Unknown。

## 8. Debug：设备突然消失时留什么

1. Presence、DLL Link Active、LTSSM history的时间顺序；
2. Slot Status/Control/Capabilities原始值，读取后再清；
3. AER Uncorrectable/Correctable、DPC（若有）、Surprise Down；
4. outstanding MRd/tag、CTO与DMA engine状态；
5. `dmesg`中hotplug、AER、IOMMU fault和driver remove先后；
6. 电源、PERST#、refclk与PRSNT#波形；
7. 是orderly操作、真实拔卡、power fault，还是Link/SI故障伪装成拔卡。

## 9. Source Reference

- PCI Express Base Specification 5.0 r1.0：§§6.7、6.2、2.9，尤其§6.7.6 Async Removal。
- Cadence PCIe Core User Guide：§§3.9–3.12、Root Port Slot registers、Appendix I Hot Plug Support、Appendix H.1。
- MindShare *PCI Express System Architecture*：Chapter 17，约PDF pp.705–750；orderly removal直觉可用，surprise-removal提前量说法已按PCIe 5.0校正。
