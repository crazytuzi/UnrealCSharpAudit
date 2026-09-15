# UnrealCSharp ScriptCodeGenerator —— 代码分析 / Doxygen 转换 / 解决方案生成

> **核对对象**：插件 git HEAD `1b29b6ed`，工作区干净。18 条发现逐条回源码核对：**确认 13 条（F-GEN3-001/002/003/005/006/007/008/011/012/015/016/017/018）、部分确认 5 条（F-GEN3-004/009/010/013/014）、证伪 0 条（无整条撤销）、无法验证 0 条**，合计 18 条。（计数以各条 `复核结论` 字段逐条聚合为准：头部曾误写“部分确认 4 条”并把 `F-GEN3-017` 列入该组，而 `017` 的字段实为“✅确认”，实有 5 条为“部分确认”，此处按逐条字段为准。）
> **没有任何级别变动**（18 条全部维持）：5 处调整（001/002/004 P1→P2、013/016 P2→P3）**成立并保留**。
> 已修正的**事实性错误**（概率算错、路径笔误、模块依赖归因错误、grep 命中数不可复现、调用链首跳写错、标题与正文矛盾）逐条见各 Finding 的「复核结论」字段。
> **发现编号顺序异常（已确认，不重排编号）**：正文物理顺序为 001-010、012、013、015、016、011、014、017、018，即 F-GEN3-011 / F-GEN3-014 排在 F-GEN3-016 之后；且 `### P1` 小节内含 3 条 P2（001/002/004），`### P2` 小节内含 2 条 P3（013/016）。编号是跨报告引用锚点，故**仅就地加注、不移动、不重编**。





> 分析范围：`Plugins/UnrealCSharp/Source/ScriptCodeGenerator`（分析层与产出层：`FCodeAnalysis`、`FDoxygenConverter`、`FSolutionGenerator`、`FAssetGenerator`、`ScriptCodeGenerator.Build.cs`、`ScriptCodeGeneratorMacro.h`）+ 触发上下文 `UnrealCSharpEditor/Private/Commandlet/GeneratorScriptCodeCommandlet.cpp`
> 覆盖文件：见 §0（本报告只覆盖上列文件；`FGeneratorCore.cpp`、`FClassGenerator.cpp` 等仅作为调用上下文读取相关片段）

> 状态：已完成（本组 10 个文件全部读完；`UnrealCSharpEditor`/Core 侧仅读与结论直接相关的片段，见 §5）
> 发现数（最终值，逐条统计自各条 `严重度` 字段）：**P0 0 / P1 1 / P2 11 / P3 6**（共 18 条，F-GEN3-001 … F-GEN3-018 无缺号、无重号）。
> 此前此行曾写 "P0 0 / P1 4 / P2 10 / P3 4"，与各条实际字段值不符（应为 P1 1 = `F-GEN3-003`、P2 11、P3 6），已按字段值改正。5 处级别调整（001/002/004 由 P1 降 P2；013/016 由 P2 降 P3）**全部维持**。

## 0. 覆盖范围与阅读清单

> **行数口径**：下表行数均为**总行数**（含空行，即 `(Get-Content $f).Count` 口径）。逐文件用 `read` 回读核对，**21 行（10 个本组源文件 + 11 个上下文/模板条目）的行数全部逐字吻合**（例：`FDoxygenConverter.cpp` 475 行 = `read` 的 count；另有"非空行数"口径 `Measure-Object -Line` 会忽略空行，两者数值不同**不算错误**）。任务书标注的行数（358/410/169）为旧口径，已按下表为准。
> **路径口径**：下表所有相对路径均相对**插件根** `Plugins/UnrealCSharp/`；模板文件位于插件根的 `Template/`（经 `GetPluginTemplateDirectory()` = 插件目录 / `Template`，见 `FUnrealCSharpFunctionLibrary.cpp:919-922` 与 `Macro.h:7`），**不是** `Script/Template/`。

| 文件 | 行数 | 是否读完 | 备注 |
|---|---|---|---|
| `Source/ScriptCodeGenerator/Public/FCodeAnalysis.h` | 14 | 读完 | 全 14 行 |
| `Source/ScriptCodeGenerator/Private/FCodeAnalysis.cpp` | 89 | 读完 | 全 89 行 |
| `Source/ScriptCodeGenerator/Public/FDoxygenConverter.h` | 14 | 读完 | 全 14 行 |
| `Source/ScriptCodeGenerator/Private/FDoxygenConverter.cpp` | 475 | 读完 | 全 475 行（注：任务书标注 358 行，实际以 read 的 475 行为准） |
| `Source/ScriptCodeGenerator/Public/FSolutionGenerator.h` | 52 | 读完 | 全 52 行 |
| `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp` | 467 | 读完 | 全 467 行（任务书标注 410 行） |
| `Source/ScriptCodeGenerator/Public/FAssetGenerator.h` | 17 | 读完 | 全 17 行 |
| `Source/ScriptCodeGenerator/Private/FAssetGenerator.cpp` | 192 | 读完 | 全 192 行（任务书标注 169 行） |
| `Source/ScriptCodeGenerator/Public/ScriptCodeGeneratorMacro.h` | 3 | 读完 | 仅 `ENUM_DEFAULT_MAX_INDEX` |
| `Source/ScriptCodeGenerator/ScriptCodeGenerator.Build.cs` | 60 | 读完 | 全 60 行 |
| 上下文：`UnrealCSharpEditor/Private/Commandlet/GeneratorScriptCodeCommandlet.cpp` | 42 | 读完 | 全 42 行，真实触发入口 |
| 上下文：`UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp` | 397 | 部分 | 读 70-199、305-397（模块启动、`Generator()` 全流程、控制台命令） |
| 上下文：`UnrealCSharpEditor/Private/Listener/FEditorListener.cpp` | 755 | 部分 | 读 160-259、340-409、460-569（增量生成、资产增删改、`EndGenerator(false)`） |
| 上下文：`UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp` | 1615 | 部分 | 读 655-834 / 896-955 / 960-1119 / 1120-1249（资产→文件名映射、路径、`SaveStringToFile`） |
| 上下文：`UnrealCSharpCore/Public/Setting/UnrealCSharpSetting.h` | 185 | 读完 | `FCustomProject::GUID()` 在此 |
| 上下文：`UnrealCSharpCore/Public/CoreMacro/Macro.h` | 133 | 读完 | 占位符/GUID/后缀常量 |
| 上下文：`UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp` | — | 部分 | 仅取关键默认值（26/29/32/36/37 行） |
| 上下文：`Template/{Script.sln,Game.csproj,Game.props,Shared.props,Interop.csproj,UE.csproj}` | 54/6/29/22/23/9 | 读完 | 逐行核对占位符一致性 + 逐字节核对换行/BOM |
| 上下文：`ScriptCodeGenerator/Private/FGameplayTagGenerator.cpp` | 282 | 部分 | 读 110-169（XML 转义与排序的正面先例） |
| 上下文：`ScriptCodeGenerator/Private/FGeneratorCore.cpp` | 1079 | 部分 | 读 960-1079（`BeginGenerator`/`EndGenerator`/`DeleteRemainGeneratorFiles`） |
| 上下文：`ScriptCodeGenerator/Private/FClassGenerator.cpp` | 1450 | 部分 | 读 400-529、1095-1129（Doxygen 调用点、`ENUM_DEFAULT_MAX_INDEX` 使用点） |

> 说明：本报告对 `FUnrealCSharpFunctionLibrary` / `UUnrealCSharpSetting` 的发现仅限"被本组文件直接调用且决定本组正确性"的部分，避免与 Core 模块报告重复。

## 1. 模块职责与架构速览

`ScriptCodeGenerator` 是 **Editor 类型模块**（`UnrealCSharp.uplugin`，`Source/ScriptCodeGenerator`），对外只导出一个真正的入口 `FUnrealCSharpEditorModule::Generator()`（在编辑器模块里，见下），本模块自身通过 4 个 public 导出点被调用：`FCodeAnalysis::CodeAnalysis()/Analysis(const FString&)`、`FSolutionGenerator::Generator()`、`FAssetGenerator::Generator()/Generator(const FAssetData&, bool)`、`FGeneratorCore::BeginGenerator()/EndGenerator()`。

四类职责与实现方式（**关键：本模块几乎没有"文本/词法分析"，真正的源码解析在 C# 侧**）：

| 组件 | 实际做什么 | 实现方式 |
|---|---|---|
| `FCodeAnalysis` | **不是** C++ 侧的文本分析器，而是外部 .NET 分析器进程的启动器：先 `dotnet build Script/CodeAnalysis/CodeAnalysis.csproj -c Debug`，再用 `SyncProcess` 跑 `CodeAnalysis.exe` 扫描工程目录 | 进程调用（`FUnrealCSharpFunctionLibrary::SyncProcess`），回调为空 lambda |
| `FDoxygenConverter` | 真正的"文本处理"组件：**手写词法分析器**（`FTextReader` + `Lex` + `Tokenize`），把 UHT `Comment` 元数据（Doxygen 风格）转成 C# `///` XML 文档注释 | 自建 token 流，非正则（但 `Lex` 里有跨行吞噬缺陷，见 F-GEN3-006） |
| `FSolutionGenerator` | 把插件自带的 `Script/`、`Template/` 工程文件复制/改写到用户工程的 `Script/` 下，生成 `.sln`、`.csproj`、`.props` | **字符串精确替换**（15 个 `Replace*` 函数 + 3 个 `Add*` 函数，共 **19 处** `FString::Replace` 调用：18 处作用于 `OutResult`，`:464` 作用于 `GetGeneratorHeaderComment()` 的返回值。原文此处写"13 处"，grep 重跑为 19 处）+ `FFileHelper` 读写 |
| `FAssetGenerator` | 遍历 AssetRegistry，为资产（Blueprint/WidgetBlueprint/UserDefinedStruct/UserDefinedEnum/其他）生成一个带 `[PathName]` 特性的 C# 分部类文件 | `GetAssetsByPaths` → 按类型分派到 `FClassGenerator`/`FStructGenerator`/`FEnumGenerator`/自身 `GeneratorAsset` |

关键类关系与生命周期：

```
FUnrealCSharpEditorModule::Generator(Platform, bForceCompileInterop)     UnrealCSharpEditor.cpp:314
  ├─ OnBeginGenerator.Broadcast()                       :318  → FEditorListener::OnBeginGenerator（可选清空 Proxy 目录）
  ├─ FSolutionGenerator::Generator()                    :335  重写 .sln/.csproj/.props（覆盖式）
  ├─ FCodeAnalysis::CodeAnalysis()                      :339  dotnet build + CodeAnalysis.exe（同步阻塞）
  ├─ FDynamicGenerator::CodeAnalysisGenerator()         :343
  ├─ FGeneratorCore::BeginGenerator()                   :345  读设置、加载 OverrideFunction.json
  ├─ FClassGenerator / FStructGenerator / FEnumGenerator / FAssetGenerator / FGameplayTagGenerator /
  │  FBindingClassGenerator / FBindingEnumGenerator      :349-378
  ├─ FGeneratorCore::EndGenerator()                     :380  bIsFull 默认 true → DeleteRemainGeneratorFiles()
  ├─ CollectGarbage(RF_NoFlags, true)                   :384  ← 显式 GC，且在全量资产生成**之后**
  └─ FCSharpCompiler::Get().ImmediatelyCompile(...)     :388
```
`FSolutionGenerator` 与 `FCodeAnalysis` 都在 `FGeneratorCore::BeginGenerator()` **之前**执行，因此它们看不到本次生成的文件清单；而 `FAssetGenerator` 在 `BeginGenerator/EndGenerator` 之间执行，其产出会进入 `GeneratorFiles` 供 `EndGenerator` 做残留清理对账。

`FDoxygenConverter` 的位置：它不在上述主流程里，而是在 `FClassGenerator::Generator(UClass*)` 内部每个 `UFUNCTION` 处被调用一次（`FClassGenerator.cpp:459-467`），**默认开启**（`UUnrealCSharpEditorSetting` 的 `bIsGenerateFunctionComment(true)`，`UnrealCSharpEditorSetting.cpp:37`）。

## 2. 关键调用链

1. 全量生成（编辑器工具栏 / 控制台 `UnrealCSharp.Editor.Generator` / 右键 New Class / Cook）：
   `UnrealCSharpEditor.cpp:124`（`FAutoConsoleCommand`）→ `FUnrealCSharpEditorModule::Generator(IniPlatformName)`（`:314`）→ `FSolutionGenerator::Generator()`（`:335` → `FSolutionGenerator.cpp:9`）→ `CopyTemplate` ×13（`FSolutionGenerator.cpp:15-136`）→ `FFileHelper::LoadFileToString`/`SaveStringToFile`（`:155`/`:162`）
2. Cook 期触发：
   `GeneratorScriptCodeCommandlet.cpp:30` → `FUnrealCSharpEditorModule::Generator(Platform, true)`；该 Commandlet 还支持 `-Rebuild` 删除已发布程序集（`:12-28`），`IsRunningGeneratorScriptCodeCommandlet()` 供别处判断（`:36-42`）
3. 编辑器启动即跑分析器：
   `FEditorListener.cpp:39-43`（`FCoreDelegates::OnPostEngineInit`/`GetOnPostEngineInit()` 注册 `&FEditorListener::OnPostEngineInit`）→ `FEditorListener::OnPostEngineInit`（`:181`）→ `FCodeAnalysis::CodeAnalysis()`（`:183`）→ `FCodeAnalysis::Compile()`（定义 `FCodeAnalysis.cpp:39`，由 `:7` 调用）→ `SyncProcess("dotnet", "build ... -c Debug")`（`:49`）
   > **更正**：原文此跳写作 "`UnrealCSharpEditor.cpp:77-82`（`FCoreDelegates::OnPostEngineInit`）→ `FEditorListener::OnPostEngineInit`"，**首跳接错**——`UnrealCSharpEditor.cpp:77-82` 注册的是 `FUnrealCSharpEditorModule::OnPostEngineInit`（`:253`，实现体只有 `RegisterMenus()`，`:255`），与监听器无关；`FEditorListener` 自己独立注册（`:39-43`）。
4. 蓝图编译后增量生成（**不触发残留清理**）：
   `FEditorListener::OnBlueprintCompiled`（`:466`）→ `CompileChangedBlueprints`（`:476`）→ `FGeneratorCore::BeginGenerator(false)`（`:478`）→ `FAssetGenerator::Generator(AssetData)`（`:504`/`:514`）→ `EndGenerator(false)`（`:521`）
5. 资产增删改（含重命名删除旧文件）：
   `FEditorListener.cpp:344-350`（`OnAssetAdded/Removed/Renamed` 委托注册）→ `OnAssetChanged`（`:529`，内部 `BeginGenerator(false)` `:538`）→ `OnAssetRenamed` 删旧文件（`:377`）→ `FAssetGenerator::Generator(InAssetData)`（`:383`）
6. Doxygen 注释转换：
   `FClassGenerator::Generator(UClass*)` → `FUnrealCSharpFunctionLibrary::IsGenerateFunctionComment()`（`FClassGenerator.cpp:459`）→ `Function->GetMetaData(TEXT("Comment"))`（`:461`）→ `FDoxygenConverter(TEXT("\t\t"))(Comment)`（`:465`）→ `FDoxygenConverter.cpp:351 operator()` → `Tokenize`（`:363`）→ `Lex`（`:321`）
