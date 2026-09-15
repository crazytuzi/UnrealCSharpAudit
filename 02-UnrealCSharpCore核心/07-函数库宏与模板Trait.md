# UnrealCSharpCore 函数库 / Core 宏 / 模板 Trait 分析报告

> 本报告各条**严重度见各 Finding 的「复核结论」字段**（逐条复核 27 条：确认 **23**、部分确认 **4**（F-FL-001/003/010/016）、证伪 **0**、无法验证 **0**；级别变动 **1** 条（F-FL-001 由 P1 降为 P2，与同源条目 `02-…/06` F-BR-004 对齐）；**撤销降级 1 条**（F-FL-019 —— `TIsTOptional_V` 经引擎源码证实存在）；**子命题证伪 1 处**（F-FL-010 的"大小写不敏感"）。最终级别分布：P1 = 1、P2 = 14、P3 = 12、**P0 = 0**）。





> 分析范围：`Source/UnrealCSharpCore/Public/Common/FUnrealCSharpFunctionLibrary.{h,cpp}`、`Source/UnrealCSharpCore/Public/CoreMacro/*.h`（**16 个**）、`Source/UnrealCSharpCore/Public/Template/*.inl`（19 个）
> 覆盖文件：1 + 1 + 16 + 19 = **37** 个（另读了 22 个调用方文件作为调用上下文证据，见 0.2）

> 状态：**已完成**。已回 `Engine\Source` 核实引擎行为，原"本工作区没有 Engine/ 源码"的说明**已作废**：`FString::Equals` 默认参数、`FJsonSerializer::Deserialize` 失败语义、`TIsTOptional_V` 的存在与特化、`FString::Equals` 之外的全部引擎论断均已改为源码实证（见各条 `复核证据`）。**仍未核实的两项**：`FPlatformProcess::CreateProc` 的管道方向语义（F-FL-009 未据其下结论）、`UStruct::StaticStruct` 对具体 USTRUCT 的可用性（F-FL-015 已按 trait 定义形态判定，未逐类型实例化）。

---

## 0. 覆盖范围与阅读清单

### 0.1 本次任务文件

| 文件 | 行数 | 是否读完 | 备注 |
|---|---|---|---|
| `Source/UnrealCSharpCore/Public/Common/FUnrealCSharpFunctionLibrary.h` | 290 | ✅ 读完 1-290 | 83 个 static 成员声明 |
| `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp` | **1615** | ✅ 读完（1-340 / 340-679 / 680-1019 / 1020-1359 / 1360-1615） | **任务书写的 1326 行与实际不符，实际 1615 行**（`read` 工具末行报告 `total 1615`） |
| `CoreMacro/AccessPrivateMacro.h` | 39 | ✅ | |
| `CoreMacro/BindingMacro.h` | 19 | ✅ | |
| `CoreMacro/BufferMacro.h` | 29 | ✅ | |
| `CoreMacro/ClassAttributeMacro.h` | 53 | ✅ | |
| `CoreMacro/ClassMacro.h` | 111 | ✅ | |
| `CoreMacro/CompilerMacro.h` | 21 | ✅ | |
| `CoreMacro/CoreCLRMacro.h` | 7 | ✅ | |
| `CoreMacro/FunctionAttributeMacro.h` | 31 | ✅ | |
| `CoreMacro/FunctionMacro.h` | 133 | ✅ | |
| `CoreMacro/GenericAttributeMacro.h` | 47 | ✅ | |
| `CoreMacro/Macro.h` | 133 | ✅ | |
| `CoreMacro/MetaDataAttributeMacro.h` | 315 | ✅ | |
| `CoreMacro/MonoMacro.h` | 9 | ✅ | |
| `CoreMacro/NamespaceMacro.h` | 17 | ✅ | |
| `CoreMacro/PropertyAttributeMacro.h` | 67 | ✅ | |
| `CoreMacro/PropertyMacro.h` | 7 | ✅ | |

> **§0.1 计数修正**：CoreMacro 记为"17 个"是**错的**。`glob "Source/UnrealCSharpCore/Public/CoreMacro/**/*"` 实测该目录只有 **16 个 `.h`**（上表所列正好 16 行，与目录实况一致）；因此 `CoreMacro` 之外的 `Source/UnrealCSharp/Public/Macro/*.h`（5 个，见 0.2）不属于本节范围。"覆盖文件：38"随之修正为 **37**。
>
> 另：本报告的统一行数口径为 **`read` 工具报告的总行数（含空行）**，等价于 `(Get-Content $f).Count`。`Measure-Object -Line` 会忽略空行（例如 `BindingMacro.h` 报 10 而实际 19），两种口径都对，本报告不采用后者。
| `Template/TAccessPrivate.inl` | 7 | ✅ | |
| `Template/TAccessPrivateStub.inl` | 17 | ✅ | |
| `Template/TFieldIteratorExt.inl` | 168 | ✅ | |
| `Template/TFunctionPointer.inl` | 17 | ✅ | |
| `Template/TGetArrayLength.inl` | 13 | ✅ | |
| `Template/TGetUtf8String.inl` | 54 | ✅ | |
| `Template/TIsNotUEnum.inl` | 7 | ✅ | |
| `Template/TIsReflectionClass.inl` | 11 | ✅ | |
| `Template/TIsScriptStruct.inl` | 7 | ✅ | |
| `Template/TIsTEnumAsByte.inl` | 17 | ✅ | |
| `Template/TIsTLazyObjectPtr.inl` | 13 | ✅ | |
| `Template/TIsTOptional.inl` | 11 | ✅ | |
| `Template/TIsTScriptInterface.inl` | 13 | ✅ | |
| `Template/TIsTSoftClassPtr.inl` | 13 | ✅ | |
| `Template/TIsTSoftObjectPtr.inl` | 13 | ✅ | |
| `Template/TIsTWeakObjectPtr.inl` | 13 | ✅ | |
| `Template/TIsUObject.inl` | 13 | ✅ | |
| `Template/TIsUStruct.inl` | 13 | ✅ | |
| `Template/TTemplateTypeTraits.inl` | 13 | ✅ | |

> 注：`Get-Content | Measure-Object -Line` 给出的行数比 `read` 少（例如 `BindingMacro.h` 报 10 行，实际 19 行），本报告一律以 `read` 工具输出的行号为准。**两种口径都正确，只是"是否含空行"不同**。

### 0.2 作为调用上下文证据额外读过的文件（非我的报告范围，仅引用）

`Source/UnrealCSharp/Public/Macro/{BindingMacro,NamespaceMacro,PropertyMacro,FunctionMacro,SignatureMacro}.h`、`Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h`、`Source/UnrealCSharp/Public/Binding/Property/TPropertyBuilder.inl`、`Source/UnrealCSharp/Public/Binding/Core/TPropertyClass.inl`、`Source/UnrealCSharp/Private/Reflection/Function/FFunctionDescriptor.cpp`、`Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp`、`Source/UnrealCSharp/Private/Domain/FDomain.cpp`、`Source/UnrealCSharpCore/Private/Domain/Script/FScriptDomainFactory.cpp`、`Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRFunctionLibrary.cpp`、`Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp`、`Source/UnrealCSharpCore/Private/Dynamic/{FDynamicClassGenerator,FDynamicGeneratorCore}.cpp`、`Source/UnrealCSharpCore/Private/Setting/UnrealCSharpSetting.cpp`、`Source/UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp`、`Source/UnrealCSharpCore/Public/Setting/UnrealCSharpSetting.h`、`Source/UnrealCSharpCore/Public/Common/FScriptDomainTypeScope.h`、`Source/UnrealCSharpCore/UnrealCSharpCore.build.cs`、`Source/UnrealCSharpEditor/Private/Listener/FEditorListener.cpp`、`Source/UnrealCSharpEditor/Private/NewClass/ClassCollector.cpp`、`Source/UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp`、`Source/ScriptCodeGenerator/Private/{FClassGenerator,FCodeAnalysis,FSolutionGenerator,FGameplayTagGenerator}.cpp`、`Source/Compiler/Private/FCSharpCompilerRunnable.cpp`、`Source/CrossVersion/Public/UEVersion.h`。

### 0.3 排除
`Source/ThirdParty/`、`Intermediate/`、`Binaries/`、`*.old`、`Script/**/obj|bin` 未分析（按 `_CONVENTIONS.md` 第 0 节）。

---

## 1. 模块职责与架构速览

`FUnrealCSharpFunctionLibrary` 是一个**无状态的静态工具类**（唯一状态是两个 `WITH_EDITOR` 静态成员：`ScriptDomainType`、`bScriptChanged`），承担 5 类职责：

1. **C# 命名映射**（全插件唯一的命名规则来源）：`GetFullClass` / `GetFullInterface` / `GetClassNameSpace` / `Encode`。C++ 侧生成器（`ScriptCodeGenerator`）用它产出 C# 标识符，运行期绑定注册（`FCSharpBind`）用同一个 `Encode` 注册描述符 —— 两端必须逐字符一致，因此这里的任何大小写/关键字行为都是**跨语言 ABI**。
2. **路径计算**：脚本根目录 `Script/`（`GetFullScriptDirectory` = `ProjectDir()/Script`）、程序集发布目录（`ProjectContentDir()/Script`）、插件目录、`UE/Game/Custom/Interop/CodeAnalysis/SourceGenerator/Weavers/Proxy` 子目录，以及**运行时**程序集搜索路径（`GetFullAssemblyPublishPath`）。
3. **程序集/依赖解析**：`UnrealCSharp_Modules.json`（271 KB，由 `UnrealCSharpCore.build.cs:120-160` 在 **UBT 编译期**写入 `Intermediate/`）被解析成 Engine/Project 两个模块名列表，用于判断某个 UField 属于 Game 还是 Engine（决定 C# 命名空间与生成目录）。
4. **文件 IO**：`SaveStringToFile`（写前比较，内容相同则不写、不置脏）、`LoadFileToArray` / `LoadFileToString`（JSON → TMap）、`SyncProcess`（同步子进程 + 管道捕获，**编辑器专用**）。
5. **反射小工具**：`IsGameField` / `IsNativeFunction` / `SetClassDefaultObject` / `IsSpecial*` / `GetMutableDefaultSafe`（用 `GExitPurge` 规避 Purge 期取 CDO）。

数据流（生成期）：
`FUnrealCSharpEditorModule::Generator` → `FScriptDomainTypeScope`（设置 `ScriptDomainType` 静态）→ `FGeneratorCore::BeginGenerator` → 遍历 UClass/UStruct/UFunction → `Encode/GetFullClass/GetClassNameSpace/GetGenerationPath`（读 `UnrealCSharp_Modules.json` 的模块表）→ 写 `Script/Proxy/**.cs` → `FCSharpCompilerRunnable`（**非游戏线程**）→ `GetDotNet()`+`SyncProcess` 调 `dotnet build` → `GetFullAssemblyPublishPath()` 得到程序集列表 → `FScriptDomainFactory::Create()` 建域 → `LoadAssembly`。

---

## 2. 关键调用链（均为实测，行号来自 `read`）

1. `UnrealCSharpEditor.cpp:316` → `FScriptDomainTypeScope(GetScriptDomainType(IniPlatformName))` → `FUnrealCSharpFunctionLibrary.cpp:1596 SetScriptDomainType` → 供 `FSolutionGenerator.cpp:212 GetScriptDomainType()` 决定是否加 `WITH_LEANCLR` 宏。
2. `FEditorListener.cpp:89` → `GetChangedDirectories()` → `FUnrealCSharpFunctionLibrary.cpp:1250` → `GetPluginScriptDirectory()/DEFAULT_UE_NAME` + `GetGameDirectory()` + `GetCustomProjectsDirectory()` → `DirectoryWatcher` 注册回调。
3. `FEditorListener.cpp:115/436` → `GetChangedDirectories()`（析构注销 / 变更过滤），每次调用**重新计算全部路径字符串**。
4. `FCSharpCompilerRunnable.cpp:291` → `SyncProcess(GetDotNet(), "build …")` → `FUnrealCSharpFunctionLibrary.cpp:1520` → `FPlatformProcess::CreateProc` + 10 ms 轮询 `ReadPipe`。
5. `FFunctionDescriptor.cpp:25` → `IsNativeFunction(GetOwnerClass(), GetFName())` → `FUnrealCSharpFunctionLibrary.cpp:1463`（沿继承链 + 所有接口逐个 `FindFunctionByName`）。
6. `FClassGenerator.cpp:482` → `IsNativeFunction(InClass, Function->GetFName())`，位于 `for (Index = 0; Index < FunctionParams.Num(); ++Index)` **循环体内**（每次 OutParam 都重算一次）。
7. `FDynamicClassGenerator.cpp:337/464` → `SetClassDefaultObject(InClass, CDO)` → `FUnrealCSharpFunctionLibrary.cpp:1510`。
8. `FMonoDomain.cpp:130/134` → `GetFullInteropPublishPath()` + `GetFullAssemblyPublishPath()` → 逐个程序集 `FPaths::FileExists` 过滤 → `LoadAssembly`；循环内 `FMonoDomain.cpp:406` 又调用一次 `GetFullInteropPublishPath()` 做等值比较。
9. `FDomain.cpp:26` → `FScriptDomainFactory::Create()`（**可能返回 nullptr**）→ `FDomain.cpp:31 ScriptDomain->Initialize()`。

---

## 3. 发现清单

### P0

> **【终裁：本节为空 —— 本报告不存在 P0 级发现】** 唯一的原 P0 候选 F-FL-001 最终裁定为 **P2**（理由见该条 `级别变动`）。**标题保留为章节占位，明确标注无 P0 条目**；为保持 Finding 编号锚点稳定，F-FL-001 仍在原位置不移动。

### [F-FL-001] 后端域工厂可返回 nullptr，`FDomain::Initialize` 直接解引用 —— 改一次脚本后端配置即崩溃

- **类别**: Bug（空指针解引用）
- **严重度**: **P2**
- **复核结论**: 部分确认（偏差：严重度由 P1 校正为 **P2**，与同源条目 `02-…/06` F-BR-004 对齐；"改 ini 后不重编译"这一主触发路径被 `AddExternalDependencies()` 显著削弱）
- **可达性**: 潜伏
- **复核证据**: `Source/UnrealCSharpCore/Private/Domain/Script/FScriptDomainFactory.cpp:24,27,33,39,44`（`:24` 运行期读 → 三处 `#if WITH_MONO/WITH_CORECLR/WITH_LEANCLR` → `:44` 裸 `return nullptr`）；`Source/UnrealCSharp/Private/Domain/FDomain.cpp:22,26,28,31`（`:31 ScriptDomain->Initialize()` 前无判空）；对照组 `Source/UnrealCSharpCore/Public/Domain/Script/FScriptDomainScope.h:15-25`（`!= nullptr` 双层判空）；`Source/UnrealCSharpCore/UnrealCSharpCore.build.cs:263-306`（UBT 读同名 ini 字符串 → 写三个宏）、`:99` 调用 + `:111-118` **`AddExternalDependencies()` 已把 `Config/DefaultUnrealCSharpSetting.ini` 登记为构建依赖**（改 ini 会触发 UBT 重建，故"不重编译"不再是默认路径）；`Source/UnrealCSharpCore/Public/Setting/UnrealCSharpSetting.h:75-80`（`Mono=0, CoreCLR=1, LeanCLR=2`）、`:150-163`（Win/Linux/Mac 无 `ValidEnumValues`，Android/iOS 有）
- **级别变动**: P1→P2（同源条目 `02-…/06` F-BR-004 定为 P2；本报告自身的同类"潜伏期空指针"F-FL-005/006/007 亦为 P2；`AddExternalDependencies()` 使"改配置即崩溃"需额外条件）
- **文件**: `Source/UnrealCSharpCore/Public/CoreMacro/MonoMacro.h:3`、`Source/UnrealCSharpCore/Public/CoreMacro/CoreCLRMacro.h:5`、`Source/UnrealCSharpCore/Public/CoreMacro/FunctionMacro.h:23,49`（根因：**后端是编译期宏**）；崩溃点 `Source/UnrealCSharp/Private/Domain/FDomain.cpp:26,31`、`Source/UnrealCSharpCore/Private/Domain/Script/FScriptDomainFactory.cpp:22-44`
- **函数**: `FScriptDomainFactory::Create()` → `FDomain::Initialize()`
- **置信度**: 高（代码事实确定；触发条件需要"运行期配置 ≠ 编译期配置"，见"问题"）

**现状（代码事实）**

```cpp
// Source/UnrealCSharpCore/Private/Domain/Script/FScriptDomainFactory.cpp:22-45
IScriptDomain* FScriptDomainFactory::Create()
{
	if (const auto ScriptDomainType = GetScriptDomainType();          // :24 运行期读 ini
		ScriptDomainType == EScriptDomainType::Mono)
	{
#if WITH_MONO                                                        // :27 编译期宏
		return new FMonoDomain();
#endif
	}
	else if (ScriptDomainType == EScriptDomainType::CoreCLR)
	{
#if WITH_CORECLR                                                      // :33
		return new FCoreCLRDomain();
#endif
	}
	else if (ScriptDomainType == EScriptDomainType::LeanCLR)
	{
#if WITH_LEANCLR                                                      // :39
		return new FLeanCLRDomain();
#endif
	}

	return nullptr;                                                       // :44  ← 三个分支全被编译掉时
}
```

```cpp
// Source/UnrealCSharp/Private/Domain/FDomain.cpp:20-34
void FDomain::Initialize()
{
	auto ScriptDomain = IScriptDomain::Get();          // :22
	if (ScriptDomain == nullptr)
	{
		ScriptDomain = FScriptDomainFactory::Create();  // :26 可能为 nullptr
		IScriptDomain::Set(ScriptDomain);               // :28
	}
	ScriptDomain->Initialize();                        // :31 ← 无检查，直接解引用
	InitializeSynchronizationContext();
}
```

对照组（同一发行版内另一处调用者**做了**检查）：
```cpp
// Source/UnrealCSharpCore/Public/Domain/Script/FScriptDomainScope.h:15-25
ScriptDomain = FScriptDomainFactory::Create();
if (ScriptDomain != nullptr) { IScriptDomain::Set(ScriptDomain); Domain = ScriptDomain; }
if (ScriptDomain != nullptr) { ... }
```

**调用上下文**
`FDomain::Initialize` 由 `FDomain`（运行时域管理器）在脚本域初始化阶段调用；`FScriptDomainFactory::GetScriptDomainType()`（`FScriptDomainFactory.cpp:12-20`）在**运行期**读 `UUnrealCSharpSetting::GetScriptDomainType(IniPlatformName)`；而 `WITH_MONO/WITH_CORECLR/WITH_LEANCLR` 由 **UBT 编译期**根据同一个 ini 值写入（`UnrealCSharpCore.build.cs:263-306`：读 `PlatformName + "ScriptDomainType"`，仅 `Mono`/`CoreCLR`/`LeanCLR` 三种字符串字面量命中，其它值 → 三个宏全为 0）。

**问题**
`WindowsScriptDomainType` / `LinuxScriptDomainType` / `MacScriptDomainType` 是 `UPROPERTY(Config, EditAnywhere)`（`UnrealCSharpSetting.h:150-157`），**没有** `ValidEnumValues` 限制（对比 Android/iOS 在 `:159-163` 有限制）。因此：
1. 用户在 Project Settings 里把 Windows 后端从 `CoreCLR` 改成 `LeanCLR`（或直接改 `Config/DefaultUnrealCSharpSetting.ini`），**不重新编译**；
2. 运行期 `Create()` 走到 `ScriptDomainType == LeanCLR` 分支，而 `WITH_LEANCLR == 0`（编译期值），`#if` 块被编译掉，函数落到 `return nullptr`；
3. `FDomain::Initialize:31` 对 nullptr 调虚函数 → 立即崩溃。
同样地，ini 里写一个拼错的值（如 `CoreCLR2`）会让三个宏全为 0，即使运行期枚举值合法也会返回 nullptr。这是"编译期后端 / 运行期后端双份真相"导致的必然裂缝。

**建议**
二选一，不要双份真相：
- 最小改动：在 `FDomain::Initialize` 的 `:26-31` 之间补检查并对缺失后端给出可读诊断（`FScriptDomainScope` 已是正确写法，直接抄）：
```cpp
ScriptDomain = FScriptDomainFactory::Create();
if (ScriptDomain == nullptr)
{
    UE_LOG(LogUnrealCSharp, Error,
        TEXT("ScriptDomain '%s' is not compiled in this binary. Rebuild after changing *ScriptDomainType."),
        *UEnum::GetValueAsString(FScriptDomainFactory::GetScriptDomainType()));
    return;
}
IScriptDomain::Set(ScriptDomain);
```
- 根因修复：把 `Create()` 的 `#if` 换成 `if constexpr`/运行时表，或在 `FScriptDomainFactory::GetScriptDomainType()` 里把运行期值夹到编译期可用的后端（`WITH_X==0` 时回退并 `UE_LOG(Warning)`）。同时给三个 `UPROPERTY` 加 `meta=(ValidEnumValues=...)` 按平台限制，或干脆把后端做成只读、只在 ini 手动改。

**验证方式**
1. `grep -n "WITH_LEANCLR" Source/UnrealCSharpCore/UnrealCSharpCore.build.cs`（:297-306）与 `FScriptDomainFactory.cpp:8-10,37-42` 对照；
2. 复现：编译后把 `Config/DefaultUnrealCSharpSetting.ini` 的 `WindowsScriptDomainType` 改成 `LeanCLR`（或任意非法值），重启编辑器 → 观察 `FDomain::Initialize` 空指针崩溃；
3. `grep -n "FScriptDomainFactory::Create()" Source/` 应得到 2 个调用点，确认只有 `FScriptDomainScope.h:15` 做了 null 检查。

---

### P1

