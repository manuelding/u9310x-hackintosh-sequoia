# U9310X 前期配置：BIOS 升级与设置

> 目标：为安装 macOS（OpenCore）准备稳定、可回退的固件环境。
>
> 本文不包含个人设备编号、硬盘信息、网络信息、文件路径、截图或已有 EFI 配置。不同 BIOS 版本的菜单名称和可选值会不同；请以实际界面及 AI 协助核对结果为准。

## 0. 开始前

- 适用机型：富士通 LIFEBOOK U9310X。
- 本文只处理 BIOS 和固件准备，不指导 macOS、EFI 或磁盘安装。
- BIOS 升级有中断风险。按官方说明提供稳定供电（常见要求是接入电源并保持足够电量）。设备为改装、裸板或供电条件不满足官方要求时，先向原厂确认是否可升级。不要在升级过程中关机、拔电或强制重启。
- 如果设备的散热、供电、显示或 USB 输入不稳定，先修复这些问题，再进行升级或安装。

## 1. 让 AI 协助确认 BIOS 文件

不要凭文件名、论坛转存或“看起来相近”的机型下载固件。打开对话式 AI 后，可以直接发送下面的提示词，并附上官方支持页面的链接或页面截图：

```text
我要为 Fujitsu LIFEBOOK U9310X 升级 BIOS，随后准备使用 OpenCore 安装 macOS。
请只检索并核对富士通官方支持页面：
1. 当前页面是否确实适用于 U9310X；
2. 最新 BIOS 的版本号、发布日期和更新说明；
3. 下载文件名、文件类型、校验信息（若官方提供）；
4. 官方升级步骤、前置条件与注意事项。

不要推荐第三方 BIOS、其他 U93xx 机型的文件，也不要猜测型号兼容性。
请把“已由官方页面确认”和“尚未确认”的内容分开列出，并在我升级前提醒我备份当前 BIOS 设置。
```

确认时至少逐项核对：**完整机型、地区/语言站点、BIOS 版本、发布日期、文件类型和官方升级说明**。若官方页面要求输入序列号、保修编号或登录账号，只在官方页面中自行填写；不要把这些信息贴进 AI 对话、文档或截图。

## 2. 升级 BIOS

1. 在 BIOS 中记录当前 BIOS 版本，并拍一张仅供自己留存的设置照片；照片不要纳入公开文档。
2. 从 U9310X 的**官方支持页面**下载经核对的 BIOS 包。
3. 严格按该版本官方说明执行升级：有的包在 Windows 中运行，有的要求准备 U 盘或从 BIOS 工具启动。不要混用其他机型的教程。
4. 升级期间保持供电和散热稳定，等待工具明确提示完成后再重启。
5. 首次重启后进入 BIOS，确认版本号已更新；如有需要，选择恢复默认设置（Load Setup Defaults），保存并重启一次，再回到 BIOS 进行下一节配置。

如果升级工具报错、机型或版本不匹配、文件校验失败，或无法确认下载来源：**停止操作**。将官方页面链接、报错原文和 BIOS 版本（不要包含序列号）交给 AI 分析，或联系原厂支持。

## 3. BIOS 设置原则

不同固件版本不一定提供全部选项。以下“目标值”是安装 macOS 时的常用方向，不是要求强行寻找隐藏选项：

- 只有 BIOS 实际提供该选项时才修改。
- 找不到项目时，记录“菜单中不存在”，不要通过来路不明的解锁工具或修改版 BIOS 强行开启。
- 每次只改一组设置，保存、重启并确认能再次进入 BIOS；出现异常可恢复默认设置后重新核对。
- 下表中的网卡、音频、摄像头、雷电等设备，是否关闭取决于你的实际安装方案。首次安装优先减少未使用硬件；需要某项功能时再单独启用和调试。

## 4. 候选设置核对表

> 下表不是适用于所有固件版本的推荐配置。只用于帮助识别可能相关的选项；实际取值应以本机 BIOS、当前 OpenCore 指南和所用 macOS 版本为准。

### 启动与安全

常见路径：`Advanced → Boot Configurations`、`Security`、`Boot`。

| 项目 | 目标值 | 说明 |
|---|---|---|
| Fast Boot | Disabled | 让 USB、存储和图形设备充分初始化。 |
| Secure Boot | Disabled | OpenCore 安装通常需要关闭。 |
| CSM / Legacy Boot | Disabled | 使用纯 UEFI 启动。 |
| OS Type（如有） | Other OS / UEFI OS | 不选择 Windows 专用安全启动模式。 |
| Network Boot | Disabled | 不从网络启动时保持关闭。 |
| UEFI Boot On-Screen Keyboard | Disabled | 通常不需要。 |

