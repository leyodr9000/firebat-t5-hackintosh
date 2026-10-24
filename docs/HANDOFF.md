# HANDOFF — 工作留痕与交接（Firebat T5 黑苹果）

> **读者**：后续接手的 AI / 人。
> **目标**：不翻聊天记录也能继续工作；所有结论都能用 §8 的命令复现。
> **最后更新**：2026-10-24 15:07 CST
> **写入者**：本次会话（DeepSeek Harness）

---

## 0. 30 秒速览

| 项 | 值 |
| --- | --- |
| 机器 | Firebat T5 / 同方 GM5ZG0O-F，R5-5600H + RTX 3060 Laptop，16 GB，双系统（Windows + macOS） |
| macOS | 15.8.1（24H32，Darwin 24.6.0），OpenCore 1.0.8（`REL-108-2026-09-27`），SMBIOS `MacBookPro16,2` |
| 核显 | Vega 7 `1002:1638`，NootedRed **1.0.1**，Metal 3 |
| 内屏 | 2560×1440 **@240Hz** eDP（BOE），当前逻辑分辨率 **1696×954**（2× 背板 3392×1908） |
| 本次会话做了什么 | ①体检报告 ②240Hz 实测 ③EFI 只读核对 + 改 boot-args ④显示缩放改 1696×954 ⑤N 卡驱动方案评估 |
| 当前状态 | 12:55 重启后稳定 **2 小时+**，无 panic；`-NRedNoSubmitGuard` 已移除，**guard 现在是开启的**（`Applied=4702`） |
| 最大未决项 | guard 开启后是否复现旧的随机 panic（观察中）；240Hz 仍无余量 |

---

## 1. 环境指纹（动手前先核对）

```text
CPU        AMD Ryzen 5 5600H (Cezanne, 6C/12T)
iGPU       Vega 7  1002:1638  ACPI 00:08.1（屏幕由核显点亮）
dGPU       RTX 3060 Laptop  10DE:2520  ACPI \_SB.PCI0.GPP0.PEGP（被 SSDT 屏蔽，IORegistry 中不可见）
内存       16 GB
磁盘       disk0 512G NVMe(FIREBAT) / disk2 1TB NVMe(Windows)
引导 ESP   disk0s1（卷名 "NO NAME"，1.07 GB，OpenCore 在这里）
其他 ESP   disk2s1（Windows 引导）、disk2s6（也有 EFI/OC，疑似早期/备用）
SMBIOS     MacBookPro16,2，SecureBootModel=Disabled，csr-active-config=FF0F0000（SIP 全关）
```

当前 boot-args（EFI `config.plist` 的 `NVRAM/Add/7C436110-…`）：

```text
-v keepsyms=1 debug=0x100 ipc_control_port_options=0 -vi2c-force-polling
-amfipassbeta -lilubetaall itlwm_he=1
-NRedGLFix -NRedVDA -NRedVCNNoGfxOff
```

> 与 10/23 相比**只少了末尾 `-NRedNoSubmitGuard`**（见 §3 变更 1）。

---

## 2. 本次会话时间线

