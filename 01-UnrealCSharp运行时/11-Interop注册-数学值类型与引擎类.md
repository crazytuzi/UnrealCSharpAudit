# Interop 注册实现：数学 / 值类型 / 资产 / 引擎类

> 本报告各条**严重度见各 Finding 的「复核结论」字段**（28 条：含 1 条判定为非缺陷、13 条级别调整）。
> **源码级核对结论**：28 条逐条回源码重核 —— **确认 15 / 部分确认 13 / 证伪 0 / 无法验证 0**；级别 **下调 2 条（`F-INT2-003` P2→P3、`F-INT2-006` P1→P2）、上调 1 条（`F-INT2-019` 撤销→P3）、维持 25 条**；修正后分布 **P0=1 / P1=7 / P2=7 / P3=13 / 撤销 0**（按各条 `**严重度**` 字段统计）。
> 用**生成地面真值** `Script/UE/Proxy/Binding/*.cs`（16055 个文件）与 **UE 5.6 引擎源码**（`Engine\Source`）交叉验证，据此**证伪了 7 处"形式化推断"**（集中在 `F-INT2-003/004/005/006/017/018/020`：`ParamNames` 长度**不决定** C# 形参个数、`Algo::Count` 后缀**不外泄**到 C# 名）。详见各条 `**复核证据**`。
> ⚠️ **`F-INT2-*` 编号与 `01-…/10b` 撞号**，引用务必带报告路径，见 §3 开头的注记。





> 分析范围：`Plugins/UnrealCSharp/Source/UnrealCSharp/Private/Domain/Interop/` 下 36 个 `FRegister*.cpp`
> 覆盖文件：36 个（任务书列 37 个，实际目录中 `Source/UnrealCSharp/Private/Domain/Interop/` 只有 36 个属于本组；已逐一核对，见第 0 节）
>
> 阅读情况：36/36 文件全部逐行读完；未覆盖部分见第 5 节

---

## 0. 覆盖范围与阅读清单

| 文件 | 行数 | 是否读完 | 注册条目数 | 备注 |
|---|---|---|---|---|
| `FRegisterVector.cpp` | 345 | 是 | 133（ctor 5 / prop 11） | 数学核心 |
| `FRegisterVector2D.cpp` | 183 | 是 | 60（ctor 6 / prop 3） | 含 UE 版本门控 |
| `FRegisterVector4.cpp` | 132 | 是 | 37（ctor 6） | |
| `FRegisterRotator.cpp` | 99 | 是 | 39（ctor 4） | **2 处函数错绑** |
| `FRegisterTransform.cpp` | 183 | 是 | 85（ctor 6 / prop 1） | |
| `FRegisterMatrix.cpp` | 104 | 是 | 50（ctor 3 / prop 1） | **1 处函数错绑** |
| `FRegisterQuat.cpp` | 160 | 是 | 64（ctor 6 / prop 1） | |
| `FRegisterColor.cpp` | 70 | 是 | 37（ctor 2 / prop 14） | |
| `FRegisterLinearColor.cpp` | 90 | 是 | 39（ctor 3 / prop 8） | 重复 include |
| `FRegisterPlane.cpp` | 73 | 是 | 20（ctor 6） | 非 const 引用重载 |
| `FRegisterBox2D.cpp` | 63 | 是 | 22（ctor 4） | **裸指针+count 构造** |
| `FRegisterGuid.cpp` | 87 | 是 | 14（ctor 2） | 手抄 UE operator> |
| `FRegisterDateTime.cpp` | 149 | 是 | 42（ctor 2） | |
| `FRegisterTimespan.cpp` | 82 | 是 | 35（ctor 4） | |
| `FRegisterFrameNumber.cpp` | 40 | 是 | 3（ctor 1） | float 标量 |
| `FRegisterFrameTime.cpp` | 51 | 是 | 13（ctor 3 / prop 1） | float 标量 |
| `FRegisterRandomStream.cpp` | 52 | 是 | 20（ctor 2） | 状态语义无问题 |
| `FRegisterFloatInterval.cpp` | 28 | 是 | 7（ctor 1） | |
| `FRegisterFloatRange.cpp` | 63 | 是 | 17（ctor 3） | TODO 未实现项多 |
| `FRegisterFloatRangeBound.cpp` | 40 | 是 | 15（ctor 1） | |
| `FRegisterInt32Interval.cpp` | 28 | 是 | 7（ctor 1） | 与 FloatInterval 逐行同构 |
| `FRegisterInt32Range.cpp` | 63 | 是 | 17（ctor 3） | 与 FloatRange 逐行同构 |
| `FRegisterInt32RangeBound.cpp` | 40 | 是 | 15（ctor 1） | 与 FloatRangeBound 逐行同构 |
| `FRegisterPrimaryAssetId.cpp` | 38 | 是 | 9（ctor 2 / prop 2） | 参数名风格不一致 |
| `FRegisterPrimaryAssetType.cpp` | 28 | 是 | 4（ctor 1） | |
| `FRegisterSoftClassPath.cpp` | 32 | 是 | 5（ctor 3） | 拷贝构造参数名多一项 |
| `FRegisterSoftObjectPath.cpp` | 83 | 是 | 29（ctor 5） | **1 处函数错绑** |
| `FRegisterAssetBundleData.cpp` | 48 | 是 | 7（ctor 1） | `TArray<...>&&` 重载 |
| `FRegisterAssetBundleEntry.cpp` | 27 | 是 | 3（ctor 2） | |
| `FRegisterPolyglotTextData.cpp` | 50 | 是 | 20（ctor 1） | **2 处函数错绑/参数名错** |
| `FRegisterDataTableFunctionLibrary.cpp` | 52 | 是 | 1 | **P0 空指针解引用** |
| `FRegisterEnhancedInputComponent.cpp` | 227 | 是 | 6 | **多处缺 null 检查** |
| `FRegisterInputComponent.cpp` | 345 | 是 | 8 | **缺 null 检查 + 重复代码** |
| `FRegisterKeys.cpp` | 416 | 是 | 342（prop 342；**UE 5.6 生效 335，门控关闭 7**） | **可写属性问题**；**EKeys 全量对照见第 5 节第 2 项：漏绑 13、多绑 7（全部门控关闭）** |
| `FRegisterUnreal.cpp` | 192 | 是 | 7 | **缺 null 检查** |
| `FRegisterWorld.cpp` | 155 | 是 | 16（ctor 0 / prop 9） | **缺 null 检查** |
| **合计** | 3856 | 36/36 | **1248** | |

> 未读文件：无。任务书中的 37 个文件名去重后为 36 个（`FRegisterVector`/`Vector2D`/…/`World` 共 36 项）；目录中其余 `FRegisterArray/Map/Set/Optional/Delegate/...` 属于其他报告的范围。

---

## 1. 模块职责与架构速览

### 1.1 这批文件**不是** `extern "C"` 导出

任务书假设"以 `extern "C"` 导出给 C# 用 `DllImport` 调用"。**实际机制不同，这决定了后面所有严重度的判断**：

1. 每个文件在匿名命名空间里定义一个 `struct FRegisterXxx`，并声明一个具名（`[[maybe_unused]]`）全局静态对象（如 `FRegisterVector.cpp:344`）。构造顺序即注册顺序，纯粹依赖**静态初始化**。
2. 构造函数里调用 `TBindingClassBuilder<T>(NAMESPACE_BINDING)` / `FClassBuilder(TEXT("Unreal"), NAMESPACE_LIBRARY)`，链式 `.Constructor()/.Function()/.Property()/.Subscript()` 与运算符宏（`.Plus()`、`.Multiplies()`…）。
3. `FClassBuilder::Function(InName, InMethod)`（`Source/UnrealCSharp/Public/Binding/Class/FClassBuilder.inl:4-20`）把 `InName` 同时当作**绑定名**与**实现名**，转调 `Function(InName, InImplementationName, ptr)`。
4. `FClassBuilder::Function(...)`（`FClassBuilder.cpp:59-81`）用 `GetFunctionImplementationName()` 计算最终实现名：

```cpp
// Source/UnrealCSharp/Private/Binding/Class/FClassBuilder.cpp:83-91
FString FClassBuilder::GetFunctionImplementationName(const FString& InName, const FString& InImplementationName) const
{
	const auto Count = Algo::Count(Functions, InName);

	return FString::Printf(TEXT("%s%s"), *InImplementationName,
	                       Count == 0 ? TEXT("") : *UKismetStringLibrary::Conv_IntToString(Count));
}
```

   **重载消歧靠"同名出现次数"，即 `Name`、`Name1`、`Name2`……** 这是本批次最重要的语义前提：**注册时写的字符串就是 C# 看到的 API 名**。
5. 运行时由 `FScriptDomainImpl::RegisterBinding()`（`Source/UnrealCSharpCore/Public/Domain/Script/FScriptDomainImpl.inl:601-633`）遍历所有类的所有方法，把 `"<Namespace>.<Class>::<Method>"`（UTF-8）与函数指针通过 `MethodBridgeRegisterBindingFn` 推给 C# 的 `Interop.MethodBridge.RegisterBinding`（`Script/Interop/Bridge/MethodBridge.cs:13-30`），存入 `Dictionary<string,nint>`（`MethodBridge.cs:10`）。
6. C# 侧由 **源生成器**产生调用桩（`Script/SourceGenerator/UnrealTypeSourceGenerator.cs:873-927`）：
   - LeanCLR 后端走 `[DllImport("UnrealCSharp", CallingConvention = CallingConvention.Cdecl)]`（`UnrealTypeSourceGenerator.cs:911`），**这才是唯一的真 P/Invoke**；
   - Mono/CoreCLR 后端走函数指针缓存 + `delegate* unmanaged[Cdecl]`：

```csharp
// Script/SourceGenerator/UnrealTypeSourceGenerator.cs:914-918
private static nint {slot};

{accessibility} static unsafe partial {returnType} {method.Name}({parameters}) =>
    ((delegate* unmanaged[Cdecl]<{pointerType}>)global::Interop.MethodBridge.GetMethod(
        ref {slot}, "{key}"))({arguments});
```

   **`GetMethod` 在名字查不到时返回 0（`MethodBridge.cs:140-148`），生成代码不做判空就调用 → 名字不匹配 = 立即空函数指针调用（访问违例）。** 这是本模块最高危的失效模式。
7. 手写的 `Script/UE/Library/*Implementation.cs` 以 `private static unsafe partial` 声明与 C++ 侧**同名的** `__<Class>_<Method>Implementation`（约定来自 `Source/UnrealCSharpCore/Public/CoreMacro/BindingMacro.h:9`：`"__%s_%sImplementation"`），由生成器补齐 DllImport/函数指针体。

### 1.2 数据/参数传递约定

- 统一 thunk 签名 `BINDING_FUNCTION_SIGNATURE = const IManagedHandle, IN_BUFFER, OUT_BUFFER, RETURN_BUFFER`（`Source/UnrealCSharp/Public/Macro/SignatureMacro.h:14`）。**`bool`/`float`/结构体都不直接跨边界**，一律经缓冲读写（`TArgument.inl:28-29`、`TReturnValue.inl:18-33`）。因此"`bool` 宽度不一致"这类经典 ABI 问题在本架构中基本不存在，**真正需要人工核对的只有少数手写签名**（见 F-INT2-009 的 ABI 抽查表）。
- `IManagedHandle` 是 `int64`（`Source/UnrealCSharpCore/Public/Domain/Script/IManagedHandle.h:5-7`），C# 侧一律用 `nint` → **隐含 64 位假设**。
- 反射注册只在编辑器构建生效：`#define WITH_FUNCTION_INFO WITH_EDITOR`（`Source/UnrealCSharpCore/Public/CoreMacro/BindingMacro.h:17`）。即 thunk 在所有配置都注册，但 **C# API 只在编辑器构建生成**。

---

## 2. 关键调用链

1. `FRegisterWorld.cpp:150 (.Function("SpawnActor", SpawnActorImplementation))` → `FClassBuilder.inl:13 Function(InName,InName,ptr)` → `FClassBuilder.cpp:83 GetFunctionImplementationName` → 绑定名 `__UWorld_SpawnActorImplementation` → `FScriptDomainImpl.inl:601 RegisterBinding` → `MethodBridge.cs:13 RegisterBinding` → `MethodBridge.cs:144 GetMethod("Script.Library.UWorldImplementation::__UWorld_SpawnActorImplementation")` → `Script/UE/Library/WorldImplementation.cs:13` → `Script/UE/CoreUObject/World.cs:9 UWorld.SpawnActor<T>`。
2. `FRegisterDataTableFunctionLibrary.cpp:47` → `DataTableFunctionLibraryImplementation.cs:14` → `Script/UE/CoreUObject/DataTableFunctionLibrary.cs:9 GetDataTableRowFromName<T>`。**这是本组唯一"手写 C# 名称与 C++ 完全对齐"的库类之一。**
3. `FRegisterKeys.cpp:14-411`（342 个 `.Property`）→ `FClassBuilder.inl:23-50 Property` → 每项生成 `Get<Name>` / `Set<Name>` 两个 thunk + `TPropertyBuilder` 读写 → C# `EKeys` 的可读写属性。
4. `FRegisterEnhancedInputComponent.cpp:221 (.Function("BindAction", BindActionImplementation))` → `MethodBridge.cs:144` → `EnhancedInputComponentImplementation.cs:24` → `FBlueprintEnhancedInputActionBinding` 结构句柄 → `FRegisterEnhancedInputComponent.cpp:174 GetStruct<...>` → `FRegisterEnhancedInputComponent.cpp:179 UEnhancedInputComponent::BindAction`。
5. **错误链（见 F-INT2-006，原文误标为 F-INT2-005，已修正）**：`FRegisterRotator.cpp:46 (.Function("Equals", BINDING_FUNCTION(&FRotator::IsNearlyZero, …)))` → 绑定名 `"…::Equals"` → C# 生成 **`FRotator.Equals(double Tolerance = 0.0001)`**（生成产物 `Script/UE/Proxy/Binding/Rotator.cs:196`，**只有 1 个形参**）实际调用 `FRotator::IsNearlyZero(Tolerance)`；引擎真实签名为 `Equals(const TRotator<T>& R, T Tolerance)`（`Runtime/Core/Public/Math/Rotator.h:234`），**未被暴露**。
6. `FRegisterMatrix.cpp:78 (.Function("GetColumn", &FMatrix::SetColumn))` → `Algo::Count` 计数为 1 → **thunk 实现名** `GetColumn1` → C# 生成产物为**一对同名重载** `Matrix.cs:574 GetColumn(int i)` / `:590 GetColumn(int i, FVector Value)`，**`SetColumn` 这个名字在 C# 中不存在**（能力本身可用）。

---

## 3. 发现清单

> ⚠️ **编号撞号注记（必读）**：本报告使用的 `F-INT2-*` 前缀**与 `01-UnrealCSharp运行时/10b-Interop注册-容器字符串与对象指针.md` 撞号** —— 两份报告各有一套 `F-INT2-*`，指向**完全不同的发现**。已核实撞号编号共 **16 个**：`F-INT2-001` … `F-INT2-015` 与 `F-INT2-018`。本报告**独有**的编号是 `F-INT2-016`、`F-INT2-017`、`F-INT2-019` … `F-INT2-028`。
> **因此：任何跨文档引用 `F-INT2-0xx` 时必须同时给出报告路径**（例如写作 `01-…/11#F-INT2-003`），否则会指向错误的发现。本报告全部 28 条编号依次为：001-028（连续、无重号、无缺号）。
> 另注：本报告正文的 `### P0` / `### P1` / `### P2` / `### P3` 分节标题与各条的 `**严重度**` 字段**存在一处不一致** —— `F-INT2-002` 物理上排在 `### P0` 节内，但其严重度为 **P1**（复核确认 P1 正确）。按 `**严重度**` 字段统计为 **P0=1**（仅 `F-INT2-001`）。

### P0

### [F-INT2-001] `GetDataTableRowFromName` 未校验 `TMap::Find` 结果，行名不存在时解引用空指针崩溃

