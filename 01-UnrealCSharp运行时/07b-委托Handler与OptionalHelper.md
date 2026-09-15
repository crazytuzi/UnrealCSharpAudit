# 07b - 委托 Handler 与 OptionalHelper（C# 委托 ↔ UE 委托桥接）

> 本报告各条**严重度见各 Finding 的「复核结论」字段**（18 条全部回源码逐条核对；含 0 条判定为非缺陷；级别调整 2 条 = F-DEL-006 P2→P1（上调）、F-DEL-005 P1→P2（下调）；另有 4 处**子论断**被证伪/撤回，详见各条 `复核证据`）。
> 各级分布：**P0 = 2、P1 = 4、P2 = 4、P3 = 8**（初判定级 P0 = 3、P1 = 5、P2 = 5、P3 = 5）。





> 分析范围：`Source/UnrealCSharp/{Public,Private}/Reflection/Delegate/` 与 `.../Reflection/Optional/`
> 覆盖文件：12 个（见第 0 节）
>
> 阅读情况：12 个指定文件全部读完 + 6 个支撑文件；未覆盖项见第 8 节

---

## 0. 覆盖范围与阅读清单

> 行数口径：下表为**总行数**（等价 pwsh `(Get-Content $f).Count`，**含空行**）。若与本项目别处的"非空行数"（`Measure-Object -Line`，忽略空行）不同，属口径差异而非错误。复核已逐文件 `read` 全文核对，12 个文件的行数**全部吻合**。

| 文件 | 行数 | 是否读完 | 备注 |
|---|---|---|---|
| `Source/UnrealCSharp/Public/Reflection/Delegate/DelegateHandler.h` | 159 | 是 | |
| `Source/UnrealCSharp/Private/Reflection/Delegate/DelegateHandler.cpp` | 101 | 是 | |
| `Source/UnrealCSharp/Public/Reflection/Delegate/MulticastDelegateHandler.h` | 122 | 是 | |
| `Source/UnrealCSharp/Private/Reflection/Delegate/MulticastDelegateHandler.cpp` | 167 | 是 | |
| `Source/UnrealCSharp/Public/Reflection/Delegate/FDelegateBaseHelper.h` | 21 | 是 | 纯头文件 |
| `Source/UnrealCSharp/Public/Reflection/Delegate/FDelegateWrapper.h` | 15 | 是 | 纯头文件 |
| `Source/UnrealCSharp/Public/Reflection/Delegate/FDelegateHelper.h` | 102 | 是 | |
| `Source/UnrealCSharp/Private/Reflection/Delegate/FDelegateHelper.cpp` | 81 | 是 | |
| `Source/UnrealCSharp/Public/Reflection/Delegate/FMulticastDelegateHelper.h` | 78 | 是 | |
| `Source/UnrealCSharp/Private/Reflection/Delegate/FMulticastDelegateHelper.cpp` | 105 | 是 | |
| `Source/UnrealCSharp/Public/Reflection/Optional/FOptionalHelper.h` | 48 | 是 | `#if UE_F_OPTIONAL_PROPERTY` 条件编译 |
| `Source/UnrealCSharp/Private/Reflection/Optional/FOptionalHelper.cpp` | 108 | 是 | 同上 |

---

## 1. 模块职责与架构速览

### 1.1 五层结构（单播，多播为同构镜像）

```
C# 侧                          C++ 侧
────────────────────────────────────────────────────────────────────────────
FDelegate (C# 类)      ──P/Invoke──▶  FRegisterDelegate            (Interop 注册层)
  .Bind(obj, method)                   Private/Domain/Interop/FRegisterDelegate.cpp
  .UnBind()                            ↓ 由句柄取 helper
  .ExecuteN()                          FDelegateHelper             (门面层, 裸 new)
                                         Public/Reflection/Delegate/FDelegateHelper.h
                                         : FDelegateBaseHelper { void* Address; }
                                         ↓ 由 TWeakObjectPtr 取 handler
                                       UDelegateHandler : UObject   (桥对象层)
                                         Public/.../DelegateHandler.h
                                         ├─ FScriptDelegate* ScriptDelegate   （绑定到自身 CSharpCallBack）
                                         ├─ FCSharpDelegateDescriptor* DelegateDescriptor
                                         └─ FDelegateWrapper { TWeakObjectPtr<UObject>; FMethodReflection*; }
                                         ↓ UE 反射调用 ProcessEvent → CSharpCallBack
                                       FCSharpDelegateDescriptor::CallDelegate
                                         Private/Reflection/Function/FCSharpDelegateDescriptor.cpp
                                         ↓
                                       FManagedFunctionDescriptor::Invoke
                                         Private/Reflection/Function/FManagedFunctionDescriptor.cpp
                                         ↓ InMethod->Runtime_Invoke(handle, params)   ← 回到 C#
                                       C# 目标方法
```

**关键设计（也是全部风险的来源）**：UE 侧的 `FScriptDelegate` **永远只绑定一个东西**——桥对象自己（`UDelegateHandler`）的 `CSharpCallBack` UFUNCTION。真正的 C# 目标（对象 + 方法）被旁路存储在 `FDelegateWrapper` 里，回调时由 `ProcessEvent` 手工分发。这样做的好处是 UE 侧不需要为每个 C# 委托生成 UFUNCTION；代价是**目标有效性完全靠桥接层自己维护**，而它并没有维护（见第 5 节）。

多播版（`UMulticastDelegateHandler`）用同一手法处理多个目标：UE 侧只 `Add` 一个 `FScriptDelegate`（值成员 `ScriptDelegate`），其余 N-1 个目标存在 `TArray<FDelegateWrapper> DelegateWrappers` 里。

### 1.2 生命周期

1. **创建**：`FDelegatePropertyDescriptor::NewRef/NewWeakRef`（`FDelegatePropertyDescriptor.cpp:33/54`）或 `FCSharpBind::Bind`（`FCSharpBind.inl:129`）→ `new FDelegateHelper(...)` → 构造中 `NewObject<UDelegateHandler>()` + **`AddToRoot()`**（`FDelegateHelper.cpp:22-24`）。
2. **注册**：`AddDelegateReference` 写入 `FDelegateRegistry` 的两个 map（`FDelegateRegistry.h:43-49`），并创建 `TDelegateReference` 挂到 C# 对象上（`FDelegateRegistry.inl:52-54`）。
3. **使用**：C# 侧每次操作都按 `IManagedHandle` 查回 helper（`FRegisterDelegate.cpp:35` 等），再转发到 handler。
4. **销毁**：C# 侧对象被 GC → `~TDelegateReference`（`TDelegateReference.h:13-16`）→ `RemoveDelegateReference` → `FDelegateRegistry::RemoveReference`（`FDelegateRegistry.inl:57-81`）→ `delete helper` → `~FDelegateHelper` → `Deinitialize()` → `handler->Deinitialize()` + `RemoveFromRoot()`。
   - **但 Interop 的 `UnRegister` 走的是 `AsyncTask(ENamedThreads::GameThread, ...)`（`FRegisterDelegate.cpp:23-27`）；而 `~TDelegateReference` 不走。** 两条路径线程假设不一致（F-DEL-012）。

### 1.3 职责重叠（对任务书第 4 问的回答）

| 层次 | 单播 | 多播 | 职责 | 是否可与下层合并 |
|---|---|---|---|---|
| Interop 注册 | `FRegisterDelegate` | `FRegisterMulticastDelegate` | 把 C# 的 P/Invoke 名映射到 C++ 静态函数 | 否（模板化程度已高） |
| 门面 | `FDelegateHelper` | `FMulticastDelegateHelper` | 持有 handler 弱指针、转发全部调用、7/4 个 `ExecuteN`/`BroadcastN` 模板 | **可合并**：模板化 `template<typename THandler> class TDelegateHelper` 即可，二者差异只有 3 个多播专有方法 |
| 基类 | `FDelegateBaseHelper` | 同 | 只存 `void* Address` 并提供 `GetAddress()` | **不必要**：全插件**没有任何地方以 `FDelegateBaseHelper*` 持有派生类**（复核 grep：插件根 `FDelegateBaseHelper` = **11 命中**，仅出现在自身头文件 3 处、两个派生类的 `#include`/继承各 1 处、构造初始化列表各 1 处）→ 虚析构与多态纯属多余，可直接把 `Address` 成员放进两个门面类 |
| 桥对象 | `UDelegateHandler` | `UMulticastDelegateHandler` | `ProcessEvent` 分发、`Bind/Add/Remove` 维护目标表、7/4 个 `ExecuteN`/`BroadcastN` 模板 | **可部分合并**（多播多一个 `DelegateWrappers` 数组） |
| 描述符 | `FCSharpDelegateDescriptor` | 同（共用） | 把 UE 参数缓冲翻译成 C# 调用 | 无需改 |

**最明确的重复实现（逐字级）**：
1. `ExecuteN`/`BroadcastN` 系列 **4 层 × 22 个模板函数**：`DelegateHandler.h:40-143`(7)、`FDelegateHelper.h:33-94`(7)、`MulticastDelegateHandler.h:46-104`(4)、`FMulticastDelegateHelper.h:36-70`(4)。每一层都是 `if (ptr != nullptr) ptr->同名函数<ReturnType>(同参数...)`。
2. `MulticastDelegateHandler::Add`（`:75-90`）与 `AddUnique`（`:92-107`）：**前 13 行（`:77-89` vs `:94-106`）逐字符相同**，只有最后一行 `Add` vs `AddUnique` 不同。
3. `FMulticastDelegateHelper::Add`（`:57-63`）与 `AddUnique`（`:65-71`）：转发体完全相同。
4. `FDelegateHelper.cpp` 与 `FMulticastDelegateHelper.cpp` 的 `Initialize`/`Deinitialize`/`IsBound`/`GetUObject`/`GetFunctionName` 结构完全同构（除了类型名与 `AddToRoot` 的接收者）。
5. `FOptionalHelper::GetData()`（`:99-102`）与 `GetAddress()`（`:104-107`）：**实现完全相同**。

---

## 2. 关键调用链

1. **C# 绑定一个 UE 单播委托**
   `Script/UE/Library/FDelegateImplementation.cs` → `FRegisterDelegate.cpp:44 DelegateHelper->Bind(...)` → `FDelegateHelper.cpp:48 DelegateHandler->Bind(...)` → `DelegateHandler.cpp:60 ScriptDelegate->BindUFunction(this, CSharpCallBack)` + `:64 DelegateWrapper = {InObject, InMethod}`

2. **C# 主动触发委托**
   `FRegisterDelegate.cpp:85 DelegateHelper->Execute0<>()` → `FDelegateHelper.h:38 DelegateHandler->Execute0<ReturnType>()` → `DelegateHandler.cpp`（模板在头 `DelegateHandler.h:49`）`DelegateDescriptor->Execute0<ReturnType>(ScriptDelegate)` → `FunctionMacro.h` 宏展开 → `FManagedFunctionDescriptor.cpp:49 InMethod->Runtime_Invoke(...)`

3. **UE 侧广播触发 C# 回调（唯一真实回调路径）**
   UE 调用 `FScriptDelegate::Execute` → `UDelegateHandler::ProcessEvent`（`DelegateHandler.cpp:4`）→ 名字命中 `CSharpCallBack` → `:10 DelegateDescriptor->CallDelegate(DelegateWrapper.Object.Get(), DelegateWrapper.Method, Parms)` → `FCSharpDelegateDescriptor.cpp:12 Invoke(InMethod, GetObject(InObject), ...)` → `FManagedFunctionDescriptor.cpp:49 InMethod->Runtime_Invoke(...)`

4. **多播广播（含 P0 风险点）**
   UE 广播 → `UMulticastDelegateHandler::ProcessEvent`（`MulticastDelegateHandler.cpp:5`）→ `:11 for (const auto& [Key, Value] : DelegateWrappers)` → `:13 CallDelegate(Key.Get(), Value, Parms)` → **回调进入 C#，C# 若调 `Remove` → 回到 `:111 DelegateWrappers.Remove(...)` 修改正在被遍历的数组**

5. **委托属性赋值（`P0` 风险点）**
   C# 写属性 → `FDelegatePropertyDescriptor::Set`（`FDelegatePropertyDescriptor.cpp:20`）→ `:24 GetDelegate<FDelegateHelper>(SrcManagedHandle)`（**可为 nullptr**）→ `:30 SrcDelegateHelper->GetUObject()`

6. **`TOptional` 从 C# 侧构造并 Set**
   `TOptional.cs:9 TOptional_Register1Implementation` → `TOptionalImplementation.cs:14` → `FRegisterOptional.cpp:14 Register1Implementation` → `:32 new FOptionalHelper(OptionalProperty, nullptr, true, true)` → `:34 AddOptionalReference<FOptionalHelper, false>`
   `TOptional.cs:46 Set(v)` → `FRegisterOptional.cpp:127 SetImplementation` → `:133/:139 OptionalHelper->Set(...)` → `FOptionalHelper.cpp:89 MarkSetAndGetInitializedValuePointerToReplace(Data)` + `:91 ValuePropertyDescriptor->Set(InValue, Data)`

7. **`TOptional` 作为函数返回值传给 C#**
   `FunctionMacro.h:143 PROCESS_RETURN` → `ReturnPropertyDescriptor->CopyValue(...)`（→ `TCompoundPropertyDescriptor.inl:35 FMemory::Malloc` 的**堆拷贝**）→ `FOptionalPropertyDescriptor.cpp:22 Get(Src,Dest,FReturn)` → `:26 new FOptionalHelper(Property, Src, true, false)` → **helper 接管这块 Malloc 内存的所有权** → 销毁时 `FOptionalHelper.cpp:40 FMemory::Free(Data)`

8. **委托/可选值的释放**
   C# GC → `~TDelegateReference`（`TDelegateReference.h:13`）→ `FDelegateRegistry.inl:57 RemoveReference` → `:61 GetAddress()` 作 key → `:71 delete *FoundValue` → `FDelegateHelper.cpp:36-38 Deinitialize()` + `RemoveFromRoot()`；以及 `:75 FDomain::GCHandle_Free(...)`

---

## 3. 逐文件 / 逐函数清单

### 3.1 `Public/Reflection/Delegate/DelegateHandler.h` + `Private/.../DelegateHandler.cpp`

`UDelegateHandler` 是 `UObject` 派生的**单播委托桥**：UE 侧的 `FScriptDelegate` 只绑定到本 UObject 的 `CSharpCallBack` UFUNCTION 上；真正的 C# 目标（UObject + FMethodReflection）存在 `DelegateWrapper` 里，回调时由 `ProcessEvent` 手工转发。

| 类::方法 | 文件:行 | 做了什么 | 调用方（grep 证据） | 结论 |
|---|---|---|---|---|
| `UDelegateHandler::ProcessEvent(UFunction*, void*)` | `Private/Reflection/Delegate/DelegateHandler.cpp:4-17` | 若 `Function->GetName() == FUNCTION_CSHARP_CALLBACK` 则把 `DelegateWrapper` 里的 C# 目标交给 `DelegateDescriptor->CallDelegate`；否则回退 `UObject::ProcessEvent` | UE 反射系统调用（`ScriptDelegate->BindUFunction(this, ...)` @ `DelegateHandler.cpp:60` 是唯一绑定点） | **未检查 `DelegateWrapper.Object` 有效性**，见 F-DEL-001 |
| `UDelegateHandler::CSharpCallBack()` | `DelegateHandler.cpp:19-21` | 空函数体，仅作为 `BindUFunction` 需要的 `UFunction` 占位符 | 同上（被 UE 反射寻址，非直接调用） | 设计手段，正常 |
| `UDelegateHandler::Initialize(FScriptDelegate*, UFunction*)` | `DelegateHandler.cpp:23-30` | `bNeedFree = (InScriptDelegate == nullptr)`；为空则 `new FScriptDelegate()`；`new FCSharpDelegateDescriptor(InSignatureFunction)` | `FDelegateHelper.cpp:26` | **`DelegateDescriptor` 为裸 `new`，无 null 检查、无 UPROPERTY 保护**；见 F-DEL-005 |
| `UDelegateHandler::Deinitialize()` | `DelegateHandler.cpp:32-52` | 仅当 `bNeedFree` 时 `Unbind()` + `delete ScriptDelegate`；总是 `delete DelegateDescriptor` | `FDelegateHelper.cpp:36`（`~FDelegateHelper`） | **`bNeedFree == false` 时不解绑**，见 F-DEL-004 |
| `UDelegateHandler::Bind(UObject*, FMethodReflection*)` | `DelegateHandler.cpp:54-65` | 若 `ScriptDelegate` 未绑定则 `BindUFunction(this, FUNCTION_CSHARP_CALLBACK)`；把 `{InObject, InMethod}` 存入 `DelegateWrapper` | `FRegisterDelegate.cpp:44` → `FDelegateHelper.cpp:48` | **无旧值处理**：重复 Bind 直接覆盖 `DelegateWrapper`，旧 C# 目标静默丢弃 |
| `UDelegateHandler::IsBound()` | `DelegateHandler.cpp:67-70` | 返回 `ScriptDelegate->IsBound()` | `FRegisterDelegate.cpp:56` | 空安全 |
| `UDelegateHandler::UnBind()` | `DelegateHandler.cpp:72-78` | `ScriptDelegate->Unbind()`，**不清理 `DelegateWrapper`** | `FRegisterDelegate.cpp:67` | 半解绑，见 F-DEL-004 |
| `UDelegateHandler::Clear()` | `DelegateHandler.cpp:80-86` | `ScriptDelegate->Clear()` | `FRegisterDelegate.cpp:76` | 同上 |
| `UDelegateHandler::GetUObject()` | `DelegateHandler.cpp:88-91` | 返回 `ScriptDelegate->GetUObject()`（即 `this`） | `FDelegatePropertyDescriptor.cpp:30` | 返回的是**桥对象自己**而非 C# 目标对象，语义易误用 |
| `UDelegateHandler::GetFunctionName()` | `DelegateHandler.cpp:93-96` | 返回 `ScriptDelegate->GetFunctionName()`（即 `CSharpCallBack`） | `FDelegatePropertyDescriptor.cpp:30` | 同上，语义易误用 |
| `UDelegateHandler::GetCallBack()` | `DelegateHandler.cpp:98-101` | `FindFunction(FUNCTION_CSHARP_CALLBACK)` | `FDelegateHelper.cpp:29` | 每次调用都做一次函数查找（非热路径可接受） |
| `Execute0/1/2/3/4/6/7<ReturnType>()` | `DelegateHandler.h:40-143` | 7 个模板函数，全部是同一模式的 4 层嵌套 `if` 后转发到 `DelegateDescriptor->ExecuteN` | `FRegisterDelegate.cpp:80-173` | **7 处近乎完全重复的样板**，见 F-DEL-014（P3） |

**注**：`Execute5` 与 `Broadcast1/3/5/7` 在**整个 `Source/` 中不存在**（grep `Execute5|Broadcast1<|Broadcast3|Broadcast5|Broadcast7` → **0 命中**）。即 buffer 组合 `(OUT_BUFFER, RETURN_BUFFER)` 这一路未被实现；`Execute` 覆盖 `{0,1,2,3,4,6,7}`、`Broadcast` 覆盖 `{0,2,4,6}`。无法仅从代码判定这是刻意裁剪还是遗漏，但对"8 种组合全覆盖"而言是一个明确的缺口（置信度: 低，缺设计文档佐证）。

### 3.2 `Public/Reflection/Delegate/MulticastDelegateHandler.h` + `Private/.../MulticastDelegateHandler.cpp`

`UMulticastDelegateHandler` 是**多播委托桥**：UE 侧 `FMulticastScriptDelegate` 里**只塞一个** `FScriptDelegate`（成员 `ScriptDelegate`），所有 C# 目标存在 `TArray<FDelegateWrapper> DelegateWrappers` 中，广播时在 `ProcessEvent` 内手工遍历。

| 类::方法 | 文件:行 | 做了什么 | 调用方（grep 证据） | 结论 |
|---|---|---|---|---|
| `UMulticastDelegateHandler::ProcessEvent` | `Private/.../MulticastDelegateHandler.cpp:5-21` | 若是 CSharpCallBack，**range-for 遍历 `DelegateWrappers`**，逐个 `CallDelegate` | UE 反射（`ScriptDelegate.BindUFunction(this, ...)` @ `:83` 与 `:100`） | **P0：遍历中回调，若 C# 侧调用 Remove 即迭代器失效** — 见 F-DEL-002 |
| `CSharpCallBack()` | `MulticastDelegateHandler.cpp:23-25` | 空占位 | 反射寻址 | 正常 |
| `Initialize(FMulticastScriptDelegate*, UFunction*)` | `MulticastDelegateHandler.cpp:27-37` | `bNeedFree` 记录；`new FMulticastScriptDelegate()` / `new FCSharpDelegateDescriptor(...)` | `FMulticastDelegateHelper.cpp:23` | 同 F-DEL-005 |
| `Deinitialize()` | `MulticastDelegateHandler.cpp:39-63` | 仅 `bNeedFree` 时 `RemoveAll(this)` + `delete`；总是 delete descriptor；`DelegateWrappers.Empty()`；`ScriptDelegate.Unbind()` | `FMulticastDelegateHelper.cpp:37` | `bNeedFree==false` 时**不 `RemoveAll(this)`**，外部 delegate 里留下失效条目，见 F-DEL-004 |
| `IsBound()` | `MulticastDelegateHandler.cpp:65-68` | 空安全查询 | `FRegisterMulticastDelegate.cpp:30` | OK |
| `Contains(UObject*, FMethodReflection*)` | `MulticastDelegateHandler.cpp:70-73` | `DelegateWrappers.Contains(FDelegateWrapper{...})` | `FRegisterMulticastDelegate.cpp:41` | **只看 C# 侧列表，不看 UE 侧 `MulticastScriptDelegate`**，两者可能不一致 |
| `Add(UObject*, FMethodReflection*)` | `MulticastDelegateHandler.cpp:75-90` | 若 UE 侧不含该 `ScriptDelegate` 则 `Unbind`+`BindUFunction`+`Add`；然后 `DelegateWrappers.Add(...)` | `FRegisterMulticastDelegate.cpp:64` | ✅ 同时维护两侧 |
| `AddUnique(...)` | `MulticastDelegateHandler.cpp:92-107` | 与 `Add` **完全相同的前半段**（`:94-104` 逐字符重复 `:77-87`），仅末尾 `AddUnique` vs `Add` | `FRegisterMulticastDelegate.cpp:85` | **重复代码**，见 F-DEL-014 |
| `Remove(UObject*, FMethodReflection*)` | `MulticastDelegateHandler.cpp:109-122` | `DelegateWrappers.Remove(...)`；**仅当列表空时**才 `RemoveAll(this)` + `ScriptDelegate.Unbind()` | `FRegisterMulticastDelegate.cpp:106` | 见 F-DEL-002（可被回调重入） |
| `RemoveAll(UObject*)` | `MulticastDelegateHandler.cpp:124-140` | `DelegateWrappers.RemoveAll(谓词按 Object 比对)`；空则同步 UE 侧 | `FRegisterMulticastDelegate.cpp:127` | 同上 |
| `Clear()` | `MulticastDelegateHandler.cpp:142-152` | `MulticastScriptDelegate->Clear()` + `DelegateWrappers.Empty()` + `ScriptDelegate.Unbind()` | `FRegisterMulticastDelegate.cpp:139` | 清理最彻底，与 `Deinitialize` 语义重叠 |
| `GetUObject()` / `GetFunctionName()` | `MulticastDelegateHandler.cpp:154-162` | 转发给 `ScriptDelegate`（`this` / `CSharpCallBack`） | `FMulticastDelegatePropertyDescriptor.cpp:33-34` | 与单播版同样语义易误用 |
| `GetCallBack()` | `MulticastDelegateHandler.cpp:164-167` | `FindFunction(FUNCTION_CSHARP_CALLBACK)` | `FMulticastDelegateHelper.cpp:30` | OK |
| `Broadcast0/2/4/6<ReturnType>()` | `MulticastDelegateHandler.h:46-104` | 4 个模板，模式与单播版完全相同 | `FRegisterMulticastDelegate.cpp:148-186` | 与 `ExecuteN` 同属重复样板 |

**关键判断**：单播版 `DelegateWrapper`（`Public/Reflection/Delegate/FDelegateWrapper.h`）与多播版 `TArray<FDelegateWrapper>` 都持有 `UObject*`/弱指针，**C# 侧委托对象的存活由 C# 侧决定**——强引用还是弱引用见 §5（`FDelegateHelper.cpp` / `FMulticastDelegateHelper.cpp`）。

---

### 3.3 `Public/Reflection/Delegate/FDelegateWrapper.h`（纯头文件，15 行）

```cpp
// FDelegateWrapper.h:5-15
struct FDelegateWrapper
{
	TWeakObjectPtr<UObject> Object;   // ← 弱引用

	FMethodReflection* Method;        // ← 裸指针
};

static bool operator==(const FDelegateWrapper& A, const FDelegateWrapper& B)
{
	return A.Object == B.Object && A.Method == B.Method;
}
```

| 成员/函数 | 文件:行 | 做了什么 | 结论 |
|---|---|---|---|
| `FDelegateWrapper::Object` | `FDelegateWrapper.h:7` | `TWeakObjectPtr<UObject>` 指向 C# 对象在 UE 侧的 UObject 包装 | **弱引用**：C# 对象被 GC 后 `Get()` 返回 `nullptr`，但桥接层**不清理该条目**，见 F-DEL-001 / F-DEL-004 |
| `FDelegateWrapper::Method` | `FDelegateWrapper.h:9` | 裸 `FMethodReflection*`，指向 C# 方法的反射描述符 | **无任何所有效期保证**：`FMethodReflection` 若由 Registry 释放，此处即悬垂，见 F-DEL-001 |
| `operator==` | `FDelegateWrapper.h:12-15` | 比较 `TWeakObjectPtr` 与裸指针 | ⚠️ `static` 非成员运算符定义在头文件中，每个 TU 各一份（非 `inline`）→ 严格来说是 ODR 隐患；`static` 使其内部链接，功能上可用，但对每个包含者都生成一份代码（P3） |

**注意**：该头文件**没有**定义 `operator!=`，但 `TArray::Remove`/`Contains` 只需要 `==`，因此可用。同时因为 `Method` 参与相等比较，**同一个 C# 对象上的同一个方法重复 Add 会被判等**（`MulticastDelegateHandler::Contains` 依赖此语义，见 `MulticastDelegateHandler.cpp:72`）。

### 3.4 `Public/Reflection/Delegate/FDelegateBaseHelper.h`（纯头文件，21 行）

```cpp
// FDelegateBaseHelper.h:3-21
class FDelegateBaseHelper
{
public:
	explicit FDelegateBaseHelper(void* InAddress) : Address(InAddress) {}
	virtual ~FDelegateBaseHelper() = default;
public:
	void* GetAddress() const { return Address; }
private:
	void* Address;
};
```

| 成员/函数 | 文件:行 | 做了什么 | 结论 |
|---|---|---|---|
| 构造 / `GetAddress()` | `FDelegateBaseHelper.h:6-9`, `:14-17` | 保存并返回一个 `void*`（实际语义是 `FScriptDelegate*`，见 `FDelegateHelper.cpp:10`） | ✅ 类本身正确；**但 `Address` 不是所有效期保证**，见 F-DEL-013 |
| 虚析构 | `FDelegateBaseHelper.h:11` | `= default`，为了让 `FDelegateHelper`/`FMulticastDelegateHelper` 多态析构 | 目前**没有任何地方通过 `FDelegateBaseHelper*` 持有派生类**（待 grep 确认），基类可能是多余的 |

### 3.5 `Public/Reflection/Delegate/FDelegateHelper.h` + `Private/.../FDelegateHelper.cpp`

`FDelegateHelper` 是 `UDelegateHandler` 的**门面/包装**：`TWeakObjectPtr<UDelegateHandler> DelegateHandler` + 从基类继承的 `Address`。

