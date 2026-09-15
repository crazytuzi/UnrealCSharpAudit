# 10b — Interop 注册层：容器 / 字符串 / 对象指针（`Domain/Interop/FRegister*.cpp`）

> 本报告各条**严重度见各 Finding 的「复核结论」字段**（16 条：**确认 8、部分确认 7、证伪 1、无法验证 0**；**级别下调 4 条**（F-INT2-002/004/005/015）、**上调 1 条**（F-INT2-012）、**撤销 1 条**（F-INT2-010；另 F-INT2-009 已撤销并补证）、**维持 9 条**）。

> ⚠️ **编号撞号警告（引用时必须带报告路径）**：前缀 `F-INT2-*` 在本项目中**存在两套互不相同的编号**——
> - 本报告（`01-UnrealCSharp运行时/10b-Interop注册-容器字符串与对象指针.md`）：`F-INT2-001` ~ `F-INT2-015` 与 `F-INT2-018`（共 16 条，**无 016/017**）；
> - `01-UnrealCSharp运行时/11-*.md`：**另一套** `F-INT2-*`，指向完全不同的发现。
>
> 因此**任何跨文档引用都必须写成 `01-…/10b` 或 `01-…/11` + 编号**，只写 `F-INT2-0xx` 是不明确的。本报告内 `F-INT2-*` 的全部编号为：
> `F-INT2-001, 002, 003, 004, 005, 006, 007, 008, 009, 010, 011, 012, 013, 014, 015, 018`。


> 分析范围：`Plugins/UnrealCSharp/Source/UnrealCSharp/Private/Domain/Interop/` 下 18 个文件（容器、字符串、委托、对象指针、纯头文件）
> 覆盖文件：**18 / 18 全部读完**
>
>

> ⚠️ **重要范围修正（影响所有"死代码"判定）**：本仓库存在**两套 `Script/` 树**：
> - `Plugins/UnrealCSharp/Script/` —— 插件自带的 C# 运行时（`UE/Library/*Implementation.cs`、`Interop/`、`SourceGenerator/` 等）；
> - `<Project>/Script/` —— **项目级生成代码**（`UE/Proxy/**`，由 `ScriptCodeGenerator` 生成）。
>
> `*Implementation.cs` 的公开包装层是**被项目级生成代码调用的**。任何只看 `Plugins/.../Script` 的死代码判定都会得出错误结论（见 §7）。

---

## 0. 覆盖范围与阅读清单

### 0.1 交付范围内文件（18 个，全部读完）

| 文件 | 行数 | 是否读完 | 备注 |
|---|---|---|---|
| `FRegisterArray.cpp` | 349 | ✅ 全读 | TArray，**29** 个注册项 |
| `FRegisterMap.cpp` | 208 | ✅ 全读 | TMap，16 个 |
| `FRegisterSet.cpp` | 134 | ✅ 全读 | TSet，11 个 |
| `FRegisterOptional.cpp` | 160 | ✅ 全读 | TOptional，8 个；整体被 `#if UE_F_OPTIONAL_PROPERTY` 包裹 |
| `FRegisterString.cpp` | 60 | ✅ 全读 | FString，4 个 |
| `FRegisterName.cpp` | 73 | ✅ 全读 | FName，5 个（含 `NAME_None`） |
| `FRegisterText.cpp` | 88 | ✅ 全读 | FText，4 个 |
| `FRegisterAnsiString.cpp` | 68 | ✅ 全读 | FAnsiString，4 个；`#if UE_F_UTF8_STR_PROPERTY` |
| `FRegisterUtf8String.cpp` | 65 | ✅ 全读 | FUtf8String，4 个；`#if UE_F_UTF8_STR_PROPERTY` |
| `FRegisterDelegate.cpp` | 198 | ✅ 全读 | TDelegate，16 个 |
| `FRegisterMulticastDelegate.cpp` | 206 | ✅ 全读 | TMulticastDelegate，**14 个注册调用 / 13 个唯一名** |
| `FRegisterLazyObjectPtr.cpp` | 66 | ✅ 全读 | TLazyObjectPtr，4 个 |
| `FRegisterWeakObjectPtr.cpp` | 66 | ✅ 全读 | TWeakObjectPtr，4 个 |
| `FRegisterSoftClassPtr.cpp` | 75 | ✅ 全读 | TSoftClassPtr，5 个 |
| `FRegisterSoftObjectPtr.cpp` | 75 | ✅ 全读 | TSoftObjectPtr，5 个 |
| `FRegisterSubclassOf.cpp` | 65 | ✅ 全读 | TSubclassOf，4 个 |
| `FRegisterObjectFlags.h` | 51 | ✅ 全读 | **纯头文件（注册枚举，非死文件）** |
| `FRegisterForceInit.h` | 18 | ✅ 全读 | **纯头文件（注册枚举，非死文件）** |

**交付范围内文件合计 137 个 `.Function(...)` 注册调用，136 个唯一名**（唯一重复：`FRegisterMulticastDelegate.cpp` 的 `Contains`，见 F-INT2-014）。

逐文件实测计数（`Select-String -Pattern '\.Function\("' | Measure-Object`）：

| 文件 | `.Function()` 调用数 | 唯一名数 | 重复 |
|---|---|---|---|
| `FRegisterArray.cpp` | **29** | 29 | — |
| `FRegisterMap.cpp` | 16 | 16 | — |
| `FRegisterSet.cpp` | 11 | 11 | — |
| `FRegisterOptional.cpp` | 8 | 8 | — |
| `FRegisterString.cpp` | 4 | 4 | — |
| `FRegisterName.cpp` | 5 | 5 | — |
| `FRegisterText.cpp` | 4 | 4 | — |
| `FRegisterAnsiString.cpp` | 4 | 4 | — |
| `FRegisterUtf8String.cpp` | 4 | 4 | — |
| `FRegisterDelegate.cpp` | 16 | 16 | — |
| `FRegisterMulticastDelegate.cpp` | **14** | **13** | **`Contains` ×2** |
| `FRegisterLazyObjectPtr.cpp` | 4 | 4 | — |
| `FRegisterWeakObjectPtr.cpp` | 4 | 4 | — |
| `FRegisterSoftClassPtr.cpp` | 5 | 5 | — |
| `FRegisterSoftObjectPtr.cpp` | 5 | 5 | — |
| `FRegisterSubclassOf.cpp` | 4 | 4 | — |
| **合计** | **137** | **136** | 1 |

### 0.2 为交叉验证而额外读取的非交付文件

| 文件 | 行数/范围 | 用途 |
|---|---|---|
| `Source/UnrealCSharpCore/Public/CoreMacro/BufferMacro.h` | 29，全读 | `*_BUFFER_SIGNATURE` 宏的真实展开 |
| `Source/UnrealCSharpCore/Public/CoreMacro/CompilerMacro.h` | 21，全读 | `PRAGMA_DISABLE_DANGLING_WARNINGS` 的真实含义（见 F-INT2-018） |
| `Source/UnrealCSharp/Public/Binding/Class/FClassBuilder.h` + `.inl` + `Private/.../FClassBuilder.cpp` | 71 + 56 + 92，全读 | 导出函数注册机制、**导出名如何生成**（F-INT2-014 的关键） |
| `Source/UnrealCSharpCore/Private/Binding/Class/FBindingClassRegister.cpp` | 144，全读 | 最终键名的拼接方式（`__<Class>_<Name>Implementation`） |
| `Source/UnrealCSharpCore/Public/Binding/Enum/TBindingEnumBuilder.inl` | 35，全读 | 头文件重复注册的后果（F-INT2-015） |
| `Source/UnrealCSharpCore/Public/Domain/Script/IManagedHandle.h` | 40，全读 | **句柄的真实 ABI 尺寸 = `int64` 按值**（F-INT2-002） |
| `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp` | 347，全读 | 容器是否暴露内部指针 / 是否 memcpy / index 校验（F-INT2-001/004/007） |
| `Source/UnrealCSharp/Private/Registry/FStringRegistry.cpp` | 113，全读 | 字符串注册表的释放路径（判定 F-INT2-011） |
| `Source/UnrealCSharp/Public/Binding/Function/TMethodHelper.inl` | :70-109 | **"正确释放句柄"的对照实现**（F-INT2-011 的决定性证据） |
| `Script/Interop/Handle/HandleData.cs` | 147，全读 | `Alloc`/`Free` 的引用计数语义（F-INT2-011） |
| `Script/Interop/Bridge/StringBridge.cs` | 45，全读 | `NewString`/`GetString` 的编码契约（§4.2） |
| `Script/SourceGenerator/UnrealTypeSourceGenerator.cs` | :860-967 | partial 方法 → `DllImport` 或函数指针（§1.2、F-INT2-009） |
| `Script/UE/Library/{TArray,TMap,TOptional,FString,FName,FText,FAnsiString,FUtf8String,FDelegate,FMulticastDelegate}Implementation.cs` | 226/176/70/44/53/53/45/45/128/109，全读 | ABI 逐参数对照 |
| `Script/UE/CoreUObject/TArray.cs` | 314，全读 | **调用方是否做 index 校验**（F-INT2-001 的决定性证据） |
| `Script/UE/CoreUObject/FString.cs` | 45，全读 | 终结器 → `UnRegister` 的触发时机（F-INT2-006/013） |
| `Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp` | grep :73-661 | 委托代理代码的实际生成形态（§5.1） |

### 0.3 未读（明确声明）

- `FDelegateHelper` / `FMulticastDelegateHelper` / `FMapHelper` / `FSetHelper` / `FOptionalHelper` 的实现体：**未读**（属其它报告的范围）。涉及它们的结论均标注为"实现细节未验证"。
- `FBinding::Get()` / `FBinding::Register` 的实现：**未读**。这影响 F-INT2-015 的定级与静态初始化顺序结论，已在 §8 标为存疑。
- ~~引擎源码（`StringConv.h`、`ScriptArrayHelper.h`、`Async/TaskGraphInterfaces.h`）：**本机不可达，未读**。~~ **该声明不成立**：UE 5.6 引擎源码在本机**存在**（`Engine\Source`），已实际读取并据此闭合了多个"存疑项"：
  - `Runtime/CoreUObject/Public/UObject/UnrealType.h`（`FScriptArrayHelper`，`:3849` 起；原报告误写为 `Containers/ScriptArrayHelper.h`，**该文件在 UE 5.6 已不存在**）
  - `Runtime/Core/Public/Containers/ScriptArray.h`（`FScriptArray::Remove/Insert/InsertZeroed/SwapMemory`）
  - `Runtime/Core/Public/Misc/AssertionMacros.h`（`check`/`checkSlow` 在 Test/Shipping 下的展开）
  - `Runtime/Core/Public/Containers/StringConv.h`（`TCHAR_TO_UTF8` 宏的真实展开）
  - `Runtime/Core/Private/Async/Async.cpp`（`AsyncTask` 的真实语义）
  - `Runtime/CoreUObject/Public/UObject/PropertyOptional.h`（`MarkSetAndGetInitializedValuePointerToReplace`）
  - `Runtime/CoreUObject/Public/Internationalization/Text.h`（`FTextStringHelper::ReadFromBuffer` 的**真实返回类型**）
  - `Runtime/Core/Public/Containers/AnsiString.h` / `Utf8String.h` / `UnrealString.h.inl`（`FAnsiString`/`FUtf8String::operator*` 的返回类型）
  - `Engine/Source/Programs/UnrealBuildTool/Platform/Android/UEBuildAndroid.cs`（Android 架构限制）
- `Script/UE/Proxy/**`（16512 个 `.cs`）：**只做全量字符串计数，未通读**。

---

## 1. 模块职责与架构速览

**一句话职责**：这 18 个文件是 C++ 容器/字符串/委托/智能指针类型向 C# 暴露的「胶水层实现体」，每个文件用一个匿名命名空间内的 `FRegisterXxx` 结构体，在静态构造期把自己的一批 `static` 函数登记进反射绑定系统。

### 1.1 关键事实：这些函数**不是** `extern "C"` 直接导出

任务书假设「这些文件把 C++ 函数以 `extern "C"` 导出给 C#（P/Invoke）」。**实际机制不同，这一点必须先纠正**，否则所有 ABI 结论都会跑偏：

```cpp
// FRegisterArray.cpp:313-345（节选）
FRegisterArray()
{
    FClassBuilder(TEXT("TArray"), NAMESPACE_LIBRARY)
        .Function("GetTypeSize", GetTypeSizeImplementation)
        ...
}

// FRegisterArray.cpp:346-348
};
[[maybe_unused]] FRegisterArray RegisterArray;
```

- `FClassBuilder::Function(const FString& InName, T InMethod)`（`FClassBuilder.inl:3-20`）取的是**函数指针的裸地址**：`FunctionPointer.Value.Pointer`，然后 `Function(InName, InName, Pointer)`。
- `FClassBuilder.cpp:83-91` 的 `GetFunctionImplementationName` 只在**同名重载**时追加计数后缀（`Count == 0 ? "" : ToString(Count)`），因此本目录所有非重载函数名都被**原样**登记。
- 登记落到 `ClassRegister->BindingMethod(Name, Pointer)`（`FClassBuilder.inl:55`）。

所以 C++ 侧既没有 `extern "C"`，也没有符号导出；真正跨边界的是一条**运行时解析的函数指针**。

### 1.2 C# 侧如何调用（决定 ABI 契约的是"生成的 partial 方法"）

`Script/UE/Library/*Implementation.cs` 里全是形如：

```csharp
// TArrayImplementation.cs:30-35
private static unsafe partial int __TArray_GetTypeSizeImplementation(nint InArray);

public static int TArray_GetTypeSizeImplementation(nint InArray)
{
    return __TArray_GetTypeSizeImplementation(InArray);
}
```

`private static unsafe partial` 的**没有方法体**，由源生成器补齐。`Script/SourceGenerator/UnrealTypeSourceGenerator.cs:909-919` 生成两种形态：

```csharp
// UnrealTypeSourceGenerator.cs:909-919（原样）
"#if WITH_LEANCLR\n" +
$"\t\t[DllImport(\"{NativeModuleName}\", CallingConvention = CallingConvention.Cdecl)]\n" +
$"\t\t{accessibility} static extern unsafe partial {returnType} {method.Name}({parameters});\n" +
"#else\n" +
$"\t\tprivate static nint {slot};\n" +
"\n" +
$"\t\t{accessibility} static unsafe partial {returnType} {method.Name}({parameters}) =>\n" +
$"\t\t\t((delegate* unmanaged[Cdecl]<{pointerType}>)global::Interop.MethodBridge.GetMethod(\n" +
$"\t\t\t\tref {slot}, \"{key}\"))({arguments});\n" +
"#endif\n\n";
```

**结论（对 ABI 分析至关重要）**：

1. 非 LeanCLR 后端（Mono / CoreCLR）走 `delegate* unmanaged[Cdecl]<...>` + `MethodBridge.GetMethod` 运行时解析，**参数类型完全来自 C# 侧 partial 声明的托管类型**，生成器用 `Qualify()`（`:929-939`）原样抄写，不做任何 `[MarshalAs]` 注入。
2. LeanCLR 后端走 `[DllImport("UnrealCSharp", CallingConvention = CallingConvention.Cdecl)]`——**但注册面上并没有这些名字的 C 符号**（见 §1.1）。这是**架构级可疑点**，见 Finding F-INT2-009。
3. 由于两边都是「裸 ABI 直传」，**`bool` 宽度问题不适用**：没有 `bool` 被跨边界传递，C++ 侧一律用 `uint8`，C# 侧一律用 `byte`。这一点是**干净**的（详见 §4.1）。

### 1.3 缓冲协议（`BufferMacro.h`）

```cpp
// Source/UnrealCSharpCore/Public/CoreMacro/BufferMacro.h:9-29（原样）
#define IN_BUFFER_SIGNATURE uint8* IN_BUFFER
#define OUT_BUFFER_SIGNATURE uint8* OUT_BUFFER
#define RETURN_BUFFER_SIGNATURE uint8* RETURN_BUFFER
#define IN_KEY_BUFFER_SIGNATURE uint8* IN_KEY_BUFFER
#define IN_VALUE_BUFFER_SIGNATURE uint8* IN_VALUE_BUFFER
```

即所有「值」都是**非类型化的 `uint8*`**。C++ 导出侧不知道缓冲区里是什么，靠 `FPropertyDescriptor` 决定；C# 侧靠 `typeof(T).IsValueType` 决定 `stackalloc byte[sizeof(T)]` 还是 `stackalloc byte[sizeof(nint)]`（`TArray.cs:78-113`）。

**这里的契约是隐式的**：两边必须对「该类型是值类型（内联字节）还是引用类型（句柄）」达成一致。C# 用 `typeof(T).IsValueType`，C++ 用 `FPropertyDescriptor::IsPrimitiveProperty()` / 属性标志。**这是整个容器层最脆的地方**，见 F-INT2-010。

---

## 2. 关键调用链

1. `TArray.cs:82` → `TArrayImplementation.TArray_GetImplementation` → 生成器生成的 `__TArray_GetImplementation` → `MethodBridge.GetMethod` → `FRegisterArray.cpp:109-119 GetImplementation` → `FArrayHelper::Get(InIndex)`（`FArrayHelper.cpp:123-131`，返回**数组内部元素的裸指针**）→ `FPropertyDescriptor::Get(Value, ReturnBuffer)`。
2. `TArray.cs:196` → `TArray_RemoveAtImplementation` → `FRegisterArray.cpp:195-203` → `FArrayHelper::RemoveAt`（`FArrayHelper.cpp:208-228`，**先用 unchecked 的 `GetRawPtr(InIndex)` 取 Dest**）→ `FScriptArrayHelper::DestroyValue(Dest)` 循环 → `FScriptArray::Remove`。
3. `TArray.cs:310` → `TArray_SwapImplementation` → `FRegisterArray.cpp:298-306` → `FArrayHelper::Swap`（`FArrayHelper.cpp:329-332`）→ `ScriptArray->SwapMemory(i, j, size)`。
4. `TArray.cs:15`（`public TArray() => TArrayImplementation.TArray_RegisterImplementation(this, GetType())`）→ `TArrayImplementation.cs:11-14` → 生成器生成的 `__TArray_RegisterImplementation` → `FRegisterArray.cpp:14-19` → `FReflectionRegistry::Get().GetClass(InManagedType)` → `FCSharpBind::Bind<FArrayHelper>(...)`（`FCSharpBind.inl:89-102`）。<br>⚠️ 原文此处写的 `Array.cs:9-14` 是**不存在的文件**：`Script/` 两棵树内均无 `Array.cs`（`glob **/Array.cs` 命中 0），已修正。
5. `FRegisterArray.cpp:34-41 UnRegister` → `AsyncTask(ENamedThreads::GameThread, λ)` → `FCSharpEnvironment::RemoveContainerReference<FArrayHelper>`。
6. `TLazyObjectPtrImplementation.cs` → `FRegisterLazyObjectPtr.cpp:47-53 Get` → `FCSharpEnvironment::GetMulti<TLazyObjectPtr<UObject>>(InManagedHandle)` → **`Multi->Get()`（`Multi` 未判空）** → `FCSharpEnvironment::Bind(...)`。

---

## 3. 导出函数清单（逐文件）

### 3.1 `FRegisterArray.cpp`（29 个，注册点 `:313-345`）

| 导出名（注册名） | 实现位置 | C++ 返回 | C++ 参数（真实类型） |
|---|---|---|---|
| `Register` | :14-19 | void | `IManagedHandle, IManagedHandle` |
| `Identical` | :21-32 | uint8 | `IManagedHandle, IManagedHandle` |
| `UnRegister` | :34-41 | void | `IManagedHandle` |
| `GetTypeSize` | :43-52 | int32 | `IManagedHandle` |
| `GetSlack` | :54-63 | int32 | `IManagedHandle` |
| `IsValidIndex` | :65-74 | uint8 | `IManagedHandle, int32` |
| `Num` | :76-85 | int32 | `IManagedHandle` |
| `IsEmpty` | :87-96 | uint8 | `IManagedHandle` |
| `Max` | :98-107 | int32 | `IManagedHandle` |
| `Get` | :109-119 | void | `IManagedHandle, int32, uint8*` |
| `Set` | :121-129 | void | `IManagedHandle, int32, uint8*` |
| `Find` | :131-140 | int32 | `IManagedHandle, uint8*` |
| `FindLast` | :142-151 | int32 | `IManagedHandle, uint8*` |
| `Contains` | :153-162 | uint8 | `IManagedHandle, uint8*` |
| `AddUninitialized` | :164-173 | int32 | `IManagedHandle, int32` |
| `InsertZeroed` | :175-183 | void | `IManagedHandle, int32, int32` |
| `InsertDefaulted` | :185-193 | void | `IManagedHandle, int32, int32` |
| `RemoveAt` | :195-203 | void | `IManagedHandle, int32, int32, uint8` |
| `Reset` | :205-212 | void | `IManagedHandle, int32` |
| `Empty` | :214-221 | void | `IManagedHandle, int32` |
| `SetNum` | :223-231 | void | `IManagedHandle, int32, uint8` |
| `Add` | :233-242 | int32 | `IManagedHandle, uint8*` |
| `AddZeroed` | :244-253 | int32 | `IManagedHandle, int32` |
| `AddUnique` | :255-264 | int32 | `IManagedHandle, uint8*` |
| `RemoveSingle` | :266-275 | int32 | `IManagedHandle, uint8*` |
| `Remove` | :277-286 | int32 | `IManagedHandle, uint8*` |
| `SwapMemory` | :288-296 | void | `IManagedHandle, int32, int32` |
| `Swap` | :298-306 | void | `IManagedHandle, int32, int32` |
| `INDEX_NONE` | :308-311 | int32 | （无参） |

### 3.2 `FRegisterMap.cpp`（16 个，注册点 `:185-204`）

| 导出名 | 实现位置 | C++ 返回 | C++ 参数 |
|---|---|---|---|
| `Register` | :13-21 | void | `IManagedHandle, IManagedHandle` |
| `UnRegister` | :23-30 | void | `IManagedHandle` |
| `Empty` | :32-39 | void | `IManagedHandle, int32` |
| `Num` | :41-50 | int32 | `IManagedHandle` |
| `IsEmpty` | :52-61 | uint8 | `IManagedHandle` |
| `Add` | :63-71 | void | `IManagedHandle, uint8*, uint8*` |
| `Remove` | :73-83 | int32 | `IManagedHandle, uint8*` |
| `FindKey` | :85-94 | void | `IManagedHandle, uint8*, uint8*` |
| `Find` | :96-105 | void | `IManagedHandle, uint8*, uint8*` |
| `Contains` | :107-116 | uint8 | `IManagedHandle, uint8*` |
| `Get` | :118-127 | void | `IManagedHandle, uint8*, uint8*` |
| `Set` | :129-137 | void | `IManagedHandle, uint8*, uint8*` |
| `GetMaxIndex` | :139-148 | int32 | `IManagedHandle` |
| `IsValidIndex` | :150-159 | uint8 | `IManagedHandle, int32` |
| `GetEnumeratorKey` | :161-171 | void | `IManagedHandle, int32, uint8*` |
| `GetEnumeratorValue` | :173-183 | void | `IManagedHandle, int32, uint8*` |

### 3.3 `FRegisterSet.cpp`（11 个，注册点 `:116-130`）

| 导出名 | 实现位置 | C++ 返回 | C++ 参数 |
|---|---|---|---|
| `Register` | :14-19 | void | `IManagedHandle, IManagedHandle` |
| `UnRegister` | :21-27 | void | `IManagedHandle` |
| `Empty` | :29-35 | void | `IManagedHandle, int32` |
| `Num` | :37-45 | int32 | `IManagedHandle` |
| `IsEmpty` | :47-55 | uint8 | `IManagedHandle` |
| `GetMaxIndex` | :57-65 | int32 | `IManagedHandle` |
| `Add` | :67-73 | void | `IManagedHandle, uint8*` |
| `Remove` | :75-83 | int32 | `IManagedHandle, uint8*` |
| `Contains` | :85-93 | uint8 | `IManagedHandle, uint8*` |
| `IsValidIndex` | :95-103 | uint8 | `IManagedHandle, int32` |
| `GetEnumerator` | :105-114 | void | `IManagedHandle, int32, uint8*` |

### 3.4 `FRegisterOptional.cpp`（8 个，注册点 `:144-155`）

| 导出名 | 实现位置 | C++ 返回 | C++ 参数 |
|---|---|---|---|
| `Register1` | :14-36 | void | `IManagedHandle, IManagedHandle` |
| `Register2` | :38-72 | void | `IManagedHandle, IManagedHandle, IManagedHandle` |
| `Identical` | :74-85 | uint8 | `IManagedHandle, IManagedHandle` |
| `UnRegister` | :87-93 | void | `IManagedHandle` |
| `Reset` | :95-101 | void | `IManagedHandle` |
| `IsSet` | :103-111 | uint8 | `IManagedHandle` |
| `Get` | :113-125 | **IManagedHandle** | `IManagedHandle` |
| `Set` | :127-142 | void | `IManagedHandle, IManagedHandle` |

### 3.5 字符串族（5 个文件）

#### `FRegisterString.cpp`（4 个，注册点 `:49-56`）

| 导出名 | 实现位置 | C++ 返回 | C++ 参数 |
|---|---|---|---|
| `Register` | :13-19 | void | `IManagedHandle, const char*` |
| `Identical` | :21-32 | uint8 | `IManagedHandle, IManagedHandle` |
| `UnRegister` | :34-40 | void | `IManagedHandle` |
| `ToString` | :42-47 | IManagedHandle | `IManagedHandle` |

#### `FRegisterName.cpp`（5 个，注册点 `:61-69`）

| 导出名 | 实现位置 | C++ 返回 | C++ 参数 |
|---|---|---|---|
| `Register` | :13-19 | void | `IManagedHandle, const char*` |
| `Identical` | :21-32 | uint8 | `IManagedHandle, IManagedHandle` |
| `UnRegister` | :34-40 | void | `IManagedHandle` |
| `ToString` | :42-47 | IManagedHandle | `IManagedHandle` |
| `NAME_None` | :49-59 | IManagedHandle | （无参） |

#### `FRegisterText.cpp`（4 个，注册点 `:75-82`）

| 导出名 | 实现位置 | C++ 返回 | C++ 参数 |
|---|---|---|---|
| `Register` | :16-45 | void | `IManagedHandle, const char*, const char*, const char*, uint8` |
| `Identical` | :47-58 | uint8 | `IManagedHandle, IManagedHandle` |
| `UnRegister` | :60-66 | void | `IManagedHandle` |
| `ToString` | :68-73 | IManagedHandle | `IManagedHandle` |

#### `FRegisterAnsiString.cpp`（4 个，注册点 `:56-63`；整体被 `#if UE_F_UTF8_STR_PROPERTY` 包裹）

| 导出名 | 实现位置 | C++ 返回 | C++ 参数 |
|---|---|---|---|
| `Register` | :15-23 | void | `IManagedHandle, const char*` |
| `Identical` | :25-36 | uint8 | `IManagedHandle, IManagedHandle` |
| `UnRegister` | :38-45 | void | `IManagedHandle` |
| `ToString` | :47-54 | IManagedHandle | `IManagedHandle` |

#### `FRegisterUtf8String.cpp`（4 个，注册点 `:53-60`；整体被 `#if UE_F_UTF8_STR_PROPERTY` 包裹）

| 导出名 | 实现位置 | C++ 返回 | C++ 参数 |
|---|---|---|---|
| `Register` | :15-23 | void | `IManagedHandle, const char*` |
| `Identical` | :25-36 | uint8 | `IManagedHandle, IManagedHandle` |
| `UnRegister` | :38-44 | void | `IManagedHandle` |
| `ToString` | :46-51 | IManagedHandle | `IManagedHandle` |

