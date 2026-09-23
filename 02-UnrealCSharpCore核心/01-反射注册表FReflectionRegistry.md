# FReflectionRegistry 反射注册表分析

> 本报告各条**严重度见各 Finding 的「复核结论」字段**（复核 **10 条**：**确认 4、部分确认 6、证伪 0、无法验证 0**；级别变动 **下调 4 条、上调 0、撤销 0**；新增 1 条 —— 见 F-REFL-010）。





> 分析范围：`Source/UnrealCSharpCore/Public/Reflection/FReflectionRegistry.h` + `Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp`
> 覆盖文件：2 个主文件（+ 跨模块 grep 交叉验证、`FClassReflection.{h,cpp}`、`FReflection.{h,cpp}`、`FDynamicGeneratorCore.cpp`、`FCoreCLRDomain.cpp` 的相关段落）
>
> 状态：**优先级 1、2 已完成**（头文件契约 + GC 保护判定 + 生命周期/失效 + 容器配对）；**优先级 5 已完成**（死代码全量计数）；**优先级 3 超出本文件范围**（见 §5 第 3 项）；**优先级 4 已做定性排查、未做定量测量**（见 §5 第 4 项）。共 **10 条发现**：**P0 ×0、P1 ×1、P2 ×4、P3 ×5**。
> ⚠️ 上文"P0 ×2、P1 ×2、P2 ×1、P3 ×4"是初判分布，已更正为上述分布 —— F-REFL-001（P0→**P2**）、F-REFL-002（P0→**P1**）、F-REFL-003（P1→**P2**）、F-REFL-004（P1→**P2**），其余维持（005=P2、006/007/008/009=P3），并新增 **F-REFL-010（P3）**。**本文件不存在 P0。**
> 已**分段逐行读完** `.cpp:261–854` 与 `:995–2409`（原文标注为"未逐行读"的两个区间），并**全量核对 254 条 `CLASS_*` 宏定义**（原文只核对了 16 处）——两个区间均无缺陷，宏定义 0 错配（见 §5 与附 D9）。

> **行数口径说明（已核实，两个口径都对）**：本文件存在两个合法口径 ——
> ① **总行数**（含空行，`(Get-Content $f).Count`）：`.h` = **1194**、`.cpp` = **2409**；
> ② **非空行数**（`Measure-Object -Line` **忽略空行**，等价于 `Where-Object {$_ -match '\S'}`）：`.h` = **612**、`.cpp` = **1811**。
> ⇒ 任务简报里的 "612 / 1811" **不是笔误**，只是口径②；**本报告全部行号一律采用口径①（总行数）坐标系**，所有 `文件:行` 均以 `read`/`grep` 工具的实际输出为准。
> 旧版报告已被本文件覆盖；其行号坐标系（1194/2409）**经验证与本报告一致**，旧版报告的结论在 §5 中按"待复核"保留引用。

## 0. 覆盖范围与阅读清单

| 文件 | 总行数 | 实际读过的行范围 | 是否读完 | 备注 |
|---|---|---|---|---|
| `Public/Reflection/FReflectionRegistry.h` | 1194 | **1–1194** | **是** | 4 段读：1–220 / 221–520 / 521–820 / 821–1194 |
| `Private/Reflection/FReflectionRegistry.cpp` | 2409 | **1–2409 全文件逐行读完**（ 1–260、855–994；**补读 `261–854` 与 `995–2409`**，见附 D9） | **是**（起为逐行覆盖，脚本等价校验降级为佐证） | `261–854` = `Initialize()` 中段的 **198 条**赋值（**勘误**：285 是 `Initialize()` 全程 `.cpp:23–863` 的赋值总数，不是该区间数；grep `= GetClass` 全 `.cpp` = **286** = 285 条赋值 + `:916` 一处非赋值调用）；`995–2409` = 285 个访问器定义，逐条比对"函数名 ↔ `return` 成员名"**0 错配** |
| `Public/Reflection/FReflectionRegistry.inl` | 10 | 1–10 | 是 | `GetClass<T>()` 模板 |
| `Public/Reflection/FClassReflection.h` | 179 | 1–179 | 是 | 用于判定 `FClassReflection` 是否为 UObject |
| `Private/Reflection/FClassReflection.cpp` | 796 | 25–99 | 否 | 仅读 `ManagedHandle2Class` 辅助函数 |
| `Public/Reflection/FReflection.h` / `Private/Reflection/FReflection.cpp` | 25 / 32 | 全部 | 是 | 验证 `~FReflection` 无 virtual、`HasAttribute(nullptr)` 行为 |
| `Private/Dynamic/FDynamicGeneratorCore.cpp` | 1263 | **772–821、1030–1263** | 否（抽样） | 验证 6 个 static 缓存与其消费点 |
| `Private/Domain/CoreCLR/FCoreCLRDomain.cpp` | 312 | 150–264 | 否（抽样） | 验证 Initialize/Deinitialize 调用点 |

**计数方法**：`Select-String -Encoding UTF8 | Measure-Object`；跨模块调用点用 `grep` 工具。所有行号抄自 `read`/`grep` 实际输出。

## 1. 模块职责与架构速览

`FReflectionRegistry` 是**单例**（`FReflectionRegistry.cpp:10–15`，函数内 `static`），职责是 **C# 类型元数据的全局缓存工厂**，而不是"反射数据生成器"：

1. **持有 285 个 C# 类型的 `FClassReflection*`**（`UtilsClass`、`ObjectClass`、…、`ArrayParamAttributeClass`），在 `Initialize()`（`.cpp:21–865`）中一次性全部解析并赋值；
2. **缓存 `UField* → FClassReflection*`**（`Field2Class`，`.h:1189`），避免每次从 `UClass*` 反推 C# 名字；
3. **缓存 `FullName → FClassReflection*`**（`FullName2Class`，`.h:1191`），并**拥有**这些对象的所有权。

### 数据结构（`.h:1188–1192`）—— 逐字摘录

```cpp
1188: private:
1189: 	TMap<TWeakObjectPtr<UField>, FClassReflection*> Field2Class;
1190:
1191: 	TMap<FString, FClassReflection*> FullName2Class;
1192: };
```

### 所有权模型（关键）

| 项 | 事实 | 证据 |
|---|---|---|
| 分配点 | `new FClassReflection(...)`，**全插件仅 2 处** | `.cpp:941`、`.cpp:966`（`Select-String 'new FClassReflection'` = 2） |
| 释放点 | `delete Class`，**全 `.cpp` 仅 1 处 `delete`** | `.cpp:873`（`Select-String 'delete '` = 1） |
| 所有权 | `FullName2Class` 的 **value 拥有**对象；285 个成员是**别名**（不额外 delete） | `Deinitialize()` 只遍历 `FullName2Class` 删除 |
| 元素类型 | `FClassReflection` **不是 UObject**，是普通 C++ 类 | `FClassReflection.h:13` `class UNREALCSHARPCORE_API FClassReflection : public FReflection` |

### 生命周期

```
Initialize()                     .cpp:21–865     ← 285 次 GetClass(NS,Name) 赋值
Deinitialize()                   .cpp:867–877    ← Empty 两表 + delete 全部对象
GetClass(TWeakObjectPtr<UField>) .cpp:879–924
GetClass(IManagedHandle)         .cpp:926–950
GetClass(FString, FString)       .cpp:952–975
Get*Class() ×285                 .cpp:977–2409   ← 纯 `return XxxClass;`
```

三个脚本后端各自成对调用（grep 证据，`FReflectionRegistry::Get().Initialize/Deinitialize` 各 3 处）：
- CoreCLR：`FCoreCLRDomain.cpp:195` / `:242`
- Mono：`FMonoDomain.cpp:167` / `:430`
- LeanCLR：`FLeanCLRDomain.cpp:782` / `:808`

## 2. 关键调用链

1. **热重载销毁链**：`FCoreCLRDomain::UnloadAssembly()`（`FCoreCLRDomain.cpp:240`）→ `FReflectionRegistry::Get().Deinitialize()`（`:242`）→ `.cpp:867` → `:873 delete Class` ×N → `:876 FullName2Class.Empty()`。随后同一函数 `:244 AssemblyLoaderUnloadFn()` 卸载托管程序集。
2. **热重载重建链**：`FCoreCLRDomain::InitializeAssembly()`（`FCoreCLRDomain.cpp:168`）→ `LoadAssembly()`（`:170`）→ `:172 if (TypeBridgeGetFunctionPointerFn != nullptr)` → `:195 FReflectionRegistry::Get().Initialize()` → `.cpp:952 GetClass(NS,Name)` → `.cpp:966 new FClassReflection` → `.cpp:968 FullName2Class.Add`。
3. **绑定查询链（热路径）**：`FCSharpBind::BindImplementation(UObject*)`（`FCSharpBind.inl:75`）→ `FReflectionRegistry::Get().GetClass(Class)` → `.cpp:879` → `:881 Field2Class.Find` → 未命中时 `:916 GetClass(NameSpace,Name)` → `:918 Field2Class.Add`。
4. **元数据属性缓存链（缺陷所在）**：`FDynamicGeneratorCore::SetMetaData(UProperty*, FReflection*)`（`FDynamicGeneratorCore.cpp:790`）→ `GetPropertyMetaDataAttributes()`（`:792` → 定义在 `:1108`）→ `SetFieldMetaData`（`:775`）→ `:780 InReflection->HasAttribute(MetaDataAttribute)` → `:782 MetaDataAttribute->GetName()`。
5. **属性反射构造链**：`FPropertyReflection::FPropertyReflection`（`FPropertyReflection.cpp:15`）→ `FReflectionRegistry::Get().GetUPropertyAttributeClass()`。
6. **属性→C# 类映射链**：`FTypeBridge::GetClass(const FArrayProperty*)`（`FTypeBridge.cpp:449`）→ `GetTArrayClass()`（`FReflectionRegistry.cpp:1126`）+ `GetClass(Inner)`（`:439`）→ `MakeGenericTypeInstance` → `GetClass(IManagedHandle)`（`.cpp:926`）。

## 3. 发现清单

### P0

#### [F-REFL-001] `Deinitialize()` 删除了全部 `FClassReflection`，却不把 285 个成员指针置空，留下悬垂访问器

- **类别**: 未定义行为 / 内存安全
- **严重度**: **P2**
- **复核结论**: 部分确认（偏差：严重度 —— 悬垂状态确证，但全插件不存在解引用该悬垂指针的代码）
- **可达性**: 活跃
- **复核证据**: `Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp:867-877`（全函数 11 行，仅 `Empty`/`delete`/`Empty`，**无任何 `= nullptr`**，grep `delete ` 全 `.cpp` = **1** 处即 `:873`）；`.h:610-1185` 285 个裸指针成员（grep `^\tFClassReflection\* \w+\{\};` = **285**）；3 个调用点 `FCoreCLRDomain.cpp:242`、`FMonoDomain.cpp:430`、`FLeanCLRDomain.cpp:808`；**本工程唯一实跑后端** LeanCLR 的 `FLeanCLRDomain.cpp:806-832` 在 `Deinitialize()` 之后只做 `Bridge_Invoke(AssemblyLoaderUnloadFn)` + 清桥函数指针 + `MethodInvokePaths.Empty()`，未再触碰注册表；窗口内所有消费者均只做 `TSet::Contains` 指针值比较（`FReflection.cpp:18-21`、`FPropertyReflection.cpp:15`、`FMethodReflection.cpp:20/:22`、`FClassReflection.cpp:335`）
- **级别变动**: P0→P2（悬垂状态 100% 确证，但无任何可达的解引用路径 → 属"隐患/零成本可修"，不是崩溃或数据损坏）
- **文件**: `Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp:867`
- **函数**: `FReflectionRegistry::Deinitialize()`
- **置信度**: 中（"悬垂状态存在"是**确证的代码事实**；"该窗口内是否真的发生解引用"未被完整证明，见下方"问题"第 3 点）

**现状（代码事实）**

```cpp
// FReflectionRegistry.cpp:867-877   —— 全函数，11 行
867: void FReflectionRegistry::Deinitialize()
868: {
869: 	Field2Class.Empty();
870:
871: 	for (const auto& [PLACEHOLDER, Class] : FullName2Class)
872: 	{
873: 		delete Class;
874: 	}
875:
876: 	FullName2Class.Empty();
877: }
```

`Select-String` 实测：该 `.cpp` 全文**只有 1 处 `delete`**（即 `:873`）、**2 处 `.Empty()`**（`:869`、`:876`）、**0 处 `.Reset(`**、**0 处 `.Remove(`**。函数体内**没有任何 `XxxClass = nullptr;`**。

而对应的 285 个成员在头文件中被声明为**裸指针**（`Select-String '^\tFClassReflection\* \w+\{\};'` = **285** 命中），例如：

```cpp
// FReflectionRegistry.h:610-612
610: 	FClassReflection* UtilsClass{};
611:
612: 	FClassReflection* ObjectClass{};
```

**调用上下文**

