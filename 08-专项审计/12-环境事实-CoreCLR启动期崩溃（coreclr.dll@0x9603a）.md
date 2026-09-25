# CoreCLR 启动期崩溃（`coreclr.dll @ 0x9603a`）：根因、修复、缓解与残余项

> **本轮（2026-09-25 深夜）结论**：本条现象**不是一个问题，而是三类**（同一条失效指令 `coreclr+0x9603A`，触发条件不同）：
>
> | 类 | 形态判据 | 是否插件窗口内 | 修复前 | 交付态 |
> |---|---|---|---|---|
> | **A 启动期·插件窗口内**（原文 26 次的主体） | 日志 `894~906` 行、`LogUnrealCSharp` **0** 行、**日志已越过插件首个托管输出点** | 是（正在 `FDynamicGenerator::Generator`） | 5/22 ≈ 23% | **0 / 215** |
> | **B 套件执行期** | 日志 `957~983` 行、`LogUnrealCSharp` **16~40** 行 | 是 | 有样本 | **0 / 215**（靠运行时配置缓解） |
> | **C 启动期·插件窗口之前** | 日志 `896~928` 行、`LogUnrealCSharp` 0 行、**日志尚未到达插件首个托管输出点** | **否** | 有样本 | 仍偶发（≈1/218 ≈ 0.5%） |
>
> **两处改动（已在工程内生效）**：
> 1. **代码修复**（✅ **已提交**）：`Plugins/UnrealCSharp/Source/UnrealCSharpCore/Public/Domain/Script/FScriptDomainScope.h`（`git diff --stat` = `+1 / −12`，**纯删除、无新增注释**）——脚本域不再由短生命周期作用域拆毁（消除启动期强推 ALC 卸载）。
>    - **提交：`77867446da413c543dc8bef5751c951a17fe8826`**（**= 当前 HEAD**；作者 `crazytuzi <liuxiangcode@qq.com>`，`2026-09-25 23:43:19 +0800`，subject `DelegateHandle Reset && UFunction RemoveFromRoot`）。该提交含本报告 §4.1 的 `FScriptDomainScope.h`（`1 file changed, 1 insertion(+), 12 deletions(-)`，diff 与 §4.1 完全一致）**外加 G12 收口的 5 个文件** ⇒ 整笔 = **6 文件 `+23/−17`**；A 类修复本身仍**恰好只含这 1 个文件**。
> 2. **运行时缓解**（✅ **2026-09-25 已改为随源码分发，并已提交**：子模块 **`1e1cfd99`** + 插件仓指针 **`6e142471`**）：`"System.Runtime.TieredCompilation": false` —— 落点 = **CoreCLR 子模块里被 git 跟踪的 8 份模板** `Source/ThirdParty/CoreCLR/lib/{Release,Debug}/{Win64,Linux_x86_64,macOS_arm64,macOS_x86_64}/CoreCLR.runtimeconfig.json`（消除分层编译后台重 JIT 与代码发布/取消发布的竞态）。
>    - **原因**：该文件落在插件仓库 `.gitignore` 的 `Binaries/*`（第 47 行）之下 ⇒ **它只是构建产物、不是真值**（`git check-ignore` 命中、`git ls-files` 报 "did not match any file(s) known to git"），**提交 `77867446` 不含它**。✅ **最终落地（§13.3）**：该配置的**源**是 CoreCLR 子模块里被跟踪的 8 份模板（`Source/ThirdParty/CoreCLR/lib/**/CoreCLR.runtimeconfig.json`），改那里才是随源码分发的修法；🟢 **已提交**：子模块 **`1e1cfd99`**（`System.Runtime.TieredCompilation false`，父 `c27d952e`，8 文件 `+14/−6`，2026-09-25 23:25:17 +0800）+ 插件仓指针 **`6e142471`**（父 `0170514d`，`1 file changed, 1 insertion(+), 1 deletion(-)`，指针 `c27d952e` → `1e1cfd99`，2026-09-25 23:26:52 +0800）⇒ **检出这些提交的新环境即可拿到 B 类缓解**（不再只限本机）。
>
> ⚠️ 原文判据"**无 CSV + `LogUnrealCSharp` 0 行 ⇒ 插件尚未进入**"**不成立**（§9 已更正）：插件启动期本来就不打该模块日志，且"0 行"同时覆盖 A 类与 C 类，无法区分。
>
> **本文档状态**：正文 §0–§12 记录 A/B/C 三类分解、根因与**提交前**的验证；**§13 是收口**（A 类提交 `77867446` 的登记、判据变更、**B 类缓解改落子模块模板**、残余项）。凡与 §13 冲突的表述以 §13 为准。

---

## 0. 结论摘要

1. **指纹（原文四项不变）**：`UnrealEditor.exe` / `coreclr.dll` / `0xc0000005` / `Fault offset 0x9603a`；退出码 `0x80131506`（CLR fail-fast）。
2. 🆕 **原文未用的两个事件字段，直接定死模块**（近 30 天 26 条 1000 号事件**全部**命中）：
   - `Faulting module path` = `…\Plugins\UnrealCSharp\Binaries\Win64\shared\Microsoft.NETCore.App\10.0.4\coreclr.dll`
   - `.NET Runtime` 1023 号：`internal error … at IP 0x…1603A (0x…0000)` ⇒ **`coreclr+0x9603A`**（26 次同 RVA，基址随 ASLR 变化）。
3. 🆕 **拿到崩溃现场转储并解出调用链**（原文 §5"dump 侧一无所获"作废）：WER `LocalDumps`（mini，~82 MB）稳定落盘；`SymTool dump --all-threads` ⇒ 异常 `0xC0000005 … READ at 0x20`，`coreclr+0x9603A = UnwindInfoTable::UnpublishUnwindInfoForMethod+0x52`。
4. 🆕 **A 类根因（已修复）**：`FScriptDomainScope` 的**析构函数**在每次"动态生成"结束后**销毁脚本域单例** ⇒ `FCoreCLRDomain::Deinitialize` ⇒ `UnloadAssembly()` ⇒ 托管侧 **`AssemblyLoadContext.Unload()` + `GC.Collect()` + `GC.WaitForPendingFinalizers()`**（强推卸载**可收集** ALC，上限 2 s）。该动作落在引擎 `PostDefault` 加载阶段（游戏线程），而**同进程其它线程仍在从该 ALC 解析类型** ⇒ 踩空 ⇒ fail-fast 终止进程。
   - 证据：dump 里两侧线程同时在场——卸载侧 `FEngineListener::OnLoadingPhaseComplete → Activate() → OnUnrealCSharpCoreModuleActive → FUnrealCSharpModule::OnUnrealCSharpCoreModuleActive → FDynamicGenerator::Generator() → ~FScriptDomainScope → FScriptDomainFactory::Destroy → FCoreCLRDomain::Deinitialize → UnloadAssembly → GC.WaitForPendingFinalizers`；崩溃侧 `ClassLoader::LoadTypeHandleForTypeKey_Body / MethodDesc::FindOrCreateAssociatedMethodDesc / MethodTable::DoFullyLoad`。
