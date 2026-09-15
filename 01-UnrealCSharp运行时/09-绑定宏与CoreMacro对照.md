# 绑定宏与 CoreMacro 对照（`Source/UnrealCSharp/Public/Macro/` 全量）

> 本报告各条**严重度见各 Finding 的「复核结论」字段**（13 条：含 0 条判定为非缺陷、4 条级别调整）。
>
> **本报告的校正说明（发现编号不变）**
> - 发现总数：13（F-MACRO-001…013，编号连续、无重号；`grep '^### \[F-'` = 13，`grep '^\| F-'` = 0，本报告不使用发现总表）
> - 复核结论：确认 **9** / 部分确认 **4**（F-MACRO-001 行号偏差、F-MACRO-002 算术偏差、F-MACRO-004 计数偏差、F-MACRO-006 触发条件与诊断号偏差且原语法推理错误）/ 证伪 0 / 无法验证 0
> - 级别变动：**0 条** —— 001 P1→P2、004 P2→P3、005 P2→P3、006 P2→P3 四条调整的理由经回源码复核**成立**，维持
> - 可达性修正：**2 条** —— F-MACRO-001 活跃→**潜伏**（全工程 0 处带实参调用，已 grep 验证）、F-MACRO-008 活跃→**不可达/潜伏**（死宏 + UE 5.7 门控）
> - 非 Finding 内容修正：**15 处**（偏特化个数 12/12/10→8/9/9、`BINDING_SCRIPT_STRUCT` 57→56、`PROCESS_*` 调用 22→23 与合计 37、`TBindingClassBuilder.inl` 不是 `BINDING_CONSTRUCTOR` 的唯一调用点、`FString::Right` 算术、`BufferAllocator->Free` 裸调用 2 处→1 处、P1/P2/P3 小节归属、§"已检查维度"5 的"无裸短名"等）
> - **最重要的一处更正**：F-MACRO-006 原文"放进无花括号 `if/else` 即编译失败（实测 `error C2062`）"**被证伪**（双编译器实测通过；真实失败条件是"调用处多写分号 / 宏体自带 `;`"，诊断是 `C2181` / clang `expected expression`）
> - **"本机实测宏展开"断言：可信且可复现**（用报告点名的同一对编译器复跑，clang 硬错误与 MSVC C5103 逐字吻合，见 §0.5；唯嵌套变参宏一项为探针假阴性）





> 分析范围：`Source/UnrealCSharp/Public/Macro/`（用户侧 C++ 绑定注册宏门面，5 个头文件）+ `Source/UnrealCSharpCore/Public/CoreMacro/`（16 个头文件，对照物）
> 覆盖文件：21 个（5 + 16），全部读完
>
>
>
> **本报告的宏展开不是人工猜测**：全部使用本机真实编译器做预处理验证
> - MSVC 14.51.36231（`<VS 安装目录>\...\cl.exe`），`/EP`（传统预处理器）与 `/EP /Zc:preprocessor`（一致性预处理器）两种模式
> - Clang 20.0.0git（引擎自带第三方 `.../NNERuntimeIREE/Binaries/ThirdParty/IREE/Windows/clang++.exe`，`-E -x c++ -std=c++20`）
> - 引擎侧事实验证基于本机 UE 5.6 源码树 `<Engine 5.6 安装目录>`（`UnrealCSharpTest.uproject` 的 EngineAssociation `{79A5A541-490E-E9AC-B2ED-E1BA4060AFC1}` → `<Engine 5.6 安装目录>`）
> - 探针文件写在 `%TEMP%\a9probe\`（非工程目录），分析结束后已删除；**未修改任何插件源码**

---

## 0. 覆盖范围与阅读清单

### 0.1 我的负责文件（全部读完）

| 文件 | 行数 | 是否读完 | 备注 |
|---|---|---|---|
| `Source/UnrealCSharp/Public/Macro/BindingMacro.h` | **352** | 是 | 任务简报写的是 314 行，实测 352 行（`read` 工具输出 `(End of file - total 352 lines)`） |
| `Source/UnrealCSharp/Public/Macro/FunctionMacro.h` | 147 | 是 | |
| `Source/UnrealCSharp/Public/Macro/NamespaceMacro.h` | 3 | 是 | 只有 1 个宏 |
| `Source/UnrealCSharp/Public/Macro/PropertyMacro.h` | 5 | 是 | 只有 2 个宏 |
| `Source/UnrealCSharp/Public/Macro/SignatureMacro.h` | 28 | 是 | 12 个宏，全部是签名/实参配对 |

### 0.2 对照物：`Source/UnrealCSharpCore/Public/CoreMacro/`（全部读完）

| 文件 | 行数 | 内容性质 |
|---|---|---|
| `BindingMacro.h` | 19 | 字符串拼接宏 + `WITH_*_INFO` 编译期开关 |
| `Macro.h` | 133 | 全局字符串常量（插件名/后缀/占位符/平台库名）+ `STR`/`TEXT_STR`/`F_STRING_STR` |
| `NamespaceMacro.h` | 17 | `COMBINE_NAMESPACE` + 7 个命名空间常量 |
| `FunctionMacro.h` | 133 | P/Invoke 目标函数名字符串常量（约 100 个） |
| `PropertyMacro.h` | 7 | 3 个 C# 属性名字符串常量 |
| `BufferMacro.h` | 29 | `IN_BUFFER`/`OUT_BUFFER`/`RETURN_BUFFER` 及其 `_SIGNATURE`/`_TEXT` |
| `ClassMacro.h` | 111 | C# 类名字符串常量 + 泛型元数拼接 |
| `ClassAttributeMacro.h` | 53 | C# 特性名字符串常量 |
| `FunctionAttributeMacro.h` | 31 | 同上 |
| `GenericAttributeMacro.h` | 47 | 同上 |
| `MetaDataAttributeMacro.h` | 315 | 同上（最大） |
| `PropertyAttributeMacro.h` | 67 | 同上 |
| `AccessPrivateMacro.h` | 39 | `ACCESS_PRIVATE_*`（生成访问私有成员的桩结构） |
| `CompilerMacro.h` | 21 | `PRAGMA_DISABLE/ENABLE_DANGLING_WARNINGS` |
| `CoreCLRMacro.h` | 7 | `CORECLR_TYPE_NAME` |
| `MonoMacro.h` | 9 | `ASSEMBLY_*` |

### 0.3 为写本报告额外阅读的"消费者/被特化模板"（非我负责范围，仅作证据）

| 文件 | 读到的行范围 | 用途 |
|---|---|---|
| `UnrealCSharp/Public/Binding/Class/TBindingClassBuilder.inl` | 全部 45 行 | `BINDING_DESTRUCTOR` 的**唯一**调用点（`:21`）；`BINDING_CONSTRUCTOR` 在此**只有 1 处**（`:18`，实参是模板形参 `T`），另 90 处散在 `FRegister*.cpp`（`grep 'BINDING_CONSTRUCTOR('` 全插件 93 命中 = 2 `#define` + 1 + 90） |
| `UnrealCSharp/Public/Binding/Class/TClassBuilder.inl` | 全部 142 行 | `PREFIX_UNARY_*`(4+2=6) + `BINARY_OPERATOR`(16) 共 **22 处**调用点；`Constructor/Destructor` 形参 |
| `UnrealCSharp/Public/Binding/Class/FClassBuilder.h` | `Function/Property` 声明 | 确认 Editor/Shipping 重载 |
| `UnrealCSharp/Public/Binding/Function/TDestructorBuilder.inl` | 全部 15 行 | 确认 `TDestructorBuilder<Args...>` 接受变参模板实参 |
| `UnrealCSharp/Public/Binding/Function/TOverloadBuilder.inl` | 全部 62 行 | `BINDING_OVERLOADS` 的目标 |
| `UnrealCSharpCore/Public/Template/TFunctionPointer.inl` | 全部 17 行 | `.Value.Pointer` 联合体来源 |
| `UnrealCSharp/Public/Binding/ScriptStruct/TScriptStruct.inl` | 全部 131 行 | `BINDING_SCRIPT_STRUCT` 的 57 处调用 |
| `UnrealCSharp/Public/Binding/ScriptStruct/TBaseStructure.inl` | 1-80 / 287 | `BINDING_REMOVE_PREFIX_CLASS_STR` 的调用形态 |
| `UnrealCSharpCore/Public/Binding/TypeInfo/TName.inl` | 全部 312 行 | 宏生成特化的"竞争对象" |
| `UnrealCSharpCore/Public/Binding/TypeInfo/TNameSpace.inl` | 全部 272 行 | 同上 |
| `UnrealCSharpCore/Public/Template/TIsUStruct.inl` / `TIsScriptStruct.inl` / `TIsNotUEnum.inl` / `TIsReflectionClass.inl` | 全部（13/7/7/11） | 排斥机制核心 |
| `UnrealCSharp/Public/Binding/Core/TPropertyValue.inl` | 470-539 / 1008 | `TIsUStruct` 的另一处叠加条件 |
| `UnrealCSharp/Public/Reflection/Function/FUnrealFunctionDescriptor.inl` | 1-175、176-262（全文 262） | `PROCESS_*` 宏的 **23 处**调用点（`:19,27,37,41,51,61,63,71,75,84,88,90,110,120,132,136,146,150,161,165,167,194,241`）；同目录 `FCSharpDelegateDescriptor.inl` 另有 14 处，合计 37 处（按 grep 逐行清单复核为 37） |
| `UnrealCSharp/Public/Reflection/Function/FFunctionDescriptor.h` | 全部 34 行 | `BufferAllocator` 类型 |
| `UnrealCSharp/Private/Reflection/Property/FPropertyDescriptor.cpp` | 1-60 | `NEW_PROPERTY_DESCRIPTOR` 的 33 处调用形态 |
| `UnrealCSharp/Private/Domain/Interop/FRegisterVector.cpp` | 1-208 / 345 | 代表性调用点（含带实参的 `BINDING_FUNCTION`/`BINDING_OVERLOAD`/`BINDING_CONSTRUCTOR`） |
| `FRegisterKeys.cpp`(416) 1-60 + :117、`FRegisterWorld.cpp`(155) 1-80、`FRegisterSoftObjectPath.cpp`(83)、`FRegisterEnhancedInputComponent.cpp` 1-40 | 部分 | `BINDING_PROPERTY`/`BINDING_ENUM`/`BINDING_CLASS` 调用点 |
| `Source/CrossVersion/Public/UEVersion.h` | 相关 `#define` 行 | `UE_U_STRUCT_*`（=UE 5.7.0）与 `UE_REGULAR_UENUM_STATIC_ENUM`（=UE 5.8.0） |

### 0.4 明确未读

- `UnrealCSharpCore/Public/Binding/TypeInfo/TStaticName.inl`、`TGeneric.inl`、`TTypeInfo.inl`（`BINDING_STRUCT` 的特化目标 `TStaticName`，见 §5 存疑项）
- `FClassBuilder.inl`、`FBindingClassRegister.cpp` 的完整实现（只看了 `Property`/`Function` 的签名）
  - **补充**：`FClassBuilder.inl`（全文 56 行）已读完并用于复核 F-MACRO-007 的链路（`:3-20` `Function`、`:22-50` `Property`）；`FBindingClassRegister.cpp` 仍只读了 `:86-98` 的 `BindingProperty` 调用链。
- `Script/`（C# 侧）——宏名在 C# 里无意义，未统计
- `Source/ThirdParty/**`（规范排除）

### 0.5 宏展开"本机实测"断言的复现证据

宏展开"本机实测"的原始探针目录 `%TEMP%\a9probe\` **已删除**（实测 `Test-Path "$env:TEMP\a9probe"` = `False`），无法直接查验原始输出；因此改用报告点名的**同一对编译器**重跑，探针写在 `%TEMP%\a9r2probe\probe.cpp`（逐字节复制 `Macro/BindingMacro.h:285-306` 的宏体，未修改任何插件源码）：

```powershell
$clang = "Engine\Plugins\Experimental\NNERuntimeIREE\Binaries\ThirdParty\IREE\Windows\clang++.exe"
$cl    = "<VS 安装目录>\VC\Tools\MSVC\14.51.36231\bin\Hostx64\x64\cl.exe"
& $clang -E -x c++ -std=c++20 $probe            # 版本：clang version 20.0.0git / Target: x86_64-pc-windows-msvc
& $cl /nologo /EP /Zc:preprocessor /W4 $probe   # 版本目录：VC\Tools\MSVC\14.51.36231
& $cl /nologo /EP /W4 $probe
```

| 复现项 | 实测输出 | 与报告 §1.2(a)/F-MACRO-001 是否吻合 |
|---|---|---|
| Clang 20.0.0git，`BINDING_DESTRUCTOR(FCustomAllocator)` | `error: pasting formed '(FCustomAllocator', an invalid preprocessing token [-Winvalid-token-paste]` + `'<FCustomAllocator'`（共 6 errors） | **逐字吻合** |
| Clang 20.0.0git，`BINDING_OVERLOADS(0)` | `error: pasting formed '(0', an invalid preprocessing token` | **逐字吻合** |
| MSVC 14.51 `/Zc:preprocessor /W4`，`BINDING_DESTRUCTOR(FCustomAllocator)` | `warning C5103: pasting '(' and 'FCustomAllocator' does not result in a valid preprocessing token` ×2 | **逐字吻合** |
| MSVC 14.51 `/Zc:preprocessor /W4`，`BINDING_OVERLOADS(0)` | `warning C5103: pasting '(' and '0' …` ×1 | **逐字吻合** |
| MSVC 14.51 传统预处理器 `/EP /W4` | 无诊断，展开文本正确 | **吻合**（无诊断） |
| MSVC 14.51 传统预处理器，`BINDING_CONSTRUCTOR_BUILDER_INVOKE(FVector)`（空 `__VA_ARGS__`） | `TConstructorBuilder<FVector >`（尾随空格） | **逐字吻合**（§1.2(b) 的"传统预处理器多一个空格"成立） |

**一处未能复现的细节（不构成报告错误，但如实记录）**：最小探针里 MSVC（传统与 `/Zc:preprocessor` **两种模式**）把 `BINDING_DESTRUCTOR()` 展开结果中的**嵌套**调用 `BINDING_DESTRUCTOR_BUILDER_INVOKE()` 原样留在输出里（未递归展开），而 Clang/一致性预处理器本应展开为 `TFunctionPointer<decltype(&TDestructorBuilder<>::Invoke)>(...)`；用 `#define ABCD(...) F1<##__VA_ARGS__>` 式最小用例可定向复现（外层名与第一个内层名同首字符时偶发，形似 MSVC 宏表状态相关的假阴性）。**判定为最小探针的宏表规模差异造成的假阴性，不是插件缺陷**：本机存在成功的 Win64/MSVC 编译产物（`Plugins/UnrealCSharp/Intermediate/Build/Win64/x64/UnrealEditor/{Development,Debug}/*/Module.*.cpp.obj` 共 12 个 + `Binaries/Win64/UnrealEditor-UnrealCSharp.dll`），而 `TBindingClassBuilder.inl:21` 的 `BINDING_DESTRUCTOR()` 在每个绑定类 TU 里都会被预处理，若真不展开则构建必然失败。因此 §1.2(a) 的空实参展开文本"**在真实 TU 中成立，但未能在最小探针中逐步复现**"。

---

## 1. 模块职责与架构速览

### 1.1 两套宏的职责边界（结论：**分工清晰，不是重复实现；但存在 3 处同体重复宏**）

| 维度 | `Macro/`（本模块，5 个头文件） | `CoreMacro/`（对照物，16 个头文件） |
|---|---|---|
| 宏的形态 | **模板偏特化生成器 + 表达式生成器**（展开后是 `struct`/`lambda`/`static_cast`） | **字符串常量 + `#if` 编译期开关 + 少量表达式拼接宏** |
| 谁用 | 游戏/插件里**手写**的绑定注册代码（`FRegister*.cpp`、`TScriptStruct.inl`、`TBaseStructure.inl`、`TClassBuilder.inl`） | 整个插件的 7 个模块（反射、域、代码生成、Dynamic 类） |
| 依赖方向 | `Macro/BindingMacro.h:3-4` **包含** `CoreMacro/Macro.h`、`CoreMacro/BindingMacro.h`；`SignatureMacro.h:3` 包含 `CoreMacro/BufferMacro.h` | **不包含** `Macro/` 下任何头文件（`grep 'Macro/BindingMacro.h\|Macro/FunctionMacro.h'` 在 `UnrealCSharpCore/` 下 0 命中） |
| 典型代表 | `BINDING_CLASS`、`BINDING_SCRIPT_STRUCT`、`BINDING_ENUM`、`BINDING_FUNCTION`、`BINDING_PROPERTY`、`BINDING_CONSTRUCTOR`、`BINDING_DESTRUCTOR`、`PREFIX_UNARY_OPERATOR`、`NEW_PROPERTY_DESCRIPTOR`、`PROCESS_SCRIPT_IN` | `NAMESPACE_*`、`CLASS_*`、`FUNCTION_*`、`COMBINE_NAMESPACE`、`WITH_*_INFO`、`IN_BUFFER*`、`ACCESS_PRIVATE_*` |

**关键点：两模块之间没有任何同名宏。** 我对插件全部 507 个 `.h/.cpp/.inl`（排除 `ThirdParty/`）做了一次 `#define` 全量扫描（757 条 `#define`，18 个重名），重名项全部是 `#if/#else` 分支对（`BINDING_PROPERTY`、`BINDING_FUNCTION`、`LIB_HOSTFXR`、`VS2022`…）、UE 惯用的 `LOCTEXT_NAMESPACE`、以及各后端各自的 `SCRIPT_DOMAIN_*`。**不存在"同名不同义"的宏。**（复现脚本：正则 `^\s*#\s*define\s+([A-Za-z_]\w*)\s*(\(([^)]*)\))?\s*(.*)$`，按名字分组，比较 body 是否只有 1 种。）

> **口径补充**：`507 个文件` 这个数**无法用可用工具复现**（`grep` 工具不支持排除 glob、也不统计文件数）；`757 条 #define` 的口径可复现其量级：`grep '^\s*#\s*define\s+[A-Za-z_]'` 在本插件 `Source/` 下共 **1777** 条（**含** `ThirdParty/`），其中 `Source/UnrealCSharp` **99** 条、`Source/UnrealCSharpCore` **524** 条（= 623），加上 `CrossVersion`（`UEVersion.h`/`DotnetVersion.h` 等约 105 条）与其余模块后与 757 同量级。
> **同名结论做了独立验证**（不依赖那 757 条统计）：① 在 `Source/UnrealCSharpCore` 内 grep `#define (BINDING_CLASS|BINDING_ENUM|BINDING_STRUCT|BINDING_PROPERTY|BINDING_FUNCTION|NAMESPACE_BINDING|PROCESS_RETURN|PROCESS_OUT|INITIALIZE_VALUE|IN_END|FUNCTION_PLUS|FUNCTION_MINUS|FUNCTION_DESTRUCTOR|FUNCTION_GET_SUBSCRIPT|FUNCTION_SET_SUBSCRIPT|FUNCTION_LOGICAL_NOT|FUNCTION_UNARY_PLUS|FUNCTION_UNARY_MINUS|FUNCTION_COMPLEMENT)\b` → **0 命中**；② `CoreMacro/FunctionMacro.h` 的 64 条 `FUNCTION_*` 全是 `FUNCTION_UTILS_*`/`FUNCTION_HOSTFXR_*`/`FUNCTION_TYPE_BRIDGE_*`/`FUNCTION_*_BRIDGE_*`，与 `Macro/FunctionMacro.h` 的 `FUNCTION_PLUS`/`FUNCTION_DESTRUCTOR`/`FUNCTION_GET_SUBSCRIPT` 等**名集不相交**（`CoreMacro/FunctionMacro.h:3-133`）。即"两模块无同名宏"成立。
> 另注：**文件**层面两模块确有 4 组同名头（`BindingMacro.h`/`FunctionMacro.h`/`NamespaceMacro.h`/`PropertyMacro.h` 各一份；`CoreMacro/Macro.h` 又与 `Macro/` 目录名相近），`#include "Macro/FunctionMacro.h"` 与 `"CoreMacro/FunctionMacro.h"` 只差 4 个字符而内容完全不同 —— 这是 F-MACRO-009 的同类命名风险，但**不是宏名冲突**。

