# 12：DX11 导出 Hook 与真实游戏捕获修复——修改记录

日期：2026-09-29。

源码基线：`2e32b910dddd44ee2a30d3afcfb1a16e8da90575`，分支 `v1.x`。仓库目录名是 `renderdoc_1.46`，本次运行日志中的实际版本为 RenderDoc 1.47。

## 结论

真实游戏最终使用的是 D3D11 呈现路径。此前 `zrender.dll` 注入和 Target Control 握手都成功，但原有 Windows Hook 没有覆盖游戏取得并缓存 D3D11/DXGI 入口地址的实际路径，因此 UI 一直显示 `API: None`。

最终需要保留的修复是：

- 只把两个 D3D11 创建设备入口和三个 DXGI factory 入口标记为 direct。
- 在目标进程内安全定位并修改这些导出的 EAT 函数 RVA。
- x64 目标过远时，通过 EAT 可表示范围内的 relay 跳转到 `zrender.dll` Hook。
- 在新启动目标的主线程恢复前，用远程工作线程预加载图形 DLL 并安装上述 Hook。
- 卸载 Hook 时恢复原始 RVA 并释放 relay。

排查阶段加入的 D3D12 delay-import 修补和 Target Control 周期轮询已经撤回。00～09 记录中的用户原始改名不属于本次清理范围，必须保留。

## 1. 修复前已经确认的事实

目标进程：

```text
D:\Program Files\Wuthering Waves\Wuthering Waves Game\Wuthering Waves.exe
D:\Program Files\Wuthering Waves\Wuthering Waves Game\Client\Binaries\Win64\Client-Win64-Shipping.exe
```

修复前日志已经证明：

- 父进程和真实游戏子进程均能建立控制连接。
- 子进程会注册 D3D11、D3D12、DXGI 等 Hook。
- 普通 D3D11 注入测试程序能够识别 API、触发捕获并回放。
- 真实游戏仍是 `API: None`。

这些证据把问题缩小到“真实游戏的图形取址/调用路径没有进入现有包装”，而不是核心 DLL 完全无法注入、控制网络断开或 D3D11 捕获实现整体损坏。

## 2. 根因应怎样表述

### 已被运行结果证实

原有 Hook 路径没有截获真实游戏的最终 D3D11/DXGI 创建与呈现路径；direct-export Hook 安装后，真实游戏被识别为 `D3D11 (Presenting & supported)`，并成功生成和回放捕获文件。

### 根据结果得到的高可信推断

游戏或其加载组件没有按普通导入槽调用这些入口，也没有经过原实现可替换结果的 `GetProcAddress` 路径，而是使用私有解析或等效方式从导出信息取得地址，并在较早阶段缓存。

### 没有直接证据支持的说法

- 不能仅凭游戏包含 ACE 文件就断言“ACE 拦截了 RenderDoc”。
- 没有逐指令跟踪取址调用者，不能把具体解析代码归因到某个确定模块。
- 这不是 SetThreadContext 注入修复；当前正常流程仍是挂起创建、注入 DLL、执行内部函数、恢复主线程。

## 3. 最终必须保留的源码修改

下表只列本次真实游戏修复，不重复 00～09 的 ZRender 改名内容。

