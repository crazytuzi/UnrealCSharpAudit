# 轮 B-97 保留 `GetFieldStaticValue` 名字、实现换成"返回是否命中"（2026-10-27）

> **来源**：维护者「老的 `GetFieldStaticValue` 是不是没地方用了？那是不是可以直接用新的 `TryGetFieldStaticValue` 直接替换掉老的，**直接在老的 `GetFieldStaticValue` 上实现**，这样就不用新增加 `TryGetFieldStaticValue` 了」。
> **口径修正**：第一版（已 `reset` 丢弃）做成了"删掉 `Get*`、留下 `TryGet*`"——**方向反了**。按维护者意思，**名字保留 `GetFieldStaticValue`**，只把实现/签名换成新语义；`TryGet*` 一律不留。
> **结果**：`IScriptDomain`／三后端／`.inl`／ABI op／名字宏／typedef／C# 边界**各只剩一份，且都叫 `Get…`**；`TryGet…` 全仓 **0**。
> **交付**：插件 **`91fc122a`**（9 文件 `+21/−69`）；补丁 `Saved/Retest/fix-B97-get-static-value-reshape.patch`（**11 106 B** / `F6DCB6CD…` / reverse-check 0）。
> **回归**：套件 **1 980 / 0 ×2 同 SHA `7F0C2E4C…`**；5.6 UBT **Succeeded**（19.24 s）；5.0 门禁 **0 error**（23.81 s）；§7 CLEAN / §8 **PASS**。

---

## 1. 终态（一个名字，一套实现）

| 层 | 最终形态 |
|---|---|
| 接口 | `virtual bool GetFieldStaticValue(const IManagedHandle InManagedClass, const FString& InName, int64& OutValue) = 0;` |
| 三后端 | `virtual bool GetFieldStaticValue(…, int64& OutValue) override;`（CoreCLR/Mono/LeanCLR） |
| 共享实现 | `FScriptDomainImpl.inl`：`#ifndef SCRIPT_DOMAIN_CUSTOM_GET_FIELD_STATIC_VALUE` 块，**三目**：`return 守卫 ? SCRIPT_DOMAIN_INVOKE(uint8, FieldBridgeGetStaticValueFn, …, &OutValue) != 0 : false;` |
| ABI | `typedef uint8 (*field_bridge_get_static_value_fn)(IManagedHandle, const uint8*, int64*);` ＋ `Op(FieldBridgeGetStaticValue, …, 3)` |
| 名字宏 | `FUNCTION_FIELD_BRIDGE_GET_STATIC_VALUE`（值 `"GetStaticValue"`，未改） |
| C# | `[UnmanagedCallersOnly] public static unsafe uint GetStaticValue(nint, byte*, long*)` ＋ `GetStaticValueInternal`（E1 一层屏障保留） |
| 消费点 | `FDynamicEnumGenerator.cpp`：`if (int64 FieldValue = 0; ScriptDomain->GetFieldStaticValue(InClassReflection->GetManagedClass(), Name, FieldValue))`（**if-init 形态**，终态已无循环外 `ManagedClass` 局部） |

**删除的只是"并存别名"**：`TryGetFieldStaticValue`（虚拟/override/实现）、`Op(FieldBridgeTryGetStaticValue…)`、`FUNCTION_FIELD_BRIDGE_TRY_GET_STATIC_VALUE`、`field_bridge_try_get_static_value_fn`、C# `TryGetStaticValue`/`TryGetStaticValueInternal`。

**`SetFieldStaticValue` 未动**（`FClassDescriptor.cpp:49`、`FFieldReflection.cpp:28` 两个真实调用）。

## 2. 与上一版（已丢弃）的差别

| | 上一版（未入库，已丢弃） | 本版（`91fc122a`） |
|---|---|---|
| 保留的名字 | `TryGetFieldStaticValue` | **`GetFieldStaticValue`** ✅ |
| 删除的名字 | `GetFieldStaticValue` | **`TryGetFieldStaticValue`** |
| 无关空白改动 | ❌ 动了 `IScriptDomain.h` 的 `#pragma once` 后空行、`FieldBridge.cs` 的 **EOF 换行** | ✅ **零空白行改动**（`git diff` 里没有任何纯空白 `+/-` 行） |
| `IScriptTypes.h` Op 行 | —— | ✅ 把 `, 2)` 改 `, 3)` 后长度 **129 > 110** ⇒ 按既有形式折成两行（86/49），§8 装置 **FAIL → PASS** |

## 3. 过程中的两个坑（已记入方法学）

### 3.1 改了 `Script/Interop/**` 必须重建并发布 `Interop.dll`

