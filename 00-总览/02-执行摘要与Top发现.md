# UnrealCSharp 插件分析 —— 执行摘要与 Top 发现（最终版）

> 分析目标：`Plugins/UnrealCSharp`（`UnrealCSharp.uplugin` VersionName = 1.2.0，作者 crazytuzi）
> 本文件性质：**跨报告综合视图（不改源码、不重新分析）**，它的每一条都指向一份已落盘的下级报告。
> 数据来源（**唯一来源**）：对 47 份报告正文的逐条汇总 + `09-问题清单/01-全局问题总表.md`（834 行总表）。
> 状态：**已完成**（现状）

---

## ⚠️ 顶部更新声明：旧版统计与部分 Top 判断**已被推翻**

**本文件是最终版。** 它的前一版（1288 行）基于一个更早的中间快照。那个快照的数字、以及建立在这些数字之上的 Top 排序，**已被 47 份报告 × 逐条全量核对取代**：

| 前一版（旧快照，**已作废**） | 本版（现状，**以此为准**） |
|---|---|
| 基于 **807 条**带 `[F-*]` 编号的发现（并称 P0 = 55、P1 = 200、P2 = 309、P3 = 243） | 逐条汇总 **834 条**发现（P0 **26** / P1 **128** / P2 **267** / P3 **381** / 撤销 **32**） |
| 下级报告 **42 份**；后置更正公告又说"44 份" | **47 份** `.md` 文档（其中 **41 份**含标题式发现块） |
| `INDEX.md` 目录总表口径的 **P0 初判 × 56** 曾被当作"最终级别" | **逐条 `严重度` 字段口径 = P0 26**；`56` 是 **V1 初判**口径，**不可作最终级别引用** |
| "`FReflectionRegistry.cpp:850-1811` 是最大覆盖空洞" | **误判**（删除中间文件造成），该区间由存活的 `02-…/01` 承接覆盖 |
| "部分报告仍未完成" | 47 份全部完成 |
| 无 `复核结论` / `可达性` 两个维度 | 复核结论 **确认 481 / 部分确认 319 / 证伪 26 / 无法验证 3**（另 **5** 条非标准书写）；可达性 **活跃 637 / 潜伏 140 / 不可达 52** |
| 排期含 `F-ARCH-005`、`F-ARCH-006`、`F-REG-005`（旧结论）、`F-CMP-031`、`F-ABI-022` 等 | 这些已**撤销 / 撤回**（`F-REG-005` 恢复 P2；`F-ARCH-005/006`、`F-CMP-031`、`F-ABI-022` 等判撤销）—— **本版不再列入 Top** |

**三条必须知道的口径纪律**（否则会重复这些统计口径错误）：

1. **凡统计级别/结论/可达性，一律复用逐条汇总的结果，不要自写正则。** 该汇总口径自身经历过三次修正：① 用"整行含 `P0`"判级别 ⇒ P0 虚高到 **38**；② 未处理 `~~P2~~ → 撤销` 形态 ⇒ 残留 **27**；③ 用"整行含 `证伪`"判结论 ⇒ 证伪虚高到 **76**（实为 **26**）。
2. **引用编号必须带报告路径** —— 存在 **31 处跨报告重号**（`F-INT2-*` 16 处：`01-…/10b` ↔ `01-…/11`；`F-PROP2-*` 15 处：`01-…/04b` ↔ `01-…/05`）。**同一编号在两份报告里指向完全不同的发现**（例：`F-PROP2-001` 在 `01-…/04b` 里是"撤销（非缺陷）"，在 `01-…/05` 里是 **P0**）。
3. **分节标题 ≠ 最终级别**：为保住编号锚点，**没有**重排报告内的 `### P0/P1/P2/P3` 分节，因此**必须读每条的 `严重度` 字段**，不能按分节标题统计（有 10 份报告存在这种错位）。

> ⚠️ **一处已知的口径不一致（如实记录）**：`09-问题清单/01-全局问题总表.md:22-27` 的表头仍写着"确认 478 / 部分确认 267 / 证伪 81 / 无法验证 8"，那是**未修正的"整行含 `证伪`"旧口径**；本文件与 `INDEX.md` 一律采用逐条汇总的 **481 / 319 / 26 / 3（+5 非标准书写）**。

---

## §0 权威数字表（快照 T1-final，逐条汇总 47 份报告正文）

**取数方式**：逐条汇总 47 份报告正文的结构化字段（口径见下）
**取数时间**：`2026-09-11 01:05:23`｜**取值规则**：用 `^#{2,4}\s*\[?F-<前缀>-<数字>\]?` 定位每条发现的标题行，取该块内**首个** `**严重度**` 字段为最终级别；`复核结论` 取块内首个含该词的行；`可达性` **同时**取独立字段与复核结论行内嵌的"可达性 X"两处。

### 0.1 总量与级别

| 指标 | 数值 | 校验 |
|---|---|---|
| 报告份数（逐条枚举 `.md`） | **47** | 其中 **41 份**含标题式发现块 |
| **发现总数（标题式）** | **834** | 有 `严重度` 字段 **834**、无字段 **0** |
| **P0** | **26** | |
| P1 | **128** | |
| P2 | **267** | |
| P3 | **381** | |
| **撤销（非缺陷）** | **32** | |
| 合计校验 | 26 + 128 + 267 + 381 + 32 = **834** ✅ | 与总数闭合 |
| 唯一编号 | **803**（= 834 − 31 处重号） | 重号仅 `F-INT2-*` 16 + `F-PROP2-*` 15 |

### 0.2 复核结论分布

| 复核结论 | 条数 |
|---|---|
| 确认 | **481** |
| 部分确认 | **319** |
| 证伪 | **26** |
| 无法验证 | **3** |
| 复核结论字段**非标准书写**（未计入四分类） | **5** |
| 合计 | 834 |

### 0.3 可达性分布

| 可达性 | 条数 |
|---|---|
| **活跃** | **637** |
| 潜伏 | **140** |
| 不可达 | **52** |
| 无字段 | **5** |
| 合计 | 834 |

> **仍无复核结论字段的条目**：脚本的"仍在复核定位"一节的明细为 `08-…/05` 2 条、`01-…/10` 1 条、`06-…/03` 1 条、`06-…/02a` 1 条（**共 5 条无结论字段**，与上表"非标准书写 5"呼应）。其余全部 47 份报告均带复核结论。

### 0.4 Top 发现的挑选方法（本文件为何是这 20 条）

排序键**依次**为：

1. **级别 = P0**（崩溃 / 数据损坏 / 内存破坏 / 进程终止）优先；P1 只收"后果确定 + 机制可逐行复现"的少数条。
2. **可达性 = 活跃**优先（`潜伏` = 当前配置不参与编译或需要未开启的选项；`不可达` = 永不触发，**一律不进 Top**）。
3. **影响面**：命中"每次调用 / 每次生成 / 编辑器常驻路径"的排在"特定参数组合"之前。
4. **是否有实机复现路径**：能给出"最小复现步骤 + 真实 `文件:行`"的排在"需要长链推理"之前。
5. **机制是否已逐行复现**：已逐行复现的条目**加分** —— 这是全库证据强度最高的子集。

**被显式排除的**：① 全部 **32 条撤销（非缺陷）**；② 所有"不可达"条目；③ 已被证伪前提的旧 P0（`F-DOM-002`、`F-ED2-004`、`F-ED2-018`、`F-ABI-022`、`F-MACRO-006` 的 `C2062` 断言等）；④ 与 Top 条目**同一病根**的重复条目（见 §3 跨模块缺陷族，聚合计数必须合并）。

> **诚实边界**：本文件的所有结论**均为静态阅读**（读插件源码 + UE 5.6 引擎源码 + LeanCLR / dotnet-runtime 源码 + 真实生成产物）。**没有任何一条 P0/P1 在本机被实际触发过** —— 41 份含发现的报告全部自述"未编译、未运行、未做 ASan/定量测量"。需要实机验证的项见各报告的"未覆盖"章节。

---

## §1 Top 发现清单（20 条：**16 条 P0 + 3 条 P1 + 1 条 P2**）

**挑选依据见 §0.4。** 每条格式：`级别 / 可达性 / 复核结论` → 现象 → 根因链 → 影响面 → 最小修复（真实 `文件:行`）→ 出处（该 Finding 在报告中的行号）。
**⚠️ 编号一律带报告路径**（31 处跨报告重号）。**★** 表示该条已被独立复现/逐字比对覆盖。

### 1.1 机制可逐行复现、且已独立复现的 P0

#### Top 1 ★ `F-BIND-001` — `01-UnrealCSharp运行时/08-绑定层公开头文件（模板元编程）.md`
- **P0 / 活跃 / 确认**（本报告唯一 P0）
- **现象**：`TOut` 用"**全参数前缀偏移**"当作写出步长，导致**第 2 个及之后的 out 参数写错槽位**（数据损坏 + 未初始化读；特定参数组合下越界写）。
- **根因链**：`TBufferOffset.inl:41-45`（`Value[i]` = **全部**参数的前缀和，对每个参数都 `Offset += GetBufferSize()`，无 `IsRef` 门控）→ `TOut.inl:22-40` 的 `:36` `Buffer += std::get<Index>(TBufferOffset<Args0...>()());`，而 `:25` 只对 `IsRef()` 参数**写**、且**先写后推**（`:29/:33` 写，`:36` 推进）→ C# 侧只按 **out 参数紧凑打包**读回（`DateTime.cs:194-216`、`LinearColor.cs:475-489`、`PolyglotTextData.cs:123-141`）。
- **影响面**：`GetDate`（`Year/Month/Day` 三个 out）**三参数全错位**：`OutYear←OutMonth`、`OutMonth←OutDay`、`OutDay←未初始化的 `stackalloc` 栈内存`；`GetIdentity`（2×`FString&`）两参数拿到**同一句柄**。这两条是**插件自带绑定**，无需用户代码即可命中。受影响面取决于 `Intermediate/Build/**/UHT/*.binding.inl` 是否被纳入编译（当前全项目 include 命中 0）。
- **最小修复**：`Public/Binding/Function/TOut.inl:36` 把步长改为"**刚写出的槽位尺寸**"（primitive 用 `sizeof(std::remove_const_t<std::decay_t<T>>)`，否则 `sizeof(void*)`），或保留绝对偏移改为**定位写**（记 `Base`，写 `Base + Value[Index]`、不累加 `Buffer`），并加边界断言。须与反射路径 `Macro/FunctionMacro.h:116-134`（只遍历 out 属性）保持一致。
- **★ 独立复现**：`TBufferOffset.inl`（51 行）与 `TOut.inl`（46 行）已按"三参数各 4 字节且都是 ref"**逐步手算**：写入偏移 **0 / 0 / 4**、C# 读 **0 / 4 / 8**，与报告给出的数字**完全一致**。这是全库证据强度最高的 P0。
- **出处**：`01-UnrealCSharp运行时/08-绑定层公开头文件（模板元编程）.md:108`

#### Top 2 `F-HLP-023` — `01-UnrealCSharp运行时/07-容器Helper.md`
- **P0 / 活跃 / 确认**
- **现象**：`FMapHelper` 的查找类方法返回 `nullptr`，而 `FRegisterMap` 立刻把它交给属性描述符 `Get` 解引用。
- **根因链**：`FMapHelper.cpp:144`（`FindKey` 未找到）/ `:171`（`Get` 未找到）/ `:255`/`:262`（`GetEnumeratorKey/Value` 的 `?: nullptr`）→ `FRegisterMap.cpp:91-92`、`:102-103`、`:124-125`、`:167-169`、`:179-181`（**五处全部无判空**）→ `FPropertyDescriptor::Get` 基类为空实现（`FPropertyDescriptor.cpp:140-155`），子类解引用（`FStrPropertyDescriptor.cpp:6`）。
- **影响面**：C# 的 `map[不存在的键]`（`Script/UE/CoreUObject/TMap.cs:256-276` 索引器 getter **无条件**走到这里）**不是返回默认值而是崩 native 进程**；`GetEnumeratorKey/Value` 的 `nullptr` 上游同样不检查，而"边遍历边修改"是最常见场景。判定理由：键不存在的查询是**常态用法**而非误用输入。
- **最小修复**：`FRegisterMap.cpp:91`、`:102`、`:124`、`:167`、`:179` 五个 `*Implementation` 统一补判空（写类型默认值或告警返回）。
- **出处**：`01-UnrealCSharp运行时/07-容器Helper.md:1759`

#### Top 3 `F-INT2-001` — `01-UnrealCSharp运行时/10b-Interop注册-容器字符串与对象指针.md`
- **P0（崩溃 / 数据损坏）/ 活跃 / 确认**
- **现象**：`TArray` 的 `RemoveAt` / `InsertZeroed` / `SetNum` / `SwapMemory` / `Swap` 导出**未做 index 校验**，Shipping 下构成堆破坏。
- **根因链**：C# 公开层直接透传（`TArray.cs:190-198`、`:306-312`；同文件 `:63-64` 已有 `IsValidIndex` **却不用**）→ 导出零校验（`FRegisterArray.cpp:175-183 / 195-203 / 223-231 / 288-306`）→ `FArrayHelper.cpp:208-228`（`:212 GetRawPtr(InIndex)` 未校验）、`:196-199`、`:251-261`、`:324-332` → 唯一保护是 `checkSlow`（`UnrealType.h:3911-3920`），而 Test/Shipping 下 `check`/`checkSlow` 展开为 `CA_ASSUME`（`AssertionMacros.h:237/245/325-327`）。
- **影响面**：越界 Index 直接进 `FMemory::Memmove`/`Memswap`/`DestructItems`；对 `TArray<FString>`/`TArray<UObject*>` 调 `DestroyValue(Dest)` = **任意地址释放**；`Arr.Add("A"); Arr.RemoveAt(0, 2);` 即可触发（无需恶意输入）。
- **最小修复**：`FArrayHelper.cpp:208-228`、`:196-199`、`:251-261`、`:324-332` 统一前置 `IsValidIndex`/计数校验。
- **出处**：`01-UnrealCSharp运行时/10b-Interop注册-容器字符串与对象指针.md:744`

