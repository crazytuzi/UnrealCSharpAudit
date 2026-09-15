# 属性描述符（二）：字符串 / 枚举 / 结构体 / Optional

> 本报告各条**严重度见各 Finding 的「复核结论」字段**（16 条：证伪并撤销 1 条、级别调整 3 条 = 上调 2 条 / 下调 1 条、其余 12 条维持）。
>
> ⚠️ **编号撞号警告（引用本报告必须带报告路径）**：本报告使用的发现前缀 `F-PROP2-*` 与 [`05-属性描述符-对象与委托类型.md`](05-属性描述符-对象与委托类型.md) **共用同一前缀，且编号区间重叠（001–015）**，同一编号在两侧指向**完全不同的发现**（例：两侧的 `F-PROP2-004` 分别是「6 个 `Identical` 不查空指针」与「`FFieldPathPropertyDescriptor` 无实现」）。任何引用都必须写成 `01-UnrealCSharp运行时/04b-属性描述符-字符串枚举结构体Optional.md#F-PROP2-00X` 这类带报告路径的全限定形式。
> 本报告实际使用的编号：`F-PROP2-001` … `F-PROP2-016`（连续、无缺号、本报告内无重号；其中 `F-PROP2-016` 为 05 报告所没有的编号）。





> 分析范围：`Source/UnrealCSharp/{Public,Private}/Reflection/Property/{StringProperty,EnumProperty,StructProperty,OptionalProperty}`
> 覆盖文件：16 个（5×String + Enum + Struct + Optional，每个 .h/.cpp），**全部读完**（共 558 行）
> 阅读情况：16 个文件（5×String + Enum + Struct + Optional）全部读完
> 引擎对照版本：`Engine\Source`（UE 5.6；全部引擎行号已在该路径下核实）
> 发现：**16 条** = 撤销（非缺陷）×1 / P0 ×2 / P1 ×2 / P2 ×4 / P3 ×7
> （口径说明：以上为**最终级别**；初判分布为 P0 ×1（待验证） / P1 ×5 / P2 ×3 / P3 ×7，与最终分布的差异来自 `F-PROP2-004`/`F-PROP2-005` 定为 P0、`F-PROP2-006` 下调为 P2、`F-PROP2-001` 撤销。）

## 0. 覆盖范围与阅读清单

行数取自 `read` 工具输出的 `total N lines`（注意 `Get-Content | Measure-Object -Line` **不计空行**会低估，例如 `FStrPropertyDescriptor.h` 实测 19 行而非 13 行）。

| 文件 | 行数 | 是否读完 | 备注 |
|---|---|---|---|
| `Public/.../StringProperty/FAnsiStrPropertyDescriptor.h` | 22 | ✅ 全读 | `#if UE_F_ANSI_STR_PROPERTY` (`:6`) |
| `Private/.../StringProperty/FAnsiStrPropertyDescriptor.cpp` | 51 | ✅ 全读 | 守卫 `:2`–`:51` |
| `Public/.../StringProperty/FNamePropertyDescriptor.h` | 19 | ✅ 全读 | |
| `Private/.../StringProperty/FNamePropertyDescriptor.cpp` | 49 | ✅ 全读 | `FReturn` 路径不泄漏（FName 无堆成员） |
| `Public/.../StringProperty/FStrPropertyDescriptor.h` | 19 | ✅ 全读 | |
| `Private/.../StringProperty/FStrPropertyDescriptor.cpp` | 49 | ✅ 全读 | 5 个 String 描述符的基准实现 |
| `Public/.../StringProperty/FTextPropertyDescriptor.h` | 19 | ✅ 全读 | |
| `Private/.../StringProperty/FTextPropertyDescriptor.cpp` | 49 | ✅ 全读 | `Identical` 用 `EqualTo` |
| `Public/.../StringProperty/FUtf8StrPropertyDescriptor.h` | 22 | ✅ 全读 | `#if UE_F_UTF8_STR_PROPERTY` (`:6`) |
| `Private/.../StringProperty/FUtf8StrPropertyDescriptor.cpp` | 51 | ✅ 全读 | 守卫 `:2`–`:51` |
| `Public/.../EnumProperty/FEnumPropertyDescriptor.h` | 21 | ✅ 全读 | 派生自 `TPrimitivePropertyDescriptor` |
| `Private/.../EnumProperty/FEnumPropertyDescriptor.cpp` | 26 | ✅ 全读 | **无 `Identical` 重写** |
| `Public/.../StructProperty/FStructPropertyDescriptor.h` | 24 | ✅ 全读 | 私有 `NewRef` |
| `Private/.../StructProperty/FStructPropertyDescriptor.cpp` | 71 | ✅ 全读 | 构造期 `Bind`，本次最长文件 |
| `Public/.../OptionalProperty/FOptionalPropertyDescriptor.h` | 22 | ✅ 全读 | 守卫 `:5`–`:22` |
| `Private/.../OptionalProperty/FOptionalPropertyDescriptor.cpp` | 44 | ✅ 全读 | **无 `Identical` 重写** |

**为得出结论额外阅读的支撑文件**（不在 16 个目标内，用于确定所有权与调用约定）：

| 文件 | 用途 |
|---|---|
| `Public/Registry/FStringRegistry.h` / `.inl` | `AddStringReference` 两个 bool 的语义、`RemoveReference` 的释放行为（§4.1 的核心证据） |
| `Public/Registry/FStructRegistry.h` / `.inl`、`FOptionalRegistry.inl` / `Private/Registry/FOptionalRegistry.cpp` | 结构体/Optional 的所有权模型 |
| `Public/Environment/FCSharpEnvironment.h` / `.inl` | `Bind<false>`、`GetString/GetStringObject/AddStringReference/GetStruct/GetOptional` 的转发 |
| `Public/Reflection/Property/TCompoundPropertyDescriptor.inl`、`FPropertyDescriptor.cpp` | `CopyValue`/`InitializeValue`/`DestroyValue`/`Factory` |
| `Public/Macro/FunctionMacro.h`、`PropertyMacro.h` | `Get<FReturn>` 与 `Set` 的实参来源、`NEW_PROPERTY_DESCRIPTOR` 宏拼接 |
| `Private/Registry/FCSharpBind.inl` / `.cpp` | 结构体绑定的重入守卫与登记顺序（§6.1） |
| `Private/Reflection/Optional/FOptionalHelper.h` / `.cpp` | Optional 的构造/析构与 `has-value` 处理（§8） |
| `Private/Domain/Interop/FRegisterString.cpp` / `FRegisterName.cpp` / `FRegisterText.cpp` | 托管边界的所有权移交（`new FString`）与 `FName`/`FText` 语义 |
| `Private/Reflection/Container/FArrayHelper.cpp` / `FMapHelper.cpp`、`Private/Reflection/Function/FManagedFunctionDescriptor.cpp` | `Set` 的真实调用点（证明 `Dest` 是活内存） |
| `Source/CrossVersion/Public/UEVersion.h` | 版本宏取值（`:150,:152`） |

**未读**：`Script/`（C# 侧）、`Source/ThirdParty/`、`*Property` 下的 Primitive/Container/Delegate/Object/FieldPath 各描述符（仅按需读了与结论相关的片段）。

## 1. 模块职责与架构速览

**这些描述符是什么**：插件把 UE 的 `FProperty` 包装成 `FPropertyDescriptor` 子类，让 C# 侧能读写任意 UE 属性。每个描述符实现同一组虚函数（注意：**实际方法名是 `Get`/`Set`/`Identical`/`CopyValue`/`DestroyValue`，不是任务书泛称的 `GetValue`/`SetValue`**）：

| 虚函数 | 方向 | 用途 |
|---|---|---|
| `Get(void* Src, void** Dest, FMember)` | 原生 → C# | 读成员属性，**缓存/别名语义** |
| `Get(void* Src, void** Dest, FReturn)` | 原生 → C# | 读返回值/out 参数，**所有权移交语义** |
| `Get(void* Src, void* Dest)` | 原生 → C# | 值语义读出（primitive 走这条） |
| `Set(void* Src, void* Dest)` | C# → 原生 | 写入 |
| `Identical(A, B, PortFlags)` | — | 容器元素比较（`Find`/`Contains`/`Remove`） |
| `CopyValue(InAddress)` | — | 深拷贝一份原生值，供 `FReturn` 使用 |
| `GetBufferSize()` | — | 告诉宏每个参数在缓冲里占多少字节 |

**继承体系**（决定每个描述符拿到哪些默认实现）：
```
FPropertyDescriptor                      (FPropertyDescriptor.h/.cpp)
├── TPropertyDescriptor<T, IsPrimitive>  (TPropertyDescriptor.inl)
│   ├── TPrimitivePropertyDescriptor<T>  (TPrimitivePropertyDescriptor.inl)
│   │   └── FEnumPropertyDescriptor              ← 纯值语义，不走 registry
│   └── TCompoundPropertyDescriptor<T>   (TCompoundPropertyDescriptor.inl)
│       ├── FStr / FName / FText / FAnsiStr / FUtf8Str PropertyDescriptor   ← registry 别名 + CopyValue 缓冲
│       ├── FStructPropertyDescriptor
│       └── FOptionalPropertyDescriptor
```
`FPropertyDescriptor::Factory`（`FPropertyDescriptor.cpp:47-134`）按 `CastField<T>` 依次试探，经 `NEW_PROPERTY_DESCRIPTOR(X)` 宏（`PropertyMacro.h:5`）拼接出 `X##Descriptor` 类名并 `new`——**注意这个宏拼接使"按类名 grep"无法证明任何描述符是死代码**。

**数据流（核心）**：C# 侧的字符串/结构体/Optional **不是值拷贝也不是 `std::string`**，而是：

```
原生内存 (UObject 属性块 / Malloc 缓冲)  ──Get──▶  托管对象 (C# class 实例)  ──Set──▶  原生内存
        │                                              │
        └── 登记表: Address2ManagedHandle (void* → IManagedHandle)  ← 以裸地址为键
                         ManagedHandle2Value (IManagedHandle → {T*, bNeedFree})
```

关键在于 `Src` **是谁的内存**，这决定了正确性：
- `Get(FMember)`：`Src` = **UObject 自己的属性内存** → 只能别名（`IsNeedFree=false` + `IsMember=true` 登记缓存）；
- `Get(FReturn)`：`Src` = `CopyValue()` 用 `FMemory::Malloc` 现分配的**新缓冲**（`TCompoundPropertyDescriptor.inl:35-45`）→ 可以移交所有权（`IsNeedFree=true`，不登记缓存）。

**调用链入口**：属性读写不直接暴露给 C#，而是经 `Macro/FunctionMacro.h` 的宏在反射调用时装配。`PROCESS_SCRIPT_IN`（`:84-87`）先 `INITIALIZE_VALUE()`（`:65-69`，对每个参数 `InitializeValue_InContainer`）再 `IN_VALUE()`（`:71-73`，`Set`）；`PROCESS_RETURN`（`:136-147`）/`PROCESS_OUT`（`:116-134`）按 `IsPrimitiveProperty()` 分流到 2 参 `Get` 或 `CopyValue`+3 参 `Get`。

## 2. 关键调用链

1. **C# 设置一个成员字符串属性**
   `FRegisterProperty.cpp:35` → `FPropertyDescriptor::Set(IN_BUFFER, ContainerPtrToValuePtr<void>(FoundAddress))` → `FStrPropertyDescriptor::Set`（`FStrPropertyDescriptor.cpp:29`）→ `GetString<FString>(handle)`（`:33`）→ `Property->InitializeValue(Dest)`（`:35`，**见 F-PROP2-002**）→ `Property->SetPropertyValue(Dest, *SrcValue)`（`:37`）

2. **C# 读取一个成员字符串属性（别名）**
   任意绑定取值路径 → `FStrPropertyDescriptor::Get(Src, Dest, FMember)`（`:4`）→ `GetStringObject<FString>(Src)`（`:6`，查 `Address2ManagedHandle`，**见 F-PROP2-007**）→ 未命中则 `Class->NewObject()`（`:10`）+ `AddStringReference<FString,false,true>`（`:12`）

3. **UFunction 返回一个 FString（所有权移交）**
   `FunctionMacro.h:143-145`（`PROCESS_RETURN`）→ `CopyValue(ContainerPtrToValuePtr<void>(Params))`（`TCompoundPropertyDescriptor.inl:33-48`，`Malloc`+`InitializeValue`+`CopySingleValue`）→ `FStrPropertyDescriptor::Get(..., FReturn)`（`FStrPropertyDescriptor.cpp:19`）→ `AddStringReference<FString,true,false>`（`:23`）→ …C# 侧 `FString.UnRegister`
   → `FRegisterString.cpp:38` → `RemoveStringReference<FString>` → `FStringRegistry.inl:69` `FMemory::Free`（**见 F-PROP2-003：缺 `~FString()`**）

4. **C# 写入 `TArray<FString>` 的某个现存元素**
   `FArrayHelper::Set(int32 Index, void* InValue)`（`FArrayHelper.cpp:133`）→ `ScriptArrayHelper.IsValidIndex(Index)`（`:135`）→ `InnerPropertyDescriptor->Set(InValue, ScriptArrayHelper.GetRawPtr(Index))`（`:137`，**Dest 是活 FString**）→ 回到链 1 的 `InitializeValue`（**F-PROP2-002 的触发点**）。对照：同文件 `RemoveAt`（`:208-228`）在复用/释放元素前**正确地**调用了 `DestroyValue`（`:218`）

5. **C# 比较容器里的字符串/结构体元素**
   `FArrayHelper::Find`（`:141-154`）/`Contains`/`IndexOf` → `InnerPropertyDescriptor->Identical(Item, InValue)`（`:148`）→ `FStrPropertyDescriptor::Identical`（`:41`）→ `GetString<FString>(handle)`（`:45`）→ `*StringB`（`:48`，**未查空 → F-PROP2-004**）

6. **C# 读取结构体成员属性（引用语义）**
   `FStructPropertyDescriptor::Get(Src, Dest, FMember)`（`FStructPropertyDescriptor.cpp:10`）→ `Class->NewObject()`（`:12`）→ `AddStructReference<false>(Property->Struct, Src, Object)`（`:14`，`IsNeedFree=false` → C# 拿到的是**原生内存别名**）；对照 `FReturn` 分支 `:23` 用 `AddStructReference<true>`（值语义）

7. **结构体描述符构造 → 递归绑定**
   `FPropertyDescriptor::Factory`（`FPropertyDescriptor.cpp:89`）→ `new FStructPropertyDescriptor(Property)` → 构造体 `Bind<false>(Property->Struct)`（`.cpp:7`）→ `FCSharpBind::Bind<UStruct>`（`FCSharpBind.inl:9`）→ 早退守卫 `GetClassDescriptor`（`:11`）→ `BindImplementation`（`FCSharpBind.cpp:106`）→ `AddClassDescriptor`（`:122`，**先登记**）→ 属性循环（`:159-182`）→ `AddPropertyHash`（`:175`）→ **回到 Factory**（自引用结构体在此靠 `:122` 的先登记避免无限递归，见 §6.1）

8. **C# 写入一个 `TOptional<T>` 属性**
   `FOptionalPropertyDescriptor::Set`（`FOptionalPropertyDescriptor.cpp:34`）→ `GetOptional(handle)`（`:38`，**未查空 → F-PROP2-005**）→ `Property->InitializeValue(Dest)`（`:40`）→ `Property->CopyCompleteValue(Dest, SrcOptional->GetData())`（`:42`）
   另：`FOptionalHelper` 的销毁路径 `FOptionalHelper.cpp:27-29`（析构）→ `Deinitialize()`（`:36-55`）→ `FMemory::Free(Data)`（`:40`，**缺 `DestroyValue`**）

## 3. 逐描述符清单

> 行号来源：`read` 工具实际输出（`total N lines`）。注意 `Get-Content | Measure-Object -Line` 不计空行，会低估，以 read 为准。

### 3.1 五个 String 描述符总览

