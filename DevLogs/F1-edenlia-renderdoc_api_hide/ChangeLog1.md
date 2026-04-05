# F1-Edenlia-renderdoc_api_hide ChangeLog

> **Feature 目标**：对 RenderDoc 进行反检测改造，使其在目标进程中不易被反作弊系统发现。
>
> **核心策略**：将所有对外暴露的 `renderdoc` / `RENDERDOC` 特征重命名为 `redendoc` / `REDENDOC`，按检测向量优先级分层实施。

---

## 一、P0：模块枚举检测（DLL/EXE 文件名）

### 检测原理

反作弊系统最常用、成本最低的检测方式：

```cpp
// 方式1：直接按名称查找
HMODULE h = GetModuleHandleA("renderdoc.dll");

// 方式2：遍历模块列表匹配名称
EnumerateLoadedModules64(hProcess, callback, ...);
```

只要进程中加载了名为 `renderdoc.dll` 的模块，就会被立即发现。

### 隐藏思路

**`RDOC_BASE_NAME` 是核心枢纽**。RenderDoc 的构建系统通过这个宏统一管理所有输出二进制的名称，代码中约 32 处通过 `STRINGIZE(RDOC_BASE_NAME)` 引用它。只需修改这一个宏，所有模块名引用自动生效。

### 具体修改

| 修改点 | 文件 | 说明 |
|--------|------|------|
| CMake 核心宏 | `CMakeLists.txt` | `set(RDOC_BASE_NAME "renderdoc")` → `set(RDOC_BASE_NAME "redendoc")`，所有 `STRINGIZE(RDOC_BASE_NAME)` 引用自动生效 |
| MSVC 预处理器 | `renderdoc/renderdoc.vcxproj` | `RDOC_BASE_NAME=$(ProjectName)` → `RDOC_BASE_NAME=redendoc`（因为 vcxproj 文件名不做重命名，`$(ProjectName)` 仍解析为 "renderdoc"，必须显式覆盖） |
| PE 版本资源 | `renderdoc/data/renderdoc.rc` | `InternalName` → `"redendoc"`；`OriginalFilename` → `"redendoc.dll"`；`FileDescription` / `ProductName` → `"RedenDoc"` |
| Linux 符号导出 | `renderdoc/renderdoc.version`、`renderdoc/rdocself.version` | 导出模式 `RENDERDOC_*;` → `REDENDOC_*;`，确保 `.so` 文件的符号表与新名称一致 |
| 进程/Shim 路径 | `renderdoc/os/win32/win32_process.cpp` | `renderdoccmd.exe` → `redendoccmd.exe`；`renderdocshim32/64.dll` → `redendocshim32/64.dll` |
| Hook 进程名匹配 | `renderdoc/os/win32/sys_win32_hooks.cpp` | `renderdoccmd.exe` → `redendoccmd.exe` |
| Cmd 加载路径 | `renderdoccmd/renderdoccmd_win32.cpp` | `renderdoc.dll` → `redendoc.dll` |

---

## 二、P0：导出符号检测（DLL Export Table）

### 检测原理

即使 DLL 改了名，反作弊系统仍可通过遍历所有已加载模块的导出表来检测特征函数名：

```cpp
// 遍历所有模块，检查是否存在 RenderDoc 的导出函数
for (auto& mod : loadedModules) {
    if (GetProcAddress(mod, "RENDERDOC_GetAPI") != NULL) {
        // 检测到 RenderDoc
    }
}
```

### 隐藏思路

需要修改两层：
1. **公共 API 头文件** `renderdoc_app.h` — 这是所有第三方应用集成 RenderDoc 时使用的头文件，定义了所有对外类型和函数指针
2. **导出函数实现** — 实际的 `__declspec(dllexport)` 函数定义

`renderdoc_app.h` 中的修改量最大（约 496 行），因为它包含了所有枚举值、函数指针类型、API 版本结构体等，全部带有 `RENDERDOC_` / `eRENDERDOC_` / `pRENDERDOC_` 前缀。

### 具体修改

