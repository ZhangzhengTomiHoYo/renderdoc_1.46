# 07：DLL 的文件描述与产品名称

## 这次改的是什么？

这是 DLL 的**版本资源信息**，通常能在 Windows“文件属性 → 详细信息”中看到：

- `FileDescription`：文件描述。
- `ProductName`：产品名称。

`.rc` 是资源脚本，会经过资源编译，再链接进 DLL。因此改完要重新生成核心 DLL，已有的 DLL 不会自动更新。

## 与前面的改名有什么关系？

**这里改的是文件内的描述信息，不是实际文件名，也不是导出符号。** 改成 `RenderTest` 不会自动生成 `rendertest.dll`，也不会改变 marker、启动路径或共享内存名称。

这两项不需要为了前面的功能配套而必须修改；如果目的是统一产品显示名称，它们才是对应的修改点。

## 修改表（按当前源码校正）

原文路径少了仓库根目录下的 `renderdoc/`，行号正确。下表补成完整资源条目。

| 文件 | 行号 | 修改类型 | 修改前 | 修改后 |
| --- | --- | --- | --- | --- |
| `renderdoc/data/renderdoc.rc` | 87 | 文件描述 | `VALUE "FileDescription", "Core DLL for RenderDoc"` | `VALUE "FileDescription", "Core DLL for RenderTest"` |
| `renderdoc/data/renderdoc.rc` | 92 | 产品名称 | `VALUE "ProductName", "RenderDoc"` | `VALUE "ProductName", "RenderTest"` |

## 完整性与验证

这两处足以更新这两个字段，**不代表所有文件信息都已改名**：同一资源中，89 行 `InternalName` 仍为 `renderdoc`，91 行 `OriginalFilename` 仍为 `renderdoc.dll`。是否需要更新它们，等实际 DLL 改名方案明确后再决定；`OriginalFilename` 本身也不会决定输出文件名。

改后重新生成核心 DLL，打开本次输出文件的“属性 → 详细信息”，检查文件描述和产品名称是否更新。这只能验证资源字段，不能证明前面各项功能配套已经正确。

状态：已核对源码；尚未实际修改或构建验证。