**但存在 3 个"同体异名"宏（可优化）**，都在 `CoreMacro/` 内，函数体逐字节相同：

```cpp
// CoreMacro/BindingMacro.h:3
#define BINDING_COMBINE_CLASS(A, B) FString::Printf(TEXT("%s.%s"), *A, *B)
// CoreMacro/NamespaceMacro.h:3
#define COMBINE_NAMESPACE(A, B) FString::Printf(TEXT("%s.%s"), *A, *B)
// CoreMacro/ClassMacro.h:5
#define COMBINE_FULL_NAME(A, B) FString::Printf(TEXT("%s.%s"), *A, *B)
```

使用量差异极大：`COMBINE_NAMESPACE` 308 次、`COMBINE_FULL_NAME` 16 次、`BINDING_COMBINE_CLASS` **1 次**（`grep '\bBINDING_COMBINE_CLASS\b'` = 2，其中 1 处是定义）。三者可合并为 1 个（见 F-MACRO-009）。

**另一处重复定义：同一个字符串常量 `"Binding"` 有两个宏名**

```cpp
// CoreMacro/Macro.h:49
#define BINDING_NAME FString(TEXT("Binding"))
// UnrealCSharp/Public/Macro/NamespaceMacro.h:3
#define NAMESPACE_BINDING FString(TEXT("Binding"))
```

`NAMESPACE_BINDING` 使用 35 次，`BINDING_NAME` 只使用 1 次 —— 后者是前者的"孤儿副本"，且分属两个模块。

### 1.2 逐宏展开（真实预处理器输出）

所有展开均为编译器实际输出（MSVC 与 Clang 结果**逐字符一致**，仅空格差异），下面按宏分组给出。
> **复现说明**：这份"逐字符一致"的结论**部分可复现**。用同一对编译器复跑后，(a) 非逗号位置 `##` 的诊断（Clang 硬错误 / MSVC C5103 / MSVC 传统无诊断）与 `BINDING_CONSTRUCTOR` 空实参的 `TConstructorBuilder<FVector >` 尾随空格形态**逐字复现**；(b) 但**嵌套变参宏 `BINDING_DESTRUCTOR_BUILDER_INVOKE()` 的递归展开在最小探针里 MSVC（两种预处理器）没有做**，与"逐字符一致"矛盾。经与真实编译产物对照（本机存在成功的 Win64/MSVC 构建：`Binaries/Win64/UnrealEditor-UnrealCSharp.dll` + `Plugins/UnrealCSharp/Intermediate/Build/Win64/x64/UnrealEditor/{Development,Debug}/**/Module.*.cpp.obj` 共 12 个）判定为**最小探针的宏表规模假阴性**，全部证据见 §0.5。

#### (a) `BINDING_DESTRUCTOR` —— `__VA_ARGS__` 三处陷阱的集中体现

```cpp
// Macro/BindingMacro.h:298-306
#define BINDING_DESTRUCTOR_BUILDER_INVOKE(...) TFunctionPointer<decltype(&TDestructorBuilder<##__VA_ARGS__>::Invoke)>(&TDestructorBuilder<##__VA_ARGS__>::Invoke).Value.Pointer
#define BINDING_DESTRUCTOR_BUILDER_INFO(...) {[](){ return TDestructorBuilder<##__VA_ARGS__>::Info(); }}
#if WITH_FUNCTION_INFO
#define BINDING_DESTRUCTOR(...) BINDING_DESTRUCTOR_BUILDER_INVOKE(##__VA_ARGS__), BINDING_DESTRUCTOR_BUILDER_INFO(##__VA_ARGS__)
#else
#define BINDING_DESTRUCTOR(...) BINDING_DESTRUCTOR_BUILDER_INVOKE(##__VA_ARGS__)
#endif
```

调用点：`TBindingClassBuilder.inl:21` `TClassBuilder<T, IsProjectClass0>::Destructor(BINDING_DESTRUCTOR());`

展开（Editor，`WITH_FUNCTION_INFO=1`，实参为空，MSVC 与 Clang 一致）：
```cpp
TFunctionPointer<decltype(&TDestructorBuilder<>::Invoke)>(&TDestructorBuilder<>::Invoke).Value.Pointer,
{[](){ return TDestructorBuilder<>::Info(); }}
```
展开（Shipping，`WITH_FUNCTION_INFO=0`）：
```cpp
TFunctionPointer<decltype(&TDestructorBuilder<>::Invoke)>(&TDestructorBuilder<>::Invoke).Value.Pointer
```
`TDestructorBuilder<>` 是合法形式（`TDestructorBuilder.inl:6` 是 `template <typename... Args>`）→ **空 `__VA_ARGS__` 在这个宏上是对的**。

带实参（`BINDING_DESTRUCTOR(FCustomAllocator)`，宏签名和 `TDestructorBuilder<Args...>` 都明确支持）：
```cpp
TFunctionPointer<decltype(&TDestructorBuilder<FCustomAllocator>::Invoke)>(&TDestructorBuilder<FCustomAllocator>::Invoke).Value.Pointer,
{[](){ return TDestructorBuilder<FCustomAllocator>::Info(); }}
```
**MSVC `/Zc:preprocessor /W4`：2 条 `warning C5103`；Clang 20（无任何额外开关）：2 条硬错误**
```
warning C5103: pasting '(' and 'FCustomAllocator' does not result in a valid preprocessing token
warning C5103: pasting '<' and 'FCustomAllocator' does not result in a valid preprocessing token
```
```
error: pasting formed '(FCustomAllocator', an invalid preprocessing token [-Winvalid-token-paste]
error: pasting formed '<FCustomAllocator', an invalid preprocessing token [-Winvalid-token-paste]
```
→ 见 **F-MACRO-001（P1）**。

#### (b) `BINDING_CONSTRUCTOR` —— `, ##__VA_ARGS__`（逗号形式，三个编译器都正确）

```cpp
// Macro/BindingMacro.h:288-296
#define BINDING_CONSTRUCTOR_BUILDER_INVOKE(T, ...) TFunctionPointer<decltype(&TConstructorBuilder<T, ##__VA_ARGS__>::Invoke)>(&TConstructorBuilder<T, ##__VA_ARGS__>::Invoke).Value.Pointer
```

| 调用（真实代码） | 展开后（Editor） |
|---|---|
| `BINDING_CONSTRUCTOR(FVector)`（`TBindingClassBuilder.inl:18`） | `TFunctionPointer<decltype(&TConstructorBuilder<FVector>::Invoke)>(&TConstructorBuilder<FVector>::Invoke).Value.Pointer, {[](){ return TConstructorBuilder<FVector>::Info(); }}` |
| `BINDING_CONSTRUCTOR(FVector, FVector::FReal)`（`FRegisterVector.cpp:92`） | `... TConstructorBuilder<FVector, FVector::FReal>::Invoke ...`（注意 MSVC 一致性预处理器下逗号后无空格，仍合法） |
| `BINDING_CONSTRUCTOR(FVector, FVector::FReal, FVector::FReal, FVector::FReal)`（`:94`） | `... TConstructorBuilder<FVector, FVector::FReal, FVector::FReal, FVector::FReal>::Invoke ...` |
| `BINDING_CONSTRUCTOR(FSoftObjectPath, const FSoftObjectPath&)`（`FRegisterSoftObjectPath.cpp:18`） | `... TConstructorBuilder<FSoftObjectPath, const FSoftObjectPath&>::Invoke ...` |

**`__VA_ARGS__` 为空**：MSVC 一致性预处理器 → `TConstructorBuilder<FVector>`（逗号被吞、**无警告**）；MSVC 传统预处理器 → `TConstructorBuilder<FVector >`（逗号被吞，多一个空格，合法）；Clang → `TConstructorBuilder<FVector>`。**这一族写法在三家编译器上都正确**（因为 `##` 左边是逗号，落在 GCC/Clang 文档化的 GNU 逗号省略扩展里；MSVC 两种预处理器也都特判了逗号）。

#### (c) `BINDING_PROPERTY` / `BINDING_READONLY_PROPERTY` —— 尾部 `, ##__VA_ARGS__`

```cpp
// Macro/BindingMacro.h:250-260
#if WITH_PROPERTY_INFO
#define BINDING_PROPERTY(Property, ...) BINDING_PROPERTY_BUILDER_GET(Property), BINDING_PROPERTY_BUILDER_SET(Property), BINDING_PROPERTY_BUILDER_INFO(Property), ##__VA_ARGS__
#else
#define BINDING_PROPERTY(Property, ...) BINDING_PROPERTY_BUILDER_GET(Property), BINDING_PROPERTY_BUILDER_SET(Property)
#endif
```

`.Property("AnyKey", BINDING_PROPERTY(&EKeys::AnyKey))`（`FRegisterKeys.cpp:14`）在 Editor（`WITH_PROPERTY_INFO=WITH_EDITOR=1`）下展开为 3 个实参：
```cpp
static_cast<void(*)(const IManagedHandle InManagedHandle, uint8* ReturnBuffer)>(
    [](const IManagedHandle InManagedHandle, uint8* ReturnBuffer){
        TPropertyBuilder<decltype(&EKeys::AnyKey), &EKeys::AnyKey>::Get(InManagedHandle, ReturnBuffer); }),
TPropertyBuilder<decltype(&EKeys::AnyKey), &EKeys::AnyKey>::Set,
{[](){ return TPropertyBuilder<decltype(&EKeys::AnyKey), &EKeys::AnyKey>::Info(); }}
```
（其中 `BINDING_PROPERTY_BUILDER_GET_SIGNATURE` → `const IManagedHandle InManagedHandle, uint8* ReturnBuffer`，由 `SignatureMacro.h:26` + `CoreMacro/BufferMacro.h:21` 两层展开得到；`BINDING_PROPERTY_BUILDER_GET_PARAM` → `InManagedHandle, ReturnBuffer`。）

Shipping 下只有前 2 个实参（`..., nullptr` for READONLY）。**额外实参在 Shipping 被静默丢弃** —— 见 F-MACRO-007。

#### (d) `BINDING_FUNCTION` / `BINDING_OVERLOAD` / `BINDING_SUBSCRIPT`（同构，共用 `WITH_FUNCTION_INFO` 开关）

```cpp
// Macro/BindingMacro.h:266-270
#if WITH_FUNCTION_INFO
#define BINDING_FUNCTION(Function, ...) BINDING_FUNCTION_BUILDER_INVOKE(Function), BINDING_FUNCTION_BUILDER_INFO(Function, ##__VA_ARGS__)
#else
#define BINDING_FUNCTION(Function, ...) BINDING_FUNCTION_BUILDER_INVOKE(Function)
#endif
```
`.Function("Cross", BINDING_FUNCTION(&FVector::Cross, TArray<FString>{"V2"}))`（`FRegisterVector.cpp:157`）Editor 展开：
```cpp
TFunctionPointer<decltype(&TFunctionBuilder<decltype(&FVector::Cross), &FVector::Cross>::Invoke)>(&TFunctionBuilder<decltype(&FVector::Cross), &FVector::Cross>::Invoke).Value.Pointer,
{[](){ return TFunctionBuilder<decltype(&FVector::Cross), &FVector::Cross>::Info(TArray<FString>{"V2"}); }}
```
Shipping 展开只留第一行 —— `TArray<FString>{"V2"}`（形参名元数据）与 `KINDA_SMALL_NUMBER`（默认值元数据）被整体丢弃，消费方 `TClassBuilder::Function`/`FClassBuilder.h:24-29`（`Function` 模板声明）、`FClassBuilder.h:32-39`（`Property` 模板声明）在 `#if WITH_FUNCTION_INFO`/`#if WITH_PROPERTY_INFO` 下也相应少一个形参，**两侧同步，逻辑自洽**（这一点我核对过 `TClassBuilder.inl:34-51`、`FClassBuilder.inl:3-20/22-50` 的实现；`FClassBuilder.h:26-28`、`:35-38` 就是那两处 `#if`）。

注意 `BINDING_OVERLOAD` 与 `BINDING_FUNCTION` 的唯一区别是第 1 个形参是 `Signature` 而非 `Function`，展开结构完全相同：
```cpp
BINDING_OVERLOAD(FVector(*)(const FVector&, const int32), &MinusImplementation)   // FRegisterVector.cpp:112
→ TFunctionPointer<decltype(&TFunctionBuilder<FVector(*)(const FVector&, const int32), &MinusImplementation>::Invoke)>(...).Value.Pointer,
  {[](){ return TFunctionBuilder<FVector(*)(const FVector&, const int32), &MinusImplementation>::Info(); }}
```
**`Signature` 里含逗号时不受影响**（`FVector(*)(const FVector&, const int32)` 的逗号在圆括号内）；但 `__VA_ARGS__` 里的 `TArray<FString>{"Tolerance", "ResultIfZero"}`（`FRegisterVector.cpp:202`）的逗号在**花括号**内 —— 花括号不保护宏实参，该逗号会把 `__VA_ARGS__` 拆成多个实参；由于 `__VA_ARGS__` 被原样重新拼接（并保留实参间逗号），最终文本 `TArray<FString>{"Tolerance", "ResultIfZero"}, SMALL_NUMBER, FVector::ZeroVector` 仍然正确。**这是一个"看起来会错但其实不会错"的巧合，依赖 `__VA_ARGS__` 的拼接保真性**（MSVC 传统/一致性、Clang 三方实测均正确）。

#### (e) `BINDING_CLASS`（8 个偏特化）/ `BINDING_ENUM`（9 个）/ `BINDING_SCRIPT_STRUCT`（9 个）/ `BINDING_STRUCT`（1 个）

> 计数口径（逐个数过 `Macro/BindingMacro.h` 里每个宏体中的 `struct X<...>` 特化）：`BINDING_CLASS`（`:49-108`）= `TName`(51)、`TNameSpace`(68)、`TPropertyClass`(78)、`TPropertyValue`(83)、`TPropertyBuilder`(88)、`TPropertyBuilder`(93)、`TArgument`(98)、`TReturnValue`(104) = **8**；`BINDING_SCRIPT_STRUCT`（`:118-168`）= 同上 8 个 + `TIsScriptStruct`(165) = **9**；`BINDING_ENUM`（`:170-234`）= 8 个 + `TIsNotUEnum`(231) = **9**；`BINDING_STRUCT`（`:110-116`）= `TStaticName`(112) = **1**。

`BINDING_CLASS(FActorSpawnParameters)`（`FRegisterWorld.cpp:12`）一次性生成 8 个偏特化：`TName`、`TNameSpace`、`TPropertyClass`、`TPropertyValue`、`TPropertyBuilder`（成员指针 2 个形态）、`TArgument`、`TReturnValue`。其中 `TName::Get()` 的展开（MSVC/Clang 实测一致）：

```cpp
template <typename T>
struct TName<T, std::enable_if_t<std::is_same_v<std::decay_t<std::remove_pointer_t<std::remove_reference_t<T>>>, FActorSpawnParameters>, T>>
{
	static auto Get()
	{
		if constexpr (!TSizeof_Args())            // ← __VA_ARGS__ 为空：TSizeof_Args() 合法，返回 0
		{
			return FString(TEXT("FActorSpawnParameters"));
		}
		else
		{
			return TGet_Args<0>() ? FString(TEXT("FActorSpawnParameters")).Left(FString(TEXT("FActorSpawnParameters")).Find(TEXT("::")))
			                      : FString(TEXT("FActorSpawnParameters")).Right(FString(TEXT("FActorSpawnParameters")).Len() - FString(TEXT("FActorSpawnParameters")).Find(TEXT("::")) - 2);
		}
	}
};
```
（`F_STRING_STR(Class)` = `FString(TEXT(STR(Class)))`，由 `CoreMacro/Macro.h:129-133` 两层宏展开得到。）

`BINDING_ENUM(FActorSpawnParameters::ESpawnActorNameMode, false)`（`FRegisterWorld.cpp:10`）：
```cpp
if constexpr (!TSizeof_Args(false))
{
	return FString(TEXT("FActorSpawnParameters::ESpawnActorNameMode"));
}
else
{
	return TGet_Args<0>(false)
		? FString(TEXT("FActorSpawnParameters::ESpawnActorNameMode")).Left(FString(...).Find(TEXT("::")))
		: FString(TEXT("FActorSpawnParameters::ESpawnActorNameMode")).Right(FString(...).Len() - FString(...).Find(TEXT("::")) - 2);
}
```
`TSizeof_Args(false)` = `sizeof...(InArgs)` = 1 → 取 `else` 分支；`TGet_Args<0>(false)` 返回 `bool&&` 指向 `forward_as_tuple` 里的临时 → `false` → 走 `Right(Len-Find-2)` = `FString("ESpawnActorNameMode")`（Len=40, Find=21 → `Right(17)`）。

**空 `__VA_ARGS__` 的行为已实测正确**：`TSizeof_Args()` 空包合法（`sizeof...` = 0 → `!0` = true → 直接返回类名）；`else` 分支里的 `TGet_Args<0>()` 虽然被写出来了，但 `Get()` 是模板实体的成员函数，`if constexpr` 的被丢弃分支不被实例化，且存在 `template <auto Index, typename... Args> auto TGet_Args()`（`BindingMacro.h:35-39`）作为兜底重载 —— **不会报错**。三方编译器实测一致。

`BINDING_SCRIPT_STRUCT(FVector)` 额外生成 `template <> struct TIsScriptStruct<FVector> { enum { Value = true }; };`（`BindingMacro.h:164-168`）；`BINDING_ENUM` 额外生成 `TIsNotUEnum<Class>{Value=true}`（`:230-234`）；`BINDING_STRUCT` **只生成 1 个** `TStaticName` 特化（`:110-116`）。

### 1.3 排斥机制（本报告最有价值的架构结论）

`BINDING_*` 生成的偏特化，与 Core 侧 `TName.inl/TNameSpace.inl/TPropertyClass.inl/TArgument.inl/TReturnValue.inl` 里"基于 trait"的通用偏特化，**形状完全相同**：`template <typename T> struct X<T, std::enable_if_t<cond, T>>`。条件同时成立就是二义（我用真实编译器复现，见 F-MACRO-004：MSVC `error C2752`）。

因此这套宏能工作，靠的是**类型层面的互斥**，而不是偏序关系：

