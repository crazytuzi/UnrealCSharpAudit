# UObject 生命周期与绑定配对专项审计

> 本报告各条的**严重度见各 Finding 的「复核结论」字段**（19 条：1 条判定为非缺陷、5 条级别调整）。
> **处置更新（修复提交 `492ce5f7`）**：`F-LIFE-003` 已修复（两个委托属性描述符 `Set` 都先把 `GetDelegate<>` 解析提到 `InitializeValue` 之前、并在解引用前判空；该条与 `01-UnrealCSharp运行时/07b-委托Handler与OptionalHelper.md` 的 `F-DEL-003` **同源，本次一并修复**，见该 Finding 正文「处置（已执行）」）。**编号与计数口径不变**（仍 **19 条：1 条判定为非缺陷、5 条级别调整**）；`严重度`/`复核结论`/`可达性` 等字段保持原值，只加处置标注。（修复提交：`492ce5f7` "Null Validation"）





> 分析范围：`Plugins/UnrealCSharp/Source/`（7 个模块）+ `Script/`
> 排除：`Source/ThirdParty/`、`Intermediate/`、`Binaries/`、`*.old`、`Script/**/obj/`、`Script/**/bin/`
> 审计范围：UObject 生命周期与绑定配对（4 类表：根引用配对 / 裸指针 GC 保护判定 / 委托解绑配对 / 共享指针判定）
> 覆盖情况：4 张配对/判定表 + 19 条发现 + 热重载泄漏清单；未覆盖项见 §7。

## 0. 覆盖范围与阅读清单

| 文件 | 行数 | 是否读完 | 备注 |
|---|---|---|---|
| `Source/UnrealCSharp/Public/Registry/FReferenceRegistry.h` | 30 | ✅ 全读 | 全插件唯一 `FGCObject` |
| `Source/UnrealCSharp/Private/Registry/FReferenceRegistry.cpp` | 81 | ✅ 全读 | 表 1 #15、表 2 #1 |
| `Source/UnrealCSharp/Public/Registry/FDelegateRegistry.h` | 52 | ✅ 全读 | 表 4 |
| `Source/UnrealCSharp/Public/Registry/FDelegateRegistry.inl` | 108 | ✅ 全读 | `GetDelegate` 可返回 nullptr（F-LIFE-003 依据） |
| `Source/UnrealCSharp/Private/Registry/FDelegateRegistry.cpp` | 55 | ✅ 全读 | `delete` + `GCHandle_Free` 配对正确 |
| `Source/UnrealCSharp/Public/Registry/FCSharpBind.h` | 91 | ✅ 全读 | `NotOverrideTypes` 是 weak |
| `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp` | 545 | 读 1-250、251-380、470-545 | 覆盖 `AddToRoot`/`DuplicateFunction`/`TObjectRange` 全部相关点 |
| `Source/UnrealCSharp/Public/Reflection/Delegate/FDelegateHelper.h` | 102 | ✅ 全读 | `TWeakObjectPtr<UDelegateHandler>` |
| `Source/UnrealCSharp/Private/Reflection/Delegate/FDelegateHelper.cpp` | 81 | ✅ 全读 | 表 1 #2 |
| `Source/UnrealCSharp/Public/Reflection/Delegate/FMulticastDelegateHelper.h` | — | ✅ 全读（grep 行） | `TWeakObjectPtr` |
| `Source/UnrealCSharp/Private/Reflection/Delegate/FMulticastDelegateHelper.cpp` | 105 | ✅ 全读 | 表 1 #3 |
| `Source/UnrealCSharp/Public/Reflection/Delegate/DelegateHandler.h` | 159 | ✅ 全读 | 表 2 #5、#6 |
| `Source/UnrealCSharp/Public/Reflection/Delegate/MulticastDelegateHandler.h` | 122 | ✅ 全读 | 同类裸指针 |
| `Source/UnrealCSharp/Private/Reflection/Delegate/DelegateHandler.cpp` | 101 | ✅ 全读 | F-LIFE-002 关键证据（:10、:64） |
| `Source/UnrealCSharp/Public/Reflection/Delegate/FDelegateWrapper.h` | 15 | ✅ 全读 | **F-LIFE-002 核心** |
| `Source/UnrealCSharp/Public/Reference/FReference.h` / `TDelegateReference.h` | 26 / 17 | ✅ 全读 | 引用链 |
| `Source/UnrealCSharp/Private/Reflection/Property/DelegateProperty/FDelegatePropertyDescriptor.cpp` | 64 | ✅ 全读 | **F-LIFE-003** |
| `Source/UnrealCSharp/Private/Reflection/Property/DelegateProperty/FMulticastDelegatePropertyDescriptor.cpp` | 77 | ✅ 全读 | **F-LIFE-003** |
| `Source/UnrealCSharp/Private/Reflection/Function/FCSharpFunctionRegister.cpp` | 81 | ✅ 全读 | 表 1 #1 的配对方 |
| `Source/UnrealCSharp/Public/Reflection/Function/FFunctionDescriptor.h` | 34 | ✅ 全读 | `TWeakObjectPtr<UFunction>` 正确 |
| `Source/UnrealCSharp/Public/Reflection/Function/FManagedFunctionDescriptor.h` | 16 | ✅ 全读 | — |
| `Source/UnrealCSharp/Public/Reflection/Function/FCSharpDelegateDescriptor.h` / `.cpp` | 54 / 27 | ✅ 全读 | — |
| `Source/UnrealCSharp/Public/Reflection/Property/FPropertyDescriptor.h` | 82 | ✅ 全读 | 无裸 UObject 成员 |
| `Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h` | 361 | ✅ 全读 | 表 2 #2-#4 |
| `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp` | 636 | ✅ 全读 | **销毁顺序 + F-LIFE-019** |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterObject.cpp` | 152 | ✅ 全读 | 表 1 #13、F-LIFE-009 |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterDelegate.cpp` | 198 | ✅ 全读 | — |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterClass.cpp` | 56 | ✅ 全读 | 表 1 #11/#12 的配对方 |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterInputComponent.cpp` | 345 | 读 285-334 | **F-LIFE-017** |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp` | 227 | 读 70-119 | **F-LIFE-017** |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterMulticastDelegate.cpp` | — | grep `.Function(` | **F-LIFE-018** |
| `Source/UnrealCSharp/Private/Domain/FDomain.cpp` | 155 | 读 1-129 | 域生命周期 |
| `Source/UnrealCSharp/Private/Listener/FUObjectListener.cpp` / `.h` | 54 / 26 | ✅ 全读 | 表 3 #8/#9 |
| `Source/UnrealCSharp/Public/Delegate/FUnrealCSharpModuleDelegates.h` | 17 | ✅ 全读 | 静态委托对象 |
| `Source/UnrealCSharp/Private/UnrealCSharp.cpp` | 57 | ✅ 全读 | 模块入口 1/3 |
| `Source/UnrealCSharpCore/Private/UnrealCSharpCore.cpp` | 64 | ✅ 全读 | 模块入口 2/3 |
| `Source/UnrealCSharpCore/Private/Dynamic/FDynamicEnumGenerator.cpp` / `.h` | 303 / — | ✅ 全读 | **F-LIFE-004** |
| `Source/UnrealCSharpCore/Private/Dynamic/FDynamicInterfaceGenerator.cpp` | 323 | 读 55-114、230-323 | **F-LIFE-005** |
| `Source/UnrealCSharpCore/Private/Dynamic/FDynamicClassGenerator.cpp` | 932 | 读 100-249、360-559、810-932 | 表 1 #6/#7/#10 |
| `Source/UnrealCSharpCore/Private/Dynamic/FDynamicStructGenerator.cpp` | 394 | 读 235-394 | 表 1 #5 |
| `Source/UnrealCSharpCore/Public/Dynamic/FDynamicBlueprintExtensionScope.h` | 53 | ✅ 全读 | 表 1 #4（正确 RAII） |
| `Source/UnrealCSharpCore/Public/Dynamic/FDynamicGeneratorCore.h` | 119 | 读 55-114 | `IteratorObject` |
| `Source/UnrealCSharpCore/Private/Dynamic/FDynamicGeneratorCore.cpp` | 1263 | 读 10-59、1036-1210 + grep | **F-LIFE-010**（复核补读 1036-1210，覆盖 6 个静态数组定义与全部消费点） |
| `Source/UnrealCSharpCore/Private/Dynamic/FDynamicDependencyGraph.cpp` / `.h` | 215 / — | ✅ 全读 | **F-LIFE-016** |
| `Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp` | 2409（总行数口径；原文写 ~1700，按 §1 口径校正） | 读 1-29、866-975 + grep | `Deinitialize` 是悬垂源头 |
| `Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRDomain.cpp` | 312 | 读 120-299 | 热重载链路 |
| `Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp` | 520 | 读 390-459 | 同构 |
| `Source/UnrealCSharpCore/Private/Domain/Script/FScriptDomainFactory.cpp` | 60 | ✅ 全读 | `Set(nullptr)` 正确 |
| `Source/UnrealCSharpCore/Private/Domain/Script/IScriptDomain.cpp` | 13 | ✅ 全读 | — |
| `Source/UnrealCSharpCore/Private/Listener/FEngineListener.cpp` | 80 | ✅ 全读 | 热重载触发点 |
| `Source/UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp` | 397 | ✅ 全读 | 模块入口 3/3 |
| `Source/UnrealCSharpEditor/Public/UnrealCSharpEditor.h` | 62 | ✅ 全读 | 表 1 #14、表 3 #40 |
| `Source/UnrealCSharpEditor/Private/Listener/FEditorListener.cpp` | 755 | ✅ 全读 | **F-LIFE-001 / F-LIFE-007** |
| `Source/UnrealCSharpEditor/Private/NewClass/ClassCollector.cpp` | 356 | 读 20-199 + grep | 表 3 #31-#34、F-LIFE-012 |
| `Source/UnrealCSharpEditor/Private/ContentBrowser/DynamicDataSource.cpp` | 882 | 读 1-115 + grep | 表 3 #35/#36 |
| `Source/UnrealCSharpEditor/Private/ToolBar/UnrealCSharpPlayToolBar.cpp` | 104 | ✅ 全读 | **F-LIFE-014** |
| `Source/Compiler/Private/FCSharpCompilerRunnable.cpp` | 483 | 读 1-60、205-264 | 热重载触发 |
| `Script/Interop/AssemblyLoader/AssemblyLoader.cs` | 63 | ✅ 全读 | **F-LIFE-015** |
| `Script/Interop/AssemblyLoader/UnrealAssemblyLoadContext.cs` | 45 | ✅ 全读 | `isCollectible: true` |
| `Script/Interop/Handle/HandleData.cs` | 147 | ✅ 全读 | `Clear()` 正确 |

**grep 模式清单（本报告实际执行过的搜索模式与命中数）**

- 第 1 类（GC 保护）：`AddToRoot|RemoveFromRoot` → **26**；`AddReferencedObjects|FGCObject` → **6**；`TStrongObjectPtr` → **1**；`TObjectIterator|TFieldIterator|GetObjectsWithOuter` → **46**；`MarkAsGarbage|MarkPendingKill|GEngine->|GetObjectsWithOuter` → **5**（`GEngine->` 与 `MarkPendingKill` 命中 **0**，即插件不使用 `GEngine` 也不使用已废弃的 `MarkPendingKill`）
- 第 2 类（共享指针）：`TSharedPtr|TSharedRef|TWeakPtr|TUniquePtr|MakeShared|MakeShareable|SharedThis|AsShared` → **165**；`CreateRaw|CreateUObject|AddRaw|AddLambda|CreateSP|CreateWeakLambda|CreateStatic` → **44**；`TSharedPtr<.*>\(this\)` → **0**
- 第 3 类（委托解绑）：见上（`AddRaw`/`AddStatic`/`AddUObject`/`AddLambda`/`CreateRaw` 共 44 处，逐处判定了捕获对象与生命周期）；`AddDynamic|AddUniqueDynamic|RemoveDynamic|BindDynamic|BindUObject|BindRaw|BindStatic|AddSP|AddWeakLambda` → **裸字符串命中 5 处，但全部是子串误命中**（`DynamicDataSource.cpp:73` `AddDynamicSection`、`PlayToolBar.cpp:54`/`DynamicDataSource.cpp:700`/`DynamicNewClassUtils.h:9`/`DynamicNewClassUtils.cpp:12` 的 `OpenAddDynamicClassToProjectDialog`）→ **实质命中 0**：插件不使用 UE 反射委托宏，全部改用自研 `UDelegateHandler`/`UMulticastDelegateHandler` 桥接（复核修正口径，避免下游按 5 误判）
- 第 4 类（热重载）：`AssemblyLoadContext|Unload\(\)|isCollectible|GCHandle` → **9**；`GCHandle_Free|GCHandleFree`（Source/UnrealCSharp）→ **32**；`UnloadAssembly|LoadAssembly\(` → **25**；`InitializeAssembly|FDomain::Initialize|ReloadCompleteDelegate` → **34**；`MarkOutdated|IsOutdated|Deactivate\(\)` → **12**

---

## 1. 模块职责与生命周期速览（本审计视角）

从 UObject 生命周期看，本插件有三层"持有 UObject"的机制，**优先级与风险依次递增**：

1. **`FReferenceRegistry : FGCObject`（正确的一层）** —— `FReferenceRegistry.h:5`，唯一实现 `AddReferencedObjects` 的类（`FReferenceRegistry.cpp:22-25`），内部是 `TArray<TObjectPtr<UObject>> ObjectArray`（`:29`）。它由托管侧显式 `AddReference`/`RemoveReference` 驱动（`FRegisterObject.cpp:111-129`），是插件**唯一**符合 UE GC 契约的 UObject 保护手段。
2. **`AddToRoot()` 层（有风险的一层）** —— 共 **12** 个 Add 调用点 + 1 处 C# 侧转发（表 1；原文写"13 个 Add 点"，按全插件 grep 逐点核对后修正为 12），用于 `UClass`/`UBlueprint`/`UScriptStruct`/`UEnum`/`UFunction`/`UDelegateHandler`/`UMulticastDelegateHandler`/`UDynamicBlueprintExtension`。这类对象永久进 rootset，靠**手工 `RemoveFromRoot`** 平衡（全插件 `RemoveFromRoot` 调用点 **10** 处；`AddToRoot|RemoveFromRoot` 总命中 **26** 处，含 `FRegisterObject.cpp:85/93` 两个函数名与 `:141/142` 两处注册，12+10+4=26 ✅）。审计发现 2 处完全未配对（枚举、接口类）、3 处只在 `ReInstance` 路径配对、1 处反向未配对。
3. **裸指针缓存层（最危险的一层）** —— 大量 `TMap`/`TSet`/`static TArray` 直接以 `UClass*`/`UEnum*`/`UDynamicScriptStruct*`/`FClassReflection*`/`FMethodReflection*` 为键或元素，**既无 `UPROPERTY`、也无 `AddReferencedObjects`、也不继承 `FGCObject`**（表 2）。它们的安全性完全"借用"了第 2 层的 root 标记，而一旦所指向的对象是**非 UObject**（如 `FClassReflection`）或**被 `MarkAsGarbage`**（如 `ReInstance` 后的旧类），root 就保护不了它们。

**三个模块入口的清理完备性**：
- `Source/UnrealCSharp/Private/UnrealCSharp.cpp:10-34`：`StartupModule` 挂 2 个 raw（`:12,:15`），`ShutdownModule` 全解（`:25,:31`）—— ✅ 配对。
- `Source/UnrealCSharpCore/Private/UnrealCSharpCore.cpp:8-17`：`StartupModule`/`ShutdownModule` 均为空实现，**本模块不做任何注册**（真正的注册发生在 `FEngineListener` 通过 `Activate()`/`Deactivate()` 广播的钩子里）—— ✅ 无泄漏，但也意味着**模块卸载本身不触发清理**。
- `Source/UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp:41-236`：13 类注册（Settings ×2、Style、Commands、PropertyEditor 自定义化 ×2、ToolMenus、`OnPostEngineInit`、GameplayTag、5 个 `FAutoConsoleCommand`、`TStrongObjectPtr<UDynamicDataSource>`、`FTSTicker`）**除 cook 路径的 `OnFilesLoaded().AddLambda`（`:167`）外全部配对** —— 见 F-LIFE-013。
- **真正的清理入口是 `FCSharpEnvironment::Deinitialize()`**（`FCSharpEnvironment.cpp:147-269`），由 `FUnrealCSharpCoreModule::Deactivate()` 经 2 层委托广播触发；其**销毁顺序刻意正确**（`Domain` 最后删，所有 `GCHandle_Free` 都在域存活时执行）。而 `FCSharpEnvironment::OnUObjectArrayShutdown()`（`:323-325`）是空实现，UObject 系统关闭时无兜底 —— 见 F-LIFE-019。

## 2. 关键调用链

1. **UObject 从 C++ 进入 C#（对象包装 → GC 保护）**
   `FCSharpEnvironment::NotifyUObjectCreated`（`FCSharpEnvironment.cpp:276-300`）← `FUObjectListener::NotifyUObjectCreated`（`FUObjectListener.cpp:41-44`）← `GUObjectArray` 创建回调（注册于 `FUObjectListener.cpp:29`）→ 游戏线程直接 `Bind<true>(InObject)`（`:291`），否则入 `AsyncLoadingObjectArray`（`:297`）→ `FCoreDelegates::OnAsyncLoadingFlushUpdate`（注册于 `:87`）→ `FCSharpEnvironment::OnAsyncLoadingFlushUpdate`（`:369-452`）→ `FCSharpBind::BindClassDefaultObject`（`:436`）/ `Bind<true>`（`:440`）→ `FClassRegistry::AddClassConstructor` + `FCSharpBind::BindImplementation`（`FCSharpBind.cpp:106-311`）。
2. **C++ 委托 → C#（`UDelegateHandler` 桥接）**
   `FDelegatePropertyDescriptor::NewRef`（`FDelegatePropertyDescriptor.cpp:33-52`）→ `new FDelegateHelper(Property->GetPropertyValuePtr(InAddress), Property->SignatureFunction)`（`:39`）→ `FDelegateHelper::Initialize`（`FDelegateHelper.cpp:20-30`）→ `NewObject<UDelegateHandler>()` + `AddToRoot()`（`:22,:24`）→ `AddDelegateReference`（`:47`）→ `FDelegateRegistry.inl:43-55`（新增 `TDelegateReference<FDelegateHelper>` 并挂到宿主对象的 `FReferenceRegistry`）。
3. **C# 绑定到 UE 委托（→ 悬垂风险链）**
   C# `FDelegate.Register` → `FRegisterDelegate.cpp:18` `FCSharpBind::Bind<FDelegateHelper>`；C# `FDelegate.Bind` → `FRegisterDelegate.cpp:30-49` → `DelegateHelper->Bind(FoundObject, FoundMethod)`（`FDelegateHelper.cpp:44-50`）→ `UDelegateHandler::Bind`（`DelegateHandler.cpp:54-65`）→ **`DelegateWrapper = {InObject, InMethod}`（`:64`，保存裸 `FMethodReflection*`）** → 域卸载时 `FReflectionRegistry::Deinitialize()`（`FReflectionRegistry.cpp:867-877`）`delete Class` → 之后 `ProcessEvent`（`DelegateHandler.cpp:4-17`）→ `CallDelegate`（`FCSharpDelegateDescriptor.cpp:10`）→ **UAF（F-LIFE-002）**。
4. **热重载（C# 编译 → 旧域卸载 → 新域加载）**
   `FCSharpCompilerRunnable.cpp:225` `Deactivate()` → `UnrealCSharpCore.cpp:29-37` → `UnrealCSharp.cpp:50-53` → `FCSharpEnvironment.cpp:332-335` → `Deinitialize()`（`:147-269`，Domain 最后删）→ `FDomain.cpp:15-18` → `FScriptDomainFactory.cpp:47-59` → `FCoreCLRDomain.cpp:240-270` → `FReflectionRegistry.cpp:867-877` → `AssemblyLoader.cs:31-61` → `FCSharpCompilerRunnable.cpp:229` `Activate()` → `FCSharpEnvironment::Initialize()`（`:57-145`）→ `FDynamicGenerator.cpp:18-34` → `SetMetaData` → **`FDynamicGeneratorCore.cpp:1040` 悬垂静态数组（F-LIFE-010，P0）**。
5. **动态类型生成与再实例化（root 配对链）**
   `FDynamicClassGenerator::Generator`（`:141-235`）→ 命中 `DynamicClassMap` 时 `OldClass = DynamicClassMap[ClassName]`（`:174`）→ `GeneratorClass`（`:387-398`）`Class->AddToRoot()`（`:393`）/ `GeneratorBlueprintGeneratedClass`（`:409-432`）`Blueprint->AddToRoot()`（`:419`）→ `ReInstance(OldClass, Class)`（`:435-544`）→ `Blueprint->RemoveFromRoot()+MarkAsGarbage()`（`:533,:535`）或 `InOldClass->RemoveFromRoot()+MarkAsGarbage()`（`:540,:542`）。结构体同构（`FDynamicStructGenerator.cpp:249` ↔ `:380`）。**枚举（`:201`）与接口（`:263`）缺失这一环**。
6. **编辑器资产事件 → 动态代码重生成（未配对链）**
   AssetRegistry 加载完成 `OnFilesLoaded` → `FEditorListener::OnFilesLoaded`（`FEditorListener.cpp:79` 注册，`:340-351` 再挂 4 个）→ 任意资产增删改名 → `OnAssetAdded`（`:353`）→ `OnAssetChanged`（`:529-556`）→ `FAssetGenerator::Generator` + `FCSharpCompiler::Get().Compile()`（`:548`）→ 回到调用链 4。**这 5 个绑定点全无解绑（F-LIFE-001，P0）**。
7. **循环源码目录监视 → 编译（句柄覆盖链）**
   `FEditorListener` 构造（`:86-97`）逐目录 `RegisterDirectoryChangedCallback_Handle(..., OnDirectoryChangedDelegateHandle, ...)` → `.cs` 文件变化 → `OnDirectoryChanged`（`:415-464`）→ `FileChanges.Add` → 应用激活时 `Compile()`（`:636-650`）→ `FCSharpCompiler::Get().Compile(FileChanges)`。析构时（`:110-120`）只有最后一个句柄能注销 —— **F-LIFE-007**。

---

## 3. 核心交付物（4 张配对/判定表）

### 表 1：UObject 根引用 / 强引用配对表（`AddToRoot` ↔ `RemoveFromRoot`，`TStrongObjectPtr` 构造 ↔ `Reset`）

grep 模式：`AddToRoot|RemoveFromRoot`（Source/ 全模块，命中 26 处）、`TStrongObjectPtr`（命中 1 处）。

| # | 操作 | 文件:行 | 对象/类型 | 配对操作 | 配对文件:行 | 是否配对 | 后果 |
|---|---|---|---|---|---|---|---|
| 1 | `AddToRoot()` | `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:530` | `UFunction* NewFunction`（重复出来的 override 函数） | `RemoveFromRoot()` | `Source/UnrealCSharp/Private/Reflection/Function/FCSharpFunctionRegister.cpp:61`（析构） | ✅ 条件配对 | 仅当 `InClass` 处于 RootSet/DisregardForGC 时才 AddToRoot；否则走 `MarkAsGarbage()`（:65） |
| 2 | `AddToRoot()` | `Source/UnrealCSharp/Private/Reflection/Delegate/FDelegateHelper.cpp:24` | `UDelegateHandler*` | `RemoveFromRoot()` | 同文件 `:38`（`Deinitialize()`，由 `~FDelegateHelper()` :17 调用） | ✅ 配对（依赖 `delete FDelegateHelper`） | 若析构被跳过则 handler 永久 rooted |
| 3 | `AddToRoot()` | `Source/UnrealCSharp/Private/Reflection/Delegate/FMulticastDelegateHelper.cpp:25` | `UMulticastDelegateHandler*` | `RemoveFromRoot()` | 同文件 `:39` | ✅ 配对（同上） | 同上 |
| 4 | `AddToRoot()` | `Source/UnrealCSharpCore/Public/Dynamic/FDynamicBlueprintExtensionScope.h:23` | `UDynamicBlueprintExtension*` | `RemoveFromRoot()` | 同文件 `:42`（RAII 析构） | ✅ 配对 | 正确，但注意 :18 `AddExtension` 也持有它，双持有 |
| 5 | `AddToRoot()` | `Source/UnrealCSharpCore/Private/Dynamic/FDynamicStructGenerator.cpp:249` | `UDynamicScriptStruct*` | `RemoveFromRoot()` | 同文件 `:380`（`ReInstance()`） | ⚠️ 只在 ReInstance 路径配对 | 首次生成后如无 ReInstance（如编辑器关闭前最后一次生成）则永久 rooted |
| 6 | `AddToRoot()` | `Source/UnrealCSharpCore/Private/Dynamic/FDynamicClassGenerator.cpp:393` | `UClass*`（非 BPGC 动态类） | `RemoveFromRoot()` | 同文件 `:540`（`ReInstance()` else 分支） | ⚠️ 只在 ReInstance 路径配对 | 同上；`MarkAsGarbage()` :542 紧接其后 |
| 7 | `AddToRoot()` | `Source/UnrealCSharpCore/Private/Dynamic/FDynamicClassGenerator.cpp:419` | `UBlueprint*`（BPGC 的 ClassGeneratedBy） | `RemoveFromRoot()` | 同文件 `:533`（`ReInstance()` if 分支） | ⚠️ 只在 ReInstance 路径配对 | 注意 :413 的 `Class` 本身**没有** AddToRoot，靠 `Blueprint->GeneratedClass`(:423) 保活，两边是对称的 |
| 8 | `AddToRoot()` | `Source/UnrealCSharpCore/Private/Dynamic/FDynamicEnumGenerator.cpp:201` | `UEnum*` | **无** | — | ❌ **完全未配对** | **永久 rooted**：枚举及其 `Names` 名字表永不回收。见 F-LIFE-004 |
| 9 | `AddToRoot()` | `Source/UnrealCSharpCore/Private/Dynamic/FDynamicInterfaceGenerator.cpp:263` | `UClass*`（动态 UInterface） | **无** | — | ❌ **完全未配对** | **永久 rooted**。见 F-LIFE-005 |
| 10 | `RemoveFromRoot()` | `Source/UnrealCSharpCore/Private/Dynamic/FDynamicClassGenerator.cpp:893` | `UActorComponent* InheritableComponentTemplate` | **无匹配的 AddToRoot** | — | ❌ **反向未配对** | `RemoveFromRoot` 作用于未 rooted 对象：UE 内部只是从 rootset 查找失败并告警/空操作，属逻辑不一致（不是崩溃），见 F-LIFE-011 |
| 11 | `AddToRoot()` | `Source/UnrealCSharp/Private/Domain/Interop/FRegisterInputComponent.cpp:312` | `UFunction*`（C# 侧动态绑定的输入函数） | 仅当 C# 侧调用 `FRegisterClass::RemoveFunctionImplementation`（`FRegisterClass.cpp:23`）才移除 | `Source/UnrealCSharp/Private/Domain/Interop/FRegisterClass.cpp:23` | ⚠️ 依赖托管侧显式反注册 | 托管侧不调用则永久 rooted，且 outer 类（池化输入组件类）销毁后成为孤立 root |
| 12 | `AddToRoot()` | `Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp:97` | `UFunction*` | 同上 | `FRegisterClass.cpp:23` | ⚠️ 同上 | 同上 |
| 13 | `AddToRoot()/RemoveFromRoot()` | `Source/UnrealCSharp/Private/Domain/Interop/FRegisterObject.cpp:89 / :97` | 任意 `UObject*` | 由 C# `UObject.AddToRoot()/RemoveFromRoot()` 成对调用（:141/:142 注册） | 托管侧 | ⚠️ 无法在 C++ 侧静态判定 | **暴露给 C# 的裸 root API，是最大的托管侧泄漏入口**，见 F-LIFE-009 |
| 14 | `TStrongObjectPtr` 构造 | `Source/UnrealCSharpEditor/Public/UnrealCSharpEditor.h:61` | `UDynamicDataSource`（`TStrongObjectPtr<UDynamicDataSource> DynamicDataSource`） | `DynamicDataSource.Reset(NewObject<UDynamicDataSource>(...))` | `Source/UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp:147`（构造）↔ `:229`（`DynamicDataSource.Reset()`） | ✅ 配对 | `TStrongObjectPtr` 只此一处，构造/Reset 都在 `StartupModule`/`ShutdownModule`，配对正确 |
| 15 | `FReferenceRegistry : FGCObject` | `Source/UnrealCSharp/Public/Registry/FReferenceRegistry.h:5`、`FReferenceRegistry.cpp:22-25` | `TArray<TObjectPtr<UObject>> ObjectArray` | `~FReferenceRegistry()` :7-20 `ObjectArray.Empty()` + 由 `delete ReferenceRegistry`（`FCSharpEnvironment.cpp:237`）触发的 FGCObject 反注册 | — | ✅ 配对 | **本插件唯一一处正确的 UObject 引用保护机制**；但 `ObjectArray` 的可见性靠托管侧显式 `AddReference`（F-LIFE-008） |

