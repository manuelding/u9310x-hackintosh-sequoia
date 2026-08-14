# 黑苹果 Sequoia 安装说明书（2026-08-09）

**用途**：用 OpCore-Simplify 重新生成 EFI 后，依据本文档校正 `config.plist`，然后安装 macOS Sequoia 15。
**读者**：执行者本人 + 协助修改 config.plist 的 AI。
**前置文档**：`Hackintosh_Sonoma_交接明细_2026-08-08_v5.md`（panic 排查过程）、`Hackintosh_Tahoe_离线安装执行方案_2026-08-09.md`（v2，U 盘制作与离线流程）

> **适用范围**：本文记录的是这一台机器、这一版 EFI 和当时安装介质的实测过程，不是通用购买指南。涉及其他硬件、OpenCore 或 macOS 版本时，应重新查阅对应版本的上游文档并验证，不能机械照抄。

---

## 一、硬件事实（已实测，可直接采信）

### 1.1 基本信息

| 项 | 值 | 来源 |
|---|---|---|
| 机型 | 富士通 LIFEBOOK U9310X **裸主板** | — |
| CPU | Intel Core i7-10610U（Comet Lake-U, 4C8T） | AIDA64 |
| 核显 | Intel UHD Graphics（Comet Lake-U GT2） | AIDA64 |
| 芯片组 | Comet Point-LP PCH | AIDA64 |
| BIOS | Version 2.28 (2025-03-14), SMBIOS 3.2 | AIDA64 |
| 存储 | 兼容的 NVMe；旧盘因兼容性问题已更换，见 1.4 | `system_profiler SPNVMeDataType` |
| 有线网卡 | **Intel I219-LM (10)，`8086:0D4E`，PCI `0:1F.6`** | AIDA64 + Recovery 实测 |
| 无线网卡 | **不存在**（M.2 WLAN 槽为空） | AIDA64 PCI 全表 |
| 雷电 | Intel JHL7540 Titan Ridge TB3 | AIDA64 |
| 音频 | Comet Point-LP cAVS | AIDA64 |
| 目标 SMBIOS | **MacBookPro16,3** | — |

### 1.2 形态约束（务必带入所有判断）

**"无头骑士"裸主板**：无电池、无内屏、HDMI 接视频采集卡看输出。最终成品也是裸板运行。

因此以下功能**本来就不是刚需，主动放弃，不要浪费排查时间**：

- 电池管理（`SMCBatteryManager` 永久禁用）
- 内屏亮度 / PNLF / 光感（`SSDT-PNLF`、`SSDT-ALS0`、`BrightnessKeys` 全部不需要）
- 内建键盘触控板（`VoodooPS2*`、`VoodooI2C*` 全部不需要，用 USB 键鼠）

### 1.3 USB 口预算（硬约束）

**只有 2 个可用 USB-A 口。**

| 用途 | 占用 |
|---|---|
| 启动 U 盘 | 1 |
| 键鼠套装接收器 | 1 |
| **合计** | **2 / 2，正好用满** |

- 板载网口走 PCH，**不占 USB**，所以有线上网是零成本的。
- 另有 TB3/USB-C（`0A:00.0 JHL7540 TB3 USB 3.1 xHCI`），但 macOS 下 TB 控制器不一定上电，**按"没有"规划**。

### 1.4 NVMe 故障记录（仅适用于本机）

旧 NVMe 在持续写入后曾从 BIOS / PCIe 设备列表消失，冷关机后恢复；更换硬盘后，本次安装成功。这组先后现象说明旧盘或其工作环境值得怀疑，但不足以单独证明具体主控、缓存结构或温度就是唯一根因，也不能据此推出品牌或型号的通用兼容性排名。

本文不提供 SSD 推荐或黑名单。遇到相似故障时，应分别检查安装日志、连接与供电、散热、SMART / NVMe 健康数据、固件以及当前 macOS/OpenCore 的已知兼容问题；更换硬盘只能作为隔离变量的一种测试。

### 1.6 无线网络说明

本机未安装无线网卡，因此没有完成 Sequoia 下的无线硬件兼容性验证。本文不推荐具体无线网卡，也不对 PCIe、USB 或特定芯片方案作通用可用性结论。选购前应核对当前 macOS 版本、驱动或补丁要求、接口键位、蓝牙内部连接方式以及 BIOS 限制。

如果只需要联网，可优先使用本机已经验证可用的有线网口；无线桥接设备通过网线接入也可作为不改动本机硬件的备选方案。

### 1.5 完整 PCI 设备表（存档，OpCore-Simplify 生成后可比对）

```
0:00.0  Comet Lake-U 4C Host Bridge          0:1C.0  PCIe Root Port 1
0:02.0  Comet Lake-U GT2 iGPU                0:1C.4  PCIe Root Port 5
0:04.0  DPTF Processor Participant           0:1D.0  PCIe Root Port 9
0:12.0  Comet Point-LP Thermal               0:1D.4  PCIe Root Port 13
0:13.0  Integrated Sensor Hub                0:1E.0  LPSS UART 0
0:14.0  USB 3.1 xHCI Host Controller         0:1E.1  LPSS UART 1
0:14.2  Shared SRAM                          0:1F.0  LPC/eSPI Controller
0:15.0  LPSS I2C Controller 0                0:1F.3  cAVS (Audio)
0:15.3  LPSS I2C Controller 3                0:1F.4  SMBus Host Controller
0:16.0  HECI Controller 1                    0:1F.5  SPI (Flash) Controller
0:16.3  KT Redirection                       0:1F.6  ★ I219-LM Ethernet (8086:0D4E)

06:00.0 / 07:00.0 / 07:01.0 / 07:02.0 / 07:04.0   JHL7540 Titan Ridge TB3 Bridge
08:00.0 JHL7540 TB3 NHI      0A:00.0 JHL7540 TB3 USB 3.1 xHCI
72:00.0 NVMe                                  77:00.0 O2Micro MMC/SD Controller
```

