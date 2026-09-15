# Dynamic 特性（Attribute）体系、Roslyn 源生成器与模板工程

> 本报告 20 条发现的**严重度见各 Finding 的「复核结论」字段**（含 0 条判定为非缺陷、2 条级别调整）。





> 分析范围：`Script/UE/Dynamic/`（255 个特性定义）、`Script/SourceGenerator/`（源生成器 + csproj + AnalyzerReleases）、`Template/`（工程与代码模板）
> 覆盖文件：255 个特性 `.cs` + `UnrealTypeSourceGenerator.cs` + `SourceGenerator.csproj` + `AnalyzerReleases.Unshipped.md` + `Template/` 全部 14 个文件（另交叉阅读 C++ 消费端与 UE 5.6 引擎源码）
>
> 全部文件已读完

---

## 0. 覆盖范围与阅读清单

### 0.1 我负责的文件（全部读完）

| 文件 | 行数 | 是否读完 | 备注 |
|---|---|---|---|
| `Script/UE/Dynamic/Class/*.cs` | 42 文件各 9–15 行 | 是 | 42 个（注：255 是 Dynamic 目录总数，非本目录文件数） |
| `Script/UE/Dynamic/Enum/*.cs` | 3 | 是 | |
| `Script/UE/Dynamic/Function/*.cs` | 62 | 是 | |
| `Script/UE/Dynamic/Generic/*.cs` | 28 | 是 | |
| `Script/UE/Dynamic/Interface/*.cs` | 2 | 是 | |
| `Script/UE/Dynamic/Property/*.cs` | 113 | 是 | |
| `Script/UE/Dynamic/Struct/*.cs` | 5 | 是 | |
| `Script/SourceGenerator/UnrealTypeSourceGenerator.cs` | 967 | 是 | 分段读 1-180 / 180-380 / 380-610 / 610-810 / 810-967 |
| `Script/SourceGenerator/SourceGenerator.csproj` | 11 | 是 | |
| `Script/SourceGenerator/AnalyzerReleases.Unshipped.md` | 9 | 是 | |
| `Template/Game.csproj` | 6 | 是 | |
| `Template/Game.props` | 29 | 是 | |
| `Template/Interop.csproj` | 23 | 是 | |
| `Template/Script.sln` | 54 | 是 | |
| `Template/Shared.props` | 22 | 是 | |
| `Template/UE.csproj` | 9 | 是 | |
| `Template/Dynamic/*.cs` | 4 文件 13–26 行 | 是 | |
| `Template/Override/*.cs` | 4 文件 12–25 行 | 是 | |

**说明**：255 个特性文件共 255 个 `AttributeUsage`（每文件恰好 1 个，pwsh `Select-String` 统计 = 255），类名与文件名 100% 一致（pwsh 逐文件比对无差异），故以"批量提取 + 代表性精读"的方式覆盖：全部 `AttributeUsage` 行、全部构造函数签名、全部 `Value` 字段形态均由带行号的 pwsh 输出取得，代表性文件（`UPropertyAttribute.cs`、`UInterfaceAttribute.cs`、`KismetHideOverridesAttribute.cs`、`CustomThunkTemplatesAttribute.cs`、`DontAutoCollapseCategoriesAttribute.cs`、`ReplicatedAttribute.cs`、`ReplicatedUsingAttribute.cs`、`AllowAbstractAttribute.cs`、`ScriptMethodAttribute.cs` 等）用 `read` 全文精读。

### 0.2 交叉阅读的消费端 / 参考文件（用于验证，不在我的产出责任内）

| 文件 | 用途 |
|---|---|
| `Source/UnrealCSharpCore/Public/CoreMacro/{Class,Function,Property,Generic,MetaData}AttributeMacro.h` | 254 个"特性名字符串"宏 |
| `Source/UnrealCSharpCore/Private/Dynamic/FDynamicGeneratorCore.cpp`（1263 行） | 特性 → UPROPERTY/UFUNCTION/元数据的实际转换 |
| `Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.{h,cpp}` | 特性类 → `FClassReflection*` 注册表 |
| `Source/UnrealCSharpCore/Private/Reflection/FReflection.cpp` | `HasAttribute` / `GetAttributeValue` |
| `Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp` | 描述符解析（特性值来源） |
| `Script/UE/CoreUObject/Utils.cs` | 运行期把 C# 特性写入描述符 |
| `Script/Weavers/UnrealTypeWeaver.cs` | 编织期把 C# 特性写入描述符 |
| `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp` | 模板占位符替换 |
| `Source/Compiler/Private/FCSharpCompilerRunnable.cpp` | `dotnet build` 实际命令行 |
| UE 5.6 引擎：`Runtime/CoreUObject/Public/UObject/ObjectMacros.h`、`Field.h`、`NameTypes.h`、`Private/UObject/MetaData.cpp`、`Programs/Shared/EpicGames.UHT/Specifiers/UhtClassSpecifiers.cs` | 权威元数据键表与布尔语义 |

**未覆盖**：`Script/SourceGenerator/bin|obj/`（构建产物）、`Script/Weavers/UnrealTypeWeaver.cs` 的编织逻辑本体（由另一份报告覆盖，本报告只做重叠性判断）。

---

## 1. 模块职责与架构速览

### 1.1 数据流：C# 特性 → UE 元数据/标志

```
用户 C# 代码               描述符（两套实现）                C++ 消费端                     UE 对象
────────────────────────────────────────────────────────────────────────────────────────────────
[UClass]                  ① 运行期：Utils.cs                FClassReflection 构造         UClass
[UProperty]                  GetClassDescriptor…             Attributes(TSet) +
[BlueprintReadWrite]         (InType.CustomAttributes)       AttributeValues(TMap)
[EditAnywhere]            ② 编织期：UnrealTypeWeaver         ↓
[Category("X")]              EmitPropertyAttributesField    FDynamicGeneratorCore::
                             → "_propertyAttributes" 字符串    SetFlags(FProperty*)  → CPF_* 标志
                                                              SetFlags(UFunction*)  → FUNC_* 标志
                                                              SetFlags(UClass*)     → CLASS_* 标志
                                                              SetMetaData(...)      → meta=(Key="Value")
```

**关键机制 1 —— 特性类名 → 元数据键**：`FDynamicGeneratorCore.cpp:764-772`

```cpp
764: void FDynamicGeneratorCore::SetMetaData(FField* InField, const FString& InAttribute, const FString& InValue)
765: {
766: 	InField->SetMetaData(*InAttribute.LeftChop(9), *InValue);   // 砍掉尾部 "Attribute"（9 字符）
767: }
```
即 `ScriptMethodAttribute` → 元数据键 `ScriptMethod`。**这要求 C# 类名恒为 `<键名>Attribute`**，255 个文件全部满足（无一例外），`LeftChop(9)` 不会截错。

**关键机制 2 —— 特性值只来自构造实参**：两套描述符实现都**只**遍历 `CustomAttribute.ConstructorArguments`：

```csharp
// Script/UE/CoreUObject/Utils.cs:348-353（属性，运行期）
348: foreach (var ConstructorArgument in CustomAttribute.ConstructorArguments)
349: {
350:     PropertyAttributeValues.Add(ConstructorArgument.Value.ToString());
```
```csharp
// Script/Weavers/UnrealTypeWeaver.cs:258-281（编织期）
258: if (attr.HasConstructorArguments)
260:     foreach (var arg in attr.ConstructorArguments)
```
C++ 侧则按"计数 + 下标"读取：`FClassReflection.cpp:186-195`（`AttributeValueCount` / `AttributeValueIndex`），`FReflection.cpp:23-32`（越界或键不存在时返回 **空 `FString`**，不崩溃）。

**结论**：C# 特性类里的 `private string Value { get; set; } = "true";` 是**死字段** —— 从声明到消费，没有任何一处读取它。凡是"无构造参数 + 把 `Value` 初始化成 `"true"`"的特性，C++ 侧一律拿到 `""`。这是本报告最重要的一条系统性缺陷（见 F-CS4-002）。

### 1.2 特性数量与分类

| 事实 | 数值 | 证据 |
|---|---|---|
| `Script/UE/Dynamic/` 特性类总数 | **255** | `Get-ChildItem -Recurse -File`，Class 42 / Enum 3 / Function 62 / Generic 28 / Interface 2 / Property 113 / Struct 5 |
| 5 个 `*AttributeMacro.h` 中的特性名字符串宏 | **254** | pwsh 提取 `FString(TEXT("..."))` |
| `FReflectionRegistry.cpp` 引用的特性宏 | **254**（全部） | pwsh 提取 `CLASS_*` 并与宏表求交 |
| 有 `AttributeUsage` 的文件 | 255/255（每文件 1 个） | `Select-String -AllMatches` 计数 = 255 |
| 类名与文件名不一致的文件 | 0 | pwsh 逐文件 `public class X` 与文件名比对 |
| **零构造参数**的特性类 | **143** | 正则 `public \w+Attribute\([^)]*[A-Za-z_][^)]*\)` 不匹配 |
| 其中位于 C++ 元数据白名单内的 | **62** | 与 `Get{X}MetaDataAttributes()` 六个列表求交 |
| `[AttributeUsage]` 中带 `AllowMultiple`/`Inherited` 的 | **0** | `grep 'AllowMultiple|Inherited\s*=|sealed|Conditional'` → No matches |
| 带 `sealed` 的特性类 | **0** | 同上 |
| 继承自 `OverrideAttribute` 的 | 4（UClass/UStruct/UEnum/UFunction） | `grep 'class OverrideAttribute'` / 逐文件基类提取 |
| 继承自 `UClassAttribute` 的 | 1（**KismetHideOverrides**，可疑） | `KismetHideOverridesAttribute.cs:6` |

### 1.3 C++ 侧的元数据白名单（决定特性是否会变成 `meta=`）

`FDynamicGeneratorCore.cpp` 用六个静态数组把 165 个特性类映射成元数据键：

| 列表 | 行号 | 元素数 |
|---|---|---|
| `GetClassMetaDataAttributes()` | 1040-1062 | 21 |
| `GetStructMetaDataAttributes()` | 1071-1077 | 5 |
| `GetEnumMetaDataAttributes()` | 1086-1090 | 3 |
| `GetInterfaceMetaDataAttributes()` | 1099-1103 | 3 |
| `GetPropertyMetaDataAttributes()` | 1112-1196 | 83 |
| `GetFunctionMetaDataAttributes()` | 1205-1259 | 53 |

注意 `SetMetaData(UClass*)` 使用**接口/类二选一**的分支（`FDynamicGeneratorCore.cpp:806-808`）：`InClass->IsChildOf(UInterface::StaticClass()) ? GetInterfaceMetaDataAttributes() : GetClassMetaDataAttributes()`。

---

## 2. 关键调用链

1. `用户 [UClass] partial class` → `Utils.cs:173-191`（收集 `Script.Dynamic` 命名空间全部特性）→ `FClassReflection.cpp:179-195`（建 `Attributes` + `AttributeValues`）→ `FDynamicGeneratorCore.cpp:716` `HasAttribute(GetUClassAttributeClass())` → `FDynamicGeneratorCore.cpp:726` `SetMetaData(UClass*)`
2. `[EditAnywhere]` → `FDynamicGeneratorCore.cpp:265-268` → `FProperty::SetPropertyFlags(CPF_Edit)`
3. `[Category("X")]` → `FDynamicGeneratorCore.cpp:782-783`（`SetFieldMetaData` 循环命中 `GetCategoryAttributeClass()`）→ `:766` `SetMetaData(TEXT("Category"), TEXT("X"))`
4. `[Blueprintable]` → `FDynamicGeneratorCore.cpp:839-844` → 同时写 `IsBlueprintBase="true"` 与 `BlueprintType="true"`（**硬编码 `TEXT("true")`**，正是对"无值特性"的补救）
5. `[Replicated(COND_X)]` → `FDynamicGeneratorCore.cpp:366-373` → **误读 `ReplicatedUsingAttribute` 的值** → 复制条件丢失（见 F-CS4-001）
6. `dotnet build "<Game>.csproj"`（`FCSharpCompilerRunnable.cpp:386-391`）→ `Template/Game.props:21-23` `AfterBuildPublish` → `Publish` → `Template/Game.props:24-33` `CopyDllsAfterPublish` → `..\..\Content\<PublishDirectory>`
7. `UnrealTypeSourceGenerator.Execute`（`UnrealTypeSourceGenerator.cs:55-65`）→ `Context.Compilation.AssemblyName != "Game"` 时直接 return（只有游戏程序集生效）
8. `LibraryBridgeGenerator.Execute`（`:823-871`）→ 扫 `Script.Binding`/`Script.Library` 中"无 body 的 `static partial` 方法" → 生成 P/Invoke 或 `MethodBridge` 桥接实现

---

## 3. 发现清单

### 3.0 严重度分布

| 严重度 | 条数 | 编号 |
|---|---|---|
| P0 | 0 | — |
| P1 | 4 | F-CS4-001 ~ F-CS4-004 |
| P2 | 7 | F-CS4-005 ~ F-CS4-011 |
| P3 | 9 | F-CS4-012 ~ F-CS4-020 |
| 合计 | **20** | |

---

### P0

未发现 P0 级问题。本模块为纯声明/生成型代码，无内存分配、无并发、无指针运算；唯一可能致崩溃的路径（`FProperty::SetBlueprintReplicationCondition` / `SetMetaData` 空值）已在 §3.1 中确认不会越界（`FReflection.cpp:25-31` 对缺失键返回空 `FString`，`MetaData.cpp:444` 对空值照常 `Add`）。

---

### P1

### [F-CS4-001] `[Replicated(...)]` 分支读取的是 `ReplicatedUsing` 的特性值，复制条件被静默丢弃

- **类别**: Bug
- **严重度**: **P1**
- **复核结论**: 确认 —— 严重度 P1 维持；可达性 活跃
- **可达性**: 活跃（动态类属性生成路径，与脚本后端无关：`FDynamicGeneratorCore::SetFlags(FProperty*, FReflection*)`）
- **复核证据**: `FDynamicGeneratorCore.cpp:366-373` 逐字复核成立 —— `:366` 判定 `GetReplicatedAttributeClass()`，而 `:372` 读的是 `GetReplicatedUsingAttributeClass()`；对照 `:375-384`（索引 0=回调名、1=条件）写法正确。C# 侧 `Script/UE/Dynamic/Property/ReplicatedAttribute.cs:9,14` 复核为 `ReplicatedAttribute(ELifetimeCondition InLifetimeCondition = ELifetimeCondition.COND_None)` + `private ELifetimeCondition LifetimeCondition { get; set; }`。取值兜底 `Source/UnrealCSharpCore/Private/Reflection/FReflection.cpp:23-32` 复核为缺失键/越界均返回 `FString()`（不崩溃）。与 `02-…/03` 的 F-DYN-005 同源，两处结论一致
- **级别变动**: 无
- **文件**: `Source/UnrealCSharpCore/Private/Dynamic/FDynamicGeneratorCore.cpp:366-373`
- **函数**: `FDynamicGeneratorCore::SetFlags(FProperty*, FReflection*)`
- **置信度**: 高

**现状（代码事实）**
```cpp
366: 	if (InReflection->HasAttribute(FReflectionRegistry::Get().GetReplicatedAttributeClass()))
367: 	{
368: 		InProperty->SetPropertyFlags(CPF_Net);
369: 
370: 		InProperty->SetBlueprintReplicationCondition(static_cast<ELifetimeCondition>(
371: 			UKismetStringLibrary::Conv_StringToInt(
372: 				InReflection->GetAttributeValue(FReflectionRegistry::Get().GetReplicatedUsingAttributeClass(), 0))));
373: 	}
374: 
375: 	if (InReflection->HasAttribute(FReflectionRegistry::Get().GetReplicatedUsingAttributeClass()))
376: 	{
377: 		InProperty->SetPropertyFlags(CPF_Net | CPF_RepNotify);
378: 
379: 		InProperty->RepNotifyFunc = FName(
380: 			InReflection->GetAttributeValue(FReflectionRegistry::Get().GetReplicatedUsingAttributeClass(), 0));
381: 
382: 		InProperty->SetBlueprintReplicationCondition(static_cast<ELifetimeCondition>(
383: 			UKismetStringLibrary::Conv_StringToInt(
384: 				InReflection->GetAttributeValue(FReflectionRegistry::Get().GetReplicatedUsingAttributeClass(), 1))));
385: 	}
```

C# 侧对应定义（`Script/UE/Dynamic/Property/ReplicatedAttribute.cs`）：
```csharp
  1: using System;
  2: using Script.CoreUObject;
  4: namespace Script.Dynamic
  5: {
  6:     [AttributeUsage(AttributeTargets.Property)]
  7:     public class ReplicatedAttribute : Attribute
  9:         public ReplicatedAttribute(ELifetimeCondition InLifetimeCondition = ELifetimeCondition.COND_None)
 11:             LifetimeCondition = InLifetimeCondition;
 14:         private ELifetimeCondition LifetimeCondition { get; set; }
```

**调用上下文**
`FDynamicGeneratorCore::SetFlags(FProperty*, FReflection*)` 由动态类生成器在创建 `FProperty` 之后调用（`FDynamicGeneratorCore.cpp:454-456` 在其尾部调用 `SetMetaData(InProperty, InReflection)`）。`GetAttributeValue` 的实现见 `Source/UnrealCSharpCore/Private/Reflection/FReflection.cpp:23-32`：
```cpp
23: FString FReflection::GetAttributeValue(const FClassReflection* InAttribute, const int32 InIndex) const
25: 	const auto FoundAttributeAttributeValue = AttributeValues.Find(InAttribute);
27: 	return FoundAttributeAttributeValue != nullptr
28: 		       ? (FoundAttributeAttributeValue->IsValidIndex(InIndex)
29: 			          ? (*FoundAttributeAttributeValue)[InIndex]
30: 			          : FString())
31: 		       : FString();
```

**问题**
第 372 行位于"属性带 `[Replicated]`"的分支内，却去查 `GetReplicatedUsingAttributeClass()`。用户只写 `[Replicated(COND_OwnerOnly)]` 时，`AttributeValues` 中**没有** `ReplicatedUsingAttribute` 这一项，`GetAttributeValue` 返回空串，`Conv_StringToInt("")` = 0，于是 `SetBlueprintReplicationCondition(COND_None)` —— 用户显式指定的 `ELifetimeCondition` 被无条件覆盖为 `COND_None`。

触发路径：任何 `[Replicated(ELifetimeCondition.COND_Custom / COND_OwnerOnly / ...)]` 的 C# 动态属性；生成出的 `FProperty::GetBlueprintReplicationCondition()` 恒为 `COND_None`，网络复制行为与声明不符（例：本应只发给 Owner 的属性会发给所有客户端）。同时该行还读不到值却不报错、不告警，属于典型的"静默错误"。

对比同一文件中的 `ReplicatedUsing` 分支（377-384）写法正确：索引 0 取回调名、索引 1 取条件，与 `ReplicatedUsingAttribute.cs:9-15` 的 `(string InRepCallbackName, ELifetimeCondition InLifetimeCondition = COND_None)` 一一对应。

