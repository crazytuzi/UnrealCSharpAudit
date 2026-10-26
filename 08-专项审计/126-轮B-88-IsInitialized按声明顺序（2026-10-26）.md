# 轮 B-88 `IsInitialized()` 严格按声明顺序（2026-10-26）

> **来源**：维护者问「`IsInitialized` 按照 `FReflectionRegistry` 中的顺序了吗」—— 复核结论：**没有**，`B-87` 把 `GetTFieldPathClass()` 挪到了末尾（`GetTOptionalClass()` 之前），只有这一处偏离。
> **口径**：G7 ＋ §8；**1 文件 `+2/−1`**（纯顺序 ＋ 分号位置）。
> **基线**：插件 **`91fc122a`** ⇒ 提交 **`91fc122a`**；套件基线 `3A2D527D…`。
> **一句话结论**：恢复**与 `.h` 声明逐位一致**的 32 项顺序（`… → TArray → TFieldPath → [Optional]`），代价是 `#endif` 后需要一行**独立的分号**（对齐到 `&&` 列）。

---

## 1. 复核结果（改前）

| 位置 | 头文件声明（`FReflectionRegistry.h:31-99`） | B-87 实现 | 判定 |
|---|---|---|---|
| 1–30 | `Utils → … → TArray` | 相同 | ✅ 逐位一致 |
| 31 | `GetTFieldPathClass()` | `GetTOptionalClass()` | ❌ 对调 |
| 32 | `GetTOptionalClass()`（`#if UE_F_OPTIONAL_PROPERTY`） | `GetTFieldPathClass()` | ❌ 对调 |

> B-87 对调的理由是"让表达式末项恒为无条件项，分号永远落在真实语句行"。这属于**为格式牺牲了与声明的一致性** —— 本轮按维护者要求改回声明顺序。

## 2. 改后实现（尾部）

```cpp
	       && GetTMapClass() != nullptr
	       && GetTSetClass() != nullptr
	       && GetTArrayClass() != nullptr
	       && GetTFieldPathClass() != nullptr
#if UE_F_OPTIONAL_PROPERTY
	       && GetTOptionalClass() != nullptr
#endif
	       ;
}
```

- 32 项顺序与 `.h` **逐位一致**（脚本对读：`header=32 / func=32`，0 处 mismatch）。
- 分号单独成行（`\t` ＋ 7 空格，与 `&&` 列对齐）；这是"声明顺序 ＋ 单 `return` ＋ 末尾有条件项"三者并存的唯一写法（`#endif` 之后不能再拼接 token）。
- 仍然**只有一个 `return`**，没有局部量。

## 3. 宏组合覆盖（重跑）

| 组合 | 方式 | 读数 |
|---|---|---|
| 全开（5.6.0） | 5.6 UBT | **Succeeded**（27.29 s） |
| 全关（5.0.3） | 5.0 门禁 | **0 error**（33.12 s） |
| 混合（Optional 开、UTF8/ANSI 关 = 5.3–5.5） | 探针（`UE_VERSION_START(9,9,9)`）→ 5.6 编译 | **Succeeded**（41.60 s）→ 探针已回退（`git diff` = 0）→ 重建 50.02 s |

## 4. 判据与回归

| 项 | 读数 |
|---|---|
| 顺序 | 脚本对读：前 32 项与头文件**逐位一致**（0 mismatch） |
| 套件 | **1 982 / 0 ×2 同 SHA `3A2D527D6F839FC3E59D84AE5CA87E76BFFBB42DE0A23AA1B25FAF715E434769`** |
| 5.6 UBT | **Succeeded**（27.29 s；探针 41.60 s；回退后 50.02 s） |
| 5.0 门禁 | **0 error**（33.12 s） |
| §7 / §8 | 语法 **CLEAN** ／ 风格 **PASS**（1 文件、Added **2**、AlignTS 2、AddedMax **42**、`diff --check` 0） |

## 5. 交付

| 项 | 内容 |
|---|---|
| 补丁 | `Saved/Retest/fix-B88-order.patch` —— **761 B** / SHA256 **`C006442E276B58DF238B083A1410F94CA24BB60CA2409682F4F30B6FBF52BDA0`** / LF / 无 BOM / reverse-check 0 |
| 提交 | 🟢 **`91fc122a`**（父 `987be131`，subject `Reflection Registry Is Initialized Declaration Order`，**1 文件 `+2/−1`**） |

## 6. 台账回写

| 台账 | 动作 |
|---|---|
| [`08-…/125`](125-轮B-87-改名IsInitialized与单return实现（2026-10-26）.md) | §2 加"🔁 顺序已由 B-88 修正"指针 |
| [`09-…/00`](../09-问题清单/00-推荐优先修复清单.md) | §0.0 追加「轮 B-88」半句 |
| [`README.md`](../README.md) / [`INDEX.md`](../INDEX.md) | 轮次行 ＋ 本报告条目 |

## 7. 边界

1. 纯顺序／格式改动，行为零变化（套件同 SHA）。
2. 三种宏组合重跑通过；其余组合结构同形。
3. `;` 独立成行是本形态的**固有代价**（`#endif` 后不能拼接 token）；若将来更看重"无独立分号行"，可退回 B-87 的对调写法（行为等价，只是与声明顺序差 2 项）。