7. 资产 → C# 文件落盘：
   `FAssetGenerator::Generator()`（`FAssetGenerator.cpp:21`）→ `AssetRegistryModule.Get().GetAssetsByPaths(...)`（`:40`）→ `Generator(AssetData, true)`（`:44`）→（`else` 分支 `:118`）→ `GeneratorAsset`（`:126`）→ `GetFileName`（`:187`，→ Core `FUnrealCSharpFunctionLibrary.cpp:703-715`）→ `AddGeneratorFile`（`:189`）→ `SaveStringToFile`（`:191`）
8. 残留文件清理：
   `UnrealCSharpEditor.cpp:380` → `FGeneratorCore::EndGenerator(true)`（`FGeneratorCore.cpp:1024`）→ `DeleteRemainGeneratorFiles()`（`:1044` → `:1050-1078`，只在两个 Proxy 目录内删除未登记的 `*.cs`）；另一条路径是 `FEditorListener::OnBeginGenerator`（`FEditorListener.cpp:238-258`，受 `bEnableDeleteProxyDirectory` 控制，**默认 false**）

## 2.1 逐函数清单（覆盖本组全部函数）

### `FCodeAnalysis`（`Private/FCodeAnalysis.cpp`，89 行，全部读完）

| 函数 | 做了什么 | 调用方（证据） | 结论 |
|---|---|---|---|
| `CodeAnalysis()` `:5-10` | 先 `Compile()` 再 `Analysis()`（无参全量） | `UnrealCSharpEditor.cpp:95`（控制台命令 `UnrealCSharp.Editor.CodeAnalysis`）、`FEditorListener.cpp:183`（`OnPostEngineInit`，即**每次编辑器启动**） | F-GEN3-012（每次启动/每次生成同步 `dotnet build`，退出码丢弃） |
| `Analysis(const FString& InFile)` `:12-37` | 拼 `CodeAnalysis.exe` 路径 + `true "<分析器目录>" "<单文件>"` 参数，`SyncProcess` 跑一次 | `FEditorListener.cpp:330`、`UnrealCSharpBlueprintToolBar.cpp:80` | 无 exe 存在性校验；`Printf` 拼参数；异常静默（并入 F-GEN3-012） |
| `Compile()` `:39-52`（private） | `dotnet build <CodeAnalysis.csproj> --nologo -c Debug` | 仅 `CodeAnalysis()` `:7` | F-GEN3-012；`static` 缓存 dotnet 路径 `:41` |
| `Analysis()` `:54-89`（private） | 拼 exe 路径 + `false "<分析器目录>" "<游戏目录>" "<自定义工程目录>"...` | 仅 `CodeAnalysis()` `:9` | 参数拼接无转义、无失败反馈（并入 F-GEN3-012） |

### `FDoxygenConverter`（`Private/FDoxygenConverter.cpp`，475 行，全部读完）

| 函数 | 做了什么 | 调用方（证据） | 结论 |
|---|---|---|---|
| `FTextReader::FTextReader` `:69-77` | 保存 `FStringView`，初始化 5 个成员（`bHasSaveText/BeginIndex/EndIndex/bIsEof/CurrentIndex`） | `Tokenize` `:315` | `Text` 是 `FStringView` → 依赖调用方字符串存活；当前调用点 `FClassGenerator.cpp:465` 传入临时 `Comment` 的 view，调用期间有效，安全 |
| `FTextReader::NextChar` `:79-94` | 前进一个字符；越过末尾时置 `bIsEof` | `Bump` `:100`、`Lex` `:207`、`:300` | 正确；`bIsEof` 只在越界后置位，`GetCurrentChar` 有双重保护 |
| `FTextReader::Bump` `:96-101` | `Save()` + `NextChar()` | `EatWhile` `:145`、`Lex` `:248`、`:260` | 正确 |
| `FTextReader::Save` `:103-113` | 首次调用记录 `BeginIndex`，每次更新 `EndIndex` | 仅 `Bump` `:98` | 正确 |
| `FTextReader::GetCurrentChar` `:115-118` | 返回当前字符或 `INDEX_NONE` | `EatWhile` `:143`、`Lex` `:176` | 正确（越界有保护） |
| `FTextReader::GetTokenRange` `:120-123` | 返回 `FTextRange(BeginIndex, EndIndex+1)` | `Tokenize` `:328` | **缺陷根源**：未 `Save()` 时返回 `(0,1)` 而非空 → F-GEN3-002，并使 `:330` 的空检查失效（F-GEN3-013） |
| `FTextReader::GetSaveText` `:125-128` | 返回已保存区间；未保存时长度为 0 | `Lex` `:232`（取标签名） | 唯一正确使用 `bHasSaveText` 的地方，但 `TagOther` 没走它（F-GEN3-002） |
| `FTextReader::ResetBuffer` `:130-137` | 复位 `bHasSaveText=false`、`BeginIndex=EndIndex=0` | `Lex` `:174` | 正确，但正是它把"未消费"状态伪装成 `(0,1)` |
| `FTextReader::EatWhile` `:139-151` | 按谓词连续 `Bump`，返回消费数 | `Lex` `:190`、`:209`、`:250`、`:278`、`:290` | 逻辑正确；**谓词由调用方决定是否跨行**，`'/'` 分支的谓词缺换行判断 → F-GEN3-006 |
| `FTextReader::IsEof` `:153-156` | 返回 `bIsEof` | `EatWhile` `:143`、`Tokenize` `:319` | 正确（保证循环终止） |
| `Lex` `:172-309`（**全局、未 static**） | 单步词法：跳过并产出 Whitespace / TagParam / TagReturn / TagBrief / TagOther / Trivial / Name / Description / EndOfText | 仅 `Tokenize` `:321` | 三个子缺陷：`Name` 消费 0 字符仍返回 `Name`（F-GEN3-002）、`'/'` 跨行吞噬（F-GEN3-006）、未 `static`（F-GEN3-013） |
| `Tokenize` `:311-339`（**全局、未 static**） | 循环 `Lex` 直到 EOF，收集 `TArray<FTokenData>` | 仅 `operator()` `:363` | `:330` 空 range 检查无效；无明显性能问题（注释量级小） |
| `Slice` `:341-344`（**全局、未 static**） | `Text.SubStr(Begin, Len)` | `operator()` `:371`/`:380`/`:389`/`:396`/`:403`/`:410` | 本身正确；被用在错误的 range 上（F-GEN3-002） |
| `FDoxygenConverter::FDoxygenConverter` `:346-349` | 存缩进字符串 | 唯一调用点 `FClassGenerator.cpp:465`（传 `TEXT("\t\t")`） | 无问题（按值拷贝 `FString` 可改 `const FString&` 存引用需注意生命周期，当前可接受） |
| `FDoxygenConverter::operator()` `:351-475` | 词法→按标签分流（brief/param/return/other）→拼 `///` XML 注释 | 唯一调用点 `FClassGenerator.cpp:465` | F-GEN3-001（无 XML 转义）、F-GEN3-005（无 `@brief` 则整体丢弃、`@code/@endcode` 不配对） |

### `FSolutionGenerator`（`Private/FSolutionGenerator.cpp`，467 行，全部读完）

| 函数 | 做了什么 | 调用方（证据） | 结论 |
|---|---|---|---|
| `Generator()` `:9-137` | 14 次 `CopyTemplate`（调用行 15/24/32/40/50/55/63/74/78/88/97/107/115/126，其中 `:50`、`:74` 两次不带替换函数），把插件 `Script/`、`Template/` 的工程文件写到 `<Project>/Script/` 下并做替换 | `UnrealCSharpEditor.cpp:106`（控制台命令）、`:335`（主生成流程）；库内无其它调用点 | F-GEN3-007（覆盖式）、F-GEN3-015（Interop 缺 `ReplacePluginBaseDir`） |
| `CopyTemplate(Dest, Src, bool)` `:139-145` | 目标不存在或允许覆盖时 `IFileManager::Copy` | `Generator()` `:50-53`、`:74-76` 共 2 次（无替换列表） | `Copy` 返回值未检查（并入 F-GEN3-007） |
| `CopyTemplate(Dest, Src, FnList, bool)` `:147-164` | 读源 → 依次执行替换函数 → 写目标 | `Generator()` 中 12 次 | F-GEN3-003（`LoadFileToString` 未检查 → 可能清空目标）、F-GEN3-007（`SaveStringToFile` 未检查） |
| `ReplacePluginBaseDir` `:166-175` | 把模板里的 `UnrealCSharp` 换成插件目录名 | `&` 取址于 `:93` | F-GEN3-016（`FindLastChar` 未校验、全局替换）；Interop 未使用 → F-GEN3-015 |
| `ReplaceImport` `:177-188` | `<Import Project="" Condition="Exists('')"/>` → Game.props | `:102` | 与 `Template/Game.csproj:2` 逐字一致，无问题 |
| `ReplaceDefineConstants` `:190-224` | 生成 `UE_5_0_OR_LATER;…` + `WITH_EDITOR` + `WITH_LEANCLR` | `:121` | 逻辑正确（`ENGINE_MAJOR/MINOR_VERSION` 循环）；`IsRunningCookCommandlet()` 判断 Cook 期不加 `WITH_EDITOR`，合理 |
| `ReplaceScriptOutputPath` `:226-234` | `<ScriptOutputPath>` → `..\..\Content\<Publish>` | `:59`（Weavers）、`:111`（Game.props） | 相对深度对 `<ScriptDir>/<Name>/` 恒定成立；`Content` 为硬编码（与 `FPaths::ProjectContentDir()` 一致），无问题 |
| `ReplaceOutputPath` `:236-244` | `<OutputPath>` → `..\..\Content\<Publish>` | `:83`（Interop） | 同上；模板 `Interop.csproj:19` 占位符一致 |
| `ReplaceHintPath` `:246-256` | `<HintPath>` → `..\..\Content\<Publish>\Interop.dll` | `:122`（Shared.props） | 占位符一致（`Shared.props:18`）；**相对基准存疑**，见 §5 第 1 条 |
| `ReplaceTargetFramework` `:258-268` | `<TargetFramework>` → `net<version>.<minor>` | `:20`、`:84`、`:120` | 占位符一致；依赖 `GetDotnetVersion()`（Core 设置），无问题 |
| `ReplaceProjectReference` `:270-281` | `<ProjectReference Include="" />` → `..\<UEName>\<UEName>.csproj` | `:103` | 与 `Template/Game.csproj:4` 一致，无问题 |
| `ReplaceYield` `:283-290` | 替换 `UENamePlaceholder`/`GameNamePlaceholder` | `:68`（Weaver） | 无问题 |
| `ReplaceGameName` `:292-296` | 替换 `GameNamePlaceholder` | `:46`（UnrealTypeSourceGenerator.cs） | 无问题 |
| `ReplaceDefinition` `:298-310` | 替换 `DefinitionPlaceholder`/`AssemblyPlaceholder` | `:69` | 无问题 |
| `ReplaceProject` `:312-337` | 把 `.sln` 里两条空名字 `Project(...)` 行填成 UE/Game 工程 | `:132` | 与 `Template/Script.sln:6`、`:8` 逐字一致；GUID 硬编码在模板里（**稳定**，这点是好的） |
| `ReplaceProjectPlaceholder` `:339-379` | 用 Interop + 各 `CustomProjects` 生成 `Project()…EndProject` 段落 | `:133` | F-GEN3-004（`CustomProject.GUID()` 可能非法）；`:346` 的 `\r\n` 与模板 CRLF 匹配（已实测） |
| `ReplaceSolutionConfigurationPlatformsPlaceholder` `:381-422` | 为 Interop/自定义工程生成配置行 | `:134` | 与 `Template/Script.sln:42` 匹配；GUID 同源 F-GEN3-004 |
| `ReplaceScriptPath` `:424-427` | `ScriptPathPlaceholder` → `Script` | `:70`（Weaver） | 无问题 |
| `AddProjectGeneratorHeaderComment` `:429-441` | 给工程文件加 `DO NOT modify this manually!` 头注释 | `:21`、`:37`、`:60`、`:85`、`:94`、`:112`、`:123`（7 次） | 无问题（注释文本与实际覆盖行为一致） |
| `AddSolutionGeneratorHeaderComment` `:443-457` | 给 `.sln` 加 `#` 头注释，**`#if !VS2026`** 包裹 | `:135` | 无问题；VS2026 下静默不加注释（信息缺失，可接受） |
| `AddCSharpGeneratorHeaderComment` `:459-467` | 给 `.cs`（生成器源码）加 `//` 头注释并把 `\n` 归一为 `\r\n` | `:29`、`:47`、`:71` | 无问题 |

### `FAssetGenerator`（`Private/FAssetGenerator.cpp`，192 行，全部读完）

| 函数 | 做了什么 | 调用方（证据） | 结论 |
|---|---|---|---|
| `Generator()` `:21-55` | 读设置 → `GetAssetsByPaths(..., true)` 递归取资产 → 逐个 `Generator(AssetData, true)` → 统一生成延迟的 `UUserDefinedEnum` → 清空静态数组 | `UnrealCSharpEditor.cpp:361` | F-GEN3-009（静态裸指针数组 + 跨循环 GC 窗口）；资产数量大时无分批/无进度细分 |
| `Generator(const FAssetData&, bool)` `:57-124` | 按 `SupportedAssetClassName` 分派：Blueprint/WidgetBlueprint→`FClassGenerator`、UserDefinedStruct→`FStructGenerator`、UserDefinedEnum→延迟或直接 `FEnumGenerator`、其它→`GeneratorAsset` | `FEditorListener.cpp:357`、`:383`、`:391`、`:504`、`:514`（均为单参=不延迟）；`Generator()` `:44` | F-GEN3-014（3 处 `LoadObject` 失败静默无日志）；分派逻辑本身无判空缺陷 |
| `GeneratorAsset(const FAssetData&)` `:126-192` | 过滤 `IsSupported` → 生成含 `[PathName]` 的 `public partial class` → `AddGeneratorFile` → 落盘 | 仅 `:118`（内部） | F-GEN3-010（命名空间未清洗、`Game` 子串替换）、F-GEN3-011（写盘失败仍登记） |

### 其余两个文件

| 文件/符号 | 做了什么 | 调用方（证据） | 结论 |
|---|---|---|---|
| `ScriptCodeGeneratorMacro.h:3` `ENUM_DEFAULT_MAX_INDEX`（值 0） | 枚举默认值回退索引 | `FClassGenerator.cpp:1106`、`:1116`、`:1192`（共 4 命中含定义） | 活代码；命名与语义不符 → F-GEN3-018 |
| `ScriptCodeGenerator.Build.cs`（`PublicDependencyModuleNames:25-32`、`PrivateDependencyModuleNames:35-50`） | 声明 `Core`/`UnrealCSharpCore` 等 11 个模块 | UBT 构建期读取 | F-GEN3-017（缺 `AssetRegistry`/`StructUtils`） |
| `Private/ScriptCodeGenerator.cpp:7-16` | `StartupModule`/`ShutdownModule` 均为空实现，`IMPLEMENT_MODULE`（`:20`） | UE 模块加载 | 无问题（模块无自身初始化需求；所有工作由编辑器模块驱动） |

## 3. 发现清单

### P0

（暂无）

### P1

### [F-GEN3-001] Doxygen 描述文本未做 XML 转义，直接拼进 C# XML 文档注释

