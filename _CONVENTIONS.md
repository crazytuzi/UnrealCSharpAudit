# 分析规范与报告格式约定（所有报告撰写者必读）

> 本文件是 UnrealCSharp 插件深度分析项目的**统一规范**。所有报告撰写者必须先读完本文件，再开始自己的模块分析，并严格按本文件的格式产出报告。

---

## 0. 项目背景

- **分析目标**：`Plugins/UnrealCSharp`
- **排除范围**：`Source\ThirdParty\`（子模块，不分析）、`Intermediate\`、`Binaries\`、`*.old`、`Script\**\obj\`、`Script\**\bin\`
- **插件性质**：UnrealCSharp —— 让 UE 用 C# 写游戏逻辑的插件。支持三种脚本后端：**Mono**、**CoreCLR**、**LeanCLR**。
- **版本**：`UnrealCSharp.uplugin` VersionName = 1.2.0，作者 crazytuzi
- **模块划分**：
  | 模块 | 类型 | 职责 |
  |---|---|---|
  | `Source/UnrealCSharp` | Runtime | C++ ↔ C# 绑定运行时：Environment、Registry（对象/引用注册表）、Reflection（属性/函数描述符）、Domain/Interop（P/Invoke 注册） |
  | `Source/UnrealCSharpCore` | Runtime | 核心：反射注册表、动态类生成、脚本域抽象（Mono/CoreCLR/LeanCLR）、绑定注册 |
  | `Source/UnrealCSharpEditor` | Editor | 编辑器：内容浏览器、新建类向导、工具栏、编译进度、Commandlet |
  | `Source/ScriptCodeGenerator` | Editor | C++ 扫描并生成 C# 绑定代码 |
  | `Source/Compiler` | Editor | 调用 dotnet build 编译 C# 程序集 |
  | `Source/CrossVersion` | Runtime | UE / C++ / .NET / VS 版本兼容宏 |
  | `Source/SourceCodeGenerator` | Program | C# 源码生成器（独立程序） |
  | `Script/` | - | C# 侧运行时（`UE/`、`Interop/`、`SourceGenerator/`、`Weavers/`、`CodeAnalysis/`）与 `Template/` |

---

## 1. 报告输出位置与命名

根目录：`UnrealCSharpAudit`（本仓库根；各报告的相对链接均以其为基准）

子目录结构（**每个撰写者只写自己指定的那一个文件，不要动别人的文件**）：

```
UnrealCSharpAudit/
├── INDEX.md                     ← 总索引（统一维护，不要自行修改）
├── 00-总览/
├── 01-UnrealCSharp运行时/
├── 02-UnrealCSharpCore核心/
├── 03-代码生成器/
├── 04-编辑器模块/
├── 05-编译器与跨版本/
├── 06-CSharp运行时与脚本/
├── 07-构建与配置/
└── 08-专项审计/                  ← 死代码、泄漏、性能、并发
```

文件名格式：`NN-简短英文或拼音标识.md`，例如 `01-Environment与Registry.md`。

---

## 2. 每条发现（Finding）的强制格式

**这是最重要的部分。** 每条发现必须是一个独立小节，用如下模板。不允许写"这里可能有问题"这类没有定位的猜测。

```markdown
### [F-<模块前缀>-<序号>] <一句话标题>

- **类别**: Bug | 内存/资源泄漏 | 性能 | 并发/线程安全 | 死代码 | 可优化/可读性 | 未定义行为 | 平台兼容 | 安全
- **严重度**: P0(崩溃/数据损坏) | P1(功能错误/泄漏) | P2(性能/隐患) | P3(风格/可读性)
- **文件**: `相对插件根的路径:行号`  ← 例如 `Source/UnrealCSharp/Private/Registry/FCSharpBind.cpp:123`
- **函数**: `全限定签名`（如 `FCSharpBind::Bind(UClass*)`）
- **置信度**: 高 / 中 / 低（低置信度必须说明为什么不确定）

**现状（代码事实）**
```cpp
// 贴出关键代码片段，必须带真实行号注释，禁止凭记忆改写
```

