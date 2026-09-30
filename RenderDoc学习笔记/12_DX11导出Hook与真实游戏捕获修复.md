# 12：DX11 导出 Hook 与真实游戏捕获修复

日期：2026-09-29。

这篇面向已经理解进程、线程、虚拟地址空间和 DLL，但还没有系统接触 Windows API Hook 的读者。先给结论：这次不是“没有把 `zrender.dll` 注入进去”，而是 **DLL 已经进入目标进程，控制连接也已建立，但游戏取得 DX11/DXGI 函数地址的路径没有经过原有拦截点**。最终修复是在目标进程恢复主线程之前，直接修改该进程内 `d3d11.dll`、`dxgi.dll` 的导出地址项，让后续从导出表取得的地址先进入 ZRender 包装函数。

本次没有修改游戏 EXE、游戏 DLL 或 ACE 文件。修改发生在 ZRender 源码和运行时目标进程的虚拟内存中。

## 1. 先分清已经成功和仍然失败的两层

失败时同时存在以下事实：

| 层次 | 当时状态 | 它能证明什么 |
| --- | --- | --- |
| 进程启动与 DLL 注入 | 成功 | `zrender.dll` 已进入目标进程并完成初始化。 |
| Target Control 握手 | 成功 | UI 与目标进程里的 ZRender 控制线程能够通信。 |
| 图形 API 注册 | 日志出现 D3D11、D3D12、DXGI hooks 注册 | Hook 定义已登记；不等于游戏的真实调用已经经过 Hook。 |
| 呈现 API 识别 | `API: None` | ZRender 仍没有看到可作为帧边界的真实呈现路径。 |
| 抓帧按钮 | 灰色 | 没有已识别的 presenting API，不能据此触发正常帧捕获。 |

因此，`Established` 和 `API: None` 并不矛盾。前者属于控制通道，后者属于图形调用拦截。

此前普通 D3D11 测试程序能够通过启动注入完成捕获，只能证明标准调用路径可用。真实游戏失败，说明两者在“如何得到图形函数地址”这一环节存在差异。

## 2. 调用 DLL 函数之前，程序必须先得到地址

CPU 最终只会跳到某个虚拟地址执行。源代码里的函数名只是链接和加载阶段使用的标识。Windows 程序常见的取址路径有三类：

| 取址方式 | 大致过程 | 常见拦截点 |
| --- | --- | --- |
| 静态导入 | 链接器生成导入信息，Windows 加载器把真实地址填入 IAT | 改写 IAT 表项 |
| 标准动态取址 | `LoadLibrary` 加载 DLL，再调用 `GetProcAddress` | 拦截 `LoadLibrary` / `GetProcAddress` |
| 自行解析导出 | 程序取得模块基址，自己解析 PE 导出目录并计算地址 | 前两种拦截都可能看不到，需要覆盖导出来源 |

取到地址后，程序通常会把它保存到函数指针中：

```cpp
using PFN_CreateDevice = HRESULT(WINAPI *)(/* 参数省略 */);
PFN_CreateDevice createDevice = /* 取得地址 */;

// 后面可能调用很多次，未必再次查询地址。
createDevice(/* 参数省略 */);
```

这就是“缓存函数地址”的含义：它不是磁盘缓存，也不是 CPU Cache，而是程序把某次解析得到的地址保存在自己的内存变量里。

## 3. IAT 是什么，IAT Hook 改了哪里？

PE 文件的导入信息描述“本模块需要哪些 DLL 的哪些函数”。模块装载后，Windows 加载器把解析出的绝对虚拟地址写入 Import Address Table，简称 IAT。

可以把一次静态导入调用简化为：

```text
游戏代码中的间接 call
        ↓
游戏模块 IAT 中的一个指针槽
        ↓
d3d11.dll!D3D11CreateDevice
```

IAT Hook 不必改写游戏的每条 `call` 指令，只要把那个指针槽换掉：

```text
游戏模块 IAT 指针槽
        ↓ 改写后
ZRender 的 D3D11CreateDevice_hook
        ↓ 包装、记录后继续调用
原始 D3D11CreateDevice
```

这种办法适合真正通过该 IAT 槽调用的代码。如果程序没有导入这个函数，或者后来把别处得到的地址保存在自己的私有函数指针里，修改 IAT 就碰不到那次调用。

