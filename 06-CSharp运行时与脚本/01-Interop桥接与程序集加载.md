# C# 侧 Interop 桥接与程序集加载分析

> 本报告发现的**严重度见各 Finding 的「复核结论」字段**（共 40 条：含 3 条判定为非缺陷、21 条级别调整）。





> 分析范围：`Plugins/UnrealCSharp/Script/Interop/**`（AssemblyLoader、Bridge 8 个、Handle）、`Script/Weavers/**`
> 覆盖文件：11 个交付文件（**全部读完**）+ 4 个辅助核对文件（见下）

> 发现 40 条；未覆盖项见第 7 节


> **处置更新（2026-09-16，G7 活跃路径子集；**已压缩提交为 `1551383b`**）**：本报告 **3 条已改** —— **`F-CS1-028`**（`GetTypeSize`/`BufferSize` 由 `sbyte` 改 `int`，新增 `GetLdcI4()` 按值选 `Ldc_I4_S`/`Ldc_I4`，6 处发射点全覆盖；**插件模板与工程部署副本 `Script/Weavers/UnrealTypeWeaver.cs` 两处同改** —— `Script/Weavers/Weavers.csproj` 只编后者）；**`F-CS1-011`**（槽位语义按 `IScriptTypes.h:79-81` 的 `IManagedHandle` 对齐：引用类型改走 `HandleData.GetObject`/`Alloc`，值类型补 `bool`/`nint`/`nuint` 分支；**保留原签名 `long` 与原命名 `LongToField`/`FieldToLong`**（该槽位即 `IManagedHandle` = int64，ABI 不变），格式回到修改前的 `switch` 表达式形态 ⇒ 与 HEAD 的差异仅 3 行；**未加 try/catch**；C++ 侧仍无调用点）；**`F-CS1-003`**（`MethodBridge.GetMethod` 增参 `paramCount`，未命中/个数不符时**静默返回 0**；⚠️ **`call 0` 未消除**，且生成侧 `#else` 分支**当前已参编** ⇒ 本条当前是**活跃**缺陷，原文"潜伏"是按 LeanCLR 前提写的）⇒ 三条中 `F-CS1-028`/`F-CS1-011` 为已修、`F-CS1-003` **仅部分处置**。本轮口径：**不新增注释 / `throw` / 日志 / try-catch**。逐条明细见 [`09-…/00`](../../09-问题清单/00-推荐优先修复清单.md) §2.1。
## 0. 覆盖范围与阅读清单

| 文件 | 行数 | 是否读完 | 备注 |
|---|---|---|---|
| `Script/Interop/AssemblyLoader/AssemblyLoader.cs` | 63 | ✅ 全文 1-63 | F-CS1-001 |
| `Script/Interop/AssemblyLoader/UnrealAssemblyLoadContext.cs` | 45 | ✅ 全文 1-45 | F-CS1-009 |
| `Script/Interop/Bridge/ArrayBridge.cs` | 44 | ✅ 全文 1-44 | F-CS1-010 / 012 |
| `Script/Interop/Bridge/FieldBridge.cs` | 65 | ✅ 全文 1-65 | F-CS1-011 / 013 / 017 |
| `Script/Interop/Bridge/LogBridge.cs` | 175 | ✅ 全文 1-175 | F-CS1-014 |
| `Script/Interop/Bridge/MethodBridge.cs` | 214 | ✅ 全文 1-214 | F-CS1-003 / 005 / 006 / 018 / 019 / 040 |
| `Script/Interop/Bridge/ObjectBridge.cs` | 21 | ✅ 全文 1-21 | F-CS1-008 |
| `Script/Interop/Bridge/StringBridge.cs` | 45 | ✅ 全文 1-45 | F-CS1-007 |
| `Script/Interop/Bridge/TypeBridge.cs` | 505 | ✅ 全文 1-505（分 3 段） | F-CS1-020~025 |
| `Script/Interop/Handle/HandleData.cs` | 147 | ✅ 全文 1-147 | F-CS1-002 / 004 |
| `Script/Weavers/UnrealTypeWeaver.cs` | 1185 | ✅ 全文 1-1185（分 5 段） | F-CS1-026~039 |
| `Script/Weavers/FodyWeavers.xml` | 3 | ✅ 全文 | F-CS1-038 |

排除：`Script/**/obj/`、`Script/**/bin/`。

**辅助核对文件**（不属于交付范围，仅为确定 C++ 契约/调用点而读；发现归入本报告但归因时已注明）：

| 文件 | 用途 | 对应发现 |
|---|---|---|
| `Source/UnrealCSharpCore/Public/Domain/Script/IScriptTypes.h`（182 行，全文） | 桥接 typedef 与 `Op(...)` 宏、逐参数类型对照 | F-CS1-011 / 013 / 020 / 021 / 025 / 040 |
| `Source/UnrealCSharpCore/Public/Domain/Script/IManagedHandle.h`（40 行，全文） | `IManagedHandle` 是 `struct{int64}`（**非** `typedef`），影响 ABI 判断 | F-CS1-011 / 016 |
| `Source/UnrealCSharpCore/Private/Domain/{CoreCLR,LeanCLR}/*.cpp`（节选） | `LoadAssembly`/`UnloadAssembly`/`RegisterBinding` 调用点，LeanCLR 旁路 | F-CS1-001 / 040 |
| `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:895-919`（节选） | **生成 `GetMethod` 调用模板**（F-CS1-003 的关键证据） | F-CS1-003 / 040 |

## 1. 模块职责与架构速览

### 1.1 总体机制（已核实的关键事实）

`Script/Interop/` 是**宿主程序集**（C# 侧 `Interop` 程序集，AssemblyName = `Interop`，见 `UnrealAssemblyLoadContext.cs:14`）。它**完全不使用 `DllImport`**，而是双向的 "字符串名 + 原生函数指针" 机制：

- **C++ → C#**：`Interop` 里所有对外方法都标了 `[UnmanagedCallersOnly]`，C++ 侧通过 `Class_Get_Method_From_Name(Module, "MethodBridge", "Invoke")` 之类的反射拿到函数指针（见 `FLeanCLRDomain.cpp:775-778` 的 `UTILS_BRIDGE_METHODS` 宏展开）。名称字符串集中在 `Source/UnrealCSharpCore/Public/CoreMacro/ClassMacro.h` / `FunctionMacro.h`。
- **C# → C++**：C++ 侧在初始化时把函数指针批量交给 C#，由 `MethodBridge.RegisterBinding(byte** InNames, nint* InMethods, int InLength)`（`MethodBridge.cs:13`）登记到 `Dictionary<string, nint> StringToMethod`；C# 侧业务代码用 `MethodBridge.GetMethod(ref InSlot, "Name")`（`MethodBridge.cs:140`）**惰性解析并缓存在自己的静态槽位里**。

### 1.2 句柄模型（HandleData）

`HandleData`（`HandleData.cs:7`）是 C# 对象的统一句柄表：

- `Handles : Dictionary<nint, GCHandle>`（`:11`）——句柄号 → `GCHandle`。
- `ObjectToHandleReference : ConditionalWeakTable<object, HandleReference>`（`:22`）——对象 → `{Value=句柄号, Count=引用计数}`；这是**引用计数**（同一对象多次 `Alloc` 只分配一个 `GCHandle`，`Count` 累加；`FreeImplementation` 递减到 0 才真正 `Free`）。
- 句柄号 `Handle`（`:13`）单调递增，`Clear()` 不重置。
- 全局 `System.Threading.Lock Lock`（`:9`）保护全部四个字典操作 → **HandleData 自身是线程安全的**（这一点与很多同类插件不同，值得肯定）。

### 1.3 程序集加载（AssemblyLoader / UnrealAssemblyLoadContext）

`UnrealAssemblyLoadContext` **确实是 `isCollectible: true`**（`UnrealAssemblyLoadContext.cs:8`），`AssemblyLoader.Unload()` 也**确实调用了 `Context.Unload()`**（`AssemblyLoader.cs:43`）。所以"没有 Unload = 每次热重载泄漏一整个程序集"这一条**不成立**——但存在下面 F-CS1-001 描述的等价问题（Unload 的守卫条件在 LeanCLR 下恒假，导致清理逻辑整段被跳过）。

## 2. 关键调用链

1. **CoreCLR 加载用户程序集**：`FCoreCLRDomain::LoadAssembly` (`Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRDomain.cpp:201`) → `FFileHelper::LoadFileToArray` (`:224`) → `assembly_loader_Load_from_stream_fn(Data.GetData(), Data.Num(), PublishDirectory)` (`:228`) → `AssemblyLoader.LoadFromStream` (`Script/Interop/AssemblyLoader/AssemblyLoader.cs:14`) → `new UnrealAssemblyLoadContext(...)` (`:18`) → `Context.LoadFromStream(Stream)` (`:24`) → `HandleData.Alloc(Assembly)` (`:24`)。
2. **CoreCLR 热重载卸载**：`FCoreCLRDomain::UnloadAssembly` (`FCoreCLRDomain.cpp:240`) → `AssemblyLoaderUnloadFn()` (`:246`) → `AssemblyLoader.Unload()` (`AssemblyLoader.cs:31`) → `HandleData.Clear()` (`:35`) + `TypeBridge.Clear()` (`:37`) → `Context.Unload()` (`:43`) → 弱引用 + `GC.Collect()` 轮询 2 秒 (`:50-59`)。
3. **LeanCLR 加载用户程序集（完全不同路径！）**：`FLeanCLRDomain::LoadAssembly` (`Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp:787`) → `leanclr::vm::Assembly::load_by_name(TCHAR_TO_UTF8(*AssemblyName))` (`:797`) → `IManagedHandleFromObject` (`:800`)。**此路径不经过 `AssemblyLoader.LoadFromStream`**。
4. **LeanCLR 热重载卸载**：`FLeanCLRDomain::UnloadAssembly` (`FLeanCLRDomain.cpp:806`) → `FReflectionRegistry::Deinitialize()` (`:808`) → `Bridge_Invoke(AssemblyLoaderUnloadFn)` (`:812`) → `AssemblyLoader.Unload()` → **`if (Context != null)` 为假，`:35`/`:37` 全部跳过**。
5. **C# 业务代码调用 C++ 函数**：`MethodBridge.GetMethod(ref Slot, "XXX")` (`MethodBridge.cs:140`) → `StringToMethod.TryGetValue` (`:144`) → 空槽位时返回 `nint.Zero` (`:144` 的三元 else 分支)。
6. **C++ 反射调用 C# 方法**：`MethodBridge.Invoke` (`MethodBridge.cs:33`) → `HandleData.GetObject(InMethod) as MethodBase` (`:37`) → `Method.Invoke(Object, Parameters)` (`:71`) → `HandleData.Alloc(Result)` (`:120`)。
7. **C++ 反射 new C# 对象**：`ObjectBridge.NewObject` (`ObjectBridge.cs:10`) → `RuntimeHelpers.GetUninitializedObject(Type)` (`:14`) → `HandleData.Alloc(Object)` (`:16`)。

## 3. 发现清单

> **阅读说明（重要）**：本报告的 40 条发现在**写入顺序**上按"逐文件分析批次"排列（因为报告是边分析边落盘的），并非严格按严重度排序。
> 下面的**严重度导航表**给出了 `_CONVENTIONS.md` §2 要求的"按 P0→P3 排序"视图 —— 请以本表为准定位发现。
> 每条发现的标题行都自带严重度标注（`- **严重度**: Px`）。

### 3.0 严重度导航表（P0 → P3）

> **⚠️ 本表的更正声明** —— 本表是导航视图，其"严重度"列与各 Finding 小节内的最终值**不一致**，请以各小节内的 `- **严重度**:` 为准：
> 1. **F-CS1-003 / F-CS1-010 两行标 `P0` 已过期**：两者在各自小节内均为 **P2**（维持 P2，理由见两条的 `- **复核结论**:` 字段；本模块**当前无 P0**）。
> 2. **F-CS1-004（表内 P1）、F-CS1-013（表内 P2）、F-CS1-026（表内 P1）三行已过期**：三者在小节内均为 **`撤销（非缺陷）`**；进一步把 004 / 026 的撤销标注为**"存疑、证据不足"**（见各条 `- **复核结论**:`）。
> 3. 本表**行数完整**（40 行 = 40 条发现，含 `F-CS1-040`），编号集合与小节一致。

| 严重度 | 编号 | 一句话 | 文件:行 |
|---|---|---|---|
| **P0** | F-CS1-003 | `MethodBridge.GetMethod` 查不到返回 0，**全部**调用点（源生成器模板）强转后立即调用 → 跳转地址 0 | `MethodBridge.cs:140-148` + `UnrealTypeSourceGenerator.cs:917` |
| **P0** | F-CS1-010 | `ArrayBridge.ArrayGet` 只判上界不判负数 → 异常穿越 `[UnmanagedCallersOnly]` → 进程终止 | `ArrayBridge.cs:31` |
| **P1** | F-CS1-001 | LeanCLR 下 `Context` 恒 null，`Unload()` 的清理整段被跳过 → 句柄表跨热重载只增不减 | `AssemblyLoader.cs:33` |
| **P1** | F-CS1-002 | `Alloc` 用 `GCHandleType.Normal`（强句柄）→ 终结器永不执行，`Free` 是唯一解绑路径 | `HandleData.cs:39` |
| **P1** | F-CS1-004 | `GetObjectPointer` 返回未 pinned 托管对象裸地址 → GC 压缩后失效 | `HandleData.cs:108-127` |
| **P1** | F-CS1-011 | `FieldBridge` 引用类型字段的取值/赋值语义完全错误（句柄 vs 值混淆） | `FieldBridge.cs:49-64` |
| **P1** | F-CS1-019 | `MethodBridge.StringToMethod` 并发读写且全程无锁 | `MethodBridge.cs:10,25,144` |
| **P1** | F-CS1-020 | `MakeGenericType2` 复制粘贴错误（`InKeyType` 用两次）→ `Dictionary<K,K>` | `TypeBridge.cs:194` |
| **P1** | F-CS1-021 | 类型名按**字符数**截断后 UTF-8 编码 → 非 ASCII 类型名抛异常 → 进程终止 | `TypeBridge.cs:112,135,162` |
| **P1** | F-CS1-026 | `WITH_LEANCLR` 从未在 `Weavers.csproj` 定义 → `__{Prop}_Attrs` 生产者恒缺失，消费者却开着 | `Weavers.csproj` |
| **P1** | F-CS1-027 | 删除属性的 backing field 却不清理引用它的 IL（属性初始化器）→ 非法程序集 | `UnrealTypeWeaver.cs:351,570` |
| **P1** | F-CS1-028 | RPC 缓冲 `sbyte BufferSize` 在 ≥128 字节时溢出 → `localloc` 尺寸错乱 = 栈损坏 | `UnrealTypeWeaver.cs:918-954` |
| **P1** | F-CS1-030 | `GetAllMeta` 13 处解析零 null 检查 → 裸 NRE / 报错点远离根因 | `UnrealTypeWeaver.cs:1110-1158` |
| **P2** | F-CS1-005 | `MethodBridge.Invoke` 反射慢路径（无委托缓存、每调用分配） | `MethodBridge.cs:33-138` |
| **P2** | F-CS1-006 | `Invoke` 的 `catch` 吞异常 + 返回 0 与"返回 null"不可区分 | `MethodBridge.cs:125-137` |
| **P2** | F-CS1-007 | `StringBridge` 编码不对称（UTF-8 入 / UTF-16 出）+ native 内存所有权不明 | `StringBridge.cs:9-43` |
| **P2** | F-CS1-008 | `ObjectBridge.NewObject` 绕过构造函数与字段初始化器（AOT 下不可用） | `ObjectBridge.cs:14` |
| **P2** | F-CS1-009 | ALC 依赖解析不校验版本；`Interop` 分支缺自递归守卫 | `UnrealAssemblyLoadContext.cs:14-29` |
| **P2** | F-CS1-012 | `ArrayBridge.NewArray` 永久 pin 数组 → 强根 + 堆碎片 + 阻碍压缩 | `ArrayBridge.cs:19` |
| **P2** | F-CS1-013 | `FieldBridge` 被注册但 C++ 零调用点（"上膛的枪"） | `FieldBridge.cs:11,22` |
| **P2** | F-CS1-014 | `LogBridge` 重入丢日志 + `LogFn` 静态字段无同步（`volatile`/`Interlocked`） | `LogBridge.cs:11,143-145` |
| **P2** | F-CS1-018 | `StringToMethod` 无 `Clear()` → 热重载后残留指向已销毁原生域的指针 | `MethodBridge.cs:10` |
| **P2** | F-CS1-022 | `TypeBridge.GetMethod` 只用名字+参数量匹配重载（静默选错）且无缓存 | `TypeBridge.cs:63-69` |
| **P2** | F-CS1-023 | `StringToType` 并发无锁，`Clear()` 也不加锁 | `TypeBridge.cs:12,448,471,501` |
| **P2** | F-CS1-024 | 无程序集限定名时全程序集线性扫描取首个同名类型（不确定绑定） | `TypeBridge.cs:453-496` |
| **P2** | F-CS1-029 | `IsCompound` 与尺寸/存取默认分支假设冲突（结构体） | `UnrealTypeWeaver.cs:1065,902,823,863` |
| **P2** | F-CS1-031 | `GetUEAssemblyPath` 无校验 → 静默拼出错误路径 | `UnrealTypeWeaver.cs:1167-1174` |
| **P2** | F-CS1-032 | 属性重写假设"新变量即 0 号槽"，不清空既有局部变量 → `stloc.0` 类型错位 | `UnrealTypeWeaver.cs:377-499` |
| **P2** | F-CS1-033 | weaver 幂等性：RPC 路径重复 `Add` 字段/方法，结构体按固定指令索引插桩 | `UnrealTypeWeaver.cs:780,910,111-116` |
| **P2** | F-CS1-034 | 只扫描顶层类型（漏 `NestedTypes`）+ 属性匹配两套策略 | `UnrealTypeWeaver.cs:1069` |
| **P2** | F-CS1-036 | `_Implementation` 丢失参数名/属性/泛型参数，属性位被改为 `Public` | `UnrealTypeWeaver.cs:1030-1038` |
| **P2** | F-CS1-037 | `_Attrs` 编码丢失命名参数；数组参数序列化成 `"System.Object[]"` | `UnrealTypeWeaver.cs:258-281` |
| **P2** | F-CS1-040 | 名字键 `"Ns.Class::Method"` 是无校验的双向字符串契约；注册失败无检测 | `MethodBridge.cs` ↔ `FScriptDomainImpl.inl:603` |
| **P3** | F-CS1-015 | 命名空间声明风格不统一（file-scoped vs block-scoped） | 4 个文件 |
| **P3** | F-CS1-016 | `[UnmanagedCallersOnly]` 是否显式 `CallConvs` 不一致（x64 下等价） | 全部 Bridge |
| **P3** | F-CS1-017 | `FieldBridge.GetField` 每次反射查找无缓存 | `FieldBridge.cs:42` |
| **P3** | F-CS1-025 | 22 个 Box/Unbox 样板可收敛；`Read/WritePrimitiveValue` 缺 `char` 分支 | `TypeBridge.cs:204-444` / `MethodBridge.cs:157-212` |
| **P3** | F-CS1-035 | `GetTypeLdind` 的 `Byte`/`SByte` 分支写反（靠后续截断救回） | `UnrealTypeWeaver.cs:831-841` |
| **P3** | F-CS1-038 | `FodyWeavers.xml` 无配置键、`xsd` 缺失、角色不清 | `FodyWeavers.xml` |
| **P3** | F-CS1-039 | 与 SourceGenerator 的属性匹配语义不一致；两者都不支持字段级 `[UProperty]` | 两文件 |
| **P3** | （F-CS1-002 的补充）| `AssemblyLoader.LoadFromStream` 的 `Context ??=` 静默忽略后续 `InPublishDirectory` | `AssemblyLoader.cs:18` |

---

### P0

> **本模块当前 P0 数 = 0。**
> 两条历史 P0 **F-CS1-003** 与 **F-CS1-010** 均已由降为 P2，**维持 P2**，并各自补上后端区分与真实调用链证据（详见两条的 `- **复核结论**:` / `- **复核证据**:` 字段）：
> - **F-CS1-003**（`MethodBridge.GetMethod` 空函数指针）：机制成立且插件内唯一"调用点"是源生成器模板（`UnrealTypeSourceGenerator.cs:917`），但该模板位于 **`#else`（非 LeanCLR）**分支；五平台全 LeanCLR → `#if WITH_LEANCLR` 分支生成 `[DllImport]`，**此路径根本不生成** → **潜伏**，不构成 P0。
> - **F-CS1-010**（`ArrayBridge.ArrayGet` 负数索引）：`:31` 确无下界检查、调用链活跃可达，但 LeanCLR 下托管异常**不终止进程**（走 `RtResult`，调用方拿零值），"进程终止"只在 CoreCLR/Mono 成立 → **P2 活跃**（后果降级），不构成 P0。
> 以下"### P0"标题仅作历史分区保留，不再含有 P0 级条目。
> F-CS1-003 的完整内容在下方"### P1 前半段"之后（因为报告按分析批次落盘，未做整体重排 —— 请以上面的 **3.0 严重度导航表**与各条 `严重度` 字段为准定位）。
> **F-CS1-003 小节位于本文件"F-CS1-002"小节之后**；**F-CS1-010 小节位于"### P2 补充"之后**。

---

### P1（第一批：AssemblyLoader / HandleData / 核心 Bridge）

### [F-CS1-001] LeanCLR 下 `AssemblyLoader.Context` 恒为 null，`Unload()` 的守卫使 `HandleData.Clear()` / `TypeBridge.Clear()` 整段被跳过 → 句柄表跨热重载只增不减

- **类别**: 内存/资源泄漏
- **严重度**: **P1**（原 P2 建立在"不可达"判定上，该判定已证伪）
- **复核结论**: 部分确认（机制成立，且**在 LeanCLR 下活跃** —— "可达性 不可达"不成立）
- **可达性**: 活跃
- **复核证据**: `Script/Interop/AssemblyLoader/AssemblyLoader.cs:33`（`if (Context != null)` 守卫）；grep 证据（模式 `assembly_loader_load_from_stream|AssemblyLoaderUnloadFn|LoadFromStream`，`Source/**`）：`LoadFromStream` 只被 **Mono**（`FMonoDomain.cpp:415`）与 **CoreCLR**（`FCoreCLRDomain.cpp:228`）调用，**LeanCLR 从不调用**（`FLeanCLRDomain.cpp` 侧只有 `:810`/`:812` 的 `Bridge_Invoke(AssemblyLoaderUnloadFn)`）→ LeanCLR 下 `Context` 恒 null、守卫恒假，而 `Unload()` 确实每次都被 LeanCLR 调用 → `HandleData.Clear()` / `TypeBridge.Clear()` **永不执行**
- **级别变动**: P2→**P1**（理由：降级依据"不可达"被 grep 证伪；LeanCLR 每次热重载都调用 `Unload` 却整段跳过清理，`GCHandle` 强句柄表单调增长，属真实可复现的资源泄漏）
- **文件**: `Script/Interop/AssemblyLoader/AssemblyLoader.cs:31-61`（守卫在 `:33`）；对照 `Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp:806-813`
- **函数**: `Interop.AssemblyLoader.Unload()`
- **置信度**: 高（两条路径都已读到源码）

**现状（代码事实）**

```csharp
// Script/Interop/AssemblyLoader/AssemblyLoader.cs
 9:         private static UnrealAssemblyLoadContext? Context;
13:         [UnmanagedCallersOnly]
14:         public static unsafe nint LoadFromStream(byte* InData, int InLength, char* InInPublishDirectory)
15:         {
16:             if (InData != null && InLength > 0)
17:             {
18:                 Context ??= new UnrealAssemblyLoadContext(InInPublishDirectory is not null
19:                     ? new string(InInPublishDirectory)
20:                     : string.Empty);          // ← Context 唯一的赋值点
31:         public static void Unload()
32:         {
33:             if (Context != null)              // ← 唯一的守卫；LeanCLR 下恒为 false
34:             {
35:                 HandleData.Clear();
36: 
37:                 TypeBridge.Clear();
...
43:                     Context.Unload();
```

LeanCLR 的加载路径完全绕开了 `LoadFromStream`，因此 `Context` 从未被赋值：

```cpp
// Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp
787: void FLeanCLRDomain::LoadAssembly(const TArray<FString>& InAssemblies)
788: {
789: 	Assemblies.Empty();
790: 
791: 	for (const auto& AssemblyPath : InAssemblies)
792: 	{
793: 		const auto AssemblyName = FPaths::GetBaseFilename(AssemblyPath);
794: 
795: 		if (AssemblyName != INTEROP_NAME)
796: 		{
797: 			if (auto AssemblyResult = leanclr::vm::Assembly::load_by_name(TCHAR_TO_UTF8(*AssemblyName));
798: 				AssemblyResult.is_ok())
799: 			{
800: 				Assemblies.Add(IManagedHandleFromObject(AssemblyResult.unwrap()));
801: 			}
802: 		}
803: 	}
804: }
806: void FLeanCLRDomain::UnloadAssembly()
807: {
808: 	FReflectionRegistry::Get().Deinitialize();
809: 
810: 	if (AssemblyLoaderUnloadFn != nullptr)
811: 	{
812: 		Bridge_Invoke(AssemblyLoaderUnloadFn);   // → AssemblyLoader.Unload()，但 Context == null
813: 	}
```

对照 CoreCLR 路径（唯一会调用 `LoadFromStream` 的地方）：

```cpp
// Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRDomain.cpp
224: 		if (TArray<uint8> Data; FFileHelper::LoadFileToArray(Data, *AssemblyPath))
226: 			if (AssemblyLoaderLoadFromStreamFn != nullptr)
228: 				if (const auto Handle = AssemblyLoaderLoadFromStreamFn(Data.GetData(), Data.Num(),
```

grep 证据（`LoadFromStream` 全插件命中，排除 obj/bin/Intermediate/Binaries）：

| 命中位置 | 说明 |
|---|---|
| `Script/Interop/AssemblyLoader/AssemblyLoader.cs:14,24` | 定义 + 内部转发 |
| `Script/Interop/AssemblyLoader/UnrealAssemblyLoadContext.cs:25,32,42` | ALC 内部私有重载 |
| `Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRDomain.cpp:226,228` | **CoreCLR 唯一外部调用方** |
| `Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp:395,415` | Mono 调用方 |
| `Source/UnrealCSharpCore/Public/CoreMacro/FunctionMacro.h:33`、`IScriptTypes.h:159` | 名称宏/反射签名 |

→ LeanCLR 侧**零命中**。

**调用上下文**
- `LoadFromStream` 调用方：`FCoreCLRDomain.cpp:228`（CoreCLR）、`FMonoDomain.cpp:415`（Mono）。
- `Unload()` 调用方：`FCoreCLRDomain.cpp:246`（CoreCLR）、`FMonoDomain.cpp:434`（Mono）、`FLeanCLRDomain.cpp:812`（LeanCLR）。
- 三处都发生在**热重载**（编辑器重新编译 C# 后重建脚本域）阶段，主线程。

**问题**
LeanCLR 后端下：
1. `HandleData.Handles` / `ObjectToHandleReference` 里的条目**永远不会被 `Clear()` 批量释放**（`:35` 被跳过）。唯一的释放途径是 C++ 主动调 `HandleData.Free(handle)`（`HandleData.cs:25`）。
2. 更严重的是 **`Handles` 里存的是 `GCHandleType.Normal`（强句柄）**（`HandleData.cs:39`），强句柄本身就是 GC root，所以只要 C++ 侧不显式 `Free`，对应 C# 对象**永久存活**——包括用户自己的 C# 业务对象、委托、数组。
3. 第二次热重载时，旧的 LeanCLR 程序集/类型已被卸载（`Assemblies.Empty()` `:815`、`UEModule = nullptr` `:817`），但句柄表里仍持有指向**已卸载程序集类型**的强引用。这属于"卸载后悬空句柄"，一旦 C++ 侧用旧句柄回调就会命中已死类型，行为未定义。
4. `TypeBridge.Clear()` 也被跳过；若 `TypeBridge` 缓存了 `Type` 对象（走查中，见 F-CS1-0xx），这些 `Type` 会把整个 ALC/程序集钉在内存里，使 LeanCLR 的 `UnloadAssembly` 事实上不释放任何托管内存。

量级估算：每次热重载泄漏 = Σ(所有被 `Alloc` 过且未被 `Free` 的句柄对应的对象图)。`Alloc` 的调用点包括 `AssemblyLoader.cs:24`（整个程序集）、`MethodBridge.cs:93/120`（**每次反射调用的 by-ref 出参与返回值**）、`ObjectBridge.cs:16`（**每次 C++ new C# 对象**）、`StringBridge.cs:13`（**每个跨边界字符串**）。也就是说 C++ 每调用一次返回 string 的 C# 方法就泄漏一个句柄条目 + 一个强根字符串，除非 C++ 显式释放。编辑器里一次热重载即可累积成千上万条。

**建议**
把"是否执行清理"与 `Context` 解耦：

```csharp
[UnmanagedCallersOnly]
public static void Unload()
{
    // 清理必须无条件执行：与 Context 是否为 null 无关
    HandleData.Clear();
    TypeBridge.Clear();
    MethodBridge.Clear();   // StringToMethod 也含 nint 函数指针，指向即将失效的 C++ 域

    if (Context is { } ContextToUnload)
    {
        Context = null;
        var Weak = new WeakReference(ContextToUnload);
        try { ContextToUnload.Unload(); }
        finally { /* no-op */ }

        const int TimeLimit = 2000;
        var Stopwatch = System.Diagnostics.Stopwatch.StartNew();
        while (Weak.IsAlive && Stopwatch.ElapsedMilliseconds < TimeLimit)
        {
            GC.Collect();
            GC.WaitForPendingFinalizers();
        }
    }
}
```

同时把 `HandleData.Handles` 从 `GCHandleType.Normal` 改为 `GCHandleType.Weak`（见 F-CS1-004），从根上消除"强句柄永久钉住"。

**验证方式**
- grep：`Select-String -Path Source\UnrealCSharpCore\Private\Domain\LeanCLR\*.cpp -Pattern 'LoadFromStream'` → 期望 0 命中（现状 0）。
- 运行时：LeanCLR 后端下连续热重载 3 次，在 `HandleData.Alloc` 后打印 `Handles.Count`；预期单调增长且 `Unload` 后不回落。
- 加断言：在 `Unload()` 开头 `check`/`Debug.Assert(Context != null || Handles.Count == 0)`，LeanCLR 下必然触发。

---

### [F-CS1-002] `HandleData.Alloc` 使用 `GCHandleType.Normal`（强句柄）：包装对象成为永久 GC root，终结器永不执行 → 依赖终结器的解绑/释放路径完全失效

- **类别**: 内存/资源泄漏
- **严重度**: **P2**
- **复核结论**: 确认（逐字成立）
- **可达性**: 活跃（每次 `HandleData.Alloc` 默认都走该分支）
- **复核证据**: `Script/Interop/Handle/HandleData.cs:27-44`，关键行 **`:39` `GCHandle.Alloc(InObject, bPinned ? GCHandleType.Pinned : GCHandleType.Normal)`** —— 默认 `bPinned = false`（`:27` 签名默认参数）→ 全部对象句柄都是 `GCHandleType.Normal`（**强句柄 = GC root**），对象因而永不被回收、其终结器永不执行；唯一解绑路径是 `FreeImplementation`（`:46-78`，`:74` `OutHandle.Free()`）与 `Clear()`（`:129-145`，`:137` `Handle.Value.Free()`）。引用计数语义经复核无误：`:31-32` 用 `ConditionalWeakTable` 分配 `Value = ++Handle`（`:13` 单调递增、`Clear()` 不重置 → 与 F-CS1-001 的"只增不减"相互印证）
- **级别变动**: 维持 P2
- **文件**: `Script/Interop/Handle/HandleData.cs:27-44`（关键行 `:39`）
- **函数**: `Interop.HandleData.Alloc(object, bool)`
- **置信度**: 高（句柄语义确定）；"终结器是 TArray/委托解绑唯一路径"这一点见下方说明，置信度中

**现状（代码事实）**

```csharp
// Script/Interop/Handle/HandleData.cs
27:         public static nint Alloc(object InObject, bool bPinned = false)
28:         {
29:             lock (Lock)
30:             {
31:                 var HandleReference = ObjectToHandleReference.GetValue(InObject, _ =>
32:                     new HandleReference { Value = ++Handle, Count = 0 });
33: 
34:                 HandleReference.Count++;
35: 
36:                 if (!Handles.TryGetValue(HandleReference.Value, out _))
37:                 {
38:                     Handles[HandleReference.Value] =
39:                         GCHandle.Alloc(InObject, bPinned ? GCHandleType.Pinned : GCHandleType.Normal);
40:                 }
41: 
42:                 return HandleReference.Value;
43:             }
44:         }
```

`bPinned` 默认 `false`。**更正**：全插件只有**一处**传 `true` —— `Script/Interop/Bridge/ArrayBridge.cs:19`：

```csharp
// Script/Interop/Bridge/ArrayBridge.cs
19:                 return HandleData.Alloc(Array.CreateInstance(Type, InLength), true);
```

即**所有由 C++ 创建的托管数组都是 `Pinned` 句柄**（强根 + 固定地址），其余句柄（对象、字符串、方法返回值、程序集）是 `Normal`。两者都是**强根**，因此 F-CS1-002 的结论（终结器永不执行、`Free` 是唯一解绑路径）对**全部**句柄成立；`ArrayBridge` 额外叠加了"地址被永久固定"的副作用（见 F-CS1-012）。

**调用上下文**
`Alloc` 的调用点（全插件 grep `HandleData.Alloc`，排除 obj/bin）：
- `AssemblyLoader.cs:24`（程序集）
- `ObjectBridge.cs:16`（**C++ `NewObject` 每次新建 C# 对象**）
- `StringBridge.cs:13`（**每个从 C++ 传入的字符串**）
- `MethodBridge.cs:93`（by-ref 引用类型出参）、`MethodBridge.cs:120`（**每次反射调用的返回值**）
- `TypeBridge.cs`（多处，走查中）

**问题**
`GCHandleType.Normal` 创建的句柄**是强根**：被 `Alloc` 过的对象在整个 AppDomain 生命周期内不可能被 GC 回收，**其 `finalizer` 也就永远不会被调用**（finalizer 只在对象不可达时入队）。

连锁后果：
1. 若脚本层的 `TArray<T>`、生成委托类、`UObject` 包装类把"向 C++ 解绑 / 释放 native 资源"写在终结器（`~T()`）里，那条路径永不执行 → **native 侧资源永久泄漏**，且 C++ 的委托列表里永远留着指向已失效托管上下文的条目。
2. `Free` 是**唯一**解绑路径，而 `Free` 由 C++ 显式调用（`HandleData.cs:25` 的 `[UnmanagedCallersOnly] Free`）。任何 C++ 侧遗漏一次 `Free`，就是永久泄漏 —— 没有任何兜底。
3. `ObjectToHandleReference` 是 `ConditionalWeakTable`，**本身不会**阻止对象回收；但因为 `Handles` 里的 `Normal` 句柄持强引用，`ConditionalWeakTable` 的"弱"语义失去意义。

**必读的配套说明**：要确认影响范围，需要核对脚本层哪些类型把关键释放写在终结器里。已知相关线索：`MethodBridge.cs:93` 对 by-ref 引用类型出参调用 `HandleData.Alloc(Parameter)` —— 这些出参句柄在 C++ 侧被使用完毕后由谁 `Free` 需在 C++ 侧核实（不在本报告交付范围，标为**未覆盖项**）。

**建议**
改为弱句柄 + 显式生命周期，或引入 `SafeHandle`：

```csharp
// 方案 A（最小改动）：弱句柄。GetObject 已经用 Handles.TryGetValue + Target，
// 目标被回收时 Target 返回 null，语义自然降级为"句柄失效"，无需改 C++。
Handles[HandleReference.Value] =
    GCHandle.Alloc(InObject, bPinned ? GCHandleType.Pinned : GCHandleType.Weak);
```

注意 `GCHandleType.Weak` 下 `GCHandle.Target` 在对象被回收后返回 `null`，而 `FreeImplementation`（`:54-56`）已经做了 `Target != null` 判空，兼容。
若需要 pinned 场景（`GetObjectPointer` 返回裸指针并被 C++ 长期持有），则必须 pinned；但那样应改用 `SafeHandle`/`GCHandle` 的显式 `IDisposable` 语义并在 `Free` 中保证执行，而不是依赖终结器。

**验证方式**
- 单元测试：`HandleData.Alloc(new object())` 后不 `Free`，循环 `GC.Collect(); GC.WaitForPendingFinalizers();`，断言 `WeakReference.IsAlive == false`。现状会**永远 alive**（强句柄）；改为 Weak 后应转为 false。
- grep：`Select-String -Path Script -Include *.cs -Pattern 'GCHandleType'` → 期望看到 `Weak`。
- 在 `Free` 里加计数器与 `Handles.Count` 日志，长时间跑编辑器观察是否单调增长。

---

### [F-CS1-003] `MethodBridge.GetMethod` 名字查不到返回 `nint.Zero`（空函数指针），且**没有任何调用点判空** → 调用即跳转到地址 0