| 文件 | 当前关键位置 | 必须保留的修改 | 原因 |
| --- | --- | --- | --- |
| `renderdoc/hooks/hooks.h` | `FunctionHook`、`HookedFunction::RegisterDirect` | 增加 `direct` 标志；普通 `Register` 保持原语义；新增显式 `RegisterDirect` | 只让经过选择的函数走 EAT 直接 Hook，不把所有 Hook 无差别扩大。 |
| `renderdoc/driver/d3d11/d3d11_hooks.cpp` | `D3D11Hook::RegisterHooks` 附近 | `D3D11CreateDevice`、`D3D11CreateDeviceAndSwapChain` 改用 `RegisterDirect` | 覆盖真实 D3D11 设备创建入口。 |
| `renderdoc/driver/dxgi/dxgi_hooks.cpp` | `DXGIHook::RegisterHooks` 附近 | `CreateDXGIFactory`、`CreateDXGIFactory1`、`CreateDXGIFactory2` 改用 `RegisterDirect` | 覆盖交换链所依赖的 DXGI factory 创建路径。 |
| `renderdoc/os/win32/win32_hook.cpp` | `DirectExportHook`、`FindExportAddressEntry`、`AllocateDirectExportRelay`、`ApplyDirectExportHook` | 验证 PE/内存范围；定位 EAT 项；保存原 RVA；必要时建立 relay；改写并记录 EAT 项 | 让不经过 IAT/标准动态取址的后续解析也得到 ZRender 包装入口。 |
| 同上 | `CachedHookData::RefreshDirectHooks`、`LibraryHooks::Refresh` | 在普通工作线程预加载 direct hook 所需 DLL，并安装 direct hooks | 避免在 loader lock 内主动加载图形 DLL；同时把安装提前到游戏首次取址之前。 |
| 同上 | `LibraryHooks::RemoveHooks` | 仅当表项仍等于本次安装值时恢复原 RVA，然后释放 relay | 避免卸载时覆盖第三方后来写入的未知值，并回收内存。 |
| `renderdoc/os/win32/win32_process.cpp` | `INTERNAL_InstallDirectHooks` | 新增导出内部函数，记录日志并调用 `LibraryHooks::Refresh()` | 允许注入控制方在目标进程的普通远程线程上安装 Hook。 |
| 同上 | `Process::InjectIntoProcess` | 设置捕获参数后、取得控制标识和恢复主线程前，调用 `INTERNAL_InstallDirectHooks` | 防止游戏主线程先解析并缓存未包装地址。 |

### 当前 direct 函数范围

```text
d3d11.dll!D3D11CreateDevice
d3d11.dll!D3D11CreateDeviceAndSwapChain
dxgi.dll!CreateDXGIFactory
dxgi.dll!CreateDXGIFactory1
dxgi.dll!CreateDXGIFactory2
```

没有把 DXGI 调试接口或其他图形 API 扩大为 direct。当前证据只要求上述五个入口。

## 4. 实现中的安全边界

`win32_hook.cpp` 的新增代码保留以下检查：

- 用 `VirtualQuery` 检查内存已提交、属于目标模块、不是 guard/no-access 区域。
- 校验 DOS/NT 签名、`SizeOfImage`、各 RVA 和数组长度，避免盲信损坏或异常 PE 数据。
- 导出名称最长读取 4096 字节，并要求字符串位于映像范围内。
- 修改 EAT 页前临时调用 `VirtualProtect`，之后恢复原保护。
- relay 写入后调用 `FlushInstructionCache`，再改为可执行只读。
- 已安装表项支持幂等检查；若表项不再等于原值或本次值，不覆盖未知第三方目标。
- 卸载时先确认表项仍属于同一模块且仍指向本次 Hook，再恢复。

原始函数地址在修改 EAT 之前通过 `GetProcAddress` 保存到 `orig`。D3D11/DXGI 包装函数因此能继续调用系统实现，而不是递归调用自己。

## 5. 已撤回的诊断性修改

### D3D12 delay-import 实验

曾在 `win32_hook.cpp` 中加入：

- PE delay import directory 解析。
- 按名称或序号匹配已登记 Hook。
- 校验 unload-IAT 后，修补尚未解析的 delay-IAT 槽。
- 必要时提前加载 `d3d12.dll`。
- `module not loaded`、`verified unresolved image thunk`、`hook installed` 等状态日志。

实验成功截获了序号 101 对应的 `D3D12CreateDevice`，看到了 Intel 和 WARP 设备，但它们创建后立即移除，UI 仍为 `API: None`。最终真实呈现 API 被日志确认为 D3D11，因此这组代码不属于最终修复，现已从源码删除。

### Target Control 周期轮询

