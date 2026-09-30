# 03：EXE改名与自身进程排除——修改记录

日期：2026-09-29。根据 `git diff HEAD` 检查。

## 实际修改

文件：`renderdoc/os/win32/sys_win32_hooks.cpp`。

| 行号 | 修改前 | 修改后 |
| --- | --- | --- |
| 340 | `app.contains("renderdoccmd.exe") \|\| app.contains("qrenderdoc.exe")` | `app.contains("zrendercmd.exe") \|\| app.contains("zqrender.exe")` |
| 349 | `cmd.contains("renderdoccmd.exe") \|\| cmd.contains("qrenderdoc.exe")` | `cmd.contains("zrendercmd.exe") \|\| cmd.contains("zqrender.exe")` |

## 评价

**两处都改到了，名字也一致。** `zrendercmd.exe` 与你第 01 篇的路径修改相同。界面程序这次用的是 `zqrender.exe`，后面检查第 04、08 篇和界面项目输出时，我会按这个名字核对，不会按文章的 `qrendertest.exe`。

原来的 `inject = false` 和判断结构都保留了。开启捕获子进程后，只要应用路径或命令行包含这两个新名字，就会被排除，不向它们注入。第 03 篇这两行没有发现改错的地方。

但目前项目还会生成旧名字的 EXE，界面的查找和启动路径也还没改。你现在是分步修改，先把这个差异记住，完成后面的步骤再一起运行。

<div style="border-left: 5px solid #d32f2f; background-color: #ffebee; color: #8b1717; padding: 12px 16px; margin: 16px 0;">
<strong>🔴 必须补：实际 EXE 名称还没跟上</strong><br>
本节表格只改了排除条件，没有改构建输出。renderdoccmd/renderdoccmd.vcxproj:25 仍使用 renderdoccmd；qrenderdoc/qrenderdoc_local.vcxproj:43 的 TargetName 仍是 qrenderdoc。后面要让实际输出对应 zrendercmd.exe、zqrender.exe。<br><br>
如果仍运行旧名称的 EXE，这两条条件不会再因为旧文件名而排除它们。另请在第 04、08 篇把界面查找和启动路径统一为 zqrender.exe，否则会找错文件。
</div>

这次没有发现原表之外还需新增的排除条件。上面的提醒是与其他步骤的关联，不是要求再额外改两条判断。

## 验证状态

- 已核对两处 Git 修改及当前项目输出名称。
- 未编译、运行。等 EXE 输出名改好后，再验证开启捕获子进程时，这两个程序都能被正确排除。

## 2026-09-29 全量复查

两条排除条件仍为 zrendercmd.exe、zqrender.exe。界面输出和启动路径已对应；辅助 EXE 输出仍未改。没有发现已改内容丢失。
