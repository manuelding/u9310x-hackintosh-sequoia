# macOS Sequoia 15.7.9 离线安装过程与关键选择（2026-08-13）

**结果**：已成功进入 macOS Sequoia 桌面。

**本文档管什么**：本次成功的安装路径与实际菜单选择，供使用同一安装介质时复现。
安装前准备见 `020_Sequoia_离线安装前准备_2026-08-10.md`；进入桌面后的配置见 `040_装机后配置与待办_2026-08-13.md`。

**本次实际路径**：Recovery + `UnPlugged.command` → Fully expand `InstallAssistant.pkg` → Copy → 图形安装器。

> 执行环境：富士通 LIFEBOOK U9310X 裸主板；OpenCore U 盘插在 USB-A；键鼠占用另一个 USB-A。安装期间保持 U 盘不拔。

## 1. 在 Recovery 格式化内置盘

1. 在 OpenCore 菜单选择 `macOS Base System` / `Recovery`。
2. 打开“磁盘工具”，选择“显示所有设备”。
3. 选中内置 NVMe 的**物理磁盘**，点击“抹掉”，填写：

   | 项目 | 值 |
   |---|---|
   | 名称（卷标） | `Macintosh HD` |
   | 格式 | `APFS` |
   | 方案 / 分区类型 | `GUID Partition Map`（GUID 分区图） |

4. 等待抹掉完成，关闭磁盘工具。

## 2. Recovery 环境自检

从“实用工具”打开“终端”，执行：

```bash
ls /Volumes
which uuidgen hdiutil diskutil pkgutil caffeinate find
df -h /Volumes/Macintosh\ HD
date
```

本次确认结果：

- `/Volumes` 中有 `MACDATA` 和 `Macintosh HD`；另有 `Untitled`，未影响安装。
- `uuidgen`、`hdiutil`、`diskutil`、`pkgutil`、`caffeinate`、`find` 均可用。
- `Macintosh HD` 容量约 233 GiB，空闲约 233 GiB，远高于 40 GiB 的最低要求。

若 `MACDATA` 未出现或无法挂载，应停止本路径；不要在目标 NVMe 上临时搬运安装包。

## 3. 运行离线安装准备脚本

在 Recovery 终端执行：

```bash
cd /Volumes/MACDATA
bash UnPlugged.command
```

脚本可能显示无法从 `InstallAssistant.pkg` 提取所需信息的警告。这在本次 Sequoia Recovery 中出现过；脚本已找到 pkg 时，按 `y` 继续。

按以下顺序作答：

| 菜单 | 选择 |
|---|---|
| 获取 `Install macOS [version].app` 的方式 | `1. Fully expand InstallAssistant.pkg` |
| 安装目标卷 | `2. Macintosh HD` |
| 拷贝方式 | `2. Copy` |
| 任务摘要确认 | `y` |

任务摘要必须包含以下三项才可确认：

```text
Expand InstallAssistant.pkg to a temp folder on /Volumes/Macintosh HD
Copy files to Install macOS [version].app/Contents/SharedSupport
Caffeinate and launch Install macOS [version].app
```

脚本完成的正常标志：终端显示已将 `InstallAssistant.pkg` 复制为 `SharedSupport.dmg`、提示正在启动 `Install macOS Sequoia.app`，然后回到 shell 提示符，同时弹出 macOS Sequoia 图形安装器。

> 本机 Recovery 的“Terminal”菜单没有“New Terminal”功能，只有 Quit；无需为了监控另开窗口，也不要退出正在运行脚本的终端。展开时长时间静默属于正常现象。

## 4. 图形安装与重启选择

1. 在“macOS Sequoia”安装器点击“继续”。
2. 接受许可协议。
3. 选择 `Macintosh HD`，点击“安装”。
4. 第一阶段结束自动重启后，在 OpenCore 菜单选 **`macOS Installer`**，不要选 Recovery。
5. 安装后续自动重启时，按 OpenCore 菜单显示的安装条目继续；完成安装后选 `Macintosh HD`。
6. 进入 macOS 桌面后，安装过程结束；后续按 `040_装机后配置与待办_2026-08-13.md` 执行。