### [F-FL-002] JSON 反序列化失败后未检查返回值，6 处空 `TSharedPtr` 解引用

- **类别**: Bug（空指针解引用）
- **严重度**: **P1**（若按"可达即 P0"口径应升为 P0：触发只需一个损坏/截断/非对象顶层的 JSON 文件）
- **复核结论**: 确认
- **可达性**: 活跃
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1189,1193,1195`（`TSharedPtr<FJsonObject> JsonObject;` → 丢弃 `Deserialize` 返回值 → `JsonObject->Values`）；同型 6 处解引用点 `:1195`、`:1235`、`:1321`、`:1333`、`:1366`、`:1378`，与报告一致。**引擎侧已核实失败语义**：`Runtime/Json/Public/Serialization/JsonSerializer.h:56-72` —— `:64-67` 在 `!State.Object.IsValid()` 时 `return false`，而 `OutObject = State.Object;` 在 `:69`，**失败时输出参数保持调用方默认构造的空 `TSharedPtr`** → `->Values` 确定是空指针解引用（原"依 UE 惯例推断"的前提已改为源码实证）
- **级别变动**: 无（P1 维持：触发需损坏/截断/顶层非对象的 JSON）
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1189-1195`、`:1229-1235`、`:1317-1321`、`:1333-1336`、`:1362-1366`、`:1378`
- **函数**: `FUnrealCSharpFunctionLibrary::LoadFileToArray(const FString&)`、`LoadFileToString(const FString&)`、`GetEngineModuleList()`、`GetProjectModuleList()`
- **置信度**: 高（代码事实确定：`Deserialize` 返回值被丢弃、紧接着解引用；失败语义依 UE 惯例"不再赋值，保持调用方默认构造的 null"）

**现状（代码事实）**

```cpp
// FUnrealCSharpFunctionLibrary.cpp:1187-1195  （LoadFileToArray）
if (FString ResultString; FFileHelper::LoadFileToString(ResultString, *InFileName))
{
	TSharedPtr<FJsonObject> JsonObject;                                   // :1189 默认 null
	const auto& JsonReader = TJsonReaderFactory<TCHAR>::Create(ResultString);
	FJsonSerializer::Deserialize(JsonReader, JsonObject);                 // :1193 返回值被丢弃
	for (const auto& [Key, Value] : JsonObject->Values)                   // :1195 ← 失败即空指针解引用
```
```cpp
// :1229-1235  （LoadFileToString）          :1362-1366  （GetProjectModuleList）
FJsonSerializer::Deserialize(JsonReader, JsonObject);                     // :1233
for (const auto& [Key, Value] : JsonObject->Values)                       // :1235 ← 同上
...
FJsonSerializer::Deserialize(JsonReader, JsonObj);                        // :1364
if (const TSharedPtr<FJsonObject>* OutObject; JsonObj->TryGetObjectField(TEXT("ProjectModules"), OutObject))  // :1366 ← 同上（:1321/:1336/:1378 同型）
```

**调用上下文**
- `LoadFileToArray`：`FDynamicGeneratorCore.cpp:21`（`BeginCodeAnalysisGenerator` 读 `Intermediate/CodeAnalysis/Dynamic.json`）、`FGeneratorCore.cpp:1016`（编辑器生成器启动时读 OverrideFunctions）。
- `LoadFileToString`：`FDynamicGenerator.cpp:99`、`UnrealCSharpBlueprintToolBar.cpp:207`（工具栏）。
- `GetEngineModuleList/GetProjectModuleList`：`UnrealCSharpSetting.cpp:327,329`，以及本文件内 `:163,299,995`（生成器的**每个** UField 都要走）。
调用时机：编辑器模块初始化 / 每次开始生成代码（游戏线程）。

**问题**
`FJsonSerializer::Deserialize(Reader, OutObject)` 失败时返回 `false` 且**不写入** `OutObject`，调用方传入的是默认构造的空 `TSharedPtr`，于是 `JsonObject->Values` 就是解引用空指针 → 崩溃。可触发的输入：文件被截断（生成器/进程被杀导致写一半）、手工编辑出错、顶层不是对象（例如 `[]`、`null`、纯文本），以及任何非法 JSON。这些都是"用户可写"的中间文件（`Intermediate/` 与 `Script/` 下），并非不可达的边角。同一函数里 `GetEngineModuleList` 还有第 2 处（`:1333`）同型缺陷。

**建议**
统一在反序列化失败时返回空容器并报日志（保持函数"返回类型即错误信号"的既有契约）：
```cpp
if (!FJsonSerializer::Deserialize(JsonReader, JsonObject) || !JsonObject.IsValid())
{
    UE_LOG(LogUnrealCSharp, Warning, TEXT("Failed to parse JSON: %s"), *InFileName);
    return Result;      // 空 TMap，调用方已能处理空表
}
```
`GetEngineModuleList/GetProjectModuleList` 同样在 `:1319/:1364` 后加 `if (!JsonObj.IsValid()) { return List; }`。

**验证方式**
1. `grep -n "FJsonSerializer::Deserialize" Source/` 共 6 处命中，逐一确认返回值是否被使用；
2. 用例：把 `Intermediate/CodeAnalysis/Dynamic.json` 内容改成 `[`（或清空该文件），启动编辑器 / 触发"生成代码" → 崩溃点应落在 `:1195`；
3. 加 `check(JsonObject.IsValid())` 到 `:1195` 可让 Debug 构建直接断言定位。

---

### [F-FL-003] 模块名列表缓存用 `IsEmpty()` 当"已加载"判据：列表为空时每次调用都重读并重解析 271 KB JSON，且永不失效

- **类别**: 性能 / 功能（缓存正确性）
- **严重度**: **P2**
- **复核结论**: 部分确认（偏差：可达性须分半判定，见下）
- **可达性**: 潜伏（"`IsEmpty()` 当已加载判据 → 每次重解析"半：需 `UnrealCSharp_Modules.json` 缺失或无 `ProjectModules`/`ProjectPlugins`，本机该文件**存在且非空**故常态不触发；"缓存永不失效"半：会话内改 `CustomProjects` 后重新生成时**活跃**）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1305,1307,1309,1311,1350,1352,1354,1356,1392`（两个 `static TArray` + `IsEmpty()` 作判据 + 库内 `static auto FilePath`）；**JSON 体量已实测**：`Intermediate/UnrealCSharp_Modules.json` = **271,805 字节**（`LastWriteTime 2026-09-10 13:32`），与报告"271 KB / 271,805 字节"完全一致；写出方 `Source/UnrealCSharpCore/UnrealCSharpCore.build.cs:129-131,160`
- **级别变动**: 无（P2 维持）
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1305-1348`、`:1350-1393`
- **函数**: `FUnrealCSharpFunctionLibrary::GetEngineModuleList()`、`GetProjectModuleList()`
- **置信度**: 高（代码事实确定；"触发频率"取决于模块表是否为空，见"问题"）

**现状（代码事实）**

```cpp
// :1350-1393  （GetProjectModuleList；:1305-1348 GetEngineModuleList 同构）
const TArray<FString>& FUnrealCSharpFunctionLibrary::GetProjectModuleList()
{
	static TArray<FString> ProjectModuleList;                              // :1352
	if (ProjectModuleList.IsEmpty())                                       // :1354 ← 用"空"表示"未加载"
	{
		static auto FilePath = FPaths::Combine(FPaths::ProjectIntermediateDir(), TEXT("UnrealCSharp_Modules.json"));  // :1356
		if (FString JsonStr; FFileHelper::LoadFileToString(JsonStr, *FilePath))  // :1358
		{
			... FJsonSerializer::Deserialize(...)                           // :1364
			if (... TryGetObjectField(TEXT("ProjectModules"), OutObject))   // :1366
			{ for (...) ProjectModuleList.AddUnique(...); }                 // :1371
			if (... TryGetObjectField(TEXT("ProjectPlugins"), OutObject))    // :1378
			{ for (...) ProjectModuleList.AddUnique(...); }                 // :1383
		}
	}
	return ProjectModuleList;                                              // :1392
}
```

**调用上下文**
`GetProjectModuleList()` 被 `GetModuleName(const FString&)`（`:163`）、`GetOuterRelativePath(const FString&)`（`:299`）、`GetGenerationPath(const FString&)`（`:995`）调用，而这三个又经 `GetModuleName(const UField*)`（`:60`）、`GetModuleName(const UPackage*)`（`:134`）、`IsGameField`（`:956`）、`GetFileName`（`:703-715`）在**每个 UField / 每个资产**上被调用 —— 生成器一次全量运行是 10^4~10^5 次量级。`GetProjectModuleList` 用 `TArray::Contains`（线性 + `FString` 比较，`:164/:299/:996`）做成员判断。

**问题**
两个缺陷叠加：
1. **空即重读**：只要 `UnrealCSharp_Modules.json` 不存在或其中没有 `ProjectModules`/`ProjectPlugins` 条目，`ProjectModuleList` 恒为空 → `IsEmpty()` 恒真 → **每次调用**都执行 `FFileHelper::LoadFileToString` + `FJsonSerializer::Deserialize`。实测该文件为 **271,805 字节**（`Get-ChildItem Intermediate` 输出），单次解析成本不低；生成期上千万次调用即等于把编辑器卡死，而日志里不会有任何提示。该文件由 **UBT 编译期**写入（`UnrealCSharpCore.build.cs:120-160`，`Path.Combine(Intermediate,"UnrealCSharp_Modules.json")`），所以"只跑了一次生成、没走完整 C++ 编译"或 `Intermediate/` 被清理后又只做增量生成时，就会出现这一状态。
2. **永不失效**：一旦填充成功，缓存**没有任何失效入口**。会话中新增 Custom Project / 新模块（`UnrealCSharpSetting->GetCustomProjects()` 的变化）不会被感知，`GetGenerationPath`（`:995-1010`）会把新模块误判为 UE 模块而生成到 `UE/Proxy` → C# 侧命名空间与程序集归属错误。

附带：`static TArray` 的惰性填充没有同步（`GetModuleName` 若被非游戏线程调用即数据竞争），不过本次未找到跨线程调用点（见第 5 节）。

**建议**
```cpp
// 方案 A（推荐）：加显式"已尝试"标志 + 显式失效接口
static bool bModuleListLoaded = false;
static TArray<FString> ProjectModuleList;
if (!bModuleListLoaded)
{
    bModuleListLoaded = true;                 // 无论成功失败都只做一次 IO
    if (!LoadModuleListFromDisk(ProjectModuleList))   // 失败只 Warn 一次
        UE_LOG(LogUnrealCSharp, Warning, TEXT("UnrealCSharp_Modules.json not found: %s"), *FilePath);
}
// 提供 static void InvalidateModuleListCache(); 在 GenerateScriptCode / 设置变更时调用
```
另外把模块判断从 `TArray::Contains` 换成 `TSet<FString>`，并把 `IsEmpty()` 判据改为 `bLoaded`；线程安全用 `FCriticalSection` 或在首次访问时 `check(IsInGameThread())`。

**验证方式**
1. `grep -n "IsEmpty()" Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp` → 确认 `:1309,:1354` 是唯一"加载判据"；
2. 计时用例：临时改名为 `UnrealCSharp_Modules.json.bak`，跑一次 `UnrealCSharp.Generator`，用 `stat` 统计该文件被 `LoadFileToString` 的次数（或在 `:1358` 打断点计数）；
3. 功能用例：会话中在设置里新增一个 Custom Project，再次生成，观察其 `.cs` 落在 `Proxy/` 还是 `UE/Proxy/`。

---

### [F-FL-004] `GetDotNet()`：非 Windows 平台一律返回 macOS 安装路径，且不校验可执行文件是否存在

- **类别**: 平台兼容 / 错误处理
- **严重度**: **P2**
- **复核结论**: 确认（代码事实与调用链全部成立；"Linux 上该默认路径通常不存在"属发行版外部知识，不在引擎源码裁决范围内，维持为**未验证假设**）
- **可达性**: 潜伏（本工程五平台全 LeanCLR；`#else` 分支只在非 Windows 编辑器构建上生效，本机为 Win64）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:52,53,55`（`#if PLATFORM_WINDOWS` → `:53` Windows 路径，`:55` `#else` 单一路径）；调用点 `Compiler/Private/FCSharpCompilerRunnable.cpp:291,413`、`ScriptCodeGenerator/Private/FCodeAnalysis.cpp:34,49,86`、`Source/UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp:152`（均为 `SyncProcess` 调用，无一检查路径存在性）；`Source/UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp:142,144`（`GetDotNetPathArray()` 整体被 `#if PLATFORM_WINDOWS` 包住，已 read 确认）
- **级别变动**: 无（P2 维持）
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:41-58`
- **函数**: `FUnrealCSharpFunctionLibrary::GetDotNet()`
- **置信度**: 中（代码事实确定；"Linux 上该路径通常不存在"属**发行版/安装器的外部知识**，不在 UE 引擎源码可裁决的范围内 —— 本报告剩余的两处"未验证假设"之一）

**现状（代码事实）**

```cpp
// :42-57
FString FUnrealCSharpFunctionLibrary::GetDotNet()
{
	if (const auto UnrealCSharpEditorSetting = GetMutableDefaultSafe<UUnrealCSharpEditorSetting>())
	{
		if (const auto& DotNetPath = UnrealCSharpEditorSetting->GetDotNetPath(); !DotNetPath.IsEmpty())
		{
			return DotNetPath;                                     // :48 用户显式配置优先
		}
	}

#if PLATFORM_WINDOWS
	return TEXT("C:/Program Files/dotnet/dotnet.exe");             // :53
#else
	return TEXT("/usr/local/share/dotnet/dotnet");                 // :55 ← macOS 官方安装路径；Linux/Android/iOS 共用
#endif
}
```

**调用上下文**
`Compiler/Private/FCSharpCompilerRunnable.cpp:279,384`（`static auto CompileTool = GetDotNet();` 然后 `SyncProcess(CompileTool, …)`）、`ScriptCodeGenerator/Private/FCodeAnalysis.cpp:41,49`、`UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp:152`。这些调用点**都不检查该路径是否存在**，直接交给 `SyncProcess` → `FPlatformProcess::CreateProc`（`:1543`）。失败时的行为链：`CreateProc` 返回无效句柄 → `while (ProcessHandle.IsValid() && …)`（`:1569`）不进入 → `GetProcReturnCode` 失败 → `ReturnCode = -1`（`:1580-1583`）→ `InOnComplete(-1, "")` → 调用方只看到"编译失败 + 空日志"（`FCSharpCompilerRunnable.cpp:298-301`）。

**问题**
平台分支只有 `PLATFORM_WINDOWS` 一档，`#else` 把 macOS/Linux（以及 Android/iOS 若启用编辑器工具）合并成同一个硬编码路径。`/usr/local/share/dotnet/dotnet` 是 macOS `dotnet-install.sh` 的默认位置；Linux 发行版包（apt/dnf/snap）通常装在别处。并且本文件所在的编辑器辅助函数 `UnrealCSharpEditorSetting::GetDotNetPathArray()`（`UnrealCSharpEditorSetting.cpp:142-155`）本身也是 `#if PLATFORM_WINDOWS`，即**非 Windows 下连"自动探测下拉框"都不提供**，用户只能手工填绝对路径。结论：Linux 上开箱即用的编译链路会静默失败，错误信息是空字符串，排查成本很高。

**建议**
```cpp
#if PLATFORM_WINDOWS
	return TEXT("C:/Program Files/dotnet/dotnet.exe");
#elif PLATFORM_MAC
	return TEXT("/usr/local/share/dotnet/dotnet");
#elif PLATFORM_LINUX
	for (const TCHAR* Candidate : { TEXT("/usr/share/dotnet/dotnet"),
	                                TEXT("/usr/lib/dotnet/dotnet"),
	                                TEXT("/usr/local/share/dotnet/dotnet") })
		if (FPaths::FileExists(Candidate)) return Candidate;
	return TEXT("dotnet");        // 交给 PATH
#endif
```
并在 `GetDotNet()` 返回处（或 `SyncProcess` 入口）检查：非空但不存在时 `UE_LOG(Error, "dotnet not found at %s; set it in UnrealCSharp Editor Settings")`，让失败可见。

**验证方式**
1. `grep -n "PLATFORM_WINDOWS\|/usr/local/share/dotnet" Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp Source/UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp`；
2. Linux 用例：清空 `DotNetPath` 设置后触发编译，观察 `SyncProcess` 回调的 ReturnCode 与输出是否为空；
3. 在 `:53-56` 前后加 `ensureMsgf(FPaths::FileExists(...))`。

---

### P2

> **排序说明**：本节含 P2 与 P3 两种级别的条目 —— F-FL-010、F-FL-012、**F-FL-018** 三条的 `严重度` 字段为 P3，但它们的物理位置仍在 P2 节。为保持编号锚点稳定，**不移动条目位置**，仅在此声明：本节中 **F-FL-010 / F-FL-012 / F-FL-018 实为 P3**，其余 11 条为 P2。

### [F-FL-005] `ParseIntoArray` 结果索引未校验（3 处）—— 空/单段路径会越界读

- **类别**: 未定义行为
- **严重度**: **P2**（隐患：现有调用点无法触发，但函数是 `UNREALCSHARPCORE_API` 公开静态接口，任何新调用点都会踩）
- **复核结论**: 确认
- **可达性**: 潜伏（三处均需"输入为空/仅分隔符"；`InName.IsEmpty()`（`:154`）、`:290`、`:986` 已挡空串，仅剩"只有 `/`"这一现有调用点产生不了的输入）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:161,164,166,170`（`ParseIntoArray` → `OutArray[0]`/`OutArray[1]`；仅 `:170` 有 `IsValidIndex(1)`，`OutArray[0]` 全程无守卫）；`:297,300,304`（同型）；`:993,995,996,997,999`（`Splits[0]`/`Splits[1]`，`:986` 仅挡空串与不以 `/` 开头）；`Source/UnrealCSharpCore/Public/CoreMacro/NamespaceMacro.h:7`（`"Script"` 字面量确实等于 `NAMESPACE_SCRIPT`）
- **级别变动**: 无（P2 维持）
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:159-176`（`:164,:166`）、`:295-314`（`:300`）、`:991-1011`（`:996,:999`）
- **函数**: `GetModuleName(const FString&)`、`GetOuterRelativePath(const FString&)`、`GetGenerationPath(const FString&)`
- **置信度**: 高（代码事实确定；触发前提"输入无 `/` 或仅有分隔符"已核对现有调用点，见"问题"）

**现状（代码事实）**

```cpp
// :159-166
TArray<FString> OutArray;
InName.ParseIntoArray(OutArray, TEXT("/"));                       // 默认剔除空段
if (const auto& ProjectModuleList = GetProjectModuleList();
	ProjectModuleList.Contains(OutArray[0]) ||                    // :164  OutArray 可能为空
	OutArray[0] == TEXT("Game") ||
	(OutArray[0] == TEXT("Script") && ProjectModuleList.Contains(OutArray[1])))   // :166 越界 + 无长度校验
```
```cpp
// :295-300    InRelativePath.ParseIntoArray(OutArray, TEXT("/"));
//             ProjectModuleList.Contains(OutArray[0]) || OutArray[0] == TEXT("Game")     // :300
// :991-999    InScriptPath.ParseIntoArray(Splits, TEXT("/"));
//             ProjectModuleList.Contains(Splits[0]) || Splits[0] == TEXT("Game") ||
//             (Splits[0] == NAMESPACE_SCRIPT && ProjectModuleList.Contains(Splits[1]))   // :996,:999
```

**调用上下文**
三个函数的入参都来自 UE 内部路径字符串：`GetModuleName(UPackage*)`（`:134-148`）传包名（`/Script/Engine`）；`GetOuterName(UClass*)`（`:347-370`）对 native 类拼 `"Outer/Class"`（`:358-362`）；`GetFileName`（`:703-715`）传 `InAssetData.PackagePath.ToString()`。我的核对结论：**这些调用点产生的段数都 ≥2**，所以 `OutArray[1]`/`Splits[1]` 在当前代码库内不可达；`OutArray[0]`/`Splits[0]` 需要在输入为 `""`、`"/"`、`"///"` 时才会命中空数组（`InName.IsEmpty()` 已在 `:154` 挡住空串，`GetGenerationPath` 在 `:986` 挡空串，但都**没有**挡"只有分隔符"）。

**问题**
`TArray::operator[]` 只在 `DO_CHECK`（Debug/Development）下做 `RangeCheck`，Shipping 下是裸指针运算。三类风险：(a) 输入只有分隔符时 `OutArray[0]` 读空数组首元素；(b) `OutArray[0] == "Script"` 且段数=1 时 `OutArray[1]` 越界（`"Script"` 恰好是 `NAMESPACE_SCRIPT` 的字面量，见 `NamespaceMacro.h:7`，所以这不是理论值）；(c) 顺序耦合：`Contains(OutArray[0]) || OutArray[0]=="Game"` 依赖短路求值才不越界，把条件顺序调换就会立刻越界。

**建议**
统一加长度守卫，并把"模块名解析"抽成一个只做一次分割的小工具：
```cpp
if (OutArray.Num() == 0) { return FString(); }
const bool bIsGame = ProjectModuleList.Contains(OutArray[0]) || OutArray[0] == TEXT("Game") ||
                     (OutArray[0] == NAMESPACE_SCRIPT && OutArray.IsValidIndex(1) && ProjectModuleList.Contains(OutArray[1]));
```
`GetGenerationPath` 同理（`:995-999`）：先 `if (!Splits.IsValidIndex(0)) { return UEProxyPath; }`。

**验证方式**
单测/控制台：`FUnrealCSharpFunctionLibrary::GetModuleName(TEXT("/"))`、`GetOuterRelativePath(TEXT("/"))`、`GetGenerationPath(TEXT("/"))` —— Development 构建下应看到 `Array index out of bounds` 断言；或 `grep -n "ParseIntoArray" Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp` 定位全部 5 处分割点。

