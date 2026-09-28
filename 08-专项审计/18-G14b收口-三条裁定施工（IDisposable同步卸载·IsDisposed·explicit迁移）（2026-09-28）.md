# G14b 施工轮：三条裁定的落地（`IDisposable` + 原生同步卸载 / 只读 `IsDisposed` / `implicit` → `explicit` 全量迁移）

> **分析范围**：`08-专项审计/17-G14收口-C#资源所有权与API契约（2026-09-28）.md` §2.5 的**三条待裁定项** —— `F-CS2A-005`（`IDisposable`）、`F-CS2A-006`（`IsDisposed`）、`F-CS2A-019`（6 处 `implicit operator`）。
> **维护者裁定（2026-09-28）**：**`F-CS2A-005` → B**（`IDisposable` **+** 19 个 `FRegister*.cpp` 的原生**同步**卸载）；**`F-CS2A-006` → A**（只读 `IsDisposed`，**不**抛异常、**不**改成员失败语义）—— 🔴 **同轮复核后回退**：`IsDisposed` 因**零生产消费者**已删除（见 **§0.1**，`F-CS2A-006` 回到未处置）；**`F-CS2A-019` → A**（6 处改 `explicit`，**含工程侧全量迁移**）。
> **基线**：插件仓 HEAD **`fca72a8c7ddf5c1d5bbae74734e99230ae0fe8c8`** **+ 上一轮 G14 的三处未提交改动**。⇒ 两轮改动**同处一个工作区**：`FRegisterOptional.cpp` 一个文件里同时含 G14 的 `FDomain::GCHandle_Free` 与本轮的同步分支。本报告 §4.3 的"改前"基线即"HEAD + G14 改动"。
> **状态**：三条**全部落地**；🟢 **已提交 `74cb5727bcaa95a945a8d3e3efb72fbc8570629b`**（父 `fca72a8c7ddf5c1d5bbae74734e99230ae0fe8c8`）—— ⚠️ **该提交同时含上一轮 G14（§2.13）的三处改动**；⚠️ **工程侧（`UnrealCSharpTest`）32 文件仍未提交**（工程仓 HEAD 仍为 2026-07-27 的 `98e6fb25`）。
> **改动面**：
> - 插件侧 **原生 19 文件 `+185/−38`** —— `UnRegisterImplementation` 加 `if (IsInGameThread()) { 同步 } else { AsyncTask }`；⚠️ `−38` 是 async 分支体的**缩进 +1 tab**，属"包裹式守卫"的必然形态（与 G9 轮同类，见 `08-…/16` §4.5）；
> - 插件侧 **C# 16 文件** —— `IDisposable` + `bIsDisposed` + `Dispose()` + `using System;`（其中 **6 个**文件同时含 `implicit` → `explicit`）；⚠️ 施工时另加的只读 `IsDisposed` **已于同日复核删除**（§0.1）⇒ **净增公开成员仅 `Dispose()`**；
> - 工程侧 **32 文件 `+372/−165`** —— 019-A 迁移（**132** 处属性赋值 cast、**7** 个混合类型 `TestEqual` 重载、**98** 处实参 cast）+ 本轮 **9 条**新用例。
> **验证**：🟢 **已提交 `74cb5727`**（本轮的 **19 原生 + 16 C# 文件**含于其中），并在提交树上强制重编后重跑 —— 见 **§0.2**（`-game` **1740 / 1740、0 failed**、两阶段 **2 × 1740 / 1740**、`dotnet build` **0/0**；UBT 首次 up-to-date ⇒ touch 后强制重编 **Succeeded**，执行器 130.88 s）。施工期（提交前）的对照证据：🟢 **改前对照（只回退原生同步分支）= 1743 / 2 失败** ⇒ **实证了裁定 B 的同步分支是承重的**（§4.3）。

---

## 0.2 🟢 提交登记与"提交树 == 已验证形态"的核对（2026-09-28）

**提交**：`74cb5727bcaa95a945a8d3e3efb72fbc8570629b`（父 `fca72a8c7ddf5c1d5bbae74734e99230ae0fe8c8`）—— **一笔同时含 §2.13（G14）与本节（G14b）两轮**。提交后插件仓工作区 `git status --porcelain` 只剩 **`M Source/SourceCodeGenerator`**（预存，非这两轮）；`git diff 74cb5727 -- Source Script` **无输出** ⇒ **提交树 == 提交时的工作区**。

🔴 **提交前维护者做了 3 处非语义微调**（逐条核对，均**不改变行为**）：

| # | 位置 | 内容 | 性质 |
|---|---|---|---|
| 1 | **15 个** C# 包装类 | 删掉 `using System;` 之后的那个空行 | 纯空白（⇒ 本报告的补丁比提交多 15 行；`FAnsiString.cs` 不在其列） |
| 2 | **16 个** C# 包装类 | `System.GC.SuppressFinalize(this);` → **`GC.SuppressFinalize(this);`** | 等价改写（同文件已有 `using System;`） |
| 3 | `FRegisterSubclassOf.cpp` | 同步分支的 `RemoveMultiReference<TSubclassOf<UObject>>(...)` **折为一行**（异步分支保持折行） | 纯排版（⇒ 该文件比补丁多 1 行） |