- **类别**: Bug / 未定义行为
- **严重度**: **P2**（空指针调用，必然崩溃）
- **复核结论**: 部分确认（机制成立；偏差：**当前五平台全 LeanCLR 下这条路径根本不生成**，故"跳转地址 0"在本工程不可达）
- **可达性**: 潜伏 —— `UnrealTypeSourceGenerator.cs:910-918`：`MethodBridge.GetMethod` + `delegate*` 模板位于 `#else`（**非** LeanCLR）分支；`#if WITH_LEANCLR` 分支生成的是 `[DllImport(NativeModuleName, CallingConvention = CallingConvention.Cdecl)] ... static extern unsafe partial`
- **复核证据**: `Script/Interop/Bridge/MethodBridge.cs:140-148`（`:144` 查找失败写 `nint.Zero`，无日志/无异常）；`Script/SourceGenerator/UnrealTypeSourceGenerator.cs:917`（插件内唯一调用点，是"生成代码的模板"，强转后立即调用、无判空）；重跑报告 §3 的 grep 表（模式 `GetMethod`，`Script/**/*.cs`）= **19 命中**，与报告表格逐条一致（含 `TypeBridge.cs:49,63,87`、`Utils.cs:83,84,443`、`UnrealTypeWeaver.cs:456,474,479,483,485,675,693,698,702,704,769` 的同名无关项）
- **级别变动**: 维持 P2（原 P0 → P2；并补上"LeanCLR 分支不生成该代码"的证据 → 可达性由"未注明的活跃"明确为**潜伏**）
- **文件**: `Script/Interop/Bridge/MethodBridge.cs:140-148`
- **函数**: `Interop.MethodBridge.GetMethod(ref nint, string)`
- **置信度**: 高（返回 0 的行为与**全部**调用点不判空都已由源码确证）

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/MethodBridge.cs
140:         public static nint GetMethod(ref nint InSlot, string InName)
141:         {
142:             if (InSlot == nint.Zero)
143:             {
144:                 InSlot = StringToMethod.TryGetValue(InName, out var Method) ? Method : nint.Zero;
145:             }
146: 
147:             return InSlot;
148:         }
```

注意 `:144` 的失败分支：查找失败时把 `InSlot` 写成 `nint.Zero` 并返回 0。**它不会抛异常、不会报错、不会写日志**。

`StringToMethod` 只在 `RegisterBinding`（`:13-30`）中被填充，而 `RegisterBinding` 只接受 `InMethods[Index] != 0` 的条目（`:19`）—— 也就是说**C++ 侧有函数指针但名字拼错、或 C++ 侧该函数解析失败没注册**，都会让 `GetMethod` 静默返回 0。

**调用上下文 / 空函数指针调用点表（已完整核实）**

grep 证据（`Script/**` 全部 `*.cs`，排除 `obj/`、`bin/`，模式 `GetMethod`）：

| 命中位置 | 性质 |
|---|---|
| `Script/Interop/Bridge/MethodBridge.cs:140` | **定义** |
| `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:917` | **唯一的调用点 —— 但它本身是"生成代码的模板"** |
| `Script/Interop/Bridge/TypeBridge.cs:49,63,87` | 同名但无关（`TypeBridge.GetMethod` / `Type.GetMethods`） |
| `Script/UE/CoreUObject/Utils.cs:83,84,443` | 同名但无关（`StackFrame.GetMethod` / `Type.GetMethods`） |
| `Script/Weavers/UnrealTypeWeaver.cs:456,474,479,483,485,675,693,698,702,704,769` | 同名但无关（`Property.GetMethod` / `Type.GetMethods`） |

也就是说：**`MethodBridge.GetMethod` 在插件源码里没有任何手写调用点**；它的全部调用者都是**源生成器为每个 `partial` 库桥接方法生成的表达式体方法**。生成模板如下（`UnrealTypeSourceGenerator.cs:895-919`）：

```csharp
// Script/SourceGenerator/UnrealTypeSourceGenerator.cs
900:                 var pointerType = string.Join(", ", method.Parameters
901:                     .Select(Parameter => Qualify(Parameter.Type, qualifiedTypes)).Concat(new[] { returnType }));
902: 
903:                 var key = $"{containingNamespace}.{InOwner.Name}::{method.Name}";
904: 
905:                 var slot = $"{method.Name}_Slot";
...
909:                 source +=
910:                     "#if WITH_LEANCLR\n" +
911:                     $"\t\t[DllImport(\"{NativeModuleName}\", CallingConvention = CallingConvention.Cdecl)]\n" +
912:                     $"\t\t{accessibility} static extern unsafe partial {returnType} {method.Name}({parameters});\n" +
913:                     "#else\n" +
914:                     $"\t\tprivate static nint {slot};\n" +
915:                     "\n" +
916:                     $"\t\t{accessibility} static unsafe partial {returnType} {method.Name}({parameters}) =>\n" +
917:                     $"\t\t\t((delegate* unmanaged[Cdecl]<{pointerType}>)global::Interop.MethodBridge.GetMethod(\n" +
918:                     $"\t\t\t\tref {slot}, \"{key}\"))({arguments});\n" +
919:                     "#endif\n\n";
```

**生成的用户代码形态**（把上面的模板代入，例如 `Script/UE/CoreUObject/Utils.cs` 里某个库桥接方法）：

```csharp
private static nint Foo_Slot;

public static unsafe partial int Foo(int A) =>
    ((delegate* unmanaged[Cdecl]<int, int>)global::Interop.MethodBridge.GetMethod(
        ref Foo_Slot, "Script.CoreUObject.Utils::Foo"))(A);

// ↑ 若 GetMethod 返回 0，这里就是 ((delegate* ...)0)(A) —— 跳转到地址 0
```

**空函数指针调用点表（`MethodBridge.GetMethod` 的返回值去向）**

| # | 调用点 | 是否判空 | 后果 |
|---|---|---|---|
| 1 | `UnrealTypeSourceGenerator.cs:917-918` 生成的所有库桥接方法（`Script/UE/Library/*Implementation.cs`、`Utils.cs` 等**全部** `partial` 桥接） | **否** | **直接把 `nint` 强转成 `delegate* unmanaged[Cdecl]<...>` 并立即调用** → 名值查不到即跳转地址 0 |
| 2 | 上述生成代码所在的 `#else` 分支（即 **非** LeanCLR 后端） | — | LeanCLR 走 `:911-912` 的 `[DllImport]`，不经过 `GetMethod`，**无此风险** |

**结论：`GetMethod` 的全部调用点（100%）都不判空**（因为调用点是生成的，模板里就没有任何判断），这使 F-CS1-003 从"理论风险"升级为**结构性的必然风险**。

**名字键（key）契约核对（已核实一致）**
生成器用的 key 是 `"{namespace}.{OwnerName}::{methodName}"`（`:903`）。C++ 侧的注册键由以下宏拼出：

```cpp
// Source/UnrealCSharpCore/Public/CoreMacro/ClassMacro.h
5: #define COMBINE_FULL_NAME(A, B) FString::Printf(TEXT("%s.%s"), *A, *B)
// Source/UnrealCSharpCore/Public/CoreMacro/BindingMacro.h
7: #define BINDING_COMBINE_FUNCTION(Function) FString::Printf(TEXT("::%s"), *Function)
```

```cpp
// Source/UnrealCSharpCore/Public/Domain/Script/FScriptDomainImpl.inl
601: void SCRIPT_DOMAIN_TYPE::RegisterBinding() const
603: 	if (MethodBridgeRegisterBindingFn != nullptr)
...
611: 		for (const auto& Class : FBinding::Get().Register().GetClasses())
613: 			for (const auto& Method : Class->GetMethods())
615: 				const auto MethodName = StringCast<UTF8CHAR>(*Method.GetMethod());
...
625: 				MethodNames.Add(reinterpret_cast<const uint8*>(Name.GetData()));
627: 				Methods.Add(reinterpret_cast<PTRINT>(const_cast<void*>(Method.GetFunction())));
...
631: 		MethodBridgeRegisterBindingFn(MethodNames.GetData(), Methods.GetData(), MethodNames.Num());
```

→ C++ 注册的键 = `"Namespace.Class::Method"`，与生成器的 key **格式一致** ✓（这是一处正确且必须存在的隐式契约；由于**两侧都是字符串拼接、没有任何校验**，任何一侧的格式微调都会让**所有** C#→C++ 调用静默返回 0 → 全量崩溃。建议加一个启动自检：注册完成后断言 `StringToMethod.Count > 0`）。

**顺带核实（LeanCLR 的机制完全不同，避免了本条风险）**
`FLeanCLRDomain::RegisterBinding()` **不调用** `MethodBridge.RegisterBinding`，而是注册到 LeanCLR 自己的 P/Invoke 表：

```cpp
// Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp
854: void FLeanCLRDomain::RegisterBinding() const
855: {
856: 	static TArray<TArray<ANSICHAR>> PInvokeNames;
858: 	auto RegisterPInvoke = [](const FString& InName, const void* InFunc)
...
870: 		leanclr::vm::PInvokes::register_pinvoke(
871: 			reinterpret_cast<const char*>(Name.GetData()),
872: 			reinterpret_cast<leanclr::vm::PInvokeFunction>(reinterpret_cast<PTRINT>(const_cast<void*>(InFunc))),
873: 			&PInvoke_Dispatch);
875: 	};
876: 	for (const auto& Class : FBinding::Get().Register().GetClasses())
...
880: 			RegisterPInvoke(Method.GetMethod(), Method.GetFunction());
884: 	RegisterPInvoke(
885: 		COMBINE_FULL_NAME(NAMESPACE_INTEROP, CLASS_LOG_BRIDGE)
886: 		+ BINDING_COMBINE_FUNCTION(FUNCTION_LOG_BRIDGE_LOG_LEANCLR),
887: 		reinterpret_cast<const void*>(&FScriptLog::Log));
```

这与生成器 `#if WITH_LEANCLR` 分支（`:911-912` 的 `[DllImport]`）**互相配套** ✓。因此 `StringToMethod` 在 LeanCLR 下**始终为空字典**，而 `MethodBridge.Invoke` 在 LeanCLR 下是通过 `Class_Get_Method_From_Name(InteropModule, "MethodBridge", "Invoke")`（`FLeanCLRDomain.cpp:844` 的 `RESOLVE_LEANCLR_METHOD`）直接拿到函数指针的 —— 两条路径互不干扰 ✓。**这一段核对结论是"设计正确"**，记录以免重复审查。

**问题**
如前所述。补充两点量化：

1. **100% 的调用点不判空**（唯一调用点是生成模板，模板无判断）。
2. **零诊断**：查不到时既不抛异常也不写日志，`GetMethod` 只把槽位写成 0（`:144`）。触发原因可能是键格式变更、C++ 侧某个 `FBinding` 项注册失败（`FScriptDomainImpl.inl:631` 只注册 `FBinding::Get()` 里实际存在的项）、或 C++ 侧 `MethodBridgeRegisterBindingFn == nullptr`（`:603` 的守卫会使**整个注册被跳过**）——最后这种情形下 `StringToMethod` 全空，**所有** C#→C++ 调用都变成跳转 0，且没有任何提示。
3. **次要风险（置信度中）**：`Slot` 是生成代码里的 `private static nint`（`:914`），`GetMethod` 的 `if (InSlot == nint.Zero)`（`:142`）是**单向**的 —— 槽位一旦填上就永不失效。在正常热重载路径下每次 `Unload()` → 下一次 `LoadFromStream` 会新建 ALC → 新程序集里的静态字段重新为 0 → 会重新解析 ✓。但若 `Unload()` 未执行（`AssemblyLoaderUnloadFn == nullptr`）或 `Context` 被复用，则旧 C++ 函数指针会被**永久缓存**；配合 `StringToMethod` 无 `Clear()`（F-CS1-018），一次 C++ 侧 Live Coding 就可能让缓存指针指向已失效的代码。建议在 `Unload()` 里同时清空 `StringToMethod`（F-CS1-001 的修复片段已包含 `MethodBridge.Clear()`）。

**问题**
`delegate* unmanaged[Cdecl]<...>` 类型的函数指针为 0 时调用是**未定义行为**：在 Windows x64 上通常表现为 `EXCEPTION_ACCESS_VIOLATION`（读/执行地址 0），在 CoreCLR 下也可能被转为 `NullReferenceException` 或直接进程终止（取决于是否在 `try` 内）。而 `MethodBridge.Invoke`（`:33`）**外层确实有 `try/catch (Exception)`**（`:125`），但 `GetMethod` 返回的指针是在**用户脚本代码**里被调用的（不属于 `Invoke` 的 try 范围），所以 catch 救不了。

触发路径（具体、可构造）：
1. C++ 侧某脚本域方法名宏（`FunctionMacro.h`）与实际注册名不一致（大小写、拼写）。
2. LeanCLR 后端热重载后 `UnloadAssembly` 把 `Name##Fn = nullptr`（`FLeanCLRDomain.cpp:821-827`），旧脚本对象上的缓存槽位（`InSlot`）**非 0 时不会被刷新**（`:142` 的 `if (InSlot == nint.Zero)` 是单向的）→ 槽位里留的是**上一代已卸载域的函数指针**，调用即跳进已卸载/已释放的代码。

第 2 点尤其危险，且是 `GetMethod` 的缓存设计直接导致的：**槽位一旦被填就永不失效**。

**建议**

1. 让失败显式化（最小修复，立刻消除静默）：

```csharp
public static nint GetMethod(ref nint InSlot, string InName)
{
    if (InSlot == nint.Zero)
    {
        if (!StringToMethod.TryGetValue(InName, out var Method) || Method == nint.Zero)
        {
            Console.Error.WriteLine($"[Interop] Unresolved native method: {InName}");
            return nint.Zero;   // 调用方必须判空
        }
        InSlot = Method;
    }
    return InSlot;
}
```

2. 增加可失效的缓存代际：`AssemblyLoader.Unload()` 里清空 `StringToMethod`（`MethodBridge.Clear()`）**并且**让业务侧槽位也失效（例如把槽位改成 `nint` + `Generation` 组合，或在 `Unload` 时把所有脚本对象重置）。至少要加：

```csharp
public static void Clear() { lock (StringToMethod) { StringToMethod.Clear(); } }
```

并在 `AssemblyLoader.Unload()` 中调用（见 F-CS1-001 的修复片段）。

3. 提供**判空安全**的调用包装，并把"直接调用裸 `nint`"从公共 API 里去掉：例如生成 `delegate* unmanaged[Cdecl]<...>` 包装时统一走一个 `Invoke(...)` 辅助方法，内部判 0 则抛 `MissingMethodException` 而不是跳 0。

**验证方式**
- 构造用例：故意把 `FunctionMacro.h` 中某个名字改错一个字符，编译插件，运行 → 现状：崩溃且无任何日志；修复后：stderr 出现 `Unresolved native method: X`。
- grep：`Select-String -Path Script -Include *.cs -Pattern 'GetMethod\(' `，逐一确认调用点后是否有 `if (X != 0)` / `!= nint.Zero` 判断。
- 静态检查：为 `GetMethod` 加 `[Obsolete]` 或改用返回 `nint?`，强制调用方处理 null。

---

### [F-CS1-004] `HandleData.GetObjectPointer` 返回托管对象裸地址，对象未 pinned → GC 压缩移动后指针失效 —— **已证伪：非缺陷，请从排期移除**

- **类别**: 未定义行为 / 内存
- **严重度**: **撤销（非缺陷）** —— 但**撤销前提无法验证**，标"存疑"
- **复核结论**: 无法验证（卡点：撤销所依赖的"该裸指针安全"前提 = LeanCLR GC 是否为**非搬移式**、以及 C++ 侧拿到指针后的生存期，均未核实）→ 建议保持"存疑"而非当已结案
- **可达性**: 活跃（C++ 侧**确有调用点**，撤销理由不能是"无人使用"）
- **复核证据**: 代码事实 `Script/Interop/Handle/HandleData.cs:108-116`（`GetObjectPointer` = `Unsafe.As<object, nint>(ref Object)` 返回托管对象裸地址）；`:118-127`（`GetObjectPointers` 批量同构，**已于 `4f351f12` 删除**，见 `08-专项审计/01` 的 F-DEAD-002）；而句柄默认**未 pin**（`:39` `GCHandleType.Normal`）。实测 grep（模式 `GetObjectPointer|GetObjectPointers`，`Source/**`）= 8 命中，**C++ 侧活跃调用**：`Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp:125, 155, 167, 176, 196`（另有声明 `Public/Domain/LeanCLR/FLeanCLRDomain.h:196`、`Public/CoreMacro/FunctionMacro.h:39,41`）
- **级别变动**: 维持"撤销（非缺陷）"状态但**降置信**：撤销证据不足，按"存疑/待验证"处理（不进 P0/P1 排期，也不宣布结案）
- **函数**: `Interop.HandleData.GetObjectPointer(nint)` / `GetObjectPointers(nint*, nint*, int)`
- **置信度**: 中—高（`Unsafe.As<object, nint>` 的语义是确定的；C++ 侧如何使用该指针需另行核实 → 标中）

**现状（代码事实）**

```csharp
// Script/Interop/Handle/HandleData.cs（**删除前快照**：`:118-127` 的 `GetObjectPointers` 已于 `4f351f12` 删除；现 `Clear()` 位移至 `:118-134`，文件 147 → 136 行）
 27:         public static nint Alloc(object InObject, bool bPinned = false)
...
 39:                         GCHandle.Alloc(InObject, bPinned ? GCHandleType.Pinned : GCHandleType.Normal);
...
108:         public static nint GetObjectPointer(nint InHandle)
109:         {
110:             if (GetObject(InHandle) is { } Object)
111:             {
112:                 return Unsafe.As<object, nint>(ref Object);
113:             }
114: 
115:             return 0;
116:         }
118:         public static unsafe void GetObjectPointers(nint* InHandles, nint* OutObjectPointers, int InLength)
119:         {
120:             if (InHandles != null && OutObjectPointers != null)
121:             {
122:                 for (var Index = 0; Index < InLength; Index++)
123:                 {
124:                     OutObjectPointers[Index] = GetObjectPointer(InHandles[Index]);
125:                 }
126:             }
127:         }
```

**调用上下文**
待走查 C++ 侧（`UTILS_BRIDGE_METHODS` 里应有 `GetObjectPointer` / `GetObjectPointers` 对应项）；`HandleData.cs:118` 的批量版本命名与签名暗示 C++ 会一次性取一批对象地址，例如做批量比较或批量 pin。
> **已走查并定论**：`GetObjectPointer`（单数）的对应项在 **LeanCLR 专用表** `LEANCLR_INTEROP_BRIDGE_METHODS`（`FLeanCLRDomain.h:196`），**不在** `UTILS_BRIDGE_METHODS`；而复数版**没有任何 `Op(...)` 表项**、没有 `Name##Fn` 成员、没有 C++ 调用点 —— 那条"批量取地址"的路径**从未接线**（当初的批量解析机制 `ResolveObjectPointers`/`RefObjectPointers`/`RefHandleKeys` 在现有代码里 0 命中）。故复数版判为死桥并**已删除**（见 `08-专项审计/01` 的 F-DEAD-002）。

**问题**
`Unsafe.As<object, nint>(ref Object)` 取的是**托管对象的引用值（即对象头地址）**。该地址在 GC 压缩后**会改变**。虽然 `Alloc` 用了 `Normal` 强句柄（对象不会被回收，所以不会悬空到已释放内存），但**强句柄不阻止对象移动** —— `GCHandleType.Normal` 只保证存活，不保证地址稳定。因此 C++ 若把该地址存起来跨帧使用，GC 一旦压缩就是**读到垃圾/写坏别的对象**。

只有 `GCHandleType.Pinned` 才保证地址稳定，而 `Alloc` 的默认是 `bPinned: false`（`:27`）。grep 全插件 `HandleData.Alloc` 实参含 `true` 的只有 **1 处**（`ArrayBridge.cs:19`），因此经 `GetObjectPointer` 取地址的对象（对象、字符串、方法返回值…）**全部是未固定**的 —— 未固定的强句柄只保证"不回收"，**不保证"不移动"**。

另：`GetObjectPointers` 在 `InHandles != null && OutObjectPointers != null` 时**不校验 `InLength`**，为负值会直接不进循环（无害），但没有上界校验——由 C++ 保证，属隐性契约。（注：该批量方法已于 `4f351f12` 删除，此条隐性契约随之消失。）

**建议**
- 若 C++ 只是做"取一批地址去 native 侧比对/当 key"，改为在 C++ 侧直接用句柄号 `nint`（`HandleData` 的 `HandleReference.Value` 本来就是稳定且唯一的 key），**不需要**对象地址。
- 若确实需要地址，`GetObjectPointer` 必须配套 `GCHandleType.Pinned`，且应改名为 `GetPinnedObjectPointer` 并在文档/注释里写明"仅在句柄以 pinned 分配时有效"；同时把 `Alloc` 的 `bPinned` 参数暴露给 C++（当前由 C# 侧决定，C++ 无法控制）。
- 至少加注释与断言：

```csharp
// 要求 InHandle 是用 bPinned: true 分配的，否则返回的地址在 GC 压缩后失效
public static nint GetObjectPointer(nint InHandle) { ... }
```

**验证方式**
- grep C++ 侧 `GetObjectPointer` 的使用点，确认是否跨 GC 保存。
- 强制 GC 压测：`GetObjectPointer` 取值 → `GC.Collect(2, GCCollectionMode.Forced, true, compacting: true)` → 再取一次，比较是否变化；现状会变化（证明不可跨 GC 使用）。

---

### [F-CS1-005] `MethodBridge.Invoke` 用 `MethodBase.Invoke` 反射慢路径，每次调用都构建 `object[]` 并逐参数 `Type` 判定 → 热路径性能

- **类别**: 性能
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Script/Interop/Bridge/MethodBridge.cs:33-138`
- **函数**: `Interop.MethodBridge.Invoke(nint, nint, int, nint*)`
- **置信度**: 高

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/MethodBridge.cs
 33:         public static nint Invoke(nint InHandle, nint InMethod, int InParamCount, nint* InParams)
 37:                 if (HandleData.GetObject(InMethod) is MethodBase Method)
 41:                     var MethodParameters = Method.GetParameters();     // 每次调用都重新分配 ParameterInfo[]
 43:                     var MethodParameterLength = MethodParameters.Length;
 45:                     var Parameters = new object?[MethodParameterLength];  // 每次调用都装箱容器
 47:                     for (var Index = 0; Index < MethodParameterLength; Index++)
 49:                         var ParameterType = MethodParameters[Index].ParameterType;
 53:                             Parameters[Index] = ParameterType.IsValueType
 54:                                 ? Activator.CreateInstance(ParameterType)   // 每次调用反射构造默认值
...
 71:                     var Result = Method.Invoke(Object, Parameters);    // 反射调用，非缓存委托
...
120:                     return HandleData.Alloc(Result);                    // 返回值装箱 + 强句柄（见 F-CS1-002）
```

**调用上下文**
C++ 侧反射调用 C# 方法的统一入口（`FReflectionRegistry` 描述符里记录的 `MethodBase` 句柄）；由 `MethodBridge.Invoke` 的 `[UnmanagedCallersOnly]` 导出名暴露。**这是 C++ 调 C# 的唯一通道**，因此**所有**跨边界 C# 调用都走这条慢路径。

**问题**（量化的性能开销，每次跨边界调用）
1. `Method.GetParameters()`（`:41`）—— 每次分配并填充 `ParameterInfo[]`，内部还有缓存但数组本身是**新分配**。
2. `new object?[N]`（`:45`）—— 每次分配。
3. 值类型参数逐参数 `Activator.CreateInstance(ParameterType)`（`:54`）—— 反射构造，慢且装箱。
4. 值类型参数读取走 `ReadPrimitiveValue`（`:157-175`）的 **`switch` 表达式 + `_ when InType == typeof(...)` 模式匹配链** —— 最坏情况 13 次类型比较，且 `typeof(X)` 每次求值（JIT 通常会折叠为常量比较，但仍非直接跳转）。
5. `Method.Invoke`（`:71`）—— 不经任何委托缓存，每次做参数类型检查、装箱、`TargetInvocationException` 包装。
6. `HandleData.Alloc(Result)`（`:120`）—— 锁 + 字典查找 + `ConditionalWeakTable` 查找。

**建议**
按句柄缓存已编译的调用器：

```csharp
// 缓存键 = MethodBase 句柄（InMethod 已经是稳定句柄号，天然适合做键）
private static readonly Dictionary<nint, Func<object?, object?[], object?>> Invokers = new();
```

更彻底的做法：为每个描述符在注册阶段（`RegisterBinding` / `FReflectionRegistry::Initialize` 时）就用 `DynamicMethod` + `ILGenerator` 或 `Expression.Lambda().Compile()` 生成一个**类型化**的调用器（`delegate* unmanaged[Cdecl]<nint, nint, int, nint*, nint>` 风格的直通 stub），把全部装箱/反射开销消灭在注册期，热路径只剩边界转换。

⚠️ **AOT/IL2CPP 注意事项**：`DynamicMethod` / `Expression.Compile()` / `Reflection.Emit` 在 IL2CPP/AOT 与部分移动平台上**不可用**（会抛 `PlatformNotSupportedException`）。因此**不能无条件使用**；应做成"检测到 Emit 可用则用编译 stub，否则回退到 `Method.Invoke`"的双路径，并把回退路径保留为当前实现。当前代码**没有使用**任何 Emit，所以目前不存在 AOT 不兼容问题——但优化时必须小心不要引入。

**验证方式**
- 基准测试：用 BenchmarkDotNet 或 UE 侧 `TRACE_CPUPROFILER_EVENT_SCOPE` 包住一次 C++ → C# 空方法调用，对比 `Method.Invoke` 与缓存委托的耗时/分配（`GC.GetAllocatedBytesForCurrentThread()` 前后差）。
- 用 dotnet-counters 观察 `gen0` GC 次数与 `MethodBridge.Invoke` 调用次数的比例。

---

### [F-CS1-006] `MethodBridge.Invoke` 的 `catch` 只写 stderr 并返回 0 → C# 异常在 C++ 侧表现为"返回 null"，静默失败

- **类别**: 异常与错误处理
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Script/Interop/Bridge/MethodBridge.cs:125-137`
- **函数**: `Interop.MethodBridge.Invoke(nint, nint, int, nint*)`
- **置信度**: 高

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/MethodBridge.cs
125:             catch (Exception Exception)
126:             {
127:                 var InnerException = Exception is TargetInvocationException
128:                 {
129:                     InnerException: not null
130:                 } TargetInvocationException
131:                     ? TargetInvocationException.InnerException
132:                     : Exception;
133: 
134:                 Console.Error.WriteLine($"\nUnhandled Exception:\n{InnerException}");
135: 
136:                 return 0;
137:             }
```

**调用上下文**
同上，C++ 反射调用 C# 的唯一通道。返回值 0 在 C++ 侧被 `IManagedHandleIsValid(0)` 判为无效句柄（参见 `FCoreCLRDomain.cpp:231` 的同款判断）。

**问题**
1. **异常被完全吞掉**：只写 `Console.Error`。在 Shipping/打包构建里 `Console.Error` 可能没有接收者（重定向到 UE 的日志才可见），脚本异常因此**不可见**。
2. **返回 0 与"方法正常返回 null"不可区分**。C++ 侧看到"句柄无效"，既可能是脚本抛异常，也可能是方法真的返回了 `null`（`:103-106` 也返回 0）。这会把脚本 bug 伪装成"空返回"，极难定位。
3. 异常未被上报给 UE 的日志系统/崩溃上报，也不触发 `ensure`，导致**脚本异常不会中断游戏逻辑**，后续代码继续在错误状态下跑。
4. `TargetInvocationException` 的解包（`:127-132`）是对的，但只解包一层；若内层还是 `TargetInvocationException`（嵌套反射）则仍包裹。

**建议**
1. 区分"异常"与"null 返回"：给 `Invoke` 增加一个 out 参数或专用哨兵值（例如 `(nint)-1` 表示异常），并在 C++ 侧映射为 `ensure`/日志错误。
2. 通过已注册的原生日志函数把异常转给 UE 日志（插件里已有 `LogBridge`，见 F-CS1-0xx），而不是 `Console.Error`：

```csharp
catch (Exception Exception)
{
    var Inner = Exception is TargetInvocationException { InnerException: not null } Tie ? Tie.InnerException : Exception;
    LogBridge.Error($"[Interop] {Method?.DeclaringType?.FullName}.{Method?.Name} threw: {Inner}");
    return ErrorSentinel;   // 例如 unchecked((nint)(-1))
}
```
3. 循环解包 `TargetInvocationException`：

```csharp
while (Exception is TargetInvocationException { InnerException: { } Inner }) Exception = Inner;
```
4. 建议在编辑器构建下让异常**穿透**（不 catch），以便开发者立刻看到；Shipping 下才降级为日志 + 哨兵。

**验证方式**
- 写一个 C# 方法 `throw new Exception("x")` 并让 C++ 反射调用它：现状 → 只有 stderr，游戏继续；期望 → UE 日志出现错误且可定位到方法名。
- grep：`Select-String -Path Script -Include *.cs -Pattern 'Console.Error'` 统计还有多少处是这样上报的。

---

### [F-CS1-007] `StringBridge.NewString` 用 `Marshal.PtrToStringUTF8`，native 侧内存所有权不明；编码是 UTF-8 而非 UE 的 UTF-16 → 非 ASCII 边界隐患

- **类别**: 未定义行为 / 平台兼容 / 内存
- **严重度**: **P3**
- **复核结论**: ⚠️部分 —— 严重度由 P2 校正为 **P3**；可达性 活跃
- **文件**: `Script/Interop/Bridge/StringBridge.cs:9-17, 20-43`
- **函数**: `Interop.StringBridge.NewString(byte*)` / `GetString(nint, char*, int)`
- **置信度**: 中（`PtrToStringUTF8` 只读不释放是确定的；"谁分配谁释放"需 C++ 侧核实 → 中）

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/StringBridge.cs
  8:         [UnmanagedCallersOnly]
  9:         public static nint NewString(byte* InText)
 10:         {
 11:             if (InText != null)
 12:             {
 13:                 return HandleData.Alloc(Marshal.PtrToStringUTF8((nint)InText)!);
 14:             }
 15: 
 16:             return 0;
 17:         }
 18: 
 19:         [UnmanagedCallersOnly]
 20:         public static int GetString(nint InHandle, char* InBuffer, int InSize)
 21:         {
 22:             if (InBuffer != null && InSize > 0)
 23:             {
 24:                 if (HandleData.GetObject(InHandle) is string String)
 25:                 {
 26:                     var Length = String.Length;
 27: 
 28:                     if (Length < InSize)
 29:                     {
 30:                         fixed (char* Ptr = String)
 31:                         {
 32:                             Buffer.MemoryCopy(Ptr, InBuffer, (InSize - 1) * sizeof(char), Length * sizeof(char));
 33:                         }
 34: 
 35:                         InBuffer[Length] = '\0';
 36: 
 37:                         return Length;
 38:                     }
 39:                 }
 40:             }
 41: 
 42:             return -1;
 43:         }
```

**调用上下文**
- `NewString`：C++ → C# 传字符串（`byte*` = UTF-8），返回**句柄**（不是字符串本身）。
- `GetString`：C# → C++，写入调用方提供的 `char*` 缓冲，返回长度或 `-1`。

**问题**

1. **编码不对称**：入口是 UTF-8（`PtrToStringUTF8`），出口是 **UTF-16**（`char*` + `sizeof(char)`）。C++ 侧若用 `TCHAR_TO_UTF8` 传入、用 `StringCast<UTF16CHAR>` 取出，那是自洽的；但**任何一侧用错**（例如 C++ 传 `TCHAR_TO_ANSI`）都会得到静默的乱码。UE 的 `FString` 内部是宽字符，UTF-8 转换每次都要分配 —— 这是一个**明确的性能与语义约定**，但代码里**一行注释都没有**，属于隐性契约（可读性/可维护性问题）。
2. **`InText == null` 返回 0**（`:16`）：调用方拿到 0 句柄，后续 `GetObject(0)` 返回 null（`HandleData.cs:97`）→ 在 C# 侧表现为 `null` 字符串。语义上"空字符串"与"null"被混同（C++ 传 null 得到 0，C++ 传 `""` 得到有效句柄指向 `""`）。需确认 C++ 侧一致。
3. **`Marshal.PtrToStringUTF8` 不释放 native 内存**（`:13`）：它只做解码拷贝。native 侧那块 `byte*` 由**谁释放**取决于 C++ 分配策略（栈缓冲 / `TArray` / `FMemory::Malloc`）。若 C++ 是用 `FMemory::Malloc` 分配的并期望 C# 释放，则**每次传字符串都泄漏一块 native 内存**。本报告无法从 C# 侧确定，列为未覆盖项，但**这是必须核实的高风险点**。
4. **`GetString` 的长度契约偏保守**：`:28` 要求 `Length < InSize`（严格小于）。若 `InSize == Length + 1` 恰好够放 N 个字符 + `'\0'`，条件成立，正常。但若 C++ 传 `InSize == Length`（刚好等于字符数、不留 `'\0'`），则返回 `-1` 且**不写入任何内容**，C++ 侧可能把缓冲当成空串或未初始化数据使用。返回 `-1` 的语义（"缓冲太小"）应当被 C++ 侧显式处理。
5. **`InBuffer[Length] = '\0'` 越界分析**（`char` 是 2 字节，`InBuffer` 是 `char*`）：
   - `:32` 拷贝 `Length * sizeof(char)` 字节到目标起始，**但 `MemoryCopy` 的目标可用大小给的是 `(InSize - 1) * sizeof(char)`**。
   - `:35` 在 `InBuffer[Length]` 写入 `'\0'` = 偏移 `Length * 2` 字节，写入 2 字节 → 需要 `2*Length + 2 <= 2*InSize`，即 `Length + 1 <= InSize`，与 `:28` 的 `Length < InSize` **等价**。→ **本处无越界**，边界是自洽的。这一点检查后**未发现问题**，记录在此以免重复审查。
6. **`Length` 用 `string.Length`（UTF-16 code unit 数）**，不是 Unicode 码点数。含超出 BMP 的字符（emoji、生僻汉字）时，C++ 侧若按码点分配会算错所需缓冲。目前 `:28` 的 `Length < InSize` 用的是同一个 `Length`，所以自洽；但 C++ 侧若按 `FString::Len()`（也是 UTF-16 code unit）算就没问题 —— 需确认。

**建议**
1. 加注释明确编码契约：

```csharp
/// <param name="InText">以 NUL 结尾的 UTF-8 字节串；本函数只读取，不拥有也不释放该内存。</param>
public static nint NewString(byte* InText)
```
2. 为 native 内存所有权引入显式约定：要么 C++ 传"调用期间有效"的临时缓冲（当前实现假设），要么提供 `FreeString(nint)` 让 C# 释放；
3. 提供 `GetString` 的"查询所需长度"模式（`InBuffer == null` 时返回所需 `InSize`），避免调用方猜测容量：

```csharp
if (InBuffer == null) return Length + 1;   // 询问所需缓冲
```
4. 若 C++ 侧 `FString` 到 UTF-8 的转换在热路径（每帧多次），考虑增加一个 UTF-16 直通入口（`char*`）以避免 UTF-8 编解码开销。

**验证方式**
- 传含非 ASCII（如 `"中文"`、emoji）的字符串双向验证长度与内容。
- grep C++ 侧 `NewString` / `GetString` 的调用点，确认传入 `byte*` 的分配方式与生命周期。
- 在 `GetString` 的 `:42` 返回 -1 分支加 `ensure`/日志，观察运行期是否经常命中。

---

### [F-CS1-008] `ObjectBridge.NewObject` 用 `RuntimeHelpers.GetUninitializedObject` 绕过构造函数与字段初始化器 → C# 对象处于"字段未初始化"状态

- **类别**: 未定义行为
- **严重度**: **P3**
- **复核结论**: ⚠️部分 —— 严重度由 P2 校正为 **P3**；可达性 活跃
- **文件**: `Script/Interop/Bridge/ObjectBridge.cs:9-20`
- **函数**: `Interop.ObjectBridge.NewObject(nint)`
- **置信度**: 中—高

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/ObjectBridge.cs
  9:     [UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
 10:     public static nint NewObject(nint InHandle)
 11:     {
 12:         if (HandleData.GetObject(InHandle) is Type Type)
 13:         {
 14:             var Object = RuntimeHelpers.GetUninitializedObject(Type);
 15: 
 16:             return HandleData.Alloc(Object);
 17:         }
 18: 
 19:         return 0;
 20:     }
```

**调用上下文**
C++ 侧"创建一个 C# 对象"（例如为 UE 对象的 C# 包装类实例化）的入口。返回句柄，很可能是 C++ 把该句柄与本地的 `UObject` 关联起来。

**问题**
1. `RuntimeHelpers.GetUninitializedObject`（= `FormatterServices.GetUninitializedObject` 的新名）**不运行构造函数、不运行字段初始化器、不运行基类构造**。所有引用类型字段保持 `null`，值类型字段保持零。
   - 若目标类型的构造函数里有"注册到某个表 / 初始化内部容器 / 订阅事件"的逻辑，**这些全部被跳过** → 对象表面上存在，实际不可用。
   - `readonly` 字段、`init` 属性、NonNullable 引用类型字段（编译器的 nullable 检查认为非空）**全是 null** → 后续访问 NRE。
2. 这是**刻意的设计选择**（因为 UE 的 spawn 流程要先把 native 对象造出来、再注入），但代码里没有任何注释说明调用方必须自己补齐初始化。属于**隐性契约**。
3. `HandleData.GetObject(InHandle) is Type Type` —— `InHandle` 必须是**类型句柄**；若 C++ 侧误传了实例句柄，则返回 0（静默失败，见 F-CS1-003 的同类问题）。
4. **AOT/IL2CPP 兼容性**：`RuntimeHelpers.GetUninitializedObject` **在 NativeAOT / IL2CPP 下会抛 `PlatformNotSupportedException`**（因为它依赖运行时的未初始化对象分配原语）。Unity IL2CPP 明确不支持；.NET NativeAOT 也不支持（自 .NET 8 起 NativeAOT 上 `GetUninitializedObject` 会抛 `NotSupportedException`，除非类型满足特定条件）。**这是平台兼容 P1 风险**：若该项目未来要支持 IL2CPP/AOT 脚本后端，`ObjectBridge` 会直接失效。当前支持的 Mono/CoreCLR/LeanCLR 都是 JIT，所以现状可用 —— 但应在注释中标注。

**建议**
1. 优先改为走真实构造函数：

```csharp
var Object = Activator.CreateInstance(Type);   // 要求无参构造
```
在生成器（`SourceGenerator`）阶段保证所有 C# 包装类都有 public 无参构造，并让生成的构造函数只做"必要的默认初始化"，把依赖 native 的初始化移到显式的 `Initialize(nativeHandle)` 调用里。
2. 若必须保留 `GetUninitializedObject`（为了性能或因为构造有副作用），加上**明确的文档注释**与一个可选的"后置初始化"回调：

```csharp
/// 注意：不调用构造函数。调用方必须随后调用该类型的无参初始化逻辑。
public static nint NewObject(nint InHandle)
```
3. 加平台守卫，避免 AOT 下静默崩溃：

```csharp
if (!RuntimeFeature.IsDynamicCodeSupported)
    throw new PlatformNotSupportedException("ObjectBridge.NewObject requires JIT.");
```

**验证方式**
- 定义一个带字段初始化器 `private readonly List<int> _list = new();` 的类，经 `ObjectBridge.NewObject` 造出后访问 `_list` → 现状应 NRE（证明初始化器被跳过）。
- grep 生成器：`Select-String -Path Script\SourceGenerator -Include *.cs -Pattern 'Constructor|\.ctor'`，确认生成的类型是否有需要运行的构造逻辑。

---

### P3 附加（位于此处是因为按分析批次落盘）：`AssemblyLoader.LoadFromStream` 的入参与 `UnmanagedMemoryStream` 生命周期 —— **检查结论：实现正确**，仅 1 处 P3 隐患（`Context ??=` 静默忽略后续 `InPublishDirectory`）

- **类别**: 未定义行为
- **严重度**: P3
- **文件**: `Script/Interop/AssemblyLoader/AssemblyLoader.cs:14-28`
- **函数**: `Interop.AssemblyLoader.LoadFromStream(byte*, int, char*)`
- **置信度**: 高

**现状（代码事实）**

```csharp
// Script/Interop/AssemblyLoader/AssemblyLoader.cs
14:         public static unsafe nint LoadFromStream(byte* InData, int InLength, char* InInPublishDirectory)
15:         {
16:             if (InData != null && InLength > 0)
17:             {
18:                 Context ??= new UnrealAssemblyLoadContext(InInPublishDirectory is not null
19:                     ? new string(InInPublishDirectory)
20:                     : string.Empty);
21: 
22:                 using var Stream = new UnmanagedMemoryStream(InData, InLength);
23: 
24:                 return HandleData.Alloc(Context.LoadFromStream(Stream));
25:             }
26: 
27:             return 0;
28:         }
```

**检查结论（这一处我逐条验证后认为基本正确，记录以免重复审查）**
- `new string(InInPublishDirectory)`（`:19`）：从 `char*` 构造 `string` 会一直读到 **NUL 终止符**。C++ 侧（`FCoreCLRDomain.cpp:229`）传的是 `reinterpret_cast<const char16_t*>(PublishDirectory.Get())` —— `StringCast<UTF16CHAR>` 的结果，**是 NUL 结尾的**。✅ 正确。
- `UnmanagedMemoryStream` 的 `using`（`:22`）：`UnmanagedMemoryStream.Dispose()` 默认**不释放底层非托管内存**（它不拥有该内存），所以 `using` 不会导致 `InData` 被释放。C++ 侧 `Data` 是 `TArray<uint8>` 局部变量，在 `LoadAssembly` 作用域内有效（`FCoreCLRDomain.cpp:224`），`Context.LoadFromStream` 会**完全读完流**（AssemblyLoadContext 要求流可读且读取到末尾）—— ✅ 生命周期正确。
- 但 `char* InInPublishDirectory` 被**立即拷贝**成 `string`，因此之后不再依赖该指针，✅ 安全。

**唯一遗留问题（P3）**：`Context ??=`（`:18`）意味着**只有第一次 `LoadFromStream` 会带上 `InPublishDirectory`**；后续调用传入的目录被**静默忽略**。`FCoreCLRDomain.cpp:210` 每次都算同一个 `PublishDirectory`，因此当前无害；但如果将来支持多目录/插件目录，这里会成为一个静默的 bug。建议加断言或在目录变化时重建 ALC。

**建议**：加一行守卫，避免未来踩坑：

```csharp
if (Context is not null && Context.PublishDirectory != PublishDirectory)
{
    Console.Error.WriteLine($"[Interop] PublishDirectory changed ({...} -> {PublishDirectory}); rebuild required.");
}
```

（注：`PublishDirectory` 需要从主构造参数提升为可读取的属性。）

**验证方式**：grep `GetFullPublishDirectory` 确认调用期是否恒定。

---

### [F-CS1-009] `UnrealAssemblyLoadContext.Load` 返回 null 后由默认 ALC 兜底 → 依赖解析失败时可能加载到**错误版本**的依赖程序集

- **类别**: Bug / 未定义行为
- **严重度**: **P3**
- **复核结论**: ⚠️部分 —— 严重度由 P2 校正为 **P3**；可达性 活跃
- **文件**: `Script/Interop/AssemblyLoader/UnrealAssemblyLoadContext.cs:10-30`
- **函数**: `UnrealAssemblyLoadContext.Load(AssemblyName)`
- **置信度**: 中

**现状（代码事实）**

```csharp
// Script/Interop/AssemblyLoader/UnrealAssemblyLoadContext.cs
  7:     internal sealed class UnrealAssemblyLoadContext(string InPublishDirectory)
  8:         : AssemblyLoadContext(name: "UnrealAssemblyLoadContext", isCollectible: true)
  9:     {
 10:         protected override Assembly? Load(AssemblyName InAssemblyName)
 11:         {
 12:             if (InAssemblyName.Name != null)
 13:             {
 14:                 if (InAssemblyName.Name == "Interop")
 15:                 {
 16:                     return GetLoadContext(typeof(AssemblyLoader).Assembly)
 17:                                ?.LoadFromAssemblyName(InAssemblyName)
 18:                            ?? Default.LoadFromAssemblyName(InAssemblyName);
 19:                 }
 20: 
 21:                 var AssemblyPath = Path.Combine(InPublishDirectory, InAssemblyName.Name + ".dll");
 22: 
 23:                 if (File.Exists(AssemblyPath))
 24:                 {
 25:                     return LoadFromStream(AssemblyPath);
 26:                 }
 27:             }
 28: 
 29:             return null;
 30:         }
```

**检查结论**
- ✅ **没有递归加载风险**：`Load` 只做**单层**解析 —— 要么从 `InPublishDirectory` 直接读文件（`:25`），要么返回 `null` 交给默认 ALC。`LoadFromStream(AssemblyPath)`（`:32`）是**私有重载**，读文件后调 `AssemblyLoadContext.LoadFromStream`（`:42`），**不会再回调 `Load(AssemblyName)`**。所以"A↔B 互相依赖导致 `Load` 递归爆栈"这一假设在本实现下**不成立**（记录以免重复审查）。
- ✅ **文件流正确释放**：`:38` `using var AssemblyStream = new MemoryStream(File.ReadAllBytes(...))`、`:40` `using var PdbStream = ...`，`File.ReadAllBytes` 内部自己 `using` 了 `FileStream`。**DLL 不会被锁定**，"下次编译无法覆盖 DLL"这一假设在本文件**不成立**（记录以免重复审查）。
  - ⚠️ 但注意：`MemoryStream` 被 `using` 之后，`AssemblyLoadContext.LoadFromStream` **会在返回前读完整个流**（这是 ALC 的契约），所以 `:42` 返回后释放流是安全的。✅
- ⚠️ **PDB 缺失是静默的**（`:36`）：`PdbBytes = File.Exists(PdbPath) ? File.ReadAllBytes(PdbPath) : null` —— 没有 PDB 就不加载符号。这本身合理，但没有日志；调试时"为什么断点不生效"会很难查。建议加一行 `LogBridge` 提示。

**问题（真正的问题）**：`:29 return null;` —— 当依赖 DLL 在 `InPublishDirectory` 里**不存在**（例如依赖在 `.NET shared framework` 里、或在 UE 的 Mono/CoreCLR 目录里），解析权交给默认 ALC。这在大多数情况下是对的。但有两个风险：

1. **`Interop` 特判分支（`:14-18`）**：`GetLoadContext(typeof(AssemblyLoader).Assembly)?.LoadFromAssemblyName(...)` —— `GetLoadContext` 会返回"加载 `AssemblyLoader` 所在程序集的 ALC"。如果它恰好返回**当前这个 ALC 自己**（在 `Interop` 由自身加载的边界情况下理论上可能），`LoadFromAssemblyName` 会**重新进入 `Load(AssemblyName)`** → **无限递归 → StackOverflow**（StackOverflowException 在 .NET 中不可捕获，直接杀进程）。当前 `Interop` 是由宿主进程加载的（不在 `InPublishDirectory` 里），所以实践中不触发；但 `GetLoadContext` 的返回值没有做"是否等于 this"的校验，是一个**潜在的自递归陷阱**。
2. **版本不匹配静默通过**：`Path.Combine(InPublishDirectory, Name + ".dll")`（`:21`）**只按简单名匹配，完全不校验 `InAssemblyName.Version` / `PublicKeyToken` / `Culture`**。若用户 publish 目录里有一份旧版本的 `Newtonsoft.Json.dll`，而某个依赖要求 13.0.0，本 ALC 会返回旧版本 —— ALC 绑定过程中若版本差异较大，会在**调用时**抛 `MissingMethodException`/`TypeLoadException`（而不是加载时），错误信息指向的地方与真实原因相去甚远。

**建议**
1. 加自递归守卫：

```csharp
if (InAssemblyName.Name == "Interop")
{
    var InteropContext = GetLoadContext(typeof(AssemblyLoader).Assembly);
    if (InteropContext is null || ReferenceEquals(InteropContext, this))
    {
        return null;   // 交给默认 ALC，不要自递归
    }
    return InteropContext.LoadFromAssemblyName(InAssemblyName);
}
```
2. 记录解析失败的日志（当前完全静默）：

```csharp
// 在 :21 之后
if (!File.Exists(AssemblyPath))
{
    Console.Error.WriteLine($"[Interop] Dependency not found in publish dir, falling back to default ALC: {InAssemblyName}");
    return null;
}
```
3. 若要严格版本匹配，读取 `AssemblyName.GetAssemblyName(AssemblyPath)` 比对 `Version`，不匹配则记警告（**不要**直接拒绝，否则可能破坏现有部署）。

**验证方式**
- 在 `Load` 里打印 `InAssemblyName.FullName` 与是否命中 `File.Exists`，跑一次完整的热重载流程，观察是否有依赖落到 fallback。
- 构造用例：向 publish 目录塞一个同名旧版本 DLL，观察是否出现 `MissingMethodException`。

### [F-CS1-010] `ArrayBridge.ArrayGet` 只检查 `InIndex < Array.Length`，**不检查 `InIndex < 0`** → 负数索引抛 `IndexOutOfRangeException` 穿越 `[UnmanagedCallersOnly]` 边界 = 进程直接终止

- **类别**: Bug / 崩溃
- **严重度**: **P2**
- **复核结论**: 部分确认（机制成立：`:31` 确无 `InIndex >= 0` 检查；偏差：**" = 进程直接终止"必须按后端区分** —— LeanCLR 下托管异常**不终止进程**，走 `RtResult` 返回值协议、调用方拿到零值；只有 CoreCLR/Mono 下才是 fail-fast 杀进程）
- **可达性**: 活跃（五平台 LeanCLR 下同样可达，但后果降级为"静默返回 0/零值"）
- **复核证据**: `Script/Interop/Bridge/ArrayBridge.cs:31`（`if (InIndex < Array.Length)` —— 无下界检查）、`:33`（`Array.GetValue(InIndex)`）；调用链逐跳已核：`Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp:51-53`（`ArrayGet` → `ScriptDomain->ArrayGet`）→ `Source/UnrealCSharpCore/Public/Domain/Script/FScriptDomainImpl.inl:362-365`（`ArrayBridgeArrayGetFn` + `SCRIPT_DOMAIN_INVOKE`）→ `Source/UnrealCSharpCore/Public/Domain/Script/IScriptTypes.h:156`（`Op(ArrayBridgeArrayGet, ...)`）；LeanCLR 侧另有 `Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp:227` 直接调用 `ArrayGet`
- **级别变动**: 维持 P2（原 P0 → P2：本工程全 LeanCLR 下不杀进程，故不构成 P0，但负数索引仍静默产生错误结果 → 保留 P2 而非撤销）
- **文件**: `Script/Interop/Bridge/ArrayBridge.cs:26-43`（判断在 `:31`）
- **函数**: `Interop.ArrayBridge.ArrayGet(nint, int)`
- **置信度**: 高

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/ArrayBridge.cs
26:     [UnmanagedCallersOnly]
27:     public static nint ArrayGet(nint InHandle, int InIndex)
28:     {
29:         if (HandleData.GetObject(InHandle) is Array Array)
30:         {
31:             if (InIndex < Array.Length)          // ← 只有上界检查，没有 InIndex >= 0
32:             {
33:                 var Value = Array.GetValue(InIndex);   // ← 负数 → IndexOutOfRangeException
34: 
35:                 if (Value != null)
36:                 {
37:                     return HandleData.Alloc(Value);
38:                 }
39:             }
40:         }
41: 
42:         return 0;
43:     }
```

**调用上下文**
C++ 侧唯一入口是 `FClassReflection` 的数组读取族：

```cpp
// Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp
51: IManagedHandle ArrayGet(const IManagedHandle InManagedHandle, const int32 InIndex) const
53: 	return ScriptDomain->ArrayGet(InManagedHandle, InIndex);
56: bool ArrayGetBool(...)  58: if (const auto ManagedHandle = ArrayGet(...); IManagedHandleIsValid(ManagedHandle))
70: int32 ArrayGetInt32(...)  84: FString ArrayGetString(...)  98: FClassReflection* ArrayGetClass(...)
```

以及 C++ 反射元数据读取的 ~20 处 `InManagedReader.ArrayGet*(InParams[N], Index)`（`FClassReflection.cpp:164,172,182,186,191,218,220,228,233,240,249,251,280,282,303,305,307,315,320,327,345,347,353,359,363,365,372`）。这些循环都由 `InManagedReader.ArrayGetInt32(InParams[N], Index)` 之类返回的计数驱动，理论上非负。

**问题**
`Array.GetValue(-1)` 抛 `IndexOutOfRangeException`。关键点是**这个异常无法被任何 `try/catch` 捕获**：

- `ArrayGet` 标了 `[UnmanagedCallersOnly]`。.NET 运行时**禁止异常从 `UnmanagedCallersOnly` 方法逃逸**：一旦逃逸，**在 CoreCLR / Mono 下**运行时会调用 `Environment.FailFast` 等价路径终止进程（.NET 5+ 的行为是打印 "Unhandled exception. System.IndexOutOfRangeException" 后**立即杀进程**）。**⚠️ 修正（按后端区分，勿再当无条件事实）**：本工程五平台**全部使用 LeanCLR**（`Definitions.UnrealCSharpCore.h`：`WITH_CORECLR 0 / WITH_MONO 0 / WITH_LEANCLR 1`），LeanCLR 下托管异常**不终止进程**，而是走 **`RtResult` 返回值协议**、调用方拿到**零值**（已实机裁决的地面事实）；只有 CoreCLR 下才 `TerminateProcess`（`call 0` 已实机复现 fail-fast，`try/catch` 无效）。因此本条的"进程终止"是 **CoreCLR/Mono 专属后果（潜伏）**，在 LeanCLR 下的实际后果是**静默返回 0 / 零值**；该异常**不会**传播到 C++，也**不会**被 UE 的 crash handler 正常处理（只会看到托管栈）。
- 与 `MethodBridge.Invoke`（`:125` 有 `try/catch`）形成鲜明对比：**同一批 Bridge 里，只有 `MethodBridge.Invoke` 做了异常兜底**。`ArrayBridge` / `FieldBridge` / `ObjectBridge` / `StringBridge` / `HandleData` 的 `[UnmanagedCallersOnly]` 导出**全部没有 try/catch**。

触发路径（现实可构造）：C++ 侧任何把"索引"当哨兵用的代码，例如以 `-1` 表示"未找到/无效"后误传给 `ArrayGet`，或某个循环的 `Index` 因计数错误变成负数，就会**整个编辑器进程消失**，而不是抛一个可诊断的错误。

**建议**
最小修复（一行）：

```csharp
if ((uint)InIndex < (uint)Array.Length)   // 一次比较同时覆盖上界与负数
```
或显式：

```csharp
if (InIndex >= 0 && InIndex < Array.Length)
```

并建议给**所有** `[UnmanagedCallersOnly]` 导出加统一兜底，把异常转成日志 + 哨兵返回值而不是进程终止：

```csharp
[UnmanagedCallersOnly]
public static nint ArrayGet(nint InHandle, int InIndex)
{
    try { /* 原实现 */ }
    catch (Exception E)
    {
        Interop.LogBridge.WriteError($"[Interop] ArrayGet({InHandle}, {InIndex}) failed: {E}");
        return 0;
    }
}
```

**验证方式**
- 从 C++ 侧（或写一个 C# 测试）调用 `ArrayGet(handleOfArray, -1)`：现状 → 进程终止（`[exit code]`）；修复后 → 返回 0 或不进入分支。
- grep 全插件 `\[UnmanagedCallersOnly\]` 与其后 3 行内是否有 `try`：`Select-String -Path Script -Include *.cs -Pattern 'UnmanagedCallersOnly' -Context 0,3`。现状只有 `MethodBridge.Invoke` 有 try。

---

### [F-CS1-011] `FieldBridge` 引用类型字段的取值/赋值路径完全错误：把**句柄号/装箱 long** 当作字段值 → `SetValue`/`Convert.ToInt64` 抛异常且无兜底（同样是进程终止）

- **类别**: Bug / 未定义行为
- **严重度**: **P1**（维持；但"同样是进程终止"须按后端区分 —— LeanCLR 下不杀进程）
- **复核结论**: 确认（代码事实逐行成立；**标题断言"同样是进程终止"须按后端区分并加限定**）
- **可达性**: 活跃（LeanCLR 下同样可达，后果为静默零值/异常转 `RtResult`）
- **复核证据**: `Script/Interop/Bridge/FieldBridge.cs:49-64`（`LongToField`：`{ IsValueType: false } => InValue != 0 ? InValue : null`，把**装箱 long** 当成引用类型字段值；`FieldToLong` 把引用类型字段值直接 `Convert.ToInt64`）；`:10-19` / `:21-32`（`SetStaticValue`/`GetStaticValue` 签名是 `long` 而非句柄）；`:13`/`:24` → `GetField` → `:36-42`
- **修正（后端区分）**: 本工程五平台**全部 LeanCLR**（`Definitions.UnrealCSharpCore.h`：`WITH_CORECLR 0 / WITH_MONO 0 / WITH_LEANCLR 1`）。LeanCLR 下托管异常**不终止进程**，走 `RtResult` 返回值协议、调用方拿到零值；"异常穿出 `[UnmanagedCallersOnly]` 即杀进程"只在 **CoreCLR/Mono** 成立 → 该后果面标 **潜伏**
- **级别变动**: 维持 P1（语义错误确定，与后端无关；但"崩溃"表述降为后端专属）
- **文件**: `Script/Interop/Bridge/FieldBridge.cs:10-19, 21-32, 49-64`
- **函数**: `Interop.FieldBridge.SetStaticValue(nint, byte*, long)` / `GetStaticValue(nint, byte*)` / `LongToField(long, Type)` / `FieldToLong(object?)`
- **置信度**: 高（代码逻辑确定）；**注**：C++ 侧目前无调用点（见 F-CS1-013），故实际影响为"潜在 + 死代码"，综合定级 P1

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/FieldBridge.cs
10:     [UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
11:     public static unsafe void SetStaticValue(nint InHandle, byte* InName, long InValue)
12:     {
13:         var Field = GetField(InHandle, InName);
14: 
15:         if (Field != null)
16:         {
17:             Field.SetValue(null, LongToField(InValue, Field.FieldType));   // ← 无 try/catch
18:         }
19:     }
...
49:     private static object? LongToField(long InValue, Type InField) => InField switch
50:     {
51:         { IsValueType: false } => InValue != 0 ? InValue : null,   // ← 返回装箱的 long 本身！
52:         { IsEnum: true } => Enum.ToObject(InField, InValue),
53:         _ when InField == typeof(nint) => (nint)InValue,
54:         _ when InField == typeof(nuint) => (nuint)InValue,
55:         _ => Convert.ChangeType(InValue, InField)
56:     };
57: 
58:     private static long FieldToLong(object? InValue) => InValue switch
59:     {
60:         null => 0,
61:         nint v => v,
62:         nuint v => (long)v,
63:         _ => Convert.ToInt64(InValue)     // ← 对任意引用类型对象调用 Convert.ToInt64
64:     };
```

