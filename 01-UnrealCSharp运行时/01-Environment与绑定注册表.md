# UnrealCSharp 绑定运行时核心：FCSharpEnvironment / FCSharpBind / FBindingRegistry

> 本报告各条**严重度见各 Finding 的「复核结论」字段**。
> **逐条复核结论**：21 条 —— **确认 15 条、部分确认 6 条（F-ENV-001/004/005/007/011/021）、证伪 0 条、无法验证 0 条**；级别**全部维持**（上调 0、下调 0、撤销 0）；可达性 活跃 20 / 潜伏 1（F-ENV-009）/ 不可达 0。
> 本报告**修正的是事实而非级别**，并以本机 UE 5.6 引擎源码（`Engine\Source` 与 `Engine\Plugins\EnhancedInput`）核实三处标为"未验证假设"的引擎行为：F-ENV-001（`BindAction` 返回堆对象引用，非数组元素引用）、F-ENV-007（`OnAsyncLoadingFlushUpdate` 实际每帧广播）、F-ENV-004（`TWeakObjectPtr` 哈希 = `ObjectIndex ^ ObjectSerialNumber`）。前两处的原始机制被**证伪并改写**，结论级别不变。





> 分析范围：`Plugins/UnrealCSharp/Source/UnrealCSharp`（Runtime 模块）中的 Environment 与 Registry 两组文件
> 覆盖文件：7 个（另读取 8 个直接相关的支撑文件用于取证，见 §0.2）
>
> 覆盖深度：7 个目标文件逐行读完（未覆盖部分见 §7）

---

## 0. 覆盖范围与阅读清单

### 0.1 目标文件（全部读完）

| 文件 | 行数 | 是否读完 | 备注 |
|---|---|---|---|
| `Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h` | 361 | 读完 | 类声明 + 12 个裸指针成员（11 个注册表 :334-354 + `FCSharpBind*`，另有 `FDomain* Domain` :309；`FOptionalRegistry*` :357 受 `UE_F_OPTIONAL_PROPERTY` 控制） |
| `Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.inl` | 383 | 读完 | 全部转发模板（注册表访问器） |
| `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp` | 636 | 读完 | 单例、生命周期、UObject 监听、信号处理 |
| `Source/UnrealCSharp/Public/Registry/FCSharpBind.h` | 91 | 读完 | 绑定门面（全 static） |
| `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp` | 545 | 读完 | 反射->C# 绑定核心逻辑 |
| `Source/UnrealCSharp/Public/Registry/FBindingRegistry.h` | 95 | 读完 | 地址键绑定注册表 |
| `Source/UnrealCSharp/Private/Registry/FBindingRegistry.cpp` | 75 | 读完 | 释放路径 |

### 0.2 为取证额外读取的支撑文件（不属于我的 7 个文件，仅用于确认调用上下文/行号）

| 文件 | 读到的行 | 用途 |
|---|---|---|
| `Source/UnrealCSharp/Private/UnrealCSharp.cpp` | 全文 57 | 模块 Active/InActive 广播点 |
| `Source/UnrealCSharp/Private/Listener/FUObjectListener.cpp` | 全文 54 | 谁调用 NotifyUObjectCreated/Deleted |
| `Source/UnrealCSharp/Private/Registry/FObjectRegistry.cpp` | 全文 122 | 弱指针安全性与句柄释放路径 |
| `Source/UnrealCSharp/Private/Registry/FReferenceRegistry.cpp` | 全文 81 | FReference 生命周期、FGCObject |
| `Source/UnrealCSharp/Private/Registry/FStructRegistry.cpp` | 全文 147 | 结构体内存 Malloc/Free 配对 |
| `Source/UnrealCSharp/Private/Registry/FClassRegistry.cpp` + `.inl` | 全文 271 / 93 | 描述符容器、重入点 |
| `Source/UnrealCSharp/Private/Registry/FDynamicRegistry.cpp` | 全文 73 | 与 FCSharpBind 的委托注册顺序 |
| `Source/UnrealCSharp/Private/Domain/FDomain.cpp` | 1-140 | GCHandle_Free 依赖的脚本域 |
| `Source/UnrealCSharp/Public/Reference/{FReference,FBindingReference}.h`、`TStructRegistry.inl` 等 | 全文 | 句柄释放链 |
| `Source/UnrealCSharp/Public/Binding/Core/TPropertyValue.inl` | 140-249 | AddBindingReference 的真实调用点 |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp` | 80-227 | 地址键注册的真实调用点 |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterWorld.cpp`、`FRegisterProperty.cpp`、`FRegisterStruct.cpp` | 部分 | GetBinding / GetAddress / Bind 调用点 |
| `Source/UnrealCSharpCore/Private/UnrealCSharpCore.cpp`、`Listener/FEngineListener.cpp`、`Compiler/Private/FCSharpCompilerRunnable.cpp` | 部分 | 域重载（Deactivate→Activate）时序 |
| `Source/UnrealCSharpCore/Public/Domain/Script/IManagedHandle.h` | 全文 40 | 句柄语义（int64，0=Invalid） |
| `Script/SourceGenerator/UnrealTypeSourceGenerator.cs`、`Script/UE/CoreUObject/{InputComponent,EnhancedInputComponent}.cs` | 片段 | C# 侧对应行为 |

### 0.3 检索方式说明（事实纪律）

