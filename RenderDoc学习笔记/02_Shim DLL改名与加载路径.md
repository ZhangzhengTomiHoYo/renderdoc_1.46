# 02：Shim DLL 改名后，加载路径必须跟着改

## 和上一节有什么关系？

上一节改的是辅助 EXE 的启动路径，这一节改的是 Shim DLL 的加载路径。**不是改了 EXE 就必须改 DLL，而是两个文件各自改名后，各自的路径都要同步。**

这里的 Shim 是一个小型引导 DLL：检查当前进程是否符合条件，符合时再加载 RenderDoc 核心 DLL。它不是上一节的辅助 EXE，也不是核心 DLL 本身。

这几条路径位于 `Process::StartGlobalHook()`，属于全局 Hook 功能。同一函数也使用了上一节提到的辅助 EXE 路径。所以只验证界面能打开，并不能证明这里已改对。

## 修改表（按当前源码校正）

文件均为 `renderdoc/os/win32/win32_process.cpp`，类型均为路径字符串替换。

| 原文行号 → 当前行号 | 修改前 | 修改后 |
| --- | --- | --- |
| 1783 → 1519 | `"\\renderdocshim64.dll"` | `"\\rendertestshim64.dll"` |
| 1792 → 1528 | `"\\Win32\\Development\\renderdocshim32.dll"` | `"\\Win32\\Development\\rendertestshim32.dll"` |
| 1803 → 1539 | `"\\Win32\\Release\\renderdocshim32.dll"` | `"\\Win32\\Release\\rendertestshim32.dll"` |
| 1811 → 1547 | `"\\x86\\renderdocshim32.dll"` | `"\\x86\\rendertestshim32.dll"` |
| 1818 → 1554 | `"\\renderdocshim32.dll"` | 原文缺失；按前文命名推断为 `"\\rendertestshim32.dll"` |

## 还缺什么配套？

**表格只改“去哪里加载”，没有改“实际生成什么文件”。** 必须确认构建产物也叫 `rendertestshim32.dll`、`rendertestshim64.dll`，否则路径虽然改了，文件仍然找不到。

当前 `renderdocshim/renderdocshim.vcxproj` 的 `TargetName` 使用 `$(ProjectName)32`、`$(ProjectName)64` 决定输出基本名称。后文若涉及项目名或输出名修改，再把配套方案补齐；不要认为改了这里的字符串，输出文件就会自动改名。

另外，`StartGlobalHook()` 内的辅助 EXE 路径仍需与实际 EXE 一致，对应上一节待补的 1510、1529、1540、1548 行。

## 改后怎么验？

- 检查对应目录里是否真的有新名称、正确位数的 Shim DLL。
- 使用自己的测试程序验证全局 Hook 路径：确认加载了预期的 Shim，并成功加载核心 DLL。该 Shim 完成工作后会卸载，不能只在事后看模块列表判断。
- 哪个位数、配置实际验证过，就记录哪个；不能仅凭编译成功判定全部通过。

状态：已核对源码；最后一行是推断补全，输出名称的配套待补，尚未实际修改或运行验证。

关联第 06 节：Shim 加载后，通过命名共享内存读取辅助 EXE 提供的配置。文件路径正确只保证能找到模块，共享内存名称还必须在两端一致。

## 补充检查（2026-09-29）

你已把五条 Shim 路径统一为 `zrendershim32.dll`、`zrendershim64.dll`，并补齐相邻的四条辅助 EXE 路径。后四条是已贴原表未列、实际文件改名后不能漏掉的修改，详见修改记录 `01_硬编码路径与辅助进程.md（辅助 EXE）及 02_Shim DLL改名与加载路径.md（Shim）`。

<div style="border-left: 5px solid #d32f2f; background-color: #ffebee; color: #8b1717; padding: 12px 16px; margin: 16px 0;">
<strong>🔴 必须补：生成的 Shim DLL 名称也要一致</strong><br>
当前加载路径已改，renderdocshim/renderdocshim.vcxproj 的输出设置还没改。实际构建产物必须对应 zrendershim32.dll 和 zrendershim64.dll；仅修改路径不会自动生成这两个新名称的文件。后续改项目配置时完成。
</div>

## 当前磁盘状态更正（2026-09-29，最新）

上面的红色块保留的是当时检查到的中间状态。现在磁盘上的项目文件已经显式设置：

```xml
<ProjectName>zrendershim</ProjectName>
```

而四处输出表达式仍保持 `$(ProjectName)32` / `$(ProjectName)64`。MSBuild 在这里的求值链是：

```text
显式 ProjectName = zrendershim
        ↓
Win32 TargetName = zrendershim + 32
x64   TargetName = zrendershim + 64
        ↓
zrendershim32.dll / zrendershim64.dll
```

所以“加载路径”和“项目将生成的名称”在当前源码中已经配套。`RootNamespace` 也改成了 `zrendershim`，但控制上述输出名的是 `ProjectName` 与 `TargetName` 的组合，不能把作用归给 `RootNamespace`。

这只解决名称契约。全局 Hook 还涉及正确位数的核心 DLL、辅助 EXE、Shim、共享内存以及实际运行验证；不能仅凭项目属性正确就宣布全局 Hook 已通过。
