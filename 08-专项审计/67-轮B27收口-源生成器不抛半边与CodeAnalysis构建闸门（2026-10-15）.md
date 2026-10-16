# 轮 B-27 收口：`F-CS4-009`（源生成器"不抛 / 不误报"半边，方案 A）＋ `F-GEN3-012`（`dotnet build` 成本实测与产物时间戳闸门）＋ 三问待决策页

> **来源**：维护者指定 —— "**开始你一句话结论中的三个任务吧**"（接 [`../09-问题清单/00-推荐优先修复清单.md`](../09-问题清单/00-推荐优先修复清单.md) §0.0 T2 表的 3.20 / 3.01 档顺位；两条施工的**方案与判据设计**来自同目录 [`64-裁定材料-F-INT1-004与口径四项（2026-10-15）.md`](64-裁定材料-F-INT1-004与口径四项（2026-10-15）.md) **§2.2 方案 A** 与 **§2.3 方案 A**；第三项 = `64` §3.3 三问的**一页待决策页** = [`66-待决策页-三问（Q1四路线·Q2口径例外·Q3失败判据）（2026-10-15）.md`](66-待决策页-三问（Q1四路线·Q2口径例外·Q3失败判据）（2026-10-15）.md)）。
> **性质**：**两条施工**（C# 侧 1 文件 ＋ C++ 侧 1 文件）＋ **一页待决策页**（0 代码改动）。判据装置为**临时装置**（Roslyn driver 项目 ＋ 进程采样脚本 ＋ 物化变换复刻脚本），全部落在工程侧 `Saved/Retest/`，**未进插件仓**。
> **环境**：UE 5.6.0 · Win64 · CoreCLR · MSVC 14.51.36260（Android NDK 见 §4 控制项）；插件基线 = **轮 B-26 交付态**；套件交付基线 = **1931 例 / 0 失败 / CSV `B429F3EE…`**。
> 🟢 **交付提交**：**`df487a6d132f451ff619014d3d0913d7ee9e4ce5`**（作者 `crazytuzi <liuxiangcode@qq.com>`，2026-10-16 10:50:24 +0800，提交信息 = `Asset NameSpace Encode && Commandlet Platform Guard && SourceGenerator Null Validation && CodeAnalysis Compile Skip`；**本轮 3 文件**，**numstat** = C# `32/22`、`FCodeAnalysis.cpp` `74/35`、`FCodeAnalysis.h` `4/0`）。⚠️ **提交树 ≠ 施工期工作区**：维护者做了 **4 处纯形态调整**（见 §10.5），故已**在提交树上重跑全部判据**（§4 与 §10.5，读数与施工期逐条相同）。
> **改动面**：插件 **3 文件**（`Script/SourceGenerator/UnrealTypeSourceGenerator.cs` **`+32/−22`**；`Source/ScriptCodeGenerator/Private/FCodeAnalysis.cpp` **`+78/−31`**（提交树 `74/35`）＋ `Source/ScriptCodeGenerator/Public/FCodeAnalysis.h` **`+4/−0`**）；工程侧 **0 交付改动**（装置在 `Saved/Retest/`；生成器的**物化副本**已按 `CopyTemplate` 的两条变换复刻并**逐字节自证**，见 §2.2）；C# 运行时 0 / 生成产物 0 / 协议面 0 / 导出签名 0 / **未新增公开 API**（新增的 `RemoveTypePrefix` = `internal static`；C++ 侧两个辅助函数 = **类内 `private static`**（声明在 `FCodeAnalysis.h` `private:` 段，`+4/−0`））/ 新增注释·日志·`throw`·`try-catch`·`ensure`·`check`·`static_assert` **全 0**。

---

## 0. 摘要

| 项 | 读数 |
|---|---|
| `F-CS4-009`（P2 · S=3.20 · T2 第 12 位） | ✅ **不开口半边（方案 A）已施工**：11 处 `… SyntaxTree ?? throw new InvalidOperationException()` → **直接取同一棵树**；3 处 `Path.GetFileName(filePath)` 补 `string.IsNullOrEmpty(filePath) == false` 守卫；4 处 `Substring(1)` → 长度守卫（新增 `RemoveTypePrefix`）；`errorAttribute` 的隐式不变量改为**按同一来源再判一次** |
| `F-CS4-009` 判据 A（**误报翻转**，唯一可构造的行为变化） | 🔴 改前：无文件路径的语法树（`SyntaxTree.FilePath == ""`）在 **enum / interface / class** 三处各产生 **1 条 `UC_ERROR_02` 误报**；interface 那处还因 `hasError = true` **抑制了生成产物**（gen = 0）→ 🟢 改后：**三条误报全消失（diag 0）**，且 interface 的**生成产物恢复**（gen = 0 → **2**，`Script.CoreUObject.IThing.gen.cs` / `UThing.gen.cs`） |
| `F-CS4-009` 判据 B（**潜伏 throw**） | 🔴 改前：`GetEngineName("")` / `GetPathName("")` **抛 `ArgumentOutOfRangeException`** → 🟢 改后：分别返回 `""` 与 `/Script/CoreUObject.`（不抛） |
| `F-CS4-009` 判据 C（**其余诊断逐条不变**） | 🟢 **22 例电池中其余 19 例的输出逐行相同**（含 `UC_ERROR_01/02/03/04/05/06/07` 七类全部出现、`GEN` 摘要逐一相同）；全部 22 例 **`exc=0`** |
| `F-CS4-009` 判据 D（真实构建） | 🟢 `dotnet build Script/Script.sln` = **exit 0 / 0 Warning / 0 Error**（19.18 s）；分析器 `SourceGenerator.dll` 于 23:41:27 重建、`Game.dll` 23:41:44 产出（**证明判据走的是打过补丁的分析器**） |
| `F-GEN3-012`（P2 · S=3.01 · T2 第 29 位）—— **触发点普查** | 🔴 **对 `64` §2.3 ①(c) 的一处更正**：`UnrealCSharpEditor.cpp:108` 是**控制台命令** `UnrealCSharp.Editor.CodeAnalysis` 的 lambda（**手动**），**不是启动路径**；**自动触发只有一个** = `FEditorListener::OnPostEngineInit()`（`FEditorListener.cpp:202`）；`:360` 在 `Generator()` 内（**每次生成一次**） |
| `F-GEN3-012` —— **单次成本实测**（精确命令行） | `dotnet build "<…>/Script/CodeAnalysis/CodeAnalysis.csproj" --nologo -c Debug`：**up-to-date 0.931 / 0.906 / 0.932 s**（MSBuild 自身 0.71–0.74 s）；**触碰源文件后重建 1.083 s**（MSBuild 0.88 s）⇒ 单次 **≈0.9–1.1 s 同步阻塞** |
| `F-GEN3-012` —— **每会话频次实测**（进程采样，非侵入） | Editor 会话 110 s：**恰好 1 次** CodeAnalysis 构建（23:46:06，命令行逐字捕获）⇒ **≈0.9 s / 每次编辑器启动**；对照 B-13 判"非热点"的 `F-RM-013`（9–23 ms/会话）大 **~40–100×**，对照 B-26 判"非热点"的 `F-MACRO-003`（0.26–0.42 ms/会话）大 **~2000–3500×** |
| `F-GEN3-012` —— **方案 A 已施工并取得 A/B** | 🟢 新增**产物时间戳闸门**（`Compile()` 起手）：**up-to-date 会话 = 0 次构建**（改前 1 次）；**正向对照**（触碰 `Script/CodeAnalysis/CodeAnalysis.cs`）**= 1 次构建**，产物 `CodeAnalysis.{exe,dll}` 由 23:42:03 → **23:52:11** 的确重建 |
| 套件（零副作用） | **1931 行 / 0 失败**，CSV `B429F3EE…` —— 与 B-24 / B-25 / B-26 交付态**逐字节相同** |
| 交付态 | Windows UBT **Succeeded**；产物 manifest **16 230 文件 / 0 行差异**；`LogUnrealCSharp: Error` / `ensure` / `assert` / `Fatal` / `Unhandled Exception` / `error CS` **全 0**；`Saved/Crashes` 今日新增 **0** |
| 第三项 | 🟢 [`08-…/66`](66-待决策页-三问（Q1四路线·Q2口径例外·Q3失败判据）（2026-10-15）.md) 已产出（84 行，含三问表 ＋ **拍板回执表** ＋"不拍板会发生什么"＋ `§复用价值`；拍板后解锁 **9 条编号**） |

