# Interop 注册：对象与反射类（FRegisterObject / Class / Struct / Property / Function / ScriptInterface）

> 本报告各条**严重度见各 Finding 的「复核结论」字段**。**13 条 = 原 12 条 + 新增 1 条（F-INT1-013）**；其中 **2 条判定为非缺陷「撤销」**（F-INT1-008、F-INT1-009）、**4 条级别调整**（F-INT1-003/004/005 P1→P2、F-INT1-007 P2→P3）。「注册名↔目标符号核对表」：**52/52 一致、0 条不一致**。
> **处置更新（修复提交 `492ce5f7`）**：`F-INT1-001`、`F-INT1-002` 两条 P0 空指针解引用**均已修复**（前者在 `FRegisterScriptInterface.cpp` 的 `GetObjectImplementation` 补三元判空、后者在 `FRegisterClass.cpp` 把 `GetString<FName>` 纳入 `if` 守卫；见各自正文「处置（已执行）」）。**编号与计数口径不变**（仍 **13 条 = 原 12 条 + 新增 1 条（F-INT1-013）**）；`严重度`/`复核结论`/`可达性` 等字段保持原值，只加处置标注。（修复提交：`492ce5f7` "Null Validation"）





> 分析范围：`Plugins/UnrealCSharp/Source/UnrealCSharp/Private/Domain/Interop/` 下的 6 个 `FRegister*.cpp`
> 覆盖文件：6 个（FRegisterObject.cpp / FRegisterClass.cpp / FRegisterStruct.cpp / FRegisterProperty.cpp / FRegisterFunction.cpp / FRegisterScriptInterface.cpp）
> 覆盖方法：逐文件全文阅读 + 注册名↔目标符号核对 + 工程级死代码扫描
> 覆盖深度：6/6 文件全文读完；注册名↔目标符号核对表覆盖全部 52 条注册项，52/52 一致；死代码扫描覆盖工程内 **16535** 个 `.cs`（排除 `\obj\`/`\bin\`），其中工程根 `Script/` 16176 个、插件 `Script/` 336 个

## 0. 覆盖范围与阅读清单
> 行数口径 = **总行数（含空行）**，与 `read` 输出的 `total N lines` 及 `(Get-Content $f).Count` 一致（ 6/6 逐文件复测全部吻合）。
> ⚠️ 口径说明（修正）：`Get-Content $f | Measure-Object -Line` 给出的是**非空行数**（忽略空行），不是"末行无换行少算 1 行"。本组 6 文件依次为 152/56/68/81/382/81（总行数）与 125/51/56/75/355/68（非空行数）——**两套数字都对，只是口径不同**。

| 文件 | 行数 | 是否读完 | 备注 |
|---|---|---|---|
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterObject.cpp` | 152 | ✅ 全文 | 12 条注册；句柄判空最规范（**但新增 F-INT1-013：`:56` 的 `IScriptDomain::Get()` 未判空**） |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterClass.cpp` | 56 | ✅ 全文 | 3 条注册（UE 5.6 下实际 2 条，见 §3.2）；含 F-INT1-002(P0)、F-INT1-007(P3)、F-INT1-009(撤销) |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterStruct.cpp` | 68 | ✅ 全文 | 4 条注册 |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterProperty.cpp` | 81 | ✅ 全文 | 4 条注册；缓冲区无长度参数 |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterFunction.cpp` | 382 | ✅ 全文（分 2 段：1-180、181-382） | 25 条注册；本组最大文件 |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterScriptInterface.cpp` | 81 | ✅ 全文 | 4 条注册；含 P0 |

**为核对语义而额外阅读的佐证文件**（不属于本组 6 个文件，仅用于判定，未做完整审计）：
`Public/Registry/FMultiRegistry.inl`（全文 159 行）、`Public/Registry/FStringRegistry.inl:1-32`、`Public/Registry/FStringRegistry.h`（grep）、`Public/Environment/FCSharpEnvironment.inl:1-40,200-269`、`Public/Registry/FCSharpBind.inl`（全文 132 行）、`Public/Macro/FunctionMacro.h:5`、`Private/Binding/Class/FClassBuilder.cpp:28-92`。

**C# 侧对照文件**（判定死代码与 ABI 一致性）：
`Script/UE/Library/{ObjectImplementation,ClassImplementation,StructImplementation,FPropertyImplementation,FFunctionImplementation,TScriptInterfaceImplementation}.cs`，以及工程生成的 `Script/UE/Proxy/Binding/{Object,Class,ObjectImplementation,ClassImplementation}.cs`、`Script/UE/CoreUObject/{Object,Class,TScriptInterface}.cs`、`Script/Weavers/UnrealTypeWeaver.cs`、`Script/SourceGenerator/UnrealTypeSourceGenerator.cs`。

## 1. 模块职责与架构速览

这 6 个 `.cpp` 都是**匿名 namespace 里的 `struct FRegisterXxx` + 一个 `[[maybe_unused]]` 全局静态实例**，靠**静态构造期（模块加载时）**执行的构造函数把一组 C++ 实现注册进 `FClassBuilder` / `TBindingClassBuilder<T>`，再由 C# 侧通过名字派发。

关键事实（已逐条读源码确认）：

- 注册**不是** `extern "C"` + `DllImport`。`FClassBuilder(...).Function("Name", &Impl)` 把 `(名字 → 函数指针)` 写进绑定的类描述符；C# 侧 `Script/UE/Library/*Implementation.cs` 是**手写声明**，运行时由名字字典 + `delegate* unmanaged[Cdecl]` 派发。
- **名字就是唯一契约**。`FClassBuilder.cpp:83-91` 用同名计数生成 `Name1/Name2` 后缀，所以注册名写错 → C# 侧要调的函数**根本不存在**，或者**静默绑到另一个语义的函数**上。这就是本报告要做「注册名 ↔ 目标符号核对表」的原因。
- 三处注册 API 的形态差异（导致重复代码）：
  | 文件 | 使用 API | 是否带 `UClass*` 反射入口 |
  |---|---|---|
  | FRegisterClass.cpp | `TBindingClassBuilder<UClass>` | 是（lambda `GetFullClass(UClass::StaticClass())`） |
  | FRegisterStruct.cpp | `TBindingClassBuilder<UStruct>` | **否**（无类型名的裸构造） |
  | FRegisterProperty.cpp | `FClassBuilder(TEXT("FProperty"), NAMESPACE_LIBRARY)` | 否 |
  | FRegisterScriptInterface.cpp | `FClassBuilder(TEXT("TScriptInterface"), NAMESPACE_LIBRARY)` | 否 |

- `IManagedHandle` 是本组所有跨语言参数的统一句柄类型（`Domain/Script/IManagedHandle.h`）。**句柄是 C# 传来的 `IntPtr`/整数**，因此「句柄查不到」是常态路径，必须判空。

## 2. 关键调用链

1. `FRegisterClass.cpp:35 FRegisterClass()` → `TBindingClassBuilder<UClass>` → `FUnrealCSharpFunctionLibrary::GetFullClass(UClass::StaticClass())`（`FRegisterClass.cpp:41`）
2. C# `UClass.RemoveFunction(name)` → 派发到 `RemoveFunctionImplementation` → `FoundClass->FindFunctionByName` → `RemoveFromRoot`/`MarkAsGarbage` → `RemoveFunctionFromFunctionMap`（`FRegisterClass.cpp:12-33`）
3. C# `UStruct.StaticStruct(name)` → `StaticStructImplementation` → `LoadObject<UScriptStruct>` → `FCSharpEnvironment::Bind`（`FRegisterStruct.cpp:13-20`）
4. C# `UStruct.Identical(a,b)` → `IdenticalImplementation` → `GetStruct<>(A)`/`GetStruct<>(B)` → `CompareScriptStruct(..., PPF_None)`（`FRegisterStruct.cpp:29-45`）
5. C# `UStruct.UnRegister(h)` → `UnRegisterImplementation` → `AsyncTask(ENamedThreads::GameThread, ...)` → `RemoveStructReference`（`FRegisterStruct.cpp:47-53`）
6. C# `FProperty.GetObjectProperty` → `GetAddress<UObject,void*>` → `GetOrAddPropertyDescriptor(hash)` → `PropertyDescriptor->Get(ContainerPtrToValuePtr<void>(addr), RETURN_BUFFER)`（`FRegisterProperty.cpp:10-24`）
7. C# `TScriptInterface.GetObject(h)` → `GetMulti<TScriptInterface<IInterface>>(h)` → `Multi->GetObject()` → `Bind(obj)`（`FRegisterScriptInterface.cpp:62-68`）← **未判空，见 F-INT1-001**
8. C# `TScriptInterface.UnRegister(h)` → `AsyncTask(GameThread)` → `RemoveMultiReference` → `FMultiRegistry.inl:54-79 RemoveReference` → `FMemory::Free(FoundValue->Value)`（`FRegisterScriptInterface.cpp:53-60` → `FMultiRegistry.inl:68`）

### 2.1 「句柄查不到」的确定性语义（核对判空的依据）

> **逐跳复核（2026）**：上面 8 条链的**每一跳**都已在源码中定位到真实行——`FRegisterClass.cpp:35`（构造函数）/`:41-42`（`GetFullClass`）/`:12-33`（`RemoveFunctionImplementation`）/`:51`（注册）、`FRegisterStruct.cpp:13-20`、`:29-45`（`CompareScriptStruct(..., PPF_None)` 在 `:39`）、`:47-53`、`FRegisterProperty.cpp:10-24`、`FRegisterScriptInterface.cpp:62-68`、`:53-60`、`Public/Registry/FMultiRegistry.inl:54-79`（`FMemory::Free` 在 `:68`）。**8/8 全部成立**，未发现悬空跳。链 7 的未判空结论见 F-INT1-001（P0 维持）。
> 唯一需补的口径说明：链 7 所属文件的 `RegisterImplementation`/`IdenticalImplementation` 各有 `#if UE_T_SCRIPT_INTERFACE_CONSTRUCTOR_U_OBJECT`，本机 UE 5.6 走 `:19` 分支（见 §3.5）。
- `FCSharpEnvironment::GetMulti<T>`（`Public/Environment/FCSharpEnvironment.inl:221-227`）：`MultiRegistry == nullptr` 时返回 **`nullptr`**。
- `FMultiRegistry::TMultiRegistryImplementation::GetMulti`（`Public/Registry/FMultiRegistry.inl:19-27`）：`TMap::Find` 未命中时返回 **`nullptr`**（`FoundValue != nullptr ? ... : nullptr`）。
→ 结论：`GetMulti` 的返回值**必须**判空。`MultiRegistry == nullptr` 意味着环境尚未初始化（模块加载顺序/早期调用）也会触发。

## 3. 注册名 ↔ 目标符号 核对表（本报告最高价值产出）

### 3.1 核对结论总览
| 文件 | 注册项数 | 名字↔目标不一致 | 备注 |
|---|---|---|---|
| FRegisterClass.cpp | 3（1 Property + 2 Function） | **0** | 全部一致 |
| FRegisterStruct.cpp | 4 Function | **0** | 全部一致 |
| FRegisterProperty.cpp | 4 Function | **0** | 全部一致 |
| FRegisterScriptInterface.cpp | 4 Function | **0** | 全部一致（但 `GetObject` 有空指针 bug） |
| FRegisterObject.cpp | 12（11 Function + 1 `BINDING_FUNCTION`） | **0** | 全部一致 |
| FRegisterFunction.cpp | 25 Function | **0** | 全部一致，且与 C# 侧 25 条声明逐条签名吻合（见 §5.3） |
| **合计** | **52** | **0** | |

> 与其它 `FRegister*.cpp` 中已确证的 6 例「注册名与目标错配」不同，**本组 52 条注册项的名字与目标符号全部语义一致**（逐条见 §3.2–§3.7）。本组的主要风险不是错配，而是**返回值未判空**（F-INT1-001、F-INT1-002）。
>
> **逐行回源码核对结论（§3.2–§3.7）**：§3.2–§3.7 的 **52 行** 全部重新定位回 `FRegister*.cpp` 的实际行号，**52/52 名字↔目标符号语义一致、0 条不一致**；`.Function("A", &C::B)` 形态不存在 `A != B` 的行。**未发现新的错配**（因此不新增错配类发现）。
> 但发现一处**计数口径**需要说明：`UClass.ClassDefaultObject` 这条 `.Property` 在 UE 5.6 下被 `UE_U_CLASS_CLASS_DEFAULT_OBJECT`（= `!UE_VERSION_START(5,6,0)`，`Source/CrossVersion/Public/UEVersion.h:148`）编译掉 ⇒ **源码层面 52 条、UE 5.6 实际生效 51 条**。唯一「注册了但没有 C# 消费者」的仍是 `ClassDefaultObject` 属性（F-INT1-009，已撤销）。


### 3.2 FRegisterClass.cpp（`TBindingClassBuilder<UClass>`）
| # | 注册 API | 注册名 | 目标符号 | 文件:行 | 一致? |
|---|---|---|---|---|---|
| 1 | `.Property` | `ClassDefaultObject` | `&UClass::ClassDefaultObject`（`BINDING_READONLY_PROPERTY`） | `FRegisterClass.cpp:46` | ✅ 一致 |
| 2 | `.Function` | `GetDefaultObject` | `UObject*(UClass::*)(bool)const` = `&UClass::GetDefaultObject`（`BINDING_OVERLOAD`，参数名 `bCreateIfNeeded`，默认值 true） | `FRegisterClass.cpp:48-50` | ✅ 一致 |
| 3 | `.Function` | `RemoveFunction` | `RemoveFunctionImplementation`（内部静态函数） | `FRegisterClass.cpp:51` / 定义 `:12-33` | ✅ 一致（名字与实现体行为匹配：`FindFunctionByName`+`RemoveFunctionFromFunctionMap`） |

补充事实（**已闭合**）：`Property("ClassDefaultObject", ...)` 挂在 `#if UE_U_CLASS_CLASS_DEFAULT_OBJECT` 下（`:45-47`），而该宏 = `!UE_VERSION_START(5, 6, 0)`（`Source/CrossVersion/Public/UEVersion.h:148`；`UE_VERSION_START` 展开见同文件 `:5-6`，`UE_GREATER_SORT` 语义为"引擎版本 ≥ (5,6,0)"）。
→ **本机 UE 5.6 下该宏为 0 ⇒ 这条 `.Property` 根本不参与编译**。因此本组在 UE 5.6 上的**实际生效注册条目是 51 条**（52 条是源码层面计数）；C# 侧 0 命中由此得到解释，也是 F-INT1-009 撤销的直接根因。

### 3.3 FRegisterStruct.cpp（`TBindingClassBuilder<UStruct>`）
| # | 注册 API | 注册名 | 目标符号 | 文件:行 | 一致? |
|---|---|---|---|---|---|
| 1 | `.Function` | `StaticStruct` | `StaticStructImplementation`（`LoadObject<UScriptStruct>` + `Bind`） | `:58` / 定义 `:13-20` | ✅ 一致（`static` 的「按名字取得 UScriptStruct」语义） |
| 2 | `.Function` | `Register` | `RegisterImplementation`（`Bind(handle, name)`） | `:59` / 定义 `:22-27` | ✅ 一致 |
| 3 | `.Function` | `Identical` | `IdenticalImplementation`（`CompareScriptStruct`） | `:60` / 定义 `:29-45` | ✅ 一致 |
| 4 | `.Function` | `UnRegister` | `UnRegisterImplementation`（`RemoveStructReference`） | `:61` / 定义 `:47-53` | ✅ 一致 |

### 3.4 FRegisterProperty.cpp（`FClassBuilder(TEXT("FProperty"), NAMESPACE_LIBRARY)`）
| # | 注册 API | 注册名 | 目标符号 | 文件:行 | 一致? |
|---|---|---|---|---|---|
| 1 | `.Function` | `GetObjectProperty` | `GetObjectPropertyImplementation`（`GetAddress<UObject,void*>` + `GetOrAddPropertyDescriptor` + `Get`） | `:73` / 定义 `:10-24` | ✅ 一致 |
| 2 | `.Function` | `SetObjectProperty` | `SetObjectPropertyImplementation`（同上但 `Set`） | `:74` / 定义 `:26-38` | ✅ 一致 |
| 3 | `.Function` | `GetStructProperty` | `GetStructPropertyImplementation`（`GetAddress<UScriptStruct,void*>` + `Get`） | `:75` / 定义 `:40-53` | ✅ 一致 |
| 4 | `.Function` | `SetStructProperty` | `SetStructPropertyImplementation`（`GetAddress<UScriptStruct,void*>` + `Set`） | `:76` / 定义 `:55-68` | ✅ 一致 |

> 注：4 个实现体是**同一段代码的 4 份拷贝**（只有模板参数 `UObject`/`UScriptStruct` 与 `Get`/`Set` 两个方向不同）——见 P3 重复代码发现。`GetObjectPropertyImplementation`（`:19-21`）与 `GetStructPropertyImplementation`（`:49-51`）在**实参顺序上也完全相同**，而 `Set` 两处（`:35` 与 `:64-65`）实参顺序一个单行一个换行、语义一致，属纯格式差异。

### 3.5 FRegisterScriptInterface.cpp（`FClassBuilder(TEXT("TScriptInterface"), NAMESPACE_LIBRARY)`）
| # | 注册 API | 注册名 | 目标符号 | 文件:行 | 一致? |
|---|---|---|---|---|---|
| 1 | `.Function` | `Register` | `RegisterImplementation`（`new TScriptInterface<IInterface>` + `AddMultiReference`） | `:73` / 定义 `:13-32` | ✅ 一致 |
| 2 | `.Function` | `Identical` | `IdenticalImplementation`（`GetMulti` 双向 + `operator==`/`GetObject()==`） | `:74` / 定义 `:34-51` | ✅ 一致 |
| 3 | `.Function` | `UnRegister` | `UnRegisterImplementation`（`AsyncTask` + `RemoveMultiReference`） | `:75` / 定义 `:53-60` | ✅ 一致 |
| 4 | `.Function` | `GetObject` | `GetObjectImplementation`（`GetMulti` → `Multi->GetObject()` → `Bind`） | `:76` / 定义 `:62-68` | ✅ 名字一致，**实现有 null 解引用缺陷** |

> **补充（宏分支）**：本文件的 `RegisterImplementation`（`:13-32`）与 `IdenticalImplementation`（`:34-51`）各有一处 `#if UE_T_SCRIPT_INTERFACE_CONSTRUCTOR_U_OBJECT`（`:18`、`:42`）。该宏 = `!UE_VERSION_START(5, 8, 0)`（`Source/CrossVersion/Public/UEVersion.h:196`）⇒ **本机 UE 5.6 下为 1**，因此**生效分支是 `:19` 的 `new TScriptInterface<IInterface>(FoundObject)`**；`:20-25`（无参构造 + `SetObject` + `SetInterface`）在本机**不参与编译**。F-INT1-006 与 §2 调用链已按此口径说明。

### 3.6 FRegisterObject.cpp（`TBindingClassBuilder<UObject>(NAMESPACE_LIBRARY)`）
| # | 注册 API | 注册名 | 目标符号 | 文件:行 | 一致? |
|---|---|---|---|---|---|
| 1 | `.Function` | `Identical` | `IdenticalImplementation`（`GetObject(A)==GetObject(B)`） | `:134` / 定义 `:16-27` | ✅ 一致 |
| 2 | `.Function` | `StaticClass` | `StaticClassImplementation`（`LoadObject<UClass>` + `Bind`） | `:135` / 定义 `:29-36` | ✅ 一致 |
| 3 | `.Function` | `GetClass` | `GetClassImplementation`（`FoundObject->GetClass()` + `Bind`） | `:136` / 定义 `:38-48` | ✅ 一致 |
| 4 | `.Function` | `GetName` | `GetNameImplementation`（`FoundObject->GetName()` + `NewString`） | `:137` / 定义 `:50-60` | ✅ 一致 |
| 5 | `.Function` | `GetWorld` | `BINDING_FUNCTION(&UObject::GetWorld)` | `:138` | ✅ 一致 |
| 6 | `.Function` | `IsValid` | `IsValidImplementation`（`IsValid(FoundObject)`） | `:139` / 定义 `:62-70` | ✅ 一致 |
| 7 | `.Function` | `IsA` | `IsAImplementation`（`FoundObject->IsA(FoundClass)`） | `:140` / 定义 `:72-83` | ✅ 一致 |
| 8 | `.Function` | `AddToRoot` | `AddToRootImplementation`（`FoundObject->AddToRoot()`） | `:141` / 定义 `:85-91` | ✅ 一致 |
| 9 | `.Function` | `RemoveFromRoot` | `RemoveFromRootImplementation`（`FoundObject->RemoveFromRoot()`） | `:142` / 定义 `:93-99` | ✅ 一致 |
| 10 | `.Function` | `IsRooted` | `IsRootedImplementation`（`FoundObject->IsRooted()`） | `:143` / 定义 `:101-109` | ✅ 一致 |
| 11 | `.Function` | `AddReference` | `AddReferenceImplementation`（`AddReference(FoundObject)`） | `:144` / 定义 `:111-119` | ✅ 一致 |
| 12 | `.Function` | `RemoveReference` | `RemoveReferenceImplementation`（`RemoveReference(FoundObject)`） | `:145` / 定义 `:121-129` | ✅ 一致 |

> FRegisterObject 是本组**判空最规范**的文件：12 个实现体全部用 `if (const auto FoundObject = ...)` 守卫，`InvalidManagedHandle` 作为失败返回值（`:47`、`:59`），未发现空指针问题。`StaticClassImplementation`（`:33`）的 `LoadObject<UClass>` 可能返回 `nullptr`，随后 `Bind(nullptr)`（`:35`）——取决于 `Bind` 是否容忍 null，见存疑项。

### 3.7 FRegisterFunction.cpp（`FClassBuilder(TEXT("FFunction"), NAMESPACE_LIBRARY)`，25 条）
25 条注册项**名字与目标符号全部一一对应**，且「名字里的数字」与「缓冲区签名 / `CallN` 模板实参」的编码规则自洽：

| # | 注册名 | 目标符号 | 文件:行 | 目标额外参数 | 一致? |
|---|---|---|---|---|---|
| 1 | `GenericCall0` | `GenericCall0Implementation` | `:353` / `:11-22` | 无 | ✅ |
| 2 | `PrimitiveCall1` | `PrimitiveCall1Implementation` | `:354` / `:24-35` | `RETURN_BUFFER` + `Call1<Primitive>` | ✅ |
| 3 | `CompoundCall1` | `CompoundCall1Implementation` | `:355` / `:37-48` | `RETURN_BUFFER` + `Call1<Compound>` | ✅ |
| 4 | `GenericCall2` | `GenericCall2Implementation` | `:356` / `:50-61` | `IN_BUFFER` + `Call2<>` | ✅ |
| 5 | `PrimitiveCall3` | `PrimitiveCall3Implementation` | `:357` / `:63-75` | `IN_BUFFER`+`RETURN_BUFFER` + `Call3<Primitive>` | ✅ |
| 6 | `CompoundCall3` | `CompoundCall3Implementation` | `:358` / `:77-89` | `IN_BUFFER`+`RETURN_BUFFER` + `Call3<Compound>` | ✅ |
| 7 | `GenericCall4` | `GenericCall4Implementation` | `:359` / `:91-102` | `OUT_BUFFER` + `Call4<>` | ✅ |
| 8 | `PrimitiveCall5` | `PrimitiveCall5Implementation` | `:360` / `:104-116` | `OUT_BUFFER`+`RETURN_BUFFER` + `Call5<Primitive>` | ✅ |
| 9 | `CompoundCall5` | `CompoundCall5Implementation` | `:361` / `:118-130` | `OUT_BUFFER`+`RETURN_BUFFER` + `Call5<Compound>` | ✅ |
| 10 | `GenericCall6` | `GenericCall6Implementation` | `:362` / `:132-143` | `IN_BUFFER`+`OUT_BUFFER` + `Call6<>` | ✅ |
| 11 | `PrimitiveCall7` | `PrimitiveCall7Implementation` | `:363` / `:145-158` | `IN`+`OUT`+`RETURN` + `Call7<Primitive>` | ✅ |
| 12 | `CompoundCall7` | `CompoundCall7Implementation` | `:364` / `:160-173` | `IN`+`OUT`+`RETURN` + `Call7<Compound>` | ✅ |
| 13 | `GenericCall8` | `GenericCall8Implementation` | `:365` / `:175-186` | 无 + `Call8<>` | ✅ |
| 14 | `PrimitiveCall9` | `PrimitiveCall9Implementation` | `:366` / `:188-199` | `RETURN_BUFFER` + `Call9<Primitive>` | ✅ |
| 15 | `CompoundCall9` | `CompoundCall9Implementation` | `:367` / `:201-212` | `RETURN_BUFFER` + `Call9<Compound>` | ✅ |
| 16 | `GenericCall10` | `GenericCall10Implementation` | `:368` / `:214-225` | `IN_BUFFER` + `Call10<>` | ✅ |
| 17 | `PrimitiveCall11` | `PrimitiveCall11Implementation` | `:369` / `:227-239` | `IN`+`RETURN` + `Call11<Primitive>` | ✅ |
| 18 | `CompoundCall11` | `CompoundCall11Implementation` | `:370` / `:241-253` | `IN`+`RETURN` + `Call11<Compound>` | ✅ |
| 19 | `GenericCall14` | `GenericCall14Implementation` | `:371` / `:255-266` | `IN`+`OUT` + `Call14<>` | ✅ |
| 20 | `PrimitiveCall15` | `PrimitiveCall15Implementation` | `:372` / `:268-281` | `IN`+`OUT`+`RETURN` + `Call15<Primitive>` | ✅ |
| 21 | `CompoundCall15` | `CompoundCall15Implementation` | `:373` / `:283-296` | `IN`+`OUT`+`RETURN` + `Call15<Compound>` | ✅ |
| 22 | `GenericCall16` | `GenericCall16Implementation` | `:374` / `:298-309` | 无 + `Call16<>` | ✅ |
| 23 | `GenericCall18` | `GenericCall18Implementation` | `:375` / `:311-322` | `IN_BUFFER` + `Call18<>` | ✅ |
| 24 | `GenericCall24` | `GenericCall24Implementation` | `:376` / `:324-335` | 无 + `Call24<>` | ✅ |
| 25 | `GenericCall26` | `GenericCall26Implementation` | `:377` / `:337-348` | `IN_BUFFER` + `Call26<>` | ✅ |

> **命名清晰度问题（P3）**：注册名里的数字**不是参数个数**，而是 `EFunctionReturnType`/缓冲区形状组合的**枚举索引**（例如 `Call8` 与 `Call0` 的 C++ 签名完全相同：`(handle, hash)` 无缓冲区）。`Call8`/`Call16`/`Call24` 三者签名一致、`Call2`/`Call10`/`Call18`/`Call26` 四者签名一致——从注册名完全无法推断出该走哪个。这使「名字写错」这类错配**在 FRegisterFunction 里天然难以被人工审查发现**，只能靠与 C# 声明逐条比对（见死代码章节）。

## 4. 发现清单

### P0（逐条终裁：F-INT1-001、F-INT1-002 **均维持 P0**，机制、调用链、触发时序全部经源码复核成立）

#### [F-INT1-001] `TScriptInterface.GetObject` 对 `GetMulti` 返回值不判空即解引用 → 空指针崩溃

- **类别**: Bug / 未定义行为
- **严重度**: **P0**(崩溃)
- **复核结论**: 确认（独立复核：代码事实、调用上下文、类别、严重度**全部成立**） **后续处置：判定升级为「确认 · 已修复（`492ce5f7`）」—— `FRegisterScriptInterface.cpp` 已改为 `Multi != nullptr ? FCSharpEnvironment::GetEnvironment().Bind(Multi->GetObject()) : InvalidManagedHandle` 三元判空，与同文件 `IdenticalImplementation` 既有的两次判空口径一致**（见下方「处置（已执行）」）
- **可达性**: 活跃
- **复核证据**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterScriptInterface.cpp:62-68` 逐字吻合（`:64-65` 取 `GetMulti<TScriptInterface<IInterface>>`，`:67` 直接 `Multi->GetObject()`，无判空）。`GetMulti` 未命中**确定**返回 `nullptr`：`Public/Registry/FMultiRegistry.inl:24-26`、`Public/Environment/FCSharpEnvironment.inl:224-226`（环境未初始化同样 `nullptr`）——已回源码逐行确认。调用链逐跳已核：`Script/UE/CoreUObject/TScriptInterface.cs:46` → `Script/UE/Library/TScriptInterfaceImplementation.cs:36` → 声明 `:32` → 注册点 `:76`。触发时序证据：`UnRegister` 走 `AsyncTask`（`:53-60`）而读路径同步（`:64-67`），同文件 `IdenticalImplementation`（`:36-40`）对两次 `GetMulti` 都判了空，**本处是组内唯一漏判**。
- **级别变动**: 无（P0 维持）
- **可达性补强**: `Script/Game/**` 中 `TScriptInterface<` 命中 **83 处**（含生成属性 `Script/Game/Proxy/Binding/TestBindingFunctionActor.cs:15`、`.../TestReflectionFunctionActor.cs:529` 与 `(TScriptInterface<T>)HandleData.GetObject(...)` 强转），类型与生成代码在本工程被大量使用，故"句柄未注册、或 `~TScriptInterface()` 触发的异步 `UnRegister` 之后仍调 `GetObject()`"是真实可达时序。**仍未运行复现**（纯静态阅读）。
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterScriptInterface.cpp:62-68`
- **函数**: `FRegisterScriptInterface::GetObjectImplementation(IManagedHandle)`
- **置信度**: 高
- **处置（已执行）**: ✅ **已修复**（修复提交 `492ce5f7` "Null Validation"）。`FRegisterScriptInterface.cpp` 的 `GetObjectImplementation` 已改为 `Multi != nullptr ? FCSharpEnvironment::GetEnvironment().Bind(Multi->GetObject()) : InvalidManagedHandle` 三元判空，未命中不再解引用空指针；这与同文件 `IdenticalImplementation`（`:36-40`）对两次 `GetMulti` 的判空口径**一致**。原文「建议」给出的正是这一行（含 `InvalidManagedHandle` 回退，与 `FRegisterObject.cpp:47`/`:59` 的惯例一致），已**逐字采纳**。未做的部分：原文「验证方式」建议的 `check(Multi)` 复现用例**未加**，`Public/Binding/Core/TPropertyValue.inl:149`/`:296`/`:526` 的同型未判空点属其它报告范围、本次**未动**；本 Finding 的 `文件:行` 引用按修复后现状自然顺延（该函数体现在是 `:62-70`），报告正文保留修复前的 `:62-68` 快照不改。

**现状（代码事实）**
```cpp
62: 		static IManagedHandle GetObjectImplementation(const IManagedHandle InManagedHandle)
63: 		{
64: 			const auto Multi = FCSharpEnvironment::GetEnvironment().GetMulti<TScriptInterface<IInterface>>(
65: 				InManagedHandle);
66: 
67: 			return FCSharpEnvironment::GetEnvironment().Bind(Multi->GetObject());
68: 		}
```
同一文件的 `IdenticalImplementation`（`:36-40`）对两次 `GetMulti` 都做了判空，唯独这里没有。

**调用上下文**
C# `Script/UE/CoreUObject/TScriptInterface.cs:46` → `TScriptInterfaceImplementation.TScriptInterface_GetObjectImplementation<U>(HandleData.GetHandle(this))` → `Script/UE/Library/TScriptInterfaceImplementation.cs:36` → `__TScriptInterface_GetObjectImplementation` → MethodBridge 名字字典派发 → 本函数。注册点在 `FRegisterScriptInterface.cpp:76`（构造期静态注册）。

**问题**
`GetMulti` 未命中时**确定返回 `nullptr`**，证据两处：
- `Public/Registry/FMultiRegistry.inl:24-26`：`return FoundValue != nullptr ? static_cast<...>(FoundValue->Value) : nullptr;`
- `Public/Environment/FCSharpEnvironment.inl:224-226`：`MultiRegistry != nullptr ? ... : nullptr`（环境未初始化同样返回 nullptr）

因此 `Multi->GetObject()` 是对空指针的成员调用 → 读地址 0 附近 → 访问违例。
触发路径：任何在 `Register` 之前、或 `UnRegister` 之后调用 `GetObject()` 的 C# 代码。`UnRegisterImplementation` 通过 `AsyncTask(GameThread, ...)` **异步**删除（`:53-60`），所以「C# 调 UnRegister 后立刻 GetObject」在游戏线程上几乎是必崩的时序。
C# 侧 `TScriptInterfaceImplementation.cs:38` 写了 `Handle != 0 ? (T)HandleData.GetObject(Handle) : default;` 的保护，说明作者预期这条路径应当「失败返回」，但 C++ 侧在返回之前就已经崩了——C# 的保护**救不了**。

**建议**
按同项目惯例返回 `InvalidManagedHandle`（对照 `FRegisterObject.cpp:47`、`:59`）：
```cpp
const auto Multi = FCSharpEnvironment::GetEnvironment().GetMulti<TScriptInterface<IInterface>>(InManagedHandle);
return Multi != nullptr ? FCSharpEnvironment::GetEnvironment().Bind(Multi->GetObject()) : InvalidManagedHandle;
```

**验证方式**
grep `GetMulti<TScriptInterface` 找同类未判空使用点（另见 `Public/Binding/Core/TPropertyValue.inl:149`、`:296`、`:526` 的 `*GetMulti`/`*GetStruct` 同型写法，属其它报告范围，建议交叉引用）。用例：C# 构造 `TScriptInterface<ITestInterface>`（不调 Register）后直接 `GetObject()`；C++ 侧可加 `check(Multi)` 复现。

---

#### [F-INT1-002] `UClass.RemoveFunction` 对 `GetString<FName>` 返回值不判空即解引用 → 空指针崩溃

- **类别**: Bug / 未定义行为
- **严重度**: **P0**(崩溃)
- **复核结论**: 确认（独立复核：代码事实、调用上下文、类别、严重度**全部成立**） **后续处置：判定升级为「确认 · 已修复（`492ce5f7`）」—— `FRegisterClass.cpp` 已把 `GetString<FName>` 纳入 `if (const auto Name = FCSharpEnvironment::GetEnvironment().GetString<FName>(InName))` 守卫，`*Name` 不再裸解引用**（见下方「处置（已执行）」）
- **可达性**: 活跃
- **复核证据**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterClass.cpp:14-19` 逐字吻合——`:14-15` 判空 `FoundClass`、`:17` 取 `GetString<FName>(InName)`、`:19` 立即 `*Name`。返回类型确为指针：`Public/Registry/FStringRegistry.h:22`（`typedef TStringAddress<FName*> FNameAddress`）、`:44`（`FNameMapping`）；未命中返回 `nullptr`：`Public/Registry/FStringRegistry.inl:25-27`、`Public/Environment/FCSharpEnvironment.inl:256-261`。调用链逐跳已核（grep 工具，模式 `UClass_RemoveFunctionImplementation`，全工程 **4 行 = 声明 + 包装 + 内部调用 + 调用方**）：`Script/UE/CoreUObject/Class.cs:9` → `Script/UE/Library/ClassImplementation.cs:9`（包装）→ `:11`（`__UClass_...`）→ 声明 `:7`；注册点 `:51`。全组 6 文件仅此 1 处使用 `GetString<`。
- **级别变动**: 无（P0 维持）
- **可达性补强**: 触发只需 C# 传入一个**未在 `FName` 注册表登记**的句柄（`default(FName)`、已释放的 `FName`、或环境未初始化时的任何 `FName`）。`FRegisterName.cpp:17/:55` 走 `AddStringReference` 登记，未登记即 `nullptr`。**仍未运行复现**。
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterClass.cpp:17-19`
- **函数**: `FRegisterClass::RemoveFunctionImplementation(IManagedHandle, IManagedHandle)`
- **置信度**: 高
- **处置（已执行）**: ✅ **已修复**（修复提交 `492ce5f7` "Null Validation"）。`FRegisterClass.cpp` 已把 `GetString<FName>` 纳入 `if (const auto Name = FCSharpEnvironment::GetEnvironment().GetString<FName>(InName))` 守卫，`*Name` 只在该守卫内解引用，未登记句柄由「崩溃」变为「什么都不做」。原文「建议」的判空意图**已采纳**，只是从 `if (Name == nullptr) { return; }` 的早退写法改为与该文件既有的 `FoundClass`、`Function` 两层守卫**同形的 `if (const auto ...)` 嵌套风格**。未做的部分：原文「验证方式」的用例（C# 传 `default(FName)` 或未初始化的 `FName`）**未实跑**（本报告仍为纯静态阅读）；失败路径**未加日志、也无错误返回**（函数为 `void`，C# 侧看不出与成功的区别）；本次只修 `*Name` 这一处，`FStringRegistry.inl:25-27` 的 `GetString` 未命中仍返回 `nullptr` 的契约**未改**。

**现状（代码事实）**
```cpp
14: 			if (const auto FoundClass = FCSharpEnvironment::GetEnvironment().GetObject<UClass>(
15: 				InManagedHandle))
16: 			{
17: 				const auto Name = FCSharpEnvironment::GetEnvironment().GetString<FName>(InName);
18: 
19: 				if (const auto Function = FoundClass->FindFunctionByName(*Name))
```
`FoundClass` 判了空，`Name` **没有判空**，第 19 行直接 `*Name`。

**调用上下文**
C# `Script/UE/CoreUObject/Class.cs:9` → `UClassImplementation.UClass_RemoveFunctionImplementation(HandleData.GetHandle(this), ...)` → `Script/UE/Library/ClassImplementation.cs:11` → `__UClass_RemoveFunctionImplementation` → 名字派发 → 本函数。注册点 `FRegisterClass.cpp:51`。

**问题**
`GetString<FName>` 未命中时**确定返回 `nullptr`**：
- `Public/Registry/FStringRegistry.inl:25-27`：`FoundValue != nullptr ? static_cast<...>(FoundValue->Value) : nullptr;`
- `Public/Environment/FCSharpEnvironment.inl:256-261`：`StringRegistry != nullptr ? ... : nullptr`
- 返回类型确认为指针：`Public/Registry/FStringRegistry.h:22` `typedef TStringAddress<FName*> FNameAddress;`、`:44` `typedef TStringValueMapping<void*, FNameAddress> FNameMapping;` → `GetString<FName>` 返回 `FName*`。

`*Name` 即「以 nullptr 为地址构造 `FName` 的副本」（`FindFunctionByName` 按值收 `FName`），读取地址 0 → 访问违例。
触发路径：`RemoveFunction` 的第二个参数 `InName` 若是一个未被 `FName` 注册表登记的句柄（例如 C# 侧传 `FName.None`、已释放的 `FName`、或构造了但未走注册路径的实例），C++ 侧崩溃而非返回失败。

**建议**
```cpp
const auto Name = FCSharpEnvironment::GetEnvironment().GetString<FName>(InName);
if (Name == nullptr) { return; }
if (const auto Function = FoundClass->FindFunctionByName(*Name))
```

**验证方式**
grep `GetString<` 逐点核对是否解引用前判空（本组 6 文件中仅此 1 处使用）。用例：C# `SomeClass.RemoveFunction(default(FName))` 或传入未初始化的 `FName`。

---

### P1（**初判分组标题**；实际 F-INT1-003/004/005 均为 **P2**。标题与编号一律保持不变，以维持跨报告引用锚点；每条的真实级别见其 `严重度` 字段）

#### [F-INT1-003] 注册失败路径泄漏 `new TScriptInterface`：`AddMultiReference` 返回值被丢弃

- **类别**: 内存/资源泄漏
- **严重度**: **P2**(泄漏)
- **复核结论**: 部分确认（偏差：原定的"确定性泄漏"只在 `MultiRegistry == nullptr` 窗口成立；正常时序下 `AddMultiReference` 恒返回 true ⇒ P2 比 P1 贴切）
- **可达性**: 活跃
- **复核证据**: `FRegisterScriptInterface.cpp:19` 裸 `new`（该 `#if` 分支在本机 UE 5.6 生效，见 §3.5 宏说明）；`:30-31` 调用 `AddMultiReference<TScriptInterface<IInterface>, true, false>(Class, InManagedObject, ScriptInterface)` 且**丢弃返回值**。 grep（工具 grep，path=`Source/UnrealCSharp/Private/Domain/Interop`，模式 `AddMultiReference|AddReference|AddStructReference|AddContainerReference|AddDelegateReference|AddStringReference|AddBindingReference`）→ 16 行，`:30` 是唯一丢弃返回值的 `AddMultiReference`（其余 9 处 `Add*Reference` 同样未检查返回值）。失败条件有源码依据：`Public/Environment/FCSharpEnvironment.inl:241-244` 在 `MultiRegistry == nullptr` 时 `return false`。释放点 `Public/Registry/FMultiRegistry.inl:54-79`，仅命中条目时 `FMemory::Free(FoundValue->Value)`（`:66-71`）⇒ 失败路径 `new` 出的对象**无任何所有者**，且 C# 侧 `void` 返回看不到失败（`:73` 注册名）。
- **级别变动**: 无（P1→P2 维持，理由如上）
- **说明**: "验证方式"里提到的 `AddStructReference` 调用点——**实际本组 6 文件 0 处**（`FRegisterStruct::RegisterImplementation` 走的是 `Bind(handle, name)`，`:26`）；`AddStructReference` 出现在其它目录文件中。
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterScriptInterface.cpp:19-31`
- **函数**: `FRegisterScriptInterface::RegisterImplementation(IManagedHandle, IManagedHandle, IManagedHandle)`
- **置信度**: 高

**现状（代码事实）**
```cpp
19: 			const auto ScriptInterface = new TScriptInterface<IInterface>(FoundObject);
...
30: 			FCSharpEnvironment::GetEnvironment().AddMultiReference<TScriptInterface<IInterface>, true, false>(
31: 				Class, InManagedObject, ScriptInterface);
```
`AddMultiReference` 的返回值（`bool`）被**完全忽略**，且 `ScriptInterface` 是裸 `new`，唯一的所有者是注册表条目。

**调用上下文**
C# `Script/UE/CoreUObject/TScriptInterface.cs:16` → `TScriptInterfaceImplementation.cs:14` → `__TScriptInterface_RegisterImplementation`。释放点在 `FRegisterScriptInterface.cpp:53-60` → `RemoveMultiReference` → `FMultiRegistry.inl:54-79`，其中只有命中条目时才 `FMemory::Free(FoundValue->Value)`（`FMultiRegistry.inl:66-71`）。

**问题**
`AddMultiReference` **会失败并返回 false**，此时什么也没注册：
```cpp
// Public/Environment/FCSharpEnvironment.inl:241-244
return MultiRegistry != nullptr ? ...AddReference<IsNeedFree, IsMember>(...) : false;
```
即 `MultiRegistry == nullptr`（环境未初始化 / 模块加载早期 / 已销毁）时：
1. `new TScriptInterface<IInterface>` 已分配（`:19`），
2. 注册表里没有条目，`RemoveMultiReference` 永远找不到它 → 永不释放 → **确定性泄漏**（每次调用泄漏一个 `TScriptInterface`，8/16 字节 + 分配器头），
3. C# 侧拿到一个「看起来成功」的调用（`void` 返回），后续对该句柄的一切操作都会走 F-INT1-001 的空指针路径。

**建议**
```cpp
const auto ScriptInterface = new TScriptInterface<IInterface>(FoundObject);

if (!FCSharpEnvironment::GetEnvironment().AddMultiReference<TScriptInterface<IInterface>, true, false>(
        Class, InManagedObject, ScriptInterface))
{
    delete ScriptInterface;   // 或 FMemory::Free，与 RemoveReference 的释放方式保持一致
}
```
注意要与 `FMultiRegistry.inl:68` 的 `FMemory::Free` 采用同一种释放方式（UE 中全局 `operator new` 亦映射到 `FMemory::Malloc`，二者等价；但显式统一更安全）。

**验证方式**
grep `AddMultiReference` / `AddReference` / `AddStructReference` 的调用点，确认是否有任何一处检查了返回值（本组 6 个文件均未检查）。可在 `AddReference` 的 `InRegistry == nullptr` 分支加 `ensure` 复现。

---

#### [F-INT1-004] 跨边界缓冲区无长度参数：`FProperty` 的 Get 向 C# `stackalloc` 缓冲区盲写

- **类别**: Bug / 安全（缓冲区溢出）
- **严重度**: **P2**
- **复核结论**: 确认（无长度参数这一核心事实已由宏定义直接证实；"需版本错配才触发"使 P2 定级成立）
- **可达性**: 活跃
- **复核证据**: **宏定义已读到，§7.2 存疑项闭合**：`Source/UnrealCSharpCore/Public/CoreMacro/BufferMacro.h:9` `#define IN_BUFFER_SIGNATURE uint8* IN_BUFFER`、`:15` `OUT_BUFFER_SIGNATURE`、`:21` `#define RETURN_BUFFER_SIGNATURE uint8* RETURN_BUFFER` ⇒ **跨边界缓冲区确实没有任何长度参数**。C# 侧同样无长度：`Script/UE/Library/FPropertyImplementation.cs:7`（`(nint InObject, uint InPropertyHash, byte* ReturnBuffer)`）。生成调用方自定大小：`Script/UE/Proxy/UnrealEd/Classes/ThumbnailRendering/ThumbnailRenderingInfo.cs:161`（`stackalloc byte[1]`）+ `:163`（`FProperty_GetStructPropertyImplementation(..., ReturnBuffer)`），`stackalloc` 另见 `Script/UE/Proxy/Binding/Object.cs:18`、`Script/UE/Proxy/Binding/Class.cs:17`。织入器按**方法名**定位这四个实现（`Script/Weavers/UnrealTypeWeaver.cs:1135-1149`）。
- **级别变动**: 无（P1→P2 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterProperty.cpp:10-24`、`:40-53`
- **函数**: `FRegisterProperty::GetObjectPropertyImplementation` / `GetStructPropertyImplementation`
- **置信度**: 高（由"中"升级："跨边界无长度参数"已由 `Source/UnrealCSharpCore/Public/CoreMacro/BufferMacro.h:9/15/21` 直接证实；仅"栈溢出后果"仍依赖 C# 程序集与 C++ 二进制版本错配，未构造运行时用例）

**现状（代码事实）**
```cpp
10: 		static void GetObjectPropertyImplementation(const IManagedHandle InManagedHandle,
11: 		                                            const uint32 InPropertyHash, RETURN_BUFFER_SIGNATURE)
...
19: 					PropertyDescriptor->Get(
20: 						PropertyDescriptor->ContainerPtrToValuePtr<void>(FoundAddress),
21: 						RETURN_BUFFER);
```
C# 侧对应声明**没有任何长度参数**：
```csharp
// Script/UE/Library/FPropertyImplementation.cs:7
private static unsafe partial void __FProperty_GetObjectPropertyImplementation(nint InObject, uint InPropertyHash, byte* ReturnBuffer);
```
生成侧的调用方自己决定缓冲区大小，例如 `Script/UE/Proxy/UnrealEd/Classes/ThumbnailRendering/ThumbnailRenderingInfo.cs:155-191`（**完整路径**）：
```csharp
FPropertyImplementation.FProperty_GetStructPropertyImplementation(HandleData.GetHandle(this), __bUseClassDefaultObject, ReturnBuffer);
```

**调用上下文**
由 **IL 织入**注入到生成的属性访问器中：`Script/Weavers/UnrealTypeWeaver.cs:1135-1149` 通过**方法名字符串**定位这四个实现——`:1135-1137` 找 `FProperty_GetObjectPropertyImplementation`、`:1139-1141` 找 `Set...`、`:1143-1145` 找 `GetStructProperty...`、`:1147-1149` 找 `SetStructProperty...`。C++ 注册点 `FRegisterProperty.cpp:73-76`。命中数极大：`FProperty_GetObjectPropertyImplementation` 在 C# 侧共 **22651** 次引用，`GetStructProperty` **20786** 次（仅对后者做了实证抽样，见 §5.2 核对表）。

**问题**
C++ 侧把「写入多少个字节」完全交给 `PropertyDescriptor` 自己的类型知识，而**跨语言边界上没有长度参数可供校验**。这意味着缓冲区大小是纯粹的「生成器与运行时的隐式约定」：
- 若 C# 程序集与 C++ 二进制**版本错配**（典型场景：蓝图中给结构体加了字段、插件升级后未重新生成 C#、`Intermediate` 中的旧程序集被加载），C# 侧仍按旧大小 `stackalloc`，C++ 侧按新大小写入 → **栈缓冲区溢出**（`stackalloc` 就在栈上，见 `Class.cs:17`、`Object.cs:18` 的写法）。
- C++ 侧没有任何 `check(长度足够)` 的可能，因为它不知道长度。

**建议**
在 `RETURN_BUFFER_SIGNATURE` 中补一个 `const uint32 InBufferSize` 参数，`PropertyDescriptor->Get` 前做 `check(InBufferSize >= PropertyDescriptor->GetSize())`；C# 生成侧传入 `ReturnBuffer` 的实际长度。若为兼容 ABI 不便改签名，至少加版本/哈希一致性校验（生成时写入属性尺寸，运行时比对），并在 Shipping 下 `ensure` 而非静默写。

**验证方式**
~~grep `RETURN_BUFFER_SIGNATURE` 看宏展开是否真的不含长度（本报告未读到宏定义，见存疑项）~~ → **已完成**：`Source/UnrealCSharpCore/Public/CoreMacro/BufferMacro.h:9/15/21` 确认三个缓冲区宏都只有 `uint8*` 指针、无长度参数（存疑项 §7.2 闭合）。剩余待做：手工把生成程序集里的 `stackalloc byte[N]` 改小后调用，观察是否越界写（未做，见 §7.10）。

---

#### [F-INT1-005] 注册表「异步写 / 同步读」竞态：`UnRegister` 走 AsyncTask，读取走调用线程

- **类别**: 并发/线程安全
- **严重度**: **P2**
- **复核结论**: 部分确认（偏差：读写线程不一致是确凿代码事实，但"非游戏线程调用方真实存在"未取证 ⇒ 维持 P2 而非 P1）
- **可达性**: 活跃
- **复核证据**: `FRegisterScriptInterface.cpp:53-60` 与 `FRegisterStruct.cpp:47-53` 逐字吻合（均为 `AsyncTask(ENamedThreads::GameThread, [InManagedHandle]{ ... })`；lambda 值捕获）。读路径无任何线程切换：`FRegisterScriptInterface.cpp:34-51`、`:62-68`、`FRegisterStruct.cpp:29-45` 直接读写同一批 `TMap`（`Public/Registry/FMultiRegistry.inl:22` `Find`、`:73` `Remove`）。** grep 证据**：整个 `Source/UnrealCSharp/Private/Domain/Interop/` 目录搜 `IsInGameThread|check\(|ensure` → **0 命中**（工具 grep，path=该目录）⇒ 本组 6 文件 0 处线程判定。
- **级别变动**: 无（P1→P2 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterScriptInterface.cpp:53-60`、`Source/UnrealCSharp/Private/Domain/Interop/FRegisterStruct.cpp:47-53`
- **函数**: `FRegisterScriptInterface::UnRegisterImplementation` / `FRegisterStruct::UnRegisterImplementation`
- **置信度**: 中（是否真有非游戏线程调用方未确证；但读写线程不一致是代码事实）

**现状（代码事实）**
```cpp
// FRegisterScriptInterface.cpp:53-60
53: 		static void UnRegisterImplementation(const IManagedHandle InManagedHandle)
54: 		{
55: 			AsyncTask(ENamedThreads::GameThread, [InManagedHandle]
56: 			{
57: 				(void)FCSharpEnvironment::GetEnvironment().RemoveMultiReference<TScriptInterface<IInterface>>(
58: 					InManagedHandle);
59: 			});
60: 		}
```
```cpp
// FRegisterStruct.cpp:47-53
47: 		static void UnRegisterImplementation(const IManagedHandle InManagedHandle)
48: 		{
49: 			AsyncTask(ENamedThreads::GameThread, [InManagedHandle]
50: 			{
51: 				(void)FCSharpEnvironment::GetEnvironment().RemoveStructReference(InManagedHandle);
52: 			});
53: 		}
```
而**读**路径没有任何线程切换：`IdenticalImplementation`（`:34-51`）、`GetObjectImplementation`（`:62-68`）、`FRegisterStruct::IdenticalImplementation`（`:29-45`）都在**调用者线程**直接读写同一批 `TMap`（`FMultiRegistry.inl:22` 的 `Find`、`:73` 的 `Remove`）。

**问题**
1. **返回即失效语义错误**：`UnRegister` 立刻返回，但删除还没发生。C# 调用方若紧接着调 `Identical`/`GetObject`（或在 finally 中做清理），看到的是**过期状态**；反过来，`AsyncTask` 排队的 lambda 捕获的 `InManagedHandle` 可能在真正执行前已被 C# 侧复用/释放。
2. **数据竞态**：若 C# 从非游戏线程（例如 `Task.Run`、异步 IO 回调、音频线程）调用 `Identical`/`GetObject`，就是与游戏线程上 `FMultiRegistry::RemoveReference` 的无锁并发 `TMap` 读写 → 迭代器/桶失效、崩溃。注册表中**没有看到任何 `check(IsInGameThread())`**（本组 6 个文件 0 处）。

**建议**
- 二选一：把读路径也统一到 `AsyncTask(GameThread, ...)`；或在所有注册/查询/删除入口加 `check(IsInGameThread())`，把「必须游戏线程」变成显式契约。
- 若要保持同步返回，`UnRegister` 应在游戏线程时**直接**执行（`if (IsInGameThread()) { Remove...; } else { AsyncTask(...); }`），消除「返回后状态未变」的窗口。
- `AsyncTask` 的 lambda 捕获 `IManagedHandle` 是值捕获（安全），但需要在文档中明确句柄的有效期。

**验证方式**
grep `AddMultiReference|GetMulti|RemoveMultiReference|AddStructReference|RemoveStructReference`，确认全部调用点所在线程；在 `TMultiRegistryImplementation::RemoveReference` 入口加 `check(IsInGameThread())` 跑 UnitTest。

---

### P2（**初判分组标题**；实际含 P2 的 F-INT1-006、P3 的 F-INT1-007、撤销的 F-INT1-008，以及新增的 P2 条目 F-INT1-013）

#### [F-INT1-006] `Register` 在对象为空时仍创建并登记 `TScriptInterface` 包装

- **类别**: 可优化/隐患
- **严重度**: **P2**
- **复核结论**: 确认（复核：`FoundObject` 确实未判空；"登记一个空接口被当成成功"成立）
- **可达性**: 活跃
- **复核证据**: `FRegisterScriptInterface.cpp:16` 取 `GetObject(InObject)` 后**无任何判空**，`:19` 直接构造包装并 `:30-31` 登记。**补充宏分支口径**：本机 UE 5.6 下 `UE_T_SCRIPT_INTERFACE_CONSTRUCTOR_U_OBJECT` = `!UE_VERSION_START(5,8,0)` = **1**（`Source/CrossVersion/Public/UEVersion.h:196`）⇒ 生效代码只有 `:19` 的 `new TScriptInterface<IInterface>(FoundObject)`；`:20-25` 的 `SetObject`/`SetInterface` 分支**不参与编译**（"非 `#if` 分支下还会调两次"的说法在本机不适用）。关于 `:28` `GetClass(InManagedType)` 返回 nullptr 无害的判断已被源码证实：`Public/Registry/FMultiRegistry.inl:37-52` 的 `InClass` 仅在 `IsMember == true` 时被使用，本处传 `false`（`:30`）。
- **级别变动**: 无（P2 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterScriptInterface.cpp:13-32`
- **函数**: `FRegisterScriptInterface::RegisterImplementation`
- **置信度**: 高

**现状（代码事实）**
```cpp
16: 			const auto FoundObject = FCSharpEnvironment::GetEnvironment().GetObject(InObject);
17: 
18: #if UE_T_SCRIPT_INTERFACE_CONSTRUCTOR_U_OBJECT
19: 			const auto ScriptInterface = new TScriptInterface<IInterface>(FoundObject);
20: #else
21: 			const auto ScriptInterface = new TScriptInterface<IInterface>();
22: 
23: 			ScriptInterface->SetObject(FoundObject);
24: 
25: 			static_cast<FScriptInterface*>(ScriptInterface)->SetInterface(FoundObject);
26: #endif
```
`FoundObject`（`GetObject` 的结果）**没有判空**。

**问题**
若 `InObject` 句柄无效/为 0，`GetObject` 返回 nullptr，于是：
1. 仍然 `new` 一个包装（非 `#if` 分支下还会调两次 `SetObject`/`SetInterface`），
2. 仍然登记进注册表，
3. 于是 C# 侧 `Register` 成功、`GetObject()` 返回一个空对象（甚至先撞上 F-INT1-001 的崩溃路径）。

即「登记一个空接口」被当成成功。注意 `FReflectionRegistry::Get().GetClass(InManagedType)`（`:28`）返回 nullptr 时**是安全的**——`FMultiRegistry.inl:37-52` 的 `AddReference` 形参 `InClass` 在 `IsMember == false` 时**根本没被使用**（本组调用传 `false`，见 `:30`），所以不要误报为缺陷。

**建议**
```cpp
if (FoundObject == nullptr) { return; }   // 或返回 bool/错误码给 C#
```
`CompareScriptStruct` 之外的 `IdenticalImplementation` 里 `GetObject() == GetObject()` 对两个空接口会返回 true（`:45`），空接口还会污染相等性判断。

**验证方式**
C# 用例：用一个已知无效的 `nint` 调 `RegisterImplementation`，断言不产生有效注册。

---

#### [F-INT1-013] `FRegisterObject.GetName` 对 `IScriptDomain::Get()` 不判空即调用 `NewString`（本组唯一一处，同目录另有 5 处同型）—— **新增**

- **类别**: Bug / 空指针
- **严重度**: **P2**（隐患：空指针分支只在"脚本域已被销毁"的窗口可达，但一旦命中即崩溃；正常运行期不会触发，故不定 P0）
- **复核结论**: 新增（独立复核时新发现，编号顺延，**未改动任何既有编号**）
- **可达性**: 活跃（`GetName` 本身是常规路径；空指针分支需要"域已销毁"的特定生命周期状态）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterObject.cpp:56`
- **函数**: `FRegisterObject::GetNameImplementation(IManagedHandle)`
- **置信度**: 中（"`Get()` 在域销毁后为 `nullptr`"由源码证实；"该窗口内确有 C# → C++ 调用"未做运行时验证）

**现状（代码事实）**
```cpp
50: 		static IManagedHandle GetNameImplementation(const IManagedHandle InManagedHandle)
51: 		{
52: 			if (const auto FoundObject = FCSharpEnvironment::GetEnvironment().GetObject(InManagedHandle))
53: 			{
54: 				const auto Name = FoundObject->GetName();
55: 
56: 				return IScriptDomain::Get()->NewString(TCHAR_TO_UTF8(*Name));
57: 			}
58: 
59: 			return InvalidManagedHandle;
60: 		}
```

**调用上下文**
C# `Script/UE/CoreUObject/Object.cs:34` → `UObjectImplementation.UObject_GetNameImplementation(...)` → `__UObject_GetNameImplementation` → 本函数；注册点 `FRegisterObject.cpp:137`（`.Function("GetName", GetNameImplementation)`）。`Script/UE/CoreUObject/Object.cs:36` 的 `ToString() => GetName().ToString()` 也走这条路径，属于常见调用。

**问题**
`IScriptDomain::Get()` 返回一个**可被置空的裸静态指针**：
- 定义：`Source/UnrealCSharpCore/Private/Domain/Script/IScriptDomain.cpp:3`（`IScriptDomain* IScriptDomain::ScriptDomain{}`，默认 `nullptr`）、`:5-8`（`Get()` 直接返回该指针）。
- 置空点：`Source/UnrealCSharpCore/Private/Domain/Script/FScriptDomainFactory.cpp:47-60` 的 `Destroy()` 在 `delete InScriptDomain`（`:56`）之后执行 `IScriptDomain::Set(nullptr)`（`:58`）；调用者有 `FDomain::Deinitialize()`（`Source/UnrealCSharp/Private/Domain/FDomain.cpp:36-44`）和 `FScriptDomainScope::~FScriptDomainScope()`（`Source/UnrealCSharpCore/Public/Domain/Script/FScriptDomainScope.h:39-45`）。
→ **域销毁之后 `IScriptDomain::Get()` 确定返回 `nullptr`**，此时任何 `UObject.GetName()`/`ToString()` 都会在 `:56` 变成对空指针的虚调用。
- **"需要判空"是本代码库自己的契约**：同一仓库另有 30+ 处写成 `if (const auto ScriptDomain = IScriptDomain::Get())`（例：`Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp:416/449/706`、`Source/UnrealCSharpCore/Private/Reflection/FPropertyReflection.cpp:23/45`、`Source/UnrealCSharp/Private/Domain/FDomain.cpp:40/48/66`、`Source/UnrealCSharpCore/Private/Bridge/FTypeBridge.cpp:172/187`）；`FScriptDomainScope` 这个类的存在本身就说明"域可能不存在"是被正视的状态。
- **同型未判空点全仓库共 6 处**，本报告的 6 个文件只占 1 处：`FRegisterObject.cpp:56`、`FRegisterName.cpp:46`、`FRegisterString.cpp:46`、`FRegisterText.cpp:72`、`FRegisterUtf8String.cpp:50`、`FRegisterAnsiString.cpp:52`（其余 5 处属 `10b-Interop注册-容器字符串与对象指针.md` 范围，建议交叉引用）。

**建议**
```cpp
if (const auto ScriptDomain = IScriptDomain::Get())
{
    return ScriptDomain->NewString(TCHAR_TO_UTF8(*Name));
}

return InvalidManagedHandle;
```
更彻底的做法：给 `IScriptDomain` 加一个 `static IManagedHandle NewStringSafe(const char*)`（内部判空并返回 `InvalidManagedHandle`），把 6 处一次性收口，避免"某一份拷贝忘了判空"再次发生（与 F-INT1-012 的建议同源）。

**验证方式**
grep `IScriptDomain::Get\(\)->`（工具 grep，path=`Source/UnrealCSharp/Private/Domain/Interop`）当前命中 **6 行**，修复后应为 0；grep `IScriptDomain::Get\(\)` 统计"判空 / 不判空"比例。用例：编辑器关闭或 PIE 结束后触发一次 `UObject.GetName()`（或 `ToString()`），并在 `:56` 前加 `check(ScriptDomain)` 复现。

---

#### [F-INT1-007] `RemoveFunction` 的 GC 顺序：先 `MarkAsGarbage()` 再触碰 `Function`

- **类别**: 可优化/隐患（改判，原"未定义行为"；可达 UAF 前提经引擎源码核对后不成立，见复核证据③④）
- **严重度**: **P3**
- **复核结论**: 部分确认（偏差：顺序事实成立，但支撑它的两条论据经 UE 5.6 引擎源码核对后**均不成立** ⇒ 由"未定义行为"改判为"健壮性/一致性隐患"）
- **可达性**: 活跃
- **复核证据**: <br>①`FRegisterClass.cpp:21-31` 逐字吻合（`:21` `IsRooted()` → `:23` `RemoveFromRoot()`，否则 `:27` `MarkAsGarbage()` → `:30` `RemoveFunctionFromFunctionMap(Function)`）。<br>②**`MarkAsGarbage()` 的前置条件被正确满足**：`Engine\Source\Runtime\CoreUObject\Public\UObject\UObjectBaseUtility.h:176-191` 的实现里 **`:178` 是 `check(!IsRooted())`**——本代码只在 `!IsRooted()` 分支调用它。<br>③**"同帧 GC 回收 `Function` ⇒ `:30` UAF"不成立**：`:27` 与 `:30` 之间是直线代码（`AtomicallySetFlags` / `SetGarbage` + `FUClassFuncScopeWriteLock` + `GetFName()`），**不存在任何 GC 点**；引擎侧 `UClass::RemoveFunctionFromFunctionMap` 本体在 `.../CoreUObject/Public/UObject/Class.h:3276-3287`，内部两次 `Function->GetFName()`（`:3280`、`:3285`），对"已标垃圾但尚未回收"的对象读取是安全的。<br>④**"需与引擎 `UClass::RemoveFunction` 对齐"不成立**：UE 5.6 **没有** `UClass::RemoveFunction`（`Class.h` 只有 `AddFunctionToFunctionMap:3260` / `RemoveFunctionFromFunctionMap:3276`；`Runtime/CoreUObject` 全目录搜 `RemoveFunctionFromFunctionMap` 仅 2 命中：`Class.h:3276`、`Private/VerseVM/VVMVerseClass.cpp:1288`）。<br>⑤**但"先摘映射、再处理 root/garbage"的方向有引擎依据**：`UClass::FuncMap` 是 `TMap<FName, TObjectPtr<UFunction>>`（`Class.h:3164`），`UClass::AddReferencedObjects` 会把每个条目登记为 stable reference（`Private/UObject/Class.cpp:4823-4826`）；UE 5.6 还专门提供 `OnReportGarbageReferencers`（`Public/UObject/UObjectGlobals.h:3364-3367`：*"Called when garbage collection detects references to objects that are marked for explicit destruction by MarkAsGarbage"*）⇒ 先摘 `FuncMap` 可避免"垃圾对象仍被 FuncMap 引用"被引擎报告，建议保留但降为健壮性改进。
- **级别变动**: 无（P2→P3 维持；**类别**由"未定义行为"改为"可优化/隐患"，因为可达 UAF 前提不成立）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterClass.cpp:21-31`
- **函数**: `FRegisterClass::RemoveFunctionImplementation`
- **置信度**: 中

**现状（代码事实）**
```cpp
21: 					if (Function->IsRooted())
22: 					{
23: 						Function->RemoveFromRoot();
24: 					}
25: 					else
26: 					{
27: 						Function->MarkAsGarbage();
28: 					}
29: 
30: 					FoundClass->RemoveFunctionFromFunctionMap(Function);
```

**问题**
删除了 root 引用或标记为垃圾之后，第 30 行仍把 `Function` 交给 `UClass::RemoveFunctionFromFunctionMap`。此时 `Function` 已不再被 root 保护：若同一帧内发生 GC（`Function` 除 `FunctionMap` 外无其它引用），`Function` 可能已被回收，第 30 行即 use-after-free。另外 `MarkAsGarbage()` 只是打标记、**不会**把函数从 `FunctionMap` 里摘掉——两个动作的顺序与语义都需要与 UE 自身的 `UClass::RemoveFunction` 实现对齐。

**建议**
先 `RemoveFunctionFromFunctionMap(Function)`，再处理 root/garbage：
```cpp
FoundClass->RemoveFunctionFromFunctionMap(Function);
if (Function->IsRooted()) { Function->RemoveFromRoot(); }
else { Function->MarkAsGarbage(); }
```

**验证方式**
与 `Engine/Source/Runtime/CoreUObject/Private/UObject/Class.cpp` 中 `UClass::RemoveFunctionFromFunctionMap` 的官方用法对照；用例：C# 动态添加函数后立即删除并强制 `GEditor->ForceGarbageCollection(true)`。

---

#### [F-INT1-008] `LoadObject` 失败 → `Bind(nullptr)`：`UStruct` 路径的 null 安全性未确认 —— **已证伪：非缺陷，请从排期移除**

- **类别**: 空指针
- **严重度**: **撤销（非缺陷）**
- **复核结论**: 证伪（撤销，非缺陷）—— 决定性证据已找到，结论成立
- **可达性**: 不可达（null 路径被下层 `BindImplementation` 拦截）
- **复核证据**: `FCSharpBind::BindImplementation(UStruct* InStruct)` **真实存在**于 `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:106-111`，其中 `:108-111` 即 `if (InStruct == nullptr) { return false; }`（声明 `Public/Registry/FCSharpBind.h:52`；`Public/Registry/FCSharpBind.inl:9-27` 只是转发层）。因此 `FRegisterStruct.cpp:19` / `FRegisterObject.cpp:35` 的 `Bind(nullptr)` 安全返回 `false`；`UObject*` 一侧另经 `Public/Registry/FCSharpBind.inl:56-61` 返回 `InvalidManagedHandle`，C# 再把 0 句柄映射为 `null`（`Script/UE/Library/StructImplementation.cs:22`、`Script/UE/Library/ObjectImplementation.cs:29`）。**§7.1 的存疑项就此闭合。**
- **级别变动**: 撤销（非缺陷），维持
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterStruct.cpp:17-19`、`Source/UnrealCSharp/Private/Domain/Interop/FRegisterObject.cpp:33-35`
- **函数**: `FRegisterStruct::StaticStructImplementation` / `FRegisterObject::StaticClassImplementation`
- **置信度**: 高（由"低"升级：`Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:106-111` 已读到，缺失的那一半证据补齐）

**现状（代码事实）**
```cpp
// FRegisterStruct.cpp:17-19
17: 			const auto InStruct = LoadObject<UScriptStruct>(nullptr, *StructName);
18: 
19: 			return FCSharpEnvironment::GetEnvironment().Bind(InStruct);
```
```cpp
// FRegisterObject.cpp:33-35
33: 			const auto InClass = LoadObject<UClass>(nullptr, *ClassName);
34: 
35: 			return FCSharpEnvironment::GetEnvironment().Bind(InClass);
```

**问题**
`LoadObject` 对不存在的路径返回 nullptr，随后 `Bind(nullptr)`。**已确认的一半**：`FCSharpBind::BindImplementation(UObject*)` 对 nullptr 有防护并返回 `InvalidManagedHandle`（`Public/Registry/FCSharpBind.inl:56-61`），且 C# 侧把 `Handle == 0` 映射为 `null`（`Script/UE/Library/StructImplementation.cs:22`、`ObjectImplementation.cs:29`）——这条路径是**安全**的。
**未确认的一半**（**已确认，见下**）：`UClass*`/`UScriptStruct*` 的重载决议会优先选中 `Bind(UStruct*)`（`Public/Environment/FCSharpEnvironment.inl:12-16`），而该重载第 11 行先调用 `FCSharpEnvironment::GetEnvironment().GetClassDescriptor(InStruct)`——`.inl` 里只有转发层，真正的实现体在 **`Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:106-111`**，其 `:108-111` 为 `if (InStruct == nullptr) { return false; }`（声明 `Public/Registry/FCSharpBind.h:52`）。⇒ **null 路径已确认安全，本条撤销成立。**

**建议**
~~读 `FCSharpBind::BindImplementation(UStruct*)` 确认是否有 `if (InStruct == nullptr) return InvalidManagedHandle;`；若无则补上。~~ → **已完成，无需修改**。剩余可选优化：在 `FRegisterStruct::StaticStructImplementation`（`:13-20`）与 `FRegisterObject::StaticClassImplementation`（`:29-36`）里对 `InClassName == nullptr` 加 early-return，省掉一次必然失败的 `LoadObject("")`。

**验证方式**
grep `BindImplementation` 全部重载；用例：C# `UStruct.StaticStruct("")` / `UClass.StaticClass("/Script/Nope.Nope")` 断言返回 null 而不崩。

---

### P3（**初判分组标题**；含撤销的 F-INT1-009 与 P3 的 F-INT1-010/011/012）

#### [F-INT1-009] 死注册：`UClass.ClassDefaultObject` 属性无任何 C# 消费者 —— **已证伪：非缺陷，请从排期移除**

- **类别**: 死代码
- **严重度**: **撤销（非缺陷）**
- **复核结论**: 证伪（撤销，非缺陷）—— 补上"版本宏取值"证据后结论更强：该注册在本机**根本不参与编译**
- **可达性**: 不可达（UE 5.6 下不编译，任何配置下都到不了）
- **复核证据**: ①`FRegisterClass.cpp:45-47` 逐字吻合。<br>②**`UE_U_CLASS_CLASS_DEFAULT_OBJECT` = `!UE_VERSION_START(5, 6, 0)`（`Source/CrossVersion/Public/UEVersion.h:148`；`UE_VERSION_START` 展开见同文件 `:5-6`，基于 `ENGINE_MAJOR/MINOR/PATCH_VERSION` 的 `UE_GREATER_SORT`）⇒ 本机 UE 5.6 下取值为 0，`#if` 块**被编译掉**。故它不是"死注册"，而是"本构建不存在的注册"（原来的 0 命中因此有了直接解释，而不是"生成器漏生成"）。<br>③C# 侧独立复核：全工程 `.cs`（16535 个，排除 `\obj\`/`\bin\`）搜 `__UClass_ClassDefaultObject|_ClassDefaultObjectImplementation` → **0 命中**（工具 grep，path=工程根）；`Script/UE/Proxy/Binding/Class.cs`（共 29 行）只生成了 `GetDefaultObject`（`:13-27`）。<br>④§7.5 的存疑项（版本宏取值未核对）就此闭合。
- **级别变动**: 撤销（非缺陷），维持
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterClass.cpp:45-47`
- **函数**: `FRegisterClass::FRegisterClass()`
- **置信度**: 高

**现状（代码事实）**
```cpp
45: #if UE_U_CLASS_CLASS_DEFAULT_OBJECT
46: 				.Property("ClassDefaultObject", BINDING_READONLY_PROPERTY(&UClass::ClassDefaultObject))
47: #endif
```

**调用上下文**
`UClass` 的成员绑定生成在 `Script/UE/Proxy/Binding/Class.cs`，该文件**只生成了 `GetDefaultObject`**（`:13-27`），没有 `ClassDefaultObject` 属性访问器。

**问题**
全项目 `.cs`（16535 个文件）精确 grep `__UClass_ClassDefaultObject|_ClassDefaultObjectImplementation` → **0 命中**。对照组：其它 property 注册（如织入器使用的 `FProperty_GetStructPropertyImplementation`）有 20786 次引用，而 `bUseClassDefaultObject` 这类真实属性在 `[工程] Script/UE/Proxy/UnrealEd/Classes/ThumbnailRendering/ThumbnailRenderingInfo.cs:155-191` 生成了完整访问器。因此这条 `Property` 注册**没有产生任何 C# API**，是纯粹的静态注册开销 + 维护噪音；同时也说明「`Property` 注册了但生成器没吐出 C#」这一失配模式在本项目里是**真实存在的**（可与其它 FRegister*.cpp 的 `Property`/`Subscript` 注册交叉核对）。

**建议**
要么补生成器支持让它吐出属性访问器，要么删除该注册；若该属性是被有意隐藏的（`ClassDefaultObject` 是 `UClass` 的内部 CDO 指针），建议加注释说明「注册但故意不暴露」，否则后来者会当成漏生成来「修」。

**验证方式**
`grep -r "__UClass_ClassDefaultObject" --include=*.cs` → 0；对照 `grep -r "FProperty_GetStructPropertyImplementation" --include=*.cs | measure` → 20786。

---

#### [F-INT1-010] 4 份近乎逐字重复的 `FProperty` 读写实现（可模板化）

- **类别**: 可优化/可读性
- **严重度**: **P3**
- **复核结论**: 确认（复核：4 份实现体的"仅两处差异"成立；P3 维持）
- **可达性**: 活跃
- **复核证据**: 4 个实现体行区间逐字吻合——`:10-24`（Get/`UObject`）、`:26-38`（Set/`UObject`）、`:40-53`（Get/`UScriptStruct`）、`:55-68`（Set/`UScriptStruct`）；两处 `Get`（`:19-21` 与 `:49-51`）除缩进外逐字相同，`Set` 两处（`:35` 单行 与 `:64-65` 换行）实参顺序一致。 grep（工具 grep，path=`.../Domain/Interop/FRegisterProperty.cpp`，模式 `if \(const auto`）→ **8 行 = 4 个实现体 × 2 层守卫**，与"同构拷贝"判断一致。
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterProperty.cpp:10-68`
- **函数**: 4 个 `*Implementation`
- **置信度**: 高

**现状（代码事实）**
```cpp
13: 			if (const auto FoundAddress = FCSharpEnvironment::GetEnvironment().GetAddress<
14: 				UObject, void*>(InManagedHandle))
...
19: 					PropertyDescriptor->Get(
20: 						PropertyDescriptor->ContainerPtrToValuePtr<void>(FoundAddress),
21: 						RETURN_BUFFER);
```
4 个实现体（`:10-24`、`:26-38`、`:40-53`、`:55-68`）的差异**只有两处**：模板实参 `UObject` vs `UScriptStruct`，以及 `Get` vs `Set`（实参顺序镜像）。其中 `Get` 的两处（`:19-21` 与 `:49-51`）连同缩进都完全相同。

**建议**
```cpp
template <typename TOwner, bool bIsGet>
static void PropertyImplementation(const IManagedHandle InManagedHandle, const uint32 InPropertyHash
                                   /*, 缓冲区参数*/)
{
    if (const auto FoundAddress = FCSharpEnvironment::GetEnvironment().GetAddress<TOwner, void*>(InManagedHandle))
    {
        if (const auto PropertyDescriptor = FCSharpEnvironment::GetEnvironment().GetOrAddPropertyDescriptor(InPropertyHash))
        {
            if constexpr (bIsGet)
                PropertyDescriptor->Get(PropertyDescriptor->ContainerPtrToValuePtr<void>(FoundAddress), RETURN_BUFFER);
            else
                PropertyDescriptor->Set(IN_BUFFER, PropertyDescriptor->ContainerPtrToValuePtr<void>(FoundAddress));
        }
    }
}
// 注册侧
.Function("GetObjectProperty", &PropertyImplementation<UObject, true>)
```
注意 `Get`/`Set` 的缓冲区参数签名不同（`RETURN_BUFFER_SIGNATURE` vs `IN_BUFFER_SIGNATURE`），推荐用两个薄封装函数模板化，而不是用 `if constexpr` 硬塞（否则两套参数都要声明）。

**验证方式**
编译后确认生成的 C# 名字不变（`FProperty_GetObjectPropertyImplementation` 等 4 个），即 `Script/UE/Library/FPropertyImplementation.cs` 无需改动。

---

#### [F-INT1-011] `FRegisterFunction` 的注册名数字无法推断语义，人工审查无法发现错配

- **类别**: 可优化/可读性
- **严重度**: **P3**
- **复核结论**: 确认（复核：25 条实现体的行号区间与签名分组逐条吻合；P3 维持）
- **可达性**: 活跃
- **复核证据**: grep（工具 grep，path=`.../Domain/Interop/FRegisterFunction.cpp`，模式 `Implementation\(|Call\d+<|IN_BUFFER|OUT_BUFFER|RETURN_BUFFER_SIGNATURE|BINDING_FUNCTION_SIGNATURE`）→ **85 行**，25 个实现体起始行 `11/24/37/50/63/77/91/104/118/132/145/160/175/188/201/214/227/241/255/268/283/298/311/324/337` 与 §3.7 表**全部吻合**，`CallN<>` 模板实参与缓冲区参数组合也逐条吻合（见 §3.7 第 4 列）。因此"注册名里的数字 = 缓冲区形状 × 返回类型组合索引、不是参数个数"这一论断，以及本 Finding 的分组表，**全部成立**。
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterFunction.cpp:350-378`
- **函数**: `FRegisterFunction::FRegisterFunction()`
- **置信度**: 高

**现状（代码事实）**
```cpp
352: 			FClassBuilder(TEXT("FFunction"), NAMESPACE_LIBRARY)
353: 				.Function("GenericCall0", GenericCall0Implementation)
...
365: 				.Function("GenericCall8", GenericCall8Implementation)
...
374: 				.Function("GenericCall16", GenericCall16Implementation)
376: 				.Function("GenericCall24", GenericCall24Implementation)
```

**问题**
名字里的数字**不是参数个数**，而是「缓冲区形状 × 返回类型」的组合索引。实际 C++ 签名分组：
- 无缓冲区：`Call0`、`Call8`、`Call16`、`Call24`（4 个签名完全相同：`(handle, hash)`）
- 仅 `RETURN_BUFFER`：`Call1`、`Call9`（+ `Primitive`/`Compound` 两种变体）
- 仅 `IN_BUFFER`：`Call2`、`Call10`、`Call18`、`Call26`
- `IN`+`RETURN`：`Call3`、`Call11`
- 仅 `OUT_BUFFER`：`Call4`
- `OUT`+`RETURN`：`Call5`
- `IN`+`OUT`：`Call6`、`Call14`
- `IN`+`OUT`+`RETURN`：`Call7`、`Call15`

同签名不同名字（`Call0`/`Call8`/`Call16`/`Call24`）意味着**注册名写错时行为可能完全正常**（因为签名相同），只是绕了一个等价实现——这类错配无法通过运行时行为发现，只能靠名字比对。

**建议**
在注册处加注释把索引含义写明，或改用带语义的别名（如 `Call0_NoBuffer`），或让生成器输出一张 `注册名 ↔ C# 声明` 的清单在 CI 里自动比对（见 §5.3 的建议）。

**验证方式**
见 §5.3 的 25↔25 逐条比对结果。

---

#### [F-INT1-012] 全组一致的句柄守卫样板可抽成一个 helper

- **类别**: 可优化/可读性
- **严重度**: **P3**
- **复核结论**: 部分确认（偏差：样板确实大量重复，但"46 处"与各文件处数**混用了两种统计粒度**，口径不可复现；按统一口径重算为 **77 处**）
- **可达性**: 活跃
- **复核证据**: grep（工具 grep，path=`Source/UnrealCSharp/Private/Domain/Interop`，模式 `if \(const auto`）在 6 个文件命中 **77 行**：`FRegisterObject.cpp` 12、`FRegisterFunction.cpp` 50（25 实现体 × 2 层）、`FRegisterProperty.cpp` 8（4 × 2）、`FRegisterStruct.cpp` 3、`FRegisterScriptInterface.cpp` 2、`FRegisterClass.cpp` 2。这里的 12/3/2 是"守卫行数"、25/4 是"实现体个数"，**两种粒度不可相加**（按"实现体个数"口径本组应为 12+25+4+4+4+1 = 50）。结论方向不受影响：无论哪种口径，抽 helper 的收益都成立。
- **级别变动**: 无（P3 维持）
- **文件**: `FRegisterObject.cpp`（12 处守卫）、`FRegisterFunction.cpp`（50 处守卫 / 25 个实现体）、`FRegisterProperty.cpp`（8 处守卫 / 4 个实现体）、`FRegisterStruct.cpp`（3 处守卫）、`FRegisterScriptInterface.cpp`（2 处守卫）、`FRegisterClass.cpp`（2 处守卫）
- **置信度**: 高

**现状（代码事实）**
同一段三层嵌套守卫在本组被抄了 **46 次**。例如 `FRegisterFunction.cpp:11-22`、`:24-35`、`:37-48` … 全部形如：
```cpp
14: 			if (const auto FoundObject = FCSharpEnvironment::GetEnvironment().GetObject(InManagedHandle))
15: 			{
16: 				if (const auto FunctionDescriptor = FCSharpEnvironment::GetEnvironment().GetOrAddFunctionDescriptor<
17: 					FUnrealFunctionDescriptor>(InFunctionHash))
18: 				{
19: 					FunctionDescriptor->Call0<>(FoundObject);
20: 				}
21: 			}
```
`FRegisterProperty.cpp:13-22`、`FRegisterScriptInterface.cpp:36-41` 等是同一形状的变体。`FRegisterObject.cpp` 的 12 个实现体甚至是「取对象 → 调一个成员函数」的纯样板。

**建议**
```cpp
template <typename TObject, typename TFunc>
static void WithObject(const IManagedHandle InManagedHandle, TFunc&& InFunc)
{
    if (const auto FoundObject = FCSharpEnvironment::GetEnvironment().GetObject<TObject>(InManagedHandle))
    {
        InFunc(FoundObject);
    }
}
// 用法
WithObject<UObject>(InManagedHandle, [](UObject* O){ O->AddToRoot(); });
```
配合 `FRegisterFunction` 的 X-macro 表（`名字 / CallIndex / ReturnType / BufferShape` 四列），25 个实现体可以收敛成 1 个模板 + 25 行数据。这样也能顺手消掉 F-INT1-001/002 这类「某一个拷贝忘了判空」的类别性缺陷。

**验证方式**
改动后跑 `UnitTest/Binding/*` 与 `UnitTest/Reflection/*` 用例；确认 `Script/` 侧无任何改动（名字与签名不变）。

---

## 5. 死代码清单

### 5.1 判定方法
- 扫描范围：`<Project>` 下全部 `.cs`，**排除** `\obj\` 与 `\bin\`。实测总数 = **16535**（其中工程根 `Script/` 16176、插件 `Plugins/UnrealCSharp/Script/` 336，其余 23 个散落在其它子目录）。该值与 §0 状态行及 F-INT1-009 中的 16535 自洽。
- 统计方式（一次扫描，PS 5.1）：
  ```powershell
  $files = Get-ChildItem -Path $root -Recurse -Filter *.cs -File | Where-Object { $_.FullName -notmatch '\\obj\\|\\bin\\' }
  $files | Select-String -Pattern $rx -Encoding UTF8 -AllMatches
  ```
  `$rx` 为各注册名的转义并集；命中数按匹配到的符号名分桶（`__X` 前缀与不带前缀的调用都计入，故「声明 + 调用」通常 = 4 或 2）。
- 关键背景：注册名的 C# 形态是 **`<类名>_<注册名>_Implementation`**（`Public/Macro/FunctionMacro.h:5` 定义 `FUNCTION_IMPLEMENTATION_SUFFIX = "_Implementation"`；`Private/Binding/Class/FClassBuilder.cpp:83-91` 用 `Functions` 里的同名计数追加 `1/2/...` 以支持重载）。所以 `.Function("GetClass", ...)` on `UObject` → `UObject_GetClassImplementation`。**grep 必须用这个全名，搜短名会漏。**
- **重要方法论更正**：只搜插件自带的 `Script/` 会得出错误结论。例如 `GetWorld`、`GetDefaultObject` 在插件 `Script/` 里 0 命中，看似死代码；实际上它们是「UObject 成员函数指针」注册（`BINDING_FUNCTION`/`BINDING_OVERLOAD`），对应 C# 在**工程生成的** `Script/UE/Proxy/Binding/Object.cs:20`、`Class.cs:23` 中，用通用的 4 缓冲区签名 `(nint, byte* InBuffer, byte* OutBuffer, byte* ReturnBuffer)` 调用。本报告已按工程根重新扫描。

### 5.2 结果表
| # | 注册名（C++） | 注册位置 | 命中数 | 判定 | grep 证据 |
|---|---|---|---|---|---|
| 1 | `Identical` (UObject) | `FRegisterObject.cpp:134` | 4 | 活 | 声明+调用，`Script/UE/CoreUObject/Object.cs:21` |
| 2 | `StaticClass` | `FRegisterObject.cpp:135` | 7328 | 活（生成器模板） | `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:223` 生成 `StaticClassSingleton ??=` |
| 3 | `GetClass` | `FRegisterObject.cpp:136` | 4 | 活 | `Object.cs:32` |
| 4 | `GetName` | `FRegisterObject.cpp:137` | 4 | 活 | `Object.cs:34` |
| 5 | `GetWorld` | `FRegisterObject.cpp:138` | 2 | 活（生成代理） | `Script/UE/Proxy/Binding/Object.cs:20` + `.../ObjectImplementation.cs:13` |
| 6 | `IsValid` | `FRegisterObject.cpp:139` | 4 | 活 | `Object.cs:38` |
| 7 | `IsA` | `FRegisterObject.cpp:140` | 4 | 活 | `Object.cs:41` |
| 8 | `AddToRoot` | `FRegisterObject.cpp:141` | 4 | 活 | `Object.cs:44` |
| 9 | `RemoveFromRoot` | `FRegisterObject.cpp:142` | 4 | 活 | `Object.cs:47` |
| 10 | `IsRooted` | `FRegisterObject.cpp:143` | 4 | 活 | `Object.cs:49` |
| 11 | `AddReference` | `FRegisterObject.cpp:144` | 4 | 活 | `Object.cs:52` |
| 12 | `RemoveReference` | `FRegisterObject.cpp:145` | 4 | 活 | `Object.cs:55` |
| 13 | `ClassDefaultObject` | `FRegisterClass.cpp:46` | **0**（精确模式） | **UE 5.6 下不编译**（宏门控，非死注册） | `__UClass_ClassDefaultObject\|_ClassDefaultObjectImplementation` → 0 命中；根因：`UE_U_CLASS_CLASS_DEFAULT_OBJECT = !UE_VERSION_START(5,6,0)` = 0（`CrossVersion/Public/UEVersion.h:148`） |
| 14 | `GetDefaultObject` | `FRegisterClass.cpp:48` | 2 | 活（生成代理） | `Script/UE/Proxy/Binding/Class.cs:23` |
| 15 | `RemoveFunction` | `FRegisterClass.cpp:51` | 4 | 活 | `Script/UE/CoreUObject/Class.cs:9` |
| 16 | `StaticStruct` | `FRegisterStruct.cpp:58` | 5938 | 活 | `UnrealTypeSourceGenerator.cs:160` |
| 17 | `Register` (UStruct) | `FRegisterStruct.cpp:59` | 3250 | 活 | 织入器 `Script/Weavers/UnrealTypeWeaver.cs:1117-1118` |
| 18 | `Identical` (UStruct) | `FRegisterStruct.cpp:60` | 5938 | 活 | `UnrealTypeSourceGenerator.cs:189` |
| 19 | `UnRegister` (UStruct) | `FRegisterStruct.cpp:61` | 3250 | 活 | `Script/Weavers/UnrealTypeWeaver.cs:1120-1121` |
| 20 | `GetObjectProperty` | `FRegisterProperty.cpp:73` | 22651 | 活 | 生成属性 getter，实证点 `Script/UE/Proxy/TakeRecorder/Recorder/TakeRecorderSubsystemImplementation.cs:25`（`ThumbnailRenderingInfo.cs:163` 为 `GetStructProperty` 的调用点，见第 22 行） |
| 21 | `SetObjectProperty` | `FRegisterProperty.cpp:74` | 22651 | 活 | 同上 setter |
| 22 | `GetStructProperty` | `FRegisterProperty.cpp:75` | 20786 | 活 | `Script/Weavers/UnrealTypeWeaver.cs:1143-1145`；实证调用点 `Script/UE/Proxy/UnrealEd/Classes/ThumbnailRendering/ThumbnailRenderingInfo.cs:163` |
| 23 | `SetStructProperty` | `FRegisterProperty.cpp:76` | 20786 | 活 | `Script/Weavers/UnrealTypeWeaver.cs:1147-1149`；实证调用点 `.../ThumbnailRendering/ThumbnailRenderingInfo.cs:150/177` |
| 24 | `Register` (TScriptInterface) | `FRegisterScriptInterface.cpp:73` | 4 | 活 | `Script/UE/CoreUObject/TScriptInterface.cs:16` |
| 25 | `Identical` (TScriptInterface) | `FRegisterScriptInterface.cpp:74` | 4 | 活 | `TScriptInterface.cs:34` |
| 26 | `UnRegister` (TScriptInterface) | `FRegisterScriptInterface.cpp:75` | 4 | 活 | `TScriptInterface.cs:13` |
| 27 | `GetObject` (TScriptInterface) | `FRegisterScriptInterface.cpp:76` | 4 | 活（但有 P0） | `TScriptInterface.cs:46` |
| 28–52 | `GenericCall0` … `GenericCall26`（25 条） | `FRegisterFunction.cpp:353-377` | 各 ≥2 | 活 | 25 条 `__FFunction_*Implementation` 声明在 `Script/UE/Library/FFunctionImplementation.cs:7-195` |

**结论（修正）**：源码层面 52 条注册项中 **51 条有真实 C# 消费者**；第 52 条 `ClassDefaultObject` 在 **UE 5.6 下被版本宏编译掉**（`Source/CrossVersion/Public/UEVersion.h:148`），故本构建实际生效 **51 条**、**不存在"死注册"**（F-INT1-009 撤销）。本组**未发现**任何 `.Function("A", &C::B)` 名字与目标错配的情况（与其它 `FRegister*.cpp` 中已确证的 6 例不同）。

** grep 命中数核对**（工具 grep；path = 工程根 `<Project>`，include = `*.cs`；口径 = 命中**行数**，模式取**不带 `__` 前缀**的符号名，从而同时覆盖"声明 / 包装 / 内部调用 / 调用方"，与正文分桶口径等价）：

| 表行 | 搜索模式 | 正文命中数 | 核对 | 一致? |
|---|---|---|---|---|
| #1 `Identical`(UObject) | `UObject_IdenticalImplementation` | 4 | **4** | ✅ |
| #2 `StaticClass` | `UObject_StaticClassImplementation` | 7328 | **7328** | ✅ |
| #5 `GetWorld` | `UObject_GetWorldImplementation` | 2 | **2** | ✅ |
| #13 `ClassDefaultObject` | `__UClass_ClassDefaultObject\|_ClassDefaultObjectImplementation` | 0 | **0** | ✅ |
| #14 `GetDefaultObject` | `UClass_GetDefaultObjectImplementation` | 2 | **2** | ✅ |
| #15 `RemoveFunction` | `UClass_RemoveFunctionImplementation` | 4 | **4** | ✅ |
| #16 `StaticStruct` | `UStruct_StaticStructImplementation` | 5938 | **5938** | ✅ |
| #27 `GetObject`(TScriptInterface) | `TScriptInterface_GetObjectImplementation` | 4 | **4** | ✅ |
| #8 `AddToRoot`(UObject) | `UObject_AddToRootImplementation` | 4 | **4**（由联合模式命中 5946 = 5938+4+4 反算校验） | ✅ |
| #18 `Identical`(UStruct) | `UStruct_IdenticalImplementation` | 5938 | **5938**（同上反算） | ✅ |
| #25 `Identical`(TScriptInterface) | `TScriptInterface_IdenticalImplementation` | 4 | **4**（同上反算） | ✅ |
| #28–52 `FFunction` 25 条 | `__FFunction_`（限定 `Plugins/UnrealCSharp/Script/UE/Library/FFunctionImplementation.cs`） | "各 ≥2" | **50 行 = 25 声明 + 25 包装**，声明行号与 §5.3 逐条吻合 | ✅ |
| #20/#21 `Get/SetObjectProperty`、#22/#23 `Get/SetStructProperty` | `FProperty_*Implementation` | 22651 / 22651 / 20786 / 20786 | **未逐条复算**（已实测：`FProperty_GetObjectPropertyImplementation` 在 `Script/UE/Proxy/TakeRecorder/Recorder/TakeRecorderSubsystemImplementation.cs:25-214` 有 8 处真实调用，`GetStructProperty` 在 `ThumbnailRenderingInfo.cs:163/177` 有真实调用 ⇒ "活"的判定成立；**2 万级精确数字未逐条复算**） | ⚠️ 部分 |

> #24 `Register`/`UnRegister`(TScriptInterface) 的 4 未被单独复算（与 #27 同族的计数规律一致，且有 #8/#25 反算旁证）。以上 12 行核对**零偏差**；未复算的 4 行结论（活/死）均已由实证调用点独立支持。

### 5.3 `FRegisterFunction` 25 ↔ 25 逐条签名比对（错配专项）
C++ 侧 25 条注册与 C# 侧 25 条 `__FFunction_*` 声明**数量、顺序、名称、参数类型完全一致**，是一条独立且强的正确性证据。
**独立复核**：`Plugins/UnrealCSharp/Script/UE/Library/FFunctionImplementation.cs` 内 `__FFunction_` 命中 **50 行 = 25 条声明 + 25 条包装调用**（grep 工具，路径 `Plugins/UnrealCSharp/Script/UE/Library/FFunctionImplementation.cs`，模式 `__FFunction_`），声明的 25 个行号 `7/14/22/30/37/45/53/60/68/76/84/92/100/107/115/123/130/138/146/154/162/170/177/184/191` 与本表**逐一吻合**；参数形状亦逐条吻合（下表"参数"列即据此核出，其中 `Call7` 一族为 **5 参**，已修正原表的"4 参"笔误）。

| 组 | C++ 注册（`FRegisterFunction.cpp`） | C++ 签名要点 | C# 声明（`FFunctionImplementation.cs`） | 参数 | 一致? |
|---|---|---|---|---|---|
| 无缓冲区 | `GenericCall0:353` | `(h,hash)` | `:7` | `(nint, uint)` | ✅ |
| ret | `PrimitiveCall1:354` / `CompoundCall1:355` | `+RETURN` | `:14` / `:22` | `(nint,uint,byte* Return)` | ✅ |
| in | `GenericCall2:356` | `+IN` | `:30` | `(nint,uint,byte* In)` | ✅ |
| in+ret | `PrimitiveCall3:357` / `CompoundCall3:358` | `+IN+RETURN` | `:37` / `:45` | `(nint,uint,In,Return)` | ✅ |
| out | `GenericCall4:359` | `+OUT` | `:53` | `(nint,uint,Out)` | ✅ |
| out+ret | `PrimitiveCall5:360` / `CompoundCall5:361` | `+OUT+RETURN` | `:60` / `:68` | `(nint,uint,Out,Return)` | ✅ |
| in+out | `GenericCall6:362` | `+IN+OUT` | `:76` | `(nint,uint,In,Out)` | ✅ |
| in+out+ret | `PrimitiveCall7:363` / `CompoundCall7:364` | `+IN+OUT+RETURN` | `:84` / `:92` | 5 参（`nint,uint,In,Out,Return`） | ✅ |
| 无缓冲区 | `GenericCall8:365` | 同 `Call0` | `:100` | `(nint, uint)` | ✅ |
| ret | `PrimitiveCall9:366` / `CompoundCall9:367` | `+RETURN` | `:107` / `:115` | 3 参 | ✅ |
| in | `GenericCall10:368` | `+IN` | `:123` | 3 参 | ✅ |
| in+ret | `PrimitiveCall11:369` / `CompoundCall11:370` | `+IN+RETURN` | `:130` / `:138` | 4 参 | ✅ |
| in+out | `GenericCall14:371` | `+IN+OUT` | `:146` | 4 参 | ✅ |
| in+out+ret | `PrimitiveCall15:372` / `CompoundCall15:373` | 5 参 | `:154` / `:162` | 5 参 | ✅ |
| 无缓冲区 | `GenericCall16:374` | 同 `Call0` | `:170` | `(nint, uint)` | ✅ |
| in | `GenericCall18:375` | `+IN` | `:177` | 3 参 | ✅ |
| 无缓冲区 | `GenericCall24:376` | 同 `Call0` | `:184` | `(nint, uint)` | ✅ |
| in | `GenericCall26:377` | `+IN` | `:191` | 3 参 | ✅ |

---

## 6. 横向维度汇总

### 6.1 空指针与边界
| 检查项 | 结果 |
|---|---|
| C# 传入 `IntPtr` 是否为 0 的校验 | `FRegisterObject.cpp`（12 个实现体全部守卫）、`FRegisterFunction.cpp`（25 个实现体全部守卫）、`FRegisterProperty.cpp`（4 个实现体，经 `GetAddress` 间接）、`FRegisterStruct.cpp`（3 层嵌套守卫，仅 `Identical`）、`FRegisterClass.cpp`（`FoundClass` 已判空）都**通过 `GetObject`/`GetStruct`/`GetAddress` 的返回值判空**间接覆盖了零句柄。**例外 3 处**：`FRegisterScriptInterface.cpp:64-67`（F-INT1-001，P0）、`FRegisterClass.cpp:17-19` 的 `*Name`（F-INT1-002，P0）、`FRegisterObject.cpp:56` 的 `IScriptDomain::Get()`（F-INT1-013，P2，**新增**）。<br>⚠️ 口径说明：上文出现的 `12/12`／`3/3`／`25/25`／`4/4` 混用了"守卫行数"与"实现体个数"两种粒度（详见 F-INT1-012 复核证据）；按统一口径（grep `if \(const auto`），本组守卫共 **77 处**。 |
| 访问"可能为 null 的全局/静态访问器" | **1 处未判空**：`FRegisterObject.cpp:56` 的 `IScriptDomain::Get()`（F-INT1-013，P2，新增）；同目录另有 5 处同型（`FRegisterName.cpp:46`、`FRegisterString.cpp:46`、`FRegisterText.cpp:72`、`FRegisterUtf8String.cpp:50`、`FRegisterAnsiString.cpp:52`），而仓库内 30+ 处同类调用都写成 `if (const auto ScriptDomain = IScriptDomain::Get())`。 |
| `TMap::Find` 后是否直接 `*` 解引用 | **是，本组有 1 处**：`FStringRegistry.inl:25-27` 返回 nullptr 后由 `FRegisterClass.cpp:19` 直接 `*Name`。`GetMulti`/`GetStruct` 的所有**其它**调用点本组均已判空。 |
| `TArray` 索引 / `[]` | 本组 6 个文件**均无** `TArray` 下标访问、无 `GetData()`、无 `Memcpy`、无裸 `FMemory::Malloc`。 |
| 字符串长度参数 | `FRegisterStruct.cpp:15` / `FRegisterObject.cpp:31` 接收 `const char*` 并用 `UTF8_TO_TCHAR` 转换——**依赖 C# 侧传入以 `\0` 结尾的缓冲区**。C# 侧确实补了 `'\0'`（`StructImplementation.cs:15`、`ObjectImplementation.cs:22`），且 null 时传 `[0]`（单字节 0），**边界一致**。若未来有调用方不补 `\0`，`UTF8_TO_TCHAR` 会越界读。 |
| `TScriptInterface::GetObject()` 为空时的路径 | `FRegisterScriptInterface.cpp:67` 直接把结果交给 `Bind`；`BindImplementation(UObject*)` 对 nullptr 返回 `InvalidManagedHandle`（`FCSharpBind.inl:56-61`），C# 转成 `default`（`TScriptInterfaceImplementation.cs:38`）。**空对象本身安全**；致命的是上一行 `Multi` 为空（F-INT1-001）。 |
| `LoadObject` / `Cast` 失败 | 见 F-INT1-008（**已撤销：非缺陷**，`UStruct` 路径已确证安全——`FCSharpBind.cpp:106-111` 对 nullptr 直接 `return false`）。 |

### 6.2 内存所有权
| 资源 | 分配点 | 释放点 | 结论 |
|---|---|---|---|
| `new TScriptInterface<IInterface>` | `FRegisterScriptInterface.cpp:19` / `:21` | `FMultiRegistry.inl:66-71` `FMemory::Free`，经 `RemoveMultiReference`（`:57`）触发 | **配对，但两个缺口**：① 注册失败时泄漏（F-INT1-003，**P2**，由 P1 校正）；② 用 `FMemory::Free` 释放 `new` 出来的对象，**不调用析构函数**（`TScriptInterface` 是平凡类型，当前无实际损失，但一旦模板被用于非平凡类型即为 UB）。UE 中全局 `operator new` 亦映射到 `FMemory::Malloc`，因此分配器本身匹配。**补充**：本机 UE 5.6 下只有 `:19` 分支参与编译（`:21-25` 被 `UE_T_SCRIPT_INTERFACE_CONSTRUCTOR_U_OBJECT` 关掉）。 |
| `UObject` 引用计数 | `FRegisterObject.cpp:115` `AddReference` | `FRegisterObject.cpp:125` `RemoveReference` | 通过 C# `UObject.AddReference()/RemoveReference()` 显式配对（`Script/UE/CoreUObject/Object.cs:52/55`），**C# 侧无 `IDisposable`/终结器兜底**——忘记配对即泄漏引用（UObject 不会被 GC）。 |
| root 引用 | `FRegisterObject.cpp:89` `AddToRoot` | `FRegisterObject.cpp:97` `RemoveFromRoot` | 同上，显式配对，无自动兜底。`RemoveFromFunctionMap` 路径见 F-INT1-007。 |
| 结构体引用 | `FRegisterStruct.cpp:26` `Bind(handle,name)` | `FRegisterStruct.cpp:51` `RemoveStructReference`（**异步**） | 释放路径异步，见 F-INT1-005。 |
| 函数指针表 / 类型描述符 | 静态构造期由 `FClassBuilder` 持有 | 进程生命周期 | 无释放，设计如此。 |
| `GCHandle` | 本组 6 个文件**完全不涉及**（C++ 侧无 GCHandle；句柄由 `IManagedHandle` + 各 Registry 管理） | — | — |

### 6.3 重复代码与可宏化/模板化方案
1. **`FRegisterFunction.cpp` 25 个实现体**（`:11-348`）：形状 = 「取对象 → 取函数描述符 → `CallN<返回类型>(...)`」。差异只有 3 个正交维度（`N`、`Primitive|Compound|无`、缓冲区组合）。**方案**：X-macro 表 + 1 个可变参数模板，把 338 行压到约 40 行；或至少用 `#define REGISTER_CALL(Tag, N, Shape)` 消掉 25×13 行样板。
2. **`FRegisterProperty.cpp` 4 个实现体**（`:10-68`）：见 F-INT1-010。
3. **句柄守卫样板 77 处**（口径见 F-INT1-012 复核证据）：见 F-INT1-012。
4. **`const char* → FString` 转换重复 3 次**（与所列的 3 行一致）：grep 工具在 `Source/UnrealCSharp/Private/Domain/Interop` 搜 `UTF8_TO_TCHAR` 共 **10 行**，其中属本组 6 文件的是 `FRegisterObject.cpp:31`、`FRegisterStruct.cpp:15`、`FRegisterStruct.cpp:24` = **3 行**（其余 7 行在 `FRegisterAnsiString.cpp:18`、`FRegisterName.cpp:15`、`FRegisterString.cpp:15`、`FRegisterText.cpp:23/27/31`、`FRegisterUtf8String.cpp:18`）。三者都是 `InX != nullptr ? FString(UTF8_TO_TCHAR(InX)) : FString(TEXT(""))`。**方案**：`static FString ToFString(const char* In)` helper（建议放在公共头，供整个 `Domain/Interop` 复用）。
5. **注册期的「类构建器 + 一串 `.Function`」结构重复 6 次**：`TBindingClassBuilder<T>` 与 `FClassBuilder(TEXT("X"), NAMESPACE_LIBRARY)` 两种写法并存（见 §1 表格），且匿名 `struct FRegisterXxx` + `[[maybe_unused]]` 静态实例 + `PRAGMA_DISABLE_DANGLING_WARNINGS` 的包装方式在每个文件重抄一遍。**建议**统一为一个宏：
   ```cpp
   #define UNREALCSHARP_REGISTER(NAME, BUILDER_EXPR) \
       namespace { struct FRegister##NAME { FRegister##NAME() { BUILDER_EXPR; } }; \
       [[maybe_unused]] FRegister##NAME Register##NAME; }
   ```
   另一个不一致点：`FRegisterObject.cpp:10,152` 与 `FRegisterStruct.cpp:7,68` 用了 `PRAGMA_DISABLE_DANGLING_WARNINGS`/`PRAGMA_ENABLE_DANGLING_WARNINGS`，而 `FRegisterClass.cpp`、`FRegisterProperty.cpp`、`FRegisterFunction.cpp`、`FRegisterScriptInterface.cpp` **没有**——说明「哪些注册需要压制悬垂警告」并无统一标准。

### 6.4 性能
| 项 | 评估 |
|---|---|
| 注册期开销 | **一次性**（静态构造，模块加载时）。总量约 52 次 `Functions.Add`（**UE 5.6 下实际 51 次**，`ClassDefaultObject` 被宏编译掉）+ `FString::Printf` 名字合成（`FClassBuilder.cpp:87-91`）+ 类型信息构造。相对引擎启动可忽略，非热路径。 |
| 注册期重复字符串构造 | `GetFunctionImplementationName`（`FClassBuilder.cpp:83-91`）对每个名字都做 `FString::Printf` + `Algo::Count` 线性扫描 `Functions` 数组。25（FFunction）+12（UObject）次线性扫描，规模极小（O(n²) 但 n≤25），**可忽略**；若要洁癖可改用 `TMap<FString,int>` 计数。 |
| 热路径查找 | `FRegisterFunction.cpp` 全部 25 个实现体每次调用都执行 `GetOrAddFunctionDescriptor<FUnrealFunctionDescriptor>(InFunctionHash)`（如 `:16-18`）→ `ClassRegistry->GetOrAddFunctionDescriptor<T>()`（`FCSharpEnvironment.inl:31-34`），即**每次 C# → C++ 的函数调用都做一次 `TMap` 查找**（含 `uint32` hash 查找，通常 O(1) 但仍有桶探测 + 指针追逐）。这是本组唯一的真热路径。已确认**没有**在 C# 侧缓存函数指针（`__FFunction_*` 每次调用都经名字字典派发 + 本次查找）。若要优化，可在首次解析后把 `FUnrealFunctionDescriptor*` 缓存到 C# 侧，或把 hash→descriptor 的映射改为按类型内联缓存。**注意**：`GetOrAdd` 而非 `Get`，意味着 miss 时会**创建**描述符——热路径上出现意外 hash 会静默扩张注册表（潜在无界增长），未读到 `GetOrAddFunctionDescriptor` 实现故标为存疑。 |
| 重复 `FName` 构造 | `FRegisterClass.cpp:17` 每次 `RemoveFunction` 构造一次 `FName`（非热路径）；`FRegisterObject.cpp:54` 每次 `GetName` 做一次 `GetName()` + `TCHAR_TO_UTF8`（UTF-8 转换有分配，但 `GetName` 不是每帧路径）。 |
| 可缓存但已缓存 | `StaticClass`/`StaticStruct` 每次 `LoadObject` 看似昂贵，但 **C# 侧已缓存**：生成器输出 `StaticClassSingleton ??= ...`（`Script/SourceGenerator/UnrealTypeSourceGenerator.cs:223`）、`StaticStructSingleton ??= ...`（`:160`），所以 C++ 侧无需再加缓存 ✓。 |
| 循环内分配 | 本组 6 个文件**无循环**（除 `FClassBuilder.cpp` 的注册循环，非本组）。 |

---

## 7. 未覆盖 / 存疑项

> **存疑项状态**：下列第 1、2、5 条**已闭合**（标注闭合证据）；第 3、4、6、7、8、9 条仍开放；另有第 **10–13** 条为新登记的开放项。
1. ~~**`FCSharpBind::BindImplementation(UStruct*)` 的实现体未读**~~ → **【已闭合】** 该实现体真实存在于 `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:106-111`（`if (InStruct == nullptr) { return false; }`），声明在 `Public/Registry/FCSharpBind.h:52`。因此 `FRegisterObject.cpp:33-35` / `FRegisterStruct.cpp:17-19` 的 `LoadObject` 失败路径**确认安全**，F-INT1-008 的撤销结论由"推断"升级为"证据"。剩余建议（可省一次必然失败的 `LoadObject("")`）保留为可选优化。
2. ~~**`RETURN_BUFFER_SIGNATURE` / `IN_BUFFER_SIGNATURE` / `OUT_BUFFER_SIGNATURE` 宏展开未读**~~ → **【已闭合】** 已读 `Source/UnrealCSharpCore/Public/CoreMacro/BufferMacro.h`：`:9` `IN_BUFFER_SIGNATURE uint8* IN_BUFFER`、`:15` `OUT_BUFFER_SIGNATURE uint8* OUT_BUFFER`、`:21` `RETURN_BUFFER_SIGNATURE uint8* RETURN_BUFFER` —— **三个宏都没有长度参数**，F-INT1-004 的核心事实由宏定义直接证实（置信度由"中（间接证据）"升为"高"）。仍开放的部分只有"触发后果"（见第 10 条）。
3. **`FUnrealFunctionDescriptor::CallN<>` 实现未读**：无法判断 `IN_BUFFER`/`OUT_BUFFER`/`RETURN_BUFFER` 的长度校验与 `GetOrAddFunctionDescriptor` 的 miss 行为（是否无界增长）。
4. **`MethodBridge.cs:140-148` 名字查不到返回 0（空函数指针）** 这一前提**沿用任务背景中的既有结论**，未复验。若成立，则本组所有注册名若与 C# 声明不一致，都会变成「静默空指针调用」；本次 25↔25、12↔12 等比对均一致，故未发现触发点。
5. ~~**版本宏取值未核对**~~ → **【已闭合】** 已读 `Source/CrossVersion/Public/UEVersion.h`：`:148` `UE_U_CLASS_CLASS_DEFAULT_OBJECT !UE_VERSION_START(5, 6, 0)` ⇒ 本机 UE 5.6 下为 **0**，`FRegisterClass.cpp:45-47` 的 `.Property("ClassDefaultObject", ...)` **不参与编译**；`:196` `UE_T_SCRIPT_INTERFACE_CONSTRUCTOR_U_OBJECT !UE_VERSION_START(5, 8, 0)` ⇒ 本机为 **1**，`FRegisterScriptInterface.cpp:19` 分支生效、`:20-25` 分支不编译。`UE_VERSION_START` 的展开定义见同文件 `:5-6`。这两点已回写进 §3.2 / §3.5 / §5.2#13 / F-INT1-006 / F-INT1-009。
6. **`HandleData.Alloc/GetObject` 的句柄生命周期未读**（C# `Script/Interop/`）：`TScriptInterfaceImplementation.cs:14-15` 对 `InType` 做 `HandleData.Alloc(InType)` 后**未见释放**，是否泄漏取决于 `HandleData` 的实现，属 C# 侧报告范围。
7. **`FReflectionRegistry::Get().GetClass(InManagedType)`（`FRegisterScriptInterface.cpp:28`）的返回语义未读**：已确认其 nullptr 传入 `AddMultiReference` 无害（`FMultiRegistry.inl:37-52` 的 `InClass` 在 `IsMember==false` 时未被使用），但 `GetClass` 自身是否可能崩未验证。
8. **未做动态验证**：本报告全部为静态阅读。F-INT1-001/002 的崩溃路径未实际运行复现（无编译/运行环境授权）。
9. **本组 6 个文件的 `.h`/`.inl` 头（`TBindingClassBuilder.inl`、`TPropertyBuilder.inl`、`BindingMacro.h` 的 `BINDING_FUNCTION`/`BINDING_OVERLOAD` 展开）只读了被调用到的片段**，未做完整阅读；`BINDING_OVERLOAD` 的参数名数组 `TArray<FString>{"bCreateIfNeeded"}`（`FRegisterClass.cpp:50`）如何变成 C# 参数名未验证（但与生成的 `Class.cs:13` 的 `bCreateIfNeeded` 名字一致 ✓）。
10. **【新增】F-INT1-004 的"栈溢出"后果未构造用例**：宏已证无长度参数（见第 2 条），但"改小 `stackalloc byte[N]` 后越界写"未实跑；`PropertyDescriptor->Get` 实际写入字节数取决于 `FPropertyDescriptor` 各子类实现（不在本报告范围）。**卡点**：需要能运行/调试 C# 与 C++ 联调的环境。
11. **【新增】F-INT1-013 的空指针窗口未做实机验证**：`IScriptDomain::Get()` 在 `FScriptDomainFactory::Destroy`（`:58`）后确定返回 `nullptr` 是源码事实，但"该窗口内确有 `UObject.GetName()` 调用"未验证。**卡点**：需要触发"域销毁后仍有 C# → C++ 调用"的真实时序（编辑器关闭 / PIE 退出 / 热重载）。建议在 `FRegisterObject.cpp:56` 前加 `check(ScriptDomain)` 跑一轮编辑器关闭流程。
12. **【新增】F-INT1-005 的非游戏线程调用方仍未取证**：整个 `Domain/Interop/` 目录 `IsInGameThread|check\(|ensure` 命中 0（实测），说明代码未做线程判定；但"C# 侧确实存在非游戏线程调用"需要读 C# 侧 `HandleData`/`Task` 相关代码才能定论（属其它报告范围）。**卡点**：跨报告，未越界核对。
13. **2 万级 grep 命中数未逐条复算**：§5.2 的 #20–#23（22651/22651/20786/20786）只做了"实证调用点存在 + 判定为活"的验证，精确数字与正文一致（见 §5.2 核对表末行）。**不影响的结论**：4 行的"活"判定已由真实调用点独立支持。

