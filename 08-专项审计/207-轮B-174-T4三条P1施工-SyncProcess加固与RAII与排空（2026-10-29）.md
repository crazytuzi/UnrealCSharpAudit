# 轮 B-174：T4 三条 P1 施工 —— `SyncProcess` 加固、依赖图 RAII/`Reset`、同步上下文排空（2026-10-29）

> **来源**：B-171 [`08-…/204`](204-轮B-171-T4-1-F-FL-009设计裁定-SyncProcess进程模型（2026-10-29）.md) §5、B-172 [`08-…/205`](205-轮B-172-T4-2-F-LIFE-016设计裁定-依赖图生命周期（2026-10-29）.md) §5、B-173 [`08-…/206`](206-轮B-173-T4-3-F-CS2B-011设计裁定-同步上下文取消语义（2026-10-29）.md) §5 的 **P1** 项。
> **性质**：插件 **8 文件 `+85/−24`**（C++ 7 ＋ C# 1；含 2 个公开头文件的方法改名 `Empty → Reset`）；"托管边界"异常隔离用 **E1**（`_CONVENTIONS` §8.5）。
> **一句话结论**：三条 P1 全部落地 —— ① `SyncProcess` 空回调守卫、终止点单一化（仅取消时）、取消原因写既有 `Result`、`FCodeAnalysis` 三处传 `GIsRequestingExit`；② 依赖图 `Empty()` **改名 `Reset()`**（内部 `NodeArray/NodeMap.Reset()`，保容量）＋ `Generator()` 用 `ON_SCOPE_EXIT` 保证必达；③ 同步上下文 `Deinitialize()` **内联快照排空**（逐任务 E1 隔离，只取 `TaskList`，不碰 `PendingTaskList`）。**探针：`rc=-2/0.27 s/Result=…Cancelled (stopped).`、空回调不崩；`post_count=1`、`Send` 由阻塞转解锁；5.6/5.0 UBT 与 C# 均 0/0；§7 CLEAN / §8 PASS；套件 1 987/0 同 SHA ×2。**

---

## 1. 施工

### 1.1 `F-FL-009` P1（`FUnrealCSharpFunctionLibrary.cpp` `+26/−5`、`FCodeAnalysis.cpp` `+16/−6`）

```diff
-		InOnComplete(-1, Result);
+		if (InOnComplete)
+		{
+			InOnComplete(-1, Result);
+		}
@@
 	auto bCancelled = false;
 
+	const TCHAR* CancelReason{};
+
@@
 		if (bIsStopped || bIsTimeout || bIsSilenceTimeout)
 		{
 			bCancelled = true;
 
-			FPlatformProcess::TerminateProc(ProcessHandle, true);
+			if (bIsStopped) { CancelReason = TEXT("stopped"); }
+			else if (bIsTimeout) { CancelReason = TEXT("timeout"); }
+			else { CancelReason = TEXT("silence"); }
 
 			break;
@@
 	if (bCancelled)
 	{
+		FPlatformProcess::TerminateProc(ProcessHandle, true);
+
+		Result.Append(FString::Printf(TEXT("\nCancelled (%s)."), CancelReason));
+
 		ReturnCode = -2;
 	}
 	else if (!FPlatformProcess::GetProcReturnCode(ProcessHandle, &ReturnCode))
@@
-	InOnComplete(ReturnCode, Result);
+	if (InOnComplete)
+	{
+		InOnComplete(ReturnCode, Result);
+	}
 
 	FPlatformProcess::ClosePipe(ReadPipe, WritePipe);
 
-	FPlatformProcess::TerminateProc(ProcessHandle, true);
-
 	FPlatformProcess::CloseProc(ProcessHandle);
```

- 终止点从"循环内 ＋ 循环后无条件"收敛为**取消时单一出口**；正常退出不再 `TerminateProc`（G2 消除）。
- 取消原因（stopped/timeout/silence）写入**既有 `Result`**（不新增日志，Q2）；`-2` 语义不变。
- `FCodeAnalysis.cpp` 三处调用追加 `FString(), {}, []{ return GIsRequestingExit; }`（按仓库排版对齐续行）。

### 1.2 `F-LIFE-016` P1（`FDynamicDependencyGraph.cpp` `+3/−3`、`FDynamicGenerator.cpp` `+10/−6`、`FDynamicGeneratorCore.cpp` `+2/−2`、两个公开头 `+1/−1`）

```diff
-void FDynamicDependencyGraph::Empty()
+void FDynamicDependencyGraph::Reset()
 {
-	NodeArray.Empty();
+	NodeArray.Reset();
 
-	NodeMap.Empty();
+	NodeMap.Reset();
 }
```
```diff
+	ON_SCOPE_EXIT
+	{
+		FDynamicGeneratorCore::Reset();
+
+#if WITH_EDITOR
+		bIsFullGenerator = false;
+#endif
+	};
+
 	FScriptDomainScope([](){ … });
-	FDynamicGeneratorCore::Empty();
-	…
```

- **公开方法改名 `Empty → Reset`**（`FDynamicDependencyGraph.h`、`FDynamicGeneratorCore.h` 及 3 处实现/调用）：名字与"保留容量、只清元素"的语义对齐；这是本轮唯一的公开面变更，由维护者决定。
- `Reset()` 只清元素、保留容量 ⇒ 热重载每轮不再重分配 N 个 `FNode`（G3）。
- `ON_SCOPE_EXIT` 保证 `Reset()`/`bIsFullGenerator` 在任何出口都复位（G1）；不新增断言/日志（G7）。