#### Top 4 `F-PROP2-001` — `01-UnrealCSharp运行时/05-属性描述符-对象与委托类型.md`
- **P0（崩溃）/ 活跃 / 确认**
- **现象**：六个「Multi」描述符在 `Class == nullptr` 时对**空指针调用 `NewObject()`**（共 18 处）。
- **根因链**：`TPropertyDescriptor.inl:10-14` 的 `:12 Class(FTypeBridge::GetClass(InProperty))` **不校验** → 未命中来源 `FReflectionRegistry.cpp:923`/`:974`、`FTypeBridge.cpp:178`、`:421` 的 `return nullptr` → `FInterfacePropertyDescriptor.cpp:10`（同型 `FWeakObjectPropertyDescriptor.cpp:10`、`FLazyObjectPropertyDescriptor.cpp:10`、`FSoftObjectPropertyDescriptor.cpp:10`、`FSoftClassPropertyDescriptor.cpp:10`、`FSubclassOfPropertyDescriptor.cpp:10`，及 `:21`/`:31`）`Class->NewObject()`。
- **影响面**：GameThread 对空指针虚调用 → 直接崩溃。该模式是**全插件系统性**的（43 处 / 18 个描述符），汇总时**按族处理**。
- **最小修复**：`FInterfacePropertyDescriptor.cpp:10`（及同型共 18 处）加 `if (Class == nullptr) { *reinterpret_cast<IManagedHandle*>(Dest) = InvalidManagedHandle; return; }`；更彻底在 `TPropertyDescriptor.inl:10-14` 构造函数 `ensureMsgf`。
- **出处**：`01-UnrealCSharp运行时/05-属性描述符-对象与委托类型.md:197`

#### Top 5 ★ `F-DEL-002` — `01-UnrealCSharp运行时/07b-委托Handler与OptionalHelper.md`
- **P0（崩溃）/ 活跃 / 确认**
- **现象**：多播广播遍历 `TArray` 期间回调可修改该数组 → 迭代器失效 / 越界。
- **根因链**：C# 入口 `FRegisterMulticastDelegate.cpp:106/127/139` → `MulticastDelegateHandler.cpp:11-13` 的 range-for 内同步 `CallDelegate`，而回调可走到 `:111 Remove`、`:126 RemoveAll`、`:149 Clear`→`Empty()` → 引擎 `Array.h:3377-3379`/`:3395-3399`（`end()` 只在循环开始求值一次）、`Empty` → `FMemory::Realloc(Data, 0)`（`Array.h:2286-2312` + `ContainerAllocationPolicies.h:700-724`）。
- **影响面**：`Remove`/`RemoveAll` → 回调被重复调用或静默跳过（Editor/Development 先报 `"Array has changed during ranged-for iteration!"`，`Array.h:263`）；**`Clear()` 路径迭代器悬垂 = use-after-free 真崩溃**（这才是 P0 的核心理由）。
- **最小修复**：`MulticastDelegateHandler.cpp:11` 改为**快照遍历**（`TArray<FDelegateWrapper> Snapshot(DelegateWrappers);` + `Key.IsValid() && Value != nullptr` 守卫），并加防重入标志。
- **★ 独立验证**：**3/3 逐字吻合**（`Array.h:3377-3381`、`:263` ensureMsgf、`TARRAY_RANGED_FOR_CHECKS` 门控）。⚠️ **依据表述须修正**：`TCheckedPointerIterator` 的 `ensureMsgf` 是**诊断而非防护**（受 `TARRAY_RANGED_FOR_CHECKS` 门控，`#else` 分支退化为裸指针），故 P0 **不得**表述为"引擎有 ensure 检查"，而应表述为"`Clear()`/`Empty()` 释放底层分配后迭代器指针悬垂"——**定级不变**。
- **出处**：`01-UnrealCSharp运行时/07b-委托Handler与OptionalHelper.md:590`

#### Top 6 `F-INT1-001` — `01-UnrealCSharp运行时/10-Interop注册-对象与反射类.md`
- **P0（崩溃）/ 活跃 / 确认**
- **现象**：`TScriptInterface.GetObject` 对 `GetMulti` 返回值**不判空即解引用**。
- **根因链**：`TScriptInterface.cs:46` → `TScriptInterfaceImplementation.cs:36`（注册点 `:76`）→ `FRegisterScriptInterface.cpp:64-65 GetMulti<...>` → 未命中确定返回 `nullptr`（`FMultiRegistry.inl:24-26`、`FCSharpEnvironment.inl:224-226`）→ `:67 Multi->GetObject()` 无判空。
- **影响面**：`Script/Game/**` 中 `TScriptInterface<` 命中 **83 处**；且 `UnRegister` 走 `AsyncTask`（`:53-60`）而读路径同步（`:64-67`）⇒ **"C# 调 UnRegister 后立刻 GetObject"在游戏线程上几乎必崩**；C# 侧 `TScriptInterfaceImplementation.cs:38` 的保护救不了。
- **最小修复**：`FRegisterScriptInterface.cpp:67` 改 `return Multi != nullptr ? FCSharpEnvironment::GetEnvironment().Bind(Multi->GetObject()) : InvalidManagedHandle;`
- **出处**：`01-UnrealCSharp运行时/10-Interop注册-对象与反射类.md:182`

#### Top 7 `F-RM-001` — `02-UnrealCSharpCore核心/02-反射模型（类字段方法参数属性）.md`
- **P0（静默数据损坏）/ 活跃 / 确认**（该条 `严重度` 字段曾写 P1、标题与 §0 写 P0，已统一为 **P0**）
- **现象**：`Methods` 去重键只用 **(Name, ParamCount)** —— 同名同参数个数的重载被**静默覆盖**，C++→C# 调用会**调到错误的方法并用错误类型读取实参**。
- **根因链**：C# 生产端不去重（`Script/UE/CoreUObject/Utils.cs:443-450`）→ 键构造 `FClassReflection.cpp:369-377` → 引擎同键**覆盖**（`Map.h:408` → `Map.h:447-453` → `Set.h:573-603`，旧裸指针**不 delete**）→ 查询只用 (名, 参数个数)（`TMethodHelper.inl:59`）→ 按被调方法形参类型解箱（`MethodBridge.cs:49`/`:65-67`）→ 异常被吞成 `return 0`（`MethodBridge.cs:125-137`）。
- **影响面**：可复现的静默数据损坏 + 必有泄漏；泄漏对象持有 `HandleData.cs:34` 计数使托管侧永不 GC。属性/字段同理同键覆盖（`:248`/`:282`）。（`.ctor` 一路经核实**不构成**触发路径。）
- **最小修复**：`FClassReflection.cpp:369-377` 改多值键，或覆盖前 `ensure` + `delete` 旧值。
- **出处**：`02-UnrealCSharpCore核心/02-反射模型（类字段方法参数属性）.md:191`

#### Top 8 `F-DYN-002` — `02-UnrealCSharpCore核心/03-动态类型生成.md`
- **P0（编辑器挂死）/ 活跃 / 确认**
- **现象**：依赖图第二阶段**没有环检测** —— 自引用 / 互相引用的动态类会让生成过程**死循环**。
- **根因链**：`FDynamicDependencyGraph.cpp:126-135`（依赖 Pending）→ `:144-151`（Pending + Enqueue）→ 第二阶段 `:157 while(Dequeue)` → `:195-200` **无条件重入队**（`:197`）→ `:210` 只有空 `// @TODO`。入队点全插件仅 `:148`/`:197`。
- **影响面**：**主线程死循环，编辑器无响应且无崩溃报告**（无法从日志定位）；自环与互环两种输入**必然**命中；附带 `:208-211` 静默丢弃致类型永不生成。
- **最小修复**：`FDynamicDependencyGraph.h` 的 `FNode` 加 `int32 PendingPasses`，在 `:197` 计数超过 `NodeArray.Num()` 时告警并强制 `Generator()/Completed()`。
- **出处**：`02-UnrealCSharpCore核心/03-动态类型生成.md:178`