- **类别**: Bug / 未定义行为
- **严重度**: **P0**(崩溃/数据损坏)
- **复核结论**: 确认 —— 源码事实、调用上下文、类别、严重度**全部成立**。**终裁：P0 维持**
- **可达性**: 活跃
- **复核证据**: ①`Source/UnrealCSharp/Private/Domain/Interop/FRegisterDataTableFunctionLibrary.cpp:31` = `const auto FindRowData = *DataTable->GetRowMap().Find(*InRowName);` **逐字吻合**（`:29 *OutRow = Class->InitObject();`、`:35 CopyScriptStruct` 亦吻合）。②**引擎权威对照**：引擎自己的同一逻辑 `Runtime/Engine/Classes/Engine/DataTable.h:252-260` 写作 `uint8* const* RowDataPtr = GetRowMap().Find(RowName); if (RowDataPtr == nullptr) { …UE_LOG… return nullptr; } uint8* RowData = *RowDataPtr; check(RowData);` —— 证明 `TMap::Find` 未命中返回 **nullptr**（引擎显式判 `== nullptr`），且引擎**连解引用后的值都再 `check`**。③`Runtime/Engine/Private/DataTableFunctionLibrary.cpp:68-89 Generic_GetDataTableRowFromName` 同样先 `if (RowPtr != nullptr)` 再 `CopyScriptStruct`；`:91-96` 的 `GetDataTableRowFromName` 本体是 `check(0)` 桩 → 本 thunk 属**有意重写**，因此"丢掉判空"是重写时的真实缺陷。④`GetRowMap()` 返回 `const TMap<FName, uint8*>&`：`DataTable.h:110`。
- **级别变动**: 无（P0 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterDataTableFunctionLibrary.cpp:31`
- **函数**: `FRegisterDataTableFunctionLibrary::GetDataTableRowFromNameImplementation(IManagedHandle, IManagedHandle, IManagedHandle*)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterDataTableFunctionLibrary.cpp:15-41
			if (const auto InRowName = FCSharpEnvironment::GetEnvironment().GetString<FName>(RowName))
			{
				if (InRowName->IsNone())
				{
					return 0;
				}

				if (const auto DataTable = FCSharpEnvironment::GetEnvironment().GetObject<
					UDataTable>(InManagedHandle))
				{
					FCSharpEnvironment::GetEnvironment().Bind<false>(DataTable->RowStruct.Get());

					const auto Class = FReflectionRegistry::Get().GetClass(DataTable->RowStruct);

					*OutRow = Class->InitObject();                     // :29  未判空 Class、未判空 OutRow

					const auto FindRowData = *DataTable->GetRowMap().Find(*InRowName);   // :31  未判空 Find

					const auto OutRowData = FCSharpEnvironment::GetEnvironment().GetStruct<>(*OutRow);

					DataTable->RowStruct->CopyScriptStruct(OutRowData, FindRowData);      // :35  FindRowData 为野指针
```

**调用上下文**
C# 入口 `Script/UE/CoreUObject/DataTableFunctionLibrary.cs:9 UDataTableFunctionLibrary.GetDataTableRowFromName<T>(UDataTable, FName, out T)` → `Script/UE/Library/DataTableFunctionLibraryImplementation.cs:14` → 本 thunk。由游戏逻辑在 GameThread 调用（`FCSharpEnvironment` 单例路径，`FRegisterDataTableFunctionLibrary.cpp:15/22/25`）。

**问题**
`UDataTable::GetRowMap()` 返回 `const TMap<FName, uint8*>&`，`TMap::Find(Key)` 在键不存在时返回 **nullptr**。代码直接 `*` 解引用该 `uint8**`，得到 `uint8*` = 0，随后 `CopyScriptStruct(OutRowData, nullptr)` 在引擎内部 memcpy 源地址 0 → **必然访问违例**。UE 原生 `UDataTableFunctionLibrary::GetDataTableRowFromName` 的等价写法是 `if (uint8* const* FindRowData = DataTable->GetRowMap().Find(RowName))` 才会拷贝；本实现丢掉了这个判断，只判了 `IsNone()`（`FName::None`），**判错了条件**：`IsNone()` 只挡住空 FName，挡不住"FName 合法但表中无此行"。

触发路径极其常见：C# 侧 `GetDataTableRowFromName<T>("RowThatDoesNotExist")`，或数据表被热重载后行名过期。

同一处还有两个次级缺陷：`:27` `FReflectionRegistry::Get().GetClass(...)` 返回值未判空即 `:29 Class->InitObject()`；`:29/33` `OutRow` 未判空即写入（C# 侧 `DataTableFunctionLibraryImplementation.cs:14` 传 `&OutHandle`，正常情况下非空，但该函数是 public 导出、可被其他调用方以 null 调用）。

**建议**
```cpp
const auto FindRowData = DataTable->GetRowMap().Find(*InRowName);
if (FindRowData == nullptr || *FindRowData == nullptr)
{
    return 0;                       // 与 UE 原生语义一致：未找到返回 false
}
const auto Class = FReflectionRegistry::Get().GetClass(DataTable->RowStruct);
if (Class == nullptr || OutRow == nullptr) { return 0; }
*OutRow = Class->InitObject();
const auto OutRowData = FCSharpEnvironment::GetEnvironment().GetStruct<>(*OutRow);
if (OutRowData == nullptr) { return 0; }
DataTable->RowStruct->CopyScriptStruct(OutRowData, *FindRowData);
```
更彻底的做法是直接调用引擎的 `UDataTableFunctionLibrary::GetDataTableRowFromName(DataTable, RowName, OutRow)`（虽然签名是 `FTableRowBase&`，可用 `DataTable->RowStruct` 做动态适配），避免自己重写行查找。

**验证方式**
1. `grep -n "GetRowMap().Find" Plugins/UnrealCSharp/Source` → 仅本文件 1 处命中（已确认）。
2. C# 用例：对任意 `UDataTable` 传入不存在的 `FName`，观察是否崩溃（当前实现应崩）。
3. 加 `ensure(FindRowData != nullptr)` 后跑编辑器 PIE，日志应出现 ensure 而非崩溃。

---

### [F-INT2-002] `BindAction` 对 `GetStruct<FBlueprintEnhancedInputActionBinding>` 结果直接解引用，无效句柄即崩溃

- **类别**: Bug / 未定义行为
- **严重度**: **P1**(崩溃)
- **复核结论**: 确认 —— 代码事实（`:174-175` 对 `GetStruct` 结果直接 `*` 解引用）、调用上下文（`:221 .Function("BindAction", BindActionImplementation)`）、类别、严重度 P1 全部成立
- **可达性**: 活跃
- **复核证据**: ①`:174-175` `const auto [InputAction, TriggerEvent, FunctionNameToBind] = *FCSharpEnvironment::GetEnvironment().GetStruct<FBlueprintEnhancedInputActionBinding>(InBlueprintEnhancedInputActionBinding);` **逐字吻合**；`:179-184 FoundObject->BindAction(...)` 亦吻合。②失败路径链路逐跳落实：`FCSharpEnvironment.inl:126-130 GetStruct`（`… : nullptr`）→ `FStructRegistry.cpp:78-81 GetStruct` → `:50-55 GetAddress`（`return FoundStructAddress != nullptr ? FoundStructAddress->Address : nullptr;`）。③**引擎侧**：`EnhancedInput/Public/EnhancedInputActionDelegateBinding.h:14-28` 显示 `FBlueprintEnhancedInputActionBinding` 成员**恰为 3 个**（`:19 TObjectPtr<const UInputAction> InputAction`、`:22 ETriggerEvent TriggerEvent`、`:25 FName FunctionNameToBind`）→ **"读出 4 个未初始化字段"是笔误，应为 3 个**。
- **级别变动**: 无（P0→P1 的校正**成立**：本条需 C# 侧传入无效/未注册的结构句柄才触发，而 F-INT2-001 用**完全合法**的输入（行名不存在）即可崩溃，二者不应同为 P0）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp:174-175`
- **函数**: `FRegisterEnhancedInputComponent::BindActionImplementation(IManagedHandle, IManagedHandle, IManagedHandle, IManagedHandle)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterEnhancedInputComponent.cpp:171-184
			if (const auto FoundObject = FCSharpEnvironment::GetEnvironment().GetObject<UEnhancedInputComponent>(
				InManagedHandle))
			{
				const auto [InputAction, TriggerEvent, FunctionNameToBind] = *FCSharpEnvironment::GetEnvironment().
					GetStruct<FBlueprintEnhancedInputActionBinding>(InBlueprintEnhancedInputActionBinding);   // :174-175

				const auto ObjectToBindTo = FCSharpEnvironment::GetEnvironment().GetObject<UObject>(InObjectToBindTo);

				const auto& EnhancedInputActionEventBinding = FoundObject->BindAction(
					InputAction, TriggerEvent, ObjectToBindTo, FunctionNameToBind);                          // :179
```

**调用上下文**
`FRegisterEnhancedInputComponent.cpp:221 .Function("BindAction", BindActionImplementation)` → `Script/UE/Library/EnhancedInputComponentImplementation.cs:24` → C# `UEnhancedInputComponent.BindAction(...)`。运行期（BeginPlay/PIE 绑定输入）在 GameThread。

**问题**
`FCSharpEnvironment::GetStruct<T>(IManagedHandle)` 的失败路径明确返回 `nullptr`：
```cpp
// Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.inl:126-130
template <typename T>
auto FCSharpEnvironment::GetStruct(const IManagedHandle InManagedHandle) const -> T*
{
	return StructRegistry != nullptr ? static_cast<T*>(StructRegistry->GetStruct(InManagedHandle)) : nullptr;
}
```
```cpp
// Source/UnrealCSharp/Private/Registry/FStructRegistry.cpp:78-81 → :50-55
void* FStructRegistry::GetStruct(const IManagedHandle InManagedHandle) { return GetAddress(InManagedHandle); }
void* FStructRegistry::GetAddress(const IManagedHandle InManagedHandle)
{
	const auto FoundStructAddress = ManagedHandle2StructAddress.Find(InManagedHandle);
	return FoundStructAddress != nullptr ? FoundStructAddress->Address : nullptr;   // 未知句柄 → nullptr
}
```
`GetRegistry` 未命中即 `nullptr`，`:175` 的 `*` 立即解引用 → 读出 **3** 个未初始化字段（`InputAction`/`TriggerEvent`/`FunctionNameToBind`；引擎定义 `EnhancedInputActionDelegateBinding.h:14-28` 恰为 3 个成员），随后 `:179` 把**垃圾指针当 UInputAction\* 传给 `BindAction`**。轻则内存泄漏/断言，重则把随机地址解释为 `UInputAction*` 并调用其方法 → 崩溃。注意紧邻的 `:171` 对 `GetObject` **做了**判空，说明作者知道有此模式，这里是遗漏。

**建议**
```cpp
const auto Binding = FCSharpEnvironment::GetEnvironment()
    .GetStruct<FBlueprintEnhancedInputActionBinding>(InBlueprintEnhancedInputActionBinding);
if (Binding == nullptr) { return InvalidManagedHandle; }
const auto [InputAction, TriggerEvent, FunctionNameToBind] = *Binding;
```
并顺带在 `GetStruct` 上加 `ensure(InManagedHandle != InvalidManagedHandle)` 便于早暴露。

**验证方式**
`grep -n "GetStruct<" Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp` → 1 处（:174）；在 `:174` 前插 `check(Binding)` 跑 PIE 绑定流程；或从 C# 传 `default` 结构体引用（句柄 0）复现。

---

### P1

### [F-INT2-003] `FMatrix::SetColumn` 被注册成 `GetColumn`，C# 中不存在可写的 `SetColumn`（getter 名隐藏了 mutator）

- **类别**: Bug
- **严重度**: **P3**(命名/API 形状)
- **复核结论**: 部分确认（偏差：**后果 1 与后果 3 被生成产物证伪**）—— 代码事实（`:76-79` 第二条把 `SetColumn` 写成 `GetColumn`）与"应为 `SetColumn`"的结论成立
- **可达性**: 活跃
- **复核证据**: ①`FRegisterMatrix.cpp:76-79` 逐字吻合。②`FClassBuilder.cpp:83-91 GetFunctionImplementationName` 生成的 `GetColumn1` 是 **thunk 实现名**（`MethodBridge` 字典键），**不是 C# API 名**。③**生成地面真值**：`Script/UE/Proxy/Binding/Matrix.cs:574 public FVector GetColumn(int i)` ↔ `:590 public void GetColumn(int i, FVector Value)`（后者调 `__FMatrix_GetColumn1Implementation`，`:600`）→ C# 侧得到的是**一对同名重载**，`SetColumn` 的能力**完全可达**。④`FMatrix::SetColumn` 名字缺失属"命名错"，不产生运行时错值。
- **级别变动**: P2→P3（后果 1/3 证伪：能力可达、重载形状正常、无静默错误与数据损坏，仅剩命名不一致）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterMatrix.cpp:76-79`
- **函数**: `FClassBuilder::Function` 链于 `FRegisterMatrix::FRegisterMatrix()`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterMatrix.cpp:76-79
				.Function("GetColumn", BINDING_FUNCTION(&FMatrix::GetColumn,
				                                        TArray<FString>{"i"}))
				.Function("GetColumn", BINDING_FUNCTION(&FMatrix::SetColumn,
				                                        TArray<FString>{"i", "Value"}))
```

**调用上下文**
`FRegisterMatrix.cpp:17 TBindingClassBuilder<FMatrix>(NAMESPACE_BINDING)`；静态注册（`FRegisterMatrix.cpp:103`）→ `MethodBridge` 字典 → 源生成器为 `Script.CoreUObject.FMatrix` 产生 C# 成员。

**问题**
第二个条目的**名字写成了 `GetColumn`**（应为 `SetColumn`）。由于 `FClassBuilder.cpp:83-91` 用 `Algo::Count` 消歧，实际注册名为：

| C++ 目标 | thunk 实现名（`MethodBridge` 键） | **C# 可见名（生成产物实测）** |
|---|---|---|
| `&FMatrix::GetColumn` | `GetColumn` | `GetColumn(int i)` → `FVector` |
| `&FMatrix::SetColumn` | `GetColumn1` | `GetColumn(int i, FVector Value)` → `void` |

后果（**已按生成产物 `Script/UE/Proxy/Binding/Matrix.cs:574/:590` 校正**）：
1. **`FMatrix::SetColumn` 这个名字在 C# 中不存在**（能力本身可达，写 `GetColumn(i, value)` 即可；原报告"只能写 `GetColumn1(i, value)`"**证伪**）。
2. `GetColumn(int, FVector Value)` 返回 `void` 且带 `Value` 形参，读起来仍是"取值"，调用方/代码审查可能误判为无副作用；`FMatrix` 是值类型，通过句柄共享，改的是 C# 那侧持有的同一份矩阵。
3. 源生成器**按名字合并重载**，`GetColumn(i)` 与 `GetColumn(i,value)` 确实被生成成**一对同名重载**（原报告"不会被生成成一对重载 / API 形状不可恢复"**证伪**）。

**建议**
```cpp
.Function("GetColumn", BINDING_FUNCTION(&FMatrix::GetColumn, TArray<FString>{"i"}))
.Function("SetColumn", BINDING_FUNCTION(&FMatrix::SetColumn, TArray<FString>{"i", "Value"}))
```
并建立一条通用规则：**`Get*` 前缀只允许绑定 `const` 成员函数**，可在 `FClassBuilder::Function` 里加 `static_assert`/编辑器启动期 `ensureMsgf` 做启发式校验（见第 7 节建议）。

**验证方式**
`grep -n 'Function("GetColumn"' Source/UnrealCSharp/Private/Domain/Interop/FRegisterMatrix.cpp` → 2 处（:76、:78），目标函数分别为 `GetColumn`/`SetColumn`；`grep -rn 'SetColumn' Source/UnrealCSharp/Private/Domain/Interop` → 仅 :78 一处且被错误命名。修复后重新生成 C#，确认 `FMatrix.SetColumn` 存在。

---

### [F-INT2-004] `FPolyglotTextData::SetNativeCulture` 被注册成 `SetCategory1`；`IsMinimalPatch` 的 setter 重载也是同名错绑

- **类别**: Bug
- **严重度**: **P2**(功能错误)
- **复核结论**: 部分确认（偏差：**"C# 中存在 `SetCategory1`" 与 "本意是 `SetIsMinimalPatch`" 两项被证伪**）
- **可达性**: 活跃
- **复核证据**: ①`FRegisterPolyglotTextData.cpp:16-20`、`:40-44` 代码**逐字吻合**。②**引擎侧**：`Runtime/Core/Public/Internationalization/PolyglotTextData.h:41 SetCategory(const ELocalizedTextSourceCategory)`、**`:52 SetNativeCulture(const FString& InNativeCulture)`（确实存在）**、`:126 void IsMinimalPatch(const bool InIsMinimalPatch)`、`:132 bool IsMinimalPatch() const` —— 即 **setter 在引擎里本来就叫 `IsMinimalPatch(const bool)`**，绑定目标**没有错**，`:42` 只是 `ParamNames` 复制残留。③**生成地面真值**：`Script/UE/Proxy/Binding/PolyglotTextData.cs:37 public void SetCategory(ELocalizedTextSourceCategory InCategory)` ↔ `:61 public void SetCategory(FString InNativeCulture)`（**同名重载，不是 `SetCategory1`**）；`:247 public void IsMinimalPatch(bool InCulture)` ↔ `:259 public bool IsMinimalPatch()`（**形参个数正确，只有形参名错**）。
- **级别变动**: 无（P2 维持）。剩余真实危害＝`SetNativeCulture(string)` 在 C# 中不存在，而 `SetCategory(string)` 会**静默改 native culture**（错名指向错目标）＋ `IsMinimalPatch` 形参名谎称 `InCulture`
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterPolyglotTextData.cpp:16-20`、`40-44`
- **函数**: `FRegisterPolyglotTextData::FRegisterPolyglotTextData()`
- **置信度**: 中（"绑定名错"为高置信，函数指针所指的真实 UE 目标为推断）

**现状（代码事实）**
```cpp
// FRegisterPolyglotTextData.cpp:16-20
				.Function("SetCategory", BINDING_FUNCTION(&FPolyglotTextData::SetCategory,
				                                          TArray<FString>{"InCategory"}))
				.Function("GetCategory", BINDING_FUNCTION(&FPolyglotTextData::GetCategory))
				.Function("SetCategory", BINDING_FUNCTION(&FPolyglotTextData::SetNativeCulture,
				                                          TArray<FString>{"InNativeCulture"}))
				.Function("GetNativeCulture", BINDING_FUNCTION(&FPolyglotTextData::GetNativeCulture))
```
```cpp
// FRegisterPolyglotTextData.cpp:40-44
				.Function("IsMinimalPatch",
				          BINDING_OVERLOAD(void(FPolyglotTextData::*)(const bool), &FPolyglotTextData::IsMinimalPatch,
				                           TArray<FString>{"InCulture", "OutLocalizedString"}))
				.Function("IsMinimalPatch",
				          BINDING_OVERLOAD(bool(FPolyglotTextData::*)()const, &FPolyglotTextData::IsMinimalPatch))
```

**调用上下文**
`FRegisterPolyglotTextData.cpp:12 TBindingClassBuilder<FPolyglotTextData>`，静态注册（`:49`）。`FPolyglotTextData` 是本地化数据载体，通常被编辑器/本地化管线在 GameThread 使用。

**问题**
1. `:19` 把 `SetNativeCulture` 挂在名字 `SetCategory` 下（与 `:16` 重复）→ 按 `FClassBuilder.cpp:83-91` 生成注册名 **`SetCategory1`**：
   - C# 里 `SetNativeCulture(string)` **不存在**；
   - C# 里存在 `SetCategory1(string)`，参数名 `InNativeCulture`，但方法名谎称在设置 category。调用 `SetCategory1("en")` 会改 native culture —— **静默改错目标**，且与 `:16` 的 `SetCategory(ELocalizedTextSourceCategory)` 语义完全不同却共享前缀。
2. `:40-42` 的 `void(const bool)` 重载被命名为 `IsMinimalPatch`（读起来是查询），参数名数组 `{"InCulture","OutLocalizedString"}` 是从 `:37` `GetLocalizedString` 复制的残留，**与被绑函数毫无关系**；源生成器用 `ParamNames` 生成 C# 形参名（`TClassInfoBuilder`/`TFunctionInfo` 路径，见 `Source/UnrealCSharp/Public/Binding/Class/TClassBuilder.inl:38-42`、`Source/UnrealCSharpCore/Public/Binding/Function/TFunctionInfo.inl:18-36`），生成的签名会写成 `IsMinimalPatch1(bool InCulture, ...)` 之类，与实际 `bool` 参数不符。按命名规律，这里几乎可以确定本意是 `FPolyglotTextData::SetIsMinimalPatch`。**由于我无法在本机核对 UE 头文件（见第 5 节），"本意是 SetIsMinimalPatch"标为推断。**

**建议**
```cpp
.Function("SetNativeCulture", BINDING_FUNCTION(&FPolyglotTextData::SetNativeCulture,
                                               TArray<FString>{"InNativeCulture"}))
...
.Function("SetIsMinimalPatch",
          BINDING_OVERLOAD(void(FPolyglotTextData::*)(const bool), &FPolyglotTextData::SetIsMinimalPatch,
                           TArray<FString>{"InIsMinimalPatch"}))
.Function("IsMinimalPatch",
          BINDING_OVERLOAD(bool(FPolyglotTextData::*)()const, &FPolyglotTextData::IsMinimalPatch))
```

**验证方式**
`grep -n 'Function("SetCategory"' Source/UnrealCSharp/Private/Domain/Interop/FRegisterPolyglotTextData.cpp` → 2 处（:16、:19），第 2 处目标为 `SetNativeCulture`；`grep -n "InCulture\", \"OutLocalizedString" ...` → 命中 :38 与 :42，:42 为残留。修复后用生成器产物检查 `SetNativeCulture` / `SetIsMinimalPatch` 是否出现。

---

### [F-INT2-005] `FRotator::SetComponentForAxis` 被绑到只读的 `GetComponentForAxis`：C# 的"设置"不产生任何效果

- **类别**: Bug
- **严重度**: **P2**(功能错误)
- **复核结论**: 部分确认（偏差：**"C# 生成 2 个形参、多余实参被丢弃"一项被证伪**）
- **可达性**: 活跃
- **复核证据**: ①`FRegisterRotator.cpp:63-66` 代码**逐字吻合**（`:65` 目标确为 `&FRotator::GetComponentForAxis`）。②**引擎侧**：`Runtime/Core/Public/Math/Rotator.h:332 [[nodiscard]] T GetComponentForAxis(EAxis::Type Axis) const` 与 `:335 void SetComponentForAxis(EAxis::Type Axis, T Component)` 确实并存 → "应在 `:65` 绑 `SetComponentForAxis`"的结论成立。③**生成地面真值**：`Script/UE/Proxy/Binding/Rotator.cs:364 public double GetComponentForAxis(EAxis Axis)` ↔ `:380 public double SetComponentForAxis(EAxis Axis)` —— **C# 侧只有 1 个形参**（`ParamNames` 的第 2 项 `"Component"` 被生成器忽略，形参个数由**真实函数签名**决定，不是由 `ParamNames` 长度决定）。故原文"`r.SetComponentForAxis(EAxis.Y, 90)` 编译通过"**证伪**；真实失效形态是 `r.SetComponentForAxis(EAxis.Y)` 只**返回当前分量值**、不修改任何东西。
- **级别变动**: 无（P2 维持）。C# 无法表达"设成 90"，因此不存在静默写坏数据；危害＝`FRotator::SetComponentForAxis` 完全未暴露 + 该名字被一个返回 `double` 的 getter 占用
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterRotator.cpp:63-66`
- **函数**: `FRegisterRotator::FRegisterRotator()`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterRotator.cpp:63-66
				.Function("GetComponentForAxis", BINDING_FUNCTION(&FRotator::GetComponentForAxis,
				                                                  TArray<FString>{"Axis"}))
				.Function("SetComponentForAxis", BINDING_FUNCTION(&FRotator::GetComponentForAxis,
				                                                  TArray<FString>{"Axis", "Component"}))
```

**调用上下文**
`FRegisterRotator.cpp:27 TBindingClassBuilder<FRotator>(NAMESPACE_BINDING)` → 静态注册（`:98`）→ C# `FRotator` 成员。

**问题**
第二个条目的目标函数是 **`GetComponentForAxis`（const 取值）**，却注册为 `SetComponentForAxis` 并声明两个参数名。UE 的 `FRotator` 同时有 `double GetComponentForAxis(EAxis::Type) const` 与 `void SetComponentForAxis(EAxis::Type, double)`，这里显然是复制粘贴时忘了改函数名。后果：

- C# 侧 `r.SetComponentForAxis(EAxis.Y, 90)` 编译通过，**运行时不修改任何东西**（`GetComponentForAxis` 是 const，返回值被丢弃），**静默失效**——这是本批次里最"安静"的一类错误；
- 参数名列表是 2 项而真实函数只有 1 参，源生成器产出的 C# 形参个数与真实调用不一致（多出的实参被 C 调用约定丢弃，参照 F-INT2-006 的机制分析）；
- 真正的 `FRotator::SetComponentForAxis` 完全没有暴露。

**建议**
```cpp
.Function("SetComponentForAxis", BINDING_FUNCTION(&FRotator::SetComponentForAxis,
                                                  TArray<FString>{"Axis", "Component"}))
```

**验证方式**
`grep -n "SetComponentForAxis" Source/UnrealCSharp/Private/Domain/Interop/FRegisterRotator.cpp` → 仅 :65 一处且 `&FRotator::GetComponentForAxis`。C# 单测：`var r = new FRotator(); r.SetComponentForAxis(EAxis.Y, 90); Assert(r.Yaw == 90);` —— 当前必失败。

---

### [F-INT2-006] `FRotator::Equals` 被绑到 `IsNearlyZero`，真实的 `Equals(R, Tolerance)` 未暴露（原标题所称"多余形参被静默忽略"**已证伪**，见下）

- **类别**: Bug / API 形状
- **严重度**: **P2**(API 形状/隐患)
- **复核结论**: 部分确认（偏差：**"参数名个数（2）多于真实参数（1）→ 生成的 `Tolerance` 形参被静默忽略"整体被证伪**；错绑本身成立）
- **可达性**: 活跃
- **复核证据**: ①`FRegisterRotator.cpp:43-47` 代码**逐字吻合**（`:46 .Function("Equals", BINDING_FUNCTION(&FRotator::IsNearlyZero, …))`）。②**引擎侧**：`Math/Rotator.h:234 [[nodiscard]] bool Equals(const TRotator<T>& R, T Tolerance = UE_KINDA_SMALL_NUMBER) const` —— 真实 `Equals` 确为 **2 参数**，`:46` 绑到 `IsNearlyZero`（`:213 [[nodiscard]] bool IsNearlyZero(T Tolerance = UE_KINDA_SMALL_NUMBER) const`，**1 参数**）属真实错绑。③**生成地面真值**：`Script/UE/Proxy/Binding/Rotator.cs:168 public bool IsNearlyZero(double Tolerance = 0.0001)`、**`:196 public bool Equals(double Tolerance = 0.0001)`** —— C# 侧 `Equals` 只有 **1 个形参**，`ParamNames` 里的 `"R"` 被丢弃，**不存在"第二个实参被静默忽略"**；`a.Equals(b, tol)` 在 C# 中**根本无法编译**（fail-fast），故原文最重的危害路径不成立。
- **级别变动**: **P1→P2**（证伪后危害降级为"+C# `Equals(double)` 与 `IsNearlyZero(double)` 语义完全重复、且真实 `FRotator::Equals(R,Tolerance)` 未暴露"的 API 形状缺陷；无静默错值）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterRotator.cpp:43-47`
- **函数**: `FRegisterRotator::FRegisterRotator()`
- **置信度**: 高（名称/目标错配为代码直读；"多出的实参被丢弃"一节**已由生成产物 `Rotator.cs:196` 证伪**，见复核证据）

**现状（代码事实）**
```cpp
// FRegisterRotator.cpp:43-47
				.Function("IsNearlyZero", BINDING_FUNCTION(&FRotator::IsNearlyZero,
				                                           TArray<FString>{"Tolerance"}, KINDA_SMALL_NUMBER))
				.Function("IsZero", BINDING_FUNCTION(&FRotator::IsZero))
				.Function("Equals", BINDING_FUNCTION(&FRotator::IsNearlyZero,
				                                     TArray<FString>{"R", "Tolerance"}, KINDA_SMALL_NUMBER))
```

**调用上下文**
同 F-INT2-005，静态注册路径；`FRotator.Equals` 会被 C# 用户/生成代码当作值相等判断使用（对比 `FRegisterTransform.cpp:128` 用 `&FTransform::Equals`、`FRegisterQuat.cpp:71` 用 `&FQuat::Equals`、`FRegisterVector.cpp:165` 用 `&FVector::Equals`，均为正确写法，**只有 FRotator 这一处是错的**）。

**问题**
1. 目标函数错误：`FRotator::Equals(const FRotator&, double Tolerance)` 未暴露；`Equals` 这个名字被 `IsNearlyZero(double Tolerance)` 占用。C# `a.Equals(b, tol)` 会**完全忽略 `b`**，只判断 `a` 本身是否接近零 → 逻辑错误且极易漏测。
2. 参数名个数不匹配：`ParamNames` 是 2 项，被绑函数只有 1 参。该元数据经 `TClassBuilder.inl:41 SetParamNames` 交给 `FFunctionInfo`，由源生成器用于生成 C# 形参（`TFunctionInfo.inl:18-36`）。C# 侧会生成两个形参并在调用点传入两个实参，而 thunk 只读取第 1 个 → 第二个参数（用户以为的 `Tolerance`）被丢弃，实际用的是默认 `KINDA_SMALL_NUMBER`。具体丢弃行为取决于生成器是否按 `ParamNames` 长度生成形参，**这一点我未在生成器中逐行确认，故对"实际丢弃"标 置信度 中**；但"元数据与真实函数不一致"是确定的。

**建议**
```cpp
.Function("Equals", BINDING_FUNCTION(&FRotator::Equals,
                                     TArray<FString>{"R", "Tolerance"}, KINDA_SMALL_NUMBER))
```
（保留 `IsNearlyZero` 的独立绑定，二者本就是不同语义。）

**验证方式**
`grep -n "FRotator::" Source/UnrealCSharp/Private/Domain/Interop/FRegisterRotator.cpp` 比对每行"名字/目标"；C# 用例 `new FRotator(0,10,0).Equals(new FRotator(0,11,0))` 应为 false，当前实现会因 `IsNearlyZero` 语义返回不同结果（视角度而定）。

---

### [F-INT2-007] `FSoftObjectPath::GetAssetPathName` 被绑到 `GetAssetName`：返回的是短资源名而非包路径

- **类别**: Bug
- **严重度**: **P1**(功能错误)
- **复核结论**: 部分确认（偏差：**"`GetAssetPathName` 与 `GetAssetName` 是 UE 里语义不同的两个函数"在 UE 5.6 不成立**；错绑本身与"返回短名而非包路径"的结果成立）
- **可达性**: 活跃
- **复核证据**: ①`FRegisterSoftObjectPath.cpp:34` 与 `:47` 代码**逐字吻合** —— `:34 .Function("GetAssetPathName", BINDING_FUNCTION(&FSoftObjectPath::GetAssetName))`。②**引擎侧**：`Runtime/CoreUObject/Public/UObject/SoftObjectPath.h` 中 **不存在任何 `AssetPathName` 标识符**（grep `AssetPathName` → 0 命中）；UE 5.6 只有 `:209 FTopLevelAssetPath GetAssetPath() const`、`:266 FString GetAssetName() const`、`:253 FString GetLongPackageName() const`。UE<5.1 才有 `GetAssetPathName`/`SetAssetPathName`（FName 版）—— 旁证：同文件 `:36 SetAssetPathName` 被正确门控于 `UE_F_SOFT_OBJECT_PATH_SET_ASSET_PATH_NAME`（`UEVersion.h:34` = `!UE_VERSION_START(5,1,0)`），而 `:34 GetAssetPathName` **没有跟随门控**。③**生成地面真值**：`Script/UE/Proxy/Binding/SoftObjectPath.cs:71 public FString GetAssetPathName()` 与 `:131 public FString GetAssetName()` —— 两个名字指同一个 native 函数，`GetAssetPathName()` 返回短名。
- **级别变动**: 无（P1 维持）。危害＝C# 方法名承诺"包路径"却静默返回叶子名（结果全错但不报错），且与 `GetAssetName()` 完全重复；正确修法是删除 `:34` 或改绑 `&FSoftObjectPath::GetAssetPath`
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterSoftObjectPath.cpp:34`、`47`
- **函数**: `FRegisterSoftObjectPath::FRegisterSoftObjectPath()`
- **置信度**: 高（名称/目标错配直读）；中（"两者返回值语义差别"依赖 UE 版本，见下）

**现状（代码事实）**
```cpp
// FRegisterSoftObjectPath.cpp:32-47
				.Function("ToString", BINDING_OVERLOAD(FString(FSoftObjectPath::*)()const, &FSoftObjectPath::ToString,
				                                       EFunctionInteract::New))
				.Function("GetAssetPathName", BINDING_FUNCTION(&FSoftObjectPath::GetAssetName))     // :34
...
				.Function("GetAssetName", BINDING_FUNCTION(&FSoftObjectPath::GetAssetName))         // :47
```

**调用上下文**
`FRegisterSoftObjectPath.cpp:17 TBindingClassBuilder<FSoftObjectPath>`，静态注册（`:82`）。资产路径解析是加载流程的热点（`ResolveObject`、`GetLongPackageName` 同在此文件 :45/:55）。

**问题**
`GetAssetPathName` 与 `GetAssetName` 是 UE 里**语义不同的两个函数**：前者返回资源所在的包路径名（UE 5.1+ 为 `FTopLevelAssetPath`，更早版本为 `FName`），后者返回资源的叶子名。本行把 `GetAssetName` 同时挂在两个名字下 → C# 的 `GetAssetPathName()` 拿不到包路径，只能拿到短名。调用方若用它的返回值再去拼路径/做 `FSoftObjectPath` 构造，会得到错误路径（静默失败/加载失败），而**绝不会崩溃** —— 属于典型的"结果全错但不报错"。

**建议**
按 UE 版本正确绑定，例如
```cpp
#if UE_VERSION_START(5, 1, 0)
.Function("GetAssetPathName", BINDING_FUNCTION(&FSoftObjectPath::GetAssetPathName))
#else
.Function("GetAssetPathName", BINDING_FUNCTION(&FSoftObjectPath::GetAssetPathName,
                                               TArray<FString>{}, ...))   // FName 版本
#endif
```
并在 `FRegisterSoftObjectPath.cpp` 顶部把该差异加入 `UEVersion.h` 门控（本文件已使用 6 个 `UE_F_SOFT_OBJECT_PATH_*` 宏，加一个即可）。

**验证方式**
`grep -n "GetAssetName\|GetAssetPathName" FRegisterSoftObjectPath.cpp` → :34 与 :47 都指向 `GetAssetName`。C# 用例：对 `/Game/Foo/Bar.Bar` 调用 `GetAssetPathName()`，期望包含 `/Game/Foo/`；当前只返回 `Bar`。

---

### [F-INT2-008] `UWorld::SpawnActor` 对 Transform 地址与 Class 均未判空

- **类别**: Bug / 未定义行为
- **严重度**: **P1**(崩溃)
- **复核结论**: 确认 —— 代码事实、调用上下文、类别、严重度 P1 全部成立
- **可达性**: 活跃
- **复核证据**: ①`FRegisterWorld.cpp:125-141` **逐字吻合**：`:127 GetObject<UClass>(InClass)`、`:129-130 GetAddress<UScriptStruct, FTransform>(InTransform)`、`:135-139 FoundWorld->SpawnActor<AActor>(FoundClass, *FoundTransform, FoundActorSpawnParameters != nullptr ? *FoundActorSpawnParameters : FActorSpawnParameters())`、`:141 Bind(Actor)`。②`:137-139` 对 `FoundActorSpawnParameters` **确实做了判空**（反证 `:127`/`:129` 两处是遗漏）——原文此推断成立。③失败路径 `GetAddress<UScriptStruct,T>` 返回 nullptr：`FCSharpEnvironment.inl:100-109`（`if (const auto FoundStruct = …GetAddress(…)) return FoundStruct; … return nullptr;`）＋ `FStructRegistry.cpp:50-55`。④`:150 .Function("SpawnActor", SpawnActorImplementation)` 吻合。
- **级别变动**: 无（P1 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterWorld.cpp:125-141`
- **函数**: `FRegisterWorld::SpawnActorImplementation(IManagedHandle, IManagedHandle, IManagedHandle, IManagedHandle)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterWorld.cpp:125-141
			if (const auto FoundWorld = FCSharpEnvironment::GetEnvironment().GetObject<UWorld>(InManagedHandle))
			{
				const auto FoundClass = FCSharpEnvironment::GetEnvironment().GetObject<UClass>(InClass);

				const auto FoundTransform = FCSharpEnvironment::GetEnvironment().GetAddress<UScriptStruct, FTransform>(
					InTransform);

				const auto FoundActorSpawnParameters = FCSharpEnvironment::GetEnvironment().GetBinding<
					FActorSpawnParameters>(InActorSpawnParameters);

				const auto Actor = FoundWorld->SpawnActor<AActor>(FoundClass,
				                                                  *FoundTransform,
				                                                  FoundActorSpawnParameters != nullptr
					                                                  ? *FoundActorSpawnParameters
					                                                  : FActorSpawnParameters());
```

**调用上下文**
`FRegisterWorld.cpp:150 .Function("SpawnActor", SpawnActorImplementation)` → `Script/UE/Library/WorldImplementation.cs:13` → `Script/UE/CoreUObject/World.cs:9 UWorld.SpawnActor<T>(UClass, FTransform, FActorSpawnParameters = null)`。其中 `FActorSpawnParameters` 允许为 `null`（`World.cs:9`），代码对**它**做了判空（`FRegisterWorld.cpp:137-139`）—— 这反证其余两处是遗漏。

**问题**
- `GetAddress<UScriptStruct, FTransform>(InTransform)` 在句柄未注册时返回 `nullptr`（`FCSharpEnvironment.inl:60-73 TGetAddress<UScriptStruct,T>::operator()`；`FStructRegistry::GetAddress` 失败返回 0，见 F-INT2-002 引用的 `FStructRegistry.cpp:50-55`），`:136` 立即 `*FoundTransform` → 空指针解引用。
- `FoundClass` 未判空即传入 `SpawnActor`（`:135`）：FTransform 是值类型、C# 侧可能传 `default(FTransform)`（句柄 0），`InClass` 也可能来自未初始化变量。两种都会在引擎内部空指针解引用或触发 ensure 后崩溃。
- 附带：`SpawnActor` 在包装/独立服务器无世界时可能返回 null，`:141 Bind(Actor)` 会把 null 交回 C#，C# `WorldImplementation.cs:15` 已处理返回 null，这一处**没有问题**。

**建议**
```cpp
const auto FoundTransform = FCSharpEnvironment::GetEnvironment()
    .GetAddress<UScriptStruct, FTransform>(InTransform);
if (FoundClass == nullptr || FoundTransform == nullptr) { return InvalidManagedHandle; }
```

**验证方式**
`grep -n "GetAddress<UScriptStruct" Source/UnrealCSharp/Private/Domain/Interop/FRegisterWorld.cpp` → 1 处（:129）。C# 用例：`World.SpawnActor<AActor>(SomeClass, default)` 当前应崩。

---

### [F-INT2-009] `Unreal.CreateWidget` 对 Outer 与 Class 均未判空；五路 `IsA` 全不匹配时静默返回 null

- **类别**: Bug / 未定义行为
- **严重度**: **P1**(崩溃)
- **复核结论**: 确认 —— 代码事实、调用上下文、类别、严重度 P1 全部成立
- **可达性**: 活跃
- **复核证据**: ①`FRegisterUnreal.cpp:135-166` **逐字吻合**：`:138 GetObject<UObject>(InOwningObject)` 与 `:140 GetObject<UClass>(InUserWidgetClass)` 均无判空即 `:144 OwningObject->IsA(UWidget::StaticClass())`；五路 `IsA` 精确位于 `:144/:148/:152/:156/:160`（原文验证方式所列一致）；`:165 Bind(UserWidget)` 在五路全不匹配时收到 `nullptr`。②`GetObject<T>` 失败返回 nullptr：`FCSharpEnvironment.inl:111-115`（`… : nullptr`）**逐字吻合**。③`:185 .Function("CreateWidget", CreateWidgetImplementation)` 吻合（原文 §6.6 写 `:185` 正确）。
- **级别变动**: 无（P1 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterUnreal.cpp:135-166`
- **函数**: `FRegisterUnreal::CreateWidgetImplementation(IManagedHandle, IManagedHandle)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterUnreal.cpp:135-166
		static IManagedHandle CreateWidgetImplementation(const IManagedHandle InOwningObject,
		                                                 const IManagedHandle InUserWidgetClass)
		{
			const auto OwningObject = FCSharpEnvironment::GetEnvironment().GetObject<UObject>(InOwningObject);

			const auto Class = FCSharpEnvironment::GetEnvironment().GetObject<UClass>(InUserWidgetClass);

			UUserWidget* UserWidget = nullptr;

			if (OwningObject->IsA(UWidget::StaticClass()))                       // :144  未判空
			{
				UserWidget = CreateWidget(Cast<UWidget>(OwningObject), Class);
			}
			else if (OwningObject->IsA(UWidgetTree::StaticClass()))               // :148
			...
			else if (OwningObject->IsA(UWorld::StaticClass()))
			{
				UserWidget = CreateWidget(Cast<UWorld>(OwningObject), Class);
			}

			return FCSharpEnvironment::GetEnvironment().Bind(UserWidget);          // :165  UserWidget 可能为 nullptr
		}
```

**调用上下文**
`FRegisterUnreal.cpp:185 .Function("CreateWidget", CreateWidgetImplementation)` → `Script/UE/Library/UnrealImplementation.cs:53`（`__Unreal_CreateWidgetImplementation(nint OwningObject, nint UserWidgetClass)`，与本 C++ 签名 2 个 `IManagedHandle` **逐参数一致**）→ C# `Unreal.CreateWidget<T>(...)`。由 UMG 逻辑在 GameThread 调用。

**问题**
1. `FCSharpEnvironment::GetObject<T>()` 未命中即返回 `nullptr`（`FCSharpEnvironment.inl:111-115`：`ObjectRegistry != nullptr ? Cast<T>(...) : nullptr`），`:144` 立即调用 `OwningObject->IsA(...)` → **空指针成员函数调用崩溃**。C# 侧传 `null` 或不存在的 UObject 包装即可触发。
2. `Class` 未判空即传给 `CreateWidget(...)`；`CreateWidget<UUserWidget>(Outer, nullptr)` 在引擎里会解引用 null 的 UClass。
3. 五路 `IsA` 覆盖 `UWidget`/`UWidgetTree`/`APlayerController`/`UGameInstance`/`UWorld`；**其他 owner 类型（例如任意 AActor、UGameInstanceSubsystem）落到末尾，`UserWidget` 保持 `nullptr`**，`:165 Bind(nullptr)` → C# 收到 null/无效句柄，调用方得到"Widget 创建了但为 null"，**无任何日志或断言**。这类静默失败很难定位。

**建议**
```cpp
if (OwningObject == nullptr || Class == nullptr) { return InvalidManagedHandle; }
...
if (UserWidget == nullptr)
{
    UE_LOG(LogUnrealCSharp, Warning,
           TEXT("CreateWidget: unsupported owning object type '%s'"), *OwningObject->GetClass()->GetName());
    return InvalidManagedHandle;
}
```

**验证方式**
`grep -n "OwningObject->" FRegisterUnreal.cpp` → :144/148/152/156/160 五处均无前置判空。C# 用例：`Unreal.CreateWidget<UMyWidget>(null, typeof(UMyWidget))`。

---

### [F-INT2-010] `BindAction` 对 `ObjectToBindTo` 未判空即解引用取 Class

- **类别**: Bug / 未定义行为
- **严重度**: **P2**(崩溃)
- **复核结论**: 确认 —— 代码事实、调用上下文、类别、严重度 P2 全部成立
- **可达性**: 活跃
- **复核证据**: ①`FRegisterEnhancedInputComponent.cpp:177` = `const auto ObjectToBindTo = FCSharpEnvironment::GetEnvironment().GetObject<UObject>(InObjectToBindTo);`、`:186` = `BindActionFunction(ObjectToBindTo->GetClass(),` —— **两处行号与代码逐字吻合**。②`:179-184` 更早把该指针传入引擎，故 `:186` 是第二次崩点（原文表述准确）。③对照项 `FRegisterInputComponent.cpp:283-319 BindFunction` 开头确有 `if (InClass == nullptr || InFunctionName == nullptr)`（`FRegisterEnhancedInputComponent.cpp:71-74` 同构）→ "模式已知"的推断成立。
- **级别变动**: 无（P2 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp:177`、`186`
- **函数**: `FRegisterEnhancedInputComponent::BindActionImplementation(...)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterEnhancedInputComponent.cpp:177-187
				const auto ObjectToBindTo = FCSharpEnvironment::GetEnvironment().GetObject<UObject>(InObjectToBindTo);

				const auto& EnhancedInputActionEventBinding = FoundObject->BindAction(
					InputAction, TriggerEvent, ObjectToBindTo, FunctionNameToBind);

				BindActionFunction(ObjectToBindTo->GetClass(),                                     // :186
				                   FCSharpEnvironment::GetEnvironment().GetString<FName>(InFunctionNameToBind));
```

**调用上下文**
同 F-INT2-002。`ObjectToBindTo` 从 C# 传入，`Script/UE/Library/EnhancedInputComponentImplementation.cs:19` 对其无任何约束（`nint InObjectToBindTo`）。

**问题**
`GetObject<UObject>` 未命中返回 `nullptr`，`:186` 直接 `->GetClass()` → 崩溃。注意 `:179` 更早把该指针传进引擎（引擎内部大概率也会崩或 ensure），所以 `:186` 只是"第二次崩点"。相比之下 `FRegisterInputComponent.cpp:286`（其 `BindFunction`）对 `InClass` **做了**判空，说明模式已知。

**建议**
在 `:177` 之后立即 `if (ObjectToBindTo == nullptr) { return InvalidManagedHandle; }`，与 F-INT2-002 的 Binding 判空合并为同一段前置校验。

**验证方式**
`grep -n "ObjectToBindTo" Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp` → :177/:179/:182/:186。

---

### [F-INT2-011] `RemoveBinding` 对 `GetBinding` 结果未判空；重复解绑/失效句柄即解引用空指针

- **类别**: Bug / 未定义行为
- **严重度**: **P1**(崩溃)
- **复核结论**: 确认 —— 代码事实、调用上下文、类别、严重度 P1 全部成立
- **可达性**: 活跃
- **复核证据**: ①`FRegisterEnhancedInputComponent.cpp:204-215` **逐字吻合**，`:213` = `FoundObject->RemoveBinding(*EnhancedInputActionEventBinding);` 无判空。②`GetBinding<T>` 失败返回 nullptr：`FBindingRegistry.inl:6-12`（`return FoundValue != nullptr ? static_cast<T*>(FoundValue->AddressWrapper->Value) : nullptr;`）**逐字吻合**。③对照项 `FRegisterWorld.cpp:132-139` 对 `GetBinding` 结果做了 `!= nullptr` 判断 —— **该对照成立**（原文引用准确）。④`:222 .Function("RemoveBinding", RemoveBindingImplementation)` 吻合。
- **级别变动**: 无（P1 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp:204-215`
- **函数**: `FRegisterEnhancedInputComponent::RemoveBindingImplementation(IManagedHandle, IManagedHandle)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterEnhancedInputComponent.cpp:204-215
		static void RemoveBindingImplementation(const IManagedHandle InManagedHandle,
		                                        const IManagedHandle InEnhancedInputActionEventBinding)
		{
			if (const auto FoundObject = FCSharpEnvironment::GetEnvironment().GetObject<UEnhancedInputComponent>(
				InManagedHandle))
			{
				const auto EnhancedInputActionEventBinding = FCSharpEnvironment::GetEnvironment().GetBinding<
					FEnhancedInputActionEventBinding>(InEnhancedInputActionEventBinding);

				FoundObject->RemoveBinding(*EnhancedInputActionEventBinding);        // :213
			}
		}
```

**调用上下文**
`FRegisterEnhancedInputComponent.cpp:222 .Function("RemoveBinding", RemoveBindingImplementation)` → `Script/UE/Library/EnhancedInputComponentImplementation.cs:35`。典型调用时机是 `EndPlay`/`Dispose`，即**解绑发生在其绑定时机之后很久，句柄很可能已失效**。

**问题**
`FCSharpEnvironment::GetBinding<T>(IManagedHandle)` 明确在未命中时返回 `nullptr`：
```cpp
// Source/UnrealCSharp/Public/Registry/FBindingRegistry.inl:6-12
template <typename T>
auto FBindingRegistry::GetBinding(const IManagedHandle InManagedHandle)
{
	const auto FoundValue = ManagedHandle2BindingAddress.Find(InManagedHandle);
	return FoundValue != nullptr ? static_cast<T*>(FoundValue->AddressWrapper->Value) : nullptr;
}
```
`:213` 的 `*` 解引用 null → 崩溃。触发场景真实且常见：**C# 调用两次 `RemoveBinding`、或在绑定对象已销毁后再解绑**。同文件 `FRegisterWorld.cpp:132-139` 对 `GetBinding` 结果做了 `!= nullptr` 判断，可以直接对照。

**建议**
```cpp
if (EnhancedInputActionEventBinding == nullptr) { return; }
FoundObject->RemoveBinding(*EnhancedInputActionEventBinding);
```

**验证方式**
C# 用例：同一个 binding 连续调用 `RemoveBinding` 两次（当前第二次应崩）；或在 `Script/UE/Library/EnhancedInputComponentImplementation.cs:35` 前加判空后回归。

---

### [F-INT2-012] `UInputComponent` 六种绑定路径对委托绑定对象与目标对象均未判空

- **类别**: Bug / 未定义行为
- **严重度**: **P2**(崩溃)
- **复核结论**: 确认 —— 代码事实、调用上下文、类别、严重度 P2 全部成立
- **可达性**: 活跃
- **复核证据**: ①`FRegisterInputComponent.cpp:50-62` **逐字吻合**：`:53-54 GetObject<T>(InInputDelegateBinding)`、`:56 GetObject<UObject>(InObjectToBindTo)`、`:58 InputDelegateBinding->BindToInputComponent(FoundObject, ObjectToBindTo);`、`:60 InFunction(ObjectToBindTo->GetClass(),` —— 两处解引用均无判空。②模板定义位于 `:42-63`（`template <typename T> static void BindImplementation(...)`），六个导出经 `:70/:106/:134/:162/:197/:252` 实例化（原文验证方式所列行号吻合）。③`:334-339` 六个 `.Function` 导出吻合。
- **级别变动**: 无（P2 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterInputComponent.cpp:50-62`
- **函数**: `FRegisterInputComponent::BindImplementation<T>(...)`（被 `:65/:101/:129/:157/:192/:247` 六处实例化）
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterInputComponent.cpp:50-62
			if (const auto FoundObject = FCSharpEnvironment::GetEnvironment().GetObject<UInputComponent>(
				InManagedHandle))
			{
				const auto InputDelegateBinding = FCSharpEnvironment::GetEnvironment().GetObject<T>(
					InInputDelegateBinding);

				const auto ObjectToBindTo = FCSharpEnvironment::GetEnvironment().GetObject<UObject>(InObjectToBindTo);

				InputDelegateBinding->BindToInputComponent(FoundObject, ObjectToBindTo);   // :58

				InFunction(ObjectToBindTo->GetClass(),                                      // :60
				           FCSharpEnvironment::GetEnvironment().GetString<FName>(InFunctionNameToBind));
			}
```

**调用上下文**
六个导出（`FRegisterInputComponent.cpp:334-339`）`BindAction`/`BindAxis`/`BindAxisKey`/`BindKey`/`BindTouch`/`BindVectorAxis` 全部转发到此模板；C# 入口 `Script/UE/Library/InputComponentImplementation.cs:20/29/38/47/56/65`。运行于 GameThread 的组件初始化阶段。

**问题**
`GetObject<T>(InInputDelegateBinding)` 与 `GetObject<UObject>(InObjectToBindTo)` 都可能为 `nullptr`；`:58` 与 `:60` 均无判空。`:58` 的 `InputDelegateBinding` 是 C# 传入的 `UInputActionDelegateBinding` 等对象——**若 C# 侧传错类型或对象已回收，这里是空指针成员调用**。`:60` 的 `ObjectToBindTo` 与 F-INT2-010 同型。

**建议**
```cpp
if (InputDelegateBinding == nullptr || ObjectToBindTo == nullptr) { return; }
```

**验证方式**
`grep -n "InputDelegateBinding->\|ObjectToBindTo->" Source/UnrealCSharp/Private/Domain/Interop/FRegisterInputComponent.cpp` → :58、:60（模板体内）；六处实例化位置见 `:70/:106/:134/:162/:197/:252`。

---

### [F-INT2-013] `EKeys` 的 342 个 `static const FKey` 全部用可写的 `BINDING_PROPERTY`，而其余文件同类静态量用 `BINDING_READONLY_PROPERTY`

- **类别**: Bug / 未定义行为 / 一致性
- **严重度**: **P1**(数据损坏)
- **复核结论**: 确认 —— 代码事实、机制、类别、严重度 P1 全部成立（并经**生成产物独立证实**）
- **可达性**: 活跃
- **复核证据**: ①`FRegisterKeys.cpp:13-15` **逐字吻合**；`:14-411` 共 **342** 条 `.Property`（逐行清点），唯一例外 `:117`（`BINDING_PROPERTY(&EKeys::Equals, EPropertyInteract::New)`）✓。②`BindingMacro.h:251 BINDING_PROPERTY(...) → GET, SET, INFO, ##__VA_ARGS__`、`:257 BINDING_READONLY_PROPERTY(...) → GET, nullptr, INFO, ##__VA_ARGS__` —— **逐字吻合**（原文行号正确）。③**生成地面真值（决定性质证）**：`Script/Game/Proxy/Binding/EKeys.cs:16-41` 的 `AnyKey` 同时生成 `get`（调 `__EKeys_GetAnyKeyImplementation`）与 **`set`**（调 `__EKeys_SetAnyKeyImplementation`，`:38`）两个块；`:8980-9010` 连 `Virtual_Accept` 也是可写属性；`:9034-9057` 连 `Invalid` 都可写 → **"全部 342 条变成可写属性"在生成产物侧得到实证**。④**计数口径**：342 = 插件 `.Property` 总数；因 7 处 `#if` 在 UE 5.6 下关闭（`:202/:217/:315/:340/:343/:374/:377`），**UE 5.6 下实际生效 335 条**；"其余文件用只读"的对照表正确。
- **级别变动**: 无（P1 维持）。触发条件是 C# 写 `EKeys.X = …`（生成的是 setter，编译器不报错）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterKeys.cpp:14-411`
- **函数**: `FRegisterKeys::FRegisterKeys()`
- **置信度**: 中（写路径由模板实现确定，但"落到只读段崩溃 / 落到可写段静默篡改全局键表"取决于链接布局，本机无法运行验证）

**现状（代码事实）**
```cpp
// FRegisterKeys.cpp:13-15
			TBindingClassBuilder<EKeys>(NAMESPACE_BINDING)
				.Property("AnyKey", BINDING_PROPERTY(&EKeys::AnyKey))
				.Property("MouseX", BINDING_PROPERTY(&EKeys::MouseX))
```
`BINDING_PROPERTY` 展开为 `GET, SET, INFO`（`Source/UnrealCSharp/Public/Macro/BindingMacro.h:251`），而 `BINDING_READONLY_PROPERTY` 展开为 `GET, nullptr, INFO`（同文件 `:257`）。本批次其余文件对静态常量一律用只读形式：

| 文件 | 写法 | 行号 |
|---|---|---|
| `FRegisterKeys.cpp` | `BINDING_PROPERTY` | :14-411（342 处，唯一例外 `:117` 多传 `EPropertyInteract::New`） |
| `FRegisterVector.cpp` | `BINDING_READONLY_PROPERTY` | :141-151 |
| `FRegisterVector2D.cpp` | `BINDING_READONLY_PROPERTY` | :103-105 |
| `FRegisterQuat.cpp` | `BINDING_READONLY_PROPERTY` | :69 |
| `FRegisterMatrix.cpp` | `BINDING_READONLY_PROPERTY` | :25 |
| `FRegisterTransform.cpp` | `BINDING_READONLY_PROPERTY` | :32 |
| `FRegisterFrameTime.cpp` | `BINDING_READONLY_PROPERTY` | :38 |
| `FRegisterColor.cpp` / `FRegisterLinearColor.cpp` | `BINDING_READONLY_PROPERTY` | :16-29 / :35-42 |

**调用上下文**
`FClassBuilder.inl:23-50 Property(...)` 为每个属性额外注册 `"Get"+Name` 与 `"Set"+Name` 两个 thunk；`Set` 的实现见下。

**问题**
对 `static const` 常量，`Set` 走的是**去掉 const 后直接赋值**的路径：
```cpp
// Source/UnrealCSharp/Public/Binding/Property/TPropertyBuilder.inl:102-108（TCompoundPropertyBuilder::Set，Class=void 分支）
	static auto Set(const IManagedHandle InManagedHandle, IN_BUFFER_SIGNATURE)
	{
		if constexpr (std::is_same_v<Class, void>)
		{
			*const_cast<std::remove_const_t<Result>*>(Member) = TPropertyValue<Result, Result>::Get(
				*(IManagedHandle*)IN_BUFFER);
		}
```
（`FKey` 是 UStruct，命中 `TIsUStruct` 特化 → `TCompoundPropertyBuilder`，`TPropertyBuilder.inl:211-216 / 416-421`；`Class=void` 分支由 `Result* Member` 系列特化提供，`:404-408`。）

后果：C# 侧 `EKeys.A` 变成**可写属性**。一旦 C# 代码写 `EKeys.A = someKey;`（编译器不会报错，因为生成的是 setter），就会通过 `const_cast` 写 `EKeys` 的 `static const FKey` 对象：
- 若链接器把该常量放入只读段（MSVC 对 `const` 且无动态初始化的对象通常如此；`FKey` 有非平凡构造，实际落在可写段的可能性更大）→ **写时访问违例**；
- 若落在可写段 → **静默篡改引擎全局按键表**，`EKeys::A` 此后不再等于"A 键"，所有依赖它的输入映射、序列化、比较全部错乱，且没有任何报错。

这是典型的"API 形状暗示了不该允许的操作"。`FRegisterKeys.cpp:117` 传 `EPropertyInteract::New` 只是为规避 C# 的 `object.Equals` 遮蔽，不能解释其余 341 处的可写性。

**建议**
1. 全部改为 `BINDING_READONLY_PROPERTY`：
```cpp
.Property("AnyKey", BINDING_READONLY_PROPERTY(&EKeys::AnyKey))
...
.Property("Equals", BINDING_READONLY_PROPERTY(&EKeys::Equals, EPropertyInteract::New))
```
2. 在 `TPropertyBuilder` 的 `Set` 里对 `Class == void && std::is_const_v<Result>` 的情形加 `static_assert` 或 `if constexpr` 编译期拒绝，从机制上杜绝同类误用（`const_cast` 去 const 后写入是 UB，任何情况下都不该出现）。

**验证方式**
`Select-String -Path FRegisterKeys.cpp -Pattern 'BINDING_PROPERTY\(&EKeys::' | Measure-Object` → 342；`Select-String -Pattern 'BINDING_READONLY_PROPERTY'` → 0。修复后重新生成 C#，确认 `EKeys.A` 只有 getter。

---

### [F-INT2-014] `FBox2D` 暴露 `(const FVector2D*, int32)` 构造：指针与计数完全由 C# 提供且不校验

- **类别**: 安全 / 未定义行为
- **严重度**: **P1**(越界读)
- **复核结论**: 确认 —— 代码事实、类别、严重度 P1 成立（置信度由"中"升为"高"，见下）
- **可达性**: 活跃
- **复核证据**: ①`FRegisterBox2D.cpp:22-23` **逐字吻合**：`.Constructor(BINDING_CONSTRUCTOR(FBox2D, const FVector2D*, const int32), TArray<FString>{"Points", "Count"})`；紧邻 `:24-25` 确有等价的 `const TArray<FVector2D>&` 版本（原文论据成立）。②**生成地面真值（解决"生成器如何 marshal 裸指针未逐行确认"的疑点）**：`Script/UE/Proxy/Binding/Box2D.cs` 与 `Script/UE/Proxy/Binding/Box2DImplementation.cs` 为该重载生成了独立构造函数与 thunk → **裸指针+`int32` 计数确实跨托管边界逐字传递**，因此 `Count` 与缓冲区长度之间无任何校验这一结论**成立**。
- **级别变动**: 无（P1 维持）。`Count` 由 C# 自由给定、指针亦由 C# 给定，越界读可稳定构造
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterBox2D.cpp:22-23`
- **函数**: `FRegisterBox2D::FRegisterBox2D()`
- **置信度**: 中（越界读取路径确定；"生成器如何 marshal 裸指针"未逐行确认）

**现状（代码事实）**
```cpp
// FRegisterBox2D.cpp:18-25
			TBindingClassBuilder<FBox2D>(NAMESPACE_BINDING)
				.Constructor(BINDING_CONSTRUCTOR(FBox2D, EForceInit))
				.Constructor(BINDING_CONSTRUCTOR(FBox2D, const FVector2D&, const FVector2D&),
				             TArray<FString>{"InMin", "InMax"})
				.Constructor(BINDING_CONSTRUCTOR(FBox2D, const FVector2D*, const int32),
				             TArray<FString>{"Points", "Count"})
				.Constructor(BINDING_CONSTRUCTOR(FBox2D, const TArray<FVector2D>&),
				             TArray<FString>{"Points"})
```

**调用上下文**
`FRegisterBox2D.cpp:62` 静态注册 → C# `FBox2D` 构造函数。**紧接着 `:24-25` 已经注册了等价的 `const TArray<FVector2D>&` 版本**——后者由框架管理元素数量，前者不需要。

**问题**
`FBox2D(const FVector2D* Points, int32 Count)` 在引擎内按 `Count` 遍历 `Points`（用于 `Init` 或计算包围盒）。本绑定把裸指针与计数一起交给 C#：
- 计数与缓冲区长度之间**没有任何关联校验**（`Count` 是独立的 `int32` 参数）；
- C# 侧若传 `Count` 大于实际元素数（或指针指向已释放/栈上的内存），引擎会**越界读取**，读到的垃圾数据被当作 `FVector2D` 用于比较/求极值，行为不可预测。

由于 `:24` 的 `TArray<FVector2D>` 版本功能等价且安全，这个裸指针重载是**纯粹的攻击面扩大**。

**建议**
删除裸指针重载（JNI/P-Invoke 场景下 `const T*` + `int32` 的绑定无法安全使用），只保留 `TArray<FVector2D>` 版本；若必须保留，改为在 thunk 内校验 `Count >= 0 && Count <= 某个上限`，并明确要求由 C# 侧持有缓冲区所有权直到构造完成。

**验证方式**
`grep -n "FVector2D\*, const int32" Source/UnrealCSharp/Private/Domain/Interop/FRegisterBox2D.cpp` → :22（全批次唯一一处裸指针构造）。删除后重新生成 C# 并确认 `FBox2D` 构造重载数从 4 降到 3。

---

### P2

### [F-INT2-015] `BindFunction` 里 `Function->AddToRoot()` 没有配对的 `RemoveFromRoot`（本批次唯二的"永久 root"点）

- **类别**: 内存/资源泄漏
- **严重度**: **P2**(泄漏/隐患)
- **复核结论**: 部分确认（偏差：**"本批次唯二的永久 root 点"的普查口径不完整**——全插件还有 2 处未配对 `AddToRoot` 不在本报告的对照表内）
- **可达性**: 活跃
- **复核证据**: ①`grep 'AddToRoot|RemoveFromRoot'`（工具 grep，路径 `Plugins/UnrealCSharp/Source`）→ **26 命中**，与原文一致 ✓。②两处目标行**逐字吻合**：`FRegisterInputComponent.cpp:312 Function->AddToRoot();`、`FRegisterEnhancedInputComponent.cpp:97 Function->AddToRoot();`。③对照表 6 行逐行核实通过（`FCSharpBind.cpp:530` ↔ `FCSharpFunctionRegister.cpp:61`/`FRegisterClass.cpp:23`；`FDynamicStructGenerator.cpp:249` ↔ `:380`；`FDynamicClassGenerator.cpp:393/419` ↔ `:533/540/893`；`FDelegateHelper.cpp:24` ↔ `:38`；`FMulticastDelegateHelper.cpp:25` ↔ `:39`）。④**偏差**：26 命中中 `AddToRoot` **调用点共 12 处**（下表只列 8 处），漏列 `FDynamicInterfaceGenerator.cpp:263`、`FDynamicEnumGenerator.cpp:201`（**这两处同样没有配对的 `RemoveFromRoot`**）、`FDynamicBlueprintExtensionScope.h:23`（有配对 `:42`）、`FRegisterObject.cpp:89`（对 C# 暴露的显式 API）。故"全插件未配对 `AddToRoot`"实为 **4 处**（含本批次 2 处）。后者属动态生成器范围（其他报告），但本报告表头写的是"全插件…配对情况"，口径需收窄或补行。
- **级别变动**: 无（P2 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterInputComponent.cpp:312`、`Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp:97`
- **函数**: `FRegisterInputComponent::BindFunction(UClass*, const FName*, const TFunction<void(UFunction*)>&)`、`FRegisterEnhancedInputComponent::BindFunction(...)`
- **置信度**: 中（`AddToRoot` 无误配对为确定；"是否构成真实泄漏"取决于该 UFunction 是否已被 `AddFunctionToFunctionMap` 保活，见下）

**现状（代码事实）**
```cpp
// FRegisterInputComponent.cpp:296-318（FRegisterEnhancedInputComponent.cpp:81-103 为逐行重复）
			const auto Function = NewObject<UFunction>(InClass, *InFunctionName, EObjectFlags::RF_Transient);

			Function->FunctionFlags = FUNC_BlueprintEvent;

			InProperty(Function);

			Function->Bind();

			Function->StaticLink(true);

			InClass->AddFunctionToFunctionMap(Function, *InFunctionName);

			Function->Next = InClass->Children;

			InClass->Children = Function;

			Function->AddToRoot();                                          // :312 / :97
```

**调用上下文**
仅从 `BindActionFunction`（`FRegisterEnhancedInputComponent.cpp:108`）与 `BindAction/BindAxis/BindAxisKey/BindKey/BindTouch/BindVectorAxis`（`FRegisterInputComponent.cpp:77/113/141/169/204/259`）经 lambda 调用，即**每次 C# 侧声明一个新的输入绑定函数名时执行一次**。运行于游戏线程。

**问题**
全插件 `AddToRoot` 的配对情况（`grep -n "AddToRoot|RemoveFromRoot" Source` 共 26 命中）：

| 位置 | 是否有配对 `RemoveFromRoot` |
|---|---|
| `FCSharpBind.cpp:530` | 有（`FCSharpFunctionRegister.cpp:61` / `FRegisterClass.cpp:23`） |
| `FDynamicStructGenerator.cpp:249` | 有（`:380`） |
| `FDynamicClassGenerator.cpp:393/419` | 有（`:533/540/893`） |
| `FDelegateHelper.cpp:24` / `FMulticastDelegateHelper.cpp:25` | 有（`:38` / `:39`） |
| **`FRegisterInputComponent.cpp:312`** | **无** |
| **`FRegisterEnhancedInputComponent.cpp:97`** | **无** |

这两个 UFunction 在 `AddFunctionToFunctionMap` 之后已被 `InClass` 的函数表引用，正常情况下 GC 会随 `UClass` 一起保活，因此 `AddToRoot()` 属于**多余**；但当 `InClass` 是瞬态类（`RF_Transient` 的生成类、PIE 反复开关、模块热重载）时，`AddToRoot()` 会让这些函数**永远无法被回收**，`InClass->Children` 链表与函数表在对象的整个生命周期内持续增长。由于 `FRegisterInputComponent.cpp:291`（及 `FRegisterEnhancedInputComponent.cpp:76`）在函数已存在时提前返回，泄漏量以"不同函数名个数"为界；若 C# 侧用动态拼接的函数名（如 `BindAction(..., "OnFire_" + id)`），这将是**无上界泄漏**。

**建议**
删除两处 `Function->AddToRoot();`；若确实需要防 GC（编辑器中 `InClass` 可能是临时类），改为在类销毁/模块卸载路径上配对 `RemoveFromRoot()`（可参照 `FDynamicBlueprintExtensionScope.h:23/42` 的 RAII 作用域写法）。

**验证方式**
1. `grep -rn "AddToRoot" Plugins/UnrealCSharp/Source | Measure-Object` → 26；对本文件命中 `FRegisterInputComponent.cpp:312`、`FRegisterEnhancedInputComponent.cpp:97`，两文件内 `grep RemoveFromRoot` → 0。
2. 编辑器内循环 1000 次创建/销毁带输入绑定的 PIE 会话，观察 `UFunction` 对象数（`obj list class=UFunction`）是否单调增长。

---

### [F-INT2-016] 防御性判空写成 `&In != nullptr`：对引用恒为真，编译期被优化掉，**不能**防止 C# 传空指针

- **类别**: 死代码 / 未定义行为
- **严重度**: **P3**(隐患)
- **复核结论**: 部分确认（偏差：**grep 命中数写错**——原文"合计约 70 处"实测为 **62** 处；且表中 `FRegisterGuid.cpp` 一行与所用模式不符）
- **可达性**: 活跃
- **复核证据**: ①工具 grep `&In != nullptr`，路径 `Plugins/UnrealCSharp/Source/UnrealCSharp/Private/Domain/Interop` → **62 命中**（原文验证方式写"约 70 处"，偏大）。②逐文件命中行与原文表格**全部核对通过**：`FRegisterVector.cpp` 16 处（11/16/21/26/31/36/41/46/51/56/61/66/71/76/81/86）、`FRegisterVector2D.cpp` 9、`FRegisterVector4.cpp` 9、`FRegisterRotator.cpp` 3、`FRegisterTransform.cpp` 1、`FRegisterMatrix.cpp` 1、`FRegisterQuat.cpp` 6、`FRegisterLinearColor.cpp` 2、`FRegisterPlane.cpp` 4、`FRegisterBox2D.cpp` 1、`FRegisterDateTime.cpp` 4、`FRegisterTimespan.cpp` 2、`FRegisterFrameNumber.cpp` 2、`FRegisterFrameTime.cpp` 2 —— 表内合计 64 行，减去 `FRegisterGuid.cpp` 那 2 行（见下）正好 **62** ✓ 自洽。③**表中 `FRegisterGuid.cpp`（34, 44）一行与表头声明不符**：该文件用的是 `&X`/`&Y`/`&Value`（`FRegisterGuid.cpp:34`、`:44`），**不含 `&In`**，因此不被 `&In != nullptr` 模式命中；它属同类写法但不应计入本表计数。④`FRegisterVector.cpp:9-27` 原文引文逐字吻合；`FRegisterVector2D.cpp:38/43` 确为对 `double` 标量"判空"（`:38 return &In != nullptr && (&A != nullptr) ? In + A : …`）。
- **级别变动**: 无（P3 维持）
- **文件**: 见下表（`FRegister*Implementation` 辅助函数的参数校验；**计数已修正为 62**）
- **函数**: 各文件的 `*Implementation(const T& In, ...)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterVector.cpp:9-27
		static FVector::FReal BitOrImplementation(const FVector& In, const FVector& V)
		{
			return &In != nullptr && (&V != nullptr) ? In | V : decltype(In | V)();
		}

		static FVector MinusImplementation(const FVector& In, const int32 Bias)
		{
			return &In != nullptr ? In - Bias : decltype(In - Bias)();
		}

		static FVector MinusImplementation(const FVector& In, const float Bias)
		{
			return &In != nullptr ? In - Bias : decltype(In - Bias)();
		}
```
```cpp
// FRegisterVector2D.cpp:36-44 —— 连标量参数也"判空"
		static FVector2D PlusImplementation(const FVector2D& In, const FVector2D::FReal A)
		{
			return &In != nullptr && (&A != nullptr) ? In + A : decltype(In + A)();
		}

		static FVector2D MinusImplementation(const FVector2D& In, const FVector2D::FReal A)
		{
			return &In != nullptr && (&A != nullptr) ? In - A : decltype(In - A)();
		}
```
同类写法分布（`grep -c '&In != nullptr'`）：

| 文件 | 命中行（部分） |
|---|---|
| `FRegisterVector.cpp` | 11, 16, 21, 26, 31, 36, 41, 46, 51, 56, 61, 66, 71, 76, 81, 86 |
| `FRegisterVector2D.cpp` | 13, 18, 23, 28, 33, 38, 43, 48, 53 |
| `FRegisterVector4.cpp` | 12, 17, 22, 27, 32, 37, 42, 47, 52 |
| `FRegisterRotator.cpp` | 12, 17, 22 |
| `FRegisterTransform.cpp` | 11 |
| `FRegisterMatrix.cpp` | 12 |
| `FRegisterQuat.cpp` | 12, 17, 22, 27, 32, 37 |
| `FRegisterLinearColor.cpp` | 13, 18 |
| `FRegisterPlane.cpp` | 12, 17, 22, 27 |
| `FRegisterBox2D.cpp` | 13 |
| `FRegisterGuid.cpp` | 34, 44 |
| `FRegisterDateTime.cpp` | 56, 62, 68, 73 |
| `FRegisterTimespan.cpp` | 11, 16 |
| `FRegisterFrameNumber.cpp` | 11, 16 |
| `FRegisterFrameTime.cpp` | 11, 16 |

**调用上下文**
这些 `*Implementation` 是 `.Function(...)` 的目标，经 `TFunctionBuilder::Invoke` 包装（`FRegisterVector.cpp:110-140` 等），最终由 C# 的函数指针调用。

**问题**
1. `In` 是**引用**，`&In != nullptr` 在良构 C++ 中恒真（绑定时若指针为 0 已是 UB），编译器直接视为 `true` 并删除该分支 → **`decltype(...)()` 的兜底返回值永不生效，是死代码**。
2. 因此这些检查**不提供任何保护**。C# 传 0 句柄时，thunk 不会走到这里的 `else`，而是在更早的 `TArgument`/`TPropertyValue` 阶段就把 0 当作地址使用（`TArgument.inl:28-29` 直接 `*(std::decay_t<Type>*)IN_BUFFER`），或在 `FStructRegistry::GetAddress` 返回 null 后被 `*` 解引用（见 F-INT2-002/008）。
3. `FRegisterVector2D.cpp:38/43` 甚至对一个 **`double` 值参数**做"判空"，说明是模板化复制时未加思考，容易让读者误以为这里做了完整的入参校验。

**建议**
删除全部 `&X != nullptr` 判断（它们唯一的实际效果是让读者高估安全性）。若要真正防御 C# 传入 0 句柄，应在**句柄层**统一加校验，例如在 `FStructRegistry::GetAddress` / `TArgument` 构造处 `ensure(InManagedHandle != InvalidManagedHandle)`，或在生成器产出的 C# 桩里对 `nint` 参数做判零抛异常（`UnrealTypeSourceGenerator.cs:916-918` 是唯一合适的注入点）。

**验证方式**
`grep -rn "&In != nullptr" Plugins/UnrealCSharp/Source/UnrealCSharp/Private/Domain/Interop | Measure-Object` → 上表合计约 70 处；在 MSVC 下开 `/W4` 观察是否报 `C4296`（表达式恒为真）之类的告警。

---

### [F-INT2-017] `FSoftClassPath` 拷贝构造的参数名数组带 2 项、其中 `"x"` 是残留

- **类别**: Bug / 一致性
- **严重度**: **P3**(隐患)
- **复核结论**: 部分确认（偏差：**危害被生成产物证伪**——`"x"` 是纯冗余元数据，对 C# 签名零影响；代码事实成立）
- **可达性**: 活跃
- **复核证据**: ①`FRegisterSoftClassPath.cpp:19-20` **逐字吻合**：`.Constructor(BINDING_CONSTRUCTOR(FSoftClassPath, FSoftClassPath const&), TArray<FString>{"Other", "x"})`；对照 `FRegisterSoftObjectPath.cpp:18-19` 写的是 `{"Other"}` ✓（原文对照正确）。②**生成地面真值**：`Script/UE/Proxy/Binding/SoftClassPath.cs:14 public FSoftClassPath(FSoftClassPath Other)` —— **只有 1 个形参**，多出的 `"x"` 被生成器静默丢弃。故原文"会导致生成的 C# 构造函数形参列表与 native 不匹配（多出的形参没有对应的 native 位置）"**证伪**：C# 形参个数由**真实构造函数签名**决定，不由 `ParamNames` 长度决定。
- **级别变动**: 无（P3 维持）—— 已降为"源码冗余/可读性"级别，与文字已更正后的危害描述一致
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterSoftClassPath.cpp:19-20`
- **函数**: `FRegisterSoftClassPath::FRegisterSoftClassPath()`
- **置信度**: 中（参数名与构造参数个数不符为确定；生成器因此产生的具体 C# 签名未逐行确认）

**现状（代码事实）**
```cpp
// FRegisterSoftClassPath.cpp:18-24
			TBindingClassBuilder<FSoftClassPath>(NAMESPACE_BINDING)
				.Constructor(BINDING_CONSTRUCTOR(FSoftClassPath, FSoftClassPath const&),
				             TArray<FString>{"Other", "x"})
				.Constructor(BINDING_CONSTRUCTOR(FSoftClassPath, const FString&),
				             TArray<FString>{"PathString"})
				.Constructor(BINDING_CONSTRUCTOR(FSoftClassPath, const UClass*),
				             TArray<FString>{"InClass"})
```

**调用上下文**
`FRegisterSoftClassPath.cpp:31` 静态注册 → C# `FSoftClassPath` 构造函数。参数名经 `TClassBuilder::Constructor`（`Source/UnrealCSharp/Public/Binding/Class/TClassBuilder.inl:34-51`）交给 `FFunctionInfo::SetParamNames`。

**问题**
拷贝构造只有 1 个参数，却给了 2 个名字；`"x"` 既不是合法参数名语义（对照同批次 `FRegisterSoftObjectPath.cpp:18-19` 的拷贝构造写的是 `{"Other"}`），也提示这里曾经想绑一个 `(const TCHAR* x)` 之类的构造。元数据与真实函数不一致会导致生成的 C# 构造函数形参列表与 native 不匹配（多出的形参没有对应的 native 位置），属于"看起来能编译、语义可疑"的一类。

**建议**
```cpp
.Constructor(BINDING_CONSTRUCTOR(FSoftClassPath, FSoftClassPath const&), TArray<FString>{"Other"})
```
如果本意是想提供 `FSoftClassPath(const TCHAR*)`，应显式加一条（并与 `FRegisterSoftObjectPath.cpp:20` 的 `const FString&` 版本保持一致的对外形态）。

**验证方式**
`grep -n '"Other", "x"' Source/UnrealCSharp/Private/Domain/Interop/*.cpp` → 仅 `FRegisterSoftClassPath.cpp:20`；对照 `FRegisterSoftObjectPath.cpp:19` 的 `{"Other"}`。

---

### [F-INT2-018] `FVector::DotProduct` 被同名 `Dot` 挤成 `Dot1`，与同文件 `CrossProduct` 的处理方式不一致

- **类别**: Bug / 命名一致性
- **严重度**: **P3**(API 形状错误)
- **复核结论**: 部分确认（偏差：**"C# 侧得到 `Dot1`"被生成产物证伪**；命名不一致本身成立）
- **可达性**: 活跃
- **复核证据**: ①`FRegisterVector.cpp:157-164` **逐字吻合**（`:161 .Function("Dot", …&FVector::Dot…)`、`:163 .Function("Dot", …&FVector::DotProduct…)`）。②**生成地面真值**：`Script/UE/Proxy/Binding/Vector.cs:781 public double Dot(FVector V)`（实例）与 `:797 public static double Dot(FVector A, FVector B)`（静态）—— **C# 侧是一对同名重载，不存在 `Dot1`**；`:747 public FVector Cross(FVector V2)` 与 `:763 public static FVector CrossProduct(FVector A, FVector B)` 确实用了两个不同名字。③对照 `Vector2D.cs:562 public static double DotProduct(FVector2D A, FVector2D B)` 与 `:786 public double Dot(FVector2D V2)` —— **`FVector2D` 有 `DotProduct`、`FVector` 没有**，命名不一致的实质成立。
- **级别变动**: 无（P3 维持）。真实缺陷仅剩"同一数学概念在 `FVector` 上叫 `Dot`、在 `FVector2D` 上叫 `DotProduct`"
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterVector.cpp:157-164`
- **函数**: `FRegisterVector::FRegisterVector()`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterVector.cpp:157-164
				.Function("Cross", BINDING_FUNCTION(&FVector::Cross,
				                                    TArray<FString>{"V2"}))
				.Function("CrossProduct", BINDING_FUNCTION(&FVector::CrossProduct,
				                                           TArray<FString>{"A", "B"}))
				.Function("Dot", BINDING_FUNCTION(&FVector::Dot,
				                                  TArray<FString>{"V"}))
				.Function("Dot", BINDING_FUNCTION(&FVector::DotProduct,
				                                  TArray<FString>{"A", "B"}))
```

**调用上下文**
同批次其它文件对同一对 API 的处理是**各自用自己的名字**：`FRegisterVector2D.cpp:125 DotProduct`（静态）与 `:148 Dot`（成员）、`FRegisterQuat/AngularDistance` 等同理。

**问题**
`Cross`/`CrossProduct` 用两个不同名字（正确），但 `Dot`/`DotProduct` 都写成 `Dot`。按 `FClassBuilder.cpp:83-91` 的同名计数规则，静态版本注册为 **`Dot1`**，因此 C# 侧得到 `FVector.Dot(V)` 与 `FVector.Dot1(A, B)`。`Dot1` 是无意义的名字，且与 C# 侧 `FVector2D` 的 `DotProduct` 不一致 —— 同一个数学概念在两个类型上名字不同，用户必须靠猜。

**建议**
```cpp
.Function("Dot", BINDING_FUNCTION(&FVector::Dot, TArray<FString>{"V"}))
.Function("DotProduct", BINDING_FUNCTION(&FVector::DotProduct, TArray<FString>{"A", "B"}))
```
**并建议把"同名计数后缀"这一消歧机制显式暴露出来**：`FClassBuilder::GetFunctionImplementationName` 生成 `Name1/Name2` 是正确且必要的（C 侧无法重载），但**开发者看不到后缀会被生成**，这是本批次 3 个错绑 bug 的共同成因（见第 7 节）。

**验证方式**
`grep -n '"Dot"' Source/UnrealCSharp/Private/Domain/Interop/FRegisterVector.cpp` → :161、:163 两处；C# 生成产物里搜索 `Dot1`。

---

### [F-INT2-019] `FFrameNumber`/`FFrameTime` 的标量重载只有 `float`（**两个部分**：`FFrameNumber` 部分**证伪·非缺陷**——引擎只有 `float` 重载且内部已提升 `double`；`FFrameTime` 部分**成立·P3**——引擎有 `double` 重载而插件窄化为 `float`）

- **类别**: 平台兼容 / 精度
- **严重度**: **P3**(精度/一致性，仅 `FFrameTime` 部分；`FFrameNumber` 部分撤销)
- **复核结论**: 部分确认 —— **结论方向正确但过宽**：`FFrameNumber` 一半**证伪（非缺陷）**；`FFrameTime` 一半**成立**（应保留）
- **可达性**: 活跃
- **复核证据**（**引擎源码已核对，原文 §5 第 1 项"未能核对"就此关闭**）：①`Runtime/Core/Public/Misc/FrameNumber.h:66-67` —— 引擎对 `FFrameNumber` **只有** `friend FFrameNumber operator*(FFrameNumber A, float Scalar)` 与 `operator/(FFrameNumber A, float Scalar)` 两个重载，**没有 `double` 版本**；且引擎自身实现为 `static_cast<int32>(FMath::Clamp(FMath::FloorToDouble(double(A.Value) * Scalar), …))` —— **内部已提升到 `double` 再取整并饱和到 int32 范围**。故 `FRegisterFrameNumber.cpp:9/:14` 绑 `float` 是**唯一可选**且与引擎一致的写法 → 原文"大帧号下精度丢失"对 `FFrameNumber` **证伪**。②但 `Runtime/Core/Public/Misc/FrameTime.h:232-245` 显示 `FFrameTime` **有 `double` 重载**（`operator*(FFrameTime A, double Scalar)`、`operator*(double Scalar, FFrameTime A)`、`operator/(FFrameTime A, double Scalar)`，实现为 `FFrameTime::FromDecimal(A.AsDecimal() * Scalar)`），而 `FRegisterFrameTime.cpp:9/:14` 却把标量窄化成 `float` → `In * Scalar` 经 float→double 隐式提升调用的仍是引擎 double 重载，**但标量已在 C#→C++ 边界被截成 24 位精度**。这是**真实但轻微**的精度收窄。
- **级别变动**: **撤销（非缺陷）→ P3**（拆分定性：`FFrameNumber` 部分撤销；`FFrameTime` 部分保留为 P3 精度/一致性问题，可把 `:9/:14` 的 `float` 改为 `double` 直接对齐引擎重载）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterFrameNumber.cpp:9-17`、`34-35`（**非缺陷**）；`FRegisterFrameTime.cpp:9-17`、`36-37`（**P3 成立**）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterFrameNumber.cpp:9-17`、`34-35`；`FRegisterFrameTime.cpp:9-17`、`36-37`
- **函数**: `MultipliesImplementation` / `DividesImplementation`
- **置信度**: 中（`float` 精度上限是确定的；"UE 是否已改为 double 重载"未在本机核对，见第 5 节）

**现状（代码事实）**
```cpp
// FRegisterFrameNumber.cpp:9-17
		static FFrameNumber MultipliesImplementation(const FFrameNumber& In, const float Scalar)
		{
			return &In != nullptr ? In * Scalar : decltype(In * Scalar)();
		}

		static FFrameNumber DividesImplementation(const FFrameNumber& In, const float Scalar)
		{
			return &In != nullptr ? In / Scalar : decltype(In / Scalar)();
		}
```
```cpp
// FRegisterFrameTime.cpp:9-17 同型；FFrameTime = FFrameNumber + float SubFrame（本文件 :38 MaxSubframe、:24-27 三参构造）
```

**调用上下文**
`FRegisterFrameNumber.cpp:39` / `FRegisterFrameTime.cpp:50` 静态注册 → C# `FFrameNumber`/`FFrameTime` 的 `operator *` / `operator /`，用于把帧号缩放（时间膨胀、快进、区间映射）。

**问题**
`FFrameNumber` 内部是 `int32`（本文件 `:22` 构造参数名为 `InValue`，`FRegisterDateTime.cpp:79` 用 `int64` 表示 ticks 形成对照）。乘以 `float` 时，`float` 只有 24 位有效位：帧号超过 `2^24 ≈ 1.6e7` 后 `float` 无法精确表示整数（按 30fps 约 6.5 天连续运行可达），`In * Scalar` 的中间结果会**静默丢精度**，长时间运行的播放/回放、`SetPlayRate` 类逻辑会出现累积漂移。`FRegisterTimespan.cpp:9/14` 和 `FRegisterDateTime.cpp:79` 都使用了 `double`/`int64`，本文件是唯一退化到 `float` 的浮点标量重载。

**建议**
优先绑定 `double` 重载（UE5 已将多数 `float` 数学重载改为 `double`）；若必须保留 `float`，在 thunk 内改为先提升到 `double` 计算再取整，并加注释说明精度边界：
```cpp
static FFrameNumber MultipliesImplementation(const FFrameNumber& In, const double Scalar)
{
    return In * Scalar;   // 走 double 重载
}
```

**验证方式**
`grep -n "const float Scalar" Source/UnrealCSharp/Private/Domain/Interop/FRegisterFrame*.cpp` → `FrameNumber.cpp:9/:14`、`FrameTime.cpp:9/:14`（4 处，全批次唯一的 `float` 标量重载）。对照 `FRegisterTimespan.cpp:9/:14` 的 `double`。用例：`new FFrameNumber(20000000) * 0.5f` 与 `* 0.5` 结果比较。

---

### [F-INT2-020] `FPlane` 的 `operator*` 重载用了非 const 引用参数，与 `BINDING_OVERLOAD` 声明的签名不一致

- **类别**: 可优化/可读性 / 一致性
- **严重度**: **P3**(隐患)
- **复核结论**: 部分确认（偏差：**"生成的 C# 侧把接收者当作可写引用处理"被生成产物证伪**；非 const 引用本身与同批次惯例不一致属实）
- **可达性**: 活跃
- **复核证据**: ①`FRegisterPlane.cpp:25-28`、`:48-51` **逐字吻合**：`:25 static FPlane MultipliesImplementation(FPlane& In, const FPlane& V)`（同文件 `:20` 的同名重载是 `const FPlane&`）、`:51 BINDING_OVERLOAD(FPlane(*)(FPlane&, const FPlane&), &MultipliesImplementation)`。②**生成地面真值**：`Script/UE/Proxy/Binding/Plane.cs:190 public static FPlane operator *(FPlane InValue0, double InValue1)` 与 `:208 public static FPlane operator *(FPlane InValue0, FPlane InValue1)` —— 均为 `static` 值语义运算符，**没有任何"可写接收者"形态**；`FPlane` 以句柄封送、thunk 不回写。故原文"一旦 C# 侧把它当作原地修改使用…静默错误"**证伪**。
- **级别变动**: 无（P3 维持）。剩余问题＝源码风格不一致（非 const 引用阻止对临时量/只读包装调用），无运行时影响
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterPlane.cpp:25-28`、`50-51`
- **函数**: `FRegisterPlane::MultipliesImplementation(FPlane& In, const FPlane& V)`
- **置信度**: 中

**现状（代码事实）**
```cpp
// FRegisterPlane.cpp:20-28
		static FPlane MultipliesImplementation(const FPlane& In, const FPlane::FReal Scale)
		{
			return &In != nullptr ? In * Scale : decltype(In * Scale)();
		}

		static FPlane MultipliesImplementation(FPlane& In, const FPlane& V)      // 注意：非 const 引用
		{
			return &In != nullptr && (&V != nullptr) ? In * V : decltype(In * V)();
		}
```
```cpp
// FRegisterPlane.cpp:48-51
				.Function("operator *", FUNCTION_MULTIPLIES,
				          BINDING_OVERLOAD(FPlane(*)(const FPlane&, const FPlane::FReal), &MultipliesImplementation))
				.Function("operator *", FUNCTION_MULTIPLIES,
				          BINDING_OVERLOAD(FPlane(*)(FPlane&, const FPlane&), &MultipliesImplementation))
```

**调用上下文**
`FRegisterPlane.cpp:72` 静态注册 → C# `FPlane.operator *`。

**问题**
同批次所有其它二元重载的第一个参数都是 `const T&`（如 `FRegisterVector.cpp:112-140`、`FRegisterVector4.cpp:79-95`、`FRegisterQuat.cpp:59-67`），**只有 `FPlane` 这一处**把左操作数写成 `FPlane&`，并且 `BINDING_OVERLOAD` 里也照着写成了 `FPlane&`（`FPlane(*)(FPlane&, const FPlane&)`）。这意味着生成的 C# 侧把这个重载的接收者当作**可写引用**处理，与 UE 的 `operator*` 是 const 值运算的语义不符；一旦 C# 侧把它当作原地修改使用，就会出现"看起来修改了、其实返回新值被丢弃"的静默错误（与 F-INT2-005 同一类失效）。此外非 const 引用会阻止对临时量/只读包装调用该重载。

**建议**
```cpp
static FPlane MultipliesImplementation(const FPlane& In, const FPlane& V) { return In * V; }
...
.Function("operator *", FUNCTION_MULTIPLIES,
          BINDING_OVERLOAD(FPlane(*)(const FPlane&, const FPlane&), &MultipliesImplementation))
```

**验证方式**
`grep -n "^		static FPlane MultipliesImplementation" Source/UnrealCSharp/Private/Domain/Interop/FRegisterPlane.cpp` → :20（const）、:25（非 const）；`grep -rn "FPlane&, const FPlane&" Source/UnrealCSharp/Private/Domain/Interop` → :25、:51 两处。

---

### [F-INT2-021] 重复代码：`BindFunction` 与 `GetDynamicBindingObjectImplementation` 在两个文件里逐行重复

- **类别**: 可优化/可读性
- **严重度**: **P2**(可维护性)
- **复核结论**: 确认 —— 代码事实（两文件逐行重复）、类别、严重度 P2 全部成立
- **可达性**: 活跃
- **复核证据**: ①`FRegisterInputComponent.cpp:17-24` 与 `FRegisterEnhancedInputComponent.cpp:43-50` 的 `GetDynamicBindingObjectImplementation` **逐字吻合**（含 `:46-47`/`:43-44` 的换行位置）；后者整体为 `:43-66`。②`BindFunction`：`FRegisterInputComponent.cpp:283-319`、`FRegisterEnhancedInputComponent.cpp:68-104` 均**逐行同构**，且**两处都含 `Function->AddToRoot();`**（`:312` / `:97`）—— 原文"修复 F-INT2-015 必须同时改两处"成立。③两文件的 `.Function` 导出：`FRegisterEnhancedInputComponent.cpp:220-222`（3 个）与 `FRegisterInputComponent.cpp:333-340`（8 个）均吻合。
- **级别变动**: 无（P2 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterInputComponent.cpp:283-319`、`Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp:68-104`、`FRegisterInputComponent.cpp:17-40`、`FRegisterEnhancedInputComponent.cpp:43-66`
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterInputComponent.cpp:283-319`、`Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp:68-104`、`FRegisterInputComponent.cpp:17-40`、`FRegisterEnhancedInputComponent.cpp:43-66`
- **函数**: `BindFunction(...)`、`GetDynamicBindingObjectImplementation(...)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterInputComponent.cpp:17-24
		static IManagedHandle GetDynamicBindingObjectImplementation(const IManagedHandle InThisClass,
		                                                            const IManagedHandle InBindingClass)
		{
			const auto ThisClass = FCSharpEnvironment::GetEnvironment().GetObject<
				UBlueprintGeneratedClass>(InThisClass);

			const auto BindingClass = FCSharpEnvironment::GetEnvironment().GetObject<UClass>(InBindingClass);
```
```cpp
// FRegisterEnhancedInputComponent.cpp:43-50（逐字符相同）
		static IManagedHandle GetDynamicBindingObjectImplementation(const IManagedHandle InThisClass,
		                                                            const IManagedHandle InBindingClass)
		{
			const auto ThisClass = FCSharpEnvironment::GetEnvironment().GetObject<
				UBlueprintGeneratedClass>(InThisClass);

			const auto BindingClass = FCSharpEnvironment::GetEnvironment().GetObject<UClass>(InBindingClass);
```
另外两组：
- `BindFunction`：`FRegisterInputComponent.cpp:283-319`（37 行）与 `FRegisterEnhancedInputComponent.cpp:68-104`（37 行）除缩进外完全一致，**包括 F-INT2-015 提到的无配对 `AddToRoot()`**。
- `BindAxisImplementation` 与 `BindAxisKeyImplementation` 的 lambda：`FRegisterInputComponent.cpp:113-126` 与 `:141-154` 的 `FFloatProperty(TEXT("AxisValue"))` 构造代码逐行相同（只差缩进）。

**调用上下文**
六个 `Bind*Implementation`（`FRegisterInputComponent.cpp:334-339`）与 `BindAction`（`FRegisterEnhancedInputComponent.cpp:221`）都经这些函数。

**问题**
- 修复 F-INT2-015（删除无配对 `AddToRoot`）必须**同时改两处**，漏一处即留下不一致；五、六两处 `Bind*` 的差异只在属性类型/取参方式上，正是最容易被"只改一边"的地方。
- `GetDynamicBindingObjectImplementation` 完全相同却被复制，说明这两个文件是复制粘贴产生的，**未来的修复不会自动传播**。

**建议**
把 `GetDynamicBindingObjectImplementation` 与 `BindFunction` 提取到共享的静态工具（例如放在 `Source/UnrealCSharp/Private/Domain/Interop/FRegisterInputBindingShared.h`，或在 `FRegisterInputComponent.cpp` 里做非匿名命名空间的 `FInputBindingRegistrationHelper` 供两处 include）。属性 lambda 可用表驱动消除：
```cpp
struct FBindingParamDesc { const TCHAR* Name; UScriptStruct* Struct; EPropertyFlags Flags; };
```
连同 F-INT2-022 提出的"运算符宏化"一起，本批次 1248 个注册条目里有大量机械重复可收敛。

**验证方式**
`grep -n "GetDynamicBindingObjectImplementation" Source/UnrealCSharp/Private/Domain/Interop/*.cpp` → 4 处（两个定义 + 两个 `.Function` 引用）；`grep -c "AddToRoot" ...` → 两文件各 1。

---

### [F-INT2-022] 36 个文件高度模板化，但 1248 个注册条目全靠手写：给出可宏化/表驱动的具体方案

- **类别**: 可优化/可读性
- **严重度**: **P3**(可维护性)
- **复核结论**: 部分确认（偏差：**"本批次 6 处错绑"的清单需按证伪结果重算**——`F-INT2-006` 的机制被证伪、`F-INT2-018` 的 `Dot1` 被证伪，真正"名字≠目标"的错绑为 **3 处**：`Matrix.cpp:78`、`PolyglotTextData.cpp:19`、`Rotator.cpp:46`、`Rotator.cpp:65`、`SoftObjectPath.cpp:34` 中的 5 处**名字错**（其中 `Rotator.cpp:65`/`SoftObjectPath.cpp:34` 是"错名覆盖正确目标"，仍属错绑）＋ `Vector.cpp:163` 的命名不一致；**"1248 个条目"与"36 个文件"的规模陈述保持不变**）
- **可达性**: 活跃
- **复核证据**: ①第 0 节表格 36 行逐行核对，文件数、行数、条目数**与实文件一致**（复核了其中 12 个文件的 `Get-Content .Count`）。②`FRegisterVector4.cpp:78-95` 重复 9 次 `BINDING_OVERLOAD` 的陈述与实文件相符。③同构对（`FloatInterval`/`Int32Interval` 28 行、`FloatRange`/`Int32Range` 63 行、`FloatRangeBound`/`Int32RangeBound` 40 行）**行数完全相等** ✓。④"规模越大越机械的部分越安全"的论断被 EKeys 审计正面支持：`FRegisterKeys.cpp` 342 条 **0 条名字≠目标**，而 6 处名字错全部出现在需要"动脑"的标量/语义映射处。
- **级别变动**: 无（P3 维持）
- **文件**: 全部 36 个文件（规模见第 0 节表格）
- **函数**: `FRegister*::FRegister*()`
- **置信度**: 高

**现状（代码事实）**
`FRegisterVector.cpp` 单个构造 133 个条目、`FRegisterTransform.cpp` 85 个、`FRegisterQuat.cpp` 64 个、`FRegisterKeys.cpp` 342 个 `.Property`。每类值的"运算符族"写法完全一致，例如 `FRegisterVector4.cpp:78-95` 重复 9 次同样的 `BINDING_OVERLOAD(FVector4(*)(const FVector4&, const T), &MultipliesImplementation)`，只换标量类型 `int32/float/double`；`FRegisterTimespan.cpp`/`FRegisterFloatRange`/`FRegisterInt32Range` 与各自的 `Bound`/`Interval` 版本**逐行同构**（`FRegisterFloatInterval.cpp` 与 `FRegisterInt32Interval.cpp` 除类型名外完全相同，各 28 行）。

**调用上下文**
全部为静态注册（每个文件末尾 `[[maybe_unused]] FRegisterX RegisterX;`），在模块加载时一次性执行，运行时无成本 —— 因此这里的重构风险很低。

**问题**
手工逐条书写的直接后果已经在本报告中量化：**6 处"名字≠目标"错绑**（F-INT2-003/004/005/006/007/018）全部出现在这些手写条目里，而 `FRegisterKeys.cpp` 的 342 条由于是机械复制反而 0 错（见第 4 节审计）。也就是说，**规模越大越机械的部分越安全，需要"动脑"的标量重载与 API 语义映射才是错误来源**。

**建议（分三层，按收益排序）**
1. **标量重载族宏化**（收益最高，正对着本批次的 bug 成因）：
```cpp
#define REGISTER_SCALAR_OPERATORS(T, Op, OpName, FieldOp)                       \
	.Function(OpName, BINDING_OVERLOAD(T(*)(const T&, const int32),             \
	             &ScalarOp<T, int32, FieldOp>::Apply))                          \
	.Function(OpName, BINDING_OVERLOAD(T(*)(const T&, const float),             \
	             &ScalarOp<T, float, FieldOp>::Apply))                          \
	.Function(OpName, BINDING_OVERLOAD(T(*)(const T&, const double),            \
	             &ScalarOp<T, double, FieldOp>::Apply))
```
配合一个模板（替代每个文件里的 3~9 个 `*Implementation` 静态函数）：
```cpp
template <typename T, typename Scalar, typename = void>
struct ScalarOp { static T Apply(const T& In, const Scalar S) { return In * S; } };
```
`FRegisterVector/Vector4/Rotator/Quat/Matrix/Timespan/Plane/FrameNumber/FrameTime/LinearColor` 这 10 个文件的标量重载（合计约 40 个手写 `*Implementation`）可压到 1 个宏 + 1 个模板，且**改动标量类型只需改一处**。

2. **"名字/目标"一致性校验**：在 `FClassBuilder::Function` 加入注册期断言，把本批次 6 个 bug 变成启动即报错：
```cpp
// FClassBuilder.cpp:59 附近
auto FClassBuilder::Function(const FString& InName, const FString& InImplementationName, const void* InMethod
#if WITH_FUNCTION_INFO
                             , const TOptional<TFunction<FFunctionInfo*()>>& InFunctionInfoFunction
#endif
)
{
#if WITH_FUNCTION_INFO
	// 启发式 1：Get*/Is*/Has*/To* 前缀不应绑定到非 const 成员函数
	// 启发式 2：InFunctionInfoFunction 的形参个数必须等于被绑函数的形参个数
#endif
}
```
其中**"形参个数必须一致"这一条能直接抓住 F-INT2-005（1 vs 2）与 F-INT2-006（1 vs 2）**，性价比最高。

3. **同构类型抽公共注册函数**：把 `FFloatInterval`/`FInt32Interval`、`FFloatRange`/`FInt32Range`、`FFloatRangeBound`/`FInt32RangeBound` 各自的 7/17/15 条注册改为一个 `template <typename RangeT, typename BoundT> void RegisterRange(TBindingClassBuilder<RangeT>& B)`，六个文件可合并为两个（各让出 28~63 行）。

**验证方式**
重构后逐条比对 C# 生成产物与重构前的 API 清单（同名同参数），差异应为空；`FRegisterFloatInterval.cpp` 与 `FRegisterInt32Interval.cpp` 的注册结果可用同一份 C# 单测覆盖。

---

### [F-INT2-023] `FRegisterGuid` 手抄 UE 的 `operator>` 八层嵌套三元；`LexToString` 包装无可读性收益

- **类别**: 可优化/可读性
- **严重度**: **P3**(可维护性)
- **复核结论**: 确认 —— 代码事实、类别、严重度 P3 全部成立
- **可达性**: 活跃
- **复核证据**: ①`FRegisterGuid.cpp:32-45`、`:57-59` **逐字吻合**：`:32 bool GreaterThanImplementation(const FGuid& X, const FGuid& Y)`、`:34-39` 的 8 层嵌套三元（含 `:34` 的 `return\t(&X != nullptr && (&Y != nullptr))` 制表符）、`:42-45 LexToStringImplementation` 的恒真判空包装、`:54 .Less()`、`:57 .Function("operator >", FUNCTION_GREATER, BINDING_FUNCTION(&GreaterThanImplementation))`。②逐字段比较逻辑复核：`>`→true、`<`→false、全等→false，**语义正确、与 `Less()` 互补**（原文"此处不是 bug"的判断成立）。③**引擎侧**：`FGuid` 确有 `operator>`（原文"既然 `operator>` 本身存在，应直接绑定"的前提成立，可在 `Runtime/Core/Public/Misc/Guid.h` 复核）。
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterGuid.cpp:32-45`、`57-59`
- **函数**: `FRegisterGuid::GreaterThanImplementation(const FGuid&, const FGuid&)`、`LexToStringImplementation(const FGuid&)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterGuid.cpp:32-45
		static bool GreaterThanImplementation(const FGuid& X, const FGuid& Y)
		{
			return	(&X != nullptr && (&Y != nullptr))
					? ((X.A > Y.A) ? true : ((X.A < Y.A) ? false :
					((X.B > Y.B) ? true : ((X.B < Y.B) ? false :
					((X.C > Y.C) ? true : ((X.C < Y.C) ? false :
					((X.D > Y.D) ? true : ((X.D < Y.D) ? false : false))))))))
					: false;
		}

		static FString LexToStringImplementation(const FGuid& Value)
		{
			return (&Value != nullptr) ? LexToString(Value) : decltype(LexToString(Value))();
		}
```

**调用上下文**
```cpp
// FRegisterGuid.cpp:54-59
				.Less()
				.Subscript(BINDING_SUBSCRIPT(FGuid, uint32, int32, TArray<FString>{"Index"}))
				.Function("operator >", FUNCTION_GREATER, BINDING_FUNCTION(&GreaterThanImplementation))
				.Function("LexToString", BINDING_FUNCTION(&LexToStringImplementation,
				                                          TArray<FString>{"Value"}))
```

**问题**
1. `GreaterThanImplementation` 是 UE `FGuid::operator>` 的**逐字符手抄**（那 8 层嵌套三元就是引擎原样），而 `:54` 的 `.Less()` 宏已经直接绑定了 `operator<`。既然 `operator>` 本身存在，这里应当直接绑定它；手抄的副本一旦 UE 修改比较语义（例如把 `A>B>C>D` 改为按字节比较）就会与引擎不一致，且**静默产生与 `Less()` 不互补的比较结果**（`!(a<b) && !(a>b)` 的边界语义会分裂）。`A/B/C/D` 逐字段比较本身是正确实现（已核对逻辑：`>` 则 true，`<` 则 false，相等继续到 D，全等返回 false），**此处不是 bug，是可维护性风险**。
2. `LexToStringImplementation` 只是给自由函数 `LexToString` 加了一个恒真的判空包装，没有增加任何语义，却占一个注册条目。

**建议**
```cpp
.Function("operator >", FUNCTION_GREATER, BINDING_FUNCTION(&FGuid::operator>))   // 直接绑引擎实现
.Function("LexToString", BINDING_FUNCTION(&LexToString, TArray<FString>{"Value"}))
```
并加一条单测固定 `Less`/`Greater` 的互补性。

**验证方式**
`grep -n "operator >" Source/UnrealCSharp/Private/Domain/Interop/FRegisterGuid.cpp` → :57；对照 `:54 .Less()`。C# 用例：对随机 GUID 对验证 `(a<b) ^ (b<a) ^ (a==b)` 恒真。

---

### P3

### [F-INT2-024] `FRegisterLinearColor.cpp` 重复 include 同一头文件

- **类别**: 可读性
- **严重度**: **P3**(风格)
- **复核结论**: 确认 —— 代码事实、类别、严重度 P3 全部成立
- **可达性**: 活跃
- **复核证据**: `FRegisterLinearColor.cpp:2-3` **逐字吻合**（连续两行 `#include "Binding/ScriptStruct/TScriptStruct.inl"`）。同批次其余 35 个文件中，同一直觉模式复查未再发现重复 include。因 `TScriptStruct.inl` 有 `#pragma once`，**无功能影响**（原文判断正确）。
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterLinearColor.cpp:2-3`
- **函数**: 文件级
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterLinearColor.cpp:1-5
#include "Binding/Class/TBindingClassBuilder.inl"
#include "Binding/ScriptStruct/TScriptStruct.inl"
#include "Binding/ScriptStruct/TScriptStruct.inl"
#include "Macro/NamespaceMacro.h"
#include "FRegisterForceInit.h"
```

**调用上下文**
文件级；`TScriptStruct.inl` 应当有 include guard/pragma once，因此**无功能影响**，只是复制粘贴痕迹。

**建议** 删除 :3。**验证方式** `Select-String -Path FRegisterLinearColor.cpp -Pattern 'TScriptStruct.inl'` → 2 命中；全目录唯一一处重复。

---

### [F-INT2-025] `FRegisterPlane.cpp` 缺空格、`FRegisterGuid.cpp` 混用制表符等格式不一致

- **类别**: 可读性
- **严重度**: **P3**(风格)
- **复核结论**: 确认 —— 三处代码事实与行号**全部逐字吻合**（仅原文"验证方式"里给的 grep 模式无效，见下）
- **可达性**: 活跃
- **复核证据**: ①`FRegisterPlane.cpp:55` = `.Function("PlaneDot",BINDING_FUNCTION(&FPlane::PlaneDot,`、`:60` = `.Function("Flip",BINDING_FUNCTION(&FPlane::Flip))` —— **缺空格属实**。②`FRegisterGuid.cpp:34` = `return\t(&X != nullptr && (&Y != nullptr))`（`return` 后用**制表符**）属实。③`FRegisterPrimaryAssetId.cpp:20-21` = `.Constructor(BINDING_CONSTRUCTOR(FPrimaryAssetId, const FString&), {"TypeAndName"})`（未写 `TArray<FString>`）、`:33` = `.Function("FromString", BINDING_FUNCTION(&FPrimaryAssetId::FromString));`（**完全省略参数名**）—— 属实；同文件 `:18-19`/`:24-29` 则显式给出 `TArray<FString>{…}`，不一致成立。④**模式修正**：原文验证方式用的 `\),BINDING` 实测 **0 命中**（该处 `,BINDING` 前面是 `"` 不是 `)`）；正确模式 `",BINDING` → **2 命中**，正好是 `FRegisterPlane.cpp:55/:60`，结论不变。
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterPlane.cpp:55`、`:60`；`FRegisterGuid.cpp:34`；`FRegisterPrimaryAssetId.cpp:21`、`:33`
- **函数**: 各注册构造函数
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterPlane.cpp:55-56
				.Function("PlaneDot",BINDING_FUNCTION(&FPlane::PlaneDot,
				                                      TArray<FString>{"P"}))
// FRegisterPlane.cpp:60
				.Function("Flip",BINDING_FUNCTION(&FPlane::Flip))
```
```cpp
// FRegisterGuid.cpp:34-35
			return	(&X != nullptr && (&Y != nullptr))
					? ((X.A > Y.A) ? true : ...
```
```cpp
// FRegisterPrimaryAssetId.cpp:20-21 / :33
				.Constructor(BINDING_CONSTRUCTOR(FPrimaryAssetId, const FString&),
				             {"TypeAndName"})                       // 未用 TArray<FString>{...}
...
				.Function("FromString", BINDING_FUNCTION(&FPrimaryAssetId::FromString))   // 无参数名
```

**调用上下文**
纯格式。注意 `:21` 的 `{"TypeAndName"}` 与 `:33` 的"完全省略 `TArray<FString>`"是两种不同的"偷懒"形式，**都依赖隐式转换**：`BINDING_FUNCTION(&FPrimaryAssetId::FromString)` 无参数名时，源生成器只能自行推导 C# 形参名（`Source/UnrealCSharp/Private/…` 之外），与同文件 `:24-29` 显式给出 `TArray<FString>{"TypeAndName"}` 的写法不一致。

**建议** 统一为 `TArray<FString>{...}` 并补全 `FromString` 的参数名（`{"TypeAndName"}`），`PlaneDot`/`Flip` 补空格；建议接入 `clang-format` 并把 `.clang-format` 加进仓库，一次性收敛 36 个文件。

**验证方式** `Select-String -Path Source/UnrealCSharp/Private/Domain/Interop/*.cpp -Pattern '\),BINDING'` → `FRegisterPlane.cpp:55/60`；`Select-String -Pattern 'BINDING_FUNCTION\(&[A-Za-z:]+\)\)' ` → `FRegisterPrimaryAssetId.cpp:33` 等。

---

### [F-INT2-026] `@TODO` 注释与"已实现"代码混排，容易误读为已完成；注释掉的 API 无追踪

- **类别**: 可读性 / 一致性
- **严重度**: **P3**(风格)
- **复核结论**: 确认 —— 代码事实、行号、类别、严重度 P3 全部成立
- **可达性**: 活跃
- **复核证据**: ①工具 grep `@TODO`（路径 `…/Private/Domain/Interop`）→ **11 命中**，分布为 `FRegisterFloatRange.cpp:18/28/38/42/54`（5）、`FRegisterColor.cpp:30`（1）、`FRegisterInt32Range.cpp:18/28/38/42/54`（5）—— **与原文声称的"两文件 TODO 行集合完全一致"吻合**。②`FRegisterFloatRange.cpp:18-24`（Adjoins/Conjoins/Contains/Contiguous/GetLowerBound/SetLowerBound）、`:28-30`（GetUpperBound/SetUpperBound）、`:38-39`（Overlaps）、`:42-45`（Difference/Hull/Intersection）、`:54-58`（Exclusive/GreaterThan/Inclusive/LessThan）**逐行吻合**。③`:53 .Function("Empty", …);` 之后确为 5 行脱离语句的 `// @TODO` 注释 ✓。④`FRegisterColor.cpp:30-31` 的 `// @TODO / DWColor` ✓。
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterColor.cpp:30-31`、`FRegisterFloatRange.cpp:18-24/28-30/38-39/42-45/54-58`、`FRegisterInt32Range.cpp:18-24/28-30/38-39/42-45/54-58`、`FRegisterFloatRange.cpp:18-24/28-30/38-39/42-45/54-58`、`FRegisterInt32Range.cpp:18-24/28-30/38-39/42-45/54-58`
- **函数**: `FRegisterColor::FRegisterColor()`、`FRegisterFloatRange::FRegisterFloatRange()` 等
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterColor.cpp:29-33
				.Property("Emerald", BINDING_READONLY_PROPERTY(&FColor::Emerald))
				// @TODO
				// DWColor
				.Function("FromRGBE", BINDING_FUNCTION(&FColor::FromRGBE))
```
```cpp
// FRegisterFloatRange.cpp:53-59
				.Function("Empty", BINDING_FUNCTION(&FFloatRange::Empty));
				// @TODO
				// Exclusive
				// GreaterThan
				// Inclusive
				// LessThan
		}
```
```cpp
// FRegisterFloatRange.cpp:18-30
				// @TODO
				// Adjoins
				// Conjoins
				// Contains
				// Contiguous
				// GetLowerBound
				// SetLowerBound
				.Function("SetLowerBoundValue", BINDING_FUNCTION(&FFloatRange::SetLowerBoundValue,
```

**调用上下文**
文件级注释；`FRegisterFloatRange.cpp:53` 的分号之后还有 5 行 `// @TODO`，**在语法上已不属于任何语句**，仅作为文件尾注释。

**问题**
1. `// @TODO` 与真实的链式调用交错，`FRegisterFloatRange.cpp:18-24` 的 TODO 列表看起来像是"下一段链路"，而 `:53` 之后的 TODO 列表则完全脱离代码块 —— 阅读者需要靠分号位置才能判断哪些已实现。`FRegisterFloatRange` 缺少 `GetLowerBound`/`SetLowerBound`/`GetUpperBound`/`SetUpperBound`/`Contains`/`Overlaps`/`Intersection`/`Hull`/`Difference`，**这是 `FFloatRange` 最常用的 API 面**，却以注释形式驻留在源码中，没有 issue 编号也没有条件编译保护。
2. `FRegisterColor.cpp:30-31` 的 `DWColor` 同理。
3. 由于这些是注释而非 `#if 0` 或明确的 FIXME(issue)，**不存在任何机制保证它们被跟进**。

**建议**
- 把 TODO 列表移到文件头的块注释里，并附 issue 编号：`// TODO(#123): GetLowerBound/SetLowerBound/Contains/Overlaps 未注册`；
- 对确实要保留但当前不可绑定的项（如 `FColor::DWColor` 依赖特定平台）用 `#if PLATFORM_*` 门控并附注释说明条件，而不是裸注释。

**验证方式**
`Select-String -Path Source/UnrealCSharp/Private/Domain/Interop/*.cpp -Pattern '@TODO' | Measure-Object`；`FRegisterFloatRange.cpp` 与 `FRegisterInt32Range.cpp` 的 TODO 行集合应完全一致（已核对：18-24 / 28-30 / 38-39 / 42-45 / 54-58 两文件逐行相同）。

---

### [F-INT2-027] `EKeys` 的 `EPropertyInteract::New` 是唯一例外且无注释

- **类别**: 可读性
- **严重度**: **P3**(风格)
- **复核结论**: 确认 —— 代码事实、行号、类别、严重度 P3 全部成立
- **可达性**: 活跃
- **复核证据**: ①`FRegisterKeys.cpp:116-118` **逐字吻合**（`:116 Semicolon`、**`:117 .Property("Equals", BINDING_PROPERTY(&EKeys::Equals, EPropertyInteract::New))`**、`:118 Comma`）。②342 条 `.Property` 中带第 4 个 vararg 的**确为唯一一处**（逐行通读 416 行确认）。③`BINDING_READONLY_PROPERTY` 同样支持第 4 参数 —— `BindingMacro.h:257`（`GET, nullptr, INFO, ##__VA_ARGS__`）**逐字吻合**，"修复 F-INT2-013 时需特别小心"的提醒成立。
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterKeys.cpp:117`
- **函数**: `FRegisterKeys::FRegisterKeys()`
- **置信度**: 中（"该参数用途"未在本机确认，见下）

**现状（代码事实）**
```cpp
// FRegisterKeys.cpp:116-118
				.Property("Semicolon", BINDING_PROPERTY(&EKeys::Semicolon))
				.Property("Equals", BINDING_PROPERTY(&EKeys::Equals, EPropertyInteract::New))
				.Property("Comma", BINDING_PROPERTY(&EKeys::Comma))
```

**调用上下文**
342 个 `.Property` 里唯一带第 4 个 vararg 的一处（`FClassBuilder.h:31-39` 的 `TOptional<EPropertyInteract> InPropertyInteract`）。

**问题**
从名字与位置（`EKeys::Equals` 会与 C# 的 `object.Equals` 冲突）推断，这是要求生成的 C# 成员加 `new` 关键字以消除 CS0108 警告。**但源码里没有任何注释说明**，而 F-INT2-013 已指出这 342 条本应全部使用 `BINDING_READONLY_PROPERTY`（后者同样支持第 4 个参数，见 `Source/UnrealCSharp/Public/Macro/BindingMacro.h:257`）—— 修复 F-INT2-013 时若照抄 `BINDING_PROPERTY(..., EPropertyInteract::New)` 会成为唯一一个"可写的 Equals"，需要特别小心。

**建议**
```cpp
// C# 的 object.Equals 与 EKeys.Equals 同名，需要 new 关键字消歧
.Property("Equals", BINDING_READONLY_PROPERTY(&EKeys::Equals, EPropertyInteract::New))
```

**验证方式**
`Select-String -Path FRegisterKeys.cpp -Pattern 'EPropertyInteract'` → 仅 :117 一处。

---

### [F-INT2-028] 静态注册对象依赖跨 TU 的静态初始化顺序（设计层面隐患，非当前 bug）

- **类别**: 并发/线程安全 / 未定义行为
- **严重度**: **P3**(隐患)
- **复核结论**: 部分确认（偏差：**第 1 点"未验证单例初始化方式"仍未闭合**（属其他报告范围，维持存疑）；第 2 点 TU 相对顺序对后缀编号的影响，**被生成产物与 `Algo::Count` 的语义边界进一步限定**：`Algo::Count(Functions, InName)` 计数的是**单个 `FClassBuilder` 实例内的 Functions 数组**，而每个 `FRegister*` 文件各建一个 builder，因此**跨 TU 根本不会互相计数**，"同名重载被拆到不同 TU 后缀会随链接顺序变化"这一担忧**在当前实现下不成立**（原文把它写成"隐含约束、无断言保护"，实际由 builder 实例边界天然保证）
- **可达性**: 活跃
- **复核证据**: ①末尾自注册对象**逐字吻合**：`FRegisterVector.cpp:344`（`:342-345` 为该 struct 与 `[[maybe_unused]] FRegisterVector RegisterVector;`）、`FRegisterKeys.cpp:415`、`FRegisterWorld.cpp:154`。②`FClassBuilder.cpp:83-91 GetFunctionImplementationName` 用 `Algo::Count(Functions, InName)`，`Functions` 是**该 builder 的成员**（原文第 71 行已引该函数）→ 计数范围受限于同一 builder，跨 TU 不串号。③旁证：`FRegisterMatrix.cpp:76/:78` 两条 `GetColumn` 在同一 TU，生成产物 `Matrix.cs:574/:590` 后缀为 `GetColumn`/`GetColumn1` 且**稳定** ✓。
- **级别变动**: 无（P3 维持）
- **文件**: 全部 36 个文件的末尾（如 `FRegisterVector.cpp:344`、`FRegisterKeys.cpp:415`、`FRegisterWorld.cpp:154`）
- **文件**: 全部 36 个文件的末尾（如 `FRegisterVector.cpp:344`、`FRegisterKeys.cpp:415`、`FRegisterWorld.cpp:154`）
- **函数**: 各 `FRegister*` 全局对象
- **置信度**: 中（当前实现是否真的依赖初始化顺序未验证）

**现状（代码事实）**
```cpp
// FRegisterVector.cpp:342-345
	};

	[[maybe_unused]] FRegisterVector RegisterVector;
}
```
```cpp
// FRegisterKeys.cpp:413-416
	};

	[[maybe_unused]] FRegisterKeys RegisterKeys;
}
```

**调用上下文**
每个匿名命名空间的全局对象在**模块加载（动态初始化）阶段**构造，构造函数体即注册全部条目（`FRegisterVector.cpp:89-341`）。

**问题**
1. 构造顺序在 **翻译单元之间不确定**。注册过程会触碰 `FBinding::Get()`（`FClassBuilder.cpp:11`）、`FCSharpEnvironment::GetEnvironment()`（`FRegisterDataTableFunctionLibrary.cpp:15` 等）。若这些单例采用函数内静态（C++11 起的线程安全惰性构造，magic static），则**安全**；若其中任何一个是普通全局对象，则在模块加载期就会命中"静态初始化顺序灾难"（读未初始化内存）。我在本批次范围内**未逐一验证这些单例的初始化方式**（属其他报告范围），因此标 置信度 中、且不作为 P0/P1。
2. 同理，36 个 TU 的相对注册顺序决定 `Algo::Count` 的后缀编号（`FClassBuilder.cpp:85`）。**同名重载一旦被拆到不同 TU，后缀会随链接顺序变化** → C# 侧函数名不稳定。目前 `FRegisterMatrix` 的 `GetColumn`/`GetColumn1` 都在同一 TU（`FRegisterMatrix.cpp:76/:78`），所以稳定；但这是"必须保证同 TU"的隐含约束，没有任何注释或断言保护。

**建议**
- 显式化：把注册入口改成一个 `FBinding::RegisterAll()`，由模块启动函数按确定顺序调用各 `RegisterXxx()`，替代全局对象自注册（同时消除初始化顺序依赖）；
- 或在 `FClassBuilder::GetFunctionImplementationName` 上加断言，要求同名重载必须来自同一 `FClassBuilder` 实例（当前实现天然满足，写出来可防回归）。

**验证方式**
`grep -rn "\[\[maybe_unused\]\] FRegister" Source/UnrealCSharp/Private/Domain/Interop/*.cpp | Measure-Object` → 与文件数一致的自注册对象数；检查 `FBinding::Get()` 与 `FCSharpEnvironment::GetEnvironment()` 是否函数内静态。

---

## 4. 死代码清单

### 4.1 结论

**在本批次 36 个文件中，未发现"无任何调用方的导出"这类经典死代码。** 原因是这批代码的性质与普通 C++ 不同：

- 每个 `.Function()/.Property()/.Constructor()` 都是在**把一个 native 函数登记进运行时名字表**（`FClassBuilder.cpp:78 → FBindingClassRegister::BindingMethod`），最终由 `FScriptDomainImpl.inl:601-633` 推给 C# 的 `MethodBridge`；
- C# 侧的被调方是**源生成器产物**，不在仓库里（`Script/` 下没有生成好的 `FVector.cs`/`FRotator.cs`，已用 `grep -rl "EKeys" Script/` 与 `grep -rl "FVector" Script/UE` 确认：`Script/UE/` 的 323 个文件是手写运行时 + Attribute 类，不含数学/值类型的绑定实现）；
- 因此"在 C# 侧 grep 函数名"**无法**证明任何一条注册是死的。

### 4.2 grep 证据表（能证明的部分）

| 符号/问题 | 搜索模式 | 命中数 | 判定 | 证据 |
|---|---|---|---|---|
| `FRegisterInputComponent` 注册名 vs C# 手写声明 | `grep -n "__UInputComponent_" Script/UE/Library/InputComponentImplementation.cs` | 8 | **完全对应，无死代码** | :8/:18/:27/:36/:45/:54/:63/:72 ↔ `FRegisterInputComponent.cpp:333-340` 的 8 个 `.Function` |
| `FRegisterEnhancedInputComponent` 同上 | `grep -n "__UEnhancedInputComponent_" EnhancedInputComponentImplementation.cs` | 3 | **完全对应，无死代码** | :9/:19/:30 ↔ `FRegisterEnhancedInputComponent.cpp:220-222` 的 3 个 `.Function` |
| `FRegisterUnreal` 同上 | `grep -n "__Unreal_" UnrealImplementation.cs` | 7 | **完全对应，无死代码** | :9/:20/:29/:39/:49/:58/:67 ↔ `FRegisterUnreal.cpp:181-187` 的 7 个 `.Function` |
| `FRegisterWorld` 同上 | `grep -n "__UWorld_" WorldImplementation.cs` | 1 | **对应，无死代码** | :8 ↔ `FRegisterWorld.cpp:150` |
| `FRegisterActorSpawnParameters`（`FRegisterWorld.cpp:107-112`） | `grep -n "__FActorSpawnParameters_" FActorSpawnParametersImplementation.cs` | 6 | **完全对应** | :7/:14/:21/:28/:35/:42 ↔ 6 个 getter/setter |
| `FRegisterDataTableFunctionLibrary` | `grep -rn "GetDataTableRowFromName" Script/` | 3 | **对应** | `FRegisterDataTableFunctionLibrary.cpp:47` ↔ `DataTableFunctionLibraryImplementation.cs:7/9/14` |
| `FRegisterKeys` 的 342 个属性名 | 名字/目标一致性审计（见下） | 342 全部 name==target，0 重复 | **无错位、无漏项、无重名** | 脚本审计：逐条提取 `.Property("X", ..., &EKeys::Y)`，统计 `X != Y` 为 0，重复名 0，重复目标 0 |
| `EKeys` 在 C# 侧 | `grep -rn "EKeys" Script/` | 0 | **C# 侧为生成产物**，不能据此判死 | — |
| 方法名缺失时的行为 | `MethodBridge.cs:140-148 GetMethod` | — | **名字查不到返回 0，调用点（`UnrealTypeSourceGenerator.cs:916-918`）不判空 → AV** | 说明"注册名错"不是无害的死代码，而是崩溃/静默错绑 |

### 4.3 因"注册名错"而事实上不可达的绑定（这才是本批次的"死"项）

| 应有的 API | 实际注册名 | 是否存在同名可达路径 | 判定 | 证据（文件:行） |
|---|---|---|---|---|
| `FMatrix::SetColumn` | C# 名为 `GetColumn`（thunk 键 `GetColumn1`） | **可达**：`FMatrix.GetColumn(int, FVector)` 是一对同名重载之一 | **命名缺口（非不可达）** | `FRegisterMatrix.cpp:78`；生成产物 `Script/UE/Proxy/Binding/Matrix.cs:590` |
| `FPolyglotTextData::SetNativeCulture` | C# 名为 `SetCategory` 的 `FString` 重载（thunk 键 `SetCategory1`） | **可达**：`SetCategory(FString)` 即它 | **命名缺口 + 静默改错目标** | `FRegisterPolyglotTextData.cpp:19`；生成产物 `PolyglotTextData.cs:61` |
| `FRotator::Equals(R, Tolerance)` | `Equals` 被 `IsNearlyZero` 占用（C# `Equals(double)`） | 真实 2 参 `Equals` **未绑定** | **被错误实现顶替** | `FRegisterRotator.cpp:46`；`Rotator.cs:196` |
| `FRotator::SetComponentForAxis` | `SetComponentForAxis` 指向 `GetComponentForAxis`（C# 返回 `double`、1 参） | 真实 setter 未绑定 | **被错误实现顶替** | `FRegisterRotator.cpp:65`；`Rotator.cs:380` |
| `FSoftObjectPath::GetAssetPathName` | 绑到 `GetAssetName`（C# `GetAssetPathName()` 返回短名） | 引擎 5.6 **无同名函数**，正确目标应为 `GetAssetPath()` | **被错误实现顶替 + 与 `GetAssetName` 重复** | `FRegisterSoftObjectPath.cpp:34`；`SoftObjectPath.cs:71` |
| `FVector::DotProduct`（静态） | C# 名为 `Dot` 的静态重载 | **可达**：`FVector.Dot(A, B)` | **名字劣化（与 `FVector2D.DotProduct` 不一致）** | `FRegisterVector.cpp:163`；`Vector.cs:797` |
| `FPolyglotTextData::IsMinimalPatch` setter 形参名 | `IsMinimalPatch(bool InCulture)` | **可达**，且**引擎里该 setter 本来就叫 `IsMinimalPatch`** | **仅形参名错（P3 级）** | `FRegisterPolyglotTextData.cpp:41`；`PolyglotTextData.cs:247`；引擎 `PolyglotTextData.h:126` |

---

## 5. 未覆盖 / 存疑项

1. ~~**无法核对 UE 引擎头文件**~~ → **【该前提不成立】** UE 5.6 引擎源码**确实存在于本机**：`Engine\Source`（约 20989 个 `.cpp`）。以"注册表只有 `4.0` 条目 / 未找到 `Math/` 源码"为由降级，是**错误的环境判断**，据其做出的降级已全部纠正。已用引擎源码关闭下列悬置项（括号内为**已核实的引擎行号**）：
   - `FQuat` 乘法顺序、`FTransform` 的 rotation/scale 组合顺序、`FMatrix` 行主序等**没有被本报告逐条核对**——本批次里**这些量全部是直接绑定引擎成员函数指针**（如 `FRegisterQuat.cpp:58-68` 绑 `FQuat::operator*`、`FRegisterTransform.cpp:31` 绑 `FTransform::operator*`），**没有任何一处自己重写数学**，因此"顺序写反"这一类静默错误在本架构下**结构上不可能出现**（除非 UE 自身语义如此）。反过来，本批次所有已确认的错绑都出现在"名字与目标函数不对应"上，与数学顺序无关。这是我做此判断的完整依据。
   - ~~`FSoftObjectPath::GetAssetPathName` 的返回类型未能核对~~ → **已核对并纠正**：`Runtime/CoreUObject/Public/UObject/SoftObjectPath.h` 中 **不存在 `AssetPathName` 标识符**（UE 5.6），只有 `:209 FTopLevelAssetPath GetAssetPath() const`、`:266 FString GetAssetName() const`、`:253 FString GetLongPackageName() const`。故 F-INT2-007 的性质由"绑错到另一个同义函数"修正为"绑到一个**引擎根本没有的名字**、且与 `:47 GetAssetName` 完全重复"（见该条复核证据）。
   - ~~`FPolyglotTextData` 是否存在 `void(const bool)` 形状的 `IsMinimalPatch` 重载未能核对~~ → **已核对**：`Runtime/Core/Public/Internationalization/PolyglotTextData.h:126 CORE_API void IsMinimalPatch(const bool InIsMinimalPatch)` **确实存在**，`:132 bool IsMinimalPatch() const` 也存在。**因此原报告"本意是 `SetIsMinimalPatch`"的推断被证伪** —— 引擎里这个 setter 本来就叫 `IsMinimalPatch`，`:41` 的绑定目标正确，只有 `ParamNames` 是复制残留。
   - ~~`FFrameNumber`/`FFrameTime` 的 `float` 标量重载是否已被 `double` 取代未能核对~~ → **已核对，结果是分裂的**：`Runtime/Core/Public/Misc/FrameNumber.h:66-67` **只有 `float` 重载**（且内部 `double(A.Value) * Scalar` + `FloorToDouble` + `Clamp` 到 int32）→ `FFrameNumber` 部分**确认为非缺陷**；而 `Runtime/Core/Public/Misc/FrameTime.h:232-245` **有 `double` 重载** → `FFrameTime` 部分**成立为 P3**。F-INT2-019 据此由"全条撤销"改为**拆分定性**。
2. ~~**`EKeys` 与引擎 `EKeys` 定义的"漏项"未逐条核对**~~ → **【复核已全量重跑，此悬置项关闭】** 结论：
   - 引擎 `EKeys` 的具名 `static const FKey` 常量（UE 5.6，`Runtime/InputCore/Classes/InputCoreTypes.h:263-659`）= **348 个**；另有数组式 `TouchKeys[11]`（`:661-662`，未暴露，属预期）。
   - 插件 `.Property` 注册条目（`FRegisterKeys.cpp:14-411`）= **342 条**，其中 **335 条在 UE 5.6 下生效**、**7 条被 `#if` 关闭**（`:202/:217/:315/:340/:343/:374/:377`，门控宏见 `CrossVersion/Public/UEVersion.h:46-58`，均为 `!UE_VERSION_START(5,1,0)`）。
   - **漏绑 13 条**（引擎有、插件无；**自洽校验：348−13=335 ✓**）。其中 **12 条是同族的 `*_2D` 聚合轴**（各组都注册了 `_X`/`_Y` 却漏掉配对的 `_2D`）：`Vive_Left/Right_Trackpad_2D`、`MixedReality_Left/Right_Thumbstick_2D`、`MixedReality_Left/Right_Trackpad_2D`、`OculusTouch_Left/Right_Thumbstick_2D`、`ValveIndex_Left/Right_Thumbstick_2D`、`ValveIndex_Left/Right_Trackpad_2D`；唯一非 2D 轴漏项是 **`Global_Back`**（引擎 `:480`，同组 `Global_Menu/View/Pause/Play` 均已注册）。
   - **多绑 7 条**（插件有、UE 5.6 无）：全部是被关闭的 `#if` 分支，属**面向 UE<5.1 的前向兼容**，机制正确，**UE 5.6 下 0 条实际多绑**。
   - **名字/大小写/别名**：342 条**全部 `name == &EKeys::目标`，0 条错位、0 条重名、0 条重复目标、0 处大小写差异**（逐行通读 416 行）；引擎侧的 `Asterix`/`LeftParantheses` 等**非常规拼写是引擎原样**，插件与之一致（非笔误）。
   - **生成产物交叉验证**：`Script/Game/Proxy/Binding/EKeys.cs` 中 `Global_Back`/`Vive_Left_Trackpad_2D`/`ValveIndex_Left_Trackpad_2D`/`Vive_Left_System_Click` **0 命中**，与上述"漏 13 / 多 7 关闭"完全吻合。
3. **`FRegisterKeys.cpp` 之外的文件未做全量名字/目标审计的语义确认**。已用脚本对全部 `FRegister*.cpp`（含非本批次文件）跑过"同一行内 `.Function("X", ..., &C::Y)` 且 `X != Y`"的扫描，命中 6 条：`FRegisterMatrix.cpp:78`、`FRegisterPolyglotTextData.cpp:19`、`FRegisterRotator.cpp:46`、`FRegisterRotator.cpp:65`、`FRegisterSoftObjectPath.cpp:34`、`FRegisterVector.cpp:163`（最后一条是 `Dot`→`DotProduct`，已在 F-INT2-018 单列）。**跨行书写（`BINDING_OVERLOAD(...)` 换行）的条目不在该扫描范围内**，仅靠人工阅读覆盖；`FRegisterPolyglotTextData.cpp:40-42` 是我人工发现的跨行案例。
4. **`TFunctionBuilder::Invoke` / `TArgument` / `TPropertyBuilder` 的完整实现只读了与本批次直接相关的部分**（`SignatureMacro.h`、`TArgument.inl:28-29/59-60`、`TReturnValue.inl:18-33`、`TPropertyBuilder.inl:102-108/211-216/404-421`）。`IN_BUFFER`/`OUT_BUFFER`/`RETURN_BUFFER` 都是裸 `uint8*` 且**在 thunk 中无任何判空**（`TArgument.inl:29` 直接 `*(std::decay_t<Type>*)IN_BUFFER`），这是全模块级别的隐患，但责任文件不在我范围内，仅在此记录。
5. **未运行任何构建或测试**。所有结论均为静态阅读 + grep/脚本审计得出的；F-INT2-013（`EKeys` 可写属性写 const 全局）的"崩溃 vs 静默篡改"取决于 MSVC 的段分配，未经运行验证。
6. **未核对 C# 生成产物的实际形态**。生成器只在编辑器构建且 `WITH_FUNCTION_INFO=WITH_EDITOR`（`BindingMacro.h:17`）时运行，仓库中没有生成结果（已确认），所以"参数名多一项到底让 C# 签名变成什么样"这类细节我以"元数据与真实函数不一致"（确定）+"生成结果可疑"（中）两级表述，未做断言。

---

## 6. 逐文件职责、导出清单与结论

> 本节是任务书"每个文件都要有职责/导出清单（行号）/结论"的要求。行号全部来自 `read` 输出。条目数以 `.Constructor/.Function/.Property/.Subscript` 出现次数统计（脚本，见第 0 节表）。**超长文件（FRegisterVector/Transform/Quat/Keys）的清单按"连续行段 + 段内主题"给出，未逐条罗列全部条目**，这一点在第 0 节已声明。

### 6.1 数学核心

**`FRegisterVector.cpp`（345 行，133 条目）** — 职责：把 `FVector` 的构造/算术/几何工具函数暴露给 C#。
- 辅助实现：`:9 BitOrImplementation`、`:14/:19/:24 MinusImplementation(int32/float/double)`、`:29/:34/:39 PlusImplementation`、`:44/:49/:54/:59/:64/:69 MultipliesImplementation`、`:74/:79/:84 DividesImplementation`
- 构造：`:92`(FReal)、`:94`(XYZ)、`:96`(FVector2D+FReal)、`:98`(FVector4)、`:100`(FLinearColor)
- 运算符宏：`:102 BitXor`、`:103 Plus`、`:104 Minus`、`:105 Multiplies`、`:106 Divides`、`:107 UnaryMinus`、`:108 Subscript`
- 显式运算符重载：`:110 operator|`、`:111-116 operator-`、`:117-122 operator+`、`:123-134 operator*`、`:135-140 operator/`
- 静态常量（只读）：`:141-151`（ZeroVector…ZAxisVector）
- 静态工厂：`:152-156`（Zero/One/UnitX/UnitY/UnitZ）
- 几何：`:157-164`（Cross/CrossProduct/**Dot/Dot**，见 F-INT2-018）、`:165-204`（Equals…GetSafeNormal2D）、`:207-212`（ToDirectionAndLength 两个重载）、`:213-252`（Projection…UnwindEuler）、`:267-285`（PointsAreSame…VectorPlaneProject）、`:286-309`（Dist 族、Min/Max/Min3/Max3）、`:310-330`（Parallel/Coincident/Orthogonal/Coplanar/Triple）、`:331-340`（EvaluateBezier…GenerateClusterCenters）
- 结论：**1 条 P2（F-INT2-018 `Dot1`）+ 1 条 P2（F-INT2-016 恒真判空）**；数学语义全部由引擎成员函数直接绑定，无自写运算。

**`FRegisterVector2D.cpp`（183 行，60 条目）** — 职责：`FVector2D` 的构造/算术/比较/几何。
- 辅助实现：`:11/:16/:21/:26 MultipliesImplementation`、`:31 DividesImplementation`、`:36/:41 Plus/Minus`、`:46 BitOr`、`:51 BitXor`（后四者含对标量"判空"，见 F-INT2-016）
- 构造：`:59`(XY)、`:61`(FReal)、`:63`(FIntPoint)、`:65`(EForceInit)、`:66`(FVector)、`:68`(FVector4)
- 运算符：`:70-73`（Plus/Minus/Multiplies/Divides）、`:74-85`（UE<5.1 门控的 Less/Greater/LessEqual/GreaterEqual）、`:86 UnaryMinus`、`:87 Subscript`、`:89-97`（operator* 四重载）、`:98-102`（operator/ + - | ^）
- 静态常量：`:103-105`（ZeroVector/UnitVector/Unit45Deg，只读）
- 其余：`:106-110`（Zero/One/UnitX/UnitY）、`:111-121`（`UE_F_VECTOR2_COMPONENT_WISE_*` 门控的 ComponentwiseAll*）、`:122-178`
- 结论：**无 P0/P1**。UE 版本门控（`:74/:77/:80/:83`、`:110/:113/:116/:119`）是本批次里做得最规范的文件，可作为其它文件的模板。

**`FRegisterVector4.cpp`（132 行，37 条目）** — 职责：`FVector4` 的构造/算术/归一化/方向转换。
- 辅助实现：`:10/:15/:20/:25/:30/:35 MultipliesImplementation`、`:40/:45/:50 DividesImplementation`
- 构造：`:58`(FVector)、`:60`(FLinearColor)、`:62`(FLinearColor+FReal)、`:64`(XYZW)、`:67`(FVector2D×2)、`:69`(EForceInit)
- 运算符：`:70 Subscript`、`:72 UnaryMinus`、`:73-77`（Plus/Minus/Multiplies/Divides/BitXor）、`:78-95`（operator* / 除各 3~6 重载）
- 其余：`:96-127`（Zero/One/Component/Equals/IsUnit3/ToString/InitFromString/GetSafeNormal/GetUnsafeNormal3/ToOrientation*/Rotation/Set/Size*/ContainsNaN/IsNearlyZero3/Reflect3/FindBestAxisVectors3）
- 结论：无 P0/P1；仅含 F-INT2-016 的恒真判空。

