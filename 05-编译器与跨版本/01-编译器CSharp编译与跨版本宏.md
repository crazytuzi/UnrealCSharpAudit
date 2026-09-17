# UnrealCSharp — Compiler / CrossVersion / SourceCodeGenerator 分析报告

> 本报告发现的**严重度见各 Finding 的「复核结论」字段**。
>
> **字段完整性**：33 条发现的 `复核结论`/`可达性`/`复核证据`/`级别变动` 字段**全部**已补齐（其中逐行深读的仅 5 条）。
> 结论分布：**确认 23 / 部分确认 9 / 证伪 0 / 无法验证 1**（`F-CMP-022`，仅其核心子项；卡点：`UhtExportFactory.CommitOutput` 源码不在本机）。
> **级别变动 7 条（全部下调，维持 26 条）**：`F-CMP-001`/`F-CMP-003`/`F-CMP-005` P1→P2（触发条件窄、无数据损坏）；`F-CMP-010`/`F-CMP-013`/`F-CMP-021` P2→P3（证据不足 / 前提被引擎源码证伪）；`F-CMP-031` P3→**撤销（非缺陷）**。
> 可达性分布：**活跃 21 / 潜伏 10 / 不可达 2**（`F-CMP-016`、`F-CMP-027`）。





> 分析范围：`Plugins/UnrealCSharp/Source/Compiler`、`Source/CrossVersion`、`Source/SourceCodeGenerator`
> 覆盖文件：13 个指定文件（9 个 Compiler + 6 个 CrossVersion 中的 5 个 .h/.cpp/build.cs + 1 个 C# 程序）＋ 为取得证据而读取的支撑文件 9 个

> 未覆盖部分见 §5，逐函数清单见 §3.0

---

## 0. 覆盖范围与阅读清单

| 文件 | 实际行数 | 是否读完 | 备注 |
|---|---|---|---|
| `Source/Compiler/Public/Compiler.h` | 15 | 读完 | 空模块声明 |
| `Source/Compiler/Private/Compiler.cpp` | 20 | 读完 | `StartupModule`/`ShutdownModule` 均为空 |
| `Source/Compiler/Public/FCSharpCompiler.h` | 32 | 读完 | |
| `Source/Compiler/Private/FCSharpCompiler.cpp` | 79 | 读完 | 线程创建/销毁、同步编译入口 |
| `Source/Compiler/Public/FCSharpCompilerRunnable.h` | 77 | 读完 | |
| `Source/Compiler/Private/FCSharpCompilerRunnable.cpp` | **483** | 读完 | 任务书写 375 行，实际 483 行，已按实际全文阅读 |
| `Source/Compiler/Public/FCSharpCompileProgress.h` | 52 | 读完 | |
| `Source/Compiler/Private/FCSharpCompileProgress.cpp` | 136 | 读完 | |
| `Source/Compiler/Compiler.Build.cs` | 58 | 读完 | |
| `Source/CrossVersion/CrossVersion.build.cs` | 53 | 读完 | |
| `Source/CrossVersion/Private/CrossVersion.cpp` | 20 | 读完 | 空壳模块 |
| `Source/CrossVersion/Public/CrossVersion.h` | 15 | 读完 | |
| `Source/CrossVersion/Public/CppVersion.h` | 9 | 读完 | 4 个宏 |
| `Source/CrossVersion/Public/DotnetVersion.h` | 13 | 读完 | 5 个宏 |
| `Source/CrossVersion/Public/UEVersion.h` | 206 | 读完 | **101 个宏** |
| `Source/CrossVersion/Public/VSVersion.h` | 11 | 读完 | 4 个 `#define`（2 个宏 × 2 分支） |
| `Source/SourceCodeGenerator/SourceCodeGenerator.cs` | **873** | 读完 | 任务书写 707 行，实际 873 行，已按实际全文阅读 |
| `Source/SourceCodeGenerator/SourceCodeGenerator.ubtplugin.csproj` | 49 | 读完 | 项目类型是 **UBT/UHT 插件**，不是 .uplugin 里写的 Program |
| `Source/SourceCodeGenerator/SourceCodeGenerator.ubtplugin.csproj.props` | 6 | 读完 | 硬编码 `EngineDir=Engine` |

**为取证而读取的支撑文件（非本人负责文件，仅引用）**

| 文件 | 读取范围 | 用途 |
|---|---|---|
| `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp` | 42-57、720-792、1085-1103、1385-1405、1520-1592 | **进程/管道唯一实现 `SyncProcess`**、`GetDotNet`、路径函数、`GetDotnetVersion` |
| `Source/UnrealCSharpCore/Public/Common/FUnrealCSharpFunctionLibrary.h` | 230-290 | `SyncProcess` 声明 |
| `Source/UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp` | 21-45、120-180 | 默认值（`bEnableCompiled=true`、`bEnableExport=false`）、`GetDotNetPathArray` |
| `Source/UnrealCSharpCore/Public/Setting/UnrealCSharpEditorSetting.h` | 1-120 | `ESolutionConfiguration`、`DotNetPath` |
| `Source/UnrealCSharpCore/Public/Setting/UnrealCSharpSetting.h` | 60-140 | **`EDotnetVersion` 枚举（Latest=V10+1）** |
| `Source/UnrealCSharpCore/Private/Setting/UnrealCSharpSetting.cpp` | 9-26、155-157 | `DotnetVersion(EDotnetVersion::Latest)` 默认值 |
| `Source/UnrealCSharpCore/UnrealCSharpCore.build.cs` | 231、258-306 | `WITH_BINDING`、Mono/CoreCLR/LeanCLR 三选一与 `DOTNET_*_VERSION` 来源 |
| `Source/ThirdParty/{CoreCLR,LeanCLR,Mono}/*.Build.cs` | CoreCLR 1-45、LeanCLR 1-45、Mono 13-15 | **`DOTNET_MAJOR/MINOR/PATCH_VERSION` 的真实值（10.0.4 / 10.0.4 / 10.0.1）** |
| `Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp` | 1-30、50-99 | `DOTNET8`/`DOTNET9` 的唯一消费者 |
| `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp` | 1-160、200-330、435-467 | `net%d.%d` 生成逻辑、`VS2026` 唯一消费者 |
| `Source/ScriptCodeGenerator/Private/FCodeAnalysis.cpp` | 全文 89 | 与 SourceCodeGenerator 的关系判定 |
| `Source/UnrealCSharpEditor/Private/{UnrealCSharpEditor.cpp,FEditorListener.cpp,ContentBrowser/DynamicDataSource.cpp}` | 90-140、250-270、370-414、620-755 | 编译触发点、进度窗口、取消能力 |
| 引擎源码 `Engine\...` | 见各 Finding | 验证 `ReadPipe`/`CreateProc`/`CreatePipe`/`Kill`/TaskGraph/`EngineVersionComparison.h`/UBT 的 `/Zc:__cplusplus` |
| 工程根 `Script/`、`Config/`、`Binaries/DotNET/`、`Plugins/UnrealCSharp/Intermediate/` | 目录与文件内容抽查 | TargetFramework 一致性、SourceCodeGenerator 是否真正在用的实证 |

---

## 1. 模块职责与架构速览

**Compiler（Editor 模块）** —— 只有两层：一个**单例包装** + 一个**常驻后台线程**。

- `FCSharpCompiler`（`FCSharpCompiler.h:5`，带 `COMPILER_API`）是进程内单例，`Get()` 用**函数局部静态对象**实现（`FCSharpCompiler.cpp:29-34`），构造时 `new FCSharpCompilerRunnable()` 并 `FRunnableThread::Create(Runnable, TEXT("CSharpCompiler"))`（`FCSharpCompiler.cpp:8-10`）。
- `FCSharpCompilerRunnable` 继承 `FRunnable`，`Run()` 是一个 `while(true)` 轮询循环（`FCSharpCompilerRunnable.cpp:52-90`）：检查 `bIsStopped` → 检查 `bIsGenerating` → 从 `TQueue<bool> Tasks` 取任务 → `DoWork()`；队列空则 `Event->Wait()` 睡眠。
- **所有进程创建都在模块外**：`FCSharpCompilerRunnable` 自己不做 `CreateProc`，而是两次调用 `FUnrealCSharpFunctionLibrary::SyncProcess`（`FCSharpCompilerRunnable.cpp:291`、`413`），由后者在 `FUnrealCSharpFunctionLibrary.cpp:1520-1592` 完成 `CreatePipe`/`CreateProc`/轮询读/`ClosePipe`/`CloseProc`。
- **进度不是百分比**：`FCSharpCompileProgress` 只维护一个"阶段文本"（8 个 `StageXxx` 常量，`FCSharpCompileProgress.cpp:5-19`），并把 dotnet 输出逐行解析成阶段（`ParseLine`，`FCSharpCompileProgress.cpp:89-136`）。UI 侧是两条消费链：后台线程里的 Slate 通知 + `FTSTicker`（`FCSharpCompilerRunnable.cpp:325-382`），以及 `FEditorListener::WaitForCompile()` 的模态进度窗（`FEditorListener.cpp:676-745`，显示"状态文本 + 已用秒数"，**无取消按钮**——`SCompileProgressDialog` 中不存在 `SButton`/`OnClicked`，grep 命中 0）。
- 编译完成后的收尾全部 marshal 回游戏线程：`AsyncTask(ENamedThreads::GameThread, ...)`（`FCSharpCompilerRunnable.cpp:325`、`419`、`441`）以及 `FFunctionGraphTask::CreateAndDispatchWhenReady(..., ENamedThreads::GameThread)` + `WaitUntilTaskCompletes`（`FCSharpCompilerRunnable.cpp:213-242`）。**这一点经引擎源码核对后确认不是死锁**（见 §4-F-CMP-031 说明）。

**CrossVersion（Runtime 模块）** —— 纯宏模块，模块类只有两个空函数（`CrossVersion.cpp:7-16`），`IMPLEMENT_MODULE` 之外无任何逻辑；没有 `.cpp` 消费任何宏。101 个 UE 版本宏 + 5 个 .NET 宏 + 4 个 C++ 标准宏 + 2 个 VS 宏被其余 6 个模块共用（详见 §2.6 交叉核对表）。

**SourceCodeGenerator** —— `.uplugin` 把它注册为 `"Type": "Program"`（`UnrealCSharp.uplugin:49-53`），但目录里**没有 `.Build.cs`、也没有 `.Target.cs`**，只有 `SourceCodeGenerator.ubtplugin.csproj` + `.cs`；真正的接线方式是 `.uplugin:17` 的 `"CanBeUsedWithUnrealHeaderTool": true` 配合 `[UnrealHeaderTool]` 类属性（`SourceCodeGenerator.cs:18`）与 `[UhtExporter(Name="UnrealCSharp", ModuleName="UnrealCSharp")]`（`SourceCodeGenerator.cs:21-22`）。它**不是运行时程序，而是 UHT 构建期导出器**，产物是 `Intermediate/Build/<Platform>/<Target>/Inc/<Module>/UHT/*.binding.inl` 与 `*.header.inl`。本工作区实测存在 **631 个** 这种文件（mtime 2026-05-18），文件头就是 `SourceCodeGenerator.cs:67-72` 的 `GeneratedHeaderComment` 原文 → **它确实在跑，不是死代码**（详见 §4）。

---

## 2. 关键调用链

1. 文件监听触发：`FEditorListener.cpp:630 RequestCompile()` → `642 FCSharpCompiler::Get().Compile(FileChanges)` → `FCSharpCompiler.cpp:52-58` → `Runnable->EnqueueTask(FileChanges)` → `FCSharpCompilerRunnable.cpp:128-144`（加锁、清空队列、`Tasks.Enqueue(true)`、`Event->Trigger()`）。
2. 后台线程执行：`FCSharpCompilerRunnable.cpp:52 Run()` → `78 DoWork()` → `156-162 Compile(lambda, bCompileInterop=false)` → `198 Compile()` → `413 SyncProcess(dotnet, "build \"Game.csproj\" ...")` → `FUnrealCSharpFunctionLibrary.cpp:1543 CreateProc` → 轮询 `1569-1574`。
3. 输出→进度：`SyncProcess:1557 ReadPipe` → `1564 InOnOutput(Output)` → `FCSharpCompilerRunnable.cpp:414-417 CompileProgress.SetOutput` → `FCSharpCompileProgress.cpp:39-55`（按 `\n` 切行）→ `73 ProcessLine` → `89 ParseLine` → 命中 `" -> "` 时调用 `121 GetUEName()` / `123 GetGameName()`。
4. 结束与重载：`SyncProcess:1580 GetProcReturnCode` → `1585 InOnComplete` → `FCSharpCompilerRunnable.cpp:395-411`（`bSucceeded = InReturnCode == 0`、`SetStage(Succeeded/Failed)`、失败时 `UE_LOG`）→ `207 OnCompile.Broadcast(bSucceeded)` → `213-242` 派发游戏线程任务 → `225-229 UnrealCSharpCoreModule.Deactivate()` / `InFunction(...)`(= `FDynamicGenerator::Generator`) / `Activate()`。
5. 立即编译（绕过队列）：`UnrealCSharpEditor.cpp:114`（控制台命令 `UnrealCSharp.Editor.Compile`）→ `FCSharpCompiler::Get().Compile(TFunction)` → `FCSharpCompiler.cpp:60-69` **直接调 `Runnable->Compile(...)`，在当前线程（游戏线程）同步执行**；同理由 `DynamicDataSource.cpp:43/51 ImmediatelyCompile()` → `FCSharpCompiler.cpp:44-50` → `Runnable->ImmediatelyDoWork()`。
6. 关闭与销毁：`FCSharpCompiler.cpp:13-27` 析构 → `17 Thread->Kill(true)` → 引擎 `MicrosoftRunnableThread.h:79-101`：先 `Runnable->Stop()`（只置 `bIsStopped` 并 `Trigger`，`FCSharpCompilerRunnable.cpp:92-100`），再 `WaitForSingleObject(Thread, INFINITE)`。
7. CrossVersion 宏消费示例：`FCSharpCompilerRunnable.cpp:13 UE_F_APP_STYLE_GET_BRUSH` → `UEVersion.h:64` → `UE_VERSION_START(5,1,0)`；`FMonoDomain.cpp:62/71/78 !DOTNET9 / DOTNET8` → `DotnetVersion.h:9-11` → `DOTNET_MAJOR_VERSION`（由 `Mono.Build.cs:13` 注入 10）。
8. 生成器（构建期）：UBT 加载 `Binaries/DotNET/UnrealBuildTool/Plugins/SourceCodeGenerator/` → `SourceCodeGenerator.cs:23 SourceCodeGeneratorExporter` → 读 `Config/DefaultUnrealCSharpEditorSetting.ini`（`:25-48`）→ `:82 Generate()` → `:161 QueueClassExports` → `:340 ExportClass` → `:788 SaveIfChanged` → `Factory.CommitOutput`。

---

## 2.5 专项核对表 A：进程 / 管道 / 句柄配对表

这是本报告要求"做全"的第一处产出。**结论先说：任务书预期的"经典泄漏（每个进程泄漏两个管道句柄 + 进程句柄）"在本插件中不存在**——`SyncProcess` 是全插件唯一的进程创建点（grep `SyncProcess` 命中 8 处 = 声明 1（`.h:272`）+ 定义 1（`.cpp:1520`）+ 调用 **6** 处，全部指向同一个实现；原文写"5 处调用"为计数错误，已修正），它在函数末尾无提前返回地关闭了全部三样资源。

| 资源 | 分配点（文件:行） | 释放点 | 所有提前退出/异常路径是否覆盖 | 结论 |
|---|---|---|---|---|
| 匿名管道 `ReadPipe`/`WritePipe` | `FUnrealCSharpFunctionLibrary.cpp:1531` `FPlatformProcess::CreatePipe(ReadPipe, WritePipe)` | `:1587` `FPlatformProcess::ClosePipe(ReadPipe, WritePipe)` | 函数体 `1531`→`1587` 之间**没有 `return`/`goto`/`break`**；唯一非直线控制流是 `:1585 InOnComplete(ReturnCode, Result)` 回调 | **无泄漏**。残留风险：若回调内抛 C++ 异常或 `check` 失败则跳过 1587/1589/1591（UE 默认关闭异常，风险 P3 级，见 §3-P3） |
| 子进程句柄 `ProcessHandle` | `:1543 FPlatformProcess::CreateProc(...)` | `:1591 FPlatformProcess::CloseProc(ProcessHandle)` | 同上，且 `CreateProc` 失败返回无效句柄时仍会走到 `CloseProc` | **无泄漏** |
| 子进程本体（`dotnet.exe`） | `:1543` | `:1589 FPlatformProcess::TerminateProc(ProcessHandle, true)` | **只在进程自然结束后**执行；整个 `SyncProcess` 没有任何"提前终止"路径（无 `bCancelled`、无超时、无外部 `TerminateProc`） | **异常/挂起时无人回收**：dotnet 卡死 → 编译线程与其子进程树永久存活 → 见 §3-F-CMP-001/002 |
| 轮询期间的读取缓冲 `FString Result` | `:1529` + `:1560 Result.Append(Output)` | 随函数返回析构 | — | 无泄漏，但**无上限累积**整个构建日志（P3） |
| `FEvent* Event` | `FCSharpCompilerRunnable.cpp:47 GetSynchEventFromPool(true)` | `:106 ReturnSynchEventToPool(Event)`（`Exit()`） | `Exit()` 只在 `Run()` 返回后由线程框架调用；`Stop()`（`:92-100`）**不归还** | 正常路径无泄漏；`Run()` 若永不返回（挂在 `SyncProcess`）→ `Exit()` 永不执行 → **Event 与线程一起泄漏到进程结束** |
| `FCSharpCompilerRunnable* Runnable` | `FCSharpCompiler.cpp:8 new` | `:23 delete`（仅析构函数） | 单例无中途销毁路径 | 无泄漏 |
| `FRunnableThread* Thread` | `FCSharpCompiler.cpp:10 Create` | `:19 delete`，`Kill(true)` 之后 | 同上 | 无泄漏，但 `Kill(true)` 会**无限等待**（§3-F-CMP-001） |
| `TSharedPtr<SNotificationItem> NotificationItem` | `FCSharpCompilerRunnable.cpp:354 AddNotification` | `:430 Fadeout()` + `:432 Reset()`（游戏线程 `AsyncTask`） | `:421-424 GExitPurge` 时提前 `return` → 不移除通知 | 无泄漏；退出期残留通知 |
| `FTSTicker::FDelegateHandle ProgressTickerHandle` | `:360 AddTicker` | `:358`/`:426 RemoveTicker` | `:421-424 GExitPurge` 提前返回 → 不移除 | 无泄漏（lambda 自己 `return false` 自注销，`:364-367`） |
| `OnBeginGenerator/OnEndGenerator` 委托句柄 | `:25-29 AddRaw` | `:34-42 Remove`（析构） | 均先 `IsValid()` 判断 | 配对正确，但**析构发生在静态销毁期**（§3-F-CMP-011） |
| `CompileProgress` 内的 `FCriticalSection` | `FCSharpCompileProgress.h:45` | 随对象销毁 | — | 配对正确 |

**补充事实（引擎侧核对，用于判定上面的"无泄漏"不是巧合）**：`FWindowsPlatformProcess::CreateProc` 会把 `"\"%s\" %s"` 格式化成命令行后交给 `CreateProcessW`（`WindowsPlatformProcess.cpp:528-531`），并把 `STARTUPINFO.hStdOutput/hStdError` 指向同一根管道的写端（`:520-522`、`:465-467`），`bInheritHandles` 由 `STARTF_USESTDHANDLES` 决定（`:503-525`）；`FWindowsPlatformProcess::CreatePipe(ReadPipe, WritePipe, bWritePipeLocal = false)` 的默认值在 **5.6 是 `false`**（`WindowsPlatformProcess.h:198`），即引擎**已经**清掉 `ReadPipe` 的继承位、保留 `WritePipe` 可继承（`WindowsPlatformProcess.cpp:1832-1847`）。因此本插件 `FUnrealCSharpFunctionLibrary.cpp:1533-1539` 那段被 `UE_SET_HANDLE_INFORMATION` 门控的 `SetHandleInformation` 在 ≤5.6 上属于**冗余但无害**（详见 §2.6 与 §3-F-CMP-027）。

---

## 2.6 专项核对表 B：版本宏一致性交叉核对表

这是本报告要求"做全"的第二处产出。

### B-1 版本比较宏的语义（与引擎源码逐字核对）

| 宏 | 定义位置 | 展开语义 | 与引擎宏对比 | 边界正确性 |
|---|---|---|---|---|
| `UE_VERSION_START(M,m,p)` | `UEVersion.h:5-6` | `UE_GREATER_SORT(MAJOR,M,UE_GREATER_SORT(MINOR,m,UE_GREATER_SORT(PATCH,p,true)))` = **`引擎版本 >= M.m.p`** | 与引擎 `UE_VERSION_NEWER_THAN_OR_EQUAL`（`EngineVersionComparison.h:14-15`，最内层 tie-breaker 也是 `true`）**逐字等价** | **无 off-by-one**：`UE_VERSION_START(5,6,0)` 在 5.6.0 上为真，符合"该特性自 5.6 起可用"的用法；`!UE_VERSION_START(...)` 即 `<`，取反也正确 |
| 引擎 `UE_VERSION_NEWER_THAN` | `EngineVersionComparison.h:19-20` | 最内层 tie-breaker 为 `false` = **严格大于** | 与本插件宏只差一个布尔，极易混淆 | 本插件宏**没有**误用，但命名 `..._START` 与引擎 `..._NEWER_THAN` 形式相似（P3，见 §3-F-CMP-032） |
| `DOTNET_VERSION_START` | `DotnetVersion.h:6-7` | 与上同构（tie-breaker `true`）= **`>= M.m.p`** | 无引擎对应物 | 语义自洽；`DOTNET8/9/10` 分别 = `>=8.0.0 / >=9.0.0 / >=10.0.0` |
| `VS2022` / `VS2026` | `VSVersion.h:4,6` / `:8,10` | MSVC：`1930<=_MSC_VER<1950` / `_MSC_VER>=1950`；非 MSVC：恒 `0` | 无引擎对应物 | 边界合理（VS2019=192x 被排除）；非 Windows/clang 下为 `0`，`#if !VS2026` 会走进"非 VS2026"分支，行为与 Windows 一致（无平台错路，见 §3-F-CMP-029 说明） |
| `STD_CPP_11/14/17/20` | `CppVersion.h:3,5,7,9` | `__cplusplus >= 201103L/201402L/201703L/202002L` | 无引擎对应物 | 依赖 UBT 的 `/Zc:__cplusplus`（`VCToolChain.cs:671-674`，由 `Target.WindowsPlatform.bUpdatedCPPMacro` 控制），存在"MSVC 下恒假"风险 → §3-F-CMP-013 |

### B-2 `DOTNET_VERSION` ↔ `.csproj TargetFramework` ↔ 捆绑运行时 一致性