| 修改点 | 文件 | 说明 |
|--------|------|------|
| 公共 API 头文件 | `renderdoc/api/app/renderdoc_app.h` | **全量替换**：`RENDERDOC_CC` → `REDENDOC_CC`、`RENDERDOC_CaptureOption` → `REDENDOC_CaptureOption`、所有 `eRENDERDOC_*` 枚举值、`pRENDERDOC_*` 函数指针、`RENDERDOC_API_1_*_*` 版本结构体、`RENDERDOC_OverlayBits` 等。注意：`RENDERDOC_ShaderDebugMagicValue` 等 GUID 宏名**不改**（编译后不出现在二进制中） |
| 内部 API 头文件 | `renderdoc/api/replay/renderdoc_replay.h` | 所有导出函数声明：`RENDERDOC_GetAPI` → `REDENDOC_GetAPI`、`RENDERDOC_SetDebugLogFile` → `REDENDOC_SetDebugLogFile`、`RENDERDOC_ProgressCallback` → `REDENDOC_ProgressCallback`、`RENDERDOC_InjectIntoProcess` → `REDENDOC_InjectIntoProcess` 等 |
| 导出函数实现 | `renderdoc/replay/app_api.cpp` | `RENDERDOC_GetAPI` 函数定义 → `REDENDOC_GetAPI` |
| 入口点字符串 | `renderdoc/replay/entry_points.cpp` | 所有导出函数定义和 `"RENDERDOC_GetAPI"` 字符串字面量 → `"REDENDOC_GetAPI"` |
| 注入调用 | `renderdoc/os/win32/win32_process.cpp` | `RENDERDOC_SetDebugLogFile(...)` 调用 → `REDENDOC_SetDebugLogFile(...)` |
| 单元测试入口 | `renderdoc/3rdparty/catch/catch.cpp` | `RENDERDOC_RunUnitTests` → `REDENDOC_RunUnitTests` |

---

## 三、P1：Vulkan 层名称检测

### 检测原理

反作弊系统可以通过 Vulkan API 枚举已注册的层，或检查环境变量：

```cpp
// 方式1：枚举 Vulkan 层
vkEnumerateInstanceLayerProperties(&count, layers);
// 检查是否存在 "VK_LAYER_RENDERDOC_Capture"

// 方式2：检查环境变量
getenv("ENABLE_VULKAN_RENDERDOC_CAPTURE");
```

### 隐藏思路

Vulkan 层的名称由两部分控制：
1. **宏定义** `RENDERDOC_VULKAN_LAYER_NAME` — 控制运行时注册的层名
2. **导出函数名** — Vulkan loader 通过特定命名规则的导出函数来发现层（如 `VK_LAYER_RENDERDOC_CaptureGetInstanceProcAddr`）
3. **JSON 清单文件** — Vulkan loader 通过 JSON 文件发现隐式层

三者必须同步修改，否则 Vulkan 层无法正常加载。

### 具体修改

| 修改点 | 文件 | 说明 |
|--------|------|------|
| 层名称宏 | `renderdoc/common/globalconfig.h` | `RENDERDOC_VULKAN_LAYER_NAME` → `REDENDOC_VULKAN_LAYER_NAME`（值 `"VK_LAYER_RENDERDOC_Capture"` → `"VK_LAYER_REDENDOC_Capture"`） |
| 环境变量宏 | `renderdoc/common/globalconfig.h` | `RENDERDOC_VULKAN_LAYER_VAR` → `REDENDOC_VULKAN_LAYER_VAR`（值 `"ENABLE_VULKAN_RENDERDOC_CAPTURE"` → `"ENABLE_VULKAN_REDENDOC_CAPTURE"`） |
| 层导出函数 | `renderdoc/driver/vulkan/vk_layer.cpp` | 所有 `VK_LAYER_RENDERDOC_Capture*` 导出函数名 → `VK_LAYER_REDENDOC_Capture*`（含 `GetDeviceProcAddr`、`GetInstanceProcAddr`、`NegotiateLoaderLayerInterfaceVersion` 等）；MSVC `/EXPORT` 链接指令同步更新 |
| JSON 清单 | `renderdoc/driver/vulkan/renderdoc.json` | 层名称和版本引用更新 |
| Android 层注册 | `renderdoc/driver/vulkan/vk_layer_android.cpp` | 相关宏引用更新 |
| POSIX 层路径 | `renderdoc/driver/vulkan/vk_posix.cpp` | 相关宏引用更新 |

---

## 四、P1：Windows 命名对象检测

### 检测原理

反作弊系统可以通过枚举系统命名对象（事件、管道、共享内存）来检测 RenderDoc：

```cpp
// 检查命名事件
OpenEventA(EVENT_ALL_ACCESS, FALSE, "RENDERDOC_CRASHHANDLE");

// 检查命名管道
CreateFileA("\\\\.\\pipe\\RenderDocBreakpadServer", ...);

// 检查共享内存
OpenFileMappingA(FILE_MAP_READ, FALSE, "RenderDocGlobalHookData64");
```