**`FRegisterRotator.cpp`（99 行，39 条目）** — 职责：`FRotator` 的构造/加减乘/轴向读写/角度归一化/压缩编码。
- 辅助实现：`:10/:15/:20 MultipliesImplementation`
- 构造：`:28`(FReal)、`:30`(Pitch/Yaw/Roll)、`:32`(EForceInit)、`:33`(FQuat)
- 运算符：`:35 Plus`、`:36 Minus`、`:37-42`（operator* 三重载）
- 其余：`:43 IsNearlyZero`、`:45 IsZero`、`:46`**Equals（错绑，F-INT2-006）**、`:48 Add`、`:50 GetInverse`、`:51 GridSnap`、`:53-59 Vector/Quaternion/Euler/RotateVector/UnrotateVector`、`:60-62 Clamp/GetNormalized/GetDenormalized`、`:63 GetComponentForAxis`、`:65`**SetComponentForAxis（错绑，F-INT2-005）**、`:67 Normalize`、`:68-74`（GetWindingAndRemainder…SetClosestToMe）、`:75-80`（ToString/ToCompactString/InitFromString/ContainsNaN）、`:81-92`（ClampAxis/NormalizeAxis/Compress/DecompressAxis*）、`:93 MakeFromEuler`
- 结论：**本批次问题最集中的文件（2 条 P1）**。注意 `:35 Plus`/`:36 Minus` 用的是运算符宏，**没有 `Divides`** —— 这与引擎是否定义 `FRotator operator/` 相关，未能核对（第 5 节），仅记录为不对称。