**结论：12 处 `AddToRoot` 调用点 / 26 处根操作（含 4 处函数名与注册行）中，2 处完全无 RemoveFromRoot（枚举、接口类），2 处依赖 C# 侧显式调用，1 处反向未配对（RemoveFromRoot 无 AddToRoot），3 处只在 ReInstance 路径配对。**（原文写"13 处 AddToRoot"，按 grep 逐点核对修正为 **12**：表 1 中含 AddToRoot 的行是 #1-#9、#11、#12、#13，共 12 行；#10 只有 RemoveFromRoot，#14/#15 是 `TStrongObjectPtr`/`FGCObject`。算术自洽性验证：12 Add + 10 Remove + 4 非调用命中 = 26 = grep 实测值。）

### 表 2：裸 UObject 指针 GC 保护判定表

判定口径：**类持有裸 `UObject*`/`UClass*`/`UFunction*`/`FProperty*`/`UStruct*` 成员，且既无 `UPROPERTY()`、也无 `AddReferencedObjects`、也不继承 `FGCObject` → GC 后悬垂。**

grep 证据：`AddReferencedObjects|FGCObject` 全插件仅 6 处命中，其中真正的 GC 保护实现只有 `FReferenceRegistry` 一处（`FReferenceRegistry.h:5/13`、`FReferenceRegistry.cpp:22/24`）；`FDynamicInterfaceGenerator.cpp:165` 与 `FDynamicClassGenerator.cpp:295` 只是把父类函数指针 `ClassAddReferencedObjects` 抄给动态类（不是本插件自己的保护）。

| # | 类（文件） | 成员名 | 类型 | 文件:行 | 有 UPROPERTY | 有 AddReferencedObjects | 继承 FGCObject | 判定 | 严重度 |
|---|---|---|---|---|---|---|---|---|---|
| 1 | `FReferenceRegistry`（`Source/UnrealCSharp/Public/Registry/FReferenceRegistry.h`） | `ObjectArray` | `TArray<TObjectPtr<UObject>>` | `FReferenceRegistry.h:29` | 否（非 UObject） | ✅ 是（`:13` 覆写，`.cpp:22-25`） | ✅ 是（`:5`） | **受保护** | — |
| 2 | `FCSharpEnvironment`（`Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h`） | `AsyncLoadingObjectArray` | `TArray<FWeakObjectPtr>` | `FCSharpEnvironment.h:327` | 否 | 否 | 否 | 安全（weak） | — |
| 3 | `FCSharpEnvironment`（同上） | `PendingBindClasses` | `TArray<TWeakObjectPtr<UClass>>` | `FCSharpEnvironment.h:330` | 否 | 否 | 否 | 安全（weak） | — |
| 4 | `FCSharpEnvironment`（同上） | `Domain/DynamicRegistry/CSharpBind/*Registry` | 裸 `T*`（含 `FReferenceRegistry*`） | `FCSharpEnvironment.h:309,334-354` | 否 | 否 | 否 | 安全（非 UObject，进程期单例） | — |
| 5 | `UDelegateHandler`（`Source/UnrealCSharp/Public/Reflection/Delegate/DelegateHandler.h`） | `ScriptDelegate` | `FScriptDelegate*`（**指向宿主 UObject 的属性内存**，`DelegateHandler.cpp:27`） | `DelegateHandler.h:154` | 否 | 否 | 否 | **裸指针指向 UObject 属性内存**：宿主 UObject 被 GC/销毁后 `ScriptDelegate` 悬垂。但 `UDelegateHandler` 自身被 `AddToRoot`（`FDelegateHelper.cpp:24`）且其生命周期由 `FDelegateHelper` 绑定到宿主 C# 包装对象（`TDelegateReference`），**理论上有宿主引用链**；`bNeedFree==false` 时不 delete → **置信度：中** | P2 |
| 6 | `UDelegateHandler`（同上） | `DelegateDescriptor` | `FCSharpDelegateDescriptor*` | `DelegateHandler.h:156` | 否 | 否 | 否 | 非 UObject 堆对象，由 `~UDelegateHandler`/`Deinitialize()`（`DelegateHandler.cpp:46-51`）delete；但其内部持有 `UFunction*`（见 #7） | P2 |
| 7 | `FPropertyDescriptor`（`Source/UnrealCSharp/Public/Reflection/Property/TPropertyDescriptor.inl`，`FCSharpDelegateDescriptor`/`FCSharpFunctionDescriptor` 经由 `FFunctionDescriptor::PropertyDescriptors` 持有） | `Property`（`T*`，即裸 `FProperty*`）/ `Class`（`FClassReflection*`） | 裸 `FProperty*` / 裸 `FClassReflection*` | `TPropertyDescriptor.inl:44`（`T* Property{}`）、`:46`（`FClassReflection* Class{}`）；容器侧 `:25` | 否 | 否 | 否 | **已核实（原文标"待核"）**：descriptor 自身持**裸 `FProperty*`**（`TPropertyDescriptor.inl:44`）与**裸 `FClassReflection*`**（`:46`），无 `UPROPERTY`/`AddReferencedObjects`。其生存期有两种来源：(a) 引擎属性（由 outer `UStruct`/`UClass` 的 `Children` 链保活）→ 只有当宿主类被销毁/`MarkAsGarbage` 时才悬垂；(b) 插件自建属性，由 `DestroyProperty()`（`TPropertyDescriptor.inl:22-30` 内 `delete Property; Property = nullptr;`）显式删除，且调用点受 `bNeedFreeProperty` 门控（如 `FArrayHelper.cpp:41-48`、`FMapHelper.cpp:54,60`、`FSetHelper.cpp:50`）→ 该门控逻辑自洽，未见 double free。`FFunctionDescriptor.h:23` 的 `Function` 是 `TWeakObjectPtr<UFunction>`（安全，见表 4 #7），`:25` 的 `TArray<FPropertyDescriptor*> PropertyDescriptors` 是普通堆对象指针 | **P1（与 #8 合并）** |
| 8 | `FDelegateWrapper`（`Source/UnrealCSharp/Public/Reflection/Delegate/FDelegateWrapper.h`） | `Object` / `Method` | `TWeakObjectPtr<UObject>` / `FMethodReflection*` | `FDelegateWrapper.h:7,9` | 否 | 否 | 否 | `Object` 是 weak → 安全；`Method` 是裸 `FMethodReflection*`，**热重载后反射注册表重建 → 悬垂**，`UDelegateHandler::ProcessEvent` 直接用它（`DelegateHandler.cpp:10`），见 F-LIFE-002 | **P0** |
| 9 | `FCSharpBind` | `NotOverrideTypes` | `TSet<TWeakObjectPtr<UStruct>>` | `FCSharpBind.h:86` | 否 | 否 | 否 | 安全（weak，静态累加但不阻止 GC） | — |
| 10 | `FDynamicClassGenerator` | `NamespaceMap` / `DynamicClassMap` / `DynamicClassSet` | `TMap<UClass*,FString>` / `TMap<FString,UClass*>` / `TSet<UClass*>` | `FDynamicClassGenerator.h:127,131` 等 | 否 | 否 | 否 | **裸 UClass* 静态容器**；靠 `AddToRoot()`(:393/:419) 保活，无 GC 悬垂，但**永不清理 → 热重载累积**，见 F-LIFE-006 | P1 |
| 11 | `FDynamicStructGenerator` | `NamespaceMap` / `DynamicStructMap` / `DynamicStructSet` | `TMap<UDynamicScriptStruct*,FString>` 等 | `FDynamicStructGenerator.h:48,50,52` | 否 | 否 | 否 | 同 #10，靠 `AddToRoot()`(:249) | P1 |
| 12 | `FDynamicEnumGenerator` | `NamespaceMap` / `DynamicEnumMap` / `DynamicEnumSet` | `TMap<UEnum*,FString>` 等 | `FDynamicEnumGenerator.h:42,44,46` | 否 | 否 | 否 | 同 #10，靠 `AddToRoot()`(:201) → 永久 raw 指针 + 永久 root | P1 |
| 13 | `FDynamicInterfaceGenerator` | `NamespaceMap` / `DynamicInterfaceMap` / `DynamicInterfaceSet` | `TMap<UClass*,FString>` 等 | `FDynamicInterfaceGenerator.h:45` | 否 | 否 | 否 | 同 #10 | P1 |
| 14 | `FEditorListener` | `OnDirectoryChangedDelegateHandle` 等 | `FDelegateHandle`（非 UObject） | `FEditorListener.h` | — | — | — | 非 UObject，见表 3 | — |

**GC 保护机制总结**：本插件对 UObject 的存活保护只有两条腿 —— (a) `FReferenceRegistry`（FGCObject + `TObjectPtr` 数组）用于"托管侧显式 AddReference"的对象；(b) 大量 `AddToRoot()` 用于动态类/函数/handler。**没有**任何一处利用 `UPROPERTY()` 保护裸指针成员；所有 C++ 侧缓存容器都是裸 `UClass*/UFunction*/UEnum*/UStruct*`，其正确性完全依赖 (b) 的 root 标记不被误用，而 (b) 本身已发现 2 处完全未配对。

### 表 3：委托 / 事件绑定解绑配对表

grep 模式：`AddDynamic`、`AddUniqueDynamic`、`RemoveDynamic`、`AddUObject`、`AddRaw`、`AddSP`、`AddLambda`、`AddWeakLambda`、`BindDynamic`、`BindUObject`、`BindRaw`、`BindStatic`、`RemoveAll`、`FDelegateHandle`、`CreateRaw`、`CreateUObject`（Source/ 全模块共命中 44 处 `Add*/Create*`）。

| # | 绑定 | 绑定方式 | 绑定文件:行 | 解绑文件:行 | 是否配对 | 未解绑的后果 |
|---|---|---|---|---|---|---|
| 1 | `FUnrealCSharpModuleDelegates::OnUnrealCSharpModuleActive`（`FCSharpEnvironment`） | `AddRaw` | `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:37` | `:48/:53`（`~FCSharpEnvironment`） | ✅ | — |
| 2 | `…OnUnrealCSharpModuleInActive`（`FCSharpEnvironment`） | `AddRaw` | `FCSharpEnvironment.cpp:40` | `:48` | ✅ | — |
| 3 | `FCoreDelegates::OnAsyncLoadingFlushUpdate` | `AddRaw` | `FCSharpEnvironment.cpp:87` | `:174`（`Deinitialize()`） | ✅ | — |
| 4 | `GEditor->OnBlueprintPreCompile()` | `AddRaw` | `FCSharpEnvironment.cpp:93` | `:165` | ✅ | — |
| 5 | `GEditor->OnBlueprintCompiled()` | `AddRaw` | `FCSharpEnvironment.cpp:96` | `:158` | ✅ | — |
| 6 | `FUnrealCSharpModuleDelegates::OnCSharpEnvironmentInitialize`（`FCSharpBind`） | `AddRaw` | `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:30` | `:38`（`Deinitialize()`，由 `~FCSharpBind` :23 调用） | ✅ | — |
| 7 | `FUnrealCSharpModuleDelegates::OnCSharpEnvironmentInitialize`（`FDynamicRegistry`） | `AddRaw` | `Source/UnrealCSharp/Private/Registry/FDynamicRegistry.cpp:39` | `:47` | ✅ | — |
| 8 | `FUnrealCSharpModuleDelegates::OnUnrealCSharpModuleActive/InActive`（`FUObjectListener`） | `AddRaw` ×2 | `Source/UnrealCSharp/Private/Listener/FUObjectListener.cpp:7,10` | `:18,23`（`~FUObjectListener`） | ✅ | — |
| 9 | `GUObjectArray::AddUObjectCreateListener/AddUObjectDeleteListener` | 直接注册 | `FUObjectListener.cpp:29,31` | `:36,38`（`OnUnrealCSharpModuleInActive`） | ✅ | — |
| 10 | `FUnrealCSharpCoreModuleDelegates::OnUnrealCSharpCoreModuleActive`（`FUnrealCSharpModule`） | `AddRaw` | `Source/UnrealCSharp/Private/UnrealCSharp.cpp:12` | `:31`（`ShutdownModule`） | ✅ | — |
| 11 | `…OnUnrealCSharpCoreModuleInActive`（`FUnrealCSharpModule`） | `AddRaw` | `UnrealCSharp.cpp:15` | `:25` | ✅ | — |
| 12 | `OnBeginGenerator` / `OnEndGenerator`（`FCSharpCompilerRunnable`） | `AddRaw` ×2 | `Source/Compiler/Private/FCSharpCompilerRunnable.cpp:25,28` | `:36,41` | ✅ | — |
| 13 | `GEditor->OnBlueprintCompiled()`（`FEditorListener`） | `AddRaw` | `FEditorListener.cpp:70` | `:107` | ✅（条件注册，`IsValid()` 判空） | — |
| 14 | `FEditorDelegates::PreBeginPIE/PrePIEEnded/EndPIE/CancelPIE` | `AddRaw` ×4 | `FEditorListener.cpp:46,48,50,52` | `:167,157,162,152` | ✅ | — |
| 15 | `OnBeginGenerator/OnEndGenerator/OnCompile`（`FEditorListener`） | `AddRaw` ×3 | `FEditorListener.cpp:54,57,60` | `:147,142,137` | ✅ | — |
| 16 | `IMainFrameModule::OnMainFrameCreationFinished()` | `AddRaw` | `FEditorListener.cpp:83` | `:132` | ✅ | — |
| 17 | `FSlateApplication::OnApplicationActivationStateChanged()` | `AddRaw`（**在回调里注册**） | `FEditorListener.cpp:397` | `:124`（带 `IsInitialized()` 判空） | ✅ | — |
| 18 | `FCoreDelegates::OnPostEngineInit`（`FEditorListener`） | `AddRaw` | `FEditorListener.cpp:39/42` | `:173/175` | ✅ | — |
| 19 | `IDirectoryWatcher::RegisterDirectoryChangedCallback_Handle` | `CreateRaw(this,…)` + 共享句柄 | `FEditorListener.cpp:91-96`（**在 `for` 循环内对 `OnDirectoryChangedDelegateHandle` 反复赋值**） | `:117-119`（同样循环 + 单一 `OnDirectoryChangedDelegateHandle`） | ❌ **仅配对最后一个目录** | 前面 N-1 个目录的回调**永不注销**，且 `CreateRaw(this)` 是裸 this → 见 F-LIFE-007 |
| 20 | `AssetRegistryModule.Get().OnFilesLoaded()` | `AddRaw`，**句柄丢弃** | `FEditorListener.cpp:79` | 无 | ❌ 未配对 | 委托永久驻留 + 悬挂 raw this，见 F-LIFE-001 |
| 21 | `AssetRegistryModule.Get().OnAssetAdded()` | `AddRaw`，**句柄丢弃** | `FEditorListener.cpp:344` | 无 | ❌ 未配对 | 同上 |
| 22 | `AssetRegistryModule.Get().OnAssetRemoved()` | `AddRaw`，**句柄丢弃** | `FEditorListener.cpp:346` | 无 | ❌ 未配对 | 同上 |
| 23 | `AssetRegistryModule.Get().OnAssetRenamed()` | `AddRaw`，**句柄丢弃** | `FEditorListener.cpp:348` | 无 | ❌ 未配对 | 同上 |
| 24 | `AssetRegistryModule.Get().OnAssetUpdatedOnDisk()` | `AddRaw`，**句柄丢弃** | `FEditorListener.cpp:350` | 无 | ❌ 未配对 | 同上 |
| 25 | `AssetRegistryModule.Get().OnFilesLoaded()`（**cook 命令**路径） | `AddLambda`，**句柄丢弃** | `Source/UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp:167` | 无 | ❌ 未配对 | lambda 捕获列表为空（`[]`），**无悬挂 `this`**，但绑定永久驻留；`ShutdownModule` 不清理 → `Generator(...)` 会在模块卸载后仍被触发，见 F-LIFE-013 |
| 26 | `FCoreDelegates::GetOnPostEngineInit()`（编辑器模块） | `AddRaw` | `UnrealCSharpEditor.cpp:77/80` | `:213/215` | ✅ | — |
| 27 | `UGameplayTagsManager::OnEditorRefreshGameplayTagTree` | `AddRaw` | `UnrealCSharpEditor.cpp:86` | `:207` | ✅（`IsRunningCommandlet()` 条件注册，解绑用 `IsValid()` 判空） | — |
| 28 | `FUnrealCSharpCoreModuleDelegates::OnEndGenerator`（BluePrint 工具栏） | `AddRaw` | `Source/UnrealCSharpEditor/Private/ToolBar/UnrealCSharpBlueprintToolBar.cpp:38` | `:46`（`Deinitialize()`） | ✅ | — |
| 29 | `FOnGetContent::CreateRaw(this, &FUnrealCSharpPlayToolBar::GeneratePlayToolBarMenu)` 挂进 `UToolMenus` | `CreateRaw` | `Source/UnrealCSharpEditor/Private/ToolBar/UnrealCSharpPlayToolBar.cpp:26` | 由 `UToolMenus::UnregisterOwner(this)`（`UnrealCSharpEditor.cpp:188`）连带移除 | ⚠️ 间接配对：`Initialize()`(:17) 注册，但 `FUnrealCSharpPlayToolBar::Deinitialize()`(:33-35) **是空函数** | 依赖 `UnregisterOwner` 的 owner 归属正确，见 F-LIFE-014 |
| 30 | `FCoreDelegates::OnEndFrame`（`FClassCollector`） | `AddStatic` | `Source/UnrealCSharpEditor/Private/NewClass/ClassCollector.cpp:142`（每次请求都 `AddStatic` 并覆盖同一句柄） | `:94`（析构，只 Remove 最后一个句柄） | ❌ **确认不配对（已由引擎源码判定）** | UE 的 `AddDelegateInstance` **不去重**且每次生成新句柄（`MulticastDelegateBase.h:277-291`、`DelegateInstancesImpl.h:38-42`）→ `OnEndFrame` 绑定逐帧累积，析构只清最后一条；但绑定的 `PopulateClassHierarchy` 是**静态成员函数**（`ClassCollector.h:42`），无悬垂 `this`，危害是重复执行全类遍历。见 F-LIFE-012 |
| 31 | AssetRegistry `OnFilesLoaded/OnAssetAdded/OnAssetRemoved/OnAssetRenamed`（`FClassCollector`） | `AddStatic` | `ClassCollector.cpp:53,56,59,62` | `:101,106,111,116` | ✅ | **同类代码的正确写法，可作为 F-LIFE-001 的反例证据** |
| 32 | `FCoreUObjectDelegates::ReloadCompleteDelegate` | `AddStatic` | `ClassCollector.cpp:43` | `:81` | ✅ | — |
| 33 | `GEditor->OnBlueprintCompiled()`（`FClassCollector`） | `AddStatic` | `ClassCollector.cpp:48` | `:88` | ✅ | — |
| 34 | `OnDynamicClassUpdated` / `OnEndGenerator`（`FClassCollector`） | `AddStatic` ×2 | `ClassCollector.cpp:36,40` | `:71,76` | ✅ | — |
| 35 | `OnDynamicClassUpdated` / `OnEndGenerator`（`UDynamicDataSource`） | `AddUObject` ×2 | `Source/UnrealCSharpEditor/Private/ContentBrowser/DynamicDataSource.cpp:63,66` | `:98,103`（`Shutdown()`） | ✅ | — |
| 36 | `UToolMenus::ExtendMenu("ContentBrowser.AddNewContextMenu")->AddDynamicSection` | `CreateLambda`（`WeakThis` 捕获，:78） | `DynamicDataSource.cpp:73-84` | 依赖 `UToolMenus` 生命周期；**未显式 UnregisterOwner** | ⚠️ 弱引用捕获避免悬挂，但 section 名字含 `GetName()` 会随对象变化 → 潜在重复 section | P3 |
| 37 | `FCoreDelegates::OnPreExit` / `IPluginManager::OnLoadingPhaseComplete`（`FEngineListener`） | `AddRaw` ×2 | `Source/UnrealCSharpCore/Private/Listener/FEngineListener.cpp:13,17` | `:24,29` | ✅ | — |
| 38 | C# 委托 → UE `FScriptDelegate`（`UDelegateHandler`） | `FScriptDelegate::BindUFunction(this, CSharpCallBack)` | `Source/UnrealCSharp/Private/Reflection/Delegate/DelegateHandler.cpp:60` | `UnBind()`(:72-78)/`Clear()`(:80-86)，由 `~FDelegateHelper`(`FDelegateHelper.cpp:17`) → `Deinitialize()`(:32) → `UDelegateHandler::Deinitialize()`(`DelegateHandler.cpp:32`) 路径回收 | ⚠️ 有解绑 API，但 **C# 侧若不调用对应 `UnBind`/`Clear`/`UnRegister`，`UDelegateHandler` 因 `AddToRoot` 永久存活并持有 `FScriptDelegate*`** | C# 对象被 GC 的语义由 `TWeakObjectPtr<UObject> Object`（`FDelegateWrapper.h:7`）保证不悬挂；**但 `Method` 裸指针会悬挂**，见 F-LIFE-002 |
| 39 | C# 多播委托 → UE `FMulticastScriptDelegate` | `FScriptDelegate::BindUFunction` + `Add` | `Source/UnrealCSharp/Private/Reflection/Property/DelegateProperty/FMulticastDelegatePropertyDescriptor.cpp:33-36` | `FMulticastDelegateHelper::Remove/RemoveAll/Clear`（`FMulticastDelegateHelper.cpp:73,81,89`；C# API 见 `FRegisterMulticastDelegate.cpp:195,196,197`） | ⚠️ API 齐全，同 #38 的托管侧依赖 | 同上 |
| 40 | `FAutoConsoleCommand` ×5（编辑器模块） | `MakeUnique` | `UnrealCSharpEditor.cpp:90,98,109,119,127` | 依赖 `TUniquePtr` 成员析构（`UnrealCSharpEditor.h:45-53`），`ShutdownModule` **未显式 Reset** | ⚠️ 隐式配对 | 模块对象析构时自动注销，可接受；但 `ShutdownModule` 之后到模块对象析构之间命令仍可用 |
| 41 | `FOnClicked::CreateSP(this, …)`（`DirectoryPathCustomization`） | `CreateSP` | `Source/UnrealCSharpEditor/Private/DetailCustomization/DirectoryPathCustomization.cpp:34` | Slate 生命周期管理 | ✅（`CreateSP` 持共享引用，安全） | — |
| 42 | `FExecuteAction::CreateStatic` / `FNewToolMenuDelegate::CreateLambda` | `CreateStatic`/`CreateLambda` | `DynamicNewClassContextMenu.cpp:54`、`DynamicDataSource.cpp:722,817,835` | 无捕获或无生命周期捕获 | ✅ | — |

**表 3 结论**：模块入口三处的 `StartupModule`↔`ShutdownModule` 反注册**基本齐全**（`UnrealCSharp.cpp`、`UnrealCSharpCore.cpp`（空实现，本就无注册）、`UnrealCSharpEditor.cpp` 的 `OnPostEngineInit`/`OnEditorRefreshGameplayTagTree`/`ToolMenus`/`PropertyEditor`/Style/Commands/Ticker/`TStrongObjectPtr` 全部配对）。**唯一但严重的缺口集中在 `FEditorListener` 的 AssetRegistry 与 DirectoryWatcher 绑定（#19-#24）**，以及 cook 路径的 `AddLambda`（#25）。所有 C# 委托桥接（#38/#39）**有解绑 API 但完全依赖托管侧主动调用**，插件没有在 C++ 侧做兜底清理。