#### Top 9 `F-BR-001` — `02-UnrealCSharpCore核心/06-类型桥接编码设置与引擎监听.md`
- **P0（空指针写）/ 活跃 / 部分确认**（机制/后果/类别/P0 成立；**原文"7 调用点 / 6 处解引用"实测为 18 调用点 / 17 处未判空**）
- **现象**：`FTypeBridge::Factory` 以 `nullptr` 表示"不支持"，但 **18 处调用点中 17 处直接解引用** → 空指针写。
- **根因链**：`FTypeBridge.inl:193 default: return nullptr;`（另 `:492`）→ `FDynamicGeneratorCore.cpp:904-912` 未判空即 `AddCppProperty`（`:910`）→ 引擎 `Class.cpp:726-730` 的 `Property->Next = ChildProperties;` 无判空。**唯一判空点是 `FDynamicGeneratorCore.cpp:942`。**
- **影响面**：对地址 `0 + offsetof(FField, Next)` 写指针 → 访问违例；`FCSharpBind.inl:95`、`FRegisterOptional.cpp:28` 是"调用空指针的成员函数"；可触达类型 = `TOptional`/`TFieldPath`/委托/**任何未注册的 C# 类型**（`FTypeBridge.cpp:11-165` 无分支）。
- **最小修复**：`FDynamicGeneratorCore.cpp:904` 与 `:957` 照抄 `:942` 的 `if (CppProperty == nullptr) { continue; }`；另在 `FTypeBridge.inl:193` 加 `UE_LOG`。
- **出处**：`02-UnrealCSharpCore核心/06-类型桥接编码设置与引擎监听.md:110`

#### Top 10 ★ `F-GENC-001` — `03-代码生成器/01-代码生成框架与模块入口.md`
- **P0（崩溃）/ 活跃 / 确认**
- **现象**：`LoadFileToArray` **反序列化失败时对空 `JsonObject` 解引用**（崩溃）。
- **根因链**：`FGeneratorCore.cpp:1016` 无条件调用 → `FUnrealCSharpFunctionLibrary.cpp:1189` 的 `JsonObject` 默认空 → `:1193` **Deserialize 的返回值被丢弃** → `:1195 for (... : JsonObject->Values)`；另一调用点 `FDynamicGeneratorCore.cpp:21`；入口 `UnrealCSharpEditor.cpp:343` 早于 `:345`。
- **影响面**：触发条件只差"**JSON 内容非法**"这一个条件 —— 两个 JSON 文件**实测均存在**（3835 / 841 字节）⇒ **每次代码生成都会执行到该行**；写一半被强杀即命中；崩溃发生在生成流程最开始几步，**用户无法通过重开编辑器绕过**。（与 `F-ABI-033`、`F-FL-002` 同源。）
- **最小修复**：`FUnrealCSharpFunctionLibrary.cpp:1193` 改为 `if (!FJsonSerializer::Deserialize(JsonReader, JsonObject) || !JsonObject.IsValid()) { UE_LOG(Error, ...); return Result; }`
- **出处**：`03-代码生成器/01-代码生成框架与模块入口.md:182`

#### Top 11 `F-ED2-001` — `04-编辑器模块/02-工具栏监听器进度对话框与模块入口.md`
- **P0（UAF）/ 活跃 / 确认**（与 `F-LIFE-001` 同机制，见 §3 缺陷族 —— **聚合计数须合并**）
- **现象**：`FEditorListener` 对 AssetRegistry 的 **5 个 `AddRaw(this,…)` 委托句柄全部丢弃**且从不反注册 → 监听器销毁后回调悬空 `this`。
- **根因链**：`FEditorListener.cpp:79`（构造，句柄未存成员）+ `:344/346/348/350`（`OnFilesLoaded` 内再挂 4 个）→ 析构 `:101-179` 处理了 12 个句柄，**这 5 个 Remove 命中 0**；宿主为值成员（`UnrealCSharpEditor.h:55`）；AssetRegistry 是**引擎级单例**，生存期长于编辑器模块。
- **影响面**：5 处必然的 use-after-free；且 `OnFilesLoaded` 每次加载完成都广播 ⇒ 4 个委托被**重复 AddRaw**（N 次代码生成 + N 次编译请求）。对照 `ClassCollector.cpp:53-63 ↔ :99-117` 是同类正确写法。
- **最小修复**：`FEditorListener.cpp:79`/`:344-350` 保存 5 个 `FDelegateHandle` 成员，并在 `~FEditorListener()`（`:101-179`）按 `IsValid` 逐一 `Remove`。
- **出处**：`04-编辑器模块/02-工具栏监听器进度对话框与模块入口.md:125`

#### Top 12 `F-ED2-017` — `04-编辑器模块/02-工具栏监听器进度对话框与模块入口.md`
- **P0（UAF）/ 活跃 / 确认**
- **现象**：`FUnrealCSharpBlueprintToolBar` 把捕获 `this` 的扩展器委托挂到蓝图编辑器上却**永不反注册** → 工具栏销毁后每次打开蓝图编辑器都调用到已释放对象。
- **根因链**：`UnrealCSharpBlueprintToolBar.cpp:25-32`（`:26 CreateLambda` 捕获 `this` 且 `Add` 返回的 `FDelegateHandle` **被丢弃**）→ `Deinitialize()`（`:42-48`）只 Remove 了 `OnEndGeneratorDelegateHandle`（来自 `:38-39`）→ 全 `Source/` grep `GetExtenderDelegates` 只有 1 处 `.Add`、`Remove` **命中 0**；`UnrealCSharpEditor.cpp:188 UnregisterOwner` 只管 UToolMenus 条目。
- **影响面**：**必然 UAF**（每次打开任意蓝图都会调用 `GenerateBlueprintExtender`）；每次 `Initialize` 累积一次 `Add` ⇒ 热重载后出现**两个按钮，其中一个是悬空的**。
- **最小修复**：`UnrealCSharpBlueprintToolBar.cpp:25-32` 保存句柄为成员，并在 `:42-48 Deinitialize()` 中 `GetExtenderDelegates().Remove(handle)`。
- **出处**：`04-编辑器模块/02-工具栏监听器进度对话框与模块入口.md:323`

#### Top 13 `F-CS2A-002` — `06-CSharp运行时与脚本/02a-容器与指针包装（资源所有权）.md`
- **P0（崩溃）/ 活跃 / 部分确认**（`Find`/`FindKey`/索引器 getter 成立；原文并列的"**setter 同样崩溃**"已证伪）
- **现象**：`TMap` 键不存在时 `Find` / `FindKey` / 索引器 getter **空指针解引用崩溃**。
- **根因链**：`TMap.cs:192`/`:135`/`:272` → `FMapHelper.cpp:171`（`Get` 键不存在返回 `nullptr`）/ `:144`（`FindKey`）/ `:147-150`（`Find` 转发 `Get`）→ `FRegisterMap.cpp:91-92`/`:102-103`/`:124-125` 未判空 → `TPrimitivePropertyDescriptor.inl:18 CopySingleValue(Dest, Src=nullptr)`。
- **影响面**：本模块**触发概率最高**的崩溃路径 —— 对应 .NET `Dictionary[key]` 抛 `KeyNotFoundException` 的常见用户错误，迁移到 `TMap` 却**无编译期提示**；第二种表现是句柄失效时输出缓冲未写 ⇒ 读出未初始化栈内容。只有 `Contains`（`TMap.cs:233-254`）不会崩。
- **最小修复**：`FRegisterMap.cpp:124`（`GetImplementation`）加判空并改协议返回成败（`Find`/`FindKey` 同）。
- **出处**：`06-CSharp运行时与脚本/02a-容器与指针包装（资源所有权）.md:1150`

#### Top 14 `F-CS2A-017` — `06-CSharp运行时与脚本/02a-容器与指针包装（资源所有权）.md`
- **P0（崩溃）/ 活跃 / 确认**（原文"四类按同构推定"的保留意见已全部消除，六条链路实读）
- **现象**：六个指针包装类的**无参构造函数为空**，`Get()` 对未注册句柄解引用 ⇒ **默认构造后取值必崩**。
- **根因链**：`TWeakObjectPtr.cs:8-10`（空构造不注册）→ `HandleData.cs:80-93`（`:88` 未 `Alloc` 返回 **0**，而句柄 0 永不被分配）→ `TWeakObjectPtr.cs:45 Get()` 把 0 送进 native → `FRegisterWeakObjectPtr.cpp:47-53`（`:52 Multi->Get()` 不判空）→ `TWeakObjectPtr::Get()` 非虚成员，`this == nullptr` 即访问违例。
- **影响面**：不止"无参构造" —— **finalizer 已跑 / `AsyncTask` 未排空 / 脚本域已关闭**三种失效场景下 `GetMulti` 都返回 `nullptr`，而这些代码**全部**空指针解引用。（`TArray`/`TMap`/`TSet` 的无参构造有注册，不在其中。）
- **最小修复**：六个 native 侧判空返回 `InvalidManagedHandle`：`FRegisterWeakObjectPtr.cpp:47-53`、`FRegisterSubclassOf.cpp:46-52`、`FRegisterLazyObjectPtr.cpp:47`、`FRegisterSoftObjectPtr.cpp:47`、`FRegisterSoftClassPtr.cpp:47`、`FRegisterScriptInterface.cpp:62-68`。
- **出处**：`06-CSharp运行时与脚本/02a-容器与指针包装（资源所有权）.md:1370`

#### Top 15 `F-CS2B-001` — `06-CSharp运行时与脚本/02b-对象字符串与工具类型包装.md`
- **P0（进程崩溃）/ 活跃 / 确认**
- **现象**：未注册句柄的 `FString.ToString()` 触发**原生空指针解引用崩溃**（默认构造即触发）。
- **根因链**：`FString.cs:8-10`（默认构造不注册）→ `FString.cs:43 ToString()` 传 `HandleData.GetHandle(this)`（= 0）→ `FRegisterString.cpp:44 GetString<FString>` → `FStringRegistry.inl:23-27` 未命中返回 `nullptr` → `:46 TCHAR_TO_UTF8(**String)` **无判空**；对照 `:23-29 IdenticalImplementation` 却是**双重判空**。
- **影响面**：用户一行代码即可触发；`HandleData.Clear()`（`HandleData.cs:129-145`）清表后**所有**存活包装对象的句柄查询都返回 0 ⇒ 不限于默认构造；5 个字符串注册器中 **4 个**（FString/FName/FText/FUtf8String）都崩，只有 `FRegisterAnsiString.cpp:49-53` 正确判空。
- **最小修复**：`FRegisterString.cpp:42-47` 加 `String != nullptr ? ... : InvalidManagedHandle`，另 4 个 `ToStringImplementation` 同改（清单见该报告 `:410-422`）。
- **出处**：`06-CSharp运行时与脚本/02b-对象字符串与工具类型包装.md:344`

#### Top 16 `F-LIFE-003` — `08-专项审计/02b-UObject生命周期与绑定配对审计.md`
- **P0（空指针解引用）/ 活跃 / 确认**
- **现象**：`FDelegatePropertyDescriptor::Set` / `FMulticastDelegatePropertyDescriptor::Set` 未判空即解引用 `SrcDelegateHelper`。
- **根因链**：C# 属性 setter → `FDelegatePropertyDescriptor.cpp:20-31`（`:24` 取 helper 无判空 → `:30 BindUFunction(SrcDelegateHelper->GetUObject(), ...)`）；`FMulticastDelegatePropertyDescriptor.cpp:20-37`（`:24-25` → `:33-34` 同样）；`nullptr` 来源 `FDelegateRegistry.inl:19-25`；异步注销 `FRegisterDelegate.cpp:21-28`（`AsyncTask` 中 `delete FDelegateHelper` + `GCHandle_Free`）。
- **影响面**：插件里**唯一一处**"注册表查询结果直接解引用"未判空（同批 descriptor 的 `NewRef` 在 `:37` 用了 `IManagedHandleIsValid`）；触发 = 托管侧把已 `UnRegister` / 从未 `Register` 的句柄写回 delegate 属性。
- **最小修复**：两个 `Set` 都加 `if (SrcDelegateHelper == nullptr) { return; }`。
- **出处**：`08-专项审计/02b-UObject生命周期与绑定配对审计.md:396`

### 1.2 Top P1 / P2：后果确定（4 条，含 2 条已独立复现/验证）

#### Top 17 ★ `F-CS2A-023` — `06-CSharp运行时与脚本/02a-容器与指针包装（资源所有权）.md`
- **P1（功能错误 / 泄漏）/ 活跃 / 确认**（原文自述"`RemoveContainerReference` 实现未读取"，补齐链路后结论与原文乐观假设**相反**）
- **现象**：**闭环自锁** —— 容器 / 包装对象的唯一释放入口由 C# finalizer 触发，但对象被 `HandleData` 的**强 `GCHandle` 永久 root** ⇒ **finalizer 永不执行** ⇒ **native 与托管资源在进程生命周期内都不释放**。
- **根因链（闭环，逐跳）**：`HandleData.cs:39 GCHandle.Alloc(InObject, GCHandleType.Normal)`（**强**句柄，`:27-44`）→ `:11` 的 static `Handles` 持根使对象**永不不可达** → 唯一释放写点 `FContainerRegistry.inl:76 FDomain::GCHandle_Free`（在 `:58-82 RemoveReference` 内，`:72 delete`）→ 只被 `UnRegisterImplementation` 调用（如 `FRegisterArray.cpp:34-41`）→ **只由 C# finalizer 触发**（`TArray.cs:17 ~TArray()`）→ `~TArray()` **永不执行**。第三条释放路径**不存在**：`AddReference` 走 3 参数重载 `FContainerRegistry.inl:35-41`（`:38` 只登记、**不建 `TContainerReference`、无 owner**），对比 `:43-56` 的 6 参数重载；`FDomain.cpp:94-98 → FScriptDomainImpl.inl:337-348 → HandleData.cs:46-78` 这条链在 C# 侧**0 调用点**。
- **影响面**：native 侧 `FArrayHelper`+`FScriptArray` 及元素缓冲（`FArrayHelper.cpp:19` new、`:36` 释放永不执行）、`FMapHelper`/`FSetHelper`（`FSetHelper.cpp:18`/`:43`）、`FOptionalHelper`+`FMemory::Malloc`（`FOptionalHelper.cpp:21`/`:40`）、六个 `new T*Ptr<UObject>`（如 `FRegisterWeakObjectPtr.cpp:17`）**全部进程内累积**；托管侧包装对象**自身也永不回收**；同构于 `FMultiRegistry`/`FOptionalRegistry` ⇒ **本模块全部 11 个类型共用**。危害高于 `F-CS2A-005`（后者只谈"无 IDisposable"）。
- **最小修复**（报告给三方案，**原文未逐条标注 `文件:行`，只给代码片段 —— 如实标注**）：A `HandleData.cs:39` 改 `GCHandleType.Weak`；B 把 `FContainerRegistry.inl:76` 的释放提前到 native 引用计数归零，而非只挂 finalizer；C 增加 `Dispose()`。
- **★ 关联独立发现**：本条的"强句柄 ⇒ 终结器永不执行"闭环在**另外 11 份报告里被独立命中**（`06-…/02b F-CS2B-002`、`08-…/02 F-LEAK-001`、`06-…/01 F-CS1-002`、`01-…/02 F-REG-007`、`01-…/03 F-REG2-002`、`01-…/10b F-INT2-011`、`01-…/07b F-DEL-008`、`08-…/02 F-LEAK-002` …）—— **同一病根被多处独立命中**，属相互印证（见 §3 族 3）。
- **实机边界**：该条**未实机证实**（属"必须实机验证后才能当结论用"的项，`06-…/02a` 自标"未实机证实"）。
- **出处**：`06-CSharp运行时与脚本/02a-容器与指针包装（资源所有权）.md:3129`

#### Top 18 `F-INT2-012` — `01-UnrealCSharp运行时/10b-Interop注册-容器字符串与对象指针.md`
- **P1（功能错误：输出乱码 + 越界读）/ 活跃 / 部分确认 + 上调**（级别变动：**P3 → P1**；原有"低置信度"部分已由引擎源码确证为真 bug）
- **现象**：`TCHAR_TO_UTF8` 是**宏**且硬转 `const TCHAR*`，而 `FAnsiString`/`FUtf8String` 的 `operator*` 返回 **1 字节窄指针** ⇒ **窄串被当 UTF-16 解释**。
- **根因链**：`AnsiString.h:8-13` + `UnrealString.h.inl:369-372` → `operator*` 返回 `const ANSICHAR*`（`FUtf8String` 同理，`Utf8String.h:10-11`）→ `FRegisterAnsiString.cpp:52 TCHAR_TO_UTF8(*FAnsiString(*AnsiString))` → `StringConv.h:1021` 宏展开 `(ANSICHAR*)FTCHARToUTF8((const TCHAR*)str).Get()`，而 `FTCHARToUTF8_Convert::FromType = TCHAR`（`StringConv.h:226-233`）⇒ **乱码 + 按 TCHAR 求长度越界读**；`FRegisterUtf8String.cpp:50` 完全同型。
- **影响面**：本项目五个平台 `TCHAR` **都是 2 字节**（`AndroidPlatform.h:47`、`IOSPlatform.h:29`、`UnixPlatform.h:50`、`MacPlatform.h:55`、`HAL/Platform.h:280-282`）⇒ **不是"仅某平台"的问题，全部目标平台都错**，每次 `ToString()` 都触发。入口方向（`FRegisterAnsiString.cpp:18 UTF8_TO_TCHAR`）**正确**，**错只在出口**；另 `FAnsiString` 的 C# API 本身有损（非 ANSI 字符静默变 `?`）。
- **最小修复**：`FRegisterAnsiString.cpp:52` 去掉中间拷贝并显式用正确转换（`ANSI_TO_TCHAR` 再 `TCHAR_TO_UTF8`）；`FRegisterUtf8String.cpp:50` 改 UTF-8 直通 `NewString(reinterpret_cast<const char*>(*Utf8String))`。
- **出处**：`01-UnrealCSharp运行时/10b-Interop注册-容器字符串与对象指针.md:1814`

#### Top 19 `F-CMP-002` — `05-编译器与跨版本/01-编译器CSharp编译与跨版本宏.md`
- **P1（挂死）/ 活跃 / 确认**（**原文"5 个调用点"计数错误，实为 6 个**，已就地修正）
- **现象**：`SyncProcess` **完全没有超时保护** —— `dotnet` 一旦卡住就是**永久挂起**，无取消、无静默检测、无可中断点。
- **根因链**：`FUnrealCSharpFunctionLibrary.cpp:1520-1592` 全函数只有一个循环条件（`:1569 while (ProcessHandle.IsValid() && FPlatformProcess::IsProcRunning(ProcessHandle))`，体内 `Sleep(0.01f)` + `ReadOutput()`），**无任何超时变量/静默检测/外部中断点**；6 个调用点全部是同步等待：`FCSharpCompilerRunnable.cpp:291`（Interop 编译）、`:413`（游戏程序集编译）、`FCodeAnalysis.cpp:34/49/86`、`UnrealCSharpEditorSetting.cpp:152`。
- **影响面**：`dotnet build` 会（a）联网还原 NuGet 包、（b）等待 `VBCSCompiler`/MSBuild 节点锁、（c）被杀毒软件长时间静默 —— 任一成立则循环永不退出：**编译线程永久占用、`bIsCompiling` 永远为真** ⇒ `FEditorListener.cpp:718` 的 `while (FCSharpCompiler::Get().IsCompiling())` **永久自旋**，编辑器被锁在模态进度窗里（且因 `F-CMP-001` 连关闭都不干净）；在**游戏线程**调 `GetDotNetPathArray()`（`UnrealCSharpEditorSetting.cpp:152`）时同样冻结。
- **最小修复**：在 `SyncProcess` 加两个阈值 —— 总超时（如 600s，ini 可配）与"无输出静默超时"（如 120s）；超时后 `TerminateProc` 并以 `ReturnCode = -1` 回调，把原因拼进 `InResult`。可复用 `FPlatformTime::Seconds()`。
- **验证方式（报告给出）**：把 `GetDotNet()` 换成一个假可执行文件（如 `cmd /c pause`），观察调用方是否永久阻塞。
- **出处**：`05-编译器与跨版本/01-编译器CSharp编译与跨版本宏.md:321`

#### Top 20 `F-ENV-001` — `01-UnrealCSharp运行时/01-Environment与绑定注册表.md`
- **P2 / 活跃 / 部分确认**（**机制已被引擎源码收窄**：原"`BindAction` 返回 `TArray` 元素引用、扩容即令此前地址失效"一条**被证伪**；"注册表以裸地址为键且无任何有效性/代次校验"一条**确认成立**）
- **现象**：`FBindingRegistry` **以裸地址为键且无任何生命周期/有效性校验**，调用方把"堆上绑定对象地址 / **栈上实参地址**"存了进去 ⇒ 悬垂地址解引用（UAF 读写）。
- **根因链（收窄后的真实来源）**：(a) 引擎侧**移除/清空绑定**后注册表条目残留 —— 引擎 `EnhancedInputComponent.cpp:36-98` 销毁 `TUniquePtr` 后**条目无任何回收路径**，此后任何 `GetBinding<FEnhancedInputActionEventBinding>(handle)` 都返回已释放地址；(b) **栈上实参地址入表** —— `TArgument.inl:71-72 Set()` 传 `const_cast<std::decay_t<Type>*>(&Value)`，而 `TFunctionHelper.inl:21` 的 `std::tuple<TArgument<Args,Args>...> Argument(...)` 是**函数局部变量** ⇒ 栈地址进表。两处都进 `FBindingRegistry.inl:15-25`（`:17` 地址→句柄、`:19` 存进 wrapper），而 `FBindingRegistry.cpp:40-45 GetObject` **只做指针相等比较**、不校验地址归属；登记点 `FRegisterEnhancedInputComponent.cpp:194-196`、解引用点 `:210-213`。
- **影响面**：两张表都是"地址⇒句柄"且只比较指针相等，**无法区分"地址仍有效"与"已回收/复用"**；同模式还出现于"按值参数的 `ref/out` 编组"路径（`TPropertyValue.inl:174-192`、`TArgument.inl:71-73`）。**修不动单点** —— 表结构里没有 `IsValid()`/代次字段，必须改数据结构。级别维持 **P2**（触发路径收窄后不再按"若触发则等同 P0"记账；但机制本身是 UAF 读写，一旦命中仍属数据损坏级）。
- **最小修复**：`FBindingRegistry.h:38-51` 的 `FBindingAddress` 增加 `TWeakObjectPtr<const UObject> Owner`，并在 `FBindingRegistry.inl:6-12 GetBinding`（及 `FBindingRegistry.h:77-92`、`FBindingRegistry.cpp:47-75`）按 Owner 有效性校验（报告方案 A，最小改动、推荐）。
- **出处**：`01-UnrealCSharp运行时/01-Environment与绑定注册表.md:141`

### 1.3 Top 之外的"次高影响"补充（不进 Top 20，但立项时不应遗漏）

| 编号（带报告路径） | 级别 / 可达性 | 一句话 |
|---|---|---|
| `01-…/10b F-INT2-011` | P1 / 活跃 | C++ 现场新建并直接返回给 C# 的托管句柄**从不释放** ⇒ `HandleData` 强 `GCHandle` 与字典条目**无界泄漏**（族 3） |
| `01-…/07b F-DEL-004` | P1 / 活跃 | 单播 `DelegateWrapper` **没有任何解绑点**，`UnBind`/`Clear`/`Deinitialize` 都不清理 C# 目标 |
| `02-…/03 F-DYN-006` | P1 / 活跃 | 元数据属性类被缓存进**函数级 `static`**，跨 C# 热重载持有已 `delete` 的 `FClassReflection*` |
| `02-…/03 F-DYN-005` | P1 / 活跃 | `[Replicated]` **读的是 `ReplicatedUsing` 的特性值** ⇒ 复制生命周期条件**永远被写成 `COND_None`** |
| `03-…/02 F-GEN2-004` | P3 / 部分确认 / 活跃 | 生成文件的 `using` 顺序由 `TSet<FString>` 遍历顺序决定（**未排序、输出不可复现**）：抽 400 份真实产物 **331 份非字典序**（机制证实并加强；"跨机器/跨时间不一致"的表述**已证伪**，顺序由哈希+容量+插入/删除历史决定） |
| `05-…/01 F-CMP-003` | P2 / 活跃 | 同步编译路径**绕过队列与 `IsCompiling()` 守卫**，可同时跑两个 `dotnet build` 写同一输出目录（P1→**P2**） |
| `08-…/03 F-CONC-*` | P1×8 / 活跃 | 并发族：8 条 P1（`08-…/03` 全报告 P1 = 8、活跃 19）—— 详见该报告的线程模型章节 |
| `08-…/04 F-ABI-020` | P2 / 确认 / **潜伏** | 插件**自己**的 `DirectoryInfo.GetFiles("*.Build.cs")` 大小写敏感（`UnrealCSharpCore.build.cs`/`CrossVersion.build.cs` 是小写）⇒ Linux/macOS 匹配不到；UBT **自身不会**漏模块（`F-ABI-021` 前提已证伪 → P3） |

> ⚠️ **`F-DOM-003`（`02-…/04`，P0）虽为 P0，但可达性为「潜伏」**（本工程 ini 与构建产物一致故不触发；**没有 ini 的工程必然触发**），按 §0.4 的排序键 2 未进 Top 20，但**理由已更换**：原文场景"LinuxArm64 回退 CoreCLR 而构建 LeanCLR"**已证伪**（`Build.cs:263-267` 构建期同样回退 `"CoreCLR"`）；新理由更硬 —— 插件**不含任何 `Config` 文件**（glob `**/Config/*` 命中 **0**）⇒ 无 ini 工程运行期默认 `Mono`（枚举零值 + CDO 内存被引擎清零，`UObjectGlobals.cpp:3845`）、构建期默认 `CoreCLR` ⇒ **必然** `Create() == nullptr` → `FDomain.cpp:31` 空指针崩溃**且无日志**。修法：`FDomain.cpp:31` 前加判空 + `FScriptDomainFactory.cpp:44` 加日志/回退。

---

## §2 逐报告统计表（47 份 × 发现数 / 级别分布 / 引文行号吻合率）

**取数**：前 5 列（发现数、级别、复核结论、可达性）来自逐条汇总的逐份明细；**引文行号吻合率**来自各份的 `lines_match / lines_sampled`（**不是推算**）。
**列含义**：`N` = 发现数；`P0/P1/P2/P3/撤销` = 级别分布；`确/部/证/无/非` = 复核结论（确认/部分确认/证伪/无法验证/非标准书写）；`活/潜/不/无` = 可达性（活跃/潜伏/不可达/无字段）。

### 2.1 00 总览 与 09 问题清单（5 份）

| 报告 | N | P0/P1/P2/P3/撤销 | 确/部/证/无/非 | 活/潜/不/无 | 引文行号吻合率 |
|---|---|---|---|---|---|
| `00-总览/01-架构与数据流总览.md` | 10 | 0/0/1/7/2 | 6/2/2/0/0 | 6/2/2/0 | **16/20 = 80.0%** ⚠️ |
| `00-总览/02-执行摘要与Top发现.md`（本文件） | 0 | — | — | — | 不适用（纯聚合视图，0 条标题式发现） |
| `00-总览/03-修复优先级评估.md` | 0 | — | — | — | 不适用（纯聚合视图） |
| `09-问题清单/00-推荐优先修复清单.md` | 0 | — | — | — | 不适用（纯聚合视图） |
| `09-问题清单/01-全局问题总表.md` | 0 | — | — | — | 不适用（派生视图） |

### 2.2 01 UnrealCSharp 运行时（14 份）

| 报告 | N | P0/P1/P2/P3/撤销 | 确/部/证/无/非 | 活/潜/不/无 | 引文行号吻合率 |
|---|---|---|---|---|---|
| `01-…/01-Environment与绑定注册表.md` | 21 | 0/1/7/13/0 | 15/6/0/0/0 | 20/0/1/0 | 12/12 = 100% |
| `01-…/02-对象引用结构体注册表.md` | 18 | 0/6/2/10/0 | 10/8/0/0/0 | 16/2/0/0 | 15/15 = 100% |
| `01-…/03-容器委托字符串注册表与模块入口.md` | 14 | 0/2/4/6/2 | 6/6/2/0/0 | 6/5/3/0 | **37/40 = 92.5%** |
| `01-…/04-属性描述符-基类与基本类型.md` | 7 | 0/1/1/5/0 | 3/4/0/0/0 | 6/0/1/0 | 10/10 = 100% |
| `01-…/04b-属性描述符-字符串枚举结构体Optional.md` | 16 | **2**/2/4/7/1 | 12/3/1/0/0 | 14/0/2/0 | 🔴 **无数据**（`lines_sampled: 0`） |
| `01-…/05-属性描述符-对象与委托类型.md` | 15 | **2**/4/3/6/0 | 7/8/0/0/0 | 14/0/1/0 | 12/12 = 100% |
| `01-…/06-函数与类描述符.md` | 16 | 0/3/5/8/0 | 12/4/0/0/0 | 8/8/0/0 | 16/16 = 100% |
| `01-…/07-容器Helper.md` | 30 | **1**/10/9/6/4 | 19/9/2/0/0 | 23/3/2/0 | 🔴 **无数据**（**风险最高的一份**，见 §2.7） |
| `01-…/07b-委托Handler与OptionalHelper.md` | 18 | **2**/4/4/8/0 | 7/11/0/0/0 | 15/2/1/0 | 12/12 = 100% |
| `01-…/08-绑定层公开头文件（模板元编程）.md` | 12 | **1**/3/3/5/0 | 6/6/0/0/0 | 8/4/0/0 | **50/54 = 92.6%** |
| `01-…/09-绑定宏与CoreMacro对照.md` | 13 | 0/0/3/10/0 | 9/4/0/0/0 | 7/5/1/0 | 20/20 = 100% |
| `01-…/10-Interop注册-对象与反射类.md` | 13 | **2**/0/5/4/2 | 6/4/2/0/1 | 11/0/2/0 | 14/14 = 100% |
| `01-…/10b-Interop注册-容器字符串与对象指针.md` | 16 | **1**/4/1/8/2 | 8/7/1/0/0 | 13/0/2/0 | 16/16 = 100% |
| `01-…/11-Interop注册-数学值类型与引擎类.md` | 28 | **1**/7/7/13/0 | 15/13/0/0/0 | 28/0/0/0 | 12/12 = 100% |