原始报告：`001_硬件信息_u9310x.txt`（AIDA64，GBK 编码；此表为换盘前采集）

---

## 二、目标版本范围

本文只记录 macOS Sequoia 15.7.9 的已完成安装。其他 macOS 版本的机型支持、补丁和 kext 兼容性会变化，本文未验证，不作比较或兼容性承诺。

---

## 三、★ 血泪教训（OpCore-Simplify 不知道这些，优先级最高）

### 3.1 `DevirtualiseMmio = True` 会导致确定性 kernel panic ★★★

**这是 v3–v5 三个版本卡住的根因，本文档最重要的一条。**

- **症状**：verbose 走到 `[ PCI configuration end, bridges 2, devices 19 ]` → `console relocated to 0x60000000` → 立刻 panic (type 14 page fault)
- **根因**：`DevirtualiseMmio = True` 且 `MmioWhitelist` 为空 → OpenCore 剥掉 `0xff650000`（富士通 Insyde 固件的 SPI flash 映射区）的 `EFI_MEMORY_RUNTIME` 属性 → macOS 不重定位 EFI Runtime Services 表 → `AppleEFINVRAM` 拿物理地址调 `RT->SetVariable`（表偏移 `0x58`）→ 页不存在 → panic
- **证据**：两次独立测试 `RIP − kernel text base = 0x44453BE` 完全一致（确定性故障）；`CR2 = 0xff650058`、`R10 = 0x50`（`GetNextVariableName`）精确命中 RT 表布局；`last started kext = AppleEFINVRAM`
- **修复**：`Booter → Quirks → DevirtualiseMmio = False`

> 复现本机已成功的配置时保持 `False`。不要据此推断 OpCore-Simplify 在其他机器上的默认值。

### 3.2 `PanicNoKextDump = True` 会屏蔽诊断信息

v4 阶段一直拿不到 `Kernel Extensions in backtrace`，不是拍照漏了，是这个 quirk 屏蔽的。
**整个调试期必须保持 `False`**，装稳定后再考虑打开。

### 3.3 本次 EFI 中的 USB 端口映射曾伴随卡死

v3 记录：当时启用的 `USBToolBox` + `UTBMap` 配置与 `console relocated` 卡点同时出现，移除后流程继续。该对照没有证明 USB 映射机制本身有通用问题；复现本次安装时先沿用禁用状态，后续映射应按 USBToolBox 当前上游说明重新制作和验证。

### 3.4 SMC 插件会导致假死

v4 记录：`SMCBatteryManager` / `SMCProcessor` / `SMCSuperIO` 会让流程卡在 `AppleCredentialManager` 假死。禁用后才推进到真实 panic。
**首次安装全部禁用。**

### 3.5 累计已排除方向（不要重复踩坑）

| 方向 | 结论 |
|---|---|
| `RestrictEvents` / `UTBDefault` / 旧版 WEG / `Voodoo*` | 导致早期 panic，禁用 |
| 完整 USB 端口映射 | 卡 `console relocated`，禁用 |
| `XhciPortLimit` True/False | 无决定性影响 |
| `SSDT-USB-Reset` / `SSDT-USBX` | 无决定性影响 |
| 内存映射 Quirks 各种翻转组合 | 对 `AppleCredentialManager` 卡点无效 |
| 重新启用 WEG + IGPU DeviceProperties 注入 | 对卡点无效，但配置保留，无害 |
| **禁用 SMC 插件三件套** | ✅ 有效 |
| **`DevirtualiseMmio = False`** | ✅✅ 有效，彻底解决 panic |

### 3.6 已验证可出图的核显配置

```
DeviceProperties → Add → PciRoot(0x0)/Pci(0x2,0x0)
  AAPL,ig-platform-id      = 07009B3E     (即 0x3E9B0007)
  framebuffer-patch-enable = 01000000
```

这套值**已实测能通过 HDMI 输出到采集卡**。若 OpCore-Simplify 生成了不同的 ig-platform-id，先用它的；**如果黑屏无输出，换回上面这组**。

### 3.7 Recovery 联网重装的已知失败模式

2026-08-09 实测（Tahoe recovery）：网络完全正常（DHCP、DNS、Apple CDN、时钟均正确），但点 Reinstall 报
`Installation requires downloading important content. That content can't be downloaded at this time.`

`install.log` 显示：

```
mobileassetd: Connection to service named com.apple.softwareupdated was invalidated:
              Connection init failed at lookup with error 3 – No such process.
```

即 Recovery 环境里 `softwareupdated` 服务缺失，下载链路断在中间。
**若 Sequoia recovery 出现同样症状，不要纠缠，直接转离线方案（第七节路线 B）。**

### 3.8 本次 Sequoia Recovery 环境的观察

以下是本机、当时介质和连接方式下的观察，不应外推为所有 Sequoia Recovery 环境的限制：

| 限制 | 实测证据 | 后果 |
|---|---|---|
| **该 exFAT 分区挂载失败** | `diskutil mount disk3s1` → `Volume on disk3s1 failed to mount`（分区类型 `Microsoft Basic Data`，62.3G） | 为复现已成功的流程，继续使用 HFS+；单次失败不能证明 Recovery 普遍不支持 exFAT |
| **该 Hub 下的键鼠无响应** | 键鼠接收器插入当时使用的 hub 后 Recovery 里无法操作 | 本次流程使用两个直插口；不能据此断言其他 Hub 或外置安装方案均不可行 |
| **没有 `clear` / `system_profiler`** | `command not found` | 清屏用 `printf '\033c'` 或 `Ctrl+L`；查机型用 `sysctl hw.model` 或 `ioreg -l \| grep -i product-name` |

