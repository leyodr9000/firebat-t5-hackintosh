# RTX 3060 独显直连（NullMoth 驱动）操作手册

适用：Firebat T5A / R5-5600H + RTX 3060 Laptop `10de:2520` · macOS 15.8.1

---

## 1. 前置条件

| 项 | 要求 |
| --- | --- |
| BIOS | Above 4G Decoding = On，CSM = Off，Secure Boot = Off |
| macOS | 15.x（驱动支持 15 / 26） |
| OpenCore | 1.0.8 |
| 显示模式 | BIOS/Windows 侧支持 MUX 切换（本机可在 NVIDIA 控制面板选择"仅 NVIDIA"） |
| 现有系统 | 混合模式可正常进系统（用于对照与回滚） |

## 2. OpenCore 配置

**基础**

| 键 | 值 | 说明 |
| --- | --- | --- |
| `PlatformInfo/Generic/SystemProductName` | `iMacPro1,1` | 无核显机型；`MacBookPro16,2` 会因缺 iGPU 卡住 |
| `UEFI/Quirks/ResizeGpuBars` | `13` | BAR1 扩到 8 GB，NVRM 才能把 BAR1 移出控制台帧缓冲 |
| `Booter/Quirks/ResizeAppleGpuBars` | `-1` | 不在 macOS 侧再改 BAR |
| `Kernel/Block` | `com.apple.iokit.IONDRVSupport`，Strategy=`Exclude` | 让 NVRMFB 独占 display index 0 |
| `NVRAM/csr-active-config` | `0x0A43`（`0xFF0F0000` 同样可用） | |
| `Misc/Security/SecureBootModel` | Disabled | |

**boot-args**

```
-v keepsyms=1 debug=0x100 ipc_control_port_options=0 -vi2c-force-polling
-amfipassbeta -lilubetaall itlwm_he=1
nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth -nvrmnobootscreen
nvrmsettle=10000 amfi_get_out_of_my_way=0x1 amfi=0x80
```

| 参数 | 作用 |
| --- | --- |
| `nvfb=1` | 启用 NVRMFB（IOFramebuffer） |
| `nvaccel=1` | 启用 NVAccel |
| `nvfbheads=4` | 期望发布的 head 数 |
| `-nvkmsnosmooth` | 关闭平滑接管 |
| `-nvrmnobootscreen` | 跳过 boot screen 快照，缩短启动等待 |
| `nvrmsettle=10000` | 启动等待上限（ms），规避早期挂起 |
| `amfi_get_out_of_my_way=0x1` `amfi=0x80` | 允许 WindowServer 加载驱动 bundle |

**同 EFI 内的其它改动**

| 对象 | 处理 |
| --- | --- |
| NootedRed | 关闭（`Enabled=false`） |
| `SSDT-AMD-PGPU-DISABLE` | 关闭（`Enabled=false`），否则 dGPU 在 PCI 上不可见 |
| ACPI | 其余 SSDT 保留 |

## 3. 制作 U 盘 EFI

```sh
# 1) 复制现有可用 EFI 到 U 盘 ESP
cp -R /Volumes/NO\ NAME/EFI /Volumes/U盘ESP/

# 2) 按第 2 节修改 config.plist（建议直接替换整份 config）
plutil -lint /Volumes/U盘ESP/EFI/OC/config.plist
ocvalidate /Volumes/U盘ESP/EFI/OC/config.plist
```

U 盘建议同时保留：原始 EFI 备份 zip、混合模式 EFI 备份 zip、切换步骤说明。

## 4. 切换 MUX

1. 进入 Windows → NVIDIA 控制面板 → 显示 → 管理显示模式 → **仅 NVIDIA GPU**
2. 完全关机（不是重启）
3. 开机按 F7（本机引导菜单）→ 选 U 盘 → 进 OpenCore → 选 macOS

## 5. 安装驱动

```sh
tar xzf nullmoth-nvidia-2.5.3.tar.gz
cd package
sudo ./install.sh
```

`install.sh` 行为：

| 步骤 | 说明 |
| --- | --- |
| 校验 | `SHA256SUMS` 逐文件校验 |
| 检查硬件 | `ioreg -r -c IOPCIDevice` 中必须有 `vendor-id = <de100000>`，否则直接退出 |
| 安装 | 4 个 kext → `/Library/Extensions`；`NVMTLDriver.bundle` → `/Library/GPUBundles`；固件 → `/Library/NullMoth/nvfw/nvidia/615.78.08` |
| 重建缓存 | `kmutil` 重建 Auxiliary Kernel Collection |

卸载：同目录 `uninstall.sh`。

## 6. 启动与等待

| 阶段 | 现象 |
| --- | --- |
| GSP 加载 | 日志 `NVHm-xnu: PRISS 2 REACHED: the adapter is up & GSP-RM is running` |
| head 发布 | `NVRMDisplay nub(s) published`，随后 `headless check: not headless` |
| 耗时 | 约 16 s（`nvrmsettle` + head 发布） |
| 桌面 | 出现后由 NVRMFB 驱动内屏 |

启动期间可能出现固件帧缓冲与驱动切换造成的短暂花屏，head 发布完成后消失。

## 7. 验证

```sh
# 内屏是否由 NVIDIA 驱动
system_profiler SPDisplaysDataType | grep -E 'Chipset Model|Resolution|UI Looks|Metal'

# kext 状态
kextstat | grep -i nvrm

# 连接器与 head
log show --last 10m --predicate 'eventMessage CONTAINS "ENUMERATED" OR eventMessage CONTAINS "headless"' | tail

# 实际刷新率（CVDisplayLink 采样）
python3 display-bench.py --list
```

判定标准：

| 项 | 正常值 |
| --- | --- |
| `Chipset Model` | `NVIDIA GeForce RTX 3060 Laptop GPU` |
| `Metal` | Metal 3 |
| `UI Looks like` | 含 `240.00Hz` |
| CVDisplayLink | 239.7–240.0 Hz |
| iGPU | `ioreg -l -w0 \| grep -c 1638` = 0 |

## 8. 故障对照

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 桌面 2560×1440 但只有 30 Hz，系统报 `0.0 Hz` | 缺少 `ResizeGpuBars=13` / `ResizeAppleGpuBars=-1` / amfi 参数，驱动未接管显示 | 补齐配置 |
| 日志 `ENUMERATED: 0 connected display(s)` | 同上 | 同上 |
| 启动卡住无画面 | MUX 未切换 / 用了内置 ESP | 确认 BIOS 显示模式，改从 U 盘启动 |
| 无 PANIC 但自动重启 | 看门狗复位 | 加 `watchdog=1` 观察 |
| H.264 播放 CPU 高 | `libnvdec_h264.dylib` adhoc 签名被 AMFI 拒绝，`VTIsHardwareDecodeSupported(avc1)=False` | 无法在用户态修复 |

## 9. 代价

| 项 | 数值 |
| --- | --- |
| 开机 | 比混合模式多约 16 s |
| WindowServer CPU | 84–129 %（混合模式 33–42 %） |
| 功耗 | dGPU 常开，待机温度约 52 °C |
| 内屏 HiDPI | 仅 1280×720（2×）；其余为 1:1 缩放 |
| H.264 硬解 | 不可用 |

## 10. 回滚

1. Windows → NVIDIA 控制面板 → 管理显示模式 → 自动 / 混合
2. 关机后用内置硬盘启动（内置 ESP 已屏蔽 dGPU，MUX 为独显时不会亮屏）
3. 如需彻底移除驱动：`sudo ./uninstall.sh`
