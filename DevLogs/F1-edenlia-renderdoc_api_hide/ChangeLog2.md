# F1-qrenderdoc-process-hide ChangeLog2

> **Feature 目标**：隐藏 qrenderdoc.exe 及 renderdoccmd.exe 的进程名和所有可被游戏反作弊系统外部扫描到的环境特征。
>
> **背景**：ChangeLog1已完成对 renderdoc 核心库（DLL/导出符号/Vulkan 层/命名对象等）的反检测改造，但用户反馈发现：**即使没有通过 RenderDoc 注入目标进程，仅仅是打开 qrenderdoc.exe，游戏就会报检测到调试设备。**
>
> **分析结论**：这不是 renderdoc 主动做了什么（qrenderdoc 启动时只做标准的 Qt 应用初始化，不会主动注入其他进程），而是**游戏的反作弊系统主动扫描系统环境**检测到了 RenderDoc 的存在。
>
> **核心策略**：将 qrenderdoc.exe 及其关联的所有可被外部扫描到的特征进行重命名，命名规则与 F1 保持一致：`renderdoc` → `redendoc`，`RenderDoc` → `RedenDoc`。

---

## 一、进程名隐藏：qrenderdoc 构建输出名

### 检测原理

反作弊系统通过进程枚举检测 RenderDoc UI 进程：

```cpp
// 遍历进程列表匹配进程名
HANDLE hSnap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
Process32First(hSnap, &pe);
do {
    if (strstr(pe.szExeFile, "renderdoc") || strstr(pe.szExeFile, "qrenderdoc"))
        // 检测到 RenderDoc
} while (Process32Next(hSnap, &pe));
```

### 隐藏思路

修改 qrenderdoc 的构建系统输出名，使编译产物为 `qredendoc.exe` 而非 `qrenderdoc.exe`。qrenderdoc 使用 qmake（`.pro`）、CMake 和 MSVC（`.vcxproj`）三套构建系统，三者都需要同步更新。

### 具体修改

| 修改点 | 文件 | 说明 |
|--------|------|------|
| qmake 目标名 | `qrenderdoc/qrenderdoc.pro` | `TARGET = qrenderdoc` → `TARGET = qredendoc` |
| MSVC 项目名/输出名 | `qrenderdoc/qrenderdoc_local.vcxproj` | `<ProjectName>` → `qredendoc`；`<PrimaryOutput>` → `qredendoc`；`<TargetName>` → `qredendoc` |
| CMake install/Xcode | `qrenderdoc/CMakeLists.txt` | `install` 命令中的 `qrenderdoc` → `qredendoc`；Xcode scheme 中的 `qrenderdoc.app` → `qredendoc.app`。注意：CMake target 名 `build-qrenderdoc` 和 option 名 `ENABLE_QRENDERDOC` 等不影响输出文件名，保留不改 |

---

## 二、进程名隐藏：renderdoccmd 构建输出名

### 检测原理

同上，反作弊系统可通过进程枚举匹配 `renderdoccmd.exe`。

### 隐藏思路

修改 renderdoccmd 的构建系统输出名为 `redendoccmd.exe`。F1 中 `win32_process.cpp` 已将对 `renderdoccmd.exe` 的引用改为 `redendoccmd.exe`，此处确保实际输出名与之匹配。

### 具体修改

| 修改点 | 文件 | 说明 |
|--------|------|------|
| MSVC 输出名 | `renderdoccmd/renderdoccmd.vcxproj` | 添加 `<TargetName>redendoccmd</TargetName>`（保留 `<ProjectName>renderdoccmd</ProjectName>` 不变，避免影响项目引用） |
| CMake 输出名 | `renderdoccmd/CMakeLists.txt` | 在非 Android 平台添加 `set_target_properties(renderdoccmd PROPERTIES OUTPUT_NAME "redendoccmd")`；CMake target 名 `renderdoccmd` 保留不改 |

---

## 三、PE 版本资源特征隐藏

### 检测原理

反作弊系统可读取 PE 文件的版本信息资源：

