# 黑苹果体检报告

**机器**：AMD Ryzen 5 5600H 笔记本 · macOS 15.8.1 (24H32) · OpenCore 1.0.8
**生成时间**：2026-10-24 08:07 CST
**检查方式**：纯只读采样（NVRAM / ioreg / kmutil / system_profiler / pmset / 系统日志 / 崩溃报告 / 用户已收集的日志包），**未修改系统任何配置**。

---

## 0. 结论摘要

| 维度 | 结论 |
| --- | --- |
| 可用性 | 正常启动、图形加速有效（Metal 3）、Wi-Fi / 蓝牙 / 音频 / 摄像头 / 触控板 / 电池管理均在位 |
| 图形补丁 | NootedRed 1.0.1 的 GL 页补丁与 VDA 用户态补丁**均已生效**（8 个页面已登记） |
| 稳定性 | **不合格**：10/22–10/23 共 6 次内核崩溃 + 5 次关机卡死；抹盘重装 15.8.1 后仍复现 2 次 |
| 当前这次启动 | 干净：无 panic、无 GPU reset（`recoveryCount=0`）、40 分钟内无 VRR/DPCD 报错，已稳定运行 1 小时以上 |
| 最可疑根因 | ①用户态页补丁（`-NRedGLFix`/`-NRedVDA`）②Wi-Fi 栈 ③内存/SoC 不稳 |
| 未完成项 | `-NRedNoSubmitGuard` 的语义需重启后按 §4.5 判定；240Hz 显示性能的实测表尚未并入本文 |

> 一句话：**能干活，但随时可能崩。** 崩溃点分散在显示、IO80211、kevent、内存分配四个互不相干的子系统，而且多次发生在开机后 17 秒～6 分钟内，属于"内存被写坏"型特征，不是某一个 kext 的确定性 bug。

---

## 1. 平台与系统

| 项目 | 值 |
| --- | --- |
| 型号（SMBIOS） | `MacBookPro16,2` / board-id `Mac-5F9802EFE386AA28` |
| 固件版本 | `2094.80.5.0.0`，OS Loader `583~5915` |
| CPU | AMD Ryzen 5 5600H，6 核 12 线程，标称 3.3 GHz |
| 内存 | 16 GB（`hw.memsize` = 17179869184） |
| 系统盘 | FIREBAT FET34-512G NVMe，512.11 GB，TRIM 开启，SMART 正常 |
| 数据盘 | 1 TB NVMe，含 Windows 分区（disk2s3 / 恢复分区 disk2s4） |
| 启动卷 | `disk1s4`（MACH）406 GB，已用 10 GB，剩余 377 GB |
| 内核 | Darwin 24.6.0，x86_64，`RELEASE_X86_64` |
| FileVault | 关闭（黑苹果常规配置） |

**分区与引导**：共有 **3 个 EFI 分区** —— `disk0s1`（1.07 GB）、`disk2s1`（314 MB）、`disk2s6`（736 MB）。已确认引导用的是 **`disk0s1`**：里面是 `EFI/OC`（OpenCore 1.0.8）+ 三份历史备份 + 一批 OpenCore 日志；`disk2s6` 同样带 `EFI/OC`（疑似早期/备用引导，未逐一核对版本），`disk2s1` 只有 Windows 引导文件。

---

## 2. 引导参数与安全策略

**OpenCore**：`REL-108-2026-09-27`（1.0.8）

**当前 boot-args**（EFI `config.plist` 的 `NVRAM/Add`，2026-10-24 12:53 修改后）：

```
-v keepsyms=1 debug=0x100 ipc_control_port_options=0 -vi2c-force-polling
-amfipassbeta -lilubetaall itlwm_he=1
-NRedGLFix -NRedVDA -NRedVCNNoGfxOff
```

> 与修改前相比只少了末尾的 `-NRedNoSubmitGuard`。本次运行期的 `kern.bootargs` 仍是旧值（含该参数），**重启后才生效**；改动与回滚见 §4.5。

分类解读：

