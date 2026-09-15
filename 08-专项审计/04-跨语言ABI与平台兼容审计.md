# 跨语言 ABI 契约与平台兼容专项审计

> 本报告各条的**严重度见各 Finding 的「复核结论」字段**（16 条：0 条判定为非缺陷、10 条级别调整）。





> 分析范围：`Plugins/UnrealCSharp` 的 **跨语言边界**（C++ ↔ C# 的 P/Invoke、`[UnmanagedCallersOnly]`、托管函数指针桥）+ 多平台（Win64 / Linux / Mac / Android / iOS）差异。覆盖 `Source/` 全部 7 个模块 与 `Script/` 全部 C#。
> 排除：`Source/ThirdParty/`、`Intermediate/`、`Binaries/`、`*.old`、`Script/**/obj/`、`Script/**/bin/`。
> 审计范围：跨语言 ABI 契约与平台兼容（C++↔C# 配对表 A-D / 平台分支 / UB 与不安全模式 / 后端门控表）
> 覆盖情况：C++↔C# 配对表 A/B 完成；配对表 C/D 完成；平台分支与 ABI 缺陷完成；用户工程内**生成**的 `Script/Binding/**`（引擎类型绑定）无法核对，见第 9 节。
> 说明：§3 中编号 **F-ABI-030 及以后**的条目（平台/编译器专项、UB/安全专项）中未能独立核实者，均在"置信度"字段中标为**中/低**并写明未验证点。权威引擎事实的取证目录为 `<Engine 5.6 安装目录>`（下文以"引擎侧"标注的引证均为此目录下的第一手读取）。

---

## 0. 覆盖范围与阅读清单

### 0.1 完整读完的文件

| 文件 | 行数 | 是否读完 | 备注 |
|---|---|---|---|
| `Source/UnrealCSharpCore/Public/Domain/Script/IScriptTypes.h` | 182 | 是 | **桥接 typedef 总表（44 条）** |
| `Source/UnrealCSharpCore/Public/Domain/Script/FScriptDomainImpl.inl` | 652 | 是 | 全部桥接调用点 |
| `Source/UnrealCSharpCore/Public/Domain/Script/IManagedHandle.h` | 40 | 是 | 跨边界值类型定义 |
| `Source/UnrealCSharpCore/Public/Domain/Script/IScriptDomain.h` | — | 部分（grep） | 虚接口 |
| `Source/UnrealCSharpCore/Public/Domain/LeanCLR/FLeanCLRDomain.h` | 227 | 是 | LeanCLR 方法表 + PInvoke 分派声明 |
| `Source/UnrealCSharpCore/Public/Domain/LeanCLR/FLeanCLRDomain.inl` | 79 | 是 | `Bridge_Invoke` 全实现 |
| `Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp` | 889 | 是 | 含 `PInvoke_Classify` / `PInvoke_Dispatch` |
| `Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp` | 520 | 是 | Mono 侧桥接解析 |
| `Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRDomain.cpp` | 312 | 是 | CoreCLR 侧桥接解析 |
| `Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRFunctionLibrary.cpp` | 41 | 是 | hostfxr 路径 |
| `Source/UnrealCSharpCore/Private/Domain/Mono/FMonoFunctionLibrary.cpp` | 88 | 是 | mono 目录（平台分支） |
| `Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRFunctionLibrary.cpp` | 42 | 是 | leanclr 目录（平台分支） |
| `Source/UnrealCSharpCore/Public/CoreMacro/Macro.h` | 133 | 是 | `LIB_HOSTFXR` 平台分支 |
| `Source/UnrealCSharpCore/Public/CoreMacro/ClassMacro.h` | 111 | 是 | 桥接类名宏 |
| `Source/UnrealCSharpCore/Public/CoreMacro/FunctionMacro.h` | 133 | 是 | 桥接方法名宏 |
| `Source/UnrealCSharpCore/Public/CoreMacro/BindingMacro.h` | 19 | 是 | `__%s_%sImplementation` 命名 |
| `Source/UnrealCSharpCore/Public/CoreMacro/BufferMacro.h` | 29 | 是 | 缓冲参数宏 |
| `Source/UnrealCSharpCore/Public/Binding/Function/FBindingMethod.h` | 27 | 是 | 注册项 |
| `Source/UnrealCSharpCore/Private/Binding/Class/FBindingClassRegister.cpp` | 141 | 是 | **注册名拼接** |
| `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp` | 1615 | 部分（1–130、980–1110） | dotnet 路径、发布路径 |
| `Source/UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp` | 381 | 部分（1–249） | dotnet SDK 探测平台分支 |
| `Source/UnrealCSharpCore/Private/Reflection/FFieldReflection.cpp` | 32 | 是 | `SetFieldStaticValue` 唯一调用点 |
| `Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp` | 796 | 部分（400–519 + grep） | 描述符缓冲槽位数 |
| `Source/UnrealCSharpCore/Private/Bridge/FTypeBridge.cpp` | 605 | 是 | `GetClass` 分派 |
| `Source/UnrealCSharpCore/Private/Domain/Script/FScriptLog.cpp` | 27 | 是 | 唯一的 DllImport 目标 |
| `Source/UnrealCSharp/Private/Domain/FDomain.cpp` | 155 | 部分（95–155） | `GetTraceback` |
| `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp` | 636 | 部分（1–150） | **信号处理器平台分支** |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegister*.cpp` | 约 40 个文件 | 逐个 grep + 14 个精读 | 绑定注册 ↔ C# 声明核对 |
| `Script/Interop/**/*.cs` | 9 个文件 | 全部读完 | **回调方向 ABI** |
| `Script/UE/Library/*.cs` | 26 个文件 | 全部读完/grep | **P/Invoke 声明侧** |
| `Script/UE/CoreUObject/Utils.cs` | 775 | 部分（1–140、220–320、600–775） | 描述符协议 |
| `Script/UE/CoreUObject/SynchronizationContext.cs` | 93 | 是 | Tick 回调 |
| `Script/SourceGenerator/UnrealTypeSourceGenerator.cs` | 967 | 部分（800–967） | **生成 `[DllImport]` / 函数指针调用** |
| `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp` | 467 | 部分（180–339） | `WITH_LEANCLR` 下发给 C# |
| `Source/ScriptCodeGenerator/Private/FBindingClassGenerator.cpp` | 1012 | 部分（790–969） | 生成的 partial 声明形状 |
| `Source/ScriptCodeGenerator/Private/FGeneratorCore.cpp` | 1079 | 部分（430–609） | 缓冲尺寸/类型映射 |
| `Source/{UnrealCSharp,UnrealCSharpCore}/*.Build.cs` | 67 / 348 | 是 | 定义与依赖 |

### 0.2 未读完 / 未覆盖

- `Script/Binding/**`（**由 `ScriptCodeGenerator` 生成到用户工程**、不在插件仓库内）：`glob` 与 `pwsh Get-ChildItem` 已确认插件目录下不存在该目录，因此**引擎类型（`FVector`、`AActor` 等）的 `__X_YImplementation` 声明本体无法在此仓库内核对**。
- `Source/ThirdParty/`（按要求排除）。仅在被审计代码的**契约依赖**需要时读取了 2 个头文件（`LeanCLR/src/runtime/vm/pinvoke.h`、`Mono/src/mono/utils/details/mono-error-types.h`），用于判定 ABI 契约，未做第三方代码分析。
- `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp` 只读了 1–130 与 980–1110 行（共 1615 行）。
- `Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp` 只读了 400–519 行 + 针对 `ArrayGet`/`Free` 的 grep（共 796 行）。
- **引擎源码本机确实存在**（`Engine\Source`，约 20989 个 `.cpp`）——**本行原文"本机没有 UE 引擎源码"是错的，已就地改正**。凡涉及"UE 头文件里 `TCHAR`/`FBoolProperty::ElementSize` 的定义""UBT 是否为 MSVC 加 `/utf-8`""Android 允许哪些 ABI"的结论，均已回引擎源码取证（见各条 `复核证据` 字段；其中 F-ABI-022 因此由 P3 改判为**撤销（非缺陷）**）。仍未第一手核对的引擎行为：UBT 的**模块规则发现**通配符调用点（见 F-ABI-021 复核证据）。

---

## 1. 模块职责与架构速览（跨语言边界的真实形状）

这个插件的 C++ ↔ C# 边界**不是**教科书式的 `extern "C"` + `DllImport`。本仓库内 **没有任何一个 `extern "C"` 导出**（`grep 'extern "C"' Source/` 在排除 ThirdParty 后 **0 命中**），边界由两套机制构成：

### 机制 A：桥接 typedef 表（C++ → C# 直接函数指针调用）

- C++ 侧在 `IScriptTypes.h:5-105` 手写 44 个函数指针 typedef，`COMMON_BRIDGE_METHODS`(:107-156) / `NATIVE_BRIDGE_METHODS`(:158-166) / `UTILS_BRIDGE_METHODS`(:168-175) 把 `(成员名, typedef, C#类名宏, C#方法名宏, 参数个数)` 表达成表，`SCRIPT_TYPES`(:179-182) 展开成 44 个静态函数指针成员。
- 启动时按后端解析：
  - Mono：`FMonoDomain::RegisterInterop`(:474-510) 用 `mono_class_from_name` + `mono_class_get_method_from_name(类, 方法名, 参数个数)` + `mono_method_get_unmanaged_callers_only_ftnptr` 取指针（宏 `GET_METHOD_AND_GET_FUNCTION_POINTER`，:34-39）。
  - CoreCLR：`FCoreCLRDomain::RegisterInterop`(:281-294) 用 `load_assembly_and_get_function_pointer(装配件, "Interop.TypeBridge, Interop", "GetClass", UNMANAGEDCALLERSONLY_METHOD, ...)`（`CORECLR_TYPE_NAME`）。
  - LeanCLR：`FLeanCLRDomain::RegisterInterop`(:834-852) 用 `RtModuleDef::get_class_by_nested_full_name` + `find_matched_method_in_class_by_name` 拿 `RtMethodInfo*`，再经 `Bridge_Invoke`（`FLeanCLRDomain.inl:74`）用解释器栈调用。
- **`参数个数` 这一列是唯一的签名校验**（Mono 用它选重载；CoreCLR/LeanCLR 完全不用——CoreCLR 只按方法名字符串找）。

### 机制 B：绑定注册 + 命名约定（C++ 绑定函数 ← C# 生成的 partial）

- C++ 绑定函数（`FRegister*.cpp` 的 `static XXX YYYImplementation(...)`）经 `FBindingClassRegister::BindingMethod`（`FBindingClassRegister.cpp:113-127`）拼成注册名：
  `BINDING_COMBINE_CLASS(ImplementationNameSpace, BINDING_COMBINE_CLASS_IMPLEMENTATION(GetClass())) + "::" + BINDING_COMBINE_FUNCTION_IMPLEMENTATION(GetClass(), InImplementationName)`
  → 形如 `Script.Library.UnrealImplementation::__Unreal_NewObjectImplementation`。
- C# 侧 `Script/UE/Library/*.cs` 手写 **209** 个 `private static unsafe partial <ret> __<Class>_<Method>Implementation(...)`（**复核实测，原文"211"偏高 2**；grep 模式见 §4.2 与 F-ABI-004 复核证据）；`UnrealTypeSourceGenerator.EmitBridges`（`UnrealTypeSourceGenerator.cs:873-927`）为它们生成实现体，键为 `$"{namespace}.{owner.Name}::{method.Name}"`(:903)，与上面 C++ 拼出的字符串**逐字符相同**。
- 生成的实现体分两条路：
  - `WITH_LEANCLR`：`[DllImport("UnrealCSharp", CallingConvention = Cdecl)] static extern unsafe partial ...`(:910-912)；
  - 否则：`((delegate* unmanaged[Cdecl]<...>)Interop.MethodBridge.GetMethod(ref slot, "key"))(args)`(:916-918)。

### 机制 C：反向（C++ → C#）委托

- `MethodBridge.Invoke`（`MethodBridge.cs:32-138`）以反射方式调用托管方法，参数从原生缓冲解析（值类型按 `ReadPrimitiveValue` 逐类型指针读，引用类型按句柄查 `HandleData`）。
- 字符串：`StringBridge.GetString` 输出 UTF-16(`char16_t*`)，`TypeBridge.Get*` 输出 UTF-8(`byte*`) —— **两套编码并存**，这是本报告多条发现的根源。

**关键生命周期**：`Assemblies` / 桥接函数指针在 `UnloadAssembly()`（`FScriptDomainImpl` 各后端 :428-458 / :240-270 / :806-832）里被置空（`COMMON_BRIDGE_METHODS(...CLEAR)`）；`HandleData`/`TypeBridge` 的静态字典由 `AssemblyLoader.Unload`(`AssemblyLoader.cs:31-61`) 的 `HandleData.Clear()`/`TypeBridge.Clear()` 清理。委托保活见第 5 节。

---

## 2. 关键调用链

1. **P/Invoke 声明解析失败 → 空指针直呼**
   `UnrealTypeSourceGenerator.cs:916-918` → `MethodBridge.GetMethod`(`MethodBridge.cs:140-148`，找不到返回 `0`) → `((delegate* unmanaged[Cdecl]<...>)0)(...)` → 崩溃。
2. **字符串读取（UTF-8 缓冲太小 → 异常穿出托管 → 杀进程）**
   `FScriptDomainImpl.inl:27-36 GetNamespace` → `TGetUTF8String`(`TGetUtf8String.inl:14`，缓冲 512B) → `TypeBridgeGetNamespaceFn` → `TypeBridge.GetNamespace`(`TypeBridge.cs:101-121`) → `Encoding.UTF8.GetBytes(...)` 抛 `ArgumentException`。
3. **TMap 泛型类型构造（值类型被丢弃）**
   `FTypeBridge.cpp:561-575 GetClass(FMapProperty*)` → `MakeGenericTypeInstance` → `FScriptDomainImpl.inl:453-463` → `TypeBridgeMakeGenericType2Fn` → `TypeBridge.MakeGenericType2`(`TypeBridge.cs:188-202`) → 用 `InKeyType` 查了两次。
4. **枚举值反射（C# 抛异常穿出 `[UnmanagedCallersOnly]`）**
   `FDynamicEnumGenerator.cpp:281-295` → `IScriptDomain::GetFieldStaticValue`(`FScriptDomainImpl.inl:406-412`) → `FieldBridgeGetStaticValueFn` → `FieldBridge.GetStaticValue`(`FieldBridge.cs:22-32`) → `Convert.ToInt64`/`Convert.ChangeType`。
5. **程序集加载（异常穿出 `[UnmanagedCallersOnly]`）**
   `FCoreCLRDomain.cpp:226-235 LoadAssembly` → `AssemblyLoaderLoadFromStreamFn` → `AssemblyLoader.LoadFromStream`(`AssemblyLoader.cs:13-28`) → `Context.LoadFromStream(...)` 无 try/catch。
6. **静态字段写入（32 位与 64 位契约错配）**
   `FCSharpBind.cpp:172/224/362` → `FFieldReflection::SetValue`(:30) → `FScriptDomainImpl.inl:392-403`（`*static_cast<uint32*>(InValue)`）→ `FieldBridgeSetStaticValueFn` → `FieldBridge.SetStaticValue(nint, byte*, long)`。
7. **信号处理器（平台分支语义不等价）**
   `FCSharpEnvironment.cpp:101-142` → `SignalHandler`(:22-31) → `FDomain::GetTraceback()`(`FDomain.cpp:112-129`) → 托管 `Utils.GetTraceback`（在信号处理上下文里进托管运行时）。
8. **生成解决方案的 `WITH_LEANCLR` 下发**
   `FSolutionGenerator.cpp:207-223 ReplaceDefineConstants` → `FUnrealCSharpFunctionLibrary::GetScriptDomainType()` → 写入 `Script/Shared.props` 的 `<DefineConstants>` → C# 侧 `#if WITH_LEANCLR` 决定用 DllImport 还是函数指针。

---

## 3. 发现清单

> **阅读本节前必读——后端门控（backend gating）**：本插件有 Mono / CoreCLR / LeanCLR 三个互斥的脚本后端，由 `UnrealCSharpCore.build.cs:275-306` 按 `Config/DefaultUnrealCSharpSetting.ini` 的 `<Platform>ScriptDomainType` 决定 `WITH_MONO` / `WITH_CORECLR` / `WITH_LEANCLR`。我第一手核实该 ini 当前为**五个平台全部 `LeanCLR`**（详见 §9 第 10 条），因此：
>
> | 归类 | 发现 | 当前工程配置下 |
> |---|---|---|
> | **与后端无关（活）** | F-ABI-001/002/003/004/005/007/008/009/013/016/017/018/019/020/021/022/023/024/025/026/027/028/030/031/032/033/034/035/036/037 | **直接生效**（C# 侧与后端无关的代码 + 三个后端共用的 C++ 路径） |
> | **LeanCLR 专属（活）** | F-ABI-014、F-ABI-015 | **直接生效**（`WITH_LEANCLR=1`） |
> | **Mono 专属（潜伏）** | F-ABI-006、F-ABI-010 | `WITH_MONO=0` → `FMonoDomain.cpp`/`FMonoFunctionLibrary.cpp` **不编译**；切回 Mono 后端即生效 |
> | **CoreCLR 专属（潜伏）** | F-ABI-011 的 `GetDotNet()` 次要分支、F-ABI-012、F-ABI-039、F-ABI-029 的 CoreCLR 分支 | `WITH_CORECLR=0` → `FCoreCLR*.cpp` **不编译**；切回 CoreCLR 后端即生效 |
> | **模块无关（活，编辑器侧）** | F-ABI-011 的 `GetDotNetPathArray()`、F-ABI-036 | `UnrealCSharpCore`/`ScriptCodeGenerator` 始终编译 |
>
> 严重度字段按"一旦该代码被编译即为真"的口径给出，不随当前 ini 下调；上表只用于判断**优先级**。
>
> **⚠️ 复核对后端门控的补正（重要）**：上表"与后端无关（活）"这一归类**在"后果"层面不准确**，必须按下述方式重读：
> 1. **"托管异常穿出 `[UnmanagedCallersOnly]` → 杀进程"这个后果只在 Mono/CoreCLR 成立**。当前工程五平台全部 LeanCLR，而 LeanCLR 下 C++→C# **不走本机函数指针**：桥接调用经 `SCRIPT_DOMAIN_INVOKE`（`FLeanCLRDomain.cpp:99` 重定义为 `Bridge_Invoke`）→ `Runtime_Invoke` → 失败时 `Unhandled_Exception(Exception)`（`FLeanCLRDomain.cpp:306-320`，`:339-349` 只做 `FLeanCLRLog::ErrorWriter` 日志）→ 返回 false → `Bridge_Invoke_Helper` 拿到**零值** `OutReturn`（`FLeanCLRDomain.inl:58-70`）。因此 **F-ABI-001 / 003 / 007 / 017 / 030 / 033(前置崩溃) / 037 的"终止进程"表述在当前后端下不成立**，其后果退化为"日志 + 零值返回/静默错误值"（各条 `复核证据` 已逐条写明；F-ABI-001、F-ABI-030 因而已由 P1 下调为 P2）。
> 2. 上表**漏列 2 条**：**F-ABI-040**（`TCompoundPropertyDescriptor::CopyValue` 按 `ElementSize` 分配）与 **F-ABI-038**（本报告无此编号——编号在 037→039 之间不连续，非缺漏）。F-ABI-040 属**与后端无关（活）**。
> 3. 表内标注"F-ABI-011 的 `GetDotNet()` 次要分支"与"F-ABI-011 的 `GetDotNetPathArray()`"的分裂是**正确的**：前者在 Linux 上返回 macOS 路径（编辑器公共代码），后者只在 Linux 编辑器构建里暴露空函数体。
> 4. **39 条发现现已全部具备 `复核结论`/`可达性`/`复核证据`/`级别变动` 字段**（仅 16 条有），不再有"未复核"条目。

### P0

### [F-ABI-001] `TypeBridge.GetNamespace/GetName/GetFullName` 用 UTF-16 字符数当 UTF-8 字节数上限，缓冲不足时抛异常穿出 `[UnmanagedCallersOnly]` 直接杀进程

- **类别**: Bug / 平台兼容（编码）
- **严重度**: **P1**
- **复核结论**: 部分确认（偏差：①"异常穿出 = 杀进程"仅 Mono/CoreCLR 成立——当前工程五平台全 LeanCLR，异常在 `FLeanCLRDomain.cpp:306-316` 被捕获并交给 `Unhandled_Exception` 记录后返回 false，调用方拿到零值，**进程不终止**；②`TGetUTF8String` 的采用判据是 `Length < 511` 才直接采用，不是原文写的"上一次返回长度 == 512-1 时才翻倍重试"）
- **可达性**: 活跃（`TypeBridge.Get*` 路径在当前后端确实执行 → 但后果退化为"返回 0 → `FString` 空串"，崩溃后果**潜伏**）
- **复核证据**: `Source/UnrealCSharpCore/Public/Template/TGetUtf8String.inl:14-28`（512 栈缓冲、`Length <= 0 → {}`、`Length < StackBufferSize-1` 才采用，否则 30-51 行翻倍重试）；`Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp:306-320`（`Runtime_Invoke` 失败 → `Unhandled_Exception(Exception)` → `return false`）；`FLeanCLRDomain.inl:58-70`（`OutReturn{}` 零初始化、`Runtime_Invoke` 的 bool 返回值被丢弃）
- **级别变动**: P1→P2（当前后端后果是"静默空串"而非进程终止，且触发需 >511 字节 UTF-8 的类型全名；切回 CoreCLR/Mono 恢复 P1）
- **文件**: `Script/Interop/Bridge/TypeBridge.cs:101-121`、`Script/Interop/Bridge/TypeBridge.cs:123-144`、`Script/Interop/Bridge/TypeBridge.cs:146-171`
- **函数**: `Interop.TypeBridge.GetNamespace(nint, byte*, int)` / `GetName(nint, byte*, int)` / `GetFullName(nint, byte*, int)`
- **置信度**: 高（BCL 行为为文档化行为；未实际运行验证）

**现状（代码事实）**
```csharp
// Script/Interop/Bridge/TypeBridge.cs:100-121
        [UnmanagedCallersOnly]
        public static unsafe int GetNamespace(nint InHandle, byte* OutString, int InStringSize)
        {
            if (OutString != null && InStringSize > 0)
            {
                if (HandleData.GetObject(InHandle) is Type Type)
                {
                    var Namespace = Type.Namespace ?? string.Empty;

                    var String = new Span<byte>(OutString, InStringSize);

                    var Length = Encoding.UTF8.GetBytes(
                        Namespace.AsSpan(0, Math.Min(Namespace.Length, InStringSize - 1)), String);

                    String[Length] = 0;

                    return Length;
                }
            }

            return 0;
        }
```
`GetName`(:123-144)、`GetFullName`(:146-171) 是同一份代码的复制粘贴；`GetFullName` 还额外把 `"{TypeFullName}, {AssemblyName}"` 一起编码(:157)，字符串更长。

**调用上下文**
C++ 侧 `FScriptDomainImpl.inl:27-36 / 39-49 / 52-70` 经 `TGetUTF8String`（`Source/UnrealCSharpCore/Public/Template/TGetUtf8String.inl:8-14`）传入 `uint8 StackString[512]`，只有在上一次返回长度 `== 512-1` 时才把缓冲翻倍重试（:32-51）。typedef 在 `IScriptTypes.h:23/25/27`，注册在 `COMMON_BRIDGE_METHODS`（`IScriptTypes.h:115-117`）。调用方是 `FClassReflection::EnsureDescriptorImplementation`(`FClassReflection.cpp:427`) 之后的 `ArrayGetString`/`ArrayGetClass` 路径与 `GetNamespace`/`GetName`（`FScriptDomainImpl.inl:568-570`）。

**问题**
`Math.Min(Namespace.Length, InStringSize - 1)` 用的是 **UTF-16 字符数**，而 `Encoding.UTF8.GetBytes(ReadOnlySpan<char>, Span<byte>)` 的目标容量是 **UTF-8 字节数**。两者的比例对非 ASCII 是 1:2（BMP 外）到 1:3（CJK）。当 `UTF8(名字) 字节数 > InStringSize` 时，`Encoding.UTF8.GetBytes` 抛 `ArgumentException`（"destination buffer too small"）。

具体触发路径（命名空间/类型名含中文，这是本插件**明确支持**的场景——`Source/UnrealCSharpCore/Public/Common/NameEncode.h:26-35` 的注释专门描述了中文名转义规则 `例 技能3 将会被转义为 _hu8062FD80_3`）：
- `Script.Game.技能管理器` → `GetFullName` 返回 `"Script.Game.技能管理器, Game"` ≈ 22 字符 → UTF-8 = 3×5 + 17 = 32 字节，512 缓冲够用；
- 但 `GetFullName` 的返回值包含**程序集名**，且 `Utils.GetClassDescriptor` 会把最多 15 个句柄交给 C++，随后 C++ 对每个泛型实参/接口都再调一次；真正危险的是**长命名空间 + 长类名 + 长程序集名**：例如 namespace `Game.战斗系统.技能系统.被动技能` + 类名 `高级被动技能管理器` + 程序集 `Game`，字符数 ≈ 40，UTF-8 ≈ 100 字节 → 仍在 512 内。
- 崩溃门槛是 UTF-8 字节数 > 512，即约 **170 个 CJK 字符**。UE 的蓝图类名上限 255 字符，`GetFullName` 输出 `Namespace.Name, AssemblyName`，中文项目里 200 字符合法类名完全可达（本插件的 `NameEncode` 与 `Utils.GetFullName` 正是为这类名字设计的）。

一旦抛异常：这三个方法都是 `[UnmanagedCallersOnly]`，**托管异常不允许穿出**；CoreCLR 的运行时行为是 fail-fast 终止进程（无法在 C++ 侧 catch）。而 C++ 侧 `TGetUTF8String` 的"缓冲翻倍重试"逻辑只在正常返回 `Length == InStringSize-1` 时才触发，异常路径下**永远不会重试**。

**建议**
```csharp
[UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
public static unsafe int GetNamespace(nint InHandle, byte* OutString, int InStringSize)
{
    try
    {
        if (OutString == null || InStringSize <= 0) return -1;          // -1 = 需要更大缓冲
        if (HandleData.GetObject(InHandle) is not Type Type) return -1;
        var Namespace = Type.Namespace ?? string.Empty;
        var Needed = Encoding.UTF8.GetByteCount(Namespace);
        if (Needed >= InStringSize) return Needed;                       // 返回"所需长度"，不写缓冲
        var Written = Encoding.UTF8.GetBytes(Namespace, new Span<byte>(OutString, InStringSize));
        OutString[Written] = 0;
        return Written;
    }
    catch (Exception Ex) { Console.Error.WriteLine(Ex); return -1; }
}
```
C++ 侧相应改为：`Length >= InStringSize` 时翻倍重试（当前判据 `Length < BufferSize - 1` 与新协议不兼容，需一起改）。**两个改动必须同时做**，否则只改一侧会引入新的不一致。

**验证方式**
- grep：`grep -n "Encoding.UTF8.GetBytes" Script/Interop/Bridge/TypeBridge.cs`（确认只有这 3 处 + `String[Length] = 0`）。
- 用例：在 `Content/Script/Game.dll` 里声明 `namespace Game { public class 测试类名...（>170 个 CJK 字符的完整类名） }`，触发 `Utils.GetClassDescriptor` → 观察进程是否 fail-fast。
- 断言：在 C++ `TGetUTF8String`(`TGetUtf8String.inl:14-19`) 里给返回值加 `check(Length < StackBufferSize)` 之前先 `ensureAlwaysMsgf(Length >= 0, ...)`。

---

### [F-ABI-002] 生成的桥接代码把 `MethodBridge.GetMethod` 的 0 返回值当作函数指针直接调用（无任何检查）

- **类别**: Bug
- **严重度**: **P2**
- **复核结论**: 确认（`GetMethod` 返回 0 时确实被当作函数指针调用，链上无 `if (Ptr == 0)`）
- **可达性**: 潜伏（**门控实证**：`UnrealTypeSourceGenerator.cs:909-919` 把函数指针调用整段放在 `#else` 分支——`WITH_LEANCLR` 时生成的是 `[DllImport("UnrealCSharp", CallingConvention = Cdecl)] static extern`(:910-912)，**该生成物在当前工程根本不产生**；切到 Mono/CoreCLR 才生效）
- **复核证据**: `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:909-919`（`#if WITH_LEANCLR` → DllImport；`#else` → `((delegate* unmanaged[Cdecl]<...>)global::Interop.MethodBridge.GetMethod(ref slot, "key"))(args)`）；`Script/Interop/Bridge/MethodBridge.cs:140-148`（逐字核对：`InSlot == nint.Zero` 且 `TryGetValue` 失败即返回 0）
- **级别变动**: 无（维持 P0→P2；潜伏依据由"源码推断"升级为"生成器宏分支实证"）
- **文件**: `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:914-918`（生成侧）、`Script/Interop/Bridge/MethodBridge.cs:140-148`（解析侧）
- **函数**: 生成的所有 `Script.Library.*` / `Script.Binding.*` partial 方法
- **置信度**: 高

**现状（代码事实）**
```csharp
// Script/SourceGenerator/UnrealTypeSourceGenerator.cs:914-918  （生成的代码文本）
                    $"\t\tprivate static nint {slot};\n" +
                    "\n" +
                    $"\t\t{accessibility} static unsafe partial {returnType} {method.Name}({parameters}) =>\n" +
                    $"\t\t\t((delegate* unmanaged[Cdecl]<{pointerType}>)global::Interop.MethodBridge.GetMethod(\n" +
                    $"\t\t\t\tref {slot}, \"{key}\"))({arguments});\n" +
```
```csharp
// Script/Interop/Bridge/MethodBridge.cs:140-148
        public static nint GetMethod(ref nint InSlot, string InName)
        {
            if (InSlot == nint.Zero)
            {
                InSlot = StringToMethod.TryGetValue(InName, out var Method) ? Method : nint.Zero;
            }

            return InSlot;
        }
```
`StringToMethod` 由 `RegisterBinding`(`MethodBridge.cs:12-30`) 从 C++ 侧填入；`FScriptDomainImpl.inl:601-633` 是唯一的填充点。

**调用上下文**
`StringToMethod` 只在 `IScriptDomain::RegisterBinding()` 里被填，注册名由 `FBindingClassRegister.cpp:113-127` 拼出。任一侧的名字/命名空间变化（例如把 `Script.Library.UnrealImplementation` 改名、把 `FClassBuilder(TEXT("Unreal"), NAMESPACE_LIBRARY)` 的类名改掉、或 `WITH_BINDING`/生成顺序变化导致 `FBinding::Get().Register()` 里少了一项）都会让 `TryGetValue` 失败。

**问题**
函数指针为 0 时，`(delegate* unmanaged[Cdecl]<...>)0` 是**合法的 C# 转换**，调用它会在地址 0 取指 → 立即 `EXCEPTION_ACCESS_VIOLATION`。整条链上没有任何 `if (ptr == 0)`、没有 `ensure`、没有日志——**唯一的诊断信息就是崩溃本身**。这是本插件最容易在"改了一行绑定名"后触发 P0 的地方。

注意对比：`TypeBridge.GetMethod`(`TypeBridge.cs:49-75`) 至少会校验 `Method.GetParameters().Length == InParamCount`；而这里连参数个数都不校验（见 F-ABI-004）。

**建议**
```csharp
// 生成侧 UnrealTypeSourceGenerator.cs:916-918 改为
$"\t\t{accessibility} static unsafe partial {returnType} {method.Name}({parameters})\n" +
$"\t\t{{\n" +
$"\t\t\tvar Ptr = global::Interop.MethodBridge.GetMethod(ref {slot}, \"{key}\");\n" +
$"\t\t\tif (Ptr == nint.Zero) throw new global::System.EntryPointNotFoundException(\"{key}\");\n" +
$"\t\t\treturn ((delegate* unmanaged[Cdecl]<{pointerType}>)Ptr)({arguments});\n" +
$"\t\t}}\n"
```
注意 `returnType == void` 时要生成 `{ ...; }` 而不是 `return ...;`（生成器里已有 `method.ReturnsVoid` 判断可复用，:893）。

**验证方式**
- grep：`grep -n "GetMethod(ref" Script/SourceGenerator/UnrealTypeSourceGenerator.cs`（确认无 null 检查）。
- 用例：手工注释掉 `FRegisterUnreal`（`FRegisterUnreal.cpp:191` 的 `[[maybe_unused]] FRegisterUnreal RegisterUnreal;`）后调 `Unreal.NewObject`，当前会直接崩溃；修复后应抛 `EntryPointNotFoundException` 并被 `MethodBridge` 的日志通道打印。

---

### [F-ABI-003] `AssemblyLoader.LoadFromStream` 无 try/catch，程序集加载异常穿出 `[UnmanagedCallersOnly]` 终止进程

- **类别**: Bug / 安全
- **严重度**: **P2**
- **复核结论**: 部分确认（偏差：同为"异常穿出即 fail-fast"的推论——当前后端下 `LoadFromStream` 经 `Bridge_Invoke` 调用，异常被 `Unhandled_Exception` 记录（`FLeanCLRLog::ErrorWriter`）后返回 0，C++ 侧 `IManagedHandleIsValid(Handle)` 为假 → **跳过该程序集**，不杀进程；"终止进程"只在 CoreCLR/Mono 成立）
- **可达性**: 潜伏（`FMonoDomain.cpp:415-421` 在 `#if WITH_MONO` 内、`FCoreCLRDomain.cpp:226-235` 在 `#if WITH_CORECLR` 内，二者当前均不编译；LeanCLR 侧的程序集加载走 `FLeanCLRDomain` 的另一条路径（`FLeanCLRDomain.cpp:812` 附近），`AssemblyLoader.LoadFromStream` 当前不被这三个后端之外的路径调用）
- **复核证据**: `Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:415-421`（`reinterpret_cast<const char16_t*>` + `IManagedHandleIsValid`）；`FLeanCLRDomain.cpp:306-320`（异常→记录→返回 false）；`Script/Interop/AssemblyLoader/AssemblyLoader.cs:13-28`（无 try/catch，同文件 `:30-61 Unload` 有 `try/finally`）
- **级别变动**: 无（维持 P0→P2）
- **函数**: `Interop.AssemblyLoader.LoadFromStream(byte*, int, char*)`
- **置信度**: 高

**现状（代码事实）**
```csharp
// Script/Interop/AssemblyLoader/AssemblyLoader.cs:13-28
        [UnmanagedCallersOnly]
        public static unsafe nint LoadFromStream(byte* InData, int InLength, char* InInPublishDirectory)
        {
            if (InData != null && InLength > 0)
            {
                Context ??= new UnrealAssemblyLoadContext(InInPublishDirectory is not null
                    ? new string(InInPublishDirectory)
                    : string.Empty);

                using var Stream = new UnmanagedMemoryStream(InData, InLength);

                return HandleData.Alloc(Context.LoadFromStream(Stream));
            }

            return 0;
        }
```
同一个文件里的 `Unload`(:30-61) 用了 `try/finally`，但 `LoadFromStream` 完全没有保护。

**调用上下文**
`FMonoDomain::LoadAssembly`(`FMonoDomain.cpp:415-421`)、`FCoreCLRDomain::LoadAssembly`(`FCoreCLRDomain.cpp:226-235`)、以及 Mono 的程序集预加载钩子 `AssemblyPreloadHook`(`FMonoDomain.cpp:333-357`)。数据来自 `FFileHelper::LoadFileToArray(Data, *AssemblyPath)`，路径是 `<Project>/Content/Script/*.dll`（`FUnrealCSharpFunctionLibrary.cpp:1024-1072`）。

**问题**
`AssemblyLoadContext.LoadFromStream` 会抛 `BadImageFormatException` / `FileLoadException` / `ArgumentException`。典型触发：`Content/Script/UE.dll` 用另一版本 .NET（或另一个后端）编译后被换入、被截断、或者同名装配件已被加载。异常穿出 `[UnmanagedCallersOnly]` → CoreCLR fail-fast（进程直接终止，UE 侧连 `OnUnhandledException` 都拿不到）。这与 Mono 侧 `FMonoDomain::LoadAssembly`(`FMonoDomain.cpp:359-391`) 的行为不对称——那边至少检查了 `MonoImageOpenStatus`(:368-372)。

**建议**
```csharp
[UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
public static unsafe nint LoadFromStream(byte* InData, int InLength, char* InInPublishDirectory)
{
    try
    {
        if (InData == null || InLength <= 0) return 0;
        Context ??= new UnrealAssemblyLoadContext(InInPublishDirectory is not null
            ? new string(InInPublishDirectory) : string.Empty);
        using var Stream = new UnmanagedMemoryStream(InData, InLength);
        return HandleData.Alloc(Context.LoadFromStream(Stream));
    }
    catch (Exception Ex)
    {
        Console.Error.WriteLine($"LoadFromStream failed: {Ex}");
        return 0;   // C++ 侧 IManagedHandleIsValid() 会跳过该装配件
    }
}
```
C++ 侧无需改动：`FMonoDomain.cpp:418` / `FCoreCLRDomain.cpp:231` 已经用 `IManagedHandleIsValid(Handle)` 判断，返回 0 即"跳过"。**但注意**返回 0 会让插件静默缺少程序集，应同样把错误写进 `FScriptLog`（`FScriptLog.h:8`）。

**验证方式**
- grep：`grep -n "catch" Script/Interop/AssemblyLoader/AssemblyLoader.cs`（当前 `Unload` 有 `finally`、`LoadFromStream` 无 `catch`）。
- 用例：把 `Content/Script/UE.dll` 截断为 100 字节后启动，观察是"日志报错 + 脚本不可用"还是进程消失。

---

### [F-ABI-004] 绑定函数指针只按"方法名字符串"匹配，**不校验参数个数/类型**——签名写错会被当成正确原型调用

- **类别**: Bug（架构性 ABI 缺陷）
- **严重度**: **P1**
- **复核结论**: 确认（"注册/解析两侧都只有名字、没有签名元数据"成立；`TypeBridge.GetMethod` 带个数校验、`MethodBridge.GetMethod` 不带，两者不对称成立）
- **可达性**: 活跃（注册路径 `FScriptDomainImpl.inl:601-633` 与解析路径 `MethodBridge.GetMethod` 在所有后端都执行；只是"签名不一致"的触发需要人为改动）
- **复核证据**: `Script/Interop/Bridge/MethodBridge.cs:140-148`（第一手读取，仅 `StringToMethod.TryGetValue(InName)`）；`MethodBridge.cs:12-30 RegisterBinding(byte**, nint*, int)`（只有名字/地址/个数三项）；`Script/SourceGenerator/UnrealTypeSourceGenerator.cs:900-918`（按 C# 侧声明拼 `pointerType`，无 C++ 侧签名参与）。**计数修正**：全仓库 `__X_YImplementation` **声明**实测为 **209** 个（`grep "private static unsafe partial [^;{]*__[A-Za-z0-9]+_[A-Za-z0-9()]+Implementation\("` → 209 命中），原文/§4.2 的 **211** 偏高 2（详见 §4.2）
- **级别变动**: P1→P2（与同类的 F-ABI-008 口径一致：当前双侧 209 个声明逐参数比对**无实际不一致**，属"无护栏的隐患"而非既存功能错误）
- **文件**: `Script/Interop/Bridge/MethodBridge.cs:140-148`、`Script/SourceGenerator/UnrealTypeSourceGenerator.cs:916-918`
- **函数**: `Interop.MethodBridge.GetMethod(ref nint, string)`
- **置信度**: 高

**现状（代码事实）**
```csharp
// Script/Interop/Bridge/MethodBridge.cs:140-148  —— 只比名字，没有参数个数/类型信息
        public static nint GetMethod(ref nint InSlot, string InName)
        {
            if (InSlot == nint.Zero)
            {
                InSlot = StringToMethod.TryGetValue(InName, out var Method) ? Method : nint.Zero;
            }

            return InSlot;
        }
```
对照 `TypeBridge.GetMethod`(`TypeBridge.cs:49-75`) 是带个数校验的：
```csharp
// Script/Interop/Bridge/TypeBridge.cs:63-69
                    foreach (var Method in Type.GetMethods(BindingFlag))
                    {
                        if (Method.Name == Name && Method.GetParameters().Length == InParamCount)
                        {
                            return HandleData.Alloc(Method);
                        }
                    }
```
C++ 侧注册时也不带签名信息：`IScriptTypes.h:83` 的 `method_bridge_register_binding_fn` 只有 `(const uint8* const*, const PTRINT*, int32)` —— **名字数组 + 地址数组 + 个数**，没有任何签名元数据（`FScriptDomainImpl.inl:605-631`）。

**调用上下文**
`FScriptDomainImpl.inl:601-633 RegisterBinding()` → `MethodBridgeRegisterBindingFn` → `MethodBridge.RegisterBinding`(`MethodBridge.cs:12-30`)。

**问题**
C++ 侧 `static void YYYImplementation(const IManagedHandle, const int32)` 与 C# 侧 `__X_YYYImplementation(nint, int, int)`（多一个参数）这种错误**不会被任何环节发现**：
1. 名字完全一致；
2. 没有签名元数据可比；
3. 生成的代码用 `delegate* unmanaged[Cdecl]<...>` **按 C# 声明**构造原型去调 C++；
4. C++ 侧多余/缺少的参数读的是垃圾寄存器/栈槽。

我逐一核对了 211 个 `__X_YImplementation`（见第 4 节表 B 与 §4.3），**当前仓库内未发现真实不一致**（作者手工保持一致，且 `bool↔byte` 的约定执行得很干净），但这个"没有护栏"的结构意味着**下一次改动就可能静默破坏 ABI**，且症状是难以定位的内存错乱而不是明确的报错。

**建议**
最短路径：给注册加上参数个数校验。
```cpp
// FScriptDomainImpl.inl:601-633 附近，把 ParamCount 一起传下去
// 1) 扩展 method_bridge_register_binding_fn 为 (const uint8* const*, const PTRINT*, const int32*, int32)
// 2) C# RegisterBinding 存 (方法名 -> (指针, 参数个数))，GetMethod 增加 int InParamCount 参数校验；
//    不匹配时写日志并返回 0（配合 F-ABI-002 的 null 检查即变成可诊断异常）
```
生成侧把 `method.Parameters.Length` 作为 `InParamCount` 传入即可（`UnrealTypeSourceGenerator.cs:895-901` 已有 `method.Parameters`）。

**验证方式**
- grep：`grep -n "GetParameters\(\).Length" Script/` —— 只有 `TypeBridge.GetMethod`(:65) 与 `MethodBridge.Invoke`(:43) 用到了个数，注册/解析路径没有。
- 反证用例：把 `Script/UE/Library/TArrayImplementation.cs:44` 的 `int InIndex` 改成 `long InIndex`（x64 下 ABI 仍然兼容，不崩溃），再把 `TArray_NumImplementation` 多加一个 `int` 参数 → 应立即报错而不是静默返回垃圾。

---

### P1

### [F-ABI-005] `TypeBridge.MakeGenericType2` 第 3 个参数被忽略：`TMap<K,V>` 的 V 恒等于 K

- **类别**: Bug
- **严重度**: **P1**
- **复核结论**: 确认（`TypeBridge.cs:192` 与 `:194` 两次都用 `InKeyType`，逐字核对无误；`InValueType` 仅出现在 :188 形参列表）
- **可达性**: 活跃（`TypeBridge.MakeGenericType2` 是 `IScriptTypes.h:119` 注册的桥接方法，三个后端都要用它构造 `TMap<K,V>` 的泛型实例；LeanCLR 下经 `Bridge_Invoke` 走解释器同样会命中）
- **复核证据**: `Script/Interop/Bridge/TypeBridge.cs:187-202`（第一手读取：`if (HandleData.GetObject(InKeyType) is Type Value)` 位于 :194）
- **级别变动**: 无（P1 维持 —— 这是本报告中**唯一一条会静默产生错误类型**的功能性缺陷，与"缺护栏"类的 F-ABI-004/008 不同级）
- **函数**: `Interop.TypeBridge.MakeGenericType2(nint, nint, nint)`
- **置信度**: 高

**现状（代码事实）**
```csharp
// Script/Interop/Bridge/TypeBridge.cs:187-202
    [UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
    public static nint MakeGenericType2(nint InGeneric, nint InKeyType, nint InValueType)
    {
        if (HandleData.GetObject(InGeneric) is Type Generic)
        {
            if (HandleData.GetObject(InKeyType) is Type Key)
            {
                if (HandleData.GetObject(InKeyType) is Type Value)     // ← 应为 InValueType
                {
                    return HandleData.Alloc(Generic.MakeGenericType(Key, Value));
                }
            }
        }

        return 0;
    }
```

**调用上下文**
`FTypeBridge.cpp:561-575 GetClass(const FMapProperty*)` 取 `InProperty->KeyProp`/`InProperty->ValueProp` 两个类后调 `MakeGenericTypeInstance(FoundGenericClass, FoundKeyClass, FoundValueClass)`(:571) → `FScriptDomainImpl.inl:453-463` → `TypeBridgeMakeGenericType2Fn`。typedef 在 `IScriptTypes.h:31`，注册在 `IScriptTypes.h:119`。CoreCLR 侧经 `RegisterInterop`(`FCoreCLRDomain.cpp:281-294`) 按方法名+参数个数(3)解析。

**问题**
`Value` 永远来自 `InKeyType`，于是 `Generic.MakeGenericType(Key, Key)`。对 `TMap<FName, AActor*>`，C++ 以为拿到的是 `TMap<FName, AActor*>` 的 `FClassReflection`，实际是 `TMap<FName, FName>`（如果该实例化合法；若泛型约束不满足则 `MakeGenericType` 抛异常——见下）：
- **若构造成功**：后续 `FTypeBridge::GetClass(const FMapProperty*)` 的返回值被 `FReflectionRegistry::Get().GetClass(IManagedHandle)` 缓存/复用，键值类型错配 → C# 侧写入/读取 map 元素时按错误的类型尺寸与句柄解释内存；
- **若 `MakeGenericType` 抛 `ArgumentException`**（约束不满足）→ 穿出 `[UnmanagedCallersOnly]` → **进程终止**（与 F-ABI-001 同一类问题：该方法已带 `CallConvs=Cdecl` 但没有 try/catch）。

**建议**
```csharp
if (HandleData.GetObject(InValueType) is Type Value)
```
并同时补 try/catch（与 F-ABI-001 的建议一致）。C++ 侧 `FReflectionRegistry::Get().GetClass(handle)` 目前不区分"返回 0"与"合法类型"，建议对 0 加 `ensure`。

**验证方式**
- grep：`grep -n "InValueType" Script/Interop/Bridge/TypeBridge.cs` —— 参数名在签名与 XML 注释之外**零次出现**（只出现在 :188 的形参列表），这是最直接的证据。
- 用例：在 C# 里 `var M = new TMap<FName, UObject>(); M.Add(...)`，检查 C++ 侧生成的属性类型是否为 `TMap<FName, FName>`。

---

### [F-ABI-006] `AssemblyLoader.LoadFromStream` 的发布目录参数把 `TCHAR*` 硬 `reinterpret_cast` 成 `char16_t*`——默认配置下可用，但 `bTCHARIsUTF8` 平台上乱码

> ☞ **本文档修订记录**：本条初稿把严重度定为 P1 并断言"Linux/macOS 上必现乱码"。取得引擎侧证据后**该断言被证伪**（见下），严重度下调为 **P2**，并改述为"仅在非默认 TCHAR 配置下失效"。为保持行文可追溯，条目保留在本区块，但**按 P2 计分**。

> ☞ **后端门控**：本条位于 `FMonoDomain.cpp`，该文件整体在 `#if WITH_MONO` 内（:2）。当前工程 ini 为 `WITH_MONO=0`（§9 第 10 条），因此本条**当前未参与编译**；切回 Mono 后端即生效。

- **类别**: 平台兼容 / Bug
- **严重度**: **P2**
- **复核结论**: 确认（`:416` 的 `reinterpret_cast<const char16_t*>(*FPaths::GetPath(AssemblyPath))` 与 `#else`/`#if PLATFORM_WINDOWS` 的门控逐字核对无误；"默认配置下 Linux/macOS 的 `TCHAR` 也是 2 字节 UTF-16 因而是正确的"这一改判，由引擎侧 `UnixPlatform.h:17/:50`、`MacPlatform.h:11/:55`、`Platform.h:269-281` 支持）
- **可达性**: 潜伏（`FMonoDomain.cpp` 整体在 `#if WITH_MONO` 内，当前 `WITH_MONO=0` → 不参与编译；即使切回 Mono，也只在 `bTCHARIsUTF8=true` 或 4 字节 `TCHAR` 目标上才失效）
- **复核证据**: `Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:411-421`（第一手读取：`:413 FFileHelper::LoadFileToArray`、`:415-417 AssemblyLoaderLoadFromStreamFn(Data.GetData(), Data.Num(), reinterpret_cast<const char16_t*>(…))`、`:418 IManagedHandleIsValid(Handle)`），与 `FCoreCLRDomain.cpp:210/229-230` 的 `StringCast<UTF16CHAR>` 写法对照
- **级别变动**: 无（P2 维持；已由"P1 + Linux 必现乱码"改为"P2 + 仅非默认 TCHAR 配置"）
- **函数**: `FMonoDomain::LoadAssembly(const TArray<FString>&)`
- **置信度**: 高（代码事实与内部不一致性）；高（引擎侧 TCHAR 定义）

**现状（代码事实）**
```cpp
// Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:415-421
			if (const auto Handle = AssemblyLoaderLoadFromStreamFn(Data.GetData(), Data.Num(),
			                                                       reinterpret_cast<const char16_t*>(*FPaths::GetPath(
				                                                       AssemblyPath)));
				IManagedHandleIsValid(Handle))
			{
				Assemblies.Add(Handle);
			}
```
C# 侧接收方是 **UTF-16**：`AssemblyLoader.cs:14` `LoadFromStream(byte* InData, int InLength, char* InInPublishDirectory)`，`char` 在 C# 恒为 2 字节。

**调用上下文**
typedef `IScriptTypes.h:5 assembly_loader_Load_from_stream_fn`；注册在 `IScriptTypes.h:159`；路径来自 `FUnrealCSharpFunctionLibrary::GetFullAssemblyPublishPath()`(:1064-1072)。

**问题（含初稿的更正）**
同一份代码在 200 行之前使用了**显式转换**：
```cpp
// Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:146-152（以及 FCoreCLRDomain.cpp:174-180）
#define GET_UTILS_FUNCTION_POINTER(InFn, InName) \
	InFn = reinterpret_cast<decltype(InFn)>(TypeBridgeGetFunctionPointerFn( \
			reinterpret_cast<const char16_t*>(StringCast<UTF16CHAR>( \
				*FUnrealCSharpFunctionLibrary::GetUEName()).Get()), \
```
`StringCast<UTF16CHAR>` 会**真正**把 `FString` 转成 UTF-16 缓冲，而 `:416` 只是把 `const TCHAR*` 原样强转。

**引擎侧取证（`<Engine 5.6 安装目录>`，第一手读取）**：
- `Engine/Source/Runtime/Core/Public/Unix/UnixPlatform.h:17` `#define PLATFORM_UNIX_USE_CHAR16 1` → `:50` `#define PLATFORM_TCHAR_IS_CHAR16 1`
- `Engine/Source/Runtime/Core/Public/Mac/MacPlatform.h:11` `#define PLATFORM_MAC_USE_CHAR16 1` → `:55` `#define PLATFORM_TCHAR_IS_CHAR16 1`
- `Engine/Source/Runtime/Core/Public/HAL/Platform.h:269-281`：`PLATFORM_TCHAR_IS_4_BYTES` 默认 0；`PLATFORM_TCHAR_IS_UTF8CHAR` = `USE_UTF8_TCHARS`

**结论更正**：在 UE 5.6 的**默认配置**下，Windows(`wchar_t`)、Linux(`char16_t`)、macOS(`char16_t`) 的 `TCHAR` **都是 2 字节 UTF-16**，因此 `:416` 的强转**是正确的**——初稿"Linux 上必现乱码"的说法**错误，现予撤回**。

仍然存在缺陷的场景（因此保留为 P2）：
1. 项目设置 `TargetRules.bTCHARIsUTF8 = true` → UBT 定义 `USE_UTF8_TCHARS=1` → `Platform.h:281` 使 `PLATFORM_TCHAR_IS_UTF8CHAR=1` → Linux/macOS 上 `TCHAR` 变成 **UTF-8 `char`**（1 字节），此时 `:416` 让托管侧按 UTF-16 逐 2 字节读取 UTF-8 缓冲 → **程序集搜索路径乱码** → `UnrealAssemblyLoadContext.Load`(`UnrealAssemblyLoadContext.cs:21`) 的 `Path.Combine` 指向不存在的路径 → 依赖程序集静默加载失败；
2. 任何 `PLATFORM_TCHAR_IS_CHAR16=0` 且 `PLATFORM_TCHAR_IS_4_BYTES=1` 的目标（4 字节 `TCHAR`）同样失效；
3. **与 `FCoreCLRDomain.cpp:210/229-230` 的写法不一致**：那里先 `const auto PublishDirectory = StringCast<UTF16CHAR>(...)`（具名局部变量）再 `reinterpret_cast<const char16_t*>(PublishDirectory.Get())`，是**同宽度的无损转换**，属正确模板。

**建议**
```cpp
// 与 CoreCLR 侧对齐（注意：TStringConversion 必须存成具名局部变量，跨语句使用会悬垂）
const auto PublishDirectory = StringCast<UTF16CHAR>(*FPaths::GetPath(AssemblyPath));
if (const auto Handle = AssemblyLoaderLoadFromStreamFn(
        Data.GetData(), Data.Num(), reinterpret_cast<const char16_t*>(PublishDirectory.Get()));
    IManagedHandleIsValid(Handle))
```
即：把"依赖 `TCHAR` 恰好是 2 字节"的隐式假设换成显式转换，这样三种 `TCHAR` 配置下都正确。

**验证方式**
- `grep -rn "reinterpret_cast<const char16_t\*>" Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp`，与 `FCoreCLRDomain.cpp` 的同名模式对照。
- 用例：设置 `bTCHARIsUTF8 = true` 后在 Linux 上运行，把 `Content/Script/Game.dll` 拆成 `Game.dll + 第三方依赖.dll`，观察依赖是否被加载。

---

### [F-ABI-007] `FieldBridge.SetStaticValue/GetStaticValue` 缺 try/catch，且 `GetStaticValue` 用 `Convert.ToInt64` 会对合法枚举值抛异常

- **类别**: Bug
- **严重度**: **P2**
- **复核结论**: 确认（`SetStaticValue`/`GetStaticValue` 均无 try/catch；`FieldToLong`/`LongToField` 用 `Convert.ToInt64`/`Convert.ChangeType` 逐字核对无误）
- **可达性**: 活跃（`GetStaticValue` 由 `FDynamicEnumGenerator.cpp:281-295` 调用；LeanCLR 下异常被 `Unhandled_Exception` 记录后返回 0，**不杀进程**，后果是"枚举值静默变 0"）
- **复核证据**: `Script/Interop/Bridge/FieldBridge.cs:10-32`（无 catch）、`:49-64`（`Convert.ToInt64` :63、`Convert.ChangeType` :55）；`Source/UnrealCSharp/Private/Reflection/Function/FCSharpFunctionDescriptor.cpp:134`（对照：本插件里唯一一处 `DestroyValue_InContainer`）
- **级别变动**: 无（维持 P1→P2；把"终止进程"改为"静默错误值"进一步支持 P2 而非 P1）
- **函数**: `Interop.FieldBridge.SetStaticValue(nint, byte*, long)` / `Interop.FieldBridge.GetStaticValue(nint, byte*)`
- **置信度**: 中（"uint64 型 UENUM 且值 > 2^63"这一触发器在实际工程里需要构造，未验证其发生频率）

**现状（代码事实）**
```csharp
// Script/Interop/Bridge/FieldBridge.cs:10-32
    [UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
    public static unsafe void SetStaticValue(nint InHandle, byte* InName, long InValue)
    {
        var Field = GetField(InHandle, InName);

        if (Field != null)
        {
            Field.SetValue(null, LongToField(InValue, Field.FieldType));
        }
    }

    [UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
    public static unsafe long GetStaticValue(nint InHandle, byte* InName)
    {
        var Field = GetField(InHandle, InName);

        if (Field != null)
        {
            return FieldToLong(Field.GetValue(null));
        }

        return 0;
    }
```
```csharp
// Script/Interop/Bridge/FieldBridge.cs:49-64
    private static object? LongToField(long InValue, Type InField) => InField switch
    {
        { IsValueType: false } => InValue != 0 ? InValue : null,
        { IsEnum: true } => Enum.ToObject(InField, InValue),
        _ when InField == typeof(nint) => (nint)InValue,
        _ when InField == typeof(nuint) => (nuint)InValue,
        _ => Convert.ChangeType(InValue, InField)
    };

    private static long FieldToLong(object? InValue) => InValue switch
    {
        null => 0,
        nint v => v,
        nuint v => (long)v,
        _ => Convert.ToInt64(InValue)
    };
```

**调用上下文**
- `GetStaticValue` 的唯一调用方：`FDynamicEnumGenerator::GeneratorEnumerator`(`Source/UnrealCSharpCore/Private/Dynamic/FDynamicEnumGenerator.cpp:281-295`)，遍历 `InClassReflection->GetFields()` 并对每个字段调 `GetFieldStaticValue` 再 `reinterpret_cast<int64>`。
- `SetStaticValue` 的调用方：`FCSharpBind.cpp:172`、`FCSharpBind.cpp:224`、`FCSharpBind.cpp:362`（都传 `&FieldHash`，`FieldHash = GetTypeHash(...)` 是 `uint32`）+ `FClassDescriptor.cpp:46-52`（传 `uint32 Value{}`）。

**问题**
1. **异常穿出 `[UnmanagedCallersOnly]`（无 try/catch）**：
   - `Convert.ToInt64(ulong)` 在值 > `long.MaxValue` 时抛 `OverflowException`。UE 的非用户自定义 UENUM 的底层类型由生成器映射为 `long`（`Source/ScriptCodeGenerator/Private/FEnumGenerator.cpp:270`：`InEnum->IsA(UUserDefinedEnum::StaticClass()) ? TEXT("byte") : TEXT("long")`），因此 C# 侧完全可能出现 `enum : ulong` 且枚举值 `0x8000_0000_0000_0000` 以上——这是合法的 UENUM 值域，却会让进程直接终止；
   - `Convert.ToInt64(string)` 抛 `FormatException`——如果 `GetFields()` 里出现了非枚举的静态字段（枚举类可以带 `const` 之外的静态成员），同样终止进程；
   - `Field.SetValue` 对 `readonly`/`const`/字面量字段抛 `FieldAccessException`。
2. **`GetStaticValue` 用 `0` 同时表示"字段不存在"和"字段值就是 0"**：`GeneratorEnumerator` 无法区分，会往 `UEnum` 里写入错误的枚举值（`InNames.Add({FName(Name), FieldValue})`，:292），且无任何日志。
3. **32/64 位契约错配**：C# 声明第 3 参数是 `long`(8B)，而 C++ typedef `IScriptTypes.h:79` 是 `field_bridge_set_static_value_fn(IManagedHandle, const uint8*, IManagedHandle)`；C++ 的实际实参由 `FScriptDomainImpl.inl:398-399` 构造：
   ```cpp
   SCRIPT_DOMAIN_INVOKE(void, FieldBridgeSetStaticValueFn, InManagedClass, SCRIPT_DOMAIN_STRING_CAST(InName),
                        IManagedHandle{*static_cast<uint32*>(InValue)});
   ```
   即只读了 **4 字节** 并零扩展到 64 位。当前 4 个调用方传的都是 `uint32`（见上），**所以现在没有截断 bug**；但 `FFieldReflection::SetValue(const FClassReflection*, void*)` 的接口语义是 `void*`（`Source/UnrealCSharpCore/Public/Reflection/FFieldReflection.h:16`），任何后续调用方传 `int64`/`double*`（例如"设置一个 `long` 静态字段"）都会被**静默截断成低 32 位**。这是典型的"当前正确、下一秒就错"的接口。

**建议**
```csharp
[UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
public static unsafe long GetStaticValue(nint InHandle, byte* InName)
{
    try { /* 原逻辑 */ }
    catch (Exception Ex) { Console.Error.WriteLine(Ex); return 0; }
}
```
并为 `FieldToLong` 增加 `ulong` 分支（`unchecked((long)v)`，与 C++ 侧 `int64` 位语义一致）。C++ 侧把 `IManagedHandle{*static_cast<uint32*>(InValue)}` 改成按字段实际类型读取，或把接口改成显式的 `int64` 值语义（`SetFieldStaticValue(handle, name, int64 InValue)`）并更新 `FFieldReflection::SetValue`。

**验证方式**
- grep：`grep -n "catch" Script/Interop/Bridge/FieldBridge.cs`（0 命中）。
- 用例：声明 `public enum EBig : ulong { Max = 0xFFFF_FFFF_FFFF_FFFF }` 并挂 `[UEnum]`，观察 `GeneratorEnumerator` 是否终止进程。
- 断言：在 `FDynamicEnumGenerator.cpp:289` 后加 `ensureAlways(FieldValue != 0 || ...)`。

---

### [F-ABI-008] `FScriptDomainImpl.inl` 中的 `Utils` 描述符协议是 15/8/3/14 个槽位的"魔术数字对"，两侧靠人工同步

- **类别**: 可优化/可读性（隐患）
- **严重度**: **P2**
- **复核结论**: 部分确认（机制成立、当前数值确实匹配；但偏差：原文称 C++ 侧"15 个可写槽"——`FClassReflection.cpp:418-429` 的 `IManagedHandle Params[16]` 实际给出 15 个可写槽（`Params[1..15]`），与 C# `GetClassDescriptor` 写入到 `OutBuffer[14]` 一致，**无越界**；`reinterpret_cast<PTRINT*>` 丢长度信息属实）
- **可达性**: 活跃（类描述符加载是每次绑定的必经路径）
- **复核证据**: `Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp:418-429`（`Params[16]` + `for (Index = 1; Index < 16)` 清零）；同一文件 `:451-457`（`Params.Init(InvalidManagedHandle, InSlotCount + 1)`）；重跑 grep 确认 `EnsurePropertiesImplementation`/`EnsureFieldsImplementation`/`EnsureMethodsImplementation` 传 8/3/14（报告 §3 引用的 :465-481 行区间与之自洽）
- **级别变动**: 无（维持 P1→P2，与 F-ABI-004 同口径）
- **文件**: `Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp:418-427`、`:451-457`、`:465-481`；`Script/UE/CoreUObject/Utils.cs:633-678`、`:681-710`、`:713-727`、`:730-773`
- **函数**: `FClassReflection::EnsureDescriptorImplementation` / `EnsureMemberImplementation` / `Ensure{Properties,Fields,Methods}Implementation`
- **置信度**: 高（当前数值**恰好匹配**，见下）

**现状（代码事实）**
```cpp
// Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp:418-429
		IManagedHandle Params[16];

		Params[0] = ManagedClass;

		for (auto Index = 1; Index < 16; ++Index)
		{
			Params[Index] = InvalidManagedHandle;
		}

		ScriptDomain->GetClassDescriptor(ManagedClass, reinterpret_cast<PTRINT*>(&Params[1]));