**调用上下文**
谁调用它、它调用谁、在什么线程/生命周期阶段被调用（初始化？Tick？GC？销毁？）。必须给出证据：
调用方文件:行。

**问题**
为什么这是问题。给出具体触发路径或数据流。若是泄漏，说明分配点与缺失的释放点。

**建议**
具体的修改方案（可以是代码片段）。若涉及权衡，说明取舍。

**验证方式**
如何确认（grep 什么符号、跑什么用例、加什么 assert）。
```

### 排序规则
报告中所有发现按 **严重度降序**（P0 → P3）排列，同级别按文件路径字典序。

---

## 3. 必须覆盖的"横向维度"

对负责的每个文件，除了逐函数分析，还要专门检查以下维度，并在报告末尾用独立章节汇总：

1. **空指针 / 越界 / 未检查返回值**
   - `Cast<>` 结果、`FindObject`、`LoadObject`、容器 `operator[]`、`GetData()`、`Memcpy` 长度
   - `FindPropertyByName(...)` 等返回 `nullptr` 后是否解引用
   - C# 侧 `IntPtr` 是否为 `IntPtr.Zero` 校验
2. **内存与资源泄漏**
   - `new` / `FMemory::Malloc` / `malloc` / `GCHandle` / `Alloc` 与匹配的 `delete`/`Free`/`FreeGCHandle`
   - `TSharedPtr`/`TUniquePtr` 循环引用、`TStrongObjectPtr`、`AddToRoot`/`RemoveFromRoot` 配对
   - UObject 引用是否被 `UPROPERTY()` 或 `AddReferencedObjects` 保护（GC 悬挂）
   - C# 侧 `IDisposable` / `Dispose` / 句柄释放路径
   - 委托（delegate）绑定后是否解绑，`AddDynamic` / `AddUniqueDynamic` / `RemoveDynamic` 配对
3. **线程安全**
   - `FCSharpEnvironment`/Registry 是否假设 GameThread；是否有 `check(IsInGameThread())`；异步编译线程（`FCSharpCompilerRunnable`）与主线程共享数据的锁
   - 静态/全局可变状态
4. **异常与错误处理**
   - C++ 异常穿越托管边界、`try/catch` 是否吞掉错误、`ensure`/`check` 是否会在 Shipping 崩溃
5. **性能**
   - 每帧调用的热路径中的 `FString` 拼接、`FName` 构造、`TMap` 查找、字符串比较、重复 `FindProperty`/`StaticClass()` 查找
   - 可缓存的反射元数据、可按引用传递的 `const T&` 参数（当前是否按值传大对象）
   - 循环内的重复分配
6. **死代码**
   - 该函数在**整个插件源码**（`Source/` 全部 7 个模块 + `Script/`）中没有任何调用点、也没有被导出为 public API 供外部使用
   - 判定方法：用 grep 搜函数名（去掉类名限定），统计出现次数；`==1`（仅声明/定义）视为强死代码嫌疑；声明在 `.h` 的 public 区域且插件有对外 API 属性时标为"可能是外部 API，需确认"
   - **必须给出 grep 证据**（搜索模式 + 命中数）
7. **可优化 / 可读性 / 一致性**
   - 重复代码块、可以模板化/宏化的重复实现
   - 命名不一致、magic number、注释与代码不符（**特别注意：注释说的和代码做的不一致时要单独指出**）
   - 可 `constexpr`/`static`/`inline` 化的东西
8. **平台兼容**
   - 32/64 位、Windows/Linux/Mac/Android/iOS 差异（`PLATFORM_*` 宏）、字节序、`int` 宽度假设、`sprintf` 用法

---

## 4. 事实纪律（非常重要）

- **只报告你真正读过的代码**。行号必须来自你读取到的实际内容，禁止估算。
- 使用 `read` 工具（带 offset/limit 读长文件）、`grep` 工具（不要用 shell grep）。
- **不要修改任何插件源码**。你只写自己的那份 md 报告。
- 不确定就标 `置信度: 低` 并写清"未验证的假设"。
- 不要为了凑数编造发现。**宁少而准，不多少而虚**。文件确实健康，就写"未发现 P0/P1 级问题"，并列出你检查过的维度。
- 如果某个文件你因为太长没读完，必须在报告开头"覆盖范围"章节明确写出"读了哪些行范围、哪些没读"。

---

## 5. 报告文件骨架（每个报告文件都用这个结构）

```markdown
# <报告标题>