- **类别**: Bug
- **严重度**: **P2**
- **复核结论**: **确认**（代码事实、调用上下文、类别、严重度全部成立）
- **可达性**: 活跃（`DefaultUnrealCSharpEditorSetting.ini:27 bIsGenerateFunctionComment=True`，本项目每次生成都会走这条路径）
- **复核证据**: `Source/ScriptCodeGenerator/Private/FDoxygenConverter.cpp:440/444/455/458/468` 五行 `StringBuilder.Append(...)` 均已逐字回读确认**无任何转义**；`grep "&lt;|&amp;|&gt;|&quot;|EscapeXml"` 于 `Source/ScriptCodeGenerator` 命中 **3 处**，全部在 `FGameplayTagGenerator.cpp:135-137`（`FDoxygenConverter.cpp` 内 0 命中）；调用点 `FClassGenerator.cpp:459/461/465` 逐字吻合（输入为 UHT `Comment` 元数据原文，`FClassGenerator.cpp:461`）；产物侧旁证：`Script/Shared.props:16,19` 的 `NoWarn` 只含 `0109;1701;1702;8500`，且全仓库 `grep GenerateDocumentationFile` 于 `Script/`（含插件侧 `Script/`）**0 命中** → 默认构建不会因非法 XML 报 CS1570（不改变本条的"隐患"定性）
- **级别变动**: 无（维持 P2；P1→P2 的调整**成立**）
- **文件**: `Source/ScriptCodeGenerator/Private/FDoxygenConverter.cpp:440`、`:444`、`:455`、`:458`、`:468`（`Descriptions`/`ParamName` 逐条 `Append`）
- **函数**: `FDoxygenConverter::operator()(const FStringView&) const`
- **置信度**: 高（代码路径确定；影响面取决于注释内容，已确认调用方把 C++ 注释原文传进来）

**现状（代码事实）**
```cpp
// FDoxygenConverter.cpp:436-460
	for (auto& Tag : TagData)
	{
		if (Tag == TagParam)
		{
			StringBuilder.Append(Indent).Append(TEXT("/// <param name=\"")).Append(Tag.ParamName).Append(TEXT("\">\n")); // 440 未转义

			for (const auto& Description : Tag.Descriptions)
			{
				StringBuilder.Append(Indent).Append(TEXT("/// ")).Append(Description).Append(TEXT("\n")); // 444 未转义
			}

			StringBuilder.Append(Indent).Append(TEXT("/// </param>\n"));
		}
		else if (Tag != TagBrief && Tag != TagReturn && Tag != TagDefault)
		{
			StringBuilder.Append(Indent).Append(TEXT("/// <")).Append(Tag.Name).Append(TEXT(">\n")); // 451 未转义
			...
```
`Descriptions` 直接来自被转换文本的切片（`:410` `TagData.Last().Descriptions.Emplace(Slice(InText, Token.TextRange));`），`Slice` 只是 `SubStr`（`:341-344`），全程没有 `Replace(TEXT("<"), TEXT("&lt;"))` 之类的转义。

**调用上下文**
唯一调用点：`Source/ScriptCodeGenerator/Private/FClassGenerator.cpp:465`
```cpp
// FClassGenerator.cpp:459-467
		if (FUnrealCSharpFunctionLibrary::IsGenerateFunctionComment())
		{
			auto Comment = Function->GetMetaData(TEXT("Comment"));   // 461 UHT 抓取的 C++ 注释原文

			if (!Comment.IsEmpty())
			{
				FunctionComment = FDoxygenConverter(TEXT("\t\t"))(Comment);   // 465
			}
		}
```
即输入是 **UHT 反射元数据 `Comment`**（C++ 头文件里的注释原文），不是经过清洗的文本。触发时机是编辑器/Commandlet 触发的代码生成（非运行时热路径）。

**问题**
C++ 注释里 `<`、`>`、`&` 极常见（`TArray<FString>`、`a < b`、`read & write`、`&&`）。这些字符被原样写进 `/// ` 注释后：
1. 生成的 C# 文档注释 XML 非法（`&` 未定义为实体、孤立 `<` 无闭合）→ 若项目开启 `/doc`（`GenerateDocumentationFile`）会得到 CS1570，开启 `TreatWarningsAsErrors` 时直接影响编译；未开启时至少导致 IDE 悬浮文档解析失败、内容显示错乱。
2. `@param` 名字同样未转义（`:440` 的 `name="..."` 属性值，含 `"` 或 `&` 会破坏属性）。
已在本机生成的工程中未发现对 `&`/`<` 的处理代码，且**同一模块内存在"应该怎么做"的先例**（这使该缺陷更像遗漏而非有意设计）：
```cpp
// FGameplayTagGenerator.cpp:128-146 —— 同一模块的另一个生成器做对了
void FGameplayTagGenerator::GeneratorDocComment(FString& OutContent, const FString& InPad, const FString& InComment)
{
	if (!InComment.IsEmpty())
	{
		const auto Comment = InComment.Replace(TEXT("\r\n"), TEXT(" "))
		                              .Replace(TEXT("\n"), TEXT(" "))
		                              .Replace(TEXT("\r"), TEXT(" "))
		                              .Replace(TEXT("&"), TEXT("&amp;"))   // 135
		                              .Replace(TEXT("<"), TEXT("&lt;"))    // 136
		                              .Replace(TEXT(">"), TEXT("&gt;"));   // 137

		OutContent += FString::Printf(TEXT(
			"%s/// <summary>%s</summary>\n"
		),
		                              *InPad,
		                              *Comment
		);
	}
}
```
（`FGameplayTagGenerator.cpp:135-137` 逆序替换 `&` 在前的写法也是正确的转义顺序，可直接照搬；该实现未处理 `"`，若 comment 会进入 `name="..."` 属性，需要补 `&quot;`。）

**建议**
在 `StringBuilder.Append(Description)` 之前统一走一个转义函数（只转 `&`→`&amp;`、`<`→`&lt;`、`>`→`&gt;`、`"`→`&quot;`，注意 `&` 必须最先替换），`Tag.ParamName` 同样处理：
```cpp
static FString EscapeXml(const FStringView In) { ... }
StringBuilder.Append(Indent).Append(TEXT("/// ")).Append(EscapeXml(Description)).Append(TEXT("\n"));
```

**验证方式**
`grep -n "&lt;\|&amp;\|EscapeXml" Source/ScriptCodeGenerator` → 当前 0 命中；构造 `/** @brief A & B < C */` 的 UFUNCTION 后重新生成，检查产出的 `///` 行是否含裸 `&`/`<`。

---

### [F-GEN3-002] `@param[in]` / `@param[out]` 会把"整段注释的第一个字符"当成参数名

- **类别**: Bug
- **严重度**: **P2**
- **复核结论**: **确认**（按源码逐字符推演成立：`TagParam` → `Expect=Name` → `'['` 使 `EatWhile` 消费 0 字符 → `ResetBuffer()` 把 `BeginIndex/EndIndex` 归零 → `GetTokenRange()` 返回 `(0,1)` → `Slice` 取到整段注释第 0 个字符）
- **可达性**: 活跃（`bIsGenerateFunctionComment=True`；触发条件仅为注释使用 `@param[in]/[out]/[in,out]` 这类标准 Doxygen 写法）
- **复核证据**: `FDoxygenConverter.cpp:276-286`（Name 分支整段）、`:120-123`（`GetTokenRange`）、`:130-137`（`ResetBuffer` 清零）、`:234-237`（`TagParam` 置 `Expect=Name`）、`:403`（`ParamName = Slice(...)`）、`:396`（`TagOther` 也走 `Slice` 而非 `GetSaveText()`，`:125-128` 的 `bHasSaveText` 保护因此形同虚设）—— 全部逐字回读吻合
- **级别变动**: 无（维持 P2；P1→P2 的调整**成立**）
- **文件**: `Source/ScriptCodeGenerator/Private/FDoxygenConverter.cpp:276-286`（Name 分支）、`:120-123`（`GetTokenRange`）、`:403`（ParamName 赋值）
- **函数**: `Lex(FTextReader&, EExpect&)`、`FTextReader::GetTokenRange()`、`FDoxygenConverter::operator()`
- **置信度**: 高（纯本地逻辑推演，可直接断点验证）

**现状（代码事实）**
```cpp
// FDoxygenConverter.cpp:274-286
	switch (OutExpect)
	{
	case EExpect::Name:
		{
			OutTextReader.EatWhile([](const int InChar)
			{
				return InChar > 0 && InChar < 128 && (FChar::IsAlnum(InChar) || InChar == '_');
			});                       // 278-281：遇到 '[' 时一个字符都不消费

			OutExpect = EExpect::Description;

			return ETokenKind::Name;  // 285：仍然产出一个 Name token
		}
```
```cpp
// FDoxygenConverter.cpp:120-123 —— 未调用过 Save() 时 BeginIndex/EndIndex 仍是构造函数里的 0/0
	FTextRange GetTokenRange() const
	{
		return FTextRange(BeginIndex, EndIndex + 1);   // → (0,1)，长度 1，"非空"
	}
```
```cpp
// FDoxygenConverter.cpp:401-406
		case ETokenKind::Name:
			{
				TagData.Last().ParamName = Slice(InText, Token.TextRange);  // 403 → 切片 [0,1) = 整段注释首字符
```
`@param` 的 `TagParam` 分支（`:234-237`）会置 `OutExpect = EExpect::Name`；下一个字符若是 `[`，`EatWhile` 立即返回 0，而 `ResetBuffer()`（`:130-137`）把 `BeginIndex/EndIndex` 归零 → `GetTokenRange()` 返回 `(0,1)`。

**调用上下文**
同 F-GEN3-001：`FClassGenerator.cpp:465` ← `FClassGenerator::Generator(UClass*)` ← `FAssetGenerator::Generator(FAssetData, bool)` ← `FAssetGenerator::Generator()`。输入为 UHT `Comment` 元数据。

**问题**
Doxygen/UE 风格里 `@param[in]`、`\param[out]`、`@param[in,out]` 是标准写法。此时：
- `ParamName` = **整段注释文本的第 0 个字符**（通常是 `/`、`*` 或空格），生成 `/// <param name="/">` 这类垃圾；
- 真正的名字 `Foo` 以及 `[in]` 一起被后面那个 `Description` token（`:288-296`，吃到行尾）吞掉，`[in]` 也原样出现在描述里。
同一个"未消费就返回 token"缺陷还有第二个出口：`@`/`\` 后面若不是 ASCII 字母（如 `/** @ 5 */`、`/` 行首的孤立 `@`），`GetSaveText()` 为空（`:125-128` 有 `bHasSaveText` 保护），但 `operator()` 的 `TagOther` 分支用的是 `Slice(InText, Token.TextRange)`（`:396`）而不是 `GetSaveText()`，于是同样取到首字符 → 输出 `/// <X>` … `/// </X>` 的畸形标签。

**建议**
两处一起修：
1. `Lex` 的 `Name` 分支在 `EatWhile` 返回 0 时不要返回 `Name`（返回 `Trivial` 并让 `OutExpect` 保持/降级为 `Description`），并让 `GetTokenRange()` 在 `!bHasSaveText` 时返回空 `FTextRange`；
2. `TagOther` 分支改用 `GetSaveText()`（需要把 tag 名一起存进 `FTokenData`），空名直接丢弃而不是生成 `<>`。
另建议显式吃掉 `@param` 后面可选的 `[in]`/`[out]`/`[in,out]`。

**验证方式**
对含 `/** @param[in] int32 Foo xxx */` 的函数重新生成 C#，观察 `<param name="...">` 的值；或对 `FDoxygenConverter(TEXT(""))(TEXT("/** @param[in] A b */"))` 写单测断言 `ParamName == "A"`。

---

### [F-GEN3-003] `CopyTemplate` 不检查 `LoadFileToString` 返回值：源模板缺失/被占用时会把空内容写进目标文件

- **类别**: Bug（数据损坏）
- **严重度**: **P1**
- **复核结论**: **确认**（`LoadFileToString` 返回 `bool` 而代码未接收；`Src` 完全无校验；空串会被写盘，且 `SaveStringToFile` 只在"内容相同"时提前 `return true`，内容不同必然真写）
- **可达性**: 活跃（`CopyTemplate` 默认 `bReplaceExistingFile = true`，`FSolutionGenerator.h:11/15`，每次生成都重写目标文件）
- **复核证据**: `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:139-145`、`:147-164` 逐字回读（`:155 LoadFileToString` 无接收、`:159 Function(Result)` 对空串做无操作替换、`:162 SaveStringToFile` 同样丢弃返回值）；grep `LoadFileToString|SaveStringToFile` 于该文件命中 **2 处（155/162）**；被调用方契约：`FUnrealCSharpFunctionLibrary.cpp:1150-1179`（`:1154-1163` 内容相同才提前返回、`:1174 CreateDirectoryTree` 未检查、`:1177 ForceUTF8WithoutBOM` 真写并返回 `bool`）
- **级别变动**: 无（维持 P1——"把完好的 `.sln`/`.props` 清成 0 字节"属数据损坏，且静默）
- **文件**: `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:147-164`
- **函数**: `FSolutionGenerator::CopyTemplate(const FString&, const FString&, const TArray<TFunction<void(FString&)>>&, bool)`
- **置信度**: 高（`LoadFileToString` 返回 bool 而代码未接收；`SaveStringToFile` 会把空串写盘）

**现状（代码事实）**
```cpp
// FSolutionGenerator.cpp:147-164
void FSolutionGenerator::CopyTemplate(const FString& Dest, const FString& Src,
                                      const TArray<TFunction<void(FString& OutResult)>>& InFunction,
                                      const bool bReplaceExistingFile)
{
	if (auto& FileManager = IFileManager::Get(); !FileManager.FileExists(*Dest) || bReplaceExistingFile)
	{
		FString Result;

		FFileHelper::LoadFileToString(Result, *Src);   // 155 返回值被丢弃，只检查了 Dest 存在与否

		for (const auto& Function : InFunction)
		{
			Function(Result);                          // 159 对空串做 Replace：全部静默 no-op
		}

		FUnrealCSharpFunctionLibrary::SaveStringToFile(*Dest, Result);  // 162 返回值也被丢弃
	}
}
```
对照实现：`FUnrealCSharpFunctionLibrary::SaveStringToFile`（`UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1150-1179`）会 `CreateDirectoryTree`（`:1174`，返回值同样未检查）后 `ForceUTF8WithoutBOM` 写盘并返回 bool。

**调用上下文**
`FSolutionGenerator::Generator()`（`:9-137`）共 14 次调用 `CopyTemplate`（`CopyTemplate(` 起始行 15/24/32/40/50/55/63/74/78/88/97/107/115/126）；`Generator()` 是 public 导出 API（`FSolutionGenerator.h:8` `SCRIPTCODEGENERATOR_API`）。

**问题**
`Dest` 存在性检查了，`Src` 完全没检查。任一源文件缺失/被独占（插件目录只读、杀软占用、`.cs` 被 IDE 锁）时：`LoadFileToString` 返回 false 且 `Result` 保持空串，随后**空串被写进目标**——原本完好的 `<Project>/Script/Script.sln`、`Interop.csproj`、`Shared.props` 会变成 0 字节文件，且没有任何日志。因为默认 `bReplaceExistingFile = true`（`FSolutionGenerator.h:11`、`:15`），这对**每次**生成都成立。`SaveStringToFile` 的返回值被忽略意味着"写失败"也完全静默（例如 VS 正打开 `.sln` 时的文件锁）。

**建议**
```cpp
if (!FFileHelper::LoadFileToString(Result, *Src)) { UE_LOG(..., Error, TEXT("template missing: %s"), *Src); return; }
```
并对 `SaveStringToFile` 的 false 返回值记录 Warning（至少让"生成失败"可见）。若确实想"源缺失时清空目标"，也必须显式写出来而不是隐式发生。

**验证方式**
`grep -n "LoadFileToString\|SaveStringToFile" Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp` → 155/162 两处均为无返回值检查的语句；把 `Template/Shared.props` 临时改名后触发一次生成，观察 `<Project>/Script/Shared.props` 是否被清零（只读验证，勿改仓库）。