**C++ 侧的契约（关键证据）** —— `Source/UnrealCSharpCore/Public/Domain/Script/IScriptTypes.h:79-81`：

```cpp
79: typedef void (*field_bridge_set_static_value_fn)(IManagedHandle, const uint8*, IManagedHandle);
81: typedef IManagedHandle (*field_bridge_get_static_value_fn)(IManagedHandle, const uint8*);
```

C++ 的第三个参数与返回值类型都是 **`IManagedHandle`（即托管对象句柄）**，而 C# 侧把它们当作**裸标量值** `long` 处理。两边对同一个参数的解释**语义相反**。

**问题**

1. **`SetValue` 到引用类型字段**（`:51` + `:17`）：C++ 传进来一个句柄（按 C++ 类型定义就是句柄），`LongToField` 因为 `IsValueType == false` 直接返回 `InValue`（装箱的 `long`）。`FieldInfo.SetValue` 会做类型校验 → 抛 `ArgumentException: Object of type 'System.Int64' cannot be converted to type 'System.String'`（以 `string` 字段为例）。异常从 `[UnmanagedCallersOnly]` 逃逸 → **进程终止**（同 F-CS1-010）。
   正确实现应当是：
   ```csharp
   { IsValueType: false } => InValue != 0 ? HandleData.GetObject(InValue) : null,
   ```
   —— 这也解释了 `FieldBridge.cs:1-6` 的 `using` 里**没有** `Interop.HandleData` 之外的依赖，作者很可能把"值"和"句柄"两个概念混淆了。

2. **`GetValue` 从引用类型字段读**（`:58-64`）：`Convert.ToInt64(someObject)` 只对实现 `IConvertible` 的类型有效。对 `string` 字段会抛 `FormatException`/`InvalidCastException`；对任意 POCO 会抛 `InvalidCastException`。同样无兜底 → 进程终止。
   正确实现应当是返回 `HandleData.Alloc(值)` 的句柄。

3. **`Convert.ChangeType` 对 `char` / `decimal`** （`:55`）：`long → char` 会成功但**截断**（丢高位），`long → decimal` 正常。对 `bool` 字段，`Convert.ChangeType(1L, typeof(bool))` **抛 `InvalidCastException`**（`long` 不实现到 `bool` 的转换）。→ 用 `FieldBridge` 给 `static bool` 字段赋值也会崩溃。

4. **`sbyte`/`byte` 等**同理可能出现 `OverflowException`（`Convert.ChangeType(300L, typeof(byte))`）。

5. **`GetField` 每次调用都做一次反射查找**（`:42`），无缓存：

```csharp
42:                 return Type.GetField(Name, BindingFlags.Static | BindingFlags.Public | BindingFlags.NonPublic);
```

`Type.GetField` 内部有 `MemberInfo` 缓存但每次仍要构造 `RuntimeType.GetField` 调用 + 字符串比较。若该路径在热循环里（C++ 侧读写静态字段），是 P2 级性能问题。建议加 `Dictionary<(nint, string), FieldInfo?>` 缓存（注意 `nint` 是类型句柄，随热重载失效，需配合 `Clear`）。

6. **字段名不存在时静默返回 null / 返回 0**（`:15-18`、`:26-31`）：与 F-CS1-003 同类问题，没有任何日志。

**建议**

```csharp
[UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
public static unsafe void SetStaticValue(nint InHandle, byte* InName, long InValue)
{
    try
    {
        var Field = GetField(InHandle, InName);
        if (Field is null) { LogUnresolved(InName); return; }

        object? Value;
        var T = Field.FieldType;
        if (!T.IsValueType)                    Value = InValue != 0 ? HandleData.GetObject(InValue) : null;
        else if (T.IsEnum)                     Value = Enum.ToObject(T, InValue);
        else if (T == typeof(bool))            Value = InValue != 0;
        else if (T == typeof(nint))            Value = (nint)InValue;
        else if (T == typeof(nuint))           Value = (nuint)InValue;
        else                                   Value = Convert.ChangeType(InValue, T);

        if (Value is not null && !T.IsInstanceOfType(Value))
            throw new InvalidOperationException($"FieldBridge: {T} <- {Value.GetType()}");

        Field.SetValue(null, Value);
    }
    catch (Exception E) { LogBridge.WriteError($"[Interop] SetStaticValue failed: {E}"); }
}
```

并在 C++ 侧把 `field_bridge_get_static_value_fn` 的返回类型由 `IManagedHandle` 改为真正的标量（或让 C# 侧返回句柄），**两边统一语义**。

**验证方式**
- 单元测试：`HandleData.GetObject` 侧造一个 `static string`，`SetStaticValue(类型句柄, "S", 该字符串的句柄)` → 现状应抛 `ArgumentException`（并杀进程）；修复后字段值正确。
- 同上测 `static bool`（现状 `Convert.ChangeType` 抛）、`static byte = 300`（现状 `OverflowException`）。
- 交叉核对 C++ 调用点：`Select-String -Path Source -Pattern 'FieldBridgeGetStaticValueFn|FieldBridgeSetStaticValueFn'`（现状 0 命中，见 F-CS1-013）。

---

### [F-CS1-012] `ArrayBridge.NewArray` 把数组以 `Pinned` 句柄永久固定：强根 + 固定地址 + `Free` 是唯一解绑路径 → 大数组永久占用且阻碍 GC 压缩

- **类别**: 内存/资源泄漏 / 性能
- **严重度**: **P3**
- **复核结论**: ⚠️部分 —— 严重度由 P2 校正为 **P3**；可达性 活跃
- **文件**: `Script/Interop/Bridge/ArrayBridge.cs:8-24`（关键行 `:19`）
- **函数**: `Interop.ArrayBridge.NewArray(byte*, int)`
- **置信度**: 高

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/ArrayBridge.cs
 8:     [UnmanagedCallersOnly]
 9:     public static nint NewArray(byte* InFullName, int InLength)
