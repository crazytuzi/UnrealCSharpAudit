# 轮 B-28 收口：`F-CS1-040` 注册整体失败 → 既有状态通道 fail-closed ＋ `F-FL-014` 使用面普查与公开 API 替换（含 1 处前提更正）

> **来源**：维护者指定 —— 轮 B-28 = [`09-…/05`](../09-问题清单/05-已完成（P2与P3）.md) §27 与 [`08-…/67`](67-轮B27收口-源生成器不抛半边与CodeAnalysis构建闸门（2026-10-15）.md) §263 共同给出的**下一顺位**两条：`F-FL-014`（P2 · S=3.08 · T2#26）＋ `F-CS1-040`（P2 · S=3.08 · T2#28），同轮并行。
> **插件基线**：**`df487a6d`**（B-27 交付提交）→ 本轮工作树。
> **环境**：UE 5.6.0（`D:\file\UnrealEngine\5.6`）· 后端 **CoreCLR**（`-game` 套件）＋ **Android（NDK r25b / clang 14.0.6）控制项**；跨版本对照 = **UE 5.0 引擎源码**（`D:\file\UnrealEngine\5.0`，注册表 `HKCU:\…\Epic Games\Unreal Engine\Builds` 同机可得）。
> **口径**：遵守 `README.md` `#L143` 与 [`08-…/66`](66-待决策页-三问（Q1四路线·Q2口径例外·Q3失败判据）（2026-10-15）.md) 的 Q2 裁定 —— **不新增注释 / 日志 / `throw` / `try-catch` / `ensure` / `check` / 公开 API**；失败一律静默收敛到**既有取值域**。

---

## 0. 摘要

| 项 | 读数 |
|---|---|
| **改动面** | **4 文件 `+8/−7`**（`FCSharpEnvironment.cpp` `+1/−4`、`FScriptDomainImpl.inl` `+5/−1`、`FCoreCLRDomain.h` `+1/−1`、`FMonoDomain.h` `+1/−1`）；C# 0 / 生成器 0 / 生成产物 0 / 协议面 0 / 导出签名 0 / **公开 API 0** / 注释·日志·异常·断言 **全 0** |
| **对象** | `F-CS1-040`（06-CSharp运行时与脚本）＝**注册整体失败可检测**（施工）＋ **"跳转 0 崩溃"前提更正**；`F-FL-014`（02-UnrealCSharpCore核心）＝**使用面普查 ＋ 1 处公开 API 替换**，宏层面收窄为 P3 |
| **F-FL-014 普查** | 5 个宏中 **4 个零使用**；宏调用点 **3 → 2**；**UE 5.6 上参与编译的调用点 2 → 1**（唯一存活 = `FObjectInitializer::bIsDeferredInitializer`，**5.6 无公开替代**，已逐行核实）；使用点 ① 换公开 `UObjectBase::GetFlags()`（**5.0.0–5.8.3 全 32 个 release tag 逐一核验：签名/可见性/位语义一致、无需版本门控**）、使用点 ③ 在 5.6 上**本来就**走公开 `FGameplayTagNode::GetDevComment()`（版本门控 `UE_F_GAMEPLAY_TAG_NODE_GET_DEV_COMMENT = 5.6.0`） |
| **F-CS1-040 前提更正** | ① "契约漂移 ⇒ 全量调用**跳转 0 崩溃**"**已由 `90f239f4` 消解** —— 生成产物早带 `if (MethodPointer != nint.Zero)` 守卫（`UnrealTypeSourceGenerator.cs:1197`）＋ 非 void `return default`；真实后果 = **静默 no-op**（`-game` 实测见 §3）；② `GetMethod` 现在**同时核对参数计数**（`MethodBridge.cs:148-160`）；③ 注册整体失败**在改前完全静默**（域报"健康"，harness 归档一张**只有表头**的 CSV） |
| **F-CS1-040 施工** | 共享 `RegisterBinding()` 的"桥函数指针为 `nullptr`" ⇒ **既有状态通道 fail-closed**（`bIsInitialized = false` ⇒ `IsInitialized()` / `FDomain::IsLoadSucceed()`），形态同轮 A `F-DOM-004`；`const` 面 3 处（两个 private 声明 ＋ 共享定义） |
| **A/B（人为把 `FUNCTION_METHOD_BRIDGE_REGISTER_BINDING` 改名 ⇒ 指针 `nullptr`）** | 改前：域**全程报健康**（`is-initialized 1`）、`RegisterBinding` 调用 1 次；改后：域**全程 fail-closed**（`is-initialized` 从不为真）、`RegisterBinding` 调用 **2 次**（`FScriptDomainScope` 重试 1 次）⇒ **信号从"无"变为"既有下游据此停手"**（§5） |
| **交付态判据** | Windows UBT **Succeeded**（7 actions / 52.28 s）＋ **Android UBT Succeeded**（`BUILD SUCCESSFUL in 26s` / 82.77 s，控制项）＋ 套件 **1931 / 0 ×2**、CSV SHA **`B429F3EE…`（与 B-24～B-27 同 SHA）** ＋ 产物 manifest **16 230 / added=0 removed=0 changed=0** ＋ `dotnet build Script/Script.sln` **0 Warning / 0 Error** ＋ 日志 `LogUnrealCSharp: Error` = 0 |
| **提交级补丁** | `Saved/Retest/fix-B28-fl014-cs1040-df487a6d.patch` —— **2 972 B** / SHA256 `896F48035973DD74CE984150AC6C66B0D68632897C1CE670FA9D3C2A842A1E7D` / **LF、无 BOM** / `git apply --reverse --check` **exit 0** |
| **探针** | 全部临时装置 **整块还原**（`TEMP-B28` / `UCS_B28_PROBE` grep = **0**） |