> 分析范围：<模块/目录>
> 覆盖文件：N 个（逐一列出，或列出 glob）
>
> 状态：已完成 / 部分完成（说明未覆盖部分）

## 0. 覆盖范围与阅读清单
| 文件 | 行数 | 是否读完 | 备注 |

## 1. 模块职责与架构速览
（这一段要有信息量：数据流、关键类关系、生命周期）

## 2. 关键调用链
（列出 3-8 条真实调用链，格式：`A.cpp:12 → B::Fn → C.cpp:34`）

## 3. 发现清单
### P0
### P1
### P2
### P3
（用第 2 节的 Finding 模板）

## 4. 死代码清单
| 符号 | 声明位置 | grep 命中数 | 判定 | 证据 |

## 5. 未覆盖/存疑项
```

---

## 6. 报告撰写效率提示

- 长文件用 `read` 的 `offset`/`limit` 分段读，**不要**试图一次读完 1800 行。
- `grep` 的 `include` 参数只接受**一个** glob，不支持列表；需要多后缀时多次调用。
- 行号从 `read` 输出里抄，不要自己数。
- 报告写完后，**在最终消息里**给出：报告路径、发现数量（按严重度分）、最重要的 3 条发现的一句话摘要、以及任何未能覆盖的部分。

---

## 7. 【硬规范】新调用的接口：先做版本兼容性检测，再落地（2026-10-16 由轮 B-29 确立）

> **一句话**：插件的支持区间是 **UE 5.0–5.8**、且必须 **C++17 可编**；因此**任何新增或改动的引擎接口调用（函数/方法/枚举值/类型/宏/include 路径）与任何新写法（语法、模板用法），在落地前都必须逐条给出"最低支持版本"依据**。拿不出依据 = 不许合入。
> 🔴 **"本机 5.6 编得过"不是依据**：轮 B-29 的 8 条编译错误（[`08-…/69`](08-专项审计/69-跨版本编译兼容收口-UE5.0八处报错与版本门控（2026-10-16）.md)）在 5.6 / C++20 上**一条都不出现**——它们只在 **UE 5.0** 上暴露。审计库此前所有报告都在 5.6 上完成，因此"跨版本通用"的表述此前都只是**头文件级推断**。

### 7.1 每处新调用必做三步

| 步 | 做什么 | 证据要求 |
|---|---|---|
| **1. 查可用性** | 该符号在**哪个 release tag 首次出现**；以及**可见性**（`public`/`protected`/`private`） | **有符号 ≠ 可调用**：`UUserWidget::UpdateCanTick()` 在 5.0.0–5.0.3 **存在但为 private**（5.1 起才 public）；`FEnhancedInputActionEventBinding::GetUObject()` **5.6.0 才出现** |
| **2. 查值类别 / 重载 / 形参位置** | 同一签名在不同版本对**实参值类别**（左值/右值/`const`）与**重载选择**可能不同 ⇒ 不能只看"函数名存在" | 实例 A：`TUniqueFunction` 在 **5.0–5.4** 只有 `explicit TFunction_UniqueOwnedObject(T&&)`（仅右值），**5.5** 起重构后才接受 const 左值 ⇒ `FFunctionGraphTask::CreateAndDispatchWhenReady(Reload, …)` 在 5.0 报 C2665。实例 B：`TArray::RemoveAt(Index, false)` 在 5.0 把 `false` 当 **`Count`**（第 2 形参）⇒ `static_assert('unexpected bool passed as the Count argument')`，而 `FString` 的 bool 在**别的形参位置**，**先例不可照抄** |
| **3. 处置（优先级从高到低）** | 见下表 | —— |

**处置选项（按优先级）**

| 选项 | 何时用 | 要求 |
|---|---|---|
| **a. 两版通用写法**（首选，**零门控**） | 存在一种写法在最低版与最高版都合法 | 必须**同一份代码在两个版本都编过**。例：`[this]`（替 `[=, this]`）、`MoveTemp(Reload)`、把 range-for 的 init-statement 提为独立语句、`RemoveAt(Index, 1, false)` |
| **b. 门控宏** | 旧版**没有**等价 API/成员 | 宏**必须**落在 `Source/CrossVersion/Public/UEVersion.h`，**按版本升序分组**，命名 `UE_F_*`（特性）/`UE_E_*`（枚举值）/`UE_U_*`（UClass 成员）/`UE_T_*`（模板）；`#if` 两侧都必须是**可编译**的完整分支；🔴 **旧版分支的实参要逐形参核对**（见 7.1 步 2 实例 B） |
| **c. 回退（旧版无等价能力）** | 公开 API 也没有 | 用**公开 API 等效复刻**（例：5.0 的 `UpdateCanTick` 等效 = `GetDesiredTickFrequency() != Never` ⇒ `GetCachedWidget()->SetCanTick(true)`），或**显式退化**（例：5.0 的增强输入幂等守卫退化为 `(action, trigger)` 匹配）——**必须在报告里写清"退化点 + 精度损失 + 未验证部分"**，**禁止静默降级** |
| **d. 记为未取得项** | 上述都不可行 | 如实登记，**不得**用私有成员/`AccessPrivate` 等 hack 绕过（该路线已有裁定：不可替换、不新增） |