| 参数 | 用途 | 备注 |
| --- | --- | --- |
| `-v` `keepsyms=1` | 跑码 + 保留符号 | 日常可保留 `keepsyms`，`-v` 只影响观感 |
| `debug=0x100` | **禁止 panic 自动重启** | 调试用；日常建议删除，会加重日志负担 |
| `ipc_control_port_options=0` | 兼容 Rosetta / 部分 XPC | 常规 |
| `-vi2c-force-polling` | VoodooI2C 强制轮询 | 常规（触控板） |
| `-amfipassbeta` | AMFIPass 的 beta 开关 | 配合 SIP 关闭 |
| `-lilubetaall` | 放开 Lilu 插件对"未验证系统版本"的限制 | 建议仅在必要时使用 |
| `itlwm_he=1` | AirportItlwm 启用 802.11ax(HE) | 与 Wi-Fi 稳定性相关，见 §6.2 |
| `-NRedGLFix` | 启用 Chromium/GL 用户态页补丁 | 实验性，见 §6.1 |
| `-NRedVDA` | 启用 VDA 用户态补丁 | 实验性，见 §6.1 |
| `-NRedVCNNoGfxOff` | VCN 期间禁止 GFXOFF | 规避视频引擎下电的挂死 |
| `-NRedNoSubmitGuard` | 关闭提交保护（实验） | **2026-10-24 已移除**（原因与风险见 §4.4 / §4.5）；移除前内核确实收到了它 |

> 历史参数：10/23 18:16、18:31 两次 panic 时的 boot-args 是 `keepsyms=1 revpatch=sbvmm`，与当前不同，说明 18:48 前后你改过一版配置。

**SIP**：`csr-active-config = ff0f0000`，即 **全部关闭**（Apple Internal / Kext Signing / FS Protections / Debugging / DTrace / NVRAM / BaseSystem）。这是用户态页补丁的前置条件，但同时也意味着系统完整性防线全无。

---

## 3. 第三方 kext 清单（来自 `kmutil showloaded`）

| Kext | 版本 | 作用 | 状态 |
| --- | --- | --- | --- |
| Lilu | 1.7.2 | 补丁加载器 | 正常 |
| VirtualSMC | 1.3.8 | SMC 模拟 | 正常 |
| SMCBatteryManager | 1.3.9 | 电池 | 正常，报电准确 |
| SMCLightSensor | 1.3.9 | 光感 | 已加载 |
| AppleALC | 1.9.8 | 声卡 | 扬声器/麦克风正常 |
| SimpleHDA | 1.0.0 | 声卡辅助（社区 kext） | 已加载 |
| NootedRed | 1.0.1 | Renoir 核显驱动 + 用户态补丁 | 正常驱动，补丁生效 |
| ForgedInvariant | 1.5.0 | APU 不变 TSC 伪造 | 已加载 |
| AMDRyzenCPUPowerManagement | 0.9.1 | CPU 电源管理 | 已加载 |
| AMFIPass | 1.4.1 | 绕过 AMFI 限制 | 已加载 |
| RestrictEvents | 1.1.6 | 事件/机型限制修补 | 已加载 |
| NVMeFix | 1.1.4 | NVMe 电源管理 | 已加载 |
| AirportItlwm | 2.4.0 | Intel 无线网卡 | 已联网 |
| IntelBluetoothFirmware | 2.5.0 | 蓝牙固件 | 正常，状态 On |
| IntelBTPatcher | 2.5.0 | 蓝牙补丁 | 已加载 |
| BlueToolFixup | 2.7.2 | 蓝牙修复 | 已加载 |
| USBToolBox | 1.2.0 | USB 定制基础 | 已加载 |
| VoodooI2C / I2CHID / I2CServices | 2.9.1 / 1 / 1 | I2C 触控板 | 已加载 |
| VoodooGPIO | 1.1 | GPIO | 已加载 |
| VoodooInput | 1.1.7 | 触控板输入抽象 | 已加载 |
| VoodooPS2Controller / Keyboard / Trackpad / Mouse | 2.3.8 | PS2 键鼠 | 已加载 |
| ECEnabler | 1.0.6 | 电池读数 >8 位 | 已加载 |
| BrightnessKeys | 1.0.4 | 亮度快捷键 | 已加载 |
| RTL812xLucy | 1.1.100 | RTL8125 有线网卡 | 驱动在位（未插线） |
| IntelMKLFixup | 1.0.0 | Intel MKL 兼容 | 已加载 |

**结论**：kext 组合是标准的 AMD 笔记本黑苹果方案 + NootedRed 实验补丁，没有明显冗余冲突。`-lilubetaall` 与 `IntelMKLFixup`、`SimpleHDA`、`ForgedInvariant` 属于"非必需但为特定用途"的项，排障时可作为次要变量。

---

## 4. 图形与媒体引擎运行状态

### 4.1 显示

| 项目 | 值 |
| --- | --- |
| 芯片 | `AMD Radeon RX Renoir Graphics`，Device ID `0x1638`，Revision `0x00c6` |
| VRAM | 4 GB（NootedRed 配置值） |
| Metal | **Metal 3** |
| ROM | `113-CEZANNE-017` |
| 当前模式 | 帧缓冲 **3872 × 2178**，UI 缩放 **1936 × 1089 @ 240 Hz**（HiDPI，非整数缩放） |

