# UnrealCSharp 插件深度分析 — 总索引

> **分析对象**：`Plugins/UnrealCSharp`（版本 1.2.0，作者 crazytuzi）
> **排除范围**：`Source/ThirdParty/`（Mono / CoreCLR / LeanCLR 三个 git 子模块）、`Intermediate/`、`Binaries/`、`*.old`、`Script/**/obj/`、`Script/**/bin/`
> **分析规模**：约 40,200 行 C++/C# 源码、858 个可分析文件
> **产出**：**45 份报告** + 本索引 + 3 份派生视图（[09-问题清单](09-问题清单/00-推荐优先修复清单.md)）
> **方法**：按「模块 × 横向专项」分工，**逐文件、逐函数、逐条发现**做**源码级复核**；所有结论附 `文件:行` 与 grep 证据
> **统一规范**：见 [`_CONVENTIONS.md`](_CONVENTIONS.md)（Finding 模板、严重度定义、死代码判定）
> **⚠️ 文档范围**：本索引与各报告只保留分析结果本身；各条结论的最终状态以报告内的 `复核结论` / `复核证据` / `可达性` / `级别变动` 字段为准，并参见本索引的「推翻/更正的关键结论」与「系统性方法学缺陷」两节。

---

## ⚠️ 使用本套文档前必读的 4 件事

### 0. **全部结论已做源码级逐条复核，并推翻了多份报告的"权威结论"**
已对**全部 47 份报告、834 条发现**做源码级复核（回到插件真实源码 `Plugins/UnrealCSharp/**`、真实 UE 5.6 引擎源码 `Engine\Source`、**LeanCLR 源码** `<LeanCLR 源码>`、**dotnet/runtime** `<dotnet/runtime 源码>`），并在可用时以**真实生成产物**（`Script/UE/Proxy/**`，**16 055** 份 `.cs`）为地面真值。

**权威统计（逐条汇总 47 份报告正文得出）**：

| 指标 | 复核后（当前） |
|---|---|
| 发现总数（**编号口径**） | **834** |
| 另：**未编号的子项/发现块** | **3**（`08-…/01` 的 `P3-a`/`P3-b` + `06-…/01` 的一条无编号 P3 块）⇒ 完整表述为"**834 条编号发现 + 3 条未编号子项**" |
| **P0** | **26** |
| P1 / P2 / P3 | **128 / 267 / 381** |
| **撤销（非缺陷，从排期移除）** | **32** |
| 复核结论 | 确认 **481** / 部分确认 **319** / 证伪 **26** / 无法验证 **3** |
| 可达性 | 活跃 **637** / 潜伏 **140** / 不可达 **52** |
| 复核结论字段非标准书写（未计入四分类） | **5** |

> ⚠️ **本表取代了旧版 INDEX 的两套互相矛盾的数字。** 旧版在此处给出 `P0=26`，并在另一处给出合计指向 749 与 829 的两组不同 P2/P3。上表由**逐条汇总 47 份报告正文**得到。
>
> 🔴 **统计口径（重新统计时务必注意）**：
> 1. **不能用"整行是否含 `P0`"判级别** —— 严重度的括号说明会提到别的级别（`**P2**（… 不再按"若触发则等同 P0"记账 …）`）⇒ 会虚高为 **P0=38**；
> 2. **要处理带箭头的改写形态** —— `~~P2~~ → **撤销（非缺陷）**` 取第一个 token 会得到已作废的 `P2` ⇒ 按本口径为 **P0=26**；
> 3. **不能用"整行是否含 `证伪`"判结论** —— 复核结论常写"部分确认（偏差：…一条**被证伪**…）" ⇒ 会虚高为 **证伪=76**，实为 **26**。

