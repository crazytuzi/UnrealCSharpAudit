# 轮 B-104：`SanitizeFloat` 精度/极小值对齐 ＋ `RemoveAt` 门控改 5.4 ＋ UE 5.4 实编（2026-10-27）

> **来源**：维护者「对齐精度和极小值，然后 一行 + 5.4 实编复核」＋ 对 B 方案的质疑「这个函数好像低版本引擎（5.0 或者其他 5.x）没有吧，你再看下」。
> 🔁 **命名已由 [`140-轮B-105`](140-轮B-105-SanitizeFloat标识符零新名对齐（2026-10-27）.md) 收口**：本页 §2 代码里的 `Value`/`bIsBelowFixedPrecision`/`bIsFixedUnderflow` 三个新名已被消除（条件内联、形参就地归一），**行为与判据逐字不变**；交付形态以 140 为准。
> **口径**：G7 ＋ Q2 ＋ §7 ＋ §8；**插件 3 文件**（`TDefaultArgument.inl` `+19/−5`、`UEVersion.h` `+1/−1`、生成器模板 `+9/−3`）；累计 `+29/−9`。
> **基线**：插件 **`91fc122a`**；套件 1 980 断言 / `7F0C2E4C…`。
> **一句话结论**：**B（删 fork、直接调引擎）不成立** —— `FString::SanitizeFloat` 在 **5.0.0–5.2.1 的声明没有 `CORE_API`**（5.3.0 才加），模块化插件在 5.0–5.2 上会链接失败；因此改为**在 fork 内对齐**：默认精度 **6 位**（= 引擎 `%f`）、`≤5e-7` 回退 9 位（保住 `1e-8` 类默认值）、`≤5e-10` 再回退 `%.9g`、阈值改用引擎的 `FMath::IsNearlyZero`；`RemoveAt` 门控 **5.5.0 → 5.4.0**（32 个 tag 实证）并在 **UE 5.4 实编点 exit 0**。

---

## 0. 结论摘要

| # | 事项 | 读数 |
|---|---|---|
| 1 | **B 方案（直接调引擎函数）** | ❌ **否决**：`FString::SanitizeFloat` 的 `CORE_API` **5.3.0 才出现**；5.0.0–5.2.1 无导出 ⇒ 模块化构建 LNK2019 |
| 2 | **精度对齐**（fork 内） | 默认 `%.6f`（= 引擎 `%f`）；`|x| ≤ 0.5e-6` 回退 `%.9f` |
| 3 | **极小值对齐**（fork 内） | `|x| ≤ 0.5e-9` 再回退 `%.9g`；两个阈值都用 `FMath::IsNearlyZero` |
| 4 | **探针**（11 个值，已移除） | `1e-10\|0.00000001\|5e-10\|0.0001\|0.0\|0.0\|0.999845\|0.123457\|0.000001\|0.0000005\|1234.56789` |
| 5 | **产物影响** | `1e-8` 的 16 处默认值**不变**；**3 处** `Vector.cs` 的 `0.999845028 → 0.999845` |
| 6 | **一行（门控）** | `UE_F_STRING_REMOVE_AT_E_ALLOW_SHRINKING` **5.5.0 → 5.4.0**（`RemoveAt(…, EAllowShrinking)` 首现 **5.4.0**） |
| 7 | **5.4 实编** | 子模块 `5.5`（net8.0）⇒ `NETSDK1045` **exit 6**；切 `5.3`（net6.0）⇒ **exit 0**（48.03 s，插件三模块编译 ＋ 链接） |
| 8 | **5.6 回归** | UBT **Succeeded**（47.53 s 探针态 / 38.54 s 交付态）；套件 **×3 同 SHA `7F0C2E4C…`**（1 981 行 / 0 失败） |
| 9 | **§7 / §8** | **CLEAN** / **PASS**（3 文件；Added 1/9/19；AddedMax 73/93/88；OverWidth 0；`diff --check` 0） |
| 10 | **补丁** | `fix-B101-to-B105-91fc122a-20fe4218.patch`（提交级，累计 B-101～B-105）**4 434 B** / `FBD5FF68…` / reverse-check 0 |