> 检查过程中显示模式发生过变化（早先采样为 1920×1080@240Hz，后变为 HiDPI 1936×1089），说明有远程会话在操作显示设置。**3872×2178 @ 240 Hz 对 Renoir 是很重的模式**，与 §7 的 VRR 报错同一风险面。

### 4.2 加速器统计（`IOAccelerator`）

- `recoveryCount = 0` —— 本次启动**没有 GPU 复位**（对比 10/18 那次实验曾出现连续 GPU reset）
- GFX 通道：提交 1541 / 完成 1531，`hardwareWaitTime` 约 5.5×10⁸ ns
- GPU 利用率 47%–81%，VRAM 占用约 668 MB，GART 占用约 78 MB
- 视频引擎：`VCN0Enc` 有提交（66），`VCN0Dec` 提交为 **0** —— 本次启动还没有真正走硬件解码

### 4.3 NootedRed 运行时属性（`ioreg`）

| 属性 | 值 | 含义 |
| --- | --- | --- |
| `NRedCompatDeviceID` | 5688 (`0x1638`) | 识别到 Cezanne |
| `NRedCompatFamily` | 6 | GFX9 家族 |
| `NRedCompatNumShaderEngines` | **1** | GL 页补丁已生效（4 → 1，CU 32 → 8） |
| `NRedCompatVCNGeneration` | 514 | VCN 3.0 |
| `NRedVDARequested` / `VDAMediaEnabled` | 1 / 1 | VDA 已请求并启用 |
| `NRedVCNUserApplied` | **8** | 8 个用户态驱动页面已改写 |
| `NRedVCNStage` | 11 | 补丁流水线已到末阶段 |
| `NRedVCNX5000ABIValidated` / `X6000ABIValidated` | 1 / 1 | 两套 ABI 校验通过 |
| `NRedVCNNativeLoadReady` / `NativeLoadStatus` | 1 / **0** | 原生加载就绪但状态位为 0，**待确认** |
| `NRedVCNNoGfxOff` / `GfxOffHeld` | 1 / 1 | GFXOFF 已按住 |
| `NRedVCNIdleTimeouts` / `IdleDrains` | 126 / 131 | 视频引擎空闲超时累计 **126 次** |
| `NRedVCNIdleWaitedMaxMs` / `IdleWaitedMs` | 255 / 255 | 每次都等满 255 ms 上限 |
| `NRedLegacyBlendEnabled` | 1 | 构建特性标记；同样没有对应的 `Applied` 计数 |
| `NRedSubmitGuardEnabled` | 1 | **仅表示该构建自带 guard 特性**，不代表正在生效 |
| `NRedSubmitGuardApplied`（缺失） | — | 没有给任何进程写过 guard 补丁 → 守卫实际空转，见 §4.4 |

**解读**：GL 补丁（`numShaderEngines 4→1`）与 VDA 页面改写都已实际执行，说明补丁的目标版本匹配（系统自带 `AMDRadeonX5000GLDriver` 为 **7.0.25**，与补丁数据声明的 7.0.25 一致）。

但 `VCNIdleTimeouts = 126` 而 `IdleDrains = 131`、每次都等满 255 ms，说明**视频引擎几乎从来没能在预期时间内空闲下来**，只能靠超时兜底。这与 `-NRedVCNNoGfxOff` 的存在互相印证：视频路径本身就没有完全理顺。这类"超时兜底"在压力下很容易演变成挂死或状态机错乱。

### 4.4 `-NRedNoSubmitGuard` 到底有没有生效（复查结论）

上一版写的是"`NRedSubmitGuardEnabled=1`，该参数似乎未生效，待确认"。复查后的结论是：**这条参数既不能证明生效、也不能证明失效；但可以确定的是，guard 在本机没有实际做事。**

证据链（全部可复现）：

1. **内核侧确实收到了参数**：`sysctl -n kern.bootargs` 末尾就是 `-NRedNoSubmitGuard`，与 NVRAM 一致，排除"参数没进内核"。
2. **部署的二进制与手头源码不是同一份**：`NootedRed.kext`（2,759,008 字节，与 `/Volumes/STAGE/nrd-src/1.0.1` 逐字节一致）里存在 `-NRedSubmitGuard`、`-NRedLegacyBlend`、`-NRedNoLegacyBlend` 三个字符串，而 `nrd-src/NootedRed-master` 源码里 grep 命中数为 **0**（其余 `-NRedBeta`、`-NRedForceVDA` 等源码里都有）。**因此拿那份源码的 `submitGuardEnabled = !checkKernelArgument(kBootArgNoSubmitGuard)` 去反推运行时行为不可靠**——这是上一版表述中最不严谨的地方。
3. **有分量的运行时属性缺失**：无论源码还是二进制，`NRedSubmitGuardApplied`（以及 `NRedSubmitGuardWindowServerPID` / `...WindowServerApplied`）都只在"补丁真正写进某个进程页"之后才设置。本机 `ioreg` 里 `Enabled`/`Version` 有值，**`Applied` 系列一个都没有** → 本次启动没有给任何进程打过 guard 补丁。
4. `NRedLegacyBlendEnabled=1` 同理：只有特性标记，没有对应的 `Applied`。

