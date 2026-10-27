# 轮 B-110：`CopyValue` 所有权 sweep —— `F-PROP-002` 结案（契约级，无泄漏）（2026-10-27）

> **来源**：维护者「按照你的推荐，依次开始」⇒ 落地 [`142-轮B-107`](142-轮B-107-T3第十次选片与立项建议（2026-10-27）.md) 回执 **Q3=A**（先做 0 代码的 9 类所有权 sweep，再决定立项/结案）。
> **口径**：本页 **0 代码改动、0 探针、未编译未跑套件**；`file:line` 回 HEAD **`9703f592`** 工作树（B-109 改动不涉及这些文件）。
> **一句话结论**：`IsCopyValueProperty` 覆盖的 **11 个描述符**（FName/Struct/Array/**Str**/**Utf8Str**/**AnsiStr**/Text/Map/Set/FieldPath/Optional）的 `Get(Src, Dest, FReturn)` **全部把 `Src` 交给"接管并释放"的注册表**（模板参数 `IsNeedFree=true` / `bNeedFreeData=true`）⇒ `CopyValue` 的 `FMemory::Malloc` **没有泄漏面**；`F-PROP-002` 的剩余只是**契约问题**（同名虚函数三种所有权）⇒ **结案（契约级 · 决定不做）**，登记触发条件。

---

## 1. 方法

1. `IsCopyValueProperty`（`FManagedFunctionDescriptor.cpp:125-146`）列出 `CopyValue` 的**全部消费类型**（9 个 `IsA<>` ＋ 2 个条件编译 = **11 个**）；
2. 对每个类型的 `Get(void* Src, void** Dest, FPropertyArgument::FReturn)` 取实现体；
3. 判定 `Src`（= `CopyValue` 返回的 `Malloc` 指针）是否被**接管释放**：看注册 API 的模板参数/构造参数**名**（`IsNeedFree` / `InbNeedFreeData`）与析构路径。

---

## 2. 逐类证据（HEAD `9703f592`）

| # | 类型 | `Get(FReturn)` 的接管调用 | 参数语义 | 接管？ |
|---|---|---|---|---|
| 1 | `FName` | `AddStringReference<FName, true, false>(Class, Object, Src)`（`FNamePropertyDescriptor.cpp:30-31`） | 模板 `template <typename T, auto IsNeedFree, auto IsMember>`（`FCSharpEnvironment.h:204`）⇒ `IsNeedFree=true` | ✅ |
| 2 | `FStr` | `AddStringReference<FString, true, false>`（`FStrPropertyDescriptor.cpp:30-31`） | 同上 | ✅（10-08 已证） |
| 3 | `FUtf8Str` | `AddStringReference<FUtf8String, true, false>`（`:31-32`） | 同上 | ✅ |
| 4 | `FAnsiStr` | `AddStringReference<FAnsiString, true, false>`（`:31-32`） | 同上 | ✅ |
| 5 | `FText` | `AddStringReference<FText, true, false>`（`:30-31`） | 同上 | ✅ |
| 6 | `FStruct` | `AddStructReference<true>(Property->Struct, Src, Object)`（`FStructPropertyDescriptor.cpp:35`） | `template <auto IsNeedFree>`（`FCSharpEnvironment.h:110`）⇒ `true` | ✅ |
| 7 | `FArray` | `NewWeakRef(Src, true)`（`FArrayPropertyDescriptor.cpp:12`）→ `new FArrayHelper(Property->Inner, InAddress, bIsCopy, false)`（同文件 `:19-20`） | `FArrayHelper(FProperty*, void* InData, bool InbNeedFreeData, bool InbNeedFreeProperty, …)`（`FArrayHelper.h:10-12`）⇒ 第 3 参 = `bIsCopy` = **true** | ✅ |
| 8 | `FMap` | `NewWeakRef(Src, true)`（`FMapPropertyDescriptor.cpp:12`）→ `FMapHelper(…, bIsCopy, false)` | 同构（`FMapHelper` 同签名） | ✅ |
| 9 | `FSet` | `NewWeakRef(Src, true)`（`FSetPropertyDescriptor.cpp:12`）→ `FSetHelper(…, bIsCopy, false)` | 同构 | ✅ |
| 10 | `FFieldPath` | `AddFieldPathReference<true, false>(Class, Object, Src)`（`FFieldPathPropertyDescriptor.cpp:30-31`） | `template <auto IsNeedFree, auto IsMember>`（`FCSharpEnvironment.h:219`）⇒ `IsNeedFree=true` | ✅ |
| 11 | `FOptional` | `new FOptionalHelper(Property, Src, true, false)` ＋ `AddOptionalReference<FOptionalHelper, false>(…)`（`FOptionalPropertyDescriptor.cpp:32-35`） | `FOptionalHelper(FOptionalProperty*, void* InData, bool InbNeedFreeData, bool InbNeedFreeProperty)`（`FOptionalHelper.h:10-11`）⇒ 第 3 参 = **true** | ✅ |