> 本模块 P0 合计 **10** 条：`04b` 2、`05` 2、`07` 1、`07b` 2、`08` 1、`10` 2、`10b` 1、`11` 1（`11` 与 `10b` 各有一条 `F-INT2-001`，**是两条不同的发现**）。

### 2.3 02 UnrealCSharpCore 核心（7 份）

| 报告 | N | P0/P1/P2/P3/撤销 | 确/部/证/无/非 | 活/潜/不/无 | 引文行号吻合率 |
|---|---|---|---|---|---|
| `02-…/01-反射注册表FReflectionRegistry.md` | 9 | 0/1/4/4/0 | 7/2/0/0/0 | 9/0/0/0 | **19/20 = 95.0%** |
| `02-…/02-反射模型（类字段方法参数属性）.md` | 15 | **1**/1/3/9/1 | 9/5/1/0/0 | 8/2/5/0 | 20/20 = 100% |
| `02-…/03-动态类型生成.md` | 21 | **2**/5/4/9/1 | 12/8/1/0/0 | 12/6/2/0 | 12/12 = 100% |
| `02-…/04-脚本域抽象与三大后端.md` | 28 | **1**/3/7/16/1 | 14/13/1/0/0 | 12/14/2/0 | 21/21 = 100% |
| `02-…/05-绑定注册与类型信息.md` | 21 | 0/0/10/11/0 | 12/9/0/0/0 | 18/3/0/0 | 20/20 = 100% |
| `02-…/06-类型桥接编码设置与引擎监听.md` | 21 | **1**/1/5/13/1 | 9/11/1/0/0 | 13/8/0/0 | 🔴 **无数据** |
| `02-…/07-函数库宏与模板Trait.md` | 27 | 0/1/14/12/0 | 23/4/0/0/0 | 16/10/1/0 | 20/20 = 100% |

> ⚠️ `02-…/07` 的 `### P0` **分节标题存在**，但其中唯一条目 `F-FL-001` 的 `严重度` 字段是 **P1** ⇒ **逐条口径下本报告 P0 = 0**（典型的统计陷阱）。

### 2.4 03 代码生成器（4 份）

| 报告 | N | P0/P1/P2/P3/撤销 | 确/部/证/无/非 | 活/潜/不/无 | 引文行号吻合率 |
|---|---|---|---|---|---|
| `03-…/01-代码生成框架与模块入口.md` | 14 | **1**/0/2/11/0 | 7/7/0/0/0 | 12/1/1/0 | 12/12 = 100% |
| `03-…/01b-类生成器与绑定类生成器.md` | 29 | 0/2/3/21/3 | 10/16/3/0/0 | 26/1/2/0 | 20/20 = 100% |
| `03-…/02-委托结构体枚举与GameplayTag生成器.md` | 16 | 0/2/5/9/0 | 10/6/0/0/0 | 13/1/2/0 | 40/40 = 100% |
| `03-…/03-代码分析Doxygen转换与解决方案生成.md` | 18 | 0/1/11/6/0 | 13/5/0/0/0 | 17/1/0/0 | 🔴 **无数据** |

> `03-…/01b` 是**级别分布与本报告头部自述差最大**的一份：头部/附录曾写 `P1=9/P2=13/P3=7`，**逐条实测 `P1=2/P2=3/P3=21/撤销=3`**。

### 2.5 04 编辑器 / 05 编译器 / 06 C# 运行时 / 07 构建（9 份）

| 报告 | N | P0/P1/P2/P3/撤销 | 确/部/证/无/非 | 活/潜/不/无 | 引文行号吻合率 |
|---|---|---|---|---|---|
| `04-…/01-内容浏览器与新建类向导.md` | 23 | 0/1/8/13/1 | 21/1/1/0/0 | 18/4/1/0 | 12/12 = 100% |
| `04-…/02-工具栏监听器进度对话框与模块入口.md` | 27 | **2**/7/4/13/1 | 14/10/3/0/0 | 20/4/2/0 | **30/32 = 93.8%** |
| `05-…/01-编译器CSharp编译与跨版本宏.md` | 33 | 0/2/19/11/1 | 23/9/0/1/0 | 21/6/6/0 | **27/30 = 90.0%** |
| `06-…/01-Interop桥接与程序集加载.md` | 40 | 0/4/16/17/3 | 20/18/0/2/0 | 32/8/0/0 | 20/20 = 100% |
| `06-…/02a-容器与指针包装（资源所有权）.md` | 23 | **3**/9/5/6/0 | 11/11/0/0/1 | 21/1/1/0 | **17/20 = 85.0%** ⚠️最低 |
| `06-…/02b-对象字符串与工具类型包装.md` | 31 | **1**/9/4/17/0 | 17/14/0/0/0 | 26/3/2/0 | 32/32 = 100% |
| `06-…/03-引擎实现库Library（跨语言ABI契约）.md` | 10 | 0/4/3/2/1 | 5/4/0/0/1 | 8/2/0/0 | 31/31 = 100% |
| `06-…/04-Dynamic特性源生成器与模板工程.md` | 20 | 0/3/7/10/0 | 16/4/0/0/0 | 19/1/0/0 | 12/12 = 100% |
| `07-…/01-构建脚本模块依赖与插件配置.md` | 21 | 0/0/4/14/3 | 9/9/3/0/0 | 13/5/3/0 | **38/42 = 90.5%** |