**建议**
```cpp
// 第 370-372 行应使用 GetReplicatedAttributeClass()，且索引取 0（ReplicatedAttribute 的
// 第 0 个构造实参就是 InLifetimeCondition）
InProperty->SetBlueprintReplicationCondition(static_cast<ELifetimeCondition>(
    UKismetStringLibrary::Conv_StringToInt(
        InReflection->GetAttributeValue(FReflectionRegistry::Get().GetReplicatedAttributeClass(), 0))));
```
并建议把 `ReplicatedAttribute` 的构造实参与 `ELifetimeCondition` 的映射显式化（C# 端 `ELifetimeCondition` 的整数值被 `.ToString()` 序列化，见 `Utils.cs:350`；两端枚举值必须一致，否则同样静默错位）。

**验证方式**
1. 写一个 C# 动态 Actor，属性标注 `[UProperty, Replicated(ELifetimeCondition.COND_OwnerOnly)]`，在编辑器 `-game` 下检查 `FProperty::GetBlueprintReplicationCondition()`（或 `GetPropertyFlags() & CPF_Net` 后打印条件）。
2. grep 确认修正后全插件只剩 `GetReplicatedAttributeClass()` 与 `GetReplicatedUsingAttributeClass()` 各自成对出现：`grep -n 'GetReplicated\(Using\)\?AttributeClass' Source/UnrealCSharpCore/Private/Dynamic/FDynamicGeneratorCore.cpp`。

---

### [F-CS4-002] 143/255 个特性类没有任何可传参的构造函数，`Value` 字段从不被读取 → C++ 侧拿到空字符串，UE 布尔语义的元数据恒为 false

- **类别**: Bug
- **严重度**: **P1**
- **复核结论**: 部分确认（偏差：`143/255` 与目录分布已实测并完全吻合；但"其中 62 个位于 C++ 元数据白名单内"本次未求交，属未复核子结论）
- **可达性**: 活跃
- **复核证据**: 用 pwsh 对 `Script/UE/Dynamic/**` 实测：`TOTAL=255`，分目录 `Class=42 / Enum=3 / Function=62 / Generic=28 / Interface=2 / Property=113 / Struct=5`（与报告 §1.2 完全一致）；无带参构造函数的文件 `NOARG=143`（正则 `public\s+\w+Attribute\([^)]*[A-Za-z_][^)]*\)` 不匹配）→ **143/255 成立**。引擎侧布尔语义 `Engine/Source/Runtime/CoreUObject/Public/UObject/Field.h:825-830`（`GetBoolMetaData` 要求字符串等于 `"true"`）未重读，沿用既有结论
- **级别变动**: 无
- **文件**: `Script/UE/Dynamic/Property/ScriptNoExportAttribute.cs:8`、`Script/UE/Dynamic/Function/ScriptMethodAttribute.cs:8`、`Script/UE/Dynamic/Enum/BitflagsAttribute.cs:8`、`Script/UE/Dynamic/Generic/BlueprintThreadSafeAttribute.cs:8`、`Script/UE/Dynamic/Property/OnlyPlaceableAttribute.cs:8`、`Script/UE/Dynamic/Function/VariadicAttribute.cs:8` 等 143 个（完整名单见 §3.1 附表）
- **函数**: `UnrealTypeSourceGenerator` 无关；消费端为 `FDynamicGeneratorCore::SetFieldMetaData` + `FDynamicGeneratorCore::SetMetaData(FField*, ...)`
- **置信度**: 高

**现状（代码事实）**

（a）C# 侧的典型写法 —— `Value` 有默认值但没有构造参数：
```csharp
// Script/UE/Dynamic/Function/ScriptMethodAttribute.cs
  5:     [AttributeUsage(AttributeTargets.Method)]
  6:     public class ScriptMethodAttribute : Attribute
  7:     {
  8:         private string Value { get; set; } = "true";
  9:     }
```
```csharp
// Script/UE/Dynamic/Enum/BitflagsAttribute.cs
  5:     [AttributeUsage(AttributeTargets.Enum)]
  6:     public class BitflagsAttribute : Attribute
  7:     {
  8:         private string Value { get; set; } = "true";
  9:     }
```

（b）运行期描述符只读构造实参（`Script/UE/CoreUObject/Utils.cs`）：
```csharp
340: foreach (var CustomAttribute in OutPropertyInfos[i].CustomAttributes)
342:     if (CustomAttribute.AttributeType.Namespace == UClassAttributeNamespace)
344:         var PropertyAttributeValueCount = 0;
346:         PropertyAttributes.Add(CustomAttribute.AttributeType);
348:         foreach (var ConstructorArgument in CustomAttribute.ConstructorArguments)
350:             PropertyAttributeValues.Add(ConstructorArgument.Value.ToString());
352:             PropertyAttributeValueCount++;
355:         PropertyAttributeIndex.Add(PropertyAttributeValueCount);
```
方法属性同理（`Utils.cs:564-569`），类属性同理（`Utils.cs:182-187`）。

（c）编织期同样只读构造实参（`Script/Weavers/UnrealTypeWeaver.cs`）：
```csharp
258: if (attr.HasConstructorArguments)
260:     foreach (var arg in attr.ConstructorArguments)
283: parts.Insert(0, parts.Count.ToString(CultureInfo.InvariantCulture));
285: parts.Insert(0, attrType.FullName.Replace('/', '+'));
287: lines.Add(string.Join("|", parts));
```

（d）C++ 侧取值并写入（`Source/UnrealCSharpCore/Private/Dynamic/FDynamicGeneratorCore.cpp`）：
```cpp
775: template <typename T>
776: static void SetFieldMetaData(T InField, const TArray<FClassReflection*>& InMetaDataAttributes,
777:                              FReflection* InReflection, const TFunction<void()>& InSetMetaData)
778: {
779: 	for (const auto& MetaDataAttribute : InMetaDataAttributes)
780: 	{
781: 		if (InReflection->HasAttribute(MetaDataAttribute))
782: 		{
783: 			FDynamicGeneratorCore::SetMetaData(InField, MetaDataAttribute->GetName(),
784: 			                                   InReflection->GetAttributeValue(MetaDataAttribute));
785: 		}
786: 	}
```
`:783-784` 不传下标，`GetAttributeValue` 默认 `InIndex = 0`（`FReflection.h:17`）；无构造实参时 `AttributeValues` 中该键不存在 → `FReflection.cpp:31` 返回 `FString()`。

（e）UE 侧对空值的语义（引擎源码）：
```cpp
// Engine/Source/Runtime/CoreUObject/Public/UObject/Field.h:825-830
825: 	bool GetBoolMetaData(const TCHAR* Key) const
826: 	{
827: 		const FString& BoolString = GetMetaData(Key);
828: 		// FString == operator does case insensitive comparison
829: 		return (BoolString == "true");
830: 	}
```
```cpp
// Engine/Source/Runtime/CoreUObject/Private/UObject/MetaData.cpp:430-445
430: void FMetaData::SetValue(const UObject* Object, FName Key, const TCHAR* Value)
432: 	check(Key != NAME_None);
444: 	ObjectValues->Add(Key, Value);        // 空字符串照常写入，键存在但值为 ""
```

**调用上下文**
`SetMetaData` 在动态类生成期（编辑器内、GameThread）由 `FDynamicGeneratorCore::SetFlags` 家族调用：属性 `:455`、函数 `:701`、类 `:726`、结构体 `:742`、枚举 `:758`。值来源是加载 C# 程序集时解析的描述符（`FClassReflection.cpp:179-201` / `226-244`）。

**问题**
两个独立事实叠加：
1. **143/255** 个特性类（全部 255 个中有 143 个）**没有任何带参数的构造函数**，因此描述符里的 `AttributeValueCount` 恒为 0，C++ 永远读到 `""`；
2. 其中 **62 个**位于 C++ 的六个元数据白名单中（即它们**会**被写进 UE 元数据），于是 UE 里出现 `meta=(Key="")`。

受影响键中，UE 以布尔语义读取的会**恒为 false**（`Field.h:829` 要求字符串等于 `"true"`）：

| C# 特性 | 写入 UE 的元数据 | UE 语义 | 实际结果 |
|---|---|---|---|
| `[Bitflags]` | `Bitflags=""` | `InterfaceMetadata`，枚举是否按位标志处理（`ObjectMacros.h:1743-1744`） | 位标志枚举退化为普通枚举 |
| `[BlueprintThreadSafe]` | `BlueprintThreadSafe=""` | `ClassMetadata`，函数库可否在非游戏线程调用（`ObjectMacros.h:1248-1249`） | 恒为 false |
| `[OnlyPlaceable]` | `OnlyPlaceable=""` | `PropertyMetadata`，类选择器是否只显示可放置类（`ObjectMacros.h:1447-1448`） | 恒为 false |
| `[Variadic]` | `Variadic=""` | 函数元数据（变参） | 空值 |
| `[ScriptMethod]` / `[ScriptMethodSelfReturn]` | `ScriptMethod=""` | 值语义是**方法名覆盖**（`ObjectMacros.h:1647-1649`） | 空方法名，脚本导出异常 |
| `[HiddenByDefault]` / `[DisableSplitPin]` | `…=""` | 结构体引脚行为 | 恒为 false |
| `[CallableWithoutWorldContext]`、`[BlueprintAutoCast]`、`[CustomizeProperty]`、`[PinShownByDefault]`、`[PinHiddenByDefault]`、`[NeverAsPin]`、`[Latent]`、`[LatentCallbackTarget]`、`[NeedsLatentFixup]`、`[DevelopmentOnly]`、`[Untracked]`、`[ShowTreeView]`、`[ShowOnlyInnerProperties]`、`[MaxLength]`、`[Multiple]`、`[NoEditInline]`、`[NoElementDuplicate]`、`[NoResetToDefault]`、`[IgnoreForMemberInitializationTest]`、`[IgnoreTypePromotion]`、`[ForceAsFunction]`、`[DeprecatedFunction]`/`[DeprecatedProperty]`/`[DeprecatedNode]`、`[RelativePath]`、`[RelativeToGameDir]`、`[RelativeToGameContentDir]`、`[ContentDir]`、`[ConfigHierarchyEditable]`、`[LongPackageName]`、`[MakeStructureDefaultValue]`、`[HideAlphaChannel]`、`[HideInDetailPanel]`、`[HideViewOptions]`、`[EditConditionHides]`、`[EditFixedOrder]`、`[HideAssetPicker]`、`[BlueprintCompilerGeneratedDefaults]`、`[BlueprintSpawnableComponent]`、`[ChildCanTick]`、`[ChildCannotTick]`、`[DebugTreeLeaf]`、`[IgnoreCategoryKeywordsInSubclasses]`、`[ShowWorldContextPin]`、`[UsesHierarchy]`、`[ConversionRoot]`、`[CannotImplementInterfaceInBlueprint]`、`[AllowAnyActor]`、`[ArrayParam]`、`[Bitmask]`、`[NativeBreakFunc]`、`[NativeMakeFunc]`、`[ScriptNoExport]`（`ScriptNoExportAttribute.cs:8` 同为 `= "true"` 默认） | `Key=""` | 同上 | 恒为 false 或空串 |

**62 个受影响特性的完整机器可核对清单**（pwsh 求交结果）：
`AllowAnyActor, ArrayParam, Bitflags, Bitmask, BlueprintAutoCast, BlueprintCompilerGeneratedDefaults, BlueprintSpawnableComponent, BlueprintThreadSafe, CallableWithoutWorldContext, CannotImplementInterfaceInBlueprint, ChildCannotTick, ChildCanTick, ConfigHierarchyEditable, ContentDir, ConversionRoot, CustomizeProperty, DebugTreeLeaf, DeprecatedFunction, DeprecatedNode, DeprecatedProperty, DevelopmentOnly, DisableSplitPin, EditConditionHides, EditFixedOrder, ForceAsFunction, HiddenByDefault, HideAlphaChannel, HideAssetPicker, HideInDetailPanel, HideViewOptions, IgnoreCategoryKeywordsInSubclasses, IgnoreForMemberInitializationTest, IgnoreTypePromotion, Latent, LatentCallbackTarget, LongPackageName, MakeStructureDefaultValue, MaxLength, Multiple, NativeBreakFunc, NativeMakeFunc, NeverAsPin, NoEditInline, NoElementDuplicate, NoResetToDefault, NotBlueprintThreadSafe, OnlyPlaceable, PinHiddenByDefault, PinShownByDefault, RelativePath, RelativeToGameContentDir, RelativeToGameDir, ScriptMethod, ScriptMethodSelfReturn, ScriptNoExport, ShowOnlyInnerProperties, ShowTreeView, ShowWorldContextPin, Untracked, UsesHierarchy, Variadic`

另有 9 个"双构造"特性有**同一个陷阱的第二形态**：存在无参构造，但无参构造只设置 `Value`（同样不被读取），因此 `[X]` 得到 `""` 而 `[X("true")]` 才得到 `"true"`：
`AllowAbstractAttribute.cs:8,13`、`AllowPreserveRatioAttribute.cs:8,13`、`AlwaysAsPinAttribute.cs:8,13`、`BlueprintBaseOnlyAttribute.cs:8,13`、`CallInEditorAttribute.cs:8,13`、`EditorConfigAttribute.cs:8,13`、`ExactClassAttribute.cs:8,13`、`ExposeOnSpawnAttribute.cs:8,13`、`InlineEditConditionToggleAttribute.cs:8,13`。
例：
```csharp
// Script/UE/Dynamic/Property/AllowAbstractAttribute.cs
  8:         public AllowAbstractAttribute()
  9:         {
 10:             Value = "true";
 11:         }
 13:         public AllowAbstractAttribute(string InValue)
 14:         {
 15:             Value = InValue;
 16:         }
 18:         private string Value { get; set; }
```
`[AllowAbstract]` → `meta=(AllowAbstract="")` → `GetBoolMetaData` 返回 false；只有 `[AllowAbstract("true")]` 才正确。

**建议**
三种修法（按推荐度排序）：
1. **让 C++ 侧兜底**（改动最小、覆盖面最大）：在 `FDynamicGeneratorCore::SetFieldMetaData` 中，当取到的值为空串而特性存在时，写入 `"true"`：
   ```cpp
   auto Value = InReflection->GetAttributeValue(MetaDataAttribute);
   if (Value.IsEmpty())
   {
       // 无构造实参的"存在即真"型元数据，UE 要求字面量 "true"（Field.h:829）
       Value = TEXT("true");
   }
   FDynamicGeneratorCore::SetMetaData(InField, MetaDataAttribute->GetName(), Value);
   ```
   注意副作用：对**值语义**为字符串的键（`ScriptMethod`、`ScriptConstantHost`、`Category`…）写 `"true"` 也是错的；因此更稳妥的做法是给 `FClassReflection` 增加"该特性是否为 bool 标志"的信息，或对 `ScriptMethod` 一类无参数字符串键保持空并单独修正 C# 侧。
2. **C# 侧统一改为带默认值的字符串构造**（逐个文件改）：把
   `private string Value { get; set; } = "true";`
   改为
   `public XAttribute(string InValue = "true") { Value = InValue; }`
   这样描述符里就会出现实参 `"true"`。**这是最符合当前数据流的修法**，因为默认参数值会被 C# 编译器写进特性 blob（`Utils.cs:348` 能读到）。
3. 混合：对 62 个纯布尔键采用方案 2；对 `Value` 语义是字符串的键保留现状并在文档中说明必须显式传值。

**验证方式**
1. grep 统计零构造参数文件数：
   `Get-ChildItem Script/UE/Dynamic -Recurse -Filter *.cs | Where-Object { (Get-Content $_ -Raw) -notmatch 'public\s+\w+Attribute\([^)]*[A-Za-z_][^)]*\)' } | Measure-Object`
2. 编辑器内跑一个带 `[Bitflags]` 的 C# 枚举，检查 `UEnum` 的元数据：`Enum->GetMetaData(TEXT("Bitflags")) == TEXT("true")`。
3. 或者在 C++ 侧对已生成字段加断言：`check(!Field->HasMetaData(Key) || !Field->GetMetaData(Key).IsEmpty())`。

---

### [F-CS4-003] `[VisibleDefaultsOnly]` 在属性标志映射中完全没有分支，特性静默无效

- **类别**: Bug
- **严重度**: **P1**
- **复核结论**: ✅确认 —— 严重度 P1 维持；可达性 活跃
- **文件**: `Source/UnrealCSharpCore/Private/Dynamic/FDynamicGeneratorCore.cpp:270-288`（缺失点）；`Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp:1488`（唯一引用）；`Script/UE/Dynamic/Property/VisibleDefaultsOnlyAttribute.cs:5-8`（定义）
- **函数**: `FDynamicGeneratorCore::SetFlags(FProperty*, FReflection*)`
- **置信度**: 高

**现状（代码事实）**
可见性三态中只实现了两个（`FDynamicGeneratorCore.cpp:270-288`）：
```cpp
270: 	if (InReflection->HasAttribute(FReflectionRegistry::Get().GetEditInstanceOnlyAttributeClass()))
271: 	{
272: 		InProperty->SetPropertyFlags(CPF_Edit | CPF_DisableEditOnTemplate);
273: 	}
275: 	if (InReflection->HasAttribute(FReflectionRegistry::Get().GetEditDefaultsOnlyAttributeClass()))
276: 	{
277: 		InProperty->SetPropertyFlags(CPF_Edit | CPF_DisableEditOnInstance);
278: 	}
280: 	if (InReflection->HasAttribute(FReflectionRegistry::Get().GetVisibleAnywhereAttributeClass()))
281: 	{
282: 		InProperty->SetPropertyFlags(CPF_Edit | CPF_EditConst);
283: 	}
285: 	if (InReflection->HasAttribute(FReflectionRegistry::Get().GetVisibleInstanceOnlyAttributeClass()))
286: 	{
287: 		InProperty->SetPropertyFlags(CPF_Edit | CPF_EditConst | CPF_DisableEditOnTemplate);
288: 	}
```
缺 `VisibleDefaultsOnly`（应为 `CPF_Edit | CPF_EditConst | CPF_DisableEditOnInstance`）。

全局搜索证明该特性无其他消费者：
```
grep -n 'GetVisibleDefaultsOnlyAttributeClass' Source/**/*.cpp
→ FReflectionRegistry.cpp:1488   （仅此一处 = getter 自身的定义）
```
（对比：`GetEditDefaultsOnlyAttributeClass()` 命中 `FDynamicGeneratorCore.cpp:275`。）

**调用上下文**
`SetFlags(FProperty*, FReflection*)` 在动态类的属性创建流程内被调用（同文件 `:455` 后接 `SetMetaData`）。`VisibleDefaultsOnlyAttribute` 在 `Source/UnrealCSharpCore/Public/CoreMacro/PropertyAttributeMacro.h:45` 有宏、在 `FReflectionRegistry.h:239` 有 getter、在 `FReflectionRegistry.cpp:312-313` 有注册 —— 即"能被 C# 识别、能被描述符携带"，但**没有任何一处查询它**。