### 表 4：共享指针所有权 / 所有权环判定表

grep 模式：`TSharedPtr`、`TSharedRef`、`TWeakPtr`、`TUniquePtr`、`MakeShared`、`MakeShareable`、`SharedThis`、`AsShared`（Source/ 命中 165 处；**`TSharedPtr<.*>(this)` 命中 0 处**）。

| # | 持有者 | 被持有者 | 持有方式 | 源码位置 | 释放路径 | 是否循环/悬垂 | 严重度 |
|---|---|---|---|---|---|---|---|
| 1 | `FUnrealCSharpEditorModule` | `FUnrealCSharpPlayToolBar` / `FUnrealCSharpBlueprintToolBar` | `TSharedPtr` ×2 | `UnrealCSharpEditor.h:37,39`；`MakeShared` 于 `UnrealCSharpEditor.cpp:72,74` | 模块对象析构（`ShutdownModule` 只调 `Deinitialize()`，不 `Reset()`） | ✅ 无环（工具栏不反向持有模块，工具栏内的 `CreateLambda` 通过 `FModuleManager::GetModulePtr` 弱获取模块） | — |
| 2 | `FUnrealCSharpPlayToolBar`/`BlueprintToolBar` | `FUICommandList` | `TSharedRef` | `UnrealCSharpPlayToolBar.h:21`（`MakeShared` 于 `.cpp:12`）、`UnrealCSharpBlueprintToolBar.h:35` | 工具栏析构 | ✅ 无环 | — |
| 3 | `UnrealCSharpEditorModule::DynamicDataSource` | `UDynamicDataSource` | `TStrongObjectPtr<UDynamicDataSource>` | `UnrealCSharpEditor.h:61`；构造 `.cpp:147`，`Reset()` `.cpp:229` | `ShutdownModule` 显式 `Reset()` | ✅ 正确配对（GC 保护 + 释放都有） | — |
| 4 | `UDynamicDataSource` | `FDynamicHierarchy` | `TSharedPtr<FDynamicHierarchy>` | `DynamicDataSource.h:146`；`MakeShareable` 于 `DynamicDataSource.cpp:743` | `UDynamicDataSource::Shutdown()` 里 `DynamicHierarchy.Reset()`（`.cpp:94`） | ✅ 无环（`FDynamicHierarchy::Root` 是 `TSharedPtr<FDynamicHierarchyNode>`，节点只持有子节点，树向下，父指针缺失 → 无环） | — |
| 5 | `FDynamicHierarchy` 节点 | 子节点 `TMap<FName, TSharedPtr<FDynamicHierarchyNode>>` | `TSharedPtr` | `DynamicHierarchy.h:17` | 随 `DynamicHierarchy.Reset()` 级联析构 | ✅ 无环（无父指针） | — |
| 6 | `FFunctionDescriptor`（及派生 `FCSharpDelegateDescriptor`/`FCSharpFunctionDescriptor`） | `FFunctionParamBufferAllocator` | `TSharedPtr` | `FFunctionDescriptor.h:33`；`MakeShared` 于 `FFunctionParamBufferAllocator.h:65,70` | `~FFunctionDescriptor` | ✅ 无环（allocator 不反向持有 descriptor） | — |
| 7 | `FFunctionDescriptor` | `Function` | `TWeakObjectPtr<UFunction>` | `FFunctionDescriptor.h:23` | — | ✅ weak，**这是插件里少见的正确用法** | — |
| 8 | `FCSharpCompilerRunnable` | `SNotificationItem` | `TSharedPtr` | `FCSharpCompilerRunnable.h:76` | 未发现 `Reset()`（见 §7 未覆盖项） | ⚠️ 若通知项在编译期间反复创建并赋值，旧项由 Slate 自行管理，无环但可能持有过期项 | P3 |
| 9 | `SDynamicClassViewer` | `FClassCollector` | **`static TSharedPtr<FClassCollector>`** | `SDynamicClassViewer.cpp:7`；`MakeShared` 于 `:19`（`if (ClassCollector == nullptr)` 惰性单例） | **永不 Reset**（静态成员，直到进程结束） | ❌ **进程级单例泄漏**：`FClassCollector` 的静态共享引用使其实例及其订阅的 8 个全局委托绑定无法随编辑器模块卸载而释放 → 与 F-LIFE-001 叠加 | P1 |
| 10 | `FClassCollector` | `AllNodes` / `NodesSet` | **`static TArray/TSet<TSharedPtr<FDynamicClassViewerNode>>`** | `ClassCollector.cpp:9,11` | 静态容器，**无清空点**（仅 `RefreshAllNodes`/`PopulateClass*` 内增删元素） | ⚠️ 节点数量随资产增长；`TSharedPtr` 引用计数正确，无环 | P2 |
| 11 | Slate 控件（`SCompileProgressDialog`） | `StatusText`/`ElapsedText` | `TSharedPtr<STextBlock>` | `SCompileProgressDialog.h:22,24` | Slate 面板析构 | ✅ 无环（标准 Slate 模式） | — |
| 12 | `FDirectoryPathCustomization` | `BrowseButton` | `TSharedPtr<SButton>` | `DirectoryPathCustomization.h:31`；`SAssignNew` 于 `.cpp:27` | 自定义化实例析构 | ✅ 无环 | — |
| 13 | `FUnrealCSharpEditorStyle` | `FSlateStyleSet` | **`static TSharedPtr<FSlateStyleSet> StyleInstance`** | `UnrealCSharpEditorStyle.h:30`；`MakeShareable` 于 `.cpp:49` | `FUnrealCSharpEditorStyle::Shutdown()`（配对 `Initialize()`） | ✅ 配对 | — |
| 14 | `SDynamicNewClassDialog` 系列 | 各种 `TSharedPtr<T>` 成员 | `TSharedPtr` | `SDynamicNewClassDialog.h:104-138` 等 | 对话框析构 | ✅ 无环（`SelectedParentClassInfo` 等均为叶子数据节点） | — |
| 15 | `FGameplayTagGenerator` | `TUniquePtr<FGameplayTagTreeNode>` / `TSharedPtr<FGameplayTagNode>` | `TUniquePtr` / `TSharedPtr` | `FGameplayTagGenerator.cpp:21,252` | 局部变量，函数结束释放 | ✅ 无环 | — |

**共享指针结论（"未发现"清单）**：
- **未发现** `TSharedPtr<...>(this)` 形式的裸 `this` 二次引用计数（grep 模式 `TSharedPtr<.*>\(this\)` 命中 **0**）。所有需要 `this` 的共享指针场景都走了 Slate 的 `SNew`/`SAssignNew`/`CreateSP`，或 `AsShared()`（`SDynamicNewClassDialog.cpp:790,975`）。
- **未发现** 所有权环（`A`↔`B` 互持 `TSharedPtr`）：`FDynamicHierarchy` 树无父指针、工具栏与模块为单向、descriptor↔allocator 为单向。
- **未发现** `CreateUObject`（命中 0）；`CreateRaw` 出现在 3 处（`UnrealCSharpPlayToolBar.cpp:26`、`FEditorListener.cpp:93`、`DetailCustomization`），其中 `FEditorListener.cpp:93` 的 `CreateRaw(this)` 与表 3 #19 的句柄覆盖问题叠加。
- **发现 1 处进程级 `static TSharedPtr` 单例**（`SDynamicClassViewer::ClassCollector`，表 4 #9）和 **2 处进程级 `static` 共享容器**（#10、`FClassCollector::AllNodes/NodesSet`），它们在编辑器模块卸载后**不会释放**。

---

## 4. 发现清单

### P0

#### [F-LIFE-001] `FEditorListener` 在资产注册表上的 5 个 Raw 委托从未解绑 → 编辑器模块卸载后悬垂调用

- **类别**: 内存/资源泄漏 | 未定义行为
- **严重度**: **P0**(崩溃/数据损坏)
- **复核结论**: 部分确认（偏差：5 处"注册无解绑"全部复核成立，但 UAF 的触发窗口比原文描述窄——只在"模块已卸载、AssetRegistry 尚未销毁"这一段；正常退出路径上未观测到资产事件，故 UAF 属潜伏后果而非每次退出必炸；确定性委托泄漏成立）
- **可达性**: 活跃（绑定点在 `StartupModule` 必然执行；悬垂窗口在每次编辑器退出时必然出现）
- **复核证据**: `FEditorListener.cpp:79` / `:344` / `:346` / `:348` / `:350` 五处 `AddRaw` 逐字确认；同文件 `101-179` 析构 12 处 `Remove` 逐行确认，**无任何 AssetRegistry 解绑**（工具 grep `OnFilesLoaded|OnAssetAdded|OnAssetRemoved|OnAssetRenamed|OnAssetUpdatedOnDisk` 命中 10 处 = 5 处 AddRaw + 5 处函数定义/实现，解绑 0 处）。宿主确认：`UnrealCSharpEditor.h:55` `FEditorListener EditorListener;`（按值成员）。引擎侧证据（原文缺失）：① 模块**会**在退出时被卸载 —— `Runtime/Core/Public/Modules/ModuleInterface.h:74-77` `SupportsAutomaticShutdown()` 默认 `return true`，本模块未覆写；② 卸载顺序为**后加载先卸载** —— `Runtime/Core/Private/Modules/ModuleManager.cpp:1224-1227`（`return LoadOrder > Other.LoadOrder; //intentionally backwards`）、`:1256-1262`，而 AssetRegistry 早于本插件模块加载 → `FEditorListener` 先死、AssetRegistry 后死；③ 但 AssetRegistry 的资产事件直到 `UAssetRegistryImpl::FinishDestroy()` 才 `Clear()` —— `Runtime/AssetRegistry/Private/AssetRegistry.cpp:1858-1873`（`AssetAddedEvent.Clear(); ... FileLoadedEvent.Clear();`），故悬垂条目在该窗口内确实存在于活着的多播委托中。
- **级别变动**: 无（P0 维持：确定的裸 `this` 悬垂注册 + 同名同源报告 `04-…/02` F-ED2-001 同为 P0，跨报告保持一致；且原文标题所述"5 个委托从未解绑"经复核完全成立）
- **文件**: `Source/UnrealCSharpEditor/Private/Listener/FEditorListener.cpp:79`、`:344`、`:346`、`:348`、`:350`
- **函数**: `FEditorListener::FEditorListener()` / `FEditorListener::OnFilesLoaded()` / `~FEditorListener()`
- **置信度**: 高（原文标为"未验证"的模块卸载顺序已回引擎源码核实：`ModuleInterface.h:74-77` + `ModuleManager.cpp:1224-1227`，见复核证据；唯一仍属推断的是"悬垂窗口内是否真有资产事件广播"）

**现状（代码事实）**
```cpp
// FEditorListener.cpp:76-79  构造函数：句柄未保存
const auto& AssetRegistryModule = FModuleManager::LoadModuleChecked<
    FAssetRegistryModule>(TEXT("AssetRegistry"));

AssetRegistryModule.Get().OnFilesLoaded().AddRaw(this, &FEditorListener::OnFilesLoaded);   // ← 返回值被丢弃

// FEditorListener.cpp:340-351  OnFilesLoaded 里再挂 4 个，同样全部丢弃句柄
void FEditorListener::OnFilesLoaded()
{
    const auto& AssetRegistryModule = FModuleManager::LoadModuleChecked<FAssetRegistryModule>(TEXT("AssetRegistry"));

    AssetRegistryModule.Get().OnAssetAdded().AddRaw(this, &FEditorListener::OnAssetAdded);            // :344
    AssetRegistryModule.Get().OnAssetRemoved().AddRaw(this, &FEditorListener::OnAssetRemoved);        // :346
    AssetRegistryModule.Get().OnAssetRenamed().AddRaw(this, &FEditorListener::OnAssetRenamed);        // :348
    AssetRegistryModule.Get().OnAssetUpdatedOnDisk().AddRaw(this, &FEditorListener::OnAssetUpdatedOnDisk); // :350
}
```
析构函数 `FEditorListener.cpp:101-179` 逐个 `Remove` 了 `OnBlueprintCompiled`(:107)、`OnDirectoryChanged`(:117)、`OnApplicationActivationStateChanged`(:124)、`OnMainFrameCreationFinished`(:132)、`OnCompile`(:137)、`OnEndGenerator`(:142)、`OnBeginGenerator`(:147)、`OnCancelPIE`(:152)、`OnPrePIEEnded`(:157)、`OnEndPIE`(:162)、`OnPreBeginPIE`(:167)、`OnPostEngineInit`(:173) —— **唯独上面 5 个 AssetRegistry 委托没有对应 Remove；全文件 grep `OnAssetAdded|OnAssetRemoved|OnAssetRenamed|OnAssetUpdatedOnDisk|OnFilesLoaded` 的解绑命中数为 0。**

**调用上下文**
`FEditorListener` 是**按值成员**：`Source/UnrealCSharpEditor/Public/UnrealCSharpEditor.h:55` `FEditorListener EditorListener;`，其宿主是 `FUnrealCSharpEditorModule`，即 `ShutdownModule()`（`UnrealCSharpEditor.cpp:183`）之后随模块对象析构。绑定点在 `FEditorListener` 构造（编辑器模块 `StartupModule` 期间创建模块对象时），触发点是 AssetRegistry 的任意资产事件（导入/删除/重命名/另存）。

**问题**
`AddRaw(this, ...)` 保存的是**裸 `this`**，没有 weak 语义。`FEditorListener` 析构后，这 5 个 `TMulticastDelegate` 里仍留有指向已释放对象的条目。触发路径：
1. 编辑器运行 → `FEditorListener` 构造，5 个 raw 委托挂到 `AssetRegistry`；
2. 模块被卸载（Live Coding / 编辑器模块热重载 / 编辑器退出阶段早于 AssetRegistry 关闭）；
3. 之后任意资产事件（例如 `OnAssetAdded`）→ AssetRegistry 遍历委托列表调用已析构的 `FEditorListener::OnAssetAdded` → `OnAssetChanged` → 读 `bIsPIEPlaying` / `bIsGenerating`（已释放内存）→ **use-after-free 崩溃**。
即使不崩溃，这也是确定性的**永久委托泄漏**（进程内模块反复加载会线性累积）。

**建议**
```cpp
// 头文件增加 5 个句柄
FDelegateHandle OnFilesLoadedDelegateHandle;
FDelegateHandle OnAssetAddedDelegateHandle;
FDelegateHandle OnAssetRemovedDelegateHandle;
FDelegateHandle OnAssetRenamedDelegateHandle;
FDelegateHandle OnAssetUpdatedOnDiskDelegateHandle;

// 构造/OnFilesLoaded 里保存
OnFilesLoadedDelegateHandle = AssetRegistryModule.Get().OnFilesLoaded().AddRaw(this, &FEditorListener::OnFilesLoaded);
// ... 其余 4 个同理
```
并在 `~FEditorListener()` 中按 `IsValid()` 逐一 `Remove`；`OnAssetAdded` 等 4 个只在 `OnFilesLoaded` 里注册，故解绑前需先判 `FModuleManager::Get().IsModuleLoaded("AssetRegistry")`。更稳的做法是全部改用 `AddSP(AsShared(), ...)` 或 `AddWeakLambda`，但该类不是 `TSharedFromThis`，最小改动仍是补句柄。

**验证方式**
`grep -n "OnAssetAdded\|OnAssetRemoved\|OnAssetRenamed\|OnAssetUpdatedOnDisk\|OnFilesLoaded" Source/UnrealCSharpEditor/Private/Listener/FEditorListener.cpp` → 只有 AddRaw 无 Remove；或在 `~FEditorListener` 里加 `ensureAlwaysMsgf(!AssetRegistryModule.Get().OnAssetAdded().IsBound(), ...)` 复现。

#### [F-LIFE-002] `FDelegateWrapper::Method` 是裸 `FMethodReflection*`，域重载后被 `delete` → UE 委托回调时 UAF —— **已证伪：非缺陷，请从排期移除**

- **类别**: 未定义行为 | 内存/资源泄漏
- **严重度**: **撤销（非缺陷）**
- **复核结论**: 证伪（撤销，非缺陷）—— 原文的"撤销"结论**经独立复核成立**，且原文缺"为什么撤销"的证据，现补齐（见下）
- **可达性**: 不可达（该 UAF 路径被 `DelegateDescriptor != nullptr` 守卫切断）
- **复核证据**: 原文声称的悬垂确实存在——`FDelegateWrapper.h:7,9`（`TWeakObjectPtr<UObject> Object;` / `FMethodReflection* Method;` 无 UPROPERTY）、`DelegateHandler.cpp:10` 解引用 `DelegateWrapper.Method`、`FReflectionRegistry.cpp:871-874` 无条件 `delete Class`。**但撤销理由成立**：① `DelegateHandler.cpp:8` 在同一表达式前有 `if (DelegateDescriptor != nullptr)` 守卫，而 `DelegateDescriptor` 由 `UDelegateHandler::Initialize`（`DelegateHandler.cpp:29`）创建、由 `UDelegateHandler::Deinitialize`（`:46-51`）`delete` 并置空；② `FDelegateHelper::Deinitialize()`（`FDelegateHelper.cpp:32-42`）先调 `DelegateHandler->Deinitialize()` 再 `RemoveFromRoot()`——即 **handler 一旦脱离 helper，`DelegateDescriptor` 必为 nullptr，`:10` 永不执行**；③ 销毁顺序保证 helper 先于反射注册表消失：`FCSharpEnvironment.cpp:207-212` `delete DelegateRegistry`（其析构 `FDelegateRegistry.cpp:9-12 → :18-55` 逐个 `delete` 每个 `FDelegateHelper`）**早于** `:263-268` `delete Domain`（→ `~FDomain` → `FCoreCLRDomain/LeanCLR/Mono` 的 `UnloadAssembly()` → `FReflectionRegistry::Get().Deinitialize()`，见 `FCoreCLRDomain.cpp:137,242`、`FLeanCLRDomain.cpp:96,808`、`FMonoDomain.cpp:181,430`）；④ 所有 `FDelegateHelper` 必然登记在注册表内 —— `FDelegateRegistry.inl:38` 与 `:50` 两个 `AddReference` 重载都写入 `ManagedHandle2Value`，无"创建未登记"路径；⑤ `UnloadAssembly` 全插件只有 3 个调用点，全部位于各域的 `Deinitialize()` 内（grep `UnloadAssembly` 命中 10 处，其中调用 4 处：`FMonoDomain.cpp:181`、`FLeanCLRDomain.cpp:79,96`、`FCoreCLRDomain.cpp:137`），不存在"绕过 DelegateRegistry 直接卸域"的旁路。残留事实（不构成缺陷）：`FDelegateWrapper.h:14` 的 `operator==` 会比较已释放的 `Method` 指针值，属指针值比较、不解引用。
- **级别变动**: 无（维持 `撤销（非缺陷）`）
- **文件**: `Source/UnrealCSharp/Public/Reflection/Delegate/FDelegateWrapper.h:9`、`Source/UnrealCSharp/Public/Reflection/Delegate/DelegateHandler.h:158`、`Source/UnrealCSharp/Private/Reflection/Delegate/DelegateHandler.cpp:10,64`
- **函数**: `FDelegateWrapper`、`UDelegateHandler::ProcessEvent(UFunction*, void*)`、`UDelegateHandler::Bind(UObject*, FMethodReflection*)`
- **置信度**: 高（撤销所依赖的两条事实——`DelegateDescriptor != nullptr` 守卫与"DelegateRegistry 先于 Domain 销毁"——均为直读源码，不再是推断）

**现状（代码事实）**
```cpp
// FDelegateWrapper.h:5-10
struct FDelegateWrapper
{
    TWeakObjectPtr<UObject> Object;   // 安全（weak）

    FMethodReflection* Method;        // ← 裸指针，无 UPROPERTY、无 owner 语义
};

// DelegateHandler.h:151-158
private:
    bool bNeedFree;
    FScriptDelegate* ScriptDelegate;
    FCSharpDelegateDescriptor* DelegateDescriptor;
    FDelegateWrapper DelegateWrapper;   // ← 内嵌持有裸 Method

// DelegateHandler.cpp:4-17  回调时直接解引用
void UDelegateHandler::ProcessEvent(UFunction* Function, void* Parms)
{
    if (Function != nullptr && Function->GetName() == FUNCTION_CSHARP_CALLBACK)
    {
        if (DelegateDescriptor != nullptr)
        {
            DelegateDescriptor->CallDelegate(DelegateWrapper.Object.Get(), DelegateWrapper.Method, Parms);  // :10
        }
    }
    ...
}

// DelegateHandler.cpp:54-65  绑定点写入
void UDelegateHandler::Bind(UObject* InObject, FMethodReflection* InMethod)
{
    if (ScriptDelegate != nullptr) { if (!ScriptDelegate->IsBound()) { ScriptDelegate->BindUFunction(this, *FUNCTION_CSHARP_CALLBACK); } }
    DelegateWrapper = {InObject, InMethod};   // :64
}
```
`FMethodReflection` 由 `FClassReflection` 拥有，`FClassReflection` 由 `FReflectionRegistry` 拥有：
```cpp
// Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp:867-877
void FReflectionRegistry::Deinitialize()
{
    Field2Class.Empty();
    for (const auto& [PLACEHOLDER, Class] : FullName2Class) { delete Class; }   // ← 全部 delete
    FullName2Class.Empty();
}
// 域卸载时被调用：
// FMonoDomain.cpp:430 / FCoreCLRDomain.cpp:242 / FLeanCLRDomain.cpp:808
FReflectionRegistry::Get().Deinitialize();
```
而 `UDelegateHandler` 本体被 `AddToRoot()`（`FDelegateHelper.cpp:24`）保活，且其 `ScriptDelegate` 可能挂在宿主 UObject 的委托属性上（`FDelegatePropertyDescriptor.cpp:39`），因此**处理器可以比反射注册表活得更久**。

**调用上下文**
注册路径：C# `FDelegate.Register` → `FRegisterDelegate.cpp:18` `FCSharpBind::Bind<FDelegateHelper>`；绑定路径：C# `FDelegate.Bind` → `FRegisterDelegate.cpp:44` `DelegateHelper->Bind(FoundObject, FoundMethod)` → `UDelegateHandler::Bind`（`DelegateHandler.cpp:64`）。触发路径：域重载后任何一次 UE 侧 `Broadcast`/`Execute` → `ProcessEvent`（`DelegateHandler.cpp:10`）→ `CallDelegate`（`FCSharpDelegateDescriptor.cpp:10-26`）→ `Invoke(InMethod, ...)` 解引用已释放的 `FMethodReflection`。

**问题**
`FClassReflection` 及其 `FMethodReflection` 在 `FReflectionRegistry::Deinitialize()` 中被无条件 `delete`，但**域重载不清理 `FDelegateRegistry` 中的 `FDelegateHelper`**（`FDelegateRegistry` 只在 `FCSharpEnvironment::Deinitialize()`（`FCSharpEnvironment.cpp:207-212`）里被 delete，而后者由 `OnUnrealCSharpModuleInActive` → `FUnrealCSharpCoreModule::Deactivate()` 触发，**与域卸载不是同一条路径**）。于是 `UDelegateHandler::DelegateWrapper.Method` 悬垂，任何后续委托触发都是读写已释放堆内存。

**建议**
1. `FDelegateWrapper::Method` 改为弱引用语义：保存 `FMethodReflection` 的稳定标识（`FName`+`uint32 Hash`）或引入 `TWeakPtr<FMethodReflection>`（需让 `FClassReflection` 用 `TSharedPtr` 管理 method）。
2. 在任何域卸载路径（`UnloadAssembly()`）中，先遍历并清空 `FDelegateRegistry`（调用 `FCSharpEnvironment::GetEnvironment().GetRegistry<FDelegateRegistry>()->Deinitialize()`），再 `FReflectionRegistry::Get().Deinitialize()`。
3. 防御性最小改动：`UDelegateHandler` 增加 `void Invalidate();`，在域卸载时置 `DelegateWrapper.Method = nullptr; DelegateWrapper.Object = nullptr;` 并让 `ProcessEvent` 判空（当前 `DelegateHandler.cpp:8` 只判了 `DelegateDescriptor`，没判 `DelegateWrapper.Method`）。

**验证方式**
grep `DelegateWrapper.Method` 与 `FReflectionRegistry::Deinitialize` 的调用点；或加 `check(DelegateWrapper.Method != nullptr)` 后执行一次 C# 编译重载（触发域卸载）再让该委托广播。

#### [F-LIFE-003] `FDelegatePropertyDescriptor::Set` / `FMulticastDelegatePropertyDescriptor::Set` 未判空即解引用 `SrcDelegateHelper`

