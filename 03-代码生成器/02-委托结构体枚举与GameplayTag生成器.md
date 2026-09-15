# ScriptCodeGenerator 生成器群分析（委托 / 结构体 / 枚举 / BindingEnum / GameplayTag）

> 本报告各条**严重度见各 Finding 的「复核结论」字段**。**终值**：16 条 = **确认 10**（`F-GEN2-001/003/005/008/009/010/011/014/015/016`）+ **部分确认 6**（`F-GEN2-002/004/006/007/012/013`，偏差逐条写在各条"复核结论"里）+ 证伪 0 + 无法验证 0；级别变动 **2 条**：上调 1（`F-GEN2-014` P2→P1）、下调 1（`F-GEN2-004` P2→P3）；可达性 活跃 14 / 潜伏 1（`F-GEN2-010`）/ 不可达 1（`F-GEN2-002`）。四处**事实/计数更正**：`F-GEN2-002` `NameSpaceContent[0]` **总数 8 处**（本文件 2 处 + `FBindingClassGenerator.cpp` 6 处；`03-…/01b` F-GEN-012 记 4 处，两份报告都漏了 `:927`/`:1001`，**建议统一裁决为 8 处**）、`F-GEN2-004` 的 `TSet` 迭代机制（**插入序/稠密数组**，非哈希桶序）、`F-GEN2-007` 调用点 **6 → 7 处**、`F-GEN2-013` 缺头注释的生成点**不止 1 处**（多播委托模板同样缺）。





> 分析范围：`Plugins/UnrealCSharp/Source/ScriptCodeGenerator`（Public/Private）中 5 个"生成器"子系统
> 覆盖文件：10 个（5 组 .h/.cpp）

> 状态：已完成（全部文件逐行读完；另交叉阅读了基类 `FGeneratorCore`、`FUnrealCSharpFunctionLibrary`、C# 侧运行时与**真实生成产物** `UnrealCSharpTest/Script/{UE,Game}/Proxy/**` 作为地面真值）

---

## 0. 覆盖范围与阅读清单

| 文件 | 行数 | 是否读完 | 备注 |
|---|---|---|---|
| `Source/ScriptCodeGenerator/Public/FDelegateGenerator.h` | 18 | 读完 | |
| `Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp` | 719 | 读完 | 任务描述写 584 行，实际 719 行 |
| `Source/ScriptCodeGenerator/Public/FStructGenerator.h` | 11 | 读完 | |
| `Source/ScriptCodeGenerator/Private/FStructGenerator.cpp` | 351 | 读完 | 任务描述写 294 行，实际 351 行 |
| `Source/ScriptCodeGenerator/Public/FEnumGenerator.h` | 38 | 读完 | |
| `Source/ScriptCodeGenerator/Private/FEnumGenerator.cpp` | 306 | 读完 | 任务描述写 248 行，实际 306 行 |
| `Source/ScriptCodeGenerator/Public/FBindingEnumGenerator.h` | 12 | 读完 | |
| `Source/ScriptCodeGenerator/Private/FBindingEnumGenerator.cpp` | 67 | 读完 | |
| `Source/ScriptCodeGenerator/Public/FGameplayTagGenerator.h` | 27 | 读完 | |
| `Source/ScriptCodeGenerator/Private/FGameplayTagGenerator.cpp` | 282 | 读完 | 任务描述写 224 行，实际 282 行 |
| 交叉阅读（非本人负责，仅作契约/证据） | | | `FGeneratorCore.h/.inl/.cpp`、`FUnrealCSharpFunctionLibrary.h/.cpp`、`NameEncode.cpp`、`ScriptCodeGenerator.Build.cs`、`UnrealCSharpEditor.cpp`、`FRegisterDelegate.cpp`、`FRegisterMulticastDelegate.cpp`、`MulticastDelegateHandler.cpp/.h`、`FCSharpBind.cpp`、`FRegisterProperty.cpp`、`FBindingEnum.h`/`FBindingEnumRegister.cpp`/`TTypeInfo.inl`/`TNameSpace.inl`、`Script/UE/Library/FDelegateImplementation.cs`、`FMulticastDelegateImplementation.cs` |
| **地面真值（关键）** | | | 工程实际生成物 `<Project>/Script/UE/Proxy/**`（16055 个 .cs）与 `Script/Game/Proxy/**`（75 个 .cs）——本报告多条发现由真实产物反证/证实 |

未读文件（不属于我）：`FClassGenerator`、`FBindingClassGenerator`、`FAssetGenerator`、`FCodeAnalysis`、`FSolutionGenerator`、`FDoxygenConverter`（只在需要确认调用关系时 grep）。

**行数口径（注记）**：本表用的是**总行数**（`(Get-Content $f).Count`，含空行）；若用 `Measure-Object -Line`（**忽略空行**）会得到明显更小的数字。上表 10 个文件逐个 `read` 到文件尾复读确认，**10/10 全部吻合**，故表中"任务描述写 N 行、实际 M 行"的差异属**口径差异**（简报用的可能是非空行数），不是缺陷。产物规模口径同理：`Script/Game/Proxy/**` glob 实测 **75 个 .cs**（吻合）；UE 侧 16055 需 shell 统计，**未复算**。
**编号连续性**：本篇 `F-GEN2-001` … `F-GEN2-016` 共 **16 个，连续无重号**（`grep '^### \[F-'` = 16 命中）；`F-GEN2-*` 前缀为本报告独有，**与其它报告无撞车**（§7.1 列出的两处跨报告撞车是 `F-INT2-*` 与 `F-PROP2-*`）。

---

## 1. 模块职责与架构速览

### 1.1 五个生成器在流水线中的位置

生成入口只有一处：`FUnrealCSharpEditorModule::Generator()`（`Source/UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp:314`）。固定顺序（同一函数内）：

```
Solution Generator     (335)
Code Analysis          (339)  → FDynamicGenerator::CodeAnalysisGenerator (343)
FGeneratorCore::BeginGenerator()   (345)   ← 加载设置/OverrideFunctions
Class Generator        (349)  FClassGenerator::Generator()
Struct Generator       (353)  FStructGenerator::Generator()      ← 本报告
Enum Generator         (357)  FEnumGenerator::Generator()        ← 本报告
Asset Generator        (361)  FAssetGenerator::Generator()
GameplayTag Generator  (365)  FGameplayTagGenerator::Generator() ← 本报告
BindingClass Generator (374)
BindingEnum Generator  (378)  FBindingEnumGenerator::Generator() ← 本报告
FGeneratorCore::EndGenerator()     (380)  ← 清空各生成器静态缓存 + 删除未登记的旧文件
CollectGarbage / Compile           (384/388)
```

另一条入口：GameplayTag 单独刷新 `OnEditorRefreshGameplayTagTree()`（`UnrealCSharpEditor.cpp:258-268`）→ 只调 `FGameplayTagGenerator::Generator()`，且*不*调用 `BeginGenerator/EndGenerator`。

**顺序是本报告的关键**：`GetPropertyType(FProperty*)` 在为类/结构体属性、函数参数生成类型名的同时会顺带注册枚举的底层类型（`FGeneratorCore.cpp:89`、`:170` → `FEnumGenerator::AddEnumUnderlyingType`）。因此"枚举被生成为什么宽度"取决于**在它自己被生成之前**是否已有别的东西引用过它。见 `F-GEN2-005`。

### 1.2 各生成器的数据流

| 生成器 | 输入 | 输出位置 | 去重/幂等手段 |
|---|---|---|---|
| `FDelegateGenerator` | `FDelegateProperty*` / `FMulticastDelegateProperty*`（来自类/结构体属性遍历、容器内层属性） | `GetFileName()`（`FGeneratorCore.inl:10-29`）= `<GenerationPath>/<ModuleName>/<ModuleRelativePath>/F<签名名>.cs` | 静态 `TSet<TTuple<FString,FString>> Delegate`，键 =（C# 命名空间, C# 类名）（`FDelegateGenerator.cpp:41-46 / 409-414`） |
| `FStructGenerator` | `TObjectIterator<UScriptStruct>`（排除 `UUserDefinedStruct`）+ `FAssetGenerator` 对 BP 结构体的单独调用 | `<GenerationPath>/<ModuleName>/<ModuleRelativePath>/<StructName>.cs` | 无（每个结构体只生成一次） |
| `FEnumGenerator` | `TObjectIterator<UEnum>`（排除 `UUserDefinedEnum`）+ 显式一次 `GeneratorCollisionChannel()` | 同上，`<EnumName>.cs` | 无 |
| `FBindingEnumGenerator` | `FBinding::Get().Register().GetEnums()`（C++ 手工绑定枚举，`BINDING_ENUM` 宏 + `TBindingEnumBuilder`） | `<GenerationPath("/" + 命名空间点转斜杠)>/Binding/<短名>.cs` | 无（枚举来自注册表） |
| `FGameplayTagGenerator` | `UGameplayTagsManager::Get().GetFilteredGameplayRootTags` 递归出的 `FGameplayTagNode` 树 | `<GameProxy>/<ProjectName>/GameplayTags.cs` | 无（每次全量重写/删除） |

文件写入统一走 `FUnrealCSharpFunctionLibrary::SaveStringToFile()`（`FUnrealCSharpFunctionLibrary.cpp:1150-1179`）：先比对旧内容，相同则直接返回 `true`（不标记 dirty）；否则 `MarkScriptChanged()`（**在写盘之前**）+ `CreateDirectoryTree` + `ForceUTF8WithoutBOM` 写入。各生成器在写盘前调用 `FGeneratorCore::AddGeneratorFile(FileName)` 登记，`EndGenerator→DeleteRemainGeneratorFiles()`（`FGeneratorCore.cpp:1050-1079`）删除 Proxy 目录下未登记的同后缀文件。

### 1.3 委托生成器的运行时对端（理解生成代码所必需）

- 单播：`FDelegate_*` → `Source/UnrealCSharp/Private/Domain/Interop/FRegisterDelegate.cpp`（C++ 绑定注册为 `Script.Library.FDelegate`）+ `Script/UE/Library/FDelegateImplementation.cs`（C# partial 声明）。
- 多播：`FMulticastDelegate_*` → `FRegisterMulticastDelegate.cpp` + `FMulticastDelegateImplementation.cs`。
- 调用索引由 `FGeneratorCore::GetFunctionIndex(bHasReturn,bHasInput,bHasOutput,bIsNative,bIsNet)`（`FGeneratorCore.cpp:681-689`）= 位组合生成，方法名 = `F{Delegate|MulticastDelegate}_{Generic|Primitive|Compound}Execute{Index}Implementation` / `...Broadcast{Index}Implementation`。

---

## 2. 关键调用链

1. `UnrealCSharpEditor.cpp:349 FClassGenerator::Generator()` → `FClassGenerator.cpp:152/436 FDelegateGenerator::Generator(FProperty*)` → `FDelegateGenerator.cpp:16-23` 分派 → `Generator(FDelegateProperty*)`(`:26`) / `Generator(FMulticastDelegateProperty*)`(`:394`) → `FGeneratorCore::GetFileName` → `FUnrealCSharpFunctionLibrary::SaveStringToFile`。
2. `UnrealCSharpEditor.cpp:353 FStructGenerator::Generator()` → `FStructGenerator.cpp:20-26`（`TObjectIterator<UScriptStruct>`）→ `Generator(const UScriptStruct*)`(`:29`) → `FStructGenerator.cpp:204 FDelegateGenerator::Generator(*PropertyIterator)`（结构体内的委托）与 `:217 FGeneratorCore::GetPropertyType`（→ 通过 `ArrayProperty->Inner` 等递归到 `FDelegateGenerator::Generator`）。
3. `UnrealCSharpEditor.cpp:357 FEnumGenerator::Generator()` → `FEnumGenerator.cpp:12-18` → `Generator(const UEnum*)`(`:23`) → `GetEnumUnderlyingTypeName`(`:250`) → `FEnumGenerator.cpp:20 GeneratorCollisionChannel()`(`:173`) → `SaveStringToFile`。
4. `FGeneratorCore.cpp:89/170 GetPropertyType` → `FEnumGenerator::AddEnumUnderlyingType` → 静态 `FEnumGenerator::EnumUnderlyingType` →（被）`FEnumGenerator.cpp:265` 读取；`FGeneratorCore::EndGenerator`(`:1040`) 清空。
5. `UnrealCSharpEditor.cpp:378 FBindingEnumGenerator::Generator()` → `FBindingEnumGenerator.cpp:9 FBinding::Get().Register().GetEnums()` → `Generator(const FBindingEnum*)`(`:15`) → `NameSpaceContent[0]`(`:52`) → `SaveStringToFile`。
6. C# 侧调用回 C++：`FOnPickUp.cs:39 Clear()` → `FMulticastDelegate_ClearImplementation` → `FRegisterMulticastDelegate.cpp:139` → `FMulticastDelegateHelper::Clear` → `UMulticastDelegateHandler::Clear()`(`MulticastDelegateHandler.cpp:142`，`MulticastScriptDelegate->Clear()` + `DelegateWrappers.Empty()`)。
7. 委托生命周期：`FOnPickUp.cs:11 构造函数` → `FMulticastDelegate_RegisterImplementation` → `FRegisterMulticastDelegate.cpp:14` → `FCSharpBind::Bind<FMulticastDelegateHelper>`；`FOnPickUp.cs:13 析构(finalizer)` → `UnRegisterImplementation`(`:21`) → `AsyncTask(GameThread, RemoveDelegateReference<...>)`。
8. 结构体属性访问：`ActorSequenceObjectReference.cs:55 GetStructProperty` → `FRegisterProperty.cpp:40 GetStructPropertyImplementation` → `FCSharpEnvironment::GetOrAddPropertyDescriptor(InPropertyHash)`；该 hash 由 `FCSharpBind.cpp:167-175` 在绑定时用 `GetTypeHash(FProperty*)` 反射写回生成类里的 `private static uint __Type = 0;`（`FStructGenerator.cpp:282-287` 生成该占位字段）。

---

## 3. 发现清单

### P0

未发现 P0 级问题（无必然崩溃/数据损坏路径；下面 P1-001 会产生错误数据，但只在特定签名形态下触发）。

---

### P1

### [F-GEN2-001] 单播委托 `Execute` 回写第 2 个及以后的引用型 out 参数时指针运算缺少括号，取到错误的对象句柄