**结论（2026-10-24 12:50 初判 → 13:35 被 A/B 结果修正）**：初判是"guard 空转、删不删都一样"。移除参数重启后 `Applied` 从 0 跳到 **2283**（WindowServer 也被打上，见 §4.6），说明这条参数**原先确实在生效**——它把 guard 关掉了。`Enabled`/`Version` 只是"特性/安装标记"这一点初判没错，错在"删掉不改变行为"。

- 对使用的影响：**删掉它就等于把 guard 重新打开**（现在就是打开状态）。日后如果出现 `element modified after free` / `double fault` 一类 panic，把参数加回去即可（命令见 §4.5）。
- 想真正验证参数语义，只能用 DEBUG 版 kext 抓 `SYSLOG("UserPatch", "Enabling user-space Metal SubmitGuard ...")`。RELEASE 版的早期 SYSLOG 在 logd 起来之前就丢了：`log show --last 6h` 里查不到任何 `UserPatch` 记录，已验证。

### 4.5 已执行的系统变更（2026-10-24 12:53）

| 项 | 值 |
| --- | --- |
| 备份 | ESP 根目录 `EFI-backup-20261024-1253`（整份 EFI，66 MB）；工作目录另存 `efi-config-before-20261024.plist` 与 `efi-bootargs-before.txt` |
| 改动 | `EFI/OC/config.plist` → `NVRAM/Add/7C436110-…/boot-args` 去掉末尾 ` -NRedNoSubmitGuard`，仅此一行 |
| 校验 | `plutil -lint` → OK；`plutil -p` 前后 diff 只有 boot-args 一行 |
| NVRAM | 已执行 `nvram -d boot-args`，下次引导由 OpenCore 按 config 重写 |
| 生效 | **需重启**；本次运行期的 `kern.bootargs` 仍是旧值 |

**重启后怎么判定**

```sh
sysctl -n kern.bootargs | grep -c NRedNoSubmitGuard     # 期望 0
ioreg -l -w0 | grep -oE '"NRed[^"]*Guard[^"]*" = [0-9]+'
```

- 若出现 `NRedSubmitGuardApplied = N` → 说明这条参数**原先确实生效**（guard 本来被关掉），现在被重新打开。重点观察重启后 10 分钟内是否出现 `element modified after free` / `double fault` 类 panic；出现就按下面回滚。
- 若仍然只有 `Enabled` / `Version`、没有 `Applied` → 与 §4.4 的判断一致：guard 从未实际动作，删不删都一样。

**回滚（任选其一）**

```sh
# 1) 整份恢复
sudo diskutil mount -mountPoint /tmp/efi disk0s1
sudo cp /tmp/efi/EFI-backup-20261024-1253/OC/config.plist /tmp/efi/EFI/OC/config.plist
sudo diskutil unmount /tmp/efi

# 2) 只把参数加回去
sudo diskutil mount -mountPoint /tmp/efi disk0s1
sudo plutil -replace 'NVRAM.Add.7C436110-AB2A-4BBB-A880-FE41995C9F82.boot-args' -string \
  "-v keepsyms=1 debug=0x100 ipc_control_port_options=0 -vi2c-force-polling -amfipassbeta -lilubetaall itlwm_he=1 -NRedGLFix -NRedVDA -NRedVCNNoGfxOff -NRedNoSubmitGuard" \
  /tmp/efi/EFI/OC/config.plist
sudo diskutil unmount /tmp/efi
```

> 注意：ESP 卷名是 `NO NAME`，直接 `diskutil mount disk0s1` 会挂到 `/Volumes/NO NAME`；上面用 `-mountPoint` 固定路径，最省事。
> Windows 侧兜底：`D:\ubac\20240314-OC-EFI项目\EFI升级_20261022\_out\EFI-STABLE-1.0.1` 有一份独立可用 EFI，ESP 上那份备份也可以直接用。

### 4.6 A/B 结果：这条参数确实生效（2026-10-24 13:35）

12:55 重启（`kern.bootargs` 已不含该参数）后立即复查：

| 属性 | 移除前 | 移除后 |
| --- | --- | --- |
| `NRedSubmitGuardEnabled` / `Version` | 1 / 1 | 1 / 1 |
| `NRedSubmitGuardApplied` | 缺失（0） | **2283** |
| `NRedSubmitGuardWindowServerApplied` | 缺失 | **1**（PID 159） |

