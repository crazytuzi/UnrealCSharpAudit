# 轮 B-28 收口：`F-CS1-040` 注册整体失败 → 既有状态通道 fail-closed ＋ `F-FL-014` 使用面普查（使用点 ① 不可替换 ⇒ **经维护者裁定删除恒假腿**；含 1 处前提更正）

> 🔴 **修订记录**：**v1**（首版）结论"`F-FL-014` 使用点 ① 已换公开 `UObjectBase::GetFlags()`、P2 → P3"**已被推翻**（维护者复核指出：那条裸读是**为了绕开 `GetFlags()` 的 `checkfSlow`** 才这么写的）⇒ **v2**：据 `git log -S` ＋ 引擎源码给出三层证据（§2.1–§2.4），改为"**① 不可替换**"并撤回替换，新增"该腿在 5.0–5.8.3 **恒假**"的事实与一项待裁定；⇒ **v3（本版）**：维护者裁定"**那条恒假腿删掉吧**" ⇒ **已施工删除**（§2.5；A/B 探针读数**逐行相同**），私有访问 **使用面 3 → 2**、**UE 5.6 活面 2 → 1**。`F-CS1-040` 部分（§3–§5）**未受影响**，仅按最终二进制重取了一遍读数。作废补丁 `fix-B28-fl014-cs1040-df487a6d.patch`（2 972 B / `896F4803…`）不再交付。
>
> **来源**：维护者指定 —— 轮 B-28 = [`09-…/05`](../09-问题清单/05-已完成（P2与P3）.md) §27 与 [`08-…/67`](67-轮B27收口-源生成器不抛半边与CodeAnalysis构建闸门（2026-10-15）.md) §263 共同给出的**下一顺位**两条：`F-FL-014`（P2 · S=3.08 · T2#26）＋ `F-CS1-040`（P2 · S=3.08 · T2#28），同轮并行；`F-FL-014` 的处置经维护者**两轮裁定**（先"不可替换（撤回替换）"、后"删除恒假腿"）。
> 🟢 **提交锚点：`a9e10378636504f55e9bee4f376d944c918d16e0`**（父 **`df487a6d132f451ff619014d3d0913d7ee9e4ce5`**，作者 `crazytuzi <liuxiangcode@qq.com>`，**2026-10-16 15:09:56 +0800**，subject `ScriptDomain Register Validation && FCSharpEnvironment Dead Code Cleanup && Compatible with 5.0`；**本轮 4 文件 `+8/−8`**：`FCSharpEnvironment.cpp` `+1/−5`、`FScriptDomainImpl.inl` `+5/−1`、`FCoreCLRDomain.h`／`FMonoDomain.h` 各 `+1/−1`）。
> **提交树复验**：`git diff a9e10378 -- Source/UnrealCSharp Source/UnrealCSharpCore` = **空**（提交内容 = 施工期工作树，**维护者未做任何调整**），且**提交级补丁与施工期补丁逐字节相同**（`[string]::Equals(..., Ordinal)` = True）⇒ 施工期已测读数即提交树读数；另在提交树上重跑：Windows UBT **`Target is up to date`**（11.92 s）＋ 套件 **1931 / 0 ×2**、CSV **同 SHA `B429F3EE…`**（`Saved/Retest/out/b28d-commit/`）。
> **插件基线**：**`df487a6d`**（B-27 交付提交）→ 本轮工作树 → 提交 `a9e10378`。
> **环境**：UE 5.6.0（`D:\file\UnrealEngine\5.6`）· 后端 **CoreCLR**（`-game` 套件）＋ **Android（NDK r25b / clang 14.0.6）控制项**；**跨版本核验** = `D:\file\UnrealEngine\Release`（**UE 5.8.2 源码仓，带 5.0.0–5.8.3 全 32 个 release tag**）＋ 本机 `D:\file\UnrealEngine\5.0`。
> **口径**：遵守 `README.md` `#L143` 与 [`08-…/66`](66-待决策页-三问（Q1四路线·Q2口径例外·Q3失败判据）（2026-10-15）.md) 的 Q2 裁定 —— **不新增注释 / 日志 / `throw` / `try-catch` / `ensure` / `check` / 公开 API**；失败一律静默收敛到**既有取值域**。

---

## 0. 摘要