> 📌 **一句话**：把源生成器里**"必抛"的写法换成"直接取既有的非空来源"**、把**无路径语法树的 3 处误报**关掉（顺带恢复了被误报抑制掉的生成产物），并把 `FCodeAnalysis::Compile()` 的"每次启动都同步跑一遍 `dotnet build`"改成**产物时间戳闸门**（实测省下 ≈0.9 s/次启动，且触碰源文件后仍会重建）。

> 🔴 **收尾阶段抓出的一个真缺陷（见 §10.4）**：应维护者复审做"命名返工"时，把 `IsCompileUpToDate()` 改名成 `IsProgramOutdated()` **只翻了调用点的 `!`、没翻函数体的返回值** ⇒ 闸门语义**整体反转**（新鲜时构建、陈旧时跳过）。它被本轮的**重取判据**当场抓住（反转态的 A/B 读数与期望恰好相反），修正后三会话 A/B/C 全对：**陈旧 ⇒ 1 次构建且产物重建 / 新鲜 ⇒ 0 次 / 触碰源 ⇒ 1 次且产物重建**。

> ⚠️ **判据能力边界（先读）**：① `F-CS4-009` 的 11 处 `?? throw` 在本轮**未能构造出可达输入**（§2.4-1）⇒ 那一半是**形态消除**，不是"修掉了一次真实崩溃"；② `F-GEN3-012` 的闸门**未测**非源码因素（NuGet / SDK 升级）导致的重建，且**只动 `Compile()`**、`Analysis()` 侧未加闸门；③ 读数含**施工期**与**提交树**两套：**最终以提交树（`df487a6d`）读数为准**（§4 与 §10.5；两者逐条相同）。

---

## 1. 设计前提

### 1.1 `F-CS4-009` 改前代码事实（轮 B-26 交付态，逐行核对）

| # | 事实 | 证据（HEAD，插件仓） |
|---|---|---|
| C1 | **11 处** `… SyntaxTree ?? throw new InvalidOperationException()`（10 行，1 行折行） | `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:329 / :341 / :376 / :392-393 / :480 / :493 / :506 / :522 / :535 / :817 / :836`（与 `64` §4.2-4 的 HEAD 行号**逐点相同**） |
| C2 | **4 处** `Substring(1)` | `:66`（`GetEngineName`）、`:107` / `:117`（interface 名去前缀）、`:703`（文件名推导）；另有一处 `:384` 为 `Substring(1, Length-1)`**长度受控**、不在清单内 |
| C3 | **3 处** `Path.GetFileName(filePath)` | `:337`（enum）、`:386`（interface）、`:705`（class）；`filePath` 均为 `…GetLocation().SourceTree?.FilePath`（`:308` / `:351` / `:450`） |
| C4 | `errorAttribute` 的**隐式不变量** | `:545` 声明 → `:551` / `:557` 在两个分支赋值 → `:563` **直接取用**；不变量靠 `:444` `bIsUClass = attributeUClass != null` 与 `:446` `bIsUStruct = attributeUStruct != null` 维持 |
| C5 | **零 `try/catch`** | 全文件 `\btry\b\|\bcatch\b` grep = **0**（此前的"8 处"是 `TryGetValue` 被泛匹配误计） |
| C6 | 两个 `[Generator]` 类 | `:10-11` `UnrealTypeSourceGenerator`、`:1072-1073` `LibraryBridgeGenerator`（本轮只动前者） |
| C7 | 程序集名闸门 | `:81` `if (Context.Compilation.AssemblyName != GameAssemblyName) return;`；模板里 `GameAssemblyName = "GameNamePlaceholder"`（物化时由 `ReplaceGameName` 改成 `"Game"`） |

### 1.2 `F-GEN3-012` 改前代码事实与**触发点普查**（轮 B-26 交付态）

| 项 | 内容 |
|---|---|
| **命令形态** | `FCodeAnalysis.cpp:43-47`：`"build \"%s\" --nologo -c Debug"`，`%s` = `GetCodeAnalysisProjectPath()`；`:49-51` 的 `SyncProcess` 回调是**空 lambda**（返回码/输出丢弃） |
| **组件形态** | `:89-107 Compile()`（`static auto CompileTool = GetDotNet()`）；`Analysis()` 两个重载（`:12-37` / `:54-89`）都**不校验 exe 是否存在** |
| **产物位置** | 工程侧 `Script/CodeAnalysis/`（`CodeAnalysis.csproj` 的 `<OutputPath>.</OutputPath>`）⇒ 程序 = `Script/CodeAnalysis/CodeAnalysis.exe`（Windows）＋ `CodeAnalysis.dll` |
| 🔴 **触发点更正** | `64` §2.3 ①(c) 记 "`UnrealCSharpEditor.cpp` HEAD `:108`（**启动路径**）与 `:360`（在 `Generator()` 内）" —— **本轮实读**：`:103-109` 是 `FAutoConsoleCommand(TEXT("UnrealCSharp.Editor.CodeAnalysis")…)` 的 lambda（**手动**触发）；**自动路径只有一个** = `FEditorListener.cpp:200-205 OnPostEngineInit()` → `:202 FCodeAnalysis::CodeAnalysis()`；`:360` 在工具栏 `Generator()` 内（每次生成一次）。⇒ "每次编辑器启动 1 次"是**实测坐实**的（§3.3） |
| **同仓先例** | `FUnrealCSharpFunctionLibrary::IsScriptPublishOutdated()`（`…Common/FUnrealCSharpFunctionLibrary.cpp:1301-1327`）：`PlatformFile.FileExists` ＋ `GetTimeStamp` 比对 |

### 1.3 装置（三条，均为**临时**、可复算）

| 装置 | 落点 | 用途 / 关键设计 |
|---|---|---|
| **Roslyn driver** `B27Driver` | `Saved/Retest/b27-driver/`（`B27Driver.csproj` + `Program.cs`） | **直接编译被测生成器的唯一源文件**（`<Compile Include="..\..\..\Plugins\UnrealCSharp\Script\SourceGenerator\UnrealTypeSourceGenerator.cs" Link=…/>`），经 `CSharpGeneratorDriver` 跑 **22 例电池 ＋ 4 个直调探针**，输出固定语法记录（`CASE/FILE/ASSEMBLY/DIAG/GEN/EXC/END/PROBE`）⇒ 改前 / 改后**逐行 diff** |
| **进程采样器** `b27-proc-sample.ps1` | `Saved/Retest/` | 启动编辑器后按 **300 ms** 采样 `Win32_Process`（`dotnet.exe`），记录**首见时刻 ＋ 命令行**，把 `MSBuild.dll … nodemode` 的常驻 worker 单列，命中 `CodeAnalysis.csproj` 的记为一次构建 ⇒ **非侵入**地取"每会话构建次数" |
| **物化变换复刻** `b27-refresh-generator.ps1` | `Saved/Retest/` | 复刻 `FSolutionGenerator` 对生成器的两条变换（`ReplaceGameName` ＋ `AddCSharpGeneratorHeaderComment`，后者 = 头注释 LF→CRLF ＋ 一个 CRLF），**`-Mode validate` 要求"HEAD 模板 → 变换后"与现工程副本 `Ordinal` 逐字符相同**（实测 `True`，48 056 字符），`-Mode refresh` 才写盘 |

> **电池规模**：`uclass`（9 例）／`ustruct`（3 例）／`uenum`（4 例）／`uinterface`（4 例）／成员级（2 例）—— 每类都含"合法 / 命名违规 / 文件名不匹配 / **无路径**"四种形态，覆盖 `UC_ERROR_01…07` 全部分支。

---

## 2. `F-CS4-009`（源生成器"不抛 / 不误报"半边）—— 施工与判据

### 2.1 改法（`Script/SourceGenerator/UnrealTypeSourceGenerator.cs`，`+32/−22`）