### 隐藏思路

RenderDoc 在运行时创建了多个命名对象用于进程间通信，这些名称是硬编码的字符串常量，需要逐一替换。

### 具体修改

| 修改点 | 文件 | 说明 |
|--------|------|------|
| 崩溃处理事件/管道 | `renderdoc/core/crash_handler.h` | 事件名 `RENDERDOC_CRASHHANDLE` → `REDENDOC_CRASHHANDLE`；管道名 `RenderDocBreakpadServer` → `RedenDocBreakpadServer`；文件夹名 `RenderDoc` → `RedenDoc` |
| 全局 Hook 共享内存 | `renderdocshim/renderdocshim.h` | `RenderDocGlobalHookData32/64` → `RedenDocGlobalHookData32/64` |

---

## 五、P1：OpenGL 窗口类名检测

### 检测原理

```cpp
// 通过窗口类名检测
FindWindowA("renderdocGLclass", NULL);
```

RenderDoc 在初始化 OpenGL hook 时会创建一个隐藏窗口，窗口类名是硬编码的。

### 具体修改

| 修改点 | 文件 | 说明 |
|--------|------|------|
| GL 窗口类名 | `renderdoc/driver/gl/wgl_platform.cpp` | `"renderdocGLclass"` → `"redendocGLclass"` |

---

## 六、P1：文件路径 / 注册表项检测

### 检测原理

反作弊系统可以扫描文件系统或注册表来发现 RenderDoc 的安装痕迹：

```cpp
// Windows 注册表
RegOpenKeyExA(HKEY_LOCAL_MACHINE, "SOFTWARE\\RenderDoc\\...", ...);

// 文件路径
PathFileExistsA("C:\\Program Files\\RenderDoc\\renderdoc.dll");

// Linux 配置目录
access("~/.config/renderdoc/...", F_OK);
```

### 隐藏思路

所有平台的路径和注册表项中的 `renderdoc` / `RenderDoc` 需要全面替换。这些分散在各平台的 `stringio` 文件中。

### 具体修改

| 修改点 | 文件 | 说明 |
|--------|------|------|
| Windows 路径/注册表 | `renderdoc/os/win32/win32_stringio.cpp` | 路径和注册表项中的 `RenderDoc` → `RedenDoc` |
| Linux 配置路径 | `renderdoc/os/posix/linux/linux_stringio.cpp` | `~/.config/renderdoc` 等路径替换 |
| BSD 配置路径 | `renderdoc/os/posix/bsd/bsd_stringio.cpp` | 同上 |
| macOS 配置路径 | `renderdoc/os/posix/apple/apple_stringio.cpp` | 同上 |
| Android 配置路径 | `renderdoc/os/posix/android/android_stringio.cpp` | 同上 |
| POSIX 通用路径 | `renderdoc/os/posix/posix_stringio.cpp` | 同上 |

---

## 七、P2：版本号 / 编译宏 / 内部符号全面清理

### 思路

前面的 P0/P1 修改覆盖了所有**对外直接暴露**的特征。但代码内部仍有大量 `RENDERDOC_` 前缀的宏、类型、回调等。虽然这些宏名在编译后通常不直接出现在二进制中，但在以下场景可能泄露：

1. **字符串化宏**（`STRINGIZE`）会将宏名嵌入二进制
2. **调试信息**（PDB）中保留类型名和符号名
3. **RTTI**（运行时类型信息）中保留类型名

因此需要全面清理，确保编译后的二进制中不残留任何 `RENDERDOC` 字符串。

### 具体修改

#### 7.1 版本号宏

| 文件 | 说明 |
|------|------|
| `renderdoc/api/replay/version.h` | `REDENDOC_VERSION_MAJOR/MINOR` → `REDENDOC_VERSION_MAJOR/MINOR`；`RENDERDOC_STABLE_BUILD` → `REDENDOC_STABLE_BUILD`；`RENDERDOC_OFFICIAL_BUILD` → `REDENDOC_OFFICIAL_BUILD` |
| `renderdoc/renderdoc.vcxproj` | 版本号解析中的宏引用同步更新 |

#### 7.2 平台配置宏

