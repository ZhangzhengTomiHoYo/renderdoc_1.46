# RenderDoc 学习：宏、DLL 与导出函数

## 1. 宏：编译前替换代码

`REPLAY_PROGRAM_MARKER()` 看起来像函数调用，实际上是宏。编译前，它会被替换成一个空函数的定义：

```cpp
extern "C" __declspec(dllexport) void __cdecl renderdoc__replay__marker()
{
}
```

这里记住三点：

- `extern "C"`：使用 C 的链接规则，避免 C++ 重载等机制带来的复杂函数名修饰。
- `__declspec(dllexport)`：把函数导出，让其他模块能按名字找到它。
- `__cdecl`：调用约定，规定函数如何传参等细节，初学时知道用途即可。

## 2. EXE 和 DLL：主程序与动态库

`qrenderdoc.exe` 是界面程序，`renderdoc.dll` 提供核心功能。DLL 被加载到进程中，与主程序一起运行。

**EXE 也可以导出函数。** 这里的 EXE 导出 marker，DLL 查找 marker，用来判断自己是否运行在回放程序中。

## 3. 空函数有什么用？当标记

这个函数不需要执行。RenderDoc 只检查它是否存在：

```text
找到标记 → 按回放程序初始化
没找到   → 通常按捕获程序初始化，注册 Hook
```

Windows 的 `GetProcAddress` 可以按名字查询导出函数的地址。当前实现会检查已加载的模块，不只检查 EXE。

所以，这个函数的作用是“让别人认出我”，不需要在函数体里做事。

## 4. 为什么改名后编译能过，运行却可能出错？

先记住程序生成和运行的顺序：

```text
源代码 → 编译成 .obj → 链接成 EXE/DLL → 加载运行
```

DLL 在运行时按字符串查找函数。假设 EXE 导出的是 `marker_A`，DLL 查找的却是 `marker_B`：

- 两边的代码都合法，可能正常编译、链接。
- 运行时名字对不上，DLL 找不到标记，就可能走错初始化分支。

**编译成功不等于运行正确。跨 EXE/DLL 使用的函数名，也是双方需要遵守的约定。** 改了相关头文件后，还要重新构建受影响的项目，避免新 DLL 配上旧 EXE。

文章说的“导致 UI 崩溃”是可能的后果；仅凭这段代码，只能确定名字不匹配会影响模式判断，不能确定必然崩溃。

## 对照源码

- `renderdoc/api/replay/renderdoc_replay.h:50`：宏定义。
- `qrenderdoc/Code/qrenderdoc.cpp:151`：EXE 使用宏，生成标记函数。
- `renderdoc/os/win32/win32_libentry.cpp:53`：DLL 根据标记选择初始化分支。
- `renderdoc/os/win32/win32_hook.cpp:1018`：按名字查找导出函数。

## 修改表（原文方案，按当前源码校正）

| 文件 | 原文行号 → 当前行号 | 修改类型 | 修改前 | 修改后 |
| --- | --- | --- | --- | --- |
| `renderdoc/api/replay/renderdoc_replay.h` | 52 → 51 | 标记函数改名 | `renderdoc__replay__marker()` | `rendertest__replay__marker()` |

**名称纠正：** 最初表格写的是 `rendertest`，不是后来口述的 `renderdoctext`。这里统一记录原表的名字，避免照着笔记修改时混用。

**不完整：不能只改这一行。** DLL 查找的名字来自 `STRINGIZE(RDOC_BASE_NAME) "__replay__marker"`，当前普通构建仍对应旧名。只有导出方和查找方一致，这项修改才配套。`RDOC_BASE_NAME` 还用于其他模块名称和路径，不能把它当成仅控制 marker 的开关随手改；配套方案等后文补齐。

**改后怎么验：** 重新生成受影响的宿主和 DLL，检查宿主导出表中的标记名称，再在 `LibraryHooks::Detect` 处确认查找的是同名符号、能够找到，并进入回放分支。仅编译通过或界面出现不算验证完成。

状态：已核对源码；修改方案缺配套，尚未实际修改或验证。