### 3.6 纯头文件（非"死文件"，见 §7）

#### `FRegisterObjectFlags.h`（51 行，无导出函数——**它注册枚举**）

`BINDING_ENUM(EObjectFlags)`（`:7`）+ `TBindingEnumBuilder<EObjectFlags, false>`（`:13`）登记 27 个 `EObjectFlags` 枚举值（`:14-46`，其中 `RF_KeepForCooker`/`RF_AllocatedInSharedPage` 受 UE 版本宏保护）。
实例：`[[maybe_unused]] static FRegisterObjectFlags RegisterObjectFlags;`（`:51`）

#### `FRegisterForceInit.h`（18 行，无导出函数）

`BINDING_ENUM(EForceInit)`（`:6`）+ 登记 `ForceInit` / `ForceInitToZero`（`:13-14`）。
实例：`[[maybe_unused]] static FRegisterForceInit RegisterForceInit;`（`:18`）

### 3.7 委托族（2 个文件）

#### `FRegisterDelegate.cpp`（16 个注册项，注册点 `:175-194`）

| 导出名 | 实现位置 | C++ 返回 | C++ 参数 |
|---|---|---|---|
| `Register` | :14-19 | void | `IManagedHandle, IManagedHandle` |
| `UnRegister` | :21-28 | void | `IManagedHandle` |
| `Bind` | :30-49 | void | `IManagedHandle ×4` |
| `IsBound` | :51-60 | uint8 | `IManagedHandle` |
| `UnBind` | :62-69 | void | `IManagedHandle` |
| `Clear` | :71-78 | void | `IManagedHandle` |
| `GenericExecute0` | :80-87 | void | `IManagedHandle` |
| `PrimitiveExecute1` | :89-96 | void | `IManagedHandle, uint8*` |
| `CompoundExecute1` | :98-105 | void | `IManagedHandle, uint8*` |
| `GenericExecute2` | :107-114 | void | `IManagedHandle, uint8*` |
| `PrimitiveExecute3` | :116-124 | void | `IManagedHandle, uint8*, uint8*` |
| `CompoundExecute3` | :126-134 | void | `IManagedHandle, uint8*, uint8*` |
| `GenericExecute4` | :136-143 | void | `IManagedHandle, uint8*` |
| `GenericExecute6` | :145-153 | void | `IManagedHandle, uint8*, uint8*` |
| `PrimitiveExecute7` | :155-163 | void | `IManagedHandle, uint8*, uint8*, uint8*` |
| `CompoundExecute7` | :165-173 | void | `IManagedHandle, uint8*, uint8*, uint8*` |

#### `FRegisterMulticastDelegate.cpp`（**14 个 `Function()` 调用，但只有 13 个唯一名**，注册点 `:185-202`）

| 导出名（注册名） | 实现位置 | C++ 返回 | C++ 参数 |
|---|---|---|---|
| `Register` | :14-19 | void | `IManagedHandle, IManagedHandle` |
| `UnRegister` | :21-28 | void | `IManagedHandle` |
| `Contains` | :41-62 | uint8 | `IManagedHandle ×4` |
| `IsBound` | :30-39 | uint8 | `IManagedHandle` |
| **`Contains`（第 2 次）** | :41-62 | uint8 | `IManagedHandle ×4` | ← **重复注册，见 F-INT2-014** |
| `Add` | :64-83 | void | `IManagedHandle ×4` |
| `AddUnique` | :85-104 | void | `IManagedHandle ×4` |
| `Remove` | :106-125 | void | `IManagedHandle ×4` |
| `RemoveAll` | :127-137 | void | `IManagedHandle ×2` |
| `Clear` | :139-146 | void | `IManagedHandle` |
| `GenericBroadcast0` | :148-155 | void | `IManagedHandle` |
| `GenericBroadcast2` | :157-164 | void | `IManagedHandle, uint8*` |
| `GenericBroadcast4` | :166-173 | void | `IManagedHandle, uint8*` |
| `GenericBroadcast6` | :175-183 | void | `IManagedHandle, uint8*, uint8*` |

### 3.8 智能指针族（5 个文件，共 22 个注册项）

| 文件 | 导出名 | 实现位置 | 返回 | 参数 |
|---|---|---|---|---|
| `FRegisterWeakObjectPtr.cpp` | `Register` | :12-23 | void | `IManagedHandle ×3` |
| | `Identical` | :25-36 | uint8 | `IManagedHandle ×2` |
| | `UnRegister` | :38-45 | void | `IManagedHandle` |
| | `Get` | :47-53 | IManagedHandle | `IManagedHandle` |
| `FRegisterSoftClassPtr.cpp` | `Register` | :12-23 | void | `IManagedHandle ×3` |
| | `Identical` | :25-36 | uint8 | `IManagedHandle ×2` |
| | `UnRegister` | :38-45 | void | `IManagedHandle` |
| | `Get` | :47-53 | IManagedHandle | `IManagedHandle` |
| | `LoadSynchronous` | :55-61 | IManagedHandle | `IManagedHandle` |
| `FRegisterSoftObjectPtr.cpp` | `Register` | :12-23 | void | `IManagedHandle ×3` |
| | `Identical` | :25-36 | uint8 | `IManagedHandle ×2` |
| | `UnRegister` | :38-45 | void | `IManagedHandle` |
| | `Get` | :47-53 | IManagedHandle | `IManagedHandle` |
| | `LoadSynchronous` | :55-61 | IManagedHandle | `IManagedHandle` |
| `FRegisterLazyObjectPtr.cpp` | `Register` | :12-23 | void | `IManagedHandle ×3` |
| | `Identical` | :25-36 | uint8 | `IManagedHandle ×2` |
| | `UnRegister` | :38-45 | void | `IManagedHandle` |
| | `Get` | :47-53 | IManagedHandle | `IManagedHandle` |
| `FRegisterSubclassOf.cpp` | `Register` | :12-23 | void | `IManagedHandle ×3` |
| | `Identical` | :25-36 | uint8 | `IManagedHandle ×2` |
| | `UnRegister` | :38-44 | void | `IManagedHandle` |
| | `Get` | :46-52 | IManagedHandle | `IManagedHandle` |

**这 5 个文件的 `Register`/`Identical`/`UnRegister` 三段实现逐字相同**（只有 `TWeakObjectPtr`/`TSoftObjectPtr`/`TSoftClassPtr`/`TLazyObjectPtr`/`TSubclassOf` 与 `InObject`/`InClass` 的措辞差别）：

```cpp
// FRegisterWeakObjectPtr.cpp:12-23（原样）
static void RegisterImplementation(const IManagedHandle InManagedObject,
                                   const IManagedHandle InObject, const IManagedHandle InManagedType)
{
    const auto FoundObject = FCSharpEnvironment::GetEnvironment().GetObject(InObject);

    const auto WeakObjectPtr = new TWeakObjectPtr<UObject>(FoundObject);

    const auto Class = FReflectionRegistry::Get().GetClass(InManagedType);

    FCSharpEnvironment::GetEnvironment().AddMultiReference<TWeakObjectPtr<UObject>, true, false>(
        Class, InManagedObject, WeakObjectPtr);
}

// FRegisterLazyObjectPtr.cpp:12-23（原样）—— 除类型名与局部变量名外一字不差
static void RegisterImplementation(const IManagedHandle InManagedObject, const IManagedHandle InObject,
                                   const IManagedHandle InManagedType)
{
    const auto FoundObject = FCSharpEnvironment::GetEnvironment().GetObject(InObject);

    const auto LazyObjectPtr = new TLazyObjectPtr<UObject>(FoundObject);

    const auto Class = FReflectionRegistry::Get().GetClass(InManagedType);

    FCSharpEnvironment::GetEnvironment().AddMultiReference<TLazyObjectPtr<UObject>, true, false>(
        Class, InManagedObject, LazyObjectPtr);
}
```

**注意 `FRegisterSubclassOf` / `FRegisterSoftClassPtr` 用 `GetObject<UClass>(InClass)`，其余三者用 `GetObject(InObject)`**——这是有意的类型差异，不是 bug。

### 3.9 `FRegisterSubclassOf.cpp`（4 个，注册点 `:54-61`）

见 §3.8 表格最后 4 行（此处不再重复；`Register`/`Identical`/`UnRegister`/`Get` 的位置与 `TLazyObjectPtr` 逐一对应）。

---

## 4. ABI 对照表（C++ 导出 ↔ C# 声明 ↔ 生成器产出）

### 4.1 基元类型映射（抽 6 个代表性函数逐个核对）

| C++ 签名（文件:行） | C# partial 声明（文件:行） | 生成器产出的 `delegate* unmanaged[Cdecl]<...>` | 判定 |
|---|---|---|---|
| `void GetImplementation(IManagedHandle, int32, uint8*)` `FRegisterArray.cpp:109-110` | `void __TArray_GetImplementation(nint InArray, int InIndex, byte* ReturnBuffer)` `TArrayImplementation.cs:72` | `<nint, int, byte*, void>` | ✅ 一致 |
| `uint8 IsValidIndexImplementation(IManagedHandle, int32)` `FRegisterArray.cpp:65` | `byte __TArray_IsValidIndexImplementation(nint, int)` `TArrayImplementation.cs:44` | `<nint, int, byte>` | ✅ 一致（**注意：这里没有用 C# `bool`，避免了 4 字节封送陷阱**） |
| `int32 FindImplementation(IManagedHandle, uint8*)` `FRegisterArray.cpp:131` | `int __TArray_FindImplementation(nint, byte*)` `TArrayImplementation.cs:86` | `<nint, byte*, int>` | ✅ 一致 |
| `void RemoveAtImplementation(IManagedHandle, int32, int32, uint8)` `FRegisterArray.cpp:195-196` | `void __TArray_RemoveAtImplementation(nint, int, int, byte)` `TArrayImplementation.cs:128` | `<nint, int, int, byte, void>` | ✅ 一致（C# 公开层 `:130-134` 把 `bool` 显式转 `(byte)(b ? 1 : 0)`） |
| `IManagedHandle GetImplementation(IManagedHandle)` `FRegisterOptional.cpp:113` | `nint __TOptional_GetImplementation(nint)` `TOptionalImplementation.cs:53` | `<nint, nint>` | ⚠️ 见 F-INT2-002 |
| `void AddImplementation(IManagedHandle, uint8*, uint8*)` `FRegisterMap.cpp:63-64` | `void __TMap_AddImplementation(nint, byte*, byte*)` `TMapImplementation.cs:58` | `<nint, byte*, byte*, void>` | ✅ 一致 |

**关于任务书假设的 `bool` 宽度问题**：**在本目录中不存在**。C++ 侧所有布尔量都以 `uint8` 返回/接收（`FRegisterArray.cpp:21,65,87,153` 等），C# 侧对应声明一律是 `byte`（`TArrayImplementation.cs:16,44,58,100`），公开包装层才做 `!= 0` / `(byte)(b ? 1 : 0)`。这是**正确做法，应予肯定**，不是 bug。

### 4.2 字符串编码对照（任务书重点）

**结论先说：`FString` / `FName` / `FText` 的编码链路是 UTF-8 ↔ UTF-8，两侧完全匹配，无乱码风险。** 链路的两端都已读到实际代码：

**入口方向（C# → C++）**

```csharp
// Script/UE/Library/FStringImplementation.cs:9-19（原样）
private static unsafe partial void __FString_RegisterImplementation(nint InString, byte* InValue);

public static unsafe void FString_RegisterImplementation(FString InString, string InValue)
{
    var UTF8 = InValue != null ? Encoding.UTF8.GetBytes(InValue + '\0') : [0];

    fixed (byte* Ptr = UTF8)
    {
        __FString_RegisterImplementation(HandleData.Alloc(InString), Ptr);
    }
}
```

```cpp
// FRegisterString.cpp:13-19（原样）
static void RegisterImplementation(const IManagedHandle InManagedObject, const char* InValue)
{
    const auto String = new FString(InValue != nullptr ? FString(UTF8_TO_TCHAR(InValue)) : FString(TEXT("")));

    FCSharpEnvironment::GetEnvironment().AddStringReference<FString, true, false>(
        FReflectionRegistry::Get().GetStringClass(), InManagedObject, String);
}
```

→ C# 写 UTF-8 字节 + `'\0'`；C++ 用 `UTF8_TO_TCHAR` 解码。**✅ 匹配。** 注意 C# 侧主动追加了 `'\0'`（`+ '\0'`），因此 `const char*` 是 NUL 结尾的合法 C 字符串——这是必要的，因为 C++ 侧按 `const char*` 处理。

**出口方向（C++ → C#）**

```cpp
// FRegisterString.cpp:42-47（原样）
static IManagedHandle ToStringImplementation(const IManagedHandle InManagedHandle)
{
    const auto String = FCSharpEnvironment::GetEnvironment().GetString<FString>(InManagedHandle);

    return IScriptDomain::Get()->NewString(TCHAR_TO_UTF8(**String));
}
```

```csharp
// Script/Interop/Bridge/StringBridge.cs:8-17（原样）
[UnmanagedCallersOnly]
public static nint NewString(byte* InText)
{
    if (InText != null)
    {
        return HandleData.Alloc(Marshal.PtrToStringUTF8((nint)InText)!);
    }

    return 0;
}
```

→ C++ 用 `TCHAR_TO_UTF8` 产出 UTF-8；C# 用 `Marshal.PtrToStringUTF8` 解码。**✅ 匹配。**

**逐类型映射表**

| 类型 | C# 入口编码 | C++ 解码 | C++ 出口编码 | C# 出口解码 | 判定 |
|---|---|---|---|---|---|
| `FString` | `Encoding.UTF8.GetBytes`（`FStringImplementation.cs:13`） | `UTF8_TO_TCHAR`（`FRegisterString.cpp:15`） | `TCHAR_TO_UTF8`（`:46`） | `Marshal.PtrToStringUTF8`（`StringBridge.cs:13`） | ✅ 匹配 |
| `FName` | `Encoding.UTF8.GetBytes`（`FNameImplementation.cs:13`） | `UTF8_TO_TCHAR`（`FRegisterName.cpp:15`） | `TCHAR_TO_UTF8`（`:46`） | 同上 | ✅ 匹配 |
| `FText` | `Encoding.UTF8.GetBytes` ×3（`FTextImplementation.cs:13,16,20`） | `UTF8_TO_TCHAR` ×3（`FRegisterText.cpp:23,27,31`） | `TCHAR_TO_UTF8`（`:72`） | 同上 | ✅ 匹配 |
| `FAnsiString` | `Encoding.UTF8.GetBytes`（`FAnsiStringImplementation.cs:13`） | `UTF8_TO_TCHAR`（`FRegisterAnsiString.cpp:18`） | `TCHAR_TO_UTF8`（`:52`） | 同上 | ⚠️ 见 F-INT2-012 |
| `FUtf8String` | `Encoding.UTF8.GetBytes`（`FUtf8StringImplementation.cs:13`） | `UTF8_TO_TCHAR`（`FRegisterUtf8String.cpp:18`） | `TCHAR_TO_UTF8`（`:50`） | 同上 | ⚠️ 见 F-INT2-012 |

**任务书问「C# `FString`/`FName`/`FText`/`FAnsiString`/`FUtf8String` 分别对应哪种编码？」的答案**：**五者**在 C# 侧都用同一个 `Encoding.UTF8.GetBytes(...)`，在 C++ 侧都用同一个 `UTF8_TO_TCHAR` / `TCHAR_TO_UTF8`。也就是说**它们之间没有编码差异**——`FAnsiString` 并不按 ANSI 编码传输，这一点与类型名暗示的语义不符（见 F-INT2-012）。

**参数个数/顺序核对（字符串族，逐个核对，全部一致）**

| C++ 签名 | C# partial 声明 | 判定 |
|---|---|---|
| `void RegisterImplementation(IManagedHandle, const char*)` `FRegisterString.cpp:13` | `void __FString_RegisterImplementation(nint, byte*)` `FStringImplementation.cs:9` | ✅ |
| `IManagedHandle ToStringImplementation(IManagedHandle)` `FRegisterString.cpp:42` | `nint __FString_ToStringImplementation(nint)` `FStringImplementation.cs:35` | ✅ |
| `void RegisterImplementation(IManagedHandle, const char*, const char*, const char*, uint8)` `FRegisterText.cpp:16-20` | `void __FText_RegisterImplementation(nint, byte*, byte*, byte*, byte bRequiresQuotes)` `FTextImplementation.cs:8` | ✅ |
| `IManagedHandle NAME_NoneImplementation()` `FRegisterName.cpp:49` | `nint __FName_NAME_NoneImplementation()` `FNameImplementation.cs:44` | ✅ |

**关于 `char*` 的宽度**：C++ `const char*` = 1 字节；C# 侧全部用 `byte*`（**不是 `sbyte*`，也不存在 `[MarshalAs(UnmanagedType.LPStr)]`**）。**✅ 一致**——因为走的是函数指针直传而非 `DllImport` 封送，`[MarshalAs]` 根本不参与，这一点反而避免了经典的 `LPStr`/`LPUTF8Str` 选错问题。

另外 `StringBridge.GetString`（`StringBridge.cs:20-43`）对应 C++ `string_bridge_get_string_fn(IManagedHandle, char16_t*, int32)`（`IScriptTypes.h:89`）：C# 用 `char*`（2 字节）接收，写完后 `InBuffer[Length] = '\0'`，返回 `Length`。✅ `char16_t` ↔ `char` 宽度一致。

---

### 4.3 `IManagedHandle`：真正的 ABI 隐患所在

```cpp
// Source/UnrealCSharpCore/Public/Domain/Script/IManagedHandle.h:5-7（原样）
struct IManagedHandle
{
    int64 Value{};
```

C++ 侧句柄是**含单个 `int64` 的结构体，按值传递 = 8 字节**。
C# 侧声明是 `nint`（`TArrayImplementation.cs:9`、`TMapImplementation.cs:9`、`TOptionalImplementation.cs:10`）= **指针宽度**。

- 64 位（Win64 / Linux x64 / Mac / Android arm64 / iOS arm64）：`nint` = 8 字节 → 匹配。
- ~~**32 位（Win32 / Android armeabi-v7a / 旧 iOS armv7）：`nint` = 4 字节，C++ 仍按 8 字节结构体传参** → 调用约定不匹配，栈/寄存器错位。见 F-INT2-002。~~ **（更正：本插件不支持也无法构建 32 位目标 —— `.uplugin` 无 `PlatformAllowList`、`UEBuildAndroid.cs:143-150` 只允许 arm64/x64、插件内 32 位分支命中 0、第三方库只有 64 位目录。该后果**不可达**，F-INT2-002 定为 P3「缺 `static_assert`」）**

### 4.4 缓冲区类型：`uint8*` ↔ `byte*` ↔ `stackalloc`

| C++ 宏 | 展开（`BufferMacro.h:9-29`） | C# 类型 | C# 缓冲区来源 |
|---|---|---|---|
| `IN_VALUE_BUFFER_SIGNATURE` | `uint8* InValueBuffer` | `byte*` | `stackalloc byte[sizeof(T)]`（值类型）或 `stackalloc byte[sizeof(nint)]`（引用类型），`TArray.cs:80/108` |
| `IN_KEY_BUFFER_SIGNATURE` | `uint8* InKeyBuffer` | `byte*` | 同上，`TMap.cs` |
| `RETURN_BUFFER_SIGNATURE` | `uint8* ReturnBuffer` | `byte*` | `stackalloc byte[sizeof(nint)]`，`TArrayImplementation.cs:217` |

C# 侧 `TArray_GetCompoundImplementation<T>`（`TArrayImplementation.cs:215-224`）假设**复合类型的返回缓冲恒为一个 `nint` 句柄**：

```csharp
// TArrayImplementation.cs:215-224（原样）
public static T TArray_GetCompoundImplementation<T>(nint InArray, int InIndex)
{
    var ValueBuffer = stackalloc byte[sizeof(nint)];
    __TArray_GetImplementation(InArray, InIndex, ValueBuffer);
    var Handle = *(nint*)ValueBuffer;
    return Handle != 0 ? (T)HandleData.GetObject(Handle) : default;
}
```

**这意味着 C++ 侧 `FPropertyDescriptor::Get` 对复合类型必须写满 8 字节句柄**。若某个 descriptor 走的是「写值语义」（例如把 `FString` 的 16 字节内联进缓冲区），就会**栈溢出写**（C# 缓冲只有 8 字节）。~~见 F-INT2-010。~~
**⚠️ 更正：这个"如果"在实际产物中不成立** —— 生成器把**所有** UStruct 都生成为 C# `class`（`Script/UE/Proxy/CoreUObject/UObject/Vector.cs:12`：`public partial class FVector : IStaticStruct`），`Script/UE/Proxy/**` 内 `public struct` 命中 **0**；因此 C# 永远不会为 UE 复合类型给出 `sizeof(T)` 内联缓冲，"写内联值 / 读句柄"两侧始终一致。**F-INT2-010 已撤销（非缺陷）**，其"栈溢出写"后果不可达 —— 详见该条。

---

## 5. 内存所有权 / 生命周期表（容器与指针）

| 类型 | C++ 持有者 | 分配点 | 释放点 | C# 是否拿到内部指针 | 判定 |
|---|---|---|---|---|---|
| `TArray` | `FArrayHelper`（`new`） | `FReflectionRegistry`/`FCSharpBind::Bind<FArrayHelper>` | `RemoveContainerReference<FArrayHelper>` via `AsyncTask`（`FRegisterArray.cpp:34-41`） | **`Get` 内部先取内部裸指针**（`FArrayHelper.cpp:123-131`），但导出层随即交给 `PropertyDescriptor->Get` 转换后写入 RETURN_BUFFER（`FRegisterArray.cpp:115-117`）→ **C# 拿到的是转换结果，不是容器内部地址** | 无悬垂（对 GoV/句柄型 descriptor）；但见 F-INT2-010 |
| `TMap` | `FMapHelper` | 同上 | 同上（`:23-30`） | 同上（`:91-92,102-103,124-125,167-169,179-181`） | 同上 |
| `TSet` | `FSetHelper` | 同上 | 同上（`:21-27`） | 同上（`:110-112`） | 同上 |
| `TOptional` | `FOptionalHelper`（`new`，`:32`/`:57`） | `Register1/2Implementation` | `RemoveOptionalReference` via `AsyncTask`（`:87-93`） | `Get` 返回**句柄**（`:113-125`），非内部地址 | 无悬垂 |
| `TLazyObjectPtr` | `new TLazyObjectPtr<UObject>`（`:17`） | `RegisterImplementation` | `RemoveMultiReference<...>` via `AsyncTask`（`:38-45`） | 否（`Bind` 一个 UObject） | ⚠️ F-INT2-003 |
| `TSubclassOf` | `new TSubclassOf<UObject>`（`:17`） | `RegisterImplementation` | 同上（`:38-44`） | 否 | ⚠️ F-INT2-003 |

### 5.0b 字符串 / 委托族的原生内存所有权

| 类型 | C++ 分配点 | 释放点 | `bNeedFree` | 判定 |
|---|---|---|---|---|
| `FString` | `new FString(...)`（`FRegisterString.cpp:15`） | `RemoveStringReference<FString>` via `AsyncTask`（`:38`） | `AddStringReference<FString, true, false>`（`:17`）→ **true = 需要释放** | 配对 ✅ |
| `FName` | `new FName(...)`（`FRegisterName.cpp:15`）+ `new FName(NAME_None)`（`:56`） | `RemoveStringReference<FName>`（`:38`） | `<FName, true, false>`（`:17, :55`） | 配对 ✅ |
| `FText` | `new FText()`（`FRegisterText.cpp:34`） | `RemoveStringReference<FText>`（`:64`） | `<FText, true, false>`（`:43`） | 配对 ✅（但失败时仍注册，见 F-INT2-018） |
| `FAnsiString` | `new FAnsiString(...)`（`FRegisterAnsiString.cpp:17`） | `RemoveStringReference<FAnsiString>`（`:42`） | `<FAnsiString, true, false>`（`:21`） | 配对 ✅ |
| `FUtf8String` | `new FUtf8String(...)`（`FRegisterUtf8String.cpp:17`） | `RemoveStringReference<FUtf8String>`（`:42`） | `<FUtf8String, true, false>`（`:21`） | 配对 ✅ |
| `FDelegateHelper` | `FCSharpBind::Bind<FDelegateHelper>`（`FRegisterDelegate.cpp:18`） | `RemoveDelegateReference<FDelegateHelper>`（`:25`） | — | 配对 ✅ |
| `FMulticastDelegateHelper` | `FCSharpBind::Bind<FMulticastDelegateHelper>`（`FRegisterMulticastDelegate.cpp:18`） | `RemoveDelegateReference<...>`（`:25`） | — | 配对 ✅ |

**注册表的实际释放动作**（以 `FStringRegistry::Deinitialize` 为例，`:20-34`）：`FDomain::GCHandle_Free(Key)` **与** `FMemory::Free(Value.Value)` 成对执行——即原生对象与托管句柄都被释放。这证明**释放机制本身是完整的**；F-INT2-011 的问题恰恰在于**这条路径只覆盖"登记在 C++ 注册表里的句柄"，不覆盖"C++ 现场新建并直接返回给 C# 的句柄"**。

**关于"是否用 `memcpy` 搬运非平凡类型"（任务书重点）**：**未发现任何 `memcpy`**——本目录 18 个文件与 `FArrayHelper.cpp` 全文中 `memcpy` / `FMemory::Memcpy` 命中数为 **0**。所有非平凡类型的搬运一律经由 `FPropertyDescriptor::Set` / `Get` / `Identical`（`FArrayHelper.cpp:137,269`、`FArrayHelper.cpp:148,164,180,293`）。`TSet`/`TMap` 同理（导出层只调用 `SetHelper->Add`、`MapHelper->Add/Set`）。**这是本模块最健康的一个维度**——具体证据：

```cpp
// FArrayHelper.cpp:263-272（原样）—— 正确的"先 AddUninitialized 再 Set"
int32 FArrayHelper::Add(void* InValue) const
{
    auto ScriptArrayHelper = CreateHelperFormInnerProperty();
    const auto Index = ScriptArrayHelper.AddUninitializedValue();
    InnerPropertyDescriptor->Set(InValue, ScriptArrayHelper.GetRawPtr(Index));
    return Index;
}
```

`RemoveAt` 也正确地按 `CPF_IsPlainOldData | CPF_NoDestructor` 决定是否 `DestroyValue`（`FArrayHelper.cpp:214-220`）。

**唯一的例外是 `SwapMemory` / `Swap`**，见 F-INT2-004。

### 5.1 委托解绑与去重路径核查（任务书重点）

**结论：解绑路径齐备，去重能力齐备但非默认，未发现 P0/P1 级的"绑定了却无法解绑"。** 但存在一个与 F-INT2-011 同源、且由于调用点数量巨大而更严重的句柄泄漏（见下方第 4 点）。

1. **`TDelegate` 的解绑路径** —— 齐备：
   - `UnBind`（`FRegisterDelegate.cpp:62-69`）→ `DelegateHelper->UnBind()`
   - `Clear`（`:71-78`）→ `DelegateHelper->Clear()`
   - `UnRegister`（`:21-28`）→ `RemoveDelegateReference<FDelegateHelper>`（对象级销毁）
   - C# 侧由生成器注入终结器：`ScriptCodeGenerator/Private/FDelegateGenerator.cpp:79` 生成 `~%s() => FDelegateImplementation.FDelegate_UnRegisterImplementation(HandleData.GetHandle(this));`，`:332/:336` 生成 `Unbind()` / `Clear()` 公开方法。