> `04-…/02` 的 P0 **实为 2 条**：`F-ED2-004`、`F-ED2-018` 已证伪撤销，**这 2 条 P0 是 `F-ED2-001` 与 `F-ED2-017`**。

### 2.6 08 专项审计 与 08-07 裁决书（8 份）

| 报告 | N | P0/P1/P2/P3/撤销 | 确/部/证/无/非 | 活/潜/不/无 | 引文行号吻合率 |
|---|---|---|---|---|---|
| `08-…/01-全局死代码与死宏审计.md` | 11 | 0/0/8/3/0 | 6/5/0/0/0 | 5/5/1/0 | 12/12 = 100%（11 编号式 + 2 未编号 P3 子项 = 13，**口径需写明**） |
| `08-…/02-资源泄漏审计（内存句柄与容器）.md` | 15 | 0/3/5/7/0 | 5/10/0/0/0 | 12/3/0/0 | **18/20 = 90.0%** |
| `08-…/02b-UObject生命周期与绑定配对审计.md` | 19 | **2**/3/8/5/1 | 12/6/1/0/0 | 17/1/1/0 | 66/68 = 97.1% |
| `08-…/03-线程安全与并发审计.md` | 24 | 0/8/10/6/0 | 11/13/0/0/0 | 19/5/0/0 | 20/20 = 100% |
| `08-…/04-跨语言ABI与平台兼容审计.md` | 39 | 0/9/19/11/0 | 28/10/1/0/0 | 23/14/2/0 | 34/34 = 100% |
| `08-…/05-性能热点与优化清单.md` | 32 | 0/2/20/9/1 | 16/14/0/0/2 | 32/0/0/0 | 35/38 = 92.1% |

### 2.7 引文行号吻合度总账

| 项 | 数值 | 说明 |
|---|---|---|
| 文档总数 | **47** | 含发现的 **41** 份 |
| **有可用引文行号数据的报告** | **38 份** | 含已回填的 `08/02b`、`08/05` |
| 🔴 **有发现但无引文行号数据** | **4 份 / 85 条** | `01-…/04b`(16，`findings_total: 0`、`lines_sampled: 0`)、`01-…/07`(30)、`02-…/06`(21)、`03-…/03`(18) |
| **逐行吻合合计** | **870 条，吻合 839 条 = 96.44%** | 由 38 份报告的 `lines_sampled`/`lines_match` **直接相加**得出（非推算） |
| 对照：**全量逐条核对**引用一次命中率 | **96.48%（2661/2758）**，**行号越界 0 处** | 两条**完全独立**的方法（人工抽样回源码 vs 全量逐条核对）差 **0.04** 个百分点 |

> **抽样率 < 100% 的报告（10 份）**：`00-总览/01` 16/20、`01/03` 37/40、`01/08` 50/54、`02/01` 19/20、`04/02` 30/32、`05/01` 27/30、`06/02a` 17/20、`07/01` 38/42、`08/02` 18/20、`08/05` 35/38、`08/02b` 66/68 —— **所有不吻合项均已逐条列明并就地修正**（`04/02` 与 `07/01` 修正最多）。
> **⚠️ 4 份报告的引文行号数据缺失**，其中 **`01-…/07-容器Helper.md` 是全库最高风险面**：`FScriptArrayHelper` 的 `EmptyValues`/`RemoveValues`/`MoveAssign` **会**析构非 POD 元素 ⇒ **推翻了 `01/07` 若干 P0/P1 赖以成立的"引擎 Helper 不析构"前提**，而该报告**尚未逐条重判**。
---

## §3 跨模块缺陷族（**聚合计数时必须合并**）

> **为什么必须合并**：834 条发现是按"编号"计的，**不是按"病根"计的**。同一个病根在不同模块被各记一条，导致"Top 里看起来有 N 个 P0，实际只需修 1 处机制" —— 例如某条与执行摘要"族 3「只 Free 不析构」"**同一病根，不应作为独立 P1 重复计数**。
> **合并规则**：① 同族条目**只计一次严重度**（取族内最高级别）；② 统计"待修点"时按 `文件:行` 计数，**不按编号计数**；③ 若两份报告对**同一机制**各记一条 P0，P0 总数会虚高 —— 本文件已逐一标出。

### 族 1 🔴 「未命中返回 `nullptr` + 调用点零判空」⇒ 空指针解引用 / 空指针写（**本族覆盖 26 条 P0 中的 17 条**）

| 编号（带报告路径） | 级别 / 可达性 | 未命中的 `nullptr` 源 |
|---|---|---|
| `01-…/04b F-PROP2-004` | P0 / 活跃 | `FStringRegistry.inl:25-27`（`Identical` 族，6 处无守卫覆写） |
| `01-…/04b F-PROP2-005` | P0 / 活跃 | `FOptionalRegistry.cpp:41-46`（`:45`）而 `Get` 却查空（`:9`） |
| `01-…/05 F-PROP2-001` | P0 / 活跃 | `FTypeBridge::GetClass` → `FReflectionRegistry.cpp:923`/`:974`、`FTypeBridge.cpp:178`/`:421` |
| `01-…/05 F-PROP2-002` | P0 / 活跃 | `FMultiRegistry.inl:26` / `FCSharpEnvironment.inl:224` |
| `01-…/07 F-HLP-023` | **P0** / 活跃 | `FMapHelper.cpp:144`/`:171`/`:255`/`:262` → `FRegisterMap.cpp` 五处无判空 |
| `01-…/07 F-HLP-006` | P1 / 活跃 | `FArrayHelper` 越界返回 `nullptr` → `FRegisterArray::GetImplementation` 不判空 |
| `01-…/07b F-DEL-003` | **P0** / 活跃 | `FDelegateRegistry.inl:24` |
| `01-…/10 F-INT1-001` | **P0** / 活跃 | `FMultiRegistry.inl:24-26` |
| `01-…/10 F-INT1-002` | **P0** / 活跃 | `FStringRegistry.inl:25-27` |
| `01-…/10b F-INT2-003` | P1 / 活跃 | 5 个智能指针族的 `Get`/`LoadSynchronous` 对 `GetMulti` 不判空（范围内 7 处 / 全目录 8 处） |
| `01-…/10b F-INT2-013` | P1 / 活跃 | 4 个字符串 `ToString` 对 `GetString` 不判空（同族内不一致） |
| `01-…/11 F-INT2-001` | **P0** / 活跃 | `TMap::Find` 未命中 → `FRegisterDataTableFunctionLibrary.cpp:31` 直接 `*` |
| `01-…/01 F-ENV-001` | P2 / 活跃 | 裸地址为键、无有效性校验（**机制已收窄**，见 Top 20） |
| `01-…/03 F-REG2-003` | P1 / 潜伏 | `FScriptDomainFactory::Create()` → `FDomain::Initialize()` 不判空 |
| `02-…/03 F-DYN-001` | **P0** / 活跃 | `FClassReflection` Parent 未注册 → `BeginGenerator` 无条件解引用 |
| `02-…/04 F-DOM-003` | **P0** / **潜伏** | `FScriptDomainFactory.cpp:44` |
| `02-…/06 F-BR-001` | **P0** / 活跃 | `FTypeBridge.inl:193` → **17/18** 调用点未判空 |
| `06-…/02a F-CS2A-001` | **P0** / 活跃 | `FArrayHelper.cpp:130`（索引越界） |
| `06-…/02a F-CS2A-002` | **P0** / 活跃 | `FMapHelper.cpp:144`/`:171` |
| `06-…/02a F-CS2A-017` | **P0** / 活跃 | `HandleData.cs:88` 返回句柄 0（六个包装类） |
| `06-…/02b F-CS2B-001` | **P0** / 活跃 | `FStringRegistry.inl:23-27`（4/5 个字符串注册器均崩） |
| `08-…/02b F-LIFE-003` | **P0** / 活跃 | `FDelegateRegistry.inl:19-25` |

> ⚠️ **合并提示**：本族内 **17 条 P0** 共享同一句式「**注册表/工厂未命中返回 `nullptr`，调用点不判空**」。**修法也是同一个**：在**未命中处**返回 `InvalidManagedHandle` / 类型默认值，或统一在调用点判空。若按"独立缺陷"报 17 个 P0 会严重误导排期；按"病根 1 类 + 20 余个调用点"表述才准确。**26 条 P0 中另 9 条不属本族**：`F-BIND-001`、`F-DEL-002`、`F-INT2-001`(10b)、`F-RM-001`、`F-DYN-002`、`F-GENC-001`、`F-ED2-001`、`F-ED2-017`、`F-LIFE-001`。
> ⚠️ **同族撞号**：`01-…/10b F-INT2-001`（容器索引堆破坏）与 `01-…/11 F-INT2-001`（DataTable 行名）**编号相同、发现不同**，二者都属本族但与"判空"具体机制不同 —— 引用必须带报告路径。

### 族 2 「参数缓冲不释放」（`F-FN-001` ↔ `F-ABI-031`/`F-ABI-032`）

| 编号（带报告路径） | 级别 / 可达性 | 一句话 |
|---|---|---|
| `01-…/06 F-FN-001` | P1 / 活跃 | 参数缓冲池复用时**既不清零也不析构**，销毁时直接 `FMemory::Free` ⇒ 非平凡类型（FString/FText/TArray/TSet/TMap/对象引用）的内部堆**永久泄漏** |
| `01-…/06 F-FN-002` | P2 / 活跃 | 11 个调用点 `Malloc` 后**从不 `Free`**：参数池"只增不减"（涨到 256 块才回绕），把 F-FN-001 的窗口从"复用"扩大到"跨代残留" |
| `08-…/04 F-ABI-031` | P2 / 活跃 | `FUnrealFunctionDescriptor` 的 5 个调用路径分配参数缓冲后从不释放 |
| `08-…/04 F-ABI-032` | P1 / 活跃 | `INITIALIZE_VALUE()` 只 `InitializeValue_InContainer` **从不 `DestroyValue_InContainer`** |

> ⚠️ **合并计数**：本族四条是**同一病根**（参数缓冲的所有权/析构责任无人承担）。依据 `ScriptCore.cpp:2213-2225` **逐字吻合**（引擎刻意把 `ParmsSize` 区间内参数的析构责任留给调用方；`FCSharpFunctionDescriptor.cpp:134` 的手动 `DestroyValue_InContainer` 是**正确写法**，`FUnrealFunctionDescriptor` 的 14 个 `Malloc` 只配对 3 个 `Free` ⇒ 泄漏成立）。**汇总时不要按 4 条独立缺陷重复计数。**

### 族 3 🔴 「强 `GCHandle` ⇒ C# 终结器永不执行 ⇒ 闭环自锁」（**12 条同族**）

| 编号（带报告路径） | 级别 / 可达性 | 一句话 |
|---|---|---|
| `06-…/02a F-CS2A-023` | P1 / 活跃 | 释放入口只挂在 finalizer 上，而强句柄使 finalizer 永不运行（**Top 17**） |
| `06-…/01 F-CS1-002` | P2 / 活跃 | `HandleData.Alloc` 用 `GCHandleType.Normal`（强句柄）⇒ 依赖终结器的解绑/释放路径**完全失效**（这是本族的**根节点**） |
| `06-…/02b F-CS2B-002` | P1 / 活跃 | 强句柄使字符串包装的终结器永不执行 ⇒ 原生 `new FString/FName/FText` 永久泄漏 |
| `06-…/02b F-CS2B-012` | P1 / 活跃 | `Unreal.LoadObject<T>()` 缺省路径每次调用都 `new FString(...)` 且永不释放（上条的热路径实例） |
| `08-…/02 F-LEAK-001` | P1 / 活跃 | 字符串注册表的释放入口只挂在 C# 终结器上（闭环泄漏） |
| `08-…/02 F-LEAK-002` | P1 / 活跃 | `HandleData.Free` 引用计数"分配>释放"时永久保留 GCHandle |
| `01-…/02 F-REG-007` | P1 / 活跃 | `FMultiRegistry::RemoveReference` **完全不释放 GCHandle** |
| `01-…/02 F-REG-004` | P1 / 活跃 | 反向映射不匹配时**跳过** `GCHandle_Free` |
| `01-…/03 F-REG2-002` | P1 / 活跃 | 3 参 `AddXxxReference` 注册的容器/委托**没有任何回收路径** |
| `01-…/10b F-INT2-011` | P1 / 活跃 | C++ 现场新建并直接返回的托管句柄从不释放 ⇒ 强句柄 + 字典条目无界泄漏 |
| `01-…/07b F-DEL-008` | P1 / 活跃 | `FOptionalRegistry::RemoveReference` 缺 `FDomain::GCHandle_Free` |
| `02-…/02 F-RM-004` | P1 / 活跃 | 句柄释放被 `#if WITH_CORECLR` 屏蔽 ⇒ Mono/LeanCLR 后端泄漏（**本工程五平台全 LeanCLR ⇒ 必然命中**） |

> ⚠️ **合并提示**：族内 **11 条 P1** 的**根节点是同一条**（强句柄 vs 终结器）。`F-CS2A-023` 的价值在于它把"闭环"讲全了（唯一释放写点 + 唯一触发者 + 缺失的第三条路径），**它是本族的代表条目**。实机验证一条即可同时验证全族（"`F-CS2A-023` finalizer 永不执行"属必须实机验证项）。族内 `F-RM-004`、`F-DOM-006` 属"后端宏屏蔽"子机制，合并时注明。

### 族 4 「活值 `Dest` 泄漏」（`InitializeValue` 不先 `DestroyValue`）—— **权威修复清单只有 4 个点**

| 编号（带报告路径） | 级别 / 可达性 | 一句话 |
|---|---|---|
| `01-…/04 F-PROP-003` | **P3（字段）/ 实为 P1** / 活跃 | 对 11 个 Primitive + `FEnumProperty` **不产生堆泄漏**；但在 **20 个 Compound 描述符**的 `Set` 中、当 `Dest` 已是活值时构成**真实堆泄漏**（计数口径 22 → **21 处 / 20 个描述符**） |
| `01-…/04b F-PROP2-002` | P1 / 活跃 | 7 个描述符的 `Set` 对**已初始化内存**调 `InitializeValue` 而不先 `DestroyValue` |
| `01-…/04b F-PROP2-003` | P1 / 活跃 | `Get<FReturn>` 的堆缓冲只 `FMemory::Free`、从不 `DestroyValue` ⇒ 字符串内容整体泄漏 |
| `01-…/05 F-PROP2-007` | P1 / 活跃 | **所有** `Set` 都在"已初始化的属性内存"上调 `Property->InitializeValue(Dest)` |
| `01-…/07 F-HLP-014` | P1 / 活跃 | `FStrPropertyDescriptor::Set` 对每个元素都先 `InitializeValue` 再赋值 |