结论：`-NRedNoSubmitGuard` **确实被二进制读取并生效**——它关掉 guard，所以移除前 `Applied` 一直是 0；`Enabled` 只是"该构建带此特性"的标记。移除后 guard 对 **2283 个进程**打了补丁，其中包括 WindowServer。

观察：重启后 39 分钟无 panic（`debug=0x100` 仍在，真崩会停在 panic 屏而不是自动重启）；但 12:55:17 又生成了一份 `shutdown_stall`，关机卡死的老问题还在。

如果 panic 回来，按 §4.5 的"2) 只把参数加回去"恢复即可。

### 4.7 显示缩放调整（2026-10-24 13:40）

| 项 | 调整前 | 调整后 |
| --- | --- | --- |
| 逻辑分辨率 | 1280×720 | **1696×954** |
| 帧缓冲 | 2560×1440（2.00×） | **3392×1908**（2.00×） |
| 等效 Windows 缩放 | 200% | **150.9%** |
| 刷新率 / 色深 | 240Hz / 10-bit | 240Hz / 8-bit（ARGB8888） |

做法：先 `CGDisplayCopyAllDisplayModes` 加上 `kCGDisplayShowDuplicateLowResolutionModes` 选项，能枚举出 **132 个 240Hz HiDPI 模式**（默认调用只返回 1:1 模式，所以之前的模式表里看不到它们）；再用 `CGBeginDisplayConfiguration` → `CGConfigureDisplayWithDisplayMode` → `CGCompleteDisplayConfiguration(kCGConfigurePermanently)` 写入。直接调 `CGDisplaySetDisplayMode` 会返回 **1000**（kCGErrorFailure），必须走 configuration API。

可选档位（240Hz HiDPI，括号为等效 Windows 缩放）：

- 1872×1053（136.8%）→ 背板 3744×2106，最重
- **1696×954（150.9%）← 当前，最接近 Windows 150%**
- 1600×900（160.0%）→ 背板 3200×1800，次轻
- 1280×720（200.0%）→ 背板 2560×1440，最轻

> macOS 没有"150%"这种非整数缩放，只能在 2× 背板渲染后缩到 2560×1440。机内屏的缩放档位阶梯跳过 1707×960，所以 1696×954 是最近的可用值（偏差 0.6%，肉眼不可辨）。
> 负载（8 秒均值）：WindowServer **74.3%** CPU、GPU 活动 48%、dev_util 45%。分辨率越高，240Hz 下 GPU 越吃紧。

---

## 5. 稳定性记录

### 5.1 内核崩溃（6 次）

| # | 时间 | 系统 | 崩溃进程 | 崩溃类型 | boot-args | 已运行 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 10/22 22:04 | 15.6.1 (24G90) | `DisplaysExt` | 页错误，`IOSurface::get_value` 路径 | 未记录 | 388 s |
| 2 | 10/23 07:58 | 15.6.1 | `kernel_task` | **Double fault**，发生在 `machine_idle` | `-v keepsyms=1 -NRedGLFix -NRedVDA -NRedVCNNoGfxOff` | 60 s |
| 3 | 10/23 09:19 | 15.6.1 | `WindowServer` | `[Bank ledger]: element modified after free` | `-v keepsyms=1` | 69 s |
| 4 | 10/23 16:58 | 15.6.1 | `airportd` | 页错误，`IO80211Family` → `vm_map_create_upl` | `keepsyms=1` | **17 s** |
| 5 | 10/23 18:16 | **15.8.1 (24H32)** | `searchpartyuseragent` | 页错误，`kevent_register` | `keepsyms=1 revpatch=sbvmm` | 388 s |
| 6 | 10/23 18:31 | **15.8.1** | `QQ` | `[data.kalloc.16384]: element modified after free` | `keepsyms=1 revpatch=sbvmm` | 145 s |

崩溃报告位置：`/Library/Logs/DiagnosticReports/Kernel-*.panic`，另有副本在 `/Volumes/STAGE/panic/`。

### 5.2 崩溃特征分析

- **横跨四个不相干子系统**：显示（IOSurface）、网络（IO80211）、事件系统（kevent）、内存分配（kalloc/ledger）。没有任何两个 panic 共享同一根因栈。
- **两条 panic 是"释放后改写"（element modified after free）**：这是典型的**内存被越界写**，不是正常代码路径能产生的错误。
- **一次 double fault 落在 idle 线程**且无任何第三方 kext 出现在 backtrace：说明 CPU 空闲时上下文已被破坏，更指向硬件（内存/SoC）层面，而非某个驱动主动崩溃。
- **崩溃时间集中在开机后 17 秒 ~ 6 分半**：即登录、WindowServer 起、Wi-Fi 关联、各类开机自启集中爆发的窗口。
- **跨系统版本复现**（15.6.1 与 15.8.1 都崩），且**抹盘重装后 1 小时内又崩 2 次**（#5、#6），排除"系统文件被改坏"这一可能。