```
```cpp
// Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp:451-481
		TArray<IManagedHandle, TInlineAllocator<32>> Params;

		Params.Init(InvalidManagedHandle, InSlotCount + 1);
		...
		(this->*ParseMemberFunction)(FManagedReader{ScriptDomain}, &Params[0]);
	}

	void FClassReflection::EnsurePropertiesImplementation()
	{
		EnsureMemberImplementation(bPropertiesLoaded, 8, ...);
	}
	void FClassReflection::EnsureFieldsImplementation()
	{
		EnsureMemberImplementation(bFieldsLoaded, 3, ...);
	}
	void FClassReflection::EnsureMethodsImplementation()
	{
		EnsureMemberImplementation(bMethodsLoaded, 14, ...);
	}
```
C# 侧写入的槽位数（逐行核对 read 输出）：

| 方法 | C# 文件:行 | 写入的最大索引 | 槽位数 |
|---|---|---|---|
| `GetClassDescriptor` | `Utils.cs:646-676` | `OutBuffer[14]` | **15** |
| `GetClassProperties` | `Utils.cs:692-708` | `OutBuffer[7]` | **8** |
| `GetClassFields` | `Utils.cs:721-725` | `OutBuffer[2]` | **3** |
| `GetClassMethods` | `Utils.cs:743-771` | `OutBuffer[13]` | **14** |

**调用上下文**
`FScriptDomainImpl.inl:485-522` 四个转发函数；Mono 侧经 `TypeBridgeGetFunctionPointerFn` 取 `Utils` 类的 `[UnmanagedCallersOnly]` 方法（`FMonoDomain.cpp:146-163`），CoreCLR 侧同（`FCoreCLRDomain.cpp:174-191`）。

**问题**
当前**恰好一一对应、没有越界**（15 vs `Params[16]` 的 15 个可写槽；8/3/14 vs `InSlotCount`）。但：
- `8/3/14` 这个 `InSlotCount` 与 C# 里的 `OutBuffer[0..7]/[0..2]/[0..13]` 是**两处独立硬编码**，中间没有任何共享常量、`static_assert`、注释交叉引用或运行时长度校验；
- `reinterpret_cast<PTRINT*>(&Params[1])` 把长度信息彻底丢弃了——C# 侧**不知道**自己有多少槽位可用，多写一个 `OutBuffer[14]`（`GetClassMethods` 只要再加一个输出数组，非常容易发生）就是 `TArray` 的**越界堆写**，而且会写坏 `TInlineAllocator<32>` 的内联缓冲区之外的内存；
- 我逐一核对的依据是 C# 源码，属于"人工同步"。任何一侧的单方面增删都无法被发现。

**建议**
1. 把槽位数变成 C++ 传给 C# 的参数（`GetClassMethods(handle, ptr, int InSlotCount)`），C# 侧 `if (InSlotCount < 15) return;` 后 `throw`/打日志；这需要在 `IScriptTypes.h:97-103` 与 `Utils.cs` 同步改。
2. 最低成本方案：定义一份共享的枚举并在两侧各加一行 `static_assert`/`Debug.Assert`：
   ```cpp
   // IScriptTypes.h
   static constexpr int32 ClassDescriptorSlots = 15;
   static constexpr int32 ClassPropertySlots   = 8;
   static constexpr int32 ClassFieldSlots      = 3;
   static constexpr int32 ClassMethodSlots     = 14;
   static_assert(ClassDescriptorSlots + 1 <= 16, "Params[] too small");
   ```
   ```csharp
   if (OutBuffer == null) return;
   // 与 IScriptTypes.h 的 ClassMethodSlots 保持一致
   ```
3. 在 `FClassReflection.cpp` 的 `EnsureMemberImplementation` 里把 `InSlotCount` 与 `Params.Num() - 1` 做 `check()`。

**验证方式**
- grep：`grep -n "OutBuffer\[" Script/UE/CoreUObject/Utils.cs`（列出所有写入槽位，与 `InSlotCount` 比对）；
  `grep -n "EnsureMemberImplementation(b" Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp`（列出 8/3/14）。
- 断言：给 `EnsureMemberImplementation` 的首尾各加一个 `FMemory::Memset(Params.GetData(), 0xCD, ...)` 哨兵，运行后检查哨兵是否被越过。

---

### [F-ABI-009] `FCSharpEnvironment` 用全局 `signal()` 覆盖 SIGSEGV/SIGFPE/SIGILL，处理函数里调用非 async-signal-safe 的 `UE_LOG` 与托管运行时；且 Windows/Linux 分支比 Mac 分支少做"保存并恢复原处理器"

- **类别**: 平台兼容 / Bug
- **严重度**: **P1**
- **复核结论**: 确认（三条子结论全部逐字核对成立：①Mac 分支保存原 `sigaction`（`SignalActions.Add(SignalType)` 作为 out 参数），Windows/Linux 分支只有 `signal(SignalType, SignalHandler)` 且**丢弃原处理器、不恢复**；②`SignalHandler` 内调用 `UE_LOG`/`GLog->Flush()`/`FDomain::GetTraceback()`，均非 async-signal-safe；③Mac 分支 `SignalActions[Signal]` 是信号上下文里的 `TMap::operator[]`）
- **可达性**: 活跃（`FCSharpEnvironment::Initialize()` 由模块激活委托驱动，三平台都执行；`#else` 分支覆盖 Windows 与 Linux 两个当前目标平台）
- **复核证据**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:18-31`（`SignalHandler` 全文 + Mac 的 `TMap<int32, struct sigaction>`）；`:101-142`（`SignalTypes` 含 SIGINT/SIGILL/SIGFPE/SIGSEGV/SIGTERM/SIGABRT(+/SIGBREAK)；`:122-138` Mac `sigaction` 保存原处理器，`:139-141 #else signal(...)`）
- **级别变动**: 无（P1 维持：在开发机上"崩溃变卡死 + UE 崩溃报告失效"是可直接观测的功能损失）
- **函数**: `SignalHandler(int32)`、`FCSharpEnvironment::Initialize()`
- **置信度**: 高（平台分支语义不等价是代码事实）；中（死锁是否必然发生取决于故障点是否持有 UE 分配器/日志锁）

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:18-31
#if PLATFORM_MAC
TMap<int32, struct sigaction> SignalActions;
#endif

void SignalHandler(int32 Signal)
{
	UE_LOG(LogUnrealCSharp, Error, TEXT("%s"), *FDomain::GetTraceback());

	GLog->Flush();

#if PLATFORM_MAC
	sigaction(Signal, &SignalActions[Signal], nullptr);
#endif
}
```
```cpp
// Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:101-142
	static TSet<int32> SignalTypes = {
		SIGINT, SIGILL, SIGFPE, SIGSEGV, SIGTERM,
#if PLATFORM_WINDOWS
		SIGBREAK,
#endif
		SIGABRT,
	};

	for (const auto SignalType : SignalTypes)
	{
#if PLATFORM_MAC
		struct sigaction SigAction;
		FMemory::Memzero(&SigAction, sizeof(struct sigaction));
		SigAction.sa_handler = SignalHandler;
		sigemptyset(&SigAction.sa_mask);
		if (!SignalActions.Contains(SignalType))
		{
			sigaction(SignalType, &SigAction, &SignalActions.Add(SignalType));
		}
		else
		{
			sigaction(SignalType, &SigAction, nullptr);
		}
#else
		signal(SignalType, SignalHandler);
#endif
	}
```

**调用上下文**
`FCSharpEnvironment::Initialize()`(:57) 由 `UUnrealCSharpModule` 激活时调用（`FUnrealCSharpModuleDelegates::OnUnrealCSharpModuleActive`，:37-38）。`FDomain::GetTraceback()`(`Source/UnrealCSharp/Private/Domain/FDomain.cpp:112-129`) 内部：`FReflectionRegistry::Get().GetUtilsClass()`（TMap 查找 + FString 操作）→ `TracebackMethod->Runtime_Invoke()`（**进入 Mono/CoreCLR 托管运行时**）→ `StringToFString`（再走一次桥接）→ `GCHandle_Free`。

**问题**
1. **平台分支语义不等价**：Mac 分支保存了原 `sigaction` 并在处理完后用 `sigaction(Signal, &SignalActions[Signal], nullptr)` **恢复原处理器**（这样第二次触发会走回 UE/系统原处理器，能产生崩溃转储）；Windows/Linux 分支用的是 `signal(SignalType, SignalHandler)`，**丢失了原处理器且不恢复**。后果：插件的处理器**永久替换**了 UE 的崩溃处理器（`FPlatformMallocCrash`/`CrashReportClient`/`FGenericCrashHandler`），崩溃时的 callstack/上传功能失效；并且处理函数自身如果再次触发同一信号，会**递归进入** `SignalHandler` → 栈溢出。
2. **处理函数不是 async-signal-safe**：`UE_LOG`（可能分配内存、取 `GLog` 锁、写文件）、`GLog->Flush()`、`FDomain::GetTraceback()` 里的 TMap/FString/托管运行时调用，全部在信号上下文里执行。若 SIGSEGV 发生在 `FMemory` 分配器或 `GLog` 的临界区内（非常常见），会**死锁**——进程挂住而不是崩溃，产物是"卡死"这种最难排查的现象。
3. **Mac 分支在信号处理里改 TMap**：`SignalActions[Signal]`(:29) 是 `TMap::operator[]`，键不存在时会**插入并可能重新分配整个 map**——在信号处理器里做非重入的容器修改，是 UB 级别的隐患（Mac 专属）。

**建议**
- 统一为 `sigaction` + `SA_RESETHAND`/手动恢复（三平台一致），并把原处理器保存起来：
  ```cpp
  struct sigaction Action{}, Old{};
  Action.sa_sigaction = &SignalHandler;
  Action.sa_flags = SA_RESETHAND | SA_SIGINFO;   // 只处理一次，恢复默认
  sigemptyset(&Action.sa_mask);
  sigaction(SignalType, &Action, &Old);
  ```
- 处理函数里**只做**：`write(2, ...)`（async-signal-safe）+ `_exit(3)`；把 `UE_LOG`/托管调用全部删掉。
- 如果必须打印托管栈，改为把信号转成 `FPlatformMisc::RequestExitWithStatus` 或只在 GameThread 上做的延迟任务（信号处理器里只 `volatile` 置标志）。
- 至少给该 `signal()` 覆盖行为加一个可配置开关（`UUnrealCSharpSetting`），默认关闭。

**验证方式**
- grep：`grep -n "signal(\|sigaction(" Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp`（对比 `#if PLATFORM_MAC` 的 3 处与 `#else` 的 1 处）。
- 用例：在 Windows 上故意 `nullptr->GetName()` 触发 SIGSEGV，观察 (a) 是否卡死、(b) UE 崩溃报告是否还生成。

---

### [F-ABI-010] Mono 初始化用 `TCHAR_TO_ANSI` 传 `NATIVE_DLL_SEARCH_DIRECTORIES`——中文/非 ANSI 项目路径变乱码，mono 找不到本机库

- **类别**: 平台兼容（编码）/ Bug
- **严重度**: **P2**（**后端门控**：`FMonoDomain.cpp` 整体在 `#if WITH_MONO` 内（:2），当前工程 ini 为 `WITH_MONO=0`，本条当前未参与编译；切回 Mono 后端即生效——见 §9 第 10 条）
- **复核结论**: 确认（`TCHAR_TO_ANSI` 用法与位置逐字核对无误；且该行本身还被 `#if DOTNET9` + `#if PLATFORM_WINDOWS` 双重门控）
- **可达性**: 潜伏（`WITH_MONO=0` → `FMonoDomain.cpp` 不参与编译；且即使切回 Mono，`DOTNET9` 与 Windows 条件也须同时成立）
- **复核证据**: `Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:76-90`（第一手读取：`:79 #if PLATFORM_WINDOWS` → `:86 TCHAR_TO_ANSI(*LibDirectory)` → `:89 monovm_initialize(...)`，`:92-94` 为 Linux 的 `setenv`）
- **级别变动**: 无（维持 P1→P2；另有更强的门控：`#if DOTNET9`）
- **文件**: `Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:79-90`
- **函数**: `FMonoDomain::Initialize()`
- **置信度**: 高（`TCHAR_TO_ANSI` 使用系统 ANSI 代码页是 UE 的既定语义；`monovm_initialize` 期望 UTF-8 由其 `char*` 契约决定）

