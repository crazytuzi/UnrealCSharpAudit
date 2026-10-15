# 轮 B-23 收口：`F-FN-006` 遗留面**收尾** —— 按值 `TFieldPath` 与 `TOptional` 参数改为"每次调用一份快照"（A1 再扩展）

> **来源**：维护者指定 —— "**开始FFieldPath/FOptional**"（接 [`08-…/60`](60-轮B22收口-按值容器与字符串参数快照A1扩展（2026-10-14）.md) §5.1 登记的同条遗留面）；复核意见二则 —— "**谓词命名再想一下 ＋ 按项目规范应写成类静态函数**"（见 §2.1）。
> **插件提交**：🟢 **已提交 `cf27778f`**（2026-10-15 17:14:36 +0800，父 `f4b45fab`，**2 文件 `+32/−1`**）—— ⚠️ **B-22 与 B-23 改同一处代码 ⇒ 两轮补丁累积式**；**本轮 2/2 文件 blob 与已验证工作区逐字节相同**（`.cpp` `741cc269…` / `.h` `39777d39…`）⇒ 本轮补丁转为**留档**（对 HEAD `--reverse --check` 仍 **exit 0**，正向 apply 对应父提交 `cf27778f`）。
> **改动面**：插件 **2 文件**（B-23 增量 = `FManagedFunctionDescriptor.{h,cpp}`：谓词**改名 ＋ 移入类**；连同 B-22 累计 `+32/−1`）＋ 工程侧 3 文件（**新 2 个 `BlueprintImplementableEvent`（+8）＋ 2 条 C# 永久判据（+42）＋ 驱动 4 次调用（+13）**）＋ **生成产物重跑**（内容 delta = **1 文件**）；C# 运行时 0 / 协议面 0 / 导出签名 0 / **未新增公开 API**（谓词为 `private static` 成员）/ **未新增 `UE_F_*` 宏** / 注释·日志·`throw`·`try-catch`·`ensure`·`check` **全 0**（工程侧仅 1 行分节注释，与驱动既有 `// TArray`/`// TSet`/`// TMap` 同形）。
> 环境：UE 5.6.0 · Win64 · `-game` 套件（CoreCLR）；tag = **`b23-*`**；对照基线 = 轮 B-22 的 **1929 行 / `3D51F26C…`**。

---

## 0. 摘要

| 项 | 读数 |
|---|---|
| 语义目标 | 把"**按值参数 = 每次调用专属、归托管所有的快照**"（B-20 结构体 → B-22 容器/字符串族）再扩到**`FFieldPathProperty`（`TFieldPath<FField>`）与 `FOptionalProperty`（`TOptional<T>`）**；谓词白名单 +2，落点/判定/形态全部不变 |
| 形式规范（维护者复核） | 谓词由**文件级 `static` 自由函数**改为 **`FManagedFunctionDescriptor` 的 `private static` 成员**（声明进 `FManagedFunctionDescriptor.h`）；同时改名 `IsCopyParameterProperty` → **`IsCopyValueProperty`**（理由与备选见 §2.1） |
| 判据（**永久**用例 ×2） | C# 侧留存第 1 次收到的**参数对象**＋其值的 C# 快照（`TFieldPath.IsValid()` / `TOptional.Get()`），第 2 次调用带**不同值**（空路径 / `7`）时断言留存对象仍是旧值 |
| 🔴 改前（`b23-pre`） | 2 条**全 `false`**（**1931 行 / 2 红**）⇒ 别名在两个族上**实测成立** |
| 🟢 改后（`b23-post`） | 2 条**全 `true`**（**1931 行 / 0 红 ×2**，两趟 CSV SHA256 相同 **`B429F3EE…`**） |
| 🟢 复跑（共 6 趟） | **交付态复跑**（编辑器会话生成后，§3.4）＋ **行宽修订后复跑**（`b23-style`，§3.5）＋ **改名/移入类后复跑**（`b23-rename`，§3.6）＋ **顺序修订后复跑**（`b23-order`，§3.7）＋ **写法修订后复跑**（`b23-tail`＋`b23-tail2`，§3.8）—— 后七趟 CSV **均与 `b23-post` 逐字节 0 差异** |
| 🔴 **零副作用证据** | 改前 ↔ 改后 CSV 差异 = **仅这 2 行翻转**（共 4 行 = 2 删 2 增）；与 B-22 基线比 = **只多这 2 行**，其余 **1929 行逐字节不变** |
| 🟢 **生成产物 delta** | 全量重跑生成器：**16230 → 16230 文件**，内容变化 **1 个文件**（`Script/Game/Proxy/…/TestCSharpFunctionActor.cs`：47 231 B `ED04BC0D…` → 48 120 B `398C8206…`），其余 **16 229 个产物逐字节不变**（`b23-before` ↔ `b23-after` manifest 对照） |
| 编译 | UBT **Succeeded** ×3（工程模块 7 actions 含 UHT 重新生成；插件 `Module.UnrealCSharp.cpp` ＋ 重链；**改名后再次** 11 actions：头文件消费者重编 ＋ 两个模块重链）；`dotnet build Script/Game/Game.csproj -c Debug` **0 Warning / 0 Error** |
| 日志（改前 1 ＋ 改后 2 ＋ 复跑 3 趟） | `LogUnrealCSharp: Error` **0**、`LogBlueprint: Error` **0**、`ensure` **0**、`Assertion failed` **0**、`Fatal error` **0**、`Unhandled Exception` **0** |
| 生成会话 | **真实编辑器会话**（既有装置 `run-g13-generate.ps1` ＋ `pie_driver_g13gen.py`）：wall **281.1 s**、产物计数 **delta 0/0**、`LogBlueprint: Error` **18**（= 历史基线 18，见 §1.6）、`ensure/assert` 0、`LogUnrealCSharp: Error` 0 —— 🔴 **不采用 commandlet 路径**（本机实测：`-run=GeneratorScriptCode` 会话 **756 个 `LogBlueprint: Error` ＋ "Failure - 706 error(s)"**；这正是 `pie_driver_g13gen.py:3-5` 早已记录"commandlet 不可用"的同一条事实） |
| 补丁留档 | `Saved/Retest/fix-B23-fieldpath-optional-param-snapshot.patch`（**2 685 B**，SHA256 `BF9EA92498DF991F5C94E279E10ACDD24033514B1F69D904D9D82B2EC6B3E765`，UTF-8 无 BOM / LF，`git apply --reverse --check` **exit 0**（改动已入库 ⇒ 补丁转为**留档**）；⚠️ **累积式**——含 B-22 内容） |