| 项 | 读数 |
|---|---|
| **改动面** | **4 文件 `+8/−8`**（`FScriptDomainImpl.inl` `+5/−1`：`RegisterBinding()` 去 `const` ＋ 失败半边；`FCoreCLRDomain.h`／`FMonoDomain.h` 各 `+1/−1`：private 声明去 `const`；**`FCSharpEnvironment.cpp` `+1/−5`：删除恒假腿（该 `if` 首行改写）＋ 其宏展开与 include**）；C# 0 / 生成器 0 / 生成产物 0 / 协议面 0 / 导出签名 0 / **公开 API 0** / 注释·日志·异常·断言 **全 0**。⚠️ v1 的"使用点 ① 换 `GetFlags()`"**已撤回**（替换版补丁 2 972 B / `896F4803…` 作废，不再交付） |
| **对象** | `F-CS1-040`（06-CSharp运行时与脚本）＝**注册整体失败可检测**（施工）＋ **"跳转 0 崩溃"前提更正**；`F-FL-014`（02-UnrealCSharpCore核心）＝**使用面普查收口**：使用点 ① 经复核**不可替换**（v1 结论推翻）⇒ 经维护者裁定**删除恒假腿**（v3 施工），私有访问面收窄 |
| **F-FL-014 普查** | 5 个宏中 **4 个零使用**；宏调用点 **3 → 2**（删除 ①）、**UE 5.6 上参与编译的调用点 2 → 1**（②；③ 受 `UE_F_GAMEPLAY_TAG_NODE_GET_DEV_COMMENT = 5.6.0` 门控，5.6 上走公开 `GetDevComment()`）；**② 无公开替代**（`UObjectGlobals.h:1681` 在 private 段、全文件无访问器）⇒ 级别**维持 P2**（② 不可消除且同为 Shipping 面） |
| **🔴 本轮新增事实（F-FL-014 ①）** | 那条 `ObjectFlags & ~RF_AllFlags` 在 **5.0.0–5.8.3 全 32 个 release tag** 上**恒为 0** —— 因为 `RF_AllFlags = (EObjectFlags)0xffffffff`（**全版本**）⇒ 该腿**从不跳过任何对象**（既是"死条件"，也是"不可能触发 `GetFlags()` 的 `checkfSlow`"的原因）⇒ 本轮按裁定**删除**（§2.5） |
| **F-CS1-040 前提更正** | ① "契约漂移 ⇒ 全量调用**跳转 0 崩溃**"**已由 `90f239f4` 消解** —— 生成产物早带 `if (MethodPointer != nint.Zero)` 守卫（`UnrealTypeSourceGenerator.cs:1197`）＋ 非 void `return default`（`:1184`）；真实后果 = **静默 no-op**（`-game` 实测见 §3）；② `GetMethod` 现在**同时核对参数计数**（`MethodBridge.cs:148-160`）；③ 注册整体失败**在改前完全静默**（域报"健康"，harness 归档一张**只有表头**的 CSV） |
| **F-CS1-040 施工** | 共享 `RegisterBinding()` 的"桥函数指针为 `nullptr`" ⇒ **既有状态通道 fail-closed**（`bIsInitialized = false` ⇒ `IsInitialized()` / `FDomain::IsLoadSucceed()`），形态同轮 A `F-DOM-004`；`const` 面 3 处（两个 private 声明 ＋ 共享定义） |
| **A/B（人为把 `FUNCTION_METHOD_BRIDGE_REGISTER_BINDING` 改名 ⇒ 指针 `nullptr`；**撤回后重跑**）** | **改前（P0）**：域**全程报健康**（`is-initialized 0→1`）、`RegisterBinding` 1 次、套件落**只有表头**的 CSV；**改后（P1，3 次）**：域**全程 fail-closed**（`is-initialized` 从未变真）、`RegisterBinding` **2 次**（`FScriptDomainScope` 重试 1 次）、套件同样只有表头 CSV（2/3 次）＋ **1 次进程以已登记的环境崩溃指纹 `coreclr+0x9603a` 退出**（见 §5.3） |
| **交付态判据** | Windows UBT **Succeeded**（4 actions / 46.92 s）＋ **Android UBT Succeeded**（`BUILD SUCCESSFUL in 30s` / 94.43 s，控制项）＋ 套件 **1931 / 0 ×2**、CSV SHA **`B429F3EE…`（与 B-24～B-27 同 SHA）** ＋ 产物 manifest **16 230 / added=0 removed=0 changed=0** ＋ 健康会话错误/崩溃全 0、`Saved/Crashes` 今日新增 0 |
| **提交级补丁（由提交树 `a9e10378` 导出）** | `Saved/Retest/fix-B28-cs1040-fl014-deadleg.patch` —— **3 015 B** / SHA256 `CDEE4AE34FDE449336BD7DF2C1494E0200E12911C6FC7A7AB05CEDBB361583CC` / **LF、无 BOM** / `git apply --reverse --check` **exit 0**（与施工期 `…-df487a6d.patch` **逐字节相同**） |
| **探针** | 全部临时装置 **整块还原**（`TEMP-B28` / `UCS_B28_PROBE` grep = **0**） |

---

## 1. `F-FL-014` 使用面 vs 受益面普查（**先普查后定施工面**）

审计原文（[`02-…/07`](../02-UnrealCSharpCore核心/07-函数库宏与模板Trait.md):916-984）列了三个弱点与三个使用点，并建议"**先做使用面 vs 受益面普查**"。普查结果（源码 = 本轮工作树；引擎 = **5.6 / 5.0 本机源码 ＋ `D:\file\UnrealEngine\Release` 仓库的 5.0.0–5.8.3 全 32 个 release tag 逐版核验**）：