---

## 1. `F-FL-014` 使用面 vs 受益面普查（**先普查后定施工面**）

审计原文（[`02-…/07`](../02-UnrealCSharpCore核心/07-函数库宏与模板Trait.md):916-984）列了三个弱点与三个使用点，并建议"**先做使用面 vs 受益面普查**"。普查结果（源码 = 本轮工作树；引擎 = **5.6 / 5.0 本机源码 ＋ `D:\file\UnrealEngine\Release` 仓库的 5.0.0–5.8.3 全 32 个 release tag 逐版核验**）：

| # | 使用点（改前） | 现状（代码事实） | 引擎侧公开替代 | 结论 |
|---|---|---|---|---|
| ① | `FCSharpEnvironment.cpp:16` ＋ `:429`（**Runtime / Shipping 也编入**） | 读 `UObjectBase::ObjectFlags`，表达式 `… & ~RF_AllFlags`（"越界位"防御腿） | **有，且 5.0.0–5.8.3 全版本都有** —— `FORCEINLINE EObjectFlags GetFlags() const`（5.6 `UObjectBase.h:255`／5.0 `:171`／5.8.3 `:278`，**32 个 tag 全为 `public:`**、无 `UE_DEPRECATED`） | ✅ **可换**（本轮已换，见 §2） |
| ② | `FDynamicClassGenerator.cpp:22` ＋ `:939`（**无 `WITH_EDITOR` 门控**，Runtime 模块） | 写 `FObjectInitializer::bIsDeferredInitializer = true` | **无** —— 该成员在 `UObjectGlobals.h:1681`，夹在 `private:`(1585) 与 `public:`(1716) 之间，**全文件无访问器**（`grep DeferredInitializer` 仅 1 处命中） | ❌ **必须保留**（5.6 无公开替代） |
| ③ | `FGameplayTagGenerator.cpp:13` ＋ `:273`（Editor 模块） | `#if UE_F_GAMEPLAY_TAG_NODE_GET_DEV_COMMENT` → **公开** `GetDevComment()`；`#else` → 私有访问 | **有** —— `GameplayTagsManager.h:269` `FString GetDevComment() const`（public，`WITH_EDITORONLY_DATA`） | 🟡 **5.6 上不参与编译**（门控 = `UEVersion.h:170` `UE_VERSION_START(5, 6, 0)`）；`#else` 是 **UE < 5.6 的兼容位**，**不能删** |

**另两项普查结论**：

| 项 | 读数 |
|---|---|
| 宏定义数 | **5**（`AccessPrivateMacro.h:6/13/20/27/34`）；其中 **4 个零使用**（`STATIC_PROPERTY` / `MEMBER_FUNCTION` / `CONST_MEMBER_FUNCTION` / `STATIC_FUNCTION`）—— 与 `F-DEAD-019` 的"保持现状"裁定一致，本轮**不删**、只登记 |
| 调用点 / 消费点 | 改前 **3 / 3** ⇒ 改后 **2 / 2**；**UE 5.6 上参与编译的 = 1 / 1**（②） |
| 插件外使用者 | **0** —— 工程 `Source/`、`Script/` 与插件 `Script/` 全树 grep 五个宏名与 `TAccessPrivate<` **零命中**（⇒ 改名/替换不会动到仓内下游；但头文件在 `Public/CoreMacro/`，**对外契约仍属半公开面**） |

