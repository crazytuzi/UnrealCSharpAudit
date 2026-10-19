# 轮 B-45 设计页：compound 批量「strong-handle 通路」（选项 A）—— **不可行，需生命周期重设计**

> **来源**：维护者指定"**按照 A**"（B-44 的三个选项之一：strong-handle bulk 通路）。**结论**：🟥 **A 在当前句柄/注册表生命周期模型下不可行** —— 不是"多写一个 bulk 函数"，而是要先重设**托管包装对象的存活与释放**。**本轮 0 代码改动**（插件仍 `87dac201`），只出设计结论与最小可行替代。
> **口径**：G7 / [`08-…/66`](66-待决策页-三问（Q1四路线·Q2口径例外·Q3失败判据）（2026-10-15）.md) Q2。

---

## 0. 结论摘要

| 项 | 结论 |
|---|---|
| **A 的字面方案** | compound bulk 用 **strong** GCHandle 创建包装对象，C# 物化后显式释放 |
| **为什么不可行（一）** | **提前释放会打断用户持有的对象**：包装对象的每个方法都用 `HandleData.GetHandle(this)` 取句柄调原生；`HandleData.Free(handle)` 会从 `Handles` 删除该条目 ⇒ 用户手里的 `FString`/`FObject` 之后所有调用都拿不到原生对象 |
| **为什么不可行（二）** | **不释放则泄漏**：包装对象按"元素地址"登记在 `FStringRegistry`/`FObjectRegistry` 等（`Address2ManagedHandle`），`RemoveReference` 才 `GCHandle_Free`；strong 句柄会让每个被枚举过的元素**常驻内存**直到元素移除/域卸载 |
| **为什么不能"物化后降级为 weak"** | `HandleData.Alloc` 用 `ConditionalWeakTable` 按对象缓存 `HandleReference`（句柄值不可替换）；`Free` 不清 CWT ⇒ 重新 `Alloc(obj, weak)` 复用旧句柄值虽可行，但**与注册表的 address→handle 映射、移除时的 `GCHandle_Free` 形成双重所有权**，任何一条移除路径都会二次释放 |
| **真正的 A** | 需要**引用计数句柄 + 包装对象 `IDisposable`/释放契约 + 注册表让出所有权**的整体改造 —— 属跨模块设计（`HandleData` / `FStringRegistry` / `FObjectRegistry` / 各 descriptor / C# 包装基类） |
| **建议** | ① 改用 **B（`GC.TryStartNoGCRegion` 包住"批量 + 物化"）** 做一次受控探针；或 ② **compound 保持逐元素**（按设计收口）。**不建议**在没有生命周期设计的情况下动 A |

---

## 1. 证据链（代码级）

### 1.1 包装对象是 **weak** 句柄

`FStrPropertyDescriptor::Get(Src, Dest, FMember)`（`Private/Reflection/Property/StringProperty/FStrPropertyDescriptor.cpp`）：

```cpp
auto Object = FCSharpEnvironment::GetEnvironment().GetStringObject<FString>(Src);

if (!IManagedHandleIsValid(Object))
{
    if (Class != nullptr)
    {
        Object = Class->NewObject(true);          // ← bIsWeak = true
        FCSharpEnvironment::GetEnvironment().AddStringReference<FString, false, true>(Class, Object, Src);
    }
}

*reinterpret_cast<IManagedHandle*>(Dest) = Object;
```

`FClassReflection::NewObject(bool bIsWeak)` → `ScriptDomain->NewObject(ManagedClass, bIsWeak)` → `ObjectBridge.NewObject(InHandle, bIsWeak)`：

```csharp
var Object = RuntimeHelpers.GetUninitializedObject(Type);
return HandleData.Alloc(Object, bIsWeak != 0);    // weak GCHandle
```

### 1.2 注册表**拥有句柄**，移除时释放

`FStringRegistry.inl`：

```cpp
template <auto IsNeedFree, auto IsMember>
static auto AddReference(Class* InRegistry, FClassReflection* InClass, IManagedHandle InManagedHandle, Address InAddress)
{
    if constexpr (IsMember) { (InRegistry->*Address2ManagedHandle).Add(InAddress, InManagedHandle); }
    (InRegistry->*ManagedHandle2Value).Add(InManagedHandle, Value(InAddress, IsNeedFree));
}

static auto RemoveReference(Class* InRegistry, IManagedHandle InManagedHandle)
{
    ...
    FDomain::GCHandle_Free(InManagedHandle);      // ← 句柄由注册表释放
    return true;
}
```