- **类别**: Bug
- **严重度**: **P1**（功能错误/错误数据）
- **复核结论**: ✅确认 —— 代码事实、调用上下文、类别、严重度全部成立（逐行推演指针算术 + 真实产物双证据）
- **可达性**: 活跃
- **复核证据**: `Source/ScriptCodeGenerator/Private/FGeneratorCore.cpp:625-633` 非原生分支格式串 = `"\n%s%s = (%s)HandleData.GetObject(*(nint*)%s%s);\n"`，偏移量 `%s` 被拼在解引用 `*` **之内**（原生分支 `:616-624` 为 `*(%s*)(%s%s)`，括号正确）；`Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:263-277` 以 `" + %d"` 把 `GetBufferSize` 偏移传进去。产物 `Script/UE/Proxy/Engine/FViewportDisplayCallback.cs:36`（偏移为空 → `*(nint*)OutBuffer`，侥幸正确）与 `:38`（`*(nint*)OutBuffer + 8`，缺陷）逐字印证。UE 侧真签名 = `<Engine 5.6 安装目录>/Engine/Source/Runtime/Engine/Classes/Engine/ViewportStatsSubsystem.h:17`：`DECLARE_DYNAMIC_DELEGATE_RetVal_TwoParams(bool, FViewportDisplayCallback, FText&, OutText, FLinearColor&, OutColor);`（正文 L144 的宏名少了 `_DYNAMIC_` 与参数名，精确形式以此为准；该声明位于**文件作用域**，正好解释产物命名空间是 `Script.Engine` 而非 `Script.Engine.UViewportStatsSubsystem`）。**同缺陷另有 2 个使用点，已定位到行**：`Source/ScriptCodeGenerator/Private/FClassGenerator.cpp:723`、`Source/ScriptCodeGenerator/Private/FBindingClassGenerator.cpp:597` → 类/绑定类生成产物同样被污染
- **级别变动**: 无
- **文件**: `Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:263`（触发点）→ `Source/ScriptCodeGenerator/Private/FGeneratorCore.cpp:625-633`（缺陷所在），生成物证据 `Script/UE/Proxy/Engine/FViewportDisplayCallback.cs:38`
- **函数**: `FDelegateGenerator::Generator(FDelegateProperty*)` → `FGeneratorCore::GetOutParam(bool, const FString&, const FString&, const FString&, const FString&, const FString&)`
- **置信度**: 高（生成器源码 + 真实产物 + 多播路径正确写法三者互证）

**现状（代码事实）**

```cpp
// FDelegateGenerator.cpp:259-278  （单播：为每个 ref 参数拼回写语句）
for (auto Index = 0; Index < DelegateRefParamIndex.Num(); ++Index)
{
    const auto Param = DelegateParams[DelegateRefParamIndex[Index]];

    ExecuteFunctionOutParamBody += FGeneratorCore::GetOutParam(
        Param,
        FUnrealCSharpFunctionLibrary::Encode(Param),
        FGeneratorCore::GetPropertyType(Param),
        OUT_BUFFER_TEXT,
        BufferSize == 0 ? TEXT("") : *FString::Printf(TEXT(" + %d"), BufferSize),  // :268-274
        TEXT("\t\t\t\t"));
    BufferSize += FGeneratorCore::GetBufferSize(Param);                            // :277
}
```

```cpp
// FGeneratorCore.cpp:611-634  （非原生分支：偏移量被拼在解引用*之内*）
	return bIsPrimitive
		       ? FString::Printf(TEXT(
			       "\n%s%s = *(%s*)(%s%s);\n"                     // 正确：*(T*)(BUF + N)
		       ), ...)
		       : FString::Printf(TEXT(
			       "\n%s%s = (%s)HandleData.GetObject(*(nint*)%s%s);\n"  // :625-633 ← 缺括号
		       ), *InIndent, *InName, *InPropertyType, *InBuffer, *InOffset);
```

真实生成物（`FViewportDisplayCallback` 的 UE 签名是 `DECLARE_DELEGATE_RetVal_TwoParams(bool, FViewportDisplayCallback, FText&, FLinearColor&)`，两个都是 ref）：

```csharp
// Script/UE/Proxy/Engine/FViewportDisplayCallback.cs:30-40
var OutBuffer = stackalloc byte[16];
var ReturnBuffer = stackalloc byte[1];
FDelegateImplementation.FDelegate_PrimitiveExecute7Implementation(HandleData.GetHandle(this), InBuffer, OutBuffer, ReturnBuffer);
OutText  = (FText)HandleData.GetObject(*(nint*)OutBuffer);          // :36 偏移为空 → 侥幸正确
OutColor = (FLinearColor)HandleData.GetObject(*(nint*)OutBuffer + 8); // :38 ← (*(nint*)OutBuffer) + 8
return *(bool*)ReturnBuffer;
```

`*(nint*)OutBuffer + 8` 按 C# 运算符优先级解析为 `(*(nint*)OutBuffer) + 8`：先读出第 1 个 out 参数的**对象句柄**，再把它当整数加 8，然后拿这个"句柄+8"去 `GetObject`。正确写法应为 `*(nint*)(OutBuffer + 8)`。

**调用上下文**
`FDelegateGenerator::Generator(FDelegateProperty*)` 由 `FClassGenerator.cpp:152/436`、`FStructGenerator.cpp:204`、`FGeneratorCore.cpp:159/232/234/246`（数组/Map/Set 内层）在编辑器代码生成阶段、GameThread 上调用；`GetOutParam` 是 `FGeneratorCore` 的公开静态成员，`FClassGenerator` / `FBindingClassGenerator` 生成函数调用回写时同样使用它（grep：`FGeneratorCore::GetOutParam` 在本文件外另有调用点，见"验证方式"），因此同一缺陷会同时污染类函数生成产物。

**问题**
任何**单播**委托，只要签名同时满足「有返回值或 void」+「≥2 个非 const 引用/out 参数」+「第 2 个及其后的 out 参数为引用型（非原生标量）」，生成的 C# 在 `Execute()` 里回写这些参数时就会拿到一个非法的句柄：
- `HandleData.GetObject(badHandle)` 大概率返回 `null`（强转后为 `null`），调用方得到一个空对象/`NullReferenceException`；
- 若该"句柄+8"恰好命中另一个已注册对象，则**静默写入/读出完全不相干的对象**（数据损坏，且无异常）。
原生标量 out 参数（走 `*(int*)(BUF + 4)` 分支）不受影响，第 1 个 out 参数（偏移为空）也不受影响——这解释了为什么该缺陷能在测试工程中长期存在：需要"两个及以上引用型 out 参数"的委托极少。

**建议**
把偏移拼进括号内（`FGeneratorCore.cpp:625-633`）：
```cpp
: FString::Printf(TEXT("\n%s%s = (%s)HandleData.GetObject(*(nint*)(%s%s));\n"),
                  *InIndent, *InName, *InPropertyType, *InBuffer, *InOffset);
```
多播生成器的手写同义代码已是正确形式，可与之统一：`FDelegateGenerator.cpp:583-596` 的 `"%s = *(%s*)(%s%s);"`。建议顺便把多播这段也改成调用 `GetOutParam`，消除两套实现。

**验证方式**
- grep 生成物：`Select-String -Path Script/UE/Proxy/**/*.cs -Pattern 'GetObject\(\*\(nint\*\)\w+ \+ '`，命中即缺陷（当前工程命中 `FViewportDisplayCallback.cs:38`，可用该文件作为回归用例）。
- 反查生成器：grep `GetOutParam` 在 `Source/` 下的调用点（`FDelegateGenerator.cpp:263`、`FClassGenerator`/`FBindingClassGenerator` 内数处），修复后需重新生成全部产物再 diff。
- 功能用例：C# 侧对 `FViewportDisplayCallback` 绑定一个把 `OutText/OutColor` 设为已知值的实现，`Execute` 后断言两个 out 参数都被正确回写。

---

### P2

### [F-GEN2-002] `FBindingEnumGenerator` 对 `NameSpaceContent[0]` 无边界检查，`TypeInfo` 未设置时越界读

- **类别**: 未定义行为
- **严重度**: **P3**（隐患；当前编辑器配置下不可达，但无任何断言保护）
- **复核结论**: ⚠️部分确认（偏差：**原文"确认只有本文件直接用 `[0]`"不成立** —— `FBindingClassGenerator.cpp` 有 6 处同款 `NameSpaceContent[0]`；全插件真实总数 **8 处**，既非本报告的 2 处，也非 `03-…/01b` F-GEN-012 记的 4 处。越界机制与严重度本身成立）
- **可达性**: 不可达
- **复核证据**: `grep 'NameSpaceContent\[0\]' Plugins/UnrealCSharp/Source` = **8 命中**：`Source/ScriptCodeGenerator/Private/FBindingEnumGenerator.cpp:52`、`:59`（本条的两处，行号吻合）+ `Source/ScriptCodeGenerator/Private/FBindingClassGenerator.cpp:288`、`:686`、`:713`、`:730`（= 01b F-GEN-012 已记的 4 处）+ `Source/ScriptCodeGenerator/Private/FBindingClassGenerator.cpp:927`、`:1001`（**两份报告都漏记**，是同一函数家族的第二份拷贝）。空数组路径仍由 `Source/UnrealCSharpCore/Public/Binding/TypeInfo/FBindingTypeInfo.h:22-27` 的 `static TArray<FString> Instance` 提供 → 越界读结论不变。原文 L231 "此处是唯一的例外"与 L245 验证方式"确认只有本文件直接用 `[0]`"两句已被本字段更正（未改动那两句原文以外的字句）
- **级别变动**: 无（原 P2→P3 的校正成立，维持 P3）
- **文件**: `Source/ScriptCodeGenerator/Private/FBindingEnumGenerator.cpp:52`、`:59`
- **函数**: `FBindingEnumGenerator::Generator(const FBindingEnum*)`
- **置信度**: 中（越界路径确凿；"是否会真的发生"取决于编译期 `WITH_TYPE_INFO` 与注册代码，见下）

**现状（代码事实）**

```cpp
// FBindingEnumGenerator.cpp:15-21
void FBindingEnumGenerator::Generator(const FBindingEnum* InEnum)
{
	const auto& NameSpaceContent = InEnum->GetTypeInfo().GetNameSpace();
	auto ClassContent = InEnum->GetEnum();
	...
// :52  *NameSpaceContent[0]                     ← 无 IsEmpty() 检查
// :59  FPaths::Combine(FUnrealCSharpFunctionLibrary::GetGenerationPath(
//          TEXT("/") + NameSpaceContent[0].Replace(TEXT("."), TEXT("/"))), ...)
```

```cpp
// Binding/TypeInfo/FBindingTypeInfo.h:22-27
const TArray<FString>& GetNameSpace() const
{
    static TArray<FString> Instance;                 // ← 空数组
    return TypeInfo != nullptr ? TypeInfo->GetNameSpace() : Instance;
}
```

```cpp
// Binding/TypeInfo/FBindingTypeInfoRegister.h:13-16
explicit operator FBindingTypeInfo() const
{
    return FBindingTypeInfo(TypeInfoFunction.IsSet() ? TypeInfoFunction.GetValue()() : nullptr);
}
// Binding/Enum/TBindingEnumBuilder.inl:13-19：TypeInfo 函数只在 #if WITH_TYPE_INFO 下传入
// CoreMacro/BindingMacro.h:15： #define WITH_TYPE_INFO WITH_EDITOR
```

**调用上下文**
`FBindingEnumGenerator::Generator()` 由 `UnrealCSharpEditor.cpp:378` 在编辑器生成流程中调用；枚举来自 `FBinding::Get().Register().GetEnums()`（`FBindingEnumGenerator.cpp:9`），注册在 `TBindingEnumBuilder` 的静态构造里（如 `Source/UnrealCSharp/Private/Domain/Interop/FRegisterGuid.cpp:15`、`FRegisterDateTime.cpp:17/34`、`FRegisterObjectFlags.h:13`、`FRegisterUnreal.cpp:19`、`FRegisterWorld.cpp:20`、`FRegisterForceInit.h:12`）。

**问题**
`TArray::operator[]` 在 Shipping 下不做边界检查（只有 `checkSlow`/`RangeCheck` 在调试构建生效）。当 `TypeInfo` 为 `nullptr` 时 `GetNameSpace()` 返回一个静态空数组，`NameSpaceContent[0]` 读的是空数组的 `Data[0]`（未分配指针的越界访问）→ 得到垃圾 `FString` → `.Replace()`/`FPaths::Combine` 可能崩溃或生成乱码路径，且**不会有任何日志**。
可达条件：`WITH_TYPE_INFO` 为 0（`WITH_EDITOR` 为 0 的目标，例如打包 Runtime 目标里编译的 `UnrealCSharp` 运行时模块）时 `TypeInfoRegister` 是空 optional；此时 `FBindingEnum::TypeInfo` 恒为 nullptr。当前工程的生成动作只发生在编辑器进程（`UnrealCSharpEditor` 是 Editor 模块），因此**实际不会命中**——但同一份头文件同时被 Runtime 目标编译，这是一种"靠调用环境兜底"的隐式契约；另外若将来有任何 `BINDING_ENUM` 注册漏传 `TTypeInfo`，编辑器下也会立刻越界。`FGameplayTagGenerator` 等其它生成器对空值都做了显式防御；但就 `NameSpaceContent[0]` 这一写法而言，**此处并非唯一的例外** —— `FBindingClassGenerator.cpp:288`/`:686`/`:713`/`:730`/`:927`/`:1001` 另有 6 处同款（全插件共 **8 处**，详见本条"复核证据"）。

**建议**
```cpp
const auto& NameSpaceContent = InEnum->GetTypeInfo().GetNameSpace();
if (NameSpaceContent.IsEmpty() || InEnum->GetEnum().IsEmpty())
{
    UE_LOG(LogUnrealCSharp, Warning, TEXT("Skip binding enum without type info: %s"), *InEnum->GetEnum());
    return;
}
```
或让 `FBindingEnum` 在注册时 `check(TypeInfo.IsSet())`，把契约前移到注册点。

**验证方式**
grep `NameSpaceContent[0]`（**不是** `GetNameSpace()` —— 后者有 6 处以上调用点，不能用来证明"只有本文件用 `[0]`"）；实测全插件 **8 命中**：本文件 2 处 + `FBindingClassGenerator.cpp` 6 处；在 `WITH_TYPE_INFO=0` 的配置下（或用 `#if !WITH_TYPE_INFO` 临时构造）运行一次生成流程并观察崩溃；加 `checkf(!NameSpaceContent.IsEmpty(), ...)` 可让问题在编辑器下立即显形。

---

### [F-GEN2-003] Binding 枚举平铺写入同一个 `Binding/` 目录，不同命名空间的同名枚举互相覆盖

- **类别**: Bug
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 路径拼接事实、平铺后果、类别、严重度全部成立（产物侧逐文件核对）
- **可达性**: 活跃
- **复核证据**: `Source/ScriptCodeGenerator/Private/FBindingEnumGenerator.cpp:58-62`：`DirectoryName = GetGenerationPath("/" + NameSpaceContent[0].Replace(".","/")) + GetBindingDirectory()`，`FileName = Combine(DirectoryName, ClassContent) + CSHARP_SUFFIX` —— 文件名只取枚举短名。产物实证：`Script/UE/Proxy/Binding/` 下恰为 `EForceInit.cs / EDayOfWeek.cs / EMonthOfYear.cs / EGuidFormats.cs / EObjectFlags.cs / ELoadFlags.cs / ESpawnActorNameMode.cs`（**与原文列举逐字一致**，且**无任何命名空间子目录**）；`Script/Game/Proxy/Binding/` 另有 `ERawTestEnum.cs`、`ERawTestEnumClass.cs` → 同一 `Binding/` 目录承载 `Script.CoreUObject` 与 `Script.UnrealCSharpTest` 两个命名空间，短名相同即覆盖
- **级别变动**: 无（原 P2→P3 的校正成立，维持 P3）
- **文件**: `Source/ScriptCodeGenerator/Private/FBindingEnumGenerator.cpp:58-62`
- **函数**: `FBindingEnumGenerator::Generator(const FBindingEnum*)`
- **置信度**: 中（路径拼接逻辑确凿；是否已有真实冲突取决于注册的枚举集合）

**现状（代码事实）**