**问题**
用户在 C# 里写 `[UProperty, VisibleDefaultsOnly]`，期望得到 UE 的 `UPROPERTY(VisibleDefaultsOnly)` 语义（在 CDO/蓝图中可见、实例上只读）。实际结果是：属性既没有 `CPF_Edit` 也没有 `CPF_EditConst`，在细节面板中**完全不出现**。这是"特性存在但无声失效"的典型（编辑器里看不到属性，正是任务描述的高价值症状）。

**建议**
在 `:285` 之前插入：
```cpp
if (InReflection->HasAttribute(FReflectionRegistry::Get().GetVisibleDefaultsOnlyAttributeClass()))
{
    InProperty->SetPropertyFlags(CPF_Edit | CPF_EditConst | CPF_DisableEditOnInstance);
}
```
（UE 的对应实现见 `UPROPERTY(VisibleDefaultsOnly)` 展开为 `CPF_Edit | CPF_EditConst | CPF_DisableEditOnInstance`。）

**验证方式**
1. grep 断言：修正后 `grep -c 'GetVisibleDefaultsOnlyAttributeClass' Source/UnrealCSharpCore/Private/Dynamic/FDynamicGeneratorCore.cpp` ≥ 1。
2. 单元测试：定义 `[UProperty, VisibleDefaultsOnly] public int X { get; set; }`，断言 `Property->HasAnyPropertyFlags(CPF_Edit | CPF_EditConst | CPF_DisableEditOnInstance)`。

---

### [F-CS4-004] `[Localized]` 与 `[SealedEvent]` 的命中分支体为空 —— 明确的无操作特性

- **类别**: Bug（未实现功能以已实现的形式暴露给用户）
- **严重度**: **P2**
- **复核结论**: ⚠️部分 —— 严重度由 P1 校正为 **P2**；可达性 活跃
- **文件**: `Source/UnrealCSharpCore/Private/Dynamic/FDynamicGeneratorCore.cpp:320-323`、`:544-548`
- **函数**: `FDynamicGeneratorCore::SetFlags(FProperty*, FReflection*)`、`FDynamicGeneratorCore::SetFlags(UFunction*, FReflection*)`
- **置信度**: 高

**现状（代码事实）**
```cpp
320: 	if (InReflection->HasAttribute(FReflectionRegistry::Get().GetLocalizedAttributeClass()))
321: 	{
322: 		// @TODO
323: 	}
```
```cpp
544: #if WITH_EDITOR
545: 	if (InReflection->HasAttribute(FReflectionRegistry::Get().GetSealedEventAttributeClass()))
546: 	{
547: 	}
548: #endif
```
C# 侧两个特性都有完整定义（`Script/UE/Dynamic/Property/LocalizedAttribute.cs:6`、`Script/UE/Dynamic/Function/SealedEventAttribute.cs:6`），并且在 `PropertyAttributeMacro.h:5` / `FunctionAttributeMacro.h:7` 有对应宏、在注册表中已注册（`FReflectionRegistry.cpp:252-253` / `354-355`）。

**调用上下文**
`[Localized]` 应设置 `CPF_Localized`（UE 侧还有 `CPF_Config` 之外的本地化路径）；`[SealedEvent]` 应设置 `FUNC_Final`（UE 的 `UFUNCTION(SealedEvent)` → `FUNC_Final`）。两者都在动态类生成期被检查，然后什么也不做。

**问题**
这是"显式留白"。与 F-CS4-003 的区别在于这里有 `@TODO` 注释与空块，说明作者知情；但对用户而言仍然是**静默失效**：`[Localized]` 的 `FText` 属性不会被纳入本地化收集，`[SealedEvent]` 的事件不会被标记为 final（子类可继续覆写）。这类"暴露了 API 但未接线"的特性比没有该 API 更危险。

**建议**
- 短期：在 `FDynamicGeneratorCore` 的这两个分支里加 `UE_LOG(LogUnrealCSharp, Warning, TEXT("Attribute %s is not implemented yet"), ...)`，让用户至少能在日志里看到；或者直接从 `Script/UE/Dynamic/` 删除这两个类并移除宏/注册，避免误用。
- 长期实现：`Localized` → `InProperty->SetPropertyFlags(CPF_Localized)`；`SealedEvent` → `InFunction->FunctionFlags |= FUNC_Final`。
- 建议顺带清理同类"空实现"：全文检索 `{` + `// @TODO` + `}` 模式，`FDynamicGeneratorCore.cpp:450-451` 还有一处（在 `SetFlags(FProperty*)` 尾部）。

**验证方式**
`grep -n -A2 'GetLocalizedAttributeClass()\|GetSealedEventAttributeClass()' Source/UnrealCSharpCore/Private/Dynamic/FDynamicGeneratorCore.cpp` 确认分支体是否仍为空。

---

### P2

### [F-CS4-005] `CustomThunkTemplatesAttribute` / `DontAutoCollapseCategoriesAttribute` 在 C++ 侧零引用 —— 纯装饰（死代码）

- **类别**: 死代码
- **严重度**: **P2**
- **复核结论**: 确认 —— 严重度 P2 维持；可达性 活跃
- **可达性**: 活跃（特性类存在即对用户可见；但"无实现"意味着它不产生任何效果）
- **复核证据**: 对 `Source/` 全部模块 grep `GetCustomThunkTemplatesAttributeClass|GetDontAutoCollapseCategoriesAttributeClass` → **0 命中**（同时对照 `GetVisibleDefaultsOnlyAttributeClass` 命中 2 处，证明该 grep 模式有效），故"无 getter、未注册、C++ 侧零消费"成立。与 `08-…/01` 的 F-DEAD-004 同源，两处结论一致
- **级别变动**: 无
- **文件**: `Script/UE/Dynamic/Class/CustomThunkTemplatesAttribute.cs:5-14`、`Script/UE/Dynamic/Class/DontAutoCollapseCategoriesAttribute.cs:5-14`
- **函数**: `FReflectionRegistry`（无对应 getter）
- **置信度**: 高

**现状（代码事实）**
```csharp
// Script/UE/Dynamic/Class/CustomThunkTemplatesAttribute.cs
  5:     [AttributeUsage(AttributeTargets.Class)]
  6:     public class CustomThunkTemplatesAttribute : Attribute
  7:     {
  8:         public CustomThunkTemplatesAttribute(string InValue)
```
```csharp
// Script/UE/Dynamic/Class/DontAutoCollapseCategoriesAttribute.cs
  5:     [AttributeUsage(AttributeTargets.Class)]
  6:     public class DontAutoCollapseCategoriesAttribute : Attribute
  7:     {
  8:         public DontAutoCollapseCategoriesAttribute(string InValue)
```

**调用上下文 / grep 证据**

| 检查项 | 结果 |
|---|---|
| 5 个 `*AttributeMacro.h` 中是否有对应宏 | **无**（254 个宏里没有这两个）；`grep 'CUSTOM_THUNK_TEMPLATES\|DONT_AUTO_COLLAPSE' Source/` → **No matches** |
| `FReflectionRegistry.cpp` 是否注册 | **未注册**（`FReflectionRegistry.h` 无 `GetCustomThunkTemplatesAttributeClass` / `GetDontAutoCollapseCategoriesAttributeClass`） |
| 全插件（`Source/` + `Script/` + `Template/`，869 个文件，排除 ThirdParty/obj/bin）字符串命中 | `CustomThunkTemplatesAttribute` = 2（仅为 C# 类声明与本文件）；`DontAutoCollapseCategoriesAttribute` = 2（同上） |

**参考**：`DontAutoCollapseCategories` 在 UE 里是**真实存在的 UHT 类说明符**：
```csharp
// Engine/Source/Programs/Shared/EpicGames.UHT/Specifiers/UhtClassSpecifiers.cs:293-298
293: 		[UhtSpecifier(Extends = UhtTableNames.Class, ValueType = UhtSpecifierValueType.NonEmptyStringList)]
294: 		private static void DontAutoCollapseCategoriesSpecifier(UhtSpecifierContext specifierContext, List<StringView> value)
296: 			UhtClass classObj = (UhtClass)specifierContext.Type;
297: 			classObj.AutoCollapseCategories.RemoveSwapRange(value);
```
而 `CustomThunkTemplates` 在 UE 5.6 的 UHT 源码与 `ObjectMacros.h` 中**均查不到**（`grep 'CustomThunkTemplates' Engine/Source/Programs/Shared/EpicGames.UHT` → 无命中），属于 UHT 之外的、插件自造的名字。

**问题**
- `[CustomThunkTemplates("...")]` 对 C# 用户完全无效：不生成任何东西、不影响任何标志、不产生任何告警。它既不是 UE 说明符，也没有插件内实现。
- `[DontAutoCollapseCategories("...")]` 是 UE 真实说明符却未接线：用户以为能撤销 `AutoCollapseCategories`，实际 `UClass::ClassFlags` 不变。

**建议**
要么补实现（`DontAutoCollapseCategories` → 从 `AutoCollapseCategories` 元数据里移除对应项；`CustomThunkTemplates` → 需要先定义它在 C# 语义下的含义，或直接删除），要么从 `Script/UE/Dynamic/Class/` 删掉这两个文件。当前状态是"API 表面积大于实现"，属于应当收敛的债务。

**验证方式**
```powershell
Select-String -Path (Get-ChildItem Source,Script,Template -Recurse -File -Include *.h,*.cpp,*.cs).FullName `
  -Pattern 'CustomThunkTemplates|DontAutoCollapseCategories'
# 期望（修正后）：Source/ 下至少出现一次
```

---

### [F-CS4-006] 33 个特性类"已注册但全插件无人查询" —— 静默无效的特性（死代码清单）

- **类别**: 死代码
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Source/UnrealCSharpCore/Public/Reflection/FReflectionRegistry.h:143,151,153,159,173,177,191,195,239,247,253,263,283,285,297,317,323,333,341,347,353,379,381,387,395,399,405,411,417,423,429,435,439`（33 个 getter 声明）
- **函数**: `FReflectionRegistry::Get{X}AttributeClass()`
- **置信度**: 高

**现状（代码事实）**
这些特性在 `Script/UE/Dynamic/` 中有定义、在 `*AttributeMacro.h` 中有宏、在 `FReflectionRegistry.cpp` 中被 `GetClass(COMBINE_NAMESPACE(NAMESPACE_ROOT, NAMESPACE_DYNAMIC), CLASS_X_ATTRIBUTE)` 解析成 `FClassReflection*` 缓存起来 —— 但除了 getter 自身的返回语句（`FReflectionRegistry.cpp` 内部），**全插件 7 个 C++ 模块中没有任何一处调用**。

**调用上下文 / grep 证据**
度量方法：抽取 `FReflectionRegistry.h` 中全部 254 个 `Get\w+AttributeClass() const;` 声明，在 `Source/` 全部模块（排除 `ThirdParty/` 与 `FReflectionRegistry.*` 自身）统计 `Get\w+AttributeClass` 出现次数，命中数为 0 的即为"无人查询"：

```powershell
$getters = Select-String FReflectionRegistry.h -Pattern '(Get\w+AttributeClass)\(\)\s*const;' -AllMatches ...
$files   = Get-ChildItem Source -Recurse -Include *.h,*.cpp,*.inl | ? { $_ -notmatch 'ThirdParty' -and $_.Name -notlike 'FReflectionRegistry.*' }
# → 未使用 getter 数: 33 / 254
```

| # | 特性（C# 类名去掉 Attribute） | getter 声明 | 外部命中数 | 判定 | 证据 |
|---|---|---|---|---|---|
| 1 | `Abstract` | `FReflectionRegistry.h:139` | 0 | 死代码（`Abstract` 的 `CLASS_Abstract` 只在 `FDynamicInterfaceGenerator.cpp:170` 对接口硬编码设置） | `grep 'GetAbstractAttributeClass' Source` → 仅 `.h` 声明 + `.cpp:1238` 定义 |
| 2 | `AdvancedClassDisplay` | `:191` | 0 | 死代码 | 同上模式 |
| 3 | `AutoCollapseCategories` | `:183` | 0 | 死代码（注意：`AutoCollapseCategoriesSpecifier` 在 UHT 里是真实说明符） | |
| 4 | `AutoExpandCategories` | `:181` | 0 | 死代码 | |
| 5 | `CollapseCategories` | `:185` | 0 | 死代码（UE 侧对应 `CLASS_CollapseCategories`） | |
| 6 | `ComponentWrapperClass` | `:177` | 0 | 死代码 | |
| 7 | `ConfigDoNotCheckDefaults` | `:165` | 0 | 死代码 | |
| 8 | `Const` | `:159` | 0 | 死代码 | |
| 9 | `CustomConstructor` | `:151` | 0 | 死代码 | |
| 10 | `DefaultConfig` | `:167` | 0 | 死代码 | |
| 11 | `DefaultToInstanced` | `:157` | 0 | 死代码 | |
| 12 | `Deprecated` | `:161` | 0 | 死代码 | |
| 13 | `DocumentationPolicy` | `:304` | 0 | 死代码（UE 侧是 `FString` 值语义并要求 `"Strict"`，`ObjectMacros.h:1189-1190`） | |
| 14 | `DontCollapseCategories` | `:187` | 0 | 死代码 | |
| 15 | `EarlyAccessPreview` | `:193` | 0 | 死代码 | |
| 16 | `EditInline` | `:450` | 0 | 死代码（UE 的 `EditInline` 是 `Instanced` 的历史别名；`[Instanced]` 有实现，`[EditInline]` 没有） | |
| 17 | `EditInlineNew` | `:169` | 0 | 死代码（UE 侧 `CLASS_EditInlineNew`） | |
| 18 | `EditorConfig` | `:143` | 0 | 死代码 | |
| 19 | `Experimental` | `:141` | 0 | 死代码 | |
| 20 | `HideDropdown` | `:173` | 0 | 死代码 | |
| 21 | `HideFunctions` | `:179` | 0 | 死代码 | |
| 22 | `Intrinsic` | `:153` | 0 | 死代码 | |
| 23 | `NoExport` | `:137` | 0 | 死代码 | |
| 24 | `NotBlueprintable` | `:135` | 0 | 死代码（与 `[Blueprintable]` 不对称：后者在 `FDynamicGeneratorCore.cpp:839-844` 有实现） | |
| 25 | `NotBlueprintType` | `:131` | 0 | 死代码（与 `[BlueprintType]` 不对称：后者在 `:830-837`） | |
| 26 | `NotEditInlineNew` | `:171` | 0 | 死代码 | |
| 27 | `PerObjectConfig` | `:163` | 0 | 死代码 | |
| 28 | `PrioritizeCategories` | `:189` | 0 | 死代码 | |
| 29 | `ShortTooltip` | `:302` | 0 | 死代码（`ToolTip` 可用，`ShortTooltip` 不可用） | |
| 30 | `ShowCategories` | `:175` | 0 | 死代码 | |
| 31 | `SparseClassDataType` | `:195` | 0 | 死代码 | |
| 32 | `VisibleDefaultsOnly` | `:239` | 0 | **死代码且构成功能缺陷**（见 F-CS4-003，建议提升优先级） | |
| 33 | `Within` | `:147` | 0 | 死代码（`UCLASS(Within=...)` 的 `CLASS_Within` 未设置） | |

**问题**
这 33 个特性构成了"可见但无行为"的 API：IDE 会补全它们、编译通过、不报错、不告警，编辑器里却没有任何效果。对用户而言，"特性无效"比"特性不存在"更难排查。其中 `NotBlueprintable`/`NotBlueprintType`/`VisibleDefaultsOnly`/`Within`/`DefaultConfig`/`EditInlineNew` 属于常用 UCLASS/UPROPERTY 说明符，误用概率高。

**建议**
1. 为不可能短期实现的特性加"未实现"诊断或运行期告警，把这些条目集中登记在一张表里（可放在 `FReflectionRegistry` 旁），避免以后再逐个 grep。
2. 至少把与"已有正向实现"成对的负向特性补上（`NotBlueprintable`/`NotBlueprintType`/`NotEditInlineNew`/`VisibleDefaultsOnly`/`DontCollapseCategories`），它们是同构代码，成本极低。
3. 长期：用表驱动（键名 → 标志位/元数据）替代 `if (HasAttribute(...))` 的长链条，减少"漏一条"的概率 —— 当前 `SetFlags(FProperty*)` 是 34 个近似 if 块的线性展开（`FDynamicGeneratorCore.cpp:257-446`），`SetFlags(UFunction*)` 同理（`:477-690`）。

**验证方式**
```powershell
# 每次改动后重跑本表
$getters = Select-String Source/UnrealCSharpCore/Public/Reflection/FReflectionRegistry.h -Pattern '(Get\w+AttributeClass)\(\)\s*const;' -AllMatches |
           % { $_.Matches } | % { $_.Groups[1].Value }
$files = Get-ChildItem Source -Recurse -Include *.h,*.cpp,*.inl |
         ? { $_.FullName -notmatch 'ThirdParty' -and $_.Name -notlike 'FReflectionRegistry.*' }
$all = Select-String $files.FullName -Pattern '(Get\w+AttributeClass)' -AllMatches |
       % { $_.Matches } | % { $_.Groups[1].Value }
$getters | ? { $all -notcontains $_ }   # 期望：仅剩有意保留的项
```

---

### [F-CS4-007] 诊断 ID 与 `AnalyzerReleases.Unshipped.md` 不一致，且该 md 从未被声明为 `AdditionalFiles` → 发布跟踪完全失效

- **类别**: Bug / 可优化
- **严重度**: **P3**
- **复核结论**: ⚠️部分 —— 严重度由 P2 校正为 **P3**；可达性 活跃
- **文件**: `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:15-48`、`Script/SourceGenerator/AnalyzerReleases.Unshipped.md:1-9`、`Script/SourceGenerator/SourceGenerator.csproj:1-11`
- **函数**: `UnrealTypeSourceGenerator`（`DiagnosticDescriptor` 静态字段）
- **置信度**: 高（ID 与 md 内容已逐字比对；RS2008 的具体触发是 Roslyn 分析器的标准行为，未在本机编译验证 → 该子结论置信度中）

**现状（代码事实）**

代码里的 5 个 ID 全部形如 `UC_ERROR_0N`（**含下划线**）：
```csharp
 15:         public static readonly DiagnosticDescriptor ErrorDynamicClassNotAPartialClass = new DiagnosticDescriptor(
 16:             "UC_ERROR_01",
 17:             "UClass or UStruct must be a partial class", "{0} \"{1}\" must be a partial class",
 18:             "UnrealCSharp",
 19:             DiagnosticSeverity.Error,
 20:             isEnabledByDefault: true);
 22:         public static readonly DiagnosticDescriptor ErrorFileNameNotMatch = new DiagnosticDescriptor(
 23:             "UC_ERROR_02",
 24:             "The file name and class name do not match", "The file where {0} \"{1}\" is located must be \"{2}\"",
 29:         public static readonly DiagnosticDescriptor ErrorTypeNameNotMatch = new DiagnosticDescriptor(
 30:             "UC_ERROR_03",
 31:             "The name of dynamic class is error", "{0}",
 36:         public static readonly DiagnosticDescriptor ErrorUClassHasNoBaseClass = new DiagnosticDescriptor(
 37:             "UC_ERROR_04",
 39:         public static readonly DiagnosticDescriptor ErrorTypeMustBeUnique = new DiagnosticDescriptor(
 44:             "UC_ERROR_05",
```

