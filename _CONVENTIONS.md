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

### 编号唯一性（2026-10-17 追加，来源 = 两处真实撞号）
新增任何 `F-<模块前缀>-<序号>` 之前，**必须先做全库机械检查**：① `grep` 全树（插件 `Source/**`、`Script/**`、`08-专项审计/**`、`09-问题清单/**`）确认该编号**尚未存在**；② 同族编号一律取 **"当前最大号 ＋ 1"**（**不复用空号** —— 空号可能对应过已删条目或仍被引用）。

⚠️ 同一编号来自**不同报告**时必须带**报告号后缀**（本仓既有 7 组，如 `F-PROP2-006 (04b)` / `F-PROP2-006 (05)`、`F-INT2-004 (10b)` / `F-INT2-004 (11)`），**不改写既有编号**。

若事后发现真撞号：**只给"后登记者"改号**（取同族最大号 ＋ 1），并在 ① 台账行、② 登记报告对应小节、③ 全部引用处写明"**编号更正：原登记 X ⇒ 改为 Y（撞的是哪一条）**"，报告正文的小节标题同步改号（案例见 2026-10-17：`F-GEN-017` → `F-GEN-030`、`F-ABI-036` → `F-ABI-041`）。

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

- 🆕 **跨版本实编点清单（2026-10-16 轮 B-31 更新为四点）**：`5.0.3`（`D:\file\UnrealEngine\5.0` ＋ `UE50Check`）、`5.4.4`（`E:\UnrealEngine\5.4` ＋ `UE54Check`）、`5.6.0`（`D:\file\UnrealEngine\5.6` ＋ 本仓工程）、**`5.8.2`（`D:\file\UnrealEngine\Release` ＋ `E:\UnrealEngine\UE58Check`）**。检查工程一律 = "4–5 文件的最小工程 ＋ 插件 `Source`/`Script` **目录 junction** 回本仓 ＋ `.uplugin` **逐字副本**"（产物写在检查工程内、不动本仓）；子模块 `Source/SourceCodeGenerator` 必须切到与引擎匹配的分支（`5.0`/`5.1`/`5.3`/`5.5`/`5.8`，`TargetFramework` = net6.0/net6.0/net8.0/net10.0），**收口后切回工作副本原分支**。
- 🆕 **已构建引擎的"离线上手三步"（2026-10-16 轮 B-31 实测定型；NuGet/GitHub 可用时不需要）**：若 `Build.bat` 卡在 `Building UnrealBuildTool...`（`NU1301`；本机 TLS 吊销检查不可用 ⇒ 无法 restore），而目标引擎其实**已构建**：① 在 `Engine\Binaries\DotNET\UnrealBuildTool\` 写入 `UnrealBuildTool.runtimeconfig.json`（`net10.0` / `Microsoft.NETCore.App 10.0.0` / `rollForward: LatestMinor`）；② 把 `runtimes\win-x64\native\*`（`blake3_dotnet`/`e_sqlite3`/`oo2core`/`UbaDetours`/`UbaHost`）复制到该目录根（**无 `deps.json` 时宿主不探 `runtimes/<rid>/native`**）；③ 以 `dotnet <该目录>\UnrealBuildTool.dll <Target> <Platform> <Config> -Project=<检查工程>.uproject -WaitMutex` **直跑**（**CWD 必须是 `Engine\Source`** —— rsp 里的 `/I` 是相对它的）。⚠️ `dotnet publish --no-build --no-restore` **仍会触发 restore**，离线不可用。
- 🆕 **无实编点时的静态核查装置（2026-10-16 轮 B-32 定型）**：当某个门控档**没有实编点**（本机无该版本引擎 ＋ GitHub/NuGet 不可达 ⇒ 无法克隆）时，用 **32 个 release tag 的静态证据**替代一次真编 —— 对实编侧引用的**每一个引擎符号**取四要素：**首个含它的 tag** ＋ **可见性** ＋ **精确重载/签名** ＋ **是否 `UE_DEPRECATED`**（在窗口的**两端 tag** 上分别取）。装置：
  ```powershell
  $b32 = 'D:\file\UnrealProjects\UnrealCSharpTest\Saved\Retest\b32'
  # ① 抽取"UE X.Y 真正编译的那一支"（正极性门控取 #if、负极性取 #else）
  & "$b32\b32-gate-extract.ps1"                      # 46 宏 / 70 点 ⇒ b32-gates-5.2-usage.md
  # ② 批量符号/可见性查询（逐查询扫全部 32 tag；query 行格式 = <引擎相对路径>|<正则>|<备注>）
  & "$b32\b32-symcheck.ps1" -QueryFile "$b32\q.txt" -OutFile "$b32\out.csv"
  # ③ 定点读原文（tag + 行区间 / 正则 + 上下文）
  . "$b32\b32-probe.ps1"; P '5.2.1-release' 'Engine/Source/.../Foo.h' 'operator\+'; EX '5.1.0-release' '...' 128 140
  ```
  - ⚠️ **装置必须先用"已知答案"样本自检**（轮 B-32 抓到两处：tag 名要按 `^5\.2\.1` 匹配 `5.2.1-release`，否则"是否存在于 5.2.1"一列**恒为假阴性**；可见性判定要**跟踪花括号深度**，否则**命名空间作用域**符号会报出假阳性）。
  - ⚠️ **`UE_DEPRECATED` 不算失败**：本插件无 warnings-as-errors（响应文件实测 `/W4` ＋ 少量 `/we44xx`、**无 `/WX`**），判定只看"存在性 ＋ 可见性 ＋ 签名"。
  - **覆盖边界（必须写进报告）**：静态核查**能**判定 C2039/C2661/C2664/C2248/C3668 这类编译级失败，**不能**替代真编（编译器实际诊断、链接期、运行期）⇒ 缺口只收窄到"只剩真编"这一格，并在报告 §7 写清**真编触发条件**。
- 🔴 **探针纪律：「表达式级」实测 ≠ 「TU 级」实测（2026-10-16 轮 B-32 确立，含一次自伤案例）**：跨版本核查若用最小 TU 复现某段代码，**必须先证明它在真实 TU 里真的被实例化/odr-use**。反例：`FRegisterDateTime.cpp:56` 的 `const FDateTime + const FDateTime` 在 5.1–5.6 **无可行重载** —— **把该表达式单独提取成被调用的函数 ⇒ 实测 `error C2678`**；但**逐字复刻真实声明/定义形状（含门控、含"都不被调用"）⇒ C2678 命中 0**（MSVC 不对未被 odr-use 的类内静态成员函数体做完整语义检查）。**正确手法**：把待验证的**声明/定义形状逐字复刻**进最小 TU（namespace、门控、是否被调用都要一致），再编译。
- 🆕 **覆盖缺口要"按门控档位清点"，不是按版本号清点**（2026-10-16 轮 B-31 确立）：`UEVersion.h` 的**版本门控**按"支持起点"分档（当前 104 个：5.1 **44** / 5.2 2 / 5.3 3 / 5.4 6 / 5.5 11 / 5.6 19 / 5.7 **11** / 5.8 **8**）；**只要"最高实编点 < 最高门控档"，起点高于它的那批门控的"新版侧"就从未参与过编译**（轮 B-31 前 = **19 宏 / 87 处 `#if`**）。⇒ **选新实编点的依据 = 未被编过的门控面**，且**最低版本与最高版本都要编**（B-29 的错误只在 5.0 暴露；B-31 的 19 个宏新版侧只在 5.7+ 参与编译）。

