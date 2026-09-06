# 《永劫无间》消除 CPU 降频与后台节流：禁用 GameDVR 与 PowerThrottling 实战

**核心排查结论：** 处理器明明具备睿频能力但在游戏对局中频率频繁从 5.0GHz 跌落至 3.2GHz 导致对战掉帧，根本原因并非处理器散热撞墙，而是 Windows 11 电源服务（PowerThrottling）误判后台伴侣进程优先级，加之 Xbox GameDVR 后台静默抓取录屏缓冲持续抢占 CPU 指令缓存，通过组策略与注册表彻底关闭 PowerThrottling 并清除 GameDVR 后台捕获机制，可锁定 CPU 满血最高主频。

---

## 一、 底层成因排查与参数对比表

| 系统特性 | 默认配置状态 | 对永劫无间的影响 | 电竞调优状态 | 预期优化效果 |
| --- | --- | --- | --- | --- |
| Power Throttling | 开启（节能省电） | 误把游戏辅助线程判为后台降频 | 彻底禁用 (Off) | CPU 频率平直锁定，无偶发掉帧 |
| Xbox GameDVR | 开启（自动缓冲录屏） | 持续写入磁盘并抢占 GPU 显存编码器 | 彻底禁用 (0) | 消除后台编码瓶颈，提升 10FPS |
| Game Bar | 开启 | 快捷键覆盖与前台焦点掠夺 | 完全关闭 | 避免误触弹窗导致游戏瞬间失焦 |

---

## 二、 核心排查与系统实操调优步骤

### 步骤 1：注册表彻底全局关闭 Windows 电源节流
以管理员身份打开 CMD，写入 PowerThrottling 停用注册表键值：

```cmd
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Power\PowerThrottling" /v PowerThrottlingOff /t REG_DWORD /d 1 /f
```

### 步骤 2：注册表清空 Xbox GameDVR 与后台录屏
以管理员身份打开 CMD，执行以下命令消除系统静默录屏开销：

```cmd
reg add "HKCU\System\GameConfigStore" /v GameDVR_Enabled /t REG_DWORD /d 0 /f
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\GameDVR" /v AppCaptureEnabled /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\GameDVR" /v AllowGameDVR /t REG_DWORD /d 0 /f
```

### 步骤 3：Windows 11 图形设置中指定高性能 GPU
打开系统设置 -> 屏幕 -> 显示卡 -> 自定义应用选项，添加《永劫无间》主程序路径，在图形首选项中明确勾选【高性能（锁定独立显卡）】并开启【可变刷新率】。

```powershell
# 检查 Windows 游戏模式状态（推荐保持开启）
Get-ItemProperty -Path "HKCU:\Software\Microsoft\GameBar" -Name "AutoGameModeEnabled" -ErrorAction SilentlyContinue
```


---

## 三、 常见故障排查与深度 FAQ

#### Q: 关掉 GameDVR 之后还想录制精彩高光怎么办？
> **A:** 建议使用显卡硬件驱动自带的影子录制功能（如 NVIDIA ShadowPlay 或 AMD Relive），它们直接调用显卡独立的 NVENC 硬件芯片，对游戏 CPU 与游戏帧率完全零损耗。

#### Q: 笔记本电脑关掉电源节流会不会大幅发热？
> **A:** 游戏运行时散热器本就处于全速运转状态，关闭节流能让风扇与供电策略更平稳地维持在最高频段，避免频繁“过热降频-降温升频”的恶性循环。


---
*本文由 746km.com 电竞技术研究室独家实测原创，定位于合规的电竞外设调试、掉帧排查与客户端崩溃修复。*