| 类 | 文件:行 | 关键方法 | 做了什么 | 调用方(grep 证据) | 结论 |
|---|---|---|---|---|---|
| `FStrPropertyDescriptor` | `Public/.../StringProperty/FStrPropertyDescriptor.h:5-19`<br>`Private/.../FStrPropertyDescriptor.cpp:4-49` | `Get(FMember)`:4、`Get(FReturn)`:19、`Set`:29、`Identical`:41 | 用 `AddStringReference<FString,false,true>`(成员) / `<FString,true,false>`(返回值) 把原生 `FString*` 包成托管对象；`Set` 先 `InitializeValue` 再 `SetPropertyValue` | 见 §9 死代码表 | 与 FName/FText/Ansi/Utf8 **逐字符同构**（仅模板实参不同）；`Set` 存在 `InitializeValue` 重复构造隐患，见 `F-PROP2-001` |
| `FNamePropertyDescriptor` | `.../FNamePropertyDescriptor.h:5-19`<br>`.../FNamePropertyDescriptor.cpp:4-49` | 同上 | 同上，模板实参 `FName` | 同上 | 同上；`Identical` 用 `NameA == *NameB`（`FName` 比较是索引比较，大小写不敏感语义正确） |
| `FTextPropertyDescriptor` | `.../FTextPropertyDescriptor.h:5-19`<br>`.../FTextPropertyDescriptor.cpp:4-49` | 同上 | 同上，模板实参 `FText`；`Identical`(41) 用 `TextA.EqualTo(*TextB)` | 同上 | `EqualTo` 的默认比较级别是 **`ETextComparisonLevel::Default`**（引擎 `Runtime/Core/Public/Internationalization/Text.h:570`；枚举成员定义见 `TextComparison.h:8-19`，注释为 "Locale-specific Default" —— **不存在 `DisplayString` 这一级**），语义上仍**非 key 级**；描述符本身无损（别名原生 `FText`），但 C# 常用的 `ToString`→`new FText` 往返会丢失 key/namespace，见 `F-PROP2-006` |
| `FAnsiStrPropertyDescriptor` | `.../FAnsiStrPropertyDescriptor.h:6-22`<br>`.../FAnsiStrPropertyDescriptor.cpp:2-51` | 同上 | 同上，模板实参 `FAnsiString`；**整文件被 `#if UE_F_ANSI_STR_PROPERTY` 包裹** | 同上 | 版本守卫存在（`.h:6`、`.cpp:2`、`.cpp:51`），无 P1 编译失败风险，但需确认宏定义来源 |
| `FUtf8StrPropertyDescriptor` | `.../FUtf8StrPropertyDescriptor.h:6-22`<br>`.../FUtf8StrPropertyDescriptor.cpp:2-51` | 同上 | 同上，模板实参 `FUtf8String`；**整文件被 `#if UE_F_UTF8_STR_PROPERTY` 包裹** | 同上 | 同上 |

**取值模式（5 个描述符完全一致）**

```cpp
// Private/Reflection/Property/StringProperty/FStrPropertyDescriptor.cpp:4-17
void FStrPropertyDescriptor::Get(void* Src, void** Dest, FPropertyArgument::FMember) const
{
	auto Object = FCSharpEnvironment::GetEnvironment().GetStringObject<FString>(Src);   // 6: 从"原生地址 → 托管句柄"缓存表查

	if (!IManagedHandleIsValid(Object))                                                 // 8: 查空
	{
		Object = Class->NewObject();                                                    // 10

		FCSharpEnvironment::GetEnvironment().AddStringReference<FString, false, true>(  // 12: <T, bReturn?, bMember?>
			Class, Object, Src);
	}

	*reinterpret_cast<IManagedHandle*>(Dest) = Object;                                  // 16
}
```

```cpp
// Private/Reflection/Property/StringProperty/FStrPropertyDescriptor.cpp:19-27  —— 返回值语义：每次新建
void FStrPropertyDescriptor::Get(void* Src, void** Dest, FPropertyArgument::FReturn) const
{
	const auto Object = Class->NewObject();                                             // 21
	FCSharpEnvironment::GetEnvironment().AddStringReference<FString, true, false>(      // 23
		Class, Object, Src);
	*reinterpret_cast<IManagedHandle*>(Dest) = Object;
}
```

```cpp
// Private/Reflection/Property/StringProperty/FStrPropertyDescriptor.cpp:29-39
void FStrPropertyDescriptor::Set(void* Src, void* Dest) const
{
	const auto SrcManagedHandle = *static_cast<IManagedHandle*>(Src);                   // 31

	if (const auto SrcValue = FCSharpEnvironment::GetEnvironment().GetString<FString>(SrcManagedHandle))  // 33: 查空
	{
		Property->InitializeValue(Dest);                                                // 35: 对已初始化内存重新构造！
		Property->SetPropertyValue(Dest, *SrcValue);                                    // 37
	}
}
```

```cpp
// Private/Reflection/Property/StringProperty/FStrPropertyDescriptor.cpp:41-49
bool FStrPropertyDescriptor::Identical(const void* A, const void* B, const uint32 PortFlags) const
{
	const auto StringA = Property->GetPropertyValue(A);                                 // 43
	const auto StringB = FCSharpEnvironment::GetEnvironment().GetString<FString>(       // 45: 返回值未查空
		*static_cast<IManagedHandle*>(const_cast<void*>(B)));
	return StringA == *StringB;                                                         // 48: 未查空直接解引用
}
```

### 3.2 `FStructPropertyDescriptor`

| 类 | 文件:行 | 关键方法 | 做了什么 | 调用方(grep 证据) | 结论 |
|---|---|---|---|---|---|
| `FStructPropertyDescriptor` | `Public/Reflection/Property/StructProperty/FStructPropertyDescriptor.h:5-24`<br>`Private/.../FStructPropertyDescriptor.cpp:4-71` | 构造:4、`Get(FMember)`:10、`Get(FReturn)`:19、`Get(Src,Dest)`:28、`Set`:33、`Identical`:45、`NewRef`:55 | 构造时 `Bind<false>(InProperty->Struct)`；成员取值走 `NewRef` 缓存（`GetObject(Struct,Address)` 查重），返回值/Out 参数走 `AddStructReference<true>` 所有权移交 | `Macro/FunctionMacro.h:128-130,143-145` | 拷贝用 `CopySingleValue`（深拷贝，**未使用 memcpy，无双重释放**，见 §6）；但 `Identical`:49 未查空、`Set`:39 `InitializeValue` 重复构造 |

```cpp
// Private/Reflection/Property/StructProperty/FStructPropertyDescriptor.cpp:4-8
FStructPropertyDescriptor::FStructPropertyDescriptor(FStructProperty* InProperty) :
	TCompoundPropertyDescriptor(InProperty)
{
	FCSharpEnvironment::GetEnvironment().Bind<false>(InProperty->Struct);   // 7: 构造期递归入口
}
```
```cpp
// Private/Reflection/Property/StructProperty/FStructPropertyDescriptor.cpp:33-43
void FStructPropertyDescriptor::Set(void* Src, void* Dest) const
{
	const auto SrcManagedHandle = *static_cast<IManagedHandle*>(Src);                       // 35
	if (const auto SrcStruct = FCSharpEnvironment::GetEnvironment().GetStruct<>(SrcManagedHandle))  // 37: 查空 ✓
	{
		Property->InitializeValue(Dest);                                                     // 39: 对已初始化内存再构造
		Property->CopySingleValue(Dest, SrcStruct);                                          // 41: 深拷贝 ✓
	}
}
```
```cpp
// Private/Reflection/Property/StructProperty/FStructPropertyDescriptor.cpp:45-53
bool FStructPropertyDescriptor::Identical(const void* A, const void* B, const uint32 PortFlags) const
{
	const auto StructA = Property->ContainerPtrToValuePtr<void>(A);                          // 47: 又加了一次偏移
	const auto StructB = FCSharpEnvironment::GetEnvironment().GetStruct<>(                  // 49: 未查空 ✗
		*static_cast<IManagedHandle*>(const_cast<void*>(B)));
	return Property->Identical(StructA, StructB, PortFlags);                                 // 52: StructB 为空则解引用
}
```
```cpp
// Private/Reflection/Property/StructProperty/FStructPropertyDescriptor.cpp:55-71
IManagedHandle FStructPropertyDescriptor::NewRef(void* InAddress) const
{
	auto Object = FCSharpEnvironment::GetEnvironment().GetObject(Property->Struct, InAddress);   // 57: 按地址查缓存
	if (!IManagedHandleIsValid(Object))
	{
		Object = Class->NewObject();
		const auto OwnerManagedHandle = FCSharpEnvironment::GetEnvironment().GeManagedHandle(   // 63
			InAddress, Property);
		FCSharpEnvironment::GetEnvironment().AddStructReference(OwnerManagedHandle, Property->Struct,
		                                                        InAddress, Object);              // 66
	}
	return Object;                                                                            // 70
}
```

### 3.3 `FEnumPropertyDescriptor`

| 类 | 文件:行 | 关键方法 | 做了什么 | 调用方(grep 证据) | 结论 |
|---|---|---|---|---|---|
| `FEnumPropertyDescriptor` | `Public/Reflection/Property/EnumProperty/FEnumPropertyDescriptor.h:5-21`<br>`Private/.../FEnumPropertyDescriptor.cpp:4-26` | `Get(FMember)`:4、`Get(FReturn)`:9、`Get(Src,Dest)`:14、`Set`:19、`DestroyValue`:24 | 取值统一 `Class->BoxValue(Src)` 装箱；拷贝走 `Property->GetUnderlyingProperty()->CopySingleValue`；`DestroyValue` 空实现 | `Macro/FunctionMacro.h:121-131,137-146` | **不走任何 registry，每次调用都新建装箱对象**；`GetUnderlyingProperty()` 未查空；`DestroyValue` 空实现见 §7 |

```cpp
// Private/Reflection/Property/EnumProperty/FEnumPropertyDescriptor.cpp:4-22
void FEnumPropertyDescriptor::Get(void* Src, void** Dest, FPropertyArgument::FMember) const
{
	*Dest = IManagedHandleToObject(Class->BoxValue(Src));                       // 6
}

void FEnumPropertyDescriptor::Get(void* Src, void** Dest, FPropertyArgument::FReturn) const
{
	*Dest = IManagedHandleToObject(Class->BoxValue(Src));                       // 11  ← 与 6 完全相同
}

void FEnumPropertyDescriptor::Get(void* Src, void* Dest) const
{
	Property->GetUnderlyingProperty()->CopySingleValue(Dest, Src);              // 16
}

void FEnumPropertyDescriptor::Set(void* Src, void* Dest) const
{
	Property->GetUnderlyingProperty()->CopySingleValue(Dest, Src);              // 21
}
```
> ⚠ `Get(FMember)` 与 `Get(FReturn)` **逐字节相同**（`4-7` vs `9-12`）：枚举属性没有"返回值是否需要新建对象"的区别，`FEnumProperty` 不是 compound 类型语义。这一点与 String/Struct/Optional 形成鲜明对照（见 §5）。

### 3.4 `FOptionalPropertyDescriptor`

| 类 | 文件:行 | 关键方法 | 做了什么 | 调用方(grep 证据) | 结论 |
|---|---|---|---|---|---|
| `FOptionalPropertyDescriptor` | `Public/Reflection/Property/OptionalProperty/FOptionalPropertyDescriptor.h:10-21`<br>`Private/.../FOptionalPropertyDescriptor.cpp:5-43` | `Get(FMember)`:5、`Get(FReturn)`:22、`Set`:34 | 用 `new FOptionalHelper(Property, Src, bNeedFreeData, false)` 包装；`AddOptionalReference` 的模板参数控制是否登记地址缓存；`Set` 用 `CopyCompleteValue` | `Macro/FunctionMacro.h:128-130,143-145` | 无 `Identical` 重写；`Set`:38 未查空 → 空指针（`F-PROP2-005`）；helper 析构缺 `DestroyValue` → 泄漏（`F-PROP2-002`） |

```cpp
// Private/Reflection/Property/OptionalProperty/FOptionalPropertyDescriptor.cpp:5-43
void FOptionalPropertyDescriptor::Get(void* Src, void** Dest, FPropertyArgument::FMember) const
{
	auto Object = FCSharpEnvironment::GetEnvironment().GetOptionalObject<FOptionalHelper>(Src);  // 7
	if (!IManagedHandleIsValid(Object))
	{
		Object = Class->NewObject();
		const auto OptionalHelper = new FOptionalHelper(Property, Src, false, false);            // 13: new
		FCSharpEnvironment::GetEnvironment().AddOptionalReference<FOptionalHelper, true>(        // 15: IsNeedFree=true
			Src, OptionalHelper, Class, Object);
	}
	*reinterpret_cast<IManagedHandle*>(Dest) = Object;                                           // 19
}

void FOptionalPropertyDescriptor::Get(void* Src, void** Dest, FPropertyArgument::FReturn) const
{
	const auto Object = Class->NewObject();
	const auto OptionalHelper = new FOptionalHelper(Property, Src, true, false);                 // 26: new
	FCSharpEnvironment::GetEnvironment().AddOptionalReference<FOptionalHelper, false>(           // 28: IsNeedFree=false
		Src, OptionalHelper, Class, Object);
	*reinterpret_cast<IManagedHandle*>(Dest) = Object;
}

void FOptionalPropertyDescriptor::Set(void* Src, void* Dest) const
{
	const auto SrcManagedHandle = *static_cast<IManagedHandle*>(Src);
	const auto SrcOptional = FCSharpEnvironment::GetEnvironment().GetOptional(SrcManagedHandle); // 38: 未查空 ✗
	Property->InitializeValue(Dest);                                                             // 40: 重复构造
	Property->CopyCompleteValue(Dest, SrcOptional->GetData());                                   // 42: 空则崩
}
```

## 4. 字符串返回值所有权表（核心）

### 4.1 三个自由度的定义（来自实际代码）

`AddStringReference<T, IsNeedFree, IsMember>` 的两个 bool 的语义由 `FStringRegistry.inl:38-53` 与 `RemoveReference`（`FStringRegistry.inl:55-82`）确定：

| 模板参数 | 含义 | 证据 |
|---|---|---|
| `IsMember` | `true` 时额外把 **原生地址 → 托管句柄** 写进 `Address2ManagedHandle`（做缓存/别名） | `FStringRegistry.inl:42-45`（`if constexpr (IsMember)` 内 `(InRegistry->*Address2ManagedHandle).Add(InAddress, InManagedHandle)`） |
| `IsNeedFree` | `true` 时该 `FString*` **由插件自己堆分配**，`RemoveReference` 会 `FMemory::Free` 它；`false` 表示只是别名，绝不释放 | `FStringRegistry.inl:67-72`；分配点见 `FRegisterString.cpp:15` `new FString(...)` |

两个方向的映射表：
- `ManagedHandle2Value`（`FStringRegistry.h:79,83,88,94,99`）：托管句柄 → `TStringAddress<T*>{Value, bNeedFree}`。
- `Address2ManagedHandle`（`FStringRegistry.h:81,85,90,96,101`）：原生 `void*` 地址 → 托管句柄。

### 4.2 逐路径所有权表

| 描述符 | 方法(行) | 传入的 `Src` 实际是什么 | `IsNeedFree` | `IsMember` | 谁负责释放 | 判定 |
|---|---|---|---|---|---|---|
| `FStrPropertyDescriptor` | `Get(FMember)` `FStrPropertyDescriptor.cpp:4-17` | 调用方给出的**原生 `FString` 地址**（成员属性块地址） | `false` | `true` | 无人（纯别名） | ✅ 语义正确（别名，不释放），但引入地址键缓存失效问题，见 `F-PROP2-007` |
| `FStrPropertyDescriptor` | `Get(FReturn)` `:19-27` | `CopyValue()` 的返回：`FMemory::Malloc(ElementSize)` + `InitializeValue` + `CopySingleValue`（`TCompoundPropertyDescriptor.inl:33-48`） | `true` | `false` | 托管侧 `UnRegister` → `FMemory::Free` | ⚠ 指针释放配对了，但 **`~FString()` 从未执行 → 内部字符数组永久泄漏**，见 `F-PROP2-003` |
| `FNamePropertyDescriptor` | `Get(FMember)` `FNamePropertyDescriptor.cpp:4-17` | 原生 `FName` 地址 | `false` | `true` | 无人 | ✅ 同 FStr |
| `FNamePropertyDescriptor` | `Get(FReturn)` `:19-27` | `CopyValue()` malloc 缓冲 | `true` | `false` | `FMemory::Free` | ✅ `FName` 无非平凡成员，**不泄漏**（与 FStr/FText 不同，值得注意：同一份代码对不同 T 的安全性不同） |
| `FTextPropertyDescriptor` | `Get(FMember)` `FTextPropertyDescriptor.cpp:4-17` | 原生 `FText` 地址 | `false` | `true` | 无人 | ✅ 同 FStr |
| `FTextPropertyDescriptor` | `Get(FReturn)` `:19-27` | `CopyValue()` malloc 缓冲 | `true` | `false` | `FMemory::Free` | ⚠ 同 FStr：`FText` 持有 `TSharedRef<ITextData>` 引用计数，**不析构即泄漏一份共享引用**，见 `F-PROP2-003` |
| `FAnsiStrPropertyDescriptor` | `Get(FMember)` `FAnsiStrPropertyDescriptor.cpp:5-18` | 原生 `FAnsiString` 地址 | `false` | `true` | 无人 | ✅（仅 UE≥5.6 编译） |
| `FAnsiStrPropertyDescriptor` | `Get(FReturn)` `:20-28` | `CopyValue()` malloc 缓冲 | `true` | `false` | `FMemory::Free` | ⚠ 同 FStr，泄漏内部数组 |
| `FUtf8StrPropertyDescriptor` | `Get(FMember)` `FUtf8StrPropertyDescriptor.cpp:5-18` | 原生 `FUtf8String` 地址 | `false` | `true` | 无人 | ✅（仅 UE≥5.6 编译） |
| `FUtf8StrPropertyDescriptor` | `Get(FReturn)` `:20-28` | `CopyValue()` malloc 缓冲 | `true` | `false` | `FMemory::Free` | ⚠ 同 FStr，泄漏内部数组 |
| `FStructPropertyDescriptor` | `Get(FMember)` `FStructPropertyDescriptor.cpp:10-17` | 原生结构体地址 | `false`（`AddStructReference<false>`） | 隐式登记（`FStructRegistry.inl:19-20` 无条件写缓存） | 无人（别名） | 见 §6 |
| `FStructPropertyDescriptor` | `Get(FReturn)` `:19-26` | `CopyValue()` malloc 缓冲 | `true`（`AddStructReference<true>`，`.cpp:23`） | 同上 | `bNeedFree` 触发释放 | 所有权移交，见 §6 |
| `FOptionalPropertyDescriptor` | `Get(FMember)` `FOptionalPropertyDescriptor.cpp:5-20` | 原生 `TOptional` 地址 | helper 参数 `bNeedFreeData=false`（`:13`） | **`IsMember=true`**（`:15`）→ 登记地址缓存 | 注册表 `delete` helper（`FOptionalRegistry.cpp:69`） | ✅ 配对正确 |
| `FOptionalPropertyDescriptor` | `Get(FReturn)` `:22-32` | `CopyValue()` malloc 缓冲 | helper 参数 `bNeedFreeData=true`（`:26`）→ helper 析构时 `FMemory::Free(Data)` | **`IsMember=false`**（`:28`）→ 不登记缓存（正确，缓冲地址不应被缓存） | 注册表 `delete` helper | ✅ 所有权配对正确；**但仍缺 `DestroyValue`**，泄漏 `TOptional` 内部值，见 `F-PROP2-002` |