### 1.3 `F-CS2B-011` P1（`Script/UE/CoreUObject/SynchronizationContext.cs`，`+26/−0`）

```diff
 public static void Deinitialize()
 {
+    var OldContext = Context;
+
     SetSynchronizationContext(null);
 
+    if (OldContext != null)
+    {
+        List<TaskInfo> Tasks;
+
+        lock (OldContext.TaskList)
+        {
+            Tasks = new List<TaskInfo>(OldContext.TaskList);
+
+            OldContext.TaskList.Clear();
+        }
+
+        foreach (var Task in Tasks)
+        {
+            try
+            {
+                Task.Invoke();
+            }
+            catch (Exception)
+            {
+            }
+        }
+    }
+
     Context = null;
 }
 ```
- 排空顺序：先摘全局上下文 → 内联快照旧实例的 `TaskList` → 清引用（排空期间新 `Post` 不再落入旧实例）。
- **不碰 `PendingTaskList`**：它是正在执行的 `Process()` 所持有的批次，必然会被外层循环执行；清掉它反而会在重入时破坏 `Process()` 的 `foreach`。
- 逐任务 `try/catch` 属**托管边界 E1**：一个坏续体不阻断域销毁；`TaskInfo.Invoke` 的 `finally` 仍会 `Set()`，因此阻塞中的 `Send` 同时被解锁。
- 无 `[UnmanagedCallersOnly]` 签名变化；未新增命名（`Drain` 已按方案 A 删除）。

---

## 2. 探针

| 条目 | 装置 | 读数 |
|---|---|---|
| `SyncProcess` 取消 | 临时 `UnrealCSharp.ProbeB174`：`cmd /c ping -n 8` ＋ `InIsStopped` 0.2 s 后为真 | **`rc=-2`、elapsed 0.27 s、`Result` 末尾 `Cancelled (stopped).`**（`probe-sync.log`） |
| `SyncProcess` 空回调 | 同探针第二次调用，传默认 `InOnComplete` | 不崩、`empty_callback_ok=1` |
| `F-CS2B-011` 排空 | 独立 `net10.0` 控制台引用 `Content/Script/{UE,Interop}.dll`：`Initialize → Post(写文件) → Deinitialize` | **`post_count=1`、`post_marker=True`** |
| `F-CS2B-011` `Send` | 同上：工作线程 `Context.Send(...)` 阻塞后主线程 `Deinitialize` | **`send_blocked_before=True`、`send_unblocked_after=True`、`send_count=1`** |
| `F-LIFE-016` | 代码形态（`ON_SCOPE_EXIT` 必然执行）＋ 套件回归；未单独造早退 | 说明：早退探针需临时改控制流，出于"不改交付逻辑"未做（登记为残余） |

> 说明：`SyncProcess` 探针取自"语义等价"的中间版本（终版只差 `CancelReason{}` 初始化与 `Empty→Reset` 改名）；`F-CS2B-011` 探针在方案 A 定稿后重跑，读数与表中一致。

---

## 3. 跨版本与规范

| 轴 | 读数 |
|---|---|
| UE 5.6 实编 | **Succeeded**（110.21 s，`Saved/Retest/b174/ubt-final.log`；改名触发多模块重编） |
| UE 5.0 实编（`UE50Check`） | **Succeeded**（205.54 s、13/13 action、**0 error**） |
| C#（`Script/Script.sln` Release） | **0 warning / 0 error**（50.81 s） |
| §7 语法 lint | **CLEAN** |
| §8 风格 | **PASS**：8 文件，Added 26/16/26/3/10/2/1/1、AddedMax **74**、C# 4 空格、0 违规（`FCodeAnalysis.cpp` 15 行续行对齐为 INFO） |
| 套件 | `B174d-suite-deliver` ×2：**1 987 行 / 0 fail / `D520F5AD71AE6B1D9534DA27FAF22FB1C7AEAC110C025F30FFB1B77BFB0C54C6`** |

---

## 4. 台账

| 台账 | 动作 |
|---|---|
| 插件 | **8 文件 `+85/−24`**；🟢 **已提交 `4564008d`**；插件树仅剩既有 `M Source/SourceCodeGenerator` |
| `01` | 三条行内追加「B-174 P1 已施工（`4564008d`）」；**级别均维持 P2**（FL-009 的 GT 外移、LIFE-016 的名字捕获、CS2B-011 的 token 仍待做） |
| `05` | 新增 **§115** |
| `07` | §5 T4-1/2/3 标「P1 已施工（B-174）；P2 待做」；§7 B-174 完成 |
| `00` / `README` / `INDEX` | 追加轮 B-174 段落 |

> **终版相对中间版的差异**：① 删除重复的 `#include "Misc/CoreMisc.h"`；② 两个 `if (bCancelled)` 合并为单块；③ 公开方法 `Empty → Reset` 改名（含 2 个头文件）；④ C# 去掉 `Drain()`、改为 `Deinitialize` 内联快照（只取 `TaskList`），循环变量保持 `Task`；⑤ `CancelReason{}` 聚合初始化。

---

## 5. 下一步

1. **B-175+ = T4 剩余三条设计**：`F-ARCH-009`（编译队列静默丢弃 ＋ `while(true)` 轮询）、`F-DEL-010`（`FOptionalHelper::Get()` 所有权）、`F-CS1-002`（强句柄残余）。
2. 三条 P2 硬化（FL-009 GT 外移、LIFE-016 名字捕获、CS2B-011 token）等待维护者排期；其中 GT 外移会影响编辑器交互，建议单独一轮。
