# 轮 B-87 改名 `IsInitialized()` ＋ 单 `return` 实现（宏内联）(2026-10-26)

> **来源**：维护者对 [`91fc122a`](124-轮B-86-谓词并入FReflectionRegistry（HasAllCoreClasses）（2026-10-26）.md) 的取舍 —— 「`IsInitialized` 吧，然后函数内部实现，就用一个 `return` 就行，处理好宏就行」。
> **口径**：G7 ＋ §8.5（E2 只读访问器）＋ §8；**插件 5 文件 `+38/−43`**（纯改名 ＋ 函数体重写）。
> **基线**：插件 **`91fc122a`** ⇒ 提交 **`91fc122a`**；套件基线 `3A2D527D…`（1 982 断言）。
> **一句话结论**：谓词改名 **`FReflectionRegistry::IsInitialized() const`**，函数体从"局部量累加"改为**单个 `return` 表达式**，三个版本门控 `#if` **内联在表达式里**（`&&` 项形式，无需尾随分号）；**三种宏组合实测编译**（5.0.3 全关 / 5.6.0 全开 / 探针混合），套件仍 **1 982 / 0 ×2 同 SHA**。

---

## 1. 改名与调用点

| 位置 | 旧 | 新 |
|---|---|---|
| `FReflectionRegistry.h:30` | `bool HasAllCoreClasses() const;` | `bool IsInitialized() const;` |
| `FReflectionRegistry.cpp:986` | `HasAllCoreClasses()` | `IsInitialized()` |
| `FCoreCLRDomain.cpp:210` / `FMonoDomain.cpp:169` | `if (…HasAllCoreClasses() == false)` | `if (…IsInitialized() == false)` |
| `FLeanCLRDomain.cpp:799` | `return …HasAllCoreClasses();` | `return …IsInitialized();` |

> **语义口径**：`IsInitialized()` 回答"**`Initialize()` 之后 32 个核心类反射是否都解析成功**"（不是"`Initialize()` 是否被调用过" —— 注册表不跟踪该状态，值完全由 32 个 `FClassReflection*` 推导）。

> 🔁 **顺序已由轮 B-88 修正**：`91fc122a` 版把 `GetTFieldPathClass()` 移到末尾（为让分号落在真实行），与 `.h` 声明顺序差 2 项；`91fc122a` 已恢复**逐位一致**的声明顺序（代价：`#endif` 后一行独立 `;`，见 [`126-轮B-88`](126-轮B-88-IsInitialized按声明顺序（2026-10-26）.md)）。本页以下代码块为**当时**形态。

## 2. 函数体：单个 `return` ＋ 宏内联

```cpp
bool FReflectionRegistry::IsInitialized() const
{
	return GetUtilsClass() != nullptr
	       && GetObjectClass() != nullptr
	       /* …29 个无条件项… */
	       && GetStringClass() != nullptr
#if UE_F_UTF8_STR_PROPERTY
	       && GetUtf8StringClass() != nullptr
#endif
#if UE_F_ANSI_STR_PROPERTY
	       && GetAnsiStringClass() != nullptr
#endif
	       && GetTextClass() != nullptr
	       /* …容器与指针包装… */
	       && GetTArrayClass() != nullptr
#if UE_F_OPTIONAL_PROPERTY
	       && GetTOptionalClass() != nullptr
#endif
	       && GetTFieldPathClass() != nullptr;
}
```

**两处刻意的结构选择**：

1. **`#if` 作为"`&&` 项"内联**，而不是把三条门控拆成语句 ⇒ 不需要 `bComplete` 局部量，也不需要 `#endif` 后再补分号。
2. **把 `GetTFieldPathClass()` 放在最后**（`GetTOptionalClass()` 之前移）⇒ 表达式的**末项恒为无条件项**，分号永远落在真实语句行上，**不出现孤立的 `;` 行**。代价只是 Optional 与 FieldPath 的求值顺序对调（两者都是 `const` 空判，无语义差异）。

## 3. 宏组合覆盖（"处理好宏"的证据）

三个门控的定义（`Source/CrossVersion/Public/UEVersion.h`）：`UE_F_OPTIONAL_PROPERTY = UE_VERSION_START(5,3,0)`（`:102`）、`UE_F_UTF8_STR_PROPERTY = (5,6,0)`（`:158`）、`UE_F_ANSI_STR_PROPERTY = (5,6,0)`（`:160`）。