- **类别**: Bug | 未定义行为
- **严重度**: **P0**(崩溃/数据损坏)（空指针解引用）
- **复核结论**: 确认 **后续处置：判定升级为「确认 · 已修复（`492ce5f7`）」—— 两个 `Set` 均已把 `GetDelegate<>` 解析提到 `InitializeValue` 之前并在解引用前判空；与 `01-…/07b` 的 `F-DEL-003` 同源，本次一并修复（非两处独立改动）**（见下方「处置（已执行）」）
- **可达性**: 活跃
- **复核证据**: 两个 `Set` 逐字确认无判空：`FDelegatePropertyDescriptor.cpp:20-31`（`:24` 取 helper → `:30` `DestScriptDelegate->BindUFunction(SrcDelegateHelper->GetUObject(), SrcDelegateHelper->GetFunctionName());`）；`FMulticastDelegatePropertyDescriptor.cpp:20-37`（`:24-25` 取 helper → `:33-34` 同样直接解引用）。解引用后果已核到实现：`FDelegateHelper::GetUObject()`（`FDelegateHelper.cpp:73-76`）读成员 `DelegateHandler`，在 `this == nullptr` 时即空指针访问（非虚函数、无内联空判）。`GetDelegate` 可返回 nullptr 已核实：`FDelegateRegistry.inl:19-25`（`return FoundValue != nullptr ? *FoundValue : nullptr;`）。对照 `NewRef` 的判空写法同样核实：`FDelegatePropertyDescriptor.cpp:37` `if (!IManagedHandleIsValid(Object))`。异步注销点核实：`FRegisterDelegate.cpp:21-28`（`AsyncTask(ENamedThreads::GameThread, ...)` 内 `RemoveDelegateReference<FDelegateHelper>`）。
- **级别变动**: 无（P0 维持：空指针解引用，且存在"托管侧把已 `UnRegister`/`InvalidManagedHandle` 的句柄写回 delegate 属性"的活跃触发路径）
- **文件**: `Source/UnrealCSharp/Private/Reflection/Property/DelegateProperty/FDelegatePropertyDescriptor.cpp:20-31`（`Set`）、`Source/UnrealCSharp/Private/Reflection/Property/DelegateProperty/FMulticastDelegatePropertyDescriptor.cpp:20-37`（`Set`）
- **函数**: `FDelegatePropertyDescriptor::Set(void*, void*)`、`FMulticastDelegatePropertyDescriptor::Set(void*, void*)`
- **置信度**: 高
- **处置（已执行）**: ✅ **已修复**（修复提交 `492ce5f7` "Null Validation"）。两个 `Set` 都先把 `GetDelegate<>` 的解析提到 `Property->InitializeValue(Dest)` **之前**（`FDelegatePropertyDescriptor::Set` 取 `FDelegateHelper`、`FMulticastDelegatePropertyDescriptor::Set` 取 `FMulticastDelegateHelper`），并在解引用前判空：只有 helper 命中时才走 `DestScriptDelegate->BindUFunction(SrcDelegateHelper->GetUObject(), SrcDelegateHelper->GetFunctionName())` 与 `MulticastScriptDelegate->Add(ScriptDelegate)`，两个 `->GetUObject()`/`GetFunctionName()` 不再在空指针上解引用。**该条与 `01-UnrealCSharp运行时/07b-委托Handler与OptionalHelper.md` 的 `F-DEL-003` 同源** —— 两者是**同一处代码的两次登记**，本次的代码改动只有这一处、**一并修复（非两处独立改动）**。原文「建议」的判空意图**已采纳**，形态从 `if (SrcDelegateHelper == nullptr) { return; }` 改为 `if (SrcDelegateHelper != nullptr) { ... }` 正向包裹（与同文件 `NewRef` 的 `if (!IManagedHandleIsValid(Object))` 判空风格同族）。未做的部分：原文建议里的 `UE_LOG(LogUnrealCSharp, Warning, ...)` 日志**未加**（未命中是静默跳过赋值）；原文结尾"更深一层"的两条——把 `UnRegisterImplementation` 的异步 `AsyncTask` 改为 `check(IsInGameThread())`、以及在 C# 侧保证句柄不复用——**均未做**。

**现状（代码事实）**
```cpp
// FDelegatePropertyDescriptor.cpp:20-31
void FDelegatePropertyDescriptor::Set(void* Src, void* Dest) const
{
    const auto SrcManagedHandle = *static_cast<IManagedHandle*>(Src);

    const auto SrcDelegateHelper = FCSharpEnvironment::GetEnvironment().GetDelegate<FDelegateHelper>(SrcManagedHandle);
    // ← 无 if (SrcDelegateHelper == nullptr) 判断

    Property->InitializeValue(Dest);

    const auto DestScriptDelegate = Property->GetPropertyValuePtr(Dest);

    DestScriptDelegate->BindUFunction(SrcDelegateHelper->GetUObject(), SrcDelegateHelper->GetFunctionName());  // :30 解引用
}
```
`FMulticastDelegatePropertyDescriptor.cpp:24-34` 完全相同的形态（`:33-34` 解引用）。对照：`FDelegateRegistry.inl:24` 的 `GetDelegate` 明确"找不到即返回 `nullptr`"（`return FoundValue != nullptr ? *FoundValue : nullptr;`）。

**调用上下文**
`Set` 由 C# 侧属性写入路径调用（属性 setter → `FPropertyDescriptor::Set`）。`SrcManagedHandle` 由 `FDelegatePropertyDescriptor::Get`（`:7/:12/:17`）产出；若 C# 侧把一个**已被 `FDelegate.UnRegister` 释放**或**从未 Register** 的委托句柄赋给 UE 委托属性/参数，`GetDelegate` 返回 `nullptr`。

**问题**
这是插件里唯一一处"注册表查询结果直接解引用"而未判空的地方（同一批 descriptor 中 `NewRef` 用了 `IManagedHandleIsValid` 检查，`Set` 却裸用）。触发路径：`FDelegate.UnRegister(handle)`（`FRegisterDelegate.cpp:21-28`，**异步** `AsyncTask(ENamedThreads::GameThread, ...)` 里 `RemoveDelegateReference` → `delete FDelegateHelper` + `GCHandle_Free`）之后，托管侧仍持有旧句柄并再次赋值 → `Set` 拿到 `nullptr` → 空指针解引用崩溃。

**建议**
```cpp
if (SrcDelegateHelper == nullptr)
{
    return;   // 或 UE_LOG(LogUnrealCSharp, Warning, ...) 后 return
}
```
两个 descriptor 都要加。更深一层：`UnRegisterImplementation` 用了异步 `AsyncTask` 而非 `check(IsInGameThread())`，在非游戏线程调用时句柄会在任意时刻失效，建议同时在 C# 侧保证句柄不复用。

**验证方式**
`grep -n "GetDelegate<FDelegateHelper>\|GetDelegate<\s*FMulticastDelegateHelper>" Source/UnrealCSharp/Private/Reflection/Property/DelegateProperty/` 后逐处检查判空；或写单测：C# 侧 `var h = ...Register(); ...UnRegister(h); obj.SomeDelegate = <null-ish handle>;`。

#### [F-LIFE-010] `FDynamicGeneratorCore` 的 6 个 `static TArray<FClassReflection*>` 元数据缓存跨域重载悬垂 → 第二次热重载即 UAF

- **类别**: 未定义行为 | 内存/资源泄漏
- **严重度**: **P1**(崩溃/数据损坏)
- **复核结论**: 部分确认（偏差：机制与 6 个静态数组全部复核成立；但原文"UAF → 必崩溃"的表述过强，"P0→P1"的降级由补齐的证据支撑、**成立但理由需改写**——见"复核证据"）
- **可达性**: 活跃（自同一会话内的第 2 代反射注册表起生效；单次会话只编译一次 C# 则不受影响）
- **复核证据**: 6 个函数签名逐行确认（工具 grep `^const TArray<FClassReflection\*>& FDynamicGeneratorCore::` → 命中 6 处：`:1036`、`:1067`、`:1082`、`:1095`、`:1108`、`:1201`，与原文完全一致）；6 个 `static TArray<FClassReflection*>` 定义行确认：`1040 / 1071 / 1086 / 1099 / 1112 / 1205`。消费点确认：`FDynamicGeneratorCore.cpp:792`、`:799`、`:806-808`、`:855`、`:874`。`delete` 侧确认：`FReflectionRegistry.cpp:867-877`。**降级理由**：`SetFieldMetaData`（`FDynamicGeneratorCore.cpp:775-788`）对每个属性先调 `InReflection->HasAttribute(MetaDataAttribute)`，而 `FReflection::HasAttribute` 是 `return Attributes.Contains(InAttribute);`（`FReflection.cpp:18-21`）、`GetAttributeValue` 是 `AttributeValues.Find(InAttribute)`（`:23-32`）——**两者都是"按指针值做 TSet/TMap 查询"，不解引用该属性对象**。因此悬垂指针的主要后果是"匹配到错误的/匹配不到属性"→ 元数据静默错配（数据损坏），而**只有当 `HasAttribute` 意外命中（新分配对象恰好复用同一地址）时**才会走到 `FDynamicGeneratorCore.cpp:782` 的 `MetaDataAttribute->GetName()` 真正解引用已释放对象。据此 P1（功能错误/数据损坏隐患）比 P0 更贴合，**维持原文的 P0→P1 校正**。
- **级别变动**: 无（维持 P0→P1；已补齐降级理由，原文未给理由）
- **文件**: `Source/UnrealCSharpCore/Private/Dynamic/FDynamicGeneratorCore.cpp:1036-1210`（6 个函数：`:1036`、`:1067`、`:1082`、`:1095`、`:1108`、`:1201`）
- **函数**: `FDynamicGeneratorCore::GetClassMetaDataAttributes()` 等 6 个
- **置信度**: 高（初始化只发生一次是 C++ 语言保证；`FReflectionRegistry::Deinitialize()` 无条件 `delete` 是已读代码事实）

**现状（代码事实）**
```cpp
// FDynamicGeneratorCore.cpp:1036-1065
const TArray<FClassReflection*>& FDynamicGeneratorCore::GetClassMetaDataAttributes()
{
    static auto& ReflectionRegistry = FReflectionRegistry::Get();       // :1038  一次性静态引用

    static TArray<FClassReflection*> ClassMetaDataAttributes = {        // :1040  一次性初始化，永不重取
        ReflectionRegistry.GetHideCategoriesAttributeClass(),
        ReflectionRegistry.GetToolTipAttributeClass(),
        // …共 21 个 FClassReflection*（:1041-1061）
    };

    return ClassMetaDataAttributes;
}
```
同样形态的还有 `:1067`（Struct，:1071）、`:1082`（Enum，:1086）、`:1095`（Interface，:1099）、`:1108`（Property，`:1112`）、`:1201`（Function，`:1205`）。

这些数组被热路径直接消费：
```cpp
// FDynamicGeneratorCore.cpp:792
SetFieldMetaData(InProperty,   GetPropertyMetaDataAttributes(),   InReflection, ...);
// :799
SetFieldMetaData(InFunction,   GetFunctionMetaDataAttributes(),   InReflection, ...);
// :807-808
SetFieldMetaData(InClass, IsInterface ? GetInterfaceMetaDataAttributes() : GetClassMetaDataAttributes(), ...);
// :855
SetFieldMetaData(InScriptStruct, GetStructMetaDataAttributes(), ...);
// :874
SetFieldMetaData(InEnum,         GetEnumMetaDataAttributes(), ...);
```
而 `FReflectionRegistry::Deinitialize()`（`FReflectionRegistry.cpp:867-877`）：
```cpp
void FReflectionRegistry::Deinitialize()
{
    Field2Class.Empty();
    for (const auto& [PLACEHOLDER, Class] : FullName2Class) { delete Class; }   // :873  全部 delete
    FullName2Class.Empty();
}
```
调用点：`FMonoDomain.cpp:430`、`FCoreCLRDomain.cpp:242`、`FLeanCLRDomain.cpp:808`（域卸载）。

**调用上下文**
完整热重载链路（已逐跳核对）：
1. C# 编译成功 → `FCSharpCompilerRunnable.cpp:225` `UnrealCSharpCoreModule.Deactivate()`；
2. `UnrealCSharpCore.cpp:29-37` `Deactivate()` → 广播 `OnUnrealCSharpCoreModuleInActive`；
3. `UnrealCSharp.cpp:50-53` → 广播 `OnUnrealCSharpModuleInActive`；
4. `FCSharpEnvironment.cpp:332-335` `OnUnrealCSharpModuleInActive()` → `Deinitialize()`（`:147-269`）：按 Optional→Binding→String→Multi→**Delegate**→Container→Struct→Object→Reference→Class→CSharpBind→Dynamic→**最后 `delete Domain`**（`:263-268`）的顺序销毁；
5. `~FDomain()`（`FDomain.cpp:15-18`）→ `Deinitialize()` → `FScriptDomainFactory::Destroy`（`FScriptDomainFactory.cpp:47-59`）→ `IScriptDomain::Deinitialize()` → `FCoreCLRDomain::Deinitialize()`（`:128-162`）→ `UnloadAssembly()`（`:240-270`）→ `FReflectionRegistry::Get().Deinitialize()`（`:242`，**此处 `delete` 全部 `FClassReflection`**）；
6. `UnrealCSharpCoreModule.Activate()`（`FCSharpCompilerRunnable.cpp:229`）→ `FCSharpEnvironment::Initialize()`（`:57-145`）→ 新建 `Domain`/各 registry；`FCoreCLRDomain::Initialize` → `FReflectionRegistry::Get().Initialize()`（`FReflectionRegistry.cpp:21-865`）**重新创建一套全新的 `FClassReflection` 并写入新的 `FullName2Class`**；
7. 随后 `FDynamicGenerator::Generator()`（`FDynamicGenerator.cpp:18`）→ `FDynamicGeneratorCore::Generator()`（`:34`）→ 生成动态类/结构体/枚举 → 调用 `SetMetaData` → 消费步骤 3 里那 6 个**仍指向已删除对象**的静态数组。

**问题**
第 2 次及以后的 C# 热重载，`SetMetaData` 会用悬垂的 `FClassReflection*` 去读 `GetName()`/`GetManagedClass()`/`GetCustomAttribute(...)`。触发路径明确且**每次热重载都会走**（只要存在任何一个带 `[UClass]`/`[UProperty]`/`[UStruct]`/`[UEnum]` 特性的 C# 动态类型）。因为数组是函数局部 `static`，它甚至在**同一个** `FDynamicGeneratorCore` 翻译单元的生命周期内永远不会被刷新 —— 这与 `FReflectionRegistry` 的销毁/重建形成了必然失配。
严重度理由：读已释放的堆对象 → 轻则元数据错乱（Pin 类型/默认值错误，静默数据错误），重则崩溃。属于"崩溃/数据损坏"。

**建议**
方案 A（最小改动，推荐）：不要缓存，改为每次从注册表取：
```cpp
const TArray<FClassReflection*>& FDynamicGeneratorCore::GetClassMetaDataAttributes()
{
    static TArray<FClassReflection*>* Cached = nullptr;
    if (Cached == nullptr || FDynamicGeneratorCore::bMetaDataCacheDirty)
    { /* 重建 */ }
    return *Cached;
}
```
即引入缓存失效标志，并在 `FReflectionRegistry::Deinitialize()` 结束处（`FReflectionRegistry.cpp:877` 之后）通知清理；或更简单地提供 `FDynamicGeneratorCore::ResetMetaDataAttributesCache()` 并在第 5 步与第 6 步之间调用。
方案 B：让 `FReflectionRegistry` 不 `delete` 而是复用 `FClassReflection`（用 `TSharedPtr` 管理），代价是 `FMethodReflection*`（见 F-LIFE-002）也要一起改成弱引用。

**验证方式**
grep `static TArray<FClassReflection\*>` → 命中 `FDynamicGeneratorCore.cpp` 6 处；复现用例：编辑器内改一个 C# 动态类的 `[UProperty]` 特性 → 首次编译（第 1 次重载）正常 → 再改一次 → 第 2 次重载时在 `SetFieldMetaData`（`:792`）打 `check(IsValidLowLevel())` 或直接看是否崩溃。

### P1

#### [F-LIFE-006] 动态类型的静态裸指针注册表永不清理（`NamespaceMap`/`DynamicXMap`/`DynamicXSet`）+ `SDynamicClassViewer` 的 `static TSharedPtr` 进程级单例

- **类别**: 内存/资源泄漏
- **严重度**: **P1**(功能错误/泄漏)
- **复核结论**: 部分确认（偏差：机制成立、P1 维持；但原文的 grep 命中数 **46 写错**，真实为 **56**，且原文给的搜索模式**漏掉了 `DynamicClassSet`**——已修正）
- **可达性**: 活跃
- **复核证据**: 8 个静态容器声明与定义逐行确认：`FDynamicClassGenerator.h:127,131,133` / `.cpp:31,35,37`；`FDynamicStructGenerator.h:48,50,52` / `.cpp:17,19,21`；`FDynamicEnumGenerator.h:42,44,46` / `.cpp:16,18,20`；`FDynamicInterfaceGenerator.h:45,47,49` / `.cpp:15,17,19`。写入点确认：`FDynamicEnumGenerator.cpp:175,177,179`、`FDynamicStructGenerator.cpp:218,220,222`、`FDynamicClassGenerator.h:67,69,71`、`FDynamicInterfaceGenerator.cpp:235,237,239`。**grep 重跑**：工具 grep `NamespaceMap|DynamicClassMap|DynamicStructMap|DynamicStructSet|DynamicEnumMap|DynamicEnumSet|DynamicInterfaceMap|DynamicInterfaceSet` → **命中 56**（原文 46 ❌）；补上被漏掉的 `DynamicClassSet` 后为 **64 处**；两个口径下 `Empty()`/`Reset()` 均 **0 处** ✅。唯二例外确认：`FDynamicClassGenerator.cpp:176` `DynamicClassSet.Remove(OldClass);`、`FDynamicStructGenerator.cpp:94` `DynamicStructSet.Remove(OldScriptStruct);`。单例侧确认：`SDynamicClassViewer.cpp:7`（`static TSharedPtr<FClassCollector>`）、`:19` 惰性 `MakeShared`；`ClassCollector.cpp:9,11`（`AllNodes`/`NodesSet` 静态）。
- **级别变动**: 无（P1 维持：静态容器只增不减 + 持有 `MarkAsGarbage` 后的旧 `UClass*`）
- **文件**: `Source/UnrealCSharpCore/Public/Dynamic/FDynamicClassGenerator.h:127,131,133`、`FDynamicStructGenerator.h:48,50,52`、`FDynamicEnumGenerator.h:42,44,46`、`FDynamicInterfaceGenerator.h:45,47,49`（定义于对应 `.cpp` 的 `:31/:35/:37`、`:17/:19/:21`、`:16/:18/:20`、`:15/:17/:19`）；`Source/UnrealCSharpEditor/Private/NewClass/SDynamicClassViewer.cpp:7`
- **函数**: `FDynamicClassGenerator::Generator`、`FDynamicStructGenerator::Generator`、`FDynamicEnumGenerator::Generator`、`FDynamicInterfaceGenerator::Generator`、`SDynamicClassViewer::Construct`
- **置信度**: 高（grep 全插件未发现任何 `NamespaceMap.Empty()` / `DynamicXMap.Empty()` / `DynamicXSet.Empty()`）

**现状（代码事实）**
```cpp
// FDynamicClassGenerator.cpp:31,35  （静态定义）
TMap<UClass*, FString> FDynamicClassGenerator::NamespaceMap;
TMap<FString, UClass*> FDynamicClassGenerator::DynamicClassMap;

// FDynamicStructGenerator.cpp:17,19,21
TMap<UDynamicScriptStruct*, FString> FDynamicStructGenerator::NamespaceMap;
TMap<FString, UDynamicScriptStruct*> FDynamicStructGenerator::DynamicStructMap;
TSet<UDynamicScriptStruct*>              FDynamicStructGenerator::DynamicStructSet;

// FDynamicEnumGenerator.cpp:16,18,20  /  FDynamicInterfaceGenerator.cpp:15,17,19
TMap<UEnum*, FString> FDynamicEnumGenerator::NamespaceMap;   // … 同样三件套
TMap<UClass*, FString> FDynamicInterfaceGenerator::NamespaceMap;
```
写入点：`FDynamicEnumGenerator.cpp:175,177,179`、`FDynamicStructGenerator.cpp:218,220,222`、`FDynamicClassGenerator.h:67,69,71`（内联）、`FDynamicInterfaceGenerator.cpp:235,237,239`。**全插件 grep `NamespaceMap|DynamicClassMap|DynamicStructMap|DynamicStructSet|DynamicEnumMap|DynamicEnumSet|DynamicInterfaceMap|DynamicInterfaceSet` 共 56 处命中（原文误写 46，复核修正；补上被漏掉的 `DynamicClassSet` 后为 64 处），其中 `Empty()`/`Reset()` 命中 0 处**（唯一例外是 `FDynamicStructGenerator.cpp:94` 的 `DynamicStructSet.Remove(OldScriptStruct)` 与 `FDynamicClassGenerator.cpp:176` 的 `DynamicClassSet.Remove(OldClass)`，只移除被替换的单个元素）。

另一个进程级单例：
```cpp
// SDynamicClassViewer.cpp:7,19
TSharedPtr<FClassCollector> SDynamicClassViewer::ClassCollector;   // :7 静态
…
if (!ClassCollector.IsValid()) { ClassCollector = MakeShared<FClassCollector>(); }   // :17,:19 惰性单例，永不 Reset
// ClassCollector.cpp:9,11
TArray<TSharedPtr<FDynamicClassViewerNode>> FClassCollector::AllNodes;
TSet<TSharedPtr<FDynamicClassViewerNode>, FDynamicClassViewerNodeKeyFuncs> FClassCollector::NodesSet;
```

**调用上下文**
`FDynamicGenerator::Generator()`（`FDynamicGenerator.cpp:18`）在**每次模块 Activate**（`FCSharpCompilerRunnable.cpp:229`、`FUnrealCSharpModule::OnUnrealCSharpCoreModuleActive`（`UnrealCSharp.cpp:36-48`））以及每次工具栏"Generator Code"（`UnrealCSharpEditor.cpp:314-393`）时执行，从而反复走这些写入点。

**问题**
1. `NamespaceMap` 以 `UClass*`/`UEnum*`/`UDynamicScriptStruct*` 为键，是**裸指针**（无 `UPROPERTY`、非 `FGCObject`）。虽然表 1 的 `AddToRoot`/ReInstance 机制让对象本身目前不会 GC 悬垂，但这些表**只增不减**：
   - 每次生成一个**新名字**的动态类型 → 3 个容器各 +1 条目；
   - 若 C# 侧重命名/删除一个动态类型，旧条目永久保留（`DynamicStructSet.Remove`/`DynamicClassSet.Remove` 只在"同名重建"时移除）；
   - 被 `ReInstance` 标记 `MarkAsGarbage()`（`FDynamicClassGenerator.cpp:542`）的旧 `UClass*` 仍留在 `NamespaceMap`/`DynamicClassMap` 里 → **裸指针指向已被 GC 回收的对象**，而 `NamespaceMap.Find(旧Class)` 之类的查询会拿它当有效键（`FDynamicClassGenerator.cpp:279`、`FDynamicInterfaceGenerator.cpp:149`、`FDynamicEnumGenerator.cpp:129`、`FDynamicStructGenerator.cpp:153`）。
2. `SDynamicClassViewer::ClassCollector` 是 `static TSharedPtr`，使 `FClassCollector` 实例（以及表 3 #31-#34 的 8 个全局委托绑定）**在编辑器模块卸载后依然存活**且永不解绑；`AllNodes`/`NodesSet` 静态容器同样不随编辑器模块卸载清空。
3. 量级：`NamespaceMap` 每条目 ≈ `FString`（几十字节），`DynamicXMap`/`DynamicXSet` 每条目 ≈ 指针+哈希 ≈ 16-48 字节；以 1000 个动态类型计约 0.1 MB 级 —— **量级不大，但语义上是"永不释放 + 可能持有已回收指针"**，且与 F-LIFE-010 的悬垂问题同源。

**建议**
- 在域卸载路径（`FReflectionRegistry::Deinitialize()` 之前，或 `FCSharpEnvironment::Deinitialize()` 开头）统一清空这 8 个静态容器，并顺手 `NamespaceMap.Empty()` 等；前提是确认动态对象会被重新生成（当前设计是复用，见 F-LIFE-004）。
- 更稳的设计：把 `NamespaceMap` 的键从裸指针改成 `FSoftObjectPath`/`FName`，或改成 `TMap<TWeakObjectPtr<UClass>, FString>`，消除悬垂键。
- `SDynamicClassViewer::ClassCollector` 改为在 `SDynamicClassViewer` 的析构（或编辑器模块 `ShutdownModule`）里 `Reset()`；`AllNodes`/`NodesSet` 一并清空。

**验证方式**
`grep -n "NamespaceMap\|DynamicClassMap\|DynamicClassSet\|DynamicStructMap\|DynamicStructSet\|DynamicEnumMap\|DynamicEnumSet\|DynamicInterfaceMap\|DynamicInterfaceSet" Source/ -r` 后统计 `Empty()`/`Reset()` 命中数（当前为 0，总命中 64）；或运行编辑器、反复执行 "Generator Code" 观察 `NamespaceMap.Num()` 单调增长。

#### [F-LIFE-007] `FEditorListener` 目录监视回调句柄在 `for` 循环内被反复覆盖 → 多目录监听永不注销

- **类别**: 内存/资源泄漏 | 未定义行为
- **严重度**: **P1**(功能错误/泄漏)
- **复核结论**: 确认
- **可达性**: 活跃
- **复核证据**: 注册/注销两侧逐行确认：`FEditorListener.cpp:89-97`（`:89` `for (const auto& Directory : FUnrealCSharpFunctionLibrary::GetChangedDirectories())`、`:91-96` `RegisterDirectoryChangedCallback_Handle`，输出参数是**同一个** `OnDirectoryChangedDelegateHandle` = `:94`）；`FEditorListener.cpp:110-120`（`:115` 同样循环、`:117-118` 用同一句柄逐个 `Unregister`）。成员为单个 `FDelegateHandle`（`FEditorListener.h:87`）。**引擎侧行为已核实（原文标注"置信度：高"但未给引擎证据）**：`UnregisterDirectoryChangedCallback_Handle` 要求"目录键 + 句柄"都匹配才会移除——`Developer/DirectoryWatcher/Private/Windows/DirectoryWatcherWindows.cpp:82-109`（`:86` 先比目录、`:91` `if (RequestPair.Value->RemoveDelegate(InHandle))`，不匹配则 `:108` `return false`）；代理层同构 `DirectoryWatcherProxy.cpp:43-45`。故前 N-1 个目录的委托**保持挂载**且是 `CreateRaw(this, …)`（`:93`）→ 悬垂 `this` 成立。附带事实：`RegisterDirectoryChangedCallback_Handle` 的 `bool` 返回值在 `:91-96` 被忽略（未检查返回值维度的一处实例）。
- **级别变动**: 无（P1 维持；与姊妹报告 `04-…/02` 的 F-ED2-003 定级一致（同为 P1），而 F-LIFE-001 取 P0 亦与那里的 F-ED2-001 一致——两处定级差异沿用同源报告的既定口径，不单方面改动）
- **文件**: `Source/UnrealCSharpEditor/Private/Listener/FEditorListener.cpp:89-97`（注册）、`:110-120`（注销）、`Source/UnrealCSharpEditor/Public/Listener/FEditorListener.h:87`（单句柄成员）
- **函数**: `FEditorListener::FEditorListener()`、`~FEditorListener()`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FEditorListener.cpp:86-97  注册：同一个句柄变量被循环覆盖
auto& DirectoryWatcherModule = FModuleManager::LoadModuleChecked<FDirectoryWatcherModule>(TEXT("DirectoryWatcher"));

