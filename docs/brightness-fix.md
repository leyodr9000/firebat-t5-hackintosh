# macOS 黑苹果 AMD 核显：背光曲线修复

## 1. 背光控制链路

```
Control Center / 系统设置（UI 0..1）
  → DisplayServicesSetBrightness()
  → AppleBacklightDisplay（IOGraphicsFamily）
      IODisplayParameters:
        "brightness"               {min=0, max=1024,  value=N}   ← UI 侧
        "linear-brightness"        {min=0, max=65403, value=M}   ← 真正下发
        "usable-linear-brightness" {min=655, max=65403}
  → NootedRed: Backlight::wrapSetAttributeForConnection()
        percentage = cur * 100 / max
        ├─ supportsAUX → dc_link_set_backlight_level_nits(link, true, maxMillinits*pct/100, 50)
        └─ 否则        → dc_link_set_backlight_level(link, pct*0xFFFF/100, 0)
  → 面板 PWM 占空比
```

关键点：`linear-brightness` 是**线性亮度**，不是百分比。NootedRed 默认把它直接当百分比乘 PWM，于是 `linear/max = 2%` 时 PWM 只有 2%。

## 2. 两个必须知道的硬件/系统常数

**① UI → linear 是约 4 次方关系**

| UI | linear | linear/max |
| --- | --- | --- |
| 0.05 | 655 | 0.0100 |
| 0.10 | 771 | 0.0118 |
| 0.20 | 1180 | 0.0180 |
| 0.30 | 1859 | 0.0284 |
| 0.40 | 3020 | 0.0462 |
| 0.50 | 4980 | 0.0761 |
| 0.60 | 8228 | 0.1258 |
| 0.70 | 13684 | 0.2092 |
| 0.80 | 22824 | 0.3490 |
| 0.90 | 38930 | 0.5952 |
| 1.00 | 65403 | 1.0000 |

推论：按 **linear 比例**做下限（例如 22%）等价于把 **UI 70%** 以下全部钉死。

**② 面板 PWM 阈值 ≈ 2%**
低于约 2% 占空比背光直接熄灭（不是变暗）。该值因面板而异，用二分法可测：从 0 缓慢上调，记录首次可见的 `linear/max`。

## 3. 修复：把 linear 比例线性铺到 [PWM_min, 100%]

`NootedRed/Backlight.cpp` → `Backlight::wrapSetAttributeForConnection()`：

```cpp
singleton().curBacklightLvl = static_cast<UInt32>(value);

static constexpr double kMinPwmPercent = 2.0;    // 面板 PWM 熄灭阈值
static constexpr double kMaxPwmPercent = 100.0;
UInt32 percentage = 0;
if (singleton().maxBacklightLvl != 0) {
    double frac = static_cast<double>(singleton().curBacklightLvl) /
                  static_cast<double>(singleton().maxBacklightLvl);
    frac = frac < 0.0 ? 0.0 : (frac > 1.0 ? 1.0 : frac);
    percentage = static_cast<UInt32>(
        kMinPwmPercent + frac * (kMaxPwmPercent - kMinPwmPercent) + 0.5);
}
```

后续两个分支（nits / pwm）保持原样，`percentage` 直接复用。

**变体：中段提亮（gamma）**

```cpp
#include <cmath>
double ui = std::sqrt(frac);                    // 或 std::pow(frac, 0.45)
percentage = static_cast<UInt32>(kMinPwmPercent + ui * (kMaxPwmPercent - kMinPwmPercent) + 0.5);
```

`ui = frac` 时保留原曲线形状（中段偏暗）；`sqrt` 后中段明显提亮。

## 4. 为什么不在其他层修（技术约束）

| 层 | 结论 | 原因 |
| --- | --- | --- |
| ACPI `SSDT-PNLF` | 无效 | Dortania 最小版只有 `_HID APP0002` / `_CID backlight` / `_UID 19` / `_STA`，无 `_BCM`/`_BQC`；背光走 DC 硬件，不经 ACPI |
| IOKit 参数 | 只能改值 | `IODisplaySetIntegerParameter` / `IODisplaySetFloatParameter` 存在且可调用，但 min/max 只读；`IODisplayGetIntegerParameter` 未导出（`dlsym` 失败） |
| 用户态守护 | 治标 | `DisplayServicesGet/SetBrightness` 轮询回写属事后纠偏：面板在写低的那一帧已熄灭，轮询再快也有可见闪黑 |

