# 引擎实现库 Library（跨语言 ABI 契约）分析报告

> 本报告各条**严重度见各 Finding 的「复核结论」字段**。
> **两处关键更新**：
> (1) §3.2/§3.3 的「3 个 C++ 注册无 C# 声明」**已被推翻**——其中 2 个（`__UClass_GetDefaultObjectImplementation`、`__UObject_GetWorldImplementation`）在**项目侧生成产物** `Script/UE/Proxy/Binding/` 里有声明**且有真实调用方**，原判定源于 grep 范围只覆盖了插件自身的 `Script/` 树。真实不一致条目只有 **1** 个（`Contains1`）。见 §3.2 与新增 F-CS3-010。
> (2) F-CS3-006 的原始结论因此**部分撤销**；F-CS3-004 的「建议」被 UE 5.6 引擎源码**证伪**（`FSoftObjectPath` 无 `GetAssetPathName`）。





> 分析范围：`Plugins/UnrealCSharp/Script/UE/Library/`（28 个 C# 文件）↔ `Plugins/UnrealCSharp/Source/UnrealCSharp/Private/Domain/Interop/FRegister*.cpp`（58 个文件）
> 覆盖文件：C# 侧 28 个（全部读完）；C++ 侧 **28 个 LIBRARY 注册块（分布在 27 个文件）** + 58 个文件的 948 个 `.Function(` 全量扫描（逐一重数：`.Function(` 总数 = 948、LIBRARY 命名空间 = 214）
>
> **优先级 1 已完成并闭合验证**；优先级 2 完成签名级对照；优先级 3/4 部分完成（见第 11 节）

> **阅读提示**：本报告包含两处对自身假设的**推翻与更正**，都写在正文里并保留证据：
> (1) 任务书假设的 `[DllImport]`/`extern "C"` 机制不成立（§1.1）；
> (2) "同名重复注册会互相覆盖"这一假设被 `FClassBuilder.cpp:83-91` 的**自动数字后缀**机制证伪（§5.3）。
> 更正后的结论是：6 例名字↔目标错配里只有 **3 例是真 bug**，另 3 例是**公开 API 命名错误**。

---


> **处置更新（2026-09-16，G7 活跃路径子集；**已压缩提交为 `1551383b`**）**：本报告 **`F-CS3-001` 仅部分处置** —— `MethodBridge.GetMethod` 增参 `int InParamCount`，未命中/签名不符时**静默返回 `nint.Zero`**（返回值契约与原文一致）。⚠️ **`call 0` 本身未消除**：唯一的行为差别是"个数不符"从"照错原型调用"改为"返回 0（fail-closed）"；可见化（日志/异常）按本轮口径（**不新增注释 / `throw` / 日志 / try-catch**）全部撤回。（与本条同族的 `F-CS1-003`/`F-ABI-002` 是**同一个 `nint.Zero`**，同一补丁一并处理。）⚠️ **可达性更正**：该 `#else` 生成分支**当前已参编**（`Content/Script/UE.dll` 实测无 `DllImportAttribute`、引用 `MethodBridge.GetMethod`），故本条当前是**活跃**而非潜伏；见 [`00-总览/03`](../../00-总览/03-修复优先级评估.md) §1.3 事实更正。
## 0. 覆盖范围与阅读清单

### 0.1 C# 侧（`Script/UE/Library/`，28 个文件，全部读完）

| 文件 | 行数 | C# 类名 | 桥接声明数 | C++ 对应注册块 |
|---|---|---|---|---|
| ClassImplementation.cs | 14 | `UClassImplementation` | 1 | FRegisterClass.cpp:37-51 |
| DataTableFunctionLibraryImplementation.cs | 20 | `UDataTableFunctionLibraryImplementation` | 1 | FRegisterDataTableFunctionLibrary.cpp:46-47 |
| EnhancedInputComponentImplementation.cs | 40 | `UEnhancedInputComponentImplementation` | 3 | FRegisterEnhancedInputComponent.cpp:219-222 |
| FActorSpawnParametersImplementation.cs | 49 | `FActorSpawnParametersImplementation` | 6 | FRegisterWorld.cpp:96-112 |
| FAnsiStringImplementation.cs | 45 | `FAnsiStringImplementation` | 4 | FRegisterAnsiString.cpp:58-62 |
| FDelegateImplementation.cs | 128 | `FDelegateImplementation` | 16 | FRegisterDelegate.cpp:177-193 |
| FFunctionImplementation.cs | 198 | `FFunctionImplementation` | 25 | FRegisterFunction.cpp:352-377 |
| FMulticastDelegateImplementation.cs | 108 | `FMulticastDelegateImplementation` | 13 | FRegisterMulticastDelegate.cpp:187-201 |
| FNameImplementation.cs | 52 | `FNameImplementation` | 5 | FRegisterName.cpp:63-68 |
| FPropertyImplementation.cs | 39 | `FPropertyImplementation` | 4 | FRegisterProperty.cpp:72-76 |
| FStringImplementation.cs | 44 | `FStringImplementation` | 4 | FRegisterString.cpp:51-55 |
| FTextImplementation.cs | 52 | `FTextImplementation` | 4 | FRegisterText.cpp:77-81 |
| FUtf8StringImplementation.cs | 45 | `FUtf8StringImplementation` | 4 | FRegisterUtf8String.cpp:55-59 |
| InputComponentImplementation.cs | 79 | `UInputComponentImplementation` | 8 | FRegisterInputComponent.cpp:332-340 |
| ObjectImplementation.cs | 98 | `UObjectImplementation` | 11 | FRegisterObject.cpp:133-145 |
| StructImplementation.cs | 51 | `UStructImplementation` | 4 | FRegisterStruct.cpp:57-61 |
| TArrayImplementation.cs | 215 | `TArrayImplementation` | 29 | FRegisterArray.cpp:315-344 |
| TLazyObjectPtrImplementation.cs | 39 | `TLazyObjectPtrImplementation` | 4 | FRegisterLazyObjectPtr.cpp:57-61 |
| TMapImplementation.cs | 122 | `TMapImplementation` | 16 | FRegisterMap.cpp:187-203 |
| TOptionalImplementation.cs | 69 | `TOptionalImplementation` | 8 | FRegisterOptional.cpp:146-154 |
| TScriptInterfaceImplementation.cs | 39 | `TScriptInterfaceImplementation` | 4 | FRegisterScriptInterface.cpp:72-76 |
| TSetImplementation.cs | 86 | `TSetImplementation` | 11 | FRegisterSet.cpp:118-129 |
| TSoftClassPtrImplementation.cs | 44 | `TSoftClassPtrImplementation` | 5 | FRegisterSoftClassPtr.cpp:65-70 |
| TSoftObjectPtrImplementation.cs | 44 | `TSoftObjectPtrImplementation` | 5 | FRegisterSoftObjectPtr.cpp:65-70 |
| TSubclassOfImplementation.cs | 39 | `TSubclassOfImplementation` | 4 | FRegisterSubclassOf.cpp:56-60 |
| TWeakObjectPtrImplementation.cs | 39 | `TWeakObjectPtrImplementation` | 4 | FRegisterWeakObjectPtr.cpp:57-61 |
| UnrealImplementation.cs | 74 | `UnrealImplementation` | 7 | FRegisterUnreal.cpp:180-187 |
| WorldImplementation.cs | 15 | `UWorldImplementation` | 1 | FRegisterWorld.cpp:149-150 |
| **合计** | | | **211** | 214 个 `.Function()` 调用点 |

> **关键校正**：C# 类名**不等于**文件名去后缀。`ClassImplementation.cs` 里的类是 `UClassImplementation`，
> `ObjectImplementation.cs` 里是 `UObjectImplementation`，`WorldImplementation.cs` 里是 `UWorldImplementation`。
> 这直接决定 MethodBridge 的 key（§1.1）。取证：对 28 个文件逐个提取 `partial class` 名与
> `private static unsafe partial` 声明（pwsh `Select-String -Encoding UTF8`）。
>
> **实测（`grep` 工具）**：`private static unsafe partial` 在 `Script/UE/Library/` 下命中 **恰好 211** 条
> （不截断），与上表逐文件数值**逐行吻合**（例：`TArrayImplementation.cs` 29 条、`FFunctionImplementation.cs` 25 条、
> `ObjectImplementation.cs` 11 条、`ClassImplementation.cs` 1 条）。**211 这个数字成立。**
>
> **但 211 不是"C++ 注册的 C# 对端总数"**：`Script/UE/Library/` 只是**插件手写**的那一半；
> 另有 **2 个** LIBRARY 命名空间的 C# 声明位于**项目侧生成产物** `Script/UE/Proxy/Binding/`
> （`ClassImplementation.cs:13`、`ObjectImplementation.cs:13`，命名空间同为 `Script.Library`，
> 与手写文件是**同一个 partial class**）。详见 §3.2/§3.3 与 F-CS3-010。

### 0.2 C++ 侧

| 项 | 数值 | 取证方式 |
|---|---|---|
| `Domain/Interop/` 下 `FRegister*.cpp` 总数 | 58 | glob（复跑：58） |
| 全部 `.Function(` 调用点 | **948** | 用 `grep '\.Function\('`（转义点号，比 `Select-String` 更严）复跑 → **恰好 948**，该数字成立 |
| 挂 `NAMESPACE_LIBRARY`（`Script.Library`）的 `.Function()` 调用点 | **214** | **不用减法**，逐个 `read` 28 个 LIBRARY 块后手工求和 = 214（见 §3.2） |
| `NAMESPACE_LIBRARY` 出现的次数 / 涉及文件数 | **28 / 27** | `grep 'NAMESPACE_LIBRARY'` → 28 命中；`FRegisterWorld.cpp` 有 2 个块（`:96`、`:149`） |
| 挂 `NAMESPACE_BINDING`（`Script.Binding`）的其余注册点 | 734 | = 948 − 214（减法，见下方口径说明） |
| `NAMESPACE_BINDING` 注册块数 / 涉及文件数 | **33 / 32** | `grep 'ClassBuilder[<(]'` → 61 个 builder 块 = 28 LIBRARY + 33 BINDING；`FRegisterEnhancedInputComponent.cpp` 同时含 2 个 BINDING 块与 1 个 LIBRARY 块 |
| LIBRARY 块中用 `BINDING_FUNCTION`/`BINDING_OVERLOAD` 的 | **2**（`UClass::GetDefaultObject`、`UObject::GetWorld`） | `read` 逐块确认 |

> **口径警告（新增）**：**948 不是"全部注册 key 数"**。`FClassBuilder::Property`（`FClassBuilder.inl:48-49`）
> 会派生 `Get<Name>`/`Set<Name>` 两个注册点，`TBindingClassBuilder` 构造时（`TBindingClassBuilder.inl:14-22`）
> 还会自动注册 `Constructor`/`Destructor`，`TClassBuilder::Constructor/Destructor/Property` 都**不经过 `.Function(`**，
> 因此**不在 948 之内**。实测对照：项目侧生成产物 `Script/UE/Proxy/Binding/` 的 40 个 `*Implementation.cs`
> 共声明 **950** 个桥接槽（`static unsafe partial` 990 命中 − 40 个 class 声明行），其中包含大量
> `__FXxx_Xxx1Implementation`（构造函数重载）、`__FXxx_DestructorImplementation`、`__FXxx_Get<Prop>Implementation`。
> 所以 **734 ↔ 950 不是矛盾，是两个不同口径**，不能直接对账；LIBRARY 命名空间侧不受影响（该侧 0 个 `.Property(` 派生键，
> 唯一相关的 `FRegisterWorld.cpp:97-106` 的 10 个 `.Property(` 已在 §11 记录）。

### 0.3 **重大发现：`NAMESPACE_BINDING` 的 C# 对端不在插件里**

任务书假设 28 个 `*Implementation.cs` 是 C++ 注册的 C# 对端——这只对 `NAMESPACE_LIBRARY` 成立。
`NAMESPACE_BINDING` 的 C# 对端是**生成代码**，位于**项目侧**（不在插件仓库内）：

```
<Project>/Script\UE\Proxy\Binding\       ← 87 个 .cs（其中 40 个 *Implementation.cs）
<Project>/Script\UE\Proxy\<Module>\      ← 数百个 UE 模块目录
```

例：`Script/UE/Proxy/Binding/RotatorImplementation.cs:11` `public static unsafe partial class FRotatorImplementation`
（命名空间 `Script.Binding`），声明 `__FRotator_EqualsImplementation(...)`（`:55`），被 `Rotator.cs:206` 调用。

**这解释了为什么先例 bug 出现在 `NAMESPACE_BINDING` 文件里**：那条链路上 C++ 注册是唯一真相源，
C# 是前一次生成出来的产物，两者之间没有编译期/链接期一致性校验，也无法在插件内自洽检查。
本报告 §5 对两个命名空间都做了扫描，**6 例错配全部落在 `NAMESPACE_BINDING`**。

> **重要补充（此前的漏项，直接导致 F-CS3-006 误判）**：`Script/UE/Proxy/Binding/` 里**不只有 `Script.Binding`**。
> 该目录的 `*Implementation.cs` 中**至少 8 个**属于 **`Script.Library` 命名空间**——它们是 LIBRARY 命名空间类的生成对端：
> `ClassImplementation.cs:9-13`、`ObjectImplementation.cs:9-13`（有方法声明），
> 以及 `DataTableFunctionLibraryImplementation.cs:11`、`EnhancedInputComponentImplementation.cs:11`、
> `InputComponentImplementation.cs:11`、`StructImplementation.cs`、`WorldImplementation.cs`、
> `FActorSpawnParametersImplementation.cs:12`（`using Script.Library;` 的反向引用，见 `FActorSpawnParameters.cs:6`）。
> 因此**"LIBRARY 命名空间的 C# 对端"= 插件 `Script/UE/Library/` 的 211 个手写声明 + 项目 `Script/UE/Proxy/Binding/` 的 2 个生成声明**，
> 二者是**同一个 assembly 里的同一个 partial class**（`Proxy/Binding/FActorSpawnParameters.cs:6` 的 `using Script.Library;` 是直接证据）。
> §3.2/§3.3/§10 若只用 `Select-String -Path "Script\*\*\*.cs"`（**插件自身的 `Script/` 树**）去查生成对端，
> 范围少了一整个目录，这正是 F-CS3-006 把两个**活的**注册判成"纯死代码"的原因。

### 0.4 未覆盖声明

- 31 个 `NAMESPACE_BINDING` 文件的 734 个注册点只做了**名字↔目标语义**扫描（§5）与**文件级 class 一致性**检查，
  **没有**做 `Script/UE/Proxy/Binding/` 那 211+ 个生成声明的逐条三方对账（见 §11 第 1 条，这是最大剩余风险面）。
- 优先级 2 只做到签名级对照（§4/§6），**未**逐字节核对缓冲区偏移与结构体布局。
- 优先级 3（枚举/常量）只覆盖 4 处；优先级 4 只覆盖 `MethodBridge` 周边。

