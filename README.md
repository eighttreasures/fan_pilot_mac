# FanPilot

> A native macOS menu bar fan control tool that drives fan speed by temperature with hysteresis and per-power-state rules — plus a root SMC helper with a 10-second watchdog, and an always-on system monitor for CPU, GPU, memory, disk and network. Built with Swift / AppKit, no kernel patches.

**FanPilot** 是一款原生 macOS 菜单栏风扇策略工具：按温度自动调速，用**回差（hysteresis）**避免风扇在阈值附近频繁启停，并把**通用 / 插电 / 电池**三组规则分开管理。它通过一个特权后台服务直接读写 AppleSMC，配合 **10 秒看门狗**——应用退出、休眠或后台服务失联时自动把风扇交还 macOS。菜单栏面板同时显示 CPU、内存、GPU、磁盘和网络状态，把「调风扇」和「看机器」合在一个小工具里。

> 本仓库是**二进制分发仓**：只放安装包（GitHub Release assets）与项目说明，源码私有不公开。

## Why FanPilot

- **Temperature rules with hysteresis（温度回差）**：每条规则设开始温度 / 停止温度 / 传感器 / 风扇百分比。触发后必须回落到停止温度才降级，不会在两个阈值之间来回抖动。
- **Per-power-state rules（按供电状态分组）**：通用、插电、电池三套规则独立配置。自动模式下通用规则与当前供电状态规则一起计算；也可手动只跑其中一组。
- **Multi-rule arbitration（多规则仲裁）**：同时触发多条规则时取最高百分比；CPU 规则要求超过半数所选温度指标达到开始温度才触发，且需全部低于停止温度才允许更低规则接管，避免误判。
- **10-second watchdog（看门狗兜底）**：手动控制每 2 秒发一次心跳，超过 10 秒未收到即恢复 macOS 自动控制。应用退出、系统休眠、后台服务失联同样自动交还控制权。
- **System monitor in the menu bar（菜单栏系统监控）**：CPU / 内存 / GPU / 磁盘 / 网络一屏可见；CPU、内存、GPU 列表按应用汇总，可展开看子进程；CPU 核心负载曲线默认展示最近 1 小时，可改为分钟 / 小时 / 天或保留本次运行全部数据。
- **Manual override（手动覆盖）**：控制面板可直接拖滑杆手动定速，临时压过温控规则，关闭后自动恢复规则判断。0% 映射到硬件最低安全转速，100% 映射到最高转速。
- **Resource-conscious（按需采样）**：GPU 整体负载通过 IOKit 直读；GPU 应用排名只在详情面板打开时采样，关闭详情后不做进程扫描。
- **Local update check（更新检查）**：客户端匿名请求本仓 `/releases/latest`，对比 tag 版本号后提示，节流 6 小时，可在设置里调整。

## How it compares

| | FanPilot | Macs Fan Control | iStat Menus | smcFanControl |
| --- | :---: | :---: | :---: | :---: |
| 温度回差避免频繁启停 | **✅ 内建** | 曲线可配置 | — | ❌ |
| 通用 / 插电 / 电池分组规则 | **✅** | ❌ | ❌ | ❌ |
| 手动定速临时覆盖规则 | **✅** | ✅ | ❌（只读监控） | ✅ |
| 失联 / 退出自动交还系统控制 | **✅ 10s 看门狗** | 依赖进程退出 | — | — |
| 菜单栏内置 CPU/GPU/内存/磁盘/网络监控 | **✅** | ❌ | ✅ | ❌ |
| CPU 核心负载曲线（可配时间范围） | **✅** | ❌ | ✅ | ❌ |
| 特权后台服务 + 签名客户端校验 | **✅** | ✅（驱动） | — | — |
| Apple Silicon 原生 | ✅ | ✅ | ✅ | ✅ |

> 对比依据：2026-11 月各项目公开功能自述，仅作能力维度参考；各工具持续演进。iStat Menus 以只读监控为主，不等同于风扇控制工具。

## Install

