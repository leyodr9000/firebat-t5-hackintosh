# N 卡驱动（NullMoth NVIDIA 2.5.3）与 EFI 方案评估

**对象**：Firebat T5 / 同方 GM5ZG0O-F · R5-5600H + RTX 3060 Laptop（`10DE:2520`）
**评估时间**：2026-10-24
**结论先行**：驱动是真实存在的、Ampere 有代码路径，**技术上不是空想**；但现有的 `EFI-独显直连` 方案里有 **2 处必然导致失败的配置**，按现状执行只会在切 MUX 之后黑屏。值得做，但必须"隔离环境 + 分步验证"。

---

## 1. 驱动本体

包：`_downloads/1401/1401/NullMoth/nullmoth-nvidia-2.5.3.tar.gz`（252 MB，120 个条目）

| 项 | 内容 |
| --- | --- |
| 版本 | 2.5.3（上一版 2.5.2，`BASELINE-IDENTITY.json` 带两版 SHA-256） |
| 内核态 | `NVRM.kext`、`NVRMFB.kext`、`NVAccel.kext`、`NVRMAGDC.kext` → 装到 `/Library/Extensions` |
| 用户态 | `NVMTLDriver.bundle`（Metal）、`NVIDIAShared.bundle`、`nvmtl`（`libvulkan.1.4.362` + `nvk_icd.json`）→ `/Library/GPUBundles` |
| 编解码 | `libnvdec_h264/hevc`、`libnvenc_h264/hevc`（NVDEC/NVENC 用户态库） |
| 驱动分支 | `libnvidia-ptxjitcompiler.so.610.57.04` → NVIDIA **610** 系 |
| 匹配方式 | `IOPCIPrimaryMatch = 0x000010de&0x0000ffff` → **所有 N 卡都匹配**，内部按架构挑 |
| 系统要求 | `runtime_minimum_macos: 15.5`；安装器明示 **只装 macOS 15** |
| 安装方式 | `install.sh` 拷 kext → 重建 `AuxiliaryKernelExtensions.kc`（`--allow-missing-kdk`），带安装锁、崩溃回滚、`uninstall.sh` |

**资格测试（package 自带声明）**：macOS 15.8.1 + **RTX 5060** + 2 显示器；`guarded_boot: passed`；刷新率切换只验证了 **60 / 75 / 100 Hz**；**已知问题**：改一块屏的刷新率后，另一块屏可能出现黑帧直到重启。

**参数消费者已核实**（这点很关键，排除了"boot-args 是摆设"的怀疑）：

| 二进制 | `nvfb` | `nvaccel` | `nvkms` | `nvfbheads` |
| --- | --- | --- | --- | --- |
| NVRM | 8 | 5 | 4 | 3 |
| NVRMFB | 12 | 0 | 3 | 0 |
| NVAccel | 0 | 79 | 1 | 0 |

```text
NVRM-fb: fb%u NVKMS mode %u: %ux%u @%u Hz pclk %u valid %u preferred %u
```

即显示链路走 **NVKMS**（把 NVIDIA 的开源内核显示模块搬过来），模式可以是任意 mHz，不写死 60Hz。

**Ampere 支持痕迹**：`NVRM` 二进制里 `GA106` 出现 **64 次**、`Ampere` **37 次**；但**没有 `2520` 这种 device id 字符串**，也没有 `laptop/mobile` 字样 —— 说明"支持"是按**架构**判断的，**笔记本变体、MUX 机型没有出现在资格测试里**。

---

## 2. 你这台机器的匹配度

