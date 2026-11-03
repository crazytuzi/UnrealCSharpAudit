# 轮 B-228 `F-REG2-006` 施工：`FUObjectListener` 注册生命周期收口（2026-11-03）

> **分析范围**：`F-REG2-006`（P2 · S=2.80 · 潜伏）——「模块 `ShutdownModule` 不解除 `FUObjectListener` 在 `GUObjectArray` 上的注册」
> **方案**：B-227 设计页 [`08-…/267`](267-设计页-F-REG2-006-UObjectListener注册生命周期（2026-11-03）.md) 的 **方案 A（收敛版）**（`AddListeners()`/`RemoveListeners()` 内部自持 `bIsRegistered` 幂等守卫；析构与数组关停两处摘除；**未**改 `ShutdownModule`）
> **规模**：插件 **2 文件 `+36/−2`**（0 新增公开 API / 日志 / 断言）
> **判据**：**改前**双次 InActive ⇒ `Assertion failed: NumRemoved==1`（`UObjectArray.cpp:436`）崩溃；**改后**同探针 `call1 → call2 → done`、退出 exit 0；UBT 5.6 Editor **Succeeded**、UBT 5.0 **Succeeded**、套件 ×2 **1 992 / 0 / 同 SHA `085D3178…`**、全量生成 **16 231 文件 delta 0**、§7 **CLEAN**、§8 **PASS**
> **状态**：🟢 **已提交** `efdca39f`（父 `07ea9e7d`）

---

## 1. 改前读数：第二次摘除必断言（实测）

### 1.1 探针（临时，已删）

`FUObjectListener` 加临时 `ProbeInActiveTwice()`（连调两次 `OnUnrealCSharpModuleInActive()`，中间打点），由 `FUnrealCSharpModule::StartupModule` 注册临时 console command `UnrealCSharp.Probe.Listener`；Python driver 在 T+25 s 发出该命令（此时模块已 Active、listener 已注册）。

### 1.2 读数（`Saved/Retest/b228/b228-pre/editor.log`）

```
09:33:04.167  [b228] probe call 1
09:33:04.167  [b228] probe call 2
09:33:04.176  LogWindows: Error: appError called: Assertion failed: NumRemoved==1
              [File:…\CoreUObject\Private\UObject\UObjectArray.cpp] [Line: 436]
exit code = 3
```

- 第一次摘除成功；**第二次摘除**触发引擎 `RemoveUObjectCreateListener` 的 `check(NumRemoved==1)` ⇒ 编辑器崩溃。
- 这条读数同时证明了：**只要同一次会话里有第二条"注销"路径**（模块卸载、数组关停），原实现就会断言 —— 这正是设计页 §1.2 预判的"修法必须幂等"。

## 2. 施工（方案 A 收敛版）

| 文件 | 改动 |
|---|---|
| `Public/Listener/FUObjectListener.h` | 新增私有 `AddListeners()`/`RemoveListeners()`；新增 `bool bIsRegistered{}`（可见性零变化，两个 handler 仍在 private） |
| `Private/Listener/FUObjectListener.cpp` | `AddListeners()`/`RemoveListeners()` 内部用 `bIsRegistered` 做幂等守卫（同时防 `Add` 的 `check(!Contains)` 与 `Remove` 的 `check(NumRemoved==1)`）；`OnActive`/`OnInActive` 退化成一行转发；析构与 `OnUObjectArrayShutdown()` 直接调 `RemoveListeners()`（后者满足引擎"回调内自摘"契约） |