| 环节 | 位置 | 实测值 | 一致性判定 |
|---|---|---|---|
| 宏定义输入 | `Source/ThirdParty/CoreCLR/CoreCLR.Build.cs:13-24` | `10 / 0 / 4` | — |
| 宏定义输入 | `Source/ThirdParty/LeanCLR/LeanCLR.Build.cs:11-22` | `10 / 0 / 4` | — |
| 宏定义输入 | `Source/ThirdParty/Mono/Mono.Build.cs:13-15` | `10 / 0 / 1` | — |
| 哪个后端生效 | `Config/DefaultUnrealCSharpSetting.ini`（`WindowsScriptDomainType=LeanCLR`）→ `UnrealCSharpCore.build.cs:263-306` | **LeanCLR** → `DOTNET_MAJOR/MINOR/PATCH_VERSION = 10.0.4` | ✅ 与下方 TargetFramework 主版本一致 |
| 运行时目录 | `CoreCLR.Build.cs:40` `shared/Microsoft.NETCore.App/10.0.4` | 10.0.4 | ✅ |
| TargetFramework 生成公式 | `FSolutionGenerator.cpp:258-268`：`net{GetDotnetVersion()}.{DOTNET_MINOR_VERSION}` | `net10.0` | ✅ |
| `GetDotnetVersion()` | `FUnrealCSharpFunctionLibrary.cpp:1395-1405` + `UnrealCSharpSetting.cpp:26`（默认 `EDotnetVersion::Latest`）+ `UnrealCSharpSetting.h:66-72`（`V8=8,V9=9,V10=10,Latest`→11） | `Latest-1 = 10` | ✅（但依赖"枚举末项恒为下一版本"这一隐含假设 → P3 `F-CMP-015`） |
| 生成物实测（工程根） | `Script/Shared.props:8`、`Script/CodeAnalysis/CodeAnalysis.csproj:9`、`Script/Interop/Interop.csproj:15` | 均为 `net10.0` | ✅ **端到端一致** |
| 模板占位 | `Template/Shared.props:3`、`Template/Interop.csproj:10`、`Script/CodeAnalysis/CodeAnalysis.csproj:4` | 空 `<TargetFramework></TargetFramework>` | ⚠️ 必须由 `FSolutionGenerator` 在生成时替换（`FSolutionGenerator.cpp:15-22,115-124`）；**未被替换的源文件本身是不可构建的** |
| 分析器/织入器（不受 DOTNET_VERSION 管） | `Script/SourceGenerator/SourceGenerator.csproj:3`、`Script/Weavers/Weavers.csproj:4` | `netstandard2.0` | ✅ 与 `Template/Game.props:10` 引用的 `..\Weavers\bin\$(Configuration)\netstandard2.0\Weavers.dll` 路径一致 |
| UHT 宿主 | `SourceCodeGenerator.ubtplugin.csproj:5` | `net8.0` | ✅ 与游戏程序集无关（UBT 自身 TFM），但与 `Engine` 绑定（`.props:4` 硬编码绝对路径，换机需改） |
| 配置可选值风险 | 用户在 ini 里把 `DotnetVersion` 设为 `V8`/`V9` | 会生成 `net8.0`/`net9.0`，而捆绑运行时仅为 `10.0.x` | ❌ **不一致**，跨大版本 roll-forward 默认不允许 → 运行时加载失败风险（§3-F-CMP-014） |

### B-3 `DOTNET8/9/10` 的实际可达性（死分支核对）

| 宏 | 唯一消费者 | 在该消费者处的常量上下文 | 可达性 |
|---|---|---|---|
| `DOTNET9` | `FMonoDomain.cpp:62 #if !DOTNET9`、`:78 #if DOTNET9` | `FMonoDomain.cpp:2 #if WITH_MONO`；`WITH_MONO=1` 时必然依赖 `Mono` 模块（`UnrealCSharpCore.build.cs:286-290`）→ `DOTNET_MAJOR_VERSION=10`（`Mono.Build.cs:13`） | `DOTNET9` **恒真**；`:62` 的 `!DOTNET9` 分支 **恒假（死）** |
| `DOTNET8` | `FMonoDomain.cpp:71 #if DOTNET8`（嵌套在 `:62 !DOTNET9` 之内） | 同上 | **双重恒假（死）**（§3-F-CMP-016） |
| `DOTNET10` | 无消费者 | — | 定义存在但从未被使用（§4 死代码清单） |

**结论**：`DotnetVersion.h` 的宏在数值上和 `.csproj` 的 `net10.0` 自洽，但 `FMonoDomain.cpp:62-73` 这段 iOS/AOT 的 `mono_dllmap_insert` 注册代码在当前 `Mono.Build.cs`（硬编码 10.0.1）下**永远不会被编译进去**——要么是死代码，要么说明 `Mono.Build.cs` 的版本号已过期一个"语义代"。

---

## 3. 发现清单

### 3.0 逐函数清单（无问题函数一行式）

> 说明：本表覆盖我负责文件的**每一个函数**；有独立 Finding 的函数在"结论"列标注编号。"调用方"均给出 `文件:行`。

**`FCSharpCompilerRunnable`（`FCSharpCompilerRunnable.cpp` / `.h`）**

| 函数 | 做了什么 | 调用方 | 结论 |
|---|---|---|---|
| `FCSharpCompilerRunnable()` `:19-30` | 初始化成员，`AddRaw` 两个生成器委托 | `FCSharpCompiler.cpp:8` | 无问题（`bIsCompiling` 用 `std::atomic` ✓） |
| `~FCSharpCompilerRunnable()` `:32-43` | 移除两个委托 | 静态销毁期（`FCSharpCompiler.cpp:23 delete`） | `F-CMP-011` |
| `Init()` `:45-50` | `GetSynchEventFromPool(true)` | 线程框架 `Create` 流程 | 无问题 |
| `Run()` `:52-90` | 轮询任务队列/等待事件 | 线程框架 | `F-CMP-004`/`F-CMP-012`（锁与原子性）；`Tasks.IsEmpty()` 的双检在 SPSC 契约内，**不构成队列竞争**（已核对 UE `TQueue` SPSC 用法） |
| `Stop()` `:92-100` | 置 `bIsStopped`、`Trigger` | 引擎 `MicrosoftRunnableThread.h:87` | `F-CMP-001`（不终止子进程） |
| `Exit()` `:102-110` | 归还 `Event` | 线程框架 | 无问题（但见句柄表：挂起时永不执行） |
| `EnqueueTask()` `:112-126` | 清队列→入队→`Trigger`（去重语义） | `FCSharpCompiler.cpp:40` | 语义正确；`Event` 无空判 → 归入 `F-CMP-011`（退出期竞态） |
| `EnqueueTask(TArray<FFileChangeData>)` `:128-144` | 加锁清队列、追加文件变更、入队 | `FCSharpCompiler.cpp:56` | `F-CMP-004`（`FileChanges` 的另一半无锁） |
| `IsCompiling()` `:146-149` | `bIsCompiling || !Tasks.IsEmpty()` | `FCSharpCompiler.cpp:73`、`FEditorListener.cpp:199/213/468/630/718` | 无问题（未被同步路径使用才是问题 → `F-CMP-003`） |
| `GetCompileProgress()` `:151-154` | 转发 `CompileProgress.GetStage()` | `FCSharpCompiler.cpp:78`、`FEditorListener.cpp:729` | 无问题（跨线程读取已被锁保护） |
| `DoWork()` `:156-162` | 以**默认参数** `bCompileInterop=false` 调 `Compile` | `Run():78` | `F-CMP-006` |
| `ImmediatelyDoWork()` `:164-170` | `bCompileInterop=true, bReloadImmediately=true` | `FCSharpCompiler.cpp:48` | 无问题（但它的调用者绕过队列 → `F-CMP-003`/`005`） |
| `Compile(TFunction,...)` `:172-248` | 总控：置阶段→可选 Interop→主编译→广播→派发游戏线程任务 | `DoWork`、`ImmediatelyDoWork`、`FCSharpCompiler.cpp:64` | `F-CMP-003`/`004`/`005`/`012` |
| `GetBuildConfiguration()` `:250-263` | 运行 Cook 时读 `RuntimeConfiguration`，否则 `EditorConfiguration`，映射为 `Debug`/`Release` | `:285`、`:390` | 无问题；注意 `ESolutionConfiguration`（`UnrealCSharpEditorSetting.h:7-11`）**只有 Debug/Release**，不存在与 UE 的 Development/Shipping 映射（见 §5 维度表） |
| `CompileInterop()` `:265-314` | 若 Interop 工程存在且产物不存在（或强制）则 `dotnet build` | `:185` | 无问题（逻辑正确）；`:284` 重复调用 `GetInteropProjectPath()` 而非复用 `:267` 的局部量（P3 风格） |
| `Compile()` `:316-437` | 主编译：`dotnet build Game.csproj`，显示/隐藏通知，返回 `bSucceeded` | `:198` | `F-CMP-007`（失败无诊断）、`F-CMP-008`；**退出码判断正确**（`:397 bSucceeded = InReturnCode == 0`，任务书担心的"不识别非 0"**不成立**） |
| `ShowCompileResultNotification()` `:439-465` | `AsyncTask(GameThread)` 推成功/失败通知 | `:189`、`:410` | 无问题（**已正确 marshal 回游戏线程**） |
| `OnBeginGenerator()` `:467-474` | `bIsGenerating=true`、清 `Tasks`/`FileChanges`（**无锁**） | 委托广播 `UnrealCSharpEditor.cpp:318` | `F-CMP-004` |
| `OnEndGenerator()` `:476-483` | `bIsGenerating=false`、清 `Tasks`/`FileChanges`（**无锁**） | 委托广播 `UnrealCSharpEditor.cpp:392` | `F-CMP-004` |

**`FCSharpCompileProgress`**

| 函数 | 做了什么 | 调用方 | 结论 |
|---|---|---|---|
| `SetStage()` `FCSharpCompileProgress.cpp:21-37` | 加锁写 `Stage`、按阶段更新 `CompilePhase`、清 `PendingOutput` | `FCSharpCompilerRunnable.cpp:183/187/277/323/401` | 无问题（锁保护完整） |
| `SetOutput()` `:39-55` | 加锁追加并按 `\n` 切行→`ProcessLine` | `FCSharpCompilerRunnable.cpp:306/416` | 无问题（`\r` 由 `ProcessLine` 修剪）；`PendingOutput` 无上限（P3） |
| `Flush()` `:57-64` | 加锁处理残余行 | `:296/399` | 无问题 |
| `GetStage()` `:66-71` | 加锁读 `Stage` | `FCSharpCompilerRunnable.cpp:153` | 无问题；`const_cast` 加锁是风格问题（`F-CMP-025`） |
| `ProcessLine()` `:73-87` | 修剪空白→`ParseLine` | `:53/61` | 无问题 |
| `ParseLine()` `:89-136` | 用 5 类文本规则推断阶段，命中 `" -> "` 时比较工程名 | `:81` | `F-CMP-010`（锁内 `GetMutableDefault`）、`F-CMP-009`（英文文本依赖）、`F-CMP-029`（命名与语义） |

**`FCSharpCompiler`**

| 函数 | 做了什么 | 调用方 | 结论 |
|---|---|---|---|
| `FCSharpCompiler()` `FCSharpCompiler.cpp:4-11` | `new Runnable` + 创建线程 | `Get():31` | 无问题 |
| `~FCSharpCompiler()` `:13-27` | `Kill(true)` → delete | 静态销毁期 | `F-CMP-001`/`011` |
| `Get()` `:29-34` | 函数局部静态单例 | 16 处（见 §2） | 单例性正确；销毁时序见 `F-CMP-011` |
| `Compile()` `:36-42` | 异步入队 | `UnrealCSharpEditor.cpp:266`、`FEditorListener.cpp:548/648` | 无问题 |
| `ImmediatelyCompile()` `:44-50` | **当前线程同步**执行 | `UnrealCSharpEditor.cpp:388`、`DynamicDataSource.cpp:43/51` | `F-CMP-003`/`005` |
| `Compile(TArray<FFileChangeData>)` `:52-58` | 异步入队 | `FEditorListener.cpp:642` | 无问题 |
| `Compile(TFunction)` `:60-69` | **当前线程同步**执行 | `UnrealCSharpEditor.cpp:114`（控制台命令） | `F-CMP-003`/`005` |
| `IsCompiling()` `:71-74` / `GetCompileProgress()` `:76-79` | 空指针后转发 | `FEditorListener.cpp:630/729` | 无问题 |

**模块壳与构建脚本**

| 符号 | 做了什么 | 结论 |
|---|---|---|
| `FCompilerModule::StartupModule/ShutdownModule` `Compiler.cpp:7-16` | 空实现 | 无问题（模块只为承载 `FCSharpCompiler` 编译单元；任务书问的"是否需要 StartupModule"——**不需要**） |
| `FCrossVersionModule::StartupModule/ShutdownModule` `CrossVersion.cpp:7-16` | 空实现 | 无问题（纯宏模块，无入口逻辑） |
| `Compiler.Build.cs` | 依赖 `Core/DirectoryWatcher/CoreUObject/Engine/Slate/SlateCore/Json/UnrealCSharpCore/CrossVersion/EditorStyle` | `F-CMP-028`（`EditorStyle` 在 ≥5.1 已无用；**修正：原文误写为 `F-CMP-030`**） |
| `CrossVersion.build.cs` | 仅 `Core/CoreUObject/Engine/Slate/SlateCore` | 无问题 |

**`SourceCodeGenerator.cs`（C#）**

| 函数 | 做了什么 | 调用方 | 结论 |
|---|---|---|---|
| `SourceCodeGeneratorExporter(IUhtExportFactory)` `:23-49` | UHT 入口：读 ini → 若 `bEnableExport` 为真则 `Generate()` | UBT/UHT（`CanBeUsedWithUnrealHeaderTool`） | `F-CMP-017`（`bool.Parse` 无保护）、`F-CMP-021`（静默禁用） |
| `SourceCodeGenerator(factory)` `:74-77` | 存 factory | `:44` | 无问题 |
| `Generate()` `:82-172` | 扫描模块表→读 `ExportModule`/`ClassBlacklist`→按模块排队导出→`Task.WaitAll`→`Finish()` | `:44` | `F-CMP-018`（空 `ExportModule` 语义）、`F-CMP-019`（目录不存在抛异常）、`F-CMP-020`（`File.Exists`/`ConfigFile` 无保护） |
| `QueueClassExports(range)` `:179-193` | 递归收集 `UhtClass` 并 `Factory.CreateTask` | `:161` | 无问题（递归深度=类型树深度） |
| `CanExportClass()` `:200-205` | 黑名单 + 非接口 + 类型受支持 | `:183` | 无问题 |
| `CanExportFunction()` `:212-266` | 8 条过滤规则 | `:433` | 无问题（逻辑完备性无法用静态阅读完全确认，见 §5） |
| `CanExportProperty()` `:273-279` | 4 条过滤规则 | `:419` | 无问题 |
| `IsClassTypeSupported()` `:281-287` | RequiredAPI/MinimalAPI 等判定 | `:204/293` | 无问题 |
| `IsPropertyTypeSupported()` `:289-334` | 递归容器/结构体/委托类型判定 | `:259/319/324/329/330` | 无问题 |
| `ExportClass(UhtClass)` `:340-352` | 借用 StringBuilder→写 `.binding.inl`→入队 | `:185` | 无问题 |
| `Finish()` `:354-389` | 按包聚合→排序→写 `.header.inl` | `:171` | `F-CMP-022`（`Dictionary` 遍历顺序） |
| `ExportClass(StringBuilder, UhtClass)` `:391-472` | 生成注册结构体文本，返回"是否有内容" | `:344` | 无问题（`:456 Remove(len-2,2)` 安全性已核对：该分支的 body 必以 `\r\n` 结尾） |
| `GetDependencyClasses()` `:474-512` | 收集依赖类 | `:421/439` | `F-CMP-022`（`HashSet` 遍历顺序） |
| `ExportFunction()` `:514-535` | 追加 `.Function(...)` 行 | `:443` | 无问题（`:519 HasParameters` 判据已与 UHT 源码核对，见下） |
| `ExportProperty()` `:537-541` | 追加 `.Property(...)` 行 | `:423` | 无问题 |
| `GetParamPropertySignature()` `:543-628` | 生成形参签名（const/&/枚举 CppType） | `:685` | 无问题（规则复杂，未逐条验证语义正确性） |
| `GetReturnPropertySignature()` `:630-659` | 生成返回类型 | `:666` | 无问题 |
| `GetFunctionSignature()` `:662-705` | 拼 `ret(Class::*)(args) [const]` | `:517` | 无问题（`:692-695 Remove(len-2,2)`：`UhtFunction.HasParameters` 定义为 `Children.Count>0 && !Children[0].ReturnParm`（引擎 `UhtFunction.cs:375`），为真时循环必已追加 `", "` → 安全） |
| `GetFunctionParamName()` `:707-729` | 拼 `TArray<FString>{...}`（无守卫的 `Remove(len-2,2)`） | `:521` | 无问题（同上，`HasParameters` 保证至少一个非返回参数） |
| `GetFunctionDefaultValue()` `:731-771` | 拼 BlueprintCallable 默认值，`NULL`→`nullptr` | `:531` | 无问题 |
| `GetHeaderFile()` `:773-776` | `Path.Combine(HeaderPath[...], classObj.HeaderFile.FilePath)` | `:785` | `F-CMP-018`（索引器 + `Path.Combine` 绝对路径吞噬第一参数） |
| `GenerateInclude()` `:778-781` | 生成 `#include "..."` | `:382/785` | `F-CMP-018`（绝对路径）、`F-CMP-023`（无 BOM） |
| `GetInclude()` `:783-786` | 转发 | `:451` | 同上 |
| `SaveIfChanged()` `:788-796` | 内容相同则跳过，否则 `Factory.CommitOutput` | `:348/387` | `F-CMP-020`（`File.ReadAllText` 无保护）、`F-CMP-023`（编码） |
| `GetPlugins(dir, dict)` `:798-822` | 扫 `*.uplugin` 建 插件名→目录 表 | `:92/104` | `F-CMP-019`（目录不存在抛 `DirectoryNotFoundException`） |
| `GetModules(dir, dict)` `:824-848` | 扫 `*.Build.cs` 建 模块名→目录 表 | `:86-102` | `F-CMP-019`；`:834/844` 同名模块后者覆盖前者（与 `TMap` 语义相同，非缺陷） |
| `GetModules(dir, HashSet)` `:850-871` | 只收集模块名 | `:86` | `F-CMP-024`（唯一使用点是把结果丢进只写不读的 `Project` 字段） |

---

### P0

**无。** 本次未发现可复现的崩溃或数据损坏（已按规范逐维度检查：空指针、越界、句柄配对、锁、原子性、退出码、编码、平台分支）。

---

### P1

#### [F-CMP-001] 编译无法取消：`Stop()` 不终止 `dotnet` 子进程，关闭编辑器时 `Kill(true)` 无限等待

- **类别**: 并发/线程安全（生命周期）
- **严重度**: **P2**
- **复核结论**: 部分确认（偏差：级别。原文按 P1 计；`Kill(true)` 只在"关闭编辑器时恰有编译在跑"这一窄窗口阻塞，不产生数据损坏，属可恢复的关闭迟滞 → P2）
- **可达性**: 活跃（五平台 LeanCLR 配置下，`bEnableCompiled=true` 默认开启编译，关闭编辑器即走该析构路径）
- **复核证据**: `Source/Compiler/Private/FCSharpCompilerRunnable.cpp:92-100`（`Stop()` 全文只有 `bIsStopped=true` + `Event->Trigger()`，已读）；`bIsStopped` 唯一读取点 `:56`（已读）；`Source/Compiler/Private/FCSharpCompiler.cpp:17` `Thread->Kill(true)`（已读）；`Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1569-1574` 轮询循环无取消检查（已读）
- **级别变动**: P1→P2（触发条件窄、无数据损坏、无泄漏，仅关闭迟滞）
- **文件**: `Source/Compiler/Private/FCSharpCompilerRunnable.cpp:92-100`、`Source/Compiler/Private/FCSharpCompiler.cpp:17`、`Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1569-1591`
- **函数**: `FCSharpCompilerRunnable::Stop()`、`FCSharpCompiler::~FCSharpCompiler()`
- **置信度**: 高（引擎侧 `Kill` 语义已读源码确认）

**现状（代码事实）**
```cpp
// FCSharpCompilerRunnable.cpp:92
void FCSharpCompilerRunnable::Stop()
{
	bIsStopped = true;                 // 只置标志
	if (Event != nullptr)
	{
		Event->Trigger();              // 只唤醒等待，不碰子进程
	}
}
```
```cpp
// FCSharpCompiler.cpp:15
	if (Thread != nullptr)
	{
		Thread->Kill(true);            // → Stop() 之后 WaitForSingleObject(Thread, INFINITE)
```
```cpp
// FUnrealCSharpFunctionLibrary.cpp:1569  —— 轮询循环里没有 bIsStopped / 取消检查
	while (ProcessHandle.IsValid() && FPlatformProcess::IsProcRunning(ProcessHandle))
	{
		FPlatformProcess::Sleep(0.01f);
		ReadOutput();
	}
```
引擎侧：`MicrosoftRunnableThread.h:79-101` → `Runnable->Stop();` 然后 `WaitForSingleObject(Thread, INFINITE);`。

**调用上下文**
`Stop()` 的唯一调用者是引擎的 `Kill`（`MicrosoftRunnableThread.h:87`），而 `Kill(true)` 的唯一调用者是 `FCSharpCompiler::~FCSharpCompiler`（`FCSharpCompiler.cpp:17`）。全插件 grep `->Stop()|Kill(|TerminateProc` 命中 6 处，**没有任何 UI 取消按钮/命令**（`SCompileProgressDialog` 无 `SButton`，grep 0 命中）。

**问题**
`bIsStopped` 只在 `Run()` 的循环顶部（`FCSharpCompilerRunnable.cpp:56`）被读取；一旦线程进入 `Compile()`→`SyncProcess()` 的轮询循环，它就**再也不看这个标志**，也没有任何代码路径去 `TerminateProc` 那个 dotnet 进程。因此：编辑器关闭时若正在编译，`Kill(true)` 会一直阻塞到 dotnet 自然退出（大工程可达数十秒到数分钟），用户看到的是"编辑器关不掉"；同时 `Runnable->Exit()` 不会执行，`FEvent` 也不归还池（见 §2.5 句柄表）。

**建议**
1. 在 `FCSharpCompilerRunnable` 中保存"当前子进程句柄"，`Stop()` 里对它 `FPlatformProcess::TerminateProc(Handle, true)`；`SyncProcess` 增加一个 `const TFunction<bool()>& InShouldCancel`（或 `std::atomic<bool>*`）参数，在 `:1569` 的循环条件中检查，取消时 `TerminateProc` 并以 `ReturnCode = -2`（"cancelled"）回调。
2. 析构路径把 `Kill(true)` 改成 `Thread->Kill(false)` + 有超时的 `WaitForCompletion`，避免无限等待。

**验证方式**
`grep -n "Stop()" Source/Compiler -r` 确认唯一入口；在编辑器中启动一次编译后在编译途中直接关闭编辑器，观察是否卡住；或加 `UE_LOG` 打印 `Stop()` 与 `SyncProcess` 退出的时间差。

---

#### [F-CMP-002] `SyncProcess` 完全没有超时保护，`dotnet` 一旦卡住就是永久挂起