| 时间(10/24) | 动作 | 结果 / 证据 |
| --- | --- | --- |
| 07:50–08:07 | 只读体检：版本、kext、GPU、音频、网络、电池、崩溃报告 | 生成 `hackintosh-status-2026-10-24.md`；发现 6 次 panic + 5 次 shutdown stall |
| 10:46 | 核对 TCC 权限 | `com.deepseek.dsh` 已有 **屏幕录制 + 辅助功能**；`screencapture` 实测可用 |
| 10:49 | CVDisplayLink 测刷新节拍 | **240.0 Hz，间隔 4.165 ms，min=max（零抖动）** |
| 10:52–11:06 | 显示模式 A/B（HiDPI vs 原生 vs 60Hz） | WS CPU / GPU 数据见 §9；结论：240Hz 下没有余量 |
| 11:06 | 写 `display-bench.py`（可复跑） | 见 §7 |
| 12:48 | 用户提供密码，允许动系统 | — |
| 12:53 | 挂载 ESP(rw)、整份备份、改 boot-args、`nvram -d boot-args` | 备份 `EFI-backup-20261024-1253`（66 MB）；`plutil -lint` OK；diff 仅 1 行 |
| 12:55 | 用户重启（boot-args 生效） | `kern.bootargs` 已无该参数 |
| 13:34 | 复查 guard | **`NRedSubmitGuardApplied=2283`**、`WindowServerApplied=1` → 参数确实生效 |
| 13:37 | 更新报告 §4.4/§4.5/§4.6 | 修正"guard 空转"的错误结论 |
| 13:40 | 改显示缩放 | 1280×720（200%）→ **1696×954（150.9%）**，permanent |
| 15:03–15:07 | 评估 N 卡驱动 + EFI 方案 | 生成 `nvidia-dgpu-efi-evaluation.md` |
| 15:20 | 修复 `EFI-独显直连` 两处冲突（副本） | `EFI-独显直连-fixed/`，`ocvalidate 1.0.8` → No issues found |
| 15:13–15:16 | 备份 U 盘 EFI 并写入修复版 | disk3s1（FAT32 209.7 MB）：旧 EFI 56 MB → 新 96 MB；备份在工作目录 + ESP 根 zip |
| 15:24 | 为"不会命令行"准备一键脚本 | 桌面：`N卡-1-检查.command`、`N卡-2-装驱动.command`、`N卡-操作说明.md`（已本地试跑，报告 2.3 KB） |
| 15:40 | 用户首次从 U 盘启动，**卡苹果 Logo / 进度 0** | 原因：U 盘里是 2024 版 `EFI-独显直连`（`AmdTscSync` + 3 个 `AirportItlwm` 版本 + SMBIOS `iMacPro1,1`），与现用系统不匹配 |
| 15:44 | 改用"现用 EFI + 2 处改动"重建并写回 U 盘 | `EFI-usb-nv-stable/`（65 MB / 225 文件）：SMBIOS 回 `MacBookPro16,2`、kext 照搬、ACPI[7] PGPU 关、boot-args 加 `nvfb/nvaccel/nvfbheads/-nvkmsnosmooth`、保留 `-v`；lint + ocvalidate 通过 |
| 15:56 | 又一次 panic（`Kernel-2026-10-24-155609`，`element modified after free`，14 s，`logd`） | panic 文件记录的 boot-args **没有 nvfb** → 是**内置 EFI**启动时崩的，与独显上电无关；相关性指向 12:53 移除的 `-NRedNoSubmitGuard` |
| 16:08 | 回滚：内置 ESP 的 boot-args 加回 `-NRedNoSubmitGuard`，清 NVRAM boot-args | `config-backup-20261024-1610.plist`；`plutil -lint` + `ocvalidate` 通过；下次重启生效 |

---

## 3. 已执行的系统变更（唯一权威清单）

