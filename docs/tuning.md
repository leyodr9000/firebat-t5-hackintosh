# 调优记录

配置项、取值理由与验证方法。对象：内置 EFI（混合模式）与 U 盘 EFI（独显直连）。

---

## 1. 显示

### 1.1 模式与 HiDPI

| 项 | 混合模式（iGPU） | 独显直连（NVIDIA） |
| --- | --- | --- |
| 240 Hz 模式数 | 195 个 1:1 + 132 HiDPI（需 duplicate-lowres 选项才可见） | 94 个（含 32 HiDPI） |
| HiDPI 上限 | 允许大于面板原生的背板（如 3392×1908） | 仅面板一半（1280×720），无更大背板 |
| 常用 | 1696×954（背板 3392×1908，≈150.9 %） | 2560×1440 1:1 或 1888×1062（≈135.6 %） |

限制：NVIDIA 驱动不提供大于面板原生的模式，因此无法构造 135 %–150 % 的 HiDPI（需要 3792×2132 以上背板）。

### 1.2 切换模式

`CGDisplaySetDisplayMode` 对 HiDPI 返回 1000，必须走配置 API：

```python
CGBeginDisplayConfiguration(&cfg)
CGConfigureDisplayWithDisplayMode(cfg, display, mode, None)
CGCompleteDisplayConfiguration(cfg, kCGConfigurePermanently)   # 1 = 永久
```

列出全部模式（含隐藏的 HiDPI）需设置 `kCGDisplayShowDuplicateLowResolutionModes = true`。

### 1.3 刷新率验证

`system_profiler` 报的刷新率不可靠（驱动缺项时显示 `0.0 Hz`）。用 CVDisplayLink 计 tick：

```python
cv.CVDisplayLinkCreateWithActiveCGDisplays(&link)
cv.CVDisplayLinkSetCurrentCGDisplay(link, CGMainDisplayID())
cv.CVDisplayLinkSetOutputCallback(link, cb, None)   # cb 里取时间戳
cv.CVDisplayLinkStart(link); sleep(4); CVDisplayLinkStop(link)
```

| 场景 | 结果 |
| --- | --- |
| 混合模式 240 Hz | 240.0 Hz，间隔恒 4.165 ms |
| 独显直连 240 Hz | 239.9–240.0 Hz，间隔 4.04 / 4.17 / 4.32 ms |
| 独显直连配置不全 | 30.0 Hz，间隔 7–58 ms 抖动 |

### 1.4 合成开销基线

WindowServer CPU，`ps -o cputime` 差值 / 10 s：

| 模式 | 分辨率 | 占用 |
| --- | --- | --- |
| 混合 | 1696×954 HiDPI | 33–42 % |
| 混合 | 2560×1440 1:1 | 71–87 % |
| 独显直连 | 1888×1062 | 84–129 % |
| 独显直连 | 同上，60 Hz | 86 % |

独显直连的占用与刷新率基本无关，为常驻拷贝路径成本；降低透明度、关闭动态效果可小幅改善。

## 2. 电源

常插电、禁休眠配置：

```sh
sudo pmset -a disablesleep 1 sleep 0 hibernatemode 0 disksleep 0
sudo pmset -a powernap 0 womp 0 proximitywake 0 tcpkeepalive 0
sudo rm -f /var/vm/sleepimage          # 释放 8 GB
```

| 项 | 取值 | 原因 |
| --- | --- | --- |
| `SleepDisabled` | 1 | 睡眠/唤醒易挂死；本机曾出现无 panic、无 SIGTERM 的静默复位 |
| `powernap` / `womp` / `proximitywake` | 0 | 暗唤醒路径在黑苹果上不稳定 |
| `hibernatemode` | 0 | 不需要休眠镜像 |
| `disksleep` | 0 | 与 `sleep 0` 配套，避免告警 |

## 3. kext 顺序

`Kernel/Add` 顺序影响 AppleGFXHDA 与 AppleALC 的加载冲突：

```
Lilu → VirtualSMC → NootedRed → AppleALC → SimpleHDA → AppleMCEReporterDisabler → …
```

验证：

```sh
kextstat | grep -iE 'nootedred|applealc' | awk '{print $1,$6}'   # NootedRed 的 id 应更小
```

## 4. 已停用的 kext

| kext | 原因 |
| --- | --- |
| `NVMeFix` | 构造函数空指针（page fault），启动 0.76 s 崩溃；本机 NVMe 不需要 APST 补丁 |

启用方式：`Kernel/Add` 对应条目 `Enabled=true`。

## 5. boot-args 取舍

| 参数 | 取舍 |
| --- | --- |
| `-v` | 日常移除（减少启动时间与日志）；排障时加回 |
| `debug=0x100` | 日常移除（panic 自动重启）；定位崩溃时加回 |
| `keepsyms=1` | 保留 |
| `watchdog=1` | 定位静默重启时加；会把看门狗超时转为 panic |
| `-nvoff` | 混合模式保留，避免 NVRM 在无卡环境加载 |
| `-NRedNoSubmitGuard` | 保留；移除后复现 `element modified after free` |

修改后必须清 NVRAM，否则沿用旧值：

```sh
sudo nvram -d boot-args
```

## 6. 启动时间

| 场景 | 额外耗时 |
| --- | --- |
| 移除 `-v` / `debug=0x100` | 减少数秒 |
| 独显直连 head 发布等待 | 约 16 s（`nvrmsettle` 与 head 计数相关） |

## 7. 背光

面板 PWM 低于约 2 % 熄灭；系统 `linear-brightness` 约为 UI 的 4 次方，直接当 PWM 会导致底部黑屏。修复见 `brightness-fix.md`。

## 8. 验证清单

```sh
sysctl -n kern.bootargs                       # boot-args 是否符合预期
kextstat | grep -iE 'nootedred|nvrm|applealc'
ioreg -l -w0 | grep -c de10                   # 0 = dGPU 已屏蔽
ioreg -rc AppleBacklightDisplay -l | grep -o '"linear-brightness"={[^}]*}'
pmset -g | grep -E 'SleepDisabled|hibernatemode|disksleep|powernap'
ls /Library/Logs/DiagnosticReports/*.panic 2>/dev/null | tail -3
```
