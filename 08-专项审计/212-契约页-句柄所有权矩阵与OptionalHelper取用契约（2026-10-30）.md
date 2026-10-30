# 契约页：句柄所有权矩阵与 `FOptionalHelper` 取用契约（2026-10-30）

> **来源**：T4-6 `F-CS1-002` 设计页 [`08-…/210`](210-轮B-177-T4-6-F-CS1-002设计裁定-强弱句柄所有权矩阵（2026-10-29）.md) §4 方案 D（"把矩阵写进维护文档"）＋ T4-5 `F-DEL-010` 设计页 [`08-…/209`](209-轮B-176-T4-5-F-DEL-010设计裁定-OptionalHelper未set契约（2026-10-29）.md) §5 P2-D（"在维护文档写明 `Get()` 契约"）。
> **性质**：**规范页（0 代码）**。登记两处"文档型 P2"的落点；本页不新增 `F-*` 编号。
> **一句话**：**强句柄 = C++ 持有、C++ `Free`；弱句柄 = C# 侧另有强根、只用于包装类自注册**；`FOptionalHelper::Get()` **只在 `IsSet()` 为真时返回有效指针，否则 `nullptr`**。

---

## 1. 句柄所有权矩阵（HEAD `4564008d` ＋ 2026-10-30 命名化）

| 类别 | 代表调用点 | 句柄类型 | 谁负责 `Free` | 备注 |
|---|---|---|---|---|
| **C++ 持有的对象** | [`ObjectBridge.cs:16`](<D:\file\UnrealProjects\UnrealCSharpTest\Plugins\UnrealCSharp\Script\Interop\Bridge\ObjectBridge.cs>)（`bIsWeak` 由 C++ 传入）、[`AssemblyLoader.cs:24`](<D:\file\UnrealProjects\UnrealCSharpTest\Plugins\UnrealCSharp\Script\Interop\AssemblyLoader\AssemblyLoader.cs>)（程序集） | **强**（`Alloc`） | **C++**（`FDomain::GCHandle_Free`） | C++ 侧持有对象并跨帧使用 ⇒ 必须被 root |
| **包装类自注册** | `FString`／`FName`／`FText`／`FAnsiString`／`FUtf8String`、`TArray`／`TMap`／`TSet`／`TOptional`、`TSoftObjectPtr`／`TSoftClassPtr`／`TSubclassOf`／`TScriptInterface`／`TLazyObjectPtr`／`TWeakObjectPtr`／`TFieldPath`、`FDelegate`／`FMulticastDelegate` | **弱**（`Alloc(x, true)`） | **C++**（`*_UnRegisterImplementation`） | 对象由用户/GC 持有；native 侧必须容忍 `GetObject` 返回 `null`（弱句柄被回收后 `Target == null`） |
| **属性容器 helper 包装** | 各 `*PropertyDescriptor.cpp` 的 `Class->NewObject(true)` | **弱** | C++ | 同上 |
| **C++ 创建的托管数组** | [`ArrayBridge.cs:19`](<D:\file\UnrealProjects\UnrealCSharpTest\Plugins\UnrealCSharp\Script\Interop\Bridge\ArrayBridge.cs>) | **强**（B-178 起） | C++ | 旧码因 `3da38bb6` 改名从 pinned 误变 weak（F-CS1-002 G0），已改回强 |
| **反射返回值/出参** | `MethodBridge.cs:101/128`、`StringBridge.cs:13`、`TypeBridge` 多处、`Utils.cs` 的出参槽位 | **强**（`Alloc`） | 消费方 | 每次 `Alloc` 都 `Count++`；漏 `Free` = 表项与句柄号泄漏 |
| **C# 侧的临时强句柄** | [`HandleOwnership/UnitTestSubsystem.cs`](<D:\file\UnrealProjects\UnrealCSharpTest\Script\Game\UnrealCSharpTest\UnitTest\Interop\HandleOwnership\UnitTestSubsystem.cs>) | 强 | 测试自身 | 判据型用例（`IsAlive`／`GetObject`） |

## 2. 新调用点怎么选（规则）

1. **C++ 长期持有对象** ⇒ `HandleData.Alloc(object)`（强），并在销毁/解绑路径 `Free`。
2. **包装类自注册**（对象生命周期由用户或 GC 决定）⇒ `HandleData.Alloc(object, true)`（弱）；native 侧先判 `GetObject` 非空再解引用。
3. **反射返回值 / 出参** ⇒ `Alloc`（强），由取走句柄的一方释放。
4. 🔴 **禁止**用弱句柄（`Alloc(x, true)`）承载"C++ 唯一持有"的对象 —— 无其它托管根时下一次 GC 即回收（B-177 探针：`weak_alive=False` / `strong_alive=True`）。
5. 同一对象重复 `Alloc`／`AllocWeak` 会走 `HandleReference.Count++`；**只有最后一次 `Free` 才真正释放**（`HandleData.FreeImplementation` 先减计数、归零才 `GCHandle.Free`）。
6. 弱句柄同样必须 `Free`：对象被回收后 `FreeImplementation` 会直接移除表项，但**不会自动回收句柄号**（表项残留到 `Clear()` 或进程结束）。