```cpp
// FBindingEnumGenerator.cpp:58-66
auto DirectoryName = FPaths::Combine(
    FUnrealCSharpFunctionLibrary::GetGenerationPath(TEXT("/") + NameSpaceContent[0].Replace(TEXT("."), TEXT("/"))),
    FUnrealCSharpFunctionLibrary::GetBindingDirectory());          // ← 只用 NameSpaceContent[0] 决定生成根，再拼固定的 "Binding"

const auto FileName = FPaths::Combine(DirectoryName, ClassContent) + CSHARP_SUFFIX;  // ← 文件名只取枚举短名
FGeneratorCore::AddGeneratorFile(FileName);
FUnrealCSharpFunctionLibrary::SaveStringToFile(FileName, Content);
```

真实产物的确全部平铺在一个目录（无命名空间子目录）：`Script/UE/Proxy/Binding/{EForceInit,EDayOfWeek,EMonthOfYear,EGuidFormats,EObjectFlags,ELoadFlags,ESpawnActorNameMode}.cs`（同时项目侧 `Script/Game/Proxy/Binding/` 也有一套）。

**问题**
文件路径 = `"Binding" + "/" + 枚举短名 + ".cs"`，与枚举的**完整 C# 命名空间无关**（命名空间只写进文件内容，不参与路径）。两个不同命名空间下同名的 binding 枚举（例如游戏模块的 `EGuidFormats` 与引擎的 `EGuidFormats`）会命中同一个路径：
1. 后写的文件**静默覆盖**先写的文件（两者都已被 `AddGeneratorFile` 登记，`DeleteRemainGeneratorFiles` 不会发现异常）；
2. 实际编译进去的只有一个类型的定义，另一处的 C# 引用会解析到错误类型（或在同命名空间下变成重复定义）。
另注：`GetGenerationPath(TEXT("/") + NameSpaceContent[0]...)` 传的是**命名空间**而非包路径（`"Script.CoreUObject"` → `/Script/CoreUObject`），`FUnrealCSharpFunctionLibrary::GetGenerationPath` 只用它区分 UE/Game 两个 Proxy 根（`FUnrealCSharpFunctionLibrary.cpp:984-1011`：`Splits[0]=="Script" && ProjectModuleList.Contains(Splits[1])`），因此 `Script.CoreUObject` 会落到 UE Proxy —— 这一点是"恰好正确"，因为它把命名空间当包名解析，仅当命名空间第二段恰好等于项目模块名时行为会反转。

**建议**
把命名空间并入路径，与 `FGeneratorCore::GetFileName` 保持一致的层级策略：
```cpp
auto DirectoryName = FPaths::Combine(
    FUnrealCSharpFunctionLibrary::GetGenerationPath(InEnum->IsProjectEnum() ? TEXT("/Game") : TEXT("/Script/") + NameSpaceContent[0]),
    FUnrealCSharpFunctionLibrary::GetBindingDirectory(),
    NameSpaceContent[0].Replace(TEXT("."), TEXT("/")));
```
更稳的做法是在 `Generator()`（`:7-13`）里维护 `TSet<FString> UsedPaths`，路径冲突时报 `UE_LOG(Warning)` 而不是覆盖。

**验证方式**
grep 所有 `BINDING_ENUM`/`TBindingEnumBuilder<T>` 注册点（`Source/UnrealCSharp/Private/Domain/Interop/FRegister*.cpp` 与工程侧 `Source/UnrealCSharpTest/Private/UnitTest/Core/FRawTestStruct.cpp:13/26`），列出 `TName<T,T>::Get()` 短名，检查是否有重复；或在 `Generator()` 开头断言 `!AllPaths.Contains(FileName)`。

---

### [F-GEN2-004] 生成文件的 `using` 指令顺序由 `TSet<FString>` 遍历顺序决定（未排序），输出不可复现

- **类别**: 可优化/可读性（确定性）
- **严重度**: **P3**（版本库噪声 + 生成内容不能由内容集合唯一决定 + 触发无意义重编译）
- **复核结论**: ⚠️部分确认（偏差：**机制解释写错了** —— 原文"'`TSet` 迭代顺序取决于哈希表槽位布局（由哈希值与表容量决定），不是插入序"**不成立**。UE 的 `TSet` 把元素存放在稠密数组 `Elements` 中，迭代器直接遍历该数组，**顺序即插入序**，哈希桶只用于查找。现象（同一 `using` 集合出现多种顺序）与建议（输出前排序）仍然成立，但成因是"各文件**插入顺序**不同"）
- **可达性**: 活跃
- **复核证据**: 引擎 `<Engine 5.6 安装目录>/Engine/Source/Runtime/Core/Public/Containers/Set.h:1451-1476`（`TBaseIterator` 持有 `ElementItType ElementIt`，`operator++` 只做 `++ElementIt`，`GetId()` 返回 `ElementIt.GetIndex()`）；`:566`、`:627`、`:652`、`:2025` 均为 `Elements.AddUninitialized(...)` 追加；`:1426-1448` 的 `Rehash()` 只重建 `Hash` 并按 `Elements` 顺序重新哈希，**不重排 `Elements`**。本报告 5 处遍历点行号全部吻合：`FDelegateGenerator.cpp:347-353`、`:670-676`、`FStructGenerator.cpp:301-307`、`FEnumGenerator.cpp:87-93`、`:216-222`
- **级别变动**: **P2→P3**（机制更正后，同一份反射数据下顺序是确定的，噪声只来自插入集合变化，性质属"可重复性/风格"；与 `03-…/01b` F-GEN-005 对同类 `using` 顺序问题的定级 P3 保持一致）
- **文件**: `Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:347-353` 与 `:670-676`；`Source/ScriptCodeGenerator/Private/FStructGenerator.cpp:301-307`；`Source/ScriptCodeGenerator/Private/FEnumGenerator.cpp:87-93` 与 `:216-222`
- **函数**: `FDelegateGenerator::Generator(FDelegateProperty*)` / `Generator(FMulticastDelegateProperty*)`、`FStructGenerator::Generator(const UScriptStruct*)`、`FEnumGenerator::Generator(const UEnum*)` / `GeneratorCollisionChannel()`
- **置信度**: 高（有实测证据）

**现状（代码事实）**

```cpp
// FDelegateGenerator.cpp:60-64  （TSet 收集）
TSet<FString> UsingNameSpaces{
    COMBINE_NAMESPACE(NAMESPACE_ROOT, NAMESPACE_CORE_UOBJECT),
    COMBINE_NAMESPACE(NAMESPACE_ROOT, NAMESPACE_LIBRARY),
    NAMESPACE_INTEROP
};
// FDelegateGenerator.cpp:343-353  （直接按 TSet 迭代顺序拼接输出）
UsingNameSpaces.Remove(UsingNameSpaceContent);
UsingNameSpaces.Remove(TEXT(""));
for (auto UsingNameSpace : UsingNameSpaces)
{
    UsingNameSpaceContent += FString::Printf(TEXT("using %s;\n"), *UsingNameSpace);
}
```
（另外 `:96`/`:227`/`:461` 用 `GetPropertyTypeNameSpace()` 返回的 `TSet` 做 `Append`；`GetPropertyTypeNameSpace` 内部是 `TMap`/`TSet` 的 `Union`，顺序同样不可控。）

**调用上下文**
这 5 处都在生成阶段（GameThread，编辑器启动/手动生成）执行，结果既写入磁盘（`SaveStringToFile`）又参与 `SaveStringToFile` 的"内容是否相同"判断（`:1156-1162`）——顺序一变，内容就被判定不同 → `MarkScriptChanged()` 被调用 → 触发 C# 重编译。

**问题**
`TSet` 的迭代顺序取决于哈希表槽位布局（由哈希值与表容量决定），**不是插入序、也不是字典序**。相同 `using` 集合在不同文件里已经出现多种顺序——对全部 16055+75 个生成文件做"using 集合相同则顺序应相同"的检查，**446 个集合中有 119 个存在多种顺序**，例如：

```
SET: using Interop; | using Script.Binding; | using Script.CoreUObject; | using Script.Library;
   ORDER A(3 个文件): Script.Binding, Interop, Script.Library, Script.CoreUObject
   ORDER B(3 个文件): Script.Binding, Interop, Script.CoreUObject, Script.Library
SET: ... | using Script.RigVM;
   ORDER A(1): CoreUObject, Interop, Engine, RigVM, Library
   ORDER B(1): CoreUObject, Interop, Library, RigVM, Engine
   ORDER C(5): CoreUObject, Library, Interop, RigVM, Engine
```
这证明顺序是哈希表布局的副产物而非内容决定。后果：(1) 任何无关改动（例如新增一个命名空间、或 `Remove()` 改变了槽位）都可能整段重排 `using`，产生 VCS 噪声；(2) 生成结果无法按内容比对复现；(3) 由于 `SaveStringToFile` 以字节相等作为"无需重写"的判据，这类重排会额外触发一次全量重编译。

**建议**
输出前排序，一处修复即可覆盖 5 个生成器：
```cpp
TArray<FString> Sorted = UsingNameSpaces.Array();
Sorted.Sort();
for (const auto& UsingNameSpace : Sorted) { ... }
```
（`FGameplayTagGenerator` 已经这么做了：`:158-162` 用 `GetKeys(Keys); Keys.Sort();`，可作为一致性参照。）

**验证方式**
对比两次生成产物：`git diff` 中出现的纯 `using` 行序变化即为本问题；或对生成目录执行上面的"集合→顺序"分组脚本（本报告使用的脚本：读取每个文件前 15 行中 `^using .*;$` 行，按排序后的集合分组，统计不同顺序数）。

---

### [F-GEN2-005] 枚举底层类型回退为 `long`，且取决于"谁先引用过它"的生成顺序

- **类别**: 未定义行为（潜伏的宽度不一致）
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 回退值、顺序依赖机制、后果链（缓冲宽度 vs `*(Enum*)` 解引用）全部成立；统计数未复算
- **可达性**: 活跃
- **复核证据**: `Source/ScriptCodeGenerator/Private/FEnumGenerator.cpp:265-270`（`Find` 未中 → `IsA(UUserDefinedEnum) ? "byte" : "long"`，行号吻合）、`:61`（唯一使用点）、`:87-93`（using 循环，行号吻合）；注册点唯一：`Source/ScriptCodeGenerator/Private/FGeneratorCore.cpp:89`（ByteProperty 分支）、`:170`（EnumProperty 分支），"缓存首值生效"见 `FEnumGenerator.cpp:130-133`。宽度链路：`FGeneratorCore.cpp:469-478` `GetBufferSize` = 原生取 `Property->GetElementSize()` / 非原生取 `sizeof(void*)`；`FGeneratorCore.cpp:592-602` `GetReturn` 原生分支 = `*(%s*)%s`（按 C# 枚举类型名解引用）。产物对照：`Script/UE/Proxy/ActorSequence/EActorSequenceObjectReferenceType.cs:11` = `public enum EActorSequenceObjectReferenceType : byte`，而 `Script/UE/Proxy/ActorSequence/ActorSequenceObjectReference.cs:53-57` 正是 `stackalloc byte[1]` + `return *(EActorSequenceObjectReferenceType*)ReturnBuffer;` —— 其安全只因该枚举在结构体生成阶段已被注册为 `byte`。419 / 1697 / 229 三个统计数需 shell 全量扫描，**未复算**
- **级别变动**: 无
- **文件**: `Source/ScriptCodeGenerator/Private/FEnumGenerator.cpp:250-271`（回退）与 `:61`（使用点）；注册点 `Source/ScriptCodeGenerator/Private/FGeneratorCore.cpp:89`、`:170`
- **函数**: `FEnumGenerator::GetEnumUnderlyingTypeName(const UEnum*)`、`FEnumGenerator::AddEnumUnderlyingType(const UEnum*, const FNumericProperty*)`
- **置信度**: 中（机制确凿并有实测统计；"是否已产生实际错误值"在本工程中为否，见下）

**现状（代码事实）**

```cpp
// FEnumGenerator.cpp:250-271
FString FEnumGenerator::GetEnumUnderlyingTypeName(const UEnum* InEnum)
{
	static TMap<EEnumUnderlyingType, FString> EnumUnderlyingTypeName
	{ {None,"byte"},{Int8,"sbyte"},{UInt8,"byte"},{Int16,"short"},{UInt16,"ushort"},
	  {Int,"int"},{UInt32,"uint"},{Int64,"long"},{UInt64,"ulong"} };      // :252-263 静态局部（OK：C++11 magic static）

	if (const auto FoundEnumUnderlyingType = EnumUnderlyingType.Find(InEnum))     // :265
	{
		return EnumUnderlyingTypeName[*FoundEnumUnderlyingType];                  // :267 operator[] 读静态 TMap（会插入默认值）
	}

	return InEnum->IsA(UUserDefinedEnum::StaticClass()) ? TEXT("byte") : TEXT("long");   // :270 ← 未知时一律 long
}
```
```cpp
// 唯一注册点（FGeneratorCore.cpp）
if (ByteProperty->Enum != nullptr) { FEnumGenerator::AddEnumUnderlyingType(ByteProperty->Enum, ByteProperty); ... }   // :89
if (const auto EnumProperty = CastField<FEnumProperty>(Property)) { FEnumGenerator::AddEnumUnderlyingType(EnumProperty->GetEnum(), EnumProperty->GetUnderlyingProperty()); ... }  // :170
```

**调用上下文**
`FEnumGenerator::Generator()` 在流水线中排在第 6 位（`UnrealCSharpEditor.cpp:357`），而 `FClassGenerator`(349)、`FStructGenerator`(353) 都在它**之前**，`FAssetGenerator`(361) 在它**之后**。也就是说：
- 被"类/结构体属性、函数参数"引用过的枚举 → 底层类型已注册 → 生成正确的 `byte`/`int` 等；
- 没被引用过的枚举 → `long`；
- **只被资源（BP 类属性/BP 结构体）引用的原生枚举 → 生成时还没被引用**（资源生成在后面），于是枚举文件先落地为 `long`，随后 `FAssetGenerator` 为 BP 属性生成访问器时又按 1 字节处理。

真实统计：`public enum X : long` 的生成文件 **419** 个（`byte` 1697、`int` 229）。反向检查：这 419 个枚举名在**任何**非枚举生成文件中都没有被引用（`*(Name*)` 强转命中 0 文件、名字引用命中 0 文件）——即当前工程里这些 `long` 枚举从未进入 C# 互操作面，所以尚未产生错误值；但这是**顺序巧合**而非设计保证。

为什么危险：反射路径的访问器缓冲宽度取自 C++ 属性大小（`FGeneratorCore::GetBufferSize` → `Property->GetElementSize()` = 1 字节，`FGeneratorCore.cpp:469-478`），而读回语句是"按 C# 枚举类型解引用"（`GetReturn` 的原生分支 `*(%s*)%s`，`FGeneratorCore.cpp:592-602`，`PropertyType` 就是枚举的 C# 类型名）。若枚举声明为 `: long`，生成的 `stackalloc byte[1]` 上就会执行 `*(Enum*)buf` → **越界读 7 字节**，高位是栈垃圾 → 枚举值随机化。实例对照：`Script/UE/Proxy/ActorSequence/ActorSequenceObjectReference.cs:53-57` 正是这一形态（`stackalloc byte[1]` + `return *(EActorSequenceObjectReferenceType*)ReturnBuffer;`），它之所以安全，只因为该枚举在结构体生成阶段已被注册为 `byte`：

```
EActorSequenceObjectReferenceType.cs → public enum EActorSequenceObjectReferenceType : byte
```