### 弱点处置（三条，逐条给"做/不做"的依据）

| 弱点 | 本轮处置 | 依据 |
|---|---|---|
| **① 全局类型名按短名拼接**（`Class##_##PropertyName` 文件作用域） | **登记**（不施工） | 冲突要求**同一 TU 内 ≥2 次展开且短名相同**；改后插件内每 TU 只有 **1** 次展开（②）⇒ **当前语料不可达**。真修法 = 把类型身份从"短名字符串拼接"改为"**按类类型键化**"（`TAccessPrivateTag<Class>` 主模板 ＋ 逐使用点显式特化），属**公开模板契约变更**，与"零新增公开面 / 改动面克制"的施工纪律相悖 ⇒ 登记触发条件（出现第二个使用点，或下游项目引用该宏）。⚠️ 报告原建议（"加命名空间/前缀"）**只消除与引擎/第三方全局名的撞名**，对"同名类"这一半**无效**（固定命名空间下仍同名），且本仓**手写 C++ 零 `namespace` 先例**（B-27 返工轮已明文确立） |
| **② 依赖非标准访问控制放宽** | **登记**（不施工） | 需要 **MSVC / Clang / GCC 编译器矩阵**实测才能判定；本机只有 MSVC（`14.51.36231`）＋ Android NDK clang（可编 Android 目标，但**该宏在 Android 上是否实例化**未构造）。审计原文置信度本就是 **中**（未验证假设） |
| **③ 依赖动态初始化** | **登记**（不施工） | 三个读点**都在函数体内**（运行期），现在不触发；审计建议的 `check(TAccessPrivate<T>::Value != nullptr)` **撞 Q2 口径**（断言不在白名单内），且在 **Shipping 下 `check` 本就编译为空**（而 ① 正是 Shipping 面）；"懒加载/返回引用包装"会**改公开模板形态** ⇒ 同上，登记触发条件（出现"静态初始化期读取"的新代码） |

> ⇒ **`F-FL-014` 的宏层面本体收敛为"登记"**，本轮只落地**受益面确定且可证等价**的那一处（①）。这与 `F-ABI-012`（前提回核后收窄为 P3/不施工）、`F-MACRO-003`（量化后降级 P3/本体不施工）同型 ⇒ **台账按"部分收口、降级 P3、事实与编号保留"归档**。

---

## 2. `F-FL-014` 施工：使用点 ① 换公开 API ＋ 等价性证据

**改动（`FCSharpEnvironment.cpp`，唯一施工面）**

```diff
-#include "CoreMacro/AccessPrivateMacro.h"
-ACCESS_PRIVATE_MEMBER_PROPERTY(UObjectBase, ObjectFlags, EObjectFlags)
 void FCSharpEnvironment::OnAsyncLoadingFlushUpdate()          // :398
-			if (Object->*TAccessPrivate<UObjectBase_ObjectFlags>::Value & ~RF_AllFlags ||
+			if (Object->GetFlags() & ~RF_AllFlags ||          // :426
```

**等价性证据（三层，均实测/逐行）**