for (const auto& Directory : FUnrealCSharpFunctionLibrary::GetChangedDirectories())
{
    DirectoryWatcherModule.Get()->RegisterDirectoryChangedCallback_Handle(
        Directory,
        IDirectoryWatcher::FDirectoryChanged::CreateRaw(this, &FEditorListener::OnDirectoryChanged),   // :93 裸 this
        OnDirectoryChangedDelegateHandle,                                                             // :94 每轮被覆盖
        IDirectoryWatcher::WatchOptions::IncludeDirectoryChanges
    );
}

// FEditorListener.cpp:110-120  注销：拿最后一个句柄去循环注销
if (OnDirectoryChangedDelegateHandle.IsValid())
{
    auto& DirectoryWatcherModule = FModuleManager::LoadModuleChecked<FDirectoryWatcherModule>(TEXT("DirectoryWatcher"));

    for (const auto& Directory : FUnrealCSharpFunctionLibrary::GetChangedDirectories())
    {
        DirectoryWatcherModule.Get()->UnregisterDirectoryChangedCallback_Handle(
            Directory, OnDirectoryChangedDelegateHandle);
    }
}
```

**调用上下文**
`GetChangedDirectories()` 返回的目录数量 = C# 脚本目录列表（`UE/`、`Game/`、`Script/` 等，通常 ≥ 2）。注册发生在编辑器模块对象构造时；触发点是任意 `.cs` 文件变化（`OnDirectoryChanged`，`:415`）。

**问题**
`GetChangedDirectories()` 返回 N>1 个目录时，`OnDirectoryChangedDelegateHandle` 只保留第 N 次（最后一次）注册返回的句柄。析构时循环对每个目录都用**同一个（最后一个）句柄**去 `Unregister`：
- 第 N 个目录：成功注销；
- 前 N-1 个目录：`UnregisterDirectoryChangedCallback_Handle` 用它不匹配的句柄 → 查找失败，**委托保持挂载**；而该委托是 `CreateRaw(this, …)`，`this` 即将析构 → **下一次文件变化时调用已析构的 `FEditorListener` → use-after-free**。
这与 F-LIFE-001 是同一类缺陷（raw 委托未注销），但成因不同（句柄覆盖 vs 完全没存句柄），且这里的崩溃触发条件更容易命中（只要有两个脚本目录 + 之后动一次 `.cs` 文件）。

**建议**
```cpp
// 成员改为容器
TArray<FDelegateHandle> OnDirectoryChangedDelegateHandles;
// 注册
for (const auto& Directory : FUnrealCSharpFunctionLibrary::GetChangedDirectories())
{
    FDelegateHandle Handle;
    if (DirectoryWatcherModule.Get()->RegisterDirectoryChangedCallback_Handle(
            Directory, IDirectoryWatcher::FDirectoryChanged::CreateRaw(this, &FEditorListener::OnDirectoryChanged),
            Handle, IDirectoryWatcher::WatchOptions::IncludeDirectoryChanges))
    {
        OnDirectoryChangedDelegateHandles.Add(Handle);
    }
}
// 注销：逐目录逐句柄
```
若目录列表固定且顺序稳定，也可保留单句柄但改成 `TArray<FDelegateHandle>` 一一对应。同时建议把 `CreateRaw` 换成 `CreateWeakLambda`（捕获 `TWeakObjectPtr<FEditorListener>`，但该类非 UObject，需自建 weak 标志位）或让 `FEditorListener` 继承 `TSharedFromThis` 后用 `CreateSP`。

**验证方式**
在注册/注销处打日志输出 `Directory` 与 `Handle.ToString()`，编辑器启动+关闭后对比：当前实现下只有最后一对的 handle 值一致。或在 `~FEditorListener` 后手动触发一次 `.cs` 文件写入。

#### [F-LIFE-008] 托管侧 `AddReference`/`RemoveReference` 是本插件唯一的 UObject GC 保护，且无 C++ 侧兜底

- **类别**: 内存/资源泄漏 | 未定义行为
- **严重度**: **P2**(功能错误/泄漏)
- **复核结论**: 部分确认（偏差：全部代码事实与 P1→P2 降级成立；但原文第 2 点的一句推演——"仅 MarkOutdated 而未 Deactivate 的路径不会清空 `ObjectArray`"——**无法验证**，因为未找到"仅 MarkOutdated 而不 Deactivate"的实际调用路径，该句属推测）
- **可达性**: 活跃（`AddReferencedObjects` 每次 GC 必经；但"泄漏"需要托管侧漏调 `RemoveReference`）
- **复核证据**: 三处引用逐行核实——① `FReferenceRegistry.h:5` `class UNREALCSHARP_API FReferenceRegistry : FGCObject`（`class` → **私有继承**，原文第 3 点正确）、`:13` `AddReferencedObjects` 覆写、`:29` `TArray<TObjectPtr<UObject>> ObjectArray;`；② `FReferenceRegistry.cpp:22-25` `Collector.AddReferencedObjects(ObjectArray);`（`:24`）、`:59-69` `AddReference(UObject*)`/`:71-81` `RemoveReference(UObject*)`（原文写 `:32-81`，涵盖全部 4 个重载，正确）、`:7-20` 析构 `ObjectArray.Empty()`（`:19`）；③ `FRegisterObject.cpp:111-119` `AddReferenceImplementation`（`:115` 调 `AddReference`）、`:121-129` `RemoveReferenceImplementation`（`:125`）、`:141/:142` 注册 `AddToRoot`/`RemoveFromRoot`、`:144/:145` 注册 `AddReference`/`RemoveReference`。**验证方式里给的调用点也全部复核成立**：`FRegisterObject.cpp:115`、`FDelegateRegistry.inl:52`、`FCSharpEnvironment.cpp:615`（`AddReference(InOwner, FReference*)`）/`:625`（`AddReference(UObject*)`）、Remove 侧 `FRegisterObject.cpp:125`、`FCSharpEnvironment.cpp:620`/`:630`；`FDelegateRegistry.inl:52-54` 确认委托引用也挂到 `FReferenceRegistry`。
- **级别变动**: 无（维持 P1→P2：本条为"覆盖面/设计"陈述而非确定性缺陷路径，且唯一的确定性后果需要托管侧漏调 `RemoveReference`）
- **文件**: `Source/UnrealCSharp/Public/Registry/FReferenceRegistry.h:5,13,29`、`Source/UnrealCSharp/Private/Registry/FReferenceRegistry.cpp:7-25,59-81`、`Source/UnrealCSharp/Private/Domain/Interop/FRegisterObject.cpp:111-129,141-145`
- **函数**: `FReferenceRegistry::AddReference(UObject*)` / `RemoveReference(UObject*)`、`AddToRootImplementation` / `RemoveFromRootImplementation`（C# 绑定）
- **置信度**: 高

**现状（代码事实）**
```cpp
// FReferenceRegistry.h:5-29  —— 全插件唯一继承 FGCObject 的类
class UNREALCSHARP_API FReferenceRegistry : FGCObject
{
    virtual void AddReferencedObjects(FReferenceCollector& Collector) override;   // :13
    TManagedHandleMapping<TSet<class FReference*>> ReferenceRelationship;         // :27
    TArray<TObjectPtr<UObject>> ObjectArray;                                      // :29
};

// FReferenceRegistry.cpp:22-25
void FReferenceRegistry::AddReferencedObjects(FReferenceCollector& Collector)
{
    Collector.AddReferencedObjects(ObjectArray);   // 通过 TObjectPtr 正确保活
}

// FReferenceRegistry.cpp:59-81
bool FReferenceRegistry::AddReference(UObject* InObject)
{
    if (InObject != nullptr) { ObjectArray.AddUnique(InObject); return true; }
    return false;
}
bool FReferenceRegistry::RemoveReference(UObject* InObject)
{
    if (InObject != nullptr) { ObjectArray.Remove(InObject); return true; }
    return false;
}
```
```cpp
// FRegisterObject.cpp:111-129,141-145  —— 只有 C# 显式调用才会进这个保护
static uint8 AddReferenceImplementation(const IManagedHandle InManagedHandle)
{
    if (const auto FoundObject = FCSharpEnvironment::GetEnvironment().GetObject(InManagedHandle))
    {
        return FCSharpEnvironment::GetEnvironment().AddReference(FoundObject) ? 1 : 0;
    }
    return 0;
}
// …绑定注册：
.Function("AddReference", AddReferenceImplementation)       // :144
.Function("RemoveReference", RemoveReferenceImplementation) // :145
```

**调用上下文**
`AddReferencedObjects` 由 UE GC 在每次 `CollectGarbage` 时调用（`FGCObject` 走全局 `FGCObject::GGCObjectReferencer`）。`AddReference`/`RemoveReference` 只有两个入口：C# 侧 `UObject.AddReference()`/`RemoveReference()`，以及 `FDelegateRegistry.inl:52-54`（`AddReference(InOwner, new TDelegateReference<...>)`）。

**问题**
1. **覆盖面不足**：整个插件只在 `ObjectRegistry`/`ReferenceRegistry` 这一条链上做了正确的 GC 保护；所有其它持有 UObject 的地方（`FDynamicClassGenerator` 的 3 张静态 map、`FCSharpBind::NotOverrideTypes`、`FDelegateWrapper::Method`、`UDelegateHandler::ScriptDelegate`）都靠"对象处于 ReferenceRegistry.ObjectArray 中"或"已被 `AddToRoot`"间接保证。**一旦 C# 侧忘记调用 `RemoveReference`，对象就永久留在 `ObjectArray` 里**（`TObjectPtr` 强引用，GC 永远不会回收）——这是比 `AddToRoot` 更隐蔽的泄漏，因为没有任何日志/断言。
2. **语义不对称**：`FReferenceRegistry` 提供了"从 C# 侧手动保活"的能力，但**没有任何地方在域卸载或模块停用时清空 `ObjectArray`**（`~FReferenceRegistry()` :7-20 只在 registry 被 `delete` 时清空，这发生在 `FCSharpEnvironment::Deinitialize()` :235-240，即模块 Deactivate 时；**热重载路径会走，但"仅 MarkOutdated 而未 Deactivate"的路径不会**）。更关键的是：如果 C# 侧持有了 `AddReference` 的对象但域被卸载，C++ 的 `ObjectArray` 会随 registry 一起清空（安全）；反过来，如果 C++ 侧还在用而 C# 侧提前 `RemoveReference`，对象可能在下一帧被 GC，而 C++ 缓存的裸指针（表 2 各行）就悬垂了。
3. `FReferenceRegistry` 用的是 `class FReferenceRegistry : FGCObject` **私有继承**（`class` 默认 private），`FGCObject` 的 `AddReferencedObjects` 在私有基类里 `override` 是合法的，但外部无法把它当 `FGCObject*` 使用 —— 只影响可读性，不影响功能。

**建议**
- 在 `FReferenceRegistry` 增加诊断：`#if !UE_BUILD_SHIPPING` 下在 `AddReference` 里 `UE_LOG(LogUnrealCSharp, Verbose, TEXT("AddReference %s (total %d)"), *InObject->GetName(), ObjectArray.Num())`，并在 `RemoveReference` 找不到对象时 `ensureAlways`。
- 对"必须保活"的 C++ 内部对象，改为 `AddToRoot` + RAII 或 `FGCObject` 子类，而不是依赖托管侧调用。
- 与 F-LIFE-002 一起修：`FMethodReflection*` 的存活不应依赖 C# 侧是否调用 `AddReference`。

**验证方式**
grep `AddReference(` 的全部调用点（`FRegisterObject.cpp:115`、`FDelegateRegistry.inl:52`、`FCSharpEnvironment.cpp:615/625`）与 `RemoveReference`（`FRegisterObject.cpp:125`、`FCSharpEnvironment.cpp:620/630`）做配对审计；或运行时打印 `ReferenceRegistry.ObjectArray.Num()` 观察是否单调增长。

#### [F-LIFE-015] CoreCLR `AssemblyLoader.Unload()` 只等 2 秒就放弃，可回收 ALC 若未回收则永久驻留且无任何上报

- **类别**: 内存/资源泄漏
- **严重度**: **P2**(功能错误/泄漏)
- **复核结论**: 部分确认（偏差：代码事实全对、P1→P2 降级成立；**但"可达性 活跃"是错的，应改为"潜伏"**——见复核证据）
- **可达性**: 潜伏（当前工程五平台全部 LeanCLR：`Unload()` 确实被调用，但其整个函数体被 `if (Context != null)` 守卫，而 `Context` 只在 `LoadFromStream`（`AssemblyLoader.cs:18`）里被赋值，LeanCLR 路径**从不调用** `LoadFromStream` → 2 秒等待那段代码在 LeanCLR 下不可能执行；切回 CoreCLR/Mono 后端即生效）
- **复核证据**: `AssemblyLoader.cs` 逐行确认（真路径 `Script/Interop/AssemblyLoader/AssemblyLoader.cs`，相对插件根）：`:31` `public static void Unload()`、`:35` `HandleData.Clear();`、`:37` `TypeBridge.Clear();`、`:39` `new WeakReference(Context)`、`:43` `Context.Unload();`、`:45-48` `finally { Context = null; }`（赋值在 `:47` ✅）、`:50` `const int TimeLimit = 2000;`、`:54` `while (ContextWeakReference.IsAlive && Stopwatch.ElapsedMilliseconds < TimeLimit)`、`:56-58` `GC.Collect(); GC.WaitForPendingFinalizers();`、`:18` `Context ??= new UnrealAssemblyLoadContext(...)`（**`Context` 的唯一赋值点**）。`UnrealAssemblyLoadContext.cs:7-8` `internal sealed class UnrealAssemblyLoadContext(string InPublishDirectory) : AssemblyLoadContext(name: ..., isCollectible: true)` ✅。**可达性证据**：`FLeanCLRDomain.cpp:806-813` `UnloadAssembly()` 确实 `Bridge_Invoke(AssemblyLoaderUnloadFn)`（`:810-812`），但 LeanCLR 的加载走 `leanclr::vm::Assembly::load_by_name`（`FLeanCLRDomain.cpp:795-802`），全插件 `AssemblyLoaderLoadFromStreamFn` 的调用点只有 `FMonoDomain.cpp:395,415` 与 `FCoreCLRDomain.cpp:226,228`（grep `AssemblyLoader` 命中 31 处，LeanCLR 域内无 LoadFromStream 调用）→ `Context` 保持 `null`。
- **级别变动**: 无（维持 P1→P2：CoreCLR/Mono 专属，当前工程不可达；作为潜伏缺陷 P2 合理）
- **文件**: `Script/Interop/AssemblyLoader/AssemblyLoader.cs:30-61`（尤其 `:39-59`）；`Script/Interop/AssemblyLoader/UnrealAssemblyLoadContext.cs:7-8`
- **函数**: `Interop.AssemblyLoader.Unload()`、`UnrealAssemblyLoadContext`
- **置信度**: 中（`GC.Collect()` 循环确实存在；未验证在真实工程中 2 秒是否足够）

**现状（代码事实）**
```csharp
// UnrealAssemblyLoadContext.cs:7-8  —— 可回收 ALC 配置正确
internal sealed class UnrealAssemblyLoadContext(string InPublishDirectory)
    : AssemblyLoadContext(name: "UnrealAssemblyLoadContext", isCollectible: true)

// AssemblyLoader.cs:30-61
[UnmanagedCallersOnly]
public static void Unload()
{
    if (Context != null)
    {
        HandleData.Clear();      // :35  释放全部 GCHandle
        TypeBridge.Clear();      // :37
        var ContextWeakReference = new WeakReference(Context);   // :39
        try { Context.Unload(); }                                 // :43
        finally { Context = null; }                               // :47  ← 无论成功与否都置空
        const int TimeLimit = 2000;                               // :50  硬编码 2 秒
        var Stopwatch = System.Diagnostics.Stopwatch.StartNew();
        while (ContextWeakReference.IsAlive && Stopwatch.ElapsedMilliseconds < TimeLimit)  // :54
        {
            GC.Collect();
            GC.WaitForPendingFinalizers();
        }
    }
}
```
对照 `HandleData.Clear()`（`HandleData.cs:129-145`）确实逐个 `Free()` 并清空字典，`isCollectible: true` 也正确设置。

**调用上下文**
`FMonoDomain::UnloadAssembly` / `FCoreCLRDomain::UnloadAssembly`（`:240-270`）/ `FLeanCLRDomain::UnloadAssembly`（`:806`）在域卸载时调用 `AssemblyLoaderUnloadFn()`；触发链路见 F-LIFE-010 的 1-6 步。

**问题**
1. 2 秒后无条件退出循环（`:54` 的条件不满足即跳出），**不做任何日志/上报**。若旧 ALC 未被回收（只要有任意一个 `Type`、`MethodInfo`、`Assembly` 或 lambda 被 Default ALC、static 字段或正在运行的线程引用，就会发生），`Context` 已被置 `null`（`:47`），下一次 `LoadFromStream`（`:18` 的 `Context ??= …`）会创建**全新的 collectible ALC**；旧的那个既不能卸载也无法再被卸载（没有任何引用可用了）→ **每次热重载永久泄漏一整个程序集加载上下文**（`Interop.dll` 之外的全部脚本程序集 + 元数据 + JIT 代码 ≈ 数十 MB 级）。
2. 由于 `Context = null` 在 `finally` 里，**泄漏是静默的**：插件无法知道自己刚刚泄漏了一个 ALC。
3. 一个具体的高风险引用源：`AddDynamic`/`AddStatic` 类静态注册、`SynchronizationContext` 的 tick 委托、以及 `TypeBridge` 缓存的 `MethodInfo`（`TypeBridge.Clear()` 在卸载时执行，方向正确）。

**建议**
```csharp
var ContextWeakReference = new WeakReference(Context);
Context.Unload();
Context = null;
for (var i = 0; i < 10 && ContextWeakReference.IsAlive; ++i)
{
    GC.Collect();
    GC.WaitForPendingFinalizers();
}
if (ContextWeakReference.IsAlive)
{
    // 上报给 C++ 侧（例如 Console.Error / 一个 exported 计数接口），至少不要静默
}
```
同时把 2 秒硬编码改为可配置，并把"未回收"计入诊断计数器。若要根治，需要排查并把所有跨越 ALC 边界的引用改为 `WeakReference`/`UnmanagedCallersOnly` 静态函数指针。

**验证方式**
在 `Unload()` 里加 `ContextWeakReference.IsAlive` 的结果输出，编辑器内连续热重载 10 次，观察是否有任何一次未回收；或用 `dotnet-counters`/任务管理器观察进程私有内存是否每次重载后单调上升。

#### [F-LIFE-016] 进程级单例 `FDynamicDependencyGraph` 的 `NodeArray` 只增不减，且 `Node::OnCompleted` 的 `TFunction` 捕获了 `FClassReflection*`

- **类别**: 内存/资源泄漏 | 未定义行为
- **严重度**: **P2**(功能错误/泄漏)
- **复核结论**: 部分确认（偏差：代码事实与 P1→P2 降级成立；但验证方式里给的 `NodeArray` grep 命中清单**漏了 `.cpp:40`**，且 `FDynamicDependencyGraph.cpp:14` 应为 `:15`，已修正）
- **可达性**: 活跃
- **复核证据**: `FDynamicDependencyGraph.cpp:4-16` 逐行确认（`:6` `static FDynamicDependencyGraph Instance;`、`:13` `NodeArray.Emplace(InNode);`、**`:15`** `NodeMap.Emplace(InNode.Name, NodeArray.Num() - 1);`——原文标 `:14`，实为 `:15`）。头文件：`.h:83` `TArray<FDependency> Dependencies;`、`:87` `TArray<TFunction<void()>> OnCompleted;`、`:105` `TArray<FNode> NodeArray;`、`:107` `TMap<FString, int32> NodeMap;` 全部 ✅（另有 `:85` `TFunction<void()> GeneratorImplementation;`）。消费点确认：`.cpp:140`、`:204` 调 `NodeArray[NodeMap[...]].Generator()`，而 `FNode::Generator()`（`.h:69-77`）先执行 `GeneratorImplementation()` 再逐个执行 `OnCompleted` 项。**grep 重跑**（模式 `NodeArray|@TODO|OnCompleted`）：`NodeArray` 在 `.cpp` 命中 **25** 行（`13,15,28,32,36,40,50,64,68,72,76,82,105,112,126,130,140,142,146,159,168,179,186,204,206`）——原文清单为 24 行、**漏 `40`**；`NodeArray.Empty()`/`Reset()` 命中 **0** ✅；`// @TODO` 命中 **3** 处（`:86`、`:101`、`:210`）✅ 与原文一致。捕获点确认：`FDynamicEnumGenerator.cpp:39-45`（`:40` `[InClassReflection]()` 捕获 `FClassReflection*`）✅；`FReflectionRegistry.cpp:867-877` 无条件 `delete` ✅。
- **级别变动**: 无（维持 P1→P2：`NodeArray` 无界增长量级小，悬垂捕获需要"遗留 Pending 节点"这一未验证前提，置信度中）
- **文件**: `Source/UnrealCSharpCore/Private/Dynamic/FDynamicDependencyGraph.cpp:4-16`（`Get()` 与 `AddNode`）、`Source/UnrealCSharpCore/Public/Dynamic/FDynamicDependencyGraph.h:83,85,87,105,107`
- **函数**: `FDynamicDependencyGraph::AddNode(const FNode&)`、`FDynamicDependencyGraph::Generator()`
- **置信度**: 中高（泄漏确定；`OnCompleted` 悬垂捕获是否可达取决于是否有上次遗留的 Pending 节点）

**现状（代码事实）**
```cpp
// FDynamicDependencyGraph.cpp:4-16
FDynamicDependencyGraph& FDynamicDependencyGraph::Get()
{
    static FDynamicDependencyGraph Instance;   // :6  进程级单例，永不销毁
    return Instance;
}

void FDynamicDependencyGraph::AddNode(const FNode& InNode)
{
    NodeArray.Emplace(InNode);                        // :13  只增
    NodeMap.Emplace(InNode.Name, NodeArray.Num() - 1); // :15  同名时覆盖索引
}
```
```cpp
// FDynamicDependencyGraph.h:83-107
struct FNode { … TArray<FDependency> Dependencies; … TArray<TFunction<void()>> OnCompleted; … };
TArray<FNode> NodeArray;          // :105
TMap<FString, int32> NodeMap;     // :107
```
捕获点的典型形态（`FDynamicEnumGenerator.cpp:39-43`，其余生成器同构）：
```cpp
const auto Node = FDynamicDependencyGraph::FNode(
    ClassName, [InClassReflection]() { Generator(InClassReflection); });   // 捕获反射对象裸指针
FDynamicGeneratorCore::AddNode(Node);
```

**调用上下文**
`FDynamicGeneratorCore::AddNode`（`FDynamicGeneratorCore.cpp:57-59`）→ `FDynamicDependencyGraph::Get().AddNode`；每次 `FDynamicGenerator::Generator()`（`FDynamicGenerator.cpp:18-34`）都会为每个动态类型 AddNode 一次。`Generator()`（`FDynamicDependencyGraph.cpp:80-215`）遍历 `NodeArray` 时对未完成节点调用 `NodeArray[…]` 里保存的 `TFunction`（`:140`、`:204`）。

**问题**
1. **`NodeArray` 每次 Generator 运行都追加 N 个 `FNode`（N = 动态类型数），永不释放**。`NodeMap` 因同名覆盖而有界，但 `NodeArray` 无界。以 2000 个动态类型、每次运行 `FNode` 约 100-300 字节计，**每次运行泄漏约 0.2-0.6 MB**，热重载 50 次即 10-30 MB。
2. `Node::OnCompleted` 保存的 `TFunction<void()>` 捕获了 `FClassReflection*`（`FDynamicEnumGenerator.cpp:40`、`FDynamicStructGenerator.cpp`、`FDynamicClassGenerator.cpp` 同构）。`FReflectionRegistry::Deinitialize()`（`FReflectionRegistry.cpp:867-877`）会 `delete` 全部 `FClassReflection` → 这些捕获变成悬垂。触发 UAF 的条件是"前一次运行遗留了未被 `Completed()` 的节点，且本次 `Generator()` 迭代到它"——代码里确实存在不置完成态的分支（`:86-87`、`:101-102`、`:210` 三处 `// @TODO`，以及 `bIsPending` 时的重新入队逻辑）。置信度中：未验证实际运行中是否会产生遗留 Pending 节点。
3. 与 F-LIFE-010 同源（都是"跨域重载的静态缓存持有 `FClassReflection*`"），建议一并修。

**建议**
- 在 `FDynamicGeneratorCore::Generation()`/`FDynamicGenerator::Generator()` 入口处调用 `FDynamicDependencyGraph::Get().Reset()`（新增方法：`NodeArray.Empty(); NodeMap.Empty();`），并确保与 F-LIFE-010 的元数据缓存清理在同一处触发。
- 把 `FNode` 的 `Generator`/`OnCompleted` 捕获从 `FClassReflection*` 改为"名字 + 每次从注册表查询"，彻底消除跨域重载的裸指针捕获。

**验证方式**
grep `NodeArray`（命中 `FDynamicDependencyGraph.h:105`、`.cpp:13,15,28,32,36,50,64,68,72,76,82,105,112,126,130,140,142,146,159,168,179,186,204,206`）确认无 `Empty()`/`Reset()`；运行时打印 `NodeArray.Num()` 观察逐次增长。

### P2

#### [F-LIFE-004] 动态生成的 `UEnum` 被 `AddToRoot()` 后全插件无任何 `RemoveFromRoot` → 枚举永久 rooted