| # | 变更 | 前 | 后 | 备份 / 回滚 | 验证 |
| --- | --- | --- | --- | --- | --- |
| 1 | ESP `config.plist` 的 boot-args 去掉 `-NRedNoSubmitGuard` | 含该参数 | 不含 | ESP 根目录 `EFI-backup-20261024-1253`；工作目录 `efi-config-before-20261024.plist`、`efi-bootargs-before.txt` | `sysctl -n kern.bootargs` 已无该参数；`plutil -lint` OK |
| 2 | 删除 NVRAM `boot-args` 变量 | 有 | 无（下次引导由 OC 按 config 重写） | 值已存 `efi-bootargs-before.txt` | `nvram boot-args` → not found |
| 3 | 显示模式（永久写入） | 1280×720 pt / 2560×1440 px（200%） | **1696×954 pt / 3392×1908 px**（150.9%），240Hz，8-bit | 随时可切回（§8） | `system_profiler SPDisplaysDataType` |
| 4 | 会话中途的临时模式切换（2560×1440 / 1920×1080 / 640×480 / 60Hz） | — | 已全部恢复 | — | 每次脚本结束都 restore |
| 5 | 修复 `EFI-独显直连` 的 2 处冲突（**只改工作目录副本**） | PGPU-DISABLE 启用 + `amfi` 参数与 AMFIPass 并存 | 两份 config：ACPI[0] `Enabled=false`、boot-args 去掉 `amfi_get_out_of_my_way=0x1 amfi=0x80` | 原版在 `D:\ubac\…\EFI-独显直连` **未动**；副本 `EFI-独显直连-fixed/` | `plutil -lint` OK；`ocvalidate 1.0.8` → No issues found |
| 6 | 把修复版写入 **U 盘 EFI**（disk3s1，FAT32 209.7 MB） | 旧的 56 MB EFI（OC + HfsPlus/OpenRuntime/ResetNvramEntry） | 修复版 96 MB（361 文件，30 个 kext） | ①工作目录 `usb-efi-backup-20261024-1514/`（56 MB/166 文件）②U 盘 ESP 根目录 `EFI-backup-20261024-1514.zip`（47 MB） | `plutil -lint` + `ocvalidate` 均过；两份 config 都在；`BOOTx64.efi` 24 KB |
| 7 | 内置 ESP 的 boot-args 去掉 `-v` 与 `debug=0x100`（2026-10-24 17:52） | `-v keepsyms=1 debug=0x100 …` | `keepsyms=1 …`（其余不变，含 `-NRedNoSubmitGuard`） | ESP 根目录 `config-backup-1614.plist`；NVRAM `boot-args` 已清 | `plutil -lint` OK；`ocvalidate` → No issues found；下次重启生效 |
| 8 | 混合模式 boot-args 加 `-nvoff`（2026-10-24 17:55） | 末尾无 `-nvoff` | 末尾 `-nvoff`（NVRM 在混合模式完全不碰 GPU） | 同 #7 的 `config-backup-1614.plist` | `plutil -lint` + `ocvalidate` 通过；下次重启生效 |
| 9 | 停用 `NVMeFix.kext`（2026-10-24 18:05） | Kernel/Add 第 21 条启用 | **Enabled=false** | ESP `config-backup-1805.plist` | `plutil -lint` + `ocvalidate` 通过；原因见 §5 第 10 条 |
| 10 | 验证：18:04:43 重启后 **无 panic**（此前每次开机都先在 0.76 s 崩一次再自动重启） | — | — | — | `last reboot` 只有一次 18:04 重启；`NVMeFix` 实例数 0 |
| 11 | 加诊断参数 `watchdog=1 debug=0x100`（2026-10-24 18:40） | 无 | boot-args 末尾追加 | ESP `config-backup-1838.plist` | 目的：把"静默卡死重启"变成可读的 panic（18:35 那次无 panic 文件、无 SIGTERM，判为看门狗/硬复位） |
| 12 | 电源设置修正（2026-10-24 18:42） | `SleepDisabled=0`（睡眠被重新打开！）、`powernap=1`、`womp=1`、`proximitywake=1`、`tcpkeepalive=1`、`standby=1` | `SleepDisabled=1`、`powernap=0`、`womp=0`、`proximitywake=0`、`tcpkeepalive=0`、`standby=0` | `sudo pmset -a ...` 可随时改回 | 怀疑 18:35 的闲置自动重启是"睡眠/唤醒挂死"（当时 SleepDisabled 已是 0）；改完立即生效 |
| 13 | 按"游戏本常插电"彻底关休眠（2026-10-24 18:45） | `sleep=1`、`hibernatemode=3`、`disksleep=10`、8 GB `sleepimage` | `sleep=0`、`hibernatemode=0`、`disksleep=0`，已删 `/var/vm/sleepimage`（释放 8 GB） | 可 `pmset` 改回 | `pmset -g`：SleepDisabled 1 / sleep 0 / hibernatemode 0 / disksleep 0 |
| 14 | 加"亮度下限守护"（2026-10-24 19:30） | 亮度可拖到 0 → 面板全黑 | LaunchAgent `local.brightnessfloor`，FLOOR 0.23→0.28，5 ms 响应的轮询抬回（**19:45 已移除**，文件归档到 `brightness-daemon-backup/`） | `launchctl unload ~/Library/LaunchAgents/local.brightnessfloor.plist` 并删两个文件 | 实测设 0.15 → 1 s 内回到 0.23；原因：`AppleBacklightDisplay` 的 brightness 22% 仅对应 linear 2%，再低 PWM≈0 |
| 15 | **自编译 NootedRed 亮度曲线补丁**（2026-10-24 19:55） | 官方 1.0.1 kext | `Backlight.cpp`：把 `percentage` 从「linear 比例」改为 `pwm = 2% + linear_frac × 98%`（底部 2% 不黑、顶部 100%）；xmake 2.9.9 + AppleForge（`~/nrd-build/`），ad-hoc 签名，部署版 sha `ccda8a21` | 原版备份：ESP `EFI/OC/Kexts-backup/NootedRed.kext.orig`、工作目录 `NootedRed.kext.orig-1.0.1`；桌面 `回滚-NootedRed.command` | 实测曲线（UI→linear/max）：0.3→0.028、0.5→0.076、0.7→0.209、0.9→0.595；教训：**不要按 linear 比例钳制**，linear 22% ≈ UI 70%，会把大半个滑块钉死 |

