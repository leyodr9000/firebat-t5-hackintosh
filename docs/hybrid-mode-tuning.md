# 混合模式（核显日常）配置体检 & 优化建议

**对象**：内置 ESP（当前 `disk0s1`，卷名 `NO NAME`）上的 `EFI/OC`
**对照基准**：NootedRed weachy fork `v1.0.0` 发布说明（2026-10-22）
**检查时间**：2026-10-24 17:4x

---

## 1. 上游发布说明要点（weachy fork v1.0.0）

来源：<https://github.com/weachy/NootedRed/releases/tag/v1.0.0>

**推荐 boot-args（全功能）**

```
-NRedGLFix -NRedVDA -NRedVCNNoGfxOff
```

| 参数 | 作用 | 默认 | 说明 |
| --- | --- | --- | --- |
| `-NRedGLFix` | 用户态 OpenGL/Metal 双源补丁 | 关 | 修 Chromium/Electron（Chrome、VS Code、剪映等）花屏/菜单残影 |
| `-NRedVDA` | 打开 VideoToolbox 硬解（AVC+HEVC） | 关 | VCN/VDA 桥 |
| `-NRedVCNNoGfxOff` | VDA 会话期间禁止 GFXOFF | 关 | 防 Vega/Raven 视频播放/全屏切换时挂死 |
| `-NRedAVC` | **只硬解 H.264**，HEVC 退回软解 | 关 | `-NRedVDA` 的替代；HEVC 10-bit 不稳时用 |
| `-NRedAVCEncode` | 实验性 H.264 硬编 | 关 | — |
| `-NRedNoSubmitGuard` | **关闭** Metal SubmitGuard（默认开启的保护） | 保护默认开 | 官方说"仅排障用"；与我们的实测相反，见 §3 |
| `-NRedDebug` | 详细内核日志 | 关 | 排障用 |
| `-NRedOff` | 完全停用所有补丁 | 关 | 救砖/对比用 |

其它硬性要求：

- Lilu **≥ 1.6.7**，且必须在 NootedRed 之前加载
- **不要**与 WhateverGreen 同用
- 支持 Vega/GCN APU（含 Cezanne）；RDNA APU（680M/780M）不支持
- FairPlay 硬件 DRM 有限制（Netflix/Apple TV 用 Chromium 浏览器）

---

## 2. 本地配置对照

| 项 | 本机现状 | 结论 |
| --- | --- | --- |
| 推荐三项参数 | ✅ 全有（`-NRedGLFix -NRedVDA -NRedVCNNoGfxOff`） | 符合 |
| Lilu | 1.7.2（≥1.6.7 ✓） | 符合 |
| WhateverGreen | ✅ 未安装 | 符合 |
| NootedRed | **1.0.1**（比 fork 发布的 v1.0.0 还新，来自群/1401） | 注意版本来源 |
| SMBIOS | `MacBookPro16,2` | 正常 |
| csr-active-config | `0xFF0F0000`（全关） | 用户态页补丁必需，别收 |
| Kernel/Patch | 55 条，`MaxKernel` 覆盖到 `25.99.99` | 正常 |
| ACPI | 8 个 SSDT，含 `SSDT-AMD-PGPU-DISABLE`（混合模式屏蔽独显） | 正确 |
| DeviceProperties | 仅 2 项（音频 `layout-id=39`、`acpi-wake-type=1`） | 正常 |
| 睡眠 | 全局 `SleepDisabled=1` | 半残状态 |
| 显示 | 回混合后需重设 HiDPI（1696×954） | 记得切 |

---

## 3. 与官方说法冲突的一处

weachy v1.0.0 说明里，SubmitGuard 是**默认开启的保护**（拦截不合法的 Metal 命令缓冲、防 WindowServer panic），`-NRedNoSubmitGuard` 被标为"仅排障用"。

但本机实测：

| 状态 | 结果 |
| --- | --- |
| 带 `-NRedNoSubmitGuard` | 连续运行数小时稳定（10/23 起） |
| 移除该参数（10/24 12:53） | 当日 **15:56 复现** `element modified after free`（14 s，logd），与 10/22–10/23 那 6 次同签名 |
| 加回（16:08） | 恢复稳定 |

