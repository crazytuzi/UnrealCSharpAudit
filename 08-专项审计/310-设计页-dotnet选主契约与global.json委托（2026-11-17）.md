# 设计页：`dotnet` 选主契约（`global.json` / feature band / shim）（`F-CMP-036` 残余）（2026-11-17，v2 修订）

> **性质**：设计页（**0 行代码**）。对象 = `F-CMP-036` 残余（B-271 部分修后剩下的"不读 `global.json`、不比 patch/feature band、shim 假阴性"）。
> **修订说明（v1 → v2）**：v1 曾建议"委托 `dotnet --version`"。**该建议已否决** —— 这套 `<exe>/../sdk/*` 目录扫描正是**为了不启动子进程**才引入的（`01ad70e3`，2026-11-13）；v1 违背了这条历史约束，故重写。
> **一句话（v2）**：**维持零子进程**。`IsDotNetSdkSupported` 只是**候选排序**，不是"构建一定成功"的判定 —— `GetDotNet()` 的 `Fallback` 保证任何情况下都会返回一个可执行文件；`global.json` / `rollForward` / feature band 由**构建期的 CLI 自己解析**，插件不复制这些规则。真正剩下的只是"多安装时的排序精度"（可选改进 B-lite），当前**不需要改代码**。

---

## 0. 结论摘要

| 项 | 结论 |
|---|---|
| 历史约束 | 自动选主**刻意不跑子进程**（`GetDotNetExecutablePaths` + `GetDotNetSdkVersion` 纯文件系统；`01ad70e3` 引入）；面板的 `--list-sdks` 是**用户触发**的编辑器侧另一条路径（`UnrealCSharpEditorSetting.cpp:148-152`） |
| 现状再评估 | `IsDotNetSdkSupported` 只决定**谁先返回**；不支持的候选仍进 `Fallback` ⇒ **谓词从不阻断**，最坏情况也只是"排序不同" |
| 三个"反例"的真实后果 | g2/g4/g5 下插件仍返回 `11.0.100`（Fallback），**构建期 CLI 自己报错**并给出 `Requested SDK version` 与 `global.json file` —— 信息完整、无静默错误 |
| 真正残余 | **多安装**场景：`global.json` 钉的版本只在安装 A 里有、安装 B 没有时，按"major ≥ 目标"排序可能先选到 B ⇒ CLI 报"找不到"（本机只有 1 个 SDK，未实测） |
| 建议 | **不改代码**（当前）；若将来出现多安装实证，做 **B-lite**（只读 `global.json` 的 `sdk.version` 用于排序，仍零子进程） |
| 明确不做 | 不跑 `dotnet --version`、不实现 6 种 `rollForward`、不复制 feature band 规则、不新增日志（G7） |

---

## 1. 现状（代码锚点）

| 位置 | 行为 |
|---|---|
| `GetDotNet()` `:45-82` | 面板路径优先 → 遍历 `GetDotNetExecutablePaths()`：**首个 `IsDotNetSdkSupported()` 为真的胜出**；否则返回 `Fallback`（首个候选）；都没有 → 硬编码默认路径 |
| `IsDotNetSdkSupported()` `:1845-1857` | `GetDotNetSdkVersion()` 取 `<exe>/../sdk/*` 的最大 major → `major >= 目标 major` |
| `GetDotNetExecutablePaths()` `:1901-1962` | PATH ＋ `DOTNET_ROOT` ＋ `LOCALAPPDATA`/`ProgramFiles`（Linux：`/usr/share`、`/usr/lib`、`/usr/lib64`）——**全部文件系统操作，无子进程** |
| 面板 `UUnrealCSharpEditorSetting::GetDotNetPathArray()` | `SyncProcess(dotnet, "--list-sdks")` —— 子进程，但只在用户展开面板/刷新选项时跑 |