`GetObject(Address)` 还会检查 `GCHandle_IsAlive`，死了就 `RemoveReference` 并清映射 —— 这套机制的前提就是**包装对象可被回收**。

### 1.3 提前 `Free` 的后果（A 的"物化后释放"）

`HandleData.FreeImplementation(handle)` 会 `Handles.Remove(handle)` 并 `OutHandle.Free()`；而 C# 包装对象的 `GetHandle(this)` 依赖 `ObjectToHandleReference`（CWT）返回同一个句柄值。释放后：

- 用户手中对象的方法（`ToString()`/`Equals()`/属性访问）取到的句柄在 `Handles` 里已不存在 ⇒ 原生侧解析为 `nullptr`/0 ⇒ 静默失效或空值；
- 这正是 B-44 里"批量返回的 wrapper 为 null"的**镜像问题**（那边是对象没了，这边是句柄没了）。

### 1.4 不释放的后果（A 的"strong 常驻"）

- 每个被枚举元素的包装对象被 strong GCHandle 钉住；
- 注册表按**地址**缓存（`Address2ManagedHandle`），同一元素只建一次 ⇒ 不会按次泄漏，但**每个枚举过的元素常驻**，直到该地址被 `RemoveReference`（元素移除/覆盖）或域卸载；
- 对大数组/大 Map 的"只读遍历"来说，这是**不可接受的常驻增长**（原本 weak 的语义就是"遍历完即可回收"）。

---

## 2. 若要真的走 A：最小改造清单（供裁量）

| # | 改造点 | 内容 | 风险 |
|---|---|---|---|
| 1 | `HandleData` | 句柄增加 `IsStrong` + 引用计数；新增 `Retain/Release`；`Free` 只在计数归零时释放 | 影响全部互操作（P/Invoke 面） |
| 2 | 包装对象 | C# 包装基类实现 `IDisposable` 的"句柄释放"语义（或 finalizer 兜底） | 用户代码需配合；finalizer + strong 会互锁 |
| 3 | 注册表 | `FStringRegistry`/`FObjectRegistry`/`FStructRegistry` 的 address→handle 映射改为**借用**（不拥有），移除时不再 `GCHandle_Free` | 涉及 B-44 之外的多条既有 Finding（F-CS2A/F-REG 族） |
| 4 | bulk 协议 | 新增 `GetElementsStrong` / `GetPairsStrong`，槽位写 strong 句柄；C# 物化后 `Release` | 需要 1–3 先成立 |

⇒ 这是一次**跨模块设计**，不是一轮能安全落地的改动。在没有第 1–3 项之前写第 4 项，只会把"崩溃"换成"静默失效/泄漏"。

---

## 3. 建议的最小可行替代

| 方案 | 内容 | 判据 | 代价 |
|---|---|---|---|
| **B（推荐做探针）** | `GC.TryStartNoGCRegion(预算)` 包住"一次 bulk 调用 + 把句柄物化成 `object[]`"，块内不 `yield`；块外再 `yield` | chunk=256 连续 20×10 000 不崩、`sum` 与逐元素一致；套件 1 963/0 同 SHA | 全局 GC 抑制（一个 chunk 的时长）、预算失败要兜底；不跨 `yield` |
| **C（当前）** | compound 逐元素 | 已成立 | compound `foreach` 仍 2/3/4 次 P/Invoke/元素 |

**建议**：先做 B 的**受控探针**（只开 `TArray<FString>` 一个容器，测"能否稳定 + GC 抑制代价"）。若 B 通过且代价可接受，再推广到 Map/Set；若不可接受，compound 按 C 收口并在台账写"按设计不批量"。

---

## 4. 本轮状态

| 项 | 值 |
|---|---|
| 插件 | **`87dac201`**（无改动、无提交） |
| 审计库 | 本报告（`08-…/87`）＋ §0.0/§44 台账注 |
| 复验 | 无代码变更，无需重跑；B-44 回滚后的 1 963/0 同 SHA 仍有效 |