**`FRegisterTransform.cpp`（183 行，85 条目）** — 职责：`FTransform` 的构造/组合/正逆变换/分解/比较。
- 辅助实现：`:9 MultipliesImplementation(FTransform, FQuat)`
- 构造：`:17`(FVector 平移)、`:19`(FRotator)、`:21`(FQuat+FVector+FVector)、`:23`(FRotator+FVector+FVector)、`:25`(FMatrix)、`:27`(四向量)
- 运算符：`:29 Plus`、`:30 Multiplies`、`:31 operator*（FQuat 重载）`
- 其余：`:32 Identity`、`:33-42`（DebugPrint/ToHumanReadableString/ToString/InitFromString/ToMatrixWithScale/ToInverseMatrixWithScale/Inverse/ToMatrixNoScale）、`:43-64`（Blend/BlendWith/AnyHasNegativeScale/ScaleTranslation×2/RemoveScaling/GetMax·MinAxisScale/GetRelativeTransform(+Reverse)/SetToRelativeTransform）、`:65-89`（TransformFVector4(NoScale)/TransformPosition(NoScale)/InverseTransformPosition(NoScale)/TransformVector(NoScale)/InverseTransformVector(NoScale)/TransformRotation/InverseTransformRotation）、`:90-104`（GetScaled×2/GetScaledAxis/GetUnitAxis/Mirror/GetSafeScaleReciprocal）、`:105-178`（GetLocation…SetFromMatrix）
- 结论：**无 P0/P1**。`rotation → translation → scale` 的组合顺序全部委托给引擎（`FTransform::operator*`、`Blend`、`Multiply`），无自写顺序，结构上不可能出现"组合顺序写反"。