| # | 判据 | 读数 |
|---|---|---|
| 1 | **引擎侧逐行**：`GetFlags()` 读的就是同一个成员 | 5.6 `UObjectBase.h:255-260` → `(EObjectFlags)GetFlagsInternal()`，`:330-342` = `FPlatformAtomics::AtomicRead_Relaxed((int32*)&ObjectFlags)`；**5.0 `:171-175`** → `checkfSlow(…); return ObjectFlags;`（**直接返回同一成员**） |
| 1b | **跨版本全量核验（插件最低支持 5.0，最高 5.8）**：`D:\file\UnrealEngine\Release`（= **UE 5.8.2 源码仓**，`Build.version` `MajorVersion 5 / MinorVersion 8 / PatchVersion 2`）的 git 仓库按 tag 逐版取 `Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectBase.h` | **5.0.0 / 5.0.1 / 5.0.2 / 5.0.3 / 5.1.0 / 5.1.1 / 5.2.0 / 5.2.1 / 5.3.0 / 5.3.1 / 5.3.2 / 5.4.0 / 5.4.1 / 5.4.2 / 5.4.3 / 5.4.4 / 5.5.0 / 5.5.1 / 5.5.2 / 5.5.3 / 5.5.4 / 5.6.0 / 5.6.1 / 5.7.0 / 5.7.1 / 5.7.2 / 5.7.3 / 5.7.4 / 5.8.0 / 5.8.1 / 5.8.2 / 5.8.3** 共 **32 个 release tag**：**全部存在 `FORCEINLINE EObjectFlags GetFlags() const`**、**全部位于 `public:` 段**、**全部无 `UE_DEPRECATED`**（跨 32 tag 名称/签名/可见性完全一致；行号 `171 → 188 → 199 → 193 → 207 → 220 → 255 → 285 → 278`）。**实现只有两种形态、位语义相同**：**5.0.0–5.2.1（8 tag）** = `checkfSlow((ObjectFlags & ~RF_AllFlags) == 0, …); return ObjectFlags;`（**直接读同一成员**）；**5.3.0–5.8.3（24 tag）** = `EObjectFlags Flags = (EObjectFlags)GetFlagsInternal(); checkfSlow(…); return Flags;`，其中 `GetFlagsInternal()` = 对**同一 `ObjectFlags` 成员**的 relaxed 原子读（5.3.0 `:238-243` `FPlatformAtomics::AtomicRead_Relaxed((int32*)&ObjectFlags)`；**5.8.3 `:366-377`** 已改为 `((std::atomic<int32>*)&ObjectFlags)->load(std::memory_order_relaxed)`，**取到的仍是同一 32 位**）⇒ **全支持区间内替换为位级同值，无需 `UE_F_*` 版本门控** |
| 2 | **路径可达性 + 该腿恒假（探针，`-game` 套件会话）** | `OnAsyncLoadingFlushUpdate` 被调用 **18 次**（对象非空），累计对象 **520** 个（最大一次 `objects=445 pending=439`），**`invalid-flags = 0` 全部 18 次** ⇒ `ObjectFlags & ~RF_AllFlags` 在真实会话中**从未为真**，替换零行为差异 |
| 3 | **套件 / 产物** | 套件 **1931 / 0 ×2**、CSV SHA `B429F3EE…`（与基线**同 SHA**）；产物 manifest **16 230 / 0 差异** |

> ⚠️ **一处必须写清的前提差异**：`GetFlags()` 自带 `checkfSlow((Flags & ~RF_AllFlags) == 0, …)` —— 即"该腿为真"这一情形**引擎自己也认定不该发生**。因此若真有越界位，Development/Editor 下会先触发引擎的 `checkfSlow`，而**不再是**插件原有的"跳过本轮绑定"分支（Shipping 下 `checkfSlow` 编译为空 ⇒ 行为与改前完全一致）。属**防御面的语义升级**，与审计建议"优先用引擎公开 API"一致，登记为残留面（§7）。

---

## 3. `F-CS1-040` 前提更正（三条，全部代码事实）

审计原文（[`06-…/01`](../06-CSharp运行时与脚本/01-Interop桥接与程序集加载.md):1496-1577）的核心后果句是"**每一个** C#→C++ 桥接调用都变成 `((delegate*)0)(args)` → **进程立即终止**，且**不产生任何日志**"。逐条回核：

| # | 原文断言 | 当前事实（HEAD `df487a6d`） | 判定 |
|---|---|---|---|
| 1 | 查找失败 ⇒ **调用地址 0** ⇒ 进程立即终止 | 生成器模板**早带守卫**：`Script/SourceGenerator/UnrealTypeSourceGenerator.cs:1197` `if (MethodPointer != nint.Zero)`（`:1198-1200` 才落调用）＋ `:1184` 非 void 分支 `defaultStatement` = `return default;`。**引入提交 = `90f239f4`**（"Method Pointer Guard …"，2026-10-08，晚于审计基线 `1b29b6ed`） | 🔴 **已消解**：真实后果 = **静默 no-op（void）／`default` 返回值（非 void）** |
| 2 | 失败**无任何日志** | `-game` 实测（§5 的 P0 会话）：C# 侧仍然经**既有日志桥**打出 `LogUnrealCSharp: Error:` ＋ `Unhandled Exception:` ＋ `System.NullReferenceException`（日志行 843-845）；该桥 = `LogBridge`→`FScriptLog.cpp` 的 `UE_LOG(LogUnrealCSharp, Error, …)`（`64` §2.1 已证） | 🟡 **部分成立**：桥接层"有日志"；**"注册整体失败"本身仍无任何信号**（改动前域报健康） |
| 3 | 键 `"Namespace.Class::Method"` 两侧拼接、零校验 | `UnrealTypeSourceGenerator.cs:1179` 生成 key；`MethodBridge.cs:14/16` 两张 `Ordinal` 字典；**新事实**：`:148-160` 的 `GetMethod` 现在**同时核对参数计数**（`StringToParamCount` ＋ `UnknownParamCount = -1`，提交 `1551383b`）⇒ 键或计数任一不匹配都静默归 0 | ✅ **成立且更严** |