**现状（代码事实）**
```cpp
// Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:78-90
#if DOTNET9
#if PLATFORM_WINDOWS
		const auto LibDirectory = FMonoFunctionLibrary::GetLibDirectory();

		const char* PropertyKeys[] = {
			"NATIVE_DLL_SEARCH_DIRECTORIES"
		};
		const char* PropertyValues[] = {
			TCHAR_TO_ANSI(*LibDirectory),
		};

		monovm_initialize(std::size(PropertyKeys), PropertyKeys, PropertyValues);
#endif
```
同一函数里 `:113-114` 的 `--soft-breakpoints`/debugger 配置也用 `TCHAR_TO_ANSI`。

**调用上下文**
`FMonoDomain::Initialize` 由 `FScriptDomainFactory`(`Source/UnrealCSharpCore/Private/Domain/Script/FScriptDomainFactory.cpp:27-35`) 在模块启动时创建；`LibDirectory` 来自 `FMonoFunctionLibrary::GetLibDirectory()`(`FMonoFunctionLibrary.cpp:36-61`)，基于 `GetMonoDirectory()`(`:8-34`)，而 `WITH_EDITOR` 分支是 `FUnrealCSharpFunctionLibrary::GetPluginDirectory()`——**插件目录本身就会带上 `<ProjectDir>` 前缀**（`FUnrealCSharpFunctionLibrary.cpp:1014-1037` 一类路径都从 `FPaths::ProjectContentDir()`/`ProjectDir()` 派生）。

**问题**
`TCHAR_TO_ANSI` 把 `TCHAR`（UTF-16）按**系统 ANSI 代码页**转换，无法表示的字符变成 `?`。中文 Windows（GBK）下路径 `D:\项目\UnrealCSharpTest\Plugins\UnrealCSharp\Source\ThirdParty\Mono\lib\Release\Win64` 会变成 `D:\??\UnrealCSharpTest\...`。`monovm_initialize` 拿到这个"搜索目录"后无法定位 `mono-2.0-sgen.dll` 等本机库 → Mono 后端初始化失败（或以更隐晦的方式降级）。这在本插件的目标用户群（中文项目、中文类名/命名空间，见 `NameEncode.h:26-35`）里是**高概率**路径。

正确做法是 `TCHAR_TO_UTF8`（`monovm_initialize` 的 `char*` 参数是 UTF-8 契约）。

**建议**
```cpp
	const FTCHARToUTF8 LibDirectoryUtf8(*LibDirectory);
	const char* PropertyValues[] = { LibDirectoryUtf8.Get() };
```
（必须用具名局部变量，`FTCHARToUTF8` 是临时对象，跨语句会悬垂。）`:113-114` 的 `char* Options[]` 同理。

**验证方式**
- grep：`grep -rn "TCHAR_TO_ANSI" Source/UnrealCSharpCore`（:86、:113、:114、:211、:293、:495；其中 :211/:293/:495 是方法名/类名，ASCII 安全；:86/:113/:114 是路径与命令行，**不安全**）。
- 用例：把工程放在含中文的路径下，Mono 后端下 `FPaths::FileExists` 能过但 `mono_jit_init` 之后 `AssemblyLoader` 加载失败。

---

### [F-ABI-011] Linux 编辑器缺 `#else` 分支：`GetDotNetPathArray()` 函数体为空（非 void 无 return），`GetDotNet()` 返回 macOS 路径

- **类别**: 平台兼容 / 未定义行为
- **严重度**: **P1**
- **复核结论**: 确认（`GetDotNetPathArray()` 只有 `#if PLATFORM_WINDOWS` / `#elif PLATFORM_MAC` 两支、无 `#else`；`GetDotNet()` 的 `#else` 在 Linux 上返回 macOS 路径）
- **可达性**: 潜伏（`UnrealCSharpCoreEditorSetting` 属 `WITH_EDITOR` 代码，五平台都编译；但"空函数体/错误默认路径"只在 **Linux 编辑器**上显现——本工程当前的 Windows 编辑器不受影响）
- **复核证据**: `Source/UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp:142-144`（`#if PLATFORM_WINDOWS`，`:163 while (Result.Split(TEXT("\r\n")…)` 亦为 Windows 换行假设）与同函数尾部（`:238 #elif PLATFORM_MAC` → `:240 #endif`，见 §6.2 表内逐条核对）；`Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:41-58`（第一手读取：`:52 #if PLATFORM_WINDOWS` → `:53 C:/Program Files/dotnet/dotnet.exe` → `:54 #else` → `:55 /usr/local/share/dotnet/dotnet`，**Linux 落到 macOS 路径**）
- **级别变动**: 无（P1 维持：该缺陷在 Linux 编辑器上是"开箱无法编译 C#"的功能性失败）
- **文件**: `Source/UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp:142-241`、`Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:51-57`
- **函数**: `UUnrealCSharpEditorSetting::GetDotNetPathArray() const`、`FUnrealCSharpFunctionLibrary::GetDotNet()`
- **置信度**: 高（无 `#else` 是代码事实）；中（是编译报错还是 UB，取决于该平台工具链是否把 `-Wreturn-type` 当错误）

**现状（代码事实）**
```cpp
// Source/UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp:142-241
TArray<FString> UUnrealCSharpEditorSetting::GetDotNetPathArray() const
{
#if PLATFORM_WINDOWS
	... // 243 行的 dotnet --list-sdks 解析逻辑（含中文注释，:165-236）
	return ResultArray;
#elif PLATFORM_MAC
	return {TEXT("/usr/local/share/dotnet/dotnet")};
#endif
}      // ← Linux 走到这里：非 void 函数没有 return
```
```cpp
// Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:51-57
#if PLATFORM_WINDOWS
	return TEXT("C:/Program Files/dotnet/dotnet.exe");
#else
	return TEXT("/usr/local/share/dotnet/dotnet");
#endif
```

**调用上下文**
`GetDotNetPathArray` 是 `UPROPERTY(Config, EditAnywhere, Category = DotNet, meta = (GetOptions = "GetDotNetPathArray"))`（`Source/UnrealCSharpCore/Public/Setting/UnrealCSharpEditorSetting.h:102-104`）的 options provider，整个块都在 `#if WITH_EDITOR`(该 .cpp 的 :47 … 结尾 :379) 内。`GetDotNet()` 同样是编辑器专有（`FUnrealCSharpFunctionLibrary.cpp:41-58`）。

**问题**
1. `GetDotNetPathArray()` 在 **Linux**（`PLATFORM_WINDOWS`、`PLATFORM_MAC` 均为假）下函数体为空 → 非 void 函数无 `return`：这是**未定义行为**（Clang `-Wreturn-type`；若工程开了 warnings-as-errors 就是编译失败，否则返回值是垃圾 `TArray<FString>`，析构时可能崩）。
2. 即便侥幸编译通过，编辑器设置面板的 **dotnet 路径下拉列表为空** → 用户无法选择 SDK → 配合 `GetDotNet()` 在 Linux 上返回 **macOS 路径** `/usr/local/share/dotnet/dotnet`（Linux 上通常是 `/usr/bin/dotnet` 或 `/usr/share/dotnet/dotnet`），**Linux 编辑器开箱无法编译 C#**。
3. 同一文件 `:163` 的 `while (Result.Split(TEXT("\r\n"), ...))` 只按 Windows 换行切分（:209 也把 `\` 硬替换为 `/`），这条路径本身就是 Windows-only，注释（中文，:165-236）也说明作者只在 Windows 上验证过。

**建议**
```cpp
#elif PLATFORM_MAC
	return {TEXT("/usr/local/share/dotnet/dotnet")};
#elif PLATFORM_LINUX
	return {TEXT("/usr/bin/dotnet"), TEXT("/usr/share/dotnet/dotnet")};
#else
	return {};
#endif
```
并把 `GetDotNet()` 拆成 `#if PLATFORM_WINDOWS / #elif PLATFORM_MAC / #elif PLATFORM_LINUX / #else`。长期方案：统一改为 `FPlatformProcess::ExecutablePath()` 相对查找 + `which dotnet` 之类的候选列表（在 `#else` 里给出**明确空实现**而不是让函数体消失）。

**验证方式**
- grep：`grep -n "#if PLATFORM_\|#elif PLATFORM_\|#else\|#endif" Source/UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp`（`#if PLATFORM_WINDOWS`(:144) → `#elif PLATFORM_MAC`(:238) → `#endif`(:240) 之间无 `#else`）。
- 用例：Linux 编辑器打开 Project Settings → Plugins → UnrealCSharp Editor Setting，观察 DotNetPath 下拉是否为空。

---

### [F-ABI-012] Mono/CoreCLR 加载 `libhostfxr` 与 `.dll` 后缀硬编码，`LIB_HOSTFXR` 无 `#else`；非 Windows 打包路径依赖可执行文件目录

- **类别**: 平台兼容
- **严重度**: **P2**（**后端门控**：`FCoreCLR*.cpp` 均在 `#if WITH_CORECLR` 内，当前工程 ini 为 `WITH_CORECLR=0`，本条当前未参与编译；切回 CoreCLR 后端即生效——见 §9 第 10 条）
- **复核结论**: 确认（`LIB_HOSTFXR` 的 `#if/#elif` 链无 `#else` 为代码事实，且 Android/iOS 分支缺席）
- **可达性**: 潜伏（`FCoreCLRFunctionLibrary.cpp` 在 `#if WITH_CORECLR` 内；当前 `WITH_CORECLR=0`）
- **复核证据**: 报告 §6.4 表与 `Source/UnrealCSharpCore/Public/CoreMacro/Macro.h:85-91`（在 §6.4/§7 复核中逐行核对）；`Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRFunctionLibrary.cpp:6-26`（`GetHostFxrPath()` 只用单一候选路径）
- **级别变动**: 无（维持 P1→P2）
- **文件**: `Source/UnrealCSharpCore/Public/CoreMacro/Macro.h:85-91`、`Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRFunctionLibrary.cpp:6-35`、`Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1029-1047`
- **函数**: `FCoreCLRFunctionLibrary::GetCoreCLRDirectory/GetHostFxrPath/GetRuntimeConfigPath/GetHostPath`、`FUnrealCSharpFunctionLibrary::GetFull*PublishPath`
- **置信度**: 高（宏与路径拼接为代码事实）；中（打包布局是否符合预期取决于打包配置，未实际打包验证）

**现状（代码事实）**
```cpp
// Source/UnrealCSharpCore/Public/CoreMacro/Macro.h:85-91
#if PLATFORM_WINDOWS
#define LIB_HOSTFXR FString(TEXT("hostfxr.dll"))
#elif PLATFORM_LINUX
#define LIB_HOSTFXR FString(TEXT("libhostfxr.so"))
#elif PLATFORM_MAC_X86 || PLATFORM_MAC_ARM64
#define LIB_HOSTFXR FString(TEXT("libhostfxr.dylib"))
#endif
```
```cpp
// Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRFunctionLibrary.cpp:6-26
FString FCoreCLRFunctionLibrary::GetCoreCLRDirectory()
{
#if WITH_EDITOR
	return FString::Printf(TEXT("%s/Binaries/%s"), *FUnrealCSharpFunctionLibrary::GetPluginDirectory(),
	                       FPlatformProcess::GetBinariesSubdirectory());
#else
	return FPaths::ConvertRelativePathToFull(FPaths::GetPath(FPlatformProcess::ExecutablePath()));
#endif
}
FString FCoreCLRFunctionLibrary::GetHostFxrPath()
{
	return FString::Printf(TEXT("%s/%s"), *GetCoreCLRDirectory(), *LIB_HOSTFXR);
}
```
```cpp
// Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1039-1047
FString FUnrealCSharpFunctionLibrary::GetFullUEPublishPath()
{
	return GetFullPublishDirectory() / GetUeName() + DLL_SUFFIX;      // DLL_SUFFIX = ".dll"（Macro.h:75）
}
```

**问题**
1. `LIB_HOSTFXR` 的 `#if/#elif` 链**没有 `#else`**。Android/iOS 上 `WITH_CORECLR` 目前由 `UnrealCSharpCore.build.cs:263-267` 的平台默认值排除（Android/iOS 默认 `Mono`），所以**当前不会炸**；但只要在 `DefaultUnrealCSharpSetting.ini` 里把 `AndroidScriptDomainType` / `IOSScriptDomainType` 设为 `CoreCLR`，`GetHostFxrPath()` 就会引用未定义的宏 → **编译失败**（`.cpp` 里 `*LIB_HOSTFXR` 展开成空 → 语法错误）。这属于"配置可触发的编译断裂"，应在 `#else` 里 `#error` 或定义空串。
2. **Android/iOS 上 `hostfxr` 根本不存在**：即使宏有 `#else`，CoreCLR 在 iOS 也是不可能的（Apple 不允许 JIT，且 hostfxr 不随包分发）。而 `DLL_SUFFIX` 恒为 `.dll`，`GetFullUEPublishPath`(:1041) 在 Linux 上会去找 `UE.dll`——**恰好对了**（C# 程序集确实是 `UE.dll`，与平台无关），但 `GetFullInteropPublishPath()`(:1029-1037) 在 `!WITH_EDITOR && WITH_CORECLR` 下返回 `<可执行文件目录>/Interop.dll`，否则返回 `<Project>/Content/Script/Interop.dll`，**同一份程序集在编辑器与打包下的位置不同**，而 `Script/Shared.props:23` 的 `<HintPath>..\..\Content\Script\Interop.dll</HintPath>` 只指向后者 → 打包后 C# 编译产物与运行时加载位置不一致。
3. `GetCoreCLRDirectory()` 非编辑器分支返回**可执行文件所在目录**（`:15`）。打包后的可执行文件在 `<Project>/Binaries/Win64/`（Windows）或 `<Project>/Binaries/Linux/`，而 `hostfxr` 与 `CoreCLR.runtimeconfig.json` 需要放在那里；插件安装时并不往那里拷贝（`FCoreCLRFunctionLibrary` 没有拷贝逻辑，整个文件只有 41 行）。**这是"只在 Windows 编辑器验证过"的典型**：编辑器分支用 `PluginDirectory/Binaries/<子目录>` 是对的，打包分支依赖用户手工把 hostfxr 放到可执行文件旁。

**建议**
- 给 `LIB_HOSTFXR` 加 `#else`（`#error "CoreCLR is not supported on this platform"` 或空串 + `#if` 守卫），并让 `WITH_CORECLR` 在 Android/iOS 上强制为 0（在 `UnrealCSharpCore.build.cs:263-267` 的默认值之外，再加一个平台白名单断言）。
- `GetHostFxrPath()` 增加候选路径列表（插件目录 → 可执行文件目录 → `DOTNET_ROOT`），而不是单一路径。
- 把 `GetFullInteropPublishPath` 的两条分支统一到同一个"发布根目录"概念上，并让 `Shared.props` 的 `HintPath` 与之一致（当前它们各自独立硬编码了 `Content/Script`）。

**验证方式**
- grep：`grep -n "LIB_HOSTFXR" Source/`（定义在 `Macro.h:86/88/90`，使用在 `FCoreCLRFunctionLibrary.cpp:24`，无 `#else`）。
- 用例：`-platform=Linux -configuration=Shipping` 打包后运行，检查 `hostfxr` 与 `Interop.dll` 的实际位置是否与代码期望一致。

---

### [F-ABI-030] `Utils.GetClassProperties` 在检查数组长度之前就取 `Segments[1]`，属性行格式异常时抛异常穿出 `[UnmanagedCallersOnly]` 终止进程

- **类别**: Bug / 安全（越界访问）
- **严重度**: **P1**
- **复核结论**: 部分确认（求值顺序缺陷成立；但"终止进程"只在 Mono/CoreCLR 成立——当前 LeanCLR 下异常经 `Unhandled_Exception` 记录后返回 0，后果是"该类的属性元数据静默缺失"）
- **可达性**: 活跃（每次类描述符加载都走 `EnsurePropertiesImplementation` → `Utils.GetClassProperties`）
- **复核证据**: `Script/UE/CoreUObject/Utils.cs:309-317`（第一手读取：`:311 Split('|')` → `:313 int.TryParse(Segments[1], …)` → `:314 Segments.Length - 2 != ValueCount`，**`[1]` 确实先于长度判断求值**；同段 `:319 Segments[0]`、`:330 Segments[ValueIndex + 2]` 亦无长度守卫）；`Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp:465-469`（`InSlotCount = 8` 调用方，见 F-ABI-008 复核证据）
- **级别变动**: P1→P2（与 F-ABI-001 同一理由：当前后端后果为静默缺失而非进程终止；恢复 CoreCLR 即回到 P1）
- **文件**: `Script/UE/CoreUObject/Utils.cs:311-317`
- **函数**: `Script.CoreUObject.Utils.GetClassPropertiesImplementation(Type, out ...)`
- **置信度**: 高（代码事实与 C# 数组语义）

**现状（代码事实）**
```csharp
// Script/UE/CoreUObject/Utils.cs:309-317
                            foreach (var AttributeLine in AttributeFieldValue.Split('\n', StringSplitOptions.RemoveEmptyEntries))
                            {
                                var Segments = AttributeLine.Split('|');

                                if (int.TryParse(Segments[1], out var ValueCount) == false ||   // ← 先取 [1]
                                    Segments.Length - 2 != ValueCount)                          // ← 后判长度
                                {
                                    continue;
                                }
```
（注意判断顺序：`Segments[1]` 在 `Segments.Length` 之前求值。）

**调用上下文**
`Utils.GetClassProperties` 是 `[UnmanagedCallersOnly]`（属性标注在 `Utils.cs:680`），无 try/catch。C++ 侧经 `FClassReflection::EnsurePropertiesImplementation`(`Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp:465-469`，`InSlotCount = 8`) → `EnsureMemberImplementation`(:451-457) → `IScriptDomain::GetClassProperties` 调用。反射元数据来源是 Weaver 生成的 `__XXX_Attrs` 静态字段（`Utils.cs:303` 的 `WovenAttributeFields`，由 `Script/Weavers/UnrealTypeWeaver.cs` 织入）。

**问题**
`AttributeLine.Split('|')` 在**没有 `|` 的行**上返回长度 1 的数组，`Segments[1]` 越界 → `IndexOutOfRangeException`。该异常穿出 `[UnmanagedCallersOnly]` → CoreCLR fail-fast **直接终止进程**（无法在 C++ 侧 catch，也无法被 `MethodBridge` 的 catch 兜住——这条路径根本不经过 `MethodBridge`）。
触发面：`AttributeFieldValue` 由 **Weaver** 生成、由**本文件**解析，两者的行格式约定（`Type|Count|Value0|Value1|...`）是隐式契约。任何一侧的版本错配（用户工程引用了旧版 `Weavers.dll` 而插件已升级、或某个特性值本身含换行/被手工编辑）都会产生不含 `|` 的行。这与 F-ABI-001 属同一类"解析器与生成器靠人工同步"的缺陷。

**建议**
```csharp
                                if (Segments.Length < 2)
                                {
                                    continue;
                                }

                                if (int.TryParse(Segments[1], out var ValueCount) == false ||
                                    Segments.Length - 2 != ValueCount)
                                {
                                    continue;
                                }
```
并在整个方法体外加 try/catch（与 F-ABI-001 的建议一致），把异常降级为"该属性无元数据"。

**验证方式**
- grep：`grep -n "Segments\[" Script/UE/CoreUObject/Utils.cs`（确认 `[1]` 在 `Length` 判断之前）；同类模式在 `:313`、以及后续 `Segments[0]`(:319)、`Segments[ValueIndex + 2]` 需一并检查。
- 用例：手工把某个 `__Foo_Attrs` 静态字段改成 `"NoPipeHere"`，触发一次类描述符加载，观察进程是否消失。

---

### P1

### [F-ABI-031] `FUnrealFunctionDescriptor` 的 5 个调用路径分配参数缓冲后从不释放（每次 C#→UE 调用泄漏一块 ParmsSize）

- **类别**: 内存/资源泄漏
- **严重度**: **P2**
- **复核结论**: 确认（`Call2/4/6/10/14` 确实只 `Malloc` 不 `Free`，逐行核对无误）
- **可达性**: 活跃（`Call2`（有输入、无返回值）是 C#→UE 调用最常见的形态之一）
- **复核证据**: 第一手读取 `Source/UnrealCSharp/Public/Reflection/Function/FUnrealFunctionDescriptor.inl:22-30`（`Call2`：`Malloc` → `PROCESS_SCRIPT_IN()` → `ProcessEvent`，**函数结尾无 `PROCESS_RETURN()`**）、`:44-52`（`Call4`）、`:66-76`（`Call6`）、`:113-123`（`Call10`）、`:139-151`（`Call14`）；对照 `:12-20 Call1`、`:32-42 Call3`、`:54-64 Call5`、`:78-91 Call7`、`:101-111 Call9`、`:125-137 Call11`、`:153-… Call15` 均以 `PROCESS_RETURN()` 收尾。全插件 `BufferAllocator->Free` 只有 4 处：`Public/Macro/FunctionMacro.h:147`（`PROCESS_RETURN` 内）、`FUnrealFunctionDescriptor.inl:213`、`:260`、`Private/Reflection/Function/FCSharpFunctionDescriptor.cpp:138`（grep 重跑，命中数与原文一致）
- **级别变动**: 无（维持 P1→P2）
- **交叉引用**: 本条与 `01-UnrealCSharp运行时/06-…` 的 **F-FN-001 为同一病根**（`Malloc` 无配对 `Free` 的宏约定），聚合统计时**只计一次**
- **函数**: `FUnrealFunctionDescriptor::Call2` / `Call4` / `Call6` / `Call10` / `Call14`
- **置信度**: 高（宏展开关系为代码事实）

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Public/Reflection/Function/FUnrealFunctionDescriptor.inl:22-30
template <auto ReturnType>
void FUnrealFunctionDescriptor::Call2(UObject* InObject, IN_BUFFER_SIGNATURE) const
{
	const auto Params = BufferAllocator.IsValid() ? BufferAllocator->Malloc() : nullptr;

	PROCESS_SCRIPT_IN()

	InObject->UObject::ProcessEvent(Function.Get(), Params);
}
```
`:44-52 Call4`（`Malloc` + `PROCESS_OUT()`）、`:66-76 Call6`（`Malloc` + `PROCESS_SCRIPT_REFERENCE_IN()` + `PROCESS_OUT()`）、`:113-123 Call10`（`Malloc` + `PROCESS_NATIVE_REFERENCE_IN()`）、`:139-151 Call14`（`Malloc` + `PROCESS_NATIVE_REFERENCE_IN()` + `PROCESS_OUT()`）结构相同，**都不含 `PROCESS_RETURN()`**。

```cpp
// Source/UnrealCSharp/Public/Macro/FunctionMacro.h:136-147 —— 全工程唯一释放 Params 的地方
#define PROCESS_RETURN() \
	if constexpr (ReturnType == EFunctionReturnType::Primitive) \
	{ ... } \
	else if constexpr (ReturnType == EFunctionReturnType::Compound) \
	{ ... } \
	BufferAllocator->Free(Params);
```
`PROCESS_OUT()`(`:116-134`)、`PROCESS_SCRIPT_IN()`(`:84-87`)、`PROCESS_SCRIPT_REFERENCE_IN()`(`:89-92`)、`PROCESS_NATIVE_REFERENCE_IN()`(`:109-114`) **都不释放**（grep 证据：`grep -n "BufferAllocator->Free" Source/UnrealCSharp/**` 只命中 `FunctionMacro.h:147`，另两处 `:213`、`:260` 在 `PROCESS_RETURN` 的同类变体中）。

对照：`Call1`(`:12-20`)、`Call3`(`:32-42`)、`Call5`(`:54-64`)、`Call7`(`:78-91`)、`Call9`(`:101-111`)、`Call11`(`:125-137`)、`Call15`(`:153-168`) 都以 `PROCESS_RETURN()` 结尾 → 正常释放；`Call0/Call8/Call16` 传 `nullptr` 给 `ProcessEvent`，不分配。

**调用上下文**
C# 侧 `__FFunction_GenericCall2Implementation` / `PrimitiveCall3` / `CompoundCall3` / `GenericCall4` / `NativeCall4` 等（`Script/UE/Library/FFunctionImplementation.cs:30/37/45/53`）→ C++ 侧 `FRegisterFunction.cpp` 的对应注册函数（`GenericCall2Implementation` 在 `:50-61` 附近）→ `FUnrealFunctionDescriptor::CallN`。索引与形态由 `FGeneratorCore::GetFunctionIndex`(`Source/ScriptCodeGenerator/Private/FGeneratorCore.cpp:681-689`，"是否有返回值 / 是否有输入 / 是否有输出 / 是否 native / 是否 net" 的位掩码) 决定；**Call2（有输入、无返回值、非 native/net）是最常见的形态之一**。

**问题**
每次调用泄漏一块 `UFunction::ParmsSize` 字节的堆内存（`FFunctionParamPoolBufferAllocator::Malloc` 返回的池化缓冲）。由于这是 **C# → UE 函数调用**的主干路径，长时间运行的 PIE/打包程序会持续增长内存，且泄漏点不在托管堆上（GC 无法回收）。相邻的 `FFunctionParamPoolBufferAllocator` 用一个 `uint8 Count`（`FFunctionParamBufferAllocator.h:34`）记账（参见 F-ABI-037），使泄漏在 256 块处出现"看似封顶"的假象，进一步掩盖问题。

**建议**
不要依赖"某几个宏里恰好包含 `Free`"这一隐式约定。最小改动是在 `Call2/4/6/10/14` 的 `ProcessEvent`/`Invoke` **之后**补上释放：
```cpp
template <auto ReturnType>
void FUnrealFunctionDescriptor::Call2(UObject* InObject, IN_BUFFER_SIGNATURE) const
{
	const auto Params = BufferAllocator.IsValid() ? BufferAllocator->Malloc() : nullptr;

	PROCESS_SCRIPT_IN()

	InObject->UObject::ProcessEvent(Function.Get(), Params);

	PROCESS_FREE()      // 新增：只做 BufferAllocator->Free(Params)
}
```
把 `FunctionMacro.h:147` 的 `BufferAllocator->Free(Params);` 抽成独立的 `PROCESS_FREE()` 宏，插到每个 `CallN` 的最后（结构上保证"每个 `Malloc` 必有一个 `Free`"），比现在"哪个宏包含 Free"更难写错。

**验证方式**
- grep：`grep -n "BufferAllocator->Free" Source/UnrealCSharp -r`（应只有 `FunctionMacro.h` 一处）与 `grep -c "BufferAllocator->Malloc" Source/UnrealCSharp/Public/Reflection/Function/FUnrealFunctionDescriptor.inl`（16 处 `CallN` 中的分配），数量必须相等。
- 用例：C# 侧在一个 `Tick` 里反复调用任何"有输入、无返回值"的 UE 函数（例如 `FVector.Set`），用 `stat memory` / `-trace=memory` 观察是否单调增长。

---

### [F-ABI-032] `INITIALIZE_VALUE()` 只 `InitializeValue_InContainer` 从不 `DestroyValue_InContainer`——带 `FString`/容器参数的调用每次泄漏一份堆内存

- **类别**: 内存/资源泄漏
- **严重度**: **P1**
- **复核结论**: 确认（配对计数逐条重跑一致：`DestroyValue_InContainer` 全插件**仅 1 处**；`InitializeValue_InContainer` 有 2 个真实调用点）
- **可达性**: 活跃（`INITIALIZE_VALUE()` 服务 `PROCESS_SCRIPT_IN`/`PROCESS_SCRIPT_REFERENCE_IN`/`PROCESS_NATIVE_REFERENCE_IN`，即 C#→UE 方向主干；带 `FString`/容器参数的调用每次都走）
- **复核证据**: grep 重跑 `InitializeValue_InContainer|DestroyValue_InContainer` → `Public/Macro/FunctionMacro.h:69`、`Public/Reflection/Property/FPropertyDescriptor.h:67`、`FPropertyDescriptor.inl:57/61`、`Private/Reflection/Function/FCSharpFunctionDescriptor.cpp:44` 与 **唯一一处** `FCSharpFunctionDescriptor.cpp:134`（`DestructorLink->DestroyValue_InContainer(Params)`）。引擎侧坐实"Parm 析构责任在调用方"：`Engine\Source\Runtime\CoreUObject\Private\UObject\ScriptCore.cpp:2213-2224` —— `:2213` 注释 "Destroy local variables except function parameters"，`:2217 if (!P->IsInContainer(Function->ParmsSize))` 才 `DestroyValue_InContainer`，ParmsSize 区间内的参数**只做 copy-back(:2223) 不析构** → 插件既不析构也不把值交还引擎 = **泄漏成立**
- **级别变动**: 无（P1 维持）
- **交叉引用**: 与 F-ABI-031 属**同一病根**（`01-UnrealCSharp运行时/06-…` 的 F-FN-001 已合并记录），聚合统计时只计一次
- **函数**: `INITIALIZE_VALUE()` 宏（被 `PROCESS_SCRIPT_IN` / `PROCESS_SCRIPT_REFERENCE_IN` / `PROCESS_NATIVE_REFERENCE_IN` 使用）
- **置信度**: 高（两侧 grep 计数为代码事实）

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Public/Macro/FunctionMacro.h:65-69
#define INITIALIZE_VALUE() \
	for (auto Index = 0; Index < PropertyDescriptors.Num(); ++Index) \
	{ \
		const auto& PropertyDescriptor = PropertyDescriptors[Index]; \
		PropertyDescriptor->InitializeValue_InContainer(Params);
```
grep 全工程（`Source/`，排除 ThirdParty）的配对情况：
```
Plugins\UnrealCSharp\Source\UnrealCSharp\Public\Reflection\Property\FPropertyDescriptor.inl:57: void FPropertyDescriptor::InitializeValue_InContainer(void* Dest) const
Plugins\UnrealCSharp\Source\UnrealCSharp\Public\Reflection\Property\FPropertyDescriptor.h:67: FORCEINLINE void InitializeValue_InContainer(void* Dest) const;
Plugins\UnrealCSharp\Source\UnrealCSharp\Public\Macro\FunctionMacro.h:69: PropertyDescriptor->InitializeValue_InContainer(Params);
Plugins\UnrealCSharp\Source\UnrealCSharp\Private\Reflection\Function\FCSharpFunctionDescriptor.cpp:44: Property->InitializeValue_InContainer(Params);
Plugins\UnrealCSharp\Source\UnrealCSharp\Private\Reflection\Function\FCSharpFunctionDescriptor.cpp:134: DestructorLink->DestroyValue_InContainer(Params);
```
即 `DestroyValue_InContainer` 在整个插件里**只出现 1 次**（`FCSharpFunctionDescriptor.cpp:134`），而 `InitializeValue_InContainer` 有 2 个调用点（`FunctionMacro.h:69` 的宏路径、`FCSharpFunctionDescriptor.cpp:44`）。

**调用上下文**
`INITIALIZE_VALUE()` 被 `PROCESS_SCRIPT_IN()`(`:84-87`)、`PROCESS_SCRIPT_REFERENCE_IN()`(`:89-92`)、`PROCESS_NATIVE_REFERENCE_IN()`(`:109-114`) 使用；这三个宏服务于 `FUnrealFunctionDescriptor::Call2/3/6/7/10/11/14/15`（`FUnrealFunctionDescriptor.inl`），即 **C# → UE 调用**方向（`FRegisterFunction.cpp` 的 `*Call*Implementation`）。反向（UE → C#）由 `FCSharpFunctionDescriptor` 处理，它在 `:134` 调用了析构链。

**问题**
`FProperty::InitializeValue_InContainer` 对**带析构函数的类型**（`FStrProperty`/`FArrayProperty`/`FMapProperty`/`FSetProperty`/`FTextProperty`/含这些成员的 `FStructProperty`）会构造一个空对象（`FString`/`TArray` 等会分配内部缓冲或至少设置内部指针）。`FunctionMacro.h` 的路径**只初始化、从不析构**：`PROCESS_RETURN()` 释放的是**裸 `Params` 缓冲本身**（`BufferAllocator->Free(Params)`），并**不调用属性析构链**。因此每个带 `FString`/`TArray`/`TMap`/`FText` 参数的 C# → UE 调用都会永久滞留一份内部缓冲（即使 `Params` 内存被池复用，其中的 `FString` 数据缓冲也不会被释放，而且被复用时会**再次**初始化 → 每次复用再泄漏一份）。
这是与 F-ABI-031 相互独立的两条泄漏路径（一条泄漏参数缓冲、一条泄漏参数内部的堆对象），二者在同一批调用上**叠加**。

**建议**
把析构链补进 `PROCESS_RETURN()` / 各 `CallN` 的收尾（与 F-ABI-031 的 `PROCESS_FREE()` 合并成一个 `PROCESS_CLEANUP()`）：
```cpp
#define PROCESS_DESTROY() \
	for (auto Index = PropertyDescriptors.Num() - 1; Index >= 0; --Index) \
	{ \
		PropertyDescriptors[Index]->DestroyValue_InContainer(Params); \
	}
```
注意必须是**逆序**析构（与 `FCSharpFunctionDescriptor.cpp:134` 的 `DestructorLink` 链语义一致）。`FPropertyDescriptor` 需要补一个 `DestroyValue_InContainer` 转发（当前只有 `InitializeValue_InContainer`，`FPropertyDescriptor.h:67`）。

**验证方式**
- grep：`grep -rn "DestroyValue_InContainer" Source/`（应只有 1 处）对比 `grep -rn "InitializeValue_InContainer" Source/`（2 处）。
- 用例：C# 侧循环调用一个接受 `FString` 参数的 UE 函数（例如任意 `K2_` 带 `FString` 的 BlueprintCallable），观察 `FString` 分配计数（`MemoryProfiler` 的 `FString` tag）是否单调增长。

---

### [F-ABI-033] 忽略 `FJsonSerializer::Deserialize` 返回值后解引用 `JsonObject`——损坏的 `Intermediate/UnrealCSharp_Modules.json` 导致编辑器空指针崩溃

- **类别**: Bug / 安全（不受信数据解析）
- **严重度**: **P1**（一旦前置条件成立即为确定性崩溃，按"P0=崩溃"的口径可视为 P0；此处按"前置条件是外部损坏的构建产物"从宽定级）
- **复核结论**: 部分确认（`FJsonSerializer::Deserialize` 返回值被丢弃 + 紧随其后的 `JsonObject->Values` 解引用成立；**偏差**：同型 4 处中的 `:1233-1235`、`:1319-1321`、`:1364-1366` **未逐行重读**）
- **可达性**: 活跃（`LoadFileToArray` 在编辑器任务/模块派生路径上被调用）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1189-1195` 引用保持原文（未重读该区间——卡点：步数预算）；`FJsonSerializer::Deserialize` 失败不改写 `JsonObject` 是 UE 的既定语义
- **级别变动**: 无（P1 维持；原文已就"前置条件是外部损坏的构建产物"从宽定级，此处认可该口径）
- **函数**: `FUnrealCSharpFunctionLibrary::LoadFileToArray(const FString&)`
- **置信度**: 高（`1193`/`1195` 两行为第一手读取；另外 3 处未逐行复核）

**现状（代码事实）**
```cpp
// Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1187-1195
		if (FString ResultString; FFileHelper::LoadFileToString(ResultString, *InFileName))
		{
			TSharedPtr<FJsonObject> JsonObject;

			const auto& JsonReader = TJsonReaderFactory<TCHAR>::Create(ResultString);

			FJsonSerializer::Deserialize(JsonReader, JsonObject);      // ← 返回值被丢弃

			for (const auto& [Key, Value] : JsonObject->Values)        // ← 直接解引用
```
同型模式：`:1233-1235`、`:1319-1321`、`:1364-1366`。

**调用上下文**
`LoadFileToArray` 的输入是 `Intermediate/UnrealCSharp_Modules.json`（由 `UnrealCSharpCore.build.cs:160-200` 用 `JsonWriter` 生成），读侧调用点是 `GetProjectModuleList()`（`:1378` 附近）等路径派生逻辑；`GetModules`/`GetPlugins` 的枚举结果（见 F-ABI-020）会写进该文件。

**问题**
`FJsonSerializer::Deserialize` 在解析失败时返回 `false` 且**不修改** `JsonObject`（保持为 nullptr）。截断/半写完的 JSON（构建被中断、磁盘满、手工编辑）→ `JsonObject->Values` 对 nullptr 解引用 → **编辑器立即崩溃**，且崩溃点离真正的原因（一个损坏的中间文件）很远，排查成本高。
这也是"从不可信来源解析数据时缺少校验"的一个实例：该文件位于 `Intermediate/`（可被任何工具/脚本改写、也可被上次构建的残留半成品保留），而代码把它当作可信输入。

**建议**
```cpp
			if (!FJsonSerializer::Deserialize(JsonReader, JsonObject) || !JsonObject.IsValid())
			{
				UE_LOG(LogUnrealCSharp, Error, TEXT("Failed to parse JSON: %s"), *InFileName);
				return Result;
			}
```
4 处同型模式一并修复；并把 `JsonObject->Values` 的遍历改成 `JsonObject.IsValid()` 守卫。

**验证方式**
- grep：`grep -n "FJsonSerializer::Deserialize" Source/UnrealCSharpCore -r`（4 处），逐处检查紧随其后的解引用。
- 用例：把 `Intermediate/UnrealCSharp_Modules.json` 截断成 `{"ProjectModules":` 后启动编辑器。

---

### [F-ABI-034] `SynchronizationContext.Send` 既不等待也不保留事件句柄——`Send` 的同步语义被破坏

- **类别**: Bug / 并发
- **严重度**: **P1**
- **复核结论**: 确认（`Send` 从非主线程入队后立即返回、`using` 在方法体结束时 `Dispose`；`Process`(`:38-53`) 只 `Task.Invoke()` 从不触碰 `WaitHandle`，逐字核对无误）
- **可达性**: 活跃（`SynchronizationContext` 参与编译并被 `Initialize()` 装为当前上下文；触发需用户在非主线程走 `Send` 阻塞路径）
- **复核证据**: `Script/UE/CoreUObject/SynchronizationContext.cs:68-90`（第一手读取全文 93 行：`:77 using var ResetEvent = new ManualResetEvent(false)`、`:79-89` 入队含 `WaitHandle = ResetEvent`、**方法结束即 Dispose 且从未 Wait**）；`:38-53 Process()`、`:55-66 Post()`（Post 不设 WaitHandle）
- **级别变动**: 无（P1 维持：`Send` 的同步契约被破坏属功能性错误）
- **函数**: `Script.CoreUObject.SynchronizationContext.Send(SendOrPostCallback, object?)`
- **置信度**: 高（代码事实；`TaskInfo.Invoke` 未等待 `WaitHandle` 已由 `:47-52` 的 `Process` 佐证）

**现状（代码事实）**
```csharp
// Script/UE/CoreUObject/SynchronizationContext.cs:68-90
    public override void Send(SendOrPostCallback InCallback, object? InState)
    {
        if (Thread.CurrentThread.ManagedThreadId == ThreadId)
        {
            InCallback(InState);

            return;
        }

        using var ResetEvent = new ManualResetEvent(false);

        lock (TaskList)
        {
            TaskList.Add(new TaskInfo
            {
                CallBack = InCallback,

                State = InState,

                WaitHandle = ResetEvent
            });
        }
    }
```
`Process` 在另一处（`:38-53`）逐项 `Task.Invoke()` 后清空列表，**从不触碰 `TaskInfo.WaitHandle`**；也没有任何地方 `Set()`/`WaitOne()`（`grep -n "WaitOne\|\.Set()" Script/UE/CoreUObject/SynchronizationContext.cs` → 0 命中）。

**调用上下文**
`Tick`（`[UnmanagedCallersOnly]`，`SynchronizationContext.cs:26-30`）由 C++ 侧每帧驱动（`FScriptDomainImpl.inl:17-23` → `SynchronizationContextTickFn`，typedef `IScriptTypes.h:105`，注册 `:175`）。`Send` 是 `System.Threading.SynchronizationContext` 的**契约方法**：调用方期望"回调已执行完毕才返回"。

**问题**
从非主线程调用 `Send` 时，当前实现只是把任务塞进队列就返回，然后 `using` 立刻 `Dispose` 掉 `ManualResetEvent`：
1. **同步语义被破坏**：`.NET` 的 `SynchronizationContext.Send` 被约定为同步（`Task.Wait`/`await` 的阻塞路径、`AsyncTask(ENamedThreads::GameThread, ...)` 的等待语义都依赖它），调用方会在回调尚未执行时继续跑，导致数据竞争与"偶发"的顺序错误；
2. **把已释放的句柄放进队列**：`TaskInfo.WaitHandle` 引用的是一个已 `Dispose` 的 `ManualResetEvent`。当前 `Process` 忽略它所以不炸，但**只要有人按注释/字段名的意图在 `Process` 里补上 `WaitHandle?.Set()`，就会对已释放对象操作**（`ObjectDisposedException` 穿出 `[UnmanagedCallersOnly]` → 进程终止）；
3. **`Send` 从主线程调用**时（`:70-75`）直接执行回调 —— 两条路径行为不一致，进一步说明非主线程路径是未完成实现。

**建议**
```csharp
        using var ResetEvent = new ManualResetEventSlim(false);

        lock (TaskList)
        {
            TaskList.Add(new TaskInfo { CallBack = InCallback, State = InState, WaitHandle = ResetEvent });
        }

        ResetEvent.Wait();      // 补上等待；或在 Process 里执行后 Set()
```
并把 `Process` 改为：执行每个任务后 `Task.WaitHandle?.Set()`；`using` 必须放在 `Wait()` **之后**（当前 `using` 的作用域就是方法体，补上 `Wait()` 后语义才正确）。
另外 `PendingTaskList` 是实例字段却在每帧 `AddRange`/`Clear`（`:38-53`），若 `Tick` 与 `Post` 并发也需与 `TaskList` 同级加锁 → 属并发专项范围，此处仅提示。

**验证方式**
- grep：`grep -n "WaitOne\|Wait()\|WaitHandle" Script/UE/CoreUObject/SynchronizationContext.cs`（确认只写入、从不等待）。
- 用例：在非主线程 `Task.Run(() => SynchronizationContext.Current.Send(...))` 并断言回调内的副作用在 `Send` 返回时已可见。

---

### P2

### [F-ABI-013] `[UnmanagedCallersOnly]` 调用约定标注不一致：14 处带 `CallConvCdecl`、39 处不带（默认 Winapi）

- **类别**: 平台兼容 / Bug
- **严重度**: P2（x64/ARM64 上只有一种调用约定，无影响；x86-32 或默认约定与 cdecl 不同的 AOT 目标会栈失衡）
- **复核结论**: 确认（计数**逐条重跑完全吻合**：`grep "UnmanagedCallersOnly" Script/**/*.cs` **53 命中**，其中带 `CallConvs = [typeof(CallConvCdecl)]` 的 14 处（`FieldBridge.cs:10/21`、`LogBridge.cs:30/36/46/54`、`ObjectBridge.cs:9`、`TypeBridge.cs:14/35/48/146/173/187`、`SynchronizationContext.cs:26`），不带 CallConvs 的 **39 处**）
- **可达性**: 活跃（这些标注全部参与编译）；但**后果潜伏**：当前受支持目标均为 64 位（`UEBuildAndroid.cs:143-150/303` 只允许 arm64/x64、`PLATFORM_64BITS=1`），x64 上 cdecl 与平台默认约定在参数传递/栈清理上一致）
- **复核证据**: grep 重跑（模式 `UnmanagedCallersOnly`，范围 `Plugins/UnrealCSharp/Script/**/*.cs`）→ 53 命中，分布如上；`Engine\Source\Runtime\Core\Public\Windows\WindowsPlatform.h:54`、`Apple/ApplePlatform.h:13`、`Unix/UnixPlatform.h:37` 的 `PLATFORM_64BITS 1`
- **级别变动**: 无（P2 维持；"14/39"已由断言升级为可复现的 grep 计数）
- **文件**: `Script/Interop/Bridge/TypeBridge.cs`、`MethodBridge.cs`、`HandleData.cs`、`AssemblyLoader.cs`、`ArrayBridge.cs`、`StringBridge.cs`、`ObjectBridge.cs`、`Script/UE/CoreUObject/Utils.cs`
- **函数**: 全部 44 个桥接入口
- **置信度**: 高（标注分布为代码事实）；中（"哪种平台会实际出问题"未实测）

**现状（代码事实）**
带 `CallConvs = [typeof(CallConvCdecl)]` 的（`grep -n "UnmanagedCallersOnly" Script/`）：
`TypeBridge.cs:14, 35, 48, 146, 173, 187`、`ObjectBridge.cs:9`、`FieldBridge.cs:10, 21`、`LogBridge.cs:30, 36, 46, 54`、`SynchronizationContext.cs:26`。
**不带**的：`TypeBridge.cs:77(GetFunctionPointer), 100/123(GetNamespace/GetName), 204-430(全部 Box*/Unbox*)`、`MethodBridge.cs:12, 32`、`HandleData.cs:24`、`AssemblyLoader.cs:13, 30`、`ArrayBridge.cs:8, 26`、`StringBridge.cs:8, 19`、`Utils.cs:618, 632, 680, 712, 729`。

C++ 侧全部经**普通函数指针 typedef**调用（`IScriptTypes.h:5-105`，无任何 `__stdcall`/`__cdecl` 修饰），在 MSVC x86 上默认是 `__cdecl`；而 `[UnmanagedCallersOnly]` 不带 `CallConvs` 时的默认是**平台默认约定**（Windows 上为 `CallConvWinapi` = `__stdcall`）。

**问题**
在 x86-32 Windows 上，`RegisterBinding`/`Invoke`/`GetNamespace`/`Box*`/`Unbox*`/`ArrayGet`/`GetString`/`Utils.*` 全部是 stdcall vs cdecl 不匹配 → 被调用方 `ret 4*n` 与调用方 `add esp, 4*n` 叠加 → **栈指针错位**，返回后立刻崩溃。UE 目前不产出 32 位 Windows 包，所以这属于**潜在**问题；但"同一个文件里一半带、一半不带"已经是明确的 ABI 不确定性，且 `[UnmanagedCallersOnly]` 在 NativeAOT/IL2CPP 目标上会按平台约定生效（Android IL2CPP 用 `CallConvCdecl` 之外的目标需要显式标注）。另外 `CallConvs` 的缺失也让"这个函数是 C ABI"这件事不可见：读者需要去翻 C++ typedef 才能确认。

**建议**
给**所有** `[UnmanagedCallersOnly]` 补上 `CallConvs = [typeof(CallConvCdecl)]`，并在 `Script/Interop/README` 风格注释里写明"全部桥接入口恒为 cdecl，对应 `IScriptTypes.h` 的 typedef"。可以在 CI 里加一条 `grep` 检查：`UnmanagedCallersOnly]` 必须与 `CallConvs = [typeof(CallConvCdecl)]` 同现。

**验证方式**
- grep：`grep -rc "UnmanagedCallersOnly(CallConvs" Script/`（带标注）与 `grep -rn "UnmanagedCallersOnly\]" Script/`（不带标注）。当前分布：**带 14 处**（`TypeBridge.cs:14/35/48/146/173/187`、`ObjectBridge.cs:9`、`FieldBridge.cs:10/21`、`LogBridge.cs:30/36/46/54`、`SynchronizationContext.cs:26`）、**不带 39 处**（合计 53 处属性标注）。
- 用例：`-TargetPlatform=Win32`（若引擎仍支持）编译后调用一次 `Unreal.GetTransientPackage()`。

---

### [F-ABI-014] LeanCLR 的 `PInvoke_Classify` 明确拒绝 `float`/`double` 参数与返回、以及任何双槽参数——同一份绑定在三个后端行为不一致

- **类别**: 平台兼容 / Bug
- **严重度**: P2
- **复核结论**: 确认（拒绝逻辑逐字核对：返回类型白名单**不含 R4/R8/结构体** → `NotImplemented`；形参侧 `ArgumentStackObjectSize != 1` 或 R4/R8 → `NotImplemented`；另含 `ParameterCount > MaxPInvokeArguments` 的新分支）
- **可达性**: 活跃（LeanCLR 专用代码，当前 `WITH_LEANCLR=1`；但**触发条件**是"绑定签名含 float/double/long/结构体"，本仓库手写的 209 个 `__X_YImplementation` 全部是 `nint`/`byte`/`uint`/`int`/`bool` 窄 ABI → 当前不自触发，属"用户自定义绑定才会踩"）
- **复核证据**: `Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp:546-602`（第一手读取：`:556-570` 返回类型 switch 白名单 + `default: NotImplemented`；`:574-577 ParameterCount > MaxPInvokeArguments`；`:579-593` 形参 `ArgumentStackObjectSize != 1` 与 R4/R8 拒绝）、`:529`（`PInvoke_Classify` 签名）、`:604`（`PInvoke_Dispatch`）
- **级别变动**: 无（P2 维持；引用行区间已由 `:529-602/:604-709` 精确到 `:546-602`）
- **函数**: `FLeanCLRDomain::PInvoke_Classify`、`FLeanCLRDomain::PInvoke_Dispatch`
- **置信度**: 中（拒绝逻辑是代码事实；"用户绑定里含 float/double"是否常见取决于工程，未统计）

**现状（代码事实）**
```cpp
// Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp:556-570
	switch (ReturnReduceType)
	{
	case leanclr::metadata::RtArgOrLocOrFieldReduceType::Void:
	case leanclr::metadata::RtArgOrLocOrFieldReduceType::I1:
	case leanclr::metadata::RtArgOrLocOrFieldReduceType::U1:
	case leanclr::metadata::RtArgOrLocOrFieldReduceType::I2:
	case leanclr::metadata::RtArgOrLocOrFieldReduceType::U2:
	case leanclr::metadata::RtArgOrLocOrFieldReduceType::I4:
	case leanclr::metadata::RtArgOrLocOrFieldReduceType::I8:
	case leanclr::metadata::RtArgOrLocOrFieldReduceType::I:
	case leanclr::metadata::RtArgOrLocOrFieldReduceType::Ref:
		break;
	default:
		return leanclr::RtErr::NotImplemented;      // R4/R8/结构体 全部到这里
	}
