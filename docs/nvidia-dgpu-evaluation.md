# NullMoth NVIDIA 驱动（2.5.3）实测记录

对象：Firebat T5 · R5-5600H + RTX 3060 Laptop `10de:2520` · macOS 15.8.1

---

## 1. 驱动组成

| 组件 | 位置 | 作用 |
| --- | --- | --- |
| `NVRM.kext` | `/Library/Extensions` | RM + NVKMS，加载 GSP 固件 |
| `NVRMFB.kext` | 同上 | IOFramebuffer，head/mode/vblank |
| `NVAccel.kext` | 同上 | IOAccelerator，合成表面 |
| `NVRMAGDC.kext` | 同上 | AppleGraphicsDeviceControl 策略 |
| `NVMTLDriver.bundle` | `/Library/GPUBundles` | Metal 驱动插件（含 `libnvdec_*`） |
| GSP 固件 | `/Library/NullMoth/nvfw/nvidia/615.78.08` | |

安装脚本：`install.sh`（tarball 内 `pkgroot/` + `SHA256SUMS`）。
`install.sh` 在 PCI 上找不到 `10de` 设备时直接退出。

## 2. OpenCore 配置要求

| 项 | 值 |
| --- | --- |
| `UEFI/Quirks/ResizeGpuBars` | `13`（8 GB BAR1，NVRM 才能把 BAR1 移出控制台） |
| `Booter/Quirks/ResizeAppleGpuBars` | `-1` |
| `Kernel/Block` | `com.apple.iokit.IONDRVSupport`（Exclude） |
| `csr-active-config` | `0x0A43`（更宽松值同样可用） |
| `SecureBootModel` | Disabled |
| boot-args | `nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth nvrmsettle=10000 amfi_get_out_of_my_way=0x1 amfi=0x80` |

原始配置缺 `ResizeGpuBars` / `ResizeAppleGpuBars` / amfi 参数时，独显直连只能停在内核控制台帧缓冲（30 Hz）。

## 3. 实测：混合模式

| 项 | 结果 |
| --- | --- |
| 加载 | NVRM / NVRMFB / NVAccel / NVRMAGDC 全部加载 |
| Metal | Metal 3，`NVMTLDriver` 注册 |
| 显示 | 由 iGPU 驱动，dGPU 不接管 |
| `recoveryCount` | 0 |
| 硬解 | H.264 不可用（见 §5） |

## 4. 实测：独显直连（MUX = 仅 NVIDIA）

| 项 | 结果 |
| --- | --- |
| SMBIOS | `iMacPro1,1`，NootedRed 关闭 |
| 内屏 | RTX 3060 驱动，2560×1440 **240 Hz**（CVDisplayLink 239.9–240.0 Hz） |
| 可选模式 | 94 个，含 32 个 HiDPI（最大 1280×720@240） |
| headless 等待 | `17:12:57` armed → `17:13:13` 有 head 发布，约 16 s |
| iGPU | 不可见（`1002:1638` = 0） |
| 稳定性 | 无 panic，`recoveryCount=0` |

配置缺项时（无 `ResizeGpuBars` 等）的现象：

| 项 | 值 |
| --- | --- |
| 连接器 | `getDisplays: 4 handle(s)`，4 个均 `connected 0 edid 0 connector 0x0` |
| 枚举 | `ENUMERATED: 0 connected display(s)` |
| 帧缓冲 | 仅接管固件控制台缓冲（`vtFb` 14745600 B = 2560×1440×4） |
| 刷新率 | 系统报 `0.0 Hz`，实测 30.0 Hz |
| 加速器 | `heads published mask 0x0`、`accelerator level 1 NEVER took` |

## 5. 硬解状态（两种模式）

| 项 | 值 |
| --- | --- |
| `VTIsHardwareDecodeSupported(avc1)` | **False** |
| `VTIsHardwareDecodeSupported(hvc1)` | True |
| 原因 | `libnvdec_h264.dylib` / `libnvdec_hevc.dylib` 为 adhoc 签名，amfid 返回 `-423`（`not valid`） |
| 结论 | H.264 走软解；无法在用户态修复（需上游更换签名或彻底关闭 AMFI 库校验） |

## 6. 性能对比

| 指标 | 混合（iGPU 点屏） | 独显直连 |
| --- | --- | --- |
| 刷新 | 240.0 Hz | 239.9–240.0 Hz |
| 内屏 | HiDPI 1696×954（背板 3392×1908） | 1:1 2560×1440 或缩放 |
| WindowServer CPU | 33–42 % | 84–129 %（60 Hz 下 86 %） |
| 开机 | 直接 | 多约 16 s |
| dGPU 可用性 | 被 SSDT 屏蔽 | 唯一 GPU，直接上屏 |

独显直连的合成开销与刷新率基本无关，属常驻拷贝路径成本。

## 7. 驱动开关（二进制提取）

| 来源 | 开关 |
| --- | --- |
| `NVRM` | `-nvkmsnosmooth` `-nvoff` `-nvrmnobootscreen` `-nvrmnoflip` `-nvrmnogo` `-nvrmnopm` `-nvrmsmallbar` |
| `NVRMFB` | `-nvfbsurvey` `-nvoff` `-nvrmnodpms` `-nvrmnogamma` |
| 运行时可调 | `debug.nvrmfb_agdc` `debug.nvaccel_iop` `debug.nvaccel_heads_published` 等 `sysctl` |

## 8. 结论

| 场景 | 选择 |
| --- | --- |
| 日常（办公 / 视频 / 续航） | 混合模式 |
| 需要 dGPU 直接出画面（渲染 / 外接屏） | 独显直连（U 盘 EFI） |