**"契约漂移"实测（人为把 `BindingMacro.h:7` 的 `"::%s"` 换成 `"#%s"` 的等价物 = 把桥函数名改名，见 §5）**：套件**不是**安静通过，而是**早期即崩**在一个非 void 桥接调用上（`GetWorld()` 返回 null ⇒ 后续 NRE），最终落盘 **只有表头**的 CSV（1 行）—— 而 `run-batch.ps1` 只看"文件稳定"就归档为一次运行，**不会**判红。⇒ 结论：**契约漂移在套件里是"硬红但形态隐蔽"**（无 CSV / CSV 只有表头），**没有构建期契约测试**（登记，§7）。

---

## 4. `F-CS1-040` 施工：注册整体失败 → 既有状态通道 fail-closed

**落点（Q2 白名单内的唯一可落点）**

| 候选落点 | 判定 |
|---|---|
| C# 侧 `RegisterBinding` 末尾 `Count == 0` 自检 | ❌ **无出口**：该方法是 `[UnmanagedCallersOnly] void`（`MethodBridge.cs:18-19`），无返回通道；`GetMethod` 的返回值域 `0` 已被"未找到"占用、`-1` 已被 `ExceptionHandle` 占用（B-14）⇒ 表达失败只能"开口"（新增日志/返回通道），**Q2 不解锁** |
| C++ 侧共享 `RegisterBinding()` 的失败半边 → **既有状态通道** | ✅ **采用**：`bIsInitialized = false` ⇒ `IsInitialized()`（`FScriptDomainImpl.inl:534`）⇒ `FDomain::IsLoadSucceed()`（`FDomain.cpp:105-113`）⇒ **既有消费者**：`FDynamicRegistry.cpp:60`（动态类注册）、`FClassRegistry.cpp:265`（C# 包装对象构造路径）⇒ fail-closed，**零新增面**；形态 = 轮 A `F-DOM-004`（"删掉说谎的那一行 ⇒ fail-closed"）同一范式 |

**改动（`FScriptDomainImpl.inl` ＋ 两个 private 声明）**

```diff
-void SCRIPT_DOMAIN_TYPE::RegisterBinding() const        // :610（`const` 面：两个 private 声明同步去 const）
+void SCRIPT_DOMAIN_TYPE::RegisterBinding()
 	{
 		…
 		MethodBridgeRegisterBindingFn(MethodNames.GetData(), …);   // :644
 	}
+	else
+	{
+		bIsInitialized = false;                            // :649
+	}
```

**登记面（本轮**不**做的半边，逐条给依据）**

| 半边 | 处置 | 依据 |
|---|---|---|
| `InLength == 0`（名表为空） | **登记** | 三会话探针实测**名表恒为 2370**（`-game` 套件 ×2 ＋ `-run=GeneratorScriptCode -Platform=Windows`）⇒ **实测会话中不出现**；且"空注册表"在别的会话形状（打包版 / 无绑定工程 / 生成前）**语义未取证**，翻转状态可能改动行为（含产物路径）⇒ 需单独裁定与构造手段 |
| **LeanCLR** 半边 | **登记** | LeanCLR **不走**共享实现（`FLeanCLRDomain.cpp:107` 定义 `SCRIPT_DOMAIN_CUSTOM_REGISTER_BINDING`），有自有 `RegisterBinding()`（`:869`，P/Invoke 注册，与 MethodBridge 无关）；其 `bIsInitialized = true` 在 **`:86`**、而自有 `RegisterBinding()` 在 **`:57`** ⇒ 即便照抄也会**被随后覆盖**，须改初始化顺序 ⇒ 跨后端改动，登记 |
| 消费点"契约漂移"守卫 | **不做**（已由 `90f239f4` 完成） | 见 §3-#1 |
| 构建期契约测试 | **登记** | 度量结论 = 漂移下套件**硬红但形态隐蔽**（§3）；要"构建期"判据需枚举 `FBinding` 名集与生成 key 集做差集，属**新增装置**（且 C# 侧无既有调用先例：全 `Script` 树 `MethodBridge.GetMethod` 只出现在生成器模板里） |