md 里登记的是**无下划线**的 ID，且描述与代码语义错位：
```
AnalyzerReleases.Unshipped.md
1: ### Dynamic Rules
3: | Rule ID    | Category     | Severity | Notes  |
5: | UC_ERROR01 | UnrealCSharp | Error    | The partial keyword must be added to the dynamic class/dynamic struct ... |
6: | UC_ERROR02 | UnrealCSharp | Error    | Dynamic classes inherited from blueprints must end with "_C", dynamic AActor classes ... |
7: | UC_ERROR03 | UnrealCSharp | Error    | The file name where the dynamic class and dynamic struct are located must be consistent ... |
8: | UC_ERROR04 | UnrealCSharp | Error    | UClass must have a base class |
9: | UC_ERROR05 | UnrealCSharp | Error    | Type must be unique |
```

**对照表（C# 代码 ↔ AnalyzerReleases.Unshipped.md）**

| 代码 ID | 代码 Title（`UnrealTypeSourceGenerator.cs`） | 代码 Category / Severity | md 中的 ID | md 中该行的 Notes | 一致性判定 |
|---|---|---|---|---|---|
| `UC_ERROR_01` (:16) | UClass or UStruct must be a partial class | `UnrealCSharp` / Error (:18-19) | `UC_ERROR01` (:5) | partial keyword 必须添加 | **ID 格式不一致**；语义一致 |
| `UC_ERROR_02` (:23) | **The file name and class name do not match** | `UnrealCSharp` / Error (:26-27) | `UC_ERROR02` (:6) | **继承来的动态类命名规则（_C / A / U / F）** | **ID 不一致 + 语义错位** |
| `UC_ERROR_03` (:30) | **The name of dynamic class is error** | `UnrealCSharp` / Error (:33-34) | `UC_ERROR03` (:7) | **文件名必须与类名一致** | **ID 不一致 + 语义错位（与 02 互换）** |
| `UC_ERROR_04` (:37) | UClass must have a base class | `UnrealCSharp` / Error (:40-41) | `UC_ERROR04` (:8) | UClass must have a base class | **ID 格式不一致**；语义一致 |
| `UC_ERROR_05` (:44) | Type must be unique | `UnrealCSharp` / Error (:47-48) | `UC_ERROR05` (:9) | Type must be unique | **ID 格式不一致**；语义一致 |
| — | — | — | — | — | 代码中**没有**未登记的诊断（共 5 个，全部在 md 中有一行） |

同时 `SourceGenerator.csproj` 里**没有**把 md 声明为 `AdditionalFiles`：
```xml
  1: <Project Sdk="Microsoft.NET.Sdk">
  2:   <PropertyGroup>
  3:     <TargetFramework>netstandard2.0</TargetFramework>
  4: 	<IsRoslynComponent>true</IsRoslynComponent>
  5: 	<EnforceExtendedAnalyzerRules>true</EnforceExtendedAnalyzerRules>
  6:   </PropertyGroup>
  7: 	<ItemGroup>
  8: 		<PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.5.0" PrivateAssets="all"/>
  9: 		<PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.3.4" PrivateAssets="all" />
 10: 	</ItemGroup>
 11: </Project>
```

**调用上下文**
`FSolutionGenerator::Generator`（`Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:32-53`）把 `SourceGenerator.csproj`、`UnrealTypeSourceGenerator.cs`、`AnalyzerReleases.Unshipped.md` 三个文件原样复制到 `<Script>/SourceGenerator/`。校验实际生成物：
- `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:18` = `private const string GameAssemblyName = "Game";`（占位符已替换），`:21,28,35,42,49` = `"UC_ERROR_01"`…（带下划线）；
- `Script/SourceGenerator/AnalyzerReleases.Unshipped.md` 与模板内容一致（`UC_ERROR01`…无下划线）。
- 全 `Script/` 树（`*.csproj`/`*.props`/`*.targets`/`*.json`）grep `AnalyzerReleases|AdditionalFiles` → **无任何命中**。

**问题**
两点独立缺陷：
1. **ID 不同名**：`UC_ERROR_01` vs `UC_ERROR01`，发布跟踪文件按 ID 精确匹配，因此这份 md 一行也匹配不上（`UC_ERROR02`/`UC_ERROR03` 连语义都是反的）。
2. **md 不是输入项**：微软的 `AnalyzerReleases` 机制要求把文件作为 `AdditionalFiles` 传给分析器（标准模板写法是 `<AdditionalFiles Include="AnalyzerReleases.Unshipped.md" />`），本工程没有。因此 `Microsoft.CodeAnalysis.Analyzers` 的 ReleaseTracking 分析器看不到任何已登记 ID，会对 5 个 `DiagnosticDescriptor` 各报一次 **RS2008**（"Enable analyzer release tracking"）警告；由于同时打开了 `<EnforceExtendedAnalyzerRules>true</EnforceExtendedAnalyzerRules>`，该项目的分析器规则被提升为强制级别，这类噪声更应消除。

**建议**
```xml
<!-- SourceGenerator.csproj 增加 -->
<ItemGroup>
  <AdditionalFiles Include="AnalyzerReleases.Unshipped.md" />
</ItemGroup>
```
并把代码中的 ID 改为与 md 一致（推荐去掉下划线，`UC_ERROR01`…`UC_ERROR05`），同时修正 md 第 6/7 行的 Notes 顺序（文件名规则 ↔ 命名规则互换）。若想保留下划线风格，则应反过来改 md 的 5 行 ID。

**验证方式**
```powershell
# 1) ID 集合一致性
Select-String Script/SourceGenerator/UnrealTypeSourceGenerator.cs -Pattern '"UC_ERROR_\d+"' -AllMatches
Select-String Script/SourceGenerator/AnalyzerReleases.Unshipped.md -Pattern 'UC_ERROR\d+'
# 2) 在 Script/SourceGenerator 下 dotnet build，确认不再出现 RS2008
```

---

### [F-CS4-008] 源生成器是旧式 `ISourceGenerator`（无增量管道），且生成的文件既无 `// <auto-generated>` 头、hint name 也不是 Roslyn 识别的生成文件后缀