> ⚠ **说明**：`AddOptionalReference<FOptionalHelper, bool>` 的 bool 是 **`IsMember`**，不是 `IsNeedFree` —— 本节曾据后者得出"`Get(FReturn)` 的 helper 泄漏"的结论，该结论**不成立**。核实 `FOptionalRegistry.inl:6-19` 与 `FOptionalRegistry.cpp:69` 后确认：该 bool 只控制是否写 `Address2ManagedHandle`；helper **总是**被存入 `ManagedHandle2Helper` 并在 `RemoveReference` 中 `delete`，因此**不存在 helper 泄漏**。发现清单按该事实编号：空指针类问题为 `F-PROP2-004`/`F-PROP2-005`。

### 4.3 关于"临时对象悬垂"的结论

**未发现** `GetValue` 返回 `std::string`/`char*` 内部指针的悬垂模式——本插件不用 `std::string` 传字符串，而是走 **托管句柄（`IManagedHandle`）+ 注册表**：

| 检查项 | 结论 | 证据 |
|---|---|---|
| 是否把 `FString` 转成 `std::string`/`char*` 后返回其内部指针 | **否**。`Dest` 写入的是 `IManagedHandle`（`*reinterpret_cast<IManagedHandle*>(Dest) = Object`） | 5 个描述符 `Get` 的末行，如 `FStrPropertyDescriptor.cpp:16,26` |
| 是否存在 `const char* GetData()` 式的裸指针出口 | 存在但**立即被拷贝**：`FRegisterString.cpp:46` `TCHAR_TO_UTF8(**String)` 传给 `IScriptDomain::NewString(...)`，托管侧拷贝 | `FRegisterString.cpp:42-47` |
| 别名（`IsMember=true`）是否可能悬垂 | **是**：托管对象内部保存原生 `FString*`（`FStringRegistry.h:24` `TStringAddress<FString*>`），原生对象被 GC/销毁后 C# 仍可读 | `FStringRegistry.inl:47-50` |
| `GetProperty()` 返回值是否检查 | **这些描述符不调用 `GetProperty()`**；改为直接解引用 `Property`（来自 `TPropertyDescriptor<T,E>` 的成员），**未做 null 检查** | `FStrPropertyDescriptor.cpp:35,37,43`；`FStructPropertyDescriptor.cpp:41,47`；`FOptionalPropertyDescriptor.cpp:40` |
| 字符串属性是否用 `memcpy` 搬运 | **否**。全部走 `SetPropertyValue`/`CopySingleValue`/`CopyCompleteValue`，没有 `FMemory::Memcpy` | `grep "Memcpy\|memcpy" Private/Reflection/Property/StringProperty/*.cpp` → 0 命中 |

> 因此任务书里预设的 **"字符串 `memcpy` 双重释放 = P0"** 在**这 5 个描述符中不成立**（已核实 0 命中）。真正的问题是 §4.2 表中的**析构缺失型泄漏**与 `Set` 的 `InitializeValue` 重复构造。

## 5. 成对方法一致性对照表

| 描述符 | `Get` 是否查空 | `Set` 是否查空 | 用 `GetElementSize()` 还是 `ElementSize` | 返回值所有权 | 备注 |
|---|---|---|---|---|---|
| `FStrPropertyDescriptor` | ✅ `GetStringObject` 结果查 `IManagedHandleIsValid`（`:8`）；`Identical`(41-49) **不查空** | ✅ `GetString<FString>` 用 `if` 查空（`:33`） | 自身不用；基类 `CopyValue` 用 `GetElementSize()`（`TCompoundPropertyDescriptor.inl:36-40`，有 `UE_F_PROPERTY_GET_ELEMENT_SIZE` 守卫） | 别名/移交，见 §4.2 | `Identical`:48 未查空 |
| `FNamePropertyDescriptor` | ✅ `:8`；`Identical`(41-49) 不查空 | ✅ `:33` | 同上 | 同上 | 与 FStr 逐字同构 |
| `FTextPropertyDescriptor` | ✅ `:8`；`Identical`(41-49) 不查空 | ✅ `:33` | 同上 | 同上 | `Identical`:48 用 `EqualTo`（文本级比较，非 key 级） |
| `FAnsiStrPropertyDescriptor` | ✅ `:9`；`Identical`(42-50) 不查空 | ✅ `:34` | 同上 | 同上 | 版本守卫 ✓ |
| `FUtf8StrPropertyDescriptor` | ✅ `:9`；`Identical`(42-50) 不查空 | ✅ `:34` | 同上 | 同上 | 版本守卫 ✓ |
| `FStructPropertyDescriptor` | ✅ `GetStruct<>` 查空（`:37`）；`Identical`(45-53) **不查空** | ✅ `:37` | 同上 | 移交（`AddStructReference<true>`） | **`Identical`:52 把可能为 null 的 `StructB` 传入 `Property->Identical`** |
| `FEnumPropertyDescriptor` | ❌ 无 registry 查询，无查空对象 | ❌ `DestroyValue` 空实现；`GetUnderlyingProperty()` 未查空（`:16,21`） | **不涉及**（不做 buffer 拷贝，用 `CopySingleValue` 到原生地址） | **装箱数值，无缓存**（每次新建） | `Get(FMember)`/`Get(FReturn)` 完全相同；无 `Identical` 重写 |
| `FOptionalPropertyDescriptor` | ✅ `GetOptionalObject` 查空（`:9`）；`Set` 中 `GetOptional` **不查空**（`:38`） | ❌ `:38` 未查空即 `:42` 解引用 | 同上 | `FMember` 移交 helper 所有权；`FReturn` **既不释放也不移交** | 无 `Identical` 重写 |

**不一致点汇总（每条对应一个 Finding）**

| # | 不一致 | 涉及位置 | Finding |
|---|---|---|---|
| 1 | `Set` 中 `GetStruct<>` 查空，但 `Identical` 中同样的 `GetStruct<>` 不查空 | `FStructPropertyDescriptor.cpp:37` vs `:49-52` | `F-PROP2-003` |
| 2 | `FOptionalPropertyDescriptor::Set` 中 `GetOptional` 不查空（同文件 `Get` 路径却查空） | `FOptionalPropertyDescriptor.cpp:9` vs `:38,42` | `F-PROP2-003` |
| 3 | 字符串 `Set` 查空、但 `Identical` 不查空（5 个文件同病） | `FStrPropertyDescriptor.cpp:33` vs `:45-48`（FName/FText/Ansi/Utf8 同） | `F-PROP2-003` |
| 4 | `FOptionalPropertyDescriptor` 两条 `Get` 的 `new`/`IsNeedFree` 组合相反 | `.cpp:13,15` vs `:26,28` | `F-PROP2-004` |
| 5 | `Get(FMember)` 对 String 系列**复用缓存**，对 Struct/Optional 也复用，对 Enum **每次新建** | `FEnumPropertyDescriptor.cpp:6` vs `FStrPropertyDescriptor.cpp:6-16` | `F-PROP2-006` |
| 6 | 所有 `Set` 一律 `InitializeValue(Dest)`（对已初始化内存） | `FStrPropertyDescriptor.cpp:35`、`FNamePropertyDescriptor.cpp:35`、`FTextPropertyDescriptor.cpp:35`、`FAnsiStrPropertyDescriptor.cpp:36`、`FUtf8StrPropertyDescriptor.cpp:36`、`FStructPropertyDescriptor.cpp:39`、`FOptionalPropertyDescriptor.cpp:40` | `F-PROP2-002` |

> **编号对应**：上表第 1/3 行引用 `F-PROP2-003`；空指针类问题编号为 `F-PROP2-004`/`F-PROP2-005`；Optional 的 `AddOptionalReference` bool 实为 `IsMember`，**不是 bug**。详见 §4.2 末尾的说明。

## 6. 结构体描述符专项

### 6.1 递归 / 自引用结构体 → **无无限递归**（已核实，给出证据链）

`FStructPropertyDescriptor` 构造函数（`.cpp:7`）无条件调用 `Bind<false>(InProperty->Struct)`，这是一个**构造期重入**入口。递归链与守卫顺序：

```
FPropertyDescriptor::Factory (FPropertyDescriptor.cpp:89)
  → new FStructPropertyDescriptor(Property)          // 构造开始，类描述符尚未登记?
      → ctor 体 .cpp:7  Bind<false>(Property->Struct)
          → FCSharpBind::Bind<UStruct> (FCSharpBind.inl:9-27)
              → if (GetClassDescriptor(InStruct)) return true;   // inl:11-14 ★早退守卫
              → BindImplementation(InStruct) (FCSharpBind.cpp:106)
                  → AddClassDescriptor(InStruct)               // :122  ★先登记
                  → for (Properties) AddPropertyHash(...)      // :159-182 ★后才建 PropertyDescriptor
```

**结论**：`AddClassDescriptor`（`FCSharpBind.cpp:122`）**先于**属性描述符创建循环（`:159-182`，`AddPropertyHash` at `:175` 才触发 `FPropertyDescriptor::Factory`）。因此自引用结构体（如 `TArray<FSelf>`、`FSelf*` 之外的嵌套）第二次进入 `Bind` 时，`GetClassDescriptor` 已在 `:122` 登记并命中，在 `instantiation.inl:11-14` 直接 `return true` → **不会无限递归**。递归深度上界 = 结构体静态嵌套深度（有界）。

> 但这条**依赖"先登记后建描述符"的顺序**，没有 assert/注释保护。若将来有人把 `AddClassDescriptor` 移到属性循环之后，会立刻变成栈溢出。建议加 `check(!BindingInProgress.Contains(InStruct))` 或注释固化该不变量（见 `F-PROP2-016`）。

### 6.2 构造 / 析构语义

| 检查项 | 结论 | 证据 |
|---|---|---|
| `InitializeStruct`/`DestroyStruct` 是否配对 | **描述符自身不调用**；`CopyValue` 只 `InitializeValue`（`TCompoundPropertyDescriptor.inl:43`）从不 `DestroyValue`；`FStructPropertyDescriptor::Set`（`.cpp:39`）也只 `InitializeValue` | `grep "DestroyStruct\|InitializeStruct"` → 仅 `FCSharpBind.cpp:412`（对象创建路径） |
| 拷贝是否用 `memcpy` | **没有直接 memcpy**。走 `Property->CopySingleValue`（`.cpp:41`）——但该调用对非平凡结构体是否安全**取决于 UE 侧实现**，见 `F-PROP2-001` | `grep "FMemory::Memcpy\|memcpy"` 在 `Reflection/Property/` 下 **0 命中** |
| 大小 / 对齐 | `CopyValue`（`TCompoundPropertyDescriptor.inl:35-41`）用 `FMemory::Malloc(GetElementSize())`，**未传对齐参数**。`FOptionalHelper` 分配时却传了对齐（`FOptionalHelper.cpp:21` `Malloc(size, GetMinAlignment())`）→ **同一插件两处不一致** | `TCompoundPropertyDescriptor.inl:35-41` vs `FOptionalHelper.cpp:21` |
| 值语义 vs 引用语义 | **两种都有，按调用点分**：`Get(FMember)`:14 `AddStructReference<false>` → C# 拿到的是**原生内存别名（引用语义）**；`Get(FReturn)`:23 `AddStructReference<true>` → 传入 `CopyValue` 缓冲，**C# 拥有该拷贝（值语义）** | `.cpp:14,23`；`FStructRegistry.inl:15-29` 中 `IsNeedFree` 只被存进 `FStructAddress::bNeedFree` |

### 6.3 `GetStruct<>` 只做 static_cast，无类型校验

`FCSharpEnvironment.inl:127-131`：`GetStruct<T>` → `static_cast<T*>(StructRegistry->GetStruct(handle))`。`FStructPropertyDescriptor` 用 `GetStruct<>`（**空模板参数**，`.cpp:37,49`）→ 返回 `void*`，**不做结构体类型匹配**。若 C# 侧把一个 `FVector` 的托管对象传给一个期望 `FRotator` 的属性 `Set`，原生侧会把 `FVector` 的内存按 `FRotator` 解释（同样 12 字节，静默数据错误）。`FStructRegistry` 的键是 `(UScriptStruct*, void*)`（`FStructRegistry.h:5-8`），说明类型信息是有的，但 `Set` 路径没有用它校验。

---

## 7. 枚举描述符专项

| 问题 | 结论 | 证据 |
|---|---|---|
| 底层类型（uint8/int64）映射 | 通过 `Property->GetUnderlyingProperty()` **原样**拷贝底层字节；底层类型由 UE 的 `FEnumProperty` 决定（`FByteProperty`/`FIntProperty`/`FInt64Property`…） | `FEnumPropertyDescriptor.cpp:16,21` |
| `GetValue` 返回数值还是名字 | **返回装箱对象**，不是名字：`Class->BoxValue(Src)` → `IManagedHandleToObject(...)`。名字↔数值的映射完全交给 C# 侧 | `FEnumPropertyDescriptor.cpp:6,11` |
| `Enum`/`UEnum` 为空（`FEnumProperty` 可以没有 `UEnum`） | 代码**从不触碰** `Property->GetEnum()`，因此为空的 `UEnum` 在这里是安全的；但也意味着**无法按名字解析**，C# 拿到的是裸数值 | `grep "GetEnum()"` 在 EnumProperty 目录 → 0 命中 |
| `GetUnderlyingProperty()` 是否可能为空 | **未查空**（`.cpp:16,21`）。`FEnumProperty` 构造时必然创建底层属性，故实际风险低；但同一插件的 `Set` 家族其它描述符都查空，此处不一致 | `.cpp:16,21` |
| C# 枚举不存在时的回退 | **无回退、不抛异常**：`BoxValue` 装箱的是底层数值，即使 C# 侧没有对应枚举类型也能拿到数值（`Class` 为枚举的 C# 类反射） | `.cpp:6,11` |
| `DestroyValue` 空覆盖 | 空实现（`.cpp:24-26`），基类实现（`FPropertyDescriptor.cpp:190-196`）对枚举同样是无操作 → **冗余覆盖**，见 `F-PROP2-010` | 对比 `FPropertyDescriptor.cpp:190-196` |
| `Identical` | **未重写**，落到基类 `FPropertyDescriptor::Identical`（`FPropertyDescriptor.cpp:180-188`）→ 走原生 `FProperty::Identical` ✓ | — |

---

## 8. Optional 描述符专项

**先纠正一个容易误判的点**：`AddOptionalReference<T, bool>` 的 bool 是 **`IsMember`**，不是 `IsNeedFree`。

```cpp
// Public/Registry/FOptionalRegistry.inl:6-19
template <auto IsMember>
auto FOptionalRegistry::AddReference(..., const FOptionalHelperValueMapping::ValueType& InValue, ...)
{
	if constexpr (IsMember) { Address2ManagedHandle.Add(InAddress, InManagedHandle); }   // 11-14: 仅控制缓存
	ManagedHandle2Helper.Add(InManagedHandle, InValue);                                   // 16: ★总是存
	return true;
}
```
`RemoveReference`（`FOptionalRegistry.cpp:55-77`）**总是** `delete *FoundValue`（`:69`）。因此 `FOptionalPropertyDescriptor::Get(FReturn)` 里的 `new FOptionalHelper(...)`（`.cpp:26`）**不会泄漏**——helper 的所有权归注册表。