`DOTNET_MAJOR_VERSION`/`MINOR`：`ThirdParty/Mono/Mono.Build.cs:13-14` 写死 `10/0`；`CoreCLR`/`LeanCLR` 由参数注入；目标框架 `net10.0`（`Script/CodeAnalysis/CodeAnalysis.csproj:9`）。

---

## 2. 实测（本机：`C:\Program Files\dotnet\dotnet.exe`，已装 SDK 仅 `11.0.100`）

| # | fixture | `dotnet --version`（CLI 视角） | 插件实际返回 | 后果 |
|---|---|---|---|---|
| g0 | 无 `global.json` | `11.0.100`，exit 0 | `11.0.100` | ✅ 一致 |
| g1 | 钉 `11.0.100` | `11.0.100`，exit 0 | 同上 | ✅ |
| g2 | 钉 `10.0.100`（未装） | **exit `-2147450725`**：`A compatible .NET SDK was not found. Requested SDK version: 10.0.100` / `global.json file: …` | **仍返回 `11.0.100`**（谓词 `11 ≥ 10` 为真） | 构建期 CLI 报上述错误 ⇒ **可接受**（信息完整） |
| g3 | g2 ＋ `latestMajor` | `11.0.100`，exit 0 | `11.0.100` | ✅ |
| g4 | g2 ＋ `disable` | exit `-2147450725` | `11.0.100` | 同 g2 ⇒ 可接受 |
| g5 | `global.json` 在**父目录** | exit `-2147450725`（向上找到） | `11.0.100` | 同 g2 ⇒ 可接受 |

**结论修正**：v1 把"插件判定可用"误读成"插件会成功构建"。实际上插件只**选一个可执行文件**，是否可构建由 `dotnet build` 在**项目目录**下自己判定（`global.json` 由上溯发现）——所以 g2/g4/g5 的输出是正确的：CLI 给出精确错误，插件没有掩盖它。

**唯一构造性风险（g6，未实测）**：两个安装根 A/B，`global.json` 钉 `9.0.100`（只在 A 有 install），B 只有 10.0.100 ⇒ 谓词 `10 ≥ 9` 为真且 B 排在 A 前 ⇒ 插件返回 B ⇒ CLI 在 B 的根下找不到 9.0.100 ⇒ 报错（而用 A 本可成功）。**本机只有一个 SDK，无法构造**。

---

## 3. 契约（v2）

| 编号 | 契约 | 判据 |
|---|---|---|
| **C-DOTNET-ORDER-1** | `IsDotNetSdkSupported` 是**候选排序**，不是构建成败判定；`GetDotNet()` 必须始终返回一个路径（面板 → 排序结果 → `Fallback` → 硬编码默认） | 三个反例 fixture 下插件仍返回可用路径 |
| **C-DOTNET-ORDER-2** | 这条路径**零子进程**：不得改成 `dotnet --version`；`global.json`/`rollForward`/feature band 由**构建期 CLI** 解析，插件不复制规则 | 代码审计：`GetDotNet*` 内无 `CreateProc`/`SyncProcess` |
| **C-DOTNET-ORDER-3** | 若将来做排序改进，只能**读 `global.json` 的 `sdk.version` 用于排序**（B-lite），不得据此**拒绝**候选；解析失败 ⇒ 静默回落到现状 | 改动后 g0–g5 行为不变、g6 通过 |
| **C-DOTNET-ORDER-4** | 与 `global.json` 相关的失败信息由 CLI 输出（含 `Requested SDK version` 与 `global.json file`）；插件**不新增**日志/断言（G7） | G7 机械读数 0/0/0/0 |

---

## 4. 方案对比（v2）