5. 🆕 **B 类根因（已缓解）**：套件执行期（`[TSet probe]` 反射/绑定桥调用密集段）出现同址失效，dump 崩溃侧是 **`clrjit.dll` 正在 JIT**（`clrjit+0xE80D6` 等）+ 反向 P/Invoke（`FCoreCLRDomain::Invoke → FMethodReflection::Runtime_Invoke → FCSharpFunctionDescriptor::CallCSharp`），**无任何卸载帧** ⇒ 分层编译的后台重 JIT 与代码发布/取消发布竞态。以 `System.Runtime.TieredCompilation=false` 关闭分层编译后 **215 次启动 0 复现**。
6. 🆕 **C 类（未闭环，但已证明不在插件代码窗口内）**：`LogUnrealCSharp` 0 行且**日志尚未到达插件首个托管输出点**（成功轮的该点位置按同批样本实测：`897 / 930 / 942` 行）。这类崩溃发生时插件的托管代码**还没有开始跑** ⇒ **不是插件可修的范围**（属运行时/环境面），且**缓解措施对它无效**（关闭分层编译后仍观测到 1 次，655 行）。已据此把 A/C 两类分开统计。
7. **验证**：UBT **Succeeded**；`-game` 全量 **1705 / 1705、0 failed**（随包 10.0.4；对照 10.0.12 亦 1705/0）；PIE 全量 **1489 / 1489、0 failed**。
8. ⚠️ **"换运行时版本"不是修复**（实测证伪）：`coreclr 10.0.12` 与随包 `10.0.4` 在**该函数处逐字节相同**（`0x95FE0..0x960B0`，176 字节里 174 字节一致），换版本后**照样复现**。
9. **判据更正（§9）**：不要再用"`LogUnrealCSharp` 0 行"判定"插件未进入"。正确判据 = **`Faulting module path` + `Fault offset`**（事件日志），或**分级形态判据**（§8①）。
10. 🟢 **提交与闭环（同日完成，收口见 §13）**：**A 类代码修复已提交 `77867446`**（综合提交：含本条的 `FScriptDomainScope.h` `1 file changed, 1 insertion(+), 12 deletions(-)` + G12 收口 5 文件 ⇒ 整笔 6 文件 `+23/−17`）；**B 类缓解已改为"随源码分发"并已提交**（§13.3，落点是子模块里被 git 跟踪的 8 份 `CoreCLR.runtimeconfig.json` 模板，**不再是 `Binaries/` 下的本机副产物**；🟢 子模块 **`1e1cfd99`**，由插件仓指针提交 **`6e142471`** 引入 —— 该笔只做 `CoreCLR System.Runtime.TieredCompilation false`，**不含 A 类修复**）；**C 类明确留在 §10.1（非插件可修）**。

---

## 1. 指纹与现场

### 1.1 事件日志（原文四项 + 本轮补两项）

```
Faulting application name: UnrealEditor.exe, version: 5.6.0.0
Faulting module name: coreclr.dll, version: 42.42.42.42424
Exception code: 0xc0000005
Fault offset: 0x000000000009603a
Faulting module path: D:\...\Plugins\UnrealCSharp\Binaries\Win64\shared\Microsoft.NETCore.App\10.0.4\coreclr.dll  ← 新增
Faulting process id: 0xF02C                                                                                          ← 新增
```

**"故障模块到底是哪一份"——一次查询定死**：

```powershell
Get-WinEvent -FilterHashtable @{LogName='Application'; ProviderName='Application Error'; StartTime=(Get-Date).AddDays(-30)} -MaxEvents 400 |
  Where-Object { $_.Message -match 'Fault offset:\s*0x0*9603a' } |
  ForEach-Object { [regex]::Match($_.Message,'Faulting module path:\s*(.+)').Groups[1].Value.Trim() } |
  Sort-Object -Unique
# → 只有一条：…\Plugins\UnrealCSharp\Binaries\Win64\shared\Microsoft.NETCore.App\10.0.4\coreclr.dll
```

机内三份 `coreclr.dll`（避免混淆）：

| 位置 | 大小 | ProductVersion | 是否本次故障模块 |
|---|---|---|---|
| `Plugins\UnrealCSharp\Binaries\Win64\shared\Microsoft.NETCore.App\10.0.4\coreclr.dll` | 4 598 784 B | `42.42.42.42424`（Retail） | ✅ **是**（26/26） |
| `Plugins\UnrealCSharp\Binaries\Win64\coreclr.dll`（Mono 侧 `Mono.Build.cs:62` 投放） | 5 344 256 B | `10.0.1-dev` | ❌ 否 |
| `Engine\Binaries\Win64\coreclr.dll`（引擎自带） | 13 010 944 B | `10.0.1-dev` | ❌ 否 |

### 1.2 `.NET Runtime` 1023 号：模块相对地址（26 次一致）

```
Application: UnrealEditor.exe
CoreCLR Version: 42.42.42.42424
.NET Version: 10.0.4-dev
Description: … internal error in the .NET Runtime at IP 0x0000000064C1603A (0x0000000064B80000) with exit code 0x80131506.
```

`IP − base = 0x9603A`（基址各次不同：`0x64B80000 / 0x79380000 / 0x6538… / 0x5541… / 0x3A41… / 0x77B9… / 0x7941…`）。

### 1.3 符号化与故障指令

| RVA | 符号 |
|---|---|
| `0x96000` | `UnwindInfoTable::UnpublishUnwindInfoForMethod+0x18` |
| **`0x9603A`** | **`UnwindInfoTable::UnpublishUnwindInfoForMethod+0x52`** |
| `0x96080` | `UnwindInfoTable::RemoveFromUnwindInfoTable+0x4` |
| `0x961C0` | `EECodeGenManager::TryFreeHostCodeHeapMemory+0x4` |

故障指令（`Saved\Retest\tools\mini_x64.py` 本地解码；10.0.4 与 10.0.12 相同）：