**释放路径的落地证据**（以 `FArrayHelper` 为例，其余 helper 同构）：`~FArrayHelper()` → `Deinitialize()` → `if (bNeedFreeData && ScriptArray != nullptr) { … Empty(0); delete ScriptArray; }`（`FArrayHelper.cpp:36-55`）。

**新消费点**（B-103 复核时新增）：`FManagedFunctionDescriptor.cpp:51-53` 把 `CopyValue` 结果直接传给 `Get<FReturn>(…)` ⇒ 走上表同一接管路径 ⇒ **同样不泄漏**。

---

## 3. 结论

1. **无泄漏面**：11/11 类在 `Get<FReturn>` 里都接走 `Src` 并负责释放 ⇒ `F-PROP-002` 不能按 P1/P2 泄漏记；
2. **真实缺陷只剩契约**：同一虚函数 `CopyValue` 有**三种所有权语义**（`nullptr` / 入参别名 / 新分配），且**基类默认返回 `nullptr`**；调用方必须"知道接收方会接管"——这一点在 `CopyValue` 的声明（`FPropertyDescriptor.h:47`）上没有任何表达；
3. **为什么不做**：把契约显式化需要改**公开虚函数签名**（加 `IsNeedFree` 出参/改名/新增配对释放 API）⇒ 撞 G7 的"公开面"与 Q2 的"既有取值域"；而**收益 = 0 泄漏 + 0 现有缺陷**（当前只有 2 个调用族，且都走接管路径）。

**处置**：⚫ **结案（契约级 · 决定不做）**，保留全部事实；
**触发条件**：① 新增 `CopyValue` 覆盖（返回第四种所有权）；② 新增消费者**不**经 `Get<FReturn>` 接管（直接取指针自行处理）；③ 维护者批准一次公开面收敛（届时把 `CopyValue` 改成显式所有权枚举/配对释放）。

---

## 4. 台账回写

| 台账 | 动作 |
|---|---|
| [`09-…/01`](../09-问题清单/01-全局问题总表.md) → [`05`](../09-问题清单/05-已完成（P2与P3）.md) | `F-PROP-002` 行迁 `05`（**结案 · 契约级 · 决定不做**，附本页 11/11 证据摘要） |
| [`09-…/00`](../09-问题清单/00-推荐优先修复清单.md) / [`README.md`](../README.md) / [`INDEX.md`](../INDEX.md) | 轮 B-110 注/行 |

---

## 5. 边界

1. 本页**只核 `Get(FReturn)` 这一条消费路径**（`FunctionMacro.h:133/:148` 与 `FManagedFunctionDescriptor.cpp:51-53`）；若将来出现**不经 `Get` 的裸 `CopyValue` 消费**，契约问题会立刻变成泄漏（这正是触发条件 ②）。
2. "接管"= 注册表在**引用移除/析构**时释放；释放时机与 C# 句柄生命周期绑定，本页未核"句柄长期不释放"时的驻留时长。
3. 未跑构建/套件（本页 0 代码）。