- **类别**: 并发/线程安全（挂死）
- **严重度**: **P1**
- **复核结论**: 确认
- **可达性**: 活跃（`FCSharpCompilerRunnable.cpp:291`/`:413` 两条编译路径每次编译都会走到 `SyncProcess`）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1520-1592` 全函数已读：唯一循环条件在 `:1569`，函数体内**无任何超时变量/静默检测/外部中断点**。调用点 grep `SyncProcess` 命中 **8** 处 = 声明 1（`.h:272`）+ 定义 1（`.cpp:1520`）+ 调用 **6**（`FCSharpCompilerRunnable.cpp:291`、`:413`、`FCodeAnalysis.cpp:34/49/86`、`UnrealCSharpEditorSetting.cpp:152`）
- **级别变动**: 无（**但原文"5 个调用点"计数错误，实为 6 个**，已就地修正）
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1569-1574`（调用点：`Source/Compiler/Private/FCSharpCompilerRunnable.cpp:291`、`413`）
- **函数**: `FUnrealCSharpFunctionLibrary::SyncProcess`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FUnrealCSharpFunctionLibrary.cpp:1569
	while (ProcessHandle.IsValid() && FPlatformProcess::IsProcRunning(ProcessHandle))
	{
		FPlatformProcess::Sleep(0.01f);

		ReadOutput();
	}
```
整个函数（1520-1592）只有上面这一处循环条件，**没有时间上限、没有输出静默检测、没有可中断点**。

**调用上下文**
5 个调用点全部是同步等待：`FCSharpCompilerRunnable.cpp:291`（Interop 编译）、`:413`（游戏程序集编译）、`FCodeAnalysis.cpp:34/49/86`、`UnrealCSharpEditorSetting.cpp:152`（`GetDotNetPathArray()` → 设置界面下拉框，**游戏线程**）。

**问题**
`dotnet build` 会（a）联网还原 NuGet 包，（b）等待 `VBCSCompiler`/MSBuild 节点锁，（c）在杀毒软件扫描时长时间静默。任一情形成立时，`IsProcRunning` 一直为真，循环永不退出：编译线程永久占用，`bIsCompiling` 永远为真 → `FEditorListener.cpp:718` 的 `while (FCSharpCompiler::Get().IsCompiling())` 会**永久自旋**（该循环只在编译结束时退出，见 `FEditorListener.cpp:718-745`），编辑器实际上被锁在一个模态进度窗里，且因为 `F-CMP-001` 连关闭都不干净。设置界面调 `GetDotNetPathArray()`（`UnrealCSharpEditorSetting.cpp:152`）时同样会冻结。

**建议**
在 `SyncProcess` 加两个阈值：总超时（如 600s，可用 ini 配置）与"无输出静默超时"（如 120s）；超时后 `TerminateProc` 并以 `ReturnCode = -1` 回调，同时把这一信息拼进 `InResult`。可复用 `FPlatformTime::Seconds()`。

**验证方式**
用一个假的可执行文件（如 `cmd /c pause`）替换 `GetDotNet()` 返回值，观察调用方是否永久阻塞；或 grep 确认 `SyncProcess` 内无任何超时常量。

---

#### [F-CMP-003] 同步编译路径绕过队列与 `IsCompiling()` 守卫，可同时跑两个 `dotnet build` 写同一输出目录

- **类别**: 并发/线程安全
- **严重度**: **P2**
- **复核结论**: 部分确认（偏差一：调用点路径写错——资源变更回调是 `UnrealCSharpEditor.cpp:388`，**不是** `FEditorListener.cpp:388`；偏差二：级别由 P1 校正为 P2）
- **可达性**: 活跃（右键菜单 / 控制台命令都是在游戏线程直调同步入口）
- **复核证据**: `Source/Compiler/Private/FCSharpCompiler.cpp:44-50`、`:60-69` 已读（均直接调 `Runnable->ImmediatelyDoWork/Compile`，无 `IsCompiling()` 守卫）；grep `ImmediatelyCompile` 命中调用点 **3** 处：`UnrealCSharpEditor.cpp:388`、`DynamicDataSource.cpp:43`、`DynamicDataSource.cpp:51`（另有声明 `FCSharpCompiler.h:18`、定义 `FCSharpCompiler.cpp:44`）；grep `IsCompiling()` 全部命中：`FEditorListener.cpp:199/213/468/630/718`、`FCSharpCompiler.cpp:73`、`FCSharpCompilerRunnable.cpp:364`（自用）——**同步入口处确实一个都没有**
- **级别变动**: P1→P2（需用户在一次编译进行中再次手动触发同一入口；后果为 MSBuild 文件锁报错 / 构建失败，非静默数据损坏）
- **文件**: `Source/Compiler/Private/FCSharpCompiler.cpp:44-50`、`:60-69`；触发点 `Source/UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp:114`、`Source/UnrealCSharpEditor/Private/ContentBrowser/DynamicDataSource.cpp:43/51`、`Source/UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp:388`
- **函数**: `FCSharpCompiler::ImmediatelyCompile()`、`FCSharpCompiler::Compile(const TFunction<void()>&)`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FCSharpCompiler.cpp:44
void FCSharpCompiler::ImmediatelyCompile(const bool bForceCompileInterop) const
{
	if (Runnable != nullptr)
	{
		Runnable->ImmediatelyDoWork(bForceCompileInterop);   // ← 直接执行，不进 Tasks 队列
	}
}
// FCSharpCompiler.cpp:60
void FCSharpCompiler::Compile(const TFunction<void()>& InFunction) const
{
	if (Runnable != nullptr)
	{
		Runnable->Compile([InFunction](const TArray<FFileChangeData>&) { InFunction(); });  // ← 当前线程同步
	}
}
```
而队列路径（`:36-42`、`:52-58`）只是 `EnqueueTask()`。两者之间没有任何互斥，`IsCompiling()` 也**没有**在这两个同步入口里被检查。

**调用上下文**
`UnrealCSharpEditor.cpp:109-117` 注册控制台命令 `UnrealCSharp.Editor.Compile` → `:114`；`DynamicDataSource.cpp:43/51` 在内容浏览器右键菜单里调 `ImmediatelyCompile()`；`FEditorListener.cpp:388` 在资源变更回调里调 `ImmediatelyCompile(bForceCompileInterop)`。这些都发生在**游戏线程**。全插件 grep `IsCompiling()` 的命中只有 `FEditorListener.cpp:199/213/468/630/718` 和 `FCSharpCompiler.cpp:73`（转发）——即只有监听器在防重入。

**问题**
用户在后台编译进行中（例如文件监听已经排好队）再点一次右键菜单/再敲一次控制台命令，就会有**两个线程同时执行 `dotnet build`**，目标都是同一批路径（`Script/Game/Game.csproj`、`Script/UE/UE.csproj`，输出到工程根 `Script/*/bin`、`Content/Script/*.dll`）。dotnet/MSBuild 对同一 obj/bin 目录的并发写入会产生文件锁冲突（`MSB3021`/`MSB3027`）、产物半写、以及 `bIsCompiling` 被先完成者置回 `false` 导致进度通知提前消失（`FCSharpCompilerRunnable.cpp:191/245`）。`Tasks` 队列自身的去重（`EnqueueTask` 先 `Empty()` 再 `Enqueue`，`:117-122`）只对队列路径有效，挡不住同步路径。

**建议**
在 `ImmediatelyCompile`/`Compile(TFunction)` 入口加 `if (IsCompiling()) { return false; }`，或把同步语义改为"入队并等待完成"（入队后 `Event` 超时等待 + 结果查询），彻底收敛到单执行者。

**验证方式**
在编辑器里同时触发 `UnrealCSharp.Editor.Compile` 与内容浏览器菜单的"立即编译"，观察是否有两个 `dotnet.exe` 并存（任务管理器 / `Process Explorer`），或给 `SyncProcess` 开头加 `UE_LOG` 打印线程 id 观察重叠。

---

#### [F-CMP-004] `FileChanges`/`Tasks` 在生成器回调中被无锁修改，与编译线程构成数据竞争

- **类别**: 并发/线程安全
- **严重度**: **P1**
- **复核结论**: 确认
- **可达性**: 活跃
- **复核证据**: `Source/Compiler/Private/FCSharpCompilerRunnable.cpp:467-483` 已读：`OnBeginGenerator`/`OnEndGenerator` 各写 `bIsGenerating` + `Tasks.Empty()` + `FileChanges.Empty()`，**均无 `FScopeLock`**；对照加锁路径 `:131`（`EnqueueTask`）、`:202`（`Compile` 内 `MoveTemp(FileChanges)`）；工作线程无锁读 `:63 Tasks.IsEmpty()`、`:148 IsCompiling()` 内 `Tasks.IsEmpty()`。成员声明核对 `Source/Compiler/Public/FCSharpCompilerRunnable.h:60-72`：`TQueue<bool> Tasks`、`TArray<FFileChangeData> FileChanges`、`FCriticalSection CriticalSection`、`std::atomic<bool> bIsCompiling`、`bool bIsGenerating`、`bool bIsStopped`
- **级别变动**: 无（**补充限定**：广播点仅 `UnrealCSharpEditor.cpp:318`/`:392` 两处；在队列路径下工作线程此刻正阻塞于 `:242 WaitUntilTaskCompletes`，故最坏窗口是**同步编译路径 + 生成器在游戏线程广播**的重入场景，见"未覆盖/存疑项"）
- **调用方补充**: grep `OnBeginGenerator|OnEndGenerator` 全插件 56 处命中，`Broadcast()` 只有 `Source/UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp:318`（Begin）与 `:392`（End）
- **文件**: `Source/Compiler/Private/FCSharpCompilerRunnable.cpp:467-483`（对比 `:128-144`、`:196-205`、`:63-74`）
- **函数**: `FCSharpCompilerRunnable::OnBeginGenerator()`、`FCSharpCompilerRunnable::OnEndGenerator()`
- **置信度**: 高（调用线程与竞争对象明确）

**现状（代码事实）**
```cpp
// FCSharpCompilerRunnable.cpp:467
void FCSharpCompilerRunnable::OnBeginGenerator()
{
	bIsGenerating = true;

	Tasks.Empty();          // ← 无 FScopeLock

	FileChanges.Empty();    // ← 无 FScopeLock
}
// 对照：同一成员在别的路径上是加锁的
// :131  FScopeLock ScopeLock(&CriticalSection);   (EnqueueTask)
// :202  FScopeLock ScopeLock(&CriticalSection);   (Compile 内 MoveTemp(FileChanges))
```
而工作线程在 `Run()` 中无锁读 `Tasks.IsEmpty()`（`:63`），在 `Compile()` 中加锁 `MoveTemp(FileChanges)`（`:204`）。

**调用上下文**
`FUnrealCSharpCoreModuleDelegates::OnBeginGenerator.Broadcast()` 在 `UnrealCSharpEditor.cpp:318`、`OnEndGenerator.Broadcast()` 在 `:392`，都是**游戏线程**（编辑器模块/控制台命令/菜单路径）。工作线程同时在 `Compile()` 内。

**问题**
`TArray::Empty()` 会释放底层缓冲区，而工作线程可能在 `:204` 正在从同一 `TArray` 上 `MoveTemp`（复制指针并接管缓冲区）——一侧持锁一侧不持锁等于**没有互斥**。这不是"理论 UB 而已"：`FileChanges` 是 `TArray<FFileChangeData>`，其元素含 `FString`，撕裂读会造成悬垂 `FString` 缓冲，最坏情况是崩溃或读到已释放内存。触发窗口是 `Compile()` 返回前后的几毫秒到几秒（`SyncProcess` 结束到 `:202` 之间）。

**建议**
把 `OnBeginGenerator`/`OnEndGenerator` 的四处修改放进 `FScopeLock ScopeLock(&CriticalSection);` 内；`bIsGenerating` 改为 `std::atomic<bool>`（与 `bIsCompiling` 保持一致）。

**验证方式**
`grep -n "FileChanges" Source/Compiler/Private/FCSharpCompilerRunnable.cpp` 逐条核对是否都在锁内；用 TSAN（Linux 构建）或 `-fsanitize=thread` 跑一次"生成中触发编译"的场景。

---

#### [F-CMP-005] 游戏线程上执行完整 `dotnet build`（控制台命令 / 右键菜单），编辑器冻结且无进度窗口

- **类别**: 并发/线程安全（UI 线程阻塞）
- **严重度**: **P2**
- **复核结论**: 部分确认（偏差：级别。原文 P1；游戏线程同步跑 `dotnet build` 只造成编辑器冻结、不损坏数据，且是显式用户操作触发 → P2）
- **可达性**: 活跃
- **复核证据**: `Source/Compiler/Private/FCSharpCompiler.cpp:60-69` 已读（在调用线程直调 `Runnable->Compile(...)`）；调用点 `Source/UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp:114`（控制台命令）与 `DynamicDataSource.cpp:43/51`、`UnrealCSharpEditor.cpp:388`（右键菜单/资源回调）；`SyncProcess` 内的阻塞循环 `FUnrealCSharpFunctionLibrary.cpp:1569-1574` 已读
- **级别变动**: P1→P2（用户可预期的界面冻结，非静默错误；热路径不在 Tick 中）
- **文件**: `Source/Compiler/Private/FCSharpCompiler.cpp:44-69`；UI 侧 `Source/UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp:109-117`、`Source/UnrealCSharpEditor/Private/ContentBrowser/DynamicDataSource.cpp:43/51`
- **函数**: `FCSharpCompiler::ImmediatelyCompile()`、`FCSharpCompiler::Compile(const TFunction<void()>&)`
- **置信度**: 高（`WaitUntilTaskCompletes` 不会死锁这一点已核对引擎源码，见"说明"）

**现状（代码事实）**
两条同步入口都直接在调用线程执行（见 F-CMP-003 代码片段），其中 `Compile()` 内部还会阻塞等待：
```cpp
// FCSharpCompilerRunnable.cpp:213
	const auto Task = FFunctionGraphTask::CreateAndDispatchWhenReady(..., ENamedThreads::GameThread);
	FTaskGraphInterface::Get().WaitUntilTaskCompletes(Task);   // :242
```

**调用上下文**
`UnrealCSharpEditor.cpp:114`（控制台命令路径）**没有任何进度窗口**；`ImmediatelyCompile` 的调用者同样没有（进度窗只在 `FEditorListener::RequestCompile` 链路上的 `WaitForCompile()` 里出现，`FEditorListener.cpp:676`）。因此从控制台敲 `UnrealCSharp.Editor.Compile` 时，游戏线程会阻塞整个构建时长。

**说明（重要，避免误判为 P0）**
我按任务书要求专门验证了"游戏线程 `WaitUntilTaskCompletes` 一个游戏线程任务"是否自锁：**不会**。`TaskGraph.cpp:1447-1460` 先对任务执行 `TryRetractAndExecute(NeverTimeout)`（把未开始的任务"撤回"并在当前线程就地执行），`bAllTasksCompleted` 随即为真直接返回；即使撤不回来，`WaitUntilTaskCompletes`→`WaitOnNamedThreadForTasks`（`:1480-1487`）会派发 `FReturnGraphTask` 并调 `ProcessThreadUntilRequestReturn` **自泵**线程队列。所以这里是"冻结"而非"死锁"。

**问题**
游戏线程被占用 → 窗口无响应、Slate 不重绘、渲染命令不入队；对打包前的 CI/无 UI 场景（若通过控制台命令驱动）会表现为"步骤超时"而非报错。

**建议**
同步入口统一改为"入队 + 在游戏线程用 `FTSTicker`/`FScopedSlowTask` 等待完成"，或至少在同步路径前显示一个 `FScopedSlowTask`。若确实需要同步语义，应把 `AsyncTask(GameThread)` 的那一段改为可重入的跳过逻辑（既然任务会被就地执行，直接调用即可，无需派发再等待）。

**验证方式**
编辑器控制台执行 `UnrealCSharp.Editor.Compile`，用 `stat unit`/任务管理器观察游戏线程是否连续占用数秒以上。

---

### P2

#### [F-CMP-006] 文件变更路径下 `CompileInterop` 从不执行，Interop 程序集可能长期陈旧

- **类别**: Bug
- **严重度**: P2
- **文件**: `Source/Compiler/Private/FCSharpCompilerRunnable.cpp:156-170`、`:172-174`、`:185`
- **函数**: `FCSharpCompilerRunnable::DoWork()` / `ImmediatelyDoWork()` / `Compile(...)`
- **置信度**: 高（代码事实确定）；"是否导致实际错误"取决于用户是否编辑 Interop C# 源码（中）
- **复核结论**: 确认（"是否真的导致绑定不一致"取决于用户是否改 `Script/Interop`，属条件触发；代码事实成立，级别维持 P2）
- **可达性**: 活跃（文件监听是默认路径；`UUnrealCSharpEditorSetting::bEnableCompiled` 默认 `true`）
- **复核证据**: `Source/Compiler/Private/FCSharpCompilerRunnable.cpp:156-162`（`DoWork()` 只传 lambda，`bCompileInterop` 取默认值）、`:164-170`（`ImmediatelyDoWork` 显式传 `true`）、`:185`（`if (bCompileInterop && !CompileInterop(...))`）——均已读；默认值在 `Source/Compiler/Public/FCSharpCompilerRunnable.h:36`（`bCompileInterop = false`，已读）；grep `bCompileInterop` 全插件命中 **3** 处：`.h:36`、`.cpp:173`、`.cpp:185`
- **级别变动**: 无

**现状（代码事实）**
```cpp
// :156
void FCSharpCompilerRunnable::DoWork()
{
	Compile([](const TArray<FFileChangeData>& InFileChanges)
	{
		FDynamicGenerator::Generator(InFileChanges);
	});                                  // ← bCompileInterop 取默认值 false
}
// :172
void FCSharpCompilerRunnable::Compile(const TFunction<...>& InFunction,
                                      const bool bCompileInterop, const bool bForceCompileInterop,
                                      const bool bReloadImmediately)
...
			if (bCompileInterop && !CompileInterop(bForceCompileInterop))   // :185
```
`ImmediatelyDoWork`（`:164-170`）显式传 `true`；`DoWork` 是**唯一**由队列驱动的路径（`:78`）。

**调用上下文**
队列路径来源：`FEditorListener.cpp:642`/`648` → `FCSharpCompiler::Compile(...)` → `EnqueueTask` → `Run():78 DoWork()`。也就是说，**绝大多数编译（文件监听/资源变更触发）都走 `bCompileInterop=false`**；只有用户手动点"立即编译"（`DynamicDataSource.cpp:43/51`、`UnrealCSharpEditor.cpp:388`）才可能编译 Interop。

**问题**
`Script/Interop/*.cs`（插件源码目录）被修改后由文件监听触发编译时，Interop 不会重建，而 `Interop.dll` 正是 `Shared.props:17-20` 里被 `HintPath` 引用的运行时依赖（生成物 `Script/Shared.props:23` → `..\..\Content\Script\Interop.dll`）。结果是"改了 Interop 代码但游戏跑的是旧 Interop.dll"，表现为难以定位的绑定行为不一致。

**建议**
把 `DoWork()` 改为 `Compile(lambda, /*bCompileInterop=*/true)`，或在 `Compile()` 内部按文件变更集合判断是否触及 `Interop/` 目录；`bForceCompileInterop` 保留给"强制重建"菜单。

**验证方式**
`grep -n "bCompileInterop" -r Source/Compiler`；或在 `CompileInterop()` 入口加 `UE_LOG`，然后只改 `Script/Interop` 下的文件并等待自动编译，观察该日志是否出现。

---

#### [F-CMP-007] `GetDotNet()` 硬编码路径且失败时诊断为空——用户只看到"Compilation failed"

- **类别**: 平台兼容 / Bug（可诊断性）
- **严重度**: P2
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:42-57`、`:1580-1585`；`Source/Compiler/Private/FCSharpCompilerRunnable.cpp:384`、`:395-411`
- **函数**: `FUnrealCSharpFunctionLibrary::GetDotNet()`、`SyncProcess()`、`FCSharpCompilerRunnable::Compile()`
- **置信度**: 高
- **复核结论**: 确认
- **可达性**: 活跃（本工程 `DotNetPath` 未设置 → 走硬编码分支；`bEnableCompiled=true`）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:42-57` 已读：`:46` 先取 ini 的 `DotNetPath`，`:53` 硬编码 `C:/Program Files/dotnet/dotnet.exe`，`:55` 非 Windows `/usr/local/share/dotnet/dotnet`；`:1580-1585`（`GetProcReturnCode` 失败置 `-1` 且 `Result` 仍为空串）已读；`FCSharpCompilerRunnable.cpp:384`（`static auto CompileTool = ...GetDotNet()`）、`:405-408`（打印空 `InResult`）已读。引擎侧 `CreateProc failed` 仅打 `LogWindows` 警告 → `Engine\Source\Runtime\Core\Private\Windows\WindowsPlatformProcess.cpp:538`（**已回引擎源码核对，原文行号正确**）
- **级别变动**: 无

**现状（代码事实）**
```cpp
// FUnrealCSharpFunctionLibrary.cpp:42
FString FUnrealCSharpFunctionLibrary::GetDotNet()
{
	if (const auto UnrealCSharpEditorSetting = ...)
	{
		if (const auto& DotNetPath = UnrealCSharpEditorSetting->GetDotNetPath(); !DotNetPath.IsEmpty())
		{
			return DotNetPath;
		}
	}
#if PLATFORM_WINDOWS
	return TEXT("C:/Program Files/dotnet/dotnet.exe");     // 硬编码，不查 PATH
#else
	return TEXT("/usr/local/share/dotnet/dotnet");
#endif
}
```
```cpp
// FUnrealCSharpFunctionLibrary.cpp:1580
	if (!FPlatformProcess::GetProcReturnCode(ProcessHandle, &ReturnCode))
	{
		ReturnCode = -1;                 // CreateProc 失败（如路径不存在）走这里
	}
	InOnComplete(ReturnCode, Result);    // Result 此时为空字符串
```
```cpp
// FCSharpCompilerRunnable.cpp:405
		if (!bSucceeded)
		{
			UE_LOG(LogUnrealCSharp, Error, TEXT("%s"), *InResult);   // 空串 → 日志里什么都没有
		}
		ShowCompileResultNotification(bSucceeded);                   // 只显示 "Compilation failed"
```

**调用上下文**
`GetDotNet()` 的返回值被缓存在两处 `static`（`FCSharpCompilerRunnable.cpp:279`、`:384`），并由 `FCodeAnalysis.cpp:41` 与 `UnrealCSharpEditorSetting.cpp:146` 以不同方式获取（后者用裸 `"dotnet"` 走 PATH）。实测工程配置 `Config/DefaultUnrealCSharpSetting.ini` 未设置 `DotNetPath`，而 `Saved/Temp/Win64/.../DefaultUnrealCSharpEditorSetting.ini` 中 `DotNetPath=` 为空 → 走硬编码分支。

