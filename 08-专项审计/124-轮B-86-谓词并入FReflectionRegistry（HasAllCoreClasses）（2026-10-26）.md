# 轮 B-86 重构：`HasAllCoreClasses()` 并入 `FReflectionRegistry`（2026-10-26）

> **来源**：维护者复核 [`91fc122a`](119-轮B-81-G7裁定落地（F-REFL-004零例外与E1边界屏障）（2026-10-25）.md) 指出「不该新增 `Private/Domain/FReflectionRegistryValidator.h`，直接在 `FReflectionRegistry` 里边实现，并换个更合适的函数名」。
> **口径**：G7 ＋ §8.5（E2 只读访问器）＋ §8；**插件 6 文件 `+54/−60`**（删 1 个私有头 52 行，净 −6 行）。
> **基线**：插件 **`91fc122a`** ⇒ 提交 **`91fc122a`**；套件基线 `3A2D527D…`（1 982 断言）。
> 🔁 **本页的函数已在轮 B-87 改名**：`HasAllCoreClasses()` → **`IsInitialized()`**（`91fc122a`，见 [`125-轮B-87`](125-轮B-87-改名IsInitialized与单return实现（2026-10-26）.md)）；本页以下读数保留为**当时**的交付记录。

> **一句话结论**：删掉自由函数私有头，把判定做成 `FReflectionRegistry` 的**成员谓词** `bool HasAllCoreClasses() const`（命名取"**是否含有**全部核心类"语义 → 仓内 `Has*` 族）；顺带把 `GetTScriptInterfaceClass()` 补 `const`（它本来就只读，是它挡住了谓词的 `const`）。行为零变化：套件 **1 982 / 0 ×2 同 SHA**。

---

## 1. 设计变更

| 项 | 旧（`91fc122a`） | 新（`91fc122a`） |
|---|---|---|
| 载体 | 新私有头 `Private/Domain/FReflectionRegistryValidator.h` ＋ 自由函数 `IsReflectionRegistryComplete()` | `FReflectionRegistry` 的**成员函数** `bool HasAllCoreClasses() const`（`.h:30` 声明、`.cpp:986` 定义） |
| 域侧 | `#include "Domain/FReflectionRegistryValidator.h"` ＋ 直接调用自由函数 | 无新 include；`FReflectionRegistry::Get().HasAllCoreClasses()` |
| 顺序依赖 | 自由函数在头里 include `Reflection/FReflectionRegistry.h`（头文件互相牵引） | 无（调用方本来就 include 了 registry） |
| 可发现性 | 一个"藏在 Domain 目录里的全局函数" | 与 `GetXxxClass()` 同类的成员，IDE 补全可见 |
| 行数 | ＋52（新文件） | ＋49（.cpp）＋1 声明 −52（删文件） |

**为什么不再用自由函数**：本仓手写源码**不用匿名 namespace**，自由函数要么放 `.inl`/`Private` 头（就是旧方案，等于给一个一次性判定新增一个头文件），要么登记到公共头（更重）。成员函数是唯一"零新增文件"的形态。

## 2. 命名：为什么是 `HasAllCoreClasses()`

| 候选 | 仓内先例 | 判定 |
|---|---|---|
| **`HasAllCoreClasses()`** ✅ | `Has*` 族 **8** 处（`HasFunctionDescriptor`、`HasFunctionDefaultParam`…），语义正是"**是否含有**某物" | **采用** |
| `IsCoreComplete()` | `Is*` 族 **122** 处 | 次选：`Is*` 在仓内是"对象当前处于某状态"（`IsEmpty`/`IsValidIndex`/`IsCompiling`），而这里是"**是否含齐 32 项**"，`Has*` 更准 |
| `AreAllCoreClassesRegistered()` | `Are*` 族 **0** 处 | 否（新词形） |
| `IsInitialized()` | —— | 否：会被读成"`Initialize()` 调用过没有"，与"32 个类是否解析成功"不是一回事（且本类不跟踪状态） |

调用点读作：`if (FReflectionRegistry::Get().HasAllCoreClasses() == false) { bIsInitialized = false; }`。

## 3. 顺带修的一处 `const`

`GetTScriptInterfaceClass()` 原本**没有** `const`（`FReflectionRegistry.h` 旧行 `FClassReflection* GetTScriptInterfaceClass();`），但它的函数体就是 `return TScriptInterfaceClass;`（`.cpp:1123`）—— 这是**既有遗漏**。它挡住了谓词的 `const`（32 个检查里只有这一个是非 const getter）。

处置：给该方法补 `const`（`.h:69`、`.cpp:1123`）。**影响面**：非虚、无 override、仓内所有调用点都在非 const 对象上 ⇒ **源码兼容**；插件整体重编，ABI 影响不外溢。