### 7.2 语言标准面（机械可查，必须 CLEAN）

```powershell
# 新增行专用（<rev> = 本轮基线提交）
& 'D:\file\UnrealProjects\UnrealCSharpTest\Saved\Retest\b29-xversion-lint.ps1' -Mode Syntax -DiffBase <rev>
# 全树体检
& '...\b29-xversion-lint.ps1' -Mode Syntax
```

- **UE 5.0–5.2 默认 C++17**（5.0.3 `VCToolChain.cs:765` 只加 `/std:c++17`，`/std:c++20` 被注释；5.3 起 `BuildSettingsVersion.V4` 起默认 C++20）⇒ **未门控代码与门控的旧版分支必须 C++17-clean**。
- 内置 C++20-only 清单：`[=, this]` / `[&, this]`、**带初始化语句的 range-for**、`consteval`/`constinit`、协程（`co_await`/`co_return`）、`std::span`/`std::bit_cast`、`<=>`、`[[likely]]`/`[[unlikely]]`、`char8_t`。
- **`#if STD_CPP_20` 分支内的 C++20 语法是允许的**（脚本会自动跳过并给出 `note: N hit(s) skipped`）；这正是 `FBindingClassRegister.cpp:116-120` 的既有形态。

### 7.3 版本查证装置（本机已具备）

```powershell
$lint = 'D:\file\UnrealProjects\UnrealCSharpTest\Saved\Retest\b29-xversion-lint.ps1'
# 支持起点：扫全部 32 个 release tag（5.0.0 … 5.8.3），给出首次出现的 tag
& $lint -Mode Symbol -Path <引擎仓内相对路径> -Symbol '<正则>'
# 可见性：给出该符号在各 tag 的 public/protected/private 与"可见性变化发生在哪个 tag"
& $lint -Mode Access -Path <引擎仓内相对路径> -Symbol '<正则>'
```

- 引擎源码仓：`D:\file\UnrealEngine\Release`（带 **5.0.0–5.8.3 全 32 个 `*-release` tag**）；等价的裸命令是 `git show <tag>:<path>`。
- **必须把"首个 tag + 可见性"抄进报告**，不能只写"较新版本"。

### 7.4 强制验证：**最低支持版本上必须真编一次**

- 报告里原有的"交付前格式与规范核查"一节**新增子项**：**版本兼容性核查表**

  | 接口/写法 | 最低支持版本 | 依据（tag + 路径:行） | 处置（通用/门控/回退） | 5.0 实编读数 |
  |---|---|---|---|---|