---

### [F-FL-006] `IsNativeFunction` 未检查 `Function == nullptr`；且被放在 per-parameter 循环内重复调用

- **类别**: Bug（空指针）+ 性能
- **严重度**: **P2**
- **复核结论**: 确认
- **可达性**: 潜伏（空指针半：需调用方传入不在继承链/接口集合中的 `FName`，现有两个调用点都不满足；性能半：`FClassGenerator.cpp:482` 的逐参数循环**活跃**）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1470`（`const UFunction* Function{};`）、`:1486-1491`（仅命中时赋值）、`:1496`（`Function->IsNative()` 无判空）；`grep "IsNativeFunction("` 全 `Source/` **命中 3 处**：定义 `FUnrealCSharpFunctionLibrary.cpp:1463` + 调用 `Source/ScriptCodeGenerator/Private/FClassGenerator.cpp:482` + 调用 `Source/UnrealCSharp/Private/Reflection/Function/FFunctionDescriptor.cpp:26`（报告写 `:25`，实际 `:26`，±1）；`FClassGenerator.cpp:477`（`for` 循环头）与 `:482`（循环体调用）之间无提前求值，循环不变式判断成立
- **级别变动**: 无（P2 维持）
- **函数**: `FUnrealCSharpFunctionLibrary::IsNativeFunction(const UClass*, const FName&)`
- **置信度**: 高（空指针路径代码事实确定、当前不可达；循环内重复调用确定）

**现状（代码事实）**

```cpp
// :1463-1497
bool FUnrealCSharpFunctionLibrary::IsNativeFunction(const UClass* InClass, const FName& InFunctionName)
{
	if (InClass == nullptr) { return false; }                 // :1465-1468
	const UFunction* Function{};                              // :1470  ← null 初始化
	auto OwnerClass = InClass;
	auto Class = InClass;
	while (Class != nullptr)
	{
		for (const auto& Interface : Class->Interfaces)        // :1478 每个接口一次 FindFunctionByName（TMap 查找）
		{
			if (Interface.Class->FindFunctionByName(InFunctionName, EIncludeSuperFlag::Type::ExcludeSuper))
			{ return Interface.Class->IsNative(); }            // :1482
		}
		if (const auto Result = Class->FindFunctionByName(InFunctionName, EIncludeSuperFlag::Type::ExcludeSuper))
		{ Function = Result; OwnerClass = Class; }             // :1486-1491 故意不 break：继续向上找最基类的声明
		Class = Class->GetSuperClass();
	}
	return Function->IsNative() && OwnerClass->IsNative();     // :1496 ← Function 可能仍为 nullptr
}
```
循环调用点：
```cpp
// Source/ScriptCodeGenerator/Private/FClassGenerator.cpp:477-483
for (auto Index = 0; Index < FunctionParams.Num(); ++Index)
{
	if (FunctionParams[Index]->HasAnyPropertyFlags(CPF_OutParm) && !FunctionParams[Index]->HasAnyPropertyFlags(CPF_ConstParm))
	{
		if (FUnrealCSharpFunctionLibrary::IsNativeFunction(InClass, Function->GetFName()) ||   // :482 循环不变式
```

**调用上下文**
`FFunctionDescriptor.cpp:25-27`（运行期绑定初始化，`Function` 已被 `IsValid()` 检查过且必然是 `GetOwnerClass()` 的成员 → 安全）；`FClassGenerator.cpp:482`（编辑器生成，`Function` 来自正在遍历的函数列表 → 安全）。

**问题**
1. **空指针**：`Function` 只在"沿继承链（含每个接口）找到同名函数"时才被赋值；若调用方给的 `InFunctionName` 不在该类的继承链/接口集合里，`:1496` 就是空指针解引用。当前两个调用点都传"自己刚拿到的函数名"，因此**现在不可达**；但该函数是导出 API，`FName` 又极易被上层用 `ScriptName`/重命名元数据算错，属于"一旦踩到就崩溃"的接口缺口。
2. **性能**：`IsNativeFunction` 是**循环不变式**却被放在逐参数循环内（`FClassGenerator.cpp:477-482`）。单次调用要遍历 `GetSuperClass()` 全链，且每层对每个 `Interfaces` 元素做一次 `FindFunctionByName`（TMap 查找）；UE 里 `AActor` 之类深层类 + 多接口的类，一次生成函数会重复几十遍。

**建议**
```cpp
if (Function == nullptr)
{
    UE_LOG(LogUnrealCSharp, Warning, TEXT("IsNativeFunction: '%s' not found in hierarchy of %s"),
           *InFunctionName.ToString(), *InClass->GetName());
    return false;      // 或 checkNoEntry()，取决于是否希望暴露调用方错误
}
```
调用侧把不变量提出来：`const bool bIsNativeFunction = IsNativeFunction(InClass, Function->GetFName());` 放在 `for` 之前。

**验证方式**
1. `grep -n "IsNativeFunction(" Source/` → 确认只有 `FFunctionDescriptor.cpp:25` 与 `FClassGenerator.cpp:482`；
2. `FClassGenerator.cpp:482` 提循环外后对比生成耗时（同一工程 `UnrealCSharp.Generator` 两次运行）；
3. 临时构造 `IsNativeFunction(AActor::StaticClass(), TEXT("NoSuchFn"))` 单测，Development 下应看到我在 `:1496` 加的告警而非崩溃。

---

### [F-FL-007] `GetPluginBaseDir()` 未检查 `FindPlugin` 返回值

- **类别**: Bug（空指针）
- **严重度**: **P2**
- **复核结论**: 确认
- **可达性**: 潜伏（仅在插件未注册/被禁用而模块仍加载时触发）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:905`（`IPluginManager::Get().FindPlugin(PLUGIN_NAME)->GetBaseDir()` 无判空）；同文件对照 `:965` 的 `if (const auto Plugin = IPluginManager::Get().FindPlugin(ModuleName))` **确实有判空**（已 read 956-971 确认）；`Source/UnrealCSharpCore/Public/CoreMacro/Macro.h:3`（`PLUGIN_NAME` = `FString(TEXT("UnrealCSharp"))`）
- **级别变动**: 无（P2 维持）
- **函数**: `FUnrealCSharpFunctionLibrary::GetPluginBaseDir()`
- **置信度**: 中（代码事实确定；"何时 FindPlugin 返回 nullptr"依赖插件管理器状态，未构造出实际用例）

**现状（代码事实）**

```cpp
// :903-906
FString FUnrealCSharpFunctionLibrary::GetPluginBaseDir()
{
	return IPluginManager::Get().FindPlugin(PLUGIN_NAME)->GetBaseDir();   // :905 无 null 检查
}
```
`PLUGIN_NAME` = `FString(TEXT("UnrealCSharp"))`（`CoreMacro/Macro.h:3`）。

**调用上下文**
`GetPluginDirectory()`（`:908-911`）→ `GetPluginScriptDirectory()`（`:913-916`）→ `GetChangedDirectories()`（`:1250-1257`，被 `FEditorListener.cpp:89,115,436` 在模块启动/析构/文件变更回调里调用）；以及 `GetPluginTemplateDirectory()`（`:919-922`）→ 模板/Override/Dynamic 文件名（`:929-953`）。

**问题**
`IPluginManager::FindPlugin` 在插件未注册时返回 nullptr（例如插件被禁用但模块仍被加载、`.uplugin` 被改名/移动、纯 commandlet 环境下插件尚未挂载）。这里直接解引用 → 崩溃，且崩溃点离原因很远。同文件其它函数（`GetUEName`/`GetGameName`/`GetPublishDirectory`）都写了 null 兜底，此处风格不一致。

**建议**
```cpp
if (const auto Plugin = IPluginManager::Get().FindPlugin(PLUGIN_NAME))
{
	return Plugin->GetBaseDir();
}
UE_LOG(LogUnrealCSharp, Error, TEXT("Plugin '%s' not found"), *PLUGIN_NAME);
return FString();
```
并在 `GetPluginDirectory()` 的返回值上加 `ensure`/空串检查，避免把 `""` 当路径继续拼接。

**验证方式**
`grep -n "FindPlugin(" Source/` → 对比 `:905`（无检查）与 `:965` 的 `if (const auto Plugin = …)`（有检查）；用例：把 `.uplugin` 改为 `UnrealCSharp.disabled.uplugin` 后启动，观察是否崩溃。

---

### [F-FL-008] 发布程序集路径不校验存在性，运行时消费者静默跳过 —— iOS/Android 或无编译产物时无声失败

- **类别**: 平台兼容 / 错误处理
- **严重度**: **P2**
- **复核结论**: 确认（仅"调用点计数"一处需修正：不是 7 个而是 **11 个外部调用点 + 1 个库内调用**）
- **可达性**: 活跃（`FLeanCLRDomain.cpp:77` 在五平台 LeanCLR 下每次域初始化都会走到）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1064-1072`（`TArrayBuilder` 无条件拼接，无 `FPaths::FileExists`）、`:1029-1037`（`#if !WITH_EDITOR && WITH_CORECLR` 分支）；`Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:399,401-404,406`（`continue` 静默跳过 + 循环内重算，已 read 393-422 确认）。**调用点重跑**：`grep "GetFullAssemblyPublishPath|GetFullInteropPublishPath"`（`include=*.cpp`）共 **14 行命中** = 2 处定义（`:1029`、`:1064`）+ 1 处库内调用（`:1067`）+ **11 处外部调用**：`FCSharpCompilerRunnable.cpp:274`、`FEditorListener.cpp:578,582`、`GeneratorScriptCodeCommandlet.cpp:21`、`FLeanCLRDomain.cpp:77`、`FCoreCLRDomain.cpp:110,125,219`、`FMonoDomain.cpp:130,134,406`
- **级别变动**: 无（P2 维持）
- **函数**: `GetFullInteropPublishPath()`、`GetFullUEPublishPath()`、`GetFullGamePublishPath()`、`GetFullCustomProjectsPublishPath()`、`GetFullAssemblyPublishPath()`
- **置信度**: 高（本文件代码事实确定；静默跳过行为在调用方已核实）

**现状（代码事实）**

```cpp
// :1064-1072
TArray<FString> FUnrealCSharpFunctionLibrary::GetFullAssemblyPublishPath()
{
	return TArrayBuilder<FString>().
	       Add(GetFullInteropPublishPath()).
	       Add(GetFullUEPublishPath()).
	       Add(GetFullGamePublishPath()).
	       Append(GetFullCustomProjectsPublishPath()).
	       Build();                       // 全部无条件拼接，不检查文件是否存在
}
```
```cpp
// :1029-1037
FString FUnrealCSharpFunctionLibrary::GetFullInteropPublishPath()
{
#if !WITH_EDITOR && WITH_CORECLR
	return FPaths::ConvertRelativePathToFull(FCoreCLRFunctionLibrary::GetCoreCLRDirectory() / (INTEROP_NAME + DLL_SUFFIX));
#else
	return FPaths::ConvertRelativePathToFull(GetFullPublishDirectory() / (INTEROP_NAME + DLL_SUFFIX));
#endif
}
```

**调用上下文**
`FMonoDomain.cpp:130,134`、`FCoreCLRDomain.cpp:110,125`、`FLeanCLRDomain.cpp:77`（域初始化）、`GeneratorScriptCodeCommandlet.cpp:21`、`FEditorListener.cpp:578,582`。消费侧只做存在性过滤后 `continue`：
```cpp
// Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:399-409
for (const auto& AssemblyPath : InAssemblies)
{
	if (!FPaths::FileExists(AssemblyPath)) { continue; }          // :401-404 静默跳过
	if (AssemblyPath == FUnrealCSharpFunctionLibrary::GetFullInteropPublishPath()) { continue; }   // :406 循环内再次全量计算
```

**问题**
(a) 该函数族把"路径存在"当既定事实返回，缺失时不产生任何诊断；调用方 `continue` 掉之后，症状是"C# 逻辑全不生效、无报错"，排查成本高。iOS（AOT 静态链接）/Android 上发布产物形态与桌面 `.dll` 目录不同，最容易命中。(b) `FMonoDomain.cpp:406`/`FCoreCLRDomain.cpp:219` 在**程序集循环内**调用 `GetFullInteropPublishPath()`，而该函数每次都做 `ConvertRelativePathToFull` + `FPaths::Combine` + `GetMutableDefaultSafe` 查 CDO + 遍历 CustomProjects，属于可在循环外算一次的重复工作（见 F-FL-011）。
补充说明（**排除误报**）：`.dll` 后缀本身在 Linux/macOS 上是**正确**的 —— .NET 托管程序集在所有平台上都是 `.dll`；只有原生库才有 `.so`/`.dylib` 之分，而这一点在 `CoreMacro/Macro.h:85-91` 的 `LIB_HOSTFXR` 里已按平台正确分支。

**建议**
```cpp
// 在 GetFullAssemblyPublishPath 里标注缺失项（只 Warn 一次，避免每帧刷屏）
for (const FString& Path : Result)
    if (!FPaths::FileExists(Path))
        UE_LOG(LogUnrealCSharp, Warning, TEXT("Published assembly missing: %s"), *Path);
```
调用侧把 `GetFullInteropPublishPath()` 提到循环外缓存为局部常量。

**验证方式**
`grep "GetFullAssemblyPublishPath|GetFullInteropPublishPath" Source/`（`include=*.cpp`）→ **14 行命中 = 2 定义 + 1 库内调用 + 11 个外部调用点**（原文写"7 个"，已修正）；用例：删除 `Content/Script/Game.dll` 后启动，观察是否有任何日志。

---

### [F-FL-009] `SyncProcess` 无超时、无取消、在调用线程上同步忙等；无条件 `TerminateProc`；忽略 `CreateProc` 失败

- **类别**: 并发/线程安全 / 错误处理
- **严重度**: **P2**
- **复核结论**: 确认（"6 个调用点"经重跑证实；其中"3 个在游戏线程"这一分布未逐一回溯调用栈，见第 6 节）
- **可达性**: 活跃（`FCodeAnalysis.cpp:34,49,86` 在 `OnPostEngineInit` 路径上、`UnrealCSharpEditorSetting.cpp:152` 在设置面板路径上）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1543-1553`（`CreateProc` 返回值不检查）、`:1569`（`while (ProcessHandle.IsValid() && IsProcRunning(...))`）、`:1571`（`Sleep(0.01f)`，无超时/取消）、`:1585`（`InOnComplete` 无条件调用）、`:1587`（`ClosePipe`）、`:1589`（`TerminateProc` 无条件）、`:1591`（`CloseProc`）；**`grep "SyncProcess("`（`include=*.cpp`）共 7 行命中 = 1 定义（`:1520`）+ 6 调用**：`Compiler/Private/FCSharpCompilerRunnable.cpp:291,413`、`ScriptCodeGenerator/Private/FCodeAnalysis.cpp:34,49,86`、`Source/UnrealCSharpCore/Private/Setting/UnrealCSharpEditorSetting.cpp:152` —— 与报告"6 个调用点"一致
- **级别变动**: 无（P2 维持）
- **函数**: `FUnrealCSharpFunctionLibrary::SyncProcess(const FString&, const FString&, const TFunction<void(const int32, const FString&)>&, const FString&, const TFunction<void(const FString&)>&)`
- **置信度**: 高（本文件代码事实确定；调用线程已核实）

**现状（代码事实）**

```cpp
// :1543-1553
auto ProcessHandle = FPlatformProcess::CreateProc(*InURL, *InParms, false, true, true, nullptr, 1,
                                                  WorkingDirectory, WritePipe, ReadPipe);   // :1543 返回值不检查
...
// :1569-1574
while (ProcessHandle.IsValid() && FPlatformProcess::IsProcRunning(ProcessHandle))
{
	FPlatformProcess::Sleep(0.01f);          // :1571 调用线程上 10ms 轮询，没有超时/取消判断
	ReadOutput();
}
...
// :1585-1591
InOnComplete(ReturnCode, Result);            // :1585 无条件调用（空 TFunction 会崩）
FPlatformProcess::ClosePipe(ReadPipe, WritePipe);
FPlatformProcess::TerminateProc(ProcessHandle, true);   // :1589 进程已退出仍强制终止
FPlatformProcess::CloseProc(ProcessHandle);
```

**调用上下文**
`FCSharpCompilerRunnable.cpp:291,413`（**异步编译线程**：`static auto CompileTool = GetDotNet();` 后同步阻塞）、`FCodeAnalysis.cpp:34,49,86`（`FCodeAnalysis::CodeAnalysis()` 由 `FEditorListener.cpp:183 OnPostEngineInit` 在**游戏线程**调用）、`UnrealCSharpEditorSetting.cpp:152`（设置面板"刷新 dotnet 路径"下拉，游戏线程）。所有调用点的 `InOnComplete` 都是非空 lambda。

**问题**
1. **无超时/无取消**：`dotnet build` 若卡住（NuGet 还原等网络、锁文件、等待 stdin），循环永不退出；游戏线程上的调用（`OnPostEngineInit` / 设置面板）会**永久冻结编辑器**，编译线程上的调用会让"编译中"状态永远为真、取消按钮无效。函数签名里也没有 `bCancel`/`Timeout` 参数可传。
2. **无条件 `TerminateProc`**：循环退出条件本身就包含"进程已结束"，此处再 `TerminateProc` 是对已结束进程的操作，语义错误（掩盖了"循环为何退出"）；若进程只是 `IsProcRunning` 误报，会静默杀掉它。
3. **`CreateProc` 失败路径**：句柄无效时不报错、继续走 `ReadPipe`/`GetProcReturnCode` 失败 → `-1`，且 `ClosePipe/TerminateProc/CloseProc` 都作用在无效句柄上。
4. **`InOnComplete` 无条件调用**：当前所有调用点都传了 lambda，故不可触发；但默认参数 `InOnOutput = {}` 的存在说明作者允许空回调，`InOnComplete` 却没有对应的 `if (InOnComplete)` 保护（对比 `:1562` 对 `InOnOutput` 有保护）—— 风格不一致且是隐患。

**建议**
```cpp
if (!ProcessHandle.IsValid())
{
	UE_LOG(LogUnrealCSharp, Error, TEXT("Failed to launch: %s %s"), *InURL, *InParms);
	if (InOnComplete) { InOnComplete(-1, FString()); }
	return;
}
const double StartTime = FPlatformTime::Seconds();
constexpr double TimeoutSeconds = 600.0;         // 或由设置项给出
while (FPlatformProcess::IsProcRunning(ProcessHandle))
{
	FPlatformProcess::Sleep(0.01f);
	ReadOutput();
	if (FPlatformTime::Seconds() - StartTime > TimeoutSeconds)
	{
		UE_LOG(LogUnrealCSharp, Error, TEXT("dotnet build timeout, killing"));
		FPlatformProcess::TerminateProc(ProcessHandle, true);   // 只在超时时终止
		ReturnCode = -1;
		break;
	}
}
```
若确实需要非阻塞，改为 `FMonitoredProcess`（UE 自带异步 + 取消 + 超时支持）而不是自己写轮询。

**验证方式**
1. `grep -n "SyncProcess(" Source/` → 6 个调用点，确认其中 3 个在游戏线程；
2. 用例：把 `DotNetPath` 指向一个会挂起的脚本（如 `sleep 999`），触发编译，观察编辑器是否冻结、取消是否生效；
3. 加 `ensure(FPlatformProcess::IsProcRunning(ProcessHandle))` 到 `:1589` 前可验证"无条件终止"确实作用在已退出进程上。

---

### [F-FL-010] `Encode()`：每次调用线性扫描 77 个关键字；且映射非单射（`Object` → `__Object` 会与真实成员撞名）

- **类别**: 性能 / 隐患
- **严重度**: **P3**
- **复核结论**: 部分确认（偏差：原文"问题 (c) 大小写不敏感"**子命题证伪** —— 引擎侧已核实 `FString::Equals` 默认就是 `CaseSensitive`；主体（77 项线性扫描 + 非单射 `__X` 撞名）成立）
- **可达性**: 活跃
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1262`（`TInlineAllocator<77>`）、`:1263-1281`（**逐个数出正好 77 个字面量**，与 `InlineAllocator<77>` 一致 —— 原文这一计数已实证）、`:1284-1287`（`ContainsByPredicate` 线性扫描）、`:1289`（`FString::Printf(TEXT("__%s"), *InName)` 非单射）。**引擎侧裁决 (c)**：`Runtime/Core/Public/Containers/UnrealString.h.inl:1492` —— `bool Equals(const UE_STRING_CLASS& Other, ESearchCase::Type SearchCase = ESearchCase::CaseSensitive)`，**默认大小写敏感**，故 `Class`/`object` 等不同大小写的名字**不会**被改写；`:1504` 起的分支也印证该语义。原文自注的"未验证假设（默认可能是 IgnoreCase）"据此**撤销**，该子项不再计为缺陷
- **级别变动**: 无（P3 维持；(c) 证伪后只剩性能与撞名两项，均属 P3）
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1260-1293`（`:1262-1282,:1284-1290`）
- **函数**: `FUnrealCSharpFunctionLibrary::Encode(const FString&, bool, bool)`
- **置信度**: 中（代码事实确定；"非单射导致 TMap 键覆盖"的后果依赖调用方是否用 TMap::Add 去重，已核实 `FCSharpBind.cpp:155/194/206/253` 用 `TMap::Add`）

**现状（代码事实）**

```cpp
// :1260-1293
FString FUnrealCSharpFunctionLibrary::Encode(const FString& InName, const bool bIsNative, const bool bEncodeWideString)
{
	static TArray<FString, TInlineAllocator<77>> KeyWords{          // :1262 77 个 C# 关键字（与 InlineAllocator<77> 数量一致）
		TEXT("abstract"), TEXT("as"), ... TEXT("while") };          // :1263-1281

	if (KeyWords.ContainsByPredicate([&](const FString& Name)     // :1284 每次调用线性扫描 + FString 比较
	{
		return InName.Equals(Name);                               // :1286 Equals 的 SearchCase 参数被省略
	}))
	{
		return FString::Printf(TEXT("__%s"), *InName);            // :1289 非单射：任何 InName 都映射到 "__"+InName
	}
	return bIsNative ? InName : FNameEncode::Encode(InName, bEncodeWideString);
}
```

**调用上下文**
33 个 `Encode(...)` 调用点（`Select-String "FUnrealCSharpFunctionLibrary::Encode\("` 命中 36 处，其中 3 处是库内定义 `.cpp:1260,1295,1300`）：生成器侧 `FClassGenerator.cpp`（17 处）、`FDelegateGenerator.cpp`（4）、`FStructGenerator.cpp`/`FEnumGenerator.cpp`/`FGeneratorCore.cpp`/`FGameplayTagGenerator.cpp`；运行期 `FCSharpBind.cpp:155,194,206,253`（把属性/函数注册进 `TMap<FString, …>`）。属于"每个属性、每个函数各一次"的批量路径，不是每帧路径。

**问题**
(a) **性能**：每次编码都要在 77 个 `FString` 上做线性、大小写不敏感的 `Equals`，且 `static TArray` 首次构造 77 个 `FString`。改成 `static const TSet<FString>`（或小写 FName 哈希）是等价且 O(1) 的，代价可忽略。
(b) **非单射**：`Encode` 把 `X` 映射成 `__X`。若同一个类里既有关键字成员（如 `Object`）又恰好存在名为 `__Object` 的成员，两者编码结果相同；而 `FCSharpBind.cpp:155` 用 `TMap::Add`（已存在则覆盖）→ 静默丢一个成员，C# 侧永久访问不到。`Encode` 没有任何"结果是否已被占用"的校验。
(c) ~~**大小写不敏感**~~ **【证伪 —— 此子项不是缺陷，已撤销】**：`InName.Equals(Name)` 省略 `ESearchCase` 参数，但引擎的默认实参就是 **`CaseSensitive`**，源码依据 `Engine\Source\Runtime\Core\Public\Containers\UnrealString.h.inl:1492`：`bool Equals(const UE_STRING_CLASS& Other, ESearchCase::Type SearchCase = ESearchCase::CaseSensitive) const`（`:1504` 的分支亦印证）。因此 `Class`/`class`、`Object`/`object` **不会**被改写 —— 这是**正确**行为（C# 关键字大小写敏感）。原"按 UE 惯例默认 `IgnoreCase`"的推断是**错的**（`:1158` 显式写 `CaseSensitive` 只是作者按需显式化，并不能反推默认值不同）。本子项撤销，F-FL-010 只保留 (a) 性能 与 (b) 非单射两项。

**建议**
```cpp
static const TSet<FString> KeyWords = { TEXT("abstract"), ... };          // O(1)
static const TSet<FString> KeyWordsLower = /* 全小写副本 */;
if (KeyWords.Contains(InName)) { return FString::Printf(TEXT("__%s"), *InName); }   // 大小写敏感
```
并在编码结果上加唯一性断言（在 `FCSharpBind` 的 `Add` 处用 `Add` 前 `Contains` 检查 + `UE_LOG(Warning)`），把 (b) 类撞名暴露出来而不是静默覆盖。

**验证方式**
1. `grep -F "KeyWords.ContainsByPredicate" -A3` 确认比较方式；
2. 用例：给一个 UCLASS 加两个属性 `Object` 与 `__Object`，跑生成 + 绑定，检查 C# 侧是否少一个属性；
3. 性能：在 `:1284` 前后加 `TRACE_CPUPROFILER_EVENT_SCOPE`，对比 TSet 版本。

---

### [F-FL-011] 路径 getter 全部按值返回 `FString` 且每次调用重算；热循环与逐帧路径上无缓存

- **类别**: 性能 / 可优化
- **严重度**: **P2**
- **复核结论**: 确认
- **可达性**: 活跃（生成期每个类/资产都要算输出路径）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1024-1027`（`GetFullPublishDirectory` 按值返回且每次 `ConvertRelativePathToFull`）、`:1085-1088`（`GetFullScriptDirectory` 同型）、`:735-743`（`GetUEName` 按值返回，`:742 return DEFAULT_UE_NAME;` 每次构造临时 `FString`）、`:856-864`/`:866-874`（`GetOverrideFunctionNamePrefix/Suffix` 同型）。**底层 getter 确为引用返回**：`Source/UnrealCSharpCore/Public/Setting/UnrealCSharpSetting.h:100`（`const FString& GetUEName() const;`）、`:102`（`const FString& GetGameName() const;`）—— 与本库按值返回形成对照，原文这一论点实证成立。缓存矛盾点仍为 `:1001`/`:1007` 两处 `static auto`
- **级别变动**: 无（P2 维持）
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1024-1027`、`:1085-1088`、`:735-792`、`:856-896`；声明 `Source/UnrealCSharpCore/Public/Common/FUnrealCSharpFunctionLibrary.h:113-133,145-161,183-195,247`
- **函数**: `GetFullScriptDirectory()`、`GetFullPublishDirectory()`、`GetUEName()`、`GetGameName()`、`GetOverrideFunctionNamePrefix/Suffix()`、`GetUE/Game/CustomProjectsDirectory()`
- **置信度**: 高

**现状（代码事实）**

```cpp
// :1085-1088
FString FUnrealCSharpFunctionLibrary::GetFullScriptDirectory()
{
	return FPaths::ConvertRelativePathToFull(FPaths::ProjectDir() / GetScriptDirectory());   // 每次：ProjectDir 拼接 + 转全路径
}
// :1024-1027
FString FUnrealCSharpFunctionLibrary::GetFullPublishDirectory()
{
	return FPaths::ConvertRelativePathToFull(FPaths::ProjectContentDir() / GetPublishDirectory());
}
// :735-743
FString FUnrealCSharpFunctionLibrary::GetUEName()
{
	if (const auto UnrealCSharpSetting = GetMutableDefaultSafe<UUnrealCSharpSetting>())
	{ return UnrealCSharpSetting->GetUEName(); }        // :739 底层 getter 返回 const FString&（UnrealCSharpSetting.h:100），此处按值拷出
	return DEFAULT_UE_NAME;                              // :742 宏展开为 FString(TEXT("UE")) —— 每次都构造临时 FString
}
```
`GetUEName()` 21 处分命中 10 个文件；`GetGameName()` 18 处/7 文件；`GetFullScriptDirectory()` 15 处/5 文件；且它们互相嵌套（`GetGameDirectory()` → `GetFullScriptDirectory()`，`GetGameProxyDirectory()` → `GetGameDirectory()` …），一次 `GetGameProxyDirectory()` 会触发 3 层各一次 `ConvertRelativePathToFull`。

**调用上下文**
生成期（每个资产/每个类都要算输出路径，`FGeneratorCore`/`FClassGenerator`/`FAssetGenerator`）、`FEditorListener.cpp:89/115/436`（`GetChangedDirectories()` 每次文件变更回调都重算全部目录）、`FMonoDomain.cpp:406`/`FCoreCLRDomain.cpp:219`（程序集循环内）。

**问题**
1. 底层 `UUnrealCSharpSetting` 的 getter 已经是 `const FString&`（`UnrealCSharpSetting.h:98-110`），但本库把返回类型写成 `FString` → 每次调用多一次拷贝（并且返回 `DEFAULT_UE_NAME` 时构造临时对象）。
2. 组合式路径（`GetGameProxyDirectory`/`GetUEProxyDirectory`/`GetChangedDirectories`/`GetFullAssemblyPublishPath`）每层都重做 `FPaths::Combine` + `ConvertRelativePathToFull`，没有任何缓存；而**恰恰** `GetGenerationPath`（`:1001,:1007`）用了 `static auto` 缓存 —— 缓存策略在同一文件内自相矛盾（该缓存的反而稳定，不该缓存的没缓存；见 F-FL-012）。
3. 大字符串按值返回在 `FString::Printf` 参数里产生额外临时对象与引用计数操作（`*GetUEName()` 这种写法会在整条表达式中多持有一个临时）。

**建议**
- 把只读设置 getter 的返回类型改为 `const FString&`（`return DEFAULT_UE_NAME;` 处改用 `static const FString Default = TEXT("UE");`），或至少提供 `void GetUEName(FString& Out)` 形式的内部接口；
- 对纯派生路径（`GetFullScriptDirectory`/`GetFullPublishDirectory`/`GetGameDirectory`/`GetUEProxyDirectory`）用"设置变更时失效"的缓存：
```cpp
// 由 UUnrealCSharpSetting::PostEditChangeProperty / 配置重载时调用
static void InvalidatePathCache();
```
- `FMonoDomain.cpp:406`/`FCoreCLRDomain.cpp:219` 的循环内调用提到循环外。

**验证方式**
1. `grep -n "static auto" Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp` → 只有 `:1001,:1007`（缓存策略矛盾点）；
2. 在 `:1087`/`:1026` 加计数器，跑一次全量生成统计调用次数；
3. 用 `TRACE_CPUPROFILER_EVENT_SCOPE` 对比缓存前后。

---

### [F-FL-012] `GetChangedDirectories()` 硬编码 `DEFAULT_UE_NAME` 且根目录与 `GetUEDirectory()` 不同

- **类别**: 可优化/一致性（潜在功能错误）
- **严重度**: **P3**
- **复核结论**: 确认
- **可达性**: 潜伏（需把 `UEName` 改成非 `"UE"` 值才触发不一致）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1253`（`Add(GetPluginScriptDirectory() / DEFAULT_UE_NAME)` —— 插件内、硬编码字面量）vs `:748`（`return GetFullScriptDirectory() / GetUEName();` —— 项目内、可配置）；`Source/UnrealCSharpCore/Public/CoreMacro/Macro.h:19`（`DEFAULT_UE_NAME` = `FString(TEXT("UE"))`）；`Source/UnrealCSharpCore/Public/Setting/UnrealCSharpSetting.h:132-133`（`UPROPERTY(Config, EditAnywhere, ...) FString UEName;` —— 确认可被用户改）
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1249-1258`（`:1253`）vs `:745-759`（`:748`）
- **函数**: `GetChangedDirectories()` / `GetUEDirectory()`
- **置信度**: 中（两处不一致是代码事实；哪一处才是"对的"取决于设计意图，见"问题"）

**现状（代码事实）**

```cpp
// :1249-1257
TArray<FString> FUnrealCSharpFunctionLibrary::GetChangedDirectories()
{
	return TArrayBuilder<FString>().
	       Add(GetPluginScriptDirectory() / DEFAULT_UE_NAME).   // :1253 插件 Script/UE（硬编码 "UE"）
	       Add(GetGameDirectory()).                             // :1254 项目 Script/<GameName>
	       Append(GetCustomProjectsDirectory()).
	       Build();
}
// :745-749
FString FUnrealCSharpFunctionLibrary::GetUEDirectory()
{
	return GetFullScriptDirectory() / GetUEName();              // :748 项目 Script/<UEName>
}
```

磁盘现状（实测）：插件侧 `Plugins/UnrealCSharp/Script/{UE,Interop,CodeAnalysis,SourceGenerator,Weavers}`；项目侧 `Script/{UE,Game,Interop,CodeAnalysis,SourceGenerator,Weavers,Script.sln,Shared.props}`。两个 `UE` 目录**都存在**。

**调用上下文**
`FEditorListener.cpp:89`（启动时注册 `DirectoryWatcher`）、`:115`（析构注销）、`:436`（文件变更过滤，与 `PROXY_NAME`/`obj` 一起判断是否忽略）。`GetUEDirectory()` 则被 `GetUEProxyDirectory()`（`:751-754`）与 `GetUEProjectPath()`（`:756-759`）使用，即决定生成代码的落地目录。

**问题**
同一个概念"UE 目录"在这两个函数里指向**不同根**且用了不同命名来源：`GetChangedDirectories` 用 `GetPluginScriptDirectory()/DEFAULT_UE_NAME`（插件内、字面量 `"UE"`，`Macro.h:19`），`GetUEDirectory` 用 `GetFullScriptDirectory()/GetUEName()`（项目内、可配置）。后果：用户在设置里把 `UEName` 改成非 `"UE"`（`UnrealCSharpSetting.h:132-133` 是 `EditAnywhere`）之后，生成产物落到 `Project/Script/<自定义名>/`，而目录监视器仍在看 `Plugin/Script/UE` —— **项目侧源码改动不再触发重新编译**，且没有任何提示。两处必有一处是笔误（若监视插件侧是有意的，则 `DEFAULT_UE_NAME` 是对的、`GetUEName()` 才是可疑的配置项；反之亦然）。附带：`IgnoreDirectories` 只忽略 `PROXY_NAME` 与 `obj`，不忽略 `bin`。

**建议**
明确二者语义后写清注释并统一：若"监视的是插件内随版本发布的 C# 绑定源"，应改为 `GetPluginScriptDirectory()/GetUEName()` 或直接定义一个 `GetPluginUEDirectory()`，与 `GetUEDirectory()` 并列命名；若只需要项目侧，则改为 `GetUEDirectory()`。建议同时补 `TEXT("bin")` 到忽略列表（`:424-428` 在 `FEditorListener.cpp`）。

**验证方式**
1. `grep -n "DEFAULT_UE_NAME\|GetUEName()" Source/` → 确认 `:1253` 与 `:748` 的差异；
2. 用例：把 `UEName` 设为 `UE2`，重启编辑器，对比 `Script/` 下产物目录与 `DirectoryWatcher` 注册的目录（在 `:1253` 打断点）。

---

### [F-FL-013] `ScriptDomainType` 是可被跨模块写入的可变静态全局（RAII 保存/恢复）

- **类别**: 并发/线程安全 / 设计隐患
- **严重度**: **P2**
- **复核结论**: 确认
- **可达性**: 潜伏（当前无跨线程读取，属"设计上不设防"）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:36`（`EScriptDomainType ... ScriptDomainType = EScriptDomainType::CoreCLR;`）、`:38`（`bool ... bScriptChanged{};`）、`:1596-1599`（`SetScriptDomainType` 裸写）、`:1611-1614`（`GetScriptDomainType()` 裸读）；RAII 写入方 `Source/UnrealCSharpCore/Public/Common/FScriptDomainTypeScope.h:9-18`（构造保存 + 覆盖、析构恢复，已 read 确认）；**调用点 grep 重跑**（`FScriptDomainTypeScope|GetScriptDomainType\(\)`）→ 写入仅 `Source/UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp:103,316`，读取 `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:212`（`== EScriptDomainType::LeanCLR`）—— 全在游戏线程，原文"未发现编译线程读取"成立。默认值 `CoreCLR`（`:36`）在 commandlet 场景下被当真值使用这一点亦成立
- **级别变动**: 无（P2 维持）
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:35-39`、`:1595-1615`；写入方 `Source/UnrealCSharpCore/Public/Common/FScriptDomainTypeScope.h:9-18`
- **函数**: `FUnrealCSharpFunctionLibrary::SetScriptDomainType(EScriptDomainType)`、`GetScriptDomainType()`
- **置信度**: 中（静态可变全局是代码事实；"当前是否存在跨线程读"我做了 grep 但无法排除外部/未来调用）

**现状（代码事实）**

```cpp
// :35-39
#if WITH_EDITOR
EScriptDomainType FUnrealCSharpFunctionLibrary::ScriptDomainType = EScriptDomainType::CoreCLR;   // :36 默认 CoreCLR
bool FUnrealCSharpFunctionLibrary::bScriptChanged{};                                              // :38
#endif
// :1595-1614
void FUnrealCSharpFunctionLibrary::SetScriptDomainType(const EScriptDomainType InScriptDomainType) { ScriptDomainType = InScriptDomainType; }
EScriptDomainType FUnrealCSharpFunctionLibrary::GetScriptDomainType() { return ScriptDomainType; }
```
```cpp
// FScriptDomainTypeScope.h:9-18 —— 唯一的写入路径（RAII）
explicit FScriptDomainTypeScope(const EScriptDomainType InScriptDomainType) :
	ScriptDomainType(FUnrealCSharpFunctionLibrary::GetScriptDomainType())          // 保存
{ FUnrealCSharpFunctionLibrary::SetScriptDomainType(InScriptDomainType); }         // 覆盖
~FScriptDomainTypeScope() { FUnrealCSharpFunctionLibrary::SetScriptDomainType(ScriptDomainType); }  // 恢复
```

**调用上下文**
写入方：`UnrealCSharpEditor.cpp:316`（`Generator()`，游戏线程，作用域覆盖整段生成流程）、`:103`（控制台命令）。读取方：`FSolutionGenerator.cpp:212`（决定是否追加 `WITH_LEANCLR` 宏定义）、`FScriptDomainFactory`（另见 F-FL-001）。

**问题**
1. 这是一个**进程级可变全局**，语义是"当前正在为哪个平台生成代码"。RAII 保证了嵌套恢复，但它把"当前平台"这种本应沿调用链传递的上下文放到了全局，任何在作用域外/跨线程的读取都会拿到另一个平台的值。我 grep 了全部 `GetScriptDomainType` 调用点，**没有**发现编译线程上的读取（编译线程只读 `GetDotNet`/路径），所以当前不是活跃的数据竞争；风险属于"设计上不设防"。
2. 默认值 `CoreCLR`（`:36`）在 `SetScriptDomainType` 从未被调用的场景（例如 commandlet、非编辑器入口）下会被当真值使用，`FSolutionGenerator.cpp:212` 会据此生成**不含** `WITH_LEANCLR` 的常量 —— 与 F-FL-001 同源。

**建议**
- 若必须在静态存储上保存，至少加线程断言：`SetScriptDomainType` 内 `check(IsInGameThread())`，`GetScriptDomainType()` 内 `check(IsInGameThread())`（或改成 `TAtomic`/`thread_local`）；
- 更好的做法是把 `ScriptDomainType` 作为参数沿 `FSolutionGenerator::Generator(Platform)` 传递，删除全局。

**验证方式**
`grep -rn "GetScriptDomainType\|SetScriptDomainType" Source/` → 确认写入只在 `FScriptDomainTypeScope`；在使用点加 `check(IsInGameThread())` 后跑一次全平台生成（含 Android/iOS 目标）观察是否触发。

---

### [F-FL-014] `AccessPrivateMacro` 的三个核心弱点：全局类型名按短名拼接、依赖非标准访问控制放宽、依赖动态初始化

- **类别**: 未定义行为 / 可移植性
- **严重度**: **P2**
- **复核结论**: 确认（代码事实与宏展开形态全部核实；"文件作用域取私有成员地址是否被标准保证"仍属**未验证假设**，无编译器矩阵可验证，置信度维持 中）
- **可达性**: 活跃（`FCSharpEnvironment.cpp:397` 在 Runtime 模块，Shipping 亦编入）
- **复核证据**: `Source/UnrealCSharpCore/Public/CoreMacro/AccessPrivateMacro.h:6-11`（`ACCESS_PRIVATE_MEMBER_PROPERTY`：`struct Class##_##PropertyName` 文件作用域 + `&Class::PropertyName`）、`:13-18`、`:20-25`、`:27-32`、`:34-39`（共 5 个宏，与原文一致）；`Source/UnrealCSharpCore/Public/Template/TAccessPrivate.inl:6`（`static inline typename T::Type Value;` 可写全局）；`Source/UnrealCSharpCore/Public/Template/TAccessPrivateStub.inl:12`（构造期写入）、`:16`（`static inline FAccessPrivateStub AccessPrivateStub;` 动态初始化）
- **级别变动**: 无（P2 维持）
- **文件**: `Source/UnrealCSharpCore/Public/CoreMacro/AccessPrivateMacro.h:6-39`、`Source/UnrealCSharpCore/Public/Template/TAccessPrivate.inl:3-7`、`Source/UnrealCSharpCore/Public/Template/TAccessPrivateStub.inl:5-17`
- **函数**: `ACCESS_PRIVATE_MEMBER_PROPERTY(Class, PropertyName, PropertyType)` 等 5 个宏 + `TAccessPrivate<T>` / `TAccessPrivateStub<T, Value>`
- **置信度**: 中（宏展开与初始化模型是代码事实；"标准是否允许在文件作用域取私有成员地址"需要 MSVC/Clang/GCC **编译器矩阵实测**，与本机有无引擎源码无关 —— 该假设保持"未验证"）

**现状（代码事实）**

```cpp
// AccessPrivateMacro.h:6-11
#define ACCESS_PRIVATE_MEMBER_PROPERTY(Class, PropertyName, PropertyType) \
struct Class##_##PropertyName \            // 生成**全局**类型名 Class_PropertyName（无命名空间/无前缀）
{ \
	typedef PropertyType (Class::*Type); \
}; \
template struct TAccessPrivateStub<Class##_##PropertyName, &Class::PropertyName>;   // 在文件作用域形成 &Class::私有成员
```
```cpp
// TAccessPrivateStub.inl:5-17
template <class T, typename T::Type Value>
struct TAccessPrivateStub
{
	struct FAccessPrivateStub
	{
		FAccessPrivateStub() { TAccessPrivate<T>::Value = Value; }   // :12 构造期写入
	};
	static inline FAccessPrivateStub AccessPrivateStub;              // :16 inline static，动态初始化
};
// TAccessPrivate.inl:5-7
template <class T> struct TAccessPrivate { static inline typename T::Type Value; };   // :6 可写全局
```
三处使用（全部实测）：
```cpp
// Source/UnrealCSharpCore/Private/Dynamic/FDynamicClassGenerator.cpp:22
ACCESS_PRIVATE_MEMBER_PROPERTY(FObjectInitializer, bIsDeferredInitializer, bool)
//   ... 消费点 :928
	ObjectInitializer.*TAccessPrivate<FObjectInitializer_bIsDeferredInitializer>::Value = true;
// Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:16
ACCESS_PRIVATE_MEMBER_PROPERTY(UObjectBase, ObjectFlags, EObjectFlags)
//   ... 消费点 :397（未被任何 WITH_EDITOR 包住，运行期/Runtime 模块）
	if (Object->*TAccessPrivate<UObjectBase_ObjectFlags>::Value & ~RF_AllFlags || ...
// Source/ScriptCodeGenerator/Private/FGameplayTagGenerator.cpp:13
ACCESS_PRIVATE_MEMBER_PROPERTY(FGameplayTagNode, DevComment, FString)
```

**调用上下文**
`FCSharpEnvironment.cpp:397` 位于 `UnrealCSharp`（**Runtime** 模块）中一段遍历 `LocalAsyncLoadingObjectArray` 的逻辑里 —— 即"访问私有成员"这套机制在 Shipping 下也会被编译进运行时代码，而不只是编辑器工具（回答任务书疑问：`TAccessPrivateStub.inl` **不**只在 `WITH_EDITOR` 下有效，`UnrealCSharpCore/.../FDynamicClassGenerator.cpp:22` 与 `UnrealCSharp/.../FCSharpEnvironment.cpp:16` 都是运行期模块；只有 `FGameplayTagGenerator.cpp` 属于编辑器模块）。

**问题**
1. **命名冲突（可确定）**：`Class##_##PropertyName` 生成的是**文件作用域**类型名，只取类的**短名**。两个不同命名空间下的同名类（或同名类的同名属性）在同一 TU 中同时使用该宏，会生成同名 struct → 重定义/ODR 错误。`UObjectBase`、`FObjectInitializer` 这类名字在引擎里曾有别名/历史版本，风险并非纯理论。
2. **依赖非标准的访问控制放宽**：宏在文件作用域形成 `&Class::PrivateMember` 并把它作为非类型模板实参，宏本身**没有**任何 `friend` 声明来授予访问权（对比 C++ 界经典的 "Rob" 技巧，其可移植性依赖 `friend` 声明 + 特定编译器的宽松实现）。标准对"模板实参中的名字"的访问检查与"显式实例化"的关系在这里是被当作可利用的灰色地带。**未验证**：无法在本机取得 MSVC/Clang/GCC 三套编译器矩阵来确认（这与"有无引擎源码"无关）；`置信度: 中`。可确定的是这种做法没有标准保证，且一旦被 LTO/链接器优化或编译器收紧诊断影响，症状是**静默取到错误的地址**而不是编译失败。
3. **依赖动态初始化**：`TAccessPrivate<T>::Value` 只在 `TAccessPrivateStub<...>::AccessPrivateStub` 的构造函数里被赋值（`TAccessPrivateStub.inl:12`）。标准**不保证**这个 inline static 的动态初始化与*其它 TU 的*动态初始化之间的顺序。当前三个消费点都在函数体内（运行期读取），因此现在不触发；但任何"静态初始化期读取该值"的新代码都会读到空指针并调用它 → 崩溃。
4. **Shipping 也带着这个 hack**（`FCSharpEnvironment.cpp:397`），使得"运行时代码依赖 `UObjectBase::ObjectFlags` 的私有偏移"成为发布构建的一部分。

**建议**
- 宏生成的类型名加模块前缀并放进命名空间：`#define ACCESS_PRIVATE_MEMBER_PROPERTY(Class, PropertyName, PropertyType) namespace UnrealCSharpAccess { struct Class##_##PropertyName { ... }; template struct TAccessPrivateStub<Class##_##PropertyName, &Class::PropertyName>; }`；
- 给 `TAccessPrivate<T>::Value` 的读取加断言：`check(TAccessPrivate<T>::Value != nullptr)`（或改用返回引用的包装函数，在首次调用时懒加载）；
- 长期方案：优先用引擎公开 API（例如 `UObjectBase` 的 flags 访问器、`FObjectInitializer` 的 `IsDeferredInitializer()` 之类的公开查询，若引擎不提供则把该能力封装到唯一的 `.cpp` 里并注明"依赖引擎内部布局，升级引擎时必须复核"）；把 `FCSharpEnvironment.cpp:397` 的 Shipping 依赖也一并评估。

**验证方式**
1. `grep -rn "ACCESS_PRIVATE" Source/` → 5 个宏定义、3 个使用点、4 个零使用的宏（见死代码表）；
2. `grep -rn "TAccessPrivate<" Source/` → 3 个消费点；
3. 在一个 TU 内对两个不同命名空间的同名类各展开一次该宏，观察是否重定义。

---

### [F-FL-015] `TIsUStruct` 与 `TIsScriptStruct` 可同时为真 → `FCSharpEnvironment::TGetObject` 的 4 个部分特化存在歧义隐患

- **类别**: 隐患 / 未定义行为
- **严重度**: **P2**
- **复核结论**: 确认（trait 定义、消费点条件重叠、当前未触发三件事全部核实）
- **可达性**: 潜伏（`TGetObject<` 的现有实例化都把"成员所属 UObject 类"当第一个模板实参，故只命中 `:245`，歧义未实例化）
- **复核证据**: `Source/UnrealCSharpCore/Public/Template/TIsUStruct.inl:5-12`（`Test(decltype(&U::StaticStruct))` SFINAE，`:12` 取 `Value`）、`Source/UnrealCSharpCore/Public/Template/TIsScriptStruct.inl:3-7`（主模板恒 `false`）、`Source/UnrealCSharpCore/Public/Template/TIsReflectionClass.inl:7-11`（`:10` 用 `||` 组合三者）；消费方 `Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h:239-242`（主模板）、`:244-253`（`enable_if_t<TIsUObject<T>::Value>`）、`:255-264`（`TIsUStruct`）、`:266-275`（`TIsScriptStruct`）、`:277-289`（`!(...)` 三否定）—— **四个部分特化行号与原文完全一致**，且 `:255` 与 `:266` 两个 `enable_if` 表达式互不蕴含，确认"歧义隐患"这一机制成立
- **级别变动**: 无（P2 维持）
- **文件**: `Source/UnrealCSharpCore/Public/Template/TIsUStruct.inl:3-13`、`Source/UnrealCSharpCore/Public/Template/TIsScriptStruct.inl:3-7`、`Source/UnrealCSharpCore/Public/Template/TIsReflectionClass.inl:7-11`；消费方 `Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h:239-289`
- **函数**: `TIsUStruct<T>` / `TIsScriptStruct<T>` / `FCSharpEnvironment::TGetObject<T, Enable>`
- **置信度**: 中（trait 定义与消费点的条件重叠是代码事实；"是否真的会实例化出歧义"我核对了全部 5 个 `TGetObject<` 调用点，当前不触发）

**现状（代码事实）**

```cpp
// TIsUStruct.inl:3-13 —— 用 StaticStruct 的存在性做 SFINAE
template <typename T> struct TIsUStruct
{
	template <typename U> static std::true_type  Test(decltype(&U::StaticStruct));
	template <typename U> static std::false_type Test(...);
	enum { Value = std::is_same_v<decltype(Test<T>(0)), std::true_type> };
};
// TIsScriptStruct.inl:3-7 —— 主模板恒 false，靠显式/部分特化打开
template <typename T> struct TIsScriptStruct { enum { Value = false }; };
// 由 Source/UnrealCSharp/Public/Macro/BindingMacro.h:164-168 显式打开：
//   template <> struct TIsScriptStruct<Class> { enum { Value = true }; };
// 注册清单 Source/UnrealCSharp/Public/Binding/ScriptStruct/TScriptStruct.inl:5-130
//   例如 BINDING_SCRIPT_STRUCT(FVector)（:17）、BINDING_SCRIPT_STRUCT(FTransform)（:9）
```
```cpp
// FCSharpEnvironment.h:239-289 —— 4 个互斥性靠 enable_if 表达，但条件彼此不互斥
template <typename T> class TGetObject<T, std::enable_if_t<TIsUObject<T>::Value, T>> { ... };        // :245
template <typename T> class TGetObject<T, std::enable_if_t<TIsUStruct<T>::Value, T>> { ... };        // :256
template <typename T> class TGetObject<T, std::enable_if_t<TIsScriptStruct<T>::Value, T>> { ... };   // :267
template <typename T> class TGetObject<T, std::enable_if_t<!(TIsUObject<T>::Value || TIsUStruct<T>::Value || TIsScriptStruct<T>::Value), T>> { ... };  // :278
```

**问题**
`TIsScriptStruct<X>`（白名单显式特化）与 `TIsUStruct<X>`（`StaticStruct` 存在性 SFINAE）是**两个正交判据**：任何用 `BINDING_SCRIPT_STRUCT` 注册的引擎 USTRUCT（`FVector`/`FTransform`/…）同时满足两者。一旦某个类型 T 同时命中 `:256` 与 `:267`，两个部分特化都不比对方更特化（条件是不同的 `enable_if` 表达式）→ **歧义错误**，这也说明这组特化把"UStruct"与"ScriptStruct"误当成互斥分类。我逐个核对了 `TGetObject<` 的 5 个调用点（`TPropertyBuilder.inl:44,60,92,111,309`、`TSubscriptHelper.inl:14,27`、`TFunctionHelper.inl:39`）：它们的第一个模板实参都是**成员所属的 UObject 类**（`FoundObject->*Member` 形式），因此当前只命中 `:245`，隐患**未触发**。`TIsReflectionClass`（`TIsReflectionClass.inl:10`）用 `||` 组合三者，本身没有歧义，但它把"三类"当成互斥分类的同一个误解带到了别处。

**建议**
- 让 `TIsScriptStruct` 明确表达"已注册的脚本结构"并在消费点用**优先级**而非并列 `enable_if` 表达（例如 `TGetObject` 只保留 `TIsUObject` 与 `TIsScriptStruct || TIsUStruct` 两个分支）：
```cpp
template <typename T> class TGetObject<T, std::enable_if_t<TIsUObject<T>::Value, T>> { ... };
template <typename T> class TGetObject<T, std::enable_if_t<!TIsUObject<T>::Value &&
                                                           (TIsUStruct<T>::Value || TIsScriptStruct<T>::Value), T>> { ... };
template <typename T> class TGetObject<T, std::enable_if_t<!(TIsUObject<T>::Value || TIsUStruct<T>::Value ||
                                                             TIsScriptStruct<T>::Value), T>> { ... };
```
- 给 `TIsScriptStruct` 加注释说明它**不是** `TIsUStruct` 的子集判据，二者可同时为真。

**验证方式**
1. `grep -rn "TIsScriptStruct" Source/` 与 `grep -rn "BINDING_SCRIPT_STRUCT(" Source/` → 确认同一类型同时命中两条路径；
2. 写一个编译期探针：`using X = FCSharpEnvironment::TGetObject<FVector, FVector>;` → 预期出现 "ambiguous partial specializations" 报错，即可确认歧义真实存在（当前代码库没走到这里）。

---

### [F-FL-016] `TFunctionPointer` 用 union 做函数指针↔`void*` 类型双关，标准不保证

- **类别**: 未定义行为 / 平台兼容
- **严重度**: **P2**
- **复核结论**: 部分确认（偏差：`T` 是否**全部**为自由/静态函数指针这一子命题未完全证实，见复核证据；union 双关 + 19 处实例化计数成立）
- **可达性**: 活跃（绑定注册期，模块启动即执行）
- **复核证据**: `Source/UnrealCSharpCore/Public/Template/TFunctionPointer.inl:3-17`（`:6-9` 只写 `Value.Function`、`:11-16` union 含 `T Function;` 与 `:15 void* Pointer;`）；**`grep "TFunctionPointer"` 全 `Source/` 命中 19 处**，与原文一致：定义文件 2（`:4,:6`）、`FDynamicClassGenerator.h:4,10`、`FClassBuilder.inl:11,32,34,53`、`FClassBuilder.h:59`、`TClassBuilder.inl:91,93`、`Macro/BindingMacro.h:18,262,272,288,298,308,310`、`FClassBuilder.cpp:78`；其中 `BindingMacro.h:262,272,288,298,308,310` **正好 6 个宏**读 `.Value.Pointer`，与原文"6 个宏"一致。**未证实项**：`FClassBuilder.inl:11/32/34` 的实参类型是模板形参 `T`/`InMethod`/`InGetMethod`/`InSetMethod`（`FClassBuilder.inl:4-5,23-25`），其真实类型由各 `BINDING_*` 宏展开决定，未逐一定位全部展开点，故"当前实例全为普通函数指针"**未穷尽验证**
- **级别变动**: 无（P2 维持）
- **文件**: `Source/UnrealCSharpCore/Public/Template/TFunctionPointer.inl:3-17`
- **函数**: `TFunctionPointer<T>::TFunctionPointer(const T&)`
- **置信度**: 中（条款判断属标准知识；实际使用点已核实，未做多平台编译验证）

**现状（代码事实）**

```cpp
// TFunctionPointer.inl:3-17
template <typename T>
struct TFunctionPointer
{
	explicit TFunctionPointer(const T& InFunction) { Value.Function = InFunction; }   // :6-9 只写 union 的 Function 成员
	union
	{
		T Function;
		void* Pointer;                                                                // :15 之后读 Pointer 成员
	} Value;
};
```
```cpp
// 消费点 Source/UnrealCSharp/Public/Macro/BindingMacro.h:262
#define BINDING_FUNCTION_BUILDER_INVOKE(Function) \
	TFunctionPointer<decltype(&TFunctionBuilder<decltype(Function), Function>::Invoke)>(&TFunctionBuilder<decltype(Function), Function>::Invoke).Value.Pointer
```
共 19 处命中（`FClassBuilder.inl:11,32,34,53`、`TClassBuilder.inl:91,93`、`BindingMacro.h:262,272,288,298,308,310` 等），其中 6 个宏把它当作 `void*` 塞进一个函数指针表（P/Invoke 注册表）。

**调用上下文**
绑定注册期（模块启动时构造生成好的 C++ 绑定表），把成员/静态函数指针转换成统一的 `void*` 交给 C# 侧 `MethodBridge`/`GetFunctionPointer`。

**问题**
1. 从 union 的一个成员写入、从另一个成员读出：C++ 允许读 union 的"非活跃"成员仅限共同初始序列等受限情形，函数指针与 `void*` 不构成共同初始序列 → **未定义行为**（实践中 MSVC/GCC/Clang 都按位重解释，符合"能跑"的预期，但不会被标准保证，也不受 LTO 保护）。
2. 函数指针 → `void*` 的转换本身在标准里是实现定义（POSIX 要求支持 `dlsym` 场景，Windows 上 MSVC 支持），x86 上 `__stdcall`/`__cdecl` 在"数据指针大小相同"的前提下才成立；对**成员函数指针**（MSVC 下可能是 4/8/16 字节的多重继承/虚继承 thunk 结构）该 union 的 `Pointer` 成员读出的将是前 8 字节，值不等于可调用地址。当前实例化的 `T` 是自由函数/静态成员函数指针（`&TFunctionBuilder<...>::Invoke` 是静态成员函数，属于普通函数指针），所以现在成立；但模板本身对 `T` 无约束，一旦有人传入成员函数指针就会静默错。
3. `TGetUtf8String` 与 `TFunctionPointer` 都存在"隐式 ABI 约定"却没有注释说明。

**建议**
- 改用标准工具并加静态断言：
```cpp
template <typename T>
struct TFunctionPointer
{
	static_assert(std::is_pointer_v<T> && std::is_function_v<std::remove_pointer_t<T>>,
	              "TFunctionPointer only supports free/static function pointers");
	explicit TFunctionPointer(T InFunction) { Value = reinterpret_cast<void*>(InFunction); }
	void* Value;
};
```
- 或提供 `void* GetPointer() const { return reinterpret_cast<void*>(Function); }` 并保留原始指针，避免 union。
- 若必须保留 union，至少加 `static_assert(sizeof(T) == sizeof(void*))` 并在注释里写明依赖平台实现定义行为。

**验证方式**
`grep -rn "TFunctionPointer<" Source/` → 确认当前全部实例都是静态成员函数；加 `static_assert` 后可编译性验证。

---

### [F-FL-017] `TFieldIteratorExt` 复制了引擎 `TFieldIterator` 的内部实现且无版本守卫

- **类别**: 平台/版本兼容（可维护性）
- **严重度**: **P2**
- **复核结论**: 确认（且**结论强于原文**：不是"将来可能静默分叉"，而是**已经分叉**；原文"工作区无引擎源码、无法逐行 diff"的前提已撤销，已逐行比对）
- **可达性**: 活跃（唯一使用点 `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:239`，模块启动时注册绑定）
- **复核证据（与 UE 5.6 引擎逐行比对实测）**：引擎类定义 `Engine\Source\Runtime\CoreUObject\Public\UObject\UnrealType.h:6658-6796`。
  1) **完全一致**：成员声明与注释（引擎 `:6662-6673` ≡ 本文件 `:9-20`）、主构造的 `EFieldIterationFlags` 位标志形态（引擎 `:6676-6685` ≡ 本文件 `:23-32`）、遗留三 flag 构造（引擎 `:6688-6694` ≡ 本文件 `:35-41`，且引擎 5.6 **仍保留**该重载）、`operator bool`/`!`/`++`/`*`/`->`/`GetStruct`（引擎 `:6696-6734` ≡ 本文件 `:43-81`）、`IterateToNext` 的骨架与 `Struct=/Field=` 收尾（引擎 `:6736-6764,6790-6795` ≡ 本文件 `:83-108,135-140`）、以及 `// We shouldn't be able to get here for non-classes` 注释（引擎 `:6768` ≡ 本文件 `:112`）。
  2) **已分叉（文本级）**：引擎已改写为 `for (; CurrentField; CurrentField = CurrentField->Next)` + 两处 early `continue`（`UnrealType.h:6743-6759`），本文件仍是 `while (CurrentField)` + 单个复合 `if`（`TFieldIteratorExt.inl:90-108`）——语义等价但文本不一致，**证明该拷贝已不再与引擎同步维护**。
  3) **已分叉（语义级，本报告新增的关键结论，也是原文未发现的一点）**：本文件**新增了引擎没有的** `GetAllInterfaceClasses()`（`TFieldIteratorExt.inl:142-167`：从 `InClass->Interfaces` 出发、沿 `GetSuperClass()` 收集 `CLASS_Interface`、用 `Contains` 去重），并把引擎的 `CurrentClass->Interfaces[InterfaceIndex]`（`UnrealType.h:6771-6774`）替换为该函数的**扁平化+去重**结果（本文件 `:110-121`）。**引擎 `TFieldIterator` 中不存在同名函数**（`grep GetAllInterfaceClasses` 于 `UnrealType.h` 无命中）→ 接口字段的遍历集合与顺序**与引擎不同**，属"有意增强但无注释说明"。
  4) **已分叉（次要）**：`operator==`/`operator!=` 由引擎的成员函数（`UnrealType.h:6707-6708`）改成了 friend 函数（本文件 `:54-55`）；引擎另有 `TFieldRange`（`UnrealType.h:6798+`）未被拷贝。
  5) 该文件**仍然没有任何 `UE_VERSION` 守卫**（`TFieldIteratorExt.inl:1-3` 仅 `#pragma once` + `#include "CoreMinimal.h"`），`CrossVersion/Public/UEVersion.h` 里也没有对应宏 —— 与原文结论一致
