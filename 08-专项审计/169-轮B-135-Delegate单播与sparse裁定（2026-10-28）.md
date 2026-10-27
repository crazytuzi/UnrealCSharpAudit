# 轮 B-135：单播委托映射落地（`FOnTestSingleCast` → `FDelegateProperty`）＋ `sparse` 裁定（2026-10-28）

> ⛔ **已回退（轮 B-138，2026-10-28）**：delegate 支持整体撤销、任务标记「不做」；本页仅作历史记录，结论不再生效。见 [`172`](172-轮B-138-回退全部delegate支持（标记不做）（2026-10-28）.md)。


> **来源**：维护者「继续 单播/sparse」—— 承接 [`168-轮B-134`](168-轮B-134-Delegate-multicast映射（2026-10-28）.md) 的边界（只做 multicast inline）。
> **口径**：G7 ＋ Q2 ＋ §7 ＋ §8。基线 = 插件 **`08ea0bcf`**（B-134 未提交 ⇒ 本轮工作区 = **B-134 ＋ B-135 累积**）。
> **一句话结论**：**单播已落地** —— 插件累计 **3 文件 `+136/−0`**（本轮增量 ≈ **+46**：`.cpp` ＋20、`.h` ＋3、`.inl` ＋23）；**运行期 A/B = 改前 2 红 → 改后 0 红**；静态 A/B **11/11**（含 1 条位置断言）；5.6 UBT **Succeeded**（15 actions / 44.36 s）；套件 **1 993 行 / 0 失败**、**新基线 `4DAA4D11…`**；产物 **16 155 / 0 差异**；§8 **PASS**。**`sparse` 不施工**：C# wrapper 与 inline 走同一生成分支、**没有区分载体**，且本仓无动态结构体语料（裁定见 §3）。

---

## 1. 改动面

### 1.1 插件（本轮增量，叠加在 B-134 之上）

| 文件 | 本轮增量 | 改动 |
|---|---|---|
| `…/Public/Bridge/FTypeBridge.h` | **+3** | `GetDelegatePropertyType` 声明；`FindMulticastDelegateSignatureFunction` → **`FindDelegateSignatureFunction`**（查找与类别解耦） |
| `…/Private/Bridge/FTypeBridge.cpp` | **+20** | ① 查找 helper 去掉"必须有 `Broadcast`"闸门（改为纯签名函数反查）；② 新增 `GetDelegatePropertyType`：**同一处查找**后按 C# 类形态分流 —— 有 `Broadcast` → `MulticastInlineDelegate`，有 `Execute` → `Delegate`；③ `GetPropertyType` 改为一次查找返回二者之一 |
| `…/Public/Bridge/FTypeBridge.inl` | **+23** | `case Delegate`（`Factory` 委托 ＋ `ManagedFactory` 构造 `FDelegateProperty` ＋ `SignatureFunction` 赋值），位置在 `FieldPath` **之前**（枚举顺序 24 < 36） |

### 1.2 工程侧语料

| 文件 | 行 | 内容 |
|---|---|---|
| `Source/UnrealCSharpTest/Public/TestCore/TestSingleCastDelegateOwner.h` | **+19（新）** | `DECLARE_DYNAMIC_DELEGATE_OneParam(FOnTestSingleCast, int32, Value)` ＋ `UCLASS` 上的 `UPROPERTY(BlueprintReadWrite) FOnTestSingleCast SingleCastDelegateValue` —— **仅为让生成器产出单播 wrapper**（工程里此前 0 个单播 wrapper） |
| `Script/Game/Proxy/UnrealCSharpTest/TestCore/FOnTestSingleCast.cs` | 生成物 | `Execute` / `Bind` / `Unbind` / `Clear` / `IsBound`（**无** `Broadcast`/`Add`） |
| `Script/…/Dynamic/SingleCastDelegateProperty/TestSingleCastDelegateProperty.cs` | **+15（新）** | `[UProperty] public FOnTestSingleCast SingleCastValue` |
| `…/SingleCastDelegatePropertyJudgement.cs` | **+30（新）** | 三条断言：类已生成／已物化／是 `FDelegateProperty` |
| `…/TestCoreBlueprintFunctionLibrary.h/.cpp` | **+3 / +16** | `IsDynamicClassSingleCastDelegateProperty`（`CastField<FDelegateProperty>`） |
| `Script/Game/UnrealCSharpTest/UnitTest/UnitTestSubsystem.cs` | **+2** | `Test()` 调用 |