| # | 使用点（改前） | 现状（代码事实） | 引擎侧公开替代 | 结论 |
|---|---|---|---|---|
| ① | ~~`FCSharpEnvironment.cpp:16` ＋ `:429`（**Runtime / Shipping 也编入**）~~ **✅ 已删除（v3，维护者裁定）** | 裸读 `UObjectBase::ObjectFlags` —— 该条件里**唯一的免检查腿**（紧邻的 `:430` `HasAnyFlags` 内部就是 `GetFlags()`，`:431`/`:438` `HasAnyInternalFlags` 走 `IndexToObject()` 的硬 `check`）；**实测恒假**（`leg-true = 0` × 18 次调用） | **API 存在但不可用**：`GetFlags()`（5.0.0–5.8.3 全 32 个 release tag 都是 `public FORCEINLINE EObjectFlags GetFlags() const`、无 `UE_DEPRECATED`）**自带 `checkfSlow((Flags & ~RF_AllFlags) == 0, …)`**，而插件这条腿测的**正是同一个条件** | ❌ **不可替换**（v1 替换已撤回，见 §2.1–§2.4）⇒ **经裁定删除**（可证中性，§2.5） |
| ② | `FDynamicClassGenerator.cpp:22` ＋ `:939`（**无 `WITH_EDITOR` 门控**，Runtime 模块） | 写 `FObjectInitializer::bIsDeferredInitializer = true` | **无** —— 该成员在 `UObjectGlobals.h:1681`，夹在 `private:`(1585) 与 `public:`(1716) 之间，**全文件无访问器**（`grep DeferredInitializer` 仅 1 处命中） | ❌ **必须保留**（5.6 无公开替代） |
| ③ | `FGameplayTagGenerator.cpp:13` ＋ `:273`（Editor 模块） | `#if UE_F_GAMEPLAY_TAG_NODE_GET_DEV_COMMENT` → **公开** `GetDevComment()`；`#else` → 私有访问 | **有** —— `GameplayTagsManager.h:269` `FString GetDevComment() const`（public，`WITH_EDITORONLY_DATA`） | 🟡 **5.6 上不参与编译**（门控 = `UEVersion.h:170` `UE_VERSION_START(5, 6, 0)`）；`#else` 是 **UE < 5.6 的兼容位**，**不能删** |

**另两项普查结论**：

| 项 | 读数 |
|---|---|
| 宏定义数 | **5**（`AccessPrivateMacro.h:6/13/20/27/34`）；其中 **4 个零使用**（`STATIC_PROPERTY` / `MEMBER_FUNCTION` / `CONST_MEMBER_FUNCTION` / `STATIC_FUNCTION`）—— 与 `F-DEAD-019` 的"保持现状"裁定一致，本轮**不删**、只登记 |
| 调用点 / 消费点 | **3 → 2**（删除 ①，v3）；**UE 5.6 上参与编译的 = 2 → 1**（②；③ 走公开 getter） |
| 🔴 **这条腿在引擎侧能不能触发** | **不能**：`RF_AllFlags = (EObjectFlags)0xffffffff`（**5.0.0–5.8.3 全 32 个 release tag 一致**）⇒ `X & ~RF_AllFlags` **恒为 0** ⇒ 该腿**从不生效**（也正因如此，`GetFlags()` 里那句同掩码的 `checkfSlow` 同样永不触发）。运行期探针（18 次调用 / 520 个对象 / `invalid-flags`＝`leg-true` 全 0）与之一致 —— **此前把它读作"恰好为 0"，实为"必然为 0"** ⇒ **据此删除（§2.5）** |
| 插件外使用者 | **0** —— 工程 `Source/`、`Script/` 与插件 `Script/` 全树 grep 五个宏名与 `TAccessPrivate<` **零命中**（⇒ 改名/替换不会动到仓内下游；但头文件在 `Public/CoreMacro/`，**对外契约仍属半公开面**） |

### 弱点处置（三条，逐条给"做/不做"的依据）

| 弱点 | 本轮处置 | 依据 |
|---|---|---|
| **① 全局类型名按短名拼接**（`Class##_##PropertyName` 文件作用域） | **登记**（不施工） | 冲突要求**同一 TU 内 ≥2 次展开且短名相同**；插件内每个使用点各在**自己的 TU** 中展开一次（3 处 / 3 个 TU）⇒ **当前语料不可达**。真修法 = 把类型身份从"短名字符串拼接"改为"**按类类型键化**"（`TAccessPrivateTag<Class>` 主模板 ＋ 逐使用点显式特化），属**公开模板契约变更**，与"零新增公开面 / 改动面克制"的施工纪律相悖 ⇒ 登记触发条件（同一 TU 出现第二个使用点，或下游项目引用该宏）。⚠️ 报告原建议（"加命名空间/前缀"）**只消除与引擎/第三方全局名的撞名**，对"同名类"这一半**无效**（固定命名空间下仍同名），且本仓**手写 C++ 零 `namespace` 先例**（B-27 返工轮已明文确立） |
| **② 依赖非标准访问控制放宽** | **登记**（不施工） | 需要 **MSVC / Clang / GCC 编译器矩阵**实测才能判定；本机只有 MSVC（`14.51.36231`）＋ Android NDK clang（Android 目标已随本轮 UBT 编译，但**该宏在 Android 上是否实例化**未构造）。审计原文置信度本就是 **中**（未验证假设） |
| **③ 依赖动态初始化** | **登记**（不施工） | 三个读点**都在函数体内**（运行期），现在不触发；审计建议的 `check(TAccessPrivate<T>::Value != nullptr)` **撞 Q2 口径**（断言不在白名单内），且在 **Shipping 下 `check` 本就编译为空**（而 ① 正是 Shipping 面）；"懒加载/返回引用包装"会**改公开模板形态** ⇒ 同上，登记触发条件（出现"静态初始化期读取"的新代码） |
| **④（本轮新增）使用点 ① 的腿当前恒假** | ✅ **已删除（维护者裁定，v3）** | `RF_AllFlags = 0xffffffff`（全 32 tag）⇒ 该腿从不生效；`git log -S` 显示它由 `b38391bd`（#509）**有意引入**（同提交的真正修复是 `NotifyUObjectDeleted` 里把已删对象移出 `AsyncLoadingObjectArray`）。⇒ **不要**用 `GetFlags()`/`HasAnyFlags()` 替换（会把"安静跳过"变成"先断言"，见 §2.1–§2.4）；维护者裁定**删除该恒假腿**（可证行为中性，§2.5 已施工并取 A/B 读数） |