- **级别变动**: 无（P2 维持；不上调：`GetAllInterfaceClasses` 是**有意增强**而非漏抄，当前无"绑定字段缺失"的实证；但性质应从"潜在漂移风险"改写为"已发生且无守卫/无注释的既有分叉"）
- **文件**: `Source/UnrealCSharpCore/Public/Template/TFieldIteratorExt.inl:1-168`（重点 `:23,:29,:54-55,:90-108,:110-121,:142-167`）
- **函数**: `TFieldIteratorExt<T>::IterateToNext()`、`GetAllInterfaceClasses()`
- **置信度**: **高**（已对 UE 5.6 引擎源码完成逐行比对，"本文件是 `TFieldIterator` 的拷贝"由结构与注释一致 + 两处已定位的分叉共同证实；由"中"上调为"高"）

**现状（代码事实）**

```cpp
// TFieldIteratorExt.inl:25-29
, Field ( InStruct ? GetChildFieldsFromStruct<typename T::BaseFieldClass>(InStruct) : NULL )
...
, bIncludeInterface ( EnumHasAnyFlags(InIterationFlags, EFieldIterationFlags::IncludeInterfaces) && InStruct && InStruct->IsA(UClass::StaticClass()) )
// :94-99  直接使用引擎的类转换旗标与 FProperty 布局
if (FieldClass->HasAllCastFlags(T::StaticClassCastFlags()) &&
	( bIncludeDeprecated || !FieldClass->HasAllCastFlags(CASTCLASS_FProperty)
	  || !((FProperty*)CurrentField)->HasAllPropertyFlags(CPF_Deprecated) ))
// :110-121  自己实现接口迭代（含注释 "We shouldn't be able to get here for non-classes"）
// :142-167  GetAllInterfaceClasses()：从 InClass->Interfaces 出发，沿 GetSuperClass() 收集 CLASS_Interface
```
唯一使用点：`Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:239`
```cpp
for (TFieldIteratorExt<UFunction> It(FoundClass, EFieldIteratorFlags::IncludeSuper, ...))
```