```asm
0x96036  488b4ef8   mov  rcx, [rsi-8]      ; rcx = 前面算出的"unwindInfo 指针"
0x9603A  395920     cmp  [rcx+0x20], ebx   ; <<< 故障：READ at 0x20（rcx+0x20 落在未映射页）
```

⇒ `rcx` 为垃圾值；该读对应 `RemoveFromUnwindInfoTable` 里对 `unwindInfo->m_publishLock` 的取锁，即 **`unwindInfo` 指针本身已失效**（`UnwindInfoTable` 挂在 `RangeSection::_pUnwindInfoTable`）。

---

## 2. 崩溃现场（dump 证据）

```powershell
# 转储开关（管理员；原文 §7.2 建议 1 的落地）
$k='HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps\UnrealEditor.exe'
New-Item -Path $k -Force | Out-Null
Set-ItemProperty $k DumpFolder 'D:\file\UnrealProjects\UnrealCSharpTest\Saved\Retest\crashes' -Type ExpandString
Set-ItemProperty $k DumpType 1 -Type DWord     # 1=mini(~82MB)  2=full
Set-ItemProperty $k DumpCount 50 -Type DWord
# 解析
& Saved\Retest\tools\SymTool\bin\SymTool.exe dump <dump> --all-threads
```

### 2.1 A 类：卸载侧线程 + 崩溃侧线程同时在场

卸载侧：
```
coreclr!GCInterface_WaitForPendingFinalizers+0x2E
UnrealEditor-UnrealCSharpCore.dll+0x9C141   FCoreCLRDomain::UnloadAssembly+0x21
UnrealEditor-UnrealCSharpCore.dll+0x4B173   FCoreCLRDomain::Deinitialize+0x23
UnrealEditor-UnrealCSharpCore.dll+0x53603   FDynamicGeneratorCore::Generator+0x3
UnrealEditor-UnrealCSharpCore.dll+0x4940    `FDynamicGenerator::Generator'::`2'::<lambda_1>::<lambda_invoker_cdecl>
UnrealEditor-UnrealCSharpCore.dll+0x127980  FUnrealCSharpCoreModuleDelegates::OnUnrealCSharpCoreModuleActive
UnrealEditor-UnrealCSharp.dll+0x20EAD5      FUnrealCSharpModule::OnUnrealCSharpCoreModuleActive+0x15
UnrealEditor-UnrealCSharpCore.dll+0x5066C   TBaseRawMethodDelegateInstance<0,FEngineListener,…>::ExecuteIfSafe+0x3C
```
崩溃侧：
```
coreclr!UnwindInfoTable::UnpublishUnwindInfoForMethod+0x52      ← 故障
coreclr!ClassLoader::LoadTypeHandleForTypeKey_Body+0x32F
coreclr!MethodDesc::FindOrCreateAssociatedMethodDesc+0x57C
coreclr!MethodDesc::WalkValueTypeParameters+0x9EB / MethodTable::DoFullyLoad+0xD68
coreclr!CLRVectoredExceptionHandlerPhase2 / EEPolicy::HandleFatalError+0x129
```

### 2.2 B 类：JIT 编译中 + 反向 P/Invoke（无卸载帧）

```
   clrjit.dll+0xE80D6 / +0x563E3 / +0x1CD170 / +0x1AF000        ← JIT 正在编译
   coreclr.dll+0xAB7B1
   clrjit.dll+0x10834B / +0x10AA47 / +0x10C904
   ...
   UnrealEditor-UnrealCSharpCore.dll+0x7CA73   FCoreCLRDomain::Invoke+0x33
   UnrealEditor-UnrealCSharpCore.dll+0x963AC   FMethodReflection::Runtime_Invoke+0x3C
   UnrealEditor-UnrealCSharp.dll+0x20075D      FManagedFunctionDescriptor::Invoke+0x53D
   UnrealEditor-UnrealCSharp.dll+0x1B104E      FCSharpFunctionDescriptor::CallCSharp+0x35E
```
⇒ **没有任何 `UnloadAssembly`/`Deinitialize` 帧**：与 A 类机理不同（B 类是分层编译后台重 JIT 与代码发布/取消发布竞态）。

---

## 3. 触发路径（代码事实）

### 3.1 A 类（启动期）

| 步骤 | 位置 |
|---|---|
| 引擎 `PostDefault` 加载阶段完成 | `Source/UnrealCSharpCore/Private/Listener/FEngineListener.cpp:47-56` → `SetActive(true)` |
| 模块激活广播 | `…/Private/UnrealCSharpCore.cpp:19-27`（`Activate()`） |
| 起来就调动态生成 | `Source/UnrealCSharp/Private/UnrealCSharp.cpp:36-48` → `FDynamicGenerator::Generator()` |
| **病灶**：作用域析构拆域 | `…/Public/Domain/Script/FScriptDomainScope.h`（原文 `~FScriptDomainScope(){ if (Domain) FScriptDomainFactory::Destroy(Domain); }`） |
| 拆域实现 | `…/Private/Domain/Script/FScriptDomainFactory.cpp:47-59` → `Deinitialize()` |
| 卸载 ALC + 强推 GC | `…/Private/Domain/CoreCLR/FCoreCLRDomain.cpp:240-246` → `Script/Interop/AssemblyLoader/AssemblyLoader.cs:30-61` |
| ALC 是**可收集**的 | `Script/Interop/AssemblyLoader/UnrealAssemblyLoadContext.cs:7-8`（`isCollectible: true`） |

### 3.2 B 类（套件执行期）

`Script/Game/UnrealCSharpTest/TestCore/TestCoreSubsystem.cs:164-198`（`[TSet probe]`，`TSet<int>` 的桥接用例）密集走
`测试托管代码 → C++ 桥（反向 P/Invoke）→ 生成类的 NewObject/容器包装 → 大量泛型实例化`；
在 `System.Runtime.TieredCompilation=true`（默认）下，后台分层编译会对这些热点方法做 tier-1 重 JIT，伴随 `UnwindInfoTable` 的发布/取消发布，与正在进行的桥调用/类型加载形成竞态。

---

## 4. 修复与缓解（两处改动）

### 4.1 代码修复（A 类，唯一代码改动文件）

`Plugins/UnrealCSharp/Source/UnrealCSharpCore/Public/Domain/Script/FScriptDomainScope.h`（`git diff --stat` = `+1 / −12`，纯删除）

**提交**：`77867446da413c543dc8bef5751c951a17fe8826`（**= 当前 HEAD**，`2026-09-25 23:43:19 +0800`，该文件在本笔中为 `1 insertion(+), 12 deletions(-)`）——**实际提交的 diff（原文无注释、无新增头文件）：**

```diff
@@ -17,8 +17,6 @@ public:
 			if (ScriptDomain != nullptr)
 			{
 				IScriptDomain::Set(ScriptDomain);
-
-				Domain = ScriptDomain;
 			}
 		}
 