| 类::方法 | 文件:行 | 做了什么 | 调用方（grep 证据） | 结论 |
|---|---|---|---|---|
| `FDelegateHelper::FDelegateHelper()` | `Private/.../FDelegateHelper.cpp:3-7` | `FDelegateBaseHelper(nullptr)` 后 `Initialize(nullptr, nullptr)` | `FCSharpBind.inl:127`（`new T()`）→ `FRegisterDelegate.cpp:18` | **默认构造仍会 `NewObject` 出一个 handler**，`Address` 保持 nullptr |
| `FDelegateHelper::FDelegateHelper(FScriptDelegate*, UFunction*)` | `FDelegateHelper.cpp:9-13` | `FDelegateBaseHelper(InDelegate)`（Address = InDelegate）→ `Initialize(...)` | `FDelegatePropertyDescriptor.cpp:39`、`:56` | 见 F-DEL-013 |
| `~FDelegateHelper()` | `FDelegateHelper.cpp:15-18` | 调 `Deinitialize()` | `FDelegateRegistry.inl:71`（`delete` → 析构） | ✅ 析构即解绑 |
| `Initialize(FScriptDelegate*, UFunction*)` | `FDelegateHelper.cpp:20-30` | `NewObject<UDelegateHandler>()` → **`AddToRoot()`** → `DelegateHandler->Initialize(...)`；`InSignatureFunction == nullptr` 时回退到 `DelegateHandler->GetCallBack()` | `FCSharpBind.inl:127`；`FDelegatePropertyDescriptor.cpp:39` | **`AddToRoot()` = 该 UObject 被 GC root 锚定**（回收依赖 `RemoveFromRoot` + 终结器，见 F-DEL-005，复核后 P2） |
| `Deinitialize()` | `FDelegateHelper.cpp:32-42` | `DelegateHandler->Deinitialize()` → **`RemoveFromRoot()`** → 置空 | `~FDelegateHelper`（`FDelegateHelper.cpp:17`） | ✅ 与 `AddToRoot` 配对；触发者是 `FDelegateRegistry.inl:71` 的 `delete`（由 C# 终结器/`~TDelegateReference` 驱动） |
| `Bind(UObject*, FMethodReflection*) const` | `FDelegateHelper.cpp:44-50` | 空检查后转发 `DelegateHandler->Bind` | `FRegisterDelegate.cpp:44` | ✅ |
| `IsBound()` | `FDelegateHelper.cpp:52-55` | 转发 | `FRegisterDelegate.cpp:56` | ✅ 空安全 |
| `UnBind()` | `FDelegateHelper.cpp:57-63` | 转发 | `FRegisterDelegate.cpp:67` | ✅ |
| `Clear()` | `FDelegateHelper.cpp:65-71` | 转发 | `FRegisterDelegate.cpp:76` | ✅ |
| `GetUObject()` | `FDelegateHelper.cpp:73-76` | 返回 `DelegateHandler->GetUObject()` = **handler 自身**（`DelegateHandler.cpp:88-91`） | `FDelegatePropertyDescriptor.cpp:30` | 语义陷阱：名为 `GetUObject` 但返回的是桥对象而非 C# 目标对象 |
| `GetFunctionName()` | `FDelegateHelper.cpp:78-81` | 返回 `CSharpCallBack` | `FDelegatePropertyDescriptor.cpp:30` | 同上 |
| `Execute0/1/2/3/4/6/7<ReturnType>()` | `Public/.../FDelegateHelper.h:33-94` | 7 个模板，全是 `if (DelegateHandler != nullptr) DelegateHandler->ExecuteN<...>(...)` | `FRegisterDelegate.cpp:80-173` | **与 `UDelegateHandler` 的同名模板构成第三层重复**，见 F-DEL-014 |

**关键事实（GC 相关）**：`FDelegateHelper` 通过 `AddToRoot()`（`FDelegateHelper.cpp:24`）让 `UDelegateHandler` UObject 强驻留；而 **C# 委托对象本体完全不在 C++ 侧持有**——`FDelegateWrapper::Object` 是 `TWeakObjectPtr`（`FDelegateWrapper.h:7`），`Method` 是裸指针（`:9`）。因此 C++ 桥接层对 C# 委托**不做强引用**，不存在"C++ 侧强引用导致 C# 委托永不回收"的泄漏；对应风险是**反向的悬垂/静默失效**（详见第 5 节与 F-DEL-001）。

**调用方证据（`GetAddress()` 是注册表的查找键）**
- `Public/Registry/FDelegateRegistry.inl:61` — `const auto Address = (*FoundValue)->GetAddress();`
- `Public/Registry/FContainerRegistry.inl:62`、`Private/Registry/FOptionalRegistry.cpp:59` — 同样模式
- 即 `FDelegateBaseHelper::Address` 是 `TMap<void*, ManagedHandle>` 的 key，**一旦 `Address` 指向的 `FScriptDelegate` 生命周期结束（属性所在 UObject 被 GC），map 中的 key 成为悬垂 key**，见 F-DEL-013。

### 3.6 `Public/Reflection/Delegate/FMulticastDelegateHelper.h` + `Private/.../FMulticastDelegateHelper.cpp`

结构与 `FDelegateHelper` **几乎逐行对应**，仅把 `UDelegateHandler` 换成 `UMulticastDelegateHandler`、`ExecuteN` 换成 `BroadcastN`，并多出 `Contains/Add/AddUnique/Remove/RemoveAll`。

| 类::方法 | 文件:行 | 做了什么 | 调用方（grep 证据） | 结论 |
|---|---|---|---|---|
| `FMulticastDelegateHelper::FMulticastDelegateHelper()` | `Private/.../FMulticastDelegateHelper.cpp:3-7` | `FDelegateBaseHelper(nullptr)` + `Initialize(nullptr, nullptr)` | `FCSharpBind.inl:127`（`new T()`）→ `FRegisterMulticastDelegate.cpp:18` | 与单播版同构 |
| `FMulticastDelegateHelper(FMulticastScriptDelegate*, UFunction*)` | `FMulticastDelegateHelper.cpp:9-14` | `FDelegateBaseHelper(InMulticastDelegate)` → `Initialize` | `FMulticastDelegatePropertyDescriptor.cpp:50`、`:68` | ✅ 真实使用点 |
| `~FMulticastDelegateHelper()` | `FMulticastDelegateHelper.cpp:16-19` | `Deinitialize()` | `FDelegateRegistry.inl:71`（`RemoveReference` → `delete`） | **若创建点从不 delete，则 handler 永久 `AddToRoot` 泄漏**，见 F-DEL-005 |
| `Initialize(...)` | `FMulticastDelegateHelper.cpp:21-31` | `NewObject<UMulticastDelegateHandler>()` → **`AddToRoot()`** → `Initialize(...)`，签名为空时回退 `GetCallBack()` | `:6`、`:13` | 同 F-DEL-005 |
| `Deinitialize()` | `FMulticastDelegateHelper.cpp:33-43` | `Deinitialize()` + **`RemoveFromRoot()`** + 置空 | `:~FMulticastDelegateHelper` | ✅ 配对 |
| `IsBound()` | `FMulticastDelegateHelper.cpp:45-48` | 转发 | `FRegisterMulticastDelegate.cpp:30` | 空安全 |
| `Contains(UObject*, FMethodReflection*)` | `FMulticastDelegateHelper.cpp:50-55` | 转发到 handler | `FRegisterMulticastDelegate.cpp:41` | ✅ |
| `Add(UObject*, FMethodReflection*) const` | `FMulticastDelegateHelper.cpp:57-63` | 转发 | `FRegisterMulticastDelegate.cpp` 系列（`:33,:47,:70,:91,:112,:130,:142,:151,:160,:169,:179`） | ✅ 真实使用 |
| `AddUnique(...) const` | `FMulticastDelegateHelper.cpp:65-71` | 转发 | `FRegisterMulticastDelegate.cpp:85` | 转发体与 `Add` 逐字重复（`FMulticastDelegateHelper.cpp:57-71`），见 F-DEL-014 |
| `Remove(...) const` | `FMulticastDelegateHelper.cpp:73-79` | 转发 | `FRegisterMulticastDelegate.cpp` | ✅ |
| `RemoveAll(UObject*) const` | `FMulticastDelegateHelper.cpp:81-87` | 转发 | `FRegisterMulticastDelegate.cpp:25`（`RemoveDelegateReference`） | ✅ |
| `Clear() const` | `FMulticastDelegateHelper.cpp:89-95` | 转发 | `FRegisterMulticastDelegate.cpp:139` | ✅ |
| `GetUObject()` / `GetFunctionName()` | `FMulticastDelegateHelper.cpp:97-104` | 转发给 handler（返回 `this` / `CSharpCallBack`） | `FMulticastDelegatePropertyDescriptor.cpp:33-34` | 同单播版的语义陷阱 |
| `Broadcast0/2/4/6<ReturnType>()` | `Public/.../FMulticastDelegateHelper.h:36-70` | 4 个模板转发 | `FRegisterMulticastDelegate.cpp:148-186` | 与 handler 层重复，见 F-DEL-014 |

**创建/销毁点（已确认）**：`FDelegatePropertyDescriptor.cpp:39,56` 与 `FMulticastDelegatePropertyDescriptor.cpp:50,68` 有 `new FDelegateHelper/FMulticastDelegateHelper(...)`；`FCSharpBind.inl:127` 的 `new T()` 提供默认构造入口（C# 侧 `new FDelegate()` / `new FMulticastDelegate()`）。**配对的 `delete` 只有一处**：`FDelegateRegistry.inl:71`（经 `FDelegateRegistry::RemoveReference`），由 C# 侧 `UnRegister` 或 `~TDelegateReference` 驱动。因此 helper 与 rooted `UDelegateHandler` 的释放**完全依赖 C# 侧销毁**，见 F-DEL-005 / F-DEL-007。

### 3.7 `Public/Reflection/Optional/FOptionalHelper.h` + `Private/.../FOptionalHelper.cpp`

`FOptionalHelper` 包装一个 `FOptionalProperty` + 一块值内存 `Data`。整个类被 `#if UE_F_OPTIONAL_PROPERTY`（`FOptionalHelper.h:6` / `.cpp:2`）包住，只在 UE 提供 `FOptionalProperty` 的版本上编译。

| 类::方法 | 文件:行 | 做了什么 | 调用方（grep 证据） | 结论 |
|---|---|---|---|---|
| `FOptionalHelper(FOptionalProperty*, void*, bool, bool)` | `Private/.../FOptionalHelper.cpp:5-25` | 存 `OptionalProperty`；`FPropertyDescriptor::Factory(GetValueProperty())` 造值描述符；`InData != nullptr` 就借用，否则 `FMemory::Malloc` + `InitializeValueInternal(Data)` | `FOptionalPropertyDescriptor.cpp`（待读） | **`Factory` 返回值未判空**；`InOptionalProperty` 也未判空（`:13` 直接解引用） |
| `~FOptionalHelper()` | `FOptionalHelper.cpp:27-30` | `Deinitialize()` | 待确认 | 见下 |
| `Initialize()` | `FOptionalHelper.cpp:32-34` | **函数体为空** | 无调用点（待 grep） | **死代码**，见第 7 节 |
| `Deinitialize()` | `FOptionalHelper.cpp:36-55` | `bNeedFreeData` 时 **只 `FMemory::Free(Data)`**；`bNeedFreeProperty` 时 `delete ValuePropertyDescriptor` + `delete OptionalProperty` | `~FOptionalHelper` | **P1：只释放原始内存、不调用值类型析构**，见 F-DEL-006 |
| `Identical(const FOptionalHelper*, const FOptionalHelper*)` | `FOptionalHelper.cpp:57-70` | 任一为 null 返回 false；`SameType` 不同返回 false；否则 `OptionalProperty->Identical(A->Data, B->Data, 0)` | 待确认 | ⚠️ `:64` 解引用 `GetValuePropertyDescriptor()` 未判空；且 `InA->Data`/`InB->Data` 未判空 |
| `Reset() const` | `FOptionalHelper.cpp:72-75` | `OptionalProperty->MarkUnset(Data)` | 待确认 | ✅ 语义正确（`MarkUnset` 会析构内部值）；但 **`OptionalProperty` 未判空**（对比 `Deinitialize:45` 判了空） |
| `IsSet() const` | `FOptionalHelper.cpp:77-80` | `OptionalProperty->IsSet(Data)` | 待确认 | 未判空 |
| `Get() const` | `FOptionalHelper.cpp:82-85` | `OptionalProperty->GetValuePointerForReadOrReplace(Data)` | 待确认 | **返回的指针在 `Reset()`/`Set()` 后可能失效**，若 C# 侧长期持有即悬垂，见 F-DEL-010 |
| `Set(void* InValue) const` | `FOptionalHelper.cpp:87-92` | `MarkSetAndGetInitializedValuePointerToReplace(Data)`（**返回值被丢弃**）后 `ValuePropertyDescriptor->Set(InValue, Data)` | 待确认 | **疑点：写回目标用的是 `Data` 而非 API 返回的指针**；若两者不等价 → 写错位置，见第 8 节 #5 |
| `GetValuePropertyDescriptor() const` | `FOptionalHelper.cpp:94-97` | 返回 `ValuePropertyDescriptor` | 待确认 | ⚠️ `Deinitialize()` 后返回 `nullptr`，调用方是否判空待查 |
| `GetData() const` | `FOptionalHelper.cpp:99-102` | 返回 `Data` | 待确认 | ✅ |
| `GetAddress() const` | `FOptionalHelper.cpp:104-107` | 返回 `Data`（**与 `GetData()` 完全相同的实现**） | `Private/Registry/FOptionalRegistry.cpp:59` 作为注册表 key | **`GetData()` 与 `GetAddress()` 是重复实现**（P3）；作为 map key 使用意味着 `Data` 是身份标识 |

**关于 `TOptional<T>` 构造/析构配对（任务要求的重点）**：
- 构造：`InitializeValueInternal(Data)`（`:23`）—— 让 `Data` 成为一块"已初始化但未 set"的可选值存储。
- set：`MarkSetAndGetInitializedValuePointerToReplace`（`:89`）—— 由 UE 负责把 flags 标记为 has-value 并对值做初始化。
- unset：`MarkUnset`（`:74`）—— 由 UE 负责析构内部值。
- 销毁：`FMemory::Free(Data)`（`:40`）—— **只 free 内存，不析构内部值**。
→ 对 `T = int32` 这类 POD 无影响；对 `T = FString` / `TArray` / 含堆的 `USTRUCT`，**内部堆内存泄漏**（P1，见 F-DEL-006）。若在 `Free` 之前 `MarkUnset` 即可避免（UE 的 `MarkUnset` 会析构）。

### 3.8 调用方：`FDelegatePropertyDescriptor.cpp` / `FMulticastDelegatePropertyDescriptor.cpp`（**不在指定文件清单内，但为回答"绑 定↔解绑配对"与"泄漏/悬垂"必须读**）

| 类::方法 | 文件:行 | 做了什么 | 结论 |
|---|---|---|---|
| `FDelegatePropertyDescriptor::Get(Src,Dest,FMember)` / `(FReturn)` | `Private/Reflection/Property/DelegateProperty/FDelegatePropertyDescriptor.cpp:5-13` | 返回 `NewWeakRef(Src)` | C# 侧拿到**弱引用**句柄 |
| `FDelegatePropertyDescriptor::Get(Src,Dest)` | `FDelegatePropertyDescriptor.cpp:15-18` | 返回 `NewRef(Src)` | 强引用句柄 |
| `FDelegatePropertyDescriptor::Set(void*, void*)` | `FDelegatePropertyDescriptor.cpp:20-31` | 从 C# 句柄取 `FDelegateHelper`，然后 `DestScriptDelegate->BindUFunction(SrcDelegateHelper->GetUObject(), SrcDelegateHelper->GetFunctionName())` | **P0：`SrcDelegateHelper` 无判空**（`GetDelegate<>` 可返回 nullptr）；`Property->InitializeValue(Dest)` 会覆盖已有值 |
| `FDelegatePropertyDescriptor::NewRef(void*)` | `FDelegatePropertyDescriptor.cpp:33-52` | 若 C# 侧无对象则 `new FDelegateHelper(...)` + `Class->NewObject()` + `AddDelegateReference(OwnerManagedHandle, InAddress, DelegateHelper, Class, Object)` | helper 由 Registry 接管（`AddDelegateReference` 存的是裸指针），**创建点无配对 delete** |
| `FDelegatePropertyDescriptor::NewWeakRef(void*)` | `FDelegatePropertyDescriptor.cpp:54-63` | **无条件 `new FDelegateHelper(...)`** 并 `AddDelegateReference` | **每次 `NewWeakRef` 都新建一个 helper 与一个 UObject**（`:56`、`:59`）→ 同一属性被反复以弱引用方式读取时**累积分配**，见 F-DEL-007 |
| `FMulticastDelegatePropertyDescriptor::Set(void*, void*)` | `FMulticastDelegatePropertyDescriptor.cpp:20-37` | 取 `FMulticastDelegateHelper` → `InitializeValue(Dest)` → 造一个临时 `FScriptDelegate` 绑到 helper 的 `CSharpCallBack` → `MulticastScriptDelegate->Add(ScriptDelegate)` | **P0：`SrcMulticastDelegateHelper` 无判空**（`:33-34`） |
| `FMulticastDelegatePropertyDescriptor::GetMulticastDelegate(void*)` | `FMulticastDelegatePropertyDescriptor.cpp:39-42` | 转发 `Property->GetMulticastDelegate(InAddress)` | ✅ |
| `FMulticastDelegatePropertyDescriptor::NewRef(void*)` | `FMulticastDelegatePropertyDescriptor.cpp:44-64` | 同单播版模式 | 创建点无配对 delete |
| `FMulticastDelegatePropertyDescriptor::NewWeakRef(void*)` | `FMulticastDelegatePropertyDescriptor.cpp:66-77` | 无条件 `new FMulticastDelegateHelper(...)` | 同 F-DEL-007 |

**这对文件揭示了 `FDelegateBaseHelper::Address` 的真实用途与风险**：`FDelegateHelper` 的 `Address` 就是 `Property->GetPropertyValuePtr(InAddress)`，即**UObject 内联存储里的那个 `FScriptDelegate` 地址**。该地址随 UObject 生死而存亡，而 `FDelegateRegistry` 用它作 map key（`FDelegateRegistry.inl:61`）——**UObject 被 GC 后 key 悬垂**，见 F-DEL-013。

### 3.9 支撑文件（为结论提供证据，非指定清单内）

#### 3.9.1 `Public/Registry/FDelegateRegistry.h` + `.inl`

| 类::方法 | 文件:行 | 做了什么 | 结论 |
|---|---|---|---|
| 成员映射 | `FDelegateRegistry.h:43-49` | 4 个 map：`ManagedHandle2Value` + `Address2ManagedHandle`（单播/多播各一对） | `Value` 是**裸指针** `FDelegateHelper*` / `FMulticastDelegateHelper*` |
| `GetDelegate(Class*, IManagedHandle)` | `FDelegateRegistry.inl:19-25` | `Find` 后 `*FoundValue`，**找不到返回 `nullptr`** | ✅ 调用方大多判空（`FRegisterDelegate.cpp:35` 等）；但 `FDelegatePropertyDescriptor.cpp:24` **未判空**，见 F-DEL-003 |
| `GetObject(Class*, Address)` | `FDelegateRegistry.inl:27-33` | 按地址查句柄，找不到返回 `InvalidManagedHandle` | ✅ |
| `AddReference(3 参)` | `FDelegateRegistry.inl:35-41` | **只写 `ManagedHandle2Value`，不写 `Address2ManagedHandle`** | **关键缺陷**：以弱引用方式创建的 helper 无法再按地址查回 → 见 F-DEL-007 |
| `AddReference(6 参)` | `FDelegateRegistry.inl:43-55` | 同时写两个 map，并 `FCSharpEnvironment::AddReference(Owner, new TDelegateReference<...>(InManagedHandle))` | ✅ 完整注册；返回值是 `AddReference` 的结果 |
| `RemoveReference(Class*, IManagedHandle)` | `FDelegateRegistry.inl:57-81` | `GetAddress()` 用作 key 清 `Address2ManagedHandle` → **`delete *FoundValue`** → 清 `ManagedHandle2Value` → **`FDomain::GCHandle_Free(InManagedHandle)`** | ✅ 这是唯一释放 helper 的地方（→ `~FDelegateHelper` → `RemoveFromRoot`），**`AddToRoot` 在此闭合** |

#### 3.9.2 `Public/Reference/TDelegateReference.h`

```cpp
// TDelegateReference.h:6-17
template <typename T>
class TDelegateReference final : public FReference
{
public:
	using FReference::FReference;
public:
	virtual ~TDelegateReference() override
	{
		(void)FCSharpEnvironment::GetEnvironment().RemoveDelegateReference<T>(ManagedHandle);
	}
};
```

| 结论 | 说明 |
|---|---|
| ✅ 生命周期闭合 | C# 侧引用对象析构 → `RemoveDelegateReference` → `FDelegateRegistry::RemoveReference` → `delete helper` + `GCHandle_Free` → `~FDelegateHelper` → `RemoveFromRoot`。**所以 `AddToRoot()` 不是无配对的**——前提是 C# 侧的对象**确实**被析构（见 F-DEL-005 的风险描述）。 |
| ⚠️ 线程 | 析构函数**直接**调用 `RemoveDelegateReference`（`TDelegateReference.h:15`），**没有** `AsyncTask(ENamedThreads::GameThread, ...)` 保护。对比 `FRegisterDelegate::UnRegisterImplementation`（`FRegisterDelegate.cpp:23-27`）**是**显式切回 GameThread 的 → 两条释放路径线程假设不一致，见 F-DEL-012 |

#### 3.9.3 `Private/Domain/Interop/FRegisterDelegate.cpp`（单播的 C# 侧 P/Invoke 入口）

| 注册名 | 实现文件:行 | 注册点 | 说明 |
|---|---|---|---|
| `Register` | `:14-19` | `:178` | `FCSharpBind::Bind<FDelegateHelper>(Class, InManagedObject)` |
| `UnRegister` | `:21-28` | `:179` | **`AsyncTask(ENamedThreads::GameThread, [..]{ RemoveDelegateReference<FDelegateHelper>(..); })`** |
| `Bind` | `:30-49` | `:180` | 4 层判空后 `DelegateHelper->Bind(FoundObject, FoundMethod)` ✅ |
| `IsBound` | `:51-60` | `:181` | ✅ |
| `UnBind` | `:62-69` | `:182` | ✅ |
| `Clear` | `:71-78` | `:183` | ✅ |
| `GenericExecute0` | `:80-87` | `:184` | ✅ |
| `PrimitiveExecute1` / `CompoundExecute1` | `:89-105` | `:185` / `:186` | ✅ |
| `GenericExecute2` | `:107-114` | `:187` | ✅ |
| `PrimitiveExecute3` / `CompoundExecute3` | `:116-134` | `:188` / `:189` | ✅ |
| `GenericExecute4` | `:136-143` | `:190` | ✅ |
| `GenericExecute6` | `:145-153` | `:191` | ✅ |
| `PrimitiveExecute7` / `CompoundExecute7` | `:155-173` | `:192` / `:193` | ✅ |

**核对结果**：`FRegisterDelegate` 共定义 16 个静态函数（含 `Register`/`UnRegister`），构造中注册 16 个（`:178-193`）——**没有"定义了但未注册"的死函数**。所有执行类入口都通过 `DelegateHelper->ExecuteN<>()` 触达 `UDelegateHandler::ExecuteN`，**没有任何直接调用 `UDelegateHandler` 的路径**（`ExecuteN` 的调用者全部是 `FDelegateHelper.h:33-94` 的模板）。

#### 3.9.4 `Private/Reflection/Function/FCSharpDelegateDescriptor.cpp`（回调真正落到 C# 的地方）

```cpp
// FCSharpDelegateDescriptor.cpp:10-27
bool FCSharpDelegateDescriptor::CallDelegate(const UObject* InObject, const FMethodReflection* InMethod, void* InParams)
{
	return Invoke(
		InMethod,
		FCSharpEnvironment::GetEnvironment().GetObject(InObject),   // ← InObject 可能为 nullptr
		[this, InParams](const int32 InIndex) -> void*
		{
			return PropertyDescriptors[InIndex]->ContainerPtrToValuePtr<void>(InParams);
		},
		...
```

| 结论 | 说明 |
|---|---|
| **`InObject` 可为 `nullptr`** | 调用点 `DelegateHandler.cpp:10` 传 `DelegateWrapper.Object.Get()`（`TWeakObjectPtr`，**过期即 nullptr**）；`MulticastDelegateHandler.cpp:13` 传 `Key.Get()`（同样）。`CallDelegate` **不做任何判空**，直接把 nullptr 交给 `FCSharpEnvironment::GetObject(InObject)` 和 `Invoke(InMethod, ...)` → 见 F-DEL-001 |
| **`InMethod` 可为 `nullptr`** | 默认构造的 `FDelegateWrapper{}`（`FDelegateWrapper.h:5-10` 无默认成员初始化器，但 `DelegateHandler.h:158` 的成员在 `UDelegateHandler` 构造时值初始化 → 实际为 `{}`）在 `Bind` 之前即为 `{nullptr, nullptr}`；只要 UE 侧有人在 `Bind` 之前触发 `CSharpCallBack`，`Invoke(nullptr, ...)` 即空指针调用 → 见 F-DEL-001 |
| `PropertyDescriptors[InIndex]` | `TArray::operator[]` 无边界检查；`InIndex` 来自 `Invoke` 内部循环，正常情况安全（低风险） |

---

### 3.10 `FOptionalHelper` 的全部创建/使用点（用于判断所有权与泄漏）

| 创建点 | 文件:行 | `(Data, bNeedFreeData, bNeedFreeProperty)` | 所有权语义 | 结论 |
|---|---|---|---|---|
| `FRegisterOptional::Register1Implementation` | `Private/Domain/Interop/FRegisterOptional.cpp:32` | `(nullptr, true, true)` | helper 自己 `Malloc` + 拥有 property | ✅ 分配/释放配对 |
| `FRegisterOptional::Register2Implementation` | `FRegisterOptional.cpp:57` | `(nullptr, true, true)` | 同上 | ✅ |
| `TPropertyValue::Get(InMember, handle)` | `Public/Binding/Core/TPropertyValue.inl:951` | `(InMember, false, true)` | **借用** UObject/结构体内联的 `TOptional` | ✅ 不 Free 正确 |
| `TPropertyValue::Get<IsReference=true>(InMember)` | `TPropertyValue.inl:983` | `(InMember, false, true)` | 借用 | ✅ |
| `TPropertyValue::Get<IsReference=false>(InMember)` | `TPropertyValue.inl:991` | `(new std::decay_t<T>(*InMember), true, true)` | **堆拷贝**，helper 接管 | ⚠️ 分配器**是匹配的**（UBT 注入 `REPLACEMENT_OPERATOR_NEW_AND_DELETE`，`new` 最终走 `FMemory::Malloc`）；真缺陷是 `FMemory::Free` **不调用 `~T()`** → F-DEL-006 |
| `FOptionalPropertyDescriptor::Get(FMember)` | `Private/.../FOptionalPropertyDescriptor.cpp:13` | `(Src, false, false)` | 借用 UObject 属性内存；`Property` 归描述符所有，不能删 | ✅ |
| `FOptionalPropertyDescriptor::Get(FReturn)` | `FOptionalPropertyDescriptor.cpp:26` | `(Src, true, false)` | `Src` 来自 `FunctionMacro.h:143 ReturnPropertyDescriptor->CopyValue(...)`，即 `TCompoundPropertyDescriptor.inl:35` 的 `FMemory::Malloc` **堆拷贝**，所有权转移给 helper | ✅ `Malloc`↔`Free` 配对正确，**但同样不析构内部值** → F-DEL-006 |

| 释放点 | 文件:行 | 做了什么 |
|---|---|---|
| `FOptionalRegistry::RemoveReference` | `Private/Registry/FOptionalRegistry.cpp:55-77` | `GetAddress()` 清地址映射 → `:69 delete *FoundValue` → `:71` 清句柄映射。**没有 `FDomain::GCHandle_Free`**（对比 `FDelegateRegistry.inl:75`）→ F-DEL-008 |
| `FOptionalRegistry::Deinitialize` | `FOptionalRegistry.cpp:20-39` | 遍历全部 helper `delete` + `FDomain::GCHandle_Free(Key)` + 清两个 map。**这里做了 `GCHandle_Free` 而 `RemoveReference` 没做** → 两者不一致，佐证 F-DEL-008 |
| `FRegisterOptional::UnRegisterImplementation` | `FRegisterOptional.cpp:87-93` | `AsyncTask(ENamedThreads::GameThread, ...)` → `RemoveOptionalReference`（由 `TOptional.cs:14` 的**终结器**触发）✅ 线程处理正确 |

