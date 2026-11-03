# 设计页 轮 B-227 `F-REG2-006` `FUObjectListener` 的注册生命周期（2026-11-03）

> **分析范围**：`F-REG2-006`（P2 · S=2.80 · 潜伏）——「模块 `ShutdownModule` 不解除 `FUObjectListener` 在 `GUObjectArray` 上的注册（对外广播依赖，重载/卸载路径可能留悬垂监听器）」
> **规模**：**0 代码**（只读复核 ＋ 方案设计）；基线 = 插件 HEAD `efdca39f`
> **结论**：原文只说了"悬垂监听器"，实际还有**第二条更硬的后果** —— 引擎的 `ShutdownUObjectArray()` 要求每个 listener 在 `OnUObjectArrayShutdown()` 里**自摘**，否则 `Fatal`（`All UObject delete listeners should be unregistered…`）；当前实现只转发、不自摘
> **推荐**：**方案 A**（listener 自持 `bIsRegistered` ＋ 三处幂等摘除 ＋ `ShutdownModule` 显式摘），≈15 行 / 2 文件；**不本轮施工**，待维护者对 §6 两个决策点定口径
> **状态**：`F-REG2-006` 维持 **P2 / 潜伏 / 不施工**

---

## 1. 现状：注册/注销的四条路径

### 1.1 代码锚点（HEAD `efdca39f`）

| 路径 | 位置 | 行为 |
|---|---|---|
| 订阅 | `FUObjectListener.cpp:5-12`（ctor） | 订阅 `FUnrealCSharpModuleDelegates::OnUnrealCSharpModule{Active,InActive}` |
| 注册 | `:29-34`（Active 回调） | `GUObjectArray.AddUObjectCreateListener/DeleteListener(this)` |
| 注销 | `:36-41`（InActive 回调） | `RemoveUObjectCreateListener/DeleteListener(this)` |
| 析构 | `:14-27` | **只 Remove 两个模块委托**，不动 `GUObjectArray` |
| 数组关停 | `:53-56` | `OnUObjectArrayShutdown()` **只转发**给 `FCSharpEnvironment`，不自摘 |
| 模块卸载 | `UnrealCSharp.cpp:19-34` | 只 Remove 订阅 Core 的两个委托句柄，**不广播自己的 InActive** |
| Core 卸载 | `UnrealCSharpCore.cpp:13-17` | **空体**（不调 `Deactivate()`） |

- `FUObjectListener` 是 `FUnrealCSharpModule` 的**值成员**（`UnrealCSharp.h:27`）⇒ 随模块构造/析构。
- 正常退出的摘除链：`FEngineListener::OnPreExit()`（`FEngineListener.cpp:92-95`）→ `SetActive(false)` → `FUnrealCSharpCoreModule::Deactivate()` → 广播 Core InActive → `FUnrealCSharpModule::OnUnrealCSharpCoreModuleInActive` → 广播模块 InActive → listener 摘除 ＋ `FCSharpEnvironment::Deinitialize()`。

### 1.2 引擎侧的三条硬契约（🔴 本轮新读）

| 引擎位置 | 契约 |
|---|---|
| `UObjectArray.cpp:422-426` `AddUObjectCreateListener` | `check(!UObjectCreateListeners.Contains(Listener))` ⇒ **重复注册必断言** |
| `:433-437` `RemoveUObjectCreateListener` | `RemoveSingleSwap` 后 `check(NumRemoved == 1)` ⇒ **摘未注册的 listener 必断言** |
| `:534-549` `ShutdownUObjectArray` | 逐个回调 `OnUObjectArrayShutdown()` 之后：`UE_CLOG(Num(), LogUObjectArray, Fatal, "All UObject … listeners should be unregistered …")` ⇒ **回调必须自摘** |

