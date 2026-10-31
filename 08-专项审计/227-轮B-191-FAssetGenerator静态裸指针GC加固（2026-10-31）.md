# 轮 B-191 FAssetGenerator 静态裸 UObject 数组 GC 加固（2026-10-31）

> 分析范围：`F-GEN3-009`（P2）
> 覆盖文件：插件 2 文件（`FAssetGenerator.h`／`FAssetGenerator.cpp`）
> 状态：已完成 —— 插件提交 **`5593df88`**（本轮 2 文件 `+11/−5`）

---

## 0. 覆盖范围与阅读清单

| 文件 | 本轮动作 | 依据 |
|---|---|---|
| `Source/ScriptCodeGenerator/Public/FAssetGenerator.h` | 容器类型改 `TArray<TStrongObjectPtr<UUserDefinedEnum>>`（＋include／前置声明） | `F-GEN3-009` |
| `Source/ScriptCodeGenerator/Private/FAssetGenerator.cpp` | 入口 `Reset()`、消费 `.Get()`、末尾 `Reset()` | 同上 |

---

## 1. 本轮结论（摘要）

1. **`F-GEN3-009` 已修**：静态容器由**裸 `UUserDefinedEnum*` 数组**改为 **`TArray<TStrongObjectPtr<UUserDefinedEnum>>`**；`Generator()` 入口先 `Reset()`（异常/提前返回也不会跨调用残留），消费处 `.Get()`，末尾 `Reset()`。
2. **判据（同一装置 A/B）**：临时控制台命令把对象放进**真实容器** → `TWeakObjectPtr` 观察 → `CollectGarbage(RF_NoFlags, true)` 全量 GC：
   - 改前：`B191PROBE num=1 weak=0`（静态裸指针**被 GC 回收**）；
   - 改后：`B191PROBE num=1 weak=1`（`TStrongObjectPtr` 钉住）。
3. **产物零差异**：`g8-manifest.ps1` 对 `Script/UE/Proxy` ＋ `Script/Game/Proxy` 共 **16 231** 个 `.cs` 做 `path|length|SHA256`：
   - **HEAD（裸指针）版 vs 修复版：0 行差异**；
   - 修复版连续两次生成：**0 行差异**；两次生成计数均 **16 155 / 76**（delta 0）。
4. **回归**：UBT **5.6 Succeeded**（22.13 s）／**5.0 0 error**（32.17 s）＋ `Target is up to date`（6.89 s）；套件 **1 992 行 / 0 fail / `085D3178…`**（首跑命中已知 `coreclr.dll@0x9603a` 环境崩溃，重跑即过）。

---

## 2. 缺陷与改法

### 2.1 改动前（基线 `5593df88`）

```cpp
// Public/FAssetGenerator.h:16
static TArray<class UUserDefinedEnum*> UserDefinedEnums;

// Private/FAssetGenerator.cpp:19
TArray<UUserDefinedEnum*> FAssetGenerator::UserDefinedEnums;

// :42-52 —— 两个分离循环，中间隔着一整轮类生成
for (const auto& AssetData : OutAssetData)
{
    Generator(AssetData, true);                         // 内部 :108 Emplace 裸指针
}

for (const auto& UserDefinedEnum : UserDefinedEnums)
{
    FEnumGenerator::Generator(UserDefinedEnum);         // 裸指针解引用
}

UserDefinedEnums.Empty();
```

`TStrongObjectPtr` 之外没有任何 GC 保护：容器是**静态**、元素是**裸指针**、无 `UPROPERTY`、无 `AddReferencedObjects`。复核结论已证伪"生成调用树内会自然触发 GC"的原始机制，但**"静态裸指针 ＋ 静态可变状态"**本身仍是隐患（`F-GEN3-009` 维持 P2 的依据）。

### 2.2 改动后（`5593df88`）