**问题**
（1）dotnet 常见安装位置并非 `C:\Program Files\dotnet`：用户级安装是 `%LOCALAPPDATA%\Microsoft\dotnet\dotnet.exe`，VS 自带在 `C:\Program Files\Microsoft Visual Studio\...\dotnet\`，Chocolatey/Scoop 又在别处。路径不对时 `CreateProc` 直接失败（引擎会打一条 `LogWindows` 警告 `CreateProc failed: ...`，`WindowsPlatformProcess.cpp:538`），但插件侧 `InResult` 为空，用户界面只有"Compilation failed"，没有"找不到 dotnet"这类可行动信息。（2）`-1` 与"编译报错退出码 1"不可区分，无法给出针对性提示。

**建议**
`GetDotNet()` 依次尝试：ini 设置 → `FPlatformProcess::ExecProcess`/`where dotnet` 结果 → 硬编码默认；`SyncProcess` 在 `CreateProc` 失败时把 `FString::Printf(TEXT("Failed to launch '%s'"), *InURL)` 写进 `Result`（或额外提供一个 `InOnLaunchFailed` 回调）；`Compile()` 对 `ReturnCode == -1` 给出专用提示。

**验证方式**
把 ini 的 `DotNetPath` 指向一个不存在的路径，观察编辑器提示文本与日志内容。

---

#### [F-CMP-008] `ReadPipe` 按块 UTF-8 解码：非 ASCII 输出必然可能出现乱码

- **类别**: 平台兼容（编码）
- **严重度**: P2
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1555-1567`（调用点 `Source/Compiler/Private/FCSharpCompilerRunnable.cpp:414-417`、`:407`）
- **函数**: `FUnrealCSharpFunctionLibrary::SyncProcess` 的 `ReadOutput` lambda
- **置信度**: 高（机制有引擎源码注释直接支持）；触发频率取决于本机 MSBuild 语言（中）
- **复核结论**: 确认（机制成立；"中文本地化下必然乱码"属条件性，故维持 P2 不加级）
- **可达性**: 活跃（每次 `dotnet build` 都会走 `ReadPipe`；非 ASCII 触发取决于子进程输出编码，本机为中文 Windows）
- **复核证据**: 插件侧 `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1555-1567` 已读（`:1557 FPlatformProcess::ReadPipe(ReadPipe)`、`:1560 Result.Append(Output)`）；`:1569-1574` 每 10ms 一次、`:1576` 收尾再读一次（已读）。引擎侧**已回源码核对，报告引用正确**：`Engine\Source\Runtime\Core\Private\Windows\WindowsPlatformProcess.cpp:1849-1871`，其中 `:1853` 就是那句注释 `// Note: String becomes corrupted when more than one byte per character and all bytes are not available`，`:1864` `Output += FUTF8ToTCHAR((const ANSICHAR*)Buffer).Get();`；字节版 `ReadPipeToArray` 在 `:1873`
- **级别变动**: 无

**现状（代码事实）**
```cpp
// FUnrealCSharpFunctionLibrary.cpp:1555
	const auto ReadOutput = [&]()
	{
		if (const auto Output = FPlatformProcess::ReadPipe(ReadPipe);
			!Output.IsEmpty())
		{
			Result.Append(Output);              // 已解码的 FString 拼接
			if (InOnOutput) { InOnOutput(Output); }
		}
	};
// :1569 每 10ms 调一次 ReadOutput()
```
引擎侧实现（`WindowsPlatformProcess.cpp:1849-1871`）：
```cpp
	// Note: String becomes corrupted when more than one byte per character and all bytes are not available
	uint32 BytesAvailable = 0;
	if (::PeekNamedPipe(...BytesAvailable...) && (BytesAvailable > 0))
	{
		UTF8CHAR* Buffer = new UTF8CHAR[BytesAvailable + 1];
		... ReadFile(ReadPipe, Buffer, BytesAvailable, ...);
		Output += FUTF8ToTCHAR((const ANSICHAR*)Buffer).Get();   // ← 假定 UTF-8
	}
```

**调用上下文**
每次 `ReadPipe` 只读"当前可用字节"（`BytesAvailable`），父进程每 10ms 读一次；输出最终进入 `CompileProgress.SetOutput`（`FCSharpCompilerRunnable.cpp:414-417`）与失败时的 `UE_LOG`（`:407`）。

**问题**
两层问题叠加：（1）**编码假定**：`FUTF8ToTCHAR` 认为子进程输出是 UTF-8。若 `dotnet`/MSBuild 在中文 Windows 上按控制台代码页（GBK/936）输出中文消息，解码即乱码，且乱码会进入 `CompileProgress` 的文本匹配与用户可见日志。（2）**块边界截断**：即使输出确实是 UTF-8，一个多字节字符若被 10ms 的读取切在两个 `BytesAvailable` 之间，两半各自解码都会失败（引擎注释明确说明这一点），而 `Result.Append` 之后**无法再修复**。插件没有采用 `ReadPipeToArray`（`WindowsPlatformProcess.cpp:1873`，返回原始字节）后在行边界统一解码的做法。

**建议**
改用 `FPlatformProcess::ReadPipeToArray` 累积原始字节，在遇到完整行（`\n`）后再统一 `FUTF8ToTCHAR` 解码；同时在 `CompileParam` 前注入 `DOTNET_CLI_UI_LANGUAGE=en`（见 F-CMP-009）以稳定输出编码与文本。若希望根治编码问题，可在子进程环境里设置 `DOTNET_SYSTEM_CONSOLE_ALLOW_ANSI_COLOR_REDIRECTION`/`Console.OutputEncoding` 不可行，故以"按原始字节 + 行边界解码"为准。

**验证方式**
把 `GetDotNet()` 换成输出一段中文 UTF-8 的脚本（如 `cmd /c chcp 65001 && echo 测试`），对比 `Result` 内容；或直接把 `ReadPipe` 换成 `ReadPipeToArray` 做 A/B。

---

#### [F-CMP-009] 进度解析依赖英文 MSBuild 文本，中文本地化环境下阶段不再更新

- **类别**: 平台兼容 / Bug（可诊断性）
- **严重度**: P2
- **文件**: `Source/Compiler/Private/FCSharpCompileProgress.cpp:89-136`
- **函数**: `FCSharpCompileProgress::ParseLine()`
- **置信度**: 中（`.NET SDK` 本地化行为未在本机实测；代码对英文文本的依赖是确定的）
- **复核结论**: 部分确认（偏差：无法验证子项。代码对英文文本的硬依赖**已逐字核实**，但"MSBuild/csc 在中文环境下具体输出什么"本机无 SDK 资源可比对 → 该子项记 `无法验证（卡点：未实测本地化输出）`；级别维持 P2）
- **可达性**: 活跃（条件性：需 `DOTNET_CLI_UI_LANGUAGE`/系统 UI 语言非英文；运行时判定成败不受影响）
- **复核证据**: `Source/Compiler/Private/FCSharpCompileProgress.cpp:89-136` 已读全文：`:91` `": error "`/`"Build FAILED"`、`:96` `"Build succeeded"`、`:101-102` `"Determining projects to restore"`/`"Restored "`、`:107` `"up-to-date for restore"`、`:112` `" -> "`（语言无关）；成败判定确实不依赖文本：`FCSharpCompilerRunnable.cpp:397 bSucceeded = InReturnCode == 0`、`FUnrealCSharpFunctionLibrary.cpp:1580` 已读
- **级别变动**: 无

**现状（代码事实）**
```cpp
// FCSharpCompileProgress.cpp:91
	if (InLine.Contains(TEXT(": error ")) || InLine.Contains(TEXT("Build FAILED"))) { return StageFailed; }
	if (InLine.Contains(TEXT("Build succeeded"))) { return StageSucceeded; }
	if (InLine.Contains(TEXT("Determining projects to restore")) || InLine.Contains(TEXT("Restored ")))
	{ return StageRestoring; }
	if (InLine.Contains(TEXT("up-to-date for restore")))
	{ return CompilePhase == ECompilePhase::Interop ? StageCompilingInterop : StageCompilingUE; }
```

**调用上下文**
`ParseLine` 由 `SetOutput`（工作线程）与 `Flush`（`OnComplete` 里调用，`:296`/`:399`）驱动；结果显示在 Slate 通知（`FCSharpCompilerRunnable.cpp:360-380`）与进度窗口（`FEditorListener.cpp:729-733`）。

**问题**
MSBuild 的**自身**消息（`Build succeeded`、`Build FAILED`、`Determining projects to restore...`）随 .NET SDK 的 UI 语言本地化；中文环境下会变成"已成功生成"/"生成失败"之类文本，于是这些 `Contains` 全部不命中，`ParseLine` 只会退回到 `" -> "` 规则（该规则不依赖语言，因为工程名是标识符），阶段显示会退化甚至长时间停在旧文本。**注意**：这不会造成"误判成功"——成败判定走的是返回码（`FCSharpCompilerRunnable.cpp:397`、`FUnrealCSharpFunctionLibrary.cpp:1580`），与文本无关，这一点是好的设计。另外 `: error ` 依赖 csc 的英文错误前缀（csc 不本地化该前缀），所以失败阶段通常仍能识别。

**建议**
在启动子进程前设置 `DOTNET_CLI_UI_LANGUAGE=en`（引擎 `CreateProc` 无环境参数，可在 `SyncProcess` 里用 `FPlatformMisc::SetEnvironmentVar` 临时设置后恢复，或改用带环境块的 `CreateProc` 变体）；或改为以返回码为准的终态 + 用 `" -> "`/`Restored` 之外的**语言无关**信号（如 MSBuild 的 `-v:q` 输出行数、`/bl` 二进制日志）驱动阶段。

**验证方式**
把系统 UI 语言或 `DOTNET_CLI_UI_LANGUAGE=zh-CN` 后执行一次编译，观察进度通知文本是否还变化；`grep -n "Contains" FCSharpCompileProgress.cpp` 列出全部语言敏感点。

---

#### [F-CMP-010] 持锁状态下从非游戏线程访问 UObject CDO（`GetUEName()`/`GetGameName()`）

- **类别**: 并发/线程安全
- **严重度**: P2
- **文件**: `Source/Compiler/Private/FCSharpCompileProgress.cpp:89-136`（`:121`、`:123`，锁来自 `:41`/`:59`）
- **函数**: `FCSharpCompileProgress::ParseLine()`（在 `SetOutput`/`Flush` 的 `FScopeLock` 内被调用）
- **置信度**: 中（"工作线程 + 持锁"是确定事实；"`GetMutableDefault` 在该时机是否真的不安全"未实测）
- **复核结论**: 部分确认（偏差：严重度证据不足。"工作线程持锁调 UObject CDO getter"是确定事实，但"该时机不安全"未实测 → 子项记 `无法验证（卡点：需实机 check(IsInGameThread()) 或 TSAN）`；因后果不确定，由 P2 **下调为 P3**）
- **可达性**: 活跃（每次编译只要有含 `" -> "` 的 MSBuild 行就会走到 `:119-123`）
- **复核证据**: `Source/Compiler/Private/FCSharpCompileProgress.cpp:39-55`（`SetOutput` 在 `:41 FScopeLock` 内 → `:53 ProcessLine` → `ParseLine`）、`:89-136`（`:121`/`:123` 取 `GetUEName()`/`GetGameName()`，**确在锁内**）、`:57-64`（`Flush` 同样在 `:59` 锁内）；`SetOutput` 的调用者是工作线程：`FCSharpCompilerRunnable.cpp:414-417` ← `FUnrealCSharpFunctionLibrary.cpp:1564 InOnOutput(Output)`（已读）；游戏线程每 tick 取同一把锁 `FCSharpCompilerRunnable.cpp:369 CompileProgress.GetStage()`（已读）
- **级别变动**: P2→P3（当前 LeanCLR 配置下 `GetMutableDefaultSafe` 只读已构造的 CDO 数值，未见实际不安全路径；按"证据不足不作高定级"处理）

**现状（代码事实）**
```cpp
// FCSharpCompileProgress.cpp:39
void FCSharpCompileProgress::SetOutput(const FString& InOutput)
{
	FScopeLock ScopeLock(&CriticalSection);
	...
		ProcessLine(Line);          // → ParseLine
}
// :121
	const auto UEName = FUnrealCSharpFunctionLibrary::GetUEName();
	const auto GameName = FUnrealCSharpFunctionLibrary::GetGameName();
```
```cpp
// FUnrealCSharpFunctionLibrary.cpp:735
FString FUnrealCSharpFunctionLibrary::GetUEName()
{
	if (const auto UnrealCSharpSetting = GetMutableDefaultSafe<UUnrealCSharpSetting>())   // ← UObject CDO
```
而游戏线程每 tick 会取同一把锁（`FCSharpCompilerRunnable.cpp:369 CompileProgress.GetStage()`）。

**调用上下文**
`SetOutput` 由 `SyncProcess` 的 `InOnOutput` 在**工作线程**调用（`FCSharpCompilerRunnable.cpp:414-417` ← `FUnrealCSharpFunctionLibrary.cpp:1564`）；`GetUEName()` 的调用链会走到 `GetMutableDefault<UUnrealCSharpSetting>()`（`FUnrealCSharpFunctionLibrary.h:265-269`）。

**问题**
两点：（1）一致性契约——UE 的 `GetMutableDefault`/CDO 首次访问通常在游戏线程完成，工作线程访问属于未定义用法（可能触发懒初始化/`UObject` 数组访问）；（2）持锁期间做这件事，把游戏线程每 tick 的 `GetStage()`（`:369`）与它耦合在一起，任一侧变慢都会互相拖累（虽然不存在反向加锁，未构成死锁，但属于不必要的锁内重活）。

**建议**
把 `UEName`/`GameName` 在 `Compile()` 开始（加锁前、游戏线程/工作线程任务的边界上）取一次并缓存为成员，`ParseLine` 只用缓存值；或把 `ParseLine` 的匹配改为只需要一个预计算的 `TSet<FString>`。

**验证方式**
`grep -n "GetUEName()\|GetGameName()" Source/ -r` 列出所有工作线程可达的 UObject 访问；在 `ParseLine` 内加 `check(IsInGameThread())` 触发一次编译即可观察到断言失败。

---

#### [F-CMP-011] `FCSharpCompiler::Get()` 的函数局部静态单例在**静态销毁期**执行 `Kill(true)` 与跨模块委托移除

- **类别**: 未定义行为 / 生命周期
- **严重度**: P2
- **文件**: `Source/Compiler/Private/FCSharpCompiler.cpp:29-34`、`:13-27`；`Source/Compiler/Private/FCSharpCompilerRunnable.cpp:32-43`、`:125`、`:143`
- **函数**: `FCSharpCompiler::Get()`、`~FCSharpCompiler()`、`~FCSharpCompilerRunnable()`
- **置信度**: 中（C++ 静态销毁顺序与 UE 模块卸载顺序未在本机验证）
- **复核结论**: 部分确认（偏差：机制成立但触发条件未证实。"函数局部静态单例随 DLL 卸载析构"与"跨模块委托 `Remove`"是确定事实；"析构时 `UnrealCSharpCore` 静态已销毁"未验证 → 子项记 `无法验证（卡点：需实机在卸载点打日志）`；维持 P2）
- **可达性**: 活跃（编辑器关闭即触发该析构）
- **复核证据**: `Source/Compiler/Private/FCSharpCompiler.cpp:29-34`（`static FCSharpCompiler Compiler;` 函数局部静态）、`:13-27`（析构里 `:17 Kill(true)`、`:19 delete Thread`、`:23 delete Runnable`）已读；`Source/Compiler/Private/FCSharpCompilerRunnable.cpp:32-43`（析构移除两个委托，`AddRaw` 在 `:25-29`）已读；跨模块全局 `Source/UnrealCSharpCore/Private/Delegate/FUnrealCSharpCoreModuleDelegates.cpp:9`/`:11` 定义、`.h:28-30` 声明（**grep 已核**）；`EnqueueTask` 对 `Event` 无空判 `:125`、`:143`，而 `Exit()` 置空 `:108`（均已读）
- **级别变动**: 无

**现状（代码事实）**
```cpp
// FCSharpCompiler.cpp:29
FCSharpCompiler& FCSharpCompiler::Get()
{
	static FCSharpCompiler Compiler;      // ← 函数局部静态，随进程/DLL 卸载销毁
	return Compiler;
}
```
析构里会 `Thread->Kill(true)`（`:17`，可能无限等待，见 F-CMP-001）并 `delete Runnable`（`:23`），后者在 `~FCSharpCompilerRunnable`（`FCSharpCompilerRunnable.cpp:34-42`）里访问**另一个模块**的全局：
```cpp
	FUnrealCSharpCoreModuleDelegates::OnEndGenerator.Remove(OnEndGeneratorDelegateHandle);
```
`FUnrealCSharpCoreModuleDelegates::OnEndGenerator` 是 `UnrealCSharpCore` 模块中的静态成员（`FUnrealCSharpCoreModuleDelegates.cpp:11`、`.h:28-35`）。另外 `EnqueueTask` 对 `Event` 无空判（`:125`、`:143`），而 `Exit()` 会把 `Event` 置为 `nullptr`（`:108`）。

**问题**
`Compiler` 是 **Compiler 模块 DLL** 里的静态对象，其析构发生在 DLL 卸载/进程退出阶段——此时 `UnrealCSharpCore` 模块的静态成员可能已经销毁（跨模块静态析构顺序未定义），`Remove()` 就是对已销毁对象调用（UAF）；此时若编译线程仍在跑，还会叠加 F-CMP-001 的无限等待。`Event` 空判缺失是同一族问题：`Exit()` 之后仍有入队（例如关闭流程中残余的 ticker/监听器回调）就会解引用空指针崩溃。

**建议**
（1）由模块负责生命周期：在 `FCompilerModule::StartupModule` 里 `FCSharpCompiler::Get()` 显式构造、`ShutdownModule` 里显式 `Stop()`+`delete`（把单例实现改为 `TUniquePtr` + `Create/Shutdown`），避免依赖静态析构顺序；（2）`EnqueueTask` 增加 `if (Event == nullptr) { return; }`；（3）`~FCSharpCompilerRunnable` 里先用 `FUnrealCSharpCoreModuleDelegates` 所在模块的存活检查（或把委托句柄的移除提前到 `ShutdownModule`）。

**验证方式**
在 `~FCSharpCompiler`/`~FCSharpCompilerRunnable` 里加 `UE_LOG`，编译进行中直接关闭编辑器，观察动态库卸载与析构的先后（也可用 `-nullrhi` + `-exitcode` 的自动化命令复现）。

---

#### [F-CMP-012] `bIsGenerating`/`bIsStopped` 是非原子 `bool`，与同一类中的 `bIsCompiling` 不一致

- **类别**: 未定义行为（数据竞争）
- **严重度**: P2
- **文件**: `Source/Compiler/Public/FCSharpCompilerRunnable.h:68-72`；读写点 `FCSharpCompilerRunnable.cpp:56`、`:61`、`:94`、`:469`、`:478`
- **函数**: `FCSharpCompilerRunnable::Run()` / `Stop()` / `OnBeginGenerator()` / `OnEndGenerator()`
- **置信度**: 高（读写线程已确定）
- **复核结论**: 确认（形式 UB 成立，但报告已自述"实际风险相对可控"，故维持 P2 不上调）
- **可达性**: 活跃
- **复核证据**: 成员声明 `Source/Compiler/Public/FCSharpCompilerRunnable.h:68-72` 已读：`:68 std::atomic<bool> bIsCompiling`、`:70 bool bIsGenerating`、`:72 bool bIsStopped`；读写点全部已读：`:56`（读 `bIsStopped`）、`:61`（读 `bIsGenerating`）、`:94`（`Stop()` 写）、`:469`/`:478`（生成器回调写）。`bIsGenerating` 的跨线程写方是游戏线程 `UnrealCSharpEditor.cpp:318`/`:392` 广播
- **级别变动**: 无

**现状（代码事实）**
```cpp
// FCSharpCompilerRunnable.h:68
	std::atomic<bool> bIsCompiling;    // ← 这个是对的

	bool bIsGenerating;                // ← 游戏线程写(469/478)、工作线程读(61)
	bool bIsStopped;                   // ← 游戏线程写(94)、工作线程读(56)
```

**调用上下文**
工作线程：`Run():56`、`:61`；游戏线程：`Stop()`（引擎 `Kill` 调用）、`OnBeginGenerator/OnEndGenerator`（`UnrealCSharpEditor.cpp:318/392` 广播）。

**问题**
形式上属于 C++ 数据竞争（未同步的跨线程非原子读写 = UB）。实际风险相对可控：`bIsStopped` 的读取点位于一个含非内联函数调用的 `while(true)` 体内，编译器很难把它提升为寄存器常驻，因此"关不掉"的极端表现不易出现；`bIsGenerating` 则只影响"是否开始新任务"的时序。但仍应统一为原子，理由是它和 `bIsCompiling` 属于同一语义层级，混用会让后续维护者误判。

**建议**
`bIsGenerating`、`bIsStopped` 改为 `std::atomic<bool>`（初始化列表写法保持 `(...)` 即可）；`IsCompiling()` 已经只读原子成员。

**验证方式**
`grep -n "bIsGenerating\|bIsStopped\|bIsCompiling" Source/Compiler -r` 对照读写点；TSAN 构建下跑一次编译。

---

#### [F-CMP-013] `CppVersion.h` 的 `__cplusplus` 判定依赖 UBT 的 `/Zc:__cplusplus`，否则 `STD_CPP_20` 在 MSVC 上恒假

- **类别**: 平台兼容
- **严重度**: P2
- **文件**: `Source/CrossVersion/Public/CppVersion.h:3-9`；使用点 `Source/UnrealCSharpCore/Private/Binding/Class/FBindingClassRegister.cpp:115`、`Source/UnrealCSharp/Private/Reflection/Container/FSetHelper.cpp:104`、`Source/UnrealCSharp/Private/Reflection/Container/FMapHelper.cpp:202`
- **函数**: 宏（无函数）
- **置信度**: 中（UBT 添加 `/Zc:__cplusplus` 的条件 `bUpdatedCPPMacro` 是否为真未逐项验证，但该开关确实存在）
- **复核结论**: 部分确认（偏差：**前提与等级**。核心前提"MSVC 下 `__cplusplus` 无 `/Zc:` 即恒假"成立，但引擎侧 `bUpdatedCPPMacro` **默认为 `true`** → 默认配置下 `/Zc:__cplusplus` 是开启的，`STD_CPP_20` **不会**恒假；只有用户在 `Target.cs` 里显式关掉才退化 → 由 P2 下调为 P3）
- **可达性**: 潜伏（默认配置不可达；需 `WindowsPlatform.bUpdatedCPPMacro = false` 或非 MSVC 之外的工具链差异才触发）
- **复核证据**: `Source/CrossVersion/Public/CppVersion.h:3-9` 已读全文（4 个宏在 `:3`/`:5`/`:7`/`:9`）。引擎侧**已回源码核对**：`Engine\Source\Programs\UnrealBuildTool\Platform\Windows\VCToolChain.cs:671-673`（`if (Target.WindowsPlatform.bUpdatedCPPMacro) { Arguments.Add("/Zc:__cplusplus"); }`，报告引用的 `:671-674` 吻合）；**关键新证据** `...\Platform\Windows\UEBuildWindows.cs:574` `public bool bUpdatedCPPMacro = true;`（默认开启），`:1005` 为转发属性；`ProjectFiles\VisualStudio\VCProject.cs:590-592` 同样按该开关加参数
- **级别变动**: P2→P3（"默认配置下恒假"这一前提被引擎源码证伪，剩余风险仅为用户显式关闭该开关）