| 组 | 改前 | 改后 | 依据 |
|---|---|---|---|
| ① **11 处 `?? throw`** | `X.Identifier.SyntaxTree ?? throw new InvalidOperationException()` | `X.SyntaxTree`（token 所在节点与声明节点**同一棵树**） | 语义等价（`Location.Create(tree, span)` 的 tree 与 span 都不变），**只剩"不抛"这一差别**；`:392-393` 的折行写法同时收敛为一行 |
| ② **3 处 `Path.GetFileName`** | `if (Path.GetFileName(filePath) != currentFileName)` | `if (string.IsNullOrEmpty(filePath) == false &&\n    Path.GetFileName(filePath) != currentFileName)` | Roslyn 的无路径树 `FilePath` **默认空串**（不是 null）⇒ 必须 `IsNullOrEmpty` 才覆盖两种"无路径"形态 |
| ③ **4 处 `Substring(1)`** | `Name.Substring(1)` / `@interface.Name.Substring(1)` / `name.Substring(1)` | 新增 `internal static string RemoveTypePrefix(string Name) => Name.Length > 0 ? Name.Substring(1) : Name;`（模板 `:69-72`），四处改调它（`:66` / `:112` / `:122` / `:712`） | 唯一"会抛"的 `Substring(1)` 形态是**空名**；除 `GetEngineName` 外三处调用点位于**另一类**（`UnrealTypeReceiver`）⇒ 按同仓既有风格限定为 `UnrealTypeSourceGenerator.RemoveTypePrefix(…)`（`:712`） |
| ④ **`errorAttribute` 不变量** | 直接取用 `errorAttribute.SyntaxTree`（靠 `bIsUClass/bIsUStruct` 与 `attributeUClass/attributeUStruct` **隐式同步**） | 加 `if (errorAttribute != null)` 包裹（`:568-573`）—— **同一来源再判一次** | 不变量成立时**行为逐字不变**；不成立时不再 `Location.Create(null, …)` |

### 2.2 判据装置（为什么必须是 driver，而不是套件）

- 套件（`-game` 1931 例）里**没有**"缺 `partial` 的用户类"这类输入（§5-1），且源生成器只在**编译期**运行 ⇒ 套件判不了本条（同 `64` §2.2 ⑤ 的判断）。
- 工程侧的**物化副本**才是真正被 `<ProjectReference … OutputItemType="Analyzer">`（`Script/UE/UE.csproj`、`Script/Game/Game.props`）使用的分析器 ⇒ **改模板后必须刷新副本**，否则 `dotnet build` 跑的是旧分析器、判据**全假绿**。本轮用 `b27-refresh-generator.ps1 -Mode validate` 先自证变换忠实（HEAD 模板 → 现副本**逐字符相同**），再 `-Mode refresh` 落地（副本 1 256 行 / SHA256 `981EB6CD…`；🔁 **规范返工后副本更新为 `787062F9…`**，见 §10）。

### 2.3 改前 ↔ 改后读数（同一装置、同一命令）

`pre.txt` ↔ `post.txt` 的**完整差集**（`Compare-Object`，共 6 组）：

```
=> PROBE GetEngineName(empty)|ok|[]                        <= PROBE GetEngineName(empty)|exc|ArgumentOutOfRangeException
=> PROBE GetPathName(empty)|ok|/Script/CoreUObject.        <= PROBE GetPathName(empty)|exc|ArgumentOutOfRangeException
=> END uclass-partial-ok-pathless|diag=0|gen=1|exc=0       <= DIAG UC_ERROR_02|…|"UFoo" … must be "Foo.cs"|(empty)|3|55
                                                           <= END uclass-partial-ok-pathless|diag=1|gen=1|exc=0
=> END uenum-file-mismatch-pathless|diag=0|gen=0|exc=0     <= DIAG UC_ERROR_02|…|enum "EThing" … must be "EThing.cs"|(empty)|3|45
                                                           <= END uenum-file-mismatch-pathless|diag=1|gen=0|exc=0
=> GEN Script.CoreUObject.IThing.gen.cs|be161cda7ad235ac   <= DIAG UC_ERROR_02|…|interface "IThing" … must be "Thing.cs"|(empty)|3|55
=> GEN Script.CoreUObject.UThing.gen.cs|8898be78b5424c41   <= END uinterface-file-mismatch-pathless|diag=1|gen=0|exc=0
=> END uinterface-file-mismatch-pathless|diag=0|gen=2|exc=0
```

读数归纳：

| 面 | 改前 | 改后 | 判读 |
|---|---|---|---|
| 无路径树的误报 | **3 条** `UC_ERROR_02`（enum / interface / class） | **0 条** | 🟢 **唯一可构造的行为翻转**（判据 A） |
| 误报的**二阶后果** | interface 例 `hasError = true` ⇒ **不产出生成源**（gen=0） | gen=**2**（`IThing.gen.cs` ＋ `UThing.gen.cs`） | 🟢 误报不修正则"合法输入被静默漏产物" |
| 空名输入 | 2 个探针**抛** `ArgumentOutOfRangeException` | 不抛（`""` / `/Script/CoreUObject.`） | 🟢 潜伏 throw 消除（判据 B） |
| 其余 19 例（含 7 类 `UC_ERROR_*` 全部命中、`GEN` 摘要） | — | **逐行相同** | 🟢 诊断集合不变（判据 C） |
| 异常计数 | `exc=0` × 22 | `exc=0` × 22 | ⚠️ **改前也未触发**（见 §2.4-1） |

### 2.4 前提更正（三条，逐条给出依据）

1. 🔴 **11 处 `?? throw` 对"已解析语法树"静态不可达** —— `Identifier.SyntaxTree` 对任何来自 `CSharpSyntaxTree` 的节点**恒非空**（`SyntaxNode.SyntaxTree` 是 Roslyn 的非空属性；只有 `SyntaxFactory` 合成节点才为 null，而 `ISyntaxReceiver` 只收到解析树里的节点）。**实测印证**：22 例电池（含全部命名/文件名/唯一性违规、含无路径树）**没有一次**进入 `throw`（`exc=0`）。⇒ `64` §2.2 ①(b) 记的"会抛的写法"是**代码事实**，但"异常会以 `source generator failed` 中断整个编译"这一**后果**在本轮**未能构造**；本轮据此把判据 A 从"防 CS8785"改为"**误报翻转**"（`64` §2.2 ⑤ 的判据 1 因此**未取得**，见 §5-1）。
2. 🔴 **`Path.GetFileName(null)` 不抛**（.NET 对 null 返回 null）⇒ 三处 `Path.GetFileName` 的真实风险不是异常，而是 `FilePath == ""` 时的**误报**；且 Roslyn 的默认值是**空串**而非 null ⇒ 守卫必须写成 `string.IsNullOrEmpty`。这一条把 `64` §2.2 方案 A 的"③ 加空值守卫（消除误报诊断）"从**推断**变成**实测**。
3. 🔴 **触发点更正**（`F-GEN3-012`）：见 §1.2 的"触发点更正"行 —— `UnrealCSharpEditor.cpp:108` **不是**启动路径，自动路径只有 `FEditorListener::OnPostEngineInit()` 一个。

### 2.5 未做 / 残余

- **开口半边不做**：外层 `try/catch` → 既有诊断通道（`64` §2.2 方案 B）**仍不施工**（`05-…` §18.2 的裁定不变；口径是否松动见 `66` 的 Q2）。
- **未加诊断 ID**：不新增 `UC_ERROR_08`、不动 `AnalyzerReleases.Unshipped.md`（`F-CS4-007` 未修）。
- 其它 `Substring` / `SyntaxTree` 直接取用点（如 `:718` 的 `Location.Create(Syntax.Identifier.SyntaxTree, …)`）**未动**（无 `?? throw`、无长度风险，属零改动原则）。
- `LibraryBridgeGenerator`（`:1072+`）**未动**（不在本条对象内；其语义模型分支未被电池覆盖）。

---

## 3. `F-GEN3-012`（`dotnet build` 成本实测 ＋ 产物时间戳闸门）—— 施工与判据

### 3.1 触发点普查（决定"每会话成本"的分母）

| 调用链 | 位置 | 性质 | 频次 |
|---|---|---|---|
| `FCoreDelegates::OnPostEngineInit` → `FEditorListener::OnPostEngineInit()` → `FCodeAnalysis::CodeAnalysis()` | `FEditorListener.cpp:200-202` | **自动** | **1 次 / 编辑器会话**（实测坐实） |
| 工具栏生成 → `FUnrealCSharpEditorModule::Generator()` → `FCodeAnalysis::CodeAnalysis()` | `UnrealCSharpEditor.cpp:360` | 手动（生成） | 1 次 / 次生成 |
| 控制台命令 `UnrealCSharp.Editor.CodeAnalysis` → `FCodeAnalysis::CodeAnalysis()` | `UnrealCSharpEditor.cpp:103-109` | 手动 | 按需 |
| 🔴 `64` §2.3 ①(c) 记的"`:108`（启动路径）" | 同上 | — | **更正**：`:108` 在 `FAutoConsoleCommand` 的 lambda 内 |

### 3.2 单次成本实测（精确命令行；Development；同机同时段另有其它工程在构建）

