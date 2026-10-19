# 轮 B-37 收口 · `F-GEN-030`：`BINDING_CLASS` 等价运算符缺空值守卫（生成器模板修复）

> **口径**：G7 / [`08-…/66`](66-待决策页-三问（Q1四路线·Q2口径例外·Q3失败判据）（2026-10-15）.md) Q2 —— **不新增注释 / 日志 / `throw` / `try-catch` / `ensure` / `check` / 公开 API**；生成物**只由生成器产出**（不手改 `Script/*/Proxy/**`）。
> **插件提交**：🟢 **`87dac2017bb8194eccf0b930a5d4ed2408c2886d`**（`87dac201`；父 `e6ce0a3f`；2026-10-19 23:42:24 +0800；subject `BindingClass Equality Null Guard && UProperty Initializer IL Strip && Container Bulk Enumerate && Reference Owner Guard`；**本轮 1 文件 ＋35 行**）。
> **状态**：🟢 **`F-GEN-030` 已修**（P3 / 潜伏 → 已修）。原登记留下的"运行期后果未取证"由本轮**静态回核 ＋ 4 条永久判据**关闭，且**推翻了原预判的"可能崩"**。
> **来源**：[`09-…/00`](../09-问题清单/00-推荐优先修复清单.md) §0.0「轮 B-36」注的下一顺位 ①（"`F-GEN-030` 受控探针"），本轮把"探针"与"修法"合并为一轮。

---

## 0. 结论摘要

| 项 | 结论 |
|---|---|
| **原登记** | `Equals(object Other) => this == Other as X;` 在 `Other` 非同类时 `as` 得 `null` ⇒ **0 句柄**进原生 `operator ==`；后果"未知"（[`08-…/77`](77-轮B-35收口-F-GEN-015残余量化（2026-10-17）.md) §6） |
| **后果（本轮取证）** | 🔴 **不崩，但静默错值**：0 句柄在原生侧回退 `static Default`（`TPropertyValue.inl:234-241`）⇒ 零值实例的 `Equals(非同类)` / `Equals(null)` / `== null` 返回 **true**、`!= null` 返回 **false**（.NET 契约全反） |
| **缺陷面** | **生成器模板面**，不是"所有生成类"：5 975 个含 `Equals(object)` 的代理里 **5 974 个是 UStruct 代理**（`FStructGenerator.cpp:151-167` 本就发三段守卫）；**唯一缺口 = `BINDING_CLASS` 生成路径**（`operator ==` 走通用函数发射器、无守卫） |
| **今日命中面** | **1 个类型**（`Script/Game/Proxy/Binding/FRawTestStruct.cs`）；`Script/UE/Proxy/Binding/**` 的 **47 个类型**（87 个 `.cs`）中 `operator ==` **0 个** |
| **改动** | `FBindingClassGenerator::GeneratorPartial` **＋35 行**：对 `operator ==` / `operator !=`（两参数均为本类、静态、二元）发**三段空值守卫**，形态镜像 `FStructGenerator` |
| **判据** | **4 条永久判据**：改前 **4 红**（1963 行）→ 改后 **0 红**（1963 行，CSV `150779588C…` **×2 逐字节相同**） |
| **产物** | `Script/UE/Proxy` **16 155** / `Script/Game/Proxy` **75**（计数不变）；**仅 `FRawTestStruct.cs` 变化**（2 315 → 2 609 B，＋294 B），其余 **16 229 文件 0 差异** |
| **提交树** | 提交 blob 与已验证工作树**相同**（`430483c2…`）；§7 **CLEAN** / §8 **PASS**（`b30-style-lint.ps1`） |

---

## 1. 静态回核：为什么"0 句柄"不崩、而是静默错值

### 1.1 三条代码事实（逐跳）