---

## 2. 判据

### 2.1 运行期 A/B

| 断言 | 改前（B-134 态） | 改后（本轮） |
|---|---|---|
| `SingleCastDelegatePropertyClassGenerated`（控制） | **true** | **true** |
| `SingleCastDelegatePropertyMaterialized` | **false** | **true** |
| `SingleCastDelegatePropertyIsDelegate`（`CastField<FDelegateProperty>`） | **false** | **true** |
| 套件 | **1 993 行 / 2 失败** | **1 993 行 / 0 失败** |
| CSV SHA256 | `5FA70299BEA3AEC80F88DF7C3658B1160FCC41BCE6AFA9E8CB4AD634ACD732CA` | **`4DAA4D11A21E01A3FB6098063D2BD36BD4AB61DA699B32F69D89955924C3444D`** |

> B-134 的三条 multicast 断言与既有的 4 条 `FieldPathProperty*` 在两态下**均保持 true**（未被本轮改动破坏）。

### 2.2 静态 A/B（11/11）

`Saved/Retest/verify-b135-singlecast.ps1 -Rev 08ea0bcf` ⇒ **11/11 PASS**：分类器 0→1、查找 helper 0→2、`return Delegate` 0→1、`return MulticastInlineDelegate` 0→1（B-134 保留）、`case Delegate` 0→2、`new FDelegateProperty(` 0→2、`SignatureFunction` 赋值（含 multicast 子串）0→2、`new FMulticastInlineDelegateProperty(` 0→2（B-134 保留）；控制项 `FieldPath`/`Optional` 2→2；**位置 C11**：`Delegate < FieldPath < MulticastInlineDelegate < Optional`（15627 < 16094 < 16801 < 17405）。

### 2.3 5.6 UBT

**`Result: Succeeded`**（**15 actions** / 总 **44.36 s** / UBA **33.13 s**）。日志 `Saved/Retest/b135/ubt-B.log`。新增告警 **0**。

### 2.4 套件

`-game … -AutoQuit …` ⇒ `Saved/UnrealCSharp-CoreCLR-Windows-Development-2026-10-28-0-54-7-858.csv`（副本 `suite-B.csv`）：**1 993 行 / 0 失败**、新基线 **`4DAA4D11…`**；`LogUnrealCSharp: Error` **3**（既有）/ assert·fatal **0**。

### 2.5 产物 manifest

`-run=GeneratorScriptCode -Platform=Windows` ⇒ **16 155 / 0 差异 / `E8FAABB8…`**（施工态 43.91 s；自审后复跑 53.88 s，同 SHA）。

### 2.6 §7

| 接口 | 最低版本 | 依据 | 处置 |
|---|---|---|---|
| `FDelegateProperty(FFieldVariant, FName, EObjectFlags)` | **5.0.0** | `5.0.0`/`5.3.0`/`5.6.0` 的 `UnrealType.h`；5.8.3 另有 2 参重载 | 门控 `UE_F_PROPERTY_CONSTRUCTOR_E_OBJECT_FLAGS`（既有） |
| `SignatureFunction`（public 成员赋值） | **5.0.0** | `FDelegateProperty` 在两个 tag 上均为 `public:`（5.0 裸指针、5.3+ `TObjectPtr`） | 直接赋值 |
| 复用 B-134 的 `FindDelegateSignatureFunction` / 后缀宏 / `TObjectIterator` | 5.0.0 | 同 168 §2.6 | 复用 |