**结论**：目前保留该参数。但这与官方文档矛盾，属于"本机实证优先"的临时结论；如果后续再出现 panic，优先排查这里，并考虑向上游报告（1.0.1 fork 的 SubmitGuard 与 15.8.1 组合）。

---

## 4. 可优化项（按性价比）

| # | 优化 | 做法 | 收益 | 风险 |
| --- | --- | --- | --- | --- |
| 1 | 去掉跑码与调试参数 | ✅ **已执行（2026-10-24 17:52）**：boot-args 删 `-v`、`debug=0x100`，保留 `keepsyms=1` | 开机快、日志干净、panic 自动重启恢复 | 出问题看不到跑码 |
| 2 | 睡眠 | `SleepDisabled=1` → 0，配 `hibernatemode 0` 试 | 省电、合盖可睡 | 可能唤醒黑屏，可改回 |
| 3 | 视频解码策略 | 若 HEVC 播放不稳 → 加 `-NRedAVC`（只硬解 H.264） | 稳定性换 HEVC 硬解 | 无 |
| 4 | 硬编码 | 需要时加 `-NRedAVCEncode` | H.264 硬编 | 实验特性 |
| 5 | 无用 kext | `SMCLightSensor`（无光感）、`IntelMKLFixup`（不跑 MKL 应用）可停用 | 极小 | 低，建议不动 |
| 6 | 独显驱动 | 混合模式下 NullMoth 驱动仍会加载但无卡可驱动（NVRMAGDC 进 idle） | 无 | 无害，可保留；要彻底干净可 `uninstall.sh` |

> `-lilubetaall` 与 `-amfipassbeta` 建议保留：1.0.1 是非官方构建，Lilu/AMFIPass 的版本门槛仍按 beta 通道放开更稳。

---

## 5. 切回混合模式的步骤

1. **Windows**：NVIDIA 控制面板 → 显示 → 管理显示模式 → **自动选择 / 混合**
2. 重启 → 用**内置硬盘**启动（不必按 F7，除非默认引导被改成 U 盘）
3. 进 macOS 后确认：
   ```sh
   sysctl -n kern.bootargs | grep -c nvfb      # 期望 0（说明是内置 EFI）
   ioreg -l -w0 | grep -c de10                 # 期望 0（独显被屏蔽）
   ```
4. 分辨率切回 **1696×954 HiDPI**（`python3 display-bench.py --list` 可查）
5. U 盘 EFI 保留为「独显直连专用」，随时可切回


---

## 6. 开发者公告的硬性要求核对（2026-10-24 17:50）

公告来源：开发者群置顶（v1.0.0 发布 + v1.0.1 更新）

| # | 要求 | 本机 | 结论 |
| --- | --- | --- | --- |
| 1 | Lilu ≥ **1.7.0** | 1.7.2 | ✅ |
| 2 | 替换 NootedRed 为 v1.0.0 / v1.0.1 | **1.0.1** | ✅ |
| 3 | **去掉 bfixup.kext** | Kexts 里没有（只有 BlueToolFixup / IntelMKLFixup，属正常组件） | ✅ |
| 4 | **NootedRed.kext 放在 AppleALC.kext 之前** | 原本在 **第 6 位**（AppleALC 之后）❌ | **已修正到第 3 位（VirtualSMC 之后、AppleALC 之前）** |
| 5 | boot-args：`-NRedGLFix -NRedVDA -NRedVCNNoGfxOff` | 三项齐全 | ✅ |
| 6 | 用每个大版本靠后的小版本（如 15.8） | 15.8.1 | ✅ |

修正后的 `Kernel/Add` 顺序：

```text
[0] Lilu
[1] VirtualSMC
[2] NootedRed      ← 移到这里
[3] AppleALC
[4] SimpleHDA
[5] AppleMCEReporterDisabler
...
```

备份：ESP 根目录 `config-backup-1750.plist`；`plutil -lint` OK、`ocvalidate 1.0.8` → No issues found。

> v1.0.1 的更新点里提到「修复 15 系统下通知中心占用高」「增强用户空间补丁规则，降低占用、提升流畅度」——正好对应我们这台 15.8.1，换到混合模式后可以观察 `NotificationCenter` 的 CPU 占用是否正常。