---

### [F-GEN3-004] 自定义工程的 GUID 由 32 位 hash 用 `%X` 格式化：高位为 0 时产出**非法 GUID**，写进 `.sln`

- **类别**: Bug / 平台兼容
- **严重度**: **P2**
- **复核结论**: **部分确认**（机制成立；**偏差：非法概率与示例算错**——原文写"概率 1/256"、"段长 6-4-4-4-10"，重算为 **1/16**，正确段长为 `L-min(L,4)-max(L-4,0)-min(L,4)-(max(L-4,0)+L)`）
- **可达性**: 潜伏（代码路径每次生成都会执行，但**本项目 `Config/DefaultUnrealCSharpSetting.ini` 未配置任何 `CustomProjects`**（全文 9 行，只有五平台 `*ScriptDomainType=LeanCLR`），`GetCustomProjects()` 返回空数组 → `.sln` 里根本没有名字派生 GUID；用户一旦添加自定义工程即**活跃**）
- **复核证据**: `UnrealCSharpSetting.h:36-50` 逐字回读（`:38 %X`、`:41 "%s-%s-%s-%s-%s%s"`、`:43-48` 六段参数）；`FSolutionGenerator.cpp:359-369`（`:368 *CustomProject.GUID()`）、`:399-411`（`:407` 起四处写入 `ProjectConfigurationPlatforms`）；引擎侧依据：`GetTypeHash(FString)` 在 **UE 5.6 是 `FCrc::Strihash_DEPRECATED(S.Len(), *S)` 返回 `uint32`**（`Runtime/Core/Public/Containers/UnrealString.h.inl:2350-2354`；`TStringView` 版本见 `Containers/StringView.h:459-463`，二者注释互相要求一致）——原文写的 `FCrc::StrCrc32` **不是** `GetTypeHash(FString)` 的实现（`StrCrc32` 才返回 `uint32` 大小写敏感 CRC，见 `Misc/Crc.h:45`）；真实产物验证：`Script/Script.sln`（58 行）与 `UnrealCSharpTest.sln` 中 `grep "\{[0-9A-Fa-f]{1,7}[-,}]"` **0 命中**，30 处 GUID 全部为合法 8-4-4-4-12（含硬编码 `INTEROP_GUID` `Macro.h:53` → `Script/Script.sln:19/43-46`），各 `.csproj` `grep ProjectGuid|<Guid>` **0 命中**（SDK 风格工程不带 GUID）
- **级别变动**: 无（维持 P2；P1→P2 的调整**成立**）
- **文件**: `Source/UnrealCSharpCore/Public/Setting/UnrealCSharpSetting.h:36-50`（定义）→ `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:368`、`:407`（使用）
- **函数**: `FCustomProject::GUID() const`
- **置信度**: 高（格式化逻辑可直接推导；概率为数学结论）

**现状（代码事实）**
```cpp
// UnrealCSharpSetting.h:36-50
	FString GUID() const
	{
		const auto Hex = FString::Printf(TEXT("%X"), GetTypeHash(Name));   // 38：32 位 hash，前导零被吃掉

		return FString::Printf(TEXT(
			"%s-%s-%s-%s-%s%s"                                            // 期望拼成 8-4-4-4-12
		),
		                       *Hex,            // 第 1 段：应为 8 位
		                       *Hex.Mid(0, 4),  // 第 2 段
		                       *Hex.Mid(4, 4),  // 第 3 段
		                       *Hex.Mid(0, 4),  // 第 4 段
		                       *Hex.Mid(4, 4),  // 第 5 段
		                       *Hex             // 第 6 段：应为 12 位
		);
	}
```
使用点：
```cpp
// FSolutionGenerator.cpp:359-369
			Projects += FString::Printf(TEXT(
				"Project(\"{%s}\") = \"%s\", \"%s\\%s%s\", \"{%s}\"\r\n"   // 360
				"EndProject\r\n"
			),
			                            *CSHARP_GUID,
			                            *CustomProject.Name,
			                            *CustomProject.Name,
			                            *CustomProject.Name,
			                            *PROJECT_SUFFIX,
			                            *CustomProject.GUID()          // 368
			);
```
（`:407` 同样把该值写进 `SolutionConfigurationPlatforms` 段；`.sln` 模板见 `Template/Script.sln:6,8,19,42`。）

**调用上下文**
`FSolutionGenerator::Generator()`（`:9`）→ `ReplaceProjectPlaceholder`（`:339`）/ `ReplaceSolutionConfigurationPlatformsPlaceholder`（`:381`），由编辑器/Commandlet 触发的代码生成调用；`GetCustomProjects()` 来自 `UUnrealCSharpSetting`（`UnrealCSharpSetting.h:139`，config 持久化）。

**问题**
**（重算：原文的概率与示例均为错误，已改正）**

`GetTypeHash(FString)` 在 UE 5.6 = `FCrc::Strihash_DEPRECATED(Len, *S)`，返回 `uint32`（`UnrealString.h.inl:2350-2354`），因此 `%X` 的输出长度 `L ∈ [1,8]`，**无补零**。把六段长度写成 `L` 的函数：

| 段 | 表达式 | 实际长度 |
|---|---|---|
| 1 | `Hex` | `L` |
| 2 | `Hex.Mid(0,4)` | `min(L,4)` |
| 3 | `Hex.Mid(4,4)` | `max(L-4,0)` |
| 4 | `Hex.Mid(0,4)` | `min(L,4)` |
| 5+6 | `Hex.Mid(4,4)` + `Hex` | `max(L-4,0) + L` |

要凑成合法的 `8-4-4-4-12`，唯一解是 **`L == 8`**（此时最后一段 = 4+8 = 12 ✓）。而 `L == 8` 当且仅当哈希的**最高 4 位（nibble）非 0**，即 `hash >= 0x10000000`。

- **非法概率 = P(最高 nibble == 0) = 1/16 ≈ 6.25%**（不是原文的 1/256 —— 1/256 对应"最高**字节**为 0"的口径，与本格式化式无关）。
- 原文示例 `0x00A1B2C3 → "A1B2C3"` 的段长也算错了：`Mid(4,4)` 于 6 字符串上只取到 `"C3"`，实际输出是 `A1B2C3-A1B2-C3-A1B2-C3A1B2C3`（段长 **6-4-2-4-8**），不是原文写的 `A1B2C3-A1B2-B2C3-A1B2-B2C3A1B2C3`（6-4-4-4-10）。
- 极端例 `L == 4`（`hash < 0x10000`）：`ABCD-ABCD--ABCD-ABCD`（第三段为空），与 `02-UnrealCSharpCore核心/06-...md` 的 `F-BR-008` 描述一致 —— **两条发现同源**（`F-BR-008` 的"约 1/16"是对的，本报告原文的 1/256 才是错的；已对齐）。

一旦命中，`.sln` 里 `Project("{...}")` 与 `ProjectConfigurationPlatforms` 两处 GUID 一起失真 → VS/Rider 打开解决方案时报错或该工程不被加载，且用户完全无法从日志定位原因。
附带两个相关事实：
- **稳定性**：CRC32 对同一 `Name` 是确定性的，所以"每次生成都换 GUID"**不成立**（这点不构成问题）；但 `GUID` 与 `Name` 绑定，**用户重命名自定义工程 = GUID 变化** → VS 里该项目的历史引用关系/`.suo` 状态丢失。
- **无冲突检测**：`GetCustomProjects()` 是 TArray，代码不检查重复 `Name` 或重复 GUID；两个工程名哈希碰撞或重名时 `.sln` 出现重复 GUID 段，代码不做任何校验。

**建议**
不要从名字派生 GUID。要么在 `FCustomProject` 里加一个 `UPROPERTY(Config) FGuid Guid` 字段（`FGuid::NewGuid()` 首次生成后持久化，重命名也稳定），要么至少改成 `FString::Printf(TEXT("%08X"), GetTypeHash(Name))` 保证段长固定，并对重复 GUID 做一次 `TSet` 校验 + `UE_LOG(Warning)`。

**验证方式**
`grep "GetTypeHash(Name)|%08X"` 于 `Source/` → 确认只有 `UnrealCSharpSetting.h:38` 一处 `%X`、无 8 位补齐（已跑）。产物侧：`grep "\{[0-9A-Fa-f]{1,7}[-,}]"` 于全部 `*.sln`（命中 0，即当前产物无非法 GUID，因为 `CustomProjects` 为空）；在 `CustomProjects` 里加若干自定义工程后重新生成，再用正则 `\{[0-9A-Fa-f]{8}(-[0-9A-Fa-f]{4}){3}-[0-9A-Fa-f]{12}\}` 校验每个 `{...}`。若要复现 1/16，可用 PowerShell 枚举若干 `Name` 算 `GetTypeHash` 并检查输出是否短于 8 位（约每 16 个命中 1 个）。

---

### P2

### [F-GEN3-005] 没有 `@brief` 的注释被整体丢弃；`@code/@endcode` 等成对标签被转成不配对的畸形 XML 标签

- **类别**: Bug（功能缺失 + 输出非法）
- **严重度**: **P2**
- **复核结论**: **确认**（按源码逐 token 推演，两条结论均成立：① `Expect{}` 初值 = `EExpect::None`，首个标签之前的自由文本每字符产出一个 `Trivial` 并被 `operator()` 的 `default:` 丢弃；若注释以 `/**` 开头，`'/'` 分支（`:246-256`）直接把余下全文吞成一个 `Trivial` → 输出空串；② `@code`→`<code>`+`</code>`、`@endcode`→`<endcode>`+`</endcode>`，配对失配）
- **可达性**: 活跃（`bIsGenerateFunctionComment=True`；本项目 `Content/UnitTest/**` 的 UFUNCTION 注释即走此路径）
- **复核证据**: `FDoxygenConverter.cpp:315-317`（`Expect{}`）、`:298-308`（None 分支 `NextChar()` 后 `break` → 返回 `Trivial`）、`:394-399`（`TagOther` 落 `Slice`）、`:449-459`（一律 `<Name>`/`</Name>`）、`:214-232`（只有 `param`/`return`/`brief` 三个被映射）—— 全部逐字回读吻合
- **级别变动**: 无（维持 P2）
- **文件**: `Source/ScriptCodeGenerator/Private/FDoxygenConverter.cpp:317`、`:298-305`、`:394-399`、`:449-459`
- **函数**: `FDoxygenConverter::operator()(const FStringView&) const`、`Lex`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FDoxygenConverter.cpp:311-339（Tokenize 起始状态）
	FTextReader TextReader(InText);

	EExpect Expect{};        // 317 → EExpect::None
```
```cpp
// FDoxygenConverter.cpp:298-305 —— Expect 为 None 时每次只前进一个字符
	case EExpect::None:
		{
			OutTextReader.NextChar();   // 300：无 Bump()，不保存文本

			OutExpect = EExpect::None;

			break;
		}

	return ETokenKind::Trivial;         // 308
```
```cpp
// FDoxygenConverter.cpp:394-399 / 449-459
		case ETokenKind::TagOther:
			{
				TagData.Emplace(Slice(InText, Token.TextRange));   // 396：名字就是标签名（未清洗）
				break;
			}
...
		else if (Tag != TagBrief && Tag != TagReturn && Tag != TagDefault)
		{
			StringBuilder.Append(Indent).Append(TEXT("/// <")).Append(Tag.Name).Append(TEXT(">\n"));   // 451
			...
			StringBuilder.Append(Indent).Append(TEXT("/// </")).Append(Tag.Name).Append(TEXT(">\n"));  // 458
		}