2. **`TMulticastDelegate` 的解绑路径** —— 齐备：
   - `Remove`（`FRegisterMulticastDelegate.cpp:106-125`）、`RemoveAll`（`:127-137`）、`Clear`（`:139-146`）、`IsBound`（`:30-39`）、`Contains`（`:41-62`）
   - 生成器对应：`FDelegateGenerator.cpp:653/:657/:661`。

3. **重复绑定是否去重** —— **`Add` 不去重，`AddUnique` 去重，两者都暴露**。这与 UE 原生语义一致（`AddDynamic` vs `AddUniqueDynamic`），且提供了 `Contains`（`:41-62`）供调用者自查。
   ```cpp
   // FRegisterMulticastDelegate.cpp:64-104（原样，节选）
   static void AddImplementation(...)        // :64  —— 允许重复
   { ... MulticastDelegateHelper->Add(FoundObject, FoundMethod); ... }

   static void AddUniqueImplementation(...)  // :85  —— 去重
   { ... MulticastDelegateHelper->AddUnique(FoundObject, FoundMethod); ... }
   ```
   ✅ 未发现问题。**去重语义的实现细节在 `FMulticastDelegateHelper`（非本报告范围），未验证。**

4. **⚠️ 委托路径是 F-INT2-011 泄漏的重灾区**：`Bind` / `Add` / `AddUnique` / `Remove` / `Contains` 每次调用都会对两个托管反射对象分配句柄：
   ```csharp
   // Script/UE/Library/FMulticastDelegateImplementation.cs:39-46（原样）
   public static void FMulticastDelegate_AddImplementation(nint InDelegate, nint InObject, Type InType,
       MethodInfo InMethodInfo)
   {
       __FMulticastDelegate_AddImplementation(InDelegate, InObject, HandleData.Alloc(InType),
           HandleData.Alloc(InMethodInfo));      // ← 两个句柄，永不释放
   }
   ```
   这些 `Type` / `MethodInfo` 是**长期存活的反射对象**，`HandleData.Alloc` 对它们只会把既有引用计数 `Count++`（`HandleData.cs:31-34`），而 C# 侧从不调用 `HandleData.Free`（全量 16512 个 `.cs` 文件中 `HandleData.Free(` 命中数为 **0**）。后果是**引用计数单调上涨、永不归零**，这些反射对象的 `GCHandle` 再也无法被释放。
   而 `FMulticastDelegate_AddImplementation` 在 `Script/UE/Proxy/**` 生成的代理代码中有 **321 个调用点**（**🔁 抽测实测 316**；`FDelegate_*` 系列各 90 个），因此任何"每次 Spawn 都 `Add` 一个委托"的常见写法都会稳定泄漏句柄与计数。

5. **C# 委托被 GC 后是否悬垂** —— **不悬垂，但代价是能力受限**。绑定时不保留托管委托实例，只取反射元数据：
   ```csharp
   // Plugins/UnrealCSharp/Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:328（原样，生成到 C#）
   "public void Bind(UObject InObject, Delegate InDelegate) => FDelegateImplementation.FDelegate_BindImplementation(HandleData.GetHandle(this), HandleData.GetHandle(InObject), InDelegate.Method.DeclaringType, InDelegate.Method);\n\n"
   ```
   传过去的是 `InDelegate.Method.DeclaringType` 与 `InDelegate.Method`（两个**反射对象**），C++ 侧再经 `FoundClass->GetMethod(InManagedMethod)` 解析成 UFunction（`FRegisterDelegate.cpp:42-44`）。因此：
   - ✅ 托管委托实例本身不被 C++ 持有 → **不存在托管委托被 GC 后 C++ 侧悬垂**的问题；
   - ✅ 被绑定的 `UObject` 通过 `GetObject(InObject)` 转为原生指针（`FRegisterDelegate.cpp:38`），由 UE 自身的 `FScriptDelegate` 弱引用语义管理；
   - ⚠️ **代价**：只能绑定"某个 `MethodInfo` 所描述的方法"，**无法绑定 C# 闭包 / lambda / 局部委托**（没有地方承载捕获的环境）。这是设计上的能力边界，不是 bug，但值得在文档里写明。

6. **`FRegisterDelegate` / `FRegisterMulticastDelegate` 缺少 `Identical` 导出**（对比容器/字符串/指针族都有）。不影响功能（委托比较另有 `Contains`/`IsBound`），属一致性差异。

---

## 6. 发现清单

**汇总：16 条 —— 复核后最终级别：P0 × 1、P1 × 4、P2 × 1、P3 × 8、撤销（非缺陷）× 2。**
（原文汇总为 P0×1 / P1×6 / P2×4 / P3×5；**下调 4 条**（002/004/005/015）、**上调 1 条**（012）、**撤销 1 条**（010）、**维持 9 条**，另有 009 已撤销、此处仅补证。）
**编号 ↔ 最终级别 ↔ 物理所在章节速查**（⚠️ 各 Finding 的**物理分组标题**仍保留**原定级**（为保持编号锚点稳定，未搬迁块体），**级别一律以各条 `- **严重度**` 字段与上表为准**）：

| 编号 | 最终级别 | 物理所在分组 | 备注 |
|---|---|---|---|
| F-INT2-001 | P0 | `### P0` | 维持 |
| F-INT2-011 / 013 / 003 | P1 | `### P1` | 维持 |
| F-INT2-012 | **P1** | `### P2` | 由 P3 上调，物理位置未搬迁 |
| F-INT2-006 | P2 | `### P2` | 维持 |
| F-INT2-002 / 004 | **P3** | `### P1` | 由 P1/P1 下调 |
| F-INT2-015 / 005 | **P3** | `### P2` | 由 P2/P2 下调 |
| F-INT2-014 / 018 / 007 / 008 | P3 | `### P3` | 维持 |
| F-INT2-010 | **撤销（非缺陷）** | `### P1` | 由 P1→P2→撤销 |
| F-INT2-009 | **撤销（非缺陷）** | `### P3` | 已撤销，此处补证 |

| 编号 | 严重度 | 一句话 | 文件 |
|---|---|---|---|
| F-INT2-001 | **P0** | `TArray` 的 `RemoveAt`/`InsertZeroed`/`SetNum`/`SwapMemory`/`Swap` 无 index 校验 → Shipping 下堆破坏 | `FRegisterArray.cpp` |
| F-INT2-011 | P1 | C++ 现场新建并直接返回给 C# 的托管句柄从不释放 → `HandleData` 强 `GCHandle` 与字典无界泄漏（`ToString()`、委托 `Add` 等 411+ 调用点） | `FRegisterString.cpp` 等 8 处 |
| F-INT2-013 | P1 | 4 个字符串 `ToString` 对 `GetString` 结果不判空（仅 `FAnsiString` 判了） | `FRegister{String,Name,Text,Utf8String}.cpp` |
| F-INT2-002 | **P3**（原 P1） | `IManagedHandle`（`int64` 按值）vs C# `nint` → 32 位目标 ABI 不匹配；**本插件不支持 32 位**，只剩"缺 `static_assert`" | 本目录全部导出 |
| F-INT2-003 | P1 | 5 个智能指针族的 `Get`/`LoadSynchronous` 对 `GetMulti` 结果不判空（8 处） | `FRegister{Lazy,Weak,SoftClass,SoftObject}ObjectPtr/SubclassOf` |
| F-INT2-004 | **P3**（原 P1） | `FArrayHelper::Swap` 与 `SwapMemory` 实现完全相同；**但 UE 的 `TArray::Swap` 也只是字节交换** → 非"语义错误"，仅"缺 `check` + API 冗余" | `FRegisterArray.cpp:298` |
| F-INT2-010 | **撤销（非缺陷）**（原 P1→P2） | 容器「值/句柄」双模式契约靠 `IsValueType` 与 `IsPrimitiveProperty()` 隐式对齐 —— **不一致类不存在**（生成器把 UStruct 全生成为 C# `class`，`Proxy/**` 内 `public struct` 命中 0） | `FRegisterArray/Map/Set/Optional.cpp` |
| F-INT2-012 | **P1**（原 P2→P3） | `FAnsiString`/`FUtf8String` 出口把**窄串硬转成 `const TCHAR*`** 再"转 UTF-8" → **确证的编码错误 + 越界读**（`StringConv.h:1021`）；另有冗余拷贝 | `FRegister{Ansi,Utf8}String.cpp` |
| F-INT2-015 | P3 | 两个纯头文件的注册实例是 `static` 而非 `inline` → `EForceInit` **重复注册 8 次**、`EObjectFlags` **2 次**（追加语义已证实，但下游幂等 ⇒ 仅冗余） | `FRegister{ForceInit,ObjectFlags}.h` |
| F-INT2-005 | P3 | **56 处**「取容器 + 判空」样板逐字重复，可宏化（原文"64/63"计数已更正） | `FRegister{Array,Map,Set,Optional}.cpp` |
| F-INT2-006 | P2 | 所有 `UnRegister` 走 `AsyncTask(GameThread)` → 已注销未释放窗口、关闭期 UAF、潜在泄漏 | 6 个文件 |
| F-INT2-014 | P3 | `FRegisterMulticastDelegate` 重复注册 `Contains` → 生成死名 `Contains1` | `FRegisterMulticastDelegate.cpp:192` |
| F-INT2-018 | P3 | `FText::Register` 忽略 `ReadFromBuffer` 返回值；同文件文件级关闭 `-Wdangling` | `FRegisterText.cpp` |
| F-INT2-007 | P3 | `FArrayHelper::Remove` 有未使用局部变量，且为 O(n²) | `FRegisterArray.cpp:277` |
| F-INT2-008 | P3 | 「失败即返回 0」与合法值冲突（`Num`/`GetMaxIndex`） | `FRegisterArray/Map/Set.cpp` |
| F-INT2-009 | **撤销（非缺陷）** | LeanCLR 后端走 `[DllImport]` 但注册面无 C 符号 —— **已证伪**：`FLeanCLRDomain::RegisterBinding()`（`FLeanCLRDomain.cpp:854-888`）把每个绑定方法按名字注册进 LeanCLR 的 P/Invoke 表 | 生成器 + LeanCLR 域 |

> 编号为**发现顺序**；下面按**严重度降序**排列（P0 → P1 → P2 → P3）。
> **本次分析的方法论要点**：「`Script/` 中无调用方」不能判定为死代码 —— 本仓库有**两套 `Script/` 树**（插件自带 + 项目级生成代理 `Script/UE/Proxy/**`），只看一套会得到完全相反的结论：委托族的 29 个包装层实际有 90～321 个调用点。**教训：单树 grep 会系统性误判死代码。**

### P0

> （分组标题为**原定级**；与最终级别不一致时的对照见上方速查表）

#### [F-INT2-001] `TArray` 的 `RemoveAt` / `InsertZeroed` / `SetNum` / `SwapMemory` / `Swap` 导出未做 index 校验，Shipping 下构成堆破坏

- **类别**: 未定义行为 / 安全
- **严重度**: **P0**（崩溃 / 数据损坏）
- **复核结论**: 确认（代码事实、调用上下文、类别、P0 级严重度全部成立；已用引擎源码闭合原文唯一的存疑点）
- **可达性**: 活跃
- **复核证据**: `FRegisterArray.cpp:175-183 / 195-203 / 223-231 / 288-306` 五个导出**零 index 校验**（逐字比对通过）；`FArrayHelper.cpp:208-228`（`Dest = ScriptArrayHelper.GetRawPtr(InIndex)`，`:212` 未校验）、`:196-199`、`:251-261`、`:324-332` 同样零校验；C# 公开层 `TArray.cs:190-198`、`:206-207`、`:306-312` 全部直接透传（同文件 `:63-64` 已有 `IsValidIndex` 却不用）。**引擎侧（决定性）**：`FScriptArrayHelper::GetRawPtr` 的**唯一**保护是 `checkSlow(IsValidIndex(Index))`（`Runtime/CoreUObject/Public/UObject/UnrealType.h:3911-3920`，断言在 `:3918`）；`FScriptArray::Remove` 的 4 条校验全是 `checkSlow`（`Runtime/Core/Public/Containers/ScriptArray.h:195-198`）；`FScriptArray::InsertZeroed`→`Insert` 用的是 `check`（`ScriptArray.h:57-61`）；`FScriptArray::SwapMemory` **连 `checkSlow` 都没有**（`ScriptArray.h:162-169`）。而 `Misc/AssertionMacros.h:237/245/325/327`（`DO_CHECK`）与 `:340-348`（`DO_GUARD_SLOW`）证明 **Test/Shipping 下 `check` 与 `checkSlow` 均展开为 `CA_ASSUME(expr)`（无运行时行为）**。即：Shipping 下越界 `Index` 直接进 `FMemory::Memmove`/`Memswap`/`DestructItems`，`RemoveAt(0, 2)`（1 个元素）会算出 `NumToMove = -1` → `(SIZE_T)(-1) * ElementSize` 的巨型 `Memmove` → 立即堆破坏。**终裁：维持 P0**（唯一保留意见：UE 原生 `TArray` 的 `RangeCheck` 同为 `checkf`，也在 Shipping 失效，故此条带有"继承引擎语义"的成分；但本导出层是**无类型的托管边界**，C# 侧无 Debug 断言可依赖，且同层 `Get`/`Set` 已自证有能力校验 → 判为缺陷而非设计约定）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterArray.cpp:175-183, 195-203, 223-231, 288-306`；下游 `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:208-228, 324-332`
- **函数**: `FRegisterArray::RemoveAtImplementation(IManagedHandle, int32, int32, uint8)`、`InsertZeroedImplementation`、`SetNumImplementation`、`SwapMemoryImplementation`、`SwapImplementation`
- **级别变动**: 无（P0 维持）
- **置信度**: **高**（原文为"中"，因为当时未读引擎源码；现已读到 `checkSlow`/`check` 的宏展开与 `GetRawPtr`/`Remove`/`SwapMemory` 的真实实现，存疑点已闭合）

**现状（代码事实）**

```cpp
// FRegisterArray.cpp:195-203（原样）
static void RemoveAtImplementation(const IManagedHandle InManagedHandle,
                                   const int32 InIndex, const int32 InCount, const uint8 bAllowShrinking)
{
    if (const auto ArrayHelper = FCSharpEnvironment::GetEnvironment().GetContainer<FArrayHelper>(
        InManagedHandle))
    {
        ArrayHelper->RemoveAt(InIndex, InCount, bAllowShrinking != 0);
    }
}
```

```cpp
// FRegisterArray.cpp:175-183（原样）—— 完全没有 index 相关校验
static void InsertZeroedImplementation(const IManagedHandle InManagedHandle,
                                       const int32 InIndex, const int32 InCount)
{
    if (const auto ArrayHelper = FCSharpEnvironment::GetEnvironment().GetContainer<FArrayHelper>(
        InManagedHandle))
    {
        ArrayHelper->InsertZeroed(InIndex, InCount);
    }
}
```

下游实现（这才是破坏发生的地方）：

```cpp
// FArrayHelper.cpp:208-228（原样，节选）
void FArrayHelper::RemoveAt(const int32 InIndex, const int32 InCount, const bool bAllowShrinking) const
{
    auto ScriptArrayHelper = CreateHelperFormInnerProperty();

    auto Dest = ScriptArrayHelper.GetRawPtr(InIndex);   // ← 先取指针，Index 未校验

    if (!(InnerPropertyDescriptor->GetPropertyFlags() & (CPF_IsPlainOldData | CPF_NoDestructor)))
    {
        for (auto Index = 0; Index < InCount; ++Index, Dest += InnerPropertyDescriptor->GetElementSize())
        {
            InnerPropertyDescriptor->DestroyValue(Dest);   // ← 对越界地址调用析构
        }
    }

    ScriptArray->Remove(InIndex, InCount, InnerPropertyDescriptor->GetElementSize(), __STDCPP_DEFAULT_NEW_ALIGNMENT__);
    ...
}
```

```cpp
// FArrayHelper.cpp:196-199（InsertZeroed，原样）—— 直接进 FScriptArray
void FArrayHelper::InsertZeroed(const int32 InIndex, const int32 InCount) const
{
    ScriptArray->InsertZeroed(InIndex, InCount, InnerPropertyDescriptor->GetSize(), __STDCPP_DEFAULT_NEW_ALIGNMENT__);
}
```

```cpp
// FArrayHelper.cpp:324-332（原样）
void FArrayHelper::SwapMemory(const int32 InFirstIndexToSwap, const int32 InSecondIndexToSwap) const
{
    ScriptArray->SwapMemory(InFirstIndexToSwap, InSecondIndexToSwap, InnerPropertyDescriptor->GetSize());
}

void FArrayHelper::Swap(const int32 InFirstIndexToSwap, const int32 InSecondIndexToSwap) const
{
    ScriptArray->SwapMemory(InFirstIndexToSwap, InSecondIndexToSwap, InnerPropertyDescriptor->GetSize());
}
```

**调用上下文**

C# 公开层把**未经校验的 `InIndex` 直接透传**，而**同一份 C# API 里明明已经暴露了 `IsValidIndex`** 却不在这些路径上使用：

```csharp
// Script/UE/CoreUObject/TArray.cs:63-64（原样）—— 有能力校验，且导出存在
public bool IsValidIndex(int InIndex) =>
    TArrayImplementation.TArray_IsValidIndexImplementation(HandleData.GetHandle(this), InIndex);
```

```csharp
// Script/UE/CoreUObject/TArray.cs:190-198, 306-312（原样）
public void InsertZeroed(int InIndex, int InCount = 1) =>
    TArrayImplementation.TArray_InsertZeroedImplementation(HandleData.GetHandle(this), InIndex, InCount);

public void InsertDefaulted(int InIndex, int InCount) =>
    TArrayImplementation.TArray_InsertDefaultedImplementation(HandleData.GetHandle(this), InIndex, InCount);

public void RemoveAt(int InIndex, int InCount, bool bAllowShrinking = true) =>
    TArrayImplementation.TArray_RemoveAtImplementation(HandleData.GetHandle(this), InIndex, InCount,
        bAllowShrinking);

public void SwapMemory(int InFirstIndexToSwap, int InSecondIndexToSwap) => ...
public void Swap(int InFirstIndexToSwap, int InSecondIndexToSwap) => ...
```

注意 `InsertDefaulted`（`:193`）与 `RemoveAt`（`:196`）**连 `InCount` 都没有默认值保护**：`RemoveAt(5, 0)` 或 `RemoveAt(-1, 1)` 都要走到上面的析构循环。

对比：`Get`/`Set` 是安全的，因为 `FArrayHelper::Get`/`Set` 自己做了 `IsValidIndex` 检查（`FArrayHelper.cpp:125,135`）。**同一层 API 内部安全策略不一致**，说明这不是设计决定而是遗漏。

**问题**

`FArrayHelper::RemoveAt` 在取得 `Dest = GetRawPtr(InIndex)` 时**唯一的保护是 `FScriptArrayHelper::GetRawPtr` 内部的 `checkSlow(IsValidIndex(Index))`**（真实位置：`Runtime/CoreUObject/Public/UObject/UnrealType.h:3911-3920`，断言在 `:3918`；原文写的 `Containers/ScriptArrayHelper.h` 在 UE 5.6 已不存在）。`checkSlow` 在 Test / Shipping 配置下被编译掉（`Misc/AssertionMacros.h:340-348`：`DO_GUARD_SLOW` 为 0 时 `checkSlow(expr)` → `{ CA_ASSUME(expr); }`），因此：

1. 越界 `InIndex` → `Dest` 指向分配块之外；
2. 对 `TArray<FString>` / `TArray<UObject*>` 等**非 POD** 元素，循环对这块垃圾内存调用 `DestroyValue(Dest)`；
3. `FString` 的析构会去 `free` 一个从垃圾字节读出的指针 → **任意地址释放 / 立即崩溃**；
4. 即便是 POD 元素，随后 `ScriptArray->Remove(InIndex, InCount, ...)` 也会 `FMemory::Memmove` 越界长度（`Count` 取自参数，不受容器长度约束）——`RemoveAt(0, 1000000)` 是稳定的堆溢出。

触发路径无需恶意输入：**C# 侧一个再普通不过的写法就够了**：

```csharp
var Arr = new TArray<FString>();
Arr.Add("A");
Arr.RemoveAt(0, 2);   // InCount 大于剩余元素数
```

`InsertZeroed(-1, 1)`、`SetNum(-1)`（`FArrayHelper.cpp:251-261` 直接把 `InNewNum` 交给 `Resize`）、`Swap(0, 999)` 同理。

**建议**

在 `FArrayHelper` 的**变更类**方法里统一前置校验（这是与 `Get`/`Set` 一致的、最小侵入的修法）：

```cpp
void FArrayHelper::RemoveAt(const int32 InIndex, const int32 InCount, const bool bAllowShrinking) const
{
    const auto Total = Num();
    if (InIndex < 0 || InIndex > Total || InCount < 0 || InCount > Total - InIndex) { return; }  // 或 ensureMsgf
    ...
}
```

`InsertZeroed` / `InsertDefaulted`：`if (InIndex < 0 || InIndex > Num()) return;`
`Swap` / `SwapMemory`：`if (!IsValidIndex(a) || !IsValidIndex(b)) return;`
`SetNum`：`if (InNewNum < 0) return;`

若希望**保留 UE 语义**（越界即崩溃），则应改为 `checkf`（Shipping 也生效）而非静默返回——**这是一个需要产品决策的取舍**，但当前"只在 Debug 有断言"是最坏组合：开发期看不到（因为 C# 侧没有触发），发布期直接内存破坏。

**验证方式**

- grep 确认无校验：`grep -n "IsValidIndex" FArrayHelper.cpp` → 仅 `:99-104` 定义与 `:125,135` 在 `Get`/`Set` 中使用，变更类方法零命中。
- 用例：C# 构造 `TArray<FString>`，`Add` 一个元素后调用 `RemoveAt(0, 5)`，在 Shipping/Win64 运行；配合 `-fsanitize=address`（Linux）或 PageHeap（Win）可稳定复现。
- 加断言：在 `FArrayHelper::RemoveAt` 入口加 `check(IsValidIndex(InIndex) && InCount <= Num() - InIndex)`，用现有测试跑一遍即可看到是否被触发。

---

### P1

> （分组标题为**原定级**。本组内含 3 条维持 P1、2 条已下调为 P3（F-INT2-002 / F-INT2-004）、1 条已撤销（F-INT2-010）—— 以各条 `**严重度**` 与上方速查表为准）

#### [F-INT2-011] C++ 现场新建并直接返回给 C# 的托管句柄从不释放 —— `HandleData` 强 `GCHandle` 与字典条目无界泄漏

- **类别**: 内存/资源泄漏
- **严重度**: **P1**（泄漏；触发条件为最常见的脚本写法）
- **复核结论**: 确认
- **可达性**: 活跃
- **复核证据**: `HandleData.cs:24-25`（`Free` 是 `[UnmanagedCallersOnly]`，只等 C++ 来调）、`:27-44`（`Alloc`：新建 `Handles[++Handle] = GCHandle.Alloc(obj, GCHandleType.Normal)` = **强引用**）、`:46-78`（`FreeImplementation` 由 `Count` 归零才真正释放）；`FStringImplementation.cs:37-42` 取到 `Handle` 后**只读不还**（`:41`）；`Script/Interop/Bridge/StringBridge.cs:8-17` 每次 `NewString` 都 `Alloc` 一个新的托管 `string`。**"C# 侧从不释放"已全量复跑**：`grep "HandleData\.Free"` 在**两套 `Script/` 树**命中 **0**（`Contains1` 同样 0）；对照实现 `TMethodHelper.inl:88-95`（`:92` `FDomain::GCHandle_Free(ReturnValue)`）确认"正确做法"存在而本层缺失。
  **⚠️ 更正原文一处事实错误**：原文称"`FDomain::GCHandle_Free` 的全部调用点都在 `Registry/*.cpp` 的 `Deinitialize()`"。实测 `grep GCHandle_Free Source/` 共 **32** 处（含 1 处声明 `FDomain.h:32`、1 处定义 `FDomain.cpp:94`），真正的释放调用点远不止 `Deinitialize()`：`FStringRegistry.inl:76`、`FContainerRegistry.inl:76`、`FDelegateRegistry.inl:75`、`FStructRegistry.cpp:25,117`、`FOptionalRegistry.cpp:31`、`FObjectRegistry.cpp:24,90,108`、`FMultiRegistry.cpp:22,40,58,76,94,112`、`FDelegateRegistry.cpp:29,47`、`FContainerRegistry.cpp:29,47,65`、`FBindingRegistry.cpp:23,60`、`FDomain.cpp:122`。**但结论不变**：这些点全部只释放"**登记进 C++ 注册表**的句柄"（`FStringRegistry.cpp:22,40,59,79,98` 那 5 处已逐一核对无误），而 `FRegisterString::ToStringImplementation` 现场新建的句柄**从未进入任何注册表**，因此没有任何 `GCHandle_Free` 会碰到它。
- **级别变动**: 无（P1 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterString.cpp:42-47`、`FRegisterName.cpp:42-47, 49-59`、`FRegisterText.cpp:68-73`、`FRegisterAnsiString.cpp:47-54`、`FRegisterUtf8String.cpp:46-51`、`FRegisterArray.cpp:109-119`、`FRegisterMap.cpp:85-105, 118-127, 161-183`、`FRegisterSet.cpp:105-114`、`FRegisterOptional.cpp:113-125`
- **函数**: 所有**返回 `IManagedHandle`** 或**通过 `RETURN_BUFFER` 回写句柄**的导出；**以及** C# 侧所有把 `Type`/`MethodInfo` 交给 `HandleData.Alloc` 的包装层（`FDelegateImplementation.cs`、`FMulticastDelegateImplementation.cs`）
- **置信度**: 高（分配点、释放点、以及"正确做法"的对照实现三者均已读到）

**现状（代码事实）**

分配点在 C# 侧，`NewString` 对**每次调用都创建一个全新的托管 `string`**，并为其分配一个强 `GCHandle`：

```csharp
// Script/Interop/Bridge/StringBridge.cs:8-17（原样）
[UnmanagedCallersOnly]
public static nint NewString(byte* InText)
{
    if (InText != null)
    {
        return HandleData.Alloc(Marshal.PtrToStringUTF8((nint)InText)!);
    }

    return 0;
}
```

```csharp
// Script/Interop/Handle/HandleData.cs:27-44（原样）
public static nint Alloc(object InObject, bool bPinned = false)
{
    lock (Lock)
    {
        var HandleReference = ObjectToHandleReference.GetValue(InObject, _ =>
            new HandleReference { Value = ++Handle, Count = 0 });

        HandleReference.Count++;

        if (!Handles.TryGetValue(HandleReference.Value, out _))
        {
            Handles[HandleReference.Value] =
                GCHandle.Alloc(InObject, bPinned ? GCHandleType.Pinned : GCHandleType.Normal);
        }

        return HandleReference.Value;
    }
}
```

释放点由 `Count` 引用计数控制，**只有被减到 0 才真正 `Free`**：

```csharp
// Script/Interop/Handle/HandleData.cs:46-78（原样，节选）
public static void FreeImplementation(nint InHandle)
{
    if (InHandle != 0)
    {
        lock (Lock)
        {
            if (Handles.TryGetValue(InHandle, out var OutHandle))
            {
                var Target = OutHandle.Target;

                if (Target != null)
                {
                    if (ObjectToHandleReference.TryGetValue(Target, out var OutHandleReference))
                    {
                        if (OutHandleReference.Value == InHandle)
                        {
                            OutHandleReference.Count--;

                            if (OutHandleReference.Count > 0)
                            {
                                return;      // ← 计数未归零，句柄继续存活
                            }
                        }
                    }
                }

                Handles.Remove(InHandle);

                OutHandle.Free();
            }
        }
    }
}
```

C++ 侧把句柄直接交给 C#，**不留任何副本**：

```cpp
// FRegisterString.cpp:42-47（原样）
static IManagedHandle ToStringImplementation(const IManagedHandle InManagedHandle)
{
    const auto String = FCSharpEnvironment::GetEnvironment().GetString<FString>(InManagedHandle);

    return IScriptDomain::Get()->NewString(TCHAR_TO_UTF8(**String));
}
```

C# 侧取出对象后**丢弃句柄、不做任何释放**：

```csharp
// Script/UE/Library/FStringImplementation.cs:35-42（原样）
private static unsafe partial nint __FString_ToStringImplementation(nint InString);

