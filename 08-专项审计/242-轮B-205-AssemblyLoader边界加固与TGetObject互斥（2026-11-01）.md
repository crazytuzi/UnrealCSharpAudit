# 轮 B-205 `F-ABI-003`≡`F-DOM-019` 边界加固 ＋ `F-FL-015` 模板互斥（2026-11-01）

> 分析范围：P2 队列「可直接施工」里最便宜的两项（[`08-…/241`](../08-专项审计/241-轮B-204-E组三条迁出与P2全量复核（2026-11-01）.md) §3/§7 建议 1–2）
> 覆盖文件：插件 **2 文件 `+17/−10`** —— `Script/Interop/AssemblyLoader/AssemblyLoader.cs`（`+16/−9`）、`Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h`（`+1/−1`）
> 状态：已完成 —— 🟢 **已提交 `da0db161`**（父 `ca513cb3`）；P2 **63 → 60**

---

## 1. `F-ABI-003` ≡ `F-DOM-019`：边界入口的 E1 加固

### 1.1 改前 / 改后

```csharp
// Script/Interop/AssemblyLoader/AssemblyLoader.cs:13-28（改前）
[UnmanagedCallersOnly]
public static unsafe nint LoadFromStream(byte* InData, int InLength, char* InInPublishDirectory)
{
    if (InData != null && InLength > 0)
    {
        Context ??= new UnrealAssemblyLoadContext(…);
        using var Stream = new UnmanagedMemoryStream(InData, InLength);

        return HandleData.Alloc(Context.LoadFromStream(Stream));
    }

    return 0;
}
```

```csharp
// 改后（`:13-37`）：整段原函数体包进一层 try，catch 返回既有失败值 0
[UnmanagedCallersOnly]
public static unsafe nint LoadFromStream(byte* InData, int InLength, char* InInPublishDirectory)
{
    try
    {
        …原函数体逐字不动…

        return 0;
    }
    catch (Exception)
    {
        return 0;
    }
}
```

### 1.2 E1 机械判据（四条硬约束逐条对照）

| 约束 | 读数 |
|---|---|
| ① `catch` 内**不得**新增日志 / 断言 / `throw` / 公开 API | 块内只有 `return 0;` ⇒ **0/0/0/0** |
| ② 返回值必须落在该函数**既有**失败取值 | 原文 `:27` 即 `return 0` ⇒ 一致，**未新增错误码** |
| ③ 只允许一层，无嵌套、无 `when` 过滤 | 单层 `try` / 单层 `catch` |
| ④ `try` 必须覆盖**整个**原函数体 | `:16-27` 全在 `try` 内（`if` 分支与其 `return 0` 都在） |

### 1.3 为什么两条编号一处修

`F-ABI-003`（[`08-…/04`](../08-专项审计/04-跨语言ABI与平台兼容审计.md)）与 `F-DOM-019`（[`02-…/04`](../02-UnrealCSharpCore核心/04-脚本域抽象与三大后端.md)）描述同一处代码。`F-DOM-019` 的 `01` 行内 B-125 处置为「暂缓（**Q2 排除新 try/catch**）」；该封锁已由 **2026-10-25 的 `E1` 边界屏障白名单**（`_CONVENTIONS` §8.5）解除，故本轮按 E1 形态一直接落地，**无需新裁定**。

---

## 2. `F-FL-015`：`TGetObject` 部分特化互斥

### 2.1 缺陷（本轮实测复现，非静态推断）

`FCSharpEnvironment::TGetObject<T, Enable>` 有 4 个部分特化，其中两个用 `enable_if_t<条件, T>` 把第二参数折回 `T`；当某类型**同时**满足 `TIsUStruct`（有 `StaticStruct`）与 `TIsScriptStruct`（被 `BINDING_STRUCT` 宏登记）时，两个特化同时匹配 ⇒ 歧义。