**关于 `TOptional<T>` 的 C# 侧表示是否一致（任务书第 3 问）**

| C++ | C#（`Script/UE/CoreUObject/TOptional.cs`） | 一致性 |
|---|---|---|
| `FOptionalHelper::Reset()` → `MarkUnset` | `:40 Reset()` | ✅ 语义对应 |
| `FOptionalHelper::IsSet()` | `:42 IsSet()` | ✅ |
| `FOptionalHelper::Get()` → `GetValuePointerForReadOrReplace` | `:44 Get()` → 解箱为 `T` | ⚠️ C++ 侧**不检查是否 set** 就返回指针，C# 侧会解箱**未初始化内存** → F-DEL-010 |
| `FOptionalHelper::Set(void*)` | `:46 Set(T InValue)` | ✅ |
| `FOptionalHelper::Identical(A,B)` | `:16-32 operator==` | ✅ 且 null 处理在 C# 侧 |
| **双方均无 `Emplace()`** | 同 | C++ 侧只有 `Set`（`MarkSet...` + `Set` ≈ Emplace）；C# 侧无 `Emplace`/`HasValue` |
| 值语义（`TOptional<T>` 是值类型） | `TOptional<T>` 是 **`class`（引用类型）**（`TOptional.cs:7`） | ⚠️ 表示不一致：C# 侧是"带 GC 句柄的引用对象"；**C# 中不存在拷贝语义**（引用赋值共享同一 handle），因此"拷贝/赋值时旧值是否析构"的风险转化为**别名共享**而非泄漏 → F-DEL-017 |
| `FOptionalHelper` 有 `~` 析构 | `TOptional.cs:14 ~TOptional()` 终结器 → `TOptional_UnRegisterImplementation` | ✅ 路径存在，但由 C# 终结器驱动（非确定性、终结器线程），C++ 侧用 `AsyncTask(GameThread)` 兜住 ✅ |

---

## 4. 绑定 ↔ 解绑配对表

| # | 绑定（文件:行） | 解绑（文件:行） | 是否配对 | 未解绑后果 |
|---|---|---|---|---|
| 1 | `UDelegateHandler::Bind` → `ScriptDelegate->BindUFunction(this, FUNCTION_CSHARP_CALLBACK)` — `DelegateHandler.cpp:60` | `Deinitialize` → `ScriptDelegate->Unbind()` — `DelegateHandler.cpp:38`（**仅当 `bNeedFree`**）；`UnBind()` — `:76`；`Clear()` — `:84` | ⚠️ **部分** | 当 `bNeedFree == false`（外部 `FScriptDelegate`，即 UObject 属性）时，`Deinitialize` 不解绑 → 原本属于该 C# 委托的 `FScriptDelegate` 仍绑定在**即将销毁**的 handler 上 → 见 F-DEL-004 |
| 2 | `UDelegateHandler::Bind` 覆盖 `DelegateWrapper = {InObject, InMethod}` — `DelegateHandler.cpp:64` | **无任何解绑点**（`UnBind()`/`Clear()` 均只动 `ScriptDelegate`，不改 `DelegateWrapper`；`Deinitialize` 也不清） | ❌ **不配对** | 重新 `Bind` 会静默丢弃前一个 C# 目标；`UnBind` 后 `DelegateWrapper` 仍残留旧 `InMethod` 裸指针 → 若之后 UE 侧又触发回调，会用**已解绑的旧方法**执行 |
| 3 | `FDelegateHelper::Initialize` → `DelegateHandler->AddToRoot()` — `FDelegateHelper.cpp:24` | `Deinitialize` → `RemoveFromRoot()` — `FDelegateHelper.cpp:38`（由 `~FDelegateHelper` 触发，`:17`） | ✅ 是（依赖析构 + `FDelegateRegistry` 的 `delete`） | 未删除的 helper 会一直锚定 `UDelegateHandler`；但复核确认 C# 包装对象的终结器由生成器无条件生成（`FDelegateGenerator.cpp:79`）→ 后果是**非确定性驻留**（P2），不是永久泄漏（见 F-DEL-005） |
| 4 | `FMulticastDelegateHelper::Initialize` → `AddToRoot()` — `FMulticastDelegateHelper.cpp:25` | `Deinitialize` → `RemoveFromRoot()` — `FMulticastDelegateHelper.cpp:39` | ✅ 是（同上） | 同上 |
| 5 | `UMulticastDelegateHandler::Initialize` → `new FMulticastScriptDelegate()` — `MulticastDelegateHandler.cpp:32-34` | `Deinitialize` → `RemoveAll(this)` + `delete` — `:45-47`（**仅当 `bNeedFree`**） | ⚠️ **部分** | 内部自建的 delegate 会泄漏；但 `bNeedFree == false` 时本就不该删 → 逻辑正确，**只是没有解绑**（与 #1 同构） |
| 6 | `MulticastScriptDelegate->Add(ScriptDelegate)` — `MulticastDelegateHandler.cpp:85`（`Add`）、`:102`（`AddUnique`） | `RemoveAll(this)` — `:117`（`Remove` 到空时）、`:135`（`RemoveAll` 到空时）；`Clear()` — `:146` | ⚠️ **部分** | `Remove(IObject, IMethod)` 只在 `DelegateWrappers` **清空后**才 `RemoveAll(this)`；只要还剩一个 C# 目标，UE 侧的 `FScriptDelegate` 条目就保留（**设计如此**，因为 UE 侧只存了一个共享条目）。真正的风险是**回调内重入**，见 F-DEL-002 |
| 7 | `FDelegatePropertyDescriptor::NewRef` → `new FDelegateHelper(...)` + `AddDelegateReference(Owner, Address, ...)` — `FDelegatePropertyDescriptor.cpp:39,47` | `FDelegateRegistry::RemoveReference` → `delete *FoundValue` — `FDelegateRegistry.inl:71` | ✅ 是 | — |
| 8 | `FDelegatePropertyDescriptor::NewWeakRef` → `new FDelegateHelper(...)` + **3 参** `AddDelegateReference` — `FDelegatePropertyDescriptor.cpp:56,61` | 只有 C# 侧句柄被回收时才经 `~TDelegateReference` → `RemoveReference` 释放 | ⚠️ **地址映射不配对** | **3 参 `AddReference` 不写 `Address2ManagedHandle`**（`FDelegateRegistry.inl:35-41`）→ 同一属性的后续 `NewRef`/`NewWeakRef` **无法复用**已存在的 helper（`GetDelegateObject(InAddress)` 查不到）→ 每次访问都 `new` 一个 helper + 一个 UObject，见 F-DEL-007 |
| 9 | `FMulticastDelegatePropertyDescriptor::NewRef` / `NewWeakRef` — `FMulticastDelegatePropertyDescriptor.cpp:50,59` / `:68,74` | 同 #7 / #8 | 同 #7/#8 | 同 #8 |
| 10 | `FDelegatePropertyDescriptor::Set` → `DestScriptDelegate->BindUFunction(...)` — `FDelegatePropertyDescriptor.cpp:30` | 无显式解绑；靠 `:26 Property->InitializeValue(Dest)` 覆盖旧值 + UE 属性自身析构 | ✅ 是（交由 UE） | `FScriptDelegate` 只含 `TWeakObjectPtr`+`FName`（引擎 `ScriptDelegates.h:180-186` 只写这两个字段），**无堆所有权** → 重复 `InitializeValue` 不构成泄漏；多播版的 `FMulticastDelegatePropertyDescriptor.cpp:27` 目标若已持有 `InvocationList`（`TArray`）则属"活值 Dest"族，见第 8 节 #5 |
| 11 | `FDelegateRegistry.inl:52-54` `AddReference(Owner, new TDelegateReference<T>(Handle))` | `TDelegateReference.h:15` `~TDelegateReference` → `RemoveDelegateReference` | ✅ 是 | — |
| 12 | `NewObject<UDelegateHandler>()` — `FDelegateHelper.cpp:22`（及多播版 `FMulticastDelegateHelper.cpp:23`） | 无显式 `delete`；依靠 `RemoveFromRoot()` 后被 GC | ⚠️ **非确定性** | `RemoveFromRoot` 只是"允许 GC"，实际回收要等下一个 GC 周期；且若该 UObject 被别处引用则永不回收 |
| 13 | `FDomain::GCHandle_Free(InManagedHandle)` — `FDelegateRegistry.inl:75` | 对应的 `GCHandle_Alloc`/`NewObject` 分配点**未在本组文件内** | 待确认 | 见第 8 节未覆盖项 |

**小结**：`AddToRoot`/`RemoveFromRoot`（#3/#4）和 `new`/`delete helper`（#7/#11）这两组**是配对的**；真正不配对的是**单播的 `DelegateWrapper` 清理**（#2，无任何解绑点）和**弱引用路径的地址映射**（#8）。

---

## 5. GC 引用方式表（C# 委托对象强/弱引用）

### 5.1 结论先行（对任务书核心问题的回答）

任务书设想了两种可能：**（A）强引用 → C# 委托永不回收（P1 泄漏）**，或 **（B）仅存 `IntPtr` → 调用悬垂指针（P0 崩溃）**。

**实测结论：本层是 (B) 的变体，并且两侧同时存在不同性质的缺陷。**

- **C++ 侧对 C# 委托对象既不 `GCHandle.Alloc(Normal)` 强引用，也不做弱引用封装**；它存的是一组"三件套"：
  1. `IManagedHandle`（C# 对象的 GC 句柄，存在 `FDelegateRegistry` 的 map 里）；
  2. `TWeakObjectPtr<UObject> Object` —— **UE 对象的弱引用**；
  3. `FMethodReflection* Method` —— **裸指针**。
- 因此 **不存在"C++ 强引用 C# 委托导致永不回收"这一泄漏路径**（任务书假设 A 不成立）。
- 但存在**悬垂/静默失效**：`Object` 是弱引用，`Method` 是裸指针，**桥接层没有任何机制在二者失效时清理或跳过该条目**。→ F-DEL-001（复核后 **P1**：`Method` 子路径不可达、`Object` 失效子路径在 LeanCLR 下为静默失败）。
- **UE 侧的泄漏是"非确定性驻留"而非永久泄漏**：`AddToRoot()` 锚定 handler UObject + `new FDelegateHelper`，释放点有两条（`FDelegateRegistry::RemoveReference` 与 `FDelegateRegistry::Deinitialize`），且 C# 包装对象的终结器由生成器无条件生成（`FDelegateGenerator.cpp:79`/`:451`）→ 见 F-DEL-005（复核后 **P2**）；`NewWeakRef` 路径**每次都新建且不写地址映射**造成的重复分配才是主要资源问题 → F-DEL-006/007（**P1 / P2**）。

### 5.2 引用方式明细表

| # | 持有点（谁持有） | 文件:行 | 持有的类型 | 强/弱 | 后果 |
|---|---|---|---|---|---|
| 1 | `UDelegateHandler::DelegateWrapper.Object` | `Public/Reflection/Delegate/FDelegateWrapper.h:7` | `TWeakObjectPtr<UObject>` | **弱** | C# 目标 UObject 被 GC 后 `Get()` 返回 `nullptr`，条目**不被清理** → `CallDelegate(nullptr, Method, Parms)`（`DelegateHandler.cpp:10`）→ F-DEL-001 |
| 2 | `UDelegateHandler::DelegateWrapper.Method` | `FDelegateWrapper.h:9` | `FMethodReflection*` | **裸指针，无所有权** | 若 `FMethodReflection` 所属 `FClassReflection` 被销毁（域重载/热重载），指针悬垂 → `Invoke(InMethod, ...)` → `FManagedFunctionDescriptor.cpp:49 InMethod->Runtime_Invoke(...)` → **野指针调用，P0** |
| 3 | `UMulticastDelegateHandler::DelegateWrappers` | `Public/Reflection/Delegate/MulticastDelegateHandler.h:121` | `TArray<FDelegateWrapper>` | 同上（弱 + 裸） | 同上，且**元素永远不会因对象死亡而自动移除** → 广播时对每个失效条目都走一次空指针路径 |
| 4 | `UDelegateHandler::ScriptDelegate` | `Public/Reflection/Delegate/DelegateHandler.h:154` | `FScriptDelegate*`（`new` 或外部地址） | **强（裸所有权）** | `bNeedFree` 决定是否 `delete`；外部地址情形不 delete —— 正确 |
| 5 | `UDelegateHandler::DelegateDescriptor` | `DelegateHandler.h:156` | `FCSharpDelegateDescriptor*`（`new`） | **强（裸所有权）** | `Deinitialize` 中 `delete`（`DelegateHandler.cpp:48`）✅ |
| 6 | `FDelegateHelper::DelegateHandler` | `Public/Reflection/Delegate/FDelegateHelper.h:101` | `TWeakObjectPtr<UDelegateHandler>` | **弱** | 弱指针**却**是唯一访问通路；之所以不失效，是因为 #7 的 `AddToRoot()` 把它钉住 → 二者耦合，见 F-DEL-005 |
| 7 | `UDelegateHandler` UObject 自身的 GC 根 | `Private/Reflection/Delegate/FDelegateHelper.cpp:24` | `AddToRoot()` | **强（GC root）** | 该 UObject **永不被 GC**，直到 `RemoveFromRoot()`（`:38`）。释放路径只有 `FDelegateRegistry::RemoveReference`（`FDelegateRegistry.inl:71 delete` → `~FDelegateHelper`）→ F-DEL-005 |
| 8 | `FDelegateHelper` 对象的持有者 | `Public/Registry/FDelegateRegistry.h:43,47` | `TMap<IManagedHandle, FDelegateHelper*>` | **强（裸指针，手动 delete）** | 由 `RemoveReference`（`FDelegateRegistry.inl:71`）显式 `delete`；C# 包装对象终结器会驱动 `UnRegister`（生成器保证，`FDelegateGenerator.cpp:79`），故"从不释放"不成立；残留的是"回收被 GC/`AsyncTask` 延迟"（P2，见 F-DEL-005） |
| 9 | 同上，地址索引 | `Public/Registry/FDelegateRegistry.h:45,49` | `TMap<void*, IManagedHandle>` | **强（值语义句柄）** | key 是 `Address`（= UObject 内联属性地址，`FDelegatePropertyDescriptor.cpp:39`）→ **UObject 被 GC 后 key 悬垂**，见 F-DEL-013 |
| 10 | `UMulticastDelegateHandler::ScriptDelegate` | `MulticastDelegateHandler.h:119` | `FScriptDelegate`（**值成员**） | 强（值） | 被 `MulticastScriptDelegate` 持有的是**这个值的一份拷贝**（`:85 MultAdd(ScriptDelegate)`），因此 UE 侧只看到一个条目 |
| 11 | `FDelegateRegistry.inl:52-54` 的 `TDelegateReference` | `Public/Reference/TDelegateReference.h:7-16` | `FReference` 子对象，存 `ManagedHandle` | **强（C++ 侧引用计数对象）** | 其析构触发 `RemoveDelegateReference` → 这才是释放路径的起点 ✅ |
| 12 | C# 侧 `TOptional<T>` / 委托对象的实际对象 | `Script/UE/CoreUObject/TOptional.cs:7-48` | C# 引用类型，句柄经 `HandleData.GetHandle` / `HandleData.Alloc` | **由 C# GC 决定** | C++ 侧只存 `IManagedHandle`（整数），**不构成 GC root** → C# 委托可被回收，UE 侧条目随之失效（与 #1/#2 共同构成悬垂） |

### 5.3 一句话回答

> **不是"强引用泄漏"，而是"弱引用 + 裸指针 + 无失效清理"的悬垂组合（F-DEL-001，复核后 P1：LeanCLR 下表现为静默失败而非崩溃）；UE 侧则是"`AddToRoot` 锚定 + 弱引用路径重复 `new`"造成的资源驻留与放大（F-DEL-005 P2 / F-DEL-007 P2）与 `FOptionalHelper` 的析构缺失泄漏（F-DEL-006 P1）。三者的方向都与任务书假设 A（强引用导致 C# 委托永不回收）不同。**

### 5.4 `Exec` 与 `Deinitialize` 判空的对照（F-DEL-001 的证据链）

| 位置 | `InObject`/`Key.Get()` 判空 | `InMethod`/`Value` 判空 |
|---|---|---|
| `UDelegateHandler::ProcessEvent` — `DelegateHandler.cpp:4-17` | ❌ 无 | ❌ 无 |
| `UMulticastDelegateHandler::ProcessEvent` — `MulticastDelegateHandler.cpp:5-21` | ❌ 无 | ❌ 无 |
| `FCSharpDelegateDescriptor::CallDelegate` — `FCSharpDelegateDescriptor.cpp:10-27` | ❌ 无 | ❌ 无 |
| `FManagedFunctionDescriptor::Invoke` — `FManagedFunctionDescriptor.cpp:5-49` | ❌ 无（`InManagedHandle` 可为 `InvalidManagedHandle`） | ❌ **`:49 InMethod->Runtime_Invoke(...)` 直接解引用** |

全链路上**没有任何一层**对 `Method == nullptr` 做检查（这是本条最硬的代码事实）；但复核进一步确认：UE 侧 `ProcessDelegate`（`ScriptDelegates.h:432-459`）用 `checkf(Object.IsValid())` + `FindFunctionChecked` 保证"进入 `ProcessEvent` 时桥对象有效"，而单播 `ScriptDelegate` 的绑定恰好发生在 `Bind()` 内部（`DelegateHandler.cpp:58-61`）——因此 `Method == nullptr` 这一子路径在现有代码里不可达，实际可达的是 `Object` 失效子路径（定级 P1）。

---

## 6. 发现清单

> 排序：分组按**初判定级**（P0 → P3）排列以保持 Finding 编号稳定；**组内每条的最终级别见其 `严重度` 字段**。
> 实际级别分布在组标题中标注（2 条调整：F-DEL-006 上调 P1、F-DEL-005 下调 P2）。

### P0 组（初判定级 P0 / 3 条 → 实际：F-DEL-002 = **P0**、F-DEL-003 = **P0**、F-DEL-001 = **P1**）

#### [F-DEL-001] 委托回调链全程不校验 `Method`/`Object`，空指针直达 `InMethod->Runtime_Invoke`