**没有动过**：SIP、kext、DeviceProperties、ACPI、键盘映射、Windows 盘（只读挂载）。

---

## 4. 事实 / 推断 / 未知

### 4.1 事实（有命令输出支撑，可复现）

- 6 次 panic（10/22 22:04 – 10/23 18:31），崩溃点分散：`DisplaysExt` / `kernel_task` double fault / `WindowServer` / `airportd` / `searchpartyuseragent` / `QQ`；崩溃时 uptime 17 s – 388 s。
- 5 次 shutdown stall（10/23 18:54–23:07）+ 1 次（10/24 12:55:17）。
- 12:55 之后无 panic。
- macOS 系统里 **NVIDIA kext 数量 = 0**；IORegistry 中无 `10de` 设备（SSDT 已屏蔽 dGPU）。
- 显示枚举：默认 195 个模式（全 1:1）；加 `kCGDisplayShowDuplicateLowResolutionModes` 后 786 个，其中 **240Hz HiDPI 132 个**。
- 显示节拍：CVDisplayLink **240.0 Hz**，间隔 4.165 ms 零抖动（HiDPI 与原生模式都一样）。
- `WindowServer` 合成是**逐帧**的（sample 5 秒：4404 样本中 1442 个在 `CGXUpdateDisplay → CompositorMetal::CompositeLayersToDestination`），静态桌面也不跳过。
- guard 计数轨迹：移除参数前 **0（属性缺失）** → 重启后 **2283** → 15:07 **4702**，WindowServer 被补（PID 159）。
- 部署的 NootedRed 二进制（2,759,008 B，md5 `007b3352…`）与 `/Volumes/STAGE/nrd-src/1.0.1` 完全一致；二进制含 `-NRedSubmitGuard`/`-NRedLegacyBlend`/`-NRedNoLegacyBlend` 三个**源码里没有**的字符串。

### 4.2 推断（有依据，但没做对照实验）

- 旧 panic 与 guard 有关（用户文档结论；移除参数后 2 小时内未复现，但样本不足）。
- 240Hz 负载的主因之一是 HiDPI 背板像素过多（1280×720 HiDPI 只画 3.69 MP，却让 GPU 99%；原生 1440p 只 57–73%）。
- N 卡方案里 `SSDT-AMD-PGPU-DISABLE` + `amfi` 参数与 `AMFIPass` 同存，会导致失败（静态分析，未实测）。

### 4.3 未知（需要实验）

- guard 开启后的**长期**稳定性（观察窗口应 ≥24 h）。
- 240Hz 的实际掉帧率（无帧计数手段：`bufferFlipCount` 恒为 0，CGDisplayStream 的 block 在 ctypes 下不可用）。
- HDMI 口是否走 dGPU（HDMI 音频在 `GPP0.NHDA`，但不能证明视频走线）。
- 键盘（Veekos 68 键）是否有硬件 Mac 模式。
- N 卡驱动对 240Hz / 笔记本 MUX 的支持情况（无样本）。

---

## 5. 踩过的坑（别重复）