- **汇总入口**：[`09-问题清单/01-全局问题总表.md`](09-问题清单/01-全局问题总表.md)（全量 834 条）与 [`09-问题清单/00-推荐优先修复清单.md`](09-问题清单/00-推荐优先修复清单.md)
- **🔴 三条会改变结论方向的配置/引擎事实**：
  ① 本工程五平台全 **LeanCLR**（`Definitions.UnrealCSharpCore.h:19-21` = `WITH_CORECLR 0 / WITH_MONO 0 / WITH_LEANCLR 1`），Mono/CoreCLR 专属缺陷当前**不参与编译**（判「潜伏」）；
  ② **LeanCLR 下托管异常不终止进程**（走 `RtResult` 返回值协议，调用方拿到零值），**CoreCLR 下会 `TerminateProcess`**（`call 0` 已用 net10.0 探针实机复现 fail-fast，`try/catch` 无效）；且 **LeanCLR 下 C++→C# 不走本机函数指针**（`Bridge_Invoke` 走解释器）；
  ③ **`FString` 无 SSO**、**`TSet` 迭代是插入序**（不是哈希序）、**`FString::Equals` 默认 `CaseSensitive`**（而 `operator==` 是 `IgnoreCase`）、**`TMap::Remove` 不搬移元素**。

### 1. 🔴 **推翻/更正**的关键结论（引用旧结论前务必先看这张表）
发现**多处"已裁决/已撤销"的结论本身是错的**。已全部就地更正：

| 曾被当作权威的结论 | 复核后的事实 |
|---|---|
| "`Set`/`InitializeValue` 泄漏族的权威修复清单 = **7 个**活值 Dest 调用点" | **实为 4 个**：`FRegisterProperty.cpp:35`、`:64`、`FArrayHelper.cpp:137`、`FOptionalHelper.cpp:91`。另 4 个点（`FArrayHelper.cpp:269`、`FMapHelper.cpp:200/218`、`FSetHelper.cpp:102`）的 `Dest` 来自 `AddUninitialized`（`UnrealType.h:4015-4021`，**不构造**），**在那里补 `DestroyValue` 会触发 UB** |
| `F-REG-005` 已"撤销（非缺陷）" | **撤回，恢复 P2**（重入链每一跳在源码成立、路径唯一） |
| `F-CS1-004`、`F-CS1-026` 已"撤销（非缺陷）" | **撤销证据不足 → 改判「存疑」**，**不得**写入非缺陷清单 |
| "`FString::Equals` 默认 `ESearchCase` 无法裁决" | **实为 `CaseSensitive`**（`UnrealString.h.inl:1492`）。此前 0 命中，是因为搜了 `.h` 而实现在 **`.inl`** |
| `F-MACRO-006` 的"宏进无花括号 `if/else` 即 `error C2062`" | **证伪**：本机 MSVC 14.51 + clang 20 四变体实测**均编译通过**；真实失败条件是调用处多写分号（`C2181`） |
| `F-LEAK-012` 的"元素堆缓冲泄漏"及其修法 | **一半证伪**：缓冲由分配器基类析构释放（`ContainerAllocationPolicies.h:682-693`）；且**原文建议的 `FMemory::Free(ScriptArray)` 修法方向是反的**，会**制造**它想修的泄漏 |
| `F-DOM-002`（use-after-`hostfxr_close`）P0 | **前提证伪**（`fx_muxer.cpp:182` "we do not unload coreclr"）→ **P3**；CoreCLR 真正的失败态是 `F-DOM-005` |
| `F-ABI-022`（无 BOM → ANSI 误读） | **撤销**：UBT **无条件**加 `/utf-8`（`VCToolChain.cs:650`）并紧跟 `/wd4819`（`:653`） |
| `F-ABI-021`（Linux 上模块不被发现） | **前提证伪**：UBT 的扩展名判定显式大小写不敏感（`FileSystemReference.cs:31`）→ P2→P3 |
| `06-…/03` 的"214−211 = **3** 个死注册" | **实为 1 个**：原 grep **只查了插件自身的 `Script/` 树**，漏掉工程侧生成产物 |
| "LeanCLR 完全没有 `DOTNET_*` 定义" | **错误**：`LeanCLR.Build.cs:11-22` 定义 **10/0/4**；真实口径是 Mono 10.0.1 / CoreCLR 10.0.4 / LeanCLR 10.0.4 |
| "`FReflectionRegistry.cpp:850-1811` 无报告覆盖 = 最大空洞" | **误判**：该区间已由存活的 `02-…/01` 覆盖（§0 声明 855–994 逐行读 + 995–2409 等价校验，区间内有 10 处独立行号引用） |
| "生成产物 16 384 份" | **实测 16 055**（`Script/UE/Proxy`）+ 75（`Script/Game/Proxy`），两目录**非 `.cs` 文件均为 0** |
| LeanCLR 下 `SynchronizationContext.Tick` 疑恒为空操作（曾被建议立 P0） | **证伪**：根因是**跨后端同名成员类型不同构**（LeanCLR 下该成员是 `const RtMethodInfo*`，非函数指针）——"某后端注册函数缺席"**不能**推出"成员无效" |

