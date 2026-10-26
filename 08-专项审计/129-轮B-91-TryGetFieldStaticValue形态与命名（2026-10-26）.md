# 轮 B-91 `TryGetFieldStaticValue` 形态与命名整改（单出口／去临时量／`bSucceeded`）（2026-10-26）

> **来源**：维护者对 [`91fc122a`](121-轮B-83-新增公开面清掉最后两条G7（2026-10-25）.md) 的 `FScriptDomainImpl.inl` 提三点：①「提前 `return`」不符合本文件形态（其他都是 `if` 条件式）；②`auto Value = int64{0}` 写法奇怪、没必要；③`bFound` 是否有更好命名。
> **口径**：§8.1-A（形态/命名）＋ G7 ＋ §8；**插件 2 文件、2 次提交**（`91fc122a` `+6/−10`；`91fc122a` `+1/−1`）。
> **基线**：插件 **`91fc122a`** ⇒ **`91fc122a`** ⇒ **`91fc122a`**；套件 `7F0C2E4C…`（1 980 断言）。
> **一句话结论**：三点全部成立并已改 —— 去掉提前 `return`（改 **单一出口**）、**删掉临时量**（直接把 `&OutValue` 传给 op）、`bFound` → **`bSucceeded`**（仓内 32 处先例）；顺带把同批引入的 `auto FieldValue = int64{0}` 也改成 `int64 FieldValue = 0`，全仓 `int64{0}` 写法**清零**。

---

> 🔁 **该函数名已改回 `GetFieldStaticValue`**（2026-10-27 轮 B-97 `91fc122a`）：保留旧名、实现换成"返回是否命中"；本页所有 `TryGetFieldStaticValue` 字样均指**同一位**的实现，形态/三目等结论不变。见 [`132-轮B-97`](132-轮B-97-保留GetFieldStaticValue名字换实现（2026-10-27）.md)。

## 1. 三点意见的落实

### 1.1 提前 `return` → 单一出口

**依据（同文件形态普查）**：

| 函数 | 形态 |
|---|---|
| `GetClass`（`:382-385`） | 三元 **单 `return`**，守卫在条件位 |
| `GetMethod`（`:393-396`） | 三元 **单 `return`** |
| `GetFieldStaticValue`（`:417-420`） | 三元 **单 `return`** |
| `SetFieldStaticValue`（`:403-410`） | **嵌套 `if`**，无提前 `return` |

⇒ 本文件**没有**"守卫 + 提前 `return`"的先例；`TryGetFieldStaticValue` 是唯一一处。

**改后**：

```cpp
bool SCRIPT_DOMAIN_TYPE::TryGetFieldStaticValue(const IManagedHandle InManagedClass, const FString& InName,
	                                                int64& OutValue)
{
	auto bSucceeded = false;

	if (IManagedHandleIsValid(InManagedClass) && FieldBridgeTryGetStaticValueFn != nullptr)
	{
		bSucceeded = SCRIPT_DOMAIN_INVOKE(uint8, FieldBridgeTryGetStaticValueFn, InManagedClass,
		                                  SCRIPT_DOMAIN_STRING_CAST(InName), &OutValue) != 0;
	}

	return bSucceeded;
}
```

### 1.2 `auto Value = int64{0}` → 直接传 `&OutValue`

- **删掉临时量**：op 的第三个参数本来就是 `int64*`（`typedef uint8 (*field_bridge_try_get_static_value_fn)(IManagedHandle, const uint8*, int64*)`），`&OutValue` 类型完全吻合 ⇒ 少一次写-拷贝。
- **不再有奇怪写法**：全仓 `int64{0}` 从 **2 处 → 0 处**（另一处 `FDynamicEnumGenerator.cpp` 的 `auto FieldValue = int64{0}` 同批改成 `int64 FieldValue = 0`）。
- **语义更贴 `Try`**：失败路径**不写** `OutValue`（旧版失败会把它写 0）—— 见 §2 的行为差异与安全论据。

### 1.3 `bFound` → `bSucceeded`

| 候选 | 仓内先例 | 判定 |
|---|---|---|
| **`bSucceeded`** | **32** 处（`FCSharpCompilerRunnable` 等，语义 = "这次操作成不成"） | ✅ **采用** |
| `bFound` | **2** 处（都是本轮自己的） | ✗ 无先例 |
| `bHasValue` / `bIsFound` | 0 / 0 | ✗ |

### 1.4 形态终稿（三目，B-94 `91fc122a`）

中间还试过两版：`bSucceeded` 累加（`91fc122a`）、分支内 `return` ＋ 末尾 `return false`（`91fc122a`）。维护者最终定**三目** —— 这也正是本文件 `GetClass`／`GetMethod`／`GetFieldStaticValue` 三个相邻实现的写法：

```cpp
bool SCRIPT_DOMAIN_TYPE::TryGetFieldStaticValue(const IManagedHandle InManagedClass, const FString& InName,
	                                                int64& OutValue)
{
	return IManagedHandleIsValid(InManagedClass) && FieldBridgeTryGetStaticValueFn != nullptr
		       ? SCRIPT_DOMAIN_INVOKE(uint8, FieldBridgeTryGetStaticValueFn, InManagedClass,
		                              SCRIPT_DOMAIN_STRING_CAST(InName), &OutValue) != 0
		       : false;
}
```