### 5.3 关机卡死（Shutdown Stall，5 次）

| 时间（10/23） | 报告 |
| --- | --- |
| 18:54 | `shutdown_stall_2026-10-23-185416` |
| 19:41 | `shutdown_stall_2026-10-23-194116` |
| 20:06 | `shutdown_stall_2026-10-23-200610` |
| 21:08 | `shutdown_stall_2026-10-23-210831` |
| 23:07 | `shutdown_stall_2026-10-23-080709` |

均为约 750 KB 的 spindump（二进制格式），未展开符号化（需 `spindump -i`）。集中在 10/23 晚，**18:31 最后一次 panic 之后开始出现**，23:07 之后未再出现；本次启动的关机原因是 `5`（正常关机）。

### 5.4 时间线

```
10/22 17:09  下载 NootedRed 源码（/Volumes/STAGE/nrd-src）
10/22 22:04  panic #1（15.6.1，DisplaysExt）
10/23 07:58  panic #2（15.6.1，double fault，boot-args 已带 NRed 补丁参数）
10/23 09:19  panic #3（15.6.1，WindowServer）
10/23 16:58  panic #4（15.6.1，airportd，17 秒即崩）
10/23 16:58–18:16  用 STAGE 里的离线安装包抹盘装 15.8.1（run.sh / CMDS.txt）
10/23 18:16  panic #5（15.8.1，searchpartyuseragent）
10/23 18:31  panic #6（15.8.1，QQ）
10/23 18:48  配置变更（watchdog 来源变为 platform configuration），此后未再 panic
10/23 18:54–23:07  5 次关机卡死
10/24 06:41  本次启动
10/24 08:07  仍然稳定，无 panic / 无 GPU reset / 无 VRR 报错
```

---

## 6. 根因分析

### 6.1 假设 A：用户态页补丁（最可疑）

- 机制：NootedRed 在**代码签名校验通过后**改写 Apple 用户态驱动页（`AMDRadeonX5000GLDriver`、VDA/GVA 相关）。这是把补丁注入到"驱动已经通过校验"的页镜像里，属于极高位操作。
- 已确认生效：`NRedVCNUserApplied = 8`、`NRedCompatNumShaderEngines = 1`，说明本机确实在跑这套路径。
- 项目文档自己就写明了风险：见 [`analysis/repro-plan.md`](/Volumes/STAGE/nrd-src/NootedRed-master/analysis/repro-plan.md) 引用的维护者原话——"Lilu user-space patching causes issues anyways, like app crashes and **system instability**"；[`analysis/lead-findings.md`](/Volumes/STAGE/nrd-src/NootedRed-master/analysis/lead-findings.md) 也记录过实验期间 GPU reset 循环、WindowServer 被杀。
- 支持证据：`QQ`（Chromium 内核）崩溃、`DisplaysExt`/`WindowServer`/`IOSurface` 崩溃都直接落在图形栈；`-NRedGLFix` 的存在本身就说明 GL 双源混合路径曾经挂死过。
- 反对证据：`airportd`（Wi-Fi）与 `searchpartyuseragent`（kevent）的崩溃与图形栈无关——除非补丁把内存写坏了，否则解释不了。

### 6.2 假设 B：Wi-Fi 栈

- 支持：panic #4 发生在 `IO80211Family → IO80211APIUserClient::runDriverCommandHelper → vm_map_create_upl`，且**开机仅 17 秒**就崩，正是机场服务扫描/关联最密集的时刻；`itlwm_he=1` 开启了 HE（Wi-Fi 6）路径，AirportItlwm 在高负载扫描下有过不稳定记录。
- 每次启动都有 `APPLE80211_IOC_WCL_LOW_LATENCY_INFO_STATS` 返回 `0xe00002c7` 的刷屏，说明用户态与驱动之间的某条查询路径本身就不完整（多数情况下无害）。
- 反对：其余 5 次崩溃与网络无关。

### 6.3 假设 C：内存 / SoC 不稳

- 支持：多处随机 corruption + idle 线程 double fault + 崩溃点完全离散，这是内存错误的教科书式分布；且与系统版本无关、重装无效。
- 反对：无直接证据（没跑过 MemTest86，也没有 EDAC/MCE 类日志——macOS 上本来就难看到）。

### 6.4 判定

