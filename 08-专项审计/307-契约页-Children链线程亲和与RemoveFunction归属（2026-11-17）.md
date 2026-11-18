# 契约页：`Children` 链线程亲和 与 `RemoveFunction` 归属（`F-CB-022` / `F-REG2-016`）（2026-11-17）

> **用途**：把 `FCSharpBind::RemoveFunctionFromClass` 的**两条隐式约定**写成可复核契约 —— ① `UStruct::Children` 链表没有同步，只能在游戏线程读写；② `UClass.RemoveFunction(FName)` 只删**本类自有**函数。来源 = B-267 登记的 `F-CB-022`（无锁面）与 B-268 已修的 `F-REG2-016`（归属守卫）。
> **性质**：契约页（**0 行插件代码**）。不改代码、不改级别，只登记"什么算违约、违约怎么判"。

---

## 0. 契约条目

| 编号 | 契约 | 违约形态 | 现状 |
|---|---|---|---|
| **C-CHILDREN-1** | `UStruct::Children` 链（含 `UField::Next`）只在**游戏线程**读写；`FCSharpBind::RemoveFunctionFromClass` 必须在游戏线程调用 | 在 GC 线程（`UStruct::AddReferencedObjects`）、保存线程（`UStruct::Serialize`）、编辑器 UI 线程（`TFieldIterator`）与摘链并发 | 摘链**无锁**（`FuncMap` 有 `FUClassFuncScopeWriteLock`，`Children` 没有）；调用点均按"游戏线程"写，但**没有** `check(IsInGameThread())`（G7 封死新增断言） |
| **C-REMOVE-1** | `UClass.RemoveFunction(FName)` 只删**本类自有**函数；继承函数 = **静默 no-op** | 传继承函数名时删掉父类（含引擎类）函数、或留下调用者类缓存悬垂 | 已由 `FRegisterClass.cpp:20` 的 `EIncludeSuperFlag::ExcludeSuper` 保证（B-268 已修 `F-REG2-016`） |
| **C-REMOVE-2** | 摘链命中后，被摘节点必须 `Next = nullptr`（成为叶子） | 被摘节点仍指向原后继，按 `Next` 遍历会"带出"已不属于该类的链 | `FCSharpBind.cpp:181` 已保证（B-269） |

---

## 1. 证据（引擎侧）

| 事实 | 位置（UE 5.6） |
|---|---|
| `Children` 是**内建 GC 成员**（编辑器期 `GIsEditor` 分支才遍历） | `CoreUObject/Private/UObject/Class.cpp:2514-2530`、`:2723`（`DeclareIntrinsicMembers(… Children …)`) |
| `FuncMap` 被 `UClass::AddReferencedObjects` 以 stable reference 遍历 | 同文件 `:4823-4826` |
| `FuncMap` 的增删**有锁**，`Children` 的插链**无锁** | `Class.h:3276-3287`（`FUClassFuncScopeWriteLock`）；`Class.cpp:6888-6889`（`Func->Next = Children; Children = Func;` 裸写） |
| 引擎**不提供**从 `Children` 摘节点的 API（三档一致） | 5.0 `Class.h:2973-2976`、5.6 `:3276-3287`、5.8 `:4275-4286`（都只删 `FuncMap` ＋ `AllFunctionsCache`） |
| `FindFunctionByName` 默认含父类，且把继承结果缓存进**被查询类** | 5.6 `Class.h:3293`、`Class.cpp:6977-6979`（"Cache the result (even if it's nullptr)"） |
| GC 只处理 `GetFirstGCIndex()` 之后的对象（disregard 类的成员不被遍历） | `GarbageCollection.cpp:4245/5180/4334`（B-267 D 包读数） |

---

## 2. 消费点（谁依赖这两条契约）

| 调用点 | 线程/时机 | 契约 |
|---|---|---|
| `FRegisterClass::RemoveFunctionImplementation`（`FRegisterClass.cpp:13-37`，C# 公开 API `UClass.RemoveFunction`） | 游戏线程（C# 侧调用） | C-CHILDREN-1 ＋ C-REMOVE-1 |
| `FCSharpFunctionRegister::~FCSharpFunctionRegister`（`FCSharpFunctionRegister.cpp:101`） | 域卸载/模块关停，游戏线程 | C-CHILDREN-1 ＋ C-REMOVE-2 |
| `FRegisterInputComponentHelper::BindFunction`（`InClass->Children = Function`，`:63`） | 游戏线程（绑定注册） | C-CHILDREN-1（**插链**侧，同一锁面） |
| `TFieldIterator<UFunction>`（`FCSharpBind.cpp:262-282` 等） | 游戏线程 | C-CHILDREN-1（只读侧） |

---

## 3. 违约判据（怎么证明破了）

| # | 判据 | 装置 |
|---|---|---|
| 1 | 线程：在**非游戏线程**调用 `UClass.RemoveFunction`，观察 `Children` 遍历并发读 | 临时探针 `check(IsInGameThread())`（**仅探针**、交付前删除）或 `FPlatformTLS::GetCurrentThreadId()` 打点 |
| 2 | 归属：`UClass.RemoveFunction("ReceiveBeginPlay")`（继承名）后，`AActor::FindFunctionByName` 仍非空；`UClass.RemoveFunction(自有名)` 后该函数消失且其它函数数量不变 | C# 用例 ＋ 两条断言（B-268 报告 §3.1） |
| 3 | 断链：摘链后从被摘节点出发遍历，`Next == nullptr` | C++ 探针（`Class->Children` 手工走链） |
| 4 | 缓存：先对子类查一次继承函数（填 `AllFunctionsCache`），再删同名继承函数，断言子类查询不返回垃圾对象 | C# 用例（B-267 `F-REG2-016` 验证方式） |

---

## 4. 触发条件（什么时候必须重开）

| 契约 | 触发条件 | 重开动作 |
|---|---|---|
| C-CHILDREN-1 | ① 出现"非游戏线程调用 `RemoveFunction`/`BindFunction`"的实证；② 引擎改为对 `Children` 加锁或提供摘链 API（5.8+ 需复核） | 出设计页：选择 `check`（G7 需裁定）/ 迁移到 `AsyncTask(GameThread)` / 改走引擎 API |
| C-REMOVE-1 | 需要"跨类删除（含继承）"的真实需求；或引擎把 `FuncMap` 改为跨类共享 | 重开 `F-REG2-016`，按"两套缓存都清"的方案（`RemoveFunctionFromClass` ＋ `FoundClass->ClearFunctionMapsCaches()`） |
| C-REMOVE-2 | 有人从被摘节点继续走 `Next` 的实证 | 无需动作（已 `nullptr`） |

---

## 5. 未取得 / 限制

| # | 未取得 | 说明 |
|---|---|---|
| 1 | **运行期线程读数** | 契约是"按代码形态 ＋ 调用点时机"定的，未构造跨线程调用（G7 禁止交付 `check`） |
| 2 | **5.8 对 `Children` 的改动** | 5.8 头文件同形（`Class.h:4275-4286`），但未实编（`F-BIND-013` 的 5.8 阻塞同源） |
| 3 | **GC 线程实际并发窗口** | 摘链发生在游戏线程、GC 在独立线程，但未量化"同时命中"的概率 |

---

## 6. 版本记录

| 日期 | 变更 |
|---|---|
| 2026-11-17 | 新建本页：C-CHILDREN-1 / C-REMOVE-1 / C-REMOVE-2；消费点 4 处、判据 4 条、触发条件 3 条 |
