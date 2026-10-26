# 轮 B-90 复核 `TryGetStaticValue` 实现：规范/风格核查 ＋ 1 处视觉超宽修复（2026-10-26）

> **来源**：维护者「再看下 `TryGetStaticValue` 相关的实现是不是都满足项目规范、代码风格」。
> **范围**：提交 [`91fc122a`](121-轮B-83-新增公开面清掉最后两条G7（2026-10-25）.md) 的 **8 个存活文件**（`FText.ToBufferString` 半边已被 [`B-89`](127-轮B-89-撤回ToBufferString（2026-10-26）.md) 撤回，不在本页范围）。
> **结论**：**语义与规范满足**（E1 单站点合规、G7 其余 0、Q2 落在既有值域、§7 零门控）；**风格 1 处不合格已修**（`FDynamicEnumGenerator.cpp` 的判据行按 TAB=4 计为 **113 列**，装置因按 raw 字符计宽未拦住）；**记录 1 处缺口已补**（§8.5 要求 A 轴写机械判据，`121` 只正文提及 E1）。
> **交付**：提交 **`91fc122a`**（1 文件 `+3/−1`）＋ `121` 补 §2.4 A 轴块。

---

## 1. A 轴（项目规范）

| 检查项 | 读数 | 判定 |
|---|---|---|
| G7 ① 注释 | 新增 110 行里 `^\s*(//\|/\*\|\*)` 命中 **1** ⇒ **假阳性**（`*OutValue = FieldToLong(...)`，C# 指针解引用以 `*` 开头）；人工复核真实新增注释 **0** | ✅ |
| G7 ② 日志 | `UE_LOG`/`Console.`/`Debug.Write` **0** | ✅ |
| G7 ③ `throw` | **0** | ✅ |
| G7 ④ `ensure`/`check` | **0** | ✅ |
| G7 ⑤ `try-catch` | **1 个站点**（`try` ＋ `catch (Exception)` 两行）⇒ 见下"E1 判定" | ✅ **E1** |
| 公开面 | `IScriptDomain::TryGetFieldStaticValue`（纯虚）＋ `Op(FieldBridgeTryGetStaticValue, …)` ＋ `FUNCTION_FIELD_BRIDGE_TRY_GET_STATIC_VALUE` ＋ C# `TryGetStaticValue` | ⚠️ **维护者特批**（不是 E1/E2/E3 任一条，属 121 §0 的"新增公开面"口径）；边界（外部自实现 `IScriptDomain` 编译期破坏）已在 121 §6 记录 |
| Q2 失败收敛 | C++ 返回 `false`；C# 边界返回 `0`（既有失败值）；`.inl` 未命中时 `OutValue = Value` 写 `0`（确定性） | ✅ 未新增错误码 |

> 🔁 **该表两行读数已过时**：`FScriptDomainImpl.inl` 的提前 `return` 与 `auto Value = int64{0}` 已按 [B-91](129-轮B-91-TryGetFieldStaticValue形态与命名（2026-10-26）.md) 改为**单出口 ＋ `bSucceeded` ＋ 直接传 `&OutValue`**（`91fc122a`），消费点初始化改 `int64 FieldValue = 0`（`91fc122a`）。

| 形态（单一出口） | `if (Field != null) { *OutValue = …; return 1; } return 0;`；`.inl` 为 `if (守卫) return false; … return bFound != 0;` | ✅ |
| 局部量直接声明 | `auto Value = int64{0};`（`.inl:433`）、`auto FieldValue = int64{0};`（消费点）、`const auto bFound = …` | ✅ 与同文件同形 |
| 命名 | C++ `TryGetFieldStaticValue` ↔ C#/宏 `TryGetStaticValue`，与既有 `GetFieldStaticValue`↔`GetStaticValue` **同构**；私有实现 `TryGetStaticValueInternal` 与同文件 `SetStaticValueInternal` 等同形 | ✅ |
| §7 跨版本 | 本轮新增调用全是**仓内**符号（`SCRIPT_DOMAIN_INVOKE` 走已解析的函数指针），**零新增引擎 API** ⇒ 零门控；5.0 门禁实编覆盖 | ✅ |

**E1 判定（§8.5）**：`FieldBridge.cs:35-46` 的 `[UnmanagedCallersOnly] TryGetStaticValue` —— ① `catch` 内无日志/断言/`throw`/新 API（`Console`/`Log`/`throw` 命中 **0**）；② 返回 `0` 落在既有失败值；③ **只有一层**、无 `when` 过滤；④ `try` 覆盖**整个**函数体（函数体只有 `return TryGetStaticValueInternal(...)`）⇒ 四条全中。同文件另两处 `catch`（`:17`/`:29`）是**既有**站点，本轮未新增。