| 跳 | 代码事实 | 依据 |
|---|---|---|
| ① C# 侧取句柄 | `HandleData.GetHandle(null)` ⇒ **0**（`InObject != null` 才查表，否则末尾 `return 0`） | `Script/Interop/Handle/HandleData.cs:94-107` |
| ② 原生侧按句柄取对象 | `FBindingRegistry::GetBinding<T>(0)` ⇒ `ManagedHandle2BindingAddress.Find(0)` 未命中 ⇒ **`nullptr`** | `Source/UnrealCSharp/Public/Registry/FBindingRegistry.inl:7-12` |
| ③ 值的回退语义 | `TBindingPropertyValue<T>::Get(const IManagedHandle)` ⇒ `SrcBinding != nullptr ? *SrcBinding : Default`，`Default` 是 **`static std::decay_t<T>`（零初始化）** | `Source/UnrealCSharp/Public/Binding/Core/TPropertyValue.inl:234-241`；入参构造见 `TArgument.inl:59-62` |

⇒ 原生 `EqualToImplementation` 收到的是 **(this 的值, Default 值)**，**不会解引用空指针**。因此原登记里"若崩会污染整趟套件"的预案**不成立** —— 判据可以直接进套件，不需要独立会话（这是对 [`08-…/77`](77-轮B-35收口-F-GEN-015残余量化（2026-10-17）.md) §6 取证形态的一处更正）。

### 1.2 缺陷面清点（为什么只有 1 个类型）

- `Script/UE/Proxy` ＋ `Script/Game/Proxy` 共 **5 975** 个含 `public override bool Equals(object …)` 的文件，其中 **5 974** 个的 `operator ==` **在取句柄之前**就把 `null` 短路成 `false`（`FStructGenerator.cpp:151-167` 生成的三段守卫）。
- 唯一没有守卫的是 `FRawTestStruct` —— 它是**项目侧 `BINDING_CLASS`**：`Equals` / `GetHashCode` 由 `FBindingClassGenerator` 追加（`:671-679`），而 `operator ==` / `!=` 走**通用函数发射器**（同一 `GeneratorPartial` 的函数循环），该路径**从不判空**。
- `Script/UE/Proxy/Binding/**` 的 **47 个类型**（87 个 `.cs`）**没有一个定义 `operator ==`**（实测 `bHasEqualTo` 只在 `FRawTestStruct` 成立）⇒ 今日命中面 = 1 个类型；但**模板面**属于"将来任何用户 `BINDING_CLASS` 带 `operator ==` 都会中"。

### 1.3 与 `F-GEN-015` 的关系（不重复登记）

同一文件里 `GetHashCode() => 0` 是 [`08-…/77`](77-轮B-35收口-F-GEN-015残余量化（2026-10-17）.md) 已量化的 `F-GEN-015` 残余面（**降级、不施工**）。本轮**不动哈希**，只修空值语义；两条判据集互不重叠。

---

## 2. 判据：改前 / 改后

**装置**：套件内 4 条永久判据，插在 `BindingFunction` 测试的 `// RawStruct` 段之后（`Script/Game/UnrealCSharpTest/UnitTest/Binding/BindingFunction/UnitTestSubsystem.cs:429-440`）。

| # | 判据名 | 断言（期望） | 改前 | 改后 |
|---|---|---|---|---|
| 1 | `BindingRawStructEqualsNonMatchingType` | `new FRawTestStruct{Value=0}.Equals("RawTestStruct") == false` | 🔴 **false**（实得 true） | 🟢 **true** |
| 2 | `BindingRawStructEqualsNull` | `new FRawTestStruct{Value=0}.Equals(null) == false` | 🔴 **false** | 🟢 **true** |
| 3 | `BindingRawStructEqualNullOperator` | `new FRawTestStruct{Value=0} == null` 为 `false` | 🔴 **false** | 🟢 **true** |
| 4 | `BindingRawStructNotEqualNullOperator` | `new FRawTestStruct{Value=0} != null` 为 `true` | 🔴 **false** | 🟢 **true** |

> 判据取 `Value = 0` 是**故意的**：0 句柄回退到零值，与 `Value=0` 的实例"相等"才会翻转；`Value=1` 的实例改前改后都返回 false（**假绿**），不能用作判据。

**读数（三态对照）**：

