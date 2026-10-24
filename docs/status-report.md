# 系统状态报告

R5-5600H · macOS 15.8.1 (24H32) · OpenCore 1.0.8 · 采集时间 2026-10-24

---

## 1. 平台

| 项 | 值 |
| --- | --- |
| CPU | AMD Ryzen 5 5600H（Cezanne，6C/12T） |
| iGPU | Vega 7 `1002:1638`，NootedRed 1.0.1，Metal 3 |
| dGPU | RTX 3060 Laptop `10de:2520`（混合模式由 SSDT 屏蔽） |
| 内屏 | BOE NE156QHM-NZ2 2560×1440@240Hz，逻辑 1696×954（背板 3392×1908） |
| 内存 | 16 GB |
| 存储 | 512 GB NVMe（macOS）+ 1 TB NVMe（Windows） |
| 无线 | Intel AX200 / AirportItlwm 2.4.0 |
| 有线 | RTL8125 / RTL812xLucy |

## 2. 引导与安全

| 项 | 值 |
| --- | --- |
| OpenCore | REL-108 |
| SMBIOS | `MacBookPro16,2` |
| `csr-active-config` | `0xFF0F0000` |
| `SecureBootModel` | Disabled |
| `ResizeGpuBars` | `-1` |
| `SleepDisabled` | `1` |

**boot-args**

```
keepsyms=1 ipc_control_port_options=0 -vi2c-force-polling -amfipassbeta -lilubetaall
itlwm_he=1 -NRedGLFix -NRedVDA -NRedVCNNoGfxOff -NRedNoSubmitGuard -nvoff
watchdog=1 debug=0x100
```

## 3. 第三方 kext

| 类别 | kext |
| --- | --- |
| 基础 | Lilu 1.7.2 · VirtualSMC 1.3.8 · RestrictEvents · AMFIPass 1.4.1 |
| 图形 | NootedRed 1.0.1（自编译，背光补丁版本） |
| 音频 | AppleALC · SimpleHDA（layout-id 39） |
| 网络 | AirportItlwm 2.4.0 · RTL812xLucy |
| 蓝牙 | IntelBluetoothFirmware · IntelBTPatcher · BlueToolFixup |
| 输入 | VoodooI2C（+Services/GPIO/Input/HID）· VoodooPS2Controller（+Keyboard/Mouse/Trackpad） |
| 电源 / 传感 | AMDRyzenCPUPowerManagement 0.7.2 · SMCBatteryManager · SMCLightSensor · ECEnabler · ForgedInvariant |
| 其它 | USBToolBox · UTBDefault · NVMeFix（**已停用**）· AppleMCEReporterDisabler · BrightnessKeys · IntelMKLFixup |

## 4. 图形

| 项 | 值 |
| --- | --- |
| 显示节拍 | 240.0 Hz（CVDisplayLink，间隔 4.165 ms 恒定） |
| Metal | Metal 3 |
| IOAccelerator | `AMDRadeonX6000` / NootedRed 加速器已注册，`recoveryCount=0` |
| WindowServer CPU | 33–42 %（10 s 采样） |
| 背光 | `brightness` 0–1024 / `linear-brightness` 0–65403，下限已由 kext 钳制 |

## 5. 崩溃记录

| 时间 | 签名 | 现场 |
| --- | --- | --- |
| 10-22 ～ 10-23 | `element modified after free`（kalloc） | 6 次，调用栈无第三方 kext |
| 10-24 15:56 | 同上 | 移除 `-NRedNoSubmitGuard` 后复现，14 s |
| 10-24 17:49 | `AppleAPFSContainerScheme::acquireCryptoBuffer` 失败 `0xe00002de` | `apfsd`，启动 4.26 s |
| 10-24 18:01 | `NVMeFix` 构造函数空指针（page fault） | `kernel_task`，启动 0.76 s |

处置：

- `-NRedNoSubmitGuard` 恢复 → `element modified after free` 不再出现
- `NVMeFix` 停用 → 早启动空指针消失，此后冷启动一次通过

## 6. 关机卡死

| 记录 | 说明 |
| --- | --- |
| `shutdown_stall_*.shutdownStall` | 10-23 共 5 次，10-24 若干 |
| 18:35 静默重启 | 无 panic 文件、无 SIGTERM 记录、`shutdown cause 5`，判定为看门狗复位 |
| 处置 | `SleepDisabled=1` + 关闭 powernap/womp/proximitywake/tcpkeepalive；`watchdog=1 debug=0x100` 用于后续定位 |

## 7. 遗留问题

| 问题 | 状态 |
| --- | --- |
| 硬件光标失效 | 未解决 |
| eDP DPCD 读取失败 | 未解决 |
| Android USB 网络共享 | 不可用 |
| 启动台切页掉帧 | 未解决 |
| 独显直连下 H.264 无硬解 | AMFI 拒绝 adhoc 签名解码库 |
| Wi-Fi `0xe00002c7` 刷屏 | 日志噪音，不影响运行 |