1. 从本仓库 [Releases](../../releases/latest) 下载最新的 `FanPilot-<version>.zip`。
2. 双击解压得到 `FanPilot.app`，拖进「应用程序」文件夹。
3. 首次打开：右键 →「打开」。应用为本地 ad-hoc 签名，未做 Apple 公证，Gatekeeper 会提示来源未知。
4. 菜单栏出现图标后，点开控制面板 →「安装后台服务」，输入一次管理员密码（用于安装 root LaunchDaemon 访问 AppleSMC）。
5. 打开「设置…」，在通用 / 插电 / 电池页添加规则，设好开始温度、停止温度与转速，再开启规则控制。

要求 macOS 13.0 及以上，Apple Silicon 与 Intel 均可。

> ⚠️ 不要同时启用 iStat Menus、Macs Fan Control 等其他软件的手动风扇规则。多个工具同时写 AppleSMC 时，最终转速取决于最后一次写入，行为不可预测。

## FAQ

**Q: FanPilot 会不会一直把风扇拉满？**
不会。初始示例规则全部处于关闭状态，安装后不会自动改写风扇控制。只有开启规则控制或手动定速后才会介入。

**Q: 什么是回差（hysteresis），为什么重要？**
如果只有一个温度阈值，温度在阈值附近波动时风扇会反复启停。FanPilot 用「开始温度 / 停止温度」两个阈值：达到开始温度才升速，回落到停止温度才降速，中间区间保持当前状态。

**Q: 应用崩溃或忘了退出，风扇会一直被手动控制吗？**
不会。手动控制期间每 2 秒发一次心跳，后台服务超过 10 秒未收到就自动恢复 macOS 自动控制。退出应用、系统休眠、后台服务失联同样触发兜底。

**Q: 为什么不生效？后台服务是什么？**
FanPilot 需要一个 root LaunchDaemon 才能访问 AppleSMC。首次使用必须在控制面板点「安装后台服务」并授权一次。该服务只接受当前控制台用户、签名标识为 `com.YANG-sty.FanPilot` 的客户端连接。

**Q: 可以同时用 Macs Fan Control 或 iStat Menus 吗？**
不建议同时启用两者的手动风扇写入。多个工具同时写 AppleSMC 时结果取决于最后一次写入。只读监控（不写 SMC）可以共存。

**Q: 上传或收集我的数据吗？**
有一次**匿名启动统计**：上报到 TelemetryDeck，字段仅包含应用版本、macOS 版本、架构、以及一个本地随机 UUID（服务端再次加盐哈希），不含任何个人数据或温度 / 规则配置；fire-and-forget，无重试、无 UI 提示。除此之外不上传风扇规则或监控数据，没有账号体系。风扇规则、监控历史、收款码等数据全部留在本地。

**Q: 更新检查是怎么工作的？**
客户端匿名请求 `https://api.github.com/repos/eighttreasures/fan_pilot_mac/releases/latest`，对比 tag 版本号后提示；节流 6 小时，可在设置里调整或关闭。菜单里的「检查更新…」只在你主动点击时触发。

**Q: 应用里的「支持 FanPilot」是什么？**
一个可选的打赏窗口，只在菜单里**人工点击**时弹出，展示微信 / 支付宝收款码，没有任何自动弹出或行为调度。不点就完全不会出现。

**Q: 为什么 ad-hoc 签名第一次打开要右键？**
本地 ad-hoc 签名未做 Apple Developer ID 签名与公证，Gatekeeper 会拦截首次启动。右键 →「打开」即可放行一次。这是本地构建的正常现象。

## Source

源码私有，不在本仓公开。本仓仅用于二进制分发与项目说明。

## License

二进制发布，源码不公开。免费下载使用。

---

Keywords: macOS fan control, Mac fan speed control, temperature based fan curve, fan hysteresis, per-power-state fan rules, AppleSMC fan control, privileged SMC helper, macOS menu bar utility, native Swift AppKit fan control, MacBook fan control, iMac fan control, Mac mini fan control, CPU temperature fan, manual fan override, fan watchdog, macOS system monitor menu bar, CPU core load graph, GPU load monitor IOKit, memory swap monitor, network disk monitor, low resource macOS utility, ad-hoc signed macOS app, 风扇控制, 温度调速, 回差, 菜单栏风扇, 苹果风扇调速, SMC 风扇, 系统监控, 低资源占用