⇒ 差异合计 **16 行 = 15 空行 + 1 折行**，与"补丁 `+294/−44` vs 提交 `+279/−44`（C# 16 文件）"和"补丁 `+185/−38` vs 提交 `+186/−38`（原生 19 文件）"逐项吻合。

**提交树重跑（本轮唯一有效的运行期证据，替换 §4.2 的施工期数字）**：

| 项 | 结果 | 证据 |
|---|---|---|
| `dotnet build Script/Script.sln -c Debug -nodeReuse:false` | **0 Warning / 0 Error**（真实重编，59.8 s） | 终端输出 |
| UBT `UnrealCSharpTestEditor Win64 Development`（首次，提交树） | ⚠️ **`Target is up to date`** ⇒ **不构成**"提交源码编出"的证据（见下方踩坑） | `Saved/Retest/build-74cb5727-g14b.log` |
| UBT（**touch 19 个已改文件后强制重编**） | **Result: Succeeded**，执行器 **130.88 s** ⇒ 确有编译动作 | **`build-74cb5727-g14b-forced.log`** |
| `-game` 全量 | **1740 / 1740、0 failed**（`LogUnrealCSharp: Error` 0、`ensure`/`assert` 0） | **`out/G14b-committed2/G14b-committed2-r01-csv01.csv`** |
| 两阶段域重建（`SetActive 0` → `SetActive 1` → PIE #2） | **2 × 1740 / 1740**、`LogBlueprint: Error` 0、`ensure`/`assert` 0 | **`out/G14b-committed2-r1/`**（2 CSV + 驱动日志） |
| 产物新鲜度 | 强制重编后的 `UnrealEditor-UnrealCSharp.dll` 与 `Content/Script/{UE,Game}.dll` 均晚于提交时间（15:26）⇒ 上述运行期证据对应**提交树** | 文件 mtime |

> ♻️ **本条踩坑（第 7 条，已计入 §4.6）**：**`git status` 干净 + UBT 报 `Target is up to date` ≠ "二进制是提交树编的"** —— 本轮实际发生：提交后第一次 UBT 直接 up-to-date（前一次构建的源码内容与提交树**内容一致**时 UBT 按内容哈希判定，故不重编）；要拿到"提交树编出来的"证据，必须 `touch` 受影响源文件后重编，并在构建日志里看到 `Compile/Link` 动作与可观的执行器时间。

---

## 0.1 🔴 维护者复核后的追加变更（同日）：`IsDisposed` 已删除 ⇒ **006-A 回退**

复核发现：**`IsDisposed` 零生产消费者** —— 插件运行时 **0** 读、16 055 份生成产物 **0** 读，唯一读取点是本报告 §7 新加的 5 条测试断言；而裁定 A 的原意是"给用户一个只读查询手段"，在**没人读**的前提下它只是 16 处无消费者的公开 API ⇒ 按"不留无消费者的公开 API"删除。

| 项 | 内容 |
|---|---|
| 删除范围 | 16 个包装类各 **1 行** `public bool IsDisposed => bIsDisposed;`（+ 其后的空行）＝ `g14b-drop-isdisposed.ps1`（幂等、CRLF/BOM 保持） |
| **保留** | `bIsDisposed` 字段（**不能删**：它是 `Dispose()` 与终结器的**幂等守卫**，被 `Dispose()` 自身读写，每个文件 3 处）与 `Dispose()` 本身（`IDisposable` 契约完整） |
| 测试调整 | 删 3 条（`DisposeIsDisposedFalseBefore`/`…TrueAfter`/`ExplicitCastSubclassOfIsDisposed`）、改判 2 条为"句柄已移除"判据（`DisposeIsIdempotent`→`DisposeTwiceKeepsHandleRemoved`、`DisposeMapIsDisposed`→`DisposeMapRemovesHandleImmediately`）⇒ Dispose 家族 **9 → 6 条**，套件 **1743 → 1740** |
| 影响 | **`F-CS2A-006` 回退为未处置**（权威口径回到原文：`Dispose` 后继续调用成员**静默降级**，用户无法区分"已释放"与"内容为空"）；`01-全局问题总表` 的 `F-CS2A-006` 行、`09-…/00` §2.14、本表 `00-总览/03`、`INDEX.md`/`README.md` 均已同步 |
| 验证（复跑） | `dotnet build` **0 Warning / 0 Error**；`-game` **1740 / 1740、0 failed**（用例名 `onlyInNew = 2`（两条改名）/ `onlyInPrevious = 5`（3 删 + 2 改名））；两阶段 **2 × 1740 / 1740**；`LogUnrealCSharp: Error` 0、`ensure`/`assert` 0 |
| ⚠️ 仍然成立的部分 | 裁定 **005-B 不受影响**（`IDisposable` + 19 文件原生同步卸载全部保留，改前对照结论不变）；裁定 **019-A 不受影响**（`explicit` 迁移完整）；§2.2 的"语义边界"一行随属性删除而**作废**（不再有该属性可读） |

