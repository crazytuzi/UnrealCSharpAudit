# F-CS1-020 收口：`TypeBridge.MakeGenericType2` 读错泛型实参（2026-09-24）

> **对象**：`09-问题清单/00-推荐优先修复清单.md` §3 G13 名下、**同一根因被 4 份报告各自登记**的那一条 —— `F-CS1-020`（[`06-…/01`](../06-CSharp运行时与脚本/01-Interop桥接与程序集加载.md)）、`F-LEAK-007`（[`08-…/02`](../08-专项审计/02-资源泄漏审计（内存句柄与容器）.md)）、`F-ABI-005`（[`08-…/04`](../08-专项审计/04-跨语言ABI与平台兼容审计.md)）、`F-PERF-002`（[`08-…/05`](../08-专项审计/05-性能热点与优化清单.md)）。
> **快照与口径**：行号是本轮实测的插件工作区状态，基线提交 = **`0170514d`**（G2 + 本轮 + G13 前段三笔的**压缩提交**，2026-09-25）；工程侧 `UnrealCSharpTest` 有维护者**未提交的改动**（`Config/**`、`Script/Game/**` 等），本轮**未触碰**。
> 🟢 **提交状态（分两处，须知）**：**插件侧已提交 `0170514dc320848307c3f0f9f7f8e41fe75ead21`**（**压缩提交 `0170514d`**：2026-09-25 00:05:45 +0800，提交信息 `Handle Validation && MakeGenericType2 Value && String Literal Escape`，父提交 `7138ae1b102f50412c619d2f74454d1e06f1277a`；**本轮部分 1 文件 `+1/−1`**，提交级 **85 文件 `+702/−193`** —— ⚠️ 与 G2、G13 前段两轮**压缩为同一次提交**）。⚠️ **工程侧（`UnrealCSharpTest`）的两处改动仍未提交**：`Script/Game/UnrealCSharpTest/UnitTest/Container/TMap/UnitTestSubsystemGenericType.cs`（🆕）与 `…/UnitTestSubsystem.cs`（+1 行调用）—— 它们属**另一个仓库**，本轮未动其工作区状态。✅ **发布产物即该提交构建**：`Content/Script/{UE,Interop,Game}.dll` 时间戳 **16:10:36 ~ 16:10:51**，均早于提交时间 16:16:02，且此后未再构建 ⇒ §6 的成绩对应 `0170514d` 的代码。
> **方法**：源码级定位（含 C++ 侧调用链回核）+ C# 构建 + **"改前失败 / 改后通过"的端到端对照** + `-game` 全量套件 + 两阶段域重建。

---

## 0. 结论摘要