### 3.9 联网重装 error 702 的完整诊断（2026-08-10）

路线 A 在 Sequoia recovery 下的失败形态与 3.7 的 Tahoe 不同，**下载确实启动了**，但死在下载之后：

```
04:36:10  Retrieving 1 packages (15.656 GB) — com.apple.pkg.InstallAssistant.macOSSequoia
04:36:10  Started downloading ... swcdn.apple.com
04:39:22  最后一条相关日志，之后 38 分钟 osinstallersetupd 完全静默（只有 powerd 断言）
05:17:01  Couldn't mount dmg! (error code 2)
05:17:01  Failed to find the payload dmg.
05:17:01  Operation failed = OSISMountPkgDmgOperation
05:17:01  Error Domain=com.apple.OSInstallerSetup.error Code=702
05:26:41  Cleaning up installer temporary files ... Will purge all copies except: (null)
```

判读要点：

- `error code 2` = **ENOENT，文件不存在**，不是 I/O 错误、不是网络握手失败
- 失败点在**下载完成之后的解包/挂载**环节
- `Macintosh HD` 占用 15.7 GB ≈ 包体 15.656 GB，说明数据基本落盘了
- **05:26 的清理会删光下载物，重试等于重下 15.6 GB**
- 同时排除的因素：`date` 正确（`Aug 10 05:55 UTC 2026`）、`SMART Status: Verified`、APFS 容器完好、容器剩余 >110 GB

**结论：路线 A 在本机上视为不可用，不要再尝试。**

### 3.10 旧 NVMe 消失故障记录

> 更换旧盘后，本次安装成功且未再次观察到设备消失。这个结果支持“旧盘或其工作环境是变量之一”，但不能单独证明具体故障机理。

**2026-08-10 已复现**：安装后进 BIOS，NVMe **从设备列表中消失**；关机冷却后自行恢复。

- 旧盘：具体型号已脱敏。
- 已观察事实：持续写入后设备从 BIOS 列表消失，冷关机后恢复。
- 未确认推断：过热、供电、固件、控制器重置或其他硬件问题均可能产生类似现象；当时没有足够的温度和控制器日志确认其中某一种。
- 与 error 702 的关系未被证明：时间接近不等于因果关系，应分别保留安装日志和存储健康证据。

**关键认知：路线 B 并不比路线 A 写得少。**

```
路线 A：下载 15.6G → 展开 → 装系统                      ≈ 35–40G 写入
路线 B（旧）：installer -pkg 展开 15.6G → startosinstall  ≈ 35–40G 写入
路线 B（新，第六节已改）：Linux 上预先展开 app → 直接 startosinstall   ≈ 20G 写入
```

两条路线都会产生大量写入。再次测试前应保证正常散热与稳定供电，并用厂商诊断工具或 `nvme smart-log` 保存健康和温度基线。任何异常计数都应结合设备文档解释；本文不设通用于所有 SSD 的报废阈值。

### 3.11 已否决的方案（不要重复提出）

| 方案 | 否决理由 |
|---|---|
| 路线 A 联网重装 | 见 3.9，error 702，且每次重试要重下 15.6G |
| 安装到外置 USB SSD，先跑通再迁移 | 见 3.8，Recovery 下 hub 里的键鼠无响应，凑不出第三个 USB 口 |
| 本次介质的数据分区改用 exFAT | 见 3.8；该分区曾挂载失败，复现流程继续使用已验证的 HFS+ |

---

## 四、★ config.plist 校正表（对 OpCore-Simplify 新生成的文件逐项执行）

> 操作方式：**只改下表列出的键，其余保持 OpCore-Simplify 的输出**。改完跑第五节的验证脚本。

### 4.1 Booter → Quirks

| 键 | 必须值 | 理由 |
|---|---|---|
| `DevirtualiseMmio` | **`False`** | 本机相关测试中，设为 `True` 时重复出现相同 panic；见 3.1 |
| `AvoidRuntimeDefrag` | `True` | 标准 |
| `EnableWriteUnprotector` | `True` | 老式内存映射组合，v5 已验证可用 |
| `RebuildAppleMemoryMap` | `False` | 同上，与 EWU 配对 |
| `SyncRuntimePermissions` | `False` | 同上 |
| `SetupVirtualMap` | `True` | 标准 |
| `ProtectUefiServices` | `True` | 标准 |
| `ProvideCustomSlide` | `True` | 标准 |
| `FixupAppleEfiImages` | `True` | 标准 |
| `ProtectMemoryRegions` | `False` | v5 基准值 |

> **关于内存映射组合**：表中组合只表示本机 v5 的成功基线，不代表对其他固件更优。若重新生成 EFI，应按当前 OpenCore 指南整体核对相关选项，不要只翻转其中一个键，也不要把睡眠问题直接归因于这组设置。

### 4.2 Booter → Patch

| 动作 | 理由 |
|---|---|
| 若存在 `Skip Board ID check`（Find = `PlatformSupport.plist`）→ `Enabled = False` | 本机成功配置未启用该补丁；其他 SMBIOS 或系统版本需重新核对 |
| 若存在 `macOS to hacOS`（Find = `macOS`）→ `Enabled = False` | 纯装饰性，无用 |

### 4.3 Kernel → Quirks

| 键 | 必须值 | 理由 |
|---|---|---|
| `PanicNoKextDump` | **`False`** | ★ 见 3.2，调试期必须 |
| `DisableIoMapper` | `True` | Comet Lake 需要（除非 BIOS 已关 VT-d） |
| `AppleXcpmCfgLock` | `True` | Comet Lake 需要（除非 BIOS 可解 CFG Lock） |
| `DisableLinkeditJettison` | `True` | Lilu 需要 |
| `PowerTimeoutKernelPanic` | `True` | 标准 |
| `XhciPortLimit` | `False` | Sequoia 上必须 False |
| `SetApfsTrimTimeout` | `-1` | 标准 |