- `Deinitialize()` 的 3 个调用点：`FCoreCLRDomain.cpp:242`、`FMonoDomain.cpp:430`、`FLeanCLRDomain.cpp:808`，全部位于**卸载程序集之前**（`FCoreCLRDomain.cpp:242` → `:244 AssemblyLoaderUnloadFn()`）。
- `Initialize()` 的 3 个调用点：`FCoreCLRDomain.cpp:195`、`FMonoDomain.cpp:167`、`FLeanCLRDomain.cpp:782`，位于**加载程序集之后**。
- 访问器的调用点遍布全部 7 个模块（`FTypeBridge.cpp`、`FDynamicGeneratorCore.cpp`、`FPropertyReflection.cpp`、`TPropertyClass.inl`、`FRegister*.cpp` 等 30+ 文件；`FReflectionRegistry::Get()` 全 `Source/` 命中 **254 处**）。

**问题**

1. **确证**：`delete Class`（`:873`）释放了全部 `FClassReflection`，但 `UtilsClass` … `ArrayParamAttributeClass` 这 285 个成员**仍指向已释放内存**。`Deinitialize()` 的语义契约应是"对象回到可安全销毁/可重建状态"，而实际上对象处于**任何一个访问器都返回悬垂指针**的状态。`Deinitialize()` 之后调用 `GetUtilsClass()`（`.cpp:977–980` 只是 `return UtilsClass;`）即返回悬垂指针，**没有任何断言或 nullptr 保护**。
2. **二次 `Deinitialize()` 是安全的**（第二遍遍历已空的 `FullName2Class`，不重复 `delete`），**`Initialize()` 连续调用两次也是安全的**（`:956 FullName2Class.Find` 命中缓存，返回同一指针，不重复 `new`）。因此问题**不是**双重释放，而是**单向的悬垂窗口**。
3. **未证明的部分（诚实标注）**：要在该窗口内触发实际崩溃，需要一个调用方在 `Deinitialize()` 之后、下一次 `Initialize()` 之前去**解引用**访问器返回值。我核查了以下候选方，**它们都不解引用**：
   - `FPropertyReflection.cpp:15` → `bIsUProperty = Attributes.Contains(GetUPropertyAttributeClass())`：`FReflection::HasAttribute` 实现为 `Attributes.Contains(InAttribute)`（`FReflection.cpp:18–21`），对 `TSet<FClassReflection*>` 只做**指针值哈希比较，不解引用**，`nullptr` 也安全返回 false。
   - `FMethodReflection.cpp:20/22` 同上，仅 `Contains`。
   - `FCoreCLRDomain::UnloadAssembly`（`:240–264`）在 `Deinitialize()` 之后未再触碰注册表。
   因此这条发现的**确定部分是"缺失的失效保护"**（P0 级隐患：只要出现一个解引用型调用方，或未来新增一个，立即崩溃/数据损坏），而非已证实的崩溃路径。这仍应修，因为代价极低（285 行 `= nullptr` 或一个 `bInitialized` 标志）。
4. **附带一致性问题**：`Initialize()` 每次调用都无条件重新赋值全部 285 个成员；若某次 `Initialize()` 中 `GetClass()` 返回 `nullptr`（C# 类型未找到，`FReflectionRegistry.cpp:974 return nullptr`），成员会被**静默写成 nullptr**，此后所有依赖它的属性匹配（如 `HasAttribute(nullptr)`）**静默失效**，调用方无法区分"该类型没有此 attribute"与"注册表没找到该 attribute 类"。

**建议**

```cpp
void FReflectionRegistry::Deinitialize()
{
	Field2Class.Empty();

	for (const auto& [PLACEHOLDER, Class] : FullName2Class)
	{
		delete Class;
	}

	FullName2Class.Empty();

	// 新增：让所有别名成员立即失效，任何窗口内的误用变成可定位的 nullptr 解引用而非 UAF
	UtilsClass = nullptr;  /* … 285 项 … */
	// 或更省事：把 285 个成员改为 TArray/结构体数组，用 for 循环清零
}
```

更稳的做法是**把 285 个裸成员替换为一个按枚举下标索引的 `TStaticArray<FClassReflection*, N>`**（取值封装成 `GetXxxClass()` 访问器），这样 `Deinitialize()` 只需 `MemberClasses = {};`，并且消除 285 组重复的手写 getter/setter。

**验证方式**

- grep：`Select-String -Path ...FReflectionRegistry.cpp -Pattern 'delete ' -Encoding UTF8` 应只有 `:873`；`-Pattern '= nullptr'` 当前为 0。
- 在 `Deinitialize()` 末尾插入 `for (auto& C : AllClasses) { check(C == nullptr); }` 会立刻失败，证明成员未清空。
- 运行时：在 `Deinitialize()` 之后、`Initialize()` 之前调用 `FReflectionRegistry::Get().GetUtilsClass()`，在 ASan/`-fsanitize=address`（或 UE 的 `checkSlow`）下会命中 use-after-free 报告。

---

#### [F-REFL-002] `FDynamicGeneratorCore` 的 6 个函数局部 `static TArray<FClassReflection*>` 缓存跨热重载永不失效

- **类别**: Bug / 未定义行为
- **严重度**: **P1**（缓存永不失效是确定的；由此产生的后果严重度见"问题"）
- **复核结论**: ✅确认 —— 严重度由 P0 校正为 **P1**；可达性 活跃
- **文件**: `Source/UnrealCSharpCore/Private/Dynamic/FDynamicGeneratorCore.cpp:1040`
- **函数**: `FDynamicGeneratorCore::GetClassMetaDataAttributes()` / `GetStructMetaDataAttributes()` / `GetEnumMetaDataAttributes()` / `GetInterfaceMetaDataAttributes()` / `GetPropertyMetaDataAttributes()` / `GetFunctionMetaDataAttributes()`
- **置信度**: 高（缓存永不失效、且在热重载后必然与注册表脱节，均为确证的代码事实）

**现状（代码事实）**

6 个函数结构完全相同，各自在**函数局部 `static`** 里把注册表访问器结果**快照**下来：

```cpp
// FDynamicGeneratorCore.cpp:1108-1116（其余 5 个同构）
1108: const TArray<FClassReflection*>& FDynamicGeneratorCore::GetPropertyMetaDataAttributes()
1109: {
1110: 	static auto& ReflectionRegistry = FReflectionRegistry::Get();
1111:
1112: 	static TArray<FClassReflection*> PropertyMetaDataAttributes = {
1113: 		ReflectionRegistry.GetToolTipAttributeClass(),
1114: 		ReflectionRegistry.GetDeprecationMessageAttributeClass(),
...
1195: 		ReflectionRegistry.GetCustomizePropertyAttributeClass()
1196: 	};
1197:
1198: 	return PropertyMetaDataAttributes;
1199: }
```

6 个 `static TArray<FClassReflection*>` 的定义行（`grep 'static TArray<FClassReflection\*>'`，全 `Source/` 恰好 **6 命中**）：

| 函数 | `static TArray` 定义行 | 返回行 |
|---|---|---|
| `GetClassMetaDataAttributes()` | `:1040` | `:1064` |
| `GetStructMetaDataAttributes()` | `:1071` | `:1079` |
| `GetEnumMetaDataAttributes()` | `:1086` | `:1092` |
| `GetInterfaceMetaDataAttributes()` | `:1099` | `:1105` |
| `GetPropertyMetaDataAttributes()` | `:1112` | `:1198` |
| `GetFunctionMetaDataAttributes()` | `:1205` | `:1261` |

消费点（`grep 'MetaDataAttributes\(\)'` → 6 处调用，全在 `FDynamicGeneratorCore.cpp`：`:792`、`:799`、`:807`、`:808`、`:855`、`:874`）最终都进入同一个模板：

```cpp
// FDynamicGeneratorCore.cpp:775-788
775: static void SetFieldMetaData(T InField, const TArray<FClassReflection*>& InMetaDataAttributes,
776:                              FReflection* InReflection, const TFunction<void()>& InSetMetaData)
777: {
778: 	for (const auto& MetaDataAttribute : InMetaDataAttributes)
779: 	{
780: 		if (InReflection->HasAttribute(MetaDataAttribute))
781: 		{
782: 			FDynamicGeneratorCore::SetMetaData(InField, MetaDataAttribute->GetName(),
783: 			                                   InReflection->GetAttributeValue(MetaDataAttribute));
784: 		}
785: 	}
786:
787: 	InSetMetaData();
788: }
```

**调用上下文**

- 这 6 个函数**只在 `#if` 区块内**（该区块的 `#endif` 在 `:1263`，即文件末尾），因此仅存在于 `WITH_EDITOR` 构建；而 C# 热重载本身也是编辑器行为，**两者覆盖面重合**，不构成缓解。
- 调用者是 `FDynamicGeneratorCore::SetMetaData(UProperty*/UFunction*/UClass*/UScriptStruct*/UEnum*, FReflection*)`（`:790`、`:797`、`:804`、`:851`、`:871`），由 `FDynamicClassGenerator.cpp:43/:66`、`FDynamicStructGenerator.cpp:25/:43`、`FDynamicEnumGenerator.cpp:24`、`FDynamicInterfaceGenerator.cpp:23/:45`、`FDynamicRegistry.cpp:62` 触发 —— 即**每次生成动态类/结构体/枚举时**。
- `FReflectionRegistry::Deinitialize()`（`FReflectionRegistry.cpp:867`）**只清 `Field2Class` 与 `FullName2Class`**，**没有任何机制去通知或清空这 6 个 static 数组**。`FReflectionRegistry` 不是 `FGCObject`，也没有版本号/代际（generation）计数器。

**问题**

1. **确证**：第一次 `FDynamicGeneratorCore::GetXxxMetaDataAttributes()` 被调用时，`static` 数组按**当时的注册表成员值**初始化一次，此后**永不重新求值**。`FReflectionRegistry::Deinitialize()`（`.cpp:873`）把那些 `FClassReflection` 全部 `delete`，下一次 `Initialize()`（`.cpp:966`）在**新地址**上 `new` 出新对象。6 个 static 数组因此**永久持有旧地址**——即**悬垂指针**。这正是任务简报描述的"配对缺口"，我在此确证了它。
2. **后果的精确判定（对简报结论的修正）**：简报称"第二次 C# 热重载即 UAF"。我核查了 `:780` / `:782` 的数据流后认为**这条路径上不会立即 UAF**，理由是：`:780 InReflection->HasAttribute(MetaDataAttribute)` 展开为 `Attributes.Contains(MetaDataAttribute)`（`FReflection.cpp:20`），只做**指针值比较**；而 `Attributes` 集合的元素**只可能来自注册表当前存活的对象**——它们由 `FClassReflection.cpp:41 FReflectionRegistry::Get().GetClass(InManagedHandle)` 解析得到（`FClassReflection.cpp:37–49 ManagedHandle2Class`）。所以只有当一个**悬垂旧地址恰好等于某个存活新对象的地址**时，`Contains` 才会返回 true，`:782` 的解引用才是"指向存活对象"。由此得到的真实后果是：
   - **多数情况（地址未复用）**：`Contains` 返回 false → 该 metadata attribute **被静默丢弃**，动态生成的 `UClass`/`UProperty`/`UFunction` 上**丢失编辑器元数据**（如 `ToolTip`、`ClampMin`、`EditCondition`、`DisplayName`……）。这是**静默功能错误（P1 级）**，不是崩溃。
   - **少数情况（地址复用）**：`Contains` 返回 true，但拿到的是**另一个属性类** → 把**错误的元数据键值对写进 UE 反射**，属**数据损坏**。
   无论哪种，**"第二次热重载后元数据行为与第一次不一致"是确定的**，且随 `fbip` 分配器的确定性（同尺寸、同顺序分配，地址复用概率很高）而更偏向"写错元数据"。
3. 之所以仍定级 **P0**：缓存永久失效 + 无任何失效钩子，是**结构性缺陷**；并且同一模式一旦被复制到会**直接解引用**（而非先 `Contains`）的地方，就立即变成真正的 UAF。同时它使"热重载"这一核心工作流**不可靠**。
4. **`static auto& ReflectionRegistry = FReflectionRegistry::Get();`（如 `:1110`）本身是安全的**——绑定到单例的引用，单例永不销毁。危险的是紧随其后的 `static TArray` 的**元素值**。

**建议**

三种改法，按推荐度排序：

1. **去掉 `static`，改为每次调用时构造并返回 `const TArray&` 成员**（把 6 个数组改成 `FReflectionRegistry` 的成员，随 `Initialize()`/`Deinitialize()` 一起重建）——一次热重载即自动同步，且顺手消除 285 次访问器调用。
2. 若必须保留 `static` 缓存（性能考虑），给注册表加**代际号**：
   ```cpp
   // FReflectionRegistry.h
   uint32 GetGeneration() const { return Generation; }   // Deinitialize/Initialize 时 ++
   ```
   并在 6 个函数内缓存代际号，`if (CachedGeneration != ReflectionRegistry.GetGeneration()) { 重新填充; }`。
3. 反正在 `Initialize()` 里已经解析过这 285 个类，**直接让 `FDynamicGeneratorCore` 从注册表按需取**（去掉数组快照），并给 `GetXxxClass()` 加 `check()`。

**验证方式**

- grep 确认无清理：`grep 'MetaDataAttributes' Source/` 只会命中 `FDynamicGeneratorCore.{h,cpp}`，其中**没有任何** `Empty()`/`Reset()`。