| 状态 | 行数 | 失败 | CSV SHA256 | 备注 |
|---|---|---|---|---|
| 基线（B-36 交付态，无新判据） | **1 959** | **0** | `BC3FD9E9B15764AFE08B3CBB6BB14057A1278D61745075C1361E2759C27E5F2D` | 与 B-36 报告记录的提交树 SHA **逐字节相同** |
| 改前（＋4 条判据，生成物未修） | **1 963** | **4** | `43C31E6951C6074E6C2ADC5E82FDE085274D6E56152F180FF539CECF7796143A` | 4 条判据全红 |
| 改后（第 1 次） | **1 963** | **0** | `150779588C88EE00DC0F5270B010FD9B0E25063A0FD14194A1A3F7D11E2B3870` | 4 条翻绿 |
| 改后（第 2 次） | **1 963** | **0** | 同上（**逐字节相同**） | 确定性 |

**CSV 差分**：`Compare-Object 基线 ↔ 改后` = **4 行**（全部是新增判据，其余 1 959 行逐条相同）；`Compare-Object 改前 ↔ 改后` = **8 行**（4 条 `<` + 4 条 `>`，其余逐条相同）⇒ **除 4 条判据外零回归**。

---

## 3. 施工

### 3.1 改动面

- **位置**：`Source/ScriptCodeGenerator/Private/FBindingClassGenerator.cpp`（`GeneratorPartial`），**＋35 行 / −0 行**。
- **触发条件**（四条同时成立才发守卫）：`Function.IsStatic()` ∧ `Params.Num() == 2` ∧ `FunctionParamName.Num() == 2` ∧ 函数名 ∈ {`operator ==`, `operator !=`} ∧ **两个参数的类型名都等于本类**。
- **形态**：把守卫体插在方法 `{` 与 `unsafe {` 之间（新增一个 `%s` 占位），与 `FStructGenerator.cpp:151-167` **逐字同形**：

```csharp
public static bool operator ==(FRawTestStruct InValue0, FRawTestStruct InValue1)
{
    if (InValue0 is null && InValue1 is null)
    {
        return true;
    }

    if (InValue0 is null || InValue1 is null)
    {
        return false;
    }

    unsafe { … 原样 … }
}
```

`!=` 的守卫是镜像（`null/null ⇒ false`、`单 null ⇒ true`）。

### 3.2 为什么不止修 `Equals`（范围说明）

`Equals` 的模板行**一字未改**（仍是 `this == Other as X`）—— 它由 `operator ==` 的守卫兜住：`Other as X` 得 null ⇒ `this == null` ⇒ 第二段守卫返回 false。

但同时修 `operator ==` / `!=` 本身，是因为**同一根因的第二个面**也被本轮判据抓到（判据 3/4）：`x == null` / `x != null` 在改前同样是错的（`==` 返回 true、`!=` 返回 false）。只修 `Equals` 会留下"同一个生成器、同一个漏判空，只修了一半"的形态；而守卫形态在 UStruct 路径已有**多年先例**，不是新开口。

### 3.3 产物对照

装置：`Saved/Retest/b37/manifest-{UE,Game}-{before,after}.txt`（`path|length|sha256`，按路径排序）。

| 侧 | 文件数 | 差异 |
|---|---|---|
| `Script/UE/Proxy` | 16 155 | **0 行**（逐字节相同） |
| `Script/Game/Proxy` | 75 | **2 行** = 同一路径的长度/SHA 变化：`FRawTestStruct.cs` 2 315 → **2 609** B；`7F736239…` → **`24F45964…`** |

> 生成方式是**真实编辑器会话**（`run-g13-generate.ps1` ＋ `pie_driver_g13gen.py`），**不是** `-run=GeneratorScriptCode` commandlet（后者会静默删除 107 个平台编辑器模块产物，见 [`08-…/11`](11-G13前段收口-转义与模板写盘-动态特性与UHT诊断（2026-09-24）.md)）。

---

## 4. 验证