| 宏 | 适用类型 | 互斥机制（已逐条实测确认） |
|---|---|---|
| `BINDING_SCRIPT_STRUCT` | Core 的 `noexport` 类型（`FVector`/`FGUID`/`FFrameRate`/`FPolyglotTextData`…） | 这些类型**没有 `StaticStruct()` 成员** → `TIsUStruct<T>::Value == false` → `TName.inl:170` 的条件不成立。实测：`NoExportTypes.h:588 struct FVector` / `:534 struct FGuid` 等**没有 `GENERATED_BODY()`**；`PolyglotTextData.h:16`、`FrameRate.h:20`、`RandomStream.h:19`、`MaterialExpressionIO.h:22` 同样没有 GENERATED 宏；而常规 USTRUCT 由 UHT 生成 `static class UScriptStruct* StaticStruct();`（本工程 `Intermediate/Build/Win64/UnrealCSharpTest/Inc/ActorLayerUtilities/UHT/ActorLayerUtilities.generated.h:26` 等 1658 个文件均有） |
| `BINDING_STRUCT` | 真正的 USTRUCT（有 `StaticStruct()`） | 它**只**特化 `TStaticName`，不去碰 `TName` 等 → 天然不冲突 |
| `BINDING_ENUM` | 枚举 | 自己写 `TIsNotUEnum<Class>{true}`，掐掉 `TName.inl:286` 的 `TIsEnum && !TIsNotUEnum` 条件 |
| `BINDING_CLASS` | 纯 C++ 结构体（**不是** USTRUCT） | 实测 `World.h:418 struct FActorSpawnParameters`、`InputCoreTypes.h:261 struct EKeys`、`EnhancedInputComponent.h` 的 `FInputBindingHandle` 都**没有** `GENERATED_USTRUCT_BODY()` → `TIsUStruct` 为假 |

**版本门控**（`UEVersion.h`）：`UE_U_STRUCT_F_SOFT_OBJECT_PATH/CLASS_PATH/PRIMARY_ASSET_TYPE/PRIMARY_ASSET_ID/ASSET_BUNDLE_DATA/ASSET_BUNDLE_ENTRY/TEST_UNINITIALIZED_SCRIPT_STRUCT_MEMBERS_TEST` **都等于 `UE_VERSION_START(5, 7, 0)`**（`UEVersion.h:176-188`），`UE_REGULAR_UENUM_STATIC_ENUM` = `UE_VERSION_START(5, 8, 0)`（`:206`）。即：UE 5.7 起这 7 个类型变成真正的 USTRUCT，插件用 `#if` 从 `BINDING_SCRIPT_STRUCT` 切到 `BINDING_STRUCT`（`TScriptStruct.inl:47-83` 与 `FRegister*.cpp` 成对 `#if !UE_U_STRUCT_*` / `#if UE_U_STRUCT_*`）。本工程是 UE 5.6 → 这些分支**全部走 `BINDING_SCRIPT_STRUCT`**。

### 1.4 编译期开关

```cpp
// CoreMacro/BindingMacro.h:15-19
#define WITH_TYPE_INFO     WITH_EDITOR
#define WITH_FUNCTION_INFO WITH_EDITOR
#define WITH_PROPERTY_INFO WITH_EDITOR
```
使用量：`WITH_FUNCTION_INFO` 26 处、`WITH_PROPERTY_INFO` 6 处、`WITH_TYPE_INFO` 3 处。**所有"元数据"实参（形参名数组、默认值、`EPropertyInteract`）只在 Editor 构建存在**，Shipping 下由宏与消费方同步裁掉（已核对 `TClassBuilder.inl`、`FClassBuilder.h`）。

---

## 2. 关键调用链

1. `FRegisterVector.cpp:91` → `TBindingClassBuilder<FVector>(NAMESPACE_BINDING)` → `TClassBuilder.inl:17-29` → `TName<FVector,FVector>::Get()`（**宏生成**，`Macro/BindingMacro.h:118-123`，返回 `FString("FVector")`）→ `FClassBuilder` 构造 → C# 侧 `Script.Binding.FVector`。
2. `FRegisterVector.cpp:141` → `.Property("ZeroVector", BINDING_READONLY_PROPERTY(&FVector::ZeroVector))` → `Macro/BindingMacro.h:259` → `FClassBuilder` 的 2 实参 `Property` 重载 → `TPropertyBuilder<decltype(&FVector::ZeroVector), &FVector::ZeroVector>::Get/Set`（`TPropertyBuilder.inl`，**由 `BINDING_SCRIPT_STRUCT(FVector)` 提供特化**）。
3. `TBindingClassBuilder.inl:21` → `BINDING_DESTRUCTOR()` → `Macro/BindingMacro.h:303/305` → `TDestructorBuilder<>::Invoke` 的函数指针（`TDestructorBuilder.inl:10`）。
4. `FRegisterVector.cpp:108` → `BINDING_SUBSCRIPT(FVector, FVector::FReal, int32, TArray<FString>{"Index"})` → `Macro/BindingMacro.h:315` → `TSubscriptBuilder<FVector, FVector::FReal, int32>::Get/Set`（签名来自 `Macro/SignatureMacro.h:18-24`）。
5. `FUnrealFunctionDescriptor.inl:19` → `PROCESS_RETURN()` → `Macro/FunctionMacro.h:136-147` → `ReturnPropertyDescriptor->Get<FPropertyArgument::FReturn>(...)` + `BufferAllocator->Free(Params)`。
6. `FPropertyDescriptor.cpp:49` → `NEW_PROPERTY_DESCRIPTOR(FByteProperty)` → `Macro/PropertyMacro.h:5` → `if (auto Property = CastField<FByteProperty>(InProperty)) return new FBytePropertyDescriptor(Property);`（共 33 条串联）。
7. `TBaseStructure.inl:41` → `BINDING_REMOVE_PREFIX_CLASS_STR(FIntPoint)` → `Macro/BindingMacro.h:47` → `FString(TEXT("FIntPoint")).RightChop(1)` → `StaticGetBaseStructureInternal(FName("IntPoint"))`（共 26 处）。
8. `TScriptStruct.inl:17` → `BINDING_SCRIPT_STRUCT(FVector)` → 9 个偏特化（其中 1 个是 `TIsScriptStruct<FVector>{true}`，另 8 个是 `TName`/`TNameSpace`/…/`TReturnValue`）→ 被 `FCSharpEnvironment.h:267` 的 `TGetObject<TIsScriptStruct>`、`TIsProjectClass.inl:21`、`TConstructorHelper.inl:33` 消费。

---

## 3. 发现清单

### P0

未发现 P0 级问题（无内存破坏/必然崩溃路径；见 F-MACRO-005 的 Null 解引用为条件性）。

### P1

**为空**：F-MACRO-001 的严重度为 P2，已排在小节 `### P2` 之下（级别与小节归属一致），**编号不变**。

### P2

### [F-MACRO-001] `##__VA_ARGS__` 用在非逗号位置：Clang 下是硬错误，MSVC 一致性预处理器下是 C5103 警告

- **类别**: 平台兼容
- **严重度**: **P2**
- **置信度**: 高（编译器实测；"当前无调用点触发"亦为 grep 实测）
- **复核结论**: ✅部分确认（偏差：**文件行号**。`:288`/`:290` 是 `BINDING_CONSTRUCTOR_BUILDER_INVOKE`/`_INFO`（逗号位置，属 §1.2(b) 的**正确**写法），真正"非逗号位置"的 `BINDING_DESTRUCTOR_*` 在 `:298`/`:300`；机制、Clang 硬错误、MSVC C5103 三档结论**全部复现成立**）
- **可达性**: **潜伏**（全工程 0 处带实参调用 —— 插件侧 `BINDING_DESTRUCTOR` 只有 `TBindingClassBuilder.inl:21` 一处**空实参**调用，`BINDING_OVERLOADS` 0 处调用；游戏侧 `UnrealCSharpTest/Source` 也只用 `BINDING_CLASS`/`BINDING_ENUM`，无 `BINDING_DESTRUCTOR`。缺陷只在"传实参"时触发，故为潜伏而非活跃）
- **文件**: `Source/UnrealCSharp/Public/Macro/BindingMacro.h:298`（主）、`:300`、`:303`、`:305`、`:285`
- **函数**: 宏 `BINDING_DESTRUCTOR_BUILDER_INVOKE(...)` / `BINDING_DESTRUCTOR_BUILDER_INFO(...)` / `BINDING_DESTRUCTOR(...)` / `BINDING_OVERLOADS(...)`
- **复核证据**: `read` 逐字核对 —— `:285`=`#define BINDING_OVERLOADS(...) TOverloadBuilder<void*>::Get(##__VA_ARGS__)`、`:298`=`BINDING_DESTRUCTOR_BUILDER_INVOKE`、`:300`=`BINDING_DESTRUCTOR_BUILDER_INFO`、`:303`/`:305`=`BINDING_DESTRUCTOR`（`#if WITH_FUNCTION_INFO` 两分支）。**用报告点名的同一对编译器复跑**（探针 `%TEMP%\a9r2probe\probe.cpp`，§0.5）：clang 20.0.0git 报 `error: pasting formed '(FCustomAllocator', an invalid preprocessing token [-Winvalid-token-paste]` 与 `'<FCustomAllocator'`（M02）及 `'(0'`（M03），MSVC 14.51 `/Zc:preprocessor /W4` 报 `warning C5103: pasting '(' and 'FCustomAllocator' …` ×2 + `'(' and '0'` ×1，MSVC 传统模式无诊断 —— 与报告表格**逐字吻合**；`UEBuildWindows.cs:589 public bool bStrictPreprocessorConformance = false;`、`VCToolChain.cs:681-684` 追加 `/Zc:preprocessor` 亦核实。唯一未复现项：最小探针里 MSVC **两种模式**都没递归展开嵌套的 `BINDING_DESTRUCTOR_BUILDER_INVOKE()`（用最简 `ABCD/ABCE` 用例可定向复现的宏表假阴性）；因本机存在成功的 Win64/MSVC 编译产物（12 个 `Module.*.cpp.obj` + `Binaries/Win64/UnrealEditor-UnrealCSharp.dll`）且该宏每个绑定类 TU 都展开，判定为**探针假阴性**，见 §0.5
- **级别变动**: 无（P1→P2 的调整理由成立；未再降级 —— 按 `_CONVENTIONS.md` "P2=性能/隐患"，这是会在 Clang 平台直接构建失败的**门面隐患**）

**现状（代码事实）**
```cpp
// Macro/BindingMacro.h:285（#else 分支，即 Shipping 走这条）
#define BINDING_OVERLOADS(...) TOverloadBuilder<void*>::Get(##__VA_ARGS__)
// Macro/BindingMacro.h:298
#define BINDING_DESTRUCTOR_BUILDER_INVOKE(...) TFunctionPointer<decltype(&TDestructorBuilder<##__VA_ARGS__>::Invoke)>(&TDestructorBuilder<##__VA_ARGS__>::Invoke).Value.Pointer
// Macro/BindingMacro.h:300
#define BINDING_DESTRUCTOR_BUILDER_INFO(...) {[](){ return TDestructorBuilder<##__VA_ARGS__>::Info(); }}
// Macro/BindingMacro.h:303
#define BINDING_DESTRUCTOR(...) BINDING_DESTRUCTOR_BUILDER_INVOKE(##__VA_ARGS__), BINDING_DESTRUCTOR_BUILDER_INFO(##__VA_ARGS__)
```
`##` 的**左操作数分别是 `<`、`(`、`(`**，都不是逗号 —— 即不在 GCC/Clang 文档化的 `, ## __VA_ARGS__` 特殊情形内，而是"真的做 token paste"。

**调用上下文**
- `Source/UnrealCSharp/Public/Binding/Class/TBindingClassBuilder.inl:21`：`TClassBuilder<T, IsProjectClass0>::Destructor(BINDING_DESTRUCTOR());`（空实参 → 不触发）
- `BINDING_DESTRUCTOR` 全插件仅此 1 处调用；`BINDING_OVERLOADS` 0 处调用（见 §4）
- 宏签名 `BINDING_DESTRUCTOR(...)` 与目标模板 `TDestructorBuilder<typename... Args>`（`TDestructorBuilder.inl:6`）都明确接受实参，**API 门面在邀请用户传参**

**问题**
MSVC 与 Clang 实测（探针 `%TEMP%\a9probe\probe.cpp`，逐字节复制宏体）：

| 编译器/模式 | `BINDING_DESTRUCTOR()`（空） | `BINDING_DESTRUCTOR(FCustomAllocator)`（带参） | `BINDING_OVERLOADS(0)` |
|---|---|---|---|
| MSVC 14.51 传统预处理器 `/EP /W4` | 正确，无警告 | 正确，**无警告** | 正确，无警告 |
| MSVC 14.51 `/EP /Zc:preprocessor /W4` | 正确，无警告 | 正确，**2×`warning C5103`** | **1×`warning C5103`** |
| Clang 20.0.0git `-E -std=c++20`（无额外开关） | 正确，无诊断 | **2×hard error** | **1×hard error** + `BINDING_DESTRUCTOR` 共 9 errors |

Clang 原文（`-Winvalid-token-paste` 默认为 error）：
```
error: pasting formed '(FCustomAllocator', an invalid preprocessing token [-Winvalid-token-paste]
error: pasting formed '<FCustomAllocator', an invalid preprocessing token [-Winvalid-token-paste]
```
触发路径：Linux/Mac/Android/iOS 工具链（UE 在这些平台用 Clang），以及 `-StrictPreprocessor` / C++20 modules（MSVC）。注意 UE 侧 `/Zc:preprocessor` 是**默认关闭**的（`Engine/Source/Programs/UnrealBuildTool/Platform/Windows/UEBuildWindows.cs:589 public bool bStrictPreprocessorConformance = false;`，`VCToolChain.cs:681-684` 据此才追加该开关），所以 Windows 默认构建"看起来没事"，平台差异被掩盖。MSVC 的 C5103 一旦配合 `bWarningsAsErrors` 或 `-StrictPreprocessor` 也会变成构建失败。

**建议**
三种写法任选其一，优先第 1 种（零成本、三家编译器一致）：
```cpp
// 1) 让 ## 只出现在逗号后面（GNU 逗号省略扩展，MSVC 传统/一致性 + Clang 都支持）
#define BINDING_DESTRUCTOR_BUILDER_INVOKE(...) \
    TFunctionPointer<decltype(&TDestructorBuilder<, ##__VA_ARGS__>::Invoke)>  // 不可行，模板实参不能这么写
```
更实际的方案是把"可选模板实参"改成"显式重载 + 不用 `##`"：
```cpp
// 方案 A：C++20 __VA_OPT__（MSVC /Zc:preprocessor、Clang、GCC 均支持；UE 5.6 用 C++20）
#define BINDING_DESTRUCTOR_BUILDER_INVOKE(...) \
    TFunctionPointer<decltype(&TDestructorBuilder<__VA_OPT__(__VA_ARGS__)>::Invoke)>(&TDestructorBuilder<__VA_OPT__(__VA_ARGS__)>::Invoke).Value.Pointer
#define BINDING_DESTRUCTOR(...) \
    BINDING_DESTRUCTOR_BUILDER_INVOKE(__VA_ARGS__), BINDING_DESTRUCTOR_BUILDER_INFO(__VA_ARGS__)
// 方案 B：把 ## 挪到逗号后 —— 需要 TDestructorBuilder 额外接受一个"哨兵类型"，代价大
// 方案 C（最小改动、建议至少立即做）：用 static_assert 把"带参调用"变成自救性错误
#define BINDING_DESTRUCTOR(...) static_assert(TSizeof_Args(__VA_ARGS__) == 0, \
    "BINDING_DESTRUCTOR does not accept arguments on Clang; see F-MACRO-001")
```
若确实需要带参析构（`TDestructorBuilder<Args...>`），必须走方案 A。
**注意**：`Macro/BindingMacro.h:285` 的 `TOverloadBuilder<void*>::Get(##__VA_ARGS__)` 应改成 `Get(__VA_ARGS__)`（`Get` 本身就是 `template <typename... Args>`，空包合法，`TOverloadBuilder.inl:52-60`），**根本不需要 `##`**。

**验证方式**
```powershell
# 复现（把 BindingMacro.h:285-306 逐字节复制到 probe.cpp，末尾加 M01: BINDING_DESTRUCTOR() 与 M02: BINDING_DESTRUCTOR(FCustomAllocator)）
& "<engine>\Engine\Plugins\Experimental\NNERuntimeIREE\Binaries\ThirdParty\IREE\Windows\clang++.exe" -E -x c++ -std=c++20 -o NUL probe.cpp   # 期望 0 error
cl /nologo /EP /Zc:preprocessor /W4 probe.cpp                                                                                                  # 期望 0 C5103
```
grep 回归：`grep -n '##__VA_ARGS__' Source/UnrealCSharp/Public/Macro/BindingMacro.h` → 应只剩 `, ##__VA_ARGS__` 形式。

---

### P3（F-MACRO-004～006：由 P2 下调为 P3 的潜伏项）

### [F-MACRO-002] 命名空间剥离宏在类型名不含 `::` 时静默产生空串或"丢掉首字符"

- **类别**: Bug
- **严重度**: **P2**
- **文件**: `Source/UnrealCSharp/Public/Macro/BindingMacro.h:41-45`
- **函数**: 宏 `BINDING_REMOVE_LEFT_NAMESPACE_CLASS_STR(Class)` / `BINDING_REMOVE_RIGHT_NAMESPACE_CLASS_STR(Class)`
- **置信度**: 高（宏体 + UE 5.6 `FString` 实现逐行核对）
- **复核结论**: ✅部分确认（偏差：**算术**。结论"`Left(-1)` 返回空串""`Right` 丢首字符"成立，但报告写的 `Right(8-(-1)-2)`=`Right(7)` 与 `**this - 1` 是错的，已修正）
- **可达性**: **潜伏**（`BINDING_ENUM` 全工程 7 处调用中只有 `FRegisterWorld.cpp:10` 传了标志且**含** `::`；`BINDING_CLASS` 的 4 处（`FRegisterKeys.cpp:5`、`FRegisterWorld.cpp:12`、`FRegisterEnhancedInputComponent.cpp:11,13`）与游戏侧 `UnrealCSharpTest/Source` 的 4 处（`FTestBindingFunction.cpp:7` 等）**全部不传标志**，两个剥离宏都不参与）
- **复核证据**: 宏体 `Macro/BindingMacro.h:41-45` 逐字一致；引擎侧重读 —— `Left`=`UnrealString.h.inl:1146-1149`（`ConstructFromPtrSize(**this, FMath::Clamp(Count, 0, Len()))`）、`Right`=`UnrealString.h.inl:1196-1200`（`Length = Len(); return UE_STRING_CLASS(**this + Length - FMath::Clamp(Count, 0, Length));`）、`Find` 未命中返回 `INDEX_NONE` = `UnrealString.h.inl:1314-1318`。修正后的算术：`Left(-1)`→`Clamp(-1,0,Len)`=0→空串；`"EForceInit"`（Len=**10**，非报告写的 8）`Right(10-(-1)-2)`=`Right(9)`→`"ForceInit"`（丢首字符）；若真的越界（`Right(Len+1)`）会被 clamp 成 `Right(Len)` → 返回**整个字符串**（不是报告写的 `**this - 1`）
- **级别变动**: 无（P2 维持：`F_STRING_STR` 门面上的静默错名路径，且 `BINDING_ENUM`/`BINDING_CLASS` 确被游戏侧手写绑定代码使用，属"API 门面 + 静默错误"）

**现状（代码事实）**
```cpp
// Macro/BindingMacro.h:41-45
#define BINDING_REMOVE_LEFT_NAMESPACE_CLASS_STR(Class) F_STRING_STR(Class).Left( \
	F_STRING_STR(Class).Find(TEXT("::")))

#define BINDING_REMOVE_RIGHT_NAMESPACE_CLASS_STR(Class) F_STRING_STR(Class).Right( \
	F_STRING_STR(Class).Len() - F_STRING_STR(Class).Find(TEXT("::")) - 2)
```
唯一消费者是自己文件里的 `BINDING_CLASS(Class, ...)`（`:61-63`）与 `BINDING_ENUM(Class, ...)`（`:182-184`）的三元表达式。