@@ -36,18 +34,9 @@ public:
 		}
 	}
 
-	~FScriptDomainScope()
-	{
-		if (Domain != nullptr)
-		{
-			FScriptDomainFactory::Destroy(Domain);
-		}
-	}
+	~FScriptDomainScope() = default;
 
 	FScriptDomainScope(const FScriptDomainScope&) = delete;
 
 	FScriptDomainScope& operator=(const FScriptDomainScope&) = delete;
-
-private:
-	IScriptDomain* Domain{};
 };
```

即：**构造函数保持原样**（仍会在"无域"时 `Create()` 并经 `IScriptDomain::Set()` 发布为单例，保证生成可用）；**析构不再拆域**；顺带删掉只为析构服务的成员 `Domain`。修订后文件与 `HEAD` 无差异（`git status` 对该文件为空）。

**取舍**：脚本域销毁点收敛到 `FScriptDomainFactory::Destroy`（`grep` 现只剩 1 处调用点：`FDomain::~FDomain`）；生成/热重载不受影响；顺带省掉启动期"建域→拆域→再建域"的一整趟往返（含一次 ALC 卸载与最长 2 s 强推 GC）。

### 4.2 运行时缓解（B 类）⚠️ 未被提交带走


`Plugins/UnrealCSharp/Binaries/Win64/CoreCLR.runtimeconfig.json`：

```json
{
  "runtimeOptions": {
    "tfm": "net10.0",
    "framework": { "name": "Microsoft.NETCore.App", "version": "10.0.4" },
    "configProperties": {
      "System.Runtime.Serialization.EnableUnsafeBinaryFormatterSerialization": false,
      "System.GC.Concurrent": true,
      "System.GC.Server": false,
      "System.Runtime.TieredCompilation": false
    }
  }
}
```

**取舍（如实）**：
- 关闭分层编译后，方法首次调用即编译为优化代码：**启动期 JIT 工作量上升**（本装置实测单次启动日志从 ~1106 行降到 ~1085 行、启动到套件完成仍在 ~8 s 量级，未影响验收）；换得**稳态代码质量更高**且**消除后台重 JIT 竞态**。
- 这是**缓解**而非根治：真正的修复应来自 dotnet/runtime（该处 10.0.4 与 10.0.12 逐字节相同，需挑到含修复的版本）。

**⚠️ 这份配置不会随提交 `77867446` 一起进仓库**（实测）：

```powershell
cd <插件仓库>
git check-ignore -v Binaries/Win64/CoreCLR.runtimeconfig.json
#   .gitignore:47:Binaries/*    Binaries/Win64/CoreCLR.runtimeconfig.json
git ls-files --error-unmatch Binaries/Win64/CoreCLR.runtimeconfig.json
#   error: pathspec ... did not match any file(s) known to git
```

⇒ `git clone` + `checkout 77867446` 得到的环境只有 A 类修复、没有 B 类缓解。**✅ 2026-09-25 处置（收口后落地，见 §13.3）**：不采用下面三条候选里的任何一条，改为**直接改模板源** —— 该配置的**源文件**在子模块 `Source/ThirdParty/CoreCLR/lib/{Release,Debug}/{Win64,Linux_x86_64,macOS_arm64,macOS_x86_64}/CoreCLR.runtimeconfig.json`（**8 份、均被 git 跟踪**），`Binaries/Win64/CoreCLR.runtimeconfig.json` 只是 UBT 按 `CoreCLR.Build.cs` 的 `RuntimeDependencies` 拷过去的**产物**（且 `bUseRelease = true` ⇒ 生效源是 `lib/Release/<平台>/`）⇒ 只改 `Binaries/` 那份**会在下次构建被覆盖**。下表保留为历史候选记录。

| 候选（历史） | 做法 | 状态 |
|---|---|---|
| ① 代码化 | 启动 CoreCLR 前 `FPlatformMisc::SetEnvironmentVar(TEXT("DOTNET_TieredCompilation"), TEXT("0"))`（与 `FCoreCLRDomain.cpp:50-57` 的 `DOTNET_EnableDiagnostics` 同处） | **未采用**（需改 C++，且环境变量与 json 两处配置会形成双份真值） |
| ② 生成式 | 把 json 改成模板放非忽略路径，由构建/部署脚本生成到 `Binaries/Win64/` | **未采用**（实际上"模板"一直都在子模块里，见 §13.3，无需再建一套） |
| ③ 只记文档 | 接受"仅本机生效"，把 B 类当已知运行时竞态 | **未采用** |


---

## 5. 验证（全部本轮实测）

| 项 | 装置 / 命令 | 结果 |
|---|---|---|
| 构建 | `Build.bat UnrealCSharpTestEditor Win64 Development -project=… -waitmutex` | **Succeeded** |
| `-game` 全量套件（随包 10.0.4） | `Saved\Retest\run-game-accept.ps1` + `game_accept_driver.py` | **1705 / 1705，0 failed**；139 行 `LogUnrealCSharp`；0 `ensure`/`assert`；0 `LogUnrealCSharp: Error` |
| `-game` 全量套件（对照 10.0.12） | 同上（临时换运行时） | **1705 / 1705，0 failed** |
| PIE 全量（editor） | `pie_driver_g5g6b.py`（`G5G6_MODE=pie_generator`） | **1489 / 1489，0 failed** |
| **A 类复现**（修复前） | `repro-prod-verify.ps1`（跑到 CSV、正常退出、静置） | **5 / 22 ≈ 23%** |
| **A 类复现**（修复后） | 同装置 | **0**（本类判据一次未再命中） |
| **B 类缓解 A/B**（关键对照） | 同一台机、同一装置，仅切运行时配置 | 原配置（`TieredCompilation=true`）：**3 / 8 = 37.5%**；关闭分层编译：**0 / 215** |
| **交付态复测**（两处改动同时生效） | `repro-prod-verify.ps1`，5 批 | **0 / 215**（`tier-off` 45 + `tier-off-2` 45 + `mitig-stat` 45 + `final-state` 40 + `final-state-2` 40） |
| **独立旁证**（WER 事件监听） | `watch-crash-events.ps1` 全程在线 | 配置切换（03:18）**之后 0 条** `0x9603a` 事件（切换前：02 时 5 条、03 时 3 条） |
| **C 类**（插件窗口之前） | 同装置 | 仍偶发 **≈1/218 ≈ 0.5%**，不受上述缓解影响 |

**B 类 A/B 明细（同一晚、同一装置，连续时段）**：

| 批 | 运行时配置 | 启动次数 | 崩溃 |
|---|---|---|---|
| `control-orig`（r01–r07，原配置） | 原始 | 8 | **3**（`926/957/926` 行；1 次带插件日志） |
| `tier-off` | 关分层编译 | 45 | **0** |
| `tier-off-2` | 关分层编译 | 45 | **0** |
| `mitig-stat` | 关分层编译 | 45 | **0** |
| `final-state`（两处改动同时生效） | 关分层编译 + 域生命周期修复 | 40 | **0** |
| `final-state-2`（同上，复测） | 关分层编译 + 域生命周期修复 | 40 | **0** |
| 合计（修复+缓解交付态） | — | **215** | **0** |

> 统计口径：若真实崩溃率仍为 37.5%，215 次连续无崩溃的概率 < 1e-38 ⇒ 差异是配置造成的，不是机器状态漂移。
> **反向对照**：`System.GC.Concurrent=false` 单改也未见崩溃（40 次），但该批与本机状态漂移混在一起、证据力弱于上面的"同晚 A/B"，故**未采用**该项改动。

**🆕 提交后复验（去注释版 = 提交 `77867446` 的内容，30 次启动）**：

| 项 | 结果 |
|---|---|
| UBT 构建 | **Succeeded** |
| `-game` 全量套件（`nocomment-accept`） | **1705 / 1705，0 failed**；139 行 `LogUnrealCSharp`；0 `ensure`/`assert` |
| 30 次启动跑测（`nocomment-verify`） | **0 崩溃** |
| 与 `fix-archive\FScriptDomainScope.h.fixed` | SHA256 **一致**（存档已同步为无注释版） |

**装置/脚本产物**（均在工程侧）：

| 文件 | 用途 |
|---|---|
| `Saved\Retest\fix-archive\` | 🆕 **改动存档**：`FScriptDomainScope.h.{fixed,original}` + `CoreCLR.runtimeconfig.json.{fixed,original}`（可反复"还原↔恢复"做 A/B，不依赖 git） |
| `Saved\Retest\repro-prod-verify.ps1` | **推荐**：生产式反复跑测（跑到 CSV → 正常退出 → 静置 10~25 s） |
| `Saved\Retest\run-game-accept.ps1` + `game_accept_driver.py` | `-game` 全量套件验收 |
| `Saved\Retest\classify-crashes.ps1` | **把崩溃样本按"A/B/C 三类"自动判据分类**（用同批成功轮的首个托管输出行号作基准） |
| `Saved\Retest\repro-burst.ps1` | 高频启动压力复现（**会中途强杀编辑器**，其形态不代表真实使用） |
| `Saved\Retest\keep-dumps.ps1` / `watch-crash-events.ps1` | 转储搬运 / 事件+转储被动监听 |
| `Saved\Retest\tools\{SymTool,mini_x64.py,pebytes.py,pefind.py,imports.py,walk.py}` | 符号化、指令解码、PE 节/字节比对、IAT 解析、转储栈遍历 |

### 5.1 🆕 "存档 → 还原 → 复现 → 恢复存档 → 确认"的双轮 A/B（本机实测）

按同一装置、连续时段，把改动**真的还原回修复前**再编译，然后恢复存档再编译，各跑 30 次启动，重复两轮：

| 轮 | 构建 | 启动次数 | 崩溃 | 崩溃样本形态 |
|---|---|---|---|---|
| A（第 1 轮·修复前） | `~FScriptDomainScope(){ Destroy(Domain); }` + `TieredCompilation=true` | 30 | **1** | `ab-A-prefix-r03`：`907` 行 / `csharp=0` / +14 s |
| B（第 1 轮·修复后） | 存档恢复（`= default` + `TieredCompilation=false`） | 30 | **0** | — |
| A2（第 2 轮·修复前） | 再次还原 | 30 | **1** | `ab-A2-prefix-r11`：`896` 行 / `csharp=0` / +10 s |
| B2（第 2 轮·修复后） | 再次恢复存档 | 30 | **0** | — |
| **合计** | 修复前 60 / 修复后 60 | 120 | **2 / 0** | 两次命中均为文档记载形态（`894~907` 行 + 0 行插件日志） |

> 还原/恢复均以 `fix-archive` 的两份存档覆盖文件 + UBT 重编（各约 12 s），因此"A/B 差异"只可能来自这两处改动本身。

---

## 6. ⚠️ 换运行时版本不是修复（实测证伪）

| 版本 | 大小 | `.text` / `.pdata` | `UnpublishUnwindInfoForMethod` 等价字节 | 复现 |
|---|---|---|---|---|
| 随包 `10.0.4`（`Flavor=Retail`、`Company=Microsoft`） | 4 598 784 B | 3 618 085 / 171 600 | 见 §1.3 | **是** |
| 系统 `10.0.12` | 4 614 992 B | 3 622 709 / 171 600 | **逐字节相同**（`pebytes.py`：176 字节里 174 字节一致，前 4 字节属 `.pdata` 尾随） | **是**（`coreclr+0xFBEE2`，同一指令） |

---

## 7. 与历史各轮次的关系

原文 §2/§3 的双来源 26 次统计与逐轮对照**全部有效**；本轮补一条**硬关联**：把 26 条 1000 号事件时间与产物日志**逐条对齐**后统计行数分布 —— **26/26 全部落在 `894~906` 行且 `LogUnrealCSharp` = 0**（脚本见 §8③）⇒ 这 26 次与 §4.1 修的是**同一条路径（A 类）**。

---

## 8. 复用片段

### ① 三类判定（推荐直接用脚本）

```powershell
& Saved\Retest\classify-crashes.ps1 -Tags 'control-orig','tier-off','tier-off-2','mitig-stat','residual-a'
# 输出每个崩溃样本：Lines / CSharp 行数 / 同批成功轮的"首个托管输出行号" / PluginHadStarted(yes|NO)
#   有托管输出 或 行数 ≥ 该基准  -> 插件已运行（A 或 B 类）
#   0 行托管日志 且 行数 < 该基准 -> C 类（插件窗口之前，非插件可修）
```

### ② 事件日志侧确认（最可靠）

```powershell
Get-WinEvent -FilterHashtable @{LogName='Application'; ProviderName='Application Error'; StartTime=(Get-Date).AddHours(-2)} -MaxEvents 100 |
  Where-Object { $_.Message -match 'Fault offset:\s*0x0*9603a' -and $_.Message -match 'UnrealCSharp\\Binaries\\Win64\\shared\\Microsoft\.NETCore\.App' } |
  Select-Object TimeCreated