| 文件 | 说明 |
|------|------|
| `renderdoc/common/globalconfig.h` | `RENDERDOC_WINDOWING_XLIB/XCB/WAYLAND` → `REDENDOC_WINDOWING_XLIB/XCB/WAYLAND`；`RENDERDOC_ANDROID_PACKAGE_BASE` → `REDENDOC_ANDROID_PACKAGE_BASE`；`RENDERDOC_ANDROID_LIBRARY` → `REDENDOC_ANDROID_LIBRARY` |

#### 7.3 核心框架内部类型

| 文件 | 说明 |
|------|------|
| `renderdoc/core/core.cpp` | 注解类型 `RENDERDOC_AnnotationType` → `REDENDOC_AnnotationType`；所有 `eRENDERDOC_*` 枚举值引用 |
| `renderdoc/core/core.h` | 回调类型、API 版本引用等 |
| `renderdoc/core/remote_server.cpp/h` | `RENDERDOC_ProgressCallback` → `REDENDOC_ProgressCallback` |
| `renderdoc/core/replay_proxy.cpp/h` | 同上 |
| `renderdoc/core/target_control.cpp`、`plugins.cpp`、`settings.cpp/h`、`crash_handler.h` | 同上 |

#### 7.4 图形 API 驱动内部符号

| 模块 | 涉及文件 |
|------|---------|
| **D3D11** | `d3d11_context.cpp/h`、`d3d11_context_wrap.cpp`、`d3d11_debug.cpp`、`d3d11_device.cpp/h`、`d3d11_pixelhistory.cpp`、`d3d11_rendertext.cpp`、`d3d11_rendertexture.cpp`、`d3d11_resources.cpp/h` |
| **D3D12** | `d3d12_command_list.h`、`d3d12_command_list_wrap.cpp`、`d3d12_command_queue.h/wrap.cpp`、`d3d12_commands.cpp`、`d3d12_debug.cpp`、`d3d12_device.cpp/h`、`d3d12_manager.cpp`、`d3d12_overlay.cpp`、`d3d12_pixelhistory.cpp`、`d3d12_rendertext.cpp`、`d3d12_rendertexture.cpp`、`d3d12_resources.cpp/h` |
| **OpenGL** | `gl_common.h`、`gl_driver.cpp/h`、`gl_overlay.cpp`、`gl_pixelhistory.cpp`、`gl_rendertexture.cpp`、`gl_replay.cpp`、`gl_debug_funcs.cpp`、`gl_interop_funcs.cpp`、`egl_hooks.cpp` |
| **Vulkan** | `vk_common.h`、`vk_core.cpp/h`、`vk_pixelhistory.cpp`、`vk_replay.cpp`、`vk_resources.h`、`vk_device_funcs.cpp`、`vk_misc_funcs.cpp`、`vk_queue_funcs.cpp`、`vk_wsi_funcs.cpp` |
| **Metal** | `metal_device.h` |
| **IHV** | `ags_wrapper.cpp`（AMD AGS 相关） |

