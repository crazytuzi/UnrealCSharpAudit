# 容器 Helper 分析（FArrayHelper / FMapHelper / FSetHelper）

> 本报告 30 条发现的**严重度见各 Finding 的「复核结论」字段**（确认 **19**、部分确认 **9**、证伪 **2**、无法验证 **0**；含 4 条判定为 `撤销（非缺陷）`、15 条级别调整）。
> **新增关键依据**：本机 UE 5.6 引擎源码已可直接引用（`Engine\Source`），因此原先所有"引擎侧假设"（`FScriptArrayHelper` 是否析构、`FScriptXxx::Empty`/`RemoveAt` 是否维护哈希、`TArray::Reset` 语义、容器与 `T*` 的布局等价性）**已全部落地为可核对的行号**，见 §0.3 与各条的 `复核证据` 字段。
> **处置更新（修复提交 `492ce5f7`）**：`F-HLP-023` 已修复（修复落在**调用侧**：`FRegisterMap`（`FindKey`/`Find`/`Get`/`GetEnumeratorKey`/`GetEnumeratorValue`）、`FRegisterArray::Get`、`FRegisterSet::GetEnumerator` 共 **7 处**取值实现改为先判 helper 是否命中，未命中回退到该提交新增的 `FPropertyDescriptor::GetDefaultValue`，不再把 `nullptr` 交给描述符 `Get`；见该 Finding 正文「处置（已执行）」）。**编号与计数口径不变**（仍 **30 条发现**：确认 **19**、部分确认 **9**、证伪 **2**、无法验证 **0**；含 4 条判定为 `撤销（非缺陷）`、15 条级别调整）；`严重度`/`复核结论`/`可达性` 等字段保持原值，只加处置标注。（修复提交：`492ce5f7` "Null Validation"）





> 分析范围：`Plugins/UnrealCSharp/Source/UnrealCSharp/{Public,Private}/Reflection/Container/`
> 覆盖文件：6 个（FArrayHelper.h/.cpp、FMapHelper.h/.cpp、FSetHelper.h/.cpp）
>
> 6 个指定文件全部读完；辅助阅读文件见 §0.2
> 结构与规范的偏差（已声明）：规范 §2 要求"发现按 P0→P3 降序"，本报告把 **FArrayHelper 的发现正文放在 §5**（已按 P0→P3 排序），把 **FMapHelper/FSetHelper 的发现正文放在 §7.3/§7.6**（与各自的逐函数清单相邻，便于对照），并在 **§5.0 给出全部 30 条发现按严重度排序的索引表**，以保证按严重度检索的能力。每条 Finding 自身都带 `严重度` 字段。

---

## 0. 覆盖范围与阅读清单

### 0.1 本报告主体（6 个指定文件）

| 文件 | 行数 | 是否读完 | 备注 |
|---|---|---|---|
| `Source/UnrealCSharp/Public/Reflection/Container/FArrayHelper.h` | 90 | 是（1–90） | |
| `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp` | 347 | 是（1–347） | 任务书写的 268 行是"非空行数"（PowerShell `Measure-Object -Line` 不计空行），实际 347 行 |
| `Source/UnrealCSharp/Public/Reflection/Container/FMapHelper.h` | 67 | 是（1–67） | |
| `Source/UnrealCSharp/Private/Reflection/Container/FMapHelper.cpp` | 263 | 是（1–263） | 非空行 210 |
| `Source/UnrealCSharp/Public/Reflection/Container/FSetHelper.h` | 53 | 是（1–53） | |
| `Source/UnrealCSharp/Private/Reflection/Container/FSetHelper.cpp` | 189 | 是（1–189） | 非空行 153 |

**结论：6 个指定文件全部逐行读完，无遗漏区间。**

### 0.2 为判定"调用方/内存所有权"而额外阅读的文件（非本报告主体，仅作证据）

| 文件 | 读到的行 | 用途 |
|---|---|---|
| `Source/UnrealCSharp/Public/Reflection/Property/FPropertyDescriptor.h` | 1–82 | 元素属性描述符接口（`Set`/`DestroyValue`/`Identical`/`GetSize`） |
| `Source/UnrealCSharp/Public/Reflection/Property/FPropertyDescriptor.inl` | 1–103 | `GetElementSize`/`GetPropertyFlags`/`GetSize` 实现 |
| `Source/UnrealCSharp/Private/Reflection/Property/FPropertyDescriptor.cpp` | 1–206 | `Factory` 支持的类型白名单；基类 `Set`/`DestroyValue` 语义 |
| `Source/UnrealCSharp/Private/Reflection/Property/StringProperty/FStrPropertyDescriptor.cpp` | 1–49 | 确认 `FString` 元素的赋值方式（非 memcpy） |
| `Source/UnrealCSharp/Private/Reflection/Property/ContainerProperty/FArrayPropertyDescriptor.cpp` | 1–62 | `FArrayHelper` 的两个主要构造点及其所有权标志 |
| `Source/UnrealCSharp/Public/Binding/Core/TPropertyValue.inl` | 795–894 | `TArray<T>` 的三个构造点 + 一处 `GetData()` 批量暴露 |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterArray.cpp` | 1–349 | C# P/Invoke 入口，用于逐函数"调用方"证据与死代码判定 |
| `Source/UnrealCSharp/Private/Reflection/Property/ContainerProperty/FMapPropertyDescriptor.cpp` | 1–62 | `FMapHelper` 的构造点与所有权标志 |
| `Source/UnrealCSharp/Private/Reflection/Property/ContainerProperty/FSetPropertyDescriptor.cpp` | 3、24、33、39、56（grep 命中行） | `FSetHelper` 的构造点与所有权标志 |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterMap.cpp` | 1–208 | `TMap` 的 P/Invoke 入口（16 个）与注册名 |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterSet.cpp` | 1–134 | `TSet` 的 P/Invoke 入口（11 个）与注册名 |
| `Script/UE/CoreUObject/TArray.cs` | 62–106、150–269 | 用于确认"托管侧是否做边界校验"（结论：不做，见 F-HLP-006/007）与这些 API 是否为公开托管 API（是，见 §6.3） |

### 0.3 引擎侧与外部依据

> UE 5.6 引擎源码位于 `Engine\Source`，已逐条打开并引用。下表是本报告全部"引擎行为"论断的**唯一依据清单**。

| # | 引擎论断 | 真实依据（文件:行） | 对本报告的意义 |
|---|---|---|---|
| E1 | **`FScriptArrayHelper::EmptyValues` 会析构元素** | `Runtime/CoreUObject/Public/UObject/UnrealType.h:4046-4058`（`if (OldNum) DestructItems(0, OldNum);`），`DestructItems` 见 `:4182-4198`（非 `CPF_IsPlainOldData\|CPF_NoDestructor` 时逐个 `InnerProperty->DestroyValue(Dest)`） | "引擎 Helper 不析构"的前提**在本处被证伪**；`FArrayHelper::Empty`（`FArrayHelper.cpp:244-249`）因此是**正确**的 |
| E2 | **`FScriptArrayHelper::RemoveValues` 会析构元素** | `UnrealType.h:4064-4070` → `DestructItems`（`:4068`） | 同上；`FArrayHelper::Reset` 的分支 A（`FArrayHelper.cpp:236`）确实是"删除并析构全部元素" |
| E3 | **`FScriptArrayHelper::MoveAssign` 会析构目标原有元素** | `UnrealType.h:4099-4105`（`// FScriptArray::MoveAssign does not call destructors for our elements, so do that before calling it.` + `DestructItems(0, Num());`） | 证伪"Helper 搬运一律不析构"的泛化说法；本报告未使用 `MoveAssign`，仅作反例记录 |
| E4 | `FScriptArray::AddZeroed` / `InsertZeroed` **只零填充，不构造元素** | `Runtime/Core/Public/Containers/ScriptArray.h:93-98`（`Add()` + `FMemory::Memzero`）、`:50-54`（`Insert()` + `FMemory::Memzero`） | **确认 F-HLP-005** |
| E5 | `FScriptArrayHelper::AddUninitializedValues` 只扩容量 | `UnrealType.h:4015-4020`（只有 `Array->Add(Count,...)`，无 `ConstructItems`） | **确认 F-HLP-005** |
| E6 | `FScriptArrayHelper::Resize` 会构造/析构 | `UnrealType.h:3974-3990`：增长 → `AddValues`（`:3996-4001` → `ConstructItems(OldNum, Count)`）；缩小 → `RemoveValues`（`:3988`，即 E2） | **确认** `FArrayHelper::SetNum`（`FArrayHelper.cpp:251-261`）是三个"改数量"实现里唯一正确的 |
| E7 | `FScriptArray` **无虚析构**，且 `delete` 不经元素析构 | `ScriptArray.h:322-357`（`class FScriptArray : public TScriptArray<FHeapAllocator>`，只有 `= default` 构造与 `check(false)` 拷贝，**未声明析构**）；`Runtime/Core/Public/Containers/Array.h:965-972`（`~TArray()` 只 `DestructItems(GetData(), ArrayNum)`，**不释放 Data**） | **确认 F-HLP-001/019/026 的"元素析构被跳过"**；同时说明"容器自身缓冲"是**由 allocator 基类析构释放**的（见 E8） |
| E8 | 容器自身的缓冲由 allocator 基类析构释放 | `Runtime/Core/Public/Containers/ContainerAllocationPolicies.h:683-693`（`TSizedHeapAllocator::ForAnyElementType::~ForAnyElementType(){ if(Data) BaseMallocType::Free(Data); }`）、`:813`（`class ForElementType : public ForAnyElementType`） | **推翻** F-HLP-019/026 原文"整表/整集存储与哈希索引全部泄漏"的表述（详见该两条的复核证据） |
| E9 | `TMap`/`TSet` 与 `FScriptMap`/`FScriptSet` **布局等价由引擎 static_assert 保证** | `Runtime/Core/Public/Containers/Set.h:2072-2090`（`static_assert(sizeof(TScriptSet)==sizeof(TSet), ...)` + 逐成员 size/offset 断言）、`:2076-2083`；`Containers/Map.h:2079-2093`（`sizeof(TScriptMap)==sizeof(TMap<int32,int8>)`） | 升级 F-HLP-001/019/026 的置信度（不再是"实现巧合"） |
| E10 | `FScriptSet::Empty` **不析构元素**（只清空存储 + 清哈希） | `Containers/Set.h:1843-1865`（`Elements.Empty(...)` + 重置 `Hash`），`Set.h:1867-1885` 是 `RemoveAt`；`FScriptMap::Empty` 转调 `Pairs.Empty`：`Containers/Map.h:1955-1958` | **确认 F-HLP-020/025** |
| E11 | `FScriptSet::RemoveAt` **自己会从哈希链摘除该槽位** | `Containers/Set.h:1867-1885`（`:1873-1881` 沿 `HashNextId` 链找到并 `*NextElementId = GetHashNextIdRef(ElementBeingRemoved, Layout);`，之后才 `Elements.RemoveAtUninitialized`） | **证伪 F-HLP-029**，并证伪 F-HLP-022 的"陈旧哈希索引"一半 |
| E12 | `FScriptSet::AddNewElement` 只在**哈希桶不足**时才 `Rehash`，否则只挂链 | `Containers/Set.h:2022-2047`（`:2030-2044`） | **确认 F-HLP-028** 的性能论断（插件每次插入都全表 `Rehash`） |
| E13 | 引擎**提供**哈希驱动的 `TMap`/`TSet` 查找，且 `FScriptMapHelper`/`FScriptSetHelper` 都有 `Rehash()` | `Containers/Map.h:1982-2001`（`FindPairIndex`）、`:2004-2014`（`FindValue`）；`Containers/Set.h:1944-1984`（`FindIndexImpl`/`FindIndex`/`FindIndexByHash`）；`UnrealType.h:4787`、`:5597`（`COREUOBJECT_API void Rehash();`） | **确认 F-HLP-021/028 的可行性论断**：不用哈希是**插件的实现选择，而不是引擎限制**（交叉印证 `08-专项审计/05` 的 F-PERF-007/008） |
| E14 | `FScriptArrayHelper::GetRawPtr` 与 `TScriptArray::Remove/Insert` 的校验级别 | `UnrealType.h:3911-3920`（`checkSlow(!Index)` / `checkSlow(IsValidIndex(Index))`，**无 `check`**）、`ScriptArray.h:191-222`（`Remove` 内全部是 `checkSlow`）、`:55-76`（`Insert` 内是 `check`，非 Slow） | **确认 F-HLP-007**：越界在 **Shipping 下没有任何引擎兜底**（Dev/Debug 下也仅 `GetRawPtr` 的 `checkSlow`） |
| E15 | `TArray::Reset(NewSize)` 的语义是"清空 + 预留容量"，**不是**"设成 N 个元素" | `Array.h:2259`（`void Reset(SizeType NewSize = 0)`）、`ScriptArray.h:149-161`（`TScriptArray::Reset`：`NewSize <= ArrayMax` → `ArrayNum = 0`；否则 `Empty(NewSize,...)`） | **证伪 F-HLP-009 的前提** |
| E16 | 已裁决的 `Set` 不析构族 | `UnrealType.h:1434`（`InitializeValue` 的注释 "assumes over uninitialized memory"） | 支撑 F-HLP-014 维持 P1 |
| E17 | 分配器匹配性 | UBT 每模块 `PerModuleInline.gen.cpp` → `HAL/PerModuleInline.inl` 展开 `REPLACEMENT_OPERATOR_NEW_AND_DELETE` | 与本报告相关：`new` + `FMemory::Free` 的"内存损坏"已**证伪**，真缺陷是 `FMemory::Free` **跳过析构**（泄漏族），本报告 `NewRef`/`new FScriptXxx()` 路径不涉及 `FMemory::Free` |

**已裁决（补录）**：`FScriptArrayHelper::ConstructItems` 的函数体**已裁决**：`Runtime/CoreUObject/Public/UObject/UnrealType.h:4155-4175` —— `:4164` 判 `CPF_ZeroConstructor` → `:4166` `Memzero`；**否则 `:4168-4174` 逐元素 `InitializeValue`**。**但该裁定的覆盖范围只到 `AddValues`（`:3998-3999`）与 `InsertValues`（`:4040`）**；而本报告 `F-HLP-005`/`F-HLP-014` 依赖的是 `AddUninitializedValues`（`:4015-4021`，**无** `ConstructItems` 调用）与单数 `AddUninitializedValue()`（`:4026-4029`）⇒ **该槽位确实未构造，两条发现仍然成立**。这是**正面确认**（G1.6 的裁定**并未**推翻本报告），请勿误读为"本报告被推翻"。

**仍然无法裁决（不得猜测）**：`FScriptArrayHelper::AddZeroed` 的函数体、`FScriptMap::FindIndex` / `FScriptMapHelper::Rehash` 的函数体。本报告**没有**任何一条结论依赖它们（E6 只用 `Resize` 的 `RemoveValues` 侧 + `AddValues` 的文档注释，已验证的行是 `:3974-3990`、`:3996-4001`、`:4064-4070`）。

### 0.4 插件自身排除范围

- 插件自身的 `Source/ThirdParty/`（子模块）未纳入搜索范围（按规范排除）。
- 工作区**未修改任何插件源码**（git HEAD 仍为 `1b29b6ed`）。

---

## 1. 模块职责与架构速览

三个 Helper 是 C# 侧 `TArray<T>` / `TMap<K,V>` / `TSet<T>` 的 **native 镜像（proxy）**。C# 的 `TArray<T>` 是纯静态类，所有实例方法都是 `[LibraryImport]` 的 extern 静态方法，通过 `IManagedHandle` 句柄指回 native 侧的一个 Helper 对象：

```
C# TArray<int>.Add(value)
  → P/Invoke
  → FRegisterArray::AddImplementation(handle, value)      FRegisterArray.cpp:233
  → FCSharpEnvironment::GetEnvironment().GetContainer<FArrayHelper>(handle)   FRegisterArray.cpp:235
  → FArrayHelper::Add(void* InValue)                      FArrayHelper.cpp:263
  → FScriptArrayHelper::AddUninitializedValue()           FArrayHelper.cpp:267
  → FPropertyDescriptor::Set(InValue, rawptr)             FArrayHelper.cpp:269
  → (元素类型对应描述符) FStrPropertyDescriptor::Set(...)  FStrPropertyDescriptor.cpp:29
  → FProperty::SetPropertyValue(Dest, FString)            属性感知赋值
```

关键设计要点：

1. **元素语义的外部化**：Helper 自己不关心元素类型。元素的大小/构造/析构/比较/赋值全部委托给 `InnerPropertyDescriptor`（`FPropertyDescriptor::Factory(FProperty*)`，`FPropertyDescriptor.cpp:47`）和 UE 自带的 `FScriptArrayHelper`。所以"容器搬运"本身**没有**在插件层做字节级拷贝。
2. **生命周期三态**：构造参数 `bNeedFreeData` / `bNeedFreeProperty` 决定析构时是否释放 `ScriptArray` 与 `InnerPropertyDescriptor`（`FArrayHelper.cpp:32-49`）。这两个标志由调用方在 4 个构造点分别给出，语义并不统一（见 F-HLP-003）。
3. **句柄 ↔ Helper 映射**：`FContainerRegistry` 维护 `handle → helper`（`ArrayManagedHandle2Helper`）与 `address → handle`（`ArrayAddress2ManagedHandle`，见 `Registry/FContainerRegistry.h:46,48`），使同一 native 地址多次访问只产生一个 Helper（`NewRef` 路径），但 `NewWeakRef` 路径不做缓存。
4. **所有 Helper 公开方法都已被注册为 P/Invoke 入口**（`FRegisterArray.cpp:313-349` 的 `FClassBuilder(TEXT("TArray"))`），因此从"整个插件源码有无调用点"的角度看几乎没有死代码；死代码判定需要下沉到 **C# 侧（`Script/`）** 与 **重复实现**（如 `Swap` vs `SwapMemory`、`Contains` vs `Find`）。

---

## 2. 关键调用链

| # | 链路 | 阶段/线程 |
|---|---|---|
| 1 | `FArrayPropertyDescriptor::NewRef` (`FArrayPropertyDescriptor.cpp:31`) → `new FArrayHelper(Property->Inner, InAddress, false, false)` (`:39`) → `AddContainerReference` (`:45`) → C# 拿到句柄 | C# 首次读取某 `TArray` 属性时 |
| 2 | `FArrayPropertyDescriptor::NewWeakRef` (`:52`) → `new FArrayHelper(Property->Inner, InAddress, bIsCopy, false)` (`:56`) → `AddContainerReference` (`:59`) | C# 每次读取 `TArray` 成员/返回值（**无缓存，必新建**） |
| 3 | `TPropertyValue<TArray<T>>::Get(std::decay_t<T>*, handle)` (`TPropertyValue.inl:814`) → `new FArrayHelper(Property, InMember, false, true)` (`:829`) | C# 把 `TArray<T>` 传给 native 函数（引用语义） |
| 4 | `TPropertyValue<TArray<T>>::Get<false>(std::decay_t<T>*)` (`:839`) → `new FArrayHelper(Property, new std::decay_t<T>(*InMember), true, true)` (`:858`) | C# 传值语义（Helper 拥有这份 **新拷贝** 的 TArray） |
| 5 | `TPropertyValue<TArray<T>>::Get(handle)` (`:866`) → `SrcContainer->GetScriptArray()->GetData()` + `Num()` (`:872-873`) | C# 取回容器内容；**直接读 native 数据指针 + 元素个数** |
| 6 | `FRegisterArray::UnRegisterImplementation` (`FRegisterArray.cpp:34`) → `AsyncTask(ENamedThreads::GameThread, ...)` → `RemoveContainerReference<FArrayHelper>(handle)` (`:38`) | C# 侧对象被 GC/Dispose；**跨线程切换到 GameThread** |
| 7 | `FArrayHelper::~FArrayHelper` (`FArrayHelper.cpp:23`) → `Deinitialize` (`:32`) → `delete ScriptArray` (`:36`) / `delete InnerPropertyDescriptor` (`:45`) | 销毁 |
| 8 | `FStrPropertyDescriptor::Set` (`FStrPropertyDescriptor.cpp:29`) → `Property->InitializeValue(Dest)` (`:35`) → `Property->SetPropertyValue` (`:37`) | 元素赋值（`FArrayHelper::Add`/`Set` 的最终落点） |
| 9 | `FRegisterArray::UnRegisterImplementation` (`FRegisterArray.cpp:34`) → `RemoveContainerReference<FArrayHelper>(handle)` → `FContainerRegistry::TContainerRegistry<...>::RemoveReference` (`FContainerRegistry.inl:58`) → **`(*FoundValue)->GetAddress()`** (`:62`) → `delete *FoundValue` (`:72`) | 句柄失效路径；**这是三个 `GetAddress()` 的真实调用点**（新证据，见 F-HLP-013/F-HLP-030） |

---

## 3. 逐函数清单

> FArrayHelper 的逐函数清单见 §3.1；**FMapHelper 见 §7.1，FSetHelper 见 §7.4**（因报告追加顺序，Map/Set 的分析正文物理上位于 §7，已在 §5.0 与本节相互交叉引用）。
> FArrayHelper 的发现正文见 §5（已按 P0→P3 排序），FMapHelper/FSetHelper 的发现正文见 §7.3/§7.6。

### 3.1 FArrayHelper

| 方法签名 | 文件:行 | 做了什么 | 调用方（证据） | 结论 |
|---|---|---|---|---|
| `FArrayHelper(FProperty*, void*, bool, bool)` | `FArrayHelper.cpp:4-21` | `InnerPropertyDescriptor = FPropertyDescriptor::Factory(InProperty)`；`InData != nullptr` 则当作 `FScriptArray*`，否则 `new FScriptArray()`；**全程不判 `Factory` 是否返回 nullptr** | `FArrayPropertyDescriptor.cpp:39`、`:56`；`TPropertyValue.inl:829`、`:852`、`:858` | 有问题 → F-HLP-002 / F-HLP-003 |
| `~FArrayHelper()` | `:23-26` | 仅调用 `Deinitialize()` | 隐式；句柄销毁路径 `FRegisterArray.cpp:38`（经容器注册表 `delete`） | 无问题 |
| `Initialize()` | `:28-30` | **空函数体** | grep `Initialize` 命中 950（含 `InitializeValue` 等噪声）；精确模式 `ArrayHelper->Initialize` / `->Initialize()` 无命中 | 死代码 → F-HLP-013 |
| `Deinitialize()` | `:32-49` | `bNeedFreeData && ScriptArray` → `delete ScriptArray` 并置空；`bNeedFreeProperty && InnerPropertyDescriptor` → `DestroyProperty()` + `delete` 并置空 | 仅 `~FArrayHelper` (`:25`) | 有问题 → F-HLP-004 |
| `static Identical(const FArrayHelper*, const FArrayHelper*)` | `:51-87` | 比较 `GetTypeSize()`、`Num()`，然后逐元素 `InnerPropertyDescriptor->GetProperty()->Identical(ItemA, ItemB)` | `FRegisterArray.cpp:27` | 无功能问题；循环条件重复求值 `InA->Num()`（`:63`、`:74`）→ P3 性能 |
| `GetTypeSize()` | `:89-92` | `return InnerPropertyDescriptor->GetSize();`（未判空） | `FRegisterArray.cpp:48` | 见 F-HLP-002 |
| `GetSlack()` | `:94-97` | `return ScriptArray->GetSlack();` | `FRegisterArray.cpp:59` | 无问题 |
| `IsValidIndex(int32)` | `:99-104` | 新建 `FScriptArrayHelper` 后转调 `IsValidIndex` | `FRegisterArray.cpp:70` | 无问题 |
| `Num()` | `:106-111` | **每次调用都 `CreateHelperFormInnerProperty()`** 后再取 `Num()` | `FRegisterArray.cpp:81`；被 `IsEmpty`/`Find`/`FindLast`/`Contains`/`RemoveSingle`/`Remove`/`Identical` 内部调用 | 无功能问题；构造开销 → P3 性能 |
| `IsEmpty()` | `:113-116` | `return Num() == 0;` | `FRegisterArray.cpp:92` | 无问题 |
| `Max()` | `:118-121` | `return GetSlack() + Num();`（两次构造 helper） | `FRegisterArray.cpp:103` | 无问题（P3 开销） |
| `Get(int32)` | `:123-131` | `IsValidIndex(Index)` 通过才返回 `GetRawPtr(Index)`，否则 `nullptr` | `FRegisterArray.cpp:115`（**返回值未判空直接传给 `Descriptor->Get`**，见 `:117`） | 有问题 → F-HLP-006 |
| `Set(int32, void*)` | `:133-139` | `IsValidIndex` 通过才调 `InnerPropertyDescriptor->Set(InValue, GetRawPtr(Index))`；越界静默忽略 | `FRegisterArray.cpp:127` | **无问题（索引已校验）** |
| `Find(const void*)` | `:141-155` | 从 0 线性扫描，用 `InnerPropertyDescriptor->Identical(Item, InValue)` 比较；未找到返回 `INDEX_NONE` | `FRegisterArray.cpp:136` | 无问题（O(n)，符合 UE 语义） |
| `FindLast(const void*)` | `:157-171` | 从 `Num()-1` 反向线性扫描，同上 | `FRegisterArray.cpp:147` | 无问题（`Num()==0` 时起始 `-1`，循环条件 `>=0` 不进入，安全） |
| `Contains(const void*)` | `:173-187` | 与 `Find` 完全同构的线性扫描，只是返回 `bool` | `FRegisterArray.cpp:158` | 无功能问题；与 `Find` 重复 → P3 可读性 |
| `AddUninitialized(int32)` | `:189-194` | `ScriptArrayHelper.AddUninitializedValues(InCount)` —— **只扩容量，不构造元素** | `FRegisterArray.cpp:169`（已注册为 `TArray.AddUninitialized`） | 有问题 → F-HLP-005 |
| `InsertZeroed(int32, int32)` | `:196-199` | `ScriptArray->InsertZeroed(InIndex, InCount, GetSize(), __STDCPP_DEFAULT_NEW_ALIGNMENT__)` —— **零字节填充，不构造元素；且不校验索引** | `FRegisterArray.cpp:181` | 有问题 → F-HLP-005 / F-HLP-007 |
| `InsertDefaulted(int32, int32)` | `:201-206` | `ScriptArrayHelper.InsertValues(InIndex, InCount)`（默认构造）；**不校验索引** | `FRegisterArray.cpp:191` | 有问题 → F-HLP-007 |
| `RemoveAt(int32, int32, bool)` | `:208-228` | `GetRawPtr(InIndex)` 取基点；非 POD/非 NoDestructor 时循环 `DestroyValue(Dest)`；再 `ScriptArray->Remove(...)`；`bAllowShrinking` 时 `Shrink(...)` | `FRegisterArray.cpp:201`；被 `RemoveSingle` (`:295`)、`Remove` (`:316`) 内部调用 | 有问题 → F-HLP-007（边界）/ F-HLP-008（每次 Shrink） |
| `Reset(int32)` | `:230-242` | `if (InNewSize <= Slack + Num())` 走 `ScriptArrayHelper.RemoveValues(0, Num())`；否则 `EmptyValues(InNewSize)` | `FRegisterArray.cpp:210` | 有问题 → F-HLP-009（两分支结果都是**空数组**，与 `TArray::Reset(NewSize)` 语义不符） |
| `Empty(int32)` | `:244-249` | `ScriptArrayHelper.EmptyValues(InSlack)` | `FRegisterArray.cpp:219` | 无问题 |
| `SetNum(int32, bool)` | `:251-261` | `ScriptArrayHelper.Resize(InNewNum)` + 可选 `Shrink`（**唯一正确区分的实现**：交给 FScriptArrayHelper 处理构造/析构） | `FRegisterArray.cpp:229` | 无问题（对比 F-HLP-005） |
| `Add(void*)` | `:263-272` | `AddUninitializedValue()` 取新槽位 → `InnerPropertyDescriptor->Set(InValue, GetRawPtr(Index))` | `FRegisterArray.cpp:238`；被 `AddUnique` (`:283`) 内部调用 | **无问题（逐元素属性感知赋值，非 memcpy）** |
| `AddZeroed(int32)` | `:274-277` | `ScriptArray->AddZeroed(InCount, GetSize(), alignment)` —— **零填充，不构造元素** | `FRegisterArray.cpp:249` | 有问题 → F-HLP-005 |
| `AddUnique(void*)` | `:279-284` | `Find` 命中则返回既有索引，否则 `Add` | `FRegisterArray.cpp:260` | 无功能问题（O(n) 语义正确） |
| `RemoveSingle(const void*)` | `:286-302` | 线性扫描，命中即 `RemoveAt(Index, 1, true)` 并返回 1 | `FRegisterArray.cpp:271` | 无功能问题（继承了 F-HLP-008 的 Shrink 开销） |
| `Remove(const void*)` | `:304-322` | `while (Find != INDEX_NONE) { ++RemovedNum; RemoveAt(Index,1,true); }` | `FRegisterArray.cpp:282` | 有问题 → F-HLP-010（O(n²) + 每次都 Shrink）；`TArray<int32> x;` (`:308`) 未使用 → F-HLP-011 |
| `SwapMemory(int32, int32)` | `:324-327` | `ScriptArray->SwapMemory(A, B, GetSize())`（**原始字节交换**）；不校验索引 | `FRegisterArray.cpp:294` | 有问题 → F-HLP-007 |
| `Swap(int32, int32)` | `:329-332` | **与 `SwapMemory` 完全同体**：同样调 `ScriptArray->SwapMemory(A, B, GetSize())` | `FRegisterArray.cpp:304` | 重复 API → F-HLP-012 |
| `GetInnerPropertyDescriptor()` | `:334-337` | 返回 `InnerPropertyDescriptor` | `FRegisterArray.cpp:117`；`FArrayHelper.cpp:72` | 无问题 |
| `GetScriptArray()` | `:339-342` | 返回 `ScriptArray` | `TPropertyValue.inl:872`（`->GetScriptArray()->GetData()`） | 无问题（但见 F-HLP-001 的借用风险） |
| `GetAddress()` | `:344-347` | 返回 `ScriptArray`（**注意：返回的是 `FScriptArray*`，不是构造时传入的 `InData`**；两者仅在 `InData != nullptr` 时相等） | **`FContainerRegistry.inl:62`**（`(*FoundValue)->GetAddress()`，经 `FContainerRegistry::TContainerRegistry<FArrayHelper>` 特化 `:85-96` → `TContainerRegistryImplementation<...>::RemoveReference`）。调用点是泛型指针，所以 `grep "ArrayHelper->GetAddress"` 必然为 0 | 无功能问题（当前所有构造点都传非空 `InData`）→ 记录于 F-HLP-013 |
| `CreateHelperFormInnerProperty()`（`.h` inline） | `FArrayHelper.h:77-80` | `FScriptArrayHelper::CreateHelperFormInnerProperty(InnerPropertyDescriptor->GetProperty(), ScriptArray)` | 本 cpp 内 **17 处**（`:68`/`:70`/`:101`/`:108`/`:125`/`:135`/`:143`/`:159`/`:175`/`:191`/`:203`/`:210`/`:232`/`:246`/`:253`/`:265`/`:288`）；全插件 `CreateHelperFormInnerProperty` 命中 **19** 处（含 `.h:77`、`.h:79`） | 有问题 → F-HLP-002（无条件解引用，不判空） |

---

## 4. 非平凡元素搬运方式对照表

**核心结论：本插件在容器搬运上完全没有使用 `memcpy` / `FMemory::Memcpy` 来搬运元素。** 逐元素写入一律经由 `FPropertyDescriptor::Set(...)`，最终落到 `FProperty::SetPropertyValue` / `CopyCompleteValue` 这类**属性感知**的赋值（对 `FString` 即 `FStrPropertyDescriptor::Set`，`FStrPropertyDescriptor.cpp:29-39`）。因此不存在"`TArray<FString>` 用 memcpy 导致双重释放"这一 P0 形态。

存在风险的是**另外三类搬运**：(a) 零填充族（`AddZeroed`/`InsertZeroed`）跳过构造；(b) 原始字节交换（`SwapMemory`/`Swap`）；(c) 把 native 数据指针直接交给 C#（`TPropertyValue.inl:872`）。

