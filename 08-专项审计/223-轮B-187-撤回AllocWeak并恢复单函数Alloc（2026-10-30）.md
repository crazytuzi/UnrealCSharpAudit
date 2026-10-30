# 轮 B-187：撤回 `AllocWeak`，恢复单函数 `Alloc(object, bool bIsWeak = false)`（2026-10-30）

> **来源**：维护者「还是用原来一样，只用 `Alloc` 一个函数」（在讨论过 C# 命名语义与 BCL 的 `GCHandle.Alloc(obj, GCHandleType)` 形态之后）。
> **性质**：**撤回轮**——回退 [`213-轮B-179`](213-轮B-179-T4剩余P2批次一-两条前提更正与句柄命名（2026-10-30）.md) 的 `F-CS1-002` P2 半边；插件 **21 文件 `+23/−27`**（全部 C#）。
> **基线**：插件 **`376d8cfd`** ⇒ 提交 **`376d8cfd`**；套件基线 `D520F5AD…`。
> **一句话结论**：`HandleData` 恢复为**单一** `Alloc(object, bool bIsWeak = false)`，21 处 `AllocWeak(x)` 回改为 `Alloc(x, true)`，`ObjectBridge` 回改为 `Alloc(Object, bIsWeak != 0)`；**行为零变化**（套件 2 趟同 SHA）。B-179 的另一半（`GIsRequestingExit` → `IsEngineExitRequested`）**保留**。

---

## 1. 撤回内容

| 文件 | 变化 |
|---|---|
| `Script/Interop/Handle/HandleData.cs` | `Alloc(object)` ＋ `AllocWeak(object)` ＋ 私有 `AllocHandle` ⇒ **`Alloc(object InObject, bool bIsWeak = false)`**（单函数） |
| 19 个包装类（21 处） | `HandleData.AllocWeak(x)` ⇒ `HandleData.Alloc(x, true)` |
| `Script/Interop/Bridge/ObjectBridge.cs` | 三元 ⇒ `HandleData.Alloc(Object, bIsWeak != 0)` |

**与 `4564008d`（B-179 之前）的差异核算**：`git diff 4564008d -- Script/Interop Script/UE` 只剩 **1 处**——`FMulticastDelegateImplementation.cs:14` 由单行（**116 列**）折成两行。这是**纯排版**差异（同一串实参），起因是 B-179 当初为过 §8 行宽把该行折过，撤回时保留折行即可继续合规（否则新增行 116 列会判 `OverWidth`）。其余 20 个文件与改前**逐字节相同**。

> `Script/CodeAnalysis/CodeAnalysis.cs` 与本次撤回无关（那是 B-185 的批处理），保留。

## 2. 判据

| 项 | 读数 |
|---|---|
| C#（`Script.sln` Release） | **0 Warning / 0 Error**（43.88 s） |
| 套件 | `B187-suite` ×2：**1 987 / 0 / `D520F5AD…`**（与改前逐字节相同 ⇒ 行为零变化） |
| UE 5.6 UBT | **Succeeded**（18.25 s；无 C++ 改动） |
| UE 5.0 UBT | **`Target is up to date`**（11.18 s） |
| §7 语法 | **CLEAN** |
| §8 风格 | **PASS**（21 文件、Added 1/1/2/1…、AddedMax **110**、`diff --check` 0） |
| G7（新增行） | 注释 **0**、日志 **0**、`throw` **0**、`try-catch` **0**、`ensure`/`check` **0** |

## 3. 交付

| 项 | 内容 |
|---|---|
| 补丁 | `Saved/Retest/fix-B187-revert-allocweak.patch` —— **14 907 B** / SHA256 **`A6A5E68E4B7B2DCD53803D9DECF45CDFEED04774245713059F2299ED867F279F`** / `git apply --reverse --check` **exit 0** |
| 提交 | 🟢 **`376d8cfd`**（subject `Revert HandleData Alloc Weak`，**21 文件 `+23/−27`**） |
| 保留 | B-179 的另一半：`FCodeAnalysis.cpp` 的 3 处 `GIsRequestingExit` → `IsEngineExitRequested()`（现合并为 `RunCommand` 内 1 处）**不回退** |

## 4. 撤回后的口径与残余

1. **`F-CS1-002` P2 状态**：**撤回（维护者裁定保留单函数）** ⇒ 该条回到"P1 已施工（G0：`ArrayBridge` 强句柄）＋ P2 不施工"。
2. **布尔参数陷阱回归**：`Alloc(x, true)` 重新依赖"看定义"。**已知先例**：`3da38bb6` 改签名时 `ArrayBridge.cs:19` 的 `true` 从 pinned 静默变成 weak（G0，已由 B-178 单独修复）。今后若再改 `Alloc` 签名，必须**逐调用点核对位置参数含义**——本页把这条写成硬约束。
3. **契约页同步**：[`212`](212-契约页-句柄所有权矩阵与OptionalHelper取用契约（2026-10-30）.md) §3 的"API 变更记录"已加 🔁 撤回注，矩阵里 `Alloc`/`AllocWeak` 的写法改为"`Alloc`（强）/ `Alloc(x, true)`（弱）"。
4. **未做**：`AllocStrong`/`AllocPinned` 的讨论随之作废（不再需要）。

## 5. 边界

1. 本轮只动 C#，未碰 C++；`Interop.dll` 由 C# 构建刷新。
2. 折行的那 1 处（`FMulticastDelegateImplementation.cs:14`）与外层基线的单行形态不同——属排版，不影响语义；若将来做"逐字节回滚"核对，需把这 1 处列入白名单。
3. 未跑 Mono/LeanCLR（纯 C# 句柄 API 形态变化，行为由套件覆盖）。