**调用上下文**
- `FRegisterWorld.cpp:10`：`BINDING_ENUM(FActorSpawnParameters::ESpawnActorNameMode, false)` —— 含 `::`，走 `Right` 分支，结果正确（`Right(40-21-2)` = `"ESpawnActorNameMode"`）
- `FRegisterWorld.cpp:12`、`FRegisterKeys.cpp:5`、`FRegisterEnhancedInputComponent.cpp:11,13`：`BINDING_CLASS(X)` 不传标志 → `TSizeof_Args()==0` → 直接返回全名，**两个剥离宏都不参与**

**问题**
1. `F_STRING_STR(Class).Find(TEXT("::"))` 找不到时返回 `INDEX_NONE`（`-1`）（UE 5.6 `Engine/Source/Runtime/Core/Public/Containers/UnrealString.h.inl:1317`：`return SubStr ? Find(...) : INDEX_NONE;`）。
2. `FString::Left` 的 UE 5.6 实现是 `ConstructFromPtrSize(**this, FMath::Clamp(Count, 0, Len()))`（`UnrealString.h.inl:1146-1149`）→ `Left(-1)` **返回空字符串**（不报错、不崩溃）。
3. `FString::Right` 同样是 `FMath::Clamp(Count, 0, Length)`（`UnrealString.h.inl:1196-1200`）→ 实现是 `return UE_STRING_CLASS(**this + Length - FMath::Clamp(Count, 0, Length));`，即 `Count` 先被 clamp 到 `[0, Len]` **再**从长度里减：`Right(Len + 1)` 会被 clamp 成 `Right(Len)` → 返回**整个字符串**；真正导致"丢首字符"的是下面表格里的 `Right(Len - Find - 2)` 在 `Find == INDEX_NONE` 时退化为 `Right(Len - 1)`。

具体后果（对"不含 `::` 的名字"）：

| 调用 | 期望 | 实际（空串 / 丢首字符） |
|---|---|---|
| `BINDING_CLASS(EKeys, true)` | 报错或返回 `"EKeys"` | `Left(-1)` → `Clamp(-1,0,5)`=0 → `""`（**空类名**，C# 侧绑定到一个空名类型） |
| `BINDING_ENUM(EForceInit, false)` | 报错或返回 `"EForceInit"` | `Right(Len-(-1)-2)` = `Right(10+1-2)` = `Right(9)` → `"ForceInit"`（**悄悄丢掉首字符 `E`**；`"EForceInit"` 的 `Len` 是 10） |

`BINDING_ENUM` 现有 7 处调用中仅 `FRegisterWorld.cpp:10` 传了标志且含 `::`，`BINDING_CLASS` 的 4 处全部不传标志 —— 所以**当前没有触发**，但这是对外 API 门面上的静默错误路径：错误不会在编译期或运行期报出来，只会生成错误的 C# 类型名，症状离现场极远。

**建议**
在 `Get()` 里加显式断言 + 兜底：
```cpp
#define BINDING_REMOVE_RIGHT_NAMESPACE_CLASS_STR(Class) \
    (F_STRING_STR(Class).Contains(TEXT("::")) \
        ? F_STRING_STR(Class).Right(F_STRING_STR(Class).Len() - F_STRING_STR(Class).Find(TEXT("::")) - 2) \
        : F_STRING_STR(Class))
```
`Left` 侧同理（`!Contains` 时返回全名），并考虑加 `ensureMsgf` 提示"标志与类型名不匹配"。

**验证方式**
- `grep -n 'BINDING_ENUM(\|BINDING_CLASS(' Source -r` 列出全部调用点，确认第 2 个实参是否为 `true/false` 且名字含 `::`。
- 单测：`static_assert` 无法覆盖（运行时 FString），改成对 `TName<EForceInit, EForceInit>::Get().IsEmpty()` 的运行时断言。

---

### [F-MACRO-003] 宏生成的 `TName::Get()`/`TNameSpace::Get()` 每次调用都重建 FString（3 次临时构造 + FName→FString 转换）

- **类别**: 性能
- **严重度**: **P2**
- **文件**: `Source/UnrealCSharp/Public/Macro/BindingMacro.h:53-76`（`BINDING_CLASS`）、`:127-131`（`BINDING_SCRIPT_STRUCT`）、`:174-196`（`BINDING_ENUM`）
- **函数**: 宏生成的 `TName<T,T>::Get()` / `TNameSpace<T,T>::Get()`
- **置信度**: 中（宏展开与调用点已确证；"是否落在每帧热路径"未逐一追踪，见"未验证的假设"）
- **复核结论**: ✅确认（严重度 P2 维持；"每次调用都重建 `FString`/`TArray`"在宏体层面无可争辩 —— `F_STRING_STR(Class)` 在 `:44-45`、`:57`、`:62-63`、`:122`、`:178`、`:183-184` 每处都重新构造临时对象，且 `TNameSpace::Get()` 返回**值**）
- **可达性**: **活跃**（`TClassBuilder.inl:20/44/62`、`TBindingClassBuilder.inl:28` 在每个 `TBindingClassBuilder<T>` 构造与每个 `Constructor`/`Destructor` 注册时都会求值；`BINDING_FUNCTION`/`BINDING_PROPERTY` 调用点合计数百处，模块加载阶段必然执行）
- **复核证据**: `Macro/BindingMacro.h:41-47,53-76,127-131,174-196` 逐字核对成立；对照物 `TName.inl:180-186` 的 `return CLASS_F_STRING;`（返回字符串**常量**，无构造）确认位于 `:184`；调用点 `TClassBuilder.inl:20`（`TName<T,T>::Get()`）、`:44`（`TStaticName<T,T>::IsEmpty() ? … : TName<T,T>::Get()`）、`:62`、`TBindingClassBuilder.inl:28`、`Macro/BindingMacro.h:72`（`TNameSpace::Get()` 内部再调 `TName<T,T>::Get()`）全部核实
- **级别变动**: 无（P2 维持；"是否落在**每帧**热路径"仍是未验证假设 → 见 §5.3，若只在注册期执行则实际影响应降 P3）

**现状（代码事实）**
```cpp
// Macro/BindingMacro.h:61-63（BINDING_CLASS 的 TName::Get 三分支之一）
			return TGet_Args<0>(__VA_ARGS__) \
				? BINDING_REMOVE_LEFT_NAMESPACE_CLASS_STR(Class) \
				: BINDING_REMOVE_RIGHT_NAMESPACE_CLASS_STR(Class); \
// Macro/BindingMacro.h:44-45 → 展开后 Class 出现 3 次：
	F_STRING_STR(Class).Right(F_STRING_STR(Class).Len() - F_STRING_STR(Class).Find(TEXT("::")) - 2)
// Macro/BindingMacro.h:70-76
	static auto Get() \
	{ \
		return FBinding::Get().Register().IsProjectClass(TName<T, T>::Get()) \
			? TArray<FString>{COMBINE_NAMESPACE(NAMESPACE_ROOT, FString(FApp::GetProjectName()))} \
			: TArray<FString>{static_cast<FName>(GLongCoreUObjectPackageName).ToString().RightChop(1).Replace(TEXT("/"), TEXT("."))}; \
	} \
```
对比 Core 侧同类特化是**返回常量**（`TName.inl:180-186` `return CLASS_F_STRING;`），而宏生成的这 3 个特化没有任何缓存：每次调用都要
- `FString(TEXT("Xxx"))` × 3（`F_STRING_STR` 每次出现都重新构造一个临时 FString → **3 次堆分配 + 字符串拷贝**，字符串字面量非空时 UE 的 FString 走分配路径）；
- `TNameSpace::Get()` 还要 `FName→FString` 转换 + `RightChop` + `Replace`（`Replace` 再分配一次）+ `IsProjectClass(FString)` 注册表查询（`FString` 作为 key → 哈希/比较）；
- 返回值是 **`TArray<FString>`（每次构造一个新数组）**，不是 `const TArray<FString>&`。

**调用上下文**
- `TClassBuilder.inl:20`（`FClassBuilder` 构造参数 lambda 内 `TName<T,T>::Get()`）、`:44`、`:62` —— 每个绑定类/构造函数/析构函数注册 1 次
- `TBindingClassBuilder.inl:28`（`Inheritance` 里 `TName<Class,Class>::Get()`）
- `TClassBuilder.inl:44` 的 `TStaticName<T,T>::IsEmpty() ? TStaticName<T,T>::Get() : TName<T,T>::Get()`
- `Macro/BindingMacro.h:72` 自身：`TNameSpace::Get()` 又调一次 `TName<T,T>::Get()`

**问题**
这些宏是"每个绑定成员注册时都会展开并求值"的路径。以本插件规模计（**grep 命中数口径**：`BINDING_FUNCTION(` **647** 命中 = 2 处 `#define`（`:267`/`:269`）+ 1 处宏内部使用（`:323`）+ 644 处外部调用；`BINDING_PROPERTY(` **354** 命中 = 2 处 `#define`（`:251`/`:253`）+ 1 处生成器字符串（`SourceCodeGenerator.cs:540`）+ 351 处 C++ 调用），仅在模块加载阶段就会重复构造数万次临时 FString；若 `TName<T,T>::Get()` 同样被属性访问路径调用（`TPropertyClass<T,T>::Get()` → 环境按名字查类），则每次 C#→C++ 属性访问都要重新拼名字并做一次 `TMap<FString,...>` 查找。**均可在不改变语义的前提下改为一次求值的函数内静态量。**

**建议**
```cpp
	static auto Get()
	{
		static const FString Value = []() -> FString
		{
			if constexpr (!TSizeof_Args(__VA_ARGS__)) { return FString(TEXT(STR(Class))); }
			else { /* Left/Right */ }
		}();
		return Value;
	}
```
`TNameSpace::Get()` 同理缓存 `TArray<FString>`（返回 `const TArray<FString>&` 更佳，但需同步修改消费者；C++11 magic static 保证线程安全）。对 `BINDING_CLASS` 的 `TNameSpace`，`IsProjectClass` 的结果取决于注册表内容（初始化顺序），缓存要放在**首次查询之后**，或用 `TMap` 记忆化。

**验证方式**
- 在 `TName<FVector,FVector>::Get()` 内加计数器（或 `LLM`/`-trace=memory`），跑一次编辑器启动 + 一轮属性访问，对比 `Malloc` 次数。
- 快速核对消费方是否容忍返回引用：`grep -n 'TName<.*>::Get()' Source | wc -l`（当前全部按值使用）。

---

### [F-MACRO-004] 宏生成的偏特化与 Core 的 trait 偏特化形状相同：条件一旦重叠即 `C2752`，且报错点在用户调用处

- **类别**: 平台兼容 / 可优化（隐患）
- **严重度**: **P3**
- **文件**: `Source/UnrealCSharp/Public/Macro/BindingMacro.h:49-108`（`BINDING_CLASS`）、`:118-168`（`BINDING_SCRIPT_STRUCT`）、`:170-234`（`BINDING_ENUM`）
- **函数**: 宏生成的 `TName<T, enable_if_t<...>>`、`TNameSpace`、`TPropertyClass`、`TPropertyValue`、`TArgument`、`TReturnValue` 偏特化
- **置信度**: 高（二义机制用真实编译器复现；"当前不触发"用引擎源码实测确认）；**未验证的假设**：若未来某个被绑定类型同时满足 `TIsUStruct`/`TIsEnum`，构建即失败
- **复核结论**: ✅部分确认（偏差：**两处计数**。`TScriptStruct.inl` 的 `BINDING_SCRIPT_STRUCT` 是 **56** 处不是 57，未门控的是 **49** 个不是 50；机制、`C2752`+连带 `C2039` 的诊断、以及 6 组"竞争对"的行号**全部复核成立**）
- **可达性**: **潜伏**（当前 UE 5.6 下互斥成立：7 个 `UE_U_STRUCT_*` 全为 `UE_VERSION_START(5,7,0)` = 0 → 走 `BINDING_SCRIPT_STRUCT`；noexport 类型无 `StaticStruct()`）
- **复核证据**: ① 二义机制**已独立复现**（`%TEMP%\a9r2probe\probe_c2752.cpp`，报告 §"验证方式"给的快速版）：MSVC 14.51 `/c /Zc:preprocessor /std:c++20` → `error C2752: 'X<int,int>': more than one partial specialization matches the template argument list` + `note: could be 'X<T,enable_if<std::is_integral_v<T>,T>::type>'` + `note: or 'X<T,enable_if<sizeof(T)==4,T>::type>'` + **连带 `error C2039: 'v': is not a member of 'X<int,int>'`**（与报告"MSVC 这里是 C2752 + 连带 C2039"完全一致）；clang 20 同一探针给出 `error: ambiguous partial specializations of 'X<int, int>'`（措辞不同，印证报告"MSVC/GCC 诊断可能完全不同"）。
  ② 6 组竞争对行号**逐个核实**：`TName.inl:171`(TIsUStruct) 与 `TName.inl:286`(TIsEnum)、`TNameSpace.inl:144`/`:244`、`TPropertyClass.inl:201`（注意真实路径是 `Source/UnrealCSharp/Public/Binding/Core/TPropertyClass.inl`，**在 UnrealCSharp 模块而非 UnrealCSharpCore**）、`TPropertyValue.inl:478`/`:531`（`:478` 确实用 `&&` 叠了第二条件 —— 可作修法参照）、`TArgument.inl:266`、`TReturnValue.inl:137`（后两个真实路径是 `Source/UnrealCSharp/Public/Binding/Function/…`）。
  ③ 引擎侧：`UEVersion.h:176,178,180,182,184,186,188` 全部 `UE_VERSION_START(5, 7, 0)`；`NoExportTypes.h:534 struct FGuid` / `:588 struct FVector` 且附近无 `GENERATED_BODY()`（该文件 GENERATED_BODY 从 `:2654` 才开始）→ 报告"这些类型没有 GENERATED_BODY"成立。
- **级别变动**: 无（P2→P3 成立：机制真实但当前 0 触发点、且修法（给 Core trait 叠加 `!TIsScriptStruct<...>` 条件）成本明确）

**现状（代码事实）**
```cpp
// Macro/BindingMacro.h:50-52（宏生成，BINDING_CLASS）
template <typename T> \
struct TName<T, std::enable_if_t<std::is_same_v<std::decay_t<std::remove_pointer_t<std::remove_reference_t<T>>>, Class>, T>> \
{ ... };
```
```cpp
// UnrealCSharpCore/Public/Binding/TypeInfo/TName.inl:170-177（Core 通用特化，条件基于 trait）
template <typename T>
struct TName<T, std::enable_if_t<TIsUStruct<std::remove_pointer_t<std::decay_t<T>>>::Value, T>>
{ ... };
// TName.inl:285-292（枚举版）
template <typename T>
struct TName<T, std::enable_if_t<TIsEnum<std::decay_t<T>>::Value && !TIsNotUEnum<std::decay_t<T>>::Value, T>>
{ ... };
```
两者都是 `<T, enable_if_t<cond, T>>`。

**调用上下文**
同形状的"竞争对"至少 6 组：`TName`（`TName.inl:170`、`:286`）、`TNameSpace`（`TNameSpace.inl:144`、`:244`）、`TPropertyClass`（`TPropertyClass.inl:201`）、`TPropertyValue`（`TPropertyValue.inl:478`、`:531`）、`TArgument`（`TArgument.inl:266`）、`TReturnValue`（`TReturnValue.inl:137`）。全部由 `BINDING_SCRIPT_STRUCT`/`BINDING_CLASS`/`BINDING_ENUM` 一次生成（见 §1.2(e)）。

**问题**
复现探针（精确复制两边形状，`T = FVector` 同时满足两个条件）：
```
error C2752: 'TName<FVector,FVector>': more than one partial specialization matches the template argument list
note: could be 'TName<T,enable_if<TIsUStructProbe<...>::Value,T>::type>'
note: or       'TName<T,enable_if<std::is_same_v<...FVector>,T>::type>'
```
把 trait 改成 false 后立刻编译通过（对照组验证）。也就是说：
1. 这套宏的正确性**不来自偏序**，而来自 §1.3 的互斥机制（`TIsUStruct` 对 noexport 类型为假、`TIsNotUEnum` 掐枚举）。
2. 互斥一旦被打破，**报错信息里不会出现任何宏名**，只会指向用户写 `BINDING_SCRIPT_STRUCT(X)` 的那一行，且 MSVC/GCC 可能给出完全不同的诊断（MSVC 这里是 C2752 + 连带 C2039 `'Get' is not a member`）。
3. 已知的破环路径是**引擎升级**：`UE_U_STRUCT_F_SOFT_OBJECT_PATH` 等 7 个标志都是 `UE_VERSION_START(5, 7, 0)`（`UEVersion.h:176,178,180,182,184,186,188`），作者已为这 7 个类型准备了 `BINDING_STRUCT` 分支；但 `TScriptStruct.inl` 里共 **56 处 `BINDING_SCRIPT_STRUCT`**（grep 逐行数：`BINDING_SCRIPT_STRUCT(` 命中 56 行，与 7 行 `#if !UE_U_STRUCT_*` 相加得该文件 63 行总命中），只有 7 处有版本门控 —— 其余 **49** 个 Core noexport 类型（如 `FTransform`、`FMatrix`、`FColor`）若在某个 UE 版本被 UHT 加上 `GENERATED_BODY()`，构建会以极难定位的方式失败。

**建议**
1. 在宏生成的 `Get()` 里加自证断言，把"隐形耦合"变成显式约束：
```cpp
static_assert(!TIsUStruct<Class>::Value, "BINDING_SCRIPT_STRUCT/BINDING_CLASS 与 Core 的 TIsUStruct 特化冲突（C2752）");
static_assert(!TIsEnum<Class>::Value || TIsNotUEnum<Class>::Value, "BINDING_ENUM 与 Core 的 TIsEnum 特化冲突");
```
2. 结构性修法：让 Core 的 trait 特化统一带上"未被宏接管"的条件，例如把 `TName.inl:170` 改为
`std::enable_if_t<TIsUStruct<...>::Value && !TIsScriptStruct<...>::Value, T>`（`TPropertyValue.inl:478` 已经用 `&&` 叠了第二条件的写法，可参照），这样即使条件重叠也是"宏特化更专用"，不再二义。
3. 给 `TScriptStruct.inl` 的 50 个未门控类型补 `UE_U_STRUCT_*` 门控，或在 CI 里对每个 UE 小版本跑一次编译。

**验证方式**
- 探针：见上（`%TEMP%\a9probe\probe3.cpp`，已删除）。快速版：
  ```cpp
  template<class T, class E=void> struct X {};
  template<class T> struct X<T, std::enable_if_t<std::is_integral_v<T>, T>> { static constexpr int v=1; };
  template<class T> struct X<T, std::enable_if_t<sizeof(T)==4, T>>    { static constexpr int v=2; };
  int a = X<int,int>::v;   // 期望：C2752
  ```
- grep 巡检：`grep -c 'BINDING_SCRIPT_STRUCT(' Source/UnrealCSharp/Public/Binding/ScriptStruct/TScriptStruct.inl`（**应 56**，实测）与其中 `#if !UE_U_STRUCT_` 的数量（应 7，实测 `:47,51,55,59,67,71,81`）。

---

### [F-MACRO-005] `PROCESS_RETURN()` 无保护地解引用可能失效的 `TSharedPtr` —— 同一文件非宏路径有 `IsValid()`/`!= nullptr` 保护