**建议**
不要用"是否被引用过"作为推断依据，改为显式判定：
```cpp
// UEnum::GetCppForm()/GetUnderlyingType() 在 UE5 可用；退一步也可从 UEnum 的 CppType 元数据取
#if UE_F_ENUM_GET_CPP_FORM
const auto CppForm = InEnum->GetCppForm();   // EnumClass / Namespaced / Regular
#endif
const auto Underlying = InEnum->GetUnderlyingType();  // 反射可直接给出真实底层类型
```
若必须保留启发式，至少把回退值改为 `byte`（UE 绝大多数 UENUM 为 uint8）并在命中回退时打 `UE_LOG(LogUnrealCSharp, Warning, ...)`，让"推断失败"可观测；同时把 `FEnumGenerator::Generator()` 的调用点移到 `FClassGenerator` **之前**（或在 `BeginGenerator` 里先做一次全量枚举扫描填充 `EnumUnderlyingType`），消除顺序依赖。

**验证方式**
grep 生成目录 `public enum \w+ : long` 统计数量（本次实测 419）；对每个 `: long` 枚举检查是否出现在任何非枚举生成文件中（本次实测 0）；断言式验证：在 `GetEnumUnderlyingTypeName` 的回退分支加 `UE_LOG`，跑一次全量生成看命中数是否等于 419。

---

### [F-GEN2-006] 委托按（命名空间, 类名）去重后**静默丢弃**同名委托，且非原生多播委托的命名空间不含类名，放大了碰撞面

- **类别**: Bug（静默丢失）
- **严重度**: **P2**
- **复核结论**: ⚠️部分确认（偏差：去重逻辑与非原生命名空间截断**确凿成立**，但本工程产物中**未发现**任何碰撞实例，与原文置信度"中"一致；正文"极易触发"的强度表述偏高）
- **可达性**: 活跃
- **复核证据**: `Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:41-46`（单播：`Delegate.Contains({NameSpaceContent, ClassContent})` 后裸 `return`，无日志，行号吻合）、`:409-414`（多播同款，行号吻合）；命名空间计算 `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:616-658`，其中 `:630-644` 正是"原生 → `GetClassNameSpace(Class) + "." + Class->GetName()`；非原生 → 只取 `GetClassNameSpace(Class)`"，行号吻合。产物两侧实证：非原生（BP）多播 = `Script/Game/Proxy/UnrealCSharpTest/FOnPickUp.cs:5`、`FOnUseItem.cs:5` 均为 `namespace Script.UnrealCSharpTest`（模块级、**不含类名** → 碰撞面确实更大）；原生多播 = `Script/UE/Proxy/UMG/Widget/Components/FGetText.cs:10` = `namespace Script.UMG.Widget`（含类名）。`ClassContent` 规则见 `FUnrealCSharpFunctionLibrary.cpp:585-614`（`"F" + 去掉 __DelegateSignature 后的签名名`）
- **级别变动**: 无
- **文件**: `Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:41-46`（单播）、`:409-414`（多播）；命名空间计算 `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:630-644`
- **函数**: `FDelegateGenerator::Generator(FDelegateProperty*)`、`FDelegateGenerator::Generator(FMulticastDelegateProperty*)`
- **置信度**: 中（去重逻辑确凿；具体碰撞需要工程侧存在同名不同签名的委托，未在本工程产物中发现实例）

**现状（代码事实）**

```cpp
// FDelegateGenerator.cpp:37-46
auto NameSpaceContent = FUnrealCSharpFunctionLibrary::GetClassNameSpace(InDelegateProperty);
auto ClassContent = FUnrealCSharpFunctionLibrary::GetFullClass(InDelegateProperty);   // "F" + 签名名去掉 _DelegateSignature

if (Delegate.Contains({NameSpaceContent, ClassContent})) { return; }   // :41-44  直接 return，无日志
Delegate.Add({NameSpaceContent, ClassContent});                        // :46
```
```cpp
// FUnrealCSharpFunctionLibrary.cpp:630-644（多播的命名空间）
if (const auto Class = Cast<UClass>(SignatureFunction->GetOuter()))
{
    if (InMulticastDelegateProperty->IsNative())
    {
        return FString::Printf(TEXT("%s.%s"), *GetClassNameSpace(Class), *Class->GetName());  // 原生：含类名
    }
    else
    {
        return *GetClassNameSpace(Class);                                                     // 非原生：省略类名 ← 碰撞面更大
    }
}
```

**问题**
`(Namespace, Class)` 是去重键，也是 C# 类型标识。当两个**签名不同**的委托给出同一个键时，第二个被直接丢弃：
- 生成的 C# 类型只有第一个签名；第二个委托属性在类里以第一个类型出现 → 编译期类型不匹配，或在签名"恰好兼容"时**静默语义错误**；
- 对 **BP（非原生）多播委托**，命名空间被截断到模块级（`Script.Game`、`Script.<Project>.<Folder>`），因此不同 BP 类里同名的 `OnSomething__DelegateSignature` 会产生同名 `FOnSomething` 且命名空间相同 → 极易触发。
本工程产物规模（`FOnPickUp.cs`、`FOnUseItem.cs` 等在 `Script/Game/Proxy/UnrealCSharpTest/` 下）证明 BP 委托的命名空间确实只有模块/目录级（`namespace Script.UnrealCSharpTest`），与上述分析一致。

**建议**
1. 丢弃分支加日志与计数，避免"静默"：
```cpp
if (Delegate.Contains(Key))
{
    UE_LOG(LogUnrealCSharp, Warning, TEXT("Duplicate delegate signature skipped: %s.%s"), *NameSpaceContent, *ClassContent);
    return;
}
```
2. 若希望同名不同签名都可用，键应包含**签名指纹**（返回类型 + 各参数类型串），并在碰撞时改名为 `FOnSomething_2` 之类；
3. 非原生多播的命名空间应与单播一致地带上 `Class->GetName()`（`FUnrealCSharpFunctionLibrary.cpp:642`），减少不必要的碰撞。

**验证方式**
grep 生成目录中同名 `.cs`（不同目录）且内容不同的文件：`Get-ChildItem -Recurse -Filter 'F*.cs' | Group-Object Name | Where Count -gt 1`；对同一命名空间内重复的类名做 `Select-String 'public class F'` 统计；加日志后跑一次全量生成，观察是否存在命中。

---

### [F-GEN2-007] 7 处 `SaveStringToFile` 返回值全部未检查，写盘失败静默（可能残留旧文件参与编译）

- **类别**: 异常与错误处理
- **严重度**: **P2**
- **复核结论**: ⚠️部分确认（偏差：**计数少了一处** —— 原文标题与正文写"6 处"，本报告负责的 5 个生成器实际共 **7 处**；缺陷机制、后果链、严重度均成立）
- **可达性**: 活跃
- **复核证据**: `grep 'SaveStringToFile\(' Plugins/UnrealCSharp/Source/ScriptCodeGenerator` = **12 命中**，其中 5 个生成器占 **7 处**（其余 5 处在 `FSolutionGenerator.cpp:162`、`FClassGenerator.cpp:871`、`FBindingClassGenerator.cpp:739/1011`、`FAssetGenerator.cpp:191`，属他人文件）。被调方 `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1150-1179` **逐字吻合**原文引文：`:1156-1162` 内容相同即 `return true`、`:1165-1167` `MarkScriptChanged()` 在写盘**之前**、`:1171-1175` `CreateDirectoryTree` 返回值同样未检查、`:1177-1178` `ForceUTF8WithoutBOM` 的返回值即本函数返回值。登记点核对：5 个生成器写盘前均调 `FGeneratorCore::AddGeneratorFile`（`FDelegateGenerator.cpp:389/716`、`FStructGenerator.cpp:348`、`FEnumGenerator.cpp:118/245`、`FBindingEnumGenerator.cpp:64`、`FGameplayTagGenerator.cpp:92`）
- **级别变动**: 无
- **文件**: `Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:391`、`:718`；`Source/ScriptCodeGenerator/Private/FStructGenerator.cpp:350`；`Source/ScriptCodeGenerator/Private/FEnumGenerator.cpp:120`、`:247`；`Source/ScriptCodeGenerator/Private/FBindingEnumGenerator.cpp:66`；`Source/ScriptCodeGenerator/Private/FGameplayTagGenerator.cpp:94`
- **函数**: 各生成器的 `Generator(...)`
- **置信度**: 高

**现状（代码事实）**

```cpp
// FDelegateGenerator.cpp:387-391（其余 5 处形态完全相同）
const auto FileName = FGeneratorCore::GetFileName(InDelegateProperty);
FGeneratorCore::AddGeneratorFile(FileName);
FUnrealCSharpFunctionLibrary::SaveStringToFile(FileName, Content);   // ← 返回 bool，被丢弃
```
```cpp
// FUnrealCSharpFunctionLibrary.cpp:1150-1179
bool FUnrealCSharpFunctionLibrary::SaveStringToFile(const FString& InFileName, const FString& InString)
{
	... 内容相同则 return true ...
#if WITH_EDITOR
	MarkScriptChanged();                       // ← 先标脏，再写盘
#endif
	if (const auto DirectoryName = FPaths::GetPath(InFileName); !PlatformFile.DirectoryExists(*DirectoryName))
	{
		PlatformFile.CreateDirectoryTree(*DirectoryName);       // ← 返回值也未检查
	}
	return FFileHelper::SaveStringToFile(InString, *InFileName, FFileHelper::EEncodingOptions::ForceUTF8WithoutBOM, FileManager, FILEWRITE_None);
}
```

**调用上下文**
6 处调用点都在编辑器代码生成阶段；生成完成后 `FCSharpCompiler::Get().ImmediatelyCompile(...)`（`UnrealCSharpEditor.cpp:388`）会立即编译，其中依赖这些文件的类/结构体引用。

**问题**
`FFileHelper::SaveStringToFile` 返回 `false` 表示**根本没有落盘**（目录创建失败、磁盘满、路径过长、文件被 IDE/杀软独占）。此时：
1. `AddGeneratorFile(FileName)` 已经把该路径登记进 `GeneratorFiles` → `DeleteRemainGeneratorFiles`（`FGeneratorCore.cpp:1050-1079`）**不会删除**该路径上可能存在的**旧版本文件**；
2. 旧文件会继续参与 C# 编译 → 用户看到的是"改了 C++/蓝图但 C# 侧行为没变"或类型不匹配的错误，且日志里没有任何线索；
3. `MarkScriptChanged()` 在写盘**之前**调用，即使写入失败也照样标记"脚本已变化" → 触发一次无意义的编译。
路径长度方面：`GetFileName` 会把 `GetGenerationPath()`（工程磁盘绝对路径）+ 模块名 + `GetModuleRelativePath` 全部拼上（`FGeneratorCore.inl:21-28`），深目录工程很容易接近 Windows 260 字符限制，这一点是现实触发条件之一。

**建议**
```cpp
const auto FileName = FGeneratorCore::GetFileName(InDelegateProperty);
FGeneratorCore::AddGeneratorFile(FileName);
if (!FUnrealCSharpFunctionLibrary::SaveStringToFile(FileName, Content))
{
    UE_LOG(LogUnrealCSharp, Error, TEXT("Save generated file failed: %s"), *FileName);
    FGeneratorCore::RemoveGeneratorFile(FileName);   // 或提供"记录失败集合"的机制
}
```
并给 `SaveStringToFile` 内部补 `UE_LOG(LogUnrealCSharp, Error, ...)`（当前连底层都不报错）；把 `MarkScriptChanged()` 移到写盘成功之后。

**验证方式**
单元/手动：把某个生成目标目录设为只读（或用一个超长路径），跑一次生成，观察是否既无日志、又保留了旧文件；`grep -n "SaveStringToFile(" Source/ScriptCodeGenerator/Private/*.cpp` 逐一确认返回值使用情况。

---

### [F-GEN2-008] GameplayTag：读不到任何标签时**静默删除**已生成文件

- **类别**: 异常与错误处理
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 空树即删文件、无任何日志、删除失败同样静默，全部成立；并**新增交叉核对**：该显式删除在全量生成路径下与 `FGeneratorCore::DeleteRemainGeneratorFiles()` 重复（空树时该文件不会被 `AddGeneratorFile` 登记），**只有** `UnrealCSharpEditor.cpp:258-268` 的 GameplayTag 刷新入口（不调 Begin/EndGenerator）才单独依赖它
- **可达性**: 活跃
- **复核证据**: `Source/ScriptCodeGenerator/Private/FGameplayTagGenerator.cpp:43-55` 逐行吻合（`:45-52` `FileExists` + 仅当 `Delete` 成功才 `MarkScriptChanged`；`:54` 裸 `return`；**全文件 `UE_LOG` 0 命中**）；文件路径构造 `:37-41`（`GetGameProxyDirectory() / FApp::GetProjectName() / GameplayTags.cs`，`GAMEPLAY_TAGS_NAME` 见 `Source/UnrealCSharpCore/Public/CoreMacro/Macro.h:35`）。两个入口：`UnrealCSharpEditor.cpp:262`（刷新入口，随后按 `IsScriptChanged()` 决定编译，`:264-267`）与 `:365`（全量，`:380` `EndGenerator()` → 默认 `bIsFull=true`，见 `Source/ScriptCodeGenerator/Public/FGeneratorCore.h:71` → `FGeneratorCore.cpp:1044` → `:1050-1079` 清扫未登记 `.cs`）。交叉：`03-…/01` F-GENC-002（P2/部分确认/活跃）讲的是同一次清扫，与本条互补不重复
- **级别变动**: 无
- **文件**: `Source/ScriptCodeGenerator/Private/FGameplayTagGenerator.cpp:24-55`（判定与删除在 `:43-55`）
- **函数**: `FGameplayTagGenerator::Generator()`
- **置信度**: 中（行为确凿；触发概率依赖工程/命令行上下文）

**现状（代码事实）**

```cpp
// FGameplayTagGenerator.cpp:24-55
FGameplayTagTreeNode Root;
TArray<TTuple<FString, FString>> Tags;
CollectGameplayTags(Tags);                       // :30  → 读 UGameplayTagsManager
for (const auto& [Tag, Comment] : Tags) { GeneratorTag(Root, Tag, Comment); }

const auto DirectoryName = FPaths::Combine(FUnrealCSharpFunctionLibrary::GetGameProxyDirectory(), FApp::GetProjectName());
const auto FileName = FPaths::Combine(DirectoryName, GAMEPLAY_TAGS_NAME + CSHARP_SUFFIX);

if (Root.Children.IsEmpty())
{
    if (auto& FileManager = IFileManager::Get(); FileManager.FileExists(*FileName))
    {
        if (FileManager.Delete(*FileName)) { FUnrealCSharpFunctionLibrary::MarkScriptChanged(); }   // :48-51
    }
    return;                                    // :54 ← 无任何日志/告警
}
// :250-260
void FGameplayTagGenerator::CollectGameplayTags(...)
{
	TArray<TSharedPtr<FGameplayTagNode>> RootTags;
	UGameplayTagsManager::Get().GetFilteredGameplayRootTags(TEXT(""), RootTags);   // :254
	for (const auto& RootTag : RootTags) { VisitTagNodes(RootTag, OutTags); }
}
```

**调用上下文**
两个入口：`UnrealCSharpEditor.cpp:262`（`OnEditorRefreshGameplayTagTree`，编辑器刷新 Tag 树时，随后按 `IsScriptChanged()` 决定是否编译）与 `:365`（全量生成流程）。`VisitTagNodes` 只收集 `IsExplicitTag() && !IsRestrictedGameplayTag()` 的节点（`:267`），因此"标签被全部改成 Restricted/隐式"或"Tag 源未加载"都会得到空结果。