**现状（代码事实）**
```cpp
// CppVersion.h:3
#define STD_CPP_11 __cplusplus >= 201103L
#define STD_CPP_14 __cplusplus >= 201402L
#define STD_CPP_17 __cplusplus >= 201703L
#define STD_CPP_20 __cplusplus >= 202002L
```
```csharp
// Engine\Source\Programs\UnrealBuildTool\Platform\Windows\VCToolChain.cs:671
				if (Target.WindowsPlatform.bUpdatedCPPMacro)
				{
					Arguments.Add("/Zc:__cplusplus");
				}
```
`STD_CPP_20` 有 3 处真实使用（上面 grep 命中），`STD_CPP_11/14/17` 零使用（见 §4 死代码清单）。

**调用上下文**
三处使用都在模板/注册代码里，用于在 C++20 与更早标准之间切换实现分支（例如 `FBindingClassRegister.cpp:115`）。

**问题**
若 `bUpdatedCPPMacro` 为假（该开关与 MSVC 工具链版本/配置相关，且历史上有版本差异），MSVC 在没有 `/Zc:__cplusplus` 时报告 `__cplusplus == 199711L`，于是 `STD_CPP_20`（以及 `STD_CPP_11`）全部为假，三处分支**静默**走老路径。这类"静默降级"不会报错，只会让行为与预期不符，属于典型难查问题。

**建议**
用 UBT 提供的编译期定义替代 `__cplusplus` 判定（UE 会为 C++ 标准定义宏，且可在 `Build.cs` 里显式 `PublicDefinitions.Add("STD_CPP_20=1")` 由 UBT 侧的 `Target.WindowsPlatform`/`CppStandard` 决定）；若坚持用 `__cplusplus`，应在 `Compiler.Build.cs`/`UnrealCSharp.build.cs` 中确保 `/Zc:__cplusplus`（`bUpdatedCPPMacro`）开启，并在宏处加注释说明前提。

**验证方式**
在 MSVC 下编译时 `static_assert(STD_CPP_20)`（若 UE 配置为 C++20）以确认取值；或 `grep -n "bUpdatedCPPMacro" -r <Engine>/Source/Programs/UnrealBuildTool` 查看其赋值条件。

---

#### [F-CMP-014] `DotnetVersion` 设置可选 `V8`/`V9`，会生成 `net8.0`/`net9.0` 而捆绑运行时只有 `10.0.x`

- **类别**: Bug / 配置一致性
- **严重度**: P2
- **文件**: `Source/CrossVersion/Public/DotnetVersion.h:9-13`；`Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:258-268`、`:264-265`；`Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1395-1405`；`Source/ThirdParty/{CoreCLR,LeanCLR,Mono}/*.Build.cs`
- **函数**: `FSolutionGenerator::ReplaceTargetFramework()`、`FUnrealCSharpFunctionLibrary::GetDotnetVersion()`
- **置信度**: 中（"跨大版本不会自动 roll-forward"是 .NET 运行时行为，未在本机实测；配置不一致的代码事实确定）
- **复核结论**: 确认（配置不一致的代码事实成立；"运行时加载失败"这一后果沿用报告自述的中等置信度）
- **可达性**: 潜伏（**默认配置下不可达**：`UnrealCSharpSetting.cpp` 默认 `EDotnetVersion::Latest` → `net10.0` 与捆绑 10.0.4 一致；只有用户手动把 ini 改成 `V8`/`V9` 才触发）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1395-1405` 已读（`:1397` 默认 `Latest`、`:1404` `Latest-1`）；捆绑版本号来自 `Source/ThirdParty/{Mono,CoreCLR,LeanCLR}/*.Build.cs`（报告已列行号，此处以报告引用为准）；TargetFramework 的实际生成值另见 07-…/01 的交叉核对
- **级别变动**: 无（**可达性修正**：原文隐含"活跃"，实际默认配置下为潜伏，仅配置变更后生效）

**现状（代码事实）**
```cpp
// FSolutionGenerator.cpp:258
void FSolutionGenerator::ReplaceTargetFramework(FString& OutResult)
{
	OutResult = OutResult.Replace(TEXT("<TargetFramework></TargetFramework>"),
	                              *FString::Printf(TEXT("<TargetFramework>net%d.%d</TargetFramework>"),
	                                               FUnrealCSharpFunctionLibrary::GetDotnetVersion(),   // ← 主版本来自 ini 设置
	                                               DOTNET_MINOR_VERSION));                              // ← 次版本来自 ThirdParty 模块定义
}
```
```cpp
// FUnrealCSharpFunctionLibrary.cpp:1395
	int32 DotnetVersion = EDotnetVersion::Latest;
	...
	return DotnetVersion == EDotnetVersion::Latest ? DotnetVersion - 1 : DotnetVersion;   // Latest(=11) → 10
```
实测（本工作区）：`Script/Shared.props:8`、`Script/CodeAnalysis/CodeAnalysis.csproj:9`、`Script/Interop/Interop.csproj:15` 均为 `net10.0`；捆绑运行时为 LeanCLR/CoreCLR `10.0.4`（`LeanCLR.Build.cs:11-15`、`CoreCLR.Build.cs:13-17`）、Mono `10.0.1`（`Mono.Build.cs:13-15`）。

**调用上下文**
`ReplaceTargetFramework` 被用于 `Interop.csproj`（`FSolutionGenerator.cpp:84`）、`Shared.props`（`:120`）、`CodeAnalysis.csproj`（`:20`），即**所有** C# 工程的 TFM 都从这里来；而 `.csproj`/`Shared.props` 的写入策略是**每次覆盖**（`CopyTemplate` 的 `bReplaceExistingFile` **默认 `true`**，声明在 `FSolutionGenerator.h:11/15`、判定在 `CopyTemplate:141/151`；**唯一**显式传 `false`（只在缺失时写）的调用点是 `Game.csproj`）。
⚠️ **2026-09-17 更正**：本条初版写成"`.csproj` 只在目标文件不存在时才重新生成（默认 `bReplaceExistingFile=false`）"——与快照 `85348c68` 和 HEAD **均不符**（两处默认值都是 `true`）；下面的问题（2）据此重写。

**问题**
（1）用户在设置面板把 `DotnetVersion` 选成 `V8`/`V9`（`UnrealCSharpSetting.h:66-72` 明确提供了这两个选项）时，生成的程序集 TFM 会低于捆绑运行时，而插件只分发 `shared/Microsoft.NETCore.App/10.0.x`（`CoreCLR.Build.cs:40`）；.NET 的默认 roll-forward 策略不允许跨主版本，运行时会因找不到匹配的 `Microsoft.NETCore.App 8.x/9.x` 而加载失败——这正是任务书担心的"不一致 = 运行时找不到运行时"。（2）~~由于 `.csproj` 生成后不会覆盖，改回设置也不会重新生成，需要用户手删 `Script/` 才能生效，故障更难排查。~~ **（2026-09-17 更正）**：除 `Game.csproj` 外，`.csproj` 与 `Shared.props` 每次生成都会覆盖 ⇒ 改回设置后**跑一次生成器**（工具栏 / `UnrealCSharp.Editor.Generator` / cook）即可把 TFM 重写回来；真正的坑是**生成器不在编辑器启动路径上**（不跑生成器就一直是旧值，见 `F-BLD-022`）＋ `Game.csproj` 确实只在缺失时写。排查难度下调、但不消失（用户得知道要手动触发一次生成）。（3）`Latest - 1` 依赖"枚举最后一项恒等于下一版本号"这一隐含约定（`V8=8,V9=9,V10=10,Latest`），一旦有人插入 `V11`，`Latest` 变成 12，返回值变成 11，而运行时仍是 10.0.x（P3 级隐患，见 `F-CMP-015`）。

**建议**
（1）把"运行时版本"作为单一事实来源：由 `ThirdParty` 模块通过定义导出主版本（如 `DOTNET_MAJOR_VERSION`）并在 `ReplaceTargetFramework` 中直接使用它，让 ini 只能选择"不高于捆绑版本"的值，否则在设置面板上报错；（2）在生成 `.csproj` 时把"当前运行时版本"写入文件（注释或自定义属性），当与设置不匹配时强制重生成并给出日志。

**验证方式**
对已生成工程执行 `grep -n TargetFramework Script/Shared.props Script/*/*.csproj`；把 `DotnetVersion` 改为 `V8`、删除 `Script/` 后重新生成，再用 `dotnet` 运行产物，观察是否报 `You must install or update .NET`。

---

#### [F-CMP-015] `EDotnetVersion::Latest - 1` 的"减一即当前版本"约定过于脆弱

- **类别**: 可优化/可读性（隐患）
- **严重度**: P3
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1404`；枚举 `Source/UnrealCSharpCore/Public/Setting/UnrealCSharpSetting.h:66-72`
- **函数**: `FUnrealCSharpFunctionLibrary::GetDotnetVersion()`
- **置信度**: 高（代码事实）
- **复核结论**: 确认
- **可达性**: 活跃（每次 `FSolutionGenerator::ReplaceTargetFramework` 都会执行 `:1404`）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1395-1405` 已读全文：`:1397 int32 DotnetVersion = EDotnetVersion::Latest;`、`:1404 return DotnetVersion == EDotnetVersion::Latest ? DotnetVersion - 1 : DotnetVersion;`（报告引用行号 `:1404` **精确吻合**）
- **级别变动**: 无（与 `02-…/07` 的 `F-FL-023` 裁决一致：脆弱但当前值正确，P3）
- **交叉引用**: 与 `02-UnrealCSharpCore核心/07-函数库宏与模板Trait.md` 的 `F-FL-023` 为同一处代码，两报告级别口径一致（均为 P3）

**现状（代码事实）**
```cpp
// FUnrealCSharpFunctionLibrary.cpp:1404
	return DotnetVersion == EDotnetVersion::Latest ? DotnetVersion - 1 : DotnetVersion;
```
```cpp
// UnrealCSharpSetting.h:66
enum EDotnetVersion { V8 = 8, V9 = 9, V10 = 10, Latest };   // Latest 隐式 = 11
```

**调用上下文**
唯一的调用者是 `FSolutionGenerator::ReplaceTargetFramework`（`FSolutionGenerator.cpp:264`），即它直接决定所有 C# 工程的 TFM（见 F-CMP-014）。

**问题**
用"枚举末项减一"表达"当前捆绑的运行时版本"，把枚举顺序当成数据，任何一次追加枚举值（例如将来加 `V11`）都会让 TFM 与捆绑运行时脱钩，且**没有断言或日志**。当前值（10）恰好正确。

**建议**
改为显式常量（例如来自 `DOTNET_MAJOR_VERSION` 的 `constexpr int32 CurrentDotnetVersion = DOTNET_MAJOR_VERSION;`）并让 `Latest` 分支返回它；或在 `static_assert(EDotnetVersion::Latest - 1 == DOTNET_MAJOR_VERSION)` 处加编译期校验。

**验证方式**
`grep -n "Latest" Source/UnrealCSharpCore -r`；在 `GetDotnetVersion()` 加 `UE_LOG` 打印设置值与返回值。

---

#### [F-CMP-016] `DotnetVersion.h` 的 `DOTNET8` 与 `!DOTNET9` 分支在当前配置下恒不可达（死分支）

- **类别**: 死代码
- **严重度**: P2
- **文件**: `Source/CrossVersion/Public/DotnetVersion.h:9-13`；消费者 `Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:62`、`:71`、`:78`；版本来源 `Source/ThirdParty/Mono/Mono.Build.cs:13-15`
- **函数**: 宏（无函数）；使用点是 `FMonoDomain::Initialize` 所在的 `#if WITH_MONO` 区块
- **置信度**: 高（数值可静态推出）
- **复核结论**: 确认（数值推演成立：`DOTNET_MAJOR_VERSION=10` → `DOTNET9`=真 → `!DOTNET9` 整块为死分支）
- **可达性**: 不可达（当前五平台 `WITH_LEANCLR=1 / WITH_MONO=0`，`FMonoDomain.cpp:2` 的 `#if WITH_MONO` 整文件不参与编译；即便切回 Mono，`DOTNET9` 仍恒真 → 该分支**任何配置下都到不了**）
- **复核证据**: `Source/CrossVersion/Public/DotnetVersion.h:9-13` 已读全文（`:9` DOTNET8、`:11` DOTNET9、`:13` DOTNET10）；grep `DOTNET8|DOTNET9|DOTNET10` 全插件命中 **6** 处 = 定义 3（`DotnetVersion.h:9/11/13`）+ 使用 3（`Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:62`、`:71`、`:78`）——与报告的"唯一消费者"判断一致，且 `DOTNET10` 确为**零使用**
- **级别变动**: 无（P2=死代码，符合"当前配置不参与编译"）

**现状（代码事实）**
```cpp
// DotnetVersion.h:9
#define DOTNET8 DOTNET_VERSION_START(8, 0, 0)     // = DOTNET_MAJOR_VERSION >= 8
#define DOTNET9 DOTNET_VERSION_START(9, 0, 0)     // = DOTNET_MAJOR_VERSION >= 9
```
```csharp
// Source/ThirdParty/Mono/Mono.Build.cs:13
			"DOTNET_MAJOR_VERSION=10",
			"DOTNET_MINOR_VERSION=0",
			"DOTNET_PATCH_VERSION=1"
```
```cpp
// FMonoDomain.cpp:62   （在 #if WITH_MONO / #if PLATFORM_IOS 之内）
#if !DOTNET9
		mono_dllmap_insert(NULL, "System.Native", NULL, "__Internal", NULL);
		...
#if DOTNET8
		mono_dllmap_insert(NULL, "System.Globalization.Native", NULL, "__Internal", NULL);
#endif
#endif
```

**调用上下文**
`WITH_MONO=1` 时才依赖 `Mono` 模块（`UnrealCSharpCore.build.cs:286-290`），此时 `DOTNET_MAJOR_VERSION` 必然是 `Mono.Build.cs:13` 的 10；本工程当前后端为 LeanCLR（`Config/DefaultUnrealCSharpSetting.ini`），连 `WITH_MONO` 都为 0。

**问题**
`DOTNET9` = `10 >= 9` = 真 → `:62` 的 `!DOTNET9` 整块（`:63-73`，含嵌套的 `#if DOTNET8`）**永远不会被编译**。这段代码要么是历史遗留（曾经 Mono 捆绑 .NET 8），要么说明 `Mono.Build.cs` 的版本号已经语义失真（Mono 的 `mono_*` API 与 ".NET 10" 没有对应关系）。无论哪种，`DotnetVersion.h` 的 `DOTNET8` 宏在实践中是**不可达宏**，且这种"嵌套且为子集"的版本判断（`DOTNET8` 内含于 `DOTNET9`）本身就是易错的写法。

**建议**
确认 iOS/Mono 路径当前实际需要哪一套注册：若 `monovm_initialize`（`:78-89`）已是唯一路径，则删除 `:62-74` 死块并把 `DotnetVersion.h` 的宏改为语义明确的名字（如 `MONO_BUNDLED_DOTNET_GE_9`），同时把 `Mono.Build.cs` 的版本号改为真实运行时版本；若仍需要旧路径，则修正 `Mono.Build.cs` 的版本常量。

**验证方式**
`grep -rn "DOTNET8\|DOTNET9\|DOTNET10" Source/`（应只有 `DotnetVersion.h` 与 `FMonoDomain.cpp:62/71/78`）；在 Mono 构建下用 `/P` 打印预处理结果（MSVC `/E`）确认该块未进入编译单元。

---

#### [F-CMP-017] `SourceCodeGenerator`：`bool.Parse` 与配置文件读取无异常保护，会直接终止 UHT

- **类别**: 异常与错误处理
- **严重度**: P2
- **文件**: `Source/SourceCodeGenerator/SourceCodeGenerator.cs:29-48`、`:113-143`、`:790`
- **函数**: `SourceCodeGeneratorExporter(IUhtExportFactory)`、`Generate()`、`SaveIfChanged()`
- **置信度**: 高（C# 语义确定；未在本机用非法 ini 值实测）
- **复核结论**: 确认（`bool.Parse` 只接受 `True/False`（忽略大小写），ini 写 `1` 必抛 `FormatException`；`File.ReadAllText` 的 IO 异常路径成立）
- **可达性**: 潜伏（**需先有 `Config/DefaultUnrealCSharpEditorSetting.ini`**；本工作区 `Config/` 下**没有**该文件（glob `Config/*.ini` 实测只有 `DefaultUnrealCSharpSetting.ini` 等 7 个）→ `:29 File.Exists` 为假，整个导出器直接返回；`bEnableExport` 默认 `false`（`UnrealCSharpEditorSetting.cpp:38`）也为第二道闸门）
- **复核证据**: `Source/SourceCodeGenerator/SourceCodeGenerator.cs:29-48` 已读全文（`:31 ConfigFile`、`:33-34 TryGetSection`、`:39-42 bool.Parse(Value.ToLower())`、`:44 Generate()`）；`:788-796` 已读（`:790 File.Exists(file) && File.ReadAllText(file) == builder.ToString()`）；调用点 `:348`（`ExportClass`）与 `:387`（`Finish`）**行号精确吻合**
- **级别变动**: 无（**补充限定**：原文"`SaveIfChanged` 由 `:348` 与 `:387` **并发**调用"表述不准——`:168 Task.WaitAll` 之后才执行 `:171 Finish()`，两者**不并发**；真正并发的是多个 `Factory.CreateTask`（`:185`）之间的 `ExportClass`→`SaveIfChanged`）

**现状（代码事实）**
```csharp
// SourceCodeGenerator.cs:29
    if (File.Exists(SettingFilePath))
    {
        var SettingConfigFile = new ConfigFile(new FileReference(SettingFilePath));   // 可能 IOException
        if (SettingConfigFile.TryGetSection("...", out var SettingConfigSection))
        {
            ...
            if (SettingConfigHierarchySection.TryGetValue("bEnableExport", out var Value))
            {
                if (bool.Parse(Value.ToLower()))        // ← "1"/"yes"/"" → FormatException / ArgumentNullException
```
```csharp
// :790
            if (File.Exists(file) && File.ReadAllText(file) == builder.ToString())   // ← IOException / UnauthorizedAccessException
```

**调用上下文**
入口是 UHT 导出器（`SourceCodeGenerator.cs:23`，由 UBT 在构建期加载 `Binaries/DotNET/UnrealBuildTool/Plugins/SourceCodeGenerator/` 后调用）。`SaveIfChanged` 由 `ExportClass`（`:348`）与 `Finish`（`:387`）**并发调用**（任务由 `Factory.CreateTask` 创建、`:168 Task.WaitAll` 等待）。

**问题**
该导出器运行在 UHT 进程内。任何未捕获异常都会让 UHT 以非零码退出并把 .NET 堆栈直接暴露给用户，表现为"构建在 UHT 阶段莫名失败"。具体触发点：（1）ini 里 `bEnableExport=1`（UE 的 ini 惯例允许 `1`/`True`，但 `bool.Parse` 只接受 `True/False`）→ `FormatException`；（2）`.inl` 目标文件被编辑器/杀软锁定或只读 → `File.ReadAllText` 抛 `IOException`/`UnauthorizedAccessException`。

**建议**
`Enum.TryParse`/`bool.TryParse` 并在失败时回退默认值 + 输出一条 warning；`SaveIfChanged` 用 `try/catch (Exception e) when (e is IOException or UnauthorizedAccessException)` 包裹并按 UHT 的方式报错（`Factory.Session` 有日志设施）。

**验证方式**
在 `Config/DefaultUnrealCSharpEditorSetting.ini` 里写 `bEnableExport=1` 触发一次构建（预期：`FormatException`）；把某个已生成的 `.inl` 设为只读再构建。

---

#### [F-CMP-018] `SourceCodeGenerator`：空 `ExportModule` 被 C# 语义解释为"全部跳过"，与"生成全部模块"的直觉相反

- **类别**: Bug（静默失效）
- **严重度**: P2
- **文件**: `Source/SourceCodeGenerator/SourceCodeGenerator.cs:111-162`（关键 `:155-159`）
- **函数**: `SourceCodeGenerator::Generate()`
- **置信度**: 高（`Enumerable.All` 对空序列返回 `true` 是 .NET 明确定义的行为）
- **复核结论**: 确认
- **可达性**: 潜伏（需 `bEnableExport=true` **且** ini 存在；本工作区 ini 缺失 → 当前不可达，但换工程/补 ini 即生效）
- **复核证据**: `Source/SourceCodeGenerator/SourceCodeGenerator.cs:111-162` 已读全文：`:111 var ExportModule = new List<string>();`、`:123-130 TryGetValues("ExportModule")`、`:155-159 ExportModule.All(...) → continue`、`:161 QueueClassExports(...)`；`:149-150` 只放行 `EngineRuntime`/`GameRuntime` 两类模块（**这一点原文未强调，是"空列表仍为 continue 全跳过"的第二层过滤**）；工程 `Config/` 无 `DefaultUnrealCSharpEditorSetting.ini`（glob 实测）
- **级别变动**: 无

**现状（代码事实）**
```csharp
// SourceCodeGenerator.cs:111
            var ExportModule = new List<string>();
            ...
                    if (SettingConfigHierarchySection.TryGetValues("ExportModule", out var Values)) { ... }
            ...
// :155
                if (ExportModule.All(x =>
                        string.Compare(x, module.Module.Name, StringComparison.OrdinalIgnoreCase) != 0))
                {
                    continue;                       // ← 空列表时 All 恒为 true → 每个模块都被 continue
                }

                QueueClassExports(module.ScriptPackage, tasks);
```
设置侧默认值：`bEnableExport(false)`（`UnrealCSharpEditorSetting.cpp:38`），`ExportModule` 无默认值（`UnrealCSharpEditorSetting.h:163-164`）。

**调用上下文**
`ExportModule` 由用户在 ini/设置面板里手工维护（实测样例：`+ExportModule=UMG`、`+ExportModule=Engine`，见 `Saved/Temp/Win64/UnrealCSharpTest/Config/DefaultUnrealCSharpEditorSetting.ini`）。本工作区**当前 `Config/` 下没有 `DefaultUnrealCSharpEditorSetting.ini`**（只有 `Saved/Temp/{Win32,Linux,Android}/...` 的历史副本），因此 `:29 File.Exists` 为假 → 连 `Generate()` 都不会被调用。

**问题**
（1）语义陷阱：`ExportModule` 为空时"跳过全部模块"而不是"导出全部"，与插件其他地方 `bIsGenerateAllModules`（`UnrealCSharpEditorSetting.cpp:35`）所表达的"全量"直觉相反，用户填了 `bEnableExport=true` 却什么都没生成，且**没有任何日志**。（2）双重门控（`bEnableExport` + 非空 `ExportModule`）叠加"缺 ini 文件时完全静默"（`:29`），一旦 ini 丢失（本工作区即是此状态），构建期绑定代码生成会静默停摆，而 `Intermediate/` 下 631 个**旧** `.binding.inl` 仍在参与编译（实测 mtime 2026-05-18，早于 `Script/` 的 2026-07-20），产物与源码版本脱节。

**建议**
（1）`ExportModule` 为空时按"全部 EngineRuntime/GameRuntime 模块"处理（并打一条 warning），或至少在 `ExportModule.Count == 0` 时明确记录"未配置 ExportModule，跳过导出"；（2）`SourceCodeGeneratorExporter` 在 `File.Exists(SettingFilePath) == false` 时输出一条明确警告（当前是完全静默）。