- **类别**: 内存/资源泄漏
- **严重度**: **P2**(性能/隐患)
- **复核结论**: 确认
- **可达性**: 活跃
- **复核证据**: `FDynamicEnumGenerator.cpp:196-206` 逐行确认：`:199` `const auto Enum = NewObject<UEnum>(InOuter, *InName, RF_Public);`、**`:201` `Enum->AddToRoot();`**、`:203` `GeneratorEnum(InNameSpace, InName, Enum, InProcessGenerator);`、`:205` `return Enum;`。**全插件 root 操作 grep 重跑**（模式 `AddToRoot|RemoveFromRoot`，`Source/` 全模块）→ 命中 **26** 处，与原文一致；其中 `FDynamicEnumGenerator.cpp` 只有 :201 一处 `AddToRoot`，**该文件内 `RemoveFromRoot` 命中 0** ✅。且该文件存在 `ReInstance(UEnum*)`（`:209`）却也不反 root —— 与结构体/类的 `ReInstance` 路径（`FDynamicStructGenerator.cpp:380`、`FDynamicClassGenerator.cpp:533,540`）形成对照，坐实"枚举这一环缺失"。全插件 AddToRoot 调用点共 **12** 处（详见 §1 与表 1 的计数修正）。
- **级别变动**: 无（P2 维持：永久 rooted 属泄漏/隐患，非崩溃；但注意它是 `Names` 名表随动态枚举数量单调累积，量级与 F-LIFE-005 同类）
- **文件**: `Source/UnrealCSharpCore/Private/Dynamic/FDynamicEnumGenerator.cpp:196-206`（`AddToRoot` 在 `:201`）
- **函数**: `FDynamicEnumGenerator::GeneratorEnum(UPackage*, const FString&, const FString&, const TFunction<void(UEnum*)>&)`
- **置信度**: 高（`grep -n "RemoveFromRoot" FDynamicEnumGenerator.cpp` 命中 0）

**现状（代码事实）**
```cpp
// FDynamicEnumGenerator.cpp:196-206
UEnum* FDynamicEnumGenerator::GeneratorEnum(UPackage* InOuter, const FString& InNameSpace,
                                            const FString& InName, const TFunction<void(UEnum*)>& InProcessGenerator)
{
    const auto Enum = NewObject<UEnum>(InOuter, *InName, RF_Public);

    Enum->AddToRoot();                                            // :201  ← 全文件唯一一处 rootset 操作

    GeneratorEnum(InNameSpace, InName, Enum, InProcessGenerator);

    return Enum;
}
```
`FDynamicEnumGenerator.cpp` 全文 303 行，已完整读取，**没有任何 `RemoveFromRoot`**；`FDynamicEnumGenerator::ReInstance(UEnum*)`（`:209-269`）只处理引用该枚举的蓝图重编译，不处理 root。对比 `FDynamicStructGenerator.cpp:249`↔`:380`、`FDynamicClassGenerator.cpp:393`↔`:540`、`:419`↔`:533` 都有对应的 `RemoveFromRoot`。

**调用上下文**
`FDynamicEnumGenerator::Generator(FClassReflection*)`（`:66-120`）→ 名字未注册时走 `:105-109` 创建新枚举；已在 `DynamicEnumMap` 时走 `:90-102` **原地复用同一个 UEnum**（`GeneratorEnum(ClassNamespace, ClassName, Enum, …)`）并随后 `ReInstance(Enum)`（`:113-116`）。`DynamicEnumMap` 是静态容器且永不清理（F-LIFE-006）。

**问题**
1. **设计上"用 root 当永久容器"**：因为复用逻辑（`:90-102`）能命中同名枚举，热重载本身**不会**每次新增一个 rooted UEnum —— 这一点必须先说清楚，避免把量级估大。
2. 但泄漏确实存在且不可回收：**凡是在 C# 侧出现过、后来被删除或改名的动态枚举，其 `UEnum` 永久留在 rootset**（`DynamicEnumMap` 仍持有键，且 root 标记使其永不参与 GC）。量级 = O(历史枚举名数量) × (UEnum 对象 + `TArray<TPair<FName,int64>>` 名字表)，单个约 0.1-1 KB。典型工程 100 个枚举、其中 20 个曾改名 → 约几十 KB，属"有界但永不释放"。
3. 严重度定为 P2 而非 P1/P0：不会导致悬垂（root 反而防止了悬垂），也不会随热重载线性增长；但它破坏了"动态类型应当可回收"的设计意图，并与 `DynamicEnumSet`/`DynamicEnumMap` 的永不清理叠加（F-LIFE-006）使得"退役枚举"既不可回收也不能被复用。
4. 需要单独指出的一致性缺陷：`FDynamicEnumGenerator::EndGenerator(UEnum*)`（`:146-170`）在 `#if WITH_EDITOR` 里刷新 `FBlueprintActionDatabase`，说明作者预期枚举**会变化**，而 root 语义却假设它永久存活。

**建议**
- 与 F-LIFE-006 一并修：在域卸载时遍历 `DynamicEnumMap`，对确定不再被任何 `FClassReflection` 引用的枚举执行 `RemoveFromRoot()` + `MarkAsGarbage()`（对称于 `FDynamicClassGenerator::ReInstance` 的做法）。
- 或者移除 `:201` 的 `AddToRoot()`，改为把枚举放进一个 `FGCObject` 子类的 `TObjectPtr` 数组（与 `FReferenceRegistry` 同样的机制），这样能复用现成的正确模式，也避免 root 标记的"永久"语义。
- 若短期不改，至少加注释说明 `:201` 的 root 是有意为之、以及退役枚举不会被回收。

**验证方式**
`grep -n "AddToRoot\|RemoveFromRoot" Source/UnrealCSharpCore/Private/Dynamic/FDynamicEnumGenerator.cpp`；运行时在 C# 侧新增一个枚举 → 编译 → 删除该枚举 → 编译，然后 `obj gc` 观察 `UEnum` 数量是否下降（不会下降）。

#### [F-LIFE-005] 动态生成的 UInterface `UClass` 被 `AddToRoot()` 后全插件无任何 `RemoveFromRoot`

- **类别**: 内存/资源泄漏
- **严重度**: **P2**(性能/隐患)
- **复核结论**: 确认
- **可达性**: 活跃
- **复核证据**: `FDynamicInterfaceGenerator.cpp:257-268` 逐行确认（`:261` `NewObject<UClass>(InOuter, *InName.RightChop(1), RF_Public)`、**`:263` `Class->AddToRoot();`**、`:265` 递归 `GeneratorInterface`）。复用路径确认：`:107-114`（`:107` `DynamicInterfaceMap.Contains(ClassName)`、`:109` `Class = DynamicInterfaceMap[ClassName];`、**`:111` `Class->PurgeClass(true);`**、`:113-116` 带 `[InClassReflection]` 捕获的 lambda）。`CodeAnalysisGenerator` 确认为 `:60-73`，去重点在 `:65` ✅。写入点 `:235,237,239` ✅（与 F-LIFE-006 复核一致）。该文件 `RemoveFromRoot` 命中 **0** ✅（grep `AddToRoot|RemoveFromRoot` 全插件 26 处中，本文件只有 `:263` 一处）。
- **级别变动**: 无（P2 维持：与 F-LIFE-004 同构的"永久 rooted、量级有界不随重载线性增长"）
- **文件**: `Source/UnrealCSharpCore/Private/Dynamic/FDynamicInterfaceGenerator.cpp:257-268`（`AddToRoot` 在 `:263`）
- **函数**: `FDynamicInterfaceGenerator::GeneratorInterface(UPackage*, const FString&, const FString&, UClass*, const TFunction<void(UClass*)>&)`
- **置信度**: 高（`grep -n "RemoveFromRoot" FDynamicInterfaceGenerator.cpp` 命中 0）

**现状（代码事实）**
```cpp
// FDynamicInterfaceGenerator.cpp:257-268
UClass* FDynamicInterfaceGenerator::GeneratorInterface(UPackage* InOuter, const FString& InNameSpace,
                                                       const FString& InName, UClass* InParentClass,
                                                       const TFunction<void(UClass*)>& InProcessGenerator)
{
    const auto Class = NewObject<UClass>(InOuter, *InName.RightChop(1), RF_Public);

    Class->AddToRoot();                                    // :263  ← 全文件唯一一处 rootset 操作

    GeneratorInterface(InNameSpace, InName, Class, InParentClass, InProcessGenerator);

    return Class;
}
```
与 F-LIFE-004 同构。复用路径同样存在：`FDynamicInterfaceGenerator::Generator`（`:107-114`）命中 `DynamicInterfaceMap` 时走 `Class->PurgeClass(true)` 后原地重建：

```cpp
// FDynamicInterfaceGenerator.cpp:107-114
if (DynamicInterfaceMap.Contains(ClassName))
{
    Class = DynamicInterfaceMap[ClassName];
    Class->PurgeClass(true);
    GeneratorInterface(ClassNamespace, ClassName, Class, ParentClass, [InClassReflection](UClass* InInterface) { … });
```

**调用上下文**
`FDynamicInterfaceGenerator::Generator(FClassReflection*)`（`:76`→…）；`CodeAnalysisGenerator`（`:60-73`）在 `:65` 用 `DynamicInterfaceMap.Contains(InName)` 防重复。写入点 `:235,237,239`（`NamespaceMap.Add` / `DynamicInterfaceMap.Add` / `DynamicInterfaceSet.Add`）。

**问题**
与 F-LIFE-004 完全同类：**退役的接口 `UClass` 永久 rooted、永不回收**，且 `DynamicInterfaceMap`/`DynamicInterfaceSet` 永不清理（F-LIFE-006）。量级 = O(历史接口名数量) × UClass 对象（含 `UFunction` 签名链、`UInterface` 关联对象），单个约 1-5 KB。严重度同为 P2（不悬垂、不随热重载线性增长）。
需要额外指出：`Class->PurgeClass(true)`（`:111`）会清空类成员，**如果此时该 `UClass` 仍被任何蓝图/`FClassReflection` 引用，就会出现"类被清空但对象还在"的静默错误**；这与 root 标记叠加后无法通过"等 GC 回收旧类"来掩盖。置信度：中（未追踪所有 `PurgeClass` 的下游影响）。

**建议**
与 F-LIFE-004 完全一致：在域卸载时对退役接口 `RemoveFromRoot()` + `MarkAsGarbage()`，或改用 `FGCObject` + `TObjectPtr` 保活容器；并在 `PurgeClass(true)` 前断言没有其它引用。

**验证方式**
`grep -n "AddToRoot\|RemoveFromRoot\|PurgeClass" Source/UnrealCSharpCore/Private/Dynamic/FDynamicInterfaceGenerator.cpp`。

#### [F-LIFE-009] `UObject.AddToRoot()` / `RemoveFromRoot()` 被直接暴露给 C#，且没有任何配对约束或诊断

- **类别**: 内存/资源泄漏
- **严重度**: **P2**(性能/隐患)
- **复核结论**: 确认（偏差：无——代码事实与定级均成立；补充一条已核实事实：托管侧确有直接调用点）
- **可达性**: 活跃
- **复核证据**: `FRegisterObject.cpp:85-109,141-145` 逐行确认：`:85-91` `AddToRootImplementation`（**`:89` `FoundObject->AddToRoot();`**，无重入保护、无日志）、`:93-99` `RemoveFromRootImplementation`（**`:97` `FoundObject->RemoveFromRoot();`**，无 `IsRooted()` 前置判断）、`:101-109` `IsRootedImplementation`（**`:105` `FoundObject->IsRooted() ? 1 : 0;`**）、注册行 `:141`/`:142`/`:143`/`:144`/`:145` 全部 ✅。三点问题（暴露裸 root API、无配对约束、无诊断）在源码上均成立：实现体里没有任何计数/断言/日志。补充：`AddToRootImplementation`/`RemoveFromRootImplementation` 只挂在 `TBindingClassBuilder<UObject>(NAMESPACE_LIBRARY)`（`:133`）上，C# 侧拿到的是 `UObject.AddToRoot()/RemoveFromRoot()` 自由方法，C++ 侧无法得知调用次数。
- **级别变动**: 无（P2 维持：属"暴露但无约束"的隐患/设计问题；其泄漏后果依赖托管侧误用，非确定性缺陷）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterObject.cpp:85-109,141-145`
- **函数**: `AddToRootImplementation(const IManagedHandle)` / `RemoveFromRootImplementation(const IManagedHandle)` / `IsRootedImplementation`
- **置信度**: 中（托管侧调用点不在本次分析范围内，未逐一核对 Script/ 下所有 `.cs`）

**现状（代码事实）**
```cpp
// FRegisterObject.cpp:85-109
static void AddToRootImplementation(const IManagedHandle InManagedHandle)
{
    if (const auto FoundObject = FCSharpEnvironment::GetEnvironment().GetObject(InManagedHandle))
    {
        FoundObject->AddToRoot();          // :89  无重复调用保护、无日志
    }
}

static void RemoveFromRootImplementation(const IManagedHandle InManagedHandle)
{
    if (const auto FoundObject = FCSharpEnvironment::GetEnvironment().GetObject(InManagedHandle))
    {
        FoundObject->RemoveFromRoot();     // :97  即使未 rooted 也照调
    }
}

static uint8 IsRootedImplementation(const IManagedHandle InManagedHandle)
{
    if (const auto FoundObject = FCSharpEnvironment::GetEnvironment().GetObject(InManagedHandle))
    {
        return FoundObject->IsRooted() ? 1 : 0;   // :105  提供了查询，但没有调用方约束
    }
    return 0;
}
// 注册：
.Function("AddToRoot",      AddToRootImplementation)       // :141
.Function("RemoveFromRoot", RemoveFromRootImplementation)  // :142
.Function("IsRooted",       IsRootedImplementation)        // :143
.Function("AddReference",   AddReferenceImplementation)    // :144
.Function("RemoveReference",RemoveReferenceImplementation) // :145
```

**调用上下文**
这是 C# 侧 `UObject` 静态 API（`FRegisterObject` 在静态初始化时注册到 `NAMESPACE_LIBRARY`）。任何 C# 代码都可以对任意 `UObject` 调 `AddToRoot()`。`AddReference`/`RemoveReference`（`FReferenceRegistry`，表 1 #15）是官方推荐的替代方案。

**问题**
1. `AddToRoot()` 是把对象放进全局 rootset，**GC 永不回收**，比 `AddReference`（同样保活但可通过 `RemoveReference` 精确解除且有 `AddUnique` 去重）更危险：`AddToRoot` 可被重复调用（UE 内部对重复 AddToRoot 是幂等的，但从审计角度无法发现"调用方重复加了 3 次但只减了 1 次"这种错误）。
2. 插件提供了 `IsRooted` 却没有任何地方用它做断言 —— 一个"防呆"API 被浪费。
3. 与 F-LIFE-008 构成同一问题的两面：**UObject 保活完全托付给托管侧，C++ 侧不设任何兜底**。

**建议**
- 在 `AddToRootImplementation` 里加 `#if !UE_BUILD_SHIPPING` 下的 `ensureMsgf(!FoundObject->IsRooted(), …)` 与 `UE_LOG` 计数，让重复 root 可见。
- 在 `RemoveFromRootImplementation` 里对未 rooted 的对象只告警不调用（避免 F-LIFE-011 那类反向不配对）。
- 文档层面明确：C++ 内部对象一律用 `AddReference`/`FGCObject`，`AddToRoot` 只保留给"进程期存活的全局单例"。
- 若要根治，从 C# 侧 API 中**移除** `AddToRoot`/`RemoveFromRoot`，只暴露 `AddReference`/`RemoveReference`。

**验证方式**
`grep -rn "AddToRoot\|RemoveFromRoot" Script/` 统计托管侧调用点并逐个检查配对；或在 `AddToRootImplementation` 加计数器并在热重载前后对比 `GUObjectArray` 的 rootset 大小。

#### [F-LIFE-012] `FClassCollector` 在 `FCoreDelegates::OnEndFrame` 上反复 `AddStatic` 并覆盖同一句柄，析构只解绑最后一个

- **类别**: 内存/资源泄漏 | 未定义行为
- **严重度**: **P2**(性能/隐患)
- **复核结论**: 确认（偏差：原文的"置信度低/若不去重则…"已被**引擎源码判定**——UE 的 `AddDelegateInstance` **不去重**，故原文的"若不去重"分支就是实际行为；另修正一处路径引用与一句后果描述，见复核证据）
- **可达性**: 活跃
- **复核证据**: 插件侧逐行确认：`ClassCollector.cpp:131-143`（`:133` `static auto LastRequestFrame`、`:135-138` 同帧去重、**`:142` `OnEndFrameDelegateHandle = FCoreDelegates::OnEndFrame.AddStatic(&FClassCollector::PopulateClassHierarchy);`**）、`:145-152`（`PopulateClassHierarchy` 体内**没有** `Remove`）、`:92-95`（析构只 `Remove(OnEndFrameDelegateHandle)` 一次）。绑定源 5 处确认：`:37`、`:40`、`:43`、`:48`、`:54` ✅。**引擎侧判定（直接把置信度从"低"提到"高"）**：① `Runtime/Core/Public/Delegates/MulticastDelegateBase.h:277-291` `AddDelegateInstance` **只做 `InvocationList.Emplace(...)`，既不查重也不移除等价实例**；② `:299-317` `RemoveDelegateInstance` 按句柄匹配、命中第一个即返回，且 `:312` 注释明写 "each delegate binding has a unique handle"；③ `Runtime/Core/Public/Delegates/DelegateInstancesImpl.h:38-42` 每个委托实例构造时 `Handle(FDelegateHandle::GenerateNewHandle)` → **每次 `AddStatic` 都产生全新句柄并追加一条绑定**。⇒ 每经过一个新的请求帧就永久多一条 `OnEndFrame` 绑定，析构只能清掉最后一条；其余条目在 `FClassCollector` 之后仍会逐帧执行。
- **级别变动**: 无（P2 维持：不发散为崩溃的**性能**问题——见下条修正——但属于无界累积）
- **非发现修正**: 原文问题段写"其余绑定在 `FClassCollector` 析构后继续调用已释放的静态函数/实例"——**"已释放的实例"不成立**：`FClassCollector` 的方法与数据成员**全部是 `static`**（`ClassCollector.h:42-58`、`:61-81`，`PopulateClassHierarchy` 在 `:42`），`AddStatic` 绑定的是无 `this` 的静态成员函数，故不存在悬垂 `this`。真实的危害是**重复调用**：每条遗留绑定都会执行一次 `PopulateClassHierarchy()` → `PopulateClassInMemory()`（内部是 `TObjectIterator<UClass>` **全类遍历**，`ClassCollector.cpp:158`）+ `PopulateClassByAsset()` + `RefreshAllNodes()`，代价随请求帧数 K 线性叠加到**每一帧**。
- **文件**: `Source/UnrealCSharpEditor/Private/NewClass/ClassCollector.cpp:131-143`（`AddStatic` 在 `:142`）、`:92-95`（析构解绑）、`:21`（`OnEndFrameDelegateHandle` 的**定义**）；声明在 `Source/UnrealCSharpEditor/Public/NewClass/ClassCollector.h:73`
- **函数**: `FClassCollector::RequestPopulateClassHierarchy()`、`FClassCollector::PopulateClassHierarchy()`、`~FClassCollector()`
- **置信度**: 高（已读到引擎侧源码：`MulticastDelegateBase.h:277-291,299-317` 与 `DelegateInstancesImpl.h:38-42` → `AddStatic` 不去重、每次新句柄；见复核证据）

**现状（代码事实）**
```cpp
// ClassCollector.cpp:131-143
void FClassCollector::RequestPopulateClassHierarchy()
{
    static auto LastRequestFrame = 0u;

    if (LastRequestFrame == GFrameNumber) { return; }

    LastRequestFrame = GFrameNumber;

    OnEndFrameDelegateHandle = FCoreDelegates::OnEndFrame.AddStatic(&FClassCollector::PopulateClassHierarchy);  // :142
}

// ClassCollector.cpp:145-152  —— 被 OnEndFrame 回调里，但没有 Remove
void FClassCollector::PopulateClassHierarchy()
{
    PopulateClassInMemory();
    PopulateClassByAsset();
    RefreshAllNodes();
}

// ClassCollector.cpp:92-95  析构：只 Remove 最后一次拿到的句柄
if (OnEndFrameDelegateHandle.IsValid())
{
    FCoreDelegates::OnEndFrame.Remove(OnEndFrameDelegateHandle);
}
```
`RequestPopulateClassHierarchy` 的绑定源共 5 个（`ClassCollector.cpp:37`、`:40`、`:43`、`:48`、`:54`，即 `OnDynamicClassUpdated`/`OnEndGenerator`/`ReloadCompleteDelegate`/`OnBlueprintCompiled`/`OnFilesLoaded`），每次触发且跨帧时都会再 `AddStatic` 一次。

**问题**
- 若 UE 的 `AddStatic` 对**等价**委托实例做去重（`TMulticastDelegate::AddDelegateInstance` 会先 `RemoveDelegateInstance` 等价实例并复用句柄），则本项**不是**缺陷：句柄重用使 `IsValid()` 与 `Remove` 仍然正确，只是这种写法依赖引擎实现细节、可读性差。
- 若不去重，则每帧新增一个 `OnEndFrame` 绑定，且析构只能移除最后一个 → 其余绑定在 `FClassCollector` 析构后**继续调用已释放的静态函数/实例**并逐帧累积。
- 由于我无法在本次分析范围内读到引擎 `MulticastDelegateInstance.h`，**置信度低**。特别注意叠加效应：`FClassCollector` 实例本身由 `SDynamicClassViewer::ClassCollector` 这个 `static TSharedPtr` 持有（F-LIFE-006），实际几乎不会析构，所以即使不去重，崩溃风险也被"永不析构"掩盖，表现为纯**逐帧累积的性能泄漏**。

**建议**
把 `AddStatic` 提到构造函数里做一次即可（`PopulateClassHierarchy` 本身可以幂等），或在 `PopulateClassHierarchy` 开头先 `FCoreDelegates::OnEndFrame.Remove(OnEndFrameDelegateHandle);` 再干活（一次性触发语义）。无论 UE 是否去重，改成显式一次性注册都更清晰：

```cpp
void FClassCollector::RequestPopulateClassHierarchy()
{
    static auto LastRequestFrame = 0u;
    if (LastRequestFrame == GFrameNumber) { return; }
    LastRequestFrame = GFrameNumber;

    if (OnEndFrameDelegateHandle.IsValid())
    {
        FCoreDelegates::OnEndFrame.Remove(OnEndFrameDelegateHandle);
    }
    OnEndFrameDelegateHandle = FCoreDelegates::OnEndFrame.AddStatic(&FClassCollector::PopulateClassHierarchy);
}
```

**验证方式**
在 `:142` 与 `PopulateClassHierarchy` 里各打一个计数器日志，编辑器里连续触发若干次资产变化，观察"添加次数"与"实际执行次数"是否 1:1；或读引擎 `Runtime/Core/Public/Delegates/MulticastDelegateInstance.h` 确认 `AddDelegateInstance` 是否去重。

#### [F-LIFE-017] `FRegisterInputComponent` / `FRegisterEnhancedInputComponent` 动态 `UFunction` 的 `AddToRoot` 依赖托管侧显式反注册

- **类别**: 内存/资源泄漏
- **严重度**: **P2**(性能/隐患)
- **复核结论**: 确认（偏差：`FRegisterEnhancedInputComponent.cpp` 的函数起点实为 `:68`（原文 `:70`，±2 行）；其余全部吻合）
- **可达性**: 活跃
- **复核证据**: `FRegisterInputComponent.cpp:283-319` 逐行确认：`:286-289` 双空指针守卫、`:291` `FindFunctionByName` 防重复、`:296` `NewObject<UFunction>(InClass, *InFunctionName, EObjectFlags::RF_Transient)`、`:298` `FUNC_BlueprintEvent`、`:302` `Bind()`、`:304` `StaticLink(true)`、`:306` `AddFunctionToFunctionMap`、`:308-310` 手工挂 `Children` 链、**`:312` `Function->AddToRoot();`**、`:314-318` `FCSharpEnvironment::...GetBind()->Bind(...)`。`FRegisterEnhancedInputComponent.cpp:68-104` 为**完全同构**的副本（`:97` AddToRoot）✅，两文件内 `RemoveFromRoot` 各 **0** 处。唯一的配对方 `FRegisterClass.cpp:12-33` 确认：**`:21` `if (Function->IsRooted())` → `:23` `Function->RemoveFromRoot();` 否则 `:27` `MarkAsGarbage();`**、`:30` `RemoveFunctionFromFunctionMap`、`:51` `.Function("RemoveFunction", RemoveFunctionImplementation)`。全插件 root grep（26 处）中 `FRegisterInputComponent.cpp:312` / `FRegisterEnhancedInputComponent.cpp:97` 是仅有的两处"Add 无同类 Remove"的运行时路径。
- **级别变动**: 无（P2 维持：泄漏/孤立 rooted 对象，依赖托管侧是否调用 `RemoveFunction`，非确定性崩溃）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterInputComponent.cpp:283-319`（`AddToRoot` 在 `:312`）、`Source/UnrealCSharp/Private/Domain/Interop/FRegisterEnhancedInputComponent.cpp:68-104`（`AddToRoot` 在 `:97`）
- **函数**: `BindFunction(...)`（匿名命名空间内的静态函数，两文件各一份）
- **置信度**: 中（未逐一核对 C# 侧是否在组件解绑时调用 `UClass.RemoveFunction`）

**现状（代码事实）**
```cpp
// FRegisterInputComponent.cpp:285-319（FRegisterEnhancedInputComponent.cpp:70-104 同构）
    if (InClass->FindFunctionByName(*InFunctionName)) { return; }          // :291  防重复

    const auto Function = NewObject<UFunction>(InClass, *InFunctionName, EObjectFlags::RF_Transient);  // :296

    Function->FunctionFlags = FUNC_BlueprintEvent;
    InProperty(Function);
    Function->Bind();
    Function->StaticLink(true);
    InClass->AddFunctionToFunctionMap(Function, *InFunctionName);          // :306
    Function->Next = InClass->Children;                                    // :308  手工挂到 Children 链
    InClass->Children = Function;                                          // :310
    Function->AddToRoot();                                                 // :312  ← 无对应 RemoveFromRoot
    FCSharpEnvironment::GetEnvironment().GetBind()->Bind(…GetClassDescriptor(InClass), InClass, Function);  // :314-318