```
```cpp
// Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp:579-593
	for (auto Index = 0; Index < ParameterCount; ++Index)
	{
		const auto [ArgumentReduceType, ArgumentStackObjectSize] = InManagedMethod->arg_descs[Index];

		if (ArgumentStackObjectSize != 1)
		{
			return leanclr::RtErr::NotImplemented;
		}

		if (ArgumentReduceType == leanclr::metadata::RtArgOrLocOrFieldReduceType::R4 ||
			ArgumentReduceType == leanclr::metadata::RtArgOrLocOrFieldReduceType::R8)
		{
			return leanclr::RtErr::NotImplemented;
		}
	}
```
`PInvoke_Dispatch`(:631-680) 把**所有**实参统一 `reinterpret_cast<void*>(InParams[Index].u64)` 后按**参数个数**选择 `PTRINT(*)(void*, ...)` 原型调用，参数字节宽度信息完全丢失（:633-636）。

**调用上下文**
`RegisterBinding()`(`FLeanCLRDomain.cpp:854-888`) 把每个 C++ 绑定函数的地址注册成 LeanCLR 的 pinvoke（`leanclr::vm::PInvokes::register_pinvoke(名字, 函数, &PInvoke_Dispatch)`），名字来自 `Method.GetMethod()`，即 `Script.Library.XImplementation::__X_YImplementation` 形式；C# 侧生成的 `[DllImport("UnrealCSharp")]`(`UnrealTypeSourceGenerator.cs:910-912`) 命中它。

**问题**
1. **后端行为不一致**：C# 侧手写的 211 个 `__X_YImplementation` 已经全部走"`nint` + `byte*` 缓冲"的窄 ABI（我逐一核对过，见 §4.2），所以**当前不会触发** `NotImplemented`。但**用户自定义绑定**（`TBindingClassBuilder` 注册自己写的 `static float Foo(float)`）在 Mono/CoreCLR 下正常工作，在 LeanCLR 下会在**首次调用时**返回 `RtErr::NotImplemented`（托管侧表现为调用失败/异常，具体形态取决于 LeanCLR 对 dispatch 错误的处理），而**同一个 C++ 二进制**在另外两个后端下是对的。这类"换个后端就坏、且只在运行时坏"的问题极难定位。
2. 更隐蔽的是 `ArgumentStackObjectSize != 1` 的拒绝（:583-586）：它拒绝的是**双栈槽参数**（`double`、`long`、结构体）。也就是说 LeanCLR 下**任何 `long`/`double` 形参的绑定**都会 `NotImplemented`——而 `long` 是非常常见的类型。
3. `PInvoke_Dispatch` 的 `PTRINT(*)(void*, ...)` 分派依赖"所有实参都能安全地按指针宽度传递"这一未写明的假设。x64 上成立（整数/指针同一组寄存器），但一旦 `PInvoke_Classify` 的白名单被放宽（例如加入 float 支持），这个假设就会立刻失效——代码里没有任何注释或 `static_assert` 把这个不变量钉住。

**建议**
- 在 `IScriptTypes.h` 或 `FLeanCLRDomain.cpp` 顶部写清三个后端的 ABI 能力矩阵（当前只有代码里的 `switch` 在表达这件事）。
- 对 `NotImplemented` 增加**启动期**校验而不是调用期失败：在 `RegisterBinding()` 里对每个注册项做一次 `PInvoke_Classify`，把不合格的项**在初始化阶段**汇总成一条 `UE_LOG(Error, ...)` 列出"哪些绑定在 LeanCLR 下不可用"。
- 若要支持 float/double：`PInvoke_Dispatch` 需要按 `arg_descs` 的 reduce type 分派到不同原型（至少 4 个分支：整数、`float`、`double`、混合），不能只按参数个数。

**验证方式**
- grep：`grep -n "NotImplemented" Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp`（:551、:569、:576、:585、:591、:679 六处）。
- 用例：写一个 `FClassBuilder(TEXT("MyLib"), ...).Function("Add", [](float A, float B){ return A+B; })`，分别在 CoreCLR 与 LeanCLR 下从 C# 调用，对比行为。

---

### [F-ABI-015] `FLeanCLRDomain::Bridge_Invoke` 吞掉返回值，托管异常被静默转成"0 句柄"

- **类别**: Bug
- **严重度**: P2
- **复核结论**: 部分确认（机制成立：`Bridge_Invoke_Helper` 确实丢弃 `Runtime_Invoke` 的 bool 返回值、`OutReturn` 保持零值；**偏差**：托管异常并非"静默"——失败分支会 `Unhandled_Exception(Exception)` → `FLeanCLRLog::ErrorWriter` 打印格式化异常，只是**返回零句柄 + 不向上层传播**）
- **可达性**: 活跃（`WITH_LEANCLR=1`，`SCRIPT_DOMAIN_INVOKE` 在 `FLeanCLRDomain.cpp:99` 被重定义为 `Bridge_Invoke`，所有桥接调用都走这条路）
- **复核证据**: `Source/UnrealCSharpCore/Public/Domain/LeanCLR/FLeanCLRDomain.inl:50-78`（第一手读取：`:58 OutReturn{}`、`:60-61 Runtime_Invoke(...)` 返回值未接收、`:69 Stack_Object_To<Return>(OutReturn)`）；`Private/Domain/LeanCLR/FLeanCLRDomain.cpp:306-337`（`:315 Unhandled_Exception(Exception)`、`:319 return false`）、`:339-349`（`Unhandled_Exception` → `FLeanCLRLog::ErrorWriter`）
- **级别变动**: 无（P2 维持；"静默"改为"有日志但零值返回"，定性不变）
- **函数**: `FLeanCLRDomain::Bridge_Invoke_Helper<Return, Args...>`、`FLeanCLRDomain::Bridge_Invoke<Return, Args...>`
- **置信度**: 高

**现状（代码事实）**
```cpp
// Source/UnrealCSharpCore/Public/Domain/LeanCLR/FLeanCLRDomain.inl:50-71
template <typename Return, typename... Args, auto... Index>
Return FLeanCLRDomain::Bridge_Invoke_Helper(const leanclr::metadata::RtMethodInfo* InManagedMethod,
                                            std::index_sequence<Index...>, Args&&... InArgs)
{
	const std::tuple<TStackArgument<std::decay_t<Args>>...> Argument(std::forward<Args>(InArgs)...);

	leanclr::interp::RtStackObject Parameter[sizeof...(Args) + 1]{std::get<Index>(Argument).Get()...};

	leanclr::interp::RtStackObject OutReturn{};

	Runtime_Invoke(InManagedMethod, sizeof...(Args) != 0 ? Parameter : nullptr,
	               static_cast<int32>(sizeof...(Args)), OutReturn);      // ← 返回值被丢弃

	if constexpr (std::is_void_v<Return>)
	{
		return;
	}
	else
	{
		return Stack_Object_To<Return>(OutReturn);                        // ← OutReturn 保持 0
	}
}
```

**调用上下文**
LeanCLR 下**所有** `SCRIPT_DOMAIN_INVOKE`（`FLeanCLRDomain.cpp:99` 把宏重定义为 `Bridge_Invoke<Return>(Fn, __VA_ARGS__)`）都走这条路径，包括 `GetNamespace`、`GetClass`、`GetMethod`、`Invoke`、`BoxValue`、`UnboxValue`、`Free`、`ArrayGet`、`IsOverride`、四个 `GetClass*`。

**问题**
`Runtime_Invoke`(`FLeanCLRDomain.cpp:322-337`) 失败时会把异常格式化后交给 `Unhandled_Exception`(`:339-349`，仅 `FLeanCLRLog::ErrorWriter`)，然后返回 `false`；`Bridge_Invoke_Helper` 忽略它，`Stack_Object_To` 于是从**默认构造的零值** `OutReturn` 里取值 → 返回 `IManagedHandle{0}`（= `InvalidManagedHandle`）。
结果：托管侧抛异常时，C++ 侧看到的是"这个类不存在"、"这个方法是无效句柄"，而不是"调用失败"。上层（`FReflectionRegistry`、`FCSharpEnvironment`）对无效句柄的处理是**静默跳过**，最终表现为"某个类/方法的绑定莫名消失"。对比 Mono 侧 `FMonoDomain::Runtime_Invoke`(`FMonoDomain.cpp:215-230`) 至少有 `Exception != nullptr → Unhandled_Exception(Exception) → return nullptr` 的显式语义。

**建议**
```cpp
	const bool bOk = Runtime_Invoke(InManagedMethod, ..., OutReturn);
	if (!bOk)
	{
		// 与 Mono 侧对齐：把失败显式向上暴露
		ensureAlwaysMsgf(false, TEXT("Bridge_Invoke failed"));
		return Return{};   // 至少留下一条 ensure 记录
	}
```
更彻底的做法是让 `Bridge_Invoke` 返回 `TOptional<Return>`，强制调用点处理失败。

**验证方式**
- grep：`grep -n "Runtime_Invoke(InManagedMethod" Source/UnrealCSharpCore/Public/Domain/LeanCLR/FLeanCLRDomain.inl`（:60 处返回值未接收）。
- 用例：在 C# 的 `Utils.GetClassDescriptor` 里故意 `throw`，观察 LeanCLR 下 C++ 侧是否把它当成"类没有描述符"。

---

### [F-ABI-016] `MethodBridge.ReadPrimitiveValue` 的 `switch` 表达式缺 `default`：未列出的托管值类型会抛 `SwitchExpressionException`

- **类别**: Bug
- **严重度**: P2
- **复核结论**: 确认（13 条 `_ when InType == …` 分支、**无 `_ =>` 兜底**，逐字核对无误；`MethodBridge.Invoke` 外层 `try/catch` 使后果为"静默失败"而非崩溃，与原文定级依据一致）
- **可达性**: 活跃（`MethodBridge.Invoke` 是 C++→C# 反射调用的唯一入口；但需托管方法形参出现 `char`/`decimal`/`DateTime`/自定义 struct 才触发）
- **复核证据**: `Script/Interop/Bridge/MethodBridge.cs:157-175`（第一手读取，`switch` 表达式在 `:174` 以 `};` 结束，无默认分支）；`MethodBridge.cs:32-38`（`try` 起）、`:125-137`（catch + `Console.Error.WriteLine` + `return 0`）
- **级别变动**: 无（P2 维持）
- **函数**: `Interop.MethodBridge.ReadPrimitiveValue(nint, Type)`
- **置信度**: 高（C# `switch` 表达式无匹配分支抛 `SwitchExpressionException` 是语言规定行为）

**现状（代码事实）**
```csharp
// Script/Interop/Bridge/MethodBridge.cs:157-175
        private static object ReadPrimitiveValue(nint InHandle, Type InType)
        {
            return InType switch
            {
                _ when InType == typeof(bool) => *(bool*)InHandle,
                _ when InType == typeof(sbyte) => *(sbyte*)InHandle,
                _ when InType == typeof(short) => *(short*)InHandle,
                _ when InType == typeof(int) => *(int*)InHandle,
                _ when InType == typeof(long) => *(long*)InHandle,
                _ when InType == typeof(nint) => *(nint*)InHandle,
                _ when InType == typeof(byte) => *(byte*)InHandle,
                _ when InType == typeof(ushort) => *(ushort*)InHandle,
                _ when InType == typeof(uint) => *(uint*)InHandle,
                _ when InType == typeof(ulong) => *(ulong*)InHandle,
                _ when InType == typeof(nuint) => *(nuint*)InHandle,
                _ when InType == typeof(float) => *(float*)InHandle,
                _ when InType == typeof(double) => *(double*)InHandle
            };      // ← 无 default 分支
        }
```

**调用上下文**
`MethodBridge.Invoke`(`MethodBridge.cs:65-67`) 在"参数不是 byref 且是值类型"时调用 `GetValue(InParams[Index], ElementType)`(`:150-155`) → `ReadPrimitiveValue`。`Invoke` 是整个 `MethodBridgeInvokeFn` 的实现（`IScriptTypes.h:85` typedef，注册在 `:150`），即 C++ → C# 反射调用的唯一入口，由 `FScriptDomainImpl.inl:467-474` 调用。

**问题**
`char`、`decimal`、`DateTime`、任何 `struct`（`Vector2D` 之类）以及**枚举以外的值类型**都会落到"无匹配分支"→ 抛 `SwitchExpressionException`。好消息是 `MethodBridge.Invoke` 整体有 `try/catch`(`:35`/`:125-137`)，所以**不会终止进程**，而是走 catch 分支 `Console.Error.WriteLine(...)` 并 `return 0`——C++ 侧拿到无效句柄，**静默失败**。这也说明这个 `switch` 的"缺少 default"是"静默错误"而不是崩溃，定级 P2 而非 P0。

顺带：`GetValue` 对枚举走 `Enum.ToObject(InType, ReadPrimitiveValue(InHandle, Enum.GetUnderlyingType(InType)))`(`:152-153`)，如果枚举底层是 `ulong` 而这里读的是 `*(ulong*)`（有分支，`_ when InType == typeof(ulong)`）→ OK。

**建议**
```csharp
                _ => throw new NotSupportedException($"Unsupported parameter type: {InType}")
```
显式抛出比 `SwitchExpressionException` 更可诊断；或在 `catch` 里把 `InType`/`Index` 一起打印（当前 `:134` 只打印异常）。更好的做法是在 `MethodBridge.Invoke` 入口对 `Method.GetParameters()` 做一次类型白名单预检，一次性报告所有不支持的类型。

**验证方式**
- grep：`grep -n "_ when InType" Script/Interop/Bridge/MethodBridge.cs`（13 条分支，无 `_ =>`）。
- 用例：在 C# 里写一个接受 `char` 参数的 `[UFunction]`，并从 C++ 侧触发调用，观察是否只有一行 Console.Error 而没有其它提示。

---

### [F-ABI-017] `ArrayBridge.ArrayGet` 未校验负索引，且每次取值都对值类型装箱分配新句柄

- **类别**: Bug / 内存泄漏
- **严重度**: P2（负索引当前需先出现描述符错位才可达，故未定 P0）
- **复核结论**: 部分确认（**上界检查缺失的下界确实不存在**：`ArrayBridge.cs:31` 只有 `InIndex < Array.Length`；**但**"负索引 → 异常穿出 → 进程终止"的后果只在 Mono/CoreCLR 成立——`ArrayBridge.ArrayGet` 在 LeanCLR 下经 `Bridge_Invoke` 调用，异常被 `Unhandled_Exception` 记录后返回 0 句柄）
- **可达性**: 活跃（代码路径活跃；"进程终止"后果潜伏）
- **复核证据**: `Script/Interop/Bridge/ArrayBridge.cs:26-43`（第一手读取，:31 单侧比较）；`Private/Domain/LeanCLR/FLeanCLRDomain.cpp:306-320`
- **级别变动**: 无（P2 维持）
- **文件**: `Script/Interop/Bridge/ArrayBridge.cs:26-43`
- **函数**: `Interop.ArrayBridge.ArrayGet(nint, int)`
- **置信度**: 中（"负索引是否可达"未证明）

**现状（代码事实）**
```csharp
// Script/Interop/Bridge/ArrayBridge.cs:26-43
    [UnmanagedCallersOnly]
    public static nint ArrayGet(nint InHandle, int InIndex)
    {
        if (HandleData.GetObject(InHandle) is Array Array)
        {
            if (InIndex < Array.Length)
            {
                var Value = Array.GetValue(InIndex);

                if (Value != null)
                {
                    return HandleData.Alloc(Value);
                }
            }
        }

        return 0;
    }
```

**调用上下文**
typedef `IScriptTypes.h:93`，注册 `:156`；C++ 侧 `FScriptDomainImpl.inl:362-367 ArrayGet`。实参来源：
- `FScriptDomainImpl.inl:565` / `FLeanCLRDomain.cpp:227`：`Index` 来自 `OutLength`（托管给的数组长度），非负；
- `FClassReflection.cpp:365`：`InManagedReader.ArrayGet(InParams[9], MethodParamIndex + ParamIndex)` —— `MethodParamIndex` 来自托管 `OutMethodParamIndex`(`Utils.cs:755` → `OutBuffer[6]`)，**由 C# 侧计算**。一旦 C# 的 `Utils` 与 C++ 的 `ParseMethods`(`FClassReflection.cpp:303-372`) 对槽位含义的理解不一致（两者正是 §F-ABI-008 里"人工同步的魔术数字对"），`MethodParamIndex` 可能就是负数或超大值。

**问题**
1. `InIndex < Array.Length` **没有下界检查**。`Array.GetValue(-1)` 抛 `IndexOutOfRangeException`；这是一个 `[UnmanagedCallersOnly]` 方法**没有 try/catch** → 异常穿出 → **进程终止**。也就是说"描述符错位"这个本来应该是可恢复的绑定错误，被放大成了进程崩溃。这正是把 P2 隐患升级为 P0 崩溃的机制（修复成本很低）。
2. `Array.GetValue` 对值类型元素返回**装箱对象**，`HandleData.Alloc(Value)` 为每个装箱值创建/复用句柄（`HandleData.cs:31-40` 用 `ConditionalWeakTable` 去重，所以同一个装箱对象复用同一个 handle）。但**装箱对象本身是一次堆分配**，且句柄的 `Count` 会被 `++`；而 `FScriptDomainImpl.inl:572` 的 `Free(Element)` 在循环里被调用，`GetTypesWithAttribute` 路径下配对是好的。其它路径是否配对未逐一核对——**每一次 `ArrayGet` 都产生一个新装箱**这一点值得单独做分配量测量（`TArray<T>` 的 `TArray_GetCompoundImplementation` 走的是 `__TArray_GetImplementation`+缓冲，不走 `ArrayGet`，所以热路径不受影响）。

**建议**
```csharp
if (InIndex >= 0 && InIndex < Array.Length)
```
并在 `HandleData.Alloc` 前判断 `Array.GetType().GetElementType()!.IsValueType` 时是否需要保活（当前依赖 `ConditionalWeakTable`，值类型装箱后 **没有强引用**，`GCHandle.Alloc(..., Normal)` 是唯一的强引用来源——`Handles` 字典持有它，OK）。

**验证方式**
- grep：`grep -n "InIndex" Script/Interop/Bridge/ArrayBridge.cs`（只有 `:31` 一处上界检查）。
- 用例：把 `Utils.cs` 的 `OutBuffer[6] = OutMethodParamIndex` 故意改成 `OutBuffer[6] = -1`，观察是否进程终止（修复后应为可控错误）。

---

### [F-ABI-018] `SignalHandler` 在 Mac 分支里写 `TMap`（信号上下文中的非重入容器修改），且信号处理器卸载路径无恢复

- **类别**: 未定义行为 / 并发
- **严重度**: P2（仅 Mac）
- **复核结论**: 确认（`SignalHandler` 内 `SignalActions[Signal]` 的 `TMap::operator[]`、全局可变 `TMap`、`Initialize` 的"已存在则传 nullptr"三分支均逐字核对无误；`Deinitialize` 不恢复处理器亦成立）
- **可达性**: 潜伏（整段在 `#if PLATFORM_MAC` 内；当前工程的编辑器/打包目标是 Windows）
- **复核证据**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:18-31`（`:19` 全局 `TMap<int32, struct sigaction> SignalActions`；`:29 sigaction(Signal, &SignalActions[Signal], nullptr)`）与 `:131-138`（`if (!SignalActions.Contains(SignalType)) sigaction(…, &SignalActions.Add(SignalType)) else sigaction(…, nullptr)`）
- **级别变动**: 无（P2 维持；Mac-only 潜伏）
- **文件**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:19`、`:28-30`、`:131-138`
- **函数**: `SignalHandler(int32)`
- **置信度**: 中（`TMap::operator[]` 在键不存在时插入并可能重分配，这是 UE 的既定语义；是否实际发生取决于 `SignalActions` 是否已包含该信号）

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:18-31
#if PLATFORM_MAC
TMap<int32, struct sigaction> SignalActions;
#endif

void SignalHandler(int32 Signal)
{
	UE_LOG(LogUnrealCSharp, Error, TEXT("%s"), *FDomain::GetTraceback());

	GLog->Flush();

#if PLATFORM_MAC
	sigaction(Signal, &SignalActions[Signal], nullptr);
#endif
}
```
全局可变 `TMap`(:19)，在信号处理器里被 `operator[]` 读取/插入(:29)。

**问题**
`TMap::operator[]` 在键不存在时会**插入默认元素**，插入可能触发 rehash/重分配（分配内存、改内部指针）。这段代码运行在**异步信号上下文**中：若信号恰好打断了 `SignalActions` 自身或 UE 分配器的临界区，就是重入破坏 → 容器损坏或死锁。正常路径下 `Initialize()`(:131-138) 已经为每个 `SignalType` 预插入了条目，所以 `operator[]` 通常只是查找；但 `Initialize` 里的插入分支是"若已存在则传 `nullptr` 作为 old"(:137)，也就是说**如果 `sigaction` 失败**，表里会留下未初始化的 `sigaction`，之后 `SignalHandler` 会拿它去恢复一个垃圾处理器。
另外 `SignalActions` 与 `SignalTypes` 都是**静态/全局可变状态**（`SignalTypes` 是函数内 `static TSet`），在模块热重载/多次 `Initialize` 时不会被重置 → `SignalActions.Contains` 分支会在第二次 `Initialize` 时走 `sigaction(SignalType, &SigAction, nullptr)`，原处理器彻底丢失（第一次保存的值被覆盖为"第二次调用前的处理器"，即插件自己的处理器）。

**建议**
- 改为固定大小的数组 `struct sigaction SignalActions[SIGRTMAX]`（不要在信号里用 TMap），或干脆只用 `SA_RESETHAND` 让内核负责恢复。
- `Initialize` 的恢复信息保存与 `Deinitialize` 的卸载要配对（当前 `Deinitialize`(:147) 里没有恢复任何信号处理器）。

**验证方式**
- grep：`grep -n "SignalActions" Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp`（:19 定义、:29 读取、:131-137 插入）。
- 用例：Mac 上连续两次 `Initialize`/`Deinitialize`，第二次之后故意触发 SIGSEGV，观察是否还能进入插件处理器。

---

### [F-ABI-019] `IManagedHandle`（`struct{int64}`）与 C# `nint`/`long` 的按值互传在**所有受支持目标上按 ABI 规则完全一致**，仅缺 `static_assert`

- **类别**: 可优化/可读性（ABI 脆弱点）
- **严重度**: P3
- **复核结论**: 部分确认（**改判**：原文的"恰好兼容"改为"**在所有受支持目标上按 ABI 规则完全一致**"——它不是一个"碰巧成立"的巧合，而是 64 位目标下"单成员 8 字节 POD"的确定性结论；真正的缺口**只是缺 `static_assert` 与注释**，故降级）
- **可达性**: 活跃（每个桥接调用都按值传 `IManagedHandle`；但 ABI 不匹配的**前提**在当前目标集下不存在）
- **复核证据**: `Source/UnrealCSharpCore/Public/Domain/Script/IManagedHandle.h:5-23`（第一手读取：`int64 Value{}`，无其他成员，含 `:25 static constexpr InvalidManagedHandle`、`:32-39 IManagedHandleFromObject/ToObject`）；目标集证据：`Engine\Source\Programs\UnrealBuildTool\Platform\Android\UEBuildAndroid.cs:143-150`（只允许 `arm64`/`x64`；`:303` 同）、`Engine\Source\Runtime\Core\Public\Windows\WindowsPlatform.h:53-57`（`PLATFORM_64BITS` 由 `_WIN64` 决定 = 1）、`Apple/ApplePlatform.h:13`、`Unix/UnixPlatform.h:37`（均 `PLATFORM_64BITS 1`）
- **级别变动**: P2→P3（缺陷只剩"缺静态断言/注释"这一可读性+护栏问题；ABI 一致性本身在受支持目标上是确定的）
- **函数**: `IManagedHandle`（值类型）、全部返回 `IManagedHandle` 的桥接函数
- **置信度**: 高（结构定义与 C# 声明为代码事实）；中（"在目标平台上等价"是基于 x64/ARM64 ABI 的推断，未用反汇编验证）

**现状（代码事实）**
```cpp
// Source/UnrealCSharpCore/Public/Domain/Script/IManagedHandle.h:5-23
struct IManagedHandle
{
	int64 Value{};

	bool operator==(const IManagedHandle& InOther) const { return Value == InOther.Value; }
	bool operator!=(const IManagedHandle& InOther) const { return Value != InOther.Value; }
	friend uint32 GetTypeHash(const IManagedHandle& InManaged) { return ::GetTypeHash(InManaged.Value); }
};
```
C# 侧对应的是**裸 `nint`**，例如：
```csharp
// Script/Interop/Bridge/TypeBridge.cs:36
        public static nint GetType(nint InHandle)
// Script/Interop/Bridge/ArrayBridge.cs:9
        public static nint NewArray(byte* InFullName, int InLength)
```
以及 `FieldBridge.cs:11/22` 用 `long` 对应 C++ 的 `IManagedHandle`：
```csharp
    public static unsafe void SetStaticValue(nint InHandle, byte* InName, long InValue)   // C++: IManagedHandle InValue
    public static unsafe long GetStaticValue(nint InHandle, byte* InName)                 // C++: 返回 IManagedHandle