public static unsafe string FString_ToStringImplementation(nint InString)
{
    var Handle = __FString_ToStringImplementation(InString);

    return Handle != 0 ? (string)HandleData.GetObject(Handle) : null;   // ← Handle 被丢弃，从未 Free
}
```

**问题**

每一次 `FString` / `FName` / `FText` / `FAnsiString` / `FUtf8String` 的 `ToString()` 调用，以及每一次返回复合类型的 `TArray.Get` / `TMap.Get` / `TOptional.Get` 等，都会：

1. 在 `HandleData.ObjectToHandleReference` 里新建一个条目；
2. `HandleReference.Count` 从 0 变 1，`Handles[++Handle] = GCHandle.Alloc(obj, GCHandleType.Normal)`；
3. **该 `GCHandle` 是 `Normal` 类型 → 强引用，作为 GC root 永久存活**；
4. C# 拿到 `string` 后丢弃句柄 → `Count` 永远停留在 1 → **永远不可能被减到 0 → 永不释放**；
5. `Handles` 字典与 `Handle` 计数器**单调增长，永不回收**（`HandleData.Clear()` 仅在 `AssemblyLoader.cs:35` 的整个程序集卸载路径被调用，正常运行期不会触发）。

**触发面极广**，因为 `ToString()` 是 C# 语言层会被隐式调用的方法：

- `Script/UE/CoreUObject/FString.cs:42-43`、`FName.cs:43`、`FText.cs:46` 都 `override string ToString()`；
- 因此任何 **字符串插值 `$"{SomeFName}"`**、`string.Format`、`+` 拼接参与、`Debug.Log(obj)` 都会走到 `ToString()` → **每帧日志、每帧 UI 文本更新都会稳定泄漏一个 `GCHandle` + 一个字典条目 + 一个被 root 的 `string`**。

这是一条"游戏跑得越久内存越大、且泄漏对象无法被 GC"的典型 P1 泄漏。

**同类泄漏还有一条调用量更大的路径：委托包装层。** `FDelegateImplementation.cs` / `FMulticastDelegateImplementation.cs` 每个方法都 `HandleData.Alloc` 一个 `Type` 和一个 `MethodInfo`：

```csharp
// Script/UE/Library/FMulticastDelegateImplementation.cs:39-46（原样）
private static unsafe partial void __FMulticastDelegate_AddImplementation(nint InDelegate, nint InObject, nint InType, nint InMethodInfo);

public static void FMulticastDelegate_AddImplementation(nint InDelegate, nint InObject, Type InType,
    MethodInfo InMethodInfo)
{
    __FMulticastDelegate_AddImplementation(InDelegate, InObject, HandleData.Alloc(InType),
        HandleData.Alloc(InMethodInfo));      // ← 两个句柄，永不释放
}
```

这两个对象是**长期存活的反射对象**，因此 `HandleData.Alloc` 走的是 `ObjectToHandleReference.GetValue(InObject, ...)` 的"已存在"分支——`HandleReference.Count++`（`HandleData.cs:31-34`）而**不新建 `GCHandle`**。也就是说：

- 对 `ToString` 产生的**新** `string`：每次泄漏一个**新的 `GCHandle`** + 一个新的字典条目（`Handles[++Handle] = GCHandle.Alloc(...)`）；
- 对委托路径的 `Type`/`MethodInfo`：**不新建句柄，但把引用计数永久推高**。后果是这些反射对象的 `Count` 永远无法归零，即使将来有代码正确地调用 `HandleData.Free`，`FreeImplementation`（`HandleData.cs:64-67`）也会因 `Count > 0` 提前 `return` → **该 `GCHandle` 永远不能释放**。

调用量：`FMulticastDelegate_*` 系列每个方法在 `Script/UE/Proxy/**` 中有 **321 个调用点**（**🔁 抽测：`FMulticastDelegate_AddImplementation` 实测 316**），`FDelegate_*` 系列每个有 **90 个**（统计口径：`Library/*Implementation.cs` 之外的全量命中）。这意味着任何"每次 Spawn 一个带委托的 Actor 就 `Add` 一次"的写法都在稳定推高计数。

**对照证据**：同一个仓库里，**方法调用返回值的路径是正确释放的**：

```cpp
// Source/UnrealCSharp/Public/Binding/Function/TMethodHelper.inl:88-95（原样）
if constexpr (!std::is_void_v<Result>)
{
    auto Value = TPropertyValue<Result, Result>::Get(ReturnValue);

    FDomain::GCHandle_Free(ReturnValue);      // ← 正确：取出值后立刻释放句柄

    return Value;
}
```

而 `Library/*Implementation.cs` 这一层**完全没有**对应调用。全仓库证据：

- C# 侧：`HandleData.Free(` 在**两套 `Script/` 树共 16512 个 `.cs` 文件**中的命中数为 **0**（`HandleData.FreeImplementation(` 同样为 **0**）。唯一的调用点是 `HandleData.cs:25` 的定义本身——它是一个 `[UnmanagedCallersOnly]`，即**只等 C++ 来调**。
- C++ 侧：`FDomain::GCHandle_Free` 的全部调用点都在 `Registry/*.cpp` 的 `Deinitialize()`（例如 `FStringRegistry.cpp:22, 40, 59, 79, 98`）、`TMethodHelper.inl:92`、`FManagedFunctionDescriptor.cpp:64`。**这些都只释放"CI 侧注册表持有"的句柄**。
- `FRegisterString::ToStringImplementation` 产生的句柄**不进入任何 C++ 注册表**——它被直接 return 给 C#。因此没有任何 `GCHandle_Free` 会去释放它。
- 释放机制本身是完整的（`FStringRegistry::Deinitialize`，`:20-34`，`FDomain::GCHandle_Free(Key)` 与 `FMemory::Free(Value.Value)` 成对），**缺的只是"现场新建句柄"这一条路径的配对释放**。

另外注意 `FNameImplementation.cs:44-51` 与 `FName.cs:45`：

```csharp
// Script/UE/CoreUObject/FName.cs:45（原样）
public static FName NAME_None => FNameImplementation.FName_NAME_NoneImplementation();
```

`NAME_None` 是一个**属性**，每次读取都走一遍 C++ 的 `FRegisterName::NAME_NoneImplementation`（`FRegisterName.cpp:49-59`）——那里会 `FoundClass->NewObject()` 创建一个新的 `FName` 托管对象、`new FName(NAME_None)` 分配一个原生 `FName`、并 `AddStringReference` 注册一次。而 `FName.NAME_None` 在 `Script/UE/CoreUObject/Unreal.cs:23, 33` 被使用，**每次构造一个带默认 Name 的对象都会再创建一份**。这既是泄漏（每个 `FName` 托管对象都要等终结器）也是纯浪费——`NAME_None` 本应是一个缓存好的常量。

**建议**

按 `TMethodHelper.inl:92` 的既有正确模式，在 C# 侧取出值后释放句柄：

```csharp
// FStringImplementation.cs
public static unsafe string FString_ToStringImplementation(nint InString)
{
    var Handle = __FString_ToStringImplementation(InString);

    if (Handle == 0)
    {
        return null;
    }

    var Result = (string)HandleData.GetObject(Handle);

    HandleData.Free(Handle);      // ← 补上；C++ 侧已不再引用该句柄

    return Result;
}
```

**必须对所有"C++ 新建并直接返回"的句柄调用 `Free`**，同文件内受影响的方法（本报告已核实存在该模式）：

| 位置 | 方法 |
|---|---|
| `FStringImplementation.cs:37-42` | `FString_ToStringImplementation` |
| `FNameImplementation.cs:37-42` | `FName_ToStringImplementation` |
| `FNameImplementation.cs:46-51` | `FName_NAME_NoneImplementation` |
| `FTextImplementation.cs:46-51` | `FText_ToStringImplementation` |
| `FAnsiStringImplementation.cs:37-42` | `FAnsiString_ToStringImplementation` |
| `FUtf8StringImplementation.cs:37-42` | `FUtf8String_ToStringImplementation` |
| `TOptionalImplementation.cs:55-60` | `TOptional_GetImplementation` |
| `TArrayImplementation.cs:215-224` | `TArray_GetCompoundImplementation<T>` |
| `TMapImplementation.cs:121-174` | `TMap_*CompoundImplementation`（5 个） |
| `TSetImplementation.cs:94` | `TSet_GetEnumeratorCompoundImplementation<T>` |
| `TLazyObjectPtrImplementation.cs:38` | `TLazyObjectPtr_GetImplementation<T>` |
| `TSubclassOfImplementation.cs:38` | `TSubclassOf_GetImplementation` |

**注意这一修法有一个前提必须确认**：这些句柄是否是"同一对象已存在的句柄"（`Alloc` 的 `Count++` 分支）。对 `ToString` 而言每次都是**新建的 `string`**，所以 `Free` 是安全的（Count 1→0）。但对 `TArray_GetCompoundImplementation` 这类返回**已注册 UObject/容器**的方法，`Alloc` 很可能只是把已有的 `Count` 从 N 加到 N+1，此时补一个 `Free` 是**正确的配对**（把多加的那 1 减回去），而不是过早释放。**这正是引用计数设计成这样的原因**——所以补 `Free` 是安全的，但必须在每一处都补，否则计数会单向漂移。

**验证方式**

- 反证：`grep -n "HandleData\.Free" Script/**/*.cs` → 只有 `HandleData.cs:25` 的定义，零调用点。
- 用例（可在编辑器中直接跑）：
  ```csharp
  for (var i = 0; i < 100000; i++) { var S = new FString("x").ToString(); }
  ```
  观察 `HandleData.Handles.Count`（需临时加一个 internal 计数属性）是否增长 100000；同时观察进程内存。
- 更贴近实战的用例：在 `Actor.Tick` 里写 `Debug.Log($"{SomeFName}")`，跑 10 分钟后看 `GCHandle` 数量（`dotnet-counters` / `!DumpHeap` 的 `System.Runtime.InteropServices.GCHandle` 统计）。
- 加断言：在 `HandleData` 里加一个 `Handles.Count` 的 `Debug.Assert` 上限，用于回归测试。

---

#### [F-INT2-013] 4 个字符串 `ToString` 导出对 `GetString` 结果不判空，仅 `FAnsiString` 判了（同族内不一致）

- **类别**: Bug（空指针解引用）
- **严重度**: **P1**（崩溃）
- **复核结论**: 确认
- **可达性**: 活跃
- **复核证据**: 4 处 `GetString<T>()` 后直接解引用（`FRegisterString.cpp:46` `**String`、`FRegisterName.cpp:46` `Name->ToString()`、`FRegisterText.cpp:72` `Text->ToString()`、`FRegisterUtf8String.cpp:50` `*FUtf8String(*Utf8String)`），与同族**唯一**做了判空的 `FRegisterAnsiString.cpp:47-54`（`:51-53` 三元 + `InvalidManagedHandle`）构成直接对照 —— 5 个文件逐一读毕，4:1 成立。**关键前提已回源码确证**：`FCSharpEnvironment::GetString` 对未命中句柄**返回 `nullptr`**（`Public/Registry/FStringRegistry.inl:20-28`，`:25-27` 的 `FoundValue != nullptr ? ... : nullptr`），因此空指针解引用**确实会发生**而不是理论推演。可达路径同原文三点（终结器先 `UnRegister` 再被别人 `ToString`；`HandleData.GetHandle` 返回 0；容器重分配后旧 `FString`）。C# 侧 `FString.cs:42-43` 的 `ToString()` 是公开且被隐式调用的方法（字符串插值 / `string.Format` / `+`）。
- **级别变动**: 无（P1 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterString.cpp:42-47`、`FRegisterName.cpp:42-47`、`FRegisterText.cpp:68-73`、`FRegisterUtf8String.cpp:46-51`；**正确写法**在 `FRegisterAnsiString.cpp:47-54`
- **函数**: `FRegisterString/Name/Text/Utf8String::ToStringImplementation(IManagedHandle)`
- **置信度**: 高（同族 5 个文件中 4 个缺检查、1 个有检查，构成直接对照）

**现状（代码事实）**

```cpp
// FRegisterString.cpp:42-47（原样）—— 无判空
static IManagedHandle ToStringImplementation(const IManagedHandle InManagedHandle)
{
    const auto String = FCSharpEnvironment::GetEnvironment().GetString<FString>(InManagedHandle);

    return IScriptDomain::Get()->NewString(TCHAR_TO_UTF8(**String));
}
```

```cpp
// FRegisterName.cpp:42-47（原样）—— 无判空
static IManagedHandle ToStringImplementation(const IManagedHandle InManagedHandle)
{
    const auto Name = FCSharpEnvironment::GetEnvironment().GetString<FName>(InManagedHandle);

    return IScriptDomain::Get()->NewString(TCHAR_TO_UTF8(*Name->ToString()));
}
```

```cpp
// FRegisterText.cpp:68-73（原样）—— 无判空
static IManagedHandle ToStringImplementation(const IManagedHandle InManagedHandle)
{
    const auto Text = FCSharpEnvironment::GetEnvironment().GetString<FText>(InManagedHandle);

    return IScriptDomain::Get()->NewString(TCHAR_TO_UTF8(*Text->ToString()));
}
```

```cpp
// FRegisterUtf8String.cpp:46-51（原样）—— 无判空
static IManagedHandle ToStringImplementation(const IManagedHandle InManagedHandle)
{
    const auto Utf8String = FCSharpEnvironment::GetEnvironment().GetString<FUtf8String>(InManagedHandle);

    return IScriptDomain::Get()->NewString(TCHAR_TO_UTF8(*FUtf8String(*Utf8String)));
}
```

**对照——同族的 `FAnsiString` 版本做对了：**

```cpp
// FRegisterAnsiString.cpp:47-54（原样）
static IManagedHandle ToStringImplementation(const IManagedHandle InManagedHandle)
{
    const auto AnsiString = FCSharpEnvironment::GetEnvironment().GetString<FAnsiString>(InManagedHandle);

    return AnsiString != nullptr
               ? IScriptDomain::Get()->NewString(TCHAR_TO_UTF8(*FAnsiString(*AnsiString)))
               : InvalidManagedHandle;
}
```

**调用上下文**

C# 的 `ToString()` 是公开 API 且被隐式调用：

```csharp
// Script/UE/CoreUObject/FString.cs:42-43（原样）
public override string ToString() =>
    FStringImplementation.FString_ToStringImplementation(HandleData.GetHandle(this));
```

```csharp
// Script/UE/CoreUObject/FString.cs:12（原样）—— 终结器会注销
~FString() => FStringImplementation.FString_UnRegisterImplementation(HandleData.GetHandle(this));
```

注意 `HandleData.GetHandle(this)`（`HandleData.cs:80-93`）在对象**没有任何句柄时返回 0**。因此 `new FString("")` 之后若句柄已被回收，或对一个刚构造但未注册的 `FString` 调 `ToString()`，传入的就是 `IManagedHandle{}`（全零）。

**问题**

`FCSharpEnvironment::GetString<FString>` 对未知句柄返回 `nullptr`，随后：

- `FRegisterString.cpp:46`：`**String` → 对 `nullptr` 做 `operator*` → **空指针解引用，立即崩溃**；
- `FRegisterName.cpp:46`：`Name->ToString()` → 空指针调用成员函数；
- `FRegisterText.cpp:72`：`Text->ToString()` → 同上；
- `FRegisterUtf8String.cpp:50`：`*Utf8String` → 同上。

触发路径（都是在 C# 里无需刻意构造的写法）：

1. `FString` 的**终结器线程**已经调用了 `UnRegister`（`FString.cs:12`）→ 原生 `FString*` 被 `RemoveStringReference` 释放 → 此时另一个仍持有同 `nint` 的代码调用 `ToString()`；
2. `HandleData.GetHandle` 返回 0（对象尚未分配句柄）时直接 `ToString()`；
3. 容器/属性返回的 `FString` 在**原生对象已随容器重分配而失效**后调用 `ToString()`。

`FAnsiString` 的写法证明作者知道这个风险——**做成 5 个中的 1 个说明这是复制粘贴时漏掉的分支**。

**建议**

统一为 `FAnsiString` 的写法（并把 `InvalidManagedHandle` 的返回约定在 C# 侧映射为 `null`，`FStringImplementation.cs:41` 已有 `Handle != 0 ? ... : null` 的处理，✅ 已兼容）：

```cpp
static IManagedHandle ToStringImplementation(const IManagedHandle InManagedHandle)
{
    const auto String = FCSharpEnvironment::GetEnvironment().GetString<FString>(InManagedHandle);

    return String != nullptr
               ? IScriptDomain::Get()->NewString(TCHAR_TO_UTF8(**String))
               : InvalidManagedHandle;
}
```

同类问题在 §6 的 F-INT2-003 中已就 `FRegisterLazyObjectPtr` / `FRegisterSubclassOf` 报告——**这是同一个 bug 模式在 7 个文件中的重复出现**，建议一次性全目录排查：

```
grep -n "GetString<\|GetMulti<\|GetContainer<\|GetOptional(" Source/UnrealCSharp/Private/Domain/Interop/*.cpp
```
对每个"取到之后直接 `->` 或 `*`"的位置检查是否判空。

**验证方式**

- 在 4 个 `ToStringImplementation` 首行后加 `check(String)`，跑现有脚本测试即可看到是否命中。
- 用例：`var S = new FString(); GC.KeepAlive(...); S.ToString();` 或直接 `HandleData.GetObject(0)` 之类的路径。

---

#### [F-INT2-002] `IManagedHandle`（`int64` 按值）与 C# `nint` **在 32 位目标上会 ABI 不匹配**，但本插件**无法构建 32 位目标** —— 原位问题已降级为「缺 64 位 `static_assert`」（P3）

