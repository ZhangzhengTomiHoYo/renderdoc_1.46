# 07：DLL文件描述与产品名称——修改记录

日期：2026-09-29。用 `git diff --text HEAD` 检查，并对照了版权行的原始字节。

## 实际修改

文件：`renderdoc/data/renderdoc.rc`。

| 行号 | 修改前 | 修改后 |
| --- | --- | --- |
| 87 | `VALUE "FileDescription", "Core DLL for RenderDoc"` | `VALUE "FileDescription", "Core DLL for ZRender"` |
| 92 | `VALUE "ProductName", "RenderDoc"` | `VALUE "ProductName", "ZRender"` |
| 90（额外变化） | `VALUE "LegalCopyright", "Copyright © 2026 Baldur Karlsson"` | `VALUE "LegalCopyright", "Copyright ?2026 Baldur Karlsson"` |

## 评价

**第 07 篇要求的两项名称都改对了。** 但版权行也变了：原来的 `©` 和它后面的空格被替换成了一个 `?`。这不是本节要做的修改，应该恢复。

我比较了原始字节，原来这一段是 `A9 20`，现在是 `3F`，所以确实改到了文件内容，不只是终端显示乱码。具体是编辑时误改还是保存时转换编码造成的，仅凭差异不能确定。

<div style="border-left: 5px solid #777777; background-color: #f5f5f5; color: #333333; padding: 12px 16px; margin: 16px 0;">
<strong>已修复（见文末复查）：90 行版权文字曾被改坏</strong><br>
把这一行恢复为原来的内容：<br>
<code>VALUE "LegalCopyright", "Copyright © 2026 Baldur Karlsson"</code><br><br>
文件声明使用 code_page(1252)，保存时保持 Windows-1252 编码。不要只把 © 打回去，却以另一种编码保存，导致资源编译器读到的字符仍不对。这是本次额外产生的问题，不是文章漏列的改名步骤。
</div>

`InternalName` 和 `OriginalFilename` 仍是旧名称，暂时不算第 07 篇漏改：它们不是这两项显示文字的依赖，也不会控制真正的 DLL 文件名。

另外，普通 Git 差异把 `.rc` 显示为二进制，是因为仓库 `.gitattributes` 指定了 `*.rc binary`，不是你把文件改成了二进制。用 `git diff --text` 就能查看文字差异。

## 验证状态

- 两项名称和版权行已核对；`resource.h` 没有 Git 修改。
- 版权行尚未恢复，我没有替你修改源码。
- 没有编译、运行，也没有检查生成 DLL 的文件属性。

## 2026-09-29 复查：版权行已恢复

按你的要求，已恢复 90 行的 `©` 和后面的空格，并保持 Windows-1252 编码。重新检查文字差异后，这个文件只剩你计划中的两处名称修改，版权行与 HEAD 一致。

上面保留首次检查时的问题记录，现已解决。没有编译运行。

## 2026-09-29 全量复查

两项 ZRender 名称仍正确，但之前修复的版权行又变成了 `Copyright ?2026 Baldur Karlsson`。当前文件 Git blob 与首次发现损坏时相同；与修复后只剩两行名称变化的差异不同，确认发生了回退。

<div style="border-left: 5px solid #d32f2f; background-color: #ffebee; color: #8b1717; padding: 12px 16px; margin: 16px 0;">
<strong>🔴 问题重新出现：版权行再次被覆盖</strong><br>
renderdoc/data/renderdoc.rc:90 需要恢复为 Copyright © 2026 Baldur Karlsson，并保持 Windows-1252 编码。可能是编辑器保存了修复前的缓冲区，也可能再次发生编码转换；磁盘差异不能确定是哪一种。先确认 VS 当前打开的内容并处理旧缓冲区，避免修复后再次覆盖。此次只检查，没有再次修改源码。
</div>