---

## 0. 覆盖范围与阅读清单

| 对象 | 规模 | 读/改 |
|---|---|---|
| `Source/UnrealCSharp/Private/Domain/Interop/FRegister*.cpp` | 58 个文件，其中 **19** 个含 `UnRegisterImplementation` | ✅ 19 个全读全改（其余 39 个无 `UnRegister`，脚本已逐个报告"signature not found"） |
| `Script/UE/CoreUObject/{TArray,TMap,TSet,TOptional,FString,FName,FText,FAnsiString,FUtf8String,TFieldPath,TWeakObjectPtr,TLazyObjectPtr,TSoftObjectPtr,TSoftClassPtr,TSubclassOf,TScriptInterface}.cs` | **16** 个包装类 | ✅ 全读全改 |
| `Script/UE/CoreUObject/TArray.cs`（`typeof(T).IsValueType` 家法先例） / `HandleData.cs` | — | 复核 |
| `Source/UnrealCSharp/Public/Registry/FContainerRegistry.inl` / `FStringRegistry.inl` | — | ✅ 读（`RemoveReference` 尾部确证 `FDomain::GCHandle_Free`） |
| `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:288-309` | — | ✅ 读（`if (IsInGameThread())` 家法先例） |
| 引擎侧 `Runtime/Core/Private/Async/Async.cpp:56`、`TaskGraph.cpp` | — | 读（`AsyncTask` = `TGraphTask<FAsyncGraphTask>::CreateTask().ConstructAndDispatchWhenReady(...)`；**未找到"同线程内联"快速路径** ⇒ 改用实测回答，见 §1.4/§4.3） |
| 工程侧 `Script/Game/UnrealCSharpTest/**` | 32 个受跟踪文件 | ✅ 32 个（脚本化改写 + 逐文件抽查） |
| 工程侧 `Script/UE/Proxy/**`（16 055 份产物） | — | ⚪ **只做编译探测**（结论：**0 错误** ⇒ 产物不依赖这 6 个隐式转换） |

---

## 1. 第 0 步：三条的前提核查

### 1.1 `F-CS2A-005`（005-B 的面）：19 处 `UnRegisterImplementation` 同形

19 个函数的**形状完全一致**，只有 `RemoveXxx` 一行不同：

```cpp
		static void UnRegisterImplementation(const IManagedHandle InManagedHandle)
		{
			AsyncTask(ENamedThreads::GameThread, [InManagedHandle]
			{
				(void)FCSharpEnvironment::GetEnvironment().<RemoveXxx>(InManagedHandle);
			});
		}
```

| 家族 | `RemoveXxx` | 文件数 |
|---|---|---|
| 字符串 | `RemoveStringReference<{FString,FName,FText,FAnsiString,FUtf8String}>` | 5 |
| 容器 | `RemoveContainerReference<{FArrayHelper,FMapHelper,FSetHelper}>` | 3 |
| 多引用 | `RemoveMultiReference<{TWeakObjectPtr,TSoftObjectPtr,TSoftClassPtr,TLazyObjectPtr,TSubclassOf,TScriptInterface}>` | 6 |
| 委托 | `RemoveDelegateReference<{FDelegateHelper,FMulticastDelegateHelper}>` | 2 |
| 其它 | `RemoveOptionalReference` / `RemoveStructReference` / `RemoveFieldPathReference` | 3 |

### 1.2 `F-CS2A-006`（006-A 的面）：16 个包装类各有 finalizer，**0** 个可 `Dispose`

`grep -c "~"` 于 `Script/UE/CoreUObject/*.cs` ⇒ 16 个类（含 `TArray/TMap/TSet/TOptional`、5 个字符串、`TFieldPath`、6 个指针包装）**只有终结器**；`IDisposable`/`IsDisposed`/`ObjectDisposedException` 于插件整棵 `Script/` 树 **0 命中**（`Dispose(` 仅 `Interop/Bridge/LogBridge.cs:126/136` 的 `TextWriter.Dispose(bool)` 覆写，与所有权无关）——与 `08-…/17` §1.1 一致。

### 1.3 `F-CS2A-019`（019-A 的面）：**编译探测**取代静态推演

把 6 处 `implicit` 改成 `explicit` 后直接编译（`dotnet build Script/Script.sln -c Debug`），让编译器枚举全部隐式转换使用点：

| 判据 | 实测 |
|---|---|
| 编译错误总数（首轮探测） | **916** |
| 其中 `CS0266`（"需要显式 cast"的真站点） | **132** |
| 其中 `CS1503`（重载集塌陷后的级联） | **784** |
| 站点分布 | 插件**产物侧 0 错误**（16 055 份 `Script/UE/Proxy/**` 不依赖这 6 个隐式转换）；**全部落在工程侧测试代码**：312 个调用点 / 36 个文件 |
| 按目标类型 | `TSoftClassPtr<UObject>` 28、`TSoftObjectPtr<UObject>` 28、`TSubclassOf<UObject>` 26、`TWeakObjectPtr<UObject>` 18、`TLazyObjectPtr<UObject>` 18、`TScriptInterface<ITestDynamicInterface>` 14 |

