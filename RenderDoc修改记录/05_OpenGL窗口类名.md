# 05：OpenGL窗口类名——修改记录

日期：2026-09-29。根据 `git diff HEAD` 检查。

## 实际修改

| 文件与行号 | 修改前 | 修改后 |
| --- | --- | --- |
| `renderdoc/driver/gl/wgl_platform.cpp:28` | `#define WINDOW_CLASS_NAME L"renderdocGLclass"` | `#define WINDOW_CLASS_NAME L"zrenderGLclass"` |

## 评价

**这处改好了，没有发现还需要同步修改的窗口类名。**

581 行注册窗口类时使用 `WINDOW_CLASS_NAME`，102、198、421、512 行创建窗口时也使用这个宏。因此重新编译后，它们会一起使用 `zrenderGLclass`，不会出现一边注册新名、另一边还找旧名的问题。

421 行还有一句 `L"RenderDoc replay window"`，那是窗口标题，不是窗口类名。你可以为了统一显示文字而改它，但不改也不影响这里按新类名创建窗口，不属于遗漏。

这次不需要红色提醒；不能因为还有文字包含 RenderDoc，就把它都当成必须修改的内容。

## 验证状态

- 已检查宏定义及注册、创建窗口的引用，没有找到其他旧类名引用。
- `git diff --check` 通过。
- 没有编译、运行。后面实际验证时要走 OpenGL 回放路径，只打开主界面不能验证这一处。

## 2026-09-29 全量复查

zrenderGLclass 仍保留，注册和创建引用同一宏。未发现修改回退。