> 🔴 **本族的"权威修复清单"只有 4 个真泄漏点**（该清单原先列的 4 个可疑点 **4/4 全错 = 0%**）：`FRegisterProperty.cpp:35`、`FRegisterProperty.cpp:64`、`FArrayHelper.cpp:137`、`FOptionalHelper.cpp:91`。
> 🔴 **并且必须显式写明"不要改"的 4 个点**：`FArrayHelper.cpp:269`、`FMapHelper.cpp:200`、`FMapHelper.cpp:218`、`FSetHelper.cpp:102` —— 它们的 `Dest` 来自引擎 `AddUninitialized*`（`UnrealType.h:4015-4021`，**只裸分配、绝不构造**），**在那里补 `DestroyValue` 反而触发 UB**；`FMapHelper.cpp:215` 还是作者**正确**执行"复用前先析构"的对照实现。
> ⚠️ **上游旧结论已作废**：`INDEX.md:66`、`00-总览/03:89`/`:392` 曾原样引用 **7 个**点（且自称 7 个、实列 8 个路径，内部不自洽）—— **照旧清单施工会引入比原缺陷更严重的 UB**。

### 族 5 「哈希契约违反」

| 编号（带报告路径） | 级别 / 可达性 | 一句话 |
|---|---|---|
| `06-…/02b F-CS2B-004` | P1 / 活跃 | `GetHashCode()` 返回**句柄**而非内容哈希 ⇒ 违反 `Equals`/`GetHashCode` 契约，`Dictionary`/`HashSet` 失效 |
| `03-…/02 F-GEN2-014` | P1 / 活跃 | `FStructGenerator` 的 `GetHashCode()` 用**句柄（身份）**而 `Equals`/`==` 用**值比较** |

> ⚠️ **合并计数**：两条是**同一个契约违反**在"手写包装类"与"生成器模板"两侧的表现 ⇒ 修法必须**成对**（生成器里的一样错），否则生成产物会重新引入。统计时计 **1 个病根 / 2 个修复点**。

### 族 6 「委托 / 绑定悬空」（两个子族，含 **2 组同机制重复 P0**）

**子族 6a：句柄被丢弃 / 覆盖 ⇒ 悬空 `this`**

| 编号（带报告路径） | 级别 / 可达性 | 一句话 |
|---|---|---|
| `04-…/02 F-ED2-001` | **P0** / 活跃 | AssetRegistry 5 个 `AddRaw` 句柄全部丢弃且从不反注册（**Top 11**） |
| `08-…/02b F-LIFE-001` | **P0** / 活跃 | **同一机制的另一条编号**（`FEditorListener` 的 5 个 Raw 委托从未解绑） |
| `04-…/02 F-ED2-017` | **P0** / 活跃 | 蓝图工具栏 extender 委托永不反注册（**Top 12**） |
| `04-…/02 F-ED2-003` | P1 / 活跃 | 目录监视回调句柄**在循环中被覆盖**，析构时用错句柄反注册 |
| `08-…/02 F-LEAK-008` | P2 / 部分确认 / 活跃 | **同一机制的另一条编号**（N 个注册只反注册 1 个，并留下悬垂 `this`） |
| `08-…/02b F-LIFE-012` | P2 / 活跃 | `FClassCollector` 在 `OnEndFrame` 反复 `AddStatic` 并覆盖同一句柄，析构只解绑最后一个 |
| `08-…/02b F-LIFE-013` | P3 / 活跃 | Cook 路径 `OnFilesLoaded()` 只挂 lambda 不存句柄，`ShutdownModule` 无从解绑 |
| `04-…/02 F-ED2-011` | P3 / 潜伏 | 注册/反注册整体配对良好，但存在 3 处不对称 |

> 🔴 **合并提示（最重要）**：`F-ED2-001` 与 `F-LIFE-001` 是**同一处代码、同一机制**在两份报告里各占一条 **P0**；`F-ED2-003` 与 `F-LEAK-008` 同理（P1 / P2 各一）。⇒ **P0 计数若不去重会虚高 1 条**；统计"待修点"应计 **2 个真修复点**（`FEditorListener` 的 5 个句柄 + 蓝图工具栏的 1 个句柄），而不是 8 条编号。

**子族 6b：按"名字"绑定 ⇒ 点名歧义 / 悬垂函数**

| 编号（带报告路径） | 级别 / 可达性 | 一句话 |
|---|---|---|
| `06-…/02b F-CS2B-019` | P1 / 活跃 | 输入绑定以 `Action.Method.Name` 为键、绑定对象按 `UClass` 获取 ⇒ **同类多实例的同名方法绑定被静默丢弃 / 误解绑** |
| `06-…/02b F-CS2B-017` | P1 / 活跃 | `UInputComponent.Remove*` 六个方法**不做任何原生解绑**，却删除了绑定仍指向的 UFunction ⇒ 悬垂函数调用 + C# 委托永久泄漏 |
| `01-…/07b F-DEL-004` | P1 / 活跃 | 单播 `DelegateWrapper` **没有任何解绑点**，`UnBind`/`Clear`/`Deinitialize` 都不清理 C# 目标 |
| `01-…/08 F-BIND-007` | P1 / 活跃 | `TMethodHelper` **仅在"句柄变化"时**回写非平凡 ref/out 参数 ⇒ 托管侧原地修改会丢失（与 Top 1 同一"out 参数编组"病根） |

### 族 7 「容器索引 / 长度不校验 ⇒ 堆破坏」

| 编号（带报告路径） | 级别 / 可达性 | 一句话 |
|---|---|---|
| `01-…/10b F-INT2-001` | **P0** / 活跃 | `TArray` 的 5 个导出零 index 校验，Shipping 下 `checkSlow` 退化为 `CA_ASSUME`（**Top 3**） |
| `01-…/07 F-HLP-007` | P1 / 活跃 | 索引型修改接口不做 `IsValidIndex` 校验，**与同文件的 `Get`/`Set` 自相矛盾** |
| `06-…/02a F-CS2A-011` | P1 / 活跃 | `TArray<T>.RemoveAt` 不做下标校验就计算元素指针并逐元素 `DestroyValue` |

> ⚠️ **合并提示**：三条分别是"Interop 导出层 / Helper 层 / C# 包装层"的**同一校验缺失**，且 `F-HLP-007` 与 `F-INT2-001` 的调用链**首尾相接**（C# → `FRegisterArray` → `FArrayHelper`）⇒ 计 **1 个病根 / 3 个修复层**。

### 族 8 「只 `Free`/`delete` 不析构」

| 编号（带报告路径） | 级别 / 可达性 | 一句话 |
|---|---|---|
| `01-…/07 F-HLP-001` | P1 / 活跃 | `delete ScriptArray` 经 `FScriptArray*` 删除 `TArray<T>` 对象 ⇒ **元素析构被跳过** |
| `01-…/07 F-HLP-020` | P1 / 活跃 | `FMapHelper::Empty` 只调 `FScriptMap::Empty`，**不析构 key/value** |
| `01-…/07 F-HLP-025` | P1 / 活跃 | `FSetHelper::Empty` 不析构元素 |
| `01-…/02 F-REG-006` | P3 / 活跃 | `new` 分配 + `FMemory::Free` 释放 —— **分配器是匹配的**（原"分配器不匹配/堆损坏"**已证伪**），真缺陷是 **`FMemory::Free` 跳过析构** |
| `08-…/02 F-LEAK-012` | P2 / 活跃 | 「元素缓冲泄漏」**已证伪**（`delete` 会经分配器基类析构 `Free(Data)`）；成立的只有"元素析构缺失" |

> 🔴 **合并计数 + 修法警告**：以上各条与"`FMemory::Free(ptr)` 不调用 `~T()`"**是同一病根，不应作为独立 P1 重复计数**。
> 🔴 **`F-LEAK-012` 原文建议的 `FMemory::Free(ScriptArray)` 修法必须撤销** —— 它会**反向制造它想修的那个缓冲泄漏**；必须**保留 `delete`、只补元素析构**。这是"错误结论会以权威名义扩散、照做反而引入新缺陷"的典型。

### 族 9 「静态缓存 / 裸指针注册表永不清理」

| 编号（带报告路径） | 级别 / 可达性 | 一句话 |
|---|---|---|
| `02-…/01 F-REFL-002` | P1 / 活跃 | `FDynamicGeneratorCore` 的 6 个**函数局部 `static TArray<FClassReflection*>`** 缓存跨热重载**永不失效** |
| `02-…/03 F-DYN-006` | P1 / 活跃 | 元数据属性类被缓存进函数级 `static`，跨 C# 热重载持有**已 `delete`** 的 `FClassReflection*` |
| `02-…/03 F-DYN-003` | P1 / 活跃 | 旧动态类被 `MarkAsGarbage` 后，多处 `TMap` 仍持有它的**裸指针键** ⇒ UObject 地址复用 → 串数据 / 用已释放的 `FProperty` 写内存 |
| `08-…/02b F-LIFE-006` | P1 / 活跃 | 动态类型的静态裸指针注册表永不清理（`NamespaceMap`/`DynamicXMap`/`DynamicXSet`） |
| `08-…/02 F-LEAK-013` | P2 / 活跃 | 动态生成器的 4 个 `NamespaceMap` 等静态裸指针键容器只 `Add` 从不清理 |
| `02-…/05 F-CB-005` | P2 / 活跃 | 物化产物 `FBindingClass`/`FBindingEnum` 为裸指针且**无所有者**，永不释放 |

> ⚠️ **合并提示**：本族六条共享"**函数级/类级 `static` 持有裸指针，跨域重载不失效**"这一机制；`F-DYN-006` 与 `F-REFL-002` 尤其**修法相同**（改为 `TWeakObjectPtr` 或在 `Deinitialize` 清空）。族内 `F-DYN-003` 的后果最重（UAF 写），合并时按它定级。
---

## §4 已证伪 / 健康结论（**撤销 32 条**）：分两类落账

**来源**：`09-问题清单/01-全局问题总表.md` 的 32 行"级别 = 撤销"（可复现：`^\| F-… \| 撤销 \|` 命中 **32** 行）。

### 4.1 ✅ 撤销成立、可结案（30 条）

| # | 编号（带报告路径） | 原判 | 撤销依据 |
|---|---|---|---|
| 1 | `00-总览/01 F-ARCH-005` | 动态 UClass 永久 `AddToRoot()` 累积根集 | **成对调用证据**：`FDynamicClassGenerator.cpp:393 AddToRoot()` ↔ `:529-543`（`ReInstance` 内）的 `RemoveFromRoot()/MarkAsGarbage()` 成对；Core 模块 grep 中 4 处 `RemoveFromRoot` 与 4 个 `AddToRoot` **一一对应** |
| 2 | `00-总览/01 F-ARCH-006` | 全局单例群析构顺序不确定 | 机制描述属实但**后果不成立**：`FScriptDomainFactory.cpp:47-59` 在 `Deinitialize()` 后 `delete` 整个域、`:58 IScriptDomain::Set(nullptr)` |
| 3 | `01-…/03 F-REG2-001` | `NewWeakRef(bIsCopy=true)` 误 `delete` UObject 属性内存 → 堆损坏 | 已证伪（编号保留） |
| 4 | `01-…/03 F-REG2-008` | `NotifyUObjectDeleted` 未做线程归属判断 | 已证伪 |
| 5 | `01-…/04b F-PROP2-001` | `FStructPropertyDescriptor::Set` 落到 `memcpy` 兜底即双重释放 | **★ 逐字验证**：`PropertyStruct.cpp:337-340` 证明 `FStructProperty::CopyValuesInternal` 确实覆写并转调脚本层**深拷贝**（不是 `memcpy`）⇒ 假设 A 成立、假设 B 证伪 |
| 6 | `01-…/07 F-HLP-004` | 自建 `FScriptArray` 路径元素缓冲永不释放 | 机制成立但**不可达** |
| 7 | `01-…/07 F-HLP-009` | `Reset(InNewSize)` 与 `TArray::Reset(NewSize)` 语义不符 | **UE 的 `TArray::Reset` 本来就不设置元素个数** |
| 8 | `01-…/07 F-HLP-027` | `FSetHelper::GetEnumerator` 的 `nullptr` 被解引用 | 机制成立但**单线程不可达** |
| 9 | `01-…/07 F-HLP-029` | `FSetHelper::Remove` 不 `Rehash` | 引擎 `RemoveAt` **自己维护哈希链**，不 `Rehash` 是正确的 |
| 10 | `01-…/10b F-INT2-009` | LeanCLR 走 `[DllImport]` 但无对应 C 导出符号 | 已证伪 |
| 11 | `01-…/10b F-INT2-010` | 容器"值/句柄"双模式契约隐式对齐 | **不一致类不存在** |
| 12 | `01-…/10 F-INT1-008` | `LoadObject` 失败 → `Bind(nullptr)` 的 null 安全性 | 撤销**并升级为证据级** |
| 13 | `01-…/10 F-INT1-009` | 死注册：`UClass.ClassDefaultObject` 无 C# 消费者 | 撤销**并升级为证据级**；口径更正为"**UE 5.6 下被版本宏编译掉**"（源码 52 条 / 实际生效 **51** 条） |
| 14 | `02-…/02 F-RM-015` | `UnboxValue` 返回栈局部地址 | 原判"LeanCLR 实现返回栈局部地址"**不成立** |
| 15 | `02-…/03 F-DYN-007` | `PurgeClass(true)` 后不调 `DestroyPropertiesPendingDestruction()` | 已撤销 |
| 16 | `02-…/04 F-DOM-026` | `FStringToString<>` 的转换对象可能悬垂 | **✅ 引擎源码彻底证伪**：`StringConv.h:663-701` 自持缓冲 + 拷贝/移动构造 `= delete` |
| 17 | `02-…/06 F-BR-002` | `GetClass(FProperty*)` 缺 `FUtf8StrProperty`/`FAnsiStrProperty` 分派 | 已证伪 |
| 18 | `03-…/01b F-GEN-004` | `FRotator`/`FVector` 默认值被 `ParseIntoArray + Atod` 清成 0 | **✅ .NET SDK 10.0.401 实测**：往返值正确 |
| 19 | `03-…/01b F-GEN-007` | C# 关键字表漏上下文关键字 `value` | **✅ 实测** `value` 属性名往返值 **42** 正确 |
| 20 | `03-…/01b F-GEN-009` | 默认参数字面量两套实现语义分歧 | **✅ 维持撤销** |
| 21 | `04-…/01 F-ED1-008` | `CompileFilter` 只累加不清空 | 已证伪 |
| 22 | `04-…/02 F-ED2-026` | 三个路径定制是"重复实现" | 已是正确的三层结构；残留仅两个 7 行工厂文件 |
| 23 | `05-…/01 F-CMP-031` | `AsyncTask(GameThread) + WaitUntilTaskCompletes` 与自旋等待 | 经核对不是缺陷（**保留编号**、不计入待修 ⇒ 该报告待修 **33 → 32 条**） |
| 24 | `06-…/01 F-CS1-013` | `FieldBridge::SetStaticValue`/`GetStaticValue` 零调用点 | 已证伪 |
| 25 | `06-…/03 F-CS3-006` | 3 个 C++ 注册在 C# 侧无声明 | **更正为只有 1 个成立**（`Contains1`，已并入 `F-CS3-009`），另 2 个是**活的** |
| 26 | `07-…/01 F-BLD-001` | `SourceCodeGenerator` 的 props 硬编码作者机器绝对路径 | 已证伪 |
| 27 | `07-…/01 F-BLD-006` | Editor 模块把 `UnrealEd`/`DirectoryWatcher`/`CollectionManager` 放进 Public deps | 已证伪 |
| 28 | `07-…/01 F-BLD-007` | `DirectoryWatcher` 放在 `PublicDependencyModuleNames` | 已证伪 |
| 29 | `08-…/02b F-LIFE-002` | `FDelegateWrapper::Method` 裸指针域重载后 UAF | 已证伪 |
| 30 | `08-…/05 F-PERF-029` | 编辑器 ticker 注册后永不注销、每帧空转 | 已证伪 |