**问题**
1. 这份实现与引擎 `TFieldIterator` 高度同构（连注释都保留了），但**没有任何 `UE_VERSION` 守卫**。引擎一旦改动 `GetChildFieldsFromStruct`、`GetInheritanceSuper`、`EFieldIterationFlags` 的位值或 `GetAllInterfaceClasses` 的语义，此处会**静默分叉**：编译通过、遍历结果不同（少/多字段），表现为"某些 C# 绑定字段莫名缺失"。对比本插件其它版本适配点都集中在 `CrossVersion/Public/UEVersion.h`（该文件里确实有 `UE_T_IS_T_ENUM_AS_BYTE`、`UE_F_OPTIONAL_PROPERTY` 等 40+ 个宏），此处是个明显缺口。
2. `:142-167` 的 `GetAllInterfaceClasses` 只走 `InClass->Interfaces` + `GetSuperClass()` 链，不处理"父类实现了某接口、子类未重复声明"的语义差异。**已核验引擎侧**：引擎 `TFieldIterator` 中**不存在** `GetAllInterfaceClasses`（`grep` 于 `Runtime/CoreUObject/Public/UObject/UnrealType.h` 无命中），引擎直接取 `CurrentClass->Interfaces[InterfaceIndex]`（`UnrealType.h:6771-6774`，只遍历**当前类**声明的接口）—— 因此本文件与引擎在"接口字段遍历集合"上**确实不同**：本文件会把每个已实现接口的**基接口链**一并展开并去重。这属于有意增强，但文件内没有任何注释说明这一差异。
3. `(FProperty*)CurrentField` 这种 C 风格强制转换（`:98`）假设 `BaseFieldClass` 与 `FProperty` 的布局关系，是典型"引擎内部依赖"。