曾在 Windows `TargetControlClientThread` 中反复调用 `LibraryHooks::Refresh()`：前 30 秒约每 100 ms，之后约每 1 秒，最长 5 分钟。它用于追踪 delay import 首次解析时机。

当前 `renderdoc/core/target_control.cpp` 已恢复到相对基线没有功能差异；最终方案只在目标主线程恢复前调用一次 `INTERNAL_InstallDirectHooks`，不再用控制线程长期轮询。

### 清理后的源码状态

- `win32_hook.cpp` 已不存在 `GetDelayImportDirectory`、`ApplyDelayHooks`、delay-import 状态表和主模块 delay-import 刷新。
- `target_control.cpp` 已不存在本次加入的轮询逻辑。
- D3D12 Hook 实现本身没有因为这次最终修复而扩大 direct 范围。

## 6. 成功运行所用 DLL 的身份

用户确认成功时使用：

```text
D:\renderdoc_1.46\x64\Release\zrender.dll
```

| 属性 | 值 |
| --- | --- |
| 修改时间 | `2026-09-29 21:42:52`（本机 Asia/Shanghai） |
| 大小 | `25,616,896` 字节 |
| SHA-256 | `65C20E7F47B2B5B484DEA79C14D26BC449487CE142A22CF3F87BAE55EF413E56` |
| 构建类型 | 日志显示 Windows 64-bit Release |
| 源码版本标识 | 日志显示 `2e32b910dddd44ee2a30d3afcfb1a16e8da90575` |

这个哈希用于标识“已经成功捕获真实游戏”的那一份 DLL。它当时仍含后来撤回的 D3D12 delay-import 诊断代码；因此不能把该哈希写成清理后源码的构建哈希。清理后的源码尚需重新构建，产物大小、时间和哈希理应变化。

界面 `zqrender.exe` 仍为 21:20 的时间并不矛盾，因为关键修复位于同目录加载的 `zrender.dll` 中，而不是 Qt 界面 EXE 中。

## 7. 成功日志证据

日志文件：

```text
C:\Users\zhang_GI\AppData\Local\Temp\ZRender\RenderDoc_2026.09.29_21.48.37.log
```

真实游戏子进程 PID 为 48004。关键证据按时间排列：

| 时间 | 日志摘要 | 结论 |
| --- | --- | --- |
| 21:48:48 | `Loading into ...Client-Win64-Shipping.exe` | 核心 DLL 进入真实游戏子进程。 |
| 21:48:48 | `Installing direct graphics hooks before target resume` | 安装发生在恢复目标主线程之前。 |
| 21:48:48 | `Preloaded d3d11.dll for direct export hooks` | D3D11 被提前加载，关闭首次取址竞态。 |
| 21:48:48 | 五条 `Installed direct export hook ...` | 两个 D3D11 和三个 DXGI 入口安装成功。 |
| 21:48:59 | `Got remote handshake: Client_Win64_Shipping [48004]` | 控制通道建立；这本身仍不等于捕获成功。 |
| 21:49:05 | 两个 D3D12 device frame capturer 添加后立即移除 | D3D12 是短暂探测，不是最终呈现路径。 |
| 21:49:07 | `New D3D11 device created: Intel ...` | 真实 D3D11 调用进入包装。 |
| 21:49:07 | `Adding D3D11 frame capturer ...` | 已识别 D3D11 交换链/帧捕获对象。 |
| 21:49:07 | `Used API: D3D11 (Presenting & supported)` | UI 不再是 `API: None`，真实呈现 API 已确认。 |
| 21:52:26 | `Starting capture` | 捕获实际开始。 |
| 21:52:27 | `Finished capture, Frame 808` | 第 808 帧完成。 |
| 21:52:29 | `Got a new capture ... 667749034 bytes` | UI 收到捕获文件。 |
| 21:54:48 | `Created replay driver.` | 捕获文件成功建立 D3D11 回放驱动。 |

日志中的 `win32_hook.cpp` 行号来自成功 DLL 构建时的源码；撤回 delay-import 代码后，当前文件行号已经前移，不能用旧日志行号判断当前函数位置。