- **类别**: Bug / 未定义行为
- **严重度**: **P1**（由 P0 校正为 P1）
- **复核结论**: 部分确认 —— "全链 4 层无 `Method`/`Object` 判空"这一代码事实成立（`DelegateHandler.cpp:10`、`FCSharpDelegateDescriptor.cpp:10-14`、`FManagedFunctionDescriptor.cpp:49`），且 `InMethod->Runtime_Invoke` 是**非虚成员函数调用**（`FMethodReflection.h:36-37`），`InMethod == nullptr` 必崩；但 `Method == nullptr` 触发条件经引擎与插件源码核对**不成立**（见复核证据），真正可达的只有 `Object` 失效子路径
- **可达性**: 活跃（**仅** `Object` 失效子路径；`Method == nullptr` 子路径在当前代码里不可达）
- **复核证据**: ①无判空链：`Source/UnrealCSharp/Private/Reflection/Delegate/DelegateHandler.cpp:10`、`Source/UnrealCSharp/Private/Reflection/Function/FCSharpDelegateDescriptor.cpp:10-14`、`Source/UnrealCSharp/Private/Reflection/Function/FManagedFunctionDescriptor.cpp:49`（已读到 `:91`，`:50-91` 无任何判空）。②`Source/UnrealCSharp/Private/Registry/FObjectRegistry.cpp:53-58`：`Object2ManagedHandle.Find(nullptr)` 落空 → 返回 `InvalidManagedHandle`（经 `FCSharpEnvironment.cpp:533-536` 转发），故 `Object` 失效确实变成 `Runtime_Invoke(InvalidManagedHandle, ...)`。③引擎侧 `Engine\Source\Runtime\Core\Public\UObject\ScriptDelegates.h:159-170,193-197`：`IsBound()` = `FunctionName != NAME_None && Object.Get() != nullptr && FindFunction(FunctionName) != nullptr`；`:432-459` `ProcessDelegate()` 在调用 `ProcessEvent` 前有 `checkf(Object.IsValid())`、`checkf(FunctionName != NAME_None)`、`FindFunctionChecked` → **UE 保证桥对象存活**，而单播的 `ScriptDelegate` 恰在 `Bind()` 内（`DelegateHandler.cpp:58-61`）才被绑定、紧随其后 `:64` 写入 `DelegateWrapper` → "`Bind` 之前被回调"没有可达旁路。
- **级别变动**: 无（P1 维持；理由：`Method` 子路径不可达，`Object` 失效子路径在 LeanCLR 下表现为托管侧失败/静默无回调而非进程崩溃，见"LeanCLR 下托管异常不终止进程，走 RtResult 返回值协议"，属功能错误而非崩溃/数据损坏）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Delegate/DelegateHandler.cpp:10`
- **函数**: `UDelegateHandler::ProcessEvent(UFunction*, void*)`（同类问题：`UMulticastDelegateHandler::ProcessEvent`、`FCSharpDelegateDescriptor::CallDelegate`）
- **置信度**: 高

**现状（代码事实）**
```cpp
// DelegateHandler.cpp:4-17
void UDelegateHandler::ProcessEvent(UFunction* Function, void* Parms)
{
	if (Function != nullptr && Function->GetName() == FUNCTION_CSHARP_CALLBACK)
	{
		if (DelegateDescriptor != nullptr)
		{
			DelegateDescriptor->CallDelegate(DelegateWrapper.Object.Get(), DelegateWrapper.Method, Parms);
			//                                ^^^^^^^^^ 弱指针，过期即为 nullptr   ^^^^^^^^^^^^^^ 裸指针，从未判空
		}
	}
	...
}
```
```cpp
// MulticastDelegateHandler.cpp:11-14
for (const auto& [Key, Value] : DelegateWrappers)
{
	DelegateDescriptor->CallDelegate(Key.Get(), Value, Parms);
}
```
```cpp
// FCSharpDelegateDescriptor.cpp:10-14 —— 不判空
bool FCSharpDelegateDescriptor::CallDelegate(const UObject* InObject, const FMethodReflection* InMethod, void* InParams)
{
	return Invoke(
		InMethod,
		FCSharpEnvironment::GetEnvironment().GetObject(InObject),
```
```cpp
// FManagedFunctionDescriptor.cpp:49 —— 最终解引用，无任何判空
	if (auto ReturnValue = InMethod->Runtime_Invoke(InManagedHandle, CSharpParams.GetData());
```

**调用上下文**
UE 反射系统调用 `ProcessEvent`（绑定点：`DelegateHandler.cpp:60` `ScriptDelegate->BindUFunction(this, *FUNCTION_CSHARP_CALLBACK)`；`MulticastDelegateHandler.cpp:83`/`:100`），随后 `DelegateHandler.cpp:10` → `FCSharpDelegateDescriptor.cpp:12` → `FManagedFunctionDescriptor.cpp:49`。
`DelegateWrapper` 只有在 `Bind()`（`DelegateHandler.cpp:64`）之后才有有效值；`UDelegateHandler` 的成员 `FDelegateWrapper DelegateWrapper`（`DelegateHandler.h:158`）默认是 `{nullptr, nullptr}`（`FDelegateWrapper.h:6-10` 无默认成员初始化器，值初始化后 `Object` 为空弱指针、`Method` 为 `nullptr`）。

**问题**
两条独立的 nullptr 路径，都通往 `FManagedFunctionDescriptor.cpp:49` 的裸解引用：
1. **`Method == nullptr`**：只要 UE 在 `Bind()` 之前触发一次 `CSharpCallBack`（例如外部代码拿到属性上的 `FScriptDelegate*` 后调用 `Execute`，或在 `Bind` 失败而其上层忽略了 `:58 !IsBound()` 分支的情况下）。`FRegisterDelegate::BindImplementation` 有 4 层判空（`FRegisterDelegate.cpp:35-47`），所以正常 API 路径不会制造这种状态——但**桥接层自身没有防线**，任何一条旁路（蓝图 Spawn/手工 `NewObject`、序列化反序列化一个已绑定的委托属性）都能触发。
2. **`Object == nullptr`**：C# 目标 UObject 已被 GC。`FDelegateWrapper::Object` 是 `TWeakObjectPtr`（`FDelegateWrapper.h:7`），**过期时不返回 nullptr 而是让 `Get()` 返回 nullptr**，桥接层**没有任何机制在此时移除该条目**（见 F-DEL-004）。此时 `GetObject(nullptr)` 会得到 `InvalidManagedHandle`，随后 `Runtime_Invoke(InvalidManagedHandle, ...)` 的行为取决于后端（Mono/CoreCLR/LeanCLR）——最好的情况是静默失败，最坏情况是崩溃。

**建议**
在 `ProcessEvent` 与 `CallDelegate` 两级都加显式校验，并把失效条目就地清理：
```cpp
// DelegateHandler.cpp:4 附近
if (DelegateWrapper.Object.IsValid() && DelegateWrapper.Method != nullptr)
{
    DelegateDescriptor->CallDelegate(DelegateWrapper.Object.Get(), DelegateWrapper.Method, Parms);
}
// 可选：一旦失效，顺手 Clear()/RemoveAll，避免每次广播都走这条路
```
多播版在遍历时**不能**就地 `Remove`（见 F-DEL-002），需改为"两阶段"：
```cpp
// MulticastDelegateHandler.cpp:11
TArray<FDelegateWrapper> Snapshot = DelegateWrappers;   // 或 CompactionArray 模式
for (const auto& [Key, Value] : Snapshot)
{
    if (Key.IsValid() && Value != nullptr)
    {
        DelegateDescriptor->CallDelegate(Key.Get(), Value, Parms);
    }
}
```
并考虑在 `FManagedFunctionDescriptor::Invoke` 开头加 `check(InMethod != nullptr)`（Shipping 下用 `if (InMethod == nullptr) return false;`）。

**验证方式**
`grep -n "CallDelegate" Source/UnrealCSharp`（3 个调用点，均无判空）；`grep -n "Runtime_Invoke" Source/UnrealCSharp/Private/Reflection/Function/FManagedFunctionDescriptor.cpp`（`:49`）。
用例：在 C# 中 `var d = new FDelegate(); d.Execute();`（不 Bind），或在 UE 中把已绑定的委托属性所在 UObject 置为 `MarkAsGarbage()` 后手动 `Execute`。

---

#### [F-DEL-002] 多播广播遍历 `TArray` 期间回调可修改该数组 → 迭代器失效/越界

- **类别**: Bug / 未定义行为
- **严重度**: **P0**（崩溃）
- **复核结论**: **确认（成立，P0 维持）** —— 回调在 range-for 循环体内同步执行、且 `Remove`/`RemoveAll`/`Clear` 在遍历中修改同一 `TArray`，补齐引擎侧两级后果证据（ensure + use-after-free）
- **可达性**: 活跃（`FRegisterMulticastDelegate.cpp:106/127/139` 注册了 `Remove`/`RemoveAll`/`Clear`，C# 侧 `FMulticastDelegateImplementation.cs:30-35`（Contains）/`:73-77`（Clear）证明这些入口真实可用）
- **复核证据**: ①插件侧：`Source/UnrealCSharp/Private/Reflection/Delegate/MulticastDelegateHandler.cpp:11-13`（遍历中同步 `CallDelegate`）、`:111`（`Remove` 改数组）、`:126`（`RemoveAll`）、`:149`（`Clear` → `Empty()`）；`MulticastDelegateHandler.cpp:83`/`:100` 是唯一两个 UE 侧绑定点。②引擎侧**迭代器事实**：UE 5.6 `Engine\Source\Runtime\Core\Public\Containers\Array.h:3377-3379,3395-3399` 的 range-for 迭代器在非 Shipping/Test 下是 `TCheckedPointerIterator`（定义见 `:198-272`），它持有 `Data` 指针 + `ArrayNum` 的**引用** + `InitialNum` 快照，`end()` 只在循环开始时求值一次（`GetData() + Num()`，`:3398`）→ 数组缩小后 `end` 指针是陈旧的。③引擎侧**检测机制**：`Array.h:256-265` `operator!=` 内含 `ensureMsgf(CurrentNum == InitialNum, TEXT("Array has changed during ranged-for iteration!"))` → Editor/Development 构建会立即报 ensure（Shipping/Test 下 `TARRAY_RANGED_FOR_CHECKS 0`，见 `Array.h:36-40`，检查被编译掉）。④引擎侧**内存后果分两类**：(a) `Remove`/`RemoveAll`（`Array.h:3118-3162`）只做 `RelocateConstructItems` + `DestructItems` 并把 `ArrayNum` 改小，**不释放底层分配** → 循环继续读到已移位/尾部的陈旧元素 → 该回调被**重复调用**或**被跳过**（`FDelegateWrapper` 平凡可析构，元素内存仍被占用）；(b) `Clear()` → `DelegateWrappers.Empty()` → `Array.h:2286-2312` 在 `ArrayMax != 0` 时经 `ReallocTo(NewMax = 0)`（`Array.h:557-590`）调用 `ResizeAllocation(..., 0, ...)`（`ContainerAllocationPolicies.h:700-724` → `FMemory::Realloc(Data, 0)`）**释放并置空底层分配** → 迭代器的 `Ptr` 悬垂，下一次 `*Ptr` 即 **use-after-free 读取已释放内存**（真崩溃路径）。
- **级别变动**: 无（P0 维持；理由：`Clear()`/`Add()` 发生在回调内时构成 use-after-free，属崩溃/数据损坏级）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Delegate/MulticastDelegateHandler.cpp:11`
- **函数**: `UMulticastDelegateHandler::ProcessEvent(UFunction*, void*)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// MulticastDelegateHandler.cpp:5-21 —— 遍历时直接回调 C#
void UMulticastDelegateHandler::ProcessEvent(UFunction* Function, void* Parms)
{
	if (Function != nullptr && Function->GetName() == FUNCTION_CSHARP_CALLBACK)
	{
		if (DelegateDescriptor != nullptr)
		{
			for (const auto& [Key, Value] : DelegateWrappers)   // ← 引用绑定到 TArray 元素
			{
				DelegateDescriptor->CallDelegate(Key.Get(), Value, Parms);  // ← 同步进入 C#
			}
		}
	}
```
```cpp
// MulticastDelegateHandler.cpp:109-122 —— 回调内可被 C# 调到
void UMulticastDelegateHandler::Remove(UObject* InObject, FMethodReflection* InMethod)
{
	DelegateWrappers.Remove({InObject, InMethod});   // ← 修改正在被遍历的数组
	...
}
// :124-140  RemoveAll 同样 RemoveAll(...)
// :142-152  Clear 同样 Empty()
```

**调用上下文**
`ProcessEvent` 由 UE 反射触发（`FScriptDelegate` 绑定见 `:83`/`:100`）。`Remove`/`RemoveAll`/`Clear` 的 C# 入口是 `FRegisterMulticastDelegate.cpp:106/127/139`（注册名 `Remove`/`RemoveAll`/`Clear`）。回调路径 `ProcessEvent → CallDelegate → FManagedFunctionDescriptor::Invoke → Runtime_Invoke` 是**同步**调用（`FManagedFunctionDescriptor.cpp:49`），即 C# 回调代码在遍历的循环体内执行。

**问题**
**[引擎核对]** UE 5.6 的 range-for 迭代器是 `TCheckedPointerIterator`（**指针式**，不是索引式），它只在循环开始时求值一次 `end`（即持有 end 快照）；同时它带一个 `ensureMsgf` 尺寸变化检测。机制与后果如下：

`TArray::Remove` / `RemoveAll` / `Empty` 会在回调执行期间修改同一个数组；`for (const auto& [Key, Value] : DelegateWrappers)` 的 `end` 在循环开始时求值一次，因此：
- **`Remove`/`RemoveAll`（缩小但不释放，`Array.h:3118-3162`）**：C# 回调中调用 `RemoveAll(this)`（常见写法：收到事件后注销自己）→ 元素左移、`ArrayNum` 变小，但循环仍按陈旧的 `end` 指针推进 → 后续迭代读到**已移位/尾部陈旧元素** → 表现为**某个回调被重复调用、某个回调被静默跳过**（而不是立即崩溃）。本工程 Editor/Development 构建下会先报 `ensureMsgf`：`"Array has changed during ranged-for iteration!"`（`Array.h:263`）。
- **`Clear()` → `DelegateWrappers.Empty()`（真 use-after-free，`Array.h:2286-2312` + `:557-590` + `ContainerAllocationPolicies.h:700-724`）**：`Empty(Slack=0)` 在 `ArrayMax != 0` 时会把底层分配 `FMemory::Realloc(Data, 0)` **释放掉**，迭代器 `Ptr` 立即悬垂，下一次 `*Ptr` 即读取已释放内存 → **崩溃/数据损坏**（这也是本条定为 P0 的核心理由）。
- 若回调内调用 `Add`/`AddUnique` 使数组扩容，`Data` 会重分配 → 迭代器同样悬垂。
- 若 `RemoveAll` 使数组变空，`RemoveAll(this)` + `ScriptDelegate.Unbind()`（`:117`/`:135`）还会反绑 UE 侧委托，而外层循环可能仍在继续。
- 任务书提出的两种安全写法（**`RemoveAll` 后重新开始** / **拷贝一份再遍历**）**当前实现两者都没有**。

**建议**
采用 UE 官方推荐的多播遍历模式——快照或 CompactionArray：
```cpp
// 方案 A（最小改动，语义等价于 UE 的 AddUniqueDynamic 宽容行为）
TArray<FDelegateWrapper> Snapshot(DelegateWrappers);   // 拷贝，代价 N 次 TWeakObjectPtr + 指针拷贝
for (const auto& [Key, Value] : Snapshot)
{
    if (Key.IsValid() && Value != nullptr)
    {
        DelegateDescriptor->CallDelegate(Key.Get(), Value, Parms);
    }
}
```
```cpp
// 方案 B（无拷贝，UE 的 CompactionArray 模式，允许遍历中被 Remove）
// 用 FDelegateWrapper 的索引 + RemoveAll 的返回值做压实，遍历结束再统一删
```
推荐方案 A：当前 `FDelegateWrapper` 只有 12 字节（`TWeakObjectPtr` 8 + 裸指针 8，视平台），拷贝成本可接受；且 `RemoveAll` 的语义"移除后不再被本次广播调用"对 C# 语义更自然（与 UE `AddUniqueDynamic` 一致）。
另外应加防重入标志，避免广播中再次 `Broadcast` 造成递归。

**验证方式**
`grep -n "DelegateWrappers" Source/UnrealCSharp`（5 处：`MulticastDelegateHandler.h:121`、`.cpp:11,72,111,126,149`）。
用例：C# 注册两个多播回调，第一个回调内 `RemoveAll(this)` 或 `Remove(self, method)`，然后触发 UE 广播 → 期望在 ASAN/`-fsanitize=address` 或 Debug 版 UE 的 `TArray` 边界检查下报越界。

---

#### [F-DEL-003] 委托属性 `Set` 未校验 `GetDelegate<>()` 返回值 → 空指针解引用

- **类别**: Bug（空指针解引用）
- **严重度**: **P0**（崩溃）
- **复核结论**: **确认（成立，P0 维持）** —— `GetDelegate<>()` 会返回 `nullptr`，而 `:30` 直接在其结果上调用成员函数；**最短触发路径**：C# 侧传入一个 `FDelegateRegistry` 里不存在的句柄（含 `0`/`InvalidManagedHandle`，例如把 `null` 的 `FDelegate` 字段赋给委托属性，或对该句柄 `HandleData.GetHandle()` 因 `ConditionalWeakTable` 未命中而返回 0，见 `Script/Interop/Handle/HandleData.cs:80-93`）
- **可达性**: 活跃
- **复核证据**: `Source/UnrealCSharp/Private/Reflection/Property/DelegateProperty/FDelegatePropertyDescriptor.cpp:22-30`（`GetDelegate<FDelegateHelper>` 结果**未判空**即 `:30 SrcDelegateHelper->GetUObject()`）；多播同构 `FMulticastDelegatePropertyDescriptor.cpp:22-34`；`Source/UnrealCSharp/Public/Registry/FDelegateRegistry.inl:19-25`（`return FoundValue != nullptr ? *FoundValue : nullptr;` 明确可返回空）；`Source/UnrealCSharp/Public/Registry/FDelegateRegistry.inl:38`（3 参 `AddReference` 只写 `ManagedHandle2Value`，键不存在即查不到）；对照 `Source/UnrealCSharp/Private/Domain/Interop/FRegisterDelegate.cpp:35-48`（同一 API 的 4 层判空）
- **级别变动**: 无（P0 维持；理由：对标 `_CONVENTIONS.md` 的 P0=崩溃，且这不是"防御性编程建议"——`HandleData.GetHandle()` 对无句柄对象返回 `0` 是正常语义，`null` 委托赋值即可命中）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Property/DelegateProperty/FDelegatePropertyDescriptor.cpp:30`
- **函数**: `FDelegatePropertyDescriptor::Set(void*, void*)`（同类问题：`FMulticastDelegatePropertyDescriptor::Set`，`.../FMulticastDelegatePropertyDescriptor.cpp:33`）
- **置信度**: 高

**现状（代码事实）**
```cpp
// FDelegatePropertyDescriptor.cpp:20-31
void FDelegatePropertyDescriptor::Set(void* Src, void* Dest) const
{
	const auto SrcManagedHandle = *static_cast<IManagedHandle*>(Src);

	const auto SrcDelegateHelper = FCSharpEnvironment::GetEnvironment().GetDelegate<FDelegateHelper>(SrcManagedHandle);

	Property->InitializeValue(Dest);

	const auto DestScriptDelegate = Property->GetPropertyValuePtr(Dest);

	DestScriptDelegate->BindUFunction(SrcDelegateHelper->GetUObject(), SrcDelegateHelper->GetFunctionName());
	//                     ^^^^^^^^^^^^^^^^^^ 未判空
}
```
```cpp
// FMulticastDelegatePropertyDescriptor.cpp:20-36 —— 同样未判空
	const auto SrcMulticastDelegateHelper = FCSharpEnvironment::GetEnvironment().GetDelegate<
		FMulticastDelegateHelper>(SrcManagedHandle);
	...
	ScriptDelegate.BindUFunction(SrcMulticastDelegateHelper->GetUObject(),
	                             SrcMulticastDelegateHelper->GetFunctionName());
```
而 `GetDelegate` 明确会返回空：
```cpp
// FDelegateRegistry.inl:19-25
static auto GetDelegate(Class* InRegistry, const IManagedHandle InManagedHandle)
	-> typename FDelegateValueMapping::ValueType
{
	const auto FoundValue = (InRegistry->*ManagedHandle2Value).Find(InManagedHandle);
	return FoundValue != nullptr ? *FoundValue : nullptr;   // ← 可返回 nullptr
}
```

**调用上下文**
`Set` 由属性描述符的通用赋值路径调用（C# 侧给一个 `FDelegate` 类型的属性赋值，例如 `Actor.MyDelegate = someDelegate;`）。对照同一插件中**唯一正确判空的写法**：`FRegisterDelegate.cpp:35-48`（4 层判空）、`FRegisterMulticastDelegate.cpp` 各 Implementation、`FRegisterOptional.cpp:97/105/117/129`（`if (const auto X = ...GetOptional(...))`）。

**问题**
`SrcManagedHandle` 来自 C# 传入的句柄。若该句柄对应一个**已被析构/已 `UnRegister`** 的委托对象（C# 中委托对象被 GC，而属性赋值发生在其终结器完成之后；或跨域/热重载后句柄失效），`GetDelegate<FDelegateHelper>` 返回 `nullptr`，`SrcDelegateHelper->GetUObject()` 即对 `nullptr` 调用非虚成员函数（`FDelegateHelper.cpp:73-76`）→ **访问违例崩溃**。
额外问题：`Property->InitializeValue(Dest)`（`:26`）在**已持有旧绑定**的属性上重复调用，覆盖旧 `FScriptDelegate` 前没有 `Clear`/`Unbind`；`FDelegateProperty` 的 `InitializeValue` 语义是否容忍重复调用需要确认（置信度: 低，见第 8 节）。

**建议**
```cpp
const auto SrcDelegateHelper = FCSharpEnvironment::GetEnvironment().GetDelegate<FDelegateHelper>(SrcManagedHandle);
if (SrcDelegateHelper == nullptr)
{
	Property->ClearValue(Dest);          // 或至少确保 Dest 处于已初始化状态
	return;
}
```
多播版同理。更彻底的做法是让 `Set` 返回 `bool` 表示成功，并由属性描述符层决定是否向 C# 抛异常。

**验证方式**
`grep -rn "GetDelegate<" Source/UnrealCSharp`（对比 `FRegisterDelegate.cpp:35` 的判空与 `FDelegatePropertyDescriptor.cpp:24` 的缺失）。
用例：C# 中构造一个 `FDelegate`，赋值给属性后不保留引用，强制 `GC.Collect()` + `WaitForPendingFinalizers()`，再对**同一个**属性赋第二个（已回收的）委托句柄。

---

### P1 组（初判定级 P1 / 5 条 → 实际：F-DEL-004 = **P1**、F-DEL-006 = **P1**、F-DEL-008 = **P1**、F-DEL-005 = **P2**、F-DEL-007 = **P2**）

#### [F-DEL-004] 单播 `DelegateWrapper` 没有任何解绑点，`UnBind`/`Clear`/`Deinitialize` 都不清理 C# 目标

- **类别**: Bug（状态残留/静默失效）
- **严重度**: **P1**
- **复核结论**: 部分确认 —— "`DelegateWrapper` 无任何清理点"是**代码事实**（已 grep 复核）；但"`UnBind` 后 UE 侧仍可能用旧 `Method` 回调"的论证**不成立**，真正可达的是两条危害
- **可达性**: 活跃
- **复核证据**: ①事实：`DelegateHandler.cpp:64`（唯一写入点）、`:10`（唯一读取点）、`:72-78`（`UnBind` 只 `ScriptDelegate->Unbind()`）、`:80-86`（`Clear` 只 `ScriptDelegate->Clear()`）、`:32-52`（`Deinitialize` 不触碰 wrapper）——全文再无其他 `DelegateWrapper`（**复核 grep：模式 `\bDelegateWrapper\b`，路径 `Source/`，命中 3 处 = `DelegateHandler.h:158`、`DelegateHandler.cpp:10`、`DelegateHandler.cpp:64`**；多播的 `DelegateWrappers` 是另一个符号，因词尾 `s` 不匹配而不计入）。②**引擎侧核对**：引擎 `ScriptDelegates.h:159-170,193-197` 的 `IsBound()` 要求 `Object.Get() != nullptr && FindFunction(FunctionName) != nullptr`，`Unbind()`（`:240-253`）会清空 `Object`/`FunctionName`，`ProcessDelegate()`（`:432-459`）在调用 `ProcessEvent` 前还有 `checkf(Object.IsValid())` → **UnBind 之后 UE 不会再进入 `ProcessEvent`**，"用已解绑的旧方法执行"没有可达路径。③**真正可达的危害**：(a) `Bind()`（`:56-62`）只在 `!ScriptDelegate->IsBound()` 时才把桥绑到 `CSharpCallBack` → 若该委托属性**已被外部绑定**（蓝图 `Create Event`、或上一次运行遗留的绑定），桥**永不接管**，`DelegateWrapper` 却已被写入 → C# 回调静默不触发（功能错误）；(b) `bNeedFree == false`（UObject 属性内联 `FScriptDelegate`，`FDelegatePropertyDescriptor.cpp:39`）时 `Deinitialize` 不解绑 → 宿主桥对象 `RemoveFromRoot` 后被 GC，该属性留下指向已销毁 UObject 的绑定（因弱引用而"静默变未绑定"）。
- **级别变动**: 无（P1 维持；理由：危害是**静默功能失效**（P1=功能错误），但既不构成泄漏——`FDelegateWrapper` 只含 `TWeakObjectPtr`+裸指针、无所有权（`FDelegateWrapper.h:7,9`），也不构成崩溃）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Delegate/DelegateHandler.cpp:64`
- **函数**: `UDelegateHandler::Bind` / `UnBind` / `Clear` / `Deinitialize`
- **置信度**: 高

**现状（代码事实）**
```cpp
// DelegateHandler.cpp:54-65 —— Bind 直接覆盖，不处理旧值
void UDelegateHandler::Bind(UObject* InObject, FMethodReflection* InMethod)
{
	if (ScriptDelegate != nullptr)
	{
		if (!ScriptDelegate->IsBound())
		{
			ScriptDelegate->BindUFunction(this, *FUNCTION_CSHARP_CALLBACK);
		}
	}

	DelegateWrapper = {InObject, InMethod};   // ← 覆盖，旧的 Object/Method 静默丢弃
}
```
```cpp
// DelegateHandler.cpp:72-86 —— UnBind/Clear 只动 ScriptDelegate，不碰 DelegateWrapper
void UDelegateHandler::UnBind() const
{
	if (ScriptDelegate != nullptr) { ScriptDelegate->Unbind(); }
}
void UDelegateHandler::Clear() const
{
	if (ScriptDelegate != nullptr) { ScriptDelegate->Clear(); }
}
```
```cpp
// DelegateHandler.cpp:32-52 —— Deinitialize 也不清 DelegateWrapper
void UDelegateHandler::Deinitialize()
{
	if (ScriptDelegate != nullptr)
	{
		if (bNeedFree) { ScriptDelegate->Unbind(); delete ScriptDelegate; }
		ScriptDelegate = nullptr;
	}
	if (DelegateDescriptor != nullptr) { delete DelegateDescriptor; DelegateDescriptor = nullptr; }
	// ← DelegateWrapper 保持原值
}
```

**调用上下文**
`UnBind`/`Clear` 的 C# 入口是 `FRegisterDelegate.cpp:67`（`UnBind`）与 `:76`（`Clear`），二者都在 C++ 侧按 `IManagedHandle` 取 helper 后转发（`FDelegateHelper.cpp:61`/`:69`）。
`Deinitialize` 由 `~FDelegateHelper`（`FDelegateHelper.cpp:17`）触发，后者由 `FDelegateRegistry::RemoveReference` 的 `delete *FoundValue`（`FDelegateRegistry.inl:71`）触发。

**问题**
**[引擎核对]** "`UnBind` 后 UE 侧又触发回调 → 用旧 `Method` 执行"**不成立**：`FScriptDelegate::IsBound()`（`ScriptDelegates.h:159-170,193-197`）要求 `Object.Get() != nullptr`，而 `Unbind()` 会清空对象与函数名，`ProcessDelegate()`（`:432-459`）还会 `checkf(Object.IsValid())` → **UnBind 后 UE 不会进入 `ProcessEvent`**。真正可达的危害是下面两条：
- **桥永不接管（静默功能失效）**：`Bind()` 只在 `ScriptDelegate != nullptr && !ScriptDelegate->IsBound()` 时才 `BindUFunction(this, CSharpCallBack)`（`DelegateHandler.cpp:56-62`）。若该委托属性**已被外部绑定**（蓝图 `Create Event`、序列化恢复的绑定），桥不会接管，而 `DelegateWrapper` 仍被写入（`:64`）→ C# 回调**永远不会被调用且无任何报错**；同时 `IsBound()`（`:67-70`）会返回 `true`，给 C# 侧造成"已绑定"的假象。
- **`bNeedFree == false` 时 `Deinitialize` 不解绑**（`:34-44`）：外部 `FScriptDelegate`（UObject 属性内联存储，见 `FDelegatePropertyDescriptor.cpp:39`）仍指向即将被 `RemoveFromRoot()` 的桥对象；桥对象被 GC 后该属性成为"弱引用已失效"的死绑定（因引擎的 `IsBound()` 语义而静默解除，不崩溃）。
- `Bind()` 覆盖 `DelegateWrapper` 而不清理旧值：重新绑定是静默的，在 C# 侧表现为旧委托对象**不再被回调但也没有报错**（与第一条同源）。
- 影响面确认：`DelegateWrapper` 在整个插件 `Source/` 中只出现在 `DelegateHandler.h:158`、`.cpp:10`、`.cpp:64`（写入）——**没有任何读取后清理的代码路径**。grep 证据：模式 `\bDelegateWrapper\b` → **3 处命中**（不含多播的 `DelegateWrappers`）。

**建议**
把 `DelegateWrapper` 的清理与 `ScriptDelegate` 的绑定状态严格同步：
```cpp
void UDelegateHandler::UnBind()
{
	if (ScriptDelegate != nullptr) { ScriptDelegate->Unbind(); }
	DelegateWrapper = {};                       // 一并失效
}
void UDelegateHandler::Clear()
{
	if (ScriptDelegate != nullptr) { ScriptDelegate->Clear(); }
	DelegateWrapper = {};
}
void UDelegateHandler::Deinitialize()
{
	...
	DelegateWrapper = {};                       // 防悬垂
}
void UDelegateHandler::Bind(UObject* InObject, FMethodReflection* InMethod)
{
	// 断言/记录"重复绑定"，避免静默覆盖
	if (DelegateWrapper.Method != nullptr) { /* ensureMsgf 或先 UnBind */ }
	...
}
```

**验证方式**
`grep -n "\bDelegateWrapper\b" Source/UnrealCSharp`（复跑：**3 处** = `DelegateHandler.h:158`、`.cpp:10`、`.cpp:64`；不用词边界会同时命中多播的 `DelegateWrappers`，共 18 行）。
用例：`Bind(A.Foo)` → `UnBind()` → 再次由 UE 触发该委托（例如蓝图里手动 `Execute` 或在 C# 里 `Execute0`）→ 观察是否仍调用 `A.Foo`（预期：不会，`IsBound()` 已为 false）；以及"先用蓝图 `Create Event` 绑定该属性 → 再由 C# `Bind(B.Bar)` → 触发委托"→ 观察 C# 回调是否被静默吞掉。

---

#### [F-DEL-005] `AddToRoot()` 让桥 UObject 永久驻留，释放却取决于 C# 侧是否析构

- **类别**: 内存/资源泄漏（非确定性驻留）
- **严重度**: **P2**（隐患/资源驻留）
- **复核结论**: 部分确认 —— "每个 helper 都 `AddToRoot()` 一个桥 UObject，且只在 `RemoveFromRoot()` 时解根"是**代码事实**；但"释放**完全依赖 C# 侧是否析构**（= 可能永久泄漏）"经生成器与 C# 侧核对后**不成立**：每个委托包装对象的终结器是**生成器无条件生成**的，构成一条必然存在的释放路径
- **可达性**: 活跃（驻留窗口真实存在）；但"永久泄漏"需要 C# 对象一直可达，此时 helper 本就是被使用的，不构成泄漏
- **复核证据**: ①驻留点：`Source/UnrealCSharp/Private/Reflection/Delegate/FDelegateHelper.cpp:22-24`（`NewObject<UDelegateHandler>()` + `AddToRoot()`），解根点 `:32-42`（`RemoveFromRoot()`），唯一触发者是 `~FDelegateHelper`（`:15-18`）← `FDelegateRegistry.inl:71 delete *FoundValue` ← `FCSharpEnvironment.inl:213-219 RemoveDelegateReference` ← `FRegisterDelegate.cpp:21-28 UnRegisterImplementation`。②**必然释放路径**：`Source/ScriptCodeGenerator/Private/FDelegateGenerator.cpp:79` 为**每个**生成的委托类无条件输出 `~X() => FDelegateImplementation.FDelegate_UnRegisterImplementation(HandleData.GetHandle(this));`（多播版 `:451`）；已生成产物实证 `Script/UE/Proxy/Engine/FTimerDynamicDelegate.cs:18`；C# 实现 `Script/UE/Library/FDelegateImplementation.cs:16-21` → `Script/Interop/Handle/HandleData.cs:46-78`。③被 `NewObject` 创建的包装对象（`Script/Interop/Bridge/ObjectBridge.cs:10-20` 用 `RuntimeHelpers.GetUninitializedObject` + `HandleData.Alloc`，**不跑构造函数**）同样会执行终结器，且 `HandleData.GetHandle` 能取到桥侧分配的句柄（`HandleData.cs:31-43,80-93`）→ 弱引用路径（3 参 `AddDelegateReference`，无 `TDelegateReference`）也不会"只建不删"。④域销毁兜底：`Source/UnrealCSharp/Private/Registry/FDelegateRegistry.cpp:18-55`（两个 map 全量 `delete` + `GCHandle_Free`）。
- **级别变动**: **P1→P2**（理由：`AddToRoot`/`RemoveFromRoot` 是配对的，且存在由生成器保证的终结器释放路径（证据见上），实际后果是"锚定的 UObject 要等托管 GC + `AsyncTask` 排空才回收"的非确定性驻留，属隐患而非永久泄漏；对应 `_CONVENTIONS.md` 的 P2=性能/隐患）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Delegate/FDelegateHelper.cpp:24`
- **函数**: `FDelegateHelper::Initialize(FScriptDelegate*, UFunction*)`（多播同构：`FMulticastDelegateHelper.cpp:25`）
- **置信度**: 高

**现状（代码事实）**
```cpp
// FDelegateHelper.cpp:20-30
void FDelegateHelper::Initialize(FScriptDelegate* InDelegate, UFunction* InSignatureFunction)
{
	DelegateHandler = NewObject<UDelegateHandler>();

	DelegateHandler->AddToRoot();          // ← UE 的 GC root，永不被 GC

	DelegateHandler->Initialize(InDelegate,
	                            InSignatureFunction != nullptr ? InSignatureFunction : DelegateHandler->GetCallBack());
}
```
```cpp
// FDelegateHelper.cpp:32-42 —— 唯一的解根点
void FDelegateHelper::Deinitialize()
{
	if (DelegateHandler != nullptr)
	{
		DelegateHandler->Deinitialize();
		DelegateHandler->RemoveFromRoot();
		DelegateHandler = nullptr;
	}
}
```
```cpp
// 唯一的触发者：FDelegateRegistry.inl:57-81
static auto RemoveReference(Class* InRegistry, const IManagedHandle InManagedHandle)
{
	if (const auto FoundValue = (InRegistry->*ManagedHandle2Value).Find(InManagedHandle))
	{
		...
		delete *FoundValue;                  // → ~FDelegateHelper → Deinitialize → RemoveFromRoot
		(InRegistry->*ManagedHandle2Value).Remove(InManagedHandle);
		FDomain::GCHandle_Free(InManagedHandle);
		return true;
	}
	return false;
}
```

**调用上下文**
`FDelegateHelper` 的创建点：`FDelegatePropertyDescriptor.cpp:39`（`NewRef`）、`:56`（`NewWeakRef`）、`FMulticastDelegatePropertyDescriptor.cpp:50`/`:68`、`FCSharpBind.inl:129`、以及通过 `FCSharpBind::Bind<FDelegateHelper>`（`FRegisterDelegate.cpp:18` / `FRegisterMulticastDelegate.cpp:18`）走 C# 的 `Register`。
释放点**唯一**：`FDelegateRegistry::RemoveReference`，由 `FRegisterDelegate::UnRegisterImplementation`（`FRegisterDelegate.cpp:21-28`）或 `~TDelegateReference`（`TDelegateReference.h:13-16`）驱动。

**问题**
- `AddToRoot()` 把 `UDelegateHandler` 变成**永久 GC root**：只要对应的 `FDelegateHelper` 不被 `delete`，这个 UObject 及其 `FCSharpDelegateDescriptor`（含完整参数缓冲分配器）就永远活着。
- 释放路径由 C# 侧最终化触发：C# 委托对象被 GC → 终结器 → `UnRegister` 或 `~TDelegateReference` → `RemoveDelegateReference`。实测**不会被绕过**：
  1. `FDelegatePropertyDescriptor::NewWeakRef` 走的是 **3 参** `AddDelegateReference`（`:61`），它**不创建 `TDelegateReference`**（`FDelegateRegistry.inl:35-41` 只写一个 map，返回值 `true`）——这一点是事实；但 C# 包装对象的终结器由 **`FDelegateGenerator.cpp:79` 无条件生成**（已生成产物 `Script/UE/Proxy/Engine/FTimerDynamicDelegate.cs:18`），它调用 `FDelegate_UnRegisterImplementation(HandleData.GetHandle(this))`（`FDelegateImplementation.cs:16-21`）→ `FRegisterDelegate.cpp:21-28` → `RemoveDelegateReference`。因此弱引用路径同样有释放路径，**"只建不删"不成立**（`HandleData.cs:31-43,80-93` 保证句柄可被终结器取回，`ObjectBridge.cs:10-20` 创建的未初始化对象也会跑终结器）。
  2. 域重载/热重载的清理**已有实现**：`Source/UnrealCSharp/Private/Registry/FDelegateRegistry.cpp:18-55` 遍历两个 map，逐个 `delete Value` + `FDomain::GCHandle_Free(Key)` + 清空 map → 该场景不构成泄漏。
- 于是真实后果收敛为：`AddToRoot()` 使桥 UObject **在 C# 对象不可达后仍被钉住**，直到（a）托管终结器运行、（b）`AsyncTask(GameThread)` 排空、（c）下一个 GC 周期 —— 这段窗口内 UObject + `FCSharpDelegateDescriptor` + 其参数缓冲分配器都不回收；`RemoveFromRoot()` 本身也不立即销毁，只是解除锚定（`bond` 由 `FDelegateRegistry.inl:71` 的 `delete` 触发，二者不同步）。若 C# 侧把包装对象长期存放在静态容器里，那么 helper 与桥 UObject 是**被使用中的**常驻，不属泄漏。
- `AddToRoot()` 是 UE 中公认的高风险 API（会绕过 GC 的循环引用处理），这里用它来对抗"handler 只被 `TWeakObjectPtr` 持有"（`FDelegateHelper.h:101`）——**根因是持有关系设计成了弱引用**。若改为 `TStrongObjectPtr<UDelegateHandler>` 或 `UPROPERTY()`，就**不需要** `AddToRoot`，也就不会有"忘记 RemoveFromRoot"的风险（RAII 由成员析构保证）。

**建议**
首选：把 `FDelegateHelper.h:101` 的 `TWeakObjectPtr<UDelegateHandler>` 改为 `TStrongObjectPtr<UDelegateHandler>`（`FMulticastDelegateHelper.h:77` 同理），然后**删除** `AddToRoot()`/`RemoveFromRoot()` 两处调用——生命周期由 `FDelegateHelper` 的构造/析构自动管理，不再依赖 C# 侧是否记得 `UnRegister`。
次选（若必须保留弱引用）：在 `NewWeakRef` 路径也创建 `TDelegateReference`（改用 6 参 `AddDelegateReference`），保证有析构兜底。

**验证方式**
`grep -rn "AddToRoot\|RemoveFromRoot" Source/UnrealCSharp`（应只有 `FDelegateHelper.cpp:24,38` 与 `FMulticastDelegateHelper.cpp:25,39`）。
用例：UE 的 `obj gc.StressTest` 或编辑器 `Stat Memory`；在 C# 侧反复创建委托但不注册（只走 `NewWeakRef` 路径），观察 `UDelegateHandler` 实例数是否单调增长（`Obj List Class=UDelegateHandler`）。

---

#### [F-DEL-006] `FOptionalHelper` 释放时只 `Free` 内存、不析构内部值；`TPropertyValue.inl:991` 还是 `new` 分配

- **类别**: 内存/资源泄漏（活值被覆盖/未析构）
- **严重度**: **P1**（泄漏；由 P2 上调）
- **复核结论**: 部分确认 —— 两个子论断分别裁定：**（a）`Deinitialize` 只 `Free` 不析构 → 内部堆泄漏：成立**；**（b）`new` + `FMemory::Free` "分配器不匹配/UB"：证伪（撤销该子论断）**；**（c）权威点：`FOptionalHelper.cpp:91` 的 `Set` 在"已 set（已持活值）"的 `Data` 上再次写入——属已裁决的「`Set` 调 `InitializeValue` 却不 `DestroyValue`」族，权威修复清单明确包含 `FOptionalHelper.cpp:91`，维持 P1**
- **可达性**: 活跃
- **复核证据**: ①`Source/UnrealCSharp/Private/Reflection/Optional/FOptionalHelper.cpp:38-43`：`if (bNeedFreeData && Data != nullptr) { FMemory::Free(Data); Data = nullptr; }` —— 全程无 `MarkUnset`/`DestroyValue`；对照 `:72-75 Reset()` 走 `MarkUnset`。②引擎侧契约：`Engine\Source\Runtime\CoreUObject\Public\UObject\PropertyOptional.h:58-74` `MarkUnset()` 在已 set 时执行 `ValueProperty->DestroyValue(Data)`（**这是唯一会析构内部值的入口**）；`:35-57` `MarkSetAndGetInitializedValuePointerToReplace()` 在非 intrusive 布局下**仅当 `!*IsSetPointer` 时**才 `InitializeValue`，返回 `Data` 本身；`:113-122` `GetValuePointerForReadOrReplace()` 直接 `return Data`。③"活值 Dest"机制：`UnrealType.h:1434` 的 `InitializeValue` 语义注释 = "**this assumes over uninitialized memory**"，而 `FOptionalHelper.cpp:89` 的 `MarkSetAndGetInitializedValuePointerToReplace(Data)` 在**已 set** 的 `Data` 上返回同一地址、**不做 `DestroyValue`**（`PropertyOptional.h:47-55` 的 else 分支），随后 `:91 ValuePropertyDescriptor->Set(InValue, Data)` 走属性描述符的 `InitializeValue` 覆盖 → 旧值（`FString`/`TArray` 等堆对象）失去唯一引用而泄漏。④子论断（b）**证伪**：UBT 会在 `PerModuleInline.gen.cpp` 中注入 `REPLACEMENT_OPERATOR_NEW_AND_DELETE`，模块内 `operator new` 最终路由到 `FMemory::Malloc`，故 `new std::decay_t<T>` 与 `FMemory::Free` 的**分配器是匹配的**，"内存损坏/未定义行为"不成立；该处真缺陷只是"`FMemory::Free` 跳过 `~T()`"（与子论断 (a) 同族的泄漏）。
- **级别变动**: **P2→P1**（理由：泄漏族且含一个已裁决的权威修复点 `FOptionalHelper.cpp:91`；`T = FString/TArray/含堆 USTRUCT` 时每次覆盖 `Set` 与每次析构都泄漏一块堆内存，属 P1 泄漏而非 P2 隐患）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Optional/FOptionalHelper.cpp:38`（同类权威点：`:91`）
- **函数**: `FOptionalHelper::Deinitialize()`
- **置信度**: 高（"不析构"与"活值 Dest"两部分均有引擎源码依据）；"分配器不匹配"部分已**证伪撤销**，不再影响置信度

**现状（代码事实）**
```cpp
// FOptionalHelper.cpp:36-55
void FOptionalHelper::Deinitialize()
{
	if (bNeedFreeData && Data != nullptr)
	{
		FMemory::Free(Data);        // ← 只释放原始内存；不调用 MarkUnset / DestroyValue / ~T

		Data = nullptr;
	}

	if (bNeedFreeProperty && ValuePropertyDescriptor != nullptr && OptionalProperty != nullptr)
	{
		delete ValuePropertyDescriptor;
		ValuePropertyDescriptor = nullptr;
		delete OptionalProperty;
		OptionalProperty = nullptr;
	}
}
```
对比**同一文件**中正确处理析构的写法（`MarkUnset` 会析构内部值）：
```cpp
// FOptionalHelper.cpp:72-75
void FOptionalHelper::Reset() const
{
	OptionalProperty->MarkUnset(Data);      // ← 正确：析构内部值
}
```
以及一个 `bNeedFreeData=true` 且 `Data` 来自 `new` 的构造点：
```cpp
// TPropertyValue.inl:989-996
else
{
	const auto OptionalHelper = new FOptionalHelper(OptionalProperty, new std::decay_t<T>(*InMember),
	                                                true, true);   // ← new 分配
	FCSharpEnvironment::GetEnvironment().AddOptionalReference<FOptionalHelper, false>(
		InMember, OptionalHelper, SrcManagedHandle);
}
```

**调用上下文**
`~FOptionalHelper`（`:27-30`）→ `Deinitialize()`。触发者：
- `FOptionalRegistry::RemoveReference` `:69 delete *FoundValue`（由 C# `TOptional` 终结器 → `FRegisterOptional.cpp:87-93` → `RemoveOptionalReference` 驱动）；
- `FOptionalRegistry::Deinitialize` `:26 delete Value`；
- `FOptionalPropertyDescriptor` / `TPropertyValue` 创建的 helper 均由 registry 持有（`AddOptionalReference`）。
`MarkUnset` 在全插件中**只有 1 处命中**（`FOptionalHelper.cpp:74` 的定义本身）——即**除了 C# 主动 `Reset()` 之外，没有任何地方把 optional 置为 unset**。

**问题**
1. **内部堆泄漏（确定）**：`bNeedFreeData=true` 的 helper 持有的 `Data` 是一个"可能已 set 的 `TOptional<T>`"。`FMemory::Free` 不运行任何析构，因此当 `T = FString` / `FAnsiString` / `FUtf8String` / `TArray<...>` / `TMap<...>` / 含堆的 `USTRUCT` / `FText` 时，**`T` 自身持有的堆内存全部泄漏**。修复只需在 `Free` 之前调用 `OptionalProperty->MarkUnset(Data)`（若已 unset 则是幂等的）。注意 `Data` 由 `FMemory::Malloc` + `InitializeValueInternal` 建立（`:21-23`），或由 `CopyValue` 建立（`TCompoundPropertyDescriptor.inl:35` + `InitializeValue` + `CopySingleValue`），二者都只做**构造**不做**析构**，所以结束时必须显式析构。
2. **分配器不匹配：证伪（撤销）**。`TPropertyValue.inl:991` 的 `new std::decay_t<T>(*InMember)` 配 `FMemory::Free(Data)` 曾被判为"未定义行为"，但 UBT 会在每个模块注入 `REPLACEMENT_OPERATOR_NEW_AND_DELETE`（`Engine/Source/Programs/UnrealBuildTool/.../PerModuleInline.gen.cpp` → `HAL/PerInlineAllocator.h` / `HAL/PerModuleInline.inl`），模块内全局 `operator new` 被替换为 `FMemory::Malloc` 族，因此 **分配器是匹配的**，"内存损坏"不成立。该行的**真缺陷只有一条**：`FMemory::Free` **不调用 `~std::decay_t<T>()`**（等价于第 1 条泄漏，被跳过的是整个 `TOptional<T>` 的析构）。
3. **[复核新增] `Set` 覆盖活值（权威修复点）**：`FOptionalHelper::Set`（`:87-92`）在 `Data` **已经 set** 时，`MarkSetAndGetInitializedValuePointerToReplace` 直接返回原地址且不析构旧值（`PropertyOptional.h:47-55`），随后 `:91 ValuePropertyDescriptor->Set(InValue, Data)` 在活值上做 `InitializeValue`（`UnrealType.h:1434`："assumes over uninitialized memory"）→ 旧 `T` 值泄漏。修法：`Set` 开头的 `MarkSet...` 之后、写入之前，若原本已 set 则先 `OptionalProperty->MarkUnset(Data)`（或改用 `GetValuePointerForReplaceIfSet` 语义显式 `DestroyValue` 再 `InitializeValue`）。
3. `FOptionalHelper::Deinitialize()` 只由析构函数调用，且**没有幂等保护**（`bNeedFreeData` 不会被清 false，但 `Data` 被置 nullptr，所以第二次调用是安全的）——这一点没问题。

**建议**
```cpp
void FOptionalHelper::Deinitialize()
{
	if (bNeedFreeData && Data != nullptr)
	{
		if (OptionalProperty != nullptr)
		{
			OptionalProperty->MarkUnset(Data);      // 析构内部值（对 FString/TArray 必需）
		}

		FMemory::Free(Data);

		Data = nullptr;
	}
	...
}
```
并把 `TPropertyValue.inl:991` 的 `new std::decay_t<T>(*InMember)` 改为走与 `CopyValue` 一致的路径（例如由 `OptionalProperty` 分配并构造），使**构造与析构**成对——注意分配器本身已经匹配（UBT 注入 `REPLACEMENT_OPERATOR_NEW_AND_DELETE`），缺的只是 `~T()`。

**验证方式**
`grep -rn "MarkUnset" Source/UnrealCSharp`（当前仅 1 处，定义本身）。
用例：C# 中 `var o = new TOptional<string>("a very long string ...");` → 不保留引用 → 强制 GC → 用 `TOptional<FString>` 配合 `MALLOC_LEAKDETECTION=1`（UE 的 `MallocLeakDetection`）观察 FString 数据块是否泄漏。`grep -rn "bNeedFreeData" Source/UnrealCSharp` 可列出全部 7 个创建点。

---

#### [F-DEL-007] `NewWeakRef` 每次都 `new FDelegateHelper` 且用 3 参 `AddDelegateReference`（不写地址映射）→ 重复分配累积

- **类别**: 资源放大 / 可优化（重复分配）
- **严重度**: **P2**（维持）
- **复核结论**: 部分确认 —— "`NewWeakRef` 无条件 `new` helper + `Class->NewObject()`、且 3 参 `AddDelegateReference` 不写地址映射 → 无法复用"是**代码事实**；但是第 3 点"弱引用路径失去自动释放兜底 → helper 与 rooted UObject **双双永久泄漏**"**证伪（撤销该子论断）**
- **可达性**: 活跃
- **复核证据**: ①不复用：`Source/UnrealCSharp/Private/Reflection/Property/DelegateProperty/FDelegatePropertyDescriptor.cpp:54-63`（无 `GetDelegateObject(InAddress)` 复用检查）对照 `:33-52`（`NewRef` 有 `:35`/`:37` 复用检查）；`FMulticastDelegatePropertyDescriptor.cpp:66-77` 同构。②地址映射缺失：`Source/UnrealCSharp/Public/Registry/FDelegateRegistry.inl:35-41`（3 参重载只写 `ManagedHandle2Value`）对照 `:43-55`（6 参写两个 map + `new TDelegateReference`）；查询侧 `:27-33`。③**第 3 点证伪**：3 参重载确实不建 `TDelegateReference`，但 C# 包装对象的终结器由生成器**无条件生成**（`ScriptCodeGenerator/Private/FDelegateGenerator.cpp:79`；产物实证 `Script/UE/Proxy/Engine/FTimerDynamicDelegate.cs:18`）→ `FDelegateImplementation.cs:16-21` → `FRegisterDelegate.cpp:21-28` → `FDelegateRegistry.inl:57-81 delete *FoundValue`，故 helper 仍会被释放；`NewObject` 走 `ObjectBridge.cs:10-20` 的 `RuntimeHelpers.GetUninitializedObject` + `HandleData.Alloc`，终结器仍能取回句柄（`HandleData.cs:80-93`）。④因此真实缺陷是**放大**：同一属性每被读一次就多一个 helper + 一个 `AddToRoot` 桥 UObject + 一个 C# 对象，全部要等 GC 才回收（对每帧读委托属性的代码即为持续放大）。
- **级别变动**: 无（P2 维持；理由：存在释放路径，故不是永久泄漏（P1 不成立）；但重复分配 + `AddToRoot` 锚定是实打实的资源放大，P3 不足以描述）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Property/DelegateProperty/FDelegatePropertyDescriptor.cpp:56`
- **函数**: `FDelegatePropertyDescriptor::NewWeakRef(void*)`（多播同构：`FMulticastDelegatePropertyDescriptor.cpp:66-77`）
- **置信度**: 中高（创建/注册事实为高；累积量取决于 C# 侧访问模式）

**现状（代码事实）**
```cpp
// FDelegatePropertyDescriptor.cpp:54-63 —— 弱引用路径：无条件 new，无"已存在则复用"检查
IManagedHandle FDelegatePropertyDescriptor::NewWeakRef(void* InAddress) const
{
	const auto DelegateHelper = new FDelegateHelper(Property->GetPropertyValuePtr(InAddress),
	                                                Property->SignatureFunction);

	const auto Object = Class->NewObject();

	FCSharpEnvironment::GetEnvironment().AddDelegateReference(DelegateHelper, Class, Object);   // ← 3 参重载

	return Object;
}
```
对照**强引用路径**（有"已存在则复用"检查）：
```cpp
// FDelegatePropertyDescriptor.cpp:33-52
IManagedHandle FDelegatePropertyDescriptor::NewRef(void* InAddress) const
{
	auto Object = FCSharpEnvironment::GetEnvironment().GetDelegateObject<FDelegateHelper>(InAddress);

	if (!IManagedHandleIsValid(Object))       // ← 复用检查
	{
		const auto DelegateHelper = new FDelegateHelper(...);
		Object = Class->NewObject();
		const auto OwnerManagedHandle = FCSharpEnvironment::GetEnvironment().GeManagedHandle(InAddress, Property);
		FCSharpEnvironment::GetEnvironment().AddDelegateReference(OwnerManagedHandle, InAddress,
		                                                          DelegateHelper, Class, Object);   // ← 6 参重载
	}
	return Object;
}
```
而两个重载的差别是致命的：
```cpp
// FDelegateRegistry.inl:35-41 —— 3 参：只写 ManagedHandle2Value
	static auto AddReference(Class* InRegistry, typename FDelegateValueMapping::ValueType InValue,
	                         const FClassReflection* InClass, const IManagedHandle InManagedHandle)
	{
		(InRegistry->*ManagedHandle2Value).Add(InManagedHandle, InValue);
		return true;
	}
// FDelegateRegistry.inl:43-55 —— 6 参：同时写 Address2ManagedHandle 并创建 TDelegateReference
	static auto AddReference(Class* InRegistry, const IManagedHandle InOwner, ... InAddress, ...)
	{
		(InRegistry->*Address2ManagedHandle).Add(InAddress, InManagedHandle);
		(InRegistry->*ManagedHandle2Value).Add(InManagedHandle, InValue);
		return FCSharpEnvironment::GetEnvironment().AddReference(InOwner, new TDelegateReference<...>(InManagedHandle));
	}
```
且 `GetDelegateObject` 正是按地址查：
```cpp
// FDelegateRegistry.inl:27-33
	static auto GetObject(Class* InRegistry, const typename FDelegateValueMapping::FAddressType InAddress) -> IManagedHandle
	{
		const auto FoundManagedHandle = (InRegistry->*Address2ManagedHandle).Find(InAddress);
		return FoundManagedHandle != nullptr ? *FoundManagedHandle : InvalidManagedHandle;
	}
```

**调用上下文**
`Get(Src, Dest, FMember)`（`FDelegatePropertyDescriptor.cpp:5-8`）与 `Get(Src, Dest, FReturn)`（`:10-13`）都调用 `NewWeakRef(Src)`。即**每次把委托属性作为成员值或返回值读给 C#，都会走 `NewWeakRef`**。

**问题**
1. `NewWeakRef` 没有 `GetDelegateObject(InAddress)` 复用检查 → 同一个属性地址被读 N 次就产生 **N 个 `FDelegateHelper` + N 个 `UDelegateHandler`（各自 `AddToRoot`）+ N 个 C# 侧对象**。
2. 3 参 `AddReference` **不写 `Address2ManagedHandle`** → 即使后续调用 `NewRef` 也无法通过 `GetDelegateObject(InAddress)` 发现已存在的 helper（`FDelegateRegistry.inl:27-33` 查的正是这张表），于是**必然**再 `new` 一份。这使缺陷 (1) 从"可能"变成"确定"。
3. 3 参重载也**不创建 `TDelegateReference`**（`FDelegateRegistry.inl:35-41`）→ 失去 `~TDelegateReference` 这一自动释放兜底（见 F-DEL-005），只能依赖 C# 侧显式 `UnRegister`。这正是任务书假设的"绑定过就永不回收"泄漏在**本层的真实形态**——不是通过强引用 C# 委托，而是通过**泄漏 UE 侧的 helper + rooted `UDelegateHandler`**。

**建议**
```cpp
IManagedHandle FDelegatePropertyDescriptor::NewWeakRef(void* InAddress) const
{
	auto Object = FCSharpEnvironment::GetEnvironment().GetDelegateObject<FDelegateHelper>(InAddress);
	if (IManagedHandleIsValid(Object))
	{
		return Object;                         // 复用
	}
	// 并改用 6 参重载（传入 OwnerManagedHandle + InAddress），使地址映射与 TDelegateReference 都建立
	const auto DelegateHelper = new FDelegateHelper(...);
	Object = Class->NewObject();
	const auto OwnerManagedHandle = FCSharpEnvironment::GetEnvironment().GeManagedHandle(InAddress, Property);
	FCSharpEnvironment::GetEnvironment().AddDelegateReference(OwnerManagedHandle, InAddress, DelegateHelper, Class, Object);
	return Object;
}
```
若"弱引用必须与强引用区分所有权"是刻意设计，则至少要把 `Address2ManagedHandle` 写进去（让复用检查生效），并明确文档化两套语义。

**验证方式**
`grep -n "NewWeakRef\|AddDelegateReference" Source/UnrealCSharp/Private/Reflection/Property/DelegateProperty/*.cpp Source/UnrealCSharp/Public/Registry/FDelegateRegistry.inl`。
用例：C# 中循环 1000 次读取同一个 `FDelegate` 属性（例如 `actor.Delegate` 放进 `for`），观察 `UDelegateHandler` 实例数与 `FDelegateRegistry` 的两个 map 大小——预期 1000（当前）vs 1（修复后）。

---

#### [F-DEL-008] `FOptionalRegistry::RemoveReference` 缺少 `FDomain::GCHandle_Free` → C# GC 句柄泄漏

- **类别**: 内存/资源泄漏（GC 句柄）
- **严重度**: **P1**（维持）
- **复核结论**: **确认** —— `FOptionalRegistry::RemoveReference` 缺 `FDomain::GCHandle_Free` 属实，且**同族兄弟注册表提供了对照证据**：字符串/容器注册表的 `RemoveReference` **都做了** 这一步，说明是遗漏而非设计
- **可达性**: 活跃（`TOptional.cs:14` 终结器 → `FRegisterOptional.cpp:87-93` 每次 C# `TOptional<T>` 被回收都会走这条路径）
- **复核证据**: `Source/UnrealCSharp/Private/Registry/FOptionalRegistry.cpp:55-77`（`:69 delete` + `:71` 清句柄 map 后**直接 `return true`**，无 `GCHandle_Free`）对照同插件**同族实现**：`Source/UnrealCSharp/Public/Registry/FDelegateRegistry.inl:75`、`Source/UnrealCSharp/Public/Registry/FStringRegistry.inl:76`、`Source/UnrealCSharp/Public/Registry/FContainerRegistry.inl:76` —— 三者都在 `RemoveReference` 末尾 `FDomain::GCHandle_Free(InManagedHandle)`；而 `FOptionalRegistry::Deinitialize` **有**（`FOptionalRegistry.cpp:31`）→ 同一文件内部自相矛盾（`grep FDomain::GCHandle_Free` 在 `Source/` 共 32 处命中，`FOptionalRegistry.cpp` 只有 `:31` 一处，位于 `Deinitialize`）。调用链 `FCSharpEnvironment.cpp:607 RemoveOptionalReference` → 本函数。
- **级别变动**: 无（P1 维持；理由：每次 C# `TOptional<T>` 终结都泄漏一个强 GC 句柄 → 托管对象被句柄永久钉住，属 P1 泄漏）
- **文件**: `Source/UnrealCSharp/Private/Registry/FOptionalRegistry.cpp:55`
- **函数**: `FOptionalRegistry::RemoveReference(IManagedHandle)`
- **置信度**: 高（不一致性为事实；泄漏量取决于 `GCHandle_Free` 的实现，未读 `FDomain`）

**现状（代码事实）**
```cpp
// FOptionalRegistry.cpp:55-77 —— 没有 GCHandle_Free
bool FOptionalRegistry::RemoveReference(const IManagedHandle InManagedHandle)
{
	if (const auto FoundValue = ManagedHandle2Helper.Find(InManagedHandle))
	{
		const auto Address = (*FoundValue)->GetAddress();

		if (const auto FoundManagedHandle = Address2ManagedHandle.Find(Address))
		{
			if (*FoundManagedHandle == InManagedHandle)
			{
				Address2ManagedHandle.Remove(Address);
			}
		}

		delete *FoundValue;

		ManagedHandle2Helper.Remove(InManagedHandle);

		return true;                      // ← 直接 return，没有 FDomain::GCHandle_Free
	}

	return false;
}
```
对照同一插件中委托注册表的**正确写法**：
```cpp
// FDelegateRegistry.inl:57-81
static auto RemoveReference(Class* InRegistry, const IManagedHandle InManagedHandle)
{
	if (const auto FoundValue = (InRegistry->*ManagedHandle2Value).Find(InManagedHandle))
	{
		...
		delete *FoundValue;
		(InRegistry->*ManagedHandle2Value).Remove(InManagedHandle);

		FDomain::GCHandle_Free(InManagedHandle);      // ← 委托侧有，可选侧没有

		return true;
	}
	return false;
}
```
而 `FOptionalRegistry::Deinitialize()`（`:20-39`）**是**做 `GCHandle_Free` 的：
```cpp
		FDomain::GCHandle_Free(Key);     // FOptionalRegistry.cpp:31
```

**调用上下文**
`FCSharpEnvironment::RemoveOptionalReference`（`Private/Environment/FCSharpEnvironment.cpp:607`）→ `FOptionalRegistry::RemoveReference`。触发者：`FRegisterOptional::UnRegisterImplementation`（`FRegisterOptional.cpp:87-93`，`AsyncTask(GameThread, ...)`），由 C# `TOptional.cs:14` 的**终结器** `~TOptional()` 调用。即**每次一个 C# `TOptional<T>` 被 GC 时都会走这条路径**。

**问题**
`IManagedHandle` 是 C# 对象的 GC 句柄（`FDomain::GCHandle_*`）。`AddOptionalReference` 建立句柄，`RemoveReference` 删除 C++ 侧的 helper 与 map 条目，但**不释放句柄** → 该 C# 对象被句柄**永久强引用**，永远无法被 GC。
后果链条：C# `TOptional<T>` 无法被回收 → 其终结器不会再触发（已经触发过）→ 但对象本身占用的托管内存与它引用的 `T` 值（可能是大对象）**永不释放**。这是典型的"句柄表泄漏"，长时间运行会表现为托管堆单调增长（`GCHandle` 表本身也会增长）。
与 `Deinitialize` 的不一致（那里做了 `GCHandle_Free`）进一步说明这是**遗漏而非设计**。

**建议**
```cpp
bool FOptionalRegistry::RemoveReference(const IManagedHandle InManagedHandle)
{
	if (const auto FoundValue = ManagedHandle2Helper.Find(InManagedHandle))
	{
		const auto Address = (*FoundValue)->GetAddress();
		if (const auto FoundManagedHandle = Address2ManagedHandle.Find(Address))
		{
			if (*FoundManagedHandle == InManagedHandle)
			{
				Address2ManagedHandle.Remove(Address);
			}
		}

		delete *FoundValue;
		ManagedHandle2Helper.Remove(InManagedHandle);

		FDomain::GCHandle_Free(InManagedHandle);      // ← 补上，与 FDelegateRegistry.inl:75 对齐

		return true;
	}
	return false;
}
```
建议同时**审计全部 `T*Registry::RemoveReference`**（字符串/容器/多播/软引用等约 12 个注册表，见 `FRegister*.cpp` 的 `UnRegisterImplementation`）是否存在同一遗漏。

**验证方式**
`grep -rn "GCHandle_Free" Source/UnrealCSharp`（对比每个 `RemoveReference` 实现）；`grep -rn "RemoveReference" Source/UnrealCSharp/Private/Registry/`。
用例：C# 中循环创建并丢弃 100k 个 `TOptional<int>`，强制 GC，观察 `FDomain` 侧句柄表大小与托管堆是否回落。

---

---

### P2 组（初判定级 P2 / 5 条 → 实际：F-DEL-010 = **P2**、F-DEL-013 = **P2**、F-DEL-009 = **P3**、F-DEL-011 = **P3**、F-DEL-012 = **P3**）

#### [F-DEL-009] `FRegisterOptional::Register1Implementation` 未判空 `GetClass()` 结果

- **类别**: Bug / 空指针（防御性缺失）
- **严重度**: **P3**（隐患；维持）
- **复核结论**: 部分确认 —— "未判空 `GetClass()` 结果就解引用"是代码事实；但**触发条件（"类型未注册时 `GetClass` 返回 nullptr"）被证伪**：对**有效**句柄 `GetClass` 会**惰性新建** `FClassReflection`，不会因为"未注册/加载顺序"返回空
- **可达性**: 潜伏（仅在句柄无效（含 `0`）或 `IScriptDomain::Get()` 为空（域销毁/Shutdown 窗口）时才返回 nullptr；C# 常态路径传的 `GetType()` 句柄始终有效）
- **复核证据**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterOptional.cpp:16`（`Class`）+ `:24`（`Class->GetGenericArgument()` 解引用，同文件 `:41`/`:49` 的 `Register2Implementation` 同构）；`Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp:926-950`：`if (IManagedHandleIsValid(InManagedClass)) { ... if (const auto ScriptDomain = IScriptDomain::Get()) { 未命中则 new FClassReflection 并 Add 后返回；命中则返回已有 } } return nullptr;` —— **只有句柄无效或域不存在才返回 nullptr**；对照 `Source/UnrealCSharp/Public/Registry/FDelegateRegistry.inl:19-25`（委托侧"找不到即 nullptr"的约定与这里不同）。
- **级别变动**: 无（P3 维持；理由：触发前提是"无效句柄/域已销毁"，非常态路径，定 P3 隐患合理）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterOptional.cpp:16`
- **函数**: `FRegisterOptional::Register1Implementation(IManagedHandle, IManagedHandle)`
- **置信度**: 中（未验证 `GetClass` 的失败条件，但同文件其余入口都做了判空，形成明显反差）

**现状（代码事实）**
```cpp
// FRegisterOptional.cpp:14-35
static void Register1Implementation(const IManagedHandle InManagedObject, const IManagedHandle InManagedType)
{
	const auto Class = FReflectionRegistry::Get().GetClass(InManagedType);   // ← 未判空

#if UE_F_PROPERTY_CONSTRUCTOR_E_OBJECT_FLAGS
	const auto OptionalProperty = new FOptionalProperty(nullptr, "", EObjectFlags::RF_Transient);
#else
	const auto OptionalProperty = new FOptionalProperty(nullptr, "");
#endif

	const auto ValueProperty = FTypeBridge::Factory(Class->GetGenericArgument(), ...);   // ← 解引用
	...
	const auto OptionalHelper = new FOptionalHelper(OptionalProperty, nullptr, true, true);
	FCSharpEnvironment::GetEnvironment().AddOptionalReference<FOptionalHelper, false>(
		nullptr, OptionalHelper, Class, InManagedObject);
}
```
对照 `FRegisterDelegate::RegisterImplementation`（`FRegisterDelegate.cpp:14-19`）——同样未判空；但 `FRegisterDelegate::BindImplementation`（`:35-48`）与 `FRegisterMulticastDelegate.cpp` 各入口都做了 4 层判空。

**调用上下文**
C# `TOptional.cs:9` 的构造函数 `TOptional() => TOptionalImplementation.TOptional_Register1Implementation(this, GetType())` → `TOptionalImplementation.cs:14` → 本函数。即**每次 `new TOptional<T>()` 都走这里**。

**问题**
**[引擎核对]** "`FReflectionRegistry::GetClass(InManagedType)` 在**类型未注册**时返回 `nullptr`（与 `FDelegateRegistry::GetDelegate` 同样的约定）"**证伪**。实测 `FReflectionRegistry.cpp:926-950`：只要句柄有效且 `IScriptDomain::Get()` 可用，未命中 `FullName2Class` 时会 **`new FClassReflection(...)` 并登记后返回**（惰性建类），只有 `IManagedHandleIsValid(InManagedClass) == false` 或域不存在时才 `return nullptr`。因此"反射注册未完成/跨程序集加载顺序"**不会**触发本缺陷。

仍然成立的部分：`Register1Implementation`（`:14-36`）与 `Register2Implementation`（`:38-72`）都**未**对 `Class` 判空，其后 `:24`/`:49` 直接 `Class->GetGenericArgument()`；一旦拿到 `nullptr`（无效句柄，例如 C# 侧传入 `0`，或域销毁/Shutdown 窗口内 `IScriptDomain::Get()` 为空），即空指针解引用崩溃。此外 `FTypeBridge::Factory(...)` 的返回值同样未判空就被 `:28`/`:53` 的 `ValueProperty->SetPropertyFlags(...)` 解引用，属同一函数内的同族缺失（原报告未列）。
另外 `Register2Implementation`（`:38-72`）有同样问题（`:41`）。

**建议**
```cpp
const auto Class = FReflectionRegistry::Get().GetClass(InManagedType);
if (Class == nullptr)
{
	return;      // 或 ensureMsgf(false, TEXT("TOptional: unregistered type"));
}
```
并同步修复 `FRegisterDelegate.cpp:16`、`FRegisterMulticastDelegate.cpp:16`。

**验证方式**
`grep -n "GetClass(InManagedType)" Source/UnrealCSharp/Private/Domain/Interop/*.cpp` 对比每处是否判空。

---

#### [F-DEL-010] `FOptionalHelper::Get()` 返回可能失效的裸指针，且未 set 时不解箱检查

- **类别**: Bug / 未定义行为
- **严重度**: **P2**（维持）
- **复核结论**: 部分确认 —— "`Get()` 不检查 `IsSet()`"与"未 set 时返回未初始化内存并被立刻解箱"**成立且已由引擎源码确证**；但是第 2 点"返回的指针可能失效"**证伪（撤销）**
- **可达性**: 活跃（`FRegisterOptional.cpp:113-125 GetImplementation` 会无条件调用）
- **复核证据**: ①`Source/UnrealCSharp/Private/Reflection/Optional/FOptionalHelper.cpp:82-85`（无 `IsSet` 判断）；调用方 `Source/UnrealCSharp/Private/Domain/Interop/FRegisterOptional.cpp:117-121`（取到指针立刻 `GetValuePropertyDescriptor()->Get(Value, ...)` 装箱）；C# 入口 `Script/UE/CoreUObject/TOptional.cs:44`。②引擎契约：`Engine\Source\Runtime\CoreUObject\Public\UObject\PropertyOptional.h:113-122` —— `GetValuePointerForReadOrReplace` 的注释明确 "**Must be called on a non-null pointer to a set optional**"，函数体是 `checkSlow(Data && IsSet(Data)); return Data;` → 在**未 set** 的 optional 上调用属**违反前置条件**（Development 构建触发 `checkSlow`，Shipping 放行），并返回"未构造的值存储地址"，被 `Get` 装箱后对 `T=FString/TArray` 即读取垃圾指针。③**第 2 点证伪**：该 API 的实现恒为 `return Data;`（`PropertyOptional.h:78-122`，`GetValuePointerForRead/ForReplace/ForReadOrReplace` 全部返回 `Data` 本身），并不存在"set/unset 改变内部布局导致返回指针失效"的行为；`Data` 在 helper 生命周期内地址恒定（`FOptionalHelper.cpp:15-24`），故"若 C# 侧缓存该指针才悬垂"的担忧不成立——**风险只在生命周期结束后**，属 F-DEL-011 的范畴。
- **级别变动**: 无（P2 维持；理由：需要 C# 侧"未 `IsSet()` 就 `Get()`"，属可避免的误用路径，且 LeanCLR 下多为读到垃圾/托管侧失败，未观察到必然崩溃）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Optional/FOptionalHelper.cpp:82`
- **函数**: `FOptionalHelper::Get() const`
- **置信度**: 中（指针失效为 UE `FOptionalProperty` 语义推断，未读 UE 引擎侧实现）

**现状（代码事实）**
```cpp
// FOptionalHelper.cpp:82-85
void* FOptionalHelper::Get() const
{
	return OptionalProperty->GetValuePointerForReadOrReplace(Data);
}
```
```cpp
// FRegisterOptional.cpp:113-125 —— 立刻把该指针转成句柄
static IManagedHandle GetImplementation(const IManagedHandle InManagedHandle)
{
	IManagedHandle ReturnValue{};
	if (const auto OptionalHelper = FCSharpEnvironment::GetEnvironment().GetOptional(InManagedHandle))
	{
		const auto Value = OptionalHelper->Get();
		OptionalHelper->GetValuePropertyDescriptor()->Get(Value, reinterpret_cast<void**>(&ReturnValue));
	}
	return ReturnValue;
}
```
```csharp
// Script/UE/CoreUObject/TOptional.cs:44
public T Get() => (T)TOptionalImplementation.TOptional_GetImplementation(HandleData.GetHandle(this));
```

**调用上下文**
C# `TOptional<T>.Get()` → `FRegisterOptional.cpp:113` → `FOptionalHelper::Get()`。`Get()` **不检查 `IsSet()`**（对比 `FOptionalHelper::IsSet()` 在 `:77-80` 是独立方法，需要 C# 侧自觉先调用）。

**问题**
两点：
1. **未 set 时返回未初始化内存**：`GetValuePointerForReadOrReplace` 在 optional 无值时返回的是值存储的地址（内容未构造）。`FRegisterOptional.cpp:121` 立刻用 `ValuePropertyDescriptor->Get(Value, ...)` 去**读取/装箱**它 → 把未初始化内存当成有效 `FString`/`TArray` 解箱 → 读取越界指针，**可能直接崩溃**（P0 级触发条件，但需要 C# 侧忘记先 `IsSet()`，故定级 P2）。C# 侧 `TOptional<T>` 提供了 `IsSet()` 但 `Get()` 不强制它。
2. **[复核修正] 指针有效期：撤销该子论断**。`FOptionalPropertyLayout::GetValuePointerForReadOrReplace` 的实现恒为 `checkSlow(Data && IsSet(Data)); return Data;`（`PropertyOptional.h:113-122`），`GetValuePointerForRead`/`ForReplace` 同（`:78-90`）——**不存在**"set/unset 改变内部布局导致返回指针失效"的行为，返回的永远是 `Data` 本身，而 `Data` 在 `FOptionalHelper` 生存期内地址不变。真正相关的风险是 `Data` 随 helper 被销毁（`FOptionalRegistry::RemoveReference` → `delete`）后 C# 侧仍持有解箱结果——那属于 F-DEL-011 讨论的生命周期问题，不是"指针有效期"问题。

**建议**
```cpp
void* FOptionalHelper::Get() const
{
	if (OptionalProperty == nullptr || !OptionalProperty->IsSet(Data))
	{
		return nullptr;                 // 明确表示"无值"
	}
	return OptionalProperty->GetValuePointerForReadOrReplace(Data);
}
```
调用方 `FRegisterOptional.cpp:117-122` 相应判空并返回 `InvalidManagedHandle`，C# 侧 `Get()` 改为返回 `default`/抛异常而不是解箱 nullptr：
```csharp
public T Get()
{
    var handle = TOptionalImplementation.TOptional_GetImplementation(HandleData.GetHandle(this));
    return handle != 0 ? (T)HandleData.GetObject(handle) : default;
}
```

**验证方式**
用例：C# 中 `var o = new TOptional<FString>(); var s = o.Get();`（未 set 直接取）→ 期望当前实现会读出垃圾数据或崩溃。
`grep -n "GetValuePointerForReadOrReplace" Source/UnrealCSharp`（1 处命中，仅 `FOptionalHelper.cpp:84`）。

---

#### [F-DEL-011] `FOptionalHelper` 释放后各访问方法不判空，与 `Deinitialize` 的置空策略矛盾

- **类别**: Bug（防御性缺失 / 释放后使用）
- **严重度**: **P3**（维持）
- **复核结论**: 部分确认 —— "`Deinitialize` 把成员置空，但所有访问方法都不判空"是**代码事实**（复核 grep：`OptionalProperty->` 在本文件出现 5 次（`:69,74,79,84,89`）+ `InOptionalProperty->` 2 次（`:13,23`）= 7 处解引用，0 处判空，唯一判空在 `:45` 的 `Deinitialize` 中）；但"可达的释放后使用"论证**不成立**：`Deinitialize` 只由 `~FOptionalHelper`（`:27-30`）调用，`delete` 之后该 helper 已从注册表移除（`FOptionalRegistry.cpp:69-71`），后续 `GetOptional` 返回 `nullptr` 并被调用方判空挡住（`FRegisterOptional.cpp:97/105/117/129`）
- **可达性**: 潜伏（未找到可达的"成员已置空后仍被调用"路径；仅在 `Deinitialize` 被二次调用、或外部持有 helper 裸指针跨 `delete` 使用时才会命中）
- **复核证据**: `Source/UnrealCSharp/Private/Reflection/Optional/FOptionalHelper.cpp:45-54`（置空）对照 `:72-92`（`Reset`/`IsSet`/`Get`/`Set` 全部直接解引用 `OptionalProperty`/`ValuePropertyDescriptor`）、`:57-70`（`Identical` 只判 `InA/InB` 是否为 null，不判其成员）；删除点 `Source/UnrealCSharp/Private/Registry/FOptionalRegistry.cpp:69`（`delete *FoundValue`）与 `:71`（从 map 移除，同一函数内先 delete 后移除）；调用方判空证据 `Source/UnrealCSharp/Private/Domain/Interop/FRegisterOptional.cpp:97,105,117,129`。
- **级别变动**: 无（P3 维持；理由：属防御性/一致性缺陷，未找到可达的释放后使用路径）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Optional/FOptionalHelper.cpp:72`
- **函数**: `FOptionalHelper::Reset` / `IsSet` / `Get` / `Set` / `Identical`
- **置信度**: 中高

**现状（代码事实）**
```cpp
// FOptionalHelper.cpp:36-55 —— Deinitialize 把两个指针置 nullptr
	if (bNeedFreeProperty && ValuePropertyDescriptor != nullptr && OptionalProperty != nullptr)
	{
		delete ValuePropertyDescriptor;
		ValuePropertyDescriptor = nullptr;
		delete OptionalProperty;
		OptionalProperty = nullptr;
	}
```
```cpp
// FOptionalHelper.cpp:72-92 —— 但所有访问方法都不判空
void FOptionalHelper::Reset() const      { OptionalProperty->MarkUnset(Data); }
bool FOptionalHelper::IsSet() const      { return OptionalProperty->IsSet(Data); }
void* FOptionalHelper::Get() const       { return OptionalProperty->GetValuePointerForReadOrReplace(Data); }
void FOptionalHelper::Set(void* InValue) const
{
	OptionalProperty->MarkSetAndGetInitializedValuePointerToReplace(Data);
	ValuePropertyDescriptor->Set(InValue, Data);
}
```
```cpp
// FOptionalHelper.cpp:57-70 —— Identical 也只判了 helper 本身，没判其成员
bool FOptionalHelper::Identical(const FOptionalHelper* InA, const FOptionalHelper* InB)
{
	if (InA == nullptr || InB == nullptr) { return false; }
	if (!InA->GetValuePropertyDescriptor()->SameType(InB->GetValuePropertyDescriptor())) { return false; }
	return InA->OptionalProperty->Identical(InA->Data, InB->Data, 0);
}
```

**调用上下文**
`Deinitialize()` 由 `~FOptionalHelper`（`:27-30`）调用，触发者：`FOptionalRegistry::RemoveReference` `:69`、`FOptionalRegistry::Deinitialize` `:26`。
调用方全部走"按句柄取 helper，取到就用"的模式且**只判 helper 指针**：`FRegisterOptional.cpp:97-99`（`Reset`）、`:105-107`（`IsSet`）、`:117-121`（`Get`）、`:129-140`（`Set`）、`:76-80`（`Identical`）。

**问题**
`FOptionalRegistry::RemoveReference` 会 `delete` helper 并从 map 移除句柄（`FOptionalRegistry.cpp:69-71`），因此正常路径下后续 `GetOptional(handle)` 返回 `nullptr`，调用方的判空能挡住。**但**：C# `TOptional<T>` 是**终结器驱动**的（`TOptional.cs:14`），终结器与用户代码的时序不确定；在对象已经进入终结队列但 `UnRegister` 尚未执行（或已被 `AsyncTask` 排队但未运行）的窗口内，C# 侧仍可能通过 `HandleData.GetHandle(this)` 调用 `Get/Set/Reset` → 若恰好 `delete` 已完成，则 `GetOptional` 返回 nullptr，安全；真正的风险是**同一个 `FOptionalHelper` 被两条路径共同持有**（`FOptionalPropertyDescriptor::Get` 创建的 helper 由 `AddOptionalReference` 注册，而 `TPropertyValue` 路径又可能对同一地址再建一个），一条删除另一条仍在用。
无论窗口多窄，**"置空指针后不判空"本身就是防御性缺陷**：`Deinitialize` 显式置空（说明作者意识到重入），却没有任何方法来消费这个"已失效"状态。建议 `Deinitialize` 后所有方法早退，语义上等价于"空 optional"。

**建议**
```cpp
void FOptionalHelper::Reset() const   { if (OptionalProperty) OptionalProperty->MarkUnset(Data); }
bool FOptionalHelper::IsSet() const   { return OptionalProperty != nullptr && OptionalProperty->IsSet(Data); }
void* FOptionalHelper::Get() const    { return OptionalProperty != nullptr ? OptionalProperty->GetValuePointerForReadOrReplace(Data) : nullptr; }
void FOptionalHelper::Set(void* InValue) const
{
	if (OptionalProperty == nullptr || ValuePropertyDescriptor == nullptr) { return; }
	...
}
```
并在 `~FOptionalHelper`（或 `Deinitialize`）末尾加 `bNeedFreeData = bNeedFreeProperty = false;` 使其幂等语义显式化。

**验证方式**
`grep -n "OptionalProperty->" Source/UnrealCSharp/Private/Reflection/Optional/FOptionalHelper.cpp`（7 处解引用，0 处判空；唯一判空在 `:45` 的 `Deinitialize`）。

---

#### [F-DEL-012] 桥接层完全无 GameThread 断言；`~TDelegateReference` 与 Interop `UnRegister` 的线程假设不一致

- **类别**: 并发/线程安全
- **严重度**: **P3**（维持）
- **复核结论**: 部分确认 —— "两条释放路径的线程假设不一致"与"本组文件零线程断言"是**代码事实**（`grep IsInGameThread` 在 `Source/` 命中 **3** 处，均不在本组，见复核证据；本组 `Private/Reflection/Delegate`、`Private/Reflection/Optional` 下 `grep IsInGameThread|check(` = 0 命中）；但"实际竞态"仍取决于托管后端如何调度终结器线程，**未能实机验证**
- **可达性**: 活跃（同一对象的两条释放路径都在正常使用中出现：C# 终结器走 `~TDelegateReference`（仅 6 参注册路径），C# 显式 `UnRegister` 走 `AsyncTask`）
- **复核证据**: ①`Source/UnrealCSharp/Public/Reference/TDelegateReference.h:13-16`（析构函数**直接**同步调用 `RemoveDelegateReference`，无 `AsyncTask`）对照 `Source/UnrealCSharp/Private/Domain/Interop/FRegisterDelegate.cpp:21-28`（`AsyncTask(ENamedThreads::GameThread, ...)`）、`FRegisterMulticastDelegate.cpp:21-28`、`FRegisterOptional.cpp:87-93`。②被调 `FDelegateRegistry.inl:57-81`/`FOptionalRegistry.cpp:55-77` 会 `delete` 对象并改裸 `TMap`（`FDelegateRegistry.h:43-49`），无锁。③**计数修正**：`grep IsInGameThread` 在 `Source/` 共 **3** 处 —— `Private/Registry/FDynamicRegistry.cpp:23`、`Private/Registry/FClassRegistry.cpp:241`、**`Private/Environment/FCSharpEnvironment.cpp:289`**（`NotifyUObjectCreated` 内，非 GameThread 时改走 `FScopeLock` + `AsyncLoadingObjectArray`）→ 第三处恰好说明模块作者**知道**要区分线程。④`TDelegateReference` 由 `FDelegateRegistry.inl:52-54` 创建，只挂 6 参路径。
- **级别变动**: 无（P3 维持；理由：不一致性是事实，但未观察到实际竞态，且本组代码均在 GameThread 主路径上被调用）
- **文件**: `Source/UnrealCSharp/Public/Reference/TDelegateReference.h:13`
- **函数**: `TDelegateReference::~TDelegateReference()`（及其调用的 `FDelegateRegistry::RemoveReference`）
- **置信度**: 中高（不一致性为事实；实际竞态取决于 C# GC 线程模型）

**现状（代码事实）**
```cpp
// TDelegateReference.h:6-17 —— 直接调用，不切线程
template <typename T>
class TDelegateReference final : public FReference
{
public:
	using FReference::FReference;
public:
	virtual ~TDelegateReference() override
	{
		(void)FCSharpEnvironment::GetEnvironment().RemoveDelegateReference<T>(ManagedHandle);
	}
};
```
对照 Interop 路径（**显式切回 GameThread**）：
```cpp
// FRegisterDelegate.cpp:21-28
static void UnRegisterImplementation(const IManagedHandle InManagedHandle)
{
	AsyncTask(ENamedThreads::GameThread, [InManagedHandle]
	{
		(void)FCSharpEnvironment::GetEnvironment().RemoveDelegateReference<FDelegateHelper>(InManagedHandle);
	});
}
// FRegisterOptional.cpp:87-93 同样有 AsyncTask(GameThread, ...)
```
而被调用的 `RemoveReference` 会**修改裸 `TMap` 并 `delete` 对象**（`FDelegateRegistry.inl:57-81`），`FOptionalRegistry::RemoveReference` 同样（`FOptionalRegistry.cpp:55-77`）。

**调用上下文**
`TDelegateReference` 由 `FDelegateRegistry.inl:52-54` 创建，挂在 C# 对象引用上；其析构由 C# 侧引用释放驱动（可能发生在**终结器线程**或 Mono/CoreCLR 的后台 GC 线程）。`FDelegateRegistry` 的两个 `TMap`（`FDelegateRegistry.h:43-49`）与 `FOptionalRegistry` 的两个 map 都**没有任何锁**。

**问题**
1. **无锁的 map 并发修改**：`UnRegister` 路径（C# 显式调用）被 `AsyncTask` 保护；但 `~TDelegateReference` 路径**没有**。若二者并发，或 `~TDelegateReference` 在非 GameThread 上执行，则与 GameThread 上的 `Add`/`Get`（`FRegisterDelegate.cpp:35` 等每帧可能被调用）构成数据竞争 → `TMap` 内部 rehash 期间被另一线程读 → 崩溃或内存损坏。
2. **本组文件内零线程断言**：`grep -n "IsInGameThread\|check(" Source/UnrealCSharp/Private/Reflection/Delegate Source/UnrealCSharp/Private/Reflection/Optional` → **0 命中**（仅用 `Read` 工具逐文件确认）。`[复核修正]` 整个 `Source/` 中 `IsInGameThread()` 共 **3** 处：`Private/Registry/FDynamicRegistry.cpp:23`、`Private/Registry/FClassRegistry.cpp:241`、**`Private/Environment/FCSharpEnvironment.cpp:289`**（`NotifyUObjectCreated`：非 GameThread 时用 `FScopeLock Lock(&CriticalSection)` + `AsyncLoadingObjectArray` 暂存），三处均不在本组。
3. **UE 委托可能从别的线程广播**：UE 的多播委托（如 `OnAsyncThingDone`）可在工作线程 `Broadcast`，进而调用 `UMulticastDelegateHandler::ProcessEvent` → `DelegateWrappers` 遍历（`MulticastDelegateHandler.cpp:11`）。此时若 GameThread 同时在 `Add`/`Remove`（`:89`/`:111`）修改同一数组 → 与 F-DEL-002 叠加成**跨线程**的数组损坏，而不只是迭代器失效。

**建议**
- 在 `TDelegateReference::~TDelegateReference` 中与 Interop 对齐，改为投递到 GameThread；若析构可能发生在 Shutdown 之后，需要"环境已析构则直接返回"的保护（参考 `FCSharpEnvironment` 的单例空判定）。
- 在两个注册表的 `Add/Get/Remove` 上加 `check(IsInGameThread())`（Debug 下尽早暴露），或给 map 加 `FCriticalSection`（参考同插件的 `FCSharpCompilerRunnable.h:64` 用法）。
- 在 `ProcessEvent` 入口加 `check(IsInGameThread())`，并在文档中明确"桥接层假定 GameThread"；若需支持跨线程广播，则 `DelegateWrappers` 必须加锁或改快照。

**验证方式**
`grep -rn "IsInGameThread" Source/`（复跑：**3 处**，均不在本组 —— `FDynamicRegistry.cpp:23`、`FClassRegistry.cpp:241`、`FCSharpEnvironment.cpp:289`）；`grep -rn "FCriticalSection\|FScopeLock" Source/UnrealCSharp`（本组文件 0 命中；`FCSharpEnvironment.cpp:282`/`:295` 与 `FCSharpEnvironment.h` 的 `CriticalSection` 是模块内既有用法，可作为加锁先例）。
用例：在非 GameThread 上 `Broadcast` 一个含 C# 回调的多播委托，同时在 GameThread 上 `Remove`。

---

#### [F-DEL-013] 把 UObject 内联属性地址当作 `TMap` key，UObject 被 GC 后 key 悬垂

- **类别**: 未定义行为 / 隐患
- **严重度**: **P2**（维持）
- **复核结论**: 部分确认 —— "用 UObject 内联属性地址（`Address`）作 `TMap` key、key 的生命期与宿主 UObject 绑定"是**代码事实**；"宿主被 GC 后新对象复用同一地址 → 串对象"是合理但**未实测**的推论（无法构造确定性复现，标为低置信度子论断）
- **可达性**: 活跃（`NewRef` 每次读委托属性都会走 `:27-33` 的按地址查询）
- **复核证据**: `Source/UnrealCSharp/Public/Registry/FDelegateRegistry.inl:61`（`GetAddress()` 作 key）、`:30`（按 `Address` 查 `Address2ManagedHandle`）、`:48`（写）、`:67`（删）；`Source/UnrealCSharp/Public/Reflection/Delegate/FDelegateBaseHelper.h:6-17`（`Address` 就是构造传入的 `void*`）；构造传入值 = `Source/UnrealCSharp/Private/Reflection/Property/DelegateProperty/FDelegatePropertyDescriptor.cpp:39`（`Property->GetPropertyValuePtr(InAddress)`，即 UObject 属性块内联地址）；同族 `Source/UnrealCSharp/Private/Registry/FOptionalRegistry.cpp:59` 与 `Public/Registry/FContainerRegistry.inl:62`；本组**没有任何 `BeginDestroy`/`AddReferencedObjects` 钩子**（`grep` 在两个 Delegate 目录 = 0 命中）。
- **级别变动**: 无（P2 维持；理由：机制成立、后果为静默数据串扰，但需要地址复用这一未证实前提，按 P2 隐患记录）
- **文件**: `Source/UnrealCSharp/Public/Registry/FDelegateRegistry.inl:61`
- **函数**: `FDelegateRegistry::TDelegateRegistryImplementation::RemoveReference`（同类：`FOptionalRegistry.cpp:59`、`FContainerRegistry.inl:62`）
- **置信度**: 中高

**现状（代码事实）**
```cpp
// FDelegateRegistry.inl:57-81
static auto RemoveReference(Class* InRegistry, const IManagedHandle InManagedHandle)
{
	if (const auto FoundValue = (InRegistry->*ManagedHandle2Value).Find(InManagedHandle))
	{
		const auto Address = (*FoundValue)->GetAddress();     // ← 返回 FDelegateBaseHelper::Address

		if (const auto FoundManagedHandle = (InRegistry->*Address2ManagedHandle).Find(Address))
		...
```
```cpp
// FDelegateBaseHelper.h:6-17 —— Address 就是构造时传入的 void*
	explicit FDelegateBaseHelper(void* InAddress) : Address(InAddress) {}
	void* GetAddress() const { return Address; }
```
```cpp
// 构造时传入的是 UObject 内联存储里的 FScriptDelegate 地址
// FDelegatePropertyDescriptor.cpp:39-40
	const auto DelegateHelper = new FDelegateHelper(Property->GetPropertyValuePtr(InAddress),
	                                                Property->SignatureFunction);
```

**调用上下文**
`Address` 在 `FDelegateRegistry.h:45,49` 的 `TMap<void*, IManagedHandle>` 中作 key；写入在 `FDelegateRegistry.inl:48`，读取在 `:30`/`:63`，删除在 `:67`。生命周期终点是 `RemoveReference`（由 C# 侧析构驱动，见 F-DEL-012）。

**问题**
`Address` 指向**宿主 UObject 的内联属性内存**（`FDelegateProperty::GetPropertyValuePtr` 的结果落在 UObject 的属性块内）。该地址的有效期 = UObject 的生存期。但 map 条目的生存期由 **C# 对象的 GC** 决定——两者**完全解耦**：
- UObject 先被 GC（例如关卡切换销毁 Actor），而 C# 侧委托对象仍被引用（例如被一个静态字段持有）→ `Address` 成为**悬垂地址**。此时仅作 key 比较不会解引用，**不会立即崩溃**，但**新分配的 UObject 恰好落在同一地址**时，`GetObject(InAddress)`（`:27-33`）会把**另一个对象的委托**误判为已存在并复用 → **串对象**（严重的数据损坏，静默）。
- 反之，C# 先 GC 则 `RemoveReference` 用悬垂 key 查表，大概率 miss（`:63`）→ 只是少清一个条目（可接受）。
- 没有任何地方在 UObject 销毁时通知注册表（本组文件内无 `AddReferencedObjects`/`BeginDestroy` 钩子；`grep -n "AddReferencedObjects\|BeginDestroy" Source/UnrealCSharp/Public/Reflection/Delegate Source/UnrealCSharp/Private/Reflection/Delegate` → 0 命中）。

**建议**
- 用 `TWeakObjectPtr<UObject> + 属性偏移`（或 `FDelegateHandle` + `UObject*` 的组合）替代裸地址作为 key，使 key 具备"宿主已死"的可判定性；
- 或为 `UDelegateHandler`/`UMulticastDelegateHandler` 增加 `BeginDestroy()` 覆写，在那里调用注册表清除对应条目（这是唯一能保证"UObject 死亡即清理"的位置）；
- 至少在 `GetObject`/`RemoveReference` 命中后校验宿主 UObject 仍有效（需要额外保存宿主指针）。

**验证方式**
`grep -rn "GetAddress()" Source/UnrealCSharp`（4 处：`FDelegateRegistry.inl:61`、`FContainerRegistry.inl:62`、`FOptionalRegistry.cpp:59`、以及各 helper 的定义）。
用例：创建 Actor A（含委托属性）并注册 C# 委托 → `DestroyActor(A)` → 强制 GC → 创建大量新 Actor 观察是否出现委托串场（可用 `ensure` 在 `GetObject` 命中时校验宿主一致性来捕获）。

---

### P3 组（初判定级 P3 / 5 条 → 实际：F-DEL-014/015/016/017/018 **全部维持 P3**）

#### [F-DEL-014] `ExecuteN`/`BroadcastN` 四层同构模板（22 个函数）与 `Add`/`AddUnique` 逐字重复

- **类别**: 可优化/可读性
- **严重度**: **P3**（维持）
- **复核结论**: **确认** —— 模板层数与个数经复核 grep 精确吻合
- **可达性**: 活跃
- **复核证据**: `grep "template <auto ReturnType" Source/UnrealCSharp/Public/Reflection/Delegate` → **22 命中**，分布 `FDelegateHelper.h:33,42,51,60,69,78,87`(7)、`DelegateHandler.h:40,55,70,85,100,115,130`(7)、`MulticastDelegateHandler.h:46,61,76,91`(4)、`FMulticastDelegateHelper.h:36,45,54,63`(4) —— 与"4 层 × 22 个"完全一致；`Add`/`AddUnique` 逐字重复区间 `MulticastDelegateHandler.cpp:77-89` vs `:94-106`（各 13 行）与 `FMulticastDelegateHelper.cpp:57-63` vs `:65-71` 亦经逐行比对确认；`FOptionalHelper.cpp:99-102` vs `:104-107` 同（`return Data;`）。
- **级别变动**: 无（P3 维持；理由：纯重复代码/可读性）
- **文件**: `Source/UnrealCSharp/Public/Reflection/Delegate/DelegateHandler.h:40`
- **函数**: `UDelegateHandler::Execute0/1/2/3/4/6/7`、`FDelegateHelper::Execute0/1/2/3/4/6/7`、`UMulticastDelegateHandler::Broadcast0/2/4/6`、`FMulticastDelegateHelper::Broadcast0/2/4/6`
- **置信度**: 高

**现状（代码事实）**
```cpp
// DelegateHandler.h:40-53
	template <auto ReturnType = EFunctionReturnType::Void>
	void Execute0() const
	{
		if (ScriptDelegate != nullptr)
		{
			if (ScriptDelegate->IsBound())
			{
				if (DelegateDescriptor != nullptr)
				{
					DelegateDescriptor->Execute0<ReturnType>(ScriptDelegate);
				}
			}
		}
	}
// :70-83 Execute2 同样的三层 if；:85-98 Execute3；…（共 7 个）
```
```cpp
// FDelegateHelper.h:33-40 —— 纯转发，第三层
	template <auto ReturnType = EFunctionReturnType::Void>
	void Execute0() const
	{
		if (DelegateHandler != nullptr) { DelegateHandler->Execute0<ReturnType>(); }
	}
// …共 7 个
```
```cpp
// MulticastDelegateHandler.h:46-59 —— 与单播版结构完全一致，共 4 个
// FMulticastDelegateHelper.h:36-43 —— 纯转发，共 4 个
```
```cpp
// MulticastDelegateHandler.cpp:75-90 vs :92-107 —— 前 13 行逐字符相同
void UMulticastDelegateHandler::Add(UObject* InObject, FMethodReflection* InMethod)
{
	if (MulticastScriptDelegate != nullptr)
	{
		if (!MulticastScriptDelegate->Contains(ScriptDelegate))
		{
			ScriptDelegate.Unbind();
			ScriptDelegate.BindUFunction(this, *FUNCTION_CSHARP_CALLBACK);
			MulticastScriptDelegate->Add(ScriptDelegate);
		}
	}
	DelegateWrappers.Add({InObject, InMethod});
}
void UMulticastDelegateHandler::AddUnique(UObject* InObject, FMethodReflection* InMethod)
{
	if (MulticastScriptDelegate != nullptr)
	{
		if (!MulticastScriptDelegate->Contains(ScriptDelegate))
		{
			ScriptDelegate.Unbind();
			ScriptDelegate.BindUFunction(this, *FUNCTION_CSHARP_CALLBACK);
			MulticastScriptDelegate->Add(ScriptDelegate);
		}
	}
	DelegateWrappers.AddUnique({InObject, InMethod});
}
```
```cpp
// FMulticastDelegateHelper.cpp:57-63 vs :65-71 —— 转发体完全相同
// FOptionalHelper.cpp:99-102 vs :104-107 —— GetData/GetAddress 实现完全相同
```

**调用上下文**
`ExecuteN` 的调用者只有一层：`FRegisterDelegate.cpp:80-173` 的 16 个 Implementation（每个都是 `if (const auto Helper = ...GetDelegate<...>(h)) { Helper->ExecuteN<...>(...); }`），这一层本身也高度重复。
`BroadcastN` 的调用者是 `FRegisterMulticastDelegate.cpp:148-186`。

**问题**
- 22 个 hand-written 模板函数 + `FRegisterDelegate`/`FRegisterMulticastDelegate` 中 30 个同构 Implementation，构成约 **1500 行的纯样板**。新增一个参数组合（例如 `Execute5`——当前确实缺失）需要在 4 个文件里各写一遍，极易漏改（`Execute5` 和 `Broadcast1/3/5/7` 的缺失正是这种不一致的表现）。
- `Execute` 系列的守卫逻辑在 `UDelegateHandler::ExecuteN` 里重复了 7 次（`ScriptDelegate != nullptr && IsBound() && DelegateDescriptor != nullptr`），一旦需要新增条件（例如 F-DEL-001 建议的 `DelegateWrapper` 有效性检查）就要改 7 处。
- `Add`/`AddUnique` 的重复是纯粹的复制粘贴，后续修 bug 时必须记得改两处（例如将来要在这段加锁或加失效清理）。

**建议**
1. 把三层守卫抽成一个私有辅助：
```cpp
// DelegateHandler.h
	template <typename Fn>
	void WithDelegate(Fn&& InFn) const
	{
		if (ScriptDelegate != nullptr && ScriptDelegate->IsBound() && DelegateDescriptor != nullptr)
		{
			InFn(*ScriptDelegate, *DelegateDescriptor);
		}
	}
	template <auto ReturnType = EFunctionReturnType::Void>
	void Execute0() const { WithDelegate([&](FScriptDelegate& D, FCSharpDelegateDescriptor& Desc){ Desc.Execute0<ReturnType>(&D); }); }
```
2. `Add`/`AddUnique` 合并为 `AddInternal(InObject, InMethod, bool bUnique)`。
3. 门面层的 `ExecuteN` 可以用一层宏或基类模板消除（`EnableBroadcast(bool)` + `using Base::ExecuteN`）。

**验证方式**
`grep -c "Execute0\|Execute1\|..." `；或直接统计：`grep -n "template <auto ReturnType" Source/UnrealCSharp/Public/Reflection/Delegate/*.h` 应为 22 行。

---

#### [F-DEL-015] `FDelegateWrapper.h` 的 `operator==` 用 `static` 而非 `inline`；`FDelegateBaseHelper` 是无用基类

- **类别**: 可优化/可读性
- **严重度**: **P3**（维持）
- **复核结论**: **确认** —— `static` 运算符与"无用基类"两项均经 grep 复核成立
- **可达性**: 活跃
- **复核证据**: `Source/UnrealCSharp/Public/Reflection/Delegate/FDelegateWrapper.h:12`（`static bool operator==(...)` 定义在头文件，非 `inline`）；`grep FDelegateBaseHelper`（路径 = 插件根）→ **11 命中** = `FDelegateBaseHelper.h:3,6,11`(自身 3) + `FDelegateHelper.h:4,7` + `FMulticastDelegateHelper.h:4,7` + `FDelegateHelper.cpp:4,10` + `FMulticastDelegateHelper.cpp:4,11`（两个派生类各 `#include` 1 + 继承 1 + 构造初始化列表 1）→ **无任何 `FDelegateBaseHelper*` 变量/参数/返回类型**，基类的虚析构与多态未被使用 ✓。
- **级别变动**: 无（P3 维持；理由：可读性/一致性）
- **文件**: `Source/UnrealCSharp/Public/Reflection/Delegate/FDelegateWrapper.h:12`
- **函数**: `operator==(const FDelegateWrapper&, const FDelegateWrapper&)`
- **置信度**: 高（`static` 事实确定）；"是否真的产生多份代码"为高置信度推断

**现状（代码事实）**
```cpp
// FDelegateWrapper.h:5-15（完整文件）
struct FDelegateWrapper
{
	TWeakObjectPtr<UObject> Object;

	FMethodReflection* Method;
};

static bool operator==(const FDelegateWrapper& A, const FDelegateWrapper& B)
{
	return A.Object == B.Object && A.Method == B.Method;
}
```
```cpp
// FDelegateBaseHelper.h:3-21（完整文件）
class FDelegateBaseHelper
{
public:
	explicit FDelegateBaseHelper(void* InAddress) : Address(InAddress) {}
	virtual ~FDelegateBaseHelper() = default;
public:
	void* GetAddress() const { return Address; }
private:
	void* Address;
};
```

**问题**
1. `static` 给函数**内部链接**，每个包含该头文件的翻译单元都会生成一份 `operator==`（对比 `inline` 只保留一份并允许 ODR 合并）。功能上可用（不会冲突），但代码体积与"看起来像误用"的问题存在；且 `static` 定义在头文件里会触发部分编译器/静态分析告警（`-Wunused-function`）。UE 代码库里同类运算符的标准写法是 `friend bool operator==(...)` 或 `inline bool operator==(...)`。
2. `FDelegateWrapper` 没有 `operator!=`，也没有 `GetTypeHash`——当前 `TArray` 只用到 `==`（`MulticastDelegateHandler.cpp:72` 的 `Contains`、`:111` 的 `Remove`、`:126` 的 `RemoveAll` 谓词），所以**不是 bug**；但若将来放进 `TSet`/`TMap` 会编译失败。
3. `FDelegateBaseHelper` 被 `FDelegateHelper`/`FMulticastDelegateHelper` 继承（`FDelegateHelper.h:7`、`FMulticastDelegateHelper.h:7`），但**全插件没有任何地方以 `FDelegateBaseHelper*` 持有派生类**——`grep "FDelegateBaseHelper"` 的 11 处命中全部是：自身头文件（3 处）、两个派生类的 `class X final : public FDelegateBaseHelper`（各 1 处）与构造初始化列表 `FDelegateBaseHelper(InAddress)`（各 1 处），以及 `#include`。因此虚析构、多态、基类抽象**都没有被使用**，基类存在的唯一作用是把 `void* Address` 成员放到了别处。这层间接让 F-DEL-013 中的"`Address` 到底是哪个对象的地址"变得更难追踪（需要跨文件才能看出它是 `FDelegateProperty::GetPropertyValuePtr(InAddress)`）。

**建议**
```cpp
// FDelegateWrapper.h
inline bool operator==(const FDelegateWrapper& A, const FDelegateWrapper& B)
{
	return A.Object == B.Object && A.Method == B.Method;
}
```
并把 `Address` 直接作为 `FDelegateHelper`/`FMulticastDelegateHelper` 的成员（删除 `FDelegateBaseHelper`），或在保留基类的同时补一个真正使用多态的场景。删除基类时注意 `GetAddress()` 的 4 个调用点（`FDelegateRegistry.inl:61` 等）需同步改名。

**验证方式**
`grep -rn "FDelegateBaseHelper" Source/ UnrealCSharp/Script/`（确认无基类指针用法）；`grep -rn "operator==" Source/UnrealCSharp/Public/Reflection/Delegate/`。

---

#### [F-DEL-016] `GetUObject()` / `GetFunctionName()` 返回的是桥对象自身，命名严重误导

- **类别**: 可读性 / 隐患
- **严重度**: **P3**（维持）
- **复核结论**: **确认** —— 返回值语义与命名不符属实，且"所有 C# 委托的 `GetFunctionName()` 都返回同一个 `CSharpCallBack`"这一关键推论有引擎源码支撑
- **可达性**: 活跃（`FDelegatePropertyDescriptor.cpp:30` 与 `FMulticastDelegatePropertyDescriptor.cpp:33-34` 两条赋值路径都在用）
- **复核证据**: `Source/UnrealCSharp/Private/Reflection/Delegate/DelegateHandler.cpp:88-96`（返回 `ScriptDelegate->GetUObject()` / `GetFunctionName()`，即桥对象自身与固定函数名）；写入方 `DelegateHandler.cpp:60`（`BindUFunction(this, FUNCTION_CSHARP_CALLBACK)`）、`MulticastDelegateHandler.cpp:83,100`（同一手法）；引擎语义 `ScriptDelegates.h:180-186`（`BindUFunction` 只存 `Object`+`FunctionName`）、`:356-369`（`GetUObject` 返回被绑对象）→ 故"共享桥对象"推论成立：`FDelegatePropertyDescriptor.cpp:30` 把目标属性的 `FScriptDelegate` 绑到**源 helper 的桥对象**上，若源侧 C# 对象先死，目标属性会静默失效（引擎 `IsBound()` 语义，`ScriptDelegates.h:159-170`）。
- **级别变动**: 无（P3 维持；理由：命名误导 + 语义陷阱，实际后果为静默失效，未达 P2 性能/隐患的独立可复现性门槛之外——它与 F-DEL-003 的赋值路径同源，避免重复计级）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Delegate/DelegateHandler.cpp:88`
- **函数**: `UDelegateHandler::GetUObject()` / `GetFunctionName()`（多播同构；再经 `FDelegateHelper`/`FMulticastDelegateHelper` 透出）
- **置信度**: 高

**现状（代码事实）**
```cpp
// DelegateHandler.cpp:88-96
UObject* UDelegateHandler::GetUObject() const
{
	return ScriptDelegate != nullptr ? ScriptDelegate->GetUObject() : nullptr;   // ← 返回 this（桥对象）
}

FName UDelegateHandler::GetFunctionName() const
{
	return ScriptDelegate != nullptr ? ScriptDelegate->GetFunctionName() : NAME_None;   // ← 返回 CSharpCallBack
}
```
```cpp
// 透出到 C# 侧用于构造新的 FScriptDelegate
// FDelegatePropertyDescriptor.cpp:30
	DestScriptDelegate->BindUFunction(SrcDelegateHelper->GetUObject(), SrcDelegateHelper->GetFunctionName());
// FMulticastDelegatePropertyDescriptor.cpp:33-34
	ScriptDelegate.BindUFunction(SrcMulticastDelegateHelper->GetUObject(),
	                             SrcMulticastDelegateHelper->GetFunctionName());
```
即：命名暗示"取回 C# 目标对象与方法名"，实际返回的是**桥对象 `this`** 与**固定字符串 `CSharpCallBack`**。真正的 C# 目标在 `FDelegateWrapper::Object`/`Method` 里，而**没有任何 getter 暴露它们**。

**问题**
- 语义误导：读者会以为 `GetUObject()` 是"C# 那边绑定的对象"，实际上它对**每一个** C# 委托都返回不同的 `UDelegateHandler`、`GetFunctionName()` 对**所有**委托都返回同一个 `CSharpCallBack`。
- 正因为"所有委托的 `GetFunctionName()` 都相同"，`FDelegatePropertyDescriptor::Set`（`:30`）把源 helper 的 `(this, CSharpCallBack)` 绑到目标属性上时，**目标属性的 `FScriptDelegate` 指向的是源 helper**，而不是 C# 目标。这在"把 A 的委托赋给 B 的委托属性"场景下产生**共享桥对象**：B 的回调会经 A 的 handler 的 `DelegateWrapper` 分发。若 A 的 handler 被销毁（C# 侧 A 被 GC）→ B 的属性指向一个悬垂的 `UObject*`（UE 侧是弱引用，`ProcessEvent` 不会被调用，静默失效）。
- 没有暴露真实目标的 getter，使调用方无法自证"绑定是否正确"。

**建议**
- 重命名为 `GetBridgeUObject()` / `GetBridgeFunctionName()`（或 `GetCallbackUObject()`/`GetCallbackFunctionName()`），并在注释里写明"返回的是桥对象自身，固定绑定 `CSharpCallBack`"。
- 增加真实目标的只读访问器：
```cpp
UObject* GetTargetObject() const { return DelegateWrapper.Object.Get(); }
const FMethodReflection* GetTargetMethod() const { return DelegateWrapper.Method; }
```
- 对 F-DEL-003 中的属性赋值场景，考虑改为**深拷贝语义**（把源的 `Object`/`Method` 注册到目标属性的独立 helper 上），而不是共享桥对象。

**验证方式**
`grep -rn "GetUObject()\|GetFunctionName()" Source/UnrealCSharp`（4 个定义 + 2 个透出 + 4 个调用点）。
用例：C# 中 `b.Delegate = a.Delegate;` → 让 A 被 GC → 触发 B 的委托 → 观察是否静默无回调。

---

#### [F-DEL-017] `FOptionalHelper::Initialize()` 是空实现死代码；`GetData`/`GetAddress` 重复；C# `TOptional<T>` 表示与 C++ 不一致

- **类别**: 死代码 / 可读性 / 一致性
- **严重度**: **P3**（维持）
- **复核结论**: **确认** —— 死代码判定与重复实现均经 grep/逐行复核
- **可达性**: 活跃（含死代码本身；`GetData`/`GetAddress` 有真实调用点）
- **复核证据**: ①死代码：`Source/UnrealCSharp/Private/Reflection/Optional/FOptionalHelper.cpp:32-34`（空体）+ 声明 `Public/Reflection/Optional/FOptionalHelper.h:16`；复核 grep `OptionalHelper->Initialize|Helper->Initialize\(\)`（插件根）= **0 命中**，且 `grep FOptionalHelper`（`Source/`）= **49 命中**（分布：`FOptionalHelper.cpp` 13、`FOptionalHelper.h` 4、`FOptionalRegistry.h` 8、`FRegisterOptional.cpp` 6、`TPropertyValue.inl` 7、`FOptionalPropertyDescriptor.cpp` 5、`FOptionalRegistry.cpp` 2、`FOptionalRegistry.inl` 2、`FCSharpEnvironment.cpp` 1、`FCSharpEnvironment.h` 1）—— 无一处调用 `Initialize()`。②重复实现：`:99-102` vs `:104-107`（逐字相同）。③C# 表示：`Script/UE/CoreUObject/TOptional.cs:7`（`class`，非值类型）、`:38`（`(int)HandleData.GetHandle(this)` 64→32 位截断）、`:44`（`Get()` 解箱）；句柄由 `Script/Interop/Handle/HandleData.cs:27-44,80-93` 以 `ConditionalWeakTable` 管理。
- **级别变动**: 无（P3 维持；理由：死代码 + 一致性）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Optional/FOptionalHelper.cpp:32`
- **函数**: `FOptionalHelper::Initialize()`、`GetData()`、`GetAddress()`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FOptionalHelper.cpp:32-34 —— 空函数体，且全插件无调用点
void FOptionalHelper::Initialize()
{
}
```
```cpp
// FOptionalHelper.cpp:99-107 —— 两个函数体完全相同
void* FOptionalHelper::GetData() const
{
	return Data;
}

void* FOptionalHelper::GetAddress() const
{
	return Data;
}
```
```csharp
// Script/UE/CoreUObject/TOptional.cs:7, 38, 44 —— C# 侧是 class（引用类型）
    public class TOptional<T>
    ...
        public override int GetHashCode() => (int)HandleData.GetHandle(this);   // 64 位句柄截断为 int
    ...
        public T Get() => (T)TOptionalImplementation.TOptional_GetImplementation(HandleData.GetHandle(this));
```

**调用上下文 / 问题**
1. **`Initialize()` 是死代码**：声明在 `FOptionalHelper.h:16`（public），定义在 `.cpp:32-34`（空体）。对 `FOptionalHelper` 的 49 处 grep 命中中**没有任何 `->Initialize()` 调用**（`FRegisterOptional.cpp` 调用的方法是 `Reset`/`IsSet`/`Get`/`Set`/`GetValuePropertyDescriptor`；`FOptionalPropertyDescriptor.cpp` 与 `TPropertyValue.inl` 只构造与注册）。构造函数已完成全部初始化（`:5-25`），该方法是遗留占位。**判定：确认死代码**（详见第 7 节）。
2. **`GetData()` 与 `GetAddress()` 语义重复**：两者返回同一个 `Data`。用途不同——`GetData()` 被 `FOptionalPropertyDescriptor.cpp:42` 用于 `Property->CopyCompleteValue(Dest, SrcOptional->GetData())`（数据语义），`GetAddress()` 被 `FOptionalRegistry.cpp:59` 用作 `TMap` key（身份语义）。同一指针承担两种角色，正是 F-DEL-013 的根源。建议 `GetAddress()` 改名 `GetRegistryKey()` 并加注释，或干脆合并（保留一个 + 在调用点注释用途）。
3. **C# 表示不一致**：
   - C++ `TOptional<T>` 是**值类型**（拷贝即拷贝），C# `TOptional<T>` 是**引用类型**（`TOptional.cs:7`）。因此"C++ 侧拷贝/赋值时旧值是否析构"在 C# 侧**不构成问题**（C# 引用赋值不产生新底层对象），但引入了**别名共享**：两个 C# 变量指向同一 handle 即指向同一 `FOptionalHelper` 与同一块 `Data`，一处 `Set` 影响另一处——与 C++ `TOptional` 的值语义**不一致**。
   - `GetHashCode()` 把 64 位 `IManagedHandle` 截断为 `int`（`TOptional.cs:38`），句柄的高 32 位丢失 → 哈希分布退化（同插件其他类型是否同样处理未核查，故列为 P3）。
   - C# 侧缺少与 `TOptional<T>::Emplace` 对应的方法；`HasValue` 也只暴露为 `IsSet()`。

**建议**
- 删除 `FOptionalHelper::Initialize()`（声明 + 定义）。
- 合并 `GetData`/`GetAddress`，或重命名以区分"数据指针"与"注册表 key"。
- 在 `TOptional.cs` 的类注释中明确"C# 侧是引用包装、非值语义"；`GetHashCode` 改为 `HandleData.GetHandle(this).GetHashCode()`（保持 64 位）。
- 若需要值语义，考虑把 `TOptional<T>` 改为 `readonly struct` + 显式 `Dispose`（但会牵动 `HandleData`/终结器设计，属于较大改动，需权衡）。

**验证方式**
- 死代码：`grep -rn "FOptionalHelper" Source/ Script/` 共 49 处命中，逐条确认无 `Initialize()` 调用；`grep -rn "OptionalHelper->Initialize\|Helper->Initialize()" Source/UnrealCSharp` → 0。
- `grep -n "GetData\|GetAddress" Source/UnrealCSharp/Private/Reflection/Optional/FOptionalHelper.cpp`（`:99`、`:104`）。

---

#### [F-DEL-018] `FRegisterMulticastDelegate` 重复注册 `"Contains"` 两次

- **类别**: Bug（注册表污染）/ 可读性
- **严重度**: **P3**（维持）
- **复核结论**: 部分确认 —— 重复注册属实（**缺陷行是 `:192`，首次注册在 `:190`**，`文件` 字段指向 `:190` 不准确）；"是否表现为错误"仍取决于 `FClassBuilder::Function` 的重复名语义（未读该类，见第 8 节）
- **可达性**: 活跃（静态对象 `:205` 在模块加载时构造即注册）
- **复核证据**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterMulticastDelegate.cpp:190` 与 `:192` 均为 `.Function("Contains", ContainsImplementation)`（逐字相同）；同文件共 **13** 个静态函数（`:14 Register`、`:21 UnRegister`、`:30 IsBound`、`:41 Contains`、`:64 Add`、`:85 AddUnique`、`:106 Remove`、`:127 RemoveAll`、`:139 Clear`、`:148/157/166/175` 四个 `GenericBroadcastN` —— 共 13 个）而构造中 `.Function(...)` 共 **14** 次（`:188-201`），多出的正是重复的 `Contains`；对照单播版 `Source/UnrealCSharp/Private/Domain/Interop/FRegisterDelegate.cpp:177-193`（16 个函数 / 16 次注册，无重复）。
- **级别变动**: 无（P3 维持；理由：注册表污染/可读性，功能上 C# 侧只调用一次，未见行为错误）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterMulticastDelegate.cpp:192`（首次注册在 `:190`）
- **函数**: `FRegisterMulticastDelegate::FRegisterMulticastDelegate()` 构造函数中的 `.Function(...)` 链
- **置信度**: 高（重复为事实）；影响程度为低置信度（取决于 `FClassBuilder::Function` 是否覆盖还是追加）

**现状（代码事实）**
```cpp
// FRegisterMulticastDelegate.cpp:186-201
			FClassBuilder(TEXT("FMulticastDelegate"), NAMESPACE_LIBRARY)
				.Function("Register", RegisterImplementation)              // 188
				.Function("UnRegister", UnRegisterImplementation)          // 189
				.Function("Contains", ContainsImplementation)              // 190
				.Function("IsBound", IsBoundImplementation)                // 191
				.Function("Contains", ContainsImplementation)              // 192  ← 重复
				.Function("Add", AddImplementation)                        // 193
				.Function("AddUnique", AddUniqueImplementation)            // 194
				.Function("Remove", RemoveImplementation)                  // 195
				.Function("RemoveAll", RemoveAllImplementation)            // 196
				.Function("Clear", ClearImplementation)                    // 197
				.Function("GenericBroadcast0", GenericBroadcast0Implementation)
				...
```
定义的 **13** 个静态函数（`:14,21,30,41,64,85,106,127,139,148,157,166,175`）与 14 次注册（`:188-201`）对比，**唯独 `Contains` 出现两次**（`:190`、`:192`），而 `FDelegate`（单播版，`FRegisterDelegate.cpp:177-193`）是 16 个函数对 16 次注册、无重复。对比单播版的注册顺序（Register/UnRegister/**Bind**/IsBound/UnBind/Clear/…），多播版 `:192` 的位置本应对应一个"Bind"语义的入口——多播没有 `Bind`，所以这一行**很可能是复制粘贴残留**。

**调用上下文**
静态对象 `[[maybe_unused]] FRegisterMulticastDelegate RegisterMulticastDelegate;`（`:203`）在模块加载时构造并执行注册。C# 侧 `Script/UE/Library/FMulticastDelegateImplementation.cs:30-35` 只调用一次 `__FMulticastDelegate_ContainsImplementation`，因此**功能上不表现为错误**——但它污染了反射注册表。

**问题**
- 若 `FClassBuilder::Function` 是"按名字插入 map"，第二次注册会覆盖第一次（无害但冗余）；
- 若是"追加到数组"，则会产生**两个同名方法**，C# 侧反射查找（按名字）可能命中错误条目，或 `GetMethod` 行为不确定；同时任何"遍历方法列表"的功能（如代码生成、编辑器方法列表）会显示重复项。
- 这是一个明确的信号：该注册块缺少测试覆盖（否则重复注册应被立即发现）。

**建议**
删除第 192 行；并在 `FClassBuilder` 层加重复名断言：
```cpp
// FClassBuilder::Function 内
ensureMsgf(!Functions.Contains(InName), TEXT("Duplicate function registration: %s"), *InName);
```

**验证方式**
`grep -n "\.Function(" Source/UnrealCSharp/Private/Domain/Interop/FRegisterMulticastDelegate.cpp | sort` 检查同名重复；对全部 `FRegister*.cpp` 做同样检查。

---

## 7. 死代码清单

统计方法：pwsh `Get-ChildItem -Recurse -File -Path Source, Script -Include *.h,*.cpp,*.inl,*.cs | Select-String -SimpleMatch -Pattern <符号> | Measure-Object`，扫描 1373 个文件（已排除 `obj`/`bin`/`Intermediate`/`Binaries`）。
判定：命中数 `==1`（仅声明或仅定义，无调用）算强死代码嫌疑。

| 符号 | 声明位置 | grep 命中数 | 判定 | 证据 |
|---|---|---|---|---|
| `FDelegateHelper::FDelegateHelper()`（默认构造） | `Private/Reflection/Delegate/FDelegateHelper.cpp:3-7` | 有真实调用点 | **非死代码** | `FCSharpBind.inl:127` 的 `new T()`（`T = FDelegateHelper`），入口 `FRegisterDelegate.cpp:18` 的 `FCSharpBind::Bind<FDelegateHelper>`，对应 C# 侧 `new FDelegate()` |
| `FMulticastDelegateHelper::FMulticastDelegateHelper()`（默认构造） | `Private/Reflection/Delegate/FMulticastDelegateHelper.cpp:3-7` | 有真实调用点 | **非死代码** | `FCSharpBind.inl:127` 的 `new T()`，入口 `FRegisterMulticastDelegate.cpp:18` |
| `FCSharpBind::BindImplementation(FClassReflection*, IManagedHandle)` | `Public/Registry/FCSharpBind.inl:125-132` | 2 处模板实例化 | 非死代码 | `FRegisterDelegate.cpp:18`、`FRegisterMulticastDelegate.cpp:18` 的 `FCSharpBind::Bind<T>(...)`；**注意它走的是 `new T()` 默认构造**，因此两层门面的默认构造函数都不是死代码 |
| `FOptionalHelper::Initialize()` | `Public/Reflection/Optional/FOptionalHelper.h:16` | 定义 1 + 声明 1（无调用点） | **确认死代码** | `grep -rn "FOptionalHelper" Source/ Script/` 全部 49 处命中逐条核对，无 `->Initialize()`；`MarkUnset` 命中数 = 1（仅定义）另证 `Reset` 是唯一 unset 入口 |
| `FDelegateBaseHelper`（整个基类） | `Public/Reflection/Delegate/FDelegateBaseHelper.h:3` | 11 | **强嫌疑（无用基类）** | 11 处 = 自身头 3 处 + 两个派生类的 `: public FDelegateBaseHelper`（各 1）+ 构造初始化列表 `FDelegateBaseHelper(InAddress)`（各 1）+ `#include`（各 1）+ 注释。**无任何 `FDelegateBaseHelper*` 变量/参数** |
| `FDelegateWrapper` | `Public/Reflection/Delegate/FDelegateWrapper.h:5` | 8 | 使用中 | `DelegateHandler.h:8`(include)、`:158`(成员)、`MulticastDelegateHandler.h:8`(include)、`:121`(成员)、`MulticastDelegateHandler.cpp:72`(`Contains`)、`:126`(`RemoveAll` 谓词)、`FDelegateWrapper.h:12`(运算符) |
| `UDelegateHandler::CSharpCallBack()` / `UMulticastDelegateHandler::CSharpCallBack()` | `DelegateHandler.cpp:19` / `MulticastDelegateHandler.cpp:23` | 5（含宏定义） | 非死代码（**反射入口**） | 函数体为空是设计：它只是 `BindUFunction` 需要的 `UFunction` 载具（`DelegateHandler.cpp:60`、`MulticastDelegateHandler.cpp:83/100`），由 UE 反射按名寻址（`FUNCTION_CSHARP_CALLBACK = "CSharpCallBack"`，`Macro/FunctionMacro.h:7`）。静态调用图里"无调用者"是正常的 |
| `FDelegateHelper::GetUObject()` / `GetFunctionName()` | `FDelegateHelper.cpp:73` / `:78` | 各自有真实调用点 | 非死代码 | `FDelegatePropertyDescriptor.cpp:30` |
| `FMulticastDelegateHelper::GetUObject()` / `GetFunctionName()` | `FMulticastDelegateHelper.cpp:97` / `:102` | 有真实调用点 | 非死代码 | `FMulticastDelegatePropertyDescriptor.cpp:33-34` |
| `UDelegateHandler::GetCallBack()` / `UMulticastDelegateHandler::GetCallBack()` | `DelegateHandler.cpp:98` / `MulticastDelegateHandler.cpp:164` | 6（含 2 定义） | 非死代码 | `FDelegateHelper.cpp:29`、`FMulticastDelegateHelper.cpp:30` |
| `UDelegateHandler::UnBind()` / `Clear()` | `DelegateHandler.cpp:72` / `:80` | `UnBind` 23 / `Clear` 多处 | 非死代码 | `FDelegateHelper.cpp:61` / `:69`；C# 入口 `FRegisterDelegate.cpp:67` / `:76` |
| `UMulticastDelegateHandler::Contains/Add/AddUnique/Remove/RemoveAll/Clear/IsBound` | `MulticastDelegateHandler.h:32-44` | 均有 C# 注册入口 | 非死代码 | `FRegisterMulticastDelegate.cpp:190/191/193/194/195/196/197` 分别注册为 `Contains`/`IsBound`/`Add`/`AddUnique`/`Remove`/`RemoveAll`/`Clear` |
| `FMulticastDelegateHelper::Add/AddUnique/Remove/RemoveAll/Clear/IsBound/Contains` | `FMulticastDelegateHelper.h:22-34` | 同上 | 非死代码 | 由 `FRegisterMulticastDelegate.cpp` 各 Implementation 调用（`:33,47,70,91,112,130,142,151,160,169,179`） |
| `UDelegateHandler::Execute0/1/2/3/4/6/7` | `DelegateHandler.h:40-143` | `Execute0` 12、`Execute7` 18 | 非死代码 | 由 `FDelegateHelper.h` 的同名模板调用，后者由 `FRegisterDelegate.cpp:80-173` 调用 |
| `FOptionalHelper::GetData()` | `FOptionalHelper.cpp:99` | 有调用点 | 非死代码 | `FOptionalPropertyDescriptor.cpp:42` |
| `FOptionalHelper::GetAddress()` | `FOptionalHelper.cpp:104` | `GetAddress` 总命中 42（含各容器同名方法） | 非死代码 | `Private/Registry/FOptionalRegistry.cpp:59` |
| `FOptionalHelper::Identical()` | `FOptionalHelper.cpp:57` | 有调用点 | 非死代码 | `FRegisterOptional.cpp:80` |
| `FRegisterDelegate` 的 16 个静态函数 | `FRegisterDelegate.cpp:14-173` | 定义 16 / 注册 16 | **无死函数** | `:178-193` 逐一对应，无遗漏 |
| `Broadcast0/2/4/6`（两层共 8 个） | `MulticastDelegateHandler.h:46-104`、`FMulticastDelegateHelper.h:36-70` | 各 12 | 非死代码 | `FRegisterMulticastDelegate.cpp:148-186` |

**结论**：本组**只有 1 处确认的死代码**（`FOptionalHelper::Initialize()`）和 **1 处强嫌疑的无用基类**（`FDelegateBaseHelper`）。任务书怀疑的 `FDelegateWrapper.h` / `FDelegateBaseHelper.h` 两个纯头文件**都不是死代码**——`FDelegateWrapper` 被两个 handler 真实使用（8 处命中），`FDelegateBaseHelper` 虽"无用"但被继承并有 `GetAddress()` 的真实调用（4 个注册表查找点）。

**[grep 核对记录（工具 `grep`，路径限定插件 `Source/` 或插件根）**：
| 符号/模式 | 命中数 | 与正文 |
|---|---|---|
| `FDelegateBaseHelper`（插件根，含 Script） | **11**（`FDelegateBaseHelper.h:3,6,11`、`FDelegateHelper.h:4,7`、`FMulticastDelegateHelper.h:4,7`、`FDelegateHelper.cpp:4,10`、`FMulticastDelegateHelper.cpp:4,11`） | 一致 |
| `\bDelegateWrapper\b`（`Source/`） | **3**（`DelegateHandler.h:158`、`.cpp:10`、`.cpp:64`） | 一致（`{DelegateWrapper}` 的表述模式不准确，但计数相同） |
| `DelegateWrapper`（子串，`Source/`） | **18**（含多播 `DelegateWrappers` 12 处 + 类型定义 3 处 + 单播 3 处） | 补充口径 |
| `FOptionalHelper`（`Source/`） | **49** | 一致 |
| `OptionalHelper->Initialize\|Helper->Initialize\(\)`（插件根） | **0** | 一致（证实死代码） |
| `MarkUnset`（`Source/`） | **1**（仅 `FOptionalHelper.cpp:74` 定义处） | 一致 |
| `template <auto ReturnType`（`Public/Reflection/Delegate`） | **22** | 一致 |
| `Execute5\|Broadcast1\|Broadcast3\|Broadcast5\|Broadcast7`（`Source/`） | **0** | 一致 |
| `IsInGameThread`（`Source/`） | **3**（见 F-DEL-012） | 一致 |
| `FDomain::GCHandle_Free`（`Source/`） | **32**；其中 `FOptionalRegistry.cpp` 仅 `:31`（位于 `Deinitialize`，`RemoveReference` 内 0 处） | 佐证 F-DEL-008 |
| `AddDynamic\|AddUniqueDynamic\|RemoveDynamic\|RemoveAllDynamic`（`Source/`） | **5**，全部与本层无关（`DynamicNewClassUtils`/`AddDynamicSection`），**委托绑定不使用 UE 的 `AddDynamic` 宏族** | 见下"横向维度" |

---

## 8. 未覆盖/存疑项

| # | 项 | 为什么没覆盖 | 影响 |
|---|---|---|---|
| 1 | ~~`FManagedFunctionDescriptor.cpp:61-91` 未读~~ → **[已补读]** `:61-91` 只有返回值写回与 `OutPropertyIndexes` 处理（`:67-88`），**无任何针对 `InMethod`/`InManagedHandle` 的防御代码** → F-DEL-001 结论不变；`FCSharpFunctionDescriptor.cpp` 仍未读 | — | 已消除 |
| 2 | `FClassBuilder::Function(...)` 的重复名语义（**覆盖**还是**追加**） | 未读 `Public/Binding/Class/FClassBuilder.*` | 决定 F-DEL-018 是"无害冗余"还是"注册表含两个同名方法" |
| 3 | `FDomain::GCHandle_Free` / `GCHandle_Alloc` 的具体实现与句柄表语义 | 未读 `Private/Domain/FDomain.*` | F-DEL-008 的"句柄泄漏"结论基于 `FDelegateRegistry.inl:75` 与 `FOptionalRegistry.cpp:55` 的**不对称**（事实），但泄漏的**量级**未量化 |
| 4 | `IManagedHandle` 的定义与 `IManagedHandleIsValid` / `InvalidManagedHandle` 的语义 | 未读 `Public/Domain/Script/IManagedHandle.h` | 影响 F-DEL-001 中"`GetObject(nullptr)` 返回 InvalidManagedHandle 后 `Runtime_Invoke` 的行为"这一判断 |
| 5 | ~~UE 侧 `FOptionalProperty` 三个 API 的精确契约（理由是"插件仓库不含引擎源码"）~~ → **[已核对，撤销该理由]** 本机存在 UE 5.6 源码并已读 `Runtime/CoreUObject/Public/UObject/PropertyOptional.h`：`IsSet:28-34`、`MarkSetAndGetInitializedValuePointerToReplace:35-57`（非 intrusive 布局下**仅未 set 时** `InitializeValue`，且恒 `return Data`）、`MarkUnset:58-74`（**会 `DestroyValue`**）、`GetValuePointerForReadOrReplace:113-122`（`checkSlow(Data && IsSet(Data)); return Data;`）；另核对 `Runtime/Core/Public/Misc/Optional.h:440-444` 确认 **`TIsTOptional_V` 确实存在**（4 个 cv 特化）→ 任何"因该 trait 不存在而降级"的理由无效 | — | 已消除；F-DEL-010 的"指针失效"子论断据此**撤销**，F-DEL-006 的"活值 Dest"机制据此**坐实** |
| 6 | `FDelegateProperty::InitializeValue` 在**已持值**属性上重复调用的安全性 | 未读 UE 侧 `FDelegateProperty` 实现 | 影响 F-DEL-003 的第二部分（`Property->InitializeValue(Dest)` 是否需要先 `ClearValue`） |
| 7 | 各后端（Mono / CoreCLR / LeanCLR）的 `Runtime_Invoke(InvalidManagedHandle, ...)` 行为差异 | 未读三个后端的 `FDomain` 实现 | 影响 F-DEL-001 的严重度描述（崩溃 vs 静默失败） |
| 8 | 其他 11 个注册表（字符串/容器/软引用等）是否存在同样的 `GCHandle_Free` 遗漏 | 超出本组文件范围（仅抽查了 `FContainerRegistry.inl:62` 的 `GetAddress()` 用法） | F-DEL-008 的建议中已要求横向审计，但未执行 |
| 9 | 域重载/热重载（hot reload）时注册表与桥对象的清理路径 | 未读 `FDelegateRegistry.cpp` / `FOptionalRegistry.cpp` 的调用方与 `FCSharpEnvironment::Deinitialize`（只读了 `FCSharpEnvironment.cpp:84` 附近的构造） | F-DEL-005 的"泄漏场景 2"标为待确认 |
| 10 | `Script/UE/Library/FDelegateImplementation.cs` 等 C# 侧的完整 API 面 | 只做了关键字 grep（`IsBound`/`UnBind`/`AddUnique`），未通读 | 死代码判定的 C# 侧覆盖不完整；但 C++ 侧每条 public 方法都找到了 C++ 调用点，故结论不受影响 |
| 11 | 性能维度（`ProcessEvent` 中的 `FString` 比较、`FindFunction` 重复查找、`TArray` 拷贝成本） | 属于次要维度，本组文件未做量化 | 未产出性能类发现；观察到 `Function->GetName() == FUNCTION_CSHARP_CALLBACK`（`DelegateHandler.cpp:6`）每次回调都构造 `FString` 并做字符串比较，`GetCallBack()`（`:100`）每次调用都 `FindFunction`——可作为后续 P3 候选 |
| 12 | ~~`FDelegateRegistry.cpp` 未读~~ → **[已补读]** `Source/UnrealCSharp/Private/Registry/FDelegateRegistry.cpp:18-55`：`Deinitialize` 遍历**两个** map，逐个 `delete Value` + `FDomain::GCHandle_Free(Key)` + 清空 → 模块卸载/域销毁会清空 map，原 F-DEL-005"场景 2"（域重载泄漏）**不成立** | — | 已消除 |

### 复核补充：横向维度核对 + 新观察（**均不新增 Finding 编号**）

1. **`AddDynamic`/`AddUniqueDynamic`/`RemoveDynamic` 配对** → **不适用**：本层 C#↔UE 委托绑定**不使用** UE 的 `AddDynamic` 宏族（`grep "AddDynamic|AddUniqueDynamic|RemoveDynamic|RemoveAllDynamic"` 在 `Source/` 命中 5 处，全部与本层无关：`DynamicNewClassUtils.*`、`UnrealCSharpPlayToolBar.cpp:54`、`DynamicDataSource.cpp:73/700`），而是直接 `FScriptDelegate::BindUFunction`（`DelegateHandler.cpp:60`、`MulticastDelegateHandler.cpp:83,100`）配 `Unbind`（`DelegateHandler.cpp:76`、`MulticastDelegateHandler.cpp:119,137,151`）/`Clear`（`DelegateHandler.cpp:84`、`MulticastDelegateHandler.cpp:146`）。配对结论见第 4 节 + F-DEL-004。
2. **多播/单播参数收集是否过滤 `CPF_ReturnParm`** → **已正确过滤**：`Source/UnrealCSharp/Private/Reflection/Function/FFunctionDescriptor.cpp:35-46` 在遍历 `CPF_Parm` 时，命中 `CPF_ReturnParm` 的属性**只**赋给 `ReturnPropertyDescriptor` 后 `continue`（不进 `PropertyDescriptors`）；`CPF_OutParm && !CPF_ConstParm` 才进 `OutPropertyIndexes`（`:50-58`）→ 传给 C# 的参数数组不含返回值，此维度健康。
3. **`FDelegateWrapper` 句柄释放** → 见 F-DEL-005/F-DEL-007 复核后的结论：`NewRef`/`NewWeakRef`/`FCSharpBind::Bind` 三条创建路径都有释放点，其中弱引用/`Bind` 路径靠 **C# 终结器**（生成器保证，`FDelegateGenerator.cpp:79`/`:451`），强引用路径额外有 `~TDelegateReference`。**未发现无释放点的墓碑条目**。
4. **[新观察 · 未编号]** `FOptionalPropertyDescriptor::Set`（`Source/UnrealCSharp/Private/Reflection/Property/OptionalProperty/FOptionalPropertyDescriptor.cpp:34-43`）与 F-DEL-003 同型：`:38 GetOptional(SrcManagedHandle)` 结果未判空即 `:42 Property->CopyCompleteValue(Dest, SrcOptional->GetData())` → 无效句柄时空指针解引用；同一函数 `:40 Property->InitializeValue(Dest)` 又落在**可能已持值**的 `TOptional` 属性内存上（"活值 Dest"族）。本报告不为它新开编号（该文件归 `01-…/04b-属性描述符-字符串枚举结构体Optional.md` 覆盖），建议由该报告统一裁定编号。
5. **[新观察 · 未编号]** `FMulticastDelegatePropertyDescriptor::Set`（`FMulticastDelegatePropertyDescriptor.cpp:27`）的 `Property->InitializeValue(Dest)` 落在可能已持有 `InvocationList`（`TArray`）的目标属性上 → 同属"活值 Dest 覆盖"族（引擎语义见 `UnrealType.h:1434`："assumes over uninitialized memory"）。同样交给上述报告裁定。
6. **[同族复核]** `FRegisterOptional.cpp:24`/`:49` 的 `Class->GetGenericArgument()` 与 `:28`/`:53` 的 `ValueProperty->SetPropertyFlags(...)` 均未判空（前者已并入 F-DEL-009 的复核证据，后者为 F-DEL-009 内的同族缺失，未单独编号）。

**未发现问题的维度（检查过且健康）**：
- `AddToRoot()`/`RemoveFromRoot()` 成对（`FDelegateHelper.cpp:24/38`、`FMulticastDelegateHelper.cpp:25/39`），且 `RemoveFromRoot` 通过 `FDelegateRegistry.inl:71 delete` → 析构链被覆盖；**[补充]** 触发这条链的 C# 终结器由生成器无条件生成（`FDelegateGenerator.cpp:79`/`:451`，产物 `Script/UE/Proxy/Engine/FTimerDynamicDelegate.cs:18`），另有 `FDelegateRegistry::Deinitialize`（`FDelegateRegistry.cpp:18-55`）作域销毁兜底 → 见 F-DEL-005（复核后 P2）。
- `new`/`delete` 配对：`ScriptDelegate`（`DelegateHandler.cpp:27`/`:40`）与 `DelegateDescriptor`（`:29`/`:48`）在多播版亦配对（`MulticastDelegateHandler.cpp:34/47`、`:36/55`）。**未发现无条件泄漏的裸 `new`**。
- `FRegisterDelegate` 的 16 个 Interop 入口**全部**对 `GetDelegate<>()` 结果做了判空（`FRegisterDelegate.cpp:35/53/64/73/82/91/100/109/119/129/138/148/158/168`），与 F-DEL-003 形成鲜明对比——说明作者知道该 API 会返回空，只是漏了属性描述符这一条路径。
- `FOptionalRegistry`/`FDelegateRegistry` 的 `Deinitialize` 会遍历清理并 `GCHandle_Free`。
- 平台兼容：本组代码无 `PLATFORM_*` 分支、无 `int` 宽度假设、无 `sprintf`；`void*` 作 map key 在 32/64 位下均由 UE 容器正确处理。唯一平台相关点是 `TOptional.cs:38` 的 64→32 位句柄截断（已列入 F-DEL-017）。

---

## 9. 发现汇总

| ID | 严重度（复核后） | 类别 | 一句话 | 位置 |
|---|---|---|---|---|
| F-DEL-001 | P1 | Bug/UB | 回调链全程不校验 `Method`/`Object`，空指针直达 `InMethod->Runtime_Invoke` | `DelegateHandler.cpp:10` |
| F-DEL-002 | P0 | Bug/UB | 多播广播遍历 `DelegateWrappers` 时回调可 `Remove`/`Clear` → 陈旧 end 指针/越界，`Clear` 即 use-after-free | `MulticastDelegateHandler.cpp:11` |
| F-DEL-003 | P0 | Bug | 委托属性 `Set` 未判空 `GetDelegate<>()` 结果 → 空指针解引用 | `FDelegatePropertyDescriptor.cpp:30` |
| F-DEL-004 | P1 | Bug/静默失效 | 单播 `DelegateWrapper` 无任何解绑点；`Bind` 的 `!IsBound()` 前置条件使桥可能永不接管 | `DelegateHandler.cpp:64` |
| F-DEL-005 | P2 | 资源驻留 | `AddToRoot()` 锚定桥 UObject，回收要等 C# 终结器 + `AsyncTask` + GC（非确定性驻留，非永久泄漏） | `FDelegateHelper.cpp:24` |
| F-DEL-006 | P1 | 泄漏 | `FOptionalHelper` 释放只 `Free` 不析构内部值；`Set` 在活值上 `InitializeValue`（`:91`，权威修复点） | `FOptionalHelper.cpp:38`、`:91` |
| F-DEL-007 | P2 | 资源放大 | `NewWeakRef` 每次 `new` helper 且用 3 参 `AddDelegateReference`（不写地址映射）→ 重复分配累积 | `FDelegatePropertyDescriptor.cpp:56` |
| F-DEL-008 | P1 | 泄漏 | `FOptionalRegistry::RemoveReference` 缺 `FDomain::GCHandle_Free` → C# 句柄泄漏（兄弟注册表都有） | `FOptionalRegistry.cpp:55` |
| F-DEL-009 | P3 | 空指针 | `FRegisterOptional::Register1/2` 未判空 `GetClass()`（触发前提是无效句柄/域销毁，非常态） | `FRegisterOptional.cpp:16` |
| F-DEL-010 | P2 | Bug/UB | `FOptionalHelper::Get()` 未 set 时返回未构造值地址并被装箱（指针失效子论断已撤销） | `FOptionalHelper.cpp:82` |
| F-DEL-011 | P3 | 防御性 | `FOptionalHelper` 置空成员后各访问方法不判空（未找到可达的释放后使用路径） | `FOptionalHelper.cpp:72` |
| F-DEL-012 | P3 | 并发 | 无 `IsInGameThread` 断言；`~TDelegateReference` 与 Interop `UnRegister` 线程假设不一致 | `TDelegateReference.h:13` |
| F-DEL-013 | P2 | UB/隐患 | UObject 内联属性地址作 `TMap` key，宿主 GC 后 key 悬垂 → 可能串对象（低置信度子论断） | `FDelegateRegistry.inl:61` |
| F-DEL-014 | P3 | 重复代码 | `ExecuteN`/`BroadcastN` 四层 22 个同构模板 + `Add`/`AddUnique` 逐字重复 | `DelegateHandler.h:40` |
| F-DEL-015 | P3 | 可读性 | `operator==` 用 `static` 而非 `inline`；`FDelegateBaseHelper` 是无用基类 | `FDelegateWrapper.h:12` |
| F-DEL-016 | P3 | 可读性/隐患 | `GetUObject()`/`GetFunctionName()` 返回桥对象自身，命名严重误导 | `DelegateHandler.cpp:88` |
| F-DEL-017 | P3 | 死代码/一致性 | `FOptionalHelper::Initialize()` 死代码；`GetData`/`GetAddress` 重复；C# `TOptional<T>` 表示不一致 | `FOptionalHelper.cpp:32` |
| F-DEL-018 | P3 | Bug | `FRegisterMulticastDelegate` 重复注册 `"Contains"` 两次（缺陷行 `:192`） | `FRegisterMulticastDelegate.cpp:192` |

**数量统计（复核后）：P0 = 2（F-DEL-002/003），P1 = 4（F-DEL-001/004/006/008），P2 = 4（F-DEL-005/007/010/013），P3 = 8（F-DEL-009/011/012/014/015/016/017/018），合计 18。**
（初判定级为 P0 = 3 / P1 = 5 / P2 = 5 / P3 = 5；净变动 2 条：F-DEL-006 P2→P1、F-DEL-005 P1→P2，其余 16 条维持。**无撤销（非缺陷）条目**；被证伪的是 4 处**子论断**：F-DEL-006 的"分配器不匹配"、F-DEL-007 的"弱引用路径无释放兜底"、F-DEL-009 的"未注册类型返回 nullptr"、F-DEL-010 的"返回指针会失效"。）

**对任务书 6 个必答问题的直接回答**
1. **C# 委托是强引用还是弱引用？** → 都不是。C++ 侧只存 `IManagedHandle`（整数）+ `TWeakObjectPtr<UObject>`（**弱**，`FDelegateWrapper.h:7`）+ `FMethodReflection*`（**裸指针**，`:9`）。**没有 C# 强引用导致的委托永不回收**；真实缺陷是**悬垂/静默失效**（F-DEL-001，复核后 P1）**加上 UE 侧 helper/rooted UObject 的资源驻留与放大**（F-DEL-005 P2 / F-DEL-007 P2，释放由生成器保证的 C# 终结器兜底）。
2. **绑定/解绑是否配对？** → 见第 4 节 13 行表格。`AddToRoot`↔`RemoveFromRoot`、`new`↔`delete helper` **配对**；单播 `DelegateWrapper` **无解绑点**（F-DEL-004）；弱引用路径的地址映射**不配对**（F-DEL-007）。`FDelegateHandle` 在本层**完全未被使用**（无 `Remove`/`RemoveAll` 句柄式管理，全部靠 UObject+函数名绑定）。
3. **多播遍历中移除元素？** → **是 P0 缺陷**。既没有"`RemoveAll` 后重新开始"，也没有"拷贝再遍历"（F-DEL-002）。
4. **`FDelegateWrapper`/`FDelegateBaseHelper` 是死代码吗？** → **都不是**。`FDelegateWrapper` 8 处命中且是两个 handler 的真实成员；`FDelegateBaseHelper` 虽为无用基类但有 4 个 `GetAddress()` 真实调用点。唯一确认死代码是 `FOptionalHelper::Initialize()`（F-DEL-017）。
5. **`FOptionalHelper` 正确性？** → has-value 标志由 UE 的 `FOptionalProperty` 管理，`Set` 走 `MarkSet...`、`Reset` 走 `MarkUnset`，**这两个语义正确**；但**销毁路径不析构**（F-DEL-006，复核后 **P1**：既含 `Free` 跳过析构，也含 `:91` 在**已持活值**的 `Data` 上做 `InitializeValue`）、`Get` 不检查 is-set（F-DEL-010，P2），C# 侧是引用类型而非值类型（F-DEL-017，P3）。**[修正]** 原第 5 条"`Set` 丢弃 API 返回值"的疑点已解决：引擎 `GetValuePointerForReadOrReplace`/`MarkSetAndGetInitializedValuePointerToReplace` 恒 `return Data`（`PropertyOptional.h:35-57,113-122`），因此丢弃返回值**不会写错位置**，真正的缺陷是前者（活值覆盖泄漏）。
6. **五者职责是否重叠？** → 见第 1.3 节。**可合并**：`FDelegateHelper`/`FMulticastDelegateHelper` 可模板化合一；`FDelegateBaseHelper` 可删除；`UDelegateHandler`/`UMulticastDelegateHandler` 可部分合并；最明确的重复是 22 个 `ExecuteN`/`BroadcastN` 模板与 `Add`/`AddUnique` 的 13 行逐字重复（F-DEL-014）。