### 4.4 Kernel → Add（★ 首次安装用最小集）

**只启用这 5 个**：

| Kext | Enabled | 说明 |
|---|---|---|
| `Lilu.kext` | ✅ | 必须排在最前 |
| `VirtualSMC.kext` | ✅ | 无它不能引导 |
| `WhateverGreen.kext` | ✅ | 核显 |
| `NVMeFix.kext` | ✅ | 改善 NVMe 电源管理兼容性 |
| `IntelMausiEthernet.kext` | ✅ | **I219-LM `8086:0D4E`，已核对在其 `IOPCIMatch` 列表内**。路线 A 依赖它 |

**全部禁用**（无论 OpCore-Simplify 是否启用了它们）：

```
SMCBatteryManager    SMCProcessor      SMCSuperIO       SMCLightSensor
USBToolBox           UTBMap            UTBDefault
VoodooI2C（及全部 PlugIns）            VoodooI2CHID
VoodooPS2Controller（及全部 PlugIns）
RestrictEvents       BrightnessKeys    AppleALC         XHCI-unsupported
AirportItlwm / itlwm（本机无无线网卡，装了也没用）
```

> `AppleALC`：裸板无内建喇叭，装完系统再开。
> `XHCI-unsupported`：Comet Lake-U 的 XHCI 原生支持，理论上不需要。v5 带着它能进安装器，但那是历史遗留；**新 EFI 若 OpCore-Simplify 没加，就不要加**。
> **装完系统后再逐个加回来，一次只加一个，加完验证能启动再加下一个。**

### 4.5 ACPI → Add

| SSDT | Enabled | 说明 |
|---|---|---|
| `SSDT-PLUG` | ✅ | CPU 电源管理 |
| `SSDT-RTCAWAC` | ✅ | RTC |
| `SSDT-XOSI` | ✅ | 配套 `_OSI to XOSI` rename |
| `SSDT-EC` 或 `SSDT-EC-USBX` | 按当前 ACPI 状态核对 | 不应仅因机型类别直接添加；先确认已有 EC、重命名和 USBX 注入，按当前 OpenCore ACPI 指南处理 |
| `SSDT-PNLF` | ❌ | 无内屏 |
| `SSDT-ALS0` | ❌ | 无光感 |
| `SSDT-GPI0` | ❌ | 不用 I2C 触控板 |
| `SSDT-USB-Reset` / `SSDT-USBX`（独立的） | ❌ | v3 已验证无决定性影响 |
| `SSDT-MCHC` / `SSDT-SBUS` | 随 OpCore-Simplify | 无害 |

### 4.6 ACPI → Patch（★ 一致性检查）

**规则：rename 必须和对应的 SSDT 同步启用/禁用。** v5 遗留了两处不一致，必须避免：

| Patch | 应有状态 |
|---|---|
| `_OSI to XOSI rename` | ✅ 启用（`SSDT-XOSI` 已启用） |
| `GPI0 _STA to XSTA` | ❌ 禁用（`SSDT-GPI0` 禁用） |
| `PNLF to XNLF` | ❌ 禁用（`SSDT-PNLF` 禁用） |
| `EC0/H_EC _STA to XSTA` 之类 | ✅ 启用（若启用了 `SSDT-EC`） |

### 4.7 DeviceProperties

```
PciRoot(0x0)/Pci(0x2,0x0)          ← iGPU
  AAPL,ig-platform-id      = <data> 07009B3E
  framebuffer-patch-enable = <data> 01000000
```

若 OpCore-Simplify 生成了不同的 ig-platform-id，**先用它的**；黑屏无输出时换回 `07009B3E`（已实测能出图，见 3.6）。

其余 DeviceProperties（音频 layout-id 等）保持 OpCore-Simplify 输出。

### 4.8 UEFI

| 键 | 必须值 | 理由 |
|---|---|---|
| `APFS → EnableJumpstart` | **`True`** | ★ 否则 OpenCore 看不见任何 APFS 卷 → 第一次重启后菜单里没有 `macOS Installer` → 安装流程断死 |
| `APFS → MinDate` | `-1` | 最宽松 |
| `APFS → MinVersion` | `-1` | 最宽松 |
| `Drivers` | 至少含 `OpenRuntime.efi` + `HfsPlus.efi` | `HfsPlus.efi` 用于读 `com.apple.recovery.boot` |
| `Drivers → apfs_aligned.efi` | 不需要（有则禁用） | Clover 时代产物，Jumpstart 开了就不需要 |
| `Output → ProvideConsoleGop` | `True` | 标准 |
| `Quirks → ReleaseUsbOwnership` | `True` | 标准 |
| `Quirks → RequestBootVarRouting` | `True` | 标准 |

### 4.9 Misc

| 键 | 必须值 | 理由 |
|---|---|---|
| `Security → SecureBootModel` | `Disabled` | 安装期。装稳后可考虑 `Default` |
| `Security → ScanPolicy` | `0` | 扫描全部设备，否则可能看不到 U 盘/外置卷 |
| `Security → DmgLoading` | `Signed` | 能加载 Apple 签名的 `BaseSystem.dmg` |
| `Security → Vault` | `Optional` | 标准 |
| `Boot → ShowPicker` | `True` | 必须能看到菜单 |
| `Boot → Timeout` | `5`～`10` | 留出手动选择时间 |
| `Boot → HideAuxiliary` | **`False`** | ★ 装机期务必设 False，否则可能看不到 Recovery / `macOS Installer` 等辅助条目。装完再改 True |
| `Boot → PollAppleHotKeys` | `True` | 方便 |
| `Debug → Target` | `67`（或 3） | 出问题能看到日志 |