- **类别**: 未定义行为（空指针解引用） / 一致性
- **严重度**: **P3**
- **文件**: `Source/UnrealCSharp/Public/Macro/FunctionMacro.h:136-147`
- **函数**: 宏 `PROCESS_RETURN()`（在 `FUnrealFunctionDescriptor::Call1/3/5/7/9/11/15`、`FCSharpDelegateDescriptor::Execute1/3/7` 中展开 —— 该委托类的方法名是 **`Execute*`/`Broadcast*`**，不是 `Call*`）
- **置信度**: 中（宏与保护缺失是事实；`BufferAllocator` 是否真会失效未证伪，反向证据见"问题"）
- **复核结论**: ✅确认（严重度 P3 维持；宏体 `:147` 的裸 `BufferAllocator->Free(Params)` 与同族三处带保护调用形成的不一致**确实存在**）
- **可达性**: **潜伏**（宏本身在所有 Editor/Shipping 构建里都展开（`WITH_FUNCTION_INFO` 只影响末尾的 INFO 分支），但要真的空指针解引用，必须 `BufferAllocator` 无效 —— 而 `FFunctionParamBufferAllocatorFactory::Factory` 返回 `TSharedRef`（`FFunctionParamBufferAllocator.h:60-71`），派生描述符都从它构造，当前路径下 `IsValid()` 可能恒真）
- **复核证据**: `FunctionMacro.h:136-147` 逐字核对（`:147` 裸调用）；保护写法对照 —— `FUnrealFunctionDescriptor.inl:211-214`（Call18）与 `:258-261`（Call26）都在 `if (Params != nullptr)` 内，另有 **`Private/Reflection/Function/FCSharpFunctionDescriptor.cpp:138`，其保护是 `:126` 的 `if (Params != nullptr && Params != InStack.Locals)`** —— 即 `grep 'BufferAllocator->Free'` 全插件 4 命中里**只有 `FunctionMacro.h:147` 一处没有保护**；`Params` 的产生点 `BufferAllocator.IsValid() ? BufferAllocator->Malloc() : nullptr` 在 `FUnrealFunctionDescriptor.inl:15,25,35,47,57,69,82,104,116,128,142,157,192,239`（14 处）与 `FCSharpDelegateDescriptor.inl:16,26,37,49,60,73,98,113,128` 全部核实；`BufferAllocator` 类型 `TSharedPtr<FFunctionParamBufferAllocator>`（`FFunctionDescriptor.h:33`）核实
- **级别变动**: 无（P2→P3 成立：反向证据表明 `IsValid()` 在当前构造路径下恒真，属"作者自身写法不一致"的隐患而非可复现崩溃）

**现状（代码事实）**
```cpp
// Macro/FunctionMacro.h:136-147
#define PROCESS_RETURN() \
	if constexpr (ReturnType == EFunctionReturnType::Primitive) \
	{ \
		ReturnPropertyDescriptor->Get(ReturnPropertyDescriptor->ContainerPtrToValuePtr<void>(Params), RETURN_BUFFER); \
	} \
	else if constexpr (ReturnType == EFunctionReturnType::Compound) \
	{ \
		ReturnPropertyDescriptor->Get<FPropertyArgument::FReturn>( \
			ReturnPropertyDescriptor->CopyValue(ReturnPropertyDescriptor->ContainerPtrToValuePtr<void>(Params)), \
			reinterpret_cast<void**>(RETURN_BUFFER)); \
	} \
	BufferAllocator->Free(Params);      // ← 第 147 行，无任何保护
```

**调用上下文**
同一文件（`FUnrealFunctionDescriptor.inl`）的**非宏**释放路径却做了保护：
```cpp
// FUnrealFunctionDescriptor.inl:211-214（Call18）与 :258-261（Call26）
	if (Params != nullptr)
	{
		BufferAllocator->Free(Params);
	}
```
而宏的调用点把 `Params` 明确赋成"分配器无效时为 nullptr"：
```cpp
// FUnrealFunctionDescriptor.inl:15/25/35/47/57/69/82/104/116/128/142/157/192/239
	const auto Params = BufferAllocator.IsValid() ? BufferAllocator->Malloc() : nullptr;
// 随后调用 PROCESS_RETURN()（:19/41/63/90/110/136/167 等 7 处）
```
`BufferAllocator` 的类型是 `TSharedPtr<FFunctionParamBufferAllocator>`（`FFunctionDescriptor.h:33`）。

**问题**
当 `BufferAllocator` 无效（`IsValid() == false`）时，`Params == nullptr`，但 `PROCESS_RETURN()` 仍执行 `BufferAllocator->Free(Params)` —— 对**无效的 `TSharedPtr` 调用 `operator->`**：UE 的 `TSharedPtr::operator->` 会 `checkSlow(IsValid())` 后直接解引用内部指针，**Shipping（`checkSlow` 被编译掉）下就是空指针解引用 → 崩溃**。

反向证据（因此定级 P2 而非 P1）：`FFunctionParamBufferAllocatorFactory::Factory` 返回 `TSharedRef`（`FFunctionParamBufferAllocator.h:60-71`），派生描述符均以 `Factory<...>(InFunction)` 的结果构造（`FUnrealFunctionDescriptor.cpp:5`、`FCSharpFunctionDescriptor.cpp:9`、`FCSharpDelegateDescriptor.cpp:6`），`TSharedRef→TSharedPtr` 转换必然有效 —— 也就是说 `IsValid()` 在当前构造路径下**可能恒为真**。同一个文件里两种写法并存（宏不保护、手写代码保护）说明作者本人对"它是否可能失效"没有一致结论；这种二义性本身就是缺陷。

**建议**
把保护下沉进宏（与同文件手写路径统一）：
```cpp
	} \
	if (Params != nullptr) \
	{ \
		BufferAllocator->Free(Params); \
	}
```
或反过来：如果确认 `BufferAllocator` 恒有效，就把 14 处 `BufferAllocator.IsValid() ? ... : nullptr` 简化掉并在构造函数里 `check(InBufferAllocator.IsValid())`，让不变量显式化。二者选一，不要两存。

**验证方式**
```cpp
// 在 Call3 内打断点/断言
check(BufferAllocator.IsValid());
```
或在 `FFunctionDescriptor` 构造函数（`FFunctionDescriptor.cpp:4-11`）加 `ensure(InBufferAllocator.IsValid());`，跑一轮"C# 调用无返回值 FUnrealFunctionDescriptor"用例观察是否命中；grep 回归：`grep 'BufferAllocator->Free'` 在 `Source/` 下共 **4** 命中（`FunctionMacro.h:147`、`FUnrealFunctionDescriptor.inl:213`、`:260`、`FCSharpFunctionDescriptor.cpp:138`），**其中只有 `FunctionMacro.h:147` 不在保护分支内**（另三处分别由 `if (Params != nullptr)` 与 `if (Params != nullptr && Params != InStack.Locals)` 保护）—— 只有 `FunctionMacro.h:147` 一处是裸调用。

---

### [F-MACRO-006] `PROCESS_*`/`IN_*` 语句宏没有 `do { } while(0)`：放进无花括号 `if/else` 即编译失败（实测 `error C2062`）

- **类别**: 未定义行为（宏卫生）
- **严重度**: **P3**
- **文件**: `Source/UnrealCSharp/Public/Macro/FunctionMacro.h:65-147`（`INITIALIZE_VALUE` 65-69、`IN_VALUE` 71-73、`REFERENCE_IN_VALUE` 75-79、`IN_END` 81-82、`PROCESS_SCRIPT_IN` 84-87、`PROCESS_SCRIPT_REFERENCE_IN` 89-92、`NATIVE_OUT_VALUE` 94-107、`PROCESS_NATIVE_REFERENCE_IN` 109-114、`PROCESS_OUT` 116-134、`PROCESS_RETURN` 136-147）
- **函数**: 上述 10 个宏
- **置信度**: 高（编译器实测；报告引用的诊断号 `C2062` 不成立，见下）
- **复核结论**: ⚠️**部分确认（偏差：触发条件与诊断号，且报告的语法推理不成立）** —— "宏缺 `do{}while(0)` ⇒ 是上下文敏感语句宏"这个**机制成立**，但报告给出的**具体失败形态不成立**：
  1. 报告 §"问题"里的探针 `void probe_dangling_else(int c){ if (c) PROCESS_SCRIPT_IN() else ; }`（**无尾分号**）实测 **MSVC 14.51（传统 + `/Zc:preprocessor`）与 clang 20.0.0git 全部编译通过、零诊断**（探针 `%TEMP%\a9r2probe\probe_c2062b.cpp` 用逐字节复制的宏体 + 最小桩，`cl /c /Zc:preprocessor /std:c++20 /W4` 与 `clang++ -fsyntax-only -std=c++20` 均 exit 0）；最小用例 `if (c) for(int i=0;i<1;++i){} else ;` 同样通过（`probe_min.cpp`）。
  2. 报告的解释"`else` 无法挂到 `if` 上（`for` 不是 `if`）"**语法上就是错的**：`else` 只需挂到"尚未闭合的 `if`"，而 `for`-statement 是合法的 if 子语句（`if (c) <for-语句> else <语句>` 是合式的）。
  3. **真实的失败形态是"调用处多写一个分号"**（实测，四种变体矩阵）：`if (c) PROCESS_SCRIPT_IN(); else ;` → MSVC `error **C2181**: illegal else without matching if`、clang `error: expected expression`；而宏体**自身以 `;` 结尾**的 `PROCESS_RETURN()`（`:147`）即使调用处不写分号，放进无花括号 `if/else` 也会失败（clang 先给 `-Wdangling-else` 警告再报错）。即：**`error C2062: type 'void' unexpected` 无法复现，正确诊断是 C2181 / clang "expected expression"**。
- **可达性**: **潜伏**（FUnrealFunctionDescriptor.inl 23 处 + FCSharpDelegateDescriptor.inl 14 处 = **37 处**调用全部是"函数体顶层语句、独占一行、且不写分号"，因此当前的调用形态不会触发；只有在**新写**调用点时（尤其写成 `if (x) PROCESS_OUT(); else …`）才会踩到）
- **复核证据**: 宏体 `FunctionMacro.h:65-147` 逐字核对（`IN_END()` = 裸 `}`、`PROCESS_RETURN` 末尾自带 `;` 均确认）；调用点清单见 §0.3 与 F-MACRO-012；变体矩阵复现命令：`cl /nologo /c /Zc:preprocessor /std:c++20 /DVARIANT={1,2,3,4} probe_semi.cpp`、`clang++ -fsyntax-only -std=c++20 -DVARIANT={1,2,3,4} probe_semi.cpp`
- **级别变动**: 无（P2→P3 仍成立 —— 触发条件是"误用"而非当前代码；但证据以上面三条为准，`error C2062` 不成立）

**现状（代码事实）**
```cpp
// Macro/FunctionMacro.h:65-69
#define INITIALIZE_VALUE() \
	for (auto Index = 0; Index < PropertyDescriptors.Num(); ++Index) \
	{ \
		const auto& PropertyDescriptor = PropertyDescriptors[Index]; \
		PropertyDescriptor->InitializeValue_InContainer(Params);
// Macro/FunctionMacro.h:81-82
#define IN_END() \
	}
// Macro/FunctionMacro.h:84-87
#define PROCESS_SCRIPT_IN() \
	INITIALIZE_VALUE() \
	IN_VALUE() \
	IN_END()
```
即 `PROCESS_SCRIPT_IN()` 展开后是一个 **裸 `for` 语句**（末尾 `}`，不带 `;`）；`PROCESS_OUT()` 是裸 `for`；`PROCESS_NATIVE_REFERENCE_IN()` 先声明 `auto LastOut = &Stack.OutParms;` 再跟 `for`；`PROCESS_RETURN()` 是 `if constexpr/else if constexpr` 链 + 一条独立语句。

**调用上下文**
**37 处**调用全部位于函数体顶层、独占一行、**且不写分号**（与"末尾是 `}`"的设计一致）：
- `FUnrealFunctionDescriptor.inl` **23 处**：`:19,27,37,41,51,61,63,71,75,84,88,90,110,120,132,136,146,150,161,165,167,194,241`（其中 `PROCESS_RETURN` 7 处、`PROCESS_SCRIPT_IN` 4 处、`PROCESS_NATIVE_REFERENCE_IN` 4 处、`PROCESS_OUT` 6 处、`PROCESS_SCRIPT_REFERENCE_IN` 2 处）
- `FCSharpDelegateDescriptor.inl` **14 处**：`:20,28,39,43,53,62,66,75,79,81,100,121,130,138`（其中 `PROCESS_RETURN` 3 处：`:20,43,81`；`PROCESS_SCRIPT_IN` 3 处：`:28,39,100`；`PROCESS_OUT` 5 处：`:53,66,79,121,138`；`PROCESS_SCRIPT_REFERENCE_IN` 3 处：`:62,75,130`）
（完整清单 23 + 14 = 37，`grep 'PROCESS_[A-Z_]*\(\)'` 于 `Public/Reflection/Function/` 命中数 = 37。）

**问题**
探针（逐字节复制宏体 `FunctionMacro.h:65-87` + 最小可编译桩）：

```cpp
void probe_dangling_else(int c)
{
	if (c)
		PROCESS_SCRIPT_IN()      // 注意：宏末尾是 '}'，调用处没有分号
	else
		;
}
```
展开（`cl /nologo /EP /Zc:preprocessor` 实际输出，逐字）：
```cpp
	if (c)
		for (auto Index = 0; Index < PropertyDescriptors.Num(); ++Index) { const auto& PropertyDescriptor = PropertyDescriptors[Index]; PropertyDescriptor->InitializeValue_InContainer(Params); PropertyDescriptor->Set(InBuffer, PropertyDescriptor->ContainerPtrToValuePtr<void>(Params)); InBuffer += PropertyDescriptor->GetBufferSize(); } 
	else
		;
```

**"放进无花括号 `if/else` 即编译失败（`error C2062`）"这一结论不成立**：上面这段展开在 MSVC 14.51（`/c /Zc:preprocessor /std:c++20 /W4`）与 clang 20.0.0git（`-fsyntax-only -std=c++20`）下**都编译通过、零诊断**（`probe_c2062b.cpp`、`probe_min.cpp`）。原因是 `else` 只要求挂到"尚未闭合的 `if`"，而 `for`-statement 是合法的 if 子语句，所以 `if (c) <for-语句> else <语句>` 本身就是合式的 —— "`for` 不是 `if`，所以 `else` 挂不上"的推理是错的。

**真正会失败的两种写法**（四变体矩阵实测，`probe_semi.cpp`）：

| 变体 | 写法 | MSVC 14.51 | clang 20.0.0git |
|---|---|---|---|
| 1 | `if (c) PROCESS_SCRIPT_IN() else ;`（无分号，= 报告原探针） | **通过** | **通过** |
| 2 | `if (c) PROCESS_SCRIPT_IN(); else ;`（调用处补分号） | `error **C2181**: illegal else without matching if` | `error: expected expression` |
| 3 | `if (c) PROCESS_RETURN() else ;`（宏体自身以 `;` 结尾） | `error C2181` | `-Wdangling-else` 警告 + 报错 |
| 4 | 顶层 `PROCESS_SCRIPT_IN();`（补分号） | 通过（多一个空语句） | 通过 |

即：**失败条件是"分号/宏体结尾形态"，不是"无花括号 if/else"本身；诊断号是 `C2181`（clang: expected expression），不是 `C2062`**。`PROCESS_RETURN()` 末尾自带 `;`（`:147`）与其余 9 个宏风格不一致这一点仍然成立，且它正好是变体 3 的成因。

当前 **37 处**调用（`FUnrealFunctionDescriptor.inl` 23 + `FCSharpDelegateDescriptor.inl` 14）全部是顶层语句、独占一行、不写分号 → **不触发**。但这些宏放在 `Public/Macro/` 下、对游戏侧手写绑定代码可见（本工程 `Source/UnrealCSharpTest/**` 已在用 `BINDING_CLASS`/`BINDING_ENUM`），任何人把它写成 `if (bXXX) PROCESS_OUT(); else ...` 都会得到一条完全不提宏名的语法错误 —— 隐患真实，但触发条件是上表列出的两种写法。

**建议**
统一包成 `do { ... } while (0)`（同时把 `PROCESS_RETURN` 末尾的 `;` 收进 `while(0)`）：
```cpp
#define PROCESS_SCRIPT_IN() \
	do { \
		INITIALIZE_VALUE() \
		IN_VALUE() \
		IN_END() \
	} while (0)
```
同时把 `IN_END()` 的 `}` 从"配对宏"改为语义化命名（如 `PROCESS_END()`），减少"宏里放半个括号"的用法（这类配对宏一旦漏一个就是灾难性语法错误）。注意 `PROCESS_RETURN()` 内部的 `if constexpr` 不能放进 `do{}while(0)` 后会失去 `constexpr` 语义吗？不会 —— `if constexpr` 在 `do{}while(0)` 内仍然合法。

**验证方式**
```
cl /nologo /c /Zc:preprocessor /std:c++20 /DVARIANT=1 probe_semi.cpp   # 变体1（无分号）：期望 0 error（实测通过）
cl /nologo /c /Zc:preprocessor /std:c++20 /DVARIANT=2 probe_semi.cpp   # 变体2（补分号）：期望 C2181（实测命中）
clang++ -fsyntax-only -std=c++20 -DVARIANT=2 probe_semi.cpp            # 期望 "expected expression"（实测命中）
```
（`probe_semi.cpp` 用 `#if VARIANT == N` 切换四种写法，宏体逐字节取自 `FunctionMacro.h:65-87`。）
回归 grep：`grep 'PROCESS_[A-Z_]*\(\)'` 在 `Source/UnrealCSharp/Public/Reflection/Function/` 下共 **37** 处（`FUnrealFunctionDescriptor.inl` 23 + `FCSharpDelegateDescriptor.inl` 14），确认全部是顶层语句且不写分号。

---

### P3（续：F-MACRO-007～013）

### [F-MACRO-007] `BINDING_PROPERTY` / `BINDING_READONLY_PROPERTY` 的额外实参在非 Editor 构建被静默丢弃

- **类别**: 可优化/可读性（隐患）
- **严重度**: **P3**
- **文件**: `Source/UnrealCSharp/Public/Macro/BindingMacro.h:250-260`
- **函数**: 宏 `BINDING_PROPERTY(Property, ...)`、`BINDING_READONLY_PROPERTY(Property, ...)`
- **置信度**: 高（已追到唯一使用该实参的调用点与消费方）
- **复核结论**: ✅确认（严重度 P3 维持；"`EPropertyInteract` 只在 Editor 期影响生成的 C# 文本、Shipping 下被静默丢弃"的链路已从头到尾走通，且**唯一值消费者确为 Editor 模块的代码生成器**）
- **可达性**: **活跃**（`FRegisterKeys.cpp:117` 在 Editor 构建下确实传入额外实参；Shipping 构建下 `WITH_PROPERTY_INFO`=0 → 该实参被宏整段丢弃，两态都真实发生）
- **复核证据**: 宏体 `Macro/BindingMacro.h:250-254`/`:256-260` 逐字核对；链路逐跳核实 —— `FRegisterKeys.cpp:117`（唯一带实参的调用点，`grep 'BINDING_PROPERTY(&'` 只有它带第二个实参）→ `FClassBuilder.h:35-38` + `FClassBuilder.inl:26-29`（两个形参都在 `#if WITH_PROPERTY_INFO` 内）→ `FBindingClassRegister.cpp:86,98` → `FBindingPropertyRegister.h:12,16,23` → `FBindingProperty.h:10,28-30,38` → **`FBindingClassGenerator.cpp:265-272`**（`return PropertyInteract[Property.GetPropertyInteract()]`，`{EPropertyInteract::None, ""}`/`{EPropertyInteract::New, "new "}`）。`grep 'GetPropertyInteract'` 全插件**只有这一处读取**，而 `ScriptCodeGenerator` 是 Editor 模块 → "当前无功能影响"成立。实际映射表是 `:265-269`、**读取点是 `:272`**
- **级别变动**: 无（P3 维持：门面 API 的静默丢弃隐患，当前无功能影响）