| 问题 | 结论 | 证据 |
|---|---|---|
| `has-value` 标志与值构造/析构是否配对 | **不配对**。`FOptionalHelper::~FOptionalHelper` → `Deinitialize()`（`FOptionalHelper.cpp:29,36-55`）里只 `FMemory::Free(Data)`，**从不调用 `OptionalProperty->DestroyValue(Data)`** → `TOptional<FString>`/`TOptional<TArray<…>>` 的内部堆数据泄漏 | `FOptionalHelper.cpp:38-43` |
| `SetValue` 清空旧值是否析构 | **不析构**。`FOptionalPropertyDescriptor::Set`（`.cpp:40-42`）先 `InitializeValue(Dest)` 再 `CopyCompleteValue(Dest, ...)`，既没有对 `Dest` 原有的 optional 调 `DestroyValue`，也没有 `MarkUnset` → 覆盖旧值时泄漏旧值；且若源为 unset，`CopyCompleteValue` 会写入空值而非 unset | `.cpp:40-42` |
| 元素为非平凡类型时的行为 | 泄漏（同上）；此外 `FOptionalHelper::Reset()`（`FOptionalHelper.cpp:72-75`）走 `MarkUnset` ✓ 这条路径是正确的，与 `Set` 路径不一致 | `.cpp:72-75` |
| `GetOptional` 是否查空 | `Get` 路径查空（`.cpp:9`），`Set` 路径**不查空**（`.cpp:38`）→ 见 `F-PROP2-005` | — |
| `FOptionalHelper::GetData()` 语义 | 返回 `Data`，即**`TOptional` 自身的地址**（不是值的地址）；`Get()` 才返回值指针 | `FOptionalHelper.cpp:99-107`, `:82-85` |
| `FOptionalHelper::Initialize()` | **空实现且全插件 0 调用点** → 死代码，见 `F-PROP2-011` | `FOptionalHelper.cpp:32-34`；`grep "Helper->Initialize\(\)"` = 0 |

---

## 9. 死代码清单

**方法**：`grep`（`Select-String -Recurse`，范围 `Source/` 全 7 模块，`*.cpp,*.h,*.inl`）。命中数 `==1`（仅声明或仅定义）记为强嫌疑。

| 符号 | 声明位置 | grep 模式 | 命中数 | 判定 | 证据 |
|---|---|---|---|---|---|
| `FOptionalHelper::Initialize()` | `FOptionalHelper.cpp:32-34` | `(Helper\|OptionalHelper)->Initialize\(\)` | **0** | **死代码（强）** | 空函数体，无任何调用点；仅有 `Deinitialize` 被析构调用 |
| `FEnumPropertyDescriptor::DestroyValue` | `FEnumPropertyDescriptor.cpp:24-26` | `DestroyValue\(` | 10 | 可达但**冗余**（空覆盖 == 基类行为） | 调用点 `FArrayHelper.cpp:218`、`FMapHelper.cpp:119,121,215`、`FSetHelper.cpp:140` |
| `FStructPropertyDescriptor::Get(void*, void*)` | `.cpp:28-31` | `Descriptor->Get\(.*,\s*Dest\)` | 0（模式不精确） | **存疑**：未能确证 2 参 `Get` 对非 primitive 描述符的调用点 | 宏里 non-primitive 走 3 参版（`FunctionMacro.h:128,143`） |
| `FStructPropertyDescriptor::NewRef` | `.cpp:55` | `NewRef\(` | 18（含同名它项） | 非死代码 | 定义内调用点 `.cpp:30` |
| 5×String `Identical` | `FStrPropertyDescriptor.cpp:41` 等 | `Descriptor->Identical\(` | 11 | 非死代码 | `FArrayHelper.cpp:148,164,180,293`、`FSetHelper.cpp:87,124,154`、`FMapHelper.cpp:101,137,164,183` |
| 8×描述符类 | 各自 `.h:5/7/10` | 类名字面量 | 仅 `.h`+`.cpp`+include | **非死代码** | `FPropertyDescriptor::Factory`（`FPropertyDescriptor.cpp:89,93,95,98,102,105,130`）经 `NEW_PROPERTY_DESCRIPTOR(FStructProperty)` 宏**拼接**类名（`PropertyMacro.h:5`）`FPropertyType##Descriptor` |
| `CopyValue` | `TCompoundPropertyDescriptor.inl:33` / `TPrimitivePropertyDescriptor.inl:43` | `CopyValue\(` | 6 | 非死代码 | `FunctionMacro.h:129,144` |
| `GetBufferSize` | `TCompoundPropertyDescriptor.inl:50` | `GetBufferSize\(` | 39 | 非死代码 | `FunctionMacro.h:73,132` |
| `InitializeValue_InContainer` | `FPropertyDescriptor.inl:57` | `InitializeValue_InContainer` | 5 | 非死代码 | `FunctionMacro.h:69`、`FCSharpFunctionDescriptor.cpp:44` |

> `PropertyMacro.h:5` 的宏拼接意味着**按类名 grep 会把所有描述符误判为死代码**——这是本项目做死代码审计最容易踩的坑，特此记录。

---

## 10. 发现清单

> 分节标题按**每条发现的最终级别**标注；分节**排列顺序**沿用初判的编号顺序（`F-PROP2-*` 编号是跨报告引用锚点，不做块重排），因此 `### P0` / `### P1` / `### P2` 之间不是严格的全局降序 —— 以每条 finding 的 `**严重度**` 字段为权威值。

### 撤销（非缺陷）

#### [F-PROP2-001] `FStructPropertyDescriptor::Set` 对含非平凡成员的结构体依赖 `CopySingleValue`，若落到 `FProperty::CopySingleValue` 的 memcpy 兜底即为双重释放 —— **已证伪：非缺陷，请从排期移除**

- **类别**: 内存/资源泄漏 | 未定义行为
- **严重度**: **撤销（非缺陷）**
- **复核结论**: 证伪（撤销，非缺陷）—— 假设 B（落到 `FProperty::CopySingleValue` 的 memcpy 兜底）在 UE 5.6 中**不成立**
- **可达性**: 不可达（所担心的 `memcpy` 兜底分支不存在，任何配置下都不会发生双重释放）
- **复核证据**: 引擎三段链路 —— `Runtime/CoreUObject/Public/UObject/UnrealType.h:822-835`（`FProperty::CopySingleValue` 仅在 `CPF_IsPlainOldData` 时走 `FMemory::Memcpy`，否则转 `CopyValuesInternal`）；`UnrealType.h:5970`（`FStructProperty` **确实** `override CopyValuesInternal`）；`Runtime/CoreUObject/Private/UObject/PropertyStruct.cpp:337-340`（`CopyValuesInternal` → `Struct->CopyScriptStruct(Dest, Src, Count)`，即脚本结构体深拷贝）
- **级别变动**: P0（待验证）→ 撤销（非缺陷）；**置信度由「低」升为「高」**（原先的低置信度仅因"读不到引擎源码"，现已逐行核实引擎实现）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Property/StructProperty/FStructPropertyDescriptor.cpp:41`
- **函数**: `FStructPropertyDescriptor::Set(void* Src, void* Dest) const`
- **置信度**: **高**（引擎侧 `FStructProperty::CopyValuesInternal` / `UScriptStruct::CopyScriptStruct` 已逐行核实；`web_search` 不可用不再影响本结论）

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Private/Reflection/Property/StructProperty/FStructPropertyDescriptor.cpp:33-43
void FStructPropertyDescriptor::Set(void* Src, void* Dest) const
{
	const auto SrcManagedHandle = *static_cast<IManagedHandle*>(Src);                        // 35
	if (const auto SrcStruct = FCSharpEnvironment::GetEnvironment().GetStruct<>(SrcManagedHandle))  // 37
	{
		Property->InitializeValue(Dest);                                                      // 39
		Property->CopySingleValue(Dest, SrcStruct);                                            // 41 ← 关键
	}
}
```
同一文件 `Identical` 用 `Property->Identical(...)`（`:52`）；`TCompoundPropertyDescriptor::CopyValue` 也用 `CopySingleValue`（`TCompoundPropertyDescriptor.inl:45`）。

**调用上下文**
`Set` 的 `Dest` 是**已构造的原生内存**，证据：
- `Macro/FunctionMacro.h:72` → `PropertyDescriptor->Set(IN_BUFFER, PropertyDescriptor->ContainerPtrToValuePtr<void>(Params))`，而 `Params` 在 `:69`（`INITIALIZE_VALUE()`）已被 `InitializeValue_InContainer` 初始化；
- `Private/Reflection/Function/FManagedFunctionDescriptor.cpp:56,61,82` → 写回 `InReturnAddress` / `OutAddress`（UFunction 返回值/out 参数的真实内存）；
- `Private/Reflection/Container/FArrayHelper.cpp:137` → `InnerPropertyDescriptor->Set(InValue, ScriptArrayHelper.GetRawPtr(Index))`（`IsValidIndex` 已确认，即**现存元素**）；
- `FMapHelper.cpp:200,218`、`FSetHelper.cpp:102` 同理。

**问题**
任务书假设的"结构体 `memcpy` = 双重释放"在**插件源码层面不成立**：`grep "FMemory::Memcpy\|memcpy"` 在 `Source/UnrealCSharp/**/Reflection/Property/` 下 **0 命中**（仅 `UnrealCSharpCore` 的 domain 层有 4 处 `FMemory::Memcpy`，与本目录无关）。但 `CopySingleValue` 是否安全，取决于 UE 侧实现：

- **假设 A（安全）**：`FStructProperty` 重写了 `CopySingleValue` 并转调 `UScriptStruct::CopyScriptStruct` → 深拷贝，无问题。
- **假设 B（危险）**：`FStructProperty` 未重写，落到 `FProperty::CopySingleValue` 的 `FMemory::Memcpy(Dest, Src, GetElementSize())` → 对含 `FString`/`TArray`/`TMap` 的结构体，`Dest` 与 `Src` 的内部堆指针变成**同一份**，两边析构时 **double free（P0 堆损坏）**。

**复核定案（撤销）**：UE 5.6 的 `FStructProperty` 重写了 `CopyValuesInternal`（`UnrealType.h:5970`）并转调 `UScriptStruct::CopyScriptStruct`（`PropertyStruct.cpp:337-340`），即**深拷贝**，Memcpy 兜底分支不可达。因此**假设 A 成立、假设 B 被证伪**，本条**不构成缺陷，应从排期移除**；下文的"建议/验证方式"仅作历史记录保留。

**建议**
不要依赖引擎默认行为，改为显式深拷贝：
```cpp
Property->InitializeValue(Dest);
Property->Struct->CopyScriptStruct(Dest, SrcStruct, 1);   // 明确走脚本结构体深拷贝
```
**验证方式**
1. ~~打开 `Engine/Source/Runtime/CoreUObject/Public/UObject/UnrealType.h`，搜 `class FStructProperty` 下的 `CopySingleValue` / `CopyValuesInternal`，看是否重写~~ → **已执行**：`FStructProperty` 覆盖了 `CopyValuesInternal`（`UnrealType.h:5970`），实现为 `Struct->CopyScriptStruct`（`PropertyStruct.cpp:337-340`）→ 本条据此撤销；
2. 造用例：一个含 `FString` 成员的结构体（如 `FTableRowBase` 派生）成员属性，从 C# 赋值两次，用 `-stompmalloc` 或 ASan 观察 double free；
3. 加临时断言 `check(Property->Struct->GetCppStructOps() == nullptr || !Property->Struct->GetCppStructOps()->HasDestructor())` 验证假设。

---

### P1

#### [F-PROP2-002] 7 个描述符的 `Set` 对已初始化内存调用 `InitializeValue` 而不先 `DestroyValue`，非平凡类型必定泄漏

- **类别**: 内存/资源泄漏
- **严重度**: **P1**
- **复核结论**: 部分确认（机制、类别、严重度成立；**两处口径偏差**：① 范围 —— 本条列举的是本报告 16 个目标文件内的 **7 个描述符**，而全插件同族实测为 **21 处 / 20 个描述符**，本条是范围**子集**；② 触发 —— "每一次对 `TArray<FString>` 元素的 C# 赋值都泄漏一次"这一说法偏大，`Add` 类路径的 `Dest` 是**真正的未初始化内存**，那里的 `InitializeValue` 不可删，详见下文"问题"节的分点判定表）
- **可达性**: 活跃（`FArrayHelper::Set:137`、`FRegisterProperty.cpp:35,64`、`FOptionalHelper.cpp:91` 等活值路径在五平台 LeanCLR 配置下都会执行）
- **复核证据**: 插件侧 7 处逐行核实（`FStrPropertyDescriptor.cpp:35`、`FNamePropertyDescriptor.cpp:35`、`FTextPropertyDescriptor.cpp:35`、`FAnsiStrPropertyDescriptor.cpp:36`、`FUtf8StrPropertyDescriptor.cpp:36`、`FStructPropertyDescriptor.cpp:39`、`FOptionalPropertyDescriptor.cpp:40`）；**引擎契约**：`Runtime/CoreUObject/Public/UObject/UnrealType.h:1044-1050` 对 `InitializeValue` 的文档直写 "**The existing data is assumed invalid**"，与 `:963` 的 `DestroyValue` "**The existing data is assumed valid**" 恰好互为反面 —— 在活值上调用 `InitializeValue` 即为契约违约；`FOptionalProperty::InitializeValueInternal`（`Private/UObject/PropertyOptional.cpp:435-445`）把 `IsSet` 直接置回 `false` 而**不销毁**旧值（对照会销毁的 `MarkUnset`：`PropertyOptional.h:58-74`）→ Optional 分支的泄漏机制在引擎侧得到确认
- **级别变动**: 无（维持 P1）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Property/StringProperty/FStrPropertyDescriptor.cpp:35`、`FNamePropertyDescriptor.cpp:35`、`FTextPropertyDescriptor.cpp:35`、`FAnsiStrPropertyDescriptor.cpp:36`、`FUtf8StrPropertyDescriptor.cpp:36`、`StructProperty/FStructPropertyDescriptor.cpp:39`、`OptionalProperty/FOptionalPropertyDescriptor.cpp:40`
- **函数**: 各 `::Set(void* Src, void* Dest) const`
- **置信度**: 中高（假设见下；插件的**自身用法**已佐证 `InitializeValue` 的语义）

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Private/Reflection/Property/StringProperty/FStrPropertyDescriptor.cpp:29-39
void FStrPropertyDescriptor::Set(void* Src, void* Dest) const
{
	const auto SrcManagedHandle = *static_cast<IManagedHandle*>(Src);                    // 31
	if (const auto SrcValue = FCSharpEnvironment::GetEnvironment().GetString<FString>(SrcManagedHandle))  // 33
	{
		Property->InitializeValue(Dest);                                                 // 35 ★对已初始化内存
		Property->SetPropertyValue(Dest, *SrcValue);                                     // 37
	}
}
```
`FNamePropertyDescriptor.cpp:35`、`FTextPropertyDescriptor.cpp:35`、`FAnsiStrPropertyDescriptor.cpp:36`、`FUtf8StrPropertyDescriptor.cpp:36` 完全一致；`FStructPropertyDescriptor.cpp:39`、`FOptionalPropertyDescriptor.cpp:40` 同型（后者配 `CopyCompleteValue`）。

**调用上下文**
`Dest` 是活的已初始化内存（证据见 `F-PROP2-001` 的"调用上下文"）。最关键的一条：
```cpp
// Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:133-139
void FArrayHelper::Set(const int32 Index, void* InValue) const
{
	if (auto ScriptArrayHelper = CreateHelperFormInnerProperty(); ScriptArrayHelper.IsValidIndex(Index))
	{
		InnerPropertyDescriptor->Set(InValue, ScriptArrayHelper.GetRawPtr(Index));   // 137 ★现存元素
	}
}
```
即 C# 侧对 `TArray<FString>` 的某个下标赋值 → 该槽位**已含一个真实的 FString**。

**问题**
`InitializeValue` 的契约是"作用于**未初始化**内存"，这一点由本插件自己的两处用法反证：
- `TCompoundPropertyDescriptor.inl:43-45`：`FMemory::Malloc` 之后才 `InitializeValue`；
- `FOptionalHelper.cpp:21-23`：`FMemory::Malloc(size, align)` 之后才 `InitializeValueInternal`。