> ⇒ **`F-FL-014` 本轮 = 使用面普查收口 ＋ 删除一条恒假腿**：使用点 ① **不可替换**（v1 替换已撤回）⇒ 经维护者裁定**整条删除**（含其宏展开与 include）；② 无公开替代（**不可消除**）、③ 受版本门控且是 UE<5.6 兼容位（不能删）；弱点 1/2/3 登记。⇒ **级别维持 P2**（② 仍是 Shipping 面且无公开替代），台账按"**普查收口 ＋ 一处可证中性的删除**"归档（编号与全部事实保留）。⚠️ 本报告 v1 的结论（"① 已换公开 API、P2 → P3"）与 v2 的"0 代码改动"**均已被本版取代** —— 更正过程见 §2.1–§2.5。

---

## 2. `F-FL-014` 使用点 ①：为什么**不能**替换成公开 API（替换已撤回）

### 2.1 这条腿的来历 —— 提交标题就是意图

```powershell
cd Plugins\UnrealCSharp
git log --oneline -S '~RF_AllFlags' -- Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp
# → b38391bd #509 Fixed OnAsyncLoadingFlushUpdate checkfSlow
```

`b38391bd`（作者 crazytuzi，2025-06-07）的 diff 有两块：**①** 在 `OnAsyncLoadingFlushUpdate` 的条件里**新增**裸读腿
`Object->*TAccessPrivate<UObjectBase_ObjectFlags>::Value & ~RF_AllFlags ||`（此前该 `if` 的第一条腿就是 `Object->HasAnyFlags(RF_NeedPostLoad)`）；**②** 在 `NotifyUObjectDeleted` 里**把已删对象从 `AsyncLoadingObjectArray` 移除**。
⇒ 意图明确：**在"是否跳过这个对象"的判定里，用一条不可能触发引擎断言的读法**（裸读私有成员，不经任何 API）。

### 2.2 替换为什么不成立（两层；第二层是首版漏掉的）

1. **check 耦合**：`GetFlags()`（5.6 `UObjectBase.h:255-260`；5.0 `:171-175`；5.8.3 `:278-281`）自带
   `checkfSlow((Flags & ~RF_AllFlags) == 0, TEXT("%s flagged as RF_AllFlags"), *GetFName().ToString());`
   —— **测的正是插件这条腿的同一个条件**。若 `RF_AllFlags` 收窄为真正的子集掩码，该腿为真时会**先断言、后跳过**，而原语义是"安静跳过" ⇒ 语义方向相反的替换。
2. **而它今天两个都不触发（本轮新增事实）**：`RF_AllFlags = (EObjectFlags)0xffffffff` 在 **5.0.0–5.8.3 全 32 个 release tag** 上一致（`ObjectMacros.h`，按 tag 逐版核验）⇒ 引擎那句 `checkfSlow` **恒真、永不触发**；插件这条腿 `Flags & ~RF_AllFlags` **恒为 0、从不生效**（死条件）。
   ⇒ **撤回的理由**：① 它是作者**有意写的免检查读法**（意图层）；② 替换只在"掩码将来收窄"时才暴露风险（可预见性层）；③ 这条腿**今天根本不生效** ⇒ 替换的收益 **0**，而把语义从"裸读"改成"经检查"是净风险。
3. **旁证（该 `if` 里唯一免检查的腿）**：`HasAnyFlags()`（`UObjectBaseUtility.h:87-91`）实现就是 `checkSlow(...); return (GetFlags() & FlagsToCheck) != 0;`；`HasAnyInternalFlags()`（`:279-282`）走 `GUObjectArray.IndexToObject(InternalIndex)`，后者带**硬 `check(Index >= 0)`**（`UObjectArray.h:1035-1042`）。⇒ 只有这条裸读能"看一眼就走"。

**撤回的操作**：`git checkout -- Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp`（该文件回到 HEAD：`:6` include、`:16` 宏展开、`:429` 裸读原样）⇒ 替换版补丁 `fix-B28-fl014-cs1040-df487a6d.patch`（2 972 B / `896F4803…`）**作废**。
**核验（撤回后重取的读数）**：套件 **1931 / 0 ×2**、CSV SHA **`B429F3EE…`（与基线同 SHA）**；manifest **16 230 / 0 差异**；`TEMP-B28` grep = 0。

### 2.3 处置选项（v2 提交维护者 → **已于 v3 裁定：删除**，施工见 §2.5）

**是否删除这条恒假腿**（`FCSharpEnvironment.cpp:16` 的宏展开 ＋ `:429` 的第一条腿）？