- **路径覆盖**：① 正常退出（`OnPreExit` 已置 flag=false ⇒ 后两处均为 no-op）；② 模块卸载（`~FUObjectListener` 摘除）；③ 引擎数组关停（`OnUObjectArrayShutdown` 自摘，避免 `Fatal`）；④ 析构兜底与 ② 同一处。
- 🔴 **相对设计页的收敛（维护者定）**：设计页方案 A 原含"`ShutdownModule()` 显式摘"这一条；复核后**删除** —— UE 卸载模块是 `ShutdownModule()` → 立刻 `delete Module` → 成员析构，析构路径必然执行且覆盖同一窗口，显式调用是**冗余**；删除后还免掉了把 `OnUnrealCSharpModuleInActive` 移 public 的可见性变化。
- 🟡 **命名/结构整理（同日，维护者定）**：`bIsListening` → **`bIsRegistered`**（`Reset` 类词不适用；状态是"已注册为 GUObjectArray 的 create/delete listener"，与引擎 `Add/Remove…Listener` 同词根）；并把"注册/注销"收进 `AddListeners()`/`RemoveListeners()`，析构与数组关停不再借 `OnUnrealCSharpModuleInActive()` 表达摘除。规模因此由 `+20/−4` 变为 **2 文件 `+36/−2`**（纯结构，行为等价）。
- **G7**：0 新增日志/断言/`throw`/公开 API。

## 3. 判据

| 项 | 读数 |
|---|---|
| 双次 InActive 探针（改前） | `Assertion failed: NumRemoved==1`（`UObjectArray.cpp:436`）⇒ 崩溃，exit 3（§1.2） |
| 双次 InActive 探针（改后） | `probe call 1` → `probe call 2` → `probe done`，exit **0**（`b228-post/editor.log`） |
| UBT 5.6 Editor | **Succeeded**（探针态 61.05 s；修复态 47.83 s；交付态 67.65 s；结构整理后 46.06 s） |
| UBT 5.0（`UE50Check`，同一份源码） | **Succeeded**，139.56 s |
| 套件 ×2 | **1 992 行 / 0 / SHA `085D3178D15B…`**，与基线逐字节相同（`out/b228-reg2-006/`；收敛版 `b228-simplified/`、结构整理版 `b228-structure/` 均同 SHA） |
| 全量生成 | **16 231 文件逐字节 delta 0** |
| §7 语法面 | **CLEAN**（`-DiffBase 07ea9e7d`） |
| §8 风格面 | **PASS**（2 文件；Added 30/6；OverWidth 0；AddedMax 53） |
| 声明/定义顺序 | **一致**（两侧逐项：ctor → dtor → `AddListeners` → `RemoveListeners` → `OnActive` → `OnInActive` → `NotifyUObjectCreated` → `NotifyUObjectDeleted` → `OnUObjectArrayShutdown`） |
| 探针清理 | 插件 `Source/**` 中 `b228`/`ProbeInActiveTwice`/`ProbeConsoleCommand` 命中 **0**（1 处为 zlib 十六进制巧合） |

## 4. 未取得 / 边界

1. **"悬垂回调"未实测**：没有做 `FModuleManager::UnloadModule("UnrealCSharp")` 的设备（风险高于收益，设计页 §5 已列为高成本可选项）⇒ "模块卸载后到数组关停之间"这一段仍是**静态推定**；本轮实测只覆盖"幂等性"。
2. **引擎 `Fatal` 契约路径未实测**：要让 `ShutdownUObjectArray()` 在 listener 仍注册时运行，需要一个"跳过 OnPreExit"的装置；`OnUObjectArrayShutdown` 的自摘是按引擎契约写的（`UObjectArray.cpp:534-549`），未取得该路径的运行期读数。
3. **模块卸载路径未取运行期读数**：收敛版把摘除放在 `~FUObjectListener`（`ShutdownModule()` 之后必然执行），未做 `UnloadModule` 设备 ⇒ 该路径为静态推定，且与幂等探针走的是同一段代码（`OnUnrealCSharpModuleInActive`）。

## 5. 台账与提交

| 位置 | 变化 |
|---|---|
| `01` | 迁出 `F-REG2-006` 一行 ＋ 新增 B-228 banner ⇒ **732 行 / P2 20** |
| `05` | 新增 **§159**：`F-REG2-006` 已修（改前/改后读数） |
| `06` | 移除 `F-REG2-006` 一行 ⇒ **20 条（F 0 / C 17 / D 2 / G 1）** |
| `07` | 补 B-228 行 |
| 插件提交 | **`efdca39f`**（父 `07ea9e7d`；提交共 8 文件 `+135/−34`），本轮 2 文件 `+36/−2` |