**问题**
空结果有两种完全不同的成因，代码不区分：
- 合法：工程确实没有标签 → 删除旧文件是**正确**行为（避免留下无用的空类）；
- 异常：Tag 源未加载/被过滤（例如从 Commandlet 或尚未加载 Tag ini 的时机调用、`DefaultGameplayTags.ini` 解析失败、所有标签都是 Restricted）→ **静默删掉**此前生成的 `GameplayTags.cs`，用户 C# 里所有 `GameplayTags.Ability.Type.Action` 引用立刻变成编译错误，而生成日志里没有任何"我删了文件"的记录。删除失败（文件被占用）时同样静默（不标记 dirty、不报错），旧文件继续被编译。

**建议**
```cpp
if (Root.Children.IsEmpty())
{
    if (FileManager.FileExists(*FileName))
    {
        const bool bDeleted = FileManager.Delete(*FileName);
        UE_LOG(LogUnrealCSharp, Warning,
               TEXT("No gameplay tags collected; %s existing generated file '%s'."),
               bDeleted ? TEXT("deleted") : TEXT("failed to delete"), *FileName);
        if (bDeleted) { FUnrealCSharpFunctionLibrary::MarkScriptChanged(); }
    }
    return;
}
```
另建议在 `CollectGameplayTags` 中区分"管理器未初始化"（`UGameplayTagsManager::Get()` 前先判断 `IsInitialized()`/Tag 表数量）与"确实为空"两种情况，前者不应删文件。

**验证方式**
临时把 `DefaultGameplayTags.ini` 中的 `+GameplayTagList` 行全部注释 → 观察 `GameplayTags.cs` 被删除且无任何日志；`Select-String -Path Source/**/FGameplayTagGenerator.cpp -Pattern 'UE_LOG'` 当前为 0 命中，可作为"无日志"的证据。

---

### [F-GEN2-009] GameplayTag：父子同名的标签段生成 `class X { ... X ... }` → C# CS0542

- **类别**: Bug（生成非法 C#）
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 生成逻辑（局部 `UsedNames` 只保留 `Value`、不含外层类型名）+ C# CS0542 规则 + 引擎侧标签合法性三者已闭合
- **可达性**: 活跃
- **复核证据**: `Source/ScriptCodeGenerator/Private/FGameplayTagGenerator.cpp:151-156`（`UsedNames` 仅 `Add("Value")`）、`:183-189`（叶子 → 字段）、`:193-200`（有子节点 → `public static class %s`）、`MakeUniqueName` `:234-248` —— 行号全部吻合。引擎侧：`<Engine 5.6 安装目录>/Engine/Source/Runtime/GameplayTags/Private/GameplayTagsManager.cpp:2225-2299` 的 `IsValidGameplayTagString` 只拒绝空串、首尾 `.`/空格、非法字符 → **`Foo.Foo` 是合法标签**，父子同名确实可注册。产物印证 `Value` 形态：`Script/Game/Proxy/UnrealCSharpTest/GameplayTags.cs:122-124`（`class Cosmetic { FGameplayTag Value = "Cosmetic" }`）。当前工程 `Config/DefaultGameplayTags.ini:15-115` 共 101 条标签中**无 `X.X` 形态**，故目前不触发
- **级别变动**: 无
- **文件**: `Source/ScriptCodeGenerator/Private/FGameplayTagGenerator.cpp:148-232`（生成），`MakeUniqueName` 在 `:234-248`
- **函数**: `FGameplayTagGenerator::GeneratorChildren(FString&, const FGameplayTagTreeNode&, int32)`
- **置信度**: 中（生成逻辑与 C# 规则确凿；是否已有真实标签触发取决于工程标签表）

**现状（代码事实）**

```cpp
// FGameplayTagGenerator.cpp:148-162
void FGameplayTagGenerator::GeneratorChildren(FString& OutContent, const FGameplayTagTreeNode& InNode, const int32 InIndent)
{
	TSet<FString> UsedNames;
	if (!InNode.Tag.IsEmpty()) { UsedNames.Add(TEXT("Value")); }   // :153-156 只保留 "Value"
	TArray<FString> Keys;
	InNode.Children.GetKeys(Keys);
	Keys.Sort();                                                   // :162 顺序确定（好）
	...
		const auto Name = MakeUniqueName(FUnrealCSharpFunctionLibrary::Encode(Keys[Index], false), UsedNames);   // :175
		if (Child.Children.IsEmpty())
		{
			// :183-189  叶子 → 字段
			Content += FString::Printf(TEXT("%spublic static readonly FGameplayTag %s = new FGameplayTag { TagName = \"%s\" };\n"), *Pad, *Name, *Child.Tag);
		}
		else
		{
			// :193-200  有子节点 → 类
			Content += FString::Printf(TEXT("%spublic static class %s\n%s{\n"), *Pad, *Name, *Pad);
```

**问题**
`UsedNames` 是**每个节点独立**的局部集合，只防"兄弟重名"和与 `Value` 成员重名，**不包含外层类型的名字**。于是标签 `Foo.Foo`（或 `Foo.Foo.Bar`）会生成：

```csharp
public static class Foo            // FGameplayTagGenerator.cpp:193-200
{
    public static readonly FGameplayTag Foo = new FGameplayTag { TagName = "Foo.Foo" };  // :183-189 ← 与外层类型同名
}
```
C# 规则 **CS0542：成员名不能与其所属类型同名** → 整份 `GameplayTags.cs` 编译失败（而生成器自身不会报错，因为这只是"文本拼接"）。同理，嵌套类与外层类同名也报错。这是本模块唯一会**直接导致编译失败**的命名冲突；报告中"`A.B_C` 与 `A_B.C` 会映射成同一个名字"的假设经核实**不成立**（见 §5 的否定结论），但"父段与子段同名"是真实存在的同类风险。

**建议**
把外层类型名一并纳入保留集：
```cpp
FString FGameplayTagGenerator::GeneratorChildren(FString& OutContent, const FGameplayTagTreeNode& InNode,
                                                 const int32 InIndent, const FString& InEnclosingName)
{
    TSet<FString> UsedNames;
    if (!InNode.Tag.IsEmpty()) { UsedNames.Add(TEXT("Value")); }
    if (!InEnclosingName.IsEmpty()) { UsedNames.Add(InEnclosingName); }   // ← 新增
    ...
    // 递归调用处传入 *Name：GeneratorChildren(Content, Child, InIndent + 1, Name);
```
（注意根节点也要传 `TEXT("GameplayTags")`，因为 `public static class GameplayTags` 内的字段/类不能叫 `GameplayTags`。）同样建议在重命名发生时打 `Warning` 日志，便于用户发现"标签名被改了"。

**验证方式**
构造标签 `Test.Test`（在 `DefaultGameplayTags.ini` 加 `+GameplayTagList=(Tag="Test.Test")`），跑一次生成并对该文件执行 `dotnet build`/`csc` 单文件编译，应能复现 CS0542；或对现有产物做静态检查：对每个 `public static class X` 块，检查其直接成员中是否存在名为 `X` 的字段/类。

---

### P3

### [F-GEN2-010] 多播委托的参数收集未过滤 `CPF_ReturnParm`，与单播路径不对称

- **类别**: 可读性/一致性
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 单播/多播两条参数收集路径的不对称成立；可达性维持"潜伏"（引擎 `DECLARE_DYNAMIC_MULTICAST_DELEGATE_RetVal_*` 的存在性未复扫）
- **可达性**: 潜伏
- **复核证据**: `Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:456-462`（多播：`DelegateParams.Emplace(*ParamIterator)` 无任何 `CPF_ReturnParm` 过滤）对比 `:84-97`（单播：`:87-90` 显式分流 `CPF_ReturnParm → DelegateReturnParam`，`:92-94` 其余入 `DelegateParams`）—— 行号全部吻合；多播的 `DelegateDeclarationBody` 亦无返回类型槽位（`FDelegateGenerator.cpp:631-634` 硬编码 `public delegate void Delegate`）
- **级别变动**: 无
- **文件**: `Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:456-462`（多播）对比 `:84-97`（单播）
- **函数**: `FDelegateGenerator::Generator(FMulticastDelegateProperty*)`
- **置信度**: 中（不对称确凿；UE 的 UHT 通常会拒绝非 void 的多播签名，故实际影响小）

**现状（代码事实）**
```cpp
// 单播 :87-94：显式区分返回参数
if (ParamIterator->HasAnyPropertyFlags(CPF_ReturnParm)) { DelegateReturnParam = *ParamIterator; }
else { DelegateParams.Emplace(*ParamIterator); }
// 多播 :456-462：所有 Parm 一律当参数
for (TFieldIterator<FProperty> ParamIterator(SignatureFunction); ParamIterator && (ParamIterator->PropertyFlags & CPF_Parm); ++ParamIterator)
{
	DelegateParams.Emplace(*ParamIterator);
	UsingNameSpaces.Append(FGeneratorCore::GetPropertyTypeNameSpace(*ParamIterator));
}
```
**问题**：若出现带返回值签名的多播委托（UHT 一般不允许，故列为 P3），生成的 `delegate void Delegate(...)` 会把返回值变成一个普通参数，`Broadcast` 也随之要求多传一个值——语义与 UE（多播忽略返回值）不同，而且是"能编译但行为怪"的静默形态。

**建议**：在 `:456-462` 循环内补 `if (ParamIterator->HasAnyPropertyFlags(CPF_ReturnParm)) { continue; }`（或在过滤时打日志），使两条路径行为一致。

**验证方式**：grep 生成物中 `public delegate void Delegate` 的参数个数与 UE 头文件比对；构造一个非 void 多播签名尝试生成（若 UHT 报错则说明该分支不可达，可只加注释说明）。

---

### [F-GEN2-011] `GetFunctionIndex` 的第 5 号组合在运行时不存在（当前不可达，但耦合很脆）

- **类别**: 可优化/可读性（耦合）
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 正反两侧都核对过，且**运行时消费点已定位**：C++ 与 C# 两侧恰好各 10 个入口，集合 = `{0,1,2,3,4,6,7} × {Generic,Primitive,Compound}`，**无 5**
- **可达性**: 活跃（`GetFunctionIndex` 的调用与编码路径确实执行；"编号 5 的组合不可达"正是本条内容）
- **复核证据**: 生成侧 `Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:215-219`、`:556-560`（传入 `!DelegateParams.IsEmpty()` 与 `!DelegateRefParamIndex.IsEmpty()`，且后者 ⊆ 前者 → index 5 不可达）；`Source/ScriptCodeGenerator/Private/FGeneratorCore.cpp:681-689` 位拼接。**运行时消费点**：C++ 注册表 `Source/UnrealCSharp/Private/Domain/Interop/FRegisterDelegate.cpp:184-193`（10 个 `.Function("GenericExecute0") … ("CompoundExecute7")`，其中 `:190` 跳到 `Execute4`、`:191` 跳到 `Execute6`）+ `:80`；C# 声明 `Script/UE/Library/FDelegateImplementation.cs:53-125`（同样 10 个，`Execute4` 与 `Execute6` 之间无 `Execute5`）。全插件 `grep 'Execute5|PrimitiveExecute5|CompoundExecute5|Broadcast5'` = **0 命中**
- **级别变动**: 无
- **文件**: `Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:215-219`、`:556-560`；`Source/ScriptCodeGenerator/Private/FGeneratorCore.cpp:681-689`
- **函数**: `FGeneratorCore::GetFunctionIndex`、`FDelegateGenerator::Generator(...)`
- **置信度**: 高（正反两侧都核对过）

**现状（代码事实）**：索引 = `bHasReturn | bHasInput<<1 | bHasOutput<<2`。生成侧传入的是 `!DelegateParams.IsEmpty()` 与 `!DelegateRefParamIndex.IsEmpty()`，而 `DelegateRefParamIndex ⊆ DelegateParams`，故 **index 5/7 之外的"有 out 无 in"（index 5）不可达**；运行时注册的组合（`FRegisterDelegate.cpp:80-193`、`Script/UE/Library/FDelegateImplementation.cs:53-125`）恰为 `{0,1,2,3,4,6,7} × {Generic,Primitive,Compound}`，与可达集合完全一致。

**问题**：不是缺陷，但**没有任何编译期/断言约束**保证"可达集合 = 已实现集合"。本次核查同时发现：C++/C# 两侧都**没有** `Execute5`/`CompoundExecute5`（`FRegisterDelegate.cpp:184-193`）。一旦将来把 `bHasInput` 改为"仅统计非 out 参数"（这是很自然的"优化"），`index 5` 立刻可达，生成的 C# 会调用一个不存在的方法 → 整份生成工程编译失败（CS0117），且报错点在**自动生成的文件**里，排查成本高。

**建议**：在 `GetFunctionIndex` 附近加注释说明"bit1 是 bit2 的超集，5 不可达"；或在 `FRegisterDelegate` 里补齐 5（`PrimitiveExecute5`/`CompoundExecute5`）以消除隐患；再加一个 `check(...)` 断言。

**验证方式**：`Select-String -Path Script/UE/Proxy/**/*.cs -Pattern 'Execute5Implementation'`（本次 16055+75 文件命中 0）；`grep -rn "Execute5" Source/`（本次 0 命中）。

---

### [F-GEN2-012] 生成器从不输出 `[Flags]`，UE 的 `Bitflags` 元数据被忽略

- **类别**: 可优化/可读性（语义映射缺口）
- **严重度**: **P3**
- **复核结论**: ⚠️部分确认（偏差：**证据句错** —— "插件源码里也 grep 不到 `Bitflags`/`Flags]` 的处理"**不成立**；`[Flags]` 缺失这一核心结论依然成立，且只对**静态**生成器成立）
- **可达性**: 活跃
- **复核证据**: 两个输出模板确无 `[Flags]`：`Source/ScriptCodeGenerator/Private/FEnumGenerator.cpp:95-114`、`Source/ScriptCodeGenerator/Private/FBindingEnumGenerator.cpp:41-56`；产物扫描 `grep '\[Flags\]' Script/UE/Proxy` = **0 命中**。但插件**确有** Bitflags 管道（**仅动态/反射路径**）：`Source/UnrealCSharpCore/Public/CoreMacro/MetaDataAttributeMacro.h:311`（`CLASS_BITFLAGS_ATTRIBUTE`）、`Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp:856`、`:2394`、`Source/UnrealCSharpCore/Private/Dynamic/FDynamicGeneratorCore.cpp:1088`（`GetEnumMetaDataAttributes()` 把 Bitflags 列为枚举元数据）、`Script/UE/Dynamic/Enum/BitflagsAttribute.cs:6`（`class BitflagsAttribute : Attribute`）。产物引文也已修正：`Script/UE/Proxy/Binding/EObjectFlags.cs:8` 实为 `public enum EObjectFlags : int`（原文写 `: uint` 有误）
- **级别变动**: 无
- **文件**: `Source/ScriptCodeGenerator/Private/FEnumGenerator.cpp:95-114`（原生枚举输出模板）、`Source/ScriptCodeGenerator/Private/FBindingEnumGenerator.cpp:41-56`（binding 枚举模板）
- **函数**: `FEnumGenerator::Generator(const UEnum*)`、`FBindingEnumGenerator::Generator(const FBindingEnum*)`
- **置信度**: 高（实测）

**现状（代码事实）**：两个生成器输出的枚举体都是固定模板，属性只有 `[PathName("...")]`（FEnumGenerator）或无属性（FBindingEnumGenerator）：