> **补充说明（ChangeLog3 追溯）**：上述 D3D11/D3D12/GL 驱动文件中，除了通用的 `RENDERDOC_` 前缀宏/类型/枚举值引用外，还包含大量 **shader 入口点字符串** 的变更。这些字符串作为 `D3DCompile` / `GetShaderBlob` / `MakeVShader` / `MakePShader` / `MakeCShader` / `MakeGShader` / `glGetUniformLocation` 等 API 的参数，直接决定了运行时 shader 编译能否找到正确的入口点函数。具体变更如下：
>
> **D3D11 shader 入口点字符串变更：**
> - `d3d11_rendertext.cpp`（4 处）：`MakeVShader(hlsl, "RENDERDOC_TextVS", ...)` → `"REDENDOC_TextVS"`、`MakeVShader(hlsl, "RENDERDOC_Text9VS", ...)` → `"REDENDOC_Text9VS"`、`MakePShader(hlsl, "RENDERDOC_TextPS", ...)` → `"REDENDOC_TextPS"`（2 处，分别对应 FL10+ 和 FL9 模式）
> - `d3d11_debug.cpp`（约 30 处）：包括 `"RENDERDOC_FullscreenVS"` → `"REDENDOC_FullscreenVS"`、`"RENDERDOC_FixedColPS"` → `"REDENDOC_FixedColPS"`、`"RENDERDOC_CheckerboardPS"` → `"REDENDOC_CheckerboardPS"`、`"RENDERDOC_DiscardFloatPS"` → `"REDENDOC_DiscardFloatPS"`、`"RENDERDOC_DiscardIntPS"` → `"REDENDOC_DiscardIntPS"`、`"RENDERDOC_TexDisplayVS/PS"` → `"REDENDOC_TexDisplayVS/PS"`、`"RENDERDOC_TexRemapFloat/UInt/SInt"` → `"REDENDOC_TexRemapFloat/UInt/SInt"`、`"RENDERDOC_QuadOverdrawPS"` → `"REDENDOC_QuadOverdrawPS"`、`"RENDERDOC_QOResolvePS"` → `"REDENDOC_QOResolvePS"`、`"RENDERDOC_MeshVS"` → `"REDENDOC_MeshVS"`、`"RENDERDOC_TriangleSizeGS/PS"` → `"REDENDOC_TriangleSizeGS/PS"`、`"RENDERDOC_DepthCopyPS/ArrayPS/MSPS/MSArrayPS"` → `"REDENDOC_DepthCopyPS/ArrayPS/MSPS/MSArrayPS"`、`"RENDERDOC_CopyMSToArray"` 等 multisample 系列 → `"REDENDOC_*"`、`"RENDERDOC_PixelHistoryUnused/CopyPixel"` → `"REDENDOC_*"`、`"RENDERDOC_TileMinMaxCS/ResultMinMaxCS/HistogramCS"` → `"REDENDOC_*"` 等
> - `d3d11_rendertexture.cpp`（9 处）：自定义 shader cbuffer 变量名匹配字符串 `"RENDERDOC_TexDim"` → `"REDENDOC_TexDim"`、`"RENDERDOC_YUVDownsampleRate"` → `"REDENDOC_YUVDownsampleRate"`、`"RENDERDOC_YUVAChannels"` → `"REDENDOC_YUVAChannels"`、`"RENDERDOC_SelectedMip"` → `"REDENDOC_SelectedMip"`、`"RENDERDOC_SelectedSliceFace"` → `"REDENDOC_SelectedSliceFace"`、`"RENDERDOC_SelectedSample"` → `"REDENDOC_SelectedSample"`、`"RENDERDOC_TextureType"` → `"REDENDOC_TextureType"`、`"RENDERDOC_SelectedRangeMin"` → `"REDENDOC_SelectedRangeMin"`、`"RENDERDOC_SelectedRangeMax"` → `"REDENDOC_SelectedRangeMax"`
>
> **D3D12 shader 入口点字符串变更：**
> - `d3d12_rendertext.cpp`（2 处）：`GetShaderBlob(hlsl, "RENDERDOC_TextVS", ...)` → `"REDENDOC_TextVS"`、`GetShaderBlob(hlsl, "RENDERDOC_TextPS", ...)` → `"REDENDOC_TextPS"`
> - `d3d12_debug.cpp`（约 32 处）：与 D3D11 类似的全部 shader 入口点字符串，通过 `GetShaderBlob` API 调用
> - `d3d12_overlay.cpp`（2 处）：`GetShaderBlob(hlsl, "RENDERDOC_QuadOverdrawPS", ...)` → `"REDENDOC_QuadOverdrawPS"`
> - `d3d12_manager.cpp`（13 处）：raytracing 相关 shader 入口点 `"RENDERDOC_PatchShaderTableCS"` → `"REDENDOC_PatchShaderTableCS"`、`"RENDERDOC_CopyShaderTableCS"` → `"REDENDOC_CopyShaderTableCS"`、`"RENDERDOC_PrepareRayIndirectExecuteCS"` → `"REDENDOC_PrepareRayIndirectExecuteCS"`、`"RENDERDOC_PrepareTLASCopyIndirectExecuteCS"` → `"REDENDOC_PrepareTLASCopyIndirectExecuteCS"`、`"RENDERDOC_CopyBLASInstanceCS"` → `"REDENDOC_CopyBLASInstanceCS"`、`"RENDERDOC_PatchAccStructAddressCS"` → `"REDENDOC_PatchAccStructAddressCS"`，以及对应的 pipeline `SetName` 调用
> - `d3d12_rendertexture.cpp`（9 处）：与 D3D11 相同的自定义 shader cbuffer 变量名匹配字符串
>
> **GL shader uniform 变量名变更：**
> - `gl_overlay.cpp`（2 处）：`glGetUniformLocation(prog, "RENDERDOC_Fixed_Color")` → `"REDENDOC_Fixed_Color"`
> - `gl_rendertexture.cpp`（7 处）：`glGetUniformLocation(prog, "RENDERDOC_TexDim")` → `"REDENDOC_TexDim"` 等自定义 shader uniform 变量名
>
> **注意**：上述 C++ 侧的字符串变更已在 ChangeLog1 中完成，但对应的 **shader 源码文件（HLSL/GLSL）中的函数定义和变量声明** 在 ChangeLog1 中遗漏，已在 ChangeLog3 中补充修复。