- **装置（本机已具备）**：最小 5.0 工程 `D:\file\UnrealProjects\UE50Check`
  - `.uplugin` = 本仓副本**去掉 `SourceCodeGenerator` 模块条目**（该模块是 **5.5+ 的 UBT 插件** `*.ubtplugin.csproj`，5.0 的 UBT 报 `Could not find definition for module 'SourceCodeGenerator'`）；
  - `Plugins/UnrealCSharp/Source`、`/Script` 用**目录 junction** 指回本仓 ⇒ 编的是同一份源码，**产物写在检查工程内**，不动本仓 5.6 的 `Binaries/Intermediate`；
  - 命令：`D:\file\UnrealEngine\5.0\Engine\Build\BatchFiles\Build.bat UE50CheckEditor Win64 Development -Project="D:\file\UnrealProjects\UE50Check\UE50Check.uproject" -WaitMutex`（首次会重建 5.0 引擎模块 ≈ 2950 actions；之后增量约 1–2 分钟）。
- 5.6 侧仍按常规判据：Windows UBT **Succeeded** ＋ 套件与基线**同 CSV SHA** ＋ 产物 manifest **0 差异**。

### 7.5 适用范围与例外

- **适用**：`Source/**` 全部 7 个模块 ＋ `Script/**`（C# 侧若引用有版本要求的 .NET/引擎 API，同样登记）。
- **不适用**：`Source/ThirdParty/**`（子模块，不分析）；插件**自定义**的类/宏/接口（只需保证仓内一致）。
- **裁定权**：退化或例外（选项 c）需在报告里单列，必要时走待决策页（参照 [`08-…/66`](08-专项审计/66-待决策页-三问（Q1四路线·Q2口径例外·Q3失败判据）（2026-10-15）.md) 的"口径例外"流程）。
- **首次案例**：轮 B-29 —— 8 条 5.0 编译错误 → 6 类根因 → 3 个新宏 → 5.0 实编（还当场抓出 1 处自伤）。

---

## 8. 【硬规范】每轮改完代码后：交付前复核「项目规范 / 代码风格 / 要不要格式化」（2026-10-16 维护者要求）

> **一句话**：**只要本轮动过插件源码（`Source/**`、`Script/**`），在收口报告成稿前必须做一次交付前复核**，覆盖下面三个轴，并**把读数写进报告**（沿用既有子节标题「交付前格式与规范核查」，口径源自 B-21 §21.3 → B-22 §3.4 → B-23 §3.5）。🔴 **不满足的需要满足**：任一 FAIL 必须**先修到 PASS 再收口**；确实无法满足的（例如引擎既有写法与项目风格冲突）必须在报告里**单列 + 说明原因**，不能沉默放过。

### 8.1 三个轴（缺一不可）

| 轴 | 查什么 | 判定 |
|---|---|---|
| **A. 项目规范（口径）** | ① G7：**不新增**注释 / 日志 / `throw` / `try-catch` / `ensure` / `check` / 公开 API；② Q2：失败必须**收敛到既有值域**（不得静默降级）；③ **代码形态**：新加逻辑走 `if(条件)` ＋ **单一出口**（B-17 规范）、局部变量**直接声明**（B-10 规范）；④ **新增接口的版本兼容**（§7 三步 ＋ 门控宏命名/分组）；⑤ 命名与既有 API 用语一致 | 逐条对照；新增行数、注释数、日志数等**机械计数**要贴出来 |
| **B. 代码风格** | ① **缩进**：C++（`.cpp/.h/.inl`）**TAB-first**；C# **4 空格**（TAB 在 `.cs` 里即 FAIL）；`TAB + 空格` 的**对齐续行**是合法形（先例：`TDefaultArgument.inl:38-44`）；② **行尾空白 = 0**；③ **行宽**：新增行 **>110 列 = 0 处**，且文件**最大行宽不得变差**；④ **BOM / 行尾符**与该文件基线一致（插件 C++ 交付态为 **CRLF、无 BOM**）；⑤ `git diff --check` **exit 0**；⑥ **不做与改动无关的重排**（diff 只含语义改动） | 逐文件读数表（下节装置直接产出） |
| **C. 要不要代码格式化** | **先判定仓内有无格式化配置**：`.clang-format` / `.clang-tidy` / `.editorconfig` / `.gitattributes` / C# 的 `dotnet format` 配置。**2026-10-16 全树复核结论：一份都没有** ⇒ **不跑格式化器、不做全树重排**，只做 B 的对照式核查，并在报告里写明这一判定。**一旦将来加入任一配置，本轴即改为"必须按配置运行后再交付"**（配置变更本身也要在此节登记） | 给出"有无配置"的判定 ＋ 复查命令 |