**探针**（临时 1 文件，已删除）：`Source/UnrealCSharp/Private/Probe/B205Probe.cpp`

```cpp
struct FB204OverlapProbe
{
	static UScriptStruct* StaticStruct();
};

template <>
struct TIsScriptStruct<FB204OverlapProbe>
{
	enum { Value = true };
};

void FB204ProbeCall(const FCSharpEnvironment& InEnvironment, const IManagedHandle InManagedHandle)
{
	FCSharpEnvironment::TGetObject<FB204OverlapProbe, FB204OverlapProbe>()(InEnvironment,
	                                                                       InManagedHandle);
}
```

**改前读数（真实编译器，真实头文件）**：

```
B205Probe.cpp(18,70): error C2752: 'FCSharpEnvironment::TGetObject<FB204OverlapProbe,FB204OverlapProbe>':
                                      more than one partial specialization matches the template argument list
FCSharpEnvironment.h(272,8): note: could be 'TGetObject<T,enable_if<TIsUStruct<T>::Value,T>::type>'
FCSharpEnvironment.h(283,8): note: or       'TGetObject<T,enable_if<TIsScriptStruct<T>::Value,T>::type>'
```

> 🔴 该读数**推翻了「隐患/静态推断」的措辞**：它不是"可能存在歧义"，而是**已经能用两行代码构造出 C2752** —— 只是仓内当前没有任何类型**同时**满足两个 trait，所以平时编不过去的路径没人走到。

### 2.2 修法（1 行）

```cpp
// Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h:272
-	class TGetObject<T, std::enable_if_t<TIsUStruct<T>::Value, T>>
+	class TGetObject<T, std::enable_if_t<TIsUStruct<T>::Value && !TIsScriptStruct<T>::Value, T>>
```

**为什么不合并两个分支**（两处 `operator()` 的体逐字相同，合并也"等价"）：

- 合并会删掉一个特化（`−10` 行），而 `TGetObject` 是**公开**模板面，删除改变可见形态；
- 更关键的是 `TIsUStruct` / `TIsScriptStruct` 还被 `TIsReflectionClass`（`||`）、`TIsProjectClass`（`if constexpr` 先 UStruct 后 ScriptStruct）、`TName`/`TNameSpace`/`TPropertyValue` 等 20 余处消费 ⇒ **改 trait 本体**的爆炸半径远大于在 `TGetObject` 这一处加互斥。本轮取**局部互斥**，其余消费点行为零变化。

### 2.3 改后读数

同一探针、同一编译器 ⇒ **`Result: Succeeded`**（46.84 s）；删掉探针后终态再编 ⇒ **`Result: Succeeded`**（36.42 s）。

> 说明：`TGetObject` 的两条分支体相同，故"哪条胜出"不影响行为；本条修的是**可编译性**（歧义 ⇒ 硬错误），不是语义。

---

## 3. 判据汇总

| 项 | 读数 |
|---|---|
| C# 构建（Game） | **0 Warning / 0 Error**（39.42 s） |
| C# 构建（Interop） | **0 Warning / 0 Error**（0.81 s）—— `Script/Interop/Interop.csproj` 直接把 `../../Plugins/UnrealCSharp/Script/Interop/**` 编进去 ⇒ **改的就是被编的那份**（不是模板/副本两套树的问题）；产物直写 `Content/Script/Interop.dll` |
| UBT 5.6（探针 · 改前） | **FAILED** —— `error C2752`（见 §2.1，本轮 A/B 的"改前"） |
| UBT 5.6（探针 · 改后） | **Succeeded**（46.84 s） |
| UBT 5.6（终态去探针） | **Succeeded**（36.42 s） |
| UBT 5.0（`UE50Check`） | **19/19 action、0 error**（82.38 s；仅引擎头 `C4996` 弃用告警，非本插件） |
| 套件 ×2 | **1 992 行 / 1 991 例 / 0 失败**；SHA256 `085D3178D15B97…`（两次逐字节相同，与基线同 SHA） |
| §7 `b29-xversion-lint.ps1 -Mode Syntax -DiffBase HEAD` | **CLEAN** |
| §8 `b30-style-lint.ps1 -Base HEAD` | **PASS** —— 2 文件；`Added 16 / 1`；C# **4 spaces**、C++ **TAB**；`Trailing 0`、`SpaceFail 0`、`OverWidth 0`、`AddedMax 94 / 93`（≤110）；`diff --check` exit 0；无 BOM；`formatter: NOT required` |
| G7 机械计数 | `try/catch +1`（**E1 形态一**，位于 `[UnmanagedCallersOnly]` 入口本身）、`throw 0`、日志 **0**、`ensure/check 0`、新增注释 **0**（探针文件已删除）、新增公开面 **0** |
| 代码面 | 插件 2 文件 `+17/−10`；🟢 提交 **`da0db161`** |