> 📌 **一句话**：`F-FN-006` 的整改面在本轮**收官** —— 会被地址别名影响的复合类型（结构体 / 容器 / 字符串族 / FieldPath / Optional）**全部**走拥有型快照；剩余未纳入的只有**委托**（维护者裁定不做）与**对象族**（机制上无别名）。

---

## 1. 设计前提

### 1.1 落点、判定、形态与 B-22 完全一致（只加两个谓词项）

| 事实 | 证据 |
|---|---|
| 落点仍是 `Invoke` 唯一一处 | `FManagedFunctionDescriptor.cpp:46-49`（调用点条件 4 项不变） |
| 判定仍是插件自己的 `ReferencePropertyIndexes` | `FFunctionDescriptor.cpp:51-59` ↔ `FCSharpFunctionDescriptor.cpp:114-131`（同源） |
| 新谓词项只加"两个复合类型" | `#if UE_F_OPTIONAL_PROPERTY` → `IsA<FOptionalProperty>()`；尾部 `IsA<FFieldPathProperty>()` |
| 新增引擎头 include 2 个（其中一个门控） | `UObject/FieldPathProperty.h`（5.0.3 / 5.4.0 / 5.7.0-preview-1 **均存在**，`git cat-file -e` exit 0 ×3）、`UObject/PropertyOptional.h`（5.0.3 **不存在** / 5.3.0 起存在 ⇒ 与 `UE_F_OPTIONAL_PROPERTY = UE_VERSION_START(5,3,0)` **逐版本吻合**） |

### 1.2 为什么 `TFieldPath` 走"`CopyValue` ＋ `Get<FReturn>`"是安全的（引擎契约 + 拥有链）

| # | 契约 | 证据 |
|---|---|---|
| E1 | `FFieldPathProperty : TProperty<FFieldPath, FProperty>`；`LinkInternal` 令 `ElementSize = sizeof(FFieldPath)` | `FieldPathProperty.h:24`、`UnrealType.h:1533-1537` |
| E2 | `TProperty::CopyValuesInternal` 做的是**逐元素赋值**（`GetPropertyValuePtr(Dest)[i] = GetPropertyValuePtr(Src)[i]`）⇒ `FFieldPath` 的 `TArray<FName> Path` 被**深拷贝**；且引擎内**没有** `TIsPODType<FFieldPath>` 之类的特化 ⇒ `CopySingleValue` 一定走 `CopyValuesInternal`（不走 `Memcpy`） | `UnrealType.h:1539-1545`、`UnrealType.h:822-835`、全树 grep `TIsPODType<FFieldPath｜TIsZeroConstructType<FFieldPath｜TStructOpsTypeTraits<FFieldPath>` = **0 命中** |
| E3 | `Get<FReturn>` 的拥有链：`AddFieldPathReference<true,false>` → `FFieldPathAddress(Src, /*InNeedFree=*/true)` → `TOwnedValue::Free()` 闸门 → `~FFieldPath()` ＋ `FMemory::Free(Src)` ⇒ 与 `CopyValue` 的 `FMemory::Malloc` **同配** | `FFieldPathPropertyDescriptor.cpp:22-35`、`FFieldPathRegistry.inl:12`、`FFieldPathRegistry.h:25-35`、`TOwnedValue.inl:11-19` |
| E4 | UHT **显式**把 FieldPath 标为"可作参数" | `UhtFieldPathProperty.cs:44`（`IsParameterSupportedByBlueprint` ＋ `IsMemberSupportedByBlueprint`）；引擎先例 `KismetSystemLibrary.h:1040`（`const TFieldPath<FField>&` 形参的 UFUNCTION） |

### 1.3 为什么 `TOptional` 也安全（**尺寸**是这条链上唯一的坑，已逐条核销）