## 5. 采样脚本（测 UI↔linear 曲线）

```python
import ctypes, time, subprocess, re
ds = ctypes.CDLL('/System/Library/PrivateFrameworks/DisplayServices.framework/DisplayServices')
cg = ctypes.CDLL('/System/Library/Frameworks/CoreGraphics.framework/CoreGraphics')
cg.CGMainDisplayID.restype = ctypes.c_uint32
d = cg.CGMainDisplayID()
ds.DisplayServicesSetBrightness.argtypes = [ctypes.c_uint32, ctypes.c_float]

def lin():
    out = subprocess.run(['ioreg', '-rc', 'AppleBacklightDisplay', '-l'],
                         capture_output=True, text=True).stdout
    m = re.search(r'"linear-brightness"=\{"min"=(\d+),"max"=(\d+),"value"=(\d+)\}', out)
    return int(m.group(3)), int(m.group(2))

for ui in (0.05, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0):
    ds.DisplayServicesSetBrightness(d, ui)
    time.sleep(0.35)
    v, mx = lin()
    print(f"{ui:.2f} {v} {v/mx:.4f}")
```

## 6. 构建

依赖：**Command Line Tools（clang 17）即可，不需要完整 Xcode**。

| 组件 | 获取 |
| --- | --- |
| xmake 2.9.9 | `xmake-io/xmake` release `xmake-bundle-v2.9.9.macos.x86_64`（单文件可执行） |
| AppleForge | `git clone --recurse-submodules https://git.chefkiss.dev/AppleHax/AppleForge.git`（含 Lilu / MacKernelSDK 子模块） |

```sh
mkdir -p ~/nrd-build && cd ~/nrd-build
curl -LO https://github.com/xmake-io/xmake/releases/download/v2.9.9/xmake-bundle-v2.9.9.macos.x86_64
chmod +x xmake-bundle-v2.9.9.macos.x86_64 && mv xmake-bundle-v2.9.9.macos.x86_64 xmake
git clone --recurse-submodules https://git.chefkiss.dev/AppleHax/AppleForge.git

export AppleForgePath=~/nrd-build/AppleForge
cd <NootedRed 源码目录>
~/nrd-build/xmake f -a x86_64 -m release -y
~/nrd-build/xmake -y
# 产物: build/branches/shared/macosx/<plat>/x86_64/release/NootedRed.kext
```

- 结尾报 `cannot archive ... /usr/bin/zip not found` 可忽略（仅打包 zip 失败）
- **kext 必须 ad-hoc 签名**：`codesign --force --deep -s - NootedRed.kext`
- 不要用 `/tmp` 存放工具链（系统会清理）

## 7. 部署

```sh
sudo diskutil mount disk0s1                       # 内置 ESP（卷名 NO NAME）
ESP="/Volumes/NO NAME"
sudo cp -R "$ESP/EFI/OC/Kexts/NootedRed.kext" "$ESP/EFI/OC/Kexts-backup/NootedRed.kext.orig"
sudo rm -rf "$ESP/EFI/OC/Kexts/NootedRed.kext"
sudo cp -R NootedRed.kext "$ESP/EFI/OC/Kexts/NootedRed.kext"
sudo diskutil unmount disk0s1
```

回滚即反向复制 `Kexts-backup/NootedRed.kext.orig`。

## 8. 验证

```sh
kextstat | grep -i nootedred        # 确认加载的是新 kext（对比 UUID / 地址）
ioreg -rc AppleBacklightDisplay -l | grep -o '"linear-brightness"={[^}]*}'
```

行为验收：
1. 滑块从头到尾单调变化（无死区）
2. 拖到最底：暗但**不熄屏**
3. 顶部：满亮度
4. 重启后无 panic

## 9. 参数速查

| 参数 | 位置 | 含义 |
| --- | --- | --- |
| `kMinPwmPercent` | `Backlight.cpp` | 面板 PWM 熄灭阈值（本机 2.0） |
| `kMaxPwmPercent` | 同上 | 上限，通常 100 |
| `ui = frac` / `sqrt(frac)` / `pow(frac,0.45)` | 同上 | 曲线形状，越大中段越亮 |
| `maxBacklightLvl` | 运行时 | 取自 `linear-brightness.max`（65403） |