10:     {
11:         if (InFullName != null && InLength > 0)
12:         {
13:             var FullName = Marshal.PtrToStringUTF8((nint)InFullName)!;
14: 
15:             var Type = TypeBridge.GetTypeImplementation(FullName);
16: 
17:             if (Type != null)
18:             {
19:                 return HandleData.Alloc(Array.CreateInstance(Type, InLength), true);   // ← pinned: true
20:             }
21:         }
22: 
23:         return 0;
24:     }
```

**调用上下文**
`FClassReflection::NewArray`（`Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp:758-764`）→ `ScriptDomain->NewArray(NameSpace, Name, InNum)`；C++ 侧多用它来构造"字段/属性/方法描述符数组"（`InParams[N]` 那批数组，见 F-CS1-010 的调用上下文）。这些数组在 `FReflectionRegistry` 初始化时批量创建。

**问题**

1. **`GCHandleType.Pinned` 是强根**，因此数组永远不会被 GC 回收；`HandleData.Free` 不被调用就永久存活 —— 与 F-CS1-002 叠加，构成"每次热重载泄漏一整批反射元数据数组"。
2. **Pinned 会阻碍 GC 堆压缩**：pinned 对象使 GC 无法移动它，造成**堆碎片**。反射描述符数组通常不大，但**数量多**（每个类 × 每个属性/字段/方法都有若干数组），大量长期 pinned 对象会显著劣化 LOH/堆压缩效果。
3. `Array.CreateInstance(Type, InLength)` 创建的数组若元素是值类型（例如 `int[]`），`InLength` 大于 85000 字节阈值时会进 **LOH**，pinned LOH 对象几乎永久碎片化堆。
4. **为什么需要 pinned？** 走查 C++ 侧（`FLeanCLRDomain.cpp:125,155,167,176,196`）后发现，`HandleData.GetObjectPointer` 只在 **LeanCLR** 路径被使用，用来把托管对象地址交给 native 侧当"对象引用"：

```cpp
// Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp
125: 		Object_Get_From_Handle(HandleDataGetObjectPointerFn, InManagedHandle));
155: 		if (HandleDataGetObjectPointerFn != nullptr)
167: 		Entry.Get<0>() = Method_Get_From_Handle(HandleDataGetObjectPointerFn, InManagedMethod);
176: 		Object_Constructor(HandleDataGetObjectPointerFn, Entry.Get<0>(), InManagedHandle);
196: 		HandleDataGetObjectPointerFn != nullptr && HandleDataAllocFn != nullptr)
```

   也就是说 **pinned 是"为了给 LeanCLR 当对象指针用"**，但 `NewArray` 把**所有**数组都 pin 了 —— 而 `GetObjectPointer` 走的是任意句柄（对象、字符串、方法、数组都可能）。**只要 pin 数组并不够，pin 全部才是"对的"**：这恰恰说明 **`Normal` 句柄 + `GetObjectPointer` 的组合是真正的不一致**（F-CS1-004）。正确做法是让 `GetObjectPointer` 自己负责"按需临时 pin"或改用 `GCHandle` + `AddrOfPinnedObject`，而不是在 `Alloc` 处一刀切。

**建议**
- 若 native 只是把地址当**不透明句柄**用（做相等比较/当 key），**完全不需要**真实地址：直接用 `HandleData` 的句柄号 `nint`。这样所有 `Pinned` 都可以去掉，改为 `Weak`，同时修掉 F-CS1-004。
- 若 native 确实要读/写托管对象内存（`Object_Constructor` 看起来是构造对象 → 需要地址），则应：
  1. 在 `GetObjectPointer` 内部用 `GCHandle.Alloc(obj, GCHandleType.Pinned)` 临时 pin 并**在 native 使用完毕后由 C# 释放**（需要新增 `Unpin(handle)` 导出），或
  2. 用 `GC.TryStartNoGCRegion` / `fixed` 限定 pin 的作用域，绝不做长期 pin。
- 至少把 `NewArray` 的 `true` 改为 `false`，并在注释里写明"地址不保证稳定，禁止跨 GC 保存"。

**验证方式**
- 在 `NewArray` 后加日志统计 `Array.CreateInstance` 的累计字节数，跑一次完整编辑器启动 + 5 次热重载，观察是否单调增长。
- `Select-String -Path Script -Include *.cs -Pattern 'Alloc\(.*true\)'` 确认只有一处 pinned 调用点。
- 用 `dotnet-counters` 的 `dotnet-gc-heap-size` / `dotnet-gc-fragmentation`（或 PerfView 的 GC pinned 报告）观察 pinned 对象占比。

---

### [F-CS1-013] `FieldBridge` 的 `SetStaticValue`/`GetStaticValue` 被注册但**在 C++ 侧零调用点** → 死代码（且因 F-CS1-011 一旦被调用即崩溃，是"上膛的枪"） —— **已证伪：非缺陷，请从排期移除**

- **类别**: 死代码
- **严重度**: **撤销（非缺陷）** —— 复核**支持**该撤销（"注册了但无（上层）调用者"在 C++ 侧无证据支撑）
- **复核结论**: 部分确认（性质应从"死代码"改述为**"已注册、未被上层调用的导出"**：`IScriptTypes.h:147-148` 的 `Op(...)` 只保证把 C# 函数指针取回并存进 `FieldBridge*Fn` 成员，不等于被业务调用；grep 未发现任何业务调用点，但**无法排除外部/生成代码调用**）
- **可达性**: 潜伏（两个导出函数本身仍会被注册；一旦被调用即命中 F-CS1-011 的错误语义）
- **复核证据**: grep（模式 `SetStaticValue|GetStaticValue`，`Source/**`）= 8 命中，全部是"注册 + 域级包装"而非业务调用：`Source/UnrealCSharpCore/Public/Domain/Script/IScriptTypes.h:147-148`（`Op(FieldBridgeSetStaticValue, ...)` / `Op(FieldBridgeGetStaticValue, ...)`）、`Source/UnrealCSharpCore/Public/Domain/Script/FScriptDomainImpl.inl:396,398`（`if (FieldBridgeSetStaticValueFn != nullptr)` + `SCRIPT_DOMAIN_INVOKE`）、`:408,409`（`FieldBridgeGetStaticValueFn` 同构）、`Source/UnrealCSharpCore/Public/CoreMacro/FunctionMacro.h:119,121`（名称字面量）；`Script/Interop/Bridge/FieldBridge.cs:11` / `:22`（导出定义）。**注意**：`FScriptDomainImpl.inl:396-409` 的域级包装是否还有更上层调用者，未继续追（预算）
- **级别变动**: 维持"撤销（非缺陷）"（类别由"死代码"改述为"零上层的已注册导出"）
- **函数**: `Interop.FieldBridge.SetStaticValue(nint, byte*, long)` / `Interop.FieldBridge.GetStaticValue(nint, byte*)`
- **置信度**: 高（grep 证据见下）

**现状（代码事实）**
函数指针成员由宏 `SCRIPT_TYPE_MEMBER`（`IScriptTypes.h:177`）生成：`FnType Name##Fn{};`，即 `field_bridge_set_static_value_fn FieldBridgeSetStaticValueFn{};`。因此"C++ 是否调用"等价于"源码里是否出现 `FieldBridgeSetStaticValueFn` / `FieldBridgeGetStaticValueFn`"。

grep 证据（`Select-String`，作用域 `Plugins/UnrealCSharp/Source/**` 的 `*.cpp,*.h`，排除 `Intermediate/Binaries/ThirdParty`）：

| 搜索模式（`-SimpleMatch`） | 命中数 | 命中位置 |
|---|---|---|
| `FieldBridgeSetStaticValue` | **1** | 仅 `IScriptTypes.h:147`（`Op(...)` 声明本身） |
| `FieldBridgeGetStaticValue` | **1** | 仅 `IScriptTypes.h:148` |
| `SetStaticValue`（全插件，含 `Script/`） | 3 | `FieldBridge.cs:11`、`FunctionMacro.h:119`、`IScriptTypes.h:147` |
| `GetStaticValue`（全插件） | 3 | `FieldBridge.cs:22`、`FunctionMacro.h:121`、`IScriptTypes.h:148` |

对照"活的"桥接函数以证明该方法学有效：

| 搜索模式 | 命中数 | 命中位置 |
|---|---|---|
| `MethodBridgeInvoke` | 4 | `FLeanCLRDomain.cpp:151,153,182` **+** `IScriptTypes.h:150` |
| `AssemblyLoaderUnload` | 7 | `FCoreCLRDomain.cpp:244,246`、`FLeanCLRDomain.cpp:810,812`、`FMonoDomain.cpp:432,434` **+** `IScriptTypes.h:108` |
| `HandleDataFree` | 3 | `FLeanCLRDomain.cpp:139,143` **+** `IScriptTypes.h:110` |
| `ArrayBridgeArrayGet` | **1** | 仅 `IScriptTypes.h:156` ← **也是死的！** |
| `ArrayBridgeNewArray` | **1** | 仅 `IScriptTypes.h:155` ← **也是死的！** |
| `ObjectBridgeNewObject` | **1** | 仅 `IScriptTypes.h:145` ← **也是死的！** |
| `StringBridgeNewString` / `StringBridgeGetString` | **1** / **1** | 仅 `IScriptTypes.h:152/153` ← **也是死的！** |
| `LogBridgeSetLog` / `LogBridgeInitialize` | **1** / **1** | 仅 `IScriptTypes.h:161/162` ← **也是死的！** |
| `MethodBridgeRegisterBinding` | **1** | 仅 `IScriptTypes.h:166` ← **也是死的！** |

**问题（重要：这推翻了一个常见误判）**

上面这批"命中数 == 1"的桥接**不代表 C# 实现是死代码**，而代表 **C++ 不直接按成员名调用**。真正机制是 `TypeBridgeGetFunctionPointer`（`IScriptTypes.h:164`）：

```cpp
// Source/UnrealCSharpCore/Public/Domain/Script/IScriptTypes.h
21: typedef PTRINT (*type_bridge_get_function_pointer_fn)(const char16_t*, const char16_t*, const char16_t*);
164: 	Op(TypeBridgeGetFunctionPointer, type_bridge_get_function_pointer_fn, CLASS_TYPE_BRIDGE, FUNCTION_TYPE_BRIDGE_GET_FUNCTION_POINTER, 3) \
```

```cpp
// Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRDomain.cpp
172: 		if (TypeBridgeGetFunctionPointerFn != nullptr)
175: 			InFn = reinterpret_cast<decltype(InFn)>(TypeBridgeGetFunctionPointerFn(
// Source/UnrealCSharpCore/Private/Domain/Mono/FMonoDomain.cpp
147: 		InFn = reinterpret_cast<decltype(InFn)>(TypeBridgeGetFunctionPointerFn(
```

即 C++ 通过 **(程序集名, 类型名, 方法名) 三元字符串**向 C# 的 `TypeBridge.GetFunctionPointer(char*, char*, char*)`（`Script/Interop/Bridge/TypeBridge.cs:78`）**按名字要函数指针**，再由 `TypeBridge.cs:92 Method.MethodHandle.GetFunctionPointer()` 把托管方法转换成本地可调用指针。

→ **结论**：`ArrayBridge` / `ObjectBridge` / `StringBridge` / `FieldBridge` / `LogBridge` 这些**并不依赖 `IScriptTypes.h` 里那个 `Op(...)` 生成的成员**，而是通过 `GetFunctionPointer` 的**字符串名**获取。因此 `IScriptTypes.h:145-166` 这批 `Op` 声明**很可能是纯粹的死声明**（既不被直接调用，也没有被 `RegisterBinding` 遍历传递 —— 需在 `FLeanCLRDomain.cpp:854` / `FCoreCLRDomain::RegisterBinding` 中核实）。

**死代码判定（收敛结论）**

| C# 符号 | C++ 直接调用点 | C# 内调用点 | 判定 |
|---|---|---|---|
| `FieldBridge.SetStaticValue` | 0 | 0 | **死代码**（仅经 `GetFunctionPointer` 字符串可达 → 潜在活代码） |
| `FieldBridge.GetStaticValue` | 0 | 0 | 同上 |
| ~~`HandleData.GetObjectPointers`（复数）~~（**已删除**，提交 `4f351f12`） | **0**（无 typedef、无 `Op`、无 `FunctionMacro` 使用者） | **0** | **强死代码**（见第 4 节）**→ 已删除** |
| `HandleData.GetObjectPointer`（单数） | LeanCLR 专属（`FLeanCLRDomain.cpp:125,155,167,176,196`） | **0**（原 1 处为复数版内部调用，随其删除归零） | 活代码（**仅 LeanCLR**） |
| `MethodBridge.GetMethod` | — | 见第 6 节走查 | 待定 |

因此 F-CS1-011 的实际触发条件被收窄为"**未来有人通过 `GetFunctionPointer` 名字路由到 `FieldBridge.SetStaticValue` 时**"。这正是它危险的地方：**代码已经上膛，只是还没人扣扳机**。

**建议**
1. 若 `FieldBridge` 确实不再使用：删除 `FieldBridge.cs` 与 `IScriptTypes.h:147-148` 两行 `Op`，避免后人误用。
2. 若保留：**必须**先修 F-CS1-011（语义错误），再补 C++ 调用点/文档。
3. 核实 `Op(...)` 生成的成员是否真的无用：`Select-String -Path Source -Pattern 'SCRIPT_TYPES|COMMON_BRIDGE_METHODS\(|NATIVE_BRIDGE_METHODS\('`，确认除了 `SCRIPT_TYPE_MEMBER` 之外是否还有第二种展开方式（例如生成"注册表遍历"）。

**验证方式**
- `Select-String -Path Source\UnrealCSharpCore -Include *.cpp,*.h -Pattern 'FieldBridgeSetStaticValue|FieldBridgeGetStaticValue' | Measure-Object` → 期望 1（若为 1 则确认死代码）。
- 在 `RegisterBinding` 中加日志打印实际注册的方法名集合，确认 `FieldBridge` 是否在列。

---

### [F-CS1-014] `LogBridge.Flush` 在 `LogFn(...)` **之后**才 `Buffer.Clear()`，且 `[DllImport]` 与 `delegate*` 两条路径并存 → 重入日志丢失；native 回调可任意线程触发而共享静态字段无同步

- **类别**: 并发/线程安全 / 未定义行为
- **严重度**: **P3**
- **复核结论**: ⚠️部分 —— 严重度由 P2 校正为 **P3**；可达性 活跃
- **文件**: `Script/Interop/Bridge/LogBridge.cs:11-13, 25-28, 30-52, 139-174`
- **函数**: `Interop.LogBridge.Flush()` / `Flush(ReadOnlySpan<char>, bool)` / `SetLog(nint)` / `Deinitialize()`
- **置信度**: 中—高（`Console.SetOut` 的同步包装使常规路径安全，但静态字段本身无保护 → 标中高）

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/LogBridge.cs
  9: public sealed class LogBridge : TextWriter
 10: {
 11:     private static unsafe delegate* unmanaged[Cdecl]<byte*, int, byte, void> LogFn;
 13:     private static TextWriter? ConsoleOut;
 15:     private static TextWriter? ConsoleError;
 17:     private readonly bool bIsError;
 19:     private readonly StringBuilder Buffer = new();
 25:     private static bool bIsLeanCLR;
 27:     [DllImport("UnrealCSharp", CallingConvention = CallingConvention.Cdecl)]
 28:     private static unsafe extern void LogLeanCLR(byte* InBuffer, int InSize, byte InIsError);
...
139:     public override void Flush()
140:     {
141:         if (Buffer.Length > 0)
142:         {
143:             Flush(Buffer.ToString().AsSpan(), bIsError);   // ← native 调用，可能重入
144: 
145:             Buffer.Clear();                                // ← 在 native 返回之后才清空
146:         }
147:     }
...
163:         fixed (byte* Ptr = UTF8)
164:         {
165:             if (LogFn != null)
166:             {
167:                 LogFn(Ptr, Size, IsError);
168:             }
169:             else if (bIsLeanCLR)
170:             {
171:                 LogLeanCLR(Ptr, Size, IsError);
172:             }
173:         }
174:     }
```

**核查结论（先说没问题的部分，避免重复审查）**
- ✅ `stackalloc byte[MaxByteCount]` 的容量安全：`MaxByteCount = GetMaxByteCount(len) + 1`，而 `GetMaxByteCount` 是**上界**（UTF-8 每 char 最多 3 字节，.NET 实现为 `(len + 1) * 3`），`UTF8[Size] = 0`（`:159`）不会越界。512 字节的 `stackalloc` 阈值也远低于默认栈限制。
- ✅ `Dispose(bool disposing)` 在 `disposing == true` 时先 `Flush()`（`:130-133`），不会丢缓冲。`Buffer` 是**实例**字段（`:19`）而非静态，实例只经 `Console.SetOut/SetError` 暴露。
- ✅ `Console.SetOut/SetError`（`:79,81`）在 .NET Core 中会用 `TextWriter.Synchronized` 包装传入的 writer，因此 `Write`/`WriteLine`/`Flush` 的**常规并发调用是被串行化的** —— 这一点降低了严重度，否则 `StringBuilder`（非线程安全）的多线程写入会直接损坏内部状态。
- ⚠️ **重要更正**：任务背景称"`*Bridge.cs` 不是 `DllImport`"。**`LogBridge.cs:27-28` 是唯一的例外** —— 它用了静态 `[DllImport("UnrealCSharp")]` 直连原生模块。原因是 LeanCLR 后端下 `LogFn` 永不被 `SetLog` 赋值（`SetLog` 属于 `NATIVE_BRIDGE_METHODS`，走 `GetFunctionPointer` 名字路由；LeanCLR 的 `RegisterBinding` 在 `FLeanCLRDomain.cpp:854`），所以 LeanCLR 走 DllImport 兜底。**这是双机制并存的架构不一致**：名称字符串机制（`FunctionMacro.h:45`、`IScriptTypes.h:161`）与 DllImport 机制（`FunctionMacro.h:50,52`）同时存在，但 `FUNCTION_LOG_BRIDGE_LOG_LEANCLR`（`FunctionMacro.h:52`）在 `IScriptTypes.h` 里**没有对应的 `Op`**，证实它只服务于 DllImport 路径。

**问题**

1. **重入日志丢失**（`:143-145`）：`Buffer.Clear()` 在 `LogFn`/`LogLeanCLR` **返回之后**执行。而 `LogFn` 指向 UE 的日志函数 —— 它**完全可能**在写日志的过程中触发新的托管代码写出 `Console.Out`（例如 UE 的 log 输出被重定向回脚本层、或崩溃处理中打印脚本栈）。此时新内容被追加到 `Buffer`，随后 `:145` 的 `Clear()` 把它**一并丢弃**。这是**静默丢日志**，恰好发生在"最需要日志"的故障时刻。
   修复：先取走内容再清空。

   ```csharp
   public override void Flush()
   {
       if (Buffer.Length == 0) return;
       var Text = Buffer.ToString();
       Buffer.Clear();                       // ← 先清空，再调用 native
       Flush(Text.AsSpan(), bIsError);
   }
   ```

2. **`LogFn` 静态可变字段无同步**（`:11, 33, 73, 165`）：`SetLog`（初始化期，主线程）与 `Deinitialize`（域销毁期）写，`Flush`（**任意线程**，因为 `Console.WriteLine` 可从 C# 任何线程调用，且 `PendingLog` 可能来自原生工作线程）读。`delegate*` 是裸指针字段，**没有 `volatile`，没有 `Interlocked`，没有锁**。在 x64 上指针读写是原子的，所以**不会撕裂**；但存在**可见性与重排序**问题：`Deinitialize` 把 `LogFn = null`（`:73`）后，并发的 `Flush` 可能仍看到旧非空值并调用**已失效的原生函数指针**（域已销毁）→ **跳转到已卸载的地址**。这正是 F-CS1-003 的同一类风险。
   修复：

   ```csharp
   private static unsafe nint LogFnRaw;                    // 用 nint 便于 volatile
   // SetLog:   Volatile.Write(ref LogFnRaw, InLogFn);
   // Deinit:   Volatile.Write(ref LogFnRaw, 0);
   // Flush:    var Fn = Volatile.Read(ref LogFnRaw); if (Fn != 0) ((delegate* unmanaged[Cdecl]<byte*,int,byte,void>)Fn)(Ptr, Size, IsError);
   ```
   并在 `Deinitialize` 前确保"没有并发日志"（例如先 `Console.SetOut(ConsoleOut)` 恢复原 writer，**再**清空 `LogFn`，当前顺序是反的：`:57-69` 先恢复 writer，`:71-74` 才清 `LogFn` —— 这个顺序**是对的**，值得肯定；但仍有已开始的 `Flush` 在飞）。

3. **`bIsLeanCLR` 是静态可变 `bool` 且无 `volatile`**（`:25, 49, 169`）：`InitializeLeanCLR` 在初始化期写，`Flush` 在工作线程读。同样是可见性问题（虽实践影响小）。

4. **`ConsoleOut`/`ConsoleError` 的静态引用**（`:13, 15, 39-41, 57-69`）：`Deinitialize` 会把它们置 null 并恢复 Console。但**如果 `Deinitialize` 未被调用**（例如进程异常退出、或某个后端路径漏调），这两个字段会一直持有**原来的 Console writer**，而 `Console.Out` 仍指向 `LogBridge` → 形成 `LogBridge → ConsoleOut → ...` 的引用链，使 `Interop` 程序集的这段状态跨热重载一直存活。`Interop` 本来就是宿主程序集（不随热重载卸载，见 F-CS1-001 的分析），所以这是**设计如此**，但意味着 `Deinitialize` 的调用必须严格配对。grep `Bridge_Invoke(LogBridgeInitialize` 相关可确认初始化/反初始化的配对（`FLeanCLRDomain.cpp:69-74` 只看到 Initialize）。

5. **`LogBridge.Flush` 内 `Buffer.ToString()` 每次分配一个字符串**（`:143`）：日志热路径上的必然分配。可改为 `Buffer.GetChunks()` 或 `CopyTo` 到 `Span`：

   ```csharp
   if (!Buffer.TryGetSpan(out var Span)) { Span = Buffer.ToString().AsSpan(); }  // 仅当无分块时
   ```

6. **日志内容未做转义/长度限制**：脚本可以 `Console.WriteLine` 一个超长字符串，`:151-155` 会分配等长（×3）的数组，超大日志会直接吃掉大量内存。建议对单条日志加上限（例如 64 KB 截断）。

**建议**：见上逐点。优先级：第 1 点（丢日志）与第 2 点（悬空函数指针）应尽快修。

**验证方式**
- 重入测试：写一个 C# 的 `Console.WriteLine` 循环，在原生 `LogFn` 实现里反过来再调用一次托管 `Console.Write("x")` → 观察 "x" 是否丢失（现状会丢）。
- 线程测试：多线程并发 `Console.WriteLine` 各 10 万条，校验总行数与内容完整性。
- 竞态测试：在 `Deinitialize` 的同时持续 `Console.WriteLine`，在 ASAN/PageHeap 下观察是否有对已释放地址的调用。

---

### [F-CS1-040] C#→C++ 的名字键 `"Namespace.Class::Method"` 是**两侧字符串拼接、零校验**的隐式契约；且注册整体失败（`InLength == 0` / 函数指针为 `nullptr`）无任何检测 → 契约漂移即"全量 C#→C++ 调用跳转 0"

- **类别**: Bug / 未定义行为
- **严重度**: **P2**（后果是 P0，但触发需要一次不一致的改动）
- **复核结论**: ⚠️部分 —— 严重度 P2 维持；可达性 活跃
- **文件**: 生成端 `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:903,918`；注册端 `Source/UnrealCSharpCore/Public/Domain/Script/FScriptDomainImpl.inl:601-633` + `Source/UnrealCSharpCore/Public/CoreMacro/ClassMacro.h:5` + `Source/UnrealCSharpCore/Public/CoreMacro/BindingMacro.h:7`；消费端 `Script/Interop/Bridge/MethodBridge.cs:10,21,25,144`
- **函数**: `UnrealTypeSourceGenerator.Execute`（key 生成）/ `FScriptDomainImpl::RegisterBinding()` / `Interop.MethodBridge.RegisterBinding` / `Interop.MethodBridge.GetMethod`
- **置信度**: 高（两侧格式都已逐字核对，当前**一致**）

**现状（代码事实）**

生成端拼 key：

```csharp
// Script/SourceGenerator/UnrealTypeSourceGenerator.cs
903:                 var key = $"{containingNamespace}.{InOwner.Name}::{method.Name}";
...
918:                     $"\t\t\t\tref {slot}, \"{key}\"))({arguments});\n" +
```

注册端拼 key（宏展开后是 `"%s.%s"` + `"::%s"`）：

```cpp
// Source/UnrealCSharpCore/Public/CoreMacro/ClassMacro.h
5: #define COMBINE_FULL_NAME(A, B) FString::Printf(TEXT("%s.%s"), *A, *B)
// Source/UnrealCSharpCore/Public/CoreMacro/BindingMacro.h
7: #define BINDING_COMBINE_FUNCTION(Function) FString::Printf(TEXT("::%s"), *Function)
```

消费端只做一次 `Ordinal` 字典查找，失败静默返回 0：

```csharp
// Script/Interop/Bridge/MethodBridge.cs
 10:         private static readonly Dictionary<string, nint> StringToMethod = new(StringComparer.Ordinal);
...
 21:                         var Name = Marshal.PtrToStringUTF8((nint)InNames[Index]) ?? string.Empty;
...
 25:                             StringToMethod[Name] = InMethods[Index];
...
144:                InSlot = StringToMethod.TryGetValue(InName, out var Method) ? Method : nint.Zero;
```

**核查结论（当前一致 ✓）**：`FBinding` 侧登记的 `Method.GetMethod()` **本身**已经是 `"Namespace.Class::Method"` 形式（`FScriptDomainImpl.inl:615` 直接使用，未再拼接），与生成器的 `$"{containingNamespace}.{InOwner.Name}::{method.Name}"` **同构** —— 其中 `InOwner.Name` 是 C# 库桥接类名（如 `Utils`、`TArrayImplementation`），对应 C++ 侧的 `CLASS_*` 宏。因此**现状能工作**（这是一处必须存在、且当前正确的隐式契约，记入第 7.3 节的 negative result）。

**问题**

1. **契约没有任何校验或自测**：这条契约横跨 **C# 源生成器 → 用户程序集 → C# 反射 → C++ 宏** 四层，两端各自插值/`Printf`，中间没有断言、没有单元测试、没有版本标记。任何一侧把分隔符由 `.`/`::` 改掉、把类名换成含命名空间的全名、或把大小写规范化，都会让**全部** `GetMethod` 查找失败 → 按 F-CS1-003，**每一个** C#→C++ 桥接调用都变成 `((delegate*)0)(args)` → 进程立即终止，且**不产生任何日志**（`GetMethod` 不写日志）。
2. **注册整体失败无检测**（`FScriptDomainImpl.inl:603`）：`if (MethodBridgeRegisterBindingFn != nullptr)` —— 若该函数指针为 `nullptr`（初始化顺序问题，或 `TypeBridgeGetFunctionPointer` 返回 0，见第 6 节），**整个 `RegisterBinding` 被跳过**，`StringToMethod` 保持**空字典**。该 `if` **没有 `else`**，没有日志、没有 `ensure`，也没有"注册完成后 `Count > 0`"的自检。表现与"键格式不匹配"完全相同。
3. **`InLength == 0`/`Names` 为空也静默**：`MethodNames.Num() == 0`（`FScriptDomainImpl.inl:631`）时，`MethodBridge.cs:17` 的 `for` 不执行，`RegisterBinding` "成功"返回。
4. **`StringComparer.Ordinal`**（`MethodBridge.cs:10`）本身是**正确**选择 ✓（避免文化敏感比较导致的安全/一致性问题），但也意味着分隔符或大小写的任何一个字符差异都会导致查找失败，放大了第 1 点。
5. **与 F-CS1-018 / F-CS1-003 叠加**：`StringToMethod` 无 `Clear()`，生成代码的 `_Slot` 单向缓存（`GetMethod` 的 `if (InSlot == nint.Zero)`，`MethodBridge.cs:142`）永不失效。而 C++ 侧明确在热重载时把函数指针置空：

```cpp
// Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp
821: #define LEANCLR_METHOD_CLEAR(Name, FnType, ClassNameMacro, MethodNameMacro, ParamCount) Name##Fn = nullptr;
827: 	UTILS_BRIDGE_METHODS(LEANCLR_METHOD_CLEAR)
```
   说明**函数指针确实会在热重载时失效**。正常路径下每次卸载都会新建 ALC（静态槽位归零）✓，但若 `AssemblyLoaderUnloadFn == nullptr` 或 `Context` 被复用，则旧指针会被**永久缓存**。

**建议**

1. **加注册自检**（成本最低、收益最高）：

```csharp
// MethodBridge.cs —— RegisterBinding 末尾
if (StringToMethod.Count == 0)
    Console.Error.WriteLine("[Interop] FATAL: no native bindings registered; every C#->C++ call will crash.");
else
    Console.Error.WriteLine($"[Interop] Registered {StringToMethod.Count} native bindings.");
```
并在 C++ 侧 `FScriptDomainImpl.inl:603` 给 `if` 补一个 `else`（日志 + `ensure`）。
2. **让查询失败可见**（与 F-CS1-003 建议 1 合并）：`GetMethod` 查不到时写一行带 key 的错误，而不是静默 0。
3. **加一个构建期契约测试**：枚举 `FBinding` 的所有方法名，与生成器产出的 key 集合做差集断言为空 —— 唯一能在**构建期**而非运行期抓住契约漂移的手段。
4. **类型安全化**：把 `GetMethod` 的返回类型由 `nint` 改为 `nint?`（或引入 `NativeMethod<T>` 包装），从语言层面消除"强转 0 后立即调用"的可能。
5. `AssemblyLoader.Unload()` 中调用 `MethodBridge.Clear()`（见 F-CS1-001 修复片段）。
6. 给 key 加**版本前缀**（如 `"v1:Namespace.Class::Method"`），使两侧版本不匹配时可立即识别，而不是靠"恰好还能查到"。

**验证方式**
- 把 `BindingMacro.h:7` 的 `"::%s"` 改成 `"#%s"`，重新编译插件并运行 → 现状：崩溃且**无任何日志**；修复后：stderr 出现 `FATAL: no native bindings registered` 或 `Unresolved native method: ...`。
- 在 `RegisterBinding` 末尾打印 `StringToMethod.Count`，与 `FBinding::Get().Register().GetClasses()` 展开后的方法总数比对 → 应相等。
- grep 契约两端：`Select-String -Path Source -Pattern 'BINDING_COMBINE_FUNCTION'` 与 `Select-String -Path Script\SourceGenerator -Pattern 'var key ='`，人工比对格式。

---

### P3 批次：可读性 / 一致性 / 微优化

#### [F-CS1-015] `ObjectBridge` / `ArrayBridge` / `FieldBridge` 使用文件作用域命名空间 `namespace Interop;`，而同目录的 `MethodBridge` / `StringBridge` / `AssemblyLoader` / `HandleData` 用块作用域 `namespace Interop { }` —— 风格不一致

- **类别**: 可优化/可读性
- **严重度**: **P3**
- **复核结论**: ⚠️部分 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Script/Interop/Bridge/ObjectBridge.cs:5`、`Script/Interop/Bridge/ArrayBridge.cs:4`、`Script/Interop/Bridge/FieldBridge.cs:6`、`Script/Interop/Bridge/LogBridge.cs:7` vs `MethodBridge.cs:6`/`StringBridge.cs:5`/`HandleData.cs:5`/`AssemblyLoader.cs:5`/`UnrealAssemblyLoadContext.cs:5`
- **函数**: （命名空间声明）
- **置信度**: 高

**现状（代码事实）**：8 个 Bridge 文件中有 4 个用 file-scoped namespace，4 个用 block-scoped；`AssemblyLoader/`、`Handle/` 子目录全部用 block-scoped。缩进因此也不同（file-scoped 的文件比同类少一级缩进）。

**问题**：纯风格问题，但让"哪些文件属于同一批、由同一作者/同一时间写成"变得难以判断，也使得跨文件的 diff/review 噪音增加。

**建议**：统一为 file-scoped（`namespace Interop;`），这是现代 C# 的推荐写法，也能减少一级缩进。

**验证方式**：`Select-String -Path Script -Include *.cs -Pattern '^namespace'` 统计两种形态的数量。

---

#### [F-CS1-016] `[UnmanagedCallersOnly]` 属性使用不统一：部分显式写 `CallConvs = [typeof(CallConvCdecl)]`，部分裸用属性（依赖默认调用约定）

- **类别**: 平台兼容 / 可读性
- **严重度**: **P3**（**可能是 P2**，取决于默认约定假设是否成立 → 置信度中）
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: 显式写法：`ObjectBridge.cs:9`、`FieldBridge.cs:10,21`、`LogBridge.cs:30,36,46,54`；裸写法：`AssemblyLoader.cs:13,30`、`MethodBridge.cs:12,32`、`StringBridge.cs:8,19`、`ArrayBridge.cs:8,26`、`HandleData.cs:24`
- **函数**: 所有 `[UnmanagedCallersOnly]` 导出
- **置信度**: 中

**现状（代码事实）**：

```csharp
// 显式（例）
Script/Interop/Bridge/ObjectBridge.cs:9:    [UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
// 裸用（例）
Script/Interop/Bridge/MethodBridge.cs:32:        [UnmanagedCallersOnly]
Script/Interop/AssemblyLoader/AssemblyLoader.cs:13:        [UnmanagedCallersOnly]
Script/Interop/Handle/HandleData.cs:24:        [UnmanagedCallersOnly]
```

**问题**：`[UnmanagedCallersOnly]` 不指定 `CallConvs` 时，默认调用约定是**平台默认**（Windows x64 是 `__stdcall` 与 `__fastcall` 合一的约定，Linux x64 是 SysV）。而 C++ 侧的 typedef 签名为 `typedef void (*assembly_loader_unload_fn)();` 之类 —— **MSVC/GCC 的默认函数指针调用约定**。

- 在 **Windows x64**：`__cdecl` 与默认（`__fastcall` 在 x64 上等价）**完全相同**，所以现状能跑。
- 在 **Windows x86 (32-bit)**：`__cdecl` 与 `__stdcall` **不同**（栈清理责任不同）→ 裸用 `[UnmanagedCallersOnly]`（默认 `Winapi` = `__stdcall`）与 C++ 默认（`__cdecl`）**不匹配** → 栈损坏。项目支持 32 位吗？UE5 已不支持 32 位，所以实践影响为 0。
- 在 **Linux x64 / macOS**：两者等价。

→ **结论：现状在 Windows x64 / Linux x64 下正确，属于风格/一致性 P3**；但如果这个文件被当成"模板"复制到需要显式约定的场景（例如 ARM64 或未来的 32 位平台），会引入难查的 ABI bug。**置信度中**，因为我未逐一核对所有 typedef 的调用约定标注（`IScriptTypes.h:5-105` 的 typedef 里**没有**任何 `__cdecl`/`CALLBACK` 标注，依赖编译器默认）。

**建议**：统一全部显式写 `CallConvs = [typeof(CallConvCdecl)]`，并在 `IScriptTypes.h` 的 typedef 上加 `CALLBACK`/等价标注，使"约定"在两侧都是**显式**的。

**验证方式**：`Select-String -Path Script -Include *.cs -Pattern 'UnmanagedCallersOnly' | Measure-Object` 与 `... -Pattern 'UnmanagedCallersOnly\(CallConvs'` 对比数量。

---

#### [F-CS1-017] `FieldBridge.GetField` 每次调用都执行一次反射查找，无任何缓存

- **类别**: 性能
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Script/Interop/Bridge/FieldBridge.cs:34-47`（关键行 `:42`）
- **函数**: `Interop.FieldBridge.GetField(nint, byte*)`
- **置信度**: 高

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/FieldBridge.cs
34:     private static unsafe FieldInfo? GetField(nint InHandle, byte* InName)
35:     {
36:         if (HandleData.GetObject(InHandle) is Type Type)
37:         {
38:             var Name = Marshal.PtrToStringUTF8((nint)InName) ?? string.Empty;
39: 
40:             if (Name.Length > 0)
41:             {
42:                 return Type.GetField(Name, BindingFlags.Static | BindingFlags.Public | BindingFlags.NonPublic);
43:             }
44:         }
45: 
46:         return null;
47:     }
```

**问题**：每次调用包含 (a) `HandleData.GetObject`（加锁 + 字典查找）、(b) **一次 UTF-8 解码分配**（`:38` 每调用一次分配一个 `string`）、(c) `Type.GetField` 反射查找（字符串比较、`BindingFlags` 位运算、`MemberInfo` 数组遍历）。若在循环里读写静态字段，这是显著的重复开销。而字段元数据是**完全可缓存**的（键 = (类型句柄, 字段名)）。

**建议**

```csharp
private static readonly System.Threading.Lock CacheLock = new();
private static readonly Dictionary<(nint, string), FieldInfo?> Cache = new();

private static FieldInfo? GetField(nint InHandle, byte* InName)
{
    var Name = Marshal.PtrToStringUTF8((nint)InName);
    if (string.IsNullOrEmpty(Name)) return null;

    lock (CacheLock)
    {
        if (Cache.TryGetValue((InHandle, Name), out var Cached)) return Cached;
    }

    var Found = HandleData.GetObject(InHandle) is Type T
        ? T.GetField(Name, BindingFlags.Static | BindingFlags.Public | BindingFlags.NonPublic)
        : null;

    lock (CacheLock) { Cache[(InHandle, Name)] = Found; }
    return Found;
}
```
⚠️ **缓存必须在 `AssemblyLoader.Unload()`/热重载时清空**（类型句柄会失效），否则与 F-CS1-001 叠加成新的泄漏源并可能命中已卸载类型。这也说明：插件里**每个 Bridge 的缓存字典都必须有一个 `Clear()` 并被 `Unload` 调用**，当前只有 `TypeBridge.Clear()` 存在（`AssemblyLoader.cs:37`），而 `MethodBridge.StringToMethod` **没有** `Clear()`。

**验证方式**：基准测试循环调用 10 万次 `GetStaticValue`，比较加缓存前后耗时与 `GC.GetAllocatedBytesForCurrentThread()`。

---

#### [F-CS1-018] `MethodBridge.StringToMethod` 缓存字典**没有 `Clear()` 方法**，`Unload` 无法清空 → 热重载后残留指向已销毁原生域的函数指针

- **类别**: 内存/资源泄漏 / 死代码（缺失的 API）
- **严重度**: **P3**
- **复核结论**: ⚠️部分 —— 严重度由 P2 校正为 **P3**；可达性 活跃
- **文件**: `Script/Interop/Bridge/MethodBridge.cs:10`（字典声明）；缺失的清理点应对应 `Script/Interop/AssemblyLoader/AssemblyLoader.cs:31-61`
- **函数**: `Interop.MethodBridge`（缺少 `Clear()`）
- **置信度**: 高

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/MethodBridge.cs
10:         private static readonly Dictionary<string, nint> StringToMethod = new(StringComparer.Ordinal);
```

全文件（`:1-214`）**没有任何** `Clear` / `Remove` / 重新赋值。对比 `TypeBridge`（有 `Clear()`，被 `AssemblyLoader.cs:37` 调用）和 `HandleData`（有 `Clear()`，被 `AssemblyLoader.cs:35` 调用）—— `MethodBridge` 是**三者中唯一缺少清理入口**的。

grep 证据：

| 模式 | 命中数 | 说明 |
|---|---|---|
| `StringToMethod` | 3（`MethodBridge.cs:10,25,144`） | 仅本文件，无外部可见性 |
| `MethodBridge.Clear` | **0** | 不存在 |
| `TypeBridge.Clear` | 1（`AssemblyLoader.cs:37`） | 存在且被调用 |
| `HandleData.Clear` | 1（`AssemblyLoader.cs:35`） | 存在且被调用 |

**问题**
`StringToMethod` 的值是**原生函数指针** `nint`。这些指针指向 C++ 侧的函数（可能是脚本域内的 thunk）。`FLeanCLRDomain::UnloadAssembly`（`FLeanCLRDomain.cpp:821-827`）在卸载时把 `Name##Fn = nullptr` 清零并销毁脚本域：

```cpp
// Source/UnrealCSharpCore/Private/Domain/LeanCLR/FLeanCLRDomain.cpp
821: #define LEANCLR_METHOD_CLEAR(Name, FnType, ClassNameMacro, MethodNameMacro, ParamCount) Name##Fn = nullptr;
823: 	COMMON_BRIDGE_METHODS(LEANCLR_METHOD_CLEAR)
825: 	LEANCLR_INTEROP_BRIDGE_METHODS(LEANCLR_METHOD_CLEAR)
827: 	UTILS_BRIDGE_METHODS(LEANCLR_METHOD_CLEAR)
```

因此热重载后 `StringToMethod` 中残留的 `nint` 是**指向已失效代码的指针**，且**永远不会被刷新**（`GetMethod` 的 `if (InSlot == nint.Zero)` 只在槽位为空时查询，见 F-CS1-003）。这是 F-CS1-003 "调用即崩" 的**直接成因**。

**建议**

```csharp
// MethodBridge.cs 新增
internal static void Clear()
{
    lock (StringToMethod) { StringToMethod.Clear(); }
}
```
并在 `AssemblyLoader.Unload()` 中与 `HandleData.Clear()` / `TypeBridge.Clear()` 并列调用（见 F-CS1-001 的修复片段）。**注意**：`StringToMethod` 声明为 `readonly Dictionary` 且**从未加锁**，而 `RegisterBinding`（写）与 `GetMethod`（读）可能在不同线程（`RegisterBinding` 在主线程初始化期，`GetMethod` 由脚本线程惰性调用）→ **并发读写 `Dictionary` 是未定义行为**（可能死循环或抛异常）。见下一条。

**验证方式**
- `Select-String -Path Script -Include *.cs -Pattern 'MethodBridge\.'` 确认无 `Clear` 调用。
- 热重载两次后在 C# 侧打印 `StringToMethod.Count`：现状不回落。

---

#### [F-CS1-019] `MethodBridge.StringToMethod` 字典存在**并发读写竞态**：`RegisterBinding` 写、`GetMethod` 读，全程无锁

- **类别**: 并发/线程安全
- **严重度**: **P2**
- **复核结论**: 确认（代码事实成立；可达性判定为"潜伏"，证据边界已补全）
- **可达性**: 潜伏（竞态窗口只在"脚本线程已跑起来 + 注册仍在写"时存在；当前工程启动期注册先于脚本执行 → 暂不触发）
- **复核证据**: `Script/Interop/Bridge/MethodBridge.cs:10`（`private static readonly Dictionary<string, nint> StringToMethod`，**无任何同步原语**）、`:25`（`RegisterBinding` 写 `StringToMethod[Name] = InMethods[Index]`）、`:144`（`GetMethod` 读 `StringToMethod.TryGetValue`）—— 读写两侧全程无 `lock`/`ConcurrentDictionary`；对照 `HandleData.cs:9`（`System.Threading.Lock Lock`）证明同一批 Bridge 里确实存在"该加锁却没加"的差异
- **级别变动**: 维持 P2
- **文件**: `Script/Interop/Bridge/MethodBridge.cs:10, 25, 144`
- **函数**: `Interop.MethodBridge.RegisterBinding(byte**, nint*, int)` / `GetMethod(ref nint, string)`
- **置信度**: 中—高（并发场景的存在性取决于脚本线程是否在 `RegisterBinding` 之后运行 → 标中高）

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/MethodBridge.cs
10:         private static readonly Dictionary<string, nint> StringToMethod = new(StringComparer.Ordinal);
...
25:                             StringToMethod[Name] = InMethods[Index];        // 写（RegisterBinding，无锁）
...
144:                InSlot = StringToMethod.TryGetValue(InName, out var Method) ? Method : nint.Zero;   // 读（GetMethod，无锁）
```

**问题**
`Dictionary<TKey,TValue>` **不是线程安全的**：并发的"写 + 读"（或两个写）会破坏内部桶链，最坏情况是**无限循环**（经典症状：`TryGetValue` 卡死在 `FindEntry` 的循环里）或读到半更新的状态。
- 写侧：`RegisterBinding`（`MethodBridge.cs:13`）由脚本域初始化调用（`FCoreCLRDomain.cpp:197`、`FLeanCLRDomain.cpp:55`、`FMonoDomain.cpp:169`），在**游戏线程 / 初始化阶段**。
- 读侧：`GetMethod`（`:140`）由**脚本业务代码**在任意时刻惰性调用。若脚本有后台线程（`Task.Run`、`Thread`）或 `SynchronizationContext` 的 Tick 在别的线程跑（`synchronization_context_tick_fn`，`IScriptTypes.h:105`），就与"热重载后重新 `RegisterBinding`"竞争。
- 插件自身在 LeanCLR 下 `RegisterBinding` 发生在 `FLeanCLRDomain.cpp:55`（`Initialize` 内），而 `UnloadAssembly`（`:806`）并不清理 `StringToMethod`（见 F-CS1-018）→ **第二次初始化会再次写同一个字典**，若此时有脚本线程在读，就是真实的竞态。

**同目录对照（值得肯定）**：`HandleData.cs:9` 专门用了 `System.Threading.Lock Lock` 保护全部字典操作，说明作者**知道**要加锁；`MethodBridge` 是遗漏。

**建议**

```csharp
// 方案 A：加锁（与 HandleData 风格一致）
private static readonly System.Threading.Lock Lock = new();
// RegisterBinding: lock (Lock) { StringToMethod[Name] = InMethods[Index]; }
// GetMethod:      lock (Lock) { InSlot = StringToMethod.TryGetValue(...) ? ... : nint.Zero; }

// 方案 B：初始化完成后字典只读 → 用不可变快照替换
private static Dictionary<string, nint> StringToMethod = new(StringComparer.Ordinal);
// RegisterBinding 末尾: Volatile.Write(ref StringToMethod, Snapshot);
// GetMethod 读取时先 Volatile.Read 拿到本地引用
```
方案 B 更好：`RegisterBinding` 是初始化期的批量写入，之后字典只读，读侧无锁。

**验证方式**
- 压力测试：一个线程循环调 `GetMethod`（用不同的未命中名字迫使每次都查表），另一个线程循环调 `RegisterBinding` 注册新名字，运行 1 分钟后检查是否卡死。
- 用 `dotnet-counters` 或并发 profiler 观察 `MethodBridge.GetMethod` 的锁争用（方案 A）。

### [F-CS1-020] `TypeBridge.MakeGenericType2` 复制粘贴错误：用 `InKeyType` 两次代替 `InValueType` → 所有双泛型参数类型（`Dictionary<K,V>`、`TMap`）都退化成 `Dictionary<K,K>`

- **类别**: Bug
- **严重度**: **P1**（功能错误；下游可升级为 P0 —— 类型不匹配会在 `MethodBridge.Invoke` 里以异常形式爆发）
- **复核结论**: 确认（逐字成立，**错误行精确命中 `:194`**）
- **可达性**: 活跃（`Dictionary<K,V>` / `TMap<K,V>` 走 `MakeGenericType2` 时即生效）
- **复核证据**: `Script/Interop/Bridge/TypeBridge.cs:188`（签名 `MakeGenericType2(nint InGeneric, nint InKeyType, nint InValueType)`）→ `:190` `GetObject(InGeneric)` → `:192` `GetObject(InKeyType) is Type Key` → **`:194` `if (HandleData.GetObject(InKeyType) is Type Value)`（应为 `InValueType`）** → `:196` `Generic.MakeGenericType(Key, Value)`。`InValueType` 参数**在整个方法体内从未被使用**；当 `InValueType` 句柄指向不同类型（如 `TMap<FString, AActor>` 的 value 端）时 `Value == Key` → 生成 `Dictionary<K,K>`
- **级别变动**: 维持 P1
- **文件**: `Script/Interop/Bridge/TypeBridge.cs:187-202`（错误行 `:194`）
- **函数**: `Interop.TypeBridge.MakeGenericType2(nint, nint, nint)`
- **置信度**: 高（逐行核对，非推测）

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/TypeBridge.cs
187:     [UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
188:     public static nint MakeGenericType2(nint InGeneric, nint InKeyType, nint InValueType)
189:     {
190:         if (HandleData.GetObject(InGeneric) is Type Generic)
191:         {
192:             if (HandleData.GetObject(InKeyType) is Type Key)
193:             {
194:                 if (HandleData.GetObject(InKeyType) is Type Value)   // ← 应为 InValueType
195:                 {
196:                     return HandleData.Alloc(Generic.MakeGenericType(Key, Value));
197:                 }
198:             }
199:         }
200: 
201:         return 0;
202:     }
```

`:194` 读的是 **`InKeyType`**，而参数名是 `InValueType`。对照紧邻的单泛型版本 `MakeGenericType`（`:173-185`），它正确地用了传入的第二个句柄：

```csharp
173:     [UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
174:     public static nint MakeGenericType(nint InGeneric, nint InType)
176:         if (HandleData.GetObject(InGeneric) is Type Generic)
178:             if (HandleData.GetObject(InType) is Type Type)
180:                 return HandleData.Alloc(Generic.MakeGenericType(Type));
```

**调用上下文**
C++ 侧通过 `TypeBridgeGetMakeGenericType2Fn`（`IScriptTypes.h:119`：`Op(TypeBridgeMakeGenericType2, type_bridge_make_generic_type2_fn, CLASS_TYPE_BRIDGE, FUNCTION_TYPE_BRIDGE_MAKE_GENERIC_TYPE2, 3)`，typedef 见 `IScriptTypes.h:31`）调用，用于构造"键值对泛型"。

**C++ 侧的合约（关键证据）**：

```cpp
// Source/UnrealCSharpCore/Public/Domain/Script/IScriptTypes.h
29: typedef IManagedHandle (*type_bridge_make_generic_type_fn)(IManagedHandle, IManagedHandle);
31: typedef IManagedHandle (*type_bridge_make_generic_type2_fn)(IManagedHandle, IManagedHandle, IManagedHandle);
```

三个句柄分别是 **泛型定义 / 键类型 / 值类型** —— 签名本身是对的，错的是 C# 实现读错了参数。

**问题**
`Value` 与 `Key` 来自**同一个句柄** `InKeyType`，因此 `Generic.MakeGenericType(Key, Value)` 等价于 `Generic.MakeGenericType(Key, Key)`。
- UE 的 `TMap<FString, AActor>` → `Dictionary<string, string>`（值类型被键类型覆盖）。
- UE 的 `TSet<T>` 用的是单泛型版本，不受影响；**所有 TMap / 双泛型容器 / `TSubclassOf` 相关的双泛型构造全部出错**。
- `InValueType` 参数被**完全忽略**，且没有任何日志 —— 静默产生错误的类型。

**连锁后果**（为什么可能升级为 P0）
错误的 `Type` 会被 `HandleData.Alloc(Type)` 登记并交给 C++，C++ 用它构造数组/做成员读写。当后续：
1. `ArrayBridge.NewArray`（`ArrayBridge.cs:15,19`）用它 `Array.CreateInstance` → 得到一个元素类型错误的数组；C++ 读出来的句柄指向错误类型的对象 → `TypeBridge.UnboxXxx`（`:271-444`）的 `is int` / `is bool` 模式匹配失败 → **静默返回 0**（返回值被当作"未装箱成功"），数据丢失；
2. 或 `MethodBridge.Invoke`（`:37,71`）把它用于参数断言 → `TargetInvocationException`/`ArgumentException` 被 `:125` 捕获 → C++ 看到"返回 null"（见 F-CS1-006）；
3. 若 C++ 侧对该类型做 `reinterpret_cast`，就是真正的内存破坏。

**建议**
一行修复：

```csharp
if (HandleData.GetObject(InValueType) is Type Value)
```

**同时建议**：这个 bug 之所以能长期潜伏，是因为 `MakeGenericType2` 没有单元测试。建议补一个测试：

```csharp
var Dict = TypeBridge.GetClass("System.Collections.Generic.Dictionary`2"u8);   // 泛型定义
var K = TypeBridge.GetClass("System.String"u8);
var V = TypeBridge.GetClass("System.Int32"u8);
var Made = TypeBridge.MakeGenericType2(Dict, K, V);
var T = (Type)HandleData.GetObject(Made)!;
Assert.Equal(typeof(int), T.GetGenericArguments()[1]);   // 现状会得到 string → 测试失败
```

**验证方式**
- 上面这个断言：现状 `GetGenericArguments()[1]` 会返回 `System.String`（= `Key`），修复后返回 `System.Int32`。
- grep 复核同类错误：`Select-String -Path Script -Include *.cs -Pattern 'GetObject\(In(\w+)\) is Type (\w+)'`，逐条核对模式变量与参数名的对应关系（本例是唯一一处不匹配）。
- 运行期：在 `MakeGenericType2` 里加 `Debug.Assert(Key != Value || ...)`，或在返回前打印 `$"{Generic.Name}<{Key.Name},{Value.Name}>"` 核对。

---

### [F-CS1-021] `GetNamespace` / `GetName` / `GetFullName` 按**字符数**而非**字节数**截断，再交给 UTF-8 编码 → 非 ASCII 类型名会抛 `ArgumentException` 并穿越 `[UnmanagedCallersOnly]` = 进程终止

- **类别**: Bug / 崩溃 / 平台兼容
- **严重度**: **P2**（维持；"进程终止"同样是后端专属后果）
- **复核结论**: 确认（三处代码事实逐字成立；**"= 进程终止"须按后端区分**）
- **可达性**: 活跃（仅当类型名/命名空间含非 ASCII 时触发；触发后 LeanCLR 下为 `RtResult` 零值）
- **复核证据**: `Script/Interop/Bridge/TypeBridge.cs:111-112`（`Encoding.UTF8.GetBytes(Namespace.AsSpan(0, Math.Min(Namespace.Length, InStringSize - 1)), String)` —— 先按**字符数** `Namespace.Length` 截断再交给 UTF-8 编码器；非 ASCII 时 UTF-8 字节数 > 字符数 → 目标 `Span<byte>` 不足 → `ArgumentException`）、`:134-135`（`GetName` 同构）、`:161-162`（`GetFullName` 同构）
- **修正（后端区分）**: LeanCLR 下托管异常**不终止进程**（走 `RtResult` 返回值协议、调用方拿到零值）；只有 CoreCLR/Mono 下才 fail-fast 杀进程（**潜伏**）。故本条在五平台 LeanCLR 下的实际症状是"类型名静默为空/零值"，仍属功能错误 → P2 维持
- **级别变动**: 维持 P2（原 P0/P1 的历史降级方向正确）
- **文件**: `Script/Interop/Bridge/TypeBridge.cs:100-121, 123-144, 146-171`（关键行 `:112`、`:135`、`:162`）
- **函数**: `Interop.TypeBridge.GetNamespace(nint, byte*, int)` / `GetName(nint, byte*, int)` / `GetFullName(nint, byte*, int)`
- **置信度**: 高

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/TypeBridge.cs
100:     [UnmanagedCallersOnly]
101:     public static unsafe int GetNamespace(nint InHandle, byte* OutString, int InStringSize)
102:     {
103:         if (OutString != null && InStringSize > 0)
104:         {
105:             if (HandleData.GetObject(InHandle) is Type Type)
106:             {
107:                 var Namespace = Type.Namespace ?? string.Empty;
108: 
109:                 var String = new Span<byte>(OutString, InStringSize);      // 目标 = InStringSize 字节
110: 
111:                 var Length = Encoding.UTF8.GetBytes(
112:                     Namespace.AsSpan(0, Math.Min(Namespace.Length, InStringSize - 1)), String);
113:                                                    // ↑ 按“字符数”截断到 InStringSize-1
114:                 String[Length] = 0;
115: 
116:                 return Length;
...
132:                 var String = new Span<byte>(OutString, InStringSize);
134:                 var Length = Encoding.UTF8.GetBytes(
135:                     Name.AsSpan(0, Math.Min(Name.Length, InStringSize - 1)), String);
137:                 String[Length] = 0;
...
159:                 var String = new Span<byte>(OutString, InStringSize);
161:                 var Length = Encoding.UTF8.GetBytes(
162:                     FullName.AsSpan(0, Math.Min(FullName.Length, InStringSize - 1)), String);
164:                 String[Length] = 0;
```

**问题**
`Math.Min(X.Length, InStringSize - 1)` 限制的是**源字符个数**，不是**目标字节数**。UTF-8 下一个 `char` 最多编码 3 字节（代理对为 4 字节/2 char，仍可能超过 1:1）。因此当名字含非 ASCII 字符时，`GetBytes` 需要的字节数最多可达 `3 × (InStringSize - 1)`，而目标 `Span<byte>` 只有 `InStringSize` 字节：

- `Encoding.UTF8.GetBytes(ReadOnlySpan<char>, Span<byte>)` 在目标不足时**抛 `ArgumentException`**（"destination buffer too small"），不会自动截断。
- 该异常在 `[UnmanagedCallersOnly]` 方法内逃逸 → 运行时终止进程（同 F-CS1-010）。

**具体触发算例**：`InStringSize = 16`，命名空间 `"我的游戏模块"`（6 个 CJK 字符 = 18 字节 UTF-8）。
- 源被截到 `min(6, 15) = 6` 个字符 → 6 个 CJK。
- `GetBytes` 需要 18 字节，目标只有 16 字节 → **抛 `ArgumentException`，进程终止**。

**触发条件现实性**：C++ 侧缓冲区大小的来源需核实（`type_bridge_get_namespace_fn`，`IScriptTypes.h:23`），但从 C++ 传入的大小通常按 `FString` 容量估算。**C# 类型名/命名空间在 UE-C# 项目里经常被自动生成且可能含非 ASCII**（编辑器里用中文命名类；或 `Template/` 目录里若含日文/中文注释性命名）。此外 **`GetFullName` 还会拼上程序集简单名**（`:157`），如果用户程序集名含中文（UE 工程名允许中文），命中面更大。

**建议**
改为"先算需要多少字节，再按字节截断"，或直接用 `Encoder.Convert`：

```csharp
// 方案 A：用 Encoder.Convert，它会安全截断到目标容量
var Encoder = Encoding.UTF8.GetEncoder();
Encoder.Convert(Source.AsSpan(), String, flush: true, out _, out var BytesUsed, out _);
String[BytesUsed] = 0;
return BytesUsed;

// 方案 B：显式循环，保证不切断多字节序列
var MaxBytes = InStringSize - 1;
var Used = 0;
foreach (var Rune in Source.EnumerateRunes())
{
    var Need = Rune.Utf8SequenceLength;
    if (Used + Need > MaxBytes) break;
    Rune.EncodeToUtf8(String.Slice(Used));
    Used += Need;
}
String[Used] = 0;
return Used;
```

并**加 try/catch + 明确的截断语义文档**（"返回实际写入字节数，超出即截断"），不要让它抛。

**顺带修一个相关问题**（`:153-157`）：

```csharp
153:                 var TypeFullName = Type.IsGenericTypeDefinition
154:                     ? $"{Type.Namespace}.{Type.Name}"
155:                     : Type.FullName;
157:                 var FullName = $"{TypeFullName}, {Type.Assembly.GetName().Name}";
```
- `Type.FullName` 对**泛型类型参数 / 开放泛型 / 某些 `ref struct`** 会返回 **null** → 插值得到 `", MyAssembly"`（前导逗号 + 空格），是一个**语法上无效的程序集限定名**，C++ 侧再回传给 `GetClass` 时无法解析。
- `Type.Namespace` 为 null 时（全局命名空间类型）`:154` 得到 `".MyType"`（前导点），同样无效。
- 建议：`TypeFullName ?? Type.Name`，并在 `Namespace` 为空时不要拼点。

**验证方式**
- 单测：构造一个在中文命名空间下的类型，`GetName(handle, buf, 16)` 且命名含 6 个 CJK → 现状抛 `ArgumentException`（进程终止），修复后返回按字节截断的长度。
- 边界测试：`InStringSize = 1` → `MaxBytes = 0`，`Math.Min(len, 0) = 0` → 只写 `String[0] = 0`，返回 0，**不崩**（这一边界现状是安全的，值得记录）。
- 断言每次返回前 `Debug.Assert(Length < InStringSize)`。

---

### [F-CS1-022] `TypeBridge.GetMethod` 只用"方法名 + 参数个数"匹配重载，且无缓存 → 同参数量重载静默选错

- **类别**: Bug / 性能
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Script/Interop/Bridge/TypeBridge.cs:48-75`（匹配逻辑 `:63-69`）
- **函数**: `Interop.TypeBridge.GetMethod(nint, byte*, int)`
- **置信度**: 高（匹配逻辑确定）；"是否会真的选错"取决于脚本层是否存在同参数量重载 → 中

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/TypeBridge.cs
48:     [UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
49:     public static unsafe nint GetMethod(nint InHandle, byte* InName, int InParamCount)
50:     {
51:         if (InName != null)
52:         {
53:             if (HandleData.GetObject(InHandle) is Type Type)
54:             {
55:                 var Name = Marshal.PtrToStringUTF8((nint)InName)!;
56: 
57:                 if (!string.IsNullOrEmpty(Name))
58:                 {
59:                     const BindingFlags BindingFlag =
60:                         BindingFlags.Public | BindingFlags.NonPublic |
61:                         BindingFlags.Static | BindingFlags.Instance;
62: 
63:                     foreach (var Method in Type.GetMethods(BindingFlag))
64:                     {
65:                         if (Method.Name == Name && Method.GetParameters().Length == InParamCount)
66:                         {
67:                             return HandleData.Alloc(Method);
68:                         }
69:                     }
70:                 }
71:             }
72:         }
73: 
74:         return 0;
75:     }
```

**问题**

1. **重载歧义（正确性）**：`Method.Name == Name && 参数个数 == InParamCount` 完全**不考虑参数类型**。C++ 侧只提供名字与个数（typedef `IScriptTypes.h:19`：`IManagedHandle (*type_bridge_get_method_fn)(IManagedHandle, const uint8*, int32)`），因此：
   - `void Foo(int)` 与 `void Foo(string)` 同参数量 → `Type.GetMethods` 的返回顺序**不保证**，`foreach` 取第一个 → **可能选错重载**。
   - 后果通过 `MethodBridge.Invoke`（`:37,71`）暴露：参数被按错误的类型读取（`ReadPrimitiveValue`，`:157-175` 按 `ParameterType` 解释同一段内存）→ 读到垃圾值，或抛 `ArgumentException`（被 `:125` 吞掉，见 F-CS1-006）。
   - `BindingFlags` 同时含 `Instance | Static`（`:60-61`）→ 若存在同名的**静态与实例**方法且参数量相同，也可能选错（后续 `Method.Invoke(Object, ...)`（`MethodBridge.cs:71`）对静态方法会用非 null 的 `Object`，反之亦然 → `TargetException`）。
   - `Type.GetMethods` 还**包含继承链**上的方法（`NonPublic` 含 `protected`/`internal`）→ 基类与派生类同名同参数量时先遍历到哪个不确定。

2. **无缓存（性能）**：每次调用都 (a) UTF-8 解码分配 `string`（`:55`）；(b) `Type.GetMethods(...)` 每次返回**新数组**（`MemberInfo` 缓存不能复用为数组）；(c) 循环内 `Method.GetParameters()`（`:65`）**每次分配一个 `ParameterInfo[]`**。对含 N 个方法的类型，最坏 N 次分配。这在 C++ 反射初始化阶段（`FReflectionRegistry::Initialize`）是批量调用的 → 一次启动可产生数万次分配。

3. **未找到时静默返回 0**（`:74`）：与 F-CS1-003 同类；`GetMethod` 返回 0 后 C++ 侧若不做 `IManagedHandleIsValid` 判断就会以 0 去 `MethodBridge.Invoke`（`:37` `HandleData.GetObject(0)` → null → 返回 0），表现为"方法静默不执行"。

**建议**

1. **把参数类型纳入匹配键**。既然 C++ 只有名字+个数，就让 C++ 也传参数类型名（或让 C# 侧在**注册阶段**由生成器提供精确签名）。最小改动：当同名同参数个数的候选**多于一个**时，记录一条错误日志并以 `AmbiguousMatchException` 语义失败，而不是静默取第一个：

```csharp
MethodInfo? Found = null;
foreach (var Method in Type.GetMethods(BindingFlag))
{
    if (Method.Name != Name || Method.GetParameters().Length != InParamCount) continue;
    if (Found is not null)
    {
        Console.Error.WriteLine($"[Interop] Ambiguous method: {Type.FullName}.{Name}/{InParamCount}");
        return 0;              // 或抛，交由统一兜底
    }
    Found = Method;
}
return Found is null ? 0 : HandleData.Alloc(Found);
```

2. **加缓存**（键 = (类型句柄, 名字, 参数个数)），并在 `Clear()`（`:501`）里一并清空：

```csharp
private static readonly Dictionary<(nint, string, int), MethodInfo?> MethodCache = new();
```

3. **复用参数数组**：用 `Type.GetMethods(BindingFlag)` 一次拿全部，把"名字 → (个数 → MethodInfo)"预先构造成 `ILookup`/嵌套字典，后续 O(1)。

**验证方式**
- 单测：定义一个含 `Foo(int)` 与 `Foo(string)` 的类型，调 `GetMethod(type, "Foo", 1)` 两次 → 现状可能返回不同结果或稳定返回错误那个；修复后应记录歧义并返回 0。
- 基准：对含 200 个方法的类型调 `GetMethod` 1 万次，测耗时与分配（现状应显著高于加缓存后）。
- grep：`Select-String -Path Script -Include *.cs -Pattern 'Type.GetMethods'` 统计同类无缓存反射点。

---

### [F-CS1-023] `TypeBridge.StringToType` 与 `MethodBridge.StringToMethod` 一样**并发无锁**，且 `Clear()` 也不加锁 → 与 `HandleData` 的加锁风格不一致

- **类别**: 并发/线程安全
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Script/Interop/Bridge/TypeBridge.cs:12, 448, 471, 501-504`
- **函数**: `Interop.TypeBridge.GetTypeImplementation(string)` / `Clear()`
- **置信度**: 中—高

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/TypeBridge.cs
12:     private static readonly Dictionary<string, Type> StringToType = new(StringComparer.Ordinal);
...
446:     internal static Type? GetTypeImplementation(string InFullName)
447:     {
448:         if (StringToType.TryGetValue(InFullName, out var OutType))     // 读，无锁
449:         {
450:             return OutType;
451:         }
...
469:         if (Type != null)
470:         {
471:             StringToType.TryAdd(InFullName, Type);                      // 写，无锁
472:         }
473: 
474:         return Type;
475:     }
...
501:     internal static void Clear()
502:     {
503:         StringToType.Clear();                                          // 清空，无锁
504:     }
```

**调用上下文**
- 读/写：`GetTypeImplementation` 被 `GetClass`（`:23`）、`ArrayBridge.NewArray`（`ArrayBridge.cs:15`）、`GetFunctionPointer`（`:82`）调用，全部是 `[UnmanagedCallersOnly]` 导出 → **可能从任意线程进入**。
- `Clear()`：`AssemblyLoader.Unload()`（`AssemblyLoader.cs:37`），在脚本域卸载时（游戏线程）调用。

**问题**
`Dictionary` 的非并发 `TryGetValue` + `TryAdd` 组合在多线程下会破坏内部状态：可能读到半更新条目、`TryAdd` 覆盖、或在 resize 期间进入死循环。`Clear()` 与并发的 `TryGetValue` 竞争尤其危险（`Clear` 会把 `_buckets` 换新，正在遍历的线程可能读到已释放的 `Entry[]` 索引 → 返回错误类型或抛索引异常）。
另外 `Clear()` 本身也**没有**加锁，而 `HandleData.Clear()`（`HandleData.cs:129-145`）是**加了锁**的 —— 同一批桥接里锁策略不一致。

**建议**
- 与 `HandleData` 保持一致，引入 `System.Threading.Lock`（`TypeBridge.cs` 顶部），并在 `GetTypeImplementation`、`Clear` 中加锁。
- 更好的方案：把 `Dictionary` 换成 `ConcurrentDictionary<string, Type>`（`TryGetValue`/`TryAdd`/`Clear` 都是线程安全的原子操作，且 `GetOrAdd` 可省掉一次查找）：

```csharp
private static readonly System.Collections.Concurrent.ConcurrentDictionary<string, Type> StringToType =
    new(StringComparer.Ordinal);
```

**验证方式**
- 压力测试：8 个线程并发调用 `GetClass`（用 200 个互不相同的类型名迫使每次走 miss + 插入），同时另一线程周期性调 `Clear()`，运行 30 秒；现状可能出现异常/卡死。
- 用 `dotnet-counters` 的 `System.Runtime` 事件观察是否有 `InvalidOperationException`。

---

### [F-CS1-024] `TypeBridge.GetTypeImplementation` 在无程序集限定名时**线性扫描当前域所有程序集**并取首个同名类型 → 可能绑定到错误的同名类型（且每次都要全扫描，无负缓存）

- **类别**: Bug / 性能
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Script/Interop/Bridge/TypeBridge.cs:446-499`（关键行 `:453-467`, `:480-496`）
- **函数**: `Interop.TypeBridge.GetTypeImplementation(string)` / `GetTypeImplementation(IEnumerable<Assembly>, string?, string)`
- **置信度**: 中—高

**现状（代码事实）**

```csharp
// Script/Interop/Bridge/TypeBridge.cs
453:         var Index = InFullName.IndexOf(',');
455:         var TypeName = Index >= 0 ? InFullName[..Index].Trim() : InFullName;
457:         var AssemblyName = Index >= 0 ? InFullName[(Index + 1)..].Trim() : null;
459:         var Type = System.Type.GetType(InFullName, throwOnError: false, ignoreCase: false);
461:         if (Type == null)
462:         {
463:             var Assemblies = AssemblyLoader.CurrentContext?.Assemblies
464:                              ?? AppDomain.CurrentDomain.GetAssemblies();
466:             Type = GetTypeImplementation(Assemblies, AssemblyName, TypeName);
467:         }
469:         if (Type != null)
471:             StringToType.TryAdd(InFullName, Type);
...
477:     private static Type? GetTypeImplementation(IEnumerable<Assembly> InAssemblies, string? InAssemblyName,
478:         string InTypeName)
479:     {
480:         foreach (var Assembly in InAssemblies)
481:         {
482:             if (InAssemblyName != null)
483:             {
484:                 if (!string.Equals(Assembly.GetName().Name, InAssemblyName, StringComparison.OrdinalIgnoreCase))
485:                 {
486:                     continue;
487:                 }
488:             }
489: 
490:             var Type = Assembly.GetType(InTypeName, throwOnError: false, ignoreCase: false);
...
496:             }
497:         }
```

**问题**

1. **首个匹配即返回，顺序不确定**（`:480-496`）：当 `InFullName` 不含程序集名（`Index < 0` → `AssemblyName = null`），`:482-488` 的过滤被整体跳过，于是**遍历所有程序集**并返回第一个包含该类型名的程序集里的类型。若两个程序集都有 `MyNamespace.MyType`（例如用户程序集与 `Interop`，或旧/新版本），结果取决于 `Assemblies` 的枚举顺序 —— **不确定**。这是"热重载后行为诡异"的一类经典根因：旧 ALC 的程序集可能仍在 `AppDomain.CurrentDomain.GetAssemblies()`（`:464` 的 fallback）里，于是**旧类型被复用**，其静态字段/类型句柄指向已卸载代码。
2. **fallback 到 `AppDomain.CurrentDomain.GetAssemblies()`**（`:464`）在 LeanCLR 下尤其危险：`AssemblyLoader.CurrentContext` 恒为 null（F-CS1-001），因此 LeanCLR **总是**走这个 fallback → 拿到的是**默认域的全部程序集**，包括上一次热重载遗留的。配合 `StringToType` 缓存（永不清空，见 F-CS1-001/023）→ **旧类型被永久缓存**。
3. **负结果不缓存**（`:469-472` 只缓存成功）：对每个查不到的类型名，每次调用都要重新做 `Type.GetType`（内部会遍历）**加上**一遍全程序集线性扫描。反射初始化阶段若有大量"预期不存在"的类型探测，这是 O(N×M) 的重复工作。
4. **`IndexOf(',')` 解析脆弱**（`:453-457`）：如果 `InFullName` 是泛型参数个数带逗号的开放泛型名（例如 ``"System.Collections.Generic.Dictionary`2[[System.String, mscorlib],[System.Int32, mscorlib]]"``），第一个逗号出现在 `[[` 之后 → `TypeName`/`AssemblyName` 都被切错。虽然 `:459` 的 `System.Type.GetType(InFullName, ...)` 会先尝试且通常成功（泛型限定名是 `Type.GetType` 支持的形式），所以这条路径**只在 `GetType` 失败时**才踩到 —— 低概率但真实。

**建议**
1. **强制程序集限定**：让 C++ 侧总是传 `"{FullName}, {AssemblyName}"`（`GetFullName`（`:157`）已经在拼程序集名，说明设计上是这样），并对**不带程序集名**的调用记一条警告（而不是静默全扫描）。
2. **限定扫描范围**：优先只在 `AssemblyLoader.CurrentContext.Assemblies` 里找；`AppDomain` fallback 仅在前者不可用时启用，且**排除**已知的宿主程序集（`Interop`、`System.*`）。
3. **缓存负结果**：用 `Dictionary<string, Type?>` 显式存 `null`（需注意 `Dictionary` 允许 null 值）。
4. 解析改为使用 `Type.GetType` 的返回值为主，只有在确定没有 `[[...]]` 时才做逗号切分。

**验证方式**
- 在两个程序集里各定义一个同名类型（同命名空间），用**不带程序集名**的 `GetClass` 调用 → 现状结果依赖程序集枚举顺序。
- LeanCLR 后端连续热重载 2 次，打印 `GetClass("SomeUserType")` 得到的 `Type.Assembly` 的 ALC —— 若指向旧的 collectible ALC，即证实问题 2。
- grep C++ 侧传给 `GetClass` 的字符串是否总是带 `, AssemblyName`（看 `GetFullName` 的消费者）。

---

### [F-CS1-025] `TypeBridge` 的 22 个 `BoxXxx`/`UnboxXxx` 是完全同构的样板代码，可用泛型 + `Unsafe.As` 收敛；且 `Unbox` 失败静默返回 0

- **类别**: 可优化/可读性 / 死代码（个别方法可能无人调用）
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Script/Interop/Bridge/TypeBridge.cs:204-268`（11 个 Box）、`:270-444`（11 个 Unbox）
- **函数**: `Interop.TypeBridge.BoxBool..BoxDouble` / `UnboxBool..UnboxDouble`
- **置信度**: 高

**现状（代码事实）** —— 11 个 Box 方法是同一个模子，仅指针类型不同：

```csharp
// Script/Interop/Bridge/TypeBridge.cs
204:     [UnmanagedCallersOnly]
205:     public static unsafe nint BoxBool(int* InValue)
207:         return InValue != null ? HandleData.Alloc(*InValue != 0) : 0;
210:     [UnmanagedCallersOnly]
211:     public static unsafe nint BoxSByte(sbyte* InValue)
213:         return InValue != null ? HandleData.Alloc(*InValue) : 0;
...
264:     [UnmanagedCallersOnly]
265:     public static unsafe nint BoxDouble(double* InValue)
267:         return InValue != null ? HandleData.Alloc(*InValue) : 0;
```

11 个 Unbox 方法也是同一模式，仅 `is T` 的类型不同：

```csharp
270:     [UnmanagedCallersOnly]
271:     public static unsafe int UnboxBool(nint InHandle, int* OutValue)
273:         if (OutValue != null)
275:             if (HandleData.GetObject(InHandle) is bool Value)
277:                 *OutValue = Value ? 1 : 0;
279:                 return 1;
...
430:     [UnmanagedCallersOnly]
431:     public static unsafe int UnboxDouble(nint InHandle, double* OutValue)
435:             if (HandleData.GetObject(InHandle) is double Value)
437:                 *OutValue = Value;
439:                 return 1;
```

**问题**
1. **约 240 行纯样板**（占 `TypeBridge.cs` 505 行的 **48%**）。每加一种数值类型都要改 4 处（Box、Unbox、`IScriptTypes.h` 的 typedef、`Op` 宏、`FunctionMacro.h` 的名字宏），极易漏改（`char` 就没有 Box/Unbox，而 `MethodBridge.WritePrimitiveValue`（`:196-211`）**也没有 `char` 分支** → 含 `char` 的 by-ref 参数会被静默忽略，见下）。
2. **`Unbox` 失败静默返回 0**（`:283, 299, 315, 331, 347, 363, 379, 395, 411, 427, 443`）：`is bool` 之类的类型测试失败时（例如 C++ 把 `int` 句柄传给 `UnboxInt32` 但实际值是 `int64`），返回 0 **且不写 `*OutValue`** → 调用方读到**未初始化**的局部变量。`FClassReflection` 侧的模式是 `if (const auto X = ...; IManagedHandleIsValid(X))`（见 `FClassReflection.cpp:58,72,86,100`），但**标量 Unbox 的返回值语义不同**（是成功标志，不是句柄），如果 C++ 侧误按句柄语义判断，就会把 `0` 当成"无效句柄"而**跳过赋默认值**，或反之。需 C++ 侧核实（列未覆盖项）。
3. **`char` 缺失**（对照 `MethodBridge.WritePrimitiveValue`，`:177-212`）：`ReadPrimitiveValue`（`:157-175`）与 `WritePrimitiveValue`（`:194-212`）的 `switch` 都**没有 `char` 分支** → 若 C# 方法有 `ref char` 参数，`ReadPrimitiveValue` 的 `switch` 会**走到末尾没有匹配项**，由于该 `switch` 是**表达式形式且无 `_ =>` 默认分支**，运行时会抛 `SwitchExpressionException`（.NET 6+ 对无匹配的非穷尽 switch 表达式抛出此异常）。`MethodBridge.Invoke` 的 `try/catch`（`:125`）会吞掉它 → C++ 看到"返回 null"。**这是一个真实的 P2 功能缺陷**，已并入本条记录。

**建议**
1. 样板收敛（保持 `[UnmanagedCallersOnly]` 导出签名不变，内部转发到泛型实现）：

```csharp
private static nint BoxCore<T>(T Value) where T : unmanaged => HandleData.Alloc(Value);

[UnmanagedCallersOnly] public static unsafe nint BoxBool(int* InValue)
    => InValue != null ? BoxCore(*InValue != 0) : 0;
// 其余同理，每行一句

private static unsafe int UnboxCore<T>(nint InHandle, T* OutValue) where T : unmanaged
{
    if (OutValue == null || HandleData.GetObject(InHandle) is not T Value) return 0;
    *OutValue = Value; return 1;
}
```
   注意：**不能**把 `[UnmanagedCallersOnly]` 加在泛型方法上（CLR 禁止），所以外层仍需 22 个薄壳 —— 但可以把每个壳压到 1 行，总体积降到约 60 行。
2. 给 `ReadPrimitiveValue` / `WritePrimitiveValue` 补上 `char` 与 `_ =>` 默认分支（默认分支应抛带方法名与参数类型的明确异常，而不是让 switch 表达式抛泛化的 `SwitchExpressionException`）。
3. `Unbox` 失败时**显式写默认值**（`*OutValue = default;`），避免调用方读未初始化内存：

```csharp
if (OutValue == null) return 0;
if (HandleData.GetObject(InHandle) is not T Value) { *OutValue = default; return 0; }
```

**验证方式**
- 检查 `Installed` 二进制里 `TypeBridge` 的 IL 行数（`ildasm`/`ilspycmd -il`）对比重构前后。
- 单测：`UnboxInt32(handleOfLong, &i)` → 现状返回 0 且 `i` 未变；修复后 `i == 0`（确定值）。
- 单测：C# 方法 `void M(ref char c)` 经 `MethodBridge.Invoke` 调用 → 现状内层抛 `SwitchExpressionException` 被吞、C++ 得到 0；修复后正常。

### [F-CS1-026] `Weavers.csproj` **从未定义 `WITH_LEANCLR`** → LeanCLR 专属的 `__{Property}_Attrs` 发射逻辑被永久编译掉，而消费端 `Utils.cs` 的 `#if WITH_LEANCLR` 是**开着的** → 生产者/消费者不匹配 —— **已证伪：非缺陷，请从排期移除**

- **类别**: Bug / 平台兼容
- **严重度**: **撤销（非缺陷）** —— **撤销前提未验证，标"存疑"（勿当已结案）**
- **复核结论**: 无法验证（卡点：weaver 项目自身编译时 `WITH_LEANCLR` 从何而来）
- **可达性**: 潜伏（若生产端确实被编译掉、而消费端开着，则在 LeanCLR 下会变成活跃的真实缺陷；两种可能无法区分）
- **复核证据（正反两侧，均来自 grep）**:
  - **支持原发现（生产端确实缺符号）**：weaver 侧 4 处 `#if WITH_LEANCLR` 位于 `Script/Weavers/UnrealTypeWeaver.cs:7, 229, 354, 573`（`:354-356` 即 `EmitPropertyAttributesField` 的唯一调用点）；对**整个插件根**跑 grep（模式 `WITH_LEANCLR|DefineConstants`）= 37 命中，**`Script/Weavers/**` 与所有 `Script/*.csproj` 中没有任何定义点**；`glob Script/**/*.props` 只命中两个 `obj/*.nuget.g.props`（自动生成，无该符号）→ `Script/` 树下不存在给 weaver 注入该符号的 props。
  - **支持撤销（生产端可能其实为开）**：唯一注入 `WITH_LEANCLR;` 的地方是 `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:214`（`DefineConstants += TEXT("WITH_LEANCLR;")`），它改写的是 `Template/Shared.props:7` 的空占位 `<DefineConstants></DefineConstants>`（同文件 `:190-223` 的 `ReplaceDefineConstants`）。若 **weaver 项目也被纳入该生成解决方案 / 也 import 该 props**，则生产端为开、原发现不成立 → 与"证伪"结论一致。
  - **消费端确为开**：`Script/UE/CoreUObject/Utils.cs:15, 255, 293`（`#if WITH_LEANCLR`）与 `:230`（`#if !WITH_LEANCLR`），由生成解决方案注入符号。
- **级别变动**: 维持"撤销（非缺陷）"的标记，但**置信度降为"存疑"**；建议后续用"生成解决方案里是否含 Weavers.csproj / 该 csproj 是否 import Shared.props"一锤定音
- **函数**: `Weavers.UnrealTypeWeaver.EmitPropertyAttributesField(TypeDefinition, PropertyDefinition)`
- **置信度**: 高

**现状（代码事实）**

生产端（weaver）——三个 `#if WITH_LEANCLR` 门：

```csharp
// Script/Weavers/UnrealTypeWeaver.cs
  7: #if WITH_LEANCLR
  8: using System.Globalization;
  9: #endif
...
229: #if WITH_LEANCLR
230:         private static string GetRootNamespace(TypeReference Type)
...
242:         private void EmitPropertyAttributesField(TypeDefinition Type, PropertyDefinition Property)
...
297:             var attrsField = new FieldDefinition("__" + Property.Name + "_Attrs",
298:                 FieldAttributes.Private | FieldAttributes.Static | FieldAttributes.InitOnly,
299:                 ModuleDefinition.TypeSystem.String);
301:             Type.Fields.Add(attrsField);
...
327: #endif
...
354: #if WITH_LEANCLR
355:             EmitPropertyAttributesField(Type, Property);      // ProcessUClassProperty 内
356: #endif
...
573: #if WITH_LEANCLR
574:             EmitPropertyAttributesField(Type, Property);      // ProcessUStructProperty 内
575: #endif
```

`Weavers.csproj` 全文（**20 行，无任何 `DefineConstants`**）：

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <Import Project="../Shared.props" Condition="Exists('../Shared.props')" />
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>
  <ItemGroup>
    <Compile Remove="Properties\**" />
    <EmbeddedResource Remove="Properties\**" />
    <None Remove="Properties\**" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="FodyHelpers" Version="6.8.0" />
  </ItemGroup>
  <ItemGroup>
    <Reference Remove="Interop" />
  </ItemGroup>
</Project>
```

关键事实：**`Script/Shared.props` 这个文件不存在**（`Get-ChildItem Script -File` 返回空；`Test-Path` 为 false）。`Condition="Exists('../Shared.props')"` 使 import 静默失效。也就是说这个 csproj **既没有内联 `DefineConstants`，也没有任何会提供它的 props 文件**。

grep 证据（`WITH_LEANCLR` 全插件，排除 obj/bin/Intermediate/Binaries/ThirdParty）：

| 位置 | 性质 |
|---|---|
| `Script/Weavers/UnrealTypeWeaver.cs:7, 229, 354, 573` | **C# 端唯一**的 4 处使用（weaver 自身） |
| `Script/UE/CoreUObject/Utils.cs:15, 230, 255, 293` | **C# 端被织入的程序集**（`UENamePlaceholder.dll`） |
| `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:876, 910` | 生成代码里的字面量 `"#if WITH_LEANCLR\n"` |
| `Source/UnrealCSharpCore/UnrealCSharpCore.build.cs:301, 305` | **只有 C++** `PublicDefinitions.Add("WITH_LEANCLR=1" / "=0")` |
| `Script/Weavers/Weavers.csproj` | **0 命中** |

消费端（UE 程序集，`#if WITH_LEANCLR` 在这些构建配置下**是开着的**）：

```csharp
// Script/UE/CoreUObject/Utils.cs
15: #if WITH_LEANCLR
16:         private static Dictionary<string, Type> FullName2Type;
18:         private const string FieldPrefix = "__";
20:         private const string AttributeFieldSuffix = "_Attrs";
22:         private static readonly int FieldPrefixLength = FieldPrefix.Length;
24:         private static readonly int AttributeFieldPrefixLength = FieldPrefixLength + AttributeFieldSuffix.Length;
```

**问题**
`EmitPropertyAttributesField` 会为每个 `[UProperty]` 属性发射一个 `private static readonly string __{Property}_Attrs` 字段（`:297-301`）并在 `.cctor` 里赋值（`:322-325`），把属性上的自定义 attribute 以 `"FullName|ArgCount|arg0|arg1..."` 的形式**编码进程序集元数据**（`:283-295`），供 LeanCLR 下 `Utils` 读取（`Utils.cs:18-24` 的 `FieldPrefix`/`AttributeFieldSuffix` 正是对应这个命名约定）。

因为 `WITH_LEANCLR` 在 `Weavers.csproj` 里**从未定义**：
- `:355` 与 `:574` 两处调用被预处理掉 → **该字段永远不会被发射**；
- `:229-327` 整段 99 行（含 `GetRootNamespace`）是**不可达代码**（不是"死代码"意义上的未调用，而是**编译期恒不参与**）；
- `Script/Weavers/UnrealTypeWeaver.cs:8` 的 `using System.Globalization;` 也随之被裁掉 —— 这是一处**内部自洽的信号**：作者知道该文件会被以不带 `WITH_LEANCLR` 的方式编译，否则 `CultureInfo` 会报未定义。

**后果**
LeanCLR 后端下：
- `Utils` 的反射路径按 `__{Prop}_Attrs` 查找属性元数据 → **必然找不到** → 属性上的 `[UProperty(DisplayName=...)]` / `[EditAnywhere]` / `[BlueprintReadWrite]` 等**全部丢失**（表现为编辑器细节面板缺少元数据、蓝图不可见/不可编辑）。
- 由于**没有日志、没有断言**，症状会表现为"某些属性在编辑器里显示异常"，排查方向完全不会指向"weaver 没发射字段"。
- 更隐蔽的是：`Utils.cs` 是否走 `_Attrs` 路径可能由别的运行期条件决定，因此这个不匹配可能在"非 LeanCLR 配置"的测试中被完全掩盖，只在 LeanCLR 打包时暴露。

**修复方案（两选一，必须明确其一）**

方案 A（推荐，让生产者/消费者一致）——在 `Weavers.csproj` 里显式定义，且**与 C++ 的配置对齐**：

```xml
<PropertyGroup>
  <TargetFramework>netstandard2.0</TargetFramework>
  <LangVersion>latest</LangVersion>
</PropertyGroup>
<PropertyGroup Condition="'$(LeanCLR)' == 'true'">
  <DefineConstants>$(DefineConstants);WITH_LEANCLR</DefineConstants>
</PropertyGroup>
```
但 `netstandard2.0` 下的 `WITH_LEANCLR` 含义是"目标运行时是 LeanCLR"，而 weaver 是**编译期**工具、一次编译出的 `Weavers.dll` 可能被三种后端共用 —— 因此**更好的做法是方案 B**。

方案 B（推荐）——**把 `WITH_LEANCLR` 从编译期开关改为配置开关**，让 weaver 在**运行期**决定是否发射。Fody 已经现成支持这个：`FodyWeavers.xml` 的 XML 属性会作为 `Config` 传给 weaver。

```xml
<!-- Script/Weavers/FodyWeavers.xml -->
<Weavers xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="FodyWeavers.xsd">
	<UnrealTypeWeaver EmitPropertyAttrs="$(LeanCLR)" />
</Weavers>
```
```csharp
// UnrealTypeWeaver.cs
public override void Execute()
{
    var bEmitAttrs = bool.TryParse(Config.Attribute("EmitPropertyAttrs")?.Value, out var V) && V;
    ...
    if (bEmitAttrs) { _classTypes.ForEach(T => T.Properties.Where(HasUProperty).ToList().ForEach(P => EmitPropertyAttributesField(T, P))); }
}
```
（同时删掉三个 `#if WITH_LEANCLR`，让代码**始终可编译**，消除"代码在某个配置下从未被编译过"的整类风险。）

**验证方式**
- `Select-String -Path Script\Weavers\Weavers.csproj -Pattern 'DefineConstants|WITH_LEANCLR'` → 现状 0 命中，证明开关从未生效。
- 反编译织入后的程序集：`ilspycmd -il UENamePlaceholder.dll | Select-String '_Attrs'` → 现状 0 命中；期望在 LeanCLR 配置下命中 `__{Prop}_Attrs`。
- 直接用 Fody 的 `WeavingException` 机制：若运行期检测到 `Utils` 期望该字段而 weaver 未发射，应主动报错而不是静默。

---

### [F-CS1-027] `ProcessUClassProperty` / `ProcessUStructProperty` **删除属性的 backing field 但不清理引用它的 IL** → 带初始化器的 `[UProperty]` 自动属性会产生非法程序集

- **类别**: Bug / 未定义行为
- **严重度**: **P2**
- **复核结论**: 部分确认（类路径事实逐字成立；**结构体路径 `:560-571` 未覆盖** → 该半句维持原结论）
- **可达性**: 活跃（`[UProperty]` 自动属性即触发；**仅在属性带初始化器**时产生非法程序集）
- **复核证据**: `Script/Weavers/UnrealTypeWeaver.cs:341-352`：`:344` `var fieldName = $"<{Property.Name}>k__BackingField";`、`:346-347` `Type.Fields.FirstOrDefault(Field => Field.Name == fieldName && Field.FieldType.FullName == Property.PropertyType.FullName)`、**`:351` `Type.Fields.Remove(backingField);`** —— 只移除字段定义，未遍历 `Property.GetMethod/SetMethod` 的 IL 清理对 backing field 的 `ldfld/stfld` 引用；`:354-356` 紧随 `#if WITH_LEANCLR` + `EmitPropertyAttributesField`
- **级别变动**: 维持 P2
- **文件**: `Script/Weavers/UnrealTypeWeaver.cs:341-352`（类）、`:560-571`（结构体，未读该区间）
- **函数**: `Weavers.UnrealTypeWeaver.ProcessUClassProperty(TypeDefinition, PropertyDefinition)` / `ProcessUStructProperty(...)`
- **置信度**: 中—高（机制确定；最终症状取决于 Cecil 在写回时如何处理"已从类型移除但被 IL 引用"的 `FieldDefinition`）

**现状（代码事实）**

```csharp
// Script/Weavers/UnrealTypeWeaver.cs  —— 类版本
341:         private void ProcessUClassProperty(TypeDefinition Type, PropertyDefinition Property)
342:         {
343:             // 删除默认生成
344:             var fieldName = $"<{Property.Name}>k__BackingField";
345: 
346:             var backingField = Type.Fields.FirstOrDefault(Field =>
347:                 Field.Name == fieldName && Field.FieldType.FullName == Property.PropertyType.FullName);
348: 
349:             if (backingField != null)
350:             {
351:                 Type.Fields.Remove(backingField);
352:             }
...
// 结构体版本（完全同构，只是后面调用的是 Struct 版本的两个实现方法）
562:             // 删除默认生成
563:             var fieldName = $"<{Property.Name}>k__BackingField";
565:             var backingField = Type.Fields.FirstOrDefault(Field =>
566:                 Field.Name == fieldName && Field.FieldType.FullName == Property.PropertyType.FullName);
568:             if (backingField != null)
569:             {
570:                 Type.Fields.Remove(backingField);
571:             }
```

**全文件只有这两处 `.Remove(`**（grep 证据：`Select-String -Path Script\Weavers\UnrealTypeWeaver.cs -Pattern '\.Remove\('` → **恰好 2 命中，`:351` 与 `:570`**）。**没有任何一处**遍历方法体去删除/重写引用该字段的 `stfld` / `ldfld` 指令。

**问题的触发路径（非常现实）**
C# 自动属性的**初始化器**会被编译器发射到**构造函数**里，直接写 backing field：

```csharp
[UClass]
public partial class AMyActor : AActor
{
    [UProperty] public int Health { get; set; } = 100;      // ← 常见写法
    [UProperty] public string Name { get; set; } = "None";
}
```
编译后构造函数体等价于：
```
ldarg.0
call        AActor::.ctor()
ldarg.0
ldc.i4.s    100
stfld       int32 AMyActor::<Health>k__BackingField     // ← 只被这两处引用
ret
```
weaver 把 `<Health>k__BackingField` 从 `Type.Fields` 移除（`:351`），但**构造函数里的 `stfld` 仍然指向那个 `FieldDefinition` 对象**。属性访问器自身的 IL 被 `ilProcessor.Clear()`（`:384`/`:487`/`:603`/`:706`）整体替换，所以**访问器不再引用字段——构造函数仍然引用**。

**两种可能的恶劣后果**（取决于 Mono.Cecil 写回时的行为，两者都不可接受）：
1. **Cecil 在 `Write` 阶段抛异常**：`ArgumentException: Member 'AMyActor::<Health>k__BackingField' is declared in another module / not found` 之类 → **整个用户程序集编译失败**，错误信息指向 weaver，用户无从下手。
2. **Cecil 保留该字段原有的 metadata token 并把它写进 IL**：新程序集里该 token 位置已被**别的字段**占用（字段表被压缩重排）→ `stfld` 写到**错误的字段**上 → **静默的数据损坏**（例如把 `100` 写进了另一个 `[UProperty]` 的 backing field 或某个 `__{Prop}` 静态字段）。这类 bug 极难定位。

**为什么这是 P1 而不是 P0**：触发需要用户在 `[UProperty]` 属性上写初始化器。但这是**极其常见**的 C# 惯用法（`= 0`、`= "None"`、`= new()`），而且插件**没有任何诊断**提示"不要在 `[UProperty]` 属性上使用初始化器"。

**建议**

1. **扫除引用**（最小正确修复）：在移除字段前，遍历该类型**所有**方法的 IL，把引用该字段的 `ldfld/stfld/ldflda/ldsfld/stsfld` 指令**删除**（它们只出现在构造函数初始化器里，删除赋值是安全的，因为值将由 UE 的属性系统提供）：

```csharp
foreach (var method in Type.Methods.Where(M => M.HasBody))
{
    var il = method.Body.GetILProcessor();
    foreach (var ins in method.Body.Instructions.ToList())
    {
        if ((ins.OpCode == OpCodes.Stfld || ins.OpCode == OpCodes.Ldfld || ins.OpCode == OpCodes.Ldflda)
            && ins.Operand is FieldReference F && F.Name == fieldName)
        {
            il.Remove(ins);      // 仅初始化器会引用；如遇 ldfld 需成对移除前一条 ldarg.0
        }
    }
}
```
   注意 `stfld` 需要同时移除压栈的 `ldarg.0` 与值加载指令 —— 实现上更稳妥的做法是**直接用 Cecil 反编译式地处理构造函数**，或**保留该字段但改名/标记为不使用**（见方案 2）。

2. **更稳妥的替代方案**：**不要删除** backing field，而是把它**改成一个不会被 UE 属性系统看到的私有字段**（例如重命名为 `__Backing_{Property}` 并加 `[CompilerGenerated]` 已经是现状）。但这会与"属性值必须存在 UE 侧而非托管侧"的设计冲突 —— 需要权衡。

3. **至少要有诊断**：若检测到"属性有初始化器"（即构造函数 IL 引用了该 backing field 而访问器已被替换），**抛 `WeavingException`** 给出清晰指引。当前 `:106` 是唯一的 `WeavingException`，说明有抛异常的习惯，这里正是最需要它的地方。

**验证方式**
- 极简用例：`[UClass] partial class AMyActor : AActor { [UProperty] public int Health { get; set; } = 100; }`，编译后 `ilspycmd -il` 查看构造函数里是否还有 `stfld <Health>k__BackingField` 而字段已不存在。
- 运行期：实例化该类型 → 观察是 `TypeLoadException`/`InvalidProgramException` 还是静默错误值。
- grep 确认无扫除逻辑：`Select-String -Path Script\Weavers\UnrealTypeWeaver.cs -Pattern 'Stfld|Ldfld'` → 现状 **0 命中**（证明从不处理字段访问指令）。

---

### [F-CS1-028] `ModifyRpcMethod` 的 `sbyte BufferSize` 累加**会在参数总大小 ≥ 128 字节时溢出** → `Localloc` 得到巨大的无符号尺寸或过小缓冲 = 栈损坏

- **类别**: Bug / 未定义行为
- **严重度**: **P1**
- **复核结论**: 确认（算术链逐行复核成立）
- **可达性**: 活跃（任何 RPC/参数总尺寸 ≥ 128 字节的脚本方法即触发）
- **复核证据**: `Script/Weavers/UnrealTypeWeaver.cs:918`（`sbyte BufferSize = 0;`）、`:922`（`BufferSize += GetTypeSize(param.ParameterType);` —— **累加进 `sbyte`**）、`:933`（`Instruction.Create(OpCodes.Ldc_I4_S, BufferSize)` —— `Ldc_I4_S` 的操作数是 **int8**）、`:937`（`Localloc` 尺寸即来自该值）、`:941-954`（第二次累加同样进 `sbyte`，`:949` 亦用 `Ldc_I4_S` 做偏移）。参数总尺寸 ≥ 128 时 `sbyte` 溢出为负/回绕 → `Conv_U` 后变成巨大无符号尺寸或错误偏移 → 栈/`localloc` 尺寸错乱
- **级别变动**: 维持 P1
- **文件**: `Script/Weavers/UnrealTypeWeaver.cs:916-954`（累加点 `:922`、`:954`）、尺寸函数 `:866-903`
- **函数**: `Weavers.UnrealTypeWeaver.ModifyRpcMethod(TypeDefinition, MethodDefinition)`
- **置信度**: 高（算术自明）

**现状（代码事实）**

```csharp
// Script/Weavers/UnrealTypeWeaver.cs
905:         private void ModifyRpcMethod(TypeDefinition Type, MethodDefinition Method)
...
916:             if (Method.Parameters.Count > 0)
917:             {
918:                 sbyte BufferSize = 0;                      // ← sbyte！
919: 
920:                 foreach (var param in Method.Parameters)
921:                 {
922:                     BufferSize += GetTypeSize(param.ParameterType);   // ← 逐参数累加，无溢出检查
923:                 }
...
925:                 Method.Body.Variables.Add(new VariableDefinition(new PointerType(ModuleDefinition.TypeSystem.Byte)));
...
933:                 Method.Body.GetILProcessor().Append(Instruction.Create(OpCodes.Ldc_I4_S, BufferSize));  // ← ldc.i4.s <int8>
934:                 Method.Body.GetILProcessor().Append(Instruction.Create(OpCodes.Conv_U));
935:                 Method.Body.GetILProcessor().Append(Instruction.Create(OpCodes.Localloc));
...
941:                 BufferSize = 0;
942: 
943:                 foreach (var param in Method.Parameters)
944:                 {
945:                     Method.Body.GetILProcessor().Append(Instruction.Create(OpCodes.Ldloc_0));
946: 
947:                     if (BufferSize != 0)
948:                     {
949:                         Method.Body.GetILProcessor().Append(Instruction.Create(OpCodes.Ldc_I4_S, BufferSize));  // ← 同样受限于 int8
950: 
951:                         Method.Body.GetILProcessor().Append(Instruction.Create(OpCodes.Add));
952:                     }
953: 
954:                     BufferSize += GetTypeSize(param.ParameterType);   // ← 同样溢出
```

`GetTypeSize`（`:866-903`）返回类型也是 `sbyte`：

```csharp
866:         private static sbyte GetTypeSize(TypeReference Type)
...
870:                 case MetadataType.Boolean:
871:                 case MetadataType.Byte:
872:                     return sizeof(bool);          // sizeof(bool) == 1
...
877:                 case MetadataType.Int64:
878:                     return sizeof(long);          // 8
...
902:             return (sbyte)IntPtr.Size;            // 引用类型 = 8
903:         }
```

**问题**

1. **累加溢出（核心问题）**：`BufferSize` 是 `sbyte`（范围 −128..127），`BufferSize += X` 在默认的 **unchecked** 上下文里按 `(sbyte)(BufferSize + X)` **回绕**。当 RPC 方法的参数总字节数超过 **127** 时：
   - `Ldc_I4_S, BufferSize`（`:933`）发射的 `ldc.i4.s` 只接受 **int8** 立即数，此时 `BufferSize` 已是负数（例如 128 → −128）→ `Conv_U`（`:934`）把它变成 `0xFFFFFFFFFFFFFF80` → `Localloc` 试图在**求值栈上分配约 1.8×10¹⁹ 字节** → 立即 `StackOverflowException` 或进程崩溃。
   - 即使某个具体值回绕成小的正数，`Localloc` 得到的缓冲也会**小于实际需要**，而 `:943-994` 会按同一套（同样回绕的）偏移把参数逐个 `stind` 进去 → **写出 `localloc` 缓冲边界 → 栈内存损坏**（可能覆盖返回地址或调用者的局部变量）。
   
2. **偏移回绕（次级问题）**：`:949` 的 `Ldc_I4_S, BufferSize` 用于计算每个参数的写入偏移，同样是 int8 → 偏移错误 → 参数被写到**错误的位置** → native 侧读到错位的参数值（静默的数据错误，比崩溃更难查）。

3. **触发门槛很低**：`GetTypeSize` 对引用类型返回 8。一个带 16 个引用/`long`/`double` 参数的 RPC 就是 128 字节：
   ```csharp
   [UFunction, Server, Reliable]
   public void Server_Spawn(int A, int B, ... )   // 只要总字节 ≥ 128
   ```
   或者混合类型：`FVector`/`FRotator` 这类结构体参数（`GetTypeSize` 默认分支 → 8，见 F-CS1-029）会**低估**其真实大小，反而"帮助"避开 128 门槛，但同时引入另一个 bug。

**建议**

1. 把类型改为 `int`（`GetTypeSize` 返回 `int`），并用 `Ldc_I4` / `Ldc_I4_S` 按值选择指令：

```csharp
private static int GetTypeSize(TypeReference Type) { ... }

private static Instruction LdcI4(int Value) => Value switch
{
    >= -128 and <= 127 => Instruction.Create(OpCodes.Ldc_I4_S, (sbyte)Value),
    _                 => Instruction.Create(OpCodes.Ldc_I4, Value)
};
```
2. **加断言/编译期校验**：
```csharp
if (BufferSize > 0xFFFF)
    throw new WeavingException($"RPC {Type.FullName}.{Method.Name}: parameter payload too large ({BufferSize} bytes).");
```
3. 更根本的问题：**逐参数按类型大小拼一块裸缓冲**这个设计本身就很脆弱（对齐、结构体布局、`bool` 的 1 字节、引用类型的 handle 宽度都靠手写 `GetTypeSize` 维护）。建议改为让运行时库（`FFunction_GenericCall26Implementation`）接受一个**参数描述符数组**，由 C# 侧按 `Marshal.SizeOf` / 运行时类型信息计算布局，而不是在 weaver 里硬编码。

**验证方式**
- 写一个带 16 个 `long` 参数的 `[UFunction, Server]` RPC，编译后 `ilspycmd -il` 查看 `localloc` 前的 `ldc.i4.s` 立即数 → 现状应为负数（−128 或类似）。
- 运行该 RPC → 现状应崩溃或读到错位参数。
- 单测 `GetTypeSize` 的返回值类型：`Assert.IsType<int>(...)` 或直接看签名 `sbyte GetTypeSize` → 现状为 `sbyte`。

---

### [F-CS1-029] `IsCompound` 对**结构体（值类型）判定为 false**，而 `GetTypeSize`/`GetTypeStind` 的默认分支按**引用类型**（8 字节 / `stind.i`）处理 → 结构体属性与结构体参数的大小/存取方式错误

- **类别**: Bug / 未定义行为
- **严重度**: **P3**（若 `[UProperty]` 确实可标注在结构体类型属性上，则为 P1 → 置信度中）
- **复核结论**: ⚠️部分 —— 严重度由 P2 校正为 **P3**；可达性 潜伏
- **文件**: `Script/Weavers/UnrealTypeWeaver.cs:1065`（判定）、`:866-903`（尺寸）、`:786-824`（stind）、`:341-546` / `:560-765`（属性）、`:905-1020`（RPC 参数）
- **函数**: `Weavers.UnrealTypeWeaver.IsCompound(TypeReference)` / `GetTypeSize` / `GetTypeStind`
- **置信度**: 中（逻辑确定；"是否存在结构体类型的 `[UProperty]` 属性"未能在本次窗口内核实）

**现状（代码事实）**

```csharp
// Script/Weavers/UnrealTypeWeaver.cs
1065:         private static bool IsCompound(TypeReference Type) => !Type.Resolve().IsValueType;
```

`IsCompound` 对**类**（引用类型）返回 `true`，对**结构体/枚举/基元**返回 `false`。而两个"尺寸/存取宽度"函数的**默认分支**（即所有未被显式列举的非基元类型）返回的是**指针宽度**：

```csharp
// GetTypeStind 的结尾
823:             return Instruction.Create(OpCodes.Stind_I);      // 存 8 字节（native int）
// GetTypeLdind 的结尾
863:             return Instruction.Create(OpCodes.Ldind_Ref);    // 读一个对象引用
// GetTypeSize 的结尾
902:             return (sbyte)IntPtr.Size;                       // 8
```

**问题**

对照三个消费者：

1. **属性 setter**（`:359-453` / `:578-672`）：`bIsCompound == false` 时走 `:434`/`:653` 的 `ilProcessor.Append(GetTypeStind(Property.PropertyType))`。若属性类型是**结构体**（例如 `FVector`，24 字节）：
   - `IsCompound` → `false`（结构体是值类型）→ 走**值类型分支**；
   - `GetTypeStind` → 默认 → `Stind_I`（8 字节）；
   - `GetTypeSize` → 默认 → 8。
   - 而 `:400` 的 `Ldarg_1` 把**整个 24 字节结构体值**压栈，紧接着 `stind.i` 期望栈顶是 **native int** → **IL 栈类型不匹配**。对全信任程序集运行时不做严格校验，JIT 会把栈上的 8 字节当作 native int 存下去 → `FVector` 的 `X/Y/Z` 三份 double 里只落下一份（或落到错位的位置）→ **静默的数据损坏**；若 JIT 选择报错则是 `InvalidProgramException`。
   - 对比：正确的做法要么把结构体当"复合"（`bIsCompound == true`，走 `GetHandle` + `stind.i` 存**句柄**），要么按真实大小（24）用 `cpblk`/逐字段展开。

2. **属性 getter**（`:456-545` / `:675-764`）：`bIsCompound == false` 分支用 `GetTypeLdind` → 结构体走默认 → `Ldind_Ref`（读取一个**对象引用**）→ 然后 `Stloc_1`（局部变量类型是结构体）→ **栈类型不匹配**，同样损坏或抛异常。

3. **RPC 参数**（`:943-994`）：结构体参数被 `GetTypeSize` 记为 **8 字节**，却按值 `Ldarg_S` 压入整个结构体再 `Stind_I` 存 8 字节 → 缓冲区布局与 native 侧对结构体的预期（24 字节 `FVector`）**完全不一致** → native 侧读到错位/截断的参数。

**关键的不确定性（必须在别处核实）**
`IsCompound` 的语义看起来假设"非值类型 = 需要句柄"，而 `GetTypeSize`/`GetTypeStind`/`GetTypeLdind` 的默认分支假设"未列举类型 = 8 字节引用"。当"**值类型但大小 > 8**"（即结构体）出现时两套假设**冲突**。是否真的会触发，取决于插件是否允许（或生成器是否允许）`[UProperty]` 标注在结构体类型的属性上，以及 `FVector`/`FRotator` 在 C# 侧被生成为 **struct 还是 class**：
- 若生成器把 `FVector` 生成为 `class` → `IsCompound == true` → 走句柄路径 → **无此问题**（那么本条降为 P3 说明性）；
- 若生成为 `struct`（UE 侧就是 `struct`，且 `Script/UE/Reflection/Property/FStructProperty.cs` 的存在暗示结构体属性是被支持的）→ **本条成立**。

**建议**
1. 把"是否值类型"与"是否超过指针宽度"分成两个判断：

```csharp
private static bool bNeedsHandle(TypeReference Type) => !Type.Resolve().IsValueType;

private static bool bIsInlineBlittable(TypeReference Type)
{
    var R = Type.Resolve();
    return R.IsValueType && R.IsEnum == false && GetTypeSize(Type) <= 8;
}
```
   对"值类型且大小 > 8"的结构体，按真实大小分配缓冲并用 `cpblk`（`ldloc; ldarg; ldc.i4 size; cpblk`）搬运，而不是 `stind.i`。
2. 在 weaver 里**显式拒绝**它无法正确处理的组合，并抛出 `WeavingException`（当前只有 `:106` 一处抛异常，这类"我处理不了"的场景正应该抛）：
```csharp
if (Type.Resolve().IsValueType && GetTypeSize(Type) > IntPtr.Size)
    throw new WeavingException($"Unsupported struct-typed member ({Type.FullName}); size {GetTypeSize(Type)} > pointer size.");
```
3. `GetTypeSize` 的默认分支应改为**报错**而不是返回 `IntPtr.Size` —— 静默的 8 是这里所有不确定性中的根源。

**验证方式**
- 反编译 `FVector` 在 C# 侧的声明（`ilspycmd` 或直接看生成器输出）：确认是 `struct` 还是 `class`。
- 写一个 `[UProperty] public FVector Location { get; set; }`，编译并检查 weaver 输出的 IL 是 `stind.i` 还是按 24 字节搬运。
- 运行期设置该属性后从 native 侧读回，比较三个分量是否正确。

---

### [F-CS1-030] `GetAllMeta` 的 13 处类型/方法解析**全部没有 null 检查**：任何一个内部符号改名都会让所有用户的项目编译失败，且报错是裸 `NullReferenceException`

- **类别**: Bug / 可维护性
- **严重度**: **P2**
- **复核结论**: 确认（"13 处解析全部无 null 检查"成立）
- **可达性**: 活跃（weaver 每次织入都执行 `GetAllMeta`）
- **复核证据**: `Script/Weavers/UnrealTypeWeaver.cs:1099-1159` 完整复核：`:1110` `definition.GetType("Script.CoreUObject.PathNameAttribute")`、`:1112-1113` 与 `:1115-1116`（`definition.GetType(...).Methods.FirstOrDefault(...)`）、`:1118, 1120, 1122, 1124, 1126, 1128`（5 个 attribute 类型）、`:1130-1132`、`:1134-1136`、`:1138-1140`、`:1142-1144`（`FPropertyImplementation` 4 个实现方法）、`:1146-1148`、`:1156-1158`（`AssemblyResolver.Resolve(...)` 后接 `.MainModule?.GetType(...)`）—— 其中 `GetType` 返回 `null` 时下一跳 `.Methods` / `.FirstOrDefault` **立即 NRE**；`Resolve` 一路有 `?.`，但 `.Methods` 无 `?.`。共 **13 处**（与标题一致）；`:1103-1108` 的 `DefinitionPlaceholder` 分支说明模块可能来自占位符 → 符号改名/缺失会让**所有用户脚本项目**编译期报裸 `NullReferenceException`，且报错点远离根因
- **级别变动**: 维持 P2
- **文件**: `Script/Weavers/UnrealTypeWeaver.cs:1099-1159`
- **函数**: `Weavers.UnrealTypeWeaver.GetAllMeta()`
- **置信度**: 高

**现状（代码事实）**

```csharp
// Script/Weavers/UnrealTypeWeaver.cs
1099:         private void GetAllMeta()
1100:         {
1101:             var definition = ModuleDefinition;
1102: 
1103:             if (definition.Name != "DefinitionPlaceholder")
1104:             {
1105:                 var path = GetUEAssemblyPath(AssemblyFilePath);
1106: 
1107:                 definition = ModuleDefinition.ReadModule(path);       // 无 try/catch，文件不存在/损坏即抛
1108:             }
1109: 
1110:             _pathNameAttributeType = definition.GetType("Script.CoreUObject.PathNameAttribute");
1111: 
1112:             _structRegisterImplementation = definition.GetType("Script.Library.UStructImplementation").Methods
1113:                 .FirstOrDefault(Method => Method.Name == "UStruct_RegisterImplementation");
1114: 
1115:             _structUnRegisterImplementation = definition.GetType("Script.Library.UStructImplementation").Methods
1116:                 .FirstOrDefault(Method => Method.Name == "UStruct_UnRegisterImplementation");
1117: 
1118:             _ufunctionAttributeType = definition.GetType("Script.Dynamic.UFunctionAttribute");
1119: 
1119-1144:  ……（同构，共 13 处 GetType + FirstOrDefault）
1146:             _getObject = ModuleDefinition.AssemblyResolver.Resolve(ModuleDefinition.AssemblyReferences
1147:                     .FirstOrDefault(Assembly => Assembly.Name == "Interop")).MainModule?.GetType("Interop.HandleData")
1148:                 .Methods.FirstOrDefault(Method => Method.Name == "GetObject");
1149: 
1150:             _genericCall24Implementation = definition.GetType("Script.Library.FFunctionImplementation").Methods
1151:                 .FirstOrDefault(Method => Method.Name == "FFunction_GenericCall24Implementation");
1152: 
1153:             _genericCall26Implementation = definition.GetType("Script.Library.FFunctionImplementation").Methods
1154:                 .FirstOrDefault(Method => Method.Name == "FFunction_GenericCall26Implementation");
1155: 
1156:             _getHandle = ModuleDefinition.AssemblyResolver.Resolve(ModuleDefinition.AssemblyReferences
1157:                     .FirstOrDefault(Assembly => Assembly.Name == "Interop")).MainModule?.GetType("Interop.HandleData")
1158:                 .Methods.FirstOrDefault(Method => Method.Name == "GetHandle");
1159:         }
```

**问题**

1. **`definition.GetType(...)` 返回 `null` 时立刻 `.Methods`**（`:1112`、`:1115`、`:1130`、`:1134`、`:1138`、`:1142`、`:1150`、`:1153`）→ **`NullReferenceException`**。用户/维护者看到的是 "Object reference not set to an instance of an object" + Fody 的堆栈，**看不出是哪个类型/方法没找到**。
2. **`GetType(...)` 找到类型但方法名不匹配**时：`.FirstOrDefault(...)` 返回 `null` → 字段为 `null` → 延迟到 `ModuleDefinition.ImportReference(null)`（如 `:117`、`:132`、`:447`、`:511`、`:666`、`:730`、`:1015`）时抛 NRE 或 `ArgumentException`，**报错点距离真实原因很远**（在 `ProcessStructRegister` / `ProcessUClassProperty` 里，而不是 `GetAllMeta` 里）。
3. **`AssemblyReferences.FirstOrDefault(A => A.Name == "Interop")`**（`:1146`、`:1156`）：若织入的程序集**没有**引用 `Interop`（例如用户的 Game 程序集只在 LeanCLR 下引用 `Interop`，或某个精简配置），`FirstOrDefault` 返回 **`null`** → `Resolve(null)`。Cecil 的 `Resolve(AssemblyNameReference)` 对 null 的处理是**未定义/抛异常**，并且 `.MainModule?.` 只保护了 `MainModule` 为 null 的情况，**没有保护 `Resolve(...)` 返回 null** → `.MainModule` 直接 NRE。这是**唯一一处用了 `?.` 却没有先判 Resolve 结果**的地方，说明作者意识到了 null 风险但没有系统处理。
4. **`:1103` 的 `definition.Name != "DefinitionPlaceholder"`**：`definition.Name` 是**程序集文件名**（Cecil 里 `ModuleDefinition.Name` 是文件路径/名）。占位符替换由 C++ 的 `FSolutionGenerator` 完成（`Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp:300` 替换 `DEFINITION_PLACEHOLDER`，宏定义在 `Source/UnrealCSharpCore/Public/CoreMacro/Macro.h:63`）✓ 机制存在。但若替换**失败**，`ModuleDefinition.ReadModule(".../DefinitionPlaceholder")` → `FileNotFoundException`，同样是裸异常。
5. **硬编码内部私有方法名**（`UStruct_RegisterImplementation`、`FProperty_GetObjectPropertyImplementation`、`FFunction_GenericCall24Implementation` …）：weaver 与运行时库的**私有成员名**形成编译期强耦合。任何一次重构/改名都会**同时**打破所有用户项目的构建，且报错不可读。这类耦合应至少有集中的常量 + 启动自检。

**与 `:106` 的对照**：全文**只有一处** `throw new WeavingException`（`:106`，UStruct 缺无参构造），说明作者知道 `WeavingException` 是 Fody 上报问题的正确方式。`GetAllMeta` 这个"所有 meta 的入口"恰恰是最需要它、却一处都没用的地方。

**建议**

```csharp
private TypeDefinition RequireType(ModuleDefinition Module, string FullName)
{
    var T = Module.GetType(FullName);
    if (T is null)
        throw new WeavingException($"UnrealTypeWeaver: required type '{FullName}' not found in '{Module.Name}'. " +
                                   "The runtime library (UnrealCSharp Script) may be out of sync with the weaver.");
    return T;
}

private static MethodDefinition RequireMethod(TypeDefinition Type, string Name)
{
    var M = Type.Methods.FirstOrDefault(X => X.Name == Name);
    if (M is null)
        throw new WeavingException($"UnrealTypeWeaver: required method '{Type.FullName}.{Name}' not found.");
    return M;
}

private ModuleDefinition RequireModule(AssemblyNameReference Reference)
{
    if (Reference is null)
        throw new WeavingException("UnrealTypeWeaver: assembly reference 'Interop' not found. " +
                                   "Ensure the project references Interop.");
    var Module = ModuleDefinition.AssemblyResolver.Resolve(Reference)?.MainModule;
    if (Module is null)
        throw new WeavingException($"UnrealTypeWeaver: could not resolve assembly '{Reference.FullName}'.");
    return Module;
}
```
并把 13 处调用点改写为 `RequireType(...)` / `RequireMethod(...)`。同时把 `:1103` 的路径解析失败也转成 `WeavingException`（见 F-CS1-031）。

**验证方式**
- 临时把 `Script/Library/FPropertyImplementation.cs` 里的方法 `FProperty_GetObjectPropertyImplementation` 改名，重新编译用户项目 → 现状报 NRE（无信息量）；修复后报 "required method ... not found"。
- `Select-String -Path Script\Weavers\UnrealTypeWeaver.cs -Pattern 'WeavingException'` → 现状 1 命中（`:106`），修复后应显著增加。

---

### [F-CS1-031] `GetUEAssemblyPath` 在占位符缺失时做**无校验的 `Substring`**：抛 `ArgumentOutOfRangeException` 或静默拼出错误路径

- **类别**: Bug / 未定义行为
- **严重度**: **P3**
- **复核结论**: ⚠️部分 —— 严重度由 P2 校正为 **P3**；可达性 潜伏
- **文件**: `Script/Weavers/UnrealTypeWeaver.cs:1161-1183`（关键行 `:1167-1174`）
- **函数**: `Weavers.UnrealTypeWeaver.GetUEAssemblyPath(string)`
- **置信度**: 高（算术可逐步验证）

**现状（代码事实）**

```csharp
// Script/Weavers/UnrealTypeWeaver.cs
1161:         private string GetUEAssemblyPath(string assemblyFilePath)
1162:         {
1163:             var scriptPathName = "ScriptPathPlaceholder";
1164: 
1165:             var ueAssemblyName = "AssemblyPlaceholder";
1166: 
1167:             var scriptIndex = assemblyFilePath.LastIndexOf(scriptPathName, StringComparison.Ordinal);
1168: 
1169:             var scriptPathNameLength = scriptPathName.Length;          // == 19
1170: 
1171:             var relativePath = assemblyFilePath.Substring(scriptIndex + scriptPathNameLength + 1,
1172:                 assemblyFilePath.Length - scriptIndex - scriptPathNameLength - 1);
1173: 
1174:             var basePath = assemblyFilePath.Substring(0, scriptIndex + scriptPathNameLength);
1175: 
1176:             var segments = relativePath.Split(new[] { '\\', '/' });
1177: 
1178:             segments[0] = ueAssemblyName;
1179: 
1180:             segments[segments.Length - 1] = ueAssemblyName + ".dll";
1181: 
1182:             return Path.Combine(basePath, Path.Combine(segments));
1183:         }
```

**问题**

1. **`scriptIndex == -1`（占位符未被替换）**：
   - `:1171` → `Substring(-1 + 19 + 1, len - (-1) - 19 - 1)` = `Substring(19, len - 19)`。因为 `19 + (len - 19) = len`，其实**恰好合法**（`startIndex + length == len`）。所以这里**不抛**。
   - `:1174` → `Substring(0, -1 + 19)` = `Substring(0, 18)` → **静默取路径的前 18 个字符**当作 `basePath`。
   - 结果：返回一个**完全无意义的路径**（把真实路径的前 18 个字符 + 被替换掉首段的相对路径拼起来）→ `ModuleDefinition.ReadModule(path)`（`:1107`）抛 `FileNotFoundException`，或更糟——**若那个诡异路径恰好存在**，就读取了错误的程序集。
   - 结论：**静默产生错误路径**，不是明确报错。这是"占位符替换失败"这类构建配置问题的典型表现：报错信息完全指不到根因。
   （我最初以为会抛 `ArgumentOutOfRangeException`，逐步代入后确认**不会**；此处更正为"静默错误路径"，置信度高。）
2. **`segments[0]` 与 `segments[^1]` 重叠**（`:1178`、`:1180`）：若 `relativePath` 只含一个段（即 `ScriptPathPlaceholder\X.dll` 之后没有子目录），`segments.Length == 1` → 两次赋值同一个元素 → 最终 `segments[0] = ueAssemblyName + ".dll"`，**把模块名覆盖成文件名**，意图（第一段=程序集名、最后一段=程序集名.dll）被破坏。需要 `segments.Length >= 2` 的前提，但**没有断言**。
3. `:1176` 的 `Split` 同时处理 `\\` 与 `/` ✓ 跨平台友好；`Path.Combine(segments)` 会用平台分隔符重新拼接 ✓。这一点是正确的（值得肯定）。
4. 整个方法**没有任何 try/catch 或校验**，也没有日志。

**建议**

```csharp
private string GetUEAssemblyPath(string assemblyFilePath)
{
    const string ScriptPathName = "ScriptPathPlaceholder";
    const string UEAssemblyName = "AssemblyPlaceholder";

    var ScriptIndex = assemblyFilePath.LastIndexOf(ScriptPathName, StringComparison.Ordinal);
    if (ScriptIndex < 0)
        throw new WeavingException(
            $"UnrealTypeWeaver: '{ScriptPathName}' not found in '{assemblyFilePath}'. " +
            "The project file placeholder was not replaced; regenerate the C# solution.");

    var BasePath = assemblyFilePath.Substring(0, ScriptIndex + ScriptPathName.Length + 1);
    var RelativePath = assemblyFilePath.Substring(ScriptIndex + ScriptPathName.Length + 1);
    var Segments = RelativePath.Split(new[] { '\\', '/' }, StringSplitOptions.RemoveEmptyEntries);

    if (Segments.Length < 2)
        throw new WeavingException(
            $"UnrealTypeWeaver: unexpected assembly path relative segment '{RelativePath}'.");

    Segments[0] = UEAssemblyName;
    Segments[^1] = UEAssemblyName + ".dll";

    var Result = Path.Combine(BasePath, Path.Combine(Segments));

    if (!File.Exists(Result))     // 加一条落地校验，把错误挡在 ReadModule 之前
        throw new WeavingException($"UnrealTypeWeaver: UE definition assembly not found: {Result}");

    return Result;
}
```

**验证方式**
- 构造用例：把 `AssemblyFilePath` 设为不含 `ScriptPathPlaceholder` 的字符串 → 现状返回一个含真实路径前 18 字符的畸形路径（可用单元测试断言）；修复后抛 `WeavingException`。
- 构造用例：相对路径只有一个段 → 现状 `segments[0]` 被覆盖成 `AssemblyPlaceholder.dll`；修复后抛异常。
- 关联：确认 `FSolutionGenerator.cpp:426` 的 `SCRIPT_PATH_PLACEHOLDER` 替换是否总是成功（这是同一个契约的两端）。

---

### [F-CS1-032] `ProcessUClassProperty`/`ProcessUStructProperty` 依赖"新加的局部变量正好是 0 号"，并对**已有访问器局部变量的自定义 `[UProperty]` 属性**会发射 `stloc.0`/`stloc.1` 到错误类型的槽位

- **类别**: 未定义行为 / 可维护性
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Script/Weavers/UnrealTypeWeaver.cs:377-396, 474-499, 596-615, 693-718`
- **函数**: `ProcessUClassProperty` / `ProcessUStructProperty`
- **置信度**: 中

**现状（代码事实）**

```csharp
// Script/Weavers/UnrealTypeWeaver.cs  —— 类 setter
377:                 Property.SetMethod.Body.Variables.Add(
378:                     new VariableDefinition(new PointerType(ModuleDefinition.TypeSystem.Byte)));
380:                 Property.SetMethod.Body.InitLocals = true;
384:                 ilProcessor.Clear();
...
396:                 ilProcessor.Append(Instruction.Create(OpCodes.Stloc_0));   // ← 硬编码 0 号槽
398:                 ilProcessor.Append(Instruction.Create(OpCodes.Ldloc_0));
...
// 类 getter
474:                 Property.GetMethod.Body.Variables.Add(
475:                     new VariableDefinition(new PointerType(ModuleDefinition.TypeSystem.Byte)));
477:                 if (!bIsCompound)
479:                     Property.GetMethod.Body.Variables.Add(
480:                         new VariableDefinition(ModuleDefinition.ImportReference(Property.PropertyType)));
...
499:                 ilProcessor.Append(Instruction.Create(OpCodes.Stloc_0));
...
533:                     ilProcessor.Append(Instruction.Create(OpCodes.Stloc_1));   // ← 硬编码 1 号槽
535:                     var i1 = Instruction.Create(OpCodes.Ldloc_1);
```

**问题**

1. **`Variables.Add` 之后就用 `Stloc_0`/`Ldloc_0`/`Stloc_1`/`Ldloc_1`**：这假设新加的变量位于索引 0（getter 里索引 1 也在**同一轮 Add 之后**立刻使用）。对**自动属性**（编译器生成的访问器**没有局部变量**）这个假设成立 ✓。但对**手写访问器**（`[UProperty] public int X { get => ...; set { ... } }`）：
   - 原访问器体里已有的局部变量**不会被清除**（`ilProcessor.Clear()` 只清**指令**，`Body.Variables` 里用户声明的局部变量仍在，见 `ModifyRpcMethod` 里作者明确写了 `Method.Body.Variables.Clear()`（`:914`）——**说明作者知道这两件事是分开的**，但属性路径上**没有**调用 `Variables.Clear()`）；
   - 于是新加的 `byte*` 变量落在**索引 N**（N>0），而 `Stloc_0` 写的仍是**用户原变量 0 号槽** → **栈类型不匹配**（往 `int`/`string` 局部写 `byte*`）→ `InvalidProgramException`，或把指针写进错误槽位后按原类型读回 → **内存损坏**。
   - getter 更严重：`Stloc_1`（`:533`）假设 1 号槽是"属性类型的局部变量"，若用户 getter 已有 ≥2 个局部变量，1 号槽是**用户的**变量 → 同上。
   
2. **对比证据（体现不一致）**：`ModifyRpcMethod`（`:912-914`）做了 `Clear()` **+** `Variables.Clear()`，而四个属性重写点（`:377`、`:474`、`:596`、`:693`）**只做了 `Clear()`，没有 `Variables.Clear()`**。同一份代码里同一件事有两种做法 → 至少有一处是错的。

3. **`Add` 的位置也在 `Clear()` 之前**（`:377` 先 Add、`:384` 才 Clear），顺序上没问题（`Clear` 不清变量），但也说明这段代码对 Cecil 的"指令 vs 变量"语义依赖很细。

**建议**
1. 统一为 `Body.Variables.Clear()` + `Body.ExceptionHandlers.Clear()` + `ilProcessor.Clear()`，然后用 **`Ldc_I4` + `Stloc` 显式索引**（或像 `ModifyRpcMethod` 那样重建变量表），彻底摆脱"我加的就是 0 号"的隐式假设：

```csharp
SetMethod.Body.Variables.Clear();
SetMethod.Body.ExceptionHandlers.Clear();
var BufferLocal = new VariableDefinition(new PointerType(ModuleDefinition.TypeSystem.Byte));
SetMethod.Body.Variables.Add(BufferLocal);
SetMethod.Body.InitLocals = true;
// …用 ilProcessor.Append(Instruction.Create(OpCodes.Stloc, BufferLocal)) 而非 Stloc_0
```
2. 或者在 `ProcessUClassProperty` 开头**拒绝**手写访问器（`Property.GetMethod?.Body.Instructions.Count > 0` 说明不是编译器生成），抛 `WeavingException` 给出清晰指引。这是最省事且最安全的：
```csharp
// 自动属性的访问器只有 ldarg.0 / ldfld / ret 或 stfld 这类极短序列
```
3. 加断言：`Debug.Assert(Property.SetMethod.Body.Variables.Count == 0, "手写 setter 不受支持");`

**验证方式**
- 写 `[UProperty] public int X { get { var t = 1; return t; } set { var u = value; } }`，编译并检查 weaver 输出 → 现状 `stloc.0` 指向用户的 `t`/`u` 槽（类型是 `int` 而非 `byte*`）→ `InvalidProgramException`。
- 反编译织入后的程序集确认 `stloc.0` 的目标类型。
- `Select-String -Path Script\Weavers\UnrealTypeWeaver.cs -Pattern 'Variables.Clear'` → 现状只有 `:914` 一处（RPC 路径）。

---

### [F-CS1-033] weaver 的**幂等性**问题：RPC 路径会**重复添加字段与方法**，结构体路径按**固定指令索引**插入 → 二次织入产生非法元数据

- **类别**: Bug / 未定义行为
- **严重度**: **P3**
- **复核结论**: ⚠️部分 —— 严重度由 P2 校正为 **P3**；可达性 潜伏
- **文件**: `Script/Weavers/UnrealTypeWeaver.cs:905-910, 767-784, 92-185`
- **函数**: `ModifyRpcMethod` / `ProcessRpcMethods` / `ProcessStructRegister`
- **置信度**: 中—高（代码事实高；"是否真的会二次织入"取决于构建管线 → 中）

**现状（代码事实）**

**（a）RPC 路径完全不幂等**：

```csharp
// Script/Weavers/UnrealTypeWeaver.cs
767:         private void ProcessRpcMethods(TypeDefinition Type)
...
778:             foreach (var method in rpcMethods)
779:             {
780:                 Type.Methods.Add(CreateRpcMethodImplementation(method));   // ← 无条件 Add，无"已存在则跳过"
781: 
782:                 ModifyRpcMethod(Type, method);
783:             }
...
905:         private void ModifyRpcMethod(TypeDefinition Type, MethodDefinition Method)
906:         {
907:             var hashField = new FieldDefinition("__" + Method.Name, FieldAttributes.Private | FieldAttributes.Static,
908:                 ModuleDefinition.TypeSystem.IntPtr);
909: 
910:             Type.Fields.Add(hashField);                                        // ← 无条件 Add，无查重
```

对比**属性路径是有查重的**（`:363-373`、`:458-468`、`:582-592`、`:677-687` 都用 `FirstOrDefault(Field => Field.Name == "__" + Property.Name && ...)` 后 `if (field == null)` 再 Add）—— 说明作者在属性路径上考虑过幂等，**RPC 路径漏了**。

**（b）结构体路径按固定指令索引插桩**：

```csharp
// Script/Weavers/UnrealTypeWeaver.cs
100:             var constructors = Type.GetConstructors();
102:             var constructor = constructors.FirstOrDefault(Method => !Method.Parameters.Any());
...
111:             ilProcessor.InsertAfter(constructor.Body.Instructions[1], Instruction.Create(OpCodes.Ldarg_0));
113:             ilProcessor.InsertAfter(constructor.Body.Instructions[2],
114:                 Instruction.Create(OpCodes.Ldstr, GetPathName(Type)));
116:             ilProcessor.InsertAfter(constructor.Body.Instructions[3],
117:                 Instruction.Create(OpCodes.Call, ModuleDefinition.ImportReference(_structRegisterImplementation)));
...
119:             if (Type.Methods.Any(Method => Method.Name == "Finalize") == false)   // ← 有查重
...
173:                 ilProcessor.InsertBefore(finalize.Body.Instructions[0], Instruction.Create(OpCodes.Ldarg_0));
```

`Finalize` 有查重（`:119`），但**构造函数插桩没有**：`:111/:113/:116` 依赖 `Instructions[1]`/`[2]`/`[3]` 是构造函数的**原始**指令（典型形态：`[0]=ldarg.0, [1]=call base.ctor, [2]=nop, [3]=ret`）。第二次织入时 `Instructions[1..3]` 已经是上次插入的指令 → 会插到**错误的位置**（例如把 `ldstr` 插到 `call base.ctor` 之前，栈上顺序错乱）→ **IL 栈不平衡 → `InvalidProgramException`**。

**（c）幂等的部分（值得肯定）**：`AddPathNameAttributeToUnrealType` 用 `Type.CustomAttributes.Any(...) == false` 做了查重（`:189-190`、`:208-209`）✓；`EmitPropertyAttributesField` 会 `Type.Fields.Add(attrsField)`（`:301`）**无查重** ✗（但该方法当前恒被编译掉，见 F-CS1-026）；`CreateRpcMethodImplementation` 里的 `.cctor`（`:303-316`）有查重 ✓。

**问题**
Fody 的正常流程是"编译 → 织入一次 → 写回"，因此**在标准构建里不会二次织入**。但二次织入会在以下场景发生：
- **增量构建**：若织入后的 DLL 被当作输入再次织入（例如输出目录被配置成同时是输入目录、或复制了已织入的 DLL 再编译）；
- **手动/脚本化重织**（CI 里对已发布的 DLL 再跑一次 Fody）；
- **同一 `Weavers.dll` 被两个 Fody 目标同时应用**（`Script/Weavers/FodyWeavers.xml` 与用户工程的 `FodyWeavers.xml` 同时生效）。

一旦发生，后果是**构建产物非法**：RPC 方法出现**重名字段**（Cecil 允许同名重复字段，但 CLR 元数据校验会拒绝）与**重复的 `{Name}_Implementation` 方法**，属性/结构体构造函数 IL 错位。这类故障的报错（"Duplicate field" / `InvalidProgramException`）完全指不到"weaver 跑了两次"。

**建议**
1. RPC 路径加查重，与属性路径保持一致：
```csharp
var hashFieldName = "__" + Method.Name;
var hashField = Type.Fields.FirstOrDefault(F => F.Name == hashFieldName && F.FieldType.FullName == ModuleDefinition.TypeSystem.IntPtr.FullName)
                ?? new FieldDefinition(hashFieldName, FieldAttributes.Private | FieldAttributes.Static, ModuleDefinition.TypeSystem.IntPtr);
if (!Type.Fields.Contains(hashField)) Type.Fields.Add(hashField);
```
2. `ProcessRpcMethods` 加"已处理"标记（最干净的做法是加一个自定义 attribute 作为哨兵）：
```csharp
private const string WeavedMarker = "Script.CoreUObject.WeavedAttribute";
if (Type.CustomAttributes.Any(A => A.AttributeType.FullName == WeavedMarker)) return;   // 已织入
// …末尾：Type.CustomAttributes.Add(new CustomAttribute(markerCtor));
```
   在 `Execute()` 最开头对**整个模块**做一次"已织入"检查则更稳：
```csharp
if (ModuleDefinition.Assembly.CustomAttributes.Any(A => A.AttributeType.FullName == WeavedMarker))
{ LogInfo("UnrealTypeWeaver: assembly already weaved, skipping."); return; }
```
3. 构造函数插桩改为**不依赖索引**：先在 ctor 体里定位 `call base..ctor` 的那条指令，再 `InsertAfter` 它：
```csharp
var baseCtorCall = constructor.Body.Instructions.FirstOrDefault(I => I.OpCode == OpCodes.Call && I.Operand is MethodReference M && M.Name == ".ctor");
if (baseCtorCall is null) throw new WeavingException($"...{Type.FullName}: cannot locate base ctor call");
// 依次 InsertAfter 到 baseCtorCall，并在每次插入后更新锚点为刚插入的最后一条
```

**验证方式**
- 对同一个程序集连续跑两次 Fody（`dotnet fody` 或 `Fody` MSBuild target 加一次手动调用），对比第一次与第二次的输出：现状第二次会新增 `__{Rpc}` 重复字段与 `{Rpc}_Implementation` 重复方法。
- `ilspycmd -il` + `Select-String` 统计 `__` 前缀字段的出现次数（>1 即重复）。
- `Select-String -Path Script\Weavers\UnrealTypeWeaver.cs -Pattern 'Fields.Add'` → 现状 4 处，其中 `:301`、`:910` 无查重。

---

### [F-CS1-034] `GetAllDynamic` 只扫描**顶层类型**（不含 `NestedTypes`），且部分属性匹配用**硬编码字符串字面量**、部分用解析出的 `TypeDefinition` → 同一属性族有两套匹配策略

- **类别**: Bug / 可维护性 / 死代码（静默跳过）
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Script/Weavers/UnrealTypeWeaver.cs:1067-1097`（嵌套类型）、`:1071-1095`（字面量匹配）、`:334, 553`（字面量匹配）、`:1110-1128`（解析匹配）
- **函数**: `Weavers.UnrealTypeWeaver.GetAllDynamic()` / `ProcessUClassType` / `ProcessUStructType`
- **置信度**: 高

**现状（代码事实）**

```csharp
// Script/Weavers/UnrealTypeWeaver.cs
1067:         private void GetAllDynamic()
1068:         {
1069:             foreach (var type in ModuleDefinition.Types)          // ← 仅顶层，不含 NestedTypes
1070:             {
1071:                 if (type.CustomAttributes.Any(Attribute =>
1072:                         Attribute.AttributeType.FullName == "Script.Dynamic.UClassAttribute"))
1073:                 {
1074:                     _classTypes.Add(type);
1075:                 }
1076:                 else if (type.CustomAttributes.Any(Attribute =>
1077:                              Attribute.AttributeType.FullName == "Script.Dynamic.UStructAttribute"))
...
1091:                 else if (type.IsInterface && type.Interfaces.ToList().Any(Interface =>
1092:                              Interface.InterfaceType.FullName == "Script.CoreUObject.IInterface"))
```

以及属性扫描处同样是**字面量**：

```csharp
333:                 if (property.CustomAttributes.Any(Attribute =>
334:                         Attribute.AttributeType.FullName == "Script.Dynamic.UPropertyAttribute"))
...
552:                 if (property.CustomAttributes.Any(Attribute =>
553:                         Attribute.AttributeType.FullName == "Script.Dynamic.UPropertyAttribute"))
```

而 RPC 扫描处用的是**从程序集解析出的类型对象**：

```csharp
769:             var rpcMethods = Type.GetMethods()
770:                 .Where(Method => Method.CustomAttributes.Any(Attribute =>
771:                     Attribute.AttributeType.FullName == _ufunctionAttributeType.FullName))    // ← 解析来的
772:                 .Where(Method => Method.CustomAttributes.Any(Attribute =>
773:                     Attribute.AttributeType.FullName == _serverAttributeType.FullName ||       // ← 解析来的
774:                     Attribute.AttributeType.FullName == _clientAttributeType.FullName ||
775:                     Attribute.AttributeType.FullName == _netMulticastAttributeType.FullName))
```

`GetAllMeta`（`:1110-1128`）则把 `PathNameAttribute`、`UFunctionAttribute`、`Client/Server/NetMulticast`、`NotPlaceable`、`Override` **都**解析成了 `TypeDefinition` 字段。

**问题**

1. **嵌套类型被静默忽略**（`:1069`）：`ModuleDefinition.Types` **不包含** `TypeDefinition.NestedTypes`。因此
   ```csharp
   [UClass]
   public partial class AOuter : AActor
   {
       [UClass]
       public partial class AInner : AActor { [UProperty] public int X { get; set; } }   // ← 静默不被织入
   }
   ```
   会**完全没有** `[PathName]`（`:66-72` 只遍历 `_classTypes`）、**完全不处理** `[UProperty]`（`:331-338`）、**完全不处理** RPC（`:769-776` 只对 `_classTypes` 调用）、结构体也**完全不注册**（`:75` 只遍历 `_structTypes`）。
   若生成器（`UnrealTypeSourceGenerator`）对嵌套类型**也**生成 `U{Name}` partial 声明，则会出现"生成器认了、weaver 没认"的半织入状态 → 表现为部分元数据缺失，无任何诊断。
   `TypeDefinition.NestedTypes` 的遍历 + 递归是唯一正确的做法。

2. **两套匹配策略并存**：`Script.Dynamic.UClassAttribute` 等被当作**字符串字面量**（`:1072, 1077, 1082, 1087, 1092` 与 `:334, 553`），而 `UFunctionAttribute` 等被解析为 `TypeDefinition`（`:1118-1128`）。后果：
   - 两种方式的**失效模式不同**：字面量匹配在命名空间/程序集变化时**静默失效**；解析匹配在类型缺失时**抛 NRE**（见 F-CS1-030）。同一份代码对同一族的属性采取"一半静默、一半崩溃"的策略，难以推理。
   - 更要紧的是**跨程序集场景**：`Attribute.AttributeType.FullName` 是**被标注程序集里记录的引用名**。若用户程序集引用的是**另一个**程序集里的同名 `Script.Dynamic.UClassAttribute`（例如 UE 程序集与 Game 程序集各自带一份，或版本不一致），字面量匹配仍然成立，但 `_pathNameAttributeType`（从 UE 程序集解析）与它**不是同一个类型** → weaver 会把 `PathNameAttribute`（UE 程序集那份）加到用户类型上，而用户程序集引用的是另一份 → 运行期 `Attribute.GetCustomAttributes` 找不到 → **属性静默失效**。
   - 与 `UnrealTypeSourceGenerator` 的差异：该生成器用 `GetAttributeFromClass(Syntax, "UClass")`（`Script/SourceGenerator/UnrealTypeSourceGenerator.cs:684`）按**简单名**匹配（`attributeName == Name || attributeName == Name + "Attribute"`），**完全忽略命名空间** → 用户自建的 `MyOrg.UClassAttribute` 也会被生成器认成 UClass。**两个组件对"什么算 UClass"的判定标准不同**（weaver 用全名、生成器用简单名）→ 同名不同命名空间时会出现"生成器生成了 `U{Name}` 声明、weaver 不织入"的组合。

3. **静默跳过点汇总**（`return` 而无日志）：`:96`（`[UStruct]` 基类不是 `System.Object`）、`:168`（`Finalize` 存在但 `FirstOrDefault` 返回 null 的兜底）、`:292`（属性无 attribute 可发射）。加上 `GetAllDynamic` 的"没匹配上就什么也不做"，weaver 共有**至少 5 条静默路径**，而显式诊断只有 `:106` 一处。

**建议**
1. 递归遍历嵌套类型：
```csharp
private void GetAllDynamic()
{
    foreach (var type in EnumerateAllTypes(ModuleDefinition)) { ... }
}

private static IEnumerable<TypeDefinition> EnumerateAllTypes(ModuleDefinition Module)
{
    var Stack = new Stack<TypeDefinition>(Module.Types);
    while (Stack.Count > 0)
    {
        var T = Stack.Pop();
        yield return T;
        foreach (var N in T.NestedTypes) Stack.Push(N);
    }
}
```
   注意嵌套类型的 `GetPathName`（`:222-227` 的 `name.Substring(1)`）与 `GetRootNamespace`（`:230-240`）也需相应处理（嵌套类型的 `Type.Name` 只是内层名）。
2. **统一匹配策略**：全部改为"从程序集解析出 `TypeDefinition`，再比较 `.FullName`"，把 `:1072` 等 6 处字面量替换为字段引用；并新增 `_uclassAttributeType` / `_ustructAttributeType` / `_uenumAttributeType` / `_uinterfaceAttributeType` / `_upropertyAttributeType` 五个 `GetAllMeta` 字段（配合 F-CS1-030 的 `RequireType`）。
3. 与 SourceGenerator 对齐匹配语义（都用全名），或在两个组件里各自写清"我的匹配规则"。

**验证方式**
- 用例：定义一个嵌套 `[UClass]`，编译后反编译检查是否有 `[PathName]` 与 `__{Prop}` 字段 → 现状无。
- `Select-String -Path Script\Weavers\UnrealTypeWeaver.cs -Pattern 'NestedTypes'` → 现状 **0 命中**。
- 用例：定义 `MyOrg.UClassAttribute` 并标注一个类 → 观察生成器（生成 `U{Name}`）与 weaver（不织入）行为不一致。

---

### [F-CS1-035] `GetTypeLdind` 的 `Byte`/`SByte` 分支**写反了**（Byte→`ldind.i1`、SByte→`ldind.u1`）；`GetTypeSize` 用 `sizeof(bool)` 作为 `byte` 的尺寸

- **类别**: 可优化/可读性 / 未定义行为（当前被后续截断掩盖）
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Script/Interop/../Weavers/UnrealTypeWeaver.cs:826-864`（`:831-833`、`:840-841`）、`:866-872`
- **函数**: `Weavers.UnrealTypeWeaver.GetTypeLdind(TypeReference)` / `GetTypeSize(TypeReference)`
- **置信度**: 高（代码事实）；"是否可观测"→ 见下分析

**现状（代码事实）**

```csharp
// Script/Weavers/UnrealTypeWeaver.cs
826:         private static Instruction GetTypeLdind(TypeReference Type)
827:         {
828:             switch (Type.MetadataType)
829:             {
830:                 case MetadataType.Boolean:
831:                     return Instruction.Create(OpCodes.Ldind_U1);
832:                 case MetadataType.Byte:
833:                     return Instruction.Create(OpCodes.Ldind_I1);      // ← byte 应为 Ldind_U1
...
840:                 case MetadataType.SByte:
841:                     return Instruction.Create(OpCodes.Ldind_U1);      // ← sbyte 应为 Ldind_I1
842:                 case MetadataType.UInt16:
843:                     return Instruction.Create(OpCodes.Ldind_U2);
...
// 尺寸
870:                 case MetadataType.Boolean:
871:                 case MetadataType.Byte:
872:                     return sizeof(bool);                              // ← Byte 用 sizeof(bool)，语义混淆
```

对照**写入侧是正确的**：

```csharp
788:             switch (Type.MetadataType)
789:             {
790:                 case MetadataType.Boolean:
791:                     return Instruction.Create(OpCodes.Stind_I1);
792:                 case MetadataType.Byte:
793:                     return Instruction.Create(OpCodes.Stind_I1);      // ✓ 1 字节写入，对 byte/sbyte 都成立
...
800:                 case MetadataType.SByte:
801:                     return Instruction.Create(OpCodes.Stind_I1);      // ✓
```

**问题**

1. **`Byte`/`SByte` 的读指令互相交换**：`ldind.i1` 会**符号扩展**、`ldind.u1` 会**零扩展**。对 `byte` 用 `ldind.i1` 会把 `200`（0xC8）读成 `-56`；对 `sbyte` 用 `ldind.u1` 会把 `-1`（0xFF）读成 `255`。
2. **为什么当前"可能"不出错**：getter 的后续序列是 `GetTypeLdind(...)` → `Stloc_1`（`:533`/`:752`），而局部变量 1 的类型是 `Property.PropertyType`（`byte` 或 `sbyte`）。`stloc` 存入窄整型局部时会**截断**到目标宽度，`-56 → 0xC8 → 200` ✓、`255 → 0xFF → -1` ✓。也就是说**截断恰好把它救回来了**。
3. **但这只是巧合**：任何后续重构（例如把 `Stloc_1`/`Ldloc_1` 换成 `conv` 序列、或让 `GetTypeLdind` 的结果参与算术/比较）都会立刻暴露为**符号错误**。同时 `GetTypeSize` 的 `sizeof(bool)` 用在 `Byte` 上也是同类"能跑但语义错"的代码 —— 一旦有人给 `char`/`bool` 加上不同尺寸就会连锁出错。
4. **可读性差**：与 `MethodBridge.ReadPrimitiveValue`（`Script/Interop/Bridge/MethodBridge.cs:161-173`）对照可见同样的"每种基元一份手写映射"的模式，两处都要维护 —— 属于可收敛的重复。

**建议**

```csharp
case MetadataType.Boolean: return Instruction.Create(OpCodes.Ldind_U1);   // ECMA 上 bool 为 1 字节无符号
case MetadataType.Byte:    return Instruction.Create(OpCodes.Ldind_U1);   // 无符号
case MetadataType.SByte:   return Instruction.Create(OpCodes.Ldind_I1);   // 有符号
...
// GetTypeSize
case MetadataType.Boolean:
case MetadataType.Byte:
case MetadataType.SByte:
    return 1;                                                              // 显式 1，不用 sizeof(bool)
```
并考虑把 `GetTypeStind`/`GetTypeLdind`/`GetTypeSize` 三张表合并成**一张** `Dictionary<MetadataType, (OpCode St, OpCode Ld, int Size)>`，消除三处手工同步的不一致（这正是本条 bug 的成因）。

**验证方式**
- 单测每个分支的 `OpCode`（`Assert.Equal(OpCodes.Ldind_U1, GetTypeLdind(byteType).OpCode)`）—— 但 `GetTypeLdind` 是 `private static`，需要 `InternalsVisibleTo` 或改成 `internal`。
- 运行期：`[UProperty] public byte B { get; set; }` 赋 200 后读回 → 现状因截断而正确（记录为"靠巧合正确"）。
- `Select-String -Path Script\Weavers\UnrealTypeWeaver.cs -Pattern 'Ldind_'` 逐条核对。

---

### [F-CS1-036] `ProcessRpcMethods` 生成的 `{Name}_Implementation` 丢失泛型参数、参数名与除 `[Override]` 外的全部自定义属性，且属性位被强行改成 `Public`

- **类别**: Bug / 可维护性
- **严重度**: **P2**
- **复核结论**: ✅确认 —— 严重度 P2 维持；可达性 活跃
- **文件**: `Script/Weavers/UnrealTypeWeaver.cs:1022-1063`（关键行 `:1030-1038`）
- **函数**: `Weavers.UnrealTypeWeaver.CreateRpcMethodImplementation(MethodDefinition)`
- **置信度**: 中—高

**现状（代码事实）**

```csharp
// Script/Weavers/UnrealTypeWeaver.cs
1022:         private MethodDefinition CreateRpcMethodImplementation(MethodDefinition Method)
1023:         {
1024:             var constructor = _overrideAttributeType.GetConstructors().FirstOrDefault();
1025: 
1026:             var constructorRef = ModuleDefinition.ImportReference(constructor);
1027: 
1028:             var attribute = new CustomAttribute(constructorRef);
1029: 
1030:             var newMethod = new MethodDefinition(Method.Name + "_Implementation", MethodAttributes.Public,
1031:                 Method.ReturnType);
1032: 
1033:             newMethod.CustomAttributes.Add(attribute);          // 只有 [Override]
1034: 
1035:             foreach (var param in Method.Parameters)
1036:             {
1037:                 newMethod.Parameters.Add(new ParameterDefinition(param.ParameterType));   // ← 不带名字
1038:             }
1039: 
1040:             if (Method.HasBody)
1041:             {
1042:                 newMethod.Body.InitLocals = true;
1043: 
1044:                 foreach (var variableDefinition in Method.Body.Variables)
1045:                 {
1046:                     newMethod.Body.Variables.Add(new VariableDefinition(variableDefinition.VariableType));
1047:                 }
1048: 
1049:                 foreach (var exceptionHandler in Method.Body.ExceptionHandlers)
1050:                 {
1051:                     newMethod.Body.ExceptionHandlers.Add(exceptionHandler);     // ← 共享同一批 handler 对象
1052:                 }
1053: 
1054:                 foreach (var instruction in Method.Body.Instructions)
1055:                 {
1056:                     newMethod.Body.Instructions.Add(instruction);              // ← 共享同一批 Instruction 对象
1057:                 }
1058: 
1059:                 newMethod.Body.OptimizeMacros();
1060:             }
1061: 
1062:             return newMethod;
1063:         }
```

**问题**

1. **参数名全部丢失**（`:1037`）：`new ParameterDefinition(param.ParameterType)` 不带 name → `_Implementation` 的参数名为空。C++ 侧 `FClassReflection` 会读取参数名（`Source/UnrealCSharpCore/Private/Reflection/FClassReflection.cpp:359: auto ParamName = InManagedReader.ArrayGetString(InParams[8], MethodParamIndex + ParamIndex);`）→ **参数名变为空字符串**。若参数名被用于蓝图引脚名、日志或参数匹配，会静默退化。
   修复：`new ParameterDefinition(param.Name, param.ParameterType)`。
2. **自定义属性不复制**（`:1033` 只加 `[Override]`）：原 RPC 上的 `[UFunction]`、`[Server]`/`[Client]`/`[NetMulticast]`、`[Reliable]`、`[BlueprintCallable]`、`[Category]` 等**全部不复制**到 `_Implementation`。若 native 侧需要这些元数据（例如把 `_Implementation` 当作可反射的 UFUNCTION 找出来），就会出现"RPC 有实现体但找不到元数据"。
   修复（若确实需要）：`foreach (var a in Method.CustomAttributes) if (a.AttributeType.FullName != _overrideAttributeType.FullName) newMethod.CustomAttributes.Add(a);`
   ⚠️ 注意：**共享 `CustomAttribute` 对象**跨两个方法也是 Cecil 的反模式（同 `:1051`/`:1056`），应深拷贝。
3. **属性位被覆盖为 `Public`**（`:1030`）：原方法的 `Virtual`/`Static`/`SpecialName`/`Final`/`Abstract` 等全部丢弃。
   - 若原 RPC 是 `protected virtual void Server_X()`（UE 的 Server RPC 惯例是 `protected`），生成出来的 `_Implementation` 是 `public` 非虚 → 可见性放宽（安全影响小），但**丢失 `virtual`** 会破坏"派生类覆盖 RPC 实现"的语义。
   - 若原 RPC 是 `static`，新的 `_Implementation` 变成实例方法，而 `ModifyRpcMethod` 生成的 IL 用 `Ldarg_0`（`:999`）取 `this` → **对静态方法 `ldarg.0` 正是第一个参数**，语义错位。
   修复：复制除了 `Abstract` 之外的属性位，并按需去掉 `SpecialName`。
4. **泛型参数不复制**：`Method.GenericParameters` 未复制，但 `Method.ReturnType` 与 `param.ParameterType` 可能是**泛型参数引用**（`!0`/`!!0`）→ 新方法里这些引用在**新方法的作用域中没有对应的 `GenericParameter`** → **无效元数据 / `TypeLoadException`**。这在 RPC 泛型方法上必然失败（UE 的 UFUNCTION 不支持泛型，所以实际可能不会遇到；但 weaver 没有拒绝它 → 静默产生非法 IL）。
5. **`Instruction` 与 `ExceptionHandler` 对象跨方法共享**（`:1049-1057`）：这些对象从**原方法体**直接搬进 `newMethod`，之后 `ModifyRpcMethod` 会 `Clear()` 原方法体（`:912`）与 `Clear()` 其 handler 列表（`:1019`）。Cecil 的 `Instruction` 内部带 `Previous`/`Next`/`Offset`/`body` 关联状态，**一个 Instruction 属于一个 body** 是其隐含契约。这里虽然"恰好"可能工作（因为原体随后被整体丢弃），但属于**踩在未定义行为上**。正确做法是逐条**重建**指令（`Instruction.Create(ins.OpCode, ins.Operand)`），分支目标用「旧→新」映射表重定向：
   ```csharp
   var Map = new Dictionary<Instruction, Instruction>();
   foreach (var ins in Method.Body.Instructions) Map[ins] = Instruction.Create(ins.OpCode, ins.Operand);
   foreach (var (Old, New) in Map)
   {
       if (New.Operand is Instruction Target && Map.TryGetValue(Target, out var NewTarget)) New.Operand = NewTarget;
       else if (New.Operand is Instruction[] Targets) New.Operand = Targets.Select(T => Map[T]).ToArray();
       newMethod.Body.Instructions.Add(New);
   }
   ```
   （现实现依赖"原体随即被丢弃"，因此**这条与 F-CS1-033 的幂等性缺陷叠加时会真正出错**。）

**建议**：见逐点。优先级：第 1、3 点（参数名、属性位）应尽快修；第 5 点应改为重建指令。

**验证方式**
- 反编译织入后的程序集：检查 `{Rpc}_Implementation` 的参数名是否为空、是否丢失 `[Server]` 等属性、`IsVirtual`/`IsStatic` 是否为 false。
- `Select-String -Path Script\Weavers\UnrealTypeWeaver.cs -Pattern 'GenericParameters'` → 现状 **0 命中**（证明泛型参数从不复制）。
- 用例：泛型 RPC → 期望现状产生非法 IL 或运行时 `TypeLoadException`。

---

### [F-CS1-037] `EmitPropertyAttributesField` 的属性编码**丢失命名参数/字段参数**，且数组类型的构造参数被序列化成 `"System.Object[]"` 之类的无意义文本

- **类别**: Bug
- **严重度**: **P3**（仅 LeanCLR 路径 —— 但该路径当前恒不生效，见 F-CS1-026）
- **复核结论**: ⚠️部分 —— 严重度由 P2 校正为 **P3**；可达性 潜伏
- **文件**: `Script/Weavers/UnrealTypeWeaver.cs:242-326`（关键行 `:246-288`）
- **函数**: `Weavers.UnrealTypeWeaver.EmitPropertyAttributesField(TypeDefinition, PropertyDefinition)`
- **置信度**: 中（编码格式与 C++/`Utils` 消费端的契约未在本次窗口核实 → 中）

**现状（代码事实）**

```csharp
// Script/Weavers/UnrealTypeWeaver.cs
246:             foreach (var attr in Property.CustomAttributes)
247:             {
248:                 var attrType = attr.AttributeType;
249: 
250:                 if (GetRootNamespace(attrType) != "Script.Dynamic" ||
251:                     attrType.Name == "UPropertyAttribute")
252:                 {
253:                     continue;
254:                 }
255: 
256:                 var parts = new List<string>();
257: 
258:                 if (attr.HasConstructorArguments)
259:                 {
260:                     foreach (var arg in attr.ConstructorArguments)
261:                     {
262:                         var value = arg.Value;
263: 
264:                         string text;
265: 
266:                         if (value == null)                    { text = string.Empty; }
270:                         else if (value is IFormattable formattable) { text = formattable.ToString(null, CultureInfo.InvariantCulture); }
274:                         else                                  { text = value.ToString(); }
275: 
279:                         parts.Add(text);
280:                     }
281:                 }
282: 
283:                 parts.Insert(0, parts.Count.ToString(CultureInfo.InvariantCulture));
284: 
285:                 parts.Insert(0, attrType.FullName.Replace('/', '+'));
286: 
287:                 lines.Add(string.Join("|", parts));
288:             }
```

**问题**

1. **命名参数/字段参数完全丢失**（`:258-281` 只看 `ConstructorArguments`）：C# 属性允许 `[UProperty(DisplayName = "生命值", Category = "Stats")]` 这种写法，此时值在 `attr.Properties` / `attr.Fields` 里，**不会被编码**。→ LeanCLR 下这类元数据**静默丢失**。而 `Script/UE/Dynamic/Property/DisplayNameAttribute.cs` 等大量属性只有**属性/字段**形式的元数据（例如 `DisplayPriorityAttribute` 等 15 行的文件通常是 `public int Priority { get; }`）→ 影响面广。
   修复：把 `attr.Properties` 与 `attr.Fields` 一并序列化，并在编码里区分"构造参数/命名属性/命名字段"。
2. **数组参数序列化错误**（`:274`）：`CustomAttribute` 的数组类型构造参数，其 `arg.Value` 是 `CustomAttributeArgument[]`，而代码走 `value.ToString()` → 得到 **`"System.Object[]"`** 或 `"Script.Dynamic.FooAttribute[]"` 这类类型名字符串，**不是数据**。例如 `[AllowedClasses(typeof(A), typeof(B))]`、`[DisallowedClasses(...)]`、`[RequiredAssetDataTags(...)]` 等（`Script/UE/Dynamic/Property/AllowedClassesAttribute.cs:15` 等）都会退化成垃圾串。
   修复：对 `CustomAttributeArgument[]` 递归编码（`string.Join(",", arr.Select(Encode))`）。
3. **`meta` 风格（`[UProperty("DisplayName", "X")]`）的键值对是压平的**（`:283-287`）：编码成 `"Type|ArgCount|arg0|arg1"`，消费端必须**按位置**解释，无法表达"这是键、这是值"。`Script/UE/Dynamic/Property/DefaultValueAttribute.cs`（45 行）这类多参数属性可以工作，但一旦某个属性同时用构造参数与命名参数，位置协议就不够用。
4. **`argType` 用 `FullName.Replace('/', '+')`**（`:285`）：把嵌套类型分隔符 `/` 换成 `+`。若消费端用 `.` 或 `/` 匹配，就完全找不到——需与 `Utils.cs` 的读取逻辑逐一核对（**未覆盖项**）。这是一处**不做校验的隐式格式约定**。
5. **`lines` 的顺序**来自 `Property.CustomAttributes` 的枚举顺序 ✓ 稳定（元数据表顺序），但**多个属性同名参数的冲突**没有检测。

**建议**
1. 递归编码所有三种参数（构造参数、`attr.Properties`、`attr.Fields`）：
```csharp
static string Encode(CustomAttributeArgument Arg) => Arg.Value switch
{
    null => "",
    CustomAttributeArgument[] Arr => string.Join(",", Arr.Select(Encode)),
    TypeReference T => T.FullName.Replace('/', '+'),
    IFormattable F => F.ToString(null, CultureInfo.InvariantCulture),
    var V => V.ToString() ?? ""
};
```
2. 在 `Utils.cs`（消费端）与 weaver（生产端）之间建立**单一权威的格式常量/文档**，并在两侧各加一个往返自测（encoder/decoder 单元测试）。
3. 对"无法编码的参数类型"**抛出 `WeavingException`** 而不是写 `"System.Object[]"` —— 静默的错误编码比不编码更难查。

**验证方式**
- 反编译织入后的程序集，读取 `__{Prop}_Attrs` 字段的**字面量字符串**（`ilspycmd` 会显示 `.cctor` 里的 `ldstr`），核对内容。
- 用例：`[UProperty(DisplayName = "X")]`（命名参数）与 `[AllowedClasses(typeof(AActor))]`（数组参数）→ 现状前者不出现、后者出现 `"System.Object[]"`。
- 与 `Utils.cs` 的解析函数做一次 encode→decode 往返测试。

---

### [F-CS1-038] `Weavers/FodyWeavers.xml` 使用**非 Fody 标准节点形态**，且与 `Weavers.csproj` 的属性不相干：weaver 既不读 `Config` 也无任何配置键 → 配置面为零

- **类别**: 可优化/可读性 / 可维护性
- **严重度**: **P3**
- **复核结论**: ✅确认 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Script/Weavers/FodyWeavers.xml:1-3`；`Script/Weavers/Weavers.csproj`
- **函数**: —（配置）
- **置信度**: 高（**已核对结论：不存在"配置键不一致导致配置失效"的问题**，因为根本没有配置键）

**现状（代码事实）**

```xml
<!-- Script/Weavers/FodyWeavers.xml（全文 3 行） -->
<Weavers xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="FodyWeavers.xsd" >
	<UnrealTypeWeaver/>
</Weavers>
```

grep 证据：`Select-String -Path Script\Weavers\UnrealTypeWeaver.cs -Pattern 'Config'` → **0 命中**。也就是说 weaver **从不读取 `Config`**，因此"`FodyWeavers.xml` 的配置键与代码里读的键是否一致"这一检查项在本项目上**结论为：无配置键，不存在不一致**。这是一个**negative result**，明确记录以免重复审查。

同时 `FodyWeavers.xsd` 在该目录下**不存在**（`Get-ChildItem Script\Weavers -File` 只有 `FodyWeavers.xml`、`UnrealTypeWeaver.cs`、`Weavers.csproj`）—— `xsi:noNamespaceSchemaLocation="FodyWeavers.xsd"` 指向一个缺失的文件，编辑器无法做 schema 校验（纯 IDE 体验问题）。

**问题**
1. `Weavers.csproj` 是一个**类库项目**（含 `UnrealTypeWeaver.cs`，引用 `FodyHelpers`），而 `FodyWeavers.xml` 是**给被织入的项目**用的配置文件。两者放在同一目录会让"这个 FodyWeavers.xml 是配置 weaver 自己的，还是配置谁的"产生歧义。实际生效的是**用户工程**（`UENamePlaceholder` / 游戏工程）目录下的 `FodyWeavers.xml`（由 `FSolutionGenerator` 生成，见 `Source/ScriptCodeGenerator/Private/FSolutionGenerator.cpp` 的占位符替换），而这个目录里的 `FodyWeavers.xml` 更像是**模板/示例**。缺少注释说明。
2. `xsi:noNamespaceSchemaLocation="FodyWeavers.xsd"` 指向不存在的文件（同上）。
3. 结合 F-CS1-026：**这里正是引入配置键的最佳位置**（`<UnrealTypeWeaver EmitPropertyAttrs="true"/>`），把当前恒失效的 `#if WITH_LEANCLR` 编译期开关改为**运行期配置开关**。

**建议**
1. 加 XML 注释说明本文件的角色（模板 vs 生效文件）：
```xml
<!-- 本文件是 weaver 侧的模板；真正生效的是每个被织入工程目录下的 FodyWeavers.xml -->
<Weavers xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
	<!-- EmitPropertyAttrs: LeanCLR 需要 __{Property}_Attrs 元数据字段（见 UnrealTypeWeaver.EmitPropertyAttributesField） -->
	<UnrealTypeWeaver EmitPropertyAttrs="false" />
</Weavers>
```
2. 移除或补上 `FodyWeavers.xsd` 引用。
3. 在 `UnrealTypeWeaver.cs` 的 `Execute()` 里用 `Config` 读取并 `LogInfo` 回显实际配置，使"配置是否生效"可观测。

**验证方式**
- `Select-String -Path Script\Weavers\UnrealTypeWeaver.cs -Pattern 'Config'` → 现状 0；引入配置后应 >0。
- 检查用户工程目录下的 `FodyWeavers.xml`（由 `FSolutionGenerator` 生成）确认实际生效的那份包含 `<UnrealTypeWeaver/>`。

---

### [F-CS1-039] weaver 与 `UnrealTypeSourceGenerator` 的职责边界：互补但**属性匹配语义不一致**，且二者都不处理字段上的 `[UProperty]`

- **类别**: 可维护性 / 一致性
- **严重度**: **P3**
- **复核结论**: ⚠️部分 —— 严重度 P3 维持；可达性 活跃
- **文件**: `Script/Weavers/UnrealTypeWeaver.cs`（IL 层）；`Script/SourceGenerator/UnrealTypeSourceGenerator.cs:676-687, 398-404, 499-521`（语法层）
- **函数**: `GetAllDynamic` / `ProcessUClassProperty` vs `UnrealTypeSourceGenerator.Execute` / `GetAttributeFromClass`
- **置信度**: 高（两侧代码都已读）

**现状（代码事实/对照）**

| 维度 | `UnrealTypeSourceGenerator`（Roslyn 源生成器，967 行） | `UnrealTypeWeaver`（Fody IL 织入器，1185 行） |
|---|---|---|
| 作用层 | 语法树 → **新增 partial 声明** | IL → **改写既存成员** |
| 触发判定 | `GetAttributeFromClass(Syntax, "UClass")`，**按简单名匹配**（`UnrealTypeSourceGenerator.cs:684: attributeName == Name \|\| attributeName == Name + "Attribute"`） | `Attribute.AttributeType.FullName == "Script.Dynamic.UClassAttribute"`，**按全名匹配**（`UnrealTypeWeaver.cs:1072`） |
| 强制要求 | `partial`，否则报错（`:499-521`，诊断 `ErrorDynamicClassNotAPartialClass`） | 无此检查 |
| `[UProperty]` 处理 | **不处理**（grep 结果中无属性级别的 `UProperty` 生成逻辑） | 核心职责：删 backing field、改写 getter/setter |
| 结构体注册/注销 | 不处理 | `ProcessStructRegister` 注入 `UStruct_RegisterImplementation` + `Finalize` |
| RPC | 不处理 | `ProcessRpcMethods` 重写方法体 + 生成 `_Implementation` |
| 库桥接 | `LibraryBridgeGenerator`（`:812-960`）生成 `partial` 方法声明/`DllImport` 分派 | 不处理 |

**结论（职责边界）**：两者**互补而非冲突** —— 生成器负责"补声明"（让 `[UClass] partial class AMyActor` 拿到编译器生成的 `UMyActor` 静态类等），weaver 负责"改实现"（把属性访问器改成走 native 反射）。这一点核对后**未发现问题**（record 为 negative result，避免后续重复审查）。

**但存在两处真实不一致**：

1. **属性匹配语义不同**（简单名 vs 全名）：
   - 用户若定义 `MyOrg.UClassAttribute` 并标注一个类，**生成器会认**（简单名匹配 → 生成 `U{Name}` partial 声明，且因为需要 `partial`，用户必须写 `partial`），**weaver 不认**（全名不匹配 → 不织入 `[PathName]`、不处理 `[UProperty]`）。
   - 结果：编译**成功**，但运行期元数据缺失（找不到 `/Script/...` 路径、属性不生效）。这是最难查的一类"半生效"状态。
2. **都不处理字段上的 `[UProperty]`**：两者都只看 `Type.Properties` / 属性级 attribute（weaver `:331`、`:550`；生成器未处理属性）。若用户在**字段**上写 `[UProperty] public int Health;`：
   - weaver 的 `ProcessUClassType`（`:331-338`）只遍历 `Type.Properties` → 字段被**静默忽略**；
   - 该字段**保留在托管侧**，成为一份与 UE 属性系统并存的"影子状态" → 从 C# 里读到的值与从编辑器里改的值**不一致**（经典的"改了不生效"）。
   - 由于这是**静默**的（无日志、无诊断），建议至少报一个 warning。

**建议**
1. 把"什么算 `[UClass]`"的判定统一为**全名匹配**（生成器侧改为解析 `using`/命名空间后的符号全名，或用一个共享的常量列表）。若因 Roslyn 语法层拿不到全名，至少在生成器侧**同时**要求"属性的简单名与全名后缀都匹配"。
2. weaver 的 `ProcessUClassType` / `ProcessUStructType` 增加**字段扫描**，对字段上的 `[UProperty]` 抛 `WeavingException` 或至少 `LogWarning("UProperty on fields is not supported; use a property")`。
3. 在 `Script/Weavers/` 或插件文档里写一份**权威的"支持/不支持"清单**（`partial` 必需、只支持属性、不支持嵌套类型、不支持属性初始化器…），把当前散落在代码里的隐式约束集中起来。

**验证方式**
- grep：`Select-String -Path Script\SourceGenerator\UnrealTypeSourceGenerator.cs -Pattern 'UProperty'`（现状 0，证明生成器不处理属性）。
- 用例：字段上加 `[UProperty]` → 观察是否被静默忽略。
- 用例：`MyOrg.UClassAttribute` → 观察生成器与 weaver 行为分裂。

---

## 4. 死代码清单

判定方法：符号名在**整个插件**（`Source/` 全部模块 + `Script/`，排除 `obj/`、`bin/`、`Intermediate/`、`Binaries/`、`ThirdParty/`）中的出现次数。`==1` 表示仅声明/定义，无调用点。
**关键注意**：Bridge 类的方法**多由 C++ 侧以字符串名反射调用**（`TypeBridge.GetFunctionPointer(assembly, type, method)`，见 `IScriptTypes.h:164` 与 `FCoreCLRDomain.cpp:172-175`），因此**必须**同时搜索字符串字面量形态（`FunctionMacro.h` 里的 `FString(TEXT("Xxx"))`）。

### 4.1 C++ 侧桥接函数指针的使用（`Select-String -SimpleMatch`，作用域 `Source/**` 的 `*.cpp,*.h`）

| 生成的成员名 | 命中数 | 命中位置 | 判定 |
|---|---|---|---|
| `MethodBridgeInvoke` | 4 | `FLeanCLRDomain.cpp:151,153,182` + `IScriptTypes.h:150` | **活** |
| `AssemblyLoaderUnload` | 7 | `FCoreCLRDomain.cpp:244,246`、`FLeanCLRDomain.cpp:810,812`、`FMonoDomain.cpp:432,434` + 声明 | **活** |
| `AssemblyLoaderLoadFromStream` | 5 | `FCoreCLRDomain.cpp:226,228`、`FMonoDomain.cpp:395,415` + 声明 | **活（仅 CoreCLR/Mono）** |
| `HandleDataFree` | 3 | `FLeanCLRDomain.cpp:139,143` + 声明 | **活（仅 LeanCLR）** |
| `HandleDataGetObjectPointer` | LeanCLR 专属宏：`FLeanCLRDomain.h:196` 声明；使用 `FLeanCLRDomain.cpp:125,155,167,176,196` | — | **活（仅 LeanCLR）** |
| `MethodBridgeRegisterBinding` | **1** | 仅 `IScriptTypes.h:166` | 声明使用；实例以 `GetFunctionPointer` 名字路由（`FCoreCLRDomain.cpp:197`、`FLeanCLRDomain.cpp:55`、`FMonoDomain.cpp:169` 调 `RegisterBinding()`）→ **活，但 `Op` 声明本身是死声明** |
| `ObjectBridgeNewObject` | **1** | 仅 `IScriptTypes.h:145` | 同上（`FClassReflection.cpp:704-708` → `ScriptDomain->NewObject`） |
| `StringBridgeNewString` / `StringBridgeGetString` | **1** / **1** | 仅 `IScriptTypes.h:152/153` | 同上（`FRegisterString.cpp:46`、`FDomain.cpp:78`） |
| `ArrayBridgeNewArray` / `ArrayBridgeArrayGet` | **1** / **1** | 仅 `IScriptTypes.h:155/156` | 同上（`FClassReflection.cpp:758-764`、`:51-53`） |
| `LogBridgeSetLog` / `LogBridgeInitialize` | **1** / **1** | 仅 `IScriptTypes.h:161/162` | 同上（`FLeanCLRDomain.cpp:69-74` 用 `InitializeLeanCLR`；`SetLog` 见 `FunctionMacro.h:45`） |
| `FieldBridgeSetStaticValue` / `FieldBridgeGetStaticValue` | **1** / **1** | 仅 `IScriptTypes.h:147/148` | **疑似无论哪种机制都无调用点**（见 4.2） |

### 4.2 C# 侧符号的调用点（`Script/**`）

| C# 符号 | 声明 | 全插件命中 | 判定 | 证据 |
|---|---|---|---|---|
| ~~`HandleData.GetObjectPointers`（复数）~~（**已删除**，提交 `4f351f12`） | ~~`HandleData.cs:118`~~ | **2** → **0**（自身声明 + `FunctionMacro.h:41` 的宏；原文写 3 系把单数内部调用计入） | **强死代码 → 已删除** | 复数**无 typedef、无 `Op`、无 C++ 调用**（判据 = 无 `Op(...)` 表项，而非 C# 引用数）；C# 内亦无调用者 |
| `HandleData.GetObjectPointer`（单数） | `HandleData.cs:108` | **5**（1 声明 + C++ LeanCLR 4 处；原 6 含复数版内部调用，随其删除归零） | **活（仅 LeanCLR）** | `FLeanCLRDomain.cpp:125,155,167,176,196` |
| `MethodBridge.GetMethod` | `MethodBridge.cs:140` | 待 `Script/UE/**` 走查（见第 6 节） | 待定 | — |
| `MethodBridge.Invoke` | `MethodBridge.cs:33` | C++ 3 处 + 声明 | 活 | `FLeanCLRDomain.cpp:151,153,182` |
| `MethodBridge.RegisterBinding` | `MethodBridge.cs:13` | 3 调用 + 声明 | 活 | `FCoreCLRDomain.cpp:197`、`FLeanCLRDomain.cpp:55`、`FMonoDomain.cpp:169` |
| `MethodBridge.Clear` | — | **0（不存在）** | **缺失 API** | 见 F-CS1-018 |
| `FieldBridge.SetStaticValue` / `GetStaticValue` | `FieldBridge.cs:11,22` | 各 **3**（声明 + `FunctionMacro.h:119/121` 名字 + `IScriptTypes.h:147/148` 声明） | **无任何调用点 → 死代码嫌疑（高）** | 见 F-CS1-013 |
| `HandleData.Alloc(object, bool bPinned)` 的 `bPinned: true` 用法 | `HandleData.cs:27` | **1**（`ArrayBridge.cs:19`） | 活但仅 1 处 | — |
| `LogBridge.InitializeLeanCLR` | `LogBridge.cs:47` | `FunctionMacro.h:50` + `FLeanCLRDomain.cpp:69` + 声明 | 活（仅 LeanCLR） | — |
| `LogBridge.SetLog` / `Initialize` / `Deinitialize` | `LogBridge.cs:31,37,55` | `FunctionMacro.h:45`/`IScriptTypes.h:161`；`Deinitialize` 需在 C++ 侧复核 | `Deinitialize` **待核实**（见第 7 节） | — |
| `TypeBridge.Clear` | `TypeBridge.cs:501` | **1**（`AssemblyLoader.cs:37`） | 活（**但被 F-CS1-001 的守卫挡住**） | — |
| `TypeBridge.GetTypeImplementation` | `TypeBridge.cs:446` | 4（`ArrayBridge.cs:15`、`TypeBridge.cs:23,82` + 声明） | 活 | — |
| `AssemblyLoader.CurrentContext` | `AssemblyLoader.cs:11` | 2（`TypeBridge.cs:80,463`） | 活 | — |
| `TypeBridge.Box*/Unbox*`（22 个） | `TypeBridge.cs:204-444` | 各 1（仅自身）+ `IScriptTypes.h:121-143` 的 `Op` 声明 | 经 `GetFunctionPointer` 名字路由 → **活**，但直接调用点为 0 | — |
| `Weavers.UnrealTypeWeaver.EmitPropertyAttributesField` | `:242` | 2 调用（`:355`、`:574`）+ 声明 | **编译期恒不可达**（`WITH_LEANCLR` 从未定义）→ 见 F-CS1-026 | `Weavers.csproj` 无 `DefineConstants`；`Script/Shared.props` 不存在 |
| `Weavers.UnrealTypeWeaver.GetRootNamespace` | `:230` | 1 调用（`:250`）+ 声明 | 同上（恒不可达） | — |
| `Weavers.UnrealTypeWeaver.GetAllMeta` 中的 13 个 `_xxx` 字段 | `:25-57` | 全部被使用 | 活 | — |

### 4.3 grep 命令（可复现）

```powershell
# C++ 侧桥接成员（PascalCase）
Get-ChildItem -Path Source -Recurse -Include *.cpp,*.h -File |
  Where-Object { $_.FullName -notmatch '\\(Intermediate|Binaries|ThirdParty)\\' } |
  Select-String -Pattern 'FieldBridgeSetStaticValue' -SimpleMatch | Measure-Object