- 所有行号来自 `read` 工具输出，未估算。
- 全局计数使用 PowerShell `Select-String`；范围 = `Source/` 下 `*.h,*.cpp,*.inl`，排除 `\ThirdParty\`。**实测重数：507 个文件**（`Get-ChildItem -Recurse -Include *.h,*.cpp,*.inl` 过滤 `\ThirdParty\`；含 ThirdParty 为 1027，ThirdParty 自身 520）。原文写的"543 个文件"为误记，已修正。
- 全文引用检索一律使用 `grep` 工具或上述脚本，模式在正文中给出。
- 未验证的假设一律在正文标 `置信度: 中/低` 并写明假设内容。

---

## 1. 模块职责与架构速览

### 1.1 对象关系

```
FCSharpEnvironment (:309 Domain, :334-354 11 个注册表裸指针)   ← 进程级单例
  ├─ static FCSharpEnvironment Environment;                  (FCSharpEnvironment.cpp:33)
  ├─ FDomain*               Domain              → 创建/销毁脚本域（Mono/CoreCLR/LeanCLR 抽象）
  ├─ FCSharpBind*           CSharpBind          → 反射 <-> C# 绑定门面（全 static 方法，实例只为了注册委托）
  ├─ FDynamicRegistry*      → 动态类生成回调
  ├─ FClassRegistry*        → UStruct → FClassDescriptor / FunctionHash / PropertyHash
  ├─ FReferenceRegistry*    → FGCObject，持有 TArray<TObjectPtr<UObject>>（唯一受 GC 保护的强引用容器）
  │                           证据：`FReferenceRegistry.h:5` 继承 `FGCObject`、`:13` `AddReferencedObjects` override、`:29` `TArray<TObjectPtr<UObject>> ObjectArray;`；`FReferenceRegistry.cpp:22-25` `Collector.AddReferencedObjects(ObjectArray);`、`:27-30` `GetReferencerName()`、`:59-69` `AddReference(UObject*)` → `ObjectArray.AddUnique(InObject)`
  │                           ★ 跨报告一致性核查追加（F-CS2B-008）：本条与 `06-CSharp运行时与脚本/02b-对象字符串与工具类型包装.md` 对 F-CS2B-008 的证伪结论**方向一致** —— 即"原生侧未保活 UObject / 句柄可能指向已被 UE GC 回收的 UObject"这一前提**不成立**；本报告**未复用**该假设（详见 §6.2 同一行的说明）
  ├─ FObjectRegistry*       → TMap<IManagedHandle, TWeakObjectPtr<const UObject>>（弱引用，安全）
  ├─ FStructRegistry* / FContainerRegistry* / FDelegateRegistry* / FMultiRegistry* / FStringRegistry*
  ├─ FBindingRegistry*      → TMap<void*, IManagedHandle> + TMap<IManagedHandle, {void* wrapper,bool}>
  └─ FOptionalRegistry*     (#if UE_F_OPTIONAL_PROPERTY)
```

`FCSharpEnvironment` 只在 `UnrealCSharp` 模块内部被引用（证据见 §6.6），是"模块内单例 + 对外导出符号"的混合形态。

### 1.2 生命周期（域重载 / 热重载）

```
引擎/编辑器事件
  FEngineListener::OnPreBeginPIE / OnPreExit            (FEngineListener.cpp:34-61)
  FCSharpCompilerRunnable 编译成功后的 GameThread 任务   (FCSharpCompilerRunnable.cpp:213-243)
      └─ FUnrealCSharpCoreModule::Deactivate()          (UnrealCSharpCore.cpp:29-37, 带 State 保护)
             └─ OnUnrealCSharpCoreModuleInActive.Broadcast()          (UnrealCSharpCore.cpp:35)
                    └─ FUnrealCSharpModule::OnUnrealCSharpCoreModuleInActive (UnrealCSharp.cpp:50-53)
                           └─ FUnrealCSharpModuleDelegates::OnUnrealCSharpModuleInActive.Broadcast()
                                  ├─ FCSharpEnvironment::OnUnrealCSharpModuleInActive → Deinitialize()   (FCSharpEnvironment.cpp:332)
                                  └─ FUObjectListener::OnUnrealCSharpModuleInActive → 摘除 GUObjectArray 监听 (FUObjectListener.cpp:34)
      └─ FUnrealCSharpCoreModule::Activate()            (UnrealCSharpCore.cpp:19-27, State==Inactive 才广播)
             └─ ... → FCSharpEnvironment::OnUnrealCSharpModuleActive → Initialize()   (FCSharpEnvironment.cpp:327)
```

关键结论：**Deactivate/Activate 是成对的**（`UnrealCSharpCore.cpp:21`、`:31` 的 State 守卫），因此 `Initialize()` 的正常路径只会在 `Deinitialize()` 之后发生；但 `FCSharpEnvironment` 自身**没有任何重入/幂等保护**（见 F-ENV-009）。

### 1.3 数据流：一次 C#↔C++ 互调

```
C# 对象方法 → P/Invoke → FRegisterXxx 中的静态 Implementation  (Domain/Interop/*.cpp)
   → FCSharpEnvironment::TGetObject<T,T>()(env, handle)          (FCSharpEnvironment.h:238-289)
        ├ UObject  : GetObject<T>(handle)   → FObjectRegistry TMap 查找 + Cast<T>   (inl:112-115)
        ├ UStruct  : GetStruct<T>(handle)   → FStructRegistry TMap 查找             (inl:127-130)
        └ 其它     : GetBinding<T>(handle)  → FBindingRegistry TMap 查找            (inl:290-295)
   → 业务调用 → 返回值经 TBindingPropertyValue::Get() 反向注册回注册表
```

---

## 2. 关键调用链

1. `FUObjectListener::NotifyUObjectCreated` (FUObjectListener.cpp:43) → `FCSharpEnvironment::NotifyUObjectCreated` (FCSharpEnvironment.cpp:276) → `Bind<true>(InObject)` (cpp:291) → `FCSharpBind::BindImplementation<true>(UObject*)` (FCSharpBind.inl:56) → `FClassRegistry::AddClassDescriptor` (FClassRegistry.cpp:97) —— **每个 UObject 创建都会进入**。
2. `FDynamicRegistry::Initialize` 的 `OnPostClassConstructor` lambda (FDynamicRegistry.cpp:19-37) → `FCSharpEnvironment::Bind<true>` (FDynamicRegistry.cpp:25) → `FCSharpEnvironment::GetObject` (FDynamicRegistry.cpp:27) → `FClassReflection::ConstructorObject`。
3. `FCSharpEnvironment::Deinitialize` (FCSharpEnvironment.cpp:147) → `delete BindingRegistry` (cpp:188) → `FBindingRegistry::Deinitialize` (FBindingRegistry.cpp:19) → `FDomain::GCHandle_Free` (FDomain.cpp:94) → `IScriptDomain::Free`。
4. `FCSharpEnvironment::Deinitialize` (cpp:242) → `delete ClassRegistry` → `FClassRegistry::Deinitialize` (FClassRegistry.cpp:40) → `delete FClassDescriptor` → `FClassDescriptor::Deinitialize` (FClassDescriptor.cpp:56-68) → **回调用正在析构的 `FCSharpEnvironment::ClassRegistry`** → `FClassRegistry::RemoveFunctionDescriptor`。
5. `FCSharpBind::OnCSharpEnvironmentInitialize` (FCSharpBind.cpp:536) → `TObjectRange<UClass>` → `BindClassDefaultObject` (FCSharpBind.cpp:78) → `CanBind` (FCSharpBind.cpp:419) → `FReflectionRegistry::Get().GetClass` / `UnrealCSharpSetting->GetBindClass()` 线性扫描 (FCSharpBind.cpp:93-99)。
6. `FRegisterEnhancedInputComponent.cpp:194` → `FCSharpEnvironment::AddBindingReference<T,false>` (FCSharpEnvironment.inl:298) → `FBindingRegistry::AddReference` (FBindingRegistry.inl:15) → `TBindingAddressWrapper::new` (inl:19)；回收侧 `FBindingRegistry::RemoveReference` (FBindingRegistry.cpp:47)。
7. `TPropertyValue.inl:182/229` → `AddBindingReference<std::decay_t<T>, false>` → 同一 wrapper 分配，`bNeedFree=false`（泄漏路径，见 F-ENV-002）。
8. `FCSharpEnvironment::Initialize` (cpp:57) → `new FDomain()` (cpp:59) → `FDomain::Initialize` (FDomain.cpp:20) → `FScriptDomainFactory::Create` → 加载 C# 程序集；随后 `new FCSharpBind()` (cpp:63) 清空 `NotOverrideTypes` (FCSharpBind.cpp:28)，最后 `OnCSharpEnvironmentInitialize.Broadcast()` (cpp:144)。

---

## 3. 发现清单

> 严重度排序：P0（无）→ P1 → P2 → P3；同级按文件路径字典序。
>
> **关于分组标题与最终级别的差异**：下面 `### P1 / ### P2 / ### P3` 三个分组标题沿用**初判**的排序位置，编号与位置不再变动（编号是跨报告引用锚点）。**最终严重度以每条 Finding 的 `- **严重度**:` 字段为准**：位于 `### P2` 分组内但最终为 P3 的有 5 条 —— **F-ENV-003 / F-ENV-004 / F-ENV-005 / F-ENV-007 / F-ENV-012**；`### P1` 分组内 F-ENV-001 最终为 P2、F-ENV-002 为 P1（该组共 2 条）；`### P3` 分组（**F-ENV-014 ~ F-ENV-021，共 8 条**）全部为 P3。按最终级别统计全局：**P0=0、P1=1（F-ENV-002）、P2=7（F-ENV-001/006/008/009/010/011/013）、P3=13（P2 组内 5 条 + P3 组内 8 条）**，合计 21。

### P0

**未发现 P0 级（确定性崩溃/数据损坏）问题。** 最严重的两条 F-ENV-001（最终 P2）与 F-ENV-002（最终 P1，也是本报告唯一的 P1）见下方 `### P1` 分组的前两条。

### P1

---

### [F-ENV-001] FBindingRegistry 以裸地址为键且无任何生命周期/有效性校验，调用方把"堆上绑定对象地址 / 栈上实参地址"存了进去（悬垂地址解引用）

- **类别**: 未定义行为 | 内存/资源泄漏
- **严重度**: **P2**（触发路径已由引擎源码收窄为"引擎侧移除/清空绑定后注册表条目残留"与"栈上实参地址入表"，不再按"若触发则等同 P0"记账；机制本身是 UAF 读写，一旦命中仍属数据损坏级）
- **复核结论**: 部分确认（偏差：**触发路径修正** —— "`BindAction` 返回 `TArray` 元素引用、扩容即令此前地址失效"一条经引擎源码核对**被证伪**；"注册表以裸地址为键且无任何有效性/代次校验"一条**确认成立**。真实的悬垂来源是"引擎侧移除/清空绑定后注册表仍保留已释放地址"与"栈上实参地址被登记进全局表"）
- **可达性**: 活跃（C# 每次 `BindAction`、以及每次按 `ref/out` 编组的属性访问都会执行 `AddBindingReference`）
- **复核证据**: 引擎侧 —— `Engine\Plugins\EnhancedInput\Source\EnhancedInput\Public\EnhancedInputComponent.h:362` `TArray<TUniquePtr<FEnhancedInputActionEventBinding>> EnhancedActionEventBindings;`，`:453`/`:468` `return *EnhancedActionEventBindings.Add_GetRef(MoveTemp(AB));` → 返回的是 **`TUniquePtr` 所拥有的堆对象**的引用，数组扩容只搬移 `TUniquePtr` 本身、**pointee 地址不变**；`...\Private\EnhancedInputComponent.cpp:82-98`（`RemoveBindingByHandle` / `RemoveBinding`）会销毁该 `TUniquePtr` → 对象释放，而 `FBindingRegistry` 的条目不会随之移除 → 二次 `RemoveBinding` 即读已释放内存、或新绑定复用同一地址时 `GetObject(address)` 返回陈旧句柄。插件侧 —— `Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp:194-196`（登记处）、`:210-213`（解引用处）；`Source/UnrealCSharp/Public/Binding/Function/TArgument.inl:71-72` `Set()` 传入 `const_cast<std::decay_t<Type>*>(&Value)`、`TFunctionHelper.inl:21` 的 `std::tuple<TArgument<Args, Args>...> Argument(...)` 是**函数局部变量** → 栈地址确实会进注册表
- **级别变动**: 无（P2 维持；原标题"若触发则等同 P0（UAF 读写）"的副词因触发路径收窄而删除，仍按 P2 隐患计）
- **文件**: `Source/UnrealCSharp/Public/Registry/FBindingRegistry.h:38-51,77-92`、`Source/UnrealCSharp/Public/Registry/FBindingRegistry.inl:15-25,28-38`、`Source/UnrealCSharp/Private/Registry/FBindingRegistry.cpp:40-45,47-75`
- **函数**: `FBindingRegistry::AddReference<T,IsNeedFree>(const T*, FClassReflection*, IManagedHandle)` / `FBindingRegistry::GetBinding<T>(IManagedHandle)` / `FBindingRegistry::RemoveReference(IManagedHandle)`
- **置信度**: 高（已用本机 UE 5.6 引擎源码核实 `BindAction` 的返回语义，见 `复核证据`；"注册表无有效性校验"与"栈地址入表"两条均为直接可读的代码事实）

**现状（代码事实）**

```cpp
// FBindingRegistry.h:38-51
	struct FBindingAddress
	{
		typedef FBindingAddressWrapper FWrapperType;

		explicit FBindingAddress(FWrapperType* InAddressWrapper, const bool InNeedFree = true) :
			AddressWrapper(InAddressWrapper),
			bNeedFree(InNeedFree)
		{
		}

		FWrapperType* AddressWrapper;   // ← 只有裸指针，没有所属 UObject、没有句柄代次

		bool bNeedFree;
	};
```

```cpp
// FBindingRegistry.inl:15-25
template <typename T, auto IsNeedFree>
auto FBindingRegistry::AddReference(const T* InObject, FClassReflection* InClass, const IManagedHandle InManagedHandle)
{
	BindingAddress2ManagedHandle.Add(static_cast<void*>(const_cast<T*>(InObject)), InManagedHandle); // :17 地址 → 句柄

	auto BindingAddressWrapper = new TBindingAddressWrapper(InObject);                               // :19 地址存进 wrapper

	ManagedHandle2BindingAddress.Add(InManagedHandle,
	                                 FBindingValueMapping::ValueType(BindingAddressWrapper, IsNeedFree));

	return true;
}
```

```cpp
// FBindingRegistry.cpp:40-45  查询：命中即返回，不校验地址是否仍属于活对象
IManagedHandle FBindingRegistry::GetObject(const FBindingValueMapping::FAddressType InAddress)
{
	const auto FoundManagedHandle = BindingAddress2ManagedHandle.Find(InAddress);

	return FoundManagedHandle != nullptr ? *FoundManagedHandle : InvalidManagedHandle;
}
```

```cpp
// FBindingRegistry.inl:6-12  取值：直接把存下来的地址 static_cast 后返回给调用方
template <typename T>
auto FBindingRegistry::GetBinding(const IManagedHandle InManagedHandle)
{
	const auto FoundValue = ManagedHandle2BindingAddress.Find(InManagedHandle);

	return FoundValue != nullptr ? static_cast<T*>(FoundValue->AddressWrapper->Value) : nullptr;
}
```

**调用上下文**

真实注册点（写在别的文件里，但注册的正是"某个 UObject 内部 TArray 元素的地址"）：

```cpp
// FRegisterEnhancedInputComponent.cpp:179-196
				const auto& EnhancedInputActionEventBinding = FoundObject->BindAction(
					InputAction, TriggerEvent, ObjectToBindTo, FunctionNameToBind);
				...
				FCSharpEnvironment::GetEnvironment().AddBindingReference<
					std::decay_t<FEnhancedInputActionEventBinding>, false>(   // ← IsNeedFree=false
					FoundClass, Object, &EnhancedInputActionEventBinding);    // ← 取元素地址
```

消费点（同一个文件）：

```cpp
// FRegisterEnhancedInputComponent.cpp:210-213
				const auto EnhancedInputActionEventBinding = FCSharpEnvironment::GetEnvironment().GetBinding<
					FEnhancedInputActionEventBinding>(InEnhancedInputActionEventBinding);

				FoundObject->RemoveBinding(*EnhancedInputActionEventBinding);   // ← 解引用保存的地址
```

另一条注册链（值对象取址）：`TPropertyValue.inl:166-167`、`182-183`、`213-214`、`229-230`，其中 `:166`/`:213` 走 owner 重载（`FBindingRegistry.inl:28-38`，同样保存裸地址）。

**问题**

1. `FBindingRegistry` 的两张表都是"地址 ⇒ 句柄"，**只做指针相等比较**：既不知道这个地址属于哪个 UObject，也没有任何失效标记。`GetBinding` 返回的指针无法区分"地址仍有效"与"地址已被回收/复用"。
2. `FRegisterEnhancedInputComponent.cpp:196` 存的是 `BindAction()` 返回的引用地址。**修正**：该引用指向 `UEnhancedInputComponent` 内部 `TArray<TUniquePtr<FEnhancedInputActionEventBinding>>`（`EnhancedInputComponent.h:362`）中 `TUniquePtr` 所**拥有的堆对象**（`:453`/`:468` `*Add_GetRef(...)`），因此**数组扩容不会使此前登记的地址失效**（原稿"扩容即 UAF"的说法已被引擎源码证伪）。真正的悬垂窗口有两个：(a) 引擎侧一旦移除该绑定（`RemoveBinding`/`RemoveBindingByHandle`/`ClearActionBindings`，`EnhancedInputComponent.cpp:36-98`）或 `UEnhancedInputComponent` 被销毁，`TUniquePtr` 随即析构、绑定对象被释放，而 `FBindingRegistry` 里的条目**没有任何回收路径**（`AddBindingReference<..., false>` 既不挂 owner 引用，C# `UEnhancedInputComponent.RemoveAction` 也只调用 `RemoveBindingImplementation` 与 `RemoveFunction`，见 `Script/UE/CoreUObject/EnhancedInputComponent.cs:54-82`）；此后任何一次 `GetBinding<FEnhancedInputActionEventBinding>(handle)`（`FRegisterEnhancedInputComponent.cpp:210`）都返回已释放地址，`:213` 的 `RemoveBinding(*ptr)` 会从已释放内存中读 `FInputBindingHandle::Handle`（`Runtime/Engine/Classes/Components/InputComponent.h:342,366`）。(b) `TArray` 一旦插入/删除中间元素，地址本身不变但语义可能已错位（同一绑定对象被移动位置不影响地址，故这条只是次要）。
3. 同一模式还出现在"按值参数的 `ref/out` 编组"路径（`TPropertyValue.inl:174-192` 的 `IsReference=true` 分支）：它把**栈上实参对象**的地址注册进全局表（`TArgument.inl:71-73` 传入的是 `TBaseArgument::Value` 的地址），而返回给 C# 的句柄是新建的托管对象（`TPropertyValue.inl:178`）。互调函数返回后栈帧失效，注册表条目仍指向已失效栈地址。
4. 表本身没有任何 `IsValid()`/代次（generation）字段可供校验，**修复时无法只改一个函数**——需要改数据结构。

**建议**

```cpp
// 方案 A（最小改动，推荐）：把 wrapper 的所有权与生命周期绑到 owning UObject
struct FBindingAddress
{
    FWrapperType* AddressWrapper;
    bool bNeedFree;
    TWeakObjectPtr<const UObject> Owner;   // 新增：地址所属对象（若有）
};
// GetBinding 中：
if (FoundValue->Owner.IsValid() == false && FoundValue->Owner.IsExplicitlyNull() == false)
{
    return nullptr;                    // 宿主已回收，拒绝返回悬垂地址
}
```
```cpp
// 方案 B（根治）：对 EnhancedInput 这类"容器元素引用"，不要存元素地址。
// 注册时按 (Component, InputAction, TriggerEvent) 三元组为键做 TMap 查找，
// 取值时现场重新计算地址；即把 FBindingRegistry 从"地址表"改为"稳定键表"。
```
权衡：方案 A 改动小但只能挡住"宿主已 GC"的情形，挡不住"宿主仍活着但容器已扩容"；方案 B 需要在 `FRegisterEnhancedInputComponent.cpp` 增加稳定键，改动面较大，但能同时消除 3 中的栈地址问题（栈地址根本不该进注册表）。

**验证方式**

- `grep -n "AddBindingReference" Source/`（**实测 12 处**：声明 `FCSharpEnvironment.h:216,219`；定义 `FCSharpEnvironment.inl:298,307`；调用 `FRegisterEnhancedInputComponent.cpp:194`、`TPropertyValue.inl:166,182,187,213,229,234`、`TConstructorHelper.inl:42`）逐一确认注册值的来源是否为"容器元素引用/栈对象地址"。
- 复现用例：在 C# 中连续 `BindAction` 两次以上（第二次绑定新 InputAction），随后对**第一个**绑定对象调用 `RemoveBinding`，在 `FBindingRegistry::GetBinding` 返回后加 `check(IsValidAddress)` 或用 ASan/`-fsanitize=address` 构建观察。
- 长期守卫：在 `FBindingRegistry::GetBinding<T>` 返回前加 `ensureMsgf` 校验地址落在已知 UObject 的 `[Base, Base+GetStructureSize)` 范围内。

---

### [F-ENV-002] `bNeedFree == false` 时 `TBindingAddressWrapper` 永不释放：每次都漏一个堆对象，只在 Deinitialize 清理表（表清了，内存仍漏）

- **类别**: 内存/资源泄漏
- **严重度**: **P1**
- **复核结论**: 确认
- **可达性**: 活跃（`TPropertyValue.inl:182`/`:229` 的 `IsReference=true` 分支与 `FRegisterEnhancedInputComponent.cpp:194-196` 每执行一次即漏一个 wrapper）
- **复核证据**: `Source/UnrealCSharp/Public/Registry/FBindingRegistry.inl:19`、`:33` 无条件 `new TBindingAddressWrapper(InObject)`；`Source/UnrealCSharp/Private/Registry/FBindingRegistry.cpp:27-32`（`Deinitialize`）与 `:62-67`（`RemoveReference`）的 `delete Value.AddressWrapper` 都包在 `if (Value.bNeedFree)` 内；`:69` 的 `ManagedHandle2BindingAddress.Remove(InManagedHandle)` 立刻丢弃 wrapper 指针 → 永久不可达。`FBindingRegistry.inl:35` 的 owner 重载**硬编码** `false`，调用点 `TPropertyValue.inl:166`、`:213`（`grep` 实测）。`grep "bNeedFree"` 在 `Source/UnrealCSharp` 内 FBindingRegistry 相关命中点确认仅 `FBindingRegistry.h:44,50`、`FBindingRegistry.cpp:27,62`、`FBindingRegistry.inl:22,35`
- **级别变动**: 无（P1 维持：无条件、确定性泄漏，非条件触发）
- **文件**: `Source/UnrealCSharp/Public/Registry/FBindingRegistry.inl:19,33`、`Source/UnrealCSharp/Private/Registry/FBindingRegistry.cpp:27-32,62-67`
- **函数**: `FBindingRegistry::AddReference<T,IsNeedFree>` / `FBindingRegistry::RemoveReference` / `FBindingRegistry::Deinitialize`
- **置信度**: 高

**现状（代码事实）**

```cpp
// FBindingRegistry.inl:19（以及 owner 重载 :33）—— wrapper 无条件 new
	auto BindingAddressWrapper = new TBindingAddressWrapper(InObject);
```

```cpp
// FBindingRegistry.cpp:47-75  RemoveReference
bool FBindingRegistry::RemoveReference(const IManagedHandle InManagedHandle)
{
	if (const auto FoundValue = ManagedHandle2BindingAddress.Find(InManagedHandle))
	{
		...
		FDomain::GCHandle_Free(InManagedHandle);        // :60 句柄释放了

		if (FoundValue->bNeedFree)                      // :62 ← 只有 true 才 delete
		{
			delete FoundValue->AddressWrapper;          // :64 而 wrapper 的析构函数才负责 delete 被指向的对象
			FoundValue->AddressWrapper = nullptr;
		}

		ManagedHandle2BindingAddress.Remove(InManagedHandle);   // :69 表项删了，wrapper 指针丢了
		return true;
	}
	return false;
}
```

```cpp
// FBindingRegistry.cpp:19-38  Deinitialize（同样的守卫）
	for (auto& [Key, Value] : ManagedHandle2BindingAddress.Get())
	{
		FDomain::GCHandle_Free(Key);
		Key = IManagedHandle{};
		if (Value.bNeedFree)          // :27
		{
			delete Value.AddressWrapper;   // :29
			Value.AddressWrapper = nullptr;
		}
	}
```

配套的析构语义（说明 `bNeedFree` 一个标志承担了两件事）：

```cpp
// FBindingRegistry.h:24-36
	template <typename T>
	struct TBindingAddressWrapper final : FBindingAddressWrapper
	{
		explicit TBindingAddressWrapper(T* InValue) : FBindingAddressWrapper((decltype(Value))InValue) {}

		virtual ~TBindingAddressWrapper() override
		{
			delete static_cast<T*>(Value);      // :34 释放"被包装的对象"
		}
	};
```

**调用上下文**

`IsNeedFree`（`bNeedFree`）在源码中取 `false` 的调用点至少 4 处：

- `Source/UnrealCSharp/Public/Binding/Core/TPropertyValue.inl:182`（`AddBindingReference<std::decay_t<T>, false>`，`IsReference` 分支）
- `Source/UnrealCSharp/Public/Binding/Core/TPropertyValue.inl:229`（同上，指针版）
- `FBindingRegistry.inl:35`（owner 重载**硬编码** `false`），其调用点 `TPropertyValue.inl:166`、`213`
- `Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp:194-196`

**问题**

`AddressWrapper` 是一个用 `new` 分配的 16 字节对象（vptr + `void*`），它的唯一作用是持有地址；无论被包装的对象是否需要释放，**这个 wrapper 本身都必须 delete**。当前代码把 `delete wrapper` 与 `delete pointee` 绑在同一个 `bNeedFree` 上，导致：

- 分配点：`FBindingRegistry.inl:19` / `:33`（无条件）。
- 缺失的释放点：`FBindingRegistry.cpp:62-67`（`bNeedFree==false` 时不执行）与 `:27-32`（同上）。
- 结果：`RemoveReference` 删掉了表项、释放了 GCHandle，但 wrapper 指针随表项一起丢失，**永久不可达**；`Deinitialize` 虽然清空了两张表，但 `bNeedFree==false` 的 wrapper 同样没有被 delete。

泄漏是"每次按引用编组的属性访问 + 每个 EnhancedInput 绑定"各漏一个对象，且随域重载持续累积（每次重载重建注册表，旧 wrapper 不会回收）。注意这不是"仅在程序退出时漏一次"。

**建议**

```cpp
// FBindingRegistry.h: 让 wrapper 的析构不再删除被包装对象，把"是否释放 pointee"显式带入
template <typename T>
struct TBindingAddressWrapper final : FBindingAddressWrapper
{
    TBindingAddressWrapper(T* InValue, const bool bInNeedFree)
        : FBindingAddressWrapper((decltype(Value))InValue), bNeedFree(bInNeedFree) {}

    virtual ~TBindingAddressWrapper() override
    {
        if (bNeedFree)
        {
            delete static_cast<T*>(Value);
        }
    }
    bool bNeedFree = false;
};
```
```cpp
// FBindingRegistry.cpp:62-67 / :27-32 改为无条件回收 wrapper
    delete FoundValue->AddressWrapper;      // 总是释放包装器；是否释放 pointee 由 wrapper 自己决定
    FoundValue->AddressWrapper = nullptr;
```
权衡：把 `bNeedFree` 下沉到 wrapper 会多 1 字节；若不想改结构，最小改法是删掉 `if (bNeedFree)` 守卫、并在 `TBindingAddressWrapper` 中增加 `bNeedFree` 条件——两者都需要改 `inl:19/33` 的构造参数。

**验证方式**

- 加计数器：在 `TBindingAddressWrapper` 构造函数里 `++GWrapperCount`，析构里 `--GWrapperCount`，跑一段"大量 `ref/out` 复合参数调用 + EnhancedInput 绑定/解绑"，观察计数单调增长。
- `grep -n "bNeedFree" Source/UnrealCSharp` 确认所有读写点（当前：`FBindingRegistry.h:44,50`、`FBindingRegistry.cpp:27,62`、`FBindingRegistry.inl:22,35`）。

---

### P2

---

### [F-ENV-003] 重复注册会静默覆盖 `ManagedHandle2BindingAddress` 旧值，旧 wrapper 指针被丢弃且不可回收（`TMap::Add` 覆盖语义无任何守卫）

- **类别**: 内存/资源泄漏
- **严重度**: **P3**
- **复核结论**: 确认（`TMap::Add` 的覆盖语义、以及"无任何守卫、无旧值回收"两半均为代码事实。原文把"存在同一 handle 被注册两次的触发点"明确标为未验证假设，故不构成偏差）
- **可达性**: 活跃（机制随每次 `AddReference` 生效）；**触发条件仍未证实**
- **复核证据**: `Source/UnrealCSharp/Public/Registry/TMapping.inl:27-30` `Add()` 直接 `Map.Add(InKey, InValue)`（UE `TMap::Add` 对已存在键即覆盖）；`Source/UnrealCSharp/Public/Registry/FBindingRegistry.inl:21-22`、`:35` 均无 `Contains` 判定、无旧值回收；`grep` 实测 `ManagedHandle2BindingAddress.Add` 全 `Source/` 仅 `FBindingRegistry.inl:21`、`:35` 两处。**反证线索**：`TPropertyValue.inl:158-168`/`:205-215` 重注册前先做 `GetBinding(InMember)` 判定，且键 `SrcManagedHandle` 来自 `FoundClass->NewObject()`（每次新句柄），`FRegisterEnhancedInputComponent.cpp:192` 同样新建句柄 → 现有调用链不产生同句柄二次注册
- **级别变动**: 无（P3 维持：无已知触发点 → 按隐患而非泄漏计）
- **文件**: `Source/UnrealCSharp/Public/Registry/FBindingRegistry.inl:21-22,35`、`Source/UnrealCSharp/Public/Registry/TMapping.inl:27-30`
- **函数**: `FBindingRegistry::AddReference<T,IsNeedFree>`
- **置信度**: 中（覆盖语义**高**；"同一 handle 被注册两次"的现存触发点未找到，属未验证假设）

**现状（代码事实）**

```cpp
// TMapping.inl:27-30  TMap::Add 对已存在键 = 覆盖
	auto Add(const KeyType& InKey, const ValueType& InValue)
	{
		Map.Add(InKey, InValue);
	}
```
```cpp
// FBindingRegistry.inl:21-22  句柄 → wrapper；无 Contains 检查、无旧值回收
	ManagedHandle2BindingAddress.Add(InManagedHandle,
	                                 FBindingValueMapping::ValueType(BindingAddressWrapper, IsNeedFree));
```

**调用上下文**

`AddReference` 的公开入口：`FCSharpEnvironment.inl:298-304`（模板）与 `:306-313`（owner 重载）→ `FBindingRegistry::AddReference`。现存调用点见 §4 的 `AddBindingReference` 证据（9 处）。

**问题**

若同一个 `InManagedHandle` 被注册两次（例如 C# 侧把同一包装对象重复 `Register`，或 `TPropertyData` 的 owner 重载在 `GetBinding(InMember)` 判定失效后重入），`TMap::Add` 会：

1. 直接丢弃旧的 `FBindingAddress`（裸指针 + bool 的 POD），旧 `AddressWrapper` 指针**再无任何引用**，无法释放（即使 `bNeedFree==true` 也一样丢失）；
2. 旧 `AddressWrapper->Value` 对应的 `BindingAddress2ManagedHandle` 条目**残留**，把该地址永久指向旧句柄——之后 `GetObject(address)`（`FBindingRegistry.cpp:40-45`）返回的是已经不再被跟踪的旧句柄。

`FObjectRegistry::AddReference`（`FObjectRegistry.cpp:72-80`）有完全相同的覆盖模式，属同族问题。

**建议**

在覆盖前显式回收，或改为"重复注册不覆盖并返回 false"：

```cpp
if (const auto Existing = ManagedHandle2BindingAddress.Find(InManagedHandle))
{
    BindingAddress2ManagedHandle.Remove(Existing->AddressWrapper->Value);
    delete Existing->AddressWrapper;             // 按 F-ENV-002 修复后无条件回收
}
```
权衡：若 C# 侧真的会重复注册同一句柄，抛错/返回 false 更安全（暴露调用方 bug）；若这是"重新绑定地址"的正常用法，则应显式提供 `Rebind()` 语义而不是复用 `Add`。

**验证方式**

- `grep -n "ManagedHandle2BindingAddress.Add" Source/UnrealCSharp`（当前仅 `FBindingRegistry.inl:21`、`:35`）。
- 在 `Add` 前加 `ensureMsgf(!ManagedHandle2BindingAddress.Contains(InManagedHandle), ...)`，跑完整测试集观察是否触发（能直接判定现存触发点是否存在）。

---

### [F-ENV-004] `FCSharpBind::NotOverrideTypes` 是只增不减的静态 `TSet<TWeakObjectPtr<UStruct>>`，GC 后的条目永久残留、白占内存（无任何压缩/移除路径）

- **类别**: 内存/资源泄漏 | 性能
- **严重度**: **P3**
- **复核结论**: 部分确认（偏差：**哈希语义写错** —— 原文"元素的 `GetTypeHash` 从'活对象指针'退化为 0"经引擎源码核对不成立；"集合只增不减、失效条目永不移除"这一主结论**确认成立**。另修正 `文件` 字段：`NotOverrideTypes.Add` 在 `.inl` 而非 `.cpp:20`）
- **可达性**: 活跃（`Bind<true>` 的入口 `FCSharpEnvironment.cpp:291` 对每个新建 UObject 都会走到）
- **复核证据**: 写入点 `Source/UnrealCSharp/Public/Registry/FCSharpBind.inl:20` `NotOverrideTypes.Add(InStruct)`；唯一清空点 `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:28`（仅在 `FCSharpBind::Initialize`，即每次域重建）；唯一读取点 `FCSharpBind.cpp:421`；定义 `FCSharpBind.cpp:14`、声明 `FCSharpBind.h:86`。引擎侧 —— `Engine\Source\Runtime\CoreUObject\Public\UObject\WeakObjectPtr.h:353-360` `FWeakObjectPtr::GetTypeHash()` = `uint32(ObjectIndex ^ ObjectSerialNumber)`（非 `Get()` 指针哈希），`:172-181` 相等判定为"index+serial 相同 或 两者皆 invalid"；`Runtime\Core\Public\UObject\WeakObjectPtrTemplates.h:494-497` 转发到该函数 → **GC 后条目的哈希不会退化为 0**，也基本不会再被同 index 的探针命中（需 serial 也相同）。因此该集合的性质是"纯内存只增不减"（原结论方向正确），**不是**"哈希退化后误判"
- **级别变动**: 无（P3 维持：增长速率受进程内 UStruct 数量约束，内存量级小）
- **文件**: `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:14,28,421`、`Source/UnrealCSharp/Public/Registry/FCSharpBind.inl:20`、`Source/UnrealCSharp/Public/Registry/FCSharpBind.h:86`
- **函数**: `FCSharpBind::Bind<IsNeedOverride>(UStruct*)`（`FCSharpBind.inl:18-23`）/ `FCSharpBind::CanBind`（`FCSharpBind.cpp:419-432`）/ `FCSharpBind::Initialize`（:26-32）
- **置信度**: 中（增长机制**高**；"实际增长速度"依赖工程中被创建的非 C# 类数量与类对象被 GC 的频率，未实测）

**现状（代码事实）**

```cpp
// FCSharpBind.cpp:14  文件级 static，全局可变状态
TSet<TWeakObjectPtr<UStruct>> FCSharpBind::NotOverrideTypes;
```
```cpp
// FCSharpBind.inl:16-24  唯一写入点
	if constexpr (IsNeedOverride)
	{
		if (!CanBind(InStruct))
		{
			NotOverrideTypes.Add(InStruct);      // ← 加进去
			return false;
		}
	}
```
```cpp
// FCSharpBind.cpp:26-32  唯一清空点（每次环境 Initialize 才会走到）
void FCSharpBind::Initialize()
{
	NotOverrideTypes.Empty();
	...
}
```
```cpp
// FCSharpBind.cpp:419-432  唯一读取点
bool FCSharpBind::CanBind(UStruct* InStruct)
{
	if (NotOverrideTypes.Contains(InStruct))     // ← 缓存"这个类不是 C# 类"
	{
		return false;
	}
	if (auto FoundClass = FReflectionRegistry::Get().GetClass(InStruct)) { return FoundClass->IsOverride(); }
	return false;
}
```

**调用上下文**

`Bind<true>` 的入口全部在 GameThread：
- `FCSharpEnvironment::NotifyUObjectCreated`（`FCSharpEnvironment.cpp:291`，GT 分支）——**每个新建 UObject 都会走到**；
- `FDynamicRegistry` 的 `OnPostClassConstructor` lambda（`FDynamicRegistry.cpp:25`，有 `IsInGameThread()` 守卫）与 `RegisterDynamic`（`:68`）。

**问题**

1. `NotOverrideTypes` 的键是 `TWeakObjectPtr<UStruct>`，但**它只在 `Initialize()` 被清空**；`UStruct` 对象被 GC（蓝图重新编译产生的旧 GeneratedClass、动态类重建、编辑器热重载）后，弱指针失效，而源码里**没有任何遍历/压缩/移除逻辑**。**用引擎源码修正了失效后的语义**：哈希是 `ObjectIndex ^ ObjectSerialNumber`（`Runtime/CoreUObject/Public/UObject/WeakObjectPtr.h:353-360`），**不是** `Get()` 的指针哈希，所以它**不会退化为 0**，也因此基本不会被新探针误命中（需 index 与 serial 同时相同）。该集合的性质是**纯粹只增不减的内存占用**，而**不是**"失效后哈希退化导致误判"。
2. 因此本意是"加速过滤非 C# 类"的缓存会退化成纯内存占用：每遇一个未绑定的类就长一个元素（约 8-12 字节 + 哈希桶）。
3. 反面还有一层：这个 static 集合跨域重载存活（`FCSharpEnvironment::Deinitialize` 不碰它），只在 `FCSharpBind::Initialize`（构造时）清空；如果 `FCSharpBind` 实例化与 `Initialize()` 的调用顺序被改动（见 F-ENV-009），缓存即可跨域存活并产生错误过滤。

**建议**

```cpp
// 1) 把 NotOverrideTypes 从 static 改为 FCSharpBind 的成员（生命周期随环境，天然随域重载清零）；
// 2) 或在插入前/查询时做一次惰性压缩：
void FCSharpBind::CompactNotOverrideTypes()
{
    for (auto It = NotOverrideTypes.CreateIterator(); It; ++It)
    {
        if (!It->IsValid())
        {
            It.RemoveCurrent();
        }
    }
}
// 在 CanBind 的 Contains 命中失败后、或每 N 次插入后调用一次。
```
权衡：成员化最干净但需要把 `Bind`/`CanBind` 从 `static` 改为实例方法（`FCSharpBind` 已是 `FCSharpEnvironment` 的成员指针，具备条件）；惰性压缩改动最小但仍有峰值占用。

**验证方式**

- `grep "NotOverrideTypes"` 全 `Source/`（**实测 5 处**：定义 `FCSharpBind.cpp:14`、清空 `FCSharpBind.cpp:28`、读取 `FCSharpBind.cpp:421`、写入 `FCSharpBind.inl:20`、声明 `FCSharpBind.h:86`；无模块外引用）。原文的"4 处 / `FCSharpBind.cpp:20`"为误记，已修正。
- 加日志：在 `FCSharpBind::Initialize` 与 `Deinitialize` 加 `UE_LOG(... NotOverrideTypes.Num())`，在编辑器里连续做若干次"新建蓝图类 + C# 热重载"后观察是否单调增长。

---

### [F-ENV-005] `BindImplementation(IManagedHandle, const FName&)` 未检查 `AddStructReference` 的返回值；重复调用时已 `FMemory::Malloc` 的结构体内存无法回收

- **类别**: 内存/资源泄漏 | 未检查返回值
- **严重度**: **P3**
- **复核结论**: 部分确认（偏差：**"唯一丢弃返回值的一处"不成立** —— `TPropertyValue.inl`、`TStructPropertyDescriptor.cpp` 等处同样丢弃；但"该处丢弃导致 `FMemory::Malloc` 内存无主"这一主结论**确认成立**，且找到了同 handle 二次注册的**具体触发链**）
- **可达性**: 活跃（C# 侧 `UStruct.Register` 二次调用即可命中）
- **复核证据**: `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:410` `FMemory::Malloc(StructureSize)`、`:412` `InitializeStruct`、`:414` `AddStructReference<true>(...)`（返回值丢弃）、`:416` `return true`。**同 handle 二次注册链**：`Script/UE/Library/StructImplementation.cs:27-35` 的 `UStruct_RegisterImplementation` 用 `HandleData.Alloc(InObject)` 取句柄，而 `Script/Interop/Handle/HandleData.cs:31-42` 的 `Alloc` 以 `ConditionalWeakTable<object, HandleReference>`（**:22**）按**对象身份**缓存 → 同一托管对象**永远得到同一 handle**；故对同一对象调用两次 `UStruct.Register` 会以同一 `InManagedObject` 二次 `TMap::Add`（`FStructRegistry.cpp:98-102`）覆盖旧 `{Address,bNeedFree}`，第一块 `Malloc` 内存永久丢失
- **级别变动**: 无（P3 维持：泄漏量小、需 C# 侧重复 Register 才触发；主结论不变）
- **文件**: `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:394-417`（分配点 `:410-414`）
- **函数**: `FCSharpBind::BindImplementation(const IManagedHandle, const FName&)`
- **置信度**: 高

**现状（代码事实）**

```cpp
// FCSharpBind.cpp:408-417
	const auto StructureSize = InScriptStruct->GetStructureSize() ? InScriptStruct->GetStructureSize() : 1;

	const auto Structure = static_cast<void*>(static_cast<uint8*>(FMemory::Malloc(StructureSize)));   // :410

	InScriptStruct->InitializeStruct(Structure);                                                      // :412

	FCSharpEnvironment::GetEnvironment().AddStructReference<true>(InScriptStruct, Structure, InManagedObject);  // :414 返回值被丢弃

	return true;                                                                                      // :416 无条件 true
```

对应的释放点（在 `FStructRegistry`）：`FStructRegistry.cpp:29-42`（Deinitialize）与 `:126-139`（RemoveReference），两者都以 `Value.bNeedFree` **且 `Value.Value.IsValid()`** 为条件：

```cpp
// FStructRegistry.cpp:126-139
		if (FoundValue->bNeedFree)
		{
			if (FoundValue->Value.IsValid())      // ← 脚本结构体已被 GC 时，FMemory::Free 被跳过
			{
				...
				FMemory::Free(FoundValue->Address);
			}
			FoundValue->Address = nullptr;        // 指针被清空 → 内存彻底不可达
		}
```

**调用上下文**

调用方：`FRegisterStruct.cpp:22-27`（C# 的 `UStruct.Register(handle, structName)`）→ `FCSharpEnvironment::Bind(handle, FName)`（`FCSharpEnvironment.cpp:469-472`）→ 本函数。每次 C# 侧对同一 handle 调 `Register` 都会重新 `Malloc` + `InitializeStruct`，并以**同一个 `InManagedObject` 作为键**写入 `FStructRegistry::ManagedHandle2StructAddress`（`FStructRegistry.cpp:98`，`TMap::Add` 覆盖）。

**问题**

1. `AddStructReference<true>` 返回 `false`（`StructRegistry == nullptr`，或注册表内部失败）时，`Structure` 这块内存**没有任何人记录**，直接泄漏；函数仍返回 `true` 告知调用方成功。
2. 同一 handle 重复 `Register`：`TMap::Add` 覆盖上一个 `{Address, bNeedFree}`，旧 `Address` 丢失 → 旧结构体内存永久泄漏（即使 `bNeedFree==true` 也不会被 free）。而 C# 侧确实会复用同一个包装对象的 Handle（`FRegisterStruct.cpp:22` 的 `InManagedHandle` 就是被注册对象自己的 handle）。
3. `FStructRegistry` 的释放又把 `FMemory::Free` 挂在 `Value.Value.IsValid()` 上（脚本结构体对象被 GC 后跳过 free），使泄漏面进一步扩大。

**建议**

```cpp
	if (!FCSharpEnvironment::GetEnvironment().AddStructReference<true>(InScriptStruct, Structure, InManagedObject))
	{
		InScriptStruct->DestroyStruct(Structure);      // 与 InitializeStruct 对称
		FMemory::Free(Structure);
		return false;                                  // 不要把失败当成功
	}
```
并在 `FStructRegistry::AddReference` 中，对已存在键做显式回收（同 F-ENV-003 的处理）。

**验证方式**

- `grep "AddStructReference"` 全 `Source/`（**实测 22 处**：声明/定义 `FCSharpEnvironment.h:110,113`、`FCSharpEnvironment.inl:118`、`FCSharpEnvironment.cpp:548`；调用 `TPropertyValue.inl:264,282,287,317,337,343,494,512,517,547,566,571`（12 处）、`TConstructorHelper.inl:30,37`、`FStructPropertyDescriptor.cpp:14,23,66`、`FCSharpBind.cpp:414`）。原文的"20 处""`FCSharpBind.cpp:414` 是唯一丢弃返回值的一处"两句均误，已修正 —— `:414` 的**特殊性**不是"唯一不检查返回值"（`TPropertyValue.inl`、`FStructPropertyDescriptor.cpp` 同样丢弃），而是它丢弃的是"刚 `FMemory::Malloc` 出来、此外没有任何持有者的内存"的登记结果。
- 用例：C# 侧对同一对象连续调用两次 `UStruct.Register(...)`，用 `Malloc`/`Free` 的统计（`-trace=memory` 或 `FMemory::Malloc` 包装计数）观察是否净增长。

---

### [F-ENV-006] 信号处理块：SIGSEGV 处理函数直接 return 造成无限故障循环；处理函数内调用 `UE_LOG`/`GLog->Flush` 非异步信号安全；macOS 分支 `SignalActions[Signal]` 可能插入零值 handler；`Deinitialize` 从不恢复

- **类别**: 平台兼容 | 未定义行为 | 并发/线程安全
- **严重度**: **P2**
- **复核结论**: 确认（信号安装点、handler 体、`Deinitialize` 无还原三处代码事实逐字复核一致；§3 建议与触发条件描述成立。补正一处措辞：只有**非 Mac** 分支才可能形成"无限故障循环"，Mac 分支在 `:29` 先恢复旧 action 再返回，第二次信号会交给旧 handler）
- **可达性**: 活跃（`:120-142` 在所有平台执行；Windows 走 `:140` `signal(SignalType, SignalHandler)`）
- **复核证据**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:22-31`（`SignalHandler`，`:24` `UE_LOG`、`:26` `GLog->Flush()`、`:29` Mac `sigaction(..., &SignalActions[Signal], ...)`）；`:19` `TMap<int32, struct sigaction> SignalActions;`（仅 `PLATFORM_MAC`，非 static 全局）；`:101-118` 信号表、`:120-142` 安装；`grep "signal(\|sigaction("` 全 `Source/` 实测命中 **4 处**：`:29,133,137,140`（另 2 处在被排除的 `Source/ThirdParty/LeanCLR/src/runtime/platform/rt_console.cpp:411,415`），**无任何还原点**；`grep "SignalActions"` 命中 `:19,29,131,133`，与原文一致。`Deinitialize()`（`:147-269`）全文无 `signal`/`sigaction`
- **级别变动**: 无（P2 维持：只在崩溃路径上恶化可诊断性，不影响正常运行）
- **文件**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:18-31,101-142,147-269`
- **函数**: `SignalHandler(int32)`（文件级，非成员）/ `FCSharpEnvironment::Initialize` / `FCSharpEnvironment::Deinitialize`
- **置信度**: 高（代码事实可直接读出）；"是否真的会无限循环"取决于平台对 `signal()` 语义的实现，见"问题"第 1 点

**现状（代码事实）**

```cpp
// FCSharpEnvironment.cpp:18-31
#if PLATFORM_MAC
TMap<int32, struct sigaction> SignalActions;      // :19 非 static 全局可变容器
#endif

void SignalHandler(int32 Signal)
{
	UE_LOG(LogUnrealCSharp, Error, TEXT("%s"), *FDomain::GetTraceback());   // :24 非 async-signal-safe：FString + 反射调用 + 日志锁

	GLog->Flush();                                                          // :26 同上

#if PLATFORM_MAC
	sigaction(Signal, &SignalActions[Signal], nullptr);                     // :29 operator[] 若不存在会插入零值
#endif
}
```

```cpp
// FCSharpEnvironment.cpp:120-142（安装点，位于 Initialize()）
	for (const auto SignalType : SignalTypes)
	{
#if PLATFORM_MAC
		struct sigaction SigAction;
		FMemory::Memzero(&SigAction, sizeof(struct sigaction));
		SigAction.sa_handler = SignalHandler;
		sigemptyset(&SigAction.sa_mask);
		if (!SignalActions.Contains(SignalType))
		{
			sigaction(SignalType, &SigAction, &SignalActions.Add(SignalType));   // :133 只在这一支保存旧 handler
		}
		else
		{
			sigaction(SignalType, &SigAction, nullptr);                          // :137 这一支不保存
		}
#else
		signal(SignalType, SignalHandler);                                         // :140 非 Mac：不保存旧 handler
#endif
	}
```

`Deinitialize()`（`:147-269`）中**没有任何** `signal`/`sigaction` 还原调用（全文检索见 §6 死亡/遗漏清单）。

**调用上下文**

`Initialize()` 由 `FCSharpEnvironment::OnUnrealCSharpModuleActive`（`:327-330`）在一次模块激活时调用一次；域重载会反复进出（`FCSharpCompilerRunnable.cpp:223-230`、`FEngineListener.cpp:38-43`）。

**问题**

1. **不终止故障**：非 Mac 分支用 `signal()` 安装的 handler 在 SIGSEGV/SIGABRT/SIGFPE 时只是记录日志并 `return`。从同步信号（如 SIGSEGV）的 handler 返回后，控制流回到出错指令，**再次触发同一信号**——形成"记录→返回→再崩"的无限循环（日志被反复刷写，进程既不退出也不产出 crash dump）。
2. **非 async-signal-safe 调用**：`:24` 的 `FDomain::GetTraceback()` 会走反射注册表、反射调用 C# 方法并构造 `FString`（`FDomain.cpp:112-129`），`UE_LOG` 会取日志互斥锁。在信号处理函数里做这些都属未定义行为，**最容易观察到的表现是死锁（卡死在日志锁上），比原来的崩溃更难诊断**。
3. **macOS 的 `SignalActions[Signal]`**：若 `Signal` 不在表中（例如 `Initialize` 之前就发生信号），`operator[]` 会**插入一个零值 `sigaction`** 并把它安装为 handler——即把 handler 设为 nullptr，下一次同名信号直接跳到地址 0。此外 `:133` 只在"首次安装"时保存旧 handler，`:137` 不保存，因此第 2 次及以后的重载/重装会丢失旧 handler 的备份。
4. **不恢复**：`Deinitialize()` 完全不还原 handler。模块在编辑器里可能被卸载/热重载（代码段被换出），此时仍指向旧模块 `SignalHandler` 的函数指针会留在进程里 → 之后任意一次信号都会跳入已卸载代码。SIGINT/SIGTERM 这类"进程生命周期"信号的接管还会持续影响引擎其他子系统（例如覆盖 UE 自己的 Ctrl-C 处理）。

**建议**

```cpp
void SignalHandler(int32 Signal)
{
	// 1) 只做异步信号安全的事：write(2) 到 STDERR + 立即恢复默认动作
	const char* Name = strsignal(Signal);            // 或用一张静态表
	(void)write(2, Name, strlen(Name));
	(void)write(2, "\n", 1);

	signal(Signal, SIG_DFL);                         // 2) 交还默认行为，避免无限故障循环
	raise(Signal);                                   //    重新触发以走正常的 crash 处理/crash dump
}
```
```cpp
// 3) Initialize 中保存旧 handler 到成员，Deinitialize 中还原（所有平台都做）
private:
	TMap<int32, FSignalHandlerBackup> SignalBackups;   // 保存 sigaction 或 signal() 的返回值
```
权衡：`UE_LOG`/traceback 对定位托管侧崩溃很有价值，但**必须在信号安全的上下文里做**——可行替代是"handler 只用 `write` 打印固定字符串并落一个标记文件，由下一次 GameThread 心跳读取该标记再去取 traceback"，代价是多一层间接与丢失"崩溃现场"，收益是进程能正常 dump 而不是挂死。

**验证方式**

- 用例 1：在编辑器里触发一次 SIGSEGV（例如临时引入空指针解引用），观察是否反复打印同一行 UE_LOG 且进程不退出（复现则确认第 1 点）。
- 用例 2：把 handler 换成 `abort()` 前的断点，确认第 2 点的锁：crash 报告中调用栈停在日志互斥锁上。
- `grep -n "signal(\|sigaction(" Source/`（当前命中 `FCSharpEnvironment.cpp:29,133,137,140`，无还原点）。
- `grep -n "SignalActions" Source/`（当前命中 `:19,29,131,133`）。

---

### [F-ENV-007] `OnAsyncLoadingFlushUpdate` 用裸 `UObject*` 中间数组搬运"仅弱引用"的对象去绑定；且 CDO 的绑定路径完全押在 async flush 事件上、无兜底（事件"每帧必来"已由引擎源码确认，故降级为设计脆弱性）

- **类别**: 未定义行为 | 内存/资源泄漏 | 并发/线程安全
- **严重度**: **P3**
- **复核结论**: 部分确认（偏差：**第 2 条"事件不来就永久滞留"被引擎源码证伪** —— `OnAsyncLoadingFlushUpdate` 在默认配置下**每帧都会广播**，CDO 不会长期滞留。第 1 条"裸指针窗口"（代码事实与风险机制）与第 3 条"重复 `#if` 块 + 引擎私有成员访问"**确认成立**）
- **可达性**: 活跃（`NotifyUObjectCreated` 由 `GUObjectArray` 在任意线程回调；`OnAsyncLoadingFlushUpdate` 每帧执行）
- **复核证据**: 代码事实 —— `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:373` `TArray<UObject*> PendingBindObjects;`、`:381` 锁内只拷贝 `FWeakObjectPtr`、`:388` 有效性检查、`:417` 转入裸指针数组、`:432-451` 解引用（`:434` `HasAnyFlags`、`:446` `PendingBindObject->GetClass()`）；`:397-412` 同一 `#if UE_E_INTERNAL_OBJECT_FLAGS_ASYNC_LOADING` 块重复两遍，且经 `ACCESS_PRIVATE_MEMBER_PROPERTY`（`:16`）+ `~RF_AllFlags` 读引擎私有 `ObjectFlags`。**引擎侧（推翻"事件不来"）**：`Engine\Source\Runtime\Core\Private\Misc\CoreDelegates.cpp:276` 为普通多播；广播点 `Runtime\CoreUObject\Private\Serialization\AsyncLoading2.cpp:9583`（"Call update callback once per tick on the game thread"，位于 `FAsyncLoadingThread2::TickAsyncLoadingFromGameThread`，:9519）与 `:9187`；该函数由 `ProcessLoadingFromGameThread`（:11034,11044）调用 ← `ProcessAsyncLoading`（`Runtime\CoreUObject\Private\Serialization\AsyncPackageLoader.cpp:377-382`）← `StaticTick`（`Runtime\CoreUObject\Private\UObject\UObjectGlobals.cpp:895-900`）← **每帧** `EditorEngine.cpp:1757-1761`（编辑器）与 `GameEngine.cpp:1824-1829`（Game）—— 二者均以 `if (!GUseUnifiedTimeBudgetForStreaming)` 为条件，而 `GUseUnifiedTimeBudgetForStreaming = 0` 是默认值（`Runtime\Engine\Private\CoreSettings.cpp:12`）。故"编辑器空闲态/纯 -game 会话不广播"不成立
- **级别变动**: 无（P3 维持：仅剩"裸指针窗口内触发 GC"这一未实测假设 + 引擎私有成员访问的版本脆弱性）
- **文件**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:276-300,369-452`
- **函数**: `FCSharpEnvironment::OnAsyncLoadingFlushUpdate` / `FCSharpEnvironment::NotifyUObjectCreated`
- **置信度**: 高（代码事实全部复核一致；但"CDO 依赖 flush 事件、事件不来则滞留"这一**后果**经引擎源码核对后被削弱，见"问题"第 2 点）

**现状（代码事实）**

```cpp
// FCSharpEnvironment.cpp:369-421（摘取关键行）
void FCSharpEnvironment::OnAsyncLoadingFlushUpdate()
{
	TArray<int32> RemovedIndexes;
	TArray<UObject*> PendingBindObjects;          // :373 ← 裸强指针数组（对象本身无人强引用）

	{
		TArray<FWeakObjectPtr> LocalAsyncLoadingObjectArray;
		{
			FScopeLock Lock(&CriticalSection);
			LocalAsyncLoadingObjectArray.Append(AsyncLoadingObjectArray);   // :381 锁内拷贝弱指针
		}

		for (auto i = LocalAsyncLoadingObjectArray.Num() - 1; i >= 0; --i)
		{
			auto ObjectPtr = LocalAsyncLoadingObjectArray[i];
			if (!ObjectPtr.IsValid()) { RemovedIndexes.Add(i); continue; }   // :388 有效性检查（好的部分）
			auto Object = ObjectPtr.Get();
			...
			PendingBindObjects.Add(Object);      // :417 ← 从此以裸指针持有
			RemovedIndexes.Add(i);
		}
	}
	...
	for (const auto& PendingBindObject : PendingBindObjects)      // :432
	{
		if (PendingBindObject->HasAnyFlags(EObjectFlags::RF_ClassDefaultObject))    // :434
		{
			FCSharpBind::BindClassDefaultObject(PendingBindObject);
		}
		else
		{
			Bind<true>(PendingBindObject);                                             // :440
		}
		// :443-450 之后仍然继续解引用 PendingBindObject->GetClass()
	}
}
```
```cpp
// FCSharpEnvironment.cpp:276-300  CDO 一律进延迟数组（即使就在 GameThread）
	if (InObject->HasAnyFlags(EObjectFlags::RF_ClassDefaultObject))
	{
		FScopeLock Lock(&CriticalSection);
		AsyncLoadingObjectArray.Add(InObject);      // :284
		return;
	}
```
延迟数组的**唯一**消费点是 `FCoreDelegates::OnAsyncLoadingFlushUpdate`（注册于 `:87-88`，注销于 `:174`）。

**调用上下文**

- 生产者：`FUObjectListener::NotifyUObjectCreated`（`FUObjectListener.cpp:43`，由 `GUObjectArray` 在任意线程回调，见 `FUObjectListener.cpp:29`）。
- 消费者：`OnAsyncLoadingFlushUpdate`（绑在 `FCoreDelegates::OnAsyncLoadingFlushUpdate` 上）。
- 另一处 CDO 绑定路径：`FCSharpBind::OnCSharpEnvironmentInitialize`（`FCSharpBind.cpp:536-544`）在 `Initialize()` 广播时会遍历全部 `UClass` 兜底绑定 CDO——因此"CDO 没被及时绑定"通常会在下一次域重载时补上，但两次重载之间 C# 侧看不到这些 CDO。

**问题**

1. **裸指针窗口**：`PendingBindObjects` 里的对象在 `AsyncLoadingObjectArray` 中只有弱引用（`:381` 拷贝的就是弱指针）。`:417` 之后循环体只持裸指针，而 `:440-450` 会调用 `Bind<true>` → `FClassReflection::NewObject()` / `ConstructorObject()`（`FCSharpBind.inl:82`、`FClassReflection.cpp:704-744`），即**回调进托管代码并可能再回调进 UE**；若期间发生 UE GC，则 `PendingBindObjects` 中的对象可能在 `:434`/`:446` 被解引用前已被回收。
2. **事件依赖（修正：后果被引擎源码削弱）**：`RF_ClassDefaultObject` 的对象（`:280-287`）无论线程都被丢进延迟数组，而清空该数组的唯一时机是 `OnAsyncLoadingFlushUpdate`。这条"唯一清空点"是代码事实，但**"事件可能长期不来"不成立**：该多播由异步加载的 GameThread tick 每次广播（`AsyncLoading2.cpp:9583`），而该 tick 由 `StaticTick`（`UObjectGlobals.cpp:895-900`）经 `EditorEngine.cpp:1757-1761` / `GameEngine.cpp:1824-1829` **每帧**调用（默认 `GUseUnifiedTimeBudgetForStreaming = 0`，`CoreSettings.cpp:12`）。因此滞留窗口实际是"创建后到本帧 flush 之间"（亚帧级），只是在异步加载被挂起（`IsAsyncLoadingSuspended`）或引擎不进入正常 Tick 的上下文（commandlet/cook 等）时才可能拉长。→ 本条降为**设计脆弱性**（"正确性完全押在一个引擎事件上、且没有兜底绑定路径"），不是可观测的滞留问题。
3. `:397-412` 把同一段 `#if UE_E_INTERNAL_OBJECT_FLAGS_ASYNC_LOADING` 条件**重复写了两遍**（`Object` 与 `Object->GetClass()`），且通过 `ACCESS_PRIVATE_MEMBER_PROPERTY`（`:16`）+ `~RF_AllFlags` 直接读引擎私有 `ObjectFlags`——这是随引擎版本变化的脆弱点，一旦偏移/语义变化就会静默误判（该表达式没有 `!= 0` 显式比较，也没有注释解释 `~RF_AllFlags` 的意图）。

**建议**

```cpp
	// 1) 用 TStrongObjectPtr 或 AddToRoot/RemoveFromRoot 保护窗口
	TArray<TStrongObjectPtr<UObject>> PendingBindObjects;   // 或 GUObjectArray 引用保护
	// 2) CDO 不必等 flush：在 GT 上可直接绑定
	if (InObject->HasAnyFlags(EObjectFlags::RF_ClassDefaultObject))
	{
		if (IsInGameThread()) { FCSharpBind::BindClassDefaultObject(InObject); return; }
		FScopeLock Lock(&CriticalSection);
		AsyncLoadingObjectArray.Add(InObject);
		return;
	}
	// 3) 抽出谓词函数消除重复的 #if 块
	static bool IsPendingAsyncLoad(const UObject* Object);
```
权衡：`TStrongObjectPtr` 会延长对象生命周期（对半加载对象可能掩盖"加载失败"），因此更稳的是"在 GT 上直接处理 + 只在真正的异步线程路径上排队"，即第 2 点与第 1 点一起改。

**验证方式**

- 用例：编辑器里只创建 CDO（新建蓝图类）而不触发任何异步加载，`grep`/日志打印 `AsyncLoadingObjectArray.Num()`，确认其不归零。
- 在 `:417` 与 `:440` 之间插入 `CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS)`（仅调试构建）观察是否出现已回收对象。
- `grep "OnAsyncLoadingFlushUpdate"` 全 `Source/`（**实测 7 处**：`FCSharpEnvironment.cpp:87,88,172,174,369`、声明 `FCSharpEnvironment.h:42`、成员句柄 `FCSharpEnvironment.h:316`；原文漏记 `:172` 与 `h:316`）。

---

### [F-ENV-008] 静态初始化/析构顺序未定义：`FCSharpEnvironment::Environment` 的构造/析构都跨 TU 触碰 `FUnrealCSharpModuleDelegates` 的静态多播委托

- **类别**: 未定义行为 | 生命周期
- **严重度**: **P2**
- **复核结论**: 确认（构造/析构跨 TU 触碰静态多播委托是代码事实；"跨 TU 静态初始化顺序未指定"是 C++ 标准事实，结论成立）
- **可达性**: 活跃（静态对象的构造在模块加载时、析构在模块卸载/进程退出时都会执行）
- **复核证据**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:33` `FCSharpEnvironment FCSharpEnvironment::Environment;`、`:35-42` 构造函数 `AddRaw` 两个委托、`:44-55` 析构函数 `Remove` 两个委托；委托定义在**另一编译单元** `Source/UnrealCSharp/Private/Delegate/FUnrealCSharpDelegates.cpp:3`(`OnUnrealCSharpModuleActive`)、`:5`(`OnUnrealCSharpModuleInActive`)、`:7-8`(`OnCSharpEnvironmentInitialize`) —— 逐行复核一致
- **级别变动**: 无（P2 维持：进程退出期偶发 UAF，非正常路径）
- **文件**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:33,35-55`
- **函数**: `FCSharpEnvironment::FCSharpEnvironment()` / `~FCSharpEnvironment()` / 静态对象 `FCSharpEnvironment::Environment`
- **置信度**: 中（跨 TU 顺序未定义是标准事实；"实际是否被零初始化掩盖"为未验证假设，见"问题"）

**现状（代码事实）**

```cpp
// FCSharpEnvironment.cpp:33
FCSharpEnvironment FCSharpEnvironment::Environment;

// FCSharpEnvironment.cpp:35-42  构造函数跨 TU 触碰另一个静态对象
FCSharpEnvironment::FCSharpEnvironment()
{
	OnUnrealCSharpModuleActiveDelegateHandle = FUnrealCSharpModuleDelegates::OnUnrealCSharpModuleActive.AddRaw(
		this, &FCSharpEnvironment::OnUnrealCSharpModuleActive);
	OnUnrealCSharpModuleInActiveDelegateHandle = FUnrealCSharpModuleDelegates::OnUnrealCSharpModuleInActive.AddRaw(
		this, &FCSharpEnvironment::OnUnrealCSharpModuleInActive);
}

// FCSharpEnvironment.cpp:44-55  析构函数同样触碰
FCSharpEnvironment::~FCSharpEnvironment()
{
	if (OnUnrealCSharpModuleInActiveDelegateHandle.IsValid())
	{
		FUnrealCSharpModuleDelegates::OnUnrealCSharpModuleInActive.Remove(OnUnrealCSharpModuleInActiveDelegateHandle);
	}
	if (OnUnrealCSharpModuleActiveDelegateHandle.IsValid())
	{
		FUnrealCSharpModuleDelegates::OnUnrealCSharpModuleActive.Remove(OnUnrealCSharpModuleActiveDelegateHandle);
	}
}
```

两个多播委托的定义在**另一个编译单元**：

```cpp
// Source/UnrealCSharp/Private/Delegate/FUnrealCSharpDelegates.cpp:3
FUnrealCSharpModuleDelegates::FOnUnrealCSharpModuleActive FUnrealCSharpModuleDelegates::OnUnrealCSharpModuleActive;
// :5  ... OnUnrealCSharpModuleInActive;   :7-8 ... OnCSharpEnvironmentInitialize;
```

**调用上下文**

`FCSharpEnvironment` 由 `FCSharpEnvironment::GetEnvironment()`（`:271-274`）在全模块广泛使用；`Environment` 的构造发生在模块 DLL 的静态初始化阶段，析构发生在模块卸载/进程退出。

**问题**

1. `FCSharpEnvironment.cpp` 中的 `Environment` 与 `FUnrealCSharpDelegates.cpp` 中的多播委托分属不同 TU，**跨 TU 的静态初始化顺序是未指定的**。若 `Environment` 先构造，则 `AddRaw` 作用在尚未执行动态初始化的委托对象上——技术上属 UB（对象生命周期未开始）。实践中 `TMulticastDelegate` 的内部 `TArray` 依赖"静态存储零初始化即合法空数组"这一 UE 惯例，因此**通常不会立刻出错**（这也是我把置信度定为中而非高的原因）。
2. 析构侧风险更大：若委托对象先于 `Environment` 构造，则它按逆序**先被析构**，随后 `~FCSharpEnvironment()` 对已析构对象调用 `Remove(...)` → 真正意义上的 UAF。该路径只在模块被卸载/进程退出时走，属于"偶发崩溃"类问题。
3. 另外，`~FCSharpEnvironment()` **不调用 `Deinitialize()`**（见 F-ENV-010），所以析构的唯一结果就是"摘掉两个委托"，这进一步说明这里的析构逻辑更像是为"模块卸载"补形式，而非真正的资源回收。

**建议**

```cpp
// 1) 用函数局部静态（C++11 起保证线程安全的懒初始化）替代文件级静态对象，
//    把构造点推迟到首次 GetEnvironment() 调用（此时模块已加载、委托已构造）：
FCSharpEnvironment& FCSharpEnvironment::GetEnvironment()
{
    static FCSharpEnvironment Environment;   // 首次调用时构造，顺序敏感期已过
    return Environment;
}
// 2) 析构中显式调用 Deinitialize()（见 F-ENV-010）。
```
权衡：函数局部静态把"构造时机"变成"首次使用时机"，需确认首次使用一定在 `GUObjectArray` 监听安装之前（`FUObjectListener::OnUnrealCSharpModuleActive` 会调用 `GetEnvironment()`，因此顺序仍成立）。若担心析构顺序（局部静态在 `atexit` 链上析构），可再包一层 `TUniquePtr` 且故意不释放（进程退出时泄漏少量固定对象，换取可控顺序）。

**验证方式**

- 在每个 TU 的静态对象构造函数/析构函数里打印 `this` 与一句日志，观察 `Environment` 与 `FUnrealCSharpModuleDelegates::OnUnrealCSharpModuleActive` 的构造顺序在不同平台/链接顺序下是否改变。
- 反汇编/调试器断点：在 `~FCSharpEnvironment` 中打印 `&FUnrealCSharpModuleDelegates::OnUnrealCSharpModuleActive` 指向的内存是否已被 `TArray` 析构（`Data == nullptr && ArrayNum == 0` 无法区分，需看 `InvocationList` 的内存魔数）。

---

### [F-ENV-009] `Initialize()` 无重入/重复调用保护：二次调用会泄漏 13 个对象（11 个注册表 + `FCSharpBind` + `FDomain`，关闭 `UE_F_OPTIONAL_PROPERTY` 时 12 个），并重复注册委托与信号处理器

- **类别**: Bug | 内存/资源泄漏
- **严重度**: **P2**
- **复核结论**: 确认（"无重入/幂等保护""`Initialize` 无条件 `new` 13 个对象""`OnAsyncLoadingFlushUpdateHandle` 缺 `Reset()`"三处代码事实逐行复核一致；"当前所有调用方都有 `EState` 守卫、无现网触发路径"也经 `UnrealCSharpCore.cpp:19-37` 复核一致）
- **可达性**: 潜伏（危害路径当前被上层 `EState` 守卫拦截，**不是**当前会执行到的路径；但 `Initialize()/Deinitialize()` 是 `UNREALCSHARP_API` 类上的 public 方法（`FCSharpEnvironment.h:24,26`），任何外部调用方或未来新增调用方即可命中，故按"潜伏"而非"不可达"计）
- **复核证据**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:57-145`（`:59` `Domain = new FDomain()` … `:85` `OptionalRegistry`，共 **13 个** `new`；`:87-88` 注册 async flush；`:120-142` 重复安装信号；`:144` 广播）；释放点 `:147-269`（**13 段** `if (X != nullptr) { delete X; X = nullptr; }`）；缺口 `:172-175` 只 `Remove` 不 `Reset`，对照 `:156-168` 的 `OnBlueprintCompiledHandle.Reset()`；`grep "Initialize()|Deinitialize()"` 在 `FCSharpEnvironment.cpp` 实测 4 处：定义 `:57`/`:147`、唯一调用 `:329`/`:334`。上游守卫 `Source/UnrealCSharpCore/Private/UnrealCSharpCore.cpp:19-27`（`State == Inactive` 才 `Activate`）、`:29-37`（`State != Inactive` 才 `Deactivate`）；编辑器内 `Source/UnrealCSharp/Private/UnrealCSharp.cpp:36-48`/`:50-53` 转发广播
- **级别变动**: 无（P2 维持：API 无防护属隐患，非现网错误）
- **文件**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:57-145`（分配点 `:59-85`）、`:147-269`（释放点）
- **函数**: `FCSharpEnvironment::Initialize` / `FCSharpEnvironment::Deinitialize` / `OnUnrealCSharpModuleActive`
- **置信度**: 中（"当前调用方都有 State 守卫，因此暂无已知触发路径"是已核实的事实；风险来自 API 本身无防护）

**现状（代码事实）**

```cpp
// FCSharpEnvironment.cpp:57-85  Initialize 无条件 new，且不检查是否已初始化
void FCSharpEnvironment::Initialize()
{
	Domain = new FDomain();                 // :59
	DynamicRegistry = new FDynamicRegistry();
	CSharpBind = new FCSharpBind();
	ClassRegistry = new FClassRegistry();
	ReferenceRegistry = new FReferenceRegistry();
	ObjectRegistry = new FObjectRegistry();
	StructRegistry = new FStructRegistry();
	ContainerRegistry = new FContainerRegistry();
	DelegateRegistry = new FDelegateRegistry();
	MultiRegistry = new FMultiRegistry();
	StringRegistry = new FStringRegistry();
	BindingRegistry = new FBindingRegistry();
#if UE_F_OPTIONAL_PROPERTY
	OptionalRegistry = new FOptionalRegistry();
#endif
	OnAsyncLoadingFlushUpdateHandle = FCoreDelegates::OnAsyncLoadingFlushUpdate.AddRaw(   // :87 重复注册
		this, &FCSharpEnvironment::OnAsyncLoadingFlushUpdate);
	...
	FUnrealCSharpModuleDelegates::OnCSharpEnvironmentInitialize.Broadcast();             // :144
}
```

对应的释放只在 `Deinitialize()` 中（`:147-269`，逐项 `delete` + 置 `nullptr`）。

**调用上下文**

- 唯一 Active 入口：`OnUnrealCSharpModuleActive`（`:327-330`）← `FUnrealCSharpModule::OnUnrealCSharpCoreModuleActive`（`UnrealCSharp.cpp:36-48`）← `FUnrealCSharpCoreModule::Activate()`（`UnrealCSharpCore.cpp:19-27`，`State == Inactive` 守卫）。
- 唯一 InActive 入口：`OnUnrealCSharpModuleInActive`（`:332-335`）← `Deactivate()`（`UnrealCSharpCore.cpp:29-37`，`State != Inactive` 守卫）。
- 次级入口：`FEngineListener::OnPreBeginPIE`（`FEngineListener.cpp:34-44`）在 `IsOutdated()` 时先 `Deactivate()` 再 `Activate()`，成对。

**问题**

1. `Initialize()` 不是幂等的：若被调用两次而未插入 `Deinitialize()`，12 个注册表 + `FDomain` 的旧实例**全部泄漏**（且旧的 `OnAsyncLoadingFlushUpdate` 委托句柄被覆盖，导致旧句柄永远无法 `Remove`），同时信号处理器被重复安装（`:120-142`）。新的 `CSharpBind` 会在构造时再注册一次 `OnCSharpEnvironmentInitialize`（`FCSharpBind.cpp:30-31`），旧的实例因为被覆盖而**永远不会 `Deinitialize`**（`~FCSharpBind` 不会被调用，因为它已被指针覆盖且无人 delete）→ 委托里残留悬垂 `this`。
2. 当前所有调用方都有 `State` 守卫，因此我不声称存在现网触发路径；但 `Initialize()/Deinitialize()` 是 `UNREALCSHARP_API` 类上的 **public** 方法（`FCSharpEnvironment.h:24,26`），外部代码（或未来新增的调用方）可以直接调用；而 `FCSharpEnvironment` 在整个插件源码中没有任何 `check` 断言（`grep "check("` 在 `Source/` 下唯一命中是 `UnrealCSharpEditorStyle.cpp:25` 的 `ensure(StyleInstance.IsUnique())`）。
3. 另一个不对称：`Deinitialize()` 对 `OnAsyncLoadingFlushUpdateHandle` **只 Remove 不 Reset**（`:172-175`），而同函数内的蓝图委托句柄都做了 `Reset()`（`:160,167`）。这使该句柄在 `Deinitialize` 之后仍 `IsValid()`，第二次 `Deinitialize` 会拿一个陈旧句柄去 `Remove`。`FDelegateHandle::IsValid()` 仅判断 `HandleId != 0`，因此这是"看似有效但已失效"的语义错误（UE 的 `Remove` 对找不到的句柄是安全的，故不构成崩溃，属隐患/一致性）。

```cpp
// FCSharpEnvironment.cpp:172-175  ← 缺 Reset()
	if (OnAsyncLoadingFlushUpdateHandle.IsValid())
	{
		FCoreDelegates::OnAsyncLoadingFlushUpdate.Remove(OnAsyncLoadingFlushUpdateHandle);
	}
// FCSharpEnvironment.cpp:156-168  ← 同类代码都 Reset() 了
		if (OnBlueprintCompiledHandle.IsValid())
		{
			GEditor->OnBlueprintCompiled().Remove(OnBlueprintCompiledHandle);
			OnBlueprintCompiledHandle.Reset();
		}
```

**建议**

```cpp
void FCSharpEnvironment::Initialize()
{
	check(IsInGameThread());
	if (Domain != nullptr)                 // 或引入 bInitialized 标志
	{
		UE_LOG(LogUnrealCSharp, Warning, TEXT("Initialize called twice; deinitializing first."));
		Deinitialize();
	}
	Domain = new FDomain();
	...
}
// 并在 Deinitialize 中补齐：
	OnAsyncLoadingFlushUpdateHandle.Reset();
	OnUnrealCSharpModuleActiveDelegateHandle / InActive 不需要 Reset（需保持注册）
```
权衡：自动 `Deinitialize()` 会让"重复初始化"变成静默行为而非崩溃；若希望严格暴露调用方错误，可改为 `ensureMsgf(Domain == nullptr, ...)` + 直接 `return`，由调用方修正时序。建议至少加 `check(IsInGameThread())`，因为所有注册表都假定 GameThread（见 §6.3）。

**验证方式**

- 用例：在 `FCSharpEnvironment::Initialize()` 末尾加 `ensure(Domain == nullptr)`，跑"编辑器启动→C# 编译→PIE→停止"完整循环；当前应不触发，若触发即证明存在未成对的 Activate。
- `grep -n "Initialize()" Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp`（命中 `:57`、`:329`）。

---

### [F-ENV-010] `~FCSharpEnvironment()` 不调用 `Deinitialize()`：只要 InActive 广播没走到，全部注册表、`FClassDescriptor`、wrapper 与 GCHandle 都不会释放

- **类别**: 内存/资源泄漏
- **严重度**: **P2**
- **复核结论**: 确认（析构体 8 行、只摘两个委托、不触碰任何注册表；`Deinitialize()` 是唯一释放点且只由 `:334` 驱动 —— 逐行复核一致）
- **可达性**: 活跃（`~FCSharpEnvironment()` 在静态析构/模块卸载时必然执行；但在**正常关停路径**上 `Deinitialize()` 已被 `OnPreExit → SetActive(false) → Deactivate()` 提前调用，故本条的危害只在"InActive 广播未走到"的非正常路径上兑现）
- **复核证据**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:44-55`（析构，全文无 `delete`/`Deinitialize`）、`:147-269`（13 段释放）、`:332-335` `OnUnrealCSharpModuleInActive → Deinitialize()`；正常关停链 `Source/UnrealCSharpCore/Private/Listener/FEngineListener.cpp:58-61`（`OnPreExit → SetActive(false)`）→ `:76-79`（`Deactivate()`）；编辑器模块 `.Function("SetActive")` 路径 `Source/UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp:136,140` 成对；非正常路径示例 `Source/Compiler/Private/FCSharpCompilerRunnable.cpp:216-219`（`GExitPurge` 早退）
- **级别变动**: 无（P2 维持：条件性、非正常关停才发生的跨语言泄漏 + 无兜底）
- **文件**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:44-55`（析构）vs `:147-269`（`Deinitialize`）
- **函数**: `FCSharpEnvironment::~FCSharpEnvironment()` / `FCSharpEnvironment::Deinitialize()`
- **置信度**: 高

**现状（代码事实）**

析构函数体只有 8 行（`:44-55`），只做两件事：`Remove` 两个 `FUnrealCSharpModuleDelegates` 句柄。**没有** `delete Domain`、没有 `delete BindingRegistry`，也没有调用 `Deinitialize()`。而 `Deinitialize()`（`:147-269`）是唯一承担释放职责的函数，只由 `OnUnrealCSharpModuleInActive`（`:332-335`）驱动。

**调用上下文**

正常关停路径确实会走到：`FEngineListener::OnPreExit`（`FEngineListener.cpp:58-61`）→ `SetActive(false)` → `Deactivate()`（`:78`）→ 广播 → `Deinitialize()`。编辑器模块也成对（`UnrealCSharpEditor.cpp:136,140`）。

**问题**

释放完全依赖"InActive 广播一定发生"这一外部条件，而对象自身的析构不做任何兜底：

- 若模块在未广播 InActive 的情况下被卸载（热重载/异常卸载路径、`GExitPurge` 早退路径 `FCSharpCompilerRunnable.cpp:216-219` 之后的状态错位、或未来新增的加载阶段变化），则 `~FCSharpEnvironment` 只摘委托，注册表整体泄漏：包括 `FClassRegistry` 里每个类的 `FClassDescriptor`、`FClassReflection` 相关描述符、`FBindingRegistry` 的 wrapper、以及**所有注册表持有的 C# GCHandle**（`FObjectRegistry.cpp:24`、`FBindingRegistry.cpp:23` 等 20 余处 `FDomain::GCHandle_Free` 都不会执行）。句柄不释放意味着对应托管对象被永久根引用——这是跨语言泄漏，比 C++ 侧堆对象更严重。
- 与 `FReferenceRegistry`（FGCObject）不同，这些泄漏不会随 UE GC 收敛。

**建议**

```cpp
FCSharpEnvironment::~FCSharpEnvironment()
{
	Deinitialize();          // 幂等（内部全是 if (X != nullptr) 守卫），可安全重入
	if (OnUnrealCSharpModuleInActiveDelegateHandle.IsValid()) { ... Remove ... }
	if (OnUnrealCSharpModuleActiveDelegateHandle.IsValid()) { ... Remove ... }
}
```
安全性论证：`Deinitialize()` 的每一项都是 `if (X != nullptr) { delete X; X = nullptr; }`，二次调用不会崩溃；`FUnrealCSharpModuleDelegates` 的析构顺序问题见 F-ENV-008，二者需一起修（先解 F-ENV-008 的顺序问题，再加这里的兜底）。

**验证方式**

- 在 `Deinitialize()` 每个 `delete` 处加断点/日志，执行"编辑器退出"，确认日志出现；再人为注释掉 `Deactivate()` 调用（仅本地实验）验证泄漏量（`-trace=memory`）。
- `grep -n "Deinitialize()" Source/UnrealCSharp/Private/Registry/*.cpp Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp`：确认 `FCSharpEnvironment::Deinitialize` 只在 `:334` 被调用。

---

### [F-ENV-011] 域重载后仍在飞的延迟任务携带"旧句柄"：句柄空间无代次标记，可命中新域的无关条目

- **类别**: 并发/线程安全 | 未定义行为
- **严重度**: **P2**
- **复核结论**: 部分确认（偏差：**可达性由"潜伏"改为"活跃"** —— 该延迟回调族与后端无关，在五平台 LeanCLR + 编辑器热重载下确实会执行；"句柄空间无代次标记、注册表查找纯值比较"这一核心结论**确认成立**。"旧句柄值会被新域复用"由"未验证"升为"有明确证据、但仍未实测"）
- **可达性**: 活跃（C# 侧**每个** Unreal 类型的终结器都会把句柄排进 GameThread 队列，见 `复核证据`；域重载会先清空并重建注册表）
- **复核证据**: 代码事实 —— `Source/UnrealCSharp/Private/Domain/Interop/FRegisterStruct.cpp:47-53` `AsyncTask(ENamedThreads::GameThread, [InManagedHandle]{ ...RemoveStructReference(InManagedHandle); })`；`Source/UnrealCSharpCore/Public/Domain/Script/IManagedHandle.h:5-30`（`int64 Value{}`，无域标识）；`FBindingRegistry.h:90-92`/`FObjectRegistry.h:47-49` 以 `IManagedHandle` 为键；域重载点 `FCSharpEnvironment.cpp:263-268`（删旧域）与 `:59`（建新域），驱动方 `Source/Compiler/Private/FCSharpCompilerRunnable.cpp:223-229`（GameThread 任务里 `Deactivate → 重载 → Activate`）。**补强的两条证据**：(1) 该延迟族是**系统性**的而非个例 —— 织入器给每个 Unreal 类型注入终结器 `Script/Weavers/UnrealTypeWeaver.cs:123-160`（`GetHandle` → `UStruct_UnRegisterImplementation`），C# 侧实例如 `Script/UE/CoreUObject/TArray.cs:17` `~TArray() => TArray_UnRegisterImplementation(HandleData.GetHandle(this));`；(2) 句柄值**确实会被复用** —— `Script/Interop/Handle/HandleData.cs:13` `private static nint Handle;`、`:32` `new HandleReference { Value = ++Handle, Count = 0 }` 是**单调计数器且 `Clear()`（:129-145）不重置它**，而域卸载走 `Script/Interop/AssemblyLoader/AssemblyLoader.cs:31-48`（`HandleData.Clear()` + `Context.Unload()` + `GC.WaitForPendingFinalizers()`），新域的托管静态量从 0 重新起算 → 旧域前 N 个句柄在新域被逐个复用
- **级别变动**: 无（P2 维持：错误释放会 `GCHandle_Free` + 对错地址 `DestroyStruct`/`FMemory::Free`，属数据损坏级；但需"排队 → 域重载 → 执行"的时序恰好成立）
- **文件**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:147-269`（旧域整体销毁）、`Source/UnrealCSharp/Public/Registry/FBindingRegistry.h:90-92`、`Source/UnrealCSharp/Public/Registry/FObjectRegistry.h:47-49`（以 `IManagedHandle` 为键）
- **函数**: `FCSharpEnvironment::Deinitialize` / `Initialize`（`IManagedHandle` 复用）
- **置信度**: 中（"句柄值会被复用"依赖具体脚本域实现，未在本机验证；"延迟任务捕获裸句柄"是**高**置信度代码事实）

**现状（代码事实）**

句柄是 64 位纯标量，没有域标识：

```cpp
// Source/UnrealCSharpCore/Public/Domain/Script/IManagedHandle.h:5-30
struct IManagedHandle
{
	int64 Value{};
	bool operator==(const IManagedHandle& InOther) const { return Value == InOther.Value; }
	...
};
static constexpr IManagedHandle InvalidManagedHandle{};
```

域重载时旧域被整体销毁、随后重建（`FCSharpEnvironment.cpp:263-268` 删 `Domain`，`:59` 新建），而注册表键仍是原始 `int64`。

延迟任务捕获裸句柄的例子：

```cpp
// Source/UnrealCSharp/Private/Domain/Interop/FRegisterStruct.cpp:47-53
		static void UnRegisterImplementation(const IManagedHandle InManagedHandle)
		{
			AsyncTask(ENamedThreads::GameThread, [InManagedHandle]
			{
				(void)FCSharpEnvironment::GetEnvironment().RemoveStructReference(InManagedHandle);   // 之后才执行
			});
		}
```

**调用上下文**

`FRegisterStruct::UnRegisterImplementation`（C# 的 `UStruct.UnRegister`）把释放动作排到 GameThread 队列；`Deactivate/Activate`（域重载）同样在 GameThread 上执行（`FCSharpCompilerRunnable.cpp:213-242` 用 `ENamedThreads::GameThread` 的 `FFunctionGraphTask` + `WaitUntilTaskCompletes`）。因此存在"调用方排队 → 域重载先执行 → 排队的 lambda 后执行"的次序。

**问题**

1. 旧域销毁后重建新域，`FCSharpEnvironment::Deinitialize` 已把所有注册表清空（`X = nullptr` 后重建），因此排队 lambda 执行时面对的是**新域的注册表**。它携带的 `InManagedHandle` 来自旧域。
2. `IManagedHandle` 只有 `int64` 值，注册表查找是纯值比较（`TMapping::Find` → `TMap::Find`）。若旧值恰好在新区里用于另一个对象/结构体，则 `RemoveStructReference` / `RemoveObjectReference` / `RemoveBindingReference` 会**删除并释放另一个条目**（错误释放：`FDomain::GCHandle_Free` + `FMemory::Free`），造成数据损坏（最坏情况是 `FStructRegistry` 对错误地址调用 `DestroyStruct` + `FMemory::Free`，见 `FStructRegistry.cpp:117-138`）。
3. 这一族延迟回调不止一处：`FRegisterStruct.cpp:49`、`TPropertyValue` 的 return/ref 包装、`FCSharpBind` 的异步绑定（`FCSharpEnvironment.cpp:432-451`）都在跨帧持有句柄。缺乏"域代次"校验意味着任何一处漏掉"域已重载"检查都会静默作用于错误对象。

**建议**

```cpp
// 1) 给句柄加域代次（最小侵入：把 InvalidManagedHandle 之外的高位留给 generation）
struct IManagedHandle { int64 Value{}; /* 高 8 位 = generation，低 56 位 = payload */ };
// 由 FDomain/IScriptDomain 在每次 Initialize 时 ++Generation，查找前校验：
if (InManagedHandle.Generation() != CurrentGeneration) { return false; }   // 拒绝跨域句柄

// 2) 或者更保守：域重载时把 generation 存进 FCSharpEnvironment，
//    所有 registry 的 Find/Remove 入口统一做一次校验（集中在 TMapping 的包装层）。
```
权衡：改 `IManagedHandle` 影响面大（`UnrealCSharpCore` 也用它、`IManagedHandleFromObject`/`ToObject` 做指针转换，见 `IManagedHandle.h:32-40`），因此更实际的做法是在 `FCSharpEnvironment` 加一个 `uint32 DomainGeneration`，并在**延迟回调**处捕获调用时的 generation，执行前比对后放弃（`if (CapturedGeneration != Env.GetDomainGeneration()) return;`）——改动局限在少数几个 `AsyncTask`/延迟 lambda。

**验证方式**

- `grep -n "AsyncTask\|FFunctionGraphTask\|AddRaw(this" Source/UnrealCSharp/Private/Domain/Interop/*.cpp`：列出所有"跨帧执行且捕获句柄"的点。
- 用例：C# 调用 `UStruct.UnRegister(handle)` 后**立即**触发一次 C# 编译（域重载），观察 `RemoveStructReference` 是否作用在新域条目上（加日志打印 handle 与是否命中）。

---

### [F-ENV-012] `GeManagedHandle` 可能返回无效 owner 句柄，而 `AddReference(invalid, new FReference(...))` 仍会分配，并以 `{0}` 为键累积、无法按句柄删除

- **类别**: 内存/资源泄漏 | 未检查返回值
- **严重度**: **P3**
- **复核结论**: 确认（`GeManagedHandle` 在"成员属 UScriptStruct 且该实例未登记"时返回 `InvalidManagedHandle`；`AddReference` 侧无 `IManagedHandleIsValid` 校验；`{0}` 桶无常规删除路径 —— 三点均逐行复核一致）
- **可达性**: 活跃（`FStructPropertyDescriptor::NewRef`、`FArray/FMap/FSet/FDelegate/FMulticastDelegatePropertyDescriptor` 的 `NewRef` 在反射成员访问时都会走到）
- **复核证据**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:573-587`（`:584` 查不到即返回 `IManagedHandle()`）、`:613-616`（无校验转发）；`Source/UnrealCSharp/Private/Registry/FReferenceRegistry.cpp:32-42`（`if (!Contains(InOwner)) Add(InOwner, {}); ReferenceRelationship[InOwner].Emplace(InReference);` 恒返回 true）、`:44-57`（`RemoveReference` 需调用方给 owner）；owner 侧调用者 `FObjectRegistry.cpp:92,110`、`FStructRegistry.cpp:119` 传的都是真实句柄。**`grep "GeManagedHandle"` 全 `Source/` 实测 11 处**，其中 6 处外部调用：`FStructPropertyDescriptor.cpp:63`、`FArrayPropertyDescriptor.cpp:42`、`FMapPropertyDescriptor.cpp:42`、`FSetPropertyDescriptor.cpp:42`、`FDelegatePropertyDescriptor.cpp:44`、`FMulticastDelegatePropertyDescriptor.cpp:56`（与原文一致）
- **级别变动**: 无（P3 维持：小对象泄漏 + 桶只增不减，需"脚本结构体成员未登记"这一路径）
- **文件**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:573-587`（owner 计算）→ `:613-616`（`AddReference`）→ `Source/UnrealCSharp/Private/Registry/FReferenceRegistry.cpp:32-42`
- **函数**: `FCSharpEnvironment::GeManagedHandle(void*, const FProperty*)` / `FCSharpEnvironment::AddReference(const IManagedHandle, FReference*)` / `FReferenceRegistry::AddReference`
- **置信度**: 中（"无效 owner 会实际出现"未实测；"无效句柄不被拒绝"是**高**置信度代码事实，配套 `IManagedHandleIsValid` 工具函数的用法见 §6.1）

**现状（代码事实）**

```cpp
// FCSharpEnvironment.cpp:573-587
IManagedHandle FCSharpEnvironment::GeManagedHandle(void* InAddress, const FProperty* InProperty) const
{
	const auto Owner = static_cast<uint8*>(InAddress) - InProperty->GetOffset_ForInternal();   // :575

	if (InProperty->GetOwnerClass())
	{
		return GeManagedHandle(reinterpret_cast<UObject*>(Owner));       // :579 只查对象表
	}
	else
	{
		return StructRegistry != nullptr
			       ? StructRegistry->GetManagedHandle(InProperty->GetOwner<UScriptStruct>(), Owner)   // :584 查不到 → InvalidManagedHandle
			       : IManagedHandle();
	}
}
```
```cpp
// FCSharpEnvironment.cpp:613-616
bool FCSharpEnvironment::AddReference(const IManagedHandle InOwner, FReference* InReference) const
{
	return ReferenceRegistry != nullptr ? ReferenceRegistry->AddReference(InOwner, InReference) : false;
}
```
```cpp
// FReferenceRegistry.cpp:32-42  不校验 InOwner 有效性
bool FReferenceRegistry::AddReference(const IManagedHandle InOwner, FReference* InReference)
{
	if (!ReferenceRelationship.Contains(InOwner))
	{
		ReferenceRelationship.Add(InOwner, {});
	}
	ReferenceRelationship[InOwner].Emplace(InReference);
	return true;
}
```

**调用上下文**

`GeManagedHandle(address, property)` 的调用点共 6 处，全部形如"取某成员的地址 → 求它的宿主 → 把成员包装对象挂到宿主句柄下"：

- `FStructPropertyDescriptor.cpp:63-67`（`AddStructReference(OwnerManagedHandle, ...)`）
- `FArrayPropertyDescriptor.cpp:42-46`、`FMapPropertyDescriptor.cpp:42`、`FSetPropertyDescriptor.cpp:42`（`AddContainerReference(OwnerManagedHandle, ...)`）
- `FDelegatePropertyDescriptor.cpp:44`、`FMulticastDelegatePropertyDescriptor.cpp:56`

`AddContainerReference`/`AddDelegateReference` 的 owner 重载（`FContainerRegistry.inl:43-53`、`FDelegateRegistry.inl:43-53`）与 `FStructRegistry.cpp:104-105` 都会 `new` 一个 `FReference` 子类并调用本函数的 `AddReference(InOwner, ...)`，把自己的移除动作挂在宿主上（如 `TContainerReference.h:15`）。

**问题**

1. `:584` 在"成员属于 UScriptStruct 且该结构体实例未注册"时返回 `InvalidManagedHandle`（值 0），调用方**没有做 `IManagedHandleIsValid` 校验**，于是 `new FStructReference(...)` / `new TContainerReference(...)` / `new FBindingReference(...)` 照常分配，并以 `{0}` 为键写进 `ReferenceRelationship`。
2. 这类条目**不会被常规路径删除**：`FReferenceRegistry::RemoveReference(const IManagedHandle)`（`:44-57`）需要调用方提供 owner 句柄，而所有调用方的 owner 都是"查到的有效句柄"（`FObjectRegistry.cpp:92,110`、`FStructRegistry.cpp:119` 传的都是真实句柄），永远不会传 `{0}`。因此 `{0}` 桶只增不减，直到 `~FReferenceRegistry`（`:7-20`）才被清空——即每次域重载才回收一次。
3. 同一段落还有一处未检查：`:575` 直接解引用 `InProperty`（无空指针检查），`InProperty->GetOwner<UScriptStruct>()` 也可能为 nullptr 并原样传入 `FStructRegistry::GetManagedHandle`（目前该函数对 nullptr 是安全的：`FStructRegistry.cpp:85-89` 只做 `TMap::Find`）。

**建议**

```cpp
// 在 owner 重载入口统一拒绝无效 owner，避免"孤儿引用"：
bool FCSharpEnvironment::AddReference(const IManagedHandle InOwner, FReference* InReference) const
{
	if (!IManagedHandleIsValid(InOwner))      // 新增
	{
		delete InReference;                   // 由调用方决定语义：这里直接释放更安全
		return false;
	}
	return ReferenceRegistry != nullptr ? ReferenceRegistry->AddReference(InOwner, InReference) : false;
}
// 并给 GeManagedHandle 加前置校验与注释：
if (InProperty == nullptr || InAddress == nullptr) { return InvalidManagedHandle; }
```
权衡：直接 `delete` 会改变语义（调用方可能希望延迟释放），也可改为"接受但只做弱登记"；核心是**不能把 `{0}` 当作真实 owner**——建议至少在 `IManagedHandleIsValid` 为假时 `ensureMsgf` 出来，把这条路径变成可观测的。

**验证方式**

- 在 `FReferenceRegistry::AddReference` 加 `ensureMsgf(IManagedHandleIsValid(InOwner), ...)`，跑"含 USTRUCT 成员（数组/映射/委托/嵌套结构体）的反射读写"用例，观察是否触发。
- `grep -n "GeManagedHandle" Source/`（命中 11 处，6 处为外部调用，见 §4）。

---

### [F-ENV-013] 每次 `Initialize()` 都全量遍历 `TObjectRange<UClass>()` 并对每个类做 `IsA` 线性扫描 / O(N×M) 字符串比较——域重载时阻塞主线程

- **类别**: 性能
- **严重度**: **P2**
- **复核结论**: 确认（三处遍历/O(N×M) 结构与主线程同步等待均为代码事实；`grep -c "GetField(FString::Printf"` 实测 **3 处**：`:167`、`:219`、`:356`，与原文一致）
- **可达性**: 活跃（定义在 `FCSharpEnvironment.cpp:144` 的 `OnCSharpEnvironmentInitialize.Broadcast()`，域重建时 `FCSharpCompilerRunnable.cpp:223-229` 在 GameThread 任务里同步 `WaitUntilTaskCompletes`（`:242`）等待）
- **复核证据**: `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:536-544`（`:538` `TObjectRange<UClass>()`、`:540` `GetDefaultObject(false)`、`:542` `BindClassDefaultObject`）；`:88-103`（`:93` 线性遍历 `GetBindClass()`、`:95` `IsA`）；`:159-182`、`:211-235`、`:282-307` 三处双层循环 + `FString::Printf(TEXT("__%s"))`；委托注册 `FCSharpBind.cpp:30-31` ← 广播 `FCSharpEnvironment.cpp:144`；同类叠加 `FDynamicRegistry.cpp:58-72`
- **级别变动**: 无（P2 维持：编辑器常规工作流上的可感知卡顿，非正确性错误）
- **文件**: `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:536-544`（遍历）、`:78-104`（`IsA` 扫描）、`:106-311`（O(N×M) 字符串比较）
- **函数**: `FCSharpBind::OnCSharpEnvironmentInitialize` / `BindClassDefaultObject` / `BindImplementation(UStruct*)`
- **置信度**: 高（结构与复杂度可直接读出；缺绝对耗时实测）

**现状（代码事实）**

```cpp
// FCSharpBind.cpp:536-544
void FCSharpBind::OnCSharpEnvironmentInitialize()
{
	for (const auto Class : TObjectRange<UClass>())            // :538 进程内所有 UClass
	{
		if (const auto DefaultObject = Class->GetDefaultObject(false))
		{
			BindClassDefaultObject(DefaultObject);             // :542
		}
	}
}
```
```cpp
// FCSharpBind.cpp:88-103  非 C# 类要走"设置表线性扫描 + IsA"
	else
	{
		if (const auto UnrealCSharpSetting = FUnrealCSharpFunctionLibrary::GetMutableDefaultSafe<
			UUnrealCSharpSetting>())
		{
			for (const auto& [Class, bNeedOverrideAttribute] : UnrealCSharpSetting->GetBindClass())   // :93 线性
			{
				if (InObject->IsA(Class))                                                            // :95 IsA
				{
					return bNeedOverrideAttribute ? false : IManagedHandleIsValid(Bind<false>(InObject));
				}
			}
		}
	}
```
```cpp
// FCSharpBind.cpp:159-182 / :211-235 / :282-307  三处 O(N×M) 双层循环 + FString 比较
	for (const auto& [PropertyName, Property] : Properties)      // :159
	{
		for (const auto& Field : Fields)                        // :161
		{
			if (Field == PropertyName)                          // :163  FString 比较
			...
				if (auto FoundField = Class->GetField(FString::Printf(TEXT("__%s"), *Field)))   // :167 每次构造 FString
```

**调用上下文**

`OnCSharpEnvironmentInitialize` 通过 `FUnrealCSharpModuleDelegates::OnCSharpEnvironmentInitialize`（注册于 `FCSharpBind.cpp:30-31`）在 `FCSharpEnvironment::Initialize()` 末尾（`:144`）被广播。域重载时该走法每次编译后都发生：

- 触发点：`FCSharpCompilerRunnable.cpp:223-229`（`Deactivate()` → 重载程序集 → `Activate()`）在 GameThread 任务里同步 `WaitUntilTaskCompletes`（`:242`），即**编辑器会等它跑完**。
- 同类全量遍历还会叠加：`FDynamicRegistry::OnCSharpEnvironmentInitialize` → `FDynamicGeneratorCore::Generator`（`FDynamicRegistry.cpp:58-72`）遍历所有带 `UClass` 特性的类。

**问题**

1. 遍历粒度是"所有 UClass"（引擎 + 项目 + 插件，量级数千），其中绝大多数不是 C# 类。对每个类先 `GetDefaultObject(false)`，再 `CanBind`（`TSet.Contains` + `FReflectionRegistry::Get().GetClass`）→ 对非 C# 类还要遍历 `UUnrealCSharpSetting::GetBindClass()` 并逐个 `IsA`（`IsA` 需要沿类层次上溯）。
2. 只有真正是 C# override 类的才会进入 `BindImplementation`（`:106-311`），那一层是三组 O(N×M) 双层循环：属性 × 字段名（`:159-182`）、函数 × 方法名（`:282-307`，还带 `GetParamCount` 比较）、以及 `Methods.Remove(MethodName)` 的容器修改；每命中一次都会 `FString::Printf` 构造 `"__%s"` 再 `GetField` 查名字（`:167`、`:219`、`:356`）。对一个拥有数百属性/函数的类，这是万级字符串构造与比较。
3. 这套代价发生在**每次 C# 编译后**（这是编辑器的常规工作流），且在主线程同步等待中；表现为"编译后编辑器卡顿"。

**建议**

```cpp
// 1) 缩小遍历范围：只遍历"可能是 C# 类"的类。
//    做法：在 Initialize 之前先让 C# 侧把它的类名集合交给 C++（FReflectionRegistry 已有类注册表），
//    按 UClass 名字做集合查找，命中的才 GetDefaultObject + BindClassDefaultObject。
// 2) 消除 O(N x M)：把 Fields/Properties/Functions/Methods 从"双层循环 + FString 比较"
//    改为按名哈希的单层查找（例如把 __ 前缀字段建成 TMap<FString,int32> 索引，
//    属性/函数也用 Encode 结果直接 TMap::Find）。
//    现有代码已有 TMap<FString, FProperty*> Properties / TMap<FString, UFunction*> Functions，
//    只需把 Fields 也变成可哈希查找的容器，即可把 :159-182 与 :282-307 从 O(N*M) 降为 O(N)。
// 3) "__%s" 前缀的 FString 可以缓存：Field 名 -> FName 的映射只与类相关，可在绑定期一次性构建。
```
权衡：第 1 点需要 C# 侧提供类集合（信息已存在于 `FReflectionRegistry`，主要是接线成本）；第 2 点是纯局部优化，风险低、收益随类规模线性放大，建议优先做。

**验证方式**

- 用 `TRACE_CPUPROFILER_EVENT_SCOPE(FCSBind_OnInitialize)` 包住 `FCSharpBind::OnCSharpEnvironmentInitialize` 与 `BindImplementation`，在编辑器里做一次 C# 编译，看 `UnrealCSharp` 相关 scope 的耗时与调用次数。
- `grep -c "GetField(FString::Printf" Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp`（当前 3 处：`:167`、`:219`、`:356`）。

---

### P3

---

### [F-ENV-014] `GeManagedHandle` 拼写错误（缺 `t`），且与 `FObjectRegistry::GetManagedHandle` 命名不一致

- **类别**: 可优化/可读性
- **严重度**: **P3**
- **复核结论**: 确认（两处声明、两处定义、以及"注册表层命名正确"三处对照均逐行复核一致；`grep "GeManagedHandle"` 11 处与原文一致）
- **可达性**: 活跃（6 个反射描述符的 `NewRef` 都会调用 2 参重载）
- **复核证据**: `Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h:123,125`；`Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:566,573`（另 `:579` 为内部自调用）；命名对照 `Source/UnrealCSharp/Public/Registry/FObjectRegistry.h:37` `GetManagedHandle`、`Source/UnrealCSharp/Public/Registry/FStructRegistry.h:63` `GetManagedHandle`
- **级别变动**: 无（P3 维持：纯可读性/API 一致性问题）
- **文件**: `Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h:123,125`、`Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:566,573`
- **函数**: `FCSharpEnvironment::GeManagedHandle(const UObject*)` / `GeManagedHandle(void*, const FProperty*)`
- **置信度**: 高

**现状（代码事实）**

```cpp
// FCSharpEnvironment.h:123-125
	IManagedHandle GeManagedHandle(const UObject* InObject) const;

	IManagedHandle GeManagedHandle(void* InAddress, const FProperty* InProperty) const;
```
```cpp
// FCSharpEnvironment.cpp:566,573
IManagedHandle FCSharpEnvironment::GeManagedHandle(const UObject* InObject) const
IManagedHandle FCSharpEnvironment::GeManagedHandle(void* InAddress, const FProperty* InProperty) const
```
同类功能在注册表层的命名是正确的：`FObjectRegistry::GetManagedHandle`（`FObjectRegistry.h:37`）、`FStructRegistry::GetManagedHandle`（`FStructRegistry.h:63`）。

**调用上下文**

6 处外部调用全部通过 `FCSharpEnvironment::GetEnvironment().GeManagedHandle(...)`（见 §4 证据），加上 `:579` 的内部自调用，共 11 处命中。

**问题**

`GetEnvironment().GeManagedHandle(...)` 是**公开 API**（类带 `UNREALCSHARP_API`），拼写错误的符号会让外部使用者困惑；改名属破坏性变更，因此只能靠别名过渡。此外两个重载同名但语义差别大（一个按对象取句柄，一个按"成员地址+属性"反推宿主），也没有注释说明。

**建议**

```cpp
// 新增正确拼写并保留旧名转发（一个版本周期后再删）
IManagedHandle GetManagedHandle(const UObject* InObject) const;
UE_DEPRECATED(5.x, "Use GetManagedHandle")
IManagedHandle GeManagedHandle(const UObject* InObject) const { return GetManagedHandle(InObject); }
```
**验证方式**

- `grep -rn "GeManagedHandle" Source/`（11 处，无外部模块引用——见 §6.6，因此改名成本可控）。

---

### [F-ENV-015] `FCSharpEnvironment::Domain` 只被写入与删除，从不读取：成员的唯一作用是"生命周期占位"，易被误读为可用句柄

- **类别**: 可优化/可读性
- **严重度**: **P3**
- **复核结论**: 确认（`FCSharpEnvironment.cpp` 中 `Domain` 成员只有 `:59` 赋值、`:263` 判空、`:265` 删除、`:267` 置空四类出现，确无 `Domain->` 形式的读取；`GetBind()` 有公开访问器而 `Domain` 没有，风格不一致成立）
- **可达性**: 活跃（`Initialize/Deinitialize` 每次域重建都会执行）
- **复核证据**: `Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h:309` `FDomain* Domain;`；`Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:59,263,265,267`；访问器对照 `FCSharpEnvironment.h:304` `class FCSharpBind* GetBind() const;`。`FDomain::Tick` 等由 `FTickableGameObject` 机制驱动（`Source/UnrealCSharp/Private/Domain/FDomain.cpp:46-62`），与成员无关 —— 逐条复核一致
- **级别变动**: 无（P3 维持：可读性）
- **文件**: `Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h:309`、`Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:59,263-268`
- **函数**: `FCSharpEnvironment::Initialize` / `Deinitialize`
- **置信度**: 高

**现状（代码事实）**

```cpp
// FCSharpEnvironment.h:309
	FDomain* Domain;
```
全文检索 `Source/UnrealCSharp/` 下 `\bDomain\b` 的结果中，与该成员相关的只有三处：`:59 Domain = new FDomain();`、`:263 if (Domain != nullptr)`、`:265 delete Domain;` / `:267 Domain = nullptr;`。没有 `Domain->` 形式的调用（`FDomain::Tick` 等由 `FTickableGameObject` 机制自行驱动，其他代码用的是 `FDomain::` 静态调用，与成员无关）。

**调用上下文**

`Initialize()`（`:57`）创建它，`Deinitialize()`（`:147`）销毁它——即"域重载时重建脚本域"的唯一实现手段。

**问题**

成员命名为 `Domain` 并持有指针，读者会预期它被用于转发调用（例如 `Domain->Tick()`），实际它只是 RAII 式的"存在即初始化脚本域"。同时没有注释说明这一点；`GetBind()`（`:304`）有公开访问器而 `Domain` 完全没有访问器，两者风格不一致。

**建议**

```cpp
	// 仅用于承载脚本域的生命周期：构造即 Initialize（加载程序集），析构即销毁。无其它访问者。
	TUniquePtr<FDomain> Domain;
```
权衡：改成 `TUniquePtr` 后 `Deinitialize` 里 `delete Domain; Domain = nullptr;` 可以简化为 `Domain.Reset();`（顺带消除"忘记置空"的可能）。

**验证方式**

- `grep -rn "Domain" Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp`（结果仅 59/263/265/267 四处）。

---

### [F-ENV-016] `GetRegistry<T>()` 的 `if constexpr` 链没有 `else`/`static_assert`：传入未支持的注册表类型时返回类型退化为 `void`，错误只能在调用点以晦涩的编译错误暴露；且函数非 `const`

- **类别**: 可读性 | 可优化
- **严重度**: **P3**
- **复核结论**: 确认（`if constexpr` 链无 `else`、无 `static_assert`；函数未标 `const`；其余访问器均 `const` —— 三点逐行复核一致。补正一处计数：链上实际是 **10 个** `else if constexpr` 分支 + `FOptionalRegistry` 条件分支（共 11 种注册表），原文 §5.2 写"11 种"正确、本条正文写"共 10 个分支"指常规分支，二者不矛盾）
- **可达性**: 活跃（2 个调用点都在运行期 interop 路径上）
- **复核证据**: `Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.inl:315-364`（`:318-357` 10 个分支，`:358-363` 为 `#if UE_F_OPTIONAL_PROPERTY` 下的 `FOptionalRegistry` 分支，`:364` 函数结束无 `else`）；`Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h:301-302` `template <typename T> auto GetRegistry();`（**无 `const`**），对照 `:304` `GetBind() const`、`:66` `GetClassDescriptor(...) const`。`grep "GetRegistry"` 全 `Source/` 实测 4 处：声明 `h:302`、定义 `inl:316`、调用 `FRegisterEnhancedInputComponent.cpp:99`、`FRegisterInputComponent.cpp:314`（与原文"调用点只有 2 处"一致）
- **级别变动**: 无（P3 维持：编译期诊断质量 + `const` 正确性）
- **文件**: `Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h:301-302`、`Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.inl:315-364`
- **函数**: `FCSharpEnvironment::GetRegistry<T>()`
- **置信度**: 高

**现状（代码事实）**

```cpp
// FCSharpEnvironment.inl:315-364（结尾）
template <typename T>
auto FCSharpEnvironment::GetRegistry()
{
	if constexpr (std::is_same_v<T, FDynamicRegistry>) { return DynamicRegistry; }
	else if constexpr (std::is_same_v<T, FClassRegistry>) { return ClassRegistry; }
	...  // 共 10 个分支 + FOptionalRegistry
#if UE_F_OPTIONAL_PROPERTY
	else if constexpr (std::is_same_v<T, FOptionalRegistry>) { return OptionalRegistry; }
#endif
}          // ← 没有 else 分支，也没有 static_assert
```

**调用上下文**

调用点只有 2 处：`FRegisterEnhancedInputComponent.cpp:99`、`FRegisterInputComponent.cpp:314`，两者都把返回值直接传给 `GetBind()->Bind(...)`（即依赖返回值存在）。

**问题**

若传入未列举的类型（例如把 `FOptionalRegistry` 传给关闭了 `UE_F_OPTIONAL_PROPERTY` 的构建，或笔误写成 `FStructRegistrys`），`if constexpr` 全部为假 → 函数体无 `return` → 返回类型推导为 `void`，报错发生在**调用点**（"void 值不能用作实参"），定位成本高。加 `static_assert` 可在定义处给出明确信息。此外函数未标注 `const`（`h:302`），而所有其它访问器（`GetBind()` `:304`、`GetClassDescriptor` `:66` 等）都是 `const`。

**建议**

```cpp
	else
	{
		static_assert(sizeof(T) == 0, "FCSharpEnvironment::GetRegistry<T>: unsupported registry type");
	}
```
并改为 `auto GetRegistry() const;`（内部成员指针不变，语义更准确）。

**验证方式**

- 临时写 `GetEnvironment().GetRegistry<FStructRegistry>()` 观察编译期报错信息是否可读。

---

### [F-ENV-017] `Bind` 重载集陷阱：`Bind(UScriptStruct*)` 会静默走 `Bind(UObject*)`（UStruct 模板需要显式模板实参，无法参与重载决议）

- **类别**: 可读性 | 未定义行为（语义误用）
- **严重度**: **P3**
- **复核结论**: 确认（重载决议结论正确：非类型模板参数 `auto IsNeedOverride` 不可推导，`UScriptStruct*` 实参在 `Bind(UObject*)` 与 `Bind(const UObject*)` 之间按"cv 限定更少者优先"解析到前者；且核对 C# 用法后确认当前语义是**符合预期**的，故维持 P3 而非 Bug。修正 C# 片段的行号与引用文本）
- **可达性**: 活跃（`FRegisterStruct.cpp:19` 每次 C# `StaticStruct()` 都走这条）
- **复核证据**: `Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h:50-63`（`:51-52` 模板非类型参数、`:54`/`:56` `UObject*`/`const UObject*`、`:58` `UClass*`、`:60-61` 模板、`:63` handle+FName 重载）；调用点 `Source/UnrealCSharp/Private/Domain/Interop/FRegisterStruct.cpp:17,19`；语义核对面 `Source/UnrealCSharp/Public/Registry/FCSharpBind.inl:56-87`（`:84` `AddObjectReference(FoundClass, InObject, NewObject)` 登记的是**真正的 UScriptStruct 对象**）；显式模板实参用法对照 `TPropertyValue.inl:262,278,494`、`FStructPropertyDescriptor.cpp:7`；C# 侧 `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:158-160`（`:158` 生成 `static UScriptStruct StaticStruct()`，`:160` `return StaticStructSingleton ??= UStructImplementation.UStruct_StaticStructImplementation("{fullPath}");` —— 原文写作 `:158-159` 并把两行混引成一行，已修正）
- **级别变动**: 无（P3 维持：可读性 + 误用风险，当前用法正确）
- **文件**: `Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h:50-63`
- **函数**: `FCSharpEnvironment::Bind` 重载集
- **置信度**: 高（重载决议规则确定；实际语义经 `FRegisterStruct.cpp:13-20` + C# `StaticStruct()` 的用法核对后**是符合预期的**，因此定级 P3 而非 Bug）

**现状（代码事实）**

```cpp
// FCSharpEnvironment.h:50-63
	template <auto IsNeedOverride>
	auto Bind(UStruct* InStruct) const;          // :51-52 ← 非类型模板参数不可推导，必须写 Bind<...>

	IManagedHandle Bind(UObject* Object) const;  // :54
	IManagedHandle Bind(const UObject* Object) const;
	IManagedHandle Bind(UClass* Class) const;    // :58
	template <auto IsNeedOverride>
	auto Bind(UObject* Object) const;            // :60-61

	bool Bind(const IManagedHandle InManagedObject, const FName& InStructName) const;   // :63
```

**调用上下文**

```cpp
// Source/UnrealCSharp/Private/Domain/Interop/FRegisterStruct.cpp:13-20
		static IManagedHandle StaticStructImplementation(const char* InStructName)
		{
			const auto StructName = ...;
			const auto InStruct = LoadObject<UScriptStruct>(nullptr, *StructName);   // :17 UScriptStruct*
			return FCSharpEnvironment::GetEnvironment().Bind(InStruct);              // :19 未写模板实参
		}
```
C# 侧对应实现把返回值当 `UScriptStruct` 使用并缓存：

```csharp
// Script/SourceGenerator/UnrealTypeSourceGenerator.cs:158-160
	public{newBody} static UScriptStruct StaticStruct()          // :158
	{                                                            // :159
	    return StaticStructSingleton ??= UStructImplementation.UStruct_StaticStructImplementation("{fullPath}");   // :160
```
核对结论：`Bind(UScriptStruct*)` 走 `Bind(UObject*)` → `FCSharpBind::BindImplementation<false>(UObject*)`（`FCSharpBind.inl:56-87`）：`Class = UScriptStruct::StaticClass()`，绑定的是 `UScriptStruct` 这个 **UClass** 的描述符，但 `AddObjectReference(FoundClass, InObject, NewObject)`（`inl:84`）登记的 `InObject` 是**真正的那个 UScriptStruct 对象**——正是 C# 需要的语义。因此当前行为正确。

**问题**

正确性依赖"读者知道模板实参不可推导"这一 C++ 细节：`Bind(InStruct)` 与 `Bind<false>(InStruct)` 语义完全不同（后者绑定该结构体自身的描述符，前者按对象绑定其 Class 的描述符）。两者在源码中同时存在（前者见 `FRegisterStruct.cpp:19`，后者见 `TPropertyValue.inl:492`、`FStructPropertyDescriptor.cpp:7`），极易误用，且没有任何注释区分。

**建议**

- 在 `h:51` 与 `:60` 两处模板上方加注释，明确"`Bind<...>(UStruct*)` 必须显式模板实参；`Bind(UStruct*)` 会解析到 `Bind(UObject*)`"；
- 或把模板改名（如 `BindStruct<IsNeedOverride>(UStruct*)`）彻底消除重载歧义；
- 至少在 `FRegisterStruct.cpp:19` 补一句注释说明"此处故意按 UObject 绑定以便返回 UScriptStruct 句柄"。

**验证方式**

- 静态检查：`grep -rn "\.Bind(" Source/` 逐个确认是否有把 `UScriptStruct*` 传给 `Bind` 却期望"绑定结构体描述符"的地方（当前仅 `FRegisterStruct.cpp:19`，语义已核对）。

---

### [F-ENV-018] `FCSharpBind::Bind(UClass*)` 丢弃了 class descriptor 绑定结果，失败会被静默吞掉

- **类别**: 未检查返回值
- **严重度**: **P3**
- **复核结论**: 确认（`:55` 丢弃 `Bind<false>(InClass)` 的 bool 返回值、`:57` 仍继续走对象路径 —— 代码事实与调用点全部复核一致）
- **可达性**: 活跃（`FCSharpEnvironment::Bind(UClass*)` → 本函数）
- **复核证据**: `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:53-58`；转发层 `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:464-467`；调用点实测 `Source/UnrealCSharp/Private/Domain/Interop/FRegisterUnreal.cpp:132`、`FRegisterObject.cpp:35,44`、`Source/UnrealCSharp/Private/Reflection/Property/ObjectProperty/FClassPropertyDescriptor.cpp:8,15`（与原文一致）；失败分支 `FCSharpBind.cpp:124-136`（`NewClassDescriptor == nullptr` → 先 `RemoveClassDescriptor` 再 `return false`）
- **级别变动**: 无（P3 维持：静默吞错，无已知现网触发）
- **文件**: `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:53-58`
- **函数**: `FCSharpBind::Bind(UClass*)`
- **置信度**: 高

**现状（代码事实）**

```cpp
// FCSharpBind.cpp:53-58
IManagedHandle FCSharpBind::Bind(UClass* InClass)
{
	Bind<false>(InClass);                                  // :55 返回值（bool：是否成功建立 class descriptor）被丢弃

	return Bind(static_cast<UObject*>(InClass));           // :57
}
```

**调用上下文**

`FCSharpEnvironment::Bind(UClass*)`（`FCSharpEnvironment.cpp:464-467`）→ 本函数；外部调用点：`FRegisterUnreal.cpp:132`、`FRegisterObject.cpp:35,44`、`FClassPropertyDescriptor.cpp:8,15`。

**问题**

`:55` 的 `Bind<false>(InClass)` 是"把 `UClass` 自身的 `FClassDescriptor` 建好"这一步；若失败（例如 `AddClassDescriptor` 返回后 `NewClassDescriptor->GetClass()` 为空，见 `FCSharpBind.cpp:124-136`），`:57` 的 `Bind(UObject*)` 仍会尝试走对象路径，最终可能返回一个"有对象句柄但类描述符缺失"的不一致状态，且调用方无法区分。`FCSharpEnvironment::Bind` 又把这个 `IManagedHandle` 直接返回给 C#。

**建议**

```cpp
	if (!Bind<false>(InClass))
	{
		ensureMsgf(false, TEXT("Bind(UClass*) failed for %s"), *GetNameSafe(InClass));
		return InvalidManagedHandle;
	}
```
**验证方式**

- 在 `BindImplementation(UStruct*)`（`FCSharpBind.cpp:106`）的 `return false` 分支加日志，跑完整测试集确认是否可达；可达则说明当前存在被吞掉的失败。

---

### [F-ENV-019] `FCSharpEnvironment::OnUObjectArrayShutdown()` 是空实现，却在 `GUObjectArray` 关停时被调用

- **类别**: 死代码 | 可读性
- **严重度**: **P3**
- **复核结论**: 确认（空实现、唯一调用者、`FUObjectListener` 的继承与重写关系均逐行复核一致。把原文的"⚠️部分"改为"确认" —— 原文未给出任何偏差说明，而事实层面无偏差；"此处是最后一次安全清理点"属建议性判断，不影响事实成立）
- **可达性**: 活跃（`GUObjectArray` 关停时由 `FUObjectListener::OnUObjectArrayShutdown` 回调）
- **复核证据**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:323-325`（函数体为空）；`Source/UnrealCSharp/Private/Listener/FUObjectListener.cpp:51-54`（唯一调用者 `:53`）；`Source/UnrealCSharp/Public/Listener/FUObjectListener.h:20` `virtual void OnUObjectArrayShutdown() override;`；监听器宿主 `Source/UnrealCSharp/Public/UnrealCSharp.h:27` `FUObjectListener UObjectListener;`；`grep "OnUObjectArrayShutdown"` 全 `Source/` 实测 **5 处**（`FUObjectListener.h:20`、`FCSharpEnvironment.h:36`、`FCSharpEnvironment.cpp:323`、`FUObjectListener.cpp:51,53`），与原文一致，**无其它实现者**
- **级别变动**: 无（P3 维持：缺失防御/可读性，非现网错误）
- **文件**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:323-325`
- **函数**: `FCSharpEnvironment::OnUObjectArrayShutdown()`
- **置信度**: 高

**现状（代码事实）**

```cpp
// FCSharpEnvironment.cpp:323-325
void FCSharpEnvironment::OnUObjectArrayShutdown()
{
}
```
```cpp
// Source/UnrealCSharp/Private/Listener/FUObjectListener.cpp:51-54
void FUObjectListener::OnUObjectArrayShutdown()
{
	FCSharpEnvironment::GetEnvironment().OnUObjectArrayShutdown();
}
```

**调用上下文**

`FUObjectListener` 是 `FUnrealCSharpModule` 的成员（`UnrealCSharp.h:27 FUObjectListener UObjectListener;`），它继承 `FUObjectArray::FUObjectCreateListener/FUObjectDeleteListener`（`FUObjectListener.h:3`）并重写 `OnUObjectArrayShutdown`（`FUObjectListener.h:20`）。因此当 `GUObjectArray` 关闭（引擎退出/`UObjectBaseUtility` 级别清理）时，会走到这个空函数。

**问题**

空钩子本身是**有意义的占位**（"关闭时不做任何事"），但在 `GUObjectArray` 关停时**所有 UObject 的弱指针都已失效**，而环境仍持有 `AsyncLoadingObjectArray`（`h:327`）与各注册表中的句柄/指针。此处是"最后一次安全清理点"（例如释放句柄、清空 `AsyncLoadingObjectArray`、标记 `bShuttingDown` 让后续查询直接返回 null），不做任何事的代价是：关停序列中若还有代码触碰环境，会读到已失效的弱指针（当前 `FObjectRegistry::GetAddress` 用 `Get()` 是安全的，但 `FObjectRegistry::GetAddress(handle, UStruct*&)` 在 `FObjectRegistry.cpp:45` 用的是 `(*FoundObject)->GetClass()`——见 §6.1）。
函数体为空且无注释说明"为什么可以什么都不做"，读者无法判断是遗漏还是有意为之。

**建议**

```cpp
void FCSharpEnvironment::OnUObjectArrayShutdown()
{
	// GUObjectArray 已关停：所有弱指针失效，这里只做"拒绝后续访问"的标记，避免关停序列访问悬垂数据。
	{
		FScopeLock Lock(&CriticalSection);
		AsyncLoadingObjectArray.Empty();
	}
	bObjectArrayShutdown = true;
}
```
并在 `GetObject/GetAddress/GetBinding` 的入口检查该标志快速返回 `InvalidManagedHandle`。

**验证方式**

- `grep -rn "OnUObjectArrayShutdown" Source/`（命中 5 处，见 §4），确认没有其它实现者。
- 在引擎退出路径打断点确认调用时序，以及此时是否仍有环境查询发生。

---

### [F-ENV-020] `DuplicateFunction` 的 `AddToRoot()` 依赖**另一个文件 + C# 侧**才能回收，配对关系无断言、无注释（跨语言不变量）

- **类别**: 可读性 | 内存/资源泄漏（潜在）
- **严重度**: **P3**
- **复核结论**: 确认（`AddToRoot` 在 `FCSharpBind.cpp` 内无配对、配对点分处 `~FCSharpFunctionRegister` 与 C# `UClass.RemoveFunction` 两处 —— 全部逐行复核一致；C# 调用点的 6+1 行号与原文完全吻合）
- **可达性**: 活跃（`DuplicateFunction` 在 override 类绑定期执行；C# 侧 `RemoveFunction` 在解除输入绑定时执行）
- **复核证据**: `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:528-531`（`:530` `NewFunction->AddToRoot()`）；配对点 1 `Source/UnrealCSharp/Private/Reflection/Function/FCSharpFunctionRegister.cpp:29-71`（`:52-56` 从类函数表移除、`:59-62` `RemoveFromRoot()`、`:63-66` `MarkAsGarbage()`）；配对点 2 `Source/UnrealCSharp/Private/Domain/Interop/FRegisterClass.cpp:12-33`（`:23` `RemoveFromRoot`、`:27` `MarkAsGarbage`、`:30` `RemoveFunctionFromFunctionMap`）← C# `Script/UE/CoreUObject/Class.cs:8-9` ← 调用点 `Script/UE/CoreUObject/InputComponent.cs:257,281,305,330,369,393`、`Script/UE/CoreUObject/EnhancedInputComponent.cs:81`（与原文完全一致）；interop 侧 `AddToRoot` 在 `FRegisterInputComponent.cpp:312`、`FRegisterEnhancedInputComponent.cpp:97`
- **级别变动**: 无（P3 维持：跨文件/跨语言不变量缺注释与可观测性，非已证实的泄漏）
- **文件**: `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:526-531`
- **函数**: `FCSharpBind::DuplicateFunction`
- **置信度**: 中（配对确实存在，见"调用上下文"；"无断言"是事实，是否会造成实际泄漏取决于 C# 是否始终解绑）

**现状（代码事实）**

```cpp
// FCSharpBind.cpp:524-533
	InClass->AddFunctionToFunctionMap(NewFunction, InFunctionName);

	NewFunction->ClearInternalFlags(EInternalObjectFlags::Native);

	if (InClass->HasAnyInternalFlags(EInternalObjectFlags::RootSet) || GUObjectArray.IsDisregardForGC(InClass))
	{
		NewFunction->AddToRoot();          // :530 ← 本文件内没有任何 RemoveFromRoot
	}

	return NewFunction;
```

**调用上下文**（配对点确实存在，但都在别处）

1. `~FCSharpFunctionRegister`（`Source/UnrealCSharp/Private/Reflection/Function/FCSharpFunctionRegister.cpp:52-67`）会对 `FunctionRemove` 做 `RemoveFromRoot()`（`:61`）或 `MarkAsGarbage()`（`:65`）并从类函数表移除（`:56`）。该对象的生命周期挂在 `FClassRegistry` 的容器上（`FClassRegistry.cpp:62,143-157,196,205`）。
2. 另有一批 `AddToRoot` 在 interop 里创建的函数（`FRegisterInputComponent.cpp:312`、`FRegisterEnhancedInputComponent.cpp:97`），其回收依赖 **C# 主动调用** `UClass.RemoveFunction`（`FRegisterClass.cpp:12-33`），C# 侧调用点：`Script/UE/CoreUObject/InputComponent.cs:257,281,305,330,369,393`、`EnhancedInputComponent.cs:81`。

**问题**

`AddToRoot`（把对象钉进 RootSet，永不回收）与 `RemoveFromRoot` 分处三个文件 + C# 代码，且 `FCSharpBind.cpp` 内没有任何注释或断言说明这一点。任何一处 C# 忘记 `RemoveFunction`（或异常路径提前返回）都会留下永久 root 的 `UFunction`（附带其全部 `FProperty` 与参数元数据）。这种"跨语言配对"是典型的高风险不变量。

**建议**

- 在 `:530` 加注释明确写出回收责任方（`~FCSharpFunctionRegister` 与 C# `UClass.RemoveFunction`）；
- 增加可观测性：用一个 static 计数器记录本插件 `AddToRoot` 的净数量，在 `Deinitialize` 时 `UE_LOG` 输出非零值；
- 若可行，改为 `TStrongObjectPtr` 之类的显式所有权容器（放在 `FClassRegistry` 内），避免依赖 RootSet。

**验证方式**

- `grep "AddToRoot|RemoveFromRoot"`（范围 `Source/UnrealCSharp`，**实测按行计：含 `AddToRoot` 的行 8 行、含 `RemoveFromRoot` 的行 6 行**；原文写的"AddToRoot 13 处 / RemoveFromRoot 10 处"与实测不符，已修正）。其中**真正的 `UObject::AddToRoot()` 调用**为 6 处：`FCSharpBind.cpp:530`、`FRegisterInputComponent.cpp:312`、`FRegisterEnhancedInputComponent.cpp:97`、`FMulticastDelegateHelper.cpp:25`、`FDelegateHelper.cpp:24`、`FRegisterObject.cpp:89`（最后一条是 C# 可调的 `UObject.AddToRoot` interop API）；`RemoveFromRoot()` 为 5 处：`FCSharpFunctionRegister.cpp:61`、`FRegisterClass.cpp:23`、`FMulticastDelegateHelper.cpp:39`、`FDelegateHelper.cpp:38`、`FRegisterObject.cpp:97`。逐条核对配对后，本报告已确认的例外只有"interop 动态函数 + C# `RemoveFunction`"这一族。
- 运行时：在 `Deinitialize` 打印上述计数器。

---

### [F-ENV-021] `Deinitialize` 中 13 段逐字重复的 `delete` 块、`TMap` 键就地修改、以及 `Initialize/Deinitialize` 的不对称维护成本

- **类别**: 可优化/可读性
- **严重度**: **P3**
- **复核结论**: 部分确认（偏差：**"12 段逐字重复的 delete 块"实际是 13 段** —— `Deinitialize` 的 `if (X != nullptr) { delete X; X = nullptr; }` 形态共 13 处（含 `Domain`，`UE_F_OPTIONAL_PROPERTY=0` 时为 12 处），原文少计 1。其余两点（`TMap` 键就地修改、`Initialize/Deinitialize` 顺序约束无注释）**确认成立**）
- **可达性**: 活跃（`Deinitialize` 每次域重载都执行；`FBindingRegistry::Deinitialize`/`FObjectRegistry::Deinitialize` 由析构驱动）
- **复核证据**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:177-184`(`OptionalRegistry`)、`:186-191`(`BindingRegistry`)、`:193-198`(`StringRegistry`)、`:200-205`(`MultiRegistry`)、`:207-212`(`DelegateRegistry`)、`:214-219`(`ContainerRegistry`)、`:221-226`(`StructRegistry`)、`:228-233`(`ObjectRegistry`)、`:235-240`(`ReferenceRegistry`)、`:242-247`(`ClassRegistry`)、`:249-254`(`CSharpBind`)、`:256-261`(`DynamicRegistry`)、`:263-268`(`Domain`) = **13 段**；键就地修改 `Source/UnrealCSharp/Private/Registry/FBindingRegistry.cpp:21-33`（`:25` `Key = IManagedHandle{};`，`:35` 紧接 `Empty()`）与 `Source/UnrealCSharp/Private/Registry/FObjectRegistry.cpp:22-27`（`:26` 同形，`:29` 紧接 `Empty()`）；同族拷贝粘贴另见 `FStringRegistry.cpp:22,40,59,79,98`、`FMultiRegistry.cpp:22,40,58,76,94,112`、`FContainerRegistry.cpp:29,47,65`、`FDelegateRegistry.cpp:29,47` —— 全部行号逐行复核一致
- **级别变动**: 无（P3 维持：可维护性/一致性；顺序约束当前经复核是**正确**的）
- **文件**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:177-268`（释放）、`:323-325` 无关；`Source/UnrealCSharp/Private/Registry/FBindingRegistry.cpp:21-33`、`Source/UnrealCSharp/Private/Registry/FObjectRegistry.cpp:22-27`（键就地修改）
- **函数**: `FCSharpEnvironment::Deinitialize` / `FBindingRegistry::Deinitialize` / `FObjectRegistry::Deinitialize`
- **置信度**: 高

**现状（代码事实）**

```cpp
// FCSharpEnvironment.cpp:177-268 的形态：13 段逐字重复
	if (OptionalRegistry != nullptr) { delete OptionalRegistry; OptionalRegistry = nullptr; }
	if (BindingRegistry != nullptr)  { delete BindingRegistry;  BindingRegistry = nullptr; }
	if (StringRegistry != nullptr)   { delete StringRegistry;   StringRegistry = nullptr; }
	... （共 13 段，含 Domain 与 CSharpBind；`UE_F_OPTIONAL_PROPERTY=0` 时 12 段）
```
```cpp
// FBindingRegistry.cpp:21-33（FObjectRegistry.cpp:22-27 同形）
	for (auto& [Key, Value] : ManagedHandle2BindingAddress.Get())
	{
		FDomain::GCHandle_Free(Key);

		Key = IManagedHandle{};      // :25 ← 就地修改 TMap 的键（会破坏哈希不变量）
		...
	}
	ManagedHandle2BindingAddress.Empty();   // :35 ← 紧跟其后清空，因此当前无害
```

**问题**

1. 12 段重复代码没有价值（可用一个 `DeleteAndNull(ptr)` 模板或 `TArray<void*>` 循环），且 `Initialize`（`:57-85`）与 `Deinitialize` 必须手工保持**同一顺序的逆序**——顺序本身有语义（`BindingRegistry` 必须早于 `ReferenceRegistry` 释放，否则 `FBindingReference::~` 会访问已释放对象；`Domain` 必须最后释放，因为所有注册表在 `Deinitialize` 里要调 `FDomain::GCHandle_Free`）。这个顺序约束没有任何注释，改动极易破坏（见 §2 调用链 3、4）。
2. `Key = IManagedHandle{}` 在遍历中就地改写 `TMap` 的键：虽然紧接着 `Empty()` 使当前实现无害，但这种写法一旦被人复制到"不 Empty（例如只清一部分）"的场景就会破坏 TMap 的查找不变量。同时这几个注册表的 `Deinitialize` 都是同一段拷贝粘贴的产物（`FStringRegistry.cpp:22,40,59,79,98`、`FMultiRegistry.cpp:22,40,58,76,94,112`、`FContainerRegistry.cpp:29,47,65`、`FDelegateRegistry.cpp:29,47`），应当抽成一个 `TManagedHandleMapping` 上的模板函数。

**建议**

```cpp
// 1) 抽公共函数，并把"顺序约束"写成注释
template <typename T> static void DeleteAndNull(T*& Ptr) { delete Ptr; Ptr = nullptr; }
// 注意顺序：先把所有"转发给其它注册表"的注册表释放（BindingRegistry/String/Multi/Delegate/Container/
// Struct/Object/Reference），再释放 ClassRegistry（其 FClassDescriptor 析构会回调环境），
// 最后释放 Domain（其余注册表在析构中要调 FDomain::GCHandle_Free）。
DeleteAndNull(OptionalRegistry); ... DeleteAndNull(Domain);

// 2) 键代次清零改为"构造新容器 + Swap"，避免就地改键：
auto Old = MoveTemp(ManagedHandle2BindingAddress);   // 遍历 Old（此时已不被任何查找使用）
for (auto& [Key, Value] : Old.Get()) { ... }
```
**验证方式**

- 用 `-Wduplicate-branches`/`clang-tidy` 之类的静态检查提示重复块；更实际的做法是在 `FBindingRegistry::Deinitialize` 与 `FOptionRegistry::Deinitialize` 之间做一次 diff，确认只有类型名不同。

---

## 4. 死代码清单

检索范围：`Source/` 下 `*.h,*.cpp,*.inl`，**实测 507 个文件**（排除 `\ThirdParty\`；含 ThirdParty 为 1027，ThirdParty 自身 520）。原文写"543 个文件"为误记，已修正。
**复核方式**：下表"命中数"已全部用 `grep` 工具**重跑**（按**命中行数**计，与原文的"`Select-String -AllMatches` 按出现次数计"口径不同），凡与原文不符处均已在"命中数"列注明。

| 符号 | 声明位置 | grep 模式 | 命中数 | 判定 | 证据 |
|---|---|---|---|---|---|
| `FCSharpEnvironment::GetClassDescriptor(const FName&)` | `FCSharpEnvironment.h:68` / 定义 `FCSharpEnvironment.cpp:479-482` | `\bGetClassDescriptor\b` | **22**（实测；原文写 16）。其中 FName 重载出现 **4 处**：声明 `FCSharpEnvironment.h:68`、`FClassRegistry.h:24`，定义 `FCSharpEnvironment.cpp:479`、`FClassRegistry.cpp:90`（原文写"FName 重载仅 2 处"偏低）；其余是 `UStruct*` 重载、`IScriptDomain::GetClassDescriptor`/`UtilsGetClassDescriptor` 等无关同名符号 | **死代码（强）**，且是 public API → 需确认是否对外承诺 | 唯一转发目标 `FClassRegistry::GetClassDescriptor(const FName&)`（`FClassRegistry.cpp:90-95`）的唯一调用者就是本函数（`FCSharpEnvironment.cpp:481`），即整条 FName→`LoadObject<UStruct>` 路径无调用者 |
| `FCSharpEnvironment::RemoveObjectReference(const IManagedHandle)` | `FCSharpEnvironment.h:107` / 定义 `FCSharpEnvironment.cpp:543-546` | `\bRemoveObjectReference\b` | 5（`cpp:312` 调用的是 `UObject*` 重载、`cpp:538` `UObject*` 定义、`h:105` 声明） | **死代码（强）**，public API | 无任何调用点；`UObject*` 重载仅被 `cpp:312` 内部调用 |
| `FCSharpEnvironment::GetAddress<T>(const IManagedHandle, UStruct*&)`（含 `inl:82` / `inl:97` 两个显式特化） | `FCSharpEnvironment.h:92-93` / 定义 `FCSharpEnvironment.inl:81-109` | `GetAddress<\s*UObject\s*>\s*\(`、`GetAddress<\s*UScriptStruct\s*>\s*\(` | 2（仅 `inl:82`、`inl:97` 两个定义） | **死代码（强）**，public API | 全部外部调用都是"单模板参数"形式 `GetAddress<UObject, void*>(handle)`（`FRegisterProperty.cpp:13,29,43,58`）与 `GetAddress<UScriptStruct, FTransform>(handle)`（`FRegisterWorld.cpp:129`），都不带 `UStruct*&` 出参 |
| `FObjectRegistry::GetAddress(const IManagedHandle, UStruct*&)` | `FObjectRegistry.h:31` / 定义 `FObjectRegistry.cpp:41-51` | `\bGetAddress\b` | **36**（实测；原文写 42）。相关点：`FObjectRegistry.cpp:41` 定义、`FObjectRegistry.h:31` 声明、`:34` 与 `:62` 是同名兄弟重载/内部转发；调用者只有上一条死代码 | 随上一条一起成为**死代码**（间接） | 唯一调用者是 `FCSharpEnvironment.inl:87` |
| `FCSharpEnvironment::OnUObjectArrayShutdown()` | `FCSharpEnvironment.h:36` / 定义 `FCSharpEnvironment.cpp:323-325` | `\bOnUObjectArrayShutdown\b` | 5（`FUObjectListener.cpp:51` 覆写、`:53` 调用、`FUObjectListener.h:20` 声明、本节符号声明+定义） | **空实现**（有唯一调用者，但函数体为空） | 见 F-ENV-019 |
| `FCSharpEnvironment::GeManagedHandle(const UObject*)`（1 参重载） | `FCSharpEnvironment.h:123` / 定义 `FCSharpEnvironment.cpp:566-571` | `\bGeManagedHandle\b` | 11（6 处外部调用全部是 2 参重载；1 参重载的调用者只有同文件 `:579` 的内部自调用） | 仅内部使用（外部无调用点） | 见 F-ENV-014 |
| `FBindingRegistry::Initialize()` | `FBindingRegistry.h:69` / 定义 `FBindingRegistry.cpp:15-17` | `\bInitialize\b`（`FBindingRegistry` 上下文） | 3（声明、定义、构造函数 `:7` 调用） | **空实现**（仅被构造函数调用，无副作用） | 与 `Deinitialize`（非空）不对称；`FObjectRegistry::Initialize`（`FObjectRegistry.cpp:16-18`）、`FStructRegistry::Initialize`（`FStructRegistry.cpp:17-19`）同样为空 |
| `FCSharpEnvironment::Initialize()` / `Deinitialize()` | `FCSharpEnvironment.h:24,26` | `GetEnvironment\(\)\.Initialize` / `GetEnvironment\(\)\.Deinitialize` | 0（两者都只被自身的委托回调调用：`cpp:329`、`cpp:334`） | **可能是外部 API，需确认**（类带 `UNREALCSHARP_API`，但整个 `Source/` 无外部调用，见 §6.6） | 若确认无外部使用者，应改为 private 或加 `ensureMsgf` 防误用（见 F-ENV-009） |

**已核实"非死代码"的高风险候选**（避免误报，列出反证）：

| 符号 | 证据 |
|---|---|
| `FCSharpEnvironment::AddReference(UObject*)` / `RemoveReference(UObject*)` | 调用点 `FRegisterObject.cpp:115`、`:125` |
| `FCSharpEnvironment::GetOrAddPropertyDescriptor` | 4 个调用点：`FRegisterProperty.cpp:17,33,47,62` |
| `FCSharpEnvironment::GetOrAddFunctionDescriptor<T>` | 26 个调用点（`FRegisterFunction.cpp` 多处 + `CSharpFunction.cpp:7`） |
| `FCSharpEnvironment::AddStructReference<IsNeedFree>` / owner 重载 | **22 处命中**（实测；原文写 20）：声明/定义 `FCSharpEnvironment.h:110,113`、`FCSharpEnvironment.inl:118`、`FCSharpEnvironment.cpp:548`；调用 `TPropertyValue.inl:264,282,287,317,337,343,494,512,517,547,566,571`（**12 处**，原文写 11 处）、`TConstructorHelper.inl:30,37`、`FStructPropertyDescriptor.cpp:14,23,66`、`FCSharpBind.cpp:414` |
| `FCSharpEnvironment::GetRegistry<T>()` / `GetBind()` | `GetRegistry` 全 `Source/` 实测 **4 处**：声明 `h:302`、定义 `inl:316`、调用 2 处 `FRegisterEnhancedInputComponent.cpp:99`、`FRegisterInputComponent.cpp:314`；`GetBind()` 调用点为同 2 处 |
| `FCSharpEnvironment::Bind(const IManagedHandle, const FName&)` | `FRegisterStruct.cpp:26` |
| `FCSharpEnvironment::TGetObject<T,Enable>` | 调用点 8 处（另 5 处在 `FCSharpEnvironment.h:240,245,256,267,278` 为模板声明）：`TFunctionHelper.inl:39`、`TSubscriptHelper.inl:14,27`、`TPropertyBuilder.inl:44,60,92,111,309` |
| `FBindingRegistry::GetBinding<T>` / `RemoveReference` | `FRegisterEnhancedInputComponent.cpp:210`、`FRegisterWorld.cpp:35,46,55,66,75,87,132`、`TPropertyValue.inl:158,196,205,243`、`FBindingReference.h:14`、`TDestructorHelper.inl:18` |
| `FCSharpBind` 的全部 private 静态（`GetOriginalFunction`/`IsCallCSharpFunction`/`RegisterCallCSharpNativeFunction`/`DuplicateFunction`/`CanBind`） | 均在 `FCSharpBind.cpp` 内部互相调用，**每个符号 2-3 处**（实测：`GetOriginalFunction` 定义 `:434` + 调用 `:326`；`IsCallCSharpFunction` 定义 `:466` + 调用 `:441,452`；`RegisterCallCSharpNativeFunction` 定义 `:471` + 调用 `:370,388`；`DuplicateFunction` 定义 `:490` + 调用 `:339,381`；`CanBind` 定义 `:419` + 调用 `:80` 与 `FCSharpBind.inl:18`）。原文写"4-13 处命中"偏高，已修正 —— 非死代码的结论不变 |

---

## 5. 逐函数清单

> 说明：`FCSharpEnvironment.inl` 中 20 余个注册表转发模板形态完全一致（`Registry != nullptr ? Registry->Xxx(...) : 默认值`），按模式分组列出，每个函数名与其行号均已列出，覆盖全部函数。

### 5.1 `FCSharpEnvironment.cpp`

| 函数（行） | 做了什么 | 调用方 | 结论 |
|---|---|---|---|
| `SignalHandler(int32)` (:22-31) | 打印 C# traceback 并 flush 日志；Mac 下恢复旧 action | 引擎信号机制（`signal`/`sigaction` 安装于 `:120-142`） | **问题**：见 F-ENV-006（不终止 + 非信号安全 + `operator[]` 插入风险） |
| 静态 `Environment` (:33) | 进程级单例对象 | `GetEnvironment()` | **问题**：见 F-ENV-008（静态初始化顺序） |
| `FCSharpEnvironment()` (:35-42) | 注册 ModuleActive/InActive 两个委托 | 静态初始化 | 无问题；但与 F-ENV-008 相关 |
| `~FCSharpEnvironment()` (:44-55) | 只摘掉两个委托 | 静态析构 | **问题**：见 F-ENV-010（不调用 `Deinitialize`） |
| `Initialize()` (:57-145) | new **13 个**对象（11 个注册表 + `FCSharpBind` + `FDomain`，`UE_F_OPTIONAL_PROPERTY=0` 时 12 个）+ 注册 async flush 委托 + 编辑器委托 + 安装信号 + 广播 `OnCSharpEnvironmentInitialize` | `OnUnrealCSharpModuleActive` (:329) | **问题**：见 F-ENV-009（非幂等）；无 `check(IsInGameThread())` |
| `Deinitialize()` (:147-269) | 清 2 个数组、摘 2 个编辑器委托、摘 async flush 委托、逆序 delete **13 个**对象（同上口径） | `OnUnrealCSharpModuleInActive` (:334) | **问题**：见 F-ENV-009（`OnAsyncLoadingFlushUpdateHandle` 缺 `Reset`）、F-ENV-021（重复块/顺序无注释）；顺序本身经核对是**正确**的（`BindingRegistry` 早于 `ReferenceRegistry`、`Domain` 最后） |
| `GetEnvironment()` (:271-274) | 返回静态单例 | 全模块（44+ 处 `Bind` 等） | 无问题 |
| `NotifyUObjectCreated(Object, Index)` (:276-300) | CDO 或非 GT：入 `AsyncLoadingObjectArray`（加锁）；GT 且非 CDO：立即 `Bind<true>` | `FUObjectListener.cpp:43` ← `GUObjectArray` 创建回调（任意线程） | **问题**：见 F-ENV-007（CDO 依赖 flush 事件）；加锁正确（`:282`、`:295`） |
| `NotifyUObjectDeleted(Object, Index)` (:302-321) | `UStruct` → 移除类描述符；否则移除对象引用；再从延迟数组移除 | `FUObjectListener.cpp:48` ← 删除回调 | 无 P0/P1；`Remove(InObject)`（`:318`）在删除回调中把裸指针隐式转成 `FWeakObjectPtr` 参与比较，语义依赖"删除回调时 index/serial 仍有效"（未发现反例，置信度中） |
| `OnUObjectArrayShutdown()` (:323-325) | 空 | `FUObjectListener.cpp:53` | **问题**：见 F-ENV-019 |
| `OnUnrealCSharpModuleActive()` (:327-330) | `Initialize()` | 模块广播 | 无问题 |
| `OnUnrealCSharpModuleInActive()` (:332-335) | `Deinitialize()` | 模块广播 | 无问题 |
| `OnBlueprintPreCompile(UBlueprint*)` (:338-348) | 若已有描述符则移除并记入 `PendingBindClasses` | `GEditor->OnBlueprintPreCompile()`（`:93`，仅编辑器） | 无问题；`PendingBindClasses` 为弱指针数组（安全） |
| `OnBlueprintCompiled()` (:350-366) | 对 pending 类移除描述符并绑定 CDO，随后清空 | `GEditor->OnBlueprintCompiled()`（`:96`） | 无 P0/P1；`:356` 的第二次 `RemoveClassDescriptor` 对已移除者是 no-op（重复但无害） |
| `OnAsyncLoadingFlushUpdate()` (:369-452) | 拷贝延迟数组 → 过滤未就绪对象 → 在 GT 上绑定 + `ConstructorObject` | `FCoreDelegates::OnAsyncLoadingFlushUpdate`（`:87`） | **问题**：见 F-ENV-007（裸指针窗口 + 重复 `#if` 块） |
| `Bind(UObject*)` (:454-457) | 转发 `FCSharpBind::Bind(UObject*)` | `FRegisterUnreal.cpp:67,84,108,165,170,175` 等 | 无问题 |
| `Bind(const UObject*)` (:459-462) | `const_cast` 转发 | `FStructPropertyDescriptor`/`FObjectPropertyDescriptor` 等 | 无问题（const 正确性被 `const_cast` 绕过，属既有设计） |
| `Bind(UClass*)` (:464-467) | 转发 `FCSharpBind::Bind(UClass*)` | `FRegisterUnreal.cpp:132`、`FRegisterObject.cpp:35,44`、`FClassPropertyDescriptor.cpp:8,15` | 无问题（转发层）；下游丢弃返回值见 F-ENV-018 |
| `Bind(IManagedHandle, const FName&)` (:469-472) | 转发 | `FRegisterStruct.cpp:26` | 无问题 |
| `GetClassDescriptor(const UStruct*)` (:474-477) | 转发；`ClassRegistry==nullptr` 返回 null | `FCSharpBind.inl:11` | 无问题（唯一调用点已判空） |
| `GetClassDescriptor(const FName&)` (:479-482) | 转发（内部会 `LoadObject`） | **无调用方** | **死代码**，见 §4 |
| `AddClassDescriptor` (:484-487) | 转发 | `FCSharpBind.cpp:122` | 无 P0/P1；重入风险见 §7 |
| `RemoveClassDescriptor` (:489-495) | 转发 | `FCSharpBind.cpp:133`、`FCSharpEnvironment.cpp:308,344,356` | 无问题 |
| `RemoveFunctionDescriptor` (:497-503) | 转发 | `FClassDescriptor.cpp:58`（在 `Deinitialize` 期间回调，见 §2 链 4） | 无问题（`ClassRegistry` 判空后转发，但对象正在析构，见 §7 存疑） |
| `GetOrAddPropertyDescriptor` (:505-508) | 转发 | `FRegisterProperty.cpp:17,33,47,62` | 无问题 |
| `AddPropertyHash` (:510-517) | 转发 | `FCSharpBind.cpp:175` | 无问题 |
| `RemovePropertyDescriptor` (:519-525) | 转发 | `FClassDescriptor.cpp:65` | 无问题 |
| `AddObjectReference` (:527-531) | 转发 | `FCSharpBind.inl:84` | 无问题 |
| `GetObject(const UObject*)` (:533-536) | 转发，未命中返回 `InvalidManagedHandle` | `FCSharpBind.cpp:44`、`FClassRegistry.cpp:253`、`FDynamicRegistry.cpp:27` | 无问题（3 个调用点都做了 `IManagedHandleIsValid`） |
| `RemoveObjectReference(const UObject*)` (:538-541) | 转发 | `FCSharpEnvironment.cpp:312` | 无问题 |
| `RemoveObjectReference(IManagedHandle)` (:543-546) | 转发 | **无调用方** | **死代码**，见 §4 |
| `AddStructReference(owner,…)` (:548-554) | 转发 owner 重载 | `FStructPropertyDescriptor.cpp:66`、`TPropertyValue.inl:264,317,494,547` | 无问题；但 owner 可能无效 → F-ENV-012 |
| `GetObject(UScriptStruct*, const void*)` (:556-559) | 转发，未命中返回 `InvalidManagedHandle` | `FStructPropertyDescriptor.cpp:57` | 无问题 |
| `RemoveStructReference` (:561-564) | 转发 | `FRegisterStruct.cpp:51`、`FStructReference.h:14` | 无问题 |
| `GeManagedHandle(const UObject*)` (:566-571) | 转发，未命中返回 `IManagedHandle()`（全零） | 仅同文件 `:579` | 拼写问题见 F-ENV-014；仅内部使用见 §4 |
| `GeManagedHandle(void*, const FProperty*)` (:573-587) | 用 `InAddress - Property->GetOffset_ForInternal()` 反推宿主，按 owner 类型分派 | 6 处（`FStructPropertyDescriptor.cpp:63`、`FArray/FMap/FSetPropertyDescriptor.cpp:42`、`FDelegate/FMulticastDelegatePropertyDescriptor.cpp:44,56`） | **问题**：见 F-ENV-012（可返回无效 owner、`InProperty` 无判空） |
| `GetBinding(void*)` (:589-592) | 转发，未命中返回 `InvalidManagedHandle` | `TPropertyValue.inl:158,205` | 无问题（调用点判空）；底层地址有效性见 F-ENV-001 |
| `RemoveBindingReference` (:594-597) | 转发 | `FBindingReference.h:14`、`TDestructorHelper.inl:18` | 无问题（转发层）；实现在 F-ENV-001/002 |
| `GetOptional` (:600-605) | 转发 | `FRegisterOptional.cpp:76,78,97,105,117,129`、`FOptionalPropertyDescriptor.cpp:38`、`TPropertyValue.inl:1003` | 无问题 |
| `RemoveOptionalReference` (:607-610) | 转发 | `FRegisterOptional.cpp:91` | 无问题 |
| `AddReference(owner, FReference*)` (:613-616) | 转发 | `FBindingRegistry.inl:37`、`FStructRegistry.cpp:105`、`FContainerRegistry.inl:52`、`FDelegateRegistry.inl:52` | **问题**：见 F-ENV-012（不拒绝无效 owner） |
| `RemoveReference(IManagedHandle)` (:618-621) | 转发 | `FObjectRegistry.cpp:92,110`、`FStructRegistry.cpp:119` | 无问题 |
| `AddReference(UObject*)` (:623-626) | 转发 | `FRegisterObject.cpp:115` | 无问题（走 FGCObject 强引用保护） |
| `RemoveReference(UObject*)` (:628-631) | 转发 | `FRegisterObject.cpp:125` | 无问题 |
| `GetBind()` (:633-636) | 返回 `CSharpBind` | `FRegisterEnhancedInputComponent.cpp:99`、`FRegisterInputComponent.cpp:314` | 无问题（但 `GetBind()` 可能在 `Initialize()` 之前返回 nullptr，调用点未判空——两处调用都在运行时 interop 中，早于 Initialize 不会发生，置信度中） |

### 5.2 `FCSharpEnvironment.inl`

| 函数（行） | 做了什么 | 调用方 | 结论 |
|---|---|---|---|
| `Bind<IsNeedOverride>(UStruct*)` (:12-16) | 转发 `FCSharpBind::Bind` | `TPropertyValue.inl:262,278,314,332,492,508,545,562`、`TConstructorHelper.inl:28,35`、`FStructPropertyDescriptor.cpp:7`、`FRegisterDataTableFunctionLibrary.cpp:25` | 无问题；重载陷阱见 F-ENV-017 |
| `Bind<IsNeedOverride>(UObject*)` (:18-22) | 转发 | `FDynamicRegistry.cpp:25,68`、`FCSharpBind.inl:70` 间接 | 无问题 |
| `GetFunctionDescriptor<T>` (:24-28) | 判空转发 | `FClassDescriptor.cpp:80` | 无问题 |
| `GetOrAddFunctionDescriptor<T>` (:30-34) | 判空转发 | `FRegisterFunction.cpp` 26 处、`CSharpFunction.cpp:7` | 无问题 |
| `AddFunctionHash<T>` (:36-43) | 判空转发 | `FCSharpBind.cpp:227,343,366` | 无问题 |
| `TGetAddress<UObject,T>::operator()` (:45-58) | 查对象表并 `static_cast<T*>` | `GetAddress<T,U>` | 无问题（底层用 `Get()`，判空安全） |
| `TGetAddress<UScriptStruct,T>::operator()` (:60-73) | 查结构体表并 cast | 同上 | 无问题 |
| `GetAddress<T,U>(IManagedHandle)` (:75-79) | 分派到 `TGetAddress`（未支持类型 = 空类 → 编译错误，属**编译期**保护） | `FRegisterProperty.cpp:13,29,43,58`、`FRegisterWorld.cpp:129` | 无问题 |
| `GetAddress<UObject>(handle, UStruct*&)` (:81-94) | 查对象表并回填 `InStruct` | **无调用方** | **死代码**，见 §4；注意底层 `FObjectRegistry.cpp:45` 用 `(*FoundObject)->GetClass()`（未判空），复活此路径前需先修 |
| `GetAddress<UScriptStruct>(handle, UStruct*&)` (:96-109) | 查结构体表并回填 `InStruct` | **无调用方** | **死代码**，见 §4 |
| `GetObject<T>(IManagedHandle)` (:111-115) | 查对象表 + `Cast<T>` | `TGetObject`（`TPropertyBuilder.inl` 等 8 处）、`FRegisterWorld.cpp:125,127`、`FRegisterStruct.cpp:32`、`FRegisterEnhancedInputComponent.cpp:171,177`、`FRegisterClass.cpp:14` | **热路径**，无冗余构造（一次 `TMap` 查找 + `Cast`），结论：可接受 |
| `AddStructReference<IsNeedFree>(…)` (:117-124) | 判空转发 | 14 处（见 §4） | 无问题 |
| `GetStruct<T>` (:126-130) | 查结构体表 + cast | 多处（`TPropertyValue.inl` 等） | **热路径**，无冗余构造，可接受 |
| `GetContainer<T>` / `GetContainerObject<T>` / `AddContainerReference<T>`×2 / `RemoveContainerReference<T>` (:132-175) | 5 个判空转发（`FContainerRegistry::TContainerRegistry<T>`） | `FRegisterArray/Map/Set.cpp`、`FArray/FMap/FSetPropertyDescriptor.cpp`、`TPropertyValue.inl`、`TContainerReference.h:15` | 无问题 |
| `GetDelegate<T>` / `GetDelegateObject<T>` / `AddDelegateReference<T>`×2 / `RemoveDelegateReference<T>` (:177-219) | 5 个判空转发（`FDelegateRegistry::TDelegateRegistry<T>`） | `FRegisterDelegate/MulticastDelegate.cpp`、`FDelegate/MulticastDelegatePropertyDescriptor.cpp`、`TDelegateReference.h:15` | 无问题 |
| `GetMulti<T>` / `GetMultiObject<T>` / `AddMultiReference<T,…>` / `RemoveMultiReference<T>` (:221-253) | 4 个判空转发（`FMultiRegistry::TMultiRegistry<T,T>`） | `FRegisterSubclassOf/WeakObjectPtr/LazyObjectPtr/SoftObjectPtr/SoftClassPtr/ScriptInterface.cpp`、对应 PropertyDescriptor、`TPropertyValue.inl:111,140` | 无问题 |
| `GetString<T>` / `GetStringObject<T>` / `AddStringReference<T,…>` / `RemoveStringReference<T>` (:255-287) | 4 个判空转发（`FStringRegistry::TStringRegistry<T>`） | `FRegisterName/String/AnsiString/Utf8String/Text.cpp`、对应 PropertyDescriptor、`TPropertyValue.inl:64` | 无问题 |
| `GetBinding<T>` (:289-295) | 判空转发 | `FRegisterEnhancedInputComponent.cpp:210`、`FRegisterWorld.cpp` 7 处、`TPropertyValue.inl:196,243` | 无问题（转发层）；底层见 F-ENV-001/002 |
| `AddBindingReference<T,IsNeedFree>` (:297-304) | 判空转发 | 4 处模板实例化（见 §4） | **问题**：`IsNeedFree=false` → F-ENV-002 |
| `AddBindingReference<T>(owner,…)` (:306-313) | 判空转发（内部恒 `bNeedFree=false`） | `TPropertyValue.inl:166,213` | **问题**：F-ENV-002 + F-ENV-012 |
| `GetRegistry<T>` (:315-364) | `if constexpr` 分派 11 种注册表 | `FRegisterEnhancedInputComponent.cpp:99`、`FRegisterInputComponent.cpp:314` | **问题**：见 F-ENV-016 |
| `GetOptionalObject<T>` (:367-373) | 判空转发 | `FOptionalPropertyDescriptor.cpp:7`、`TPropertyValue.inl:930` | 无问题 |
| `AddOptionalReference<T,IsMember>` (:375-382) | 判空转发 | `FOptionalPropertyDescriptor.cpp`、`TPropertyValue.inl` | 无问题 |

### 5.3 `FCSharpBind.h` / `FCSharpBind.cpp` / `FCSharpBind.inl`

| 函数（行） | 做了什么 | 调用方 | 结论 |
|---|---|---|---|
| `FCSharpBind()` (:16-19) | 调 `Initialize()` | `FCSharpEnvironment.cpp:63` | 无问题 |
| `~FCSharpBind()` (:21-24) | 调 `Deinitialize()` | `FCSharpEnvironment.cpp:251` | 无问题（但见 F-ENV-009：`Initialize` 重入时旧实例不会被析构） |
| `Initialize()` (:26-32) | 清空 `NotOverrideTypes` + 注册 `OnCSharpEnvironmentInitialize` | 构造函数 | **问题**：`NotOverrideTypes` 归零点见 F-ENV-004；清空与 `FCSharpEnvironment::Initialize` 的时序耦合（`:59` 建域 → `:63` 建 bind → `:144` 广播）经核对**正确** |
| `Deinitialize()` (:34-40) | 摘委托 | 析构 | 无问题 |
| `Bind(UObject*)` (:42-51) | 已有句柄直接返回，否则 `Bind<false>` | `FCSharpEnvironment.cpp:456`、`FCSharpBind.cpp:57` | 无问题（`IManagedHandleIsValid` 判定正确） |
| `Bind(UClass*)` (:53-58) | `Bind<false>(UClass*)` + `Bind(UObject*)` | `FCSharpEnvironment.cpp:466` | **问题**：见 F-ENV-018 |
| `Bind(IManagedHandle, const FName&)` (:60-63) | 转发 `BindImplementation` | `FCSharpEnvironment.cpp:471` | 无问题 |
| `Bind(desc, class, function)` (:65-70) | 有 function 则取名字转发 | `FRegisterInputComponent.cpp:314`、`FRegisterEnhancedInputComponent.cpp:99` | 无问题 |
| `Bind(desc, class, methodName, function)` (:72-76) | 转发 | `FCSharpBind.cpp:68,298` | 无问题 |
| `BindClassDefaultObject(UObject*)` (:78-104) | `CanBind` → 注册类构造器 + `Bind<false>`；否则按设置表 `IsA` 兜底 | `FCSharpEnvironment.cpp:360,436`、`FCSharpBind.cpp:542` | **性能**：见 F-ENV-013；`InObject` 无判空但 3 个调用点均已判空 → 无问题；`CanBind` 失败时不注册构造器（`:82` 只在其内）行为正确 |
| `BindImplementation(UStruct*)` (:106-311) | 先绑父链（`:113-120`），建类描述符，匹配字段/属性/函数/方法并写入 hash | `FCSharpBind.inl:26`（`Bind<...>(UStruct*)`）、`:70`（UObject 路径先绑类） | 三处 O(N×M) 字符串匹配见 F-ENV-013；`Properties`/`Functions` 用 `Encode` 后的名字作键，因 UE 要求同一类内 `UFunction` 名唯一（`TMap<FName, UFunction*>` 函数表），不存在重载覆盖问题（已核实）；`:159-182`/`:211-235` 在遍历 `Fields` 的同时 `Fields.Remove(Field)` 后立即 `break`（`inl` 无迭代器失效风险，但属脆弱写法） |
| `BindImplementation(desc, class, name, function)` (:313-392) | 已存在则返回；取原始函数；按 outer 是否同类分两条路径复制函数并注册 hash/native | `FCSharpBind.cpp:75,298` | 无 P0/P1；`:321` 的 `HasFunctionDescriptor(GetTypeHash(InFunction))` 提前返回会把"重复绑定"当失败（语义上应算成功），属边界语义（P3 级，未单列） |
| `BindImplementation(IManagedHandle, const FName&)` (:394-417) | `LoadObject<UScriptStruct>` → `Bind` → `Malloc` + `InitializeStruct` → `AddStructReference<true>` | `FCSharpBind.cpp:62` ← `FCSharpEnvironment.cpp:471` ← `FRegisterStruct.cpp:26` | **问题**：见 F-ENV-005（未检查返回值 + 重复注册泄漏） |
| `CanBind(UStruct*)` (:419-432) | `NotOverrideTypes` 过滤 + 反射注册表 `IsOverride` | `FCSharpBind.inl:18`、`FCSharpBind.cpp:80` | **问题**：见 F-ENV-004 |
| `GetOriginalFunction(desc, function)` (:434-464) | 递归上溯找到"非 C# 实现"的原始函数 | `FCSharpBind.cpp:326` | 无问题（`nullptr` 已判空；递归深度 = 类层次深度，且有 `superClass != nullptr` 守卫 `:459-461`；`:446-447` 用 `GetFunctionDescriptor<FCSharpFunctionDescriptor>(FString)` 只看 CSharp 描述符，不存在把 `FUnrealFunctionDescriptor` 误 `static_cast` 的问题——已核实 `FClassDescriptor.cpp:80-81` 的模板实参） |
| `IsCallCSharpFunction` (:466-469) | 比较 `GetNativeFunc() == &UCSharpFunction::execCallCSharp` | `FCSharpBind.cpp:441,452` | 无问题 |
| `RegisterCallCSharpNativeFunction(class, function)` (:471-488) | 设置 native func、置 `FUNC_Native`、按需加入 `NativeFunctionLookupTable` | `FCSharpBind.cpp:370,388` | 无问题 |
| `DuplicateFunction(original, class, name)` (:490-534) | 临时清 `FUNC_Native` → `StaticDuplicateObjectEx` → 恢复 → 设 native func → `StaticLink` → 入类函数表 → 按需 `AddToRoot` | `FCSharpBind.cpp:339,381` | 无 P0/P1；`:530` 的 root 配对见 F-ENV-020；`:499`/`:511` 临时改写引擎对象的 Flags（若 `StaticDuplicateObjectEx` 期间有重入会观察到中间态，风险低，未单列） |
| `OnCSharpEnvironmentInitialize()` (:536-544) | 全量遍历 `UClass` 绑定 CDO | 委托（`FCSharpEnvironment.cpp:144` 广播） | **性能**：见 F-ENV-013 |
| `Bind<IsNeedOverride>(UStruct*)` (`FCSharpBind.inl:8-27`) | 已有描述符 → true；`IsNeedOverride` 时 `CanBind` 过滤并缓存 | `FCSharpEnvironment.inl:15` | 见 F-ENV-004 |
| `Bind<IsNeedOverride>(UObject*)` (`inl:29-33`) | 转发 `BindImplementation` | `FCSharpEnvironment.inl:21` | 无问题 |
| `Bind(FClassReflection*, FClassReflection*, handle)` / `(…, key, value, handle)` (`inl:35-47`) | 转发容器/多值注册 | `FRegisterArray/Map/Set.cpp`、`FRegisterSubclassOf` 等 | 无问题 |
| `Bind(FClassReflection*, handle)` (`inl:49-53`) | 转发委托注册 | `FRegisterDelegate/MulticastDelegate.cpp` | 无问题 |
| `BindImplementation<IsNeedOverride>(UObject*)` (`inl:55-87`) | 判空 → 绑类描述符 → 取 `FClassReflection` → `NewObject()` → `AddObjectReference` | `FCSharpBind.inl:32` | 无问题；`AddObjectReference` 返回值未检查（`:84`）→ 与 F-ENV-005 同类，但失败时对象句柄仍返回给 C#（若 `ObjectRegistry==nullptr` 则句柄未被登记，C# 侧持有无法反查的对象）——已并入 F-ENV-005 的"未检查返回值"范畴 |
| `BindImplementation(FClassReflection*, FClassReflection*, handle)` / `(…, key, value, handle)` (`inl:89-122`) | `FTypeBridge::Factory` 造属性 + `new T(...)` 容器助手 + 注册 | 容器注册链 | 无 P0/P1；`Property`/`ContainerHelper` 均未判空（`Factory` 返回 nullptr 时会崩），属同类"未检查返回值"（已在 F-ENV-005 归类，未单列） |
| `BindImplementation(FClassReflection*, handle)` (`inl:124-132`) | `new T()` 委托助手 + 注册 | 委托注册链 | 无问题 |

### 5.4 `FBindingRegistry.h` / `.cpp` / `.inl`

| 函数（行） | 做了什么 | 调用方 | 结论 |
|---|---|---|---|
| `FBindingRegistry()` (:5-8) | 调 `Initialize()` | `FCSharpEnvironment.cpp:81` | 无问题 |
| `~FBindingRegistry()` (:10-13) | 调 `Deinitialize()` | `FCSharpEnvironment.cpp:188` | 无问题 |
| `Initialize()` (:15-17) | **空** | 构造函数 | 空实现，见 §4 |
| `Deinitialize()` (:19-38) | 遍历句柄表：`GCHandle_Free` + 条件 `delete wrapper`；然后清空两表 | 析构 | **问题**：见 F-ENV-002（条件 delete 漏） + F-ENV-021（就地改键） |
| `GetObject(void*)` (:40-45) | 地址 → 句柄，未命中返回 `InvalidManagedHandle` | `FCSharpEnvironment.cpp:591` | **问题**：见 F-ENV-001（无有效性校验） |
| `RemoveReference(IManagedHandle)` (:47-75) | 释放句柄、条件释放 wrapper、删两表条目 | `FCSharpEnvironment.cpp:596` | **问题**：见 F-ENV-001/002；`:52` 的 `FoundValue->AddressWrapper->Value` 无判空（当前不可达 null，属防御性缺失） |
| `GetBinding<T>(handle)` (`inl:6-12`) | 句柄 → wrapper → `static_cast<T*>(Value)` | 见 §4 | **问题**：见 F-ENV-001；无 `AddressWrapper` 判空 |
| `AddReference<T,IsNeedFree>(…)` (`inl:14-25`) | 写地址表 + `new wrapper` + 写句柄表，恒返回 true | 4 处模板实例化 | **问题**：见 F-ENV-001/002/003 |
| `AddReference<T>(owner,…)` (`inl:27-38`) | 同上（`bNeedFree` 恒 false）+ `new FBindingReference` 挂 owner | `TPropertyValue.inl:166,213` | **问题**：F-ENV-002（wrapper 必漏）+ F-ENV-012（owner 可能无效）；`:37` 的 `new FBindingReference` 在 `AddReference` 返回 false 时泄漏（`FReferenceRegistry` 未接管）——已归入 F-ENV-012 的修复建议 |

---

## 6. 横向维度汇总

### 6.1 空指针 / 越界 / 未检查返回值

| 位置 | 事实 | 判定 |
|---|---|---|
| `FCSharpEnvironment.cpp:575` | `InProperty->GetOffset_ForInternal()` 无判空 | 见 F-ENV-012（低风险，调用点均传有效属性） |
| `FCSharpEnvironment.cpp:584` | `GetManagedHandle(nullptr, Owner)` 原样传入 | `FStructRegistry.cpp:85-89` 只用 `TMap::Find`，安全 |
| `FObjectRegistry.cpp:45`（**死路径**） | `InStruct = (*FoundObject)->GetClass();` —— `(*FoundObject)` 先取出 `TWeakObjectPtr`，再由 `operator->` 取值（引擎实现 `Engine\Source\Runtime\Core\Public\UObject\WeakObjectPtrTemplates.h:204-207` `operator->() { return Get(); }`），**对象已被 GC 时 `Get()` 返回 nullptr，随即 `->GetClass()` 即空指针解引用** | 该路径唯一入口 `FCSharpEnvironment.inl:87` 已是死代码；若复活需先改为 `if (const auto Obj = FoundObject->Get()) { InStruct = Obj->GetClass(); ... }`（对照组：同文件 `:38` 用的是安全的 `Get()`） |
| `FCSharpEnvironment.cpp:414` | `AddStructReference<true>` 返回值丢弃 | 见 F-ENV-005 |
| `FCSharpBind.inl:84` | `AddObjectReference(...)` 返回值丢弃 | 并入 F-ENV-005 |
| `FCSharpBind.cpp:55` | `Bind<false>(InClass)` 返回值丢弃 | 见 F-ENV-018 |
| `FCSharpBind.cpp:80` | `CanBind(InObject->GetClass())` 未判 `InObject` | 3 个调用点均已判空 → 无问题 |
| `FBindingRegistry.cpp:52`、`inl:11` | `FoundValue->AddressWrapper->...` 未判空 | 防御性缺失，并入 F-ENV-001（当前不可达 null） |
| `FObjectRegistry::GetAddress` / `GetObject` | 用 `Get()` 取弱指针，失效返回 nullptr，**3 个调用点都判空** | 正确 |
| `FCSharpEnvironment::GetObject<T>`（`inl:114`） | `Cast<T>(nullptr)` 安全；`GetStruct<T>`（`inl:129`）`static_cast<T*>(nullptr)` 安全 | 正确 |
| 句柄 `IntPtr.Zero` 校验 | `IManagedHandleIsValid`（`IManagedHandle.h:27-30`）在环境层入口广泛使用（`FCSharpBind.cpp:45`、`FDynamicRegistry.cpp:28`、`FRegisterEnhancedInputComponent.cpp:171` 等） | 覆盖率好；缺口是 `AddReference(InOwner=无效)`（F-ENV-012）与 `GetBinding` 的返回地址（F-ENV-001） |

### 6.2 内存与资源泄漏

| 对象 | 分配点 | 释放点 | 结论 |
|---|---|---|---|
| `TBindingAddressWrapper` | `FBindingRegistry.inl:19,33` | `FBindingRegistry.cpp:29,64`（**仅 `bNeedFree==true`**） | **泄漏**：F-ENV-002 |
| `FClassDescriptor` | `FClassRegistry.cpp:104` | `FClassRegistry.cpp:159` / `FClassRegistry.cpp:42` | 配对完整；但重入时可创建两份（§7 存疑） |
| `FDomain` / 12 个注册表 | `FCSharpEnvironment.cpp:59-85` | `FCSharpEnvironment.cpp:177-268` | 配对；但 `~FCSharpEnvironment` 不触发（F-ENV-010），`Initialize` 重入会漏（F-ENV-009） |
| `FStructReference` / `TContainerReference` / `TDelegateReference` / `FBindingReference` | `FStructRegistry.cpp:105`、`FContainerRegistry.inl:52`、`FDelegateRegistry.inl:52`、`FBindingRegistry.inl:37` | `FReferenceRegistry.cpp:13,50`（`delete Reference`） | 配对完整；但键为 `{0}` 的条目不会被常规路径删除（F-ENV-012），且 `AddReference` 失败时 `new FBindingReference` 泄漏 |
| `FMemory::Malloc` 的结构体内存 | `FCSharpBind.cpp:410` | `FStructRegistry.cpp:38,135`（受 `bNeedFree` **且** `Value.IsValid()` 双重限制） | 见 F-ENV-005 |
| C# `GCHandle` | C# 侧创建（`IScriptDomain::NewObject` 等） | C++ 侧 `FDomain::GCHandle_Free`（20 余处注册表 Deinitialize/Remove） | 正常路径配对完整（顺序正确：注册表先于 `Domain` 释放，`Domain` 最后 → `FDomain.cpp:94-100` 仍能拿到脚本域） |
| UObject 强引用保护 | `FReferenceRegistry`（`FReferenceRegistry.h:5` 继承 `FGCObject`、`:13` 实现 `AddReferencedObjects`、`:29` `TArray<TObjectPtr<UObject>> ObjectArray`；`FReferenceRegistry.cpp:22-25` 把整个 `ObjectArray` 交给 `Collector`、`:27-30` `GetReferencerName()`、`:59-69` `AddReference(UObject*)` → `ObjectArray.AddUnique(InObject)`） | — | **正确**：`FCSharpEnvironment` 自身只持弱指针（`h:327 TArray<FWeakObjectPtr>`、`h:330 TArray<TWeakObjectPtr<UClass>>`），因此无需 `AddReferencedObjects`（也没有实现），设计自洽。**★ 跨报告一致性（复核追加，逐行实测确认）**：`FReferenceRegistry` 确实是**会被 UE GC 收集的 `FGCObject`**、`ObjectArray` 中的引用**受 GC 保护**，因此"原生侧未保活 UObject / 句柄可能指向已被 UE GC 回收的 UObject"这一前提（`06-CSharp运行时与脚本/02b-对象字符串与工具类型包装.md` 的 F-CS2B-008 已据此证伪）**在本报告中从未被复用**：本报告 21 条 Finding 的悬垂/失效论证都不依赖"缺少保活" —— F-ENV-001/002/003 的失效对象是 `FBindingRegistry` 里 `new` 出来的 wrapper 与引擎侧 `TUniquePtr` 拥有的绑定对象，F-ENV-011 的失效对象是跨域句柄值，F-ENV-012 的失效对象是 `FReference` 子对象；唯一涉及"UObject 被 GC 后失效"的观察是 §6.1 的 `FObjectRegistry.cpp:45`，其失效来源是 `FObjectRegistry` 自己那张 `TWeakObjectPtr` 弱表（`FObjectRegistry.h:47`，与保活无关），且该路径已被标为**死代码**、未列为 Finding。故本报告**无任何条目需要因该假设而调级别或调可达性** |
| `TStrongObjectPtr` | 全插件仅 1 处且不在本模块（`UnrealCSharpEditor.h:61`） | — | 本模块不使用 |
| `AddToRoot`/`RemoveFromRoot` | **实测（范围 `Source/UnrealCSharp`，按行计）：含 `AddToRoot` 的行 8 行、含 `RemoveFromRoot` 的行 6 行**；其中真正的 `UObject::AddToRoot()` 调用 **6 处**：`FCSharpBind.cpp:530`、`FRegisterInputComponent.cpp:312`、`FRegisterEnhancedInputComponent.cpp:97`、`FMulticastDelegateHelper.cpp:25`、`FDelegateHelper.cpp:24`、`FRegisterObject.cpp:89`（末条是 C# 可调 API） | `RemoveFromRoot()` **5 处**：`FCSharpFunctionRegister.cpp:61`、`FRegisterClass.cpp:23`、`FMulticastDelegateHelper.cpp:39`、`FDelegateHelper.cpp:38`、`FRegisterObject.cpp:97` | 逐条核对后**均有配对**（`FRegisterObject.cpp:89/97` 与 `FMulticastDelegateHelper.cpp:25/39`、`FDelegateHelper.cpp:24/38` 各自成对），但 `FCSharpBind.cpp:530` 与 `FRegisterInputComponent.cpp:312`/`FRegisterEnhancedInputComponent.cpp:97` 的配对跨文件/跨语言（F-ENV-020）。原文写"本模块 4 处 / 3 处"漏计，已修正 |

### 6.3 线程安全

| 检查项 | 事实 | 判定 |
|---|---|---|
| 是否有 `check(IsInGameThread())` | **实测：整个 `Source/` 里 `check(` 命中 0 处**（原文写"唯一命中为 `UnrealCSharpEditorStyle.cpp:25 ensure(...)`"把 `ensure` 误记为 `check(`）；`ensure` 全 `Source/` 仅 1 处：`Source/UnrealCSharpEditor/Private/UnrealCSharpEditorStyle.cpp:25` `ensure(StyleInstance.IsUnique());` | 所有注册表**无断言**保护；只能靠 `IsInGameThread()` 分支（实测全 `Source/` 仅 3 处：`FCSharpEnvironment.cpp:289`、`FClassRegistry.cpp:241`、`FDynamicRegistry.cpp:23`） |
| 锁的覆盖范围 | 仅 `FCSharpEnvironment::CriticalSection`（`h:325`）保护 `AsyncLoadingObjectArray`（`cpp:282,295,316,379,424`） | 12 个注册表**全部无锁**；因此"非 GT 只入队"是唯一防线 |
| 非 GT 入口 | `NotifyUObjectCreated`（`cpp:276`，任意线程）→ 非 GT 分支入队（`:295-297`），CDO 一律入队（`:282-284`） | 正确；`AsyncLoadingObjectArray` 的所有读写都在锁内（`:284,297,318,381,426-429`）✔ |
| 异步编译线程与主线程共享表 | `FCSharpCompilerRunnable` 在 worker 线程编译，最终用 `FFunctionGraphTask(..., ENamedThreads::GameThread)` + `WaitUntilTaskCompletes`（`FCSharpCompilerRunnable.cpp:213-242`）把域重载切到 GT | 编译线程本身**不直接触碰**环境表（`FCSharpEnvironment` 在 `Source/` 中零外部引用，见 §6.6）✔ |
| 静态/全局可变状态 | `FCSharpBind::NotOverrideTypes`（`FCSharpBind.cpp:14`）、`FCSharpEnvironment::Environment`（`FCSharpEnvironment.cpp:33`）、`SignalActions`（`FCSharpEnvironment.cpp:19`，仅 Mac） | F-ENV-004 / F-ENV-008 / F-ENV-006；`SignalActions` 还会在信号处理函数里被 `TMap::operator[]` 访问（分配内存，非信号安全） |
| 跨帧延迟回调 | `FRegisterStruct.cpp:49`、`FCSharpEnvironment.cpp:432-451` | F-ENV-011（无域代次校验） |

### 6.4 异常与错误处理

- 本模块 7 个文件中**没有 `try/catch`、没有 `ensure`、没有 `check`**（脚本域侧 `FDomain.cpp:112-129 GetTraceback` 的存在说明"托管异常"被当作可预期事件处理，而非 C++ 异常）。
- `ensure`/`check` 的缺失意味着：`FBindingRegistry` 的空指针解引用、`AddStructReference` 失败的静默、`Bind<false>` 失败的丢弃都**不会在 Shipping 触发断言**，只会静默产生错误状态（并可能延迟到 GC 时崩溃）。建议按 §3 各条建议补 `ensureMsgf`（`ensure` 在 Shipping 下不崩，只上报，符合"不引入新崩溃"的要求）。

### 6.5 性能

| 位置 | 事实 | 判定 |
|---|---|---|
| `GetObject<T>` / `GetStruct<T>` / `GetBinding<T>`（互调热路径，`inl:112,127,290`） | 一次 `TMap` 查找 + `Cast`/`static_cast`；**无 `FString`/`FName` 构造、无字符串比较、无按值传大对象** | 结论：热路径没有明显的字符串/拷贝开销（**这一维度是健康的**） |
| `FCSharpEnvironment.h:98,113,135,138,152,155,191,204,216,219,232` 等模板参数 | 大对象均以 `const T&`/指针传递；`IManagedHandle` 是 16 字节 POD 按值传递 | 可接受（`IManagedHandle` 可按 `const&` 优化，收益极小） |
| `FCSharpBind.cpp:159-182,211-235,282-307` | 三处 O(N×M) FString 比较 + `FString::Printf("__%s")`（`:167,219,356`） | 绑定期一次性成本，但发生在域重载的 GT 同步路径上 → F-ENV-013 |
| `FCSharpBind.cpp:93-99` | 每类遍历设置表 + `IsA` | F-ENV-013 |
| `FCSharpBind.cpp:538` | 全量 `TObjectRange<UClass>` | F-ENV-013 |
| `FCSharpEnvironment.cpp:371-430` | 每次 flush 拷贝整个 `AsyncLoadingObjectArray` 两次（弱指针数组 + 裸指针数组），并逐个 `RemoveAt`（`:428`，O(N) 位移） | P3 级：可用 `RemoveAllSwap`/一次性重建替代；量级取决于数组长度，未实测，未单列为 Finding |
| `FClassDescriptor::GetFunctionDescriptor(FString)` 线性扫描（`FClassDescriptor.cpp:78-88`） | 被 `FCSharpBind::GetOriginalFunction`（`FCSharpBind.cpp:446-447`）调用，绑定期 | 与 F-ENV-013 同类，未单列 |

### 6.6 死代码

见 §4。补充事实：**`FCSharpEnvironment` 在整个 `Source/` 中只被 `UnrealCSharp` 模块内部引用**——检索模式 `FCSharpEnvironment`，范围 `Source/**` 下 `*.h,*.cpp,*.inl`（**实测 507 个文件**；原文写 543），过滤掉 `\Source\UnrealCSharp\` 之后命中 **0**（用 `Select-String` 在 507 个文件的非 `Source\UnrealCSharp\` 子集上重跑，结果为 0，与原文一致）。因此本文件中 `UNREALCSHARP_API` 导出的 public 方法即便无内部调用者，也只对有外部源码依赖的使用者可访问——按规范标为"可能是外部 API，需确认"。

### 6.7 平台兼容

| 项 | 事实 | 判定 |
|---|---|---|
| 句柄宽度 | `IManagedHandle` = `int64`（`IManagedHandle.h:7`），`IManagedHandleFromObject`/`ToObject` 走 `UPTRINT` | 32/64 位均安全 |
| 信号处理 | `#include <signal.h>`（`cpp:9`）；`SIGBREAK` 仅在 `PLATFORM_WINDOWS` 使用（`cpp:112-115`）；Mac 用 `sigaction`（`cpp:122-138`），其它平台用 `signal`（`cpp:140`） | 宏使用正确；但存在 F-ENV-006 的语义问题 |
| Mac 全局容器 | `TMap<int32, struct sigaction> SignalActions;`（`cpp:19`，全局非 static） | F-ENV-006 第 3 点 |
| 引擎私有成员访问 | `ACCESS_PRIVATE_MEMBER_PROPERTY(UObjectBase, ObjectFlags, EObjectFlags)`（`cpp:16`）+ `Object->*TAccessPrivate<...>::Value`（`cpp:397`） | 版本脆弱点，随引擎结构变化即失效；被 `~RF_AllFlags` 的语义问题放大（F-ENV-007 第 3 点） |
| 引擎版本宏 | `UE_E_INTERNAL_OBJECT_FLAGS_ASYNC_LOADING`（`cpp:400,407`）、`UE_F_OPTIONAL_PROPERTY`（`h:224,356` 等）、`UE_F_UTF8_STR_PROPERTY`/`UE_F_ANSI_STR_PROPERTY`（跨文件） | 使用一致；`UE_F_OPTIONAL_PROPERTY` 关闭时 `GetRegistry<T>` 的分支缺失属 F-ENV-016 |
| `int` 宽度假设 / `sprintf` | 7 个文件中无 `sprintf`；索引/长度使用 `int32`/`uint32`/`uint16` 明确类型（`cpp:384` `auto i = Num() - 1` 推导为 `int32`） | 无问题 |
| 字节序 | 无字节序相关代码 | 无问题 |

### 6.8 可优化 / 可读性 / 一致性

- F-ENV-014（拼写）、F-ENV-015（只写成员）、F-ENV-016（缺 `else`/`const`）、F-ENV-021（**13 段**重复 delete、就地改键）。
- 命名不一致：`GeManagedHandle` vs `GetManagedHandle`；`FCSharpBind::Bind` 有 10 个重载而 `FCSharpEnvironment::Bind` 有 6 个，可读性差（F-ENV-017）。
- **注释与代码不一致**：未发现"注释说了但代码没做"的情形——本模块 7 个文件**几乎没有注释**（`FCSharpEnvironment.cpp` 中仅信号类型处有 6 条单行注释 `:102-117`，语义与代码一致；`FCSharpEnvironment.h` 全文件无注释）。因此这里的"不一致"表现为**关键不变量无注释**：注册表释放顺序（`cpp:186-268`）、`AddToRoot` 的跨语言配对（`FCSharpBind.cpp:530`）、`Bind` 模板重载语义（`h:51-63`）、`Domain` 成员的用途（`h:309`）。这些都已分别对应 F-ENV-020/021/017/015。

---

## 7. 未覆盖 / 存疑项

1. **~~未验证假设（F-ENV-001）~~ → 已用引擎源码解决（并推翻原假设）**：原文写"本机无引擎源码可核对"是**错误的**——本机存在完整 UE 5.6 引擎源码 `Engine\Source`（另有 `Engine\Plugins\EnhancedInput`、`Engine\Plugins\...`）。核对结论：`UEnhancedInputComponent::BindAction` 返回的**不是** `TArray` 元素引用，而是 `TArray<TUniquePtr<FEnhancedInputActionEventBinding>>`（`Engine\Plugins\EnhancedInput\Source\EnhancedInput\Public\EnhancedInputComponent.h:362`）中 `TUniquePtr` **所拥有的堆对象**的引用（`:453`、`:468` `return *EnhancedActionEventBindings.Add_GetRef(MoveTemp(AB));`）。因此原"数组扩容导致已登记地址失效"的机制**不成立**，F-ENV-001 的问题描述已按真实机制（引擎侧移除绑定/清空绑定后注册表条目残留 → 悬垂；栈上实参地址入表 → 悬垂）改写。**表结构缺少有效性校验**这一半结论与引擎实现无关，仍然成立。
2. **~~未验证假设（F-ENV-004）~~ → 已用引擎源码解决**：`TWeakObjectPtr` 的 `GetTypeHash` **不取** `Get()` 的指针哈希，而是 `uint32(ObjectIndex ^ ObjectSerialNumber)`（`Engine\Source\Runtime\CoreUObject\Public\UObject\WeakObjectPtr.h:353-360`，由 `Runtime\Core\Public\UObject\WeakObjectPtrTemplates.h:494-497` 转发）。故 GC 后条目的哈希**不会退化为 0**；相等判定为"index+serial 相同 或 两者皆 invalid"（`WeakObjectPtr.h:172-181`），新探针基本不会被陈旧条目误命中。结论：F-ENV-004 的性质是**纯内存只增不减**（原方向正确），**不是**"跨代误判"；F-ENV-004 正文与 §7 的相应表述均已修正。
3. **未实测的量化项**：F-ENV-004 的增长速率、F-ENV-013 的实际耗时、F-ENV-007 中裸指针窗口内是否真的会触发 GC、F-ENV-011 的"旧句柄命中新域条目"是否真会发生——都只有结构性证据，没有 profiling/压测数据。（**进展**：F-ENV-007 的"事件不来"一半已被引擎源码排除，见 F-ENV-007 的 `复核证据`；F-ENV-011 的"句柄会被复用"一半已找到计数器不重置 + 托管静态量重新起算的证据，仍缺实机复现。）
4. **重入创建的重复 `FClassDescriptor`（未列为 Finding，因未证实）**：`FClassRegistry::AddClassDescriptor`（`FClassRegistry.cpp:97-109`）先 `new FClassDescriptor(InStruct)` 再写 `ClassDescriptorMap`；而 `FClassDescriptor::Initialize`（`FClassDescriptor.cpp:25-30`）会执行 `Class->ConstructorClass()`（`FClassReflection.cpp:738-744`，反射调用托管方法）。**进展**：`FUNCTION_CLASS_CONSTRUCTOR` 的实值是 **`FString(".cctor")`**（`Source/UnrealCSharpCore/Public/CoreMacro/FunctionMacro.h:3`），即调用该 C# 类型的**静态构造函数**——C# 编译器生成的 `.cctor` 会执行全部静态字段初始化（例如 `StaticClass()`/`StaticStruct()` 的 `??=` 单例、静态容器初始化），因此"`.cctor` 里回调进 C++ 并重入 `Bind`/`AddClassDescriptor`"的路径在机制上是通的。我仍未找到**确凿的**"某个类型的 `.cctor` 会重入同一 UStruct 的 `AddClassDescriptor`"实例（`.cctor` 每类型只运行一次，重入同一类型需要其 `.cctor` 引用自己的 `StaticClass()`），因此仍标为存疑而非 Finding。定位方式：在 `FClassRegistry.cpp:106` 的 `ClassDescriptorMap.Add` 前加 `ensure(!ClassDescriptorMap.Contains(InStruct))`，跑一次全量域重建观察。
5. **`FObjectRegistry::GetAddress(handle, UStruct*&)` 的 `(*FoundObject)->GetClass()`（`FObjectRegistry.cpp:45`）**：目前是死代码（F-ENV-001 之外的独立观察），所以未列为 Finding。**补充证据**：`TWeakObjectPtr::operator->` 的实现就是 `return Get();`（`Engine\Source\Runtime\Core\Public\UObject\WeakObjectPtrTemplates.h:204-207`），故对象被 GC 后这里确实是 **nullptr 解引用**。**若**未来有人复活 `FCSharpEnvironment::GetAddress<T>(handle, UStruct*&)`（`FCSharpEnvironment.inl:81-109`），这条会立刻变成 P1。
6. **`~FCSharpEnvironment` 与 `FUnrealCSharpModuleDelegates` 的析构顺序**（F-ENV-008）：我未实测两个静态对象的实际构造/析构顺序（需要平台相关的链接顺序实验），只给出标准结论（跨 TU 顺序未定义）。
7. **未覆盖的相邻实现**：`FStructRegistry` / `FContainerRegistry` / `FDelegateRegistry` / `FMultiRegistry` / `FStringRegistry` / `FOptionalRegistry` / `FObjectRegistry` / `FReferenceRegistry` / `FClassRegistry` 的内部实现不在我的 7 个文件范围内，我只读了为取证所需的部分（行号已在 §0.2 列出）。其中已发现的、**归属其它报告**的问题（不重复编号，仅提示）：
   - `FStructRegistry.cpp:126-139`：释放 `FMemory::Free` 被 `Value.Value.IsValid()` 条件跳过（与 F-ENV-005 相关）。
   - `FRegisterEnhancedInputComponent.cpp:194-196` / `FRegisterInputComponent.cpp:312`：绑定地址来源与 `AddToRoot` 的跨文件配对。
   - `FRegisterStruct.cpp:49-52`：延迟释放携带旧句柄（F-ENV-011 的实例）。
   - `FClassRegistry.cpp:40-47` 与 `FCSharpEnvironment.cpp:242` 的交互：`delete ClassRegistry` 期间 `FClassDescriptor::~` 会通过 `FCSharpEnvironment::ClassRegistry`（**尚未置空**）回调 `RemoveFunctionDescriptor`/`RemovePropertyDescriptor`。经核对这条回调链只触碰 `CSharpFunctionDescriptorMap`/`UnrealFunctionDescriptorMap`/`PropertyDescriptorMap`（都还没被销毁），因此**当前不崩溃**；但"在析构中的对象上接受回调"是脆弱不变量，建议 `Deinitialize` 改为先把指针搬进局部变量再置空：
     ```cpp
     if (auto* Registry = ClassRegistry) { ClassRegistry = nullptr; delete Registry; }   // 置空前置
     ```
     为避免与"注册表释放顺序"这条主线重复，我把它作为建议写在这里，未单列 Finding。
8. **未做的实验**：没有编译、没有运行编辑器、没有做内存/性能 profiling 或 fuzz；所有结论均基于静态阅读与全文检索。§3 各条"验证方式"是可执行的下一步，但尚未执行。
9. **取证范围与遗留**：
   - **实际读过的行区间（`read` 工具实际输出）**：报告正文 1-1919 行**全读**，改写区域亦回读（全文 1992 行）；命令文件 `FCSharpEnvironment.cpp` 1-636、`FCSharpEnvironment.h` 1-361、`FCSharpEnvironment.inl` 1-383、`FCSharpBind.cpp` 1-545、`FCSharpBind.h/.inl` 全、`FBindingRegistry.h/.cpp/.inl` 全、`TMapping.inl` 全；取证文件 `FUObjectListener.cpp` 全、`FDynamicRegistry.cpp` 全、`FClassRegistry.cpp` 全、`FObjectRegistry.cpp` 全、`FStructRegistry.cpp` 全、`FReferenceRegistry.cpp` 全、`FClassDescriptor.cpp` 全、`FDomain.cpp` 全、`FRegisterStruct.cpp` 全、`FRegisterProperty.cpp` 全、`FRegisterObject.cpp` 25-74、`FRegisterClass.cpp` 全、`FRegisterEnhancedInputComponent.cpp` 80-227、`FRegisterInputComponent.cpp` 295-324、`FStructPropertyDescriptor.cpp` 全、`FClassPropertyDescriptor.cpp` 全、`FArrayPropertyDescriptor.cpp` 36-55、`FDelegatePropertyDescriptor.cpp` 38-62、`FCSharpFunctionRegister.cpp` 全、`FEngineListener.cpp` 1-90、`UnrealCSharpCore.cpp` 全、`UnrealCSharp.cpp` 全、`FUnrealCSharpDelegates.cpp` 全、`UnrealCSharpEditor.cpp` 128-152、`UnrealCSharpEditorStyle.cpp` 1-40、`TArgument.inl` 全、`TOut.inl` 全、`TDestructorHelper.inl` 全、`TContainerReference.h` 全、`TDelegateReference.h` 全、`FStructReference.h` 全、`FBindingReference.h` 全、`FReferenceRegistry.h` 全、`FStructRegistry.h` 50-79、`FObjectRegistry.h` 全、`IManagedHandle.h` 全、`UnrealCSharp.h` 全、`FContainerRegistry.inl` 30-89、`FDelegateRegistry.inl` 30-89、`TPropertyValue.inl` 150-304 与 930/1003 命中行；C# 侧 `StructImplementation.cs` 全、`TArray.cs` 全、`HandleData.cs` 全、`AssemblyLoader.cs` 全、`EnhancedInputComponent.cs` 全、`UnrealTypeWeaver.cs` 120-189、`UnrealTypeSourceGenerator.cs` 145-189。
   - **引擎源码（本报告使用）**：`Engine\Source\Runtime\CoreUObject\Public\UObject\WeakObjectPtr.h`（:160-204、:348-365）、`Runtime\Core\Public\UObject\WeakObjectPtrTemplates.h`（:190-229、:400-514）、`Runtime\CoreUObject\Private\Serialization\AsyncLoading2.cpp`（:9166-9200、:9519-9597）、`Runtime\CoreUObject\Private\UObject\UObjectGlobals.cpp`（:889-909）、`Runtime\CoreUObject\Private\Serialization\AsyncPackageLoader.cpp`（:369-382）、`Runtime\Engine\Private\GameEngine.cpp`（:1818-1835、:1985-2044）、`Runtime\Engine\Private\UnrealEngine.cpp`（:11878-11935、:19360-19389）、`Runtime\Engine\Private\CoreSettings.cpp`（:12）、`Runtime\Engine\Classes\Components\InputComponent.h`（:125-374）、`Editor\UnrealEd\Private\EditorEngine.cpp`（:1750-1767）、`Engine\Plugins\EnhancedInput\Source\EnhancedInput\Public\EnhancedInputComponent.h`（:352-476）、`Engine\Plugins\EnhancedInput\Source\EnhancedInput\Private\EnhancedInputComponent.cpp`（:36-98）。
   - **仍未覆盖**：`FStructRegistry` / `FContainerRegistry` / `FDelegateRegistry` / `FMultiRegistry` / `FStringRegistry` / `FOptionalRegistry` 的内部实现（非本报告 7 个目标文件，仅读取证所需片段）；`Script/Weavers/UnrealTypeWeaver.cs` 的完整织入逻辑；F-ENV-011 的实机复现；F-ENV-013 的 profiling。

---

*报告结束。所有行号来自 `read`/`grep` 的实际输出；未修改任何插件源码。21 条 Finding 的 `复核结论/可达性/复核证据/级别变动` 四个字段逐条给出；§0/§4/§5/§6/§7 中的计数、文件数与引擎侧论断均按源码核对。*