```
唯一的 `RemoveFromRoot` 在 `FRegisterClass.cpp:23`，由 C# 侧 `UClass.RemoveFunction(handle, name)` 触发。

**调用上下文**
由 C# 侧在为一个 `UInputComponent` 子类绑定 `BindAction`/`BindAxis`（或 EnhancedInput 的 `BindAction`）时调用；`InClass` 是哈希/池化后的输入组件类（由 `GetDynamicBindingObject` 系列 API 得到）。触发点是游戏运行期每次遇到一个新的输入组件类。

**问题**
- `Function->AddToRoot()` 使该 `UFunction` **脱离其 outer `UClass` 的 GC 生命周期**。UE 的 GC 不会因为 "outer 被回收" 而回收 rooted 子对象 —— 因此当对应的 `UClass`（池化的动态输入组件类）被销毁/丢弃时，这些 `UFunction` 变成**孤立 rooted 对象**，永久占用内存，且 `Function->Next`/`InClass->Children` 链上的指针也随之失效。
- 只有在托管侧显式调用 `UClass.RemoveFunction` 时才会走到 `FRegisterClass.cpp:21-31` 的 `RemoveFromRoot`/`MarkAsGarbage`。C++ 侧**没有**任何兜底（例如在 `FClassRegistry` 析构或域卸载时遍历清理）。
- 量级：每个输入绑定函数约 0.5-2 KB（含 `FObjectProperty` 等参数属性，见 `FRegisterEnhancedInputComponent.cpp:111-119`）；一个输入组件类可能有 10-30 个绑定函数 → 每类约 10-60 KB。若运行期反复创建/丢弃输入组件类，会持续累积。

**建议**
1. 在 `FCSharpEnvironment::Deinitialize()` 或域卸载路径中，遍历 `FClassRegistry` 中已登记的所有 `FCSharpFunctionDescriptor`/`FClassDescriptor`，对由本函数创建的 `UFunction`（可通过 `RF_Transient` + `FUNC_BlueprintEvent` 特征识别，或引入一个显式的登记容器）统一 `RemoveFromRoot()`。
2. 更好的做法：不要 `AddToRoot` 单个 `UFunction`，而是把 `InClass` 本身保活（`AddReference` / `FGCObject`），`UFunction` 作为 `UClass::Children` 链上的子对象会随之存活 —— 这也消除了 `Function->Next = InClass->Children`（`:308-310`）手工维护链的脆弱性。
3. 至少加注释说明 `:312`/`:97` 的 root 依赖托管侧配对。

**验证方式**
grep `AddToRoot` 与 `RemoveFromRoot` 在 `FRegisterInputComponent.cpp`/`FRegisterEnhancedInputComponent.cpp`/`FRegisterClass.cpp` 的分布（前者 2 处 Add、0 处 Remove；后者 1 处 Remove）；或在 `:312` 后加 `UE_LOG` 计数，运行期反复加载/卸载带输入绑定的关卡，观察计数是否单调上升。

### P3

#### [F-LIFE-011] `FDynamicClassGenerator.cpp:893` 的 `RemoveFromRoot()` 没有匹配的 `AddToRoot()`

- **类别**: 逻辑不一致 | 可优化/可读性
- **严重度**: **P3**(风格/可读性)
- **复核结论**: 确认（偏差：无——行号与事实全部吻合；补强了"该文件 rootset 操作只有 5 处"这一断言）
- **可达性**: 活跃
- **复核证据**: `FDynamicClassGenerator.cpp:886-895` 逐行确认：`:889` `Rename(nullptr, GetTransientPackage(), REN_DoNotDirty | REN_DontCreateRedirectors)`、`:891` `ClearFlags(RF_Standalone)`、**`:893` `ComponentTemplate->RemoveFromRoot();`**、`:895` `RemoveOverridenComponentTemplate(ComponentKey)`。`RemoveComponentTemplate` 的完整签名与区间确认为 `:873-899`（`const UBlueprintGeneratedClass*, const USCS_Node*`，`#if WITH_EDITOR` 包裹 :872/:900），调用点在 **`:773`** ✅（位于 `:770 else if (PropertyClass != Node->ComponentClass)` 分支，随后 `:776` 调 `NewComponentTemplate`）。`NewComponentTemplate` 确认为 `:845-870`：`:850-851` `NewObject<UActorComponent>(GetTransientPackage(), InClass, NAME_None, RF_ArchetypeObject | RF_Public)`（**无 `AddToRoot`**）、`:868` `InNode->ComponentTemplate = ComponentTemplate;`。全插件 root grep（26 处）确认本文件只有 5 处：`:393`、`:419`、`:533`、`:540`、`:893` ✅（`:893` 是唯一孤儿）。
- **级别变动**: 无（P3 维持：`RemoveFromRoot` 作用于未 rooted 对象不会崩溃；引擎 `FUObjectArray::RemoveFromRoot` 对非 root 对象为空操作/告警）
- **文件**: `Source/UnrealCSharpCore/Private/Dynamic/FDynamicClassGenerator.cpp:886-895`（函数区间 `:873-899`）
- **函数**: `FDynamicClassGenerator::RemoveComponentTemplate(const UBlueprintGeneratedClass*, const USCS_Node*)`
- **置信度**: 中（未追踪 `UDynamicBlueprintExtension`/`UInheritableComponentHandler` 是否可能在别处 root 该模板对象）

**现状（代码事实）**
```cpp
// FDynamicClassGenerator.cpp:886-895
if (const auto ComponentTemplate = Blueprint->InheritableComponentHandler->GetOverridenComponentTemplate(ComponentKey))
{
    ComponentTemplate->Rename(nullptr, GetTransientPackage(), REN_DoNotDirty | REN_DontCreateRedirectors);   // :889

    ComponentTemplate->ClearFlags(RF_Standalone);                                                            // :891

    ComponentTemplate->RemoveFromRoot();                                                                     // :893  ← 无匹配 AddToRoot

    Blueprint->InheritableComponentHandler->RemoveOverridenComponentTemplate(ComponentKey);                   // :895
}
```
`grep -n "AddToRoot\|RemoveFromRoot\|ComponentTemplate" Source/UnrealCSharpCore/Private/Dynamic/FDynamicClassGenerator.cpp` 显示该文件的 rootset 操作只有 `:393`(Add Class)、`:419`(Add Blueprint)、`:533`(Remove Blueprint)、`:540`(Remove Class)、`:893`(Remove ComponentTemplate) —— **`:893` 是唯一的孤儿**。创建点 `NewComponentTemplate`（`:845-870`）用 `NewObject<UActorComponent>(GetTransientPackage(), InClass, NAME_None, RF_ArchetypeObject | RF_Public)`，既没有 `AddToRoot()`，`RF_ArchetypeObject`/`RF_Public` 也都不是 rootset 标志。

**调用上下文**
`RemoveComponentTemplate` 由 `:773` 调用（在生成器重建继承组件模板时）。

**问题**
`RemoveFromRoot()` 作用于未 rooted 的对象：UE 内部会走 `FUObjectArray::RemoveFromRoot`，在对象无 `RootSet`/`ClusterRoot` 标志时命中不了任何条目（部分版本会打 warning 或走 `ObjectsToRemoveFromRoot` 延迟告警路径），属**静默的逻辑不一致**。它不会崩溃，但会掩盖真正的配对错误，也让后来维护者误以为"这里加过 root"。对比 `FDynamicBlueprintExtensionScope.h:23`↔`:42` 与 `FDynamicClassGenerator.cpp:393`↔`:540` 的正确配对，这里是明显的复制粘贴残留。

**建议**
二选一并加注释说明意图：
- 若模板**确实需要**保活：在 `NewComponentTemplate`（`:850-851`）后补 `ComponentTemplate->AddToRoot();`，保持与 `:893` 配对；
- 若**不需要**保活（模板由 `InNode->ComponentTemplate`（`:868`）这个 UPROPERTY 引用保活，看起来是这种）：删除 `:893` 的 `RemoveFromRoot()`。
从 `:868` `InNode->ComponentTemplate = ComponentTemplate;`（UBPGS 的 UPROPERTY）看，第 2 种更可能正确，即 `:893` 应删除。

**验证方式**
在该处加 `ensureMsgf(ComponentTemplate->IsRooted(), TEXT("RemoveFromRoot on non-rooted template"))` 运行一次动态类重建，观察是否触发。

#### [F-LIFE-013] Cook 路径的 `OnFilesLoaded()` 只挂 lambda 不存句柄，`ShutdownModule` 无从解绑

- **类别**: 内存/资源泄漏
- **严重度**: **P3**(风格/可读性)
- **复核结论**: 确认（偏差：无；另**解决了原文 §7 遗留的"`Generator` 是否有命名空间级定义"之疑问**——它是类的静态成员函数，见复核证据）
- **可达性**: 活跃（但**仅 `IsRunningCookCommandlet()` 为真时**执行；编辑器普通会话不注册）
- **复核证据**: `UnrealCSharpEditor.cpp:158-180` 逐行确认（`:158` `if (IsRunningCookCommandlet())`、`:164-165` `LoadModulePtr<FAssetRegistryModule>`、**`:167` `AssetRegistryModule->Get().OnFilesLoaded().AddLambda([]() {...});`**、`:174` `Generator(ActiveTargetPlatforms[0]->IniPlatformName());`）；`ShutdownModule`（`:183-236`）逐行确认**无任何 `OnFilesLoaded` 移除**，`UnrealCSharpEditor.h:41-61` 的成员列表里也没有对应句柄 ✅。原文的存疑点已核实并闭环：lambda 内调用的 `Generator` 是**本模块的静态成员函数**（`UnrealCSharpEditor.h:34` `static void Generator(const FString& InPlatformName, bool bForceCompileInterop = false);`），因此 `[]` 无捕获也能编译、且**不携带 `this`** → 不存在悬垂 `this`。
- **级别变动**: 无（P3 维持：无 UAF 风险，仅"绑定永久驻留 + 与模块风格不一致"）
- **文件**: `Source/UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp:158-180`（`AddLambda` 在 `:167`）
- **函数**: `FUnrealCSharpEditorModule::StartupModule()`
- **置信度**: 高（代码事实清楚）

**现状（代码事实）**
```cpp
// UnrealCSharpEditor.cpp:158-180
if (IsRunningCookCommandlet())
{
    if (const auto UnrealCSharpEditorSetting = FUnrealCSharpFunctionLibrary::GetMutableDefaultSafe<UUnrealCSharpEditorSetting>();
        UnrealCSharpEditorSetting != nullptr && !UnrealCSharpEditorSetting->IsSkipGenerateScriptCode())
    {
        if (const auto AssetRegistryModule = FModuleManager::LoadModulePtr<FAssetRegistryModule>(TEXT("AssetRegistry")))
        {
            AssetRegistryModule->Get().OnFilesLoaded().AddLambda([]()      // :167  ← 返回值被丢弃
            {
                if (const auto TargetPlatformManager = GetTargetPlatformManager())
                {
                    if (const auto& ActiveTargetPlatforms = TargetPlatformManager->GetActiveTargetPlatforms();
                        !ActiveTargetPlatforms.IsEmpty())
                    {
                        Generator(ActiveTargetPlatforms[0]->IniPlatformName());
                    }
                }
            });
        }
    }
}
```
`ShutdownModule`（`:183-236`）里没有对应移除。

**调用上下文**
只在 `IsRunningCookCommandlet()` 为真的 cook 进程里执行一次。

**问题**
与 F-LIFE-001 同类的"句柄丢弃"，但**严重度远低**：捕获列表是 `[]`（无 `this`、无任何对象引用），因此**不存在 use-after-free 风险**；而且 cook 进程通常在生成代码后很快结束。剩下的问题只有：(1) 模块若被卸载且进程继续存活，`Generator(…)` 会在模块卸载后被调用（`:167` lambda 内部是静态自由函数 `Generator`，因此能编译通过 —— 需要确认它在 `UnrealCSharpEditor.cpp:314` 之外是否另有命名空间级定义，见 §7 未覆盖项）；(2) 与模块"所有全局绑定都显式配对"的风格不一致。

**建议**
把返回值存入新的成员 `FDelegateHandle OnFilesLoadedDelegateHandle;`（`UnrealCSharpEditor.h:41` 附近），并在 `ShutdownModule` 里按 `IsValid()` 移除；或改用 `FTSTicker`/一次性任务替代，避免长期全局绑定。

**验证方式**
`grep -n "OnFilesLoaded" Source/UnrealCSharpEditor/Private/UnrealCSharpEditor.cpp` → 1 处 Add、0 处 Remove。

#### [F-LIFE-014] `FUnrealCSharpPlayToolBar::Deinitialize()` 是空函数，与 `Initialize()` 不对称

- **类别**: 逻辑不一致 | 可读性
- **严重度**: **P3**(风格/可读性)
- **复核结论**: 确认
- **可达性**: 活跃
- **复核证据**: `UnrealCSharpPlayToolBar.cpp` 逐行确认：`:17-31` `Initialize()`（`:19` `UToolMenus::Get()->ExtendMenu("LevelEditor.LevelEditorToolBar.PlayToolBar")`、`:21` `AddSection("UnrealCSharpEditor")`、`:23-30` `AddEntry(FToolMenuEntry::InitComboButton(...))`、**`:26` `FOnGetContent::CreateRaw(this, &FUnrealCSharpPlayToolBar::GeneratePlayToolBarMenu)`**）、**`:33-35` `Deinitialize()` 空函数体** ✅。owner 机制确认：`UnrealCSharpEditor.cpp:270-284` `RegisterMenus()`（**`:273` `FToolMenuOwnerScoped OwnerScoped(this);`**、`:277` `UnrealCSharpPlayToolBar->Initialize();`）、`:253-256` `OnPostEngineInit()` → `RegisterMenus()`、`ShutdownModule` 里 `:188` `UToolMenus::UnregisterOwner(this);` 早于 `:226` `UnrealCSharpPlayToolBar->Deinitialize();` ✅；工具栏本身由 `TSharedPtr` 成员持有（`UnrealCSharpEditor.h:37`）✅。对照写法确认：`FUnrealCSharpBlueprintToolBar` 在 `Deinitialize()` 里显式 `Remove`（`UnrealCSharpBlueprintToolBar.cpp:38` 注册 / `:46` 解绑）。
- **级别变动**: 无（P3 维持：当前由 `UnregisterOwner` 兜住，属维护陷阱/不一致，非缺陷）
- **文件**: `Source/UnrealCSharpEditor/Private/ToolBar/UnrealCSharpPlayToolBar.cpp:17-35`
- **函数**: `FUnrealCSharpPlayToolBar::Initialize()` / `FUnrealCSharpPlayToolBar::Deinitialize()`
- **置信度**: 高

**现状（代码事实）**
```cpp
// UnrealCSharpPlayToolBar.cpp:17-31
void FUnrealCSharpPlayToolBar::Initialize()
{
    const auto PlayToolBar = UToolMenus::Get()->ExtendMenu("LevelEditor.LevelEditorToolBar.PlayToolBar");

    auto& EditorSection = PlayToolBar->AddSection("UnrealCSharpEditor");

    EditorSection.AddEntry(FToolMenuEntry::InitComboButton(
        "UnrealCSharpEditor",
        FUIAction(),
        FOnGetContent::CreateRaw(this, &FUnrealCSharpPlayToolBar::GeneratePlayToolBarMenu),   // :26  裸 this
        LOCTEXT("UnrealCSharpEditor_Label", "UnrealCSharp"),
        LOCTEXT("UnrealCSharpEditor_ToolTip", "UnrealCSharp"),
        FSlateIcon(FUnrealCSharpEditorStyle::GetStyleSetName(), "UnrealCSharpEditor.PluginAction")
    ));
}

// UnrealCSharpPlayToolBar.cpp:33-35  空实现
void FUnrealCSharpPlayToolBar::Deinitialize()
{
}
```
对照 `FUnrealCSharpBlueprintToolBar`：它在 `Deinitialize()`（`UnrealCSharpBlueprintToolBar.cpp:46`）里显式 `Remove` 了自己注册的 `OnEndGeneratorDelegateHandle`（`:38`），是**正确的对照写法**。

**调用上下文**
`Initialize()` 由模块在 `RegisterMenus()`（`UnrealCSharpEditor.cpp:277`，经 `OnPostEngineInit` :253-256 → :255）中调用；`Deinitialize()` 由 `ShutdownModule`（`:226`）调用。

**问题**
功能上目前**是安全的**：菜单项归属由 `FToolMenuOwnerScoped OwnerScoped(this)`（`UnrealCSharpEditor.cpp:273`，`this` 为模块）确定，`ShutdownModule` 的 `UToolMenus::UnregisterOwner(this)`（`:188`）会连带移除该 entry，从而释放 `FOnGetContent::CreateRaw(this=工具栏)` 里的裸指针；且工具栏本身由 `TSharedPtr` 成员（`UnrealCSharpEditor.h:37`）在模块析构时释放，顺序上 `UnregisterOwner` 先于 `TSharedPtr` 释放。因此**不是** P0/P1 级问题。
但它是一个明确的维护陷阱：`Deinitialize()` 空实现会让人以为"无需清理"，一旦后续有人在 `Initialize()` 里注册了**不属于该 owner** 的东西（例如直接 `AddRaw` 到全局委托），就会立刻变成 F-LIFE-001 那类缺陷。

**建议**
在 `Deinitialize()` 里做与 `Initialize()` 对称的清理：
```cpp
void FUnrealCSharpPlayToolBar::Deinitialize()
{
    UToolMenus::UnregisterOwner(this);   // 若 Initialize 改用工具栏自身作为 owner
}
```
或至少加注释：`// 菜单项由模块的 FToolMenuOwnerScoped 拥有，UToolMenus::UnregisterOwner(Module) 已负责清理`。更稳的做法是把 `Initialize()` 里的 `FToolMenuOwnerScoped` 放进 `FUnrealCSharpPlayToolBar::Initialize()`，让 owner 与注册者一致。

**验证方式**
`grep -n "Deinitialize" Source/UnrealCSharpEditor/Private/ToolBar/UnrealCSharpPlayToolBar.cpp Source/UnrealCSharpEditor/Private/ToolBar/UnrealCSharpBlueprintToolBar.cpp` 对比两者实现。

#### [F-LIFE-018] `FRegisterMulticastDelegate` 重复注册 `Contains` 绑定

- **类别**: 死代码 | 可读性
- **严重度**: **P3**(风格/可读性)
- **复核结论**: 确认
- **可达性**: 活跃
- **复核证据**: `FRegisterMulticastDelegate.cpp:185-202` 逐行确认：`:190` `.Function("Contains", ContainsImplementation)` 与 **`:192` `.Function("Contains", ContainsImplementation)` 完全重复**（同一实现指针），中间 `:191` 是 `IsBound` ✅。验证方式给出的 grep 复核成立：工具 grep `\.Function\("Contains"` 在该文件命中 **2** 处。注册器实例确认在 `:205` `[[maybe_unused]] FRegisterMulticastDelegate RegisterMulticastDelegate;`（原文写"`:203` 附近"，实际 `:205`，±2 行）。
- **级别变动**: 无（P3 维持：重复绑定无功能风险，属复制粘贴遗留）
- **文件**: `Source/UnrealCSharp/Private/Domain/Interop/FRegisterMulticastDelegate.cpp:190,192`
- **函数**: `FRegisterMulticastDelegate()`（匿名命名空间内的静态对象构造函数）
- **置信度**: 高

**现状（代码事实）**
```cpp
// FRegisterMulticastDelegate.cpp:188-198
.Function("Register",        RegisterImplementation)          // :188
.Function("UnRegister",      UnRegisterImplementation)        // :189
.Function("Contains",        ContainsImplementation)          // :190  ← 第一次
.Function("IsBound",         IsBoundImplementation)           // :191
.Function("Contains",        ContainsImplementation)          // :192  ← 重复注册，同一实现
.Function("Add",             AddImplementation)               // :193
.Function("AddUnique",       AddUniqueImplementation)         // :194
.Function("Remove",          RemoveImplementation)            // :195
.Function("RemoveAll",       RemoveAllImplementation)         // :196
.Function("Clear",           ClearImplementation)             // :197
```

**调用上下文**
`FRegisterMulticastDelegate` 是静态初始化注册器（同文件 `:203` 附近的 `[[maybe_unused]]` 实例），进程启动时构造一次。

**问题**
纯重复绑定：`TBindingClassBuilder::Function(name, …)` 对同名函数会再插入一条记录到 C# 侧的绑定表，若下游按名字查找，后写覆盖先写则无影响；若按列表遍历（例如生成 C# 绑定代码时逐个 emit），会**生成两个同名 `Contains` 重载/重复方法**，导致 C# 侧编译告警或绑定表膨胀。无功能风险，明确是复制粘贴遗留。

**建议**
删除 `:192` 这一行。同时检查其它 `FRegister*.cpp` 是否有同类重复（本次仅在本文件发现）。

**验证方式**
`grep -n '\.Function("Contains"' Source/UnrealCSharp/Private/Domain/Interop/FRegisterMulticastDelegate.cpp` → 2 处命中。

---

#### [F-LIFE-019] `FCSharpEnvironment::OnUObjectArrayShutdown()` 是空实现 —— UObject 系统关闭时插件不做任何清理

- **类别**: 内存/资源泄漏 | 死代码
- **严重度**: **P3**(风格/可读性)
- **复核结论**: 确认（偏差：验证方式里的 grep 命中数 **4 应为 5**，见复核证据）
- **可达性**: 活跃（钩子确实会被引擎调用如上，但"不清理"的坏后果被正常退出路径掩盖）
- **复核证据**: `FCSharpEnvironment.cpp:323-325` 确认是**完全空**的函数体 ✅；`FUObjectListener.cpp:51-54` 确认转发（`:53` `FCSharpEnvironment::GetEnvironment().OnUObjectArrayShutdown();`）✅；查询/注册点 `FUObjectListener.cpp:29,31`（`GUObjectArray.AddUObjectCreateListener(this)` / `AddUObjectDeleteListener(this)`）与注销 `:36,38` ✅；声明 `FCSharpEnvironment.h:36` ✅、接口覆写 `FUObjectListener.h:20` ✅（`FUObjectListener.h:3` 私有继承 `FUObjectArray::FUObjectCreateListener, FUObjectDeleteListener`）。**grep 重跑**（模式 `OnUObjectArrayShutdown`，`Source/` 全模块）→ 命中 **5** 处：`FUObjectListener.h:20`（声明）、`FCSharpEnvironment.h:36`（声明）、`FCSharpEnvironment.cpp:323`（空定义）、`FUObjectListener.cpp:51`（转发定义）、`FUObjectListener.cpp:53`（转发调用）——原文写"4 处命中（1 声明 + 1 空定义 + 1 转发 + 1 声明）"漏计了 `FUObjectListener.cpp:51` 这个定义，**修正为 5 处**，但"无实质逻辑"的结论不变 ✅。正常退出路径确认：`FEngineListener.cpp:17` 挂 `OnPreExit`、`:58-61` `OnPreExit()` → `SetActive(false)`、`:78` `FUnrealCSharpCoreModule::Get().Deactivate()`；`UnrealCSharp.cpp:19-34` `ShutdownModule` 确实**只解绑两个委托、不调 `Deactivate()`** ✅（原文建议成立）。
- **级别变动**: 无（P3 维持：未实现的兜底钩子，正常退出路径已被 `OnPreExit`→`Deactivate()` 覆盖）
- **文件**: `Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:323-325`、`Source/UnrealCSharp/Private/Listener/FUObjectListener.cpp:51-54`、`Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h:36`
- **函数**: `FCSharpEnvironment::OnUObjectArrayShutdown()`
- **置信度**: 高

**现状（代码事实）**
```cpp
// FUObjectListener.cpp:51-54
void FUObjectListener::OnUObjectArrayShutdown()
{
    FCSharpEnvironment::GetEnvironment().OnUObjectArrayShutdown();
}

// FCSharpEnvironment.cpp:323-325  —— 空函数体
void FCSharpEnvironment::OnUObjectArrayShutdown()
{
}
```

**调用上下文**
`FUObjectListener` 以 `GUObjectArray::AddUObjectCreateListener/AddUObjectDeleteListener`（`FUObjectListener.cpp:29,31`）注册；`OnUObjectArrayShutdown` 是 `FUObjectArray::FUObjectCreateListener` 接口的一部分，在 `FUObjectArray::ShutdownUObjectArray()` 中被调用（引擎退出/`GExitPurge` 阶段）。

**问题**
这是任务书第 1 类里"**UObject 系统关闭时是否清理了这些根引用与缓存**"这个问题的直接答案：**没有**。
`GUObjectArray` 关闭意味着随后所有 `UObject` 都会被销毁。此时插件的以下状态仍然存在且**不再有任何机会被清理**：
- `FReferenceRegistry::ObjectArray` 里的 `TObjectPtr<UObject>`（若模块从未 `Deactivate()`，即"从未激活过就直接退出"或"激活后未走 InActive 路径"）；
- 4 组动态类型静态注册表里的裸 `UClass*`/`UEnum*`/`UDynamicScriptStruct*`（F-LIFE-006）；
- `FDynamicDependencyGraph::NodeArray`（F-LIFE-016）；
- `FCSharpBind::NotOverrideTypes`（`FCSharpBind.h:86`，`TSet<TWeakObjectPtr<UStruct>>`，本就是 weak，无悬挂风险）。

**说明严重度为什么只有 P3**：正常退出路径上 `FEngineListener::OnPreExit`（`FEngineListener.cpp:17,58-61`）→ `SetActive(false)` → `FUnrealCSharpCoreModule::Get().Deactivate()`（`:78`）会先走一遍完整的 `Deinitialize()`，所以这条 `OnUObjectArrayShutdown` 路径在实践中几乎不会被"最后一道防线"依赖。因此它更像一个**未实现的钩子/死行为**，而不是可观测的泄漏源。

**建议**
在 `OnUObjectArrayShutdown()` 里补最末一段兜底：
```cpp
void FCSharpEnvironment::OnUObjectArrayShutdown()
{
    // UObject 系统即将关闭，确保不再持有任何 UObject 相关状态
    Deinitialize();   // 幂等：内部对每个指针都做了 != nullptr 判断（:177-268）
}
```
并在 `FUnrealCSharpModule::ShutdownModule()`（`UnrealCSharp.cpp:19-34`）里也显式 `FUnrealCSharpCoreModule::Get().Deactivate()`（当前只在 `FEngineListener::OnPreExit` 里做），使"编辑器是否退出、是否 PIE、是否 cook"等所有路径都能走到清理。