#### 7.5 序列化 / 回放模块

| 文件 | 说明 |
|------|------|
| `renderdoc/replay/capture_file.cpp`、`capture_options.cpp`、`replay_controller.cpp`、`replay_output.cpp` | `RENDERDOC_` 前缀引用更新 |
| `renderdoc/serialise/serialiser.cpp/h`、`streamio.cpp/h` | 同上 |
| `renderdoc/serialise/codecs/chrome_json_codec.cpp`、`xml_codec.cpp` | 同上 |

#### 7.6 API 头文件类型定义

| 文件 | 说明 |
|------|------|
| `renderdoc/api/replay/capture_options.h`、`control_types.h`、`data_types.h`、`pipestate.inl` | `RENDERDOC_ProgressCallback` 等类型引用 |
| `renderdoc/api/replay/rdcarray.h`、`rdcstr.h`、`rdcdatetime.h`、`resourceid.h`、`shader_types.h`、`structured_data.h` | `RENDERDOC_` 前缀宏引用 |

#### 7.7 Android 平台

| 文件 | 说明 |
|------|------|
| `renderdoc/android/android.cpp` | `RENDERDOC_ANDROID_PACKAGE_BASE`、`RENDERDOC_VULKAN_LAYER_NAME`、`RENDERDOC_ANDROID_LIBRARY`、`RENDERDOC_APK_PATH`、`RENDERDOC_CreateTargetControl`、`RENDERDOC_CheckAndroidPackage` 等 |
| `renderdoc/android/android_utils.cpp` | `RENDERDOC_ANDROID_PACKAGE_BASE` |
| `renderdoc/android/jdwp.cpp` | `RENDERDOC_ANDROID_LIBRARY` |

#### 7.8 POSIX / Linux / BSD / macOS 平台内部引用

| 文件 | 说明 |
|------|------|
| `posix_libentry.cpp`、`posix_process.cpp` | `RENDERDOC_` 前缀引用 |
| `linux_hook.cpp`、`linux_callstack.cpp` | 同上 |
| `bsd_hook.cpp`、`bsd_callstack.cpp` | 同上 |
| `apple_callstack.cpp` | 同上 |
| `android_hook.cpp`、`android_callstack.cpp` | 同上 |

#### 7.9 Windows 平台内部引用

| 文件 | 说明 |
|------|------|
| `win32_callstack.cpp`、`sys_win32_hooks.cpp` | `RENDERDOC_` 引用更新 |

#### 7.10 Qt 前端