---

## 1. 为什么 B 不成立（维护者质疑的回核）

### 1.1 32 个 `*-release` tag 的声明扫描（`engine` 仓 `D:\file\UnrealEngine\Release`，`git show <tag>:…UnrealString.h*`）

| 版本段 | 声明 | 导出 |
|---|---|---|
| **5.0.0 – 5.2.1** | `UE_NODISCARD static FString SanitizeFloat(double, const int32 = 1);` | ❌ **无 `CORE_API`** |
| **5.3.0 – 5.3.2** | `UE_NODISCARD static CORE_API FString SanitizeFloat(…);` | ✅（**5.3.0 才加**） |
| 5.4.0 – 5.8.4 | `[[nodiscard]] static CORE_API UE_STRING_CLASS SanitizeFloat(…);` | ✅ |

- 定义侧同样：5.0 的 `FString FString::SanitizeFloat(...)`（`Private/Containers/String.cpp:1214`）也没有导出宏；5.4+ 才是 `UE_STRING_CLASS` 模板 ＋ `CORE_API`。
- ⇒ **模块化构建（编辑器/DLL）在 5.0–5.2 上调用它会得到未解析外部符号**；而本插件支持 5.0–5.8、且 5.0 是实编点。若改成"按版本门控调用"，同一份 C# 产物会随引擎版本不同（5.3+ 6 位、5.0–5.2 9 位）—— 比保留 fork 更糟。
- 附带：5.0 的 `StripNegativeZero` 也只声明在 `String.cpp` 内（138 §2.4.1 已记），同样不可跨 TU 调用。

### 1.2 结论

**fork 必须保留**（这也是它存在的原因之一）；"对齐"只能在 fork 内做，且**不能**把精度改成 6 位后一删了之（会丢 `1e-8` 一类默认值）。

---

## 2. 精度与极小值对齐（fork 内）

### 2.1 终态实现（`TDefaultArgument.inl:5-31`）

```cpp
inline FString SanitizeFloat(const double InFloat, const int32 InMinFractionalDigits = 1)
{
	const auto Value = InFloat == 0.0 ? 0.0 : InFloat;

	const auto bIsBelowFixedPrecision = Value != 0.0 && FMath::IsNearlyZero(Value, 0.5e-6);

	const auto bIsFixedUnderflow = Value != 0.0 && FMath::IsNearlyZero(Value, 0.5e-9);

	FString TempString;

	if (bIsBelowFixedPrecision)
	{
		TempString = FString::Printf(TEXT("%.9f"), Value);
	}
	else
	{
		TempString = FString::Printf(TEXT("%.6f"), Value);
	}
	…（截断/补零四段与引擎逐字同源，未改）…
	if (bIsFixedUnderflow)
	{
		TempString = FString::Printf(TEXT("%.9g"), Value);
	}

	return TempString;
}
```

| 档位 | 条件 | 格式 | 引擎对应 |
|---|---|---|---|
| 默认 | `|x| > 0.5e-6` | `%.6f` | 引擎 `%f` = **6 位**（对齐） |
| 定点回退 | `0 < |x| ≤ 0.5e-6` | `%.9f` | 引擎无（否则 `1e-8 → 0.0`） |
| 科学回退 | `0 < |x| ≤ 0.5e-9` | `%.9g` | 引擎无 |
| 阈值写法 | —— | `FMath::IsNearlyZero(Value, tol)` | 引擎同名 helper（`UnrealMathUtility.h:415`，5.0 起可用） |

### 2.2 探针 A/B（`Saved/Retest/b104-probe.txt`，已移除）