## 4. `GetProcAddress` Hook 为什么也可能不够？

标准动态加载通常写成：

```cpp
HMODULE d3d11 = LoadLibraryW(L"d3d11.dll");
auto createDevice = (PFN_CreateDevice)GetProcAddress(d3d11, "D3D11CreateDevice");
```

RenderDoc 原有 Windows Hook 会覆盖这条常见路径：当程序调用 `GetProcAddress` 查询一个已登记的图形函数时，返回包装函数地址，而不让调用方直接拿到真实地址。

但 PE 的导出目录本来就位于已映射模块内。程序也可以不调用 `GetProcAddress`，而是自行完成以下步骤：

1. 取得 `d3d11.dll` 的模块基址。
2. 读取 DOS Header 和 NT Headers。
3. 定位 Export Directory。
4. 在导出名称表中找到 `D3D11CreateDevice`。
5. 通过名称序号定位函数地址表项。
6. 用模块基址加 RVA 得到函数虚拟地址。

这样既没有供 ZRender 修改的普通导入槽，也没有发生一次可被原有代码替换结果的 `GetProcAddress` 调用。

### 本次能够确定到什么程度？

**已证实的事实：** 原有注入、IAT 和 `GetProcAddress` 覆盖不足以让真实游戏进入 D3D11 呈现路径；启用导出表直接 Hook 后，同一游戏被识别为 D3D11，并成功捕获、回放。

**高可信推断：** 游戏或其保护/加载组件使用了自行解析导出表、等效的私有解析器，或其他绕过原拦截点的取址方式，并缓存了结果。

**尚未直接证明的细节：** 没有对取址调用者逐指令跟踪，所以不能断言具体是哪一个模块、哪段汇编或 ACE 本身执行了解析。游戏目录存在 ACE 不是“ACE 导致失败”的直接证据。

## 5. EAT、RVA 和函数地址之间是什么关系？

Export Address Table 常简称 EAT。严格说，PE 导出目录中有名称表、名称序号表和函数地址表；当前实现最后修改的是 `AddressOfFunctions` 指向的函数 RVA 表项。

RVA 是 Relative Virtual Address，即相对模块装载基址的偏移：

```text
函数虚拟地址 = 模块基址 + 函数 RVA
```

例如只为说明计算方式：

```text
d3d11.dll 基址 = 0x00007FFA00000000
表中 RVA        = 0x00123450
函数地址        = 0x00007FFA00123450
```

EAT 中的函数项是 32 位 `DWORD` RVA，不是任意 64 位绝对地址。这一点直接引出了 relay。

## 6. 这次“直接导出 Hook”实际改了什么？

当前实现没有覆盖 `D3D11CreateDevice` 开头的机器指令，因此不是传统意义上在函数开头写跳转的 inline hook。它做的是：

1. 验证目标模块的 PE Header、映像大小和相关内存范围。
2. 按导出名称找到函数 RVA 表项。
3. 在修改前用 `GetProcAddress` 保存原始函数地址。
4. 把 EAT 表项改为 ZRender Hook 的 RVA，或改为一个 relay 的 RVA。
5. 游戏以后再次从 EAT 解析该函数时，得到的是 ZRender 路径。
6. ZRender 包装完成后，仍可通过第 3 步保存的地址调用真实系统函数。

代码中的 `DirectExportHook` 保存：

- `module`：被修改的导出模块。
- `entry`：被修改的函数 RVA 表项地址。
- `originalRVA`：修改前的值。
- `hookRVA`：安装后的值。
- `relay`：必要时分配的中继代码页。

安装前还会检查表项是否已安装、是否仍是原值。若发现被别的目标替换，不会强行覆盖未知值。

## 7. 为什么需要 relay？

64 位进程中的函数指针是 64 位，但 EAT 函数项仍是相对当前模块基址的 32 位 RVA。`d3d11.dll` 和 `zrender.dll` 可能相距超过 4 GiB，此时不能直接把 ZRender Hook 地址减去 `d3d11.dll` 基址后塞进一个 `DWORD`。

当前方案在目标 DLL 基址之后、32 位 RVA 能表示的范围内寻找空闲页，放置一个很短的 relay：

```text
EAT 表项
   ↓ 32 位 RVA 可以到达
附近的 relay
   ↓ relay 内保存完整 64 位地址并跳转
zrender.dll 中的 Hook 函数
```