**现状（代码事实）**
```cpp
// Macro/BindingMacro.h:250-254
#if WITH_PROPERTY_INFO
#define BINDING_PROPERTY(Property, ...) BINDING_PROPERTY_BUILDER_GET(Property), BINDING_PROPERTY_BUILDER_SET(Property), BINDING_PROPERTY_BUILDER_INFO(Property), ##__VA_ARGS__
#else
#define BINDING_PROPERTY(Property, ...) BINDING_PROPERTY_BUILDER_GET(Property), BINDING_PROPERTY_BUILDER_SET(Property)
#endif
```
全插件唯一传额外实参的调用点：
```cpp
// Source/UnrealCSharp/Private/Domain/Interop/FRegisterKeys.cpp:117
				.Property("Equals", BINDING_PROPERTY(&EKeys::Equals, EPropertyInteract::New))
```

**调用上下文**
消费方 `FClassBuilder::Property(..., const TOptional<FTypeInfo*()>&, const TOptional<EPropertyInteract>&)`（`FClassBuilder.h:37`）——这两个形参本身就在 `#if WITH_PROPERTY_INFO` 内（`FClassBuilder.inl:28`），所以 Shipping 下 `.Property(name, GET, SET)` 与重载匹配、`EPropertyInteract::New` 文本被宏整段丢弃。
`EPropertyInteract` 的唯一消费者是代码生成器：`ScriptCodeGenerator/Private/FBindingClassGenerator.cpp:266-269`（`{EPropertyInteract::New, TEXT("new ")}`），而 `ScriptCodeGenerator` 是 **Editor 模块**。

**问题**
**当前无功能影响**（我把链路追到底了，这里不是 bug）：`EPropertyInteract` 只影响 Editor 期生成的 C# 源码文本。
但作为门面 API 的隐患是真实的：`BINDING_PROPERTY(P, X)` 在 Editor 下 `X` 生效、在 Shipping 下静默消失，**没有 `static_assert`/`#error` 保护**；若将来有人把某个**运行期**行为挂在 `__VA_ARGS__` 上（比如抄一份 `EPropertyInteract` 语义到运行时），Editor 与 Shipping 会给出不同行为，而且编译完全通过。

**建议**
Shipping 分支显式拒绝多余实参，把"静默丢弃"变成编译错误：
```cpp
#else
#define BINDING_PROPERTY(Property, ...) \
	static_assert(TSizeof_Args(__VA_ARGS__) == 0, "BINDING_PROPERTY 的元数据实参仅在 WITH_EDITOR 下有效"); \
	BINDING_PROPERTY_BUILDER_GET(Property), BINDING_PROPERTY_BUILDER_SET(Property)
#endif
```
（或保持现状，但在 `BindingMacro.h:250` 上方补一条注释说明"多余实参在非 Editor 下被有意丢弃"。）

**验证方式**
`grep -rn 'BINDING_PROPERTY(&' Source | grep -c ','` → 1（即 `FRegisterKeys.cpp:117`）；`grep -rn 'EPropertyInteract' Source` → 确认只有 Editor 侧代码生成器读取。

---

### [F-MACRO-008] 当前 UE 5.6 配置下的空展开宏与零调用宏（版本门控死代码）

- **类别**: 死代码
- **严重度**: **P3**
- **文件**: `Source/UnrealCSharp/Public/Macro/BindingMacro.h:236-242`（`BINDING_REGULAR_UENUM`）、`:110-116`（`BINDING_STRUCT`）、`:282-286`（`BINDING_OVERLOADS`）
- **函数**: 同上
- **置信度**: 高（grep 命中数 + 版本宏取值 + 条件编译链全部核对）
- **复核结论**: ✅确认（严重度 P3 维持；三个宏的"当前无效"性质与分类**全部复核成立**）
- **可达性**: **不可达**（`BINDING_OVERLOADS`、`BINDING_REGULAR_UENUM`）**+ 潜伏**（`BINDING_STRUCT`）—— 前两个在全工程**从无调用点**（任何配置下都到不了 = 死宏），`BINDING_STRUCT` 的 6 个调用点全被 `#if UE_U_STRUCT_*`（UE 5.6 下为 0）屏蔽，切到 UE 5.7 即生效，属"潜伏"
- **复核证据**: `grep '\bBINDING_OVERLOADS\b'` = **2**（均 `#define`：`:283`/`:285`）、`grep '\bBINDING_REGULAR_UENUM\b'` = **2**（均 `#define`：`:237`/`:241`）、`grep '\bBINDING_STRUCT\b'` = **7**（1 `#define` `:110` + 6 调用：`FRegisterSoftObjectPath.cpp:8`、`FRegisterSoftClassPath.cpp:9`、`FRegisterPrimaryAssetType.cpp:8`、`FRegisterPrimaryAssetId.cpp:8`、`FRegisterAssetBundleData.cpp:9`、`FRegisterAssetBundleEntry.cpp:8`）—— 三个数已全部复现；版本宏 `UE_REGULAR_UENUM_STATIC_ENUM`（`UEVersion.h:206`）= `UE_VERSION_START(5,8,0)`、7 个 `UE_U_STRUCT_*`（`UEVersion.h:176-188`）= `UE_VERSION_START(5,7,0)` 全部核实；另注 `TOverloadBuilder` 类本身（`TOverloadBuilder.inl:1-62`）亦无使用者，其 `#else` 分支的 `Get` 在 `:52-60`
- **级别变动**: 无（P3 维持：死代码/版本门控，无运行时影响）

**现状（代码事实）**
```cpp
// Macro/BindingMacro.h:236-242
#if UE_REGULAR_UENUM_STATIC_ENUM
#define BINDING_REGULAR_UENUM(Class, ...) \
template <> \
__VA_ARGS__ UEnum* StaticEnum<Class>();
#else
#define BINDING_REGULAR_UENUM(Class, ...)
#endif
// Macro/BindingMacro.h:282-286
#if WITH_FUNCTION_INFO
#define BINDING_OVERLOADS(...) TOverloadBuilder<void*, TFunction<FFunctionInfo*()>>::Get(__VA_ARGS__)
#else
#define BINDING_OVERLOADS(...) TOverloadBuilder<void*>::Get(##__VA_ARGS__)
#endif
```
```cpp
// Source/CrossVersion/Public/UEVersion.h:206
#define UE_REGULAR_UENUM_STATIC_ENUM UE_VERSION_START(5, 8, 0)
// UEVersion.h:176-188
#define UE_U_STRUCT_F_SOFT_OBJECT_PATH UE_VERSION_START(5, 7, 0)   // 另 6 个同值
```

**调用上下文**
- `BINDING_REGULAR_UENUM`：全插件 `\bBINDING_REGULAR_UENUM\b` = **2 命中，均为 `#define` 自身**（`:237`、`:241`）→ 从无调用；且 `UE_REGULAR_UENUM_STATIC_ENUM` 在 UE 5.6 为 0 → 即使被调用也展开为空。
- `BINDING_OVERLOADS`：`\bBINDING_OVERLOADS\b` = **2 命中，均为 `#define`**（`:283`、`:285`）→ 从无调用（`TOverloadBuilder` 类本身存在但无人使用）。
- `BINDING_STRUCT`：`\bBINDING_STRUCT\b` = **7 命中 = 1 定义 + 6 调用**，6 个调用全部在 `#if UE_U_STRUCT_*` 内（`FRegisterSoftObjectPath.cpp:7-8`、`FRegisterSoftClassPath.cpp:8-9`、`FRegisterPrimaryAssetType.cpp:7-8`、`FRegisterPrimaryAssetId.cpp:7-8`、`FRegisterAssetBundleData.cpp:8-9`、`FRegisterAssetBundleEntry.cpp:7-8`），而这些标志在 UE 5.6 全为 0 → **6 处全部不展开**。

**问题**
三类"当前无效"的宏混在一起，容易被误判为可删除：
- `BINDING_OVERLOADS` 是**真正的死宏**（无调用、无条件）→ 可删。
- `BINDING_REGULAR_UENUM` 是**版本门控为空**（UE 5.8+ 才有效）→ 保留，但应加注释说明当前恒为空。
- `BINDING_STRUCT` 是**版本门控调用点**（UE 5.7+ 才展开）→ 必须保留；同时注意它与 `#if !UE_U_STRUCT_*` 的 `BINDING_SCRIPT_STRUCT` 是互斥对（`TScriptStruct.inl:47-83`），删任何一个都会破坏 5.7+ 的构建。

**建议**
1. 删除 `BINDING_OVERLOADS`（连同 `TOverloadBuilder.inl` 若无其他用途，另见专项死代码报告）。
2. 在 `:236` 与 `:110` 上方加注释：`// 注意：UE 5.6 下本宏展开为空；有效起始版本见 UEVersion.h:206 / :176`。
3. 若希望 CI 能发现"门控宏长期为空"，可加一条构建期统计（`/showIncludes` 或 `cl /P` 后 grep）。

**验证方式**
```powershell
Select-String -Path (Get-ChildItem -Recurse -Include *.h,*.cpp,*.inl Source) -Pattern '\bBINDING_OVERLOADS\b'   # 期望 2（仅定义）
Select-String -Path ... -Pattern '\bBINDING_STRUCT\('                                                          # 期望 6，逐个确认处于 #if UE_U_STRUCT_ 内
```

---

### [F-MACRO-009] 名称家族混淆：`BINDING_PROPERTY` / `BINDING_PROPERTY_SET` / `BINDING_PROPERTY_BUILDER_SET` 三名一义各异；`REMOVE_LEFT/RIGHT` 命名与语义相反；3 个同体拼接宏

- **类别**: 可优化/可读性（一致性）
- **严重度**: **P3**
- **文件**: `Source/UnrealCSharp/Public/Macro/BindingMacro.h:41-47,244-248`、`Source/UnrealCSharpCore/Public/CoreMacro/BindingMacro.h:3-13`、`Source/UnrealCSharpCore/Public/CoreMacro/NamespaceMacro.h:3`、`Source/UnrealCSharpCore/Public/CoreMacro/ClassMacro.h:5`
- **函数**: 宏常量族
- **置信度**: 高
- **复核结论**: ✅确认（严重度 P3 维持；四条子结论逐条回源码核实）
- **可达性**: **活跃**（这些宏名/常量在编译期与运行期都被真实使用：`COMBINE_NAMESPACE` 308 次、`COMBINE_FULL_NAME` 16 次、`BINDING_COMBINE_CLASS` 1 次、`NAMESPACE_BINDING` 35 次、`BINDING_NAME` 1 次，均为 grep 实测）
- **复核证据**: ① `Macro/BindingMacro.h:41-47`（两个 `REMOVE_*_NAMESPACE_*` 的命名与返回侧相反）与 `:244-248` 逐字核对；② `CoreMacro/BindingMacro.h:3`（`BINDING_COMBINE_CLASS`）与 `CoreMacro/NamespaceMacro.h:3`（`COMBINE_NAMESPACE`）、`CoreMacro/ClassMacro.h:5`（`COMBINE_FULL_NAME`）三者函数体**逐字节相同**（`FString::Printf(TEXT("%s.%s"), *A, *B)`）—— 核实成立；③ 计数重跑：`\bCOMBINE_NAMESPACE\b`=309（1 定义 + 308 使用）、`\bCOMBINE_FULL_NAME\b`=17（1 + 16）、`\bBINDING_COMBINE_CLASS\b`=2（1 定义 `CoreMacro/BindingMacro.h:3` + 1 使用 `FBindingClassRegister.cpp:121`）、`\bNAMESPACE_BINDING\b`=36（1 + 35）、`\bBINDING_NAME\b`=2（1 定义 `CoreMacro/Macro.h:49` + 1 使用 `FUnrealCSharpFunctionLibrary.cpp:900`）—— 与 §1.1/§4 的每个数字**逐一吻合**；④ `BINDING_PROPERTY_SET`/`_GET`（`CoreMacro/BindingMacro.h:11,13`）与 `BINDING_PROPERTY_BUILDER_SET`/`_GET`（`Macro/BindingMacro.h:244,246`）确实只差 `_BUILDER`，前者是 `FString` 常量、后者是函数指针
- **级别变动**: 无（P3 维持：命名/一致性，无功能影响）

**现状（代码事实）**
```cpp
// UnrealCSharpCore/Public/CoreMacro/BindingMacro.h:3-13
#define BINDING_COMBINE_CLASS(A, B) FString::Printf(TEXT("%s.%s"), *A, *B)
#define BINDING_PROPERTY_SET FString(TEXT("Set"))
#define BINDING_PROPERTY_GET FString(TEXT("Get"))
// UnrealCSharp/Public/Macro/BindingMacro.h:41-47
#define BINDING_REMOVE_LEFT_NAMESPACE_CLASS_STR(Class) F_STRING_STR(Class).Left(F_STRING_STR(Class).Find(TEXT("::")))      // 返回 :: 左侧（命名空间），不是"移除左侧"
#define BINDING_REMOVE_RIGHT_NAMESPACE_CLASS_STR(Class) F_STRING_STR(Class).Right(F_STRING_STR(Class).Len() - F_STRING_STR(Class).Find(TEXT("::")) - 2)  // 返回 :: 右侧（类名）
// UnrealCSharp/Public/Macro/BindingMacro.h:244-246
#define BINDING_PROPERTY_BUILDER_SET(Property) TPropertyBuilder<decltype(Property), Property>::Set
#define BINDING_PROPERTY_BUILDER_GET(Property) static_cast<void(*)(BINDING_PROPERTY_BUILDER_GET_SIGNATURE)>(...)
```

**调用上下文**
- `BINDING_PROPERTY_SET`/`BINDING_PROPERTY_GET`（`FString("Set")`/`FString("Get")`）各 3 处使用，**与** `BINDING_PROPERTY_BUILDER_SET/GET`（函数指针）**只差一个 `_BUILDER`**，却在语义上一个是字符串常量、一个是可调用指针。
- `BINDING_PROPERTY` 本身是**初始化列表片段**（不是"属性"）；`BINDING_PROPERTY_BUILDER_GET_SIGNATURE`（`SignatureMacro.h:26`）是**形参列表**。
- `COMBINE_NAMESPACE`(308 次) / `COMBINE_FULL_NAME`(16 次) / `BINDING_COMBINE_CLASS`(1 次) 三个宏函数体完全相同。
- `BINDING_NAME`（`CoreMacro/Macro.h:49`，1 处使用）与 `NAMESPACE_BINDING`（`Macro/NamespaceMacro.h:3`，35 处使用）都是 `FString(TEXT("Binding"))`，跨模块重复。

**问题**
1. `BINDING_REMOVE_LEFT_NAMESPACE_CLASS_STR` 实际返回"`::` 左边那段"（即保留命名空间、丢掉类名），`..._RIGHT_...` 返回"`::` 右边那段"。名字里的 "REMOVE" 修饰的是**被丢掉的部分**，与读代码时的直觉（"移除哪边"）恰好相反；`BINDING_CLASS` 的三元表达式 `TGet_Args<0>(...) ? LEFT : RIGHT` 读起来像是在"选要移除的命名空间"，极易误用（这正是 F-MACRO-002 静默错误的温床）。
2. 同一概念（`"Set"`/`"Get"` 字符串 vs `Set`/`Get` 函数指针）用只差一个词的宏名表达，`grep BINDING_PROPERTY_SET` 会同时命中两类，可维护性差。
3. 三个同体宏导致 `grep` 统计"类全名拼接"必须搜 3 个名字。

**建议**
- 重命名（保留旧名做 `#define` 兼容过渡）：`BINDING_TAKE_NAMESPACE_PART` / `BINDING_TAKE_CLASS_PART`，或在宏上补一行注释说明返回的是哪一侧。
- 统一 `COMBINE_NAMESPACE`/`COMBINE_FULL_NAME`/`BINDING_COMBINE_CLASS` 为 1 个；`BINDING_NAME` 与 `NAMESPACE_BINDING` 合并。
- 把 `BINDING_PROPERTY_SET/GET` 改名为 `FUNCTION_NAME_PROPERTY_SET/GET`（与 `CoreMacro/FunctionMacro.h` 的 `FUNCTION_*` 家族一致），或直接复用 `BINDING_PROPERTY_BUILDER_SET/GET` 的邻居命名规范。

**验证方式**
`grep -rn 'BINDING_PROPERTY_SET\|BINDING_PROPERTY_GET' Source` → 应只命中定义 + 3 处字符串使用；重命名后重跑生成的 C# 绑定代码 diff（应完全一致）。

---

### [F-MACRO-010] `BINDING_REMOVE_PREFIX_CLASS_STR` 用 `RightChop(1)` 盲删首字符：当前 26 处恰好都为 `F` 前缀

- **类别**: 可优化/可读性（隐含约定）
- **严重度**: **P3**
- **文件**: `Source/UnrealCSharp/Public/Macro/BindingMacro.h:47`
- **函数**: 宏 `BINDING_REMOVE_PREFIX_CLASS_STR(Class)`
- **置信度**: 高
- **复核结论**: ✅确认（严重度 P3 维持；"26 处调用、实参全部以 `F` 开头"与"`RightChop(1)` 与语义脱钩"两点均成立）
- **可达性**: **活跃**（26 处调用分布在 `TBaseStructure.inl:41…284`，是 C# 侧访问 `FIntPoint`/`FTimespan` 等基础结构体时的**运行期**查名路径）
- **复核证据**: `grep 'BINDING_REMOVE_PREFIX_CLASS_STR\('` = **27** 命中 = 1 定义（`Macro/BindingMacro.h:47`）+ **26 处调用**，调用行号与报告列举的 26 个**逐个吻合**（`TBaseStructure.inl:41,51,60,69,78,88,98,107,116,125,134,143,153,163,173,184,195,206,217,228,238,247,256,265,274,284`），且 26 个实参**全部**以 `F` 开头（`FIntPoint`、`FTimespan`、`FAssetBundleEntry`…`FStrataMaterialInput`）；落地路径 `TBaseStructure.inl:41 → StaticGetBaseStructureInternal`（`:7-33`，其中 `:24-30` 是 `#if !(UE_BUILD_SHIPPING || UE_BUILD_TEST)` 内的 `UE_LOG(LogClass, Fatal, …)`）逐字核实 —— "Shipping 下静默返回 nullptr"成立
- **级别变动**: 无（P3 维持：隐含约定 + 名字误导）

**现状（代码事实）**
```cpp
// Macro/BindingMacro.h:47
#define BINDING_REMOVE_PREFIX_CLASS_STR(Class) F_STRING_STR(Class).RightChop(1)
```

**调用上下文**
全部 26 处调用在 `Source/UnrealCSharp/Public/Binding/ScriptStruct/TBaseStructure.inl:41,51,60,69,78,88,98,107,116,125,134,143,153,163,173,184,195,206,217,228,238,247,256,265,274,284`，形态统一为
```cpp
return StaticGetBaseStructureInternal(*BINDING_REMOVE_PREFIX_CLASS_STR(FIntPoint));
```
即 `FString("FIntPoint").RightChop(1)` → `FName("IntPoint")`，用运行时名字查找 `UScriptStruct`。