### 8.2 机械装置（每个改动文件一行读数）

```powershell
$style = 'D:\file\UnrealProjects\UnrealCSharpTest\Saved\Retest\b30-style-lint.ps1'
& $style -Base <本轮基线 rev>                    # 工作树 vs 基线
& $style -Base <基线 rev> -Target <交付 rev>      # 提交对提交（收口时用）
```

逐文件输出：`Added / Indent / Trailing / SpaceFail / SpaceAlgn / TabInCs / AlignTS / OverWidth / AddedMax / BaseMax / EOL / BOM / Comments / Verdict`，并汇总 `git diff --check`、新增注释行清单、以及"是否需要格式化"的结论；**`result: PASS` 才算过**。

**装置自检（2026-10-16，双向）**

| 自检 | 输入 | 读数 |
|---|---|---|
| 正向 | `-Base df487a6d -Target a9e10378`（交付的 12 个文件） | **PASS**：Trailing 0 / SpaceFail 0 / OverWidth 0 / AlignTS 4（`#if` 分支内的合法对齐）/ EOL **CRLF** / BOM **no** / Comments **0** / `diff --check` exit 0 |
| 负向 | 向 `FSetHelper.cpp` 注入 3 类违规（4 空格缩进行、行尾空格、135 列超宽行） | **FAIL**：Trailing **1** / SpaceFail **1** / OverWidth **1** / AddedMax 135 ＋ `diff --check` 报 `:231: trailing whitespace` ⇒ 修回后文件 blob 与基线**逐一相同** |

### 8.3 报告必录

1. **三轴读数表**（A 用计数 ＋ 逐条对照；B 直接贴装置输出；C 给出"有无格式化配置"的判定）。
2. **装置的完整输出**（或落盘日志路径，如 `Saved/Retest/out/style-<tag>.log`）。
3. **判定句**：是否需要格式化、是否有 FAIL、FAIL 如何修到 PASS（或为何无法满足并单列）。

### 8.4 适用范围

- **适用**：任何改动 `Source/**` 或 `Script/**` 的轮次（含只改 1 行的轮次）。
- **可叠加**：与 §7 的版本兼容核查**并行执行**（`b29-xversion-lint.ps1 -Mode Syntax -DiffBase <rev>` 覆盖语言标准面，本节的 B 轴覆盖排版面；两者都要求 `CLEAN` / `PASS`）。
- **纯文档轮次**（只改审计库 `.md`）：不触发本节，但仍须保证文档自身 **CRLF / 无 BOM / 无乱码**（既有惯例）。

---

## 9. 版本记录

| 日期 | 变更 |
|---|---|
| 2026-10-16 | 新增 **§7【硬规范】新调用的接口先做版本兼容性检测**（轮 B-29 确立）；配套装置 `Saved/Retest/b29-xversion-lint.ps1`（`Syntax`/`Symbol`/`Access` 三模式）与最小 5.0 检查工程 `D:\file\UnrealProjects\UE50Check` |
| 2026-10-16 | 规范首个落地案例（轮 B-29，8 文件 `+43/−9`）**已提交 `a9e10378636504f55e9bee4f376d944c918d16e0`**（父 `df487a6d`；blob 与已验证工作树逐一相同、零调整；提交级补丁 `Saved/Retest/fix-a9e10378.patch` `4C58F788…`）⇒ 本规范的"支持起点 + 门控宏 + 最低版本实编"三件套已完成一次端到端闭环 |
| 2026-10-16 | 新增 **§8【硬规范】每轮改完代码后：交付前复核「项目规范 / 代码风格 / 要不要格式化」**（维护者要求；三轴 ＋ 不满足必须修到 PASS）；配套装置 `Saved/Retest/b30-style-lint.ps1`（逐文件风格读数 ＋ `git diff --check` ＋ 新增注释清单 ＋ 格式化判定；正/负双向自检通过）；原 §8 版本记录顺延为 **§9** |