x64 relay 的逻辑是：

```text
endbr64
jmp qword ptr [rip]
<紧随其后的 64 位 Hook 地址>
```

`endbr64` 用于兼容启用硬件间接分支跟踪的目标；RIP 相对间接跳转不需要先占用一个通用寄存器。写完代码后会刷新指令缓存，并把页面从可写改为可执行只读。

在 32 位构建中，Hook 地址本身能装进 32 位立即数，relay 使用 `mov eax, hook; jmp eax`。

这里的 relay 只负责“把 EAT 能表示的近地址转到真正的远地址”，并不是保存被覆盖函数开头指令的传统 trampoline。

## 8. 为什么安装时机必须早于游戏主线程恢复？

假设游戏只在启动时解析一次地址：

```text
先解析真实地址并缓存 → 后安装 EAT Hook → 已缓存指针仍指向真实函数
```

修改 EAT 只影响之后的解析，不会自动追踪并改写游戏已经保存到任意位置的所有指针。因此最终流程是：

```text
CREATE_SUSPENDED 创建目标进程
        ↓
注入 zrender.dll，并等待其初始化
        ↓
设置捕获参数
        ↓
在目标进程创建远程工作线程
调用 INTERNAL_InstallDirectHooks
        ↓
预加载 d3d11.dll、dxgi.dll
安装选定导出项的直接 Hook
        ↓
取得 Target Control 标识
        ↓
恢复目标主线程
        ↓
游戏第一次解析并缓存图形入口时，看到的是 Hook 后的地址
```

`INTERNAL_InstallDirectHooks` 在目标进程内调用 `LibraryHooks::Refresh()`。当前 Windows 的 `Refresh()` 只负责这组 direct hooks，不再承担之前实验性的周期扫描。

把安装放到远程工作线程而不是 DLL 入口回调中，还有一个原因：DLL 初始化阶段可能持有 Windows loader lock。在该阶段再次主动加载 `d3d11.dll`、`dxgi.dll` 会带来重入和死锁风险；普通工作线程上执行更合适。

## 9. 为什么 D3D11 和 DXGI 都要覆盖？

最终标记为 direct 的入口只有五个：

| DLL | 函数 | 作用 |
| --- | --- | --- |
| `d3d11.dll` | `D3D11CreateDevice` | 创建 D3D11 设备和上下文，不直接创建交换链。 |
| `d3d11.dll` | `D3D11CreateDeviceAndSwapChain` | 一次创建 D3D11 设备、上下文和交换链。 |
| `dxgi.dll` | `CreateDXGIFactory` | 创建较早版本的 DXGI factory。 |
| `dxgi.dll` | `CreateDXGIFactory1` | 创建 DXGI 1.x factory。 |
| `dxgi.dll` | `CreateDXGIFactory2` | 创建支持新选项的 DXGI factory。 |

D3D11 负责设备和命令执行环境；DXGI 负责适配器、输出、交换链等。RenderDoc 不只需要看到“设备被创建”，还要跟踪用于显示画面的交换链和 `Present`，才能建立帧边界并让 UI 进入可抓帧状态。

DXGI 的调试接口仍使用原有普通注册方式，因为这次真实问题对应的是设备与 factory 创建入口，没有证据要求扩大 direct hook 范围。

## 10. 成功日志形成了怎样的证据链？

成功运行日志：

```text
C:\Users\zhang_GI\AppData\Local\Temp\ZRender\RenderDoc_2026.09.29_21.48.37.log
```

关键顺序如下：

```text
Loading into ...\Client-Win64-Shipping.exe
Installing direct graphics hooks before target resume
Preloaded d3d11.dll for direct export hooks
Installed direct export hook for D3D11CreateDevice
Installed direct export hook for D3D11CreateDeviceAndSwapChain
Installed direct export hook for CreateDXGIFactory
Installed direct export hook for CreateDXGIFactory1
Installed direct export hook for CreateDXGIFactory2
Got remote handshake: Client_Win64_Shipping [48004]
New D3D11 device created: Intel / Intel(R) UHD Graphics 730 ...
Adding D3D11 frame capturer ...
Used API: D3D11 (Presenting & supported)
Starting capture
Finished capture, Frame 808
Got a new capture: 0 (frame 808) (667749034 bytes) ...
Created replay driver.
```