1. **用 `nrd-src/NootedRed-master` 源码推断部署二进制行为** —— 错。二进制比那份源码新（多 3 个参数、多 `Applied` 系列属性）。判断运行时行为只能看 `ioreg` 属性，不能看源码。
2. **`ps -o cputime` 解析** —— 早期用"把所有数字加起来"的写法，cputime 超过 1 分钟就错（出现过 −859%）。必须按 `[[dd-]hh:]mm:ss` 解析。
3. **`CGDisplaySetDisplayMode` 设 HiDPI 返回 1000（kCGErrorFailure）** —— 必须走 `CGBeginDisplayConfiguration → CGConfigureDisplayWithDisplayMode → CGCompleteDisplayConfiguration`（`kCGConfigurePermanently=1` 才会重启保留）。
4. **默认 `CGDisplayCopyAllDisplayModes` 看不到 HiDPI** —— 要传 `kCGDisplayShowDuplicateLowResolutionModes` 的字典。
5. **BetterDisplay 菜单栏无法用 AX 自动化**（AXPress 后子树为空，系统无 focused element）—— 改用 CoreGraphics 直接设模式，绕开 GUI。
6. **对 guard 的两次误判**：先按 `Enabled=1` 说"仍生效"，又按 `Applied` 缺失说"空转、删了没影响"。最终 A/B 证明：**删掉参数会让 guard 真正开始工作**（Applied 0→2283）。教训：只看 `Applied` 的**变化**，不要看 `Enabled`。
7. `screencapture -h` 会报 illegal option；用 `-J video` / 不存在的 `-v` 都不行。
8. **拿 2024 版 `EFI-独显直连` 直接启动会卡苹果 Logo / 进度 0**（2026-10-24 15:40 实测）。那套里是 `AmdTscSync` + 三个 `AirportItlwm` 版本同时启用 + SMBIOS `iMacPro1,1`，与现用系统（`ForgedInvariant` + 单个 `AirportItlwm` + `MacBookPro16,2`）差太远。正确做法：**复制现用 EFI，只改要改的那一处**。
9. **不要随便移除 `-NRedNoSubmitGuard`**：2026-10-24 12:53 我把它从内置配置删掉（当作"空转参数"），当日 15:56 就复现了 `element modified after free`（14 s、`logd`）——与 10/22–10/23 的 6 次同签名。16:08 已回滚。特性标记（`Enabled`/`Version`）不能用来判断它是否生效，但**也不能据此删除**。
10. **启动早期 panic 要读 backtrace 定位 kext**：2026-10-24 两次早崩（17:49 `apfsd`/APFS crypto 4.26 s；18:01 `NVMeFix` 空指针 0.76 s），都是"首次启动崩 → 自动重启后正常"。后者 backtrace 明确指向 `org.acidanthera.NVMeFix : __ZN7NVMeFixC2Ev + 0xd96`，已停用 NVMeFix 观察。

| 16:18 | 装 NullMoth 2.5.3（混合模式，dGPU 可见） | 4 个 kext 加载，NVAccel + Metal 3 可用；`recoveryCount=0` |
| 16:46 | 独显直连首次成功进桌面（`iMacPro1,1`，MUX=仅 NVIDIA） | 2560×1440 / Metal 3 / 无 panic，但 **30 Hz**、`ENUMERATED: 0 connected display(s)` |
| 17:02 | `nvfbheads=1` 复测 | 仍是 0 heads；但节拍变稳定 30.0 Hz（间隔 33 ms） |
| 17:07 | 结论：瓶颈在 NVKMS 连接器识别，不是配置 | 详见 `nvidia-dgpu-efi-evaluation.md` §8 |
| 17:11 | **独显直连成功点亮内屏 240 Hz** | 补 `ResizeGpuBars=13` / `ResizeAppleGpuBars=-1` / amfi 参数 / `nvrmsettle=10000` 后生效；实测 239.9 Hz，WS CPU ≈84%，headless 等待 ≈16 s |
| 17:30 | U 盘定位为「独显直连专用」；加 `-nvrmnobootscreen`；分辨率按 135% 调到 1888×1062 | 性能对比数据见 `nvidia-dgpu-efi-evaluation.md` §9（独显直连 WS CPU 84–129% vs 混合 33–42%） |

---

## 6. 未决事项（按优先级）

| # | 事项 | 下一步 | 判据 |
| --- | --- | --- | --- |
| 1 | guard 开启后的稳定性 | 继续用，记录 uptime；≥24 h 无 panic 可认为 1.0.1 已修好 guard | 出现 `element modified after free`/`double fault` → 按 §8 回滚 |
| 2 | 关机卡死 | 12:55 又生成一份 stall；如需排查，用 `spindump -i` 解符号 | — |
| 3 | N 卡方案 | 两处冲突**已修并已写入 U 盘**（disk3s1）；下一步是实机启动验证 NVRM 能否 attach | 见 `nvidia-dgpu-efi-evaluation.md` §6 与 U 盘根目录 `修复说明.md` |
| 4 | 键盘映射 | 三选一：系统设置手点 / 命令写 defaults / 装 Karabiner | 未选定，用户没拍板 |
| 5 | 240Hz 余量 | 可选：建 120Hz 模式、开"降低透明度"、或退回 1600×900 | 用 `display-bench.py` 量 |
| 6 | HDMI 走线 | 插 HDMI 实测（macOS 下有无画面） | — |

---

## 7. 文件索引（工作目录 `~/Documents/deepseek-harness/default-workspace`）