### 存储与图形

| 项目 | 目标值 | 说明 |
|---|---|---|
| SATA Mode（如有） | AHCI | 不使用 RAID / RST 模式。 |
| DVMT Pre-Allocated / iGPU Memory（如有） | 64 MB 或更高 | 以 BIOS 提供的可选值及后续 EFI 方案为准。 |
| Above 4G Decoding（如有） | Enabled | 先记录实际选项；与 EFI 配置一起复核。 |
| Resizable BAR（如有） | Disabled | 旧平台安装 macOS 时通常关闭。 |

### CPU 与虚拟化

常见路径：`Advanced → CPU Features`。

| 项目 | 目标值 | 说明 |
|---|---|---|
| Multi-core | Enabled | 保留全部 CPU 核心。 |
| HT Technology | Enabled | 保留超线程。 |
| Intel Speed Shift | Enabled | 有利于动态调频。 |
| Virtualization Technology (VT-x) | Enabled | 可保持开启。 |
| Intel VT-d | Disabled | 若无法关闭，后续需在 OpenCore 中配置对应兼容项。 |
| Intel TXT | Disabled | 保持关闭。 |
| Intel SGX | Disabled；没有该值时保持默认 | 不强行修改。 |
| CFG Lock（如有） | Disabled | 若 BIOS 不提供，后续由 EFI 配置处理；不要刷修改版 BIOS。 |

### USB 与内建设备

常见路径：`Advanced → USB Features`、`Advanced → Internal Device Configurations`。

| 项目 | 目标值 | 说明 |
|---|---|---|
| Legacy USB Support | Enabled | 确保 BIOS / OpenCore 阶段键盘可用。 |
| USB Port | Enabled | 安装 U 盘和输入设备需要。 |
| XHCI Controller Setting | Standard | 不要选择 Compatible。 |
| 音频、摄像头、指纹、读卡器、传感器 | 不使用则 Disabled | 减少首次安装变量；需要时可恢复。 |
| 有线网卡 | Enabled（需要联网时） | 若安装过程依赖有线网络，不要关闭。 |
| Thunderbolt Device / Boot Support | 不使用则 Disabled | 关闭后相关 USB-C / 雷电接口可能不可用，属于预期现象。 |

### 电源、唤醒与散热

常见路径：`Advanced → Miscellaneous Configurations`。

| 项目 | 目标值 | 说明 |
|---|---|---|
| Wake up on LAN / Wake up on USB | Disabled | 首次安装避免意外唤醒。 |
| Resume on LAN | On AC Mode Only 或 Disabled | 按 BIOS 实际可选值选择。 |
| Auto Save To Disk | Off | 避免固件休眠机制干扰。 |
| Power Delivery on System-Off | Disabled | 不需要关机供电时关闭。 |
| USB 快充 | Normal Charge 或 Disabled | 优先稳定与低发热。 |
| FAN Control | 可选项中散热最强的模式 | 不选 Silent / Quiet；若只有 Normal 则保持 Normal。 |

## 5. 保存前的复核提示词

修改完成后，可将**不含设备编号的 BIOS 菜单文字或打码截图**发给 AI，并使用：

```text
这是 U9310X 为 OpenCore/macOS 安装准备的 BIOS 设置。
请逐项核对以下目标：UEFI、关闭 Secure Boot 和 Fast Boot、AHCI（如有）、USB Legacy/XHCI、VT-d、CFG Lock、未使用内建设备及散热设置。
请只根据截图中实际存在的选项判断；不要把未出现的选项当作错误，也不要建议修改版 BIOS。
输出三栏：已确认、建议调整、无法从资料确认。
```

截图脱敏方法：裁掉或遮住 `Serial Number`、`UUID`、`Asset Tag`、MAC 地址、Windows 产品密钥、用户名、设备管理编号和任何二维码。只保留菜单路径、设置项与当前值即可。

## 6. 完成标准与下一步

完成以下内容即可进入后续 EFI / 安装盘准备：

- BIOS 版本已从官方页面核对并成功升级（或确认无需升级）；
- 已保存一份仅供自己回退的原始设置记录；
- 已启用 UEFI、USB 支持及必要网络，关闭 Secure Boot 与 Fast Boot；
- 其他设置已按本机 BIOS 实际存在的选项核对；
- 未向 AI、公开仓库或文档提交任何设备唯一标识、网络信息或私人截图。

下一篇文档应从 EFI 与安装介质准备开始，并继续遵循“先让 AI 读取公开资料与当前状态、一次只改变一个变量、每步留下可回退记录”的原则。