各行证据强度不同：

| 日志 | 能证明 | 不能单独证明 |
| --- | --- | --- |
| `Loading into` | 核心 DLL 已在目标进程初始化 | 图形调用一定已被拦截 |
| `Installed direct export hook` | 五个 EAT 项的修改函数返回成功 | 游戏一定已经调用这些入口 |
| `New D3D11 device created` | D3D11 创建调用已经进入包装代码 | 该设备一定用于最终呈现 |
| `Adding D3D11 frame capturer` + `Used API ... Presenting` | ZRender 已看到可呈现的 D3D11 路径 | 尚未证明捕获文件可回放 |
| `Finished capture` + `Got a new capture` | 第 808 帧已完成并交给 UI | 尚未证明回放初始化成功 |
| `Created replay driver` | 该捕获文件已成功建立 D3D11 回放驱动 | 不代表所有事件分析功能都已逐项测试 |

因此最终结论不是建立在“界面看起来正常”上，而是从安装、真实 API、Present、落盘到回放的一整条证据链。

## 11. 为什么前面的 D3D12 delay-import 修改被撤回？

排查中曾发现实际游戏 EXE 对 `d3d12.dll` 使用 delay import，并按序号导入 101、102。实验代码解析 delay-import descriptor，在首次调用前修补尚未解析的 IAT，并由 Target Control 线程短期轮询刷新。

它确实取得了新证据：

- `D3D12CreateDevice` 进入了包装代码。
- Intel D3D12 设备和 Microsoft Basic Render Driver 设备都曾创建。
- 两个 device frame capturer 随后立即移除。
- 界面仍保持 `API: None`。

这说明那些 D3D12 设备是短暂探测路径，不能代表游戏主画面采用 D3D12。之后真实成功日志明确出现：

```text
Used API: D3D11 (Presenting & supported)
```

所以 delay-import 代码完成了诊断任务，但不是最终修复所需。当前源码已经撤回：

- D3D12 delay-import descriptor 扫描和未解析 IAT 修补。
- 主模块 delay-import 状态日志。
- Target Control 中 100 ms / 1 s 的周期 `LibraryHooks::Refresh()` 轮询。

最终只保留启动前的一次 direct-export 安装。这样既缩小修改范围，也避免让一次诊断实验永久扩大所有目标程序的加载和轮询行为。

## 12. 是否修改了游戏代码或绕过了反作弊？

没有修改游戏磁盘文件，也没有修改游戏源码。实际过程是：

1. ZRender 正常创建目标进程并将其主线程保持挂起。
2. 把 `zrender.dll` 加载到目标进程地址空间。
3. 由目标进程中的 ZRender 代码修改该进程已映射的系统图形 DLL 导出表项。
4. 游戏退出后，该进程的整个虚拟地址空间被操作系统销毁，这些内存修改随之消失。

本次没有关闭、修改或卸载 ACE，也没有实现针对 ACE 的规避逻辑。“保护型应用可能采用私有解析路径”是对调用路径的技术描述，不等于已证明具体由 ACE 实现。

## 13. 当前边界和后续验证

- 已成功运行的是 x64 Release、D3D11、Intel UHD Graphics 730 和当前游戏版本。
- 未由这次成功证明：Win32、D3D12 主渲染、Vulkan、OpenGL、全局 Hook、其他游戏以及未来系统 DLL 版本。
- direct-export 方案会提前加载 `d3d11.dll`、`dxgi.dll`，改变被捕获进程很早期的模块集合与部分初始化时序，需要用其他 D3D11 程序做回归测试。
- 成功捕获所用 DLL 当时仍包含已撤回的 D3D12 诊断代码。清理后的源码需要重新构建并对真实游戏至少再做一次 API 识别、抓帧和回放验证，才能把“最小最终版本已运行验证”写成完成。

## 14. 最后用一句技术上准确的话概括

原版 Windows Hook 能覆盖常见 IAT 和 `GetProcAddress` 取址，但这款游戏的实际 DX11/DXGI 入口取址没有被它覆盖；ZRender 因此在目标主线程运行前，进程内修改五个图形导出的 EAT 函数 RVA，并在需要时借助近地址 relay 跳转到包装函数，使游戏第一次取得的地址就进入 ZRender，最终识别 D3D11 Present 并成功捕获、回放。