```

### ③ 历史 26 次与日志行数的逐条对齐

```powershell
$cut=[datetime]'2026-09-25 01:18:00'
$ev = Get-WinEvent -FilterHashtable @{LogName='Application'; ProviderName='Application Error'; StartTime=(Get-Date).AddDays(-12); EndTime=$cut} -MaxEvents 500 |
      Where-Object { $_.Message -match 'Fault offset:\s*0x0*9603a' }
$files = Get-ChildItem 'D:\file\UnrealProjects\UnrealCSharpTest\Saved\Retest\out' -Recurse -Filter '*.log' | Where-Object { $_.LastWriteTime -lt $cut }
foreach($e in $ev){ $b=$files | Sort-Object { [Math]::Abs(($_.LastWriteTime-$e.TimeCreated).TotalSeconds) } | Select-Object -First 1
  $c=Get-Content $b.FullName; "{0} lines={1} csharp={2} {3}" -f $e.TimeCreated,$c.Count,@($c|Select-String '\]LogUnrealCSharp').Count,$b.Name }
```

---

## 9. ⚠️ 原文判据更正

| 原文判据 | 实测 | 更正 |
|---|---|---|
| "无 CSV + `LogUnrealCSharp` **0 行** ⇒ 插件尚未进入" | ① 插件启动期**不打**该模块日志；成功运行里 `LogUnrealCSharp:` 首行出现在**托管套件开始之后**（本批实测基准行号 `897 / 930 / 942`）；② A 类崩溃（正在 `Generator()`）也是 0 行 | **"0 行"既不能证明"未进入"，也不能区分 A/C 类**；改用 **`Faulting module path`+`Fault offset`** + **日志是否越首个托管输出基准行** |
| "启动后约 6 秒必崩" | `Log file open 20:51:45` → 末行 `20:51:56` = **11 s**；实测崩溃落点 **+10~15 s** | 更正为「**插件 CoreCLR 域初始化 / 首次动态生成窗口**」 |
| "崩溃发生在插件代码进入之前"（原文 §0.4 / §4.4） | A 类 dump 崩溃侧含托管帧、卸载侧是插件 `FCoreCLRDomain::Deinitialize` | **A、B 类发生在插件代码执行期间并由插件触发；仅 C 类在插件窗口之前**（三类分开） |

---

## 10. 残余项（**未闭环**）

### 10.1 C 类：启动期、插件窗口之前

判据：`LogUnrealCSharp` 0 行 **且** 日志行数 < 同批成功轮的"首个托管输出行号"（本批基准 `897 / 930 / 942`）；样本行数 `896 / 915 / 926 / 928`。观测频率 ≈ **1/218 ≈ 0.5%**。
**为什么判为"非插件可修"**：崩溃时插件的托管代码尚未开始执行（日志停在引擎初始化/Python 插件初始化之间），但故障模块仍是插件随包的 `10.0.4\coreclr.dll` ⇒ 触发者是**进程内其它线程**（引擎/其它插件/运行时自身）对该 CoreCLR 的使用。

### 10.2 改动是否"泄露"（占位/句柄/内存）——结论：交付态**不新增泄露**

| 关注点 | 分析 | 判据 |
|---|---|---|
| 脚本域单例（`IScriptDomain`） | 修复后由 `Generator()` 作用域创建时仍 `IScriptDomain::Set()` 发布为单例；**同一次激活广播里紧接着的 `FCSharpEnvironment::Initialize()`（`FDomain` ctor）复用它**（`FDomain::Initialize()` 先 `IScriptDomain::Get()`，非空则不新建）⇒ 该域本来就会被创建并活到模块去激活，**生命周期不变** | `FDomain.cpp:22-31`、`FCSharpEnvironment.cpp:325-333` |
| 是否存在"创建了却没人认领"的域 | 两个 `Generator` 重载（`FDynamicGenerator.cpp:20` 与 `:143`）都把生成体包在 `FScriptDomainScope` 内，作用域不逃逸；**若某个上层路径调用了 `Generator` 但从未走 `FCSharpEnvironment::Initialize()`**（例如早退、或激活广播被跳过），则该次创建的可收集 ALC 会活到进程退出——**量级为 1 个 ALC，不随运行时间增长** | `grep FScriptDomainScope FDynamicGenerator.cpp` 仅 2 处 |
| 可收集 ALC | 卸载点仍是 `FScriptDomainFactory::Destroy → FCoreCLRDomain::Deinitialize → AssemblyLoaderUnloadFn`（热重载/去激活路径未动），因此**每次重建域仍会按原逻辑卸载旧 ALC**；本修复只是不再"生成结束就拆" | `FCoreCLRDomain.cpp:128-137, 240-246` |
| GC 句柄 / 注册表引用 | 本次改动**未触碰**任何 `GCHandle`/注册表路径（未新增、未删除分配点），泄露面不变 | `git diff` 仅 1 文件、11 增 12 删 |
| 内存/句柄随轮次增长 | 装置侧观察：连续 40~45 次启动（含套件跑满）无退出码异常、无 OOM、`MSBuild` OOM 事件与本改动无关（§11）；**未做专门的内存曲线测量**（如实标注） | ❌ 未测量 RSS/句柄曲线 |

> ⚠️ 如实标注：上表除"内存曲线"一行为**未测量**外，其余均有代码证据。若要 100% 坐实"无泄露"，建议做一次**长时驻留**测量（单实例跑 1~2 h，采样 `Get-Process` 的 `WorkingSet64/HandleCount`）——本次未做。

### 10.3 后续闭环建议

1. 抓 C 类的**完整转储**（`DumpType=2`）并装早期注入调试器（cdb/WinDbg）——托管 DAC 在该点报 "target runtime may not be initialized"，只有原生调试器能在异常发生时读寄存器与 XOR 链；
2. 在 `UnrealAssemblyLoadContext.Load()` 加**卸载哨兵**（卸载开始后返回 `null`），把"踩死亡 loader allocator"变成可捕获的托管异常，用于判定是否仍有残余 ALC 生命周期用法；
3. 若确认是运行时缺陷，按 dotnet/runtime issue 上报（**注意**：10.0.4 与 10.0.12 该处逐字节相同）。

---

## 11. 未归属项（原文 §7 保持）

| 族 | 计数 | 时间（+0800） | 运行时 | 偏移 |
|---|---|---|---|---|
| `dotnet.exe` | 5 | 2026-09-11 00:19:40 ~ 00:20:21 | `coreclr.dll 10.0.1226.42308` | `0x3596cf` |
| `UnrealEditor.exe` | 1 | 2026-09-25 00:26:40 | 同上 | `0x3596cf`（1026 号，托管栈 `P.RaiseException`） |

附带发现（与本文无关，单独登记）：`MSBuild.exe` 近 30 天约 60 次 `Unhandled Exception: OutOfMemoryException`（退出码 `-532462766`），Rider 后端日志有原文（`%LOCALAPPDATA%\JetBrains\Rider2026.2\log\backend.*.log`）。

工程侧另有一处**配置隐患**（非本次故障模块，但造成"同名多版本共存"）：`Source/ThirdParty/Mono/Mono.Build.cs:62` 把 **CoreCLR 10.0.1** 的 `coreclr.dll` / `System.IO.Compression.Native.dll` / `System.Globalization.Native.dll` 复制到 `$(BinaryOutputDir)`，与 CoreCLR 后端的 `shared/Microsoft.NETCore.App/10.0.4/*` 在同一目录树共存；实测编辑器**实际加载 10.0.4**（模块监视证据），建议清理。

---

## 12. 交叉引用

| 位置 | 记了什么 |
|---|---|
| [`08-…/08`](08-G5与G6首刀（编译链路并发与关停，2026-09-23）.md) **§3.5** | **首次登记**（`G5G6e-final` / `G5G6f-final`）与"无 CSV ⇒ 重跑"口径 |
| [`08-…/07`](07-G1句柄释放责任归属（2026-09-22）.md) | 09-22 的 3 次命中 |
| [`08-…/06`](06-复测轮：G7间歇项定位与null句柄缺陷（2026-09-17）.md) | 09-17 的 12 次命中（当轮记 `0x80131506`，与 1023 号事件一致） |
| [`08-…/09`](09-G2注册表句柄契约收口（2026-09-24）.md) | G2 末轮命中 |
| [`08-…/11`](11-G13前段收口-转义与模板写盘-动态特性与UHT诊断（2026-09-24）.md) | G13 前段收口轮 3 次命中 |
| **改动 1（代码修复，✅ 已提交）** | `Plugins/UnrealCSharp/Source/UnrealCSharpCore/Public/Domain/Script/FScriptDomainScope.h` —— 提交 **`778674463fb66c3c22422c1ddec9f5d924923191`**（`1 file changed, 1 insertion(+), 12 deletions(-)`） |
| **改动 2（运行时缓解，✅ 已随源码分发并已提交）** | `System.Runtime.TieredCompilation=false` —— 落点 = **CoreCLR 子模块里被 git 跟踪的 8 份模板** `Source/ThirdParty/CoreCLR/lib/{Release,Debug}/{Win64,Linux_x86_64,macOS_arm64,macOS_x86_64}/CoreCLR.runtimeconfig.json`（构建期由 `CoreCLR.Build.cs` 按 `bUseRelease=true` 从 `lib/Release/<平台>/` 拷到 `Binaries/`）；`Binaries/Win64/` 那份**只是产物**（`.gitignore:47 Binaries/*`），已与模板同步为逐字节一致（SHA256 `C60FD9F7…`）。🟢 **提交**：子模块 **`1e1cfd99`**（父 `c27d952e`，8 文件 `+14/−6`）+ 插件仓指针 **`6e142471`**（父 `0170514d`，`1 文件 +1/−1`）；决策与实测见 §4.2 / **§13.3** |
| 本文 | 三类分解 + 根因 + 修复/缓解 + A/B 验证 + 判据更正 |

---

## 13. 收口（2026-09-25 同日；本文写完后落库的提交与口径变更）

> 本文正文（§0–§12）记录的是**提交前**的施工与验证状态；本节是写完后**当日**发生的三件事与最终口径。

### 13.1 修复合入提交

| 项 | 值 |
|---|---|
| 提交 | **`77867446da413c543dc8bef5751c951a17fe8826`**（subject `DelegateHandle Reset && UFunction RemoveFromRoot`，作者 `crazytuzi <liuxiangcode@qq.com>`，`2026-09-25 23:43:19 +0800`，**= 当前 HEAD**）—— **本条（A 类）的代码修复就在这一笔里** |
| 与 B 类缓解那一笔的分工 | 父提交 **`6e14247199d957d22b0389b7c285da06d34b6a2c`**（subject **`CoreCLR System.Runtime.TieredCompilation false`**）**只做一件事**：把 `Source/ThirdParty/CoreCLR` 指针从 `c27d952e` 改到 **`1e1cfd99`**（`1 file changed, 1 insertion(+), 1 deletion(-)`）⇒ 它是 **B 类缓解**的入口、**不含 A 类修复**；A 类修复全在本行这一笔 |
| 改动面 | 整笔 = **`6 files changed, 23 insertions(+), 17 deletions(-)`**：其中 A 类修复 = **`1 file changed, 1 insertion(+), 12 deletions(-)`**（唯一文件 `Source/UnrealCSharpCore/Public/Domain/Script/FScriptDomainScope.h`，**纯删除、无新增注释**，diff 与 §4.1 完全一致）+ G12 收口 **5 文件 `+22/−5`**（见 [`08-…/13`](13-G12收口-编辑器对象与委托配对（2026-09-25）.md) §4.1） |
| 效果 | **A 类 `5/22 ≈ 23%` → `0/215`**；修复后按同一装置另跑 **30 次启动 0 崩溃**、`-game` 全量 **1705 / 1705** |
| 后续关联提交 | `Plugins/UnrealCSharp` 当前 `HEAD` = 上述 `77867446`（本库后续引用该修复一律用此哈希） |

### 13.2 口径变更（已同步到索引与清单）

1. **判据**：弃用"**无 CSV + `LogUnrealCSharp` 0 行 ⇒ 重跑**"（§9 已列明三条理由）。新判据 = 事件日志 **`Faulting module path`（必须是插件随包 `Binaries\Win64\shared\Microsoft.NETCore.App\10.0.4\coreclr.dll`）+ `Fault offset 0x9603a`**，形态分级用 §8① 片段或 `Saved\Retest\classify-crashes.ps1`（判据三行：日志 `894~906`/`957~983`/`896~928` 行；`LogUnrealCSharp` 0/16~40/0 行；是否越过首个托管输出基准行 `897 / 930 / 942`）。
2. **处置分级已改变**：**A 类与 B 类现在应当"不再出现"**（分别由 `77867446` 与本机运行时配置压到 0/215）⇒ 若再撞到，**按新缺陷处理，不要按"环境 flake 重跑"放过**；**只有 C 类样本才继续"重跑、不当回归"**。
3. **已同步的落点**：`README.md` 的"环境事实（CoreCLR 启动期崩溃）"段（含出处表的 `77867446` 登记）、`INDEX.md` 的 12 号行 + 当日注、`09-…/00` **§6 实机清单第 1 行**（把它作为"P0/P1 崩溃类断言必然性"的**首个实机坐实案例**）、`08-…/08` §3.5 与 §221 行的追加更正块（旧判据保留但标注作废）。
4. ⚠️ **不进 `F-*` 编号体系、不参与级别统计**（本文性质是环境事实）；`README` 的"已修复范围"一节讲的是 **26 条 P0 编号**，本修复**不在其中**（避免把它读成第 27 条）。

### 13.3 ✅ B 类缓解已落地：改**子模块里的模板源**（2026-09-25，当日决策 + 实施）

**决策**：不采用 §4.2 的三条候选，直接改**模板源** —— 该 json 的源文件本来就在子模块里、且被 git 跟踪，`Binaries/` 下那份只是构建产物。

**装置链（本轮核实）**：

| 环节 | 事实 |
|---|---|
| 源 | `Plugins/UnrealCSharp/Source/ThirdParty/CoreCLR/lib/{Debug,Release}/{Win64,Linux_x86_64,macOS_arm64,macOS_x86_64}/CoreCLR.runtimeconfig.json` —— **8 份，全部被 CoreCLR 子模块 git 跟踪**（子模块 HEAD `c27d952e`，改前工作区干净） |
| 拷贝规则 | `CoreCLR.Build.cs` 的 `RuntimeDependencies.Add($"{BinaryOutputDirectory}/CoreCLR.runtimeconfig.json", Path.Combine(PlatformLibraryPath, "CoreCLR.runtimeconfig.json"))`；且 **`bUseRelease = true`** ⇒ 生效源 = **`lib/Release/<平台>/`**（`Debug/` 那一套当前不参与构建） |
| 目标 | `Binaries/Win64/CoreCLR.runtimeconfig.json`（编辑器构建）或 `$(BinaryOutputDir)`（非编辑器）；该路径被插件 `.gitignore:47 Binaries/*` 忽略 ⇒ **产物不是真值** |

**改动**：8 份模板的 `configProperties` 末尾加 `"System.Runtime.TieredCompilation": false`（Linux 两份的 `AppLocalIcu` 之后、其余在 `GC.Server` 之后 —— 即"插在块末"，不动既有键序）；**CRLF / 无 BOM / 无尾换行**逐文件保持；`ConvertFrom-Json` 全部可解析、键数 4（Linux 5，多 `AppLocalIcu`）。**子模块 diff = 8 文件 `+14/−6`**（那 6 行"删除"是给上一行补的逗号，不是内容丢失）。

**部署副本同步**：把 `lib/Release/Win64/CoreCLR.runtimeconfig.json` **逐字节复制**到 `Binaries/Win64/CoreCLR.runtimeconfig.json` ⇒ 两侧 **SHA256 同为 `C60FD9F7F02CB09390C5FFBAE6B6F0A200ED91CFA64C0021F525D6FAFB074BF9`**（各 391 B）⇒ "下次 UBT 会拷什么"与"现在跑的是什么"**逐字节同源、可判定**（改前两者是 341 B 模板 vs 382 B 手工版，只语义相同、字节不同）。

**为什么把平台一起改**：8 份模板同形同源，B 类根因（分层编译后台重 JIT 与"代码发布/取消发布"竞态）**不依赖平台**；且平台间文件内容差异只有 `AppLocalIcu` 一处 ⇒ 只改 Win64 会留下"同一缺陷在其它平台仍活跃"的分叉。（代价如实：非 Win 平台的绝对 JIT 成本未测，同 §4.2 的"启动期 JIT 工作量上升"。）

**🟢 已闭环（2026-09-25 23:25/23:26 提交）**：CoreCLR 子模块 **`1e1cfd9939109f6c59dace5c97ee1c302bd0a117`**（subject **`System.Runtime.TieredCompilation false`**，父 `c27d952e1d58d06936033f0fd45e2aebdae980d5`，**`8 files changed, 14 insertions(+), 6 deletions(-)`** —— 与本节上面实测的"8 份模板"**逐文件一致**）+ 插件仓指针提交 **`6e14247199d957d22b0389b7c285da06d34b6a2c`**（subject **`CoreCLR System.Runtime.TieredCompilation false`**，父 **`0170514dc320848307c3f0f9f7f8e41fe75ead21`**，**`1 file changed, 1 insertion(+), 1 deletion(-)`**，`Source/ThirdParty/CoreCLR` 由 `c27d952e` 指向 `1e1cfd99`）⇒ **新 clone / 换机器 / CI 现在会拿到 B 类缓解**（不再只限本机）。⚠️ 顺序：指针提交 `6e142471` 的父是 `0170514d`，而 **`77867446` 的父就是 `6e142471`** ⇒ 自 `0170514d` 起是一条线性链；`1e1cfd99` 的父是子模块原 HEAD `c27d952e`。

### 13.4 残余（本节收口后仍开放）

| 项 | 状态 | 出处 |
|---|---|---|
| **C 类**（启动期、插件窗口之前） | 🔴 仍偶发 `≈1/218 ≈ 0.5%`，**缓解措施无效**、**非插件可修** | §10.1 |
| C 类闭环三条建议（完整转储 `DumpType=2` + 原生调试器 / `UnrealAssemblyLoadContext.Load()` 卸载哨兵 / 按 dotnet/runtime issue 上报） | 未做 | §10.3 |
| 改动是否"泄露"（占位/句柄/内存） | 代码证据齐（§10.2），**仅"内存/句柄随轮次增长"未测量**（建议 1~2 h 长时驻留采样 `WorkingSet64`/`HandleCount`） | §10.2 |
| 两条未归属同形崩溃（`0x3596cf`） | 未定位；`MSBuild.exe` 的 ~60 次 `OutOfMemoryException` 单独登记 | §11 |
| 工程侧配置隐患（`Mono.Build.cs:62` 把 CoreCLR 10.0.1 的几个原生库投放到同一目录树） | 建议清理，**未做** | §11 |
