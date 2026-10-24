# Firebat T5（R5-5600H）黑苹果 macOS 15 实操报告

**日期**：2026-10-22 ～ 2026-10-24
**机器**：Firebat T5 / 同方 GM5ZG0O-F
**结果**：macOS 15.8.1（24H32）可用，核显加速正常，Wi-Fi 正常，连续稳定运行 1 小时以上
**崩溃根因**：NootedRed fork 1.0.0 的 user-space Metal SubmitGuard 补丁

---

## 0. 结论摘要

| 项目 | 结果 |
|---|---|
| 系统 | macOS 15.6.1 (24G90) → 在线增量升级 → **15.8.1 (24H32)** |
| 引导 | OpenCore 1.0.8 RELEASE，写入内置 NVMe 的 ESP（disk1 p1） |
| 核显 | Vega 7 (1002:1638) 加速可用，2K 240Hz 输出 |
| Wi-Fi | Intel AX200，laobamac AirportItlwm 2.4.0 **Sequoia 构建** |
| 稳定性 | 稳定版连续运行 1 小时以上无 panic（此前最好 48 分钟） |
| 掉帧 | 启动台左右切页掉帧，尚未解决（疑填充率/软件光标） |
| 遗留 | 硬件光标失效、eDP DPCD 读取失败、Android USB 网络共享不可用 |

**最关键的一条**：此前 6 次 panic 全部是"内核堆内存被写坏"，崩在 Apple 自己的代码里，没有任何第三方 kext 出现在调用栈。真正的原因直到从 NootedRed 二进制里提取出 `NRedSubmitGuard` 这组符号、再用 `-NRedNoSubmitGuard` 关掉它之后才被确认。

---

## 1. 硬件与环境

| 部件 | 型号 / 参数 |
|---|---|
| 机身 | Firebat T5，主板同方 GM5ZG0O-F |
| CPU | AMD Ryzen 5 5600H（Cezanne，6 核 12 线程） |
| 核显 | AMD Radeon Vega 7，`1002:1638`，ACPI 路径 `\_SB.PCI0.GP17.IGPU`（00:08.1） |
| 独显 | NVIDIA RTX 3060 Mobile，`10DE:2520`（00:01.0），**ACPI 层禁用** |
| 屏幕 | BOE0A55，2560×1440 eDP，240Hz |
| 无线 | Intel AX200，`8086:2723` |
| 有线 | Realtek RTL8125 |
| 声卡 | Realtek ALC274，`layout-id 39` |
| 内存 | 16 GB DDR4-3200 |
| 硬盘 | FIREBAT FET34-512G，476.9 GB NVMe |

**磁盘布局**（不可动的部分）

```text
disk0            Windows 系统盘 —— 全程未触碰
disk1 p1  1 GB   ESP（EFI 系统分区）—— 引导文件放这里
disk1 p2  16 MB  MSR —— 未触碰
disk1 p3+        macOS APFS 卷
```

启动方式：每次开机按 **F7** 选内置硬盘 → 进 OpenCore 菜单 → 选 macOS。

---

## 2. 起点：原始 EFI 的状况