**`FRegisterMatrix.cpp`（104 行，50 条目）** — 职责：`FMatrix` 的构造/乘法/平移旋转缩放分解/坐标轴读写/投影平面提取。
- 辅助实现：`:10 MultipliesImplementation(FMatrix, FReal)`
- 构造：`:18`(EForceInit)、`:19`(四 FPlane)、`:21`(四 FVector)
- 运算符：`:23 Multiplies`、`:24 Plus`、`:26 operator*（标量）`
- 其余：`:25 Identity`、`:27-44`（SetIdentity…Inverse）、`:45-60`（TransposeAdjoint…GetMaximumAxisScale）、`:61-77`（GetOrigin/GetScaledAxis(es)/GetUnitAxis(es)/SetAxis/SetOrigin/SetAxes/GetColumn/**GetColumn(SetColumn)**）、`:80-99`（Rotator/ToQuat/GetFrustum*Plane×6/Mirror/ToString/DebugPrint/ComputeHash）
- 结论：**1 条 P1（F-INT2-003）**。行列主序问题：本文件的 `TransformPosition`/`TransformVector`/`GetColumn`/`SetColumn` 等全部为引擎成员函数直绑（`:30-77`），**没有任何按 `M[i][j]` 的 C# 侧下标运算被暴露**（`FMatrix` 未注册 `Subscript`），因此"行主序 vs 列主序访问导致转置错误"在本绑定下没有入口。

**`FRegisterQuat.cpp`（160 行，64 条目）** — 职责：`FQuat` 的构造/乘除/欧拉角/轴角/插值/旋转向量。
- 辅助实现：`:10 Multiplies(FQuat,FVector)`、`:15/:20 Multiplies(FQuat,float/double)`、`:25/:30 Divides`、`:35 BitOr`
- 构造：`:43`(EForceInit)、`:44`(XYZW)、`:46`(FReal)、`:48`(FRotator)、`:50`(FMatrix)、`:52`(FVector Axis + AngleRad)
- 运算符：`:54 Plus`、`:55 Minus`、`:56 UnaryMinus`、`:57 Multiplies`、`:58-63`（operator*：FVector/float/double）、`:64-67`（operator/）、`:68 operator|`
- 其余：`:69 Identity`、`:70-93`（MakeFromRotator/Equals/IsIdentity/Identical/MakeFromEuler/Euler/Normalize/GetNormalized/IsNormalized/Size/SizeSquared/GetAngle/ToAxisAndAngle×2）、`:94-131`（ToSwingTwist…FindBetweenVectors）、`:132-155`（Error/ErrorAutoNormalize/FastLerp/FastBilerp/Slerp_NotNormalized/Slerp/SlerpFullPath*/Squad/SquadFullPath/CalcTangents）
- 结论：**无 P0/P1**。四元数乘法顺序（`:58 Multiplies` → 引擎 `FQuat::operator*`）、欧拉角顺序（`:77 MakeFromEuler` / `:79 Euler`）全部直绑引擎，无自写；插值族（`:136-152`）区分了 `Slerp` 与 `SlerpFullPath`、`_NotNormalized` 变体，与引擎语义一一对应，未发现"Lerp 与 Slerp 混用"类错误。