### 7.4 强制验证：**最低支持版本上必须真编一次**

- 报告里原有的"交付前格式与规范核查"一节**新增子项**：**版本兼容性核查表**

  | 接口/写法 | 最低支持版本 | 依据（tag + 路径:行） | 处置（通用/门控/回退） | 5.0 实编读数 |
  |---|---|---|---|---|

- **装置（本机已具备）**：最小 5.0 工程 `D:\file\UnrealProjects\UE50Check`
  - `.uplugin` = 本仓副本**去掉 `SourceCodeGenerator` 模块条目**（该模块是 **5.5+ 的 UBT 插件** `*.ubtplugin.csproj`，5.0 的 UBT 报 `Could not find definition for module 'SourceCodeGenerator'`）；
  - `Plugins/UnrealCSharp/Source`、`/Script` 用**目录 junction** 指回本仓 ⇒ 编的是同一份源码，**产物写在检查工程内**，不动本仓 5.6 的 `Binaries/Intermediate`；
  - 命令：`D:\file\UnrealEngine\5.0\Engine\Build\BatchFiles\Build.bat UE50CheckEditor Win64 Development -Project="D:\file\UnrealProjects\UE50Check\UE50Check.uproject" -WaitMutex`（首次会重建 5.0 引擎模块 ≈ 2950 actions；之后增量约 1–2 分钟）。
  - 🆕 **`.uplugin` 的"回灌"陷阱（2026-10-23 轮 B-68/B-69 连踩两次）**：该副本会因"重新复制插件"而**回到全量版本**（含 `SourceCodeGenerator`），于是 5.0 的 UBT 在**解析阶段**就 `ERROR: Could not find definition for module 'SourceCodeGenerator'` —— **这不是插件缺陷，是装置回灌**。⇒ 每次使用 5.0 实编点前先确认该副本**已去掉该模块条目**；处置惯例 = 原件留档 `UnrealCSharp.uplugin.full.bak`、当场去掉条目再编，并把"装置被回灌"记进报告 §7.4（不要记成 finding）。
  - 🆕 **"改前 FAILED / 改后 Succeeded"必须同装置对照（2026-10-23 轮 B-69 确立）**：5.0 侧的最小判据 = **同一 `UE50Check` ＋ 同一命令**下 `error C????` 条数由 >0 → **0**，并**再跑一次拿到 `Target is up to date`**（排除"恰好没重编"的假绿）。只跑一次 Succeeded 不足以证明门禁恢复。