| # | 契约 | 证据 |
|---|---|---|
| E5 | `FOptionalProperty::LinkInternal` 里 **`SetElementSize(CalcSize())`** ⇒ `ElementSize` **已含**尾随的 set 标志；`FOptionalProperty::HasIntrusiveUnsetOptionalState()` **恒 false** ⇒ `CalcSize() = Align(valueSize + 1, valueMinAlign)` 这一档恒成立 | `PropertyOptional.cpp:149-155`、`PropertyOptional.h:139-149`、`PropertyOptional.cpp:67-71` |
| E6 | ⇒ `TCompoundPropertyDescriptor::CopyValue` 的 `Malloc(ElementSize)` **尺寸正确**（这正是 `F-LEAK-016` 当年在 `FOptionalHelper` 里手算并修好的同一公式，见 [`08-…/27`](27-F-LEAK-016收口-可选值owned分配尺寸与set标志越界（2026-09-29）.md)） | `TCompoundPropertyDescriptor.inl:33-48`、`FOptionalHelper.cpp:21-26` |
| E7 | `CopyValuesInternal` 是深拷贝：set ⇒ `MarkSetAndGetInitializedValuePointerToReplace` ＋ `ValueProperty->CopySingleValue`；unset ⇒ `MarkUnset` | `PropertyOptional.cpp:413-428` |
| E8 | ⚠️ `LinkInternal` **会**从内层属性传播 `CPF_IsPlainOldData`（`:163-164`）⇒ `TOptional<int32>` 走 `Memcpy` 分支、`TOptional<FString>` 走 `CopyValuesInternal` 分支 —— **两者都正确**（前者本来就是 POD；后者是真深拷贝） | `PropertyOptional.cpp:163-164`、`UnrealType.h:822-835` |
| E9 | `Get<FReturn>` 的拥有链：`new FOptionalHelper(Property, Src, /*bNeedFreeData=*/true, false)` ＋ `AddOptionalReference<...,false>`；helper 析构 `MarkUnset(Data)` ＋ `FMemory::Free(Data)`；注册表 `RemoveReference` `delete` helper ⇒ 与 `CopyValue` 的 `Malloc` **同配** | `FOptionalPropertyDescriptor.cpp:25-40`、`FOptionalHelper.cpp:42-52`、`FOptionalRegistry.cpp:65-89`、`FOptionalRegistry.inl:6-19` |
| E10 | UHT 支持 TOptional 参数（内层类型支持即可；且专门有"参数"校验分支） | `UhtOptionalProperty.cs:79-87`、`:186-200` |

### 1.4 判据装置：为什么**必须新增测试函数**（不能复用既有语料）

- UE→C# 方向要出现 `TFieldPath`/`TOptional` **参数**，必须有一个**由 C# 覆盖**的 UFUNCTION；全量普查：`Source/UnrealCSharpTest/**` 里 `TFieldPath|TOptional` 命中 **0**（改前），`Script/Game/**` 命中的只是**属性**用法（`FPropertyAccessSegment.Property`、`TOptional<...>` 属性），**没有任何** `..._Override` 或引擎函数带这两类**参数** ⇒ 只能新增。
- 新增的 2 个函数（`TestCSharpFunctionActor.h:219-223`）：
  ```cpp
  	UFUNCTION(BlueprintCallable, BlueprintImplementableEvent)
  	void SetFieldPathValueFunction(const TFieldPath<FField>& InFieldPathValue);

  	UFUNCTION(BlueprintCallable, BlueprintImplementableEvent)
  	void SetOptionalValueFunction(const TOptional<int32>& InOptionalValue);
  ```
- 生成器对这两类参数的产物形态（本轮实测）：`public new void SetFieldPathValueFunction_Override(TFieldPath<FField> InFieldPathValue)`、`public new void SetOptionalValueFunction_Override(TOptional<int> InOptionalValue)` ⇒ **类型映射两条都通**（C# 侧 `TFieldPath<FField>` = `Script.CoreUObject.TFieldPath<T>`，`where T : Script.Reflection.Property.FField`）。

### 1.5 判据形态（与 B-20/B-22 同形）

| # | 判据名 | 首次值（驱动） | 留存断言 | 第二次值 |
|---|---|---|---|---|
| 1 | `CSharpFieldPathParameterKeepsValue` | `TFieldPath<FField>` ← `Generate(ATestCSharpFunctionActor::StaticClass()->FindPropertyByName("Int32Value"))`（**有效**路径） | `RetainedFieldPathParameter.IsValid() == 首次读数(true)` | `TFieldPath<FField>()`（**空**路径） |
| 2 | `CSharpOptionalParameterKeepsValue` | `TOptional<int32>(2)` | `RetainedOptionalParameter.IsSet() && .Get() == 首次读数(2)` | `TOptional<int32>(7)` |

> 📌 判别力三条件同 B-22：**留存对象** ＋ **第二次带不同值** ＋ **同一描述符（同一池缓冲）**。本轮 FieldPath 的"不同值"取"有效 → 空"（`IsValid()` 是可观测翻转量）；Optional 取 2 → 7。

### 1.6 🔴 生成会话：**只有真实编辑器会话可用**（commandlet 会话实测被证否）

| 会话 | 命令 | 产物 | 日志 |
|---|---|---|---|
| ❌ commandlet | `UnrealEditor-Cmd -run=GeneratorScriptCode -Platform=Windows` | 产物**正确**（delta 1 文件）但会话报 **"Failure - 706 error(s), 157 warning(s)"**、`LogBlueprint: Error` **756**（全部落在 4 个 `Content/UnitTest/Dynamic/**` BP 资产：`BP_TestBlueprintRawDynamicFunctionActor` 322 / `BP_TestBlueprintBlueprintDynamicFunctionActor` 295 / `BP_TestBlueprintRawDynamicPropertyActor` 81 / `BP_TestBlueprintBlueprintDynamicPropertyActor` 58） | —— |
| ✅ 编辑器会话 | `UnrealEditor.exe <uproject> -ExecCmds="py pie_driver_g13gen.py"`（既有装置） | **delta 0/0**（16155/75 不变）＝ 与 commandlet 产物**逐字节一致**（manifest 对照） | `LogBlueprint: Error` **18** ＝ 历史基线（`b17-gen` 同为 **18**）、`ensure/assert` **0**、`LogUnrealCSharp: Error` **0** |

**解释（与既有结论同源）**：命令 `FDynamicGenerator::Generator()` 在 commandlet 会话里被**跳过**（`UnrealCSharpEditor.cpp:62-65`）⇒ C# 动态类未物化 ⇒ 依赖它们的 4 个 Dynamic BP 资产在该会话里编译失败；编辑器会话里 C# 域先初始化 ⇒ 只剩 18 条历史 BP 告警。这条事实与 `pie_driver_g13gen.py:3-5` 的"why not the commandlet"注释**互相印证**（那里的理由是"commandlet 注册的文件集更小、会被 `EndGenerator` 删掉 107 个产物"；本轮实测的**同一条路**另有 756 个 BP 错误）。

