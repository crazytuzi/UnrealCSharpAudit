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