| 文件 | 说明 |
|------|------|
| `qrenderdoc/Code/CaptureContext.cpp` | `REDENDOC_VERSION_MAJOR/MINOR` → `REDENDOC_VERSION_MAJOR/MINOR`；`RENDERDOC_PROFILEFUNCTION`/`RegisterMemoryRegion`/`UnregisterMemoryRegion`/`GetVersionString`/`OpenCaptureFile` → `REDENDOC_*` |
| `qrenderdoc/Code/CaptureContext.h` | 仅含平台宏 `RENDERDOC_PLATFORM_*`，无需修改 |
| `qrenderdoc/Code/qrenderdoc.cpp`、`qrenderdoc/Windows/MainWindow.cpp` | UI 显示文本 `"RenderDoc"` → `"RedenDoc"`；所有 `RENDERDOC_*` 导出函数调用 → `REDENDOC_*`（含 `LogMessage`、`InitialiseReplay`、`ShutdownReplay`、`GetCommitHash`、`GetVersionString`、`UpdateInstalledVersionNumber`、`EnumerateRemoteTargets`、`UpdateVulkanLayerRegistration`、`GetSupportedDeviceProtocols`、`GetDeviceProtocolController`、`CanSelfHostedCapture`、`StartSelfHostCapture`、`OpenCaptureFile`、`InjectIntoProcess`、`CreateBugReport`、`GetLogFile`、`EndSelfHostCapture`、`IsReleaseBuild`、`IsGlobalHookActive` 等）；`RENDERDOC_STABLE_BUILD` → `REDENDOC_STABLE_BUILD` |
| `qrenderdoc/Code/ReplayManager.h` | `RENDERDOC_ProgressCallback` → `REDENDOC_ProgressCallback` |
| `qrenderdoc/Code/ReplayManager.cpp` | `RENDERDOC_RegisterMemoryRegion`/`UnregisterMemoryRegion`/`OpenCaptureFile`/`ExecuteAndInject`/`ProgressCallback` → `REDENDOC_*` |
| `qrenderdoc/Code/QRDUtils.cpp` | `RENDERDOC_SetColors` → `REDENDOC_SetColors` |
| `qrenderdoc/Code/pyrenderdoc/qrenderdoc_stub.cpp` | `RENDERDOC_GetDefaultCaptureOptions` → `REDENDOC_GetDefaultCaptureOptions` |
| `qrenderdoc/Code/pyrenderdoc/function_conversion.h` | `RENDERDOC_LogMessage` → `REDENDOC_LogMessage` |
| `qrenderdoc/Code/pyrenderdoc/PythonContext.cpp` | `RENDERDOC_LogMessage` → `REDENDOC_LogMessage`；`RENDERDOC_STABLE_BUILD` → `REDENDOC_STABLE_BUILD` |
| `qrenderdoc/Code/Interface/QRDInterface.h` | **关键修复**：`#define RENDERDOC_QT_COMPAT` → `#define REDENDOC_QT_COMPAT`（此宏控制 `rdcstr`/`rdcarray` 等类型与 Qt `QString`/`QVariant` 之间的隐式转换运算符） |
| `qrenderdoc/Code/Interface/Extensions.h` | `#ifdef RENDERDOC_QT_COMPAT` → `#ifdef REDENDOC_QT_COMPAT` |
| `qrenderdoc/Code/Interface/QRDInterface.cpp` | `RENDERDOC_GetDefaultCaptureOptions` → `REDENDOC_GetDefaultCaptureOptions` |
| `qrenderdoc/Code/Interface/RemoteHost.cpp` | `RENDERDOC_GetDeviceProtocolController`/`CheckRemoteServerConnection`/`CreateRemoteServerConnection` → `REDENDOC_*` |
| `qrenderdoc/Code/Interface/PersistantConfig.cpp` | `RENDERDOC_SetConfigSetting`/`SaveConfigSettings`/`GetSupportedDeviceProtocols`/`GetDeviceProtocolController` → `REDENDOC_*` |
| `qrenderdoc/Windows/Dialogs/CaptureDialog.cpp` | `RENDERDOC_NeedVulkanLayerRegistration`/`CheckAndroidPackage`/`IsGlobalHookActive`/`StopGlobalHook`/`StartGlobalHook`/`CanGlobalHook` → `REDENDOC_*` |
| `qrenderdoc/Windows/Dialogs/SettingsDialog.cpp` | `RENDERDOC_GetConfigSetting`/`SetConfigSetting`/`SaveConfigSettings`/`CanGlobalHook` → `REDENDOC_*` |
| `qrenderdoc/Windows/Dialogs/RemoteManager.cpp` | `RENDERDOC_EnumerateRemoteTargets`/`CreateTargetControl` → `REDENDOC_*` |
| `qrenderdoc/Windows/Dialogs/LiveCapture.cpp` | `RENDERDOC_CreateTargetControl` → `REDENDOC_CreateTargetControl` |
| `qrenderdoc/Windows/Dialogs/CrashDialog.cpp` | `RENDERDOC_OpenCaptureFile` → `REDENDOC_OpenCaptureFile` |
| `qrenderdoc/Windows/Dialogs/UpdateDialog.cpp` | `RENDERDOC_EnumerateRemoteTargets`/`CreateTargetControl` → `REDENDOC_*` |
| `qrenderdoc/Windows/Dialogs/AboutDialog.cpp` | `RENDERDOC_GetCommitHash` → `REDENDOC_GetCommitHash` |
| `qrenderdoc/Windows/Dialogs/ConfigEditor.cpp` | `RENDERDOC_SetConfigSetting` → `REDENDOC_SetConfigSetting` |
| `qrenderdoc/Windows/BufferViewer.cpp` | `RENDERDOC_InitCamera`/`NumVerticesPerPrimitive` → `REDENDOC_*` |
| `qrenderdoc/Windows/LogView.cpp` | `RENDERDOC_GetLogFile`/`GetLogFileContents` → `REDENDOC_*` |
| `qrenderdoc/Windows/PipelineState/*.cpp` | `RENDERDOC_NumVerticesPerPrimitive`/`PROFILEFUNCTION` → `REDENDOC_*`（D3D11、D3D12、GL、Vulkan、PipelineStateViewer） |
| `qrenderdoc/Widgets/ReplayOptionsSelector.cpp` | `RENDERDOC_OpenCaptureFile` → `REDENDOC_OpenCaptureFile` |
| `qrenderdoc/Windows/PixelHistoryView.cpp` | `RENDERDOC_VertexOffset` → `REDENDOC_VertexOffset` |
| `qrenderdoc/Windows/ShaderMessageViewer.cpp` | `RENDERDOC_VertexOffset` → `REDENDOC_VertexOffset` |
| `qrenderdoc/CMakeLists.txt` | `RENDERDOC_STABLE_BUILD` → `REDENDOC_STABLE_BUILD`（qmake 宏定义） |
| `qrenderdoc/Resources/qrenderdoc.rc` | `RENDERDOC_VERSION_MAJOR/MINOR` → `REDENDOC_VERSION_MAJOR/MINOR`（PE 版本资源）；UI 文本 `"RenderDoc"` → `"RedenDoc"` |
| `qrenderdoc/renderdocui_stub.cpp` / `.vcxproj` | 项目名和 exe 名 |