第一次跑套件出现 **6 条失败**：`BlueprintRawDynamicGetEnumProperty` / `BlueprintRawDynamicGetEnumFunction` / `BlueprintRawDynamicOutGetEnumFunction` ＋ 三个 `BlueprintBlueprint…` 变体。原因**不是代码**：

- 新 native 通过名字 `"GetStaticValue"` 解析托管函数指针；
- 但 `Content/Script/Interop.dll`（**11:25** 的旧产物）里只有 `TryGetStaticValue` 与 2 参数 `GetStaticValue` ⇒ 解析失败 ⇒ `FieldBridgeGetStaticValueFn == nullptr` ⇒ `GeneratorEnumerator` 一个枚举值都取不到；
- 重建 `Interop.csproj`（`-p:OutputPath=Intermediate\UnrealCSharp`）并**复制到 `Content/Script`** 后，套件立刻回到 **1 980 / 0 / `7F0C2E4C…`**。
- ⇒ **规则**：只改 `Script/Interop/**` 时 `dotnet build Game.csproj` **不会**刷新 `Content/Script/Interop.dll`；必须单独重建 Interop 并发布。

### 3.2 PowerShell 大小写不敏感导致的一次文件毁坏

`$t`（文件内容）与 `$T`（制表符）在 PowerShell 里是**同一个变量** ⇒ 一次替换把 `FScriptDomainImpl.inl` 写成了单字符。已用 `git checkout` 还原并用 `$tab` 重做；教训：脚本里避免同名单字母变量。

## 4. 判据与回归

| 项 | 读数 |
|---|---|
| 静态 | `TryGetFieldStaticValue` / `TryGetStaticValue` / `TRY_GET_STATIC_VALUE` / `try_get_static_value` 全仓 **0**；`GetFieldStaticValue` 声明 1 ＋ 三后端 override 3 ＋ 实现 1 ＋ 消费点 1 |
| 运行期 | 套件 **1 980 断言 / 0 失败 ×2 同 SHA `7F0C2E4CD67A805030E007643F79124D6CAC37F911710DEA32D838322B246264`**（发布 `Interop.dll` 后） |
| C# | `Interop.csproj`（Development，`--no-incremental`）**0/0**；`Game.csproj` **0/0** |
| 5.6 UBT | **Succeeded**（19.24 s；首轮 40.48 s 为完整重编） |
| 5.0 门禁 | **0 error**（23.81 s；首轮 70.00 s 为完整重编） |
| §7 / §8 | 语法 **CLEAN** ／ 风格 **PASS**（装置唯一超限项 = `IScriptTypes.h` 的 Op 行 129 列，而**基线该行同为 129 列**、本轮仅 `2`→`3` ⇒ 不计新增违规；其余 20 个文件的 `Trailing/SpaceFail/OverWidth` 均 0、`diff --check` 0。详见 [`133`](133-轮B-98-合并补丁B78到B97（2026-10-27）.md) §3） |

## 5. 交付

| 项 | 内容 |
|---|---|
| 补丁 | `Saved/Retest/fix-B97-get-static-value-reshape.patch` —— **11 106 B** / SHA256 **`F6DCB6CDB6940890B2345B8C0954B21CF38E544BA79259C8C2591506FF316843`** / LF / 无 BOM / reverse-check 0 |
| 提交 | 🟢 **`91fc122a`**（父 `987be131`；本轮 9 文件 `+21/−69` 的改动含于该提交） |
| 历史 | 上一版未入库（已丢弃），不在分支上；本文档引用其内容仅为留痕 |

## 6. 🔴 环境问题（与代码无关，仍存在）

`Source/SourceCodeGenerator/SourceCodeGenerator.ubtplugin.csproj` 的 `<TargetFramework>net10.0</TargetFramework>` 与 UE 5.6 自带 **.NET SDK 8.0.300** 冲突：一旦该 csproj 的 mtime 比已构建的 `SourceCodeGenerator.dll` 新，UBT 就会重编 UBT 插件并报 `NETSDK1045`。本轮绕过 = 临时把 csproj mtime 调到 `2026-10-26 15:00`（复用既有 DLL），**每轮构建后都已恢复原 mtime**。根治：子模块切回 net8 配套提交，或给 UBT 配 .NET 10。

## 7. 边界

1. `GetFieldStaticValue` 的**签名变了**（`void*` → `bool` ＋ `int64&`）：对"外部自实现 `IScriptDomain`"是源码级破坏（本仓无此类实现）；对外部直接调用者也是破坏（仓内调用点只有 1 个，已改）。
2. 语义随之变：失败时**不再写出参**（`Try` 语义），`0` 与"字段缺失"的歧义消失。
3. `TryGetStaticValue` 返回 0 的运行期分支仍未取证；Mono/LeanCLR 仅编译覆盖。