## 2. B 轴（代码风格）—— 装置读数与视觉读数的分歧

### 2.1 装置（提交对提交）

`b30-style-lint.ps1 -Base 91fc122a -Target 91fc122a` ⇒ **PASS**：12 文件、Trailing **0**、SpaceFail **0**、OverWidth **0**、TabInCs **0**、`diff --check` **exit 0**、EOL **CRLF**、BOM **no**；`AddedMax` 最大 **107**（`FScriptDomainImpl.inl`）。

### 2.2 视觉（TAB = 4 列）—— 发现 1 行超限

装置用 `$_.Length` 计宽（**TAB 记 1 列**）；按编辑器/UE 习惯 **TAB = 4 列**重算，新增行里有 **1 行 = 113 列**：

```
FDynamicEnumGenerator.cpp:291
    if (ScriptDomain->TryGetFieldStaticValue(InClassReflection->GetManagedClass(), Name, FieldValue))
```

⇒ 按 §8.1-B"行宽 ≤ 110"的**本意**（列）这是超标；装置没拦住是因为 TAB 只算 1。**本轮修复**（`91fc122a`）：

```cpp
	const auto ManagedClass = InClassReflection->GetManagedClass();   // 提到循环外
	…
				if (ScriptDomain->TryGetFieldStaticValue(ManagedClass, Name, FieldValue))
```

- 行宽 **113 → 96**（视觉），全函数再无一超 110 的行走；
- 顺带省掉**每轮循环**重复调用的 `GetManagedClass()`。

> **口径提醒（新发现，未擅自改装置）**：`b30-style-lint.ps1` 的 `OverWidth` 只按 raw 字符计宽，对**深层缩进**的长行会漏报（本轮漏了 1 行）。若要把装置改成 TAB=4，历史各轮的 `AddedMax/OverWidth` 读数会整体变化，需要重跑并改 `_CONVENTIONS` 的口径 —— **留给维护者决定**，本轮只登记。

## 3. C 轴（格式化）

仓内无 `.clang-format`/`.editorconfig`/`.gitattributes` ⇒ **不跑格式化器**；本轮 12 文件无"与语义无关的重排"混入（装置 `files: 12 changed` 与提交面一致）。

## 4. 修复与回归

| 项 | 读数 |
|---|---|
| 修复提交 | 🟢 **`91fc122a`**（父 `987be131`，subject `Dynamic Enum Generator Managed Class Local`，1 文件 `+3/−1`） |
| 补丁 | `Saved/Retest/fix-B90-enumgen-width.patch` —— **1 042 B** / SHA256 **`15535462A836F5ABAE3B7951415515691BB2592E4BF0225955025F4FCFB3DC4D`** / LF / 无 BOM / reverse-check 0 |
| 套件 | **1 980 / 0 ×2 同 SHA `7F0C2E4C…`**（与修复前逐字节相同 ⇒ 纯形态修正） |
| 5.6 UBT | **Succeeded**（20.02 s） |
| 5.0 门禁 | **0 error** |
| §7 / §8 | 语法 **CLEAN** ／ 风格 **PASS**（1 文件、Added **3**、AddedMax **77**、`diff --check` 0） |

## 5. 记录缺口已补

§8.5 的**记录要求**：用 E1/E2/E3 时，报告的「交付前格式与规范核查」**A 轴**必须写明"用的哪一条 ＋ 机械判据读数"。`121` 只在 §2.2 改动面表里写了"带 E1 一层屏障"，**没有 A 轴机械表** ⇒ 本轮在 `121` 补 **§2.4 A 轴（2026-10-26 补录）**，含上表读数（注释假阳性说明、`catch` 计数 1、`Console/Log/throw` 0、E1 四条件）。

## 6. 结论（可直接引用的判据）

1. **规范**：满足（E1 唯一 try-catch 站点合规；其余 G7 项 0；Q2 既有值域；§7 零门控）；**唯一的口径例外**是"新增公开面"（维护者特批，非 E1/E2/E3），边界已记录。
2. **风格**：装置 PASS；视觉口径 1 处超限已修（`91fc122a`）。
3. **装置口径**：`OverWidth` 按 raw 计宽会漏报深缩进长行 —— 已登记，未擅改。
4. 未取得项与前几轮一致：`TryGetStaticValue` 返回 `0` 的运行期分支未构造；Mono/LeanCLR 未实跑。