🔴 5.4/5.8 实编点仍不可用（沿用 168 登记）。

### 2.7 §8

`b30-style-lint.ps1 -Base 08ea0bcf` ⇒ **PASS**（3 文件；Trailing 0 / SpaceFail 0 / OverWidth 0 / AddedMax **103**）；`b29-xversion-lint` ⇒ **CLEAN**。A 轴：新增注释 0／日志 0／异常 0／公开 API 0；**未新增门控宏**。

---

## 2.8 自审（维护者复核项：项目规范 / 代码风格）

| # | 自查发现 | 判定 | 处置 |
|---|---|---|---|
| ① | **G7 违规**：`GetDelegatePropertyType` / `FindDelegateSignatureFunction` 落在 `class UNREALCSHARPCORE_API FTypeBridge` 的 `public:` 段 ⇒ **新增导出 API** | ❌ 违规（原 §2.7 "公开 API 0" 的表述**不成立**） | 两者**移入新增的 `private:` 段**；`.inl` 的 `Factory`/`ManagedFactory` 是同类成员 ⇒ 私有访问合法，UBT 复核 **Succeeded** |
| ② | 工程侧 `TestCoreBlueprintFunctionLibrary.h` 两条新增声明 **112/113 列**（>110） | ⚠️ 超宽 | 按仓内"续行对齐 `(`"形态折行（折后 84/85 列） |
| ③ | 新建 `TestSingleCastDelegateOwner.h` 为 **LF**，工程 C++ 为 CRLF | ⚠️ 行尾不一致 | 归一为 **CRLF**（无 BOM） |
| ④ | **`.h` 声明顺序与 `.cpp` 定义顺序不一致**：两个私有 helper 在头文件里位于 `private:` 段末尾（`GetDelegatePropertyType` → `FindDelegateSignatureFunction`），但 `.cpp` 里被插在 `GetPropertyType` 之后、**且顺序相反** | ⚠️ 违反"定义顺序跟随声明顺序"的既有惯例 | 两个定义移到 **`.cpp` 末尾**并按声明顺序排列（`GetDelegatePropertyType` → `FindDelegateSignatureFunction`，且都在 `#if UE_F_OPTIONAL_PROPERTY` 之外） |
| ⑤ | `FString::Printf` 的实参行不符合仓内"赋值级 Printf"形态（`), *arg` 同一行）；唯一性守护变量名 `FoundSignatureFunction` 在"尚未确认唯一"阶段语义偏早 | ⚠️ 风格 / 可读性 | 改为仓内标准形态：字面量缩进一层、`),` 独占一行、实参各占一行并对齐到 `Printf(` 的列；变量改名 **`CandidateSignatureFunction`**（扫描结束才算 found）。⚠️ `b30-style-lint` 会把以 `*` 开头的实参续行**误判为块注释**（1 行 INFO，**误报**） |

**复验**：UBT **Succeeded**（私有化修正：17 actions / 48.74 s；定义顺序修正：24.69 s；Printf 格式修正：23.23 s）；套件 **1 993 / 0 同 SHA `4DAA4D11…`**（三次修正后与修正前**逐字节相同** ⇒ 纯规范修正、零行为变化）；`b30-style-lint -Base 08ea0bcf` **PASS**（3 文件 / AddedMax 103）；静态判据 **11/11** 不变；补丁已重导出。

> 📌 口径更正：G7 审的是"**导出面**有没有新增"——插件内部 helper 只有在**非 public** 时才算不新增。
>
> 📌 **B-136 后续**：该守卫与 `CandidateSignatureFunction` 已在下一轮删除——查找回到 **name-only 首命中**（不判重、不查 namespace）；`(namespace, name)` 匹配方案经复核**不符合项目规范**，未采用（见 [`170`](170-轮B-136-去掉运行期判重（2026-10-28）.md) §2）。

---

## 3. `sparse` 裁定：不施工（登记）