**正面控制（正常路径未被破坏）**：套件 1 991 例全部跑通 ⇒ 游戏程序集必然被加载并执行，而 CoreCLR 下 `AssemblyLoaderLoadFromStreamFn` 的**唯一调用点**是 `FCoreCLRDomain.cpp:244`（`AssemblyLoader.cs:13` 那个入口）⇒ 改后的函数在真实路径上被调用且行为不变。

---

## 4. 未取得（如实登记）

1. **失败路径的运行时 A/B 未取得**。原因两条：① `[UnmanagedCallersOnly]` 方法**不能从托管代码直接调用**（编译器禁止），现有探针手段（`FAutoConsoleCommand` → 原生）可以构造，但 ② 该缺陷在 **CoreCLR 下表现为 `TerminateProcess`** ⇒ 同一进程内的 A/B 是**破坏性**的（"改前"必然让 `-game` 进程直接终止、拿不到完整 CSV），而本机套件后端正是 CoreCLR。**补测设计**（留待需要时）：临时 `FAutoConsoleCommand` 用伪造字节调 `IScriptDomain::LoadAssembly`，改前预期进程终止、改后预期返回 0 且编辑器存活；需两次「回退/恢复 + 重编」，成本约 4 次 UBT。
2. **`F-FL-015` 的歧义"可达性"未证**：本轮只证明**可构造**（探针），未证明**仓内真实存在**同时满足两个 trait 的类型（若有，则当前 HEAD 本来编不过 ⇒ 反证没有）。**该条的处置口径**：属于「潜伏但可构造」，按"编译期硬错误 + 1 行互斥"收口，**不**宣称"已消除某条实际编译失败"。
3. 生成器产物 manifest **未跑** —— 两项改动都不在生成路径上（`TGetObject` 是 C++ 运行时 dispatch；`AssemblyLoader` 是 C# 运行时），产物必然零变化，但按纪律**不冒充已测**。

---

## 5. 台账

| 位置 | 改前 | 改后 |
|---|---|---|
| `01` | 754 行 / P2 63 | **752 行 / P2 60**（迁出 3 行 ＋ 1 行迁移说明） |
| `06` | 63 条（F 16 / C 36） | **60 条（F 14 / B 0 / C 35 / D 2 / E 8 / G 1）** |
| `05` | 至 `§140` | 新增 **`§141`**（3 条已修） |
| `07` | 至 B-204 | §7 补 **B-205**（3 条施工） |
| 插件 | `ca513cb3` | **`da0db161`** |

> **顺带确认**：`F-DOM-019` 的"暂缓（Q2 排除 try/catch）"理由**自 2026-10-25 起即已作废**——这是本轮复核（[`08-…/241`](../08-专项审计/241-轮B-204-E组三条迁出与P2全量复核（2026-11-01）.md) §4）的结论，本轮只是把它施工掉。今后判"撞 G7"时，**必须先看口径是否已被 E1/E2/E3 解除**，不能沿用旧裁定。