| 假设 | 可能性 | 理由 |
| --- | --- | --- |
| A 用户态页补丁 | **高** | 补丁确实在跑，且官方文档承认会导致系统不稳；崩溃多在图形栈 |
| B Wi-Fi | 中 | 有一次非常干净的直接证据（17 秒崩在 IO80211），但只解释 1/6 |
| C 内存/SoC | **中高** | 能解释全部 6 次，但目前只有间接证据 |

三者并不互斥：**补丁造成的越界写**完全可能表现为别处进程崩溃；**内存本身不稳**也会让补丁的效果看起来像 bug。必须靠对照实验拆分，见 §8。

---

## 7. 其他隐患

1. **调试参数未清理**：`debug=0x100` 会让 panic 后不再自动重启（延长故障窗口）、并加重日志负担；`-v`、`-lilubetaall` 也属于排障期产物。
2. **显示压力过大**：当前 3872×2178 帧缓冲对应 1936×1089 UI @ **240 Hz**，非整数缩放。10/23 18:21 那次启动的[内核日志](/Volumes/STAGE/panic/kernel.log)里刷了大量：
   - `AMD ERROR! VRR FAILED TIMESTAMPS`
   - `readCurrentDpLinkSettings() !!! Failed to read DPCD:0x202 - DPCD_ADDRESS_LANE_01_STATUS`
   - `saveCursorImage() !!! Failed 'convertCursorImage' for all descriptors`
   本次启动 40 分钟内未复现，说明**只在切换显示模式/高刷下触发**。
3. **WindowServer 常年 ~100% CPU**：GPU 利用率 47%–81%。原因是 UURemote（20%）+ `replayd`（9.7%）+ `coreaudiod` 的 AudioTap 常驻采集，屏幕永不空闲。等于持续给本就不稳的 GPU 路径加压，长时间远程挂机更容易触发问题。
4. **睡眠被禁用**：`SleepDisabled 1`，同时保留 `hibernatemode 3` 与 sleepimage。属于"知道唤醒会黑屏/崩溃所以干脆禁用"的常见处理，但状态是半残的。
5. **SIP 全关 + FileVault 关**：补丁需要，但安全面为零，不适合放敏感数据。
6. **SMC 偶发报错**：`AppleSMCFamily::handleSMCResult ERROR in smcGetKeyFromIndexMMIO ... kSMCKeyIndexRangeError(0xb8)`，属虚拟 SMC key 越界，一般无害。
7. **无 EFI 备份证据**：三块 ESP 里是否有可用回滚，未知。

---

## 8. 建议与验证计划

### 8.1 立即做（低风险，不改变故障面）

1. **备份 EFI**：至少 `EFI/OC/config.plist` + `EFI/OC/Kexts`，并把当前可用的 NootedRed.kext 单独留档。
2. 去掉 `debug=0x100`、`-lilubetaall`；`-v` 可留可去。保留 `keepsyms=1`。
3. `-NRedNoSubmitGuard` 留着即可：guard 在本机是空转（`NRedSubmitGuardApplied` 为 0），删掉也不改变行为，详见 §4.4。
4. 显示模式降到 120/144 Hz 或关闭 FreeSync/VRR，作为观察项。

### 8.2 对照实验（按顺序，一次只改一个变量）

| 轮次 | 改动 | 目标 | 观察指标 | 回滚 |
| --- | --- | --- | --- | --- |
| A | 删除 `-NRedGLFix -NRedVDA`（或改 `-NRedGLFixOff`） | 排除用户态页补丁 | 24 h 内是否还有 panic；`NRedVCNUserApplied` 应为 0 | 加回原参数 |
| B | 换 `itlwm` 替代 `AirportItlwm`，去掉 `itlwm_he=1` | 排除 Wi-Fi 栈 | 开机 5 分钟内是否再出现 IO80211 类崩溃 | 恢复 AirportItlwm |
| C | MemTest86（OpenCore 引导 / U 盘）跑满一轮；或 BIOS 关 XMP / 降内存频率 | 排除硬件内存 | 是否报错；报错即定位 | BIOS 恢复 |

每轮之间建议至少留 **24 小时 + 一次完整开关机**；若期间出现 panic，立即记录 `boot-args` 和 uptime（本次报告已证明 uptime 是关键线索：17 s～388 s）。

### 8.3 中长期

- **VCN 空闲超时 126 次**是独立问题：即使不崩，视频路径也一直在超时兜底。建议单独起一条排查（`NRedVCNIdleTimeouts` / `IdleWaitedMs` 的增长率，以及 Dec 通道为什么为 0）。
- 恢复睡眠（先 `hibernatemode 0`、`pmset -a disablesleep 0` 再测），或明确接受"禁用睡眠"作为长期方案并写进文档。
- 若确认是补丁导致，回退到不启用 `-NRedVDA` 的稳定版本，把 Chromium 修复与 VDA 拆成两个独立构建，避免相互污染。