**建议**
- 在文件顶部加版本断言：不支持的引擎版本直接 `#error`，或对已知变化的版本区间写 `#if UE_VERSION_START(x,y,0)` 分支（与 `UEVersion.h` 一致的风格）；
- 补一条注释写明"本文件是 `TFieldIterator` 的拷贝，升级引擎时必须与 `Engine/Source/Runtime/CoreUObject/Public/UObject/UnrealType.h` 逐行比对"；
- 若引擎版本已支持接口迭代（`EFieldIterationFlags::IncludeInterfaces` 已存在，见 `:29` 的用法），评估直接删除本文件、改用引擎 `TFieldIterator` —— 这将同时消掉本文件 168 行的维护面（也是死代码的候选）。

**验证方式**
1. `grep -rn "TFieldIteratorExt" Source/` → 只有定义文件 + `FCSharpBind.cpp:8,239`；
2. 升级引擎时用同版本引擎的 `TFieldIterator::IterateToNext` 与本文件做 diff；
3. 用例：对 `AActor` 之类多接口类分别用 `TFieldIteratorExt<UFunction>` 与引擎 `TFieldIterator<UFunction>`（带 `IncludeInterfaces`）遍历，比对元素数量与顺序。

---

### [F-FL-018] `LIB_HOSTFXR` 无兜底分支；CoreCLR 的 host 库名在未知平台会变成未定义宏

- **类别**: 平台兼容
- **严重度**: **P3**
- **复核结论**: 确认
- **可达性**: 潜伏（本工程五平台全 LeanCLR，`WITH_CORECLR = 0`，`FCoreCLRFunctionLibrary.cpp` 整体不参与编译；切回 CoreCLR 即生效）
- **复核证据**: `Source/UnrealCSharpCore/Public/CoreMacro/Macro.h:85-91`（`#if PLATFORM_WINDOWS` / `#elif PLATFORM_LINUX` / `#elif PLATFORM_MAC_X86 || PLATFORM_MAC_ARM64`，`:91` 直接 `#endif`，**确无 `#else`/`#error` 兜底**）、`:93`（`CORECLR_RUNTIME_CONFIG`）；使用点 `Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRFunctionLibrary.cpp:19-26`（`:24` `*LIB_HOSTFXR` 解引用一个可能未定义的宏）；该文件整体被 `:1 #if WITH_CORECLR` 包住（已 read 确认）；`Source/UnrealCSharpCore/UnrealCSharpCore.build.cs:275-284`（`bWithCoreCLR` → `WITH_CORECLR=1/0`，原文行号正确）
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharpCore/Public/CoreMacro/Macro.h:85-93`；使用点 `Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRFunctionLibrary.cpp:19-26`
- **函数**: 宏 `LIB_HOSTFXR` / `FUnrealCSharpFunctionLibrary` 无关 —— 属 CoreMacro 平台分支
- **置信度**: 高（宏定义形态确定；本工程实际平台为 Win64，其它平台未编译验证）

**现状（代码事实）**

```cpp
// CoreMacro/Macro.h:85-91
#if PLATFORM_WINDOWS
#define LIB_HOSTFXR FString(TEXT("hostfxr.dll"))
#elif PLATFORM_LINUX
#define LIB_HOSTFXR FString(TEXT("libhostfxr.so"))
#elif PLATFORM_MAC_X86 || PLATFORM_MAC_ARM64
#define LIB_HOSTFXR FString(TEXT("libhostfxr.dylib"))
#endif                                    // :91  ← 没有 #else / #error
#define CORECLR_RUNTIME_CONFIG FString(TEXT("CoreCLR.runtimeconfig.json"))   // :93
```
```cpp
// FCoreCLRFunctionLibrary.cpp:19-26
FString FCoreCLRFunctionLibrary::GetHostFxrPath()
{
	return FString::Printf(TEXT("%s/%s"), *GetCoreCLRDirectory(), *LIB_HOSTFXR);   // :24 未定义宏 → 编译报错
}
```

**调用上下文**
`FCoreCLRFunctionLibrary::GetHostFxrPath()` 被 CoreCLR 域初始化使用（`WITH_CORECLR`，本工程 Win64 上是 1，由 `UnrealCSharpCore.build.cs:275-284` 写入）。

**问题**
`#if` 链覆盖 `PLATFORM_WINDOWS`/`PLATFORM_LINUX`/`PLATFORM_MAC_*`；在其它已定义 `WITH_CORECLR` 的组合下（例如 Windows ARM64 之外的平台、或未来新增平台、或 Mac 上 `PLATFORM_MAC_X86/ARM64` 未按预期定义时）`LIB_HOSTFXR` 就是**完全未定义**的标识符：`*LIB_HOSTFXR` 会变成"解引用一个未知类型名"的奇异诊断，而不是清晰的"平台不支持"。`CoreCLRMacro.h:5-7` 的 `CORECLR_TYPE_NAME` 与 `MonoMacro.h:3-9` 的三个名字同样只在 `WITH_*` 为真时定义（好在 `build.cs:275-306` 保证 `WITH_MONO/WITH_CORECLR/WITH_LEANCLR` 永远被定义为 0 或 1，所以 `#if WITH_X` 不会遇到未定义宏）。

**建议**
```cpp
#else
#error "UnrealCSharp: hostfxr library name is unknown for this platform"
#endif
```
并对 `MonoMacro.h` / `CoreCLRMacro.h` 里条件定义的宏做同样处理（或加 `#if !defined(...)` 的兜底 + 静态提示）。

**验证方式**
`grep -rn "LIB_HOSTFXR" Source/` → 只有 `Macro.h:86,88,90` 与 `FCoreCLRFunctionLibrary.cpp:24`；尝试为一个未覆盖平台配置 `WindowsScriptDomainType=CoreCLR` 并让 `PLATFORM_MAC_ARM64` 不成立，观察编译诊断质量。

---

### P3

> **排序说明**：本节 9 条（F-FL-019…F-FL-027）全为 P3。另有 3 条 P3 条目（F-FL-010、F-FL-012、F-FL-018）按编号锚点保留在上一节，未移动。

### [F-FL-019] `TIsTOptional` 依赖引擎内部符号 `TIsTOptional_V`，并与引擎已有同名 trait 重复

- **类别**: 可维护性 / 版本兼容
- **严重度**: **P3**
- **复核结论**: 确认（**此前的降级已撤销**：`TIsTOptional_V` 经引擎源码核实**确实存在**，原"该符号不在本仓库、其公开性/等价语义无法核验"的降级理由无效）
- **可达性**: 活跃（`UE_F_OPTIONAL_PROPERTY` 在 UE 5.6 为真，故 `TIsTOptional` 被定义并被 18 处消费点实例化，五平台均参与编译）
- **复核证据**: `Source/UnrealCSharpCore/Public/Template/TIsTOptional.inl:5-11`（`:5 #if UE_F_OPTIONAL_PROPERTY`、`:7 struct TIsTOptional`、`:9 enum { Value = TIsTOptional_V<T> };`）。**引擎侧已实测**：`Engine\Source\Runtime\Core\Public\Misc\Optional.h:440`（主模板 `TIsTOptional_V = false`）+ `:441-444`（**4 个 cv 限定特化**：`TOptional<T>`、`const TOptional<T>`、`volatile TOptional<T>`、`const volatile TOptional<T>`），并带注释 `Traits which determines whether or not a type is a TOptional.`（`:437-439`）。`grep "TIsTOptional_V" Source/` → **仅 1 处命中**（`:9` 自身），与原文一致。**子命题 (b) 修正**：`grep "struct TIsTOptional\b"` 于整个 `Engine/Source/**/*.h` **无命中** → 引擎**并未**提供同名 `struct TIsTOptional`，故"重复定义/ADL 歧义"仅是**前瞻性**风险（未来引擎版本可能引入），不是现存冲突。使用点计数修正为 **18 处**（原文"19 处"含 `:7` 的 trait 自身定义）：`TNameSpace.inl:268`、`TName.inl:308`、`TIsRef.inl:29`、`TGeneric.inl:165`、`TDefaultArgument.inl:474`、`TPropertyBuilder.inl:325,526`、`TReturnValue.inl:254`、`TArgument.inl:384`、`TPropertyValue.inl:926`、`TPropertyClass.inl:339` 等
- **级别变动**: 撤销降级 → **P3 维持为最终级别**（trait 真实存在、代码可用，故不构成功能缺陷；剩余价值仅为"依赖引擎 `*_V` 内部命名约定 + 无版本守卫注释"的可维护性提示）
- **文件**: `Source/UnrealCSharpCore/Public/Template/TIsTOptional.inl:5-11`（`:9`）
- **函数**: `TIsTOptional<T>`
- **置信度**: **高**（`TIsTOptional_V` 已于 `Engine\Source\Runtime\Core\Public\Misc\Optional.h:437-444` 实证存在：`Traits which determines whether or not a type is a TOptional.` 注释 + 主模板 + 4 个 cv 限定特化；`grep "TIsTOptional_V" Source/` 仍为 1 处命中，确认插件侧未自定义该符号）

**现状（代码事实）**

```cpp
// TIsTOptional.inl:5-11
#if UE_F_OPTIONAL_PROPERTY                      // UEVersion.h:96  → UE_VERSION_START(5,3,0)
template <typename T>
struct TIsTOptional
{
	enum { Value = TIsTOptional_V<T> };          // :9 引擎内部变量模板，本仓库无定义
};
#endif
```
`grep -rn "TIsTOptional_V" Source/` → **仅 1 处命中**（即 `:9` 自身）。

**问题**
该 trait 不自己实现判定，而是转发给引擎的 `TIsTOptional_V`。这比复制实现要好，但缺点是：(a) `*_V` 变量模板是引擎**内部**命名约定，不属公开 API，版本升级可能移除或改语义；(b) 引擎若已提供 `TIsTOptional`，本文件的同名 trait 就是重复定义（当前靠"插件不定义引擎已定义的东西"来回避冲突，一旦引擎在某个版本加上同名 trait，就会与本文件冲突/被 ADL 歧义）。19 处使用（`TPropertyClass.inl:18,339`、`TPropertyValue.inl:20,926`、`TArgument.inl:384` 等）都依赖它。

**建议**
把引擎调用集中到一个适配点并写清依赖来源与最低版本：
```cpp
// UEVersion.h 内统一：
#if UE_VERSION_START(5,3,0)
	#define UECSHARP_HAS_T_IS_T_OPTIONAL_V 1
#endif
```
并加注释 "依赖 Engine/Source/Runtime/Core/Public/Templates/Optional.h 的 TIsTOptional_V；引擎升级时复核"。若引擎已在目标版本公开 `TIsTOptional`，直接删除本文件并改 `using` 别名。

**验证方式**
`grep -rn "TIsTOptional_V" Source/` 只有 1 处；升级引擎时在此处编译即可暴露。

---

### [F-FL-020] `TGetUtf8String`：缓冲区 ABI 约定写成 `Size-1`/`64KB` 上限且超限静默返回空串；文件名与符号大小写不一致

- **类别**: 可读性 / 隐患
- **严重度**: **P3**
- **复核结论**: 确认（全部行号逐条命中）
- **可达性**: 活跃（`FScriptDomainImpl.inl` 内 3 处调用在三大后端共用路径上）
- **复核证据**: `Source/UnrealCSharpCore/Public/Template/TGetUtf8String.inl:6`（`static FString TGetUTF8String(T&& InFunction)` —— 文件名 `TGetUtf8String.inl` 与符号大小写确不一致）、`:8`（`512`）、`:10`（`64 * 1024`）、`:14`（回调写入+返回长度）、`:21`（`Length < StackBufferSize - 1`）、`:32`（`BufferSize = StackBufferSize * 2; BufferSize <= MaxBufferSize; BufferSize *= 2` → 1024…65536）、`:43`（`Length < BufferSize - 1`）、`:53`（超限 `return {}` **无任何日志**）—— **8 个引用行号全部命中**，与原文一致
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharpCore/Public/Template/TGetUtf8String.inl:5-54`（`:8,:10,:14,:21,:32,:43,:53`）；文件名 `TGetUtf8String.inl` vs 符号 `TGetUTF8String`
- **函数**: `TGetUTF8String(T&& InFunction)`
- **置信度**: 高（代码事实确定；回调约定属"未书面化的 ABI"，见"问题"）

**现状（代码事实）**

```cpp
// TGetUtf8String.inl:6-53
template <typename T> static FString TGetUTF8String(T&& InFunction)
{
	constexpr int32 StackBufferSize = 512;                 // :8
	constexpr int32 MaxBufferSize = 64 * 1024;              // :10
	uint8 StackString[StackBufferSize];
	auto Length = InFunction(StackString, StackBufferSize); // :14 回调约定：写入 + 返回长度
	if (Length <= 0) { return {}; }                        // :16-19
	if (Length < StackBufferSize - 1) { ...构造 FString... } // :21 用 Size-1 判断"装得下"
	TArray<uint8> HeapString;
	for (auto BufferSize = StackBufferSize * 2; BufferSize <= MaxBufferSize; BufferSize *= 2)  // :32 1024→65536
	{ ... if (Length < BufferSize - 1) { ...构造... } }
	return {};                                             // :53 超过 64KB 静默返回空串
}
```
使用点：`Source/UnrealCSharpCore/Public/Domain/Script/FScriptDomainImpl.inl`（`:4` 为 `#include`，实际调用在 `:30,:43,:56`，共 3 处）。

**问题**
1. 回调 `InFunction` 的契约没有注释：返回值到底是"写入的字节数"还是"所需缓冲区大小"？两种约定下 `Length < Size - 1` 的判断含义不同（前者偏保守、后者正确）。这是跨 C++/C# 的隐式 ABI，写在模板里却没有任何文档。
2. 超过 64 KB 时 `return {}`，与"编码失败"无法区分 —— 调用方拿到空 `FString`，无法判断是"空字符串"还是"太长了"。异常栈/日志这类可能超长（>64KB）的输出正好会命中该分支，且**没有任何日志**。
3. 文件名 `TGetUtf8String.inl` 与符号 `TGetUTF8String` 大小写不一致（同目录其它文件为 `TGetArrayLength.inl`→`TGetArrayLength`），不利于检索。

**建议**
1. 在文件头补一段契约注释，并在检测到 `Length >= MaxBufferSize` 时 `UE_LOG(Warning)` 而不是静默返回空；若确实需要截断，提供 `TGetUTF8String(InFunction, MaxLength)`；
2. 把 `StackBufferSize - 1` 的含义显式化（例如 `constexpr int32 NeedNulTerminator = 1;`）；
3. 统一命名（`TGetUtf8String.inl` ↔ `TGetUtf8String`，或全部改成 `UTF8`）。

**验证方式**
`grep -rn "TGetUTF8String" Source/` → 5 处命中（1 定义 + 1 `#include` + 3 调用）；构造一份 >64KB 的异常输出观察返回值与日志。

---

### [F-FL-021] CoreMacro 宏卫生：`Macro.h` 非自包含头、`NAMESPACE_ROOT == NAMESPACE_SCRIPT`、`STR/TEXT_STR` 通用名污染、`BufferMacro` 用参数名当宏名、`CompilerMacro` 空分支

- **类别**: 可读性 / 一致性
- **严重度**: **P3**
- **复核结论**: 确认（5 个子项全部逐一核实）
- **可达性**: 活跃
- **复核证据**: `Source/UnrealCSharpCore/Public/CoreMacro/Macro.h:1`（仅 `#pragma once`）、`:3`（`PLUGIN_NAME`）—— **全文件 133 行内确实没有任何 `#include`**，而 `:15`（`DYNAMIC_CLASS_DEFAULT_NAMESPACE` 用 `UObject::StaticClass()`）、`:119-125`（`ACTOR_PREFIX`/`OBJECT_PREFIX`/`INTERFACE_PREFIX`/`STRUCT_PREFIX` 用 `AActor`/`UObject`/`UInterface`/`TBaseStructure<FVector>`）都是运行时表达式 → 非自包含头成立；`:127-133`（`PLACEHOLDER _`、`STR`、`TEXT_STR`、`F_STRING_STR`）；`Source/UnrealCSharpCore/Public/CoreMacro/NamespaceMacro.h:5` 与 `:7`（**字面量同为 `TEXT("Script")`**）、`:9-17`（其余 5 个常量确无此重复）；`Source/UnrealCSharpCore/Public/CoreMacro/BufferMacro.h:5-29`（`IN_BUFFER`/`OUT_BUFFER`/`RETURN_BUFFER` 各有 `*_TEXT`，而 `:23` `IN_KEY_BUFFER`、`:27` `IN_VALUE_BUFFER` **无 `*_TEXT`**，仅 `:25`/`:29` 的 `*_SIGNATURE`）；`Source/UnrealCSharpCore/Public/CoreMacro/CompilerMacro.h:3`（`#if defined(_MSC_VER)` 空体）、`:4`（`#elif defined(__clang__)` → clang-cl 同时定义两者时永不进入）、`:15-21`（`#ifndef` 兜底空定义）
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharpCore/Public/CoreMacro/Macro.h:1,15,117,119-125,127-133`、`NamespaceMacro.h:5-7`、`BufferMacro.h:1-29`、`CompilerMacro.h:3-21`
- **函数**: 宏（无函数）
- **置信度**: 高

**现状（代码事实）**

```cpp
// Macro.h:1-3 —— 文件内**没有任何 #include**，却在宏体里使用引擎类型
#pragma once
#define PLUGIN_NAME FString(TEXT("UnrealCSharp"))
// Macro.h:15 / :119-125 —— 宏体展开为运行时表达式，依赖调用点已包含 CoreMinimal/UObject
#define DYNAMIC_CLASS_DEFAULT_NAMESPACE UObject::StaticClass()->GetPackage()->GetName().RightChop(1).Replace(TEXT("/"), TEXT("."))
#define ACTOR_PREFIX AActor::StaticClass()->GetPrefixCPP()
#define STRUCT_PREFIX TBaseStructure<FVector>::Get()->GetPrefixCPP()
// Macro.h:127-133
#define PLACEHOLDER _
#define STR(Str) #Str
#define TEXT_STR(Str) TEXT(STR(Str))
#define F_STRING_STR(Str) FString(TEXT_STR(Str))
// NamespaceMacro.h:5-7
#define NAMESPACE_ROOT FString(TEXT("Script"))
#define NAMESPACE_SCRIPT FString(TEXT("Script"))     // :7 与 NAMESPACE_ROOT 字面量相同（本文件内 grep 命中 2 处，正文见"证据"）
// BufferMacro.h:5-29
#define IN_BUFFER InBuffer
#define OUT_BUFFER OutBuffer
#define RETURN_BUFFER ReturnBuffer
#define IN_KEY_BUFFER InKeyBuffer          // 无 *_TEXT 变体（对比 IN/OUT/RETURN 都有）
#define IN_VALUE_BUFFER InValueBuffer      // 同上
// CompilerMacro.h:3-4
#if defined(_MSC_VER)
#elif defined(__clang__)                   // :4 首个分支为空体，MSVC/clang-cl 落到 :15-21 的 #ifndef 兜底
```

**问题**
1. `Macro.h` 不是自包含头：任何先包含它、后包含 `CoreMinimal.h` 的 TU 会在 `UObject::StaticClass()`（`:15,121`）、`TBaseStructure<FVector>`（`:125`）处报错。头文件应当 `#include "CoreMinimal.h"`，或把这些"运行时表达式宏"降级为函数。
2. `NAMESPACE_ROOT`（`:5`）与 `NAMESPACE_SCRIPT`（`NamespaceMacro.h:7`）字面量完全相同（都是 `"Script"`），却承担两种语义（C# 根命名空间 vs UE 模块命名空间），改一处漏一处；`NamespaceMacro.h:9-17` 的其它常量无此重复。
3. `STR`/`TEXT_STR`/`PLACEHOLDER`/`DYNAMIC`/`INTEROP_NAME` 这类**无前缀通用名宏**进入全局宏命名空间。`STR` 尤其危险（大量第三方/引擎代码可能定义或使用同名标识符）；`#define PLACEHOLDER _`（`:127`）把标识符 `PLACEHOLDER` 改写成 `_`，在结构化绑定等场景会静默改变代码含义（本文件 `:1323,:1335,:1368,:1380` 的 `for (const auto& [Key, PLACEHOLDER] : ...)` 正是依赖它）。
4. `BufferMacro.h` 把**参数名**定义成宏（`IN_BUFFER`→`InBuffer`）。虽然展开是同一标识符的不同大小写，实际风险是：任何写了全大写 `IN_BUFFER` 的代码会被静默改写；反向地，若 `InBuffer` 与宏名只差大小写，宏不生效 —— 这种"看起来是类型/标记、实际是参数名"的宏很难静态检索。
5. `CompilerMacro.h:3-4` 的空 `#if` 首分支让 MSVC 与 clang-cl 走 `:15-21` 的兜底空定义（行为正确），但阅读上像"漏写"，且 `#elif defined(__clang__)` 在 clang-cl（同时定义 `_MSC_VER` 与 `__clang__`）下永不生效 —— 即 **clang-cl 下拿不到 `-Wdangling` 抑制**，这是实际可观察的行为差异。