**代价（必须写清的副作用，均已实测）**

1. **`FScriptDomainScope` 会重试初始化**：`FScriptDomainScope.h:25-33` —— `if (!IsInitialized()) Initialize();` 然后 `if (IsInitialized()) InFunction();` ⇒ fail-closed 之后，(a) 作用域包裹的托管工作**不再执行**（fail-closed 的本意），(b) 后续每次进入该作用域都会**再试一次初始化**。P1 会话实测：`RegisterBinding()` 共 **2 次**（首启 1 ＋ 重试 1），**未出现无限重试**。
2. **`Deinitialize()` 会早退**：`FScriptDomainFactory::Destroy` 与两个后端的 `Deinitialize()` 都以 `IsInitialized()` 为闸 ⇒ 该状态下**不卸载托管程序集**（会话已不可用，进程退出即回收）。属"分开的失败位"缺失所致；新增独立失败位 = **新增取值域**（开口）⇒ 不做。
3. 该状态**只在注册整体失败时出现**（正常会话读数与基线逐条相同：套件同 SHA、manifest 0 差异）。

---

## 5. A/B 判据：人为构造"桥函数指针为 `nullptr`"

**装置**（全部临时、已整块还原）：把 `CoreMacro/FunctionMacro.h:123` 的 `FUNCTION_METHOD_BRIDGE_REGISTER_BINDING` 由 `TEXT("RegisterBinding")` 改为 `TEXT("RegisterBindingX")` ⇒ `LOAD_ASSEMBLY_AND_GET_FUNCTION_POINTER`（`FCoreCLRDomain.cpp:20-27`）按名查不到 ⇒ 指针保持 `{}`（`nullptr`）。**该装置不需要改 C# 侧、不需要重发布程序集**。探针 = 临时计数器（`UCS_B28_PROBE` 环境门控；`RegisterBinding` 入口/失败半边 ＋ `IsInitialized()` 跃变）⇒ 已还原（grep `TEMP-B28` = 0）。

| 读数 | **P0 = 改前**（无 `else`） | **P1 = 改后**（fail-closed） |
|---|---|---|
| 探针 `register-binding fn=` | **0**（指针为空，与预期一致） | **0** |
| 探针 `is-initialized` 跃变 | **0 → 1**（`FScriptDomainScope` 起手查询得 false，域初始化后置真 ⇒ **整场保持真**） | **0（全程未变真）** |
| `FDomain::IsLoadSucceed()` | **true（整场）** | **false（整场）** ⇒ `FDynamicRegistry`/`FClassRegistry` 的 C# 路径**不再执行** |
| `RegisterBinding()` 调用次数 | 1 | **2**（`FScriptDomainScope` 重试 1 次） |
| 套件 CSV | **只有表头**（1 行；1931 条用例一条未写） | **只有表头**（同） |
| C# 侧日志 | `LogUnrealCSharp: Error:` ＋ `Unhandled Exception: NullReferenceException`（843-845） | **同** |
| 产物 | 未变（注册失败不产出） | 未变 |

⇒ **改前的缺陷形态 = "域报健康 ＋ 所有 C#→C++ 调用静默失效 ＋ 归档一张只有表头的 CSV"**（正是死代码轮登记的"表头 CSV 不得当成功引用"同型）；**改后 = 同一故障下域状态不再说谎，既有下游据此停手**。这正是 Q2 "失败一律静默收敛到**既有**返回值（这里是既有状态通道）"的形态。

---

## 6. 交付态判据（全部在**提交树等价态** = 工作树 ＋ 探针已还原）

