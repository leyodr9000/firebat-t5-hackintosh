# Firebat T5A 黑苹果 — 技术档案

机器：Firebat T5A / 主板 GM5ZG0O-F · R5-5600H（Cezanne）· macOS 15.8.1 (24H32) · OpenCore 1.0.8

---

## 1. 硬件与系统

| 项 | 值 |
| --- | --- |
| CPU | AMD Ryzen 5 5600H（6C12T，Cezanne） |
| iGPU | Radeon Vega 7 `1002:1638`（NootedRed 驱动） |
| dGPU | RTX 3060 Laptop `10de:2520`（MUX，可切换） |
| 内屏 | BoE NE156QHM-NZ2，2560×1440 240Hz eDP（VendorID 2533 / ProductID 2645） |
| 无线 | Intel AX200（AirportItlwm 2.4.0） |
| 有线 | RTL8125（RTL812xLucy） |
| SMBIOS | `MacBookPro16,2` |
| 引导 | OpenCore REL-108，ESP 卷名 `NO NAME`，1.1 GB |

## 2. 卷与路径

| 卷 / 路径 | 说明 |
| --- | --- |
| 内置 ESP（1.1 GB，`NO NAME`） | OpenCore 主 EFI；disk 编号每次启动会变，按容量定位 |
| Windows ESP（314 MB / 736 MB） | 不含 OC |
| U 盘 ESP（209.7 MB） | 独显直连专用启动盘 |
| `~/Documents/deepseek-harness/default-workspace` | 文档、脚本、EFI 变体、备份 |
| `/Volumes/STAGE/nrd-src/NootedRed-master` | 已打补丁的 NootedRed 源码 |
| `~/nrd-build/` | xmake + AppleForge 构建环境 |

## 3. 生效配置（内置 EFI）

**boot-args**

```
keepsyms=1 ipc_control_port_options=0 -vi2c-force-polling -amfipassbeta -lilubetaall
itlwm_he=1 -NRedGLFix -NRedVDA -NRedVCNNoGfxOff -NRedNoSubmitGuard -nvoff
watchdog=1 debug=0x100
```

| 参数 | 作用 |
| --- | --- |
| `-NRedGLFix -NRedVDA -NRedVCNNoGfxOff` | NootedRed 推荐的 GL 修复 / 硬解 / VCN 禁 GFXOFF |
| `-NRedNoSubmitGuard` | 关闭 Metal SubmitGuard；移除后出现 `element modified after free` panic |
| `-nvoff` | 混合模式下 NVRM 不接触 GPU |
| `watchdog=1` | 看门狗超时改为 panic，定位静默重启 |
| `debug=0x100` | panic 后不自动重启 |
| `-vi2c-force-polling` | VoodooI2C 轮询模式 |

**其它**

| 项 | 值 |
| --- | --- |
| `csr-active-config` | `0xFF0F0000` |
| `SecureBootModel` | Disabled |
| `ResizeGpuBars` | `-1`（内置 EFI；U 盘独显 EFI 为 `13`） |
| `ResizeAppleGpuBars` | U 盘独显 EFI 为 `-1` |
| `Kernel/Add` | 33 项，顺序 Lilu → VirtualSMC → NootedRed → AppleALC → … |
| `Kernel/Block` | `com.apple.iokit.IONDRVSupport`（仅独显直连 EFI） |
| ACPI | SSDT-ALS0 / EC / PLUG-ALT / PNLF / USB-Reset / USBX / XOSI / AMD-PGPU-DISABLE |
| `NVMeFix` | 停用（第 21 条，`Enabled=false`） |

**电源（常插电、禁休眠）**

```
SleepDisabled=1  sleep=0  hibernatemode=0  disksleep=0
standby=0  powernap=0  womp=0  proximitywake=0  tcpkeepalive=0
```

## 4. 变更记录（内置 EFI）