| 面 | 内容 |
|---|---|
| **可证收益** | 删除后该文件的私有访问 hack **归零**（`F-FL-014` 使用面 **3 → 2**、UE 5.6 活面 **2 → 1**），且**行为中性可证**（掩码全 1 ⇒ 条件恒假，且探针 18 次 / 520 对象全 0 与之一致） |
| **代价/风险** | ① 删掉作者对 #509 的防御**意图**（同提交的有效修复"移出已删对象"**保留**，相邻三条腿**未动**）；② 若本意是"跳过 mirrored-garbage 对象"，正确形式不是掩码而是 `RF_MirroredGarbage`（**仅 5.4+ 存在**，需门控）或 `HasAnyInternalFlags(EInternalObjectFlags::Garbage)` —— 那是**行为变更**（会真的开始跳过对象），须先构造判据；③ Shipping 面不再有"越界位"检查（该检查也不生效） |
| **裁定结果** | 🟢 **维护者 2026-10-16 裁定：删除**（"那条恒假腿删掉吧"）⇒ 施工与 A/B 读数见 **§2.5**；未采用"换 `GetFlags()`"（§2.1–§2.2 已证其方向相反） |

### 2.4 首版错在哪（方法论留档）

首版只做了"**逐 tag 比对新旧读法的位值与可见性**"（那一层结论没错：32 tag 全有 `GetFlags()`、位级同值），但**没有回答"作者为什么要裸读"** —— 而 `git log -S` 一条命令就能看到提交标题写着 `Fixed … checkfSlow`。
⇒ 教训：**"引擎 API 存在且等价"与"被替换代码的意图"是两件事**；只做前者，会把"有意绕开检查"读成"陈旧的 hack"。

### 2.5 施工（v3）：按裁定删除恒假腿 ＋ 可证中性的 A/B

**维护者裁定（2026-10-16）**："**那条恒假腿删掉吧**"。删除内容（`FCSharpEnvironment.cpp`，`+1/−5`）：

```diff
-#include "CoreMacro/AccessPrivateMacro.h"
-ACCESS_PRIVATE_MEMBER_PROPERTY(UObjectBase, ObjectFlags, EObjectFlags)
 			auto Object = ObjectPtr.Get();
-			if (Object->*TAccessPrivate<UObjectBase_ObjectFlags>::Value & ~RF_AllFlags ||
-				Object->HasAnyFlags(RF_NeedPostLoad) ||
+			if (Object->HasAnyFlags(RF_NeedPostLoad) ||
```

**为什么可证中性（两层）**：① **静态**：`RF_AllFlags = (EObjectFlags)0xffffffff`（全 32 tag）⇒ `X & ~RF_AllFlags ≡ 0` ⇒ 被删的是条件的**常数假项**，`&&`/`||` 语义下删除不改变结果；② **实测（同一会话形态的改前/改后探针）**：

| 读数（`-game` 套件单趟，`UCS_B28_PROBE` 门控落盘） | **改前**（腿在） | **改后**（腿已删） |
|---|---|---|
| `OnAsyncLoadingFlushUpdate` 调用次数（对象非空） | **18** | **18** |
| 累计遍历对象 `objects` | **520** | **520** |
| 累计进入绑定 `pending` | **490** | **490** |
| 累计跳过 `skipped`（其余三条腿） | **30** | **30** |
| **该腿为真次数 `leg-true`** | **0（18 次全 0）** | ——（已删） |
| 套件 CSV | 1932 行 / 1931 true / SHA `B429F3EE…` | **同 SHA** |

⇒ **逐项相同**（18 / 520 / 490 / 30），且"腿为真"从未发生 —— 删除的**行为中性**同时由静态与运行期两侧成立。

**副作用面（如实）**：删除后 `FCSharpEnvironment.cpp` **不再包含任何 `ACCESS_PRIVATE_*` 用法**（该文件曾把 hack 编进 Shipping）；`#509` 的防御**意图**随之消失，但同提交里的**有效修复**（`NotifyUObjectDeleted` → `AsyncLoadingObjectArray.Remove`）**原样保留**，且相邻的 `HasAnyFlags(RF_NeedPostLoad)`／`HasAnyInternalFlags(...)` 三条腿**一字未动**。

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

| 读数 | **P0 = 改前**（无 `else`；撤回后重跑） | **P1 = 改后**（fail-closed；撤回后重跑 ×3） |
|---|---|---|
| 探针 `register-binding fn=` | **0**（指针为空，与预期一致） | **0** |
| 探针 `is-initialized` 跃变 | **0 → 1**（`FScriptDomainScope` 起手查询得 false，域初始化后置真 ⇒ **整场保持真**） | **0（全程未变真）** |
| `FDomain::IsLoadSucceed()` | **true（整场）** | **false（整场）** ⇒ `FDynamicRegistry`/`FClassRegistry` 的 C# 路径**不再执行** |
| `RegisterBinding()` 调用次数 | 1 | **2**（`FScriptDomainScope` 重试 1 次） |
| 套件 CSV | **只有表头**（1 行；1931 条用例一条未写） | **只有表头**（1 行）—— 3 次中 2 次 |
| C# 侧日志 | `LogUnrealCSharp: Error:` ＋ `Unhandled Exception: NullReferenceException` | **同**（3 次中 2 次各 1 条；崩溃那次无 CSV、日志止于启动期） |
| 进程退出 | 正常（harness 强杀） | **3 次中 1 次异常退出** —— 事件日志 `Faulting module = Plugins\UnrealCSharp\Binaries\Win64\shared\Microsoft.NETCore.App\10.0.4\coreclr.dll` / `Exception code 0xc0000005` / **`Fault offset 0x9603a`** / `.NET Runtime` 1023 号 `internal error … exit code 0x80131506` |