---

## 2. 施工（唯一落点）

### 2.1 谓词：改名 ＋ 移入类（维护者复核意见）

**命名（备选与取舍）**：

| 候选 | 取舍 |
|---|---|
| `IsCopyParameterProperty`（初版） | 语义正确，但"Parameter ＋ Property"双后缀拗口，且在类里读起来重复 |
| **`IsCopyValueProperty`（采用）** | ① **直接映射它触发的那个 API**（`PropertyDescriptor->CopyValue(...)`），读名即知走哪条路；② 与既有分类谓词 **`FPropertyDescriptor::IsPrimitiveProperty()`** **同形**（"这是一类按值拷贝的属性"）；③ 去掉冗余的"Parameter"（调用点就在参数循环里，语境自明）；④ 对"返回值/出参"方向同样成立，不会因复用而说谎 |
| `IsOwnedParameterProperty` | 顺"拥有型"口径，但 UE 语境里 `Owned` 首先指 **Outer 归属**，易误读 |
| `IsSnapshotParameterProperty` | 只与报告口径（"快照"）对齐；插件代码里 `Snapshot` 仅作局部变量名（`FCSharpCompilerRunnable.cpp:542` 的 `FileChangesSnapshot`），不是 API 词汇 |
| `ShouldCopyParameter(...)` | 动词语义更准，但全库谓词命名以 `Is*` 为绝对主流（`Is*` **993** 处 / `Has*` 176 / `Can*` 19 / **`Should*` 仅 2**）⇒ 不采用 |

**形式（按项目规范改为类静态成员）**：先例 = 轮 B-21 新增的 `IsValidPath` 是 `FDirectoryPathCustomization` 的**类内成员**（`DirectoryPathCustomization.h:27`），不是文件级 `static`；本仓头文件里 `static` 成员函数亦是主流写法（`FClassGenerator.h:16-34`、`FCSharpCompilerRunnable.h:42-48` 等）。谓词只服务 `Invoke` ⇒ 收成 **`private static`**（`FManagedFunctionDescriptor` 未标 `UNREALCSHARP_API` ⇒ 不构成公开 API 面）。

```cpp
// Source/UnrealCSharp/Public/Reflection/Function/FManagedFunctionDescriptor.h:10-18
protected:
	bool Invoke(const FMethodReflection* InMethod,
	            const IManagedHandle InManagedHandle,
	            const TFunctionRef<void*(int32)>& InPropertyAddress,
	            void* InReturnAddress,
	            const TFunctionRef<void*(const FPropertyDescriptor*)>& InOutPropertyAddress);

private:
	static bool IsCopyValueProperty(const FProperty* InProperty);
```

```cpp
// Source/UnrealCSharp/Private/Reflection/Function/FManagedFunctionDescriptor.cpp:1-8
#include "Reflection/Function/FManagedFunctionDescriptor.h"
#include "Reflection/FReflectionRegistry.h"
#include "Domain/FDomain.h"
#include "UEVersion.h"
#include "UObject/FieldPathProperty.h"
#if UE_F_OPTIONAL_PROPERTY
#include "UObject/PropertyOptional.h"
#endif
```

```cpp
// 同文件 :125-146（谓词；定义放在 Invoke 之后 —— 与头文件"先 Invoke、后谓词"的声明顺序一致）
bool FManagedFunctionDescriptor::IsCopyValueProperty(const FProperty* InProperty)
{
	return InProperty->IsA<FNameProperty>() ||
		InProperty->IsA<FStructProperty>() ||
		InProperty->IsA<FArrayProperty>() ||
		InProperty->IsA<FStrProperty>() ||
#if UE_F_UTF8_STR_PROPERTY
		InProperty->IsA<FUtf8StrProperty>() ||
#endif
#if UE_F_ANSI_STR_PROPERTY
		InProperty->IsA<FAnsiStrProperty>() ||
#endif
		InProperty->IsA<FTextProperty>() ||
		InProperty->IsA<FMapProperty>() ||
		InProperty->IsA<FSetProperty>() ||
		InProperty->IsA<FFieldPathProperty>()
#if UE_F_OPTIONAL_PROPERTY
		|| InProperty->IsA<FOptionalProperty>();
#else
		;
#endif
}
```