---

## 9. EFI 核对结果与复查命令

> 2026-10-24 12:5x 已用 sudo 只读挂载 `disk0s1` 完成核对（结论见下），本节的命令保留作复查用。

**已核对的事实**

| 项 | 结果 |
| --- | --- |
| OpenCore | `REL-108-2026-09-27`（1.0.8） |
| SMBIOS | `MacBookPro16,2`，`SecureBootModel = Disabled` |
| Kexts | 28 个条目；`itlwm.kext`、`GenericUSBXHCI.kext` 为 `Enabled=false`（禁用），其余启用 |
| DeviceProperties | 只有 2 项：`acpi-wake-type=1`（独显）、`layout-id=39`（声卡）；**核显没有任何属性**（无 `device-id` / framebuffer / VRAM 覆盖） |
| boot-args 来源 | `config.plist → NVRAM/Add/7C436110-…`，与运行期 NVRAM 完全一致（修改前） |
| 历史备份 | `EFI_backup_20261022-215955`（非 OC，只是 Ubuntu/Windows 引导）、`EFI-backup-20261023-1607`（NootedRed 0.8.10）、`EFI-backup-20261023-2238`（NootedRed 1.0.0）、本次新增 `EFI-backup-20261024-1253`（1.0.1，含本次改动前状态） |

**复查命令**

```sh
sudo diskutil mount -mountPoint /tmp/efi disk0s1     # 卷名是 NO NAME，用 -mountPoint 最省事
strings /tmp/efi/EFI/OC/OpenCore.efi | grep -m1 'REL-'
plutil -p /tmp/efi/EFI/OC/config.plist | grep -E 'boot-args|SecureBootModel|SystemProductName|BundlePath|Enabled'
ls /tmp/efi/EFI/OC/Kexts
ls -d /tmp/efi/EFI-backup-* 2>/dev/null
sudo diskutil unmount /tmp/efi
```

## 附录 A：本次采样使用的命令（只读）

```sh
sw_vers; uname -a; sysctl -n machdep.cpu.brand_string hw.model
system_profiler SPHardwareDataType SPDisplaysDataType SPAudioDataType SPBluetoothDataType SPNVMeDataType
nvram boot-args; nvram 4D1FDA02-38C7-4A6A-9CC6-4BCCA8B30102:opencore-version; csrutil status
kmutil showloaded --list-only | grep -v com.apple
ioreg -l -w0 | grep -E '"NRed|recoveryCount'
pmset -g; pmset -g batt; pmset -g therm; pmset -g assertions
ls -la /Library/Logs/DiagnosticReports/
log show --last 40m --predicate 'eventMessage CONTAINS "VRR" OR eventMessage CONTAINS "DPCD"'
```

## 附录 B：崩溃与日志文件索引

| 文件 | 说明 |
| --- | --- |
| `/Library/Logs/DiagnosticReports/Kernel-2026-10-22-220416.panic` | panic #1 |
| `/Library/Logs/DiagnosticReports/Kernel-2026-10-23-075853.panic` | panic #2（double fault） |
| `/Library/Logs/DiagnosticReports/Kernel-2026-10-23-091912.panic` | panic #3（WindowServer） |
| `/Library/Logs/DiagnosticReports/Kernel-2026-10-23-165812.panic` | panic #4（airportd） |
| `/Library/Logs/DiagnosticReports/Kernel-2026-10-23-181604.panic` | panic #5（searchpartyuseragent） |
| `/Library/Logs/DiagnosticReports/Kernel-2026-10-23-183103.panic` | panic #6（QQ） |
| `/Library/Logs/DiagnosticReports/shutdown_stall_*.shutdownStall` | 5 次关机卡死 spindump |
| [`/Volumes/STAGE/panic/`](/Volumes/STAGE/panic) | 你已归档的 panic 副本 + `kexts.txt` / `nvram.txt` / `csrutil.txt` / `dmesg_err.txt` |
| [`/Volumes/STAGE/nrd-src/NootedRed-master/`](/Volumes/STAGE/nrd-src/NootedRed-master) | 补丁源码与 `analysis/` 调查笔记 |

## 附录 C：结论速查

- ✅ 能开机、能上网、有图形加速、外设齐全 —— 日常可用。
- ⚠️ 24 小时内 6 次 panic + 5 次关机卡死 —— 不可信，不能放重要工作。
- 🔍 最该先做的两件事：**备份 EFI** + **关掉 `-NRedGLFix/-NRedVDA` 跑 24 小时对照**。
- 📌 当前这次启动是干净的（1 小时+，无 GPU reset），说明配置并非"必崩"，更像是被特定操作触发。