| 方案 | 做法 | 子进程 | 结论 |
|---|---|---|---|
| **A（现状）** | 目录扫描 + 排序 + Fallback | **无** | **保留**：g2/g4/g5 行为可接受；剩余仅 g6 排序精度 |
| **B-lite（可选改进）** | 读 `global.json`（仅 `sdk.version`）→ 排序时优先"安装根里含该 version 的候选"；解析失败不动 | **无** | 若出现 g6 实证再做（B-275）；`Json` 已是 `UnrealCSharpCore` 的私有依赖（`.Build.cs:77`） |
| C（委托 `--version`） | 每次探测起子进程 | 有 | ❌ **否决**：违背历史约束（本轮 v1 的建议，已撤回） |
| D（检测到 `global.json` 就取首个候选） | 放弃排序 | 无 | 比 A 更差（把"排序"也丢了），不取 |

---

## 5. B-lite 实现轮廓（仅当需要时）

```cpp
// 仅用于排序：不改变"是否返回候选"，只改变顺序
static bool HasPinnedSdk(const FString& InExePath, const FString& InPinnedVersion)
{
	// <exe>/../sdk/ 下是否存在以 InPinnedVersion 为前缀的目录
	// 前缀匹配 = "10.0" 覆盖 "10.0.1xx"；不做 rollForward 表
}
```

要点：
- 只在 `GetDotNet()` 内部生效（编辑器侧，`#if WITH_EDITOR`）；
- `global.json` 查找：`FPaths::ProjectDir()` **向上**逐级（g5 实测 CLI 也上溯）；
- 解析失败/字段缺失 ⇒ 返回 false（不影响现状排序）；
- **不**新增日志、**不**拒绝候选 —— 保持 C-ORDER-1/3。

---

## 6. 判据（若做 B-lite）

| # | 判据 | 通过线 |
|---|---|---|
| 1 | g0–g5：插件返回路径与 CLI 结果**语义**一致（CLI 失败时插件不掩盖） | 6/6 |
| 2 | g6（多安装）：钉 `9.0.100`、A 有 B 无 ⇒ 返回 A | 需第二份 SDK 安装（本机未取得） |
| 3 | 零子进程 | `GetDotNet*` 内无 `CreateProc`/`SyncProcess` |
| 4 | G7 | 0/0/0/0 |
| 5 | 回归 | 5.6 UBT Succeeded ＋ 套件与 `A4BFF990…` 同 SHA（该路径不参与 `-game`） |

---

## 7. 风险与未取得

| # | 项 | 处置 |
|---|---|---|
| 1 | **多安装 / g6 未实测**（本机仅 11.0.100） | 保留为触发条件；有第二份 SDK 时补 fixture |
| 2 | feature band 规则未实测 | **不需要**（CLI 解析；插件只做前缀排序） |
| 3 | `global.json` 上溯范围依赖 CLI 版本 | B-lite 按"向上到根"实现（与 g5 一致），失败即回落 |
| 4 | 面板与自动选主不同源 | 面板走 `--list-sdks`（用户触发）；**本轮不统一**，登记为已知差异 |
| 5 | shim（无 `sdk/` 同级）被判 false | 无害：进 `Fallback` 仍会被返回；仅影响排序 |

---

## 8. 触发与后续

| 项 | 内容 |
|---|---|
| 当前动作 | **不改代码**；契约页作为"为什么这样设计"的依据 |
| 触发条件 | 出现 **多安装 + `global.json` 钉旧 major** 的实证（g6）⇒ 才做 B-lite（B-275） |
| 关联 | `F-CMP-036` 维持 **P3**；残余由"不读 global.json"改写为"**多安装排序精度**（g6，未实测）" |
| 明确撤回 | v1 的"委托 `dotnet --version`"方案 —— 违背零子进程约束 |

---

## 9. 版本记录

| 日期 | 变更 |
|---|---|
| 2026-11-17 | v1：实测 6 个 fixture，建议"委托 `dotnet --version`" |
| 2026-11-17 | **v2 修订**：补记历史约束（零子进程，`01ad70e3`），**撤回 v1 建议**；重新解释三个反例（Fallback ⇒ 谓词不阻断，CLI 报错可接受）；结论改为"不改代码"，残余收窄为多安装排序（g6）；契约改为 ORDER-1..4 |