- **类别**: 平台兼容 / 未定义行为
- **严重度**: **P3**（缺编译期守卫；**原判 P1「32 位 ABI 不匹配」已被证伪**，见下）
- **复核结论**: 部分确认（偏差：**触发条件不成立** —— 本插件不支持也无法构建 32 位目标，故"32 位下崩溃"这一后果在当前及可预见的配置下**不可达**；仅保留"缺少 `static_assert` 防御"这一点）
- **可达性**: 不可达（32 位路径）
- **复核证据**: ① 类型事实成立 —— `IManagedHandle.h:5-7` `struct IManagedHandle { int64 Value{}; }`（**40 行文件已全读**），C# 侧 `nint`（`TArrayImplementation.cs:72` 等）；② **32 位前提不成立**：`UnrealCSharp.uplugin` **无 `PlatformAllowList`/`SupportedTargetPlatforms`**（61 行全读，仅 7 个 Module + EnhancedInput 依赖），Android 在 UE 5.6 只允许 **arm64/x64**（`Engine/Source/Programs/UnrealBuildTool/Platform/Android/UEBuildAndroid.cs:143-150` 只 Add "arm64"/"x64"，`:153-163` 无架构即 `BuildException`；该文件中 `armv7`/`armeabi` **命中 0**）；③ **插件内无任何 32 位分支**：`grep "PLATFORM_32BITS|sizeof\(void\*\) == 4|Win32|armv7|armeabi" Source/` 在**非 ThirdParty** 代码中命中 **0**；④ 第三方库只有 64 位：`ThirdParty/LeanCLR/lib/Release/{Win64,Linux_x86_64,macOS_arm64,macOS_x86_64,Android,IOS}`（无 x86/armv7 目录），且 `LeanCLR.Build.cs:81-89` 对 Android 只取单一 `Android/libleanclr.a`，**没有按架构分目录** → 32 位目标会在链接期直接失败，不会"编过但运行期崩溃"。**结论：原文"32 位 ABI 不匹配"是真实存在的类型事实，但对本插件不构成可触发缺陷**；保留的唯一价值是"补 `static_assert` 让未来不支持平台早失败"。
- **级别变动**: **P1 → P3**（理由：后果不可达，仅剩防御性改进价值；原文 §4.3 与 §8.2 的相应表述已同步更正）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterArray.cpp:14, 21, 34, 43, ...`（本目录全部导出）；契约定义在 `Source/UnrealCSharpCore/Public/Domain/Script/IManagedHandle.h:5-7`
- **函数**: 本目录**所有**导出的第一个参数
- **置信度**: **高**（原文"仅'插件是否真在 32 位目标上构建'未验证"这一保留已在第 ②③④ 条证据下闭合）

**现状（代码事实）**

```cpp
// Source/UnrealCSharpCore/Public/Domain/Script/IManagedHandle.h:5-7（原样）
struct IManagedHandle
{
    int64 Value{};
```

```cpp
// FRegisterArray.cpp:76-85（原样，任取一例）
static int32 NumImplementation(const IManagedHandle InManagedHandle)
{
    if (const auto ArrayHelper = FCSharpEnvironment::GetEnvironment().GetContainer<FArrayHelper>(
        InManagedHandle))
    {
        return ArrayHelper->Num();
    }

    return 0;
}
```

```csharp
// Script/UE/Library/TArrayImplementation.cs:51-56（原样，对应上面）
private static unsafe partial int __TArray_NumImplementation(nint InArray);

public static int TArray_NumImplementation(nint InArray)
{
    return __TArray_NumImplementation(InArray);
}
```

生成器不做任何尺寸适配，直接照抄托管类型进函数指针签名：

```csharp
// UnrealTypeSourceGenerator.cs:900-901, 916-918（原样）
var pointerType = string.Join(", ", method.Parameters
    .Select(Parameter => Qualify(Parameter.Type, qualifiedTypes)).Concat(new[] { returnType }));
...
$"\t\t{accessibility} static unsafe partial {returnType} {method.Name}({parameters}) =>\n" +
$"\t\t\t((delegate* unmanaged[Cdecl]<{pointerType}>)global::Interop.MethodBridge.GetMethod(\n"
```

**调用上下文**

本目录 18 个文件的**每一个导出函数**，以及 `SubclassOf`/`LazyObjectPtr` 的 `Register` 这类传 3 个句柄的函数。全插件层面 `IManagedHandle` 出现在 164 处（`grep -c "IManagedHandle"` on `Source/UnrealCSharpCore/Public/Domain/**/*.h`）。

**问题**

- 64 位：`int64` 与 `nint` 都是 8 字节，恰好一致 → **当前在 Win64 上一切正常，问题被掩盖**。
- ~~32 位（`Win32`、`Android armeabi-v7a`、旧 `iOS armv7`）：`nint` = 4 字节，而 C++ 按 8 字节的聚合体传值。~~ **（已证伪其前提：UE 5.6 的 Android 只允许 arm64/x64，插件无 32 位分支与 32 位第三方库 —— 以下推演仅作为"若将来支持 32 位会怎样"的说明保留）** 在 `__cdecl` 下这将导致**参数读取错位**：C++ 读到低 4 字节 + 把下一个参数的 4 字节当作高 4 字节拼成一个错误句柄 → `GetContainer<FArrayHelper>` 查表失败（返回 0 / 静默无操作）或命中错误条目。
- 更危险的是**多句柄 + 混合标量的函数**（如 `RemoveAt(IManagedHandle, int32, int32, uint8)`，`FRegisterArray.cpp:195`）：在 x86 `__cdecl` 上参数全部走栈，错位只会读到垃圾；但在**寄存器传参的 32 位 ARM 调用约定（AAPCS）**上，8 字节聚合体需要 2 个寄存器对齐到偶数寄存器号，错位会直接把后续所有参数映射到错误的寄存器 → **崩溃而非静默错误**。

**建议**

三选一，按侵入度排序：

1. **最小改动（推荐）**：把句柄在 ABI 边界上改为指针宽度类型，保留结构体用于内部。
   ```cpp
   // IManagedHandle.h
   struct IManagedHandle { int64 Value{}; };
   // 导出层参数改为：using IManagedHandleABI = UPTRINT;  // 或 void*
   ```
   并在 `FRegisterXxx` 里做一次 `IManagedHandleFromObject` / `IManagedHandleToObject` 转换——代价是每个导出多一行。
2. 让生成器在 32 位平台产出 `long` 而不是 `nint`（`UnrealTypeSourceGenerator.cs:929-939` 的 `Qualify` 里加平台分支）——但这需要 C# 编译期知道目标位数，且 `[DllImport]` 分支仍会不一致。
3. 在 `IManagedHandle.h` 加编译期守卫，**把不支持的平台显式切断**，避免出现"能编过但运行期崩溃"：
   ```cpp
   static_assert(sizeof(void*) == 8, "UnrealCSharp interop currently requires a 64-bit target");
   ```
   这是本报告推荐的**最低成本止血**：要么修好，要么明确不支持。

**验证方式**

- `grep -rn "sizeof(void\*) == 4\|PLATFORM_32BITS" Source/` 看是否已有 32 位处理。
- 若项目有 Android/Win32 目标：构建后在 32 位设备上跑任意容器用例，观察 `Num()` 是否返回 0（查表失败）或直接崩溃。
- 加 `static_assert(sizeof(IManagedHandle) == sizeof(void*))` 可在编译期立刻暴露。

---

#### [F-INT2-003] 全部 5 个智能指针族的 `Get` / `LoadSynchronous` 导出对 `GetMulti` 结果不判空（**本报告范围内 7 处；全目录共 8 处**）

- **类别**: Bug（空指针解引用）
- **严重度**: **P1**（崩溃）
- **复核结论**: 部分确认（偏差：**处数**。原文标题写"共 8 处"但文件清单只列了 7 项 —— 第 8 处在 `FRegisterScriptInterface.cpp:67`，**不属于本报告的 18 文件范围**。机制、类别、级别全部成立）
- **可达性**: 活跃
- **复核证据**: 全目录 `grep "Multi->"` 命中 **恰为 8**，逐条为 `FRegisterLazyObjectPtr.cpp:52`、`FRegisterSoftClassPtr.cpp:52`、`FRegisterSoftClassPtr.cpp:60`、`FRegisterSoftObjectPtr.cpp:52`、`FRegisterSoftObjectPtr.cpp:60`、`FRegisterSubclassOf.cpp:51`、`FRegisterWeakObjectPtr.cpp:52`（**以上 7 处在范围内**）+ `FRegisterScriptInterface.cpp:67`（`Multi->GetObject()`，**范围外，同一 bug 模式**）。**空指针前提已确证**：`GetMulti` 未命中即返回 `nullptr`（`Public/Registry/FMultiRegistry.inl:19-27`，`:24-26`）。**"`Bind(nullptr)` 是否安全"这一原存疑项已闭合**：`FCSharpBind.inl:56-61` 首行即 `if (InObject == nullptr) return InvalidManagedHandle;` → **安全**，所以本条的危害**只**在 `Multi == nullptr`（原有 null 检查缺失），而不在 `Multi->Get()` 返回空。
- **级别变动**: 无（P1 维持）
- **文件**（范围内 7 处，全部已逐一读取确认；另 1 处越界见上）：
  - `Source/UnrealCSharp/Private/Domain/Interop/FRegisterLazyObjectPtr.cpp:47-53`（`Get`）
  - `Source/UnrealCSharp/Private/Domain/Interop/FRegisterWeakObjectPtr.cpp:47-53`（`Get`）
  - `Source/UnrealCSharp/Private/Domain/Interop/FRegisterSoftClassPtr.cpp:47-53`（`Get`）、`:55-61`（`LoadSynchronous`）
  - `Source/UnrealCSharp/Private/Domain/Interop/FRegisterSoftObjectPtr.cpp:47-53`（`Get`）、`:55-61`（`LoadSynchronous`）
  - `Source/UnrealCSharp/Private/Domain/Interop/FRegisterSubclassOf.cpp:46-52`（`Get`）
- **函数**: `FRegister{LazyObjectPtr,WeakObjectPtr,SoftClassPtr,SoftObjectPtr,SubclassOf}::{Get,LoadSynchronous}Implementation(IManagedHandle)`
- **置信度**: 高（同文件内 `IdenticalImplementation` 的判空写法构成直接对照证据；5 个文件的写法完全一致）

**现状（代码事实）**

```cpp
// FRegisterLazyObjectPtr.cpp:47-53（原样）
static IManagedHandle GetImplementation(const IManagedHandle InManagedHandle)
{
    const auto Multi = FCSharpEnvironment::GetEnvironment().GetMulti<TLazyObjectPtr<UObject>>(
        InManagedHandle);

    return FCSharpEnvironment::GetEnvironment().Bind(Multi->Get());   // ← Multi 未判空
}
```

```cpp
// FRegisterSubclassOf.cpp:46-52（原样）
static IManagedHandle GetImplementation(const IManagedHandle InManagedHandle)
{
    const auto Multi = FCSharpEnvironment::GetEnvironment().GetMulti<TSubclassOf<UObject>>(
        InManagedHandle);

    return FCSharpEnvironment::GetEnvironment().Bind(Multi->Get());   // ← Multi 未判空
}
```

**同样 8 处写法完全相同**——`FRegisterSoftObjectPtr.cpp:55-61` 是最具代表性的一个（`LoadSynchronous` 会**真正触发资源加载**，失败概率更高）：

```cpp
// FRegisterSoftObjectPtr.cpp:55-61（原样）
static IManagedHandle LoadSynchronousImplementation(const IManagedHandle InManagedHandle)
{
    const auto Multi = FCSharpEnvironment::GetEnvironment().
        GetMulti<TSoftObjectPtr<UObject>>(InManagedHandle);

    return FCSharpEnvironment::GetEnvironment().Bind(Multi->LoadSynchronous());
}
```

`FRegisterWeakObjectPtr.cpp:47-53` 与 `FRegisterSoftClassPtr.cpp:47-53` 的 `Get` 与上述逐字一致（仅模板参数不同）。

**同一文件内的对照**（作者自己知道要判空，只是没在 `Get` 上做）：

```cpp
// FRegisterLazyObjectPtr.cpp:25-36（原样）
static uint8 IdenticalImplementation(const IManagedHandle InA, const IManagedHandle InB)
{
    if (const auto FoundA = FCSharpEnvironment::GetEnvironment().GetMulti<TLazyObjectPtr<UObject>>(InA))
    {
        if (const auto FoundB = FCSharpEnvironment::GetEnvironment().GetMulti<TLazyObjectPtr<UObject>>(InB))
        {
            return *FoundA == *FoundB ? 1 : 0;
        }
    }

    return 0;
}
```

**调用上下文**

`UnRegister` 走**异步**路径：

```cpp
// FRegisterLazyObjectPtr.cpp:38-45（原样）
static void UnRegisterImplementation(const IManagedHandle InManagedHandle)
{
    AsyncTask(ENamedThreads::GameThread, [InManagedHandle]
    {
        (void)FCSharpEnvironment::GetEnvironment().RemoveMultiReference<TLazyObjectPtr<UObject>>(
            InManagedHandle);
    });
}
```

**问题**

存在两条无需构造非法输入的触发路径：

1. **`UnRegister` 之后仍调用 `Get`**。`UnRegister` 只是把移除请求丢进 `AsyncTask` 队列（`FRegisterLazyObjectPtr.cpp:40`）。从 C# 调用 `UnRegister` 到 lambda 真正执行之间存在**时间窗口**；即使 lambda 已执行，C# 侧仍持有一个 `nint` 句柄，任何后续 `Get` 都会让 `GetMulti` 返回 `nullptr` → `Multi->Get()` 解引用空指针。C# 侧没有任何"已注销"状态位可以阻止这一点（详见 §7 死代码/生命周期分析中对 `*Implementation.cs` 的检查）。
2. **重复 `UnRegister` / 句柄复用**：容器句柄表被 `Remove` 后，同一地址值可能被后续 `new` 出来的对象复用，`GetMulti` 的键比较若按值匹配（`IManagedHandle` 是 `int64`）会命中新对象——这是**类型混淆**而非空指针，更隐蔽。

另外 `Bind(Multi->Get())` 中 `Multi->Get()` 可能返回 `nullptr`（弱/懒引用在对象已 GC 或未加载时的正常状态）。`IdenticalImplementation` 里 `*FoundA == *FoundB` 对此是安全的，但 `Bind(nullptr)` 的行为未在本目录内验证 → 见 §8 存疑项。

**建议**

```cpp
static IManagedHandle GetImplementation(const IManagedHandle InManagedHandle)
{
    const auto Multi = FCSharpEnvironment::GetEnvironment().GetMulti<TLazyObjectPtr<UObject>>(
        InManagedHandle);
    if (Multi == nullptr)          // ← 补上
    {
        return IManagedHandle{};
    }

    const auto Object = Multi->Get();
    return Object != nullptr ? FCSharpEnvironment::GetEnvironment().Bind(Object) : IManagedHandle{};
}
```

C# 侧公开层应把 `IManagedHandle{}` / `nint` 0 映射为 `null`（`TOptionalImplementation.cs:59` 已有此模式：`Handle != 0 ? HandleData.GetObject(Handle) : null`，可直接照搬）。

**验证方式**

- C#：`var P = new TLazyObjectPtr<AActor>(null); P.Dispose(); /* 或不 Dispose 直接 */ P.Get();`
- 在 `GetImplementation` 里加 `check(Multi)` 后运行现有测试，看是否命中。
- grep 全目录同类模式：`grep -n "Multi->\|Helper->" FRegister*.cpp` 找出所有"取返回值后立即解引用"的点，逐一确认是否判空。

---

#### [F-INT2-004] `FArrayHelper::Swap` 与 `SwapMemory` 实现完全相同 —— **但"`Swap` 语义错误"已被证伪**（UE 的 `TArray::Swap` 也只是字节交换）；剩余问题仅"缺 `check` + 两个 API 冗余"

- **类别**: 可优化/一致性（**原文的"Bug / 未定义行为"与"`Swap` 语义错误"已被引擎对照证伪**，见下）
- **严重度**: **P3**（不再是"对非平凡类型结果错误"；只剩"未实现 UE `TArray::Swap` 的 sanity check 契约 + 两个公开 API 完全冗余"）
- **复核结论**: 部分确认（**成立的部分**：两函数体一字不差，已逐字比对；**被证伪的部分**：原文"缺少移动构造语义 → 对非平凡类型语义上是错的"——UE 自己的 `TArray::Swap` **同样是字节交换**）
- **可达性**: 活跃
- **复核证据**: `FArrayHelper.cpp:324-332` 两函数体完全相同（已确证）。**决定性引擎对照**：UE 5.6 的 `TArray::Swap`（`Runtime/Core/Public/Containers/Array.h:3307-3315`）实现为
  ```cpp
  check((FirstIndexToSwap >= 0) && (SecondIndexToSwap >= 0));
  check((ArrayNum > FirstIndexToSwap) && (ArrayNum > SecondIndexToSwap));
  if (FirstIndexToSwap != SecondIndexToSwap) { SwapMemory(FirstIndexToSwap, SecondIndexToSwap); }
  ```
  而 `SwapMemory`（`Array.h:3291-3297`）就是 `::Swap(元素地址A, 元素地址B)`。**即 UE 原生 `Swap` 与 `SwapMemory` 的唯一区别就是那两条 `check` 与 `i != j` 短路**，两者都不做移动构造。因此"`Swap` 悄悄退化成未定义行为级别的字节交换"这一论断与引擎语义不符 —— 字节交换**就是** UE 对 `Swap` 的定义。
- **级别变动**: 无（P3 维持，但**降级理由已从"语义错误"改为"契约不完整 + API 冗余"**）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterArray.cpp:298-306`（导出）、`:288-296`（对照）；实现 `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:324-332`
- **函数**: `FArrayHelper::Swap(int32, int32)`
- **置信度**: 高（两函数体逐字节相同，且已读到 UE 原生 `Swap` 的实现作为对照）

**现状（代码事实）**

```cpp
// FArrayHelper.cpp:324-332（原样）—— 注意两函数体一字不差
void FArrayHelper::SwapMemory(const int32 InFirstIndexToSwap, const int32 InSecondIndexToSwap) const
{
    ScriptArray->SwapMemory(InFirstIndexToSwap, InSecondIndexToSwap, InnerPropertyDescriptor->GetSize());
}

void FArrayHelper::Swap(const int32 InFirstIndexToSwap, const int32 InSecondIndexToSwap) const
{
    ScriptArray->SwapMemory(InFirstIndexToSwap, InSecondIndexToSwap, InnerPropertyDescriptor->GetSize());
}
```

导出层把两者作为**两个不同的公开 API** 同时暴露：

```cpp
// FRegisterArray.cpp:342-343（原样）
.Function("SwapMemory", SwapMemoryImplementation)
.Function("Swap", SwapImplementation);
```

```csharp
// Script/UE/CoreUObject/TArray.cs:306-312（原样）—— C# 侧也确实是两个独立 API
public void SwapMemory(int InFirstIndexToSwap, int InSecondIndexToSwap) =>
    TArrayImplementation.TArray_SwapMemoryImplementation(HandleData.GetHandle(this), InFirstIndexToSwap,
        InSecondIndexToSwap);

public void Swap(int InFirstIndexToSwap, int InSecondIndexToSwap) =>
    TArrayImplementation.TArray_SwapImplementation(HandleData.GetHandle(this), InFirstIndexToSwap,
        InSecondIndexToSwap);
```

**调用上下文**

`TArray.Swap(i, j)` 的行为契约（对应 UE 的 `TArray::Swap`）是"交换两个元素，对非平凡类型执行正确的移动构造/赋值"。`TArray.SwapMemory(i, j)` 的契约是"按原始字节交换"。

**问题**

> **⚠️ 更正：以下第 1 点已被引擎源码对照证伪，第 2 点成立但归 F-INT2-001。**
> UE 5.6 的 `TArray::Swap`（`Containers/Array.h:3307-3315`）**本身就是 `SwapMemory` + 两条 `check`**，从不做移动构造/赋值。所以"`Swap` 必须与 `SwapMemory` 分开"的推理不成立，"对非平凡类型语义错误"的结论也不成立。本条**剩余的真实偏差**只有两条：
> (i) 未复现 UE `Swap` 的两条 `check`（`FirstIndexToSwap >= 0`、`ArrayNum > FirstIndexToSwap`）与 `i != j` 短路 —— 而这些在 Shipping 下也会被编译掉（`AssertionMacros.h:325-328`），实际影响≈0；
> (ii) 向脚本暴露两个**完全等价**的公开 API（`TArray.Swap` / `TArray.SwapMemory`），属冗余与误导。

`ScriptArray->SwapMemory(...)` 展开为对两个元素块做 `Memswap`。对**非平凡类型**这至少有两个问题：

1. **缺少移动构造语义**：`FString` 内部是 `TArray<TCHAR>`（指针 + Num + Max）。裸字节互换在两者都"自持有"内存时通常侥幸正确，但对于带**自引用**或**外部指向内部**的类型（`FText` 的共享引用、自定义结构体中的自指针）会产生悬垂内部指针——**这正是 `Swap` 与 `SwapMemory` 必须分开的原因**。
2. **完全没有 index 校验**（与 F-INT2-001 同源），`Swap(0, 1<<30)` 直接越界 `Memswap`。

~~因此 `TArray<T>.Swap(i, j)` 在任何 `T != POD` 的场景下**语义上是错的**——它悄悄退化成了一次未定义行为级别的字节交换，而调用者以为在做安全交换。~~ **（已证伪：UE 的 `TArray::Swap` 就是字节交换，见上）**

**更正后的结论**：`FArrayHelper::Swap` 与 UE `TArray::Swap` 的差别**仅**是那两条 `check` 与短路（`Array.h:3307-3315` vs `FArrayHelper.cpp:329-332`）。建议在 `Swap` 里补上与引擎一致的前置条件，或直接把 `Swap` 实现委托给 `SwapMemory` 并**让两个公开 API 之一下线**（避免让脚本作者以为 `Swap` 更安全）。**注意：下面建议里的 `ScriptArrayHelper.SwapValues` 不是修法** —— UE 在 `UnrealType.h:4084-4092` 明确注释 `SwapValues` "**does not call constructors and destructors**"，其内部同样是 `Array->SwapMemory(A, B, ElementSize)`，与现状等价。

**建议**

```cpp
void FArrayHelper::Swap(const int32 InFirstIndexToSwap, const int32 InSecondIndexToSwap) const
{
    if (!IsValidIndex(InFirstIndexToSwap) || !IsValidIndex(InSecondIndexToSwap))
    {
        return;
    }

    auto ScriptArrayHelper = CreateHelperFormInnerProperty();

    ScriptArrayHelper.SwapValues(InFirstIndexToSwap, InSecondIndexToSwap);   // 走移动构造
}

void FArrayHelper::SwapMemory(const int32 InFirstIndexToSwap, const int32 InSecondIndexToSwap) const
{
    if (!IsValidIndex(InFirstIndexToSwap) || !IsValidIndex(InSecondIndexToSwap))
    {
        return;
    }

    ScriptArray->SwapMemory(InFirstIndexToSwap, InSecondIndexToSwap, InnerPropertyDescriptor->GetSize());
}
```

若 `SwapMemory` 被判定为"危险原语、不应向脚本暴露"，则应从 `FRegisterArray.cpp:342` 与 `TArray.cs:306-308` **同时删除**（这是更彻底的方案，但需要确认没有外部使用者——见 §7）。

**验证方式**

- `grep -n "SwapValues\|SwapMemory" Source/` 确认 `FScriptArrayHelper::SwapValues` 是否可用（UE 提供）。
- 用例：C# `var A = new TArray<FString>{"a","b"}; A.Swap(0,1);` 检查结果；再用 `TArray<FText>` 复现更明显。
- 代码层面：`diff <(sed -n '324,327p' FArrayHelper.cpp) <(sed -n '329,332p' FArrayHelper.cpp)` 应为空——即两函数体相同。

---

#### [F-INT2-010] 容器「值 / 句柄」双模式契约依赖 `typeof(T).IsValueType` 与 `IsPrimitiveProperty()` 隐式对齐 —— **已证伪：不一致类不存在，撤销（非缺陷）**

- **类别**: ~~未定义行为 / 平台兼容~~ → **撤销（非缺陷）**
- **严重度**: **撤销（非缺陷）**（原 P2）
- **复核结论**: **证伪**（前提不成立）—— "C# 侧存在 `IsValueType == true` 但 C++ descriptor 非 primitive 的类型"这一不一致类**在本项目的生成产物中不存在**，因此"写少了读垃圾 / 写多了栈溢出"两条后果**均不可达**
- **可达性**: 不可达
- **复核证据**: 三份独立证据链：
  1. **生成器把每个 UStruct 都生成成 C# `class`，不是 `struct`**：`Script/UE/Proxy/CoreUObject/UObject/Vector.cs:11-13` → `[PathName("/Script/CoreUObject.Vector")] public partial class FVector : IStaticStruct`。因此 `typeof(FVector).IsValueType == false` → C# 走**句柄路径**（`TArray.cs:88-90` → `TArrayImplementation.cs:215-224` → `stackalloc byte[sizeof(nint)]`），与 `FStructPropertyDescriptor::Get(void*, void*)` 写 8 字节 `IManagedHandle`（`FStructPropertyDescriptor.cpp:28-31`，`*reinterpret_cast<IManagedHandle*>(Dest) = NewRef(Src)`）**完全对齐** ✅。
  2. **整个生成代理树里没有任何 C# `struct`**：`grep "public (partial )?struct "` 在 `Script/UE/Proxy/**`（16512 个 `.cs`）命中 **0**；在两套 `Script/` 树中唯一命中是手写的 `Plugins/UnrealCSharp/Script/UE/CoreUObject/TaskInfo.cs:6`，与 UE 反射类型无关。也就是说"用户自定义 `struct` 作为 `TArray<T>` 元素"这条路**根本不存在** —— 它连 `TArray<T>` 的注册都过不了：`TArray.cs:15` → `FRegisterArray.cpp:16-18` 需要 `FReflectionRegistry::Get().GetClass(InManagedType)` 返回非空并取 `Class->GetGenericArgument()`，非 UE 类型会在这一步就失败。
  3. **C# 值类型集合与 C++ primitive 集合恰好相等**：C++ 侧 `IsPrimitiveProperty() == true` 的只有 `TPrimitivePropertyDescriptor<T>`（`TPropertyDescriptor.inl:33-36` 返回模板参数 `IsPrimitive`，`PrimitiveProperty/` 下 11 个 descriptor 全部继承它）**以及 `FEnumPropertyDescriptor`**（`FEnumPropertyDescriptor.h:5`：`class FEnumPropertyDescriptor final : public TPrimitivePropertyDescriptor<FEnumProperty>` → 同为 true）；而 C# 侧 `IsValueType == true` 的正好是"托管基元 + 生成出来的 `enum`"。`FEnumPropertyDescriptor::Get(void*, void*)` 写的是 `GetUnderlyingProperty()->CopySingleValue(Dest, Src)`（`FEnumPropertyDescriptor.cpp:14-17`）**内联写**，与 C# `enum` 的 `sizeof(T)` 内联路径同样对齐 ✅。
  4. 反向核对 `FRegisterOptional.cpp:62,131` 的 `IsPrimitiveProperty()` 分流也没问题：`T = int` → `FIntPropertyDescriptor`（primitive）→ `FDomain::Object_Unbox(InValue)` ✅；`T = FVector`（C# `class`）→ `FStructPropertyDescriptor`（非 primitive）→ `&ManagedHandle` ✅；`T = EMyEnum`（C# `enum`）→ `FEnumPropertyDescriptor`（**primitive**）→ `Object_Unbox` ✅（三个组合全部自洽）。
- **级别变动**: **P2 → 撤销（非缺陷）**（理由：不一致类不存在，契约由**代码生成器**这一唯一生产者保证；原文 §4.3、§6 汇总表、§8.1/§8.4 中"唯一可能升级为 P0 的未闭合项"的表述已同步更正）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterArray.cpp:109-119`、`FRegisterMap.cpp:96-105, 118-127`、`FRegisterSet.cpp:105-114`、`FRegisterOptional.cpp:127-142`
- **函数**: 所有 `RETURN_BUFFER_SIGNATURE` / `IN_VALUE_BUFFER_SIGNATURE` 导出
- **置信度**: **高**（原文为"中：具体某个 `FPropertyDescriptor` 对某个类型选择哪种写法的全量映射未逐一验证"——已用生成产物 + descriptor 继承链 + 枚举例外三点闭合；**残留**：未逐一枚举 `PrimitiveProperty/` 下 11 个 descriptor 的完整清单，只核对了继承关系与枚举特例）

**现状（代码事实）**

C# 侧按托管类型分流，**并据此决定缓冲区大小**：

```csharp
// TArray.cs:78-91（原样）
if (typeof(T).IsValueType)
{
    var ValueBuffer = stackalloc byte[sizeof(T)];
    TArrayImplementation.TArray_GetImplementation(HandleData.GetHandle(this), InIndex, ValueBuffer);
    return *(T*)ValueBuffer;
}
else
{
    return TArrayImplementation.TArray_GetCompoundImplementation<T>(
        HandleData.GetHandle(this), InIndex);
}
```

```csharp
// TArrayImplementation.cs:215-224（原样）—— 引用类型路径恒为 8 字节
public static T TArray_GetCompoundImplementation<T>(nint InArray, int InIndex)
{
    var ValueBuffer = stackalloc byte[sizeof(nint)];
    __TArray_GetImplementation(InArray, InIndex, ValueBuffer);
    var Handle = *(nint*)ValueBuffer;
    return Handle != 0 ? (T)HandleData.GetObject(Handle) : default;
}
```

C++ 侧则把缓冲区原样转交 descriptor，**不做任何尺寸断言**：

```cpp
// FRegisterArray.cpp:109-119（原样）
static void GetImplementation(const IManagedHandle InManagedHandle,
                              const int32 InIndex, RETURN_BUFFER_SIGNATURE)
{
    if (const auto ArrayHelper = FCSharpEnvironment::GetEnvironment().GetContainer<FArrayHelper>(
        InManagedHandle))
    {
        const auto Value = ArrayHelper->Get(InIndex);

        ArrayHelper->GetInnerPropertyDescriptor()->Get(Value, reinterpret_cast<void**>(RETURN_BUFFER));
    }
}
```

**问题**

这个契约有**两个独立的判定依据**，它们必须对所有类型给出相同答案，但代码中没有任何东西保证这一点：

| 侧 | 判定依据 | 位置 |
|---|---|---|
| C# | `typeof(T).IsValueType` | `TArray.cs:78/98/122/145/168/213/239/262/287` |
| C++ | `FPropertyDescriptor::IsPrimitiveProperty()` 与属性标志 | `FRegisterOptional.cpp:62,131`；`FArrayHelper.cpp:214` |

**~~已知的不一致类别是 `struct`~~（已证伪，见下）**：~~C# 里用户自定义的 `struct` 是 `IsValueType == true`（走 `sizeof(T)` 内联路径），而 C++ 侧一个 `FVector` 的 descriptor 若被判定为非 primitive，就会往缓冲区写一个 8 字节句柄——C# 只给了 `sizeof(FVector) == 24` 字节，**这次是"写少了"不会溢出，但 `*(FVector*)ValueBuffer` 读出来的是垃圾数据**（前 8 字节是句柄地址，后 16 字节是 `stackalloc` 未初始化内容），随后被当作 `FVector` 使用 → 静默错误坐标。~~
**更正**：`FVector` 在生成产物里是 **`class`**（`Script/UE/Proxy/CoreUObject/UObject/Vector.cs:12`），不是 `struct`；`Script/UE/Proxy/**` 中 `public struct` 命中 **0**。所以 C# 为 `FVector` 走的是**句柄路径**（8 字节缓冲 + `FStructPropertyDescriptor` 写 8 字节句柄），两侧一致 —— 上述"静默错误坐标"的场景**不存在**。

反向情形（C++ 写内联、C# 只给 8 字节）则直接**栈溢出写**：`RETURN_BUFFER` 指向 `stackalloc byte[sizeof(nint)]`，descriptor 却写入 16/24 字节。栈溢出写是**当前函数返回地址被覆盖**级别的问题。

~~无法在本目录内确定映射是否真的存在缺口——需要逐一核对 `FPropertyDescriptor::Factory` 对每个类型选择哪个子类。**这是本报告最高优先级的下钻验证项**（见 §8）。~~ **已闭合：缺口不存在**（证据见本条「复核证据」4 点）。**本条撤销（非缺陷）**；`FPropertyDescriptor::Factory`（`FPropertyDescriptor.cpp:47-134`）的类型分派清单也已读毕，`FEnumPropertyDescriptor` 归 primitive 是唯一容易误判的点，已单独核对。原文"建议"中的第 2 点（C# 侧统一 `stackalloc byte[64]`）仍可作为**廉价防御**保留，但已无已知触发路径。

**建议**

1. **协议自描述**：让 C++ 侧在写入前返回所需字节数，或让导出函数接收缓冲区容量并在不足时返回失败。
   ```cpp
   static int32 GetImplementation(const IManagedHandle InManagedHandle, const int32 InIndex,
                                  RETURN_BUFFER_SIGNATURE)   // 返回写入字节数，0 表示失败
   {
       ...
       const auto Descriptor = ArrayHelper->GetInnerPropertyDescriptor();
       // Descriptor->Get 内部应断言 sizeof(ReturnBuffer 容量) >= 所需
       Descriptor->Get(Value, reinterpret_cast<void**>(RETURN_BUFFER));
       return Descriptor->GetSize();   // 或 8（句柄）
   }
   ```
2. **C# 侧统一给足上限**：把 `stackalloc byte[sizeof(nint)]` 改为 `stackalloc byte[64]`（或 `Math.Max(sizeof(T), sizeof(nint))`），把"写少"从静默错误变成"最多浪费 56 字节栈"。这是**一行改动、零 API 破坏**的止血方案，建议立刻先做。
3. 在 `TArray_GetCompoundImplementation<T>` 里对 `ReturnsHandle` 做一次显式断言（可由生成器注入类型信息）。

**验证方式**

- 遍历 `Source/UnrealCSharp/Private/Reflection/Property/*Descriptor.cpp`，统计 `IsPrimitiveProperty() == false` 但对应 C# 类型是 `struct` 的组合。
- 用例：`var A = new TArray<FVector>(); A.Add(new FVector(1,2,3)); var V = A[0];` 检查 `V` 是否为 `(1,2,3)`。
- 在 `PropertyDescriptor::Get` 里加"写入字节数"计数与容量比对即可自动化发现。

---

### P2

> （分组标题为**原定级**。本组内含 1 条维持 P2（F-INT2-006）、1 条已**上调为 P1**（F-INT2-012）、2 条已下调为 P3（F-INT2-015 / F-INT2-005）—— 以各条 `**严重度**` 与上方速查表为准）

#### [F-INT2-015] 两个"纯头文件"中的注册实例是 `static` 而非 `inline` —— 每个包含它的编译单元各注册一次（`EForceInit` 重复 8 次、`EObjectFlags` 重复 2 次）

- **类别**: 冗余 / 一致性（追加语义已证实，但下游幂等，故不是 Bug）
- **严重度**: **P3**（确定的冗余：8×/2× 静态构造 + 枚举内容重算 + 重复写同一文件；**不产生功能错误**）
- **复核结论**: 部分确认（偏差：原文的两处保留意见均被**否证**——① 追加语义已确证；② 但"追加 → 升为 P1"的原文预案**不成立**，因为下游是幂等覆盖写。静态初始化顺序风险也**不成立**）
- **可达性**: 活跃
- **复核证据**:
  - 包含点数**已全量复跑**：`grep "FRegisterForceInit\.h"` → **8** 个 TU（`FRegisterBox2D.cpp:5`、`FRegisterLinearColor.cpp:5`、`FRegisterVector4.cpp:4`、`FRegisterRotator.cpp:4`、`FRegisterVector2D.cpp:4`、`FRegisterQuat.cpp:4`、`FRegisterPlane.cpp:4`、`FRegisterMatrix.cpp:4`），与原报告**逐行吻合**；`grep "FRegisterObjectFlags\.h"` → **2**（`FRegisterWorld.cpp:8`、`FRegisterUnreal.cpp:9`），同样吻合。
  - **注册是追加语义（原文存疑项 ② 闭合）**：`FBinding::Register(枚举版)`（`UnrealCSharpCore/Private/Binding/FBinding.cpp:67-76`）→ `EnumRegisters.Add_GetRef(new FBindingEnumRegister(...))`；随后 `FBinding::Register()`（`:25-32`）`for (const auto& Enum : EnumRegisters) Enums.Emplace(...)` → `TArray::Emplace` **无去重** ⇒ `Enums` 里**真有 8 份 `EForceInit`、2 份 `EObjectFlags`**。
  - **但后果被下游幂等吸收（原文预案 P1 不成立）**：`Enums` 的唯一消费者是 `ScriptCodeGenerator/Private/FBindingEnumGenerator.cpp:9` → `Generator(Enum)`（`:15-67`），它对每个条目 `FPaths::Combine(DirectoryName, ClassContent) + CSHARP_SUFFIX`（`:62`）算出的**文件名相同**，`SaveStringToFile` 反复覆盖同一路径、内容逐字节相同；而登记用的 `FGeneratorCore::AddGeneratorFile`（`FGeneratorCore.cpp:708-711`）写入的是 `TSet<FString> GeneratorFiles`（`FGeneratorCore.h:93`）→ **集合自动去重**。故净后果 = 8 倍重复计算 + 8 次重复文件写，无非功能错误。
  - **静态初始化顺序风险不成立（原文存疑项 ③ 闭合）**：`FBinding::Get()` 是 **Meyers 单例**（`FBinding.cpp:3-8`：函数内 `static FBinding Binding;`），不存在跨 TU 静态初始化顺序问题。
- **级别变动**: 无（P3 维持；**但"若追加语义则升 P1"的预案已作废**，理由见上）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterForceInit.h:18`、`Source/UnrealCSharp/Private/Domain/Interop/FRegisterObjectFlags.h:51`
- **函数**: `FRegisterForceInit::FRegisterForceInit()`、`FRegisterObjectFlags::FRegisterObjectFlags()`
- **置信度**: 高（包含点已用 grep 全量枚举；实例的链接性、注册表语义、下游消费路径三者均已读取）

**现状（代码事实）**

```cpp
// FRegisterForceInit.h:18（原样，文件最后一行）
[[maybe_unused]] static FRegisterForceInit RegisterForceInit;
```

```cpp
// FRegisterObjectFlags.h:51（原样，文件最后一行）
[[maybe_unused]] static FRegisterObjectFlags RegisterObjectFlags;
```

关键点：**`static` 在命名空间作用域表示"内部链接"，因此头文件里的这个变量在每个包含它的 `.cpp` 中都会各自实例化一份**，每份都执行一次构造函数。构造函数体就是注册动作：

```cpp
// FRegisterForceInit.h:8-16（原样）
struct FRegisterForceInit
{
    FRegisterForceInit()
    {
        TBindingEnumBuilder<EForceInit, false>()
            .Enumerator("ForceInit", EForceInit::ForceInit)
            .Enumerator("ForceInitToZero", EForceInit::ForceInitToZero);
    }
};
```

```cpp
// TBindingEnumBuilder.inl:12-22（原样）—— 构造即调用 FBinding::Get().Register(...)
explicit TBindingEnumBuilder() :
    EnumRegister(FBinding::Get().Register({[]() { return TName<T, T>::Get(); }},
                                          TName<std::underlying_type_t<T>, std::underlying_type_t<T>>::Get(),
                                          bIsProjectEnum
                                          ...
```

**包含点全量枚举（grep 证据）**

`FRegisterForceInit.h` —— **8 个 TU**：
`FRegisterVector4.cpp:4`、`FRegisterVector2D.cpp:4`、`FRegisterRotator.cpp:4`、`FRegisterQuat.cpp:4`、`FRegisterPlane.cpp:4`、`FRegisterMatrix.cpp:4`、`FRegisterLinearColor.cpp:5`、`FRegisterBox2D.cpp:5`
→ `EForceInit` 的构造与 2 个 Enumerator 注册**执行 8 次**。

`FRegisterObjectFlags.h` —— **2 个 TU**：
`FRegisterWorld.cpp:8`、`FRegisterUnreal.cpp:9`
→ `EObjectFlags` 的构造与 **27 个** Enumerator 注册**执行 2 次**（合计 54 次 `BindingEnumerator` 调用）。

**问题**

1. **约定的不一致**：本目录的 `.cpp` 文件全部把注册实例放在**匿名命名空间**内（`FRegisterArray.cpp:10 namespace { ... :348 [[maybe_unused]] FRegisterArray RegisterArray; }`），因此每个 `.cpp`（唯一一个 TU）只产生一份——**正确**。而这两个 `.h` 文件用的是裸 `static`，在头文件场景下语义完全不同。作者显然把 `.cpp` 的写法直接搬到了 `.h`。
2. **重复注册的实际后果取决于注册表语义**：
   - 若 `FBinding::Register` 是 `TMap::Add` 语义（覆盖）或 `FindOrAdd`，则是纯冗余开销（8× + 2× 的静态构造 + 字符串构造）；
   - 若是 `TArray::Emplace` 追加语义，**枚举会真的出现 8 份重复条目**，C# 侧枚举反射会拿到重复成员。
   本报告**未读取 `FBinding::Register` 的实现**，因此无法断言是哪种（见 §8 存疑项）。无论哪种，都应当修。
   **✅ 已读取 `FBinding::Register`（`FBinding.cpp:67-76`）→ 确认为 `TArray::Add_GetRef` 追加语义、无去重**；但下游 `FBindingEnumGenerator` 是幂等覆盖写 + `TSet` 去重，故仍是"应当修"的冗余而非功能错误（详见本条「复核证据」）。
3. **~~静态初始化顺序~~（已排除）**：~~这 8+2 个实例是**动态初始化**的非局部静态对象，其构造函数会在 `main` 之前运行。它们依赖 `FBinding::Get()` 已就绪。若 `FBinding::Get()` 返回的是全局对象而非函数内 `static`（Meyers 单例），则构成经典的"静态初始化顺序惨案"。~~ **`FBinding::Get()` 实际就是 Meyers 单例（`FBinding.cpp:3-8`：函数内 `static FBinding Binding;`），首次调用即完成构造，**不存在**静态初始化顺序问题。此风险项撤销。**

**建议**

改为 C++17 的 `inline` 变量（跨 TU 唯一），或更保守地放回 `.cpp`：

```cpp
// FRegisterForceInit.h
[[maybe_unused]] inline FRegisterForceInit RegisterForceInit;   // C++17：全程序唯一实例
```

若项目仍需支持 C++14，则把定义移到一个专属 `.cpp`：

```cpp
// FRegisterForceInit.h —— 只留声明
extern FRegisterForceInit RegisterForceInit;
// FRegisterForceInit.cpp —— 唯一实例
[[maybe_unused]] FRegisterForceInit RegisterForceInit;
```

**同时建议统一约定**：既然本目录既有的正确做法是"匿名命名空间的实例 + 在 `.cpp` 中定义"，那这两个头文件应当把实例定义下沉到某个 `.cpp`（例如新建 `Domain/Interop/FRegisterEnum.cpp`），头文件只留 `struct` 定义。

**验证方式**

- `grep -rn "FRegisterForceInit.h" Source/ | Measure-Object -Line` → 应为 8（已确认）。
- 在 `FRegisterForceInit` 构造函数里打一条 `UE_LOG`，观察启动日志中出现几次（当前预期 8 次）。
- 读 `FBinding::Register` 确认是覆盖还是追加，据此定级（追加语义 → 升为 P1）。

---

#### [F-INT2-012] `FAnsiString` / `FUtf8String` 的出口转换把**窄串硬转成 `const TCHAR*`** 再"转 UTF-8" —— **确证的编码错误 + 越界读**（另有无意义的拷贝构造）

- **类别**: Bug（编码错误 / 功能错误）+ 可优化/可读性（冗余拷贝）
- **严重度**: **P1**（功能错误：`FAnsiString` / `FUtf8String` 的出口 `ToString()` 返回**错误文本**，并伴随对源缓冲的**越界读**）
- **复核结论**: 部分确认 + **上调**（"编码语义"这一原文标注为"低置信度/未验证假设"的部分，**已被引擎源码确证为真实 bug**；"冗余拷贝"部分成立）
- **可达性**: 活跃
- **复核证据**（4 步链条，全部为引擎/插件实测源码）:
  1. `FAnsiString::operator*` 返回**窄字符指针**：`FAnsiString` 由 `Containers/AnsiString.h:8-13` 以 `UE_STRING_CLASS=FAnsiString / UE_STRING_CHARTYPE=ANSICHAR` 生成，`operator*` 定义在 `Containers/UnrealString.h.inl:369-372` → `const ElementType*` = **`const ANSICHAR*`**；`FUtf8String` 同理（`Utf8String.h:10-11` → **`const UTF8CHAR*`**）。
  2. `TCHAR_TO_UTF8` **不是重载、是宏**：`Containers/StringConv.h:1021` → `#define TCHAR_TO_UTF8(str) (ANSICHAR*)FTCHARToUTF8((const TCHAR*)str).Get()` —— 它把参数**硬转成 `const TCHAR*`**，而 `FTCHARToUTF8_Convert` 的 `FromType` 就是 `TCHAR`（`StringConv.h:226-233`）。
  3. 于是 `FRegisterAnsiString.cpp:52` 的 `TCHAR_TO_UTF8(*FAnsiString(*AnsiString))` = 把 **1 字节窄串重新解释成 2 字节 `TCHAR` 串**再做"TCHAR→UTF-8"转换 → 输出乱码；且转换需要先按 `TCHAR` 求长度，遇到窄串的 NUL 后仍会继续按 2 字节步进找 `0x0000` 终止符 → **越界读**。`FRegisterUtf8String.cpp:50` 完全同型。
  4. 本项目五个平台的 `TCHAR` 都是 **2 字节**（Windows `wchar_t`；Android/iOS/Linux/Mac 显式 `PLATFORM_TCHAR_IS_CHAR16 1`：`AndroidPlatform.h:47`、`IOSPlatform.h:29`、`UnixPlatform.h:50`、`MacPlatform.h:55`；`HAL/Platform.h:280-282` 的 `PLATFORM_TCHAR_IS_UTF8CHAR` 默认 `USE_UTF8_TCHARS`=0）→ **不是"仅某平台"的问题，全部目标平台都错**。
  - 注：入口方向（`UTF8_TO_TCHAR(InValue)`，`FRegisterAnsiString.cpp:18`）是正确的，因为 C# 传的确实是 UTF-8 字节 —— 错只在**出口**。
- **级别变动**: **P3 → P1**（理由：从"可疑的冗余拷贝"变成"已证实的编码错误 + 越界读"，且每次 `FAnsiString/FUtf8String.ToString()` 都触发）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterAnsiString.cpp:47-54`、`Source/UnrealCSharp/Private/Domain/Interop/FRegisterUtf8String.cpp:46-51`
- **函数**: `FRegisterAnsiString::ToStringImplementation`、`FRegisterUtf8String::ToStringImplementation`
- **置信度**: **高**（原文为"低"，其不确定性来自"未读引擎 `StringConv.h`"；现已读该宏与 `operator*` 的定义，假设已闭合）

**现状（代码事实）**

```cpp
// FRegisterAnsiString.cpp:47-54（原样）
static IManagedHandle ToStringImplementation(const IManagedHandle InManagedHandle)
{
    const auto AnsiString = FCSharpEnvironment::GetEnvironment().GetString<FAnsiString>(InManagedHandle);

    return AnsiString != nullptr
               ? IScriptDomain::Get()->NewString(TCHAR_TO_UTF8(*FAnsiString(*AnsiString)))
               : InvalidManagedHandle;
}
```

```cpp
// FRegisterUtf8String.cpp:46-51（原样）
static IManagedHandle ToStringImplementation(const IManagedHandle InManagedHandle)
{
    const auto Utf8String = FCSharpEnvironment::GetEnvironment().GetString<FUtf8String>(InManagedHandle);

    return IScriptDomain::Get()->NewString(TCHAR_TO_UTF8(*FUtf8String(*Utf8String)));
}
```

注意 `*FAnsiString(*AnsiString)`：`*AnsiString` 已是 `FAnsiString&`，`FAnsiString(...)` 再**拷贝构造一个临时对象**，然后 `*` 取字符指针。这个中间拷贝**没有任何作用**——直接写 `*AnsiString` 与 `**AnsiString` 语义相同（`FAnsiString` 既提供 `operator*` 又提供拷贝构造）。作者写成这样通常是在**规避某个类型不匹配的编译错误**，这本身就是信号。

相比之下 `FRegisterString.cpp:46` 用的是 `**String`（`FString*` → 解引用得 `FString&` → `operator*` 得 `const TCHAR*`），**没有多余的中间拷贝**——同族内的写法不一致。

**问题**

1. **冗余拷贝（高置信度）**：每次 `FAnsiString.ToString()` / `FUtf8String.ToString()` 都会构造并销毁一个临时字符串对象。对短字符串是栈上拷贝（`FAnsiString`/`FUtf8String` 是带小缓冲的实现），成本不大但完全没有意义。
2. **编码语义（✅ 由"低置信度"升为"确证"）**：`FAnsiString::operator*` 返回 `const ANSICHAR*`、`FUtf8String::operator*` 返回 `const UTF8CHAR*`（`UnrealString.h.inl:369` + `AnsiString.h:9` / `Utf8String.h:11`），而 `TCHAR_TO_UTF8` 是**宏**：`StringConv.h:1021` = `(ANSICHAR*)FTCHARToUTF8((const TCHAR*)str).Get()`，其中 `FTCHARToUTF8_Convert::FromType = TCHAR`（`StringConv.h:232`）。
   - 原文担心的"`StringCast` 是否对窄字符有恒等直通重载"**不适用**：这里根本没有重载解析，宏把窄指针**硬转**成 `const TCHAR*`（`TCHAR` = 2 字节，五平台一致）。
   - 后果：① 输出**乱码**（把 1 字节窄码元两两拼成 UTF-16 码元再转 UTF-8）；② 按 `TCHAR` 求长度时越过窄串的真实 NUL → **越界读**。`FAnsiString` 与 `FUtf8String` **两者都错**（不是"其中一支恰好正确"）。
   - ✅ 修法以本节「建议」第 2 条为准（直接透传 UTF-8 字节，不绕 `TCHAR`）。
3. **入口方向本身就有损**：C# 侧对 `FAnsiString` **也用 `Encoding.UTF8.GetBytes`**（`FAnsiStringImplementation.cs:13`），C++ 侧 `UTF8_TO_TCHAR` 得到 `FString`（UTF-16），再 `FAnsiString(FString)` 构造——**任何不能表示为 ANSI 的字符会在这一步被静默替换成 `?`**。因此 `new FAnsiString("中文").ToString()` 会丢失信息，而 C# 公开 API 是 `string`，调用者无从得知。

**建议**

1. 去掉无意义的中间拷贝：
   ```cpp
   return IScriptDomain::Get()->NewString(TCHAR_TO_UTF8(*AnsiString));
   ```
   （若 `FAnsiString::operator*` 的返回类型不便直接喂给 `TCHAR_TO_UTF8`，就显式用正确的转换宏，例如 `ANSI_TO_TCHAR` 再 `TCHAR_TO_UTF8`，让意图显式化——**关键在于不要让重载解析替你做编码决策**。）
2. 对 `FUtf8String` 直接用**UTF-8 直通**路径，不要绕 `TCHAR`：
   ```cpp
   // 语义明确：FUtf8String 内部就是 UTF-8 字节，直接交给托管侧
   return IScriptDomain::Get()->NewString(reinterpret_cast<const char*>(*Utf8String));
   ```
3. 在文档/注释里明确 `FAnsiString` 的 C# API 是**有损**的（只支持 ANSI 字符集），或改为在 C# 侧用 `Encoding.Default`/`Latin1` 编码以匹配类型语义。

**验证方式**

- 读引擎 `Runtime/Core/Public/Containers/StringConv.h` 与 `StringFwd.h`，确认 `StringCast<UTF8CHAR>` 是否有 `const UTF8CHAR*` / `const ANSICHAR*` 的专用构造以及各自语义。
- 用例（能直接判定）：
  ```csharp
  var S = new FUtf8String("中");      // U+4E2D → UTF-8: E4 B8 AD
  Console.WriteLine(S.ToString() == "中");   // 若为 false 则是二次转换 bug
  var A = new FAnsiString("中");
  Console.WriteLine(A.ToString());            // 预期 "??"（有损），确认这一行为是否符合设计
  ```
- 用非 ASCII + 非 UTF-8 合法序列的字节做边界测试。

---

#### [F-INT2-005] **56 处**「取容器 + 判空」样板逐字重复，可宏化为单一入口（原文"64 处 / 63 次"两处数字均不正确）

- **类别**: 可优化/可读性
- **严重度**: **P3**
- **复核结论**: 部分确认（偏差：**计数**。机制与结论成立；原文标题"64 处"与正文"63 次（28+16+11+8）"互相矛盾，且两者都不等于实测值）
- **可达性**: 活跃
- **复核证据**: 用 `grep` 工具全量重跑（非脚本）：
  - `grep "GetEnvironment\(\)\.GetContainer<"` 于 `Source/UnrealCSharp/Private/Domain/Interop/` → **50** 处：`FRegisterArray.cpp` **27**（`:23,25,45,56,67,78,89,100,112,124,133,144,155,166,178,188,198,207,216,226,235,246,257,268,279,291,301`）、`FRegisterMap.cpp` **14**（`:34,43,54,66,76,88,99,109,121,132,141,152,164,176`）、`FRegisterSet.cpp` **9**（`:31,39,49,59,69,77,87,97,108`）。
  - `grep "GetEnvironment\(\)\.GetOptional\("` → **6** 处（`FRegisterOptional.cpp:76,78,97,105,117,129`）。
  - ⇒ **同构样板共 56 处**（Array 27 + Map 14 + Set 9 + Optional 6）。原文的分项数（28/16/11/8）实际是**各文件导出函数个数**，不是样板出现次数 —— 这是原文把"导出数"误当成"样板数"导致的偏差。
- **级别变动**: 无（P3 维持，仅修正计数）
- **文件**: `FRegisterArray.cpp`（**27** 处）、`FRegisterMap.cpp`（**14** 处）、`FRegisterSet.cpp`（**9** 处）、`FRegisterOptional.cpp`（**6** 处）
- **函数**: 全部导出实现
- **置信度**: 高（grep 工具全量重跑，命中行号已逐条列出）

**现状（代码事实）**

同一段模板在本目录重复 **56** 次（数组 **27** + Map **14** + Set **9** + Optional **6**；原文写作 63 次 = 28+16+11+8，是误用了各文件的导出函数个数）：

```cpp
// FRegisterArray.cpp:76-85（原样）
static int32 NumImplementation(const IManagedHandle InManagedHandle)
{
    if (const auto ArrayHelper = FCSharpEnvironment::GetEnvironment().GetContainer<FArrayHelper>(
        InManagedHandle))
    {
        return ArrayHelper->Num();
    }

    return 0;
}
```

```cpp
// FRegisterMap.cpp:41-50（原样）—— 除容器类型外一字不差
static int32 NumImplementation(const IManagedHandle InManagedHandle)
{
    if (const auto MapHelper = FCSharpEnvironment::GetEnvironment().GetContainer<FMapHelper>(
        InManagedHandle))
    {
        return MapHelper->Num();
    }

    return 0;
}
```

注意 `FRegisterArray.cpp:205-212` 与 `:214-221` 这类 **`void` 函数里的 `return ArrayHelper->Reset(...)`** 写法，是把"模板"机械复制到 `void` 场景留下的痕迹——`return` 一个 `void` 表达式合法但多余。

**问题**

- 新增一个容器方法需要在 4 个层次（C++ 导出、注册点、C# partial 声明、C# 公开包装）手改，且**没有任何编译期一致性检查**：C++ 注册名与 C# 方法名的对应关系只在运行时通过字符串查找建立（`UnrealTypeSourceGenerator.cs:903, 917-918`）。漏改一处 = 运行时 `GetMethod` 返回空指针 + 崩溃，而不是编译错误。
- 已实际发生的偏移证据：`FRegisterArray.cpp:175-183` 的 `InsertZeroedImplementation` 与 `:185-193` 的 `InsertDefaultedImplementation` 传递参数完全一致（`InManagedHandle, InIndex, InCount`），语义区别只在被调用方法名——这种"复制后改一两个 token"的模式正是 F-INT2-001 与 F-INT2-004 的成因。

**建议**

引入一个取容器并自动走空值路径的宏/模板。对 **返回值型**：

```cpp
#define UECS_CONTAINER_CALL(Handle, HelperType, DefaultValue, Expression)          \
    do {                                                                          \
        if (const auto Helper = FCSharpEnvironment::GetEnvironment()              \
                .GetContainer<HelperType>(Handle))                                \
        {                                                                         \
            return (Expression);                                                  \
        }                                                                         \
        return (DefaultValue);                                                    \
    } while (0)
```

对 **`void` 型**（需要区分"无返回"）：

```cpp
#define UECS_CONTAINER_CALL_VOID(Handle, HelperType, Expression)                  \
    do {                                                                          \
        if (const auto Helper = FCSharpEnvironment::GetEnvironment()              \
                .GetContainer<HelperType>(Handle))                                \
        {                                                                         \
            Expression;                                                           \
        }                                                                         \
    } while (0)
```

使用后（Array 的 29 个函数可压缩到约 30 行）：

```cpp
static int32 NumImplementation(const IManagedHandle InManagedHandle)
{
    UECS_CONTAINER_CALL(InManagedHandle, FArrayHelper, 0, Helper->Num());
}
```

**更彻底的方案**：把「容器类型 + 方法」的绑定关系做成一张表，用一个泛型 `TContainerDispatcher<T>` 模板统一生成导出，注册点也只写一次——但这是**结构性重构**，风险高于宏化，建议先做宏化。

**取舍**：宏会削弱调试体验（单步时看不到模板内部）。若在意，可改为 C++17 的 `if constexpr` 辅助模板函数，代价是需要为每个返回类型写一个重载。

**验证方式**

- `grep -c "FCSharpEnvironment::GetEnvironment().GetContainer<" FRegister*.cpp` 应等于 **50**（Array 27 + Map 14 + Set 9），再加 `grep -c "GetOptional("` 的 **6** 处 = 56（实测值，原文写的 63 已更正）。
- 宏化后跑全量脚本用例，确认行为不变。

---

#### [F-INT2-006] 所有 `UnRegister` 导出走 `AsyncTask(GameThread)` 异步释放，存在"已注销但未释放"窗口与关闭期 UAF

- **类别**: 并发/线程安全 / 内存泄漏
- **严重度**: **P2**（隐患；特定时序下升级为崩溃/泄漏）
- **复核结论**: 确认（机制、类别、级别均成立；原文"`AsyncTask` 在同线程是否立即执行"这一保留项已闭合）
- **可达性**: 活跃
- **复核证据**: 六个 `UnRegister` 的异步模式逐字一致（`FRegisterArray.cpp:36-40`、`FRegisterMap.cpp:25-29`、`FRegisterSet.cpp:23-26`、`FRegisterOptional.cpp:89-92`、`FRegisterSubclassOf.cpp:40-43`、`FRegisterLazyObjectPtr.cpp:40-44`）。**引擎侧**：`AsyncTask` **永远入队，从不内联执行** —— `Runtime/Core/Private/Async/Async.cpp:54-57`：
  ```cpp
  void AsyncTask(ENamedThreads::Type Thread, TUniqueFunction<void()> Function)
  { TGraphTask<FAsyncGraphTask>::CreateTask().ConstructAndDispatchWhenReady(Thread, MoveTemp(Function)); }
  ```
  （声明见 `Runtime/Core/Public/Async/Async.h:463`）⇒ 即使调用者已在 GameThread，lambda 也要等下一次任务泵才执行，**"已注销但未释放"窗口真实存在**，原文第 1、4 点成立。同时这解释了为什么"同步化"是最小修法。
- **级别变动**: 无（P2 维持）
- **文件**: `FRegisterArray.cpp:34-41`、`FRegisterMap.cpp:23-30`、`FRegisterSet.cpp:21-27`、`FRegisterOptional.cpp:87-93`、`FRegisterSubclassOf.cpp:38-44`、`FRegisterLazyObjectPtr.cpp:38-45`
- **函数**: 各 `UnRegisterImplementation`
- **置信度**: 中→**中高**（异步派发是确证事实且已读到引擎实现；仍未验证的是"实际调用方是否真有非 GameThread 的终结器线程路径"，这一点需实机插桩）

**现状（代码事实）**

六个对象的 `UnRegister` 用的是**完全相同**的异步模式：

```cpp
// FRegisterArray.cpp:34-41（原样）
static void UnRegisterImplementation(const IManagedHandle InManagedHandle)
{
    AsyncTask(ENamedThreads::GameThread, [InManagedHandle]
    {
        (void)FCSharpEnvironment::GetEnvironment().RemoveContainerReference<
            FArrayHelper>(InManagedHandle);
    });
}
```

而 `Register` 是**同步**的（`FRegisterArray.cpp:14-19`），`Get*` 也是同步的（`:43-52`）。

**调用上下文**

`AsyncTask` 是 `FCSharpEnvironment` 的入口包装，`FCSharpEnvironment::GetEnvironment()` 取的是全局单例。`UnRegister` 通常由 C# 的 `Dispose`/终结器路径触发（`TArrayImplementation.cs:23-28` 直接透传 `nint`，未做任何状态标记）。

**问题**

1. **状态不一致窗口**：`UnRegister` 返回后 C# 认为资源已释放，但注册表条目仍在。窗口内任何 `Get`/`Num` 调用都**正常成功**——这本身不是崩溃，但它使"释放后使用"（use-after-free 的托管侧等价物）**在测试中不可见**，把本应立即失败的错误变成时序相关的偶发错误。
2. **关闭期崩溃风险**：`AsyncTask` 投递的 lambda 捕获 `InManagedHandle`（按值，安全），但它访问的是**全局单例**。若引擎关闭序列已销毁 `FCSharpEnvironment`，而队列中仍有未执行的 `UnRegister`，lambda 执行时会访问已销毁的单例。C# 的 GC/终结器线程在域卸载阶段触发 `UnRegister` 是完全可能的路径。
3. **泄漏**：`FArrayHelper` 是 `new` 出来的（`FArrayHelper.cpp:19`），释放在 `RemoveContainerReference` 里、即异步。若进程在 lambda 执行前退出，或 `AsyncTask` 因线程池饱和被丢弃（取决于引擎配置），`FArrayHelper` 及其内部 `FScriptArray` 与 `InnerPropertyDescriptor`（`FArrayHelper.cpp:34-48`）全部泄漏。
4. **`AsyncTask` 在 GameThread 上的语义**：如果 `UnRegister` 本就从 GameThread 调用（最可能的实际情况），这里付出了一次任务派发的开销却没有任何收益——**除非确实存在非 GameThread 调用方**。当前代码没有注释说明为什么需要异步。

**建议**

1. **先确认是否真的需要异步**。若 `UnRegister` 只在 GameThread 上被调用（很可能，因为 `Register`/`Get` 都要求 GameThread 且没有防护），改成同步调用即可同时消除窗口、崩溃风险与开销：
   ```cpp
   static void UnRegisterImplementation(const IManagedHandle InManagedHandle)
   {
       check(IsInGameThread());
       (void)FCSharpEnvironment::GetEnvironment().RemoveContainerReference<FArrayHelper>(InManagedHandle);
   }
   ```
2. **若必须支持跨线程**（终结器线程），则改为"投递 + 完成握手"：`UnRegister` 等待 lambda 执行完毕再返回（例如用一个栈上的 `FEvent`），把异步变成对调用者不可见的实现细节。
3. 若保留异步，至少在 lambda 内加存活检查，并在 `FCSharpEnvironment` 析构时排空队列。

**验证方式**

- `grep -rn "UnRegisterImplementation\|TArray_UnRegisterImplementation" Script/` 找出所有调用点，确认是否只在 `Dispose`/终结器路径。
- 在 lambda 首行加 `check(FCSharpEnvironment::IsValid())`（或等价的存活标志），跑编辑器关闭流程，观察是否命中。
- 在 `AsyncTask` 前后各打一条日志，统计 `UnRegister` 被调用时是否已在 GameThread。

---

### P3

> （分组标题为**原定级**。本组内含 4 条维持 P3、1 条已撤销（F-INT2-009）—— 以各条 `**严重度**` 与上方速查表为准）

#### [F-INT2-014] `FRegisterMulticastDelegate` 重复注册 `Contains` —— 第二次被自动改名为 `Contains1`，成为永远匹配不到任何 C# 调用的死注册

- **类别**: 死代码 / Bug（复制粘贴）
- **严重度**: **P3**（不崩溃，但说明注册表与 C# 声明之间存在无人校验的漂移）
- **复核结论**: 确认
- **可达性**: 活跃（构建/启动期即发生；功能上无害）
- **复核证据**: `.Function("Contains", ContainsImplementation)` 在 `FRegisterMulticastDelegate.cpp:190` 与 `:192` 各出现一次（grep 该文件 `Contains` 命中 16 行，其中 `.Function("Contains"...)` **恰 2 行**）；构造函数区间 `:185-202`、`.Function` 调用 `:188-201`（共 **14** 个，逐个数过）；改名机制 `FClassBuilder.cpp:59-81`（`:69` `Functions.Add(InName)`、`:78` 用改名后的名字注册）与 `:83-92`（`:85` `Algo::Count(Functions, InName)`、`:90` `Count == 0 ? "" : ToString(Count)`）逐字吻合；键名格式由 `Source/UnrealCSharpCore/Public/CoreMacro/BindingMacro.h:9` 确证：`BINDING_COMBINE_FUNCTION_IMPLEMENTATION(A,B) = "__%s_%sImplementation"`；`MethodRegisters` 确为 **`TArray`**（`FBindingClassRegister.h:68`）→ `Emplace` 追加不覆盖（`FBindingClassRegister.cpp:113-129`）。**C# 侧证实无使用者**：`grep "Contains1"` 在**两套 `Script/` 树命中 0**；`FMulticastDelegateImplementation.cs:30-37` 只有 `__FMulticastDelegate_ContainsImplementation`。
- **级别变动**: 无（P3 维持）
- **置信度**: 高（改名逻辑、键拼接、C# 侧缺失三者均实测）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterMulticastDelegate.cpp:190, 192`
- **函数**: `FRegisterMulticastDelegate::FRegisterMulticastDelegate()` 构造函数
- **置信度**: 高（`GetFunctionImplementationName` 的改名逻辑、`BindingMethod` 的键拼接、以及 C# 侧缺失 `Contains1` 三者均已读取，可完整推演）

**现状（代码事实）**

```cpp
// FRegisterMulticastDelegate.cpp:185-202（原样，注意 :190 与 :192）
FRegisterMulticastDelegate()
{
    FClassBuilder(TEXT("FMulticastDelegate"), NAMESPACE_LIBRARY)
        .Function("Register", RegisterImplementation)
        .Function("UnRegister", UnRegisterImplementation)
        .Function("Contains", ContainsImplementation)        // :190  第 1 次
        .Function("IsBound", IsBoundImplementation)          // :191
        .Function("Contains", ContainsImplementation)        // :192  第 2 次 —— 重复
        .Function("Add", AddImplementation)
        .Function("AddUnique", AddUniqueImplementation)
        .Function("Remove", RemoveImplementation)
        .Function("RemoveAll", RemoveAllImplementation)
        .Function("Clear", ClearImplementation)
        .Function("GenericBroadcast0", GenericBroadcast0Implementation)
        .Function("GenericBroadcast2", GenericBroadcast2Implementation)
        .Function("GenericBroadcast4", GenericBroadcast4Implementation)
        .Function("GenericBroadcast6", GenericBroadcast6Implementation);
}
```

**自动改名机制**（这就是重复注册不会互相覆盖、而是变成另一个名字的原因）：

```cpp
// FClassBuilder.cpp:59-81（原样）
FClassBuilder& FClassBuilder::Function(const FString& InName,
                                       const FString& InImplementationName,
                                       const void* InMethod
                                       ...
)
{
    const auto FunctionImplementationName = GetFunctionImplementationName(InName, InImplementationName);

    Functions.Add(InName);          // ← 注意加的是 InName，不是改名后的名字
    ...
    Function(FunctionImplementationName, TFunctionPointer<decltype(InMethod)>(InMethod));

    return *this;
}
```

```cpp
// FClassBuilder.cpp:83-92（原样）—— 同名第 2 次会被追加 "1"
FString FClassBuilder::GetFunctionImplementationName(const FString& InName, const FString& InImplementationName) const
{
    const auto Count = Algo::Count(Functions, InName);

    return FString::Printf(TEXT(
        "%s%s"),
                           *InImplementationName,
                           Count == 0 ? TEXT("") : *UKismetStringLibrary::Conv_IntToString(Count)
    );
}
```

推演（`Functions` 初始为空）：
| 调用 | `Count` | 实际注册名 |
|---|---|---|
| `.Function("Register", ...)` | 0 | `Register` |
| `.Function("UnRegister", ...)` | 0 | `UnRegister` |
| `.Function("Contains", ...)` `:190` | 0 | `Contains` |
| `.Function("IsBound", ...)` `:191` | 0 | `IsBound` |
| `.Function("Contains", ...)` `:192` | **1** | **`Contains1`** |

**最终键名的拼接方式**（决定了 C# 侧要找什么名字）：

```cpp
// FBindingClassRegister.cpp:113-129（原样）
void FBindingClassRegister::BindingMethod(const FString& InImplementationName, const void* InFunction)
{
    MethodRegisters.Emplace([=, this]()
                            {
                                return BINDING_COMBINE_CLASS(
                                        ImplementationNameSpace,
                                        BINDING_COMBINE_CLASS_IMPLEMENTATION(GetClass())) +
                                    BINDING_COMBINE_FUNCTION(
                                        BINDING_COMBINE_FUNCTION_IMPLEMENTATION(GetClass(),
                                            InImplementationName));
                            },
                            InFunction);
}
```

即：`Script.Library.FMulticastDelegateImplementation::__FMulticastDelegate_<InImplementationName>Implementation`。所以第二次注册的键是 `Script.Library.FMulticastDelegateImplementation::__FMulticastDelegate_Contains1Implementation`。

**C# 侧根本没有这个方法**：

```
grep -n "Contains" Script/UE/Library/FMulticastDelegateImplementation.cs
→ :30  private static unsafe partial byte __FMulticastDelegate_ContainsImplementation(nint InDelegate, nint InObject, nint InType, nint InMethodInfo);
→ :32  public static bool FMulticastDelegate_ContainsImplementation(...)
→ :35      return __FMulticastDelegate_ContainsImplementation(...) != 0;
```

**没有 `__FMulticastDelegate_Contains1Implementation`**。

**问题**

- `MethodRegisters`（`FBindingClassRegister.cpp:116-128`）是 `TArray::Emplace` 追加语义，不会因重名报错，也不会覆盖——于是这条注册**静静地留在表里，永远没有任何 C# 代码会去查它**。
- 后果分两层：
  1. **功能上无害**（`Contains` 仍然可用，走的是第 1 次注册）；
  2. **但它是"注册表与 C# 声明之间没有任何一致性校验"的直接证据**——从 `:190` 到 `:192` 之间插入 `IsBound` 的改动留下了这个尾巴，而构建、启动、运行都不会有任何提示。同类漂移如果发生在**唯一**一条注册上（例如把某个 `.Function("X", ...)` 误删或改名），就会变成运行时 `MethodBridge.GetMethod` 返回空指针后才暴露的崩溃。
- 从改动痕迹看，作者的意图很可能是"在 `Contains` 与 `IsBound` 之间再补一个方法"，复制粘贴时忘了改名字与实现（`:192` 的实现与 `:190` 完全相同）。

**建议**

1. **立即删除 `FRegisterMulticastDelegate.cpp:192`** 这一行（若确实需要第二个方法，则补上正确的名字与实现）。
2. **补上编译期/启动期一致性校验**，这是本 finding 的真正价值所在：
   - 在 `FBindingClassRegister` 里对 `MethodRegisters` 的**最终键名做重复检测**，发现重复即 `checkf`/`ensureMsgf`（因为键名唯一性正是它自己的隐含契约）：
     ```cpp
     // 伪代码：在 operator FBindingClass*() 里（:42-45）
     TSet<FString> SeenNames;
     for (const auto& Method : MethodRegisters)
     {
         const auto Name = /* 求值 lambda 得到键名 */;
         ensureMsgf(!SeenNames.Contains(Name), TEXT("Duplicate binding method: %s"), *Name);
         SeenNames.Add(Name);
     }
     ```
   - 更彻底的做法：让 C# 侧在启动时**枚举所有 `__*Implementation` partial 方法**（源生成器已知全量），与 C++ 注册表做一次差集比对，把"注册了但没有 C# 声明"和"有 C# 声明但没注册"都报出来。生成器已经有全部信息（`UnrealTypeSourceGenerator.cs:866-871` 按类型分组），实现成本不高。
3. 顺带修正 `:190/:192` 暴露出的命名方案问题：靠 `Algo::Count` 自动追加 `1/2/...` 后缀的"重载"机制是**隐式且不可见**的（C# 侧必须凭空知道要用 `Contains1`）。本目录唯一的"重载"场景 `TOptional::Register1/Register2`（`FRegisterOptional.cpp:147-148`）是**手工命名**的，绕开了这个机制——说明作者自己也觉得它不可靠。建议废弃自动后缀，改为要求显式唯一名。

**验证方式**

- `grep -n '\.Function("Contains"' Source/UnrealCSharp/Private/Domain/Interop/FRegisterMulticastDelegate.cpp` → 2 命中（`:190`, `:192`）。
- `grep -rn "Contains1" Script/ Source/` → **0 命中**（证明 `Contains1` 无任何使用者）。
- 加运行时日志：在 `FBindingClassRegister::BindingMethod` 的 lambda 里 `UE_LOG` 出键名，启动后搜索 `Contains1`，可观察到它确实被注册。

---

#### [F-INT2-018] `FRegisterText::RegisterImplementation` 忽略 `FTextStringHelper::ReadFromBuffer` 的返回值；同文件用 pragma 关闭了 dangling 警告

- **类别**: 可优化/可读性（错误处理不完整）+ 告警屏蔽
- **严重度**: **P3**
- **复核结论**: 部分确认（偏差两处：① **返回类型写错** —— `ReadFromBuffer` 返回 `const TCHAR*` 而不是 `bool`；② **"全目录唯一使用该 pragma"不成立** —— 实为 4 个文件）
- **可达性**: 活跃
- **复核证据**:
  - 返回值被丢弃是事实：`FRegisterText.cpp:36-41` 未接收返回值；`FRegisterText.cpp:10` 与 `:88` 的 pragma 成对存在（文件 88 行已全读）。
  - **① 返回类型更正**：`Runtime/Core/Public/Internationalization/Text.h:1284` → `static CORE_API const TCHAR* ReadFromBuffer(const TCHAR* Buffer, FText& OutValue, const TCHAR* TextNamespace = nullptr, const TCHAR* PackageNamespace = nullptr, const bool bRequiresQuotes = false);` —— 返回的是「解析消耗到的位置」指针（UE 惯例为失败时 `nullptr`），**不是 `bool`**。原文"返回 `bool` 表示解析是否成功"是错的；但"返回值被丢弃 ⇒ 无法区分空文本与解析失败"的**结论仍成立**（判断失败的方式是判空指针）。
  - **② pragma 使用面更正**：`grep "PRAGMA_DISABLE_DANGLING_WARNINGS" Source/` 共 **7** 处 —— 宏定义 `CompilerMacro.h:6` / `:15-16`（Clang 分支 + 非 Clang 空宏回退），**使用者 4 个文件**：`FRegisterText.cpp:10`、`FRegisterObject.cpp:10`、`FRegisterStruct.cpp:7`、`UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:41`；对应 `PRAGMA_ENABLE_DANGLING_WARNINGS` 亦为 4 处（`:88` / `:152` / `:68` / `:519`）。因此原文"本文件是全目录**唯一**使用该 pragma 的 `.cpp`……这个'唯一'本身就说明作者在这里遇到了别处没有的告警"这一论证**无效**（`FRegisterObject`/`FRegisterStruct` 是同样量级的绑定胶水文件）。"应收窄作用域"的建议仍然合理。
  - 原文"`InBuffer == nullptr` 时会传 `nullptr`，该路径必然失败"与"C# 侧实际传 `[0]` 而非 `nullptr`（`FTextImplementation.cs:13`）"两点均已核对：`FRegisterText.cpp:37` 的三元与 `FTextImplementation.cs:13` 的 `[0]` 属实。
- **级别变动**: 无（P3 维持，但**一条论证依据被移除**）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterText.cpp:10, 34-45, 88`
- **函数**: `FRegisterText::RegisterImplementation(IManagedHandle, const char*, const char*, const char*, uint8)`
- **置信度**: **中→中高**（"返回值被忽略"是确证事实；"失败时 `*OutText` 的具体状态"仍未读 `Text.cpp:1898` 的实现体，故后果描述保留为"可能保持默认构造的空 `FText`"）