| 判据 | 命令 | 读数 |
|---|---|---|
| Windows UBT | `Build.bat UnrealCSharpTestEditor Win64 Development -Project=… -WaitMutex` | **Succeeded**；`[7/7] WriteMetadata …`；**52.28 s**（日志 `Saved/Retest/b28-final-ubt.log:49-54`） |
| Android UBT（控制项） | `Build.bat UnrealCSharpTest Android Development -Project=… -WaitMutex` | **Succeeded**；`BUILD SUCCESSFUL in 26s` ＋ `COMPLETED MAKE APK`；**82.77 s**（`b28-final-android-ubt.log:303-313`，NDK `r25b / clang 14.0.6`）；⚠️ 该目标的 `Intermediate/Build/Android/…/Definitions.UnrealCSharpCore.h` = **`WITH_CORECLR 0 / WITH_MONO 1 / WITH_LEANCLR 0`**（`Config/DefaultUnrealCSharpSetting.ini` `AndroidScriptDomainType=Mono`）⇒ **Mono 路径（含共享 `RegisterBinding()` 的 `else` 半边与 `FMonoDomain.h` 的去 `const`）已随本目标编译通过** |
| 套件 ×2 | `run-batch.ps1 -Tag b28-final -Runs 2`（`-game -nullrhi -nosound`） | **1932 行 = 表头 ＋ 1931 条全部 `true`**（failed = 0）；CSV SHA **`B429F3EE013B8C73FADEAF12E9BFA13C6B28C4FFE280312DE64F6843EB4386F8`**（两份同 SHA，且与 B-24/B-25/B-26/B-27 基线**同 SHA**） |
| 产物 manifest | `g8-manifest.ps1 -Tag b28-pre` / `-Tag b28-after` ＋ `g8-manifest-diff.ps1` | `before=16230 after=16230` / `added=0 removed=0 changed=0` |
| C# 侧（控制） | `dotnet build Script/Script.sln` | **0 Warning / 0 Error**（2.87 s） |
| **跨版本 API 核验（支持区间）** | `git show <tag>:Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectBase.h`（`D:\file\UnrealEngine\Release`，32 个 release tag） | **5.0.0–5.8.3 全部**：`public FORCEINLINE EObjectFlags GetFlags() const`、无 `UE_DEPRECATED`；实现两种形态（8 tag 直读成员／24 tag 经 `GetFlagsInternal()` 原子读同一成员）⇒ **无需 `UE_F_*` 门控** |
| 生成器副产物（控制） | `run-b26-commandlet.ps1 -Tag b28-cmd-platform -WithPlatform` | `proxy after cmd : UE=16155 (was 16155)  Game=75 (was 75)` / `proxy diff : removed=0 added=0 changed=0` / `restore : stillMissing=0 stillDifferent=0` |
| 日志/崩溃 | `LogUnrealCSharp: Error` / `ensure` / `assert` / `Fatal` / `Unhandled Exception` | **0**（健康会话） |
| 装置还原 | `TEMP-B28` ｜ `UCS_B28_PROBE` 全树 grep | **0 / 0** |
| 补丁 | `git diff --output=…`（仅 `Source/UnrealCSharp`、`Source/UnrealCSharpCore`） | **2 972 B** / SHA256 `896F4803…` / LF / 无 BOM / reverse-check **exit 0** |

---

## 7. 未取得与残留面（如实登记）

1. **编译器矩阵（弱点 ②）**：MSVC 之外的 Clang/GCC 对"模板实参中的私有成员地址"的接受度**未取得**（本机无矩阵）；
2. **同 TU 双展开的冲突复现（弱点 ①）**：未构造（需两个短名相同的类同 TU 展开；且改后插件内每 TU 只剩 1 次展开 ⇒ 属"删掉最后一个使用点才彻底消解"的长尾）；
3. **Shipping 配置下的实跑**：本轮 UBT 只编 Development（`Win64 Development`），**未编 Shipping**；`checkfSlow` 在 Shipping 下为空属引擎宏知识（未实测编译产物）；
4. **`GetFlags()` 的 `checkfSlow` 语义升级**：§2 已说明；"越界位为真"这一情形**不可构造**（需故意写脏 `ObjectFlags`）⇒ 无实测读数；
5. **`InLength == 0` 半边**：三会话名表恒 2370，**未能构造**"名表为 0"的会话 ⇒ 该半边保持登记（§4）；
6. **Mono 后端实跑**：本轮 A/B 与交付态判据都在 **CoreCLR** 下取得；**编译面已覆盖**（Android 目标 = `WITH_MONO 1`，见 §6），但 Mono 的**运行读数**未取得（未切 `WindowsScriptDomainType=Mono` 跑套件）；
7. **`F-CS1-040` 的"生产版静默"**：`-game` 会话里漂移是"硬红"，打包版/玩家机上的形态**未实测**（`void` 桥接调用静默 no-op 是从生成产物形态推出的）。