⇒ **这一步把"破坏性变更"的规模从形容词变成数字**（本库的既定纪律：先量后改）。

### 1.4 🔴 一个必须先回答的引擎侧疑点：`AsyncTask` 在游戏线程上会内联执行吗？

**这决定了 005-B 的原生改动是不是"等价改写"（即是否有实际作用）。**

- 源码线索：`AsyncTask(ENamedThreads::Type Thread, TUniqueFunction<void()> Function)` 的实现只有一行 —— `TGraphTask<FAsyncGraphTask>::CreateTask().ConstructAndDispatchWhenReady(Thread, MoveTemp(Function));`（`Runtime/Core/Private/Async/Async.cpp:56`），**没有**"目标线程==当前线程则直接调用"的快速路径；但任务图的**派发路径**（`FTaskGraphInterface::QueueTask`）是否会在同线程时内联，本轮**未逐行走完**。
- ⇒ **改用实测**：先实现 C# 侧 `Dispose()`（原生不动）跑一轮，再把原生同步分支加上跑一轮（§4.3）。判据 = "`Dispose()` 返回时该包装的托管句柄是否已从 `HandleData` 移除"（`HandleData.GetObject(handle) == null` 即已移除）。

**结果（§4.3）**：不加同步分支时两条"立即性"判据均为 **false**，加上后均为 **true** ⇒ **`AsyncTask` 不会在游戏线程内联**，`Dispose()` 之后的原生释放要等游戏线程下一次排空任务图 ⇒ **裁定 B 的原生同步分支是承重改动，不是等价改写**。

---

## 2. 逐条改法

### 2.1 裁定 005-B：19 个 `FRegister*.cpp` 的 `UnRegisterImplementation` 加游戏线程同步分支

```cpp
		static void UnRegisterImplementation(const IManagedHandle InManagedHandle)
		{
			if (IsInGameThread())
			{
				(void)FCSharpEnvironment::GetEnvironment().RemoveContainerReference<FArrayHelper>(
					InManagedHandle);
			}
			else
			{
				AsyncTask(ENamedThreads::GameThread, [InManagedHandle]
				{
					(void)FCSharpEnvironment::GetEnvironment().RemoveContainerReference<FArrayHelper>(
						InManagedHandle);
				});
			}
		}
```

- **形状依据**：`FCSharpEnvironment.cpp:298-307` 的既有家法（`if (IsInGameThread()) { 同步 } else { 异步排队 }`）——**未新增**任何 helper、宏或公开 API。
- **线程模型**：`Dispose()` 由用户调用（通常在游戏线程）⇒ 走同步分支；终结器在**终结器线程**运行 ⇒ `IsInGameThread()` 为 false ⇒ 仍走 `AsyncTask`（保持原语义）。
- ⚠️ **残余风险（如实登记）**：同步分支引入"**在 native 调用栈内 Dispose 自身**"的新时序（例如某个 C# 回调里把正在被 native 使用的同一容器 `Dispose` 掉）。原异步语义对这类重入是"延后一拍"的天然缓冲，同步后不再有。本轮**未构造该时序的运行期用例**（§5），建议后续轮次按"回调内 Dispose"补一条探针。
- ⚠️ **缩进口径**：async 分支体 **+1 tab** 是包裹式守卫的必然结果（本轮 `−38` 行全出自此，无逻辑改动）。

### 2.2 裁定 005-A（+ 006-A，后者已回退）：16 个包装类的 `IDisposable`（~~只读 `IsDisposed`~~ ⇒ §0.1 删除）

```csharp
using System;
...
    public class TArray<T> : IEnumerable<T>, IDisposable
    {
        public TArray() => TArrayImplementation.TArray_RegisterImplementation(this, GetType());

        ~TArray() => Dispose();

        private bool bIsDisposed;

        public bool IsDisposed => bIsDisposed;

        public void Dispose()
        {
            if (!bIsDisposed)
            {
                bIsDisposed = true;

                TArrayImplementation.TArray_UnRegisterImplementation(HandleData.GetHandle(this));
            }

            System.GC.SuppressFinalize(this);
        }
```

- **幂等**：`bIsDisposed` 使 `Dispose()` 与终结器**互斥**（无论哪个先到，实际释放只发生一次）；native 侧 `RemoveReference` 另有 `Find` 守卫（`FContainerRegistry.inl:70`）⇒ 双保险。
- **`SuppressFinalize` 的必要性**：若省略，Dispose 后终结器会再排一次 `AsyncTask`（无害但多一次任务图往返）⇒ 按 .NET 标准模式保留。
- **`IsDisposed` 的语义边界（⚠️ 该属性已于同日删除，本节保留为历史记录）**：施工时它表示"**本包装的释放已被发起**"（用户 `Dispose` 或终结器），**不**表示"native 侧已释放" —— 后者在 `Dispose` 走异步分支（非游戏线程）时仍在下一 tick；也**不**能反映"native 驱动的释放"（那需要 `HandleData.IsAlive` 桥接，属新面）。复核确认它**零生产消费者** ⇒ 按"不留无消费者的公开 API"删除（§0.1）；删除后 `F-CS2A-006` 回到原文口径：**用户无法区分"已释放"与"内容为空"**。
- **未覆盖**：生成产物里的 `UStruct` 包装（`Script/UE/Proxy/**` 的 `FVector` 等 16 000+ 份）**未**加 `IDisposable` —— 它们由生成器产出，加接口需改生成器 + 全量重生成（本库一贯"生成产物 0 改动"）⇒ 登记为下一轮候选（§5）。