⇒ **改前的缺陷形态 = "域报健康 ＋ 所有 C#→C++ 调用静默失效 ＋ 归档一张只有表头的 CSV"**（正是死代码轮登记的"表头 CSV 不得当成功引用"同型）；**改后 = 同一故障下域状态不再说谎，既有下游据此停手**。这正是 Q2 "失败一律静默收敛到**既有**返回值（这里是既有状态通道）"的形态。

### 5.3 那次异常退出：指纹已登记为**环境事实**，非本轮改动

- 指纹四项与 [`08-…/12-环境事实-CoreCLR启动期崩溃`](12-环境事实-CoreCLR启动期崩溃（coreclr.dll@0x9603a）.md) **逐项相同**：`coreclr.dll`（插件 `Binaries\Win64\shared\…\10.0.4`）／`0xc0000005`／`Fault offset 0x9603a`／退出码 `0x80131506`；该文件把这类崩溃分 A（`FScriptDomainScope` 析构拆域，**已修** `77867446`）／B（分层编译竞态，**已缓解**）／**C（启动期·插件窗口之前，非插件可修，基线 ≈0.5%）** 三类。
- 本轮证据：**近 40 分钟事件日志中该指纹只出现 1 次**（即 P1 首次那一趟）；P1 随后重复 **2 次均正常产出 CSV**；同一二进制（撤回后）的**健康会话套件 2 次、Android/Windows 构建各 1 次、commandlet 1 次全部无异常**。⇒ 判定为 **C 类环境崩溃的 1 次命中**，**不作为本轮改动的读数**。
- 若要更严：该状态下 `FScriptDomainScope` 会重试初始化（本报告已把"重试"列为代价 §4-1）⇒ 理论上会多一次"装载程序集 + 新 ALC"的动作，而 A/B 类的邻居正是 ALC/GC 交互。**本轮无证据支持"重试放大崩溃率"**（n=1 vs 2 正常），故只登记不确定，不据此改设计。

---

## 6. 交付态判据（全部在**提交树等价态** = 工作树 ＋ 探针已还原）

| 判据 | 命令 | 读数 |
|---|---|---|
| **跨版本 API 核验（支持区间）** | `git show <tag>:Engine/Source/Runtime/CoreUObject/Public/UObject/UObjectBase.h` / `…/ObjectMacros.h`（`D:\file\UnrealEngine\Release`，32 个 release tag） | **5.0.0–5.8.3 全部**：`GetFlags()` = `public FORCEINLINE EObjectFlags GetFlags() const`、无 `UE_DEPRECATED`（实现两种形态、位语义相同）；**同时 `RF_AllFlags = (EObjectFlags)0xffffffff` 全版本一致** ⇒ 插件那条腿恒假、`GetFlags()` 的 `checkfSlow` 恒真 |
| Windows UBT | `Build.bat UnrealCSharpTestEditor Win64 Development -Project=… -WaitMutex` | **Succeeded**；**4 actions / 46.92 s**（删除恒假腿后的交付态；日志 `Saved/Retest/b28c-final-ubt.log:34-39`） |
| Android UBT（控制项） | `Build.bat UnrealCSharpTest Android Development -Project=… -WaitMutex` | **Succeeded**；`BUILD SUCCESSFUL in 30s` ＋ `COMPLETED MAKE APK`；**94.43 s**（`b28c-final-android-ubt.log:276-285`，NDK `r25b / clang 14.0.6`）；⚠️ 该目标的 `Intermediate/Build/Android/…/Definitions.UnrealCSharpCore.h` = **`WITH_CORECLR 0 / WITH_MONO 1 / WITH_LEANCLR 0`**（`Config/DefaultUnrealCSharpSetting.ini` `AndroidScriptDomainType=Mono`）⇒ **Mono 路径（含共享 `RegisterBinding()` 的 `else` 半边与 `FMonoDomain.hpp` 的去 `const`）已随本目标编译通过** |
| 套件 ×2 | `run-batch.ps1 -Tag b28c-final -Runs 2`（`-game -nullrhi -nosound`） | **1932 行 = 表头 ＋ 1931 条全部 `true`**（failed = 0）；CSV SHA **`B429F3EE013B8C73FADEAF12E9BFA13C6B28C4FFE280312DE64F6843EB4386F8`**（两份同 SHA，且与 B-24/B-25/B-26/B-27 基线**同 SHA**） |
| **删除恒假腿的 A/B（`-game` 单趟，探针落盘）** | `run-batch.ps1 -Tag b28c-del-pre`（腿在）↔ `-Tag b28c-del-post`（腿已删） | **逐项相同**：调用 **18** 次 / `objects` **520** / `pending` **490** / `skipped` **30**；改前 `leg-true` **0（18 次全 0）**；两趟 CSV **同 SHA**（§2.5） |
| 产物 manifest | `g8-manifest.ps1 -Tag b28-pre` / `-Tag b28c-after` ＋ `g8-manifest-diff.ps1` | `before=16230 after=16230` / `added=0 removed=0 changed=0`（`diff-b28-pre-b28c-after.txt`） |
| 日志/崩溃（健康会话） | `LogUnrealCSharp: Error` / `ensure` / `assert` / `Fatal` / `Unhandled Exception`；`Saved/Crashes` | **0**；今日新增崩溃目录 **0**（A/B 那次环境崩溃见 §5.3，事件日志 1 条） |
| C# 侧（控制） | `dotnet build Script/Script.sln` | **0 Warning / 0 Error**（2.74 s，撤回后重跑；本轮无 C# 改动） |
| 生成器副产物（控制） | `run-b26-commandlet.ps1 -Tag b28b-cmd-platform -WithPlatform` | `proxy after cmd : UE=16155 (was 16155)  Game=75 (was 75)` / `proxy diff : removed=0 added=0 changed=0` / `restore : stillMissing=0 stillDifferent=0`（wall 44.0 s；exit 1 = 既有 706 error 的固定形态） |
| 装置还原 | `TEMP-B28` ｜ `UCS_B28_PROBE` ｜ `TempB28*` 全树 grep | **0 / 0 / 0** |
| 🟢 **提交树复验（`a9e10378`）** | `git diff a9e10378 -- Source/UnrealCSharp Source/UnrealCSharpCore`；`Build.bat … -WaitMutex`；`run-batch.ps1 -Tag b28d-commit -Runs 2` | **diff 为空**（提交 = 施工期工作树、**零调整**）；UBT **`Target is up to date`**（11.92 s）；套件 **1932 行 / 0 failed ×2**、CSV **同 SHA `B429F3EE…`** |
| 补丁（**由提交树导出**） | `git diff --output=… df487a6d a9e10378 -- <本轮的 4 个文件>` | `Saved/Retest/fix-B28-cs1040-fl014-deadleg.patch` —— **3 015 B** / SHA256 `CDEE4AE3…` / LF / 无 BOM / reverse-check **exit 0**（与施工期 `…-df487a6d.patch` **逐字节相同**） |