| 项 | 读数 |
|---|---|
| 5.6 UBT | `UnrealCSharpTestEditor Win64 Development` **Result: Succeeded**（**26 actions / 52.86 s**；`[20/26] Link UnrealEditor-ScriptCodeGenerator.dll`、`[25/26] Link UnrealEditor-UnrealCSharpTest.dll` ⇒ 确实参编） |
| C# 构建 | `dotnet build Script/Script.sln -c Debug` **0 Warning / 0 Error**（生成期由编辑器自动编译产出 `Game.dll` 23:18:04，随后的手工构建为 up-to-date 空转） |
| 套件 | **1 963 / 0**（`B37-postfix` 与 `B37-postfix2` 两次，同 CSV SHA） |
| 套件日志 | `ensure/assert` **0**、`LogBlueprint: Error` **0**、`Unhandled Exception / Fatal error` **0**；`LogUnrealCSharp: Error` **3 条**，与基线/改前**逐字相同**（全部是判据故意触发的负向动态类：`BlueprintNativeEvent × 属性访问器`、`BlueprintGetter`、`SealedEvent × BlueprintImplementableEvent`） |
| 提交树复核 | `HEAD:Source/ScriptCodeGenerator/Private/FBindingClassGenerator.cpp` blob = `430483c2…` = 工作树 blob ⇒ **施工期读数即提交树读数**（无需重跑） |
| 补丁留档 | `Saved/Retest/b37/fix-B37-gen030-eba9958c.patch`（**2 078 B** / SHA256 `00067491EFFFF06FFBA754F3A1B47261603B72227D1368443D948536537C5600` / **LF** / 无 BOM / `git apply --reverse --check` **exit 0**） |

---

## 5. 未取得 / 边界（如实登记）

1. **只测 CoreCLR / Win64 / 编辑器 Development**：Mono / LeanCLR 未复跑。生成物与后端无关，但"套件在三后端一致"本轮未取得。
2. **覆盖面 = 两个参数都是本类的 `==` / `!=`**：混合类型（`T == OtherType`）的操作符不发守卫（当前语料 0 个）；若将来出现，需另判。
3. **用户侧 `BINDING_CLASS` 未实测**：本工程只有 `FRawTestStruct` 一个带 `operator ==` 的 binding class；"第三方工程重新生成后同样生效"是**模板层推断**（判据在生成器，不在具体类型）。
4. **未跑 5.0 / 5.4 / 5.8 实编**：`ScriptCodeGenerator` 是 Editor 模块，本轮只用到 `FString` / `TArray` / `Printf` / `GetName()` 等通用 API（无新增引擎接口）；§7 语言标准面全树 ＋ `-DiffBase e6ce0a3f` 均 **CLEAN**。触发条件 = 这些版本上出现与本改动相关的编译/运行异常。
5. **`GetHashCode() => 0` 未动**：属 `F-GEN-015`（已降级、不施工），与本章空值语义互不重叠。
6. **`Equals` 模板本身未改**：若将来某条生成路径只发 `Equals` 而不发 `operator ==`（当前由 `bHasEqualTo` 绑定，结构上不可能），守卫不会生效 —— 登记为形态前提。

---

## 6. 台账回写

| 台账 | 动作 |
|---|---|
| [`09-…/01`](../09-问题清单/01-全局问题总表.md) | `F-GEN-030` 行：状态 → **已修（`87dac201`）**；结论列补"后果更正 = 静默错值"＋判据/产物读数 |
| [`09-…/00`](../09-问题清单/00-推荐优先修复清单.md) | §0.0 新增"轮 B-37"更新注 ＋ **下一顺位重排**（T3 选片 / `F-DEF-010` 电池） |
| [`09-…/05`](../09-问题清单/05-已完成（P2与P3）.md) | 新增 **§38 轮 B-37 收口** |
| [`README.md`](../README.md) / [`INDEX.md`](../INDEX.md) | 轮次行 ＋ 报告条目 |

---

## 7. 复用价值

1. **"0 句柄 ⇒ 崩"必须先证明回退语义**：本轮三条静态事实（`GetHandle(null)=0` → `GetBinding(0)=nullptr` → `TPropertyValue` 回退 `static Default`）把"可能崩"改写成"静默错值"，直接决定了判据可以进套件、省掉独立会话装置。**任何"句柄 0 / 非法句柄"类缺陷都应先走这条链**。
2. **生成面缺陷要先做"生成器分叉清点"**：`Equals` 在 UStruct 与 BINDING_CLASS 两条生成器路径上形态不同（前者自带守卫、后者没有）⇒ 用"5 975 / 5 974 / 1"三个数字把缺陷面从"所有生成类"收窄到"一个模板分支"，避免按 grep 计数得出错误规模。
3. **判据要选"能翻转的输入"**：`Value = 0` 才与 0 句柄的 Default 相等；`Value = 1` 改前改后都 false（假绿）。**判据的输入必须落在缺陷的等价类里**。
4. **"修一半"要显式论证**：只修 `Equals` 也能让 4 条判据里的 2 条翻绿；本轮把 `==` / `!=` 一并守卫，是为了让"同一漏判空"不在生成器里留第二个面 —— 范围扩大有判据支撑（判据 3/4），不是顺手改。