---

## 1. 模块职责与架构速览

### 1.1 重要更正：这些文件里**没有 `[DllImport]`**

任务书前提"每个 `*Implementation.cs` 用 `[DllImport]` 声明了 C++ 侧 `extern "C"` 导出函数"**与实际代码不符**。实测：

- 全插件 `Source/` 下 `extern "C"` 命中 11 处，**全部位于被排除的 `Source/ThirdParty/LeanCLR/`**；插件自身 7 个模块命中 **0**。
- 28 个 `*Implementation.cs` 中 `[DllImport]` 命中 **0**（仅出现在 `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:911`
  与 `Script/Interop/Bridge/LogBridge.cs:27`）。

真实机制是**三层**：

**第一层（C# 侧）**：只写 `private static unsafe partial` 的**无体声明**，加一层 public 包装做编解码。
```csharp
// Script/UE/Library/FStringImplementation.cs:9
private static unsafe partial void __FString_RegisterImplementation(nint InString, byte* InValue);
```

**第二层（SourceGenerator，编译期）**：`Script/SourceGenerator/UnrealTypeSourceGenerator.cs`（`IIncrementalGenerator`）
把 `Script.Binding` / `Script.Library` 命名空间下、无体的 `static partial` 方法（`LibraryBridgeReceiver`，`:942-966`）
生成实现体（`:873-927`）：
```csharp
// UnrealTypeSourceGenerator.cs:909-919（生成器输出模板）
#if WITH_LEANCLR
        [DllImport("UnrealCSharp", CallingConvention = CallingConvention.Cdecl)]
        private static extern unsafe partial void __FString_RegisterImplementation(nint InString, byte* InValue);
#else
        private static nint __FString_RegisterImplementation_Slot;

        private static unsafe partial void __FString_RegisterImplementation(nint InString, byte* InValue) =>
            ((delegate* unmanaged[Cdecl]<nint, byte*, void>)global::Interop.MethodBridge.GetMethod(
                ref __FString_RegisterImplementation_Slot, "Script.Library.FStringImplementation::__FString_RegisterImplementation"))(InString, InValue);
#endif
```
`NativeModuleName = "UnrealCSharp"`（`:816`）、`BridgeNamespaces = { "Script.Binding", "Script.Library" }`（`:814`）。
Mono / CoreCLR 分支**没有 P/Invoke**，而是把 C++ 传来的裸函数指针强转成 `delegate* unmanaged[Cdecl]<...>` 直接调用。

**第三层（C++ 侧）**：没有 `extern "C"` 导出，而是注册 C++ 静态成员函数指针：
```cpp
// Source/UnrealCSharp/Private/Domain/Interop/FRegisterString.cpp:49-56
FRegisterString()
{
    FClassBuilder(TEXT("FString"), NAMESPACE_LIBRARY)
        .Function("Register", RegisterImplementation)
        .Function("Identical", IdenticalImplementation)
        .Function("UnRegister", UnRegisterImplementation)
        .Function("ToString", ToStringImplementation);
}
```

**名字怎么对上**：C++ 侧 key 由 `FBindingClassRegister::BindingMethod` 拼出
（`Source/UnrealCSharpCore/Private/Binding/Class/FBindingClassRegister.cpp:113-129`），用
`Source/UnrealCSharpCore/Public/CoreMacro/BindingMacro.h:3-9` 的宏：
```cpp
#define BINDING_COMBINE_CLASS(A, B)                     FString::Printf(TEXT("%s.%s"), *A, *B)
#define BINDING_COMBINE_CLASS_IMPLEMENTATION(A)         FString::Printf(TEXT("%sImplementation"), *A)
#define BINDING_COMBINE_FUNCTION(Function)              FString::Printf(TEXT("::%s"), *Function)
#define BINDING_COMBINE_FUNCTION_IMPLEMENTATION(A, B)   FString::Printf(TEXT("__%s_%sImplementation"), *A, *B)
```
代入 `ImplementationNameSpace = "Script.Library"`、`GetClass() = "FString"`、`InImplementationName = "Register"`：
```
"Script.Library" + "." + "FString" + "Implementation" + "::" + "__FString_" + "Register" + "Implementation"
= "Script.Library.FStringImplementation::__FString_RegisterImplementation"
```
与生成器 `UnrealTypeSourceGenerator.cs:903` 的 `key = $"{containingNamespace}.{InOwner.Name}::{method.Name}"`
（`containingNamespace="Script.Library"`、`InOwner.Name` = **C# 类名**、`method.Name` = 声明的方法名）逐字符相等。

**约定**：`C# 方法名 = "__" + <C++ 类名> + "_" + <C++ 实现名> + "Implementation"`，且 `C# 类名 = <C++ 类名> + "Implementation"`。
C++ 类名来源：
- `FClassBuilder(TEXT("X"), NAMESPACE_LIBRARY)` → 字面量 `X`（`FClassBuilder.cpp:19-30`）；
- `TBindingClassBuilder<T>(NAMESPACE_LIBRARY)` → `TClassBuilder.inl:20` 的 `TName<T,T>::Get()`；对 UObject 派生类是
  `TName.inl:129-135` 的 `GetFullClass(T::StaticClass())` = `GetPrefixCPP() + GetName()`（`FUnrealCSharpFunctionLibrary.cpp:387-409`）；
- `FRegisterClass.cpp:37-44` 显式传入 lambda 返回 `GetFullClass(UClass::StaticClass())` = `"UClass"`。

**`<C++ 实现名>` 不是 `InName` 的原样**，而是经过 `FClassBuilder::GetFunctionImplementationName`
（`FClassBuilder.cpp:83-91`）**自动加数字后缀**去重的：同名第 k 次（0-based）注册追加后缀 `k`。
**详见 §5.3——这是本报告最重要的机制发现。**

这条 key 字符串**没有任何编译期或链接期校验**：任一侧拼错 → `MethodBridge.GetMethod` 返回 `0` → 生成代码把 `0` 当函数指针调用 → **空指针调用**（F-CS3-001）。

### 1.2 数据流（一次 `FString` 构造）

```
C# FString ctor
  → FStringImplementation.FString_RegisterImplementation(FString, string)      Script/UE/Library/FStringImplementation.cs:11
  → Encoding.UTF8.GetBytes(InValue + '\0')                                    FStringImplementation.cs:13
  → HandleData.Alloc(InString)                                                FStringImplementation.cs:17
  → __FString_RegisterImplementation(handle, ptr)                             （生成器产物）
  → MethodBridge.GetMethod(ref slot, "Script.Library.FStringImplementation::__FString_RegisterImplementation")
                                                                              Script/Interop/Bridge/MethodBridge.cs:140
  → C++ FRegisterString::RegisterImplementation(IManagedHandle, const char*)  FRegisterString.cpp:13
  → FCSharpEnvironment::AddStringReference<FString,true,false>(...)           FRegisterString.cpp:17
```

注册方向（C++→C#，进程启动时一次）：
```
FRegisterString RegisterString;（静态初始化）                    FRegisterString.cpp:59
  → FClassBuilder(...).Function(...)                              FRegisterString.cpp:51-55
  → FClassBuilder.cpp:67 GetFunctionImplementationName（去重加后缀）  FClassBuilder.cpp:83-91
  → FBindingClassRegister::BindingMethod                          FBindingClassRegister.cpp:113
  → MethodRegisters.Emplace(...)（TArray，插入序）                  FBindingClassRegister.cpp:116-128
  → :42-45  Methods.Emplace(FBindingMethod(Method))
  → FScriptDomainImpl.inl:611-631  遍历 Classes→GetMethods 展平     FScriptDomainImpl.inl:611-631
  → MethodBridgeRegisterBindingFn(Names, Methods, Count)          FScriptDomainImpl.inl:631
  → Interop.MethodBridge.RegisterBinding(byte**, nint*, int)      MethodBridge.cs:13
  → StringToMethod[Name] = Method                                 MethodBridge.cs:25
```

### 1.3 最严重的结构性缺陷：`GetMethod` 返回 0 后不判空

```csharp
// Script/Interop/Bridge/MethodBridge.cs:140-148   ← 逐行复核
public static nint GetMethod(ref nint InSlot, string InName)
{
    if (InSlot == nint.Zero)
    {
        InSlot = StringToMethod.TryGetValue(InName, out var Method) ? Method : nint.Zero;
    }

    return InSlot;
}
```
生成器把返回值**无条件**强转成函数指针并调用（`UnrealTypeSourceGenerator.cs:918-919`）。查不到 → `0` → `call 0` → AV 崩溃。
所有名字/注册风险都汇聚到这一个崩溃点。见 F-CS3-001。

**附带**：查不到时 `InSlot` 仍是 0，于是每次调用都重新查一次字典（负结果不缓存）。见 F-CS3-008。

### 1.4 参数宽度约定（成对核对判据；已逐条验证与实现一致）

| C++ 类型 | 字节 | C# 侧写法 | 证据 |
|---|---|---|---|
| `IManagedHandle` | 8 | `nint` | `Source/UnrealCSharpCore/Public/Domain/Script/IManagedHandle.h:5-7`（`int64 Value`） |
| `const char*` | 8 | `byte*` | `FRegisterString.cpp:13` ↔ `FStringImplementation.cs:9` |
| `uint8` | 1 | `byte` | `FRegisterString.cpp:21` ↔ `FStringImplementation.cs:21` |
| `int32` | 4 | `int` | `FRegisterArray.cpp:43` ↔ `TArrayImplementation.cs:30` |
| `uint32` | 4 | `uint32` | `FFunctionImplementation.cs:7`（`uint InFunctionHash`） |
| `uint8*` 缓冲区 | 8 | `byte*` | `CoreMacro/BufferMacro.h:9,15,21,25,29`——`IN/OUT/RETURN/IN_KEY/IN_VALUE_BUFFER_SIGNATURE` 全部展开为 `uint8*` |
| 枚举 `EObjectFlags`/`ELoadFlags` | 4 | `EObjectFlags`/`ELoadFlags` | `UnrealImplementation.cs:9,29` |

> **与任务书给的"已知校正"一致**：C++ 侧**一律用 `uint8` 而非 `bool`** 作为返回/入参，C# 侧**一律用 `byte`**。
> 这 211 个声明里**没有找到任何 `bool` 直接过界的例外**。需要布尔语义的入参（`bAllowShrinking`/`bNoFail` 等）
> C# 侧显式转换：`TArrayImplementation.cs:133` `(byte)(bAllowShrinking ? 1 : 0)`、
> `FActorSpawnParametersImplementation.cs:18` `(byte)(InValue ? 1 : 0)`——**正确**。

---

## 2. 关键调用链

1. `FStringImplementation.cs:13` → `Encoding.UTF8.GetBytes` → `:17` `HandleData.Alloc` → 生成器 → `MethodBridge.cs:140` → `FRegisterString.cpp:13`
2. `UnrealTypeSourceGenerator.cs:903`（key）↔ `CoreMacro/BindingMacro.h:3-9`（key）——必须逐字符相等
3. `FRegisterString.cpp:59`（静态构造）→ `FClassBuilder.cpp:67/83-91`（去重加后缀）→ `FBindingClassRegister.cpp:113-129` → `FScriptDomainImpl.inl:611-631` → `MethodBridge.cs:25`
4. `TArrayImplementation.cs:9` → `FRegisterArray.cpp:316` `.Function("Register", RegisterImplementation)`
5. `Script/UE/Proxy/Binding/Rotator.cs:206` → `FRotatorImplementation.__FRotator_EqualsImplementation` → `FRegisterRotator.cpp:46` → **`FRotator::IsNearlyZero`**（见 F-CS3-002）
6. `Script/UE/Proxy/Binding/Rotator.cs`（`SetComponentForAxis`）→ `RotatorImplementation.cs:81` → `FRegisterRotator.cpp:65` → **`FRotator::GetComponentForAxis`**（见 F-CS3-003）
7. `Script/UE/Proxy/Binding/SoftObjectPath.cs:71-77` → `SoftObjectPathImplementation.cs:36` → `FRegisterSoftObjectPath.cpp:34` → **`FSoftObjectPath::GetAssetName`**（见 F-CS3-004）

---

## 3. 三方名字对账表（优先级 1 主体产出）

**核对方法**：C# 声明字面名 `__{Class}_{Name}Implementation` 反解 `(Class, Name)`；C++ 侧限定在**同一 builder 块**
（块的命名空间取块内首个 `NAMESPACE_LIBRARY`/`NAMESPACE_BINDING`；块内 class 名取 builder 模板实参或 `TEXT("...")` 字面量）
中找 `.Function("{Name}", ...)`。三个方向：
- **①** C# 有声明、C++ 无注册 → **空函数指针调用（P0）**
- **②** C++ 有注册、C# 无声明 → 字典死条目（P2）
- **③** 名字在、目标符号语义不对 → **静默错误（P1）** 或公开 API 命名错误（P2）

### 3.1 逐组对账结果（28 组全覆盖）