### 4.10 NVRAM → `7C436110-AB2A-4BBB-A880-FE41995C9F82`

| 键 | 值 |
|---|---|
| `boot-args` | `-v debug=0x100 keepsyms=1`（可保留 `npci=0x2000`，那是 v5 已验证组合的一部分；装完系统后再尝试移除） |
| `csr-active-config` | `03 0A 00 00`（部分关闭 SIP，装机够用） |

**不要**加 `-lilubetaall`（Sequoia 下不需要）、不要加 `-no_compat_check`（Sequoia 下不需要）。

### 4.11 PlatformInfo

| 键 | 值 |
|---|---|
| `Generic → SystemProductName` | `MacBookPro16,3` |
| `Generic → SystemSerialNumber` / `MLB` / `SystemUUID` / `ROM` | **用 OpCore-Simplify 新生成的一整套，不要混用旧值** |
| `Automatic` | `True` |
| `UpdateSMBIOSMode` | `Custom` |

---

## 五、验证脚本（改完 config.plist 后必跑）

保存为 `verify_config.py`，用法：`python3 verify_config.py <EFI/OC/config.plist>`

```python
#!/usr/bin/env python3
import plistlib, sys

p = sys.argv[1] if len(sys.argv) > 1 else "EFI/OC/config.plist"
d = plistlib.load(open(p, "rb"))
ok = fail = warn = 0

def chk(cond, msg, hard=True):
    global ok, fail, warn
    if cond:
        print(f"  PASS  {msg}"); ok += 1
    elif hard:
        print(f"! FAIL  {msg}"); fail += 1
    else:
        print(f"~ WARN  {msg}"); warn += 1

bq = d["Booter"]["Quirks"]; kq = d["Kernel"]["Quirks"]
print("== Booter/Quirks ==")
chk(bq["DevirtualiseMmio"] is False, "DevirtualiseMmio == False   <<< 最关键")
chk(bq["EnableWriteUnprotector"] is True,  "EnableWriteUnprotector == True")
chk(bq["RebuildAppleMemoryMap"] is False,  "RebuildAppleMemoryMap == False")
chk(bq["SyncRuntimePermissions"] is False, "SyncRuntimePermissions == False")
chk(bq["AvoidRuntimeDefrag"] is True, "AvoidRuntimeDefrag == True")
chk(bq["SetupVirtualMap"] is True, "SetupVirtualMap == True")

print("== Kernel/Quirks ==")
chk(kq["PanicNoKextDump"] is False, "PanicNoKextDump == False")
chk(kq["DisableIoMapper"] is True,  "DisableIoMapper == True")
chk(kq["AppleXcpmCfgLock"] is True, "AppleXcpmCfgLock == True")
chk(kq["XhciPortLimit"] is False,   "XhciPortLimit == False")

print("== UEFI ==")
chk(d["UEFI"]["APFS"]["EnableJumpstart"] is True, "APFS/EnableJumpstart == True")
drv = [x["Path"] for x in d["UEFI"]["Drivers"] if x.get("Enabled")]
chk("OpenRuntime.efi" in drv, f"Drivers 含 OpenRuntime.efi  ({drv})")
chk("HfsPlus.efi" in drv, "Drivers 含 HfsPlus.efi")

print("== Misc ==")
ms, mb = d["Misc"]["Security"], d["Misc"]["Boot"]
chk(ms["SecureBootModel"] == "Disabled", "SecureBootModel == Disabled")
chk(ms["ScanPolicy"] == 0, f"ScanPolicy == 0  (现: {ms['ScanPolicy']})")
chk(mb["ShowPicker"] is True, "ShowPicker == True")
chk(mb["HideAuxiliary"] is False, "HideAuxiliary == False (装机期)")

print("== PlatformInfo ==")
g = d["PlatformInfo"]["Generic"]
chk(g["SystemProductName"] == "MacBookPro16,3", f"SMBIOS == MacBookPro16,3 (现: {g['SystemProductName']})")
for k in ("SystemSerialNumber", "MLB", "SystemUUID"):
    chk(bool(g.get(k)), f"{k} 非空")

print("== Booter/Patch ==")
for x in d["Booter"].get("Patch", []):
    chk(not x.get("Enabled"), f"Booter patch 已禁用: {x.get('Comment')!r}", hard=False)

print("== Kernel/Add ==")
MUST = {"Lilu.kext", "VirtualSMC.kext", "WhateverGreen.kext",
        "NVMeFix.kext", "IntelMausiEthernet.kext"}
BAN_SUB = ("SMCBattery", "SMCProcessor", "SMCSuperIO", "SMCLightSensor",
           "USBToolBox", "UTBMap", "UTBDefault", "Voodoo",
           "RestrictEvents", "BrightnessKeys", "AppleALC",
           "itlwm", "Itlwm", "XHCI-unsupported")
en = [k["BundlePath"] for k in d["Kernel"]["Add"] if k["Enabled"]]
print(f"  已启用: {en}")
for m in sorted(MUST):
    chk(m in en, f"必须启用: {m}")
for k in en:
    chk(not any(b in k for b in BAN_SUB), f"不应启用: {k}")
if en:
    chk(en[0] == "Lilu.kext", "Lilu.kext 排在第一位")

print("== ACPI 一致性 ==")
acpi = {x["Path"]: x["Enabled"] for x in d["ACPI"]["Add"]}
print(f"  已启用 SSDT: {[k for k, v in acpi.items() if v]}")
for pat in d["ACPI"].get("Patch", []):
    c = (pat.get("Comment") or "")
    if not pat.get("Enabled"):
        continue
    if "PNLF" in c:
        chk(any("PNLF" in k and v for k, v in acpi.items()),
            f"rename {c!r} 启用但 SSDT-PNLF 未启用", hard=False)
    if "GPI0" in c:
        chk(any("GPI0" in k and v for k, v in acpi.items()),
            f"rename {c!r} 启用但 SSDT-GPI0 未启用", hard=False)

print("== boot-args ==")
ba = d["NVRAM"]["Add"]["7C436110-AB2A-4BBB-A880-FE41995C9F82"].get("boot-args", "")
print(f"  {ba!r}")
chk("-v" in ba.split(), "含 -v")
chk("keepsyms=1" in ba, "含 keepsyms=1")
chk("-lilubetaall" not in ba, "不含 -lilubetaall")

print(f"\n结果: PASS={ok}  FAIL={fail}  WARN={warn}")
sys.exit(1 if fail else 0)
```