### 2. **发现编号存在 31 处跨报告重复，不要裸引用编号**
`F-INT2-*`（`01-…/10b` 与 `01-…/11`）**16 处**、`F-PROP2-*`（`01-…/04b` 与 `01-…/05`）**15 处**：
**同一编号在两份报告里指向不同发现**。834 条发现只有 **803 个唯一编号**（= 834 − 31）。
**引用时请始终带上报告路径**，例如 `01-UnrealCSharp运行时/10b-…md#F-INT2-001`，而不是只写 `F-INT2-001`。

### 3. 本机**有** UE 5.6 引擎源码，且它推翻了多份报告的降级理由
引擎源码在 `Engine\Source\`（约 20989 个 `.cpp`）。
但有 **9 份报告**声明"本机无引擎源码"并据此把 30+ 条结论降级为"置信度中/低"。**这些声明全部是错的**，已逐条回引擎源码核销。

### 4. 🔴 系统性方法学缺陷（引用任何"0 命中/不存在"结论前请自查）
| 陷阱 | 说明 |
|---|---|
| **两棵 `Script/` 树** | 插件自身 `Plugins/UnrealCSharp/Script/`（342 个 `.cs`）**与**工程侧生成产物 `<Project>/Script/`（16 055 + 75）；"某符号 0 命中/无消费者"类结论**必须两棵树都查** |
| **两种行数口径** | 总行数（含空行）vs `Measure-Object -Line`（**忽略空行**）。多处"行数矛盾"的统一根因（实测样例 160/98、142/92、227/144）。**两口径都可能是对的** |
| **同名/近似文件找错** | `UnrealString.h` vs **`.inl`**；`CoreUserObject` vs `CoreUObject`；`Public/` vs `Private/` 颠倒 |
| **三种根路径歧义** | 报告中的 `Source/` 与 `Script/` 可能分别指**插件根 / 引擎根 / 工程根**（78 处歧义） |
| **"已裁决"不是免检凭证** | 聚合型结论（修复清单、范围数字、计数口径）**必须回原始代码复验**；据此推翻了 12 项 |
| **错误撤销 > 高估** | 风险不对称：高估浪费工时，**错误撤销让缺陷永久漏修**。故"撤销（非缺陷）"是最高抽验优先级 |

---

## 快速导航

### 严重度定义
| 严重度 | 含义 |
|---|---|
| **P0** | 崩溃 / 数据损坏 / 内存破坏 / 进程终止 |
| **P1** | 功能错误 / 资源泄漏 / 跨平台不可用 |
| **P2** | 性能问题 / 潜在隐患 |
| **P3** | 风格 / 可读性 / 微优化 |
| **撤销** | 经复核判定为非缺陷，**已从排期移除**（保留编号作为历史锚点） |

### 类别定义
| 类别 | 含义 |
|---|---|
| Bug | 明确的逻辑错误 |
| 内存/资源泄漏 | 分配未释放、句柄未关闭、容器只增不减 |
| 性能 | 热路径开销、算法复杂度、重复分配 |
| 并发/线程安全 | 数据竞争、UI 跨线程、死锁 |
| 死代码 | 全插件无调用点的函数/宏/文件 |
| 可优化/可读性 | 重复代码、可模板化、命名与注释 |
| 未定义行为 | 别名违规、悬垂引用、有符号溢出 |
| 平台兼容 | Windows/Linux/Mac/Android/iOS 差异、ABI |
| 安全 | 路径穿越、格式串、DLL 劫持 |

### 建议阅读顺序
| 你的目的 | 读这些 |
|---|---|
| **想知道先修哪些** | ⭐ [09-问题清单/00-推荐优先修复清单.md](09-问题清单/00-推荐优先修复清单.md) |
| **想知道每条到底是不是真的 / 最终级别** | ⭐ [09-问题清单/01-全局问题总表.md](09-问题清单/01-全局问题总表.md)（834 条，级别已复核，含复核结论与可达性） |
| **只想知道最该修什么** | [00-总览/02-执行摘要与Top发现.md](00-总览/02-执行摘要与Top发现.md) |
| **想快速理解架构** | [00-总览/01-架构与数据流总览.md](00-总览/01-架构与数据流总览.md) |
| **想知道哪些是死代码** | [08-专项审计/01-全局死代码与死宏审计.md](08-专项审计/01-全局死代码与死宏审计.md) |
| **想知道哪里会泄漏** | [08-专项审计/02-资源泄漏审计（内存句柄与容器）.md](08-专项审计/02-资源泄漏审计（内存句柄与容器）.md) + [08-专项审计/02b-UObject生命周期与绑定配对审计.md](08-专项审计/02b-UObject生命周期与绑定配对审计.md) |
| **想知道并发风险** | [08-专项审计/03-线程安全与并发审计.md](08-专项审计/03-线程安全与并发审计.md) |
| **想优化性能** | [08-专项审计/05-性能热点与优化清单.md](08-专项审计/05-性能热点与优化清单.md) |
| **想修某个具体模块** | 按下面的目录表定位；各报告内发现均按 P0 → P3 排序 |

---

## 目录总表

> 「发现数」为**当前实测值**（表格式 + 标题式去重后）。

### 00-总览
| 文档 | 内容 | 发现数 |
|---|---|---|
| [00-总览/01-架构与数据流总览.md](00-总览/01-架构与数据流总览.md) | 模块地图与依赖图、**6 条端到端数据流**（逐跳已回源码核对）、互操作边界全景（三后端差异表，含 **C++→C# 通道**与**托管异常语义**两行）、关键类清单、架构级发现 | 10 |
| [00-总览/02-执行摘要与Top发现.md](00-总览/02-执行摘要与Top发现.md) | Top 发现、逐报告统计、跨模块缺陷族、已证伪/健康结论、修复路线图 | 派生视图 |
| [00-总览/03-修复优先级评估.md](00-总览/03-修复优先级评估.md) | 修复优先级的完整论证：评分维度与公式、门控规则、去重规则、按裁决校正定级对照表、逐条根因链与依赖关系 | 派生视图 |

### 01-UnrealCSharp 运行时（绑定运行时层）
| 文档 | 覆盖内容 | 发现数 |
|---|---|---|
| [01-…/01-Environment与绑定注册表.md](01-UnrealCSharp运行时/01-Environment与绑定注册表.md) | `FCSharpEnvironment`、`FCSharpBind`、`FBindingRegistry` | 21 |
| [01-…/02-对象引用结构体注册表.md](01-UnrealCSharp运行时/02-对象引用结构体注册表.md) | `FClassRegistry`、`FObjectRegistry`、`FReferenceRegistry`、`FStructRegistry`、`FMultiRegistry`、`FDynamicRegistry`、`FReference` 族 | 18 |
| [01-…/03-容器委托字符串注册表与模块入口.md](01-UnrealCSharp运行时/03-容器委托字符串注册表与模块入口.md) | `FContainerRegistry`、`FDelegateRegistry`、`FStringRegistry`、`FOptionalRegistry`、`FUObjectListener`、`FDomain`、模块入口 | 14 |
| [01-…/04-属性描述符-基类与基本类型.md](01-UnrealCSharp运行时/04-属性描述符-基类与基本类型.md) | `FPropertyDescriptor` 基类 + 11 个 Primitive 描述符（模板在 `TPrimitivePropertyDescriptor.inl`） | 7 |
| [01-…/04b-属性描述符-字符串枚举结构体Optional.md](01-UnrealCSharp运行时/04b-属性描述符-字符串枚举结构体Optional.md) | 5 个 String + Enum + Struct + Optional 描述符 ⚠️ 与本目录 `05` **重号** | 16 |
| [01-…/05-属性描述符-对象与委托类型.md](01-UnrealCSharp运行时/05-属性描述符-对象与委托类型.md) | 8 个 ObjectProperty + 3 个 DelegateProperty + FieldPath ⚠️ 与本目录 `04b` **重号** | 15 |
| [01-…/06-函数与类描述符.md](01-UnrealCSharp运行时/06-函数与类描述符.md) | `FFunctionDescriptor` 家族、`FCSharpFunctionRegister`、**`FFunctionParamBufferAllocator`**、`FClassDescriptor` | 16 |
| [01-…/07-容器Helper.md](01-UnrealCSharp运行时/07-容器Helper.md) | `FArrayHelper` / `FMapHelper` / `FSetHelper` | 30 |
| [01-…/07b-委托Handler与OptionalHelper.md](01-UnrealCSharp运行时/07b-委托Handler与OptionalHelper.md) | `DelegateHandler`、`MulticastDelegateHandler`、`FDelegate{Base,}Helper`、`FDelegateWrapper`、`FMulticastDelegateHelper`、`FOptionalHelper` | 18 |
| [01-…/08-绑定层公开头文件（模板元编程）.md](01-UnrealCSharp运行时/08-绑定层公开头文件（模板元编程）.md) | `Public/Binding/**` 全部 27 个头文件；**含最锋利的活跃 P0（`F-BIND-001` 出缓冲步长错配）** | 12 |
| [01-…/09-绑定宏与CoreMacro对照.md](01-UnrealCSharp运行时/09-绑定宏与CoreMacro对照.md) | `Public/Macro/**` 5 个宏头 + 与 `CoreMacro/**` 的重名核对（宏展开用本机 MSVC/clang 实测，**已复现**） | 13 |
| [01-…/10-Interop注册-对象与反射类.md](01-UnrealCSharp运行时/10-Interop注册-对象与反射类.md) | `FRegister{Object,Class,Struct,Property,Function,ScriptInterface}.cpp` + 注册名↔目标符号核对表（**52/52 一致，独立重跑**；UE 5.6 下实际生效 51） | 13 |
| [01-…/10b-Interop注册-容器字符串与对象指针.md](01-UnrealCSharp运行时/10b-Interop注册-容器字符串与对象指针.md) | `FRegister{Array,Map,Set,Optional,String,Name,Text,AnsiString,Utf8String,Delegate,MulticastDelegate,*ObjectPtr,SubclassOf}.cpp` ⚠️ 与本目录 `11` **重号** | 16 |
| [01-…/11-Interop注册-数学值类型与引擎类.md](01-UnrealCSharp运行时/11-Interop注册-数学值类型与引擎类.md) | 36 个值类型/数学/资产/引擎类注册 + **`EKeys` 全量审计**（引擎 348 具名常量 vs 插件 342 条，UE 5.6 生效 335 ⇒ **漏 13**，其中 **12 条是成体系漏掉的 `*_2D` 配对轴**） ⚠️ 与本目录 `10b` **重号** | 28 |

### 02-UnrealCSharpCore 核心
| 文档 | 覆盖内容 | 发现数 |
|---|---|---|
| [02-…/01-反射注册表FReflectionRegistry.md](02-UnrealCSharpCore核心/01-反射注册表FReflectionRegistry.md) | `FReflectionRegistry.h/.cpp`（真实总行数 **1194 / 2409**） | 9 |
| [02-…/02-反射模型（类字段方法参数属性）.md](02-UnrealCSharpCore核心/02-反射模型（类字段方法参数属性）.md) | `FReflection`、`FClassReflection`、`FFieldReflection`、`FMethodReflection`、`FParamReflection`、`FPropertyReflection` | 15 |
| [02-…/03-动态类型生成.md](02-UnrealCSharpCore核心/03-动态类型生成.md) | `FDynamicGenerator*`、`FDynamicDependencyGraph`、`DynamicBlueprintExtension`、`DynamicScriptStruct` | 21 |
| [02-…/04-脚本域抽象与三大后端.md](02-UnrealCSharpCore核心/04-脚本域抽象与三大后端.md) | `IScriptDomain` 抽象 + **三后端逐项对照（Mono/CoreCLR/LeanCLR 各 30/30 全部有真实实现，已独立重跑）** | 28 |
| [02-…/05-绑定注册与类型信息.md](02-UnrealCSharpCore核心/05-绑定注册与类型信息.md) | `FBinding*`、`FBindingTypeInfo*`、`TName`/`TTypeInfo`/`TDefaultArgument` 等模板（38 文件） | 21 |
| [02-…/06-类型桥接编码设置与引擎监听.md](02-UnrealCSharpCore核心/06-类型桥接编码设置与引擎监听.md) | `FTypeBridge`、`NameEncode`、`UnrealCSharpSetting`、`FEngineListener`、模块入口、`UnrealCSharpCore.build.cs` | 21 |
| [02-…/07-函数库宏与模板Trait.md](02-UnrealCSharpCore核心/07-函数库宏与模板Trait.md) | `FUnrealCSharpFunctionLibrary`（**1615 行**）+ `CoreMacro/**`（16 个）+ `Template/**` trait；**`TFieldIteratorExt` 已与引擎实际分叉且无版本守卫** | 27 |

### 03-代码生成器（ScriptCodeGenerator）
| 文档 | 覆盖内容 | 发现数 |
|---|---|---|
| [03-…/01-代码生成框架与模块入口.md](03-代码生成器/01-代码生成框架与模块入口.md) | `FGeneratorCore`（**1079 行**）、模块 Build.cs；**生成触发入口实测 7 条**（非 4 条） | 14 |
| [03-…/01b-类生成器与绑定类生成器.md](03-代码生成器/01b-类生成器与绑定类生成器.md) | `FClassGenerator`（**1450 行**）、`FBindingClassGenerator`（**1012 行**） | 29 |
| [03-…/02-委托结构体枚举与GameplayTag生成器.md](03-代码生成器/02-委托结构体枚举与GameplayTag生成器.md) | `FDelegateGenerator`、`FStructGenerator`、`FEnumGenerator`、`FBindingEnumGenerator`、`FGameplayTagGenerator`（用真实生成物作地面真值） | 16 |
| [03-…/03-代码分析Doxygen转换与解决方案生成.md](03-代码生成器/03-代码分析Doxygen转换与解决方案生成.md) | `FCodeAnalysis`、`FDoxygenConverter`、`FSolutionGenerator`、`FAssetGenerator` | 18 |

### 04-编辑器模块（UnrealCSharpEditor）
| 文档 | 覆盖内容 | 发现数 |
|---|---|---|
| [04-…/01-内容浏览器与新建类向导.md](04-编辑器模块/01-内容浏览器与新建类向导.md) | `DynamicDataSource`、`DynamicHierarchy`、`DynamicNewClassContextMenu`、`ClassCollector`、`SDynamicNewClassDialog`、`SDynamicClassViewer` | 23 |
| [04-…/02-工具栏监听器进度对话框与模块入口.md](04-编辑器模块/02-工具栏监听器进度对话框与模块入口.md) | `FEditorListener`、工具栏、`SCompileProgressDialog`、Commandlet、Path 定制；**含 7 组注册↔反注册成对表（逐行核对）**。原 4 条 P0 复核后**只剩 2 条成立** | 27 |

### 05-编译器与跨版本
| 文档 | 覆盖内容 | 发现数 |
|---|---|---|
| [05-…/01-编译器CSharp编译与跨版本宏.md](05-编译器与跨版本/01-编译器CSharp编译与跨版本宏.md) | `Compiler` 模块（`FCSharpCompilerRunnable` **483 行**）、`CrossVersion` 4 个版本宏头、`SourceCodeGenerator.cs`（**873 行**）；**含句柄配对表（12 行逐行核对，无泄漏）**与 TargetFramework 交叉表 | 33 |

### 06-CSharp 运行时与脚本（`Script/`）
| 文档 | 覆盖内容 | 发现数 |
|---|---|---|
| [06-…/01-Interop桥接与程序集加载.md](06-CSharp运行时与脚本/01-Interop桥接与程序集加载.md) | `AssemblyLoader`、`UnrealAssemblyLoadContext`、`Bridge/*.cs`（8 个）、`HandleData`、`UnrealTypeWeaver.cs`（1185 行）；**本模块 P0 = 0** | 40 |
| [06-…/02a-容器与指针包装（资源所有权）.md](06-CSharp运行时与脚本/02a-容器与指针包装（资源所有权）.md) | `TArray`/`TMap`/`TSet`/`TOptional` + 7 个指针包装类；含**资源所有权表**与 3 张 P/Invoke 对照表 | 23 |
| [06-…/02b-对象字符串与工具类型包装.md](06-CSharp运行时与脚本/02b-对象字符串与工具类型包装.md) | 20 个对象/字符串/工具类型 + `Reflection/Property/` 9 个；**含已证伪线索记录（LeanCLR Tick 误报的推理陷阱）** | 31 |
| [06-…/03-引擎实现库Library（跨语言ABI契约）.md](06-CSharp运行时与脚本/03-引擎实现库Library（跨语言ABI契约）.md) | `Script/UE/Library/*.cs`（28 个）；**三方名字对账重跑：C++ 214 / C# 211 + 生成侧 2 = 213 ⇒ 仅 1 条不一致** | 10 |
| [06-…/04-Dynamic特性源生成器与模板工程.md](06-CSharp运行时与脚本/04-Dynamic特性源生成器与模板工程.md) | `Script/UE/Dynamic/**`（**实测 255 个特性类**）、`Script/SourceGenerator/`、`Template/**` | 20 |

### 07-构建与配置
| 文档 | 覆盖内容 | 发现数 |
|---|---|---|
| [07-…/01-构建脚本模块依赖与插件配置.md](07-构建与配置/01-构建脚本模块依赖与插件配置.md) | 9 个模块规则文件（7 大写 + 2 小写 `build.cs`）、`UnrealCSharp.uplugin`、C# 工程文件、`.github/`；**无 P0/P1**；**含依赖↔使用核对表（逐行重跑）** | 21 |

### 08-专项审计（横向，跨模块）
| 文档 | 覆盖内容 | 发现数 |
|---|---|---|
| [08-…/01-全局死代码与死宏审计.md](08-专项审计/01-全局死代码与死宏审计.md) | 全量文件扫描：死宏、无实现头文件、未被引用文件、未被消费的导出 API、TODO、后端功能缺口。**口径：11 条编号发现 + 2 条未编号 P3 子项 = 13** | 11 |
| [08-…/02-资源泄漏审计（内存句柄与容器）.md](08-专项审计/02-资源泄漏审计（内存句柄与容器）.md) | **4 张配对表**（逐行核对，无一行被整行推翻）：C++ 堆内存、托管句柄与跨语言内存、OS 资源句柄、容器只增不减 | 15 |
| [08-…/02b-UObject生命周期与绑定配对审计.md](08-专项审计/02b-UObject生命周期与绑定配对审计.md) | **4 张配对表**：根引用配对、裸指针 GC 保护判定、委托解绑配对（42 行）、共享指针判定 | 19 |
| [08-…/03-线程安全与并发审计.md](08-专项审计/03-线程安全与并发审计.md) | **线程边界地图**（13 个线程/入口，逐行核实）、**共享可变状态清单**（24 项）、跨线程 bug、死锁、初始化竞态。**原 4 条 P0 复核后全部降为 P1** | 24 |
| [08-…/04-跨语言ABI与平台兼容审计.md](08-专项审计/04-跨语言ABI与平台兼容审计.md) | **C++↔C# 配对表 A-D**、平台分支审查、UB 与不安全模式、**后端门控表（含补正）**。**原 5 条 P0 复核后全部降级，P0 现为 0** | 39 |
| [08-…/05-性能热点与优化清单.md](08-专项审计/05-性能热点与优化清单.md) | **热路径清单表**、优化点与量化收益、优先级分组、反优化清单。`F-PERF-007/008`（TMap/TSet 未用哈希）的**可行性已升为"确定"**（`PropertyMap.cpp:1666-1675` 实证），但**收益仍标未量化** | 30 |

> 注：各条的最终级别与结论见报告内的 `复核结论` / `可达性` 字段；全量汇总见 [09-问题清单/01-全局问题总表.md](09-问题清单/01-全局问题总表.md)。

---

### 09-问题清单（派生视图：薄索引）

> 这一章**不产生新发现**——只做「一行一条」的清单索引。根因、现状代码、调用上下文、修复建议、验证方式全部留在 00–08 的原报告里。

| 文档 | 内容 |
|---|---|
| [09-问题清单/00-推荐优先修复清单.md](09-问题清单/00-推荐优先修复清单.md) | **最终修复顺序**：级别分布、立刻修（P0）、计划修（P1）、修复顺序建议、**已判定为非缺陷清单（注意含"存疑、勿结案"项）** |
| [09-问题清单/01-全局问题总表.md](09-问题清单/01-全局问题总表.md) | **全量 834 条**：编号 / 级别 / 复核结论 / 可达性 / 模块码 / 一句话问题 / 原文链接（**逐条按固定规则提取**） |

> **口径说明**：全部数字以**逐条汇总 47 份报告正文**为准，取代旧版 INDEX 中两套互相矛盾的人工统计。
> **编号重号警告**：31 处跨报告编号重复（`F-INT2-*` 16 处、`F-PROP2-*` 15 处），引用编号请务必带上报告路径。

---

## 模块与源码规模

| 模块 | 类型 | 加载阶段 | 源码文件数 | 说明 |
|---|---|---|---|---|
| `Source/UnrealCSharp` | Runtime | Default | 261 | C++↔C# 绑定运行时（Environment / Registry / Reflection / Interop） |
| `Source/UnrealCSharpCore` | Runtime | Default | 164 | 反射注册表、动态类型生成、脚本域抽象、绑定注册 |
| `Source/UnrealCSharpEditor` | Editor | Default | 43 | 内容浏览器、新建类向导、工具栏、编译进度、Commandlet |
| `Source/ScriptCodeGenerator` | Editor | Default | 29 | 扫描 C++ 反射数据 → 生成 C# 绑定代码 |
| `Source/Compiler` | Editor | Default | 9 | 后台 `dotnet build` 编译 C# 程序集 |
| `Source/CrossVersion` | Runtime | Default | 7 | UE / C++ / .NET / VS 版本兼容宏 |
| `Source/SourceCodeGenerator` | Program | PostConfigInit | 1 | UHT 导出器（`[UhtExporter]`） |
| `Script/`（插件侧） | — | — | **342** 个 `.cs`（实测） | C# 侧运行时（`UE/`、`Interop/`、`SourceGenerator/`、`Weavers/`、`CodeAnalysis/`）与 `Template/` |
| 生成产物（工程侧） | — | — | **16 055**（`Script/UE/Proxy`）+ **75**（`Script/Game/Proxy`），非 `.cs` 文件均为 0 | 真实生成的地面真值 |
| **合计**（排除 ThirdParty / Intermediate / Binaries） | | | 858 个可分析文件 / 约 40,200 行 | |

> **重要运行时事实**：
> - 跨语言机制**不是** `extern "C"` + `DllImport`，而是 **`FClassBuilder` 注册函数指针 + C# 侧 `MethodBridge` 名字字典派发**（`delegate* unmanaged[Cdecl]`），**且不校验参数个数/类型**。
> - 三个脚本后端由 `.Build.cs` 的编译期宏**互斥**启用；当前工程五平台**全部 LeanCLR**，故 Mono/CoreCLR 专属发现判「潜伏」。
> - 全部结论均为**静态阅读 + 引擎源码比对**，**未编译、未运行、未压测**。需要实机验证的项已在各报告的"未覆盖/存疑"章节逐条标注。

---

## 目录结构

```
UnrealCSharpAudit/
├── INDEX.md                        ← 本文件（总索引）
├── _CONVENTIONS.md                 ← 统一分析规范（Finding 模板 / 严重度 / 死代码判定）
├── 00-总览/                        ← 架构总览、执行摘要、修复优先级（3 份）
├── 01-UnrealCSharp运行时/           ← 14 份
├── 02-UnrealCSharpCore核心/         ← 7 份
├── 03-代码生成器/                   ← 4 份
├── 04-编辑器模块/                   ← 2 份
├── 05-编译器与跨版本/                ← 1 份
├── 06-CSharp运行时与脚本/            ← 5 份
├── 07-构建与配置/                   ← 1 份
├── 08-专项审计/                     ← 6 份（死代码 / 泄漏 / 生命周期 / 并发 / ABI / 性能）
└── 09-问题清单/                     ← 薄索引（2 份）
```

> 合计 **45 份模块报告 + 本索引 + `_CONVENTIONS.md`**。
> 各报告的最终结论以报告内的复核字段为准，并已汇总到 [09-问题清单/01-全局问题总表.md](09-问题清单/01-全局问题总表.md)。