```cpp
DWORD dwHandle;
DWORD dwSize = GetFileVersionInfoSizeA("qrenderdoc.exe", &dwHandle);
GetFileVersionInfoA("qrenderdoc.exe", dwHandle, dwSize, pData);
VerQueryValueA(pData, "\\StringFileInfo\\...\\InternalName", &pValue, &uLen);
// 检查 pValue 是否包含 "renderdoc"
```

### 隐藏思路

修改 qrenderdoc.exe 和 renderdoccmd.exe 的 `.rc` 资源文件中所有包含 "renderdoc" 的版本信息字段。

### 具体修改

| 修改点 | 文件 | 说明 |
|--------|------|------|
| qrenderdoc PE 资源 | `qrenderdoc/Resources/qrenderdoc.rc` | `InternalName` → `"qredendoc"`；`OriginalFilename` → `"qredendoc.exe"` |
| renderdoccmd PE 资源 | `renderdoccmd/renderdoccmd.rc` | `FileDescription` → `"redendoccmd - https://renderdoc.org/"`；`InternalName` → `"redendoccmd.exe"`；`OriginalFilename` → `"redendoccmd.exe"`；`ProductName` → `"RedenDoc"` |

---

## 四、代码中 qrenderdoc.exe 硬编码引用

### 检测原理

即使构建输出名已改，如果代码中仍硬编码 `qrenderdoc.exe` 字符串，这些字符串会残留在二进制中，可被内存扫描检测。同时，功能上也需要引用正确的文件名。

### 具体修改

| 修改点 | 文件 | 说明 |
|--------|------|------|
| 启动 UI 进程 | `renderdoccmd/renderdoccmd_win32.cpp` | 4 处 `qrenderdoc.exe` 字符串引用 → `qredendoc.exe`（用于从 renderdoccmd 启动 qrenderdoc UI） |
| 崩溃对话框 | `qrenderdoc/Windows/Dialogs/CrashDialog.cpp` | 3 处 `qrenderdoc.exe` 字符串引用 → `qredendoc.exe`（崩溃重启时的进程名） |
| 注释一致性 | `renderdoc/os/win32/win32_stringio.cpp` | 注释中的 `qrenderdoc.exe` → `qredendoc.exe`（不影响二进制，但保持代码一致性） |

---

## 五、代码中 renderdoccmd 硬编码引用

### 检测原理

同上，代码中硬编码的 `renderdoccmd` 字符串会残留在二进制中。此外，窗口类名和窗口标题也是重要的检测向量：

```cpp
// 通过窗口类名检测
FindWindowW(L"renderdoccmd", NULL);

// 通过窗口标题检测
EnumWindows(callback, ...); // callback 中 GetWindowText 匹配 "renderdoccmd"
```

### 具体修改

| 修改点 | 文件 | 说明 |
|--------|------|------|
| 窗口类名注册 | `renderdoccmd/renderdoccmd_win32.cpp` | `wc.lpszClassName = L"renderdoccmd"` → `L"redendoccmd"` |
| 窗口创建（预览） | `renderdoccmd/renderdoccmd_win32.cpp` | `CreateWindowEx(..., L"renderdoccmd", L"Remote Server Preview", ...)` 中的窗口类名 → `L"redendoccmd"` |
| 窗口创建（默认） | `renderdoccmd/renderdoccmd_win32.cpp` | `CreateWindowEx(..., L"renderdoccmd", L"renderdoccmd", ...)` 中的窗口类名和标题 → `L"redendoccmd"` |
| 默认参数 | `renderdoccmd/renderdoccmd_win32.cpp` | `argv.push_back("renderdoccmd")` → `"redendoccmd"` |
| 崩溃处理路径 | `renderdoc/core/crash_handler.h` | `"/renderdoccmd.exe\"` → `"/redendoccmd.exe\"`（崩溃处理器启动 renderdoccmd 的路径） |
| 更新对话框 | `qrenderdoc/Windows/Dialogs/UpdateDialog.cpp` | `lit("renderdoccmd.exe")` → `lit("redendoccmd.exe")`（更新时启动 renderdoccmd 的文件名） |

