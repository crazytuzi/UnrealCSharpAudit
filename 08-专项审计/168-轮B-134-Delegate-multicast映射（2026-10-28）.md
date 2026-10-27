# 轮 B-134：`Delegate` 家族映射落地 —— multicast inline（`FOnPickUp` → `FMulticastInlineDelegateProperty`）（2026-10-28）

> ⛔ **已回退（轮 B-138，2026-10-28）**：delegate 支持整体撤销、任务标记「不做」；本页仅作历史记录，结论不再生效。见 [`172`](172-轮B-138-回退全部delegate支持（标记不做）（2026-10-28）.md)。


> **来源**：维护者「开始 Delegate」—— 承接 [`167-轮B-133`](167-轮B-133-Delegate家族裁定（2026-10-28）.md) 的裁定（不施工/登记）。本轮把该页的"方向 A（命名契约反查，高风险）"推进为**文档化命名的精确逆运算**并落地 multicast inline 面。
> **口径**：G7 ＋ Q2 ＋ §7 ＋ §8。
> **一句话结论**：插件 **3 文件 `+90/−0`**（`.h` ＋4、`.cpp` ＋62、`.inl` ＋24）；**运行期 A/B = 改前 2 红 → 改后 0 红**；静态 A/B **13/13**（含 2 条位置断言）；5.6 UBT **Succeeded**（15 actions / 132.03 s）；套件 **1 990 行 / 0 失败**、**新基线 SHA `A2A0B18F…`**（旧 1 987 / `D520F5AD…` ＋ 3 条永久判据）；产物 manifest **16 155 / 0 差异**；§7 见 §2.6；§8 **PASS**。

---

## 0. 对 `167` 的两处更正（本轮最重要的事实）

| # | B-133 的结论 | 本轮更正 |
|---|---|---|
| ① | "C# 声明侧**没有可判定的类型载体**" | **部分不成立**：确实没有 marker class，但**存在可逆的命名契约** —— `FUnrealCSharpFunctionLibrary::GetFullClass(const FDelegateProperty*)`（`:522-551`）= `"F" ＋ SignatureFunction->GetName() − HEADER_GENERATED_DELEGATE_SIGNATURE_SUFFIX`（再按 `IsNative` 走一次 `Encode`），`GetClassNameSpace` 同族。⇒ 反向：C# 类名去掉首 `F` ＋ 恢复后缀，再**全局唯一匹配**一个 `UFunction` 即可 —— 这不是启发式，是**同一个命名函数的逆运算** |
| ② | "`SignatureFunction` 无来源" | 来源即 ①（找到的那个 `UFunction`）；且该成员在两个委托属性类里都是 **`public:`**（`UnrealType.h:6014-6016`/`:6073-6076`），可直接赋值 |

🔴 **实测踩到并纠正的第三件事（决定成败）**：**不能构造 `FMulticastDelegateProperty`**。首版按 C# 类名构造它 → 启动期 `PreInitPostStartupScreen` 直接 **assert**（`Assertion failed: 0 … Property.cpp:1261`，即 `FProperty::LinkInternal` 的 `check(0)`：**它是抽象基类**）。必须构造 **`FMulticastInlineDelegateProperty`**（`BlueprintAssignable` 的真实属性类）——这也与 C# 侧 wrapper（`Broadcast`/`Add`/`Remove`/`IsBound`）的生成来源一致。

---

## 1. 改动面

### 1.1 插件

| 文件 | 行 | 改动 |
|---|---|---|
| `…/Public/Bridge/FTypeBridge.h` | **+4** | `class UFunction;` 前置声明 ＋ `static UFunction* FindMulticastDelegateSignatureFunction(const FClassReflection*)` |
| `…/Private/Bridge/FTypeBridge.cpp` | **+62** | ① 签名函数反查：要求 C# 类名以 `F` 开头、**方法表含 `Broadcast`**（multicast 判据）；按 `"F" + 类名去F + __DelegateSignature` 扫描 `TObjectIterator<UFunction>`，**命中 0 或 >1 一律返回 `nullptr`（fail-closed）**；② `GetPropertyType` 在 **FieldPath 之后、Optional 之前**返回 `EPropertyTypeExtent::MulticastInlineDelegate` |
| `…/Public/Bridge/FTypeBridge.inl` | **+24** | ① `Factory<>` 同位置加 `case MulticastInlineDelegate` → 委托 `ManagedFactory`；② `ManagedFactory` 构造 `FMulticastInlineDelegateProperty`（双 ctor 分支，`UE_F_PROPERTY_CONSTRUCTOR_E_OBJECT_FLAGS`）＋ `SignatureFunction = Found` |