| # | C# 类 | C# 文件 | C++ 注册块（文件:行） | C# 声明 | C++ 调用点 | ① | ② | ③ |
|---|---|---|---|---|---|---|---|---|
| 1 | `FAnsiStringImplementation` | FAnsiStringImplementation.cs | FRegisterAnsiString.cpp:58-62 | 4 | 4 | 0 | 0 | 0 |
| 2 | `TArrayImplementation` | TArrayImplementation.cs | FRegisterArray.cpp:315-344 | 29 | 29 | 0 | 0 | 0 |
| 3 | `UClassImplementation` | ClassImplementation.cs | FRegisterClass.cpp:37-51 | 1 | 2 | 0 | **1** | 0 |
| 4 | `UDataTableFunctionLibraryImplementation` | DataTableFunctionLibraryImplementation.cs | FRegisterDataTableFunctionLibrary.cpp:46-47 | 1 | 1 | 0 | 0 | 0 |
| 5 | `FDelegateImplementation` | FDelegateImplementation.cs | FRegisterDelegate.cpp:177-193 | 16 | 16 | 0 | 0 | 0 |
| 6 | `FFunctionImplementation` | FFunctionImplementation.cs | FRegisterFunction.cpp:352-377 | 25 | 25 | 0 | 0 | 0 |
| 7 | `TLazyObjectPtrImplementation` | TLazyObjectPtrImplementation.cs | FRegisterLazyObjectPtr.cpp:57-61 | 4 | 4 | 0 | 0 | 0 |
| 8 | `TMapImplementation` | TMapImplementation.cs | FRegisterMap.cpp:187-203 | 16 | 16 | 0 | 0 | 0 |
| 9 | `FMulticastDelegateImplementation` | FMulticastDelegateImplementation.cs | FRegisterMulticastDelegate.cpp:187-201 | 13 | 14 | 0 | **1** | 0 |
| 10 | `FNameImplementation` | FNameImplementation.cs | FRegisterName.cpp:63-68 | 5 | 5 | 0 | 0 | 0 |
| 11 | `UObjectImplementation` | ObjectImplementation.cs | FRegisterObject.cpp:133-145 | 11 | 12 | 0 | **1** | 0 |
| 12 | `TOptionalImplementation` | TOptionalImplementation.cs | FRegisterOptional.cpp:146-154 | 8 | 8 | 0 | 0 | 0 |
| 13 | `FPropertyImplementation` | FPropertyImplementation.cs | FRegisterProperty.cpp:72-76 | 4 | 4 | 0 | 0 | 0 |
| 14 | `TScriptInterfaceImplementation` | TScriptInterfaceImplementation.cs | FRegisterScriptInterface.cpp:72-76 | 4 | 4 | 0 | 0 | 0 |
| 15 | `TSetImplementation` | TSetImplementation.cs | FRegisterSet.cpp:118-129 | 11 | 11 | 0 | 0 | 0 |
| 16 | `TSoftClassPtrImplementation` | TSoftClassPtrImplementation.cs | FRegisterSoftClassPtr.cpp:65-70 | 5 | 5 | 0 | 0 | 0 |
| 17 | `TSoftObjectPtrImplementation` | TSoftObjectPtrImplementation.cs | FRegisterSoftObjectPtr.cpp:65-70 | 5 | 5 | 0 | 0 | 0 |
| 18 | `FStringImplementation` | FStringImplementation.cs | FRegisterString.cpp:51-55 | 4 | 4 | 0 | 0 | 0 |
| 19 | `UStructImplementation` | StructImplementation.cs | FRegisterStruct.cpp:57-61 | 4 | 4 | 0 | 0 | 0 |
| 20 | `TSubclassOfImplementation` | TSubclassOfImplementation.cs | FRegisterSubclassOf.cpp:56-60 | 4 | 4 | 0 | 0 | 0 |
| 21 | `FTextImplementation` | FTextImplementation.cs | FRegisterText.cpp:77-81 | 4 | 4 | 0 | 0 | 0 |
| 22 | `FUtf8StringImplementation` | FUtf8StringImplementation.cs | FRegisterUtf8String.cpp:55-59 | 4 | 4 | 0 | 0 | 0 |
| 23 | `UnrealImplementation` | UnrealImplementation.cs | FRegisterUnreal.cpp:180-187 | 7 | 7 | 0 | 0 | 0 |
| 24 | `TWeakObjectPtrImplementation` | TWeakObjectPtrImplementation.cs | FRegisterWeakObjectPtr.cpp:57-61 | 4 | 4 | 0 | 0 | 0 |
| 25 | `FActorSpawnParametersImplementation` | FActorSpawnParametersImplementation.cs | FRegisterWorld.cpp:96-112 | 6 | 6 | 0 | 0 | 0 |
| 26 | `UWorldImplementation` | WorldImplementation.cs | FRegisterWorld.cpp:149-150 | 1 | 1 | 0 | 0 | 0 |
| 27 | `UEnhancedInputComponentImplementation` | EnhancedInputComponentImplementation.cs | FRegisterEnhancedInputComponent.cpp:219-222 | 3 | 3 | 0 | 0 | 0 |
| 28 | `UInputComponentImplementation` | InputComponentImplementation.cs | FRegisterInputComponent.cpp:332-340 | 8 | 8 | 0 | 0 | 0 |
| | | | **合计** | **211** | **214** | **0** | **3** | **0** |

### 3.2 闭合验证（本节核心证据）——**三方对账**

> 此处有一段有缺陷的算术：「214 − 2 个 `BINDING_FUNCTION` = 212 个有手写 impl 的 key，C# 手写声明 211，
> 214 − 211 = 3 个 key 无 C# 声明」。**这两步自相矛盾**（若真有 212 个 key 有手写 impl，则无 C# 声明的只可能是 2 个），
> 且"无 C# 声明"的判定用了**错误的范围**。下面是用 `grep`/`read`/`glob` 工具核对后的结果。

```
【第一方】C++ 注册侧：LIBRARY 命名空间 .Function() 调用点
    逐块 read 28 个 NAMESPACE_LIBRARY 块后手工求和                    = 214
    （Array29 Delegate16 AnsiString4 Function25 InputComponent8 DataTable1 Class2
      EnhancedInput3 LazyObjectPtr4 Map16 MulticastDelegate14 Name5 Object12 Optional8
      Property4 ScriptInterface4 Set11 SpawnParams6 World1 WeakObjectPtr4 Utf8String4
      SubclassOf4 Unreal7 String4 SoftClassPtr5 Struct4 SoftObjectPtr5 Text4）
    经 FClassBuilder.cpp:83-91 去重加后缀后 → 214 个互不冲突的注册 key

【第二方】C# 实现侧（插件手写）：Script/UE/Library/ 28 个文件
    grep 'private static unsafe partial'                              = 211   （不截断，逐文件与 §0.1 表吻合）

【第三方】C# 实现侧（项目生成产物）：Script/UE/Proxy/Binding/ 的 LIBRARY 命名空间部分
    ClassImplementation.cs:13   __UClass_GetDefaultObjectImplementation            = 1
    ObjectImplementation.cs:13  __UObject_GetWorldImplementation                   = 1
    grep '__UClass_GetDefaultObjectImplementation|__UObject_GetWorldImplementation' = 4 命中
        （2 个声明 + 2 个真实调用点：Class.cs:23、Object.cs:20）

C# 声明槽总数 = 211 + 2                                          = 213
214 − 213                                                        = 1 个 key 无任何 C# 对端
```

**结论（三条）**：
1. **方向 ① = 0**：211 + 2 = 213 个 C# 声明**全部**在 C++ 侧有同名注册，不存在"声明了但没注册"的空函数指针风险。
2. **方向 ② = 1（不是 3）**：唯一无 C# 对端的 key 是
   `Script.Library.FMulticastDelegateImplementation::__FMulticastDelegate_Contains1Implementation`
   （`FRegisterMulticastDelegate.cpp:192` 的重复注册，见 F-CS3-009）。全项目 `Script/` 树 `grep 'Contains1Implementation'` → **0 命中**。
3. **方向 ③ = 0（LIBRARY 命名空间）**：214 个调用点的 `X == Y`（去 `Implementation` 后缀）无一例外；2 个宏形式的目标符号也同名。

> **对账口径说明**：这里刻意**不用减法**得到 214（734 是 `948 − 214` 的减法结果，可接受；
> 但 214 本身必须是实测值，否则方向 ① 的结论就建立在减法误差上）。214 是 28 个块逐个读出来相加的。

### 3.3 方向 ② 的条目（**更正：3 → 1**）