### 2.3 裁定 019-A：6 处 `implicit` → `explicit` + 工程侧全量迁移

**插件侧（6 行）**：

| 文件 | 改动 |
|---|---|
| `TWeakObjectPtr.cs:19` / `TLazyObjectPtr.cs:19` / `TSoftObjectPtr.cs:19` / `TScriptInterface.cs:19` | `implicit operator X<T>(T InObject)` → `explicit` |
| `TSoftClassPtr.cs:19` / `TSubclassOf.cs:18` | `implicit operator X<T>(UClass InClass)` → `explicit` |

⚠️ **不在范围内**：5 个字符串包装的 `implicit operator FString(string)` 等**保留**（它们是"字符串字面量 → 包装"的零跨越成本转换，与 6 个指针包装"每次转换 = 一次 native `new` + 一次 `AddMultiReference`"性质不同；`F-CS2A-019` 原文点的就是这 6 个）。

**工程侧迁移（32 文件 `+372/−165`）**，三段：

1. **132 处属性赋值**（14 文件）—— 形状完全统一（`InterfaceValue = this;`、`SubclassOfValue = GetClass();` 之类），脚本在 RHS 外包一层显式 cast：
   ```csharp
   -            InterfaceValue = this;
   +            InterfaceValue = (TScriptInterface<ITestDynamicInterface>)this;
   ```
2. **7 个混合类型 `TestEqual` 重载**（1 文件，`TestCore/TestCoreSubsystem.cs`）—— 原重载集只覆盖"同型两侧"，`(wrapper, 原始类型)` 组合在隐式转换消失后无候选。新增：
   `TSubclassOf<UObject>+UClass`、`TSoftClassPtr<UObject>+UClass`、`TSoftObjectPtr<UObject>+UObject`、`TWeakObjectPtr<UObject>+UObject`、`TLazyObjectPtr<UObject>+UObject`、`TScriptInterface<ITestInterface>+UObject`、`TScriptInterface<ITestDynamicInterface>+UObject`（重载体内做显式 cast，**测试语义不变**）。
3. **98 处实参 cast**（25 文件）—— 主要是每处测试开头那句 `USubsystemBlueprintLibrary.GetGameInstanceSubsystem(this, UTestCoreSubsystem.StaticClass())`（约 90 处）与若干把 `this`/`GetClass()` 传给原生形参的点：
   ```csharp
   -            ... GetGameInstanceSubsystem(this, UTestCoreSubsystem.StaticClass()) ...
   +            ... GetGameInstanceSubsystem(this, (TSubclassOf<UGameInstanceSubsystem>)UTestCoreSubsystem.StaticClass()) ...
   ```

---

## 3. 运行时契约回核（E1–E7）

| # | 契约 | 出处 | 结论 |
|---|---|---|---|
| E1 | 移除路径**确实**释放托管句柄（"立即性"判据成立的前提） | `FContainerRegistry.inl:86`、`FStringRegistry.inl:81` | 两者都在 `RemoveReference` 尾部调用 `FDomain::GCHandle_Free(InManagedHandle)` ⇒ 同步移除后 `HandleData.GetObject(handle) == null` ✓ |
| E2 | `AsyncTask` 是否同线程内联 | `Async.cpp:56` + §4.3 实测 | **不内联**（改前两条"立即性"判据 false）⇒ 同步分支承重 |
| E3 | 家法形状 | `FCSharpEnvironment.cpp:298-307` | `if (IsInGameThread()) { sync } else { deferred }` ⇒ 本轮 19 处照此 |
| E4 | `Dispose` 与终结器互斥 | `bIsDisposed` + `~X() => Dispose()` | 两者任一先到都只释放一次；`SuppressFinalize` 抑制第二次任务图往返 |
| E5 | `Dispose` 后继续使用成员 | `GetContainer`/`GetString` 查不到 ⇒ `return 0` / no-op | **静默降级**（`F-CS2A-006` 的行为**按裁定未改**）；实测 `Array.Num() == 0` ✓ |
| E6 | `explicit` 只改"能否隐式转换"，不改造物语义 | 6 个转换运算符体外无改动 | 构造路径逐字未变；`ExplicitCastSubclassOfGet` 用例断言 `Get()` 语义不变 ✓ |
| E7 | 重复释放安全 | `RemoveReference` 的 `Find` 守卫（`FContainerRegistry.inl:70`）+ `bIsDisposed` | 二次 `Dispose()` 不崩、不重复释放 ✓（用例 `DisposeIsIdempotent`） |

---

## 4. 验证

### 4.1 构建