1. **修复 = 一行**：`Script/Interop/Bridge/TypeBridge.cs:194` 的 `HandleData.GetObject(InKeyType)` → **`InValueType`**。
2. 🔴 **"改前"证据是决定性的、且比报告推演更强**：报告说"会构造出 `Dictionary<K,K>` 并可能在下游静默失败"；**实测是在第一次读到该属性时直接抛 `InvalidCastException` 并中断整个测试套件**。原文为：
   ```
   LogUnrealCSharp: Error:
   Unhandled Exception:
   System.InvalidCastException: Unable to cast object of type
   'Script.CoreUObject.TMap`2[Script.CoreUObject.FName,Script.CoreUObject.FName]'
   to type 'Script.CoreUObject.TMap`2[Script.CoreUObject.FName,System.Int32]'.
   ```
   这条**逐字证实了缺陷机制**：C++ 侧为 `TMap<FName,int32>` 构造出来的托管类型是 **`TMap<FName,FName>`**（值实参被键实参顶掉）。
3. **"改后"通过**：同一条路径正常返回 `TMap<FName,int32>`，异常消失、套件由**中断在 1640 行**恢复为 **1705 行全绿**。
4. **新判据已入册**（`MapGenericTypeValueArgument`）：套件里**原本没有任何"键值异型"的 map 用例**（全工程只有 `TMap<int32,int32>` 与 `TMap<UObject,UObject>`，K≡V 恰好把这个 bug 完全遮住）—— 这是它长期潜伏的直接原因。本轮补上这一条永久回归判据。
5. ✅ **同族 `F-CS1-021` 经核实"已修"**（本轮未改代码，只做登记）：三处出口都已用 `GetUTF8CharCount`（见 §5）。
6. ⚠️ **未做的**：`F-CS1-022`/`023`/`024`/`025` 仍开放（本轮只三角定级，见 §5）；只测 **CoreCLR**。

---

## 1. 缺陷与修复

### 1.1 代码事实（当前基线 `0170514d`）

```csharp
// Script/Interop/Bridge/TypeBridge.cs
187:    [UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
188:    public static nint MakeGenericType2(nint InGeneric, nint InKeyType, nint InValueType)
189:    {
190:        if (HandleData.GetObject(InGeneric) is Type Generic)
191:        {
192:            if (HandleData.GetObject(InKeyType) is Type Key)
193:            {
194:                if (HandleData.GetObject(InKeyType) is Type Value)   // ← 缺陷：应为 InValueType
195:                {
196:                    return HandleData.Alloc(Generic.MakeGenericType(Key, Value));
197:                }
198:            }
199:        }
200:
201:        return 0;
202:    }
```

**修复（本轮唯一的生产代码改动，1 行）**：

```diff
-                if (HandleData.GetObject(InKeyType) is Type Value)
+                if (HandleData.GetObject(InValueType) is Type Value)
```

### 1.2 为什么"只有键值异型的 map"才暴露

`Value` 与 `Key` 取自**同一个句柄** ⇒ `MakeGenericType(Key, Key)`。于是：

| 场景 | 结果 |
|---|---|
| `TMap<int32,int32>`（= 本工程**全部**测试用例） | 退化成 `Dictionary<int,int>`，**与正确结果逐字相同** ⇒ 完全看不出 |
| `TMap<FName,int32>` / `TMap<FName,FString>` / `TMap<FString,bool>` … | 退化成 `Dictionary<FName,FName>` 等 ⇒ 类型错、值语义丢失 |

⚠️ **一个与报告不同的点**：报告推测下游后果以"**静默**返回 0 / 数据丢失"为主（`ArrayBridge` 元素类型错、`Unbox*` 模式匹配失败）。本轮实测走的是**另一条更响的路径** —— 反射 map 属性的 C# getter 是**硬转换**（`(TMap<FName,int>)HandleData.GetObject(...)`），类型不符直接在托管侧抛 `InvalidCastException`（见 §3），并被插件的 `[UnmanagedCallersOnly]` 边界吞成一条 `LogUnrealCSharp: Error`，**随后整个 `Test()` 中断**。⇒ **后果纠正**：不是静默，而是"**异常 + 该入口之后的全部工作被跳过**"（本次表现为 **64 条用例静默消失**，套件仍报 0 失败）。

---

## 2. 触发路径（C++ 侧回核，逐跳可复核）

```
C# 读反射 map 属性（例：FPerPlatformInt.PerPlatform）
  → FPropertyImplementation.FProperty_GetStructPropertyImplementation(...)      // 取属性
  → 属性描述符首次构造：TPropertyDescriptor<FMapProperty>(InProperty)
       · TPropertyDescriptor.inl:12   Class(FTypeBridge::GetClass(InProperty))
  → FTypeBridge.cpp:335               CastField<FMapProperty> → GetClass(MapProperty)
  → FTypeBridge.cpp:561-571           GetClass(const FMapProperty*)
       · :565 FoundGenericClass = FReflectionRegistry::Get().GetTMapClass()
       · :567 FoundKeyClass     = GetClass(InProperty->KeyProp)
       · :569 FoundValueClass   = GetClass(InProperty->ValueProp)
       · :571 MakeGenericTypeInstance(…, Key, Value)
  → FTypeBridge.cpp:181-194           两个非空 ⇒ ScriptDomain->MakeGenericType(…, Key, Value)
  → FScriptDomainImpl.inl:462-472     SCRIPT_DOMAIN_INVOKE(… TypeBridgeMakeGenericType2Fn, …)
  → IScriptTypes.h:122                Op(TypeBridgeMakeGenericType2, …, 3 参)
  → TypeBridge.cs:188                 MakeGenericType2(InGeneric, InKeyType, InValueType)  ← 缺陷在此读错参数
```

⇒ **结论：任何"键值异型的 `TMap` 属性"第一次被读取时都会命中**。这不是边角路径，而是 map 属性描述符构造的**必经之路**（`FMapPropertyDescriptor : TCompoundPropertyDescriptor<FMapProperty>`，`FMapPropertyDescriptor.h` + `TPropertyDescriptor.inl:12`）。

---

## 3. 改前 / 改后对照（本轮核心证据）

**判据**：新增 C# 用例 `MapGenericTypeValueArgument` —— 构造 `FPerPlatformInt`（`[PathName("/Script/CoreUObject.PerPlatformInt")]` 的 struct 包装）并读其 `PerPlatform` 属性（**`TMap<FName,int32>`，K≠V**），再从 `IEnumerable<KeyValuePair<K,V>>` 取泛型实参、断言**第二个实参是 `System.Int32`**。

> 为什么用 `IEnumerable\`1` 而不是 `IDictionary\`2`：`TMap<TKey,TValue>` **只实现** `IEnumerable<KeyValuePair<TKey,TValue>>`（`TMap.cs:8`），不实现 `IDictionary` ⇒ 按 `IDictionary` 取会拿到 `null` 而误判。

| 轮次 | 结果 | 判据 | 异常 |
|---|---|---|---|
| **改前**（`P1-cs1020-prefix`） | CSV **1640 行**（基线 1704 **少 64**）、0 失败；`MapGenericTypeValueArgument` **未出现**（异常发生在记录之前） | — | 🔴 **`InvalidCastException`：`TMap[FName,FName]` → `TMap[FName,Int32]`** |
| **改后**（`P1-cs1020-postfix`） | CSV **1705 行**、**0 失败** | ✅ **`MapGenericTypeValueArgument = true`** | ✅ `InvalidCastException` **0**、`Unhandled Exception` **0** |
| 用例名差异（改后 vs 基线） | **仅新有 = `MapGenericTypeValueArgument`**；**仅基线有 = 空** | — | — |

⇒ **"改前失败 / 改后通过"的闭环成立**，且基线 1704 条**全部仍在且全通过**（无回归）。

---

## 4. 改了什么（改动面）

| # | 文件 | 改动 | 仓库 |
|---|---|---|---|
| 1 | `Plugins/UnrealCSharp/Script/Interop/Bridge/TypeBridge.cs:194` | **1 行**（`InKeyType` → `InValueType`） | 插件仓库（`UnrealCSharp`） |
| 2 | `Script/Game/UnrealCSharpTest/UnitTest/Container/TMap/UnitTestSubsystemGenericType.cs` | 🆕 新增测试文件（1 个用例） | 工程仓库（`UnrealCSharpTest`） |
| 3 | `Script/Game/UnrealCSharpTest/UnitTest/Container/TMap/UnitTestSubsystem.cs:10` | **1 行**（在 `TestMap()` 里调用新用例） | 工程仓库 |

* **协议面 0 改动**、**C++ 侧 0 改动**、**生成产物 0 改动** ⇒ 不需要重新生成 proxy、不需要 UBT 构建。
* 新增测试文件用**独立文件**（`UnitTestSubsystemGenericType.cs`），避免与维护者在 `UnitTestSubsystem.cs` 上的**未提交改动**冲突；对该文件的唯一改动是 1 行调用。
* `Script/UE/UE.csproj` 以 `Compile Include="..\..\Plugins\UnrealCSharp\Script\UE\**"` 直接编译插件源（见 `Script/Shared.props` 与 `UE.csproj`），但 **`Interop` 不在其列** —— 实测 `dotnet build Script.sln -c Release` 会一并重编 `Interop.dll` 并把 `UE.dll`/`Game.dll`/`Interop.dll` **直接写进发布目录 `Content/Script/`**（构建输出的 `-> Content\Script\Interop.dll` 一行即证）。

---

## 5. 同族条目三角定级（本轮只核实、不改代码）

| 编号 | 级别 | 本轮核实结论 |
|---|---|---|
| **`F-CS1-021`** | P1 | ✅ **已修（无需动作，仅登记）**：报告称"按**字符数**截断后 UTF-8 编码 → 非 ASCII 类型名抛异常"。实测三处出口**都已改用 `GetUTF8CharCount`**：`TypeBridge.cs:112`（`GetNamespace`）、`:135`（`GetName`）、`:162`（`GetFullName`），helper 在 `:501`（逐字符算 1/2/3/4 字节、代理对按整对计）。⇒ **本条应从"未处置 P1"移出**；证据 = 源码 + 与 [`09-…/00`](../09-问题清单/00-推荐优先修复清单.md) §2.1 第 4 项（G7 轮 `F-ABI-001` 的改法）一致 |
| `F-CS1-022` | P2 | ⚠️ **仍开放**：`GetMethod` 只用"名字 + 参数量"匹配重载（`:63-69`）。与 `F-ABI-004` 已修的"个数量校验"是**不同**问题（这里指同元数重载的静默任选） |
| `F-CS1-023` | P2 | ⚠️ **仍开放**：`StringToType` 并发无锁（`:12`、`:448`、`:471`）。⚠️ 本轮**未复测**线程面 |
| `F-CS1-024` | P2 | ⚠️ **仍开放**：无程序集限定名时全程序集线性扫描取首个同名类型（`:453-496`） |
| `F-CS1-025` | P3 | ⚠️ **仍开放**：22 个 `Box*`/`Unbox*` 样板（`:204-444`）；`Read/WritePrimitiveValue` 缺 `char` 分支 |

> 📌 **报告"验证方式"的一处更正（重要，避免后来人误改）**：报告建议用
> `Select-String 'GetObject\(In(\w+)\) is Type (\w+)'` 逐条核对"模式变量与参数名的对应关系"。
> **该正则会产生 4 处假阳性** —— `:53` / `:105` / `:128` / `:151` 都是 `GetObject(InHandle) is Type Type`：那里的参数名**本来就不该等于**模式变量名（把 `Type` 实例取出来命名为 `Type` 是正常的）。**真正的唯一不匹配是 `:194`**（`InKeyType` → `Value`），`:192`（`InKeyType` → `Key`）是正确的配对。

---

## 6. 验证（全部实做，可复核）

| 项 | 命令 / 判据 | 结果 |
|---|---|---|
| C# 构建（改前/改后各一次） | `dotnet build Script.sln -c Release` | **Build succeeded**、**0 Error**；产物直接刷新到 `Content/Script/{UE,Interop,Game}.dll` |
| **`-game` 全量套件（改后）** | `run-batch.ps1 -Tag P1-cs1020-postfix` | **1705 例 / 0 失败**、`ensure`/`assert` **0**、`Fatal error` **0**、`InvalidCastException` **0**（`out/P1-cs1020-postfix/`） |
| **两阶段域重建（改后）** | `run-r1-verify.ps1 -Tag P1-cs1020-r1` | **2 × 1705 / 1705**、0 失败、0 `ensure`/`assert`、`LogBlueprint: Error` 0、编辑器自行退出（`out/P1-cs1020-r1/`） |
| 用例名回归判据 | 与基线 `pendinggroup-r01-csv01.csv`（1704/0）逐名比对 | **仅新有 = `MapGenericTypeValueArgument`**；**仅基线有 = 空** |
| 改前对照 | `run-batch.ps1 -Tag P1-cs1020-prefix` | CSV **1640 行**、`InvalidCastException`（`out/P1-cs1020-prefix/`，日志 L1080-1082） |
| 残留检查 | `grep InKeyType` 于 `MakeGenericType2` 体内 | 仅剩 `:192` 的**正确**用法（取 `Key`） |

---

## 7. 未做 / 未取得（如实声明）

1. **只测 CoreCLR**（当前 ini：`WindowsScriptDomainType=CoreCLR`）；Mono / LeanCLR **未复测**。⚠️ 本条的异常语义在两后端不同（见 [`06-…/01`](../06-CSharp运行时与脚本/01-Interop桥接与程序集加载.md) 对 `F-CS1-010`/`F-CS1-021` 的说明）：CoreCLR 下 `[UnmanagedCallersOnly]` 逃逸异常 = 进程终止风险，而实测本次**被插件边界吞成 Error 日志 + 中断 `Test()`**，未终止进程 —— **本轮未追究该差异的成因**。
2. **`F-CS1-022`~`025` 未修**（只三角定级，见 §5）。
3. **其它 `TMap` 键值异型组合未逐一覆盖**：新判据只覆盖 `TMap<FName,int32>` 一种。`TMap<FString,bool>` / `TMap<FName,FString>` 等（proxy 里共 20+ 种异型声明）走的是同一条 `MakeGenericType2`，**按机制推断同修**，但**未逐个用例验证**。
4. **未做提交**：插件侧 1 行 + 工程侧 2 处仍在工作区。
5. **未量化收益**：本次只证明"不再抛异常 / 类型正确"，未统计此前有多少条真实调用路径被这条缺陷影响。
