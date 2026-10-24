# Firebat T5A 黑苹果 EFI（OpenCore 1.0.8）

**适用**：Firebat T5A / 同方 GM5ZG0O-F、AMD Ryzen 5 5600H（Cezanne）、BoE NE156QHM-NZ2 2560×1440 240Hz eDP、Intel AX200。
**系统**：macOS 15.8.1 (24H32)

## ⚠️ 使用前必读

1. **SMBIOS 已脱敏**：`config.plist` 里的 `SystemSerialNumber` / `MLB` / `SystemUUID` / `ROM` 全部是占位符。
   请用 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) 生成一套自己的 `MacBookPro16,2` 三码后填入，否则 iCloud/iMessage 不可用。
2. **CPU 不同则必须重做内核补丁**：本 EFI 的 `Kernel/Patch` 是针对 **Cezanne (Ryzen 5000)** 的 AMD Vanilla 补丁，换 CPU 架构会直接起不来。
3. 装机前请先确认：BIOS 关闭 CSM、开启 Above 4G Decoding、关闭 Secure Boot。

## 目录

```
EFI/
├── BOOT/              引导入口
└── OC/
    ├── ACPI/          8 个 SSDT（EC / PLUG-ALT / PNLF / USBX / XOSI / PGPU-DISABLE 等）
    ├── Drivers/       OpenRuntime 等
    ├── Kexts/         33 个 kext（含自编译的背光补丁版 NootedRed）
    ├── Resources/     开机资源
    ├── Tools/         OpenShell 等
    ├── OpenCore.efi   REL-108
    └── config.plist   已脱敏
docs/                  完整技术文档（见 00-index.md）
```

## 关键 boot-args

```
keepsyms=1 ipc_control_port_options=0 -vi2c-force-polling -amfipassbeta -lilubetaall
itlwm_he=1 -NRedGLFix -NRedVDA -NRedVCNNoGfxOff -NRedNoSubmitGuard -nvoff
watchdog=1 debug=0x100
```

## 已知问题

- 硬件光标（hardware cursor）失效
- eDP DPCD 读取失败
- Android USB 网络共享不可用
- 启动台切页掉帧
- 亮度曲线需靠自编译 NootedRed（见 `docs/brightness-fix.md`）

## 许可

文档与配置以 CC BY-NC-SA 4.0 提供；第三方 kext 版权归各自作者。