| 趟 | 状态 | wall | MSBuild 自报 | 产物 SHA256 前 16 位（dll / exe） |
|---|---|---|---|---|
| 1–3 | **up-to-date**（无源改动） | **0.931 / 0.906 / 0.932 s** | 0.73 / 0.71 / 0.74 s | `CDBAD0E4037334DB` / `179CFDFA9F4F0F9B` |
| 4 | 触碰 `CodeAnalysis.cs` 后**重建** | **1.083 s** | 0.88 s | 同上（`<Deterministic>` ⇒ 逐字节相同） |
| 5 | 再次 up-to-date | 0.955 s | 0.74 s | 同上 |

⇒ **每次调用 ≈0.9–1.1 s 的同步阻塞**（`SyncProcess` 在调用线程上等待）；其中**绝大部分是"发现无事可做"的开销**。

### 3.3 每会话频次实测（进程采样装置；同日同时段环境噪声已逐条过滤）

| 会话 | 装置 | 读数 |
|---|---|---|
| Editor 会话（110 s，**改前**） | `b27-proc-sample.ps1 -Tag b27-editor` | **1 次**构建：`23:46:06.655` `"C:/Program Files/dotnet/dotnet.exe" build "D:/…/Script/CodeAnalysis/CodeAnalysis.csproj" --nologo -c Debug`（启动后约 9 s 处）；同会话另有 9 个与本次无关的 `dotnet` 进程（Rider 宿主 / UBT Scan / AutomationTool Turnkey），**已按命令行逐条排除** |
| Editor 会话（110 s，**改后 / up-to-date**） | `-Tag b27-editor-postgate` | **0 次** ⇒ 闸门生效（`SUMMARY … codeanalysis_build_launches=0`）；会话日志 `LogUnrealCSharp: Error` / `Assertion failed` / `Fatal error` / `Unhandled Exception` / `error CS` **全 0** |
| Editor 会话（110 s，**改后 / 正向对照**：先触碰 `Script/CodeAnalysis/CodeAnalysis.cs`） | `-Tag b27-editor-postgate-touch` | **1 次**构建（`23:52:09.878`），产物 `CodeAnalysis.{exe,dll}` `23:42:03 → 23:52:11` **确实重建** ⇒ 闸门不会"永久抑制" |

### 3.4 闸门实现（`FCodeAnalysis.cpp` `+78/−31` ＋ `FCodeAnalysis.h` `+4/−0`；**两个辅助函数 = 类内 `private static`**）

| 组 | 内容 |
|---|---|
| ① **类内私有静态**两个辅助 | `FCodeAnalysis::GetProgramPath()`（把 `Analysis()` 两处**三重复制**的 `CodeAnalysis[.exe]` 路径表达式收敛为一处）＋ `FCodeAnalysis::IsProgramOutdated()`（产物 vs 源的时间戳判据）—— **声明在 `FCodeAnalysis.h` 的 `private:` 段**（头文件 `+4/−0`），定义落在 `.cpp` 末尾（原四个定义顺序未动） |
| ② 闸门判据 | 产物 = `CodeAnalysis.exe`（Windows）**与** `CodeAnalysis.dll`（`const TArray<FString> ProductPaths{…}` ＋ 逐个 `FileExists` ＋ `GetTimeStamp < SourceTimestamp`）**都存在且都不早于源**；源 = `Script/CodeAnalysis/` 下的 `*.cs`、`*.csproj`（**`PlatformFile.IterateDirectory`** ＋ **`FPaths::GetExtension(InFilenameOrDirectory) == CSHARP_SUFFIX.RightChop(1)`**，逐字照抄 `FEditorListener.cpp:627-641` 的既有写法）**加** `Script/Shared.props`（存在才算）；任一产物缺失 ⇒ **不跳过**（fail-open 到"照旧构建"） |
| ③ 落点 | `Compile()` 起手包一层 `if (IsProgramOutdated()) { …原体重排一层… }` —— **单一出口、不新增提前 `return`**（合轮 B-17 规范；形态同 `FSolutionGenerator::CopyTemplate` 的 `if (!…) { … }`）；`Analysis()` 侧（真正跑工具）**未加闸门** |
| ④ 口径 | 零新增日志 / 注释 / `throw` / `try-catch` / 公开 API（**`FCodeAnalysis.h` 改 4 行：仅 `private:` 段新增两个 `static` 声明，`public:` 面逐字未变** ⇒ 公开 API 0 / ABI 0 —— 静态成员不进类布局）；两个辅助**不是**匿名 namespace —— 本仓**手写源码一律"类内静态函数"**（先例 `FCSharpCompilerRunnable.h:41 GetBuildConfiguration()`、`FAssetGenerator.h:12 GeneratorAsset()`、本类自身 `Compile()`/`Analysis()`；`FRegister*.cpp` 里的 `namespace {}` 是**生成物**，不构成手写规范）；判据形态照抄同仓 `IsScriptPublishOutdated()` 与 `FEditorListener.cpp:619-647` |

### 3.5 裁定

- **方案 A（时间戳闸门）成立并已施工**：`64` §2.3 ④ 的触发条件 ①（"先实测单次耗时"）**已完成**，读数 **0.9–1.1 s/次 × 1 次/会话**，**不属于"可忽略"**（对照同库两条被判"非热点"的条目：`F-RM-013` 9–23 ms、`F-MACRO-003` 0.26–0.42 ms）。
- **方案 B（用既有 `ReturnCode` 做控制流）与方案 C（转发）仍不做**：它们要的是"失败可见性"口径（`66` 的 Q2/Q3），且方案 B 会让失败**更静默**（`64` §2.3 ③ 已写明取舍）。
- **登记触发条件**：① 出现"非源码因素（NuGet / SDK / `obj` 状态）导致必要重建被闸门跳过"的实证 ⇒ 应把 `obj/` 或 `project.assets.json` 纳入源集合或回退闸门；② `Analysis()` 侧若也要提速（它每次都真跑工具）⇒ 另立条目，不在本条内。

---

## 4. 交付态验证

> 下表 = **提交树 `df487a6d` 的复验读数**（施工期读数与之一致；差额与四轮返工的重取见 §10.3 / §10.5）。

| 项 | 读数（提交树 `df487a6d`） |
|---|---|
| Windows UBT | ✅ **Succeeded**（`b27c-commit-ubt.log`；施工期另有 `b27-final-ubt.log` 等 5 趟，均 Succeeded；期间一次 `Unable to build while Live Coding is active` 为**编辑器未退净**导致，退净后重跑即过） |
| `dotnet build Script/Script.sln` | ✅ **exit 0 / 0 Warning / 0 Error**（30.69 s；分析器与 `Game.dll` 均重建，`b27c-commit-dotnet.log`；物化副本 SHA **`CEF8B168793D720043F8DCADD706280EB370A19978E347EB77C4476336F7DC52`**，与施工期末次**相同**） |
| 套件（提交树 ×2） | ✅ **1931 例 / 0 失败 ×2**，CSV SHA256 **`B429F3EE013B8C73FADEAF12E9BFA13C6B28C4FFE280312DE64F6843EB4386F8`**（两趟相同；`b27h-final`）—— 与 B-24 / B-25 / B-26 交付基线**逐字节相同** |
| 产物 manifest | ✅ **16 230 文件 / 0 行差异**（`products-b27-pre.txt` ↔ **`products-b27h-after.txt`**；四轮返工 ＋ 提交树复验各自的重取见 §10.3 / §10.5） |
| 日志 marker | ✅ `LogUnrealCSharp: Error` / `ensure` / `Assertion failed` / `Fatal error` / `Unhandled Exception` / `error CS` **全 0**（两趟套件 ＋ 六个采样会话） |
| 崩溃目录 | ✅ `Saved/Crashes` 今日新增 **0** |
| 判据装置复算 | ✅ driver：**`post4.txt`（提交树）↔ `post3.txt`（施工期）= 0 行差异**；生成器物化：`-Mode validate` = **True**（48 056 字符逐字符相同）；采样器各会话 `SUMMARY` 行齐备 |
| Android UBT（NDK clang） | ✅ **Succeeded**（`b27c-commit-android-ubt.log`；**控制项** —— 本轮改动 = 1 个 C# 编译期文件 ＋ 1 个 **Editor 模块** C++ 文件，Android 目标不编译 Editor 模块 ⇒ 该趟只证明"没有把它带进 Android 构建"，**不构成 Android 侧功能判据**；对照先例：轮 B-24 的 Android 判据力来自 `TFunctionPointer` 头改动，本轮无同类面） |

---

## 5. 未取得 / 存疑

