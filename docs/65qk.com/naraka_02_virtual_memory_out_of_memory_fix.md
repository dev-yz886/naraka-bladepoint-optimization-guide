# 《永劫无间》对局中后期突发 Out of Memory 虚拟内存溢出与贴图丢失排查指南

**核心排查结论：** 在大战场决赛圈或长时间连续对局后突发无错误提示闪退或弹出 Out of Memory 提示，核心瓶颈在于 Unity 引擎材质流送（Texture Streaming）显存溢出后溢向系统物理内存，而 Windows 默认的“系统管理的分页文件”无法在瞬时高负载下完成动态扩容导致系统分页池耗尽崩溃，通过在系统高级设置中为游戏所在 SSD 固态盘分配 24576MB（24GB）固定虚拟内存，并在游戏配置文件中将材质流送池配额限制为 4096MB 可彻底杜绝内存溢出。

---

## 一、 底层成因排查与参数对比表

| 内存占用档位 | 物理内存配置 | 虚拟内存（Pagefile）推荐设定 | 材质流送池配额 | 崩溃风险评估 |
| --- | --- | --- | --- | --- |
| 16GB 内存机型 | 16GB DDR4/DDR5 | 自定义大小：初始 24576MB / 最大 24576MB | r.Streaming.PoolSize=3072 | 高风险（必须锁固定虚拟内存） |
| 32GB 内存机型 | 32GB DDR4/DDR5 | 自定义大小：初始 16384MB / 最大 16384MB | r.Streaming.PoolSize=4096 | 中低风险（长时间多局防溢出） |
| 64GB 极客机型 | 64GB DDR5 | 系统自动管理或锁定 8192MB | r.Streaming.PoolSize=6144 | 极低风险（物理内存充沛） |

---

## 二、 核心排查与系统实操调优步骤

### 步骤 1：PowerShell 自动化配置 24GB 固定虚拟内存页面文件
避免动态扩容带来的磁盘 I/O 阻塞与分配超时。以管理员身份打开 PowerShell，在高速 NVMe 固态盘上配置固定 24576MB 页面文件：

```powershell
$sysDrive = "C:"
Get-CimInstance Win32_ComputerSystem | Set-CimInstance -Property @{AutomaticManagedPagefile=$False}
$pagefile = Get-CimInstance Win32_PageFileSetting -Filter "SettingID='$sysDrive\\pagefile.sys'" -ErrorAction SilentlyContinue
if ($pagefile) { $pagefile | Remove-CimInstance }
New-CimInstance -ClassName Win32_PageFileSetting -Property @{Name="$sysDrive\pagefile.sys"; InitialSize=24576; MaximumSize=24576}
```

### 步骤 2：注册表解除连续分页内存池分配限制
修改系统内存管理核心参数，禁止系统把游戏驱动分页交换至低速临时交换区：

```cmd
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" /v DisablePagingExecutive /t REG_DWORD /d 1 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" /v LargeSystemCache /t REG_DWORD /d 0 /f
```

### 步骤 3：调整游戏内材质流送（Texture Streaming）显存配额
进入游戏设置 -> 画面设置，将【贴图质量】设为【中】或【高】（严禁无脑开超高），并在游戏运行时使用 Process Hacker 查看 NarakaBladepoint.exe 的 Private Bytes 保证平稳在 12GB 内。

```cmd
推荐实操：每连续激战 4~5 局后重启一次游戏客户端，强制回收 Unity 引擎残留未释放的贴图句柄资源。
```


---

## 三、 常见故障排查与深度 FAQ

#### Q: 物理内存已经有 32GB 了，为什么还会报 Out of Memory？
> **A:** 《永劫无间》使用的部分底层动态链接库和 DirectX 资源分配接口在申请连续虚拟地址空间（Virtual Address Space）时若遇到内存碎片，必须通过分页文件进行映射，单纯物理内存大并不代表连续地址池充足。

#### Q: 虚拟内存应该放在 C 盘还是游戏安装盘？
> **A:** 务必设置在读写速度最高的 NVMe 协议 M.2 固态硬盘分区，严禁设置在机械硬盘（HDD）上，否则会导致贴图加载严重泥泞甚至模型变黑。


---
*本文由 65qk.com 电竞技术研究室独家实测原创，定位于合规的电竞外设调试、掉帧排查与客户端崩溃修复。*
