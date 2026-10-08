# G12「尾项」收口轮：非 BPGC 类的输入绑定通道 —— 六个 `Bind*` 改「条目结构体直传」+ `ClearBindingValues` 去参（2026-09-30）

> **分析范围**：[`09-问题清单/03-已完成（P0与P1）.md`](../09-问题清单/03-已完成（P0与P1）.md) **§2.20** 行末新登记的 **`F-CS2B-032`**（非 BPGC（向导「C++」型）类上六个 `Bind*`/`Remove*` 全程静默 no-op），顺带 **`F-CS2B-025`**（`ClearBindingValues(UObject InObject)` 死参数）。
>
> **覆盖文件**（行号 = 本轮读到的实际内容；插件仓 HEAD = `152b5681`）：
> **原生** —— `Source/UnrealCSharp/Private/Domain/Interop/FRegisterInputComponent.cpp`（改前全文 681 行）、`FRegisterEnhancedInputComponent.cpp`（全文 236 行）；
> **C#** —— `Script/UE/CoreUObject/InputComponent.cs`（改前全文 495 行）、`Script/UE/CoreUObject/EnhancedInputComponent.cs`（全文 84 行）、`Script/UE/Library/InputComponentImplementation.cs`（全文 134 行）、`Script/UE/Library/EnhancedInputComponentImplementation.cs`（`:1-30`）；
> **生成器** —— `Script/SourceGenerator/UnrealTypeSourceGenerator.cs`（`:45-72`、`:450-500`、`:682-700`）；
> **编辑器** —— `Source/UnrealCSharpEditor/Private/NewClass/SDynamicNewClassDialog.cpp`（`:610-684`、`:276-296`）、`DynamicNewClassUtils.cpp`（`:60-110`）；
> **动态类** —— `Source/UnrealCSharpCore/Private/Dynamic/FDynamicClassGenerator.cpp`（`:196-265`、`:845-855`）；
> **模板** —— `Template/Dynamic/{DynamicActor,DynamicActorComponent,DynamicObject,DynamicUserWidget}.cs`。
>
> **引擎侧**（UE 5.6，`D:\file\UnrealEngine\5.6\Engine\Source\`）：`Runtime/Engine/Private/InputDelegateBinding.cpp`（全文 92 行）、`Runtime/Engine/Classes/Engine/InputDelegateBinding.h`（`:38-70`）、`Runtime/Engine/Private/BlueprintGeneratedClass.cpp`（`:1575-1660`）、`Runtime/Engine/Classes/Engine/BlueprintGeneratedClass.h`（`:464`）、`Runtime/Engine/Private/Components/InputComponent.cpp`（`:365-395`）、`Runtime/CoreUObject/Public/UObject/Class.h`（`:3258-3287`）、`Runtime/CoreUObject/Private/UObject/Class.cpp`（`:4818-4830`）、`Runtime/Engine/Classes/Engine/InputActionDelegateBinding.h`（`:45`）、`Plugins/EnhancedInput/Source/EnhancedInput/Public/EnhancedInputComponent.h`（`:355-470`）。
>
> **状态**：**已完成**（含施工 + 三态验证：改前 **5 失败** → 改后 **0 失败** → 交付态复跑**逐条 diff 0** → 格式核查后第四态 **1816 / 0**）；🟢 **插件侧已提交 `152b56818cc6249b5487b5d4dc31ee3c287f5fc1`**（父 `77526cd2`，作者 crazytuzi，2026-09-30 14:39:01 +0800，subject `Set Enumerator && Helper Deinitialize && Lazy Rehash && Input Bind Struct`，**5 文件 `+167/−213`**；**提交树 == 已验证工作区**（5/5 blob 相等）、**提交级补丁与施工期补丁 SHA256 相同** ⇒ 无非语义微调，见 §9.1）；⚠️ **工程侧 1 新文件 + 1 个测试方法（6 条断言）+ 1 行接线 + 1 行 `using` 仍未提交**。
>
> **本文不新造 `F-*` 锚点**（除 §6 的两条显式新登记）：所有既有编号一律写成「编号 + 出处报告」。

---

## 0. 结论速览（先读这一节）

| # | 结论 | 依据 |
|---|---|---|
| **1** | **`F-CS2B-032` 的运行期坐实与修复**：非 `_C`（`A*`/`U*` 前缀，即向导「C++」型）类的动态 UClass **不是** `UBlueprintGeneratedClass` ⇒ 引擎 `GetDynamicBindingObject` 恒空 ⇒ 改前六个 `Bind*` **整段被跳过**。改后改走「**条目结构体直传**」，绑定落在当前输入组件上 | 改前对照 **5 条断言 false**（§5）、改后 **true** |
| **2** | 🔴 **登记范围偏窄（回核扩展）**：`F-CS2B-032` 的载体是 **2 个原生文件**，不是 1 个 —— `FRegisterEnhancedInputComponent.cpp:43-66` 有**逐字同型体**，报告 24 §5① 未登记；C# 侧静默点共 **14 处**（legacy 12 + EnhancedInput 2） | §3.2 |
| **3** | 🔴 **影响面比登记时大一个量级**：插件**自带的 4 个用户模板全部是非 `_C` 命名**（`ADynamicActor`/`UDynamicActorComponent`/`UDynamicObject`/`UDynamicUserWidget`），向导选原生父类时**默认**就是非 `_C` 型（`NewClassTypeIndex = 0` =「C++」）⇒ 缺陷落在**插件自己的默认流程**上 | §3.1、E9–E11 |
| **4** | 🔴 **"存活语义"这个问题不存在**：`BindInputDelegates` 只从 `BPGC->DynamicBindingObjects` 取配方（E4）⇒ **插件侧任何方案都不会被引擎重放**。故维护者原本要裁定的"无配方时是否跨输入组件重建存活"**不是取舍，而是不可实现**；三条候选路线因此只剩工程成本之争，裁定 **B′** | §1、E3/E4 |
| **5** | 🔴 **`F-CS2B-025` 的判据被更正**：引擎 `UInputComponent::ClearBindingValues()`（`InputComponent.cpp:371-389`）**只把 Axis/AxisKey/VectorAxis/Gesture 的当前值清零**，与任何绑定条目无关 ⇒ 原登记的「疑似丢失语义」**不成立**，`InObject` 是纯多余参数（判定成立、语义丢失证伪） | E13 |
| **6** | ✅ **两条"上一轮修复的漏面"已并入并收口**：① `EnhancedInputComponent.cs:81` 仍调 `RemoveFunction`（上一轮 Q2(a)「不再删 UFunction」的漏面）；② `FRegisterEnhancedInputComponent.cpp` 的 `BindFunction` 命中分支缺 `F-CS2B-018` 的配对 `RemoveFromRoot()` | §4.2 |
| **7** | 🆕 **本轮改动新暴露一条面（已登记 `F-CS2B-035`）**：EnhancedInput 族**没有** legacy 那套原生幂等扫描 ⇒ 非 `_C` 类上重复 `BindAction` 会**累加**引擎绑定（同一输入触发多次回调）。legacy 六族因有 `Has*Binding` 判据而幂等（本轮用例已坐实） | §3.3、§6 |
| **8** | 🪤 **装置实录（给下一轮）**：MSBuild 复用节点两次锁死 `Script/UE/obj/Debug/net10.0/UE.dll`（`CS2012`）⇒ 改前/改后切换后**必须** `dotnet build-server shutdown` 或 `-nodeReuse:false`；两次都是靠 **"产物 mtime > 源 mtime"断言**才发现"新代码其实没编进去" | §5.4 |

---

## 1. 前提回核与裁定

### 1.1 维护者裁定（本轮开工前）

| # | 问题 | 裁定 |
|---|---|---|
| **Q1** | 采用哪条修法路线 | **B′** —— 改**既有**六个导出的参数形态（「配方句柄 + 索引」→「条目结构体句柄」），**导出数量不变** |
| **Q2** | `F-CS2B-025` 怎么处理 | **删掉死参数**（`ClearBindingValues()`） |
| **Q3** | 两条"上一轮修复的漏面"是否并入 | **两条都并入本轮** |

### 1.2 为什么 Q1 的"存活语义"不再是选择题（本轮最重要的前提产出）

登记 `F-CS2B-032` 时留下的话是："修它需先裁定**无配方时绑定是否要跨输入组件重建存活**"。本轮回核把这个问题**证否**了：

- **引擎的"意图"是支持所有 UClass**：`UInputDelegateBinding::SupportsInputDelegate` 在 5.6 已标 `UE_DEPRECATED(5.6, "All UClasses will be considered valid for input delegate support.")`，实现是 `return bAlwaysAllowInputDelegateBindings || Cast<UBlueprintGeneratedClass>(InClass);`，而 CVar `Input.bAlwaysAllowInputDelegateBindings` **默认 true**，注释写明"即使是 native UClass，也可能有需要动态绑定的蓝图子对象/组件"（`InputDelegateBinding.h:48-49`、`InputDelegateBinding.cpp:9-12,29-35`）。
- **但"存储与遍历"仍锁在 BPGC**：重放入口 `BindInputDelegates` 取配方用的是 `UBlueprintGeneratedClass::GetDynamicBindingObject(InClass, BindingClass)`（`InputDelegateBinding.cpp:55`），而该数组是 `UBlueprintGeneratedClass::DynamicBindingObjects`（`BlueprintGeneratedClass.h:464`，遍历点 `BlueprintGeneratedClass.cpp:1583/1607/1634/1651`）。
- ⇒ **插件侧自建任何存储都不会被引擎重放**（要重放只能改引擎源码或把类做成 BPGC）。因此 A（原生侧配方存储）/ B′（直绑）/ D（C# 瞬态配方）三条路线在**语义上等价**（都只给"绑在当前输入组件上"），差别纯在工程成本与生命周期风险。

> ⚠️ **同一事实的另一个推论**：`_C` 类（BPGC）的"跨输入组件重建存活"来自**引擎重放**，不是插件行为；插件对非 `_C` 类**永远给不了同等语义**。这一条必须与"已修"并读。

### 1.3 回核扩展的三条（登记之外）

| # | 新发现 | 证据 | 本轮处置 |
|---|---|---|---|
| ① | `FRegisterEnhancedInputComponent.cpp:43-66` 的 `GetDynamicBindingObjectImplementation` 与 legacy 版**逐字同型**（同 `Cast<UBlueprintGeneratedClass>` → 同 `InvalidManagedHandle`）⇒ 增强输入在非 `_C` 类上**同样全程静默** | 两文件对照（legacy `:17-40` HEAD / enhanced `:43-66`） | **已修**（C# 侧 1 处分支，见 §4.2） |
| ② | `EnhancedInputComponent.cs:81` 仍调 `InObject.GetClass().RemoveFunction(InAction.Method.Name)` —— 上一轮 Q2(a) 裁定「**不再删 UFunction**」只落到了 `InputComponent.cs` 的六个 `Remove*` | HEAD `EnhancedInputComponent.cs:77-81`；上一轮 [`08-…/24`](24-G12剩余两条收口-输入绑定解绑语义与身份键（2026-09-29）.md) §2 Q2、§3 文件表 | **已修**（删该行，与 legacy 对齐） |
| ③ | `FRegisterEnhancedInputComponent.cpp` 的 `BindFunction` 命中分支**缺** legacy 版 :609 的 `FoundFunction->RemoveFromRoot()`（`F-CS2B-018` 的修复只落了一个文件） | HEAD enhanced `:76-79`（`return;` 无释放）↔ legacy `:607-612` | **已修**（补配对，改后 enhanced `:78`） |

---

## 2. 引擎与生成器契约（本轮取证 E1–E14）

| # | 契约 | 出处 |
|---|---|---|
| **E1** | 动态类是否 BPGC **只由类名决定**：`IsDynamicBlueprintGeneratedClass(const FString&) = InName.EndsWith(TEXT("_C"))` | `FDynamicClassGenerator.cpp:851-854`；二选一分支 `:208-223` |
| **E2** | `GetDynamicBindingObject` 是 `UBlueprintGeneratedClass` 的**静态成员且内部 Cast** ⇒ 非 BPGC 必然拿不到配方（插件随即回 `InvalidManagedHandle`） | 插件 HEAD `FRegisterInputComponent.cpp:20-21`、`:39`；引擎 `InputDelegateBinding.cpp:55` |
| **E3** | 引擎 5.6 已把"动态输入委托绑定"的适用面扩到**所有 UClass**（`SupportsInputDelegate` 弃用 + CVar 默认恒真） | `InputDelegateBinding.h:48-49`、`InputDelegateBinding.cpp:9-12,29-35` |
| **E4** | **重放只认 BPGC**：`BindInputDelegates` 逐个 `InputBindingClasses` 调 `UBlueprintGeneratedClass::GetDynamicBindingObject` | `InputDelegateBinding.cpp:52-61`；`BlueprintGeneratedClass.h:464`、`.cpp:1583/1607/1634/1651` |
| **E5** | 重放由**引擎侧**触发（插件无法替代）：`Actor.cpp:4834/4855`、`Pawn.cpp:501`、`PlayerController.cpp:2644`、`ActorComponent.cpp:1957`、`LevelScriptActor.cpp:59`、`UserWidget.cpp:1757` | 引擎全树 grep `BindInputDelegates` |
| **E6** | 插件在绑定时**现场造 UFunction**：`NewObject<UFunction>(InClass, …)` + `FUNC_BlueprintEvent` + `AddCppProperty` + `Bind()` + `StaticLink(true)` + `AddFunctionToFunctionMap` + `AddToRoot()` | legacy HEAD `:599-637`；改后 `:574-612` |
| **E7** | `UClass::AddFunctionToFunctionMap` / `RemoveFunctionFromFunctionMap` **无 `WITH_EDITOR` 门控、无断言**（仅加写锁 + 清 `AllFunctionsCache`）⇒ 在**普通 UClass** 上挂 UFunction 成立 | `Class.h:3260-3271`、`:3276-3287` |
| **E8** | `UClass::AddReferencedObjects` 显式把 `FuncMap` 的值加入 GC 稳定引用 ⇒ **UFunction 不需要 `AddToRoot` 也活着** | `Class.cpp:4823-4826` |
| **E9** | `[UClass]` 命名是**硬约束**（`UC_ERROR_03`）：`_C` 结尾 **或** 与基类前缀同形（`A*`/`U*`）；文件名为「非 `_C` ⇒ 去掉首字母」；引擎名由 `GetEngineName` 去首字母 | `UnrealTypeSourceGenerator.cs:457-494`、`:686-695`、`:64-67` |
| **E10** | 向导的"Dynamic Class Type"只有两项：**「C++」(index 0)** 与 **「Blueprint」(index 1)**；`_C` 由类型决定（切换时增删），**默认类型 = 父类名是否 `_C`** ⇒ 选原生父类默认落到「C++」型（非 `_C`） | `SDynamicNewClassDialog.cpp:293-296`、`:628-636`、`:655-658` |
| **E11** | 插件自带 4 个用户模板**全部非 `_C`**：`ADynamicActor` / `UDynamicActorComponent` / `UDynamicObject` / `UDynamicUserWidget` | `Template/Dynamic/*.cs` |
| **E12** | 引擎增强输入的 `BindAction(Action, TriggerEvent, Object, FName)` **无签名校验、无条件入队**（`Add_GetRef`）⇒ 瞬时 `UInputAction` 可绑定（本轮用例据此成立） | `EnhancedInputComponent.h:463-469` |
| **E13** | `UInputComponent::ClearBindingValues()` **只把四个值型数组的当前值清零**（Axis / AxisKey / VectorAxis / Gesture），不触碰任何绑定条目 | `InputComponent.cpp:371-389` |
| **E14** | 增强输入的绑定数组 `EnhancedActionEventBindings` 是 **protected**，但有 **public 只读访问器** `GetActionEventBindings()`；绑定条目**无函数名访问器** | `EnhancedInputComponent.h:362`、`:410` |

---

## 3. 影响面

### 3.1 谁落在"非 `_C`"这一半？

| 证据 | 数值 / 事实 |
|---|---|
| 工程侧 C# 游戏代码里的 `[UClass]` | **11 个** = `Xxx_C`（BPGC 路径）**5** + `AXxx`（非 BPGC）**6** + `UXxx` **0** |
| 6 个 `AXxx` 明细 | `ATestRawDynamicFunctionActor`、`ATestRawDynamicPropertyActor`、`ATestRawDynamicRpcActor`、`ATestBlueprintRawDynamicFunctionActor`、`ATestBlueprintRawDynamicPropertyActor` 5 个既有 + 本轮新增 `ATestInputBindingActor` |
| **插件自带模板** | **4/4 全部非 `_C`**（E11）⇒ 用户"新建动态类"照模板生成的第一版代码就在这一半上 |
| **向导默认** | 选原生父类（`AActor`/`ACharacter`/`UActorComponent`…）⇒ `NewClassTypeIndex = 0`（「C++」）⇒ 非 `_C`（E9/E10） |
| C# 侧静默点 | **14 处**（legacy 12 = 6 `Bind*` + 6 `Remove*`；EnhancedInput 2）——形态统一为 `if (X != null) { …全部行为… }`，**无 else、无日志、无异常、无返回值差异** |
| 仓内实际调用点 | 只有测试：上一轮 6 条断言 + 本轮 6 条（插件自身 `Script/**` 零调用、模板零调用）⇒ 缺陷**用户可达、仓内仅测试可达** |

> **口径提醒**：`F-CS2B-032` 的"静默"不是"日志缺失"而是"**整段行为不存在**"。上一轮之所以能发现它，正是因为第一次测试用了 `[UClass] ATestInputBindingActor` 这种非 `_C` 宿主、6 条断言全为"计数 0"。

### 3.2 `Remove*` 的真实形态（与登记文字并读的更正）

六个 `Remove*` **不是**"全程 no-op"：上一轮已把原生 `Unbind*` 调用挪到 `if (recipe != null)` **之外**（`InputComponent.cs:315-327` 等），所以非 `_C` 类上 `Remove*` 的**原生解绑仍然执行**，只是"配方条目清理"这一段被跳过（本就无配方可清）。⇒ 登记文字宜读作"**`Bind*` 全程 no-op；`Remove*` 只少了配方清理**"。

### 3.3 本轮改动新暴露的面（→ §6 `F-CS2B-035`）

把"非 `_C` 类也能绑"打开之后，**增强输入族缺少幂等判据**这件事就露出来了：

| 族 | 幂等判据 | 出处 |
|---|---|---|
| legacy 六族 | ✅ 原生**扫描活绑定**：`HasActionBinding` / `HasBinding<…>` / `HasKeyBinding` | 改后 `FRegisterInputComponent.cpp:80-106`，各 `Bind*` 内 `if (!Has…)` |
| EnhancedInput 族 | ❌ **无**：`BindActionImplementation` 直接 `FoundObject->BindAction(…)`（无条件入队，E12） | `FRegisterEnhancedInputComponent.cpp:190-195` |

`_C` 类之所以看不出这个问题，是因为 **C# 配方去重**先返回了 `null`（`EnhancedInputComponent.cs:29-35`）——**非 `_C` 类没有配方，这层保护随之消失**。⇒ 同一 `(Action, TriggerEvent, 函数)` 重复调用会累加引擎绑定，输入一次触发多次回调。

---

## 4. 施工记录

### 4.1 改动面（插件侧 5 个受跟踪文件 `+167/−213`）

| 文件 | `--numstat` | 改法 |
|---|---|---|
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterInputComponent.cpp` | **`+30/−55`** | `BindImplementation` 由「配方对象 + 索引 + 成员指针」收敛为「**条目结构体句柄**」（`template <typename TEntry, typename TProcess>`，改后 `:121-141`）；六个 `Bind*Implementation` 签名与调用同步（改后 `:159/:209/:251/:294/:343/:416`）；**导出数量与名字全不变** |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp` | **`+3/−1`** | `BindFunction` 命中分支补配对 `FoundFunction->RemoveFromRoot();`（改后 `:78`），与 legacy 对称 |
| `Script/UE/CoreUObject/InputComponent.cs` | **`+99/−117`** | 六个 `Bind*`：**条目改本地构造**（`FBlueprintInput*DelegateBinding`，改后 `:19/:63/:105/:147/…`）+ 配方退化为**只做记账**（`foreach` + `IsDuplicated`，不再维护索引）；原生调用一律挪出 `if` 且第三参改传 `HandleData.GetHandle(Binding)`；`ClearBindingValues()` 去参（改后 `:472-475`） |
| `Script/UE/CoreUObject/EnhancedInputComponent.cs` | **`+10/−14`** | `BindAction`：`Binding` 提到 `if` 之外、原生调用一律执行（改后 `:25-49`）；`RemoveAction` 删掉 `InObject.GetClass().RemoveFunction(…)`（改后 `:75-79`） |
| `Script/UE/Library/InputComponentImplementation.cs` | **`+25/−26`** | 六个包装层签名同步（去 `int InIndex`、第二参更名 `InBlueprintInput*DelegateBinding`） |

**工程侧**：新增 `Script/Game/UnrealCSharpTest/UnitTest/Input/InputBinding/TestInputBindingActor.cs`（**19 行**，类名 `ATestInputBindingActor`）+ 既有 `…/InputBinding/UnitTestSubsystem.cs` 新增 `TestInputBindingRawClass()`（**6 条断言**）+ `UnitTest/UnitTestSubsystem.cs` 一行接线 + 一行 `using Script.EnhancedInput;`。

**生成产物 0 / 协议面 0 / 公开签名变化 1 处**（`UInputComponent.ClearBindingValues(UObject)` → `ClearBindingValues()`，属 Q2 裁定项）。

### 4.2 关键设计（B′ 的形态）

```cpp
// 改后（FRegisterInputComponent.cpp:121-141）：不再依赖配方，条目的数据来源 = C# 传入的结构体
template <typename TEntry, typename TProcess>
static void BindImplementation(const IManagedHandle InManagedHandle,
                               const IManagedHandle InBlueprintInputDelegateBinding,
                               const IManagedHandle InObjectToBindTo,
                               const TProcess& InProcess)
```

```csharp
// 改后（InputComponent.cs:19-56 摘）：条目本地构造；配方只在"有配方"时落库（保 BPGC 重放与去重）
var Binding = new FBlueprintInputActionDelegateBinding { InputActionName = …, InputKeyEvent = …, FunctionNameToBind = … };
if (InputActionDelegateBinding != null) { …foreach 查重… if (!IsDuplicated) Bindings.Add(Binding); }
UInputComponentImplementation.UInputComponent_BindActionImplementation(
    HandleData.GetHandle(this), HandleData.GetHandle(Binding), HandleData.GetHandle(InObject));
```

**为什么这是最小面**：改后 legacy 侧的形态与**增强输入侧本来就有的形态**（`BindActionImplementation(this, 结构体句柄, 对象, 函数名)`）**收敛为同一种**——上一轮 A′ 改造留下的"配方 + 索引"是唯一需要索引的形态，本轮把它去掉，索引与 `IsValidIndex` 守卫一并消失。

### 4.3 六条语义变化（**必须并读**，不得只读"已修"）

| # | 变化 | 边界 / 代价 |
|---|---|---|
| 1 | `_C` 类路径**行为不变** | 条目仍写入配方、原生仍按同一组字段绑定/幂等；差别只在"原生读取来源"由配方数组元素改为 C# 传入的结构体（**同字段同值**） |
| 2 | 非 `_C` 类由"整段跳过" → "绑在当前输入组件上" | **不落配方**（无配方可落）、**不跨输入组件重建存活**（E4 决定的硬边界，见 §1.2） |
| 3 | 🔴 EnhancedInput 族在非 `_C` 类上"**可用但不幂等**" | 新登记 `F-CS2B-035`（§6）；legacy 六族由原生 `Has*Binding` 保证幂等，**已由用例坐实** |
| 4 | `ClearBindingValues()` 去参 | C# 公开签名变更；**仓内 0 调用点**（grep 命中仅定义与包装层）；判据更正见 §0 第 5 条 |
| 5 | EnhancedInput `RemoveAction` 不再删 UFunction | 与上一轮 Q2(a) 裁定对齐；安全性 = E8（`FuncMap` 保活） |
| 6 | enhanced `BindFunction` 命中分支补 `RemoveFromRoot()` | 与 legacy 对称；不改 `AddToRoot` 策略（其残余 → §6 `F-CS2B-034`） |

### 4.4 库内口径自检（对照 HEAD 基线；**收工后按维护者要求逐项复测**）

> **方法**：库里**没有**任何格式化配置（工程与插件全域 `.clang-format` / `.editorconfig` / `.gitattributes` / `Directory.Build.props` / `.DotSettings` **均无**；插件 `README.md` 也无风格小节）⇒ 家法只能**对照式**取得：① 与 **HEAD 同名文件**逐项比；② 与**同目录/同族姊妹文件**计数比。下表的"HEAD""姊妹"两列即这两个基线。

#### 4.4.1 格式（字节级，工作区 vs HEAD）

| 检查项 | 改后（工作区） | HEAD 基线 | 判定 |
|---|---|---|---|
| BOM | 5 文件**均无** | 5 文件**均无** | ✅ 一致（⚠️ 首次测量用 `Out-File -Encoding utf8` **被 BOM 污染**，已改用 `cmd` 重定向复测） |
| 行尾 | 原生 CRLF、C# CRLF（`CRLF == LF`，无混合） | 同 | ✅ |
| EOF 换行 | `.cpp` 有；三个 `.cs` **无**（末字节 `0x7D`） | 同（逐字节相同） | ✅ 保持了仓内"`.cs` 无末尾换行"的既有事实 |
| 缩进 | 原生 tab（498/154 行 tab 缩进，0 行空格缩进）；C# **0 个 tab** | 同 | ✅ |
| 行宽上限 | 108 / 108 / 120 / 118 / 173 | 108 / 108 / 120 / 118 / **180** | ✅ 均未变差（`InputComponentImplementation.cs` 因去掉 `, int InIndex` 反而**收窄 7 列**） |
| 行尾空白 | **0** | 0 | ✅ |
| 花括号 | 原生 `) {` 同行 = **0**；C# `) {` 同行 = **0**、`new F…{` 同行 = **0** | 同（Allman） | ✅ |
| lambda 收尾 | `});` = **18**、分裂写法（`}` 换行后 `);`）= **0** | 同 | ✅ |
| 续行对齐 | 7 个原生签名（含共享 `BindImplementation`）续行**逐列等于开括号列**（33/39/37/40/36/38/43 全 True）；`GetStruct<TEntry>(` 的续行 = **语句 +1 tab**，与同文件 `GetObject<UInputComponent>(` 既有形态同形 | 同 | ✅ |
| C# 包装层续行 | 6 个 `UInputComponent_Bind*Implementation` = **语句 +4** | 同 | ✅ |
| C# 对象初始化器 | 尾随逗号 = **0**、`var` 局部量 46 处 | 同 | ✅ |

#### 4.4.2 项目规范（口径面）

| 检查项 | 结果 |
|---|---|
| 新增行里的注释 / `UE_LOG` / `Console.` / `throw` / `try` / `catch` / `ensure` / `check(` | **全 0**（167 条新增行；`try` 的 2 次命中经查是模板形参 `TEntry`，`\btry\b` 计数 **0**） |
| 新增 `#include` / 成员变量 / 公开 API | **全 0**（`GetStruct<T>` 在同文件 `:530` 的 `Unbind` 路径已在用 ⇒ 无需新 include；唯一的公开签名变化是 Q2 裁定项） |
| **公开 API 面 diff**（程序化比对 HEAD ↔ 交付态） | `InputComponent.cs`：**仅** `ClearBindingValues(UObject)` → `ClearBindingValues()`（= Q2 裁定项，仓内 0 调用点）；`EnhancedInputComponent.cs`：**无差异**；六个 `Bind*` 的公开签名**零变化** |
| 旧形态残留 | 5 个受改文件 `InIndex` 命中 **0**；C# 侧"把配方句柄当绑定参数传入" **0**；原生侧旧守卫 `IsValidIndex` **0**（余下 6 处 `InBindings` 属既有的 `HasBinding`/`Unbind` 辅助函数） |
| 生成器命名规范 | 新宿主类名 `ATestInputBindingActor`（`A` 前缀 ⇒ 合规）+ 文件名 `TestInputBindingActor.cs`（非 `_C` 类 ⇒ 去首字母）**由 analyzer 强制**（`UC_ERROR_03` / `ErrorFileNameNotMatch`）：首版写成 `TestInputBindingActor` 时**构建直接报错**（见 §5.5），改名后 0 error |
| `if` 形态 | 新增分支一律**正向条件包裹 + 末尾默认返回**；`foreach` + `break` 与同文件 `Remove*` 既有形态一致 |
| 空指针/守卫 | `GetStruct<TEntry>` 判空后才用；原生 `Unbind*`、`GetNumActionBindings`、`ClearBindingValues` **一行未改**；`const` 正确性（`const auto` / `const TEntry&`）与同文件一致 |

#### 4.4.3 一处**实测偏差（已修正）**与工程侧既有噪声（非本轮引入）

| # | 项 | 事实 | 处置 |
|---|---|---|---|
| ① | 🔴 **新测试宿主的行尾** | 新建的 `TestInputBindingActor.cs` 落盘为 **LF**（22 处），而**同目录两个姊妹文件**（`UnitTestSubsystem.cs`、`CSharp_TestInputBindingActor_C.cs`）都是 **CRLF** | **已归一为 CRLF**（BOM 保持无、EOF 换行保持有），并按库内纪律**重编重跑**：`dotnet build` 0/0、新鲜度断言通过、`-game` **1816 / 0**，与上一版**逐条 diff = 0**（tag `G12TAIL-final3`） |
| ② | 工程侧行尾本就混合 | 131 个 `.cs` 中纯 CRLF **49** / 纯 LF **82** / 混合 **0** ⇒ 全仓无统一约定，只能按"就近原则"对齐同目录 | 登记为**仓级既有事实**，不在本轮扩大改动面 |
| ③ | 工程侧 EOF 换行本就混合 | 有末尾换行 **19** / 无 **112** | 新文件取**同目录姊妹**（两份都有）⇒ 保留末尾换行 |
| ④ | 套件内**重名用例** 2 处（**既有**） | `BlueprintCSharpOutSetDoubleFunction`、`BlueprintCSharpGetStringFunction` 各出现 **2 次**（1816 条里 2 组重名）—— 重名会让"按名字定位判据"失效 | 与本轮无关（不在输入绑定域），**登记给下一轮**做一次套件级去重核查 |
| ⑤ | 🔴 **本文档自身的行尾** | 本文首版落盘为 **LF**（312 处），而审计库 **72 份 `.md` 中 67 份是纯 CRLF**（纯 LF 仅 5 份，且其中 4 份是**既往轮次**留下的：`08-…/15`、`25`、`26`、`27`） | **已归一为 CRLF**（BOM 保持无、末尾换行保持有、U+FFFD = 0、锚点 `F-CS2B-03x`/`UC_ERROR_03`/`E14` 全在）。⚠️ **同时登记**：这 4 份既往报告的 LF 未动（不属本轮改动面），建议下一轮做一次库级 `.md` 行尾归一 |
| ⑥ | 🟢 我改动过的 5 份库内文件**未被改坏** | `00-推荐优先修复清单` / `01-全局问题总表` / `INDEX` / `README` / `00-总览/03` 复测**仍是纯 CRLF**（裸 LF = 0），说明逐行编辑**没有**把整文件的行尾重写掉（这是"编辑大文件"的常见事故） | 无需处置，留作判据 |

> **口径差异的自觉声明**：本轮**插件侧**新增注释 **0**（与库内口径一致）；**工程侧测试文件**新增 2 行说明性注释 —— 这是**照同文件既有形态**写的（上一轮在同一方法旁就留了同款"G12 回归判据：…"注释），故属"跟随局部家法"，不视为偏离。

---

## 5. 验证（改前 ↔ 改后，同一装置、同一用例集）

### 5.1 三态结果

| 判据 | 改前（HEAD 五文件） | 改后（交付态） | 交付态复跑 | 格式修正后复跑 |
|---|---|---|---|---|
| `dotnet build Script/Game/Game.csproj -c Debug` | **0 Warning / 0 Error**（35.41 s） | **0 Warning / 0 Error** | **0 Warning / 0 Error**（44.76 s） | **0 Warning / 0 Error** |
| UBT `UnrealCSharpTestEditor Win64 Development` | **Succeeded**（41.80 s） | **Succeeded**（69.16 s） | **Succeeded**（64.84 s） | 原生未再改动 ⇒ 沿用（DLL 新鲜度仍成立） |
| `-game` 全量套件 | **1816 / 5 失败** | **1816 / 0 失败** | **1816 / 0 失败，与改后逐条 diff = 0** | **1816 / 0 失败，与上一版逐条 diff = 0**（tag `G12TAIL-final3`） |
| 日志自检（`LogUnrealCSharp: Error` / `ensure` / `assert` / `LogBlueprint: Error`） | 0 | 0 | **0 命中** | **0 命中** |
| 用例总数核对 | 1816 = 上一轮基线 **1810** + 新增 **6** | 同 | 同 | 同 |

> **为什么有第四列**：收工后按维护者要求做格式核查时，发现新建的工程侧宿主文件**行尾为 LF**（同目录姊妹均为 CRLF）⇒ **已归一为 CRLF**。该文件属**受跟踪产物的一部分**（`Game.dll` 由它编译），按库内纪律"改动即需重编重跑"，故补了 `G12TAIL-final3`：`dotnet build` 0/0 + 新鲜度断言（`src 14:16:31 < Game.dll 14:18:58`）+ `-game` **1816 / 0**，与上一版**逐条 diff = 0**（换行符变化是**行为中性**的，实测与预期一致）。🔴 **插件侧 5 个文件在此期间未被触碰** ⇒ 补丁 `620279E8…` 与验证结论**仍然有效**（详见 §4.4.3）。

### 5.2 区分性断言（本轮收口证据）

| 断言 | 改前 | 改后 | 说明 |
|---|---|---|---|
| `InputRawBindActionAddsLiveBinding` | ❌ false | ✅ true | **非 `_C` 类根本绑不上**（`F-CS2B-032` 缺陷本体，运行期坐实） |
| `InputRawBindActionIsIdempotent` | ❌ false | ✅ true | 由原生 `HasActionBinding` 保证（legacy 幂等性同时被坐实） |
| `InputRawBindActionBindsSecondComponent` | ❌ false | ✅ true | 第二个输入组件同样拿不到绑定 |
| `InputRawEnhancedBindActionReturnsBinding` | ❌ false | ✅ true | 增强输入族同型体（回核扩展 ① 的收口证据） |
| `InputRawEnhancedBindActionKeepsAction` | ❌ false | ✅ true | 返回的绑定 `GetAction() == 传入的 UInputAction` |
| `InputRawRemoveActionRemovesLiveBinding` | ✅ true | ✅ true | **弱断言**（两态都过，只作护栏，不构成区分性证据） |
| 上一轮 6 条（`InputBindAction*` / `InputRemoveAction*`） | ✅ 全绿 | ✅ 全绿 | `_C` 路径**零回归** |

**用例名差异**：`onlyInNew = 6`（即上表 6 条）、`onlyInBaseline = 0`（对照上一轮 `G10H2-ctorinit` 的 1810 条基线）。

### 5.3 改前态是怎么造的（可复核）

1. `git -C Plugins/UnrealCSharp checkout -- <5 文件>` ⇒ 回到 HEAD；
2. **touch 五个源文件**（防"恢复文件保留旧 mtime ⇒ MSBuild/UBT 静默跳过重编"，同 [`08-…/24`](24-G12剩余两条收口-输入绑定解绑语义与身份键（2026-09-29）.md) §4 的装置教训）；
3. 重编 + **新鲜度断言**（`cpp 12:01:51 < dll 12:02:54`、`cs 12:01:51 < UE.dll 12:03:42`）；
4. 跑套件（tag `G12TAIL-before`）。
5. 恢复交付态用 `git apply`（正向，`exit=0`），并与施工期备份**逐字节比对 5/5 相同**（`2F2EDF63…` / `F9F701DA…` / `2457CB84…` / `C2A3F39E…` / `0B0DAA82…`），再 touch → 重编 → 新鲜度断言 → 复跑。

### 5.4 🪤 装置实录（两条都实测踩到，给下一轮直接用）

| # | 现象 | 判据 / 纠正 |
|---|---|---|
| ① | **MSBuild 复用节点锁 `obj`**：改前/改后各遇一次 `CS2012: Cannot open …/obj/Debug/net10.0/UE.dll for writing -- … locked by '.NET Host' (25620/36080)`（`/nodemode:1 /nodeReuse:true` 的常驻节点，是我**上一次构建**留下的） | 纠正 = `dotnet build-server shutdown`（或 `-nodeReuse:false` / `MSBUILDDISABLENODEREUSE=1`）。⚠️ **不得**据此杀 `UnrealEditor`（当时另有一棵树的 `ProjectMecury` 编辑器在跑） |
| ② | 跳过重编是**静默**的：`dotnet build` 仍打印 `Build succeeded`（若被锁则是 `Build FAILED`），而 UBT 会正常成功 ⇒ "构建成功 ≠ 新代码被编译" | **必须**断言 `产物 mtime > 源 mtime`。本轮第二次正是靠 `cs 12:05:41 < UE.dll 12:03:42 : **False**` 才发现交付态其实还是改前二进制 |
| ③ | 装置与另一棵树的编辑器共存 | `run-batch.ps1` 只杀自己启动的 PID ✓；但它"等所有 `UnrealEditor` 退出"的收尾循环会**空等满 30 s**（另一棵树 `ProjectMecury` 的编辑器 pid 23752 全程在跑）⇒ 每轮多 ~30 s，不影响判据 |

### 5.5 🟢 **E9 的实证：命名规范由 analyzer 强制，写在报告里不如撞一次**

新宿主类第一版我写成 `TestInputBindingActor`（既非 `_C` 结尾、也不以 `A` 开头），构建**直接失败**且给出的是插件自己的诊断编号：

```
TestInputBindingActor.cs(11,26): error UC_ERROR_03: The name of UClass TestInputBindingActor
  must end with "_C" or start with "A"
```

⇒ ① **E9 不是"约定"而是"硬约束"**（`UnrealTypeSourceGenerator.cs:470-481`：基类以 `A` 开头 ⇒ 类名须 `_C` 结尾或以 `A` 开头；以 `U` 开头同理 `:483-494`）；② 改名 `ATestInputBindingActor` 后，文件名还必须满足 `:686-695` 的规则（非 `_C` 类 ⇒ 文件名为**去掉首字母**，即 `TestInputBindingActor.cs`）——两条同时满足才通过；③ 这条链路同时**反证**了本轮的缺陷前提：`A*` 形态是**合法且被 analyzer 鼓励**的命名，而它恰恰是 `IsDynamicBlueprintGeneratedClass` 判为"非 BPGC"的那一半（E1）⇒ **"合法"与"有配方"是两件事**。

---

## 6. 新登记项（本轮**未修**，含理由与前置）

| # | 登记项 | 依据 | 为什么本轮不修 / 修法选项 |
|---|---|---|---|
| 🆕 **`F-CS2B-034`**（P3 / 活跃） | **`BindFunction` 的 `AddToRoot()` 与"命中分支 `RemoveFromRoot()`"不对称** ⇒ 同一函数名**只创建一次、之后不再以同名 Bind** 的路径上，UFunction **永久留在根集** | legacy 改后 `:574-612`（`AddToRoot` 在创建分支无条件执行、`RemoveFromRoot` 只在 hit 分支）；enhanced 改后 `:71-101`（本轮已补齐配对，同样只在 hit 分支释放）；**E8** 证明 `FuncMap` 已保活 ⇒ `AddToRoot` 本身多余 | 修法 = **删掉创建分支的 `AddToRoot()`**（一行），但它会**改变 `F-CS2B-018` 已登记的修复形态**（该修复正是"补配对 `RemoveFromRoot`"）⇒ 属裁定项，不在本轮口径内 |
| 🆕 **`F-CS2B-035`**（P2 / 活跃，**本轮改动新暴露**） | **非 BPGC 类上重复 `UEnhancedInputComponent.BindAction` 会累加多条引擎绑定** ⇒ 同一输入触发多次回调 | E12（引擎 `Add_GetRef` 无条件入队）+ `FRegisterEnhancedInputComponent.cpp:190-195` 无幂等判据（对照 legacy `HasActionBinding`）；`_C` 类靠 **C# 配方去重**（`EnhancedInputComponent.cs:29-35`）掩盖 | 三条候选：**(a)** 原生按 `(Action, TriggerEvent)` 扫描 `GetActionEventBindings()`（E14 有 public 访问器）—— 但条目**无函数名访问器** ⇒ 会误伤"同一 action+trigger 绑两个不同函数"的合法用法（`_C` 类允许）；**(b)** C# 侧对非 `_C` 类做本地去重（插件侧状态/静态表）；**(c)** 维持现状并登记。⚠️ **无运行期判据**：`EnhancedActionEventBindings` 无计数导出，加导出属"新增 API" ⇒ 同样需裁定 |

---

## 7. 未取得 / 未做（**不得读作"已验证"**）

1. **只测 CoreCLR / Win64 / 编辑器 Development**（与工程 ini 一致）；Mono / LeanCLR 与打包**未复测**。
2. **未跑生成器**（本轮不需要：`Script/UE/**` 是手写层，生成器只写两个 `Proxy` 根）⇒ "生成产物 0 改动"是**未触碰**而非"重生成后对照"；**未做产物 manifest 对照**。
3. **未做两阶段域重建装置**：本轮判据落在"活绑定计数 + 绑定返回值"上，不依赖域重建（口径同 [`08-…/24`](24-G12剩余两条收口-输入绑定解绑语义与身份键（2026-09-29）.md) §6-3）。
4. **活绑定计数只覆盖 legacy 的 Action 面**（新增导出只镜像 `UInputComponent::GetNumActionBindings()`）⇒ Axis / AxisKey / Key / Touch / VectorAxis 五族与 EnhancedInput 族是**结构证据 + 编译证据**（同一模板、同一签名形态、同一构造点），**未逐族建运行期判据**。
5. `F-CS2B-035`（重复绑定）**未取运行期判据**（原因见 §6）；`F-CS2B-034`（根集残留）**无任何运行期判据**（需 GC/根集探针）。
6. **未核**：增强输入侧的 `Unbind` 面与 legacy 是否等价（`RemoveAction` 只按 `FEnhancedInputActionEventBinding` 句柄移除）；`F-CS2B-025` 的**仓外** C# 调用者不可知（删参数对仓外是编译错误）；`FRegisterClass::RemoveFunctionImplementation` 是否清 `AddFunctionHash`（沿用 [`08-…/24`](24-G12剩余两条收口-输入绑定解绑语义与身份键（2026-09-29）.md) §6-6 的"未追"状态）。
7. **未构造"重建输入组件/域重建后重放"用例** ⇒ "非 `_C` 类不跨重建存活"是**引擎结构性结论**（E4），不是本轮实测值。
8. **提交状态（2026-09-30 更新）**：🟢 **插件侧 5 文件已提交 `152b56818cc6249b5487b5d4dc31ee3c287f5fc1`**（父 `77526cd2`；提交树 == 已验证工作区，见 §9.1）；⚠️ **工程侧用例仍未提交**（新宿主 + 1 个测试方法 + 1 行接线）；子模块指针未动（工程仓 `git status` 里 `M Plugins/UnrealCSharp` 是**历史累积**的未提交面，非本轮引入）。

---

## 8. 复用价值（给下一轮）

1. 🔴 **"合法类名有两种，但只有一种有配方"** —— 判据链 = `EndsWith("_C")`（E1）+ `UC_ERROR_03` 允许 `A`/`U` 前缀（E9）。**任何依赖 `UBlueprintGeneratedClass` 的机制**（`DynamicBindingObjects` 配方、`BindToInputComponent` 重放、`GetDynamicBindingObject`）**只覆盖一半合法命名**。新机制上手前先问："它在哪一半上成立？"
2. 🔴 **"引擎宣布支持" ≠ "引擎能重放"** —— E3（`SupportsInputDelegate` 弃用 + 默认恒真）与 E4（存储仍在 BPGC 上）必须并读：读引擎"意图"要读到**存储与遍历**那一层，否则会得出"引擎已支持所有 UClass，所以插件也能"的错误结论。
3. 🟢 **"静默 no-op"的第一判据是"可观测面"** —— 本轮沿用上一轮的 `GetNumActionBindings`，并为增强输入族补了"返回非 null"这一最小判据；`if (x != null) { …全部行为… }` 这种**把整段行为包进条件**的写法，是"没有判据就永远发现不了"的典型形态。
4. 🪤 **改前/改后切换的三件套**：`dotnet build-server shutdown`（防复用节点锁 `obj`）+ **touch 源文件**（防 mtime 陷阱）+ **产物 mtime > 源 mtime 断言**（防"构建成功但没编进去"）。本轮 ② 正是靠第三条抓到交付态没编。
5. 🟢 **判据代码两端共用**：改前/改后跑的是**同一份工程侧用例**（不新建判据）⇒ 一次套件同时给出"缺陷存在"（改前 5 失败）与"基线全绿"（上一轮 6 条两态皆绿）。
6. **回核的扩展价值**：登记文字说"1 个文件"时，同型体常在被登记文件的**姊妹文件**里（本轮 EnhancedInput 一次抓到三处：同型 `GetDynamicBindingObject`、`RemoveFunction` 残留、`RemoveFromRoot` 缺口）。
7. 🟢 **收敛到既有形态比新增通道更省**：B′ 之所以比"6 个新直绑导出"便宜，是因为**增强输入侧本来就是"结构体直传"**——把 legacy 收敛过去，导出数量与名字全不变。

---

## 9. 补丁与留档

| 项 | 路径 / 值 |
|---|---|
| 补丁（交付态） | `Saved/Retest/fix-G12TAIL-raw-class-binding.patch` —— **32 072 B**、SHA256 **`620279E85A857E25184C3B89D30D46C6904482984B859C36FE5D12BFAE7BF886`**、无 BOM（首 3 字节 `100,105,102`）、`git apply --reverse --check` **exit 0** ✅ |
| 施工期备份（逐字节对照用） | `Saved/Retest/G12TAIL-keep/`（5 份，恢复后 5/5 hash 相同） |
| 改前/改后/交付态证据 | `Saved/Retest/out/G12TAIL-before/`（**5 失败**，CSV `…-2026-9-30-12-3-56-991.csv`）、`out/G12TAIL-final/`（**0 失败**，`…-12-0-3-513.csv`）、`out/G12TAIL-final2/`（**0 失败**，`…-12-8-24-533.csv`）、`out/G12TAIL-final3/`（**0 失败**，`…-14-19-9-490.csv` = 格式核查后第四态） |
| 构建日志 | `Saved/Retest/g12tail-ubt-{final,before,final2}.log`（均含 `Compile Module.UnrealCSharp.cpp` + 重链；`before` 41.80 s / `final` 69.16 s / `final2` 64.84 s） |
| 工程侧改动 | 新 `UnitTest/Input/InputBinding/TestInputBindingActor.cs` + `…/InputBinding/UnitTestSubsystem.cs`（+6 断言）+ `UnitTest/UnitTestSubsystem.cs`（1 行接线）——**未提交**（工程仓 `git status` 仍显示 `?? Script/Game/UnrealCSharpTest/UnitTest/Input/` 与 `M …/UnitTest/UnitTestSubsystem.cs`） |
| 插件侧改动 | 🟢 **已提交 `152b56818cc6249b5487b5d4dc31ee3c287f5fc1`**（父 `77526cd2`，作者 crazytuzi，2026-09-30 14:39:01 +0800，subject `Set Enumerator && Helper Deinitialize && Lazy Rehash && Input Bind Struct`）—— 5 个受跟踪文件 `+167/−213`（原生 2 `+33/−56`、C# 3 `+134/−157`）；提交后工作区只剩**预存**的 `M Source/SourceCodeGenerator` |
| 提交级补丁 | `Saved/Retest/fix-G12TAIL-raw-class-binding-152b5681.patch`（由 `git diff 77526cd2 152b5681 -- <5 文件>` 导出）—— **32 072 B**、SHA256 **`620279E85A857E25184C3B89D30D46C6904482984B859C36FE5D12BFAE7BF886`**，**与施工期补丁逐字节相同** |

### 9.1 提交登记与提交树复核（2026-09-30）

| 判据 | 结果 |
|---|---|
| 本轮 5 文件的面（`git diff 77526cd2 152b5681 -- <5 文件>`） | **5 文件 `+167/−213`** —— 与施工期口径**逐字相同** |
| **blob 级复核（5/5）** | `git rev-parse 152b5681:<file>` ≡ `git hash-object <工作区>` ≡ `git hash-object <施工期备份>`：`FRegisterInputComponent.cpp` **`66ececb9d8e9…`**、`FRegisterEnhancedInputComponent.cpp` **`959d9b3dc5f3…`**、`InputComponent.cs` **`041fde9c1ce6…`**、`EnhancedInputComponent.cs` **`c88daafae3fb…`**、`InputComponentImplementation.cs` **`13829bc3373f…`** ⇒ **提交树 == 已验证工作区**是 hash 级事实 |
| 提交范围内 diff | `git diff 152b5681 -- Source/UnrealCSharp Script/UE` = **0 行** ✅（`git diff 152b5681 -- Source Script` 的 7 行**全部**属**预存**的子模块指针 `Source/SourceCodeGenerator`：`0c19c5e9…` → `0f2fd138…`，与本轮 5 文件无关） |
| 提交级补丁 vs 施工期补丁 | **SHA256 相同**（32 072 B / `620279E8…`）⇒ **维护者未做任何折行 / 空白 / EOF 修订**（对照 G8 `+66/−2` 折行、G14b 16 行、LeanCLR 5 处折行、`F-LEAK-016` 单语句折行） |
| 逻辑结论 | ⇒ **"提交树被验证过"不依赖重跑**（同 [`08-…/21`](21-G4剩余条目收口-动态类型静态容器与依赖图（2026-09-29）.md) §4 的 hash 级判据）。⚠️ 本文 §5.1 的第 2/3 列（改后与交付态复跑）与第 4 列（格式核查后 `G12TAIL-final3`）**全部对应提交树**，故三态证据对提交 `152b5681` **逐条成立** |
