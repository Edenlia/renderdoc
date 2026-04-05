# F1-Edenlia-renderdoc_api_hide ChangeLog3

> **Feature 目标**：对 RenderDoc 进行反检测改造，使其在目标进程中不易被反作弊系统发现。
>
> **背景**：ChangeLog1 已完成对 C++ 代码中所有 `RENDERDOC_` 前缀的重命名（包括 D3D11/D3D12 driver 中引用 shader 入口点的字符串），但 **shader 源码文件（HLSL / GLSL）中的函数定义和变量名仍然保持原始的 `RENDERDOC_` 前缀**，导致运行时 shader 编译失败（找不到入口点函数），返回 NULL，进而引发崩溃。
>
> **核心策略**：将所有 shader 源码文件中的 `RENDERDOC_` 前缀统一替换为 `REDENDOC_`，使 shader 入口点名称与 C++ 代码中的引用完全匹配。

---

## 一、问题根因分析

### 崩溃现象

D3D11/D3D12 driver 在初始化时调用 `D3DCompile` 编译 HLSL shader，并指定 `REDENDOC_*` 作为入口点名称（ChangeLog1 已将 C++ 侧的字符串改为 `REDENDOC_*`），但 shader 源码中的函数定义仍为 `RENDERDOC_*`，导致编译器找不到入口点，返回 NULL shader blob，后续使用时崩溃。

### 不一致状态

| 层级 | 当前状态 | 问题 |
|------|---------|------|
| C++ 代码（D3D11 driver） | 已改为 `REDENDOC_*`（`d3d11_debug.cpp` 约 30 处、`d3d11_rendertext.cpp` 4 处、`d3d11_rendertexture.cpp` 9 处） | ✅ 已完成（ChangeLog1） |
| C++ 代码（D3D12 driver） | 已改为 `REDENDOC_*`（`d3d12_debug.cpp` 约 32 处、`d3d12_rendertext.cpp` 2 处、`d3d12_overlay.cpp` 2 处、`d3d12_manager.cpp` 13 处、`d3d12_rendertexture.cpp` 9 处） | ✅ 已完成（ChangeLog1） |
| C++ 代码（GL driver） | 已改为 `REDENDOC_Fixed_Color`（`gl_overlay.cpp` 2 处） | ✅ 已完成（ChangeLog1） |
| HLSL shader 源码 | **仍为 `RENDERDOC_*`**（13 个文件，50 个入口点函数） | ❌ **本次修复** |
| GLSL shader 源码 | **仍为 `RENDERDOC_Fixed_Color`**（1 个文件，2 处引用） | ❌ **本次修复** |

---

## 二、HLSL Shader 入口点函数重命名

### 修改原理

HLSL shader 文件中的函数名是 `D3DCompile` 的入口点参数，C++ 代码通过字符串指定入口点名称来编译 shader。两侧必须完全匹配。

### 具体修改

#### 2.1 text.hlsl（3 处）

| 原函数名 | 新函数名 | C++ 调用方 |
|----------|---------|-----------|
| `RENDERDOC_TextVS` | `REDENDOC_TextVS` | `d3d11_rendertext.cpp`、`d3d12_rendertext.cpp` |
| `RENDERDOC_Text9VS` | `REDENDOC_Text9VS` | `d3d11_rendertext.cpp`（FL9 模式） |
| `RENDERDOC_TextPS` | `REDENDOC_TextPS` | `d3d11_rendertext.cpp`、`d3d12_rendertext.cpp` |

#### 2.2 misc.hlsl（6 处）

| 原函数名 | 新函数名 | C++ 调用方 |
|----------|---------|-----------|
| `RENDERDOC_FullscreenVS` | `REDENDOC_FullscreenVS` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_FixedColPS` | `REDENDOC_FixedColPS` | `d3d11_debug.cpp` |
| `RENDERDOC_CheckerboardPS` | `REDENDOC_CheckerboardPS` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_DiscardFloatPS` | `REDENDOC_DiscardFloatPS` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_DiscardIntPS` | `REDENDOC_DiscardIntPS` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_ExecuteIndirectPatchCS` | `REDENDOC_ExecuteIndirectPatchCS` | `d3d12_debug.cpp` |

#### 2.3 mesh.hlsl（6 处）

| 原函数名 | 新函数名 | C++ 调用方 |
|----------|---------|-----------|
| `RENDERDOC_MeshVS` | `REDENDOC_MeshVS` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_TriangleSizeGS` | `REDENDOC_TriangleSizeGS` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_TriangleSizePS` | `REDENDOC_TriangleSizePS` | `d3d11_debug.cpp` |
| `RENDERDOC_MeshGS` | `REDENDOC_MeshGS` | `d3d12_debug.cpp` |
| `RENDERDOC_MeshPS` | `REDENDOC_MeshPS` | `d3d12_debug.cpp` |
| `RENDERDOC_MeshPickCS` | `REDENDOC_MeshPickCS` | `d3d11_debug.cpp`（结果被截断，实际存在） |