```cpp
// Public/FAssetGenerator.h:4/6/20
#include "UObject/StrongObjectPtr.h"

class UUserDefinedEnum;
...
static TArray<TStrongObjectPtr<UUserDefinedEnum>> UserDefinedEnums;

// Private/FAssetGenerator.cpp:19/28/49-54
TArray<TStrongObjectPtr<UUserDefinedEnum>> FAssetGenerator::UserDefinedEnums;

// Generator() 入口（:28）—— 任何异常/提前返回都不会把条目留到下一次调用
UserDefinedEnums.Reset();

for (const auto& UserDefinedEnum : UserDefinedEnums)
{
    FEnumGenerator::Generator(UserDefinedEnum.Get());
}

UserDefinedEnums.Reset();
```

- **零行为变化**：迭代顺序、元素集合、生成调用参数全部不变；仅把"持有关系"从裸指针换成强对象指针。
- **G7**：无新增注释/日志/`throw`/`try-catch`/`ensure`/公开 API；`Generator(...)` 签名未动。

---

## 3. 判据

### 3.1 装置（临时探针，已移除）

在 `FAssetGenerator.cpp` 内临时加入（**不入库**）：

```cpp
void FAssetGenerator::B191Probe()
{
    auto* Probe = NewObject<UUserDefinedEnum>(GetTransientPackage());

    UserDefinedEnums.Emplace(Probe);                 // 放进真实静态容器

    TWeakObjectPtr<UUserDefinedEnum> ProbeWeak(Probe);

    CollectGarbage(RF_NoFlags, true);

    UE_LOG(LogTemp, Display, TEXT("B191PROBE num=%d weak=%d"),
        UserDefinedEnums.Num(), ProbeWeak.IsValid() ? 1 : 0);

    UserDefinedEnums.Pop();                          // 移除探针条目，不污染真实生成
}
```

＋临时 `FAutoConsoleCommand`（`UnrealCSharp.Probe.B191`）与驱动脚本 `Saved/Retest/b191-driver.py`（slate tick 15 s 时发探针、22 s 时触发完整生成、150 s 后退出）。

**为什么用真实容器**：探针不另建同类静态变量，直接 `Emplace` 到 `FAssetGenerator::UserDefinedEnums`，因此读数就是"这个容器在 GC 面前是否保得住对象"；`Pop()` 在观察后移除条目，真实生成仍从空容器开始。

### 3.2 A/B 读数（同一驱动、同一命令，日志原文）

| 版本 | 日志 | 读法 |
|---|---|---|
| 改前（`5593df88` ＋ 探针） | `LogTemp: Display: B191PROBE num=1 weak=0` | GC 后弱指针失效 ⇒ 对象**已被回收**，静态数组里留下悬垂指针 |
| 改后（`5593df88` ＋ 探针） | `LogTemp: Display: B191PROBE num=1 weak=1` | GC 后弱指针仍有效 ⇒ `TStrongObjectPtr` **钉住**了对象 |

两轮均：`num=1`（容器里确实有 1 个元素）、无 `ensure/assert`、无 `LogBlueprint: Error`。

### 3.3 产物（逐字节）

| 对照 | 读数 |
|---|---|
| HEAD 裸指针版 vs 修复版（两次完整生成后） | `g8-manifest` **16 231 文件、0 行差异** |
| 修复版连续两次生成 | **0 行差异**（确定性） |
| 生成计数（改前/改后各一次） | **16 155 / 76**，`delta UE=0 Game=0` |

> 判定口径：manifest 为 `相对路径|字节数|SHA256`；0 行差异 = 文件集合与每个文件的字节全部相同。

### 3.4 回归

| 项 | 读数 |
|---|---|
| 套件 | **1 992 行 / 0 fail / `085D3178…`**（与 B-190 交付态相同） |
| 套件异常一次 | `B191-final2` 首跑 `exit -1073741819`、无 CSV；事件日志 **`coreclr.dll` `0xc0000005` / `0x9603a`** = 库内既有环境事实（`08-…/12`）⇒ **重跑 `B191-final3` 即过**（同 1 992/0 同 SHA） |
| UBT 5.6 | **Succeeded**（22.13 s） |
| UBT 5.0（`UE50Check`） | **0 error**（32.17 s，`[7/7] WriteMetadata`）＋ 复跑 **`Target is up to date`**（6.89 s） |

---