---

## 8. 交付前格式与规范核查（§8 三轴）

> **装置**：`Saved/Retest/b30-style-lint.ps1 -Base e6ce0a3f -Target 87dac201`（提交对提交），完整输出 `Saved/Retest/b37/b37-style-lint-commit.log`。

### 8.1 机械读数

| 读数 | 值 |
|---|---|
| 变更文件 | **1**（`Source/ScriptCodeGenerator/Private/FBindingClassGenerator.cpp`） |
| Added / Indent | **35** / **TAB** |
| Trailing / SpaceFail / SpaceAlgn / TabInCs | **0 / 0 / 0 / 0** |
| OverWidth（>110）/ AddedMax | **0** / **86**（工具按"TAB=1 列"计；按 **TAB=8** 复算最大 **109** 列，仍 <110） |
| `git diff --check` | **exit 0** |
| AlignTS | 7（`TAB+空格` 对齐续行，本仓合法形，见 `TDefaultArgument.inl:38-44` 先例） |
| EOL / BOM | 与文件基线一致（CRLF / 无 BOM） |
| 新增注释 | **0**（工具报 "5 line(s)" 是**误报**：5 行是 `FString::Printf` 的 `*FunctionParamName[0]` 之类**以 `*` 开头的解引用实参**，被 `^\s*\*` 启发式当成块注释续行） |
| 结果 | **PASS** |

### 8.2 A 轴（项目规范/口径）

| 项 | 判定 | 证据 |
|---|---|---|
| ① G7 不新增注释 / 日志 / `throw` / `try-catch` / `ensure` / `check` | 🟢 | 新增 35 行内五类计数均 **0**（机械计数；注释误报已澄清） |
| ① G7 不新增公开 API | 🟢 | 只改生成器内部发射逻辑；插件 API 面 0 变化 |
| ② Q2 失败收敛到既有值域 | 🟢 | 失败路径 = **生成物里的 `false`/`true` 返回值**（.NET 契约要求的确定值），无新增日志、无异常 |
| ③ 新加逻辑 `if(条件)` ＋ 单一出口 | 🟢 | 生成器侧新增的 `if` 只做**赋值**（`FunctionGuardBody = …`），无提前 `return`/`continue`；**生成物侧**的守卫是 `FStructGenerator` 既有形态（两段 `if` + `return`，是**生成代码**不是手写交付代码） |
| ③ 局部变量直接声明 | 🟢 | `FString FunctionGuardBody;`（与同文件 `FString InBufferBody;` 同形）；`const auto& FunctionName` / `const auto bIsEqual` |
| ④ 新增接口版本兼容 | 🟢（不触发） | 无新增引擎接口调用；§7 语言标准面 CLEAN |
| ⑤ 命名与既有用语一致 | 🟢 | `FunctionGuardBody` ↔ `InBufferBody`/`OutBufferBody`/`ReturnBufferBody`；`bIsEqual` ↔ `bHasInBuffer`/`bHasOutBuffer` |

### 8.3 B 轴 / C 轴

- **B**：见 8.1；diff 只含语义改动（无重排、无无关行）。
- **C**：仓内 `.clang-format` / `.clang-tidy` / `.editorconfig` / `.gitattributes` **均不存在** ⇒ **不跑格式化器**、只做对照式核查（与 B-22/B-23/B-33/B-36 同判）。

### 8.4 §7 语言标准面

`b29-xversion-lint.ps1 -Mode Syntax`：**全树**（514 文件，排除 ThirdParty）**CLEAN**；`-DiffBase e6ce0a3f` **CLEAN**（新增行无 C++20-only 构造；1 处 `#if STD_CPP_20` 内命中按规则跳过）。