---

## 7. 未取得与残留面（如实登记）

1. **编译器矩阵（弱点 ②）**：MSVC 之外的 Clang/GCC 对"模板实参中的私有成员地址"的接受度**未取得**（本机无矩阵）；
2. **同 TU 双展开的冲突复现（弱点 ①）**：未构造（需两个短名相同的类在同一 TU 各展开一次；删除使用点 ① 后，剩余两个使用点仍各在自己的 TU 里）；
3. **Shipping 配置下的实跑**：本轮 UBT 只编 Development（`Win64 Development`），**未编 Shipping**；`checkfSlow`/`check` 在 Shipping 下为空属引擎宏知识（未实测编译产物）；
4. **"越界位为真"不可构造**：要把 `ObjectFlags` 写成含 `~RF_AllFlags` 之外的位置，在当前掩码（全 1）下**不存在**这样的位 ⇒ 该腿为真的读数**原理上取不到**（这也是"死腿"的证据之一）；`GetFlags()` 的 `checkfSlow` 因此同样无法取得实测触发；
5. **`#509` 的精确崩溃点未取得** —— 只有提交标题（`Fixed OnAsyncLoadingFlushUpdate checkfSlow`）与 diff 两块（裸读腿 ＋ `NotifyUObjectDeleted` 移出数组），**没有 issue 正文、没有调用栈**。机制推定（**置信度中**）：已删对象留在 `AsyncLoadingObjectArray` 里被后续 API 触碰（`IndexToObject` 的硬 `check(Index >= 0)`，或 `HasAnyInternalFlags` 经 `GUObjectArray` 的路径），而"移出数组"那一块才是有效修复；
6. **`InLength == 0` 半边**：三会话名表恒 2370，**未能构造**"名表为 0"的会话 ⇒ 该半边保持登记（§4）；
7. **Mono 后端实跑**：本轮 A/B 与交付态判据都在 **CoreCLR** 下取得；**编译面已覆盖**（Android 目标 = `WITH_MONO 1`，见 §6），但 Mono 的**运行读数**未取得（未切 `WindowsScriptDomainType=Mono` 跑套件）；
8. **`F-CS1-040` 的"生产版静默"**：`-game` 会话里漂移是"硬红"，打包版/玩家机上的形态**未实测**（`void` 桥接调用静默 no-op 是从生成产物形态推出的）；
9. **定制引擎不覆盖**：跨版本结论的样本是**官方 release tag**（5.0.0–5.8.3）；若宿主使用**改过 `RF_AllFlags`/`ObjectFlags`** 的定制引擎，"恒假"与"无需门控"两条结论都要重算。

---

## 8. 复用价值（方法论）