**必须 `FAIL=0` 才能写盘。** `WARN` 逐条人工确认。

另外建议跑一次官方校验（版本要和你的 OpenCore.efi 匹配）：

```bash
./ocvalidate EFI/OC/config.plist
```

---

## 六、U 盘制作

### 6.1 准备资源

```bash
# 完整安装包（~15GB）
python3 gibMacOS.py            # 选 macOS Sequoia 15.x 最新版 → InstallAssistant.pkg

# Recovery 镜像
python3 gibMacOS.py -r         # 选对应的 15.x 条目 → RecoveryHDMetaDmg.pkg
7z x RecoveryHDMetaDmg.pkg -oRHM && cd RHM && 7z x Payload
find . -name 'BaseSystem.dmg' -o -name 'BaseSystem.chunklist'
```

**BaseSystem 与 InstallAssistant 的大版本必须一致**（都是 15.x），不可与 Tahoe / Sonoma 混用。

### 6.2 设备确认（每次写盘前都跑）

```bash
lsblk -o NAME,SIZE,MODEL,TRAN,MOUNTPOINT
readlink -f /sys/block/sdb | grep -i usb
```

> 公开日志中不要加入 `SERIAL` 字段；如果排查时曾输出硬盘序列号，上传前应删除或用 `<DISK_SERIAL>` 替换。

必须确认 `Netac OnlyDisk = /dev/sdb`、`KINGSTON SKC600 = /dev/sda`。
**写错一个字母就没有系统盘。**

### 6.3 ★ 在 Linux 上预先展开 InstallAssistant.pkg（2026-08-10 新增，关键优化）

**不要**把 `InstallAssistant.pkg` 原样拷进 U 盘再到目标机上 `installer -pkg` 展开 —— 那一步就是往热盘上写 15.6 GB。
**在 Linux 上展开好，把 `Install macOS Sequoia.app` 直接放进 U 盘**，目标机跳过展开，直接 `startosinstall`。

**收益：目标盘写入量从 ~40 GB 降到 ~20 GB，砍掉约一半热负载（见 3.10）。**

```bash
sudo apt install -y p7zip-full cpio hfsprogs

7z x InstallAssistant.pkg -oIA
cd IA && 7z x Payload            # 得到 Applications/Install macOS Sequoia.app
```

展开后必须自检（缺 `SharedSupport.dmg` 就是废的）：

```bash
du -sh "Applications/Install macOS Sequoia.app"
ls -l  "Applications/Install macOS Sequoia.app/Contents/SharedSupport/SharedSupport.dmg"
ls -l  "Applications/Install macOS Sequoia.app/Contents/Resources/startosinstall"
```

`startosinstall` 必须有可执行位。用 `sudo cp -a` 拷贝以保留权限。

### 6.4 分区与写入

> 本文的已验证流程使用 HFS+。当时测试的 exFAT 分区挂载失败，但这不是对所有 Recovery 环境的通用结论（见 3.8）。
> exFAT 同时也丢权限位和符号链接，装不了展开后的 `.app`。

```bash
sudo apt update && sudo apt install -y gdisk dosfstools hfsprogs

sudo umount /dev/sdb* 2>/dev/null
sudo sgdisk --zap-all /dev/sdb
sudo sgdisk -n 1:0:+1GiB -t 1:EF00 -c 1:EFI     /dev/sdb
sudo sgdisk -n 2:0:0     -t 2:AF00 -c 2:MACDATA /dev/sdb
sudo mkfs.vfat -F 32 -n OPENCORE  /dev/sdb1
sudo mkfs.hfsplus  -v MACDATA     /dev/sdb2

# 引导分区
sudo mkdir -p /mnt/oc && sudo mount /dev/sdb1 /mnt/oc
sudo cp -a <新生成的>/EFI /mnt/oc/
sudo mkdir -p /mnt/oc/com.apple.recovery.boot
sudo cp <路径>/BaseSystem.dmg <路径>/BaseSystem.chunklist /mnt/oc/com.apple.recovery.boot/
sync && sudo umount /mnt/oc

# 数据分区（路线 B 的弹药：已展开的 .app，不是 pkg）
sudo mkdir -p /mnt/macdata && sudo mount -t hfsplus -o rw,force /dev/sdb2 /mnt/macdata
sudo cp -a "IA/Applications/Install macOS Sequoia.app" /mnt/macdata/
sync && sudo umount /mnt/macdata
```

最终结构：

```
/dev/sdb1 (FAT32, OPENCORE)
├── EFI/BOOT/BOOTx64.efi
├── EFI/OC/{OpenCore.efi, config.plist, ACPI, Drivers, Kexts, Resources, Tools}
└── com.apple.recovery.boot/{BaseSystem.dmg, BaseSystem.chunklist}

/dev/sdb2 (HFS+, MACDATA)
└── Install macOS Sequoia.app/          ← 已展开，含 SharedSupport.dmg
```