- 5.6 侧仍按常规判据：Windows UBT **Succeeded** ＋ 套件与基线**同 CSV SHA** ＋ 产物 manifest **0 差异**。

### 7.5 适用范围与例外

- **适用**：`Source/**` 全部 7 个模块 ＋ `Script/**`（C# 侧若引用有版本要求的 .NET/引擎 API，同样登记）。
- **不适用**：`Source/ThirdParty/**`（子模块，不分析）；插件**自定义**的类/宏/接口（只需保证仓内一致）。
- **裁定权**：退化或例外（选项 c）需在报告里单列，必要时走待决策页（参照 [`08-…/66`](08-专项审计/66-待决策页-三问（Q1四路线·Q2口径例外·Q3失败判据）（2026-10-15）.md) 的"口径例外"流程）。
- **首次案例**：轮 B-29 —— 8 条 5.0 编译错误 → 6 类根因 → 3 个新宏 → 5.0 实编（还当场抓出 1 处自伤）。

---

### 7.6 测试装置纪律：判据型探针（2026-10-17 轮 B-34 确立）

| 纪律 | 内容 | 依据 |
|---|---|---|
| **① 日志型判据的读法** | UE 的当次日志（`Saved/Logs/<Project>.log`）被引擎**以写方式持有** ⇒ `FFileHelper::LoadFileToStringArray` 会**失败**（本轮返 `-1`；`GetLogProbeInfo` 取证 `exists=1 size=95652 loaded=0`）。必须走 `FFileHelper::LoadFileToString(Content, Path, FFileHelper::EHashOptions::None, FILEREAD_AllowWrite)` 再 `ParseIntoArrayLines` | [`08-…/75`](08-专项审计/75-轮B-34收口-批次C三条判据补用例（2026-10-17）.md) §5.2 |
| **② `-1` 哨兵 ≠ `0`** | 动态类的 plain `[UFunction]` 的 `FunctionFlags` **实测可为 0** ⇒ "用 0 表示不存在"必然假红（本轮首版 `BatchCSealedEventChildHasOwnOpenFunctionFlags` 即此坑）。存在性只能用 **`-1` 哨兵**或"函数声明类（`Function->GetOuter()`）"判断 | 同上 §4.2 / §5.2 |
| **③ 判据前先问"分支进得去吗"** | 依赖图里**对象引用恒为软依赖**（`FDynamicGeneratorCore::GeneratorField():108-118`；图遍历 `FDynamicDependencyGraph.cpp:119-124` 跳过软依赖），硬边只到 **enum / struct / 父类** ⇒ **硬环不可构造**（struct 互含被 C# 禁止 `CS0523`）⇒ "看起来是环"的用例**命中不了**空 `else`。任何"新语料命中"型判据都要先做这一步可达性证明 | 同上 §3.3 |