| 项 | 命令 | 结果 |
|---|---|---|
| UBT（改前对照态：**只有 G14 改动**） | `Build.bat UnrealCSharpTestEditor Win64 Development -Project=… -WaitMutex` | **Result: Succeeded**（`Saved/Retest/build-g14b-beforesync2.log`） |
| UBT（终形态：+ 19 文件同步分支） | 同上 | **Result: Succeeded**（`build-g14b-final.log`，含 `Compile Module.UnrealCSharp.cpp` ⇒ 改动确实参编） |
| C# | `dotnet build Script/Script.sln -c Debug -nodeReuse:false` | **0 Warning / 0 Error**（多轮；⚠️ 首轮遇到 `CS2012` 文件锁 ⇒ 先 `dotnet build-server shutdown` 再编，见 §4.6） |
| 行尾/BOM 自检 | 19 原生 + 16 C# 全量统计 `(?<!\r)\n` | **0**（脚本化改写后逐文件核对，见 §4.6 第 3 条） |

### 4.2 `-game` 全量 + 两阶段域重建

| 轮次 | 断言 | 通过 | 失败 |
|---|---|---|---|
| `-game` 全量（**`IsDisposed` 删除前**，`out/G14b-final/`） | 1743 | 1743 | **0** |
| 两阶段 #1 / #2（删除前，`out/G14b-r1/`） | 1743 × 2 | 1743 × 2 | **0** |
| `-game` 全量（**终形态：`IsDisposed` 已删**，`out/G14b-nodisposed/`） | **1740** | 1740 | **0** |
| 两阶段 #1 / #2（终形态，`out/G14b-nodisposed-r1/`） | **1740 × 2** | 1740 × 2 | **0** |

- `LogUnrealCSharp: Error` **0**、`Unhandled Exception` **0**、`ensure`/`assert` **0**、`LogBlueprint: Error` **0**。
- 基线：G14 轮终形态 **1734** ⇒ 施工时新增 **9** 条用例；删除 `IsDisposed` 后 Dispose 家族由 9 → **6** 条（§0.1：删 3 条、改判 2 条）⇒ 终形态 **1740**。
- 用例名比对（终形态 vs 删除前）：`onlyInNew = 2`（`DisposeTwiceKeepsHandleRemoved`、`DisposeMapRemovesHandleImmediately`，均为改名）/ `onlyInPrevious = 5`（3 条删除 + 2 条改名）。

### 4.3 🟢 改前/改后对照（**本轮最硬的一段证据**，同时回答了 §1.4 的引擎疑点）

做法：用 `Saved/Retest/g14b-sync-unregister.ps1 -Mode revert` **只回退 19 个原生文件的同步分支**（C# 侧 `Dispose`/`IsDisposed` 与工程侧迁移全部保留）→ UBT 重编 → 跑同一套件。

| 判据 | 改前（原生仅 `AsyncTask`） | 改后（+ `IsInGameThread()` 同步分支） |
|---|---|---|
| 断言 / 失败 | 1743 / **2** | 1743 / **0** |
| `DisposeContainerRemovesHandleImmediately`（`TArray<int>`） | **false** | **true** |
| `DisposeStringRemovesHandleImmediately`（`FString`） | **false** | **true** |
| `DisposeIsDisposedFalseBefore` / `…TrueAfter` / `…IsIdempotent` / `DisposeThenUseIsSilent` / `DisposeMapIsDisposed` | true | true |
| `ExplicitCastSubclassOfGet` / `…IsDisposed` | true | true |

⇒ 两条结论：① **`AsyncTask` 不在游戏线程内联**，原实现在 `Dispose()` 之后的原生释放要等下一次任务图排空 ⇒ **裁定 B 的同步分支可被运行期观测到，是承重改动**；② 新用例对同步分支**敏感**（改前必失败），因此它们是这条契约的**永久回归判据**。

### 4.4 019-A 迁移的收敛过程（可复核）

| 阶段 | 剩余编译错误 | 动作 |
|---|---|---|
| 只改 6 个关键字 | **916**（132 `CS0266` + 784 `CS1503`） | 编译探测（§1.3） |
| + 132 处属性赋值 cast（脚本） | 784 → **196** | `g14b-fix-casts.ps1` |
| + 7 个混合 `TestEqual` 重载 | 196 → **98** | 手写（`TestCoreSubsystem.cs`） |
| + 98 处实参 cast（脚本） | **0** | `g14b-fix-args.ps1` |
| 去重（构建日志同一错误出现两份导致 66 行被 cast 两次） | 0 | `g14b-collapse-casts.ps1` |

### 4.5 风格自检

- 插件侧原生：**未**新增注释 / 日志 / `throw` / `try-catch` / `ensure` / `check`；**未**新增 helper、宏、协议面或导出签名（`IsInGameThread()` 是引擎既有 API）。
- 插件侧 C#：**未**新增注释 / 日志 / `throw` / `try-catch`；新增公开成员仅 `Dispose()` 与 `IsDisposed`（裁定要求的两项）与 `: IDisposable`。
- 工程侧：新增 9 条用例 + 7 个重载 + 迁移 cast；`Script/UE/Proxy/**` **0 改动**。
- ⚠️ **一处形态说明**：原生 `−38` 行全是 async 分支体缩进 +1 tab，逐行核对无逻辑改动。