---

## 8. 复用价值（方法论）

1. **"注册/初始化失败"类条目，先问"有没有既有取值域"，再问"谁在消费它"** —— 本条可施工的原因是 `IsInitialized()` **已经被两个既有消费者当成闸门**（`FDynamicRegistry` / `FClassRegistry`），所以"把状态写对"就等于"接入既有失败处理"，不需要新增任何面；若没有消费者，同样的写法只会制造一个没人看的标志位。
2. **审计条目的"后果"必须先构造一次再引用** —— `F-CS1-040` 被引用了多轮的"跳转 0 崩溃"，在 `90f239f4` 之后已不成立；本轮用"改桥函数名"这一 **1 行装置**把真实后果（表头 CSV ＋ NRE 日志）当场取到。**代价最低的构造方式通常不是改被测对象，而是改它的输入契约**（这里是按名解析的名字）。
3. **A/B 的差值要落在"既有可观测面"上** —— 改前/改后套件 CSV 同为"只有表头"，差异只在**域状态**；若只盯 CSV 会得出"没有差异"的错误结论。⇒ 判据要选"**下游会据以行动**"的那个量。
4. **普查（使用面 vs 受益面）应当允许把条目"降级并出列"** —— `F-FL-014` 的三条弱点里，两条卡在"需要举证装置"、一条卡在"修法撞口径"，唯一确定受益的是那**一处公开 API 替换**；把其余收敛为"登记 ＋ 触发条件"，比保留一个"未修"标记更接近事实。
5. **改公开头文件里的宏之前，先量"仓内下游 + 仓外可能"** —— 本仓库内 grep = 0，才使"删调用点／换公开 API"零风险；同时把"Public 头 ⇒ 对外半公开"写进结论，避免把"没搜到"当成"没人用"。
6. **改引擎 API 之前，先把"支持区间"按 tag 量一遍** —— 本插件的支持区间是 **UE 5.0–5.8**，而"本机编得过"只证明**一个**版本。`D:\file\UnrealEngine\Release` 是带 release tag 的引擎源码仓 ⇒ `git show <tag>:<path>` 可以**逐版取头文件**：本轮 32 个 tag 的 `GetFlags()` **签名/可见性完全一致**，实现只有两种形态且位语义相同 ⇒ 结论从"5.6 能编"升级为"**全支持区间等价、无需门控**"。若不查这一步，正确做法会退化成"加一个 `UE_F_*` 门控到处兜底"（多一处长期负担）。

---

## 9. 台账回写与锚点

| 位置 | 回写内容 |
|---|---|
| [`09-…/00`](../09-问题清单/00-推荐优先修复清单.md) §0.0 T2 表 | **第 26 行 `F-FL-014`** → "🟡 轮 B-28：使用面 3 → 2（5.6 活面 1），使用点 ① 已换公开 `GetFlags()`；弱点 1/2/3 与 ② 半边登记" ⇒ **P2 → P3（收窄出列）**；**第 28 行 `F-CS1-040`** → "🟡 轮 B-28：注册整体失败半边已修（既有状态通道 fail-closed）；崩溃半边前提更正；`InLength == 0` 与 LeanCLR 半边登记" ⇒ **T2 出列**；并新增"轮 B-28"更新注（含 T2 余量与干净开放面变化） |
| [`09-…/01`](../09-问题清单/01-全局问题总表.md) | `F-FL-014`（`#L377`）与 `F-CS1-040`（`#L439`）两行的「复核结论」列就地更新（含 `90f239f4` 前提更正与读数） |
| [`09-…/05`](../09-问题清单/05-已完成（P2与P3）.md) | 新增 **§28 轮 B-28 收口**（逐条处置 ＋ 交付态判据 ＋ 下一顺位） |
| [`README.md`](../README.md) | 轮次表新增 B-28 行（改动面、判据、补丁 SHA256） |
| [`INDEX.md`](../INDEX.md) | 新增本轮"另（2026-10-16，轮 B-28 …）"条目并回指本报告 |
| 本报告 | [`08-…/68`](68-轮B28收口-注册整体失败fail-closed与AccessPrivate公开API替换（2026-10-16）.md) |