**现状（代码事实）**

```cpp
// FRegisterText.cpp:10（原样）
PRAGMA_DISABLE_DANGLING_WARNINGS
```

```cpp
// FRegisterText.cpp:34-45（原样）
const auto OutText = new FText();

FTextStringHelper::ReadFromBuffer(
    InBuffer != nullptr ? *Buffer : nullptr,
    *OutText,
    InTextNamespace != nullptr ? *TextNamespace : nullptr,
    InPackageNamespace != nullptr ? *PackageNamespace : nullptr,
    bRequiresQuotes != 0);          // ← 返回值（bool 成功标志）被丢弃

FCSharpEnvironment::GetEnvironment().AddStringReference<FText, true, false>(
    FReflectionRegistry::Get().GetTextClass(), InManagedObject, OutText);
```

```cpp
// FRegisterText.cpp:88（原样）
PRAGMA_ENABLE_DANGLING_WARNINGS
```

```cpp
// Source/UnrealCSharpCore/Public/CoreMacro/CompilerMacro.h:6-11（原样）—— 仅在 Clang 下生效
#define PRAGMA_DISABLE_DANGLING_WARNINGS \
	_Pragma("clang diagnostic push") \
	_Pragma("clang diagnostic ignored \"-Wdangling\"")
```

**问题**