- **类别**: 性能 / 可优化
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:11`、`:74-111`、`:203`、`:232`、`:875`
- **函数**: `UnrealTypeSourceGenerator.Execute(GeneratorExecutionContext)`、`UnrealTypeSourceGenerator.Initialize(GeneratorInitializationContext)`、`LibraryBridgeGenerator.EmitBridges`
- **置信度**: 高（"非增量"与"无 auto-generated 头"为代码事实）；关于"分析器会分析生成代码"的具体后果，置信度中（未在本机编译比对警告数量）

**现状（代码事实）**
```csharp
 10:     [Generator]
 11:     public class UnrealTypeSourceGenerator : ISourceGenerator
 12:     {
 13:         private const string GameAssemblyName = "GameNamePlaceholder";
 ...
 55:         public void Execute(GeneratorExecutionContext Context)
 ...
237:         public void Initialize(GeneratorInitializationContext Context)
238:         {
239:             Context.RegisterForSyntaxNotifications(() => new UnrealTypeReceiver());
240:         }
```

同类文件中的第二个生成器同样如此（`:811-812` `public class LibraryBridgeGenerator : ISourceGenerator`）。

生成的三类文件都没有 `// <auto-generated>` 头（对比第二个生成器**有**）：
```csharp
 74:                 var source = "";                        // 接口包装类，无文件头
 ...
110:                 Context.AddSource("Script.CoreUObject." + @interface.Name + ".gen.cs", source);
...
203:                     Context.AddSource(type.Value.NameSpace + "." + type.Value.Name + ".gen.cs", source);   // UStruct
...
232:                     Context.AddSource(type.Value.NameSpace + "." + type.Value.Name + ".gen.cs", source);   // UClass/UInterface
```
而 `LibraryBridgeGenerator` 明确写了头（`:875`）与 `.g.cs` 后缀（`:868`）：
```csharp
875:             var source = "// <auto-generated/> LibraryBridgeGenerator -- do not edit.\n" +
...
868:                 Context.AddSource($"{pair.Key.ContainingNamespace?.ToDisplayString()}.{pair.Key.Name}.LibraryBridge.g.cs",
```

**调用上下文**
`UnrealTypeSourceGenerator.Execute` 里唯一的增量门槛是程序集名（`:62-65`）：
```csharp
 62:             if (Context.Compilation.AssemblyName != GameAssemblyName)
 63:             {
 64:                 return;
 65:             }
```
`GameAssemblyName` 由 `FSolutionGenerator::ReplaceGameName`（`FSolutionGenerator.cpp:292-296`）在生成工程时替换为 `GetGameName()`；实测 `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:18` = `"Game"`，`dotnet msbuild -getProperty:AssemblyName Script/Game/Game.csproj` = `Game` → 二者一致（这一处**正常**）。

**问题**
1. **无增量管道**：`ISourceGenerator` 每次编译都要重新遍历整个语法树（`RegisterForSyntaxNotifications` + `ISyntaxReceiver.OnVisitSyntaxNode`，`:263-392`），全项目任何一处编辑都会全量重跑。项目越大（本仓库 `Script/UE/Proxy` 有数万个生成文件）IDE 内编译越慢。这里**不存在**增量生成器的经典错误（捕获 `Compilation`/`ISymbol` 导致永不缓存）——因为根本没有增量管道可言；但这也意味着没有任何 `WithComparer`/`Equatable` 优化空间被利用。
   > 备注：把 `UnrealTypeSourceGenerator` 迁移到 `IIncrementalGenerator` 时，必须注意 `Execute` 里的 `Context.Compilation`（`:62`）只能出现在最终 `RegisterSourceOutput` 的 `context` 中（`IncrementalGeneratorOutputKind`），若把它塞进 `IncrementalValuesProvider` 的 `Select` 里就会破坏缓存 —— 这是迁移时的第一个坑。
2. **生成代码没有生成标记**：`UnrealTypeSourceGenerator` 产出的 `X.gen.cs` 既没有 `// <auto-generated>` 首行，后缀 `.gen` 也不在 Roslyn `GeneratedCodeUtilities` 识别的集合（`.designer` / `.generated` / `.g` / `.g.i`）内。因此这类文件在分析器（StyleCop、Roslynator、IDE 格式化）眼中与手写代码无异：会产生 `SA1633`/`IDE0055` 之类噪声；若用户工程开了 `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`，会被生成代码的警告直接编译失败。同一个文件里两个生成器的处理**不一致**（`:875` 有头、`.g.cs`），这本身就是可维护性问题。

**建议**
```csharp
// 在 UnrealTypeSourceGenerator 的三处 AddSource 前统一加头
private const string Header = "// <auto-generated/> UnrealTypeSourceGenerator -- do not edit.\n";
...
Context.AddSource($"{ns}.{name}.g.cs", Header + source);   // 后缀同时改为 Roslyn 识别的 .g.cs
```
并考虑把 hint name 从 `<Namespace>.<Name>.gen.cs` 改为 `<Namespace>.<Name>.g.cs`（`AddSource` 的 hint name 必须唯一，改后缀不影响唯一性）。

**验证方式**
1. 在 `Script/Game/Game.csproj` 加 `<EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>` 后 `dotnet build`，检查 `obj/Debug/net10.0/generated/SourceGenerator/SourceGenerator.UnrealTypeSourceGenerator/*.g.cs` 首行。
2. 在生成代码中故意触发一个分析器规则（如未使用 using），观察是否仍被报告。

---

### [F-CS4-009] 生成器的所有路径都没有 try/catch，异常会以 "source generator failed" 中断整个编译

- **类别**: Bug（错误处理）
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:55-235`（`UnrealTypeSourceGenerator.Execute`）、`:263-392`（`OnVisitSyntaxNode`）、`:823-871`（`LibraryBridgeGenerator.Execute`）
- **函数**: 同上
- **置信度**: 高（"无 try/catch"是代码事实；"CS8785"为该情形下 Roslyn 的标准表现）

**现状（代码事实）**
`Execute` 全流程（`:57-235`）与语法接收器（`:263-392`）没有任何 `try`。而代码里有若干**会在正常编译中抛异常**的写法：

```csharp
// 1) `?? throw new InvalidOperationException()` 直接写在诊断构造里（多处）
292:                     Errors.Add(Diagnostic.Create(UnrealTypeSourceGenerator.ErrorTypeNameNotMatch,
293:                         Location.Create(
294:                             enumDeclarationSyntax.Identifier.SyntaxTree ?? throw new InvalidOperationException(),
295:                             enumDeclarationSyntax.Identifier.Span),
296:                         $"The name of UEnum {name} must start with \"E\""));
```
同样写法出现在 `:306`、`:341`、`:357`、`:438`、`:451`、`:464`、`:480`、`:493`、`:520-521`。

```csharp
// 2) 未做空检查的订阅截断（接口名"必须以 I 开头"的校验与 Substring(1) 不在同一分支守卫下）
 86:                 source += $"\n\tpublic partial class U{@interface.Name.Substring(1)} : UInterface ";
```
```csharp
// 3) filePath 可能为 null（语法树无 FilePath 时），Path.GetFileName(null) 返回 null 而不是抛异常，
//    导致与 currentFileName 恒不相等 → 误报 UC_ERROR_02
302:                     if (Path.GetFileName(filePath) != currentFileName)
634:                     if (Path.GetFileName(filePath) != currentFileName)
```

**调用上下文**
`Execute` 由 Roslyn 在每次编译时调用；`OnVisitSyntaxNode` 由 `RegisterForSyntaxNotifications`（`:239`）在语法树遍历时回调，属于编译流水线的**不可失败**环节。生成器抛出的异常会被 Roslyn 包装成诊断 CS8785（"Generator 'X' failed to generate source"），并**中断该次编译**。

**问题**
对本插件而言后果尤其糟糕：源生成器只在**游戏程序集**（AssemblyName == `Game`）里生效（`:62`），而一个"用户动态类没写 partial / 文件名不匹配"的**普通用户错误**本来只应产生一条可读的 `UC_ERROR_0N`，现在却可能因为 `errorAttribute` 为 null（`:520-522` 直接用 `errorAttribute.SyntaxTree`）而抛 `NullReferenceException` → 用户看到的是 CS8785 而非"必须加 partial"。

具体地，`:499-526` 的路径：`Syntax.Modifiers` 不含 `partial` 且 `bIsUClass || bIsUStruct` 为真时，`errorAttribute = attributeUClass;`（`:509`）或 `attributeUStruct`（`:515`）。两者在 `:398-404` 通过 `GetAttributeFromClass` 获得，理论上与此处的 `bIsUClass/bIsUStruct` 同源，故当前不会为 null；但这是**靠两个布尔量隐式同步**维持的不变量，一旦后续有人在 `:528` 之前插入新的 `return` 或改动 `bIsUClass` 的赋值来源，就会变成 NRE。

**建议**
1. 在 `Execute` 与 `OnVisitSyntaxNode` 外层各加一层 `try/catch`，把异常降级为一条诊断（保持编译可继续、错误可定位）：
   ```csharp
   public void Execute(GeneratorExecutionContext Context)
   {
       try { ExecuteCore(Context); }
       catch (Exception Ex)
       {
           Context.ReportDiagnostic(Diagnostic.Create(
               UnrealTypeSourceGenerator.ErrorInternal,
               Location.None,
               Ex.GetType().Name + ": " + Ex.Message));
       }
   }
   ```
   并在 `AnalyzerReleases.Unshipped.md`/代码中登记该诊断（同时修掉 F-CS4-007 的登记问题）。
2. 去掉 `?? throw new InvalidOperationException()`：改用 `Location.Create(Syntax.SyntaxTree, Syntax.Identifier.Span)`（`SyntaxTree` 在 `SyntaxNode` 上不会为 null）。
3. `:52`、`:86`、`:632` 的 `Substring(1)` 前面加长度守卫，或先做命名校验再生成。

**验证方式**
用一个"标了 `[UClass]` 但没写 `partial`、且文件名与类名不一致"的类编译，观察是得到 `UC_ERROR_01` 还是 CS8785。

---

### [F-CS4-010] 特性类的继承关系有三套互不一致的写法，导致 C# 与 C++ 对 "IsOverride / UClass" 的判定可能分歧

- **类别**: Bug（一致性）/ 未定义行为
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Script/UE/Dynamic/Class/UClassAttribute.cs:7`、`Script/UE/Dynamic/Struct/UStructAttribute.cs:7`、`Script/UE/Dynamic/Enum/UEnumAttribute.cs:7`、`Script/UE/Dynamic/Function/UFunctionAttribute.cs:7`、`Script/UE/Dynamic/Property/UPropertyAttribute.cs:6`、`Script/UE/Dynamic/Interface/UInterfaceAttribute.cs:6`、`Script/UE/Dynamic/Class/KismetHideOverridesAttribute.cs:6`
- **函数**: `Utils.IsOverride(nint)`、`FClassReflection::IsOverride()`、`FDynamicGeneratorCore::SetFlags(FClassReflection*, UClass*)`
- **置信度**: 中（`KismetHideOverrides` 与 `IsDefined` 的派生匹配语义已从代码确认；"用户会不会真的只用 `[KismetHideOverrides]` 而不写 `[UClass]`"是推测）

**现状（代码事实）**

同一组"UE 类型标记"特性分成了三套基类：
```csharp
// A 组：继承 OverrideAttribute（4 个）
// Script/UE/Dynamic/Class/UClassAttribute.cs
  6:     [AttributeUsage(AttributeTargets.Class)]
  7:     public class UClassAttribute : OverrideAttribute
// Script/UE/Dynamic/Struct/UStructAttribute.cs:7   public class UStructAttribute : OverrideAttribute
// Script/UE/Dynamic/Enum/UEnumAttribute.cs:7       public class UEnumAttribute : OverrideAttribute
// Script/UE/Dynamic/Function/UFunctionAttribute.cs:7 public class UFunctionAttribute : OverrideAttribute
// （OverrideAttribute 定义在 Script/UE/CoreUObject/OverrideAttribute.cs:6，AttributeUsage = Class | Method）

// B 组：直接继承 Attribute（2 个）
// Script/UE/Dynamic/Property/UPropertyAttribute.cs
  5:     [AttributeUsage(AttributeTargets.Property)]
  6:     public class UPropertyAttribute : Attribute
// Script/UE/Dynamic/Interface/UInterfaceAttribute.cs
  5:     [AttributeUsage(AttributeTargets.Class | AttributeTargets.Interface)]
  6:     public class UInterfaceAttribute : Attribute

// C 组：继承 UClassAttribute（1 个，语义上等价于"顺带声明我是 UClass"）
// Script/UE/Dynamic/Class/KismetHideOverridesAttribute.cs
  5:     [AttributeUsage(AttributeTargets.Class)]
  6:     public class KismetHideOverridesAttribute : UClassAttribute
```

C# 侧用 `IsDefined`（**会匹配派生特性类型**）判定：
```csharp
// Script/UE/CoreUObject/Utils.cs:618-630
618:         [UnmanagedCallersOnly]
619:         public static int IsOverride(nint InTypeHandle)
621:             if (HandleData.GetObject(InTypeHandle) is Type Type)
623:                 return Type.IsDefined(typeof(UClassAttribute), false) ||
624:                        Type.IsDefined(typeof(OverrideAttribute), false)
625:                     ? 1
626:                     : 0;
```
C++ 侧用**精确类型相等**判定（`TSet<FClassReflection*>`，`FReflection.cpp:18-21`）：
```cpp
// FDynamicGeneratorCore.cpp:712-716
712: 	const auto AttributeClass = InClass->HasAnyClassFlags(CLASS_Interface)
713: 		                            ? FReflectionRegistry::Get().GetUInterfaceAttributeClass()
714: 		                            : FReflectionRegistry::Get().GetUClassAttributeClass();
716: 	if (InClassReflection->HasAttribute(AttributeClass))
```

**问题**
1. `KismetHideOverridesAttribute : UClassAttribute` 使"只写 `[KismetHideOverrides("...")]` 不写 `[UClass]`"时：
   - C# 侧 `IsOverride` 返回 **1**（`IsDefined(typeof(UClassAttribute), false)` 对派生类型返回 true）→ 框架认为这是一个 C# 动态 UClass；
   - C++ 侧 `HasAttribute(GetUClassAttributeClass())` 返回 **false**（描述符里只有 `KismetHideOverridesAttribute` 这个 `FClassReflection`，精确比较不命中）→ 不当作 UClass 注册。
   两端判定分歧，行为不可预测。另外 `[KismetHideOverrides]` 的 `AttributeUsage` 是 `Class`，与 UE 里它属于 `UCLASS(...)` 说明符的定位一致，因此**继承 `UClassAttribute` 是多余的**，且会带来上述副作用。
2. `UInterfaceAttribute : Attribute`（B 组）与 `UClassAttribute : OverrideAttribute`（A 组）不一致：C# 接口标 `[UInterface]` 时 `IsOverride` 返回 **0**（`Type.IsDefined(OverrideAttribute)` 为 false，`Type.IsDefined(UClassAttribute)` 也为 false），而类标 `[UClass]` 返回 1。若 `FClassReflection::IsOverride()`（`Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp:585-600`，内部调用 `ScriptDomain->IsOverride(ManagedClass)`）被用于接口路径，将与类路径行为不同。实测仓库里的用例（`Script/Game/UnrealCSharpTest/UnitTest/Dynamic/Core/TestDynamicInterface.cs:5` 用 `[UInterface, Blueprintable]`）确实只有 `[UInterface]`，因此这条不一致是**真实可达**的。
3. `UPropertyAttribute : Attribute` 本身无碍（属性不是 `Type`，`IsOverride` 不会被查询），但与 A 组风格不一，属于可读性债务。

**建议**
- 把 `KismetHideOverridesAttribute` 改为 `: Attribute`（去掉与 `UClassAttribute` 的继承关系），保持与其它说明符特性一致的"独立标记"语义；文档中明确"必须与 `[UClass]` 同时使用"。
- 让 `UInterfaceAttribute` 也继承 `OverrideAttribute`（或让 `IsOverride` 显式检查 `typeof(UInterfaceAttribute)`），使 C# 与 C++ 对"这是不是一个 C# 动态类型标记"的判定一致。
- 建议顺带梳理：`OverrideAttribute` 的语义是"覆写一个已存在的 UE 类型/函数"，而 `UClass/UStruct/UEnum/UFunction/UInterface` 的语义是"声明一个新动态类型"。当前把后者实现为前者的派生类，是"复用 `IsDefined` 派生匹配"的技巧，可读性差；可改为在 `IsOverride` 里显式列出这 5 个类型。

**验证方式**
```powershell
# 只写 [KismetHideOverrides] 的 C# 类，检查两端判定
# C#: Type.IsDefined(typeof(UClassAttribute), false)  → 期望 false（修正后）
# C++: InClassReflection->HasAttribute(GetUClassAttributeClass()) → false
```

---

### [F-CS4-011] 模板工程把插件路径硬编码为 `..\..\Plugins\UnrealCSharp\`，且 `ReplacePluginBaseDir` 只替换最后一级目录名、且未应用于 Interop.csproj

- **类别**: 平台兼容 / Bug
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度 P2 维持；可达性 潜伏
- **文件**: `Template/UE.csproj:4`、`Template/Interop.csproj:12`、`Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:78-95`、`:166-175`
- **函数**: `FSolutionGenerator::Generator()`、`FSolutionGenerator::ReplacePluginBaseDir(FString&)`
- **置信度**: 高（路径算术与替换范围均从代码直接可推）

**现状（代码事实）**

模板里两处都写死了 `Plugins\UnrealCSharp`：
```xml
Template/UE.csproj
  2:   <Import Project="../Shared.props" Condition="Exists('../Shared.props')" />
  4:     <Compile Include="..\..\Plugins\UnrealCSharp\Script\UE\**" Exclude="..\..\Plugins\UnrealCSharp\Script\UE\**\*.DS_Store"></Compile>
```
```xml
Template/Interop.csproj
  7:     <Compile Include="..\..\Plugins\UnrealCSharp\Script\Interop\**" Exclude="..\..\Plugins\UnrealCSharp\Script\Interop\**\*.DS_Store"></Compile>
```

替换函数只取插件目录的**最后一级名字**：
```cpp
166: void FSolutionGenerator::ReplacePluginBaseDir(FString& OutResult)
168: 	auto Index = 0;
170: 	const auto PluginBaseDir = FUnrealCSharpFunctionLibrary::GetPluginBaseDir();
172: 	PluginBaseDir.FindLastChar('/', Index);
174: 	OutResult = OutResult.Replace(*PLUGIN_NAME, *PluginBaseDir.Right(PluginBaseDir.Len() - Index - 1));
```
（`PLUGIN_NAME` = `FString(TEXT("UnrealCSharp"))`，`Source/UnrealCSharpCore/Public/CoreMacro/Macro.h:3`。）

而它只被挂给 UE.csproj：
```cpp
 88: 	CopyTemplate(
 89: 		FUnrealCSharpFunctionLibrary::GetUEProjectPath(),
 90: 		TemplatePath / DEFAULT_UE_NAME + PROJECT_SUFFIX,
 91: 		TArray<TFunction<void(FString& OutResult)>>
 92: 		{
 93: 			&FSolutionGenerator::ReplacePluginBaseDir,
 94: 			&FSolutionGenerator::AddProjectGeneratorHeaderComment
 95: 		});
...
 78: 	CopyTemplate(
 79: 		FUnrealCSharpFunctionLibrary::GetInteropProjectPath(),
 80: 		TemplatePath / INTEROP_NAME + PROJECT_SUFFIX,
 81: 		TArray<TFunction<void(FString& OutResult)>>
 82: 		{
 83: 			&FSolutionGenerator::ReplaceOutputPath,
 84: 			&FSolutionGenerator::ReplaceTargetFramework,
 85: 			&FSolutionGenerator::AddProjectGeneratorHeaderComment
 86: 		});
```
`Interop.csproj` 的分支里**没有** `ReplacePluginBaseDir`。

**调用上下文**
`FSolutionGenerator::Generator()` 在编辑器启动/代码生成时被调用（`SourceCodeGenerator` 模块），生成目标是 `<ProjectDir>/Script/{UE,Game,Interop,SourceGenerator,Weavers,CodeAnalysis}`。实测生成物 `Script/UE/UE.csproj:9` 与 `Script/Interop/Interop.csproj:12` 都保留了字面量 `..\..\Plugins\UnrealCSharp\Script\...`。

**问题**
1. **安装位置假设错误**：生成的 csproj 假定插件位于 `<Project>/Plugins/UnrealCSharp`。若插件通过 Marketplace/Fab 安装到 `<Engine>/Plugins/Marketplace/UnrealCSharp`，或装在 `<Engine>/Plugins/UnrealCSharp`，`..\..\Plugins\UnrealCSharp\...` 会解析到不存在的位置 → `Compile Include` 匹配 0 个文件，**UE 工程与 Interop 工程编译出的程序集为空**（或直接报错），而 Game 工程引用了它们 → 整个 C# 编译失败。
2. **重命名插件目录失效**：用户把 `Plugins/UnrealCSharp` 重命名为别的名字时，`ReplacePluginBaseDir` 能修正 `UE.csproj` 里的最后一级名字，却修不了 `Interop.csproj`（未挂该替换）→ 同一份解决方案里两个工程指向不同路径，其中一个必然失效。当前 `PLUGIN_NAME` 恰好等于目录名，所以只会在"引擎目录安装"和"重命名"两种场景暴露 —— 都是常见场景。
3. 同一个 `ReplacePluginBaseDir` 只替换目录名而不替换 `Plugins\` 前缀，说明作者只考虑了"项目内改名"，没有考虑"引擎内安装"。

**建议**
1. 把 `Template/{UE,Interop}.csproj` 的 `Compile Include` 改为基于属性、由生成期注入绝对或正确的相对路径，例如：
   ```xml
   <!-- 模板 -->
   <Compile Include="$(UnrealCSharpPluginDir)\Script\UE\**" Exclude="$(UnrealCSharpPluginDir)\Script\UE\**\*.DS_Store" />
   ```
   并在 `FSolutionGenerator` 中用 `FUnrealCSharpFunctionLibrary::GetPluginDirectory()` 通过新的 `ReplacePluginDir` 替换成**相对于生成工程文件的正确相对路径**（用 `FPaths::MakePathRelativeTo`）。这样"项目内/引擎内/改名"三种情形都能自洽。
2. 最低成本修法：给 `Interop.csproj` 的分支（`FSolutionGenerator.cpp:81-86`）补上 `&FSolutionGenerator::ReplacePluginBaseDir`，并把 `ReplacePluginBaseDir` 的替换目标从 `PLUGIN_NAME` 扩展为 `Plugins\UnrealCSharp` 这一整段（用 `GetPluginDirectory()` 相对 `<ProjectDir>` 计算）。

**验证方式**
1. 把插件临时放到 `<Engine>/Plugins/` 下重新生成，观察 `Script/UE/UE.csproj` 的 `Compile Include` 能否命中文件：
   `dotnet msbuild Script/UE/UE.csproj -getItem:Compile | Select-String 'Script.UE'`
2. `grep -c ReplacePluginBaseDir Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp` → 期望 ≥ 2（UE 与 Interop 各一处）。

---

### P3

### [F-CS4-012] `BlueprintAutoCast` 与 UE 的元数据键 `BlueprintAutocast` 仅大小写不同

- **类别**: 可读性 / 一致性
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Script/UE/Dynamic/Function/BlueprintAutoCastAttribute.cs:6`、`Source/UnrealCSharpCore/Public/CoreMacro/MetaDataAttributeMacro.h:289`
- **函数**: `FDynamicGeneratorCore::SetMetaData(FField*, const FString&, const FString&)`
- **置信度**: 高（差异确认；"仍然可用"的结论已用引擎源码证伪了"拼写错误即失效"的推论）

**现状（代码事实）**
```c
// Source/UnrealCSharpCore/Public/CoreMacro/MetaDataAttributeMacro.h
289: #define CLASS_BLUEPRINT_AUTO_CAST_ATTRIBUTE FString(TEXT("BlueprintAutoCastAttribute"))
```
```cpp
// Engine/Source/Runtime/CoreUObject/Public/UObject/ObjectMacros.h
1710: 		/// [FunctionMetadata] Used only by static BlueprintPure functions from BlueprintLibrary. A cast node will be automatically added ...
1711: 		BlueprintAutocast,
```
经全量比对，**255 个 C# 特性中只有这一个与 UE 官方键存在大小写差异**（pwsh 大小写不敏感 diff 的唯一输出）。

**调用上下文**
元数据以 `FName` 为键存取（`FMetaData` 内部 `TMap<FName, FString>`）。

**问题**
**这不是功能缺陷。** AE 的 `FName` 是"大小写不敏感但大小写保留"的：
```cpp
// Engine/Source/Runtime/Core/Public/UObject/NameTypes.h:573
573:  * Names are case-insensitive, but case-preserving (when WITH_CASE_PRESERVING_NAME is 1)
```
`FName::operator==` 比较的是 `ComparisonIndex`（已做大小写折叠），见 `NameTypes.h:478` / `:548`。因此 `meta=(BlueprintAutoCast="...")` 与 UE 读取 `BlueprintAutocast` 时会命中同一项，行为正确。

它的问题是**可读性/一致性**：源码搜索 `BlueprintAutocast` 在插件内 0 命中，新维护者无法通过 UE 文档反查该特性，容易误以为拼错。该特性定级 P3（原判 P1 被引擎源码证伪）—— 记录在此以免重复误判。

**建议**
把 C# 类、宏字符串、`MetaDataAttributeMacro.h` 中的键统一改为 `BlueprintAutocast`（与 UE 官方拼写一致），因为键名大小写不敏感，此改动**无运行时风险**。

**验证方式**
`grep -rn 'BlueprintAutoCast' Script Source` 修正后应为 0 命中。

---

### [F-CS4-013] `[HideAssetPicker]` 在 UE 5.6 已弃用（替代者 `HidePinAssetPicker`）；`[FixedIncrement]` UE 侧标注 Deprecated

- **类别**: 平台兼容（版本）/ 可优化
- **严重度**: **P3**
- **复核结论**: 确认 —— 严重度 P3 维持；可达性 活跃
- **可达性**: 活跃（`HideAssetPicker` 在函数元数据白名单 `FDynamicGeneratorCore.cpp:1221`、`FixedIncrement` 在属性元数据白名单 `:1147`，两键都会被投递）
- **复核证据**: **回引擎源码实证**：`Engine/Source/Runtime/CoreUObject/Public/UObject/ObjectMacros.h:1599` = `HideAssetPicker UE_DEPRECATED(5.6, "Use HidePinAssetPicker instead."),`，替代者 `:1602` = `HidePinAssetPicker,` —— 报告的"弃用于 5.6、替代者名为 `HidePinAssetPicker`"两点**完全成立**；另有独立佐证 `Engine/Source/Editor/BlueprintGraph/Classes/EdGraphSchema_K2.h:197-202` = `UE_DEPRECATED(5.6, "Use MD_HidePinAssetPicker instead.")` + `MD_HideAssetPicker` / `MD_HidePinAssetPicker`。`FixedIncrement` 在 `ObjectMacros.h` 中唯一命中 **:1369**（属性元数据段），报告引用的 `:1368-1369` 中标识符行 ±0 吻合（1368 的 `Deprecated.` 注释行未逐字重读）
- **级别变动**: 无（维持 P3：两键在 5.6 下仍被 UE 登记/接受，不构成本版本功能缺陷）
- **文件**: `Script/UE/Dynamic/Function/HideAssetPickerAttribute.cs:6`、`Script/UE/Dynamic/Property/FixedIncrementAttribute.cs:6`、`Source/UnrealCSharpCore/Public/CoreMacro/MetaDataAttributeMacro.h:237,119`
- **函数**: `FDynamicGeneratorCore::SetMetaData(UFunction*, FReflection*)`（`HideAssetPicker` 在函数元数据列表 `:1221`）、`SetMetaData(FProperty*, FReflection*)`（`FixedIncrement` 在属性元数据列表 `:1147`）
- **置信度**: 高

**现状（代码事实）**
```cpp
// Engine/Source/Runtime/CoreUObject/Public/UObject/ObjectMacros.h
1598: 		/// [FunctionMetadata] ...
1599: 		HideAssetPicker UE_DEPRECATED(5.6, "Use HidePinAssetPicker instead."),
1600: 
1601: 		/// [FunctionMetadata] Forcibly hide the asset picker for pins matching any parameter names in this piece of metadata.
1602: 		HidePinAssetPicker,
```
```cpp
// 同文件
1368: 		/// [PropertyMetadata] Deprecated.
1369: 		FixedIncrement,
```
插件侧仍按旧名投递：
```c
MetaDataAttributeMacro.h:237: #define CLASS_HIDE_ASSET_PICKER_ATTRIBUTE FString(TEXT("HideAssetPickerAttribute"))
MetaDataAttributeMacro.h:119: #define CLASS_FIXED_INCREMENT_ATTRIBUTE FString(TEXT("FixedIncrementAttribute"))
```
`HideAssetPicker` 虽仍被登记（作为弃用别名），但新 API 是 `HidePinAssetPicker`；`FixedIncrement` 的简介直接写着 "Deprecated."。

**问题**
对 UE 5.6 项目（本仓库 EngineAssociation 指向 `<Engine 5.6 安装目录>`，见 `UnrealCSharpTest.uproject:3` 与注册表 Builds 项 `{79A5A541-...} = <Engine 5.6 安装目录>`），这两个特性走的是弃用路径。`HideAssetPicker` 若在后续引擎版本被移除，插件的元数据将变成未知键（UE 对未知元数据一般不报错，只会失效），届时难以定位。这是"跨版本兼容性"的轻度风险，不构成本版本的功能缺陷。

**建议**
- 新增 `[HidePinAssetPicker]` 特性并保留 `[HideAssetPicker]` 作为别名（同时投递两个键，或按 `UE_5_6_OR_LATER` 宏二选一），因为插件已有 `UE_5_x_OR_LATER` 定义常量机制（`FSolutionGenerator::ReplaceDefineConstants`，`FSolutionGenerator.cpp:190-224` 生成 `UE_5_0_OR_LATER;...`）。注意 `DefineConstants` 是给 **C#** 用的，C++ 侧的版本判断应使用 `Source/CrossVersion` 里的 `UE_5_6_OR_LATER`。
- `[FixedIncrement]` 建议在文档/注释中标明已弃用，或直接移除。

**验证方式**
在 UE 5.6 编辑器中对标注 `[HideAssetPicker("X")]` 的函数查看 `UFunction` 元数据，确认键为 `HideAssetPicker`；搜索引擎源码确认无其他读取点：`grep -rn 'HidePinAssetPicker' Engine/Source/Editor | head`。

---

### [F-CS4-014] 全部 255 个特性类都是非 `sealed`、无 `AllowMultiple` / `Inherited` 显式声明

- **类别**: 可优化/可读性
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Script/UE/Dynamic/**/*.cs`（255 个文件的 `[AttributeUsage(...)]` 行，如 `Script/UE/Dynamic/Property/EditAnywhereAttribute.cs:5`、`Script/UE/Dynamic/Functional/…` 各处；类声明行如 `Script/UE/Dynamic/Generic/CategoryAttribute.cs:5-6`）
- **函数**: 不适用
- **置信度**: 高

**现状（代码事实）**
全目录检索：
```
grep 'AllowMultiple|Inherited\s*=|sealed|Conditional' Script/UE/Dynamic/**/*.cs → No matches found
```
即：
- 没有一个是 `sealed class`；
- 没有任何一个 `[AttributeUsage]` 显式写了 `AllowMultiple` 或 `Inherited`，全部依赖默认值（`AllowMultiple = false`、`Inherited = true`）；
- 没有 `[Conditional]`。

255 个 `AttributeUsage` 的 `ValidOn` 分布（机器核对结果）：

| ValidOn | 个数 | 举例 | 与 UE 语义是否匹配 |
|---|---|---|---|
| `AttributeTargets.Property` | 113 + 部分 Generic | `EditAnywhere`、`BlueprintReadWrite`、`ClampMin` | 匹配 |
| `AttributeTargets.Method` | 62 + 部分 Generic | `BlueprintPure`、`Exec`、`WorldContext` | 匹配 |
| `AttributeTargets.Class` | 42 + 部分 Generic | `HideCategories`、`Within`、`MinimalAPI` | 匹配 |
| `AttributeTargets.Enum` | 3 + 部分 Generic | `Bitflags`、`UseEnumValuesAsMaskValuesInEditor` | 匹配 |
| `AttributeTargets.Interface` | 2 + 部分 Generic | `UInterface`、`CannotImplementInterfaceInBlueprint` | 匹配 |
| `Method \| Property` | 10（`Category`/`BlueprintCallable`/`BlueprintGetter`/`BlueprintSetter`/`BlueprintAuthorityOnly`/`FieldNotify` 等） | | 匹配（UE 中 `Category` 同时是 UPROPERTY/UFUNCTION 说明符） |
| `Class \| Enum \| Interface` | 2（`Blueprintable`/`BlueprintType`） | | 匹配 |
| `Class \| Interface` | 3（`UInterface`/`ConversionRoot`/`MinimalAPI`/`IsBlueprintBase`/`CannotImplementInterfaceInBlueprint`） | | 匹配 |

**结论：未发现 `ValidOn` 写错的条目**（这是本报告一项重要的**否定性结论** —— 任务假设"`ValidOn` 写错会导致特性悄悄不能用"，实测 255 个 `ValidOn` 全部与 C++ 消费端一致，其中最容易被写错的 `Category`（既可用于属性也可用于函数）已在 `Generic/CategoryAttribute.cs:5` 正确写成 `Method | Property`）。

**问题**
1. 非 `sealed`：用户可以从 `EditAnywhereAttribute` 派生自定义特性。由于 C# 侧 `Utils.cs` 是按 **命名空间** 收集的（`:175` `CustomAttribute.AttributeType.Namespace == "Script.Dynamic"`），派生类若放在用户命名空间不会被收集（静默失效）；若放在 `Script.Dynamic`（用户不该这么做）则会被收集，但 `AttributeType` 是派生类型，C++ 侧精确匹配 `FClassReflection` 又会失配。也就是说"派生特性"这条路在当前实现下**必然静默失效**，`sealed` 本可把这个问题变成编译期错误。
2. `AllowMultiple = false`（默认）：`[HidePin("A")] [HidePin("B")]` 无法编译。对 UE 而言这类元数据的值本就是逗号分隔列表（`ObjectMacros.h:1590` 的 `ArrayParm`、`:1681` 的 `HidePin`、`:1597` 的 `AutoCreateRefTerm` 等），单次声明写逗号即可，因此**实际影响有限**，但值得在文档中说明。
3. `Inherited = true`（默认）：对 `AttributeTargets.Class` 的特性，若用 `GetCustomAttributes(inherit: true)` 会沿继承链收集。当前两处描述符实现用的是 `MemberInfo.CustomAttributes`（**不**沿继承链，`Utils.cs:137`/`:173`），所以默认值未造成问题；但这是一处隐含依赖。

**建议**
给全部特性类加 `sealed`（可用一次性脚本/正则批量改），把"派生特性无效"从运行期静默变成编译期报错。若确实希望支持派生（例如让用户写 `[MyGameCategory("X")]` 作为 `Category` 的别名），则需要把 C++ 侧的精确匹配改为"沿基类链查找"，而不能只加 `sealed`。

**验证方式**
```powershell
Select-String Script/UE/Dynamic -Recurse -Filter *.cs -Pattern 'sealed class' | Measure-Object
# 期望（修正后）：255
```

---

### [F-CS4-015] `LibraryBridgeGenerator` 用"方法名"生成桥接槽字段与桥接键，同名重载会生成重复字段

- **类别**: Bug（潜在）
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:903-918`
- **函数**: `LibraryBridgeGenerator.EmitBridges(INamedTypeSymbol, List<IMethodSymbol>)`
- **置信度**: 中（缺陷机制确定；当前代码库无重载，属潜在而非已触发）

**现状（代码事实）**
```csharp
903:                 var key = $"{containingNamespace}.{InOwner.Name}::{method.Name}";
905:                 var slot = $"{method.Name}_Slot";
...
909:                 source +=
910:                     "#if WITH_LEANCLR\n" +
911:                     $"\t\t[DllImport(\"{NativeModuleName}\", CallingConvention = CallingConvention.Cdecl)]\n" +
912:                     $"\t\t{accessibility} static extern unsafe partial {returnType} {method.Name}({parameters});\n" +
913:                     "#else\n" +
914:                     $"\t\tprivate static nint {slot};\n" +
915:                     "\n" +
916:                     $"\t\t{accessibility} static unsafe partial {returnType} {method.Name}({parameters}) =>\n" +
917:                     $"\t\t\t((delegate* unmanaged[Cdecl]<{pointerType}>)global::Interop.MethodBridge.GetMethod(\n" +
918:                     $"\t\t\t\tref {slot}, \"{key}\"))({arguments});\n" +
919:                     "#endif\n\n";
```

**调用上下文**
接收器 `LibraryBridgeReceiver.OnVisitSyntaxNode`（`:942-965`）收集 `Script.Binding` / `Script.Library` 命名空间下"带 `static` + `partial` 且无 body"的方法声明；`EmitBridges` 为每个这样的方法生成一个 `_Slot` 静态字段 + 一个 `partial` 实现。

**问题**
`slot` 与 `key` 都只用 `method.Name`（不含参数签名）。若某类型有两个同名重载（例如 `Foo(int)` 与 `Foo(string)`），会输出**两行** `private static nint Foo_Slot;` → CS0102（"类型已包含 'Foo_Slot' 的定义"），整个编译失败；同时 `key` 也会重复，运行时 `MethodBridge.GetMethod` 无法区分两个重载。

实测当前仓库**不存在**这种情况（对 `Script/UE/Library`、`Script/UE/Binding`、`Script/Interop` 提取 211 个 `partial` 声明，重名 0 个；所有桥接方法都以 `__` 前缀命名，如 `Script/UE/Library/ObjectImplementation.cs:9` `private static unsafe partial byte __UObject_IdenticalImplementation(...)`）。因此这是**潜在缺陷**，一旦有人给某个 Library 类加重载就会突然暴露。

**建议**
用 `method.ToDisplayString(SymbolDisplayFormat.FullyQualifiedFormat)` 或 `method.Name + "_" + 参数类型哈希` 参与 `slot` 与 `key` 的构造：
```csharp
var signature = $"{method.Name}_{string.Join("_", method.Parameters.Select(p => p.Type.ToDisplayString()))}";
var slot = $"{Sanitize(signature)}_Slot";
var key  = $"{containingNamespace}.{InOwner.Name}::{Sanitize(signature)}";
```
注意 `slot` 必须是合法标识符，需做字符清洗。

**验证方式**
在 `Script/UE/Library/` 下临时给某个 partial 类加两个同名重载的 `partial` 声明，`dotnet build` 应复现 CS0102；修正后应通过。

---

### [F-CS4-016] `IsUnique` 只按简单类型名去重，不同命名空间下的同名类会误报 `UC_ERROR_05`

- **类别**: Bug
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:660-674`、`:261`
- **函数**: `UnrealTypeReceiver.IsUnique(BaseTypeDeclarationSyntax, string)`
- **置信度**: 高

**现状（代码事实）**
```csharp
255:         public readonly Dictionary<string, TypeInfo> TypeInfos = new Dictionary<string, TypeInfo>();
261:         public HashSet<string> Types = new HashSet<string>();
...
660:         private bool IsUnique(BaseTypeDeclarationSyntax Syntax, string Name)
661:         {
662:             if (Types.Add(Name))
663:             {
664:                 return true;
665:             }
666: 
667:             Errors.Add(Diagnostic.Create(UnrealTypeSourceGenerator.ErrorTypeMustBeUnique,
668:                 Location.Create(
669:                     Syntax.SyntaxTree,
670:                     Syntax.Span),
671:                 $"{Name} must be unique"));
673:             return false;
674:         }
```
`IsUnique` 在枚举（`:285`）、接口（`:328`）、类（`:414`）三处被调用，传入的都是**简单名**（`enumDeclarationSyntax.Identifier.ToString()` / `interfaceDeclarationSyntax.Identifier.ToString()` / `Syntax.Identifier`）。而真正用于缓存 TypeInfo 的键是**全限定名**（`:591` `TypeInfos.TryGetValue(nameSpace + "." + name, ...)`）。

**调用上下文**
`OnVisitSyntaxNode`（`:263`）在每次编译中遍历所有语法节点；`IsUnique` 的返回值决定是否 `return`（跳过该类型）。

**问题**
`Types` 是简单名的 `HashSet`，因此 `Script.Game.A.Foo` 与 `Script.Game.B.Foo` 会被判为"重名"，报出 `UC_ERROR_05 Type must be unique` 并**跳过第二个类型**（不生成 `StaticClass()` 等）。这两个类在 UE 里也是不同命名的 `UClass`（UE 类名全局唯一，但 C# 命名空间不同意味着 UE 名相同 —— 严格说 UE 确实不允许同名 UClass）。所以这个限制**在 UE 语义下部分合理**：UE 的 `StaticClass()` 走 `/Script/CoreUObject.<Name>` 路径（`:50-53` `GetPathName`），同名确实会冲突。

因此我把它定位为 P3（提示性的约束，而非缺陷）：它把"UE 类名必须全局唯一"这一隐含约束实现成了"C# 简单名必须全局唯一"，且错误信息（"Foo must be unique"）没有说明是"UE 类名"层面的唯一性，用户看到会困惑。另外注意：`Types` 的键**不区分** `I`/`A`/`U`/`F` 前缀剥除后的名字，也不区分 `_C` 后缀（`:630-632`），因此 `Foo`、`AFoo`、`UFoo`、`IFoo`、`Foo_C` 会被视为 5 个不同的键（这是合理的），但 `AFoo` 与 `Foo_C` 映射到同一个 UE 类名 `Foo` 时**不会**被检出（漏报）。

**建议**
- 把错误信息改为可操作的措辞，例如 `$"UClass/UStruct name '{Name}' conflicts with another dynamic type; UE requires globally unique UClass names (offending file: {filePath})"`。
- 若想更准确，应按 UE 类名（剥除 `A`/`U`/`F` 前缀与 `_C` 后缀后的名字）去重 —— 但这会改变现有行为，需要评估仓库内是否已有依赖。

**验证方式**
在同一次编译中加入 `namespace N1 { [UStruct] public partial struct FFoo {} }` 与 `namespace N2 { [UStruct] public partial struct FFoo {} }`，观察是否报 `UC_ERROR_05`。

---

### [F-CS4-017] `Template/Override/*.cs` 保留显式无参构造函数，与仓库内实际使用的同类文件风格不一致

- **类别**: 可优化/可读性
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Template/Override/Actor.cs:9-11`、`Template/Override/ActorComponent.cs:9-11`、`Template/Override/Object.cs:8-10`、`Template/Override/UserWidget.cs:8-10`
- **函数**: 不适用（编辑器工具栏动作 `UnrealCSharpBlueprintToolBar::OverrideBlueprint`）
- **置信度**: 中（"模板会被写成这样"已由替换逻辑确认；"是否真的会 CS0111"取决于目标类是否已有无参构造，未在本机复现）

**现状（代码事实）**
```csharp
// Template/Override/Actor.cs
  4: namespace Script.Engine
  5: {
  6:     [Override]
  7:     public partial class AActor
  8:     {
  9:         public AActor()
 10:         {
 11:         }
 12: 
 13:         [Override]
 14:         public override void ReceiveBeginPlay()
 15:         {
 16:             base.ReceiveBeginPlay();
 17:         }
```

替换逻辑（`Source/UnrealCSharpEditor/Private/ToolBar/UnrealCSharpBlueprintToolBar.cpp:99-137`）会把 `namespace Script.Engine` 换成目标蓝图所在命名空间、把 `" AActor"` 换成目标 `_C` 类名，于是生成物形如：
```csharp
namespace <bp ns> { [Override] public partial class ABP_Foo_C { public ABP_Foo_C() {} [Override] public override void ReceiveBeginPlay() {...} } }
```
而仓库里真实存在的同类文件**没有构造函数**：
```csharp
// Script/Game/UnrealCSharpTest/UnitTest/Reflection/BlueprintCSharpFunction/BP_TestCSharpFunctionActor_C.cs
  4: namespace Script.Game.UnitTest.Reflection
  5: {
  6:     [Override]
  7:     public partial class BP_TestCSharpFunctionActor_C
  8:     {
  9:         [Override]
 10:         public void SetBoolValueFunction(bool InBoolValue = false)
```
（全文 373 行，无构造函数。）

另一处差异：模板里 `ReceiveBeginPlay` 写的是 `public override void`（因为模板的类是 `Script.Engine.AActor` 的 partial 扩展，`ReceiveBeginPlay` 在 UE 生成的代理类里是 `virtual`，见 `Script/UE/Proxy/Engine/Classes/GameFramework/Actor.cs:3553` `public virtual void ReceiveBeginPlay()`），替换成 `ABP_Foo_C` 后 `override` 依然合法（目标类继承自 `AActor`）—— 这一处是**正确的**。

**问题**
`public ABP_Foo_C() { }` 是否有害取决于目标 partial 类的其余部分是否已有显式无参构造。若用户的目标 `_C` 类（或其生成的另一半）已经声明了无参构造，就会出现 CS0111（"类型已定义了一个名为 … 的成员"）。更现实的问题是一致性：模板生成的文件与插件自带的示例文件风格不同，且模板里的空构造函数没有任何作用（UE 侧对象由引擎创建，C# 侧的 ctor 只是被调用）。此外 `Template/Override/Object.cs:8-10` 的 `public UObject()` 更奇怪 —— 给 `UObject` 这样的基类加公开无参构造。

**建议**
删除四个 `Override` 模板里的构造函数块（`Actor.cs:9-12` 的含空行、`ActorComponent.cs:9-12`、`Object.cs:8-11`、`UserWidget.cs:8-11`），与仓库内的实际用法对齐。如果确实需要构造函数（例如为了插入 UE 对象创建时的初始化），应在模板里加注释说明其必要性。

**验证方式**
用编辑器的 "Override Blueprint" 工具栏动作对一个已有显式构造函数的蓝图生成文件，`dotnet build` 观察是否 CS0111。

---

### [F-CS4-018] VS2026 下生成的 `Script.sln` 丢失 "DO NOT modify this manually" 头注释（同一轮生成中 csproj 有、sln 没有）

- **类别**: 可读性 / 一致性
- **严重度**: **P3**
- **复核结论**: 部分确认（偏差：**触发条件写窄了**。该结论是"**MSVC 下（任何 VS 版本）全部丢失**"，而非"VS2026 下丢失"；机制不是"VS2026 被定义"，而是 `VS2026` 宏**未加括号**导致 `#if !VS2026` 求值恒假）
- **可达性**: 活跃（Windows + MSVC 构建即触发；本仓库生成物 `Script/Script.sln` 就是 MSVC 下生成的实例）
- **复核证据**: grep `VS2026|DO NOT modify` on `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp` → `:434`（csproj 头，**在 `#if` 之外、无条件写入**）、`:445`（`#if !VS2026`）、`:456`（`#endif`）、`:449`（sln 头，**在条件块内**）。关键补充证据：`Source/CrossVersion/Public/VSVersion.h:6` = `#define VS2026 _MSC_VER >= 1950`（**无括号的表达式宏**）、`:10` = `#define VS2026 0`（非 MSVC 平台）。于是 `#if !VS2026` 展开为 `#if !_MSC_VER >= 1950`，按运算符优先级解析为 `(!_MSC_VER) >= 1950` → MSVC 下 `_MSC_VER` 非零 ⇒ `!` 得 0 ⇒ `0 >= 1950` 恒假 ⇒ **头注释在 MSVC 下（VS2022/VS2026 都一样）被静默丢弃**；非 MSVC 平台 `VS2026` 展开为字面 `0` ⇒ `!0` ⇒ 真 ⇒ 头注释反而会被写入。跨报告一致性：与 `08-…/01` F-DEAD-001（"`#define VS2026` 未加括号，`#if !VS2026` 在 MSVC 下求值反转"）**机制完全一致**，与 `05-…/01` F-CMP-029（"`.sln` 注释头在非 MSVC 平台也会被写入"）一致；本报告此前的表述是三者中唯一把触发条件写窄的一处，已就地修正
- **级别变动**: 无（P3 维持；仍是可读性/一致性问题，非功能缺陷）
- **文件**: `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:429-457`、`Source/CrossVersion/Public/VSVersion.h:6,10`、`Template/Script.sln:1`、`Script/Script.sln:1-2`
- **函数**: `FSolutionGenerator::AddProjectGeneratorHeaderComment(FString&)`、`FSolutionGenerator::AddSolutionGeneratorHeaderComment(FString&)`
- **置信度**: 高（生成物时间戳同轮 + `#if` 条件推出）

**现状（代码事实）**
```cpp
429: void FSolutionGenerator::AddProjectGeneratorHeaderComment(FString& OutResult)
430: {
431: 	OutResult = FString::Printf(TEXT(
432: 		"<!-- ===========================================================================\r\n"
433: 		"    Generated code exported from UnrealCSharp.\r\n"
434: 		"    DO NOT modify this manually!\r\n"
435: 		"============================================================================ -->\r\n"
436: 		"\r\n"
437: 		"%s"
438: 	),
439: 	                            *OutResult
440: 	);
441: }
442: 
443: void FSolutionGenerator::AddSolutionGeneratorHeaderComment(FString& OutResult)
444: {
445: #if !VS2026
446: 	OutResult = FString::Printf(TEXT(
447: 		"# ===========================================================================\r\n"
448: 		"#    Generated code exported from UnrealCSharp.\r\n"
449: 		"#    DO NOT modify this manually!\r\n"
450: 		"# ===========================================================================\r\n"
451: 		"\r\n"
452: 		"%s"
453: 	),
454: 	                            *OutResult
455: 	);
456: #endif
457: }
```
`#if !VS2026`（注意是**否定**）意味着：当构建时定义了 `VS2026` 时，`Script.sln` **不加**头注释。而 `AddSolutionGeneratorHeaderComment` 仍然被挂给了 .sln（`:135`）。

实际生成物证实了这个分支被走了：
```
Script/Script.sln                 LastWriteTime 9/3/2026 12:58:08 PM   ← 同轮生成
Script/UE/UE.csproj               LastWriteTime 9/3/2026 12:58:08 PM   ← 同轮生成，有 <!-- ... DO NOT modify ... --> 头
Script/Game/Game.props            9/3/2026 12:58:08 PM                 ← 有 <!-- 头
Script/SourceGenerator/SourceGenerator.csproj  9/3/2026 12:58:08 PM    ← 有 <!-- 头
Script/Script.sln:1 (空行) / :2 Microsoft Visual Studio Solution File, Format Version 12.00   ← 无 # 头
```
`Template/Script.sln:1` 是空行，正是头注释应该插入的位置。

**调用上下文**
`FSolutionGenerator::Generator()` 在生成解决方案时调用链：`:126-136`（Dest/Src + 4 个替换函数），其中 `:135` 是 `AddSolutionGeneratorHeaderComment`。

**问题**
两条生成路径的头注释策略不一致：csproj/props 无条件加 `<!-- DO NOT modify -->`，而 .sln 在 VS2026 构建下静默不加。用户/协作者打开 VS 时，`.sln` 看起来像手写文件，可能被手工修改后在下次代码生成时被覆盖。

**建议**
把 `#if !VS2026` 去掉（或改为在两种情况下都写注释）。.sln 支持 `#` 行注释，写头注释没有兼容性风险；若当初的意图是"VS2026 的 sln 解析器不接受首行注释"，则应改用"把注释放在第 2 行之后"的方式，而不是直接省略。这一点建议由熟悉 `VS2026` 宏的维护者确认（`Source/CrossVersion` 里应有其定义）。

**验证方式**
```powershell
# 构建时定义 VS2026，重新生成解决方案后检查首行
Get-Content Script/Script.sln -TotalCount 3
```

---

### [F-CS4-019] `Template/Dynamic/*.cs` 把用户动态类放在 `namespace Script.CoreUObject`，作为"示例"具有误导性

- **类别**: 可读性 / 一致性
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Template/Dynamic/DynamicActor.cs:5`、`DynamicActorComponent.cs:5`、`DynamicObject.cs:4`、`DynamicUserWidget.cs:5`
- **函数**: 不适用（`FDynamicNewClassUtils::GetTemplateContent`）
- **置信度**: 高

**现状（代码事实）**
四个模板都声明在框架命名空间里：
```csharp
// Template/Dynamic/DynamicActor.cs
  1: using Script.Dynamic;
  2: using Script.Engine;
  3: using Script.CoreUObject;
  4: 
  5: namespace Script.CoreUObject
  6: {
  7:     [UClass]
  8:     public partial class ADynamicActor : AActor
```
生成期会改写命名空间（`Source/UnrealCSharpEditor/Private/NewClass/DynamicNewClassUtils.cpp:106` `AddNamespace(ParentNamespace, OutContent)`），所以**运行时结果是对的**；实测仓库中的游戏类都在自己的命名空间下（如 `Script/Game/UnrealCSharpTest/UnitTest/Dynamic/RawDynamicProperty/TestRawDynamicPropertyActor.cs:5` `namespace Script.CoreUObject` —— 注意这个案例确实放在 `Script.CoreUObject`，而 `Script/Game/UnrealCSharpTest/UnitTest/Reflection/...` 下的类放在 `Script.Game.UnitTest.Reflection`，说明两种做法都在被使用）。

**问题**
- 模板里的 `using Script.CoreUObject;` + `namespace Script.CoreUObject` 组合是冗余的（同命名空间不需要 using），作为样板会把这个冗余传播给用户。
- `namespace Script.CoreUObject` 是**插件自身生成的代理类**所在命名空间（`Script/UE/Proxy/CoreUObject/...`）。把用户代码放进框架命名空间（虽然发生在游戏程序集里，类型不冲突）会让 `Script.CoreUObject` 成为"框架 + 用户"混合命名空间，长期看会与未来版本新增的框架类型撞名 —— 而 `UnrealTypeSourceGenerator` 的 `IsUnique`（`F-CS4-016`）只按简单名判重，撞名后会直接报 `UC_ERROR_05` 并跳过生成。
- 四个模板的 `using` 顺序不一致：`DynamicActor.cs:1-3` 是 `Script.Dynamic` / `Script.Engine` / `Script.CoreUObject`，`DynamicObject.cs:1-2` 是 `Script.Dynamic` / `Script.CoreUObject`，`Override/*.cs` 又是另一种顺序。

**建议**
把模板命名空间改为占位符形式（如 `namespace Script.Game`）或明确注释"该命名空间会被新建类向导改写"。同时删除同命名空间的冗余 `using`，统一 `using` 顺序。

**验证方式**
不需要运行时验证；`grep -n 'namespace' Template/Dynamic/*.cs Template/Override/*.cs` 可直接复核。

---

### [F-CS4-020] Fody 与 FodyHelpers 版本不一致（6.9.3 vs 6.8.0）

- **类别**: 可优化/一致性
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Template/Game.props:9`、`Script/Weavers/Weavers.csproj:18`（后者由插件内同名文件生成，见 `FSolutionGenerator.cpp:55-61`）
- **函数**: 不适用
- **置信度**: 中（版本不匹配本身是事实；是否引发运行期问题未验证）

**现状（代码事实）**
```xml
Template/Game.props
  4:         <PackageReference Include="Fody" Version="6.9.3">
 10:         <WeaverFiles Include="$(ProjectDir)..\Weavers\bin\$(Configuration)\netstandard2.0\Weavers.dll" WeaverClassNames="UnrealTypeWeaver" />
```
```xml
Script/Weavers/Weavers.csproj
 18:     <PackageReference Include="FodyHelpers" Version="6.8.0" />
```
实测生成物 `Script/Game/Game.props:9` 为 `Fody 6.9.3`、`Script/Weavers/Weavers.csproj:18` 为 `FodyHelpers 6.8.0`，两者在同一个解决方案里共存。

**调用上下文**
Fody 在 `Game` 工程的构建后阶段加载 `Weavers.dll`（`<WeaverFiles>`），而 `Weavers.dll` 是针对 `FodyHelpers 6.8.0` 编译的。Fody 6.9.x 与 6.8.x 的 `FodyHelpers` 属同一大版本，通常向后兼容，故判为 P3。

**问题**
版本漂移会让"编织器加载失败"这类问题难以归因（`Weavers.dll` 加载失败时 Fody 只在构建输出里给一行信息）。另外 `Game.props:10` 的 `netstandard2.0` 与 `Weavers.csproj:9` 的 `<TargetFramework>netstandard2.0</TargetFramework>` 是一致的（这一处**正常**，虽然写成硬编码，但如果以后把 Weavers 改成多目标框架就会断）。

**建议**
把 `Game.props` 的 Fody 版本与 `Weavers.csproj` 的 FodyHelpers 版本对齐（都用 `6.9.3` 或都用 `6.8.0`），或把版本抽到 `Shared.props` 的一个属性里（`<FodyVersion>`）统一引用。`Game.props:10` 的 `netstandard2.0` 也可抽成属性，与 `Weavers.csproj` 共享。

**验证方式**
`Select-String Script -Recurse -Include *.props,*.csproj -Pattern 'Fody'` 对比版本号。

---

## 4. 死代码清单 / 未消费特性清单

### 4.1 "定义了但 C++ 侧没有任何宏/字符串引用"的特性（grep 证据）

度量方法：全插件（`Source/` 7 个模块 + `Script/` + `Template/`，869 个文件，排除 `ThirdParty/`、`obj/`、`bin/`）搜索特性类名；并单独检查 5 个 `*AttributeMacro.h` 的宏表与 `FReflectionRegistry.{h,cpp}`。

| 符号（C# 特性类） | 声明位置 | grep 命中数 | 判定 | 证据 |
|---|---|---|---|---|
| `CustomThunkTemplatesAttribute` | `Script/UE/Dynamic/Class/CustomThunkTemplatesAttribute.cs:6` | **2**（仅类声明 + 文件名/本文件内引用） | **纯装饰（死代码）** | 宏表中无对应宏（254 个宏无此项）；`FReflectionRegistry` 无 getter/注册；`grep 'CUSTOM_THUNK_TEMPLATES' Source → No matches`；UE 5.6 的 UHT 源码中也无 `CustomThunkTemplates`（不是 UE 说明符） |
| `DontAutoCollapseCategoriesAttribute` | `Script/UE/Dynamic/Class/DontAutoCollapseCategoriesAttribute.cs:6` | **2** | **纯装饰（死代码）** | 宏表中无对应宏；`FReflectionRegistry` 无 getter；`grep 'DONT_AUTO_COLLAPSE' Source → No matches`。注意它在 UE 侧**是真实说明符**（`EpicGames.UHT/Specifiers/UhtClassSpecifiers.cs:293-298`），属"未实现的合法特性" |

### 4.2 宏与注册表"双向完整"（正向结论，配合上表）

| 检查 | 结果 |
|---|---|
| C# 特性类总数 | 255 |
| 5 个 `*AttributeMacro.h` 中的宏字符串数 | 254 |
| 宏字符串去掉 `Attribute` 后与 C# 类名求差 `A\B`（C# 有、宏没有） | **2**：`CustomThunkTemplates`、`DontAutoCollapseCategories` |
| 求差 `B\A`（宏有、C# 没有） | **1**：`Override`（定义在 `Script/UE/CoreUObject/OverrideAttribute.cs:6`，不在 `Dynamic/` 下，属正常） |
| `FReflectionRegistry.cpp` 引用的特性宏 | **254 / 254**（全部宏都被注册，无遗漏） |
| 宏名重复 | 0 |

**结论**：除表中两项外，C# 特性名与 C++ 宏字符串**字符级完全一致**（拼写、大小写均一致），不存在"宏字符串抄错导致元数据静默丢失"的情况。这是一项重要的**否定性结论**——任务预期的"手抄说明符拼错"风险在宏表中没有发生（唯一的拼写差异 `BlueprintAutoCast` 在 C# 侧类名与 C++ 宏字符串之间是**自洽**的，见 F-CS4-012）。

### 4.3 元数据键与 UE 5.6 官方键表的一致性核对

权威参照物：`Engine/Source/Runtime/CoreUObject/Public/UObject/ObjectMacros.h` 的 `namespace UM`（第 1178–1762 行），它按 `[ClassMetadata]` / `[StructMetadata]` / `[PropertyMetadata]` / `[FunctionMetadata]` / `[InterfaceMetadata]` 逐条列出所有元数据键。

| 检查 | 结果 |
|---|---|
| 从 `ObjectMacros.h:1152-1762` 提取的 UE 元数据键 | **165** 个 |
| C# 特性名（去掉 `Attribute`）在 UE 键表中**精确命中**（含仅大小写不同） | **150** 个 |
| 未命中（= UPROPERTY/UFUNCTION/UCLASS 的**标志类说明符**，由 `SetPropertyFlags`/`SetFunctionFlags`/`ClassFlags` 处理，而不是元数据） | **105** 个 |
| 其中"仅大小写不同"的 | **1** 个：`BlueprintAutoCast` ↔ `BlueprintAutocast`（见 F-CS4-012，因 `FName` 大小写不敏感而仍生效） |
| UE 有、插件完全没有对应 C# 特性的元数据键 | 15 个：`AllowEditInlineCustomization`、`AnimBlueprintFunction`、`Atomic`(US)、`BlueprintInternalUseOnlyHierarchical`(US)、`ForceRebuildProperty`、`GetAllowedClasses`、`GetAssetFilter`、`GetClassFilter`、`GetDisallowedClasses`、`GetRestrictedEnumValues`、`HidePinAssetPicker`(UE5.6 新增)、`Immutable`(US)、`ObjectMustImplement`、`ScriptMethodMutable` —— 属**功能缺失**（UE 侧可用、插件未暴露），非缺陷，但 `HidePinAssetPicker`/`ScriptMethodMutable` 建议补上 |

对 105 个"未命中"逐个核对消费路径后，全部有归属（`SetPropertyFlags` 34 处见 `FDynamicGeneratorCore.cpp:264-446`；`SetFunctionFlags` 见 `:488-667`；类标志见 `:716-722`、`FDynamicInterfaceGenerator.cpp:170`；类型标记见 F-CS4-010；组件相关见 `FDynamicClassGenerator.cpp:594-633`）。**未发现"投递到 UE 但 UE 不认识"的键**（除 F-CS4-013 的两个弃用键）。

### 4.4 "注册但无人查询"的特性（33 个）

完整清单与逐条证据见 **F-CS4-006**。此处只列总表：

| 度量 | 数值 |
|---|---|
| `FReflectionRegistry.h` 中声明的 `Get{X}AttributeClass()` | 254 |
| 在 `FReflectionRegistry.*` 之外**零调用**的 | **33** |
| 其中同时会造成功能缺陷的 | 1（`VisibleDefaultsOnly`，见 F-CS4-003） |
| 其中与已有正向实现成对、补实现成本最低的 | 5（`NotBlueprintable`、`NotBlueprintType`、`NotEditInlineNew`、`VisibleDefaultsOnly`、`DontCollapseCategories`） |

---

## 5. 横向维度检查结果汇总

| 维度 | 结论 |
|---|---|
| **空指针/越界/未检查返回值** | 特性消费侧安全：`FReflection.cpp:25-31` 对缺失键/越界下标返回空 `FString`（不崩溃）；`MetaData.cpp:444` 接受空值。生成器侧存在两处脆弱点：`UnrealTypeSourceGenerator.cs:520-522` 直接用 `errorAttribute.SyntaxTree`（当前有隐含不变量保护）、`:302`/`:634` 对可能为 null 的 `filePath` 调用 `Path.GetFileName` 会静默产生**误报诊断**（见 F-CS4-009）。 |
| **内存与资源泄漏** | 未发现。本模块无 `new`/`malloc`/`GCHandle`；生成器只产生字符串与 Roslyn 语法树；`Template/*.cs` 无 `IDisposable` 使用。`Template/Dynamic|Override` 模板也没有"演示 `Dispose`"的内容 —— 这是模板的**内容缺失**（任务提到"是否演示了正确的 Dispose 用法"，答案是：模板完全未涉及 Dispose/句柄管理，作为入门示例偏薄）。 |
| **线程安全** | 未发现。特性消费全部发生在编辑器内动态类生成期（GameThread）；源生成器由 Roslyn 单线程调用。 |
| **异常与错误处理** | 见 F-CS4-009：生成器零 `try/catch`。C++ 侧 `SetMetaData` 无异常路径。 |
| **性能** | 见 F-CS4-008：非增量生成器，每次编译全量遍历语法树。另注意 `UnrealTypeSourceGenerator.cs:263-392` 的 `OnVisitSyntaxNode` 对**每个**语法节点做字符串匹配（`GetAttributeFromClass` 用 `attribute.Name.ToString()` 拼 `Name + "Attribute"`，`:682-688`），在大项目里是可测量的开销。 |
| **死代码** | 见 §4：2 个零引用特性 + 33 个注册未查询特性；另 `FReflectionRegistry.h` 中 254 个 getter 有 33 个是纯样板。 |
| **可优化/可读性/一致性** | F-CS4-012（大小写）、F-CS4-014（缺 `sealed`）、F-CS4-017（模板构造函数）、F-CS4-019（模板命名空间）、F-CS4-020（Fody 版本）。 |
| **平台兼容** | F-CS4-011（模板路径假定 `<Project>/Plugins/`，Windows 反斜杠写死在 `Template/UE.csproj:4`、`Template/Interop.csproj:12`、`Template/Game.props:10`）。`Script.sln` 的路径分隔符是 `\`（`Template/Script.sln:15,17`），这是 .sln 格式惯例，`dotnet sln`/VS 均可解析，**不构成问题**（已实测 `dotnet msbuild Script/Game/Game.csproj` 正常求值）。 |

---

## 6. 模板工程的交叉核对结果（任务要求的三方核对）

### 6.1 目标框架：`Template/*` ↔ `DotnetVersion.h` ↔ 生成期替换 ↔ 实际生成物

| 环节 | 证据 | 结论 |
|---|---|---|
| 模板中的占位符 | `Template/Shared.props:3` `<TargetFramework></TargetFramework>`；`Template/Interop.csproj:10` 同 | 空占位，生成期替换 |
| 替换实现 | `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:258-268`：`"<TargetFramework>net%d.%d</TargetFramework>"`，参数为 `FUnrealCSharpFunctionLibrary::GetDotnetVersion()` 与 `DOTNET_MINOR_VERSION` | — |
| 主版本来源 | `FUnrealCSharpFunctionLibrary.cpp:1395-1405`：`GetDotnetVersion()` 返回 `EDotnetVersion` 值，`Latest` 时返回 `Latest - 1`；枚举定义 `Setting/UnrealCSharpSetting.h:66-72`（`V8=8, V9=9, V10=10, Latest`） → `Latest - 1 = 10` | 主版本 ∈ {8,9,10} |
| 次版本来源 | `DOTNET_MINOR_VERSION` 由第三方后端模块以 `PublicDefinitions` 提供：`ThirdParty/CoreCLR/CoreCLR.Build.cs:13-24`（`MajorVersion=10, MinorVersion=0, PatchVersion=4`）、`ThirdParty/Mono/Mono.Build.cs:13-14`（`10 / 0`）、`ThirdParty/LeanCLR/LeanCLR.Build.cs:19-20`。`UnrealCSharpCore.build.cs:275-299` 通过 `PublicDependencyModuleNames.Add("CoreCLR"/"Mono"/"LeanCLR")` 传递到 `ScriptCodeGenerator` | 次版本 = **0**（三种后端一致） |
| `DotnetVersion.h` | `Source/CrossVersion/Public/DotnetVersion.h:9-13`：`DOTNET8 = ...(8,0,0)`、`DOTNET9`、`DOTNET10`，由 `DOTNET_MAJOR/MINOR/PATCH_VERSION` 比较得出 | 与实际加载的运行时一致（.NET 10.0.4） |
| **实际生成物** | `Script/Shared.props:8` = `<TargetFramework>net10.0</TargetFramework>`；`Script/Interop/Interop.csproj:15` = `net10.0`；`dotnet msbuild Script/Game/Game.csproj -getProperty:TargetFramework` = `net10.0` | — |
| **结论** | `net10.0` ↔ `DOTNET_MAJOR_VERSION=10` / `DOTNET10` 宏 **一致**。**未发现目标框架不一致问题**（任务预期的 P1 未成立）。 | ✅ |
| 附带风险 | 若用户把 `DotnetVersion` 设为 `V8`，生成 `net8.0` 而运行时仍是 .NET 10（`CoreCLR.Build.cs` 的版本与设置**无关**，硬编码 10.0.4）—— 靠 roll-forward 仍可运行；但 **Mono 后端**同样会生成 `net10.0`（`Mono.Build.cs:13-14` 也是 10/0），而 Mono 运行时通常不支持 net10.0 程序集，属跨版本模块的潜在问题（**留给 05-编译器与跨版本 报告**，此处仅记录，置信度中）。 | ⚠️ |

### 6.2 程序集输出路径：模板 ↔ `FUnrealCSharpFunctionLibrary` ↔ C++ 加载路径

| 关键量 | 模板侧 | C++ 侧期望 | 实测生成物 | 一致？ |
|---|---|---|---|---|
| 发布目录 | `Template/Game.props:21` `<ScriptOutputPath></ScriptOutputPath>`（空占位） | `GetFullPublishDirectory()` = `FPaths::ProjectContentDir() / GetPublishDirectory()`（`FUnrealCSharpFunctionLibrary.cpp:1026`） | `Script/Game/Game.props:26` = `..\..\Content\Script`（从 `Script/Game/` 解析 = `<Project>/Content/Script`） | ✅ |
| 替换实现 | — | `FSolutionGenerator.cpp:226-234`（`..\\..\\Content\\%s`） | — | ✅ |
| Interop 输出 | `Template/Interop.csproj:19` `<OutputPath></OutputPath>` + `:20` `<AppendTargetFrameworkToOutputPath>false</AppendTargetFrameworkToOutputPath>` | `GetFullInteropPublishPath()` = `GetFullPublishDirectory()/GetUEName()+".dll"`（`FUnrealCSharpFunctionLibrary.cpp:1041`） | `Script/Interop/Interop.csproj:24` = `..\..\Content\Script`；`:25` `AppendTargetFrameworkToOutputPath=false`；实际存在 `<Project>/Content/Script/Interop.dll`（30720 B） | ✅ |
| 程序集引用 | `Template/Shared.props:17-19` `<Reference Include="Interop"><HintPath></HintPath>` | `FSolutionGenerator.cpp:246-256` → `..\\..\\Content\\%s\\Interop.dll` | `Script/Shared.props:23` = `..\..\Content\Script\Interop.dll`（存在） | ✅ |
| 程序集名 | 模板未设 `<AssemblyName>`，`Game.csproj` 文件名为 `<GameName>.csproj` → `AssemblyName` 默认 = 文件名 | 源码生成器要求 `AssemblyName == GetGameName()`（`UnrealTypeSourceGenerator.cs:13,62` + `FSolutionGenerator.cpp:292-296` 的 `ReplaceGameName`） | `dotnet msbuild -getProperty:AssemblyName` = `Game`；生成器内 `GameAssemblyName = "Game"`（`Script/SourceGenerator/UnrealTypeSourceGenerator.cs:18`） | ✅ |
| UE 程序集 | `Template/UE.csproj` 无 `OutputPath`，`AssemblyName` = `UE` | `GetUEAssemblyPath()` = `…/GetUEName()+".dll"` | `<Project>/Content/Script/UE.dll`（20191744 B）存在 | ✅ |
| `Nullable` | `Template/Shared.props` **未设** `<Nullable>`（默认 disable）；`Template/Interop.csproj:13` 设 `enable` | — | 生成物相同 | ⚠️ 见下 |
| `LangVersion` | 所有模板 csproj 均**未设** `<LangVersion>` → 由 TFM 决定（net10.0 ⇒ C# 14） | 源码生成器自身 `netstandard2.0`（⇒ C# 7.3） | 生成物相同 | ✅（但见下） |
| `GenerateAssemblyInfo` | 未显式设置（默认 true） | — | — | ✅ |
| `AllowUnsafeBlocks` | `Template/Shared.props:4` `True`；`Template/Interop.csproj:14` `true` | 必需（`Script/UE/Library/*.cs` 大量 `unsafe partial`） | ✅ | ✅ |

**一致的结论**：任务担心的"C++ 期望 `bin/Debug/net8.0/Game.dll` 而模板输出到别处"**不成立** —— 模板刻意用 `AppendTargetFrameworkToOutputPath=false` + 显式 `OutputPath` 把产物直接落进 `Content/<PublishDirectory>`，与 `FUnrealCSharpFunctionLibrary` 的路径计算吻合，`AfterBuildPublish`→`Publish`→`CopyDllsAfterPublish` 三步把 `$(PublishDir)*.dll` 拷到同一目录（`Template/Game.props:21-33`）。实测 `<Project>/Content/Script/` 下 `Game.dll`/`UE.dll`/`Interop.dll` 三件齐备，且 `Script/Game/bin/Debug/net10.0/publish/` 存在，证明这条链**在当前机器上确实跑通了**。

**两点提醒**（非缺陷，作为观察）：
1. `Template/Shared.props` 没有 `<Nullable>`，而 `Template/Interop.csproj:13` 有 `<Nullable>enable</Nullable>`。`Script/UE`（由 `UE.csproj` 编译）里的代码大量使用可空语义（`return Handle != 0 ? … : null;`），在 `Nullable=disable` 下不会报错但也不会得到检查。若要开启，需先清理 `Script/UE` 的警告（当前 `Template/Shared.props:11,14` 的 `<NoWarn>0109;1701;1702;8500</NoWarn>` 说明作者已在压制警告）。
2. `Script/ComplierRunnable`（`Source/Compiler/Private/FCSharpCompilerRunnable.cpp:281-287`、`:386-391`）用的是 `dotnet build "<csproj>" --nologo -c <Config>`，**没有** `/p:ProjectPath=`。`Template/Game.props:22` 使用 `$(ProjectPath)`，我为此专门做过验证：`dotnet msbuild Script/Game/Game.csproj -getProperty:ProjectPath` 返回 `<Project>/Script\Game\Game.csproj`，即该属性在 .NET SDK 下**已定义**，`AfterBuildPublish` 能正常触发（`Script/Game/bin/Debug/net10.0/publish/` 的存在即为旁证）。**此点无问题**，记录在此以免误判。

### 6.3 `Script.sln` 的 GUID 与配置

| 项目 | sln 中的 GUID | 来源 | csproj `ProjectGuid` |
|---|---|---|---|
| UE | `{7AF881DC-664B-4AF7-BCB6-8FA8CC5E8780}` | `Template/Script.sln:6`（模板里为空工程名，由 `FSolutionGenerator::ReplaceProject` 填充，`FSolutionGenerator.cpp:312-324`） | SDK 风格 csproj **不存在** `ProjectGuid`（GUID 由 sln 单方面持有，这是 VS 的常规做法） |
| Game | `{A2B210E9-51AE-490B-8B87-F8492CB2A417}` | `Template/Script.sln:8`，`FSolutionGenerator.cpp:326-336` | 同上 |
| SourceGenerator | `{095D641F-E823-4EE2-89A8-EBC636049F98}` | `Template/Script.sln:15` | 同上 |
| Weavers | `{DB42848F-4C21-4581-99BA-C92CB37D4024}` | `Template/Script.sln:17` | 同上 |
| Interop | `{E3A5C6B1-2D4F-4A8E-9B7C-1F6D3E2A5B8C}` | `Macro.h:53` `INTEROP_GUID`，由 `ReplaceProjectPlaceholder`（`FSolutionGenerator.cpp:339-379`）注入 | 同上 |

**核对结果**：实测生成物 `Script/Script.sln` 共 5 个工程（`:6,8,15,17,19`），`ProjectConfigurationPlatforms` 段对 5 个 GUID 各给出 4 行配置（`:27-46`），`NestedProjects` 把 `SourceGenerator`/`Weavers` 归入 `Utils` 解决方案文件夹（`:52-53`）。**GUID 与配置一一对应、无遗漏、无重复**，项目类型 GUID `{9A19103F-...}`（C# SDK 工程）与 `{2150E333-...}`（解决方案文件夹）使用正确。**未发现问题。**

两个小观察：
- `Template/Script.sln:9-11` 给 Game 声明了对 Weavers 的 `ProjectDependencies`（`postProject`），这是为了确保编织器先编译 —— 合理。但 `Script.sln` 中 Game 并没有对 `Interop`/`UE` 的依赖项（靠 `ProjectReference` 隐式建立）—— 一致，无问题。
- `.sln` 里路径用 `\`（`SourceGenerator\SourceGenerator.csproj`），这是 .sln 格式的惯例写法，`dotnet` 与 VS 均正常解析；**不构成非 Windows 平台的转义问题**。

### 6.4 模板代码的可编译性抽查

| 模板文件 | 关键 API / 类型 | 仓库内的真实定义 | 结论 |
|---|---|---|---|
| `Template/Dynamic/DynamicActor.cs:8,15,21` | `AActor`、`ReceiveBeginPlay()`、`ReceiveEndPlay(EEndPlayReason)` | `Script/UE/Proxy/Engine/Classes/GameFramework/Actor.cs:14`（`public partial class AActor : UObject, IStaticClass`）、`:3553`（`public virtual void ReceiveBeginPlay()`）、`:3533`（`public virtual void ReceiveEndPlay(EEndPlayReason EndPlayReason)`） | 签名匹配，`override` 合法 ✅ |
| `Template/Dynamic/DynamicActorComponent.cs:8` | `UActorComponent` | `Script/UE/Proxy/Engine/Classes/Components/ActorComponent.cs:13`（`namespace Script.Engine`）、`:761`、`:749` | 命名空间 `Script.Engine` 与模板的 `using Script.Engine;` 匹配 ✅ |
| `Template/Dynamic/DynamicUserWidget.cs:8,15,21` | `UUserWidget`、`Construct()`、`Destruct()` | `Script/UE/Proxy/UMG/Blueprint/UserWidget.cs:14`（`namespace Script.UMG`）、`:17`（`public partial class UUserWidget : UWidget, IStaticClass, INamedSlotInterface`） | `using Script.UMG;` 正确、类名/命名空间匹配 ✅ |
| `Template/Override/*.cs` | 同名 partial + `[Override]` | `Script/Game/UnrealCSharpTest/UnitTest/Reflection/BlueprintCSharpFunction/BP_TestCSharpFunctionActor_C.cs:6-9` 的写法 | 模式一致；构造函数差异见 F-CS4-017 ⚠️ |
| 全部模板的 `[Override]` | `Script.CoreUObject.OverrideAttribute` | `Script/UE/CoreUObject/OverrideAttribute.cs:6`（`AttributeUsage = Class \| Method`） | 类/方法都允许 ✅ |
| 全部模板 | `dispose`/`IDisposable` | 模板内**无任何** Dispose 示例 | 模板作为"句柄管理"的入门示例是缺失的（观察项，非缺陷） |

**命名冲突提醒**：`Template/Override/Actor.cs:7` 的 `public partial class AActor` 位于 `namespace Script.Engine`，而 `Script.Engine.AActor` 由插件生成的代码**编译进 UE 程序集**（`Script/UE/Proxy/...` 通过 `Template/UE.csproj:4` 的 `Compile Include` 纳入 UE 工程）。如果把 `Override/*.cs` 原样放进**游戏**工程，会出现"游戏程序集里另有一个 `Script.Engine.AActor`"——C# 会对同一编译单元的引用产生 CS0436 冲突警告，且 `[Override]` 注册的是游戏程序集里的那个类型。因此这四个模板**必须**经过 `UnrealCSharpBlueprintToolBar.cpp:110-134` 的命名空间/类名替换后才能使用（替换后它们才成为目标 `_C` 类的 partial 扩展，与框架类型不在同一命名空间）。建议在模板文件顶部加一行注释说明"请通过编辑器动作生成，不要手动复制"。

---

## 7. 未覆盖 / 存疑项

1. **`Script/Weavers/UnrealTypeWeaver.cs` 的编织逻辑本体**未深入分析（由另一份报告覆盖）。本报告只做了**重叠性判断**：源生成器（`UnrealTypeSourceGenerator`）负责生成 `IStaticClass`/`IStaticStruct` 实现（`StaticClass()`/`StaticStruct()` 单例、`Equals`/`GetHashCode`/`==`/`!=`）、以及 `Library`/`Binding` 桥接方法；编织器负责在 IL 层给动态类型/属性附加 `PathNameAttribute`、`NotPlaceableAttribute`（`UnrealTypeWeaver.cs:187-220`）与 `_propertyAttributes` 等描述符字段（`:242-289`）。**两者职责不同，不构成功能重复**：源生成器在**编译期产出 C# 源码**，编织器在**编译后改写 IL**。真正值得关注的交叉点是"两处都有 `GetPathName` 实现且规则必须一致"：
   - 源生成器：`UnrealTypeSourceGenerator.cs:50-53`（`"/Script/CoreUObject." + (Name.EndsWith("_C") ? Name : Name.Substring(1))`）
   - 编织器：`UnrealTypeWeaver.cs:222-227`（`"/Script/CoreUObject." + (name.EndsWith("_C") || Type.IsEnum ? name : name.Substring(1))`）
   两者对**枚举**的处理不同（编织器额外判断 `IsEnum`），而源生成器不处理枚举的 `StaticClass`。当前调用路径不同（枚举走编织器，类/结构体走生成器），**未发现实际冲突**，但这是"同一规则两处实现"的典型可维护性隐患，建议合并到一个共享工具方法。
2. **`LibraryBridgeGenerator` 生成代码的实际编译结果未验证**：本机未对 `Script/Game/Game.csproj` 做 `dotnet build --no-incremental`（该项目含 20 MB 的 `UE.dll` 与数万生成文件，完整重编代价过高，且会改写 `bin/obj`）。因此 F-CS4-015（重载导致 CS0102）与 F-CS4-008 的"分析器噪声数量"均为**静态推导**，未实测。
3. **`VS2026` 宏的来源与语义未确认**：F-CS4-018 由"生成物时间戳同轮 + `#if !VS2026`"推出，但 `VS2026` 与实际 VS 版本的关系（是否为"VS 2026 不支持 .sln 首行注释"的规避）未在 `Source/CrossVersion` 中确认。
4. **`EDotnetVersion` 设为 V8/V9 时与 Mono 后端的兼容性**：见 §6.1 的"附带风险"，`DOTNET_MINOR_VERSION` 恒为 0 而是由后端模块决定，Mono 后端会生成 `net10.0`。此问题属跨版本模块，本报告仅记录，置信度中。
5. **`FClassReflection` 对特性基类的处理**：本报告按"精确类型匹配"推断 `HasAttribute` 语义（`FReflection.cpp:18-21` + `FClassReflection.cpp:179-195` 的 `Attributes.Add(Attribute)`）。已确认描述符中的 `Attribute` 来自 `InManagedReader.ArrayGetClass(...)`（即 C# 侧 `CustomAttribute.AttributeType` 本身），因此 `[UFunction]` 在描述符中的类型是 `Script.Dynamic.UFunctionAttribute` 而非其基类 `OverrideAttribute`。这一推断支撑了 F-CS4-010，但未做运行期验证。
6. **模板里是否存在未替换的占位符残留**：我核对了 7 个占位符宏（`Macro.h:55-67`）与 `FSolutionGenerator.cpp` 的 8 个替换函数（`ReplaceTargetFramework`/`ReplaceOutputPath`/`ReplaceScriptOutputPath`/`ReplaceHintPath`/`ReplaceDefineConstants`/`ReplaceImport`/`ReplaceProjectReference`/`ReplaceProject`/`ReplaceProjectPlaceholder`/`ReplaceSolutionConfigurationPlatformsPlaceholder`/`AddSolutionGeneratorHeaderComment`）。实测生成物中 `ProjectPlaceholder`、`SolutionConfigurationPlatformsPlaceholder`、`<TargetFramework></TargetFramework>`、`<DefineConstants></DefineConstants>`、`<HintPath></HintPath>`、`<OutputPath></OutputPath>`、`<ScriptOutputPath></ScriptOutputPath>`、`<Import Project="" …>`、`<ProjectReference Include="" />` 全部已被替换（`Script/Script.sln`、`Script/Shared.props`、`Script/Game/Game.props`、`Script/Game/Game.csproj`、`Script/UE/UE.csproj`、`Script/Interop/Interop.csproj` 均无残留）。**未发现残留占位符。**
7. **`Template/Game.csproj` / `UE.csproj` 中的 `TODO`**：4 个 `Template/Dynamic/*.cs` 与 4 个 `Template/Override/*.cs` 均无 `TODO` 注释；但 `FDynamicGeneratorCore.cpp:322` 的 `// @TODO` 与 `:450-451` 的空块说明消费端存在两处已知未实现（已计入 F-CS4-004）。
8. **模板中的用户可编辑性**：模板**没有** `__PROJECT_NAME__` 一类未替换的占位符（见第 6 点），但有 7 个"看起来像标识符的空字符串占位"（`Project=""`、`Include=""`、`<TargetFramework></TargetFramework>` 等），它们在**插件源码树里是"非法/半成品 Xen"状态**。如果有人直接把 `Template/` 目录拷进新项目当模板用（跳过 `FSolutionGenerator`），得到的是一组不能编译的工程文件 —— 这与"Template 目录被视为可直接使用的模板"的直觉相悖，建议在 `Template/README`（目前不存在）中说明这 7 个占位符与负责替换它的 `FSolutionGenerator::Generator()`。

---

## 8. 给维护者的一页速览（按修复优先级）

| 优先级 | 编号 | 一句话 | 改动量 |
|---|---|---|---|
| 1 | F-CS4-001 | `SetFlags(FProperty*)` 的 `Replicated` 分支误读 `ReplicatedUsing` 的值，复制条件丢失 | 1 行 |
| 2 | F-CS4-002 | 143 个特性无构造参数 ⇒ C++ 拿到 `""` ⇒ UE 布尔元数据恒为 false（62 个键受影响） | 中（建议 C# 侧批量加 `string InValue = "true"`） |
| 3 | F-CS4-003 | `[VisibleDefaultsOnly]` 无分支，属性在细节面板完全不出现 | 6 行 |
| 4 | F-CS4-004 | `[Localized]`/`[SealedEvent]` 分支体为空 | 2–4 行 + 日志 |
| 5 | F-CS4-006 | 33 个特性注册但无人查询（含 `NotBlueprintable`/`Within`/`EditInlineNew`） | 按表补实现或加"未实现"告警 |
| 6 | F-CS4-005 | `[CustomThunkTemplates]`/`[DontAutoCollapseCategories]` 零引用 | 删除或实现 |
| 7 | F-CS4-007 | 诊断 ID 与 `AnalyzerReleases.Unshipped.md` 不一致，且 md 未声明为 `AdditionalFiles` | 5 行 + csproj 3 行 |
| 8 | F-CS4-009 | 生成器无 `try/catch`，用户错误会变成 "source generator failed" | ~20 行 |
| 9 | F-CS4-008 | 非增量生成器 + 生成文件缺 `// <auto-generated>` 头 | 迁移 `IIncrementalGenerator`（较大）；仅加头与改后缀是小改 |
| 10 | F-CS4-010 | 特性基类三套写法；`[KismetHideOverrides]` 继承 `UClassAttribute` 导致两端判定分歧 | 3 行 |
| 11 | F-CS4-011 | 模板硬编码 `Plugins\UnrealCSharp`，且 `ReplacePluginBaseDir` 未挂到 Interop.csproj | 中 |