#### 7.11 renderdoccmd 命令行工具

| 文件 | 说明 |
|------|------|
| `renderdoccmd/renderdoccmd.rc` | `RENDERDOC_VERSION_MINOR` → `REDENDOC_VERSION_MINOR`（PE 版本资源中遗漏的版本号宏） |
| `renderdoccmd/renderdoccmd.cpp` | 所有 `RENDERDOC_*` 导出函数调用 → `REDENDOC_*`，包括 `GetCommitHash`、`ExecuteAndInject`、`InjectIntoProcess`、`OpenCaptureFile`、`PreviewWindowCallback`、`BecomeRemoteServer`、`CreateRemoteServerConnection`、`RunUnitTests`、`RunFunctionalTests`、`SetDebugLogFile`、`NeedVulkanLayerRegistration`、`UpdateVulkanLayerRegistration`、`GetDefaultCaptureOptions`、`InitialiseReplay`、`ShutdownReplay` |

---

## 修改分类总结

| 优先级 | 检测向量 | 检测手段 | 隐藏策略 | 状态 |
|--------|---------|---------|---------|------|
| **P0** | DLL/EXE 模块名 | `GetModuleHandle` / 模块枚举 | `renderdoc.dll` → `redendoc.dll`（通过 `RDOC_BASE_NAME` 宏） | ✅ |
| **P0** | DLL 导出符号 | `GetProcAddress` / 导出表遍历 | `RENDERDOC_GetAPI` → `REDENDOC_GetAPI` 等 | ✅ |
| **P1** | Vulkan 层名称 | `vkEnumerateInstanceLayerProperties` | `VK_LAYER_RENDERDOC_Capture` → `VK_LAYER_REDENDOC_Capture` | ✅ |
| **P1** | Vulkan 环境变量 | `getenv` | `ENABLE_VULKAN_RENDERDOC_CAPTURE` → `ENABLE_VULKAN_REDENDOC_CAPTURE` | ✅ |
| **P1** | Windows 命名对象 | 枚举事件/管道/共享内存 | 事件名、管道名、共享内存名全部重命名 | ✅ |
| **P1** | OpenGL 窗口类名 | `FindWindow` / 窗口枚举 | `renderdocGLclass` → `redendocGLclass` | ✅ |
| **P1** | 文件路径/注册表 | 文件系统/注册表扫描 | 所有平台路径和注册表项中的 `renderdoc` → `redendoc` | ✅ |
| **P1** | PE 版本信息 | 读取 PE 资源 | `FileDescription`/`ProductName`/`InternalName`/`OriginalFilename` 全部改为 RedenDoc | ✅ |
| **P2** | 内部符号残留 | PDB/RTTI/字符串化宏 | 全面清理所有 `RENDERDOC_` 前缀的内部宏、类型、回调 | ✅ |
| **P2** | Linux 符号导出 | `dlsym` / 符号表扫描 | `.version` 文件中的导出模式更新 | ✅ |
| **P2** | Android 包名/库名 | 包名枚举 | `RENDERDOC_ANDROID_PACKAGE_BASE` 等宏更新 | ✅ |