```csharp
// Script/UE/Proxy/Binding/EObjectFlags.cs（由 FBindingEnumGenerator 生成，位标志枚举）
public enum EObjectFlags : int { ... }        // ← 实测产物 :8 = `public enum EObjectFlags : int`（原文写 `: uint`，已修正）
// Script/UE/Proxy/Engine/Classes/Engine/ECollisionChannel.cs（FEnumGenerator）
[PathName("/Script/Engine.ECollisionChannel")]
public enum ECollisionChannel : byte { ... }
```
全量扫描：`Script/UE/Proxy` 下 **`[Flags]` 命中 0**（复跑 `grep '\[Flags\]'`）；两个**静态**生成器模板（`FEnumGenerator.cpp:95-114`、`FBindingEnumGenerator.cpp:41-56`）均无 `[Flags]` 槽位。**但**"插件源码里也 grep 不到 `Bitflags`"**不成立**：`Source/UnrealCSharpCore/Public/CoreMacro/MetaDataAttributeMacro.h:311`、`FReflectionRegistry.cpp:856/2394`、`FDynamicGeneratorCore.cpp:1088`、`Script/UE/Dynamic/Enum/BitflagsAttribute.cs:6` 构成**动态/反射路径**的 Bitflags 管道 —— 静态生成器确实未使用它，但插件并非"完全没有处理"。

**问题**：UE 侧 `meta=(Bitflags)` 的 UENUM（如 `EObjectFlags`、`EPropertyFlags` 类）在 C# 里仍可做 `|`/`&` 运算（普通枚举支持），损失的只是 `Enum.ToString()` 的"组合名"输出与 `HasFlag` 的语义表达；若用户代码依赖 `ToString()` 打印标志组合，会看到数字而非名字。属于"功能缺失但不高危"。

**建议**：在 `FEnumGenerator::Generator` 中读取 `InEnum->HasMetaData(TEXT("Bitflags"))`（以及 `UEnum::GetBoolMetaDataHierarchical`），命中则输出 `[Flags]` + `using System;`；`FBindingEnumGenerator` 可在 `FBindingEnum` 上加一个 `bIsFlags` 字段由 binding 宏显式声明。

**验证方式**：`Select-String -Path Script/**/Proxy/**/*.cs -Pattern '\[Flags\]'`（当前 0）；对 `EObjectFlags` 这类枚举在 C# 中打印 `(EObjectFlags.RootSet | EObjectFlags.Transactional).ToString()` 观察输出。

---

### [F-GEN2-013] `FEnumGenerator::GeneratorCollisionChannel()` 生成的枚举文件缺少统一的"生成头注释"

- **类别**: 可读性/一致性（注释与约定不符）
- **严重度**: **P3**
- **复核结论**: ⚠️部分确认（偏差：**`GeneratorCollisionChannel` 不是唯一例外** —— **多播委托模板同样缺生成头注释**，本条范围应扩大为"2 个生成点"，见"复核证据"）
- **可达性**: 活跃
- **复核证据**: `Source/ScriptCodeGenerator/Private/FEnumGenerator.cpp:224-241`（格式串 `:224-234` 比正常路径 `:95-114` 少一个 header `%s` 槽位，实参从 `*UsingNameSpaceContent` 起 → 与原文引文吻合）；产物 `Script/UE/Proxy/Engine/Classes/Engine/ECollisionChannel.cs:1-6`：第 1 行即 `using Script.CoreUObject;`，**无头注释**。**新增同类实例**：`Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:678-697` 的多播 `Content` 只有 14 个 `%s`（`*UsingNameSpaceContent` 打头），**没有** `GetGeneratorHeaderComment()` 槽位（单播 `:355-372` 有）；产物 `Script/Game/Proxy/UnrealCSharpTest/FOnPickUp.cs:1-4`、`FOnUseItem.cs:1-4` 均以 `using` 开头，而单播产物 `Script/UE/Proxy/Engine/FViewportDisplayCallback.cs:1-4` 有头注释 → 两类产物的差异与源码完全对应
- **级别变动**: 无
- **文件**: `Source/ScriptCodeGenerator/Private/FEnumGenerator.cpp:224-241`（对比同文件 `:95-114` 的正常路径）；另 `Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:678-697`（多播路径）
- **函数**: `FEnumGenerator::GeneratorCollisionChannel()`
- **置信度**: 高

**现状（代码事实）**
```cpp
// :95-107 正常枚举：第一个 %s 是 header comment
const auto Content = FString::Printf(TEXT("%s\n" "%s\n" "namespace %s\n" ...),
                                     *FGeneratorCore::GetGeneratorHeaderComment(), *UsingNameSpaceContent, ...);
// :224-234 CollisionChannel：格式串里没有 header comment 的槽位
const auto Content = FString::Printf(TEXT("%s\n" "namespace %s\n" ...),
                                     *UsingNameSpaceContent, *NameSpaceContent, ...);   // 少一个 %s
```
产物侧证据（无 BOM 注释头）：
```csharp
// Script/UE/Proxy/Engine/Classes/Engine/ECollisionChannel.cs:1-4
using Script.CoreUObject;
namespace Script.Engine
{
```
而**多数**其它生成文件都以 `/*====...Generated code exported from UnrealCSharp. DO NOT modify this manually!...*/` 开头（如 `EObjectFlags.cs:1-4`、单播委托 `Script/UE/Proxy/Engine/FViewportDisplayCallback.cs:1-4`）。⚠️ **但不是全部**：多播委托模板 `Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:678-697` 的 `Content` 同样没有 header 槽位（14 个 `%s`，以 `*UsingNameSpaceContent` 打头），产物 `Script/Game/Proxy/UnrealCSharpTest/FOnPickUp.cs:1-4`、`FOnUseItem.cs:1-4` 均以 `using` 开头 —— 故本条的适用对象是**2 个生成点**，不是 1 个。

**问题**：这份文件同样会被 `FGeneratorCore::AddGeneratorFile` 登记、被 `DeleteRemainGeneratorFiles` 当作受管文件（可被自动删除），却缺了"不要手改"的警示；也与其余 16000+ 文件风格不一致，用户可能在不知情时手工修改后被覆盖。

**建议**：给 `:224-234` 的格式串补上 `%s` 槽位与 `*FGeneratorCore::GetGeneratorHeaderComment()` 实参（一行改动）。

**验证方式**：重新生成后检查 `ECollisionChannel.cs` 首行；或 `Select-String -Path Script/UE/Proxy/**/ECollisionChannel.cs -Pattern 'DO NOT modify'`（当前无命中）。

---

### [F-GEN2-014] `FStructGenerator` 的 `GetHashCode()` 用句柄（身份）而 `Equals/==` 用值比较，违反 C# 哈希契约

- **类别**: Bug（容器行为错误）
- **严重度**: **P1**（容器行为错误；与 `06-…/02b` F-CS2B-004 同级）
- **复核结论**: ✅确认 —— 生成代码事实、契约违反、后果成立；且**影响面已量化**
- **可达性**: 活跃
- **复核证据**: `Source/ScriptCodeGenerator/Private/FStructGenerator.cpp:150-175` 内嵌 C# 文本：`:163` `ReferenceEquals(A, B) || UStructImplementation.UStruct_IdenticalImplementation(...)`（**值比较**）、`:167` `Equals(object) => this == Other as %s`、`:168` `GetHashCode() => (int)HandleData.GetHandle(this)`（**句柄/身份**）—— 逐行吻合。产物逐字复现：`Script/UE/Proxy/ActorSequence/ActorSequenceObjectReference.cs:38`（值比较）、`:43`（Equals）、`:45`（句柄哈希）。**影响面实测**：`grep 'GetHashCode\(\) => \(int\)HandleData\.GetHandle\(this\);' Script/UE/Proxy` = **5925 处**（命中上限 250，总数 5925），即该契约违规被写进了全部生成结构体包装，而非仅少数手写类型
- **级别变动**: **P2→P1**（与 `06-…/02b` F-CS2B-004 —— 同一 `Equals`/`GetHashCode` 契约违规、已判 **P1/确认/活跃** —— 同族；本处由生成器批量写入 5925 个产物文件，覆盖面显著更大，故对齐为 P1）
- **文件**: `Source/ScriptCodeGenerator/Private/FStructGenerator.cpp:150-175`（生成 `IdenticalContent`）
- **函数**: `FStructGenerator::Generator(const UScriptStruct*)`
- **置信度**: 中（生成代码确凿；"是否真的出现不同句柄代表相等值"需要运行期确认内存模型，见"未覆盖/存疑项"）

**现状（代码事实）**

```cpp
// FStructGenerator.cpp:150-175（生成器内嵌的 C# 文本）
"public static bool operator ==(%s A, %s B)\n" ... 
"return ReferenceEquals(A, B) || UStructImplementation.UStruct_IdenticalImplementation(HandleData.GetHandle(StaticStruct()), HandleData.GetHandle(A), HandleData.GetHandle(B));\n"   // :163 值比较
"public override bool Equals(object Other) => this == Other as %s;\n"        // :167
"public override int GetHashCode() => (int)HandleData.GetHandle(this);\n"    // :168 ← 身份（句柄）哈希
```
真实产物（逐字一致，`Script/UE/Proxy/ActorSequence/ActorSequenceObjectReference.cs:26-45`、`Script/Game/Proxy/UnrealCSharpTest/UnitTest/Core/TestStruct.cs:26-45`）。

**问题**：`==`/`Equals` 依据 `UStruct_IdenticalImplementation`（**按值**相等），而 `GetHashCode` 依据 C# 包装对象的**句柄**（近似身份）。C# 要求"`a.Equals(b)` 为真 ⇒ `a.GetHashCode() == b.GetHashCode()`"。若两个 C# 包装对象分别指向两块内容相同但地址不同的原生存储（例如同一个 `FVector` 值从两个不同的属性 getter 取回——每个 getter 都把原生地址塞进返回缓冲，见 `ActorSequenceObjectReference.cs:84` 的 `HandleData.GetObject(*(nint*)ReturnBuffer)`），则：
```csharp
var d = new Dictionary<FMyStruct, int>();
d[structFromPropertyA] = 1;
d[structFromPropertyB] = 1;   // 与 A 值相等但句柄不同 → 另起一个 entry，而非覆盖
```
在 `Dictionary`/`HashSet`/`TMap` 包装层中表现为"相等但取不到"、"键重复"。另外 `(int)` 把 64 位句柄截断到 32 位是允许的（哈希冲突合法），但截断会加剧冲突。

**建议**：要么把 `GetHashCode` 改成基于值（例如对 `UStruct` 做逐属性哈希，或退化为 `0`/常量——常量虽慢但**满足契约**），要么在文档里明确"生成的结构体类型只做引用语义、不可作为字典键"，并给 `Equals` 加 `[Obsolete]` 式注释。最省事且正确的折中：
```csharp
public override int GetHashCode() => 0;   // 保持契约；结构体作为键本就应走原生 TMap
```
（更彻底的做法：为每个结构体生成逐字段哈希，但这需要跨语言字段布局知识，成本高。）

**验证方式**：C# 用例——取同一属性的结构体值两次（或取同值的两个不同实例），`Assert(a == b)` 通过后 `Assert(a.GetHashCode() == b.GetHashCode())` 是否通过；再把它们作为 `Dictionary` 键插入并检查 entry 数。

---

### [F-GEN2-015] `FEnumGenerator` 中的死局部变量、末尾逗号与静态 `TMap::operator[]` 读

- **类别**: 可优化/可读性
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 两处死变量、尾逗号写法差异、静态 `TMap::operator[]` 读，均已逐行核实
- **可达性**: 活跃
- **复核证据**: `Source/ScriptCodeGenerator/Private/FEnumGenerator.cpp:57` 与 `:192` 的 `auto ClassName = InEnum->GetName();`：读完整个 `Generator(const UEnum*)`（`:23-121`）与 `GeneratorCollisionChannel()`（`:173-248`）**均无第二次读取** → 确认 write-only；`:79-84` 尾逗号（`Index == InEnum->NumEnums() - 1 ? TEXT("") : TEXT(",")`）；`:267` `EnumUnderlyingTypeName[*FoundEnumUnderlyingType]` 读静态 `TMap`（键集 `:252-263` 完整 9 项，故当前不会插入）；`Source/ScriptCodeGenerator/Private/FStructGenerator.cpp:215` `FString PropertyAccessSpecifiers = TEXT("public");`；对照 `Source/ScriptCodeGenerator/Private/FBindingEnumGenerator.cpp:38`（最后一项不加逗号）。产物 `Script/Game/Proxy/UnrealCSharpTest/UnitTest/Core/BP_TestEnum.cs:15` = `BlueprintTestEnumTwo = 2,`
- **级别变动**: 无
- **文件**: `Source/ScriptCodeGenerator/Private/FEnumGenerator.cpp:57`、`:192`（死变量）；`:79-84`（逗号）；`:267`（`operator[]`）；`Source/ScriptCodeGenerator/Private/FStructGenerator.cpp:215`（常量局部）
- **函数**: `FEnumGenerator::Generator(const UEnum*)`、`FEnumGenerator::GeneratorCollisionChannel()`、`FEnumGenerator::GetEnumUnderlyingTypeName(const UEnum*)`、`FStructGenerator::Generator(const UScriptStruct*)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FEnumGenerator.cpp:57 与 :192 —— 两处都只赋值、从不读取（ClassName 仅出现在这两行）
auto ClassName = InEnum->GetName();
// :79-84 枚举体逐项拼装：最后一项是被 break 掉的隐藏 _MAX，因此正常项会带上尾逗号
EnumeratorContent += FString::Printf(TEXT("\t\t%s = %lld%s\n"), ...,
                     EnumeratorValue, Index == InEnum->NumEnums() - 1 ? TEXT("") : TEXT(","));
// :267 用 operator[] 读静态 TMap（键不存在时会插入默认值 → 非 const 修改共享静态）
return EnumUnderlyingTypeName[*FoundEnumUnderlyingType];
// FStructGenerator.cpp:215 只被用一次的字符串常量局部
FString PropertyAccessSpecifiers = TEXT("public");
```
产物侧证据（尾逗号）：`Script/Game/Proxy/UnrealCSharpTest/UnitTest/Core/BP_TestEnum.cs:15` `BlueprintTestEnumTwo = 2,`（合法的 C#，但与 `FBindingEnumGenerator.cpp:38` 那种"最后一项不加逗号"的写法不一致）。

**问题**：均为可读性/一致性，无功能影响。`operator[]` 读静态 `TMap` 虽然当前键集完整（9 个 `EEnumUnderlyingType` 全部有映射），但这是"读操作可能改容器"的写法，若将来少了映射会静默插入并返回空字符串 → 生成 `public enum X : ` **非法语法**。

**建议**：删除 57/192 的死变量；尾逗号判断改为"已输出项计数是否为 0"或直接统一保留尾逗号（两处风格取一）；`:267` 改用 `Find` + 空值判断；`PropertyAccessSpecifiers` 直接内联或改为 `static const TCHAR*`。

**验证方式**：编译器警告（死变量在 MSVC 下可能不报，可跑静态分析 `-Wall`/PVS）；对 `EnumUnderlyingTypeName` 做"键缺失"单元测试确认返回空串会让生成器产出非法枚举。

---

### [F-GEN2-016] 五个生成器之间大量复制粘贴（头部/using/内容组装/落盘），且 `GameplayTag` 硬编码 `using Script.GameplayTags;`