### 4.2 ⚠️ 存疑、**不得结案**（2 条）

| 编号（带报告路径） | 原判 | 发现 | 处置 |
|---|---|---|---|
| `06-…/01 F-CS1-004`（`HandleData.GetObjectPointer` 返回托管裸地址、未 pinned） | 已证伪（非缺陷） | 撤销前提**查不到证据**；且"无人使用"的理由被 grep 排除 —— `GetObjectPointer` 在 `FLeanCLRDomain.cpp:125,155,167,176,196` 有 **5 处活跃调用** | ⚠️ **改判"存疑"**，**不得写入"已判定为非缺陷"清单** |
| `06-…/01 F-CS1-026`（`Weavers.csproj` 从未定义 `WITH_LEANCLR`） | 已证伪（非缺陷） | weaver 侧 `WITH_LEANCLR` 在整个插件根 **37 命中**里**只由** `FSolutionGenerator.cpp:214` 注入 `Template/Shared.props:7`；若 weaver 项目也吃该 props 则撤销成立，**否则原发现是真缺陷** | ⚠️ **改判"存疑"**，需核 weaver csproj 是否引入 `Shared.props` |

> **两条相邻的"边界条目"（都在 32 条之外，但同样不得当成已结案）**：
> - 🔴 `01-…/04 F-PROP-007`（`FFieldPathPropertyDescriptor` 不覆盖任何方法 ⇒ `FFieldPath` 属性读写**全部静默 no-op**）：它**不在** 32 条撤销里（当前 `严重度` = **P1 / 活跃**），但**三条同类型编号**（`08-…/01 F-DEAD-007`、`01-…/05 F-PROP2-004`、`01-…/04 F-PROP-007`）都属"**不可直接结案**"。**主流结论是"不可达"**（决定性证据：`FGeneratorCore.cpp:833 if (CastField<FFieldPathProperty>(Property)) return false;` —— 生成器 `IsSupported` **显式排除**该类型 ⇒ C# 侧永不按哈希请求该描述符），**但残留未排除**：作为"受支持容器/结构体的 inner property"是否仍能触发 `FPropertyDescriptor::Factory`（`FPropertyDescriptor.cpp:127`）—— 需核 `CreateHelperFormInnerProperty` 路径。⇒ **在此之前，不得把它写入"已判定为非缺陷"。**
> - ✅ `01-…/02 F-REG-005`：原标"撤销（非缺陷）"且**已从排期移除**，现**撤回、恢复 P2**（其正文描述的机制在源码中每一跳成立、且回调路径唯一；引擎明写 `TMap::Find` 返回的指针 *"is only valid until the next change to any key in the map"*（`Map.h:564-565`））。失效是**顺序相关**而非必然；**若运行时实测出稳定双删，则应升 P0**。⇒ 它**不在** 32 条撤销内，**必须留在排期**。

### 4.3 ⚠️ 口径警告：`撤销 = 32` 与 `P3 = 381` 之间存在已知污染

存在一个**会直接污染汇总**的字段级缺陷：某些条目"**标题 / 复核结论 / 级别变动三处都写『撤销（非缺陷）』，但 `严重度` 字段仍留着 `P3`**" —— 实证是 `08-…/04:1748-1755` 的 **`F-ABI-022`**（`:1751` 严重度 = `P3`、`:1752` 复核结论 = 证伪（撤销，非缺陷）、`:1755` 级别变动 = `P3 → 撤销（非缺陷）`）。

⇒ **`P3 = 381` 可能包含少量"实际已撤销"的条目，`撤销 = 32` 可能因此偏低**；这类条目的**总数尚未全量清点（未取得）**（需要另写一个"标题/复核结论含撤销但 `严重度` 字段为 P0–P3"的扫描器，尚未编写）。⇒ 引用 `P3 = 381` 与 `撤销 = 32` 时，请把这条口径风险一并带上。
---

## §5 **已推翻的旧结论**（O-1…O-20 中最主要的 12 条）

> **性质**：下表每一行的左栏都是**曾被当作权威、并已被 `INDEX.md` / `00-总览/03` / `09-问题清单` 原样引用过**的结论。**引用旧结论前务必先看这张表。**
> **证据纪律**：右栏全部引自落盘的报告或独立抽验结果；本文件**未自行回源码复验**（本文件不改源码、不重新分析）。

| # | 原文（曾被当作权威） | **实际** |
|---|---|---|
| **O-1** | "活值 `Dest` 权威修复清单 = **7 个**调用点"（被 `INDEX.md:66`、`00-总览/03:89`/`:392` 三处引用；且自称 7 个、实列 8 个路径） | **只有 4 个**真泄漏点：`FRegisterProperty.cpp:35`、`:64`、`FArrayHelper.cpp:137`、`FOptionalHelper.cpp:91`；另 **4 个点必须显式写明"不要改"**（`FArrayHelper.cpp:269`、`FMapHelper.cpp:200`/`:218`、`FSetHelper.cpp:102`）—— 它们的 `Dest` 来自 `AddUninitialized*`（`UnrealType.h:4015-4021`，**只裸分配不构造**），**在那里补 `DestroyValue` 反而触发 UB**。该清单 4 个可疑点 **4/4 全错（吻合率 0%）**；另有 **3 份独立分析**得出同一结论 |
| **O-2** | `F-REG-005` = "撤销（非缺陷）"，**已从排期移除** | **撤回，恢复 P2**。机制在源码中**每一跳都成立**（`FStructReference` 析构 → `FStructRegistry.cpp:119` → 回调 `FReferenceRegistry.cpp:44`，且该回调路径**唯一**）；引擎明写 `TMap::Find` 的指针 *"is only valid until the next change to any key in the map"*（`Map.h:564-565`）。**"错误撤销"比"高估"危险得多 —— 缺陷会永久漏修** |
| **O-3** | `F-CS1-004` / `F-CS1-026` = "已证伪（非缺陷）"，可结案剔除 | **改判"存疑"、不得当结案剔除**：① `F-CS1-004` 的撤销前提**查不到证据**，且"无人使用"被 grep 排除（`GetObjectPointer` 有 **5 处活跃调用**）；② `F-CS1-026` 的 weaver 侧 `WITH_LEANCLR` 只由 `FSolutionGenerator.cpp:214` 注入，**若 weaver csproj 不吃 `Shared.props` 则原发现是真缺陷** |
| **O-4** | "`FString::Equals` 的默认 `ESearchCase` **无法裁决**"（`INDEX.md:76` 也列为"无法裁决"） | **已裁决：默认 `CaseSensitive`**（`UnrealString.h.inl:1492`）；同文件 `:1053-1056` 证明 **`operator==` 是 `IgnoreCase`** ⇒ **两者语义不同，任何混用结论都是错的**。**0 命中的根因：只搜了 `.h`，而成员实现体在 `.inl`** |
| **O-5** | `F-MACRO-006` 的核心断言："宏放进无花括号 `if/else` 即编译失败（**实测 `error C2062`**）" | **证伪**：用**本机真实编译器**（MSVC 14.51.36231 传统模式 + `/Zc:preprocessor`、clang 20.0.0git）**四变体矩阵实测全部编译通过**；真实失败条件是"调用处多写分号 / 宏体自带 `;`"，诊断是 MSVC **C2181**、clang `expected expression`。该 Finding 其余内容仍成立 ⇒ **部分确认**（非撤销），可达性 **活跃 → 潜伏**。⚠️ **凡引用过 "`C2062` 结论" 的其它报告需一并更正**（全局 grep 核对**尚未执行**） |
| **O-6** | `F-LEAK-012`：①"元素缓冲泄漏" ②建议改为 `FMemory::Free(ScriptArray)` | **一半证伪、一半成立，且原文的修法方向是反的**：①"缓冲泄漏"**证伪**（`delete` 经容器分配器基类析构 `Free(Data)`，`ContainerAllocationPolicies.h:682-693` 实证）；②"元素析构缺失"**成立**（`FArrayHelper.cpp:34-39` 无 `EmptyValues`/`DestroyValue`）⇒ 泄漏收窄为**元素内层堆**；③**原文修法必须撤销** —— 它会**反向制造它想修的那个缓冲泄漏**。级别 **P1 → P2** |
| **O-7** | `F-DOM-002` 的 **P0** 前提："`hostfxr_close` 后调用已失效委托 → use-after-close" | **证伪**（dotnet/runtime 源码）：`hostfxr_close` **不卸载运行时**（`hostfxr.cpp:1003-1013` → `fx_muxer.cpp:957-976` → `host_context.cpp:142-145` 只置 marker；`fx_muxer.cpp:182` 注释 *"we do not unload coreclr"*）⇒ **P1 → P3**。CoreCLR 后端**真正的失败态是 `F-DOM-005`**（初始化早退留下门闩 ⇒ 委托恒为 `nullptr`、不可恢复） |
| **O-9** | `F-ARCH-005`（动态 UClass 永久 `AddToRoot()`）/ `F-ARCH-006`（单例群析构顺序）曾被标"撤销"，但**撤销的证据本身缺失** | **撤销成立，证据充分**：`FDynamicClassGenerator.cpp:393 AddToRoot()` ↔ `:529-543` 的 `RemoveFromRoot()/MarkAsGarbage()` **成对**（Core 模块 grep 中 4 处 `RemoveFromRoot` 与 4 个 `AddToRoot` 一一对应）；`F-ARCH-006` 的机制描述属实但**后果不成立**（`FScriptDomainFactory.cpp:47-59`） |
| **O-11** | `06-…/03`："C++ 214 个导出 − C# 211 个声明槽 = **3 个死注册**" | **只有 1 条不一致**（C++ 214 − C# 声明槽 213）。**根因：原 grep 只查了插件的 `Script/` 树**（`Select-String -Path "Script\*\*\*.cs"`），**漏掉工程侧生成产物** `Script/UE/Proxy/`；被推翻的 2 项在项目侧产物里**有声明且被真实调用**。⚠️ `06-…/01` 与 `08-…/01` 可能犯同一错误，**两棵树范围的重跑尚未做** |
| **O-12** | `04-…/02` 的 **4 条 P0**（`F-ED2-001`、`F-ED2-004`、`F-ED2-017`、`F-ED2-018`） | **只剩 2 条**：`F-ED2-001`、`F-ED2-017` 存续；**`F-ED2-004`**（"启动竞态空指针"）—— 引擎 `FRunnableThread::Create` 返回前 `Init()` 已完成（`MicrosoftRunnableThread.h:179 ThreadInitSyncEvent->Wait(INFINITE)`），**竞态不存在**；**`F-ED2-018`**（"内层 `[&]` 读悬空栈帧"）—— 外层 lambda 已按值捕获 `[this, InBlueprint]`，`:168` 的 `[&]` 引用的是**闭包成员**而非栈参数 ⇒ 两条均**证伪撤销**。**下游 P0 计数须按 2 计** |
| **O-13** | `F-DOM-003` 的 **P0 理由**："LinuxArm64 回退 CoreCLR 而构建 LeanCLR" | **理由被证伪、结论维持（P0）**：该平台构建期 `Build.cs:263-267` **同样**回退 `"CoreCLR"`，与运行期一致。**新理由更硬**：插件**不含任何 `Config` 文件**（glob `**/Config/*` 命中 **0**）⇒ 无 ini 的工程运行期默认 `Mono`（枚举零值 + CDO 内存被引擎清零，`UObjectGlobals.cpp:3845`）、构建期默认 `CoreCLR` ⇒ **必然** `Create() == nullptr` → `FDomain.cpp:31` 空指针崩溃**且无日志**。这是"**结论对、理由错**"的典型，汇总时**必须用新理由** |
| **O-20** | `F-REG-006` = **P1**"`new` 分配 + `FMemory::Free` 释放 → 分配器不匹配、内存损坏" vs `F-BIND-012` = "已确认不是问题"；第一版建议"降为 P3 并交叉引用" | **"内存分配器不匹配/内存损坏"证伪，但理由与第一版不同**：分配器**是匹配的**（UBT 每模块生成 `PerModuleInline.gen.cpp` → `HAL/PerModuleInline.inl:9` 展开 `REPLACEMENT_OPERATOR_NEW_AND_DELETE`；本机 `.obj` 符号表证明模块内定义了 `??2@YAPEAX_K@Z`，DLL 不导入它）。**且第一版采信的 `F-BIND-012` 机制也是错的**（它引的 `PER_MODULE_BOILERPLATE` 在 UE 5.6 **已不成立**）。**真缺陷是 `FMemory::Free` 跳过析构**（归入泄漏族，P1 级）⇒ `F-REG-006` **P1 → P3**。⚠️ **第一版 §C1 的裁决理由已被替换** |

