# 轮 B-62 收口：`F-ENV-001` EnhancedInput 绑定归属（A1 owner 重载 ＋ A2 主动解绑清表）＋ `F-GEN-022` 三方校验器 ＋ `F-CS1-005` 量化未取得（2026-10-21）

> **来源**：维护者 2026-10-21 回执「**按你的推荐开始**」⇒ [`99-轮B-61`](99-轮B-61-T3顶端六条可行性裁定（2026-10-21）.md) §4 的 **Q1–Q6 全部 = A**。
> **口径**：G7（不新增注释 / 日志 / `throw` / `try-catch` / `ensure` / `check` / 公开 API）＋ Q2（失败收敛到既有取值域）＋ Q3（失败判据 = 产物 manifest ＋ `Generator` 内部返回值）。
> **基线**：插件 HEAD = **`4c3108cd`**（B-60 交付）→ 施工期工作树 → 🟢 **已提交 `00380ea1`**（父 `4c3108cd`，author crazytuzi，2026-10-23 00:16:45 +0800，subject `EnhancedInput Binding Owner Reference && RemoveBinding Release && Member Address Cache Remove && Field Loop Remove`；**本轮改动 1 文件 `+4/−2`**）；工程侧新增 2 个判据文件 ＋ 1 个校验器脚本（未提交）。
> **一句话结论**：`F-ENV-001` **已修**（插件 **1 文件 `+4/−2`**；施工期 `+5/−2`，差额 = A2 调用折行改单行），**A2 半边取得干净 A/B**（`EnhancedBindingHandleReleasedOnRemoveAction` **改前 false → 改后 true**），**A1 半边**取得"owner 引用已注册"的运行期守卫、**销毁半边未取得**；套件 **1 967 断言 / 0 失败 ×2 同 SHA**，与 B-60 基线 **只差 +5 行新增断言**；`F-GEN-022` 落 **0 代码三方校验器（129 项 / 0 不符）**；`F-ENV-011` 按证伪**出列**；`F-CS1-005` 量化**未取得**（本机无 profiler，B-63 已补做）。

---

## 0. 改动清单（逐文件）