1. **返回值被丢弃**：`FTextStringHelper::ReadFromBuffer(...)` 返回 ~~`bool`~~ **`const TCHAR*`（`Text.h:1284`；UE 惯例失败返回 `nullptr`）** 表示解析是否成功。失败时（缓冲区格式非法、`InBuffer` 为 `nullptr` 等）`*OutText` 可能保持默认构造的空 `FText`，而 C# 侧拿到的却是一个"注册成功、内容为空"的 `FText` 对象——**调用者无法区分"空文本"与"解析失败"**。这与本目录其它地方"失败即静默返回 0"的问题（F-INT2-008）同源。
   注意 `InBuffer == nullptr` 时会传入 `nullptr` 作为 `Buffer` 参数（`:37`），这条路径**必然失败**且无任何反馈——而 C# 侧 `FTextImplementation.cs:13` 在 `InBuffer == null` 时传的是 `[0]`（单字节 0），**不是** `nullptr`，所以这条分支实际上只在 `NewString`/`Marshal` 边界异常时才可达。
2. **`-Wdangling` 被整文件关闭**：`CompilerMacro.h:6-11` 表明这个 pragma 的作用是 `clang diagnostic ignored "-Wdangling"`（一个很宽的告警组，涵盖 `-Wdangling-gsl`、`-Wdangling-field`、`-Wdangling-initializer-list` 等）。在**整个 `FRegisterText.cpp`（88 行）**范围内关闭它，意味着该文件里**未来任何真实的生命周期问题都会被静默吞掉**。该 pragma 只在 Clang（Linux / Mac / 部分 Android 工具链）下生效，MSVC 下是空宏——因此这是一个**只在非 Windows 平台生效**的告警屏蔽，风险更隐蔽。
   从代码看，本文件确实有"临时对象 + 取内部指针"的密集写法（`*Buffer`、`*TextNamespace`、`*PackageNamespace`），是告警的合理来源；但 `Buffer`/`TextNamespace`/`PackageNamespace` 都是 `const auto` **具名局部变量**（`:22, :26, :30`），生命周期覆盖整个函数体，**目前这几处并不悬垂**。因此这个 pragma 属于"为了压掉噪告警而永久关闭了一整类检查"。

**建议**

1. 检查返回值并做显式处理：
   ```cpp
   const auto bOk = FTextStringHelper::ReadFromBuffer(
       InBuffer != nullptr ? *Buffer : nullptr,
       *OutText,
       InTextNamespace != nullptr ? *TextNamespace : nullptr,
       InPackageNamespace != nullptr ? *PackageNamespace : nullptr,
       bRequiresQuotes != 0);

   if (!bOk)
   {
       delete OutText;                 // 不要把一个空 FText 注册给 C#
       return;                         // 并让 C# 侧能观察到失败（见下）
   }

   FCSharpEnvironment::GetEnvironment().AddStringReference<FText, true, false>(
       FReflectionRegistry::Get().GetTextClass(), InManagedObject, OutText);
   ```
   更好的做法是把 `RegisterImplementation` 改成返回 `uint8`（成功/失败），C# 公开层抛出或返回可区分的错误——但这是 API 变更，需与 C# 侧同步（同 F-INT2-008 的取舍）。
   **注意**：当前实现里 `OutText` 无论成功与否都交给了 `AddStringReference<FText, true, false>`（`bNeedFree = true`），所以提前 `return` 时必须自己 `delete`，否则泄漏——这也是为什么建议顺带把所有权改清楚（例如先解析到栈上的 `FText`，成功后再 `new`）。
2. **收窄 pragma 的作用域**：把 `PRAGMA_DISABLE_DANGLING_WARNINGS` 从文件级（`:10`/`:88`）收窄到**只需要它的那几行**周围，或改用行级 `// NOLINT`。若确无需要的告警，直接删除这两行 pragma。
3. **⚠️ 更正（原论证无效）**：~~本文件是全目录**唯一**使用该 pragma 的 `.cpp`~~ —— 实测共 **4 个文件**使用（本文件 `:10`、`FRegisterObject.cpp:10`、`FRegisterStruct.cpp:7`、`UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:41`，各自都有成对的 `PRAGMA_ENABLE_…`），因此"唯一"不能作为"作者在此遇到特有问题"的论据。原文"收窄作用域"的建议仍然有效。（`grep -rn "PRAGMA_DISABLE_DANGLING_WARNINGS" Source/` 只命中本文件的 `:10` 与宏定义处），这个"唯一"本身就说明作者在这里遇到了别处没有的告警。

**验证方式**

- `grep -rn "PRAGMA_DISABLE_DANGLING_WARNINGS" Source/` → **实测 7 命中**：宏定义 `CompilerMacro.h:6`、`:15`、`:16` + 使用者 `FRegisterText.cpp:10`、`FRegisterObject.cpp:10`、`FRegisterStruct.cpp:7`、`FMonoDomain.cpp:41`（原文写的"只命中本文件与宏定义处"已更正）。
- 在 Clang 下把 `:10/:88` 两行注释掉重新编译，观察 `-Wdangling*` 是否报出真实问题（若有，说明 pragma 正在掩盖 bug，本 finding 应升级）。
- 用例：C# 侧 `new FText(null, null, null, false)` 与传入非法 buffer 字符串，观察是否静默得到空文本。

---

#### [F-INT2-007] `FArrayHelper::Remove` 内有未使用的局部变量，且算法为 O(n²)