- 调用点（`:46-49`）第 4 项随之变为 `IsCopyValueProperty(Property)`，其余 3 项与 `if/else` 形态**一字不变**。
- 🔁 **两处顺序按复核意见对齐**（`b23-order`）：① **声明/定义顺序** —— 头文件是 `Invoke`（protected）在前、谓词（private）在后，`.cpp` 也随之改为 `Invoke :10-123` → 谓词 `:125-146`（此前谓词在文件开头，与声明顺序相反）；② **白名单遍历顺序与 `FPropertyDescriptor::Factory`（`FPropertyDescriptor.cpp:47-134`）逐项同序** —— `FNameProperty`(83) → `FStructProperty`(89) → `FArrayProperty`(91) → `FStrProperty`(95) → `FUtf8StrProperty`(98) → `FAnsiStrProperty`(102) → `FTextProperty`(105) → `FMapProperty`(123) → `FSetProperty`(125) → `FFieldPathProperty`(127) → `FOptionalProperty`(130)。两处修订后 **UBT Succeeded ＋ 套件 1931 / 0、CSV 与修订前逐字节 0 差异**（§3.7）。
- 🔁 **写法按复核意见再修订**（`b23-tail` / `b23-tail2`）：① `return` 与**首个操作数同行**（`return InProperty->IsA<FNameProperty>() ||`），续行 `+1 tab` —— 仓内先例 `FDoxygenConverter.cpp:192-195`（`return InChar == ' ' ||` ＋ 续行）、同文件 `:46-49` 的 `&&` 链同款；② **去掉悬空的 `|| false;`**：因为门控项 `FOptionalProperty` 恰好是 `Factory` 的最后一项，末项不再带尾随 `||`，把**运算符与分号一起交给门控块** —— 首版为 `#if … Term(); #else false; #endif`（`b23-tail`，仍有 `false`），随后按复核意见改为 **`#if UE_F_OPTIONAL_PROPERTY` → `|| InProperty->IsA<FOptionalProperty>();` / `#else` → `;` / `#endif`**（`b23-tail2`）：**两种配置下都不出现 `false`**，<5.3 时表达式自然以 `…IsA<FFieldPathProperty>() ;` 结束。⚠️ 这一处必然采用**前置运算符**（`||` 起行）—— 若沿用尾随 `||` 写法，`#else` 会退化成非法的 `… || ;`。备选写法（未采用）：把 `FOptionalProperty` 与 `FFieldPathProperty` 互换位置彻底消掉 `#if/#else`（代价 = 与 `Factory` 的顺序在末两项上不一致）；或改成"先 `if` 大表达式 `return true` ＋ 尾部 `#if` 早返回"（多个出口、更长）。
- **白名单现在覆盖 11 个类型**：结构体 / `TArray` / `TMap` / `TSet` / `FString` / `FName` / `FText` / `FUtf8String`（5.6+） / `FAnsiString`（5.6+） / `TFieldPath` / `TOptional`（5.3+）。
- **未新增**注释/日志/异常/断言；**未新增公开 API**。
- ⚠️ B-22 与 B-23 改**同一处代码** ⇒ `fix-B23-…patch` 是**累积补丁**（含 B-22），与 `fix-B22-…patch` **不可叠加**（对当前工作区：前者 `--reverse --check` **exit 0**、后者 **exit 1**）。

---

## 3. 判据（改前 ↔ 改后）

### 3.1 装置与流程

| 步骤 | 命令 | 结果 |
|---|---|---|
| ① 装置编译 | UBT `UnrealCSharpTestEditor Win64 Development` | **Succeeded**（7 actions：UHT 重生成 ＋ `TestCSharpFunctionActor.cpp` ＋ `UnitTestSubsystem.cpp` ＋ 2 个 unity 块 ＋ 链接；唯二 2 条警告是既有 `TSoftObjectPtr` 弃用） |
| ② 产物重跑 | commandlet → **编辑器会话**（§1.6） | 产物 delta **1 文件**（commandlet）→ 编辑器会话复跑 **delta 0**（确定性） |
| ③ C# 编译 | `dotnet build Script/Game/Game.csproj -c Debug` | **0 Warning / 0 Error** |
| ④ 改前套件 | `run-batch.ps1 -Tag b23-pre -Runs 1` | **1931 行 / 2 红**（CSV `942DF2DD…`） |
| ⑤ 插件改动 ＋ UBT | §2；UBT **Succeeded**（`Module.UnrealCSharp.cpp` ＋ 重链） | —— |
| ⑥ 改后套件 | `run-batch.ps1 -Tag b23-post -Runs 2` | **1931 行 / 0 红 ×2**（CSV `B429F3EE…`，两趟相同） |
| ⑦ 交付态复跑 | 编辑器会话生成后（C# 重新发布）再跑 1 趟 | 见 §3.4 |
| ⑧ 行宽修订复跑 | 行宽修订后 `dotnet build` ＋ 再跑 1 趟（tag `b23-style`） | 见 §3.5 |
| ⑨ 改名/移入类后复跑 | UBT（11 actions，含头文件消费者重编）＋ 再跑 1 趟（tag `b23-rename`） | 见 §3.6 |
| ⑩ 顺序修订后复跑 | 声明/定义顺序 ＋ 白名单与 `Factory` 同序后 `UBT` ＋ 再跑 1 趟（tag `b23-order`） | 见 §3.7 |
| ⑪ 写法修订后复跑 | `return` 同行首操作数 ＋ 续行 `+1 tab` ＋ 门控收尾（去掉悬空 `|| false;`）后 `UBT` ＋ 再跑 1 趟（tag `b23-tail`） | 见 §3.8 |
| ⑫ 写法再修订后复跑 | 前置 `||` 放进门控块（`#else` 只留 `;`，两种配置下都没有 `false`）后 `UBT` ＋ 再跑 1 趟（tag `b23-tail2`） | 见 §3.8 |

### 3.2 读数

| # | 判据 | 改前（`b23-pre`） | 改后（`b23-post`） |
|---|---|---|---|
| 1 | `CSharpFieldPathParameterKeepsValue` | **false** | **true** |
| 2 | `CSharpOptionalParameterKeepsValue` | **false** | **true** |
| 3 | 套件行数 / 失败 | 1931 / **2** | 1931 / **0 ×2** |
| 4 | CSV SHA256 | `942DF2DD4DC54350…` | **`B429F3EE013B8C73…`（两趟相同）** |
| 5 | 与 B-22 基线（1929 行 / `3D51F26C…`）差异 | **只多 2 行**（全 false），其余 1929 行逐字节不变 | **只多 2 行**（全 true），其余 1929 行逐字节不变 |
| 6 | 改前 ↔ 改后（`Compare-Object -SyncWindow 30`） | — | **4 行 = 2 删 2 增，全部是这 2 条判据**（其余 1929 行逐字节相同） |
| 7 | 日志（Error/蓝图/ensure/assert/Fatal/Exception） | 全 0 | 全 0（两趟） |

### 3.3 生成产物 delta（本轮唯一"非源码"改动面）