```

**调用上下文**
`FClassGenerator.cpp:459-467`，仅当 `UUnrealCSharpEditorSetting::IsGenerateFunctionComment()` 为真时执行（`FUnrealCSharpFunctionLibrary.cpp:1122-1130`）。

**问题**
1. **自由文本全丢**：`Expect` 初值为 `None`，`/** 就是一段普通说明 */`（没有 `@brief`）会被逐字符吞成 `Trivial`，最后 `StringBuilder` 为空 → 函数注释完全消失。用户开启"生成函数注释"后看不到任何注释时无从判断是配置问题还是转换器问题。
2. **成对标签不配对**：`@code`/`@endcode` 分别变成 `<code>` 与 `</endcode>`（`:451`/`:458` 一律 `<Name>` + `</Name>`），`@param[in]`（见 F-GEN3-002）、`<b>`/`<i>` 原样透传，都会产出非法 XML。
3. `@note`/`@warning`/`@deprecated`/`@see`/`@ref`/`@p` 一律输出为 `<note>`…`</note>` 这类**非 C# 标准标签**（不报 CS1570，但 IDE 提示与文档工具会显示为未知标签）。任务书要求的标签覆盖表里，只有 `@brief`/`@param`/`@return` 三个被真正映射（`:214-232` 的 `TokenKind` 判定），其余全部落入 `TagOther`。

**建议**
- `Expect` 初始应为 `Description`，把首个标签之前的文本作为 summary；或至少识别 `@brief` 缺失时把第一个 `Description` 当作 summary。
- 为 `code/endcode`、`note`、`see`、`deprecated`、`warning` 建立显式映射表（`code` → `<code>` + `<![CDATA[`/闭合，或直接降级为纯文本行）；未知 `TagOther` 建议**保留为纯文本**而不是造标签。
- 对 `@ref X`/`@p X` 做参数名替换（`<paramref name="X"/>` / `<c>X</c>`）而不是 `<ref>`/`<p>`。

**验证方式**
`grep -n "TagOther\|TagDefault" Source/ScriptCodeGenerator/Private/FDoxygenConverter.cpp`（命中 394、396、449、451、458 等）；对 `/** 普通注释 */` 与 `/** @brief A @code x @endcode */` 两种输入各跑一次转换，比对输出。

---

### [F-GEN3-006] `Lex` 的 `'/'` 分支按"直到下一个 `@` 或 `\`"吞字符，会跨行丢弃注释内容

- **类别**: Bug
- **严重度**: **P2**
- **复核结论**: **确认**（`'/` 分支谓词 `InChar != '@' && InChar != '\\'` 无换行判断，`EatWhile` 只在 `IsEof()` 停 → 必然跨行；该分支只在 `/` 位于 token 起点时触发）
- **可达性**: 活跃（`bIsGenerateFunctionComment=True`；触发条件是注释中任意"行首为 `/`"的续行）
- **复核证据**: `FDoxygenConverter.cpp:246-256` 逐字回读（`:248 Bump()`、`:250-253` 谓词、`:255 return Trivial`）；`FTextReader::EatWhile` `:139-151`（`:143 while (!IsEof() && InFunction(GetCurrentChar()))` 确认无其它终止条件）；`Lex` 入口 `:174 ResetBuffer()` + `:176 switch (GetCurrentChar())` 确认只在 token 起点触发
- **级别变动**: 无（维持 P2）
- **文件**: `Source/ScriptCodeGenerator/Private/FDoxygenConverter.cpp:246-256`
- **函数**: `Lex(FTextReader&, EExpect&)`
- **置信度**: 高（谓词无换行判断）；实际触发频率未实测，标中

**现状（代码事实）**
```cpp
// FDoxygenConverter.cpp:246-256
	case '/':
		{
			OutTextReader.Bump();

			OutTextReader.EatWhile([](const int InChar)
			{
				return InChar != '@' && InChar != '\\';   // 252：没有 '\n'/'\r' 判断
			});

			return ETokenKind::Trivial;                   // 255：整段被丢弃
		}
```
`EatWhile` 只在 `IsEof()` 时停止（`:139-151`），因此该谓词会一路吃到下一个 `@`/`\`，**跨越任意多行**。

**调用上下文**
`Tokenize`（`:311-339`）→ `Lex`；输入为 UHT `Comment` 元数据（`FClassGenerator.cpp:461`）。

**问题**
- 注释里任何**行首是 `/` 的续行**（如 `@param Path /Game/Foo` 的换行续写、路径/URL 独占一行、`//` 风格补充说明）会触发该分支，把它之后直到下一个标签之间的**所有行**吞掉 → 描述文本成段丢失。
- 反过来，因为该分支只在 `'/'` 位于 token 起点时触发，同一段文本里 `/` 出现在行中时又不会触发，行为对输入排版敏感（同一份注释换个换行位置输出不同）。
- 这也解释了为什么"以 `/**` 开头、且整段没有任何标签"的注释一定输出空：首个 `/` 直接吞完整个文本。

**建议**
谓词加 `InChar != '\n' && InChar != '\r'`，并且不要返回 `Trivial` 丢弃内容——`/**`、`*/`、`///` 这类标记应只剥掉标记本身，其余按 `Description` 处理。

**验证方式**
构造 `/**\n * @brief A\n * /Game/SomePath\n * 末行说明\n */` 输入，检查输出是否含"末行说明"。

---

### [F-GEN3-007] `.sln`/`.props`/`.csproj` 全量覆盖式重写，且写入失败不可见；只有 `Game.csproj` 受保护

- **类别**: Bug（体验/数据丢失）
- **严重度**: **P2**
- **复核结论**: **确认**（14 次 `CopyTemplate` 中**确实只有 1 次**显式传 `false`（`:97-105` 的 Game.csproj，`false` 在 `:105`），其余 13 次走默认 `true`；两个重载的默认参数确在 `FSolutionGenerator.h:11/15`；`:143 FileManager.Copy` 与 `:162 SaveStringToFile` 返回值均未接收）
- **可达性**: 活跃（每次生成都重写；`bEnableDeleteProxyDirectory` 等设置不影响本路径）
- **复核证据**: `FSolutionGenerator.cpp:15-136` 14 次调用行号逐一回读（15/24/32/40/50/55/63/74/78/88/97/107/115/126），仅 `:105` 为 `false`；`:429-441`（`<!-- ... DO NOT modify this manually! -->`）、`:443-457`（`#` 注释 + `:445 #if !VS2026`）；产物侧：`Script/Interop.csproj:1-4`、`Script/CodeAnalysis/CodeAnalysis.csproj:1-4`、`Script/Shared.props:1-4` 均带 `<!-- -->` 头注释，而 `Script/Script.sln:1-5` **没有** `#` 头注释（与 `#if !VS2026` 被编译掉一致；`VSVersion.h:6 VS2026 = _MSC_VER >= 1950`，本机 MSVC 版本未验证，见 §5）
- **级别变动**: 无（维持 P2）
- **文件**: `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:139-145`、`:147-164`、`:97-105`；默认参数见 `Public/FSolutionGenerator.h:11`、`:15`
- **函数**: `FSolutionGenerator::CopyTemplate`（两个重载）、`FSolutionGenerator::Generator()`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FSolutionGenerator.h:11 / :15
	static void CopyTemplate(const FString& Dest, const FString& Src, bool bReplaceExistingFile = true);
	static void CopyTemplate(const FString& Dest, const FString& Src,
	                         const TArray<TFunction<void(FString& OutResult)>>& InFunction,
	                         bool bReplaceExistingFile = true);
```
```cpp
// FSolutionGenerator.cpp:139-145
	if (auto& FileManager = IFileManager::Get(); !FileManager.FileExists(*Dest) || bReplaceExistingFile)
	{
		FileManager.Copy(*Dest, *Src);      // 143 返回值未检查
	}
```
14 次调用中**只有** `Game.csproj` 显式传 `false`（`:105`），其余（含 `Script.sln` `:126-136`、`Shared.props` `:115-124`、`Game.props` `:107-113`、`Interop.csproj` `:78-86`、`UE.csproj` `:88-95`）都是默认 `true`＝**每次生成都重写**。

**调用上下文**
`FSolutionGenerator::Generator()`（`:9`）由编辑器/Commandlet 触发；用户拿这些文件在 Rider/VS 里开发（任务书背景）。

**问题**
- 覆盖是**有意的**（`.csproj`/`.props` 会被插入 `DO NOT modify this manually!` 头注释，见 `:429-441`、`:459-467`），但 `.sln` 是用户最常手改的文件（手动 Add 工程、调整启动工程、加 solution folder）——每次生成都会被还原，用户无提示。`:443-457` 的 `AddSolutionGeneratorHeaderComment` 在 `.sln` 里插入的是 `#` 注释，且被 `#if !VS2026` 整段排除（VS2026 下连提示都没有）。
- `Game.csproj` 走 `false` 分支，导致**模板升级永不生效**：用户在插件更新后仍用旧 `Game.csproj`，且没有任何"模板版本落后"的检测。
- 写失败（VS 打开着 `.sln` 会持锁）在 `:162` 被静默吞掉，用户看到的是"生成成功"但文件未变。
- `FileManager.Copy`（`:143`）返回值同样未检查。

**建议**
1. 对 `.sln` 采用"读入现有文件 → 只替换 UnrealCSharp 自己负责的段落"的增量策略，或在覆盖前备份 `*.sln.bak` 并 `UE_LOG(Warning)` 提示将被覆盖；
2. `Game.csproj` 增加模板版本标记注释（如 `<!-- UnrealCSharpTemplateVersion: 1.2.0 -->`），生成时若版本落后则提示而非静默保留；
3. 所有 `Copy`/`SaveStringToFile` 失败必须 `UE_LOG(LogScriptCodeGenerator, Error, ...)`。

**验证方式**
在已生成工程的 `Script.sln` 里加一个 `Project(...)`，再触发一次生成，观察是否被抹掉；对 `.sln` 加只读属性后生成，检查是否有任何日志输出。

---

### [F-GEN3-008] 所有替换都是"找不到占位符就静默不变"，模板与替换串之间没有任何一致性校验

- **类别**: Bug（隐患）/ 可读性
- **严重度**: **P2**
- **复核结论**: **确认**（`FString::Replace` 找不到目标即原样返回，无命中反馈；模板侧 6 个文件的占位符逐行回读，与替换串逐字一致，故当前"侥幸成立"）
- **可达性**: 活跃（每次生成都执行 12 组替换）
- **复核证据**: 模板逐行回读：`Template/Script.sln:19 ProjectPlaceholder`、`:42 {SolutionConfigurationPlatformsPlaceholder}`、`:6/:8` 与 `FSolutionGenerator.cpp:316/:328` 逐字一致；`Template/Game.csproj:2/:4` 与 `:179/:272` 一致；`Template/Shared.props:3/:7/:18` 与 `:260/:223/:248` 一致；`Template/Game.props:21` 与 `:228` 一致；`Template/Interop.csproj:10/:19` 与 `:260/:238` 一致；`Template/UE.csproj:4` → `ReplacePluginBaseDir`。产物侧：`Script/Shared.props:12` 已含 `WITH_LEANCLR`（`bIsGenerateFunctionComment` 无关），且该 .props **不含** `WITH_EDITOR` → 说明本次产物是在 **Cook 期间**生成的（`:207 IsRunningCookCommandlet()` 为真，与 `UnrealCSharpEditor.cpp:158-180` 的 Cook 分支一致）
- **级别变动**: 无（维持 P2）
- **文件**: `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:166-467`（全部 `Replace*` 函数）
- **函数**: `ReplacePluginBaseDir`/`ReplaceImport`/`ReplaceDefineConstants`/`ReplaceScriptOutputPath`/`ReplaceOutputPath`/`ReplaceHintPath`/`ReplaceTargetFramework`/`ReplaceProjectReference`/`ReplaceYield`/`ReplaceGameName`/`ReplaceDefinition`/`ReplaceProject`/`ReplaceProjectPlaceholder`/`ReplaceSolutionConfigurationPlatformsPlaceholder`/`ReplaceScriptPath`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FSolutionGenerator.cpp:270-281（典型形态：精确匹配 + 静默替换）
void FSolutionGenerator::ReplaceProjectReference(FString& OutResult)
{
	OutResult = OutResult.Replace(TEXT("<ProjectReference Include=\"\" />"),   // 272 精确串
	                              *FString::Printf(TEXT(
		                              "<ProjectReference Include=\"..\\%s\\%s%s\" />"
	                              ),
	                                               *FUnrealCSharpFunctionLibrary::GetUEName(),
	                                               *FUnrealCSharpFunctionLibrary::GetUEName(),
	                                               *PROJECT_SUFFIX

	                              ));
}
```
模板侧对应串已核对（`Template/Game.csproj:2` `<Import Project="" Condition="Exists('')" />`、`:4` `<ProjectReference Include="" />`；`Template/Script.sln:6`、`:8`、`:19`、`:42` 与 `:316`、`:328`、`:376`、`:416` 的匹配串逐字一致）。

**调用上下文**
`Generator()`（`:9-137`）中的 12 次带替换列表的 `CopyTemplate` 把 `Replace*` 作为 `TFunction<void(FString&)>` 列表传入。

**问题**
`FString::Replace` 找不到目标就原样返回，**没有命中数反馈**。任何一个模板文件被改一个空格/换行（尤其 `.sln` 的一批替换依赖 `\r\n`：`:373-377`、`:415-420`），生成结果里就会残留 `ProjectPlaceholder`、`{SolutionConfigurationPlatformsPlaceholder}` 或空 `""` 名字的 `Project()` 行，导致 `.sln` 无法打开——而生成流程不会有任何报错。此外 `ReplaceScriptOutputPath`/`ReplaceOutputPath`/`ReplaceHintPath` 硬编码 `..\..\Content\`（`:230`、`:240`、`:250`），一旦 `GetPublishDirectory()`/`GetScriptDirectory()` 被改成多级目录或目录层级变化，替换照样"成功"但路径错误（无存在性校验）。

**建议**
把"是否命中"变成可断言：`Replace` 前后比较长度或在每个 `Replace*` 里检查 `OutResult.Contains(占位符)` 并 `UE_LOG(Error)`；或者更彻底地把模板从"字符串替换"改为结构化生成。

**验证方式**
`grep -n "ProjectPlaceholder\|SolutionConfigurationPlatformsPlaceholder" Template/Script.sln Source/.../Macro.h` 对照；把 `Template/Script.sln` 换行改成 LF 后触发生成，检查产物是否残留占位符（只读验证）。

---

### [F-GEN3-009] `FAssetGenerator` 用**静态裸 UObject 指针数组**跨阶段延迟枚举，GC 可回收 → 悬挂指针

- **类别**: 内存/资源泄漏（GC 悬挂）
- **严重度**: **P2**
- **复核结论**: **部分确认**（偏差：**触发机制描述不成立** —— 本次生成的调用树内**没有任何 GC 触发点**，"大量分配完全可能触发 GC"与 UE 的 GC 模型不符（UE 的 GC 只在显式 `CollectGarbage`/引擎 Tick 增量 GC 时运行，不按分配量触发）；且"指针跨调用残留"路径**不可达**。**但"静态非 UPROPERTY 裸指针数组、无 `AddReferencedObjects`"是确定事实**，因而仍是一个真实的 GC 保护缺失/静态可变状态隐患）
- **可达性**: 活跃（`bIsGenerateAsset=True`；`UnrealCSharpEditor.cpp:361` 每次全量生成都会执行）
- **复核证据**:
  - 事实：`FAssetGenerator.cpp:19`（定义）、`:47-50`（消费循环）、`:52 Empty()`、`:108 Emplace()`；`Public/FAssetGenerator.h:16` 声明 —— 逐字回读吻合，`grep UserDefinedEnums` 于 `Source/ScriptCodeGenerator` 命中 **5 处（h:16、cpp:19/47/52/108）**，与 §4 表一致（原文 §4 表把 `:47` 写成 `:48`，±1）
  - 反证（机制）：`grep CollectGarbage` 于 `Source/` 只有 **2 处真调用**：`UnrealCSharpEditor.cpp:384`（在 `:380 EndGenerator()` 之后、即 `FAssetGenerator::Generator()` **之后**）与 `FDynamicGenerator.cpp:93`（由 `:343 FDynamicGenerator::CodeAnalysisGenerator()` 调用，同样在 `:361` **之前**）；`FClassGenerator`/`FStructGenerator`/`FEnumGenerator` 调用树内 0 处。引擎侧：`Runtime/CoreUObject/Private/UObject/` 下 `CollectGarbage` 仅出现在 `GarbageCollection.cpp`（定义）、`PackageReload.cpp:665/820`（热重载）、`UObjectBase.cpp:1187`（`RemoveLoadedUObjects`，模块重载路径）——`LoadObject`/`ResolveName2`（`UObjectGlobals.cpp:1381-1391`）不触发 GC
  - 反证（残留）：收集（`:44` 全量路径传 `true`）与清空（`:52`）在**同一个 `IsGenerateAsset()` 门（`:26/:62`）内的同一同步调用中**，两处之间无 `return`、无异常路径；`bIsGenerateAsset` 在一次调用内不可能由真变假，故"`Empty()` 不执行导致跨调用残留"不成立
  - 交叉核对（只读，不改 08）：`08-专项审计/02b-UObject生命周期与绑定配对审计.md:153-168` 的裸指针成员表**未收录** `FAssetGenerator::UserDefinedEnums`（对该符号 grep `UserDefinedEnums|FAssetGenerator` 于 02b **0 命中**）——该表口径（"无 UPROPERTY / 无 `AddReferencedObjects` / 非 `FGCObject` → GC 后悬垂"）本应覆盖本项，属跨报告覆盖缺口
- **级别变动**: 无（维持 P2 —— 机制虽被证伪，但"GC 无保护的静态裸 UObject 指针 + 静态可变状态（不可重入/不可并行）"本身即 `_CONVENTIONS.md` 定义的"隐患"，且 `08-.../02b` 同类静态裸指针容器定为 P1/P2）
- **文件**: `Source/ScriptCodeGenerator/Private/FAssetGenerator.cpp:19`、`:47-52`、`:106-113`；声明 `Public/FAssetGenerator.h:16`
- **函数**: `FAssetGenerator::Generator()`、`FAssetGenerator::Generator(const FAssetData&, bool)`
- **置信度**: 中（GC 是否恰好在两次循环之间触发的时机不可确定；指针无 UPROPERTY 保护是确定事实）

**现状（代码事实）**
```cpp
// FAssetGenerator.cpp:19
TArray<UUserDefinedEnum*> FAssetGenerator::UserDefinedEnums;   // 静态裸指针，非 UPROPERTY，无 AddReferencedObjects
```
```cpp
// FAssetGenerator.cpp:42-52
			for (const auto& AssetData : OutAssetData)
			{
				Generator(AssetData, true);       // 44 → 内部 Emplace 到静态数组
			}

			for (const auto& UserDefinedEnum : UserDefinedEnums)
			{
				FEnumGenerator::Generator(UserDefinedEnum);   // 49 使用可能已被 GC 回收的对象
			}

			UserDefinedEnums.Empty();             // 52 只在 IsGenerateAsset() 为真时清空
```
```cpp
// FAssetGenerator.cpp:106-109
							if (bDelayedGeneratorUserDefinedEnum)
							{
								UserDefinedEnums.Emplace(UserDefinedEnum);   // 108 先存，后统一用
							}
```
收集与消费是**两个分离的循环**，中间隔着一整轮 `FClassGenerator::Generator/…` 调用（大量 `FString`/容器分配，完全可能触发 GC）。

**调用上下文**
`Generator()`（public 导出，`FAssetGenerator.h:8`）→ `Generator(FAssetData, bool)`（public 导出，`:10`，默认 `bDelayedGeneratorUserDefinedEnum = false`），遍历 AssetRegistry 得到的资产。

**问题**
`LoadObject` 得到的 `UUserDefinedEnum*` 只被一个**非 UPROPERTY 的静态 TArray** 持有，GC 不认这个引用。在第二次循环之前若发生 GC（例如中间某个蓝图生成分配触发了 `CollectGarbage`），`UserDefinedEnum` 就变成已回收内存，`FEnumGenerator::Generator` 解引用它 = 野指针崩溃/数据损坏。另外：该静态数组是全局可变状态，`IsGenerateAsset()` 为假时第 52 行的 `Empty()` 不会执行，指针会跨调用残留（若中途 return 或异常，同样残留）；这也使该生成流程不可重入/不可并行。
（若 UE 侧对该资产包有强引用链——例如资产仍被 AssetRegistry 的 `FAssetData` 或包对象持有时——实际回收概率会降低；故标 P2 + 置信度中。）

**建议**
改用 `TArray<TStrongObjectPtr<UUserDefinedEnum>>`（或 `TWeakObjectPtr` + 使用前 `IsValid()` 校验），并在 `Generator()` 入口先 `UserDefinedEnums.Empty()`，保证任何路径都会清空。更简单：去掉延迟机制，直接在两阶段之间用 `TStrongObjectPtr` 数组传递。

**验证方式**
`grep -rn "UserDefinedEnums" Source/` → 声明 1 处 + 使用 3 处（19/48/52/108）；加 `AddReferencedObjects` 或用 `gc.StressCollect` 复现。

---

### [F-GEN3-010] 资产 → C# 标识符映射链上的三个静默失效点（`Game` 子串替换、命名空间未清洗、类型同名缺后缀）

- **类别**: Bug（静默重名/错位）
- **严重度**: **P2**
- **复核结论**: ⚠️部分 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Source/ScriptCodeGenerator/Private/FAssetGenerator.cpp:140-191`（使用点）；映射规则实现见 `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:661-720`
- **函数**: `FAssetGenerator::GeneratorAsset(const FAssetData&)`、`FUnrealCSharpFunctionLibrary::GetAssetName/GetFileName/GetSuffixName`
- **置信度**: 高（`Replace` 为全量替换，可从代码直接推演）；"实际项目是否踩中"取决于资产目录命名

**现状（代码事实）**
```cpp
// FAssetGenerator.cpp:140-148
	const auto ClassName = FUnrealCSharpFunctionLibrary::GetAssetName(InAssetData, InAssetData.AssetName.ToString());  // 140

	FString UsingNameSpaceContent;

	const auto NameSpaceContent = FString::Printf(TEXT("%s%s"),                                                       // 144
	                                              *NAMESPACE_ROOT,
	                                              *InAssetData.PackagePath.ToString().Replace(TEXT("/"), TEXT("."))); // 148 路径→命名空间，无字符清洗
```
```cpp
// FUnrealCSharpFunctionLibrary.cpp:703-715
FString FUnrealCSharpFunctionLibrary::GetFileName(const FAssetData& InAssetData, const FString& InAssetName)
{
	auto ModuleName = InAssetData.PackagePath.ToString().Replace(TEXT("Game"), FApp::GetProjectName());   // 710 全量子串替换

	auto DirectoryName = FPaths::Combine(GetGenerationPath(ModuleName), ModuleName);                      // 712

	return FPaths::Combine(DirectoryName, GetAssetName(InAssetData, InAssetName) + CSHARP_SUFFIX);         // 714
}
```
```cpp
// FUnrealCSharpFunctionLibrary.cpp:661-678
FString FUnrealCSharpFunctionLibrary::GetSuffixName(const FAssetData& InAssetData)
{
	return InAssetData.GetClass() != nullptr &&
	       (… UBlueprint … UWidgetBlueprint … UAnimBlueprint)      // 664-666 只有这三类加后缀
		       ? TEXT("_C")
		       : TEXT("");
}
```

**调用上下文**
`FAssetGenerator::GeneratorAsset`（`:126`）由 `Generator(FAssetData,bool)` 的 `else` 分支（`:118`）调用；输出经 `FGeneratorCore::AddGeneratorFile`（`:189`）登记、`SaveStringToFile`（`:191`）落盘。

**问题**
1. `:710` 用的是 `FString::Replace`（替换**所有**子串），不是前缀替换。项目名若含 `Game`（如工程 `MyGame`）或资产目录名含 `Game`（`Gameplay`/`GameModes`/`GameFeatures` —— UE 项目里极常见），路径会被改写：`/Game/Gameplay/A` → `/MyGame/MyGameplay/A`。结果是 `.cs` 落盘目录与 `:148` 算出的命名空间**互相不一致**（后者不做该替换），并且目录名被污染。
2. `:148` 直接把浮点路径 `/Game/Foo-Bar/My Asset` 变成命名空间 `..Game.Foo-Bar.My Asset`：文件夹名含空格/连字符/数字开头时生成**非法 C# 命名空间**，文件写盘成功但编译失败。`InAssetData.AssetName` 在 `:140` 走了 `Encode(...)`（会处理非法字符），但 `PackagePath` 这条路径完全没清洗。
3. `GetSuffixName` 只给 Blueprint/WidgetBlueprint/AnimBlueprint 加 `_C`。若同一目录下存在 `BP_Foo`（蓝图）与 `Foo`（数据资产/UserDefinedStruct 等非上述三类），`GetAssetName` 分别得到 `BP_Foo_C` 与 `Foo`（不同名，安全）；但**同名同目录不同类型**（如蓝图 `Foo` 与非蓝图资产 `Foo` —— UE 允许不同类资产重名吗？同包内不允许，跨包允许）在**不同目录但同命名空间**时才会撞名，此时 `:170` 生成的类名与 `:189` 的文件名都会冲突/互相覆盖，且**没有任何冲突检测**。

**建议**
1. `:710` 改为只在路径前缀 `/Game` 上替换（`ModuleName.Replace(TEXT("/Game/"), *("/" + FApp::GetProjectName() + "/"))` 或先 `StartsWith(TEXT("/Game"))` 判断）；
2. 命名空间生成前做一次标识符清洗（非法字符→`_`，数字开头加 `_`，检查 C# 关键字）；
3. 在 `AddGeneratorFile` 处对重复文件名/重复类名给出 `UE_LOG(Warning)` 并让用户可见。

**验证方式**
`grep -n "Replace(TEXT(\"Game\")" Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp`；在 `/Game/Gameplay` 下放一个蓝图生成一次并检查落盘目录名。

---

### [F-GEN3-012] `FCodeAnalysis::Compile()` 每次代码生成都同步跑一遍 `dotnet build`，且进程退出码/输出被完全丢弃

- **类别**: 性能 / 异常与错误处理
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Source/ScriptCodeGenerator/Private/FCodeAnalysis.cpp:39-52`（`Compile`）、`:34-36`、`:86-88`（回调丢参）
- **函数**: `FCodeAnalysis::Compile()`、`FCodeAnalysis::Analysis(const FString&)`、`FCodeAnalysis::Analysis()`
- **置信度**: 高（进程调用与空回调是确定事实；具体耗时取决于项目，未实测）

**现状（代码事实）**
```cpp
// FCodeAnalysis.cpp:39-52
void FCodeAnalysis::Compile()
{
	static auto CompileTool = FUnrealCSharpFunctionLibrary::GetDotNet();      // 41

	const auto CompileParam = FString::Printf(TEXT(
		"build \"%s\" --nologo -c Debug"                                     // 44
	),
	                                          *FUnrealCSharpFunctionLibrary::GetCodeAnalysisProjectPath()
	);

	FUnrealCSharpFunctionLibrary::SyncProcess(CompileTool, CompileParam, [](const int32, const FString&)   // 49
	{
	});                                                                       // 49-51 退出码与输出全丢
}
```
```cpp
// FCodeAnalysis.cpp:5-10
void FCodeAnalysis::CodeAnalysis()
{
	Compile();       // 7 先编译分析器工程

	Analysis();      // 9 再跑分析
}
```

**调用上下文**
`FCodeAnalysis::CodeAnalysis()`（public 导出，`FCodeAnalysis.h:6`）→ `Compile()` + `Analysis()`；`Analysis(const FString&)`（`:8`，public 导出）单独分析一个文件。异步性：`SyncProcess` 是**同步阻塞**调用（名字即含义，见 `FUnrealCSharpFunctionLibrary.cpp:1520`），若从编辑器 UI 线程/GameThread 触发会卡界面。

**问题**
- `dotnet build ... -c Debug` 是 MSBuild 全量/增量构建，冷启动通常数秒~数十秒。它被放在**每次**代码生成的开头同步执行，而生成流程本身可能在编辑器里被反复触发（每次新增类/每次 Commandlet）。没有任何"自上次以来 CodeAnalysis 源码是否变化"的判断，也没有产物时间戳比较。
- 空 lambda 丢弃 `(int32 ExitCode, const FString& Output)` 两个参数：分析器编译失败（例如用户没装对应 dotnet SDK）时**没有任何错误输出**，后续 `Analysis()` 仍会尝试运行不存在的 exe，同样静默失败 → 用户观察到的是"注释/分析结果莫名缺失"，无任何线索。
- `Analysis()`（`:54-89`）里 exe 路径由 `FPaths::Combine(GetCodeAnalysisCSProjPath(), CODE_ANALYSIS_NAME + ".exe")` 拼出（`:56-66`），不校验文件是否存在；参数用 `FString::Printf("... \"%s\" \"%s\"")` 拼接（`:68-75`），路径含 `"` 时无转义（Windows 路径一般不含引号，风险低）。
- `GetDotNet()` 被 `static` 缓存（`:41`）——会话内切换 dotnet 版本不会生效（与 `GetGenerationPath` 同类问题）。

**建议**
- 用产物时间戳比较（`CodeAnalysis.dll`/`.exe` 的 `GetTimeStamp()` vs `Script/CodeAnalysis/*.cs` 的最新时间戳）跳过无变化的 `dotnet build`；
- 回调里检查 exit code 并 `UE_LOG(LogScriptCodeGenerator, Error/Warning, ...)` 转发输出；`Analysis()` 前 `FileExists` 校验并给出明确提示；
- 若触发点可能在 GameThread，改为可取消的异步任务（`Async(EAsyncExecution::ThreadPool, ...)`）或至少在耗时段落显示进度。

**验证方式**
`grep -n "SyncProcess" Source/ScriptCodeGenerator/Private/FCodeAnalysis.cpp`（命中 34、49、86，回调全为空 lambda）；`grep -rn "CodeAnalysis()" Source/` 找出全部触发点并统计单次生成耗时。

---

### [F-GEN3-013] `FDoxygenConverter` 的词法层在"无标签区间"逐字符产出 token，且全局符号未 `static`

- **类别**: 性能 / 可读性 / 代码卫生
- **严重度**: **P3**（性能部分接近 P3，符号污染部分为隐患）
- **复核结论**: ⚠️部分 —— 严重度由 P2 校正为 **P3**；可达性 活跃
- **文件**: `Source/ScriptCodeGenerator/Private/FDoxygenConverter.cpp:298-308`、`:311-339`、`:172`、`:311`、`:341`
- **函数**: `Lex`、`Tokenize`、`Slice`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FDoxygenConverter.cpp:298-305（Expect==None 时每字符一次 Lex/Emplace）
	case EExpect::None:
		{
			OutTextReader.NextChar();
			...
```
```cpp
// FDoxygenConverter.cpp:319-336
	while (!TextReader.IsEof())
	{
		auto TokenKind = Lex(TextReader, Expect);
		...
		TokenData.Emplace(TokenKind, TokenRange);   // 335 每个 token 一次 TArray 扩容
	}
```
```cpp
// FDoxygenConverter.cpp:330-333 —— 这个"空 range"防护实际永不成立
		if (TokenRange.IsEmpty())
		{
			break;
		}
```
```cpp
// FDoxygenConverter.cpp:172 / 311 / 341 —— 三个自由函数都在全局命名空间且没有 static
ETokenKind Lex(FTextReader& OutTextReader, EExpect& OutExpect)
TArray<FTokenData> Tokenize(const FStringView& InText)
FStringView Slice(const FStringView& InText, const FTextRange& InTextRange)
```

**调用上下文**
`FDoxygenConverter::operator()`（`:351`）→ `Tokenize`（`:363`）→ `Lex`；每个函数一次（`FClassGenerator.cpp:465`），所以性能影响是每函数 O(注释长度) 的常数级开销，不构成热路径问题。

**问题**
1. **性能**：`Expect == None` 的区间（即注释开头到第一个 Doxygen 标签之间的所有自由文本）会**每字符**产生一个 `FTokenData`（`FTokenData` 内含 `FTextRange`，`TArray` 反复扩容）。对一个 200 字符的普通注释就是 200 次 `Emplace` + 扩容，而这些 token 在 `operator()` 里全部被 `default:` 丢弃（`:415-418`）。应改为一次 `EatWhile` 跳到下一个 `@`/`\`。
2. **`:330-333` 是无效防护**：`GetTokenRange()` 返回 `(BeginIndex, EndIndex+1)`，未 `Save()` 时为 `(0,1)`，长度恒 ≥1，`IsEmpty()` 永不成立——它既不能防死循环（靠 `while(!IsEof())` + `NextChar` 保证），也防不住 F-GEN3-002 的错切片。
3. **符号污染**：`Lex`/`Tokenize`/`Slice` 在 `.cpp` 里是**外部链接**的全局函数（缺 `static`），名字极通用。它们会进入模块符号表，若同模块其它 `.cpp` 出现同名自由函数即为 ODR/重定义链接错误。（本仓库内未发现同名定义，见 §6 证据，故仅作隐患记录。）

**建议**
`:298` 的 `None` 分支改成一个 `EatWhile(InChar != '@' && InChar != '\\')`，并只产出一个 `Trivial`；三个自由函数加 `static` 或放进匿名 namespace；删除无效的 `IsEmpty()` 检查或修正 `GetTokenRange()` 语义使其真的能为空。

**验证方式**
`grep -rn "^ETokenKind Lex\|^TArray<FTokenData> Tokenize\|^FStringView Slice" Source/` 确认无 `static`；`grep -rn "\bSlice\b" Source/` 检查是否有同名符号。

---

### [F-GEN3-015] `Interop.csproj` 里的插件相对路径没有走 `ReplacePluginBaseDir`，与 `UE.csproj` 处理不对称

- **类别**: Bug / 平台兼容
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:78-86`（Interop 调用点）、`:88-95`+`:166-175`（UE 调用点与替换函数）；模板 `Template/Interop.csproj:7`、`Template/UE.csproj:4`
- **函数**: `FSolutionGenerator::Generator()`、`FSolutionGenerator::ReplacePluginBaseDir(FString&)`
- **置信度**: 高（两个模板的对应行与两个调用点的替换列表都已逐行核对）

**现状（代码事实）**
```cpp
// FSolutionGenerator.cpp:78-86  Interop：只有 3 个替换，没有 ReplacePluginBaseDir
	CopyTemplate(
		FUnrealCSharpFunctionLibrary::GetInteropProjectPath(),
		TemplatePath / INTEROP_NAME + PROJECT_SUFFIX,
		TArray<TFunction<void(FString& OutResult)>>
		{
			&FSolutionGenerator::ReplaceOutputPath,           // 83
			&FSolutionGenerator::ReplaceTargetFramework,      // 84
			&FSolutionGenerator::AddProjectGeneratorHeaderComment   // 85
		});
```
```cpp
// FSolutionGenerator.cpp:88-95  UE：有 ReplacePluginBaseDir
	CopyTemplate(
		FUnrealCSharpFunctionLibrary::GetUEProjectPath(),
		TemplatePath / DEFAULT_UE_NAME + PROJECT_SUFFIX,
		TArray<TFunction<void(FString& OutResult)>>
		{
			&FSolutionGenerator::ReplacePluginBaseDir,        // 93
			&FSolutionGenerator::AddProjectGeneratorHeaderComment
		});
```
```cpp
// FSolutionGenerator.cpp:166-175
void FSolutionGenerator::ReplacePluginBaseDir(FString& OutResult)
{
	auto Index = 0;

	const auto PluginBaseDir = FUnrealCSharpFunctionLibrary::GetPluginBaseDir();

	PluginBaseDir.FindLastChar('/', Index);                                        // 172

	OutResult = OutResult.Replace(*PLUGIN_NAME, *PluginBaseDir.Right(PluginBaseDir.Len() - Index - 1));  // 174 全局替换 "UnrealCSharp"
}
```
```xml
<!-- Template/Interop.csproj:7 —— 插件路径写死，且不会被打补丁 -->
    <Compile Include="..\..\Plugins\UnrealCSharp\Script\Interop\**" Exclude="..\..\Plugins\UnrealCSharp\Script\Interop\**\*.DS_Store"></Compile>
<!-- Template/UE.csproj:4 —— 同样的写死路径，但会经 ReplacePluginBaseDir 修正 -->
    <Compile Include="..\..\Plugins\UnrealCSharp\Script\UE\**" Exclude="..\..\Plugins\UnrealCSharp\Script\UE\**\*.DS_Store"></Compile>
```

**调用上下文**
`FSolutionGenerator::Generator()`（`:9`）← `FUnrealCSharpEditorModule::Generator()`（`UnrealCSharpEditor.cpp:335`）← 编辑器工具栏/console 命令 `UnrealCSharp.Editor.Generator`（`UnrealCSharpEditor.cpp:119-125`）/ 右键 New Class / `UGeneratorScriptCodeCommandlet::Main`（`GeneratorScriptCodeCommandlet.cpp:30`）。

**问题**
两个 csproj 用**同样方式**引用插件目录下的 C# 源码（`<Compile Include="..\..\Plugins\UnrealCSharp\Script\<X>\**">`），但：
- `UE.csproj` 会执行 `ReplacePluginBaseDir`，把 `UnrealCSharp` 换成插件目录名 → 插件目录被改名（例如 `UnrealCSharp_1.2.0`、`MarketplaceCopy`）时仍然可用；
- `Interop.csproj` 不做这个替换 → 目录一改名，`:7` 的 glob 匹配不到任何文件。MSBuild 对空 glob **不报错**，于是 `Interop.csproj` 会以"0 个源文件"编译，产出一个空的 `Interop.dll`，而 C# 运行时的 `Interop/*`（`HandleData`、各 Bridge、`AssemblyLoader`）全部缺失 → 表现为运行期大面积 `TypeLoadException`/绑定失败，且生成阶段毫无提示。
- 两种安装位置（`<Project>/Plugins/UnrealCSharp` 与 `Engine/Plugins/...`）里，`ReplacePluginBaseDir` 只修**目录名**不修**位置**：`..\..\Plugins\` 这个前缀对装在 Engine 下的插件永远不成立。即"改了名能救、换了位置救不了"。

**建议**
- 给 Interop 的 `CopyTemplate` 也加上 `&FSolutionGenerator::ReplacePluginBaseDir`；
- 把模板里的 `..\..\Plugins\<Name>\` 前缀也做成占位符（例如 `PluginBaseDirPlaceholder`），生成时用 `FPaths::ConvertRelativePathToFull(GetPluginDirectory())` 相对 `<Project>/Script/<Proj>/` 计算真实相对路径（含 Engine 安装情形）；
- 生成后校验 glob 是否命中：至少检查目标目录存在，否则 `UE_LOG(Error)`。

**验证方式**
`grep -n "ReplacePluginBaseDir" Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp`（命中 93 与 166，**Interop 的 78-86 里没有**）；把插件目录改名后生成，检查 `<Project>/Script/Interop/Interop.csproj` 的 `Compile Include` 是否仍指向 `UnrealCSharp`。

---

### [F-GEN3-016] `ReplacePluginBaseDir` 的 `FindLastChar` 结果未校验，且做的是全局字符串替换

- **类别**: Bug（隐患）
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度由 P2 校正为 **P3**；可达性 活跃
- **文件**: `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:166-175`
- **函数**: `FSolutionGenerator::ReplacePluginBaseDir(FString&)`
- **置信度**: 中（`IPlugin::GetBaseDir()` 在 UE 里通常返回正斜杠路径，故当前大概率不触发；属防御性缺陷）

**现状（代码事实）**
```cpp
// FSolutionGenerator.cpp:168-174
	auto Index = 0;

	const auto PluginBaseDir = FUnrealCSharpFunctionLibrary::GetPluginBaseDir();

	PluginBaseDir.FindLastChar('/', Index);     // 172 返回 bool，未检查；找不到时 Index 仍为 0

	OutResult = OutResult.Replace(*PLUGIN_NAME, *PluginBaseDir.Right(PluginBaseDir.Len() - Index - 1));  // 174
```
`GetPluginBaseDir()` 实现见 `FUnrealCSharpFunctionLibrary.cpp:903-906`（`IPluginManager::Get().FindPlugin(PLUGIN_NAME)->GetBaseDir()`，且 `FindPlugin` 的返回值也未判空）。

**问题**
1. `FindLastChar` 的返回值被忽略：若 `GetBaseDir()` 返回的是不含 `/` 的路径（例如相对目录形式 `UnrealCSharp`），`Index` 保持 0，`Right(Len - 0 - 1)` 得到"去掉第一个字符的整条路径"，会被拼进 `UE.csproj` 的 `Compile Include` 里形成垃圾路径。
2. `Replace` 是**替换所有出现**，不是替换那一个路径片段：模板里任何其它位置的 `UnrealCSharp`（命名空间、程序集名、注释）都会被一并改掉。当前 `Template/UE.csproj` 只有第 4 行含该串（已核对全文 9 行），所以现在安全，但这是"模板一改就炸"的隐式契约。
3. `FindPlugin(PLUGIN_NAME)` 未判空（Core 侧，`FUnrealCSharpFunctionLibrary.cpp:905`）：`GetPluginTemplateDirectory()`/`GetPluginScriptDirectory()` 都经它，而 `FSolutionGenerator::Generator()` 第一行就调用（`:11`、`:13`）；若插件被识别为不同名字则空指针解引用。

**建议**
`FindLastChar` 返回值检查 + 找不到时 `return`；用 `FPaths::GetCleanFilename` 或 `FPaths::GetPathLeaf` 取末段；把模板里的路径改成专用占位符而不是靠"名字串替换"。

**验证方式**
`grep -n "FindLastChar" Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp`；在模板里加入一个含 `UnrealCSharp` 的命名空间后生成，观察是否被误改。

---

### P3

### [F-GEN3-011] 资产文件写盘失败仍被登记为"已生成"；跨会话的资产删除依赖下一次全量生成才被清理

- **类别**: 死代码 / 异常与错误处理
- **严重度**: **P3**（重命名/删除在**编辑器会话内已有专门处理**，故降级；仅"写盘失败被登记"与"跨会话删除"两条残留）
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Source/ScriptCodeGenerator/Private/FAssetGenerator.cpp:187-191`（写盘+登记）；清理侧 `Source/ScriptCodeGenerator/Private/FGeneratorCore.cpp:1042-1079`；会话内 rename/delete 处理在编辑器模块 `UnrealCSharpEditor/Private/Listener/FEditorListener.cpp:361-393`
- **函数**: `FAssetGenerator::GeneratorAsset`、`FGeneratorCore::EndGenerator`、`FGeneratorCore::DeleteRemainGeneratorFiles`
- **置信度**: 高（三条路径的代码都已读到；"跨会话"结论来自 `EndGenerator` 的 `bIsFull` 参数与调用点）

**现状（代码事实）**
```cpp
// FAssetGenerator.cpp:187-191
	const auto FileName = FUnrealCSharpFunctionLibrary::GetFileName(InAssetData);

	FGeneratorCore::AddGeneratorFile(FileName);                         // 189 登记"本次生成过"

	FUnrealCSharpFunctionLibrary::SaveStringToFile(FileName, Content);  // 191 返回值未检查
```
```cpp
// FGeneratorCore.cpp:1042-1047
	if (bIsFull)
	{
		DeleteRemainGeneratorFiles();      // 1044 只有全量生成才清理
	}

	GeneratorFiles.Empty();                // 1047
```
```cpp
// FEditorListener.cpp:521 / :552 —— 增量路径传 false，不触发清理
	FGeneratorCore::EndGenerator(false);
```
会话内 rename/delete **确实**被处理（这条我最初判断错了，代码事实证明已有实现）：
```cpp
// FEditorListener.cpp:373-384
void FEditorListener::OnAssetRenamed(const FAssetData& InAssetData, const FString& InOldObjectPath) const
{
	OnAssetChanged(InAssetData, [&]
	{
		if (FPlatformFileManager::Get().GetPlatformFile().DeleteFile(
			*FUnrealCSharpFunctionLibrary::GetOldFileName(InAssetData, InOldObjectPath)))   // 377-378 删除旧文件
		{
			FUnrealCSharpFunctionLibrary::MarkScriptChanged();
		}

		FAssetGenerator::Generator(InAssetData);                                            // 383 再生成新文件
	});
}
```

**调用上下文**
`FAssetGenerator::GeneratorAsset` ← `Generator(FAssetData,bool)` 的 `else` 分支（`FAssetGenerator.cpp:118`）← `Generator()`（`:361`，全量）或 `FEditorListener.cpp:357/383/391/504/514`（增量，`bDelayedGeneratorUserDefinedEnum=false`）。

**问题（修正后的两条）**
1. `:191` 写盘失败（长路径/只读/磁盘满）时文件不存在，但 `:189` 已把它登记进 `GeneratorFiles` → 同一轮 `EndGenerator(true)` 的 `DeleteRemainGeneratorFiles` 认为"这个文件是本次生成的"，不会补偿；用户得到的是"少了一个类"且无任何日志。
2. 资产在**编辑器未运行时**被删除/重命名（P4/git 同步、资源管理器删除）不会触发 `OnAssetRemoved`/`OnAssetRenamed`（`FEditorListener.cpp:344-350` 的委托是在 `OnFilesLoaded` 时才注册的），而增量路径用 `EndGenerator(false)` 不清理；默认配置 `bEnableDeleteProxyDirectory(false)`（`UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp:26`）也不会在 `OnBeginGenerator` 时清空 Proxy 目录。于是旧 `Foo_C.cs` 会一直留到下一次"全量生成"（New Class 向导 / `UnrealCSharp.Editor.Generator` 控制台命令 / Cook）为止；期间它带着指向已不存在资产的 `[PathName(...)]` 参与 C# 编译。

**建议**
`SaveStringToFile` 失败时 `UE_LOG(Error)` 并回滚登记；`EndGenerator(false)` 的增量路径也跑一次"仅针对本次涉及目录"的残留清理，或在编辑器启动的 `OnFilesLoaded` 之后做一次廉价的对账（扫描 Proxy 目录 vs AssetRegistry）。

**验证方式**
`grep -n "EndGenerator(" Source/UnrealCSharpEditor/Private/Listener/FEditorListener.cpp`（478/521、538/552 两处 false）；在编辑器关闭状态下删除一个已生成过 `.cs` 的蓝图资产，重开编辑器并触发一次增量编译，检查旧 `.cs` 是否仍在 `Script/*/Proxy/` 下。

---

### [F-GEN3-014] 逐函数清单中发现的低危项：未检查的 `GetClass()`、静默跳过的 LoadObject、重复的 `FPaths::Combine`

- **类别**: 可优化/可读性 / 异常与错误处理
- **严重度**: **P3**
- **复核结论**: ⚠️部分 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Source/ScriptCodeGenerator/Private/FAssetGenerator.cpp:64`、`:79`、`:128-133`、`:85-96`
- **函数**: `FAssetGenerator::Generator(const FAssetData&, bool)`、`FAssetGenerator::GeneratorAsset`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FAssetGenerator.cpp:64-84
			if (const auto AssetDataClass = InAssetData.GetClass())          // 64 有判空（好）
			{
				...
						if (const auto Blueprint = LoadObject<UBlueprint>(nullptr, *InAssetData.GetObjectPathString()))   // 72
						{
							if (const auto Class = Cast<UClass>(Blueprint->GeneratedClass))   // 79 Cast 结果有判空（好）
							{
								FClassGenerator::Generator(Class);                                 // 81
							}
						}
```
```cpp
// FAssetGenerator.cpp:126-138
	const auto SuperClass = InAssetData.GetClass();

	if (SuperClass == nullptr)      // 130 有判空（好）
	{
		return;
	}
```

**问题**
判空做得很完整（`:64`、`:79`、`:130` 都检查了），**没有任何一处是"判空后仍解引用"**。真实问题只有"静默"：`LoadObject` 失败（蓝图损坏/编译失败/Cooked 环境下 `GeneratedClass == nullptr`，`:79`）时直接跳过，`:72`、`:87`、`:99` 三处都没有 `UE_LOG`。用户只看到"少了几个类"，无法排查。此外：
- `:75`、`:80`、`:117` 的 `FPaths::Combine(X)` 只有一个参数（模板变参形式下等价于恒等调用），是多余包装；
- `:154-160` 用 `TSet<FString>` 收集 `using` 再遍历输出（顺序不确定但结果稳定，无重复，可接受）；`:144-148` 的 `FString::Printf("%s%s")` 双参拼接可直接用 `+` 或 `Append`（微优化）；
- `:33` `for (const auto& [Path] : ...GetSupportedAssetPath())` 只取 key，结构化绑定可读性一般。

**建议**
在三个 `LoadObject` 失败分支加 `UE_LOG(LogScriptCodeGenerator, Warning, TEXT("skip asset %s: load failed"), *InAssetData.GetObjectPathString());`；去掉单参 `FPaths::Combine`。

**验证方式**
`grep -n "LoadObject\|UE_LOG" Source/ScriptCodeGenerator/Private/FAssetGenerator.cpp` → 3 处 LoadObject、0 处 UE_LOG。

---

### [F-GEN3-017] `ScriptCodeGenerator.Build.cs` 未声明实际使用的模块依赖（`AssetRegistry`、`StructUtils`）

- **类别**: 平台兼容 / 构建正确性
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Source/ScriptCodeGenerator/ScriptCodeGenerator.Build.cs:25-50`；使用点 `Source/ScriptCodeGenerator/Private/FAssetGenerator.cpp:5`、`:13-17`、`:28-29`
- **函数**: `ScriptCodeGenerator(ReadOnlyTargetRules)`（模块规则构造）
- **置信度**: 高（依赖列表与 include/调用点都已逐行核对；"当前能否编译"取决于 UBT 的传递闭包，未实测）

**现状（代码事实）**
```csharp
// ScriptCodeGenerator.Build.cs:35-50
		PrivateDependencyModuleNames.AddRange(
			new string[]
			{
				"CoreUObject",
				"Engine",
				"GameplayTags",
				"Slate",
				"SlateCore",
				"UMGEditor",
				"UnrealEd", 
				"UnrealCSharpCore",
				"CrossVersion",
				"Json"
```
```cpp
// FAssetGenerator.cpp:5 / :13-17 / :28-29
#include "AssetRegistry/AssetRegistryModule.h"          // 5  ← AssetRegistry 模块，未在 Build.cs 中声明
#if UE_STRUCT_UTILS_U_USER_DEFINED_STRUCT
#include "StructUtils/UserDefinedStruct.h"              // 14 ← StructUtils 模块，未声明
#else
#include "Engine/UserDefinedStruct.h"
#endif
...
			const auto& AssetRegistryModule = FModuleManager::LoadModuleChecked<FAssetRegistryModule>(
				TEXT("AssetRegistry"));                     // 28-29 运行期按名字加载
```

**问题**
`AssetRegistry` 与 `StructUtils` 都不在 `PublicDependencyModuleNames`（`Build.cs:25-32`）或 `PrivateDependencyModuleNames`（`:35-50`）里。目前能编过是因为 `UnrealEd` 公开依赖 `AssetRegistry`、`Engine` 公开依赖 `StructUtils`，UBT 会把公开依赖的 include 路径传递下来——这是**隐式传递依赖**，一旦上游模块改成私有依赖（UE 大版本升级常见）或在 `bEnforceIWYU`/严格模式下，本模块会直接编译失败。`LoadModuleChecked(TEXT("AssetRegistry"))` 也把"依赖"变成了运行期字符串加载：模块缺失时会走 `check()` 崩溃而不是编译期报错。

**建议**
把 `"AssetRegistry"`（必需）与 `"StructUtils"`（版本条件）显式加入 `PrivateDependencyModuleNames`；`LoadModuleChecked` 可换成带日志的判定。

**验证方式**
`grep -n "AssetRegistry\|StructUtils" Source/ScriptCodeGenerator/ScriptCodeGenerator.Build.cs` → 0 命中；`grep -rn "AssetRegistryModule.h" Source/ScriptCodeGenerator/` → 1 命中。

---

### [F-GEN3-018] `ENUM_DEFAULT_MAX_INDEX` 宏名与实际语义不符（名为"MAX 索引"，值为 0 且被当作"默认值索引"）

- **类别**: 可读性 / 一致性（命名与代码不符）
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Source/ScriptCodeGenerator/Public/ScriptCodeGeneratorMacro.h:3`；使用点 `Source/ScriptCodeGenerator/Private/FClassGenerator.cpp:1106`、`:1116`、`:1192`
- **函数**: 宏本身；使用处为 `FClassGenerator::GetFunctionDefaultParam`/属性默认值生成
- **置信度**: 中（命名误导是确定的；"值 0 在 UE 语义上正确"是我根据 `GetDisplayNameTextByIndex` 用法的推断，未跑运行期验证）

**现状（代码事实）**
```cpp
// ScriptCodeGeneratorMacro.h:3（整个文件只有这一行有效内容）
#define ENUM_DEFAULT_MAX_INDEX 0
```
```cpp
// FClassGenerator.cpp:1115-1118 —— 元数据没有默认值时，回退到索引 0 的显示名
				return FString::Printf(TEXT(" = %s.%s"), *ByteProperty->Enum->GetName(), MetaData.IsEmpty()
						                       ? *ByteProperty->Enum->GetDisplayNameTextByIndex(ENUM_DEFAULT_MAX_INDEX).
						                                        ToString()
						                       : *MetaData);
```
（同文件 `:1106` `UserDefinedEnum->GetDisplayNameTextByIndex(ENUM_DEFAULT_MAX_INDEX)`、`:1192` `EnumProperty->GetEnum()->GetDisplayNameTextByIndex(ENUM_DEFAULT_MAX_INDEX)` 用法相同。）

**问题**
宏名 `ENUM_DEFAULT_MAX_INDEX` 读起来是"枚举 `_MAX` 哨兵项的下标"（UE 里 `_MAX` 通常是最后一个），但代码把它当作**默认值（第一个枚举项，值 0）的下标**使用，且值就是 `0`。UE 的 `UEnum` 对带 `_MAX` 的枚举，`_MAX` 位于末尾而非 0，因此名字与实际用途相反，后续维护者若"按名字理解"去改成 `_MAX` 的下标会引入行为变更。另外这个宏被放在 `ScriptCodeGeneratorMacro.h` 而只被 `FClassGenerator.cpp` 单点使用，属于过度集中。

**建议**
重命名为 `ENUM_DEFAULT_VALUE_INDEX`（或 `ENUM_FIRST_VALUE_INDEX`），并加一行注释说明"用于元数据未给出默认值时的回退显示名索引"。

**验证方式**
`grep -rn "ENUM_DEFAULT_MAX_INDEX" Source/` → 4 命中（1 定义 + 3 使用）；对照 `UEnum::GetDisplayNameTextByIndex` 的语义在 UE 源码中确认索引 0 是否为第一个枚举项。

---

## 4. 死代码清单

判定方法：`Get-ChildItem -Recurse -Include *.cpp,*.h`（排除 `Source/ThirdParty/`）+ `Select-String -Pattern "\b<符号>"` 统计命中数。
**结论：本组文件（`FCodeAnalysis` / `FDoxygenConverter` / `FSolutionGenerator` / `FAssetGenerator` / `ScriptCodeGeneratorMacro.h`）未发现死代码。** 每个符号的命中数都 ≥ 声明+定义+至少一个真实调用点：

| 符号 | 声明位置 | grep 命中数 | 判定 | 证据 |
|---|---|---|---|---|
| `FCodeAnalysis::CodeAnalysis()` | `Public/FCodeAnalysis.h:6` | `CodeAnalysis` 45（含 `FDynamicGenerator::CodeAnalysisGenerator`、路径串等同名干扰） | 活 | 调用点 `UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp:95`（`UnrealCSharp.Editor.CodeAnalysis` 控制台命令）、`Listener/FEditorListener.cpp:183`（`OnPostEngineInit`） |
| `FCodeAnalysis::Analysis(const FString&)` | `Public/FCodeAnalysis.h:8` | 同上（`Analysis` 16） | 活，仅 2 处外部调用 | `FEditorListener.cpp:330`、`ToolBar/UnrealCSharpBlueprintToolBar.cpp:80` |
| `FCodeAnalysis::Compile()` / `Analysis()`（private） | `Public/FCodeAnalysis.h:11,13` | 见上 | 活（内部） | `FCodeAnalysis.cpp:7` / `:9` 由 `CodeAnalysis()` 调用 |
| `FDoxygenConverter::operator()` + 构造函数 | `Public/FDoxygenConverter.h:8,10` | `FDoxygenConverter` 7 | 活 | 唯一调用点 `FClassGenerator.cpp:465`；`:2` 为 include |
| `FSolutionGenerator::Generator()` | `Public/FSolutionGenerator.h:8` | `Generator(` 128（宽泛） | 活 | `UnrealCSharpEditor.cpp:106`（控制台命令）、`:335`（主生成流程） |
| `FSolutionGenerator::CopyTemplate`（两个重载） | `Public/FSolutionGenerator.h:11,13` | 18 | 活 | `FSolutionGenerator.cpp:15-136` 内 14 次调用 |
| `FSolutionGenerator::Replace*` 15 个 / `Add*` 3 个（private static） | `Public/FSolutionGenerator.h:17-51` | 每个 3-9（`AddProjectGeneratorHeaderComment` 9、`AddCSharpGeneratorHeaderComment` 5 含 3 次使用、`ReplaceTargetFramework` 5 含 3 次使用、其余 3＝声明+取址+定义） | 活（仅经函数指针表在 `Generator()` 内被取址调用） | `FSolutionGenerator.cpp:18-136` 的 `TArray<TFunction<void(FString&)>>` 初始化列表 |
| `FAssetGenerator::Generator()` | `Public/FAssetGenerator.h:8` | `Generator(` 宽泛；`UserDefinedEnums` 5 | 活 | `UnrealCSharpEditor.cpp:361` |
| `FAssetGenerator::Generator(const FAssetData&, bool)` | `Public/FAssetGenerator.h:10` | 活 | 5 处调用 | `FEditorListener.cpp:357/383/391/504/514` |
| `FAssetGenerator::GeneratorAsset`（private） | `Public/FAssetGenerator.h:13` | 3 | 活（内部） | 仅 `FAssetGenerator.cpp:118` 调用 |
| `FAssetGenerator::UserDefinedEnums` | `Public/FAssetGenerator.h:16` | 5 | 活（但为静态裸指针，见 F-GEN3-009） | `FAssetGenerator.cpp:19/48/52/108` |
| `ENUM_DEFAULT_MAX_INDEX` | `Public/ScriptCodeGeneratorMacro.h:3` | 4 | 活 | `FClassGenerator.cpp:1106/1116/1192` |
| `Lex` / `Tokenize` / `Slice`（`FDoxygenConverter.cpp` 内自由函数） | 无头文件声明 | 10（`Lex(`1 + `Tokenize(`2 + `Slice(`7） | 活，但**未 `static`** → 见 F-GEN3-013 | `grep -n "^ETokenKind Lex\|^TArray<FTokenData> Tokenize\|FStringView Slice" Source/` 仅命中 `FDoxygenConverter.cpp:172/311/341`，全仓库无同名符号冲突（当前不构成链接错误） |

## 5. 未覆盖/存疑项

1. **`Template/Shared.props` 里 `HintPath` 的相对基准未验证**：`Template/Shared.props:17-20` 的 `<HintPath></HintPath>` 被 `ReplaceHintPath`（`FSolutionGenerator.cpp:246-256`）写成 `..\..\Content\<Publish>\Interop.dll`。`Shared.props` 位于 `<Project>/Script/`（2 段深度与 csproj 不同），MSBuild 究竟以"定义该项的 props 文件目录"还是"引用它的工程目录"作为相对基准我未实测。若取前者，Interop 引用会指向 `<Project>/../Content/...`（差一级）→ 建议实机打开生成的工程确认。其余 5 个模板的占位符已逐个与替换串核对一致（`Script.sln:6/8/19/42`、`Game.csproj:2/4`、`Game.props:21`、`Shared.props:3/7/18`、`Interop.csproj:10/19`、`UE.csproj:4`）。
2. **非 ASCII（中文）注释的编码往返未实测**：写入侧固定 `ForceUTF8WithoutBOM`（`FUnrealCSharpFunctionLibrary.cpp:1177`），读取侧是 `FFileHelper::LoadFileToString` 默认（AutoDetect）。模板文件已确认全部是 **CRLF + 无 BOM**（本报告 §6 实测），因此模板路径上的编码是对称的；但 FDoxygenConverter 的输入来自 UHT 元数据（`Function->GetMetaData(TEXT("Comment"))`，`FClassGenerator.cpp:461`，非本次磁盘读取），其中文注释的编码取决于 UHT 解析结果，未实测。UHT 的 `Comment` 元数据是否保留 `/**`、`*/` 与每行前导 `*` 也未实测——若是保留的，F-GEN3-006（`/` 跨行吞噬）与"无 `@brief` 则输出为空"（F-GEN3-005）的触发面会比我的估计更大。
3. **XML 转义问题（F-GEN3-001）的编译器侧后果是条件性的**：本仓库 `Template/Shared.props:11,14` 的 `NoWarn` 只含 `0109;1701;1702;8500`，未含 `1570`；但 C# 只在启用 `/doc`（`GenerateDocumentationFile`）时才报 CS1570，而模板里没有该设置（`grep -n "GenerateDocumentationFile" Script/` 无命中）→ 默认构建**不会**因为非法 XML 注释报警，主要影响是 IDE 文档提示错乱；一旦用户开启文档生成（或接入 DocFX）就会变成真实告警/错误。
4. **`FCodeAnalysis` 的实际触发时机与原判断有出入**：任务书假设它是"文本/词法层面的分析"，实际它只是**外部 .NET 分析器进程的启动器**（真正的解析在 `Script/CodeAnalysis/CodeAnalysis.cs`，属 C# 侧，不在我的范围内）。我已把触发点查清（编辑器启动 `OnPostEngineInit`、主生成流程 `UnrealCSharpEditor.cpp:339`、控制台命令 `:95`、改蓝图 `FEditorListener.cpp:330`、蓝图工具栏 `UnrealCSharpBlueprintToolBar.cpp:80`），但 `Script/CodeAnalysis/CodeAnalysis.cs` 内部的正则/性能问题未审（属于 C# 模块报告范围）。
5. `FGeneratorCore::GetOverrideFunctions`、`FClassGenerator`/`FDelegateGenerator` 等其余生成器的实现细节属其他报告范围，我只读了与上述发现相关的片段（`FGeneratorCore.cpp:984-1079`、`FClassGenerator.cpp:400-529`、`:1095-1129`）。

## 6. 实测证据附录（本次分析中直接执行的检查）

| 检查 | 命令 | 结果 |
|---|---|---|
| XML 转义缺失 | `grep -rn "&lt;\|&amp;\|&gt;\|&quot;\|EscapeXml" Source/ScriptCodeGenerator` | 3 命中，**全部在 `FGameplayTagGenerator.cpp:135-137`**；`FDoxygenConverter.cpp` 内 0 命中 → 确认 Doxygen 路径无任何 XML 转义（支撑 F-GEN3-001），同时说明"要转义"在同一模块内已有先例 |
| 文档生成是否开启 | `grep -rn "GenerateDocumentationFile\|DocumentationFile" Script/`（排除 obj/bin） | 0 命中 → 默认不会触发 CS1570（见 §5 第 3 条） |
| 模板编码/换行 | pwsh 逐字节统计 `CRLF`/`bareLF`/BOM | 6 个模板全部 `CRLF` 且无 BOM，`bareLF=0` → **基于 `\r\n` 的替换当前成立**，无编码/BOM 问题（此维度已检查、未发现问题） |
| 模板占位符一致性 | `read` 6 个模板 + 对照 15 个 `Replace*` | 逐字一致（`Script.sln:6/8/19/42`、`Game.csproj:2/4`、`Game.props:21`、`Shared.props:3/7/18`、`Interop.csproj:10/19`、`UE.csproj:4`） |
| `Interop.csproj` 插件路径是否打补丁 | `grep -n "ReplacePluginBaseDir" FSolutionGenerator.cpp` | 命中 93（UE）与 166（定义），`78-86`（Interop）**未包含** → F-GEN3-015 |
| 死代码 | pwsh `Select-String -Pattern "\b<符号>"` 全 `Source/`（排除 ThirdParty） | 无死代码（见 §4 表） |
| 整个模块的日志能力 | pwsh `Select-String -Pattern "UE_LOG"` 于 `Source/ScriptCodeGenerator/**/*.cpp` | **0 命中**；`FAssetGenerator.cpp` 内 `UE_LOG`=0、`LoadObject`=3 → 模块内**完全没有任何日志**，这是 F-GEN3-003/007/011/012/014 中"静默失败"的共同放大因素 |
| 资产重命名/删除是否处理 | `grep -rn "GetOldFileName\|OnAssetRemoved\|OnAssetRenamed" Source/` | `FEditorListener.cpp:361-385` 有实现 → 据此**下调**了原本拟定的一条 P2 |
| 关键默认值 | `grep -n "EnableDeleteProxyDirectory\|IsGenerateFunctionComment\|IsGenerateAsset" Source/UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp` | `:26 bEnableDeleteProxyDirectory(false)`、`:36 bIsGenerateAsset(true)`、`:37 bIsGenerateFunctionComment(true)` → Doxygen 转换默认参与每次生成，故 F-GEN3-001/002/005/006 的影响面为"默认开启" |
| `Lex`/`Tokenize`/`Slice` 是否同名冲突 | `grep -rn "^ETokenKind Lex\|^TArray<FTokenData> Tokenize\|FStringView Slice" Source/` | 仅 `FDoxygenConverter.cpp:172/311/341`，无外部同名定义 |