> FAT32 单文件上限 4GB，装不下 15GB 的内容，所以必须两个分区。
> U 盘容量需求：EFI 1G + app ~15G ≈ 16G 起，Netac 58G 够用。

---

## 七、安装流程

**开始前：打开 OBS 录屏。** panic 信息一次性刷出大段内容，手机拍必漏顶部。
**接线**：网线插路由器 **LAN 口**（不是 WAN 口，也不要机对机直连 —— 拿不到 DHCP）。

### 阶段 1 —— 引导

1. 插 U 盘开机 → 进 OC 菜单 → 选 `macOS Base System` / `Recovery`
2. **若此处 panic**：立刻回看录像，按 v5 方法论排查（`RIP − kernel text base` 是否两次一致 → 确定性故障；`last started kext` 是时间线证据）。首先怀疑 `DevirtualiseMmio`

### 阶段 2 —— 抹盘

3. **磁盘工具** → 选内置 NVMe → 抹掉
   - 格式：**APFS**
   - 方案：**GUID 分区图**
   - 名称：`Macintosh HD`

### 阶段 3 —— Recovery 环境自检

> **2026-08-10 起：不再有"选路线"这一步。路线 A 已否决（3.9），默认直接走路线 B。**
> 本阶段只是确认环境正常、并留下可对照的基线。

4. **实用工具 → 终端**（Recovery 里没有 `clear`，清屏用 `printf '\033c'` 或 `Ctrl+L`）：

```
date
```
```
diskutil list
```
```
ifconfig en0
```

| 检查项 | 期望 | 不符时 |
|---|---|---|
| `date` | 当前 UTC 时间 | `date mmddHHMMccyy`（UTC）修正。例：`date 080918302026` |
| `diskutil list` 能看到内置 NVMe | 盘在 | 若盘不见，停止写入并检查连接、供电、温度、固件和设备健康状态（见 3.10） |
| `diskutil info disk0` 的 `SMART Status` | `Verified` | 非 Verified → 停手，先做 `nvme smart-log` 体检 |
| `ifconfig en0` | 有 IP、`status: active` | 路线 B 不依赖网络，可忽略 |

> 参考：2026-08-09 实测 `en0` 正常获得局域网 IP，`100baseTX full-duplex`，Apple CDN ping 9ms。**板载网口确认可用**，但已用不上了。

### ~~路线 A —— 联网重装~~（★ 已否决，不要再试）

**2026-08-10 实测失败于 error 702，完整诊断见 3.9。**
失败点在下载完成后的解包挂载（`Failed to find the payload dmg` / ENOENT），且每次重试要重下 15.6 GB、并把热盘再烤一遍。
**除非换硬盘，否则此路线视为永久关闭。**

（3.7 记录的 Tahoe 时期 `softwareupdated` 缺失是另一种失败形态，同样指向放弃联网重装。）

### ★ 路线 B —— 离线安装（本文已验证的方案）

前置：U 盘按 6.3 / 6.4 制作 —— 第二分区 **HFS+**，里面放的是**已展开的 `Install macOS Sequoia.app`**，不是 pkg。

5B. 终端确认数据分区已挂载：

```
ls /Volumes
```

若没有 `MACDATA`：`diskutil list` 找到对应的 `diskXsY` 后 `diskutil mount diskXsY`。

> **若这里挂不上**：先确认分区确实是 HFS+ 而不是 exFAT（`diskutil info diskXsY` 看 `Type (Bundle)`；显示 `Microsoft Basic Data` 就是 exFAT，Recovery 认不了 —— 见 3.8）。
> 回制作机**只重做第二分区**，第一分区不动：`sudo mkfs.hfsplus -v MACDATA /dev/sdb2`，重拷 app。

6B. 校验 app 完整（缺 `SharedSupport.dmg` 会在安装中途才报错，浪费一整轮）：

```
ls -l "/Volumes/MACDATA/Install macOS Sequoia.app/Contents/SharedSupport/SharedSupport.dmg"
```

7B. **直接启动安装**（跳过了旧版的 `installer -pkg` 展开步骤，省掉 15.6 GB 写入）：

```
"/Volumes/MACDATA/Install macOS Sequoia.app/Contents/Resources/startosinstall" --volume "/Volumes/Macintosh HD" --agreetolicense --nointeraction
```

> 若 `startosinstall` 报权限错误（`Permission denied`），说明拷贝时没保权限：
> `chmod +x "/Volumes/MACDATA/Install macOS Sequoia.app/Contents/Resources/startosinstall"`，
> 根治办法是回制作机用 `sudo cp -a` 重拷。

**全程盯着温度**：一旦出现卷掉线、`I/O error`、或安装无进展超过 15 分钟，立即中止关机冷却，不要硬扛（3.10）。

> **旧做法已废弃**（2026-08-10）：早期版本是把 `InstallAssistant.pkg` 拷进 U 盘，再在 Recovery 里
> `installer -pkg ... -target "/Volumes/Macintosh HD"` 展开，然后从目标盘上的 app 启动安装。
> 这一步要往热盘上额外写 15.6 GB，是纯粹的浪费。现在改为在 Linux 上预先展开（6.3），**不要退回旧做法**。

### 阶段 4 —— 收尾

8. 之后自动重启 2–3 次。**每次回到 OC 菜单必须手动选对条目**：
   - 第一次重启后选 **`macOS Installer`**
   - 后续选 **`Macintosh HD`**
   - **不要选回 U 盘的 Recovery 条目**，否则从头重来
9. **中途绝对不改 config.plist。**

---

## 八、装好之后的待办（按优先级，一次只改一个变量）