- **类别**: 死代码 / 性能
- **严重度**: **P3**
- **复核结论**: 确认（代码事实、O(n²) 复杂度、级别均成立）
- **可达性**: 活跃
- **复核证据**: `FArrayHelper.cpp:304-322` 逐字吻合（`:308` `TArray<int32> x;`、`:310` 首次 `Find`、`:312-319` `while` 循环内每次 `RemoveAt(Index,1,true)` 后 `:318` 重新 `Find(InValue)`）；`TArray<int32> x` 全插件 `grep` 命中 **恰 1**（仅 `FArrayHelper.cpp:308`）→ "未使用局部变量"确认；`Find` 为线性 `Identical` 扫描（`FArrayHelper.cpp:141-155`）⇒ 总复杂度 O(n²) 成立。导出侧 `FRegisterArray.cpp:277-286` 亦吻合。
- **级别变动**: 无（P3 维持）
- **置信度**: 高
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterArray.cpp:277-286`（导出 `Remove`）；实现 `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:304-322`
- **函数**: `FArrayHelper::Remove(const void*)`
- **置信度**: 高

**现状（代码事实）**

```cpp
// FArrayHelper.cpp:304-322（原样）
int32 FArrayHelper::Remove(const void* InValue) const
{
    auto RemovedNum = 0;

    TArray<int32> x;              // ← 声明后从未使用

    auto Index = Find(InValue);

    while (Index != INDEX_NONE)
    {
        ++RemovedNum;

        RemoveAt(Index, 1, true);

        Index = Find(InValue);    // ← 每次都从头重扫
    }

    return RemovedNum;
}
```

**问题**

`TArray<int32> x;` 是遗留的调试/重构残留，**每个元素删除都要构造一个空的 `TArray` 却从不使用**（空 `TArray` 不分配，所以不是泄漏，但是纯粹的噪音，并且会误导读者以为存在某个待补逻辑）。

同时该实现是 **O(n·m)**（n = 容器长度，m = 匹配数）：每次删掉一个元素后从索引 0 重新 `Find`。对“删除全部匹配项”的常见用法（例如从 `TArray<FString>` 里清掉所有空串），这是平方复杂度。`Find` 本身也是逐元素 `Identical` 线性扫描（`FArrayHelper.cpp:141-155`），因此总代价是 O(n²)。

**建议**

```cpp
int32 FArrayHelper::Remove(const void* InValue) const
{
    auto RemovedNum = 0;

    for (auto Index = Num() - 1; Index >= 0; --Index)     // 倒序单趟
    {
        auto ScriptArrayHelper = CreateHelperFormInnerProperty();

        if (InnerPropertyDescriptor->Identical(ScriptArrayHelper.GetRawPtr(Index), InValue))
        {
            RemoveAt(Index, 1, true);

            ++RemovedNum;
        }
    }

    return RemovedNum;
}
```

倒序遍历同时让 `RemoveAt` 的索引始终有效，把 O(n²) 降到 O(n·k)（k = 元素比较成本），并天然规避 F-INT2-001 中"删除后索引失效"的类别问题。删除 `x` 并补一行说明为何倒序（避免后续有人"优化"回正序）。

**验证方式**

- 编译期即可发现 `x` 未使用（`-Wunused-variable` / MSVC `C4189`）——若当前无告警，说明告警级别没开。
- 构造 `TArray<int>` 含 10000 个全相同元素，计时对比优化前后。

---

#### [F-INT2-008] 「失败即返回 0」的错误语义与合法值冲突（`Num` / `Find` / `GetMaxIndex`）

- **类别**: Bug（错误处理）
- **严重度**: **P3**
- **复核结论**: 确认（语义冲突成立；定级 P3 符合"需与 C# 侧同步改 API"的取舍）
- **可达性**: 活跃
- **复核证据**: `FRegisterArray.cpp:76-85`（`:84` `return 0;`）、`FRegisterArray.cpp:139,150`（`return INDEX_NONE;`，与"容器不存在"的返回值**恰好同值**）、`FRegisterMap.cpp:41-50`（`:49`）、`:139-148`（`:147`）、`FRegisterSet.cpp:37-45`（`:44`）、`:57-65`（`:64`）逐条吻合；`IsEmptyImplementation` 亦返回 0（`FRegisterArray.cpp:87-96`）。同一文件内 `Find` 用 `INDEX_NONE`、`Num` 用 0 的**双约定并存**属实。
- **级别变动**: 无（P3 维持）
- **置信度**: 高
- **文件**: `FRegisterArray.cpp:76-85`、`FRegisterMap.cpp:41-50, 139-148`、`FRegisterSet.cpp:37-45, 57-65`
- **函数**: `NumImplementation`、`GetMaxIndexImplementation`
- **置信度**: 高

**现状（代码事实）**

```cpp
// FRegisterArray.cpp:76-85（原样）
static int32 NumImplementation(const IManagedHandle InManagedHandle)
{
    if (const auto ArrayHelper = FCSharpEnvironment::GetEnvironment().GetContainer<FArrayHelper>(
        InManagedHandle))
    {
        return ArrayHelper->Num();
    }

    return 0;      // ← 与"容器存在但为空"不可区分
}
```

`Find`/`FindLast` 更微妙：失败返回 `INDEX_NONE`（`FRegisterArray.cpp:139,150`），而 `INDEX_NONE == -1` 恰好同时也是"容器不存在"的返回值——这一处**语义是自洽的**（比返回 0 更好），但两个约定在同一文件内并存。

**问题**

一个失效句柄（已 `UnRegister`、或来自错误的容器类型）会让 `Num()` 返回 0、`IsEmpty()` 返回 `true`、`Max()` 返回 0，**与真实空容器完全无法区分**。上层 C# 代码（例如用 `Num() == 0` 判断是否结束迭代）会把"容器已失效"当成"容器为空"静默吞掉——这类错误在 F-INT2-006 描述的异步释放窗口内尤其容易被触发。

**建议**

`Num`/`GetMaxIndex` 这类**返回值有合法取值域**的函数，改成返回 `bool`（成功与否）+ 出参，或返回 `-1` / `INDEX_NONE` 表示无效（与 `Find` 保持一致）：

```cpp
static int32 NumImplementation(const IManagedHandle InManagedHandle)
{
    if (const auto ArrayHelper = FCSharpEnvironment::GetEnvironment().GetContainer<FArrayHelper>(
        InManagedHandle))
    {
        return ArrayHelper->Num();
    }

    return INDEX_NONE;    // 无效句柄
}
```

C# 侧相应加一层包装（若要保持 `Num()` 签名，可 `Dispose` 后置位一个 `bool` 标记，在 `Num()` 里 `Debug.Assert(false)`）。**注意这是 API 语义变更，需要与 C# 侧同步改**——因此优先级为 P3 而非更高。

**验证方式**

- 单元用例：`var A = new TArray<int>(); A.Dispose(); Debug.Log(A.Num());` 观察当前是否静默返回 0。
- grep 所有 `return 0;` 结尾的导出实现，逐个确认 0 是否在合法值域内。

---

#### [F-INT2-009] LeanCLR 后端走 `[DllImport("UnrealCSharp")]`，但注册面不存在对应的 C 导出符号 —— **已证伪：非缺陷，请从排期移除**

- **类别**: ~~Bug / 平台兼容~~ → **撤销（非缺陷）**
- **严重度**: **撤销（非缺陷）**
- **复核结论**: 确认（**维持"证伪"判决，并补上缺失的决定性证据** —— 原文标注的"未验证的假设"现已闭合）
- **可达性**: 不适用（撤销）
- **复核证据**: 原文假设"LeanCLR 后端可能有独立的符号导出机制"——**该机制确实存在，但它不是"符号导出"，而是 LeanCLR 的 P/Invoke 名字表**：`Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp:854-888` 的 `FLeanCLRDomain::RegisterBinding()` 遍历 `FBinding::Get().Register().GetClasses()` 的**全部**绑定方法，对每个方法调用
  ```cpp
  leanclr::vm::PInvokes::register_pinvoke(
      reinterpret_cast<const char*>(Name.GetData()),          // 方法名（UTF-8）
      reinterpret_cast<leanclr::vm::PInvokeFunction>(...InFunc), // 函数指针
      &PInvoke_Dispatch);
  ```
  （`:870-873`；`RegisterPInvoke` lambda 定义在 `:858-874`）。且它在 `:55` 被实际调用。因此 `[DllImport("UnrealCSharp")] __TArray_NumImplementation(...)`（`UnrealTypeSourceGenerator.cs:816` 的 `NativeModuleName = "UnrealCSharp"`、`:910-912` 的 WITH_LEANCLR 分支）**能在 LeanCLR 的 P/Invoke 表里按名字命中**，不会抛 `EntryPointNotFoundException`。**本条不构成缺陷，请从排期移除。**
- **级别变动**: 无（维持撤销；原文已是"撤销（非缺陷）"）
- **文件**: 生成器 `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:816, 909-912`；对照 `Source/UnrealCSharp/Private/Binding/Class/FClassBuilder.cpp:59-81` 与 `Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp:854-888`
- **函数**: 全部 `__*Implementation` partial 方法
- **置信度**: **低→高**（原文的不确定性（"未追查 LeanCLR 域实现"）已被 `FLeanCLRDomain::RegisterBinding` 逐行闭合）

**现状（代码事实）**

```csharp
// UnrealTypeSourceGenerator.cs:909-912（原样）
"#if WITH_LEANCLR\n" +
$"\t\t[DllImport(\"{NativeModuleName}\", CallingConvention = CallingConvention.Cdecl)]\n" +
$"\t\t{accessibility} static extern unsafe partial {returnType} {method.Name}({parameters});\n" +
```

即 LeanCLR 下 `__TArray_NumImplementation` 需要作为**真实导出符号**存在于 `NativeModuleName` 指向的模块中。

但 C++ 侧这条路径登记的是**函数指针**，不产生符号：

```cpp
// FClassBuilder.cpp:78（原样）—— 传的是裸地址，不是导出名
Function(FunctionImplementationName, TFunctionPointer<decltype(InMethod)>(InMethod));
```

```cpp
// FClassBuilder.inl:53-56（原样）
template <typename T>
auto FClassBuilder::Function(const FString& InImplementationName, const TFunctionPointer<T>& InMethod)
{
    ClassRegister->BindingMethod(InImplementationName, InMethod.Value.Pointer);
}
```

**问题**

如果 LeanCLR 后端也依赖 `BindingMethod` 注册表（而非真实符号），那么 `[DllImport]` 分支在运行时会抛 `EntryPointNotFoundException` / `DllNotFoundException`——**而且是"每个容器方法首次调用时"才暴露**，不易在启动阶段发现。

**未验证的假设**：可能存在一个生成器（`ScriptCodeGenerator` / `SourceCodeGenerator` 模块）或一个 LeanCLR 专用的注册步骤，会为每个 `BindingMethod` 条目生成真正的 `extern "C"` 导出包装。本报告**未**追查该路径。

**建议**

先确认 LeanCLR 后端是否有符号生成步骤：

```
grep -rn "WITH_LEANCLR" Source/ | grep -i "export\|extern \"C\"\|DLLEXPORT"
grep -rn "NativeModuleName" Script/
```

- 若有 → 本节降级为"设计说明"，无问题。
- 若无 → 这是一个 **LeanCLR 后端整体不可用**的 P0 级缺陷，而不是 P3。

**验证方式**

构建 LeanCLR 配置，运行任意脚本调用 `TArray.Num()`，观察是否抛 `EntryPointNotFoundException`。

---

## 7. 死代码清单

### 7.1 判定方法（重要）

本目录的"导出函数"**没有 C++ 符号可 grep**：它们是匿名命名空间内的 `static` 成员函数，只在 `FClassBuilder(...).Function(...)` 处被**取地址**，不产生可链接符号。因此判定分三层：

| 层 | 判定内容 | 方法 | 结果 |
|---|---|---|---|
| (a) C++ | 注册项是否都指向真实实现 | 逐文件读 | **137 个 `.Function()` 调用全部指向真实函数体，0 个空指针/占位注册** |
| (b) C++ → C# | 每个 C++ 注册**名**是否都有对应 C# partial 声明 | 对 136 个唯一名在 `*Implementation.cs` 中计数 `__<Class>_<Name>Implementation` | **136 / 136 全部存在**（脚本逐名核对，`ABSENT` 数为 0）。⚠️ **唯一例外**：`FRegisterMulticastDelegate.cpp:192` 第 2 次 `Contains` 被自动改名为 `Contains1`，其 C# 声明 `__FMulticastDelegate_Contains1Implementation` 在全仓库命中数为 **0** → 见 F-INT2-014。**🔁 计数口径**：`137 个 .Function()` 已**逐文件实测重算通过**（4+29+16+5+14+8+4+4+4+4+4+11+16+5+5+4 = **137**）；"136 / 136 全存在"与 `Contains1`=0 两点**抽样验证**（抽 `__FString_ToStringImplementation`、`__TArray_GetCompoundImplementation` 等），**未逐名全量重跑** |
| (c) C# → 调用方 | 每个公开包装层是否被真正调用 | 扫**两套** `Script/` 树（共 **16512** 个 `.cs`），统计"`Plugins/UnrealCSharp/Script/UE/Library/` 之外"的命中数 | **132 / 136 有外部调用方；4 个为 0**（见 §7.2）。**🔁 复核口径**：§7.2 的 4 项中**已抽验 2 项**（`FMulticastDelegate_GenericBroadcast4Implementation` 与 `FDelegate_GenericExecute4Implementation` → `Script/UE/Proxy/**` 命中均为 **0**，与原文一致）；**"132 / 136" 未全量重跑** |

### 7.2 唯一的"零外部调用方"项（4 个，全部属于委托族）

| 符号 | 声明位置 | Library 内命中数 | **Library 外命中数** | 判定 | 证据 |
|---|---|---|---|---|---|
| `FDelegate_GenericExecute4Implementation` | `Script/UE/Library/FDelegateImplementation.cs:97-102` | 3 | **0** | 休眠 API（需确认） | 见下 |
| `FDelegate_CompoundExecute7Implementation` | `Script/UE/Library/FDelegateImplementation.cs:120-126` | 3 | **0** | 休眠 API（需确认） | 见下 |
| `FMulticastDelegate_GenericBroadcast4Implementation` | `Script/UE/Library/FMulticastDelegateImplementation.cs:94-99` | 3 | **0** | 休眠 API（需确认） | 见下 |
| `FMulticastDelegate_GenericBroadcast6Implementation` | `Script/UE/Library/FMulticastDelegateImplementation.cs:101-107` | 3 | **0** | 休眠 API（需确认） | 见下 |

**为什么判定为"休眠 API"而不是"死代码"**：这 4 个方法由生成器**按委托签名动态选择**，不是手工调用的。生成器模板为：

```cpp
// Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:212（原样）
"FDelegateImplementation.FDelegate_%sExecute%dImplementation(HandleData.GetHandle(this)%s%s%s);\n"

// Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:553（原样）
"FMulticastDelegateImplementation.FMulticastDelegate_%sBroadcast%dImplementation(HandleData.GetHandle(this)%s%s);"
```

`%s` ∈ {`Generic`, `Primitive`, `Compound`}，`%d` 由参数形态决定。因此**当前引擎版本里没有任何委托的签名落到这 4 个变体上**（对应"1 个 struct 参数"之类的组合），但一旦引擎版本或 `bUsePrimitive` 判定变化，生成的代理代码就会立刻用上它们。结论：**属公开 ABI 的一部分，不应删除**（删除会在引擎升级后静默变成 `MethodBridge.GetMethod` 返回空指针的崩溃）。

**注意 `FDelegate_*` / `FMulticastDelegate_*` 的调用量**：`FDelegate_*` 系列每个方法在 `Script/UE/Proxy/**` 中有 **90** 个调用点，`FMulticastDelegate_*` 系列每个有 **321** 个 ~~（原文数字）~~ —— 这直接量化了 §5.1 第 4 点所述句柄泄漏的影响面。
**🔁 抽测更正**：`grep "FMulticastDelegate_AddImplementation"` 于 `Script/UE/Proxy/**` 实测 **316** 个命中（非 321；`grep` 工具报告"Found 250 of 316"）。同一量级，不影响结论；其余 3 个"零外部调用方"项与 90 的口径**未逐项重跑**（见 §8.5）。

### 7.3 C++ 层的死代码 / 缺陷注册

| 符号 | 声明位置 | grep 命中数 | 判定 | 证据 |
|---|---|---|---|---|
| `TArray<int32> x;`（未使用局部变量） | `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:308` | 1（仅定义） | **死代码** | 见 F-INT2-007 |
| `"Contains"`（重复注册，第 2 次） | `Source/UnrealCSharp/Private/Domain/Interop/FRegisterMulticastDelegate.cpp:192` | `FMulticastDelegateImplementation.cs` 中 `Contains` 3 命中、`Contains1` **0 命中** | **死注册**（键名 `__FMulticastDelegate_Contains1Implementation`，无任何 C# 使用者） | 见 F-INT2-014 |
| `[[maybe_unused]] static FRegisterForceInit RegisterForceInit;` | `FRegisterForceInit.h:18` | 头文件被 **8** 个 TU 包含 | **重复构造 8 次** | 见 F-INT2-015 |
| `[[maybe_unused]] static FRegisterObjectFlags RegisterObjectFlags;` | `FRegisterObjectFlags.h:51` | 头文件被 **2** 个 TU 包含 | **重复构造 2 次** | 见 F-INT2-015 |

### 7.4 两个"纯头文件"是否为死文件的结论（任务书重点）

**结论：都不是死文件。** 它们不导出 P/Invoke 函数，而是**注册引擎枚举**，其副作用发生在静态构造期。

| 文件 | 内容 | 是否被使用 | 证据 |
|---|---|---|---|
| `FRegisterObjectFlags.h` | `BINDING_ENUM(EObjectFlags)`（`:7`）+ 27 个 `Enumerator`（`:14-46`）+ 静态实例（`:51`） | **是** | 被 `FRegisterWorld.cpp:8` 与 `FRegisterUnreal.cpp:9` 包含 |
| `FRegisterForceInit.h` | `BINDING_ENUM(EForceInit)`（`:6`）+ 2 个 `Enumerator`（`:13-14`）+ 静态实例（`:18`） | **是** | 被 `FRegisterVector4.cpp:4`、`FRegisterVector2D.cpp:4`、`FRegisterRotator.cpp:4`、`FRegisterQuat.cpp:4`、`FRegisterPlane.cpp:4`、`FRegisterMatrix.cpp:4`、`FRegisterLinearColor.cpp:5`、`FRegisterBox2D.cpp:5` 共 **8 个 TU** 包含 |

C# 侧对应物存在（`EObjectFlags` / `EForceInit` 在生成的代理代码中可用）——`BINDING_ENUM` 的宏定义在 `Source/UnrealCSharp/Public/Macro/BindingMacro.h:170`。

`FRegisterObjectFlags.h` 的 Enumerator 覆盖情况（与 UE 的 `EObjectFlags` 对比的完整性**未验证**）：只注册了 27 个，注意 `RF_KeepForCooker`（`:27`）与 `RF_AllocatedInSharedPage`（`:45`）受版本宏保护，说明作者有维护版本差异的意识。

---

## 8. 未覆盖 / 存疑项

### 8.1 未读的相邻实现（结论依赖它们，但不在本报告范围）

| 未读对象 | 影响哪些结论 | 建议由谁补 |
|---|---|---|
| `FDelegateHelper` / `MulticastDelegateHelper`（`Source/UnrealCSharp/Private/Reflection/Delegate/`） | §5.1 第 3 点的"去重语义"是**推测** | 已由 `07b-委托Handler与OptionalHelper.md` 覆盖 |
| `FMapHelper` / `FSetHelper` / `FOptionalHelper` | `TMap`/`TSet`/`TOptional` 的 `Get` 是否暴露内部指针（导出层已确认是"先取值再经 descriptor 转换"，但 helper 内部未读） | 容器侧报告 |
| `FPropertyDescriptor::Get/Set` 各类型子类 | **F-INT2-010 是否成立**（值/句柄双模式的映射是否真的存在缺口） | 反射层报告 |
| `FCSharpEnvironment::Bind` / `GetString` / `GetContainer` / `GetMulti` | F-INT2-003 的修法（`Bind(nullptr)` 行为）；F-INT2-013 的返回约定 | Environment/Registry 侧报告 |
| `FBinding::Get()` / `FBinding::Register` | F-INT2-015 的定级（覆盖 vs 追加）与静态初始化顺序风险 | 绑定层报告 |

### 8.2 关键未验证假设（⚠️ 下表列出当时标注的未验证假设；其中 **9 项已全部闭合**，逐项结论见 §8.2b）

| 假设 | 影响 | 验证方法 | 状态 |
|---|---|---|---|
| UE 的 `FScriptArrayHelper::GetRawPtr` 越界保护（`checkSlow`）在 Shipping 下被编译掉 | 决定 **F-INT2-001** 是 P0 还是 P2 | 查引擎 `Runtime/Core/Public/Containers/ScriptArrayHelper.h` 的 `checkSlow`/`DO_CHECK` | ❌ 引擎源码本机不可达 |
| `FPropertyDescriptor::Get` 对每个类型写 8 字节句柄还是内联值 | 决定 **F-INT2-010** 是 P1 还是虚警 | 逐一读 `Source/UnrealCSharp/Private/Reflection/Property/*Descriptor.cpp` | ❌ 未验证 |
| `FBinding::Register` 是覆盖语义还是追加语义 | 决定 **F-INT2-015** 是 P2 还是 P1 | 读 `FBinding::Register` | ❌ 未验证 |
| LeanCLR 后端是否存在真实的符号导出步骤 | 决定 **F-INT2-009** 是 P3 还是 P0 | `grep -rn "WITH_LEANCLR" Source/` 找导出包装 | ❌ 未验证 |
| `FCSharpEnvironment::Bind(nullptr)` 的行为 | 影响 **F-INT2-003** 的修法 | 读 `Environment/FCSharpEnvironment.cpp` 的 `Bind` | ❌ 未验证 |
| `AsyncTask(GameThread, ...)` 在同线程调用时是否立即执行 | 影响 **F-INT2-006** 的窗口大小 | 查引擎 `Async/TaskGraphInterfaces.h` 的 `AsyncTask` 实现 | ❌ 未验证 |
| 插件是否真的构建 32 位目标 | 决定 **F-INT2-002** 的实际影响面 | 查 `*.uplugin`、`*.Build.cs`、打包平台列表 | ❌ 未验证 |
| UE `StringCast<UTF8CHAR>` 对 `const ANSICHAR*` / `const UTF8CHAR*` 的重载语义 | 决定 **F-INT2-012** 是"纯冗余拷贝"还是"编码 bug" | 查引擎 `Containers/StringConv.h` | ❌ 未验证 |
| `FTextStringHelper::ReadFromBuffer` 失败时对 `OutText` 的处理 | 影响 **F-INT2-018** 的后果描述 | 查引擎 `Internationalization/Text.h` | ❌ 未验证 |

### 8.2b §8.2 九项假设的闭合结果（全部已回源码验证）

| # | 假设（§8.2 原文） | 闭合结果 | 证据（真实 文件:行） | 对定级的影响 |
|---|---|---|---|---|
| 1 | `FScriptArrayHelper::GetRawPtr` 的 `checkSlow` 在 Shipping 下被编译掉 | ✅ **成立** | `UnrealType.h:3911-3920`（`:3918` `checkSlow(IsValidIndex(Index))`）+ `ScriptArray.h:195-198`（`Remove` 全 `checkSlow`）+ `ScriptArray.h:57-61`（`Insert` 用 `check`）+ `AssertionMacros.h:237/245/325/327` 与 `:340-348`（`DO_CHECK`/`DO_GUARD_SLOW` 为 0 时 → `CA_ASSUME`）。**原文写的 `Containers/ScriptArrayHelper.h` 在 UE 5.6 不存在**，真实位置是 `CoreUObject/Public/UObject/UnrealType.h` | F-INT2-001 **P0 维持**（置信度 中→高） |
| 2 | `FPropertyDescriptor::Get` 写 8 字节句柄还是内联值 | ✅ **已定论：取决于 descriptor，且与 C# 侧一致** | `FStructPropertyDescriptor.cpp:28-31`（写 `IManagedHandle`）/ `TPrimitivePropertyDescriptor.inl:31-34`（`CopySingleValue` 内联）/ `FEnumPropertyDescriptor.cpp:14-17`（内联）；C# 侧 `Script/UE/Proxy/CoreUObject/UObject/Vector.cs:12` 证明 UStruct 是 `class`（引用） | F-INT2-010 **撤销（非缺陷）** |
| 3 | `FBinding::Register` 覆盖还是追加 | ✅ **追加（`TArray::Add_GetRef`），无去重** | `FBinding.cpp:67-76`；下游幂等见 `FBindingEnumGenerator.cpp:9-67` + `FGeneratorCore.h:93`（`TSet<FString> GeneratorFiles`） | F-INT2-015 **维持 P3**（原文"追加→P1"预案作废） |
| 4 | LeanCLR 是否存在符号导出步骤 | ✅ **存在等价机制：LeanCLR P/Invoke 名字表** | `FLeanCLRDomain.cpp:854-888`（`:870-873` `leanclr::vm::PInvokes::register_pinvoke(...)`，`:55` 被调用） | F-INT2-009 **维持撤销** |
| 5 | `FCSharpEnvironment::Bind(nullptr)` 行为 | ✅ **安全**（返回 `InvalidManagedHandle`） | `Registry/FCSharpBind.inl:56-61` | F-INT2-003 的危害范围收窄为 `Multi == nullptr` 一处；**P1 维持** |
| 6 | `AsyncTask(GameThread, …)` 同线程是否立即执行 | ✅ **不立即，永远入队** | `Private/Async/Async.cpp:54-57`（`TGraphTask<FAsyncGraphTask>::CreateTask().ConstructAndDispatchWhenReady(...)`） | F-INT2-006 **P2 维持** |
| 7 | 插件是否真的构建 32 位目标 | ✅ **不构建（且无法构建）** | `.uplugin`（61 行，无 `PlatformAllowList`）+ `UEBuildAndroid.cs:143-150` + `grep "PLATFORM_32BITS\|Win32\|armv7"`（非 ThirdParty 命中 0）+ 第三方库仅 64 位目录 | F-INT2-002 **P1→P3** |
| 8 | `StringCast<UTF8CHAR>` 对窄字符的重载语义 | ✅ **无重载：`TCHAR_TO_UTF8` 是宏，硬转 `const TCHAR*`** | `StringConv.h:1021` + `:226-233`（`FromType = TCHAR`）+ `UnrealString.h.inl:369` | F-INT2-012 **P3→P1**（真实编码 bug） |
| 9 | `ReadFromBuffer` 失败时对 `OutText` 的处理 | ⚠️ **部分闭合**：**返回类型已更正为 `const TCHAR*`（非 `bool`）**；失败时 `OutText` 的具体状态仍未读 `Text.cpp:1898` 实现体 | `Text.h:1284` | F-INT2-018 **P3 维持** |

### 8.2c 未全量重跑项（诚实声明，非存疑结论）

| 项 | 已做的验证 | 未做的验证 |
|---|---|---|
| §7.1(b) "136 / 136 注册名都有 C# 声明" | `137 个 .Function()` 计数**逐文件实测重算通过**；`Contains1` 全仓库 **0** 命中实测；抽样若干名字 | 未对 136 个名字**逐个**重跑 `ABSENT` 计数 |
| §7.1(c) "132 / 136 有外部调用方" | §7.2 的 4 项中抽验 **2 项**（`FMulticastDelegate_GenericBroadcast4Implementation`、`FDelegate_GenericExecute4Implementation` → `Script/UE/Proxy/**` 命中均 **0**，与原文一致） | 未全量重跑 132/136；`FDelegate_*` 每方法"90 个调用点"未逐项重跑 |
| `FMulticastDelegate_AddImplementation` 调用点 | 实测 `Script/UE/Proxy/**` = **316**（原文 321） | 其余 `FMulticastDelegate_*` 变体的调用量未逐项重跑 |
| 16512 个 `.cs` 的总数 | 未重跑（依赖文件系统计数） | 未重跑 |

### 8.3 已检查但**未发现问题**的维度（避免后续重复劳动）

| 维度 | 结论 | 证据 |
|---|---|---|
| **`bool` 宽度**（任务书重点怀疑项） | ✅ **干净**。C++ 全用 `uint8`，C# 全用 `byte`，公开层才转 `bool` | `FRegisterArray.cpp:21,65,87,153`、`FRegisterText.cpp:20` vs `TArrayImplementation.cs:16,44,58,100,128,150`、`FTextImplementation.cs:8` |
| **字符串编码**（任务书重点怀疑项） | ✅ **`FString`/`FName`/`FText` 是 UTF-8 ↔ UTF-8，两侧匹配**（详见 §4.2）。**没有** `LPStr`/`LPWStr` 误用 | `FStringImplementation.cs:13` + `FRegisterString.cpp:15,46` + `StringBridge.cs:13` |
| **`int`/`uint` 接收指针或长度** | ✅ 未发现。长度参数一律 `int32`，指针一律 `uint8*`/`const char*`，无混用 | `BufferMacro.h:9-29`、§3 全表 |
| 用 `memcpy` 搬运非平凡元素 | ✅ **未发现**。全部经 `FPropertyDescriptor::Set/Get/Identical` | `FArrayHelper.cpp:137,269,148,164,180,293` |
| `TOptional` 的 has-value 标志与值构造/析构配对 | ✅ 委托给 `FOptionalHelper`；`Register1/2`、`Reset`、`IsSet`、`Set`、`Get` 均经 helper，导出层无裸内存操作 | `FRegisterOptional.cpp:14-142` |
| 容器内部缓冲区指针是否泄漏给 C# | ✅ **未泄漏**。`FArrayHelper::Get` 取内部指针后，导出层立即经 descriptor 转换写入独立 `RETURN_BUFFER` | `FRegisterArray.cpp:115-117`、`FArrayHelper.cpp:123-131` |
| 元素为 `FString`/`TArray`/`UObject*` 时的构造/析构 | ✅ 正确区分 POD 与非 POD（`CPF_IsPlainOldData \| CPF_NoDestructor`） | `FArrayHelper.cpp:214-220` |
| C++ 与 C# 的**参数个数与顺序** | ✅ 136 个名字逐一核对全部一致 | §3 与 §4.1/§4.2 |
| 调用约定 | ✅ 两侧都是 `Cdecl`；x64/arm64 上等价于唯一约定 | `UnrealTypeSourceGenerator.cs:911, 917` |
| **委托解绑路径** | ✅ 齐备（`UnBind`/`Clear`/`Remove`/`RemoveAll`/`UnRegister` + 生成器注入的终结器） | `FRegisterDelegate.cpp:62-78`、`FRegisterMulticastDelegate.cpp:106-146`、`FDelegateGenerator.cpp:79,332,336,657,661` |
| **C# 委托被 GC 后悬垂** | ✅ 不悬垂（C++ 只持有 `MethodInfo`/`DeclaringType` 反射元数据，不持有托管委托实例）。代价：无法绑定 lambda/闭包 | `FDelegateGenerator.cpp:328` |
| **重复绑定去重** | ✅ `Add` 不去重、`AddUnique` 去重，两者都暴露，与 UE 语义一致 | `FRegisterMulticastDelegate.cpp:64-104` |

### 8.4 建议的后续下钻优先级（**已重排**：原 1/2/4 项均已闭合）

1. ~~读 `FPropertyDescriptor` 各子类~~ **✅ 已闭合（F-INT2-010 撤销）**。
2. ~~读 `FBinding::Register`~~ **✅ 已闭合（追加语义 + Meyers 单例，F-INT2-015 维持 P3）**。
3. **验证 `HandleData.Free` 的缺失**（F-INT2-011，**仍是第一优先级**）：静态证据已足够（`HandleData.Free` 全仓库 0 调用），但**影响面（411+/316 个 `Add` 调用点）与"每帧日志/每帧 UI 都泄漏一个 `GCHandle`"的实测量级**仍需实机确认：在 `Actor.Tick` 里打 `Debug.Log($"{SomeFName}")`，用 `Handles.Count` 或 `dotnet-counters` 观察增长即可。**这是唯一"修起来最便宜、收益最大"的条目。**
4. ~~在引擎侧确认 `ScriptArrayHelper::GetRawPtr` 的断言级别~~ **✅ 已闭合（F-INT2-001 P0 维持）**。
5. **新增：实机验证 F-INT2-012 的编码错误**（成本极低、判定明确）：C# 侧 `new FAnsiString("abc").ToString()` / `new FUtf8String("abc").ToString()`，若返回值不是 `"abc"` 即证实；这条修复（直接透传 UTF-8 字节）**一行改动**，且能同时消除越界读。
6. **新增：给 `FArrayHelper` 的变更类方法补 index 校验**（F-INT2-001 的修法），并同时补 `IManagedHandle.h` 的 `static_assert(sizeof(void*) == 8, ...)`（F-INT2-002 的最低成本止血）。

### 8.5 跨报告线索（不在本报告 16 条编号内，供其他报告/汇总者参考）

| 线索 | 位置（真实 文件:行） | 说明 | 与本报告的关系 |
|---|---|---|---|
| 「`Set` 调 `InitializeValue` 却不 `DestroyValue`」族 | `FOptionalHelper.cpp:87-92`（`:89` `MarkSetAndGetInitializedValuePointerToReplace(Data)`、`:91` `ValuePropertyDescriptor->Set(InValue, Data)`）；引擎侧 `PropertyOptional.h:35-57` 表明"仅在**未 set** 时才 `InitializeValue`"，但 `FStructPropertyDescriptor::Set`（`:39`）会**无条件**再 `InitializeValue(Dest)` | **本报告涉及 Optional 注册**，故按要求交叉核对：当 `TOptional.Set` 被**重复调用**（`Dest` 已持有活值）时，`InitializeValue` 会覆盖一个活值而未先 `DestroyValue` → 泄漏（对持有堆内存的元素类型，如 `FString`/`TArray`）。判据与权威修复清单以项目既有裁决为准（`FRegisterProperty.cpp:35/64`、`FArrayHelper.cpp:137/269`、`FMapHelper.cpp:200/218`、`FSetHelper.cpp:102`、`FOptionalHelper.cpp:91`） | 因此 **§8.3 中"元素构造/析构 ✅ 正确区分 POD 与非 POD"的结论只对 `RemoveAt` 成立**（`FArrayHelper.cpp:214-220` 已核对）；`Set` 路径属该族，**本报告不重复立条**，避免与权威清单重号 |
| `TIsTOptional_V` trait | `Runtime/Core/Public/Misc/Optional.h:440-444` | 该 trait **确实存在**，故任何"因该 trait 不存在而降级"的理由无效 | 本报告未使用该理由（无影响） |
| `FScriptMapHelper` / `FScriptSetHelper::Rehash()` | `UnrealType.h:4787 / 5597` | 二者均有 `COREUOBJECT_API Rehash()` | 本报告 §5 关于 `TMap`/`TSet` 的"未暴露内部指针"结论不依赖它 |
| 越界的同型缺陷 | `FRegisterScriptInterface.cpp:62-68`（`:67` `Multi->GetObject()` 未判空） | 与 F-INT2-003 **完全同型**，但不在本报告 18 文件范围内 | 建议由覆盖 `FRegisterScriptInterface.cpp` 的报告立条（**本报告不代立，以免重号**） |