| 项 | 值 |
|---|---|
| 产物文件数 | **16230 → 16230**（`Script/UE/Proxy` 16155 ＋ `Script/Game/Proxy` 75） |
| 内容变化 | **1 个文件**：`Script/Game/Proxy/UnrealCSharpTest/UnitTest/Reflection/TestCSharpFunctionActor.cs`（47 231 B / `ED04BC0D…` → 48 120 B / `398C8206…`） |
| 其余产物 | **16 229 个逐字节不变**（manifest `path|length|sha256` 全表 `Compare-Object` = 2 行、同一文件的删/增） |
| 确定性 | 编辑器会话复跑 ⇒ **delta 0/0**（产物与 commandlet 会话产物逐字节一致） |

### 3.4 交付态复跑（编辑器会话生成后）

> 编辑器会话生成末段会触发 `FCSharpCompiler::ImmediatelyCompile`，重新发布了 `Content/Script/{UE,Game}.dll`（00:04:49/52）。为免"验证态 ≠ 交付态"，生成后再跑 **1 趟** `-game` 套件（tag `b23-final`）：

| 读数 | 值 |
|---|---|
| 套件 | **1931 行 / 0 失败** |
| CSV SHA256 | **`B429F3EE013B8C73…`**（与 `b23-post` **逐字节对照 = 0 行差异**） |
| 日志 | `LogUnrealCSharp: Error` / `LogBlueprint: Error` / `ensure` / `Assertion failed` / `Fatal error` / `Unhandled Exception` **全 0** |

⇒ **交付态 = 验证态**。

### 3.5 交付前格式与规范核查（沿用 B-21 §21.3 / B-22 §3.4 口径）

| 维度 | 实测（本轮触及的文件） |
|---|---|
| 行尾 | 改前改后一致 **CRLF**（插件 `.cpp`/`.h`、`TestCSharpFunctionActor.h`、`UnitTestSubsystem.cpp`、C# 判据文件 —— CR 数 == LF 数） |
| BOM | **不变**（插件 2 文件与工程侧 3 文件均无 BOM） |
| 缩进 | C++ 新增行全 **Tab**；C# 新增行全 **4 空格** |
| `git diff --check` | 插件仓 ＋ 工程仓（本轮 4 个工程侧文件）**exit 0** |
| 行宽 | 插件新增行 **>110 列 = 0 处** |
| 🔴 **发现并修掉 1 处偏差** | 首次交付的 C# 两条断言分别宽 **123 / 119** 列 ⇒ 会把该文件最大行宽由 HEAD 的 **117** 推到 **123**（"变差"）⇒ 已改为换行形态 ⇒ 文件最大行宽回到 **117（未变差）**；🟢 **重建 C#（0 Warning / 0 Error）＋ 复跑套件 1 趟**（tag `b23-style`）：**1931 行 / 0 失败、CSV SHA256 `B429F3EE…` 与 `b23-post` 逐字节 0 差异** |
| C++ 命名/形式 | 谓词采用 **`Is*` 谓词名**（全库 `Is*` 993 / `Has*` 176 / `Can*` 19 / `Should*` 2）＋ **类静态成员**（先例：`IsValidPath` 为类成员；`FClassGenerator.h:16-34` 等 `static` 成员成片） |
| 工程侧注释 | 驱动新增 **1 行分节注释**（`// TFieldPath / TOptional`），与同文件既有 `// TArray`/`// TSet`/`// TMap` 分节注释同形；插件侧 **0** |
| 不做全树格式化 | 仓内无 `.clang-format`/`.editorconfig`（B-21 §21.3 已定调）⇒ 只做上表对照 |

### 3.6 改名/移入类后的复跑（`b23-rename`）

| 读数 | 值 |
|---|---|
| UBT | **Succeeded**（11 actions：头文件消费者重编 ＋ `Module.UnrealCSharp.cpp` ＋ `UnrealEditor-UnrealCSharp{,Test}.dll` 重链） |
| 套件 | **1931 行 / 0 失败** |
| CSV SHA256 | **`B429F3EE013B8C73…`**（与 `b23-post` **逐字节对照 = 0 行差异**） |
| 日志 | `LogUnrealCSharp: Error` / `LogBlueprint: Error` / `ensure` / `Assertion failed` **全 0** |

⇒ 改名 ＋ 移入类 **语义零变化**（判据与 CSV 均不变）。

### 3.7 声明/定义顺序 ＋ 白名单顺序修订后的复跑（`b23-order`）

| 读数 | 值 |
|---|---|
| 修订内容 | ① `.cpp` 中谓词定义移到 `Invoke` 之后（与头文件声明顺序一致）：`Invoke :10-123` → 谓词 `:125-146`；② 谓词白名单改为与 `FPropertyDescriptor::Factory`（`FPropertyDescriptor.cpp:47-134`）**逐项同序** |
| UBT | **Succeeded**（4 actions：`Module.UnrealCSharp.cpp` ＋ 重链） |
| 套件 | **1931 行 / 0 失败** |
| CSV SHA256 | **`B429F3EE013B8C73…`**（与 `b23-post` **逐字节对照 = 0 行差异**） |

⇒ 两处顺序修订同样 **语义零变化**。

### 3.8 写法修订后的复跑（`b23-tail` / `b23-tail2`）