1. **`F-CS4-009` 的"改前 CS8785 → 改后消失"未取得**：11 处 `?? throw` 对已解析树**静态不可达**（§2.4-1），22 例电池 `exc=0`；`64` §2.2 ⑤ 的判据 1（"若出现 CS8785 即命中本条"）在本机**无法构造**。替代判据 = 误报翻转（§2.3）＋ 真实 `dotnet build` 0 error。
2. **严格 `filePath == null` 不可构造**：`GetLocation().SourceTree` 对解析节点恒非空；"无路径"的可构造形态是 `FilePath == ""`。守卫用 `IsNullOrEmpty` 同时覆盖两者，但**只对空串取得了读数**。
3. **driver 的忠实度边界**：电池用**内存语法树**、程序集名取模板占位符 `GameNamePlaceholder`、不引用 `Script/Interop` 语义 ⇒ 只覆盖"语法级 ＋ 属性名匹配"路径；语义模型相关分支（`LibraryBridgeGenerator`、`GetUsingList` 等**未改动面**）不在覆盖内。
4. **闸门未测的面**：① 非源码因素（NuGet 包 / SDK 升级 / `obj` 被删）导致的本应重建；② 非 Windows 平台的产物名（`CodeAnalysis` 无后缀）；③ `Analysis()` 侧未加闸门，仍是"每次都跑工具"；④ 网络盘 / 时间戳精度差异（`FDateTime` 比较未在本机以外验证）。
5. **环境噪声如实声明**：采样期间同机有**别的 UE 工程**在构建（`E:\trunk\ProjectMecury` 的 UBT / UHT / Turnkey 进程），已按命令行过滤；`dotnet` 进程采样间隔 **300 ms**，对 0.9 s 量级的构建不会漏采，但**不能**排除更短命的进程。
6. 🟢 **两条改动均已提交** —— **`df487a6d132f451ff619014d3d0913d7ee9e4ce5`**（作者 `crazytuzi`；本轮 3 文件）；`git status` 现仅剩**既有子模块指针** `Source/SourceCodeGenerator`（本轮开工前即脏）。⚠️ **提交树 ≠ 施工期**（维护者 4 处纯形态调整，见 §10.5）⇒ 已**在提交树上重跑 §4 全套判据**，读数与施工期**逐条相同**。

---

## 6. 补丁与哈希

| 项 | 值 |
|---|---|
| 🟢 **本提交的组合补丁（引用这一份；= `cf27778f → df487a6d` 净差）** | `Saved/Retest/fix-df487a6d.patch`（**21 109 B** / SHA256 `7E46D2E4D37635A7848FF02FB6B74A56FF970BA7EB6C14F2E5156A175C9D5C7A` / **LF**（478 个 `\n`）/ **无 BOM** / `git apply --reverse --check`（对 `df487a6d` 树）**exit 0** / numstat **八行**：`32 22 …UnrealTypeSourceGenerator.cs`、`17 5 …FAssetGenerator.cpp`、`74 35 …Private/FCodeAnalysis.cpp`、`4 0 …Public/FCodeAnalysis.h`、`13 2 …FUnrealCSharpFunctionLibrary.cpp`、`7 1 …GeneratorScriptCodeCommandlet.cpp`、`21 0 …DynamicDataSource.cpp`、`2 0 …DynamicDataSource.h`） |
| 🟢 **本轮（B-27 部分）补丁（从提交树导出）** | `Saved/Retest/fix-B27-cs4-009-gen3-012-c719f1b4.patch`（**15 487 B** / SHA256 `06DE12A26A4650D3F3950B32E4FC89C1AA5A754CE018E4A0A64C9F459C97E801` / **LF** / **无 BOM** / `git apply --reverse --check`（对**提交树**）**exit 0** / numstat **三行**：`32 22 …UnrealTypeSourceGenerator.cs`、`74 35 …Private/FCodeAnalysis.cpp`、`4 0 …Public/FCodeAnalysis.h`） |
| C# 侧补丁（施工期，留档） | `Saved/Retest/fix-B27-cs4-009.patch`（**9 979 B** / SHA256 `B641B62FC73D6BE7762EFC3113E6CE4A66C8DB92F650F16E81B0E2788DF63BB2` / LF（179 个 `\n`）/ 无 BOM / numstat `32 22`）；工作区导出留档 `Saved/Retest/out/b27-driver/generator-patch.diff` |
| C++ 侧补丁（施工期，留档） | `Saved/Retest/fix-B27-gen3-012.patch`（**5 226 B** / SHA256 `A20032D57715123749BA2DA4CB9C88F6911D7B6BD00FCF7CC49B8B9F4354EB29` / LF（157 个 `\n`）/ 无 BOM / numstat 两行 `78 31` ＋ `4 0`）—— ⚠️ 与提交级补丁的差额 = 维护者 4 处纯形态调整（§10.5），**语义逐字等价** |
| 文件哈希（提交树） | 模板 `UnrealTypeSourceGenerator.cs`（插件侧，1 251 行 / blob `1209c185…`）；物化副本 `Script/SourceGenerator/UnrealTypeSourceGenerator.cs`（1 256 行 / SHA256 `CEF8B168793D720043F8DCADD706280EB370A19978E347EB77C4476336F7DC52`）；`FCodeAnalysis.cpp`（**128 行**（LF 归一后）／blob `e3b762b2…`／3 665 B；工作区 CRLF 3 793 B，**无 BOM**）；`FCodeAnalysis.h`（**18 行**（LF 归一后）／blob `62453437…`／301 B；工作区 CRLF 319 B，仅 `private:` 段新增 2 个 `static` 声明） |
| 判据装置（非交付物） | `Saved/Retest/b27-driver/{B27Driver.csproj,Program.cs}`；`Saved/Retest/b27-refresh-generator.ps1`；`Saved/Retest/b27-proc-sample.ps1`；读数归档 `Saved/Retest/out/b27-driver/{pre.txt,post.txt,post-v1.txt,post2.txt,generator-patch.diff}`、`Saved/Retest/out/b27-gate/proc-sample-b27-{editor,editor-postgate,editor-postgate-touch,rework-uptodate,rework-touch,static-uptodate,static-touch}.txt`、`Saved/Retest/out/{b27-dotnet-build.log,b27-codeanalysis-uptodate1..4.log,b27-codeanalysis-rebuild1.log,b27-ubt.log,b27-final-ubt.log,b27-rework-ubt.log,b27-rework-dotnet.log,b27-rework-android-ubt.log,b27-static-ubt.log,b27-static-android-ubt.log}`、`out/{b27-final,b27c-final,b27d-final,b27e-final}`、`out/g8-manifest/products-b27-{pre,final,c-after,d-after,e-after}.txt` |
| 备份 | `Saved/Retest/backup/UnrealTypeSourceGenerator.head-62afb1e1.cs`（= HEAD blob `b54e5a50…`，`git hash-object` 逐字节校验相同） |

---

## 7. 复用价值（给下一轮）

1. **"代码里有 `throw`" ≠ "输入能让它抛"** —— 判"必抛 / 会中断编译"必须先回答"哪条输入可达"。本轮 11 处 `?? throw` 的**形态**是真的、**后果**是不可达的；真正值钱的那条是**误报**（有实测翻转）。
2. **`Path.GetFileName(null)` 不抛，Roslyn 的 `FilePath` 默认是空串** —— "看起来会抛的 .NET API"要实测；守空必须用 `IsNullOrEmpty` 而不是 `!= null`，否则**改完误报还在**。
3. **误报型诊断有"二阶后果"** —— interface 那处误报把 `hasError` 置真，连带**抑制生成产物**（gen 0 → 2）；只看到"多了一条红字"会低估它。
4. **判性能条目要"单次成本 × 每会话次数"，缺一不可** —— 单看 0.93 s 会与 B-26 的 0.26 ms 混为一谈；补上"1 次/会话"（进程采样）才得出"比被判非热点的两条大 40–3500×"。采样器是**非侵入**装置，不需要改一行被测代码。
5. **生成器的物化副本是"产物"** —— 改模板后若只 `Refresh` 不**自证变换忠实**，`dotnet build` 会在**旧分析器**上跑出假绿；本轮用"HEAD 模板 → 现副本逐字符相同"把这一步变成可审计判据。
6. **一次会话就能证伪一个前提** —— `64` §2.3 ①(c) 的"`:108`（启动路径）"在读一眼 `FAutoConsoleCommand` 上下文后就翻掉了；审计材料里的"调用链"必须在施工前重取（同 `64` §5-5 的告诫）。