**建议**
- `Macro.h` 顶部补 `#include "CoreMinimal.h"`；把 `DYNAMIC_CLASS_DEFAULT_NAMESPACE`/`ACTOR_PREFIX`/`STRUCT_PREFIX` 改成 `inline` 函数，避免宏体在任意包含顺序下解析；
- 重命名通用名宏：`UNREALCSHARP_STR`/`UNREALCSHARP_TEXT_STR`、`UNREALCSHARP_PLACEHOLDER`；`PLACEHOLDER` 在结构化绑定里改成 `Unused`；
- `NAMESPACE_ROOT` 与 `NAMESPACE_SCRIPT` 合并为一个常量（或让其一引用另一个）；
- `BufferMacro` 保留 `*_SIGNATURE`（类型+名字）但把纯 `IN_BUFFER`/`OUT_BUFFER` 换成 `inline constexpr const TCHAR*` 常量或 `constexpr` 函数；
- `CompilerMacro.h` 改为显式条件：
```cpp
#if defined(__clang__) && !defined(_MSC_VER)
	... clang 专用 ...
#else
	... 空 ...
#endif
```
并注释说明为何 clang-cl 不需要该抑制。

**验证方式**
1. `grep -rn "define STR\b\|define TEXT_STR\|define PLACEHOLDER" Source/` 确认污染面；
2. 新建一个只 `#include "CoreMacro/Macro.h"` 的 TU 编译，应复现"未知类型 UObject"错误（证明非自包含）；
3. `grep -rn "NAMESPACE_ROOT\|NAMESPACE_SCRIPT" Source/` 对比两常量在调用点是否可互换。

---

### [F-FL-022] `TIsNotUEnum` 的名字与语义相反

- **类别**: 可读性（命名）
- **严重度**: **P3**
- **复核结论**: 确认（含 `grep` 命中数重跑）
- **可达性**: 活跃
- **复核证据**: `Source/UnrealCSharpCore/Public/Template/TIsNotUEnum.inl:3-7`（主模板 `enum { Value = false };`，名实相反成立）；打开点 `Source/UnrealCSharp/Public/Macro/BindingMacro.h:230-234`（`:231 struct TIsNotUEnum<Class>`、`:233 enum { Value = true };`，位于 `BINDING_ENUM` 展开体内）；使用点 `Source/UnrealCSharp/Public/Binding/Core/TPropertyClass.inl:317`（`TIsEnum<std::decay_t<T>>::Value && !TIsNotUEnum<std::decay_t<T>>::Value`）。**`grep "TIsNotUEnum"` 全 `Source/` 命中 17 处** —— 与原文"17 处命中"**完全一致**（1 定义 + 1 include + 1 打开点 + 14 处 `enable_if` 使用：`TPropertyBuilder.inl:296,501`、`TNameSpace.inl:244`、`TName.inl:286`、`TReturnValue.inl:232`、`TArgument.inl:362`、`TDefaultArgument.inl:438`、`TPropertyValue.inl:878`、`TPropertyClass.inl:317` 等）
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharpCore/Public/Template/TIsNotUEnum.inl:3-7`；打开点 `Source/UnrealCSharp/Public/Macro/BindingMacro.h:230-234`
- **函数**: `TIsNotUEnum<T>`
- **置信度**: 高

**现状（代码事实）**

```cpp
// TIsNotUEnum.inl:3-7
template <typename T> struct TIsNotUEnum { enum { Value = false }; };     // 主模板恒 false
// BindingMacro.h:230-234（在 BINDING_ENUM 展开体中，即"用 BINDING_ENUM 注册的 C++ 枚举"）
template <> struct TIsNotUEnum<Class> { enum { Value = true }; };
// 使用点 Source/UnrealCSharp/Public/Binding/Core/TPropertyClass.inl:317
struct TPropertyClass<T, std::enable_if_t<TIsEnum<std::decay_t<T>>::Value && !TIsNotUEnum<std::decay_t<T>>::Value, T>>
```

**问题**
按字面意思 `TIsNotUEnum<int>::Value` 应当是 `true`（`int` 确实不是 `UEnum`），但主模板恒为 `false`；它实际表达的是"该类型是**由 `BINDING_ENUM` 手工注册的普通枚举**"，与"是不是 UEnum"无关。`TPropertyClass.inl:317` 的 `TIsEnum<...> && !TIsNotUEnum<...>` 双重否定进一步增加了误读概率（读起来像"是枚举 且 是 UEnum"，实际是"是枚举 且 不是已注册的普通枚举"）。

**建议**
改名为 `TIsRegisteredNonUEnum`（或 `TIsBindingEnum`），并在文件头写明"主模板 false 表示'未按 BINDING_ENUM 注册'"；使用点写成 `TIsEnum<T> && !TIsBindingEnum<T>`。

**验证方式**
`grep -rn "TIsNotUEnum" Source/` → 17 处命中，逐处确认语义都为"是否已注册的普通枚举"。

---

### [F-FL-023] `GetDotnetVersion()` 返回 `Latest - 1`，依赖枚举隐式自增的魔法语义

- **类别**: 可读性 / 隐患
- **严重度**: **P3**
- **复核结论**: 确认（代码事实与使用链全部核实；与 `07-…/01` 的 TargetFramework↔`DotnetVersion.h` 对照表交叉核对**未完成**，见第 6 节）
- **可达性**: 活跃（每次生成解决方案都写 `<TargetFramework>`）
- **复核证据**: `Source/UnrealCSharpCore/Public/Setting/UnrealCSharpSetting.h:65-72`（`UENUM()` + `enum EDotnetVersion { V8 = 8, V9 = 9, V10 = 10, Latest };` —— `:71` 的 `Latest` 隐式 = **11**）；`Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1397`（`int32 DotnetVersion = EDotnetVersion::Latest;`）、`:1404`（`DotnetVersion == EDotnetVersion::Latest ? DotnetVersion - 1 : DotnetVersion` → 11→10）；消费点 `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:258-267`（`:264 GetDotnetVersion()`、`:265 DOTNET_MINOR_VERSION`，格式串 `net%d.%d`）
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1395-1405`（`:1404`）；枚举定义 `Source/UnrealCSharpCore/Public/Setting/UnrealCSharpSetting.h:65-72`
- **函数**: `FUnrealCSharpFunctionLibrary::GetDotnetVersion()`
- **置信度**: 高

**现状（代码事实）**

```cpp
// UnrealCSharpSetting.h:65-72
UENUM() enum EDotnetVersion { V8 = 8, V9 = 9, V10 = 10, Latest };        // Latest 隐式 = 11
// FUnrealCSharpFunctionLibrary.cpp:1395-1405
int32 FUnrealCSharpFunctionLibrary::GetDotnetVersion()
{
	int32 DotnetVersion = EDotnetVersion::Latest;                        // :1397 = 11
	if (const auto UnrealCSharpSetting = GetMutableDefaultSafe<UUnrealCSharpSetting>())
	{ DotnetVersion = UnrealCSharpSetting->GetDotnetVersion(); }
	return DotnetVersion == EDotnetVersion::Latest ? DotnetVersion - 1 : DotnetVersion;   // :1404 Latest(11)→10
}
```
使用点：`Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:258-267` → `FString::Printf(TEXT("<TargetFramework>net%d.%d</TargetFramework>"), GetDotnetVersion(), DOTNET_MINOR_VERSION)` → `net10.0`。

**问题**
`Latest` 是"最高版本 + 1"的哨兵值而不是版本号；转换逻辑散落在使用点而不是定义处。任何直接使用 `EDotnetVersion::Latest` 的地方都会得到 `11`（一个不存在的 dotnet 版本，会生成 `net11.0`），任何在 `V10` 与 `Latest` 之间插入新版本的人都要重新推导 `-1` 是否仍然正确。此外 `UUnrealCSharpSetting::GetDotnetVersion()`（`UnrealCSharpSetting.cpp:155-158`）返回的是同一个含哨兵的原始值，两个同名函数语义不同（一个含哨兵、一个不含），极易误用。

**建议**
```cpp
UENUM() enum class EDotnetVersion : uint8 { V8 = 8, V9 = 9, V10 = 10, Latest = V10 };
```
即让 `Latest` 直接用最高版本的枚举值（`= V10`），删除 `-1`；并在 `GetDotnetVersion()` 上写一行 `/** 返回实际可用的最高 dotnet 主版本号，不含哨兵 */`。同时把 `UUnrealCSharpSetting::GetDotnetVersion()` 改名 `GetRawDotnetVersionSetting()`。

**验证方式**
`grep -rn "EDotnetVersion::Latest\|GetDotnetVersion" Source/` → 3 处命中，确认无其它地方直接使用哨兵值。

---

### [F-FL-024] `GetFullClass` 的 6 条件布尔表达式重复两遍；`GetFullInterface`/`Encode` 重载缺少同族函数都有的 null 检查

- **类别**: 可读性 / 一致性 / 隐患
- **严重度**: **P3**
- **复核结论**: 确认（三个子项全部核实：重复表达式、缺 null 检查、重复求值）
- **可达性**: 活跃
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:394-408`（同一个 6 条件表达式**完整写了两遍**：`:397-402` 决定是否取 `GetPrefixCPP()`，`:404-407` 作为 `bIsNative` 实参 —— 逐字一致）；对照 `:389-392`（`GetFullClass` 有 `if (InStruct == nullptr) return TEXT("");`）、`:425-428`（`GetClassNameSpace` 有同样判空）；`:411-421`（`GetFullInterface` **无任何判空**，且 `:416-418` 在同一三元表达式内把 `GetFullClass(InStruct)` 求值两次、`:419` 又求值一次 `IsInBlueprint()`）；`:1295-1298` 与 `:1300-1303`（两个 `Encode` 重载直接 `InProperty->GetName()` / `InFunction->GetName()`，无判空）
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:387-421`（`:397-399` vs `:404-407`、`:411-421`）、`:1295-1303`
- **函数**: `GetFullClass(const UStruct*)`、`GetFullInterface(const UStruct*)`、`Encode(const FProperty*)`、`Encode(const UFunction*)`
- **置信度**: 高

**现状（代码事实）**

```cpp
// :394-408 同一个条件被完整写了两遍（一处决定是否加 GetPrefixCPP，一处决定 bIsNative 实参）
return Encode(FString::Printf(TEXT("%s%s"),
	           (InStruct->IsNative() || FDynamicClassGenerator::IsDynamicClass(InStruct) ||
	            FDynamicStructGenerator::IsDynamicStruct(InStruct)) &&
	           !FDynamicClassGenerator::IsDynamicBlueprintGeneratedClass(InStruct) ? InStruct->GetPrefixCPP() : TEXT(""),
	           *InStruct->GetName()),
	           (InStruct->IsNative() || FDynamicClassGenerator::IsDynamicClass(InStruct) ||      // :404-407 完全重复
	            FDynamicStructGenerator::IsDynamicStruct(InStruct)) &&
	           !FDynamicClassGenerator::IsDynamicBlueprintGeneratedClass(InStruct));
```
```cpp
// :411-421 —— 没有 InStruct == nullptr 检查（同文件的 GetFullClass:389、GetClassNameSpace:425 都有；GetFullInterface 只判了一次却传给了两个函数）
FString FUnrealCSharpFunctionLibrary::GetFullInterface(const UStruct* InStruct)
{
	return Encode(FString::Printf(TEXT("I%s"),
	              InStruct->IsInBlueprint() ? *GetFullClass(InStruct) : *GetFullClass(InStruct).RightChop(1)),  // :416-418 同一表达式算两遍
	              InStruct->IsInBlueprint());                                                                  // :419
}
// :1295-1303 —— 无 null 检查
FString FUnrealCSharpFunctionLibrary::Encode(const FProperty* InProperty)  { return Encode(InProperty->GetName(), InProperty->IsNative()); }  // :1297
FString FUnrealCSharpFunctionLibrary::Encode(const UFunction* InFunction)  { return Encode(InFunction->GetName(), InFunction->IsNative()); }
```

**问题**
1. `:397-407` 的 6 条件表达式重复，且两处必须保持一致（一处改了另一处忘改 → C# 类型名与"是否 native"判断不一致，属于会生成错误绑定代码的静默 bug）。
2. `GetFullInterface` 缺 null 检查（`GetFullClass`/`GetClassNameSpace` 都有），且 `InStruct->IsInBlueprint()` 与 `GetFullClass(InStruct)` 被重复求值两次（`:416-418` 在同一三元表达式里调用 `GetFullClass` 两次，每次都做整套动态类判定）。
3. `Encode(FProperty*)`/`Encode(UFunction*)` 直接解引用。现有调用点（`FClassGenerator.cpp:1317+`、`FCSharpBind.cpp:155/194/206/253`）传的都是非空指针，因此是接口防御缺口而非现存崩溃。

**建议**
```cpp
FString FUnrealCSharpFunctionLibrary::GetFullClass(const UStruct* InStruct)
{
	if (InStruct == nullptr) { return TEXT(""); }
	const bool bNativeLike =
		(InStruct->IsNative() || FDynamicClassGenerator::IsDynamicClass(InStruct) ||
		 FDynamicStructGenerator::IsDynamicStruct(InStruct)) &&
		!FDynamicClassGenerator::IsDynamicBlueprintGeneratedClass(InStruct);
	return Encode(FString::Printf(TEXT("%s%s"), bNativeLike ? InStruct->GetPrefixCPP() : TEXT(""), *InStruct->GetName()), bNativeLike);
}
FString FUnrealCSharpFunctionLibrary::GetFullInterface(const UStruct* InStruct)
{
	if (InStruct == nullptr) { return TEXT(""); }
	const FString FullClass = GetFullClass(InStruct);
	const bool bInBlueprint = InStruct->IsInBlueprint();
	return Encode(FString::Printf(TEXT("I%s"), bInBlueprint ? *FullClass : *FullClass.RightChop(1)), bInBlueprint);
}
// Encode 重载加 if (InProperty == nullptr) { return FString(); }
```

**验证方式**
`grep -n "IsDynamicBlueprintGeneratedClass" Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp` → 确认只有 `:400,:407` 两处（需同步修改的风险点）。

---

### [F-FL-025] `GetAssetClass` 的函数内 `static TMap` 既是可变全局又做双重查找

- **类别**: 可优化 / 隐患
- **严重度**: **P3**
- **复核结论**: 确认（调用点亦已核实）
- **可达性**: 活跃（`FAssetGenerator.cpp:183` 为资产生成主路径，每个 Blueprint/资产都走）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:693`（函数内 `static TMap<UClass*, FString> AssetClass = {...}`，仅 1 个 `UAnimBlueprint` 条目）、`:700`（`AssetClass.Contains(...) ? AssetClass[...] : InClass` —— **`Contains` + `operator[]` 两次哈希**）；调用点 `Source/ScriptCodeGenerator/Private/FAssetGenerator.cpp:183-184`（`FUnrealCSharpFunctionLibrary::GetAssetClass(InAssetData, ...GetFullClass(SuperClass))`，已 read 178-189 确认）
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:691-701`（`:693-700`）
- **函数**: `FUnrealCSharpFunctionLibrary::GetAssetClass(const FAssetData&, const FString&)`
- **置信度**: 高

**现状（代码事实）**

```cpp
// :691-701
FString FUnrealCSharpFunctionLibrary::GetAssetClass(const FAssetData& InAssetData, const FString& InClass)
{
	static TMap<UClass*, FString> AssetClass = {                       // :693 函数内可变静态
		{ UAnimBlueprint::StaticClass(), UAnimInstance::StaticClass()->GetPrefixCPP() + UAnimInstance::StaticClass()->GetName() }
	};
	return AssetClass.Contains(InAssetData.GetClass()) ? AssetClass[InAssetData.GetClass()] : InClass;   // :700 Contains + operator[] 两次哈希
}
```
调用点：`Source/ScriptCodeGenerator/Private/FAssetGenerator.cpp:183`。

**问题**
(a) `Contains` + `operator[]` 是两次查找（应 `const FString* Found = Map.Find(Key); return Found ? *Found : InClass;`）；(b) 表内容是"仅 AnimBlueprint 一项"的静态映射，表本身是实体类型（`UClass*` 键）且在函数内可变，首次调用时才做动态初始化 —— 若首次调用发生在生成期且多线程，会竞争（当前调用点都在游戏线程）；(c) 表只有一项，条件判断比查表更直接。

**建议**
```cpp
if (InAssetData.GetClass() && InAssetData.GetClass()->IsChildOf(UAnimBlueprint::StaticClass()))
{
	return UAnimInstance::StaticClass()->GetPrefixCPP() + UAnimInstance::StaticClass()->GetName();
}
return InClass;
```
或保留 TMap 但用 `Find` 一次查找 + `static const`。

**验证方式**
`grep -rn "GetAssetClass" Source/` → 1 个调用点；行为等价性用一次 AnimBlueprint 资产生成验证。

---

### [F-FL-026] `SaveStringToFile` 在写盘成功前就置脏；`GetOldFileName` 的 `Find==INDEX_NONE` 静默退化

- **类别**: 可读性 / 隐患
- **严重度**: **P3**
- **复核结论**: 确认（两个子项的行号逐个命中）
- **可达性**: 活跃
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1154`（`FileExists` 前置）、`:1158`（`Result.Equals(InString, ESearchCase::CaseSensitive)` —— 内容相同则不写，有意优化成立；注意此处**显式**指定了 `CaseSensitive`）、`:1165-1167`（`#if WITH_EDITOR MarkScriptChanged();` **先于**写盘）、`:1177-1178`（`FFileHelper::SaveStringToFile(...)` 的返回值直接透传，与 `:1166` 的置脏无关联）；`:717-719`（`GetOldFileName` 用 `InOldObjectPath.Right(InOldObjectPath.Len() - InOldObjectPath.Find(TEXT(".")) - 1)` —— `.Find` 返回 `INDEX_NONE(-1)` 时退化为 `Right(Len())` 取整串，且无告警、无注释）
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:1150-1179`（`:1166,:1177`）、`:717-720`（`:719`）
- **函数**: `SaveStringToFile(const FString&, const FString&)`、`GetOldFileName(const FAssetData&, const FString&)`
- **置信度**: 高

**现状（代码事实）**

```cpp
// :1150-1179
	if (FileManager->FileExists(*InFileName))
		if (FString Result; FFileHelper::LoadFileToString(Result, *InFileName))
			if (Result.Equals(InString, ESearchCase::CaseSensitive)) { return true; }   // :1158-1161 内容相同则不写（有意优化）
#if WITH_EDITOR
	MarkScriptChanged();                                                                 // :1166 先置脏
#endif
	... CreateDirectoryTree ...
	return FFileHelper::SaveStringToFile(InString, *InFileName, FFileHelper::EEncodingOptions::ForceUTF8WithoutBOM, FileManager, FILEWRITE_None);  // :1177 返回值直接透传
// :717-720
FString FUnrealCSharpFunctionLibrary::GetOldFileName(const FAssetData& InAssetData, const FString& InOldObjectPath)
{
	return GetFileName(InAssetData, InOldObjectPath.Right(InOldObjectPath.Len() - InOldObjectPath.Find(TEXT(".")) - 1));  // :719 Find 失败(-1) → Right(Len()) = 整串
}
```

**问题**
1. `MarkScriptChanged()` 在写入之前调用，且不看 `:1177` 的返回布尔：写盘失败（磁盘满/只读/杀软占用）时脏标志已被置位，会触发一次"什么都没变"的重编译；反之若调用方依赖返回值回滚标志，`bScriptChanged` 已是 `true`（`IsScriptChanged` 被 `FEditorListener.cpp:523,546`、`UnrealCSharpEditor.cpp:264` 使用）。
2. `GetOldFileName`：`InOldObjectPath` 不含 `.` 时 `Find` 返回 `INDEX_NONE(-1)`，`Right(Len() - (-1) - 1) = Right(Len())` → 取回整串，静默产生错误文件名而非报错；`RightChop`/`Find` 语义没有注释，读者很难看出这里是"去掉 `包.` 前缀"。

**建议**
```cpp
// SaveStringToFile
const bool bSaved = FFileHelper::SaveStringToFile(...);
#if WITH_EDITOR
if (bSaved) { MarkScriptChanged(); }
else { UE_LOG(LogUnrealCSharp, Error, TEXT("Failed to write %s"), *InFileName); }
#endif
return bSaved;
// GetOldFileName
const int32 DotIndex = InOldObjectPath.Find(TEXT("."), ESearchCase::CaseSensitive, ESearchDir::FromEnd);
const FString ObjectName = DotIndex != INDEX_NONE ? InOldObjectPath.RightChop(DotIndex + 1) : InOldObjectPath;
```

**验证方式**
`grep -rn "IsScriptChanged\|MarkScriptChanged" Source/` 确认脏标志消费者；用例：把目标 `.cs` 文件设为只读后触发生成，观察是否仍触发一次空重编译。

---

### [F-FL-027] `GetInteropDirectory()` 与 `GetInteropPath()` 返回同一路径却命名不同（其一完全无人调用）

- **类别**: 死代码 / 可读性
- **严重度**: **P3**
- **复核结论**: 确认（含死代码 `grep` 重跑，命中数与原文完全一致）
- **可达性**: 活跃（"同语义双名字"的命名问题在活跃路径上；但 `GetInteropPath()` **本身不可达** —— 全库零调用点）
- **复核证据**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:726`（`GetInteropDirectory()` → `GetFullScriptDirectory() / INTEROP_NAME`）、`:731`（`GetInteropProjectPath()` → `GetInteropDirectory() / INTEROP_NAME + PROJECT_SUFFIX`）、`:1117`（`GetInteropPath()` → `GetFullScriptDirectory() / INTEROP_NAME`，**与 `:726` 逐字等价**）。**`grep "GetInteropPath"` 全 `Source/` 命中 2 处**：`Source/UnrealCSharpCore/Public/Common/FUnrealCSharpFunctionLibrary.h:212`（声明）+ `.cpp:1115`（定义）→ **零调用点，强死代码判定成立**，与原文一致。命名体系不一致的另一实例 `GetCodeAnalysisPath`（`:1100-1103`，返回目录却叫 `Path`）亦已确认
- **级别变动**: 无（P3 维持）
- **文件**: `Source/UnrealCSharpCore/Private/Common/FUnrealCSharpFunctionLibrary.cpp:723-733`（`:726`）、`:1115-1118`（`:1117`）；声明 `Source/UnrealCSharpCore/Public/Common/FUnrealCSharpFunctionLibrary.h:108,212`
- **函数**: `GetInteropDirectory()`、`GetInteropPath()`
- **置信度**: 高

