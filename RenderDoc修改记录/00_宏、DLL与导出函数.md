# 00：宏、DLL与导出函数——修改记录

日期：2026-09-29。根据 `git diff HEAD` 和当前 Windows 项目配置检查。

## 实际修改

| 文件与行号 | 修改前 | 修改后 |
| --- | --- | --- |
| `renderdoc/api/replay/renderdoc_replay.h:51` | `renderdoc__replay__marker()` | `zrender__replay__marker()` |

最初记录的是 `zrenderdoc`，你后来改成了 `zrender`，这里按现在的名字记录。

## 评价

这处改得没问题。不过，目前只改了导出这个函数的地方，DLL 查找它的地方还没改：

| 位置 | 当前使用的名字 |
| --- | --- |
| 使用该宏的回放程序导出 | `zrender__replay__marker` |
| 核心 DLL 查找 | `renderdoc__replay__marker` |

为什么 DLL 还在找旧名字？看 `renderdoc/os/win32/win32_libentry.cpp:53`：

```cpp
LibraryHooks::Detect(STRINGIZE(RDOC_BASE_NAME) "__replay__marker")
```

它用 `RDOC_BASE_NAME` 拼出完整名字。这个宏来自 `renderdoc/renderdoc.vcxproj:70`：

```text
RDOC_BASE_NAME=$(ProjectName)
```

同一个项目文件的第 26 行，`ProjectName` 现在还是 `renderdoc`，所以拼出来的仍是 `renderdoc__replay__marker`。

**也就是说，重新编译后，EXE 导出的新名字和 DLL 要找的名字对不上。** DLL 找不到这个标记，就不能靠它识别回放程序，可能走到捕获程序的初始化分支。至于是否崩溃，还要看实际运行。

你现在按文章分步修改，可以先记下这处。等后面改到核心 DLL 的项目名或相关配置时，再回来确认 DLL 查找的名字也变成了 `zrender__replay__marker`。`RDOC_BASE_NAME` 还用在其他 DLL 路径里，到时需要一起检查。

## 验证状态

- 已检查 Git 差异，以及 DLL 查找名称的来源。
- 还没编译、运行。
- 目前这行已改好，DLL 查找的名字还没跟上。

## 2026-09-29 全量复查

<div style="border-left: 5px solid #d32f2f; background-color: #ffebee; color: #8b1717; padding: 12px 16px; margin: 16px 0;"><strong>🔴 尚未解决：DLL 仍查找旧 marker</strong><br>头文件仍导出 zrender__replay__marker，win32_libentry.cpp:53 仍按 RDOC_BASE_NAME 拼接；renderdoc.vcxproj:26、70 仍给它 renderdoc。因此此前记录的问题还在，并不是这次保存丢失。</div>

## 2026-09-29 后续复查：查找名已一致

核心项目 `renderdoc.vcxproj:26` 的 ProjectName 已改为 zrender，RDOC_BASE_NAME 随之为 zrender。因此 win32_libentry.cpp 中拼出的查找名与头文件的 zrender__replay__marker 一致，之前的名称不匹配已解决。用户报告正常编译；回放分支尚未由我运行验证。