---

## 8. 附：本轮时间线（供审计追溯）

| 时刻 | 动作 | 读数 |
|---|---|---|
| 23:37 | 开工：插件仓 `git status` = 仅子模块指针；备份 HEAD 生成器（blob `b54e5a50…`） | HEAD（轮 B-26 交付态） |
| 23:39–23:41 | driver 建成（先两次空转：程序集名占位符 ＋ 用例命名约定未对齐 ⇒ 修正后 22 例全部产出读数）→ `pre.txt` | 3 处误报 ＋ 2 个探针抛异常；`exc=0` ×22 |
| 23:41 | C# 施工 → `-Mode validate` 自证变换忠实（True）→ refresh 物化副本 → `dotnet build Script.sln` | 0 Warning / 0 Error；`SourceGenerator.dll` 23:41:27、`Game.dll` 23:41:44 |
| 23:42–23:44 | CodeAnalysis 构建计时 ×5 | up-to-date 0.931 / 0.906 / 0.932 s；重建 1.083 s；up-to-date 0.955 s |
| 23:42–23:44 | 套件 ×2 ＋ manifest（C# 侧改动） | 1931 / 0 ×2，CSV `B429F3EE…`；manifest 16 230 / 0 差异 |
| 23:45–23:48 | 进程采样（改前）→ C++ 闸门施工 → UBT | **1 次** CodeAnalysis 构建（23:46:06）；UBT Succeeded（37 s） |
| 23:49–23:53 | 采样（改后 up-to-date）→ 触碰源 → 采样（正向对照） | **0 次** / **1 次**（产物 23:42:03 → 23:52:11） |
| 23:54–23:56 | CRLF 归一（`git diff --numstat` 不变）→ UBT 重跑 | Succeeded（14.78 s） |
| 23:57– | 交付态套件 ×2 ＋ manifest（`b27c-*`） | **1931 / 0 ×2**，CSV `B429F3EE…`（两趟相同）；manifest **16 230 / 0 差异**；Windows UBT 与 Android UBT（控制项）均 Succeeded |
| 00:19–00:21 | **规范返工（风格）**：遍历 API / 后缀判定 / 闸门形态 ＋ C# `!string.IsNullOrEmpty` → 重取 driver / UBT / 闸门 A/B | `post2` ↔ `post-v1` **0 行**；up-to-date **0 次** / 触碰 **1 次** |
| 00:32–00:38 | **类静态返工**：去匿名 namespace（两辅助声明进 `FCodeAnalysis.h` 的 `private:` 段）→ UBT / 闸门 A/B / Android / 套件 ×2 / manifest | 0 次 / 1 次；`b27e-final` 同 CSV；manifest **0 差异** |
| 00:5x–01:0x | **命名返工**：`IsProgramOutdated()` ＋ `RemoveTypePrefix()`（`GetProgramPath()` 保留）→ 重取 driver / `dotnet build` / UBT / 闸门 A/B | `post3` ↔ `post2` **0 行**；⚠️ **闸门 A/B 反向**（新鲜 1 次 / 陈旧 0 次）⇒ 抓出"改名未翻函数体"的语义反转缺陷（§10.4） |
| 01:08–01:12 | **修正 `IsProgramOutdated()` 的返回值语义** → UBT → 三会话 A/B/C | 陈旧 **1 次**（`01:08:27` 构建、产物 `01:08:28` 重建）/ 新鲜 **0 次** / 触碰 **1 次**（`01:12:01` 构建、产物 `01:12:02` 重建） |
| 01:14–01:16 | **最终树**交付态：套件 ×2 ＋ manifest ＋ Android UBT（控制项） | **1931 / 0 ×2**，CSV `B429F3EE…`（同 SHA）；manifest **16 230 / 0 差异**；Android UBT **Succeeded** |
| **10:50** | 🟢 **维护者提交 `df487a6d`**（3 文件；含 4 处纯形态调整，见 §10.5） | numstat：`32/22`、`74/35`、`4/0` |
| 10:12–10:20 | **提交树复验**：物化副本重刷 ＋ `dotnet build` → driver 电池 → UBT → 闸门三会话 → Android UBT → 套件 ×2 ＋ manifest → 提交级补丁 | 副本 SHA `CEF8B168…`（同）；`post4` ↔ `post3` **0 行**；UBT Succeeded；**0 / 1（产物 `10:14:33` 重建）/ 0**；Android Succeeded；套件 **1931 / 0 ×2**（同 CSV）；manifest **0 差异**；补丁 **15 487 B** / `06DE12A2…` |

---

## 9. 拍板回执（2026-10-15，本轮收尾后；**0 代码改动**）

> **回执原句**（维护者）："**Q1 确认 C；Q2 维持＋例外；Q3 先固化判据**"。回执页 = [`08-…/66`](66-待决策页-三问（Q1四路线·Q2口径例外·Q3失败判据）（2026-10-15）.md)（拍板回执表**已填写并归档**，本页不再是"待决策"）。

| 问 | 决定 | 与推荐答案 | 当场效果 |
|---|---|---|---|
| **Q1** | **C（维持现状）** | 一致 | `F-INT1-004`（P2 · **3.35** · T2 最高未修）**出列**：归档"**不适用（裁定不做）**"，**事实四项逐字保留 ＋ 三条触发条件、不写"已修"**（风险对称性护栏 = `64` §3.2①）；0 代码改动 |
| **Q2** | **维持口径 ＋ 例外** | 一致 | 解锁：`F-CS1-040`（"用**既有取值域**表达失败"那一半 **可开工**）、`F-DOM-020` 残余（并入"失败信号口径"一次性裁定，**本体不施工**）、死代码批次 C 的 4 组（`F-DEAD-026/027/028/029` 的"**补实现**"半边；"补 `UE_LOG`/`ensure`"半边**仍属开口、不解锁**）、`F-GEN3-012` 的开口半边（用既有 `ReturnCode` 做控制流）；白名单 = `nint` 哨兵 / `SyncProcess::ReturnCode` / 进程退出码 / Roslyn 既有诊断 / `TOut::OutLength`，**不含 `try-catch`** |
| **Q3** | **先固化判据** | 一致 | `F-ED2-021` 方案 B 的**前置固化**：判据 = **产物 manifest 计数/内容差异 ＋ `Generator` 内部返回值**（**不得**用日志行数）；余下只欠"CI 侧确认以该判据为准" |

**本轮据此完成的台账回写（全部 0 代码改动）**：`00-推荐优先修复清单.md` 的 T2 表第 6 / 24 / 27 / 28 行 ＋ §0.1 批次 C 行 ＋ §0.1 施工纪律注第 ⑤ 条 ＋ 新增"**三问拍板**"更新注；`01-全局问题总表.md` 的 `F-INT1-004` / `F-DOM-020` / `F-ED2-021` / `F-CS1-040` / `F-GEN3-012` **五行**；`05-已完成（P2与P3）.md` §18.2 的 `F-INT1-004` 行（＋ 产品侧计数订正 **8 925/87 670 → 8 927/87 675**）；`README.md` 新增"三问拍板"**裁定行**；`INDEX.md` 已含 66/67 两份新报告。

> 📌 **拍板后的排期视图**：T2 表内"**干净开放面**"由 **8 行降到 7 行**（余：`F-GEN-015` 部分修／`F-DOM-020` 残余／`F-FL-014`／`F-CS1-040`／`F-ABI-035` 的"无同步"半边／`F-GEN-016`／`F-CS2B-008` 部分修）。**下一轮建议**：① `F-CS1-040`（S=3.08）—— Q2 后**前置已齐**，判据 = "注册整体失败（`InLength == 0` / 函数指针 `nullptr`）可区分" ＋ 套件零副作用；② `F-FL-014`（S=3.08）—— 先做"使用面 vs 受益面"普查再定施工或登记；③ `F-GEN-016`（S=3.01）—— 可用本轮同一"**先量化**"手法（单次成本 × 每会话次数）。

---

## 10. 规范返工（2026-10-16 凌晨，本轮收尾后；**0 语义改动**）

> **来源**：维护者复审意见 —— "**再来看你的两个本地代码改动文件，看起来不符合项目规范和代码风格啊，然后再看下要不要代码格式化**"。

### 10.1 先回答"要不要代码格式化"：**不跑全量格式化**