**问题**
`RightChop(1)` 只是"删掉第 1 个字符"，它之所以等于"去掉 `F` 前缀"，完全依赖**这 26 个实参恰好都以 `F` 开头**（我逐个核对了 grep 输出：全部是 `F*`）。一旦有人写成 `BINDING_REMOVE_PREFIX_CLASS_STR(TBaseStructure<FVector>)`、写成 typedef 名，或新增一个不以 `F` 开头的脚本结构体（例如 `TVector2<float>` 或 `UE::Math::TVector<double>`），结果会被静默截断成错误名字（`TEXT_STR` 还会把模板参数里的逗号/空格原样 stringize 进 FString），最终走 `TBaseStructure` 查不到 → `StaticGetBaseStructureInternal` 只在 `!(UE_BUILD_SHIPPING || UE_BUILD_TEST)` 下 `UE_LOG(Fatal)`（`TBaseStructure.inl:24-30`），**Shipping 下静默返回 nullptr** 再被调用方解引用。名字里的 `CLASS_STR` 也暗示"字符串形式的类名"，与实际语义（"去掉 1 个字符"）不符。

**建议**
改成显式判断，失败即报错：
```cpp
#define BINDING_REMOVE_PREFIX_CLASS_STR(Class) \
	([] { const FString S = F_STRING_STR(Class); ensureMsgf(S.StartsWith(TEXT("F")), TEXT("%s 不以 F 开头"), *S); return S.RightChop(1); }())
```
或至少把宏改名为 `BINDING_CHOP_FIRST_CHAR_STR`，避免误导。

**验证方式**
`grep -rn 'BINDING_REMOVE_PREFIX_CLASS_STR(' Source` → 26 处，确认实参均以 `F` 开头（当前成立）。

---

### [F-MACRO-011] `TFunctionPointer<...>().Value.Pointer` 联合体类型双关，无 `sizeof` 断言

- **类别**: 未定义行为
- **严重度**: **P3**
- **文件**: `Source/UnrealCSharp/Public/Macro/BindingMacro.h:262`（`BINDING_FUNCTION_BUILDER_INVOKE`）、`:272`、`:288`、`:298`、`:308`、`:310`
- **函数**: 上述宏生成的 `TFunctionPointer<T>(...).Value.Pointer` 表达式
- **置信度**: 高（UB 是事实；实际无害亦为事实）
- **复核结论**: ✅确认（严重度 P3 维持；`TFunctionPointer` 的联合体双关与"无 `sizeof` 断言"两点复核成立）
- **可达性**: **活跃**（所有 6 个宏生成的表达式都在每次绑定注册时求值；`Value.Pointer` 会被写进 `FBindingClassRegister`）
- **复核证据**: `Macro/BindingMacro.h:262,272,288,298,308,310` 六个 `..._BUILDER_INVOKE/_GET/_SET` 宏逐字核对（都读 `.Value.Pointer`）；`TFunctionPointer.inl:3-17` 的 `union { T Function; void* Pointer; }` 与 `explicit TFunctionPointer(const T&)` 在 `:6-16` 核实；不变量"`T` 恒为普通静态函数指针"实测：`grep 'static auto Invoke' TFunctionBuilder.inl` = `:16,28,40` 三处**均为 `static`**（`TDestructorBuilder.inl:10` 是 `static void Invoke`、`TConstructorBuilder.inl:10`/`TSubscriptBuilder.inl:14,19` 同）→ 当前 8 字节同宽成立。**调用点计数口径**（两个数各按一种口径，逐项标注）：`BINDING_FUNCTION(` **647** 命中（含 2 `#define` + 1 宏内部使用）、`BINDING_OVERLOAD(` **95** 命中（含 2 `#define` + 1 处生成器字符串 `SourceCodeGenerator.cs:517`）、`BINDING_CONSTRUCTOR(` **93** 命中（含 2 `#define` + `TBindingClassBuilder.inl:18` + 90 处 `FRegister*.cpp`）、`BINDING_DESTRUCTOR(` **3** 命中（2 `#define` + **1** 处调用）、`BINDING_SUBSCRIPT(` **7** 命中（2 `#define` + **5** 处调用：`FRegisterBox2D.cpp:27`、`FRegisterGuid.cpp:55`、`FRegisterVector4.cpp:70`、`FRegisterVector2D.cpp:87`、`FRegisterVector.cpp:108`）
- **级别变动**: 无（P3 维持：标准 UB 但三家编译器均支持、当前形态安全）

**现状（代码事实）**
```cpp
// Macro/BindingMacro.h:262
#define BINDING_FUNCTION_BUILDER_INVOKE(Function) TFunctionPointer<decltype(&TFunctionBuilder<decltype(Function), Function>::Invoke)>(&TFunctionBuilder<decltype(Function), Function>::Invoke).Value.Pointer
```
```cpp
// UnrealCSharpCore/Public/Template/TFunctionPointer.inl:3-17
template <typename T>
struct TFunctionPointer
{
	explicit TFunctionPointer(const T& InFunction) { Value.Function = InFunction; }
	union { T Function; void* Pointer; } Value;
};
```

**调用上下文**
被 `BINDING_FUNCTION`(647 处)、`BINDING_OVERLOAD`(94 处)、`BINDING_CONSTRUCTOR`(93 处)、`BINDING_DESTRUCTOR`(1 处)、`BINDING_SUBSCRIPT`(7 处) 间接使用。

**问题**
"写 `Value.Function`、读 `Value.Pointer`"在 ISO C++ 中是 UB（联合体双关只在 C 与各家编译器扩展下成立；MSVC/GCC/Clang 都支持，因此实践中无害）。真正的风险是**没有尺寸/形态约束**：当前 `T` 恒为普通静态函数指针 `void (*)(...)`（`TFunctionBuilder.inl:16,28,40` 的 `Invoke` 都是 `static`），8 字节与 `void*` 同宽所以可行；但若将来有人把 `Invoke` 写成非静态成员函数指针（MSVC 下 8/16 字节，多重继承时 16 字节），`Value.Pointer` 会**静默读到半个指针** —— 变成一个"看起来能编译、调用时必崩"的绑定。

**建议**
```cpp
template <typename T>
struct TFunctionPointer
{
	static_assert(std::is_pointer_v<T> && std::is_function_v<std::remove_pointer_t<T>>,
	              "只支持普通函数指针");
	static_assert(sizeof(T) == sizeof(void*), "函数指针宽度与 void* 不一致，不能做联合体双关");
	...
};
```
若需彻底消除 UB，可改为 `std::bit_cast<void*>(InFunction)`（C++20，UE 5.6 已启用 C++20）。

**验证方式**
`grep -n 'static auto Invoke' Source/UnrealCSharp/Public/Binding/Function/TFunctionBuilder.inl` → 3 处均为 `static`（当前不变量成立）。

---

### [F-MACRO-012] `PROCESS_RETURN()` 隐式依赖调用点局部名 `ReturnType` / `BufferAllocator` / `Params` / `PropertyDescriptors`

- **类别**: 可优化/可读性（宏卫生）
- **严重度**: **P3**
- **文件**: `Source/UnrealCSharp/Public/Macro/FunctionMacro.h:136-147`（另 65-134 的 8 个宏同样）
- **函数**: `PROCESS_RETURN()` 等
- **置信度**: 高
- **复核证据**: `FunctionMacro.h:65-147` 的 10 个宏逐字核对：宏体里出现的 `ReturnType`、`BufferAllocator`、`Params`、`PropertyDescriptors`、`OutPropertyIndexes`、`ReferencePropertyIndexes`、`Stack`、`IN_BUFFER`/`OUT_BUFFER`/`RETURN_BUFFER` **全部不是宏形参**，只能由展开点作用域提供 —— 成立；其中 `ReturnType` 是 `FUnrealFunctionDescriptor.inl:6` 的 `template <auto ReturnType>` 模板形参，`FCSharpDelegateDescriptor.inl:7` 同名 —— 两个 TU 都能编译说明两边都恰好有同名形参
- **复核结论**: ✅确认（严重度 P3 维持；"宏实现与调用点作用域强耦合"是事实，且 10 个宏全部如此）
- **可达性**: **活跃**（37 处调用全部在编译期展开，作用于真实运行期函数）
- **级别变动**: 无（P3 维持：宏卫生/可读性）

**现状（代码事实）**
```cpp
// Macro/FunctionMacro.h:137
	if constexpr (ReturnType == EFunctionReturnType::Primitive) \
```
此处的 `ReturnType` **不是宏参数**，而是展开点所在模板的模板形参名（`FUnrealFunctionDescriptor.inl:6 template <auto ReturnType>`）。同类隐式依赖还包括 `BufferAllocator`、`Params`、`PropertyDescriptors`、`OutPropertyIndexes`、`ReferencePropertyIndexes`、`Function`、`Stack`、以及 `CoreMacro/BufferMacro.h` 的 `IN_BUFFER`/`OUT_BUFFER`/`RETURN_BUFFER`。

**调用上下文**
`PROCESS_RETURN` 的 10 处调用分布在两个类：`FUnrealFunctionDescriptor.inl:19,41,63,90,110,136,167` 与 `FCSharpDelegateDescriptor.inl`（3 处）。两处都能编译说明这些类都恰好有同名成员/模板形参。

**问题**
宏的实现与调用点作用域强耦合（"隐式环境"），带来的具体损失：
1. 无法在独立 TU 里理解 `PROCESS_RETURN()` 的语义，也无法为它写单元测试；
2. 若 `FCSharpDelegateDescriptor` 把模板形参 `ReturnType` 改名（或新增一个同名局部变量 `ReturnType`），宏会**静默改指**到局部变量上 —— 若该局部变量不是编译期常量，`if constexpr` 直接编译失败，若恰好是常量则**语义漂移**；
3. 宏体里出现 `BufferAllocator`/`Params` 这类普通标识符（非 `MACRO` 风格大写），阅读时无法区分"宏引入"还是"作用域提供"，F-MACRO-005 的疏漏正是这种不可见性的产物。

**建议**
把关键标识符提升为宏形参（至少 `ReturnType`）：
```cpp
#define PROCESS_RETURN(ReturnTypeEnum) \
	if constexpr (ReturnTypeEnum == EFunctionReturnType::Primitive) { ... }
// 调用：PROCESS_RETURN(ReturnType)
```
或在宏体里用 `static_assert` 明确前置条件，并统一把作用域提供的名字写成 `IN_*`/`_Params` 之类可辨识形式。

**验证方式**
把 `FCSharpDelegateDescriptor.inl` 的模板形参临时改名编译一次（预期报错），以确认耦合范围。

---

### [F-MACRO-013] `SignatureMacro.h` 的 `*_SIGNATURE`/`*_PARAM` 成对宏没有任何一致性保护

- **类别**: 可优化/可读性（一致性）
- **严重度**: **P3**
- **文件**: `Source/UnrealCSharp/Public/Macro/SignatureMacro.h:6-28`
- **函数**: 12 个宏
- **置信度**: 高
- **复核结论**: ✅确认（严重度 P3 维持；12 个宏的 `_SIGNATURE`/`_PARAM` 配对确实没有任何编译期一致性保护）
- **可达性**: **活跃**（12 个宏全部被使用，且是 C++↔C# ABI 的形参/实参两侧）
- **复核证据**: `SignatureMacro.h:6,8,10,12,14,16,18,20,22,24,26,28` = **12 个宏**（6 对）逐字核对；配对使用点 grep 实测（**按出现次数**统计）：`BINDING_FUNCTION_SIGNATURE` **6**（定义 + `TFunctionHelper.inl:19,37` + `TFunctionBuilder.inl:16,28,40`）、`BINDING_FUNCTION_PARAM` **4**（定义 + `TFunctionBuilder.inl:19,31,43`）、`BINDING_CONSTRUCTOR_SIGNATURE` **3**（定义 + `TConstructorHelper.inl:18` + `TConstructorBuilder.inl:10`）、`BINDING_CONSTRUCTOR_PARAM` **2**、`BINDING_DESTRUCTOR_SIGNATURE` **3**（定义 + `TDestructorHelper.inl:16` + `TDestructorBuilder.inl:10`）、`BINDING_DESTRUCTOR_PARAM` **2**、`BINDING_SUBSCRIPT_GET_SIGNATURE` **3** / `_PARAM` **2**、`BINDING_SUBSCRIPT_SET_SIGNATURE` **3** / `_PARAM` **2**、`BINDING_PROPERTY_BUILDER_GET_SIGNATURE` **3**（定义 + `Macro/BindingMacro.h:246` 上出现 2 次）/ `_PARAM` **2** —— 与 §4 表内的每个数字吻合；"`IN_BUFFER` 与 `OUT_BUFFER` 都是 `uint8*`"由 `CoreMacro/BufferMacro.h:9,15` 证实
- **级别变动**: 无（P3 维持：一致性/可维护性）

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Public/Macro/SignatureMacro.h:6-16
#define BINDING_CONSTRUCTOR_SIGNATURE IManagedHandle InManagedObject, IN_BUFFER_SIGNATURE, OUT_BUFFER_SIGNATURE
#define BINDING_CONSTRUCTOR_PARAM InManagedObject, IN_BUFFER, OUT_BUFFER
#define BINDING_DESTRUCTOR_SIGNATURE const IManagedHandle InManagedHandle
#define BINDING_DESTRUCTOR_PARAM InManagedHandle
#define BINDING_FUNCTION_SIGNATURE const IManagedHandle InManagedHandle, IN_BUFFER_SIGNATURE, OUT_BUFFER_SIGNATURE, RETURN_BUFFER_SIGNATURE
#define BINDING_FUNCTION_PARAM InManagedHandle, IN_BUFFER, OUT_BUFFER, RETURN_BUFFER
```
6 对宏，每对必须顺序、个数、类型逐项对应。

**调用上下文**（grep 命中 = 定义 + 使用）
- `BINDING_FUNCTION_SIGNATURE` 6 命中：定义 + `TFunctionBuilder.inl:16,28,40`（3 个偏特化）+ 另 2 处
- `BINDING_FUNCTION_PARAM` 4 命中、`BINDING_CONSTRUCTOR_SIGNATURE` 3、`BINDING_CONSTRUCTOR_PARAM` 2（`TConstructorBuilder.inl`）、`BINDING_DESTRUCTOR_SIGNATURE` 3 / `_PARAM` 2（`TDestructorBuilder.inl:10,13`）、`BINDING_SUBSCRIPT_GET/SET_*` 各 2-3、`BINDING_PROPERTY_BUILDER_GET_SIGNATURE` 3 / `_PARAM` 2（`Macro/BindingMacro.h:246-247`）

**问题**
`SIGNATURE` 与 `PARAM` 是**两份手写副本**（一个是"带类型形参列表"，一个是"实参列表"）。这是所有 P/Invoke 绑定代码最容易出错的地方：往 `_SIGNATURE` 里加一个参数而忘了同步 `_PARAM`，会得到"实参与形参数量不匹配"的编译错误（还算幸运）；若两者的**顺序**被改错但类型恰好兼容（例如两个都是 `uint8*`，`IN_BUFFER` 与 `OUT_BUFFER` 都是 `uint8*` —— `CoreMacro/BufferMacro.h:9,15` 证实），就会**静默交换缓冲区**：C# 传来的入参被当作返回值写回，属于最难查的一类 bug。

**建议**
用单一来源生成"实参列表"，彻底消除两份副本：例如定义
```cpp
#define BINDING_FUNCTION_ARGS(X) X(InManagedHandle) X(InBuffer) X(OutBuffer) X(ReturnBuffer)
#define BINDING_FUNCTION_SIGNATURE BINDING_FUNCTION_ARGS(BINDING_DECL_PARAM)   // 生成 "T name, T name, ..."
#define BINDING_FUNCTION_PARAM     BINDING_FUNCTION_ARGS(BINDING_PASS_PARAM)   // 生成 "name, name, ..."
```
（X-macro 形式；UE 已有类似用法。）至少应加注释把两行的对应关系写清楚，并在 `TFunctionBuilder` 的 `Invoke` 里对 `sizeof...(Args)` 与期望值做 `static_assert`。

**验证方式**
比较两侧 token 数：`grep -c ',' SignatureMacro.h` 无法覆盖类型；建议写一个 `static_assert` 检查 `Invoke` 的形参个数（配合 `TSizeof_Args`）。

---

### 已检查维度（未发现问题）

以下维度按要求逐项检查过，**结论是健康的**，列出以免后来者重复劳动：

1. **空 `__VA_ARGS__` 的行为**（任务重点）：`BINDING_CLASS(X)`、`BINDING_ENUM(X)`、`BINDING_CONSTRUCTOR(T)`、`BINDING_DESTRUCTOR()`、`BINDING_PROPERTY(P)`、`BINDING_FUNCTION(F)`、`BINDING_SUBSCRIPT(T,R,I)` 的**空实参形态在 MSVC（传统 + `/Zc:preprocessor`）与 Clang 上全部得到正确展开**，`TSizeof_Args()` 空包合法、`if constexpr` 被丢弃分支不被实例化。唯一"空/非空不对称"的问题是 `##` 出现在非逗号位置（F-MACRO-001），与"空实参"本身无关。
   - **补充（口径收窄）**：可复现的部分是 `##` 位于**逗号右侧**的形态（`BINDING_CONSTRUCTOR_BUILDER_INVOKE(FVector)` → `TConstructorBuilder<FVector >`，MSVC 传统模式多一个空格，逐字复现）与"Clang/一致性预处理器对非逗号位置报错"。而**嵌套变参宏在最小探针里 MSVC 未递归展开**这一现象（§0.5）说明"MSVC 侧空实参展开的正确性"**只能用插件真实编译产物间接证明**，无法用最小探针逐步复现 —— 本条保留"正确"的结论，但证据等级从"实测"降为"实测（Clang/一致性预处理器）+ 间接证明（MSVC 产物）"。