**验证方式**
`grep -n "ExportModule" -r Plugins/UnrealCSharp`；把 `ExportModule` 从 ini 中删除后构建，观察是否无任何提示地不生成；对比 `Intermediate/.../UHT/` 下文件 mtime 与源码 mtime。

---

#### [F-CMP-019] `SourceCodeGenerator`：字典索引器 + `Path.Combine` 绝对路径吞噬第一参数 → 既可能抛异常又生成了绝对 `#include`

- **类别**: Bug / 平台兼容
- **严重度**: P2
- **文件**: `Source/SourceCodeGenerator/SourceCodeGenerator.cs:59`、`:773-776`、`:783-786`、`:346`
- **函数**: `SourceCodeGenerator::GetHeaderFile()`、`GetInclude()`
- **置信度**: 高（生成物已直接证实）
- **复核结论**: 确认（**生成物实证已重新核对**：绝对路径 `#include` 确实存在）
- **可达性**: 潜伏（受 F-CMP-018 同一门控：需 ini 存在 + `bEnableExport=true`；生成物是**历史遗留**，非本次构建产生）
- **复核证据**: `Source/SourceCodeGenerator/SourceCodeGenerator.cs:773-776` 已读（`return Path.Combine(HeaderPath[classObj.Package.Module.ShortName], classObj.HeaderFile.FilePath);`——字典索引器 + 绝对第二参数）；`:783-786` 已读；`:346` 附近的 `Factory.MakePath` 未读（不影响结论）。**生成物**：`Plugins/UnrealCSharp/Intermediate/Build/Win64/UnrealEditor/Inc/UnrealCSharp/UHT/Actor.binding.inl` 第 **10-12** 行确为 `#include "Engine\Source\Runtime\Engine\Classes\...\*.h"`（**注意：原文写"第 8-10 行"，实际 include 在第 10-12 行，行号偏 2**）
- **级别变动**: 无（**行号修正**：生成物引用 `8-10` → `10-12`）

**现状（代码事实）**
```csharp
// SourceCodeGenerator.cs:773
        private string GetHeaderFile(UhtClass classObj)
        {
            return Path.Combine(HeaderPath[classObj.Package.Module.ShortName], classObj.HeaderFile.FilePath);
        }
```
`HeaderPath` 的填充（`:824-848`）用的是 `Item.DirectoryName`（**绝对目录**）；`classObj.HeaderFile.FilePath` 是 UHT 的**完整文件路径**。`Path.Combine(a, b)` 在 `b` 为绝对路径时**直接返回 `b`**，即第一参数被丢弃。

生成物实证（本工作区 `Plugins/UnrealCSharp/Intermediate/Build/Win64/UnrealEditor/Inc/UnrealCSharp/UHT/Actor.binding.inl` 第 8-10 行）：
```cpp
#include "Engine\Source\Runtime\Engine\Classes\GameFramework\Actor.h"
#include "Engine\Source\Runtime\Engine\Classes\Components\InputComponent.h"
```

**调用上下文**
`GetInclude` 由 `ExportClass(StringBuilder, UhtClass):451` 在并行任务中调用（`Factory.CreateTask`，`:185`）。

**问题**
（1）**`HeaderPath[...]` 的取值被完全丢弃**，这个字典的唯一实际作用是"当 `ShortName` 不在表中时抛 `KeyNotFoundException`"（未用 `TryGetValue`）→ 直接终止 UHT。表只由硬编码目录列表填充（`:86-104`：工程 `Source/`、工程 `Plugins/`、`Engine/Source/{Developer,Editor,Programs,Runtime}`、`Engine/Plugins/`），凡是位于这些目录之外的模块（例如 `Engine/Source/ThirdParty/` 下带 UCLASS 的模块、通过 `AdditionalPluginDirectories` 引入的外部插件、`Engine/Source/Programs` 之外的 Engine/Source 顶层模块）都会让构建崩溃。（2）**生成物里写死了绝对路径**：换机器、换引擎安装路径、CI 与本地路径不同时，这些 `#include` 就无法解析；同时绝对路径进入产物内容也破坏了"内容相同则跳过"的缓存判定（`SaveIfChanged:790`），造成无意义的重新生成。（3）本机路径含中文/空格时，绝对路径还会引出编码问题（见 F-CMP-023）。

**建议**
（1）`GetHeaderFile` 改为只用 `classObj.HeaderFile.FilePath`（去掉无意义的字典查找），或改为把绝对路径转成相对 `Session.ProjectDirectory`/`EngineDirectory` 的相对路径；（2）若确实需要按模块校验，改用 `HeaderPath.TryGetValue(...)`，失败时记录 warning 并跳过依赖而不是崩溃。

**验证方式**
`grep -rn '#include "D:' Intermediate/Build/*/*/Inc/*/UHT/ | head`（本工作区可直接复现）；`grep -n "HeaderPath" SourceCodeGenerator.cs` 确认只有写入与一次索引读取。

---

#### [F-CMP-020] `SourceCodeGenerator`：对不存在的目录调用 `DirectoryInfo.GetFiles` 会抛 `DirectoryNotFoundException`

- **类别**: 异常与错误处理 / 平台兼容
- **严重度**: P2
- **文件**: `Source/SourceCodeGenerator/SourceCodeGenerator.cs:86-104`、`:798-822`、`:824-848`
- **函数**: `SourceCodeGenerator::Generate()` → `GetModules()`/`GetPlugins()`
- **置信度**: 高（机制确定）；**触发条件未在本工作区复现**（本工程存在 `Plugins/` 目录）
- **复核结论**: 确认（机制成立；`.NET` 的 `DirectoryInfo.GetFiles` 对不存在目录确实抛 `DirectoryNotFoundException`）
- **可达性**: 潜伏（本工作区 `UnrealCSharpTest/Plugins/UnrealCSharp` 存在 → 不触发；换到"插件装在引擎侧/无工程 Plugins 目录"的布局即触发）
- **复核证据**: `Source/SourceCodeGenerator/SourceCodeGenerator.cs:820-822` 与 `:830-848` 已读；`:86-104` 的 9 次扫描调用已读（`:86` `Source/`、`:90` 工程 `Plugins/`、`:94-102` 引擎 5 个子目录、`:104` 引擎 `Plugins/`）。**注意原文的行号小偏差**：原文称"`:848`/`:871` 的 HashSet 重载在用 `Item.DirectoryName != null` 做判断"——`:848`/`:871` 实为两个函数的右花括号，真实判空点是 `:832`/`:842`（Dictionary 重载）与 `:865`（HashSet 重载内层）；两处外层 `GetFiles` 均无 `Directory.Exists` 守卫，**结论不受影响**
- **级别变动**: 无（**行号修正**：`:848`/`:871` → `:832`/`:842`/`:865`）

**现状（代码事实）**
```csharp
// SourceCodeGenerator.cs:86
                GetModules(Path.GetFullPath(Path.Combine(Session.ProjectDirectory, "Source/")), Project);
                GetModules(Path.GetFullPath(Path.Combine(Session.ProjectDirectory, "Source/")), HeaderPath);
                GetModules(Path.GetFullPath(Path.Combine(Session.ProjectDirectory, "Plugins/")), HeaderPath);   // ← 目录可以不存在
                GetPlugins(Path.GetFullPath(Path.Combine(Session.ProjectDirectory, "Plugins/")), HeaderPath);
                GetModules(Path.GetFullPath(Path.Combine(Session.EngineDirectory, "Source/Developer/")), HeaderPath);
                ...
```
`new DirectoryInfo(path)` 对不存在的路径不抛异常，但随后的 `DirectoryInfo.GetFiles(...)`（`:830`、`:804`、`:856`、`:814`）会抛 `DirectoryNotFoundException`；只有 `:848`/`:871` 的 `HashSet` 重载在用 `Item.DirectoryName != null` 做判断，`Dictionary` 重载（`:834`、`:844`）与外层的 `GetFiles` 无任何守卫。

**调用上下文**
`Generate()` 由导出器入口在**每次 UHT 运行**时调用（`:44`）。

**问题**
"工程目录下没有 `Plugins/` 文件夹"是完全正常的工程形态（插件通常装在引擎侧或通过 Marketplace 安装到别处）。此时 `GetModules(Project/Plugins/)` 直接抛异常 → UHT 以未处理异常退出 → 用户看到构建在 UHT 阶段失败且错误信息是 C# 堆栈。本工作区因为 `UnrealCSharpTest/Plugins/UnrealCSharp` 存在而不会触发，所以属于"换一个工程布局就会炸"的隐患。

**建议**
在扫描前统一 `if (!Directory.Exists(path)) return;`（或 `DirectoryInfo.Exists` 判断），并在引擎目录不存在时给出明确诊断。

**验证方式**
临时把 `Session.ProjectDirectory/Plugins` 改名（或在一个没有 `Plugins/` 目录的空工程里安装该插件）后构建，观察 UHT 报错。

---

#### [F-CMP-021] `SourceCodeGenerator`：`HashSet` 遍历顺序导致生成物不确定，`SaveIfChanged` 的增量缓存失效

- **类别**: 性能 / 可复现性
- **严重度**: P2
- **文件**: `Source/SourceCodeGenerator/SourceCodeGenerator.cs:398`、`:449-452`、`:356-388`、`:788-796`
- **函数**: `ExportClass(StringBuilder, UhtClass)`、`Finish()`、`SaveIfChanged()`
- **置信度**: 中（`HashSet<UhtClass>`（引用相等）的顺序取决于对象哈希/分配顺序；未做多次运行对比）
- **复核结论**: 部分确认（偏差：机制成立但**等级过高**。`HashSet<UhtClass>` 无自定义比较器 → 枚举顺序不稳定是事实；但 `UhtClass` 的默认 `GetHashCode` 与对象分配顺序相关，同一进程内多次运行通常稳定，"周期性失效"缺乏证据 → 由 P2 下调为 P3）
- **可达性**: 潜伏（同 F-CMP-018 门控；且即便生效，后果仅为产物抖动与无谓重写）
- **复核证据**: `Source/SourceCodeGenerator/SourceCodeGenerator.cs:398`（`new HashSet<UhtClass> { classObj }`）、`:449-452`（`foreach (var DependencyClass in DependencyClasses) builder.Append(GetInclude(DependencyClass))`）、`:468-471`、`:356-388`（`Finish()`：`:356 Dictionary<UhtPackage, List<string>>`、`:368 foreach packages`、`:370 package.Value.Sort()`）、`:788-796`（`SaveIfChanged`）——均已读，行号精确吻合
- **级别变动**: P2→P3（影响限于产物内容抖动/重写，不产生错误代码；且未实测证明跨运行不稳定）

**现状（代码事实）**
```csharp
// SourceCodeGenerator.cs:398
            var DependencyClasses = new HashSet<UhtClass> { classObj };
...
// :449
            foreach (var DependencyClass in DependencyClasses)
            {
                builder.Append(GetInclude(DependencyClass));      // ← include 顺序 = HashSet 枚举顺序，不稳定
            }
```
```csharp
// :788
        private void SaveIfChanged(string file, StringBuilder builder)
        {
            if (File.Exists(file) && File.ReadAllText(file) == builder.ToString()) { return; }
            Factory.CommitOutput(file, builder);
        }
```

**调用上下文**
`ExportClass` 在并行任务里执行（`:185`），`Finish()` 在本线程串行执行（`:171`）；`SaveIfChanged` 是唯一写盘点。

**问题**
`HashSet<UhtClass>` 没有自定义 `IEqualityComparer`，其枚举顺序在 .NET 中不保证稳定（与对象哈希/插入历史有关）。因此同一份输入在不同次运行里可能产出**不同的 `#include` 顺序**，`SaveIfChanged` 的"内容相同则跳过"于是周期性失效 → UHT 输出文件被反复重写 → 下游 C++ 编译的依赖时间戳变化 → 大量无意义的重编译（在 631 个文件的规模上尤其明显）。`Finish()` 内 `foreach (var package in packages)`（`Dictionary<UhtPackage, List<...>>`，`:368`）也是同理（只影响文件写出顺序，危害较小）；`package.Value.Sort()`（`:370`）说明作者已经在单个包内做过排序，但没有对被 include 的依赖类排序。

**建议**
把 `DependencyClasses` 换成 `SortedSet<UhtClass>`（自定义比较器按 `EngineName` 排序）或在写出前对结果 `OrderBy(x => x.EngineName, StringComparer.Ordinal)`。

**验证方式**
连续运行两次相同构建，`fc` 比对同一 `.binding.inl`；或 `grep -n "HashSet<UhtClass>" SourceCodeGenerator.cs` 后加 `OrderBy` 做 A/B。

---

#### [F-CMP-022] `SourceCodeGenerator`：生成文件无 BOM，路径含非 ASCII 时 MSVC 可能按 ANSI 解析

- **类别**: 平台兼容（编码）
- **严重度**: P2
- **文件**: `Source/SourceCodeGenerator/SourceCodeGenerator.cs:67-72`、`:378`、`:395-396`、`:469`、`:788-796`
- **函数**: `SourceCodeGenerator::SaveIfChanged()`（写入编码由 `Factory.CommitOutput` 决定）
- **复核结论**: 无法验证（卡点：`EpicGames.UHT` 的 `UhtExportFactory.CommitOutput` 实现不在插件源码内，本机也无法定位其源码 → 无法确认写入编码是否带 BOM。**实测可确证的部分**：现存生成的 `.binding.inl` 无 BOM 且内容为纯 ASCII）
- **可达性**: 潜伏（同 F-CMP-018 门控；且需要"路径含非 ASCII"这一额外前提 —— 本项目所在路径为纯 ASCII（不含非 ASCII 字符），故**当前不可达**）
- **复核证据**: `Source/SourceCodeGenerator/SourceCodeGenerator.cs:378`（`builder.Append("#pragma once\r\n\r\n");` 显式 CRLF）、`:395-396`、`:469`、`:788-796` 均已读；生成物 `.../UHT/Actor.binding.inl:1` 首行为 `/*===...`（read 输出未见 BOM，内容全 ASCII）
- **级别变动**: 无（P2；因原文自标"置信度低 + 未验证假设"，级别依赖该假设成立，若后续确认 `CommitOutput` 写 UTF-8 则应降为 P3）
- **置信度**: 低（**未验证的假设**：`EpicGames.UHT` 的 `UhtExportFactory.CommitOutput` 源码不在本机磁盘上，我无法确认它写 UTF-8 是否带 BOM；生成物仅含 ASCII，因此当前无法从产物反推）

**现状（代码事实）**
```csharp
// SourceCodeGenerator.cs:378
                builder.Append("#pragma once\r\n\r\n");        // 显式 CRLF
// :788
        private void SaveIfChanged(string file, StringBuilder builder)
        {
            if (File.Exists(file) && File.ReadAllText(file) == builder.ToString()) { return; }
            Factory.CommitOutput(file, builder);              // 编码由 UHT 决定
        }
```
实测生成物首字节不是 UTF-8 BOM（`Get-Content` 读取 `Actor.binding.inl` 首行为 `/*===...`，无 BOM 字符），文件内容为纯 ASCII（含绝对路径）。

**调用上下文**
产物被 C++ 侧 `#include`（`.header.inl` 由 `Finish()` 生成，`:385-387`），由 MSVC 编译。

**问题**
MSVC 在没有 BOM 且未指定 `/utf-8` 时按**系统 ANSI 代码页**解释源文件。生成的 `#include` 里会写入**绝对路径**（见 F-CMP-019，例如 `D:\项目\Source\Foo.h`）。若机器路径含中文，写出的 UTF-8 字节会被 MSVC 当成 GBK 解读 → 路径损坏 → `#include` 找不到文件（且报错信息本身可能也是乱码，极难排查）。同理，`ExportClass` 里若有含非 ASCII 的注释/标识符也会受影响。

**建议**
（1）修掉 F-CMP-019 的绝对路径问题（这是根因）；（2）若确实存在非 ASCII 内容，写入时显式使用 `new UTF8Encoding(true)`（带 BOM）——需先确认 `Factory.CommitOutput` 是否可控；若不可控，则改为自行 `File.WriteAllText(file, content, new UTF8Encoding(true))`（但会失去 UHT 的输出追踪）。UE 侧也可在 `.Build.cs` 加 `/utf-8`。

**验证方式**
在含中文的路径下构建一次，`Format-Hex` 查看生成的 `.binding.inl` 前 3 字节（EF BB BF？），并检查是否出现 `fatal error C1083`。

---

### P3

#### [F-CMP-023] `SetOutput`/`PendingOutput` 无上限累积，超长单行输出会持续增长

- **类别**: 性能
- **严重度**: P3
- **文件**: `Source/Compiler/Private/FCSharpCompileProgress.cpp:39-55`；`Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1529`、`:1560`
- **函数**: `FCSharpCompileProgress::SetOutput()`、`SyncProcess::ReadOutput`
- **置信度**: 高
- **复核结论**: 确认
- **可达性**: 活跃
- **复核证据**: `Source/Compiler/Private/FCSharpCompileProgress.cpp:39-55` 已读（`:41 FScopeLock`、`:43 PendingOutput.Append(InOutput)`、`:47-54` 仅在遇到 `'\n'` 时 `MidInline` 裁剪，**无长度上限**）；`Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1529`（`FString Result;`）、`:1560`（`Result.Append(Output)`）、`:1576`（收尾再读一次）、`:1585`（`InOnComplete(ReturnCode, Result)`）均已读
- **级别变动**: 无

**现状（代码事实）**
```cpp
// FCSharpCompileProgress.cpp:43
	PendingOutput.Append(InOutput);          // 只在遇到 '\n' 时才被裁剪
```
```cpp
// FUnrealCSharpFunctionLibrary.cpp:1560
			Result.Append(Output);           // 累积整个构建日志，无上限
```

**调用上下文**
两者都在编译期间持续增长；`Result` 最后作为一个 `FString` 传给 `OnComplete`（`:1585`），`PendingOutput` 由 `Flush`（`:57-64`）在结束时处理。

**问题**
若子进程长时间输出无换行内容（进度条、二进制日志），`PendingOutput` 会持续增长；`Result` 则始终持有整个日志（大型工程 + `-v:d` 级输出可达数十 MB），仅在函数返回时释放。属于浪费而非泄漏。

**建议**
`Result` 保留最近 N KB（失败时也足够给出上下文）或直接流式转发给 `InOnOutput` 而不聚合；`PendingOutput` 超过阈值（如 64KB）时强制切行。

**验证方式**
用一个输出 `yes` 之类的假 dotnet 替换后观察内存峰值。

---

#### [F-CMP-024] `GetStage()` 用 `const_cast` 加锁；`Stage` 常量与运行路径语义不一致（命名误导）

- **类别**: 可优化/可读性
- **严重度**: P3
- **文件**: `Source/Compiler/Private/FCSharpCompileProgress.cpp:66-71`、`:125-133`
- **函数**: `FCSharpCompileProgress::GetStage()`、`ParseLine()`
- **置信度**: 高
- **复核结论**: 确认（`const_cast` 加锁与"常量名/工程名相反但语义自洽"两点均已逐字核实；原文关于"不是 bug 但命名误导"的裁决**成立**）
- **可达性**: 活跃
- **复核证据**: `Source/Compiler/Private/FCSharpCompileProgress.cpp:66-71` 已读（**:68** `FScopeLock ScopeLock(&const_cast<FCSharpCompileProgress*>(this)->CriticalSection);`）；`:119-133` 已读（**:125** `if (Project == UEName) { return StageCompilingGame; }`、**:130** `if (Project == GameName) { return StagePublishing; }`）；常量定义 `:5-19`（8 个 `Stage*`）已读；调用方 `FCSharpCompilerRunnable.cpp:369`（ticker 内）、`FEditorListener.cpp:729` 已读
- **级别变动**: 无

**现状（代码事实）**
```cpp
// FCSharpCompileProgress.cpp:66
FString FCSharpCompileProgress::GetStage() const
{
	FScopeLock ScopeLock(&const_cast<FCSharpCompileProgress*>(this)->CriticalSection);
	return Stage;
}
```
```cpp
// FCSharpCompileProgress.cpp:125
	if (Project == UEName)  { return StageCompilingGame; }     // 名字里写着 "Compiling Game"
	if (Project == GameName) { return StagePublishing; }       // 名字里写着 "Publishing"
```

**调用上下文**
`GetStage` 的调用者是工作线程内的 ticker（`FCSharpCompilerRunnable.cpp:369`，经 `AsyncTask` 在游戏线程执行）与 `FEditorListener.cpp:729`；`ParseLine` 的调用者是 `ProcessLine`（`SetOutput`/`Flush` 内）。

**问题**
（1）`const_cast` 破坏 const 语义，应把 `CriticalSection` 声明为 `mutable`。（2）**这里我要明确说明经核对不是 bug**：`UEName` 分支返回 `StageCompilingGame` 意味着"UE 工程（默认 `UE`，`Macro.h:19`）的 dll 出现后，接下来是游戏工程编译"；`GameName` 分支返回 `StagePublishing` 意味着"游戏工程（默认 `Game`，`Macro.h:21`）的 dll 出现后，接下来是 `Game.props:16-18` 的 `AfterBuildPublish` 发布步骤"——语义自洽（阶段随"上一个工程产出"前进）。但常量名与实际指代的工程名相反，后续维护者极易"顺手修正"成真正的 bug。

**建议**
`mutable FCriticalSection`；把常量改名为 `StageCompilingGame` → `StageNextCompilingGame`/`StageGameBuilding`，或在 `ParseLine` 处加注释说明"命中上一工程的 dll 行表示进入下一阶段"。

**验证方式**
`grep -n "StageCompilingGame\|StagePublishing" Source/ -r`；观察一次成功编译的通知文本序列（应为 Preparing → Restoring → Compiling UE → Compiling Game → Publishing → Compile succeeded）。

---

#### [F-CMP-025] `dotnet` 路径被函数内 `static` 永久缓存，改设置不生效且重复两处

- **类别**: Bug（配置生效性）
- **严重度**: P3
- **文件**: `Source/Compiler/Private/FCSharpCompilerRunnable.cpp:279`、`:384`
- **函数**: `FCSharpCompilerRunnable::CompileInterop()`、`FCSharpCompilerRunnable::Compile()`
- **置信度**: 高
- **复核结论**: 确认
- **可达性**: 活跃
- **复核证据**: `Source/Compiler/Private/FCSharpCompilerRunnable.cpp:279`（`static auto CompileTool = FUnrealCSharpFunctionLibrary::GetDotNet();`，位于 `CompileInterop` 内）与 `:384`（同样的 `static`，位于 `Compile` 内）均已读；两处随后分别于 `:291`、`:413` 传给 `SyncProcess`（已读）；而 `GetDotNet()` 每次都会读 ini（`FUnrealCSharpFunctionLibrary.cpp:44-49`，已读）→ "改设置不生效"成立
- **级别变动**: 无

**现状（代码事实）**
```cpp
// :279（CompileInterop 内）
		static auto CompileTool = FUnrealCSharpFunctionLibrary::GetDotNet();
// :384（Compile 内）
	static auto CompileTool = FUnrealCSharpFunctionLibrary::GetDotNet();
```

**调用上下文**
两者都在编译时读取（`:291`、`:413` 传给 `SyncProcess`）。