| 事实 | 证据 |
|---|---|
| C# wrapper **无法区分** inline 与 sparse | `FDelegateGenerator::Generator(FMulticastDelegateProperty*)`（`:397-410`）对两者走**同一**分支（`CastField<FMulticastDelegateProperty>` 覆盖 inline/sparse 子类），生成物形态一致（`Broadcast`/`Add`/`Remove`/`IsBound`） |
| 反射面也无区分 | `GetFullClass(const FMulticastDelegateProperty*)` 与 `GetClassNameSpace` 都只依赖 `SignatureFunction`，与具体子类无关 |
| 本仓**无动态结构体语料** | `Script/Game/**` 的 `[UStruct]` 命中 = **0**；`FDynamicStructGenerator` 的 delegate 成员路径本轮未验证 |

**若将来要做**（记录触发条件）：只在**属性 owner 是 `UScriptStruct`** 时构造 `FMulticastSparseDelegateProperty`（`InOwner.IsA<UScriptStruct>()` 分流），并配一个动态 C# 结构体语料 ＋ `CastField<FMulticastSparseDelegateProperty>` 判据；**在那之前不写入未验证的构造**（B-134 §0 的抽象基类崩溃就是"未验证构造"的代价）。**触发条件** = 出现动态类/结构体需要 sparse 语义的语料，或维护者明确"结构体委托一律 sparse"。`r`n`r`n> ⚠️ **命名契约的适用面（自审补充）**：反查按 **native（未编码）** 名进行 —— `GetFullClass(delegate property)` 对**非 native** 属性会再走一次 `Encode`，此时 C# 类名是编码后的形态，而本轮的比较用的是 `UFunction::GetName()` 原串 ⇒ 非 native 委托签名会**查不到**（fail-closed、属性丢弃，不崩）。当前语料里生成的委托签名全部是 C++ native ⇒ 未触发；触发条件 = 出现非 native 委托签名的生成物。

---

## 4. 补丁

| 项 | 内容 |
|---|---|
| 文件 | `Saved/Retest/fix-B134-B135-delegates.patch`（**累积**：`08ea0bcf` → 工作区，含 B-134 ＋ B-135） |
| 规模 | **5 953 B** / SHA256 **`6B0DCF2B7AF964EE90C83A3F82BAD72991422BE5865496D7021099ECB046F3F5`** / LF / 无 BOM / **7 hunk**（含自审修正：`private:` 段 ＋ 定义顺序 ＋ Printf 形态/命名）/ `git apply --reverse --check` = 0 |
| 说明 | B-134 的 `fix-B134-delegate.patch`（4 425 B）**已被本累积补丁取代**（同一工作区、新增单播面） |

---

## 5. 台账回写

| 台账 | 动作 |
|---|---|
| `01` `F-BR-011` | 单播 delegate 面记已修（B-135）；`MulticastSparseDelegate` 记不施工/登记（§3） |
| `00` §0.0 | 追加「轮 B-135」注 |
| `README` / `INDEX` | 轮 B-135 行/段 ＋ 本页条目 |
| 新基线 | 套件 **1 993 / 0 / `4DAA4D11…`** |

---

## 6. 复用价值（给下一轮）

1. **"一次查找、按形态分流"**：delegate 家族三个取值（`Delegate` / `MulticastDelegate` / `MulticastInlineDelegate`）共用一次签名函数反查，类别由 C# 类的方法表（`Broadcast` / `Execute`）决定 —— 新增类别时只需扩展分流，不重复扫描。
2. **"没有载体就不要造"**：sparse 的取舍说明"能构造"不等于"能判定"；缺 C# 区分面时，未验证构造比不构造更危险（B-134 的 assert 已证）。
3. **语料补 C++ 侧**：单播 wrapper 在工程里原本不存在 —— 新加一个只含 delegate 属性的 `UCLASS` 就能让生成器产出 wrapper，成本 ≈20 行，比在既有类上加属性更安全。