- **类别**: 可优化/可读性（重复代码 + 隐式依赖）
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 5 份重复实现的行号全部吻合；硬编码 `using Script.GameplayTags;` 的目标命名空间**确实存在**，故当前不报错，但"由 `FStructGenerator` 产出 + 受 `IsSkip/IsSupported` 过滤"的隐患成立
- **可达性**: 活跃
- **复核证据**: 5 处收尾代码：`Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:343-391`（多播 `:666-676`）、`FStructGenerator.cpp:297-350`、`FEnumGenerator.cpp:87-120`、`FBindingEnumGenerator.cpp:41-66`、`FGameplayTagGenerator.cpp:64-94`。硬编码 using：`FGameplayTagGenerator.cpp:64-69` = `FString::Printf(TEXT("using %s.%s;\n"), *NAMESPACE_ROOT, *GAMEPLAY_TAGS_NAME)`；产物 `Script/Game/Proxy/UnrealCSharpTest/GameplayTags.cs:6` = `using Script.GameplayTags;`，而该命名空间确由 `FStructGenerator` 产出 —— `Script/UE/Proxy/GameplayTags/Classes/GameplayTag.cs:10` = `namespace Script.GameplayTags`（对应 `UScriptStruct FGameplayTag`，`GetFileName` 用 `InField->GetName()` 去掉 `F` 前缀，见 `Source/ScriptCodeGenerator/Public/FGeneratorCore.inl:39`）。过滤点：`FStructGenerator.cpp:46-54`（`FGeneratorCore::IsSkip` / `IsSupported`，签名见 `Source/ScriptCodeGenerator/Public/FGeneratorCore.h:51/61`）
- **级别变动**: 无
- **文件**: `FDelegateGenerator.cpp:343-391`、`FStructGenerator.cpp:297-350`、`FEnumGenerator.cpp:87-120`、`FBindingEnumGenerator.cpp:41-66`、`FGameplayTagGenerator.cpp:64-94`
- **函数**: 各 `Generator(...)`
- **置信度**: 高（重复代码）/ 中（硬编码 using 的触发条件）

**现状（代码事实）**
五份代码重复实现同一套"收尾"动作：
```cpp
UsingNameSpaces.Remove(NameSpaceContent);
UsingNameSpaces.Remove(TEXT(""));
for (auto UsingNameSpace : UsingNameSpaces) { UsingNameSpaceContent += FString::Printf(TEXT("using %s;\n"), *UsingNameSpace); }
auto Content = FString::Printf(TEXT("%s\n%s\nnamespace %s\n{\n..."), *FGeneratorCore::GetGeneratorHeaderComment(), *UsingNameSpaceContent, ...);
const auto FileName = ...; FGeneratorCore::AddGeneratorFile(FileName); FUnrealCSharpFunctionLibrary::SaveStringToFile(FileName, Content);
```
`GameplayTagGenerator` 另有一处硬编码：
```cpp
// FGameplayTagGenerator.cpp:64-69
const auto UsingNameSpaceContent = FString::Printf(TEXT("using %s.%s;\n"), *NAMESPACE_ROOT, *GAMEPLAY_TAGS_NAME);   // using Script.GameplayTags;
```

**问题**
1. 重复代码导致修一处漏三处（本报告 F-GEN2-004 的排序、F-GEN2-007 的返回值检查都要在 5 处重复修）。
2. 硬编码 `using Script.GameplayTags;` 隐含"`Script.GameplayTags.FGameplayTag` 一定存在"的假设，而该命名空间由 `FStructGenerator` 生成（`UnrealCSharpEditor.cpp:353`），其生成与否受 `FGeneratorCore::IsSkip`/`IsSupported` 过滤（`FStructGenerator.cpp:46-54`，例如勾选"跳过引擎模块生成"）。此时生成的 `GameplayTags.cs` 会因 `CS0246: 找不到命名空间 Script.GameplayTags` 直接编译失败，而报错指向自动生成文件。

**建议**
把收尾逻辑提取到 `FGeneratorCore`（该类已有 `AddGeneratorFile`/`GetGeneratorHeaderComment` 的合作基础）：
```cpp
static void EmitFile(const FString& InFileName, FString& InUsingNameSpaceContent,
                     const FString& InNameSpace, const FString& InBody,
                     const TSet<FString>& InUsingNameSpaces, const FString& InSelfNamespace);
```
`GameplayTagGenerator` 的 using 改为条件输出（先查 `FStructGenerator`/`FGeneratorCore::IsSupported(FGameplayTag::StaticStruct())` 或 `IsSkip` 的结果），或在 `Script/Game/.../GameplayTags.cs` 内改为全限定名 `Script.GameplayTags.FGameplayTag`。

**验证方式**：`Select-String -Path Source/ScriptCodeGenerator/Private/*.cpp -Pattern 'AddGeneratorFile'` 应能收敛到 1 处；打开"跳过引擎模块生成"选项后跑一次生成并编译，复现 CS0246。

---

## 4. 死代码清单

判定口径（与规范一致）：**用去掉类名限定的成员名 grep 全部 `Source/`（7 个模块）**；对 `private static` 成员，因为调用点写作非限定名（`GeneratorCollisionChannel();`），**限定名命中数 == 1 不构成死代码证据**，必须打开文件确认调用点。下表为实测结果（`Select-String ... | Measure-Object`）。

| 符号 | 声明位置 | 限定名 grep 命中 | 非限定名调用点（已核对） | 判定 |
|---|---|---|---|---|
| `FDelegateGenerator::Generator` | `FDelegateGenerator.h:8`(public)、`:13`、`:15`(private) | 10 | 外部：`FClassGenerator.cpp:152/436`、`FStructGenerator.cpp:204`、`FGeneratorCore.cpp:159/232/234/246`；内部：`FDelegateGenerator.cpp:18/22` | 活 |
| `FDelegateGenerator::Delegate` | `FDelegateGenerator.h:17`(private static) | 2 | `FGeneratorCore.cpp:1038`（`friend class FGeneratorCore` 必要） | 活（friend 必要） |
| `FStructGenerator::Generator` | `FStructGenerator.h:8`(public)、`:10` | 4 | `UnrealCSharpEditor.cpp:353`、`FAssetGenerator.cpp:94`、内部 `FStructGenerator.cpp:24` | 活 |
| `FEnumGenerator::Generator` | `FEnumGenerator.h:21`、`:23` | 6 | `UnrealCSharpEditor.cpp:357`、`FAssetGenerator.cpp:49/112`、内部 `FEnumGenerator.cpp:16` | 活 |
| `FEnumGenerator::AddEnumUnderlyingType` | `FEnumGenerator.h:25` | 3 | `FGeneratorCore.cpp:89/170` | 活 |
| `FEnumGenerator::EnumUnderlyingType` | `FEnumGenerator.h:37`(private static) | 2 | `FGeneratorCore.cpp:1040`（friend 必要） | 活 |
| `FEnumGenerator::GeneratorCollisionChannel` | `FEnumGenerator.h:28`(private) | 1 | `FEnumGenerator.cpp:20` | 活（限定名 1 是预期的） |
| `FEnumGenerator::GetEnumUnderlyingTypeName` | `FEnumGenerator.h:30`(private) | 1 | `FEnumGenerator.cpp:61/239` | 活 |
| `FEnumGenerator::IsValueInUnderlyingTypeRange` | `FEnumGenerator.h:32`(private) | 1 | `FEnumGenerator.cpp:70` | 活 |
| `FBindingEnumGenerator::Generator` | `FBindingEnumGenerator.h:8`(public)、`:11`(private) | 3 | `UnrealCSharpEditor.cpp:378`、内部 `FBindingEnumGenerator.cpp:11` | 活 |
| `FGameplayTagGenerator::Generator` | `FGameplayTagGenerator.h:10`(public, 导出) | 6 | `UnrealCSharpEditor.cpp:262/365` | 活（两个入口） |
| `FGameplayTagGenerator::GeneratorTag` | `.h:15`(private) | 1 | `FGameplayTagGenerator.cpp:34` | 活 |
| `FGameplayTagGenerator::GeneratorDocComment` | `.h:17`(private) | 1 | `FGameplayTagGenerator.cpp:181/204` | 活 |
| `FGameplayTagGenerator::GeneratorChildren` | `.h:19`(private) | 1 | `FGameplayTagGenerator.cpp:73/216` | 活 |
| `FGameplayTagGenerator::MakeUniqueName` | `.h:21`(private) | 1 | `FGameplayTagGenerator.cpp:175` | 活 |
| `FGameplayTagGenerator::CollectGameplayTags` | `.h:23`(private) | 1 | `FGameplayTagGenerator.cpp:30` | 活 |
| `FGameplayTagGenerator::VisitTagNodes` | `.h:25`(private) | 1 | `FGameplayTagGenerator.cpp:258/279` | 活（递归） |
| `FGameplayTagGenerator::FGameplayTagTreeNode` | `.h:13`(private 嵌套结构) | **7**（复跑：h:13；cpp:15/21/26/97/115/148） | 仅本文件 | 活（原文记 9，已修正） |
| `FEnumGenerator.cpp:57` `auto ClassName` | 局部变量 | — | **无任何读取** | **死代码（write-only 局部变量）** |
| `FEnumGenerator.cpp:192` `auto ClassName` | 局部变量 | — | **无任何读取** | **死代码（同上）** |

**结论**：这 5 个生成器的 public/private 成员**没有找到真正的死代码**（无"废弃的生成分支"）。两处 `write-only` 局部变量是唯一的死代码实例（见 F-GEN2-015）。另外核对了 `friend class FGeneratorCore`：`FDelegateGenerator`（`FDelegateGenerator.h:11`）与 `FEnumGenerator`（`FEnumGenerator.h:35`）**都是必要的**（`FGeneratorCore.cpp:1038` 访问 `FDelegateGenerator::Delegate`、`:1040` 访问 `FEnumGenerator::EnumUnderlyingType`），而 `FStructGenerator`/`FBindingEnumGenerator`/`FGameplayTagGenerator` 没有多余 friend。

---

## 5. 逐函数清单（无问题者一句话结论）

### FDelegateGenerator.cpp
| 函数 | 做了什么 | 调用方 | 结论 |
|---|---|---|---|
| `Generator(FProperty*)` `:9` | null 检查后按类型分派到单播/多播 | `FClassGenerator.cpp:152/436`、`FStructGenerator.cpp:204`、`FGeneratorCore.cpp:159/232/234/246` | OK |
| `Generator(FDelegateProperty*)` `:26` | 生成单播委托 C# 类（delegate/Execute/Bind/Unbind/Clear/IsBound）+ TSet 去重 | 上行分派 | 有 F-GEN2-001（out 参数回写）、F-GEN2-004、F-GEN2-006 |
| `Generator(FMulticastDelegateProperty*)` `:394` | 生成多播委托 C# 类（delegate/Broadcast/IsBound/Contains/Add/AddUnique/Remove/RemoveAll/Clear） | 上行分派 | 回写括号正确；F-GEN2-004/006/010 |

**专项核查（结论：一致）**
- 参数/返回值/`const` 引用映射：`CPF_ReturnParm → DelegateReturnParam`（`:87-90`）、非 const 的 `CPF_OutParm → ref`（`:110-116`）；`const` 引用因 `!CPF_ConstParm` 条件被正确排除（仍走值传递）——与 UE 一致。
- `TArray`/对象指针参数：走 `GetBufferCast`（非原生 → `nint`）+ `HandleData.GetHandle(...)`，产物 `FAIMoveCompletedSignature.cs:23` 为 `*(byte*)(InBuffer + 8) = (byte)Result;`，与 C++ 侧 1 字节属性宽度一致。
- 无参委托：`GenericBroadcast0Implementation`（`FOnPickUp` 类形态见 `FAIMoveCompletedSignature.cs` 之外的 0 参用例）与运行时 `FMulticastDelegate_GenericBroadcast0Implementation`（`FRegisterMulticastDelegate.cpp:148`、`FMulticastDelegateImplementation.cs:80`）一一对应；单播 `GenericExecute0`（`FRegisterDelegate.cpp:80`）一致。
- 返回 `void`/对象指针：`GetFunctionPrefix`（`FGeneratorCore.cpp:672-679`）+ `GetFunctionIndex` 组合，实测产物 `FViewportDisplayCallback.cs:34`（Primitive7）、`FOnGetItemChildrenDynamic.cs:32`（Generic6）与运行时注册集合完全吻合。
- `Add`/`AddUnique`/`Remove`/`RemoveAll`/`Clear` 语义：`UMulticastDelegateHandler` 用 `TArray<FDelegateWrapper> DelegateWrappers`（`MulticastDelegateHandler.h:121`），`Add` 追加（允许重复）、`AddUnique` 去重（`.cpp:75-107`）→ **与 UE 的 `AddDynamic`（允许重复）/`AddUniqueDynamic`（去重）语义一致**；`RemoveAll(UObject*)` 对应 UE 的 `RemoveAll`，`Remove` 对应单条移除。**未发现语义偏差**（此前担心的"Add/AddUnique 等价"不成立）。
- 委托名称冲突：见 F-GEN2-006。
- C# 关键字转义：参数名经 `FUnrealCSharpFunctionLibrary::Encode`（`FUnrealCSharpFunctionLibrary.cpp:1260-1293`，关键字表 77 项 → `__name` 前缀），非原生再走 `FNameEncode::Encode`（`NameEncode.cpp:58-166`，非法字符 → `_hXX_`，数字开头 → `_h01_`）。**关键字/非法字符处理完善**。
- XML 文档注释：单播/多播生成器**不生成任何 `///` 注释**，因此不存在"`<`/`>`/`&` 未转义导致非法 XML"的问题（该问题只存在于 `FGameplayTagGenerator::GeneratorDocComment`，而它做了 `&`/`<`/`>` 转义，`FGameplayTagGenerator.cpp:132-137`）。**未发现缺陷**。

### FStructGenerator.cpp
| 函数 | 做了什么 | 调用方 | 结论 |
|---|---|---|---|
| `Generator()` `:18` | 遍历全部 `UScriptStruct`，跳过 `UUserDefinedStruct` | `UnrealCSharpEditor.cpp:353` | OK |
| `Generator(const UScriptStruct*)` `:29` | 生成结构体包装类（`partial class` + `IStaticStruct` + `StaticStruct()` + `==/!=/Equals/GetHashCode` + 属性访问器 + `__Xxx` 属性哈希占位字段） | 上行 + `FAssetGenerator.cpp:94` | 见 F-GEN2-004/007/014/015 |