| 维度 | 现状 | 判断 |
| --- | --- | --- |
| GPU | RTX 3060 Laptop（GA106/GA107），ACPI `\_SB.PCI0.GPP0.PEGP` | 架构在驱动代码路径内，但机上无验证样本 |
| 内屏 | BOE 2560×1440 **@240Hz** eDP | 驱动只验过 100Hz；240Hz 时序/带宽**完全未知** |
| 双 GPU | Vega7 iGPU + 3060，MUX 切换 | 方案要求 MUX 固定独显，且屏蔽 `IONDRVSupport` |
| 参考先例 | 同平台蛟龙5 的文章写着 "dGPU ❌ 已屏蔽" | 同平台**没有**公开的成功案例 |
| 系统版本 | 15.8.1（≥15.5 ✓） | 但升级到 macOS 26 后驱动直接失效 |

---

## 3. 现有 EFI 方案逐项核验

目录：`EFI升级_20261022/EFI-独显直连`（两份 config：`config.plist` 混合阶段 / `config-仅独显.plist` 最终态）

| 项 | 现状 | 判断 |
| --- | --- | --- |
| SMBIOS `iMacPro1,1` | 无核显的桌面机型 | ✅ 合理（避免 iGPU/Metal 抢占） |
| `SecureBootModel=Disabled` | 已设 | ✅ 必要 |
| 屏蔽 `com.apple.iokit.IONDRVSupport` | Patch 已启用，注释 "NVRMFB must own display index 0" | ⚠️ 必须，但**失败时连兜底都没有** |
| `nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth` | 在 boot-args 里 | ✅ 参数确实被驱动消费 |
| **`SSDT-AMD-PGPU-DISABLE.aml`** | **两份 config 都 `Enabled=true`** | ❌ **硬冲突**：驱动要 attach 的 PEGP 被 _PS3 下电/屏蔽 |
| `AMFIPass.kext` + `amfi=0x80 amfi_get_out_of_my_way=0x1` | 同时存在 | ❌ **冗余/冲突**（你们 10/23 的笔记自己记过：同时用会 CS_RUNTIME 死循环 + 黑屏） |
| 混合阶段同时开 NootedRed + NVRM | `config.plist` 里两者都启用 | ⚠️ 双 GPU 驱动共存，未经验证 |

### 必须先修的两处

1. **`config-仅独显.plist` 关掉 `SSDT-AMD-PGPU-DISABLE.aml`**（`ACPI/Add` 里那条改 `Enabled=false`）。不修的话，"仅独显"配置自己把要驱动的卡屏蔽了，逻辑上不可能成功。
   - `config.plist`（混合阶段）是否也要关，取决于你想不想在核显模式下让 NVRM 摸到卡；保守做法是**安装阶段保持屏蔽**（避免两个驱动打架），**切 MUX 前的那次重启再关**。
2. **AMFIPass 与 amfi 参数二选一**：保留 `AMFIPass.kext` 就删掉 `amfi=0x80 amfi_get_out_of_my_way=0x1`；或者反过来。

---

## 4. 按现状执行会怎样

| 阶段 | 操作 | 预期结果 |
| --- | --- | --- |
| 1 | MUX 核显 + `config.plist`（NootedRed + NVRM） | 大概率能进系统（核显点亮），但 NVRM 在"被 SSDT 下电"的卡上 attach 失败，驱动等于没装上；且 AMFIPass/amfi 组合可能开局就黑屏 |
| 2 | 装驱动 → 切 MUX 独显 → `config-仅独显` | PEGP 仍被屏蔽 → NVRM 拿不到 framebuffer；而 `IONDRVSupport` 被禁、iGPU 又被 MUX 物理断开 → **全黑，无任何兜底** |

恢复路径：回 Windows（或 BIOS）把 MUX 切回混合/核显，机器就能再亮；这步成本低，但**必须提前确认你能在"黑屏状态下"进 Windows**（盲操作或依靠外接屏）。

---

## 5. 风险矩阵