**验证方式**
`grep -n "OnUObjectArrayShutdown" Source/ -r` → 4 处命中（1 声明 + 1 空定义 + 1 转发 + 1 声明），无实质逻辑；在该函数内加 `UE_LOG` 后在编辑器中退出，观察是否被调用、以及调用时 `ReferenceRegistry` 是否仍非空。



## 5. 域重载（HotReload）泄漏清单

### 5.1 热重载完整链路（已逐跳核对，作为下面表格的公共上下文）

```
C# 编译成功
→ Source/Compiler/Private/FCSharpCompilerRunnable.cpp:225   FUnrealCSharpCoreModule::Deactivate()
→ Source/UnrealCSharpCore/Private/UnrealCSharpCore.cpp:29-37  State=Inactive; 广播 OnUnrealCSharpCoreModuleInActive
→ Source/UnrealCSharp/Private/UnrealCSharp.cpp:50-53          广播 OnUnrealCSharpModuleInActive
→ Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:332-335  OnUnrealCSharpModuleInActive() → Deinitialize()
→ Source/UnrealCSharp/Private/Listener/FUObjectListener.cpp:34-39      从 GUObjectArray 摘除 create/delete listener
→ FCSharpEnvironment.cpp:147-269  依次 delete Optional/Binding/String/Multi/Delegate/Container/Struct/Object/
                                  Reference/Class registry → CSharpBind → DynamicRegistry → **最后 delete Domain**
→ Source/UnrealCSharp/Private/Domain/FDomain.cpp:15-18  ~FDomain() → Deinitialize()
→ Source/UnrealCSharpCore/Private/Domain/Script/FScriptDomainFactory.cpp:47-59  Deinitialize(); delete; Set(nullptr)
→ Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRDomain.cpp:128-162  Deinitialize() → UnloadAssembly()
→ FCoreCLRDomain.cpp:240-270  FReflectionRegistry::Get().Deinitialize() → AssemblyLoaderUnloadFn()
→ Script/Interop/AssemblyLoader/AssemblyLoader.cs:31-61  HandleData.Clear(); TypeBridge.Clear(); Context.Unload()
→ Source/Compiler/Private/FCSharpCompilerRunnable.cpp:229   FUnrealCSharpCoreModule::Activate()
→ Source/UnrealCSharp/Private/Environment/FCSharpEnvironment.cpp:57-145  Initialize()（重建全部 registry + 新 Domain）
→ FDynamicGenerator::Generator()（FDynamicGenerator.cpp:18）→ FDynamicGeneratorCore::Generator()（:34）
→ 各动态生成器 → SetMetaData → **消费 FDynamicGeneratorCore.cpp:1040 等 6 个悬垂的静态数组（F-LIFE-010）**
```

### 5.2 泄漏/悬垂清单

| # | 每次热重载泄漏什么 | 证据（文件:行） | 量级估计 | 严重度 |
|---|---|---|---|---|
| 1 | **悬垂（非增长）**：`FDynamicGeneratorCore` 的 6 个 `static TArray<FClassReflection*>` 仍指向 `FReflectionRegistry::Deinitialize()` 刚 `delete` 掉的反射对象，第二次重载起 `SetMetaData` 解引用已释放内存 | `Source/UnrealCSharpCore/Private/Dynamic/FDynamicGeneratorCore.cpp:1040,1071,1086,1099,1112,1205`（数组）+ `:792,799,807,855,874`（消费）；`Source/UnrealCSharpCore/Private/Reflection/FReflectionRegistry.cpp:873`（`delete Class`） | 不增加内存占用，但读取已释放堆 → 元数据静默错配（`FReflection.cpp:18-21` 仅按指针查询）；仅当悬垂指针与新建对象同址时才会走到 `FDynamicGeneratorCore.cpp:782` 的真解引用 | **P1**（F-LIFE-010，原 P0 已校正） |
| 2 | `FDynamicDependencyGraph::NodeArray` 每次 Generator 运行追加 N 个 `FNode`（每个含 `FName`/`FString` + `TArray<FDependency>` + `TArray<TFunction<void()>>`），永不 `Empty()` | `Source/UnrealCSharpCore/Private/Dynamic/FDynamicDependencyGraph.cpp:13`（`NodeArray.Emplace`）、`:6`（单例）；grep `Empty()`/`Reset()` 命中 **0** | N=2000 动态类型 × 约 100-300 B ≈ **0.2-0.6 MB / 次重载**；50 次 ≈ 10-30 MB | **P2**（F-LIFE-016，原 P1 已校正） |
| 3 | 若 `Context.Unload()` 后 2 秒内 ALC 未被回收，则 `Context = null`（`finally`）导致**旧 ALC 再无卸载机会**，下次加载新建一个 collectible ALC | `Script/Interop/AssemblyLoader/AssemblyLoader.cs:39`（WeakReference）、`:47`（`Context = null`）、`:50`（`TimeLimit = 2000`）、`:54`（超时即退出循环，无日志）；`UnrealAssemblyLoadContext.cs:8`（`isCollectible: true`） | 每个泄漏的 ALC 承载全部脚本程序集 + 元数据 + JIT 代码，**数十 MB / 次**（仅在未回收时发生） | **P2**（F-LIFE-015，原 P1 已校正；且当前工程 LeanCLR 下 `Context` 恒为 null → 潜伏） |
| 4 | 动态类型静态注册表只增不减：`NamespaceMap`(×4 生成器) / `DynamicClassMap` / `DynamicStructMap`+`DynamicStructSet` / `DynamicEnumMap`+`DynamicEnumSet` / `DynamicInterfaceMap`+`DynamicInterfaceSet`，且键是**裸 `UClass*`/`UEnum*`/`UDynamicScriptStruct*`**，被 `MarkAsGarbage()` 的旧对象仍留在表中 | `FDynamicClassGenerator.h:127,131`；`FDynamicStructGenerator.h:48,50,52`；`FDynamicEnumGenerator.h:42,44,46`；`FDynamicInterfaceGenerator.h:45,47,49`；定义于各 `.cpp`：`FDynamicClassGenerator.cpp:31,35`、`FDynamicStructGenerator.cpp:17,19,21`、`FDynamicEnumGenerator.cpp:16,18,20`、`FDynamicInterfaceGenerator.cpp:15,17,19` | 每个动态类型约 100-200 B（4 个容器合计）；1000 类型 ≈ 0.1-0.2 MB，**有界但永不释放 + 可能持有已回收裸指针** | **P1**（F-LIFE-006） |
| 5 | 退役（改名/删除）的动态 `UEnum` 与接口 `UClass` 永久留在 rootset | `FDynamicEnumGenerator.cpp:201`（Add，无 Remove）；`FDynamicInterfaceGenerator.cpp:263`（Add，无 Remove） | 枚举 0.1-1 KB/个、接口类 1-5 KB/个；**只在类型名退役时 +1**（同名重建走原地复用：`FDynamicEnumGenerator.cpp:90-102`、`FDynamicInterfaceGenerator.cpp:107-114`），因此**不随重载次数线性增长** | P2（F-LIFE-004 / F-LIFE-005） |
| 6 | 若热重载走 `MarkOutdated()` 而非立即 `Deactivate()`（`FCSharpCompilerRunnable.cpp:235`），则**本次不重建环境**，等到下一次 PIE 前 `FEngineListener::OnPreBeginPIE`（`FEngineListener.cpp:38-43`）才走完整 Deactivate/Activate —— 期间旧的 C++/C# 混合状态必须继续可用；此路径下**没有**泄漏（见下行正面结论） | `FCSharpCompilerRunnable.cpp:209-236`、`FEngineListener.cpp:34-44` | — | — |
| 7 | **正面结论（未发现泄漏）**：`FCSharpEnvironment::Deinitialize()` 的销毁顺序是**刻意正确的** —— `Domain` 被放在最后删除（`FCSharpEnvironment.cpp:263-268`），因此所有 registry 的 `FDomain::GCHandle_Free`（`FDomain.cpp:94-100` → `ScriptDomain->Free`）都在脚本域仍然存活时执行；`FClassRegistry`（持有 `FCSharpFunctionRegister`，其析构会 `RemoveFromRoot`/`MarkAsGarbage`：`FCSharpFunctionRegister.cpp:59-66`）与 `FDelegateRegistry`（持有 `FDelegateHelper`，其析构会 `RemoveFromRoot`：`FDelegateHelper.cpp:38`）都在域销毁前清理完毕 | `FCSharpEnvironment.cpp:147-269`、`FClassRegistry.h:62`、`FCSharpFunctionRegister.cpp:29-71`、`FDelegateRegistry.cpp:18-55` | **0** | — |
| 8 | **正面结论（未发现泄漏）**：`IsScriptDomain::Set(nullptr)` 在 `delete` 之后执行，避免野指针（`FScriptDomainFactory.cpp:56-58`）；`HandleData.Clear()` 会逐句柄 `Free()` 并清空字典（`HandleData.cs:129-145`）；`TypeBridge.Clear()` 在卸载前调用（`AssemblyLoader.cs:37`）；`FDynamicGeneratorCore::DynamicMap` 在 `EndCodeAnalysisGenerator()` 里 `Empty()`（`FDynamicGeneratorCore.cpp:29-32`），配对正确 | 同左 | **0** | — |
| 9 | `FReferenceRegistry::ObjectArray`（`TObjectPtr` 强引用）中的条目：在**走 Deactivate 的热重载路径**上随 `~FReferenceRegistry()`（`FReferenceRegistry.cpp:7-20`）清空；但**没有热重载以外的兜底**，若托管侧忘记 `RemoveReference`，对象在非重载期间会被永久保活 | `FReferenceRegistry.cpp:59-69`、`FReferenceRegistry.h:29` | 取决于托管侧纪律，无法静态估量 | **P2**（F-LIFE-008，原 P1 已校正） |

### 5.3 关于 `AssemblyLoadContext` 的审计结论（对应任务书第 4 类的三个检查点）

| 检查点 | 结论 | 证据 |
|---|---|---|
| `AssemblyLoadContext.Unload()` 是否被调用 | ✅ 被调用 | `AssemblyLoader.cs:43`；C++ 侧经 `AssemblyLoaderUnloadFn()`（`FCoreCLRDomain.cpp:244-247`）→ 绑定实体在 `Interop` 程序集里（`RegisterInterop`，`FCoreCLRDomain.cpp:281-294`） |
| 是否 `isCollectible` | ✅ 是 | `UnrealAssemblyLoadContext.cs:7-8` `: AssemblyLoadContext(name: "UnrealAssemblyLoadContext", isCollectible: true)` |
| 旧域 GCHandle 是否释放 | ✅ 释放（两处冗余且互补）| C++ 侧：各 registry 在 `FCSharpEnvironment::Deinitialize()` 里 `FDomain::GCHandle_Free`（`FObjectRegistry.cpp:24,90,108`、`FDelegateRegistry.cpp:29,47`、`FStructRegistry.cpp:25,117`、`FStringRegistry.cpp:22,40,59,79,98`、`FMultiRegistry.cpp:22,40,58,76,94,112`、`FContainerRegistry.cpp:29,47,65`、`FBindingRegistry.cpp:23,60`、`FOptionalRegistry.cpp:31`、`FDelegateRegistry.inl:75`、`FContainerRegistry.inl:76`、`FStringRegistry.inl:76`、`FManagedFunctionDescriptor.cpp:64`、`TMethodHelper.inl:92`）；C# 侧：`HandleData.Clear()`（`HandleData.cs:129-145`）兜底全部 |
| 旧域 C# 对象包装是否释放 | ⚠️ 依赖上面两条都成功；**若 `Context` 未在 2 秒内回收（F-LIFE-015），`Context = null` 后旧 ALC 内的 `Type`/`MethodInfo` 仍可能被 `TypeBridge`/静态字段引用** | `AssemblyLoader.cs:47,54` |
| 注册表条目是否释放 | ✅ C++ 侧全部 registry 显式 `delete`（`FCSharpEnvironment.cpp:177-268`）；`FCSharpEnvironment`/`FCSharpBind`/`FDynamicRegistry`/`FUObjectListener`/`FDomain` 的委托句柄也全部配对（表 3 #1-#11） | 见左列 |
| 动态生成的 `UClass`/`UStruct`/`UEnum` 是否被释放 | ⚠️ **部分是**：`UClass`/`UBlueprint`（`FDynamicClassGenerator.cpp:533,540`）与 `UDynamicScriptStruct`（`FDynamicStructGenerator.cpp:380`）在 `ReInstance` 路径上正确 `RemoveFromRoot()` + `MarkAsGarbage()`；**`UEnum`（`FDynamicEnumGenerator.cpp:201`）与接口 `UClass`（`FDynamicInterfaceGenerator.cpp:263`）没有**；另有静态注册表与 `FDynamicDependencyGraph` 的裸指针残留 | 见 5.2 #4/#5 |

**域重载总评**：C++ 侧的"销毁顺序"设计（Domain 最后删）是对的，注册表与委托句柄的配对也基本完整；真正的缺陷集中在**"跨重载存活的 `static` 缓存里保存了随域销毁的裸指针"**这一模式上 —— 已确认 4 处（6 个元数据数组、`FDynamicDependencyGraph::NodeArray`、4 组动态类型注册表、`FDelegateWrapper::Method`），以及 C# 侧的 ALC 回收失败静默化（F-LIFE-015）。

---

## 6. 死代码 / 无效代码清单（仅限本审计关注的 UObject 生命周期与绑定范畴）

判定方法：grep 函数名（去类名限定），统计**全插件**（`Source/` 7 个模块 + `Script/`）命中数；`==2`（1 声明 + 1 定义）视为强死代码嫌疑。

| 符号 | 声明位置 | grep 命中数 | 判定 | 证据 |
|---|---|---|---|---|
| `FCSharpEnvironment::OnUObjectArrayShutdown()` | `Source/UnrealCSharp/Public/Environment/FCSharpEnvironment.h:36` | **5**（`FCSharpEnvironment.h:36` 声明、`FCSharpEnvironment.cpp:323` 空定义、`FUObjectListener.h:20` 接口覆写声明、`FUObjectListener.cpp:51` 转发**定义**、`FUObjectListener.cpp:53` 转发调用；原文写 4，grep 重跑修正为 5） | **空实现的钩子 → 无效代码** | 唯一调用方 `FUObjectListener.cpp:53` 转发的目标函数体为空（`FCSharpEnvironment.cpp:323-325`）。见 F-LIFE-019 |
| `FUnrealCSharpPlayToolBar::Deinitialize()` | `Source/UnrealCSharpEditor/Public/ToolBar/UnrealCSharpPlayToolBar.h`（对应 `.cpp:33-35`） | **3**（`.h` 声明、`.cpp:33` 定义、`UnrealCSharpEditor.cpp:226` 调用） | **空实现但被调用 → 无效代码** | 与 `FUnrealCSharpBlueprintToolBar::Deinitialize()`（`UnrealCSharpBlueprintToolBar.cpp:46` 有实际 `Remove`）不对称。见 F-LIFE-014 |
| `.Function("Contains", ContainsImplementation)`（`FRegisterMulticastDelegate`） | `Source/UnrealCSharp/Private/Domain/Interop/FRegisterMulticastDelegate.cpp:192` | `grep -n '\.Function("Contains"'` → **2**（`:190`、`:192`） | **重复注册（死代码行）** | 两行参数完全相同，属复制粘贴遗留。见 F-LIFE-018 |
| `FDynamicClassGenerator::RemoveComponentTemplate` 中的 `RemoveFromRoot()` | `Source/UnrealCSharpCore/Private/Dynamic/FDynamicClassGenerator.cpp:893` | `RemoveFromRoot` 在本文件 **3** 处（`:533,:540,:893`）；`AddToRoot` **2** 处（`:393,:419`） | **反向未配对（无死对象可操作）** | `NewComponentTemplate`（`:850`）无 `AddToRoot`。见 F-LIFE-011 |
| `FCSharpFunctionRegister`（拷贝构造） | `Source/UnrealCSharp/Public/Reflection/Function/FCSharpFunctionRegister.h:11` | `FCSharpFunctionRegister(const FCSharpFunctionRegister&) = default;`；全插件 `FCSharpFunctionRegister` 命中 **19** | **⚠️ 高危隐患（非死代码）** | 该类**有析构函数**（`.cpp:29-71`，会执行 `RemoveFromRoot`/`MarkAsGarbage`/`RemoveFunctionFromFunctionMap`）且同时**默认了拷贝构造** → 一旦被拷贝，两个实例都会在析构时对同一 `UFunction` 执行移除操作（**double free 语义**：第二次 `RemoveFromRoot` 作用于已 unroot 的对象，或对已 `MarkAsGarbage` 的对象再次操作）。当前唯一使用点是 `FCSharpFunctionDescriptor`（`FCSharpFunctionDescriptor.h:24` 成员）与 `FClassRegistry::CSharpFunctionHashMap`（`FClassRegistry.h:62` 的 `TTuple<..., FCSharpFunctionRegister>`），路径上以移动构造（`.cpp:14-27` 已实现，会 `Reset()` 源对象）传入，**当前未触发**。建议显式 `= delete` 拷贝构造。置信度：中（未能穷举 `TTuple` 的所有内部拷贝点） |
| `FUnrealCSharpPlayToolBar` 的空 `Deinitialize` 之外的 `TODO` 分支 | `Source/UnrealCSharpCore/Private/Dynamic/FDynamicDependencyGraph.cpp:86,101,210` | 3 处 `// @TODO` | **未实现的分支** | `:86`/`:101` 是"已完成节点"与"已在集合中"的空分支，`:210` 是"既未 pending 也未 completed"时的空分支 —— 会导致节点被永久留在 Pending/Initial 状态，进而成为 F-LIFE-016 里"上一轮遗留节点"的来源 |

## 7. 未覆盖 / 存疑项

以下部分**未验证**，请在引用相关结论时注意：

1. **`Script/` 侧 C# 运行时未系统阅读。** 本报告对 `Script/Interop/AssemblyLoader/*.cs`（`AssemblyLoader.cs`、`UnrealAssemblyLoadContext.cs`，两个文件全读）和 `Script/Interop/Handle/HandleData.cs`（全读）做了逐行审计；但 `Script/UE/`、`Script/SourceGenerator/`、`Script/Weavers/`、`Script/CodeAnalysis/` **未读**。因此：
   - **F-LIFE-009** 中"托管侧是否成对调用 `AddToRoot`/`RemoveFromRoot`/`AddReference`/`RemoveReference`"未经核对（`grep -rn "AddToRoot" Script/` 未执行）。这是判断该条严重度（当前 P2）能否升到 P1 的关键。
   - **F-LIFE-008** 同上。
   - **F-LIFE-002** —— 原文曾写"C# 侧是否在包装对象 GC 时自动调用 `FDelegate.UnRegister` 未验证，若没有兜底则该条应保持 P0"。**已证伪该条缺陷**：撤销依据**不依赖** C# 侧纪律，而依赖 C++ 侧的确定性守卫与销毁顺序（`DelegateHandler.cpp:8` 的 `DelegateDescriptor != nullptr` 守卫 + `FDelegateHelper::Deinitialize()`（`FDelegateHelper.cpp:32-42`）+ `FCSharpEnvironment.cpp:207-212` 先于 `:263-268`），故与 C# 侧是否自动反注册无关。
2. ~~**`FCSharpDelegateDescriptor` 的基类链未完全展开 / 表 2 的 member 7/8 依赖推理。**~~ → **已解决**：`FCSharpDelegateDescriptor : FManagedFunctionDescriptor : FFunctionDescriptor`（`FCSharpDelegateDescriptor.h:7`）确认；`FFunctionDescriptor::Function` 是 `TWeakObjectPtr<UFunction>`（`FFunctionDescriptor.h:23`）、`PropertyDescriptors` 是 `TArray<FPropertyDescriptor*>`（`:25`）；`FPropertyDescriptor` 基类本身**不含** `Property` 成员（`FPropertyDescriptor.h:20-80`），真正的裸指针成员在 `TPropertyDescriptor<T, IsPrimitive>` 里：`T* Property{}`（`TPropertyDescriptor.inl:44`，`T` 派生自 `FProperty`，见协变 `GetProperty()` `:17-20`）与 `FClassReflection* Class{}`（`:46`）。表 2 row 7 已据此改写为**直读证据**，不再依赖推理。
3. ~~**`FEditorListener` 的模块卸载时序未验证。**~~ → **已解决（改为已核实）**：引擎源码确认 `IModuleInterface::SupportsAutomaticShutdown()` 默认 `return true`（`Runtime/Core/Public/Modules/ModuleInterface.h:74-77`，本模块未覆写），`UnloadModulesAtShutdown` 按**后加载先卸载**排序（`Runtime/Core/Private/Modules/ModuleManager.cpp:1224-1227`、`:1256-1262`），而 AssetRegistry 早于本插件模块加载 → 编辑器模块先死、AssetRegistry 后死；同时 AssetRegistry 的资产事件直到 `UAssetRegistryImpl::FinishDestroy()` 才 `Clear()`（`Runtime/AssetRegistry/Private/AssetRegistry.cpp:1858-1873`）。因此"模块卸载时 AssetRegistry 仍存活"**成立**，F-LIFE-001 维持 P0 的关键前提已由引擎证据支撑；但悬垂窗口内是否真有资产事件广播仍未实测，故 F-LIFE-001 的复核结论写为"部分确认"。
4. **UE 引擎源码：已按需补读。** 以下两条已由引擎源码**判定并闭环**：
   - **F-LIFE-012**：`TMulticastDelegate` 的 `AddDelegateInstance`（`Runtime/Core/Public/Delegates/MulticastDelegateBase.h:277-291`）**不去重**、只 `InvocationList.Emplace(...)`；每个委托实例构造时 `Handle(FDelegateHandle::GenerateNewHandle)`（`Runtime/Core/Public/Delegates/DelegateInstancesImpl.h:38-42`）→ 每次 `AddStatic` 都新增一条绑定与一个新句柄；`RemoveDelegateInstance`（`MulticastDelegateBase.h:299-317`）只移除首个句柄匹配项。故 F-LIFE-012 从"疑似"升级为**确认**（置信度 低→高）。
   - **F-LIFE-007**：`UnregisterDirectoryChangedCallback_Handle` 需"目录键 + 句柄"双匹配（`Developer/DirectoryWatcher/Private/Windows/DirectoryWatcherWindows.cpp:82-109`，`:91` `RemoveDelegate(InHandle)` / `:108` `return false`；代理层 `DirectoryWatcherProxy.cpp:43-45`）→ 句柄覆盖后前 N-1 个目录确实无法注销。
   - 仍未读：`FUObjectArray::RemoveFromRoot` 对未 rooted 对象是 warning 还是 silent no-op（F-LIFE-011 的 P3 定级理由，两种情况下都不会崩溃，故不影响定级）。
   - `TObjectIterator` 迭代期间触发 GC 的检查维持原结论：**未发现"在迭代体内创建/销毁 UObject 或调用 `CollectGarbage`"的模式**；唯一未排除的是 `FCSharpEnvironment::NotifyUObjectCreated`（`FCSharpEnvironment.cpp:276-300`，由任意 `NewObject` 触发，其中 `:291` `Bind<true>` 可能再创建 UObject）的**再入风险**，亦未排除，**置信度：中**。
5. **`Source/ThirdParty/` 按要求排除**（含 `Source/ThirdParty/LeanCLR/` —— 上面的 grep 在 `UnloadAssembly` 检索时命中过 `ThirdParty/LeanCLR/src/runtime/...`，这些命中**未纳入**任何结论）。
6. **`ScriptCodeGenerator` / `SourceCodeGenerator` 模块只在 grep 层面覆盖**（`FClassGenerator`/`FStructGenerator`/`FEnumGenerator`/`FDelegateGenerator`/`FGeneratorCore`/`FGameplayTagGenerator` 的 `TObjectIterator`/`TFieldIterator`/`TSharedPtr` 使用点已列出，但未逐函数阅读）。这些是编辑器离线生成器，对象生命周期风险远低于运行时与编辑器模块，**预计无 P0/P1**，置信度：中。
7. **`FReferenceRegistry` 的 `TManagedHandleMapping<TSet<FReference*>> ReferenceRelationship`**（`FReferenceRegistry.h:27`）语义未完全展开：`FReference` 派生类（`TDelegateReference` 等）在析构时会回调 `RemoveDelegateReference`（`TDelegateReference.h:13-16`），形成"引用对象析构 → 注册表删除 → 再 delete"的链式调用。我**未验证是否存在递归/重入删除同一容器的场景**（`~FReferenceRegistry` 在 `:9-15` 遍历 `ReferenceRelationship` 的同时 `delete Reference`，而 `delete Reference` 会调用 `FCSharpEnvironment::RemoveDelegateReference` → `FDelegateRegistry.inl:71` `delete *FoundValue` → `~FDelegateHelper` → `RemoveFromRoot`）。**这是本报告中最后一个未排除的重入风险点，置信度：低**。建议加 `check(!bIsDeinitializing)` 或改为先 `TArray<FReference*> Local; swap; then delete`。

### 发现数量汇总

| 严重度 | 条数 | 编号 |
|---|---|---|
| **P0** | 2 | F-LIFE-001、F-LIFE-003 |
| **P1** | 3 | F-LIFE-006、F-LIFE-007、F-LIFE-010 |
| **P2** | 8 | F-LIFE-004、F-LIFE-005、F-LIFE-008、F-LIFE-009、F-LIFE-012、F-LIFE-015、F-LIFE-016、F-LIFE-017 |
| **P3** | 5 | F-LIFE-011、F-LIFE-013、F-LIFE-014、F-LIFE-018、F-LIFE-019 |
| **撤销（非缺陷）** | 1 | F-LIFE-002 |
| **合计** | **19** | — |

> **口径说明**：本表是最终分布（2+3+8+5+1=19 ✅）。级别变动：F-LIFE-010 `P0→P1`、F-LIFE-008/015/016 `P1→P2`、F-LIFE-002 `P0→撤销（非缺陷）`，共 **5 条级别变动**；F-LIFE-009 维持 P2（不在变动之列）。可达性：活跃 **17**、潜伏 **1**（F-LIFE-015）、不可达 **1**（F-LIFE-002）。