### 4.6 踩坑（留给后续轮次，六条都是"会造成错误结论或返工"的）

1. **构建日志里同一错误出现两份** ⇒ 不去重会把 cast **应用两次**（本轮实测 66 行变成 `(T)(T)expr`）。判据：脚本必须先按 `(file,line,col,target)` 去重；已应用的要能幂等收敛（`g14b-collapse-casts.ps1`）。
2. **PowerShell 5.1 把无 BOM 的 UTF-8 `.ps1` 当 ANSI 读** ⇒ 脚本里的中文注释会破坏解析（本轮首个脚本因此报 `Unexpected token`）。⇒ 本仓脚本**一律 ASCII-only 注释**。
3. **CRLF 文件里 `$` 不匹配 `\r` 之前的位置** ⇒ 以 `[^\r\n]*$` 结尾的正则匹配失败或吞掉行尾（本轮首版 dispose 脚本因此产生 2 处 bare-LF 并吞掉一个空行）⇒ **行数组式改写**（split/join + 显式 `$nl`）比"带终止符的正则替换"更稳。
4. **缩进不能硬编码**：类的声明缩进是 4 空格、成员是 8；原生函数声明是 2 tab、lambda 体是 4 tab ⇒ 脚本必须先**从源码派生缩进**（本轮首版硬编码 3 tab，产出整体多一层缩进）。
5. 🔴 **`git checkout --` 会连带回退"上一轮的未提交改动"**：本轮为修 16 个 C# 文件而行 `git checkout`，把 G14 轮在 `TOptional.cs` 的一行（`TOptional_GetImplementation<T>`）**静默回退**，直到编译报 `CS0411` 才暴露。⇒ 凡工作区含**跨轮未提交改动**，回退必须限定到"本轮真正改过的 hunk"，或回退后用 grep/编译逐条复核上一轮的锚点（本轮的锚点：`typeof(T).IsValueType`、`FDomain::GCHandle_Free`、`TOptional_GetImplementation<T>`）。
6. **脚本追加换行的幂等性**：`if ($text.EndsWith("\n")) { $newText += $nl }` 在 split/join 之后会多加一个空行（本轮 19 个原生文件各多 1 行尾部空行）⇒ 判据 = `$newText.EndsWith($nl)` 也要判；修复后原生 diff 回到"**恰好 HEAD + G14 的 +4 行**"的干净基线。
7. 🔴 **"工作区干净 + UBT 报 `Target is up to date`"不等于"二进制是提交树编出来的"**：提交（`74cb5727`）后第一次 UBT 直接 up-to-date（当上一轮构建所见的源码**内容**与提交树一致时，UBT 按内容哈希判定 ⇒ 不重编，尽管源码在提交前被维护者改过——只是改动是空白/等价形态）⇒ **"提交树已重跑"这类声明必须以"强制重编"为前提**：`touch` 受影响的源文件 → 重编 → 构建日志里看到 `Compile/Link` 与可观的执行器时间（本轮强制重编 130.88 s），再跑套件。否则拿到的可能是**上一版二进制**的成绩（与 G9 轮记的 `Copy-Item` 保留 mtime 是同一族陷阱）。

---

## 5. 未取得 / 留给下一轮（如实声明）

1. **只测 CoreCLR / Win64 / 编辑器 Development**（Mono / LeanCLR 未复测）；同步分支的 `IsInGameThread()` 在非游戏线程 Dispose 时仍走异步 —— **该路径未被单测覆盖**（要构造"从工作线程 Dispose"的用例）。
2. **`F-CS2A-005` 的"确定性释放"只到"发起即同步移除"**：`Dispose()` 之后的 **native 资源析构时序**（`delete FArrayHelper`、`FScriptArray` 释放等）与"释放是否跨帧"未量化；`F-CS2A-019` 的 native 分配收益亦**未量化**（本轮只消除"隐式"，未建"10000 次转换 ⇒ native 计数"的装置 —— 该装置在裁定 019-A 后被判定为不再必要，但报告保留为可选项）。
3. **同步分支的重入时序未测**（§2.1 的残余风险）："在 C# 回调内 Dispose 正在被 native 使用的同一对象"这一时序本轮**未构造用例**。
4. **生成产物侧的 `IDisposable` 未做**：`Script/UE/Proxy/**` 的 `UStruct` 包装类（16 000+ 份）仍是 finalizer-only —— 需改生成器 + 全量重生成（本库一贯避免）。
5. **终结器路径未单测**：`Dispose` 未调用时终结器仍会释放（`~X() => Dispose()`），但该路径的运行期断言需要"强制 GC + 等任务图排空"，本轮未构造（`GetObject == null` 在弱句柄下会被 GC 本身满足，无法作为判据）。
6. **工程侧用例仍未提交**（与历轮同款尾巴）：本轮 **32 文件 `+372/−165`**；工程仓最近一次提交仍是 2026-07-27 的 `98e6fb25`。
7. **`F-CS2A-006` 的"抛 `ObjectDisposedException`"路线未做**（裁定选 A：只读查询）⇒ "释放后继续调用成员"仍是**静默降级**，用户需自行查 `IsDisposed`。

