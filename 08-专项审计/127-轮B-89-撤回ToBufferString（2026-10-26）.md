# 轮 B-89 撤回 `FText.ToBufferString()`：无消费者，按 §8.5 E2 撤回（2026-10-26）

> **来源**：维护者复核 [`91fc122a`](121-轮B-83-新增公开面清掉最后两条G7（2026-10-25）.md) 时问「`ToBufferString` 是干嘛用的，感觉没啥用吧」⇒ 核实后确认**无仓内消费者**，决定 **删**。
> **口径**：§8.5 **E2**（新增只读 API 必须有真实调用方，禁止死 API）＋ G7 ＋ §8；**插件 3 文件 `+1/−40`**（另有工程侧测试 `−2` 条断言）。
> **基线**：插件 **`91fc122a`** ⇒ 提交 **`91fc122a`**；套件回到 **1 980 断言**。
> **一句话结论**：`ToBufferString()` 是 `FTextStringHelper::WriteToBuffer` 的包装（返回 `INVTEXT(...)` / `NSLOCTEXT(...)` / 裸串），**对普通文本与 `ToString()` 完全相同**；仓内 **0 业务调用者**、唯一调用者是本仓测试且只覆盖了**退化情形** ⇒ 按 E2 撤回，`F-PROP2-006 (04b)` 回到 **⚫ 决定不做**。套件 **1 980 / 0 ×2 同 SHA `7F0C2E4C…`**，与 B-81 基线**逐字节相同**。

---

## 1. 它到底做了什么（引擎读数）

`FTextStringHelper::WriteToBuffer(FString&, const FText&, bRequiresQuotes=false, bStripPackageNamespace=false)` —— `5.6.0 Text.cpp:1965-1992`：

| 分支 | 条件 | 输出 |
|---|---|---|
| ① | `Value.IsCultureInvariant()` | `INVTEXT("…")`（`:1972-1974`） |
| ② | `GetTextData().GetTextHistory().WriteToBuffer(...)` 成功（本地化/命名文本） | 该 history 的导出形式，如 `NSLOCTEXT("Pkg","Key","Text")`（`:1978`） |
| ③ | 其余 | **`Value.ToString()` 原样**（`:1990`） |

**关键事实**：普通文本（`new FText("12")`）走 ③ ⇒ **`ToBufferString()` 与 `ToString()` 返回值完全相同**；只有 ①② 两类文本才有区别。

它确实与 `new FText(buffer, ns, pkg)` 构成逆运算 —— 构造走 `FTextStringHelper::ReadFromBuffer`（`:1898`），先尝试按**复杂文本导出**解析（`:1914-1922`），因此能还原 `NSLOCTEXT(...)` 的命名空间/键。也就是说：**只对"本地化文本的持久化"这一场景有价值**。

## 2. 为什么撤回

| 判据 | 读数 |
|---|---|
| 业务调用点 | **0**（`Script/**`、`Source/**` 里只有定义链） |
| 唯一调用者 | 工程侧测试 `TestTextToBufferString`（2 条断言） |
| 该测试证明了什么 | 只测了 `new FText("12")` —— 恰好是 ③ 分支、`ToBufferString() == ToString()` 的**退化情形**；**真正有价值的 NSLOCTEXT 保真一半没测** |
| 项目规则 | §8.5 **E2**："新增只读 API **必须**有真实调用方（禁止新增死 API）" ⇒ 测试不是业务消费者 |
| 反方论据 | "构造函数存在但无生产者 = 单向门" —— 用户可以自己写 `NSLOCTEXT(...)` 字面量传入，不构成阻塞 |

⇒ 按本仓自定规则撤回；`F-PROP2-006 (04b)` 的"`ToString` 往返丢 key/namespace"**事实保留**，但**不再以新增 API 解决**。

## 3. 撤回内容

| 文件 | 删除 |
|---|---|
| `Script/UE/CoreUObject/FText.cs` | `public string ToBufferString()`（4 行） |
| `Script/UE/Library/FTextImplementation.cs` | `__FText_ToBufferStringImplementation` 部分声明 ＋ 包装（18 行） |
| `Source/UnrealCSharp/Private/Domain/Interop/FRegisterText.cpp` | `ToBufferStringImplementation` ＋ `.Function("ToBufferString", …)`（19 行） |
| 工程侧（未版本化） | `TestTextToBufferString()` 及调用（**−2 条断言**） |

> `ToString()` 与 `TextToString` 用例**逐字未动**。

## 4. 判据与回归

| 项 | 读数 |
|---|---|
| 静态 | `ToBufferString` 全仓 **0** 命中（插件 ＋ 工程侧） |
| 运行期 | 套件 **1 980 断言 / 0 失败 ×2 同 SHA `7F0C2E4CD67A805030E007643F79124D6CAC37F911710DEA32D838322B246264`** —— 与 **B-81 基线逐字节相同**（⇒ 撤干净了：只少了那 2 条测例，其余零漂移） |
| 5.6 UBT | **Succeeded**（49.38 s） |
| C# | `dotnet build Game.csproj -c Debug` **0 Warning / 0 Error**（33.37 s）；`UE.dll` 25 413 120 B / `Game.dll` 903 168 B |
| 5.0 门禁 | **0 error**（80.64 s）＋ 复跑 **`Target is up to date`**（5.03 s） |
| §7 / §8 | 语法 **CLEAN** ／ 风格 **PASS**（1 文件、Added **1**、AddedMax **50**、`diff --check` 0） |

## 5. 交付

| 项 | 内容 |
|---|---|
| 补丁 | `Saved/Retest/fix-B89-drop-tobufferstring.patch` —— **2 683 B** / SHA256 **`EEF7032D67A77BE3226C6316DFA97EDCB668B8C9CBD27155CAC4B1BCCF6D2CF1`** / LF / 无 BOM / reverse-check 0 |
| 提交 | 🟢 **`91fc122a`**（父 `987be131`，subject `Revert Text Buffer String API`，**3 文件 `+1/−40`**） |

## 6. 台账回写（本轮）

| 台账 | 动作 |
|---|---|
| [`09-…/01`](../09-问题清单/01-全局问题总表.md) | `F-PROP2-006 (04b)` 由 🟢 已修 改为 **⚫ 决定不做（B-89 撤回）**，写明原因与触发条件 |
| [`09-…/05`](../09-问题清单/05-已完成（P2与P3）.md) | §68 的 `F-PROP2-006 (04b)` 行标注 **已撤回（B-89）** |
| [`08-…/121`](121-轮B-83-新增公开面清掉最后两条G7（2026-10-25）.md) | 加"🔁 `ToBufferString` 半边已被 B-89 撤回"指针 |
| [`09-…/00`](../09-问题清单/00-推荐优先修复清单.md) / [`README.md`](../README.md) / [`INDEX.md`](../INDEX.md) | 轮次注 ／ 行 ／ 条目 |
| `08-…/122`…`126` | **不改**（与本轮回撤无交集；`F-ABI-007` 半边仍有效） |

## 7. 边界与后续

1. `F-PROP2-006 (04b)` 的**能力缺口仍在**（C# 侧无法把命名文本序列化成可解析串）—— 现在登记为"**用户提出真实需求时再做**"，届时按 E2 同时给出消费者（例如用户侧的存档链路）＋ NSLOCTEXT 往返用例。
2. `TryGetStaticValue`（同批 B-83 的另一半）**保留**：它有真实消费者 `FDynamicEnumGenerator.cpp:291`，与本次撤回无关。
3. 本轮未跑 Mono/LeanCLR、PIE、打包版。