1. **"注册/初始化失败"类条目，先问"有没有既有取值域"，再问"谁在消费它"** —— 本条可施工的原因是 `IsInitialized()` **已经被两个既有消费者当成闸门**（`FDynamicRegistry` / `FClassRegistry`），所以"把状态写对"就等于"接入既有失败处理"，不需要新增任何面；若没有消费者，同样的写法只会制造一个没人看的标志位。
2. **审计条目的"后果"必须先构造一次再引用** —— `F-CS1-040` 被引用了多轮的"跳转 0 崩溃"，在 `90f239f4` 之后已不成立；本轮用"改桥函数名"这一 **1 行装置**把真实后果（表头 CSV ＋ NRE 日志）当场取到。**代价最低的构造方式通常不是改被测对象，而是改它的输入契约**（这里是按名解析的名字）。
3. **A/B 的差值要落在"既有可观测面"上** —— 改前/改后套件 CSV 同为"只有表头"，差异只在**域状态**；若只盯 CSV 会得出"没有差异"的错误结论。⇒ 判据要选"**下游会据以行动**"的那个量。
4. **普查要允许"查完发现不能动"，并把撤回本身写成结论** —— `F-FL-014` 的普查结论不是"改哪一处"，而是"**三处都不能改** ＋ 新增一条死腿事实 ＋ 一项待裁定"；首版"找到一处就替换"的结论**已被本轮自身推翻**。撤回必须留档（含作废补丁的 SHA），否则下一位读报告的人会照着做。
5. **改公开头文件里的宏之前，先量"仓内下游 + 仓外可能"** —— 本仓库内 grep = 0，才使"删调用点／换公开 API"零风险；同时把"Public 头 ⇒ 对外半公开"写进结论，避免把"没搜到"当成"没人用"。
6. **改引擎 API 之前，先量"支持区间"（tag 级）** —— 支持区间是 **UE 5.0–5.8**，"本机编得过"只证明**一个**版本。`D:\file\UnrealEngine\Release` 是带 release tag 的引擎源码仓 ⇒ `git show <tag>:<path>` 可逐版取头文件：32 个 tag 的 `GetFlags()` 签名/可见性完全一致（**这一层结论没错，但只证到"能用"，没证到"该用"**）。
7. **🔴 注释与提交史是 API 语义的一部分：`git log -S <表达式>` 先于替换** —— 本轮 `git log -S '~RF_AllFlags'` 一条命令就给出 `#509 Fixed OnAsyncLoadingFlushUpdate checkfSlow`：裸读是**有意绕开检查**的写法。**"引擎 API 存在且位级等价"与"被替换代码为什么这么写"是两件事**；只做前者会把"有意绕开"读成"陈旧 hack"，并把一个**恒假**的腿换成一条**将来可能致命**的调用。
8. **恒假条件要按"掩码是否全 1"判，而不是按探针为 0 判；判明之后，删除优于替换** —— 探针 18 次 / 520 对象全 0 只能说明"恰好没触发"；`RF_AllFlags = 0xffffffff` 才说明"**不可能**触发"。两者会导出完全相反的行动（前者"可以删/可以换"，后者"**删是中性、换有风险**"）。本轮最终采取的正是"**删除恒假项**"：既消掉 hack，又不动相邻三条腿的语义。

---

## 9. 台账回写与锚点

| 位置 | 回写内容 |
|---|---|
| [`09-…/00`](../09-问题清单/00-推荐优先修复清单.md) §0.0 T2 表 | **第 26 行 `F-FL-014`** → "🟡 2026-10-16 轮 B-28：**普查收口** —— 使用点 ① **不可替换**（有意绕开 `GetFlags()` 的 `checkfSlow`；引入提交 `b38391bd`/`#509`）⇒ 经维护者裁定**删除恒假腿（已施工，A/B 逐行相同）**；② 无公开替代、③ 受 5.6 门控不可删；🆕 该腿在 5.0–5.8.3 **恒假**（`RF_AllFlags = 0xffffffff`）"（**级别维持 P2、S 维持 3.08**，不写"已修"）；**第 28 行 `F-CS1-040`** → "🟢 轮 B-28：注册整体失败半边已修（既有状态通道 fail-closed）⇒ T2 出列；崩溃半边前提更正；`InLength == 0` 与 LeanCLR 半边登记"；并新增"轮 B-28"更新注（含 T2 余量与干净开放面变化） |
| [`09-…/01`](../09-问题清单/01-全局问题总表.md) | `F-FL-014`（`#L377`）与 `F-CS1-040`（`#L439`）两行的「复核结论」列就地更新（含 `90f239f4` 前提更正、`#509` 裸读意图、32-tag 核验、删除裁定与 A/B 读数） |
| [`09-…/05`](../09-问题清单/05-已完成（P2与P3）.md) | 新增 **§28 轮 B-28 收口**（逐条处置 ＋ 交付态判据 ＋ 撤回/删除留档 ＋ 下一顺位） |
| [`README.md`](../README.md) | 轮次表新增 B-28 行（改动面、判据、补丁 SHA256、撤回与删除说明） |
| [`INDEX.md`](../INDEX.md) | 新增本轮"另（2026-10-16，轮 B-28 …）"条目并回指本报告 |
| 本报告 | [`08-…/68`](68-轮B28收口-注册整体失败fail-closed与AccessPrivate不可替换复核（2026-10-16）.md)（**v3**；v1 结论被 v2 §2.1–§2.4 推翻、v2 的"0 代码改动"被 v3 §2.5 取代；🟢 **已提交 `a9e10378`**（本轮 4 文件 `+8/−8`，其 blob 与施工期工作树逐一相同）；作废补丁 `fix-B28-fl014-cs1040-df487a6d.patch` 2 972 B / `896F4803…` 与中间版 `fix-B28-cs1040-df487a6d.patch` 1 794 B / `25BD447E…` 均不再交付；交付补丁 = 由提交树导出的 `fix-B28-cs1040-fl014-deadleg.patch` 3 015 B / `CDEE4AE3…`） |