---

## 六、窗口标题和日志字符串

### 检测原理

反作弊系统可通过窗口枚举检测 RenderDoc UI 的窗口标题：

```cpp
EnumWindows([](HWND hwnd, LPARAM) -> BOOL {
    char title[256];
    GetWindowTextA(hwnd, title, sizeof(title));
    if (strstr(title, "RenderDoc") || strstr(title, "QRenderDoc"))
        // 检测到 RenderDoc UI
    return TRUE;
}, 0);
```

### 具体修改

| 修改点 | 文件 | 说明 |
|--------|------|------|
| 主窗口默认标题 | `qrenderdoc/Windows/MainWindow.ui` | `<string>QRenderDoc</string>` → `<string>QRedenDoc</string>` |
| 日志字符串 | `qrenderdoc/Code/ReplayManager.cpp` | `"QRenderDoc - renderer created for"` → `"QRedenDoc - renderer created for"` |
| RGP 互操作名 | `qrenderdoc/Code/RGPInterop.h` | `interop_name = QStringLiteral("RenderDoc")` → `QStringLiteral("RedenDoc")`（RGP 通过此名称发现 RenderDoc） |

---

## 七、WiX 安装脚本（Windows Installer）

### 检测原理

反作弊系统可扫描文件系统和注册表来发现 RenderDoc 的安装痕迹：

```cpp
// 注册表扫描
RegOpenKeyExA(HKEY_CLASSES_ROOT, "RenderDoc.RDCCapture.1", ...);

// 文件路径扫描
PathFileExistsA("C:\\Program Files\\RenderDoc\\qrenderdoc.exe");

// 开始菜单扫描
// 检查 Start Menu\Programs\RenderDoc\ 目录
```

### 具体修改

| 修改点 | 文件 | 说明 |
|--------|------|------|
| 32 位安装脚本 | `util/installer/Installer32.wxs` | `qrenderdoc.exe` → `qredendoc.exe`；`renderdoccmd.exe` → `redendoccmd.exe`；`RenderDoc.RDCCapture.1` → `RedenDoc.RDCCapture.1`；`RenderDoc.RDCSettings.1` → `RedenDoc.RDCSettings.1`；安装目录名 `RenderDoc` → `RedenDoc`；开始菜单快捷方式名 `RenderDoc` → `RedenDoc`；Product Name → `RedenDoc`；描述文本中的 `RenderDoc` → `RedenDoc` |
| 64 位安装脚本 | `util/installer/Installer64.wxs` | 同 32 位安装脚本的修改规则 |

> **注意**：WiX 内部的 Component Id、File Id、ComponentRef Id 等标识符（如 `QRenderDoc`、`RenderDocCPP`）不会暴露到注册表或文件系统，保留不改。DLL 文件名（`renderdoc.dll` 等）已在 F1 中通过 `RDOC_BASE_NAME` 宏处理，安装脚本中的引用需与实际构建输出匹配。

---

## 八、Linux 桌面环境文件

### 检测原理

Linux 平台下，反作弊系统可扫描桌面文件和菜单项来发现 RenderDoc：

```bash
# 检查 .desktop 文件
grep -r "renderdoc" /usr/share/applications/
# 检查菜单
grep -r "renderdoc" /usr/share/menu/
```

### 具体修改

| 修改点 | 文件 | 说明 |
|--------|------|------|
| 桌面文件 | `qrenderdoc/share/renderdoc.desktop` | `Name=RenderDoc` → `Name=RedenDoc`；`Exec=qrenderdoc %f` → `Exec=qredendoc %f` |
| 菜单文件 | `qrenderdoc/share/menu` | `command="qrenderdoc"` → `command="qredendoc"`；`title="RenderDoc"` → `title="RedenDoc"` |
| 缩略图处理器 | `qrenderdoc/share/renderdoc.thumbnailer` | `TryExec=/usr/bin/renderdoccmd` → `TryExec=/usr/bin/redendoccmd`；`Exec=/usr/bin/renderdoccmd` → `Exec=/usr/bin/redendoccmd` |