**问题**
函数内静态初始化只执行一次，之后即使 `UUnrealCSharpEditorSetting::DotNetPath` 被改动（设置面板/ini），编译仍用旧路径直到重启编辑器——与 `DotNetPath` 作为 `EditAnywhere` 配置项的期望不符（用户在设置里改为新路径后立刻编译，发现"没生效"）。同样的逻辑写了两份，也属重复。

**建议**
去掉 `static`（`GetDotNet()` 只是读一个 `FString`，开销可忽略），或把结果缓存到一个可被设置变更失效的成员里。

**验证方式**
在设置面板把 `DotNetPath` 改为一个打印版本号的包装脚本，不重启编辑器执行编译，观察是否使用新路径。

---

#### [F-CMP-026] 死字段：`SourceCodeGenerator.Project` 只写不读

- **类别**: 死代码
- **严重度**: P3
- **文件**: `Source/SourceCodeGenerator/SourceCodeGenerator.cs:57`、`:86`
- **函数**: `SourceCodeGenerator::Generate()`
- **置信度**: 高（873 行全文已读，标识符 `Project` 仅出现在这两行）
- **复核结论**: 确认（**873 行全文已重读**：`Project` 作为独立标识符只出现在 `:57` 声明与 `:86` 传参处，`:88-104` 全部填充 `HeaderPath`，`:850-871` 的 `HashSet` 重载只被 `:86` 调用一次）
- **可达性**: 潜伏（同 F-CMP-018：`Config/DefaultUnrealCSharpEditorSetting.ini` 缺失 + `bEnableExport` 默认 `false` → 当前不执行该扫描）
- **复核证据**: `Source/SourceCodeGenerator/SourceCodeGenerator.cs:57`（`private HashSet<string> Project = new();`）、`:86`（`GetModules(Path.GetFullPath(Path.Combine(Session.ProjectDirectory, "Source/")), Project);`）、`:88-104`（9 次 `HeaderPath` 填充）、`:850-871`（`GetModules(string, HashSet<string>)` 定义）——均已读
- **级别变动**: 无

**现状（代码事实）**
```csharp
// :57
        private HashSet<string> Project = new();
// :86
                GetModules(Path.GetFullPath(Path.Combine(Session.ProjectDirectory, "Source/")), Project);
```
后续（`:88-104`）全部填充 `HeaderPath`，再无任何读取 `Project` 的语句。

**调用上下文**
`Generate()` 内一次性使用，作用域受限。

**问题**
多出一次全目录递归扫描（`GetModules` 会遍历 `Project/Source` 下所有 `*.Build.cs`），结果被丢弃——纯浪费，且容易让读者误以为它在过滤工程模块。

**建议**
删除字段与该行调用（保留 `HeaderPath` 的填充）。

**验证方式**
`grep -n "Project\b" SourceCodeGenerator.cs` 对照（注意区分 `ProjectDirectory`/`ProjectFile`）。

---

#### [F-CMP-027] 死宏：`VS2022`、`STD_CPP_11/14/17` 定义后从未被使用

- **类别**: 死代码
- **严重度**: P3
- **文件**: `Source/CrossVersion/Public/VSVersion.h:4`、`:8`；`Source/CrossVersion/Public/CppVersion.h:3`、`:5`、`:7`
- **函数**: 宏（无函数）
- **置信度**: 高（全插件 1422 个文本文件的脚本化计数，见 §4）
- **复核结论**: 确认（**grep 已重跑**，结论与 §4 表一致）
- **可达性**: 不可达（死宏：定义存在，但任何配置下都没有使用点，永不参与条件编译）
- **复核证据**: 工具 grep 重跑，作用域 `Plugins/UnrealCSharp/Source`，模式 `STD_CPP_|VS2022|VS2026|DOTNET8|DOTNET9|DOTNET10` → 共 **18** 命中；其中 `VS2022` 仅 `Source/CrossVersion/Public/VSVersion.h:4`、`:8` 两条 `#define`（**0 使用**）；`STD_CPP_11/14/17` 仅 `CppVersion.h:3`/`:5`/`:7` 各 1 命中（**0 使用**）；`STD_CPP_20` 有 3 处真实使用；`VS2026` 有 1 处真实使用 `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:445`
- **级别变动**: 无

**现状（代码事实）**
```cpp
// VSVersion.h:3
#ifdef _MSC_VER
#define VS2022 _MSC_VER >= 1930 && _MSC_VER < 1950
#define VS2026 _MSC_VER >= 1950
#else
#define VS2022 0
#define VS2026 0
#endif
```
`CppVersion.h` 的 4 个宏中只有 `STD_CPP_20` 有使用点（3 处）。

**调用上下文**
`VS2026` 唯一使用点：`FSolutionGenerator.cpp:445 #if !VS2026`（决定是否给 `.sln` 写注释头）。`VS2022`、`STD_CPP_11/14/17` 使用点 0。

**问题**
死宏本身无害，但它们会让读者误判"插件区分 VS2019/VS2022"或"插件按 C++11/14/17 分支"。尤其 `VS2022` 的双分支写法暗示了跨工具集兼容，而实际代码只区分 `VS2026`。

**建议**
删除 `VS2022` 与 `STD_CPP_11/14/17`；若保留是为了对外 API 兼容，请在文件顶注明"预留未用"。

**验证方式**
见 §4 表格的 grep 证据（`\bVS2022\b` 全库 2 命中，均在 `VSVersion.h` 自身的两个 `#define` 分支）。

---

#### [F-CMP-028] `Compiler.Build.cs` 依赖 `EditorStyle`，但在 ≥5.1 的代码路径上并不需要

- **类别**: 可优化/一致性
- **严重度**: P3
- **文件**: `Source/Compiler/Compiler.Build.cs:35-48`；使用点 `Source/Compiler/Private/FCSharpCompilerRunnable.cpp:13-17`、`:340-344`、`:457-461`
- **函数**: 构建脚本
- **置信度**: 中（`EditorStyle` 模块本身在 5.6 仍存在，故不是错误依赖）
- **复核结论**: 确认
- **可达性**: 活跃（该依赖声明在当前 5.6 构建中实际生效，只是冗余）
- **复核证据**: `Source/Compiler/Compiler.Build.cs` **58 行全文已读**：`EditorStyle` 位于 `:45`（`PrivateDependencyModuleNames.AddRange` 块 `:35-48`），与 `CoreUObject/Engine/Slate/SlateCore/Json/UnrealCSharpCore/CrossVersion` 并列；`PublicDependencyModuleNames` 为 `Core`+`DirectoryWatcher`（`:25-32`）；使用点 `FCSharpCompilerRunnable.cpp:13-17`（`:14` `#include "Styling/AppStyle.h"`）、`:340-344`、`:457-461` 已读；`UEVersion.h:64` 的 `UE_F_APP_STYLE_GET_BRUSH` 定义未逐行读
- **级别变动**: 无

**现状（代码事实）**
```cpp
// FCSharpCompilerRunnable.cpp:13
#if UE_F_APP_STYLE_GET_BRUSH
#include "Styling/AppStyle.h"
#else
#include "EditorStyleSet.h"
#endif
```
`UE_F_APP_STYLE_GET_BRUSH UE_VERSION_START(5,1,0)`（`UEVersion.h:64`）在 5.6 上恒真 → 永远走 `FAppStyle`（属于 `SlateCore`）。只有在 UE 5.0 上才需要 `EditorStyleSet.h`。

**调用上下文**
`Compiler.Build.cs:25-48` 的依赖列表被所有编译器模块编译单元共享。

**问题**
`EditorStyle` 依赖在 5.1+ 的编译中是多余的（并且 `EditorStyle` 在 UE5 中是被标注为过渡性质的模块），会影响模块初始化顺序与包体。不是功能缺陷。

**建议**
把 `EditorStyle` 移到 `#if !UE_F_APP_STYLE_GET_BRUSH` 条件下的动态依赖（或直接保留并加注释说明是为 UE 5.0 兼容）。

**验证方式**
在 5.6 下注释掉 `EditorStyle` 依赖后编译 Compiler 模块（应通过）；或在 5.0 下确认需要它。

---

#### [F-CMP-029] `VS2026` 门控只在 5.6/UE 侧无法再验证：`.sln` 注释头在非 MSVC 平台也会被写入

- **类别**: 平台兼容（说明）
- **严重度**: P3
- **文件**: `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:443-457`；`Source/CrossVersion/Public/VSVersion.h:6-10`
- **函数**: `FSolutionGenerator::AddSolutionGeneratorHeaderComment()`
- **置信度**: 中
- **复核结论**: 确认（"未发现平台错路"的自述结论成立；`#` 是 `.sln` 的合法注释前缀）
- **可达性**: 潜伏（仅在生成 `.sln` 时执行；非 MSVC 平台会走 `#if !VS2026` 分支，但行为无害且与 Windows 一致）
- **复核证据**: `Source/CrossVersion/Public/VSVersion.h:3-11` **全文已读**（`:6` MSVC 分支 `#define VS2026 _MSC_VER >= 1950`、`:10` 非 MSVC `#define VS2026 0`）；使用点 grep 命中 `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:445`（`#if !VS2026`）。`FSolutionGenerator.cpp:443-457` 函数体**未逐行读**
- **级别变动**: 无

**现状（代码事实）**
```cpp
// FSolutionGenerator.cpp:445
#if !VS2026
	OutResult = FString::Printf(TEXT("# ====...\r\n%s"), *OutResult);   // 给 .sln 头部插入 '#' 注释
#endif
```
`VSVersion.h:10` 在非 MSVC 上 `#define VS2026 0` → 非 MSVC 平台同样会插入注释头。

**调用上下文**
`Generator()` 的 `.sln` 生成路径（`FSolutionGenerator.cpp:126-136`）。

**问题**
这不是"走错分支"：`#` 是 `.sln` 合法注释前缀，非 MSVC 机器上生成同样内容并无害。我核对后**未发现平台错路**，仅记录：`VS2022`/`VS2026` 的"非 MSVC 即 0"约定让 Linux/macOS 上恒为"非 VS2026"，语义上等价于"未知 VS 版本时保守不加特殊处理"，可以接受；但若将来有人把 `VS2022` 用作"仅 Windows 才启用某功能"的判据，就会在 Linux/macOS 上意外成立（因为它是 0，`#if VS2022` 为假，但 `#if !VS2022` 为真）。

**建议**
若意图是"仅 Windows"，请直接用 `PLATFORM_WINDOWS`，不要复用 VS 版本宏；并在 `VSVersion.h` 顶部注明"非 MSVC 平台恒 0"。

**验证方式**
`grep -rn "VS2022\|VS2026" Source/`（见 §4）。

---

#### [F-CMP-030] `SyncProcess` 在进程结束后仍无条件 `TerminateProc`，并复用同一管道作为子进程 stdin

- **类别**: 可优化/可读性（说明性）
- **严重度**: P3
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1543-1553`、`:1587-1591`
- **函数**: `FUnrealCSharpFunctionLibrary::SyncProcess`
- **置信度**: 高（引擎侧语义已核对）
- **复核结论**: 确认（引擎侧参数语义已**回源码逐字核实**）
- **可达性**: 活跃（每次编译都走到 `:1543-1553`/`:1587-1591`）
- **复核证据**: 插件侧 `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1543-1553`（第 9/10 实参为 `WritePipe`/`ReadPipe`）、`:1587` `ClosePipe`、`:1589` `TerminateProc(ProcessHandle, true)`、`:1591` `CloseProc` 均已读。引擎侧 `Engine\Source\Runtime\Core\Private\Windows\WindowsPlatformProcess.cpp:509-523` 的 `STARTUPINFO` 初始化明确：`:520 hStdInput = HANDLE(PipeReadChild)`、`:521 hStdOutput = HANDLE(PipeWriteChild)`、`:522 hStdError = HANDLE(PipeStdErrChild)`；`:463-467` 的 10 参数重载把 `PipeWriteChild` 同时用为 `PipeStdErrChild`（"保留旧行为"）；`:503-506` 由 `STARTF_USESTDHANDLES` 决定 `bInheritHandles` → 报告"把第 10 参数改为 `nullptr` 仍能捕获输出"的技术判断**成立**
- **级别变动**: 无（**引用计数修正**：原文"`SyncProcess` 的全部 5 个调用点"应为 **6 个**）

**现状（代码事实）**
```cpp
// :1543
	auto ProcessHandle = FPlatformProcess::CreateProc(*InURL, *InParms, false, true, true, nullptr, 1,
	                                                  WorkingDirectory, WritePipe, ReadPipe);   // 第 9/10 参数 = PipeWriteChild/PipeReadChild
// :1587
	FPlatformProcess::ClosePipe(ReadPipe, WritePipe);
	FPlatformProcess::TerminateProc(ProcessHandle, true);
	FPlatformProcess::CloseProc(ProcessHandle);
```
引擎侧：`WindowsPlatformProcess.cpp:520-522` 把 `PipeReadChild` 当作 `StartupInfo.hStdInput`、`PipeWriteChild` 当作 `hStdOutput`/`hStdError`（`:465-467`）。

**调用上下文**
`SyncProcess` 的全部 5 个调用点（见 §2.5）。

**问题**
（1）`TerminateProc` 在进程**已经退出**后才调用：对已退出进程是空操作（`GetProcReturnCode` 返回真说明已退出，`:1580`），保留它可能是为了回收孙进程树（MSBuild 节点/VBCSCompiler），但这属于"事后清理"，不能替代超时处理（F-CMP-002）。（2）把 `ReadPipe` 同时作为子进程 stdin（第 10 参数），意味着子进程的 stdin 与父进程的 stdout 读取端是**同一根管道对象**；本用例（不向子进程写数据）安全，但语义上应收敛为"只取输出"——把第 10 参数传 `nullptr` 更清晰（在 5.6 下 `dwFlags` 仍会因 `PipeWriteChild != nullptr` 置 `STARTF_USESTDHANDLES`，`:503-506`，因此不影响输出捕获）。

**建议**
第 10 参数改传 `nullptr`；`TerminateProc` 前加注释说明它是"清理孙进程"，并把它和超时逻辑一起实现。

**验证方式**
改动后跑一次 Interop/Game 编译确认输出仍完整（grep 生成的失败日志或看通知文本流）。

---

#### [F-CMP-031] 说明：`AsyncTask(GameThread)` + `WaitUntilTaskCompletes` 与 FEditorListener 的自旋等待（经核对不是缺陷，但值得记录）

- **类别**: 并发/线程安全（**核对结论：无问题**）
- **严重度**: **撤销（非缺陷）**（原定 P3）
- **复核结论**: 确认（该条自述的"经核对不是缺陷"**成立**：`AsyncTask` 已正确 marshal 回游戏线程、`WaitUntilTaskCompletes` 不会自锁两点均可复核）
- **可达性**: 活跃（`FEditorListener::WaitForCompile` 的自旋是唯一的进度反馈回路，每次文件监听编译都会跑到；但它**不是缺陷**）
- **复核证据**: `Source/UnrealCSharpEditor/Private/Listener/FEditorListener.cpp:712-745` **已读**：`:718 while (FCSharpCompiler::Get().IsCompiling())`、`:722-736` 以 `constexpr IntervalSecond = 1.0/60.0`（`:716`）节流刷新 UI、**`:738 FPlatformProcess::SleepNoStats(0.0005f)`**、`:740`/`:742`/`:744` 三个 Tick 调用；`FCSharpCompilerRunnable.cpp:213-242`（`CreateAndDispatchWhenReady(..., ENamedThreads::GameThread)` + `WaitUntilTaskCompletes`）、`:325-382`、`:419-434` 已读；引擎侧 `TaskGraph` 行为未回源码核对
- **级别变动**: P3→**撤销（非缺陷）**
- **口径说明（重要，勿被下游当成待修项）**: **本条不计入待修缺陷**。理由：正文自述"经核对不是缺陷"，其主体（两个"无问题"结论）是对**已核对无问题项**的记录，不是缺陷；唯一的可操作建议（自旋间隔 `0.0005f` → 5–10ms）是独立 P3 优化点，已单独列在下面的"建议"中。
  **编号保留**：编号是跨报告引用锚点，**不删除、不重排**。下游统计口径：`F-CMP-031` 计入"发现总数 33"但**不计入 P0–P3 待修数**（级别记 `撤销（非缺陷）`）。
- **文件**: `Source/Compiler/Private/FCSharpCompilerRunnable.cpp:213-242`、`:325-382`、`:419-434`；`Source/UnrealCSharpEditor/Private/Listener/FEditorListener.cpp:718-745`
- **函数**: `FCSharpCompilerRunnable::Compile()`、`FEditorListener::WaitForCompile()`
- **置信度**: 高（引擎源码逐段核对）

**现状（代码事实）**
```cpp
// FCSharpCompilerRunnable.cpp:213
	const auto Task = FFunctionGraphTask::CreateAndDispatchWhenReady(..., ENamedThreads::GameThread);
	FTaskGraphInterface::Get().WaitUntilTaskCompletes(Task);
```
```cpp
// FEditorListener.cpp:718
	while (FCSharpCompiler::Get().IsCompiling())
	{
		FThreadHeartBeat::Get().HeartBeat();
		...
		FPlatformProcess::SleepNoStats(0.0005f);
		FTSTicker::GetCoreTicker().Tick(FApp::GetDeltaTime());
		FThreadManager::Get().Tick();
		FTaskGraphInterface::Get().ProcessThreadUntilIdle(ENamedThreads::GameThread);
	}
```

**核对结论**
（1）`AsyncTask` 的 UI 更新**已经** marshal 回游戏线程（`:325`、`:419`、`:441`），不存在"后台线程直接碰 Slate"的问题——这是该文件做得对的地方。（2）从游戏线程 `WaitUntilTaskCompletes` 一个游戏线程任务**不会自锁**：`TaskGraph.cpp:1447-1460` 先 `TryRetractAndExecute` 就地执行，必要时 `WaitOnNamedThreadForTasks`（`:1480-1487`）会派发 `FReturnGraphTask` 并自泵队列。（3）`WaitForCompile` 的自旋（`SleepNoStats(0.0005f)` + 主动 `Tick`/`ProcessThreadUntilIdle`）是为了让模态窗口有响应，属有意设计；但它以 ~2000 次/秒的频率空转并每 1/60 秒更新一次 UI（`:722-736`），在长时间编译时会持续占用一个核心的相当比例，且这是**唯一的进度反馈回路**（没有取消按钮）。

**建议**
把自旋间隔从 0.5ms 放宽到 5-10ms（`SleepNoStats(0.005f)`），对 UI 响应无可见影响而显著降低空转；并借机补一个"取消编译"按钮（配合 F-CMP-001 的取消机制）。

**验证方式**
在长编译期间用任务管理器观察编辑器进程 CPU 占用；或统计 `IsCompiling()` 循环次数。

---

#### [F-CMP-032] `UE_VERSION_START` 与引擎 `UE_VERSION_NEWER_THAN_OR_EQUAL` 完全重复，且命名与"严格大于"宏形近

- **类别**: 可读性/一致性
- **严重度**: P3
- **文件**: `Source/CrossVersion/Public/UEVersion.h:5-6`（对比引擎 `EngineVersionComparison.h:11-15`）
- **函数**: 宏（无函数）
- **置信度**: 高（两侧源码均已读）
- **复核结论**: 确认（引擎侧已**回源码逐字比对：两宏体完全等值**；与 `UE_VERSION_NEWER_THAN` 只差 tie-breaker 一个布尔）
- **可达性**: 活跃（`UE_VERSION_START` 被 `UEVersion.h` 内其余 100 个宏使用，编译期必然求值）
- **复核证据**: 引擎侧 `Engine\Source\Runtime\Core\Public\Misc\EngineVersionComparison.h` **25 行已读**：`:8-9 UE_GREATER_SORT`、`:14-15 UE_VERSION_NEWER_THAN_OR_EQUAL`（最内层 `true`）、`:19-20 UE_VERSION_NEWER_THAN`（最内层 `false`，严格大于）、`:24-25 UE_VERSION_OLDER_THAN` → 报告引用的 `:11-15`/`:19-20` 与"完全重复 + 形近易混"的判定**成立**；插件侧 `Source/CrossVersion/Public/UEVersion.h:5-6` 未逐行读
- **级别变动**: 无

**现状（代码事实）**
```cpp
// UEVersion.h:5
#define UE_VERSION_START(MajorVersion, MinorVersion, PatchVersion) \
	UE_GREATER_SORT(ENGINE_MAJOR_VERSION, MajorVersion, UE_GREATER_SORT(ENGINE_MINOR_VERSION, MinorVersion, UE_GREATER_SORT(ENGINE_PATCH_VERSION, PatchVersion, true)))
```
```cpp
// <Engine>/Source/Runtime/Core/Public/Misc/EngineVersionComparison.h:14
#define UE_VERSION_NEWER_THAN_OR_EQUAL(MajorVersion, MinorVersion, PatchVersion)\
	UE_GREATER_SORT(ENGINE_MAJOR_VERSION, MajorVersion, UE_GREATER_SORT(ENGINE_MINOR_VERSION, MinorVersion, UE_GREATER_SORT(ENGINE_PATCH_VERSION, PatchVersion, true)))
```

**调用上下文**
`UE_VERSION_START` 在 `UEVersion.h` 内部被 100 个宏使用（全库范围内不出现于其他文件）；`UE_VERSION_NEWER_THAN_OR_EQUAL` 的引入版本我无法离线确认（本机只有 5.6），因此插件自带定义可能是为了兼容更老的引擎——这属于合理的权衡。

**问题**
（1）同一语义两套名字，读者需要额外验证"是否真的等值"（我已逐字比对，**等值**，且边界行为正确，无 off-by-one）。（2）名字 `..._START` 与引擎的 `..._NEWER_THAN`（严格大于，tie-breaker `false`，`EngineVersionComparison.h:19-20`）形近，容易被误当成严格大于使用；一旦有人按"NEWER_THAN 语义"改用 `UE_VERSION_START(5, 6, 0)` 表示"5.6 之后"，就会在 5.6.0 上得到错误结果。

**建议**
若最低支持版本 ≥ 引入 `UE_VERSION_NEWER_THAN_OR_EQUAL` 的引擎版本，直接改用引擎宏；否则保留但在 `UEVersion.h` 顶部加注释："`UE_VERSION_START(a,b,c)` == `引擎版本 >= a.b.c`（与引擎 `UE_VERSION_NEWER_THAN_OR_EQUAL` 等价），**不是** `UE_VERSION_NEWER_THAN`"。

**验证方式**
`grep -rn "UE_VERSION_START" Source/ | wc -l`（应为 101，全部在 `UEVersion.h` 内）；在 `UEVersion.h` 处静态断言 `UE_VERSION_START(5,6,0) == UE_VERSION_NEWER_THAN_OR_EQUAL(5,6,0)` 于 5.6 下成立。

---

#### [F-CMP-033] 空壳模块与 `.uplugin` 中 `SourceCodeGenerator` 的模块类型声明不一致

- **类别**: 死代码/一致性（说明性）
- **严重度**: P3
- **文件**: `UnrealCSharp.uplugin:49-53`；`Source/Compiler/Private/Compiler.cpp:7-16`；`Source/CrossVersion/Private/CrossVersion.cpp:7-16`
- **函数**: `FCompilerModule::StartupModule/ShutdownModule`、`FCrossVersionModule::StartupModule/ShutdownModule`
- **复核结论**: 部分确认（偏差：只属文档/一致性层面的问题，"UBT 如何处置插件内 `Type: Program` 模块"**未验证** → 该子项记 `无法验证（卡点：需实机构建观察 UBT 是否报错/忽略）`；维持 P3）
- **可达性**: 活跃（`.uplugin` 声明与两个空模块壳一直存在并参与构建）
- **复核证据**: `UnrealCSharp.uplugin` **61 行全文已读**：`:17 "CanBeUsedWithUnrealHeaderTool": true`、`:49-53` 的 `{"Name": "SourceCodeGenerator", "Type": "Program", "LoadingPhase": "PostConfigInit"}`、`:35-38` `Compiler` 为 `Type: Editor`、`:45-48` `CrossVersion` 为 `Type: Runtime`；`Source/Compiler/Compiler.Build.cs` **存在且 58 行全文已读**（可作为"Compiler 是真模块"的对照）；`Source/SourceCodeGenerator/` 目录内容未逐个 `read`（以报告 `Get-ChildItem` 结论为据）
- **级别变动**: 无
- **置信度**: 中（`SourceCodeGenerator` 目录内无 `.Build.cs`/`.Target.cs` 是确定事实；UBT 对"插件内 Program 模块"的处理细节未在本机验证）

**现状（代码事实）**
```json
// UnrealCSharp.uplugin:49
		{
			"Name": "SourceCodeGenerator",
			"Type": "Program",
			"LoadingPhase": "PostConfigInit"
		}