**现状（代码事实）**

```cpp
// :723-733
FString FUnrealCSharpFunctionLibrary::GetInteropDirectory()  { return GetFullScriptDirectory() / INTEROP_NAME; }   // :726
FString FUnrealCSharpFunctionLibrary::GetInteropProjectPath(){ return GetInteropDirectory() / INTEROP_NAME + PROJECT_SUFFIX; }  // :731
// :1115-1118
FString FUnrealCSharpFunctionLibrary::GetInteropPath()       { return GetFullScriptDirectory() / INTEROP_NAME; }   // :1117 与 :726 完全等价
```

**调用上下文**
`GetInteropDirectory()` 仅被 `GetInteropProjectPath()`（`:731`，再被 `FCSharpCompilerRunnable.cpp:267,284`、`FSolutionGenerator.cpp:79` 使用）调用；`GetInteropPath()` 在整个 `Source/` 下**只有声明与定义两处命中**（见第 4 节），无人调用。

**问题**
同一语义两个名字（`...Directory` / `...Path`），且其中一个（`GetInteropPath`）是死代码。同族函数里 `GetCodeAnalysisPath`（`:1100-1103`）也是"目录"语义却叫 `Path`，命名体系不一致，容易让新调用点选错（选到死代码那份）。

**建议**
删除 `GetInteropPath()`，或保留一个并把另一个改为 `[[deprecated]]`/`UE_DEPRECATED`；统一后缀语义：目录一律 `*Directory`，文件一律 `*Path`（现有多处违反：`GetCodeAnalysisPath`/`GetSourceGeneratorPath`/`GetWeaversPath` 都是目录）。

**验证方式**
`grep -rn "GetInteropPath\|GetInteropDirectory" Source/` → 应为 2 + 3 处命中，确认前者无调用点。

---

## 4. 死代码清单

统计方法：`Get-ChildItem Source -Recurse -Include *.h,*.cpp,*.inl`（排除 `ThirdParty/Intermediate/Binaries`）→ `Select-String -Pattern "\b<符号>\b"` 统计命中数与文件。命中数含"声明 + 定义"，因此 **==2（仅声明+定义）为强死代码**；`>2` 但全部落在 `FUnrealCSharpFunctionLibrary.{h,cpp}` 自身的记为"仅库内使用（对外未使用）"。

> ** grep 重跑结果（工具 `grep`，`include=*.*`，路径 `Plugins/UnrealCSharp/Source`）**：`GetInteropPath` = **2** ✓、`TGetArrayLength` = **9** ✓、`ACCESS_PRIVATE_STATIC_PROPERTY` = **1** ✓、`ACCESS_PRIVATE_MEMBER_FUNCTION` = **1** ✓、`ACCESS_PRIVATE_CONST_MEMBER_FUNCTION` = **1** ✓、`ACCESS_PRIVATE_STATIC_FUNCTION` = **1** ✓、`ACCESS_PRIVATE_MEMBER_PROPERTY` = **4**（1 定义 `AccessPrivateMacro.h:6` + 3 使用点 `FDynamicClassGenerator.cpp:22`、`FCSharpEnvironment.cpp:16`、`FGameplayTagContainer`→`FGameplayTagGenerator.cpp:13`）✓ —— **本节全部命中数与原文逐项一致**，未发现虚报。

### 4.1 强死代码（无任何调用点）

| 符号 | 声明位置 | grep 命中数 | 判定 | 证据（命中位置） |
|---|---|---|---|---|
| `GetInteropPath` | `.../FUnrealCSharpFunctionLibrary.h:212` | 2 | **强死代码** | `.h:212`（声明）+ `.cpp:1117`（定义），无其它命中 |
| `TGetArrayLength` | `.../Template/TGetArrayLength.inl:4,10` | 9（2 定义 + 7 个 `#include`） | **强死代码（函数零调用）** | 除定义外 7 处命中全部是 `#include "Template/TGetArrayLength.inl"`：`FDomain.cpp:4`、`FCSharpBind.cpp:7`、`TPropertyValue.inl:9`、`TPropertyBuilder.inl:8`、`FMonoDomain.cpp:12`、`FDynamicEnumGenerator.cpp:4`、`FClassReflection.cpp:7`。全库无 `TGetArrayLength(x)` 调用表达式 |
| `ACCESS_PRIVATE_STATIC_PROPERTY` | `.../CoreMacro/AccessPrivateMacro.h:13` | 1 | **强死代码** | 仅宏定义自身 |
| `ACCESS_PRIVATE_MEMBER_FUNCTION` | `.../CoreMacro/AccessPrivateMacro.h:20` | 1 | **强死代码** | 仅宏定义自身 |
| `ACCESS_PRIVATE_CONST_MEMBER_FUNCTION` | `.../CoreMacro/AccessPrivateMacro.h:27` | 1 | **强死代码** | 仅宏定义自身 |
| `ACCESS_PRIVATE_STATIC_FUNCTION` | `.../CoreMacro/AccessPrivateMacro.h:34` | 1 | **强死代码** | 仅宏定义自身 |
| `Template/TIsTEnumAsByte.inl` 全体 | 该文件 | — | **条件死代码（UE ≥ 5.2 编译期整体消失）** | 文件体被 `#if UE_T_IS_T_ENUM_AS_BYTE`（`:5`）包住，而 `UEVersion.h:90` = `!UE_VERSION_START(5,2,0)` → 仅在 UE < 5.2 为真。消费点 `TPropertyClass.inl:326` 无同条件守卫，说明 ≥5.2 时用的是**引擎自己**的同名 trait，本文件形同无物（也即本文件只是 <5.2 的兼容副本）|
| `TGet_Args<Index>()`（`template <auto Index, typename... Args> auto TGet_Args() { return false; }`） | `Source/UnrealCSharp/Public/Macro/BindingMacro.h:35-39` | —（非我负责文件，仅登记） | 疑似死代码 | 与 `TGet_Args<Index>(Args&&...)` 重载成对，用于 `if constexpr (!TSizeof_Args(__VA_ARGS__))` 的另一分支；本报告未做完整引用统计 |

### 4.2 仅库内使用（对外零调用）—— 不算死代码，但对外无价值

| 符号 | 声明位置 | grep 命中数（总/库内） | 判定 | 证据 |
|---|---|---|---|---|
| `GetModuleRelativePathMetaData`（3 重载） | `.h:20,38,42` | 9 / 9 | 仅被同文件 `GetModuleRelativePath` 调用 | `.cpp:81,195,236` 调用；`:91,205,246` 定义；`.h:20,38,42` 声明 |
| `GetOuterRelativePath`（3 重载） | `.h:49,51,54` | 9 / 9 | 仅库内 | `.cpp:80,194,235`（调用）、`:276,283,286`（定义） |
| `GetOuterName`（4 重载） | `.h:57,59,62,64` | 11 / 11 | 仅库内 | `.cpp:123,283`（调用）、`:316,332,336,342,347`（定义） |
| `ProcessModuleRelativePathMetaData` | `.h:66` | 5 / 5 | 仅库内 | `.cpp:102,217,258`（调用）、`:372`（定义） |
| `ProcessModuleRelativePath` | `.h:44` | 5 / 5 | 仅库内 | `.cpp:85,199,240`（调用）、`:261`（定义） |
| `GetSuffixName` | `.h:92` | 4 / 4 | 仅库内 | `.cpp:677,687`（`GetAssetName`/`GetObjectPathName` 内） |
| `GetUEDirectory` | `.h:116` | 4 / 4 | 仅库内 | `.cpp:753,758`（`GetUEProxyDirectory`/`GetUEProjectPath` 内）|
| `GetPluginTemplateOverrideDirectory` | `.h:166` | 3 / 3 | 仅库内 | `.cpp:931` |
| `GetPluginTemplateDynamicDirectory` | `.h:170` | 3 / 3 | 仅库内 | `.cpp:946` |
| `GetFullUEPublishPath` | `.h:189` | 3 / 3 | 仅库内 | `.cpp:1068`（`GetFullAssemblyPublishPath` 内）|
| `GetFullGamePublishPath` | `.h:191` | 3 / 3 | 仅库内 | `.cpp:1069` |
| `GetFullCustomProjectsPublishPath` | `.h:193` | 3 / 3 | 仅库内 | `.cpp:1070` |
| `GetInteropDirectory` | `.h:108` | 3 / 3 | 仅库内 | `.cpp:731` |
| `GetCustomProjectsName` | `.h:136` | 3（库内 2 + 外部 1） | 近乎未用 | 外部仅 1 处 |
| `GetEngineModuleList` | `.h:243` | 3（库内 2 + 外部 1） | 近乎未用 | 外部仅 `UnrealCSharpSetting.cpp:329` |

> 说明：`FUnrealCSharpFunctionLibrary` 整体标注 `UNREALCSHARPCORE_API`（`.h:7`），上述 public static 成员都是**导出符号**，理论上可被插件外的代码调用。但本仓库内无任何外部消费者、`UnrealCSharp.uplugin` 也未声明对外 API 契约，因此按 `_CONVENTIONS.md` 第 3.6 节记为"仅库内使用（对外未使用）"，**不**标为强死代码。

### 4.3 声明/定义完整性核对（无死声明）

用脚本比对 `.h` 的 `static <type> Name(` 与 `.cpp` 的 `FUnrealCSharpFunctionLibrary::Name(`：**声明 83 个 / 定义 82 个**，差集仅 `GetMutableDefaultSafe`（`.h:265-269` 的模板，内联在头中，正常）。反向差集为空 —— **没有"声明了但未定义"的成员**。

---

## 5. 横向维度汇总

| 维度 | 检查结果 | 关联 Finding |
|---|---|---|
| **1. 空指针/越界/未检查返回值** | 6 处 JSON 空 `TSharedPtr` 解引用（P1）；`Function`/`FindPlugin` 两个未检查指针（P2）；3 处 `ParseIntoArray` 索引未校验（P2）；`GetFullInterface`/`Encode` 重载缺 null 检查（P3）。**已核验安全**：`GetMutableDefaultSafe`（`.h:266-269`）用 `GExitPurge` 短路；`GetModuleName`/`GetClassNameSpace`/`GetFullClass(UEnum*)` 等 12 个函数都有显式 null 检查；`GetName`/`GetPrefixCPP` 的调用点都先判过 outer | F-FL-002/005/006/007/024 |
| **2. 内存与资源泄漏** | 本文件无 `new`/`malloc`/`GCHandle`；`SyncProcess` 的管道与进程句柄有配对的 `ClosePipe`/`CloseProc`（`:1587-1591`），但存在"提前 return 无句柄"和"对无效句柄操作"的路径（F-FL-009）。`GetAssetClass`/`GetEngineModuleList`/`GetProjectModuleList`/`Encode` 的 `static` 容器是进程级常驻（非泄漏但永不释放，`Encode` 的 77 项 FString 含内联分配） | F-FL-009/025 |
| **3. 线程安全** | `ScriptDomainType` 静态可写（P2）；模块名列表的 `static TArray` 惰性填充无同步（**P2**，原文此处误写 P1/P2）；`SyncProcess` 在两个不同线程上被同步调用（游戏线程 + 编译线程），`FPlatformProcess::ReadPipe`/进程句柄只在单一调用栈内使用，本身无共享。**已核验**：本文件**没有**任何 `check(IsInGameThread())` 断言，是面向上层信任的"裸"工具库 | F-FL-003/009/013 |
| **4. 异常与错误处理** | `ensure`/`check` 在本文件出现 0 次（`grep "check(\|ensure"` 于本文件无命中）→ Shipping 不会因本文件的断言崩溃，但也没有任何防御；`Deserialize` 返回值被丢弃（P1）；`SaveStringToFile` 置脏先于写盘（P3）；`SyncProcess` 的进程启动失败无日志（P2） | F-FL-002/009/026 |
| **5. 性能** | 见 F-FL-003（空表重解析 271KB JSON）、F-FL-006（循环不变式）、F-FL-010（77 项线性扫描）、F-FL-011（按值返回 + 无缓存 + 逐层重算）。**已核验无热路径问题**：本文件没有任何函数在 Tick/每帧路径上（`GetChangedDirectories` 只在目录变更回调与启停时调用；`Encode`/路径函数都在生成期或启动期），"每帧字符串拼接"这一常见隐患在本文件不成立 | F-FL-003/006/010/011 |
| **6. 死代码** | 见第 4 节：1 个完全无调用的函数、1 组完全无调用的模板函数、4 个零使用宏、1 个条件死代码文件 | 第 4 节 |
| **7. 可优化/可读性/一致性** | `GetFullClass` 长表达式重复（P3）；`GetFullInterface` 内 `GetFullClass` 重复求值（P3）；`GetDotnetVersion` 魔法 `-1`（P3）；`GetAssetClass` 双重查找（P3）；命名体系混乱（`*Path` vs `*Directory`，F-FL-027）；宏卫生（F-FL-021）；`TIsNotUEnum` 名实相反（F-FL-022）；文件名 `TGetUtf8String.inl` vs 符号 `TGetUTF8String`（F-FL-020）。**注释与代码不符**：未发现（`FUnrealCSharpFunctionLibrary.cpp` 注释极少，`TFieldIteratorExt.inl:9-20` 的域成员注释与代码一致，唯 `:112` "We shouldn't be able to get here for non-classes" 与 `:29` 的 `InStruct->IsA(UClass::StaticClass())` 前置判断互相印证，无矛盾） | F-FL-020/021/022/023/024/025/026/027 |
| **8. 平台兼容** | `GetDotNet()` 单分支硬编码 macOS 路径（**P2**，原文此处误写 P1）；`LIB_HOSTFXR` 无兜底（P3）；`AccessPrivate` 依赖非标准访问放宽、Shipping 也编入（P2）；`TFunctionPointer` 的 union 双关与成员函数指针尺寸假设（P2）；`TFieldIteratorExt` 已与引擎实现发生无守卫分叉（P2）。**已核验不是问题**：(a) 全库路径比较统一使用 `FPaths::Combine`（`operator/`）而非手工 `\\`，UE 内部统一以 `/` 归一化，因此 `FCoreCLRFunctionLibrary.cpp:10` 的 `"%s/Binaries/%s"` 不构成跨平台 bug；(b) 托管程序集在所有平台都是 `.dll`，`DLL_SUFFIX` 用于 `Interop/UE/Game/Custom` 是正确的，`.so/.dylib` 只适用于原生库且已在 `LIB_HOSTFXR` 正确分支；(c) 本文件**未**混用 `FPaths::LaunchDir()`（按 `GetFullScriptDirectory`=`ProjectDir()`、`GetFullPublishDirectory`=`ProjectContentDir()`、`GetCodeAnalysisPath`=`ProjectIntermediateDir()`、`GetPluginDirectory`=`IPluginManager::GetBaseDir()` 统一），打包后路径由 `ProjectContentDir()` + 打包设置 `DirectoriesToAlwaysStageAsUFS`（`UnrealCSharpEditor.cpp:307`）保证一致 | F-FL-004/008/014/016/017/018 |

---

## 6. 未覆盖 / 存疑项

1. ~~**无引擎源码**~~ **【已改正 —— 本机确有引擎源码】**：`Engine\Source` **真实存在**（约 20,989 个 `.cpp`），原"工作区无 `Engine/`"的说明是**错的**，据此打的所有"置信度降级"已逐条撤销。已核实并写入各 Finding 的引擎依据：
   - `FString::Equals(const FString&)` 的默认 `ESearchCase` = **`CaseSensitive`** → `Runtime/Core/Public/Containers/UnrealString.h.inl:1492`（**F-FL-010 的 (c) 子项据此证伪**）；
   - `FJsonSerializer::Deserialize` 失败时**不写**输出参数（`OutObject = State.Object;` 在成功分支之后）→ `Runtime/Json/Public/Serialization/JsonSerializer.h:56-72`（**F-FL-002 的触发前提由惯例推断升为源码实证**）；
   - `TIsTOptional_V` **存在**（主模板 + 4 个 cv 特化）→ `Runtime/Core/Public/Misc/Optional.h:437-444`（**F-FL-019 的降级撤销**）；
   - `TFieldIterator` 在本版本的真实形态（`EFieldIterationFlags` 位标志 + 保留的遗留三 flag 构造 + 引擎**没有** `GetAllInterfaceClasses`）→ `Runtime/CoreUObject/Public/UObject/UnrealType.h:6658-6796`（**F-FL-017 由"无法逐行 diff"改为已完成逐行比对**）。
   **仍未核实（保留）**：`FPlatformProcess::CreateProc(..., PipeWriteChild, PipeReadChild)` 的管道方向语义（F-FL-009 只报告"不检查返回值/无超时/无条件 Terminate"，**未**下"stdin/stdout 接反"的结论）；`UStruct::StaticStruct` 对逐个 USTRUCT 的可用性（F-FL-015 的"两者同时为真"由 trait 定义形态与 `BINDING_SCRIPT_STRUCT` 清单推定，未逐类型实例化探针）。
2. **`GetGenerationPath` 的函数内静态缓存**（`.cpp:1001,1007`，`static auto GameProxyPath = GetGameProxyDirectory();`）：它把"首次调用时刻的脚本目录设置"永久固化。这与 F-FL-011 的"缓存策略自相矛盾"是同一根因，但"设置变更后需要重启"是否有意为之无法判定，故未单列 Finding，仅在此登记。**建议核实**：确认 `UUnrealCSharpEditorSetting::GetScriptDirectory()`（`UnrealCSharpEditorSetting.h:56`）是否可在会话中修改；若是，则这是一个未上报的 P2（生成目录会与目录监视/发布路径不一致）。
3. **CoreMacro 的 Attribute 宏族**（`ClassAttributeMacro`/`FunctionAttributeMacro`/`GenericAttributeMacro`/`MetaDataAttributeMacro`/`PropertyAttributeMacro`，共 **5 个文件**，占 `CoreMacro/` 16 个文件中的大部分）：本次只做了"命名冲突 + 值一致性"的机械核对（结果：与 `Source/UnrealCSharp/Public/Macro/` 下 89 个宏**零交集**；在整个 `Source/` 内除 `Macro.h:86-90` 的 `LIB_HOSTFXR` 与 `CompilerMacro.h:6-20` 的条件重定义外**无重定义**），未逐一核对每个 `CLASS_XXX_ATTRIBUTE` 字符串是否与 C# 侧（`Script/CodeAnalysis`、`SourceGenerator`）读取的 attribute 名一致。**这是本报告最大的未覆盖面**，建议交给 C# 侧报告做交叉核对（方法：把 `Source/**/*Macro.h` 的 `TEXT("...Attribute")` 全量抽出，与 `Script/` 下 C# 代码里的 attribute 名集合做差集）。（原文此处写的"17 个"为计数错误，已修正为 5 个文件。）
4. **`SyncProcess` 的管道方向**：如第 1 点所述，未定论。
5. **本报告的 P 级口径（已按源码证据重定并说明）**：按 `_CONVENTIONS.md` 第 2 节 —— "需要外部损坏输入才能触发的崩溃"记为 P1（F-FL-002，维持）；"当前不可达/需误配置才触发的崩溃与越界"记为 P2（F-FL-001/005/006/007）；"风格/命名/可读性"记为 P3。**P0 = 0 条**（F-FL-001 由原 P0 → P1 → P2）。若采用"可达即 P0"口径，则 F-FL-002 应升为 P0 —— 但这与本报告自定的口径冲突，需统一裁决。
6. **与其它报告的同源条目一致性（新增）**：
   - `F-FL-001` ↔ `02-…/06` **F-BR-004**：同一缺陷（域类型编译期宏 vs 运行期 ini 双源 → `Create()` 返回 nullptr → `FDomain.cpp:31` 解引用）。**两报告级别曾不一致**（本报告 P1 / F-BR-004 P2），已把 F-FL-001 校正为 **P2** 以对齐；两报告的代码事实与可达性（潜伏）结论一致。
   - `F-FL-016` ↔ `08-…/04` **F-ABI-023**（`TFunctionPointer` union 类型双关）：**未读 `08-…/04`，交叉核对未完成**（卡点：步数预算）。
   - `F-FL-023` ↔ `07-…/01`（TargetFramework↔`DotnetVersion.h` 对照表）：**未读 `07-…/01`，交叉核对未完成**（卡点：步数预算）。
   - `F-FL-009` ↔ `05-…/01`（编译流程）：**未读 `05-…/01`，交叉核对未完成**（卡点：步数预算）。