| 元素类型 | 搬运方式 | 正确？ | 文件:行 |
|---|---|---|---|
| `int32`/`float`/`double` 等 POD | `FPropertyDescriptor::Set` → `FIntPropertyDescriptor::Set`（属性感知赋值，等价于直接写） | ✅ | `FArrayHelper.cpp:269`（Add）、`:137`（Set） |
| `FString` | **逐元素属性感知赋值**，非 memcpy：`FStrPropertyDescriptor::Set` → `Property->InitializeValue(Dest)` + `Property->SetPropertyValue(Dest, *SrcValue)` | ⚠️ **功能正确但语义可疑**：`InitializeValue` 作用在已被 `AddUninitializedValues` 扩出来的**未初始化**内存上，对 `FString` 属"在生内存上构造"，UE 允许；但对 `Add` 之外的重复 `Set` 同一槽位，`InitializeValue` 会覆盖已构造对象（泄漏见 F-HLP-014） | `FStrPropertyDescriptor.cpp:35-37`（被 `FArrayHelper.cpp:269`、`:137` 调用） |
| `FName` | 同上，经 `FNamePropertyDescriptor::Set` | ✅（未逐行读该描述符，置信度中） | `FArrayHelper.cpp:269` |
| `FText` | 同上，经 `FTextPropertyDescriptor::Set` | ✅（未读，置信度中）；但**零填充族对它不安全**（见下） | `FArrayHelper.cpp:269` |
| 嵌套 `TArray`（`TArray<TArray<int>>`） | 同上，经 `FArrayPropertyDescriptor::Set` → `Property->InitializeValue(Dest)` + `Property->CopyCompleteValue(Dest, SrcContainer->GetScriptArray())` | ⚠️ 功能正确，内层堆缓冲由 `CopyCompleteValue` 深拷贝，**不会泄漏**；但 `DestroyValue` 路径依赖 `RemoveAt` 的 flag 判断（F-HLP-005/F-HLP-007） | `FArrayPropertyDescriptor.cpp:26-28`（被 `FArrayHelper.cpp:269` 调用） |
| 嵌套 `TMap`/`TSet` 元素 | 同上，经 `FMapPropertyDescriptor::Set`/`FSetPropertyDescriptor::Set` | ⚠️ 同上；另见 F-HLP-001（`delete` 经 `FScriptArray*` 不跑元素析构） | `FArrayHelper.cpp:269` |
| `UObject*` 元素 | 同上，经 `FObjectPropertyDescriptor::Set` | ⚠️ **GC 保护问题**：Helper 只是指向 native `FScriptArray` 的视图，元素本身仍在原 `UPROPERTY` 容器里受 GC 追踪；但 `TPropertyValue.inl:858` 用 `new TArray<UObject*>(*InMember)` 造的**脱离 UPROPERTY 的拷贝**完全不被 GC 看到 → 悬垂（F-HLP-001） | `FArrayHelper.cpp:269`、`:36`；`TPropertyValue.inl:858` |
| **所有非 POD 元素（零填充路径）** | `AddZeroed` → `FScriptArray::AddZeroed`；`InsertZeroed` → `FScriptArray::InsertZeroed`：**只做零字节写入，不调用元素构造函数** | ❌ **不正确**（对 `FText`/`FString`(部分)/含 `TSharedPtr`/`FText` 的结构体）→ F-HLP-005 | `FArrayHelper.cpp:276`、`:198` |
| **所有元素（交换路径）** | `SwapMemory`/`Swap` → `FScriptArray::SwapMemory`：原始字节交换，不走 `FProperty::Swap` | ⚠️ 对 `FString`/`TArray`/`FMap`/`FText` 这些"位可搬移"类型实际安全，但绕过了属性层；且不校验索引 | `FArrayHelper.cpp:326`、`:331` |
| 容器整体（读回 C#） | `SrcContainer->GetScriptArray()->GetData()` + `Num()` → 直接暴露 native 数据指针 | ⚠️ 借用语义；若 C# 侧按元素逐次 P/Invoke 读，则退化为 O(n) 次跨边界调用（性能） | `TPropertyValue.inl:872-873` |

### 4.1 `TIsPODType<T>` / 模板特化区分

**本插件的三个 Helper 中不存在 `if constexpr (TIsPODType<T>::Value)` 或任何模板特化/`memcpy` 分支**：Helper 是**非模板**类，元素类型信息完全由运行期的 `FPropertyDescriptor*` + `FProperty::PropertyFlags` 承载。

唯一与"是否 POD"有关的运行期判断是 `FArrayHelper::RemoveAt` 的析构判定：

```cpp
// FArrayHelper.cpp:214-220
	if (!(InnerPropertyDescriptor->GetPropertyFlags() & (CPF_IsPlainOldData | CPF_NoDestructor)))
	{
		for (auto Index = 0; Index < InCount; ++Index, Dest += InnerPropertyDescriptor->GetElementSize())
		{
			InnerPropertyDescriptor->DestroyValue(Dest);
		}
	}
```

对这个判定的核查结论（详细见 F-HLP-005）：
- 用 `PropertyFlags` 位而不是 `TIsPODType<T>` ——**方向是正确**的，因为这是运行期反射，没有编译期 T。
- 但**判据不完整**：`CPF_IsPlainOldData | CPF_NoDestructor` 只反映 UHT 声明的标志。`FStrProperty`/`FTextProperty`/`FArrayProperty`/`FMapProperty`/`FSetProperty` 本身**都不带** `CPF_IsPlainOldData`（`FString` 不是 POD），所以这里会走 `DestroyValue` 分支 → **对 `TArray<FString>` 是正确的**。反过来说，`FStructProperty` 若被 UHT 标成 `CPF_IsPlainOldData` 而内部含 `FString`（UHT 不会这样标）也不会误判。**因此 `RemoveAt` 这条路没有"跳过析构 → FString 泄漏"的缺陷。**
- 真正的不一致在别处：同一份"是否 POD"知识没有用在 `AddZeroed`/`InsertZeroed`/`AddUninitialized` 上，那三个函数**无条件**跳过构造（F-HLP-005）。

---

## 5. 发现清单

### 5.0 发现索引（按 P0 → P3 排序，共 30 条）

> 严重度统计：**P0 × 1，P1 × 10，P2 × 9，P3 × 6，撤销（非缺陷）× 4** = 30 条。
> 各级别成员：P0 = 023；P1 = 001、006、007、014、016、019、020、024、025、026；P2 = 002、003、005、008、017、018、021、022、028；P3 = 010、011、012、013、015、030；撤销 = 004、009、027、029。
> **级别变动（相对本报告 30 条正文）**：上调 0 条、下调 0 条、维持 26 条、撤销 4 条。另 15 条级别调整（P0→P1 ×1、P1→P2 ×4、P1→P3 ×1、P2→P3 ×3、P2→P1 ×1、P3→P1 ×1、P1→撤销 ×1、P2→撤销 ×3 等）的逐条理由与证据行见各 Finding 的级别变动字段。
> P0 的判定标准（本报告采用）：**不需要"误用输入"之外的任何特殊条件，正常 C# 用法即导致进程崩溃或内存损坏**。按该标准，`array[越界]`（F-HLP-006）与 `TSet` 枚举器竞态（F-HLP-027）都**不满足** P0，只有 `map[不存在的键]`（F-HLP-023）满足。
> "正文位置"列给出该 Finding 的完整正文（现状代码/调用上下文/问题/建议/验证方式）所在小节。

| 编号 | 严重度 | 一句话结论 | 关键位置 | 正文位置 |
|---|---|---|---|---|
| F-HLP-023 | **P0** | ~~`map[不存在的键]` / `Find` / `FindKey`：返回 `nullptr` 后被解引用 → 崩（字典查找的常态用法）~~ → **已修复（`492ce5f7`；F-HLP-023）** | `FMapHelper.cpp:144`/`:171`、`FRegisterMap.cpp:91`/`:102`/`:124` | §7.3 |
| F-HLP-001 | P1 | `delete` 经 `FScriptArray*` 删真正的 `TArray<T>` → 元素析构被跳过，`TArray<FString>`/嵌套容器元素资源泄漏 | `FArrayHelper.cpp:34-39`、`TPropertyValue.inl:858` | §5 |
| F-HLP-006 | P1 | `array[越界索引]`：`Get` 返回 `nullptr` 后被属性描述符解引用 → 崩（属"误用输入"，故非 P0） | `FArrayHelper.cpp:123-131`、`FRegisterArray.cpp:115-117` | §5（下文） |
| F-HLP-007 | P1 | `RemoveAt`/`InsertZeroed`/`InsertDefaulted`/`Swap`/`SwapMemory` 无索引校验；`RemoveAt` 会析构任意内存（Shipping 无引擎兜底） | `FArrayHelper.cpp:196-228`、`:324-332` | §5 |
| F-HLP-014 | P1 | `FStrPropertyDescriptor::Set` 先 `InitializeValue` 再赋值 → 覆盖已构造元素时泄漏字符串缓冲 | `FStrPropertyDescriptor.cpp:35-37` | §5 |
| F-HLP-016 | P1 | Map 构造在 key/value 属性任一为空时**不初始化** `ScriptMapLayout` → 所有偏移是垃圾值 | `FMapHelper.cpp:21-31`、`FMapHelper.h:62` | §7.3 |
| F-HLP-019 | P1 | `delete` 经 `FScriptMap*` 删真正的 `TMap` → 元素（key/value 自有资源）析构被跳过而泄漏；容器自身缓冲**不**泄漏 | `FMapHelper.cpp:45-50`、`TPropertyValue.inl:697` | §7.3 |
| F-HLP-020 | P1 | Map `Empty` 不析构 key/value → 每次清空泄漏全部键值自有资源 | `FMapHelper.cpp:68-71` | §7.3 |
| F-HLP-024 | P1 | Set 构造：`ScriptSetLayout` 未初始化 + `Factory` 无判空 | `FSetHelper.cpp:21-27`、`FSetHelper.h:48` | §7.6 |
| F-HLP-025 | P1 | Set `Empty` 不析构元素 → 每次清空泄漏全部元素资源 | `FSetHelper.cpp:58-61` | §7.6 |
| F-HLP-026 | P1 | `delete` 经 `FScriptSet*` 删真正的 `TSet` → 元素资源泄漏；`Elements`/`Hash` 缓冲**不**泄漏 | `FSetHelper.cpp:41-46`、`TPropertyValue.inl:776` | §7.6 |
| F-HLP-002 | P2 | `CreateHelperFormInnerProperty()` 无条件解引用可为空的 `InnerPropertyDescriptor`（`Factory` 静默返回 `nullptr`） | `FArrayHelper.h:77-80`、`FArrayHelper.cpp:11` | §5 |
| F-HLP-003 | P2 | `NewRef`/`NewWeakRef` 传 `bNeedFreeProperty=false` → 每个 `FArrayHelper` 泄漏一个属性描述符 | `FArrayPropertyDescriptor.cpp:39`/`:56` | §5 |
| F-HLP-005 | P2 | `AddZeroed`/`InsertZeroed`/`AddUninitialized` 对非 POD 元素跳过构造（引擎已确认），且无门禁 → `TArray<FText>` 崩 | `FArrayHelper.cpp:189-199`、`:274-277` | §5 |
| F-HLP-008 | P2 | `Remove`/`RemoveSingle` 每删一个元素 `Shrink` 一次 + 反复从头 `Find` → O(n²) | `FArrayHelper.cpp:222-227`、`:295`、`:304-322` | §5 |
| F-HLP-017 | P2 | Map 构造对 `Factory` 返回值无判空 → 不支持的 key/value 类型即空指针解引用 | `FMapHelper.cpp:27-30` | §7.3 |
| F-HLP-018 | P2 | Map `Deinitialize` 用 `&&` 联判两个描述符 → "部分成功"时两个都不释放 | `FMapHelper.cpp:52-65` | §7.3 |
| F-HLP-021 | P2 | `TMap` 的 `Get`/`Set`/`Contains`/`Find`/`FindKey`/`Remove` 全是 O(MaxIndex) 线性扫描，完全没用哈希 | `FMapHelper.cpp:96-190` | §7.3 |
| F-HLP-022 | P2 | Map `Set` 每次插入新键都全表 `Rehash`（性能缺陷成立）；"Remove 后不 Rehash 留下陈旧哈希索引"已被引擎源码证伪 | `FMapHelper.cpp:196-209`、`:117-123` | §7.3 |
| F-HLP-028 | P2 | `TSet` 的 `Add`/`Remove`/`Contains` 线性扫描 + 每次插入全表 `Rehash` | `FSetHelper.cpp:82-111`、`:119-131` | §7.6 |
| F-HLP-010 | P3 | `Num()`/`Max()` 每次调用都构造 `FScriptArrayHelper`；`Find`/`Identical` 在循环条件里反复求 `Num()` | `FArrayHelper.cpp:106-121`、`:145`、`:63`/`:74` | §5 |
| F-HLP-011 | P3 | `Remove` 中未使用的局部变量 `TArray<int32> x;` | `FArrayHelper.cpp:308` | §5 |
| F-HLP-012 | P3 | `Swap` 与 `SwapMemory` 实现逐字相同（两个名字一套语义） | `FArrayHelper.cpp:324-332` | §5 |
| F-HLP-013 | P3 | `Initialize()` 空实现；**`GetAddress()` 并非死代码**（真实调用点 `FContainerRegistry.inl:62`） | `FArrayHelper.cpp:28-30`、`:344-347` | §5 |
| F-HLP-015 | P3 | C# 暴露面上的重复/危险 API（`AddUninitialized`、`Swap`/`SwapMemory`、`*Zeroed`） | `FRegisterArray.cpp:330-339` | §5 |
| F-HLP-030 | P3 | Set `Initialize()` 空实现、`static_cast<int32>(INDEX_NONE)` 冗余；**`GetAddress()` 并非死代码**（同 F-HLP-013） | `FSetHelper.cpp:35-37`、`:174-177`、`:80` | §7.6 |
| F-HLP-004 | 撤销（非缺陷） | 自建 `FScriptArray` 路径（`InData==nullptr`）不释放元素数据缓冲 —— **机制成立但不可达**（5 个构造点全部传非空地址），非待修缺陷 | `FArrayHelper.cpp:17-20`、`:34-39` | §5 |
| F-HLP-009 | 撤销（非缺陷） | `Reset(InNewSize)` 与 UE `TArray::Reset` 语义**一致**（引擎 `Array.h:2259` + `ScriptArray.h:149-161`），原判据被证伪 | `FArrayHelper.cpp:230-242` | §5 |
| F-HLP-027 | 撤销（非缺陷） | `TSet` 枚举器对失效索引返回 `nullptr` 后被解引用 —— 机制成立但**单线程不可达**（C# `TSet.cs:16-22` 先 `IsValidIndex` 再索引） | `FSetHelper.cpp:184-189`、`FRegisterSet.cpp:110-112` | §7.6 |
| F-HLP-029 | 撤销（非缺陷） | Set `Remove` 后不 `Rehash` —— 引擎 `RemoveAt` **自己维护哈希链**（`Set.h:1867-1885`），不 `Rehash` 是**正确**的 | `FSetHelper.cpp:140-142` | §7.6 |

**跨 Helper 的三条共同结论（比单条发现更重要）**

1. **"失败即 `nullptr`，上游不检查"是本项目容器层的系统性缺陷**：F-HLP-006（Array）、F-HLP-023（Map）、F-HLP-027（Set）是同一段模式的三份实现（三条的实现代码逐字同形，**已确证**）。其中只有 **F-HLP-023 是 P0**（`map[不存在的键]` 是常态用法，C# `TMap.cs:256-276` 的索引器 get 直接走到这里）；F-HLP-006 是 P1（越界索引属误用输入）；F-HLP-027 是 `撤销`（C# 枚举器先校验再取值，单线程不可达）。修复仍应做成一次统一改动（见 §8）。
2. **"`delete` 一个静态类型不是真实类型的容器"出现三次**：F-HLP-001/019/026，三条均为 **P1 维持**。分配点集中在 `TPropertyValue.inl:858/697/776`（`new std::decay_t<T>(*InMember)`），释放点在三个 `Deinitialize()`。**修正**：容器自身的 `Data`/`Pairs`/`Elements`/`Hash` 缓冲**会**被 allocator 基类析构释放（引擎 `ContainerAllocationPolicies.h:683-693`），因此泄漏的是**元素自有资源**（`FString` 缓冲、内层容器、`UObject*` 的引用处理时机），而不是"整块容器内存"。对 `TMap<int,int>` 这类无自有资源元素，此路径**不产生泄漏**。
3. **两个 Helper（Map/Set）"不用引擎的 Helper 类，手工实现哈希容器"**：`FArrayHelper` 用了 `FScriptArrayHelper` 因而元素析构/构造/边界都由引擎保证；`FMapHelper`/`FSetHelper` 却直接操作 `FScriptMap`/`FScriptSet`，于是手工实现里同时丢了"哈希查找"（F-HLP-021/028）、"`Empty` 时析构元素"（F-HLP-020/025）。**改用 `FScriptMapHelper`/`FScriptSetHelper` 可以一次消除 5 条待修发现（020、021、022、025、028）**；`FScriptXxxHelper::Rehash()` 确实存在（`UnrealType.h:4787`/`:5597`），且 `TScriptMap` 自带 `FindPairIndex`/`FindValue`（`Map.h:1982-2014`）——**不用哈希是插件的实现选择，不是引擎限制**（此点已用引擎源码确认）。

### P0（原 P0 组：仅 F-HLP-023 仍为 P0）

> **判定标准**：不需要"误用输入"之外的任何特殊条件，正常 C# 用法即导致进程崩溃或内存损坏。
> 本组原有三条（F-HLP-006/023/027），它们是"查询/取值的失败结果（`nullptr`）被无条件交给属性描述符 `Get` 解引用"这一**同一缺陷形态在三个容器上的三份实现**：
> - **F-HLP-023 维持 P0**（`map[不存在的键]` = 常态用法，非误用）；
> - **F-HLP-006 降为 P1**（`array[越界]` 属误用输入，不满足本组判据），正文保留在本节（编号不变）；
> - **F-HLP-027 撤销**（C# `TSet.cs:16-22` 枚举器先 `IsValidIndex` 再索引，单线程不可达），正文见 §7.6。
>
> F-HLP-023 的完整正文位于 §7.3；下面给出 F-HLP-006 的完整正文与三条的共同修复方案（修复方案对三条同样适用，因为实现形态同形）。

#### [F-HLP-006] `Get` 越界返回 `nullptr`，`FRegisterArray::GetImplementation` 不判空即传给属性描述符 → 正常 C# 索引访问即崩溃

- **类别**: 未定义行为 / 崩溃
- **严重度**: **P1****（正常用法即崩溃：`array[array.Num()]`、`foreach` 期间 native 侧改动、C# 传越界索引）
- **复核结论**: 确认 —— 代码事实、调用上下文、类别均成立；**严重度 P1**（原 P0 下调）
- **可达性**: 活跃
- **复核证据**: `FArrayHelper.cpp:123-131`（`IsValidIndex` 失败即 `return nullptr`）；`FRegisterArray.cpp:115`（`const auto Value = ArrayHelper->Get(InIndex);`）与 `:117`（`ArrayHelper->GetInnerPropertyDescriptor()->Get(Value, ...)`，**无判空**）；`FStrPropertyDescriptor.cpp:6` 证明子类 `Get` 会解引用 `Src`；引擎侧佐证 `Runtime/CoreUObject/Public/UObject/UnrealType.h:3911-3920`（`GetRawPtr` 只有 `checkSlow`，且"索引无效"时仍会返回数据区首址，给不出失败信号）
- **级别变动**: P0→P1（理由一句话：越界索引**本身就是误用输入**，不满足本报告自定的 P0 判据；机制与可达性都成立，故降级而非撤销）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:123-131`；`Source/UnrealCSharp/Private/Domain/Interop/FRegisterArray.cpp:109-119`
- **函数**: `FArrayHelper::Get(int32)` / `FRegisterArray::GetImplementation`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FArrayHelper.cpp:123-131
void* FArrayHelper::Get(const int32 Index) const
{
	if (auto ScriptArrayHelper = CreateHelperFormInnerProperty(); ScriptArrayHelper.IsValidIndex(Index))
	{
		return ScriptArrayHelper.GetRawPtr(Index);
	}
	return nullptr;          // ← 越界返回空指针
}
```
```cpp
// FRegisterArray.cpp:112-118
			if (const auto ArrayHelper = FCSharpEnvironment::GetEnvironment().GetContainer<FArrayHelper>(
				InManagedHandle))
			{
				const auto Value = ArrayHelper->Get(InIndex);
				ArrayHelper->GetInnerPropertyDescriptor()->Get(Value, reinterpret_cast<void**>(RETURN_BUFFER));
			}   //                                                       ^^^^^ 可能是 nullptr
```
`FArrayHelper::Get` 是**唯一**返回空指针的 getter，其余 getter 返回 0/`INDEX_NONE`。

**调用上下文**
C# 侧 `TArray<T>` 的索引器 `this[int InIndex] { get { ... TArray_GetImplementation(handle, InIndex, ValueBuffer); ... } }`（`Script/UE/CoreUObject/TArray.cs:72-92`）**没有前置边界校验**，也**不调用** `IsValidIndex`（`TArray.cs:63`，那是独立的公开 API）；`IsValidIndexImplementation`（`FRegisterArray.cpp:65`）确实被注册了，但 `Get` 路径不依赖它。C# 的 `T` 为值类型时用 `stackalloc byte[sizeof(T)]` 作返回缓冲（`TArray.cs:80-84`），若 native 侧不写入该缓冲，C# 会读到未写入的栈内存。

**问题**
1. 越界索引（`array[-1]`、`array[array.Num()]`，或在 C# 遍历过程中 native 侧修改了数组）会让 `Value == nullptr` 被传入 `FPropertyDescriptor::Get(void*, void**, FPropertyArgument::FMember)`。基类实现是空函数（`FPropertyDescriptor.cpp:136-138`），但**子类实现会解引用**：`FStrPropertyDescriptor::Get(..., FMember)` 把 `Src` 交给 `FCSharpEnvironment::GetEnvironment().GetStringObject<FString>(Src)`（`FStrPropertyDescriptor.cpp:6`）→ 空指针解引用崩溃；`FIntPropertyDescriptor`/`FObjectPropertyDescriptor` 等同理。
2. 也就是说"返回 `nullptr`"这一层防御**在调用链的下一跳立刻失效**，等价于没有防御，只是把崩溃点从 Helper 移到了描述符内部（更难定位、堆栈里看不到 `FArrayHelper`）。
3. 在 C# 侧这个 API 形态是"**索引器 get**"，即最常见的语法 `var x = array[i];`。因此这不是"调用者用错 API"，而是"任何一次越界索引访问 = 进程崩溃"。
4. 同形缺陷在 Map（F-HLP-023）与 Set（F-HLP-027）上重复，且 Map 的触发条件更普通：**查询一个不存在的键**（字典的常态），见 §7.3。

**建议**
1. 在 `GetImplementation` 里补判空并给出可诊断的错误（三个 `FRegister*` 的同类实现一并改）：
```cpp
	const auto Value = ArrayHelper->Get(InIndex);
	if (Value == nullptr)
	{
		UE_LOG(LogUnrealCSharp, Error, TEXT("TArray.Get: index %d out of range (Num=%d)"),
		       InIndex, ArrayHelper->Num());
		return;   // 或向 RETURN_BUFFER 写入该元素类型的默认值
	}
```
2. 更好的做法是**让"取默认值"成为可表达的语义**：给 `FPropertyDescriptor` 增加 `GetDefault(void** Dest)`（内部用 `FProperty::InitializeValue` 在一块临时/稳定缓冲上造默认值），使"越界/键不存在"能返回类型正确的默认值 —— 这正是 C# 的 `TryGetValue` / `GetValueOrDefault` 与索引器所期望的。这需要同时改 C# 侧（`TArray.cs:72-92`、`TMap`/`TSet` 对应文件）。
3. 若选择"失败即抛/断言"的语义，则必须在 native 侧显式 `checkf` 并**保证 C# 侧能感知失败**（例如把 `Get` 改成返回 `bool` + `out` 参数），而不是让 `nullptr` 静默流过。
4. C# 侧索引器应自行做 `IsValidIndex` 检查（属另一模块范围，仅建议）。

**验证方式**
- C# 用例：`var a = new TArray<int>(); a.Add(1); var v = a[5];` —— 当前应在 `FIntPropertyDescriptor::Get` 内崩溃；补判空后应得到日志与默认值。
- C# 用例（Map 常态路径）：`var m = new TMap<int,int>(); var v = m[42];`
- C# 用例（Set 遍历期修改）：`var s = new TSet<int>(); s.Add(1); s.Add(2); foreach (var x in s) { s.Remove(2); }`
- 静态复核：`grep -n "GetImplementation" -A6 Source/UnrealCSharp/Private/Domain/Interop/FRegisterArray.cpp`。

**共同修复方案（一次覆盖 F-HLP-006 / 023 / 027）**：在三个 `FRegister*` 的"取值类"实现里统一插入"失败即返回类型默认值 + 日志"的检查，并把 `Get`/`Find`/`GetEnumerator*` 的返回值契约从"可能为 `nullptr`"改为"永不为空（失败时指向稳定的默认值缓冲）"，让 `nullptr` 不再可能流到 `FPropertyDescriptor::Get`。涉及的 11 个实现点：`FRegisterArray.cpp:109-119`（`Get`）；`FRegisterMap.cpp:85-94`（`FindKey`）、`:96-105`（`Find`）、`:118-127`（`Get`）、`:161-171`（`GetEnumeratorKey`）、`:173-183`（`GetEnumeratorValue`）；`FRegisterSet.cpp:105-114`（`GetEnumerator`）；以及 `TPropertyValue.inl:872`（`GetScriptArray()->GetData()`）等直接读裸指针的路径。

### P1

#### [F-HLP-001] `delete ScriptArray` 经 `FScriptArray*` 删除 `TArray<T>` 对象，`TArray<FString>` / 嵌套容器 / `UObject*` 元素的析构被跳过 → 堆内存泄漏

- **类别**: 内存/资源泄漏 | 未定义行为
- **严重度**: **P1**
- **复核结论**: 确认 —— 机制、分配点/释放点、所有权标志、类别、严重度 P1 全部成立
- **可达性**: 活跃
- **复核证据**: `FArrayHelper.cpp:15`（`static_cast<FScriptArray*>(InData)`）、`:34-39`（`delete ScriptArray`）；`TPropertyValue.inl:857-858`（`new std::decay_t<T>(*InMember)` + `true, true`）；`grep "new FArrayHelper(" Source/` = **5** 处（`TPropertyValue.inl:829/852/858`、`FArrayPropertyDescriptor.cpp:39/56`），其中**只有 `:858` 传 `bNeedFreeData=true`**；引擎侧 `Containers/ScriptArray.h:322-357`（`FScriptArray` **未声明析构**、无虚函数）、`Containers/Array.h:965-972`（`~TArray()` 只做 `DestructItems`）、`Containers/ContainerAllocationPolicies.h:683-693`（allocator 基类析构**会** `Free(Data)`）
- **级别变动**: 无（维持 P1）；**修正一处范围表述**：容器自身的 `Data` 缓冲**会**被 allocator 基类析构释放，所以泄漏的仅是**元素自有资源**（`FString` 缓冲、内层容器、`UObject*` 的引用处理时机）。§5.0 索引行已据此改写；对 `TArray<int32>` 这类无自有资源元素，本路径**不产生泄漏**
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:34-39`（释放点）；`Source/UnrealCSharp/Public/Binding/Core/TPropertyValue.inl:858`（分配点）
- **函数**: `FArrayHelper::Deinitialize()` / `FArrayHelper::FArrayHelper(FProperty*, void*, bool, bool)`
- **置信度**: 高（插件侧代码与所有权标志均已确认；"`TArray<T>` 与 `FScriptArray` 内存布局兼容"亦已由引擎源码确认：`FScriptArray : TScriptArray<FHeapAllocator>`，见 `Containers/ScriptArray.h:322`；`TMap`/`TSet` 的等价性另有 `Set.h:2072-2090`、`Map.h:2079-2093` 的 `static_assert` 背书）

**现状（代码事实）**
```cpp
// FArrayHelper.cpp:13-21
	if (InData != nullptr)
	{
		ScriptArray = static_cast<FScriptArray*>(InData);   // ← InData 的真实类型由调用方决定
	}
	else
	{
		ScriptArray = new FScriptArray();
	}