---

## 九、构建脚本文件名引用

### 隐藏思路

构建脚本中引用了 `qrenderdoc.exe`、`renderdoccmd.exe` 等文件名，需要与实际构建输出名保持一致。

### 具体修改

| 修改点 | 文件 | 说明 |
|--------|------|------|
| 编译脚本 | `util/buildscripts/scripts/compile_win32.sh` | `qrenderdoc.exe` → `qredendoc.exe`（2处）；`renderdoccmd.exe` → `redendoccmd.exe`（2处） |
| 打包脚本 | `util/buildscripts/scripts/make_package_win32.sh` | `renderdoccmd.exe` → `redendoccmd.exe`（2处）；`renderdoccmd.pdb` → `redendoccmd.pdb`（2处） |

---

## 修改分类总结

| 检测向量 | 检测手段 | 隐藏策略 | 状态 |
|---------|---------|---------|------|
| **qrenderdoc 进程名** | `CreateToolhelp32Snapshot` / 进程枚举 | `qrenderdoc.exe` → `qredendoc.exe`（构建输出名） | ✅ |
| **renderdoccmd 进程名** | `CreateToolhelp32Snapshot` / 进程枚举 | `renderdoccmd.exe` → `redendoccmd.exe`（构建输出名） | ✅ |
| **窗口标题枚举** | `EnumWindows` + `GetWindowText` | 窗口标题 `"QRenderDoc"` → `"QRedenDoc"` | ✅ |
| **窗口类名枚举** | `FindWindow` / `GetClassName` | renderdoccmd 窗口类名 `"renderdoccmd"` → `"redendoccmd"` | ✅ |
| **PE 版本信息** | `GetFileVersionInfo` / PE 资源读取 | `InternalName`/`OriginalFilename` 等字段全部清除 "renderdoc" 特征 | ✅ |
| **安装目录扫描** | 文件系统扫描 | 安装目录 `RenderDoc` → `RedenDoc` | ✅ |
| **注册表扫描** | 注册表枚举 | `RenderDoc.RDCCapture.1` → `RedenDoc.RDCCapture.1` 等 | ✅ |
| **开始菜单扫描** | 文件系统扫描 | 快捷方式名 `RenderDoc` → `RedenDoc` | ✅ |
| **RGP 互操作** | RGP 工具发现 | `interop_name = "RenderDoc"` → `"RedenDoc"` | ✅ |
| **Linux 桌面文件** | 文件系统扫描 | `.desktop`/`menu`/`.thumbnailer` 中的引用全部更新 | ✅ |
| **构建脚本** | — | 所有构建/打包脚本中的文件名引用同步更新 | ✅ |

---

## 与 F1 的关系

本次 F2 修改是 F1 的补充。F1 聚焦于 **renderdoc 核心库在目标进程内部的特征隐藏**（DLL 模块名、导出符号、Vulkan 层、命名对象、窗口类名、文件路径/注册表等），而 F2 聚焦于 **qrenderdoc UI 进程和 renderdoccmd 命令行工具在系统环境中的外部可见特征隐藏**（进程名、窗口标题、PE 版本信息、安装目录、注册表项等）。

两者结合后，游戏反作弊系统通过以下所有方式均**无法检测到 RenderDoc**：

| 层级 | 检测方式 | F1 覆盖 | F2 覆盖 |
|------|---------|---------|---------|
| 进程内 | DLL 模块名 | ✅ | — |
| 进程内 | DLL 导出符号 | ✅ | — |
| 进程内 | Vulkan 层名称 | ✅ | — |
| 进程内 | Windows 命名对象 | ✅ | — |
| 进程内 | OpenGL 窗口类名 | ✅ | — |
| 系统级 | 进程名枚举 | — | ✅ |
| 系统级 | 窗口标题/类名枚举 | — | ✅ |
| 系统级 | PE 版本信息 | — | ✅ |
| 系统级 | 安装目录/注册表 | 部分 | ✅ |
| 系统级 | 文件路径扫描 | ✅ | ✅ |