| 文件 | 作用 | 权威性 |
| --- | --- | --- |
| `HANDOFF.md` | 本文件，交接总览 | **最高**（状态以此为准） |
| `hackintosh-status-2026-10-24.md` | 体检报告：平台、kext、GPU/VCN、panic 记录、根因分析、EFI 核对、变更记录（§4.5–4.7） | 高（数据来自实测） |
| `nvidia-dgpu-efi-evaluation.md` | N 卡驱动（NullMoth 2.5.3）与 `EFI-独显直连` 方案评估 | 高 |
| `display-bench.py` | 显示模式负载基准脚本（`--sweep` / `--list`） | 工具 |
| `efi-config-before-20261024.plist` | 改 boot-args 前的 config 快照 | 回滚依据 |
| `efi-bootargs-before.txt` | 改之前的 boot-args 原值 | 回滚依据 |
| `EFI-独显直连-fixed/` | **已修复的 N 卡试验 EFI**（两份 config 各改 2 处，ocvalidate 通过），内含 `修复说明.md` | 待部署 → 已部署到 U 盘 |
| `usb-efi-backup-20261024-1514/` | **U 盘原 EFI 的完整备份**（56 MB / 166 文件），另有同内容 zip 在 U 盘 ESP 根目录 | 回滚依据 |
| `EFI-usb-nv-stable/` | **当前写在 U 盘上的 EFI**（65 MB / 225 文件）= 现用 EFI + PGPU 屏蔽关闭 + nvfb 参数，`-v` 保留；内含 `怎么试.md` | 当前试验版 |
| `~/Desktop/N卡-*.command` | 一键脚本：①只读检查（输出 2.3 KB 报告到桌面）②装驱动（解包 + `sudo ./install.sh` + 日志） | 用户自用 |

**外部资源（只读挂载过的位置）**

| 路径 | 内容 |
| --- | --- |
| `/Volumes/STAGE/nrd-src/NootedRed-master` | NootedRed **1.0.1 源码**（含 `analysis/`、`docs/`） |
| `/Volumes/STAGE/nrd-src/1.0.1/NootedRed.kext` | 与 ESP 上部署的二进制**逐字节一致** |
| `/Volumes/STAGE/panic/` | 用户归档的 panic 副本、`kernel.log`、`kexts.txt`、`nvram.txt` |
| `/Volumes/Untitled`（Windows C:） | `\ubac` 不在 C:；`C:\deepseekharness` 下的 DSH 会话与黑苹果**无关**（扫过，0 命中） |
| `/Volumes/DATA-TEMP/ubac/20240314-OC-EFI项目/EFI升级_20261022/` | **历史 EFI 与工作记录**：升级报告、约 30 个 EFI 变体、`_docs`、`_out/EFI-STABLE-1.0.1*` |
| `…/EFI升级_20261022/_downloads/1401/1401/NullMoth/nullmoth-nvidia-2.5.3.tar.gz` | N 卡驱动包（252 MB） |

---

## 8. 命令速查

### 8.1 状态检查

```sh
sysctl -n kern.bootargs                       # 当前生效的引导参数
ioreg -l -w0 | grep -oE '"NRedSubmitGuard[A-Za-z]*" = [0-9]+'   # guard 关键：看 Applied
pmset -g batt; pmset -g therm
ls -lat /Library/Logs/DiagnosticReports | head
```

### 8.2 显示模式

```sh
# 列模式（含 HiDPI）——必须加 duplicate low-res 选项，否则只有 1:1
python3 display-bench.py --list
python3 display-bench.py            # 测当前模式负载
python3 display-bench.py --sweep    # 测候选并自动恢复
```

常用档位（240Hz HiDPI，括号是等效 Windows 缩放）：
`1872×1053 (136.8%)` / **`1696×954 (150.9%) ← 当前`** / `1600×900 (160%)` / `1280×720 (200%)`

### 8.3 EFI 挂载 / 回滚

```sh
sudo diskutil mount -mountPoint /tmp/efi disk0s1     # 卷名是 NO NAME，必须 -mountPoint
/usr/libexec/PlistBuddy -c 'Print :NVRAM:Add:7C436110-AB2A-4BBB-A880-FE41995C9F82:boot-args' /tmp/efi/EFI/OC/config.plist
sudo diskutil unmount /tmp/efi

# 回滚 boot-args（把参数加回去）
sudo diskutil mount -mountPoint /tmp/efi disk0s1
sudo plutil -replace 'NVRAM.Add.7C436110-AB2A-4BBB-A880-FE41995C9F82.boot-args' -string \
  "-v keepsyms=1 debug=0x100 ipc_control_port_options=0 -vi2c-force-polling -amfipassbeta -lilubetaall itlwm_he=1 -NRedGLFix -NRedVDA -NRedVCNNoGfxOff -NRedNoSubmitGuard" \
  /tmp/efi/EFI/OC/config.plist
sudo diskutil unmount /tmp/efi
```