| 风险 | 概率 | 影响 | 恢复成本 |
| --- | --- | --- | --- |
| 切 MUX 后黑屏 | 高 | 中（可切回） | 低：Windows/BIOS 切 MUX |
| AuxKC 重建后无法启动 | 中 | 高 | 中：恢复模式重建 / 还原备份 |
| 系统升级后驱动失效 | 必然 | 中 | 中：等驱动更新或留在 15.x |
| 240Hz / 2K 不支持或闪烁 | 中高 | 中 | 低：降到 60/120Hz 或回核显 |
| 笔记本功耗/风扇失控 | 中 | 中 | 低：回核显 |
| 与 NootedRed 内核态冲突 | 未知 | 高 | 低：只用其一 |

---

## 6. 建议路线（若决定做）

**隔离，不覆盖日常 ESP。**

1. 准备 U 盘 EFI（`EFI-独显直连` 修完上面两处后整体拷入），内置 ESP 保持现在的核显方案不动。
2. 先做**只读验证**：MUX 仍在核显，用 U 盘启动，确认 **dGPU 已被枚举**：
   ```sh
   ioreg -l -w0 | grep -c de10         # 期望 > 0
   system_profiler SPDisplaysDataType   # 看有没有多出来的 GPU
   ```
   - 注意：装驱动之前 `ioreg -c NVRM` 必然为空（kext 还没装），不能拿它当判据。
   - 若 `de10` 计数为 0 → 说明 SSDT 屏蔽仍在起作用或 PCI 枚举没通，先解决这一步，**不要**装驱动、更不要切 MUX。
3. 只有第 2 步成功，才切 MUX 到独显，并换成关掉 PGPU-DISABLE 的 `config-仅独显`。
4. 判据：WindowServer 能起、`SPDisplaysDataType` 显示 NVIDIA 且 Metal 正常、无 panic、240Hz 能选上（这就是未知项）。
5. 回滚：`uninstall.sh` + 恢复 ESP + Windows 侧切 MUX。

**如果目的只是"240Hz 更顺"**：先算这笔账 —— 核显侧今天已经把 HiDPI 从 3872×2178 降到 3392×1908（§4.7），成本 0；再往下还有 120Hz、或退回 1696×954 以下。N 卡路线即便成功，240Hz 是否支持、MetaI 性能如何、功耗风扇如何，全是未知数，而且只有 2.5.3 一个版本、社区样本极少。

---

## 7. 一句话判断

**能做，但现在这套 EFI 是"自相矛盾"状态**（屏蔽要驱动的卡 + AMFI 配置互斥）；修好之后值得在 U 盘上试一次，用"先核显验证 attach、再切 MUX"的方式把黑屏风险压到可恢复的范围。**不建议**在没验证的情况下直接把它写进日常 ESP。

---

## 附：本次评估的证据来源

- `_downloads/1401/1401/NullMoth/nullmoth-nvidia-2.5.3.tar.gz`（版本、MANIFEST、BASELINE-IDENTITY、install.sh/uninstall.sh、kext 二进制字符串）
- `EFI-独显直连/OC/config.plist`、`config-仅独显.plist`（ACPI 启用状态、Kernel/Patch、boot-args、AMFIPass）
- `升级报告.md`（第 113/120/127/129/192 行：独显直连设计意图与"macOS 下无画面属预期"）
- `_backup/20261023-0115-before-macos26/状态说明.md`（ACPI 路径、HDMI 音频 `\_SB.PCI0.GPP0.NHDA`、AMFIPass 与 amfi 参数冲突记录）
- 本机：`ls /System/Library/Extensions | grep -ic nvidia` → **0**（当前系统未装任何 N 卡驱动）


---

## 8. 实测结果（2026-10-24 下午，实机）

### 8.1 混合模式：驱动完全可用

- U 盘 EFI（`MacBookPro16,2`，PGPU 屏蔽关闭，`nvfb=1 nvaccel=1 nvfbheads=4`，带 `-NRedNoSubmitGuard`）
- `NullMoth 2.5.3` 安装成功；`NVRM/NVRMFB/NVAccel/NVRMAGDC` 全部加载
- `NVAccel` 注册、Metal 插件 `NVMTLDriver`、`recoveryCount=0`
- `system_profiler` 出现 `NVIDIA GeForce RTX 3060 Laptop GPU`，**Metal 3**
- 开机守卫日志：`the desktop came up and stayed up; this start counts as good`
- 面板仍由核显点亮（1696×954@240）