调用链：`FCoreDelegates::OnExit.AddStatic(StaticExit)`（`Obj.cpp:5542`）→ `StaticExit()`（`:5607`）→ `UObjectBaseShutdown()`（`UObjectBase.cpp:1296`）→ `GUObjectArray.ShutdownUObjectArray()`。

## 2. 两个后果

1. **悬垂回调**（原条目）：模块在 Active 状态下被卸载（插件热重载 / `FModuleManager::UnloadModule` / 运行时禁用插件）⇒ `GUObjectArray` 仍持有指向已卸载模块的 listener 指针；后续任何 `UObject` 创建/删除都会调到 `NotifyUObjectCreated/Deleted`（`FUObjectListener.cpp:43-51` → `FCSharpEnvironment::GetEnvironment()`）⇒ 访问已卸载代码段。
2. **引擎 Fatal**（新）：同一窗口下若进程正常走到 `ShutdownUObjectArray()`，该 listener 的 `OnUObjectArrayShutdown()` 不自摘 ⇒ 回调后 `Num()` 仍 > 0 ⇒ `Fatal`。当前之所以没炸，是因为 `OnPreExit` 先把 listener 摘干净了 —— **依赖外部回调顺序，不是代码自身保证**。

> 两条后果的触发前提相同（"模块卸载先于/绕过 `OnPreExit`"），区别只是先撞到哪一条。

## 3. 可达路径

| 路径 | 是否摘除 | 说明 |
|---|---|---|
| 正常编辑器退出 / `-game` 退出 | ✅ | `FCoreDelegates::OnPreExit` → Deactivate → 级联摘除 |
| 模块卸载（插件热重载、`UnloadModule`、运行时禁用） | ❌ | `ShutdownModule` 不广播 InActive；`Core` 的 `ShutdownModule` 空体 |
| 强杀 / 崩溃退出 | N/A | 无意义 |
| `bEnableImmediatelyActive=false` 的会话 | ✅（未注册） | 从未 Active 过 |

⇒ 可达性维持 **潜伏**：正常退出路径已覆盖，窗口只在"进程内卸载模块"。

## 4. 方案

### A（推荐）：listener 自持状态 ＋ 三处幂等摘除

| 改动 | 行数 |
|---|---|
| `FUObjectListener` 加 `bool bIsRegistered{}`；Active/InActive 各自 `if (!bIsRegistered)` / `if (bIsRegistered)` 守卫（同时防 `Add` 的 `check(!Contains)` 与 `Remove` 的 `check(NumRemoved==1)`） | ~6 |
| `OnUObjectArrayShutdown()` 首行调 `OnUnrealCSharpModuleInActive()`（满足引擎自摘契约） | 1 |
| `~FUObjectListener()` 首行兜底自摘（flag 保证 no-op） | 1 |
| `FUnrealCSharpModule::ShutdownModule()` 显式调 `UObjectListener.OnUnrealCSharpModuleInActive()`（`OnUnrealCSharpModuleInActive` 移 public，或加 public `Unregister()`） | 1–3 |

- **G7**：0 新增日志/断言/`throw`/公开 API（listener 类未导出，方法可见性调整不算公开面）。

> 🟢 **实现备注（B-228，2026-11-03）**：落地时按维护者意见收敛 —— 删去 `ShutdownModule()` 显式摘（析构已覆盖同一窗口，见 [`08-…/268`](268-轮B-228-F-REG2-006-UObjectListener注册生命周期收口（2026-11-03）.md) §2），并把"注册/注销"抽成 `AddListeners()`/`RemoveListeners()`，本页 A 表因此是**设计原案**、不是最终形态。
- **不依赖模块卸载顺序**：直接操作自己的 listener，不广播跨模块委托、不触发 environment 卸载（域清理由现有 `Deinitialize` 路径负责；F-DOM-005 的 A2 已在 `Destroy` 时兜底）。
- 风险：`OnUObjectArrayShutdown` 在引擎 `ShutdownUObjectArray` 的反向遍历里自摘 —— 这正是该回调的**设计契约**（引擎遍历时自摘，`RemoveSingleSwap` 换入的尾部元素已被处理过），不是自造形态。