在**已初始化**的 `FString`/`FText`/`FStruct`/`TOptional` 上再 `InitializeValue`，等价于对活的非平凡对象做 placement-new，其内部堆分配（`FString` 的 `TCHAR*`、`FText` 的 `TSharedRef<ITextData>` 引用计数、结构体成员的容器）**没有再被析构**→ 永久泄漏。触发路径**必须按 `Dest` 是否持有活值分类**（这是本族的真正判据，复核逐点以引擎实现确认）：

| 调用点 | `Dest` 来源 | `Dest` 是否活值 | `InitializeValue` 的后果 |
|---|---|---|---|
| `FArrayHelper.cpp:137`（`Set(Index,…)`） | `GetRawPtr(Index)` = **现存元素** | **活值** | ❌ 违约 → 旧 `FString` 内部数组泄漏 |
| `FRegisterProperty.cpp:35` / `:64` | `ContainerPtrToValuePtr<void>(FoundAddress)` = **UObject/UScriptStruct 现有属性内存** | **活值** | ❌ 违约 → 泄漏 |
| `FOptionalHelper.cpp:91` | `Data`，紧接 `MarkSetAndGetInitializedValuePointerToReplace` 之后 | **已置位时是活值** | ❌ 已置位时违约 → 泄漏；未置位时该 API 已先调用过 `InitializeValue`，此处是重复构造（`FString` 默认构造不分配，故无实际泄漏） |
| `FMapHelper.cpp:218` | 新键分支 `AddUninitialized`／已存在分支先经 `:215 DestroyValue` | 未初始化 **或** 已销毁 | ✅ **正确**（引擎语义：`DestroyValue` 之后本就应重新 `InitializeValue`） |
| `FArrayHelper.cpp:269`（`Add`） | `ScriptArrayHelper.AddUninitializedValue()` | **未初始化** | ✅ **必需且正确**（`FScriptArrayHelper::AddUninitializedValues` 只做 `Array->Add`、不构造元素：`UnrealType.h:4015-4021`） |
| `FMapHelper.cpp:200`（新键） | `ScriptMap->AddUninitialized(...)` | **未初始化** | ✅ **必需且正确** |
| `FSetHelper.cpp:102`（新元素） | `ScriptSet->AddUninitialized(...)` | **未初始化** | ✅ **必需且正确** |

因此"**每一次对 `TArray<FString>`/`TMap<_, FString>` 元素的 C# 赋值都泄漏一次**"的口径**偏大**：真正泄漏的是**写入活值**的路径（`FArrayHelper.cpp:137`、`FRegisterProperty.cpp:35,64`、`FOptionalHelper.cpp:91` 的已置位分支，及本报告范围外的同类点）；`Add` 类路径（`:269`、`FMapHelper.cpp:200`、`FSetHelper.cpp:102`）的元素内存是真的未初始化，那里的 `InitializeValue` **恰是必要步骤**。

> **跨报告审计的权威修复清单（供排期用）**：`FRegisterProperty.cpp:35`、`FRegisterProperty.cpp:64`、`FArrayHelper.cpp:137`、`FArrayHelper.cpp:269`、`FMapHelper.cpp:200`、`FMapHelper.cpp:218`、`FSetHelper.cpp:102`、`FOptionalHelper.cpp:91`（均已逐行核实存在）。**注意**：按上表的分点判定，其中 `FArrayHelper.cpp:269`、`FMapHelper.cpp:200`、`FSetHelper.cpp:102` 当前 `Dest` 为未初始化内存，属"改则坏"的点 —— 若修复方案是"统一加 `DestroyValue`"，这三点必须排除在外。

**插件内部的正确写法就在同一个目录**，说明这是遗漏而非有意：
```cpp
// Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp:214-219
if (!(InnerPropertyDescriptor->GetPropertyFlags() & (CPF_IsPlainOldData | CPF_NoDestructor)))
{
	for (auto Index = 0; Index < InCount; ++Index, Dest += InnerPropertyDescriptor->GetElementSize())
	{
		InnerPropertyDescriptor->DestroyValue(Dest);      // ★先析构
	}
}
```
`FMapHelper.cpp:119,121` 亦同。而 `FPropertyDescriptor::DestroyValue`（`FPropertyDescriptor.cpp:190-196`）/`DestroyValue_InContainer` 这些 API **在 7 个 `Set` 中一次都没被调用**。

**建议**
```cpp
void FStrPropertyDescriptor::Set(void* Src, void* Dest) const
{
	const auto SrcManagedHandle = *static_cast<IManagedHandle*>(Src);
	if (const auto SrcValue = FCSharpEnvironment::GetEnvironment().GetString<FString>(SrcManagedHandle))
	{
		// FString/Struct/Optional 均为非平凡类型：先析构再赋值
		Property->DestroyValue(Dest);        // 或按 CPF_IsPlainOldData|CPF_NoDestructor 判断后跳过
		Property->InitializeValue(Dest);
		Property->SetPropertyValue(Dest, *SrcValue);
	}
}
```
取舍（**复核修正**）：**不能整对删掉 `InitializeValue`**。同一个 `Set` 被两类调用点共用 —— `Dest` 是活值时它必须先 `DestroyValue`；`Dest` 是未初始化内存时（`FArrayHelper::Add`、`FMapHelper`/`FSetHelper` 的新键/新元素分支）它**必须**保留 `InitializeValue`，否则元素内存带着垃圾被 `SetPropertyValue` 当 `FString` 赋值 → 崩溃。可行修法（按优先级）：
1. **最小且安全**：不动 `Set`，让**活值调用点**自己先析构 —— 即 `FArrayHelper::Set:137`、`FRegisterProperty.cpp:35,64`、`FOptionalHelper::Set:91` 改为 `Property->DestroyValue(Dest);` 后再调 `descriptor->Set(...)`；
2. **语义最清晰**：给描述符拆一对 API —— `Set`（活值语义：内部 `DestroyValue` + `InitializeValue` + 赋值）与 `SetUninitialized`（仅 `InitializeValue` + 赋值），由各调用点显式选择；
3. **不要**一刀切：全删 `InitializeValue` 会破坏 `Add` 路径，无差别加 `DestroyValue` 会对未初始化内存调析构（同样崩溃）。

（"最省事的修法是整对删掉 `InitializeValue`"只适用于 `FArrayHelper::Set:137` 这类活值路径，**不适用于 `Add`**。）

**验证方式**
- `grep -n "InitializeValue" Source/UnrealCSharp/Private/Reflection/Property/**/*.cpp` 确认 7 处；
- 用例：C# 侧对同一个 `TArray<FString>` 元素连续赋值 10 万次，观察内存增长（`stat memory` / 任务管理器）；
- 在 `Set` 加 `check(Property->HasAnyPropertyFlags(CPF_IsPlainOldData | CPF_NoDestructor))` 反证该路径确实处理非平凡类型。

#### [F-PROP2-003] `Get<FReturn>` 的堆缓冲只 `FMemory::Free`、从不 `DestroyValue` → 字符串内容整体泄漏

- **类别**: 内存/资源泄漏
- **严重度**: **P1**
- **复核结论**: 确认
- **可达性**: 活跃（`FunctionMacro.h:143-145` 的 `PROCESS_RETURN` Compound 分支 → `CopyValue` 缓冲 → `AddStringReference<…,true,false>`；五平台 LeanCLR 配置下同样执行）
- **复核证据**: 三段链路逐行核实 —— 分配 `CopyValue`（`TCompoundPropertyDescriptor.inl:33-48`：`FMemory::Malloc` + `InitializeValue` + `CopySingleValue`）；移交 `FStrPropertyDescriptor.cpp:19-27`（`:23` `AddStringReference<FString, true, false>`）；释放 `FStringRegistry.inl:67-72`（`:69` `FMemory::Free(FoundValue->Value)`，**该文件全文无 `DestroyValue`/`~FString()`**）。`FText`/`FAnsiString`/`FUtf8String` 同型（`Get(FReturn)`：`FTextPropertyDescriptor.cpp:19-27`、`FAnsiStrPropertyDescriptor.cpp:20-28`、`FUtf8StrPropertyDescriptor.cpp:20-28`）；`FNamePropertyDescriptor.cpp:19-27` 因 `FName` 无堆成员而不泄漏 —— 同一份模板代码因 `T` 不同而安全性不同，此点判断正确
- **级别变动**: 无（维持 P1）
- **文件**: `Source/UnrealCSharp/Public/Registry/FStringRegistry.inl:67-72`（释放点）+ `Source/UnrealCSharp/Private/Reflection/Property/StringProperty/{FStr,FText,FAnsiStr,FUtf8Str}PropertyDescriptor.cpp:19-27`（赋值 `IsNeedFree=true` 的 4 处）
- **函数**: `FStringRegistry::TStringRegistryImplementation::RemoveReference(Class*, IManagedHandle)`
- **置信度**: 高（分配点、移交点、释放点三段链路全部有行号）

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Public/Registry/FStringRegistry.inl:67-72
			if (FoundValue->bNeedFree)
			{
				FMemory::Free(FoundValue->Value);      // 69 ★只释放 FString 本体，不调用 ~FString()

				FoundValue->Value = nullptr;
			}
```
缓冲的来源：
```cpp
// Source/UnrealCSharp/Public/Reflection/Property/TCompoundPropertyDescriptor.inl:33-48
	virtual auto CopyValue(const void* InAddress) const -> void* override
	{
		const auto Value = static_cast<void*>(static_cast<uint8*>(FMemory::Malloc(
			Super::Property->GetElementSize())));          // 35-41
		Super::Property->InitializeValue(Value);           // 43
		Super::Property->CopySingleValue(Value, InAddress); // 45 ★深拷贝：内部字符数组在此分配
		return Value;
	}
```
移交点（4 个 String 描述符的 `FReturn` 分支，`FStrPropertyDescriptor.cpp:19-27`）：
```cpp
	const auto Object = Class->NewObject();                                        // 21
	FCSharpEnvironment::GetEnvironment().AddStringReference<FString, true, false>(  // 23 ★IsNeedFree=true
		Class, Object, Src);
```

**调用上下文**
`Src` 的实参来自宏：
```cpp
// Source/UnrealCSharp/Public/Macro/FunctionMacro.h:143-145（PROCESS_RETURN）、:128-130（PROCESS_OUT）
	ReturnPropertyDescriptor->Get<FPropertyArgument::FReturn>(
		ReturnPropertyDescriptor->CopyValue(ReturnPropertyDescriptor->ContainerPtrToValuePtr<void>(Params)),  // ★Malloc 缓冲
		reinterpret_cast<void**>(RETURN_BUFFER));
```
释放时机：托管侧 `FString.UnRegister` → `FRegisterString.cpp:34-40` `AsyncTask(GameThread, RemoveStringReference<FString>)` → 上面的 `FMemory::Free`。

**问题**
`CopyValue` 在 `:45` 做的 `CopySingleValue` 为 `FString` 分配了独立的字符数组；`RemoveReference` 在 `:69` 只 `FMemory::Free` 掉 **`FString` 结构体本身**（24 字节），**从不执行 `~FString()`**，因此那串字符数组的分配**永久泄漏**。影响范围：
- `FStrPropertyDescriptor`、`FTextPropertyDescriptor`、`FAnsiStrPropertyDescriptor`、`FUtf8StrPropertyDescriptor` 的 `FReturn` 路径 → **泄漏**（`FText` 还额外泄漏一份 `TSharedRef<ITextData>` 引用计数，即便共享也可能导致文本历史/本地化缓存不释放）；
- `FNamePropertyDescriptor` 的 `FReturn` 路径 → **不泄漏**（`FName` 内部只有一个 `FNameEntryId`，无独立堆分配）。同样一份模板代码，安全性随 `T` 而变，这正是这种"5 份逐字拷贝"结构的危险之处。

插件里明明有配套 API 却没用：`FPropertyDescriptor::DestroyValue`（`FPropertyDescriptor.cpp:190-196`）→ `Property->DestroyValue(Dest)`。

**建议**
`FStringRegistry` 需要知道"如何析构这个 `T`"，最小改动是给 `TStringAddress` 增加析构回调：
```cpp
template <typename T>
struct TStringAddress : TValueWrapper<T>
{
	TStringAddress(T InValue, const bool InNeedFree, void (*InDestructor)(T) = nullptr)
		: TValueWrapper<T>(InValue), bNeedFree(InNeedFree), Destructor(InDestructor) {}
	bool bNeedFree;
	void (*Destructor)(T);
};
// RemoveReference 中：
if (FoundValue->bNeedFree)
{
	if (FoundValue->Destructor) { FoundValue->Destructor(FoundValue->Value); }  // ~FString()
	FMemory::Free(FoundValue->Value);
}
```
（`FStringRegistry.inl:47-50` 的 `AddReference` 传入 `+[](FString* P){ P->~FString(); }` 之类的 lambda。）

**验证方式**
- 分别对 `FString`/`FName`/`FText` 做 `CopyValue`→`AddStringReference<...,true,false>`→`RemoveReference` 的 10 万次循环，比对 `FPlatformMemory::GetStats().UsedPhysical`；预期仅 `FString`/`FText` 线性增长；
- `grep -n "DestroyValue" Source/UnrealCSharp/Public/Registry/FStringRegistry.inl` → 0 命中，即为证据。

### P0

#### [F-PROP2-004] 6 个 `Identical` 覆写不检查句柄有效性 → 空指针解引用

- **类别**: Bug（空指针）
- **严重度**: **P0**
- **复核结论**: 确认（机制与调用链成立；**触发前提**是 C# 侧传入失效/已 `Dispose`/类型不符的句柄）
- **可达性**: 活跃（`grep "Descriptor->Identical\("` 实测 **11** 个调用点，全部在容器元素比较路径：`FArrayHelper.cpp:148,164,180,293`、`FSetHelper.cpp:87,124,154`、`FMapHelper.cpp:101,137,164,183`；`B` 实参即 C# 传入的 `IManagedHandle`）
- **复核证据**: `FStringRegistry.inl:25-27` 显式返回 `nullptr`（未命中即空）；6 处覆写无守卫（`FStr/FName/FText PropertyDescriptor.cpp:45-48`、`FAnsi/FUtf8Str…cpp:46-49`、`FStructPropertyDescriptor.cpp:49-52`）；同文件 `Set` 有守卫作对照（`FStrPropertyDescriptor.cpp:33`、`FStructPropertyDescriptor.cpp:37`）
- **级别变动**: 无（由 P1 校正为 P0 成立：托管边界上的**无条件空指针解引用 → 崩溃**，与同前缀报告 `01-…/05` 的 `F-PROP2-002`（`Set` 不查 `GetMulti` 即解引用，P0）采用同一定级口径）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Property/StringProperty/FStrPropertyDescriptor.cpp:45-48`、`FNamePropertyDescriptor.cpp:45-48`、`FTextPropertyDescriptor.cpp:45-48`、`FAnsiStrPropertyDescriptor.cpp:46-49`、`FUtf8StrPropertyDescriptor.cpp:46-49`、`StructProperty/FStructPropertyDescriptor.cpp:49-52`
- **函数**: 各 `::Identical(const void* A, const void* B, uint32 PortFlags) const`
- **置信度**: 高（"同文件 `Set` 查空、`Identical` 不查空"这一不一致是可直接对照的代码事实）

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Private/Reflection/Property/StringProperty/FStrPropertyDescriptor.cpp:41-49
bool FStrPropertyDescriptor::Identical(const void* A, const void* B, const uint32 PortFlags) const
{
	const auto StringA = Property->GetPropertyValue(A);                                   // 43

	const auto StringB = FCSharpEnvironment::GetEnvironment().GetString<FString>(         // 45 ← 可能为 nullptr
		*static_cast<IManagedHandle*>(const_cast<void*>(B)));

	return StringA == *StringB;                                                            // 48 ★未查空即解引用
}
```
```cpp
// Source/UnrealCSharp/Private/Reflection/Property/StructProperty/FStructPropertyDescriptor.cpp:45-53
	const auto StructA = Property->ContainerPtrToValuePtr<void>(A);                        // 47
	const auto StructB = FCSharpEnvironment::GetEnvironment().GetStruct<>(                 // 49 ← 可能为 nullptr
		*static_cast<IManagedHandle*>(const_cast<void*>(B)));
	return Property->Identical(StructA, StructB, PortFlags);                               // 52 ★nullptr 传给原生比较
```
`FStringRegistry.inl:25-27` 明确会返回 `nullptr`：
```cpp
	return FoundValue != nullptr
		       ? static_cast<typename FStringValueMapping::ValueType::Type>(FoundValue->Value)
		       : nullptr;                                    // 27