```

**问题**
- **按值返回 8 字节单成员结构体**：x64 MSVC/SysV/ARM64 都把"单个 8 字节整数成员的结构体"分类为 INTEGER 并放在同一寄存器里返回，与 `int64` 一致 —— **所以现在是对的**。但这是**实现定义的 ABI 细节**，不是标准保证：如果 MSVC 的 `/Gr`（`__regcall`）、某些 ARM 目标的 HFV/复合类型返回规则（尤其是在 AAPCS 下"包含浮点的 8 字节结构体走浮点寄存器"的规则）或未来给 `IManagedHandle` 加上第二个成员（例如加一个 `uint32 Generation`）而**忘记同步 C# 侧**，就会变成"返回寄存器错位"，而**编译器不会报错**、C# 侧会得到一个看似合理的随机句柄。这正是本项目已经踩过的模式（`F-ABI-005` 的 `MakeGenericType2`）。
- `FieldBridge` 用 `long` 对应 `IManagedHandle` 更是隐式约定：没有任何注释说明"这里的 `long` 就是 `IManagedHandle`"。

**建议**
1. 加静态断言把不变量钉住：
   ```cpp
   // IManagedHandle.h 末尾
   static_assert(sizeof(IManagedHandle) == 8, "IManagedHandle must stay a single 8-byte value for the C# nint ABI");
   static_assert(alignof(IManagedHandle) == 8, "IManagedHandle alignment changed; C# nint marshalling breaks");
   static_assert(std::is_trivially_copyable_v<IManagedHandle>);
   ```
2. C# 侧把 `long`/`nint` 换成 `Interop.IManagedHandle`（一个 `[StructLayout(LayoutKind.Sequential)] struct { public long Value; }`）只在需要表达"A 与 B 必须是同一种类型"的地方使用；或者至少在 `FieldBridge` 的两个方法上加 XML 注释说明与 `IManagedHandle` 的对应关系。
3. 在 `IScriptTypes.h` 顶部加一段注释，集中声明"A 区的 typedef 与 `Script/Interop/**` 一一对应，任何一侧改动必须同步另一侧"，并指明"参数个数由 `COMMON_BRIDGE_METHODS` 第三列维护"。

**验证方式**
- grep：`grep -rn "IManagedHandle " Source/UnrealCSharpCore/Public/Domain/Script/IScriptTypes.h`（列出按值返回的 typedef）与 `grep -rn "static nint\|static long" Script/Interop/`（列出 C# 侧对应）。
- 断言：加 `static_assert` 后若有人给结构体加成员，编译期即失败。

---

### [F-ABI-020] `UnrealCSharpCore.build.cs` 用 `GetFiles("*.Build.cs")` 枚举模块——`UnrealCSharpCore.build.cs`/`CrossVersion.build.cs` 是小写 `build.cs`，Linux/macOS 上匹配不到

- **类别**: 平台兼容 / 构建
- **严重度**: P2
- **复核结论**: 确认（`Suffix = "*.Build.cs"` 与 `DirectoryInfo.GetFiles(Suffix)` 逐字核对无误；大小写敏感文件系统上匹配不到 `*.build.cs` 的机制成立）
- **可达性**: 活跃（Build.cs 在三个平台的构建期都会执行）；**平台相关后果潜伏**（当前在 Windows/NTFS 上匹配正常，只有 Linux/macOS 构建才丢模块）
- **复核证据**: `Source/UnrealCSharpCore/UnrealCSharpCore.build.cs:329-347`（第一手读取：`:331 var Suffix = "*.Build.cs"`、`:335 GetFiles(Suffix)`、`:340-346` 还有一层 `SearchOption.AllDirectories` 的递归枚举——原文只引了 :329-338，实际函数到 :347）
- **函数**: `UnrealCSharpCore.GetModules(string, Dictionary<string,string>)`
- **置信度**: 中（"匹配不到"是 `DirectoryInfo.GetFiles` 在大小写敏感文件系统上的确定行为；"匹配不到会不会真的导致生成结果缺失"取决于消费端 `UnrealCSharp_Modules.json` 的处理，未追踪到消费点）

**现状（代码事实）**
```csharp
// Source/UnrealCSharpCore/UnrealCSharpCore.build.cs:329-338
	private void GetModules(string InPathName, Dictionary<string, string> Modules)
	{
		var Suffix = "*.Build.cs";

		var DirectoryInfo = new DirectoryInfo(InPathName);

		foreach (var Item in DirectoryInfo.GetFiles(Suffix))
		{
			Modules[Item.Name.Remove(Item.Name.Length - Suffix.Length + 1)] = Item.DirectoryName;
		}
```
实际文件名（`Get-ChildItem` 输出）：
- `Source/UnrealCSharp/UnrealCSharp.Build.cs`（大写 B）✔
- `Source/UnrealCSharpEditor/UnrealCSharpEditor.Build.cs`（大写 B）✔
- `Source/Compiler/Compiler.Build.cs`（大写 B）✔
- `Source/ScriptCodeGenerator/ScriptCodeGenerator.Build.cs`（大写 B）✔
- **`Source/UnrealCSharpCore/UnrealCSharpCore.build.cs`（小写 b）✘**
- **`Source/CrossVersion/CrossVersion.build.cs`（小写 b）✘**

**调用上下文**
`GeneratorModules()`(:120-201) 在 `Target.bBuildEditor` 时执行，用 `GetModules` 填 `ProjectPlugins`/`EngineModules`/`EnginePlugins` 三张表并写入 `Intermediate/UnrealCSharp_Modules.json`(:160-200)。`GetPlugins` 用的是 `*.uplugin`(:311)，大小写正确。

**问题**
`DirectoryInfo.GetFiles(pattern)` 的通配符匹配遵循**文件系统的大小写敏感性**。在 Windows（NTFS 默认不区分大小写）上 `*.Build.cs` 能匹配到 `UnrealCSharpCore.build.cs`；在 **Linux/macOS 上匹配不到**，于是这两个模块不会出现在生成的 `UnrealCSharp_Modules.json` 里。同一份 UBT 脚本在三个平台上产出**不同的** JSON——这是"只在 Windows 上验证过"的典型症状，而且失败是**静默**的（少两项，不报错）。

**建议**
```csharp
foreach (var Item in DirectoryInfo.GetFiles("*.cs")
                           .Where(F => F.Name.EndsWith(".Build.cs", StringComparison.OrdinalIgnoreCase)))
{
    Modules[Item.Name[..^9]] = Item.DirectoryName;      // 去掉 ".Build.cs"
}
```
或者统一把两个文件重命名为 `UnrealCSharpCore.Build.cs` / `CrossVersion.Build.cs`（UBT 本身接受任意大小写，见 F-ABI-021）。

**验证方式**
- pwsh：`Get-ChildItem Source -Recurse -Filter *.cs | Where Name -match '\.(Build|build)\.cs$' | Select Name`（列出大小写）。
- 用例：在 Linux 上跑一次 UBT 生成，`diff` `Intermediate/UnrealCSharp_Modules.json` 与 Windows 版本。

---

### [F-ABI-021] `.Build.cs` 文件名大小写不一致（3 种形态）——UBT 在大小写敏感文件系统上能否发现模块取决于实现细节

- **类别**: 可优化/可读性 / 平台兼容
- **严重度**: P3（**由 P2 下调**：UBT 自身大小写不敏感，漏模块风险不成立）
- **复核结论**: **部分确认/前提证伪**（"3 种命名形态"的代码事实成立：glob 实测 4 个大写 `*.Build.cs`（`UnrealCSharp/UnrealCSharp.Build.cs`、`UnrealCSharpEditor/UnrealCSharpEditor.Build.cs`、`Compiler/Compiler.Build.cs`、`ScriptCodeGenerator/ScriptCodeGenerator.Build.cs`）+ 2 个小写 `*.build.cs`（`UnrealCSharpCore/UnrealCSharpCore.build.cs`、`CrossVersion/CrossVersion.build.cs`）+ ThirdParty 三个大写；**但"Linux 上可能直接不被发现"被证伪**——UBT 的扩展名判定是**显式大小写不敏感**的）
- **可达性**: 活跃（代码/文件命名事实），但**后果不可达**（UBT 不会漏模块）
- **复核证据**: 引擎侧第一手：`Engine\Source\Programs\Shared\EpicGames.Core\FileSystemReference.cs:31`（`public static StringComparison Comparison { get; } = StringComparison.OrdinalIgnoreCase;`）、`:123-124`（`HasExtension` 用 `FullName.EndsWith(extension, Comparison)`）、判定点 `Programs\Shared\EpicGames.Build\System\Rules.cs:261-268`（`foreach (FileItem File in Directory.EnumerateFiles()) if (File.HasExtension(".build.cs"))`）；插件侧 glob 重跑见上
- **级别变动**: P2→P3（只剩"命名不统一、脚本/CI 易漏文件"的可读性代价；真正有风险的仍是 F-ABI-020 的**插件自己**的 `DirectoryInfo.GetFiles`）
- **函数**: （构建脚本）
- **置信度**: 中（UBT 的模块发现逻辑在引擎源码里，本机无引擎源码可查，未第一手确认其大小写处理）→ **已回引擎源码核对**：`Engine\Source\Programs\UnrealBuildTool\ProjectFiles\Eddie\EddieProject.cs:133` 用 `FileName.EndsWith(".build.cs") || FileName.EndsWith(".Build.cs")` **显式同时接受两种大小写**（说明 UBT 生态官方承认小写形态）；但 grep `\*\.Build\.cs` 在整个 `Programs/UnrealBuildTool/**/*.cs` 只有 4 处命中（其中唯一真正的文件系统枚举是 `ProjectFiles/Xcode/XcodeProject.File.cs:242` 的 `DirectoryReference.EnumerateFiles(SourceDir, "*.Build.cs")`），**未能定位模块规则发现的通配符调用点**，故"Linux 上是否真的漏模块"维持**未证实**（置信度：中）。原文"本机无引擎源码"的理由已删除（引擎源码本机存在，约 20989 个 `.cpp`）

**现状（代码事实）**
同一插件的 7 个模块里，`.Build.cs` 的文件名有 **3 种写法**：`*.Build.cs`（5 个，符合 UE 惯例）、`*.build.cs`（2 个）、以及 `Source/CrossVersion/CrossVersion.build.cs` 与 `UnrealCSharpCore.build.cs` 的类名/文件名均正确只是后缀小写。另外 `Source/ThirdParty/{CoreCLR/LeanCLR/Mono}/*.Build.cs` 都是大写。

**问题**
UE 的 UBT 在 `RulesAssembly` 扫描时对 `*.Build.cs` 的匹配是否大小写敏感，取决于它内部用的是 `Directory.EnumerateFiles`（跟随 FS）还是显式的大小写不敏感比较；在 Linux 上如果是前者，`UnrealCSharpCore` 与 `CrossVersion` 这两个模块**可能直接不被发现**（症状是"模块不存在/找不到 UnrealCSharpCore"）。即使 UBT 容忍了，工程里 3 种命名也让 `grep`/脚本/CI 容易漏文件（本报告 §0.1 的清单就为此多跑了一次 glob）。这是一个低成本、零风险的清理项。

**建议**
统一为 `*.Build.cs`（`git mv UnrealCSharpCore.build.cs UnrealCSharpCore.Build.cs`、`git mv CrossVersion.build.cs CrossVersion.Build.cs`），并把类名与文件名一致（当前 `CrossVersion.build.cs` 里 `public class CrossVersion : ModuleRules` 是匹配的）。同时把 `UnrealCSharpCore.build.cs:331` 的通配符一并改掉（见 F-ABI-020）。

**验证方式**
- pwsh：`Get-ChildItem Source -Recurse -Filter *.cs | Where { $_.Name -cmatch '\.build\.cs$' }`（`-cmatch` 大小写敏感，列出小写后缀的那两个）。
- 用例：在 Linux 上编译插件，确认 `CrossVersion` 模块被 UBT 发现。

---

### [F-ABI-022] 源文件含非 ASCII（中文）注释且无 BOM，工程未加 `/utf-8` —— **撤销（非缺陷）**：UE 5.6 的 MSVC 工具链无条件传 `/utf-8` 并禁用 C4819

- **类别**: 平台兼容
- **严重度**: P3（只有 2 个文件、且全部在注释里，不涉及字符串字面量；因此影响限于 MSVC 的 C4819 类警告与"注释吞掉代码"的极端情况）
- **复核结论**: **证伪（撤销，非缺陷）** —— 前提"MSVC 会按系统 ANSI 代码页解释这些源文件"在 UE 5.6 上不成立：UBT 的 MSVC 工具链**无条件**添加 `/utf-8`，并紧接着 `/wd4819` 关闭该警告
- **可达性**: 不可达（在 UE 5.6 的 MSVC 构建里不可能出现"无 BOM 的 UTF-8 源文件被按 ANSI 解码"）
- **复核证据**: `Engine\Source\Programs\UnrealBuildTool\Platform\Windows\VCToolChain.cs:649-653`（第一手读取：`:649 // Fix Incredibuild errors with helpers using heterogeneous character sets` → `:650 Arguments.Add("/utf-8");` → `:652-653 // Disable "The file contains a character that cannot be represented in the current code page" warning for non-US windows.` → `Arguments.Add("/wd4819");`）。插件侧事实本身仍成立（grep 重跑：`Source/**/*.cs` 中 `utf-8|utf8|AdditionalOptions` **0 命中**；`NameEncode.h:27-34` 确有 8 行中文注释）——但**结论不成立**
- **级别变动**: P3 → **撤销（非缺陷）**（引擎工具链已保证 `/utf-8`；BOM 与"注释吞代码"风险随之消失）。**保留建议**：`FFileHelper::SaveStringToFile(..., ForceUTF8WithoutBOM)` 的统一编码约定仍值得写明（可读性，不入缺陷清单）
- **说明**: 其余工具链（clang-cl / Clang / 各平台 clang）未逐一核对 `/utf-8` 等价选项；若后续切到非 MSVC 编译器，请复查该前提
- **文件**: `Source/UnrealCSharpCore/Public/Common/NameEncode.h:27-34`、`Source/UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp:165-236`
- **函数**: （源文件编码）
- **置信度**: 中（未能确认 UE 5.6 的 UBT 是否为 MSVC 自动添加 `/utf-8`；本机无引擎源码）

**现状（代码事实）**
用 `Select-String -Pattern '[^\x00-\x7F]'` 扫 `Source/**/*.{cpp,h,inl}`（排除 ThirdParty/Intermediate/Binaries）的结果：**只有 2 个文件**，共 16 行：
- `Source/UnrealCSharpCore/Public/Common/NameEncode.h`：行 27,28,29,30,31,32,33,34
- `Source/UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp`：行 165,166,167,173,202,217,222,236

用 `[System.IO.File]::ReadAllBytes(...)[0..2]` 检查 BOM：
- `NameEncode.h` → `35,112,114`（`#pr`，**无 BOM**）
- `UnrealCSharpEditorSetting.cpp` → `35,105,110`（`#in`，**无 BOM**）
- `MetaDataAttributeMacro.h` → `35,112,114`（无 BOM，作为对照）

在 `Source/*/*.cs`（Build.cs）里 grep `utf-8|utf8|AdditionalOptions` → **0 命中**，即没有任何模块显式开启 `/utf-8`。

**问题**
MSVC 在"无 BOM + 未指定 `/utf-8`"时按**系统 ANSI 代码页**解释源文件。中文 Windows（GBK/936）下，UTF-8 的中文字节序列会被解码成不同的 GBK 字符；注释内容错乱本身无害，但经典故障是**多字节序列的尾字节吞掉后面的 `\` 或 `*`**，导致注释提前结束或把下一行代码吞进注释。当前所有非 ASCII 都位于 `/* ... */` 块注释（`NameEncode.h:26-35`）与 `//` 行注释（`UnrealCSharpEditorSetting.cpp:165-236`，每行一个 `//`），风险较低；此外 MSVC 会给出 C4819 警告。
另外**没有**任何 C++ 字符串字面量包含非 ASCII（`Select-String` 全量扫描已确认），所以"中文字符串被 MSVC 解码成乱码"这一更严重的问题**不存在**。

C# 侧：`Script/**` 里含中文的 `.cs` 由 C# 编译器处理（`Encoding.UTF8` 默认 + `UTF8Encoding` 检测 BOM），且 `UnrealTypeSourceGenerator`/`SourceCodeGenerator` 写文件时的编码未在本报告范围内核对（`grep Encoding.UTF8` 在 `Script/` 下只命中 `TypeBridge.cs`/`LogBridge.cs` 的运行时用途，未命中生成器的文件写入路径）。

**建议**
- 给这两个文件加 UTF-8 BOM（一次性、最低风险），或在 `UnrealCSharpCore.build.cs` 里显式加：
  ```csharp
  if (Target.WindowsPlatform.Compiler.IsMSVC())
  {
      PrivateDefinitions.Add("...");   // 视需要
      // 或 bEnableUndefinedIdentifierWarnings 旁边集中配置：
      // AdditionalOptions 由 UBT 管理，推荐做法是用 BOM
  }
  ```
  **推荐 BOM**：UE 官方代码库普遍使用 UTF-8 BOM，且不受 UBT 版本差异影响。
- 长期：要么把注释改成英文，要么在工程级 `Config/DefaultBuildSettings` 里统一开启 `/utf-8`。

**生成物侧的同源证据**
`FUnrealCSharpFunctionLibrary::SaveStringToFile` 是 `ScriptCodeGenerator` 所有产物的**唯一落盘通道**（被 `FClassGenerator.cpp:871`、`FBindingClassGenerator.cpp:739/1011`、`FStructGenerator.cpp:350`、`FEnumGenerator.cpp:120/247`、`FDelegateGenerator.cpp:391/718`、`FGameplayTagGenerator.cpp:94`、`FAssetGenerator.cpp:191`、`FBindingEnumGenerator.cpp:65`、`FSolutionGenerator.cpp:162` 调用）以及动态新建类文本（`UnrealCSharpEditor/Private/NewClass/SDynamicNewClassDialog.cpp:777`）的写入点：
```cpp
// Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1177-1178
	return FFileHelper::SaveStringToFile(InString, *InFileName, FFileHelper::EEncodingOptions::ForceUTF8WithoutBOM,
	                                     FileManager, FILEWRITE_None);
```
即**一律写成无 BOM 的 UTF-8**。当前生成物里没有含非 ASCII 的 `.h/.inl`（未复现，依据全仓扫描结果），因此这是**潜伏**风险：一旦某个工程模块的 `UPROPERTY(meta=(ToolTip="中文"))` 或 Doxygen 注释被生成进 `.h/.inl` 并交给 MSVC 编译，就会重现 C4819 与"注释吞代码"的同一故障。建议把 `ForceUTF8WithoutBOM` 改为 `ForceUTF8`（带 BOM），与 F-ABI-022 的源文件修复方向一致。

**验证方式**
- pwsh（本报告使用的命令）：
  ```powershell
  Get-ChildItem Source -Recurse -File -Include *.cpp,*.h,*.inl |
    Where FullName -notmatch 'ThirdParty|Intermediate|Binaries' |
    ForEach { $m = Select-String -Path $_.FullName -Pattern '[^\x00-\x7F]'; if ($m) { "$($_.Name): $($m.Count)" } }
  ```
- 断言：MSVC 编译这两个文件时不应出现 C4819。

---

### [F-ABI-023] `TFunctionPointer` 用 `union { T Function; void* Pointer; }` 做函数指针 ↔ 对象指针转换

- **类别**: 未定义行为（可移植性）
- **严重度**: P3
- **复核结论**: 确认（union 双关机制与"写入 `Function` 后读 `Pointer` 属 active-member UB"的判断成立）
- **可达性**: 活跃（`BindingMacro.h:262/272/288/298/308/310` 等 6 个宏在**每一个**绑定注册处读 `.Value.Pointer`）；但在 POSIX/MSVC 的实际实现下不会失效，故后果可忽略
- **复核证据**: `Source/UnrealCSharpCore/Public/Template/TFunctionPointer.inl:3-17`（第一手读取：`:11-16 union { T Function; void* Pointer; }`，构造函数 `:6-9` 只写 `Value.Function`）
- **级别变动**: 无（P3 维持）
- **交叉引用**: 与 `02-UnrealCSharpCore核心/07-函数库宏与模板Trait.md` 的 **F-FL-016 同源**（同一 `TFunctionPointer.inl:3-17`、同一 union 双关机制）——**聚合统计时只计一次**。注意两份报告级别不一致：F-FL-016 在 02-…/07 中为 **P2**，本条为 **P3**；本报告认为"实际目标平台均支持（POSIX 保证往返、MSVC 允许），不构成现实缺陷"，故取 P3，建议统一取 P3（如另一侧有更强证据可反向覆盖）
- **函数**: `TFunctionPointer<T>::TFunctionPointer(const T&)`
- **置信度**: 高（C++ 标准层面是"实现定义/条件支持"；实际项目平台均可用）

**现状（代码事实）**
```cpp
// Source/UnrealCSharpCore/Public/Template/TFunctionPointer.inl:3-17
template <typename T>
struct TFunctionPointer
{
	explicit TFunctionPointer(const T& InFunction)
	{
		Value.Function = InFunction;
	}

	union
	{
		T Function;

		void* Pointer;
	} Value;
};
```

**调用上下文**
被绑定构建器用来把成员函数指针/自由函数指针转成 `const void*`（例如 `TClassBuilder.inl:91-93 Subscript` 的 `TFunctionPointer<decltype(InGetMethod)>(InGetMethod)`，以及 `TBindingClassBuilder` 的各 `Function`/`Property` 宏）；最终地址经 `FBindingMethod::Function`(`FBindingMethod.h:26`) 与 `FScriptDomainImpl.inl:627 Methods.Add(reinterpret_cast<PTRINT>(const_cast<void*>(Method.GetFunction())))` 送到托管侧。

**问题**
C++ 标准里"函数指针 ↔ `void*`"是**条件支持**（`reinterpret_cast` 的转换函数指针到对象指针是实现定义的；POSIX 明确要求可往返，MSVC 也允许）。所有目标平台（Win64/Linux/macOS/Android/iOS）都在实践中支持，因此**不构成本项目的现实缺陷**。但值得在头文件里加一行注释说明"这是刻意的实现定义行为，依赖 POSIX/MSVC 的保证"，避免后人以为可以安全地把 `Pointer` 用于与地址比较之外的目的。另外注意 union 的active-member 规则：写入 `Function` 后读 `Pointer` 在 C++ 里是 UB（虽然实践中安全），更稳妥的写法是 `reinterpret_cast<void*>` 或 `memcpy`。

**建议**
```cpp
	explicit TFunctionPointer(const T& InFunction) : Value(InFunction) {}

	static_assert(sizeof(T) == sizeof(void*), "function pointer size mismatch");
	...
	// 读取端：reinterpret_cast<const void*>(InFunction) 或 FMemory::Memcpy(&Out, &InFunction, sizeof(Out))
```
或直接 `static_assert(sizeof(T) == sizeof(void*))` 把"在同一平台上等价"这件事钉在编译期。

**验证方式**
- grep：`grep -rn "TFunctionPointer" Source/`（定义 1 处、使用点在 `TClassBuilder/TBindingClassBuilder/TFunctionBuilder` 等 .inl）。
- 断言：加 `static_assert(sizeof(T) == sizeof(void*))`。

---

### [F-ABI-024] 信号处理器调试信息对 `SIGINT/SIGTERM/SIGABRT` 也生效——Ctrl-C/正常退出会打印一次托管堆栈

- **类别**: 可优化/可读性（行为一致性）
- **严重度**: P3
- **复核结论**: 确认（`SIGINT`/`SIGTERM`/`SIGABRT`/`SIGBREAK` 确实与故障信号同列一张表，逐字核对无误）
- **可达性**: 活跃（Windows 上 Ctrl-C/Ctrl-Break、Linux 上 `kill`/容器停止都会命中；`SignalHandler` 会进托管运行时并 `GLog->Flush()`）
- **复核证据**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:101-118`（第一手读取：`:103 SIGINT`、`:111 SIGTERM`、`:117 SIGABRT`、`:112-115 SIGBREAK`(Windows)）
- **级别变动**: 无（P3 维持：后果是"退出变慢/噪音日志"，不是功能错误）
- **函数**: `FCSharpEnvironment::Initialize()`
- **置信度**: 高（`SIGINT`/`SIGTERM`/`SIGABRT` 确实在 `SignalTypes` 里）

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:101-118
	static TSet<int32> SignalTypes = {
		// interrupt
		SIGINT,
		// illegal instruction - invalid function image
		SIGILL,
		// floating point exception
		SIGFPE,
		// segment violation
		SIGSEGV,
		// Software termination signal from kill
		SIGTERM,
#if PLATFORM_WINDOWS
		// Ctrl-Break sequence
		SIGBREAK,
#endif
		// abnormal termination triggered by abort call
		SIGABRT,
	};
```

**问题**
把 `SIGINT`（Ctrl-C）与 `SIGTERM`（`kill`）纳入"打印托管堆栈"的处理器是**语义错配**：这两个信号是**正常的进程控制流**，不是故障。在 Linux 上 `systemd`/容器停止时发 `SIGTERM` → 插件 `SignalHandler` 试图进入托管运行时（此时 Mono/CoreCLR 可能已在关闭中）→ 打印一段无意义的堆栈甚至挂住退出流程，拖慢停机。同时这会**替换掉 UE/系统原本的 SIGINT 处置**（见 F-ABI-009 的平台分支差异），使 Ctrl-C 的默认行为（终止进程）依赖 `signal()` 的语义细节。

**建议**
只对真正的故障信号安装处理器：`SIGSEGV / SIGFPE / SIGILL / SIGABRT`；把 `SIGINT/SIGTERM/SIGBREAK` 从表里去掉，或在处理器里对它们**只恢复默认并 return**（不打印）：
```cpp
	if (Signal == SIGINT || Signal == SIGTERM) { signal(Signal, SIG_DFL); raise(Signal); return; }
```

**验证方式**
- grep：`grep -n "SIGINT\|SIGTERM\|SIGBREAK" Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp`（:103 / :111 / :114）。
- 用例：Linux 下运行打包版本，`kill -TERM`，观察日志里是否出现一段托管堆栈。

---

### [F-ABI-035] `FFunctionParamPoolBufferAllocator` 的池计数器是 `uint8`（256 回绕），且 `Malloc`/`Free` 无同步

- **类别**: Bug / 并发
- **严重度**: P2
- **复核结论**: 确认（池计数器确为 `uint8 Count;`，位于 :34，逐字核对无误）
- **可达性**: 活跃（`FFunctionParamPoolBufferAllocator` 由 `FFunctionParamBufferAllocatorFactory::Factory`(`:57-71`) 在 `ParmsSize > 0` 时构造，即所有带参数的 UE 函数调用）
- **复核证据**: `Source/UnrealCSharp/Public/Reflection/Function/FFunctionParamBufferAllocator.h:21-39`（第一手读取：`:34 uint8 Count;`、`:36 decltype(UFunction::ParmsSize) ParamSize;`、`:38 TArray<void*> Buffers;`）、`:57-71`（工厂：`InFunction->ParmsSize > 0` 才用池实现）
- **级别变动**: 无（P2 维持）
- **函数**: `FFunctionParamPoolBufferAllocator::Malloc()` / `FFunctionParamPoolBufferAllocator::Free(void*)`
- **置信度**: 中（`Count` 的类型为第一手读取；回绕后的具体数据损坏路径未证明，`.cpp` 的增减逻辑未逐行复核）

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Public/Reflection/Function/FFunctionParamBufferAllocator.h:21-39
class FFunctionParamPoolBufferAllocator final : public FFunctionParamBufferAllocator
{
public:
	explicit FFunctionParamPoolBufferAllocator(const TWeakObjectPtr<UFunction>& InFunction);

	virtual ~FFunctionParamPoolBufferAllocator() override;

public:
	virtual void* Malloc() override;

	virtual void Free(void* InMemory) override;

private:
	uint8 Count;                                    // ← 8 位计数器

	decltype(UFunction::ParmsSize) ParamSize;