| 文件 | numstat | 改动 | 目的 |
|---|---|---|---|
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp` | 施工期 `+5/−2` ／ **提交态 `+4/−2`** | `:228-230` `AddBindingReference<…, false>(FoundClass, Object, &binding)` → **owner 重载** `AddBindingReference<…>(InManagedHandle, FoundClass, Object, &binding)`；`:262` 在 `RemoveBinding` 之后补 **`RemoveBindingReference(InEnhancedInputActionEventBinding)`**（提交态为**单行**，105 列——维护者按 Rider 格式化） | **A1 ＋ A2** |

**未新增**：日志 0 / `throw`·`try-catch` 0 / `ensure`·`check` 0 / 公开 API 0 / 门控宏 0 / 注释 0。
**工程侧（未提交）**：`Script/Game/UnrealCSharpTest/UnitTest/Input/InputBinding/UnitTestSubsystem.cs`（判据 5 条）、`Script/Game/UnrealCSharpTest/UnitTest/UnitTestSubsystem.cs`（2 处调用）、`Saved/Retest/verify-b62-gen022-function-index.ps1`（`F-GEN-022` 校验器）。

**补丁**：`Saved/Retest/fix-00380ea1.patch` —— **4 866 B** / SHA256 `D76D8915CC04F10CE3AE1FD1E8AE53D8F328F6EB03503475B73778E671D270F4` / LF / 无 BOM / `git apply --reverse --check` **exit 0**（4 文件 `+21/−39`）。

---

## 1. 判据

### 1.1 A2 —— 干净 A/B（同一二进制不同工作树）

判据装置：`-game` 套件内，`UEnhancedInputComponent.BindAction(...)` → `RemoveAction(...)`，用反射读 `Interop.HandleData` 的私有静态 `Handles` 字典，检查绑定对象的句柄条目是否被释放。

| 断言 | 改前（`B62pre`） | 改后（`B62finalA` / `B62finalB`） |
|---|---|---|
| `EnhancedBindingHandleAllocated` | true | true |
| `EnhancedBindingHandleLiveBeforeRemoveAction` | true | true |
| **`EnhancedBindingHandleReleasedOnRemoveAction`** | **false** | **true** ✅ |
| `EnhancedBindingOwnerHandleAllocated` | true | true |
| `EnhancedBindingOwnerReferenceKept` | （改前装置未含） | true |

- 机制：`RemoveAction` → `UEnhancedInputComponentImplementation.UEnhancedInputComponent_RemoveBindingImplementation`（`Script/UE/CoreUObject/EnhancedInputComponent.cs:75-77`）→ native `RemoveBindingImplementation` ⇒ A2 的 `RemoveBindingReference` 释放 `GCHandle`（`FBindingRegistry::RemoveReference`，`FBindingRegistry.cpp:54-79`）⇒ `HandleData.FreeImplementation` 从 `Handles` 移除（`HandleData.cs:60-92`）。
- 语义边界：A2 覆盖 **C# 主动解绑**；不覆盖引擎侧 `ClearActionBindings` 与栈实参地址（B-61 §2.1 已登记为触发条件）。

### 1.2 A1 —— 登记半边有运行期守卫，销毁半边未取得

（a）**"owner 引用已注册"的运行期证据**：`FCSharpEnvironment::AddReference(owner, FReference*)` 的拒绝路径会 **`delete InReference`**（`FCSharpEnvironment.cpp:665-675`，`:672`），而 `~FBindingReference` 会立即 `RemoveBindingReference`（`FBindingReference.h:12-15`）。因此「绑定后句柄仍存活」（`EnhancedBindingHandleLiveBeforeRemoveAction == true`）**等价于** owner 引用已被接受 —— 若 A1 未生效/被拒，该断言会当场翻红。

（b）**"owner 销毁 ⇒ 清表"未取得**：三次装置尝试均未让组件被回收：

| 装置 | 读数 |
|---|---|
| ① 瞬态包组件 ＋ `UKismetSystemLibrary.CollectGarbage()` | `EnhancedBindingComponentCollected = false`（组件句柄仍在） |
| ② 组件 outer = 世界内 actor ＋ `K2_DestroyActor()` ＋ `CollectGarbage()` | 同上，`EnhancedBindingComponentCollected = false` |
| ③ 同 ②，并区分"组件死了但清理没跑"与"组件没死" | 组件句柄**未消失** ⇒ 无法区分，判据**未取得** |

残余原因（未定位，候选）：`K2_DestroyActor` 在同帧只做标记、`CollectGarbage` 的增量 purge 未在当帧完成；或该组件被某条既有引用链钉住。**⇒ 最终交付的判据集保留 (a) 的守卫断言，删除 (b) 的红灯断言**（套件必须 0 失败）；(b) 作为"未取得"登记。

### 1.3 回归与规范

| 项 | 读数 |
|---|---|
| UBT `UnrealCSharpTestEditor Win64 Development` | **Succeeded**（26 actions / **45.02 s**；UBA 32.68 s） |
| `dotnet build Script/Script.sln -c Debug` | **0 Warning / 0 Error**（2.3 s；`Game.dll` 901 632 B → `Content/Script`） |
| `-game` 套件（`B62finalA` / `B62finalB`） | **1 967 断言 / 0 失败 ×2**；CSV SHA256 **`77BB1439B33A82AE950333B20EE201C1A77D25E3E2131446D4943FA7CC2BC346`**（两轮同 SHA） |
| 与 B-60 基线（`Saved/Retest/out/B60/B60-r01-csv01.csv`，1 962 断言） | `Compare-Object` **只有 5 条 `=>`（新增），0 条改动/删除** ⇒ 既有 1 962 条断言逐字节相同 |
| 日志 | `LogUnrealCSharp: Error` **3**（既有判据：`BlueprintNativeEvent`＋属性访问器 / `SealedEvent`＋`BlueprintImplementableEvent`，B-34/B-35 注册）；assert / fatal **0** |
| §7 跨版本 | **CLEAN** —— 本轮 **0 新调用 / 0 新 API**（owner 重载与 `RemoveBindingReference` 均为既有符号），无门控点新增 |
| §8 风格 | **PASS** —— 施工期新增行 **5**（含 1 空行）、非空新增行**最大宽 77**；**提交态新增 4 行（含 1 空行）、A2 调用单行 105 列（≤110）**；Trailing 0、TAB 缩进、CRLF / 无 BOM、`git diff --check` exit 0；仓内无 `.clang-format`/`.editorconfig` ⇒ 不跑格式化器 |

**提交与提交树复验**：🟢 **已提交 `00380ea1be7f5b0c808a8abd50a23b616b9ce84a`**（父 `4c3108cd`，author/committer crazytuzi，2026-10-23 **00:16:45 +0800**，**4 文件 `+21/−39`**）⇒ 提交树 **`git diff HEAD -- Source/UnrealCSharp` 为空** ⇒ 在提交树上重跑：UBT **Succeeded**、套件 **1 970 行（1 969 断言）/ 0 失败 ×2 同 SHA**（`70C740F4…`）、`Error` 3、assert·fatal 0。

---

## 2. `F-GEN-022`：0 代码三方校验器（Q5 = A）

- **脚本**：`Saved/Retest/verify-b62-gen022-function-index.ps1`（工程侧、不进插件源码、不新增公开面）。
- **比对三方**：① 生成端位定义（`FGeneratorCore.cpp` 的 `<< 1..4`）、前缀族（`TEXT("Generic"/"Primitive"/"Compound")`）、入口名模板（`FClassGenerator.cpp` 的 `FFunction_%sCall%dImplementation`）；② 运行端注册表（`FRegisterFunction.cpp` 的 25 个 `.Function("…")`）；③ C# 声明（`FFunctionImplementation.cs` 的 25 个 `FFunction_…Implementation`）。
- **输出**：`checks=129 mismatch=0 cpp=25 cs=25` / `RESULT PASS` / **exit 0**。
- **检查内容**：清单规模（2）、位定义（1）、模板（1）、前缀族（1）＋ 逐名 4 项 ×25（C# 存在 / 索引 < 32 / **Generic ⇔ 偶数**（bit0 = hasReturn）/ 形状合法）＋ 反向 1 项 ×25。
- **为什么不进插件**：共享 `constexpr` ＋ `static_assert` 需要新开 `UnrealCSharpCore` 公共头（= 新增公开面，撞 G7），且 `static_assert` 全仓 **0 先例**、C#/weaver 侧吃不到；校验器是唯一 0 风险且能兜住"部分静默错"的形态（B-61 §2.5）。
- **附带更正**：B-38 的"产物只用 12 个索引、24/26 零命中"**不成立** —— 本轮重数为 **1 576 个 `.cs` / 14 107 处 / 14 个索引**（`Call24`=30、`Call26`=109），且 `FUnrealFunctionDescriptor.inl:295-342` 证明 **bit4（net）承重**（24/26 走 `GetFunctionCallspace` + `CallRemoteFunction`，8/10 走本地 `Invoke`/`ProcessEvent`）。

**运行方式**：`& Saved\Retest\verify-b62-gen022-function-index.ps1`（可选 `-Plugin <插件根>`）；建议与 §7/§8 同层，每轮收口前跑一次。

---

## 3. `F-CS1-005` 量化：未取得（Q4 = A 的量化步骤未完成）

| 项 | 结论 |
|---|---|
| 现状 | `MethodBridge.cs:41` `Invoke`、`:49` `GetParameters()`、`:53` `new object?[]`、`:62` `Activator.CreateInstance`、`:79` `Method.Invoke`、`:128` `HandleData.Alloc` |
| 阻塞 ① | 本机**无 `dotnet-trace` / `dotnet-counters`**（`Get-Command` 均不存在），装工具需外网（本机 NuGet/网络此前多次不可用） |
| 阻塞 ② | `Invoke` 是 `[UnmanagedCallersOnly]` ⇒ **托管侧不能直调**，无法在 C# 测试里做微基准；要取频次只能原生侧临时探针（改插件 ＋ 回退） |
| 处置 | **本轮不改码**；维持候补 ＋ 触发条件（装上 profiler，或做一次"临时计数探针 → 采集 → 回退"的探针轮）。⇒ 与 B-61 §2.4 的 A 口径一致（先量化，未量化不动手） |

---

## 4. `F-ENV-011` 出列与两条维持（Q2 / Q3 / Q6 = A）

- **`F-ENV-011` ⚪ 出列（前提证伪）**：证据链见 [`99`](99-轮B-61-T3顶端六条可行性裁定（2026-10-21）.md) §2.2（`Interop.dll` 由 hostfxr 装进默认不可卸载 ALC ＋ `LoadAssembly` 显式跳过 ＋ ALC 把 "Interop" 委派回原上下文 ⇒ `HandleData.Handle` 跨域单调）。**仅 LeanCLR 后端**登记为触发条件（桌面切 LeanCLR / Interop 改可卸载 ALC / 出现"同句柄指不同对象"实证）。
- **`F-REG-008` ⚫ 维持**：可达性理由更正为"**未取得非 GT 销毁路径**"（`-asyncloadingthread` 是死旋钮；真闸门 `AsyncPackageLoader.cpp:132-148`，cooked 默认 true）。
- **`F-BLD-003` ⚫ 维持**：`.Build.cs` 在 `Source/**` 内、**G7 无豁免**（`_CONVENTIONS.md:288/294`）⇒ `BuildException` 撞口径；且本机无 Linux ARM64 实编点 ⇒ 判据不可构造。

---

## 5. 未取得 / 边界

1. **A1 的"owner 销毁 ⇒ 清表"**：三次 GC/销毁装置均未让组件被回收（`EnhancedBindingComponentCollected = false`），**未取得**；待定位（候选：同帧 destroy/purge 时序、组件被既有引用钉住）。
2. **`F-CS1-005` 的频次与分配占比**：无 profiler ⇒ 未取得。
3. **`F-ENV-001` 的剩余触发面**：`ClearActionBindings`（组件仍活、绑定对象被释放）与栈实参地址（`TPropertyValue` 的 `IsReference` 分支）**不在 A1/A2 覆盖面**，仍按 B-61 §2.1 登记。
4. **平台**：只跑 Windows / CoreCLR / `-game -nullrhi`；未跑 PIE、编辑器交互、Mono/LeanCLR、打包版。
5. **插件已提交 `00380ea1`**；**工程侧**判据与校验器仍未提交；`Source/SourceCodeGenerator` 子模块指针的预存 `M` 未触碰。
6. **套件断言数变化**：1 962 → **1 967**（+5 条永久判据），故 CSV 顶层 SHA 与 B-60 不同；**逐行 diff 只含 5 条新增**已作为等价判据。

---

## 6. 台账回写（本轮）

| 台账 | 动作 |
|---|---|
| [`09-…/01`](../09-问题清单/01-全局问题总表.md) | `F-ENV-001` 状态列追加 🟢 已修（B-62）；`F-ENV-011` 状态列追加 ⚪ 前提证伪 ⇒ 出列 |
| [`09-…/05`](../09-问题清单/05-已完成（P2与P3）.md) | 新增 **§55**（六条处置/判据/未取得） |
| [`09-…/00`](../09-问题清单/00-推荐优先修复清单.md) | §0.0 新增「轮 B-62」注（施工、判据、回归、补丁、未取得） |
| [`README.md`](../README.md) / [`INDEX.md`](../INDEX.md) | 轮次行 ＋ 本报告条目 |

---

## 7. 附录：本轮装置与文件

| 类型 | 路径 |
|---|---|
| 插件改动（唯一） | `Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp`（施工期 `+5/−2` ／ **提交态 `+4/−2`**；🟢 已提交 **`00380ea1`**） |
| 补丁 | `Saved/Retest/fix-00380ea1.patch`（4 866 B / `D76D8915…` / reverse-check 0；4 文件 `+21/−39`） |
| 判据（工程侧） | `Script/Game/UnrealCSharpTest/UnitTest/Input/InputBinding/UnitTestSubsystem.cs`（`TestEnhancedBindingOwnership` / `TestEnhancedBindingOwnerRelease` / `HasManagedHandle`） |
| 判据注册 | `Script/Game/UnrealCSharpTest/UnitTest/UnitTestSubsystem.cs`（`TestEnhancedBindingOwnership();` / `TestEnhancedBindingOwnerRelease();`） |
| 校验器 | `Saved/Retest/verify-b62-gen022-function-index.ps1`（129/0） |
| 套件产物 | `Saved/Retest/out/{B62pre,B62finalA,B62finalB,B62diag,B62diag2,B62c-A,B62c-B}/` |