```
**对照**：**同一个文件**的 `Set` 做了查空（`FStrPropertyDescriptor.cpp:33` `if (const auto SrcValue = ...)`；`FStructPropertyDescriptor.cpp:37` 同）。

**调用上下文**
`Identical` 被容器元素比较大量调用（`grep "Descriptor->Identical\("` = 11 命中）：
`FArrayHelper.cpp:148,164,180,293`（`Find`/`Contains`/`IndexOf`/`Remove`）、`FSetHelper.cpp:87,124,154`、`FMapHelper.cpp:101,137,164,183`。
`B` 实参是 C# 传下来的 `IManagedHandle`（`FArrayHelper.cpp:148` 的 `InValue`）——即**由托管侧输入**，可以是失效/已 Dispose/类型不符的句柄。

**问题**
一旦 `GetString<T>`/`GetStruct<>` 返回 `nullptr`：字符串族在 `:48` 对空指针解引用 `FString`（读前 8 字节 → 访问违例崩溃）；结构体族把 `nullptr` 传进 `UScriptStruct::CompareScriptStruct`，逐属性从空指针读 → 同样崩溃。这是"用 C# 侧输入触发的原生崩溃"，属于插件边界处最不该有的失败模式。

另外这条不一致在插件里是**系统性**的——托管边界函数也不查空：
- `FRegisterString.cpp:44-46`：`const auto String = GetString<FString>(InManagedHandle);` 紧接着 `TCHAR_TO_UTF8(**String)`；
- `FRegisterName.cpp:44-46`、`FRegisterText.cpp:70-72`：同型（`Text->ToString()`）。
（这三个文件不在我的分析范围内，仅作交叉引用。）

**建议**
统一加守卫，且**把失败视为"不相等"而不是崩溃**：
```cpp
	const auto StringB = FCSharpEnvironment::GetEnvironment().GetString<FString>(
		*static_cast<IManagedHandle*>(const_cast<void*>(B)));
	if (StringB == nullptr)
	{
		return false;      // 与 FRegisterString::IdenticalImplementation:31 的 return 0 保持一致
	}
	return StringA == *StringB;