| 组合 | 来源 | 编译读数 |
|---|---|---|
| **全关**（Optional/Utf8/Ansi 都不在） | 5.0.3 门禁（5.0 < 5.3 < 5.6） | **Succeeded**（79.42 s，0 error） |
| **全开** | 5.6.0 构建（≥5.6 ⇒ 三条都在） | **Succeeded**（41.36 s） |
| **混合**（Optional 开、Utf8/Ansi 关 = 5.3–5.5 的真实配置） | **临时探针**：把 `:158/:160` 临时改成 `UE_VERSION_START(9,9,9)` 后用 5.6 编译 | **Succeeded**（47.34 s）→ 探针已回退（`git diff` = 0、`(9, 9, 9)` 残留 = 0）→ 重新构建（45.86 s）→ 再跑套件 **同 SHA** |

> 其余 5 种组合（中间门控开/关、Optional 单独关）与上表**结构同形**（每个 `#if` 独立插入一个 `&&` 项，末项恒为 `GetTFieldPathClass()`）⇒ 不再逐个探针。

## 4. 判据与回归

| 项 | 读数 |
|---|---|
| 静态 | `HasAllCoreClasses` 全仓 **0** 命中；`IsInitialized()` 声明 `.h:30`、定义 `.cpp:986`、三域各 1 处调用 |
| 静态（接口冲突） | `FReflectionRegistry` 不继承 `IScriptDomain`，与域侧的 `IsInitialized() override` **无重载/覆盖关系**（同名不同类，仅语义同向） |
| 运行期 | 套件 **1 982 / 0 ×2 同 SHA `3A2D527D6F839FC3E59D84AE5CA87E76BFFBB42DE0A23AA1B25FAF715E434769`**（改名前后逐字节相同）；探针回退后复跑 **同 SHA** |
| 5.6 UBT | **Succeeded**（41.36 s；探针 47.34 s；回退后 45.86 s） |
| 5.0 门禁 | **0 error**（79.42 s）＋ 复跑 **0 error**（82.47 s） |
| §7 语法 | `b29-xversion-lint -Mode Syntax -DiffBase 91fc122a` ⇒ **CLEAN** |
| §8 风格 | `b30-style-lint -Base 91fc122a` ⇒ **PASS**（5 文件、Added **1/1/1/34/1**、Trailing 0、SpaceFail 0、OverWidth 0、AddedMax **48**、AlignTS 31（`&&` 继行对齐，合法）、`diff --check` 0）；无格式化配置 ⇒ 不跑格式化器 |

## 5. 交付

| 项 | 内容 |
|---|---|
| 补丁 | `Saved/Retest/fix-B87-isinitialized.patch` —— **6 153 B** / SHA256 **`A372A947B859C32C294F72E3C2AA6BD88750CA7AF6CBAB53F49EB002F300B64C`** / LF / 无 BOM / reverse-check 0 |
| 提交 | 🟢 **`91fc122a`**（父 `987be131`，subject `Reflection Registry Is Initialized`，**5 文件 `+38/−43`**） |
| 提交树 | 5 文件 `git diff HEAD` **0 行** |

## 6. 台账回写（本轮）

| 台账 | 动作 |
|---|---|
| [`08-…/124`](124-轮B-86-谓词并入FReflectionRegistry（HasAllCoreClasses）（2026-10-26）.md) | 标题/§1 加"🔁 已改名（B-87 → `IsInitialized()`）"指针（历史读数保留） |
| [`08-…/119`](119-轮B-81-G7裁定落地（F-REFL-004零例外与E1边界屏障）（2026-10-25）.md) | 既有"🔁 已被 B-86 重构"注追加"→ B-87 改名 `IsInitialized()`" |
| [`09-…/00`](../09-问题清单/00-推荐优先修复清单.md) | §0.0 新增「轮 B-87」注 |
| [`README.md`](../README.md) / [`INDEX.md`](../INDEX.md) | 轮次行 ＋ 本报告条目 |
| [`09-…/01`](../09-问题清单/01-全局问题总表.md) | **不改**（`F-REFL-004` 处置不变） |

## 7. 边界

1. **纯改名 ＋ 表达式重排**，行为零变化（套件同 SHA 即证）。
2. `IsInitialized() == false` 的**运行期分支仍未构造**（同 B-81/B-86：需要某个核心类真查不到，会连带打断后续路径）。
3. 三种宏组合已实测；其余组合按结构同形推断，未逐个探针。
4. 名字与域侧 `IScriptDomain::IsInitialized()` 同形但**语义不同层**（域 = 域整体是否可用；注册表 = 32 个核心类是否解析成功）—— 这是维护者选定名带来的唯一代价，已在 §1 用"语义口径"一行固定。