- 🪤 **动态类命名硬规则（源生成器，`Script/SourceGenerator/UnrealTypeSourceGenerator.cs`）**：`UClass` 名必须 **`U`/`A` 开头或 `_C` 结尾**（由基类首字母决定，`:481-521`）；**文件名** = `_C` 结尾时同名、否则**去首字母**（`:713-725`）；违者报 `UC_ERROR_02/03` 并**让整个 C# 编译失败**（不是警告）—— 直接决定"新增动态测试类"能不能落地。
- 🆕 **诊断型判据的取证方式（2026-10-17 轮 B-36 确立）**：当"修法"的落点是 **Roslyn 诊断**（`UC_ERROR_0N`）时，判据必须给**两组**：**正向**（非法形态在 ⇒ 构建 **FAILED** 且错误码/消息/位置逐字可引）＋ **控制**（合法形态不在报告里：① 结构相似但缺少触发条件的形态；② 删除非法形态后整树 **0 Warning / 0 Error**）。⚠️ **修好后要删掉"记录缺陷本身"的判据** —— 例如"强制覆写会让子类拥有自有 UFunction"这类判据在诊断落地后**无法构造**（形态编译不过）⇒ 必须删除并换成诊断判据，台账写"因修法而作废"而不是"丢了判据"。
- 🆕 **模板 ≡ 部署副本必须做成一次可复核的判据**：插件 `Script/**` 是模板（含 `GameNamePlaceholder` 等占位符），工程里的 `Script/**` 是实例化副本；改模板后要**把占位符替换掉再逐行对比**（允许的差异只有实例化头/占位符），否则"改的是模板、验的是副本"会成为隐患。
- 🪤 **`IsScriptPublishOutdated()` 只看 `Script/Shared.props` 与两个发布程序集的时间戳，不看 `*.cs`**（`FUnrealCSharpFunctionLibrary.cpp:1301-1329`）⇒ "只改 C# 源码"时启动期会自动跳过编译；命令行走 `-run=GeneratorScriptCode` 不受此限。**判据**：`Content/Script/Game.dll` 时间戳必须晚于最后一次 `.cs` 修改。

## 8. 【硬规范】每轮改完代码后：交付前复核「项目规范 / 代码风格 / 要不要格式化」（2026-10-16 维护者要求）

> **一句话**：**只要本轮动过插件源码（`Source/**`、`Script/**`），在收口报告成稿前必须做一次交付前复核**，覆盖下面三个轴，并**把读数写进报告**（沿用既有子节标题「交付前格式与规范核查」，口径源自 B-21 §21.3 → B-22 §3.4 → B-23 §3.5）。🔴 **不满足的需要满足**：任一 FAIL 必须**先修到 PASS 再收口**；确实无法满足的（例如引擎既有写法与项目风格冲突）必须在报告里**单列 + 说明原因**，不能沉默放过。

### 8.1 三个轴（缺一不可）