---

## 6. 复用价值（给下一轮）

1. **"改前对照只需回退一半"**：本轮裁定 005-B 的对照**只回退原生同步分支**（C# 侧保留），因为要验证的是"同步 vs 异步"这一条差异。为此把改动做成**带 `-Mode revert` 的脚本**（`g14b-sync-unregister.ps1`），比手工回退可靠得多，且可重复。
2. **代码探测优先于静态推演**：`implicit → explicit` 的影响面没有靠 grep 估，而是**改关键字 + 编译**，让编译器给出 132 个真站点与 312 个调用点（并证明产物侧 0 影响）。凡"改签名/改转换语义"的裁定，第一步都应是编译探测。
3. **"立即性"是最好的判据**：`HandleData.GetObject(handle) == null`（同步移除已完成）既证明了同步分支生效，又天然区分异步实现 —— 比计时/日志更硬。同型判据可用于任何"释放是否落在此刻"的契约。
4. **脚本化改写的五条纪律**（§4.6）：去重（构建日志双份）、ASCII-only、行数组式改写（CRLF 的 `$` 陷阱）、缩进从源码派生、**回退前先确认上一轮未提交改动不会被连带回退**。
5. **裁定落地的形态**：三条裁定各自落成"最小可验证改动"（19×同步分支 / 16×`IDisposable`+`IsDisposed` / 6 关键字+32 文件迁移），并在报告里明确**残余风险**（重入时序、语义边界、未覆盖面）——裁定结束不等于风险归零。

---

## 7. 附：装置与留档

| 项 | 路径 |
|---|---|
| **提交** | 🟢 **`74cb5727bcaa95a945a8d3e3efb72fbc8570629b`**（父 `fca72a8c7ddf5c1d5bbae74734e99230ae0fe8c8`；**同时含上一轮 G14（§2.13）的三处改动**）；提交后工作区仅剩预存的 `M Source/SourceCodeGenerator`；⚠️ **工程侧 32 文件仍未提交**（工程仓 HEAD `98e6fb25`） |
| 脚本（可复用、幂等） | `Saved/Retest/g14b-sync-unregister.ps1`（19 原生文件；`-Mode apply\|revert\|dry`）、`g14b-add-dispose.ps1`（16 C# 文件；`-Mode apply\|dry`）、`g14b-drop-isdisposed.ps1`（复核删除 16 处属性）、`g14b-fix-casts.ps1`（CS0266 属性赋值）、`g14b-fix-args.ps1`（CS1503 实参）、`g14b-collapse-casts.ps1`（去重收敛）、`g14b-pairs.ps1`（重载缺口枚举） |
| 补丁留档（**提交前**形态） | `fix-G14b-idisposable-explicit.patch`（**21 123 B**）、`fix-G14b-sync-unregister.patch`（**18 259 B**）；⚠️ 两者与**提交树**相差 **16 行**（15 空行 + 1 折行，见 §0.2 表）——**引用提交形态请以 §0.2 为准，补丁只作施工期留档**；上一轮：`fix-G14-optional-handles.patch`（2 538 B，与提交树一致 ✅） |
| 构建日志 | `build-74cb5727-g14b.log`（提交树首次 UBT，**up-to-date**）、**`build-74cb5727-g14b-forced.log`（touch 后强制重编 = Succeeded，执行器 130.88 s ← 唯一有效的"提交树编出"证据）**、`build-g14b-beforesync2.log`（改前对照态）、`build-g14b-final.log`（施工期终形态）、`build-g14b-1.log`（中途） |
| 运行产物 | **`out/G14b-committed2/`（终形态 `-game`）与 `out/G14b-committed2-r1/`（两阶段 ×2）← 提交树强制重编后的成绩**；施工期：`out/G14b-final/`（1743/0）、`out/G14b-r1/`、`out/G14b-nodisposed/`（1740/0）、`out/G14b-nodisposed-r1/`；改前对照：`out/G14b-before-sync2/`（1743/**2**）；⚠️ `out/G14b-before-sync/` **作废**（那一轮因 revert 脚本的括号 bug 编译失败，跑的是旧二进制，**不得引用**） |
| 新增用例（终形态 **6** 条；施工时 9 条，其中 3 条随 `IsDisposed` 删除、2 条改判 —— 见 §0.1） | `Script/Game/UnrealCSharpTest/UnitTest/Interop/HandleOwnership/UnitTestSubsystem.cs` 的 `TestDisposeOwnership()`：`DisposeContainerRemovesHandleImmediately`、`DisposeThenUseIsSilent`、`DisposeTwiceKeepsHandleRemoved`、`DisposeStringRemovesHandleImmediately`、`DisposeMapRemovesHandleImmediately`、`ExplicitCastSubclassOfGet` + `UnitTest/UnitTestSubsystem.cs` 1 行接线 |
| 删除脚本（复核用） | `Saved/Retest/g14b-drop-isdisposed.ps1`（16 处属性删除；`-Mode apply\|dry`） |