#### 2.4 texdisplay.hlsl（2 处）

| 原函数名 | 新函数名 | C++ 调用方 |
|----------|---------|-----------|
| `RENDERDOC_TexDisplayVS` | `REDENDOC_TexDisplayVS` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_TexDisplayPS` | `REDENDOC_TexDisplayPS` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |

#### 2.5 texremap.hlsl（3 处）

| 原函数名 | 新函数名 | C++ 调用方 |
|----------|---------|-----------|
| `RENDERDOC_TexRemapFloat` | `REDENDOC_TexRemapFloat` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_TexRemapUInt` | `REDENDOC_TexRemapUInt` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_TexRemapSInt` | `REDENDOC_TexRemapSInt` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |

#### 2.6 histogram.hlsl（3 处）

| 原函数名 | 新函数名 | C++ 调用方 |
|----------|---------|-----------|
| `RENDERDOC_TileMinMaxCS` | `REDENDOC_TileMinMaxCS` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_ResultMinMaxCS` | `REDENDOC_ResultMinMaxCS` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_HistogramCS` | `REDENDOC_HistogramCS` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |

#### 2.7 multisample.hlsl（6 处）

| 原函数名 | 新函数名 | C++ 调用方 |
|----------|---------|-----------|
| `RENDERDOC_CopyMSToArray` | `REDENDOC_CopyMSToArray` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_FloatCopyMSToArray` | `REDENDOC_FloatCopyMSToArray` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_DepthCopyMSToArray` | `REDENDOC_DepthCopyMSToArray` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_CopyArrayToMS` | `REDENDOC_CopyArrayToMS` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_FloatCopyArrayToMS` | `REDENDOC_FloatCopyArrayToMS` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |
| `RENDERDOC_DepthCopyArrayToMS` | `REDENDOC_DepthCopyArrayToMS` | `d3d11_debug.cpp`、`d3d12_debug.cpp` |

#### 2.8 depth_copy.hlsl（4 处）

| 原函数名 | 新函数名 | C++ 调用方 |
|----------|---------|-----------|
| `RENDERDOC_DepthCopyPS` | `REDENDOC_DepthCopyPS` | `d3d11_debug.cpp` |
| `RENDERDOC_DepthCopyArrayPS` | `REDENDOC_DepthCopyArrayPS` | `d3d11_debug.cpp` |
| `RENDERDOC_DepthCopyMSPS` | `REDENDOC_DepthCopyMSPS` | `d3d11_debug.cpp` |
| `RENDERDOC_DepthCopyMSArrayPS` | `REDENDOC_DepthCopyMSArrayPS` | `d3d11_debug.cpp` |

#### 2.9 quadoverdraw.hlsl（2 处）

| 原函数名 | 新函数名 | C++ 调用方 |
|----------|---------|-----------|
| `RENDERDOC_QuadOverdrawPS` | `REDENDOC_QuadOverdrawPS` | `d3d11_debug.cpp`、`d3d12_overlay.cpp` |
| `RENDERDOC_QOResolvePS` | `REDENDOC_QOResolvePS` | `d3d11_debug.cpp` |

#### 2.10 shaderdebug.hlsl（3 处）

| 原函数名 | 新函数名 | C++ 调用方 |
|----------|---------|-----------|
| `RENDERDOC_DebugMathOp` | `REDENDOC_DebugMathOp` | `d3d12_debug.cpp` |
| `RENDERDOC_DebugSampleVS` | `REDENDOC_DebugSampleVS` | `d3d12_debug.cpp` |
| `RENDERDOC_DebugSamplePS` | `REDENDOC_DebugSamplePS` | `d3d12_debug.cpp` |

#### 2.11 pixelhistory.hlsl（3 处）

| 原函数名 | 新函数名 | C++ 调用方 |
|----------|---------|-----------|
| `RENDERDOC_PixelHistoryUnused` | `REDENDOC_PixelHistoryUnused` | `d3d11_debug.cpp` |
| `RENDERDOC_PixelHistoryCopyPixel` | `REDENDOC_PixelHistoryCopyPixel` | `d3d11_debug.cpp` |
| `RENDERDOC_PrimitiveIDPS` | `REDENDOC_PrimitiveIDPS` | `d3d11_debug.cpp` |

#### 2.12 d3d12_pixelhistory.hlsl（4 处）

| 原函数名 | 新函数名 | C++ 调用方 |
|----------|---------|-----------|
| `RENDERDOC_PixelHistoryUnused` | `REDENDOC_PixelHistoryUnused` | `d3d12_debug.cpp` |
| `RENDERDOC_PixelHistoryCopyPixel` | `REDENDOC_PixelHistoryCopyPixel` | `d3d12_debug.cpp` |
| `RENDERDOC_PrimitiveIDPS` | `REDENDOC_PrimitiveIDPS` | `d3d12_debug.cpp` |
| `RENDERDOC_PixelHistoryFixedColPS` | `REDENDOC_PixelHistoryFixedColPS` | `d3d12_debug.cpp` |

#### 2.13 raytracing.hlsl（6 处）

| 原函数名 | 新函数名 | C++ 调用方 |
|----------|---------|-----------|
| `RENDERDOC_PatchAccStructAddressCS` | `REDENDOC_PatchAccStructAddressCS` | `d3d12_manager.cpp` |
| `RENDERDOC_PatchShaderTableCS` | `REDENDOC_PatchShaderTableCS` | `d3d12_manager.cpp` |
| `RENDERDOC_CopyShaderTableCS` | `REDENDOC_CopyShaderTableCS` | `d3d12_manager.cpp` |
| `RENDERDOC_PrepareRayIndirectExecuteCS` | `REDENDOC_PrepareRayIndirectExecuteCS` | `d3d12_manager.cpp` |
| `RENDERDOC_PrepareTLASCopyIndirectExecuteCS` | `REDENDOC_PrepareTLASCopyIndirectExecuteCS` | `d3d12_manager.cpp` |
| `RENDERDOC_CopyBLASInstanceCS` | `REDENDOC_CopyBLASInstanceCS` | `d3d12_manager.cpp` |

---

## 三、GLSL Shader uniform 变量重命名

### 修改原理

GL driver 通过 `glGetUniformLocation(prog, "REDENDOC_Fixed_Color")` 查找 uniform 变量位置（ChangeLog1 已将 C++ 侧的字符串改为 `REDENDOC_Fixed_Color`），但 GLSL shader 源码中的 uniform 声明仍为 `RENDERDOC_Fixed_Color`，导致查找失败。

### 具体修改

#### 3.1 fixedcol.frag（2 处）

| 修改点 | 文件 | 说明 |
|--------|------|------|
| uniform 声明 | `renderdoc/data/glsl/fixedcol.frag` | `uniform vec4 RENDERDOC_Fixed_Color;` → `uniform vec4 REDENDOC_Fixed_Color;` |
| uniform 使用 | `renderdoc/data/glsl/fixedcol.frag` | `color_out = RENDERDOC_Fixed_Color;` → `color_out = REDENDOC_Fixed_Color;` |

**C++ 调用方**：`gl_overlay.cpp` 中 2 处 `glGetUniformLocation(prog, "REDENDOC_Fixed_Color")`

---

## 四、不涉及修改的文件

以下 shader 相关文件经确认不包含 `RENDERDOC_` 前缀标识符，无需修改：

| 文件 | 原因 |
|------|------|
| `renderdoc/data/hlsl/hlsl_cbuffers.h` | cbuffer 成员变量名（如 `TexDim`、`MipLevel`）不带 `RENDERDOC_` 前缀 |
| `renderdoc/data/hlsl/hlsl_custom_prefix.h` | 使用 `RD_` 前缀而非 `RENDERDOC_` 前缀 |
| `renderdoc/data/hlsl/hlsl_texsample.h` | 不包含 `RENDERDOC_` 前缀标识符 |
| `renderdoc/data/hlsl/fixedcol.hlsl` | 不包含 `RENDERDOC_` 前缀标识符 |
| `renderdoc/data/hlsl/quadswizzle.hlsl` | 不包含 `RENDERDOC_` 前缀标识符 |
| 其他 GLSL 文件（除 `fixedcol.frag`） | 不包含 `RENDERDOC_` 前缀标识符 |

---

## 五、修改范围总结

| 类型 | 文件数 | 修改点数 | 修改方式 |
|------|--------|---------|---------|
| HLSL shader 入口点函数名 | 13 | 50 | 函数定义处 `RENDERDOC_` → `REDENDOC_`（仅改前缀，不改签名/参数/函数体） |
| GLSL shader uniform 变量名 | 1 | 2 | `RENDERDOC_Fixed_Color` → `REDENDOC_Fixed_Color` |
| **总计** | **14** | **52** | — |

---

## 与 ChangeLog1 / ChangeLog2 的关系

本次 ChangeLog3 是 ChangeLog1 的**配套修复**。ChangeLog1 第 7.4 节"图形 API 驱动内部符号"中已将 C++ 代码中引用 shader 入口点的字符串从 `RENDERDOC_*` 改为 `REDENDOC_*`，但遗漏了 shader 源码文件本身的同步修改，导致运行时 shader 编译失败。

| 层级 | ChangeLog1 | ChangeLog3 |
|------|-----------|-----------|
| C++ 调用方（shader 入口点字符串） | ✅ 已修改 | — |
| HLSL shader 源码（函数定义） | ❌ 遗漏 | ✅ 本次修复 |
| GLSL shader 源码（uniform 变量） | ❌ 遗漏 | ✅ 本次修复 |