| 输入 | B-103（9 位） | **B-104（对齐后）** | 说明 |
|---|---|---|---|
| `1e-10` | `1e-10` | `1e-10` | 同 |
| `1e-8` | `0.00000001` | `0.00000001` | ✅ 回退保住（引擎会 `0.0`） |
| `5e-10` | `0.000000001` | **`5e-10`** | 阈值由 `<` 改 `≤`（`IsNearlyZero`），边界值改走 `%g` |
| `1e-4` | `0.0001` | `0.0001` | 同 |
| `0.0` / `-0.0` | `0.0` / `0.0` | `0.0` / `0.0` | 同（负零归一仍在） |
| `0.999845028f` | `0.999845028` | **`0.999845`** | 6 位对齐（`THRESH_NORMALS_ARE_PARALLEL` 的真实来源） |
| `0.123456789` | `0.123456789` | **`0.123457`** | 6 位对齐 |
| `6e-7` | `0.0000006` | **`0.000001`** | 6 位舍入（>0.5e-6 走默认档） |
| `5e-7` | `0.0000005` | `0.0000005` | 回退档边界 |
| `1234.5678901` | `1234.5678901` | **`1234.56789`** | 6 位对齐 |

### 2.3 产物影响（静态 ＋ probe 佐证）

- `Script/UE/Proxy` 内 `double/float` 默认值穷举：**16 处** `0.00000001`（`SMALL_NUMBER`，全部走回退档 ⇒ **不变**）、**3 处** `0.999845028`（`Vector.cs` 的 `Parallel/Coincident/Coplanar`）⇒ 变 `0.999845`；其余（`0.0001`、`1.000000f` 等）**不变**。
- 即**本轮唯一的产品文本变化 = 3 行**，且是 6 位对齐的直接结果（相对误差 2.8e-8，对余弦阈值无实际影响）。

---

## 3. 一行：`RemoveAt` 门控 5.5.0 → 5.4.0

### 3.1 32-tag 证据（`UnrealString.h` / `.h.inl` 的 `RemoveAt` 声明）

| 版本 | 签名 |
|---|---|
| 5.3.0 – 5.3.2 | `RemoveAt(int32 Index, int32 Count = 1, bool bAllowShrinking = true)` |
| **5.4.0 – 5.4.4** | **`RemoveAt(int32 Index, int32 Count = 1, EAllowShrinking AllowShrinking = EAllowShrinking::Yes)`** |
| 5.5.0 – 5.5.4 | `… int32 Count, EAllowShrinking AllowShrinking = EAllowShrinking::Yes`（`Count` 无默认值） |
| 5.6.0+ | `… EAllowShrinking AllowShrinking = EAllowShrinking::Default` |

⇒ 阈值应为 **5.4.0**（原 5.5.0 偏晚）。改动后 5.4 走 `EAllowShrinking::No`，5.0–5.3 仍走 `false`。

### 3.2 UE 5.4 实编复核（`E:\UnrealEngine\UE54Check`，插件目录 junction 回本仓）

| 趟 | 子模块分支 / `TargetFramework` | 读数 |
|---|---|---|
| ① | `5.5` / net8.0 | `error NETSDK1045`（5.4 的 UBT 硬用随包 **SDK 6.0.302**）⇒ **exit 6**（与 B-30 记录一致） |
| ② | **`5.3` / net6.0**（`4a3273e`） | **exit 0**；`[8/19] Compile Module.UnrealCSharpCore.cpp`、`[16/19] Compile Module.UnrealCSharp.cpp`、`[17/19] Link UnrealEditor-UnrealCSharp.lib`、`[18/19] …dll` ⇒ **48.03 s** |
| 收口 | 已切回 `5.5`（`0f2fd13`） | 与改前一致 |

- 日志里**没有**任何来自 `TDefaultArgument.inl` 的告警/错误（引擎自带告警另计：`GetAssetRegistryTags` 等既有项）。
- ⚠️ **对上一页的一处更正**：5.4 的 `UE_ALLOWSHRINKING_BOOL_DEPRECATED` 是**空宏**（`AllowShrinking.h` 里 `//UE_DEPRECATED(...)` **被注释掉**）⇒ 旧 `false` 路径在 5.4 上**本来也不会出弃用告警**。因此这一行的价值 = **阈值与实际首可用版本一致**（不再依赖"意向弃用"的别名），**不是**"消警告"。138 §2.4.1 的"会出弃用告警"一句由本页更正。