拿到的 `D:\ubac\20240314-OC-EFI项目\ESP4分区 - 副本\EFI\` 是一份 28 kext 的"全家桶"，存在这些问题：

| 问题 | 说明 |
|---|---|
| `DummyPowerManagement` 放错位置 | 放在 `Kernel → Quirks`，OC 1.0.8 要求放在 `Kernel → Emulate`，ocvalidate 直接报 schema 错误 |
| 多处中英文混排的 `Comment` | OpenCore 禁止 `Comment` 含非 ASCII 字符，会导致校验失败 |
| 残留 `ApfsDriverLoader.efi` | OC ≥ 0.6.6 自带 APFS 支持，该驱动多余且可能冲突 |
| 内核补丁未按 6 核适配 | `Force cpuid_cores_per_package` 需要按物理核心数填 |
| 全部 28 个 kext 一起启用 | 无法定位任何问题 |

处理方式：不修改原 EFI，另建一套按需组合的变体目录，逐个验证。

---

## 3. 安装 macOS 15.6.1

| 步骤 | 做法 |
|---|---|
| 镜像 | `[ShiPinGe]macOS 15.6.1 Sequoia(24G90).dmg` |
| 写盘 | balenaEtcher 写入 U 盘 |
| 引导 | 把调好的 OpenCore 放进**内置 ESP**，U 盘只负责安装；开机 F7 选内置硬盘 |
| 安装 | 装到 disk1 的 APFS 分区，**不抹掉 Windows 盘** |
| 后续 | 离线 `startosinstall` 方案放弃，改用在线增量更新 |

成功后的第一次启动即进入桌面，说明 ACPI、内核补丁、SMBIOS 这一层是对的。

---

## 4. 功能补齐（15.6.1 阶段）

按需求逐项推进，每次只动一个变量：

| 需求 | 措施 | 结果 |
|---|---|---|
| 扬声器 | `AppleALC` + `layout-id 39`，后续换 `SimpleHDA` | 正常 |
| Wi-Fi | `itlwm` + HeliPort → 改 `AirportItlwm`（原生界面） | 正常 |
| 电池百分比 | `SMCBatteryManager` + `ECEnabler` | 正常 |
| 有线网卡 | `RTL812xLucy` | 正常 |
| 蓝牙 | `IntelBluetoothFirmware` + `IntelBTPatcher` + `BlueToolFixup` + `AMFIPass` | 可用，但与 AMFIPass 组合存在风险 |
| USB 端口 | `USBToolBox` + 端口表 | 部分可用 |
| 亮度键 | `BrightnessKeys` + `SSDT-PNLF` | 正常 |
| 时钟同步 | `ForgedInvariant` 1.5.0 | 正常 |
| 电源管理 | `AMDRyzenCPUPowerManagement` 0.7.2 + `SMCAMDProcessor` 1.0.1 | **锁屏输入密码一半卡死** |
| 核显加速 | NootedRed fork 1.0.0（weachy） | 能点亮 2K 240Hz，但不定期 panic |

这一阶段最容易被误判的一点：**"能用"不等于"稳定"**。11 kext 的精简组合在 15.6.1 上曾被认为"成功、不死机"，事后日志证明它只是撑了 37 分钟才 panic。

---

## 5. 崩溃排查（本项目的核心工作）

### 5.1 数据来源

| 来源 | 内容 |
|---|---|
| `/Library/Logs/DiagnosticReports/*.panic` | 6 份 panic 报告（JSON type 210） |
| `log show --predicate 'process == "kernel"'` | 4 小时内核日志，278,085 行 |
| `kextstat` | 实际加载的 31 个非苹果 kext |
| `csrutil status` / `nvram -p` | SIP 与 NVRAM 状态 |

抓取方式：把日志复制到 exFAT 共享分区，在 Windows 侧分析。

### 5.2 六次 panic 全表

| 时间 | 系统 | panic 类型 | 崩溃任务 |
|---|---|---|---|
| 10-22 22:04 | 15.6.1 | page fault @ `IOSurface` | DisplaysExt |
| 10-23 07:58 | 15.6.1 | **Double fault** | kernel_task |
| 10-23 09:19 | 15.6.1 | `[Bank ledger]: element modified after free` | WindowServer |
| 10-23 16:58 | 15.6.1 | page fault @ `IO80211Family` | airportd |
| 10-23 18:16 | 15.8.1 | page fault | searchpartyuseragent |
| 10-23 18:31 | 15.8.1 | `[data.kalloc.16384]: element modified after free` | QQ |

**6 次中有 4 次发生在 15.6.1**，这直接排除了"升级到 15.8.1 才出问题"的假设。

### 5.3 崩溃类型说明了什么

```text
[Bank ledger]: element modified after free (off:56, val:0x2400, sz:120)
[data.kalloc.16384]: element modified after free (off:15944, ...)
Double fault at 0xfffff69b00002847
Kernel trap ... type = 14=page fault
```

三种特征同时出现：

1. **释放后写入**（element modified after free）出现在两个完全不同的内存区域（120 字节的 Bank ledger 和 16 KB 的 data.kalloc）
2. **Double fault**（栈被破坏）
3. **随机 page fault**，落在 IOSurface / IO80211Family / APFS 解压 / DNS 进程四个互不相关的子系统

而且**六份报告的调用栈里没有任何第三方 kext**。

结论：某个驱动向自己缓冲区之外写数据，破坏了别处的内存；崩溃发生在"受害者"身上，所以栈里看不到元凶。这类问题无法通过调用栈定位，只能靠二分法或逐个关驱动。

### 5.4 显卡侧的直接证据

从 `kernel.log` 提取，每次加载 AMD 驱动时都会出现：

```text
[AMDRadeonX6000Framebuffer] [6:0:0] [VRAM] mapMemorySpace() !!! Failed to get descriptor for ROM Base.
[AMDRadeonX5000HWLibs] Error SW_IP_CLIENT_ID__SMU: event_id=0xc0201 ... pData=0 data_size=0
[AMDRadeonX5000HWLibs] Invalid size or CPU virtual address size = 0x0, memblock_ptr = 0x0
[AMDRadeonX6000Framebuffer] dcn10_verify_allow_pstate_change_high:1217 BREAK_TO_DEBUGGER point !!!
[AMDRadeonX6000Framebuffer] [CUR:0] saveCursorImage() !!! Failed 'convertCursorImage' for all descriptors.
[AMDRadeonX6000Framebuffer] [DAL] readCurrentDpLinkSettings() !!! Failed to read DPCD:0x202
```

旁证：在不启用 NootedRed 的那次启动中，内核日志里 `AMDRadeon` 相关行数为 **0**——AMD 驱动完全没加载，画面来自固件帧缓冲。这也解释了为什么关掉核显驱动后系统能跑但很卡。

### 5.5 二分矩阵（实测记录）

| 配置 | kext 数 | 结果 |
|---|---|---|
| 精简集（Lilu / NootedRed / VirtualSMC / MCE禁用 / VoodooPS2×2 / VoodooI2C×4） | 11 | 2K 240Hz 正常，**37 分钟后 panic** |
| + `RestrictEvents` + `revpatch=sbvmm` | 12 | 5～48 分钟区间内 panic |
| + `AMDRyzenCPUPowerManagement` 0.7.2 + `SMCAMDProcessor` 1.0.1 | 13 | **锁屏输入密码一半卡死** |
| + `ForgedInvariant` + `GenericUSBXHCI` + 上一组 | 15 | 约 5 分钟自动重启 |
| 全量 | 28 | 锁屏卡死（有画面） |
| 群友配置（含 CPUPM 0.9.1、ForgedInvariant、AMFIPass+蓝牙三件套、PS2 触控板） | 32 | 9～17 分钟重启/卡死 |
| 官方 NootedRed 0.8.10 | — | 跑码结束后立即重启，进不到 logo |
| **NootedRed 关闭** | 31 | 无 AMD 驱动，完成约 20 分钟的 OTA 安装并进入系统 |
| **NootedRed 开 + `-NRedNoSubmitGuard`** | 32 | **连续稳定 1 小时以上** |

### 5.6 定位过程

关键转折点是**直接读取 kext 二进制的字符串**，而不是继续猜测：

```text
-NRedNoSubmitGuard
@ Enabling user-space Metal SubmitGuard (free tail filler)
NRedSubmitGuardApplied
NRedSubmitGuardWindowServerPID
NRedSubmitGuardWindowServerApplied
@ Failed to resolve kernel symbols for SubmitGuard
```

这说明 NootedRed 内部有一个叫 **Metal SubmitGuard** 的补丁：在用户态 Metal 提交路径上给命令缓冲尾部填空白（free tail filler），并且对 WindowServer 单独打一次。它的实现方式是"定位内核函数地址 → 改写指令"，而定位一旦失准，就会把数据写到错误的内核内存里——正是"element modified after free"的来源。

加 `-NRedNoSubmitGuard` 关闭该补丁后，机器连续稳定 1 小时以上，6 类 panic 全部消失。

---

## 6. 系统升级：15.6.1 → 15.8.1

### 6.1 更新查不到

| 尝试 | 结果 |
|---|---|
| 系统设置 → 软件更新 | "已是最新" |
| `softwareupdate --list --all` | "No new software available" |
| SIP 改 `0x0A43` | 仍然查不到 |
| SIP 改 `0x0FFF`（12 位全开） | 仍然查不到 |

`csrutil status` 的输出可以逐位对上 `0x0FFF`（Apple Internal 一项除外），确认 SIP 已按参数生效——**所以问题不在 SIP**。

### 6.2 真正的解法

`softwareupdated` 会校验机型与虚拟化状态，在非苹果固件上会判定"无可更新"。
解法（社区文档一致）：`RestrictEvents.kext` + `boot-args` 加 **`revpatch=sbvmm`**。

加上后立刻能查到 15.8.1，在线增量更新成功。

### 6.3 安装阶段必须做的事

升级过程的第二阶段在"安装器环境"里写系统卷，此时加载显卡加速驱动极易卡死（实测卡在"还剩三分钟"）。

**正确做法**：安装前把 `NootedRed.kext` 关掉，升级完再打开。

```text
EFI-OTA-NONR  →  NootedRed 关闭，其余保持
结果：屏幕全程可能黑屏，但安装器在后台完成
```

装完后系统即为 15.8.1（24H32），核显驱动未加载，画面来自固件帧缓冲。

---

## 7. Wi-Fi 驱动版本事故

群友提供的 EFI 里，Wi-Fi 驱动是**错的版本**：把 Tahoe（macOS 26）构建塞进了 15 系统。

用哈希比对确认：

| 来源 | 可执行大小 | SHA-256 前缀 | 判定 |
|---|---|---|---|
| laobamac 官方 Sequoia | 15,918,064 | `ec4c1fc8…` | **15 用** |
| laobamac 官方 Tahoe | 15,921,120 | `259bac94…` | 26 用 |
| 群友 EFI 内 | 15,921,120 | `259bac94…` | ❌ 是 Tahoe |
| 本地备份 | 15,918,064 | `ec4c1fc8…` | ✅ 是 Sequoia |

处理：换回 Sequoia 构建，并加内核范围约束，防止误加载。

```text
AirportItlwm.kext   MinKernel 24.0.0   MaxKernel 24.99.99
itlwm.kext          Enabled = false（避免与 AirportItlwm 抢设备）
```

**教训**：同名 kext、同版本号（都是 2.4.0）、体积只差 3 KB，光看版本号无法区分。必须比对可执行文件哈希。

---

## 8. 稳定基线的确立

### 8.1 群友 EFI 带来的有效改进

| 项 | 我们的版本 | 群友版本 | 判断 |
|---|---|---|---|
| CPU 电源管理 | 0.7.2 + `SMCAMDProcessor` 1.0.1 | **0.9.1，不带 SMCAMDProcessor** | 有效：不再锁屏卡死 |
| 声卡 | `AppleALC` | `AppleALC` + **`SimpleHDA` 1.0.0** | 有效 |
| APFS | OC 内置 | 额外 `apfs_aligned_15.5.efi` | 保留（未证伪） |
| 选择器 | 文字菜单 | `OpenCanopy` 图形界面 | 保留 |
| USB | 定制端口表 | `USBToolBox` + `UTBDefault` | 保留 |

### 8.2 最终稳定参数

```text
csr-active-config = FF0F0000 (0x0FFF)

boot-args:
  -v keepsyms=1 debug=0x100
  ipc_control_port_options=0
  -vi2c-force-polling
  -amfipassbeta -lilubetaall
  itlwm_he=1
  -NRedGLFix -NRedVDA -NRedVCNNoGfxOff
  -NRedNoSubmitGuard          ← 关键
```

### 8.3 NootedRed 参数含义（依据二进制字符串）

| 参数 | 作用 |
|---|---|
| `-NRedGLFix` | OpenGL 兼容层修复（可换 `-NRedGLFixNoResolve` 等档位） |
| `-NRedVDA` | 视频硬解（实验性，需 profile 认定为 VCN 2.x 才真正生效） |
| `-NRedVCNNoGfxOff` | VCN 工作时不让 GFX 进 GFXOFF 省电态 |
| `-NRedNoSubmitGuard` | **关闭 Metal SubmitGuard 补丁** |

未启用的相关参数：`-NRedBeta`、`-NRedNoAccel`、`-NRedOff`、`-NRedForceVDA`、`-NRedDPDelay`、`-NRedCursorDebug`、`-NRedDebugUltra`、`-NRedDelayPanic`。

### 8.4 升级到 NootedRed 1.0.1

作者随后发布了 1.0.1，改动集中在 `NootedRed/UserPatches.cpp`：

```text
efe90c3e  chore: bump version to 1.0.1
986e69d0  UserPatches: eliminate unsafe single-site fallback and add Gate 0 vnode filter
f44dd3ae  UserPatches: implement Triple-Gate resolver hardening and fix Sequoia allowOffset
```

二进制中出现新字符串：

```text
@ Resolver matched and patched Metal Gate (0x%zX) and Allow (0x%zX) atomically
_vnode_getname / _vnode_putname
-NRedSubmitGuard  -NRedLegacyBlend  -NRedNoLegacyBlend  -NRedVCNNoDrain
```

"atomically" 是关键：1.0.0 先定位再打补丁，中间状态会错位；1.0.1 改为原子地同时打 Gate 与 Allow 两处，并增加 vnode 名校验。

已写入 ESP 的版本：`EFI-STABLE-1.0.1`（先保留 `-NRedNoSubmitGuard`，只换版本不叠加变量）。

---

## 9. 当前系统状态

### 可用

- 核显加速，2560×1440 高刷新输出
- Wi-Fi（AX200，原生界面）、有线（RTL8125）
- 声音、麦克风
- 电池百分比、亮度键
- 蓝牙
- 触摸板（I2C）、键盘
- 睡眠唤醒未做长测

### 不可用 / 有缺陷

| 项 | 现象 | 说明 |
|---|---|---|
| 硬件光标 | `convertCursorImage` 全部失败，退化为软件光标 | NootedRed 的 `setupCursors` 补丁在 15.8.1 上应用失败 |
| eDP 链路读取 | `Failed to read DPCD:0x202` | 每次启动都报 |
| 启动台切页 | 掉帧 | 疑填充率不足或软件光标重绘 |
| Android USB 网络共享 | 不可用 | 需 HoRNDIS，Sequoia 无可用版本 |
| iPhone USB 网络共享 | 未验证 | 取决于 USB 端口映射 |

---

## 10. 遗留问题与后续计划

### 10.1 掉帧（唯一影响日常使用的问题）

三个候选原因，可用零成本方式区分：

| 测试 | 判定 |
|---|---|
| 改 120Hz 后是否流畅 | 是 → eDP 链路带宽不足 |
| 鼠标静止流畅、一动就掉 | 是 → 软件光标重绘 |
| 降低透明度/降低动态效果后改善 | 是 → 填充率不足 |

作者已回应后续会修复光标问题。

### 10.2 NootedRed 1.0.1 的下一步验证

已备好 `EFI-STABLE-1.0.1-GUARD`：去掉 `-NRedNoSubmitGuard`，让 1.0.1 修好的守卫真正生效。

| 结果 | 含义 |
|---|---|
| 守卫开着也稳定 | 1.0.1 修复有效，可拿回守卫的渲染收益 |
| 守卫开着仍崩 | 修复未触及根本，继续用 `-NRedNoSubmitGuard` |

---

## 11. 工程资产清单

### 11.1 脚本（`_out\`）

| 脚本 | 行数 | 作用 |
|---|---|---|
| `写EFI到内置ESP.ps1` | 143 | 自动提权 → 识别内置 NVMe → 备份现有 EFI（保留最新 2 份，按写入时间排序）→ 空间预检 → 拷贝 → 校验关键文件 |
| `修复ESP空间.ps1` | 92 | ESP 写坏后的修复：清理旧备份、重命名残缺 EFI、重建 |
| `检查ESP.ps1` | 58 | 只读体检：剩余空间、备份列表、EFI 完整性、csr、boot-args |
| `替换U盘EFI.ps1` | 131 | 写 U 盘 ESP |

写入用法：

```powershell
powershell -ExecutionPolicy Bypass -File "写EFI到内置ESP.ps1" -Source STABLE-1.0.1
```

### 11.2 EFI 变体（52 个目录）

按用途分组：

```text
实验/排查     SAFE WIFI FULL-TEST BISECT-A BISECT-B BISECT-PWR BISECT-USBTSC
精简集        MIN MIN-Air MIN-Air-SIPFFF MIN-Air-OTA MIN-Air-OTA-All MIN-Plus
群友配置      FRIEND FRIEND-NONR FRIEND-NOSG
稳定基线      STABLE-15.8.1  STABLE-1.0.1  STABLE-1.0.1-GUARD
              STABLE-GLFIX-NORES  STABLE-NO-GFXHOLD
安装/升级     INSTALL INSTALL-DP OTA-NONR
```

### 11.3 关键文件哈希

| 文件 | 大小 | SHA-256 前缀 |
|---|---|---|
| `NootedRed-1.0.1-RELEASE.zip` | 1,113,403 | `26c4f4d22b78…` |
| NootedRed 1.0.1 可执行 | 2,759,008 | `04b3d31d30c6…` |
| NootedRed 1.0.0 可执行 | 2,738,000 | — |
| `NootedRed-1.0.0-RELEASE.zip` | 1,425,976 | `86d0be68821e…` |
| AirportItlwm 2.4.0 Sequoia | 15,918,064 | `ec4c1fc83c06…` |
| AirportItlwm 2.4.0 Tahoe | 15,921,120 | `259bac9432bf…` |
| `EFI-t5a-5600H-ley (1).zip` | 54,575,911 | `7cedb2d24709…` |

---

## 12. 经验与教训

### 12.1 OpenCore 层

1. **`DummyPowerManagement` 在 OC 1.0.8 属于 `Kernel → Emulate`**，不是 `Kernel → Quirks`；放错会直接被 ocvalidate 判 schema 错误。
2. **`Comment` 字段只能写 ASCII**，中文会导致 `contains illegal character`。
3. **每次改完必须跑 ocvalidate**，目标是 `No issues found`。
4. **OC ≥ 0.6.6 自带 APFS**，不要再放 `ApfsDriverLoader.efi`。

### 12.2 ESP 空间管理（踩过大坑）

- 1 GB 的 ESP 放 55 MB 的 EFI，**备份会迅速吃满**。曾出现剩余 5.5 MB、只拷进 67/186 个文件、导致完全无法引导的情况。
- 现在写入脚本会：只保留最新 2 份备份、拷贝前预检"EFI 大小 + 40 MB"的空间。
- **备份目录名的时间戳会跳变 8 小时**（macOS 按 UTC 写 RTC，回到 Windows 后时钟偏移），所以清理备份要**按写入时间排序，不能按目录名排序**。

### 12.3 排查方法层

1. **"能用"不等于"稳定"**：11 kext 的精简配置看起来成功，实际 37 分钟后 panic。判断稳定性必须看日志里的 panic 记录，不能靠主观感受。
2. **panic 栈里没有第三方 kext，不代表第三方 kext 无辜**：野写型内存破坏的崩溃点永远在受害者身上。
3. **从 `kernel.log` 里提取每次启动的 boot-args**，可以把 panic 精确对应到当时的配置，这是本项目最有效的一步。
4. **读 kext 二进制字符串**能直接发现作者未文档化的开关，比搜索文档快。

### 12.4 升级流程

1. **黑苹果查不到更新，第一嫌疑是 `softwareupdated` 的机型/虚拟化校验，不是 SIP**。解法是 `RestrictEvents` + `revpatch=sbvmm`。
2. **安装阶段必须关掉显卡加速驱动**，否则会卡在"还剩三分钟"。
3. **升级完第一件事是确认版本号**，不要急着开驱动。

### 12.5 版本管理

同一个 kext 名字、同一个版本号，可能是不同系统分支的构建（Sequoia vs Tahoe）。**必须比对可执行文件哈希**，并用 `MinKernel` / `MaxKernel` 加约束。

---

## 附录 A：常用命令

抓崩溃日志（在 macOS 上执行，导出到 exFAT 共享分区）：

```bash
VOL=/Volumes/STAGE
[ -d "$VOL" ] || VOL=$(ls -d /Volumes/*/ 2>/dev/null | grep -viE 'Macintosh|Recovery|com.apple' | head -1)
sudo mkdir -p "$VOL/panic"
sudo cp /Library/Logs/DiagnosticReports/*.panic "$VOL/panic/" 2>/dev/null
sudo log show --last 4h --style compact --predicate 'process == "kernel"' > "$VOL/panic/kernel.log" 2>/dev/null
sudo kextstat | grep -v apple > "$VOL/panic/kexts.txt"
nvram -p | grep -iE 'csr-active|boot-args' > "$VOL/panic/nvram.txt"
```

校验 OpenCore 配置（Windows 侧）：

```powershell
ocvalidate.exe "D:\...\EFI\OC\config.plist"
# 期望输出：No issues found
```

从内核日志提取每次启动的参数：

```bash
grep -o 'Boot args:.*' /path/to/kernel.log | sort -u
```

---

## 附录 B：术语

| 词 | 含义 |
|---|---|
| ESP | EFI System Partition，存放引导文件的 FAT32 分区 |
| kext | macOS 内核扩展（驱动） |
| OC | OpenCore 引导器 |
| csr-active-config | SIP 开关，小端序数据，`FF0F0000` = 0x0FFF |
| VDA | Video Decode Acceleration，视频硬解 |
| VCN | Video Core Next，AMD 视频编解码引擎 |
| GFXOFF | GPU 空闲时进入的低功耗状态 |
| SubmitGuard | NootedRed 中给 Metal 命令缓冲尾部填空白的内核补丁 |
| panic | 内核崩溃，会生成 `.panic` 报告 |
| 野写 | 驱动向自己缓冲区之外写数据，破坏其他内核内存 |