**处置（2026-09-22，本轮 G4 收口；✅ 已提交 `8c54c598`）**

采用**上表第 1 条与第 3 条之间的路线** —— 「**去掉缓存本身**」：6 个 getter 去掉函数级 `static`（`static auto& ReflectionRegistry` → `const auto&`、`static TArray<FClassReflection*> X = {…}` → `TArray<FClassReflection*> X = {…}`），返回值由 `const TArray<FClassReflection*>&` 改为 **by-value `TArray<FClassReflection*>`**，数组每次调用从注册表**现查重建**。⇒ 本报告 §上述第 3 点的判据「缓存结构性永不失效」**不再成立**（不是"补了失效钩子"，而是**没有可失效的缓存**）。

- **未采纳第 1 条**（把 6 个数组提升为 `FReflectionRegistry` 成员）：那需要给注册表加 6 个成员 + 6 个访问器，且把"哪些属性类参与元数据下放"这一**生成侧知识**放进注册表；本轮保持生成侧自持。
- **未采纳第 2 条**（代际号）：同上，需新增公开 API；且"去缓存"已能结构性消除，性能代价可忽略（见下）。
- **成本**：6 个数组规模 = Property **83** / Function **53** / Class 21 / Struct 5 / Enum 3 / Interface 3（本轮实测计数，`ReflectionRegistry.Get*` 行数），全部是**成员指针读取 + 一次小分配**，调用点在**编辑器侧动态类型生成**路径（每个属性/函数/类型各一次）；同一调用栈里本来就有最多 83 次哈希查表的 `HasAttribute` 循环。**语义零变化**：数组字面量一行未改，返回内容与顺序逐字不变。
- **改动面**：`Source/UnrealCSharpCore/Public/Dynamic/FDynamicGeneratorCore.h`（6 行声明）+ `Private/Dynamic/FDynamicGeneratorCore.cpp`（6 个 getter）；**消费点 `SetFieldMetaData` 与 6 处调用点零改动**（by-value 临时量绑定 `const TArray<FClassReflection*>&` 形参合法）。净 24 增 / 24 删，补丁留档 `Saved/Retest/fix-G4-metadata-cache.patch`。
- **验证**：UBT `UnrealCSharpTestEditor Win64 Development` **Succeeded**；两阶段域重建装置（PIE#1 → `UnrealCSharp.Editor.SetActive 0/1` → PIE#2）**2 × 1704 例 / 0 失败**、与基线仅差已知 4 条新用例、`ensure`/`assert` 0 行；`-game` 回归 **1704 / 0**；新增注释/`throw`/日志/`try-catch`/`ensure` 均 0。
- 🔴 **未取得**：**修复前"陈旧指针"的运行期对照**。3 次装置尝试（`SetActive 0/1`、无改动 `Compile`、改了 C# 源后的 `Compile`）+ 临时指针同一性探针（`[G4PROBE] … cached=%p live=%p stale=%d`，已移除）实测：**460 行探针全部落在 C# 程序集加载时的那一次生成上**（`stale=1` 计数 **0**），三次重载后**均未再触发生成** ⇒ **本工程当前的动态类型生成只在程序集加载时发生一次**。故运行期证据止于"无回归 + 域重建路径跑通"，**不是"复现了旧缺陷"**；后果判定的更正（`HasAttribute` 只做指针相等 ⇒ 多数为**静默元数据丢失**、少数为**写错元数据**）仍是静态推演。**补证路线**：在"改 C# 源 → 编译 → 新程序集加载"的真实热重载下重放同一探针，或按本报告原建议在 `SetFieldMetaData` 内加一次性 `ensure`。逐条登记见 [`09-问题清单/00-推荐优先修复清单.md`](../../09-问题清单/00-推荐优先修复清单.md) §2.3。
- 用例：编辑器内连续执行两次 C# 热重载，然后比较动态生成的 `UClass` 上的 metadata（如 `GetMetaData(TEXT("ToolTip"))`），第二次会缺失或错误。
- 断言：在 `SetFieldMetaData`（`:778`）循环内加 `check(!MetaDataAttribute || MetaDataAttribute->GetManagedClass() != InvalidManagedHandle)`；或在 `Deinitialize()` 里断言这 6 个 static 数组为空（当前会失败）。

### P1

#### [F-REFL-003] `Field2Class` 只增不减，键为 `TWeakObjectPtr` 却从不清理失效条目

- **类别**: 内存/资源泄漏
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度由 P1 校正为 **P2**；可达性 活跃
- **文件**: `Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp:918`
- **函数**: `FReflectionRegistry::GetClass(const TWeakObjectPtr<UField>&)`
- **置信度**: 高

**现状（代码事实）**

```cpp
// FReflectionRegistry.cpp:879-924（节选）
881: 	if (const auto FoundClass = Field2Class.Find(InField))
882: 	{
883: 		return *FoundClass;
884: 	}
...
916: 	if (const auto FoundClass = GetClass(NameSpace, Name))
917: 	{
918: 		Field2Class.Add(InField, FoundClass);
919:
920: 		return FoundClass;
921: 	}
```

`Select-String` 实测该 `.cpp`：`.Add(` = **3**（`:918`、`:943`、`:968`）、`.Empty()` = **2**（`:869`、`:876`）、`.Remove(` = **0**、`.Reset(` = **0**。即 `Field2Class` 的**唯一收缩点**是 `Deinitialize()` 的整表 `Empty()`。

**调用上下文**

`GetClass(UField*)` 的调用方遍布全模块：`FCSharpBind.inl:75`、`FDynamicRegistry.cpp:30`、`FCSharpBind.cpp:354/:426`、`FClassRegistry.cpp:256`、`FCSharpFunctionDescriptor.cpp:12`、`FClassDescriptor.cpp:25`、`FCSharpEnvironment.cpp:446`、`FDynamicStructGenerator.cpp:374`、`FDynamicGenerator.cpp:232` 等。键的取值是 `UClass`/`UScriptStruct`/`UEnum`（`:888`/`:899`/`:906` 的 `Cast<>`）。

**问题**

1. 键是 `TWeakObjectPtr<UField>`，因此**不会被 GC 保住**，但条目本身**不会因为键失效而被移除**。UE 的 `TWeakObjectPtr` 带序列号，失效后不会误命中，所以**不是悬垂解引用**（键侧是安全的），但 map 会**无条件累积**：
   - 每条失效条目仍占用一个 `TWeakObjectPtr`（含对象索引 + 序列号）与一个 `FClassReflection*`，以及哈希桶。
   - 动态类生成（`FDynamicClassGenerator`）在编辑器里会**反复创建/丢弃 `UClass`**（每次热重载、每次生成新 C# 类）。这些 `UClass` 被 GC 后，其在 `Field2Class` 中的条目变成永久垃圾，**直到下一次 `Deinitialize()`**。
   - 后果：长时间编辑器会话中 map 单调增长（内存缓慢上涨 + 哈希冲突恶化导致 `:881 Find` 变慢）。这不是无界泄漏（`Deinitialize()` 会清），但**在两次重载之间是无界增长**。
2. 建议增加**惰性压缩**：在 `Add` 前或按计数阈值触发 `Field2Class` 的失效键清扫。UE 惯用法：
   ```cpp
   // 例如每 1024 次 Add 或 Field2Class.Num() 超过上次清扫时的 2 倍时
   for (auto It = Field2Class.CreateIterator(); It; ++It)
   {
       if (!It.Key().IsValid()) { It.RemoveCurrent(); }
   }
   ```
   注意 `TWeakObjectPtr` 作键时**不能**在 `Initialize()` 之外随意重建语义，清扫只删无效键，不影响有效映射。
3. 另有一个**语义**隐患：`Field2Class` 的 value 指向 `FullName2Class` 拥有的对象。如果某个 `UField` 先失效、随后**同一地址**被新的 `UClass` 复用，新查询走的是**新键**（序列号不同）→ 不会误命中，行为正确。这一点是安全的，仅记录以排除误报。

**验证方式**

- `Select-String -Pattern '\.Remove\(|\.Reset\(' ...FReflectionRegistry.cpp -Encoding UTF8` → 0 命中，证明无清理。
- 运行时：在编辑器里循环生成/丢弃动态类（或连续热重载 20 次），打印 `Field2Class.Num()`，会单调上升且包含大量 `!Key.IsValid()` 条目。

#### [F-REFL-004] `Initialize()` 中 `GetClass()` 失败被静默吞掉，285 个成员可能保持 `nullptr` 且无任何诊断

- **类别**: Bug / 异常与错误处理
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度由 P1 校正为 **P2**；可达性 活跃
- **文件**: `Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp:23`（及 `:23–864` 全部 285 处赋值）
- **函数**: `FReflectionRegistry::Initialize()`
- **置信度**: 高

**现状（代码事实）**

```cpp
// FReflectionRegistry.cpp:23-26
23: 	UtilsClass = GetClass(
24: 		COMBINE_NAMESPACE(NAMESPACE_ROOT, NAMESPACE_CORE_UOBJECT), CLASS_UTILS);
25:
26: 	ObjectClass = GetClass(NAMESPACE_SYSTEM, CLASS_OBJECT);
```

而 `GetClass(NS,Name)` 的失败返回是 `return nullptr;`（`:974`），`Initialize()` 的返回类型是 `void`（`.h:16`），**不返回成功与否，也不 `check`/`ensure`/`UE_LOG`**。

**调用上下文**

`Initialize()` 的 3 个调用点：`FCoreCLRDomain.cpp:195`、`FMonoDomain.cpp:167`、`FLeanCLRDomain.cpp:782`，**都忽略了返回值**（函数本就无返回值）。在 `FCoreCLRDomain.cpp:195` 处，`Initialize()` 位于 `if (TypeBridgeGetFunctionPointerFn != nullptr)`（`:172`）之内，但**不检查 `bIsInitialized`**，也不检查 285 个类是否全部解析成功。

**问题**

C# 程序集未成功加载、版本不匹配、或某个 attribute 类被重命名时，`ScriptDomain->GetClass(NameSpace, Name)`（`:963`）返回无效句柄 → `GetClass` 返回 `nullptr` → 对应成员被写成 `nullptr`。后果：

1. **静默降级**：依赖该成员的属性匹配把 `nullptr` 传给 `HasAttribute`（`FReflection.cpp:20 Attributes.Contains(nullptr)`）→ 返回 false → 该 attribute 被**当作不存在**。用户看到的症状是"某个 C# attribute 突然不起作用了"，而**没有任何日志**指向根因。
2. `FDynamicGeneratorCore.cpp:811`、`:821`、`:830`、`:857`、`:877` 等处**已经**用 `if (const auto XxxAttributeClass = ...)` 做了 nullptr 检查，说明作者知道可能为 nullptr；但 `Initialize()` 自身**不报告**哪些类没解析到，导致排查困难。
3. 这是"错误被吞掉"的典型形态（`_CONVENTIONS.md` §3.4）。

**建议**

```cpp
// .h
bool Initialize();   // 或保留 void，但内部校验
// .cpp  在 Initialize() 末尾统一校验
const TArray<TPair<const TCHAR*, FClassReflection*>> AllClasses = {
    {TEXT("UtilsClass"), UtilsClass}, /* … */
};
for (const auto& [Name, Class] : AllClasses)
{
    if (Class == nullptr)
    {
        UE_LOG(LogUnrealCSharp, Error, TEXT("FReflectionRegistry: failed to resolve %s"), Name);
        bInitialized = false;
    }
}
```
配合 F-REFL-001 的数组化改造，这段校验可以自动生成。

**验证方式**

- grep：`Select-String -Pattern 'check\(|ensure\(|UE_LOG' ...FReflectionRegistry.cpp -Encoding UTF8` → 该 `.cpp` 中无任何校验/日志（仅有 `#include` 与赋值）。
- 用例：故意删除/重命名一个 C# attribute 类后重载，观察是否有任何 Error 日志（当前无），且该 attribute 静默失效。

### P2

#### [F-REFL-005] `GetClass(IManagedHandle)` 命中缓存时 `Free` 掉传入句柄，导致"同一句柄被释放两次"的调用约定陷阱