## 8. 捕获文件证据

```text
C:\Users\zhang_GI\AppData\Local\Temp\RenderDoc\Wuthering Waves_2026.09.29_13.48.47_frame808.rdc
```

| 属性 | 值 |
| --- | --- |
| 文件大小 | `667,749,034` 字节 |
| SHA-256 | `E97C2FD8BD2AFC2ACFF137DB4D4F974C2BE7008AD9A7C339EC3F84D035F92762` |
| 捕获 API | D3D11 |
| 捕获适配器 | Intel UHD Graphics 730 |
| 回放状态 | 日志确认创建 replay driver，用户确认能够打开 |

该捕获文件写入旧的 `%TEMP%\RenderDoc` 子目录，不影响它证明 direct hook 成功。默认路径名称和已有配置、启动参数之间的关系属于原改版记录范围，不应与本次 API 根因混为一谈。

## 9. 没有修改游戏文件

本次源码写入均位于：

```text
D:\renderdoc_1.46
```

没有编辑、替换或删除 `D:\Program Files\Wuthering Waves\...` 下的 EXE、DLL、配置和 ACE 文件。运行时改变的是目标进程地址空间中已映射模块的 EAT 表项和新分配 relay 页面；目标进程退出后这些内存状态消失。

本次未事先为全部游戏文件保存一份基线哈希清单，因此“未改游戏文件”的依据是实际操作范围和实现行为，不应伪装成已完成全目录前后哈希比对。

## 10. 与用户原始改名的边界

以下内容虽然不是 `API: None` 的根因，但属于 00～09 已记录的 ZRender 人工改版，清理排错代码时必须保留：

- `zrender.dll`、`zqrender.exe`、`zrenderui.exe`、`zrendercmd.exe` 的输出和使用路径。
- `zrendershim32.dll` / `zrendershim64.dll` 的加载路径与 ZRender 共享内存名。
- replay marker、UI 自身排除、默认目录、OpenGL 窗口类、产品文字和启动链。
- 崩溃处理、更新和辅助程序中的新文件名引用。

当前磁盘的 `renderdocshim.vcxproj` 已显式设置 `ProjectName=zrendershim`，四个配置仍使用原有的 `$(ProjectName)32/64`，所以项目配置会派生出 `zrendershim32.dll`、`zrendershim64.dll`。第 02 篇文末“撤回 ProjectName 后重新输出旧名”的结论已被这次实际磁盘状态更新，后续应在第 02 篇追加更正，而不能继续把旧结论当作当前状态。全局 Hook 的端到端运行仍未因此自动验证。安装脚本注册/打包名仍按原记录单独处理；`renderdoc.rc` 的版权字符已在提交前恢复。它们不能因为真实游戏已经能捕获就写成全部解决，也不能为了缩小本次 direct-hook 修改而撤销用户原始改版。

## 11. 验证状态

### 已验证

- x64 Release 成功 DLL 能在真实游戏父、子进程中安装 direct hooks。
- 真实子进程识别为 `D3D11 (Presenting & supported)`。
- 第 808 帧完成捕获并生成 667,749,034 字节 RDC。
- RDC 能创建 D3D11 replay driver，用户确认成功。
- D3D12 短暂设备不是最终呈现路径。

### 源码已清理但尚待重新验证

- D3D12 delay-import 实验和 Target Control 轮询已从源码撤回。
- 清理后的 x64 Release 尚未在本记录中重新构建、计算新哈希并再次运行真实游戏。

因此当前准确表述是：**根因修复已经由包含 direct hooks 的成功 DLL 证明；最小化后的最终源码结构已经完成清理，但仍需一次重新构建和真实游戏回归，才能证明清理没有引入回退。**

### 尚未覆盖

- Win32 与跨位数启动。
- 全局 Hook 完整运行。
- D3D12、Vulkan、OpenGL 的真实游戏捕获。
- 其他受保护应用以及其他 Windows/D3D11/DXGI 版本。
- 安装包、更新、崩溃处理和注册表回退的端到端验证。