# C# 侧符号
Get-ChildItem -Path Script -Recurse -Include *.cs -File |
  Where-Object { $_.FullName -notmatch '\\(obj|bin)\\' } |
  Select-String -Pattern 'GetObjectPointers' -SimpleMatch | Measure-Object

# 名字字符串形态（判断"是否只被字符串名引用"）
Select-String -Path Source\UnrealCSharpCore\Public\CoreMacro\FunctionMacro.h -Pattern 'FIELD_BRIDGE|OBJECT_POINTER'
```

## 5. IDisposable 正确性表

| 类型 | 是否 `IDisposable` | 是否有 finalizer / `SafeHandle` | 重复 `Dispose` 安全 | `Dispose` 后使用行为 | 结论 |
|---|---|---|---|---|---|
| `Interop.HandleData`（静态） | **否** | 否（内部持有 `GCHandle`，是**裸句柄而非 `SafeHandle`**） | `FreeImplementation` 对未知句柄是 no-op（`HandleData.cs:52` `TryGetValue` 失败即返回）→ **重复 Free 安全** | `GetObject(已释放句柄)` 返回 `null`（`:101`）→ 调用方拿到 `null`，静默降级 | ⚠️ 应用 `SafeHandle`：`GCHandle` 的**释放责任完全落在 C++ 侧**，遗漏即永久泄漏（F-CS1-002）。且**没有任何终结器兜底**——这是最严重的"漏 Dispose 就泄漏"设计 |
| `Interop.HandleData.HandleReference`（私有） | 否 | 否 | — | — | 纯数据容器，`ConditionalWeakTable` 管理其生命周期 ✓ |
| `Interop.AssemblyLoader`（静态） | **否** | 否 | `Unload()` 可重复调用（`Context` 置 null 后第二次进不去）→ 安全 | 二次 `LoadFromStream` 会重建 ALC ✓ | `Context` 是**强静态引用**，由 `Unload()` 在 `finally` 中置 null ✓（`:45-48`）—— 这一点做对了 |
| `Interop.UnrealAssemblyLoadContext` | 继承自 `AssemblyLoadContext`（**`IDisposable` 语义通过 `Unload()`**） | 有 `isCollectible: true` ✓，无 finalizer | `Unload()` 可重复调用（ALC 内部保证） | 卸载后使用其 `Assembly`/`Type` → 抛异常或未定义 | ✓ **正确使用了 collectible + Unload + WeakReference 轮询**（`:39-59`）—— 值得肯定 |
| `Interop.LogBridge` : `TextWriter` | **是**（继承 `TextWriter`，`IDisposable`） | 无 finalizer | `Dispose(true)` 只 `Flush()` 后调 `base.Dispose` → 重复安全 ✓ | 之后 `Write` 仍可用（`Buffer` 未被释放）→ **Dispose 后仍可写**（不抛异常），语义宽松 | ⚠️ 实例由 `Console.SetOut` 持有，**用户代码不会也不应 Dispose 它**；`Deinitialize()`（`:55-75`）负责恢复原 writer ✓ |
| `Interop.*Bridge`（8 个静态类） | 否 | 否 | — | — | 纯静态门面，无状态需要释放 ✓ |
| `Weavers.UnrealTypeWeaver` | 否 | 否 | — | — | 编译期工具 ✓ |

**核心结论**：本模块**没有任何 `SafeHandle`**，`GCHandle` 与原生函数指针都是**裸 `nint`/裸指针**；释放责任 100% 在 C++ 调用方。结合 F-CS1-001（`Clear()` 在 LeanCLR 下被跳过）与 F-CS1-002（强句柄使对象永不可回收），构成一个"**双重兜底都缺失**"的结构：既无终结器、也无 CodeAgent 侧的批量清理真正生效。

**关键交叉验证（回答任务中的"已发现"第 2 条）**：
`UnrealTypeWeaver` 为每个 `[UStruct]` 生成的**唯一**解绑路径就是终结器：

```csharp
// Script/Weavers/UnrealTypeWeaver.cs
119:             if (Type.Methods.Any(Method => Method.Name == "Finalize") == false)
120:             {
121:                 var superFinalize = Type.BaseType.Resolve().Methods.FirstOrDefault(Method => Method.Name == "Finalize");
123:                 var destructor = new MethodDefinition("Finalize",
124:                     MethodAttributes.HideBySig | MethodAttributes.Family | MethodAttributes.Virtual,
125:                     ModuleDefinition.TypeSystem.Void);
...
129:                 destructor.Body.Instructions.Add(Instruction.Create(OpCodes.Ldarg_0));
131:                 destructor.Body.Instructions.Add(Instruction.Create(OpCodes.Call,
132:                     ModuleDefinition.ImportReference(_getHandle)));           // HandleData.GetHandle(this)
134:                 destructor.Body.Instructions.Add(Instruction.Create(OpCodes.Call,
135:                     ModuleDefinition.ImportReference(_structUnRegisterImplementation)));  // UStruct_UnRegisterImplementation
```
`_getHandle` 解析为 `Interop.HandleData.GetHandle`（`:1156-1158`）。而 `HandleData.Alloc` 用 `GCHandleType.Normal`（强句柄，`HandleData.cs:39`）→ **只要该对象拿到过句柄，它就是永久 GC root → `Finalize` 永不执行 → `UStruct_UnRegisterImplementation` 永不调用**。这是 F-CS1-002 所述"终结器永不执行"的**确证链路**（不是推测）。同一个 `Finalize` 还调用 `Object.Finalize`（`:139-141`），因此该结构体对象在正常情况下会**被提升到 finalization 队列**，但在强句柄下根本不会变成不可达对象。

## 6. 空函数指针调用点表

| 返回值来源 | 返回值语义 | 是否被调用方判空 | 证据 |
|---|---|---|---|
| `MethodBridge.GetMethod(ref slot, name)`（`MethodBridge.cs:140-148`） | 查不到 → `nint.Zero`（`:144`），**不抛错、不日志** | **否 —— 唯一的调用点也不判空** | 唯一调用点 = `Script/SourceGenerator/UnrealTypeSourceGenerator.cs:917-918` 生成的模板：`((delegate* unmanaged[Cdecl]<...>)global::Interop.MethodBridge.GetMethod(ref {slot}, "{key}"))({arguments})` —— 强转后立即调用，**模板里没有任何判空**。该模板被用于**每一个**库桥接方法（`Script/UE/Library/*Implementation.cs`、`Script/UE/CoreUObject/Utils.cs` 等全部 `partial` 桥接）。**结论：100% 调用点不判空 → 名字查不到即 `((delegate*)0)(args)` = 跳转地址 0（P0）** |
| `MethodBridge.Invoke` 返回值 0（`:103-106` 方法返回 null；`:123` 句柄无效；`:136` 异常） | 0 = "无结果"（三种原因不可区分） | 未在 C++ 侧统一判空 | `FCoreCLRDomain.cpp:231` 对 `LoadFromStream` 用了 `IManagedHandleIsValid` ✓；`Invoke` 的返回值使用点未逐一核实 | 
| `AssemblyLoader.LoadFromStream` 返回 0（`:27` 入参无效） | 0 = 失败 | ✓ | `FCoreCLRDomain.cpp:231` `IManagedHandleIsValid(Handle)` |
| `TypeBridge.GetFunctionPointer` 返回 0（`:97`） | 0 = 未找到类型/方法 | **否**（需核实） | `FCoreCLRDomain.cpp:172-175`：`InFn = reinterpret_cast<decltype(InFn)>(TypeBridgeGetFunctionPointerFn(...))` 赋值后**未判空**即被后续 `InFn(...)` 调用；`FMonoDomain.cpp:147` 同 | 
| `TypeBridge.GetClass` / `GetType` / `GetMethod` / `MakeGenericType` / `MakeGenericType2` 返回 0 | 0 = 未找到 | ✓ 多数路径 | `FClassReflection.cpp:58,72,86,100,716` 用 `IManagedHandleIsValid` |
| `ArrayBridge.NewArray` / `ArrayGet` 返回 0 | 0 = 失败 | ✓ | 同上 |
| `ObjectBridge.NewObject` 返回 0 | 0 = 类型句柄无效 | ✓ | `FClassReflection.cpp:716` |
| `StringBridge.NewString` 返回 0 | 0 = 入参为 null | 待核实 | — |
| `FieldBridge.GetStaticValue` 返回 0 | 0 = 字段不存在 **或** 字段值就是 0 → **语义混淆** | 无调用点 | — |
| `TypeBridge.Unbox*` 返回 0 | 0 = 类型不匹配，且**未写入 out 参数** → 调用方可能读未初始化内存 | 待核实（C++ 侧可能误按"句柄有效性"解释） | — |
| `LogBridge.LogFn`（`static delegate*`，`:11`） | **可为 null**（LeanCLR 下永不 `SetLog`） | ✓ `if (LogFn != null)`（`:165`） | 现状正确；但无 `volatile`，`Deinitialize` 并发清空后可能命中悬空指针（F-CS1-014） |

**最严重的一条**（`MethodBridge.GetMethod`）的完整证据链见 F-CS1-003 与 F-CS1-040；`TypeBridge.GetFunctionPointer` 返回 0 未判空（`FCoreCLRDomain.cpp:172-175`）是**次严重**的一条 —— 这是 C++ 获取**所有** C# 桥接函数指针的唯一入口，返回 0 即意味着后续所有对 C# 的 `UnmanagedCallersOnly` 调用都变成跳转地址 0。**该路径需在 C++ 侧补确认**（列第 7 节）。

## 7. 未覆盖/存疑项

### 7.1 未覆盖（本次分析窗口内未完成）

1. **C++ 侧 `TypeBridgeGetFunctionPointerFn` 返回 0 后是否判空**未逐处核实 → 这决定了"次严重的一条空函数指针路径"。
   已知证据（`Source/UnrealCSharpCore/Private/Domain/CoreCLR/FCoreCLRDomain.cpp:172-175`）：
   ```cpp
   172: 	if (TypeBridgeGetFunctionPointerFn != nullptr)
   173: 	{
   174: #define GET_UTILS_FUNCTION_POINTER(InFn, InName) \
   175: 	InFn = reinterpret_cast<decltype(InFn)>(TypeBridgeGetFunctionPointerFn( \
   ```
   以及 `FMonoDomain.cpp:147` 同款。`InFn` 被赋值后是否判空需读完整的宏体（`GET_UTILS_FUNCTION_POINTER` 的后续行）与调用点。**建议在 C++ 侧补确认**（属 `01-UnrealCSharp运行时` / `02-UnrealCSharpCore核心` 范围）。
2. **C++ 侧 `HandleData.Free`（`handle_data_free_fn`）的调用点**未核对 → 决定 F-CS1-002 的**实际泄漏速率**。已知 `FLeanCLRDomain.cpp:139-143` 只在 LeanCLR 下调用（`if (HandleDataFreeFn != nullptr) Bridge_Invoke(HandleDataFreeFn, InManagedHandle);`），CoreCLR/Mono 路径是否有对应释放未核实。
3. **`FLeanCLRDomain.cpp:854` 的 `RegisterBinding()` 已读**（见 F-CS1-003 的"顺带核实"），确认 LeanCLR 不走 `MethodBridge.RegisterBinding`；但 **`CoreCLR`/`Mono` 的 `RegisterBinding()` 是 `FScriptDomainImpl.inl:601-633` 的通用实现**，其中 `Method.GetMethod()` 的**确切字符串内容**（是否已含 `"Namespace.Class::Method"` 全形式）只通过 `COMBINE_FULL_NAME`/`BINDING_COMBINE_FUNCTION` 宏推断，未读 `FBinding` 的注册实现来最终确认 → F-CS1-040 的"当前一致"结论置信度**高但非确定**。
4. **C++ 侧 `type_bridge_get_namespace_fn` / `_get_name_fn` / `_get_full_name_fn` 的缓冲区大小来源**未核实 → 决定 F-CS1-021（UTF-8 字符截断崩溃）的触发概率。
5. **C++ 侧标量 `Unbox` 返回值的语义**（成功标志 vs 句柄有效性）未核实 → F-CS1-025 第 2 点。
6. **`LogBridge.Deinitialize` 的 C++ 调用点**未核实（`IScriptTypes.h` 只有 `LogBridgeInitialize` 的 `Op`，没有 `Deinitialize`）→ 若从未被调用，`ConsoleOut`/`ConsoleError` 静态引用与 `LogFn` 都不会复位（F-CS1-014 第 4 点）。
7. **`EmitPropertyAttributesField` 的编码格式与 `Utils.cs` 消费端的契约**未逐字节核对 → F-CS1-037 第 4 点（`FullName.Replace('/', '+')` 是否被消费端正确解析）。
8. **`FVector`/`FRotator` 等 UE 结构体在 C# 侧生成为 `struct` 还是 `class`** 未核实 → 决定 F-CS1-029 是 P2 还是"不成立"。
9. **`HandleData.Free` 的配对率**：需要一个运行期统计（`Alloc` 次数 vs `Free` 次数）来量化 F-CS1-002 的泄漏速率，本次未做。
10. **LeanCLR 的 `UnmanagedCallersOnly` 异常语义**：.NET 5+ 的"异常逃逸即 FailFast"是规范行为，但 LeanCLR 是第三方运行时（`Source/ThirdParty/LeanCLR`，按约定不分析），其行为未核实 → 影响 F-CS1-010 / F-CS1-021 的实际后果描述。

### 7.2 存疑（已给出结论但置信度非"高"）

| 编号 | 存疑点 | 置信度 |
|---|---|---|
| F-CS1-027 | backing field 被移除后 Cecil 的行为（写回报错 vs 保留陈旧 token → 静默数据损坏）取决于 Mono.Cecil 版本的实现细节，未实际运行验证 | 中—高 |
| F-CS1-036 第 5 点 | `Instruction`/`ExceptionHandler` 对象跨方法体共享是否真的会出错，取决于 Cecil 内部是否只依赖 `body.Instructions` 列表顺序 | 中 |
| F-CS1-029 | 是否真的存在结构体类型的 `[UProperty]` 属性 | 中 |
| F-CS1-033 | "二次织入"是否会在本项目的真实构建管线中发生 | 中 |
| F-CS1-010 / F-CS1-021 | `[UnmanagedCallersOnly]` 中异常逃逸导致进程终止的具体表现（FailFast vs 由宿主捕获）在不同 .NET/LeanCLR 运行时上可能不同；`UnmanagedCallersOnly` 的异常语义是 .NET 5+ 的规范行为，但 LeanCLR 是第三方运行时（`Source/ThirdParty/LeanCLR`），其行为未核实 | 中—高 |

### 7.3 已核对为"无问题"的检查项（negative results，避免重复审查）

| 检查项 | 结论 | 证据 |
|---|---|---|
| `AssemblyLoadContext` 是否 `isCollectible: true` 且被 `Unload()` | **是**，且 `Unload()` 前置了 `HandleData.Clear()` / `TypeBridge.Clear()`，并用 `WeakReference` + `GC.Collect()` 轮询 2 秒 | `UnrealAssemblyLoadContext.cs:8`；`AssemblyLoader.cs:33-59` |
| 文件流 / `FileStream` 是否未释放导致 DLL 被锁定 | **已正确 `using`**，且使用 `File.ReadAllBytes` + `MemoryStream`，**不会锁定 DLL** | `UnrealAssemblyLoadContext.cs:38-42` |
| `Load(AssemblyName)` 递归加载 A↔B 互相依赖导致栈溢出 | **不成立**：`Load` 只做单层解析，`LoadFromStream(string)` 是私有重载，不会再回调 `Load` | `UnrealAssemblyLoadContext.cs:10-43` |
| `StringBridge.GetString` 的缓冲越界 | **无越界**：`Length < InSize` 与 `InBuffer[Length] = '\0'`（2 字节）所需条件 `Length + 1 <= InSize` 等价 | `StringBridge.cs:28,32,35` |
| `LogBridge` 的 `stackalloc byte[MaxByteCount]` 容量 | **安全**：`GetMaxByteCount` 是上界，`UTF8[Size] = 0` 在容量内 | `LogBridge.cs:151-159` |
| `Weavers/FodyWeavers.xml` 的配置键与代码读取的键是否一致 | **不适用**：weaver 从不读 `Config`（grep 0 命中），无配置键 → 不存在"不一致导致配置失效" | `UnrealTypeWeaver.cs` grep `Config` = 0 |
| `ProcessStructRegister` 生成的 `Finalize` 的异常处理结构是否正确 | **正确**：`TryStart=[0]`、`TryEnd=[4]`、`HandlerStart=[4]`、`HandlerEnd=[7]`，配合 `leave.s` 于索引 3，构成合法的 `try { … } finally { base.Finalize(); }` | `UnrealTypeWeaver.cs:129-158` |
| weaver 与 `UnrealTypeSourceGenerator` 是否功能冲突 | **互补，不冲突**：生成器补 `partial` 声明，weaver 改 IL 实现。但两者的属性匹配语义不一致（见 F-CS1-039） | 两侧代码均已读 |
| `AddPathNameAttributeToUnrealType` 的幂等性 | **幂等**（`Any(...) == false` 查重） | `:189-190`、`:208-209` |
| `HandleData` 的线程安全（字典操作） | **正确加锁**（`System.Threading.Lock`）—— 与 `MethodBridge`/`TypeBridge` 的裸字典形成对比 | `HandleData.cs:9,29,50,84,99,131` |
| `LogBridge.Deinitialize` 中"先恢复 Console writer 再清空 LogFn"的顺序 | **顺序正确**（先 `Console.SetOut` 再 `LogFn = null`） | `:57-74` |
| `GetTypeStind` 对 `Byte`/`SByte` 的处理 | **正确**（都用 `Stind_I1`，1 字节写入对两者均成立） | `:792-793`、`:800-801` |
| `UnrealAssemblyLoadContext.Load` 对 `Interop` 的特判分支 | 逻辑正确，但**缺少"是否等于 this"的自递归守卫**（F-CS1-009 第 1 点） | `:14-19` |

---

## 附：发现索引（按严重度）

| 编号 | 严重度 | 标题（简） | 文件 |
|---|---|---|---|
| F-CS1-003 | P0 | `MethodBridge.GetMethod` 返回 0 且无调用点判空 → 跳转地址 0 | `MethodBridge.cs:140-148` |
| F-CS1-010 | P0 | `ArrayBridge.ArrayGet` 不检查负索引 → 异常穿越 `UnmanagedCallersOnly` | `ArrayBridge.cs:26-43` |
| F-CS1-001 | P1 | LeanCLR 下 `Context` 恒 null，`Unload()` 清理被整体跳过 | `AssemblyLoader.cs:31-61` |
| F-CS1-002 | P1 | `Alloc` 用 `GCHandleType.Normal` → 终结器永不执行，`Free` 是唯一解绑路径 | `HandleData.cs:27-44` |
| F-CS1-004 | P1 | `GetObjectPointer` 返回未 pinned 托管对象裸地址 → GC 压缩后失效 | `HandleData.cs:108-127` |
| F-CS1-011 | P1 | `FieldBridge` 引用类型字段取值/赋值语义完全错误 | `FieldBridge.cs:49-64` |
| F-CS1-019 | P1 | `MethodBridge.StringToMethod` 并发读写无锁 | `MethodBridge.cs:10,25,144` |
| F-CS1-020 | P1 | `MakeGenericType2` 复制粘贴错误 → `Dictionary<K,K>` | `TypeBridge.cs:187-202` |
| F-CS1-021 | P1 | 泛型名按字符数截断后 UTF-8 编码 → 非 ASCII 类型名崩溃 | `TypeBridge.cs:100-171` |
| F-CS1-026 | P1 | `WITH_LEANCLR` 从未在 Weavers.csproj 定义 → 生产者恒缺失 | `Weavers.csproj` / `UnrealTypeWeaver.cs:229-356` |
| F-CS1-027 | P1 | 删除 backing field 不清理引用它的 IL（属性初始化器） | `UnrealTypeWeaver.cs:341-352, 560-571` |
| F-CS1-028 | P1 | RPC 缓冲 `sbyte BufferSize` 在 ≥128 字节时溢出 → 栈损坏 | `UnrealTypeWeaver.cs:916-954` |
| F-CS1-030 | P1 | `GetAllMeta` 13 处解析零 null 检查 → 裸 NRE / 远处报错 | `UnrealTypeWeaver.cs:1099-1159` |
| F-CS1-005 | P2 | `MethodBridge.Invoke` 反射慢路径（无委托缓存） | `MethodBridge.cs:33-138` |
| F-CS1-006 | P2 | `Invoke` 的 catch 吞异常，与"返回 null"不可区分 | `MethodBridge.cs:125-137` |
| F-CS1-007 | P2 | `StringBridge` 编码不对称 + native 内存所有权不明 | `StringBridge.cs:9-43` |
| F-CS1-008 | P2 | `ObjectBridge.NewObject` 绕过构造函数与字段初始化器（AOT 不可用） | `ObjectBridge.cs:9-20` |
| F-CS1-009 | P2 | ALC 依赖解析不校验版本；`Interop` 分支缺少自递归守卫 | `UnrealAssemblyLoadContext.cs:10-43` |
| F-CS1-012 | P2 | `ArrayBridge.NewArray` 永久 pin 数组 → 强根 + 堆碎片 | `ArrayBridge.cs:19` |
| F-CS1-013 | P2 | `FieldBridge` 被注册但 C++ 零调用点（上膛的枪） | `FieldBridge.cs:11,22` |
| F-CS1-014 | P2 | `LogBridge` 重入丢日志 + `LogFn` 静态字段无同步 | `LogBridge.cs:11,139-174` |
| F-CS1-018 | P2 | `StringToMethod` 无 `Clear()` → 热重载后残留失效指针 | `MethodBridge.cs:10` |
| F-CS1-022 | P2 | `TypeBridge.GetMethod` 只用名字+参数量匹配重载，且无缓存 | `TypeBridge.cs:48-75` |
| F-CS1-023 | P2 | `StringToType`/`Clear()` 并发无锁 | `TypeBridge.cs:12,448,471,501` |
| F-CS1-024 | P2 | 无程序集限定名时全程序集线性扫描取首个同名类型 | `TypeBridge.cs:446-499` |
| F-CS1-029 | P2 | `IsCompound` 与尺寸/存取默认分支假设冲突（结构体） | `UnrealTypeWeaver.cs:1065, 866-903` |
| F-CS1-031 | P2 | `GetUEAssemblyPath` 无校验 → 静默错误路径 | `UnrealTypeWeaver.cs:1161-1183` |
| F-CS1-032 | P2 | 属性重写依赖"新变量即 0 号槽"，不清空既有局部变量 | `UnrealTypeWeaver.cs:377-499` |
| F-CS1-033 | P2 | weaver 幂等性：RPC 路径重复 Add 字段/方法，结构体按固定索引插桩 | `UnrealTypeWeaver.cs:905-910, 111-116` |
| F-CS1-034 | P2 | 只扫描顶层类型（漏 `NestedTypes`）+ 属性匹配两套策略 | `UnrealTypeWeaver.cs:1067-1097` |
| F-CS1-036 | P2 | `_Implementation` 丢失参数名/属性/泛型参数，属性位被改为 Public | `UnrealTypeWeaver.cs:1022-1063` |
| F-CS1-037 | P2 | `_Attrs` 编码丢失命名参数，数组参数序列化成 `"System.Object[]"` | `UnrealTypeWeaver.cs:242-326` |
| F-CS1-015 | P3 | 命名空间声明风格不统一（file-scoped vs block-scoped） | 4 个 Bridge 文件 |
| F-CS1-016 | P3 | `[UnmanagedCallersOnly]` 是否显式 `CallConvs` 不一致 | 全部 Bridge |
| F-CS1-017 | P3 | `FieldBridge.GetField` 每次反射查找无缓存 | `FieldBridge.cs:34-47` |
| F-CS1-025 | P3 | 22 个 Box/Unbox 样板可收敛；缺 `char` 分支导致 switch 表达式抛异常 | `TypeBridge.cs:204-444` |
| F-CS1-035 | P3 | `GetTypeLdind` 的 Byte/SByte 分支写反（靠截断救回） | `UnrealTypeWeaver.cs:826-864` |
| F-CS1-038 | P3 | `FodyWeavers.xml` 无配置键、xsd 缺失、角色不清 | `FodyWeavers.xml` |
| F-CS1-039 | P3 | 与 SourceGenerator 的属性匹配语义不一致；两者都不支持字段级 `[UProperty]` | 两文件 |
| F-CS1-040 | P2 | 名字键 `"Ns.Class::Method"` 是无校验的双向字符串契约；注册失败无检测 | `MethodBridge.cs` / `FScriptDomainImpl.inl` / `UnrealTypeSourceGenerator.cs` |

**统计**：共 **40** 条发现 —— **P0 × 2**、**P1 × 11**、**P2 × 21**、**P3 × 6**。
（另含 1 条并入 F-CS1-002 的 P3 补充：`AssemblyLoader.LoadFromStream` 的 `Context ??=` 静默忽略后续 `InPublishDirectory`。）

**按文件的发现分布**

| 文件 | P0 | P1 | P2 | P3 | 合计 |
|---|---|---|---|---|---|
| `Interop/AssemblyLoader/AssemblyLoader.cs` | 0 | 1 | 0 | 0 | 1 |
| `Interop/AssemblyLoader/UnrealAssemblyLoadContext.cs` | 0 | 0 | 1 | 0 | 1 |
| `Interop/Bridge/ArrayBridge.cs` | 1 | 0 | 1 | 0 | 2 |
| `Interop/Bridge/FieldBridge.cs` | 0 | 1 | 2 | 1 | 4 |
| `Interop/Bridge/LogBridge.cs` | 0 | 0 | 1 | 0 | 1 |
| `Interop/Bridge/MethodBridge.cs` | 1 | 1 | 4 | 0 | 6 |
| `Interop/Bridge/ObjectBridge.cs` | 0 | 0 | 1 | 0 | 1 |
| `Interop/Bridge/StringBridge.cs` | 0 | 0 | 1 | 0 | 1 |
| `Interop/Bridge/TypeBridge.cs` | 0 | 2 | 3 | 1 | 6 |
| `Interop/Handle/HandleData.cs` | 0 | 2 | 0 | 0 | 2 |
| `Weavers/UnrealTypeWeaver.cs` | 0 | 4 | 7 | 2 | 13 |
| `Weavers/FodyWeavers.xml` / `Weavers.csproj` | 0 | 1 | 0 | 1 | 2 |
| 跨文件（一致性/契约） | 0 | 0 | 1 | 1 | 2 |
| **合计** | **2** | **11** | **21** | **6** | **40** |

---

## 附：执行建议（按投入产出比排序）

1. **立刻可改、收益最大（3 行以内）**
   - `ArrayBridge.cs:31`：`if (InIndex < Array.Length)` → `if ((uint)InIndex < (uint)Array.Length)`（消除 P0 崩溃）。
   - `TypeBridge.cs:194`：`GetObject(InKeyType)` → `GetObject(InValueType)`（消除 P1 的功能错误）。
   - `AssemblyLoader.cs:33`：把 `HandleData.Clear()` / `TypeBridge.Clear()` / `MethodBridge.Clear()` 移出 `if (Context != null)`（消除 LeanCLR 全量泄漏）。
2. **需要设计决策**
   - `HandleData.cs:39`：`GCHandleType.Normal` → `Weak`（或引入 `SafeHandle`）。这是本模块"终结器永不执行"的根因，影响面最广（含 weaver 为 `[UStruct]` 生成的唯一解绑路径）。
   - `Weavers.csproj`：`WITH_LEANCLR` 从未定义 → LeanCLR 的属性元数据字段永不发射（F-CS1-026）。建议改为 Fody `Config` 运行期开关。
3. **需要运行验证（不能靠静态阅读定性）**
   - F-CS1-027（backing field 移除后 Cecil 的写回行为）：用带属性初始化器的 `[UProperty]` 做一次最小复现，看是构建失败还是静默数据损坏 —— 这决定了它的最终定级（P1 vs P0）。
   - F-CS1-028（`sbyte BufferSize` 溢出）：写一个 ≥128 字节参数的 RPC，反编译看 `localloc` 前的立即数。

