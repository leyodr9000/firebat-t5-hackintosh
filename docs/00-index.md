# Workspace 索引 — Firebat T5 黑苹果

> 给后续接手的 AI / 人：**先读 [HANDOFF.md](HANDOFF.md)**，那里是唯一权威的"现状 + 变更 + 待办"。
> 最后更新：2026-10-24 15:10 CST

## 一句话现状

macOS 15.8.1（OC 1.0.8 + NootedRed 1.0.1）运行正常，自 10/24 12:55 重启后无 panic；
本次会话已把 `-NRedNoSubmitGuard` 从 boot-args 移除（guard 现为开启，`Applied=4702`），
并把显示缩放从 200% 改到 **150.9%（1696×954 HiDPI @240Hz）**。

## 文件

| 文件 | 用途 |
| --- | --- |
| [HANDOFF.md](HANDOFF.md) | **交接总览**：环境指纹、时间线、变更清单、事实/推断/未知、踩坑、待办、命令速查 |
| [hackintosh-status-2026-10-24.md](hackintosh-status-2026-10-24.md) | 体检报告：平台/kext/GPU-VCN/6 次 panic/根因分析/EFI 核对/变更记录（§4.4–4.7） |
| [nvidia-dgpu-efi-evaluation.md](nvidia-dgpu-efi-evaluation.md) | N 卡（NullMoth 2.5.3）驱动能力 + `EFI-独显直连` 方案评估（含 2 处硬冲突） |
| [display-bench.py](display-bench.py) | 显示模式负载基准：`--list` / 直接跑 / `--sweep` |
| [efi-config-before-20261024.plist](efi-config-before-20261024.plist) | 改 boot-args 前的 config 快照（回滚用） |
| [efi-bootargs-before.txt](efi-bootargs-before.txt) | 改之前的 boot-args 原值 |
| `EFI-独显直连-fixed/` | 已修复的 N 卡试验 EFI（两份 config 各 2 处修复，ocvalidate 通过）+ `修复说明.md` |
| `usb-efi-backup-20261024-1514/` | U 盘原始 EFI 的完整备份（56 MB / 166 文件）；同内容 zip 也放在 U 盘 ESP 根目录 |

## 三条硬规则

1. 动 ESP 前先整份备份（`EFI-backup-YYYYMMDD-HHMM` 放 ESP 根目录）。
2. 不要用手搓的 `CGDisplaySetDisplayMode` 改 HiDPI（返回 1000），用 `display-bench.py` 里的 configuration API 写法。
3. 判断 NootedRed 状态看 `NRedSubmitGuardApplied` 的**变化**，不要看 `Enabled`。

## 外部资源

- `D:\ubac\20240314-OC-EFI项目\EFI升级_20261022\`（Windows 盘，只读挂载于 `/Volumes/DATA-TEMP`）：历史 EFI 变体、升级报告、N 卡驱动包
- `/Volumes/STAGE/nrd-src/`（macOS 侧）：NootedRed 1.0.1 源码与二进制
- `/Volumes/STAGE/panic/`：用户归档的 panic 与内核日志
- [`brightness-fix-tech-doc.md`](brightness-fix-tech-doc.md) — 亮度黑屏问题的完整技术复盘（曲线实测、三种失败方案、自编译 NootedRed 补丁、构建与回滚）