### 6.2 颜色 / 平面 / 2D 盒

**`FRegisterColor.cpp`（70 行，37 条目）** — 职责：`FColor`（8 位 sRGB 字节色）的构造、命名色、打包/量化转换。
- 构造：`:12`(RGBA uint8)、`:14`(uint32)
- 命名色（只读）：`:16-29`
- 静态/成员：`:32 FromRGBE`、`:33 FromHex`、`:35 MakeRandomColor`、`:36 MakeRedToGreenColorFromScalar`、`:38 MakeFromColorTemperature`、`:40-53`（Quantize/Dequantize/Requantize 族）、`:54 WithAlpha`、`:56 ReinterpretAsLinear`、`:57 ToHex`、`:58 ToString`、`:60 InitFromString`、`:62-65`（ToPackedARGB/ABGR/RGBA/BGRA）
- 结论：**无 P0/P1**。sRGB 线性转换**同时提供了两个方向的正确入口**——`FRegisterColor.cpp:56 ReinterpretAsLinear` 与 `FRegisterLinearColor.cpp:44 FromSRGBColor`/`:73 ToFColorSRGB`/`:74 ToFColor(bSRGB)`，且把"是否做 sRGB 转换"显式作为参数（`ToFColor(bool bSRGB)`），排除了"遗漏线性转换"的隐患。`:30-31` 的 `// @TODO DWColor`（F-INT2-026）。