```
结构体族同理，`StructB == nullptr` 时 `return false`。

**验证方式**
- 单元用例：C# 侧先取一个 `FString` 托管对象，`Dispose`/`UnRegister` 后把它传给 `TArray<FString>.Contains(...)` → 现状崩溃；
- `grep -n "GetString<" Source/UnrealCSharp/Private/Reflection/Property/StringProperty/*.cpp` 逐条核对是否有相邻的 `if`。

#### [F-PROP2-005] `FOptionalPropertyDescriptor::Set` 未检查 `GetOptional` 返回值即解引用

- **类别**: Bug（空指针）
- **严重度**: **P0**
- **复核结论**: 确认（机制与调用链成立；触发前提同 `F-PROP2-004`：C# 侧传入失效/已 `Dispose` 的 Optional 句柄）
- **可达性**: 活跃（`FOptionalPropertyDescriptor::Set` 是 C# 写 `TOptional<T>` 属性的唯一入口，经 `FunctionMacro.h:72` 的 `IN_VALUE()` 或 `FManagedFunctionDescriptor.cpp:56,61,82` 到达）
- **复核证据**: `FOptionalPropertyDescriptor.cpp:36-42`（`:38` 取出后**无任何判空**，`:42` 直接 `SrcOptional->GetData()`）；`FOptionalRegistry.cpp:41-46` 未命中返回 `nullptr`（`:45`）；同文件 `Get` 路径却查空（`:9`），`FStructPropertyDescriptor::Set:37` 亦查空 —— 三处对照见上文
- **级别变动**: **P1 → P0**（理由一句话：与 `F-PROP2-004` 属同一失效模式 —— 托管边界传入失效句柄即触发**无条件空指针解引用崩溃**，同轮复核中 004 已定 P0，同一机制不应低一档；证据行 `FOptionalPropertyDescriptor.cpp:38,42`）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Property/OptionalProperty/FOptionalPropertyDescriptor.cpp:38,42`
- **函数**: `FOptionalPropertyDescriptor::Set(void* Src, void* Dest) const`
- **置信度**: 高

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Private/Reflection/Property/OptionalProperty/FOptionalPropertyDescriptor.cpp:34-43
void FOptionalPropertyDescriptor::Set(void* Src, void* Dest) const
{
	const auto SrcManagedHandle = *static_cast<IManagedHandle*>(Src);                          // 36

	const auto SrcOptional = FCSharpEnvironment::GetEnvironment().GetOptional(SrcManagedHandle); // 38 ← 可能 nullptr

	Property->InitializeValue(Dest);                                                            // 40
	Property->CopyCompleteValue(Dest, SrcOptional->GetData());                                  // 42 ★nullptr 解引用
}
```
`FCSharpEnvironment::GetOptional` → `FOptionalRegistry::GetOptional`（`FOptionalRegistry.cpp:41-46`）在未命中时返回 `nullptr`。

**调用上下文**
与 `FStructPropertyDescriptor::Set`（`.cpp:37` 有 `if (const auto SrcStruct = ...)`）、以及**同一文件**的 `Get`（`.cpp:9` `if (!IManagedHandleIsValid(Object))`）形成三处对照：只有这一处不查空。调用方为 `Macro/FunctionMacro.h:72` / `FManagedFunctionDescriptor.cpp:56,61,82`。

**问题**
传入一个失效的 `TOptional` 句柄（C# 侧已释放/未初始化/类型不符）时，`:42` 直接 `SrcOptional->GetData()` 即对空指针调用成员函数 → 崩溃。相比 `Get` 路径的谨慎（`:9` 查空），这是明显的遗漏。

**建议**
```cpp
	const auto SrcOptional = FCSharpEnvironment::GetEnvironment().GetOptional(SrcManagedHandle);
	if (SrcOptional == nullptr)
	{
		return;        // 或 ensure，或按"清空"处理：Property->MarkUnset(Dest)
	}
	Property->DestroyValue(Dest);   // 见 F-PROP2-002
	Property->InitializeValue(Dest);
	Property->CopyCompleteValue(Dest, SrcOptional->GetData());
```
注意取舍：静默 `return` 会让 C# 侧赋值"没生效"而不报错；建议至少 `ensure(SrcOptional != nullptr)` 以便在 Development 下暴露。

**验证方式**
C# 侧构造一个 `TOptional<T>` 托管对象，释放后赋给一个 optional 属性 → 现状崩溃。

### P2

#### [F-PROP2-006] `FText` 经 `ToString` 往返丢失 key / namespace（本地化信息）

- **类别**: Bug（静默数据错误）
- **严重度**: **P2**
- **复核结论**: 确认（`FText::ToString()` 只返回**当前文化下的显示串**，"显示串 → `new FText(string)`"必然丢失 key/namespace/`FTextHistory`，这条事实成立；注意**描述符本身是无损的**，丢失发生在 C# 侧的用户习惯往返路径上）
- **可达性**: 活跃（`FRegisterText.cpp:68-73` 已注册为 C# 的 `FText.ToString`；反向 `RegisterImplementation:16-45` 走 `FTextStringHelper::ReadFromBuffer`）
- **复核证据**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterText.cpp:68-73`（`:72` `NewString(TCHAR_TO_UTF8(*Text->ToString()))`）；反向构造 `:34-44`（`:36-41` `FTextStringHelper::ReadFromBuffer(InBuffer, *OutText, *TextNamespace, *PackageNamespace, bRequiresQuotes)`，只接受 `NSLOCTEXT`/`LOCTEXT` 形式的字面量缓冲，**无法从纯显示串恢复 key**）；`FTextPropertyDescriptor.cpp:6-16` 的 `Get(FMember)` 返回原生 `FText` 别名（`AddStringReference<FText,false,true>` at `:12`）→ 描述符无损
- **级别变动**: 无（由 P1 校正为 P2 成立：可观测后果是本地化身份静默丢失，不崩溃、不泄漏）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterText.cpp:68-73`（**相邻文件，非本次 16 个目标文件之一**，但因直接决定 `FTextPropertyDescriptor` 的语义而单列）
- **函数**: `FRegisterText::ToStringImplementation(IManagedHandle)`
- **置信度**: 中高

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Private/Domain/Interop/FRegisterText.cpp:68-73
		static IManagedHandle ToStringImplementation(const IManagedHandle InManagedHandle)
		{
			const auto Text = FCSharpEnvironment::GetEnvironment().GetString<FText>(InManagedHandle);

			return IScriptDomain::Get()->NewString(TCHAR_TO_UTF8(*Text->ToString()));   // 72 ★只取显示串
		}
```
反向构造（`:34-44`）用的是 `FTextStringHelper::ReadFromBuffer(*Buffer, *OutText, *TextNamespace, *PackageNamespace, bRequiresQuotes)`——**能**承载 namespace，但只能解析 `NSLOCTEXT`/`LOCTEXT` 形式的**字面量文本缓冲区**，无法从纯显示串恢复 key。

**调用上下文**
`FTextPropertyDescriptor::Get(FMember)`（`FTextPropertyDescriptor.cpp:6-16`）返回的是**原生 `FText` 的别名**（托管对象内部持 `FText*`），因此**描述符本身不丢失任何信息**——丢失发生在 C# 侧调 `FText.ToString()` 再 `new FText(string)` 这条**用户习惯路径**上。`PRAGMA_DISABLE_DANGLING_WARNINGS`（`FRegisterText.cpp:10`、`:88`）表明作者已意识到此处有生命周期隐患。

**问题**
`FText::ToString()` 返回的是**当前文化下的显示字符串**。用它再构造 `FText`，得到的是一个 `bIsCultureInvariant` 的字面量文本：**丢失 `ITextData` 的 key、namespace、历史（`FTextHistory`）与本地化身份**。表现：
- 编辑器里已本地化的属性，经 C# 读写一轮后变成不可本地化的硬编码字符串；
- 换语言后该文本不再跟随本地化；
- key 级比较的语义已被降级：`FTextPropertyDescriptor.cpp:48` 用的是 `EqualTo`，其默认比较级别为 `ETextComparisonLevel::Default`（引擎 `Text.h:570`；枚举成员见 `TextComparison.h:8-19`，**并不存在 `DisplayString` 这一级**），比较对象是**显示串**而非 key/identity，因此 key 不同而显示串相同的两段文本会被判为相等（结论方向不变）。

**建议**
在 C# 绑定层避免 `ToString`/`FromString` 往返：直接保留 `FText` 别名（现状 `Get(FMember)` 已经是别名，**不建议改动**），并在 C# 侧把 `FText` 当不透明句柄传递；若必须序列化，改用 `FTextStringHelper::WriteToBuffer`（保留 namespace/key）而不是 `ToString`。
```cpp
// 建议：提供 ToBuffer/FromBuffer 对，而不是 ToString
static IManagedHandle ToBufferImplementation(const IManagedHandle InManagedHandle)
{
	const auto Text = FCSharpEnvironment::GetEnvironment().GetString<FText>(InManagedHandle);
	if (Text == nullptr) { return InvalidManagedHandle; }        // 见 F-PROP2-004
	FString OutBuffer;
	FTextStringHelper::WriteToBuffer(OutBuffer, *Text);
	return IScriptDomain::Get()->NewString(TCHAR_TO_UTF8(*OutBuffer));
}
```
**验证方式**
建一个带 `NSLOCTEXT("MyNs","MyKey","Default")` 的属性，C# 侧 `Get` → `ToString` → `new FText(...)` → `Set`，然后在编辑器里查该 `FText` 是否仍有 key（`FText::GetTextHistory()` / 本地化面板），预期变为 `AsLiteral`。

#### [F-PROP2-007] 以裸地址为键的别名缓存永不失效 → GC 后地址复用导致串对象 / 悬垂引用

- **类别**: 未定义行为 | 内存/资源泄漏
- **严重度**: **P2**
- **复核结论**: 确认（两条子机制均成立：① 地址键缓存**从不失效**；② 托管对象内部持原生 `FString*`，生命周期与原生对象解耦）
- **可达性**: 活跃（`Get(FMember)` 每次先查该表：`FStrPropertyDescriptor.cpp:6`、`FStructPropertyDescriptor.cpp:57`、`FOptionalPropertyDescriptor.cpp:7`）
- **复核证据**: `FStringRegistry.inl:30-36`（`Address2ManagedHandle.Find(InAddress)`，键=裸 `void*`）、`:42-45`（仅 `IsMember` 才写缓存）、`:55-82`（清理只发生在显式 `RemoveReference`）；`FStringRegistry.h:24`（托管侧持 `FString*`）、`:81,85,90,96,101`；**"是否存在销毁钩子"已查实**：`grep Address2ManagedHandle` 全模块 122 命中，其中 `.Remove(` 仅 7 处（`FStringRegistry.inl:63`、`FContainerRegistry.inl:68`、`FDelegateRegistry.inl:67`、`FMultiRegistry.inl:62`、`FOptionalRegistry.cpp:65`、`FStructRegistry.cpp:121`、`FBindingRegistry.cpp:56`）**全部位于各注册表自己的 `RemoveReference` 内**，其余命中均为声明/模板参数或 `Deinitialize` 里的 `Empty()` → **确认不存在任何 UObject GC/销毁钩子会清理这些表**
- **级别变动**: 无（维持 P2：需"原生对象被 GC"且"地址被后续分配复用"两者同时成立才可观测，属隐患；置信度由「中」升为**高**）
- **文件**: `Source/UnrealCSharp/Public/Registry/FStringRegistry.inl:30-53`（`GetObject`/`AddReference`）、`FStringRegistry.h:81,85,90,96,101`（`Address2ManagedHandle` 成员）、`FStructRegistry.inl:19-20`、`FOptionalRegistry.inl:11-14`
- **函数**: `FStringRegistry::TStringRegistryImplementation::GetObject/addReference`
- **置信度**: 中（未验证是否存在外部对象销毁钩子清理这些表；`Initialize`/`Deinitialize` 之外我未找到清理点）

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Public/Registry/FStringRegistry.inl:30-53
	static auto GetObject(Class* InRegistry, typename FStringValueMapping::FAddressType InAddress) -> IManagedHandle
	{
		const auto FoundManagedHandle = (InRegistry->*Address2ManagedHandle).Find(InAddress);  // 33 ★键=裸 void*
		return FoundManagedHandle != nullptr ? *FoundManagedHandle : InvalidManagedHandle;
	}
	...
		if constexpr (IsMember) { (InRegistry->*Address2ManagedHandle).Add(InAddress, InManagedHandle); }  // 44
		(InRegistry->*ManagedHandle2Value).Add(InManagedHandle,
			typename FStringValueMapping::ValueType(static_cast<...::Type>(InAddress), IsNeedFree));        // 47-50
```
清除只发生在**显式的 `RemoveReference`**（`FStringRegistry.inl:55-82`）或环境销毁时。`FStringRegistry::Initialize()`/`Deinitialize()`（`FStringRegistry.cpp`）不做逐对象清理。

**调用上下文**
`Get(FMember)` 每次都先用 `Src`（UObject 属性块内的地址）查这张表：`FStrPropertyDescriptor.cpp:6`；结构体走 `GetObject(Property->Struct, InAddress)`（`FStructPropertyDescriptor.cpp:57`）；optional 走 `GetOptionalObject`（`FOptionalPropertyDescriptor.cpp:7`）。

**问题**
两个独立缺陷：
1. **地址复用串对象**：`Address2ManagedHandle` 以 `void*` 为键且不在 UObject 被 GC 时移除。UObject 销毁后其属性块可能被后续分配复用；新对象在同一地址的 `FString` 会**命中旧句柄**，于是 C# 拿到的是*另一个对象*的字符串包装 → 读到错误的数据。
2. **悬垂引用**：托管对象内部保存的是原生 `FString*`（`FStringRegistry.h:24` `TStringAddress<FString*>`），生命周期完全独立于原生对象。UObject 销毁后 C# 仍可持有该对象并读取 → use-after-free。

`FStructRegistry` 用 `FStructAddressBase{UScriptStruct*, void* Address}` 作键（`FStructRegistry.h:5-8,25-34`），比字符串表多了 struct 类型维度，缓解但不消除问题（地址仍在复用）。`FOptionalRegistry` 与字符串表同构（`FOptionalRegistry.inl:11-14`）。

**建议**
- 短期：在 `FObjectRegistry`/`FCSharpEnvironment` 的对象销毁路径（GC 或 `RemoveReference`）上挂钩子，按地址批量清除 `Address2ManagedHandle` 中属于该对象的条目；
- 或改为**不缓存**：`Get(FMember)` 每次新建句柄（牺牲性能换正确性），缓存只保留在托管侧按句柄索引；
- 参考：`FStringRegistry.inl:59-65` 已有"只删自己那条"的自校验逻辑，说明作者考虑过一致性，但没考虑地址复用。

**验证方式**
遍历 `Address2ManagedHandle` 的键，与 `TObjectRange<UObject>` 的存活对象属性地址求交集，检查是否存在指向已销毁对象的条目；或写一个 GC 压力用例（生成/销毁大量含 `FString` 的 UObject）后读旧对象句柄。

#### [F-PROP2-008] `FStructPropertyDescriptor::Identical` 多算一次 `ContainerPtrToValuePtr` 偏移，与基类及同类方法不一致

- **类别**: Bug（潜在读错地址）
- **严重度**: **P2**
- **复核结论**: 部分确认（**偏差：当前无可观测错误**。`ContainerPtrToValuePtr` 确实会叠加 `Offset_Internal`（引擎证据见下），与 `Set`/`Get`/基类"直接用入参地址"的约定不一致这一点成立；但已核实该 `Identical` 的**全部 11 个调用点都传元素/键地址**，而容器内层属性在引擎惯例下以元素地址直传（`Offset_Internal` 为 0）→ 偏移为 0 时 `A + 0 == A`，结果与基类一致，故为**隐患**而非现网错误）
- **可达性**: 活跃（`Identical` 路径本身会执行；只是其多算的偏移当前恒为 0）
- **复核证据**: `Runtime/CoreUObject/Public/UObject/UnrealType.h:674-686`（`ContainerVoidPtrToValuePtrInternal` 返回 `(uint8*)ContainerPtr + Offset_Internal + ElementSize*ArrayIndex` —— **确认会加 `Offset_Internal`**）；插件侧 11 个调用点均为容器内层/键/值描述符（`FArrayHelper.cpp:148,164,180,293` 用 `InnerPropertyDescriptor`；`FSetHelper.cpp:87,124,154` 用 `ElementPropertyDescriptor`；`FMapHelper.cpp:101,137,164,183` 用 `Key/ValuePropertyDescriptor`）；对照基类 `FPropertyDescriptor.cpp:184` 与同文件 `Set:41`/`Get:14` 均不加偏移
- **级别变动**: 无（维持 P2：属"不一致 + 潜在读错地址"的隐患；若仓库内出现 `Inner` 偏移非 0 的场景，本条应升级）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Property/StructProperty/FStructPropertyDescriptor.cpp:47`
- **函数**: `FStructPropertyDescriptor::Identical(const void* A, const void* B, uint32 PortFlags) const`
- **置信度**: 中（`A` 的确切语义取决于调用方；容器内元素场景下偏移通常为 0，故未观察到故障）

**现状（代码事实）**
```cpp
// .../FStructPropertyDescriptor.cpp:45-53
	const auto StructA = Property->ContainerPtrToValuePtr<void>(A);     // 47 ★加了 Property 的 Offset_Internal
	const auto StructB = FCSharpEnvironment::GetEnvironment().GetStruct<>(...);
	return Property->Identical(StructA, StructB, PortFlags);            // 52
```
同类中其它方法都是**直接用 `A`/`Src`，不加偏移**：
- `Get(FMember)`：`AddStructReference<false>(Property->Struct, Src, Object)`（`:14`，`Src` 直接当数据地址）；
- `Set`：`Property->CopySingleValue(Dest, SrcStruct)`（`:41`，直接当数据地址）；
- 基类：`FPropertyDescriptor::Identical`（`FPropertyDescriptor.cpp:182-185`）`Property->Identical(A, B, PortFlags)`，**不加偏移**。

**调用上下文**
`Identical` 的调用方传的是**元素/数据地址**而非容器地址：`FArrayHelper.cpp:148,164,180,293` 传 `ScriptArrayHelper.GetRawPtr(Index)`（元素裸地址）；`FSetHelper.cpp:87,124,154`、`FMapHelper.cpp:101,137,164,183` 同理。而 `ContainerPtrToValuePtr` 的语义是"`ContainerPtr` + 本属性偏移"，与上述入参约定不符。

**问题**
当 `Property->Offset_Internal != 0` 时（`FStructProperty` 作为 `UScriptStruct` 的**成员**时其偏移就是成员偏移），`StructA` 会指向 `A + Offset`，而 `A` 已经是数据地址 → 从错误位置读取并比较，得到**静默错误结果**（`Contains`/`IndexOf`/`Remove` 判断错误）。与 `Set`/`Get`/基类的约定不一致说明这是笔误而非有意。
> 注：容器内层属性（`FArrayProperty` 的 `Inner`）偏移通常为 0，因此高频路径上可能观察不到，这让问题更隐蔽。

**建议**
```cpp
	const auto StructA = static_cast<const void*>(A);   // 与 Set/Get/基类一致，不加偏移
```
若确实存在"需要从容器地址取元素"的调用方，应改为由调用方负责转换，而不是让 `Identical` 单方面加偏移。

**验证方式**
构造 `FMyStruct { FVector Inner; FString Name; }` 并用 `TArray<FMyStruct>.Contains(x)`：若 `FStructProperty` 的 `Offset_Internal` 非 0（可用 `Property->GetOffset_ForInternal()` 打印），比较结果将错误。

#### [F-PROP2-009] `FName` 每次经字符串构造 + `ToString` 每次分配：跨托管边界的高频开销

- **类别**: 性能
- **严重度**: **P2**
- **复核结论**: 部分确认（**偏差：类别与收益口径**。往返确实退化为字符串、`FNamePool` 查找带锁这两条代码事实成立；但本条衡量的是"跨边界开销"而非缺陷，且关于"`FName(TEXT("Foo_1"))` 与 `FName(TEXT("Foo"), 1)` 语义"的旁证**未回引擎源码核实**，标为存疑）
- **可达性**: 活跃（`FRegisterName.cpp:15` 的 `RegisterImplementation` 与 `:46` 的 `ToStringImplementation` 已注册为 C# 侧 `FName.Register` / `FName.ToString`）
- **复核证据**: `FRegisterName.cpp:13-19`（`:15` `new FName(... FString(UTF8_TO_TCHAR(InValue)) ...)`）、`:42-47`（`:46` `NewString(TCHAR_TO_UTF8(*Name->ToString()))`）；`FNamePropertyDescriptor.cpp:6` 走 registry 缓存（故属性取值本身不触发该路径）、`:48` 的 `NameA == *NameB` 走 `FName` 索引比较（"Identical 已做对"成立）；同型：`FRegisterString.cpp:15,46`、`FRegisterText.cpp:36-44,72`
- **级别变动**: 无（维持 P2：性能/隐患类）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterName.cpp:15,46`（**相邻文件**，直接服务 `FNamePropertyDescriptor`）
- **函数**: `FRegisterName::RegisterImplementation`、`FRegisterName::ToStringImplementation`
- **置信度**: 中

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Private/Domain/Interop/FRegisterName.cpp:13-19
		static void RegisterImplementation(const IManagedHandle InManagedObject, const char* InValue)
		{
			const auto Name = new FName(InValue != nullptr ? FString(UTF8_TO_TCHAR(InValue)) : FString(TEXT("")));  // 15 ★
			...AddStringReference<FName, true, false>(...);
		}
// :42-47
			return IScriptDomain::Get()->NewString(TCHAR_TO_UTF8(*Name->ToString()));   // 46 ★ToString + 转换 + 分配
```
每次 C# 侧构造一个 `FName`（`:15`）都要走 `UTF8_TO_TCHAR`（分配临时 `FString`）→ `FName(FString)` → **全局名表（`FNamePool`）查找/插入，带临界区**；每次转字符串（`:46`）又要 `FName::ToString()` + `TCHAR_TO_UTF8` + `NewString` 三份工作。`FRegisterString.cpp:15,46`、`FRegisterText.cpp:36-44,72` 同型。

**调用上下文**
`FNamePropertyDescriptor::Get(FMember)`（`FNamePropertyDescriptor.cpp:6-16`）本身走 registry 缓存 ✓，**不**触发这条路径；但 property 取值后的 `ToDisplayString` 类操作、以及 C# 侧 `FName` 的构造/`ToString` 会。`FRegisterInputComponent.cpp:61`、`FRegisterEnhancedInputComponent.cpp:187`、`FRegisterDataTableFunctionLibrary.cpp:15`、`FRegisterUnreal.cpp:56,78` 在**每次绑定/查找**时都用 `GetString<FName>` 取名字，属于热路径。

**问题**
`FName` 的正确用法是**比较索引**而不是字符串；插件的 `Identical` 已经做对了（`FNamePropertyDescriptor.cpp:48` `NameA == *NameB` 走 `FName` 的索引比较）。但托管边界的往返全部退化为字符串，抵消了 `FName` 的最大优势。此外 `FName::ToString()` 返回的是**显示用**字符串；数字后缀语义需注意：`FName(TEXT("Foo_1"))` 与 `FName(TEXT("Foo"), 1)` 在 UE 中都解析为同一个字面名 `Foo_1`，**不存在自动编号**，因此 C# 侧 `new FName("Foo_1")` 不会得到"第 1 个 Foo 实例"的语义——若 C# 想复用已有 `FName`，应回传句柄而不是字符串。（⚠ **存疑**：这句关于 `FName` 编号语义的旁证未回引擎源码核实，不作为结论依据。）

**建议**
- C# 侧缓存 `FName` 句柄，避免同一名字反复 `Register`；`Register` 前先查 `GetStringObject`；
- 对已知为 `FName` 的场景提供按 `FNameEntryId`/`FName` 复制的边界函数，绕开字符串；
- `ToString` 路径按需懒加载（仅调试/日志时转换）。

**验证方式**
Profiler（Unreal Insights）里观察 `FNamePool` 相关锁等待；或基准测试 `FName` 构造 10 万次 vs 句柄复用的耗时差。

### P3

#### [F-PROP2-010] `FEnumPropertyDescriptor::DestroyValue` 是冗余的空覆盖

- **类别**: 死代码 | 可读性
- **严重度**: **P3**
- **复核结论**: 确认（"空覆盖与基类行为**等价**"这一点已用引擎实现直接证实）
- **可达性**: 活跃（该 `DestroyValue` 会被容器辅助类调用：`FArrayHelper.cpp:218`、`FMapHelper.cpp:119,121,215`、`FSetHelper.cpp:140`；`grep "DestroyValue\("` 全模块 **10** 命中）
- **复核证据**: `FEnumPropertyDescriptor.cpp:24-26` 空实现；基类 `FPropertyDescriptor.cpp:190-196` → `Property->DestroyValue(Dest)`；引擎侧 `Runtime/CoreUObject/Private/UObject/EnumProperty.cpp:471`（`PropertyFlags |= CPF_IsPlainOldData | CPF_NoDestructor | CPF_ZeroConstructor;`）→ `FProperty::DestroyValue`（`UnrealType.h:968-974`）在 `CPF_NoDestructor` 下**直接跳过**，根本不进入 `DestroyValueInternal`（其基类实现是 `checkf(0)` 断言：`Private/UObject/Property.cpp:1002-1005`）。故枚举上"基类实现 == 空操作 == 本覆盖"
- **级别变动**: 无（维持 P3）；**置信度由「中」升为「高」**（"UE 侧 FEnumProperty::DestroyValue 是否可能非平凡"的存疑项已核实）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Property/EnumProperty/FEnumPropertyDescriptor.cpp:24-26`
- **函数**: `FEnumPropertyDescriptor::DestroyValue(void* Dest) const`
- **置信度**: 中（未验证 UE 侧 `FEnumProperty::DestroyValue` 是否可能非平凡；枚举底层类型均为 POD，判断为无操作）

**现状（代码事实）**
```cpp
// .../FEnumPropertyDescriptor.cpp:24-26
void FEnumPropertyDescriptor::DestroyValue(void* Dest) const
{
}
```
基类实现会转调原生：
```cpp
// Source/UnrealCSharp/Private/Reflection/Property/FPropertyDescriptor.cpp:190-196
void FPropertyDescriptor::DestroyValue(void* Dest) const
{
	if (const auto Property = GetProperty())
	{
		Property->DestroyValue(Dest);
	}
}
```
**调用上下文**
`DestroyValue` 会被容器辅助类调用（`FArrayHelper.cpp:218`、`FMapHelper.cpp:119,121,215`、`FSetHelper.cpp:140`，`grep "DestroyValue\("` = 10 命中），因此这个覆盖**是可执行的**，只是什么都不做。

**问题**
枚举的底层类型是 POD，`FEnumProperty::DestroyValue` 经 `GetUnderlyingProperty()->DestroyValue(Dest)` 同样是无操作，故两者等价。空覆盖的唯一效果是**掩盖基类的 null 防护**并让读者误以为枚举有特殊的销毁语义。

**建议**
删除该覆盖与 `.h:20` 的声明，统一走基类；若有意保留（如为防止未来 `FEnumProperty` 变得非平凡），加注释说明。

**验证方式**
`grep -n "FEnumProperty::DestroyValue" Engine/Source/Runtime/CoreUObject/...` 确认底层实现为无操作。

#### [F-PROP2-011] `FOptionalHelper::Initialize()` 是空实现且全插件 0 调用点

- **类别**: 死代码
- **严重度**: **P3**
- **复核结论**: 确认（死代码，证据为强）
- **可达性**: **不可达**（死代码：全模块 0 调用点。一个从不被执行的空函数不属于"会执行到"）
- **复核证据**: `FOptionalHelper.cpp:32-34` 空函数体；`grep "Helper->Initialize\(\)"` 全模块 = **0 命中**；`grep "Helper->Deinitialize\(\)"` = **0**（`Deinitialize` 只被自身析构调用：`FOptionalHelper.cpp:29`，且 `:36-55` 有真实逻辑）→ 与 `Deinitialize` 不对称，确为"配对 API"残骸
- **级别变动**: 无（维持 P3）；**可达性由「活跃」更正为「不可达」**
- **文件**: `Source/UnrealCSharp/Private/Reflection/Optional/FOptionalHelper.cpp:32-34`（声明 `Public/Reflection/Optional/FOptionalHelper.h:16`）
- **函数**: `FOptionalHelper::Initialize()`
- **置信度**: 高

**现状（代码事实）**
```cpp
// .../FOptionalHelper.cpp:32-34
void FOptionalHelper::Initialize()
{
}
```
**调用上下文 / 问题**
`grep`（`Source/` 全模块，`*.cpp,*.h,*.inl`）：
| 模式 | 命中数 |
|---|---|
| `(Helper\|OptionalHelper)->Initialize\(\)` | **0** |
| `Helper->Deinitialize\(\)` | 0（`Deinitialize` 仅被自身析构调用：`FOptionalHelper.cpp:29`） |

即 `Initialize` 既无外部调用者、自身也是空函数体。与 `Deinitialize`（`:36-55`，有真实逻辑并被析构调用）不对称——看起来是"配对 API"写法留下的残骸。

**建议**
删除 `Initialize`（`.h:16` + `.cpp:32-34`）。

**验证方式**
`Select-String -Recurse -Include *.cpp,*.h,*.inl -Pattern "Helper->Initialize\(\)"` → 0。

#### [F-PROP2-012] `FEnumPropertyDescriptor` 的 `Get` 与 `Set` 实现逐字相同，`Get(FMember)` 与 `Get(FReturn)` 也相同

- **类别**: 可读性 | 一致性
- **严重度**: **P3**
- **复核结论**: 确认（四段实现确实逐字相同；"参数顺序同构、**不是 bug**"的判断核对 `FunctionMacro.h` 后同样成立）
- **可达性**: 活跃（`Get`/`Set` 经 `FunctionMacro.h:72`（`IN_VALUE()`）、`:123`/`:139`（2 参 `Get`）与容器辅助类调用）
- **复核证据**: `FEnumPropertyDescriptor.cpp:4-22`（`Get(FMember)`:6 与 `Get(FReturn)`:11 逐字相同；`Get(Src,Dest)`:16 与 `Set(Src,Dest)`:21 逐字相同，均为 `Property->GetUnderlyingProperty()->CopySingleValue(...)`）；对照 `FStrPropertyDescriptor.cpp:8-14` 有 `IManagedHandleIsValid` 缓存分支，而枚举**每次调用都新建装箱对象**（`Class->BoxValue(Src)`，`:6,11`）
- **级别变动**: 无（维持 P3）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Property/EnumProperty/FEnumPropertyDescriptor.cpp:4-22`
- **函数**: `Get(void*, void**, FMember)` / `Get(void*, void**, FReturn)` / `Get(void*, void*)` / `Set(void*, void*)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// .../FEnumPropertyDescriptor.cpp:4-22
void FEnumPropertyDescriptor::Get(void* Src, void** Dest, FPropertyArgument::FMember) const
{
	*Dest = IManagedHandleToObject(Class->BoxValue(Src));            // 6
}

void FEnumPropertyDescriptor::Get(void* Src, void** Dest, FPropertyArgument::FReturn) const
{
	*Dest = IManagedHandleToObject(Class->BoxValue(Src));            // 11  ← 与 6 完全相同
}

void FEnumPropertyDescriptor::Get(void* Src, void* Dest) const
{
	Property->GetUnderlyingProperty()->CopySingleValue(Dest, Src);   // 16
}

void FEnumPropertyDescriptor::Set(void* Src, void* Dest) const
{
	Property->GetUnderlyingProperty()->CopySingleValue(Dest, Src);   // 21  ← 与 16 完全相同
}
```
**调用上下文 / 问题**
`Get(Src, Dest)` 与 `Set(Src, Dest)` 的参数顺序在 primitive 约定下确实同构（`Src`=值地址、`Dest`=原生地址；见 `FunctionMacro.h:72,123,139`），因此**不是 bug**。但：
1. `Get(FMember)`/`Get(FReturn)` 的 `FReturn` 语义（"返回值需要新建对象"）在枚举上不存在，重复实现纯粹是噪音；
2. `Get(Src, Dest)` 与 `Set(Src, Dest)` 同名不同义（一个是读、一个是写）却共用一套参数命名，极易误用；
3. 枚举是唯一**每次调用都新建装箱对象、不做任何缓存**的取值路径（对照 `FStrPropertyDescriptor.cpp:8-14` 的缓存逻辑），高频读枚举属性会产生持续的托管分配——若这是有意为之（枚举值语义，避免别名）应加注释。

**建议** 合并到基类/模板，并对"枚举不做句柄缓存"的决定加注释说明；`Set` 的形参改名为 `(void* InValue, void* Dest)` 之类的方向性命名。

**验证方式** 代码审阅 + 对枚举属性高频读取观察托管分配。

#### [F-PROP2-013] 5 个 String 描述符与 5 个 `FRegister*` 重复实现同一套比较逻辑

- **类别**: 可读性 | 一致性
- **严重度**: **P3**
- **复核结论**: 确认（同一语义确实有两份实现，且两处的**查空行为不一致** —— 调用面已用 grep 复核）
- **可达性**: 活跃（`grep "Descriptor->Identical\("` = **11** 命中，全部来自容器元素比较：`FArrayHelper.cpp:148,164,180,293`、`FSetHelper.cpp:87,124,154`、`FMapHelper.cpp:101,137,164,183`；C# 侧 `FString.Identical` 走 `FRegisterString::IdenticalImplementation`）
- **复核证据**: 描述符侧 `FStrPropertyDescriptor.cpp:41-49`（`:48` `StringA == *StringB`，**无查空**）；注册器侧 `FRegisterString.cpp:21-32`（`:23-29` 双重 `if` 查空，`:31` `return 0`）；同型 `FRegisterName.cpp:21-32`、`FRegisterText.cpp:47-58`（`:53` 用 `EqualTo`）、`FRegisterAnsiString.cpp:25-36`
- **级别变动**: 无（维持 P3）
- **文件**: `FStrPropertyDescriptor.cpp:41-49`（及 FName/FText/Ansi/Utf8 同位行）对比 `FRegisterString.cpp:21-32`、`FRegisterName.cpp:21-32`、`FRegisterText.cpp:47-58`、`FRegisterAnsiString.cpp`、`FRegisterUtf8String.cpp`
- **置信度**: 高

**现状（代码事实）**
描述符侧（`FStrPropertyDescriptor.cpp:41-49`）与注册器侧（`FRegisterString.cpp:21-32`）是**同一逻辑的两份实现**：
```cpp
// FStrPropertyDescriptor.cpp:43-48                        // FRegisterString.cpp:23-27
const auto StringA = Property->GetPropertyValue(A);        const auto FoundA = ...GetString<FString>(InA);
const auto StringB = ...GetString<FString>(handle);        const auto FoundB = ...GetString<FString>(InB);
return StringA == *StringB;                                return *FoundA == *FoundB ? 1 : 0;
```
**问题**
`grep "Descriptor->Identical\("` = 11 命中，全部来自**容器元素比较**（`FArrayHelper`/`FSetHelper`/`FMapHelper`）；C# 侧的 `FString.Identical` **不经过**描述符，而走 `FRegisterString::IdenticalImplementation`。因此：
- 两处逻辑必须手工保持同步，`F-PROP2-004`（不查空）和 `FRegisterString.cpp:23-29`（**查空**）正是这种不同步的实例——同一语义两处行为不一致；
- 5 个 String 描述符之间是逐字拷贝（仅模板实参不同），本身可用模板/宏消重。

**建议** 让 `FRegister*` 通过 `FPropertyDescriptor`（或一个共享的自由函数）做比较，单一实现点；描述符内部以 `using`/宏生成 4 个同构文件。

**验证方式** 对比 `FStrPropertyDescriptor.cpp:41-49` 与 `FRegisterString.cpp:21-32` 的行为差异（传入失效句柄：前者崩溃、后者返回 0）。

#### [F-PROP2-014] `FRegisterAnsiString.cpp` 的版本守卫用了错误的宏（`UE_F_UTF8_STR_PROPERTY`）

- **类别**: 平台兼容 | Bug（潜在）
- **严重度**: **P3**（当前无害，两宏值相同）
- **复核结论**: 确认（宏确实写错；"当前无害"也成立）
- **可达性**: 活跃（该文件参与编译，且 `UE_F_UTF8_STR_PROPERTY` 在 UE 5.6 下为真，故守卫被满足而非被跳过）
- **复核证据**: `FRegisterAnsiString.cpp:2` `#if UE_F_UTF8_STR_PROPERTY`（该文件内**未出现** `UE_F_ANSI_STR_PROPERTY`，即整份 ANSI 注册代码被 UTF8 守卫包裹）；对照正确写法 `FAnsiStrPropertyDescriptor.h:6` / `.cpp:2`；宏值 `Source/CrossVersion/Public/UEVersion.h:150`（`UE_F_UTF8_STR_PROPERTY UE_VERSION_START(5, 6, 0)`）与 `:152`（`UE_F_ANSI_STR_PROPERTY UE_VERSION_START(5, 6, 0)`）**完全相同** → 当前无行为差异
- **级别变动**: 无（维持 P3：今日无害、未来若两宏分叉则升为 P1 编译失败/绑定缺失）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterAnsiString.cpp:2`（**相邻文件，非目标 16 文件**）
- **置信度**: 高（宏值已核实）

**现状（代码事实）**
```cpp
// Source/UnrealCSharp/Private/Domain/Interop/FRegisterAnsiString.cpp:2
#if UE_F_UTF8_STR_PROPERTY      // ← 应为 UE_F_ANSI_STR_PROPERTY
```
对照正确的写法：`FAnsiStrPropertyDescriptor.h:6` 与 `.cpp:2` 用的是 `UE_F_ANSI_STR_PROPERTY` ✓。
宏定义（`Source/CrossVersion/Public/UEVersion.h`）：
```
150: #define UE_F_UTF8_STR_PROPERTY UE_VERSION_START(5, 6, 0)
152: #define UE_F_ANSI_STR_PROPERTY UE_VERSION_START(5, 6, 0)
```
**问题**
两个宏**当前取值完全相同**（都是 `5.6.0`），所以今天不产生行为差异。但这是一个**静默的定时炸弹**：若将来 `FAnsiStrProperty` 与 `FUtf8StrProperty` 的引入版本被拆分（例如其中一个被后移），`FRegisterAnsiString` 会跟随 Utf8 的条件编译，导致 ANSI 字符串在 C# 侧缺少绑定（或反过来引用不存在的类型 → 编译失败，P1）。`FAnsiStrPropertyDescriptor` 本身**没有**这个问题——任务书假设的"旧版本无条件引用 → 编译失败（P1）"**在 5 个 String 描述符中不成立**，全部有正确守卫。

**建议** 把 `:2` 改为 `#if UE_F_ANSI_STR_PROPERTY`（并顺带 grep 全仓库确认没有其它同类笔误）。

**验证方式** `Select-String -Recurse -Pattern "UE_F_UTF8_STR_PROPERTY" Source/UnrealCSharp/Private/Domain/Interop/` 逐文件核对宏与文件名是否匹配。

#### [F-PROP2-015] 描述符拷贝缓冲未传对齐参数，且与 `FOptionalHelper` 的分配方式不一致

- **类别**: 可优化/可读性 | 未定义行为
- **严重度**: **P3**
- **复核结论**: 确认（两处分配方式确实不一致；"可能未对齐构造"仍属潜在项 —— 引擎默认对齐规则已核实）
- **可达性**: 活跃（`CopyValue` 由 `FunctionMacro.h:129,144` 调用，是 Compound 返回值/out 参数的必经路径）
- **复核证据**: `TCompoundPropertyDescriptor.inl:35-41`（`FMemory::Malloc(Super::Property->GetElementSize())`，**单参**）vs `FOptionalHelper.cpp:21`（`FMemory::Malloc(ValuePropertyDescriptor->GetSize(), ValuePropertyDescriptor->GetMinAlignment())`，**双参**）；引擎侧 `Runtime/Core/Public/HAL/MemoryBase.h:19-28`（`DEFAULT_ALIGNMENT = 0`，注释："Blocks >= 16 bytes will be 16-byte-aligned, Blocks < 16 will be 8-byte aligned"）—— 默认对齐需细化为"≥16 字节块 16 对齐、<16 字节块 8 对齐"，故 `alignas(32)` 之类的结构体确有风险
- **级别变动**: 无（维持 P3）
- **文件**: `Source/UnrealCSharp/Public/Reflection/Property/TCompoundPropertyDescriptor.inl:35-41` vs `Source/UnrealCSharp/Private/Reflection/Optional/FOptionalHelper.cpp:21`
- **函数**: `TCompoundPropertyDescriptor<T>::CopyValue` / `FOptionalHelper::FOptionalHelper`
- **置信度**: 中（`FMemory::Malloc` 默认对齐为 16 字节，UE 结构体对齐要求一般不超过该值，故实际可能安全）

**现状（代码事实）**
```cpp
// TCompoundPropertyDescriptor.inl:35-41                    // FOptionalHelper.cpp:21
FMemory::Malloc(Super::Property->GetElementSize())          FMemory::Malloc(ValuePropertyDescriptor->GetSize(),
                                                                        ValuePropertyDescriptor->GetMinAlignment())
```
**问题**
同一个插件里两处分配同一类对象的内存，一处传 `GetMinAlignment()`、一处用默认对齐。`CopyValue` 随后在该缓冲上 `InitializeValue` + `CopySingleValue`（`:43-45`），若某结构体的 `GetMinAlignment() > 16`（如含 `alignas(32)` 的 SIMD 类型），则可能未对齐构造。插件已有 `GetMinAlignment` 这个 API（`FOptionalHelper.cpp:21` 在用），说明作者知道它存在。

**建议** 统一为 `FMemory::Malloc(Property->GetElementSize(), Property->GetMinAlignment())`。

**验证方式** 检查含 `alignas(32)` 的结构体（如 `FVector4d` 系列）属性在 `FReturn` 路径上的地址对齐；`check((reinterpret_cast<uintptr>(Value) % Property->GetMinAlignment()) == 0)`。

#### [F-PROP2-016] 结构体构造期的 `Bind` 递归安全性依赖隐式顺序，无断言保护

- **类别**: 可优化/可读性
- **严重度**: **P3**（当前正确）
- **复核结论**: 确认（"顺序不变量存在且无断言保护"成立；整条递归链已逐跳核实，`AddClassDescriptor` 确实早于属性循环）
- **可达性**: 活跃（结构体属性绑定路径：`FPropertyDescriptor::Factory`（`FPropertyDescriptor.cpp:89`）→ `FStructPropertyDescriptor` 构造体 `.cpp:7`）
- **复核证据**: `FStructPropertyDescriptor.cpp:7`（`Bind<false>(InProperty->Struct)`）；唯一重入守卫 `FCSharpBind.inl:11-14`（`if (GetClassDescriptor(InStruct)) return true;`）；`FCSharpBind.cpp:106`（`BindImplementation`）→ `:122`（`AddClassDescriptor`）→ `:159-182`（属性循环，`:175` `AddPropertyHash` 才触发 `Factory`）→ **`:122` 严格早于 `:159`**，故当前不会无限递归；代码中确无注释/断言固化该顺序
- **级别变动**: 无（维持 P3）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Property/StructProperty/FStructPropertyDescriptor.cpp:7` 与 `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:122,159-182`、`Public/Registry/FCSharpBind.inl:11-14`
- **函数**: `FStructPropertyDescriptor::FStructPropertyDescriptor` / `FCSharpBind::BindImplementation(UStruct*)`
- **置信度**: 高（顺序已逐行确认）

**现状（代码事实）**
```cpp
// FStructPropertyDescriptor.cpp:7
	FCSharpEnvironment::GetEnvironment().Bind<false>(InProperty->Struct);
// FCSharpBind.inl:11-14  ← 唯一的重入守卫
	if (FCSharpEnvironment::GetEnvironment().GetClassDescriptor(InStruct))
	{
		return true;
	}
// FCSharpBind.cpp:122  ← 先登记类描述符
	const auto NewClassDescriptor = FCSharpEnvironment::GetEnvironment().AddClassDescriptor(InStruct);
// FCSharpBind.cpp:159-182 ← 之后才遍历属性（AddPropertyHash:175 触发 FPropertyDescriptor::Factory）
```
**问题**
"不无限递归"这一结论**完全依赖 `AddClassDescriptor`(`:122`) 早于属性循环(`:159-182`)**；代码里没有任何注释或断言固化这个不变量。任何人把 `AddClassDescriptor` 下移（例如为了"属性失败就不登记类"的清理逻辑），自引用结构体（`TArray<FSelf>`、`FSelf` 的嵌套）立刻变成**无界递归 → 栈溢出崩溃**。

**建议** 在 `BindImplementation(UStruct*)` 入口加显式重入断言/集合：
```cpp
	if (BindingInProgress.Contains(InStruct)) { return true; }   // 或 check(!BindingInProgress.Contains(InStruct))
	TGuardValue<...> Guard(...);
```
并加注释说明 `:122` 必须早于 `:159`。

**验证方式** 构造一个含 `TArray<FSelfStruct>` 成员的 `FSelfStruct`，`Bind` 后检查是否只有一条类描述符；临时把 `:122` 移到 `:182` 之后应能复现栈溢出。

---

## 11. 未覆盖 / 存疑项

1. **UE 引擎源码不可得**：分析时未定位到引擎安装目录（该前提已在其它报告中被推翻，见 `02-…/06`、`08-…/04`）。因此 `FProperty::InitializeValue`/`DestroyValue`/`CopySingleValue`/`SetPropertyValue` 的**引擎侧实现是推断而非核实**，`F-PROP2-001`、`F-PROP2-002`、`F-PROP2-010` 依赖这些假设。本会话 `web_search` 工具返回 HTTP 401（`api key invalid`）不可用，无法在线核实。
2. **`F-PROP2-001` 未能定级**：`FStructProperty::CopySingleValue` 是否重写未验证（见上）。这是本次分析最需要后续确认的一点。
3. **未读 C# 侧代码**（`Script/`）：`FString`/`FName`/`FText`/`TArray` 在 C# 侧的 `UnRegister`/`Dispose` 时机、`GCHandle` 释放路径未核实——这直接影响 `F-PROP2-003`（泄漏是否被回收缓解）与 `F-PROP2-007`（悬垂窗口长度）。任务要求 grep `Source/` 7 个模块，`Script/` 未纳入。
4. **`FStructPropertyDescriptor::Get(void*, void*)`（`.cpp:28-31`）调用点未确证**：未能找到非 primitive 描述符的 2 参 `Get` 调用点，无法判断 `NewRef` 在此路径上是否真被执行（`grep` 模式 `Descriptor->Get\(.*,\s*Dest\)` 命中 0，但该模式不精确，不作为死代码结论）。
5. **`FEnumPropertyDescriptor::BoxValue` 未展开**：`FClassReflection::BoxValue` 的具体实现未读，因此"枚举值如何映射到 C# 枚举类型/数值"只到调用边界为止。若需确认 C# 枚举缺失时的回退行为，需读 `Reflection/FClassReflection.*`。
6. **`FAnsiString`/`FUtf8String` 的 C# 侧编码语义未验证**：`FAnsiString` 从 `FString`(UTF16) 转换在非 ASCII 下的行为（代码页依赖 vs UTF-8）未核实；`FRegisterAnsiString.cpp` / `FRegisterUtf8String.cpp` 只做了宏与调用点核对。
7. **线程安全未覆盖**：这些描述符本身无锁，未检查 `Get`/`Set` 的调用线程约束（`FStringRegistry` 的 `TMap` 是否只在 GameThread 访问）。`FRegisterString.cpp:36` 用 `AsyncTask(ENamedThreads::GameThread, ...)` 包裹 `RemoveStringReference`，**暗示其余路径存在跨线程访问的可能**，但未追证。
8. **未分析同目录其它描述符**：`PrimitiveProperty/*`、`ContainerProperty/*`、`DelegateProperty/*`、`ObjectProperty/*`、`FieldPathProperty/*`（`TPropertyDescriptor.inl`/`TPrimitivePropertyDescriptor.inl` 只做了与本次结论相关的部分阅读）。
9. **`F-PROP2-006` 的文件不在指定 16 个文件内**（`FRegisterText.cpp`），因它决定 `FTextPropertyDescriptor` 的可观测语义而保留，已在条目中标注。