**专项核查**
- `struct` vs `class`：统一生成 `public partial class`（引用语义 + 原生句柄），非 `struct`——这是插件的一致设计（句柄模型），**不需要** `[StructLayout]`/`MarshalAs`（值跨边界只传 `nint` 句柄，`GetBufferCast` 对非原生返回 `nint`、`GetBufferSize` 返回 `sizeof(void*)`，`FGeneratorCore.cpp:466-478`）。**未发现缺陷**。
- `readonly`/`ref`：属性访问器为 `get/set`（`FStructGenerator.cpp:235-280`），参数按值传递（结构体是引用类型），与运行时一致。
- **递归/自引用**：`Generator(const UScriptStruct*)` **不会**为成员结构体递归调用自身（成员仅以类型名 + 句柄引用，`GetPropertyType(StructProperty)` 只返回类名，`FGeneratorCore.cpp:152-155`）；因此 `FStructA` 内含 `TArray<FStructA>`/`FStructA*` 时**不存在无限递归**，`visited` 集合也非必需。**未发现缺陷**（此前假设不成立）。
- 内嵌/位域/联合体/对齐：生成器不展开内嵌结构体、不处理位域（`FBoolProperty` 走 `bool`，`FProperty::GetElementSize()` 的位域语义由运行时读写负责），属设计边界而非缺陷；`FVector` 等"特殊结构体"通过 `FUnrealCSharpFunctionLibrary::IsSpecialStruct`（`FUnrealCSharpFunctionLibrary.cpp:1425-1438`）与 Binding 机制覆盖。
- 构造函数/默认值/`Equals`/`GetHashCode`/`ToString`：`GetHashCode` 见 F-GEN2-014；**未生成 `ToString()`**（结构体在 C# 里 `ToString()` 只给出类型名，调试体验差，但非缺陷）；派生结构体不生成析构（`FStructGenerator.cpp:120-137` 仅在无父结构体时生成），依赖基类 finalizer——设计可行（**未发现缺陷**）。
- 重复属性名/BP 友好名：`PropertyNameSet`（`:181/189/289`）与 `GetVariableFriendlyNameForProperty`（`:229-233`）配合，`__%s` 占位字段名用 UHT 名的 `Encode`（`:219-223`），与 `FCSharpBind.cpp:142-175` 的反射回填键一致。**未发现缺陷**。

### FEnumGenerator.cpp
| 函数 | 做了什么 | 调用方 | 结论 |
|---|---|---|---|
| `Generator()` `:10` | 遍历全部 `UEnum`（跳过 `UUserDefinedEnum`）+ 追加 `GeneratorCollisionChannel()` | `UnrealCSharpEditor.cpp:357` | OK |
| `Generator(const UEnum*)` `:23` | 生成 `[PathName] public enum X : <宽度> { ... }`，丢弃末尾隐藏项 | 上行 + `FAssetGenerator.cpp:49/112` | 见 F-GEN2-005/012/013/015 |
| `AddEnumUnderlyingType` `:123` | 把 `FProperty` 的数值类型映射到 `EEnumUnderlyingType` 并缓存 | `FGeneratorCore.cpp:89/170` | OK（注册点唯一，缓存首值生效：`:130-133`） |
| `GeneratorCollisionChannel` `:173` | 用 `UCollisionProfile` 的通道名重写 `ECollisionChannel` 各成员名 | 内部 `:20` | 见 F-GEN2-013；`UCollisionProfile::Get()`（`:196`）返回值未判空即解引用（`:211`）——**存疑项**，见 §6 |
| `GetEnumUnderlyingTypeName` `:250` | 查缓存 → 宽度名；未命中回退 `byte`(BP)/`long`(原生) | `:61`、`:239` | 见 F-GEN2-005/015 |
| `IsValueInUnderlyingTypeRange` `:273` | 判断枚举值是否落在 C# 宽度范围内（用于"是否保留末尾项"） | `:70` | OK（逻辑正确；`long/ulong` 未知宽度直接 `true`） |

**专项核查**
- `uint8`（UE 默认）→ C# `byte`：实测 `ETestEnum.cs:11`/`ETestEnumClass.cs:11`/`ECollisionChannel.cs:6` 均为 `: byte`，`IsSupported` 走 `FByteProperty`/`FEnumProperty` 两条注册路径都能得到 `UInt8`。**一致**。
- `int64` 枚举 → `long`：`FInt64Property → Int64 → "long"`（`:161-164`、`:261`）。**一致**。
- `enum class : uint8` → `byte`：实测 `ERawTestEnumClass.cs:8`（`enum class ERawTestEnumClass : uint8`，`Source/UnrealCSharpTest/Public/UnitTest/Core/ERawTestEnumClass.h:5`）→ `: byte`，由 `FBindingEnumGenerator` 输出（`std::underlying_type_t` = uint8）。**一致**。
- `::` → `.`：C# 端本来就是 `Enum.Value`，生成器只写枚举成员名，不存在 `::` 转换问题；`PathName` 属性携带 UE 侧路径（`ECollisionChannel.cs:5`）。**未发现缺陷**。
- 重复值（两名同值）：生成器如实输出（C# 合法，`ToString` 取首个）——**未发现缺陷**；但没有生成 `[Flags]`（见 F-GEN2-012）。
- 枚举项关键字转义：`Encode(EnumeratorString, InEnum->IsNative())`（`:82-83`），原生枚举名恒为合法 C++ 标识符，BP 枚举走 `FNameEncode`。**未发现缺陷**。

### FBindingEnumGenerator.cpp
| 函数 | 做了什么 | 调用方 | 结论 |
|---|---|---|---|
| `Generator()` `:7` | 遍历 `FBinding::Get().Register().GetEnums()` | `UnrealCSharpEditor.cpp:378` | 注意：`FBinding::Register()` 只在首次调用时把 `EnumRegisters` 转成 `Enums`（`FBinding.cpp:12-33`），此后新注册的枚举不会出现在 `Enums`——属于 `FBinding` 的设计约束，与本文件无关（记录备查） |
| `Generator(const FBindingEnum*)` `:15` | 生成 `public enum X : <TName<std::underlying_type_t<T>>>`，扁平写入 `Binding/` 目录 | 上行 | 见 F-GEN2-002/003/007/012 |

**与 `FEnumGenerator` 的分工差异**：两者**不是重复实现**，而是互补——
| | `FEnumGenerator` | `FBindingEnumGenerator` |
|---|---|---|
| 数据源 | UE 反射（`UEnum`） | C++ 手工绑定注册表（`BINDING_ENUM` + `TBindingEnumBuilder`） |
| 覆盖范围 | 引擎/工程里被反射的枚举（受 `IsSupported`/`IsSkip` 过滤） | 引擎内部**无 UENUM 反射**或需要自定义成员名的枚举（`EGuidFormats`、`EForceInit`、`EDayOfWeek`、`EMonthOfYear`、`EObjectFlags`、`ELoadFlags`、`ESpawnActorNameMode`，以及工程侧 `ERawTestEnum`/`ERawTestEnumClass`） |
| 底层类型来源 | 运行时属性宽度（`AddEnumUnderlyingType`，顺序敏感） | C++ `std::underlying_type_t<T>`（编译期确定，可靠） |
| 输出路径 | `<Proxy>/<Module>/<RelativePath>/<Name>.cs` | `<Proxy>/Binding/<短名>.cs`（扁平） |
| 头注释 | 有（`GeneratorCollisionChannel` 例外） | 有 |

**可否合并**：可以把"枚举体文本拼装 + 头部/using + 落盘"提取为 `FGeneratorCore` 的公共函数（F-GEN2-016），但**两者不能合并成一个生成器**——数据源、路径规则、底层类型来源都不同。真正值得统一的是：把 `FEnumGenerator` 的"底层类型靠启发式 + 顺序"改为与 binding 侧一样"显式可靠"（见 F-GEN2-005 建议）。

### FGameplayTagGenerator.cpp
| 函数 | 做了什么 | 调用方 | 结论 |
|---|---|---|---|
| `Generator()` `:24` | 收集标签 → 建树 → 空则删文件、否则递归生成 + 落盘 | `UnrealCSharpEditor.cpp:262/365` | 见 F-GEN2-008 |
| `GeneratorTag` `:97` | 按 `.` 分段插入树，末节点记完整 Tag 与注释 | 内部 `:34` | OK（空段跳过 `:109`） |
| `GeneratorDocComment` `:128` | 注释换行折平 + `&`/`<`/`>` 转义后输出 `/// <summary>` | `:181/204` | OK（**本报告中唯一做 XML 转义的生成点**） |
| `GeneratorChildren` `:148` | 递归生成 `class`/字段（叶子）与注释、缩进 | `:73/216` | 见 F-GEN2-009；键已 `Sort()`（`:162`）→ 顺序确定 |
| `MakeUniqueName` `:234` | 兄弟名冲突时追加 `_2`/`_3` | `:175` | OK（只防兄弟冲突，见 F-GEN2-009） |
| `CollectGameplayTags` `:250` | `GetFilteredGameplayRootTags(TEXT(""), ...)` 取根标签 | `:30` | OK |
| `VisitTagNodes` `:262` | 递归收集 `IsExplicitTag() && !IsRestrictedGameplayTag()` 的完整 Tag 与 DevComment | `:258/279` | OK；`#if UE_F_GAMEPLAY_TAG_NODE_GET_DEV_COMMENT` 走 `ACCESS_PRIVATE_MEMBER_PROPERTY(FGameplayTagNode, DevComment, FString)`（`:13`、`:269-274`）——私有成员后门，跨版本兼容手段，**可行但脆**（版本升级需同步） |

**专项核查（回应任务里的三条假设）**
1. **"`Ability.Fire.Ball` → `Ability_Fire_Ball`"**：**不成立**。生成器不做"点号替换成下划线"，而是把每一段变成一个嵌套 `class`/字段（实测产物 `GameplayTags.cs:12-24`：`class Ability { class Dash { class Duration { ... Message } } }`），因此**不存在跨段扁平化**。
2. **"`A.B_C` 与 `A_B.C` 映射成同名 → 静默冲突"**：**不成立**。(a) 两段路径在树里位于不同层级（`A`→`B_C` vs `A_B`→`C`），生成的类型/成员不同；(b) 段内转义使用 `FNameEncode`，它对 `_h` 序列本身再做转义（`NameEncode.cpp:85/101-119`：`A_h2D_B` 里出现的字面 `_h` 会被编码为 `_h5F_...`），映射是**单射**，因此“段名字面碰撞”也不会发生。这条假设可以从报告中排除。
3. **大小写/重命名一致性**：生成器直接使用 `FGameplayTagNode::GetCompleteTagString()`（`:270-273`）原样写入 C# 字符串常量，与运行时注册名完全同源，**不存在大小写或重命名漂移**；但反之，标签重命名会使 C# 成员名消失（编译错误）——这是预期行为，非缺陷。`DevComment` 被折平并转义（`:132-137`），**不会产生非法 XML**。
4. `.Build.cs` 依赖：`ScriptCodeGenerator.Build.cs:40` 把 `GameplayTags` 列入 `PrivateDependencyModuleNames`，且 `FGameplayTagGenerator.cpp:5` 无条件 `#include "GameplayTagsManager.h"`——**没有任何 `WITH_GAMEPLAY_TAGS` 之类的守卫**。GameplayTags 是 UE 常驻运行时模块（`Runtime/GameplayTags`），因此不会因"模块缺失"编译失败；但这也意味着**该模块无法在裁剪了 GameplayTags 的自定义引擎里构建**，属可选加固点（P3 级，不单列发现）。

---

## 6. 未覆盖 / 存疑项

1. **`UCollisionProfile::Get()` 的返回值未判空**：`FEnumGenerator.cpp:196` 取得指针，`:211` 直接 `CollisionProfile->ReturnChannelNameFromContainerIndex(Index)` 解引用，没有 null 检查（两处行号已核实）。我**无法确认**该函数是否可能在特定时序（`GExitPurge`、Commandlet 早期、编辑器关闭）返回 nullptr。**注意**：本机**确实存在** UE 5.6 引擎源码（`<Engine 5.6 安装目录>/Engine/Source`，约 20989 个 `.cpp`），本条卡点**不是**"没有引擎源码"，而是预算内**未定位到 `UCollisionProfile::Get()` 的实现文件**（按 `Runtime/Engine/Private/Engine/CollisionProfile.cpp` 检索失败）→ 仍判 **无法验证（卡点：未定位实现文件）**，故不列为编号发现。若后续确认可能为 null，应升为 P1（生成阶段崩溃）。同文件 `:175` 的 `LoadObject<UEnum>` 是**检查过**的（`:177-180`）。
2. **无 BOM UTF-8 的往返一致性**：`SaveStringToFile` 用 `ForceUTF8WithoutBOM` 写盘（`FUnrealCSharpFunctionLibrary.cpp:1177`），下一次生成用 `FFileHelper::LoadFileToString` 读回比较（`:1156`）。我没有 UE 运行环境来验证 `LoadFileToString` 对"无 BOM 且含非 ASCII"（GameplayTags 的 DevComment 很容易是中文）的文件是否识别为 UTF-8；若它按 ANSI 解码，则内容比较恒不相等 → **每次生成都会重写文件并触发一次 C# 重编译**（`MarkScriptChanged`）。建议在 UE 环境里实测一次（生成两次，观察第二次是否仍改动文件时间戳）。
3. **`FDelegateGenerator` 未防御 `SignatureFunction == nullptr`**：`FGeneratorCore.inl:14-17`、`FUnrealCSharpFunctionLibrary.cpp:526-529/557-560/594-597/625-628` 都显式为 null 做了防御（返回空串/空文件名），但 `FDelegateGenerator.cpp:33` 直接取用 `InDelegateProperty->SignatureFunction` 并传入 `TFieldIterator`（`:84`/`:456`）。若真出现 null，代码会以空 `ClassContent`/空命名空间继续走完，生成 `public class `（无类名）并尝试写入空路径。我**未能确认**反射系统是否可能产出 `SignatureFunction == nullptr` 的 `FDelegateProperty`（生成器作者的防御代码暗示他们见过），因此没有单列为编号发现；建议加一行与 `FGeneratorCore.inl:14-17` 对称的早退。
4. **`UMulticastDelegateHandler` 的 `Add`/`AddUnique` 静默失败**：C++ 侧所有入参校验失败（对象/类型/方法解析不到）都只是嵌套 `if` 落空、无日志（`FRegisterMulticastDelegate.cpp:64-104`、`FRegisterDelegate.cpp:30-49`）。C# 用户传入非 UFUNCTION 的方法（lambda/静态方法）时会**静默无效果**。这属于运行时（非我负责文件），仅在此记录供运行时报告参考。
5. **`FBinding::Register()` 只构建一次 `Enums`/`Classes`**（`FBinding.cpp:12-33`）：晚于首次调用注册的 binding 枚举不会出现在 `FBindingEnumGenerator` 的输入里。未评估静态初始化顺序跨 TU 是否会导致真实丢失（属核心模块范围）。
6. **工程侧存在一份功能重叠的 GameplayTag 生成器（插件之外）**：`<Project>/Script/SourceGenerator/GameplayTagSourceGenerator.cs`（192 行，Roslyn `ISourceGenerator`）同样生成 `public static partial class GameplayTags`（`:75`）并读 `*.ini` 的 `+GameplayTagList`（`:21/46-56`），但**名字映射规则与本插件不同**：它把非法字符一律替换成 `_`（`:156`）、关键字加 `@`（`:165-169`），而插件生成器用 `_hXX_` 转义 + `__` 前缀。两者若同时生效，同一标签会得到**不同的成员名**，甚至在同一 partial 类上产生重复成员（CS0102）。该文件不在插件仓库内，且其开头带有插件的"Generated code exported from UnrealCSharp. DO NOT modify this manually!"头注释——**看起来是一处被废弃/实验性的生成分支**，但我无法确认其来源与是否被 `Game.csproj` 的 `AdditionalFiles` 引用（工程 `Script/*.csproj` 中未见相关配置）。列为存疑项，供 07/08 专项分析跟进。
7. **未做的验证**：本报告未运行 UE 编辑器，因此所有"生成顺序/运行时行为"结论均来自源码静态推导 + **既有产物**反证（产物路径见 §0）；`F-GEN2-001` 的功能影响未通过实际绑定用例复现，仅由生成代码语义断定。