---

## 9. 关键测量数据（2026-10-24）

### 9.1 显示节拍

| 指标 | 值 |
| --- | --- |
| CVDisplayLink 节拍 | 240.0 Hz（5 s 采 1201 tick） |
| 间隔 | 4.165 ms，min = median = max（零抖动） |
| 标称刷新 | 240.095 Hz |

### 9.2 各模式负载（WindowServer CPU 用 `ps cputime` 差值，已与 `top` 交叉校验）

| 模式 | 帧缓冲 | WS CPU | 每帧 CPU / 预算 | GPU 活动 | dev_util |
| --- | --- | --- | --- | --- | --- |
| HiDPI 1936×1089@240 | 3872×2178 | 32–42 % | 1.3–1.7 / 4.17 ms | 95–99 % | 77–82 % |
| 原生 2560×1440@240 | 2560×1440 | 71–87 % | 3.0–3.6 / 4.17 ms | 57–73 % | 35–52 % |
| 1920×1080@240 | 1920×1080 | 80 % | 3.3 / 4.17 ms | 70 % | 46 % |
| 1280×720@240 | 1280×720 | 98 % | 4.1 / 4.17 ms | 68 % | 50 % |
| 640×480@240 | 640×480 | 2.9 % | 0.1 / 4.17 ms | 5 % | 7 % |
| 2560×1440@60 | 2560×1440 | 40 % | 6.7 / 16.7 ms | 29 % | 21 % |
| **当前 1696×954@240** | 3392×1908 | **74 %** | 3.1 / 4.17 ms | 48 % | 45 % |

> 结论：240Hz 下桌面合成本身就吃掉大半帧预算，任何动画都会掉帧。

### 9.3 guard 轨迹

| 时间 | `Enabled`/`Version` | `Applied` | WindowServer |
| --- | --- | --- | --- |
| 12:50（移除参数前） | 1 / 1 | 缺失（0） | 未打 |
| 13:34（重启后） | 1 / 1 | 2283 | PID 159，已打 |
| 15:07 | 1 / 1 | 4702 | 同上 |

### 9.4 N 卡驱动包（NullMoth 2.5.3）

| 项 | 值 |
| --- | --- |
| 最低系统 | macOS 15.5，**只装 macOS 15** |
| 资格测试 | 15.8.1 + RTX 5060 + 2 屏；刷新率只验 60/75/100 Hz |
| 已知问题 | 改一块屏刷新率后另一块可能黑帧，直到重启 |
| 匹配 | `IOPCIPrimaryMatch = 0x000010de&0x0000ffff`（所有 N 卡，按架构挑） |
| Ampere 痕迹 | `NVRM` 里 `GA106`×64、`Ampere`×37；无 `2520` 字符串、无 laptop 标识 |
| 参数消费者 | `nvfb`/`nvaccel`/`nvkms`/`nvfbheads` 在 NVRM/NVRMFB/NVAccel 中均有引用 |
| 组成 | NVRM / NVRMFB / NVAccel / NVRMAGDC kext + NVMTLDriver(NVMTLDriver.bundle) + nvmtl(Vulkan) + nvdec/nvenc |

---

## 10. 给下一位 AI 的提示

1. **先读 §3 和 §6**：系统已经不是出厂状态，且有一项变更正在观察期。
2. **不要用 STAGE 里的源码推断运行时行为**（见 §5.1），只信 `ioreg` / 日志 / 实测。
3. **动 ESP 前必须备份**，格式参考 `EFI-backup-YYYYMMDD-HHMM`，放在 ESP 根目录。
4. **任何显示/scaling 相关改动**：用 §8.2 的脚本，别手搓 ctypes（有已知的 1000 错误）。
5. **用户偏好**：喜欢直接动手（不要反复确认），但要先给结论和风险；中文交流；文档要能独立看懂。
6. **当前这台机是"能稳定跑"的**：自 12:55 起 2 小时+ 无 panic；不要为了"优化"引入新的不稳定变量。