## 4. 版本兼容（`_CONVENTIONS.md` §7）

| 接口/写法 | 最低支持版本 | 依据 | 处置 |
|---|---|---|---|
| `TStrongObjectPtr`（`UObject/StrongObjectPtr.h`） | **5.0.0** | `b29-xversion-lint.ps1 -Mode Symbol`：33 个 tag 中 **FIRST FOUND in 5.0.0-release**（`class TStrongObjectPtr`，`:72`）；本机 5.0 实编头同路径存在 | 两版通用，**零门控** |
| `TStrongObjectPtr::Reset(T*)` / `Get()` | 5.0.0 | 5.0 头 `StrongObjectPtr.h:84-93`（ctor 调 `Reset`）、`:128-137`（`Get`） | 同上 |
| `TArray<TStrongObjectPtr<T>>` | 5.0.0 | 仓内先例 `UnrealCSharpEditor.h:65`（`TStrongObjectPtr<UDynamicDataSource>`） | 同上 |

`b29 -Mode Syntax` ⇒ **CLEAN**（无 C++20-only 构造）。

---

## 5. 交付前格式与规范核查（§8）

| 轴 | 读数 | 判定 |
|---|---|---|
| **A 项目规范** | 新增注释 **0**／日志 **0**／`throw`·`try-catch`·`ensure`·`check` **0**／公开 API **0**（成员类型变化，签名未变） | 通过 |
| **B 代码风格** | `b30-style-lint.ps1 -Base 5593df88` ⇒ **PASS**：2 文件、Trailing 0、SpaceFail 0、OverWidth **0**、AddedMax **77**（cpp）／**94**（h）、`diff --check` exit 0；EOL **CRLF**、BOM **no**；`formatter: NOT required` | PASS |
| **C 格式化** | 仓内无格式化配置 ⇒ 不跑格式化器 | 同前 |

机械计数：`git diff --numstat 5593df88` ⇒ `FAssetGenerator.cpp +5/−3`、`FAssetGenerator.h +6/−2`（合计 `+11/−5`）。

---

## 6. 未做 / 后续

1. **A 组剩 2 条**：`F-CS3-010`（已量化、只差"接 CI 或结案"的裁定）、`F-GEN3-008`（建议修法撞 G7，需裁定出口）—— 建议 B-192 用一轮 0 代码裁定清掉。
2. `06` B 组 7 条部分修残余 —— 建议 B-193 做一次残余面复核。
3. 未取得：真实工程资产上的**自然** GC 触发（复核结论本就认为生成调用树内无 GC 触发点；本轮用注入式全量 GC 构造场景，属"机制判据"而非"线上复现"）。

---

## 7. 复现命令

```powershell
# 探针 + 生成（改前/改后各一次）
& 'D:\file\UnrealProjects\UnrealCSharpTest\Saved\Retest\run-g13-generate.ps1' -Tag B191-before `
    -Driver 'D:\file\UnrealProjects\UnrealCSharpTest\Saved\Retest\b191-driver.py'
# 产物清单（逐字节）
& 'D:\file\UnrealProjects\UnrealCSharpTest\Saved\Retest\g8-manifest.ps1' -Tag b191-post1
# 构建与套件
dotnet build 'D:\file\UnrealProjects\UnrealCSharpTest\Script\Script.sln' -c Release   # 本轮 C# 未改
& 'D:\file\UnrealEngine\5.6\Engine\Build\BatchFiles\Build.bat' UnrealCSharpTestEditor Win64 Development `
    -Project='D:\file\UnrealProjects\UnrealCSharpTest\UnrealCSharpTest.uproject' -WaitMutex
& 'D:\file\UnrealProjects\UnrealCSharpTest\Saved\Retest\run-batch.ps1' -Tag B191-final3 -Runs 1
```

留档：`Saved/Retest/out/{B191-before,B191-after,B191-gen2,B191-rawgen,B191-final3}`、`Saved/Retest/out/g8-manifest/products-b191-*.txt`、`Saved/Retest/b191/{ubt,ubt-final,ubt-final2,ue50,lint-syntax,lint-style}.log`。