> 此前的判定依据是「`GetDefaultObjectImplementation` / `GetWorldImplementation` 在整个 `Script/` 树命中 **0**」，
> 其中 `Script\*\*\*.cs` 指的是**插件自身的 `Script/` 树**。这是**范围错误**：这两个符号的声明与调用方都在
> **项目侧** `<Project>/Script\UE\Proxy\Binding\`（不在插件仓库内，§0.3 自己已说明）。

| # | 注册 key | C++ 位置 | 实测证据 | 判定 |
|---|---|---|---|---|
| 1 | `Script.Library.UClassImplementation::__UClass_GetDefaultObjectImplementation` | `FRegisterClass.cpp:48-50` | **有 C# 对端且被真实调用**：`Script/UE/Proxy/Binding/ClassImplementation.cs:13`（`namespace Script.Library`，`:11` 的 `UClassImplementation`）声明；`Script/UE/Proxy/Binding/Class.cs:23` 调用 | **非死注册（原判撤销）** |
| 2 | `Script.Library.UObjectImplementation::__UObject_GetWorldImplementation` | `FRegisterObject.cpp:138` | **有 C# 对端且被真实调用**：`Proxy/Binding/ObjectImplementation.cs:13` 声明；`Proxy/Binding/Object.cs:20` 调用 | **非死注册（原判撤销）** |
| 3 | `Script.Library.FMulticastDelegateImplementation::__FMulticastDelegate_Contains1Implementation` | `FRegisterMulticastDelegate.cpp:192` | 全项目 `Script/` 树 `grep 'Contains1Implementation'` → **0 命中**（确证无对端）；`:190` 与 `:192` 名字与目标完全相同，经后缀去重成为 `Contains`/`Contains1` | **唯一真实的死 key（P3，F-CS3-009）** |

复现命令（实际执行，`grep` 工具，范围 = **项目侧** `Script/`）：
- `grep -n '__UClass_GetDefaultObjectImplementation\|__UObject_GetWorldImplementation' Script/**/*.cs` → **4 命中**（2 声明 + 2 调用）
- `grep -n 'Contains1Implementation' Script/**/*.cs` → **0 命中**
- 对照：同一模式在**插件侧** `Plugins/UnrealCSharp/Script/**` → **0 命中**（这解释了为什么会得出"死代码"）

> **`GetWorld` 的原始担忧不成立**：`UObject::GetWorld` 的 C# 公开包装就是生成产物
> `Proxy/Binding/Object.cs:20`（`UObjectImplementation.__UObject_GetWorldImplementation(...)`），
> 它是**活的、已绑定的**。"看到 `.Function("GetWorld", ...)` 会让人以为已绑定，当前确为纯死代码"**属误判**。

### 3.4 方向 ③ = 0（LIBRARY 命名空间）

对 214 个 LIBRARY 调用点逐个看 `.Function("X", YImplementation)`：**每一个 `X` 都等于 `Y`**（去掉 `Implementation` 后缀），
即 LIBRARY 命名空间的名字与目标符号按构造成立。除外的 2 个 `BINDING_FUNCTION` 形式其名字与真实目标一致
（`GetDefaultObject`→`&UClass::GetDefaultObject`，`GetWorld`→`&UObject::GetWorld`）。
**全部名字↔目标错配都集中在 `NAMESPACE_BINDING`**，见 §5。

---

## 4. 逐函数 ABI 对照表（代表性抽样）

横向约定见 §1.4。以下为逐条读过的实现签名 ↔ C# 声明对照（`IManagedHandle`≡`nint`、`uint8`≡`byte`、`int32`≡`int`、`uint8*`≡`byte*`、`const char*`≡`byte*`）：

| # | C# 声明（文件:行） | C++ 实现（文件:行） | 参数对照 | 结论 |
|---|---|---|---|---|
| 1 | FStringImplementation.cs:9 `__FString_RegisterImplementation(nint, byte*)` | FRegisterString.cpp:13 `(const IManagedHandle, const char*)` | 2↔2 | 一致 |
| 2 | FStringImplementation.cs:21 → `byte` | FRegisterString.cpp:21 → `uint8` | 2↔2，返回 `byte`↔`uint8` | 一致 |
| 3 | FStringImplementation.cs:28 `(nint)` | FRegisterString.cpp:34 | 1↔1 | 一致 |
| 4 | FStringImplementation.cs:35 → `nint` | FRegisterString.cpp:42 → `IManagedHandle` | 返回 | 一致 |
| 5 | ObjectImplementation.cs:9 `(nint, nint)` → `byte` | FRegisterObject.cpp:16 → `uint8` | | 一致 |
| 6 | ObjectImplementation.cs:16 `(byte*)` → `nint` | FRegisterObject.cpp:29 `(const char*)` → `IManagedHandle` | | 一致 |
| 7 | ObjectImplementation.cs:32/41 `(nint)` → `nint` | FRegisterObject.cpp:38/50 | | 一致 |
| 8 | ObjectImplementation.cs:50/57 → `byte` | FRegisterObject.cpp:62/72 → `uint8` | | 一致 |
| 9 | ObjectImplementation.cs:64/71 `(nint)` → `void` | FRegisterObject.cpp:85/93 | | 一致 |
| 10 | ObjectImplementation.cs:78/85/92 → `byte` | FRegisterObject.cpp:101/111/121 → `uint8` | | 一致 |
| 11 | ClassImplementation.cs:7 `(nint, nint)` | FRegisterClass.cpp:12 | 2↔2 | 一致 |
| 12 | DataTableFunctionLibraryImplementation.cs:7 `(nint, nint, nint*)` → `byte` | FRegisterDataTableFunctionLibrary.cpp:7 | `nint*`↔`IManagedHandle*` | 一致 |
| 13 | TArrayImplementation.cs:9 `(nint, nint)` | FRegisterArray.cpp:14 | 2↔2 | 一致 |
| 14 | TArrayImplementation.cs:30/37/51/65 → `int` | FRegisterArray.cpp:43/54/76/98 → `int32` | | 一致 |
| 15 | TArrayImplementation.cs:44 `(nint, int)` → `byte` | FRegisterArray.cpp:65 → `uint8` | | 一致 |
| 16 | TArrayImplementation.cs:86 `(nint, byte*)` → `int` | FRegisterArray.cpp:131 `(const IManagedHandle, IN_VALUE_BUFFER_SIGNATURE)` → `int32` | `byte*`↔`uint8*` | 一致 |
| 17 | TArrayImplementation.cs:128 `(nint, int, int, byte)` | FRegisterArray.cpp:195 | 4↔4，`byte`↔`uint8` | 一致 |
| 18 | TArrayImplementation.cs:208 `()` → `int` | FRegisterArray.cpp:308 → `int32` | 0 参 | 一致 |
| 19 | TMapImplementation.cs:9/16 `(nint, nint)`/`(nint)` | FRegisterMap.cpp:13/23 | | 一致 |
| 20 | TMapImplementation.cs:58 `(nint, byte*, byte*)` | FRegisterMap.cpp:63 `IN_KEY_BUFFER_SIGNATURE, IN_VALUE_BUFFER_SIGNATURE` | | 一致 |
| 21 | TMapImplementation.cs:107/114 `(nint, int, byte*)` | FRegisterMap.cpp:161/173 | | 一致 |
| 22 | FPropertyImplementation.cs:7 `(nint, uint, byte*)` | FRegisterProperty.cpp:10 | `uint`↔`uint32` | 一致 |
| 23 | FPropertyImplementation.cs:15 `(nint, uint, byte*)` | FRegisterProperty.cpp:26 | | 一致 |
| 24 | FPropertyImplementation.cs:23/31 | FRegisterProperty.cpp:40/55 | | 一致 |
| 25 | StructImplementation.cs:9 `(byte*)` → `nint` | FRegisterStruct.cpp:13 `(const char*)` → `IManagedHandle` | | 一致 |
| 26 | StructImplementation.cs:25 `(nint, byte*)` | FRegisterStruct.cpp:22 | | 一致 |
| 27 | StructImplementation.cs:37 `(nint, nint, nint)` → `byte` | FRegisterStruct.cpp:29 → `uint8` | | 一致 |
| 28 | StructImplementation.cs:44 `(nint)` | FRegisterStruct.cpp:47 | | 一致 |
| 29 | UnrealImplementation.cs:9 `(nint, nint, nint, EObjectFlags, nint, byte)` → `nint` | FRegisterUnreal.cpp:45 | 6↔6，`byte`↔`uint8` | 一致 |
| 30 | UnrealImplementation.cs:20 `(nint, nint, nint)` | FRegisterUnreal.cpp:70 | | 一致 |
| 31 | UnrealImplementation.cs:29/39 `(nint, nint, nint, ELoadFlags, nint)` | FRegisterUnreal.cpp:87/111 | 5↔5 | 一致 |
| 32 | UnrealImplementation.cs:49 `(nint, nint)` | FRegisterUnreal.cpp:135 | | 一致 |
| 33 | UnrealImplementation.cs:58/67 `()` → `nint` | FRegisterUnreal.cpp:168/173 | | 一致 |
| 34 | FActorSpawnParametersImplementation.cs:7/14/21/28/35/42 | FRegisterWorld.cpp:107-112 | `byte`↔`uint8` 入参与返回 | 一致 |
| 35 | WorldImplementation.cs:8 `(nint, nint, nint, nint)` → `nint` | FRegisterWorld.cpp:150 | 4↔4 | 一致 |

> **这 35 条（覆盖 §6 指定的全部 8 组）未发现任何 `bool` 宽度、`int`/`nint` 截断、`float`/`double` 混用、
> `int64`/`int32` 混用或参数个数/顺序不一致的问题。** 任务书给的"已知校正"在这 8 组上成立，未找到例外。
> 字符编码方面：`FString`/`FAnsiString`/`FUtf8String`/`FName` 的 `RegisterImplementation` 统一收 `const char*`，
> 由 C# 侧自行 UTF-8 编码（无 `LPWStr`/`LPUTF8Str` 误用）。
>
> **未核对的**：多行签名的续行部分（`FRegisterArray.cpp:109,121,175,185,195,223,288,298`、
> `FRegisterMap.cpp:63,73,85,96,118,129,161,173`、`FRegisterProperty.cpp:10,26,40,55`、
> `FRegisterUnreal.cpp:45,70,87,111,135`、`FRegisterStruct.cpp:29`）只读了签名首行与关键类型。

---

## 5. `FRegister*.cpp` 名字 ↔ 目标错配全量扫描表

### 5.1 扫描方法（可复现）

对全部 58 个 `FRegister*.cpp` 的 **948 个 `.Function(` 调用点**做括号配平解析，取每个调用的完整实参串，然后：

1. 若第一个实参不是纯标识符字符串（`^[A-Za-z_]\w*$`），跳过（63 个，全是 `"operator +"`、`"~FXxx"` 之类的运算符/构造名）；
2. 否则取该串里所有 `&Class::Method`，**当且仅当** `Method != 名字` 且 `Method != 名字 + "Implementation"` 时报警。

另做一次**跨类检查**：把每个 builder 块的目标类与 builder 模板实参比对——
**58 个文件的 builder 类型集合与目标类集合完全一致，0 例跨类错绑**
（例：FRegisterVector.cpp `builder[FVector] target[FVector]`；FRegisterEnhancedInputComponent.cpp
`builder[FEnhancedInputActionEventBinding,FInputBindingHandle,UEnhancedInputComponent] target[FEnhancedInputActionEventBinding,FInputBindingHandle]`
——LIBRARY 块用手写 impl 故无 `&C::Y`）。

### 5.2 扫描结果：948 → 6 例错配

| # | 文件:行 | 注册名 X | 目标符号 C::Y | X 是否重名出现 | 真实性质 | 发现编号 |
|---|---|---|---|---|---|---|
| 1 | `FRegisterRotator.cpp:46` | `Equals` | `FRotator::IsNearlyZero` | 否（无 `Equals1`） | **语义错配（真 bug）** | **F-CS3-002 (P1)** |
| 2 | `FRegisterRotator.cpp:65` | `SetComponentForAxis` | `FRotator::GetComponentForAxis` | 否（无 `SetComponentForAxis1`） | **setter 实为 getter（真 bug）** | **F-CS3-003 (P1)** |
| 3 | `FRegisterSoftObjectPath.cpp:34` | `GetAssetPathName` | `FSoftObjectPath::GetAssetName` | 否（`:47` 另有 `GetAssetName` 注册，名字不同） | **语义错配（真 bug）** | **F-CS3-004 (P1)** |
| 4 | `FRegisterMatrix.cpp:78` | `GetColumn` | `FMatrix::SetColumn` | 是（`:76` 也有 `GetColumn`） | 目标正确，**公开 API 命名错误** | F-CS3-005 (P2) |
| 5 | `FRegisterPolyglotTextData.cpp:19` | `SetCategory` | `FPolyglotTextData::SetNativeCulture` | 是（`:16` 也有 `SetCategory`） | 目标正确，**公开 API 命名错误** | F-CS3-005 (P2) |
| 6 | `FRegisterVector.cpp:163` | `Dot` | `FVector::DotProduct` | 是（`:161` 也有 `Dot`） | 目标正确，**公开 API 命名错误** | F-CS3-005 (P2) |

**这 6 例与任务书提到的"别的 `FRegister*.cpp` 里已找到的 6 例"数量吻合**，
其中 `FRegisterRotator.cpp:46` 完全对应任务书给出的实例，故本报告的机械扫描规则被独立复现验证。

### 5.3 **关键机制更正：同名重复注册不会互相覆盖**（推翻"后者覆盖前者"的初始假设）

"同一名字注册两次时后者覆盖前者"这一推断会把 §5.2 的 4/5/6 号判为 P0 级内存损坏，下面的证据推翻了它。
**这个推断是错误的**，证据是 `FClassBuilder::GetFunctionImplementationName`：

```cpp
// Source/UnrealCSharp/Private/Binding/Class/FClassBuilder.cpp:83-91   ← 实读
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
```cpp
// FClassBuilder.cpp:67-78（调用点）
	const auto FunctionImplementationName = GetFunctionImplementationName(InName, InImplementationName);

	Functions.Add(InName);          // 登记本次名字，供后续 Count 使用

	Function(FunctionImplementationName, TFunctionPointer<decltype(InMethod)>(InMethod));
```

即：**同名第 k 次注册（0-based）会自动追加数字后缀 `k`**，形成两个**互不冲突**的注册 key。
生成端完全对得上（实读生成产物取证）：

| C++ 注册 | 实际实现的 key 后缀 | 生成/手写的 C# 对端 | 证据 |
|---|---|---|---|
| `FRegisterMatrix.cpp:76` `"GetColumn"`→`&FMatrix::GetColumn` | 无后缀 | `__FMatrix_GetColumnImplementation` | `MatrixImplementation.cs:104` |
| `FRegisterMatrix.cpp:78` `"GetColumn"`→`&FMatrix::SetColumn` | `1` | `__FMatrix_GetColumn1Implementation` | `MatrixImplementation.cs:106` |
| `FRegisterVector.cpp:161` `"Dot"`→`&FVector::Dot` | 无后缀 | `__FVector_DotImplementation` | `VectorImplementation.cs:132` |
| `FRegisterVector.cpp:163` `"Dot"`→`&FVector::DotProduct` | `1` | `__FVector_Dot1Implementation` | `VectorImplementation.cs:134` |
| `FRegisterPolyglotTextData.cpp:16` `"SetCategory"`→`SetCategory` | 无后缀 | `__FPolyglotTextData_SetCategoryImplementation` | `PolyglotTextDataImplementation.cs:20` |
| `FRegisterPolyglotTextData.cpp:19` `"SetCategory"`→`SetNativeCulture` | `1` | `__FPolyglotTextData_SetCategory1Implementation` | `PolyglotTextDataImplementation.cs:24` |

**后果**：
1. §5.2 的 4/5/6 号**不是内存损坏**，而是**公开 API 被取了错误的名字**（P2，见 F-CS3-005）：
   用户看到的 C# 成员名来自**注册名 X**，而执行的函数由**目标 Y** 决定，两者不符时名字具有误导性。
2. §3.2 的算术必须按"214 个唯一 key"而不是"213 个唯一名字"来算（已在 §3.2 更正）。
3. 反向启示：**作者本来有去重机制兜底**，所以 6 例错配不是"重名覆盖"造成的，
   而是**纯粹的名字写错**（把相邻两行中的一个名字漏改了）。这类错误没有任何机制能兜住。

### 5.4 三个真 bug 的完整证据链

**F-CS3-002（`Equals` → `IsNearlyZero`）**
```cpp
// FRegisterRotator.cpp:43-47
				.Function("IsNearlyZero", BINDING_FUNCTION(&FRotator::IsNearlyZero,
				                                           TArray<FString>{"Tolerance"}, KINDA_SMALL_NUMBER))
				.Function("IsZero", BINDING_FUNCTION(&FRotator::IsZero))
				.Function("Equals", BINDING_FUNCTION(&FRotator::IsNearlyZero,
				                                     TArray<FString>{"R", "Tolerance"}, KINDA_SMALL_NUMBER))
```
```csharp
// Script/UE/Proxy/Binding/RotatorImplementation.cs:55（实读；grep `__FRotator_Equals` 只命中这一行 → 无 Equals1）
public static unsafe partial void __FRotator_EqualsImplementation(nint InObject, byte* InBuffer, byte* OutBuffer, byte* ReturnBuffer);
```
```csharp
// Script/UE/Proxy/Binding/Rotator.cs:196-210（实读）
public bool Equals(double Tolerance = 0.0001)
{
    unsafe
    {
        var InBuffer = stackalloc byte[8];
        *(double*)(InBuffer) = Tolerance;
        var ReturnBuffer = stackalloc byte[1];
        FRotatorImplementation.__FRotator_EqualsImplementation(HandleData.GetHandle(this), InBuffer, null, ReturnBuffer);
        return *(bool*)ReturnBuffer;
    }
}
```
参数名元数据 `{"R","Tolerance"}` 正是 `FRotator::Equals(const FRotator& R, FReal Tolerance)` 的签名
（相邻 `:43` 的 `IsNearlyZero` 用的是 `{"Tolerance"}`）→ **本意是 `&FRotator::Equals`，复制粘贴漏改**。
生成的 C# 只有 1 个参数（生成器取的是真实目标的元数），调用方**无法传入被比较的 rotator**。

**F-CS3-003（`SetComponentForAxis` → `GetComponentForAxis`）**
```cpp
// FRegisterRotator.cpp:63-66
				.Function("GetComponentForAxis", BINDING_FUNCTION(&FRotator::GetComponentForAxis,
				                                                  TArray<FString>{"Axis"}))
				.Function("SetComponentForAxis", BINDING_FUNCTION(&FRotator::GetComponentForAxis,
				                                                  TArray<FString>{"Axis", "Component"}))
```
两条注册**都**指向 getter（名字不同故都保留为独立 key）。`RotatorImplementation.cs:81` 只有
`__FRotator_SetComponentForAxisImplementation`（无 `SetComponentForAxis1`）。→ setter 执行的是 getter，返回值被丢弃。

**F-CS3-004（`GetAssetPathName` → `GetAssetName`）**
```cpp
// FRegisterSoftObjectPath.cpp:34
				.Function("GetAssetPathName", BINDING_FUNCTION(&FSoftObjectPath::GetAssetName))
...
// :47
				.Function("GetAssetName", BINDING_FUNCTION(&FSoftObjectPath::GetAssetName))
```
```csharp
// Script/UE/Proxy/Binding/SoftObjectPath.cs:71-77（实读）
public FString GetAssetPathName()
{ ... SoftObjectPathImplementation.__FSoftObjectPath_GetAssetPathNameImplementation(HandleData.GetHandle(this), null, null, ReturnBuffer); ... }
// :131-137
public FString GetAssetName()
{ ... SoftObjectPathImplementation.__FSoftObjectPath_GetAssetNameImplementation(HandleData.GetHandle(this), null, null, ReturnBuffer); ... }
```
`SoftObjectPathImplementation.cs:36` 只有 `__FSoftObjectPath_GetAssetPathNameImplementation`（无 `1`）。
→ 名为 `GetAssetPathName` 的 C# 成员返回的是 **asset name**，不是 asset path。

---

## 6. 8 组逐参数对照表

已在 §4 以 35 条逐函数对照完成，覆盖任务指定的 8 组全部：
`FStringImplementation`↔`FRegisterString`（#1-4）、`ObjectImplementation`↔`FRegisterObject`（#5-10）、
`ClassImplementation`↔`FRegisterClass`（#11）、`TArrayImplementation`↔`FRegisterArray`（#13-18）、
`TMapImplementation`↔`FRegisterMap`（#19-21）、`FPropertyImplementation`↔`FRegisterProperty`（#22-24）、
`UnrealImplementation`↔`FRegisterUnreal`（#29-33）、`StructImplementation`↔`FRegisterStruct`（#25-28）。

**结论：8 组 35 条对照，0 例参数/类型不一致。** 未找到 `bool` 宽度、`int`/`nint` 截断、`float`/`double` 混用、
`int64`/`int32` 混用、参数个数/顺序错误的例外。缓冲区宏 `IN/OUT/RETURN/IN_KEY/IN_VALUE_BUFFER_SIGNATURE`
在 `CoreMacro/BufferMacro.h:9,15,21,25,29` 全部展开为 `uint8*`，与 C# 的 `byte*` 一致。

> **优先级 2 的净收益是"排除了嫌疑"**：本模块的 ABI 宽度/编解码层面是健康的，真正的缺陷在**名字层**。

---

## 7. 枚举与常量对账（优先级 3，部分完成）

| 项 | 状态 |
|---|---|
| `UnrealImplementation.cs` 的常量/枚举 ↔ `FRegisterUnreal.cpp` | **7 个函数全部一一对应，0 拼写错误**（§3 组 23、§4 #29-33） |
| `FNameImplementation.cs:44` `__FName_NAME_NoneImplementation` ↔ `FRegisterName.cpp:68` `.Function("NAME_None", NAME_NoneImplementation)` | 一致（下划线大小写完全相符） |
| `TArrayImplementation.cs:208` `INDEX_NONE` ↔ `FRegisterArray.cpp:344` | 一致 |
| `TOptionalImplementation.cs:10,17` `Register1`/`Register2` ↔ `FRegisterOptional.cpp:147-148` | 一致（唯一自带数字后缀的类——**注意**：这里的 `1`/`2` 是作者手写的名字，与 §5.3 的自动后缀是两套机制，恰好不冲突） |
| 枚举入参 `EObjectFlags`/`ELoadFlags`（UnrealImplementation.cs:9,29） | C# 侧用了与 C++ 同名的枚举类型；**未**核对底层宽度在三个后端下的实际封送 |
| 其他文件 `enum` 成员名 ↔ C++ 定义 | **未核对**（见 §11） |

---

## 8. 健壮性与性能（优先级 4，部分完成）

| 检查项 | 结果 |
|---|---|
| `MethodBridge.GetMethod` 负结果缓存 | **不缓存**（`:142-145` 查不到时 `InSlot` 仍为 0，下次再查）→ F-CS3-008 |
| `MethodBridge.Invoke` 异常处理 | `:125-137` 捕获所有异常写 stderr 并返回 0；**但 `GetMethod` 返回 0 的调用路径不在此保护内**（崩溃发生在 C# 调用点而非 Invoke 内） |
| `RegisterBinding` 空值检查 | `:15,19,21` 对 `InNames`/`InMethods`/NULL/0 均有检查 —— **健壮** |
| 重复名防护 | `FClassBuilder.cpp:83-91` 自动后缀去重 —— **健壮**（但代价是名字错配永不被发现，见 F-CS3-007） |
| C# 侧 `IntPtr.Zero` 入参检查 | `ObjectImplementation.cs:16-30` 对句柄有处理；多数声明把 `nint` 直接透传，由 C++ 侧 `GetObject<>()` 判空（`FRegisterClass.cpp:14-15` 有判空） |
| 长度参数被信任 | `TArrayImplementation.cs:9-15` 把 C++ 侧数组指针/长度整体交给 native（`FRegisterArray.cpp:14`），C# 侧信任 C++ 提供长度，**未做越界自检** |
| 每次调用的 `Marshal.AllocHGlobal`/`PtrToString*` | `MethodBridge.cs:21` 每次 `RegisterBinding` 用 `Marshal.PtrToStringUTF8`（仅启动一次，可接受） |
| 可批量却逐元素调用 | **未核对**（`TArrayImplementation.cs:72-100` 的 `Get`/`Set`/`Find` 每元素一次 native 往返是主要嫌疑） |

---

## 9. 发现清单

### P0

#### [F-CS3-001] `MethodBridge.GetMethod` 返回 0 后生成代码不判空，任何名字不一致都会 `call 0` 崩溃

- **类别**: Bug / 未定义行为 / 平台兼容
- **严重度**: **P1**（崩溃）
- **复核结论**: **部分确认**（偏差：机制确认，但"当前就会崩"的前提不成立 —— 该代码路径在当前工程**不参与编译**）
- **可达性**: **潜伏**（当前工程五平台全部 LeanCLR → 生成器走 `UnrealTypeSourceGenerator.cs:910-912` 的 `[DllImport]` 分支，
  **字面上不经过 `MethodBridge.GetMethod`**；切回 Mono/CoreCLR 立即生效）
- **复核证据**: `Script/Interop/Bridge/MethodBridge.cs:140-148`（实读，所引内容逐字一致：`InSlot == nint.Zero` 同时表示"未查过/查不到"）；
  key 拼装 `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:903` + `Source/UnrealCSharpCore/Public/CoreMacro/BindingMacro.h:3-9`
  + `Source/UnrealCSharpCore/Private/Binding/Class/FBindingClassRegister.cpp:113-129`（三处实读，key 规则一致）；
  无判空强转模板 `UnrealTypeSourceGenerator.cs:913-918`；`WITH_LEANCLR` 分支 `:910-912`
- **级别变动**: 无（P0 → P1）
- **判定**: **P1 / 潜伏**。理由：① 生成代码确实无条件把 `GetMethod` 返回值当函数指针调用，机制成立；
  ② 但本工程 100% LeanCLR 已核实，`GetMethod` 不在编译产物内；③ 一旦切后端，本条即从"潜伏"变为实机崩溃
  （CoreCLR 下 `call 0` 会 `TerminateProcess`，`try/catch` 无效），故**不能降到 P3**。
  **未验证的假设（置信度因此为"中"）**：LeanCLR 分支的 `[DllImport("UnrealCSharp", Cdecl)]` 究竟如何在宿主里解析——
  插件 `Script/` 树内 `grep 'DllImportResolver|Bridge_Invoke|NativeLibrary'` → 0 命中，LeanCLR 源码 `grep 'UnrealCSharp'` → 0 命中，
  故**无法证明"LeanCLR 下名字不一致也不会崩"**；只能说"不经过本条的崩溃点"。
- **文件**: `Script/Interop/Bridge/MethodBridge.cs:140`；生成器模板 `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:918-919`
- **函数**: `Interop.MethodBridge.GetMethod(ref nint InSlot, string InName)`
- **置信度**: 高

**现状（代码事实）**
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
```csharp
// Script/SourceGenerator/UnrealTypeSourceGenerator.cs:918-919（生成器输出模板）
            ((delegate* unmanaged[Cdecl]<nint, byte*, void>)global::Interop.MethodBridge.GetMethod(
                ref __FString_RegisterImplementation_Slot, "Script.Library.FStringImplementation::__FString_RegisterImplementation"))(InString, InValue);
```

**调用上下文**
`Script/UE/Library/*Implementation.cs` 的 211 个手写声明 + `Script/UE/Proxy/Binding/*Implementation.cs`
的全部生成声明都经此函数取指针。注册一次性发生在 `FScriptDomainImpl.inl:631` → `MethodBridge.cs:13`。

**问题**
`GetMethod` 用 `nint.Zero` 同时表示"还没查过"和"查不到"。生成器把返回值**无条件**强转成函数指针并立即调用，
不做非空判断。C++ 侧 key 由 `BindingMacro.h:3-9` 宏拼装、C# 侧 key 由 `UnrealTypeSourceGenerator.cs:903` 拼装，
二者之间**没有任何编译期或链接期校验**。任一侧拼错一个字符（或 C++ 注册被改名而生成产物未重新生成），
都会在**第一次调用**时对地址 0 执行 `call` → 访问冲突崩溃，且崩溃点与真正原因（名字不一致）相隔很远，极难定位。

本报告 §3.2 证明当前 LIBRARY 命名空间恰好闭合（214−211=3，且 3 个都已定位），但**这个保证是偶然的**：
§5 已证明 `NAMESPACE_BINDING` 侧确实存在名字/语义错配（3 例真 bug），而唯一防线就是这条无校验的字符串。
另注意 `FClassBuilder.cpp:83-91` 的自动后缀会**掩盖**重名，使"两个不同函数共用一个公开名字"永远不报错（F-CS3-005）。

**建议**
最低成本加固（不改 ABI）：
```csharp
public static nint GetMethod(ref nint InSlot, string InName)
{
    if (InSlot == nint.Zero)
    {
        if (!StringToMethod.TryGetValue(InName, out var Method))
        {
            Console.Error.WriteLine($"[MethodBridge] unresolved bridge method: {InName}");
#if DEBUG
            throw new MissingMethodException(InName);
#endif
            return nint.Zero;
        }
        InSlot = Method;
    }
    return InSlot;
}
```
并用哨兵（如缓存 `-1`）让失败只报一次（同时解决 F-CS3-008）。
更彻底（推荐）：`RegisterBinding` 完成后做**启动期全集对账**——C++ 侧交出全部注册 key，
C# 侧反射枚举 `Script.Library`/`Script.Binding` 下所有无体 `partial` 声明，双向 diff，缺失项直接
`ensureAlways` 失败。这把 F-CS3-001 从"运行期崩溃"降级为"启动期可读报错"。

**验证方式**
在 `GetMethod` 里临时故意写错一个 key 后缀，跑一次 C# 构造函数，确认当前表现为崩溃、加固后表现为可读报错；
或在 `RegisterBinding` 末尾打印 `StringToMethod.Count` 与 C++ 侧 `MethodNames.Num()`（`FScriptDomainImpl.inl:631`）对比。

---

### P1

#### [F-CS3-002] `FRegisterRotator` 把 `Equals` 绑到 `FRotator::IsNearlyZero`，C# 的 `Equals` 变成"是否接近零"

- **类别**: Bug
- **严重度**: **P1**（静默错误）
- **复核结论**: **确认**（C++ 注册、真实目标、生成声明、公开包装四处逐行吻合）
- **可达性**: **活跃**（`FRegisterRotator.cpp:46` 无版本宏；`Rotator.cs:196` 是活的公开 API；`Rotator.cs:206` 是真实调用点）
- **复核证据**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterRotator.cpp:43-47`（实读：`:43` IsNearlyZero 用 `{"Tolerance"}`、`:46` Equals 用 `{"R","Tolerance"}` 却指向 `&FRotator::IsNearlyZero`）；
  消费端 `Script/UE/Proxy/Binding/Rotator.cs:196-210`（实读，签名确认只有 1 个 `double Tolerance` 参数）、`RotatorImplementation.cs:55`（实读，`__FRotator_EqualsImplementation` 存在且**无** `Equals1` 变体）
- **级别变动**: 无（P1 维持）
- **注**: 建议中的参照实现已核对为真——`FRegisterVector.cpp:165-166` 的 `Equals→&FVector::Equals` 用 `{"V","Tolerance"}`，写法正确 ✓
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterRotator.cpp:46-47`；消费端 `Script/UE/Proxy/Binding/Rotator.cs:196-210`、`RotatorImplementation.cs:55`
- **函数**: 注册名 `Equals` → 目标 `FRotator::IsNearlyZero(FReal Tolerance)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Private/Domain/Interop/FRegisterRotator.cpp:43-47
				.Function("IsNearlyZero", BINDING_FUNCTION(&FRotator::IsNearlyZero,
				                                           TArray<FString>{"Tolerance"}, KINDA_SMALL_NUMBER))
				.Function("IsZero", BINDING_FUNCTION(&FRotator::IsZero))
				.Function("Equals", BINDING_FUNCTION(&FRotator::IsNearlyZero,
				                                     TArray<FString>{"R", "Tolerance"}, KINDA_SMALL_NUMBER))
```
```csharp
// Script/UE/Proxy/Binding/Rotator.cs:196-210
public bool Equals(double Tolerance = 0.0001)
{
    unsafe
    {
        var InBuffer = stackalloc byte[8];
        *(double*)(InBuffer) = Tolerance;
        var ReturnBuffer = stackalloc byte[1];
        FRotatorImplementation.__FRotator_EqualsImplementation(HandleData.GetHandle(this), InBuffer, null, ReturnBuffer);
        return *(bool*)ReturnBuffer;
    }
}
```

**调用上下文**
`Rotator.cs:206` → `MethodBridge.GetMethod(ref slot, "Script.Binding.FRotatorImplementation::__FRotator_EqualsImplementation")`
→ `FRegisterRotator.cpp:46` 注册的目标 `FRotator::IsNearlyZero`。
grep `__FRotator_Equals` 在 `RotatorImplementation.cs` 只命中 `:55`（`__FRotator_Equals1Implementation` 会含该子串但未命中）
→ 该系统只有一个 `Equals` 注册，不存在重名后缀歧义。

**问题**
注册名与目标语义不符。参数名元数据 `{"R","Tolerance"}` 正是 `FRotator::Equals(const FRotator& R, FReal Tolerance)` 的签名
（相邻 `:43` 的 `IsNearlyZero` 用 `{"Tolerance"}`）→ **本意是 `&FRotator::Equals`，复制粘贴漏改**。后果：
- `FRotator::IsNearlyZero(Tolerance)` 判断 `this` 是否接近**零**；`Equals(R, Tolerance)` 判断 `this` 是否接近 **R**。
- 生成器按真实目标的元数生成了 `Equals(double Tolerance = 0.0001)` —— **只有一个参数**，
  调用方根本无法传入被比较的那个 rotator。
- 于是 `a.Equals(0.001)` 的含义从"与某值相等"变成"a 接近零"，**静默返回错误布尔值**，不崩溃、不报错，
  极易在业务逻辑里长期潜伏（如"两个 rotator 是否相等"的判断全部退化为"第一个是否约等于零"）。

**建议**
```cpp
				.Function("Equals", BINDING_FUNCTION(&FRotator::Equals,
				                                     TArray<FString>{"R", "Tolerance"}, KINDA_SMALL_NUMBER))
```
并重新生成项目侧 C#。**参照正确实现**：`FRegisterVector.cpp:165-166`
（`.Function("Equals", BINDING_FUNCTION(&FVector::Equals, TArray<FString>{"V", "Tolerance"}, KINDA_SMALL_NUMBER))`）。

**验证方式**
grep `'Function\("Equals"' FRegisterRotator.cpp` 确认目标符号；
对拍：当前实现下 `new FRotator(0,0,0).Equals(1.0)` 为 `true` 而 `new FRotator(90,0,0).Equals(90.0)` 为 `false`，即确认语义错位。

---

#### [F-CS3-003] `FRegisterRotator` 的 `SetComponentForAxis` 绑定到 getter，setter 调用被静默丢弃

- **类别**: Bug
- **严重度**: **P1**（功能完全失效且静默）
- **复核结论**: **确认**（两条注册都指向 getter，且有独立 key，无后缀合并）
- **可达性**: **活跃**
- **复核证据**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterRotator.cpp:63-66`（实读：`:63` GetComponentForAxis→`&FRotator::GetComponentForAxis`、`:65` SetComponentForAxis→**同一个** `&FRotator::GetComponentForAxis`）；
  `Script/UE/Proxy/Binding/RotatorImplementation.cs:81`（实读，`__FRotator_SetComponentForAxisImplementation`，**无** `1` 后缀 → 无重名歧义）
- **级别变动**: 无（P1 维持）
- **补充**: 因两条名字不同，§5.3 的自动后缀不会把它们合并，两条都是独立 key ✓（此点正确）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterRotator.cpp:63-66`；消费端 `RotatorImplementation.cs:81`
- **函数**: 注册名 `SetComponentForAxis` → `FRotator::GetComponentForAxis`
- **置信度**: 高

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Private/Domain/Interop/FRegisterRotator.cpp:63-66
				.Function("GetComponentForAxis", BINDING_FUNCTION(&FRotator::GetComponentForAxis,
				                                                  TArray<FString>{"Axis"}))
				.Function("SetComponentForAxis", BINDING_FUNCTION(&FRotator::GetComponentForAxis,
				                                                  TArray<FString>{"Axis", "Component"}))
```
```csharp
// Script/UE/Proxy/Binding/RotatorImplementation.cs:81（grep `SetComponentForAxis` 只命中这一行 → 无后缀歧义）
public static unsafe partial void __FRotator_SetComponentForAxisImplementation(nint InObject, byte* InBuffer, byte* OutBuffer, byte* ReturnBuffer);
```

**调用上下文**
文件为 `NAMESPACE_BINDING`，消费端是生成产物 `Script/UE/Proxy/Binding/Rotator.cs`；
同文件 `Equals` 的整条链路已实读验证（见 F-CS3-002），机制相同。

**问题**
两条注册**都**指向 getter `FRotator::GetComponentForAxis`（名字不同，故 §5.3 的自动后缀不会把它们合并，
两条都成为独立的注册 key）。后果：
- **语义**：调用 `rotator.SetComponentForAxis(Axis, Component)` 不会修改任何东西——它执行的是 getter，
  返回值被丢弃 → **setter 能力完全不存在**，而 C# 公开 API 却宣称有。用户代码编译通过、运行不报错、数值不变。
- **ABI**：注册名对应的参数元信息是 `{"Axis","Component"}`（2 个），真实目标只接受 1 个参数；
  两个不同布局的 InBuffer 打到同一个函数上。
- `FRotator` **没有** `SetComponentForAxis` 成员函数，所以不能简单改名，需要手写实现。

**建议**
按本文件已有的手写 impl 模式补一个真正的 setter：
```cpp
		static void SetComponentForAxisImplementation(FRotator& In, const EAxis::Type Axis, const FRotator::FReal Component)
		{
			In.SetComponentForAxis(Axis, Component);   // 若无此成员，则按 Axis 分支赋值 Pitch/Yaw/Roll
		}
...
				.Function("SetComponentForAxis", SetComponentForAxisImplementation)
```
（本文件 `:10-23` 的 `MultipliesImplementation` 就是同类手写先例。）
或（若 UE 确无该能力）**直接删除这条注册与对应 C# API**，避免提供一个静默无效的 setter。

**验证方式**
grep `Function\("SetComponentForAxis"` 确认唯一命中且目标是 getter；
用例：`var r = new FRotator(1,2,3); r.SetComponentForAxis(EAxis::X, 10);` 断言 `r.Pitch == 10`（当前必然失败）。

---

#### [F-CS3-004] `FRegisterSoftObjectPath` 把 `GetAssetPathName` 绑到 `GetAssetName`，C# 成员名与实际返回值不符

- **类别**: Bug
- **严重度**: **P1**（静默返回错误值）
- **复核结论**: **部分确认**（偏差：名字↔语义错位与 P1 定级**确认**，但本条原「建议」被 **UE 5.6 引擎源码证伪**、不可编译，已就地替换）
- **可达性**: **活跃**（`FRegisterSoftObjectPath.cpp:34` 无版本宏包裹，UE 5.6 下参与编译；生成产物 `SoftObjectPath.cs:71` 是活的公开 API）
- **复核证据**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterSoftObjectPath.cpp:34`（实读）`.Function("GetAssetPathName", BINDING_FUNCTION(&FSoftObjectPath::GetAssetName))`；
  同文件 `:47` `.Function("GetAssetName", BINDING_FUNCTION(&FSoftObjectPath::GetAssetName))`（实读，确为两个不同名字指向同一目标）；
  `Script/UE/Proxy/Binding/SoftObjectPathImplementation.cs:36` 只有 `__FSoftObjectPath_GetAssetPathNameImplementation`（无 `1` 后缀）；
  **UE 引擎侧（新增证据，把置信度由"中"提升为"高"）**：`Engine\Source\Runtime\CoreUObject\Public\UObject\SoftObjectPath.h` 中
  `grep 'GetAssetPathName'` → **0 命中**；存在的是 `GetAssetPathString()`（`:221`）、`GetAssetName()`（`:266`）、`GetAssetPath()`（`:209`）。
  **即 UE 5.6 根本没有 `FSoftObjectPath::GetAssetPathName`**，本条的"名字↔语义错位"确凿
- **级别变动**: 无（P1 维持）
- **原「建议」的错误（已就地更正）**: 建议"改为 `&FSoftObjectPath::GetAssetPathName` 并用 `#if UE_F_SOFT_OBJECT_PATH_GET_ASSET_PATH_NAME` 包裹"——
  两处都不成立：① UE 5.6 无该函数；② `grep 'UE_F_SOFT_OBJECT_PATH'` over `Source/CrossVersion/Public/UEVersion.h` → 8 个该前缀宏，**没有** `..._GET_ASSET_PATH_NAME`，
  照该建议改会**编译失败**。正确修法见下方新「建议」。
- **置信度**: 高（原为"中"）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterSoftObjectPath.cpp:34`；消费端 `Script/UE/Proxy/Binding/SoftObjectPath.cs:71-77`、`SoftObjectPathImplementation.cs:36`
- **函数**: 注册名 `GetAssetPathName` → `FSoftObjectPath::GetAssetName`
- **置信度**: 高（已由"中"提升：UE 头文件已核对，见上「复核证据」）

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Private/Domain/Interop/FRegisterSoftObjectPath.cpp:34
				.Function("GetAssetPathName", BINDING_FUNCTION(&FSoftObjectPath::GetAssetName))
...
// :39-47
				.Function("GetAssetPathString", BINDING_FUNCTION(&FSoftObjectPath::GetAssetPathString))
				.Function("GetSubPathString", BINDING_FUNCTION(&FSoftObjectPath::GetSubPathString))
...
				.Function("GetLongPackageName", BINDING_FUNCTION(&FSoftObjectPath::GetLongPackageName))
				.Function("GetLongPackageFName", BINDING_FUNCTION(&FSoftObjectPath::GetLongPackageFName))
				.Function("GetAssetName", BINDING_FUNCTION(&FSoftObjectPath::GetAssetName))
```
```csharp
// Script/UE/Proxy/Binding/SoftObjectPath.cs:71 / :131（实读）
public FString GetAssetPathName()   // → __FSoftObjectPath_GetAssetPathNameImplementation（SoftObjectPathImplementation.cs:36）
public FString GetAssetName()       // → __FSoftObjectPath_GetAssetNameImplementation
```

**调用上下文**
`:34` 与 `:47` 是**两个不同名字指向同一目标**（不是重名，故不涉及 §5.3 的后缀机制），
所以 `GetAssetName` 本身正确，只有 `GetAssetPathName` 错。
另注 `:36` 的 `SetAssetPathName` 存在且指向 `&FSoftObjectPath::SetAssetPathName`
（在 `UE_F_SOFT_OBJECT_PATH_SET_ASSET_PATH_NAME` 下），说明作者本意是配对 getter/setter。

**问题**
C# 公开成员 `GetAssetPathName()` 按名字应返回"资产路径名"，实际返回 `GetAssetName` 的结果（资产**名**）。
两侧 C# 可见类型都是 `FString`（生成器按真实目标推导返回类型），所以**不会崩、不会报 ABI 错**——
只是**静默返回语义不同的值**。同一文件里 `GetAssetPathString`（`:39`）、`GetLongPackageName`（`:45`）、
`GetLongPackageFName`（`:46`）三者并存，也说明这套 getter 家族本来是分开的，`GetAssetPathName` 被误接。

**建议（已重写）**
UE 5.6 的 `FSoftObjectPath` **没有** `GetAssetPathName`，所以"改成绑定真函数"这条路由不存在。二选一：
```cpp
// 方案 A（推荐，语义最清楚）：删掉误导名字，只保留正确名字
//   删除 FRegisterSoftObjectPath.cpp:34
//   （GetAssetName 由 :47 提供，GetAssetPathString 由 :39 提供，能力不缺失）
//
// 方案 B：若确实想暴露"路径名"，绑到真实存在的 GetAssetPathString
				.Function("GetAssetPathString", BINDING_FUNCTION(&FSoftObjectPath::GetAssetPathString))   // 已存在于 :39，无需新增
```
无论哪个方案都**必须重新生成项目侧 C#**（否则 `SoftObjectPath.cs:71` 仍按旧名字取指针）。
**不要**写 `#if UE_F_SOFT_OBJECT_PATH_GET_ASSET_PATH_NAME`：该宏在 `UEVersion.h` 中不存在，会编译失败。

**验证方式**
`grep 'Function("GetAssetPathName"'` 应为 0 命中（方案 A）或 ≥1 命中且目标为真实函数（方案 B）；
引擎侧已完成的核对：`SoftObjectPath.h` 中 `GetAssetPathName` 命中 0、`GetAssetName` 在 `:266`、`GetAssetPathString` 在 `:221`。

---

### P2

#### [F-CS3-005] 3 处注册名与目标函数名不符（目标正确）：公开 API 被取了误导性名字

- **类别**: Bug / 可读性 / 一致性
- **严重度**: **P2**（API 语义误导，不崩溃）
- **复核结论**: **确认**（3 组注册名/目标/自动后缀/生成成员名全部逐行吻合）
- **可达性**: **活跃**
- **复核证据**: `FRegisterMatrix.cpp:76`/`:78`（实读：GetColumn→`&FMatrix::GetColumn` / GetColumn→`&FMatrix::SetColumn`）；
  `FRegisterVector.cpp:161`/`:163`（实读：Dot→`&FVector::Dot` / Dot→`&FVector::DotProduct`）；
  去重机制 `Source/UnrealCSharp/Private/Binding/Class/FClassBuilder.cpp:83-91`（实读）+ `:67-69`（**先算 Count 再 `Functions.Add`**，故第 2 次注册得后缀 `1`——该机制描述正确）
- **级别变动**: 无（P2 维持）
- **注**: 三处均为"功能正确、名字误导"，不产生错误结果，P2 恰当（低于 F-CS3-002/003/004 的 P1）
- **文件**: `FRegisterMatrix.cpp:78`、`FRegisterPolyglotTextData.cpp:19`、`FRegisterVector.cpp:163`
- **函数**: 见下表
- **置信度**: 高（C++ 侧与生成产物两侧的注册名/后缀/C# 成员名均已实读）

**现状（代码事实）**

| C++ 位置 | 注册名 X | 目标 Y | 自动后缀 → 实现名 | 生成的 C# 公开成员 | C# 证据 |
|---|---|---|---|---|---|
| `FRegisterMatrix.cpp:76` | `GetColumn` | `&FMatrix::GetColumn` | （无） | `FVector GetColumn(int i)` | `Matrix.cs:574` → `__FMatrix_GetColumnImplementation` |
| `FRegisterMatrix.cpp:78` | `GetColumn` | `&FMatrix::SetColumn` | `1` | **`void GetColumn(int i, FVector Value)`** | `Matrix.cs:590` → `__FMatrix_GetColumn1Implementation` |
| `FRegisterPolyglotTextData.cpp:16` | `SetCategory` | `&FPolyglotTextData::SetCategory` | （无） | `void SetCategory(ELocalizedTextSourceCategory)` | `PolyglotTextData.cs:37` |
| `FRegisterPolyglotTextData.cpp:19` | `SetCategory` | `&FPolyglotTextData::SetNativeCulture` | `1` | **`void SetCategory(FString InNativeCulture)`** | `PolyglotTextData.cs:61` → `__FPolyglotTextData_SetCategory1Implementation` |
| `FRegisterVector.cpp:161` | `Dot` | `&FVector::Dot` | （无） | `double Dot(FVector V)` | `Vector.cs:781` |
| `FRegisterVector.cpp:163` | `Dot` | `&FVector::DotProduct` | `1` | **`static double Dot(FVector A, FVector B)`** | `Vector.cs:797` → `__FVector_Dot1Implementation` |

```cpp
// FRegisterPolyglotTextData.cpp:16-22
				.Function("SetCategory", BINDING_FUNCTION(&FPolyglotTextData::SetCategory,
				                                          TArray<FString>{"InCategory"}))
				.Function("GetCategory", BINDING_FUNCTION(&FPolyglotTextData::GetCategory))
				.Function("SetCategory", BINDING_FUNCTION(&FPolyglotTextData::SetNativeCulture,
				                                          TArray<FString>{"InNativeCulture"}))
				.Function("GetNativeCulture", BINDING_FUNCTION(&FPolyglotTextData::GetNativeCulture))
```

**调用上下文**
`FClassBuilder::GetFunctionImplementationName`（`FClassBuilder.cpp:83-91`）按出现次数自动加后缀，
使两条注册成为互不冲突的 key（`SetCategory` / `SetCategory1`），生成器也随之产出两个 C# 重载。

**问题**
**功能是对的，名字是错的**——因为生成的 C# 成员名取自**注册名 X**，而实际执行的函数由**目标 Y** 决定：
- `matrix.GetColumn(i, value)` 实际是 **SetColumn**（写列）；
- `polyglot.SetCategory("en")` 实际是 **SetNativeCulture**（写原生文化码）；
- `FVector.Dot(a, b)` 实际是 **DotProduct**（静态点积）。
用户通过 IntelliSense 看到的签名会诱导出错误理解。危险度低于 F-CS3-002/003/004（不产生错误结果），
但它**证明了 §5.2 那 6 例错配是一类系统性笔误**：作者每次都复制上一行再改名，漏改的那次就从"名字错"退化成了"目标错"。
对照 `FRegisterVector.cpp:157-160` 的 `Cross`/`CrossProduct` 与 `FRegisterPolyglotTextData.cpp` 已存在的
`GetNativeCulture`（`:21`），可见正确写法就在相邻行。

**建议**
```cpp
// FRegisterMatrix.cpp:78
				.Function("SetColumn", BINDING_FUNCTION(&FMatrix::SetColumn,
				                                        TArray<FString>{"i", "Value"}))
// FRegisterPolyglotTextData.cpp:19
				.Function("SetNativeCulture", BINDING_FUNCTION(&FPolyglotTextData::SetNativeCulture,
				                                               TArray<FString>{"InNativeCulture"}))
// FRegisterVector.cpp:163
				.Function("DotProduct", BINDING_FUNCTION(&FVector::DotProduct,
				                                          TArray<FString>{"A", "B"}))
```
三处都**必须重新生成**项目侧 C#（否则 C# 仍按旧后缀名取指针，直接命中 F-CS3-001 的 `call 0`）。
另建议在代码评审规则中禁止"同一 `.Function("X"` 出现两次"——这本就是 §5.2 全部 6 例的形态特征。

**验证方式**
grep 每个文件里 `Function\("GetColumn"` / `Function\("SetCategory"` / `Function\("Dot"` 应各只剩 1 次；
生成后确认 `Matrix.cs` 出现 `SetColumn`、`PolyglotTextData.cs` 出现 `SetNativeCulture`、`Vector.cs` 出现 `DotProduct`。

---

#### [F-CS3-006] 3 个 C++ 注册**声称**在 C# 侧无任何声明 —— **更正：只有 1 个成立（`Contains1`，已并入 F-CS3-009），另 2 个是活的**

- **类别**: 死代码
- **严重度**: **撤销（非缺陷）→ 3 项中 2 项撤销、1 项降为 P3 并与 F-CS3-009 合并**
- **复核结论**: **部分确认（偏差：条目数 3 → 1；判定依据被推翻）** —— 标题写的"已证伪"与正文"当前确为纯死代码"**自相矛盾**，
  以项目侧生成产物为准重新判定：`GetWorld`/`GetDefaultObject` 两项**确有 C# 声明且有真实调用方**，判定为**证伪（非缺陷）**；
  `Contains1` 一项**确认成立**，但真死 key 只有 1 个
- **可达性**: `Contains1` = 活跃（每次启动都进字典，永不被查询）；另 2 项 = 活跃且**已被调用**
- **复核证据**:
  - `Script/UE/Proxy/Binding/ObjectImplementation.cs:13` `public static unsafe partial void __UObject_GetWorldImplementation(...)`（`namespace Script.Library`，`:11`）
  - `Script/UE/Proxy/Binding/Object.cs:20` `UObjectImplementation.__UObject_GetWorldImplementation(HandleData.GetHandle(this), null, null, ReturnBuffer);`
  - `Script/UE/Proxy/Binding/ClassImplementation.cs:13` + `Class.cs:23` 同构（`__UClass_GetDefaultObjectImplementation`）
  - `grep 'Contains1Implementation'` over **项目/插件两侧** `Script/**/*.cs` → **0 命中**
  - 反证：同一模式在**插件侧** `Plugins/UnrealCSharp/Script/**` → 0 命中（该范围即导致误判）
- **级别变动**: 3 项中 2 项 → 撤销（非缺陷）；1 项 P2/P3 → 维持 P3 并并入 F-CS3-009
- **文件**: `FRegisterMulticastDelegate.cpp:192`（唯一成立的项）；~~`FRegisterClass.cpp:48-50`~~、~~`FRegisterObject.cpp:138`~~（撤销）
- **函数**: `FMulticastDelegate::Contains`（第 2 次注册）→ 死 key `Contains1`
- **置信度**: 高

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Private/Domain/Interop/FRegisterObject.cpp:133-138
			TBindingClassBuilder<UObject>(NAMESPACE_LIBRARY)
				.Function("Identical", IdenticalImplementation)
				.Function("StaticClass", StaticClassImplementation)
				.Function("GetClass", GetClassImplementation)
				.Function("GetName", GetNameImplementation)
				.Function("GetWorld", BINDING_FUNCTION(&UObject::GetWorld))
```
```cpp
// FRegisterClass.cpp:48-51
				.Function("GetDefaultObject", BINDING_OVERLOAD(UObject*(UClass::*)(bool)const,
				                                               &UClass::GetDefaultObject,
				                                               TArray<FString>{"bCreateIfNeeded"}, true))
				.Function("RemoveFunction", RemoveFunctionImplementation);
```
```cpp
// FRegisterMulticastDelegate.cpp:190-192
				.Function("Contains", ContainsImplementation)
				.Function("IsBound", IsBoundImplementation)
				.Function("Contains", ContainsImplementation)   // ← 与 :190 名字与目标完全相同
```

**调用上下文**
`FScriptDomainImpl.inl:611-631` 把类里**所有** `MethodRegisters` 无条件展平注册进 `StringToMethod`
（`MethodBridge.cs:25`），没有"是否被 C# 声明过"的过滤。C# 侧是手写的 `Script/UE/Library/*Implementation.cs`。

**问题（已重写）**
- ~~`GetWorld` 与 `GetDefaultObject` ... 但**永不被查询**~~ —— **此论断已被推翻**。二者**都有 C# 对端且在生成产物里被真实调用**
  （`Proxy/Binding/Object.cs:20`、`Proxy/Binding/Class.cs:23`）。"误导性"论证（"看到 `.Function("GetWorld", ...)` 会让人误判已绑定"）
  恰好说反了：它**确实已绑定**。这两项**从死代码清单移除**。
- **仅 `Contains1` 成立**：`FRegisterMulticastDelegate.cpp:190/192` 名字与目标完全相同，经 §5.3 后缀机制变成 `Contains` 与 `Contains1` 两个 key；
  作者只为 `Contains` 写了声明 → `Contains1` 成为死 key。按对账，它是 **214 − 213 = 1** 的**唯一**一项。

**建议（已重写）**
- **删除 `FRegisterMulticastDelegate.cpp:192`**（唯一有效项；内容与 F-CS3-009 重复，合并处理）。
- `GetWorld`/`GetDefaultObject` **不需要任何改动**——原"二选一"建议基于错误前提，撤销。
- 根本方案仍是启动期双向对账（见 F-CS3-007），但**对账范围必须同时包含插件手写与项目生成两侧**，
  否则就会重复犯本条的错误（F-CS3-010）。

**验证方式**
- `grep 'Contains1Implementation'` over `<Project>/Script\**` → 应为 0 命中（实测 0）✓
- 反向验证（防止再次误判）：`grep '__UObject_GetWorldImplementation'` over 同一项目根 → 应 ≥2 命中（声明+调用）✓

---

#### [F-CS3-007] 跨语言名字契约缺少一致性防线：名字↔目标语义无校验、C++ 注册表与 C# 声明表无对账

- **类别**: Bug / 可优化 / 平台兼容
- **严重度**: **P2**（是 §5 全部 6 例错配与 F-CS3-001 的共同使能条件）
- **复核结论**: **部分确认**（偏差：4 个失效点全部成立，但第 2 点的**对账范围**必须扩为"插件手写 + 项目生成"两侧，
  否则本条自身的"双向对账"建议会得出 F-CS3-006 那样的错误结论 → 见新增 F-CS3-010）
- **可达性**: **活跃**
- **复核证据**: `Source/UnrealCSharp/Private/Binding/Class/FClassBuilder.cpp:83-91`（实读，所引内容逐字一致）；
  `Source/UnrealCSharpCore/Private/Binding/Class/FBindingClassRegister.cpp:113-129`（实读：`BINDING_COMBINE_CLASS(ImplementationNameSpace, BINDING_COMBINE_CLASS_IMPLEMENTATION(GetClass())) + BINDING_COMBINE_FUNCTION(BINDING_COMBINE_FUNCTION_IMPLEMENTATION(GetClass(), InImplementationName))`
  → key = `Script.Library.FStringImplementation::__FString_RegisterImplementation`，与 §1.1 推导一致 ✓）；
  `Script/Interop/Bridge/MethodBridge.cs:10`（`StringComparer.Ordinal`）、`:13-30`（实读）、`:140-148`（实读）；
  `Source/UnrealCSharpCore/Public/CoreMacro/BindingMacro.h:3-9`（实读）
- **级别变动**: 无（P2 维持）
- **注**: 第 1 点（注册名 X 与目标 Y 无校验）的机制证据在 `Source/UnrealCSharp/Public/Binding/Class/FClassBuilder.inl:11-19`：
  `Function(InName, InName, FunctionPointer.Value.Pointer)` —— 模板重载路径**直接把名字当实现名用**，从不与 Y 比较 ✓
- **文件**: `Source/UnrealCSharp/Private/Binding/Class/FClassBuilder.cpp:83-91`、`Source/UnrealCSharpCore/Private/Binding/Class/FBindingClassRegister.cpp:113-129`、`Script/Interop/Bridge/MethodBridge.cs:13-30`
- **函数**: `FClassBuilder::GetFunctionImplementationName`、`FBindingClassRegister::BindingMethod`、`MethodBridge.RegisterBinding`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FClassBuilder.cpp:83-91（去重加后缀——机制本身是好的）
FString FClassBuilder::GetFunctionImplementationName(const FString& InName, const FString& InImplementationName) const
{
	const auto Count = Algo::Count(Functions, InName);

	return FString::Printf(TEXT("%s%s"), *InImplementationName,
	                       Count == 0 ? TEXT("") : *UKismetStringLibrary::Conv_IntToString(Count));
}
```
```csharp
// MethodBridge.cs:17-27
for (var Index = 0; Index < InLength; Index++)
{
    if (InNames[Index] != null && InMethods[Index] != 0)
    {
        var Name = Marshal.PtrToStringUTF8((nint)InNames[Index]) ?? string.Empty;
        if (!string.IsNullOrEmpty(Name))
        {
            StringToMethod[Name] = InMethods[Index];
        }
    }
}
```

**调用上下文**
启动期一次：`FScriptDomainImpl.inl:631` → `MethodBridge.RegisterBinding`。

**问题**
机制里有 2 个优点（`FClassBuilder.cpp:83-91` 自动去重、`MethodBridge.cs:15-21` 空值校验），
但有 4 个**静默**失效点，且全部只能靠人工审查发现：
1. **注册名 X 与目标符号 Y 的语义一致性无任何校验** —— `FClassBuilder::Function` 只把 `InName` 当字符串用
   （`FClassBuilder.inl:11-19` → `Function(InName, InName, Pointer)`），从不比较 X 与 Y。
   **这正是 §5.2 那 6 例错配能通过编译、通过测试、进入产物的根本原因。**
2. **C++ 注册表与 C# 声明表之间无对账** —— 死注册（F-CS3-006）与"名字拼错"（→ F-CS3-001 崩溃）都不报错。
3. **自动后缀掩盖重名** —— 两个不同函数共用一个公开名字时不会被发现，只会生成一个语义误导的重载（F-CS3-005）。
4. `StringToMethod` 是普通 `Dictionary`，`RegisterBinding` 标注 `[UnmanagedCallersOnly]`，
   当前只在启动期单线程写入。一旦将来挪到热重载路径或多线程注册，就是无锁数据竞争。

**建议**
三处低成本加固 + 一处根治：
```cpp
// 1) 注册期断言：名字与目标符号必须同名（对 BINDING_FUNCTION 形式可自动提取 __FUNCTION__ 尾名）
//    放在 FClassBuilder::Function(InName, InImplementationName, InMethod) 里：
//    当 InName == InImplementationName（即模板重载路径）且 InMethod 可解析出符号名时，
//    断言 符号名 == InName 或 符号名 == InName + "Implementation"。
```
```csharp
// 2) MethodBridge.RegisterBinding 内检测同名不同指针
if (StringToMethod.TryGetValue(Name, out var Existing) && Existing != InMethods[Index])
{
    Console.Error.WriteLine($"[MethodBridge] duplicate name with different target: {Name}");
}
StringToMethod[Name] = InMethods[Index];
```
3) 启动期全集对账（C++ 注册 key 集合 ↔ C# 反射出的无体 `partial` 声明），缺失项 `ensureAlways` 失败。
4) 若将来把注册挪到运行期，把 `StringToMethod` 换成不可变字典 + 写锁，或改成注册完成后冻结。

**验证方式**
加上 1) 后启动一次编辑器：应立即报出 `FRegisterRotator.cpp` 的 `Equals`/`SetComponentForAxis` 两处、
`FRegisterSoftObjectPath.cpp` 的 `GetAssetPathName` 一处（这 3 处是目标与名字不符的真 bug）。
加上 3) 后应立即报出 §3.3 的 3 个死 key。

---

### P3

#### [F-CS3-008] `GetMethod` 不缓存"查不到"的负结果，名字错误时每次调用都重复查字典

- **类别**: 性能
- **严重度**: **P3**
- **复核结论**: **确认**（代码事实逐字成立），但**可达性字段需要修正**（见下）
- **可达性**: **潜伏**（`GetMethod` 本体在 LeanCLR 下不被编译；"活跃"这一标注仅对 Mono/CoreCLR 成立）
- **复核证据**: `Script/Interop/Bridge/MethodBridge.cs:140-148`（实读：`InSlot = StringToMethod.TryGetValue(...) ? Method : nint.Zero;`
  —— 失败时不区分"未查过"与"查不到"，故负结果不缓存）；生成模板 `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:913-918`（`#else` 分支 + `ref {slot}`）
- **级别变动**: 无（P3 维持）
- **注**: 本报告已自述"由于失败必然导致 `call 0` 崩溃，这条路径实践中几乎不可达，实际性能影响接近零" —— 这个自评是对的，P3 恰当
- **文件**: `Script/Interop/Bridge/MethodBridge.cs:140-148`
- **函数**: `Interop.MethodBridge.GetMethod(ref nint, string)`
- **置信度**: 高

**现状（代码事实）**
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

**调用上下文**
由 SourceGenerator 为每个桥接方法生成的 `_Slot` + `GetMethod(...)` 调用（模板见 `UnrealTypeSourceGenerator.cs:918-919`）。

**问题**
`nint.Zero` 同时表示"未解析"和"解析失败"。成功路径只查一次字典（这部分设计是对的），
但**失败路径下 `InSlot` 仍是 0**，于是每次调用都重新做一次 `TryGetValue` + 序号字符串比较。
由于失败必然导致 `call 0` 崩溃（F-CS3-001），这条路径实践中几乎不可达，实际性能影响接近零——
列出它是因为它与 F-CS3-001 是同一处代码的同一设计缺陷，修 F-CS3-001 时应一并处理。

**建议**
按 F-CS3-001 的建议用哨兵值（如 `-1`）区分"未解析/解析失败"。

**验证方式**
在 `GetMethod` 里统计 `TryGetValue` 调用次数：对同一个不存在的名字连续调用 N 次，当前会有 N 次查找，改成哨兵后应为 1 次。

---

#### [F-CS3-009] `FRegisterMulticastDelegate` 重复注册 `Contains`（同目标），产生一个死 key `Contains1`

- **类别**: 死代码 / 可读性
- **严重度**: **P3**
- **复核结论**: **确认**（`:190` 与 `:192` 两行逐字相同，`Contains1` 两侧均无对端 —— 本条是 §3.2 方向 ② 的**唯一**成立项）
- **可达性**: **活跃**（启动期无条件注册，key 进字典后永不被查询）
- **复核证据**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterMulticastDelegate.cpp:185-201`（实读：`:190` 与 `:192` 都是 `.Function("Contains", ContainsImplementation)`）；
  `grep 'Contains1Implementation'` over 项目 `<Project>/Script\**/*.cs` → **0 命中**（实测）
- **级别变动**: 无（P3 维持）
- **与 F-CS3-006 的关系**: F-CS3-006 的 3 项里只有本项成立，故 F-CS3-006 已就地撤销 2 项、保留本项（内容重叠，可视为同一条）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterMulticastDelegate.cpp:190`、`:192`
- **函数**: `.Function("Contains", ContainsImplementation)` 出现两次
- **置信度**: 高

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Private/Domain/Interop/FRegisterMulticastDelegate.cpp:187-196
			FClassBuilder(TEXT("FMulticastDelegate"), NAMESPACE_LIBRARY)
				.Function("Register", RegisterImplementation)
				.Function("UnRegister", UnRegisterImplementation)
				.Function("Contains", ContainsImplementation)
				.Function("IsBound", IsBoundImplementation)
				.Function("Contains", ContainsImplementation)
				.Function("Add", AddImplementation)