### 8.2 独显直连（MUX=仅 NVIDIA）：能进桌面，但驱动没有接管显示

| 项 | 实测 |
| --- | --- |
| 引导 | 纯独显配置（`iMacPro1,1`、NootedRed 关、`Kernel/Block` 屏蔽 `IONDRVSupport`） |
| 结果 | **能进桌面**，2560×1440，Metal 3，无 panic |
| 关键日志 | `NVHm-xnu: PRISS 2 REACHED: the adapter is up & GSP-RM is running` → 驱动完全起来 |
| 连接器 | `getDisplays: 4 handle(s)` → 4 个全 `connected 0 edid 0 connector 0x0` → **`ENUMERATED: 0 connected display(s)`** |
| 帧缓冲 | 只接管了固件控制台缓冲：`vtFb ... size 14745600`（=2560×1440×4） |
| 刷新率 | 系统上报 `0.0 Hz`；实测 CVDisplayLink **30.0 Hz**（`nvfbheads=1`，间隔稳定 33 ms；`nvfbheads=4` 时抖动 7–58 ms） |
| 加速器 | `auto-go: heads published mask 0x0 (0 of nvfbheads 1) after 20 s`、`accelerator level 1 NEVER took after 11 tries`、`display stack: heads 0x0 accel 0 agdc -99 k5 -99 iop -99` |
| AGDC 开关 | `sysctl debug.nvrmfb_agdc=1` → `cmds=0`、`fbmap=never scanned`，**无效** |
| 守卫 | `boot guard cleared: NVKMS lists this card's connectors and none has a display`（不会自关机） |

**结论**：瓶颈是 **NVKMS 认不到这块笔记本 eDP 内屏的连接器**（不是 SMBIOS、不是 `IONDRVSupport`、不是 boot-args、不是 GSP 固件）。驱动能跑 GSP-RM、能出 Metal，但没有任何 head 可发布，WindowServer 等 20 s 超时后回落到固件帧缓冲 → 30 Hz、无模式/刷新率控制。

### 8.3 已发现但未使用的驱动开关（从二进制里翻出来的）

- `NVRM`：`-nvkmsnosmooth`、`-nvrmsmallbar`、`-nvrmnogo`、`-nvrmnoflip`、`-nvrmnobootscreen`、`-nvoff`
- `NVRMFB`：`-nvfbsurvey`、`-nvrmnodpms`、`-nvrmnogamma`、`-nvoff`

`-nvfbsurvey` 可能打印更详细的连接器探测过程（诊断用）；`-nvrmsmallbar` 影响 BAR1 放置（本机日志已显示 "BAR1 placed outside the console"）。

### 8.4 建议

1. **日常用混合模式**：核显 240 Hz 显示 + 3060 作为 Metal 加速器（已验证可用）。
2. 独显直连保留为"可启动、显示不可用（30 Hz）"的实验结论。
3. 若还要推进，唯一有信息量的下一步是用 `-nvfbsurvey` 抓一次探测日志，确认是"连接器表为空"还是"EDID/AUX 读不到"。


### 8.5 补全官方配置后：独显直连 240 Hz 成功（2026-10-24 17:11）

补上 HOW-IT-WORKS §4 要求的 `ResizeGpuBars=13`、`ResizeAppleGpuBars=-1`、`amfi_get_out_of_my_way=0x1 amfi=0x80`，以及 issue #377 的 `nvrmsettle=10000` 后：