| 读数 | 值 |
|---|---|
| 修订内容（23-tail） | ① `return` 与首个操作数同行、续行 `+1 tab`（先例 `FDoxygenConverter.cpp:192-195`）；② 去掉悬空的 `|| false;` —— 门控改套在**收尾符**上（`#if … InProperty->IsA<FOptionalProperty>(); #else false; #endif`）⇒ ≥5.3 的表达式里没有 `false`（`#else` 仍有 `false`） |
| 修订内容（23-tail2） | 按复核意见把**前置 `||` 放进门控块**：末项 `InProperty->IsA<FFieldPathProperty>()` 不再带尾随 `||`，门控块为 `#if UE_F_OPTIONAL_PROPERTY` → `|| InProperty->IsA<FOptionalProperty>();` / `#else` → `;` / `#endif` ⇒ **两种配置下都不出现 `false`**；<5.3 时表达式自然以 `…IsA<FFieldPathProperty>() ;` 结束（⚠️ 该处因此必须用**前置运算符**；尾随 `||` ＋ 空 `#else` 会退化成非法的 `… || ;`） |
| UBT | **Succeeded**（4 actions：`Module.UnrealCSharp.cpp` ＋ 重链） |
| 套件 | **1931 行 / 0 失败** |
| CSV SHA256 | **`B429F3EE013B8C73…`**（与 `b23-post` **逐字节对照 = 0 行差异**） |

⇒ 写法修订同样 **语义零变化**；谓词行号区间 `:125-146` 未变。

---

## 4. 台账与锚点

| 项 | 值 |
|---|---|
| 插件提交（B-22 ＋ B-23 累计） | 🟢 **`cf27778f383b31c22b15d61d694c2be654532760`**（父 `f4b45fab`，2026-10-15 17:14:36 +0800，`crazytuzi`；本轮 = **仅这 2 个文件**，**2 文件 `+32/−1`**）：`Source/UnrealCSharp/Public/Reflection/Function/FManagedFunctionDescriptor.h`（`+3`）＋ `…/Private/Reflection/Function/FManagedFunctionDescriptor.cpp`（`+29/−1`）；**提交树 2/2 blob 与已验证工作区逐字节相同**（`.cpp` `741cc269ef6044d8dfce3c7cd562ce3384fcf0e6` / `.h` `39777d398c10de1968098278e80ec8b6ae25d0f8`） |
| 交付文件 SHA256 | `.cpp` `8F8054BBA4BBF9C6FE1266BA954B6EDEBC0E91D3DC4334C370D514326B1BB788`（blob `741cc269ef6044d8dfce3c7cd562ce3384fcf0e6`）；`.h` `D3C86F6C3BC75E6DF587162BA713A09468F55210A7C52FFAE2995B00B66C7FC6`（blob `39777d398c10de1968098278e80ec8b6ae25d0f8`） |
| 补丁（留档） | `Saved/Retest/fix-B23-fieldpath-optional-param-snapshot.patch`（2 685 B / `BF9EA924…` / LF / 无 BOM / 反向 check exit 0；**累积式**；⚠️ 改动已入库 ⇒ 正向 apply 对应父 `f4b45fab`，对 HEAD 只作反向参照） |
| 判据留档 | `Saved/Retest/out/b23-pre/`（CSV `942DF2DD…` ＋ log）、`out/b23-post/`（×2，`B429F3EE…`）、`out/b23-final/`（交付态）、`out/b23-style/`（行宽修订）、`out/b23-rename/`（改名/移入类）、`out/b23-order/`（顺序修订）、`out/b23-tail/` 与 `out/b23-tail2/`（写法修订）—— **后七趟 CSV 同 SHA256、互相 0 差异** |
| 生成留档 | `Saved/Retest/b23-generate.log`（commandlet，含 §1.6 的失败读数）、`Saved/Retest/out/b23-gen-editor/`（编辑器会话 log ＋ 产物路径表）、manifest `Saved/Retest/out/g8-manifest/products-b23-{before,after}.txt` |
| 编译留档 | `b23-dev-ubt.log`、`b23-dev-dotnet.log`、`b23-post-ubt.log`、`b23-rename-ubt.log`、`b23-order-ubt.log`、`b23-tail-ubt.log`、`b23-tail2-ubt.log`、跑测日志 `b23-{pre,post,final,style,rename,order,tail,tail2}-run.log`、`b23-gen-editor-run.log` |
| 工程侧新增 | `Source/UnrealCSharpTest/Public/UnitTest/Reflection/TestCSharpFunctionActor.h`（+2 include、+2 UFUNCTION）＋ `Script/Game/UnrealCSharpTest/UnitTest/Reflection/CSharpFunction/TestCSharpFunctionActor.cs`（+2 判据）＋ `Source/UnrealCSharpTest/Private/UnitTest/UnitTestSubsystem.cpp`（+4 次调用） |
| 预存未提交项 | `M Source/SourceCodeGenerator`（插件子模块指针，**非本轮**）；工程侧既有未提交测试文件（**非本轮**） |

### 4.1 ⚠️ 维护记录：本轮对台账文件的一次工具事故与修复（如实登记）

- **事故**：本轮为"改名"批量更新台账时，一段 PowerShell 脚本用了 `@(@('旧','新'))` 形式的**一元嵌套数组** —— PowerShell 会把它**展平**成两个字符串，于是循环把"字符串"当成了"字符对"，执行了**逐字符替换**（而非整串替换）：
  - `INDEX.md` / `09-问题清单/01-全局问题总表.md`：**所有 U+6362（换）→ U+6210（成）**；
  - `README.md`：**所有 U+2192（→）→ 空格**；
  - 本轮新写的报告 `08-…/60`、`08-…/61` 亦被同类替换波及（`s→t`、`b→o` 等）。