2. **参数个数不匹配是否有保护**：没有 `static_assert`，但 C++ 模板机制会给出**可定位**的错误（例如 `BINDING_CONSTRUCTOR(T)` 传成 2 个模板实参会报 `TConstructorBuilder<T, X>` 无匹配），未发现"难以理解的编译错误"（F-MACRO-001/004 的问题在于**宏名不在诊断里**，而非信息不足）。
3. **`do{}while(0)`**：见 F-MACRO-006（10 个语句宏**都没包**；当前 **37 处**调用均未触发）。
4. **分号使用导致 `if/else` 断裂**：宏设计为"末尾 `}` → 调用处不写分号"（**37 处**调用一致，`grep 'PROCESS_[A-Z_]*\(\)'` = 37）；`PROCESS_RETURN` 末尾自带 `;` 与其余 9 个宏风格不一致，已并入 F-MACRO-006。**更正**："放进无花括号 `if/else` 就编译失败"的说法不成立（无分号时 MSVC/clang 都通过），真正断裂的是"调用处补分号"与"宏体自带 `;`"两种写法 —— 详见 F-MACRO-006 的四变体矩阵。
5. **`#define` 污染全局命名空间**：`Macro/*` 与 `CoreMacro/*` 的**多数**宏名全大写且带 `BINDING_`/`FUNCTION_`/`NAMESPACE_`/`CLASS_` 前缀，但**并非"没有裸短名"**（与 `02-UnrealCSharpCore核心/07` 的 F-FL-021 对齐）。已核实的裸名/通用名至少 5 个：`CoreMacro/Macro.h:127 #define PLACEHOLDER _`、`:129 #define STR(Str) #Str`、`:131 #define TEXT_STR(Str) TEXT(STR(Str))`、`:103 #define DYNAMIC FString(TEXT("Dynamic"))`、`:33 #define INTEROP_NAME FString(TEXT("Interop"))`（F-FL-021 已就其中 `STR` 的危险性单独给出结论，两报告一致）；另有 `AccessPrivateMacro.h` 生成的 `Class##_##PropertyName` 结构体名（会进全局命名空间，形如 `AActor_bReplicates`）。这些都应并入 F-MACRO-009 的命名卫生议题（不新增 Finding 编号）。
6. **与 UE 自身宏（`TEXT`/`check`/`UE_LOG`/`UPROPERTY`）冲突**：全插件 757 条 `#define` 扫描中**没有**与 UE 宏同名的定义（口径说明见 §1.1 的口径补充：`757` 这一具体数字无法复现，但"无同名"结论已用两条独立 grep 验证 —— 在 `UnrealCSharpCore` 内搜 19 个最易撞名的宏名 → 0 命中）；`BINDING_*`/`FUNCTION_*`/`CLASS_*` 前缀也不与 UE 5.6 冲突（UE 用的是 `UE_*`/`WITH_*`/`PLATFORM_*`）。`WITH_TYPE_INFO`/`WITH_FUNCTION_INFO`/`WITH_PROPERTY_INFO` 是插件自造名，需注意它们**无条件** `#define`（`CoreMacro/BindingMacro.h:15-19`，未加 `#ifndef` 保护），若外部工程自定义同名宏会产生 C4005 重定义；当前无此情况。
7. **`SignatureMacro.h` 的边界情形**（任务重点）：`void` 返回、无参、引用返回、`TArray` 参数都**不经过签名宏**——签名宏只描述"托管句柄 + 3 个 `uint8*` 缓冲区"这一固定 ABI（`SignatureMacro.h:14-16`），具体返回类型/参数类型由 `TFunctionBuilder`/`TReturnValue`/`TArgument` 的模板处理，因此不存在"某些返回类型不成立"的问题。唯一与签名相关的隐患是 F-MACRO-013 的成对宏同步。
8. **`NamespaceMacro.h` 的命名空间拼接**（任务重点）：该文件只有 1 个宏 `NAMESPACE_BINDING FString(TEXT("Binding"))`，**不含任何拼接**，类型名含 `::` 或模板的情形由 `BINDING_REMOVE_*_NAMESPACE_CLASS_STR`（`Macro/BindingMacro.h:41-45`）与 `COMBINE_NAMESPACE`（`CoreMacro/NamespaceMacro.h:3`）处理——前者的边界问题见 F-MACRO-002，后者的 `FString::Printf(TEXT("%s.%s"))` 对含 `::`/`<...>` 的名字会生成形如 `A.B<C, D>` 的字符串（`TName.inl:29-35,251-264` 实际就这么用），语义上"能跑通但不规范"（C# 泛型名应为 `A.B` 加反引号 `1` 后缀），未构成本插件内的实际错误，故不单列 Finding。

---

## 4. 死代码清单

统计口径：`Source/` 下全部 `.h/.cpp/.inl`（排除 `ThirdParty/`），按"#define 行数"与"总命中行数"分开计数；`grep` 工具与 `pwsh Select-String -Pattern '\bNAME\b'` 双向核对，两者一致。
**下表每个 grep 命中数都用 `grep` 工具重跑过，结果与表格完全一致**（`\bBINDING_OVERLOADS\b`=2、`\bBINDING_REGULAR_UENUM\b`=2、`\bBINDING_STRUCT\b`=7、`\bBINDING_REMOVE_LEFT_NAMESPACE_CLASS_STR\b`=3、`\bBINDING_REMOVE_RIGHT_NAMESPACE_CLASS_STR\b`=3、`\bOPERATOR_BUILDER\b`=4、`\bNEW_PROPERTY_DESCRIPTOR_IMPLEMENTATION\b`=2、`\bNATIVE_OUT_VALUE\b`=2、`\bREFERENCE_IN_VALUE\b`=2、`INITIALIZE_VALUE`/`IN_VALUE`/`IN_END` 各 4、4 个 `*_PARAM` 宏各 2、`BINDING_PROPERTY_BUILDER_GET_SIGNATURE`=3、`FUNCTION_IMPLEMENTATION_SUFFIX`=3、`NAMESPACE_BINDING`=36、`BINDING_COMBINE_CLASS`=2、`WITH_TYPE_INFO`=4 / `WITH_PROPERTY_INFO`=7）。唯一需要补口径的是 `WITH_FUNCTION_INFO`：`grep '\bWITH_FUNCTION_INFO\b'` = **27**（1 定义 + **26** 处使用，与 §1.4 的"26 处"一致）。

| 宏名 | 声明位置 | grep 模式 | 总命中 | #define 数 | 判定 | 证据 |
|---|---|---|---|---|---|---|
| `BINDING_OVERLOADS` | `Macro/BindingMacro.h:283,285` | `\bBINDING_OVERLOADS\b` | 2 | 2 | **死宏（可删）** | 2 命中全部是 `#define`（`#if/#else` 两分支），全插件无调用点；`TOverloadBuilder` 亦无使用者 |
| `BINDING_REGULAR_UENUM` | `Macro/BindingMacro.h:237,241` | `\bBINDING_REGULAR_UENUM\b` | 2 | 2 | **死宏（版本门控为空）** | 2 命中全部是 `#define`；`#if UE_REGULAR_UENUM_STATIC_ENUM`（`UEVersion.h:206` = `UE_VERSION_START(5,8,0)`）在 UE 5.6 为 0 → 两个分支都是空定义 |
| `BINDING_STRUCT` | `Macro/BindingMacro.h:110` | `\bBINDING_STRUCT\b` | 7 | 1 | **当前为空（6 个调用点被 `#if UE_U_STRUCT_*` 屏蔽）** | 调用点：`FRegisterSoftObjectPath.cpp:8`、`FRegisterSoftClassPath.cpp:9`、`FRegisterPrimaryAssetType.cpp:8`、`FRegisterPrimaryAssetId.cpp:8`、`FRegisterAssetBundleData.cpp:9`、`FRegisterAssetBundleEntry.cpp:8`，各自处于 `#if UE_U_STRUCT_*`（`UEVersion.h:176-188` 全为 `UE_VERSION_START(5,7,0)`）内 |
| `BINDING_REMOVE_LEFT_NAMESPACE_CLASS_STR` | `Macro/BindingMacro.h:41` | `\bBINDING_REMOVE_LEFT_NAMESPACE_CLASS_STR\b` | 3 | 1 | 二级宏（仅被 `BINDING_CLASS`/`BINDING_ENUM` 内部使用） | `:62`、`:183` 两处使用；**无外部调用点**，且其分支只有 `BINDING_ENUM(..., true)` 会走到 —— 而 7 处 `BINDING_ENUM` 调用无一传 `true` |
| `BINDING_REMOVE_RIGHT_NAMESPACE_CLASS_STR` | `Macro/BindingMacro.h:44` | `\bBINDING_REMOVE_RIGHT_NAMESPACE_CLASS_STR\b` | 3 | 1 | 二级宏 | `:63`、`:184` 两处使用；实际只在 `FRegisterWorld.cpp:10`（`..., false`）被走到 |
| `OPERATOR_BUILDER` | `Macro/BindingMacro.h:320` | `\bOPERATOR_BUILDER\b` | 4 | 1 | 二级宏 | 被 `PREFIX_UNARY_CONST_OPERATOR:329`、`PREFIX_UNARY_OPERATOR:338`、`BINARY_OPERATOR:347` 使用 |
| `NEW_PROPERTY_DESCRIPTOR_IMPLEMENTATION` | `PropertyMacro.h:3` | `\bNEW_PROPERTY_DESCRIPTOR_IMPLEMENTATION\b` | 2 | 1 | 二级宏 | 仅被 `NEW_PROPERTY_DESCRIPTOR:5` 使用 |
| `NATIVE_OUT_VALUE` | `FunctionMacro.h:94` | `\bNATIVE_OUT_VALUE\b` | 2 | 1 | 二级宏 | 仅被 `PROCESS_NATIVE_REFERENCE_IN:113` 使用 |
| `REFERENCE_IN_VALUE` | `FunctionMacro.h:75` | `\bREFERENCE_IN_VALUE\b` | 2 | 1 | 二级宏 | 仅被 `PROCESS_SCRIPT_REFERENCE_IN:91` 使用 |
| `INITIALIZE_VALUE` / `IN_VALUE` / `IN_END` | `FunctionMacro.h:65,71,81` | `\bINITIALIZE_VALUE\b` 等 | 各 4 | 各 1 | 二级宏（"半个语句"配对宏） | 分别被 `PROCESS_SCRIPT_IN`、`PROCESS_SCRIPT_REFERENCE_IN`、`PROCESS_NATIVE_REFERENCE_IN` 使用；**且 `IN_END()` 只有 `}`，任何单独调用都是语法错误** |
| `BINDING_CONSTRUCTOR_PARAM` / `BINDING_DESTRUCTOR_PARAM` / `BINDING_SUBSCRIPT_GET_PARAM` / `BINDING_PROPERTY_BUILDER_GET_PARAM` | `SignatureMacro.h:8,12,20,28` | 各自 `\b…\b` | 各 2 | 各 1 | 正常（1 定义 + 1 使用） | 使用点：`TConstructorBuilder.inl`、`TDestructorBuilder.inl:13`、`TSubscriptBuilder.inl`、`Macro/BindingMacro.h:246` |
| `BINDING_PROPERTY_BUILDER_GET_SIGNATURE` | `SignatureMacro.h:26` | `\b…\b` | 3 | 1 | 正常 | 使用点 `Macro/BindingMacro.h:246`（出现 2 次） |
| `FUNCTION_IMPLEMENTATION_SUFFIX` | `FunctionMacro.h:5` | `\bFUNCTION_IMPLEMENTATION_SUFFIX\b` | 3 | 1 | 正常（2 处使用） | 但值 `FString("_Implementation")` **缺 `TEXT()`**，且与 `CoreMacro/BindingMacro.h:5` 的 `"%sImplementation"` 后缀不一致（一个带下划线一个不带）→ 见 §5 存疑 |
| `NAMESPACE_BINDING` | `Macro/NamespaceMacro.h:3` | `\bNAMESPACE_BINDING\b` | 36 | 1 | 正常（35 处使用） | 但与 `CoreMacro/Macro.h:49 BINDING_NAME`（1 处使用）值完全相同 → 重复定义，见 F-MACRO-009 |
| `BINDING_COMBINE_CLASS` | `CoreMacro/BindingMacro.h:3` | `\bBINDING_COMBINE_CLASS\b` | 2 | 1 | 近乎死宏（1 处使用） | 与 `COMBINE_NAMESPACE`(308 次)、`COMBINE_FULL_NAME`(16 次) 函数体逐字节相同 |
| `WITH_TYPE_INFO` / `WITH_PROPERTY_INFO` | `CoreMacro/BindingMacro.h:15,19` | `\bWITH_TYPE_INFO\b` 等 | 4 / 7 | 1 / 1 | 正常 | 使用 3 / 6 处 |

**未在表中但需要提名的"零命中"检查**：任务简报举例的 `SIGNATURE_` 前缀宏在本插件**不存在**；签名宏实际命名为 `BINDING_*_SIGNATURE`（`SignatureMacro.h:6,10,14,18,22,26`），已全部纳入上表/§3。

---

## 5. 未覆盖/存疑项

1. **`TStaticName<T,T>` 未读**：`BINDING_STRUCT`（`Macro/BindingMacro.h:110-116`）特化的是 `TStaticName`，其主模板与其它特化在 `UnrealCSharpCore/Public/Binding/TypeInfo/TStaticName.inl`（被 `TClassBuilder.inl:6` 包含）。我**没有读该文件**，因此无法确认 `BINDING_STRUCT` 是否也会与 Core 的某个 trait 特化形成 F-MACRO-004 式的重叠。由于 `BINDING_STRUCT` 在当前 UE 5.6 下**展开为空**，此风险不紧急，但在升级到 UE 5.7 前必须补查。
2. **`TIsUStruct` 为假的最终依据是间接的**：我通过"`FVector`/`FGuid`（`NoExportTypes.h:534,588`）、`FPolyglotTextData`、`FFrameRate`、`FRandomStream`、`FExpressionInput` 的 C++ 声明都**没有** `GENERATED_BODY`/`GENERATED_USTRUCT_BODY`，而常规 USTRUCT 由 UHT 生成 `static class UScriptStruct* StaticStruct();`"推断出 `TIsUStruct<FVector>::Value == false`（否则 F-MACRO-004 的 C2752 必然出现、插件无法编译）。我**没有**直接编译验证 `TIsUStruct<FVector>::Value`（需要完整引擎头文件树），置信度标为"高（多路一致）"但非"实测"。若要 100% 确认，可在插件任一 `.cpp` 里加：`static_assert(!TIsUStruct<FVector>::Value, "");` 编译一次。
3. **未验证的假设（F-MACRO-003 的性能定级）**：我确认了"每次调用都重建 FString/TArray"，但**没有**逐条追踪 `TName<T,T>::Get()`/`TNameSpace<T,T>::Get()` 是否出现在**每帧**热路径上（例如 C# 访问属性时是否经由 `TPropertyClass<T,T>::Get()` 重新查名）。若只发生在模块加载/注册阶段，则实际影响应从 P2 降到 P3。需要读 `TBindingPropertyClass::Get()`（`UnrealCSharp/Public/Binding/Core/TPropertyClass.inl:1-60`）与 `FCSharpEnvironment` 的按名查类路径才能定论 —— 这属于其它报告的范围。
4. **`BufferAllocator` 是否真的可能失效未证伪**（F-MACRO-005）：我从 `FFunctionParamBufferAllocatorFactory::Factory` 返回 `TSharedRef` 判断它在当前构造路径下恒有效，因此把严重度定为 P2 而非 P0/P1。需要一个运行期断言（或在 `FFunctionDescriptor` 构造函数加 `ensure`）才能给出确定结论。
5. **Clang 版本代表性**：我使用的 Clang 是引擎第三方 IREE 自带的 `clang 20.0.0git`（Windows 目标）。UE 在 Linux/Mac/Android/iOS 上用的是各自 SDK 的 Clang（16-18 居多）。`-Winvalid-token-paste` 是 Clang 长期稳定行为，但我没有在多个 Clang 版本上复测。
6. **MSVC 传统预处理器行为**：`##__VA_ARGS__` 在传统预处理器下"无警告且结果正确"是 VS 18 (14.51) 的实测结果；旧版 MSVC（VS2019 及更早）的传统预处理器对 `##` 的边界处理存在已知差异，我**没有**旧版本可复测。
7. **`FString::Find`/`Left`/`Right` 语义基于 UE 5.6**：`UnrealString.h.inl:1146-1149/1196-1200/1317` 是 UE 5.6 的实现，UE 5.5 及更早版本的 `Left`/`Right` 对负数的处理**可能不同**（本项目还注册了 `<Engine 5.0 安装目录>`）。F-MACRO-002 的"空串/丢首字符"结论在 UE 5.0 上未验证（`CrossVersion` 模块说明插件要支持多版本）。
8. **`CoreMacro/MetaDataAttributeMacro.h`（315 行）与 `ClassMacro.h` 的常量是否与 C# 侧特性名逐一对齐**未做交叉验证（需要读 `Script/` 下的 C# 特性定义），本报告只确认了它们"被使用"（不是死宏）。

### 5.1 存疑项与已推翻项

9. **【重要更正】F-MACRO-006 的核心断言被证伪**："放进无花括号 `if/else` 即编译失败（实测 `error C2062`）"**不成立**（MSVC 14.51 与 clang 20 双实测均通过），"`else` 挂不到 `for` 上"的解释在语法上也是错的。正确的失败条件是"调用处多写一个分号 / 宏体自带 `;`"，诊断是 `C2181`（clang: expected expression）。该 Finding 的其他内容（缺 `do{}while(0)`、`PROCESS_RETURN` 末尾 `;` 与其余 9 个不一致、37 处调用形态）仍然成立。**若其它报告（如 `08-专项审计/04`）引用了本条的 `C2062` 结论，需一并更正。**
10. **MSVC 侧"空实参展开正确"缺少最小探针证据**：见 §0.5 —— 最小探针里 MSVC（两种预处理器）不递归展开嵌套的 `BINDING_DESTRUCTOR_BUILDER_INVOKE()`，与本机真实编译产物（`Binaries/Win64/UnrealEditor-UnrealCSharp.dll` + 12 个 `Module.*.cpp.obj`）矛盾，判为探针假阴性。**未找到中间规模的复现方法**（需要一个只 include `Macro/BindingMacro.h` 及其依赖、但又能独立编译的 TU；本机未搭出该编译环境）。
11. **`507 个 .h/.cpp/.inl`、`757 条 #define`、`18 个重名`三个数字无法用可用工具复现**：`grep` 工具不支持排除 glob、也不提供文件计数。已验证的替代口径：`Source/` 全量 `#define` = **1777**（含 `ThirdParty/`），其中 `UnrealCSharp`=99、`UnrealCSharpCore`=524。**结论（无同名宏）本身已被两条独立 grep 证实**，只有这三个统计数字待后续用 shell 复核。
12. **`WITH_FUNCTION_INFO` 在 Shipping 下消费者侧的同步**已核（`TClassBuilder.inl:34-51`、`FClassBuilder.h:26-28/41-55`、`FClassBuilder.inl:3-20`），但**未编译验证** Shipping 构建：只做了预处理级验证，没有跑一次 `UE_BUILD_SHIPPING` 构建来证明 `BINDING_PROPERTY`/`BINDING_FUNCTION` 的实参个数与消费者重载真的匹配（报告称"两侧同步，逻辑自洽"，属源码级推断）。
13. **与 `02-UnrealCSharpCore核心/07-函数库宏与模板Trait.md` 的 F-FL-021 的交叉核对结果**（两份报告均未改动对方）：
    - **一致**：`CoreMacro/Macro.h` 非自包含头（无 `#include`，宏体却用 `FString`/`UObject`/`TBaseStructure<FVector>`）—— 重读 `Macro.h:1-133` 证实；`STR`/`TEXT_STR`/`PLACEHOLDER` 通用名裸宏 —— 证实（**与本报告 §"已检查维度"5 的"没有裸短名"矛盾，已修正**）；`BufferMacro` 用参数名当宏名（`BufferMacro.h:5,11,17` = `InBuffer`/`OutBuffer`/`ReturnBuffer`）—— 证实；`CompilerMacro.h:3-4` 空首分支与 `:15-21` 兜底 —— 证实（`CompilerMacro.h` 共 21 行）；`NAMESPACE_ROOT`（`NamespaceMacro.h:5`）与 `NAMESPACE_SCRIPT`（`:7`）字面量相同 —— 证实。
    - **无实质矛盾**：F-FL-021 归 CoreMacro 的"宏卫生"，本报告归 `Macro/` 的绑定宏门面；两者在 `NAMESPACE_*`、`BufferMacro`、`CompilerMacro` 上**结论一致、无冲突**。唯一的表述冲突（裸短名）已消除。
    - **分工重叠提示**：`BINDING_PROPERTY_BUILDER_GET_SIGNATURE`（`SignatureMacro.h:26`）依赖 `CoreMacro/BufferMacro.h` 的 `*_SIGNATURE`，因此 F-FL-021 若改 `BufferMacro` 的形态，会直接影响本报告的 F-MACRO-013。