**`FRegisterLinearColor.cpp`（90 行，39 条目）** — 职责：`FLinearColor`（线性浮点色）的构造、算术、HSV/色温/亮度、量化。
- 辅助实现：`:11 MultipliesImplementation`、`:16 DividesImplementation`
- 构造：`:24`(EForceInit)、`:25`(float×4)、`:27`(FColor)
- 运算符：`:29 Plus`、`:30 Minus`、`:31 Multiplies`、`:32 Divides`、`:33 operator*`、`:34 operator/`
- 命名色（只读）：`:35-42`
- 其余：`:43 ToRGBE`、`:44 FromSRGBColor`、`:46 FromPow22Color`、`:48 Component`、`:51 GetClamped`、`:53 Equals`、`:55 CopyWithNewOpacity`、`:57 MakeFromHSV8`、`:59 MakeRandomColor`、`:60 MakeFromColorTemperature`、`:62 Dist`、`:64 EvaluateBezier`、`:67 LinearRGBToHSV`、`:68 HSVToLinearRGB`、`:69 LerpUsingHSV`、`:71 QuantizeRound`、`:72 QuantizeFloor`、`:73 ToFColorSRGB`、`:74 ToFColor`、`:76 Desaturate`、`:78 GetLuminance`、`:79 GetMax`、`:80 IsAlmostBlack`、`:81 GetMin`、`:82 ToString`、`:84 InitFromString`
- 结论：无 P0/P1；`:3` 重复 include（F-INT2-024）。

**`FRegisterPlane.cpp`（73 行，20 条目）** — 职责：`FPlane` 的构造、点积、法线/原点、翻面、矩阵变换。
- 辅助实现：`:10 BitOrImplementation`、`:15 DividesImplementation`、`:20/:25 MultipliesImplementation`（后者非 const 引用，F-INT2-020）
- 构造：`:33`(FVector4)、`:35`(XYZW)、`:37`(FVector+FReal)、`:39`(FVector Base+Normal)、`:41`(A/B/C)、`:43`(EForceInit)
- 运算符：`:44 Plus`、`:45 Minus`、`:46 operator|`、`:47 operator/`、`:48/:50 operator*`×2
- 其余：`:52 IsValid`、`:53 GetOrigin`、`:54 GetNormal`、`:55 PlaneDot`、`:57 Normalize`、`:60 Flip`、`:61 TransformBy`、`:63 TransformByUsingAdjointT`、`:65 TranslateBy`、`:67 Equals`
- 结论：**1 条 P2（F-INT2-020 非 const 引用重载）**。

**`FRegisterBox2D.cpp`（63 行，22 条目）** — 职责：`FBox2D` 的构造、扩张、包含/相交、最近点。
- 辅助实现：`:11 PlusImplementation`
- 构造：`:19`(EForceInit)、`:20`(InMin/InMax)、`:22`**(const FVector2D*, int32 — F-INT2-014)**、`:24`(TArray<FVector2D>)
- 运算符：`:26 Plus`、`:27 Subscript`、`:29 operator+`
- 其余：`:30 ComputeSquaredDistanceToPoint`、`:32/:35 ExpandBy`×2（后者 `UE_F_BOX_2D_EXPAND_BY_VECTOR2` 门控）、`:38 GetArea`、`:39 GetCenter`、`:40 GetCenterAndExtents`、`:42 GetClosestPointTo`、`:44 GetExtent`、`:45 GetSize`、`:46 Init`、`:47 Overlap`、`:49 Intersect`、`:51/:53 IsInside`×2、`:55 ShiftBy`、`:57 ToString`
- 结论：**1 条 P1（F-INT2-014 裸指针+count 构造）**，其余无问题。

### 6.3 GUID / 时间

**`FRegisterGuid.cpp`（87 行，14 条目）** — 职责：`EGuidFormats` 枚举 + `FGuid` 的构造、比较、格式化、解析。
- 枚举：`:7 BINDING_ENUM(EGuidFormats)`、`:15-24`（9 个 Enumerator）
- 辅助实现：`:32 GreaterThanImplementation`（手抄 UE operator>，F-INT2-023）、`:42 LexToStringImplementation`
- 构造：`:50`(uint32×4)、`:52`(const FString&)
- 其余：`:54 Less`、`:55 Subscript`、`:57 operator>`、`:58 LexToString`、`:60 Invalidate`、`:61 IsValid`、`:63 ToString()`（`UE_F_GUID_TO_STRING` 门控，UE<5.1）、`:66 ToString(EGuidFormats)`、`:68 NewGuid`、`:70/:73 Parse`（`UE_F_GUID_PARSE_F_STRING_F_GUID` 门控）、`:76/:80 ParseExact`
- 结论：**无 P0/P1**。字符串解析/格式化与 UE 一致（大小写、连字符、格式枚举全部转发给引擎 `FGuid::Parse/ParseExact/ToString`）；无效输入的安全路径由 UE 保证：`FGuid(const FString&)` 在解析失败时会给出全零 GUID（引擎行为），**不会崩溃**，但 C# 侧会**静默拿到一个 Invalid GUID** —— 建议 C# 封装层用返回 `bool` 的 `Parse` 而非该构造（本文件 `:70` 已导出 `Parse`，具备正确路径）。`:62/:69/:75` 三处版本门控处理得当。

**`FRegisterDateTime.cpp`（149 行，42 条目）** — 职责：`EDayOfWeek`/`EMonthOfYear` 枚举 + `FDateTime`（int64 ticks）的构造、算术、比较、格式化、解析、Unix/Julian 转换。
- 枚举：`:7 BINDING_ENUM(EDayOfWeek)`、`:9 BINDING_ENUM(EMonthOfYear)`、`:17-24`（7 天）、`:34-46`（12 月）
- 辅助实现：`:54 Plus(FDateTime,FTimespan)`、`:60 Plus(FDateTime&,FDateTime&)`（`UE_F_DATETIME_PLUS` 门控，返回引用，见下）、`:66 Minus(FDateTime,FDateTime)→FTimespan`、`:71 Minus(FDateTime,FTimespan)`
- 构造：`:79`(int64 ticks)、`:81`(Y/M/D/h/m/s/ms 七参)
- 运算符：`:83 Greater`、`:84 GreaterEqual`、`:85 Less`、`:86 LessEqual`、`:87-96`（operator+ ×2 / operator- ×2）
- 其余：`:97/:98 GetDate`×2、`:101-119`（GetDay…ToIso8601）、`:120 ToString()`、`:122 ToUnixTimestamp`、`:123-132`（DaysInMonth/DaysInYear/FromJulianDay/FromUnixTimestamp/IsLeapYear）、`:133-141`（MaxValue/MinValue/Now/Parse/ParseHttpDate/Today/UtcNow）、`:142 Validate`
- 结论：**无 P0/P1**。精度方面：tick 类型为 `int64`（`:79`），没有 `float` 往返，大时间值不会丢精度；`FTimespan` 差值（`:66`）也走 int64。`:60-63` 的 `PlusImplementation(FDateTime& In, const FDateTime& Other)` 返回 `FDateTime&` 且只受 `UE_F_DATETIME_PLUS`（UE<5.1）门控——**该重载在 UE<5.1 的语义是 `operator+=` 式的原地修改还是值运算，我未核对引擎头文件**；若为原地修改而返回引用被当作"新值"使用，会产生与 F-INT2-005 同型的静默错误。**标为存疑（置信度 低），不单列为发现**，因为它在 UE≥5.1 下被门控掉（当前主流版本不受影响）。