| 时间 | 变更 | 备份 |
| --- | --- | --- |
| 10-24 12:53 | 移除 `-NRedNoSubmitGuard` | `config-backup-20261024-1253.plist` |
| 10-24 16:08 | 恢复 `-NRedNoSubmitGuard` | `config-backup-20261024-1610.plist` |
| 10-24 17:52 | 删除 `-v`、`debug=0x100` | `config-backup-1614.plist` |
| 10-24 17:55 | 追加 `-nvoff` | 同上 |
| 10-24 18:05 | 停用 `NVMeFix.kext` | `config-backup-1805.plist` |
| 10-24 18:38 | 追加 `watchdog=1 debug=0x100` | `config-backup-1838.plist` |
| 10-24 17:50 | `Kernel/Add` 将 NootedRed 移到 AppleALC 之前 | `config-backup-1750.plist` |
| 10-24 19:40 | 自编译 NootedRed（背光曲线补丁） | `Kexts-backup/NootedRed.kext.orig` |

## 5. 已知问题

| 问题 | 状态 |
| --- | --- |
| 硬件光标失效 | 未解决 |
| eDP DPCD 读取失败 | 未解决 |
| Android USB 网络共享不可用 | 未解决 |
| 启动台切页掉帧 | 未解决 |
| 关机偶发卡死 | 10-23 记录 5 次 |
| Wi-Fi `0xe00002c7` 日志刷屏 | AirportItlwm 缺 WCL 统计，仅日志噪音 |
| 独显直连下 H.264 无硬解 | `libnvdec_h264.dylib` adhoc 签名被 AMFI 拒绝 |

## 6. 操作约束

1. MUX 切到独显后不能用内置 ESP 启动（内置 EFI 屏蔽 dGPU，屏不亮），只能走 U 盘独显 EFI。
2. disk 编号每次启动漂移，定位 ESP 一律按容量（`1.1 GB`）过滤。
3. `/tmp` 会被系统清理，构建工具链与临时脚本放 `~` 下。
4. 改 boot-args 后执行 `sudo nvram -d boot-args`，否则下次仍使用旧值。
5. kext 必须 ad-hoc 签名：`codesign --force --deep -s - X.kext`。

## 7. 命令

```sh
# 挂载内置 ESP（读写）
DEV=$(diskutil list | grep '1.1 GB' | grep EFI | awk '{print $NF}')
sudo diskutil mount -mountPoint /tmp/esp "$DEV"

# 备份 + 校验 config
cp /tmp/esp/EFI/OC/config.plist /tmp/esp/EFI/OC/config-backup-$(date +%Y%m%d-%H%M).plist
plutil -lint /tmp/esp/EFI/OC/config.plist
ocvalidate /tmp/esp/EFI/OC/config.plist

# 卸载
sudo diskutil unmount /tmp/esp

# 状态
kextstat | grep -iE 'nootedred|applealc|nvrm'
ioreg -rc AppleBacklightDisplay -l | grep -o '"linear-brightness"={[^}]*}'
sysctl -n kern.bootargs
ioreg -l -w0 | grep -c de10        # 0 = 独显已屏蔽
```

## 8. 测量数据

**显示节拍（CVDisplayLink）**

| 模式 | 分辨率 | 刷新 | 间隔 min/med/max |
| --- | --- | --- | --- |
| 混合（iGPU） | 1696×954 HiDPI（背板 3392×1908） | 240.0 Hz | 4.165 ms 恒定 |
| 独显直连（NVIDIA） | 2560×1440 1:1 | 239.9–240.0 Hz | 4.04 / 4.17 / 4.32 ms |

**WindowServer CPU（同口径 10 s 采样）**

| 模式 | 占用 |
| --- | --- |
| 混合 | 33–42 % |
| 独显直连 | 84–129 %（60 Hz 下仍 86 %） |

## 9. 亮度

面板 PWM 低于约 2% 时背光熄灭；系统下发的 `linear-brightness` 约为 UI 的 4 次方，直接作 PWM 使用会让底部落入黑区。

修复：自编译 NootedRed，`pwm = 2% + frac × 98%`。详见 `brightness-fix-tech-doc.md`。

## 10. 待办

| 项 | 说明 |
| --- | --- |
| 硬件光标 | 待查 |
| 关机卡死 | 待定位（`shutdown_stall` 报告） |
| 68 键键位映射 | 未定方案 |
| 独显直连硬解 | 需上游更换签名 |