```
该目录实际内容（`Get-ChildItem Source/SourceCodeGenerator`）：`SourceCodeGenerator.cs`、`SourceCodeGenerator.ubtplugin.csproj`、`SourceCodeGenerator.ubtplugin.csproj.props`、`obj/`、`.gitignore` —— **没有 `SourceCodeGenerator.Build.cs`，也没有 `SourceCodeGenerator.Target.cs`**。真实接线是 `.uplugin:17 "CanBeUsedWithUnrealHeaderTool": true` + `SourceCodeGenerator.cs:18 [UnrealHeaderTool]`。

**调用上下文**
`Compiler`/`CrossVersion` 两个模块的 `StartupModule`/`ShutdownModule` 均为空实现（`Compiler.cpp:7-16`、`CrossVersion.cpp:7-16`），模块的存在意义仅是承载编译单元（见 §1）。

**问题**
（1）`Type: "Program"` 对一个插件内模块而言是误导性的：它既不生成可执行程序，也没有 `Build.cs`；真正被构建的是 `Binaries/DotNET/UnrealBuildTool/Plugins/SourceCodeGenerator/`（本工作区实测存在，含 `EpicGames.UHT.dll` 等）。后续维护者按 `.uplugin` 的声明去找 Program 的 `Target.cs` 会一无所获。（2）`Compiler` 模块的空 `StartupModule` 与 F-CMP-011 相关：模块本可以承担单例的显式构造/销毁职责，却没有。

**建议**
把 `.uplugin` 中的 `SourceCodeGenerator` 条目改为注释说明（或移除，仅保留 `CanBeUsedWithUnrealHeaderTool`），并在 `Compiler.cpp`/`CrossVersion.cpp` 的空函数里写明"本模块无启动逻辑"；同时按 F-CMP-011 在 `StartupModule/ShutdownModule` 中接管 `FCSharpCompiler` 生命周期。

**验证方式**
`Test-Path Source/SourceCodeGenerator/SourceCodeGenerator.Build.cs`（False）；`Get-ChildItem Binaries/DotNET/UnrealBuildTool/Plugins/SourceCodeGenerator`（存在）。

---

## 4. 死代码清单

**宏的计数方法**：用 `Select-String` 对插件全部 1422 个文本文件（`.h/.cpp/.inl/.cs/.template/.csproj/.props/.targets/.json/.uplugin/.uproject/.txt/.py/.bat/.sh/.md`，排除 `obj/`、`bin/`、`Binaries/`、`Intermediate/`、`.git/`）匹配 `\b宏名\b`，并剔除定义所在文件自身的命中。**两处方法论修正（都实际发生过，记录在此以免读者误信中间结论）**：
1. 第一次统计时我漏掉了 `.inl` 文件（插件大量宏在 `.inl` 中被使用），修正后才得到下表——这也是为什么不能只按"命中数 1"下结论。
2. 脚本化计数用的是 PowerShell `Select-String`，**默认大小写不敏感**，因此 DTO 类短宏名会被 `DisplayName="dotnet8"` 之类的小写字符串污染成假阳性。下表相关行已用逐条 `read`/`grep` 修正；`UE_*` 长宏名不受影响（并已对被误列为"疑似死宏"的 11 个 `UE_T_BASE_STRUCTURE_*`/`UE_STATIC_*` 等逐条复验，全部在 `.inl` 中有真实使用点）。

| 符号 | 声明位置 | grep 命中数（全库 / 定义文件外） | 判定 | 证据 |
|---|---|---|---|---|
| `VS2022` | `Source/CrossVersion/Public/VSVersion.h:4`、`:8` | 2 / **0** | **死宏** | `grep "VS2026\|VS2022"`: 12 命中，其中 VS2022 仅 `VSVersion.h:4`、`:8` 两条 `#define`；唯一使用是 `FSolutionGenerator.cpp:445 #if !VS2026` |
| `STD_CPP_11` | `CppVersion.h:3` | 1 / **0** | **死宏** | 全库仅该 `#define` |
| `STD_CPP_14` | `CppVersion.h:5` | 1 / **0** | **死宏** | 同上 |
| `STD_CPP_17` | `CppVersion.h:7` | 1 / **0** | **死宏** | 同上 |
| `STD_CPP_20` | `CppVersion.h:9` | 4 / 3 | 在用 | `FBindingClassRegister.cpp:115`、`FSetHelper.cpp:104`、`FMapHelper.cpp:202` |
| `DOTNET10` | `DotnetVersion.h:13` | 1 / **0** | **死宏** | 全库唯一命中是它自己的 `#define`。**注意**：我脚本化计数把它算成"有 1 处使用"，原因是 PowerShell `Select-String` 默认大小写不敏感，把 `UnrealCSharpSetting.h:70` 的 `V10 = 10 UMETA(DisplayName="dotnet10")` 当成了使用点——确认是假阳性（`FMonoDomain.cpp` 全程未用 `DOTNET10`） |
| `DOTNET8` | `DotnetVersion.h:9` | 1 / **1（不可达）** | **不可达宏** | 唯一真实使用 `FMonoDomain.cpp:71`（`#if DOTNET8`），但它被 `:62 #if !DOTNET9` 包裹，而 `DOTNET9`（`DOTNET_MAJOR_VERSION=10`，`Mono.Build.cs:13`）恒真 → 见 `F-CMP-016`。（`UnrealCSharpSetting.h:68` 的 `DisplayName="dotnet8"` 同样是小写假阳性） |
| `DOTNET9` | `DotnetVersion.h:11` | 2 / 2 | 在用（其中 1 处为死分支） | `FMonoDomain.cpp:78 #if DOTNET9` **在用**；`:62 #if !DOTNET9` 恒假（死分支） |
| `UE_VERSION_START` | `UEVersion.h:5` | 101 / **0（文件外）** | **文件内 helper，非死宏** | 100 次使用全部在 `UEVersion.h` 自身（101 个宏里的其余 100 个） |
| `DOTNET_VERSION_START` | `DotnetVersion.h:6` | 4 / 0（文件外） | **文件内 helper，非死宏** | 被 `DOTNET8/9/10`（`:9-13`）使用 |
| `DOTNET_GREATER_SORT` | `DotnetVersion.h:3` | 2 / 0（文件外） | **文件内 helper，非死宏** | 被 `DOTNET_VERSION_START`（`:7`）使用 |
| `Project`（`SourceCodeGenerator` 私有字段） | `SourceCodeGenerator.cs:57` | 写入 1（`:86`）/ **读取 0** | **死字段** | 873 行全文阅读确认；对照 `F-CMP-026` |

**其余 100 个 `UE_*` 宏全部有定义文件之外的使用点**（脚本化结果 `used=100 / DEAD=1`，唯一的"DEAD"即文件内 helper `UE_VERSION_START`；已对疑似死宏逐条复验，见上方方法论说明第 2 条）。逐类抽查的典型数量：`UE_F_APP_STYLE_GET_BRUSH` 7 处、`UE_F_OPTIONAL_PROPERTY` 28 处、`UE_F_UTF8_STR_PROPERTY` 25 处、`UE_F_ANSI_STR_PROPERTY` 23 处、`UE_F_PROPERTY_CONSTRUCTOR_E_OBJECT_FLAGS` 13 处；`UE_SET_HANDLE_INFORMATION` 2 处（`FUnrealCSharpFunctionLibrary.cpp:1533`、`UEVersion.h:190`）；`UE_STRUCT_UTILS_U_USER_DEFINED_STRUCT` 3 处。

**函数/类层面的死代码结论**

| 符号 | 声明位置 | grep 命中 | 判定 | 证据 |
|---|---|---|---|---|
| `FCompilerModule::StartupModule/ShutdownModule` | `Compiler.cpp:7-16` | 各 1（空体） | 非死代码（`IModuleInterface` 必需） | 无逻辑，见 `F-CMP-033` |
| `FCrossVersionModule::StartupModule/ShutdownModule` | `CrossVersion.cpp:7-16` | 各 1（空体） | 非死代码（必需） | 同上 |
| `FCSharpCompilerRunnable::EnqueueTask()` | `FCSharpCompilerRunnable.cpp:112` | 1 调用 | 在用 | `FCSharpCompiler.cpp:40` |
| `FCSharpCompilerRunnable::ImmediatelyDoWork()` | `:164` | 1 调用 | 在用 | `FCSharpCompiler.cpp:48` |
| `FCSharpCompiler::Compile(const TArray<FFileChangeData>&)` | `FCSharpCompiler.cpp:52` | 1 调用 | 在用 | `FEditorListener.cpp:642` |
| `FCSharpCompiler::GetCompileProgress()` | `FCSharpCompiler.cpp:76` | 1 调用 | 在用 | `FEditorListener.cpp:729` |
| `FCSharpCompileProgress::StageRestoring/Publishing/...` | `FCSharpCompileProgress.cpp:5-19` | 均有读取点 | 在用 | `ParseLine:101-133`、`FCSharpCompilerRunnable.cpp:183/277/323/401` |
| `SourceCodeGenerator`（整个 UHT 导出器） | `SourceCodeGenerator.cs:19` | 见下 | **在用（非死代码）** | `.uplugin:17 CanBeUsedWithUnrealHeaderTool=true`；本工作区 `Binaries/DotNET/UnrealBuildTool/Plugins/SourceCodeGenerator/` 已构建；`Intermediate/Build/Win64/UnrealEditor/Inc/UnrealCSharp/UHT/` 下 **631 个** `*.binding.inl`/`*.header.inl` 带其专属头注释（`:67-72`）；`Source/SourceCodeGenerator/obj/` 有还原/编译产物 |
| `FCodeAnalysis`（与 SourceCodeGenerator 的关系） | `Source/ScriptCodeGenerator/Private/FCodeAnalysis.cpp` | 3 处 `SyncProcess` | 在用、**与 SourceCodeGenerator 不重复** | 两者职责不同且不重叠：`FCodeAnalysis`（C++ 侧，`ScriptCodeGenerator` 模块）构建并运行 `Script/CodeAnalysis/CodeAnalysis.exe` 做**已有产出**的分析（`:39-52`、`:54-89`）；`SourceCodeGenerator`（C#/UHT 侧）在**构建期**产出 C++ 绑定源码。二者不共享代码路径 |

---

## 5. 未覆盖/存疑项

### 5.1 横向维度检查结果（逐项给结论）

| 维度 | 结论 |
|---|---|
| **空指针/未检查返回值** | 编译模块内**未发现**空指针解引用：`FCSharpCompiler.cpp` 全程 `Runnable != nullptr` 判断（`:38/46/54/62`）；`FCSharpCompilerRunnable.cpp:83` 对 `Event` 判空。唯一例外：`EnqueueTask` 在 `:125`/`:143` 直接 `Event->Trigger()` 无判空（归入 `F-CMP-011`）。C# 侧 `HeaderPath[...]` 索引器无保护（`F-CMP-019`） |
| **内存与资源泄漏** | 进程/管道/句柄**无泄漏**（见 §2.5 完整配对表）；委托 `AddRaw/Remove` 配对正确；`TSharedPtr<SNotificationItem>` 有 `Reset`。挂起场景下 `FEvent` 会随线程一起"泄漏到进程结束"（§2.5 表末两行） |
| **线程安全** | `bIsCompiling` 用了 `std::atomic` ✓；`CompileProgress` 内部有 `FCriticalSection` ✓；UI 更新已 marshal 回游戏线程 ✓。问题集中在：`FileChanges`/`Tasks` 的无锁修改（`F-CMP-004`）、两个非原子 bool（`F-CMP-012`）、锁内访问 UObject（`F-CMP-010`）、同步编译入口绕过守卫（`F-CMP-003`） |
| **异常与错误处理** | C++ 侧无 `try/catch`（UE 默认关闭异常，故非缺陷），但退出码**被正确识别**（`FCSharpCompilerRunnable.cpp:397`、`SyncProcess:1580`），任务书担心的"非 0 被当成成功"**不成立**；C# 侧有两处未捕获异常路径（`F-CMP-017`、`F-CMP-020`） |
| **性能** | 编译线程 10ms 轮询读取开销可忽略；`Result`/`PendingOutput` 无上限（`F-CMP-023`）；`FEditorListener.cpp:738` 的 0.5ms 自旋（`F-CMP-031` 建议）；`SourceCodeGenerator` 有 1 次完全无用的全目录扫描（`F-CMP-026`）与不确定顺序导致的重复写出（`F-CMP-021`） |
| **死代码** | 4 个死宏 + 1 个死字段（§4）；1 个不可达版本分支（`F-CMP-016`）；`SourceCodeGenerator` **确认在用**（631 个产物 + 已构建的 UBT 插件） |
| **可优化/可读性** | `F-CMP-015`、`F-CMP-024`（`const_cast`+命名误导）、`F-CMP-025`（static 缓存）、`F-CMP-028`、`F-CMP-032`、`F-CMP-033` |
| **平台兼容** | 硬编码 dotnet 路径（`F-CMP-007`）；UTF-8 解码假定与分块截断（`F-CMP-008`）；英文文本依赖（`F-CMP-009`）；`__cplusplus` 依赖 `/Zc:__cplusplus`（`F-CMP-013`）；非 ASCII 路径无 BOM（`F-CMP-022`）。**任务书问到但经核对不成立**的两条：(a) 命令行引号转义——插件对项目路径统一加了 `\"%s\"`（`FCSharpCompilerRunnable.cpp:283`、`:387`；`FUnrealCSharpFunctionLibrary.cpp:281`），且引擎会用 `"\"%s\" %s"` 包裹可执行文件（`WindowsPlatformProcess.cpp:528`），**含空格路径可用**；(b) 编辑器的 Debug/Development/Shipping → `ESolutionConfiguration` **只有 Debug/Release**（`UnrealCSharpEditorSetting.h:7-11`），`GetBuildConfiguration()`（`FCSharpCompilerRunnable.cpp:250-263`）按"是否 Cook"在 `RuntimeConfiguration`/`EditorConfiguration` 间二选一，**不存在配置映射错误**，但也**不支持** Development/Shipping 这种粒度 |
| **进度百分比/除零** | 编译器模块**没有百分比计算**（`FCSharpCompileProgress` 只维护阶段文本），因此"分母为 0"不适用；UI 侧 `SCompileProgressDialog::UpdateProgress(状态文本, 已用秒数)`（`SCompileProgressDialog.cpp:56`、`:71 FText::AsNumber(InElapsedSeconds)`）也没有除法。**未发现除零风险** |
| **Commandlet/CI（无 UI）** | `Compile()` 内的 Slate 通知部分已被 `AsyncTask(GameThread)` 包裹且有 `GExitPurge` 早退（`FCSharpCompilerRunnable.cpp:327`、`:421`、`:443`），`GetBuildConfiguration()` 用 `IsRunningCookCommandlet()` 切配置（`:255`）→ 无 UI 时可运行；但**没有 `IsInGameThread`/无 Slate 模块的兜底**，且 `FEditorListener::WaitForCompile` 依赖 `FSlateApplication::Get().CanDisplayWindows()`（`FEditorListener.cpp:686`）——该路径在无 UI 下会跳过窗口，属正确处理，未发现缺陷 |

### 5.2 明确的未覆盖/存疑

1. **未运行任何构建或编辑器**：所有结论来自静态阅读与文件系统实证（631 个生成物、`Binaries/DotNet` 产物、ini 内容），没有实际执行 `dotnet build`/编辑器。涉及运行时行为的三条已标 `置信度: 中/低`：MSBuild 输出的实际编码（`F-CMP-008`）、`.NET` 跨主版本 roll-forward 行为（`F-CMP-014`）、UHT 对"插件内 Program 模块"的处置（`F-CMP-033`）。
2. **`UhtExportFactory` 源码不在本机**：`<Engine>/Source/Programs/Shared/EpicGames.UHT/Utils/UhtExportFactory.cs` 路径不存在（grep 报 `os error 2`），因此 `Factory.CommitOutput` 的写入编码/是否带 BOM **无法验证**（`F-CMP-022` 因此标为低置信度，仅作为"未验证假设"记录）。
3. **引擎版本只核对了 5.6**：本机引擎为 `Engine`（由 `SourceCodeGenerator.ubtplugin.csproj.props:4` 指明）。所有引擎侧断言（`UE_GREATER_SORT` 语义、`CreatePipe` 默认参数、`ReadPipe` 的 UTF-8 解码、`Kill(true)` 的 `INFINITE` 等待、`/Zc:__cplusplus`、TaskGraph 的 retract 行为）**只对 5.6 成立**。插件声称支持到 5.8（`UEVersion.h:192-206`），5.0–5.5 与 5.7/5.8 的差异（尤其是 `UE_SET_HANDLE_INFORMATION` 为何从 5.7 起才需要）**无法验证**。
4. **`CanExportFunction` 的过滤规则完备性**：8 条规则（`SourceCodeGenerator.cs:212-266`）的"是否漏掉某种不可导出函数"需要与 UHT 类型系统逐项对照才能确认，本次只做了语法/类型层面的静态阅读，未做语义完备性证明。
5. **`GetParamPropertySignature` 的 const/& 规则**（`SourceCodeGenerator.cs:543-628`）：涉及 `PassCppArgsByRef`、`ConstParm`、`RefQualifier`、`ArrayDimensions` 的组合，我未验证所有组合是否与 C++ 侧 `BINDING_OVERLOAD` 展开完全匹配（这需要实际编译产物才能确认）。
6. **`ThirdParty/` 子系统按规范排除**：`CoreCLR/LeanCLR/Mono` 三个模块我仅读取了版本常量与 `DOTNET_*_VERSION` 注入逻辑（为完成 §2.6 交叉核对），其运行时加载/`hostfxr` 细节未分析。
7. **`SourceCodeGenerator` 当前在本工作区是否被触发**：`Config/DefaultUnrealCSharpEditorSetting.ini` **不存在**（只有 `Saved/Temp/{Android,Linux,Win64}/...` 的历史副本，其中 `bEnableExport=True`），而 `Intermediate` 下的生成物 mtime 为 2026-05-18、`Script/` 为 2026-07-20 —— 我可以确定"导出器确实运行过（有 631 个带其专属头注释的产物 + 已构建的 UBT 插件）"，但**无法确定"当前这次构建是否仍会运行"**（取决于构建时 `Config/` 下是否存在该 ini）。这一不确定性已在 `F-CMP-018` 中明示。
8. **`Project` 字段判死的边界**：`grep` 对 `Project` 这类常见词会有大量无关命中（`ProjectDirectory`/`ProjectFile`），我的判定依据是**873 行全文阅读**（无其他读取点），而不仅是 grep 计数，特此说明以避免误导。

### 5.3 复核补记（逐条源码级复核的偏差与仍未闭合项）

**已在正文就地修正的偏差（8 处，详见各条 `复核证据`）**：

| # | 位置 | 原文 | 修正后 |
|---|---|---|---|
| 1 | `§2.5` 第 92 行 / `F-CMP-002` / `F-CMP-030` | "`SyncProcess` … **5 处调用**" | **6 处**（`FCSharpCompilerRunnable.cpp:291`/`:413`、`FCodeAnalysis.cpp:34/49/86`、`UnrealCSharpEditorSetting.cpp:152`） |
| 2 | `F-CMP-003` 文件行 / `F-CMP-006` 调用上下文 | 资源变更回调 `FEditorListener.cpp:388` | **`UnrealCSharpEditor.cpp:388`**（grep 实测；`FEditorListener.cpp` 无 388 行的该调用） |
| 3 | `F-CMP-019` | 产物绝对路径 `#include` 在 Actor.binding.inl "**第 8-10 行**" | **第 10-12 行**（已 `read` 产物确认） |
| 4 | `F-CMP-020` | 判空点 "`:**848**`/`:**871**` 的 HashSet 重载" | **`:832`/`:842`（Dictionary）、`:865`（HashSet 内层）**；`848`/`871` 实为右花括号 |
| 5 | `F-CMP-017` 调用上下文 | `SaveIfChanged` 由 `:348` 与 `:387` "**并发**调用" | 两者**不并发**（`:168 Task.WaitAll` 之后才 `:171 Finish()`）；并发的是多个 `Factory.CreateTask`（`:185`） |
| 6 | `F-CMP-003` 复核证据 | "只有监听器在防重入" | 补充确证：grep `IsCompiling()` 还命中 `FCSharpCompilerRunnable.cpp:364`（ticker 内自用，非守卫） |
| 7 | `§3.0` 第 215 行 | `Compiler.Build.cs` 依赖的交叉引用写 `F-CMP-030` | **`F-CMP-028`** |
| 8 | 报告题头 | "复核 5 条" | 口径（33 条全复核 + 分布统计） |

**仍未闭合（保留为存疑，不影响任何结论方向）**：

1. **产物数量 631 vs 629**：本机 `glob Plugins/UnrealCSharp/Intermediate/Build/Win64/**/UHT/*.binding.inl` = **629**，报告写 631（差 2，未定位；不影响"导出器确实运行过"的判定）。
2. **`F-CMP-029` 的 `FSolutionGenerator.cpp:443-457` 函数体**未逐行读（只 grep 到 `:445` 的使用点）；该条结论依赖"`#` 是 `.sln` 合法注释前缀"这一常识性判断。
3. **`F-CMP-031` 引用的引擎 `TaskGraph.cpp:1447-1460`/`:1480-1487`** 未回源码逐行核对（该条已裁定为**非缺陷**，故此核对不影响待修项）。
4. **`F-CMP-013` 的引擎新证据已改写其前提**：`Engine\Source\Programs\UnrealBuildTool\Platform\Windows\UEBuildWindows.cs:574 public bool bUpdatedCPPMacro = true;` ——默认开启，故该条已由 P2 下调 P3、可达性改为**潜伏**。