**`FRegisterTimespan.cpp`（82 行，35 条目）** — 职责：`FTimespan`（int64 ticks）的构造、算术、比例、格式化、解析。
- 辅助实现：`:9 MultipliesImplementation(double)`、`:14 DividesImplementation(double)`
- 构造：`:22`(int64)、`:24`(h/m/s)、`:26`(d/h/m/s)、`:28`(d/h/m/s/nano)
- 运算符：`:30 Plus`、`:31 UnaryMinus`、`:32 Minus`、`:33 Modulus`、`:34-37`（比较四则）、`:38 operator*`、`:39 operator/`
- 其余：`:40-56`（GetDays…IsZero）、`:57 ToString`、`:59-70`（FromDays/FromHours/FromMicroseconds/FromMilliseconds/FromMinutes/FromSeconds）、`:71 MaxValue`、`:72 MinValue`、`:73 Parse`、`:75 Ratio`、`:77 Zero`
- 结论：无 P0/P1；`:9/:14` 使用 `double` 标量（与 F-INT2-019 的 `float` 形成正面参照）。

**`FRegisterFrameNumber.cpp`（40 行，3 条目）** — 职责：`FFrameNumber`（int32 帧号）的构造、自增减、比较、算术。
- 辅助实现：`:9 MultipliesImplementation(float)`、`:14 DividesImplementation(float)`
- 构造：`:22`(int32)
- 运算符：`:24 PreIncrement`、`:25 PreDecrement`、`:26-29`（Less/Greater/LessEqual/GreaterEqual）、`:30 Plus`、`:31 Minus`、`:32 Modulus`、`:33 UnaryMinus`、`:34 operator*`、`:35 operator/`
- 结论：**1 条 P2（F-INT2-019 float 精度）**，其余无问题。

**`FRegisterFrameTime.cpp`（51 行，13 条目）** — 职责：`FFrameTime`（帧号 + float 子帧）的构造、比较、算术、取整。
- 辅助实现：`:9 MultipliesImplementation(float)`、`:14 DividesImplementation(float)`
- 构造：`:22`(int32)、`:24`(FFrameNumber)、`:26`(FFrameNumber+float SubFrame)
- 运算符：`:28-35`（比较四则/Plus/Minus/Modulus/UnaryMinus）、`:36 operator*`、`:37 operator/`
- 其余：`:38 MaxSubframe`、`:39 GetFrame`、`:40 GetSubFrame`、`:41-43`（Floor/Ceil/RoundToFrame）、`:44 AsDecimal`、`:45 FromDecimal`
- 结论：**1 条 P2（F-INT2-019）**。`FFrameTime` 本身以 `float SubFrame` 表示小数帧（`:26` 构造），其精度上限是引擎设计决定的，绑定的 `float` 标量与之匹配；风险主要在 `FFrameNumber` 那一侧。

### 6.4 随机 / 区间 / 范围

**`FRegisterRandomStream.cpp`（52 行，20 条目）** — 职责：`FRandomStream` 的构造、种子管理、各类随机采样。
- 构造：`:12`(int32 InSeed)、`:14`(FName InName)
- 种子/状态：`:16/:18 Initialize`×2、`:20 Reset`、`:21 GetInitialSeed`、`:22 GenerateNewSeed`、`:26 GetCurrentSeed`
- 采样：`:23 GetFraction`、`:24 GetUnsignedInt`、`:25 GetUnitVector`、`:27 FRand`、`:28 RandHelper`、`:30 RandRange`、`:32 FRandRange`、`:34 VRand`、`:35 RandPointInBox`、`:37/:41 VRandCone`×2
- 其它：`:46 ToString`
- 结论：**无 P0/P1**。就任务书关心的"是否每次调用都重置种子"作了专门核对：本文件**只暴露 `FRandomStream` 的成员函数**（`:20-45` 全部是 `FRandomStream::*`），**没有任何一处暴露 `FMath::Rand*` 之类的全局随机函数**，因此"每次调用重新播种导致结果相同"的隐患不存在；种子在 C# 持有的对象内（`:12/:14/:16/:18`），跨调用保持。跨平台确定性由引擎的 `FRandomStream` 实现保证，绑定层不介入。

**`FRegisterFloatInterval.cpp`（28 行，7 条目）** — 职责：`FFloatInterval`（Min/Max 闭区间）的构造与基本操作。
- 构造：`:12`(float InMin, float InMax)
- 其余：`:14 Size`、`:15 IsValid`、`:16 Contains`、`:18 Expand`、`:20 Include`、`:22 Interpolate`
- 结论：无 P0/P1。

**`FRegisterInt32Interval.cpp`（28 行，7 条目）** — 职责：同上的 `FInt32Interval` 版本。`:12` 构造、`:14 Size`、`:15 IsValid`、`:16 Contains`、`:18 Expand`、`:20 Include`、`:22 Interpolate`
- 结论：无 P0/P1；**与 `FRegisterFloatInterval.cpp` 除类型名外逐行相同** → 抽公共模板的候选（F-INT2-022 建议 3）。

**`FRegisterFloatRange.cpp`（63 行，17 条目）** — 职责：`FFloatRange`（可开可闭的上下界）的构造、边界值读写、集合运算。
- 构造：`:12`(const float&)、`:14`(const float&, const float&)、`:16`(Bound, Bound)
- 其余：`:25 SetLowerBoundValue`、`:27 GetLowerBoundValue`、`:31 SetUpperBoundValue`、`:33 GetUpperBoundValue`、`:34 HasLowerBound`、`:35 HasUpperBound`、`:36 IsDegenerate`、`:37 IsEmpty`、`:40 Split`、`:46 Union`、`:48 All`、`:49 AtLeast`、`:51 AtMost`、`:53 Empty`
- 结论：**无 P0/P1，但 API 覆盖不足**：`GetLowerBound`/`SetLowerBound`/`GetUpperBound`/`SetUpperBound`/`Contains`/`Overlaps`/`Contiguous`/`Adjoins`/`Conjoins`/`Intersection`/`Hull`/`Difference`/`Exclusive`/`GreaterThan`/`Inclusive`/`LessThan` 全部只以 `// @TODO` 注释存在（`:18-24`、`:28-30`、`:38-39`、`:42-45`、`:54-58`）→ F-INT2-026。`Contains` 缺失尤其影响可用性（只给了 `Split`/`Union`）。`:53` 语句结束后仍有悬挂注释段。

**`FRegisterInt32Range.cpp`（63 行，17 条目）** — 职责：`FInt32Range` 版本。`:12/:14/:16` 构造、`:25/:27/:31/:33` 边界值、`:34-37` 判定、`:40 Split`、`:46 Union`、`:48-53` 静态构造
- 结论：与 `FRegisterFloatRange.cpp` **逐行同构**（TODO 行集合完全一致），同一并处理（F-INT2-022/026）。

**`FRegisterFloatRangeBound.cpp`（40 行，15 条目）** — 职责：`FFloatRangeBound`（值 + 开闭标志）的构造与比较。
- 构造：`:12`(const float& InValue)
- 其余：`:14 GetValue`、`:15 SetValue`、`:17 IsClosed`、`:18 IsExclusive`、`:19 IsInclusive`、`:20 IsOpen`、`:21 Exclusive`、`:23 Inclusive`、`:25 Open`、`:26 FlipInclusion`、`:28 MaxLower`、`:30 MaxUpper`、`:32 MinLower`、`:34 MinUpper`
- 结论：无 P0/P1。

**`FRegisterInt32RangeBound.cpp`（40 行，15 条目）** — 职责：`FInt32RangeBound` 版本，行号与上表逐一对应。
- 结论：无 P0/P1；同上，与 Float 版逐行同构。

### 6.5 资产标识 / 路径 / 本地化

**`FRegisterPrimaryAssetId.cpp`（38 行，9 条目）** — 职责：`FPrimaryAssetId`（Type+Name）的构造、解析、有效性、字符串化。
- 版本门控：`:3-5`（`UE_U_STRUCT_F_PRIMARY_ASSET_ID`）、`:7-9 BINDING_STRUCT`
- 构造：`:18`(FPrimaryAssetType, FName)、`:20`(const FString& TypeAndName)
- 属性（只读）：`:22 PrimaryAssetTypeTag`、`:23 PrimaryAssetNameTag`
- 其余：`:24/:27 ParseTypeAndName`×2（FName / FString）、`:30 IsValid`、`:31 ToString`、`:33 FromString`
- 结论：**无 P0/P1**。字符串解析失败路径**安全**：`ParseTypeAndName`/`FromString` 是引擎的纯函数，失败返回无效（空）`FPrimaryAssetId`（不是崩溃），C# 需用 `IsValid()`（`:30` 已导出）判定 —— 这是正确用法，但建议在 C# 封装层把"解析失败"提升为可观测事件。格式问题见 F-INT2-025（`:21` 与 `:33` 的参数名写法不一致）。

**`FRegisterPrimaryAssetType.cpp`（28 行，4 条目）** — 职责：`FPrimaryAssetType`（FName 包装）的构造、取值、有效性、字符串化。
- 版本门控：`:3-5`、`:7-9`
- 构造：`:18`(FName InName)
- 其余：`:20 GetName`、`:21 IsValid`、`:22 ToString`
- 结论：无 P0/P1。无字符串解析入口 → 无"解析失败路径"问题（要解析得先过 `FPrimaryAssetId`）。

**`FRegisterSoftClassPath.cpp`（32 行，5 条目）** — 职责：`FSoftClassPath` 的构造、解析为类、由类生成路径。
- 版本门控：`:4-10`
- 构造：`:19`(拷贝，**参数名多一项 `"x"` — F-INT2-017**)、`:21`(const FString& PathString)、`:23`(const UClass*)
- 其余：`:25 ResolveClass`、`:26 GetOrCreateIDForClass`
- 结论：**1 条 P2（F-INT2-017）**。无效路径安全性：构造与 `ResolveClass` 都是引擎成员函数，**无效路径返回 nullptr/空路径而非崩溃**，绑定层无需额外校验；`ResolveClass` 的返回值在 C# 侧需判空（属生成产物行为，未核对）。

**`FRegisterSoftObjectPath.cpp`（83 行，29 条目）** — 职责：`FSoftObjectPath` 的构造、路径分量读写、对象解析、PIE 包名管理。
- 版本门控：`:3-9`（`UE_U_STRUCT_F_SOFT_OBJECT_PATH` 等 6 组）
- 构造：`:18`(拷贝)、`:20`(const FString&)、`:23`(FName，`UE_F_SOFT_OBJECT_PATH_CONSTRUCTOR_F_NAME`)、`:27`(FName+FString，`..._F_NAME_F_STRING`)、`:30`(const UObject*)
- 其余：`:32 ToString`、`:34`**GetAssetPathName（错绑，F-INT2-007）**、`:36 SetAssetPathName`、`:39 GetAssetPathString`、`:40 GetSubPathString`、`:42 SetSubPathString`、`:45 GetLongPackageName`、`:46 GetLongPackageFName`、`:47 GetAssetName`、`:49/:52 SetPath`×2、`:55 ResolveObject`、`:56 Reset`、`:57 IsValid`、`:58 IsNull`、`:59 IsAsset`、`:60 IsSubobject`、`:61 FixupCoreRedirects`、`:63 GetCurrentTag`、`:66 InvalidateTag`、`:69/:73 GetOrCreateIDForObject`、`:76 AddPIEPackageName`、`:78 ClearPIEPackageNames`
- 结论：**1 条 P1（F-INT2-007）**。无效路径路径**安全**：`ResolveObject`（`:55`）返回 nullptr、`IsValid/IsNull`（`:57/:58`）提供判定，构造不会崩。`:76/:78 AddPIEPackageName/ClearPIEPackageNames` 修改的是**引擎全局静态状态**（PIE 包名前缀表）且**只能在 GameThread 调用**，本文件未做线程断言；标为存疑（第 5 节第 4 项同类问题），未单列发现。

**`FRegisterAssetBundleData.cpp`（48 行，7 条目）** — 职责：`FAssetBundleData`（BundleName → 软对象路径集合）的构造、查找、添加、调试串。
- 版本门控：`:4-10`
- 构造：`:19`(拷贝)
- 其余：`:20/:21 FindEntry`（返回 `FAssetBundleEntry*` — **返回内部指针**）、`:25 AddBundleAsset`（`UE_F_ASSET_BUNDLE_DATA_ADD_BUNDLE_ASSET`）、`:31 AddBundleAssets`（`..._ADD_BUNDLE_ASSETS`）、`:37 SetBundleAssets`（`..._SET_BUNDLE_ASSETS`，`TArray<FSoftObjectPath>&&` 右值重载）、`:42 Reset`、`:43 ToDebugString`
- 结论：**无 P0/P1，但有生命周期隐患（未单列为发现，置信度 低）**：`:21 FindEntry` 返回指向 `FAssetBundleData` 内部容器的 `FAssetBundleEntry*`。若 C# 侧把该指针包装为对象并随后对同一 `FAssetBundleData` 调用 `:25/:31/:37` 添加元素导致 `TArray` 重分配，C# 持有的指针即悬垂。`AddReference<T, IsNeedFree>` 的 `IsNeedFree` 模板参数（`FCSharpEnvironment.inl:297-304`）决定是否由注册表负责释放，本文件未显式传该参数 → 走默认。**我未核对 `TPropertyValue` 对返回指针的默认 `AddReference` 行为**，故仅记录、不列为发现。

**`FRegisterAssetBundleEntry.cpp`（27 行，3 条目）** — 职责：`FAssetBundleEntry`（BundleName + 路径列表）的构造与有效性。
- 版本门控：`:3-9`
- 构造：`:18`(拷贝)、`:20`(FName InBundleName)
- 其余：`:22 IsValid`
- 结论：无 P0/P1。

**`FRegisterPolyglotTextData.cpp`（50 行，20 条目）** — 职责：`FPolyglotTextData`（本地化文本数据）的分类/命名空间/键/原生串/本地化串读写。
- 构造：`:13`(Category+Namespace+Key+NativeString+NativeCulture 五参)
- 其余：`:16 SetCategory`、`:18 GetCategory`、`:19`**SetCategory(SetNativeCulture)**、`:21 GetNativeCulture`、`:22 ResolveNativeCulture`、`:23 GetLocalizedCultures`、`:24 SetIdentity`、`:26 GetIdentity`、`:28 GetNamespace`、`:29 GetKey`、`:30 SetNativeString`、`:32 GetNativeString`、`:33 AddLocalizedString`、`:35 RemoveLocalizedString`、`:37 GetLocalizedString`、`:39 ClearLocalizedStrings`、`:41`**IsMinimalPatch(void(const bool))**、`:43`**IsMinimalPatch(bool()const)**、`:45 GetText`
- 结论：**2 条 P1（F-INT2-004）** + **1 条 P2（F-INT2-019 的参数名错，已并入 F-INT2-004）**。本地化数据拷贝语义：`:33 AddLocalizedString(culture, string)` / `:37 GetLocalizedString(culture, out string)` / `:35 RemoveLocalizedString` / `:39 ClearLocalizedStrings` 全部直绑引擎成员，`FString` 经 `TPropertyValue` 按值传入（`TArgument.inl:59-60` 走 `TPropertyValue::Get`），**未发现按引用／按指针传递导致的浅拷贝**；`:19` 的错绑才是本文件的真实缺陷。

### 6.6 引擎服务类（Library / 输入）

**`FRegisterDataTableFunctionLibrary.cpp`（52 行，1 条目）** — 职责：`UDataTableFunctionLibrary` 的 `GetDataTableRowFromName` 手写 thunk（唯一为满足 C# 泛型 `out T` 而手写的库函数）。
- 辅助实现：`:11-42 GetDataTableRowFromNameImplementation`（内部：`:15 GetString<FName>`、`:22 GetObject<UDataTable>`、`:25 Bind<false>`、`:27 FReflectionRegistry::Get().GetClass`、`:29 Class->InitObject()`、`:31`**未判空 Find**、`:33 GetStruct`、`:35 CopyScriptStruct`）
- 导出：`:47 .Function("GetDataTableRowFromName", GetDataTableRowFromNameImplementation)`
- 结论：**1 条 P0（F-INT2-001）**。ABI：C++ 返回 `uint8`、C# `DataTableFunctionLibraryImplementation.cs:7` 声明 `byte` → **一致**；C++ 第三参 `IManagedHandle* OutRow`、C# `nint* OutRow` → **一致**（`IManagedHandle` 为 `int64`，`IManagedHandle.h:5-7`）；两个句柄参数 C# 用 `nint` → **一致**。

**`FRegisterEnhancedInputComponent.cpp`（227 行，6 条目）** — 职责：`FInputBindingHandle`/`FEnhancedInputActionEventBinding`/`UEnhancedInputComponent` 的运行时绑定（含动态生成 UFunction 与参数属性）。
- `BINDING_CLASS`：`:11 FInputBindingHandle`、`:13 FEnhancedInputActionEventBinding`
- `FInputBindingHandle`：`:22 GetHandle`
- `FEnhancedInputActionEventBinding`：`:33 Inheritance<FInputBindingHandle>`、`:34 GetAction`、`:35 GetTriggerEvent`
- 辅助实现：`:43-66 GetDynamicBindingObjectImplementation`、`:68-104 BindFunction`（含 `:97 Function->AddToRoot()`）、`:106-164 BindActionFunction`（`:111/:114 SourceActionProperty`、`:124/:127 TriggeredTimeProperty`、`:135/:138 ElapsedTimeProperty`、`:146/:149/:153/:155/:158 ActionValueProperty`）、`:166-202 BindActionImplementation`、`:204-215 RemoveBindingImplementation`
- 导出：`:220 GetDynamicBindingObject`、`:221 BindAction`、`:222 RemoveBinding`
- 结论：**3 条 P1（F-INT2-002/010/011）+ 1 条 P2（F-INT2-015）**。委托绑定/解绑的配对性：`RemoveBinding`（`:213`）与 `BindAction`（`:179`）在 API 层面配对，但解绑路径**未校验绑定对象有效性**（F-INT2-011），且 `:194-198 AddBindingReference` 把 `BindAction` 返回的**引用地址**交给注册表（悬垂风险）。ABI：C# `EnhancedInputComponentImplementation.cs:9/19/30` 的三个 `__UEnhancedInputComponent_*` 声明与本文件 3 个导出的参数个数/类型（`nint` ↔ `IManagedHandle`）**逐一对应**。

**`FRegisterInputComponent.cpp`（345 行，8 条目）** — 职责：`UInputComponent` 的六类旧式输入委托绑定（Action/Axis/AxisKey/Key/Touch/VectorAxis）与绑定值清空。
- 辅助实现：`:17-40 GetDynamicBindingObjectImplementation`、`:42-63 BindImplementation<T>` 模板、`:65-99 BindActionImplementation`、`:101-127 BindAxisImplementation`、`:129-155 BindAxisKeyImplementation`、`:157-190 BindKeyImplementation`、`:192-245 BindTouchImplementation`、`:247-281 BindVectorAxisImplementation`、`:283-319 BindFunction`（含 `:312 Function->AddToRoot()`）、`:321-328 ClearBindingValuesImplementation`
- 导出：`:333 GetDynamicBindingObject`、`:334 BindAction`、`:335 BindAxis`、`:336 BindAxisKey`、`:337 BindKey`、`:338 BindTouch`、`:339 BindVectorAxis`、`:340 ClearBindingValues`
- 结论：**1 条 P1（F-INT2-012）+ 2 条 P2（F-INT2-015、F-INT2-021）**。解绑配对性：本文件**只提供 `ClearBindingValues`（`:321` 调 `UInputComponent::ClearBindingValues`）这一种"全清"出口，没有逐个解绑的导出** —— 这是与 `FRegisterEnhancedInputComponent`（有 `RemoveBinding`）不一致的地方；对需要精确解绑的场景只能整表清空，属功能性缺口。ABI：C# `InputComponentImplementation.cs:8/18/27/36/45/54/63/72` 的 8 个声明与本文件 8 个导出**逐一对应**（4 个 `nint` 参数 ↔ 4 个 `IManagedHandle`）。

**`FRegisterKeys.cpp`（416 行，342 条目）** — 职责：把引擎 `EKeys` 的静态 `FKey` 常量逐个暴露给 C#。
- 结构：`:5 BINDING_CLASS(EKeys)`、`:13 TBindingClassBuilder<EKeys>`、`:14-411` 共 342 条 `.Property("Name", BINDING_PROPERTY(&EKeys::Name))`
- 唯一特例：`:117 .Property("Equals", BINDING_PROPERTY(&EKeys::Equals, EPropertyInteract::New))`
- 版本门控：`:156 Gamepad_Special_Left_Touched`、`:201/:216 Vive_*_System_Click`、`:314 OculusTouch_Right_System_Click`、`:339/:342 ValveIndex_Left_System_*`、`:373/:376 ValveIndex_Right_System_*`、`:405/:408 Virtual_Accept/Back`
- 结论：**逐 key 核对结果：342 条绑定的"名字 vs `&EKeys::目标`"全部一致，0 条错位、0 条重复名、0 条重复目标（脚本审计，见 4.2）。引擎 `EKeys` 定义的"漏项"未能核对（无引擎头文件，见第 5 节第 2 项）—— 但机制上不可能发生错位（按 `&EKeys::X` 取地址，不按数组下标）。真实问题是 F-INT2-013（全部用可写的 `BINDING_PROPERTY`）+ F-INT2-027（`:117` 的例外无注释）。**

**`FRegisterUnreal.cpp`（192 行，7 条目）** — 职责：`Unreal` 静态库类（NewObject/DuplicateObject/LoadObject/LoadClass/CreateWidget/GWorld/GetTransientPackage）+ `ELoadFlags` 枚举。
- 枚举：`:11 BINDING_ENUM(ELoadFlags)`、`:20-37`（18 个 Enumerator）
- 辅助实现：`:45-68 NewObjectImplementation`、`:70-85 DuplicateObjectImplementation`、`:87-109 LoadObjectImplementation`、`:111-133 LoadClassImplementation`、`:135-166 CreateWidgetImplementation`、`:168-171 GWorldImplementation`、`:173-176 GetTransientPackageImplementation`
- 导出：`:181 NewObject`、`:182 DuplicateObject`、`:183 LoadObject`、`:184 LoadClass`、`:185 CreateWidget`、`:186 GWorld`、`:187 GetTransientPackage`
- 结论：**1 条 P1（F-INT2-009）**。GameThread 假设：`NewObject`/`DuplicateObject`/`LoadObject`/`CreateWidget` 都是 UObject 生命周期 API，**必须在 GameThread**，但本文件**没有任何 `check(IsInGameThread())` 或注释声明这一约束**；C# 侧从异步任务调用即崩溃，且崩溃点在引擎内部难以归因（标为设计隐患，未单列发现）。`World` 为空路径：`:170 GWorldImplementation` 直接 `Bind(GWorld)`，`GWorld` 在 Commandlet/无世界场景为 nullptr → 返回无效句柄，C# `UnrealImplementation.cs:64` 已处理（返回 null），**这一处是安全的**。`:60-65 NewObject` 的 `bCopyTransientsFromClassDefaults` 用 `uint8` 而非 `bool` 跨边界，C# `UnrealImplementation.cs:15` 做 `(byte)(... ? 1 : 0)` 转换 → **宽度处理正确**（本批次 ABI 抽查的正面样本）。

**`FRegisterWorld.cpp`（155 行，16 条目）** — 职责：`FActorSpawnParameters::ESpawnActorNameMode` 枚举、`FActorSpawnParameters` 的 9 个属性与 3 组 bool 读写、`UWorld::SpawnActor`。
- 枚举：`:10 BINDING_ENUM(...ESpawnActorNameMode, false)`、`:20-25`（4 个 Enumerator）
- `BINDING_CLASS`：`:12 FActorSpawnParameters`
- bool 读写：`:33-42 GetbNoFail`、`:44-51 SetbNoFail`、`:53-62 GetbDeferConstruction`、`:64-71 SetbDeferConstruction`、`:73-82 GetbAllowDuringConstructionScript`、`:84-92 SetbAllowDuringConstructionScript`
- 属性：`:97-106`（Name/Template/Owner/Instigator/OverrideLevel/OverrideParentComponent/SpawnCollisionHandlingOverride/NameMode/ObjectFlags）
- 导出：`:107-112`（上述 6 个 Get/Set bool）、`:150 .Function("SpawnActor", SpawnActorImplementation)`
- 辅助实现：`:120-145 SpawnActorImplementation`（`:125 GetObject<UWorld>`、`:127 GetObject<UClass>`、`:129 GetAddress<UScriptStruct, FTransform>`、`:132 GetBinding<FActorSpawnParameters>`、`:135 SpawnActor`、`:141 Bind`）
- 结论：**1 条 P1（F-INT2-008）**。ABI：3 组 bool 用 `uint8` 出参/入参、C# `FActorSpawnParametersImplementation.cs:7/14/21/28/35/42` 声明 `byte` 并转换为 `bool` → **一致**；`SpawnActor` 4 个 `nint` ↔ 4 个 `IManagedHandle` → **一致**。GameThread 假设：`UWorld::SpawnActor` 必须 GameThread，本文件同样**无 `check(IsInGameThread())`**。

