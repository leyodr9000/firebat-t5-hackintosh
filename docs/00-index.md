# 文件索引

| 文件 | 内容 |
| --- | --- |
| `HANDOFF.md` | 硬件/系统/卷/生效配置/变更记录/已知问题/命令/测量数据 |
| `hackintosh-status-2026-10-24.md` | 系统状态报告：平台、引导、kext、图形、崩溃、关机卡死 |
| `nvidia-dgpu-efi-evaluation.md` | NullMoth 2.5.3 实测：配置要求、混合/直连数据、硬解、性能对比 |
| `nvidia-dgpu-guide.md` | 独显直连操作手册：OpenCore 配置、U 盘 EFI、MUX 切换、驱动安装、故障对照、回滚 |
| `tuning.md` | 调优记录：显示模式/HiDPI、刷新验证、合成开销、电源、kext 顺序、boot-args 取舍 |
| `hybrid-mode-tuning.md` | NootedRed 混合模式基线：boot-args、硬性要求、可选项、切换步骤 |
| `brightness-fix-tech-doc.md` | 背光控制链路、UI↔linear 实测曲线、驱动补丁、构建与部署 |
| `display-bench.py` | 显示模式负载基准：`--list` / 默认 / `--sweep` |
| `efi-config-before-20261024.plist` | boot-args 修改前快照 |
| `efi-bootargs-before.txt` | 修改前 boot-args 原值 |
| `EFI-独显直连-fixed/` | 独显直连 EFI（ocvalidate 通过）+ `修复说明.md` |
| `usb-efi-backup-20261024-1514/` | U 盘原始 EFI 备份（56 MB / 166 文件） |
| `brightness-daemon-backup/` | 已弃用的亮度守护进程文件（LaunchAgent + 脚本） |
| `NootedRed.kext.orig-1.0.1/` | 官方原版 NootedRed kext |
| `nootedred-backlight-floor.patch` | 背光补丁参考 |

## 操作约定

1. 改动 ESP 前整份备份到 ESP 根目录 `EFI-backup-YYYYMMDD-HHMM`。
2. 修改 HiDPI 使用 `CGBeginDisplayConfiguration` 流程（`CGDisplaySetDisplayMode` 对 HiDPI 返回 1000）。
3. 判断 SubmitGuard 状态看 `NRedSubmitGuardApplied` 的变化，不看 `Enabled`。
4. 定位 ESP 按容量过滤，disk 编号每次启动会变。

## 外部路径

| 路径 | 内容 |
| --- | --- |
| `/Volumes/DATA-TEMP/ubac/20240314-OC-EFI项目/EFI升级_20261022/` | 历史 EFI 变体、升级报告、N 卡驱动包（Windows 盘只读挂载） |
| `/Volumes/STAGE/nrd-src/` | NootedRed 源码与二进制 |
| `/Volumes/STAGE/panic/` | panic 与内核日志归档 |
| `~/nrd-build/` | xmake + AppleForge 构建环境 |