## 3. API 变更记录（2026-10-30，轮 B-179；🔁 **已于 B-187 撤回**）

> 🔁 **撤回说明（2026-10-30，轮 B-187 `376d8cfd`）**：维护者裁定「只用 `Alloc` 一个函数」⇒ `HandleData` 恢复为 **`Alloc(object InObject, bool bIsWeak = false)`**，21 处调用点回改为 `Alloc(x, true)`，`ObjectBridge` 回改为 `Alloc(Object, bIsWeak != 0)`；`AllocWeak`／`AllocHandle` 已删除。下表为 B-179 当时的记录，**现状以"撤回说明"为准**。详见 [`223-轮B-187`](223-轮B-187-撤回AllocWeak并恢复单函数Alloc（2026-10-30）.md)。

| 项 | 内容 |
|---|---|
| 改签名（**已撤回**） | `HandleData.Alloc(object InObject, bool bIsWeak = false)` ⇒ **`Alloc(object)`（强）＋ `AllocWeak(object)`（弱）**；私有共用实现 `AllocHandle(object, bool)` |
| 调用点（**已撤回**） | **21 处**弱句柄改写为 `AllocWeak`（19 文件）；`ObjectBridge.NewObject` 的动态 `bIsWeak` 改为三元 |
| 未新增 `AllocStrong` | `Alloc` 本身就是强句柄；且工程侧 16 155＋ 生成物直接调用 `HandleData.Alloc(InObject)`（模板出处 `FBindingClassGenerator.cpp:820`）⇒ 再加同义名会造成"一名两实"，且新名无独立消费者（E2 口径） |
| 未新增 `AllocPinned` | `3da38bb6` 起 `bPinned` 形参已不存在；`GCHandleType.Pinned` 在插件 `Script/**` 命中 **0**；`ArrayBridge` 已改强句柄（B-178），无 pin 需求 |
| 生成侧 | **0 改动**：`FBindingClassGenerator.cpp:820` 产出的 `HandleData.Alloc(InObject)` 仍合法（强句柄） |

## 4. `FOptionalHelper` 取用契约

| 层 | 契约 | 落点 |
|---|---|---|
| C++ helper | `Get()` 仅在 `OptionalProperty != nullptr && Data != nullptr && IsSet(Data)` 为真时返回有效指针，否则 `nullptr` | [`FOptionalHelper.cpp`](<D:\file\UnrealProjects\UnrealCSharpTest\Plugins\UnrealCSharp\Source\UnrealCSharp\Private\Reflection\Optional\FOptionalHelper.cpp>)（B-178 三重判空） |
| C++ 调用方 | 取指针**前**必须 `IsSet()`；`nullptr` 是既有值域（Q2），不是错误码 | [`FRegisterOptional.cpp:130`](<D:\file\UnrealProjects\UnrealCSharpTest\Plugins\UnrealCSharp\Source\UnrealCSharp\Private\Domain\Interop\FRegisterOptional.cpp>) 已先判 |
| C# 侧 | `TOptional<T>.Get()` 在未设置时返回 `default`；判"是否有值"只能用 `IsSet()` | [`TOptional.cs:57/59`](<D:\file\UnrealProjects\UnrealCSharpTest\Plugins\UnrealCSharp\Script\UE\CoreUObject\TOptional.cs>) |
| 未设置 vs 值为 null | C# `Set(null)`／`new TOptional<T>(null)` 的语义**尚未裁定**（`F-CS2A-025`，P2 开放队列 A 组），本页不定义；当前实现会在 `HandleData` 处抛 `ArgumentNullException` | 见 [`09-…/01`](../09-问题清单/01-全局问题总表.md) `F-CS2A-025` 行 |

## 5. 关联条目

| 条目 | 状态 |
|---|---|
| `F-CS1-002` | P1（`ArrayBridge` 强句柄）已施工 **B-178**；P2（命名化）**2026-10-30 轮 B-179 落地**；D（本页矩阵）**本页** |
| `F-DEL-010` | P1（`Get()` 三重判空）已施工 **B-178**；P2（契约文档）**本页 §4** |
| `F-CS2A-025` | 未修（A 组，语义三选一待定） |
| `F-CS1-001`（句柄号单调） | 未修（与弱句柄表项清扫同族，见 `F-CS1-002` G2/G4） |