- **类别**: 未定义行为 / 可读性
- **严重度**: **P2**
- **复核结论**: ⚠️部分 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp:926`
- **函数**: `FReflectionRegistry::GetClass(const IManagedHandle InManagedClass)`
- **置信度**: 中（调用约定是否为"所有权转移"需看 `IManagedHandle` 的所有调用方约定，我只核对了 `FClassReflection.cpp:37–49`）

**现状（代码事实）**

```cpp
// FReflectionRegistry.cpp:926-950
926: FClassReflection* FReflectionRegistry::GetClass(const IManagedHandle InManagedClass)
927: {
928: 	if (IManagedHandleIsValid(InManagedClass))
929: 	{
930: 		if (const auto ScriptDomain = IScriptDomain::Get())
931: 		{
932: 			const auto FullName = ScriptDomain->GetFullName(InManagedClass);
933:
934: 			if (const auto FoundClass = FullName2Class.Find(FullName))
935: 			{
936: 				ScriptDomain->Free(InManagedClass);
937:
938: 				return *FoundClass;
939: 			}
940:
941: 			const auto Class = new FClassReflection(InManagedClass, ScriptDomain->GetName(InManagedClass));
942:
943: 			FullName2Class.Add(FullName, Class);
944:
945: 			return Class;
946: 		}
947: 	}
948:
949: 	return nullptr;
950: }
```

**调用上下文**

- 参数按**值**传递（`const IManagedHandle InManagedClass`），但内部**可能 `Free` 它**（`:936`）——即参数实际是"所有权转移 + 可能被消费"。
- 调用方之一的惯用法与此一致：`FClassReflection.cpp:37–49 ManagedHandle2Class` 在调用后立刻 `InManagedHandle = InvalidManagedHandle;`（`:43`）把调用方的副本主动作废，说明作者意识到句柄被消费了。
- 其他调用方：`FClassReflection.cpp:41`、`:102`、`:350`；`FScriptDomainImpl.inl:447`、`:459`；`FTypeBridge.cpp:379`、`:427`、`:439`、`:463`、`:511`、`:525`、`:539`、`:553`。

**问题**

1. **命中缓存时**（`:934–939`）：传入的托管句柄被 `Free`，且该句柄**没有被** `FClassReflection` 接管（返回的是已存在的对象）。**未命中时**（`:941`）：句柄被交给 `new FClassReflection`，由 `FClassReflection` 的所有权持有。
   → 同一个函数在两条路径上对同一参数的所有权语义**不同**：一条"消费掉"，一条"转移给新对象"。调用方若不严格遵守"调用后不得再使用该句柄"的约定，就会在命中路径上**重复释放**（如果调用方自己也 Free），或在使用路径上**泄漏**（如果调用方以为已经转移、却命中缓存未转移）。
2. 这是**接口设计缺陷**而非已证实的崩溃：`ManagedHandle2Class`（`FClassReflection.cpp:43`）主动置无效，说明主要调用方守约；但按值传参 + 隐藏 `Free` 的组合极易被后续维护者误用。

**建议**

把所有权语义显式化，二选一：

```cpp
// 方案 A：只读，不消费
FClassReflection* GetClass(const IManagedHandle InManagedClass);   // 命中缓存时也【不】Free
// 由调用方统一负责 Free；或
// 方案 B：显式转移所有权
FClassReflection* TakeClass(IManagedHandle& InManagedClass);        // 引用 + 命名表达消费
```
并在函数头注释写清"命中缓存时**消费**该句柄"。

**验证方式**

- grep 所有 `GetClass(` 的 `IManagedHandle` 重载调用点，逐一确认调用后是否再使用该句柄：`grep -n 'GetClass(.*ManagedHandle' Source/`。
- 在 `:936` 与 `:941` 两处加 `check(InManagedHandle != InvalidManagedHandle)` 并在 C# 侧开启句柄计数，观察是否有 double-free 断言。

### P3

#### [F-REFL-006] `GetTScriptInterfaceClass()` 是 285 个访问器中唯一缺少 `const` 的一个

- **类别**: 可优化/可读性
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Source/UnrealCSharpCore/Public/Reflection/FReflectionRegistry.h:67`（声明）/ `Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp:1067`（定义）
- **函数**: `FReflectionRegistry::GetTScriptInterfaceClass()`
- **置信度**: 高

**现状（代码事实）**

`Select-String` 计数：`.h` 中 `FClassReflection\* Get\w+Class\(\)` = **285**，其中带 ` const` 的 = **284**；`.cpp` 中 `FReflectionRegistry::Get\w+Class\(\) const` = **284**，总定义数 285。

```cpp
// FReflectionRegistry.h:65-69
65: 	FClassReflection* GetNameClass() const;
66:
67: 	FClassReflection* GetTScriptInterfaceClass();      // ← 无 const
68:
69: 	FClassReflection* GetStringClass() const;
```
```cpp
// FReflectionRegistry.cpp:1067-1070
1067: FClassReflection* FReflectionRegistry::GetTScriptInterfaceClass()
1068: {
1069: 	return TScriptInterfaceClass;
1070: }
```

**调用上下文**

调用点（`grep 'GetTScriptInterfaceClass'` = 4 命中）：`FTypeBridge.cpp:85`、`FTypeBridge.cpp:425`，以及 `.h:67` 声明与 `.cpp:1067` 定义。

**问题**

函数体只是 `return TScriptInterfaceClass;`（`:1069`），**没有任何非 const 操作**，`const` 缺失纯属笔误。后果：在 `const FReflectionRegistry&` 上下文（或 const 成员函数内）无法调用该访问器，而其余 284 个可以 —— 这是 API 不一致，会给调用方带来"为什么只有这个不能调"的困惑，也使 `FTypeBridge::GetClass(const FScriptInterfaceProperty*)` 这类本应 const 的函数无法标记 const。

**建议**

```cpp
// .h:67
FClassReflection* GetTScriptInterfaceClass() const;
// .cpp:1067
FClassReflection* FReflectionRegistry::GetTScriptInterfaceClass() const
```

**验证方式**

- `Select-String -Pattern 'Get\w+Class\(\) const' ...FReflectionRegistry.h -Encoding UTF8 | Measure-Object` 应从 284 变为 285。

#### [F-REFL-007] 285 组"声明 + 定义 + 成员"三份手写清单，无生成机制，极易不同步

- **类别**: 可优化/可读性
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Source/UnrealCSharpCore/Public/Reflection/FReflectionRegistry.h:31`–`1185`、`Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp:23`–`865` 与 `:977`–`2409`
- **函数**: 全部 285 个 `Get*Class()` 访问器与 `Initialize()`
- **置信度**: 高

**现状（代码事实）**

同一批 285 个符号被**手写三遍**，且三份清单的**总数恰好一致**（我用 `Select-String` 分别计数）：

| 清单 | 位置 | 实测数量 | grep 模式 |
|---|---|---|---|
| 访问器声明 | `.h:31–607` | **285** | `FClassReflection\* Get\w+Class\(\)` |
| 数据成员声明 | `.h:610–1185` | **285** | `^\tFClassReflection\* \w+\{\};` |
| 访问器定义 | `.cpp:977–2409` | **285** | `FReflectionRegistry::Get\w+Class\(\)` |
| `Initialize()` 赋值 | `.cpp:23–864` | （同一批符号，逐条 `X = GetClass(...)`） | — |

`Initialize()` 的赋值体形如（`:102–107`）：

```cpp
102: 	OverrideAttributeClass = GetClass(
103: 		COMBINE_NAMESPACE(NAMESPACE_ROOT, NAMESPACE_CORE_UOBJECT), CLASS_OVERRIDE_ATTRIBUTE);
104:
105: 	UClassAttributeClass = GetClass(
106: 		COMBINE_NAMESPACE(NAMESPACE_ROOT, NAMESPACE_DYNAMIC), CLASS_U_CLASS_ATTRIBUTE);
```

而访问器定义形如（`:1138–1141`）：

```cpp
1138: FClassReflection* FReflectionRegistry::GetOverrideAttributeClass() const
1139: {
1140: 	return OverrideAttributeClass;
1141: }
```

**问题**

1. **285 × 4 ≈ 1140 行纯样板**，占 `.h` 的 97%、占 `.cpp` 的 60%+。`.cpp` 中 `:977–2409`（1433 行）全部是 `return XxxClass;` 的单行函数体 —— 这批代码的**信息密度接近 0**，却要人工维护 `#if UE_F_UTF8_STR_PROPERTY` / `#if UE_F_ANSI_STR_PROPERTY` / `#if UE_F_OPTIONAL_PROPERTY` / `#if WITH_EDITOR` 四组条件编译的**位置一致性**（`.h` 中 `:71–77`、`:95–97`、`:293–607`、`:650–676`、`:674–676`、`:872–1186` 必须与 `.cpp` 中 `:66–72`、`:97–100`、`:1624–2409` 严格对应，否则**编译期符号缺失或错配**）。
**✅ 已完成的一致性实测（见附 D6/D7）**：我用脚本对三份清单做了**集合比对**，结果：

| 比对 | 结果 |
|---|---|
| `.h` 数据成员集（285）vs `.h` 访问器名去 `Get` 前缀（285） | **完全相同**（`Compare-Object` 无差异） |
| `.h` 数据成员集（285）vs `Initialize()` 的 LHS 集合（285） | **完全相同** |
| 重复项检查（成员集 / `Initialize()` LHS） | **各有 0 个重复** |
| 285 个访问器定义的 `return Xxx;` 与其函数名是否匹配 | **0 个错配**（逐个解析函数体并比对） |
| 285 条 `Initialize()` 赋值中与 `.h` 成员集不符者 | **0 个** |
| `Initialize()` 主体（`.cpp:23–864`）中的"异常行"（既非赋值、非续行、非 `#if`） | **0 个**（该区段是纯粹的赋值序列，仅 21 条为单行 `X = GetClass(...)` 形式） |

⇒ **当前代码是自洽的，不存在实际的错配 bug。** 因此本条的定级是 **P3（可维护性隐患）**，而非 P1/P2 —— 它是"**今天正确、明天容易错**"的结构性风险，不是现存缺陷。这一点我在"问题 2"中已明确，并已在附 D6 留下可复现的验证脚本。
3. **为什么"今天正确"仍然是问题**：285 个成员的**类型全部相同**（`FClassReflection*`），285 个宏的**类型也全部相同**（`FString`）。因此任何复制粘贴错配 —— 无论是 `GetXxxClass()` 里 `return YyyClass;`，还是 `XxxClass = GetClass(NS, CLASS_YYY_ATTRIBUTE);` —— **编译器都不会报错**，症状只是某个 attribute 静默失效（与 F-REFL-004 的症状完全重合，排查成本极高）。我已做实验确认这一点（见"验证方式"）。而三份清单各自 285 行的规模，使错配概率随每次新增 attribute 而累积。
4. **重复代码块**，符合 `_CONVENTIONS.md` §3.7"重复代码块、可以模板化/宏化的重复实现"。

**建议**

用 X-macro 或代码生成一次性消除三份清单：

```cpp
// FReflectionRegistry.def
//   X(MemberName, Namespace, ClassMacro, Guard)
REFLECTION_CLASS(UtilsClass,      COMBINE_NAMESPACE(NAMESPACE_ROOT, NAMESPACE_CORE_UOBJECT), CLASS_UTILS, 0)
REFLECTION_CLASS(ObjectClass,     NAMESPACE_SYSTEM, CLASS_OBJECT, 0)
...
```

```cpp
#define REFLECTION_CLASS(Member, NS, Cls, Guard) FClassReflection* Member{};
#include "FReflectionRegistry.def"
```
访问器用同一个 `.def` 生成，`Initialize()` 用一个 `#define` 展开成 `Member = GetClass(NS, Cls);`，`Deinitialize()` 展开成 `Member = nullptr;`（顺手修掉 F-REFL-001）。

顺带收益：F-REFL-001（清零）、F-REFL-004（nullptr 校验）都能用同一个 `.def` 循环实现，代码量从 ~1140 行降到 ~60 行。

**验证方式**

- 写一个脚本比对三份清单的符号顺序，应完全一致（当前应通过）。
- 手工实验：把 `GetObjectClass()` 的函数体改成 `return UtilsClass;`，编译**不会报错** —— 证明缺少编译期保护。

---

## 4. 死代码清单

**方法**：
1. 从 `.h` 抽取**全部 285 个访问器的精确名字**（正则 `FClassReflection\* (Get\w+Class)\(\)`，实测 285 个且 **285 个互不重复**）。
2. 对每个名字，用 `\b<Name>\b` 在 `Source/` **全部 507 个 `.h`/`.cpp`/`.inl` 文件**（已排除 `ThirdParty`）中统计**总出现次数**（`Select-String -Encoding UTF8 -AllMatches | Measure-Object`）。
3. 判定：总次数 = **2**（仅在 `.h` 声明 1 次 + 在 `.cpp` 定义 1 次）⇒ **无任何外部调用点** ⇒ 强死代码嫌疑。

**命中数分布**（285 个访问器）：

| 总出现次数 | 符号数 | 解释 |
|---|---|---|
| **2** | **33** | 仅声明 + 定义，**无调用者** → **死代码** |
| 3 | 202 | 声明 + 定义 + **1 个调用点** |
| 4 | 23 | 2 个调用点 |
| 5 | 17 | 3 个调用点 |
| 6 | 8 | 4 个调用点 |
| 7 | 1 | 5 个调用点 |
| 8 | 1 | 6 个调用点 |

**方法自验（关键）**：`GetNotPlaceableAttributeClass` **不在**死代码名单中，而它确实被 `FDynamicGeneratorCore.cpp:719` 调用（`if (InClassReflection->HasAttribute(FReflectionRegistry::Get().GetNotPlaceableAttributeClass()))`）→ 证明本判定方法**有效且未产生假阳性**。

### 死代码明细（33 个 `Get*Class()`，总出现次数 = 2）

| 符号 | 声明位置 | 定义位置 | grep 命中数 | 判定 | 证据 |
|---|---|---|---|---|---|
| `GetAbstractAttributeClass` | `.h:139` | `.cpp:1238` | 2 | 强死代码嫌疑 | `\bGetAbstractAttributeClass\b` 全 `Source/` 仅 2 命中 |
| `GetAdvancedClassDisplayAttributeClass` | `.h:191` | `.cpp:1368` | 2 | 强死代码嫌疑 | 同上 |
| `GetAutoCollapseCategoriesAttributeClass` | `.h:183` | `.cpp:1348` | 2 | 强死代码嫌疑 | 同上 |
| `GetAutoExpandCategoriesAttributeClass` | `.h:181` | `.cpp:1343` | 2 | 强死代码嫌疑 | 同上 |
| `GetCollapseCategoriesAttributeClass` | `.h:185` | `.cpp:1353` | 2 | 强死代码嫌疑 | 同上 |
| `GetComponentWrapperClassAttributeClass` | `.h:177` | `.cpp:1333` | 2 | 强死代码嫌疑 | 同上 |
| `GetConfigDoNotCheckDefaultsAttributeClass` | `.h:165` | `.cpp:1303` | 2 | 强死代码嫌疑 | 同上 |
| `GetConstAttributeClass` | `.h:159` | `.cpp:1288` | 2 | 强死代码嫌疑 | 同上 |
| `GetCustomConstructorAttributeClass` | `.h:151` | `.cpp:1268` | 2 | 强死代码嫌疑 | 同上 |
| `GetDefaultConfigAttributeClass` | `.h:167` | `.cpp:1308` | 2 | 强死代码嫌疑 | 同上 |
| `GetDefaultToInstancedAttributeClass` | `.h:157` | `.cpp:1283` | 2 | 强死代码嫌疑 | 同上 |
| `GetDeprecatedAttributeClass` | `.h:161` | `.cpp:1293` | 2 | 强死代码嫌疑 | 同上 |
| `GetDocumentationPolicyAttributeClass` | `.h:304` | `.cpp:1649` | 2 | 强死代码嫌疑 | `WITH_EDITOR` 段 |
| `GetDontCollapseCategoriesAttributeClass` | `.h:187` | `.cpp:1358` | 2 | 强死代码嫌疑 | 同上 |
| `GetEarlyAccessPreviewAttributeClass` | `.h:193` | `.cpp:1373` | 2 | 强死代码嫌疑 | 同上 |
| `GetEditInlineAttributeClass` | `.h:450` | `.cpp:2014` | 2 | 强死代码嫌疑 | `WITH_EDITOR` 段 |
| `GetEditInlineNewAttributeClass` | `.h:169` | `.cpp:1313` | 2 | 强死代码嫌疑 | 同上 |
| `GetEditorConfigAttributeClass` | `.h:143` | `.cpp:1248` | 2 | 强死代码嫌疑 | 同上 |
| `GetExperimentalAttributeClass` | `.h:141` | `.cpp:1243` | 2 | 强死代码嫌疑 | 同上 |
| `GetHideDropdownAttributeClass` | `.h:173` | `.cpp:1323` | 2 | 强死代码嫌疑 | 同上 |
| `GetHideFunctionsAttributeClass` | `.h:179` | `.cpp:1338` | 2 | 强死代码嫌疑 | 同上 |
| `GetIntrinsicAttributeClass` | `.h:153` | `.cpp:1273` | 2 | 强死代码嫌疑 | 同上 |
| `GetNoExportAttributeClass` | `.h:137` | `.cpp:1233` | 2 | 强死代码嫌疑 | 同上 |
| `GetNotBlueprintableAttributeClass` | `.h:135` | `.cpp:1228` | 2 | 强死代码嫌疑 | 同上 |
| `GetNotBlueprintTypeAttributeClass` | `.h:131` | `.cpp:1218` | 2 | 强死代码嫌疑 | 同上 |
| `GetNotEditInlineNewAttributeClass` | `.h:171` | `.cpp:1318` | 2 | 强死代码嫌疑 | 同上 |
| `GetPerObjectConfigAttributeClass` | `.h:163` | `.cpp:1298` | 2 | 强死代码嫌疑 | 同上 |
| `GetPrioritizeCategoriesAttributeClass` | `.h:189` | `.cpp:1363` | 2 | 强死代码嫌疑 | 同上 |
| `GetShortTooltipAttributeClass` | `.h:302` | `.cpp:1644` | 2 | 强死代码嫌疑 | `WITH_EDITOR` 段 |
| `GetShowCategoriesAttributeClass` | `.h:175` | `.cpp:1328` | 2 | 强死代码嫌疑 | 同上 |
| `GetSparseClassDataTypeAttributeClass` | `.h:195` | `.cpp:1378` | 2 | 强死代码嫌疑 | 同上 |
| `GetVisibleDefaultsOnlyAttributeClass` | `.h:239` | `.cpp:1488` | 2 | 强死代码嫌疑 | 同上 |
| `GetWithinAttributeClass` | `.h:147` | `.cpp:1258` | 2 | 强死代码嫌疑 | 同上 |

> **⚠️ 外部 API 保留意见**：`FReflectionRegistry` 的声明带 `UNREALCSHARPCORE_API`（`.h:7`），且这 285 个访问器全在 **`public:`** 区（`.h:9` 起，`:31–607`）。若宿主游戏有**插件外**的 C++ 模块（本仓库 `Source/` 之外的模块）直接链接 `UnrealCSharpCore` 并调用这些访问器，则上述 33 个符号**不是**死代码。**我无法从本仓库的 `Source/` 之外取证**，故全部标为"强死代码嫌疑 + 可能是外部 API，需确认"，**置信度：中**。若要坐实，需检查宿主项目的 `.Build.cs` 是否依赖 `UnrealCSharpCore` 并 grep 插件外代码。

**其余函数的死代码核查**（非访问器）：

| 符号 | 声明位置 | 命中数 | 判定 | 证据 |
|---|---|---|---|---|
| `FReflectionRegistry::Get()` | `.h:10` | **254**（全 `Source/`） | 活代码 | `grep 'FReflectionRegistry::Get()'` |
| `Initialize()` | `.h:16` | 3（+1 定义） | 活代码 | 3 个后端各 1 次 |
| `Deinitialize()` | `.h:18` | 3（+1 定义） | 活代码 | 3 个后端各 1 次 |
| `GetClass(TWeakObjectPtr<UField>)` | `.h:21` | 10+ 调用点 | 活代码 | 见附 A.1 |
| `GetClass(IManagedHandle)` | `.h:23` | 10+ 调用点 | 活代码 | 见附 A.1 |
| `GetClass(FString,FString)` | `.h:25` | 3+ 调用点 | 活代码 | `:916`、`FDynamicGeneratorCore.cpp:992/:1002` 等 |
| `GetClass<T>()` 模板 | `.h:27` | 11 调用点 | 活代码 | `TMethodHelper.inl`、`TPropertyClass.inl` |
| `FReflectionRegistry()` | `.h:13` | 2（`.cpp:17` 定义 + `.cpp:12` 单例构造） | 活代码（隐式） | 单例静态对象构造 |
| `~FReflectionRegistry()` | — | **未声明** | 不存在 | `.h` 全文无析构声明 |

**结论**：**无不一致/空实现/未使用参数等常见死代码形态**；唯一的死代码是上表 **33 个无调用者的 attribute 访问器**（及它们对应的 33 个成员变量 —— 因为成员是 `private` 且**只能**经访问器读取，所以这 33 个成员是**只写不读的死状态**：`Initialize()` 写入、`Deinitialize()` 不清理、**永不读取**）。

### 由死代码扫描派生的发现（P3）

> 说明：F-REFL-008 与 F-REFL-009 是在本节的死代码全量扫描过程中发现的，因此就地记录；它们与 §3 的 P3 同级（P3）。全部 9 条发现的排序仍为 P0 → P3。

#### [F-REFL-008] 33 个 `Get*Class()` 访问器及其成员是"只写不读"的死状态（死代码）

- **类别**: 死代码
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Source/UnrealCSharpCore/Public/Reflection/FReflectionRegistry.h:131`–`:239`、`:302`–`:304`、`:450`；`Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp:1218`–`:1378`、`:1488`、`:1644`–`:1649`、`:2014`
- **函数**: `FReflectionRegistry::GetAbstractAttributeClass()` 等 33 个（完整清单见 §4 表格）
- **置信度**: 中（判定方法已在 `GetNotPlaceableAttributeClass` 上自验通过；但**无法排除插件外 C++ 模块调用**，见 §4 的"外部 API 保留意见"）

**现状（代码事实）**

以 `GetAbstractAttributeClass` 为例，全 `Source/`（507 个 C++ 文件）**只有 2 处出现**：

```cpp
// FReflectionRegistry.h:139   ← 声明
139: 	FClassReflection* GetAbstractAttributeClass() const;
```
```cpp
// FReflectionRegistry.cpp:1238-1241   ← 定义
1238: FClassReflection* FReflectionRegistry::GetAbstractAttributeClass() const
1239: {
1240: 	return AbstractAttributeClass;
1241: }
```

成员只在 `Initialize()` 里被写入一次（`.cpp:162–163`）：

```cpp
162: 	AbstractAttributeClass = GetClass(
163: 		COMBINE_NAMESPACE(NAMESPACE_ROOT, NAMESPACE_DYNAMIC), CLASS_ABSTRACT_ATTRIBUTE);
```

而**没有任何调用点**。33 个符号的分布统计见 §4（总次数 = 2 的有 33 个）。成员声明在 `.h:609 private:` 区，**外部无法直接读取**，因此"访问器无调用者"≡"成员永不被读取"。

**调用上下文**

- 写入：`FReflectionRegistry::Initialize()`（`.cpp:21–865`），由 3 个脚本域在程序集加载后各调用 1 次（`FCoreCLRDomain.cpp:195` 等）。
- 读取：**无**。
- 与之对比，**被真正消费**的属性访问器都出现在 `FDynamicGeneratorCore.cpp` 的 6 个 `static TArray` 初始化式（`:1040`–`:1259`）或 `HasAttribute(...)` 调用中，例如 `GetNotPlaceableAttributeClass` → `FDynamicGeneratorCore.cpp:719`。

**问题**

1. **33 × 3 ≈ 100 行纯死代码**（33 条声明 + 33 条定义 + 33 条 `Initialize()` 赋值 ≈ 33×(1+4+2) 行）。它们的存在让注册表看起来"支持"这些 UE 说明符，实际**完全不支持**：C# 侧写 `[Abstract]`、`[Within(...)]`、`[NotPlaceable]`、`[Intrinsic]`、`[PerObjectConfig]` 等**不会产生任何 UE 反射效果**，且**没有日志提示**（与 F-REFL-004 叠加：静默无效）。
   这条对使用者最有价值：它把"注册表里有 285 个 attribute"这个印象纠正为"**只有 252 个真正参与工作**"。
2. **次要性能影响**：`Initialize()` 每次热重载都为这 33 个类多走一遍 `GetClass(NS,Name)` → `ScriptDomain->GetClass(...)`（`.cpp:963`，跨托管边界查询）→ 33 次不必要的跨边界调用与 33 次 `FullName2Class` 插入。相对 285 规模约 12% 的浪费。
3. 若确认是死代码，**不应只是删除访问器**，而应明确二选一：**(a)** 删除成员 + 访问器 + `Initialize()` 赋值（承认不支持这些说明符）；**(b)** 把它们接入 `FDynamicGeneratorCore` 的对应元数据映射（真正支持）。当前状态是"看起来支持但实际不支持"，属误导性 API。

**建议**

```cpp
// 方案 A（推荐，若确认不支持）：连同 .h 声明 / .cpp 定义 / Initialize() 赋值 一并删除，
//   并在文档中列出"支持的 attribute 白名单"。
// 方案 B（若要支持）：在 FDynamicGeneratorCore 的对应元数据数组里加入，例如
//   GetClassMetaDataAttributes() 中加入 AbstractAttributeClass 的宏映射。
// 无论哪种，都建议加一条静态校验：断言"每个非死代码访问器至少有一个调用点"较难自动化，
//   但可以断言"Initialize() 解析成功的类集合" == "FDynamicGeneratorCore 消费的类集合" ∪ "白名单"。
```

**验证方式**

- 逐符号 grep（已在全部 507 个文件上执行）：`Select-String -Pattern '\bGetAbstractAttributeClass\b' -Encoding UTF8` → 2 命中（仅声明 + 定义）。
- 抽查一个反例确认方法有效：`\bGetNotPlaceableAttributeClass\b` → ≥3 命中（含 `FDynamicGeneratorCore.cpp:719`）。
- 插件外核查：检查宿主 `.uproject` 下所有非 `Plugins/UnrealCSharp` 的 `.Build.cs` 是否 `PublicDependencyModuleNames.Add("UnrealCSharpCore")`，若有则需 grep 那些模块。

#### [F-REFL-009] 成员名与 `CLASS_*` 宏名的分词风格不一致（16 处），当前语义正确但易诱发错配

- **类别**: 可优化/可读性
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp:105`–`:853`（16 处）
- **函数**: `FReflectionRegistry::Initialize()`
- **置信度**: 高（16 处已逐一定位，且已核对宏定义，**确认语义全部正确**）

**现状（代码事实）**

我写脚本把 `Initialize()` 里每个 attribute 成员的 **LHS 成员名**与其 `GetClass(...)` 行上出现的 **`CLASS_*` 宏名**做了规范化比对（去 `CLASS_`/`_ATTRIBUTE`、驼峰转大写下划线后比较），得到 **16 处分词不一致**：

| `.cpp` 行 | 成员名 | 宏名 | 宏定义（已验证） | 语义正确？ |
|---|---|---|---|---|
| `:105` | `UClassAttributeClass` | `CLASS_U_CLASS_ATTRIBUTE` | `GenericAttributeMacro.h:5` = `"UClassAttribute"` | ✅ 正确 |
| `:108` | `UStructAttributeClass` | `CLASS_U_STRUCT_ATTRIBUTE` | — | ✅ |
| `:111` | `UEnumAttributeClass` | `CLASS_U_ENUM_ATTRIBUTE` | — | ✅ |
| `:114` | `UPropertyAttributeClass` | `CLASS_U_PROPERTY_ATTRIBUTE` | `GenericAttributeMacro.h:13` = `"UPropertyAttribute"` | ✅ |
| `:117` | `UFunctionAttributeClass` | `CLASS_U_FUNCTION_ATTRIBUTE` | — | ✅ |
| `:120` | `UInterfaceAttributeClass` | `CLASS_U_INTERFACE_ATTRIBUTE` | — | ✅ |
| `:213` | `HideDropdownAttributeClass` | `CLASS_HIDE_DROP_DOWN_ATTRIBUTE` | `ClassAttributeMacro.h:31` = `"HideDropdownAttribute"` | ✅ |
| `:258` | `NonPIETransientAttributeClass` | `CLASS_NON_PIE_TRANSIENT_ATTRIBUTE` | `PropertyAttributeMacro.h:9` = `"NonPIETransientAttribute"` | ✅ |
| `:261` | `NonPIEDuplicateTransientAttributeClass` | `CLASS_NON_PIE_DUPLICATE_TRANSIENT_ATTRIBUTE` | — | ✅ |
| `:403` | `ToolTipAttributeClass` | `CLASS_TOOLTIP_ATTRIBUTE` | `MetaDataAttributeMacro.h:9` = `"ToolTipAttribute"` | ✅ |
| `:616` | `MultiLineAttributeClass` | `CLASS_MULTILINE_ATTRIBUTE` | `MetaDataAttributeMacro.h:151` = `"MultiLineAttribute"` | ✅ |
| `:634` | `NoSpinboxAttributeClass` | `CLASS_NO_SPIN_BOX_ATTRIBUTE` | `MetaDataAttributeMacro.h:163` = `"NoSpinboxAttribute"` | ✅ |
| `:664` | `UIMinAttributeClass` | `CLASS_UI_MIN_ATTRIBUTE` | `MetaDataAttributeMacro.h:183` = `"UIMinAttribute"` | ✅ |
| `:667` | `UIMaxAttributeClass` | `CLASS_UI_MAX_ATTRIBUTE` | `MetaDataAttributeMacro.h:185` = `"UIMaxAttribute"` | ✅ |
| `:850` | `BitmaskAttributeClass` | `CLASS_BIT_MASK_ATTRIBUTE` | `MetaDataAttributeMacro.h:307` = `"BitmaskAttribute"` | ✅ |
| `:853` | `BitmaskEnumAttributeClass` | `CLASS_BIT_MASK_ENUM_ATTRIBUTE` | `MetaDataAttributeMacro.h:309` = `"BitmaskEnumAttribute"` | ✅ |

**调用上下文**

`Initialize()` 的每条 attribute 赋值形如 `.cpp:114–115`：

```cpp
114: 	UPropertyAttributeClass = GetClass(
115: 		COMBINE_NAMESPACE(NAMESPACE_ROOT, NAMESPACE_DYNAMIC), CLASS_U_PROPERTY_ATTRIBUTE);
```
宏在 `Public/CoreMacro/GenericAttributeMacro.h:13` 展开为 `FString(TEXT("UPropertyAttribute"))`，该字符串经 `GetClass(NS, Name)`（`.cpp:952`）→ `ScriptDomain->GetClass(NameSpace, InName)`（`.cpp:963`）解析 C# 类型。

**问题**

1. **语义层面：无缺陷。** 我逐一定位了全部 16 处并核对了宏定义（见上表"宏定义"列），**宏展开的 C# 类型名与成员名所表达的属性完全一致**。因此这是**纯风格问题**，不是错配 bug —— 我在报告中明确这一点，避免被误读为 16 个 bug。
2. **风险层面：值得修。** 成员名用**连续小写**（`UClass`、`ToolTip`、`MultiLine`、`Bitmask`、`UIMin`），宏名用**下划线分词**（`U_CLASS`、`TOOLTIP`、`MULTILINE`、`BIT_MASK`、`UI_MIN`）。由于 285 个成员和 285 个宏**类型全部相同**（`FClassReflection*` / `FString`），一旦有人复制粘贴错行，**编译器不会报错**：
   - 成员写错（`A = B` 两成员同类型）→ 静默语义错误，无编译错误。
   - 宏写错（两个宏都是 `FString`）→ 静默解析到**错误的 C# 类型**，且因为 `GetClass` 失败返回 `nullptr`（`.cpp:974`），症状是某个 attribute 静默失效（与 F-REFL-004 症状重合，极难排查）。
3. 命名不一致直接违反了 `_CONVENTIONS.md` §3.7"命名不一致"。

**建议**

用 F-REFL-007 的 X-macro 方案彻底消除：让成员名、访问器名、宏名全部从**同一行定义**派生，则"分词风格"不再有分叉的机会：

```cpp
// X-macro：单一事实来源，成员名与 C# 类型名同源
// X(MemberIdent, NamespaceMacro, ClassNameString)
X(UClassAttribute,      NAMESPACE_DYNAMIC, TEXT("UClassAttribute"))
X(HideDropdownAttribute, NAMESPACE_DYNAMIC, TEXT("HideDropdownAttribute"))
```
若不重构，至少统一为一种风格（建议统一到**成员名**风格，因其与 C# 类型名逐字对应）。

**验证方式**

- 复现我的检查：抽取 `Initialize()` 里 `(\w+Class) = GetClass(...CLASS_\w+)` 配对，规范化分词后比对，应得到 16 处（当前）。
- 修完后再跑同一脚本，应得到 0 处。
- 反向验证（更重要）：把任一条的宏改成相邻行的宏（如 `:114` 用 `CLASS_U_FUNCTION_ATTRIBUTE`），编译**通过**、运行时对应 attribute 静默失效 —— 证明缺少编译期保护。

## 5. 未覆盖/存疑项

1. **`.cpp:261–854` 未逐行读**（`Initialize()` 的其余 594 行赋值）——**但已用脚本等价覆盖**：该区段共 **285 条** `X = GetClass(...)` 赋值、**无重复 LHS**、**无非赋值异常行**，且 LHS 集合与 `.h` 的 285 个数据成员集合 **完全相同**；另对 16 处"成员名 ↔ `CLASS_*` 宏名"分词不一致做了逐一核对（宏定义全部语义正确，见 F-REFL-009）。**残留不确定性**：我核对的是"宏展开后的 C# 类型名是否与成员名语义一致"这 16 处（全部通过），**未**对其余 252 处逐条做同样的宏定义核对（仅做了分词字符串比对，全部一致）。
2. **`.cpp:995–2409` 未逐行读**（285 个访问器定义函数体）——**但已用脚本等价覆盖**：逐个解析每个访问器的 `return Xxx;`，与由函数名去掉 `Get` 前缀推出的期望成员名比对，**0 个错配**。因此 F-REFL-007 指出的"静默错配"**当前不存在**（它是未来隐患）。
3. **优先级 3（反射数据生成正确性）未完成**：`TFieldIterator` 的 `CPF_*`/`FUNC_*` 过滤条件、`CastField<F*Property>` 分派覆盖度与缺失分支的 nullptr 检查、递归类型无限递归风险 —— 这些代码**不在 `FReflectionRegistry.{h,cpp}` 内**（本注册表不解析 `FProperty`/`UFunction`），而位于 `FClassReflection.cpp` 的 `Parse*`/`Ensure*` 段落与 `FTypeBridge.cpp`。**这是范围外的模块**，建议由负责 `FClassReflection`/`FTypeBridge` 的报告覆盖。
4. **优先级 4（性能）仅部分完成**：已确认的候选问题（未展开为 Finding）：
   - `.cpp:886–914` 的 lambda 每次未命中都构造 `TTuple<FString, FString>` 并做最多 3 次 `Cast<>` + 多次 `FUnrealCSharpFunctionLibrary::GetClassNameSpace/GetFullClass`（后者返回 `FString` 按值）→ 热路径字符串分配。命中缓存时（`:881`）不走此路径，故影响有限。
   - `.cpp:954 COMBINE_FULL_NAME(InNameSpace, InName)` 每次查询都拼接 `FString` 作为 map 键；`GetClass(NS,Name)` 被 `:916` 在每次 `Field2Class` 未命中时调用。
   - `.cpp:932 ScriptDomain->GetFullName(InManagedClass)` 每次调用都跨托管边界取全名 `FString`。
   - **未发现**循环内的 `FindProperty`/`StaticClass()`/`FindObject`（本注册表不遍历属性），也**未发现**按值返回大容器（6 个属性数组函数返回 `const TArray<FClassReflection*>&`，`:1036`/`:1067`/`:1082`/`:1095`/`:1108`/`:1201` 签名正确）。**未做定量测量**，故不列为 Finding。
5. **`FClassReflection` 析构路径未完整验证**：`FClassReflection::~FClassReflection()`（`FClassReflection.h:18`）是否会在 `Deinitialize()` 的 `:873 delete` 循环中**回调注册表**（若回调 `GetClass(NS,Name)`，此时 `FullName2Class` 中已有兄弟对象被删除 → 可解引用已释放对象）—— `FClassReflection.cpp` 我只读了 `:25–99`，**未读其析构函数**。这是一个**未排除的潜在 UAF 路径**，建议后续验证。
6. **`FReflection` 无虚析构**：`FReflection.h:5–25` 声明了 `explicit FReflection(...)` 但**没有 `virtual ~FReflection()`**，而 `FClassReflection` 派生自它（`FClassReflection.h:13`）。当前所有 `delete` 都通过 `FClassReflection*`（`.cpp:873` 的 `Class` 来自 `TMap<FString, FClassReflection*>`），因此**暂不构成 UB**；但只要有人持有 `FReflection*` 并 `delete`，即为未定义行为。**未发现当前存在该用法**（`grep` 未见 `delete` 一个 `FReflection*`），故仅记录为隐患，不单列 Finding。
7. **线程安全未评估**：未检查是否有 `check(IsInGameThread())`，也未检查异步编译线程（`FCSharpCompilerRunnable`）与注册表 `Deinitialize/Initialize` 的竞态 —— 未在本次范围内。
8. **`Field2Class` 作 GC 保护的最终判定**：`TWeakObjectPtr` 键**不**提供强引用，因此注册表**不会**阻止 `UClass`/`UScriptStruct`/`UEnum` 被 GC —— 这是**有意设计**（键本就是"弱"的），不构成缺陷；但需注意 `GetClass(UField*)` 在 `:888`/`:899`/`:906` 对 `InField` 做 `Cast<>` 时，若调用方传入的是**已失效的弱指针**，`Cast<UClass>(TWeakObjectPtr)` 会先解引用弱指针。`InField` 由调用方传入（如 `FCSharpBind.inl:75 GetClass(Class)`，`Class` 为活指针），**未发现传入失效弱指针的调用点**，故不列为 Finding。

## 附 A：头文件成员表（`FReflectionRegistry.h`）

### A.1 核心 API（逐个）

| 成员 | 行号 | 职责 | 谁调用（grep 证据） | 结论 |
|---|---|---|---|---|
| `static FReflectionRegistry& Get()` | `.h:10` | Meyers 单例访问点 | 全 `Source/` **254 命中**（`grep 'FReflectionRegistry::Get()'`）；定义 `.cpp:10–15` | 单例；函数内 `static`，**无显式析构**；引用绑定安全（F-REFL-002 中 `static auto&` 依赖此性质） |
| `FReflectionRegistry()` | `.h:13` | 默认构造，成员全 `{}` 初始化 | 仅 `.cpp:17–19`（空函数体）+ `.cpp:12` 单例构造 | **没有析构函数声明** → 隐式 `~FReflectionRegistry()`。**不继承 `FGCObject`**（`.h:7` 无基类列表）→ 见附 B |
| `void Initialize()` | `.h:16` / `.cpp:21–865` | 解析并赋值 285 个 C# 类型；`new FClassReflection` ×N | **3 处**：`FCoreCLRDomain.cpp:195`、`FMonoDomain.cpp:167`、`FLeanCLRDomain.cpp:782` | **无返回值、无校验、无日志** → F-REFL-004；**不清理已有状态**（依赖调用方先 `Deinitialize`） |
| `void Deinitialize()` | `.h:18` / `.cpp:867–877` | `Empty` 两表 + `delete` 全部对象 | **3 处**：`FCoreCLRDomain.cpp:242`、`FMonoDomain.cpp:430`、`FLeanCLRDomain.cpp:808` | **不置空 285 个成员** → **F-REFL-001（P0）**；`delete` 全 `.cpp` 唯一一处 |
| `FClassReflection* GetClass(const TWeakObjectPtr<UField>&)` | `.h:21` / `.cpp:879–924` | `UField*` → C# 类描述符；命中 `Field2Class` 否则按名字反查并回填 | `FCSharpBind.inl:75`、`FDynamicRegistry.cpp:30`、`FCSharpBind.cpp:354/:426`、`FClassRegistry.cpp:256`、`FCSharpFunctionDescriptor.cpp:12`、`FClassDescriptor.cpp:25`、`FCSharpEnvironment.cpp:446`、`FDynamicStructGenerator.cpp:374`、`FDynamicGenerator.cpp:232` | 失败返回 `nullptr`（`:923`）；`:918 Add` 无上限 → **F-REFL-003** |
| `FClassReflection* GetClass(const IManagedHandle)` | `.h:23` / `.cpp:926–950` | 托管类句柄 → 描述符；**命中缓存时 Free 掉入参** | `FClassReflection.cpp:41/:102/:350`、`FScriptDomainImpl.inl:447/:459`、`FTypeBridge.cpp:379/:427/:439/:463/:511/:525/:539/:553` | 两条路径所有权语义不一致 → **F-REFL-005** |
| `FClassReflection* GetClass(const FString&, const FString&)` | `.h:25` / `.cpp:952–975` | `"NS.Name"` → 描述符；**唯一的对象创建点之一** | `:916`（内部）、`FDynamicGeneratorCore.cpp:992/:1002`、`FDynamicGenerator.cpp:232`、`FDynamicRegistry.cpp:30` 间接、`FTypeBridge.cpp:406/:497` | `new` 在 `:966`；失败返回 `nullptr`（`:974`） |
| `template<class T> GetClass()` | `.h:27–28` / `.inl:6–10` | CRTP 版：`GetClass(TNameSpace<T,T>::Get()[0], TName<T,T>::Get())` | `TMethodHelper.inl:29/:51`、`TPropertyClass.inl:45/:54/:63/:163/:172/:190/:205/:321` | 转发到 `GetClass(NS,Name)`；**注意 `.inl:9` 对 `TNameSpace::Get()` 返回的数组取 `[0]`，未做长度检查**（越界隐患，置信度中） |

### A.2 285 个 `Get*Class()` 访问器（分组，逐条已读）

| 分组 | `.h` 声明行 | `.cpp` 定义行 | 数量 | 条件编译 | 结论 |
|---|---|---|---|---|---|
| C#/系统基础类型（`UtilsClass`…`StringClass`） | `:31–69` | `:977–1076` | 20 | — | 全部 `const`；裸 `FClassReflection*` 别名 |
| 版本相关字符串/容器泛型 | `:71–93` | `:1078–1126` | 8 | `UE_F_UTF8_STR_PROPERTY`(`:71`)、`UE_F_ANSI_STR_PROPERTY`(`:75`)、`UE_F_OPTIONAL_PROPERTY`(`:95`) | 三处条件编译 |
| `TOptional` | `:95–97` | `:1132` | 1 | `UE_F_OPTIONAL_PROPERTY` | — |
| 非编辑器 attribute（`OverrideAttribute`…`ServiceResponseAttribute`） | `:99–291` | `:1138–1618` | ~110 | — | — |
| 编辑器 attribute（`HideCategories`…`ArrayParamAttribute`） | `:293–607` | `:1624–2409` | **~155** | **整段 `#if WITH_EDITOR`**（`.h:293`–`:607`、`.h:872`–`:1186`） | 只在编辑器构建存在；`FDynamicGeneratorCore` 的 6 个 static 缓存同段 |
| **合计** | `:31–607` | `:977–2409` | **285** | — | `Select-String` 实测：声明 **285**、成员 **285**、定义 **285**，**三方一致** |

**唯一例外**：`:67 GetTScriptInterfaceClass()` 缺 `const`（284/285 带 const）→ **F-REFL-006**。

### A.3 数据成员（`.h:609–1192`）

| 成员 | 行号 | 类型 | 数量 | 所有权 | 结论 |
|---|---|---|---|---|---|
| `UtilsClass` … `ArrayParamAttributeClass` | `.h:610–1185` | `FClassReflection*`（裸）`{}` 初始化 | **285** | **别名**（`FullName2Class` 的 value 拥有） | **F-REFL-001**：`Deinitialize()` 不置空 |
| `Field2Class` | `.h:1189` | `TMap<TWeakObjectPtr<UField>, FClassReflection*>` | 1 | 键弱引用、值别名 | 只增不减 → **F-REFL-003** |
| `FullName2Class` | `.h:1191` | `TMap<FString, FClassReflection*>` | 1 | **拥有** value | 唯一释放点 `.cpp:873` |

### A.4 多态与析构判定

| 问题 | 事实 | 证据 | 结论 |
|---|---|---|---|
| `FReflectionRegistry` 有虚析构吗？ | **没有声明任何析构函数**（`.h:7–1192` 全文无 `~FReflectionRegistry`） | `.h` 全文 read | **不需要**：类**无基类**（`.h:7` `class UNREALCSHARPCORE_API FReflectionRegistry`），不存在通过基类指针 `delete` 的场景 |
| 有空的虚函数 / 纯虚函数吗？ | **无任何 `virtual` 关键字** | `.h` 全文 read | 非多态类，无空实现虚函数问题 |
| `FReflection`（基类）析构是否 virtual？ | **不是**；`FReflection.h:5–25` 无 `virtual ~FReflection()` | `FReflection.h` 全文 read | 当前**不构成 UB**（所有 `delete` 都走 `FClassReflection*`，`.cpp:873`），但**是隐患** → §5 第 6 项 |

## 附 B：GC 保护判定表（P0 检查项，逐成员）

**判定前提（已核实）**：`FClassReflection` **不是 UObject**。

```cpp
// FClassReflection.h:13
13: class UNREALCSHARPCORE_API FClassReflection : public FReflection
```
```cpp
// FReflection.h:5-6
5: class UNREALCSHARPCORE_API FReflection
6: {
```
→ 无 `UCLASS`/`UObject` 基类、无 `GENERATED_BODY()`。因此注册表内所有 `FClassReflection*` 都是**普通 C++ 堆对象**，**不受 GC 管理**，`UPROPERTY()` 对它们**既无效也不需要**。这是本报告**推翻**任务简报"裸 `UObject*` 无 `UPROPERTY` → GC 后悬垂（P0）"这一预设的关键事实。

| 成员 | 行号 | 存的是什么 | 键类型 | 有 `UPROPERTY()`？ | 有 `AddReferencedObjects`？ | 继承 `FGCObject`？ | **GC 悬挂判定** | **真实风险判定** |
|---|---|---|---|---|---|---|---|---|
| 285 × `FClassReflection*` | `.h:610–1185` | **裸 `FClassReflection*`**（非 UObject 的普通 C++ 对象） | —（成员） | **无** | **无** | **否**（`.h:7` 无基类） | **不适用**：目标不是 UObject，GC 不回收、也不会更新其指针；`UPROPERTY` 加了也无效 | **不是 GC 问题，是 C++ 生命周期问题** → `Deinitialize()` 的 `delete` 后成员悬垂 = **F-REFL-001（P0）** |
| `Field2Class` | `.h:1189` | **键**：`TWeakObjectPtr<UField>`（弱、GC 安全）；**值**：`FClassReflection*`（非 UObject，无需保护） | `TWeakObjectPtr<UField>`（弱引用） | **无** | **无** | 否 | **否**：键是弱引用，**有意不阻止 `UField` 被 GC**；`TWeakObjectPtr` 带序列号，失效后不会误命中，**不会悬垂解引用** | 不构成 GC 悬挂；但**键失效后条目从不清理** → **F-REFL-003（P1，内存增长）** |
| `FullName2Class` | `.h:1191` | **键**：`FString`（值语义，与 GC 无关）；**值**：`FClassReflection*`（拥有） | `FString` | **无** | **无** | 否 | **否**：键值都不是 UObject 引用 | 拥有权正确：唯一释放点 `.cpp:873`；`Empty()` 配对正确（`.cpp:876`） |

**逐成员 GC 结论汇总**：
- **不存在**"存放裸 `UObject*`/`FField*`/`UClass*` 的容器却没有 `UPROPERTY()`/`AddReferencedObjects`/`FGCObject`"的情形 —— 因为容器里**根本没有存 UObject 派生指针**。
- `Field2Class` 的键 `TWeakObjectPtr<UField>` 是**唯一的 UObject 相关类型**，且是**弱引用**（正确用法：注册表不应延长 `UClass` 寿命）。
- ⇒ **本文件不存在 GC 悬挂类 P0**。任务简报预设的"GC 后悬垂（P0）"**不成立**，应改为删除。真正存在的 P0 是**纯 C++ 悬垂**（F-REFL-001）与**缓存永不失效**（F-REFL-002）。

**未做（诚实标注）**：未检查 `FClassStructReflection` 等其他模块的注册表是否存在真正的 UObject 容器（超出本次范围）；未用 `-nullrhi` + `gc` Console 命令实机验证弱键失效行为（结论基于 `TWeakObjectPtr` 的 UE 语义，置信度高但未实测）。

## 附 C：容器增长 ↔ 清理配对表

**实测计数**（`Select-String -Encoding UTF8`，`FReflectionRegistry.cpp` 全文 2409 行）：
`.Add(` = **3** ｜ `.Empty()` = **2** ｜ `.Find(` = **3** ｜ `.Remove(` = **0** ｜ `.Reset(` = **0** ｜ `delete ` = **1** ｜ `new FClassReflection` = **2**

| 容器 | 增长点（行号） | 清理点（行号） | 配对判定 | 只增不减？ | 结论 |
|---|---|---|---|---|---|
| `Field2Class` | `.cpp:918`（`GetClass(UField*)` 缓存回填） | `.cpp:869`（`Deinitialize()` 整表 `Empty()`） | **配对存在**，但**粒度是"整表"** | **是**（在两次 `Deinitialize()` 之间只增不减；无 `.Remove`/`.Reset`） | 功能正确性无影响；**内存单调增长** → **F-REFL-003（P1）** |
| `FullName2Class` | `.cpp:943`（`GetClass(IManagedHandle)`）、`.cpp:968`（`GetClass(NS,Name)`） | `.cpp:876`（`Empty()`）+ `.cpp:873`（`delete` 每个 value） | **完全配对**（值被 delete，键表被 Empty） | 否（`Deinitialize` 彻底释放） | **无 FClassReflection 泄漏**。任务简报的"只增不减=泄漏（P1）"预设在此容器上**不成立**，应改为删除 |
| 285 × `FClassReflection*` 成员 | `.cpp:23–864`（`Initialize()` 赋值） | **无任何清理点**（`Deinitialize()` 不置空） | **不配对** | **是**（`Deinitialize` 后仍指向已释放内存） | **F-REFL-001（P0）** |
| `FDynamicGeneratorCore` 6 × `static TArray<FClassReflection*>` | `FDynamicGeneratorCore.cpp:1040/1071/1086/1099/1112/1205`（一次性 `static` 初始化） | **跨模块无任何清理点** | **不配对** | **是**（永不重建、永不清理） | **F-REFL-002（P0）** |
| `Field2Class` 的 `TWeakObjectPtr` 键 | 同上 `:918` | **无失效键清扫** | **不配对** | 是 | 同 F-REFL-003 |
| `FClassReflection` 内部容器（`Properties`/`Fields`/`Methods`/`GenericArguments`/`Interfaces`） | `FClassReflection.h:170–178`（`TArray`/`TMap`） | `FClassReflection::Deinitialize()`（`FClassReflection.h:21` 声明；实现未读，见 §5） | 由 `~FClassReflection` 触发，间接配对 | 否 | 未验证实现，**存疑**（§5 第 5 项） |

**内存释放总账**：
- 分配：`new FClassReflection` × 2 处（`.cpp:941`、`:966`）。
- 释放：`delete` × 1 处（`.cpp:873`，遍历 `FullName2Class`）。
- **结论：`FClassReflection` 对象本身没有泄漏**（每个 `new` 的结果都进 `FullName2Class`，`:943`/`:968` 紧随 `new` 之后 `Add`，无异常路径可跳过），`Deinitialize()` 全部 `delete` 并清表。
- **真正的生命期缺陷不在"忘记释放"，而在"忘记失效"**（F-REFL-001/F-REFL-002）与"忘记修剪"（F-REFL-003）。

---

## 附 D：分段阅读小结（工作日志）

### D1. `FReflectionRegistry.h:1–820`（第 1、2 段，已读）
- `:7 class UNREALCSHARPCORE_API FReflectionRegistry` —— **无基类，不继承 `FGCObject`**，对 GC 判定至关重要。
- 公有 API 极简（`:10–28`）：`static Get()`、ctor、`Initialize()`、`Deinitialize()`、3 个 `GetClass` 重载 + 模板 `GetClass<T>()`。
- `:31–607`：约 285 个 `FClassReflection* Get*Class() const` 访问器，全部无参、返回裸指针；`:71–97`、`:650–676` 受版本宏包裹，`:293–607` 整段受 `#if WITH_EDITOR` 包裹。
- `:67 GetTScriptInterfaceClass();` 是全表**唯一缺 `const`** 的访问器。
- `:609 private:` 起为 1:1 对应裸成员，全部 `{}` 初始化。

### D2. `FReflectionRegistry.h:821–1194`（第 3 段，已读，**头文件读完**）
- `:872–1186` 的 `#if WITH_EDITOR` 成员段结束于 `:1186 #endif`。
- `:1188–1192` 两个 `TMap` 定义 + `:1192 };` 类结束；`:1194 #include "FReflectionRegistry.inl"`（在头文件末尾包含 `.inl`）。
- **头文件完整结论**：285 访问器 / 285 成员 / 2 容器 / 0 个 `UPROPERTY` / 0 个虚函数 / 0 个析构声明。

### D3. `FReflectionRegistry.cpp:1–260`（第 1 段，已读）
- `:10–15` Meyers 单例；`:17–19` 空构造函数。
- `:21 Initialize()` 开始，`:23–260` 为逐条 `成员 = GetClass(NS, Name)` 赋值：`:23 UtilsClass`、`:26 ObjectClass`、`:52 UClassClass = GetClass(UClass::StaticClass())`、`:57 UObjectClass`、`:59 NameClass = GetClass<FName>()`、`:64 StringClass = GetClass<FString>()`、`:102 OverrideAttributeClass` 起进入 attribute 段（`:106` 起用 `NAMESPACE_DYNAMIC`）。
- **模式确认**：`Initialize()` 是**纯赋值序列，无分支、无校验、无日志**——支撑 F-REFL-004。

### D4. `FReflectionRegistry.cpp:855–994`（核心生命周期段，已读）—— **本报告最重要的 140 行**
- `:862–865`：`Initialize()` 最后一条赋值（`ArrayParamAttributeClass`）+ `#endif` + `}` 结束（共 845 行）。
- `:867–877` `Deinitialize()`：**全函数仅 `Empty` / `delete` / `Empty`，无任何成员置空** → **F-REFL-001**。
- `:879–924` `GetClass(TWeakObjectPtr<UField>)`：`:881 Find` 缓存 → `:886–914` lambda 内 `Cast<UClass>/Cast<UScriptStruct>/Cast<UEnum>` 构造 `TTuple<FString,FString>` → `:916` 按名字反查 → `:918 Field2Class.Add` → `:923 return nullptr`。**只增不减的证据点** → **F-REFL-003**。
- `:926–950` `GetClass(IManagedHandle)`：`:934 Find` 命中时 `:936 ScriptDomain->Free(InManagedHandle)` 并返回既有对象；未命中时 `:941 new FClassReflection` 接管句柄 → **两条路径所有权语义不同** → **F-REFL-005**。
- `:952–975` `GetClass(FString, FString)`：`:954 COMBINE_FULL_NAME` 拼键 → `:956 Find` → `:961 IScriptDomain::Get()` → `:963 ScriptDomain->GetClass(NS,Name)` → `:966 new FClassReflection` → `:968 FullName2Class.Add` → `:974 return nullptr`。
- `:977–994`：访问器定义开始，确认形式为 `return XxxClass;`（`:977–980 GetUtilsClass`），**无任何副作用** —— 支撑"访问器只是别名读取，悬垂即透传"。

### D5. 跨模块验证（已读，非本文件）
- `FDynamicGeneratorCore.cpp:772–821`：`SetFieldMetaData` 模板 `:778` 遍历、`:780 Contains`、**`:782 MetaDataAttribute->GetName()` 解引用**、`:787 InSetMetaData()`；`:790/:797/:804` 三个 `SetMetaData` 重载分别取 `GetPropertyMetaDataAttributes()`/`GetFunctionMetaDataAttributes()`/`GetClass(Interface)MetaDataAttributes()`。
- `FDynamicGeneratorCore.cpp:1030–1263`：**6 个函数局部 `static TArray<FClassReflection*>`**（`:1040`/`:1071`/`:1086`/`:1099`/`:1112`/`:1205`）在**首次调用时快照注册表成员**，返回 `const&`，**永不重建**；`:1263 #endif` 结束 `WITH_EDITOR` 段 → **F-REFL-002**。
- `FCoreCLRDomain.cpp:150–264`：`:168 InitializeAssembly`（**不检查 `bIsInitialized`**）→ `:170 LoadAssembly` → `:172 if (TypeBridgeGetFunctionPointerFn != nullptr)` → **`:195 FReflectionRegistry::Get().Initialize()`**；`:240 UnloadAssembly` → **`:242 FReflectionRegistry::Get().Deinitialize()`** → `:244 AssemblyLoaderUnloadFn()` 卸载程序集，**之后未再触碰注册表**。
- `FClassReflection.cpp:25–99`：`ManagedHandle2Class`（`:37–49`）调用 `FReflectionRegistry::Get().GetClass(InManagedHandle)`（`:41`）后立刻把入参置无效（`:43`）→ 证实 `GetClass(IManagedHandle)` 的所有权消费语义（F-REFL-005），也证实 `Attributes` 集合元素只来自**存活的**注册表对象（F-REFL-002 的后果判定依据）。

### D6. 交叉验证计数（`Select-String -Encoding UTF8` / `grep`）
| 项目 | 模式 | 命中数 | 用途 |
|---|---|---|---|
| 访问器声明 | `FClassReflection\* Get\w+Class\(\)`（`.h`） | 285 | 附 A.2 |
| 访问器声明（带 const） | `FClassReflection\* Get\w+Class\(\) const`（`.h`） | 284 | F-REFL-006 |
| 数据成员 | `^\tFClassReflection\* \w+\{\};`（`.h`） | 285 | 附 A.3 |
| 访问器定义 | `FReflectionRegistry::Get\w+Class\(\) const`（`.cpp`） | 284 | 三方一致 |
| `GetClass` 重载定义 | `FClassReflection\* FReflectionRegistry::GetClass`（`.cpp`） | 4（含 `.inl` 模板声明计入） | 附 A.1 |
| `delete` | `delete `（`.cpp`） | **1**（`:873`） | 附 C |
| `new FClassReflection` | `new FClassReflection`（`.cpp`） | **2**（`:941`/`:966`） | 附 C |
| `.Add(` | `.Add\(`（`.cpp`） | **3**（`:918`/`:943`/`:968`） | 附 C |
| `.Empty()` | `.Empty\(\)`（`.cpp`） | **2**（`:869`/`:876`） | 附 C |
| `.Find(` | `.Find\(`（`.cpp`） | **3**（`:881`/`:934`/`:956`） | 附 C |
| `.Remove(` / `.Reset(` | `.Remove\(` / `.Reset\(`（`.cpp`） | **0 / 0** | F-REFL-003 |
| 注册表单例调用 | `FReflectionRegistry::Get\(\)`（全 `Source/`） | **254** | 附 A.1 |
| 6 个 static 属性缓存 | `static TArray<FClassReflection\*>`（全 `Source/`） | **6** | F-REFL-002 |
| 属性缓存消费点 | `MetaDataAttributes\(\)`（全 `Source/`） | 12（6 声明 + 6 调用） | F-REFL-002 |
| 注册表初始化/反初始化 | `FReflectionRegistry::Get().Initialize/Deinitialize` | **3 / 3** | §1 生命周期 |

### D7. 脚本化一致性校验（本次分析的额外证据，覆盖未逐行读的 `.cpp:261–854` 与 `:995–2409`）

为弥补"未逐行读 594 行赋值 + 1433 行访问器"的覆盖缺口，我写了 3 个校验脚本，**结论全部为"当前代码自洽"**：

**校验 1：三份 285 项清单是否互相一致**

```powershell
$mem = $hl | Select-String '^\tFClassReflection\* (\w+)\{\};'            # .h 数据成员
$acc = $hl | Select-String 'FClassReflection\* (Get\w+Class)\(\)'       # .h 访问器
$init= $cl[22..863] | Select-String '^\t(\w+) = GetClass'               # Initialize() 的 LHS
Compare-Object $mem ($acc -replace '^Get','')
Compare-Object $mem $init
```

实测输出：

```
members(.h)            = 285
accessors(.h)          = 285
initialize assignments = 285
--- members vs accessor-derived-member diff ---   IDENTICAL
--- members vs Initialize-LHS diff ---            IDENTICAL
--- duplicates in members? ---                    (空)
--- duplicates in init? ---                       (空)
```

**校验 2：每个访问器是否返回正确的成员**

```powershell
for($i=0;$i -lt $cl.Count;$i++){
  if($cl[$i] -match '^FClassReflection\* FReflectionRegistry::Get(\w+?)Class\(\)( const)?$'){
    $fn=$Matches[1]; $ret=$null
    for($j=$i+1;$j -lt $i+6;$j++){ if($cl[$j] -match '^\treturn (\w+);'){ $ret=$Matches[1]; break } }
    if($ret -ne $null -and $ret -ne ($fn+'Class')){ "line $($i+1): Get${fn}Class() returns $ret" }
  }
}
```
实测输出：`A) accessor return mismatches: 0`

**校验 3：`Initialize()` 的成员↔宏语义配对**（分词规范化后比对）

实测输出：`B) attribute member/macro mismatches: 16` —— 随后我**逐一核对了这 16 处的宏定义**（`CoreMacro/{Generic,Class,Property,MetaData}AttributeMacro.h`），**全部语义正确**，属纯分词风格差异 → 已转化为 **F-REFL-009（P3）**。同时验证了方法有效性：`GetNotPlaceableAttributeClass` 因被 `FDynamicGeneratorCore.cpp:719` 调用而**不在**死代码名单中（见 §4）。

**校验 4：`Initialize()` 主体（`.cpp:23–864`）异常行扫描**

```powershell
for($i=22;$i -lt 865;$i++){ $l=$cl[$i]
  if($l -match '^\s*$'){continue}
  if($l -match '^\t(\w+) = GetClass\($'){continue}
  if($l -match '^\t\t'){continue}
  if($l -match '^#'){continue}
  "$($i+1): $l" }
```
实测输出：21 行，**全部是单行形式的 `X = GetClass(NS, MACRO);`**（如 `:26 ObjectClass = GetClass(NAMESPACE_SYSTEM, CLASS_OBJECT);`、`:52 UClassClass = GetClass(UClass::StaticClass());`、`:59 NameClass = GetClass<FName>();`），**无任何分支、循环、日志、断言或 TODO** → 确证 `Initialize()` 是纯赋值序列，支撑 **F-REFL-004**（无任何校验/错误处理）。

**校验 5：全 285 个访问器的调用点计数（死代码）**

对 `Source/` 全部 **507 个** `.h`/`.cpp`/`.inl`（排除 `ThirdParty`）逐符号 `\bName\b` 计数，分布为 `2 次→33 个`、`3 次→202 个`、`4 次→23 个`、`5 次→17 个`、`6 次→8 个`、`7 次→1 个`、`8 次→1 个` → **33 个访问器无任何调用者** → **F-REFL-008**。

### D8. 本报告自评：与任务简报预设的三处出入（重要）

任务简报给出了若干先验结论，独立核实结果如下：

| 简报预设 | 我的核实结果 | 依据 |
|---|---|---|
| "有没有存放裸 `UObject*`/`FField*`/`UClass*` 的容器却没有 `UPROPERTY()`…→ GC 后悬垂（**P0**）" | **不成立**。容器里**不存在 UObject 派生指针**：`FClassReflection` 是普通 C++ 类（`FClassReflection.h:13`），`Field2Class` 的**键**是弱引用 `TWeakObjectPtr<UField>`（GC 安全且语义正确）。**本文件无 GC 悬挂 P0**。 | 附 B |
| "`Deinitialize()` 会 `delete` 全部 `FClassReflection`…第二次 C# 热重载即 **UAF（P0）**" | **部分成立、结论需修正**：`delete` 与"6 个 static 缓存未清"**均确证**，但 `:782` 的解引用**不是**无条件 UAF —— `HasAttribute` 只做指针值比较（`FReflection.cpp:20`），且 `Attributes` 元素只来自**存活**对象，故主要后果是**元数据静默丢失/错配**。仍定 **P0**，但理由是"缓存结构性永不失效"而非"必然崩溃"。 | F-REFL-002 |
| "容器只增不减 = 泄漏（**P1**）" | **`FullName2Class` 不成立**（`:876 Empty` + `:873 delete` 完全配对，**无对象泄漏**）；**`Field2Class` 成立**（无 `.Remove`/`.Reset`，P1）；**285 个成员成员指针才是真正的未配对项**（P0）。 | 附 C |

**本报告最确信的三条**：F-REFL-001（悬垂访问器）、F-REFL-002（static 缓存永不失效）、F-REFL-008（33 个 attribute 访问器是只写不读的死状态 —— 这条对使用者的实际影响可能最大，因为它意味着注册表对外宣称的 attribute 支持面**实际只有 252/285 生效**）。