**其余 8 条（简表）**：**O-8** `F-ABI-022` **撤销**（UBT **无条件**加 `/utf-8`，`VCToolChain.cs:650` + `/wd4819` `:653`）、`F-ABI-021` 前提证伪（UBT 扩展名判定显式大小写不敏感，`FileSystemReference.cs:31`）→ P3；**O-10** "LeanCLR 下 `SynchronizationContext.Tick` 恒为空操作 ⇒ `await` 续体永不执行"（曾被建议立 P0）**证伪**（根因是"**跨后端同名成员的类型同构假设**"：LeanCLR 下该成员是 `const RtMethodInfo*` 而非函数指针）；**O-14** `F-GENC-002` 机制（"路径风格失配 → 整目录删光"）**证伪**，真实可达条件 = "本次运行未生成的模块/资产"（开启 `IsSkipGenerateEngineModules` 会删净 **16 055** 个 `.cs`），且 `F-GENC-010` 的缓解论据"`bIsGenerateAllModules` 默认 false"**也证伪**（实测默认 **true**）；**O-15** `F-CMP-013` "`STD_CPP_20` 在 MSVC 恒假"**证伪**（UBT 默认开 `/Zc:__cplusplus`）⇒ P2→P3、活跃→潜伏；**O-16** `FFieldPathProperty` 三报告分歧（2:1）以"**不可达**"为主流（`FGeneratorCore.cpp:833` 显式排除），**残留未排除**；**O-17** "本机无引擎源码"（9 份报告据此降级 30+ 条结论）**是错的**，引擎源码真实存在（约 **20989** 个 `.cpp`），已逐行修正 **24 处**，多条结论**方向反转**；**O-18** `02-…/04` §3 链 4 的域销毁触发点写错（**"PIE 停止"不销毁域**；真实触发是 C# 编译后的下一次 PIE 启动 / 控制台 `SetActive` / 进程退出）⇒ `F-DOM-006` **P2 → P1**；**O-19** `F-INT2-002`（32 位 ABI 不匹配）**P1 → P3**（`UEBuildAndroid.cs:143-150/303` 只允许 arm64/x64，本插件**不支持也无法构建 32 位目标**）。
---

## §6 修复路线图（三阶段）

> 🔴 **贯穿全阶段的总前提（务必先读）**：
> **本文件的全部结论均为静态阅读**（读插件源码 + UE 5.6 引擎源码 + LeanCLR / dotnet-runtime 源码 + 真实生成产物），**没有任何一条 P0/P1 在本机被实际触发过** —— 41 份含发现的报告全部自述"未编译、未运行、未做 ASan/定量测量"。**需要实机验证的项见各报告自己的"未覆盖 / 无法验证"章节**（其中 7 类"必须实机验证后才能当结论用"的项已按模块聚合）。

### 阶段 1 · 立刻修（**P0 = 26 条编号；按病根去重后约 25 条**）

| 顺序 | 修什么 | 为什么排这个位置 | 改动规模 |
|---|---|---|---|
| 1-1 | **族 1「零判空」**（`01-…/04b F-PROP2-004`、`04b F-PROP2-005`、`05 F-PROP2-001`、`05 F-PROP2-002`、`07 F-HLP-023`、`07b F-DEL-003`、`10 F-INT1-001`、`10 F-INT1-002`、`11 F-INT2-001`、`02-…/03 F-DYN-001`、`02-…/04 F-DOM-003`、`02-…/06 F-BR-001`、`06-…/02a F-CS2A-001`/`F-CS2A-002`/`F-CS2A-017`、`06-…/02b F-CS2B-001`、`08-…/02b F-LIFE-003`） | **17 条 P0 同一句式、同一修法**（未命中处返回 `InvalidManagedHandle`/默认值或调用点判空）⇒ **成本最低、收益最大**；且多条是"用户一行代码即可触发"（`new FString().ToString()`、`map[不存在的键]`、错一个下标） | 20 余个调用点，**多数为 1–3 行** |
| 1-2 | **`01-…/08 F-BIND-001`**（`TOut.inl:36` 步长） | 全库**唯一**"活跃 + 后果确定 + 机制已**逐步手算复现**"的 P0；且**插件自带两条绑定即命中**（`GetDate`/`GetIdentity`） | 1 个模板文件、**1 处步长表达式** |
| 1-3 | **`01-…/07b F-DEL-002`**（多播广播改快照遍历） | `Clear()` 路径是**真 use-after-free**；改动限于 1 个函数 | 1 个函数 + 防重入标志 |
| 1-4 | **`02-…/02 F-RM-001`**（`Methods` 去重键） | **静默数据损坏**（调到错误方法 + 用错误类型读实参）+ 必有泄漏；影响**全插件每一次跨语言调用** | 键构造处 + 生产端（C#）不去重两个点 |
| 1-5 | **`02-…/03 F-DYN-002`**（依赖图环检测） | **编辑器挂死**（无崩溃报告、无法从日志定位）；自环/互环必然命中 | 1 个结构体加计数 + 1 处判断 |
| 1-6 | **`03-…/01 F-GENC-001`**（检查 `Deserialize` 返回值） | 触发只差"JSON 非法"一个条件，而**两个 JSON 文件实测存在** ⇒ 每次生成都执行到该行 | **1 行** |
| 1-7 | **`04-…/02 F-ED2-001`** + **`08-…/02b F-LIFE-001`**（**同一机制，去重）** + **`04-…/02 F-ED2-017`** | 3 条编号 / **2 个真修复点**（`FEditorListener` 的 5 个 `FDelegateHandle` + 蓝图工具栏 extender 委托）；不修则"每次退出/每次打开蓝图"必然 UAF | 2 处 `Deinitialize`/析构 + 句柄成员 |
| 1-8 | **`01-…/10b F-INT2-001`**（5 个导出的 index 校验） | Shipping 下堆破坏（`checkSlow` 退化为 `CA_ASSUME`）；`Arr.RemoveAt(0, 2)` 即可触发 | 4 个函数前置校验 |

**阶段 1 的 3 个前置动作（阻断项，不先做就地雷）**：

1. 🔴 **先补 4 份缺失的引文行号数据，优先 `01-…/07-容器Helper.md`** —— `FScriptArrayHelper` 的 `EmptyValues`/`RemoveValues`/`MoveAssign` **会**析构非 POD 元素，即**推翻了该报告若干 P0/P1 赖以成立的"引擎 Helper 不析构"前提**；而该报告**尚未逐条重判**。**在重判完成前，`01-…/07` 的 P0/P1 不得作为施工依据。**
2. 🔴 **把 `严重度` 字段与"撤销"判定同步** —— 这是**唯一会直接污染汇总**的字段缺陷（实证 `08-…/04:1748-1755` 的 `F-ABI-022`）；不改，则 `P3 = 381` 与 `撤销 = 32` **至少有一个是错的**（见 §4.3）。
3. ⚠️ **全局 grep `C2062` 清理残留引用**，并把其中引用的"**7 个**"同步为"**4 个**"。

### 阶段 2 · 计划修（**P1 = 128 条**，按病根合并后约 60 个修复批次）

| 顺序 | 修什么（含报告路径） | 依据 / 备注 |
|---|---|---|
| 2-1 | **族 4 的 4 个权威活值 `Dest` 点**：`FRegisterProperty.cpp:35`、`:64`、`FArrayHelper.cpp:137`、`FOptionalHelper.cpp:91` | 🔴 **同时把"不要改"的 4 个点写进注释**：`FArrayHelper.cpp:269`、`FMapHelper.cpp:200`/`:218`、`FSetHelper.cpp:102`（`AddUninitialized*` 产物上补 `DestroyValue` = **UB**） |
| 2-2 | **族 2 参数缓冲**：`01-…/06 F-FN-001`/`F-FN-002` ↔ `08-…/04 F-ABI-031`/`F-ABI-032`（**同一病根，合并为 1 个批次**） | 引擎刻意把 `ParmsSize` 区间内参数的析构责任留给调用方（`ScriptCore.cpp:2213-2225` 逐字验证）；`FCSharpFunctionDescriptor.cpp:134` 是**正确写法**，14 个 `Malloc` 只配 3 个 `Free` |
| 2-3 | **族 3 的根节点**：`06-…/01 F-CS1-002`（强句柄）与 `06-…/02a F-CS2A-023`（闭环）**一处修复解全族** | 族内 11 条 P1 的根节点相同；`06-…/02a` 自标"**未实机证实**"，属必须实机验证项 |
| 2-4 | `01-…/10b F-INT2-012`（出口编码） | 五个平台 `TCHAR` **都是 2 字节** ⇒ **全部目标平台都错**；修 2 个文件（`FRegisterAnsiString.cpp:52`、`FRegisterUtf8String.cpp:50`） |
| 2-5 | `05-…/01 F-CMP-002`（`SyncProcess` 超时） | 不修则 `dotnet` 卡住 = **编辑器永久挂死**（含游戏线程的 `GetDotNetPathArray()`）；改 1 个函数 + ini 项 |
| 2-6 | **族 5 哈希契约**：`06-…/02b F-CS2B-004` ↔ `03-…/02 F-GEN2-014`（**必须成对修**） | 只修手写包装类，生成产物会重新引入 |
| 2-7 | **族 6 委托解绑**：`01-…/07b F-DEL-004`、`04-…/02 F-ED2-003`/`F-LEAK-008`（去重）、`08-…/02b F-LIFE-012`/`F-LIFE-013` | 子族 6a 的遗留项（阶段 1 只修了 2 个 P0 修复点） |
| 2-8 | `06-…/02b F-CS2B-017`/`F-CS2B-019`（输入绑定按名） | 同类多实例同方法名 ⇒ 静默丢弃/误解绑 + 悬垂 UFunction 调用 |
| 2-9 | **族 9 静态缓存/裸指针注册表**：`02-…/01 F-REFL-002`、`02-…/03 F-DYN-006`/`F-DYN-003`、`08-…/02b F-LIFE-006`、`08-…/02 F-LEAK-013` | 统一改法：`TWeakObjectPtr` 或 `Deinitialize` 清空 |
| 2-10 | **族 7 容器索引校验**（`01-…/07 F-HLP-007`、`06-…/02a F-CS2A-011`）与**族 8 只 Free 不析构**（`F-HLP-001`/`F-HLP-020`/`F-HLP-025`/`F-REG-006`/`F-LEAK-012`） | 🔴 族 8 **必须保留 `delete`、只补元素析构**（`F-LEAK-012` 原文修法方向是反的，照做会制造新泄漏） |

### 阶段 3 · 观察（**不排期，但必须登记**）

| 类别 | 条数 | 处置 |
|---|---|---|
| **潜伏** | **140** | 含**全部 Mono/CoreCLR 专属发现**（本工程五平台**全 LeanCLR**：`Definitions.UnrealCSharpCore.h:19-21` = `WITH_CORECLR 0 / WITH_MONO 0 / WITH_LEANCLR 1` ⇒ 那些路径**不参与编译**）；以及需要特定 ini / 未开启选项才可达的项。**换后端或改配置时须重新评估。** |
| **不可达** | **52** | 永不触发；**保留编号作为历史锚点**（约定"不得删编号去重排"） |
| **撤销（非缺陷）** | **32** | 30 条**可结案**；⚠️ **`06-…/01 F-CS1-004`、`06-…/01 F-CS1-026` 两条"存疑、不得结案"**；`01-…/02 F-REG-005` 已**恢复 P2、不在本类**；`01-…/04 F-PROP-007` 边界项**不得写入"已判定为非缺陷"**（见 §4.2） |
| **P2 / P3** | **267 / 381** | 待实机 profiling / ASan 后再排序；⚠️ 引用 `P3 = 381` 时带上 §4.3 的口径风险 |
| **未完成面（明确未做，不要当成"已验证"）** | — | ① `01-…/07` 的 P0/P1 重判；② `06-…/01`、`08-…/01` 的"死代码/未消费"结论**两棵树范围重跑**；③ `06-…/03` 自认的"**950 槽 ↔ `BINDING` key 逐条 diff**"（其自称"最大剩余风险面"）；④ 引文完整性全量复核**重跑并让 19 处坏引用归零**（当前 11/18 已修、**7/18 仍可命中，其中 2 处是"字面省略号"**）；⑤ `07-…/01` ↔ `05-…/01` 的 `TargetFramework` ↔ `DotNetVersion.h` **双向比对**；⑥ `Source/SourceCodeGenerator`（Program 模块）覆盖不足 |

---

## §7 数据边界与未取得项（如实声明）

| 项 | 状态 |
|---|---|
| 本文件所有聚合数字的来源 | **仅**逐条汇总 47 份报告正文的结果（2026-09-11 01:05:23）与 `09-问题清单/01-全局问题总表.md`（834 行）；**无一处人工推算** |
| Top 发现的"根因链 / 最小修复"来源 | 各 Finding 自身的 `严重度`/`可达性`/`复核结论`/`复核证据`/`问题`/`建议` 字段原文（逐条 `read` 提取），**行号一律照抄报告给出的值，未自行推断**；报告统一使用 `**问题**`/`**建议**` 标签，故"影响面"取自 `问题`/`调用上下文`/`复核证据` 段 |
| 引文行号吻合率 | 由 38 份的 `lines_sampled`/`lines_match` **直接相加**（870 / 839 = 96.44%）。未回填 `08/02b`、`08/05` 时该值为 764 / 738 = 96.6%，**两者不矛盾**，本文件采用当前值 |
| **未取得** | ① **6 条无 `可达性` 字段的条目编号未定位**（统计只输出计数、不打印明细）；② "标题/复核结论含撤销但 `严重度` 字段为 P0–P3"的**条目总数**（无此扫描器）；③ `INDEX.md` 的"858 个可分析文件 / 约 40 200 行"**未复算**；④ `08-…/02b`、`08-…/05` 的"未覆盖/存疑"清单内容（§5 曾为空表）；⑤ "19 处坏引用"中**第 19 处的具体条目**（引文完整性清单表只有 18 行）；⑥ 遗留三项格式指标（`调用上下文` 缺失率、`类别` 越界率、严重度排序违规）的**复测值** |
| 与其它文件的已知不一致（**本文件不代改**） | ① `09-…/01` 表头的复核结论分布仍是旧口径（`478/267/81/8`）；② 另有 `832 / P0 38` 的旧口径未同步；③ `INDEX.md` 是否已全部按 834 / P0 26 重写，**本文件未逐项核对**（§2 的 13 项另行处理） |
| 本文件的行为约束声明 | ✅ **只修改了这 1 个文件**（`00-总览/02-执行摘要与Top发现.md`）；**未修改** `INDEX.md`、`09-问题清单/**`、`00-总览/01`、`00-总览/03`、`08-专项审计/**`、任何模块报告与插件源码。所有数字均来自逐条汇总或实际读到的文件，**无编造**。 |