- **修复（已全部完成并复核）**：
  1. **参考锚定还原**（脚本 `Saved/Retest/repair-ledger-chars*.ps1`）：以 **git HEAD 版本**为参考，逐个定位被替换字符的上下文并还原 ⇒ `INDEX.md` **31/31**、`README.md` **223/223**、`01-全局问题总表.md` **60/60**；`README.md` 另有 B-18…B-23 六行的 **35 处箭头**（新文本，不在 HEAD 内）按上下文逐处还原（多空格运行扫描：六行内已 **0** 处）。
  2. **可疑模式全表扫描**：`成成`/`成行`/`替成`/`交成`/`切成`/`更成`/`兑成`/`成取`/`变成`/`转成`/`成与解决方案生成` 于 5 个台账文件全表复核 ⇒ 除**合法词**（`切成两半`、`转成`、`完成取证`、`变成`）外**无残留**；01 的 **14 处被改坏的报告链接**（`…Doxygen转成与解决方案生成.md`）已全部还原为 `…转换与解决方案生成.md` 且目标文件 `Test-Path` = **True**。
  3. **报告 60/61**：因逐字符替换面过大（61 的 `s` 计数为 **0**），**按会话内原文整篇重写**，并把本轮改名/移入类、`b23-rename` 复跑、补丁新哈希等一并写入。
  4. **链接完整性**：三份台账的 markdown 链接目标逐一 `Test-Path`（未解析项经核为**既有**跨目录写法，非本轮引入）。
- **残留风险（如实声明）**：还原覆盖"HEAD 已包含的文本"＋"已识别的可疑模式"；**未被 HEAD 覆盖、也不落在可疑词表内的**被替换字符理论上无法机械判定 —— 本轮以"B-18…B-23 六行箭头逐处人工复核 ＋ 5 文件可疑词全表扫描"作为兜底，未发现残留。

---

## 5. 未做 / 未取得（如实声明）

1. **`F-FN-006` 整改面至此收官，但仍有"维护者裁定不做"的委托族**（`FDelegateProperty`/`FMulticastDelegate`/`FMulticastInline`/`FMulticastSparse`）与**机制上无别名的对象族 ×8**（对象标识语义）——两者都**未纳入**白名单，**触发条件**同 [`08-…/60`](60-轮B22收口-按值容器与字符串参数快照A1扩展（2026-10-14）.md) §5.1。
2. **成本仍未量化**：每族多一次 `Malloc` ＋ 深拷贝 ＋ 一个 C# 包装器；本轮同样只证"功能零副作用"（CSV 全表对照），未测开销，也未量化副本释放时点。
3. **未单独取判据的方向**：委托**回调**方向（`FCSharpDelegateDescriptor` 走同一个 `Invoke`）与"`TFieldPath`/`TOptional` 作**返回值**/出参"（后者本就走 `PROCESS_RETURN`/`PROCESS_OUT` 的同一链）。
4. **未跑**：Mono / LeanCLR、打包版、非 Windows；`TOptional` 在 **5.3/5.4/5.5** 上的运行期（本机只有 5.6；版本门控与头文件可用性已用引擎 tag 逐版本核）。
5. **`Class == nullptr` 退化态**与 `TOptional` 的 intrusive 档（`HasIntrusiveUnsetOptionalState()` 在 5.6 恒 false；侵入式档的 `CalcSize()` 走另一分支，本机无从构造）**未测**。
6. **产物重跑是全量重跑**：本轮虽只改了 1 个测试头文件，但装置是"全量生成 16230 个产物"（wall 281 s）；报告只证明了**内容 delta = 1 文件**，未优化生成耗时。

---

## 6. 复用价值（给下一轮）

1. **"能不能做判据"要先查类型在三个环节的支持**：UHT（`IsParameterSupportedByBlueprint` / `ValidateFunctionArgument`）→ 插件生成器（产物里的 C# 形参类型）→ C# 运行期（`TFieldPath.IsValid()` / `TOptional.Get()` 这类**可观测**读法）。三处都能第一手查（UHT 源码 ＋ 产物 ＋ `Script/UE/**` 的 C# 类型），比"先写装置再试"省一整轮。
2. **尺寸类前提要一路查到引擎构造函数**：`FOptionalProperty` 的坑不在插件而在"`ElementSize` 是否含 set 标志" —— 答案在 `LinkInternal` 的 `SetElementSize(CalcSize())`（`PropertyOptional.cpp:155`）。同时别忘了 `CPF_IsPlainOldData` 的**传播**（`:163-164`）会让 POD 内层走 `Memcpy`（这里恰好正确）。
3. **命名与形式都有仓内先例可查，别凭手感**：谓词用 `Is*`（993 : 2 压倒性）；"判断型小函数"放进**所属类的 `static` 成员**（`IsValidPath` 为类成员；`FClassGenerator.h`/`FCSharpCompilerRunnable.h` 里 `static` 成员成片）；命名优先**映射它触发的那个 API**（`IsCopyValueProperty` ↔ `CopyValue`）。
4. **同一文件的多轮改动 ⇒ 补丁是累积式**：必须在报告里写清"与上一轮补丁不可叠加"，并给出两者的反向 check 读数（本轮：累积 exit 0 / B-22 exit 1）。
5. **生成装置的"会话类型"是判据的一部分**：commandlet 会话在本项目**不可用**（756 BP 错误 ＋ "Failure"总结行），编辑器会话才是规范路径（18 条历史 BP 告警）；两者产物**逐字节一致**这条正好可以当"确定性"判据用。
6. **"有效 → 空"是把不可观测值变成可观测翻转量的常用手法**：`TFieldPath` 没有便于比较的 C# 值语义，但 `IsValid()` 在"有效/空"之间翻转 ⇒ 判据成立（`TOptional` 则直接用 `Get()`）。
7. 🔴 **批量改文档时，"对/组"结构必须显式构造**（PowerShell 的 `@(@(a,b))` 会**展平**）：本轮因此发生逐字符替换事故（§4.1）。安全做法 = 用 `,@($a,$b)` 强制嵌套、或逐条做**整串**替换，并在写入前**先打印替换计数**核对；涉及未提交内容的仓库，**先复制备份**再批量改。