```

**调用上下文**
启动期注册。经 `FClassBuilder.cpp:83-91` 的后缀去重，两条成为
`__FMulticastDelegate_ContainsImplementation` 与 `__FMulticastDelegate_Contains1Implementation`。

**问题**
纯重复（名字与目标都相同），无功能影响。列出的理由：
1. 它贡献了 §3.2 里 214 − 211 = 3 中的一项（`Contains1` 无 C# 声明），删掉后对账会变成 213 − 211 = 2（即 F-CS3-006 的两项），更清晰；
2. 它与 §5.2 的 4/5/6 号是**同一种复制粘贴习惯**（同一名字连写两行），
   只是这一例的目标没被改错——说明 6 例错配并非孤立笔误，而是一类系统性风险；
3. `FMulticastDelegate` 的 C# 侧只有 13 个声明对应 13 个唯一名字，第 14 条注册点无任何意义。

**建议**
删除 `:192` 那一行。

**验证方式**
`Select-String -Path FRegisterMulticastDelegate.cpp -Pattern 'Function\("Contains"'` 应只剩 1 命中。

---

#### [F-CS3-010] LIBRARY 命名空间的 C# 对端跨"插件手写 + 项目生成"两棵树，且生成侧槽位数与 `.Function(` 口径不可直接对账 —— 使 F-CS3-007 的对账建议无法按现有口径落地

- **类别**: 可优化/可读性（工具链/一致性防线）
- **严重度**: **P2**（无直接运行时崩溃，但**阻断**了 F-CS3-001/F-CS3-007 的防线落地，并已实际造成一次误判）
- **复核结论**: **新增（实测得出）**
- **可达性**: **活跃**（每次生成/编译都成立）
- **文件**: `Script/UE/Proxy/Binding/ClassImplementation.cs:9-13`、`Script/UE/Proxy/Binding/ObjectImplementation.cs:9-13`、
  `Script/UE/Proxy/Binding/FActorSpawnParameters.cs:6`、`Script/SourceGenerator/UnrealTypeSourceGenerator.cs:907`
- **函数**: `LibraryBridgeGenerator.EmitBridges(INamedTypeSymbol, List<IMethodSymbol>)`
- **置信度**: 高

**现状（代码事实）**
```csharp
// Script/UE/Proxy/Binding/ClassImplementation.cs:9-13（项目侧生成产物，实读）
namespace Script.Library
{
	public static unsafe partial class UClassImplementation
	{
		public static unsafe partial void __UClass_GetDefaultObjectImplementation(nint InObject, byte* InBuffer, byte* OutBuffer, byte* ReturnBuffer);
	}
}
```
```csharp
// Script/UE/Proxy/Binding/FActorSpawnParameters.cs:6（项目侧生成产物对插件命名空间的反向引用）
using Script.Library;
```
```csharp
// Script/SourceGenerator/UnrealTypeSourceGenerator.cs:907（可见性由原声明的 Accessibility 决定，故生成侧既有 public 也有 private）
var accessibility = method.DeclaredAccessibility == Accessibility.Public ? "public" : "private";
```

**调用上下文**
生成器 `LibraryBridgeGenerator`（`UnrealTypeSourceGenerator.cs:811-814`，`BridgeNamespaces = { "Script.Binding", "Script.Library" }`）
为**整个编译单元**里所有无体 `static partial` 方法补实现体；因此插件侧手写声明与项目侧生成声明会被**平等**处理
（证据：`Proxy/Binding/FActorSpawnParameters.cs:6` 的 `using Script.Library;` 说明二者在同一个 assembly 内、互可见）。

**问题**
1. **同一个逻辑对端分散在两棵树**：LIBRARY 命名空间的 C# 声明 = 插件 `Script/UE/Library/` 的 **211** 个（`private`）
   + 项目 `Script/UE/Proxy/Binding/` 的 **2** 个（`public`）。任何只按目录或只按 `private` 检索的对账都会漏项，
   本报告 F-CS3-006 的误判即为实例（用 `Select-String -Path "Script\*\*\*.cs"` 只查了插件树）。
2. **两侧可见性约定相反**（手写侧 `private`、生成侧 `public`），因此连"按 `private static unsafe partial` 计数"这种
   看似严谨的口径也只能覆盖手写侧（实测恰好 211，会让人误以为这就是全部 C# 对端）。
3. **生成侧槽位数与 `.Function(` 口径不可比**：`Script/UE/Proxy/Binding/` 的 40 个 `*Implementation.cs` 实测声明
   **950** 个桥接槽（`static unsafe partial` 990 命中 − 40 个 class 声明行），而 BINDING 命名空间的 `.Function(` 只有 **734**。
   差额来自 `.Constructor(`/`.Destructor(`（`TBindingClassBuilder.inl:14-22` 自动注册）与 `.Property(`（`FClassBuilder.inl:48-49` 派生 `Get`/`Set`）
   这些**不经过 `.Function(`** 的注册点。⇒ 直接 diff 两个数字会淹没在假阳性里，这正是"一致性防线一直没被真正建立"的工程原因。

**建议**
1. 对账脚本的输入必须是**三个集合**而非两个：① C++ 侧导出的注册 key 全集（应新增一个导出入口，
   把 `FBindingClassRegister::MethodRegisters` 的 key 字符串 dump 成文件，而不是靠数 `.Function(`）；
   ② 编译期反射出的**全部**无体 `partial` 声明（跨两棵树、不分可见性）；
   ③ 两侧做双向 diff，只把"② 有 ① 无"（→ F-CS3-001 暴露点）和"① 有 ② 无"（→ 死 key）报出来。
2. 若暂不能导出 key 全集，则最低成本修法：把生成器改成**同时**输出一份 `bridge-keys.txt`（`key` 逐行），
   并让 CI 对 C++ 侧 `MethodNames.Num()`（`FScriptDomainImpl.inl:631`）与行数做一致性断言。
3. 建议把"LIBRARY 命名空间的对端由两处构成"写进 `Script/README` 或代码注释，避免重复踩坑。

**验证方式**
- `grep 'static unsafe partial'` over `Script/UE/Proxy/Binding` → 990（含 40 个 class 行）；
- `grep 'private static unsafe partial'` over 同一目录 → 89（**只覆盖生成侧的 private 部分**，可见性口径会漏 901 条）；
- `grep 'private static unsafe partial'` over `Script/UE/Library` → 211；
- 三侧合并后与 214 对账，应只剩 `Contains1` 一条差异。

---

## 10. 死代码清单

> **更正**：下面前两行曾判为"死注册（P2）"，实测**不成立**（见 §3.3 / F-CS3-010）。

| 符号 | 声明位置 | grep 命中数（实测） | 判定 | 证据 |
|---|---|---|---|---|
| `Script.Library.UObjectImplementation::__UObject_GetWorldImplementation` | `FRegisterObject.cpp:138` | **2 命中**（`Proxy/Binding/ObjectImplementation.cs:13` 声明 + `Object.cs:20` 调用）；插件侧 0 | **非死代码（原判撤销）** | `grep '__UObject_GetWorldImplementation'` over **项目** `Script/**/*.cs` |
| `Script.Library.UClassImplementation::__UClass_GetDefaultObjectImplementation` | `FRegisterClass.cpp:48-50` | **2 命中**（`Proxy/Binding/ClassImplementation.cs:13` 声明 + `Class.cs:23` 调用）；插件侧 0 | **非死代码（原判撤销）** | `grep '__UClass_GetDefaultObjectImplementation'` over **项目** `Script/**/*.cs` |
| `Script.Library.FMulticastDelegateImplementation::__FMulticastDelegate_Contains1Implementation` | `FRegisterMulticastDelegate.cpp:192` | **0**（两侧都无 `Contains1`；手写文件只有 `Contains`，见 `FMulticastDelegateImplementation.cs:30`） | **真死 key（P3）** | `:190` 与 `:192` 名字/目标字节相同，经自动后缀成为两个 key，只有 `Contains` 被声明 |
| `Script/UE/Library/*Implementation.cs` 全部 211 个声明 | 28 个 `*Implementation.cs` | 211（`grep 'private static unsafe partial'`） | **非死代码** | §3.2 三方验证：214 注册 key − 211 手写 − 2 生成 = 1 个无对端 |
| `Script/UE/Proxy/Binding/{Class,Object}Implementation.cs` 的 2 个声明 | `ClassImplementation.cs:13`、`ObjectImplementation.cs:13` | 2 | **非死代码（新增行）** | 生成产物，与手写文件同属 `Script.Library` 的同一 partial class |
| `FRegisterMatrix.cpp:76` 的 `&FMatrix::GetColumn` | `FRegisterMatrix.cpp:76` | — | **非死代码**（初始假设错误，已更正） | §5.3：自动后缀使 `:78` 成为独立 key `GetColumn1`，两者都可达；`Matrix.cs:574` 走前者、`:590` 走后者 |
| `FRegisterVector.cpp:161` 的 `&FVector::Dot` | `FRegisterVector.cpp:161` | — | 同上 | `Vector.cs:781`→`__FVector_DotImplementation`；`Vector.cs:797`→`__FVector_Dot1Implementation` |
| `FRegisterPolyglotTextData.cpp:16` 的 `&FPolyglotTextData::SetCategory` | `FRegisterPolyglotTextData.cpp:16` | — | 同上 | `PolyglotTextData.cs:37`→`SetCategory`；`:61`→`SetCategory1` |

---

## 11. 未覆盖/存疑项

1. **`Script/UE/Proxy/Binding/` 的生成声明仍未做逐条 diff（已量化规模，未逐条对账）。** 该目录 87 个 `.cs`
   （**40 个 `*Implementation.cs`，已确认**；另有 4 个 `*Implementation.cs` 散落在 `TakeRecorder/`、`MovieScene/`、`Niagara/`），
   是 `NAMESPACE_BINDING` 注册点的 C# 对端。**测量**：`grep 'static unsafe partial'` → **990 命中 = 40 个 class 声明行 + 950 个桥接槽**
   （"共 2078 处 `__X_YImplementation(` 出现"是另一种口径：含 `*.cs` 里的**调用点**，非仅声明）。
   **注意 950 与 734 不可直接 diff**——差额来自 `.Constructor(`/`.Destructor(`（`TBindingClassBuilder.inl:14-22` 自动注册）
   与 `.Property(` 派生（`FClassBuilder.inl:48-49` 的 `Get`/`Set`），这些**不经过 `.Function(` 故不在 948 之内**。
   **该目录结构上仍可做与 §3 相同的自动化 diff**，但必须先把 C++ 侧 key 全集导出（见 F-CS3-010）。
   **这仍是目前最大的剩余风险面，建议优先补做（按 F-CS3-010 的三集合口径）。**
2. ~~**F-CS3-004 的 UE 侧返回类型未核**（置信度已标"中"）：需读 `SoftObjectPath.h` 确认
   `FSoftObjectPath::GetAssetPathName` 与 `GetAssetName` 的真实返回类型。~~
   → **【已闭环】** 已读 `Engine\Source\Runtime\CoreUObject\Public\UObject\SoftObjectPath.h`：
   `grep 'GetAssetPathName'` → **0 命中**（UE 5.6 **没有这个函数**），存在的是 `GetAssetPathString()`（`:221`）、
   `GetAssetName()`（`:266` → `FString`）、`GetAssetPath()`（`:209`）。F-CS3-004 的置信度已提升为**高**，
   原「建议」因依赖不存在的宏/函数已被就地替换。**剩余未核**：其它 `#if UE_F_*` 版本宏在 UE 5.6 下是否都取到预期值（未逐个验算）。
3. **优先级 2 只到签名级**：缓冲区偏移约定（`BINDING_FUNCTION` 展开后如何按 `FFunctionInfo` 解包
   `IN_BUFFER`/`OUT_BUFFER`/`RETURN_BUFFER`）未逐字段核对。多行签名续行部分
   （`FRegisterArray.cpp:109,121,175,185,195,223,288,298`、`FRegisterMap.cpp:63,73,85,96,118,129,161,173`、
   `FRegisterProperty.cpp:10,26,40,55`、`FRegisterUnreal.cpp:45,70,87,111,135`、`FRegisterStruct.cpp:29`）未逐参数读。
4. **优先级 3 只完成 4 处对应性检查**（`Unreal`/`FName`/`TArray`/`TOptional`）；其余文件的 `enum` 成员名与
   C++ 定义未核对；`EObjectFlags`/`ELoadFlags` 在三个后端下的实际封送宽度未验证。
5. **优先级 4 只覆盖 `MethodBridge` 周边**；未统计"每次调用 `Marshal.AllocHGlobal`"与"可批量却逐元素调用的循环"
   （`TArrayImplementation.cs:72-100` 的 `Get`/`Set`/`Find` 每元素一次 native 往返是主要嫌疑）。
6. **未核对 `Source/UnrealCSharpCore/` 是否有 LIBRARY 注册点**：只确认 `FClassBuilder`/`TBindingClassBuilder`
   的注册点全部位于 `Source/UnrealCSharp/Private/Domain/Interop/`；若 `Core` 内另有注册路径，本报告对账范围会漏。
7. **LeanCLR 分支未分析**：`UnrealTypeSourceGenerator.cs:909-912` 的 `WITH_LEANCLR` 分支走真正的
   `[DllImport("UnrealCSharp", CallingConvention = Cdecl)]`，与 Mono/CoreCLR 的函数指针派发是**两条不同**的 ABI 路径。
   §1.1 起的全部结论只对 Mono/CoreCLR 分支成立。
8. **`BINDING_PROPERTY` 生成的 `Get`/`Set` 前缀名字未纳入对账**：`FClassBuilder.inl:48-49` 会生成
   `BINDING_PROPERTY_GET + InName` 与 `BINDING_PROPERTY_SET + InName`（`BindingMacro.h:11,13` 的 `"Set"`/`"Get"`），
   本报告的 948 个 `.Function(` 扫描不包含这些由 `.Property(` 派生的注册点（如 `FRegisterWorld.cpp:97-106` 的 10 个属性）。
   **【补充】**：同样不在 948 之内的还有 `.Constructor(`/`.Destructor(`——`TBindingClassBuilder.inl:14-22` 在构造时
   对**非反射类且可默认构造**的 `T` 自动 `Constructor(BINDING_CONSTRUCTOR(T))` + `Destructor(...)`。
   证据：生成产物里存在 `Proxy/Binding/FActorSpawnParametersImplementation.cs:50` 的
   `__FActorSpawnParameters_FActorSpawnParametersImplementation` 与 `:57` 的 `__FActorSpawnParameters_DestructorImplementation`，
   而 `FRegisterWorld.cpp:96-112` 里**没有任何 `.Constructor(`/`.Destructor(` 调用**——注册来自 builder 构造函数本身。
   （已核对这一路径**不是**"声明了却没注册"的 F-CS3-001 暴露点，故未列为发现。）
9. **【新增】LeanCLR 下 `[DllImport("UnrealCSharp")]` 的解析机制未证实**：插件 `Script/` 树内
   `grep 'DllImportResolver|Bridge_Invoke|NativeLibrary'` → 0 命中，LeanCLR（`backend-leanclr`）`*.cs` 内
   `grep 'UnrealCSharp'` → 0 命中。因此本报告只能断言"LeanCLR 分支**不经过** `MethodBridge.GetMethod`"
   （由 `UnrealTypeSourceGenerator.cs:910-919` 的 `#if WITH_LEANCLR` 结构直接得出），
   **不能断言**"LeanCLR 下名字不一致也不会崩"。这是 F-CS3-001 定级为"潜伏"而非"不可达"的原因。
10. **【新增】`Script/UE/Proxy/` 下非 `Binding` 的模块目录**（`TakeRecorder/`、`MovieScene/`、`Niagara/` 等数百个目录）
    走的是**反射绑定**而非 `.Function(` 注册，其对账口径与 §3 完全不同，完全未覆盖。