> **E2 口径**：`HasAllCoreClasses()` 是新增的**只读**成员（`const`、无副作用、不分配、只读既有成员），有 **3 个真实消费者**（三个脚本域）⇒ 落在 §8.5 E2 内；它不是"无消费者的新 API"。

## 4. 判据

| 项 | 读数 |
|---|---|
| 静态：旧载体残留 | `IsReflectionRegistryComplete` **0** 处、`FReflectionRegistryValidator` **0** 处、文件不存在 |
| 静态：三域调用点 | `HasAllCoreClasses()` 各 **1** 处（`FCoreCLRDomain.cpp` / `FMonoDomain.cpp` / `FLeanCLRDomain.cpp`），旧 `#include` 已全部移除 |
| 静态：谓词位置 | `.h:30`（`GetClass` 组之后）／`.cpp:986`（`GetUtilsClass` 定义之前，紧邻它使用的那些 getter） |
| 运行期 | 套件 **1 982 断言 / 0 失败 ×2 同 SHA `3A2D527D6F839FC3E59D84AE5CA87E76BFFBB42DE0A23AA1B25FAF715E434769`** —— 与重构前**逐字节相同**（健康机器上恒 true，行为中性） |

> **未取得**：`HasAllCoreClasses() == false` 的运行期读数仍未构造（同 B-81 的边界：需要"某个核心类真查不到"，会连带打断后续路径）。

## 5. 回归

| 项 | 读数 |
|---|---|
| 5.6 UBT `UnrealCSharpTestEditor Win64 Development` | **`Result: Succeeded`**（52.49 s） |
| 5.0 门禁 `UE50CheckEditor` | **0 error**（91.39 s）＋ 复跑 **`Target is up to date`**（69.91 s） |
| `-game` 套件 | **1 982 / 0 ×2 同 SHA `3A2D527D…`**；`LogUnrealCSharp: Error` **3**（既有判据）/ assert·fatal **0** |
| §7 语法 | `b29-xversion-lint -Mode Syntax -DiffBase 91fc122a` ⇒ **CLEAN** |
| §8 风格 | `b30-style-lint -Base 91fc122a` ⇒ **PASS**（5 文件、Added **1/1/1/48/3**、Trailing 0、SpaceFail 0、OverWidth 0、AddedMax **71**、AlignTS 28（`&&` 继行对齐，合法）、`diff --check` 0）；无格式化配置 ⇒ 不跑格式化器 |

## 6. 交付

| 项 | 内容 |
|---|---|
| 补丁 | `Saved/Retest/fix-B86-registry-predicate.patch` —— **8 718 B** / SHA256 **`1D1309A451B7F6E5D8064A6FE3A6E091D624B785E703BC1B3CFFBFAB74EFAA6F`** / LF / 无 BOM / reverse-check 0 |
| 提交 | 🟢 **`91fc122a`**（父 `987be131`，subject `Reflection Registry Has All Core Classes`，**6 文件 `+54/−60`**，含 `delete mode … FReflectionRegistryValidator.h`） |
| 提交树 | `git diff HEAD -- Source/UnrealCSharpCore` **0 行** |

## 7. 台账回写（本轮）

| 台账 | 动作 |
|---|---|
| [`08-…/119`](119-轮B-81-G7裁定落地（F-REFL-004零例外与E1边界屏障）（2026-10-25）.md) | §1.1 加"🔁 已被轮 B-86 重构"指针（历史读数保留不改） |
| [`09-…/00`](../09-问题清单/00-推荐优先修复清单.md) | §0.0 新增「轮 B-86」注 |
| [`README.md`](../README.md) / [`INDEX.md`](../INDEX.md) | 轮次行 ＋ 本报告条目 |
| [`09-…/01`](../09-问题清单/01-全局问题总表.md) | **不改**（`F-REFL-004` 的处置不变；实施形态变化记在本页） |

## 8. 边界

1. 行为零变化（套件同 SHA）；`HasAllCoreClasses()` 的 **false 分支**运行期未取证（同 B-81）。
2. `GetTScriptInterfaceClass()` 补 `const` 是**源码兼容**但**改了既有成员签名**（mangling 变化）⇒ 只有当它与外部二进制混链时才有影响；本插件整体重编，不涉及。
3. 三域仍是 CoreCLR 实跑、Mono/LeanCLR 只编不跑（沿用前几轮边界）。
4. 未跑 PIE / 打包版 / 5.8 实编点（本轮只动 `UnrealCSharpCore` 的私有/公共头与域实现，5.0 门禁已覆盖最低版编译）。
