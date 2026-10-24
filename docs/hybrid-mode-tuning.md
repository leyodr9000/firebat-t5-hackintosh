# 混合模式配置基线（NootedRed）

对象：内置 ESP `EFI/OC` · macOS 15.8.1 · NootedRed weachy fork 1.0.1

---

## 1. boot-args

```
keepsyms=1 ipc_control_port_options=0 -vi2c-force-polling -amfipassbeta -lilubetaall
itlwm_he=1 -NRedGLFix -NRedVDA -NRedVCNNoGfxOff -NRedNoSubmitGuard -nvoff
watchdog=1 debug=0x100
```

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `-NRedGLFix` | 关 | 用户态 OpenGL/Metal 补丁，修 Chromium/Electron 花屏 |
| `-NRedVDA` | 关 | VideoToolbox 硬解（AVC + HEVC） |
| `-NRedVCNNoGfxOff` | 关 | VDA 会话期间禁止 GFXOFF |
| `-NRedAVC` | 关 | 只硬解 H.264，HEVC 回软解（替代 `-NRedVDA`） |
| `-NRedAVCEncode` | 关 | 实验性 H.264 硬编 |
| `-NRedNoSubmitGuard` | 保护默认开 | 关闭 SubmitGuard；移除后出现 `element modified after free` |
| `-NRedDebug` | 关 | 详细日志 |
| `-NRedOff` | 关 | 停用全部补丁 |

## 2. 硬性要求

| # | 要求 | 现状 |
| --- | --- | --- |
| 1 | Lilu ≥ 1.7.0 | 1.7.2 |
| 2 | NootedRed 1.0.0 / 1.0.1 | 1.0.1（自编译，背光补丁） |
| 3 | 移除 bfixup.kext | 无 |
| 4 | NootedRed 在 AppleALC 之前 | 第 3 位（VirtualSMC 之后） |
| 5 | 不与 WhateverGreen 同用 | 未安装 |
| 6 | 使用各版本末期小版本（15.8 等） | 15.8.1 |

顺序：`Lilu → VirtualSMC → NootedRed → AppleALC → SimpleHDA → AppleMCEReporterDisabler → …`

## 3. 相关配置

| 项 | 值 |
| --- | --- |
| `csr-active-config` | `0xFF0F0000` |
| `SecureBootModel` | Disabled |
| `Kernel/Patch` | 55 条 AMD Vanilla，`MaxKernel` 覆盖至 `25.99.99` |
| ACPI | 8 个 SSDT，含 `SSDT-AMD-PGPU-DISABLE`（屏蔽 dGPU） |
| `DeviceProperties` | `layout-id 39`（音频）、`acpi-wake-type 1` |
| NVRAM | `revpatch=auto,sbvmm,cpuname`、`revcpuname=AMD Ryzen 5 5600H` |

## 4. 可选项

| 项 | 做法 | 影响 |
| --- | --- | --- |
| HEVC 播放不稳 | 加 `-NRedAVC` | 仅 H.264 硬解 |
| 需要硬编 | 加 `-NRedAVCEncode` | 实验特性 |
| 停止休眠 | `sudo pmset -a disablesleep 1 sleep 0` | 常插电使用 |
| 无用 kext | 停用 `SMCLightSensor`、`IntelMKLFixup` | 收益极小 |

## 5. 切回混合模式

1. Windows：NVIDIA 控制面板 → 管理显示模式 → 自动/混合
2. 用内置硬盘启动（MUX 为独显时内置 EFI 不亮屏）
3. 校验：

```sh
sysctl -n kern.bootargs | grep -c nvfb     # 期望 0
ioreg -l -w0 | grep -c de10                # 期望 0
```

4. 分辨率切回 1696×954 HiDPI（`display-bench.py --list` 可列模式）