---

## 4. 回归

| 项 | 读数 |
|---|---|
| 5.6 UBT（探针态 / 交付态） | **Succeeded**（47.53 s / 38.54 s） |
| 5.6 套件 | **1 981 行 / 0 失败 / `7F0C2E4C…`** ×3（探针态 1 轮 ＋ 交付态 2 轮；与基线逐字节相同） |
| 5.4 UBT | **exit 0**（48.03 s；子模块 `5.3`/net6.0） |
| §7 语言标准 | **CLEAN** |
| §8 风格 | **PASS**（3 文件、Added 1/9/19、AddedMax 73/93/88、OverWidth 0、`diff --check` 0） |
| 产物 manifest | ⚠️ **未取得**：`UnrealEditor-Cmd … -run=GeneratorScriptCode -Platform=Windows` 在本机 **4 s 退出、exit 1、日志无 commandlet 段、产物 0 差异**（原因未定位）⇒ 3 行产物变化为**静态扫面 ＋ probe 佐证**，非生成器实测 |
| `Script/UE/Proxy` | **不受 git 跟踪**（`git ls-files` 为空）⇒ 仓内无脏 diff；本地产物仍为旧值（3 行），待下次生成器成功运行时刷新 |

**补丁**：`Saved/Retest/fix-B101-to-B105-91fc122a-20fe4218.patch`（提交级，累计 B-101～B-105）—— **4 434 B** / SHA256 `FBD5FF6832B3AF303BD596C0FF3454473D002A7E4E30C2A1EBB5726DB6FAF533`（**提交 `9703f592`**） / reverse-check **0**；提交 6 文件 `+56/−18`（本补丁 3 文件；不含子模块指针）。

---

## 5. 台账回写

| 台账 | 动作 |
|---|---|
| [`138-轮B-103`](138-轮B-103-gen后缀依据复核与SanitizeFloat对齐引擎（2026-10-27）.md) §2.4.1 | "待取号线索"标为**已施工（B-104）**；更正"5.4 会出弃用告警" |
| [`136-轮B-101`](136-轮B-101-两个半边施工（F-CB-011极小值·F-CS4-008生成标记）（2026-10-27）.md) §4 | 补丁读数改指本页（累计版） |
| [`09-…/00`](../09-问题清单/00-推荐优先修复清单.md) §0.0 | 追加"轮 B-104"注 |
| [`README.md`](../README.md) / [`INDEX.md`](../INDEX.md) | 轮 B-104 行 |

---

## 6. 未取得 / 边界

1. **未做 5.0 实编**：B 的否决依据是 32-tag 的 `CORE_API` 头文件证据（5.0–5.2 无导出）＋ 5.3.0 才加导出这一变化本身；若要"运行期坐实 LNK2019"，需在 5.0 检查工程里临时写一行调用再编（未做）。
2. **全量生成器未取得**（§4）：3 行产物变化未由生成器实测；`Script/UE/Proxy` 不跟踪，故不影响 git 状态。
3. 5.4 实编用的是子模块 **`5.3` 分支**（net6.0，B-30 记录的映射）；构建后已切回 `5.5`（`0f2fd13`），插件仓的 `M Source/SourceCodeGenerator` 仍为改前状态。
4. 6 位档会改变**任何** 7–9 位小数的默认值（本轮语料只有 `0.999845028` 三处）；`6e-7`、`1234.5678901` 属探针语料。
5. 5.5/5.7/5.8 的 `RemoveAt` 默认参数从 `Yes` 变 `Default`（5.6 起），本仓三参调用不受影响，但**未逐版本实编**（只有 5.4/5.6 两个点）。