| 指标 | 结果 |
| --- | --- |
| 内屏 | **NVIDIA GeForce RTX 3060 Laptop GPU** 驱动，2560×1440 **@240 Hz** |
| 实测节拍 | CVDisplayLink **239.9 Hz**（间隔 4.17 ms，min 4.04 / max 4.32） |
| 可选模式 | 94 个（含 32 个 HiDPI，最大 HiDPI 1280×720@240） |
| headless 等待 | 17:12:57 armed → 17:13:13 `not headless (a display head published)` ≈ **16 s**，不是 60 s |
| iGPU | 在 MUX=独显下不可见（`10de`=2、`1002:1638`=0） |
| panic | 无；`recoveryCount=0` |
| WindowServer CPU | **≈84%**（10 s 空闲采样；混合模式下同一动作约 35%） |

结论修正：**笔记本 eDP 是可以被驱动的**（此前 #170 笔记本案例只做到计算，说明 2.5.3 + 完整 OpenCore 配置有进展）。代价是 CPU 占用显著更高、开机多约 16 s、dGPU 常开。


---

## 9. 性能对比：混合（NootedRed 点屏） vs 独显直连（NullMoth）

测量条件：同一台机、10/24 下午；WindowServer CPU 用 `ps cputime` 差值（采样期间 DSH 自身在持续渲染，绝对值偏高，但两种模式同口径可比）。

| 指标 | 混合模式 | 独显直连 |
| --- | --- | --- |
| 刷新率 | 240.0 Hz（实测） | 239.7–240.0 Hz（实测） |
| 内屏像素 | HiDPI 1696×954（背板 3392×1908） | 1:1 1888×1062（可选 2560×1440 或 1280×720@2x） |
| WindowServer CPU | **33–42 %** | **84–129 %** |
| 60 Hz 时 | 40 %（原生 1440p） | **86 %**（几乎不降） |
| 开机 | 直接进 | 多约 **16 s**（headless check 等 head 发布） |
| 3060 可用性 | 日常内置 ESP 里被 SSDT 屏蔽（不可用） | **唯一 GPU，直接上屏** |
| 待机温度 | — | 52 °C（`ioreg` NVAccel） |
| 驱动成熟度 | NootedRed 1.0.1（需 `-NRedNoSubmitGuard`） | NullMoth 2.5.3（新；OpenGL/Chrome/Parallels 有已知 issue） |

**关键发现**：独显直连的 WindowServer CPU 在 60 Hz 和 240 Hz 下几乎一样（86 % vs 93 %），说明它的合成开销**不是按帧**计费的，而是常驻的拷贝/合成路径成本——这也是为什么 N 卡模式更耗电、发热更高。

**结论**：日常（办公/看片/续航）用混合；需要 3060 直接出画面（游戏、渲染、外接屏）时用独显直连。U 盘 EFI 已定位为「独显直连专用启动盘」。

### 9.1 已做的优化

- `ResizeGpuBars=13`、`ResizeAppleGpuBars=-1`（BAR1 8 GB，NVRM 才能把 BAR1 挪出控制台）
- `amfi_get_out_of_my_way=0x1 amfi=0x80`（让 WindowServer 加载驱动 bundle）
- `nvrmsettle=10000`（绕开启动等待）
- `-nvrmnobootscreen`（跳过 boot screen 快照，2026-10-24 17:30 加入，待下次重启验证开机速度）

### 9.2 还可以做的

1. 日常去掉 `-v`（保留 `keepsyms=1 debug=0x100`），减少跑码时间与日志量。
2. 试用 `nvrmsettle=3000`（现在 10000 只是等待上限）。
3. 系统设置 → 辅助功能 → 显示：**降低透明度 / 减弱动态效果**（直接降低合成开销）。
4. 亮度：验证 F1/F2 与控制中心滑块（NVRMFB 有 backlight handler）；异常时试 `-nvrmnogamma`。
5. 休眠：驱动支持 sleep/wake（仓库 `docs/SLEEP-WAKE.md`），当前全局 `SleepDisabled=1`，可试开启。
6. 系统升级（15.x → 26）后需重装驱动；保留 U 盘上的两份 zip 作退路。