// FArrayHelper.cpp:32-39
void FArrayHelper::Deinitialize()
{
	if (bNeedFreeData && ScriptArray != nullptr)
	{
		delete ScriptArray;        // ← 静态类型是 FScriptArray*，没有任何虚析构
		ScriptArray = nullptr;
	}
```
```cpp
// TPropertyValue.inl:856-861
		else
		{
			const auto ArrayHelper = new FArrayHelper(Property, new std::decay_t<T>(*InMember), true, true);
			//                                                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
			//                                                     真实类型是 TArray<X>，不是 FScriptArray
			FCSharpEnvironment::GetEnvironment().AddContainerReference(ArrayHelper, FoundClass, SrcManagedHandle);
		}
```

**调用上下文**
- 分配点：`Source/UnrealCSharp/Public/Binding/Core/TPropertyValue.inl:858`，`TPropertyValue<TArray<T>>::Get<false>(std::decay_t<T>* InMember)`，即 **C# 以"传值/拷贝"语义把一个 `TArray<T>` 交给 native 时**（`IsReference == false`）。这里 `new std::decay_t<T>(*InMember)` 造了一个真正的 `TArray<X>` 堆对象，并用 `void*` 传给 Helper。
- 释放弃权：`bNeedFreeData = true`，因此 `Deinitialize()`（`FArrayHelper.cpp:36`）会 `delete ScriptArray`。
- 销毁时机：句柄失效时 `FRegisterArray::UnRegisterImplementation`（`FRegisterArray.cpp:34-41`）→ `AsyncTask(ENamedThreads::GameThread, ...)` → `RemoveContainerReference<FArrayHelper>(handle)`。

**问题**
`delete` 表达式的静态类型是 `FScriptArray*`，而 `FScriptArray` 是**没有虚析构函数的聚合体**，因此：
1. `~TArray<X>()` **不会被调用**。对 `X = FString`，`TArray<FString>` 内每个已构造元素的 `FString` 堆缓冲（`TArray<TCHAR>` 的 `Data`）全部泄漏；对 `X = TArray<int>`，内层 `TArray` 的 `Data` 泄漏；对 `X = TMap<...>`/`FScriptMap` 同理会泄漏整张 map 的哈希表与元素。
2. 即使元素是 POD，`TArray<X>` 与 `FScriptArray` 的布局一致只是**实现巧合**（`{void* Data; int32 Num; int32 Max;}`），通过 `static_cast<FScriptArray*>(void*)` + `delete` 释放一个并非 `FScriptArray` 的对象，在 C++ 标准下是未定义行为（实际 ABI 上"能跑"）。
3. `1` 的后果是**确定性的泄漏**，且规模与数组长度成正比（例如一次 `TArray<FString>` 传值，泄漏 N 条字符串）。`2` 在更换 allocator（如 `TArray<X, TInlineAllocator<N>>`）时会让布局不一致，此时 `FScriptArray::GetSlack()`/`Num()`/`Data` 读取会静默取到错误地址 → 数据损坏。

**建议**
让 Helper 记住"我拥有的是一个 `FScriptArray` 还是一个完整的 `TArray`"，并用类型擦除的销毁器：
```cpp
// FArrayHelper.h
	using FDataDeleter = void(*)(void*);
	FDataDeleter  DataDeleter = nullptr;

// 构造点（TPropertyValue.inl:858）
	auto* Copy = new std::decay_t<T>(*InMember);
	const auto ArrayHelper = new FArrayHelper(Property, Copy,
	                                          [](void* P) { delete static_cast<std::decay_t<T>*>(P); },
	                                          true);

// FArrayHelper.cpp:34-39
	if (DataDeleter != nullptr && ScriptArray != nullptr)
	{
		DataDeleter(ScriptArray);
		DataDeleter = nullptr;
		ScriptArray  = nullptr;
	}
```
同时给 `FScriptArray` 路径加 `static_assert(sizeof(FScriptArray) == sizeof(TArray<X>) ...)`（在模板里可写成 `static_assert(offsetof(...))`）把"布局一致"这一假设变成编译期检查，而不是隐含约定。

**验证方式**
- 静态：`grep -n "bNeedFreeData" -r Source/` 应只命中 `FArrayHelper.cpp` + `AbstractContainerHelper`（若有）；`grep -n "new std::decay_t<T>(\*InMember)" Source/UnrealCSharp/Public/Binding/Core/TPropertyValue.inl`。
- 动态：C# 侧构造 `TArray<string>`（通过非引用参数传给 native），循环 10 万次，用 UE 的 `-llm`/`MemReport` 或 `FPlatformMemory::GetStats()` 观察 `FString` 分配量单调增长；也可在 `FString` 的 `TArray<TCHAR>` 析构处加计数断言。
- 单测：对同一 `TArray<FString>` 反复走 `TPropertyValue.inl:858` 路径并 `UnRegister`，断言进程 RSS 恒定。

#### [F-HLP-002] `InnerPropertyDescriptor` 无条件解引用：`FPropertyDescriptor::Factory` 返回 `nullptr` 时必崩

- **类别**: 未定义行为 / 崩溃
- **严重度**: **P2**
- **复核结论**: 确认 —— 机制（`Factory` 静默 `return nullptr` + 无条件解引用）、类别、严重度 P2 均成立
- **可达性**: 潜伏（需"元素属性类型落在白名单之外"或 `Property->Inner == nullptr`；5 个构造点中有 3 处（`TPropertyValue.inl:829/852/858`）的 `FProperty*` 来自 `FTypeBridge::Factory<>(GetGenericArgument())`，泛型参数不受支持时才会落空）
- **复核证据**: `FArrayHelper.cpp:11`；`FArrayHelper.h:77-80`；`FPropertyDescriptor.cpp:133`（`return nullptr;`）、`:49-131`（白名单）、`:42-45`/`:129-131`（`UE_F_OPTIONAL_PROPERTY` 等 `#if` 分支）；`FPropertyDescriptor.cpp:161-164`/`FPropertyDescriptor.inl:65-73` 证明基类各方法有安全默认实现（哨兵方案可行）；`grep "CreateHelperFormInnerProperty"` 全插件 = **19** 处（`.h:77`、`.h:79` + `FArrayHelper.cpp` 内 **17** 处）
- **级别变动**: 无（维持 P2；P1→P2 的定级依据：触发条件是"不受支持的属性类型"，属隐患而非常态崩溃路径）
- **文件**: `Source/UnrealCSharp/Public/Reflection/Container/FArrayHelper.h:77-80`；`Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:11`
- **函数**: `FArrayHelper::CreateHelperFormInnerProperty()` / `FArrayHelper::FArrayHelper(...)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FArrayHelper.cpp:11
	InnerPropertyDescriptor = FPropertyDescriptor::Factory(InProperty);   // 可能返回 nullptr

// FArrayHelper.h:77-80
	FORCEINLINE FScriptArrayHelper CreateHelperFormInnerProperty() const
	{
		return FScriptArrayHelper::CreateHelperFormInnerProperty(InnerPropertyDescriptor->GetProperty(), ScriptArray);
	}   //                                                      ^^^^^^^^^^^^^^^^^^^^^^ 无判空
```
`Factory` 的末尾是 `return nullptr;`（`FPropertyDescriptor.cpp:133`），即**遇到白名单外的属性类型就返回空**。白名单（`FPropertyDescriptor.cpp:49-131`）虽覆盖很广，但 `FStructPropertyDescriptor`/`FOptionalPropertyDescriptor` 等分支带 `#if` 条件（`:42-45`、`:129-131`），且依赖 `UEVersion.h` 的能力宏。

**调用上下文**
`CreateHelperFormInnerProperty()` 在 `FArrayHelper.cpp` 内被 **17 处**使用、加上 `.h:77`/`.h:79` 共 **19** 处（`Num`/`IsValidIndex`/`Get`/`Set`/`Find`/`FindLast`/`Contains`/`Add`/`AddUnique`/`RemoveSingle`/`AddUninitialized`/`InsertDefaulted`/`RemoveAt`/`Reset`/`Empty`/`SetNum`/`Identical`）。任何一个 C# 对 `TArray<T>` 的操作都会走到它，因此这是**必然路径**，不是边缘分支。

**问题**
1. `InProperty` 为 `nullptr`（例如 `Property->Inner` 为空、或 `FTypeBridge::Factory` 的 `GetGenericArgument()` 解析失败，见 `TPropertyValue.inl:822`/`:843`）→ `Factory(nullptr)` 中所有 `CastField<T>(nullptr)` 均返回 `nullptr` → 返回 `nullptr` → `CreateHelperFormInnerProperty` 解引用空指针 → **立即崩溃**。
2. 属性类型落在白名单外（如某些版本的 `FOptionalProperty`、`FFieldPathProperty` 未编译进 `#if`）→ 同样 `nullptr` → 崩溃。而 `FPropertyDescriptor::Factory` 的"未知类型"是**静默**返回 `nullptr`，没有任何 `ensure`/日志，崩溃点与真正原因（不支持的属性类型）相距 19 个调用点。
3. 对比：`GetTypeSize()`（`:91`）、`GetElementSize()`（`FPropertyDescriptor.inl:13`）、`GetPropertyFlags()`（`:28`）、`DestroyValue`（`FPropertyDescriptor.cpp:190`）**都做了判空**，说明作者知道 `GetProperty()` 可能为空；只有 `CreateHelperFormInnerProperty` 这个 inline 漏了。

**建议**
```cpp
// FArrayHelper.cpp:11 之后
	InnerPropertyDescriptor = FPropertyDescriptor::Factory(InProperty);
	checkf(InnerPropertyDescriptor != nullptr,
	       TEXT("FArrayHelper: unsupported inner property '%s'"),
	       InProperty ? *InProperty->GetName() : TEXT("(null)"));

// FArrayHelper.h:77-80
	FORCEINLINE FScriptArrayHelper CreateHelperFormInnerProperty() const
	{
		check(InnerPropertyDescriptor != nullptr);
		return FScriptArrayHelper::CreateHelperFormInnerProperty(InnerPropertyDescriptor->GetProperty(), ScriptArray);
	}
```
若希望 Shipping 下不崩，则改为让 Helper 持有一个"空描述符"哨兵（`FPropertyDescriptor` 基类的各方法已全部有安全默认实现：`GetProperty()` 返回 `nullptr`、`GetSize()` 返回 0，见 `FPropertyDescriptor.cpp:161-164`、`FPropertyDescriptor.inl:65-73`），并在 `Factory` 失败时用哨兵替换 + `UE_LOG(Warning)`。

**验证方式**
`grep -n "return nullptr;" Source/UnrealCSharp/Private/Reflection/Property/FPropertyDescriptor.cpp` 确认 `:133`；构造一个含未支持元素类型的 `TArray` 蓝图属性（或在 `Factory` 里加临时 `ensureAlways`），观察是否命中。

#### [F-HLP-003] 同一个 `InnerPropertyDescriptor` 的所有权标志在 5 个构造点中的 2 个取值不一致：`NewRef`/`NewWeakRef` 传 `bNeedFreeProperty=false` → 每个 `FArrayHelper` 泄漏一个属性描述符

- **类别**: 内存/资源泄漏
- **严重度**: **P2**
- **复核结论**: 部分确认（偏差：泄漏机制成立，但"4 个构造点"应为 **5 个**——多算了 `TPropertyValue.inl:852`；泄漏只发生在 `FArrayPropertyDescriptor.cpp:39`/`:56` 两处，结论本身不变）
- **可达性**: 活跃（`FArrayPropertyDescriptor::NewWeakRef`（`:52-59`）**没有** `NewRef` 那样的缓存查询，C# 每读一次 `TArray` 字段/返回值就新建一个 Helper + 一个描述符）
- **复核证据**: `FArrayPropertyDescriptor.cpp:39-40`（`false, false`）、`:56-57`（`bIsCopy, false`）——两处 `bNeedFreeProperty=false`；`FArrayHelper.cpp:41-48`（唯一的释放点受 `bNeedFreeProperty` 门控）；`FPropertyDescriptor.cpp:47-134`（每个分支都 `return new FXxxPropertyDescriptor(...)`，Helper 是唯一所有者）；`grep "new FArrayHelper(" Source/` = 5 处，其中 `TPropertyValue.inl:829/852/858` 全部 `true`
- **级别变动**: 无（维持 P2）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Property/ContainerProperty/FArrayPropertyDescriptor.cpp:39`、`:56`；`Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:11`、`:41-48`
- **函数**: `FArrayPropertyDescriptor::NewRef` / `FArrayPropertyDescriptor::NewWeakRef` / `FArrayHelper::Deinitialize`
- **置信度**: 高（分配点与"不释放"的标志取值均已读到）

**现状（代码事实）**
```cpp
// FArrayPropertyDescriptor.cpp:31-50  （NewRef）
		const auto ArrayHelper = new FArrayHelper(Property->Inner, InAddress,
		                                          false, false);   // ← bNeedFreeProperty = false
// FArrayPropertyDescriptor.cpp:52-59  （NewWeakRef）
	const auto ArrayHelper = new FArrayHelper(Property->Inner, InAddress,
	                                          bIsCopy, false);     // ← bNeedFreeProperty = false
```
```cpp
// FArrayHelper.cpp:41-48  （唯一的释放点）
	if (bNeedFreeProperty && InnerPropertyDescriptor != nullptr)
	{
		InnerPropertyDescriptor->DestroyProperty();
		delete InnerPropertyDescriptor;
		InnerPropertyDescriptor = nullptr;
	}
```
对照 `TPropertyValue.inl:829`/`:852`/`:858` 全部传 `bNeedFreeProperty = true`。

**调用上下文**
`FPropertyDescriptor::Factory` **每次调用都 `new` 一个全新的描述符对象**（`FPropertyDescriptor.cpp:47-134`，每个分支都是 `return new FXxxPropertyDescriptor(...)`），Helper 是它唯一的所有者。因此 `bNeedFreeProperty = false` 意味着这个对象**永远没有释放点**。

`NewWeakRef`（`FArrayPropertyDescriptor.cpp:52`）**没有"已存在就复用"的检查**（对比 `NewRef:33-47` 会先 `GetContainerObject<FArrayHelper>(InAddress)` 判有效）：它无条件 `Class->NewObject()` + `new FArrayHelper` + `AddContainerReference`。而它由 `Get(Src, Dest, FMember)`/`Get(Src, Dest, FReturn)`（`:5-13`）调用，即 **C# 每读一次 `TArray` 类型的字段/返回值都会新建一个 Helper 和一个描述符**。

**问题**
1. 每个 `FArrayHelper` 泄漏 `sizeof(FXxxPropertyDescriptor)`（含其 `FClassReflection*` 等成员）字节 + 一个 `new` 分配。热路径（每帧读 `TArray` 属性）下分配次数与帧数成正比。
2. 若 `FXxxPropertyDescriptor` 的构造函数里还注册了引用/创建了 `FProperty`，泄漏规模更大（`DestroyProperty()` 未被调用的副作用）。
3. 反向风险：`bNeedFreeProperty = true` 的三处（`TPropertyValue.inl:829/852/858`）传进来的 `Property` 是 `FTypeBridge::Factory<>(...)` 新建的（`:822`、`:843`），确实归 Helper 所有，标志是对的。所以**问题只出在 `FArrayPropertyDescriptor` 的两处**。
4. 安全性上不构成双重释放（`false` 只会漏，不会重复删），因此定级 **P2**（泄漏隐患）而非 P0/P1。之所以不按"泄漏一律 P1"定级：泄漏对象是**定长的小结构体**（每次一个描述符），且其是否还夹带反射创建的 `FProperty` 尚未确证（见 §9.1 U5），规模上限不确定。

**建议**
- 最小改动：`FArrayPropertyDescriptor.cpp:39`、`:56` 的 `bNeedFreeProperty` 改为 `true`（`Property->Inner` 是引擎属性，`Factory` 造出的描述符不与它共享所有权）。
- 或改成显式所有权枚举，避免两个 `bool` 参数在 5 个调用点被写错：
```cpp
enum class EDataOwner : uint8 { Borrowed, Owned };
enum class EPropertyOwner : uint8 { Borrowed, Owned };
FArrayHelper(FProperty* InProperty, void* InData, EDataOwner, EPropertyOwner);
```
- 顺带把 `NewWeakRef` 加上与 `NewRef` 相同的缓存查询，或至少在 `Class->NewObject()` 之前复用已有 Helper。

**验证方式**
- `grep -rn "new FArrayHelper(" Source/` 列出全部 5 个构造点，逐一核对第 3、4 个实参。
- 运行期：给 `FPropertyDescriptor::Factory` 加 `static FThreadSafeCounter AllocCount;`（`++AllocCount`），析构处加 `--`，在编辑器里反复读同一个 `TArray` 组件的属性，观察 `AllocCount` 是否单调增长。

#### [F-HLP-004] 自建 `FScriptArray` 路径（`InData == nullptr`）下，元素数据缓冲区永远不会被释放 —— **撤销（非缺陷）：机制成立但不可达，请从排期移除**

- **类别**: 内存/资源泄漏
- **严重度**: **撤销（非缺陷）**
- **复核结论**: 部分确认（偏差：**机制成立且已由引擎源码确证** —— `FScriptArray` 未声明析构，`delete` 不释放 `Data`；但使其成为"缺陷"的**前提"该分支会被执行"不成立**：全部 5 个构造点的实参都不为 `nullptr` → 不可达）
- **可达性**: 不可达
- **复核证据**: `FArrayHelper.cpp:13-20`（`else { ScriptArray = new FScriptArray(); }`）、`:34-39`（`delete ScriptArray`，前后无 `EmptyValues`）；`grep "new FArrayHelper(" Plugins/UnrealCSharp/Source/` = **5** 处，实参分别为 `InAddress`×2、`InMember`×2、`new std::decay_t<T>(*InMember)`×1，**无一处为 `nullptr`**；引擎侧 `Containers/ScriptArray.h:322-357`（`FScriptArray` 无析构）、`:137-148`（`TScriptArray::Empty` 才归还缓冲）、`UnrealType.h:4046-4058`（`EmptyValues` 先 `DestructItems` 再 `Array->Empty`）
- **级别变动**: 无（维持 `撤销（非缺陷）`）；**撤销理由更正**：代码事实与泄漏机制都成立（已用 `ScriptArray.h` 逐行确证），真正使其不成立的是**可达性**（无任何调用点）。故正确判定是"部分确认 + 不可达"，而不是"证伪"。建议正文中的修复示例（`Helper.EmptyValues(0)` 后再 `delete`）**依然有效**，应作为"若将来启用该分支"的前置条件保留
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:17-20`、`:34-39`
- **函数**: `FArrayHelper::FArrayHelper(...)` / `FArrayHelper::Deinitialize()`
- **置信度**: 高（插件侧逻辑与"无调用点"均已确证）；高（"`FScriptArray` 没有析构函数、`Data` 由 `Realloc`/`Empty` 管理"已由引擎源码确证：`Containers/ScriptArray.h:322-357`、`:137-148`）

**现状（代码事实）**
```cpp
// FArrayHelper.cpp:17-20
	else
	{
		ScriptArray = new FScriptArray();     // 默认构造：Data=nullptr, Num=0, Max=0
	}
```
```cpp
// FArrayHelper.cpp:34-39
	if (bNeedFreeData && ScriptArray != nullptr)
	{
		delete ScriptArray;                    // 只 free 结构体本身，不会 free ScriptArray->Data
		ScriptArray = nullptr;
	}
```
`Deinitialize()` 里**没有**任何 `ScriptArray->Empty(0, ...)` / `FScriptArrayHelper::EmptyValues(0)` 调用；`Initialize()`（`:28-30`）也是空实现。

**调用上下文**
构造点：`FArrayHelper.cpp:4` 被 5 处调用，其中 `FArrayPropertyDescriptor.cpp:39`/`:56` 与 `TPropertyValue.inl:829`/`:852`/`:858` 传的都是非空地址，**当前没有任何调用点走 `InData == nullptr` 分支**（`grep -rn "new FArrayHelper(" Source/` 确认）。因此这是一条**潜伏路径**，不是现行缺陷。

**问题**
若该分支被使用（例如未来的"从 C# 侧凭空新建一个 TArray"需求）：Helper 会自己持有 `FScriptArray`，`Add`/`SetNum` 会经由 `FScriptArray::Realloc` 分配 `Data` 缓冲；`FScriptArray` 是聚合体（无析构函数），`delete ScriptArray` 只释放 16 字节的结构体，**`Data` 缓冲整体泄漏**。而且 `bNeedFreeData` 与 `bNeedFreeProperty` 的语义在这里被"复用"成"是否是我 new 的"，与"是否需要 free 内部缓冲"混淆。

**建议**
```cpp
void FArrayHelper::Deinitialize()
{
	if (ScriptArray != nullptr)
	{
		if (bNeedFreeData)
		{
			// 释放元素并归还 Data 缓冲
			FScriptArrayHelper Helper = CreateHelperFormInnerProperty();
			Helper.EmptyValues(0);
			delete ScriptArray;
		}
		ScriptArray = nullptr;
	}
	...
}
```
（`EmptyValues` 需要非空的 `InnerPropertyDescriptor`，因此该分支必须先通过 F-HLP-002 的判空检查。）更彻底的做法是不要自己 `new FScriptArray`，而是持有一个真正的 `TArray<uint8>` 载体 + 类型擦除，避免"裸 FScriptArray"这种既不受 GC 也不知元素类型的中间态。

**验证方式**
给 `InData == nullptr` 分支加 `ensureAlwaysMsgf`，或用 `grep` 确认无调用点后直接删除该分支（更推荐：它现在是不可达代码）。若保留，写一个用例：构造 Helper（`InData=nullptr`）→ `AddZeroed(1000)` → 析构 → 用 `FMemory::Trim` 后的统计判断泄漏。

#### [F-HLP-005] `AddZeroed` / `InsertZeroed` / `AddUninitialized` 对非 POD 元素跳过构造，且无任何保护：`TArray<FText>`/`TArray<FString>` 走这些 API 会得到未初始化对象，后续赋值/析构即崩

- **类别**: 未定义行为 / 崩溃
- **严重度**: **P2**
- **复核结论**: 确认 —— 机制（零填充/仅扩容、不构造元素）、调用上下文、类别、严重度 P2 均成立；引擎侧前提已逐行确证
- **可达性**: 潜伏（三个 API 只对外部 C# 用户可见，`Script/` 库内部零调用，见 §6.2；须外部用户对**非 POD** 元素调用才触发）
- **复核证据**: `FArrayHelper.cpp:189-194`、`:196-199`、`:274-277`；`FRegisterArray.cpp:169`/`:181`/`:249`（注册）、`:330`/`:331`/`:338`（暴露名）；C# 侧无泛型约束 `TArray.cs:187`/`:190`/`:232`；**引擎侧确证"不构造元素"**：`Containers/ScriptArray.h:93-98`（`Add()` + `FMemory::Memzero`）、`:50-54`（`Insert()` + `FMemory::Memzero`）、`Runtime/CoreUObject/Public/UObject/UnrealType.h:4015-4020`（`AddUninitializedValues` 只调 `Array->Add`）；**正确对照实现**：`UnrealType.h:3974-3990`（`Resize` 增长 → `AddValues` → `ConstructItems`；缩小 → `RemoveValues` → `DestructItems`，见 `:4064-4070`、`:4182-4198`）
- **级别变动**: 无（维持 P2；P1→P2 的定级依据：这是"外部用户误用才触发"的未定义行为，不是库自身调用路径）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:189-194`（`AddUninitialized`）、`:196-199`（`InsertZeroed`）、`:274-277`（`AddZeroed`）
- **函数**: `FArrayHelper::AddUninitialized(int32)` / `FArrayHelper::InsertZeroed(int32,int32)` / `FArrayHelper::AddZeroed(int32)`
- **置信度**: 高（插件侧行为、C# 暴露面、**引擎侧"只零填充不构造"**三者均已确证：`ScriptArray.h:50-54`、`:93-98`、`UnrealType.h:4015-4020`）

**现状（代码事实）**
```cpp
// FArrayHelper.cpp:189-194
int32 FArrayHelper::AddUninitialized(const int32 InCount) const
{
	auto ScriptArrayHelper = CreateHelperFormInnerProperty();
	return ScriptArrayHelper.AddUninitializedValues(InCount);   // 只扩容量
}

// FArrayHelper.cpp:196-199
void FArrayHelper::InsertZeroed(const int32 InIndex, const int32 InCount) const
{
	ScriptArray->InsertZeroed(InIndex, InCount, InnerPropertyDescriptor->GetSize(), __STDCPP_DEFAULT_NEW_ALIGNMENT__);
}

// FArrayHelper.cpp:274-277
int32 FArrayHelper::AddZeroed(const int32 InCount) const
{
	return ScriptArray->AddZeroed(InCount, InnerPropertyDescriptor->GetSize(), __STDCPP_DEFAULT_NEW_ALIGNMENT__);
}
```
这三个函数**都没有**先判断元素是否可平凡构造；对比同文件里唯一的正确实现：

```cpp
// FArrayHelper.cpp:251-261  （SetNum 走 FScriptArrayHelper::Resize，能正确构造/析构）
void FArrayHelper::SetNum(const int32 InNewNum, const bool bAllowShrinking) const
{
	auto ScriptArrayHelper = CreateHelperFormInnerProperty();
	ScriptArrayHelper.Resize(InNewNum);
	...
}
```

**调用上下文**
三者都已注册成 C# 可见的 P/Invoke（`FRegisterArray.cpp:169`、`:181`、`:249`，并在 `:330`/`:331`/`:338` 以名字 `AddUninitialized`/`InsertZeroed`/`AddZeroed` 暴露到 `TArray` 静态类）。因此 **C# 用户可以直接调用**：`TArray<FText>` 上 `AddZeroed(1)`、`TArray<FString>` 上 `InsertZeroed(0, 4)`。

**问题**
1. 对 `FText`：`FText` 内部是一个引用计数共享体；零填充得到的是"全零的 `FText`"，其引用指针为空。之后 `RemoveAt`/`SetNum`（缩小）/`Empty` 触发 `DestroyValue` → 对空引用计数体做解引用/减计数 → 崩溃；即便不崩，任何 `Get()` 读取该元素也会崩。
2. 对 `FString`：零填充得到的 `FString`（`Data=nullptr, Num=0, Max=0`）在 UE 里恰好等价于"空字符串"，**这一项可能侥幸安全**——但这依赖 `TArray<TCHAR>` 的零值恰好是合法空数组，属于实现巧合，不是契约。
3. 对含 `FString`/`TArray`/`TSharedPtr`/`FText` 的 `USTRUCT`：零填充后调用 `RemoveAt`，`FStructPropertyDescriptor` 走 `DestroyValue` → 对零值成员做析构。`TSharedPtr`（全零）析构通常安全，但**赋值时会先析构再拷贝**，落到 `TArray::operator=` 上会读全零的 `ArrayNum`（0）→ 通常安全；一旦某个成员的"零值"不是合法状态（典型：`FText`、带自定义构造/不变量检查的 `USTRUCT`、`TObjectPtr` 取决于引擎配置）就崩。
4. `AddUninitialized` 的语义更危险：它把**完全未初始化（而非零）**的槽位暴露给 C#（C# 侧再通过 `Get(index)` 读它）。C# 侧若把返回索引处的元素取出来，读取的就是随机内存；对 `FString` 元素更可能被 `Identical`/`Set`（`FStrPropertyDescriptor::Set` 会先 `InitializeValue` 覆盖）之外的操作读到未初始化指针。
5. 这三个函数在"元素为 POD"时是完全正确且高效的——所以问题不是"用了零填充"，而是**没有把 POD 前提变成强制约束**。

**建议**
三选一（推荐 1 或 3）：
1. **运行期门禁**（最小改动，可立即止血）：
```cpp
namespace
{
	FORCEINLINE bool IsTriviallyConstructible(const FPropertyDescriptor* D)
	{
		return (D->GetPropertyFlags() & (CPF_IsPlainOldData | CPF_NoDestructor)) != 0;
	}
}
void FArrayHelper::InsertZeroed(const int32 InIndex, const int32 InCount) const
{
	checkf(IsTriviallyConstructible(InnerPropertyDescriptor),
	       TEXT("InsertZeroed is not safe for non-POD element type '%s'"),
	       *InnerPropertyDescriptor->GetName());
	...
}
```
2. 非 POD 时**自动降级**为语义等价的 `InsertDefaulted`（`ScriptArrayHelper.InsertValues`，`FArrayHelper.cpp:205`）——注意语义会从"零值"变为"默认构造值"，对 POD 二者一致，因此降级是安全的。
3. **不让 C# 看到这些 API**：从 `FRegisterArray.cpp` 的 `FClassBuilder` 里移除 `AddUninitialized`/`InsertZeroed`/`AddZeroed` 三个注册（见 F-HLP-015 的死代码/暴露面讨论），改由 C# 侧提供 `AddDefaulted`/`SetNum` 语义。

**验证方式**
- C# 侧用例：`var a = new TArray<FText>(); a.AddZeroed(1); a.Empty();`（或在 C++ 侧直接调 `FArrayHelper::AddZeroed` 后 `Empty`），在 Debug 下应命中新增的 `checkf`；移除 `checkf` 后应在 `FText` 析构/访问处崩溃。
- 对 `TArray<FString>` 做同样用例，用于确认"侥幸安全"结论（若也崩，则把本条升为 P0）。
- `grep -n "AddZeroed\|InsertZeroed\|AddUninitialized" Source/ -r` 确认没有其他内部实现会掩盖问题。

#### [F-HLP-007] 索引型修改接口（`RemoveAt`/`InsertZeroed`/`InsertDefaulted`/`Swap`/`SwapMemory`）不做 `IsValidIndex` 校验，与同文件的 `Get`/`Set` 自相矛盾

- **类别**: 未定义行为 / 崩溃
- **严重度**: **P1**
- **复核结论**: 确认 —— "插件侧无索引校验"的代码事实、调用上下文、类别、严重度 P1 全部成立；"Shipping 下无引擎兜底"已由引擎源码确证
- **可达性**: 活跃（5 个入口全部直接暴露给 C#：`FRegisterArray.cpp:175-183`/`:185-193`/`:195-203`/`:288-296`/`:298-306`；C# 侧 `index`/`count` 无任何前置校验）
- **复核证据**: `FArrayHelper.cpp:212`（`GetRawPtr(InIndex)`）、`:216-219`（`DestroyValue` 循环）、`:222`（`ScriptArray->Remove`）、`:326`/`:331`（`SwapMemory`）；对照同文件 `:125`/`:135`（`Get`/`Set` 有 `IsValidIndex`）；引擎侧 `Runtime/CoreUObject/Public/UObject/UnrealType.h:3911-3920`（`GetRawPtr` 只有 `checkSlow`，**Shipping 下被编译掉**，越界时直接返回越界地址）、`Runtime/Core/Public/Containers/ScriptArray.h:191-222`（`TScriptArray::Remove` 内部亦全部是 `checkSlow`）
- **级别变动**: 无（维持 P1；正文"最接近 P0 的形态"措辞保留，但其成因是"缺校验 + 调用方传入越界索引"，不满足本报告 P0 判据，故不上调）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:196-199`、`:201-206`、`:208-228`、`:324-327`、`:329-332`
- **函数**: `InsertZeroed` / `InsertDefaulted` / `RemoveAt` / `SwapMemory` / `Swap`
- **置信度**: 高（"插件侧无校验"已确证；越界后果亦已由引擎源码确证：`UnrealType.h:3911-3920` 与 `ScriptArray.h:191-222` 都只有 `checkSlow`，Shipping 下无断言、无边界检查）

**现状（代码事实）**
```cpp
// FArrayHelper.cpp:208-222   （RemoveAt：先取裸指针，再逐个析构）
void FArrayHelper::RemoveAt(const int32 InIndex, const int32 InCount, const bool bAllowShrinking) const
{
	auto ScriptArrayHelper = CreateHelperFormInnerProperty();
	auto Dest = ScriptArrayHelper.GetRawPtr(InIndex);        // ← 无 IsValidIndex 校验

	if (!(InnerPropertyDescriptor->GetPropertyFlags() & (CPF_IsPlainOldData | CPF_NoDestructor)))
	{
		for (auto Index = 0; Index < InCount; ++Index, Dest += InnerPropertyDescriptor->GetElementSize())
		{
			InnerPropertyDescriptor->DestroyValue(Dest);      // ← 越界后会析构任意内存
		}
	}
	ScriptArray->Remove(InIndex, InCount, InnerPropertyDescriptor->GetElementSize(), __STDCPP_DEFAULT_NEW_ALIGNMENT__);
	...
}
```
```cpp
// FArrayHelper.cpp:324-332
void FArrayHelper::SwapMemory(const int32 InFirstIndexToSwap, const int32 InSecondIndexToSwap) const
{
	ScriptArray->SwapMemory(InFirstIndexToSwap, InSecondIndexToSwap, InnerPropertyDescriptor->GetSize());
}
void FArrayHelper::Swap(const int32 InFirstIndexToSwap, const int32 InSecondIndexToSwap) const
{
	ScriptArray->SwapMemory(InFirstIndexToSwap, InSecondIndexToSwap, InnerPropertyDescriptor->GetSize());
}
```
对照**同一文件里做了校验的两个函数**：
```cpp
// FArrayHelper.cpp:125    Get:  if (ScriptArrayHelper.IsValidIndex(Index)) ...
// FArrayHelper.cpp:135    Set:  if (ScriptArrayHelper.IsValidIndex(Index)) ...
```

**调用上下文**
全部经 `FRegisterArray` 直接暴露给 C#：`RemoveAtImplementation`（`:195-203`，`InIndex`/`InCount` 都来自 C#）、`InsertZeroedImplementation`（`:175`）、`InsertDefaultedImplementation`（`:185`）、`SwapMemoryImplementation`（`:288`）、`SwapImplementation`（`:298`）。C# 侧参数是 `int index, int count`，**没有任何 C# 侧的前置校验**（`IsValidIndex` 是独立 API，由调用者自觉使用）。

**问题**
1. `RemoveAt` 是最危险的一处：`GetRawPtr(InIndex)` 在越界时（Debug/Development 下作为 `checkSlow` 只做断言）拿到越界地址，随后 `DestroyValue` 会**对任意内存调用元素析构函数**。若索引超出且溢出到相邻分配块，就是"在别人对象上执行析构"→ 释放野指针/双重释放（**这是本报告中最接近 P0 的形态**，但成因是"缺校验"而不是"memcpy"，且在 Shipping 下 UE 的 `check` 被编译掉，因此无法依赖引擎断言兜底）。负索引同理（`GetRawPtr(-1)` 指向数据头之前）。
2. `RemoveAt` 的 `InCount` 同样未校验：`InCount > Num() - InIndex` 时析构循环会写/析构越界；`InCount < 0` 时循环不执行但 `ScriptArray->Remove` 收到负值（**已确证**：引擎只有 `checkSlow(Count >= 0)`，见 `Containers/ScriptArray.h:195`，Shipping 下无任何检查）；`InCount == 0` 时引擎 `if (Count)` 直接跳过（`ScriptArray.h:193`），插件侧的 `DestroyValue` 循环也不执行，这一支是安全的。
3. `Swap`/`SwapMemory` 未校验两个索引，越界即交换数组以外的内存，静默数据损坏。
4. 与 `Get`/`Set` 的校验不一致，说明这是**遗漏**而非有意的"调用者负责"约定（否则 `Get`/`Set` 也不会校验）。

**建议**
1. 统一在 Helper 层收口，避免每个调用点重复：
```cpp
void FArrayHelper::RemoveAt(const int32 InIndex, const int32 InCount, const bool bAllowShrinking) const
{
	auto ScriptArrayHelper = CreateHelperFormInnerProperty();
	checkf(InCount >= 0 && InIndex >= 0 && InIndex + InCount <= ScriptArrayHelper.Num(),
	       TEXT("RemoveAt(%d,%d) out of range (Num=%d)"), InIndex, InCount, ScriptArrayHelper.Num());
	...
}
```
（Shipping 下若要"不崩只报错"，把 `checkf` 换成 `if (!...) { UE_LOG(Error); return; }`——`RemoveAt` 返回 `void`，直接 return 是安全的。）
2. `Swap`/`SwapMemory`：务必补 `IsValidIndex(InFirstIndexToSwap) && IsValidIndex(InSecondIndexToSwap)`。
3. `InsertZeroed`/`InsertDefaulted`：`InIndex` 的合法范围是 `[0, Num()]`（含末尾），`InCount >= 0`，需用 `0 <= InIndex && InIndex <= Num()` 而不是 `IsValidIndex`（注意这个区别，否则合法的"尾部插入"会被误拒）。
4. 把 F-HLP-005 的 POD 门禁与这里的边界检查合并成一个私有 `CheckIndexInvariants()`，减少三处重复（见 §8.2 方案 B）。

**验证方式**
C# 用例：`var a = new TArray<int>(); a.Add(1); a.RemoveAt(10, 1);`、`a.Swap(0, 99);`——当前应在 UE 的 `check` 处断言（Development）或在 Shipping 下静默损坏。加完 `checkf` 后应得到带 `Num()` 的可读断言信息。

### P2

#### [F-HLP-008] `RemoveAt(..., bAllowShrinking=true)` 每次都 `Shrink`，而 `Remove`/`RemoveSingle` 把它放进查找循环 → 删除操作退化为 O(n²) 且反复重分配

- **类别**: 性能
- **严重度**: **P2**
- **复核结论**: 确认 —— 机制（每次删除都 `Shrink` 一次 + 循环内从头 `Find`）、类别、严重度 P2 全部成立
- **可达性**: 活跃（`FRegisterArray.cpp:201` / `:271` / `:282` → `RemoveAt` / `RemoveSingle` / `Remove`）
- **复核证据**: `FArrayHelper.cpp:224-227`（`if (bAllowShrinking) ScriptArray->Shrink(...)`）、`:295`（`RemoveAt(Index, 1, true)`）、`:310`/`:318`（循环内 `Find`）；引擎侧 `Containers/ScriptArray.h:215-218`（`EAllowShrinking::Yes` → `ResizeShrink`）、`:99-107`（`Shrink` 在 `ArrayNum != ArrayMax` 时才 `ResizeTo`，因此每删一个元素都会 `Realloc`）
- **级别变动**: 无（维持 P2）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:222-227`、`:295`、`:304-322`
- **函数**: `FArrayHelper::RemoveAt` / `FArrayHelper::RemoveSingle` / `FArrayHelper::Remove`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FArrayHelper.cpp:222-227
	ScriptArray->Remove(InIndex, InCount, InnerPropertyDescriptor->GetElementSize(), __STDCPP_DEFAULT_NEW_ALIGNMENT__);

	if (bAllowShrinking)
	{
		ScriptArray->Shrink(InnerPropertyDescriptor->GetSize(), __STDCPP_DEFAULT_NEW_ALIGNMENT__);
	}
```
```cpp
// FArrayHelper.cpp:290-299   RemoveSingle
	for (auto Index = 0; Index < Num(); ++Index)
	{
		if (const auto Item = ScriptArrayHelper.GetRawPtr(Index);
			InnerPropertyDescriptor->Identical(Item, InValue))
		{
			RemoveAt(Index, 1, true);      // ← 单次删除也显式要求 Shrink
			return 1;
		}
	}
```
```cpp
// FArrayHelper.cpp:304-322   Remove
	auto Index = Find(InValue);
	while (Index != INDEX_NONE)
	{
		++RemovedNum;
		RemoveAt(Index, 1, true);          // ← 每删一个元素 Shrink 一次 + Find 从 0 重扫
		Index = Find(InValue);
	}
```

**调用上下文**
`FRegisterArray.cpp:201`（`RemoveAt`）、`:271`（`RemoveSingle`）、`:282`（`Remove`）——都由 C# 直接调用，且都是可以在 Tick 里被调用的普通 API。

**问题**
1. `Remove(InValue)` 的复杂度是 O(n²)：外层每次删除都要从头 `Find`（`FArrayHelper.cpp:310`、`:318`），而 `Find` 本身是 O(n) 的 `Identical` 扫描（`:141-155`）。删除全部 n 个元素 → O(n²) 次 `Identical` 调用，对 `FString`/`FText` 元素（每次 `Identical` 都要经 `FCSharpEnvironment::GetString` 做托管↔native 转换，见 `FStrPropertyDescriptor.cpp:45-48`）代价极高。
2. 每次 `RemoveAt(..., true)` 都调 `ScriptArray->Shrink`（`:226`），即**每删一个元素就重分配一次数据缓冲**，把一次批量删除变成 n 次 `Realloc` + n 次元素搬迁。
3. 对比 UE 自身的 `TArray::Remove` 只做一次 `RemoveAt` 批量 + 一次收缩。

**建议**
```cpp
int32 FArrayHelper::Remove(const void* InValue) const
{
	auto ScriptArrayHelper = CreateHelperFormInnerProperty();
	int32 RemovedNum = 0;
	// 反向扫描 + 原地压缩，O(n) 且只收缩一次
	for (int32 Index = Num() - 1; Index >= 0; --Index)
	{
		if (InnerPropertyDescriptor->Identical(ScriptArrayHelper.GetRawPtr(Index), InValue))
		{
			RemoveAt(Index, 1, false);
			++RemovedNum;
		}
	}
	if (RemovedNum > 0)
	{
		ScriptArray->Shrink(InnerPropertyDescriptor->GetSize(), __STDCPP_DEFAULT_NEW_ALIGNMENT__);
	}
	return RemovedNum;
}
```
注意：**反向扫描会改变"保留哪一个重复元素"的语义**（UE 的 `Remove` 是"删除全部相等的元素"，结果集合相同，顺序也相同，因为我们总是删除被匹配到的那个）。若要求与当前实现逐字节一致，可保留正向扫描但用"写指针压缩"实现（把保留元素前移再一次性 `RemoveAt` 尾部区间）。`RemoveSingle` 同理应传 `bAllowShrinking=false`（UE 的 `RemoveSingle` 默认不收缩）。

**验证方式**
构造 `TArray<FString>` 含 10 万个相同元素，调用 C# 的 `Remove(value)`，用 `UE_LOG` 计时对比改动前后；或在 `ScriptArray->Shrink` 上加计数器，断言"一次 `Remove` 调用至多收缩一次"。

#### [F-HLP-009] `Reset(InNewSize)` 的两个分支都得到"空数组"，与 `TArray::Reset(NewSize)`（设定元素个数）语义不符 —— **证伪并撤销：UE 的 `TArray::Reset` 本来就不设置元素个数**

- **类别**: Bug
- **严重度**: **撤销（非缺陷）**
- **复核结论**: 证伪 —— 原文的核心前提"`Reset(N)` 在 UE 语义里应当是把数组变成 N 个默认构造元素"**与引擎源码直接冲突**，故原判据不成立
- **可达性**: 不适用（撤销）；插件实现本身与引擎语义**一致**
- **复核证据**: **决定性反证** `Runtime/Core/Public/Containers/Array.h:2259`（`void Reset(SizeType NewSize = 0)`，其语义是"清空数组并预留 `NewSize` 的容量"，**不设置 `ArrayNum`**）与 `Runtime/Core/Public/Containers/ScriptArray.h:149-161`（`TScriptArray::Reset`：`NewSize <= ArrayMax` → `ArrayNum = 0`；否则 `Empty(NewSize,...)` —— 与插件 `FArrayHelper.cpp:234-241` 的两个分支**逐一对应**）。另：`FArrayHelper::Reset` 的两分支确实分别走 `RemoveValues`（`:236`，会析构，见 `UnrealType.h:4064-4070`）与 `EmptyValues`（`:240`，同样会析构，见 `UnrealType.h:4046-4058`），因此**没有元素泄漏**
- **级别变动**: 无（维持 `撤销（非缺陷）`）；撤销理由：**引擎源码证伪 + 与引擎语义一致**
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:230-242`
- **函数**: `FArrayHelper::Reset(int32)`
- **置信度**: 高（已对照引擎源码：`Containers/Array.h:2259` + `Containers/ScriptArray.h:149-161` 证明 UE 的 `Reset(NewSize)` 只"清空 + 预留容量"，不设置元素个数；插件两分支与之一一对应）

**现状（代码事实）**
```cpp
// FArrayHelper.cpp:230-242
void FArrayHelper::Reset(const int32 InNewSize) const
{
	auto ScriptArrayHelper = CreateHelperFormInnerProperty();

	if (InNewSize <= ScriptArray->GetSlack() + ScriptArray->Num())
	{
		ScriptArrayHelper.RemoveValues(0, ScriptArray->Num());   // ← 删掉全部元素（Num 个）
	}
	else
	{
		ScriptArrayHelper.EmptyValues(InNewSize);                // ← 全部清空，只保留 InNewSize 的 slack
	}
}
```

**调用上下文**
`FRegisterArray.cpp:205-212`（`ResetImplementation`，注册名 `Reset`，`:334`）→ C# 的 `TArray<T>.Reset(int newSize)`。

**问题**
1. 两个分支的最终 `Num()` **都是 0**：
   - 分支 A（`InNewSize <= Slack + Num`）：`RemoveValues(0, Num())` 删掉 `Num()` 个元素 → `Num() == 0`，而 `GetSlack()` 变成 `InNewSize` 附近——**看起来像是**"设成 InNewSize 个元素"，实际是"清空 + 预留"。
   - 分支 B（`InNewSize > Slack + Num`）：`EmptyValues(InNewSize)` → `Num() == 0`，slack = `InNewSize`。
2. 因此 `Reset(5)` 与 `Empty(5)`（`:244-249`）**行为完全相同** —— 但**这正是 UE 的 `TArray::Reset(NewSize)` 的行为**（`Containers/Array.h:2259`；插件两分支与 `Containers/ScriptArray.h:149-161` 的 `TScriptArray::Reset` 逐支对应），所以**不构成"语义不符"**。原文第 2 条的其余部分（"C# 用户按 UE 语义调用 `Reset(5)` 后会拿到空数组"）是对 UE 语义的**误读**：UE 用户拿到的同样是空数组。
3. 分支条件里 `InNewSize <= Slack + Num()` 是对 `Max()` 的比较（`FArrayHelper.cpp:118-121` 的 `Max()` 就是这个和），但用它做分支的目的不明确（应该是"容量够不够"），注释一行也没有。这是典型的"作者意图无法从代码推断"的地方。

**建议**
本条已撤销，**不要按原建议把 `Reset` 改成 `SetNum` 语义** —— 那反而会与 UE 的 `TArray::Reset` 不一致，并与 C# 侧 `TArray<T>.Reset(int InNewSize = 0)`（`Script/UE/CoreUObject/TArray.cs:200`）的默认参数语义冲突。**若**确实需要"设成 N 个默认构造元素"，请在 C# 侧新增一个 `SetNum`-语义的 API（`FArrayHelper::SetNum` 已经存在且正确，`FArrayHelper.cpp:251-261`），而不是改 `Reset`。

**验证方式**
C# 用例：`var a = new TArray<int>(); a.Add(1); a.Add(2); a.Reset(5); Debug.Assert(a.Num() == 5);` —— 当前实现会得到 `Num() == 0`。`grep -rn "ResetImplementation\|\"Reset\"" Source/UnrealCSharp/Private/Domain/Interop/FRegisterArray.cpp Script/` 找 C# 侧调用点确认影响面。

#### [F-HLP-010] `Num()` / `Max()` / `IsEmpty()` 每次调用都重新构造 `FScriptArrayHelper`；`Find`/`Remove` 循环里反复求 `Num()`

- **类别**: 性能
- **严重度**: **P3**
- **复核结论**: 确认 —— 机制（`Num()`/`Max()`/`IsEmpty()` 每次都构造 `FScriptArrayHelper`；`Find`/`Identical` 在循环条件里反复求 `Num()`）、类别、严重度 P3 全部成立
- **可达性**: 活跃
- **复核证据**: `FArrayHelper.cpp:108`（`Num` 内构造）、`:115`（`IsEmpty` 转调 `Num()`）、`:120`（`GetSlack() + Num()`，两次构造）、`:145`/`:161`/`:177`/`:290`（循环条件里调 `Num()`）、`:63`/`:74`（`Identical` 内两次 `InA->Num()`）；`FRegisterArray.cpp:81`/`:92`/`:103`；引擎侧 `UnrealType.h:3905-3920`/`:4015-4030` 表明 helper 是薄封装（构造只存 3 个字段），故开销是常量级
- **级别变动**: 无（维持 P3；P2→P3 的定级依据：属常量级重复构造，不是算法级退化）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:106-111`、`:118-121`、`:141-155`、`:157-171`、`:173-187`、`:290`、`:51-87`
- **函数**: `FArrayHelper::Num` / `Max` / `IsEmpty` / `Find` / `FindLast` / `Contains` / `RemoveSingle` / `Identical`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FArrayHelper.cpp:106-111
int32 FArrayHelper::Num() const
{
	const auto ScriptArrayHelper = CreateHelperFormInnerProperty();   // 每次都构造
	return ScriptArrayHelper.Num();
}
// FArrayHelper.cpp:118-121
int32 FArrayHelper::Max() const
{
	return GetSlack() + Num();                                        // 构造两次
}
// FArrayHelper.cpp:145-152
	for (auto Index = 0; Index < Num(); ++Index)                      // 每次迭代都调 Num() → 构造 helper
	{
		if (const auto Item = ScriptArrayHelper.GetRawPtr(Index);
			InnerPropertyDescriptor->Identical(Item, InValue))
```
```cpp
// FArrayHelper.cpp:63-84   Identical 也重复求值
	if (InA->Num() != InB->Num()) { return false; }
	for (auto Index = 0; Index < InA->Num(); ++Index) { ... }
```
`FScriptArrayHelper::CreateHelperFormInnerProperty` 本身在引擎里是 trivial 的（存 3 个字段），因此单次开销极小；但 `Num()` 是**每帧热路径**（C# 的 `TArray.Count` → `NumImplementation`，`FRegisterArray.cpp:76`），且 `Find`/`FindLast`/`Contains`/`RemoveSingle`/`Identical` 在循环条件里调用它，把一次 O(n) 扫描变成 n 次"构造 + 取值"。

**调用上下文**
`FRegisterArray.cpp:81`（`Num`）、`:103`（`Max`）、`:92`（`IsEmpty`）、`:136`（`Find`）、`:147`（`FindLast`）、`:158`（`Contains`）、`:271`（`RemoveSingle`）；`Num()` 还被 Helper 内部 8 处调用。

**问题**
1. 循环条件里 `Num()` 的重复求值在**多线程/GC 场景下还有正确性含义**：`Num()` 每次读的是当前 `ScriptArray->Num`，如果 native 侧在两次迭代之间改动（另一线程 `Add`），循环边界会随之变化。虽然本项目宣称 GameThread-only（未在本文件中看到 `check(IsInGameThread())`——见 §9.2 第 3 项），但把边界缓存到局部变量是零成本的正确性改善。
2. `Max()` 里 `GetSlack() + Num()` 构造了两次 `FScriptArrayHelper`，而 `GetSlack()`/`Num()` 都只需要读 `ScriptArray` 的字段。

**建议**
```cpp
int32 FArrayHelper::Num() const
{
	return ScriptArray->Num();          // FScriptArrayHelper::Num() 就是 Array->Num()，无需构造
}
```
并且把所有"循环里求 `Num()`"的模式改为：
```cpp
	const int32 Count = Num();
	for (auto Index = 0; Index < Count; ++Index) { ... }
```
`Identical`（`:51-87`）应改为缓存 `InA->Num()` 一次并把循环上界固定（同时也顺带修掉 `:63` 与 `:74` 的两次调用）。

**验证方式**
在 `CreateHelperFormInnerProperty` 上加 `static FThreadSafeCounter`，跑 C# 侧"每帧 `array.Count` + `array.Find(x)`"的用例，观察计数从 O(n) 降到 O(1)；或直接 benchmark `TArray<int>` 100 万元素的 `Find`。

#### [F-HLP-011] `FArrayHelper::Remove` 中的未使用局部变量 `TArray<int32> x;`

- **类别**: 死代码 / 可读性
- **严重度**: **P3**
- **复核结论**: 确认 —— 事实（`TArray<int32> x;` 声明后从未使用）成立；级别 P3 成立
- **可达性**: 活跃（该语句在 `Remove` 每次都执行，只是变量未被使用）
- **复核证据**: `FArrayHelper.cpp:306-310`（`auto RemovedNum = 0;` → `TArray<int32> x;` → `auto Index = Find(InValue);`）；`grep "TArray<int32> x;" Plugins/UnrealCSharp/` = **1** 处（`FArrayHelper.cpp:308`，与原文"命中数应为 1"一致）
- **级别变动**: 无（维持 P3；P2→P3 的定级依据：无功能/性能影响，仅编译告警与可读性）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:308`
- **函数**: `FArrayHelper::Remove(const void*)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FArrayHelper.cpp:304-310
int32 FArrayHelper::Remove(const void* InValue) const
{
	auto RemovedNum = 0;

	TArray<int32> x;                    // ← 从未被使用

	auto Index = Find(InValue);
```

**调用上下文**
`FRegisterArray.cpp:277-286`（注册名 `Remove`，`:340` 附近）→ C# 的 `TArray<T>.Remove(value)`。

**问题**
调试残留。除了编译告警（`-Wunused-variable` 在部分平台/工具链上会升级为错误）之外没有功能影响，但它是"该函数曾被反复改动、缺少 review"的信号——这也是我在同一函数里发现 F-HLP-008 性能问题的同一处。

**建议**
删除该行。

**验证方式**
`grep -n "TArray<int32> x;" Source/ -r`，命中数应为 1（仅此一处）。

#### [F-HLP-012] `Swap` 与 `SwapMemory` 是完全相同的实现，两个 API 名字却暗示不同语义

- **类别**: 可优化/可读性 / 重复代码
- **严重度**: **P3**
- **复核结论**: 确认 —— 两个函数实现逐字相同的事实成立；类别与级别 P3 成立
- **可达性**: 活跃（`FRegisterArray.cpp:294`（`SwapMemory`）与 `:304`（`Swap`）两个入口都可被 C# 调用）
- **复核证据**: `FArrayHelper.cpp:326` 与 `:331` 逐字相同（均为 `ScriptArray->SwapMemory(A, B, InnerPropertyDescriptor->GetSize())`）；引擎侧 `Containers/ScriptArray.h:162-169`（`SwapMemory` 即 `FMemory::Memswap` 原字节交换，不触碰元素构造/析构）
- **级别变动**: 无（维持 P3；P2→P3 的定级依据：两个名字指向同一实现，属 API 命名/可读性问题）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:324-332`
- **函数**: `FArrayHelper::SwapMemory` / `FArrayHelper::Swap`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FArrayHelper.cpp:324-332
void FArrayHelper::SwapMemory(const int32 InFirstIndexToSwap, const int32 InSecondIndexToSwap) const
{
	ScriptArray->SwapMemory(InFirstIndexToSwap, InSecondIndexToSwap, InnerPropertyDescriptor->GetSize());
}

void FArrayHelper::Swap(const int32 InFirstIndexToSwap, const int32 InSecondIndexToSwap) const
{
	ScriptArray->SwapMemory(InFirstIndexToSwap, InSecondIndexToSwap, InnerPropertyDescriptor->GetSize());   // ← 一字不差
}
```

**调用上下文**
两者都被注册（`FRegisterArray.cpp:288-306`，名字 `SwapMemory`/`Swap`；`:344` 附近），C# 侧可分别调用，但得到的行为完全一致。

**问题**
1. 名字暗示 `Swap` 是"元素级交换（走属性/`operator swap`）"、`SwapMemory` 是"字节级交换"，实现上二者都是字节级。**注释/命名与实现不一致**（规范第 3.7 条要求单独指出）。
2. 对 `FString`/`TArray`/`TMap` 这类"位可搬移"类型，字节交换在实践中安全；但既然是两个 API，就应当至少让 `Swap` 走属性语义（`FProperty::SwapValue` 或 `ElementDescriptor->GetProperty()->SwapValue(A,B)`），以覆盖"字节交换不安全"的自定义 `USTRUCT`（例如内部含指向自身成员的指针的 struct——UE 里这类 struct 确实存在，字节交换后自引用失效）。
3. 重复实现意味着修 `Swap` 的 bug 不会同时修 `SwapMemory`。

**建议**
```cpp
void FArrayHelper::Swap(const int32 A, const int32 B) const
{
	auto ScriptArrayHelper = CreateHelperFormInnerProperty();
	check(ScriptArrayHelper.IsValidIndex(A) && ScriptArrayHelper.IsValidIndex(B));
	InnerPropertyDescriptor->GetProperty()->SwapValue(
		ScriptArrayHelper.GetRawPtr(A), ScriptArrayHelper.GetRawPtr(B));   // 属性感知
}

void FArrayHelper::SwapMemory(const int32 A, const int32 B) const
{
	check(A >= 0 && B >= 0 && A < Num() && B < Num());
	ScriptArray->SwapMemory(A, B, InnerPropertyDescriptor->GetSize());
}
```
若确认插件只支持"位可搬移"的元素类型，则应**保留一个** API 并删除另一个（见 §6.2 死代码表），避免 C# 侧出现两套等价语义。

**验证方式**
`grep -n "SwapMemory\|->Swap(" Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp` 应显示两处实现体逐字相同（`:326` 与 `:331`）。

### P3

#### [F-HLP-013] `Initialize()` 空实现 / `GetAddress()` 返回 `ScriptArray` 而非构造地址

- **类别**: 死代码 / 可读性
- **严重度**: **P3**
- **复核结论**: 部分确认（偏差：`Initialize()` 是空实现且全插件无调用点——成立；但 **`GetAddress()` 不是死代码/无调用点**，其真实调用点是**泛型指针**调用 `FContainerRegistry.inl:62`；"三个 `GetAddress` 各 0 命中即死代码"的推断不成立）
- **可达性**: 活跃（`Initialize()` 虽无调用点但仍是 `UNREALCSHARP_API` 公开方法；`GetAddress()` 在**每次句柄失效**时都会被调用）
- **复核证据**: `FArrayHelper.cpp:28-30`（空体）、`:344-347`；`grep "Helper->Initialize" Plugins/UnrealCSharp/` = **0**；`grep "GetAddress"` 命中 `Registry/FContainerRegistry.inl:62`（`const auto Address = (*FoundValue)->GetAddress();`），类型来源见 `Registry/FContainerRegistry.h:19/21/23`（`TContainerValueMapping<FArrayHelper*, void*>` 等，`ManagedHandle2Value` 存的就是三个 Helper 指针）与 `FContainerRegistry.inl:85-122`（三个特化都继承 `TContainerRegistryImplementation`）；调用链：`FRegisterArray.cpp:34-41` → `RemoveContainerReference<FArrayHelper>` → `RemoveReference`（`FContainerRegistry.inl:58`）
- **级别变动**: 无（维持 P3）；**本条结论范围收缩**：仅"空 `Initialize()`"仍是缺陷面，"`GetAddress()` 无调用点"这半边**撤销（证伪）**
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:28-30`、`:344-347`
- **函数**: `FArrayHelper::Initialize()` / `FArrayHelper::GetAddress()`
- **置信度**: 高（`Initialize` 无调用点）；中（`GetAddress` 的调用点未全部核查）

**现状（代码事实）**
```cpp
// FArrayHelper.cpp:28-30
void FArrayHelper::Initialize()
{
}                                    // ← 完全空的公开方法

// FArrayHelper.cpp:344-347
void* FArrayHelper::GetAddress() const
{
	return ScriptArray;              // ← 返回的是 FScriptArray*，不是构造时的 InData
}
```

**调用上下文**
- `Initialize()`：`grep -rn "ArrayHelper->Initialize\|Helper->Initialize()" Source/ Script/` 无命中（表格中的 950 次命中全部来自 `InitializeValue`/`InitializeValue_InContainer` 等无关符号）。它是配对 `Deinitialize()` 的对称产物，但从未被调用——`~FArrayHelper`（`:25`）只调 `Deinitialize()`。
- `GetAddress()`：构造时 `InData == nullptr` 的情况下 `GetAddress() != InData`（返回自建对象），当前 5 个构造点都传非空地址，所以二者相等。

**问题**
1. `Initialize()` 是公共 API 上的空方法，会误导调用者以为它需要被调用（不放心的调用者会写 `helper->Initialize()`，得到一个静默的 no-op）。若未来在 `Initialize` 里真的放逻辑，所有现有调用点（目前为 0）反而不会调用它。
2. `GetAddress()` 的返回值语义在"自建 `FScriptArray`"路径下与构造函数参数不一致，是 F-HLP-004 那个潜伏分支的另一个表现面。

**建议**
- 删除 `Initialize()`（`.h:14`、`.cpp:28-30`）；若意图是"惰性初始化"，改为私有并明确调用点。
- `GetAddress()` 改名为 `GetScriptArray()`（已有此方法，见 `:339`）并删除，或让它 `check(bNeedFreeData == false)` 后返回原始 `InData`（需要新增成员保存 `InData`）。

**验证方式**
`grep -rn "Initialize()" Source/UnrealCSharp/Private/Reflection/Container/ Source/UnrealCSharp/Public/Reflection/Container/`；`grep -rn "GetAddress()" Source/ Script/`。

#### [F-HLP-014] `FStrPropertyDescriptor::Set` 对每个元素都先 `InitializeValue` 再赋值：`Add`/`Set` 路径下的重复初始化会泄漏元素自身资源

- **类别**: 内存/资源泄漏
- **严重度**: **P1**（在当前调用模式下不触发；`Set` 覆盖已有元素时触发）
- **复核结论**: 确认 —— 机制（在**已构造**槽位上 `InitializeValue` 而不先 `DestroyValue`）、类别、严重度 P1 成立；与工程级已裁决事实一致
- **可达性**: 活跃（`FArrayHelper.cpp:137`（`Set`）与 `:269`（`Add`）共用同一段描述符实现；`FMapHelper.cpp:218`、`FSetHelper.cpp:102` 同族）
- **复核证据**: `FStrPropertyDescriptor.cpp:33`（取 `SrcValue`）、`:35`（`Property->InitializeValue(Dest)`）、`:37`（`Property->SetPropertyValue`）；`FArrayHelper.cpp:133-139`（`Set` → 目标槽位**已构造**）与 `:263-269`（`Add` → 槽位来自 `AddUninitializedValue()`，**未构造**）；权威依据 `Runtime/CoreUObject/Public/UObject/UnrealType.h:1434`（`InitializeValue` 的语义 "assumes over uninitialized memory"，故判据是"`Dest` 是否已持有活值"）；**已裁决权威修复清单**含 `FArrayHelper.cpp:137`、`:269`、`FMapHelper.cpp:200`、`:218`、`FSetHelper.cpp:102`（同族形态另见 `FArrayPropertyDescriptor.cpp:26`、`FMapPropertyDescriptor.cpp:26`、`FSetPropertyDescriptor.cpp:26`，三处均不在已裁决清单内但形态相同，仅作线索）
- **级别变动**: 无（维持 P1；P3→P1 的定级依据：对已存在元素赋值会**每次**泄漏一个字符串缓冲，是确定性泄漏）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Property/StringProperty/FStrPropertyDescriptor.cpp:29-39`（被 `FArrayHelper.cpp:137`、`:269` 调用）
- **函数**: `FStrPropertyDescriptor::Set(void*, void*)` / `FArrayHelper::Set(int32, void*)`
- **置信度**: 高（"对已构造对象重复 `InitializeValue` 前未 `DestroyValue`"在 C++ 语义上是明确的，且有引擎注释 `UnrealType.h:1434` "assumes over uninitialized memory" 作为语义依据）

**现状（代码事实）**
```cpp
// FStrPropertyDescriptor.cpp:29-39
void FStrPropertyDescriptor::Set(void* Src, void* Dest) const
{
	const auto SrcManagedHandle = *static_cast<IManagedHandle*>(Src);

	if (const auto SrcValue = FCSharpEnvironment::GetEnvironment().GetString<FString>(SrcManagedHandle))
	{
		Property->InitializeValue(Dest);
		Property->SetPropertyValue(Dest, *SrcValue);
	}
}
```
```cpp
// FArrayHelper.cpp:133-139   Set → 对【已构造】的数组元素调用上面的 Set
void FArrayHelper::Set(const int32 Index, void* InValue) const
{
	if (auto ScriptArrayHelper = CreateHelperFormInnerProperty(); ScriptArrayHelper.IsValidIndex(Index))
	{
		InnerPropertyDescriptor->Set(InValue, ScriptArrayHelper.GetRawPtr(Index));   // 槽位已构造
	}
}
```
```cpp
// FArrayHelper.cpp:263-272   Add → 对【未初始化】的新槽位调用上面的 Set
	const auto Index = ScriptArrayHelper.AddUninitializedValue();
	InnerPropertyDescriptor->Set(InValue, ScriptArrayHelper.GetRawPtr(Index));       // 槽位未构造
```

**调用上下文**
`FRegisterArray.cpp:127`（`Set`）和 `:238`（`Add`）共用同一段 `FPropertyDescriptor::Set` 实现，但一个落在"已构造对象"上，一个落在"生内存"上。C# 侧的 `array[i] = value` 走 `Set`。

**问题**
1. `Add` 路径：`InitializeValue` 落在 `AddUninitializedValues` 扩出来的生内存上 → 语义正确（在生内存上构造）。
2. `Set` 路径：目标槽位**已经持有一个构造好的 `FString`**（可能指向已分配的 `TArray<TCHAR>` 缓冲）。`InitializeValue` 会把它重置为空（丢失原缓冲的引用）→ 旧缓冲 **泄漏**；在没有 GC 追踪该字符串缓冲的情况下（`FString` 的缓冲由 `TArray<TCHAR>` 直接 `FMemory::Malloc` 管理）这是确定性的泄漏。
3. 正确的写法是 `DestructValue(Dest)` + `InitializeValue(Dest)`，或者直接用属性感知赋值（`Property->CopyCompleteValue(Dest, &Value)`，UE 的 `CopyCompleteValue` 语义是"目标已构造"的赋值）。`FArrayPropertyDescriptor::Set`（`FArrayPropertyDescriptor.cpp:26-28`）就是 `InitializeValue` + `CopyCompleteValue` 的同一模式，有同样的问题（但那份代码不在我的 6 个文件内，仅记录线索）。
4. 由于 `FStrPropertyDescriptor::Set` 只在这一处（含 `FMapHelper`/`FSetHelper` 的对应调用）被用在"已构造槽位"上，影响面取决于 C# 侧 `array[i] = str` 的使用频率——**每次赋值泄漏一个字符串缓冲**。

**建议**
把"生内存构造"与"已构造赋值"拆成两个描述符方法：
```cpp
// FPropertyDescriptor
virtual void Set(void* Src, void* Dest) const;          // 目标已构造：CopyCompleteValue
virtual void ConstructAndSet(void* Src, void* Dest) const { Set(Src, Dest); }  // 目标未构造
```
- `FArrayHelper::Set`（`:137`）→ `Set`
- `FArrayHelper::Add`（`:269`）→ `ConstructAndSet`
- `FStrPropertyDescriptor::Set` 去掉 `InitializeValue`，`FStrPropertyDescriptor::ConstructAndSet` 保留。

**验证方式**
C# 用例：同一个 `TArray<string>` 的 `[0]` 位置反复赋值 100 万次长字符串，用 `MemReport`/`FPlatformMemory::GetStats()` 观察增长。或在 `TArray<TCHAR>` 的 `Realloc`/析构处加计数。

#### [F-HLP-015] C# 暴露面上的重复/危险 API：`Swap`vs`SwapMemory`、`Contains`vs`Find`、三个 Zeroed/Uninitialized 接口

- **类别**: 可优化/可读性 / 安全
- **严重度**: **P3**
- **复核结论**: 部分确认（偏差：暴露面事实与类别成立，但**注册条数口径需修正**——实为 **29** 条 `.Function()`（`:316-344`）；"27 个 P/Invoke 入口"是不计 `Register`/`UnRegister` 的子计数，行区间上界为 `:344`）
- **可达性**: 活跃
- **复核证据**: `FRegisterArray.cpp:313-345`：`FClassBuilder(TEXT("TArray"), NAMESPACE_LIBRARY)`（`:315`）后共 **29** 条 `.Function(...)`（`:316`–`:344`，逐条计数：Register/Identical/UnRegister/GetTypeSize/GetSlack/IsValidIndex/Num/IsEmpty/Max/Get/Set/Find/FindLast/Contains/AddUninitialized/InsertZeroed/InsertDefaulted/RemoveAt/Reset/Empty/SetNum/Add/AddZeroed/AddUnique/RemoveSingle/Remove/SwapMemory/Swap/INDEX_NONE）；其中三个危险/冗余 API 位于 `:330`（`AddUninitialized`）、`:331`（`InsertZeroed`）、`:338`（`AddZeroed`），重复对位于 `:342`（`SwapMemory`）/`:343`（`Swap`）
- **级别变动**: 无（维持 P3）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterArray.cpp:313-345`
- **函数**: `FRegisterArray::FRegisterArray()`（`FClassBuilder(TEXT("TArray"), NAMESPACE_LIBRARY)`）
- **置信度**: 高（注册列表与 C# 调用点统计均已逐行核对，见 §6.2：这 6 个 API 在 `Script/` 中只有声明、没有内部调用）

**现状（代码事实）**
```cpp
// FRegisterArray.cpp:330-339
				.Function("AddUninitialized", AddUninitializedImplementation)
				.Function("InsertZeroed", InsertZeroedImplementation)
				.Function("InsertDefaulted", InsertDefaultedImplementation)
				.Function("RemoveAt", RemoveAtImplementation)
				.Function("Reset", ResetImplementation)
				.Function("Empty", EmptyImplementation)
				.Function("SetNum", SetNumImplementation)
				.Function("Add", AddImplementation)
				.Function("AddZeroed", AddZeroedImplementation)
				.Function("AddUnique", AddUniqueImplementation)
```
`TArray` 这一个 C# 类型对外暴露了 **29 条 `.Function(...)` 注册**（`FRegisterArray.cpp:316-344`；不计 `Register`/`UnRegister` 两个生命周期入口则为 **27** 条），其中 `AddUninitialized`/`InsertZeroed`/`AddZeroed` 是 F-HLP-005 的载体，`Swap`/`SwapMemory` 是 F-HLP-012 的重复对。

**调用上下文**
这是 native→C# 的绑定注册点，在模块启动时执行（`FClassBuilder` 构造）；实际调用方在 `Script/` 的 C# 侧（C# 侧调用点统计见 §6.2）。

**问题**
1. 把 `AddUninitialized` 暴露给 C# 等于把"返回未初始化槽位索引"这一 C++ 概念交给托管代码，而 C# 侧对未初始化内存没有任何处理手段（不能 placement-new）——**该 API 在托管侧无法被正确使用**，只有"紧接着 `Set` 覆盖"这一种用法是有意义的，而那正是 `Add` 已经提供的。
2. `InsertZeroed`/`AddZeroed` 的存在理由是"比逐元素赋值快"，但它们把 F-HLP-005 的崩溃风险引入到 C# 可触达面。
3. `Swap`/`SwapMemory` 两个名字指向同一实现（F-HLP-012）。

**建议**
- 从注册列表中移除 `AddUninitialized`（`FRegisterArray.cpp:330`）。
- 保留 `AddZeroed`/`InsertZeroed` 但加 F-HLP-005 的 POD 门禁；若 C# 侧无实际使用（见 §6.2：已确认 `Script/` 内部零调用，仅作为公开 API 暴露），直接移除。
- `Swap` 与 `SwapMemory` 只保留一个。

**验证方式**
`grep -rn "\"AddUninitialized\"\|\"InsertZeroed\"\|\"AddZeroed\"\|\"Swap\"\|\"SwapMemory\"" Source/ Script/` 统计注册点与 C# 调用点，确认可安全移除。

---

## 6. 死代码清单

### 6.1 方法（C++ 侧）

grep 覆盖范围：`Source/`（7 个模块）+ `Script/`。命中数为 `Select-String -SimpleMatch` 的原始计数（含注释与无关符号匹配），因此**每行都给出可复现的精确模式**。

| 符号 | 声明位置 | 粗 grep 命中数 | 精确模式 / 有效调用点 | 判定 | 证据 |
|---|---|---|---|---|---|
| `FArrayHelper::Initialize` | `FArrayHelper.h:14` / `.cpp:28-30` | `Initialize` = 950（含 `InitializeValue` 等噪声） | `Helper->Initialize()` / `ArrayHelper->Initialize`：**0** | **死代码**（函数体还是空的） | `FArrayHelper.cpp:28-30` 空函数体；`grep -rn "ArrayHelper->Initialize\|Helper->Initialize()" Source/ Script/` → 0 |
| `FArrayHelper::Deinitialize` | `.h:16` / `.cpp:32-49` | `Deinitialize` = 101 | `->Deinitialize()` 仅 `FArrayHelper.cpp:25` | 存活（析构内部调用） | `FArrayHelper.cpp:25` |
| `FArrayHelper::SwapMemory` | `.h:67` / `.cpp:324-327` | `SwapMemory` = 12 | 注册 `FRegisterArray.cpp:294`；实现 `:326`、`:331` | 存活但**与 `Swap` 重复** | 见 F-HLP-012 |
| `FArrayHelper::Swap` | `.h:69` / `.cpp:329-332` | `SwapMemory`（含 Swap 分支）= 12 | 注册 `FRegisterArray.cpp:304` | 存活但**实现与 `SwapMemory` 相同** | `FArrayHelper.cpp:331` == `:326` |
| `FArrayHelper::InsertZeroed` | `.h:45` / `.cpp:196-199` | `InsertZeroed` = 11 | 注册 `FRegisterArray.cpp:181`、`:331`；C# 声明 `TArray.cs:190` | 存活；**库内部零调用**（仅公开 API） | §6.2 |
| `FArrayHelper::AddZeroed` | `.h:59` / `.cpp:274-277` | `AddZeroed` = 11 | 注册 `FRegisterArray.cpp:249`、`:338`；C# 声明 `TArray.cs:232` | 存活；**库内部零调用**（仅公开 API） | §6.2 |
| `FArrayHelper::AddUninitialized` | `.h:43` / `.cpp:189-194` | `AddUninitialized` = 14 | 注册 `FRegisterArray.cpp:169`、`:330` | 存活但**托管侧无法正确使用** | 见 F-HLP-015 |
| `FArrayHelper::InsertDefaulted` | `.h:47` / `.cpp:201-206` | `InsertDefaulted` = 10 | 注册 `FRegisterArray.cpp:191`、`:332` | 存活；C# 调用点未核查 | |
| `FArrayHelper::RemoveSingle` | `.h:63` / `.cpp:286-302` | `RemoveSingle` = 12 | 注册 `FRegisterArray.cpp:271` | 存活 | |
| `FArrayHelper::FindLast` | `.h:39` / `.cpp:157-171` | `FindLast` = 21 | 注册 `FRegisterArray.cpp:147` | 存活 | |
| `FArrayHelper::GetSlack` | `.h:23` / `.cpp:94-97` | `GetSlack` = 12 | 注册 `FRegisterArray.cpp:59`；被 `Max()`（`:120`）内部调用 | 存活 | |
| `FArrayHelper::GetAddress` | `.h:75` / `.cpp:344-347` | `GetAddress` = 42（含其他 Helper/其他类） | 精确核对完成：`ArrayHelper->GetAddress`/`MapHelper->GetAddress`/`SetHelper->GetAddress` **各 0 命中** | **死代码（插件内部）**；因带 `UNREALCSHARP_API`，标为"可能被第三方 native 代码使用，需确认" | §6.2 |
| `FArrayHelper::GetScriptArray` | `.h:73` / `.cpp:339-342` | `GetScriptArray` = 4 | `TPropertyValue.inl:872` | 存活 | `TPropertyValue.inl:872` |
| `FArrayHelper::GetInnerPropertyDescriptor` | `.h:71` / `.cpp:334-337` | `GetInnerPropertyDescriptor` = 4 | `FRegisterArray.cpp:117`、`FArrayHelper.cpp:72` | 存活 | |
| `FArrayHelper::CreateHelperFormInnerProperty` | `.h:77-80` | `CreateHelperFormInnerProperty` = 19 | cpp 内 19 处 | 存活（但见 F-HLP-002） | |
| `TArray<int32> x`（局部变量） | `FArrayHelper.cpp:308` | `TArray<int32> x;` = 1 | 无 | **死代码** | F-HLP-011 |

**说明（初版结论，已在 §6.2 补完并修正）**：`FArrayHelper` 的绝大多数公开方法都被 `FRegisterArray.cpp:313-349` 注册为 C# P/Invoke 入口，所以"在插件源码里没有 C++ 调用点"**不等于**死代码——它们的外部使用者是 C# 侧（`Script/`）。C# 调用点统计结果见 §6.2 / §6.3。

### 6.2 已注册但 **C# 内部零调用点** 的 API（C# 侧统计已完成）

grep 覆盖 `Script/`（`*.cs`）。统计结果（`Select-String -Pattern <符号> | Measure-Object`）：

| 符号 | C++ 注册位置 | C# 声明位置 | C# **内部调用点** | 判定 |
|---|---|---|---|---|
| `AddUninitialized` | `FRegisterArray.cpp:169`、`:330` | `Script/UE/CoreUObject/TArray.cs:187`；`Script/UE/Library/TArrayImplementation.cs:107-111` | **0** | 公开托管 API；**建议移除**（F-HLP-015：托管侧无法 placement-new，无法正确使用） |
| `InsertZeroed` | `FRegisterArray.cpp:181`、`:331` | `TArray.cs:190`；`TArrayImplementation.cs:114-118` | **0** | 公开托管 API；保留必须加 POD 门禁（F-HLP-005） |
| `AddZeroed` | `FRegisterArray.cpp:249`、`:338` | `TArray.cs:232`；`TArrayImplementation.cs:164-168` | **0** | 同上（F-HLP-005） |
| `SwapMemory` | `FRegisterArray.cpp:294` | `TArray.cs:306`；`TArrayImplementation.cs:192-197` | **0** | 与 `Swap` 实现逐字相同，二选一（F-HLP-012） |
| `Swap` | `FRegisterArray.cpp:304` | `TArray.cs:310` | **0** | 同上 |
| `InsertDefaulted` | `FRegisterArray.cpp:191`、`:332` | `TArray.cs:193`；`TArrayImplementation.cs:121-125` | **0** | **建议保留**——它是三个"插入"API 中唯一安全（走默认构造）的实现 |
| `Initialize`（三个 Helper） | 未注册 | C# 无 | **0** | **确定死代码**（函数体为空）→ F-HLP-013 / F-HLP-030 |
| `GetAddress`（三个 Helper） | 未注册 | C# 无 | **0**（仅 `grep "ArrayHelper->GetAddress"` 命中为 0，**不等于无调用点**） | **非死代码（此前误判已修正）**：真实调用点是**泛型指针**调用 **`FContainerRegistry.inl:62`**（`(*FoundValue)->GetAddress()`，经 `FContainerRegistry::TContainerRegistry<FArrayHelper>` 特化 `:85-96` → `TContainerRegistryImplementation<...>::RemoveReference`）；因是泛型调用，`grep "ArrayHelper->GetAddress"` **必然**为 0。 → F-HLP-013 / F-HLP-030 的"无调用点"半边**已撤销（证伪）**；声明带 `UNREALCSHARP_API` 的"第三方需确认"不再适用 |
| `TArray<int32> x`（局部变量） | — | — | — | 确定死代码 → F-HLP-011 |

**说明（修正 §6.1 末尾的旧结论）**：C# 调用点统计**已补完**。结论是：

1. **确定死代码（可安全删除，共 4 处）**：`FArrayHelper::Initialize`、`FMapHelper::Initialize`、`FSetHelper::Initialize`（三个空函数）、`FArrayHelper.cpp:308` 的 `TArray<int32> x;`。**修正**：三个 Helper 的 `GetAddress()` **不是死代码**——真实调用点是**泛型指针**调用 `FContainerRegistry.inl:62`（`(*FoundValue)->GetAddress()`），因是泛型调用，`grep "ArrayHelper->GetAddress"` 必然为 0；详见 F-HLP-013 / F-HLP-030 的复核结论。
2. **冗余 API（非死代码，但应合并）**：`Swap` 与 `SwapMemory`（实现相同，F-HLP-012）。
3. **公开但库内部零调用（不是死代码，但风险面在此）**：`AddUninitialized`/`InsertZeroed`/`AddZeroed`/`InsertDefaulted`/`SwapMemory`/`Swap` —— 这三个 Helper 的这些方法**只被外部 C# 用户使用**，插件库本体（`Script/`）不调用。因此它们的正确性**完全依赖外部用户按 UE 语义正确使用**：`TArray<T>` 在 C# 侧是**无泛型约束**的泛型类（`TArray.cs:187` 的 `AddUninitialized`、`:232` 的 `AddZeroed` 对任意 `T` 都可见），所以"第一次踩到 `TArray<FText>.AddZeroed()`"的人必然是外部用户（F-HLP-005）。
4. **不是死代码判据的情形**：三个 Helper 的绝大多数公开方法都被 `FRegister{Array,Map,Set}.cpp` 注册成 P/Invoke 入口（Array **29** 个、Map 16 个、Set 11 个），所以"在 C++ 源码里没有调用点"对它们**不构成**死代码证据。

### 6.3 三个 Helper 公开方法的"内部调用点 / 注册状态"汇总

| Helper | 公开方法数 | 已注册为 P/Invoke | 未注册（插件内部调用点） | 未注册且无任何调用点 |
|---|---|---|---|---|
| `FArrayHelper` | 34（含 2 个 ctor/dtor、`Initialize`/`Deinitialize`、inline `CreateHelperFormInnerProperty`） | **29**（`FRegisterArray.cpp:316-344`——实为 **29** 条 `.Function()`；不计 `Register`/`UnRegister` 的子计数，行区间上界为 `:344`） | `Deinitialize`（`~FArrayHelper` 调用）、`GetInnerPropertyDescriptor`（`FArrayHelper.cpp:72` + `FRegisterArray.cpp:117`）、`GetScriptArray`（`TPropertyValue.inl:872`）、`CreateHelperFormInnerProperty`（cpp 内 19 处）、**`GetAddress`（`FContainerRegistry.inl:62` 的泛型指针调用 —— 非死代码）** | `Initialize`（空） |
| `FMapHelper` | 19 | 16（`FRegisterMap.cpp:188-203`） | `GetScriptMap`（`FMapPropertyDescriptor.cpp:28`）、`GetKeyPropertyDescriptor`/`GetValuePropertyDescriptor`（`FRegisterMap.cpp:91`/`:102`/`:124`/`:169`/`:181`——**注意它们出现在 registered 实现内部，因此是"被间接使用"**）、**`GetAddress`（`FContainerRegistry.inl:62` 的泛型指针调用 —— 非死代码）** | `Initialize`（空） |
| `FSetHelper` | 15 | 11（`FRegisterSet.cpp:119-129`） | `GetScriptSet`（`FSetPropertyDescriptor.cpp:28`）、`GetElementPropertyDescriptor`（`FRegisterSet.cpp:112`）、**`GetAddress`（`FContainerRegistry.inl:62` 的泛型指针调用 —— 非死代码）** | `Initialize`（空） |


---

## 7. FMapHelper 与 FSetHelper 的分析正文

> 本节物理上位于 §6 之后（按"每分析完一个 Helper 立即落盘"的纪律增量追加），编号已统一为 §7。
> §7.1 = Map 逐函数清单，§7.2 = Map 元素搬运对照表，§7.3 = Map 发现正文；§7.4/§7.5/§7.6 = Set 的对应三节。

### 7.1 FMapHelper 逐函数清单

主要构造点（grep 证据：`grep -rn "new FMapHelper(" Source/`）：`FMapPropertyDescriptor.cpp:39`、`:56`（`bNeedFreeProperty = false`）；`TPropertyValue.inl:660`、`:689`（`false, true`）、`:697`（`true, true` + `new std::decay_t<T>(*InMember)` 拷贝）。

| 方法签名 | 文件:行 | 做了什么 | 调用方（证据） | 结论 |
|---|---|---|---|---|
| `FMapHelper(FProperty* InKeyProperty, FProperty* InValueProperty, void* InData, bool, bool)` | `FMapHelper.cpp:4-32` | `InData` 非空则当 `FScriptMap*`，否则 `new FScriptMap()`；**只有两个属性都非空才**建 `KeyPropertyDescriptor`/`ValuePropertyDescriptor` 并计算 `ScriptMapLayout` | `FMapPropertyDescriptor.cpp:39`、`:56`；`TPropertyValue.inl:660`、`:689`、`:697` | **有问题 → F-HLP-016（`ScriptMapLayout` 未初始化）/ F-HLP-017（`Factory` 返回空即解引用）** |
| `~FMapHelper()` | `:34-37` | 仅调用 `Deinitialize()` | 隐式 | 无问题 |
| `Initialize()` | `:39-41` | **空函数体** | 精确模式 `MapHelper->Initialize` 无命中 | 死代码 → F-HLP-013（同 FArrayHelper） |
| `Deinitialize()` | `:43-66` | 条件 `delete ScriptMap`；条件 `DestroyProperty()`+`delete` **两个**描述符。释放条件要求 key/value 描述符**都非空** | `:36` | 有问题 → F-HLP-018（不变量破坏时两者都不释放）/ F-HLP-019（与 F-HLP-001 同形的 `delete` 类型不匹配） |
| `Empty(int32)` | `:68-71` | `ScriptMap->Empty(InExpectedNumElements, ScriptMapLayout)` —— **只归还内存，不析构 key/value** | `FRegisterMap.cpp:37`（注册名 `Empty`，`:190`） | **有问题 → F-HLP-020（泄漏所有键值）** |
| `Num()` | `:73-76` | `ScriptMap->Num()` | `FRegisterMap.cpp:46` | 无问题 |
| `IsEmpty()` | `:78-81` | `Num() == 0` | `FRegisterMap.cpp:57` | 无问题 |
| `Add(void* InKey, void* InValue)` | `:83-86` | **直接转调 `Set`**（即 `TMap.Add` = 覆盖语义） | `FRegisterMap.cpp:69` | 无功能问题；`Set` 的全部缺陷（F-HLP-021/022）随之继承 |
| `Remove(const void* InKey)` | `:88-128` | `do { 线性扫 `GetMaxIndex()` 找 `Identical` 的 key → `DestroyValue(key)` + `DestroyValue(value)` + `ScriptMap->RemoveAt(Index, Layout)` } while(true)`，返回删除个数 | `FRegisterMap.cpp:79` | **有问题 → F-HLP-021（O(MaxIndex) 线性扫描）/ F-HLP-022（`RemoveAt` 后未 `Rehash`，与 `Set` 不一致）** |
| `FindKey(const void* InValue)` | `:130-145` | 线性扫，用 **value** 属性 `Identical(Data + ValueOffset, InValue)` 匹配，返回 **key** 指针 `Data`；未找到返回 `nullptr` | `FRegisterMap.cpp:91` | 有问题 → F-HLP-023（返回 `nullptr` 被 `->Get` 无条件解引用）/ F-HLP-021 |
| `Find(const void* InKey)` | `:147-150` | `return Get(InKey);` | `FRegisterMap.cpp:102` | 有问题 → F-HLP-023 / F-HLP-021 |
| `Contains(const void* InKey)` | `:152-155` | `return Get(InKey) != nullptr;` | `FRegisterMap.cpp:112` | 无功能问题（但因 `Get` 是 O(n) 而变 O(n)）→ F-HLP-021 |
| `Get(const void* InKey)` | `:157-172` | **线性扫**全表，`KeyPropertyDescriptor->Identical(Data, InKey)` 命中则返回 `Data + ValueOffset`，否则 `nullptr` | `FRegisterMap.cpp:124` | **有问题 → F-HLP-021（TMap 退化为 O(n) 线性表，完全未使用哈希）** |
| `Set(void* InKey, void* InValue)` | `:174-219` | 先**线性扫**找同 key；未找到则 `AddUninitialized(Layout)` → `KeyPropertyDescriptor->Set(key,Data)` → **`Rehash(...)`**；已存在则 `DestroyValue(Data+ValueOffset)`；最后写 value | `FRegisterMap.cpp:135`；被 `Add`（`:85`）调用 | 有问题 → F-HLP-021（线性扫）/ F-HLP-022（每次新键都全表 `Rehash`） |
| `GetKeyPropertyDescriptor()` | `:221-224` | 返回 key 描述符 | `FRegisterMap.cpp:91`、`:169` | 无问题（返回体可能为 `nullptr`，见 F-HLP-016/017） |
| `GetValuePropertyDescriptor()` | `:226-229` | 返回 value 描述符 | `FRegisterMap.cpp:102`、`:124`、`:181` | 同上 |
| `GetScriptMap()` | `:231-234` | 返回 `ScriptMap` | `FMapPropertyDescriptor.cpp:28`（`CopyCompleteValue(Dest, SrcContainer->GetScriptMap())`） | 无问题 |
| `GetAddress()` | `:236-239` | 返回 `ScriptMap` | `grep` 未命中 `MapHelper->GetAddress` | **疑似死代码** → F-HLP-013 |
| `GetMaxIndex()` | `:241-244` | `ScriptMap->GetMaxIndex()` | `FRegisterMap.cpp:144`；`TPropertyValue.inl:713` | 无问题 |
| `IsValidIndex(int32)` | `:246-249` | `ScriptMap->IsValidIndex(InIndex)` | `FRegisterMap.cpp:155`；`TPropertyValue.inl:715` | 无问题 |
| `GetEnumeratorKey(int32)` | `:251-256` | `IsValidIndex ? GetData(Index) : nullptr` | `FRegisterMap.cpp:167`；`TPropertyValue.inl:718` | 有问题 → F-HLP-023（`nullptr` 直接被 `->Get`） |
| `GetEnumeratorValue(int32)` | `:258-263` | `IsValidIndex ? GetData(Index)+ValueOffset : nullptr` | `FRegisterMap.cpp:179`；`TPropertyValue.inl:720` | 同上 |

**注册（C# 可见）**：`FRegisterMap.cpp:185-204`，`FClassBuilder(TEXT("TMap"), NAMESPACE_LIBRARY)` 共 **16 个** P/Invoke 入口。注意 `FMapHelper::GetAddress()` **没有**被注册，`Initialize()`/`Deinitialize()` 也没有。

---

### 7.2 非平凡元素搬运方式对照表（FMapHelper 部分）

| 元素类型 | 搬运方式 | 正确？ | 文件:行 |
|---|---|---|---|
| key 写入（新键） | `KeyPropertyDescriptor->Set(InKey, Data)` —— 属性感知，**非 memcpy** | ✅（隐含 F-HLP-014 的同一缺陷） | `FMapHelper.cpp:200` |
| value 写入 | `ValuePropertyDescriptor->Set(InValue, Data + ValueOffset)` —— 属性感知 | ✅ | `FMapHelper.cpp:218` |
| value 覆盖（已存在的键） | 先 `ValuePropertyDescriptor->DestroyValue(Data + ValueOffset)` 再 `Set` | ✅ **这是三个 Helper 中唯一在覆盖前显式析构的写法**，可作为 F-HLP-014 的正面参照 | `FMapHelper.cpp:215`、`:218` |
| key/value 删除 | 先 `KeyPropertyDescriptor->DestroyValue(Data)` + `ValuePropertyDescriptor->DestroyValue(Data + ValueOffset)` 再 `ScriptMap->RemoveAt` | ✅（证明 `FScriptMap::RemoveAt` 不析构 —— 这正是 F-HLP-020 的判据） | `FMapHelper.cpp:119-123` |
| **`Empty` 清空** | `ScriptMap->Empty(...)` **不析构 key/value** | ❌ **泄漏**（对比同文件 `Remove` 的手动析构）→ F-HLP-020 | `FMapHelper.cpp:70` |
| key 查找 / 比较 | `KeyPropertyDescriptor->Identical(Data, InKey)` 线性扫描（**不用 `FScriptMap` 哈希**） | ⚠️ 正确但 O(n) → F-HLP-021 | `FMapHelper.cpp:101`、`:164`、`:183` |
| 哈希 | `KeyPropertyDescriptor->GetValueTypeHash(Src)` 通过 `ScriptMap->Rehash` 求值 | ✅ 语义正确（用 `FProperty::GetValueTypeHash`） | `FMapHelper.cpp:203-209` |
| `FMap<...,UObject*>` 的 GC | Helper 只是 native `FScriptMap` 的视图，元素仍在原 `UPROPERTY` 内受 GC 追踪 | ✅（借用路径安全）；❌ 对 `TPropertyValue.inl:697` 的 `new TMap<K,V>(*InMember)` 拷贝不安全（脱离 GC） | `FMapHelper.cpp:47`；`TPropertyValue.inl:697` |

**未使用 `memcpy`**：与 `FArrayHelper` 一致，`FMapHelper.cpp` 全文没有 `memcpy`/`FMemory::Memcpy`/`FMemory::Malloc`，唯一的字节级操作是 `ScriptMapLayout.ValueOffset` 的指针算术（`:121`、`:137`、`:166`、`:215`、`:218`、`:261`）。因此"`FString` key 用 memcpy → 双重释放"这一形态**不存在**。

**`FName`/`FString` 作为 key 的哈希与拷贝语义**：
- 哈希：`ScriptMap->Rehash` 的回调用 `KeyPropertyDescriptor->GetValueTypeHash(Src)`（`:208`），即 `FProperty::GetValueTypeHash` → `FNameProperty::GetValueTypeHash` 返回 `GetTypeHash(FName)`、`FStrProperty::GetValueTypeHash` 返回 `GetTypeHash(FString)`（`FString` 内部有缓存的 `Hash` 字段，首次计算后 O(1)）。**语义正确**。
- 拷贝：`KeyPropertyDescriptor->Set` 走属性赋值，`FString` key 每插入一次就深拷贝一次 —— 这是必要的。
- **性能**：由于 `Get`/`Set`/`Contains` **根本不用哈希而是线性扫 `Identical`**（F-HLP-021），C# 每次 `map[key]` 都会构造一个临时 `FString`（在 `FStringPropertyDescriptor::Identical` 里经 `FCSharpEnvironment::GetString<FString>` 把托管句柄转成 native `FString`，见 `FStrPropertyDescriptor.cpp:45-48`）**并且**对表里每个 key 做一次字符串比较。即：一次查找 = 1 次托管→native 转换 + O(n) 次字符串比较，而 `TMap` 本应 O(1)。

---

### 7.3 FMapHelper 发现正文

> 本小节的发现编号为 F-HLP-016 ~ F-HLP-023，严重度分布（按各条 `严重度` 字段重算）：**P0 × 1**（023）、**P1 × 3**（016、019、020）、**P2 × 4**（017、018、021、022），合计 8 条。按严重度排序的完整索引见 §5.0。

#### [F-HLP-016] `FMapHelper` 构造函数在 key/value 属性任一为空时**不初始化** `ScriptMapLayout`，而所有方法都用它计算偏移 → 用未初始化数据做指针算术

- **类别**: 未定义行为 / 内存损坏
- **严重度**: **P1**
- **复核结论**: 确认 —— 机制（`ScriptMapLayout` 只在半条件分支内赋值 + 成员无默认初始化器 → 未初始化数据参与指针算术）、类别、严重度 P1 均成立
- **可达性**: 活跃（`FMapPropertyDescriptor.cpp:39`/`:56` 传的是引擎已解析的 `Property->KeyProp`/`ValueProp`；只要其中任一为 `nullptr` 即命中，且 `FMapHelper.cpp` 内使用 `ScriptMapLayout` 的 **15 处**全无判空）
- **复核证据**: `FMapHelper.h:62`（`FScriptMapLayout ScriptMapLayout;`，**无 `{}`、无默认成员初始化器**）；`FMapHelper.cpp:6-10`（成员初始化列表**未列出** `ScriptMapLayout`，只列了 `KeyPropertyDescriptor`/`ValuePropertyDescriptor`/`ScriptMap`/两个 `bool`）、`:21-31`（仅 `if (InKeyProperty != nullptr && InValueProperty != nullptr)` 内赋值）；使用点 `:70`/`:100`/`:117`/`:123`/`:136`/`:137`/`:163`/`:166`/`:182`/`:196`/`:198`/`:213`/`:218`/`:254`/`:261`
- **级别变动**: 无（维持 P1；未初始化数据被当作 layout 参与指针算术 → 内存损坏，与 P0 的差别只在"需属性解析失败"这一前提）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FMapHelper.cpp:21-31`；成员声明 `Source/UnrealCSharp/Public/Reflection/Container/FMapHelper.h:62`
- **函数**: `FMapHelper::FMapHelper(FProperty*, FProperty*, void*, bool, bool)`
- **置信度**: 高（插件侧代码确证）；中（"该分支是否会被真实触发"取决于属性元数据，未验证）

**现状（代码事实）**
```cpp
// FMapHelper.h:60-66
	FScriptMap* ScriptMap;

	FScriptMapLayout ScriptMapLayout;      // ← 未在构造初始化列表中初始化

	bool bNeedFreeData;

	bool bNeedFreeProperty;
```
```cpp
// FMapHelper.cpp:4-32
FMapHelper::FMapHelper(FProperty* InKeyProperty, FProperty* InValueProperty, void* InData,
                       const bool InbNeedFreeData, const bool InbNeedFreeProperty) :
	KeyPropertyDescriptor(nullptr),
	ValuePropertyDescriptor(nullptr),
	ScriptMap(nullptr),
	// ScriptMapLayout 缺失
	bNeedFreeData(InbNeedFreeData),
	bNeedFreeProperty(InbNeedFreeProperty)
{
	...
	if (InKeyProperty != nullptr && InValueProperty != nullptr)     // ← 只有这里才赋值
	{
		KeyPropertyDescriptor = FPropertyDescriptor::Factory(InKeyProperty);
		ValuePropertyDescriptor = FPropertyDescriptor::Factory(InValueProperty);
		ScriptMapLayout = FScriptMap::GetScriptLayout(KeyPropertyDescriptor->GetSize(), ...);
	}
}
```

**调用上下文**
`FMapPropertyDescriptor::NewRef`（`FMapPropertyDescriptor.cpp:39`，参数 `Property->KeyProp`/`Property->ValueProp`）与 `NewWeakRef`（`:56`）是主要入口；`TPropertyValue.inl:660`/`:689`/`:697` 的 `KeyProperty`/`ValueProperty` 来自 `FTypeBridge::Factory<>(FoundClass->GetGenericArgument(), ...)`（`:648-656`、`:675-683`）——**这两个值是否非空完全取决于 C# 泛型参数能否被解析成 `FProperty`**。任何泛型参数解析失败（或 `TMap` 的 `KeyProp`/`ValueProp` 为空）都会命中 `else` 路径。

**问题**
1. `FScriptMapLayout` 是聚合体（`{ int32 ValueOffset; int32 ValueSize; int32 PairSize; ... }` 之类），默认初始化后是**不确定值**。`Empty`（`:70`）、`Remove`（`:100`、`:117`、`:123`）、`FindKey`（`:136-137`）、`Get`（`:163-166`）、`Set`（`:182`、`:196-198`、`:213`、`:218`）、`GetEnumeratorKey/Value`（`:254`、`:261`）**全部**把它传给 `FScriptMap` 的引擎 API 或加到自己算出的指针上。
2. 后果不是"返回空值"而是**在任意地址上构造/析构元素**：例如 `Set` 会先在不确定的 `ValueOffset` 处 `DestroyValue`（`:215`），等于对随机地址解释成 `FString`/`FText` 并析构 → 野指针释放/崩溃；`AddUninitialized(Layout)` 会用不确定的 `PairSize` 去分配/定位 → 堆损坏。
3. 该路径**没有任何 `check`/`ensure`/日志**：构造函数静默返回一个"半成品"对象，问题在几十行之外才爆发。

**建议**
```cpp
// 最小止血：让不变量显式化
FMapHelper::FMapHelper(FProperty* InKeyProperty, FProperty* InValueProperty, void* InData,
                       const bool InbNeedFreeData, const bool InbNeedFreeProperty) :
	KeyPropertyDescriptor(nullptr),
	ValuePropertyDescriptor(nullptr),
	ScriptMap(nullptr),
	ScriptMapLayout{},                                    // ← 零初始化，避免不确定值
	bNeedFreeData(InbNeedFreeData),
	bNeedFreeProperty(InbNeedFreeProperty)
{
	checkf(InKeyProperty != nullptr && InValueProperty != nullptr,
	       TEXT("FMapHelper requires both key and value properties"));
	KeyPropertyDescriptor  = FPropertyDescriptor::Factory(InKeyProperty);
	ValuePropertyDescriptor = FPropertyDescriptor::Factory(InValueProperty);
	checkf(KeyPropertyDescriptor != nullptr && ValuePropertyDescriptor != nullptr,
	       TEXT("FMapHelper: unsupported key/value property"));
	ScriptMapLayout = FScriptMap::GetScriptLayout(...);
}
```
即：**把"缺属性"从"静默半初始化"改成构造期硬失败**（`checkf`），并在 `FMapPropertyDescriptor::NewRef/NewWeakRef` 与 `TPropertyValue::Get` 侧对 `Property->KeyProp`/`FTypeBridge::Factory` 的结果做前置校验（后者已返回 `FProperty*`/`nullptr`，见 `TPropertyValue.inl:648-656`）。若确实需要容忍缺属性，则所有使用 `ScriptMapLayout` 的方法都必须先 `if (KeyPropertyDescriptor == nullptr) return;`（`FSetHelper` 有同样的问题，见 F-HLP-024）。

**验证方式**
- 静态：`grep -n "ScriptMapLayout\|ScriptSetLayout" Source/UnrealCSharp/Private/Reflection/Container/ Source/UnrealCSharp/Public/Reflection/Container/` 确认成员未在初始化列表出现。
- 动态：临时把 `if (InKeyProperty != nullptr && InValueProperty != nullptr)` 改成 `if (false)`，在 Debug 下调用 `map.Empty(0)`，应观察到 `FScriptMap::Empty` 里读到垃圾 `PairSize`（或加 `checkf` 后立即命中断言）。

#### [F-HLP-017] `FMapHelper` 构造函数对 `Factory` 返回值无判空：任一元素属性不被支持即空指针解引用

- **类别**: 未定义行为 / 崩溃
- **严重度**: **P2**
- **复核结论**: 确认 —— 机制（条件只判 `FProperty*` 非空，而解引用的是 `Factory` 的返回值）、类别、严重度 P2 均成立
- **可达性**: 活跃（触发条件是"属性对象非空但类型不被 `Factory` 支持"；一旦命中会在构造期立即崩溃，因此比 F-HLP-016 更早暴露）
- **复核证据**: `FMapHelper.cpp:21`（只判两个 `FProperty*`）、`:23`/`:25`（两次 `Factory`）、`:27-30`（直接解引用 `GetSize()`/`GetMinAlignment()`）；`FPropertyDescriptor.cpp:133`（静默 `return nullptr;`）、`:49-131`（白名单）、`:42-45`/`:97-103`/`:129-131`（`#if` 分支）；`FPropertyDescriptor.inl:65-73`/`:75-83`（基类在 `GetProperty()==nullptr` 时 `GetSize()`/`GetMinAlignment()` 返回 0，故哨兵方案可行）
- **级别变动**: 无（维持 P2；P1→P2 的定级依据：触发前提是"元素属性类型不受支持"，属隐患而非常态路径）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FMapHelper.cpp:21-31`
- **函数**: `FMapHelper::FMapHelper(...)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FMapHelper.cpp:21-31
	if (InKeyProperty != nullptr && InValueProperty != nullptr)      // ← 只校验了 FProperty* 非空
	{
		KeyPropertyDescriptor = FPropertyDescriptor::Factory(InKeyProperty);      // 可能返回 nullptr
		ValuePropertyDescriptor = FPropertyDescriptor::Factory(InValueProperty);  // 可能返回 nullptr

		ScriptMapLayout = FScriptMap::GetScriptLayout(KeyPropertyDescriptor->GetSize(),        // ← 解引用
		                                              KeyPropertyDescriptor->GetMinAlignment(),
		                                              ValuePropertyDescriptor->GetSize(),
		                                              ValuePropertyDescriptor->GetMinAlignment());
	}
```
`FPropertyDescriptor::Factory` 的失败返回是 `return nullptr;`（`FPropertyDescriptor.cpp:133`），且失败是**静默**的。

**调用上下文**
同 F-HLP-016：5 个构造点中 `TPropertyValue.inl:660`/`:689`/`:697` 的属性对象由 `FTypeBridge::Factory<>` 现场创建（`TPropertyValue.inl:648`、`:653`、`:675`、`:680`），类型来自 C# 泛型参数 → 插件支持类型之外的元素类型（例如某些版本的 `FOptionalProperty`、`FFieldPathProperty`，或 `#if` 未开启的 `FUtf8StrProperty`/`FAnsiStrProperty`，见 `FPropertyDescriptor.cpp:97-103`、`:129-131`）会走到这里。

**问题**
1. `KeyPropertyDescriptor` 与 F-HLP-016 的 `nullptr` 是可以同时发生的：条件成立（`FProperty*` 非空）但 `Factory` 失败（描述符为空）→ 在 `:27` 立即空指针解引用崩溃。**这是比 F-HLP-016 更靠前、更确定的一处崩溃**。
2. 与 `FArrayHelper` 的对应缺陷（F-HLP-002）不同，这里的解引用发生在**构造函数内**，因此崩溃现场更接近根因，但依然没有可读的错误信息。
3. 另外，即使两个描述符都成功，`FMapHelper.h:62` 的 `ScriptMapLayout` 计算依赖 `GetSize()`/`GetMinAlignment()`——`FPropertyDescriptor::GetMinAlignment()` 在 `GetProperty()` 为空时返回 0（`FPropertyDescriptor.inl:75-83`），这会让 layout 静默变成"元素 0 字节"，同样是"无错误信息的错误结果"。

**建议**
按 F-HLP-016 的建议在构造期 `checkf` 两个描述符均非空；更彻底的做法是让 `FPropertyDescriptor::Factory` 在遇到不支持类型时返回一个**非空的哨兵描述符**（基类各方法已有安全默认实现：`FPropertyDescriptor.cpp:136-205`），从而把"崩溃"变成"可记录的错误 + 后续操作无效"，并让 `GetSize() == 0` 这类信号能被上层识别。

**验证方式**
`grep -n "NEW_PROPERTY_DESCRIPTOR" Source/UnrealCSharp/Private/Macro/PropertyMacro.h` 看白名单宏；构造一个 `TMap<FFieldPath, int>` 或 `TMap<FOptional<int>, int>` 属性并在 C# 里访问，观察 `:27` 处崩溃。

#### [F-HLP-018] `FMapHelper::Deinitialize` 的释放条件要求"两个描述符同时非空"，否则一个都不释放

- **类别**: 内存/资源泄漏
- **严重度**: **P2**
- **复核结论**: 确认 —— `&&` 联判的代码事实与"两个描述符独立分配、部分成功即全部不释放"的推理均成立；级别 P2 成立
- **可达性**: 潜伏（在构造期 F-HLP-017 会先在同一状态下崩溃；只有按 F-HLP-017 建议改成"容忍空描述符 + 报错"后，这个 `&&` 才成为真实泄漏点 —— 原文对此已写明）
- **复核证据**: `FMapHelper.cpp:52`（`if (bNeedFreeProperty && KeyPropertyDescriptor != nullptr && ValuePropertyDescriptor != nullptr)`）、`:23`/`:25`（两个描述符各自独立 `new`）；对照 `FArrayHelper.cpp:41`（单描述符只判一次）、`FSetHelper.cpp:48`（同）
- **级别变动**: 无（维持 P2）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FMapHelper.cpp:52-65`
- **函数**: `FMapHelper::Deinitialize()`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FMapHelper.cpp:52-65
	if (bNeedFreeProperty && KeyPropertyDescriptor != nullptr && ValuePropertyDescriptor != nullptr)
	{                                                                  // ← 与（&&）而非分别判断
		KeyPropertyDescriptor->DestroyProperty();
		delete KeyPropertyDescriptor;
		KeyPropertyDescriptor = nullptr;

		ValuePropertyDescriptor->DestroyProperty();
		delete ValuePropertyDescriptor;
		ValuePropertyDescriptor = nullptr;
	}
```
对照 `FArrayHelper::Deinitialize`（`FArrayHelper.cpp:41-48`）只判一个描述符——单个 Helper 的写法是对的，Map 的写法引入了一个额外的合取项。

**调用上下文**
`bNeedFreeProperty = true` 的构造点是 `TPropertyValue.inl:660`、`:689`、`:697`（三个 map 构造点）。若其中 `ValuePropertyDescriptor` 为 `nullptr`（`FTypeBridge::Factory` 解析 value 泛型失败 → `Factory` 返回空），则：
- 构造期会在 `:27` 崩溃（F-HLP-017）——所以这个分支**在崩溃被修掉之前不可达**；
- 一旦按 F-HLP-017 的建议把构造期改为"允许空描述符 + 报错"，这个 `&&` 就会变成**真实的泄漏点**：`KeyPropertyDescriptor` 非空却不会被释放。

**问题**
两个描述符是**独立**的分配（`FMapHelper.cpp:23`、`:25` 各自 `new`），因此所有权判断必须逐项进行。用一个 `&&` 联判把"部分成功"的状态变成"全部泄漏"。这是典型的"用合取表达两个独立不变量"的错误。

**建议**
```cpp
	if (bNeedFreeProperty)
	{
		if (KeyPropertyDescriptor != nullptr)
		{
			KeyPropertyDescriptor->DestroyProperty();
			delete KeyPropertyDescriptor;
			KeyPropertyDescriptor = nullptr;
		}
		if (ValuePropertyDescriptor != nullptr)
		{
			ValuePropertyDescriptor->DestroyProperty();
			delete ValuePropertyDescriptor;
			ValuePropertyDescriptor = nullptr;
		}
	}
```

**验证方式**
`grep -n "bNeedFreeProperty" -A12 Source/UnrealCSharp/Private/Reflection/Container/FMapHelper.cpp`；在 `KeyPropertyDescriptor` 非空、`ValuePropertyDescriptor` 为空的状态下析构，用 ASan/`FMemory` 统计确认泄漏。

#### [F-HLP-019] `TMap`/`TSet`/`TArray` 传值拷贝路径用 `delete` 经 `FScriptMap*`/`FScriptSet*`/`FScriptArray*` 释放真正的容器对象 → **元素自有资源**泄漏

- **类别**: 内存/资源泄漏 | 未定义行为
- **严重度**: **P1**
- **复核结论**: 部分确认（偏差：**"`delete` 静态类型不匹配 → `~TMap<K,V>()` 不执行 → 元素析构被跳过"成立**；但"整张表的键值存储与哈希索引全部泄漏"**被引擎源码证伪** —— 容器自身的 Pairs/Hash 缓冲由 allocator 基类析构释放，泄漏的只有 key/value 的**自有资源**）
- **可达性**: 活跃（`TPropertyValue.inl:697` 是唯一 `bNeedFreeData=true` 的 map 构造点）
- **复核证据**: `FMapHelper.cpp:14`（`static_cast<FScriptMap*>(InData)`）、`:45-50`（`delete ScriptMap`）；`TPropertyValue.inl:697-698`；**引擎侧**：`Containers/Map.h:2076`（`TScriptSet<AllocatorType> Pairs;` 是 `TScriptMap` 的唯一成员）、`:2119-2133`（`FScriptMap` 未声明析构）、`:2079-2093`（`static_assert(sizeof(TScriptMap) == sizeof(TMap<int32,int8>))` + 成员 offset 断言 → **布局等价由引擎保证**，不再是"实现巧合"）、`Containers/Set.h:2049-2054`（`Elements`/`Hash` 各有 allocator 成员）、`Containers/ContainerAllocationPolicies.h:683-693`（`~ForAnyElementType` 会 `Free(Data)`）
- **级别变动**: 无（维持 P1）；**范围修正**：含 `FString`/`FText`/嵌套容器 key 或 value 的 `TMap`，泄漏量与原描述同阶；对 `TMap<int,int>` 这类**元素无自有资源**的实例，本路径**不产生泄漏**（原文"泄漏量更大"的比较对象应改为"泄漏的是每个元素的自有资源"）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FMapHelper.cpp:45-50`（释放）；`Source/UnrealCSharp/Public/Binding/Core/TPropertyValue.inl:697`（分配）
- **函数**: `FMapHelper::Deinitialize()`
- **置信度**: 高（插件侧确证）；高（**布局兼容性已由引擎源码确证**：`Containers/Map.h:2079-2093` 的 `static_assert` 逐成员断言 `TScriptMap` 与 `TMap<int32,int8>` 的 size/offset 一致；`Containers/ContainerAllocationPolicies.h:683-693` 确证 allocator 基类析构会释放缓冲）

**现状（代码事实）**
```cpp
// FMapHelper.cpp:12-19
	if (InData != nullptr)
	{
		ScriptMap = static_cast<FScriptMap*>(InData);      // 真实类型可能是 TMap<K,V>
	}
	else
	{
		ScriptMap = new FScriptMap();
	}
// FMapHelper.cpp:45-50
	if (bNeedFreeData && ScriptMap != nullptr)
	{
		delete ScriptMap;                                  // 无虚析构 → ~TMap<K,V>() 不会执行
		ScriptMap = nullptr;
	}
```
```cpp
// TPropertyValue.inl:695-698
		else
		{
			const auto MapHelper = new FMapHelper(KeyProperty, ValueProperty,
			                                      new std::decay_t<T>(*InMember), true, true);
			//                                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ 真正的 TMap<K,V>
```

**调用上下文**
`bNeedFreeData = true` 只有 `TPropertyValue.inl:697` 一处（`TPropertyValue.inl:660`/`:689` 与 `FMapPropertyDescriptor.cpp:39`/`:56` 都是 `false`）。触发条件：C# 以"传值"语义把一个 `TMap<K,V>` 交给 native（`IsReference == false`）。

**问题**
1. 与 F-HLP-001（Array）完全同形，但**泄漏面更大**：一个 `TMap` 的 key/value 是两套独立对象。**修正**：`delete` 经 `FScriptMap*` 只释放那块布局结构体本身，而 **`Pairs` 展开出的 Elements 稀疏数组与 Hash 桶数组会由 allocator 基类析构释放**（引擎 `Containers/ContainerAllocationPolicies.h:683-693`；`TScriptSet::Elements`/`Hash` 是两个持有 allocator 的成员，见 `Containers/Set.h:2049-2054`）。因此**不会**泄漏"整张表的键值存储与哈希索引"；真正泄漏的是**每个 key/value 自身的资源**（`FString`/`FText` 缓冲、嵌套容器、`UObject*` 的引用处理时机），且元素类型无自有资源时**根本不泄漏**。
2. 同一问题在 `FSetHelper` 上重复（见 F-HLP-026），在 `FArrayHelper` 上即 F-HLP-001 —— **三处同一个 bug，应一次性修掉**（见 §8.2 方案 D 的类型擦除销毁器方案）。
3. 另外，`FScriptMap` 的实际布局与 `FScriptMapLayout` 的成员类型在引擎各版本间有变化，`static_cast<FScriptMap*>(void*)` 依赖调用方传入的对象**真的是** `FScriptMap` 布局；对 `TMap<K, V, TInlineAllocator<N>>`（inline 存储）等非默认 allocator 的实例，布局不一致会让释放和后续所有访问都错位。当前 `TPropertyValue` 只覆盖默认 allocator 的 `TMap`，所以是潜伏风险，但**没有任何 `static_assert` 把它固化**。**引擎侧提示（新增）**：引擎自己在 `Containers/Map.h:2079-2093` 用 `static_assert(sizeof(TScriptMap)==sizeof(TMap<int32,int8>))` 固化了"默认 allocator 下等价"，插件可照此写一行编译期检查（见 §8.2 方案 D）。

**建议**
同 F-HLP-001：给 Helper 增加类型擦除的销毁器（`void(*)(void*)`），在 `TPropertyValue.inl:697` 传 `[](void* P){ delete static_cast<std::decay_t<T>*>(P); }`，`Deinitialize` 里改调它；并加 `static_assert(sizeof(std::decay_t<T>) == sizeof(FScriptMap), ...)` 之类的编译期检查（在模板里可写）。

**验证方式**
C# 用例：把一个含 10 万条 `FString→FString` 的 `TMap` 以传值方式交给 native 函数，然后 `UnRegister`，用 `MemReport -full` 对比前后 `FString`/`TMap` 相关分配数；预期当前实现泄漏整张表。

#### [F-HLP-020] `FMapHelper::Empty` 只调 `FScriptMap::Empty`，**不析构 key/value** → 清空字典会泄漏所有键值持有的资源

- **类别**: 内存/资源泄漏
- **严重度**: **P1**
- **复核结论**: 确认 —— 机制、调用上下文、类别、严重度 P1 全部成立；"引擎 `Empty` 不析构元素"已由引擎源码确证（不再是推测）
- **可达性**: 活跃（`FRegisterMap.cpp:32-39`，注册名 `Empty`，`:190`；C# `TMap<K,V>.Empty(slack)`）
- **复核证据**: `FMapHelper.cpp:70`（`ScriptMap->Empty(...)`，全文无任何 `DestroyValue`）；同文件 `:119-123` 的手动 `DestroyValue` 对照；**引擎侧确证**：`Containers/Map.h:1955-1958`（`TScriptMap::Empty` → `Pairs.Empty`）、`Containers/Set.h:1843-1865`（`TScriptSet::Empty` 只做 `Elements.Empty(...)` + 重置/重建 `Hash`，**完全不触碰元素析构**）
- **级别变动**: 无（维持 P1）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FMapHelper.cpp:68-71`
- **函数**: `FMapHelper::Empty(int32)`
- **置信度**: 高（已用引擎源码确证："`FScriptMap::Empty` 不析构元素" —— `Containers/Map.h:1955-1958` → `Containers/Set.h:1843-1865`；插件内自相矛盾的证据同样成立：同文件 `Remove` 手动析构、`FArrayHelper::Empty` 用会析构的 `EmptyValues`）

**现状（代码事实）**
```cpp
// FMapHelper.cpp:68-71
void FMapHelper::Empty(const int32 InExpectedNumElements) const
{
	ScriptMap->Empty(InExpectedNumElements, ScriptMapLayout);   // ← 不调用任何 DestroyValue
}
```
同一文件里 `Remove` 的写法明确表明"引擎的 `RemoveAt` 不负责析构"：
```cpp
// FMapHelper.cpp:117-123
		const auto Data = static_cast<uint8*>(ScriptMap->GetData(KeyIndex, ScriptMapLayout));

		KeyPropertyDescriptor->DestroyValue(Data);                               // key 手动析构
		ValuePropertyDescriptor->DestroyValue(Data + ScriptMapLayout.ValueOffset); // value 手动析构
		ScriptMap->RemoveAt(KeyIndex, ScriptMapLayout);                          // 然后才交给引擎
```
而 `FArrayHelper::Empty`（`FArrayHelper.cpp:244-249`）用的是**会析构元素**的 `FScriptArrayHelper::EmptyValues`：
```cpp
void FArrayHelper::Empty(const int32 InSlack) const
{
	auto ScriptArrayHelper = CreateHelperFormInnerProperty();
	ScriptArrayHelper.EmptyValues(InSlack);      // 析构全部元素
}
```

**调用上下文**
`FRegisterMap.cpp:32-39`（注册名 `Empty`，`:190`）→ C# 的 `TMap<K,V>.Empty(slack)`。这是**清空字典时的必经路径**，任何"复用同一个 map 对象"的 C# 代码都会周期性调用它。

**问题**
1. 若 `FScriptMap::Empty` 与 `FScriptMap::RemoveAt` 一样只归还内存/重置计数（`Remove` 的写法证明作者本人相信这一点），那么 `Empty` 会**跳过所有 key 与 value 的析构**：
   - `FString`/`FText`/`TArray`/`TMap`/`TSet` 类型的 key 或 value → 其内部堆缓冲全部泄漏；
   - `UObject*` 类型的 value → 只是少了引用计数/弱引用处理（若有）→ 悬挂/GC 时机问题；
   - 含 `FString` 的 `USTRUCT` key/value → 每个成员都漏。
2. 泄漏量与"清空前的元素个数 × 每个元素的堆分配数"成正比，且是**每次 `Empty` 都发生**（不是一次性）。
3. 与 `FArrayHelper::Empty` 的行为不一致，说明这里的偏差是**遗漏**而非有意设计——作者在 Array 侧知道要用 `EmptyValues`，在 Map 侧直接调了底层 `ScriptMap->Empty`。

**建议**
UE 提供了与 `FScriptArrayHelper` 对称的 `FScriptMapHelper`（含 `EmptyValues`/`FindPair`/`AddPair`/`RemovePair`，都是析构感知且**基于哈希**的）。建议整个 `FMapHelper` 改为基于 `FScriptMapHelper` 实现（同时解决 F-HLP-020、F-HLP-021、F-HLP-022 三件事）：
```cpp
void FMapHelper::Empty(const int32 InExpectedNumElements) const
{
	FScriptMapHelper Helper(KeyPropertyDescriptor->GetProperty(), ScriptMap, ScriptMapLayout);
	Helper.EmptyValues(InExpectedNumElements);      // 逐个析构 key/value 后再归还内存
}
```
若不想引入 `FScriptMapHelper` 的构造签名依赖，则手工补上析构循环：
```cpp
void FMapHelper::Empty(const int32 InExpectedNumElements) const
{
	for (int32 Index = 0; Index < ScriptMap->GetMaxIndex(); ++Index)
	{
		if (ScriptMap->IsValidIndex(Index))
		{
			const auto Data = static_cast<uint8*>(ScriptMap->GetData(Index, ScriptMapLayout));
			KeyPropertyDescriptor->DestroyValue(Data);
			ValuePropertyDescriptor->DestroyValue(Data + ScriptMapLayout.ValueOffset);
		}
	}
	ScriptMap->Empty(InExpectedNumElements, ScriptMapLayout);
}
```
（注意必须先析构再 `Empty`，否则 `IsValidIndex` 已被重置、拿不到旧元素。）

**验证方式**
- C# 用例：`var m = new TMap<string,string>(); for (i<100000) m.Add(k_i, v_i); m.Empty(0);` 循环 100 次，用 `MemReport`/`FPlatformMemory::GetStats()` 观察 `FString` 分配量随循环线性增长。
- 或写 C++ 单测：构造 `TMap<FString, FString>`，用 `FMapHelper` 包装，调 `Empty`，再用 `FString` 的分配计数器断言"析构次数 == 元素数"。

#### [F-HLP-021] `TMap` 的 `Get`/`Set`/`Contains`/`Find`/`FindKey`/`Remove` 全部退化为 O(MaxIndex) 线性扫描，完全没有使用 `FScriptMap` 的哈希索引

- **类别**: 性能
- **严重度**: **P2**（性能，但它是本节最普遍的缺陷：把 O(1) 字典查找变成 O(n) 全表扫描）
- **复核结论**: 确认 —— 四处循环都退化为 O(MaxIndex) 线性扫描的代码事实、类别（性能）、严重度 P2 均成立；"这不是引擎限制"已确证
- **可达性**: 活跃
- **复核证据**: `FMapHelper.cpp:96`（`Remove`）、`:132`（`FindKey`）、`:159`（`Get`）、`:178`（`Set`）—— 四处都是 `for (auto Index = 0; Index < ScriptMap->GetMaxIndex(); ++Index)`；`FRegisterMap.cpp:79`/`:91`/`:102`/`:112`/`:124`/`:135`；**引擎侧反证**：`Containers/Map.h:1982-2001`（`TScriptMap::FindPairIndex` 哈希驱动）、`:2004-2014`（`FindValue`）、`Containers/Set.h:1944-1984`（`FindIndex`/`FindIndexByHash`）、`UnrealType.h:4787`（`COREUOBJECT_API void FScriptMapHelper::Rehash();`）、`:5597`（`FScriptSetHelper::Rehash()`）→ **插件完全可以使用哈希，未使用是设计选择**
- **级别变动**: 无（维持 P2；P1→P2 的定级依据：正确性由线性扫描兜住，损失的是性能）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FMapHelper.cpp:96-108`、`:132-142`、`:159-169`、`:178-190`
- **函数**: `FMapHelper::Remove` / `FindKey` / `Get` / `Set`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FMapHelper.cpp:157-172   Get —— 典型的哈希查找被写成线性扫描
void* FMapHelper::Get(const void* InKey) const
{
	for (auto Index = 0; Index < ScriptMap->GetMaxIndex(); ++Index)      // ← O(MaxIndex)
	{
		if (ScriptMap->IsValidIndex(Index))
		{
			if (const auto Data = static_cast<uint8*>(ScriptMap->GetData(Index, ScriptMapLayout));
				KeyPropertyDescriptor->Identical(Data, InKey))            // ← 每个槽位一次 Identical
			{
				return Data + ScriptMapLayout.ValueOffset;
			}
		}
	}

	return nullptr;
}
```
```cpp
// FMapHelper.cpp:174-190   Set —— 插入前也要先线性扫一遍找同 key
	for (auto Index = 0; Index < ScriptMap->GetMaxIndex(); ++Index)
	{
		if (ScriptMap->IsValidIndex(Index))
		{
			if (const auto Data = static_cast<uint8*>(ScriptMap->GetData(Index, ScriptMapLayout));
				KeyPropertyDescriptor->Identical(Data, InKey))
			{
				KeyIndex = Index;
				break;
			}
		}
	}
```
`Contains`（`:152-155`）转调 `Get`，`Find`（`:147-150`）转调 `Get`，`Remove`（`:96-108`）也是同样的扫描。

**调用上下文**
`FRegisterMap.cpp:124`（`Get`）、`:135`（`Set`）、`:79`（`Remove`）、`:102`（`Find`）、`:112`（`Contains`）、`:91`（`FindKey`）——**C# 侧 `TMap` 的每一次读取、写入、删除都是这条路径**。同时 `TPropertyValue.inl:713-722` 的"读回 C#"路径也是 O(MaxIndex) 循环（那里是必要的，因为要枚举全部元素）。

**问题**
1. **算法层面**：`TMap` 的全部价值就是 O(1) 平均查找。当前实现让每次 `map[key]` 变成 O(MaxIndex) 次 `KeyPropertyDescriptor->Identical`。对 n 条记录的字典，一次查找 O(n)，构建 n 条记录需要 O(n²)（每次 `Add` 都先扫一遍查重）。
2. **常数因子放大**：`Identical` 不是简单的整数比较。对 `FString` key，`FStrPropertyDescriptor::Identical`（`FStrPropertyDescriptor.cpp:41-48`）每次调用都要 `Property->GetPropertyValue(A)`（取值）**以及** `FCSharpEnvironment::GetEnvironment().GetString<FString>(*(IManagedHandle*)B)`（**每次迭代都把托管句柄重新解析成 native `FString`**）。也就是说算法是 O(n) 次"跨托管边界转换 + 字符串比较"，而不是 O(n) 次指针比较。
3. **稀疏表放大**：循环上界是 `GetMaxIndex()` 而不是 `Num()`。由于插件从不收缩 map（`Remove` 调 `RemoveAt` 但不 `Shrink`），反复增删会把 `GetMaxIndex()` 推高，使得"表里只有 1 个元素"时仍要扫描整个已分配的 slot 数组。
4. **hash 白算**：`Set` 在插入时确实调了 `Rehash` 并用 `GetValueTypeHash` 算了哈希（`:203-209`），但**没有任何查询走哈希**——哈希维护的代价付了，收益一点没拿到。这是本报告中最明确的"设计层错配"。

**建议**
改用 UE 的 `FScriptMapHelper`（与 `FArrayHelper` 用 `FScriptArrayHelper` 的做法对齐），它的 `FindPair`/`FindValue`/`AddPair`/`RemovePair` 都是哈希驱动的：
```cpp
void* FMapHelper::Get(const void* InKey) const
{
	FScriptMapHelper Helper(KeyPropertyDescriptor->GetProperty(), ScriptMap, ScriptMapLayout);
	const int32 Index = Helper.FindPair(InKey);          // O(1) 平均
	return Index == INDEX_NONE ? nullptr
	                           : static_cast<uint8*>(Helper.GetPairPtr(Index)) + ScriptMapLayout.ValueOffset;
}
```
若坚持不引入 `FScriptMapHelper`，则必须自己用 `ScriptMap->Hash` 做探测（需要 `FScriptMapLayout.HashNextOffset`/`HashSize` 等），复杂度不值得。

**验证方式**
- C# 基准：`TMap<int,int>` 插入 10 万条 vs 查询 10 万次，当前实现耗时随规模平方增长；改用 `FScriptMapHelper` 后应接近线性。
- 计数：在 `KeyPropertyDescriptor->Identical` 上加计数器，断言"一次 `Get` 的 `Identical` 调用次数应为 O(1) 量级"。

#### [F-HLP-022] `Set` 每次插入新键都做**全表 `Rehash`**（性能缺陷成立）；而 `Remove` 删除后不 `Rehash` 是**正确**的 —— 原文"留下陈旧哈希索引"一半已被引擎源码证伪

- **类别**: 性能（**收窄**：原"性能 / 潜在 Bug / 未定义行为"中的"潜在 Bug / 未定义行为"部分撤销）
- **严重度**: **P2**（纯性能；"索引陈旧性为潜在 bug"一节撤销）
- **复核结论**: 部分确认（偏差：**性能**一半成立；**"`Remove` 后不 `Rehash` 会留下指向已释放槽位的哈希条目"被引擎源码证伪** —— `TScriptSet::RemoveAt` 自己就负责把该槽位从哈希链摘除）
- **可达性**: 活跃（性能部分：`FRegisterMap.cpp:69`（`Add`）/`:135`（`Set`）→ 每次 C# 写新键）
- **复核证据**: 性能侧 `FMapHelper.cpp:196-209`（`AddUninitialized` → `KeyPropertyDescriptor->Set` → **无条件** `Rehash`）；引擎对照 `Containers/Set.h:2022-2047`（`AddNewElement` 仅在 `HashSize < DesiredHashSize` 时 `Rehash`，否则只把新元素挂到桶头，均摊 O(1)）；**证伪侧** `Containers/Set.h:1867-1885`（`RemoveAt`：`:1873-1881` 沿 `HashNextId` 链找到前驱并把 `*NextElementId` 改指向被删项的后继 → 哈希链已正确维护；`:1884` 才 `Elements.RemoveAtUninitialized`）、`Containers/Map.h:1960-1963`（`TScriptMap::RemoveAt` 转调 `Pairs.RemoveAt`）
- **级别变动**: 无（维持 P2）；**类别收窄**为纯性能；§问题 2、§建议 2 已据此改写
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FMapHelper.cpp:196-209`（Rehash）；`:117-123`（RemoveAt，**不需要** Rehash）
- **函数**: `FMapHelper::Set(void*, void*)` / `FMapHelper::Remove(const void*)`
- **置信度**: 高（已对照引擎源码："`AddUninitialized` 要求后续 `Rehash`"确为引擎约定 —— `Containers/Set.h:1887-1897` 的注释 "The set will need rehashing at some point after this call to make it valid"；同时确证 `RemoveAt` **不需要**外部补 `Rehash`，见 `Set.h:1867-1885`）

**现状（代码事实）**
```cpp
// FMapHelper.cpp:194-210   Set：新键 → AddUninitialized → Set key → Rehash（全表）
	if (KeyIndex == INDEX_NONE)
	{
		KeyIndex = ScriptMap->AddUninitialized(ScriptMapLayout);

		Data = static_cast<uint8*>(ScriptMap->GetData(KeyIndex, ScriptMapLayout));

		KeyPropertyDescriptor->Set(InKey, Data);

#if STD_CPP_20
		ScriptMap->Rehash(ScriptMapLayout, [=, this](const void* Src)
#else
		ScriptMap->Rehash(ScriptMapLayout, [=](const void* Src)
#endif
		                  {
			                  return KeyPropertyDescriptor->GetValueTypeHash(Src);
		                  });
	}
```
```cpp
// FMapHelper.cpp:117-123   Remove：RemoveAt 之后没有 Rehash
		const auto Data = static_cast<uint8*>(ScriptMap->GetData(KeyIndex, ScriptMapLayout));

		KeyPropertyDescriptor->DestroyValue(Data);

		ValuePropertyDescriptor->DestroyValue(Data + ScriptMapLayout.ValueOffset);

		ScriptMap->RemoveAt(KeyIndex, ScriptMapLayout);
	}      // ← 无 Rehash
```

**调用上下文**
插入：`FRegisterMap.cpp:69`（`Add`）/`:135`（`Set`）→ 每次 C# 写一个新键。删除：`FRegisterMap.cpp:79`（`Remove`）。

**问题**
1. **性能**：`Rehash` 是全表重算（对每个已占用槽位重新求哈希并重建 `Hash` 链）。把它放在**每次**新键插入路径上，使得向空 map 插入 n 条记录的总成本为 O(n²)（还要叠加 F-HLP-021 的线性查重，同样是 O(n²)）。UE 的 `FScriptMapHelper::AddPair` 只在**容量增长**时才 `Rehash`（把 `Rehash` 的开销均摊为摊还 O(1)）。
2. **一致性（修正：这一点并非缺陷）**：`Set` 认为"插入新键后必须 `Rehash`"，`Remove` 却不做 —— 原文据此推断"删除会留下指向已释放槽位的哈希条目"。**该推断已被引擎源码证伪**：`Containers/Set.h:1867-1885` 显示 `TScriptSet::RemoveAt` **自己就会**沿 `HashNextId` 链把该槽位从哈希链中摘除（`:1873-1881`），因此"不 `Rehash`"是**正确**的。真正需要外部补 `Rehash` 的只有 `AddUninitialized` 一侧（`Containers/Set.h:1887-1897` 的注释明确写了 "The set will need rehashing at some point after this call to make it valid"）。**因此插件当前的"Add 补 Rehash / Remove 不补"的不对称是引擎契约要求的，不是 bug。**
3. 结论：F-HLP-021 与 F-HLP-022 的**耦合关系不成立**（原判断为"改哈希前必须先确认 Rehash 缺口"）。按 `Containers/Set.h:1867-1885`，删掉的键不会因为陈旧哈希索引而被 `FindPair`/`FindIndex` 命中，所以**改用哈希查找是安全的**，不需要先补 `Remove` 侧的 `Rehash`。修复顺序上仍需注意的只是 `AddUninitialized` 必须配套 `Rehash`（或改用 `FScriptMapHelper::AddPair` 由引擎负责）。

**建议**
1. 首选：整体改用 `FScriptMapHelper`（`AddPair`/`RemovePair`/`FindPair`），让引擎负责所有的哈希索引维护，插件不再手工调 `Rehash`。
2. 若保留手工路径：给 `Set` 的 `Rehash` 加"仅在容量变化时"的条件（引擎的做法见 `Containers/Set.h:2030-2044`：仅当 `HashSize < Allocator::GetNumberOfHashBuckets(Num())` 时才重建桶数组，否则只挂链）；**不需要**给 `Remove` 补 `Rehash`（`RemoveAt` 已自己维护哈希链，见 `Set.h:1867-1885`）。
3. 无论如何，请在 `FMapHelper.cpp` 里加一行注释说明"为何 `Set` 侧必须 `Rehash` 而 `Remove` 侧不必"，因为当前的两处不对称会被误读为笔误（规范第 3.7 条：代码与意图不一致时要点出）。

**验证方式**
C# 用例：插入 3 条 → `Remove(中间那条)` → 断言 `Contains(被删的键) == false`（**结论：该用例在"改哈希实现后"也必须通过** —— 引擎 `RemoveAt` 已摘除哈希链，见 `Containers/Set.h:1873-1881`，所以这不是一个"会失败的回归"）。性能上给 `Rehash` 加计数器，断言"向 n 条空表插入 n 条记录，`Rehash` 调用次数应为 O(log n) 量级而非 n"。

#### [F-HLP-023] `FMapHelper` 的查找类方法返回 `nullptr`，而 `FRegisterMap` 立刻把它交给属性描述符 `Get` 解引用

- **类别**: 未定义行为 / 崩溃
- **严重度**: **P0****（触发条件是"查询一个不存在的键"，即字典的常态用法；`TMap` 索引器 get 直接走到这里）
- **复核结论**: 确认 —— 代码事实、调用上下文、类别、**严重度 P0** 全部成立 **后续处置：判定升级为「确认 · 已修复（`492ce5f7`）」—— 修复在**调用侧**：`FRegisterMap`/`FRegisterArray`/`FRegisterSet` 共 7 处取值实现改为先判 helper 是否命中，未命中回退到该提交新增的 `FPropertyDescriptor::GetDefaultValue`（按 `GetBufferSize()` 把 C# 返回缓冲清零）**（见下方「处置（已执行）」）
- **可达性**: 活跃（C# `TMap<TKey,TValue>` 的索引器 getter 在键不存在时**无条件**走到 `TMap_GetImplementation`，没有任何 `ContainsKey` 前置）
- **复核证据**: `FMapHelper.cpp:144`（`FindKey` 未找到 → `nullptr`）、`:171`（`Get` 未找到 → `nullptr`）、`:255`/`:262`（`GetEnumeratorKey/Value` 的 `?: nullptr`）；`FRegisterMap.cpp:91-92`、`:102-103`、`:124-125`、`:167-169`、`:179-181`（**五处取值实现全部无判空**）；**C# 侧决定性证据** `Script/UE/CoreUObject/TMap.cs:256-276`（`public TValue this[TKey InKey] { get { … TMap_GetImplementation(handle, KeyBuffer, ReturnBuffer); return *(TValue*)ReturnBuffer; } }`）；子类解引用证据 `FStrPropertyDescriptor.cpp:6`
- **级别变动**: 无（**维持 P0**）：`map[不存在的键]` 是字典的常态用法而**非误用输入**，完全满足本报告 P0 判据（"不需要'误用输入'之外的任何特殊条件"）。三条同形 P0 中只有这一条保留 P0 —— F-HLP-006 因触发条件属误用输入降为 P1，F-HLP-027 因单线程不可达撤销
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FMapHelper.cpp:144`、`:171`、`:255`、`:262`；`Source/UnrealCSharp/Private/Domain/Interop/FRegisterMap.cpp:91`、`:102`、`:124`、`:167`、`:179`
- **函数**: `FMapHelper::FindKey` / `Get` / `GetEnumeratorKey` / `GetEnumeratorValue`
- **置信度**: 高
- **处置（已执行）**: ✅ **已修复**（修复提交 `492ce5f7` "Null Validation"；修复落在**调用侧**，`FMapHelper` 本身不动）。`FRegisterMap`（`FindKey`/`Find`/`Get`/`GetEnumeratorKey`/`GetEnumeratorValue` 5 处）、`FRegisterArray::Get`、`FRegisterSet::GetEnumerator` **共 7 处**取值实现改为先判 helper 是否命中（`if (const auto Value = MapHelper->Get(IN_KEY_BUFFER))` 形态），未命中则回退到该提交新增的 `FPropertyDescriptor::GetDefaultValue(const FPropertyDescriptor*, void*)`——它按 `GetBufferSize()` 用 `FMemory::Memzero` 把 C# 的 `RETURN_BUFFER` 清零，因此 `map[不存在的键]` 现在拿到「零值默认值」而不再崩 native 进程，`nullptr` 不再可能流到描述符 `Get`。原文「建议」第 1 条（调用侧统一补判空）与第 2 条（给 `FPropertyDescriptor` 增加默认值入口）**均已采纳**，第 2 条的落点就是这个 `GetDefaultValue`；但实现形态与原文设想**不同**——不是 `GetDefault(void** Dest)` + `FProperty::InitializeValue` 造默认值，而是按缓冲大小清零。**未做的三处（直说）**：① `FMapHelper` 自身「未找到即返回 `nullptr`」的契约**未改**（`FMapHelper.cpp:144`/`:171` 仍是裸 `return nullptr`，`GetEnumeratorKey/Value` 的 `?: nullptr` 也未改），判空责任仍留在调用侧；② 原文第 3 条建议的 `checkf`／统一辅助（`Checked<T>(p, What)` 形态）**未引入**，`FRegisterMap.cpp:82` 那处本来就自洽的 `return 0` 保持原样；③ 原文建议的 `UE_LOG(LogUnrealCSharp, Warning, TEXT("TMap.Get: key not found"))` 日志**未加**，未命中是静默返回零值。

**现状（代码事实）**
```cpp
// FMapHelper.cpp:144 / :171 / :255 / :262
	return nullptr;                                    // FindKey 未找到
	return nullptr;                                    // Get 未找到
	return ScriptMap->IsValidIndex(InIndex) ? ... : nullptr;   // GetEnumeratorKey
	return ScriptMap->IsValidIndex(InIndex) ? ... : nullptr;   // GetEnumeratorValue
```
```cpp
// FRegisterMap.cpp:91-92   FindKey -> 直接解引用
				MapHelper->GetKeyPropertyDescriptor()->Get(MapHelper->FindKey(IN_VALUE_BUFFER),
				                                           reinterpret_cast<void**>(RETURN_BUFFER));
// FRegisterMap.cpp:102-103  Find
				MapHelper->GetValuePropertyDescriptor()->Get(MapHelper->Find(IN_KEY_BUFFER),
				                                             reinterpret_cast<void**>(RETURN_BUFFER));
// FRegisterMap.cpp:124-125  Get
				MapHelper->GetValuePropertyDescriptor()->Get(MapHelper->Get(IN_KEY_BUFFER),
				                                             reinterpret_cast<void**>(RETURN_BUFFER));
// FRegisterMap.cpp:167-169  GetEnumeratorKey
				const auto Key = MapHelper->GetEnumeratorKey(InIndex);
				MapHelper->GetKeyPropertyDescriptor()->Get(Key, reinterpret_cast<void**>(RETURN_BUFFER));
```
与 `FArrayHelper::Get` → `FRegisterArray::GetImplementation`（F-HLP-006）**完全同形**。

**调用上下文**
`Get`/`Find`/`FindKey` 的"未找到"是**常态**（C# 的 `map.TryGetValue`/`ContainsKey` 语义）。`GetEnumeratorKey/Value` 的 `nullptr` 则来自"枚举器传了失效索引"，例如 C# 在枚举过程中修改了 map（`TMap` 的枚举器不会版本校验）。

**问题**
1. "未找到"路径会把 `nullptr` 送进 `FPropertyDescriptor::Get(void* Src, void** Dest, ...)`。基类实现是空的（`FPropertyDescriptor.cpp:140-155`），但子类会解引用：`FStrPropertyDescriptor::Get(..., FPropertyArgument::FMember)` 把 `Src` 交给 `FCSharpEnvironment::GetEnvironment().GetStringObject<FString>(Src)`（`FStrPropertyDescriptor.cpp:6`）；`FObjectPropertyDescriptor` 等同理 → **空指针解引用崩溃**。
   也就是说 C# 的 `map[key]`（键不存在）**不是返回默认值，而是崩 native 进程**。
2. `Remove` 的返回值语义也不对称：`FRegisterMap.cpp:82` 在 helper 为空时 `return 0`，而 helper 内部找不到键时返回 `Count == 0`（`FMapHelper.cpp:127`）——这一处是自洽的。
3. `GetEnumeratorKey/Value` 的 `nullptr` 完全没有被上游检查，而枚举是 C# 侧最容易出现"边遍历边修改"的场景。

**建议**
1. 在 `FRegisterMap` 的 5 个 `*Implementation` 里统一补判空（与 F-HLP-006 的建议相同）：
```cpp
				const auto Value = MapHelper->Get(IN_KEY_BUFFER);
				if (Value == nullptr)
				{
					UE_LOG(LogUnrealCSharp, Warning, TEXT("TMap.Get: key not found"));
					return;      // 或向 RETURN_BUFFER 写入 value 类型的默认值
				}
				MapHelper->GetValuePropertyDescriptor()->Get(Value, reinterpret_cast<void**>(RETURN_BUFFER));
```
2. 更彻底：给 `FPropertyDescriptor` 增加 `GetDefault(void** Dest)`（用 `FProperty::InitializeValue` 在一块临时缓冲上造默认值），让"未找到"能返回类型正确的默认值 —— 这正是 C# 侧 `TryGetValue`/`GetValueOrDefault` 需要的语义。
3. `GetEnumeratorKey/Value` 的 `IsValidIndex` 已经做了判断（`:253`、`:260`），问题在上游不检查返回值；建议这两个方法改为 `checkf` 或返回一个稳定的"空槽位"。

**验证方式**
C# 用例：`var m = new TMap<int,int>(); var v = m[42];`（键不存在）与 `m.FindKey(1)` —— 当前应在 `FIntPropertyDescriptor::Get` 内崩溃；补判空后应得到默认值与日志。

---

### 7.4 FSetHelper

构造点（grep 证据：`grep -rn "new FSetHelper(" Source/`）：`FSetPropertyDescriptor.cpp:39`、`:56`（`bNeedFreeProperty = false`）；`TPropertyValue.inl:746`、`:769`（`false, true`）、`:776`（`true, true` + `new std::decay_t<T>(*InMember)` 拷贝）。C# 侧注册在 `FRegisterSet.cpp:116-130`，`FClassBuilder(TEXT("TSet"), NAMESPACE_LIBRARY)`，共 **11 个** P/Invoke 入口。

| 方法签名 | 文件:行 | 做了什么 | 调用方（证据） | 结论 |
|---|---|---|---|---|
| `FSetHelper(FProperty* InProperty, void* InData, bool, bool)` | `FSetHelper.cpp:5-28` | `InData` 非空则当 `FScriptSet*`，否则 `new FScriptSet()`；**仅当 `InProperty != nullptr` 时**建 `ElementPropertyDescriptor` 并算 `ScriptSetLayout` | `FSetPropertyDescriptor.cpp:39`、`:56`；`TPropertyValue.inl:746`、`:769`、`:776` | **有问题 → F-HLP-024（`ScriptSetLayout` 未初始化 + `Factory` 无判空）** |
| `~FSetHelper()` | `:30-33` | 仅调用 `Deinitialize()` | 隐式 | 无问题 |
| `Initialize()` | `:35-37` | **空函数体** | 精确模式 `SetHelper->Initialize` 无命中 | 死代码 → F-HLP-030 |
| `Deinitialize()` | `:39-56` | 条件 `delete ScriptSet`；条件 `DestroyProperty()`+`delete` 元素描述符 | `:32` | 有问题 → F-HLP-026 |
| `Empty(int32)` | `:58-61` | `ScriptSet->Empty(InExpectedNumElements, ScriptSetLayout)` —— **不析构元素** | `FRegisterSet.cpp:33`（注册名 `Empty`，`:121`） | **有问题 → F-HLP-025（泄漏所有元素）** |
| `Num()` | `:63-66` | `ScriptSet->Num()` | `FRegisterSet.cpp:41`；被 `IsEmpty`（`:70`）内部调用 | 无问题 |
| `IsEmpty()` | `:68-71` | `Num() == 0` | `FRegisterSet.cpp:51` | 无问题 |
| `GetMaxIndex()` | `:73-76` | `ScriptSet->GetMaxIndex()` | `FRegisterSet.cpp:61` | 无问题 |
| `Add(void* InValue)` | `:78-113` | **线性扫**找同值；未找到则 `AddUninitialized(Layout)` → `ElementPropertyDescriptor->Set(InValue, Data)` → **全表 `Rehash`**；已存在则什么都不做（集合语义正确） | `FRegisterSet.cpp:71` | 有问题 → F-HLP-028（O(n) + 每次 `Rehash`） |
| `Remove(const void* InValue)` | `:115-145` | **线性扫**找同值；未找到返回 0；找到则 `DestroyValue(Data)` + `ScriptSet->RemoveAt(ValueIndex, Layout)`，返回 1 | `FRegisterSet.cpp:79` | 有问题 → F-HLP-028 / F-HLP-029（无 `Rehash`） |
| `Contains(const void* InValue)` | `:147-162` | **第三次**复制同一段线性扫描（与 `Add`/`Remove` 逐字相同），返回 `bool` | `FRegisterSet.cpp:89` | 有问题 → F-HLP-028；重复代码 → §8 |
| `GetElementPropertyDescriptor()` | `:164-167` | 返回元素描述符（可能为 `nullptr`，见 F-HLP-024） | `FRegisterSet.cpp:112` | 无问题（返回体见 F-HLP-024） |
| `GetScriptSet()` | `:169-172` | 返回 `ScriptSet` | `FSetPropertyDescriptor.cpp:28`（`CopyCompleteValue(Dest, SrcContainer->GetScriptSet())`） | 无问题 |
| `GetAddress()` | `:174-177` | 返回 `ScriptSet` | `grep "SetHelper->GetAddress"` 无命中 | **疑似死代码** → F-HLP-030 |
| `IsValidIndex(int32)` | `:179-182` | `ScriptSet->IsValidIndex(InIndex)` | `FRegisterSet.cpp:99` | 无问题 |
| `GetEnumerator(int32)` | `:184-189` | `IsValidIndex ? GetData(Index, Layout) : nullptr` | `FRegisterSet.cpp:110`；`TPropertyValue.inl`（set 读回路径） | 有问题 → F-HLP-027（`nullptr` 被 `->Get` 无条件解引用） |

**注意**：`FSetHelper` 与 `FMapHelper` 一样**完全没有使用引擎提供的 `FScriptSetHelper`**（对比 `FArrayHelper` 用了 `FScriptArrayHelper`），因此哈希查找、元素析构、`Rehash` 时机全部由插件手工实现 —— 而这正是 F-HLP-025/028/029 的根源。

### 7.5 非平凡元素搬运方式对照表（FSetHelper 部分）

| 元素类型 | 搬运方式 | 正确？ | 文件:行 |
|---|---|---|---|
| 元素插入 | `ElementPropertyDescriptor->Set(InValue, Data)` —— 属性感知，**非 memcpy** | ✅ | `FSetHelper.cpp:102` |
| 元素删除 | 先 `ElementPropertyDescriptor->DestroyValue(Data)` 再 `ScriptSet->RemoveAt(...)` | ✅ | `FSetHelper.cpp:140`、`:142` |
| 元素比较（查重/查找） | `ElementPropertyDescriptor->Identical(Data, InValue)` 线性扫描 | ⚠️ 正确但 O(n) → F-HLP-028 | `FSetHelper.cpp:87`、`:124`、`:154` |
| 哈希 | `ElementPropertyDescriptor->GetValueTypeHash(Src)` 经 `ScriptSet->Rehash` | ✅ 语义正确；但每次插入都重算全表 → F-HLP-028 | `FSetHelper.cpp:105-111` |
| **`Empty` 清空** | `ScriptSet->Empty(...)` **不析构元素** | ❌ **泄漏** → F-HLP-025 | `FSetHelper.cpp:60` |
| `FString`/`FText`/嵌套容器元素 | 插入/删除走属性层（正确）；`Empty` 路径泄漏；`TPropertyValue.inl:776` 的整集拷贝在 `delete ScriptSet` 时整块泄漏 | ⚠️ 部分正确 | `FSetHelper.cpp:102`、`:140`、`:60`、`:43` |
| `UObject*` 元素 | Helper 是 native `FScriptSet` 的视图 → 借用路径受 GC 保护；`TPropertyValue.inl:776` 的拷贝脱离 GC → 悬垂 | ⚠️ | `FSetHelper.cpp:14`；`TPropertyValue.inl:776` |

**未使用 `memcpy`**：`FSetHelper.cpp` 全文（189 行）没有 `memcpy`/`FMemory::Malloc`；唯一的字节级操作是 `ScriptSetLayout` 的偏移算术（`:100`、`:138`、`:187`）。

### 7.6 FSetHelper 相关发现

> 本小节的发现编号为 F-HLP-024 ~ F-HLP-030，严重度分布（按各条 `严重度` 字段重算）：**P1 × 3**（024、025、026）、**P2 × 1**（028）、**P3 × 1**（030）、**撤销（非缺陷）× 2**（027、029），合计 7 条。按严重度排序的完整索引见 §5.0。

#### [F-HLP-024] `FSetHelper` 构造函数在 `InProperty` 为空时不初始化 `ScriptSetLayout`，且对 `Factory` 返回值无判空 —— 与 FMapHelper 同一缺陷的两个面

- **类别**: 未定义行为 / 崩溃
- **严重度**: **P1**
- **复核结论**: 确认 —— 两个面（`InProperty==nullptr` 时 `ScriptSetLayout` 未初始化；`Factory` 返回值无判空）均成立；严重度 P1 成立
- **可达性**: 活跃（触发条件与 F-HLP-016/017 同类：`FSetPropertyDescriptor.cpp:39`/`:56` 传 `Property->ElementProp`；`TPropertyValue.inl:746`/`:769`/`:776` 传 `FTypeBridge::Factory<>` 的结果）
- **复核证据**: `FSetHelper.h:48`（`FScriptSetLayout ScriptSetLayout;` 无初始化器）；`FSetHelper.cpp:7-10`（初始化列表未含 `ScriptSetLayout`）、`:21`（只判 `FProperty*`）、`:23`（`Factory`）、`:25-26`（解引用 `GetSize()`/`GetMinAlignment()`）；使用点 `:60`/`:86`/`:98`/`:100`/`:105`/`:123`/`:138`/`:142`/`:187`；`FPropertyDescriptor.cpp:133`（静默 `return nullptr;`）
- **级别变动**: 无（维持 P1）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FSetHelper.cpp:21-27`；成员声明 `Source/UnrealCSharp/Public/Reflection/Container/FSetHelper.h:48`
- **函数**: `FSetHelper::FSetHelper(FProperty*, void*, bool, bool)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FSetHelper.h:44-52
	FPropertyDescriptor* ElementPropertyDescriptor;

	FScriptSet* ScriptSet;

	FScriptSetLayout ScriptSetLayout;      // ← 未在构造初始化列表中初始化

	bool bNeedFreeData;

	bool bNeedFreeProperty;
```
```cpp
// FSetHelper.cpp:5-28
FSetHelper::FSetHelper(FProperty* InProperty, void* InData,
                       const bool InbNeedFreeData, const bool InbNeedFreeProperty) :
	ElementPropertyDescriptor(nullptr),
	ScriptSet(nullptr),
	// ScriptSetLayout 缺失
	bNeedFreeData(InbNeedFreeData),
	bNeedFreeProperty(InbNeedFreeProperty)
{
	...
	if (InProperty != nullptr)                                            // ← 只判 FProperty*
	{
		ElementPropertyDescriptor = FPropertyDescriptor::Factory(InProperty);   // 可能 nullptr

		ScriptSetLayout = FScriptSet::GetScriptLayout(ElementPropertyDescriptor->GetSize(),          // ← 解引用
		                                              ElementPropertyDescriptor->GetMinAlignment());
	}
}
```
`FScriptSetLayout` 的使用点：`Empty`（`:60`）、`Add`（`:86`、`:98`、`:100`、`:105`）、`Remove`（`:123`、`:138`、`:142`）、`GetEnumerator`（`:187`）。**全部没有判空/判有效。**

**调用上下文**
5 个构造点，其中 `TPropertyValue.inl:746`、`:769`、`:776` 的元素属性由 `FTypeBridge::Factory<>(FoundClass->GetGenericArgument(), ...)` 现场创建（`TPropertyValue.inl:739-742`、`:760-763`）——**是否为 `nullptr` 取决于 C# 泛型参数能否映射到受支持的 `FProperty`**。

**问题**
1. **空指针解引用（确定性崩溃）**：条件判的是 `FProperty*` 非空，而解引用的是 `FPropertyDescriptor*`（`Factory` 的返回值）。`Factory` 对白名单外类型静默 `return nullptr;`（`FPropertyDescriptor.cpp:133`）→ `:25` 立即崩溃。与 F-HLP-017（Map）逐字同形。
2. **未初始化 layout（内存损坏）**：若 `InProperty == nullptr`，则 `ElementPropertyDescriptor` 与 `ScriptSetLayout` **双双**保持 `nullptr`/不确定值，构造函数正常返回。此后 `Empty`（`:60`）会把不确定的 layout 传给 `FScriptSet::Empty`；`Add`（`:98`）会用不确定的 `SetLayout` 定位/分配槽位 → 在任意地址构造元素（`ElementPropertyDescriptor->Set` 会写 `FString`/`FText`）。与 F-HLP-016（Map）同形。
3. 这三个 Helper 的构造函数都采用了"条件赋值 + 无失败信号"的模式，说明这是**共享的设计缺陷**而非单点笔误（见 §8 的统一方案）。

**建议**
```cpp
FSetHelper::FSetHelper(FProperty* InProperty, void* InData,
                       const bool InbNeedFreeData, const bool InbNeedFreeProperty) :
	ElementPropertyDescriptor(nullptr),
	ScriptSet(nullptr),
	ScriptSetLayout{},                                    // ← 零初始化
	bNeedFreeData(InbNeedFreeData),
	bNeedFreeProperty(InbNeedFreeProperty)
{
	checkf(InProperty != nullptr, TEXT("FSetHelper requires an element property"));
	ElementPropertyDescriptor = FPropertyDescriptor::Factory(InProperty);
	checkf(ElementPropertyDescriptor != nullptr,
	       TEXT("FSetHelper: unsupported element property '%s'"), *InProperty->GetName());
	ScriptSetLayout = FScriptSet::GetScriptLayout(ElementPropertyDescriptor->GetSize(),
	                                              ElementPropertyDescriptor->GetMinAlignment());
	...
}
```
并在 `FSetPropertyDescriptor::NewRef/NewWeakRef`（`:39`、`:56`）与 `TPropertyValue::Get` 侧对 `Property`/`FTypeBridge::Factory` 结果做前置校验。

**验证方式**
同 F-HLP-016/017：`grep -n "ScriptSetLayout" Source/UnrealCSharp/Private/Reflection/Container/FSetHelper.cpp`；构造 `TSet<不支持类型>` 观察 `:25` 崩溃；临时把 `if (InProperty != nullptr)` 改成 `if (false)` 观察 `:60` 处读到垃圾 layout。

#### [F-HLP-025] `FSetHelper::Empty` 不析构元素 → 清空集合泄漏全部元素资源

- **类别**: 内存/资源泄漏
- **严重度**: **P1**
- **复核结论**: 确认 —— 机制、调用上下文、类别、严重度 P1 全部成立；"引擎 `Empty` 只归还内存、不析构元素"已由引擎源码确证
- **可达性**: 活跃（`FRegisterSet.cpp:29-35`，注册名 `Empty`，`:121`；C# `TSet<T>.Empty(slack)`，`TSet.cs:36-37`）
- **复核证据**: `FSetHelper.cpp:60`（`ScriptSet->Empty(...)`，全文无任何 `DestroyValue`）；同文件 `:140` 的手动 `DestroyValue` 对照；**引擎侧确证** `Containers/Set.h:1843-1865`（`TScriptSet::Empty`：只 `Elements.Empty(Slack, Layout.SparseArrayLayout)` + 重置/重建 `Hash`，**完全不触碰元素析构**）
- **级别变动**: 无（维持 P1）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FSetHelper.cpp:58-61`
- **函数**: `FSetHelper::Empty(int32)`
- **置信度**: 高（已用引擎源码确证："`FScriptSet::Empty` 不析构元素" —— `Containers/Set.h:1843-1865`；判据同 F-HLP-020：同文件 `Remove` 手动 `DestroyValue`，说明引擎 `RemoveAt`/`Empty` 都不负责析构）

**现状（代码事实）**
```cpp
// FSetHelper.cpp:58-61
void FSetHelper::Empty(const int32 InExpectedNumElements) const
{
	ScriptSet->Empty(InExpectedNumElements, ScriptSetLayout);      // ← 无任何 DestroyValue
}
```
```cpp
// FSetHelper.cpp:138-142   同一文件里的 Remove：明确手动析构
	const auto Data = static_cast<uint8*>(ScriptSet->GetData(ValueIndex, ScriptSetLayout));

	ElementPropertyDescriptor->DestroyValue(Data);                 // ← 元素手动析构

	ScriptSet->RemoveAt(ValueIndex, ScriptSetLayout);
```
对照 `FArrayHelper::Empty`（`FArrayHelper.cpp:244-249`）用的是**会析构**的 `FScriptArrayHelper::EmptyValues`。

**调用上下文**
`FRegisterSet.cpp:29-35`（注册名 `Empty`，`:121`）→ C# 的 `TSet<T>.Empty(slack)`。与 Map 的 `Empty` 一样是"复用同一集合对象"的必经路径。

**问题**
与 F-HLP-020 完全同形：若 `FScriptSet::Empty` 只归还内存（`Remove` 的写法表明作者相信引擎不做析构），则 `Empty` 跳过全部元素的析构 → `FString`/`FText`/`TArray`/`TMap` 元素的内部堆缓冲全部泄漏，泄漏量与清空前的元素数成正比，且每次 `Empty` 都发生。对 `UObject*` 元素，则少了引擎侧的引用处理时机。

**建议（与 F-HLP-020 合并修复）**
```cpp
void FSetHelper::Empty(const int32 InExpectedNumElements) const
{
	for (int32 Index = 0; Index < ScriptSet->GetMaxIndex(); ++Index)
	{
		if (ScriptSet->IsValidIndex(Index))
		{
			ElementPropertyDescriptor->DestroyValue(
				static_cast<uint8*>(ScriptSet->GetData(Index, ScriptSetLayout)));
		}
	}
	ScriptSet->Empty(InExpectedNumElements, ScriptSetLayout);
}
```
更推荐改用引擎的 `FScriptSetHelper::EmptyValues`（与 `FScriptArrayHelper::EmptyValues` 对称），一次性解决 F-HLP-025/028/029。

**验证方式**
C# 用例：`var s = new TSet<string>(); for(i<100000) s.Add(k_i); s.Empty(0);` 循环 100 次，用 `MemReport` 观察 `FString` 分配量线性增长；或 C++ 单测断言 "`Empty` 触发的 `FString` 析构次数 == 元素个数"。

#### [F-HLP-026] `delete ScriptSet` 经 `FScriptSet*` 释放真正的 `TSet<T>` → **元素自有资源**泄漏（三 Helper 同一 bug 的第三处）

- **类别**: 内存/资源泄漏 | 未定义行为
- **严重度**: **P1**
- **复核结论**: 部分确认（偏差：**"`delete` 静态类型不匹配 → `~TSet<T>()` 不执行 → 元素析构被跳过"成立**；但"`Elements`/`Hash` 两块分配以及每个元素自身的资源全部泄漏"中**前两项被引擎源码证伪**）
- **可达性**: 活跃（`TPropertyValue.inl:776` 是唯一 `bNeedFreeData=true` 的 set 构造点）
- **复核证据**: `FSetHelper.cpp:14`（`static_cast<FScriptSet*>(InData)`）、`:41-46`（`delete ScriptSet`）；`TPropertyValue.inl:776`；**引擎侧**：`Containers/Set.h:2115-2129`（`FScriptSet` 未声明析构 → `~TSet<T>()` 不会被调用，元素析构确实被跳过）、`:2049-2054`（`ElementArrayType Elements; HashType Hash;` 两个**持有 allocator 的成员**，其 `~ForAnyElementType` 会 `Free(Data)`，见 `Containers/ContainerAllocationPolicies.h:683-693`）、`:2072-2090`（`static_assert(sizeof(TScriptSet)==sizeof(TSet))` 逐成员断言 → **布局等价由引擎保证**）
- **级别变动**: 无（维持 P1）；**范围修正**：泄漏的是每个元素的**自有资源**（`FString`/`FText` 缓冲、嵌套容器、`UObject*` 的引用处理时机）；对 `TSet<int32>` 这类无自有资源元素**不产生泄漏**
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FSetHelper.cpp:12-19`、`:41-46`；`Source/UnrealCSharp/Public/Binding/Core/TPropertyValue.inl:776`
- **函数**: `FSetHelper::Deinitialize()` / `FSetHelper::FSetHelper(...)`
- **置信度**: 高（插件侧确证）；高（**布局兼容性已由引擎源码确证**：`Containers/Set.h:2072-2090` 的 `static_assert`；`Containers/ContainerAllocationPolicies.h:683-693` 确证 allocator 基类析构会释放缓冲）

**现状（代码事实）**
```cpp
// FSetHelper.cpp:12-19
	if (InData != nullptr)
	{
		ScriptSet = static_cast<FScriptSet*>(InData);      // 真实类型可能是 TSet<T>
	}
	else
	{
		ScriptSet = new FScriptSet();
	}
// FSetHelper.cpp:41-46
	if (bNeedFreeData && ScriptSet != nullptr)
	{
		delete ScriptSet;                                  // 无虚析构 → ~TSet<T>() 不执行
		ScriptSet = nullptr;
	}
```
```cpp
// TPropertyValue.inl:774-777
		else
		{
			const auto SetHelper = new FSetHelper(Property, new std::decay_t<T>(*InMember), true, true);
			//                                             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ 真正的 TSet<T>
```

**调用上下文**
`bNeedFreeData = true` 只有 `TPropertyValue.inl:776` 一处（C# 以传值语义交出 `TSet<T>`）。

**问题**
与 F-HLP-001（Array）、F-HLP-019（Map）完全同形：`delete` 的静态类型没有虚析构函数，`~TSet<T>()` 不被调用，**每个元素自身的资源全部泄漏**。**修正**：`Elements`/`Hash` 两块分配**不会**泄漏 —— 它们是 `TScriptSet` 的成员（`Containers/Set.h:2052-2053`），其 allocator 基类析构会 `Free(Data)`（`Containers/ContainerAllocationPolicies.h:683-693`）。**同一个 bug 在三个 Helper 里各写了一遍**，说明缺少统一的容器所有权抽象（见 §8）。

**建议**：同 F-HLP-001/019（类型擦除销毁器 + `static_assert` 布局检查），并作为**一次**修复覆盖三处。

**验证方式**：同 F-HLP-019，把 `TMap` 换成 `TSet<FString>`。

#### [F-HLP-027] `FSetHelper::GetEnumerator` 的 `nullptr` 被 `FRegisterSet` 无条件交给属性描述符 `Get` 解引用 —— **撤销（非缺陷）：机制成立但单线程不可达，请从排期移除**

- **类别**: 未定义行为 / 崩溃
- **严重度**: **撤销（非缺陷）**
- **复核结论**: 部分确认（偏差：`GetEnumerator` 对失效索引返回 `nullptr`、`FRegisterSet.cpp:112` 不判空 —— 这两个**代码事实成立**；但原文"**遍历期修改集合即触发**"的**可达性论断不成立**，故不作待修缺陷）
- **可达性**: 不可达（C# 枚举器在**同一线程内**先 `IsValidIndex(Index)` 再访问 `this[Index]`，两次 P/Invoke 之间没有任何可改变索引有效性的 native 调用；只有并发修改才会命中，而并发修改 UE 容器本身即未定义）
- **复核证据**: `FSetHelper.cpp:184-189`；`FRegisterSet.cpp:110-112`；**决定性反证** `Script/UE/CoreUObject/TSet.cs:14-23`（`for (var Index = 0; Index < Num(); Index++) { if (IsValidIndex(Index)) yield return this[Index]; }`，`:25-34` 同）与 `:117-139`（索引器内部即 `GetEnumeratorImplementation`）；**并修正原文一处事实错误**：原文称 C# 侧遍历 `TSet` 用 `GetMaxIndex()`，实际循环上界是 `Num()`（`TSet.GetMaxIndex()` 只是在 `:43` 作为公开方法存在，枚举器不使用；`TMap.cs:16` 的枚举器才用 `GetMaxIndex()`）。**附带发现（存疑项，未编号）**：由于上界是 `Num()` 而非 `GetMaxIndex()`，含空洞的 `TSet`（例如 `{0,5}` → `Num()==2`、`GetMaxIndex()==6`）在 C# 侧 `foreach` 会**漏掉**索引 ≥ `Num()` 的元素，这是 C# 绑定的独立问题，不属本报告 6 文件范围，已记入 §9.4
- **级别变动**: 无（维持 `撤销（非缺陷）`）；**撤销理由更正**：代码事实是成立的，真正不成立的是**可达性** —— 正确判定是"部分确认 + 不可达"
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FSetHelper.cpp:184-189`；`Source/UnrealCSharp/Private/Domain/Interop/FRegisterSet.cpp:105-113`
- **函数**: `FSetHelper::GetEnumerator(int32)` / `FRegisterSet::GetEnumeratorImplementation`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FSetHelper.cpp:184-189
void* FSetHelper::GetEnumerator(const int32 InIndex) const
{
	return ScriptSet->IsValidIndex(InIndex)
		       ? static_cast<uint8*>(ScriptSet->GetData(InIndex, ScriptSetLayout))
		       : nullptr;                                   // ← 失效索引返回空
}
```
```cpp
// FRegisterSet.cpp:110-112
				const auto Value = SetHelper->GetEnumerator(InIndex);

				SetHelper->GetElementPropertyDescriptor()->Get(Value, reinterpret_cast<void**>(RETURN_BUFFER));
			}   //                                             ^^^^^ 可能为 nullptr
```
与 `FArrayHelper::Get` → `FRegisterArray.cpp:117`（F-HLP-006）、`FMapHelper::GetEnumeratorKey/Value` → `FRegisterMap.cpp:167/179`（F-HLP-023）同形。

**调用上下文**
`FRegisterSet.cpp:105`（注册名 `GetEnumerator`，`:129`）。C# 侧枚举 `TSet<T>` 时按 `GetMaxIndex()` + `IsValidIndex` + `GetEnumerator` 三段式遍历；**在遍历过程中修改集合**（或 native 侧修改）会让 `IsValidIndex` 与 `GetEnumerator` 的判断窗口不一致。

**问题**
"索引失效"是枚举器的**正常竞态结果**，不是异常路径，而当前实现把它转成 `nullptr` 后立刻解引用 → 原生崩溃。C# 侧无法防御（`GetEnumerator` 是 `extern`，托管侧拿不到"是否有效"的额外信号）。

**建议**
1. `FRegisterSet.cpp:110-113` 补判空并返回类型正确的默认值或记录错误（同 F-HLP-006/023 的统一建议）。
2. 更稳的方案是把"取值"合并成一个 C# 可见的 `TryGetEnumerator(index, out value)` 语义，或让 `GetEnumerator` 在失效时返回指向一个**静态默认缓冲**的指针（需注意线程安全）。
3. C# 侧枚举器应改为"快照枚举"（先 `Num()` 个元素复制出来），避免遍历期修改。

**验证方式**
C# 用例：`var s = new TSet<int>(); s.Add(1); s.Add(2); foreach (var x in s) { s.Remove(2); }` —— 当前应在 `FIntPropertyDescriptor::Get` 内崩溃。

#### [F-HLP-028] `FSetHelper` 的 `Add`/`Remove`/`Contains` 都是 O(MaxIndex) 线性扫描，且每次插入新元素都做全表 `Rehash`

- **类别**: 性能
- **严重度**: **P2**（同 F-HLP-021 的算法层缺陷）
- **复核结论**: 确认 —— 三处线性扫描 + 每次插入都全表 `Rehash` 的代码事实、类别、严重度 P2 均成立
- **可达性**: 活跃（`FRegisterSet.cpp:71`/`:79`/`:89` → C# `TSet<T>.Add`（`TSet.cs:45`）/`Remove`（`:68`）/`Contains`（`:91`））
- **复核证据**: `FSetHelper.cpp:82-94`（`Add` 线性查重）、`:96-98`（`AddUninitialized`）、`:102`（`Set`）、`:104-111`（**无条件** `Rehash`）、`:119-131`（`Remove` 线性扫描）、`:149-159`（`Contains` 第三份同样的扫描）；`FRegisterSet.cpp:71`/`:79`/`:89`；**引擎对照（确证"每次全表 Rehash"是插件选择）** `Containers/Set.h:2022-2047`（`AddNewElement` 仅在 `HashSize < Allocator::GetNumberOfHashBuckets(Num())` 时 `Rehash`，否则只把新元素挂到桶头）；引擎亦提供哈希查找 `Containers/Set.h:1944-1984`、`UnrealType.h:5597`
- **级别变动**: 无（维持 P2；P1→P2 的定级依据：正确性由线性扫描兜住，损失的是复杂度）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FSetHelper.cpp:82-94`、`:96-111`、`:119-131`、`:149-159`
- **函数**: `FSetHelper::Add` / `FSetHelper::Remove` / `FSetHelper::Contains`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FSetHelper.cpp:82-111   Add：先线性查重，再插入 + 全表 Rehash
	for (auto Index = 0; Index < ScriptSet->GetMaxIndex(); ++Index)
	{
		if (ScriptSet->IsValidIndex(Index))
		{
			if (const auto Data = static_cast<uint8*>(ScriptSet->GetData(Index, ScriptSetLayout));
				ElementPropertyDescriptor->Identical(Data, InValue))
			{
				ValueIndex = Index;
				break;
			}
		}
	}

	if (ValueIndex == INDEX_NONE)
	{
		ValueIndex = ScriptSet->AddUninitialized(ScriptSetLayout);

		const auto Data = static_cast<uint8*>(ScriptSet->GetData(ValueIndex, ScriptSetLayout));

		ElementPropertyDescriptor->Set(InValue, Data);

#if STD_CPP_20
		ScriptSet->Rehash(ScriptSetLayout, [=, this](const void* Src)
#else
		ScriptSet->Rehash(ScriptSetLayout, [=](const void* Src)
#endif
		                  {
			                  return ElementPropertyDescriptor->GetValueTypeHash(Src);
		                  });
	}
```
`Contains`（`:147-162`）与 `Remove`（`:119-131`）各自**复制了同一段扫描**（逐字相同，只有命中后的动作不同）。

**调用上下文**
`FRegisterSet.cpp:71`（`Add`）、`:79`（`Remove`）、`:89`（`Contains`）——C# 的 `TSet<T>.Add/Remove/Contains` 全部走这里。`Contains` 是 `TSet` 最常用的操作。

**问题**
1. `TSet` 的价值是平均 O(1) 的成员判定；当前为 O(MaxIndex)。由于 C# 侧 `Add` 需要先查重，向空集合插入 n 个元素是 **O(n²)**（还要叠加每次插入的全表 `Rehash`，见下）。
2. **每次插入新元素都 `Rehash` 全表**（`:105-111`）：`Rehash` 会遍历所有已占用槽位重新求哈希并重建哈希索引。插入 n 个元素的总成本因此是 O(n²)（甚至更高，因为 `Rehash` 内部还要分配）。UE 的做法是仅在容量增长时 `Rehash`，均摊 O(1)。
3. 循环上界 `GetMaxIndex()` 而非 `Num()`：插件从不收缩集合（`Remove` 不 `Shrink`、`Empty` 只在显式调用时归还），反复增删会把已分配槽位数推高，使"集合里只有 1 个元素"时仍扫描整个槽位数组。
4. 常数因子：`Identical` 对 `FString`/`FText` 元素会走属性层的 `Identical`（对 `FString` 是 `FStrPropertyDescriptor::Identical`，每次调用都要把托管句柄解析成 native `FString`，`FStrPropertyDescriptor.cpp:45-48`）。
5. 哈希代价已付但收益为零 —— 与 F-HLP-021 同一种"设计层错配"。

**建议**
改用引擎的 `FScriptSetHelper`（`FindIndex`/`AddElement`/`RemoveElement`/`EmptyValues`）——与 `FArrayHelper` 用 `FScriptArrayHelper` 的做法对齐：
```cpp
void FSetHelper::Add(void* InValue) const
{
	FScriptSetHelper Helper(ElementPropertyDescriptor->GetProperty(), ScriptSet, ScriptSetLayout);
	if (Helper.FindElementIndex(InValue) == INDEX_NONE)     // O(1) 平均
	{
		const int32 Index = Helper.AddDefaultValue_Invalid_NeedsRehash();  // 或 AddElement
		Helper.Rehash();                                     // 仅在需要时
		...
	}
}
```
若坚持手工实现，至少要：把 `Rehash` 改为"仅当 `MaxIndex` 增长时"调用，并把查重/查找改为基于 `ScriptSet->Hash` 的探测。

**验证方式**
C# 基准：`TSet<int>` 插入 10 万条（当前 O(n²)）与 `Contains` 10 万次；在 `ElementPropertyDescriptor->Identical` 与 `ScriptSet->Rehash` 上加计数器，断言"一次 `Contains` 的 `Identical` 次数应为 O(1)"、"插入 n 条的 `Rehash` 次数应为 O(log n)"。

#### [F-HLP-029] `FSetHelper::Remove` 在 `RemoveAt` 之后不做 `Rehash`，而 `Add` 每次都做 —— **证伪并撤销：引擎 `RemoveAt` 自己维护哈希链，不 `Rehash` 是正确的**

- **类别**: 一致性（**原文的"潜在 Bug / 未定义行为"部分已证伪并删除**）
- **严重度**: **撤销（非缺陷）**
- **复核结论**: 证伪 —— 原文的推理链（"`RemoveAt` 同样会改动哈希表结构…同样需要 `Rehash`"、"陈旧的哈希索引会立刻变成已删除的元素仍能被 `Contains` 命中"）与引擎源码**直接冲突**；`TScriptSet::RemoveAt` 本身就把该槽位从哈希链中摘除，因此不 `Rehash` 是**正确**的
- **可达性**: 不适用（撤销）
- **复核证据**: **决定性反证** `Runtime/Core/Public/Containers/Set.h:1867-1885`：`:1869` `check(IsValidIndex(Index))`；`:1871` 取被删元素地址；`:1873-1881` 沿 `HashNextId` 链找到前驱并把 `*NextElementId = GetHashNextIdRef(ElementBeingRemoved, Layout)`（**把哈希链改指向后继**，即完成摘除）；`:1884` 才 `Elements.RemoveAtUninitialized(...)`。即"`RemoveAt` 不更新哈希链"这一前提不成立。对照 `Set.h:1887-1897` 的注释（`AddUninitialized` **才**需要外部补 `Rehash`："The set will need rehashing at some point after this call to make it valid"）→ 插件"`Add` 侧补 `Rehash` / `Remove` 侧不补"的不对称**恰是引擎契约要求的**，不是缺陷。`FScriptMap` 同理：`Containers/Map.h:1960-1963` 的 `RemoveAt` 转调 `Pairs.RemoveAt`
- **级别变动**: 无（维持 `撤销（非缺陷）`）；撤销理由：**引擎源码证伪**"
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FSetHelper.cpp:140-142`（不 Rehash，**正确**）；`:104-111`（Rehash，**必需**）
- **函数**: `FSetHelper::Remove(const void*)` / `FSetHelper::Add(void*)`
- **置信度**: 高（已对照引擎源码：`Containers/Set.h:1867-1885` 证明 `RemoveAt` 自带哈希链摘除，`Set.h:1887-1897` 的注释证明只有 `AddUninitialized` 需要外部补 `Rehash`）

**现状（代码事实）**
```cpp
// FSetHelper.cpp:138-144   Remove：RemoveAt 后直接返回
	const auto Data = static_cast<uint8*>(ScriptSet->GetData(ValueIndex, ScriptSetLayout));

	ElementPropertyDescriptor->DestroyValue(Data);

	ScriptSet->RemoveAt(ValueIndex, ScriptSetLayout);

	return 1;
	// ← 无 Rehash
```
对照同一文件的 `Add`（`:105-111`）在 `AddUninitialized` 之后**必定** `Rehash`；`FMapHelper` 也是同样的不对称（F-HLP-022）。

**调用上下文**
`FRegisterSet.cpp:79`（`Remove`）与 `:71`（`Add`）。

**问题**
1. ~~如果引擎的 `AddUninitialized` 需要后续 `Rehash`…那么 `RemoveAt` 同样会改动哈希表结构…同样需要 `Rehash`~~ —— **该推理已证伪（见上"复核证据"）**：`Containers/Set.h:1867-1885` 显示 `RemoveAt` **自己**就完成了哈希链摘除，插件不补 `Rehash` 是正确的。保留此条仅为记录推理已被推翻，**不要再据此派工**。
2. 原文"这一点被线性扫描掩盖"的说法**不成立**：既然引擎已保证删除后哈希链一致，改为哈希查找不会暴露任何"已删除元素仍被命中"的问题（见 F-HLP-022 的第 2、3 点修正）。
3. ~~**修复顺序很重要**：F-HLP-028 与 F-HLP-029 必须一起修~~ —— **该约束已解除**：F-HLP-028（改用哈希）可以独立进行，不需要先给 `Remove` 补 `Rehash`。

**建议**
1. 首选：整体改用 `FScriptSetHelper`，由引擎维护哈希索引，插件不再手工 `Rehash`（引擎侧同样不必在 `Remove` 后补）。
2. ~~保守方案：`Remove` 在 `RemoveAt` 之后补 `Rehash`~~ —— **撤销**：该方案建立在错误前提上，补 `Rehash` 只会增加无谓开销。真正需要保留的是 `Add` 侧的 `Rehash`（或改用 `FScriptSetHelper::AddElement` 让引擎按需 `Rehash`，见 `Containers/Set.h:2022-2047`）。

**验证方式**
C# 用例（**回归保障，而非当前会失败的用例**）：`s.Add("a"); s.Add("b"); s.Remove("a"); Debug.Assert(!s.Contains("a"));` —— 当前线性扫描下通过；按 F-HLP-028 改成哈希查找后**仍应通过**（引擎 `RemoveAt` 已摘除哈希链，`Containers/Set.h:1873-1881`）。计数器断言："向 n 条空集插入 n 条时 `Rehash` 次数应为 O(log n)"（针对 F-HLP-028），不要断言"`Add`/`Remove` 的 `Rehash` 次数应相同"。

#### [F-HLP-030] `FSetHelper::Initialize()` 空实现、`GetAddress()` 无调用点、`static_cast<int32>(INDEX_NONE)` 与 `auto Count = 0` 的类型冗余

- **类别**: 死代码 / 可读性
- **严重度**: **P3**
- **复核结论**: 部分确认（偏差：空 `Initialize()` 与 `static_cast<int32>(INDEX_NONE)` 冗余成立；但 **`GetAddress()` 不是"无调用点"** —— 真实调用点是泛型指针调用 `FContainerRegistry.inl:62`）
- **可达性**: 活跃（`GetAddress()` 在每次句柄失效时都会被执行）
- **复核证据**: `FSetHelper.cpp:35-37`（空体）、`:174-177`（`GetAddress`）、`:80`/`:94`/`:117`/`:126`/`:176`（`static_cast<int32>(INDEX_NONE)` 冗余）、`:90`（`auto Count = 0`）；`grep "Helper->Initialize" Plugins/UnrealCSharp/` = **0**；`grep "GetAddress"` 命中 `Registry/FContainerRegistry.inl:62`，`FSetHelper` 的特化见 `FContainerRegistry.inl:111-122`、类型绑定见 `FContainerRegistry.h:23`
- **级别变动**: 无（维持 P3）；**本条结论范围收缩**：仅"空 `Initialize()` + 冗余 `static_cast`"仍是缺陷面，"`GetAddress()` 无调用点"这半边**撤销（证伪）**
- **文件**: `Source/UnrealCSharp/Private/Reflection/Container/FSetHelper.cpp:35-37`、`:174-177`、`:80`、`:94`、`:117`、`:126`、`:176`
- **函数**: `FSetHelper::Initialize()` / `FSetHelper::GetAddress()` / `FSetHelper::Add` / `FSetHelper::Remove`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FSetHelper.cpp:35-37
void FSetHelper::Initialize()
{
}                                    // ← 空实现，与 FArrayHelper/FMapHelper 相同

// FSetHelper.cpp:174-177
void* FSetHelper::GetAddress() const
{
	return ScriptSet;                // ← grep "SetHelper->GetAddress" 无命中
}
```
```cpp
// FSetHelper.cpp:80 / :94
	auto ValueIndex = static_cast<int32>(INDEX_NONE);        // INDEX_NONE 已是 int32，static_cast 冗余
	if (ValueIndex == INDEX_NONE)
```
（`FMapHelper.cpp:94`、`:176` 有同样的 `static_cast<int32>(INDEX_NONE)`；`auto Count = 0` 出现在 `:90`，与返回类型 `int32` 不一致但无害。）

**调用上下文**
- `Initialize()`：三个 Helper 都有这个空方法且**都没有任何调用点**（`grep -rn "Helper->Initialize" Source/ Script/` 无命中）。
- `GetAddress()`：三个 Helper 都有，`FRegisterArray`/`FRegisterMap`/`FRegisterSet` 的 `FClassBuilder` 都**没有注册** `GetAddress`；`grep -rn "->GetAddress()" Source/` 命中集中在其他类型（非本三 Helper）。属公开 API 但内部无使用者 → **可能是外部 API，需确认**（头文件带 `UNREALCSHARP_API` 且插件有对外导出属性，故按规范标为"疑似"而非"确定"）。

**问题**
1. 空 `Initialize()` 存在于三个公开头文件上，会诱导调用者写出 `helper->Initialize()` 这样的无效调用；同时它是 `Deinitialize()` 的"对称装饰"，掩盖了"构造即完成初始化"这一真实契约。
2. `GetAddress()` 返回 `ScriptSet`/`ScriptMap`/`ScriptArray`（即容器指针）而非构造参数 `InData`；在"自建容器"路径下（F-HLP-004/016/024 的空属性分支）返回值与调用方直觉不符。
3. `static_cast<int32>(INDEX_NONE)` 等冗余转换是风格问题，但这类"防御性写法"容易掩盖真正需要防御的地方（本文件的真正问题恰恰是缺失的判空与边界检查）。

**建议**
- 三个 Helper 一并删除 `Initialize()`（`.h` 声明 + `.cpp` 空体），或改名为 `CheckInvariants()` 并在构造末尾调用（配合 F-HLP-002/016/017/024 的 `checkf`）。
- `GetAddress()` 与 `GetScriptArray/Map/Set()` 重复，保留后者并删除前者；若确实需要外部 API，则在头文件上写明"返回底层引擎容器指针"。
- 清理 `static_cast<int32>(INDEX_NONE)`。

**验证方式**
`grep -rn "->Initialize()" Source/ Script/ | Select-String -NotMatch "InitializeValue"`；`grep -rn "GetAddress" Source/ Script/`。

---

## 8. 三个 Helper 的重复代码与统一方案

本节的目的是把 §5 与 §7 中的发现**收敛成少数几处结构性改动**，而不是逐条打补丁。下面是逐项的"几乎相同逻辑"清单与可提取的抽象。

### 8.1 已经逐字重复或同构的代码（可量化的重复）

| # | 重复内容 | 位置 | 重复次数 | 性质 |
|---|---|---|---|---|
| R1 | `if (InData != nullptr) Xxx = static_cast<FScriptXxx*>(InData); else Xxx = new FScriptXxx();` | `FArrayHelper.cpp:13-20`、`FMapHelper.cpp:12-19`、`FSetHelper.cpp:12-19` | 3 | 逐字同构，只有类型不同 |
| R2 | `~Helper(){ Deinitialize(); }` + `Deinitialize(){ if(bNeedFreeData) delete Xxx; if(bNeedFreeProperty) DestroyProperty+delete Desc; }` | `FArrayHelper.cpp:23-49`、`FMapHelper.cpp:34-66`、`FSetHelper.cpp:30-56` | 3 | 同构；Map 版本还多一个 `&&` 联判 bug（F-HLP-018） |
| R3 | 空 `Initialize(){}` | `FArrayHelper.cpp:28-30`、`FMapHelper.cpp:39-41`、`FSetHelper.cpp:35-37` | 3 | 逐字相同（且都是死代码，F-HLP-013/030） |
| R4 | `void* GetAddress() const { return Xxx; }` | `FArrayHelper.cpp:344-347`、`FMapHelper.cpp:236-239`、`FSetHelper.cpp:174-177` | 3 | 逐字同构（都是死代码） |
| R5 | `int32 Num() const` / `bool IsEmpty() const { return Num()==0; }` | Array `:106-116`、Map `:73-81`、Set `:63-71` | 3 | 同构 |
| R6 | `IsValidIndex` 转调引擎 | Array `:99-104`、Map `:246-249`、Set `:179-182` | 3 | 同构（Array 多一次 `CreateHelperFormInnerProperty`） |
| R7 | **"线性扫描 + `Descriptor->Identical`"查找骨架** | Array `Find/FindLast/Contains/RemoveSingle`（`:141-187`、`:286-302`）；Map `Remove`/`FindKey`/`Get`/`Set`（`:96-108`、`:132-142`、`:159-169`、`:178-190`）；Set `Add`/`Remove`/`Contains`（`:82-94`、`:119-131`、`:149-159`） | **14 处** | **最高价值的一处重复**：Set 的 `Add`/`Remove`/`Contains` 三段扫描**逐字相同**（只有命中后的动作不同）；Map 的 `Set`/`Get`/`Remove` 也是同一段 |
| R8 | `AddUninitialized(Layout)` → `Descriptor->Set(...)` → `Rehash(HashFn)` 序列 | Map `:196-209`、Set `:98-111` | 2 | 逐字同构（连 `#if STD_CPP_20` 的 lambda 捕获写法都一样） |
| R9 | `EnumeratorKey/Value`（Map）与 `GetEnumerator`（Set）的 `IsValidIndex ? GetData : nullptr` | Map `:251-263`、Set `:184-189` | 3 | 同构，且同样都缺上游判空（F-HLP-023/027） |
| R10 | `bNeedFreeData`/`bNeedFreeProperty` 两个 `bool` 形参在 15 个构造点被手工填写 | `FArrayHelper.cpp:4`、`FMapHelper.cpp:4`、`FSetHelper.cpp:5` 及 12 个调用点 | 15 | 语义靠位置，已导致 F-HLP-003（3 处写 `false` 造成泄漏） |
| R11 | `FRegister{Array,Map,Set}.cpp` 里 `GetContainer<T>(handle)` + `if (const auto X = ...)` 的空检查样板 | `FRegisterArray.cpp`（29 处）、`FRegisterMap.cpp`（14 处）、`FRegisterSet.cpp`（9 处） | **52 处** | 逐字样板；且**取值类的空检查缺失正是在这里**（F-HLP-006/023/027） |

### 8.2 统一方案（按"投入产出比"排序）

**方案 A（推荐，收益最大）：提取 `TContainerHelperBase<TScriptType, TLayoutType>` 模板基类**

把 R1/R2/R3/R4/R5/R6/R10 一次收掉。示意：

```cpp
// Source/UnrealCSharp/Public/Reflection/Container/ContainerHelperBase.h
enum class EOwnership : uint8 { Borrowed, Owned };

template <typename TScriptContainer, typename TLayout>
class TContainerHelperBase
{
public:
	TScriptContainer* GetScriptContainer() const { return ScriptContainer; }
	void* GetAddress() const { return ScriptContainer; }
	int32 Num() const { return ScriptContainer->Num(); }
	bool IsEmpty() const { return Num() == 0; }

protected:
	// 唯一允许赋值 ScriptContainer / Layout 的地方，保证不变式
	void InitializeContainer(void* InData, EOwnership InDataOwner,
	                         TLayout&& InLayout, EOwnership InLayoutRelevance);

	void DeinitializeContainer()
	{
		if (DataOwner == EOwnership::Owned && ScriptContainer != nullptr)
		{
			// 注意：必须调用真实类型的析构（见 F-HLP-001/019/026）
			if (DataDeleter) { DataDeleter(ScriptContainer); }
			else             { delete ScriptContainer; }
			ScriptContainer = nullptr;
		}
	}

	TScriptContainer* ScriptContainer = nullptr;
	TLayout           Layout{};
	EOwnership        DataOwner = EOwnership::Borrowed;
	void            (*DataDeleter)(void*) = nullptr;   // 类型擦除销毁器
};
```

- 收益：删除 3 份重复的 `Initialize`/`GetAddress`/`Num`/`IsEmpty`；把 `EOwnership` 枚举替代两个 `bool`（修 F-HLP-003 的成因）；`Layout{}` 默认零初始化 + 构造函数强制赋值（修 F-HLP-016/024）；`DataDeleter` 一次修掉 F-HLP-001/019/026。
- 代价：需要一次涉及 3 个 Helper + 3 个 `F*PropertyDescriptor` + `TPropertyValue.inl` 的重构；`Layout` 对 `FArrayHelper` 不适用（可用 `std::monostate` 或给 Array 一个只用 TScriptContainer 的偏特化）。
- 风险点：`ScriptContainer` 的"真实类型"是模板参数无法保证的（调用方用 `void*` 传进来），因此 `InitializeContainer` 的入参应改为 `TScriptContainer*`（由调用方 `static_cast<TScriptContainer*>`），把类型擦除限制在**一处**（`TPropertyValue.inl:858/697/776`），而不是像现在这样在每个 `Deinitialize` 里猜。

**方案 B（投入小、收益集中）：把 R7 的 14 处扫描骨架抽成一个私有模板函数**

三个 Helper 各自加一个私有成员：

```cpp
	/** 返回第一个满足谓词的槽位索引，找不到返回 INDEX_NONE */
	template <typename TSlotPredicate>
	int32 FindSlot(TSlotPredicate&& Pred) const
	{
		const int32 MaxIndex = GetMaxIndex();
		for (int32 Index = 0; Index < MaxIndex; ++Index)
		{
			if (IsValidIndex(Index) && Pred(Index)) { return Index; }
		}
		return INDEX_NONE;
	}
```
然后：
```cpp
int32 FSetHelper::Remove(const void* InValue) const
{
	const int32 Index = FindSlot([&](int32 I)
	{
		return ElementPropertyDescriptor->Identical(GetElement(I), InValue);
	});
	if (Index == INDEX_NONE) { return 0; }
	ElementPropertyDescriptor->DestroyValue(GetElement(Index));
	ScriptSet->RemoveAt(Index, ScriptSetLayout);
	return 1;
}
```
- 收益：14 处 → 3 处（每个 Helper 一个 `FindSlot`）+ 各自谓词；同时**顺带统一修掉**"循环条件里反复求 `MaxIndex()`"的同类问题（F-HLP-010 的同型问题在 Map/Set 上也存在）。
- 代价极小，且不改变语义，可以作为**第一步**落地（此时仍是线性扫描，先不改行为，只去重）。

**方案 C（必须与 B 之后的改哈希一起做）：改用引擎的 `FScriptMapHelper` / `FScriptSetHelper`**

这是解决 §7.3/§7.6 中 6 条发现（F-HLP-020/021/022/025/028/029）最彻底的方式，也让三个 Helper 的设计对齐（Array 已经用了 `FScriptArrayHelper`）：

| 现在的实现 | 引擎等价物 | 同时修掉 |
|---|---|---|
| `for (…MaxIndex…) Identical` 查找 | `FScriptMapHelper::FindPair` / `FScriptSetHelper::FindElementIndex`（哈希驱动） | F-HLP-021 / F-HLP-028 |
| `AddUninitialized` + 手工 `Set` + 手工 `Rehash` | `FScriptMapHelper::AddPair` / `FScriptSetHelper::AddElement`（内部按需 `Rehash`） | F-HLP-021 / F-HLP-022 / F-HLP-028 |
| `DestroyValue` ×2 + `RemoveAt`（无 `Rehash`） | `FScriptMapHelper::RemovePair` / `FScriptSetHelper::RemoveElement` | F-HLP-022 / F-HLP-029 |
| `ScriptMap->Empty(...)` | `FScriptMapHelper::EmptyValues(...)` | F-HLP-020 / F-HLP-025 |

**修复顺序（关键，勿并行）**：B（纯去重，不改行为）→ 补 F-HLP-006/023/027 的判空（消除 P0）→ C（改哈希实现）。**不要先做 C 再做判空**：C 会把"哈希索引是否陈旧"从被掩盖状态变成可观测状态（F-HLP-022/029），而 P0 的判空与 C 无耦合、收益最高、风险最低，应最先做。

**方案 D（一处改动消灭三条 P1 泄漏）：统一"容器所有权 + 类型擦除销毁器"**

三个 `Deinitialize()` 与三个 `TPropertyValue.inl` 分配点（`:858`/`:697`/`:776`）改成：

```cpp
// 分配点（以 TArray 为例）
auto* Copy = new std::decay_t<T>(*InMember);
const auto Helper = new FArrayHelper(
	Property, Copy,
	EOwnership::Owned, [](void* P) { delete static_cast<std::decay_t<T>*>(P); },
	EOwnership::Owned, nullptr);

// 三个 Deinitialize() 统一走 base 的 DeinitializeContainer()（见方案 A）
```
配套加编译期检查（把"布局兼容"这一隐含假设显式化）：
```cpp
static_assert(sizeof(std::decay_t<T>) == sizeof(FScriptArray), "FScriptArray layout assumption broken");
```

**方案 E（R11，可选）：`FRegister*` 的空检查样板**

52 处 `if (const auto X = GetEnvironment().GetContainer<T>(handle))` 可以抽成一个宏或一个 `WithContainer<T>(handle, [](T* H){ ... })` 辅助函数，把"句柄无效"的处理统一（当前 52 处各自 `return 0` / `return INDEX_NONE` / 什么都不做，返回值不统一）。这一项与 F-HLP-006/023/027 的修复天然重合，建议合并进行。

### 8.3 重复代码之外的一致性建议

1. **`Remove` 的返回值语义**：`FArrayHelper::Remove` 返回删除个数（可能 >1）、`FMapHelper::Remove` 也返回个数、`FSetHelper::Remove` 返回 0/1。三者与 UE 的 `TArray::Remove`/`TMap::Remove`/`TSet::Remove` 语义一致，**不建议改**，但应在 C# 侧文档中显式写明。
2. **命名**：`FMapHelper::FindKey` 的形参叫 `InValue`（`FMapHelper.cpp:130`）而其他查找方法的形参叫 `InKey`/`InValue`（`:147`、`:152`、`:157`），三个 Helper 的 `Find` 家族命名不统一；`Swap`/`SwapMemory` 命名与实现不符（F-HLP-012）。
3. **`__STDCPP_DEFAULT_NEW_ALIGNMENT__` 的重复出现**（`FArrayHelper.cpp:198`、`:222`、`:226`、`:259`、`:276`、`:326`、`:331`）应改为 `alignof(std::max_align_t)` 或从 `FProperty::GetMinAlignment()` 取——当前硬编码"默认 new 对齐"，对 `alignas(32)` 的元素类型（如 SIMD 向量 `USTRUCT`）会给出**错误的对齐**，这是 F-HLP-005/F-HLP-007 之外的一处独立隐患（**置信度: 中**，因为 `FScriptArray` 的分配是否真的按该参数对齐取决于引擎侧实现，我未能验证引擎源码）。

---

## 9. 未覆盖/存疑项

### 9.1 已确认未能验证的事项（结论中已标注置信度）

| # | 未验证内容 | 影响的发现 | 为什么未验证 | 如何验证 |
|---|---|---|---|---|
| U1（**已作废，见 §0.3 行 52**） | ~~"本机没有可访问的 UE 引擎源码"~~ **该论断已被推翻**：UE 5.6 引擎源码位于 `Engine\Source`，逐条依据见 §0.3 的 **E1–E17**。原列 5 组"未验证的引擎侧假设"现**已全部落地**：①`FScriptXxx::Empty` 是否析构元素 → **E1** `UnrealType.h:4046-4058`、**E10** `Containers/Set.h:1843-1865` 与 `Containers/Map.h:1955-1958`；②`AddZeroed`/`InsertZeroed` 是否只零填充 → **E4** `Containers/ScriptArray.h:93-98`、`:50-54`；③`RemoveAt` 之后是否需要 `Rehash` → **E11** `Set.h:1867-1885`（`RemoveAt` 自己摘哈希链，**证伪** F-HLP-029）；④`SwapMemory`/`GetRawPtr` 的 `check`/`checkSlow` 级别 → **E14** `UnrealType.h:3911-3920`、`ScriptArray.h:191-222`、`:55-76`；⑤`FScriptXxx` 是否有析构函数 → **E7** `ScriptArray.h:322-357`、`Array.h:965-972` | F-HLP-004/005/007/019/020/022/025/029（**已落地**，逐项行号见上述 E 编号） | 原因为"无法定位 `Containers/ScriptArray.h`/`ScriptMap.h`/`ScriptSet.h`"——**已不成立** | 已完成，无需再核 |
| U2（**已作废，见 §0.3 行 52**） | ~~`TArray<T>`/`TMap<K,V>`/`TSet<T>` 与 `FScriptArray`/`FScriptMap`/`FScriptSet` 的布局等价性~~ **已落地**：布局等价由引擎 `static_assert` 保证 → **E9** `Containers/Set.h:2072-2090`、`Containers/Map.h:2079-2093` | F-HLP-001/019/026（置信度由"实现巧合"升级为**引擎保证**） | 原因为"同上"（本机无引擎源码）——**已不成立** | 已完成，无需再核 |
| U3 | `FPropertyDescriptor::Factory` 的白名单**实际覆盖哪些属性类型**（`NEW_PROPERTY_DESCRIPTOR` 宏的展开、各 `UE_F_*` 能力宏取值） | F-HLP-002/017/024 的"是否可达" | 未读 `Source/UnrealCSharp/Private/Macro/PropertyMacro.h` 与 `UEVersion.h` 的全部宏定义 | 读该宏文件 + 在 `Factory` 末尾加 `ensureAlways` 统计运行期落空次数 |
| U4 | "非 POD 元素能否真的通过 C# 侧构造出 `TArray<FText>`.`AddZeroed()`"的完整链路（C# 泛型实例化 → 属性类型 → `FTextPropertyDescriptor`） | F-HLP-005 的实际触发概率 | 未读 `FTextPropertyDescriptor.cpp`，也未读 C# 侧 `TArray<T>` 的实例化路径 | 读 `FTextPropertyDescriptor.cpp`；写 C# 用例实测 |
| U5 | `FPropertyDescriptor::DestroyProperty()` 的**子类覆写**是否存在、是否释放反射创建的 `FProperty` | F-HLP-003 的泄漏规模 | 未逐个读 30 个 `F*PropertyDescriptor.cpp` 的 `DestroyProperty` 覆写 | `grep -rn "DestroyProperty" Source/UnrealCSharp/Private/Reflection/Property/` |
| U6 | `FScriptMap::GetScriptLayout` 的**参数顺序/签名**在各 UE 版本上是否一致 | F-HLP-016/024（layout 计算） | 无法对照引擎源码 | 引擎源码比对 + 跨版本编译 |
| U7 | `FContainerRegistry`/`FCSharpEnvironment::RemoveContainerReference` **是否真的 `delete` 掉了 Helper 对象**，以及 `MapManagedHandle2Helper` 的失效时机 | F-HLP-003（泄漏点是否真的"永远不释放"） | 未读 `Registry/FContainerRegistry.h/.inl` 与 `Environment/FCSharpEnvironment.*` 的完整实现（超出本报告的 6 文件范围） | 读 `FContainerRegistry.inl` 的 `RemoveContainerReference` 实现；或在 `~FArrayHelper` 加日志观察 |
| U8 | `TPropertyValue.inl:866-874` 的 `SrcContainer->GetScriptArray()->GetData()` **句柄为空时不判空**是否可达 | 与 F-HLP-006 同形的另一条路径 | 该路径的句柄来源未追完 | `grep -rn "TPropertyValue<.*>::Get(const IManagedHandle" Source/` 追调用点 |
| U9 | `__STDCPP_DEFAULT_NEW_ALIGNMENT__` 与实际元素对齐需求是否冲突（§8.3.3） | 潜在对齐问题（未编号） | 依赖引擎侧分配实现（同 U1） | 引擎源码 + `alignas(32)` 的 `USTRUCT` 实测 |
| U10 | 三个 Helper 的线程安全性（没有任何 `check(IsInGameThread())`） | 未编号（横向维度 3） | 本报告范围内**未发现**显式线程断言；`FRegister*.cpp:34-41`/`:23-30`/`:21-27` 的 `UnRegister` 通过 `AsyncTask(ENamedThreads::GameThread, ...)` 切回主线程，说明作者知道跨线程问题，但 Helper 的方法本身没有断言 | `grep -rn "IsInGameThread" Source/UnrealCSharp/Private/Reflection/Container/ Source/UnrealCSharp/Private/Domain/Interop/FRegister*.cpp` |

### 9.2 检查过但**未发现问题**的维度（按规范 §3 的 8 个横向维度）

| 维度 | 结论 |
|---|---|
| 1. 空指针/越界 | **发现问题**（F-HLP-006/007/016/017/023/024/027） |
| 2. 内存/资源泄漏 | **发现问题**（F-HLP-001/003/004/014/018/019/020/025/026） |
| 3. 线程安全 | **未发现本报告范围内的显式问题**：三个 Helper 都没有 `check(IsInGameThread())`，因此无法断言"只能在 GameThread 用"这一前提；`UnRegister` 走 `AsyncTask(GameThread)`（`FRegisterArray.cpp:36`、`FRegisterMap.cpp:25`、`FRegisterSet.cpp:23`），说明销毁被主动切回主线程。**但**：`FArrayHelper`/`FMapHelper`/`FSetHelper` 的方法本身没有任何线程约束，若 C# 侧从非 GameThread 调用（`Script/` 的 `TArray.cs` 全是直接 P/Invoke，没有线程封送）就会与引擎容器竞争。**标为存疑而非发现**，因为我没有读到 C# 侧是否有统一的线程封送层（超出范围） |
| 4. 异常与错误处理 | **发现问题的一部分**：`FPropertyDescriptor::Factory` 静默 `return nullptr`（F-HLP-002/017/024）；三个 Helper 全文没有 `try/catch`，没有 `try/catch` 穿越托管边界的问题；`check` 的使用见 F-HLP-007 |
| 5. 性能 | **发现问题**（F-HLP-008/010/021/022/028/029） |
| 6. 死代码 | 见 §6；结论是三个 Helper 的公开方法几乎全部是"公开 API 但库内部零调用"，真正的死代码只有 **3 个空 `Initialize()` + 1 个未使用局部变量 = 4 处**（见 §6.2）。**修正**：三个 `GetAddress()` **不是死代码**——真实调用点是**泛型指针**调用 `FContainerRegistry.inl:62`（`(*FoundValue)->GetAddress()`），因是泛型调用，`grep "ArrayHelper->GetAddress"` 必然为 0 |
| 7. 可优化/可读性/一致性 | **发现问题**（F-HLP-011/012/013/015/030，以及 §8.3 的命名与 magic alignment） |
| 8. 平台兼容 | **未发现**：三个 Helper 无 `PLATFORM_*` 条件编译、无字节序假设、无 `sprintf`、无 `int` 宽度假设；唯一的平台相关点是 `__STDCPP_DEFAULT_NEW_ALIGNMENT__`（MSVC 宏，见 §8.3.3，在 Clang/GCC 下是否存在需确认——该宏是 MSVC 特有，`FArrayHelper.cpp`/`FMapHelper.cpp` 直接使用它而**未包含**定义它的头文件，若在 Clang/Linux 上编译可能失败。**置信度: 中**，因为可能有统一的兼容头（`CrossVersion` 模块）提供它，我未追查） |

### 9.3 明确超出本报告范围的部分

- `FContainerRegistry` / `FCSharpEnvironment` 的句柄生命周期与 `delete Helper` 的实际位置（U7）——属"Registry"报告。
- 30 个 `F*PropertyDescriptor` 的实现（本报告只读了 `FPropertyDescriptor` 基类 + `FStrPropertyDescriptor` + 3 个 Container 描述符）——属"属性描述符"报告。
- C# 侧 `Script/UE/CoreUObject/TArray.cs`/`TMap.cs`/`TSet.cs` 的完整实现（本报告只读了 `TArray.cs` 的索引器与相关方法）——属"C# 运行时"报告。
- `CoreCLR`/`Mono`/`LeanCLR` 三个后端对 `IManagedHandle` 的实现差异。