| 检查 | 读数 |
|---|---|
| 插件仓格式化配置 | **无** `.clang-format` / `.editorconfig` / `.gitattributes`（仓库根仅 `.gitignore`、`.gitmodules`、`LICENSE`、`README.md`、`.uplugin`） |
| 引擎侧 `.clang-format` | `D:\file\UnrealEngine\5.6\.clang-format` 与 `Engine\.clang-format` **均不存在** |
| 工程侧 `Script/` | 无 `.editorconfig` |
| 工具可用性 | `clang-format.exe`（VS LLVM）存在；`dotnet format` 随 SDK 可用 —— 但**没有可依的配置** |

⇒ **结论**：既无基线可依，全量格式化只会把**整文件**按工具默认风格重排（与文件既有风格不一致的无关 diff），不符合本项目"**最小改动 ＋ 逐行可审**"的施工纪律。**改为手工对齐同仓既有先例**（本轮实际偏差见 10.2）。

### 10.2 五处偏离与修正（**C++ 侧 4 处 ＋ C# 侧 1 处**；第 1 条是维护者点出的主项）

| # | 第一版写法（返工前） | 返工后（同仓先例） | 先例出处 |
|---|---|---|---|
| **1** | 🔴 **匿名 `namespace { … }` 里放两个文件级辅助函数** | **改为"类内 `private static`"**：`FCodeAnalysis::GetProgramPath()` ＋ `FCodeAnalysis::IsProgramOutdated()`，**声明写进 `FCodeAnalysis.h` 的 `private:` 段**（`+4/−0`），定义落在 `.cpp` 末尾（原四个定义顺序未动） | **本仓手写源码一律"类内静态函数"，没有匿名 namespace** —— 先例 `FCSharpCompilerRunnable.h:41 GetBuildConfiguration()`、`FAssetGenerator.h:12 GeneratorAsset()`、本类自身 `Compile()` / `Analysis()`。⚠️ **第一版误判的来源**：grep 到的 `namespace { }` **全部**落在 `FRegister*.cpp` —— 那是**55 个生成物文件**（生成器写出的注册桩），拿生成物的形态当手写规范用，方向一开始就错了。📌 **同款先例早在台账里**：[`../09-问题清单/01-全局问题总表.md`](../09-问题清单/01-全局问题总表.md) 的 2026-10-07 `F-ED2-005` 轮原文即写"**按项目惯例落在类内静态成员**"（`FEditorListener.h` 的静态成员段落）⇒ 这条规范**不是我该推的、而是仓里早就写着的** |
| 2 | `IFileManager::Get().IterateDirectory(...)` | **`PlatformFile.IterateDirectory(...)`** | `FEditorListener.cpp:627` 的 `PlatformFile.IterateDirectoryRecursively(...)` |
| 3 | `FString(Filename).EndsWith(CSHARP_SUFFIX)`；lambda `[&SourceTimestamp, &PlatformFile](const TCHAR* Filename, const bool bIsDirectory)` | **`FPaths::GetExtension(InFilenameOrDirectory) == CSHARP_SUFFIX.RightChop(1)`**；lambda **`[&](const TCHAR* InFilenameOrDirectory, const bool bIsDirectory) -> bool`**（逐字对齐） | `FEditorListener.cpp:629-632` |
| 4 | 产物判定 = `FMath::Min(两个产物)` 与源比；`Compile()` 起手 **提前 `return;`** | 产物 = **`const TArray<FString> ProductPaths{…}` ＋ 逐个 `FileExists` / `GetTimeStamp < SourceTimestamp`**；闸门 **包一层 `if (IsProgramOutdated()) { … }`**（**单一出口、不新增提前 `return`**，合轮 B-17 规范） | `IsScriptPublishOutdated()`（`FUnrealCSharpFunctionLibrary.cpp:1314-1326`）；`FSolutionGenerator::CopyTemplate` 的 `if (!…) { … }` 形态 |
| 5 | `if (string.IsNullOrEmpty(filePath) == false && …)`（3 处）；多留一个 `#include "HAL/FileManager.h"` | **`if (!string.IsNullOrEmpty(filePath) && …)`**；include **删除**，只留 `HAL/PlatformFileManager.h` ＋ `Misc/Paths.h` | `Script/` 内 4 处先例：`MethodBridge.cs:29`、`TypeBridge.cs:21` / `:57`、`Utils.cs:307`；`FEditorListener.cpp:10` |

### 10.3 返工后**重取**的全部判据（结论：行为与零副作用读数**逐条不变**；第四次（命名 ＋ 反转缺陷修正）= 最终树，见 §10.4）

| 判据 | 第一次（返工前） | 第二次（风格返工后） | 第三次（类静态版；⚠️ 该版**含 §10.4 的反转缺陷**） | 第四次（**命名返工 ＋ 修正 = 最终树**） |
|---|---|---|---|---|
| driver 电池 22 例 ＋ 4 探针 | `post.txt` | **`post2.txt` ↔ `post-v1.txt` = 0 行** | — | **`post3.txt` ↔ `post2.txt` = 0 行**（C# 改名零语义漂移） |
| `dotnet build Script/Script.sln` | 0 / 0（19.18 s；副本 `981EB6CD…`） | 0 / 0（30.96 s；副本 `787062F9…`） | — | **0 / 0**（29.76 s；副本 SHA **`CEF8B168793D720043F8DCADD706280EB370A19978E347EB77C4476336F7DC52`**） |
| Windows UBT | Succeeded | Succeeded（`b27-rework-ubt.log`） | Succeeded（`b27-static-ubt.log`） | **Succeeded**（`b27-name-ubt.log` ＋ 修正后 `b27-fix-ubt.log`） |
| 闸门 A/B（进程采样，各 100 s） | 0 次 / 1 次 | 0 次 / 1 次 | 0 次 / 1 次 | ⚠️ 反转态 **1 / 0**（**与期望相反**，见 §10.4）→ 修正后三会话：**陈旧 1 次（产物重建）/ 新鲜 0 次 / 触碰 1 次（产物重建）** |
| Android UBT（控制项） | Succeeded | Succeeded | Succeeded | **Succeeded**（`b27-fix-android-ubt.log`） |
| 套件（最终树 ×2） | 1931 / 0 ×2，CSV `B429F3EE…` | 同（`b27d-final`） | 同（`b27e-final`） | **1931 / 0 ×2**，CSV **`B429F3EE013B8C73FADEAF12E9BFA13C6B28C4FFE280312DE64F6843EB4386F8`**（`b27g-final`，两趟相同、与基线**同 SHA**） |
| 产物 manifest | 16 230 / 0 差异 | 16 230 / 0（`b27d-after`） | 16 230 / 0（`b27e-after`） | **16 230 / 0 差异**（`products-b27-pre` ↔ **`products-b27g-after`**） |
| 改动面 / 补丁哈希 | C# `+32/−22`；C++ `+71/−22`；补丁 `10 023 B` / `BA1A225D…`、`4 377 B` / `9AD637A2…` | C++ `+81/−31`；补丁 `4 855 B` / `BD6EE8EC…` | 补丁 `9 999 B` / `00DB3EB7…`、`5 227 B` / `42AF8573…` | **C# `+32/−22`（未变）；C++ `+78/−31`（`.cpp`）＋ `+4/−0`（`.h`）⇒ 插件共 3 文件**；补丁 **`9 979 B` / `B641B62F…`**、**`5 226 B` / `A20032D5…`**（均 LF / 无 BOM / `git apply --reverse --check` **exit 0**） |

### 10.4 命名返工（第四轮，2026-10-16 凌晨；维护者复审"还有没有更好的命名"）—— ⚠️ **并抓出一个真缺陷**

| 项 | 决定 | 依据（本仓实读） |
|---|---|---|
| `IsCompileUpToDate()` | → **`IsProgramOutdated()`** | 全仓手写源码里 **`Outdated` 6 处**（`IsScriptPublishOutdated()` ＋ 它的 **4 处正向调用点**、`FUnrealCSharpCoreModule::IsOutdated()`/`MarkOutdated()`/`EState::Outdated`）而 **`UpToDate` 0 处** ⇒ 陈旧性判据的仓内极性是"过期"；改名后调用点写成正向 `if (IsProgramOutdated())`，与既有 4 处同形。备选 `IsCompileOutdated()`（与 `Compile()` 成对，读起来略绕）／`IsCompileRequired()`（与 `FEditorListener` 撞名）未采纳 |
| `GetProgramPath()` | **保留** | 它取的就是 `Analysis()` 里的局部变量 `Program`；同仓 `GetCodeAnalysisProjectPath()` / `GetCodeAnalysisPath()`（后者是**另一个东西**：中间目录）不能借名。备选 `GetProgramFilePath()`（多一个词）／`GetToolPath()`（`CompileTool` 指的是 dotnet CLI，不是被测工具）未采纳 |
| `GetNameWithoutPrefix()` | → **`RemoveTypePrefix()`** | `Script/` 的删除家族是 **`Remove*`**（`Class.RemoveFunction`、`Object.RemoveFromRoot`/`RemoveReference`、`TArray.RemoveAt`/`Remove`、`TMap.Remove`、`InputComponent.RemoveAction`…），且原名既不点明"类型"、语感也不合仓内（维护者对 "Without" 的直觉）。备选 `TrimTypePrefix()`（仓内无 `Trim*` 先例）／`GetUnprefixedName()`（没点出"类型"）／`GetNameAfterTypePrefix()`（偏长）／`StripTypePrefix()`（无 `Strip*` 先例）未采纳 |