### B：只修 `OnUObjectArrayShutdown` 自摘

- ≈4 行 / 1 文件；消除"引擎 Fatal"，但模块卸载窗口仍留悬垂指针。**不完整**。

### C：只在 `FUnrealCSharpCoreModule::ShutdownModule()` 调 `Deactivate()`

- 1 行；靠 Core → UnrealCSharp 的委托级联摘除。**依赖模块卸载顺序**（Core 先于 UnrealCSharp 才有效），且不满足引擎自摘契约；若顺序相反则完全无效。**不推荐**。

### D：契约 ＋ 触发条件

- 声明"不支持进程内模块卸载"，触发条件 = 启用 Live Coding/插件热重载/Live Coding 的模块重载。0 代码，但把已知硬契约违反留在代码里。

### 比较

| 维度 | A | B | C | D |
|---|---|---|---|---|
| 悬垂回调 | ✅ | ❌ | ⚠️（依顺序） | ❌ |
| 引擎 Fatal 契约 | ✅ | ✅ | ❌ | ❌ |
| 双次摘除断言 | ✅（flag） | ⚠️（无 flag 时） | ✅（状态机已防） | — |
| 行数/文件 | ≈15 / 2 | ≈4 / 1 | 1 / 1 | 0 |
| G7 | 合规 | 合规 | 合规 | 合规 |

## 5. 判据设计（执行轮用，本轮未跑）

1. **静态**：`Add`/`Remove` 两侧都有 flag 守卫；`grep` 确认 `RemoveUObjectCreateListener` 只从 `OnUnrealCSharpModuleInActive()` 一处调用（幂等入口唯一）。
2. **幂等探针（低成本）**：临时 console command 连续调 `UObjectListener.OnUnrealCSharpModuleInActive()` 两次 ⇒ 改前第二次 `check(NumRemoved==1)` 断言崩溃；改后第二次 no-op。再用 `Active/InActive` 交替验证 `check(!Contains)` 不触发。
3. **引擎契约探针（中成本）**：临时把 `OnUObjectArrayShutdown()` 打日志，在 `-game` 正常退出里确认它被调用且**自摘后**引擎无 `Fatal`（改前靠 OnPreExit 提前摘除，读不到差异；可在探针里模拟"OnPreExit 不摘"的路径 —— 用一个临时开关跳过 `SetActive(false)`）。
4. **模块卸载设备（高成本，可选）**：临时命令 `FModuleManager::UnloadModule(TEXT("UnrealCSharp"))` → 触发 GC/创建 UObject → 改前崩溃、改后存活。**风险高**（卸载运行中模块可能引发无关崩溃），只在专用会话做。
5. **回归**：UBT 5.6/5.0 Succeeded；`-game` 套件 ×2 **1 992/0 同 SHA `085D3178…`**；全量生成 delta 0（本轮改动不在生成器，预期 0）；§7/§8。

## 6. 决策点

1. **是否接受 listener 类的方法可见性调整**（`OnUnrealCSharpModuleInActive` 移 public 或加 public `Unregister()`）？类未导出、不属公开 API；若维护者要求零可见性变化，可改用 `friend class FUnrealCSharpModule`。
2. **执行轮做到哪一层**：A 全量（含模块卸载设备）／A 静态 ＋ 幂等探针（不做卸载设备）／只做 B。建议 **A 静态 ＋ 幂等探针**：卸载设备风险高于收益，潜伏条目的判据以"幂等 ＋ 契约"为主。

## 7. 台账

| 位置 | 变化 |
|---|---|
| `01` | 不动（`F-REG2-006` 仍为 P2 开放行） |
| `05` | 不动 |
| `06` | 补 B-227 note（设计页已出、维持 P2、待口径） |
| `07` | 补 B-227 行（0 代码/设计） |
| `08` | 本页 |