| 轴 | 查什么 | 判定 |
|---|---|---|
| **A. 项目规范（口径）** | ① G7：**不新增**注释 / 日志 / `throw` / `try-catch` / `ensure` / `check` / 公开 API；② Q2：失败必须**收敛到既有值域**（不得静默降级）；③ **代码形态**：新加逻辑走 `if(条件)` ＋ **单一出口**（B-17 规范）、局部变量**直接声明**（B-10 规范）；④ **新增接口的版本兼容**（§7 三步 ＋ 门控宏命名/分组）；⑤ 命名与既有 API 用语一致 | 逐条对照；新增行数、注释数、日志数等**机械计数**要贴出来 |
| **B. 代码风格** | ① **缩进**：C++（`.cpp/.h/.inl`）**TAB-first**；C# **4 空格**（TAB 在 `.cs` 里即 FAIL）；`TAB + 空格` 的**对齐续行**是合法形（先例：`TDefaultArgument.inl:38-44`）；② **行尾空白 = 0**；③ **行宽**：新增行 **>110 列 = 0 处**，且文件**最大行宽不得变差**；④ **BOM / 行尾符**：**BOM 一律不得存在（全仓无 BOM，2026-10-23 轮 B-70 收口）**，C++ 交付态 **CRLF**（C# 与该文件基线一致）；⑤ `git diff --check` **exit 0**；⑥ **不做与改动无关的重排**（diff 只含语义改动） | 逐文件读数表（下节装置直接产出） |
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
| 2026-10-16 | §7.3 装置清单追加三条（**跨版本实编点清单 = 5.0 / 5.4 / 5.6 / 5.8 四点** ＋ **已构建引擎的"离线上手三步"**（runtimeconfig ＋ 原生库进 app 目录 ＋ 直跑 UBT，CWD=Engine\Source） ＋ **覆盖缺口按门控档位清点**（最高实编点 < 最高门控档 ⇒ 一整批"从未编过的分支"））；来源 = 轮 B-31（新增 UE 5.8 实编点，19 宏 / 87 处 ≥5.7 门控面首次真编）；报告 [`08-…/71`](08-专项审计/71-跨版本实编点扩容-UE5.8与门控新版侧首编（2026-10-16）.md) |
| 2026-10-16 | §7.3 装置清单追加两条（**无实编点时的静态核查装置**：`b32-gate-extract.ps1` / `b32-symcheck.ps1` / `b32-probe.ps1` ＋ 用法 ＋ 两处装置自检教训 ＋ "`UE_DEPRECATED` 不算失败" ＋ 覆盖边界；**探针纪律：「表达式级」实测 ≠ 「TU 级」实测**，含 `FRegisterDateTime.cpp:56` 的一次自伤案例）；来源 = 轮 B-32（UE 5.2 静态核查：46 宏 / 70 点 / **70 通过 0 失败**；含一条"先报后撤"的自我更正）；报告 [`08-…/72`](08-专项审计/72-跨版本实编点-UE5.2静态核查（2026-10-16）.md) |
| 2026-10-16 | §7.3 追加一条**形态纪律**（来源 = 轮 B-33 的一次自伤 A/B）：**C# 参数默认值必须是编译期常量** ⇒ 本插件的包装类型（`FName`/`FString`/`FText`/`TScriptInterface<T>`/`TSubclassOf<T>` 等**引用类型**）**只能**用 `null` 作签名默认值，真实默认值必须落在**方法体**里（`X ??= new FName("…")`，见 `FClassGenerator::GeneratorFunctionDefaultParam`）。**因此 `FClassGenerator::GetBlueprintFunctionDefaultParam` / `GetCppFunctionDefaultParam` 里各类型分支的 `" = null"` 是承重写法、不是"未实现"**；把它改成 `= new FName("…")` 会让生成物直接 `error CS1736`（本轮实测 **46 个 UE 代理文件**编不过）。**推论（写入判据纪律）**：审计报告给出的"建议修法"与"建议判据"**同样必须先做正向 A/B 才有资格进施工**——静态论证会漏掉语言/框架级的硬约束 |
| 2026-10-16 | §9 追加两条**形态纪律**（来源 = 轮 B-33 的实测与维护者回核）：① **C# 参数默认值必须是编译期常量** ⇒ 包装类型（`FName`/`FString`/`FText`/`TScriptInterface<T>`/`TSubclassOf<T>`…）**只能**用 `null` 作签名默认值、真实默认值落**方法体**（`X ??= new …`）⇒ `FClassGenerator` 的 `" = null"` 是**承重写法**，改成 `= new FName("…")` 会让生成物直接 `error CS1736`（实测 **46 个 UE 代理文件**编不过）；② 🔴 **`StaticLink()` 之前的 `UClass` 属性链不可见** —— 动态类构造期间（`SetFlags`/`ProcessGenerator` 内）`UClass::FindPropertyByName` **恒 `nullptr`**（探针 `B33-PROBE … found=0`），**属性 metadata 的写入必须延后到 `EndGenerator()`（`Bind()` ＋ `StaticLink()`）之后**；推论：凡是"函数侧要给属性写 metadata"的功能（访问器关联、属性级校验）都必须是**两段式**（构造期记待办 → 链接后 flush）。⚠️ 同轮修掉 `b30-style-lint.ps1` 一处装置缺陷：`$spaceFail` 未排除 `.cs` ⇒ **C# 的 4 空格缩进被误判为 FAIL**（该装置此前只跑过 C++ 文件） |
| 2026-10-16 | §8 追加**两条方法学**（来源 = 轮 B-33 v4，维护者复核"是不是没跑规范"）：① 🔴 **"lint PASS" ≠ "§8 PASS"** —— `b30-style-lint.ps1` 只覆盖 **B 轴（排版）**，`b29-xversion-lint.ps1 -Mode Syntax` 只覆盖**语言标准面**；**A 轴的第 ③（`if(条件)` ＋ 单一出口，B-17）与第 ⑤（命名与既有 API 用语一致）没有任何自动装置**，**必须人工逐条对照**。② **A 轴 ⑤ 的取证方式＝"新增标识符清单 × 每个的仓内先例"**（模板见 [`08-…/74`](08-专项审计/74-轮B-33收口-死代码批次C补实现（2026-10-16）.md) §6.5）：逐个新增名贴出"返工前 → 返工后 ＋ 判定依据（同文件/同模块的既有 API）"；**其中"文件作用域 static 用裸 PascalCase（不加 `G` 前缀）"与"布尔局部量用 `auto bXxx =`（仓内 103 : 13）"两条已由本轮实测确立为可引用先例** |
| 2026-10-16 | §8 追加**第三条方法学**（来源 = 轮 B-33 v5）：🔴 **"改某个可疑点无效" ≠ "它不是原因"** —— 当**形态返工**（如 `auto&`→`auto`）与**语义改动**（如元数据键名）**同批落地**时，必须**先做二分回退**（形态回退一次、语义回退一次），**不得靠连续猜测试**。同时：**观测点要选最靠前的那个**（本例最有判别力的读数是 flush 的 `entries` 计数，而不是"最终键名长什么样"）—— 我在该轮连改 5 版键名全部无效，加上 `entries` 日志后一次定位。⚠️ 附带的形态纪律：**`TMap::FindOrAdd` 的返回值必须用引用（`auto&`）接**；用 `auto` 会退化成临时对象，对其 `Emplace` 的内容在表达式结束即丢弃（本插件内 `FClassGenerator` 等处既有写法均为 `auto&`） |
| 2026-10-16 | §8 追加**第四条方法学**（来源 = 轮 B-33 v6，维护者复核"`AccessorPending` 没看到有地方清空"）：🔴 **凡"两段式"中间态（静态容器／成员缓存），其"产生点"与"清空点"必须在代码里成对可查，并在报告里画出生命周期全图**（产生点／消费＋清空点／为什么必须两段／调用链是否闭合／残留风险）。**只写 `Remove` 不够** —— 还要做到：① `Remove` **提到 `if` 之外**（只要清理函数被调用就一定清，不依赖 `Find` 成功）；② **populate 先 `Reset()` 再填**（重复生成幂等；且若某路径漏了清理，下次填充**覆盖**而非混入旧数据 —— 旧数据里的裸指针在 domain reload 后是悬垂的）。③ 举证方式 = **临时探针实测"清理后计数归零"**（本例 11 个类全部 `num=0`），而不是只靠调用链推理 |
| 2026-10-16 | §8 追加**第五条方法学**（来源 = 轮 B-33 v7，维护者追问"`AccessorPending` 还有没有更好的命名"）：🔴 **命名必须同时过"仓内词序约定"与"行宽上限"两道闸**，且**同族标识符不得只差词序**。① **词序**：仓内 `Pending` 一律作**前缀**（`PendingBindClasses`/`PendingBindObjects`/`PendingCounts`/`PendingOutput`），`XxxPending` 是反例；② **行宽**：改名不得把声明顶过 **110 列** —— 更"精确"的长名（`PendingAccessorMetaData` → 113 列）会被 `b30-style-lint.ps1` 判 `OverWidth` FAIL，**可读性不能反过来破排版红线**；③ **反混淆**：容器与局部引用**不得只差词序**（`AccessorPending` vs `PendingAccessor` 是典型反例 —— 恰好"对的名字落在局部、错的名字落在容器"）；④ **后缀语义**：`Queue` = FIFO 消费、`Cache` = 可重复读取，**一次性缓冲不得用这两个后缀** |
| 2026-10-17 | 新增 **§7.6 测试装置纪律（判据型探针）** 三条 ＋ 两条装置坑（来源 = 轮 B-34：批次 C 三条"未取得判据"**全部取得**、`F-DEF-009` 判定"空 `else` **结构性不可达**"、新登记 **`F-DEF-010`**）：① **日志型判据必须用 `FILEREAD_AllowWrite` 读**（引擎持有写句柄，普通读返回失败）；② **`-1` 哨兵 ≠ `0`**（plain `[UFunction]` 的 flags 实测可为 0）；③ **判据前先证明"分支进得去"**（对象引用恒为软依赖 ⇒ 硬环不可构造）；④ 动态类命名/文件名硬规则（`UC_ERROR_02/03` 会让 C# 编译直接失败）；⑤ `IsScriptPublishOutdated()` 不看 `*.cs`。报告 [`08-…/75`](08-专项审计/75-轮B-34收口-批次C三条判据补用例（2026-10-17）.md) |
| 2026-10-17 | §7.6 追加**诊断型判据的取证方式**（正向 FAILED ＋ 两组控制；修好后删除"记录缺陷本身"的判据并记"因修法而作废"）＋ **模板 ≡ 部署副本**的等价性判据（替换占位符后逐行对比）；来源 = 轮 B-36（`F-DEF-010` 方案 C：新增 `UC_ERROR_08` 编译期诊断，插件 1 文件 ＋90 行；判据 = 改前 build succeeded → 改后 FAILED/1 错、控制 0/0、套件 1959/0）。报告 [`08-…/78`](08-专项审计/78-轮B-36收口-四问回执与F-DEF-010编译期诊断（2026-10-17）.md) |
| 2026-10-17 | 新增 **§2 编号唯一性规则**（来源 = 台账机械扫描：`01-全局问题总表.md` 的编号行里 **2 处真撞号** —— 轮 B-35 新登记 `F-GEN-017` 撞既有同名（P3·大对象按值拷贝）、轮 B-12 新登记 `F-ABI-036` 撞既有（P2·`.csproj`/`.sln` 写死 Windows 反斜杠））。处置：前者改 **`F-GEN-030`**、后者改 **`F-ABI-041`**（各取同族"最大号 + 1"：F-GEN 29 → 30、F-ABI 40 → 41），**全库 23 行引用**一次性改名，并在台账行 / 登记报告（`08-…/50`、`08-…/77` 及其下游 `64`、`66`）写"编号更正"。**规则 = 新增前全库 grep ＋ 同族取"最大号 + 1"＋ 不复用空号；跨报告同号必须带报告号后缀；事后撞号只改后登记者** |
| 2026-10-23 | **编码基线收口（轮 B-70）**：插件仓 **940 个文本文件全部为合法 UTF-8**（非 UTF-8 **0** / UTF-16·32 **0** / 孤立 CR **0**）；一次性去掉 **9 个"自 initial 提交（`c40fe771`）就带 BOM"的上游 C++ 文件**的 UTF-8 BOM（提交 `8c6d3222`，9 文件 `+9/−9`，**字节级每文件 −3**，`SHA256(new)==SHA256(old[3:])`）⇒ **tracked 源文件 BOM = 0**（剩余 8 个 BOM 全在 `obj/`，untracked、每次 restore 重生成）。**§8.1-B 的 BOM 口径随之改为"全仓无 BOM"**。安全性依据：UE **5.0.3** 的 `VCToolChain.cs:413-414` 给 MSVC `/source-charset:utf-8` ＋ `/execution-charset:utf-8`、`:374` `/wd4819`；**5.6+** 为 `/utf-8` ⇒ 源码 BOM 对 MSVC 无影响；实测 5.0 实编 **`C4819`/`C2001`/`C1057` = 0**（唯一带中文的文件只有 8 行 `//` 注释、无中文字符串字面量）。扫描器 `Saved/Retest/enc-scan.py`（排除 `Source/ThirdParty`，可换根复跑；产物 `Saved/Retest/out/encscan/`）。报告 [`08-…/108`](08-专项审计/108-轮B-70-源文件BOM清理（2026-10-23）.md)。**同日追加（维护者选 A）**：对**本审计库 156 个文件**复扫发现 **22 个 `.md` 仍带 BOM**（`00-总览/03`、`08-专项审计/42…62` 共 20 份、`09-…/02`；成因 = 10-10…10-15 那批用 PS 5.1 `Set-Content -Encoding utf8` 写盘），已按字节去首 3 字节（22/22 `SHA256(new)==SHA256(old[3:])`、EOL 零变化、`git diff --numstat` 全 `1 1`）⇒ **审计库 BOM = 0**。**未动**：工程侧的 3 个真源码 ＋ 3 个生成/配置 ＋ 8 个日志 ＋ 3 个 UTF-16LE 草稿（非 A 档，清单见 [`108`](08-专项审计/108-轮B-70-源文件BOM清理（2026-10-23）.md) §8） |