> 🔴 **改名带出的真缺陷（本轮最有价值的一条）**：把 `IsCompileUpToDate()` 改名成 `IsProgramOutdated()` 时，**只翻了调用点的 `!`、没有翻函数体的返回值** ⇒ 语义**整体反转**（新鲜时判"过期"、陈旧时判"新鲜"）。它被本轮的**重取判据**当场抓住 —— 反转态的两组读数**恰好与期望相反**：
>
> | 会话（反转态） | 期望 | 实测 | 痕迹 |
> |---|---|---|---|
> | 新鲜态（产物 `00:34:21` > 源 `00:34:09`） | 0 次 | **1 次**（`b27-name-uptodate`） | 那次构建因 MSBuild 增量无改动，**产物 mtime 没变**（仍是 `00:34:21`）—— 正是这个"跑了但什么都没改"的痕迹暴露了它 |
> | 陈旧态（源 `01:02:11` > 产物 `00:34:21`） | 1 次 | **0 次**（`b27-name-touch`） | 产物停在 `00:34:21`，而源已被触碰 —— "改了源却不重建"是不可接受的形态 |
>
> **修正后重取（三会话 A/B/C，最终树）**：
>
> | 会话 | 状态 | 期望 | 实测 |
> |---|---|---|---|
> | `b27-fix-stale` | 陈旧（源 `01:02:11` > 产物 `00:34:21`） | 1 次 | **1 次**（`01:08:27` 构建；产物 `00:34:21 → 01:08:28` **重建**） |
> | `b27-fix-fresh` | 新鲜（产物 `01:08:28` > 源 `01:02:11`） | 0 次 | **0 次** ⇒ 闸门确实省掉了那次 `dotnet build` |
> | `b27-fix-touch` | 触碰源（`01:11:51`） | 1 次 | **1 次**（`01:12:01` 构建；产物 → `01:12:02` **重建**） |
>
> ⇒ **教训（本轮新增纪律）**：**带 `!` 的判据上"改名"＝改语义** —— 凡把 `IsXxxUpToDate()` 改成 `IsXxxOutdated()`（或反向），必须**同时**翻函数体 `return`、翻调用点，并**重取 A/B**。本轮若省掉这次重取，就会带着一个"永远反着判"的闸门交付：**改源码不重建、没改却每次启动都跑一遍 `dotnet build`**。

### 10.5 提交树复验（提交 `df487a6d` 之后；**含维护者的 4 处纯形态调整**）

> 🟢 **提交**：`df487a6d132f451ff619014d3d0913d7ee9e4ce5`（作者 `crazytuzi <liuxiangcode@qq.com>`）；**本轮 3 文件**，numstat = C# `32/22`、`FCodeAnalysis.cpp` **`74/35`**、`FCodeAnalysis.h` `4/0`。`git status` 现仅剩既有子模块指针 `Source/SourceCodeGenerator`。

**提交树 ≠ 施工期工作区** —— 维护者的 4 处**纯形态**调整（语义逐字等价）：

| # | 施工期（我验证过的形态） | 提交树（维护者调整后） | 为什么等价 |
|---|---|---|---|
| 1 | 两个 `Analysis()` 重载各取一个局部量 `const auto Program = GetProgramPath();`，再 `SyncProcess(Program, …)` | **去掉局部量**，直接把 `GetProgramPath()` 内联进 `SyncProcess(GetProgramPath(), …)`（并删掉随之多余的空行） | 同一函数调用、同一实参；少一次命名绑定（也是 `.cpp` numstat `78/31 → 74/35` 的来源） |
| 2 | `IsProgramOutdated()` 内局部量：`SourceTimestamp` / `SharedPropsPath` / `ProductPaths` / 循环变量 `ProductPath` | 改名为 **`Timestamp` / `SharedProps` / `Paths` / `Path`** | 纯改名（同一作用域内无歧义） |
| 3 | lambda 内两段式：先 `const auto Extension = FPaths::GetExtension(…);`，再 `if (!bIsDirectory && (…))` | 合并为 **C++17 if-init 形态**：`if (const auto Extension = FPaths::GetExtension(InFilenameOrDirectory); !bIsDirectory && (…))`（并把 `!bIsDirectory` 提到最前） | `Extension` 仍**每个条目**都求值一次（if-init 的初始化语句无条件执行）⇒ 求值次数与短路行为**不变** |
| 4 | `FPaths::Combine(…, CODE_ANALYSIS_NAME + DLL_SUFFIX)` 折成两行 | 收回一行 | 纯排版 |

**提交树复验读数**（与施工期**逐条相同**）：

| 判据 | 提交树读数 |
|---|---|
| driver 电池（22 例 ＋ 4 探针） | **`post4.txt`（提交树）↔ `post3.txt`（施工期）= 0 行差异**（C# 侧提交树与施工期 blob 相同：`1209c185…`） |
| `dotnet build Script/Script.sln` | **exit 0 / 0 Warning / 0 Error**（30.69 s）；物化副本（从**提交树**模板重刷）SHA **`CEF8B168…`** —— 与施工期末次**相同** |
| Windows UBT | **Succeeded**（`b27c-commit-ubt.log`） |
| 闸门三会话 | `b27c-fresh1` **0 次**（10:12）／`b27c-touch` **1 次**（`10:14:32` 构建；产物 `01:12:02 → 10:14:33` **重建**）／`b27c-fresh2` **0 次**（10:16） |
| Android UBT（控制项） | **Succeeded**（`b27c-commit-android-ubt.log`） |
| 套件（提交树 ×2） | **1931 / 0 ×2**，CSV **`B429F3EE013B8C73FADEAF12E9BFA13C6B28C4FFE280312DE64F6843EB4386F8`**（`b27h-final`，两趟相同、与基线**同 SHA**） |
| 产物 manifest | **16 230 / 0 差异**（`products-b27-pre` ↔ `products-b27h-after`） |
| 提交级补丁 | `Saved/Retest/fix-B27-cs4-009-gen3-012-c719f1b4.patch`（**15 487 B** / SHA256 `06DE12A26A4650D3F3950B32E4FC89C1AA5A754CE018E4A0A64C9F459C97E801` / LF / 无 BOM / 对提交树 `git apply --reverse --check` **exit 0**） |

> ⚠️ **本节的读数时钟**：四轮返工与重取判据发生在 **2026-10-16 00:0x–01:1x**；**提交树的复验**（§10.5）发生在**同日 10:12–10:20**（提交本身 = 10:09:49）；本轮主体工作与 §0–§9 的读数在 **2026-10-15 23:37–23:59**；报告按**轮次起始日**命名为 2026-10-15。
> 📌 **留给下一轮的三条纪律**：① **施工前先把同仓先例找齐再动手** —— C++ 侧的四个偏离在同一仓里**都有逐字可抄的先例**（`FEditorListener.cpp:619-647`、`IsScriptPublishOutdated()`、`FSolutionGenerator::CopyTemplate`、头文件里成排的 `private: static`），第一版靠"自己写干净"反而引入了风格漂移。② **"生成物不算规范"** —— 本仓 **55 个 `FRegister*.cpp`** 是生成器写出的（成排 `namespace { }` / `BINDING_*` 属**产物形态**）；判"本仓手写规范"时必须先把生成物与手写源码分开（本轮第一版正是踩在此处，被维护者一眼看出）。③ **带 `!` 的判据上"改名"＝改语义** —— 把 `IsXxxUpToDate()` 改成 `IsXxxOutdated()`（或反向）必须**同时**翻函数体 `return` ＋ 翻调用点 ＋ **重取 A/B**（§10.4 的反转缺陷就是这么来的、也是这么被抓回来的）。⇒ **"零语义漂移"是返工后才被证明的（driver 0 行差异），不是设计时就知道的**。