| 优先级 | 事项 | 说明 |
|---|---|---|
| **P0** | 把 EFI 复制到内置盘的 ESP | 之后不用插 U 盘也能启动，同时腾出一个 USB 口 |
| **P0** | 排查"进安装器前 verbose 刷屏数分钟" | v5 记录的现象。回看录像找重复行：`AppleUSBHostPort ... reset` → USB 映射缺失（可能性最大）；`NVMe` 重试 → 存储超时；`ACPI Error` 循环 → ACPI 问题 |
| **P1** | USB 端口映射 | 按 USBToolBox 当前上游说明，在目标机逐口验证并重新生成映射；不要预设它会解决其他问题 |
| **P1** | 加回 `SMCProcessor`（CPU 温度） | 有 `SSDT-EC-USBX` 后可以试。`SMCBatteryManager` 无电池永久禁用；`SMCSuperIO` 裸板意义不大 |
| **P2** | `AppleALC`（若接外置音频） | 裸板无内建喇叭，按需 |
| **P3** | 清理 `npci=0x2000` | 老古董 boot-args，现代 OpenCore 不推荐 |
| **P3** | `Misc/Boot/HideAuxiliary` 改回 `True` | 菜单清爽 |
| **P3** | `SecureBootModel` 考虑改 `Default` | 稳定后 |
| **P4** | 睡眠唤醒测试 | 若异常，再考虑切现代内存映射组合（Rebuild=True / EWU=False / SyncRT=True）。**只在遇到具体问题时才动** |
| — | ~~无线网卡~~ | **本机 M.2 WLAN 槽为空，无此器件。除非加装硬件，否则此项作废** |

---

## 九、调试方法论（v5 沉淀，可复用）

1. `PanicNoKextDump=True` 会屏蔽 kext 列表 —— 调试期必须 `False`
2. 用 **`RIP − kernel text base`** 判断故障是否确定性：绝对地址每次因 KASLR 不同，相对偏移不变。两次相同 = 确定性故障，可放心深挖；不同 = 内存/时序问题，换思路
3. **CR2 的低位要按结构体偏移解读**，不要一看到就当野指针。配合寄存器里的小整数交叉验证
4. **`last started kext` 是极强的时间线证据**
5. backtrace 显示 `Unaligned frame` / `invalid frame pointer` 时不要死磕调用栈 —— 栈已损坏。转而从寄存器 + 地址语义 + kext 时间线三方面推理
6. **地址线索**：本机日志中的高地址与固件映射问题假设相符，但仅凭地址区间不能确认根因，仍需结合内存映射和寄存器证据
7. **OBS 录屏 > 手机拍照**
8. **一次只改一个变量；某个变量解决了问题就立刻固定它**，不要继续试备选方案

---

## 十、路径速查

```
工作目录          <项目根目录>
新 EFI（待生成）   OpCore-Simplify 输出目录
当前 EFI（Tahoe）  <项目根目录>/EFI                                ← 参考用，不直接复用
config 备份        <项目根目录>/EFI/OC/config.plist.bak
Tahoe 安装包       <项目根目录>/macOS Tahoe/InstallAssistant.pkg
                   （本次不用，留作以后升级）
硬件报告           <项目根目录>/硬件报告.txt                        ← AIDA64，GBK 编码
项目导航           <项目根目录>/项目说明.md                         ← 给 AI 的项目上下文
BIOS 设置          <项目根目录>/BIOS设置说明.md
正在使用的 EFI     <项目根目录>/正在使用的EFI/EFI/
Sequoia config     <项目根目录>/sequoia/{config.plist, verify_config.py}
Sequoia 安装包     ★ 本地尚无，需 gibMacOS 下载
历史黄金基准       <项目根目录>/历史版本/config_B_no_devmmio.plist
v5 交接            <项目根目录>/历史版本/交接明细.md
Tahoe 方案 v2      <项目根目录>/Tahoe离线安装执行方案.md
目标 U 盘          /dev/sdb   (Netac OnlyDisk, ~58G, USB)
系统盘（禁写）     /dev/sda   (KINGSTON SKC600MS256G, 238.5G)
```

---

## 十一、执行顺序摘要

```
0. ★ 确认 NVMe 散热、供电和健康状态                     ← 本机旧盘曾出现设备消失(3.10)
1. OpCore-Simplify 重新生成 EFI（目标 macOS Sequoia 15，SMBIOS MacBookPro16,3）
2. 按第四节校正表逐项修改 config.plist
3. 跑第五节验证脚本，必须 FAIL=0；再跑 ocvalidate
4. gibMacOS 下 Sequoia InstallAssistant.pkg (~15GB) + recovery BaseSystem
5. ★ 在 Linux 上把 pkg 展开成 Install macOS Sequoia.app（6.3）  ← 目标盘写入量减半
6. 确认 /dev/sdb 是 U 盘 → 分两区
      sdb1 FAT32  → EFI + com.apple.recovery.boot
      sdb2 HFS+   → Install macOS Sequoia.app     ← 本文已验证的文件系统（见 3.8）
7. 目标机：开 OBS → 引导 → 抹盘 APFS/GUID，名 Macintosh HD
8. 终端自检：date / diskutil list / SMART        ← 盘不见就立即关机冷却
9. 路线 B：直接 startosinstall（不再 installer -pkg）
10. 重启时手动选 macOS Installer → Macintosh HD
11. 装好后按第八节逐项处理，一次一个变量
```

**最容易一击致命的三项，写盘前再确认一遍**：

```
Booter/Quirks/DevirtualiseMmio  = False      ← 本机成功基线，相关证据见 3.1
UEFI/APFS/EnableJumpstart       = True       ← 错了安装中途断死
Kernel/Add 只启用 5 个 kext                   ← 多了会卡死或 panic
```

---