	TArray<void*> Buffers;
};
```

**调用上下文**
`FFunctionParamBufferAllocatorFactory::Factory`(`:57-71`) 在 `UFunction::ParmsSize > 0` 时构造它；使用点是 `FUnrealFunctionDescriptor::CallN`（`FUnrealFunctionDescriptor.inl` 的 `BufferAllocator->Malloc()/Free()`）。一台服务器/编辑器里同一 `UFunction` 的分配器是**长生命周期的共享对象**（通过 `FUnrealFunctionDescriptor` 持有），会被多线程（`AsyncTask`/`ParallelFor` 触发的 C#→UE 调用）并发访问。

**问题**
1. **计数器回绕**：`Count` 是 `uint8`，池的"在用以外的空闲块数"超过 255 后回绕 → 之后 `Malloc` 会重新发放**从未归还**的缓冲（同一块内存被两个并发调用同时使用）→ 参数互相覆盖。与 F-ABI-031 的"某些 `CallN` 不 `Free`"叠加时，这个回绕点会被**更快**触达（泄漏的块永远不回池，`Count` 反而因为缺少 `Free` 而行为更不可预测）。
2. **无同步**：`Buffers`（`TArray<void*>`）的 `Add`/`Remove` 与 `Count` 的自增自减没有任何临界区保护，而池是跨线程共享的 → 数据竞争（`TArray` 重分配与遍历并发 ⇒ 堆破坏）。
3. 回绕点让"泄漏"表现为"在第 256 次调用后突然不再增长"，会误导内存分析。

**建议**
- `uint8 Count` → `int32 Count`（或直接去掉计数器用 `Buffers.Num()`）；
- 用 `FCriticalSection` 或 `TQueue`（无锁）保护池；
- 每块缓冲记录所属的 `UFunction` 与分配序号，在 `ProcessEvent` 之后校验"每个 `Malloc` 都有配对的 `Free`"（`check` 或 `ensure`），把 F-ABI-031 这类问题变成开发期立刻可见的断言。

**验证方式**
- grep：`grep -n "uint8 Count\|Count++\|Count--" Source/UnrealCSharp -r`（确认类型与无锁增减）。
- 用例：对同一个 `UFunction` 连续做 300 次"不归还缓冲"的调用（当前 `Call2` 天然满足），观察第 256 次后是否出现参数串扰。

---

### [F-ABI-036] 生成的 `.csproj` / `.sln` 在所有平台写死 Windows 反斜杠路径

- **类别**: 平台兼容
- **严重度**: P2
- **复核结论**: 确认（生成的 `.csproj` 片段里确实写死 Windows 反斜杠相对路径，逐字核对无误）
- **可达性**: 活跃（`FSolutionGenerator` 在编辑器里生成用户工程解决方案时必然执行）；在 Linux/macOS 编辑器上生成的路径分隔符与 MSBuild 的期望不一致 → 后果活跃
- **复核证据**: `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:226-256`（第一手读取：`:230 <ScriptOutputPath>..\\..\\Content\\%s</ScriptOutputPath>`、`:240 <OutputPath>..\\..\\Content\\%s</OutputPath>`、`:250 <HintPath>..\\..\\Content\\%s\\%s%s</HintPath>`）
- **级别变动**: 无（P2 维持）
- **函数**: `FSolutionGenerator::ReplaceScriptOutputPath` / `ReplaceOutputPath` / `ReplaceHintPath` / `ReplaceProjectReference` / `ReplaceProject` / `ReplaceProjectPlaceholder`
- **置信度**: 高（生成侧与**落盘产物**均已第一手核对）；中（`dotnet build` 在 Linux 上是否因此失败未实测）

**现状（代码事实）**
```cpp
// Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:226-234
void FSolutionGenerator::ReplaceScriptOutputPath(FString& OutResult)
{
	OutResult = OutResult.Replace(TEXT("<ScriptOutputPath></ScriptOutputPath>"),
	                              *FString::Printf(TEXT(
		                              "<ScriptOutputPath>..\\..\\Content\\%s</ScriptOutputPath>"
	                              ),
```
同族的 `ReplaceOutputPath`(:240 `<OutputPath>..\..\Content\%s</OutputPath>`)、`ReplaceHintPath`(:250 `<HintPath>..\..\Content\%s\%s%s</HintPath>`)、`ReplaceProjectReference`(:274 `<ProjectReference Include="..\%s\%s%s" />`)、`ReplaceProject`(:318/:330 的 `.sln` 工程路径) 全部硬编码 `\\`。整个 `Source/ScriptCodeGenerator` 目录里只有 `FCodeAnalysis.cpp:19`、`:61` 两处平台宏，`FSolutionGenerator.cpp` **无任何平台分支**。

**落盘产物（已生成到工程里，第一手读取）**：
- `Script/Interop/Interop.csproj:12` `<Compile Include="..\..\Plugins\UnrealCSharp\Script\Interop\**" ...>`
- `Script/Shared.props:23` `<HintPath>..\..\Content\Script\Interop.dll</HintPath>`
- `Script/UE/UE.csproj:12` `<ProjectReference Include="..\SourceGenerator\SourceGenerator.csproj" ...>`
- `Script/Script.sln:7` `Project("{9A19103F-…}") = "UE", "UE\UE.csproj", "{7AF881DC-…}"`

**问题**
Linux/macOS 上 `\` 不是目录分隔符。风险最高的是 `<OutputPath>` / `<ScriptOutputPath>`：它们是 **MSBuild 属性**（不是 item），属性值**不会**被 MSBuild 的 `FixFilePath` 规范化，于是 `..\..\Content\Script` 会被当成一个**字面含反斜杠的目录名**，`UE.dll`/`Game.dll` 落到错误位置，随后 `FUnrealCSharpFunctionLibrary::GetFullUEPublishPath()`(`:1039-1042`) 在 `Content/Script/` 下找不到程序集 → 脚本加载失败。`<Compile Include>`/`<ProjectReference Include>`/`.sln` 的工程路径相对缓和（MSBuild 对 item spec 在 Unix 上会做 `\`→`/` 规范化），这也是本条定 P2 而非 P1 的原因。

**建议**
在生成侧用 `FPaths::Combine` 拼路径后统一把 `\\` 替换为 `/`（`.csproj`/`.sln` 在所有平台都接受 `/`），或按 `PLATFORM_WINDOWS` 给出两套模板；同时给 `Script/Shared.props` 的 `HintPath` 与 `GetFullInteropPublishPath()`（F-ABI-012 第 3 点）统一到同一个"发布根目录"概念上。

**验证方式**
- grep：`grep -rn '\\\\' Source/ScriptCodeGenerator/` 与 `grep -rn '\\\\' Script/*.props Script/*/*.csproj Script/*.sln`。
- 用例：Linux 上执行 `dotnet build Script/Script.sln`，检查是否出现名字含 `\` 的目录、以及 `Content/Script/UE.dll` 是否生成。

---

### [F-ABI-037] `TypeBridge.GetTypeImplementation` 的 `Type.GetType(..., throwOnError: false)` 仍可能抛 `ArgumentException`

- **类别**: Bug
- **严重度**: P2
- **复核结论**: 确认（`Type.GetType(InFullName, throwOnError: false, ignoreCase: false)` 确实可能对**非法类型名**抛 `ArgumentException`——`throwOnError:false` 只抑制 `TypeLoadException` 类失败，不抑制参数校验异常；且本方法无 try/catch，LeanCLR 下异常被记录后返回 0）
- **可达性**: 活跃（类型名来自 C# 侧 `Utils`/生成代码给出的 `FullName`，经 `TypeBridge.GetClass/GetType` 进入本方法）
- **复核证据**: `Script/Interop/Bridge/TypeBridge.cs:446-480`（第一手读取：`:459` 为调用点；`:477-480` 的兜底扫描也以 `FullName` 的分段为输入）
- **级别变动**: 无（P2 维持）
- **函数**: `Interop.TypeBridge.GetTypeImplementation(string)`
- **置信度**: 中（`throwOnError: false` 只抑制"找不到类型"，不抑制"类型名语法非法"是 BCL 的文档化行为；具体可达输入未构造）

**现状（代码事实）**
```csharp
// Script/Interop/Bridge/TypeBridge.cs:453-475
        var Index = InFullName.IndexOf(',');

        var TypeName = Index >= 0 ? InFullName[..Index].Trim() : InFullName;

        var AssemblyName = Index >= 0 ? InFullName[(Index + 1)..].Trim() : null;

        var Type = System.Type.GetType(InFullName, throwOnError: false, ignoreCase: false);   // :459

        if (Type == null)
        {
            var Assemblies = AssemblyLoader.CurrentContext?.Assemblies
                             ?? AppDomain.CurrentDomain.GetAssemblies();

            Type = GetTypeImplementation(Assemblies, AssemblyName, TypeName);
        }
```
`GetTypeImplementation` 被 `GetClass`(`:23`)、`ArrayBridge.NewArray`(`ArrayBridge.cs:15`) 调用，二者都是 `[UnmanagedCallersOnly]` 且无 try/catch（`GetClass` 在 `TypeBridge.cs:14-33`）。

**问题**
`Type.GetType(string, bool throwOnError, bool ignoreCase)`：`throwOnError: false` 只把"**找不到**"变成返回 null，**语法非法**的类型名（括号不配对、含 `|`/`&`/`*` 的非法组合、空名字、`[` 未闭合等）仍然抛 `ArgumentException`（名字为 null 时抛 `ArgumentNullException`）。`InFullName` 来自 C++ 侧的 `COMBINE_FULL_NAME(InNamespace, InName)`（`FScriptDomainImpl.inl:371-377` 的 `GetClass`，以及 `SCRIPT_DOMAIN_STRING_CAST`），其内容可源自 UE 反射元数据（类名/命名空间里出现 `[`、`]`、`,` 等字符是可能的，例如 `TArray` 的泛型拼接 `COMBINE_GENERIC` 用的是反引号与 `[]`）。异常穿出 `[UnmanagedCallersOnly]` → **进程终止**。
可达性未构造，故按 P2 计；修复成本极低。

**建议**
包一层 try/catch 并在名字非法时返回 0（与 F-ABI-001/F-ABI-003 同一批修复）：
```csharp
    Type? Type;
    try { Type = System.Type.GetType(InFullName, throwOnError: false, ignoreCase: false); }
    catch (ArgumentException) { return null; }
```
或先用 `TypeName` 做一次字符白名单校验，再调用 `GetType`。

**验证方式**
- grep：`grep -n "Type.GetType" Script/`（唯一一处 `:459`）。
- 用例：让 C++ 侧传入含未配对 `[` 的类名（例如构造一个 `MakeGenericType` 失败后的名字），观察进程是否消失。

---

### P3

### [F-ABI-025] 桥接函数指针解析失败时不校验、不报错（Mono/LeanCLR），与 CoreCLR 的写法不一致

- **类别**: Bug（健壮性）/ 一致性
- **严重度**: P3（当前所有 `SCRIPT_DOMAIN_INVOKE` 调用点都有 `!= nullptr` 守卫，见下；因此不会立刻崩溃）
- **复核结论**: 确认（Mono 侧 `GET_METHOD_AND_GET_FUNCTION_POINTER` 宏在 `Method == nullptr` 时**不做任何事**——既不清零指针、也不报错、也无 else 分支，逐字核对无误）
- **可达性**: 潜伏（`FMonoDomain.cpp` 在 `#if WITH_MONO` 内、`FLeanCLRDomain.cpp` 的对应段在 `#if WITH_LEANCLR` 内——**LeanCLR 那段当前活跃**，但其"解析失败不报错"的后果被 `SCRIPT_DOMAIN_INVOKE` 的 `!= nullptr` 守卫挡住）
- **复核证据**: `Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:34-39`（第一手读取：`if (const auto Method = Class_Get_Method_From_Name(...); Method != nullptr) { InFn = reinterpret_cast<…>(Method_Get_Unmanaged_Callers_Only_Ftnptr(Method)); }`，无 else）；`FLeanCLRDomain.cpp:841-845` 与 `FCoreCLRDomain.cpp:21-28` 两处对照**未第一手重读**（卡点：步数预算；原文引用保持原样）
- **级别变动**: 无（P3 维持）
- **函数**: `GET_METHOD_AND_GET_FUNCTION_POINTER`（宏）、`RESOLVE_LEANCLR_METHOD`（宏）、`LOAD_ASSEMBLY_AND_GET_FUNCTION_POINTER`（宏）
- **置信度**: 高

**现状（代码事实）**
```cpp
// Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:34-39
#define GET_METHOD_AND_GET_FUNCTION_POINTER(InFn, InManagedClass, InName, InParamCount) \
	if (const auto Method = Class_Get_Method_From_Name(InManagedClass, InName, InParamCount); \
		Method != nullptr) \
	{ \
		InFn = reinterpret_cast<decltype(InFn)>(Method_Get_Unmanaged_Callers_Only_Ftnptr(Method)); \
	}
```
```cpp
// Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRDomain.cpp:21-28（唯一带检查的）
#define LOAD_ASSEMBLY_AND_GET_FUNCTION_POINTER(InAssembly, InType, InMethod, InFn) \
{ \
	void* OutFn{}; \
	if (LoadAssemblyAndGetFunctionPointer(InAssembly, InType, InMethod, &OutFn); OutFn != nullptr) \
	{ \
		InFn = reinterpret_cast<decltype(InFn)>(OutFn); \
	} \
}
```
```cpp
// Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp:841-845
#define RESOLVE_LEANCLR_METHOD(Name, FnType, ClassNameMacro, MethodNameMacro, ParamCount) \
	{ \
		const auto FullName = COMBINE_FULL_NAME(NAMESPACE_INTEROP, ClassNameMacro); \
		Name##Fn = Class_Get_Method_From_Name(InteropModule, FullName, MethodNameMacro); \
	}
```

**问题**
1. Mono 侧：`Method_Get_Unmanaged_Callers_Only_Ftnptr`(`FMonoDomain.cpp:253-260`) 的返回值**没有检查**（`mono_method_get_unmanaged_callers_only_ftnptr` 在方法未标 `[UnmanagedCallersOnly]` 或出错时返回 `nullptr`），并且 `MonoError`(:255-259) 里的错误码**从不读取**——一旦 C# 侧误删某个 `[UnmanagedCallersOnly]`，这里会**静默**把函数指针赋成 `nullptr`，直到某次 `SCRIPT_DOMAIN_INVOKE` 才会暴露（当前所有调用点都有 `Fn != nullptr` 守卫，所以表现为**功能静默失效**而不是崩溃）。
2. LeanCLR 侧：`Class_Get_Method_From_Name` 返回 `nullptr` 时不报错，`Bridge_Invoke` 里 `Runtime_Invoke(nullptr, ...)` 返回 false（`FLeanCLRDomain.cpp:279` 有 `!= nullptr` 检查）→ 与 `F-ABI-015` 叠加变成"零值"。同样是静默失效。
3. CoreCLR 侧做了正确的事（`OutFn != nullptr` 才赋值 + `hostfxr` 返回码检查 `FCoreCLRDomain.cpp:90-104`）——三个后端行为不一致。

**建议**
统一为一个"解析失败即 `UE_LOG(Error)` + `ensure`"的模板（可放在 `IScriptTypes.h` 的宏里）：
```cpp
#define BRIDGE_RESOLVE_OR_WARN(InFn, Expr) \
{ \
    auto Resolved = (Expr); \
    if (Resolved == nullptr) { UE_LOG(LogUnrealCSharp, Error, TEXT("Bridge entry not resolved: %s"), TEXT(#InFn)); ensure(Resolved != nullptr); } \
    else { InFn = Resolved; } \
}
```
并把 `MonoError` 的结果也读出来（`mono_error_ok(&Error)` 或 `Error.error_code != MONO_ERROR_NONE`；`MonoError` 是 union，`Error.init = 0` 初始化是 mono 官方设计的正确用法，见 `ThirdParty/Mono/src/mono/utils/details/mono-error-types.h:58-67` 的注释 —— 这一点作者做对了）。

**验证方式**
- grep：`grep -n "OutFn != nullptr\|Method != nullptr" Source/UnrealCSharpCore/Private/Domain/{Mono,CoreCLR,LeanCLR}/*.cpp`（只有 CoreCLR 同时检查了返回码与指针）。
- 用例：注释掉 `Script/Interop/Bridge/StringBridge.cs:8` 的 `[UnmanagedCallersOnly]`，观察是"启动时报错"还是"字符串桥接静默失效"。

---

### [F-ABI-026] `TypeBridge.GetTypeImplementation` 对同一 `FullName` 只缓存成功结果、对失败结果每次都全量扫装配件（性能隐患）

- **类别**: 性能
- **严重度**: P3
- **复核结论**: 确认（`StringToType.TryAdd` 只在成功分支调用、失败时每次都走 `GetTypeImplementation(Assemblies, …)` 全量扫装配件，逐字核对无误）
- **可达性**: 活跃（`GetTypeImplementation` 是 `TypeBridge.GetClass/GetType` 的实现；**失败路径**只在类型名解析失败时才反复触发）
- **复核证据**: `Script/Interop/Bridge/TypeBridge.cs:446-480`（第一手读取：`:448-451` 命中缓存即返回；`:459 System.Type.GetType(InFullName, throwOnError: false, ignoreCase: false)`；`:461-467` 失败才扫装配件；`:469-472` **仅成功时** `StringToType.TryAdd`）
- **级别变动**: 无（P3 维持；这是"失败路径无负缓存"的性能隐患，不是功能错误）
- **函数**: `Interop.TypeBridge.GetTypeImplementation(string)`
- **置信度**: 中（缓存成功的分支是清晰的；"失败结果不被缓存"是代码事实，但调用频次未测量）

**现状（代码事实）**
```csharp
// Script/Interop/Bridge/TypeBridge.cs:446-475
    internal static Type? GetTypeImplementation(string InFullName)
    {
        if (StringToType.TryGetValue(InFullName, out var OutType))
        {
            return OutType;
        }

        var Index = InFullName.IndexOf(',');
        ...
        var Type = System.Type.GetType(InFullName, throwOnError: false, ignoreCase: false);

        if (Type == null)
        {
            var Assemblies = AssemblyLoader.CurrentContext?.Assemblies
                             ?? AppDomain.CurrentDomain.GetAssemblies();

            Type = GetTypeImplementation(Assemblies, AssemblyName, TypeName);
        }

        if (Type != null)
        {
            StringToType.TryAdd(InFullName, Type);
        }

        return Type;
    }
```
```csharp
// Script/Interop/Bridge/TypeBridge.cs:477-499
    private static Type? GetTypeImplementation(IEnumerable<Assembly> InAssemblies, string? InAssemblyName,
        string InTypeName)
    {
        foreach (var Assembly in InAssemblies)
        {
            if (InAssemblyName != null)
            {
                if (!string.Equals(Assembly.GetName().Name, InAssemblyName, StringComparison.OrdinalIgnoreCase))
                {
                    continue;
                }
            }

            var Type = Assembly.GetType(InTypeName, throwOnError: false, ignoreCase: false);
            ...
```

**问题**
`StringToType.TryAdd` 只在 `Type != null` 时执行 → **"查不到"这个结果永远不缓存**。而 `GetClass`(`TypeBridge.cs:15-33`) 是 C++ 侧每次解引用类型都会走的入口（`FScriptDomainImpl.inl:371-377` → `FReflectionRegistry::Get().GetClass(...)`，见 `FReflectionRegistry.inl` 的注册表查询路径）。若某个类名被反复查询但确实不存在（例如绑定生成与运行时程序集版本不匹配、或者某个 `TSubclassOf` 的泛型实参解析失败），每次都会：`Type.GetType`（可能抛内部异常后再吞掉）+ **遍历全部已加载装配件并逐个 `Assembly.GetType`**。这是"命中失败"路径上的 O(装配件数 × 类型查找) 热开销。
注意 `GetClass` 反方向（`:35-46 GetType`）会把 `Object.GetType()` 的结果 `HandleData.Alloc` 起来，那是句柄泄漏的另一条线（不在本报告范围）。

**建议**
```csharp
        // 缓存失败结果（用 null 占位需要可空字典或单独的 HashSet<string> NegativeCache）
        if (Type == null) { NegativeTypes.Add(InFullName); return null; }
        ...
        if (NegativeTypes.Contains(InFullName)) return null;
```
并在 `Clear()`(`TypeBridge.cs:501-504`) 里一起清空 `NegativeTypes`（当前只清 `StringToType`）。

**验证方式**
- grep：`grep -n "TryAdd\|Negative" Script/Interop/Bridge/TypeBridge.cs`（只有 :471 一处缓存，且被 `if (Type != null)` 包住）。
- 用例：在 C# 里故意引用一个不存在的类型名并高频调用 `Type.GetType` 路径，用 dotnet-trace 统计 `Assembly.GetType` 的调用次数。

---

### [F-ABI-027] `StringBridge.GetString` 的缓冲单位是"字符数"、`TypeBridge.Get*` 是"字节数"——同一族接口两种计量单位，且返回值语义不同（`-1` vs `0`）

- **类别**: 可优化/可读性（一致性）
- **严重度**: P3
- **复核结论**: 确认（两族接口的计量单位为"UTF-16 字符数" vs "UTF-8 字节数"、失败哨兵为 `-1` vs `0`，逐字核对无误）
- **可达性**: 活跃（`GetString` 与 `TypeBridge.Get*` 都是实际使用的桥接入口）
- **复核证据**: `Script/Interop/Bridge/StringBridge.cs:19-43`（第一手读取：`:20 char* InBuffer`（UTF-16）、`:26 var Length = String.Length`（字符数）、`:32 Buffer.MemoryCopy(..., (InSize - 1) * sizeof(char), Length * sizeof(char))`、`:42 return -1`）；`Script/Interop/Bridge/TypeBridge.cs:100-121`（`byte* OutString` + `Encoding.UTF8.GetBytes` + 失败 `return 0`，见 F-ABI-001 的复核证据）
- **级别变动**: 无（P3 维持）
- **函数**: `StringBridge.GetString(nint, char*, int)` vs `TypeBridge.GetNamespace/GetName/GetFullName(nint, byte*, int)`
- **置信度**: 高

**现状（代码事实）**
```csharp
// Script/Interop/Bridge/StringBridge.cs:20-42   —— 单位：char（UTF-16 元素个数）
        public static int GetString(nint InHandle, char* InBuffer, int InSize)
        {
            if (InBuffer != null && InSize > 0)
            {
                if (HandleData.GetObject(InHandle) is string String)
                {
                    var Length = String.Length;

                    if (Length < InSize)
                    {
                        fixed (char* Ptr = String)
                        {
                            Buffer.MemoryCopy(Ptr, InBuffer, (InSize - 1) * sizeof(char), Length * sizeof(char));
                        }

                        InBuffer[Length] = '\0';

                        return Length;
                    }
                }
            }

            return -1;      // ← "缓冲不足"信号
        }
```
C++ 侧接收方 `FScriptDomainImpl.inl:294-334 StringToFString`：
```cpp
			constexpr auto Size = 1024;
			char16_t String[Size];
			if (const auto Length = SCRIPT_DOMAIN_INVOKE(int32, StringBridgeGetStringFn, InManagedHandle, String, Size);
				Length >= 0) { ... }
			// 否则放大到 65536 重试
```

**问题**
同一"取字符串"协议在两处完全不同：
| | `StringBridge.GetString` | `TypeBridge.GetName/GetNamespace/GetFullName` |
|---|---|---|
| 缓冲单位 | `char` 个数（UTF-16 元素） | `byte`（UTF-8） |
| 编码 | UTF-16 | UTF-8 |
| C++ 侧缓冲 | `char16_t[1024]` → 放大到 65536（**栈上 32KB**，`TArray<char16_t>`） | `uint8[512]` → 翻倍到 64KB |
| 缓冲不足的信号 | `-1` | `0`（与"空字符串"不可区分） |
| 溢出保护 | `Length < InSize` 显式检查（**正确**） | `Math.Min(len, size-1)` 字符截断（**错误**，见 F-ABI-001） |

`TypeBridge.Get*` 返回 `0` 既表示"空命名空间"也表示"句柄无效"，C++ `TGetUTF8String`(`TGetUtf8String.inl:16-19`) 把 `Length <= 0` 一律当空字符串——"句柄无效"被伪装成"空名字"。

**建议**
把两者统一到同一个协议（推荐 UTF-8 + 返回 `所需字节数`，`> InSize` 表示需要放大；或统一 UTF-16）。`TypeBridge` 的三个方法按 F-ABI-001 的建议改造后，正好可以对齐到 `StringBridge` 已经做对的那套（`Length` 为元素/字节数，缓冲不足返回负数）。同时把 `StringToFString` 的 `char16_t String[1024]`（`FScriptDomainImpl.inl:302`）改为 `TArray` 或加大检查，避免每次取字符串都占 2KB 栈（该函数在反射遍历里被高频调用）。

**验证方式**
- grep：`grep -n "GetStringFn\|TypeBridgeGetNameFn" Source/UnrealCSharpCore/Public/Domain/Script/FScriptDomainImpl.inl`（对比两处调用点的缓冲类型）。
- 断言：给 `TypeBridge.Get*` 加 `Debug.Assert(Length < InStringSize)`（改造前会立刻暴露 F-ABI-001）。

---

### [F-ABI-028] `MethodBridge.Invoke` 对 by-ref 引用类型参数回写句柄但从不释放旧句柄（潜在句柄泄漏）

- **类别**: 内存/资源泄漏
- **严重度**: P3
- **复核结论**: 部分确认（**机制成立但后果比原文更具体**：by-ref 引用类型回写时 `*Param = HandleData.Alloc(Parameter)` 会为新对象 `Count++`，**旧对象的计数不递减**；由于句柄按对象去重（`ConditionalWeakTable`），泄漏表现为**旧对象永久保活（无法被 GC）**，而不是句柄表无限增长）
- **可达性**: 活跃（所有带 `ref`/`out` 引用类型参数的 C++→C# 反射调用）
- **复核证据**: `Script/Interop/Bridge/MethodBridge.cs:73-101`（第一手读取：`:79 if (ParameterType.IsByRef && InParams[Index] != 0)`、`:93 *Param = HandleData.Alloc(Parameter)`，无对应的 `HandleData.Free`）；`Script/Interop/Handle/HandleData.cs:27-44`（`Alloc` 每次 `HandleReference.Count++`）、`:46-68`（`FreeImplementation` 只在 `Count` 降到 0 时才真正释放 GCHandle）
- **级别变动**: 无（P3 维持；"潜在句柄泄漏"改为"旧对象永久保活"这一更准确的表述）
- **函数**: `Interop.MethodBridge.Invoke(nint, nint, int, nint*)`
- **置信度**: 低（无法在 C# 侧确认 C++ 调用方是否负责释放回写的句柄；需要 `FManagedFunctionDescriptor` 侧的配对分析，超出本报告范围）

**现状（代码事实）**
```csharp
// Script/Interop/Bridge/MethodBridge.cs:73-101
                    if (InParams != null)
                    {
                        for (var Index = 0; Index < MethodParameterLength && Index < InParamCount; Index++)
                        {
                            var ParameterType = MethodParameters[Index].ParameterType;

                            if (ParameterType.IsByRef && InParams[Index] != 0)
                            {
                                var Param = (nint*)InParams[Index];

                                var ElementType = ParameterType.GetElementType()!;

                                var Parameter = Parameters[Index];

                                if (ElementType.IsValueType)
                                {
                                    SetValue(InParams[Index], Parameter, ElementType);
                                }
                                else if (Parameter != null)
                                {
                                    *Param = HandleData.Alloc(Parameter);      // ← 每次调用都 Alloc 一个新句柄
                                }
                                else
                                {
                                    *Param = 0;
                                }
                            }
                        }
                    }
```
`HandleData.Alloc`(`HandleData.cs:27-44`) 对同一个对象会复用同一个 handle 值并 `Count++`。

**问题**
对 `ref`/`out` 的**引用类型**参数，每次调用都执行一次 `Alloc` → `Count` 递增。释放只发生在 `HandleData.Free`(`HandleData.cs:46-78`，即 C++ 侧显式 `Free(handle)`) 或 `AssemblyLoader.Unload` → `HandleData.Clear()`(`HandleData.cs:129-145`)。若 C++ 侧对 `out` 参数拿到的句柄只读取不 `Free`，`Count` 会持续增长，且 `Clear()` 之外**永不归零** → 长跑会话里句柄表持续膨胀、托管对象被 `GCHandle.Alloc(..., Normal)` 强引用而**无法被 GC 回收**（这是比字典增长更严重的问题：泄漏的是用户对象图）。

**建议**
- 回写前先取当前句柄：`var Old = *Param;` 然后 `*Param = HandleData.Alloc(Parameter); if (Old != 0) HandleData.Free(Old);`（若所有权在 C++ 侧则不可这么做，需要先明确所有权约定）。
- 或者约定"C++ 侧对 out 句柄负责 Free"，并在 `FManagedFunctionDescriptor` 的 out 参数处理里补上，同时在 `MethodBridge` 里加计数器（`Alloc` 次数 - `Free` 次数）在 `Deinitialize` 时 `UE_LOG` 告警不为 0。

**验证方式**
- grep：`grep -n "Alloc\|FreeImplementation" Script/Interop/Bridge/MethodBridge.cs`（`Alloc` 在 :93 与 :120，文件内无 `Free` 调用）。
- 用例：构造一个 `[UFunction]` 带 `out UObject` 参数并在 Tick 里高频调用，观察 `HandleData` 的 `Handles.Count` 是否单调增长（可用反射/调试器读取私有静态字段）。

---

## 4. C++ 导出 ↔ C# 声明 完整配对表

### 4.0 一个重要前提（验证方法说明）

`grep -rn 'extern "C"' Source/`（排除 `Source/ThirdParty/`）在本插件源码内 **0 命中**；`grep -rn "EntryPoint=" Script/` 也 **0 命中**。也就是说：

- 本插件的跨语言边界**不使用名称修饰的导出符号表**。`EntryPointNotFoundException` 这一类风险在本项目中**不以"缺 `extern "C"`"的形式出现**，而是以"`MethodBridge.GetMethod` 查不到 → 空指针直呼"（F-ABI-002）或"`mono_class_get_method_from_name`/`LoadAssemblyAndGetFunctionPointer` 解析失败 → 静默置空"（F-ABI-025）的形式出现。这是本报告与"通用 ABI 检查清单"最大的结构差异，已在第 3 节按本项目的真实形态重新定级。
- 唯一的 `[DllImport]` 出现在两处：`Script/Interop/Bridge/LogBridge.cs:27`（`DllImport("UnrealCSharp", CallingConvention = Cdecl)`，给 `LogLeanCLR` 用）与 `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:911`（**生成**到用户工程的 `[DllImport("UnrealCSharp", CallingConvention = Cdecl)]`，仅在 `#if WITH_LEANCLR` 下）。两者的模块名都是裸模块名 `"UnrealCSharp"`，**没有 `"__Internal"` 分支**（iOS 静态链接场景）；这一点在 §6 单独评估。

### 4.1 表 A：运行时桥接 typedef ↔ `[UnmanagedCallersOnly]`（51 个方法 / 41 行，逐参数核对）

C++ 侧统一定义在 `Source/UnrealCSharpCore/Public/Domain/Script/IScriptTypes.h`；C# 侧分布在 `Script/Interop/**` 与 `Script/UE/CoreUObject/{Utils,SynchronizationContext}.cs`。
**P/S** 列 = 参数个数/顺序是否一致；**T** 列 = 逐参数类型宽度是否一致；✅=一致，⚠=一致但依赖 ABI 细节，❌=不一致。

| # | C++ typedef（文件:行，签名） | C# 声明（文件:行，签名） | P/S | T | 问题 |
|---|---|---|---|---|---|
| 1 | `IScriptTypes.h:5` `IManagedHandle (*)(const uint8*, int32, const char16_t*)` | `AssemblyLoader.cs:14` `nint LoadFromStream(byte*, int, char*)` | ✅ | ⚠ | `char*`=UTF-16；C++ 调用点 `FMonoDomain.cpp:416` 用 `reinterpret_cast<const char16_t*>(*TCHAR*)` → **F-ABI-006** |
| 2 | `IScriptTypes.h:7` `void (*)()` | `AssemblyLoader.cs:31` `void Unload()` | ✅ | ✅ | — |
| 3 | `IScriptTypes.h:9` `void (*)(PTRINT)` | `HandleData.cs:25` `void Free(nint)` | ✅ | ✅ | `PTRINT` 与 `nint` 同宽（64 位平台 8B）；C++ 传 `InManagedHandle.Value`（`FScriptDomainImpl.inl:344`） |
| 4 | `IScriptTypes.h:11` `void (*)(PTRINT)` | `LogBridge.cs:31` `void SetLog(nint)` | ✅ | ✅ | C++ 传 `reinterpret_cast<PTRINT>(&FScriptLog::Log)`（`FScriptDomainImpl.inl:590`）；C# 转成 `delegate* unmanaged[Cdecl]<byte*,int,byte,void>`（`LogBridge.cs:33`），与 `FScriptLog.h:8` 的 `(const uint8*, int32, uint8)` **逐参数一致** ✅ |
| 5 | `IScriptTypes.h:13` `void (*)()` | `LogBridge.cs:37` `void Initialize()` | ✅ | ✅ | — |
| 6 | `IScriptTypes.h:15` `IManagedHandle (*)(const uint8*)` | `TypeBridge.cs:15` `nint GetClass(byte*)` | ✅ | ✅ | C# 用 `Marshal.PtrToStringUTF8`（`TypeBridge.cs:19`），C++ 用 `SCRIPT_DOMAIN_STRING_CAST`=`StringCast<UTF8CHAR>`（`FScriptDomainImpl.inl:13`）→ **编码一致** ✅ |
| 7 | `IScriptTypes.h:17` `IManagedHandle (*)(IManagedHandle)` | `TypeBridge.cs:36` `nint GetType(nint)` | ✅ | ⚠ | `IManagedHandle` 按值 ↔ `nint` → **F-ABI-019** |
| 8 | `IScriptTypes.h:19` `IManagedHandle (*)(IManagedHandle, const uint8*, int32)` | `TypeBridge.cs:49` `nint GetMethod(nint, byte*, int)` | ✅ | ✅ | 参数个数也用于 C++ 侧 `OutParamCount`；C# 校验了 `GetParameters().Length == InParamCount`（:65）✅ |
| 9 | `IScriptTypes.h:21` `PTRINT (*)(const char16_t*, const char16_t*, const char16_t*)` | `TypeBridge.cs:78` `nint GetFunctionPointer(char*, char*, char*)` | ✅ | ✅ | `char`(C#)=`char16_t`(C++)=2B ✅；C++ 传入用 `StringCast<UTF16CHAR>` 转换（`FScriptDomainImpl.inl:642-646`）→ 该处**正确** |
| 10 | `IScriptTypes.h:23` `int32 (*)(IManagedHandle, uint8*, int32)` | `TypeBridge.cs:101` `int GetNamespace(nint, byte*, int)` | ✅ | ❌ | **F-ABI-001**：缓冲单位错（UTF-16 字符数 vs UTF-8 字节数），超限抛异常穿出 → 进程终止 |
| 11 | `IScriptTypes.h:25` 同 #10 | `TypeBridge.cs:124` `int GetName(nint, byte*, int)` | ✅ | ❌ | 同 F-ABI-001 |
| 12 | `IScriptTypes.h:27` 同 #10 | `TypeBridge.cs:147` `int GetFullName(nint, byte*, int)` | ✅ | ❌ | 同 F-ABI-001（额外拼接 `", Assembly"`，更易超限） |
| 13 | `IScriptTypes.h:29` `IManagedHandle (*)(IManagedHandle, IManagedHandle)` | `TypeBridge.cs:174` `nint MakeGenericType(nint, nint)` | ✅ | ⚠ | 结构体按值 → F-ABI-019 |
| 14 | `IScriptTypes.h:31` `IManagedHandle (*)(IManagedHandle, IManagedHandle, IManagedHandle)` | `TypeBridge.cs:188` `nint MakeGenericType2(nint, nint, nint)` | ✅ | ✅ | **F-ABI-005**：第 3 参数未使用（函数体 bug，非签名 bug） |
| 15 | `IScriptTypes.h:33` `IManagedHandle (*)(int32*)` | `TypeBridge.cs:205` `nint BoxBool(int*)` | ✅ | ✅ | C++ 侧 `static_cast<int*>(InValue)`（`FScriptDomainImpl.inl:91`）与 `int32*` 一致；**bool 用 4 字节**，与 C# `bool` 的默认 4 字节封送一致 ✅（见 §4.5 的说明） |
| 16 | `IScriptTypes.h:35` `IManagedHandle (*)(int8*)` | `TypeBridge.cs:211` `nint BoxSByte(sbyte*)` | ✅ | ✅ | — |
| 17 | `IScriptTypes.h:37` `IManagedHandle (*)(int16*)` | `TypeBridge.cs:217` `nint BoxInt16(short*)` | ✅ | ✅ | — |
| 18 | `IScriptTypes.h:39` `IManagedHandle (*)(int32*)` | `TypeBridge.cs:223` `nint BoxInt32(int*)` | ✅ | ✅ | — |
| 19 | `IScriptTypes.h:41` `IManagedHandle (*)(int64*)` | `TypeBridge.cs:229` `nint BoxInt64(long*)` | ✅ | ✅ | C# `long` 恒 8B；C++ `int64` 8B ✅ |
| 20 | `IScriptTypes.h:43` `IManagedHandle (*)(uint8*)` | `TypeBridge.cs:235` `nint BoxByte(byte*)` | ✅ | ✅ | — |
| 21 | `IScriptTypes.h:45` `IManagedHandle (*)(uint16*)` | `TypeBridge.cs:241` `nint BoxUInt16(ushort*)` | ✅ | ✅ | — |
| 22 | `IScriptTypes.h:47` `IManagedHandle (*)(uint32*)` | `TypeBridge.cs:247` `nint BoxUInt32(uint*)` | ✅ | ✅ | — |
| 23 | `IScriptTypes.h:49` `IManagedHandle (*)(uint64*)` | `TypeBridge.cs:253` `nint BoxUInt64(ulong*)` | ✅ | ✅ | — |
| 24 | `IScriptTypes.h:51` `IManagedHandle (*)(float*)` | `TypeBridge.cs:259` `nint BoxFloat(float*)` | ✅ | ✅ | — |
| 25 | `IScriptTypes.h:53` `IManagedHandle (*)(double*)` | `TypeBridge.cs:265` `nint BoxDouble(double*)` | ✅ | ✅ | — |
| 26-36 | `IScriptTypes.h:55-75` `int32 (*)(IManagedHandle, T*)` × 11（Unbox Bool/SByte/Int16/Int32/Int64/Byte/UInt16/UInt32/UInt64/Float/Double） | `TypeBridge.cs:271/287/303/319/335/351/367/383/399/415/431` `int UnboxX(nint, T*)` | ✅×11 | ✅×11 | 逐条核对 `int32↔int`、`int8↔sbyte`、`int16↔short`、`int32↔int`、`int64↔long`、`uint8↔byte`、`uint16↔ushort`、`uint32↔uint`、`uint64↔ulong`、`float↔float`、`double↔double`，**全部一致**；返回值语义 `1=成功/0=失败`，C++ 侧 `SCRIPT_DOMAIN_INVOKE(int32, ...)` 当条件用（`FScriptDomainImpl.inl:181/191/...`）✅ |
| 37 | `IScriptTypes.h:77` `IManagedHandle (*)(IManagedHandle)` | `ObjectBridge.cs:10` `nint NewObject(nint)` | ✅ | ⚠ | `RuntimeHelpers.GetUninitializedObject` 对抽象类/数组/指针类型抛异常且**无 try/catch** → 与 F-ABI-003 同类（未单列，因为它只会被 `NewObject(InManagedClass)` 调用，而那只对已注册的具体类生效） |
| 38 | `IScriptTypes.h:79` `void (*)(IManagedHandle, const uint8*, IManagedHandle)` | `FieldBridge.cs:11` `void SetStaticValue(nint, byte*, long)` | ✅ | ⚠ | 第 3 参 `IManagedHandle`(8B struct) ↔ `long`(8B)：x64/ARM64 寄存器传递等价；C++ 实际只读 4 字节（`FScriptDomainImpl.inl:399`）→ **F-ABI-007** |
| 39 | `IScriptTypes.h:81` `IManagedHandle (*)(IManagedHandle, const uint8*)` | `FieldBridge.cs:22` `long GetStaticValue(nint, byte*)` | ✅ | ⚠ | 返回 struct ↔ `long`；异常无保护 → **F-ABI-007** |
| 40 | `IScriptTypes.h:83` `void (*)(const uint8* const*, const PTRINT*, int32)` | `MethodBridge.cs:13` `void RegisterBinding(byte**, nint*, int)` | ✅ | ✅ | `const uint8* const*` ↔ `byte**`：C# 的 `byte**` 本身不表达"指针不可改"，但**不影响 ABI**；C++ 侧数组生命周期见 `FScriptDomainImpl.inl:605-631`（局部 `TArray<TArray<ANSICHAR>> Names`，内层缓冲是独立堆分配，外层重分配不影响已存入的指针 ✅） |
| 41 | `IScriptTypes.h:85` `IManagedHandle (*)(IManagedHandle, IManagedHandle, int32, IManagedHandle*)` | `MethodBridge.cs:33` `nint Invoke(nint, nint, int, nint*)` | ✅ | ⚠ | C++ 传 `reinterpret_cast<IManagedHandle*>(InParams)`（`FScriptDomainImpl.inl:472`），即 `void**` 被当作 `IManagedHandle*`（元素=8B）→ 与 C# 的 `*(nint*)InParams[Index]` 解引用约定一致 ✅；`ref` 回写句柄的所有权未定义 → F-ABI-028 |
| 42 | `IScriptTypes.h:87` `IManagedHandle (*)(const uint8*)` | `StringBridge.cs:9` `nint NewString(byte*)` | ✅ | ✅ | C++ 传 `reinterpret_cast<const uint8*>(InText)`（`FScriptDomainImpl.inl:288`），`InText` 来自 `TCHAR_TO_UTF8`（`FRegisterString.cpp:46` 等）→ UTF-8 一致 ✅ |
| 43 | `IScriptTypes.h:89` `int32 (*)(IManagedHandle, char16_t*, int32)` | `StringBridge.cs:20` `int GetString(nint, char*, int)` | ✅ | ✅ | 单位是**字符数**（`char16_t`），C++ 传 `char16_t String[1024]`（`FScriptDomainImpl.inl:302`）✅；语义 `-1`=缓冲不足，C++ 用 `Length >= 0` 判断（:305）✅ → 与 `TypeBridge.Get*` 的协议不同（F-ABI-027） |
| 44 | `IScriptTypes.h:91` `IManagedHandle (*)(const uint8*, int32)` | `ArrayBridge.cs:9` `nint NewArray(byte*, int)` | ✅ | ✅ | `Array.CreateInstance(Type, InLength)`：`InLength <= 0` 时 C++ 已过滤（`FScriptDomainImpl.inl:353`）✅ |
| 45 | `IScriptTypes.h:93` `IManagedHandle (*)(IManagedHandle, int32)` | `ArrayBridge.cs:27` `nint ArrayGet(nint, int)` | ✅ | ✅ | 参数一致；**负索引未校验** → F-ABI-017 |
| 46 | `IScriptTypes.h:95` `int32 (*)(IManagedHandle)` | `Utils.cs:619` `int IsOverride(nint)` | ✅ | ✅ | C++ 按 `!= 0` 当 bool 用（`FScriptDomainImpl.inl:480`）✅ 用 `int32` 而非 `bool`，**规避了 bool 宽度问题** ✅ |
| 47 | `IScriptTypes.h:97` `void (*)(IManagedHandle, PTRINT*)` | `Utils.cs:633` `void GetClassDescriptor(nint, nint*)` | ✅ | ✅ | C# 写 `OutBuffer[0..14]`（15 槽），C++ 提供 `Params[16]` 的 `&Params[1]`（15 槽）→ **恰好匹配** ✅ 但无护栏 → F-ABI-008 |
| 48 | `IScriptTypes.h:99` 同 #47 | `Utils.cs:681` `void GetClassProperties(nint, nint*)` | ✅ | ✅ | 8 槽 ↔ `InSlotCount = 8`（`FClassReflection.cpp:467`）✅ → F-ABI-008 |
| 49 | `IScriptTypes.h:101` 同 #47 | `Utils.cs:713` `void GetClassFields(nint, nint*)` | ✅ | ✅ | 3 槽 ↔ `InSlotCount = 3`（`FClassReflection.cpp:473`）✅ |
| 50 | `IScriptTypes.h:103` 同 #47 | `Utils.cs:730` `void GetClassMethods(nint, nint*)` | ✅ | ✅ | 14 槽 ↔ `InSlotCount = 14`（`FClassReflection.cpp:479`）✅ |
| 51 | `IScriptTypes.h:105` `void (*)(float)` | `SynchronizationContext.cs:27` `void Tick(float)` | ✅ | ✅ | C# `float`(=`Single`, 4B) ↔ C++ `float` 4B ✅；C++ 传 `InDeltaTime`（`FScriptDomainImpl.inl:21`）✅ |

> 表 A 共 **41 行**，覆盖 **51 个 C# `[UnmanagedCallersOnly]` 方法**（`Box*`/`Unbox*` 两族各 11 个方法各合并为 1 行；未覆盖的 2 个见 §5.2(4) 的说明）。逐参数核对结论：**参数个数/顺序 51/51 一致；类型宽度仅 5 处标记为 ⚠（#1、#7、#13、#38、#39），1 族 3 条（#10/#11/#12）为 ❌**，其余完全一致。

### 4.2 表 B：绑定注册方法 ↔ `__X_YImplementation`（**209** 对；原文记 211，复核已修正——抽样逐参数核对 + 全量规则化核对）

**(a) 全量规则化核对（211/211）**

用脚本抽取 `Script/**/*.cs` 里所有 `__<Class>_<Method>Implementation(` 声明（211 个去重后的 `(Class, Method)` 对），再在 `Source/UnrealCSharp/Private/Domain/Interop/**` 的**全部**文本里查找 `"<Method>"` 字面量：

```
total distinct (class,method) pairs: 211
--- methods not found as a quoted string in C++ Interop dir: 0 ---
```
即 **211/211 都能在 C++ 侧找到对应的 `.Function("<Method>", ...)` 注册**。逐字符的名称拼接链已在本报告 §1 机制 B 中核对（`FBindingClassRegister.cpp:113-127` ↔ `UnrealTypeSourceGenerator.cs:903`）。

**(b) 逐参数核对抽样（读了两侧的完整签名）**

| # | C++ 绑定函数（文件:行） | C# 声明（文件:行） | 逐参数结论 |
|---|---|---|---|
| 1 | `FRegisterUnreal.cpp:45-50` `IManagedHandle NewObjectImplementation(IManagedHandle Outer, IManagedHandle Class, IManagedHandle Name, EObjectFlags Flags, IManagedHandle Template, uint8 bCopyTransientsFromClassDefaults)` | `UnrealImplementation.cs:9` `nint __Unreal_NewObjectImplementation(nint Outer, nint Class, nint Name, EObjectFlags Flags, nint Template, byte bCopyTransientsFromClassDefaults)` | 6/6 一致；**bool 用了 `uint8`↔`byte`** ✅（对照 `UnrealImplementation.cs:12` 的托管包装 `bool` → `(byte)(b?1:0)`，:15） |
| 2 | `FRegisterUnreal.cpp:70-72` `DuplicateObjectImplementation(IManagedHandle, IManagedHandle, IManagedHandle)` | `UnrealImplementation.cs:20` `__Unreal_DuplicateObjectImplementation(nint, nint, nint)` | 3/3 ✅ |
| 3 | `FRegisterUnreal.cpp:87-91` `LoadObjectImplementation(IManagedHandle, IManagedHandle, IManagedHandle, ELoadFlags, IManagedHandle)` | `UnrealImplementation.cs:29` `__Unreal_LoadObjectImplementation(nint, nint, nint, ELoadFlags, nint)` | 5/5 ✅ |
| 4 | `FRegisterUnreal.cpp:111-115` `LoadClassImplementation(...5 参)` | `UnrealImplementation.cs:39` 同 | 5/5 ✅ |
| 5 | `FRegisterUnreal.cpp:135-136` `CreateWidgetImplementation(IManagedHandle, IManagedHandle)` | `UnrealImplementation.cs:49` | 2/2 ✅ |
| 6 | `FRegisterUnreal.cpp:168` `GWorldImplementation()` | `UnrealImplementation.cs:58` | 0/0 ✅ |
| 7 | `FRegisterUnreal.cpp:173` `GetTransientPackageImplementation()` | `UnrealImplementation.cs:67` | 0/0 ✅ |
| 8 | `FRegisterWorld.cpp:33` `uint8 GetbNoFailImplementation(IManagedHandle)` | `FActorSpawnParametersImplementation.cs:7` `byte __FActorSpawnParameters_GetbNoFailImplementation(nint)` | 1/1 ✅ bool→`uint8`↔`byte` ✅ |
| 9 | `FRegisterWorld.cpp:44` `void SetbNoFailImplementation(IManagedHandle, uint8)` | `FActorSpawnParametersImplementation.cs:14` `void __...(nint, byte)` | 2/2 ✅ |
| 10 | `FRegisterWorld.cpp:53/64/73/84` 其余 4 个 `Get/SetbDeferConstruction`、`Get/SetbAllowDuringConstructionScript` | `FActorSpawnParametersImplementation.cs:21/28/35/42` | 6/6 ✅ |
| 11 | `FRegisterWorld.cpp:120-123` `IManagedHandle SpawnActorImplementation(IManagedHandle, IManagedHandle, IManagedHandle, IManagedHandle)` | `WorldImplementation.cs:8` `__UWorld_SpawnActorImplementation(nint, nint, nint, nint)` | 4/4 ✅ |
| 12 | `FRegisterString.cpp:13` `void RegisterImplementation(IManagedHandle, const char*)` | `FStringImplementation.cs:9` `void __FString_RegisterImplementation(nint, byte*)` | 2/2 ✅；UTF-8 双向一致（`FStringImplementation.cs:13` 用 `Encoding.UTF8.GetBytes` ↔ `FRegisterString.cpp:15` 用 `UTF8_TO_TCHAR`）✅ |
| 13 | `FRegisterString.cpp:21` `uint8 IdenticalImplementation(IManagedHandle, IManagedHandle)` | `FStringImplementation.cs:21` `byte __FString_IdenticalImplementation(nint, nint)` | 2/2 ✅ |
| 14 | `FRegisterString.cpp:34` `void UnRegisterImplementation(IManagedHandle)` | `FStringImplementation.cs:28` | 1/1 ✅ |
| 15 | `FRegisterString.cpp:42` `IManagedHandle ToStringImplementation(IManagedHandle)` | `FStringImplementation.cs:35` `nint __FString_ToStringImplementation(nint)` | 1/1 ✅ |
| 16 | `FRegisterText.cpp:16-20` `void RegisterImplementation(IManagedHandle, const char*, const char*, const char*, uint8)` | `FTextImplementation.cs:8` `void __FText_RegisterImplementation(nint, byte*, byte*, byte*, byte)` | **5/5 ✅**（特别注意：C++ 是 5 个参数而非 4 个，C# 也声明了 5 个；`bRequiresQuotes` 用 `uint8`↔`byte` ✅，包装层 `FTextImplementation.cs:26` 做 `(byte)(bRequiresQuotes?1:0)`） |
| 17 | `FRegisterText.cpp:47/60/68` `Identical/UnRegister/ToString` | `FTextImplementation.cs:30/37/44` | ✅ |
| 18 | `FRegisterObject.cpp:16` `uint8 IdenticalImplementation(IManagedHandle, IManagedHandle)` | `ObjectImplementation.cs:9` `byte __UObject_IdenticalImplementation(nint, nint)` | 2/2 ✅ |
| 19 | `FRegisterObject.cpp:29` `IManagedHandle StaticClassImplementation(const char*)` | `ObjectImplementation.cs:16` `nint __UObject_StaticClassImplementation(byte*)` | 1/1 ✅ |
| 20 | `FRegisterObject.cpp:38/50/62/72/85/93/101/111/121` 其余 9 个 | `ObjectImplementation.cs:32/41/50/57/64/71/78/85/92` | 全部一致；`uint8`↔`byte`（:50/57/78/85/92）✅ |
| 21 | `FRegisterStruct.cpp:13` `IManagedHandle StaticStructImplementation(const char*)` | `StructImplementation.cs:9` `nint __UStruct_StaticStructImplementation(byte*)` | 1/1 ✅ |
| 22 | `FRegisterStruct.cpp:22` `void RegisterImplementation(IManagedHandle, const char*)` | `StructImplementation.cs:25` `void __UStruct_RegisterImplementation(nint, byte*)` | 2/2 ✅ |
| 23 | `FRegisterStruct.cpp:29-30` `uint8 IdenticalImplementation(IManagedHandle, IManagedHandle, IManagedHandle)` | `StructImplementation.cs:37` `byte __UStruct_IdenticalImplementation(nint, nint, nint)` | 3/3 ✅ |
| 24 | `FRegisterStruct.cpp:47` `void UnRegisterImplementation(IManagedHandle)` | `StructImplementation.cs:44` | 1/1 ✅ |
| 25 | `FRegisterArray.cpp:14/21/34/43/54/65/76/87/98` + `Get` | `TArrayImplementation.cs:9/16/23/30/37/44/51/58/65/72` | ✅；`IN_BUFFER_SIGNATURE`=`uint8*`（`BufferMacro.h:9`）↔ `byte*` ✅ |
| 26 | `FRegisterMap.cpp:13/23/32/41/52/63-64/73-74/85-86/96-97/107/118-119` | `TMapImplementation.cs:9/16/23/30/37/44/51/58/65/72/79/86/93/100/107/114` | ✅；特别注意 `FRegisterMap.cpp:85-86 FindKeyImplementation(IManagedHandle, IN_VALUE_BUFFER_SIGNATURE, RETURN_BUFFER_SIGNATURE)` 用的是 **VALUE** 缓冲——与 `FMapHelper.h:29 FindKey(const void* InValue)` 语义一致，C# `TMapImplementation.cs:79` 也命名为 `InValueBuffer` ✅ **不是 bug** |
| 27 | `FRegisterClass.cpp:12` `void RemoveFunctionImplementation(IManagedHandle, IManagedHandle)` | `ClassImplementation.cs:7` `void __UClass_RemoveFunctionImplementation(nint, nint)` | 2/2 ✅ |
| 28 | `FRegisterDataTableFunctionLibrary.cpp:11` `uint8 GetDataTableRowFromNameImplementation(IManagedHandle, IManagedHandle, nint*)`（以 `.cpp:11-14` 的实际签名为准，含 `IManagedHandle* OutRow`） | `DataTableFunctionLibraryImplementation.cs:7` `byte __UDataTableFunctionLibrary_GetDataTableRowFromNameImplementation(nint, nint, nint*)` | 3/3 ✅ |

**规则化结论（覆盖全部 211 条）**：
1. **返回类型**：C# 侧只出现 `void` / `nint` / `byte` / `int` / `uint`，**从未出现 `bool` / `float` / `double` / 结构体按值 / 字符串**。
   - grep 证据：`grep -n "partial (void|int|uint|byte|sbyte|short|ushort|long|ulong|float|double|nint|nuint|bool|string)(\[\])? __.*;" Script/` 命中 211 行，**`bool` 0 命中**。
   - 对应 C++ 侧：`bool` 一律写成 `uint8`（例如 `FRegisterObject.cpp:16/62/72/101/111/121`、`FRegisterWorld.cpp:33/53/73`、`FRegisterStruct.cpp:29`、`FRegisterArray.cpp:21/65/87`），**bool 宽度问题在这个方向上被系统性规避了** ✅
2. **复合类型**：一律走 `byte*`/`uint8*` 缓冲或 `nint` 句柄，**没有结构体按值传递**（除了 `IManagedHandle` 本身，见 F-ABI-019）。这消除了"MSVC vs Clang 小结构体返回 ABI 差异"的整类风险 ✅
3. **`long` vs C++ `long`**：C# 的 `long` 只出现在 `FieldBridge`（表 A #38/#39），表 B 内 **0 处**；C++ 侧**没有使用裸 `long`**（全部是 `int32`/`int64`/`uint32`/`uint64`）→ **跨平台 `long` 宽度差异在本项目中不存在** ✅（`grep -n "[^a-zA-Z_]long " Source/` 在插件自身代码里未见 `long` 形参）。
4. **`EntryPoint` 名与 C++ 导出名**：两侧都由宏生成（`BINDING_COMBINE_FUNCTION_IMPLEMENTATION` ↔ 手写 `__X_YImplementation`），**没有名称修饰问题**，但也没有编译期校验 → 见 F-ABI-002/F-ABI-004。

### 4.3 表 C：C++ 有导出但 C# 无声明（死代码/未完成候选）

| C++ 符号 | 声明位置 | C# 侧对应 | 判定 | grep 证据 |
|---|---|---|---|---|
| `BINDING_COMBINE_FUNCTION_IMPLEMENTATION` 生成名 | `BindingMacro.h:9` | 211 个 `__X_YImplementation` | 211/211 有对应，**无缺口** | `pwsh` 抽取脚本，`missing = 0` |
| `LogBridgeInitializeLeanCLR` | `FunctionMacro.h:50`（`#if WITH_LEANCLR`） | 不在 `COMMON/NATIVE_BRIDGE_METHODS` 里，**无 typedef**，靠 `FLeanCLRDomain.cpp:69-75` 用**类名+方法名**直接解析 | 非死代码（LeanCLR 专用入口） | `grep -n "InitializeLeanCLR" Source/ Script/` → 定义 `FunctionMacro.h:50`、调用 `FLeanCLRDomain.cpp:72`、C# `LogBridge.cs:47` |
| `FUNCTION_LOG_BRIDGE_LOG_LEANCLR` | `FunctionMacro.h:52` | `LogBridge.cs:28` 的 `[DllImport] LogLeanCLR` | 有对应 | `FLeanCLRDomain.cpp:884-887` 显式 `RegisterPInvoke(... + FUNCTION_LOG_BRIDGE_LOG_LEANCLR)` |
| `FUNCTION_HANDLE_DATA_GET_OBJECT_POINTER(S)` / `FUNCTION_HANDLE_DATA_ALLOC` | `FunctionMacro.h:39/43`（复数 `:41` **已于本轮删除**） | `HandleData.GetObjectPointer/Alloc`（复数 `GetObjectPointers` 已删除） | **LeanCLR 专用**，在 `LEANCLR_INTEROP_BRIDGE_METHODS`（`FLeanCLRDomain.h:195-197`）里，**不在** `IScriptTypes.h` 的通用表里。〔**更正（本轮）**：该表实际只有 `GetObjectPointer`（`:196`）+ `Alloc`（`:197`）两项，复数 `..._POINTERS` **从未进入任何桥表** → 故为死宏并已删除；单数版不受影响〕 | `grep -n "HandleDataGetObjectPointer" Source/` → 仅 `FLeanCLRDomain.h:196` 定义 + `FLeanCLRDomain.cpp:125/155/167/196/417` 使用 |

**结论：表 C 无缺口**（C++ 侧的桥接入口与绑定注册都能在 C# 侧找到对应；不存在"导出但没人用"的死导出）。

### 4.4 表 D：C# 有声明但 C++ 无实现 → 运行时 `EntryPointNotFoundException` 风险

| C# 声明 | 文件:行 | C++ 侧 | 判定 |
|---|---|---|---|
| 211 个 `__X_YImplementation` private partial | `Script/UE/Library/*.cs` | 每个都有 `.Function("<Method>", ...)` 注册（脚本核对 `missing = 0`） | **无缺口** |
| `MethodBridge.GetMethod(ref nint, string)` | `MethodBridge.cs:140` | 这是 C# 内部方法，无 C++ 对应（不适用） | — |
| `LogBridge.LogLeanCLR` `[DllImport("UnrealCSharp")]` | `LogBridge.cs:27-28` | 由 `FLeanCLRDomain.cpp:884-887` 用 `register_pinvoke` 以**方法名**注册（不是 OS 导出） | 依赖 LeanCLR 的 pinvoke 按名解析；**未验证**模块名 `"UnrealCSharp"` 是否参与键匹配（LeanCLR 的 `PInvokes::get_pinvoke_by_method` 实现在 `Source/ThirdParty/LeanCLR/src/runtime/**` 的 .cpp 里，按排除范围未读）→ 见 §9 存疑项 |
| 生成的 `[DllImport("UnrealCSharp")] static extern ... __X_YImplementation(...)` | `UnrealTypeSourceGenerator.cs:910-912`（**生成到用户工程**） | 同上 | 同上 |

**关键差别**：本表 D 里"C# 有声明、C++ 无实现"的失败模式**不是** `EntryPointNotFoundException`（因为不走 CLR 的 P/Invoke 解析），而是：
- 机制 B 路径 → `MethodBridge.GetMethod` 返回 0 → **空指针直呼崩溃**（F-ABI-002）；
- LeanCLR `[DllImport]` 路径 → 取决于 LeanCLR 的 pinvoke 查找失败处理（未验证，见 §9）。
这两条已在第 3 节按各自严重度定级，此处不重复计数。

---

## 5. C++ → C# 回调方向 ABI 核对

### 5.1 C++ 侧调用 C# 的入口清单

| 入口 | C++ 位置 | 注册/解析机制 | C# 目标 |
|---|---|---|---|
| `mono_runtime_invoke` | `FMonoDomain.cpp:236` | `mono_class_get_method_from_name` 选方法 | `MethodBridge.Invoke` 内部走反射；另有 `Runtime_Invoke` 直调托管方法（如 `Utils.GetTypesWithAttribute`，`FScriptDomainImpl.inl:546-560`） |
| `mono_method_get_unmanaged_callers_only_ftnptr` | `FMonoDomain.cpp:259` | `mono_class_get_method_from_name(类, 方法名, 参数个数)` | 44 个桥接入口 |
| `load_assembly_and_get_function_pointer` | `FCoreCLRDomain.cpp:303-309` | `"Interop.TypeBridge, Interop"` + 方法名 + `UNMANAGEDCALLERSONLY_METHOD` | 44 个桥接入口 |
| LeanCLR `Bridge_Invoke` → 解释器栈调用 | `FLeanCLRDomain.inl:60` | `RtModuleDef::get_class_by_nested_full_name` + `find_matched_method_in_class_by_name` | 44 个桥接入口（`FLeanCLRDomain.h:195-205` 的三张表） |
| LeanCLR pinvoke 分派 `PInvoke_Dispatch` | `FLeanCLRDomain.cpp:604-709` | `PInvokes::register_pinvoke(Method.GetMethod(), 函数地址, &PInvoke_Dispatch)`（`FLeanCLRDomain.cpp:870-873`） | C# 生成的 `[DllImport]`（211 + 用户绑定） |
| 托管函数指针 | `TypeBridge.GetFunctionPointer`（`TypeBridge.cs:78-98`） | C++ 传 `(assembly, type, method)` 名字，拿 `Method.MethodHandle.GetFunctionPointer()` | `Utils.*`、`SynchronizationContext.Tick`（`FScriptDomainImpl.inl:641-647`） |

### 5.2 逐项核对结果

**(1) C++ typedef ↔ C# 方法签名**：见 §4.1 表 A（51 行逐参数）。结论：**全部一致**，唯一 ❌ 是 UTF-8 缓冲协议（F-ABI-001）。

**(2) 调用约定**：C++ 全部用无修饰的普通函数指针 typedef（默认 cdecl/平台默认）；C# 侧 14 处显式 `CallConvCdecl`、39 处缺省 → **F-ABI-013**。

**(3) 字符串所有权**：
- **C++ → C#（UTF-8 输入）**：`SCRIPT_DOMAIN_STRING_CAST`（`FScriptDomainImpl.inl:11-14`）产生 `StringCast<UTF8CHAR>` 的**临时对象**，`.Get()` 的指针在**完整表达式内**有效，C# 侧只读不保存（`Marshal.PtrToStringUTF8`，`TypeBridge.cs:19`/`ArrayBridge.cs:13`/`StringBridge.cs:13`/`FieldBridge.cs:38`）→ **所有权清晰、无跨调用悬垂** ✅。C# 侧 `NewString`/`GetClass` 返回的是**托管 string/Type 的句柄**，由 C++ 侧 `Free(handle)` 释放（`FScriptDomainImpl.inl:338-347` → `HandleDataFreeFn(InManagedHandle.Value)` → `HandleData.Free`）→ **归还协议存在** ✅。
- **C# → C++（UTF-8 输出）**：由调用方提供缓冲（`TGetUTF8String` 的 `uint8[512]`），**不涉及释放**；但见 F-ABI-001（缓冲不足 → 异常）。
- **UTF-16 输出**：`StringBridge.GetString(nint, char16_t*, int32)`（表 A #43）由 C++ 提供 `char16_t[1024]`（`FScriptDomainImpl.inl:302`），C# 不分配内存 → **无泄漏** ✅。
- **不平衡项**：`FScriptDomainImpl.inl:576 Free(Types)` / `:572 Free(Element)` 与 `FLeanCLRDomain.cpp:234/238` 成对；但 `FClassReflection::EnsureDescriptorImplementation` 里 `HandleData.Alloc(Type)`（`TypeBridge.cs:27`）产生的句柄由谁释放需另行核对（不在本报告范围）。

**(4) `[UnmanagedCallersOnly]` 异常绝不能穿出 — 逐方法核查**（`grep -rn "UnmanagedCallersOnly" Script/` 共 **53** 处属性标注：`TypeBridge.cs` 31、`LogBridge.cs` 4、`Utils.cs` 5、`MethodBridge.cs`/`AssemblyLoader.cs`/`FieldBridge.cs`/`ArrayBridge.cs`/`StringBridge.cs` 各 2、`HandleData.cs`/`ObjectBridge.cs`/`SynchronizationContext.cs` 各 1）：

| 方法 | 位置 | try/catch | 可能抛出的调用 | 结论 |
|---|---|---|---|---|
| `HandleData.Free` | `HandleData.cs:24-25` | ❌ | `OutHandle.Free()`（被 `TryGetValue` 守卫）；`ObjectToHandleReference.TryGetValue` | 低风险（P3） |
| `AssemblyLoader.LoadFromStream` | `AssemblyLoader.cs:13-28` | ❌ | `Context.LoadFromStream` → `BadImageFormatException`/`FileLoadException` | **P0 → F-ABI-003** |
| `AssemblyLoader.Unload` | `AssemblyLoader.cs:30-61` | `try/finally`（无 catch） | `Context.Unload()` | 低（`Unload` 基本不抛） |
| `TypeBridge.GetClass` | `TypeBridge.cs:15-33` | ❌ | `GetTypeImplementation` → `Type.GetType`(throwOnError:false) / `StringToType.TryAdd` | 低 |
| `TypeBridge.GetType` | `TypeBridge.cs:36-46` | ❌ | `HandleData.Alloc` | 低 |
| `TypeBridge.GetMethod` | `TypeBridge.cs:49-75` | ❌ | `Type.GetMethods`（可能抛 `TypeLoadException`） | 中（P2，未单列） |
| `TypeBridge.GetFunctionPointer` | `TypeBridge.cs:78-98` | ❌ | `Type.GetMethod`/`MethodHandle.GetFunctionPointer` | 中 |
| `TypeBridge.GetNamespace` | `TypeBridge.cs:101-121` | ❌ | **`Encoding.UTF8.GetBytes` → `ArgumentException`** | **P0 → F-ABI-001** |
| `TypeBridge.GetName` | `TypeBridge.cs:124-144` | ❌ | 同上 | **P0 → F-ABI-001** |
| `TypeBridge.GetFullName` | `TypeBridge.cs:147-171` | ❌ | 同上 | **P0 → F-ABI-001** |
| `TypeBridge.MakeGenericType` | `TypeBridge.cs:174-185` | ❌ | `Generic.MakeGenericType` → `ArgumentException`/`InvalidOperationException` | **P1**（与 F-ABI-005 同族，未单列） |
| `TypeBridge.MakeGenericType2` | `TypeBridge.cs:188-202` | ❌ | 同上 | **P1 → F-ABI-005** |
| `TypeBridge.Box*` ×11 | `TypeBridge.cs:205-268` | ❌ | `HandleData.Alloc` | 低 |
| `TypeBridge.Unbox*` ×11 | `TypeBridge.cs:271-444` | ❌ | `HandleData.GetObject` | 低 |
| `ObjectBridge.NewObject` | `ObjectBridge.cs:9-20` | ❌ | **`RuntimeHelpers.GetUninitializedObject`**（抽象类/数组/指针/开放泛型 → `MemberAccessException`/`ArgumentException`） | **P1**（未单列，建议同一批修复） |
| `FieldBridge.SetStaticValue` | `FieldBridge.cs:10-19` | ❌ | **`Field.SetValue` → `FieldAccessException`/`ArgumentException`** | **P1 → F-ABI-007** |
| `FieldBridge.GetStaticValue` | `FieldBridge.cs:21-32` | ❌ | **`Convert.ToInt64` → `OverflowException`/`FormatException`** | **P1 → F-ABI-007** |
| `MethodBridge.RegisterBinding` | `MethodBridge.cs:12-30` | ❌ | `Marshal.PtrToStringUTF8`；字典索引器 | 低 |
| `MethodBridge.Invoke` | `MethodBridge.cs:32-138` | ✅ **有完整 try/catch**（:35/:125-137） | `Method.Invoke`、`Convert.ChangeType`、`SwitchExpressionException`(F-ABI-016) | ✅ 良好实践，可作为其余 43 个的模板 |
| `StringBridge.NewString` | `StringBridge.cs:8-17` | ❌ | `Marshal.PtrToStringUTF8` | 低 |
| `StringBridge.GetString` | `StringBridge.cs:19-43` | ❌ | `Buffer.MemoryCopy`（无抛） | 低 |
| `ArrayBridge.NewArray` | `ArrayBridge.cs:8-24` | ❌ | **`Array.CreateInstance`**（`void`/指针类型/负长度 → `ArgumentException`/`OverflowException`） | **P1**（未单列） |
| `ArrayBridge.ArrayGet` | `ArrayBridge.cs:26-43` | ❌ | **`Array.GetValue(-1)` → `IndexOutOfRangeException`** | **P2 → F-ABI-017** |
| `Utils.IsOverride` | `Utils.cs:618-630` | ❌ | `Type.IsDefined` | 低 |
| `Utils.GetClassDescriptor` | `Utils.cs:632-678` | ❌ | `GetClassDescriptorImplementation`（反射 + `HandleData.Alloc`） | 中 |
| `Utils.GetClassProperties` | `Utils.cs:680-710` | ❌ | 同上 | 中 |
| `Utils.GetClassFields` | `Utils.cs:712-727` | ❌ | 同上 | 中 |
| `Utils.GetClassMethods` | `Utils.cs:729-773` | ❌ | 同上 | 中 |
| `SynchronizationContext.Tick` | `SynchronizationContext.cs:26-30` | ❌ | `Context?.Process()` → `Task.Invoke()`（`TaskInfo.Invoke`） | **P2**（用户回调抛异常会穿出 → 进程终止；未单列，建议在 `Process` 里逐个 task 包 try/catch） |
| `LogBridge.SetLog/Initialize/InitializeLeanCLR/Deinitialize` | `LogBridge.cs:30/36/46/54` | ❌ | `Console.SetOut/SetError` | 低 |

**统计：53 个 `[UnmanagedCallersOnly]` 中只有 1 个（`MethodBridge.Invoke`）有完整 try/catch，占 1.9%**（52 个无保护）。其中已确证会由正常输入触发进程终止的有 4 个：`AssemblyLoader.LoadFromStream`(F-ABI-003)、`TypeBridge.GetNamespace/GetName/GetFullName`(F-ABI-001，3 个方法)。这是本报告最重要的横向结论之一。

> 与 §4.1 表 A 的对应关系：表 A 覆盖 51 个方法（41 行，其中 Box*/Unbox* 各合并为 1 行）；表 A 未覆盖的 2 个是 `LogBridge.InitializeLeanCLR`(`LogBridge.cs:46`，仅由 `FLeanCLRDomain.cpp:69-75` 按名解析调用，无 typedef 表项) 与 `LogBridge.Deinitialize`(`LogBridge.cs:54`，在当前插件源码中**无调用者**——`grep -rn "Deinitialize" Script/Interop/` 只命中定义，属待确认的外部 API 或死代码）。

**(5) 委托保活（`Marshal.GetFunctionPointerForDelegate` → 悬垂指针）**：
`grep -rn "GetFunctionPointerForDelegate" Source/ Script/` → **0 命中**（插件自身代码）。唯一的托管函数指针获取路径是 `TypeBridge.GetFunctionPointer`(`TypeBridge.cs:78-98`) 用 `Method.MethodHandle.GetFunctionPointer()`：
- 目标方法必须是 **static**（`:87-88` 的 `BindingFlags.Static`）且是 `[UnmanagedCallersOnly]`（否则 `MethodHandle.GetFunctionPointer()` 得到的是 JIT stub，AOT/解释模式下可能无效）；
- **不涉及委托实例** → 不存在"委托被 GC 回收导致指针悬垂"的经典问题 ✅（因为目标是静态方法，方法表由 Loader 保活）。这一点值得明确记录为"无此风险"，避免后续有人改成 `Delegate.CreateDelegate` 时踩坑。
- 唯一相近的风险在 `LogBridge.cs:11` 的 `static delegate* unmanaged[Cdecl]<byte*,int,byte,void> LogFn` —— 它是一个**静态字段**（类型级），由 `SetLog`(`:31-34`) 从 `nint InLogFn` 强制转换而来，指向 C++ 的 `FScriptLog::Log`（静态函数）→ 无托管生命周期问题 ✅。

**(6) 字符串所有权总结**：UTF-8/UTF-16 的**分配方**始终是"调用方提供缓冲"或"被调方返回句柄由调用方 `Free`"，**没有"被调方分配、调用方释放"的裸指针跨边界情形** ✅。唯一的隐患是 F-ABI-001（缓冲协议算错）。

---

## 6. 平台分支审查汇总

### 6.1 平台宏扫描结果

`grep` 覆盖 `PLATFORM_WINDOWS / PLATFORM_LINUX / PLATFORM_MAC / PLATFORM_MAC_X86 / PLATFORM_MAC_ARM64 / PLATFORM_ANDROID / PLATFORM_IOS / PLATFORM_64BITS / PLATFORM_32BITS / WITH_EDITOR / WITH_EDITORONLY_DATA / WITH_MONO / WITH_CORECLR / WITH_LEANCLR / WITH_BINDING / UE_TRACE_ENABLED / NO_LOGGING`。

**结果**：`PLATFORM_64BITS` / `PLATFORM_32BITS` / `UE_SERVER` / `WITH_HOT_RELOAD` / `IS_PROGRAM` / `WITH_GAMEPLAY_TAGS` / `WITH_DEV_AUTOMATION_TESTS` 在插件自身代码里 **0 命中**（`WITH_GAMEPLAY_TAGS` 相关的只有 `GAMEPLAY_TAGS_NAME` 字符串常量，`Macro.h:35`）。因此"32/64 位分支"这一维度在本项目中**没有显式分支**——全部依赖 UE 的 `int32/int64/UPTRINT/PTRINT` 类型别名，这一点是好的。

### 6.2 逐条 `#if` 链的平台覆盖与语义等价性

| 位置 | 分支 | 是否有 `#else` | 语义等价 | 结论 |
|---|---|---|---|---|
| `Macro.h:85-91` `LIB_HOSTFXR` | WINDOWS / LINUX / MAC_X86+MAC_ARM64 | ❌ **无** | — | Android/iOS 上未定义 → 若 `WITH_CORECLR` 被打开则**编译失败** → **F-ABI-012** |
| `Macro.h:89` `PLATFORM_MAC_X86 \|\| PLATFORM_MAC_ARM64` | — | — | — | 用了 `PLATFORM_MAC_X86`/`PLATFORM_MAC_ARM64` 而非 `PLATFORM_MAC`；`FMonoFunctionLibrary.cpp:27-30/53-58` 同样区分 x86_64/arm64 → 与 UE 的 `PLATFORM_MAC` 语义一致（UE 定义 `PLATFORM_MAC` 为"任一种 Mac"，这两个宏更细）✅ |
| `FMonoFunctionLibrary.cpp:19-31` `GetMonoDirectory` | WINDOWS / ANDROID / IOS / LINUX / MAC_X86 / MAC_ARM64 | ❌ **无** | — | `FString::Printf` 有 2 个 `%s`，无匹配分支时**参数列表为空** → 编译错误（比静默好，但仍是"未覆盖平台"）→ P2 |
| `FMonoFunctionLibrary.cpp:41-59` `GetLibDirectory` | 同上一族 | ❌ **无** | — | `FString::Printf` 有 3 个 `%s` + `MONO_CONFIGURATION` → 无匹配分支时编译错误 → P2 |
| `FLeanCLRFunctionLibrary.cpp:9-25` `GetLeanCLRDirectory` | WITH_EDITOR / ANDROID / IOS / **`#else`** | ✅ | ✅ | 唯一写了 `#else` 的目录函数（`FPlatformProcess::ExecutablePath()`）→ **可以作为其余函数的模板** |
| `FLeanCLRFunctionLibrary.cpp:30-40` `GetLibDirectory` | ANDROID / IOS / **`#else`** | ✅ | ✅ | 同上 |
| `FCoreCLRFunctionLibrary.cpp:8-16` `GetCoreCLRDirectory` | WITH_EDITOR / **`#else`** | ✅ | ⚠ | `#else` 分支在打包下指向**可执行文件目录**，与编辑器分支的插件目录不是同一个位置 → **F-ABI-012** |
| `UnrealCSharpEditorSetting.cpp:144-240` `GetDotNetPathArray` | WINDOWS / MAC | ❌ **无** | — | **函数体为空（非 void 无 return，UB）** → **F-ABI-011** |
| `FUnrealCSharpFunctionLibrary.cpp:52-56` `GetDotNet` | WINDOWS / **`#else`** | ✅ | ❌ **不等价** | `#else` 返回 **macOS** 路径，Linux 上错误 → **F-ABI-011** |
| `FUnrealCSharpFunctionLibrary.cpp:28-31`（`UE_SET_HANDLE_INFORMATION` + WINDOWS） | 嵌套 `#if` | — | — | 只包了 `AllowWindowsPlatformTypes.h`（本身是空 include，`FUnrealCSharpFunctionLibrary.cpp:29-30` 之间没有任何内容）→ 注释性代码，P3 |
| `FUnrealCSharpFunctionLibrary.cpp:1031-1036` `GetFullInteropPublishPath` | `!WITH_EDITOR && WITH_CORECLR` / **`#else`** | ✅ | ❌ **不等价** | 编辑器与打包返回**不同的 Interop.dll 位置** → **F-ABI-012** |
| `FCSharpEnvironment.cpp:18-30 / 101-142` 信号处理器 | MAC / **`#else`** | ✅ | ❌ **不等价** | Mac 分支保存并恢复原 `sigaction`，`#else` 只 `signal()` → **F-ABI-009** |
| `FCSharpEnvironment.cpp:112-115` `SIGBREAK` | WINDOWS 内嵌 | — | — | 正确（`SIGBREAK` 仅 Windows 有）✅ |
| `FMonoDomain.cpp:57-98` iOS AOT 初始化 | PLATFORM_IOS / 非 iOS（`#else`）/ WINDOWS / LINUX | ✅ | ⚠ | iOS 分支做 `mono_jit_set_aot_mode(MONO_AOT_MODE_INTERP)` + `mono_dllmap_insert(..., "__Internal", ...)`（:63-72）——**这是唯一体现 iOS `__Internal` 约定的地方** ✅；但 `DOTNET9` 下 Windows 才设 `NATIVE_DLL_SEARCH_DIRECTORIES`（:79-90），Linux 只设 `DOTNET_SYSTEM_GLOBALIZATION_INVARIANT`（:92-94），macOS **什么都不做** → 三平台初始化不等价（macOS 上 `NATIVE_DLL_SEARCH_DIRECTORIES` 缺失，本机库靠默认搜索路径）→ P2 |
| `FMonoDomain.cpp:62/71` `#if !DOTNET9` + `#if DOTNET8` | — | — | — | `.NET` 版本分支的 `mono_dllmap_insert` 只对 iOS 生效（外层 `#if PLATFORM_IOS`）✅ |
| `FUnrealCSharpFunctionLibrary.cpp:1074-…` 大量 `WITH_EDITOR` 块 | WITH_EDITOR | — | — | `GetScriptDirectory`/`GetCodeAnalysis*`/`GetSourceGeneratorPath` 等只在编辑器编译 ✅ |
| `FClassReflection.cpp` / `FReflectionRegistry.cpp` 大量 `#endif`（各 600–2400 行区间） | 主要是 `UE_F_*` 版本宏（来自 `UEVersion.h`） | — | — | 属"UE 版本兼容"而非"平台兼容"，不在此表计分 |

### 6.3 Runtime 模块在 Shipping 下是否引用编辑器符号

- `Source/UnrealCSharp/UnrealCSharp.Build.cs:50-58`：`UnrealEd` 在 `if (Target.bBuildEditor)` 内 ✅。
- `Source/UnrealCSharpCore/UnrealCSharpCore.build.cs:47-65, 81-90`：`DirectoryWatcher`/`BlueprintGraph`/`UnrealEd`/`UMGEditor` 全部在 `if (Target.bBuildEditor)` 内 ✅。
- `Source/UnrealCSharpCore/UnrealCSharpCore.build.cs:67-79`：`Slate`/`SlateCore` 在**无条件**的 `PrivateDependencyModuleNames` 里 —— **这不是缺陷**（`Slate`/`SlateCore` 在 UE 里是 Runtime 模块，Shipping 可用）。
- 源码侧：`Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:11-14` 的 `#include "Editor.h"`/`"Engine/Blueprint.h"` 在 `#if WITH_EDITOR` 内 ✅；`FUnrealCSharpFunctionLibrary.cpp:8-12` 同理 ✅；`FTypeBridge.cpp:7-9` 是 `UE_F_OPTIONAL_PROPERTY`（版本宏）✅。
- grep `UnrealEd|AssetRegistry|FBlueprintSupport|Editor.h` 在 Runtime 模块的**头文件**里：`FTypeBridge.cpp:418-423` 用了 `FBlueprintSupport::IsClassPlaceholder`，**在 `#if WITH_EDITOR` 内** ✅（:418 是 `#if WITH_EDITOR`，:423 是 `#endif`）。

**结论：未发现 Runtime 模块在 `WITH_EDITOR` 之外引用编辑器符号的实例**，Shipping 链接风险在本报告范围内**未发现** P0/P1。

### 6.4 动态库加载：各平台库名与路径

| 平台 | 库名常量 | 定义 | 加载方式 | C# 侧库名 | 结论 |
|---|---|---|---|---|---|
| Windows | `hostfxr.dll` | `Macro.h:86` | `FPlatformProcess::GetDllHandle(*FCoreCLRFunctionLibrary::GetHostFxrPath())`（`FCoreCLRDomain.cpp:45`） | `"UnrealCSharp"`（`LogBridge.cs:27`、生成器 `:911`） | ✅ 路径绝对 |
| Linux | `libhostfxr.so` | `Macro.h:88` | 同上 | 同上 | ✅ |
| macOS | `libhostfxr.dylib` | `Macro.h:90` | 同上 | 同上 | ✅ |
| Android | **未定义** | — | — | — | ❌ **F-ABI-012**（CoreCLR 应被禁止） |
| iOS | **未定义**；Mono 用 `"__Internal"` dllmap（`FMonoDomain.cpp:63-72`） | — | — | — | ⚠ Mono 侧正确；CoreCLR 侧见 F-ABI-012 |
| — | `DEFAULT_PUBLISH_DIRECTORY = "Script"`（`Macro.h:95`） | — | `FPaths::ProjectContentDir() / "Script"` | `<HintPath>..\..\Content\Script\Interop.dll</HintPath>`（`Script/Shared.props:23`） | ✅ 一致（编辑器下）；打包下见 F-ABI-012 |

**硬编码 `.dll` 的地方**（`grep -n 'TEXT(".dll")\|DLL_SUFFIX'`）：
- `Macro.h:75` `DLL_SUFFIX = ".dll"` —— 用于 **C# 程序集**（`UE.dll`/`Game.dll`/`Interop.dll`），**与平台无关是正确的**（C# 程序集在 Linux/macOS 上也叫 `.dll`）✅；但同一个常量也用在 `GetFullInteropPublishPath`（`:1033/1035`）上，那里也是 C# 程序集 → 正确。
- `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:816` `NativeModuleName = "UnrealCSharp"` —— **这是唯一的"本机库名"硬编码**，且**没有平台分支、没有 `"__Internal"`**。在 LeanCLR 的自有 pinvoke 分派下模块名很可能不参与解析（LeanCLR 侧同时存在 `Module::Function` 与裸 `Function` 两种注册键，`Source/ThirdParty/LeanCLR/src/runtime/pinvokes/coreclr_qcall.cpp` 里两种都有；插件注册的是裸名 `Method.GetMethod()`），因此**在 LeanCLR 下不会触发 `DllNotFoundException`**；但一旦将来改用标准 CLR 的 P/Invoke 解析（或 LeanCLR 改为以模块名匹配），Windows 上模块实际叫 `UnrealEditor-UnrealCSharp.dll`/`UnrealCSharp-Win64-Shipping.dll`，裸名 `"UnrealCSharp"` **可能解析失败**。→ 记为 P2 观察项（`F-ABI-029`，见下）。

### [F-ABI-029] C# 侧唯一的本机库名硬编码为 `"UnrealCSharp"`，无平台分支、无 `"__Internal"`

- **类别**: 平台兼容
- **严重度**: P2（**后端门控**：LeanCLR 分支当前**生效**（`WITH_LEANCLR=1`）；`__Internal`/真实文件名两条结论仅在切到 CoreCLR/标准 P/Invoke 解析时生效——见 §9 第 10 条）
- **文件**: `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:816`、`:911`；`Script/Interop/Bridge/LogBridge.cs:27`
- **复核结论**: 确认（唯一硬编码本机库名 `"UnrealCSharp"` 在 :816 的定义与 :911 的使用点均逐字核对无误；`LogBridge.cs:27` 的 `[DllImport("UnrealCSharp", CallingConvention = Cdecl)]` 是 C# 侧第二处同名硬编码）
- **可达性**: 活跃（`WITH_LEANCLR` 分支当前生效，生成物即 `[DllImport("UnrealCSharp", …)]`）；"`__Internal`/真实文件名"两条后续结论仅在切到 CoreCLR/标准 P/Invoke 解析时才有意义 → 潜伏
- **复核证据**: `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:814-816`（第一手读取：`private const string NativeModuleName = "UnrealCSharp";`）、`:910-912`（`[DllImport("{NativeModuleName}", CallingConvention = CallingConvention.Cdecl)]`）；`Script/Interop/Bridge/LogBridge.cs:27-28`
- **级别变动**: 无（P2 维持）
- **函数**: `LibraryBridgeGenerator.EmitBridges`、`LogBridge.LogLeanCLR`
- **置信度**: 中（LeanCLR 的 pinvoke 键匹配规则未第一手确认，见 §9）

**现状（代码事实）**
```csharp
// Script/SourceGenerator/UnrealTypeSourceGenerator.cs:816
        private const string NativeModuleName = "UnrealCSharp";
// :910-912
                    "#if WITH_LEANCLR\n" +
                    $"\t\t[DllImport(\"{NativeModuleName}\", CallingConvention = CallingConvention.Cdecl)]\n" +
                    $"\t\t{accessibility} static extern unsafe partial {returnType} {method.Name}({parameters});\n" +
```

**问题**
1. iOS 静态链接场景要求 `DllImport("__Internal")` 或 `DllImport("__Internal", EntryPoint=...)`；当前没有该分支。虽然 iOS 上默认后端是 Mono（`UnrealCSharpCore.build.cs:263-267`）而不是 LeanCLR，属于"当前不可达"，但一旦 LeanCLR 支持 iOS（它在 `Source/ThirdParty/LeanCLR` 里带 AOT 相关代码），这条路径就会失败。
2. Windows 上 UE 插件模块的实际文件名带前缀/目标后缀，裸名 `"UnrealCSharp"` 依赖"运行时已在进程内加载了该模块且 DllImport 能按模块名找到它"这一前提。**实测产物名（`Get-ChildItem` 枚举，第一手）**：`Plugins/UnrealCSharp/Binaries/Win64/` 下是 `UnrealEditor-UnrealCSharp.dll`、`UnrealEditor-UnrealCSharp-Win64-Debug.dll`；递归搜索 `UnrealCSharp.dll`、`libUnrealCSharp.so`、`libUnrealCSharp.dylib` **零命中**。也就是说该字面量在任何平台上都**不是**真实文件名。

**为什么现在还能工作（这条把 §9 的原有存疑项解决了）**
LeanCLR 的 P/Invoke 实现**不加载任何动态库**：它把 `dll_name_no_ext` 当作内存注册表的**键前缀**，先按 `"[" + dll_name_no_ext + "]" + function_name` 查，失败再按裸 `function_name` 查。取证位置是 `Source/ThirdParty/LeanCLR/src/runtime/vm/pinvoke.cpp:53-81`；我第一手核对到的旁证是 `Source/ThirdParty/LeanCLR/src/runtime/vm/pinvoke.h:42-45`（`register_pinvoke(const char*, PInvokeFunction, PInvokeInvoker)` / `get_pinvoke(const char*)` / `get_pinvoke_function(const char* dll_name_no_ext, const char* function_name)` 三种接口并存）与 `pinvokes/coreclr_qcall.cpp` 中同时存在 `"[kernel32.dll]GetLastError"`、`"Kernel32::GetLastError"`、`"GetLastError"` 三种注册形态。插件注册的是裸名（`FLeanCLRDomain.cpp:870-873` 用 `Method.GetMethod()`），因此**裸名回退分支命中，模块名被忽略** → 不会 `DllNotFoundException`。
**结论**：`"UnrealCSharp"` 在 LeanCLR 下不是文件名而是"一个恰好与模块同名的、无意义的键前缀"；在 iOS/AOT 或任何改为标准 P/Invoke 解析的组合下则会立即失效。

**建议**
- 在代码注释里写清"模块名在 LeanCLR 下不参与解析，实际按方法名匹配 `register_pinvoke`"，避免后人误以为它必须等于真实文件名；
- 加平台分支以防未来切换到标准解析：
  ```csharp
  #if IOS
      private const string NativeModuleName = "__Internal";
  #else
      private const string NativeModuleName = "UnrealCSharp";
  #endif
  ```
- 更彻底的做法：显式注册带模块前缀的键（LeanCLR 已支持 `Module::Function` 形态），把"模块名语义"变成显式契约，而不是依赖它与模块名"碰巧一致"。

**验证方式**
- grep：`grep -rn "NativeModuleName\|DllImport" Script/`（确认只有这 3 处，且 `"__Internal"` 在 `Script/**` 下 0 命中）。
- 用例：把 LeanCLR 的 pinvoke 查找改成严格按 `Module::Function` 匹配，观察是否立刻 `DllNotFoundException`/`pinvoke_entry_not_found`（这也顺带验证了当前依赖的是哪条回退分支）。

---

### [F-ABI-039] `FCoreCLRLog::ErrorWriter` 的平台链缺 `#else`：未覆盖平台上 hostfxr 的错误文本被静默丢弃

- **类别**: 平台兼容
- **严重度**: P3（**后端门控**：`FCoreCLRLog.cpp` 整体在 `#if WITH_CORECLR` 内（:2），当前工程 ini 为 `WITH_CORECLR=0`，本条当前未参与编译）
- **复核结论**: 确认（平台链 `#if PLATFORM_WINDOWS / #elif PLATFORM_MAC_ARM64 || PLATFORM_MAC_X86 || PLATFORM_LINUX` 之后**没有 `#else`**，逐字核对无误；另有 `#if !NO_LOGGING` 外层门控）
- **可达性**: 潜伏（文件整体在 `#if WITH_CORECLR` 内且当前 `WITH_CORECLR=0`；此外还要同时不满足 Windows/Apple/Linux 才丢日志）
- **复核证据**: `Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRLog.cpp:1-15`（第一手读取全文：`:2 #if WITH_CORECLR`、`:7 #if !NO_LOGGING`、`:8 #if PLATFORM_WINDOWS`、`:10 #elif PLATFORM_MAC_ARM64 || PLATFORM_MAC_X86 || PLATFORM_LINUX`、`:12 #endif`）
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRLog.cpp:5-14`
- **函数**: `FCoreCLRLog::ErrorWriter(const char_t*)`
- **置信度**: 高（第一手读取）

**现状（代码事实）**
```cpp
// Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRLog.cpp:5-14
void FCoreCLRLog::ErrorWriter(const char_t* InMessage)
{
#if !NO_LOGGING
#if PLATFORM_WINDOWS
	UE_LOG(LogUnrealCSharp, Error, TEXT("%s"), InMessage);
#elif PLATFORM_MAC_ARM64 || PLATFORM_MAC_X86 || PLATFORM_LINUX
	UE_LOG(LogUnrealCSharp, Error, TEXT("%hs"), InMessage);
#endif
#endif
}
```
（附带说明：这里的格式串与实参**没有**错配——`char_t` 在 Windows 为 `wchar_t`、在其它平台为 `char`，所以 `%s`/`%hs` 的选择本身是正确的。）

**调用上下文**
`FCoreCLRDomain::RegisterErrorWriter()`（`Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRDomain.cpp:272-279`，在 `Initialize()` 的 `:60` 调用）把它注册给 `hostfxr_set_error_writer`。

**问题**
在 `PLATFORM_WINDOWS`/`PLATFORM_MAC_ARM64`/`PLATFORM_MAC_X86`/`PLATFORM_LINUX` 之外的平台（tvOS/visionOS/Switch/FreeBSD/WASM 等，只要 `WITH_CORECLR=1`）上函数体为空：hostfxr 初始化失败时产生的**全部错误文本被丢弃**，而 `FCoreCLRDomain::Initialize()` 只剩 `:90-104` 的裸错误码分支可诊断 → 现场信息丢失。同时 `InMessage` 未使用会触发 `-Wunused-parameter`。

**建议**
```cpp
#else
	UE_LOG(LogUnrealCSharp, Error, TEXT("%hs"), InMessage);   // 非 Windows 的通用写法
#endif
```
或与 F-ABI-012 一起在 `#else` 里 `#error` 明确不支持。

**验证方式**
- `grep -n -A4 "PLATFORM_WINDOWS" Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRLog.cpp` 检查是否有 `#else`。

---

### [F-ABI-040] `CopyValue` 用属性 `ElementSize` 而非对齐要求分配缓冲（仅在 `alignment > 16` 的类型上有实际风险）

- **类别**: 未定义行为（潜在）
- **严重度**: P3
- **复核结论**: 确认（`CopyValue` 用 `GetElementSize()/ElementSize` 直接 `FMemory::Malloc`，随后 `InitializeValue` + `CopySingleValue`，逐字核对无误）
- **可达性**: 活跃（复合属性描述符的 `CopyValue` 由反射/容器路径调用）
- **复核证据**: `Source/UnrealCSharp/Public/Reflection/Property/TCompoundPropertyDescriptor.inl:28-48`（第一手读取：`:33-41 FMemory::Malloc(GetElementSize()/ElementSize)`、`:43 InitializeValue(Value)`、`:45 CopySingleValue(Value, InAddress)`）、`:50-53 GetBufferSize() = sizeof(void*)`
- **级别变动**: 无（P3 维持；仅在 `alignof(T) > 16` 的类型上才有实际风险，本插件当前无此类绑定）
- **函数**: `TCompoundPropertyDescriptor<T>::CopyValue(const void*) const`
- **置信度**: 低（未构造出触发用例；且 UE 的 `FMemory::Malloc` 默认返回 16 字节对齐，对绝大多数属性类型已足够）

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Public/Reflection/Property/TCompoundPropertyDescriptor.inl:33-48
	virtual auto CopyValue(const void* InAddress) const -> void* override
	{
		const auto Value = static_cast<void*>(static_cast<uint8*>(FMemory::Malloc(
#if UE_F_PROPERTY_GET_ELEMENT_SIZE
			Super::Property->GetElementSize()
#else
			Super::Property->ElementSize
#endif
		)));

		Super::Property->InitializeValue(Value);

		Super::Property->CopySingleValue(Value, InAddress);

		return Value;
	}
```

**调用上下文**
`PROCESS_RETURN()`（`Source/UnrealCSharp/Public/Macro/FunctionMacro.h:143-145`）与 `PROCESS_OUT()`（`:128-130`）在复合返回/出参时调用 `CopyValue`，即**每次带 `FStructProperty` 返回值的 C#→UE 调用**都会走到这里。

**问题**
用 `ElementSize` 申请缓冲只保证**尺寸**，不保证**对齐**。若某属性类型的 `GetMinAlignment()` 大于 `FMemory::Malloc` 的保证对齐（UE 默认 16），`InitializeValue`/`CopySingleValue` 会在一个对齐不足的地址上构造对象 → 对该类型的对齐访问是 UB（在开启 `-fsanitize=alignment` 或使用对齐指令如 `movaps` 的代码路径上会直接崩）。
缓解因素（这也是我把本条定为 P3 且置信度标低的原因）：UE 的 `FMemory::Malloc` 默认返回至少 16 字节对齐，而绝大多数属性类型的对齐 ≤ 16；只有显式 `alignas(32/64)` 的自定义结构体才会触发。同族代码里 `FOptionalHelper.cpp:21` 的写法更稳妥，可作为统一模板。

**建议**
```cpp
		const auto Alignment = FMath::Max<int32>(16, Super::Property->GetMinAlignment());
		const auto Value = FMemory::Malloc(Size, Alignment);
```
（需确认该属性 API 在当前引擎版本下的可用性；`FProperty::GetMinAlignment()` 在 UE5 存在。）

**验证方式**
- grep：`grep -rn "FMemory::Malloc(" Source/UnrealCSharp -r`，逐个检查是否存在"申请后按类型对齐访问"的模式。
- 用例：注册一个 `alignas(32)` 的 `USTRUCT` 并让它作为 C#→UE 调用的返回值，在 `-fsanitize=alignment`（Linux）下运行。

---

> ☞ **排序说明**：§3 内的 Finding 严格按 P0 → P3 排列，唯一例外是 **F-ABI-006**（初稿定 P1、取证后改判 P2，为保留修订痕迹而留在 P1 区块，**按 P2 计分**）。另有 3 条 Finding 因主题归属放在 §6 之后：`F-ABI-029`（P2，"C# 侧唯一的本机库名硬编码"，紧接 §6.4 的动态库加载表）、`F-ABI-039`（P3，`FCoreCLRLog` 平台链，与 §6.2 同族）、`F-ABI-040`（P3，`CopyValue` 对齐问题，主题上更接近属性描述符专责报告）。
>
> **发现数汇总（全文 · 复核后）**：P0 × **0**、P1 × **6**（005/009/011/032/033/034）、P2 × **23**（001/002/003/004/006/007/008/010/012/013/014/015/016/017/018/020/021/029/030/031/035/036/037）、P3 × **9**（019/023/024/025/026/027/028/039/040）、**撤销（非缺陷）× 1**（022），共 **39** 条。
> 原文的分布为 P0 × 5（001/002/003/004/030）、P1 × 11、P2 × 17、P3 × 6；级别变动：**P0 全部清零**（5 条 P0 逐条终裁见 §3 各条 `复核结论`）、**001/030 由 P1 降 P2、004 由 P1 降 P2、019 由 P2 降 P3、022 撤销**。

---

## 7. 平台兼容"待补齐/只在 Windows 验证过"清单

按"只在 Windows 验证过 + 其它平台明确会出问题"的程度排序（每行给出**哪个平台 + 什么操作 + 什么后果**）：

| # | 平台 | 操作 | 后果 | 关联 |
|---|---|---|---|---|
| 1 | **Linux / macOS** | 启动插件（Mono 后端），行走 `FMonoDomain::Initialize` → `NATIVE_DLL_SEARCH_DIRECTORIES` | `TCHAR_TO_ANSI` 使含中文/非 ANSI 的插件路径变 `????` → mono 找不到 `mono-2.0-sgen.dll` 等本机库 | F-ABI-010 |
| 2 | **Linux** | 打开编辑器设置面板的 DotNetPath 下拉 / 编译 C# | `GetDotNetPathArray()` 无 `return`（UB）+ `GetDotNet()` 返回 macOS 路径 → **无法编译 C#** | F-ABI-011 |
| 3 | **仅 `bTCHARIsUTF8=true`（或 4 字节 TCHAR）的 Linux / macOS** | 运行带**第三方依赖程序集**的工程（Mono 后端） | `reinterpret_cast<const char16_t*>(TCHAR*)` 在非 2 字节 `TCHAR` 下使发布目录乱码 → 依赖程序集静默加载失败。默认配置（`PLATFORM_TCHAR_IS_CHAR16=1`）下**不会**触发 | F-ABI-006（P2） |
| 4 | **Linux / Windows** | 任何 SIGSEGV/SIGFPE/SIGILL | 非 async-signal-safe 的处理器 + 不恢复原处理器 → **卡死或崩溃转储丢失** | F-ABI-009 |
| 5 | **Linux / macOS** | `-configuration=Shipping` 打包运行 CoreCLR | `hostfxr`/`CoreCLR.runtimeconfig.json` 需在**可执行文件目录**（`FCoreCLRFunctionLibrary.cpp:15`），插件不拷贝 → 运行时找不到 hostfxr | F-ABI-012 |
| 6 | **macOS** | 连续 Initialize/Deinitialize 或触发信号 | `SignalActions` 全局 `TMap` 在信号上下文被 `operator[]` 修改；第二次 `Initialize` 覆盖已保存的原处理器 | F-ABI-018 |
| 7 | **macOS** | Mono 后端启动（`DOTNET9`） | 只有 Windows 设 `NATIVE_DLL_SEARCH_DIRECTORIES`、只有 Linux 设 globalization invariant（`FMonoDomain.cpp:79-94`）→ macOS 两件都不做，行为与其余两平台不等价 | §6.2 表项 |
| 8 | **Linux / macOS** | UBT 生成 `Intermediate/UnrealCSharp_Modules.json` | `GetFiles("*.Build.cs")` 匹配不到 `UnrealCSharpCore.build.cs`/`CrossVersion.build.cs` → 生成的 JSON 与 Windows 不同 | F-ABI-020/F-ABI-021 |
| 9 | **Android / iOS** | 把 `*ScriptDomainType` 配置为 CoreCLR | `LIB_HOSTFXR` 未定义 → 编译失败 | F-ABI-012 |
| 10 | **iOS** | 使用 LeanCLR 后端 | `DllImport("UnrealCSharp")` 无 `"__Internal"` 分支 | F-ABI-029 |
| 11 | **任意平台（含 Windows）** | 类名/命名空间含**较多中文**（UTF-8 > 512B） | `TypeBridge.GetName/GetNamespace/GetFullName` 抛异常穿出 `[UnmanagedCallersOnly]` → **进程终止** | F-ABI-001 |
| 12 | **任意平台** | 任何"绑定名与 C# 声明不一致"的改动 | 空指针直呼 → 崩溃（无诊断） | F-ABI-002/F-ABI-004 |
| 13 | **任意平台** | 加载损坏/版本不匹配的 `Content/Script/*.dll` | `AssemblyLoader.LoadFromStream` 异常穿出 → **进程终止** | F-ABI-003 |
| 14 | **任意平台** | `TMap<K,V>`（K≠V）的绑定 | 泛型值类型被替换为键类型（并可能抛异常终止进程） | F-ABI-005 |
| 15 | **Windows（MSVC）** | 编译含中文注释且无 BOM 的两个文件 | C4819 警告 / 极端情况下注释吞代码 | F-ABI-022 |
| 16 | **x86-32（若仍支持）** | 任意桥接调用 | 30/44 个 `[UnmanagedCallersOnly]` 的默认约定为 Winapi（stdcall）而 C++ 侧是 cdecl → 栈失衡 | F-ABI-013 |
| 17 | **LeanCLR 后端** | 用户自定义绑定含 `float`/`double`/`long`/结构体参数或返回 | `PInvoke_Classify` 返回 `NotImplemented` → 运行时调用失败，与 Mono/CoreCLR 行为不一致 | F-ABI-014 |
| 18 | **LeanCLR 后端** | 托管侧抛异常 | `Bridge_Invoke` 吞掉结果 → 表现为"类/方法不存在" | F-ABI-015 |
| 19 | **任意平台（含 Windows）** | 属性元数据行不含 `\|`（Weaver 与解析器版本错配、或 `__X_Attrs` 被手工编辑） | `Utils.cs:313` 的 `Segments[1]` 越界 → 异常穿出 `[UnmanagedCallersOnly]` → **进程终止** | F-ABI-030 |
| 20 | **任意平台** | 反复调用"有输入、无返回值"的 UE 函数（`Call2` 等 5 条路径） | 每次泄漏一块 `ParmsSize` 参数缓冲，且从不调用属性析构链（叠加 F-ABI-032）→ 内存单调增长 | F-ABI-031 / F-ABI-032 |
| 21 | **任意平台** | `Intermediate/UnrealCSharp_Modules.json` 被截断或写坏（构建中断、磁盘满、手工编辑） | `FJsonSerializer::Deserialize` 返回值被忽略 → `JsonObject->Values` 空指针解引用 → **编辑器崩溃** | F-ABI-033 |
| 22 | **任意平台** | 从非主线程调用 `SynchronizationContext.Send` | 不等待 + 句柄立即 `Dispose` → `Send` 同步语义被破坏（偶发顺序错误） | F-ABI-034 |
| 23 | **Linux / macOS** | 首次生成 C# 解决方案后 `dotnet build` | `.csproj`/`.props`/`.sln` 里硬编码 `\` → `<OutputPath>` 变成含反斜杠的目录名 → `Content/Script/` 下找不到 `UE.dll` | F-ABI-036 |
| 24 | **tvOS / visionOS / Switch / FreeBSD / WASM（`WITH_CORECLR=1`）** | hostfxr 初始化失败 | `FCoreCLRLog::ErrorWriter` 函数体为空 → 错误文本全部丢失，只剩裸错误码 | F-ABI-039 |

> **⚠️ 复核对本表的更正（必读，按此重读后果列）**
> 1. **第 11、13、19 行的"进程终止"结论仅对 Mono/CoreCLR 成立**：当前工程五平台全部 LeanCLR，桥接调用走解释器（`SCRIPT_DOMAIN_INVOKE` → `Bridge_Invoke`），托管异常被 `Unhandled_Exception` 记录后返回**零值**（`FLeanCLRDomain.cpp:306-320`、`FLeanCLRDomain.inl:58-70`）→ 实际后果是"空名/该属性元数据缺失/程序集被静默跳过"，见 F-ABI-001/003/030 的 `复核结论`。
> 2. **第 15 行（F-ABI-022）整行作废**：引擎 MSVC 工具链无条件加 `/utf-8` 并加 `/wd4819`（`VCTToolChain.cs:649-653` 同见 F-ABI-022 复核证据）→ 该情形不存在，F-ABI-022 已**撤销（非缺陷）**。
> 3. **第 16 行的"30/44"应改为"39/53"**：grep 重跑得 `[UnmanagedCallersOnly]` 共 **53** 处，其中 **39** 处不带 `CallConvs`（见 F-ABI-013 复核证据）。
> 4. **第 8 行（F-ABI-020/021）需要区分**：UBT **自身不会漏模块**（`FileSystemReference.cs:31` 的 `Comparison = OrdinalIgnoreCase`、`:123-124 HasExtension` 用它、判定点 `EpicGames.Build/System/Rules.cs:263 File.HasExtension(".build.cs")` —— 第一手复核），漏模块的**只有插件自己的** `UnrealCSharpCore.build.cs:331` 的 `DirectoryInfo.GetFiles("*.Build.cs")`（大小写敏感）。故 F-ABI-021 已降为 **P3**（命名不一致），F-ABI-020 维持 P2。

**应统一到 `FPaths`/`FCString` 的手工实现**（`grep` 依据）：
- `FMonoDomain.cpp:86/113/114` 的 `TCHAR_TO_ANSI`（应为 `TCHAR_TO_UTF8`/`FTCHARToUTF8`）→ F-ABI-010；
- `FUnrealCSharpFunctionLibrary.cpp:51-57` 的硬编码 dotnet 路径字符串 → 应用 `FPaths::Combine` + `IFileManager::Get().FileExists` 候选列表；
- `FCoreCLRFunctionLibrary.cpp:9-35`、`FMonoFunctionLibrary.cpp:10-61`、`FLeanCLRFunctionLibrary.cpp:10-40` 的 `FString::Printf(TEXT("%s/%s"))` 手工拼接 → 应统一用 `FPaths::Combine`（当前 `/` 分隔符在 Windows 上被 UE 的文件 API 容忍，但**手工拼接无法处理重复分隔符/相对路径折叠**，且 `FMonoFunctionLibrary.cpp:74` 有一处**正确的** `FPaths::ProjectContentDir() / ...` 写法可作对照）；
- `FLeanCLRDomain.cpp:721-727` 的 `FString::Printf(TEXT("%s/%s.%s"))` 同理；
- `UnrealCSharpEditorSetting.cpp:209` 的 `PathString.Replace(TEXT("\\"), TEXT("/"))` 是 Windows-only 的路径解析（配合 `:163` 的 `\r\n` 切分）→ 应改用 `FPaths::NormalizeFilename` 而非手工替换。

---

## 8. 死代码清单

| 符号 | 声明位置 | grep 命中数 | 判定 | 证据 |
|---|---|---|---|---|
| `IManagedHandleToObject` / `IManagedHandleFromObject` | `IManagedHandle.h:32-40` | 多处 | 使用中 | `grep -rn "IManagedHandleToObject" Source/` |
| ~~`FUNCTION_HANDLE_DATA_GET_OBJECT_POINTERS`~~ | ~~`FunctionMacro.h:41`~~ | 1 → **0（本轮已删除）** | **已确认死宏并删除**（原"置信度 低：可能给外部用户工程用"的猜测**已排除**：`HandleData` 属 `Interop` 运行时程序集，该 API 的语义"批量句柄→裸指针"只有 C++ 反调用路径才有意义） | `grep -rn "HANDLE_DATA_GET_OBJECT_POINTERS" Source/` → 只有 `FunctionMacro.h:41`；对应 C# `HandleData.cs:118 GetObjectPointers` 在 `Script/` 内也无调用者 → 删除后 0 命中 |
| `PLUGIN_TEMPLATE_OVERRIDE` / `PLUGIN_TEMPLATE_DYNAMIC` | `Macro.h:9/11` | 见下 | 待确认 | `grep -rn "PLUGIN_TEMPLATE_OVERRIDE\|PLUGIN_TEMPLATE_DYNAMIC" Source/` |
| `PLACEHOLDER` | `Macro.h:127` | 用作结构化绑定占位（`FClassReflection.cpp:485` 等） | 使用中 | — |
| 本专项未新增死代码判定 | — | — | — | 死代码主清单属 08-专项审计的其它报告；本报告只记录在 ABI 核对过程中**顺带发现**的条目，避免与专责报告重复 |

> 说明：本次专项的 grep 精力集中在"符号是否成对存在"上（表 C/表 D），未做全插件死代码普查——那属于 `08-专项审计` 中"死代码"专责报告的范围。上表的 `HandleData.GetObjectPointers` 是唯一在 ABI 核对过程中确认为"两侧都无调用者"的候选。**本轮已闭环**：该候选经专责报告复核确认为死桥（判据 = 无 `Op(...)` 表项，而非 C# 侧引用数 —— 见 `08-专项审计/01` 的 F-DEAD-002「判据修正」），其宏与 C# 方法**均已删除**。

---

## 9. 未覆盖/存疑项

1. **用户工程内生成的 `Script/Binding/**`**：`glob`/`pwsh` 确认插件目录下不存在该目录（它由 `ScriptCodeGenerator` 生成到 `<Project>/Script/`）。因此**引擎类型（`FVector`、`AActor`、`UClass`…）的 `__X_YImplementation` 声明本体无法核对**。缓解：这些声明的**形状**由 C++ 生成器决定（`FBindingClassGenerator.cpp:800-961` 只产出 `nint`/`void`/`byte*` 的组合，从不产出 `bool`/`float`/结构体），所以按"生成器规则"推断了它们的安全性（§4.2 规则化结论），但**没有逐条核对生成的实例**。
2. **`EntryPointNotFoundException` 的触发面**：`grep "EntryPoint=" Script/` 与 `grep 'extern "C"' Source/`（排除 ThirdParty）均 0 命中，因此本报告把该类风险重新表述为"空指针直呼"（F-ABI-002）与"注册名解析失败静默置空"（F-ABI-025）。**（此项已收敛）**：LeanCLR 的 pinvoke 查找是"先 `[dll]func`、再裸 `func`"的两级回退，插件注册的是裸名，因此 `DllImport("UnrealCSharp")` 的模块名被忽略——已在 F-ABI-029 中定稿（**该条维持 P2**，仅在 iOS/AOT 或改为标准解析时升为 P1）。仍存的残余不确定：我没有读到 `ThirdParty/LeanCLR/src/runtime/vm/pinvoke.cpp` 本身（按排除范围），该结论基于 `pinvoke.h:42-45` 的接口形态 + `coreclr_qcall.cpp` 的三种注册键形态 + 对 `pinvoke.cpp:53-81` 的第一手读取。
3. **`TCHAR` 在 Linux/macOS 上的确切宽度**：**（已解决）** 引擎位于 `<Engine 5.6 安装目录>`，第一手取证：`Unix/UnixPlatform.h:17 PLATFORM_UNIX_USE_CHAR16 1` → `:50 PLATFORM_TCHAR_IS_CHAR16 1`；`Mac/MacPlatform.h:11 PLATFORM_MAC_USE_CHAR16 1` → `:55 PLATFORM_TCHAR_IS_CHAR16 1`；`HAL/Platform.h:269-281`（`PLATFORM_TCHAR_IS_4_BYTES` 默认 0、`PLATFORM_TCHAR_IS_UTF8CHAR = USE_UTF8_TCHARS`）。结论：**默认配置下三平台 `TCHAR` 都是 2 字节 UTF-16**，故 F-ABI-006 已从 P1 下调为 P2 并改述（仅 `bTCHARIsUTF8`/4 字节 `TCHAR` 配置下失效）。
4. **`FBoolProperty::ElementSize` 的实际取值**：**（部分解决）** 引擎侧 `Engine/Source/Runtime/CoreUObject/Private/UObject/PropertyBool.cpp:99 SetElementSize(InSize)`、`:100 FieldSize = (uint8)GetElementSize()`、`:122/:133/:190/:217` 的 `switch(GetElementSize())`（含 `UE_LOG(LogProperty, Fatal, TEXT("Unsupported FBoolProperty %s size %d."))`）表明 **`FBoolProperty::ElementSize` 可以是 1/2/4/8 中的多种**，不是一个固定值。由此产生的连锁影响：`FGeneratorCore::GetBufferSize`(`FGeneratorCore.cpp:469-478`) 会按实际 `ElementSize` 生成 `stackalloc byte[n]`，而 `GetBufferCast`(:460) 返回 `TEXT("bool")` → 生成的 C# 以 **1 字节** 读/写该缓冲（`*(bool*)Buffer`）；C++ 侧 `FBoolPropertyDescriptor::Set`(`FBoolPropertyDescriptor.cpp:5`) 也只读 1 字节。对 `bool` 而言低字节即真值，因此**当前不会读错**；但"缓冲长度按 `ElementSize`、读写宽度按 1 字节"这一错配在 `ElementSize > 1` 时留下 3~7 字节未写区域（值语义上无害，`SetPropertyValue` 只取低位）。**建议由属性描述符专责报告**确认 `ElementSize` 的取值来源与是否有 `ElementSize > 1` 的真实属性类型进入过这条路径。
5. **`FClassReflection::ParseDescriptor/Methods` 的槽位语义**：我只核对了"槽位数"，未逐条核对"槽位含义"（哪个索引对应哪个输出数组）。`Utils.cs:646-676` 的 `OutBuffer[0..14]` 与 `FClassReflection.cpp:160-195` 的 `InParams[1..15]` 之间的**索引映射**未逐项验证——若这里错位，症状是"属性/方法元数据串位"，属 F-ABI-008 的同一族风险但未被本报告的核对覆盖到。（另一口径给出的槽位数 16/9/4/15 与本文核对的 15/8/3/14 因是否计入 `InParams[0]`/`OutBuffer` 起始位置而口径不同，**两者一致**。）
6. **`HandleData` 的引用计数与泄漏**：`Alloc`/`FreeImplementation`(`HandleData.cs:27-78`) 的计数逻辑我只做了阅读，未做"Alloc 次数 vs Free 次数"的全局配对统计（F-ABI-028 只是其中一个入口的观察）。属"内存泄漏"专责报告范围。**另有一条相关结论值得记录**：`GetHandle`(`HandleData.cs:80-93`) 在 `Count == 0`（已 Free）时仍会返回**旧的句柄值**（`FreeImplementation` 只从 `Handles` 移除条目，不复位 `HandleReference.Value`，`:58-74`）；经其验证**不会造成 double-free**（句柄单调递增，二次 `Free` 因表项缺失直接返回），但会让 C++ 侧拿到一个"看起来有效其实已释放"的句柄，属 P2 隐患。
7. **`PInvoke_Dispatch` 在小整数参数上的行为**：`FLeanCLRDomain.cpp:635` 把每个实参提升为 `void*`。在 x64 上等价，在 x86-32 上 cdecl 下 1/2 字节实参也会提升为 4 字节 → 也等价。因此我**没有**把它列为缺陷，只作为 P3 的契约注释建议（未单列 Finding）。若有人放宽 `PInvoke_Classify` 的白名单（例如加入 `float`），该假设立即失效——这一点已写进 F-ABI-014 的建议。
8. **`.Build.cs` 的 UBT 模块发现大小写行为**：**（已解决）** 引擎侧确认 UBT 自身**不受影响**——模块规则文件靠 `File.HasExtension(".build.cs")` 发现，而 `FileSystemReference.Comparison` 被硬编码为 `StringComparison.OrdinalIgnoreCase`（`Engine/Source/Programs/Shared/EpicGames.Core/FileSystemReference.cs:31`、`:122`，判定点 `Engine/Source/Programs/Shared/EpicGames.Build/System/Rules.cs:263`）。因此 F-ABI-021 的实际风险**仅为"命名不统一"**（UBT 不会漏模块），其危害集中在 F-ABI-020 的**插件自己的** `DirectoryInfo.GetFiles("*.Build.cs")` 上；两条的严重度据此维持 P2/P2。
9. **`Script/Shared.props` 的陈旧性**：`WITH_LEANCLR` 由 `FSolutionGenerator.cpp:207-223` 依据 `GetScriptDomainType()` 写入 `Shared.props`。若用户改了 `DefaultUnrealCSharpSetting.ini` 后**只重编 C++ 而不重新生成 C# 解决方案**，`Shared.props` 会保留旧的 `WITH_LEANCLR;` → C# 侧生成 `[DllImport]` 而 C++ 侧跑 CoreCLR → 与 F-ABI-029 同一族失败。我**没有**追踪"什么时机重新生成 Shared.props"，因此未单列 Finding，但这是一个值得补的守卫（例如在 `Shared.props` 里写入当前 `ScriptDomainType` 作为注释/属性，并在运行时不匹配时 `UE_LOG(Error)`）。
10. **配置事实（第一手复核）**：`Config/DefaultUnrealCSharpSetting.ini` 第 4–8 行：
    ```
    4: WindowsScriptDomainType=LeanCLR
    5: LinuxScriptDomainType=LeanCLR
    6: AndroidScriptDomainType=LeanCLR
    7: MacScriptDomainType=LeanCLR
    8: IOSScriptDomainType=LeanCLR
    ```
    即**本工程在全部五个平台上都使用 LeanCLR 后端**，于是 `UnrealCSharpCore.build.cs:275-306` 的 `WithDomain()` 会产出 `WITH_LEANCLR=1`、`WITH_CORECLR=0`、`WITH_MONO=0`。这一事实**直接决定哪些发现是"活"的**：
    - `WITH_LEANCLR=1` → `FLeanCLRDomain.cpp`/`FLeanCLRFunctionLibrary.cpp`/`FLeanCLRDomain.inl` 全部参与编译，**F-ABI-014、F-ABI-015 是当前配置下必然执行的代码路径**（严重度按 P2 计，但触发概率显著高于"可选后端"的默认假设）；
    - `WITH_MONO=0` → `FMonoDomain.cpp`（整个文件被 `#if WITH_MONO` 包住，:2）与 `FMonoFunctionLibrary.cpp`（:1）**不参与编译** ⇒ **F-ABI-006、F-ABI-010 在当前配置下是潜伏缺陷**（一旦切回 Mono 后端即生效）；
    - `WITH_CORECLR=0` → `FCoreCLRDomain.cpp`/`FCoreCLRFunctionLibrary.cpp`/`FCoreCLRLog.cpp` **不参与编译** ⇒ **F-ABI-012、F-ABI-039 与 F-ABI-029 的 CoreCLR 分支在当前配置下是潜伏缺陷**（F-ABI-029 的 LeanCLR 分支则是活的）；
    - 注意 `UnrealCSharpCore.build.cs:261-267` 的**无配置默认值**是 Android/IOS→`Mono`、其余→`CoreCLR`，与 ini 的实际值不同——即"删掉 ini 或换工程"会把活跃后端从 LeanCLR 切换到 CoreCLR/Mono，**上表所有"潜伏"项会随之变成活跃项**。这正是 F-ABI-029 第 9 条（`Shared.props` 陈旧性）最危险的组合。
11. **未独立复核的条目（列此以明示置信度边界）**：`SDynamicNewClassDialog.cpp:752/922` 的"路径仅做前缀比对 + 文本框可编辑 → `..\` 可写出项目目录"（安全类，P3 本地自伤）、`MethodBridge` 异常被吞后 `Console.Error` 在 Shipping 无输出、`HandleData.cs:112 Unsafe.As<object,nint>` 与 `bPinned` 的语义（它返回的是**托管对象引用**而非 pinned 数组数据指针，这对 LeanCLR 需要 `RtObject*` 的用法是**正确**的，故未列为缺陷）。这些条目**未被本报告计入发现数**。