### 1.2 工程侧语料（永久判据装置）

| 文件 | 行 | 内容 |
|---|---|---|
| `Script/…/Dynamic/DelegateProperty/TestDelegateProperty.cs` | **+15（新）** | `[UClass] UTestDelegateProperty : UObject` ＋ `[UProperty] public FOnPickUp OnPickUpValue { get; set; }`（`FOnPickUp` 是工程里既有的 multicast delegate wrapper） |
| `…/DelegateProperty/DelegatePropertyJudgement.cs` | **+30（新）** | 三条断言：类已生成／属性已物化／属性是 `FMulticastDelegateProperty`（`CastField`，inline/sparse 都是它的子类） |
| `Source/UnrealCSharpTest/Public/TestCore/TestCoreBlueprintFunctionLibrary.h` | **+3** | `IsDynamicClassMulticastDelegateProperty` 声明 |
| `…/Private/TestCore/TestCoreBlueprintFunctionLibrary.cpp` | **+16** | 实现（`FindObject<UClass>` → `FindPropertyByName` → `CastField<FMulticastDelegateProperty>`） |
| `Script/Game/UnrealCSharpTest/UnitTest/UnitTestSubsystem.cs` | **+2** | `Test()` 内调用 |

> ⚠️ 工程树仍带预存 WIP（60 `M` ＋ 120 `??`），3 个「修改」文件为追加。

---

## 2. 判据

### 2.1 运行期 A/B

| 断言 | 改前（`08ea0bcf`） | 改后（本轮） |
|---|---|---|
| `DelegatePropertyClassGenerated`（控制） | **true** | **true** |
| `DelegatePropertyMaterialized` | **false** | **true** |
| `DelegatePropertyIsMulticastDelegate`（`CastField<FMulticastDelegateProperty>`） | **false** | **true** |
| 套件总计 | **1 990 行 / 2 失败** | **1 990 行 / 0 失败** |
| CSV SHA256 | `F8A5B86C835D04A20A0EDCF3FCEAFB1BFD8EC0CB1A37B90493299E83DF2F3AAC` | **`A2A0B18FA947348E61E5CE33EF526F9C5B7D54A0D37A67352C5547D406BBC6B8`** |

> ⚠️ **中间态留痕**：首版构造 `FMulticastDelegateProperty` ⇒ 启动期断言崩溃（`Saved/Retest/b134/suite-B.log`，`assert=2`、无 CSV）；改为 `FMulticastInlineDelegateProperty` 后本轮通过。**"能编译"与"能 link"是两回事**。

### 2.2 静态 A/B（13/13）

`Saved/Retest/verify-b134-delegate.ps1 -Rev 08ea0bcf` ⇒ **13/13 PASS**：`.h` 前置声明/声明 0→1、`return EPropertyTypeExtent::MulticastInlineDelegate` 0→1、helper 定义＋调用 0→2、后缀宏 0→1、`case MulticastInlineDelegate` 0→2、`new FMulticastInlineDelegateProperty(` 0→2、`SignatureFunction` 赋值 0→1、Factory 委托调用 0→1；控制项 `case FieldPath`/`case Optional` 2→2；**位置** C12/C13：`FieldPath < MulticastInlineDelegate < Optional`（4505 < 4625 < 4801 / 15483 < 16190 < 16803）。

### 2.3 5.6 UBT

**`Result: Succeeded`**（**15 actions** / 总 **132.03 s** / UBA executor **108.68 s** —— 头文件改动触发多 TU 重编）。日志：`Saved/Retest/b134/ubt-B3.log`。

### 2.4 套件

`-game … -AutoQuit …` ⇒ `Saved/UnrealCSharp-CoreCLR-Windows-Development-2026-10-28-0-42-47-999.csv`（副本 `suite-B2.csv`）：**1 990 行 / 0 失败**、新基线 **`A2A0B18F…`**；`LogUnrealCSharp: Error` **3**（既有）/ assert·fatal **0**。

### 2.5 产物 manifest

`-run=GeneratorScriptCode -Platform=Windows` ⇒ **16 155 文件 / 0 差异 / `E8FAABB8…`**（87.70 s）⇒ 不影响 C++→C# 生成物。

### 2.6 §7 版本兼容性