| 判据 | 读数 |
|---|---|
| 与 `GetFieldStaticValue`（`:417-420`）的 `?`/`:` 对齐 | **完全一致**（`?` 均在第 16 列） |
| 新增行视觉宽（TAB=4） | 最大 **93**（`:428`），远低于 110 |
| 布尔局部量 | 无（`bFound`／`bSucceeded` 均下线） |
| 语义 | 不变：命中写 `OutValue` 并返回 true；守卫失败/字段缺失返回 false 且**不写出参** |
| 回归 | 套件 **1 980 / 0 ×2 同 SHA `7F0C2E4C…`**、5.6 UBT **Succeeded**（19.54 s）、5.0 门禁 **0 error**、§7 **CLEAN** / §8 **PASS**（Added **4**、AddedMax **90**） |

> **口径说明**：§1.1 曾以"本文件无提前 `return` 先例"论证单一出口，§1.3 曾推出 `bSucceeded` —— 两者都被后续选择取代，**均不再作为结论**，仅作形态演进的记录保留。

## 2. 行为差异与安全论据

| 场景 | 改前 | 改后 |
|---|---|---|
| 句柄无效 / op 指针为空 | 提前 `return false`，`OutValue` **不动** | 跳过 `if`，`OutValue` **不动**，`return false` |
| 字段缺失（C# 返回 0） | `OutValue = 0`（临时量回填） | `OutValue` **不动** |
| 命中 | `OutValue = 值` | `OutValue = 值`（op 直接写入） |

**唯一消费者**（`FDynamicEnumGenerator::GeneratorEnumerator`）在调用前有 `int64 FieldValue = 0;` ⇒ 第三种分支（`OutValue` 不动 vs 写 0）对其**不可观测**。套件 **逐字节同 SHA** 佐证：1 980 断言、488 个枚举登记值不变。

> 这是本轮唯一的行为面变化（失败路径不再清零出参），方向上是**更正确的 `Try` 语义**；若将来有调用方依赖"失败即清零"，需显式初始化 —— 已写进 §6 边界。

## 3. 判据与回归

| 项 | 读数 |
|---|---|
| 静态 | `FScriptDomainImpl.inl` 内 `return false;` 提前出口 **0**；`int64{0}` 全仓 **0**；`bFound` 全仓 **0** |
| 运行期 | 套件 **1 980 / 0 ×2 同 SHA `7F0C2E4CD67A805030E007643F79124D6CAC37F911710DEA32D838322B246264`**（两次提交后各跑一轮，均同 SHA） |
| 5.6 UBT | **Succeeded**（19.74 s / 21.69 s） |
| 5.0 门禁 | **0 error**（两轮各 1 次） |
| §7 / §8 | 语法 **CLEAN** ／ 风格 **PASS**（`91fc122a`：1 文件、Added **6**、AddedMax **90**；`91fc122a`：1 文件、Added **1**、AddedMax **25**；`diff --check` 均 0） |

## 4. 交付

| 提交 | 内容 | 补丁 |
|---|---|---|
| 🟢 **`91fc122a`**（父 `987be131`） | `FScriptDomainImpl.inl` 单出口 ＋ 去临时量 ＋ `bSucceeded`，**1 文件 `+6/−10`** | `Saved/Retest/fix-B91-try-shape.patch` —— **1 319 B** / `75FAF9995D918081F1B6D3501C35FE51D85CB729951E0AC6C146D545F91C0448` / reverse-check 0 |
| 🟢 **`91fc122a`**（父 `987be131`） | `FDynamicEnumGenerator.cpp` 的 `int64` 初始化写法，**1 文件 `+1/−1`** | `Saved/Retest/fix-B92-fieldvalue-init.patch` —— **639 B** / `E0703E40C1D65DF81ECA6C26DD754200F9F2079A06C86DDD910BBE476198382D` / reverse-check 0 |
| 🟢 **`91fc122a`**（父 `987be131`） | 分支内 `return` ＋ 末尾 `return false`，**1 文件 `+3/−5`** | `Saved/Retest/fix-B93-try-return.patch` —— **1 138 B** / `E338488CCAD583939DE48A57A5DA6019B4401C73C82BFF6E71697AA2779AF81D` / reverse-check 0 |
| 🟢 **`91fc122a`**（父 `987be131`） | 改**三目**（终稿），**1 文件 `+4/−7`** | `Saved/Retest/fix-B94-try-ternary.patch` —— **1 190 B** / `A640C35777DA69052FF2B2245EC61EEDDB792502410E543E37AADFCF65204D04` / reverse-check 0 |

## 5. 台账回写

| 台账 | 动作 |
|---|---|
| [`08-…/121`](121-轮B-83-新增公开面清掉最后两条G7（2026-10-25）.md) | §2.4 后加"🔁 形态与命名已按 B-91 调整"指针 |
| [`08-…/128`](128-轮B-90-TryGetStaticValue规范与风格复核（2026-10-26）.md) | 其 §1 里对 `.inl` 形态的描述加同一指针 |
| [`09-…/00`](../09-问题清单/00-推荐优先修复清单.md) / [`README.md`](../README.md) / [`INDEX.md`](../INDEX.md) | 轮次注 ／ 行 ／ 条目 |

## 6. 边界

1. **失败路径不再清零出参**（§2）—— 唯一消费者已预初始化，套件同 SHA；外部若依赖清零需自行初始化。
2. 语义与 ABI 未变（仍 `bool` / `uint8`，仍"命中才写值"）；Mono/LeanCLR 未实跑，仅编译覆盖。
3. `TryGetStaticValue` 返回 `0`（字段缺失）的运行期分支仍未构造（同 B-90 §6）。

