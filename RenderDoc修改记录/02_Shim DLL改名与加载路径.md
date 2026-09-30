# 02：Shim DLL改名与加载路径——修改记录

日期：2026-09-29。文件：renderdoc/os/win32/win32_process.cpp。

## 实际修改

| 行号 | 修改前 | 修改后 |
| --- | --- | --- |
| 1519 | `"\\renderdocshim64.dll"` | `"\\zrendershim64.dll"` |
| 1528 | `"\\Win32\\Development\\renderdocshim32.dll"` | `"\\Win32\\Development\\zrendershim32.dll"` |
| 1539 | `"\\Win32\\Release\\renderdocshim32.dll"` | `"\\Win32\\Release\\zrendershim32.dll"` |
| 1547 | `"\\x86\\renderdocshim32.dll"` | `"\\x86\\zrendershim32.dll"` |
| 1554 | `"\\renderdocshim32.dll"` | `"\\zrendershim32.dll"` |

这五处也都改对了，32、64 位没有混用。它们本来就在你贴的 Shim 表中，不属于这次发现的文章遗漏。


## 评价

这五条路径都已改对。相邻的四条辅助 EXE 路径也已补齐，详情放在同目录的 01_硬编码路径与辅助进程.md 中，属于原表未列但改名后必须同步的地方。

<div style="border-left: 5px solid #d32f2f; background-color: #ffebee; color: #8b1717; padding: 12px 16px; margin: 16px 0;">
<strong>🔴 必须补：实际输出的 Shim DLL 名称还没改</strong><br>
renderdocshim/renderdocshim.vcxproj:73、78、83、88 的 TargetName 仍按原项目名生成文件。路径已经使用 zrendershim32.dll、zrendershim64.dll，实际输出也必须对应，否则找不到文件。后续修改项目配置时补上。
</div>

## 验证状态

- 已核对五条路径，32、64 位没有混用。
- Git 差异的空白格式检查通过。
- 未编译、运行。

## 2026-09-29 全量复查

<div style="border-left: 5px solid #d32f2f; background-color: #ffebee; color: #8b1717; padding: 12px 16px; margin: 16px 0;"><strong>🔴 尚未解决：Shim 输出名还是旧名</strong><br>五条新路径仍完整保留。renderdocshim.vcxproj:73、78、83、88 的 TargetName 仍使用原项目名加 32/64，未生成与路径对应的新名字。这是之前未补的项目配置，不是修改回退。</div>

## 2026-09-29 用户授权修复：Shim 输出名已配套并实际构建

本次修改 renderdocshim/renderdocshim.vcxproj 的四处 TargetName：Development/Release 的 Win32 配置输出 zrendershim32，x64 配置输出 zrendershim64。直接设置输出名，保留项目名和中间目录，避免依赖 $(ProjectName) 的默认值。

实际构建验证：
- x64/Development/zrendershim64.dll 已生成。
- Win32/Development/zrendershim32.dll 已生成。
- 项目 XML 可解析，针对该项目的 git diff --check 通过。

此前“Shim 输出名称未配套”已解决，历史提醒保留。Release 配置只核对配置，未构建；没有启用全局 Hook 或进行目标游戏验证。x64/Development/zrendercmd.exe 也已生成，详见第 01 篇后续记录。Win32 辅助 EXE 及核心 DLL 尚未构建，因此不能宣称全局 Hook 所有位数组件已配齐。

## 2026-09-29 按用户反馈调整：恢复 ProjectName 派生输出名

上一节直接设置四处 TargetName 的方案已撤换。用户指出应保留原有 $(ProjectName)32/64 的派生方式。MSBuild 实际查询确认，修改前 Shim 的 ProjectName 为 renderdocshim，并未随核心项目改名而成为 zrendershim。

当前最终修改：在 Globals 中增加 <ProjectName>zrendershim</ProjectName>；四处 TargetName 恢复原有 $(ProjectName)32/64。这样统一控制名称，避免四处重复写死。RootNamespace 保持原值，它不是控制此输出名称的属性。

重新构建 Development/x64 和 Development/Win32 均成功，分别生成 zrendershim64.dll、zrendershim32.dll。之前写死 TargetName 的做法并非编译错误，但本次采用用户要求的项目命名机制。历史记录保留，以本节最终配置为准。

## 2026-09-29 项目名称属性统一

按用户反馈，将 RootNamespace 也由 renderdocshim 改为 zrendershim，与 ProjectName 保持一致；四处 TargetName 继续保持原来的 $(ProjectName)32/64。MSBuild 属性求值确认 RootNamespace=zrendershim、ProjectName=zrendershim、x64 TargetName=zrendershim64。本次仅补齐命名属性，未把 RootNamespace 描述为生成文件名的控制项，未重复构建。

## 2026-09-29 追踪默认来源，撤回新增 ProjectName

用户指出原项目没有显式 ProjectName，应先检查导入来源。本机 Microsoft.Cpp.Default.props:217 定义：ProjectName 为空时取 MSBuildProjectName（项目文件名不含扩展名）。已撤回此前新增的 ProjectName 行，保留 RootNamespace=zrendershim 与原 TargetName 表达式。

撤回后的 MSBuild 实际求值：MSBuildProjectName=renderdocshim，ProjectName=renderdocshim，RootNamespace=zrendershim，TargetName=renderdocshim64。因此当前配置再次默认输出旧名；磁盘此前生成的新名 DLL 仍保留，但不能把它当成此配置以后会生成新名的证据。此次没有重编译或改名项目文件，前述已配套结论不再适用于当前最终配置。

## 2026-09-29 当前磁盘状态更正（最新）

本次按用户要求重新读取磁盘，而不是继续沿用上一节描述。当前 `renderdocshim/renderdocshim.vcxproj` 的 `Globals` 中实际同时存在：

```xml
<RootNamespace>zrendershim</RootNamespace>
<ProjectName>zrendershim</ProjectName>
```

四个配置的 `TargetName` 仍是原来的 `$(ProjectName)32` / `$(ProjectName)64`。因此当前源码配置会派生 `zrendershim32.dll`、`zrendershim64.dll`，与 `win32_process.cpp` 中的加载路径一致。上一节“已撤回 ProjectName、当前重新输出旧名”只描述当时的中间状态，已经被当前实际文件推翻，以本节为准。

这项 `ProjectName` 不是本次真实游戏 `API: None` 的根因，但它是 ZRender 改名后保持全局 Hook 文件名配套所必需的配置，因此保留。本次只做静态核对，没有重新编译或运行全局 Hook，不能把“名称配置已配套”扩大成“全局 Hook 已验证”。