| 接口/写法 | 最低支持版本 | 依据 | 处置 | 实编 |
|---|---|---|---|---|
| `FMulticastInlineDelegateProperty(FFieldVariant, const FName&, EObjectFlags)` | **5.0.0** | `5.0.0`／`5.3.0`／`5.6.0` 的 `UnrealType.h:6183-6199`（5.0/5.3 只有 3 参；5.8.3 有 2 参 ＋ 3 参转发） | 门控 `UE_F_PROPERTY_CONSTRUCTOR_E_OBJECT_FLAGS`（既有宏） | 5.6 **Succeeded** |
| `SignatureFunction`（公开成员，赋值） | **5.0.0** | 两个委托属性类均 `public:`；5.0 为裸指针、5.3+ 为 `TObjectPtr<UFunction>`（两者均可由 `UFunction*` 赋值） | 直接赋值 | 同上 |
| `HEADER_GENERATED_DELEGATE_SIGNATURE_SUFFIX` | 引擎既有宏 | 仓内既有先例 `FUnrealCSharpFunctionLibrary.cpp:538` | 复用 | 同上 |
| `TObjectIterator<UFunction>` | 5.0.0 | 引擎 CoreUObject 既有 API | 复用 | 同上 |

🔴 **未取得**：5.4/5.8 实编点仍不可用（装置链接指向迁移前路径，沿用 B-131/132 记录）。

### 2.7 §8

`b30-style-lint.ps1 -Base 08ea0bcf` ⇒ **PASS**（3 文件；Trailing 0 / SpaceFail 0 / OverWidth 0 / AddedMax **103**；`diff --check` 0）；`b29-xversion-lint -Mode Syntax` ⇒ **CLEAN**。A 轴：新增注释 0／日志 0／异常 0／公开 API 0（`GetPropertyType`/`Factory` 签名未变；新增的 `FindMulticastDelegateSignatureFunction` 当时落在导出类的 `public:` 段（**G7 违规**），已在 B-135 自审中移入 `private:`（见 [`169`](169-轮B-135-Delegate单播与sparse裁定（2026-10-28）.md) §2.8））；Q2 失败面 = `nullptr`（不新增错误码）；**未新增门控宏**。

---

## 3. 边界

1. **只做 multicast inline**（`BlueprintAssignable` 形态）：单播 `FDelegateProperty` 与 `MulticastSparseDelegateProperty` **未做**（触发条件 = 出现对应 C# 声明语料）；`FMulticastDelegateProperty` 抽象基类**永远不能**直接构造（本轮断言实证）。
2. **只验"属性物化 ＋ 类型正确"**：未验 C# 侧 get/set 语义（wrapper 的 native helper 建立于"属性被读进 C#"路径；动态类上的写入语义未构造判据）。
3. **反查是 fail-closed**：0 命中或 >1 命中都返回 `nullptr`（属性丢弃，不崩）；跨模块同名委托签名会落到 >1 ⇒ 放弃（登记）。
4. **扫描成本**：仅在"未匹配任何已知类型 ＋ C# 类方法表含 `Broadcast`"时触发一次 `TObjectIterator<UFunction>`（生成期）。
5. 工程树 WIP；子模块仍 `5.5`。

---

## 4. 补丁

~~`Saved/Retest/fix-B134-delegate.patch`~~（4 425 B / `5F027C4C…` / 5 hunk）—— 🟢 **已被累积补丁取代**：`Saved/Retest/fix-B134-B135-delegates.patch`（**5 953 B** / `6B0DCF2B…` / 7 hunk，含 B-134 ＋ B-135 ＋ 自审修正；见 [`169`](169-轮B-135-Delegate单播与sparse裁定（2026-10-28）.md) §4）。

---

## 5. 台账回写（本轮执行）

| 台账 | 动作 |
|---|---|
| `01` `F-BR-011` | 追加「multicast inline delegate 面已修（B-134）」；单播/sparse 与 C# 写入语义登记 |
| `00` §0.0 | 追加「轮 B-134」注 |
| `README` / `INDEX` | 轮 B-134 行/段 ＋ 本页条目 |
| `167` | 标注"已被 B-134 部分推翻并落地"（页面保留作裁定史） |
| 新基线 | 套件 **1 990 / 0 / `A2A0B18F…`** |

---

## 6. 复用价值（给下一轮）

1. **"命名契约的逆运算"是一种可复用的绑定手段**：当 C# 类型是由 C++ 生成的"无标记类"时，先找生成它的命名函数，再反推；关键是加**全局唯一性守卫**（0/>1 放弃），把不确定性变成 fail-closed。
2. **抽象基类陷阱**：引擎里同名族常有"抽象基类 ＋ inline/sparse 两个具体子类"（委托、可能还有别的）——构造前必须确认 `LinkInternal` 有覆写；本轮以一次启动期 assert 换来这条纪律。
3. **判据装置的复用**：`CastField<FMulticastDelegateProperty>` 覆盖 inline/sparse 两个子类，只需换第三件断言即可扩展到 sparse。
