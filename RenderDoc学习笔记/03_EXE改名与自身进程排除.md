# 03：EXE 改名后，自身进程的排除判断也要改

## 为什么前面改完，这里还要改？

第 01 节解决的是“找到并启动改名后的辅助 EXE”。这里解决的是：**不要把 RenderDoc 自己的程序当成需要捕获的子进程。**

`ShouldInject()` 在开启 `hookIntoChildren`（捕获子进程）时，检查将要启动的程序。如果是 RenderDoc 的命令行程序或界面程序，就返回 `false`，不向它注入。

```text
辅助 EXE 或界面 EXE 改名
    ↓
排除判断仍然只认旧名字
    ↓
改名后的自身进程可能不再被排除
    ↓
可能误向自身程序注入，甚至形成递归链路
```

源码注释明确说明，这个判断用于防止向自身反复注入。**这里应跟随实际 EXE 文件名修改，与 marker 符号或 Shim DLL 名称无直接对应关系。**

## 修改表（按当前源码校正）

文件：`renderdoc/os/win32/sys_win32_hooks.cpp`。类型：自身进程排除条件中的字符串替换。原文复制时缺了部分表达式，下面补成完整判断条件。

| 原文行号 → 当前行号 | 修改前 | 修改后 |
| --- | --- | --- |
| 342 → 340 | `app.contains("renderdoccmd.exe") \|\| app.contains("qrenderdoc.exe")` | `app.contains("rendertestcmd.exe") \|\| app.contains("qrendertest.exe")` |
| 351 → 349 | `cmd.contains("renderdoccmd.exe") \|\| cmd.contains("qrenderdoc.exe")` | `cmd.contains("rendertestcmd.exe") \|\| cmd.contains("qrendertest.exe")` |

保留原来的 `if`、花括号和 `inject = false;`，表中仅列条件。

## 为什么要改两处？

进程创建时，程序名可能通过 `lpApplicationName` 提供，也可能放在 `lpCommandLine` 中。因此两处都要检查，只改一处可能漏掉另一种启动方式。

## 配套与验证

**待补：** 本段首次出现界面程序的新名称 `qrendertest.exe`，但目前材料还没有给出实际输出名称的修改。如果界面程序仍叫 `qrenderdoc.exe`，就不能提前只保留新名字，否则旧名字反而失去排除保护。辅助 EXE 同理。

改后在自己的测试环境中检查 `ShouldInject()`：开启 `hookIntoChildren`，分别以应用路径、命令行提供新 EXE 名称，应返回 `false`；普通测试子程序应仍允许进入后续注入流程。关闭该选项时会直接返回 `false`，不能拿这个结果证明名称判断已改对。

状态：已核对源码和完整条件；EXE 输出名称的配套待补，尚未实际修改或运行验证。

关联第 04 节：界面 EXE 改名后，除了本节的自身进程排除条件，还需同步 `GetReplayAppFilename()` 的界面查找路径。注册表回退需要另行与注册信息配套。
