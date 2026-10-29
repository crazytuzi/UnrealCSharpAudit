# 轮 B-176：T4-5 `F-DEL-010` 设计裁定 —— `FOptionalHelper::Get()` 的未 set 契约（2026-10-29）

> **来源**：[`09-问题清单/07-P2优先级评估（2026-10-28）.md`](../09-问题清单/07-P2优先级评估（2026-10-28）.md) §5 T4-5；原条目 [`01-UnrealCSharp运行时/07b`](../01-UnrealCSharp运行时/07b-委托Handler与OptionalHelper.md) §F-DEL-010。
> **性质**：**设计页（0 代码）**。
> **一句话结论**：审计的核心风险"未 `IsSet()` 就 `Get()` → 把未构造内存当 `FString`/`TArray` 解箱"**在 HEAD 的互操作边界已被堵住** —— `FRegisterOptional::GetImplementation` 先判 `IsSet()`（`:130`），C# `TOptional_GetImplementation` 对 handle 0 返回 `default`（`:57-71`）。**残余只剩 helper 自身的契约**：`FOptionalHelper::Get()`（`:94-97`）仍无条件调 `GetValuePointerForReadOrReplace`，在 Development 下对未 set 的 optional 触发 `checkSlow`、Shipping 下返回未构造值地址；今天唯一调用方是安全的，但这是**未写进接口的隐式前提**。建议 **P1 在 helper 内补 `IsSet()` 保护并返回 `nullptr`（2–3 行、无新公开面）**；**P2 仅补文档**（C# 侧 `Get()` 已按 `default` 收敛）。

---

## 1. HEAD 事实（`4564008d`）

### 1.1 helper（`Source/UnrealCSharp/Private/Reflection/Optional/FOptionalHelper.cpp`）

```cpp
bool FOptionalHelper::IsSet() const            // :89-92
{
	return OptionalProperty->IsSet(Data);
}

void* FOptionalHelper::Get() const             // :94-97
{
	return OptionalProperty->GetValuePointerForReadOrReplace(Data);
}

void FOptionalHelper::Reset() const            // :84-87
{
	OptionalProperty->MarkUnset(Data);
}
```

- `Data` 在 helper 构造时分配（`:15-26`），生存期内地址恒定；`GetValuePointerForReadOrReplace` 恒返回 `Data` 本身（引擎 `PropertyOptional.h`），**不存在"set/unset 导致指针失效"**（审计已自行撤销该子论断）。
- `Get()` 不判 `OptionalProperty != nullptr`、不判 `Data != nullptr`、不判 `IsSet()`。

### 1.2 互操作边界（`Source/UnrealCSharp/Private/Domain/Interop/FRegisterOptional.cpp`）

```cpp
static IManagedHandle GetImplementation(const IManagedHandle InManagedHandle)   // :124-140
{
	IManagedHandle ReturnValue{};

	if (const auto OptionalHelper = FCSharpEnvironment::GetEnvironment().GetOptional(InManagedHandle))
	{
		if (OptionalHelper->IsSet())                       // :130  ← 已在这里挡住未 set
		{
			const auto Value = OptionalHelper->Get();

			OptionalHelper->GetValuePropertyDescriptor()->Get(Value, reinterpret_cast<void**>(&ReturnValue),
			                                                  FPropertyArgument::FMember());
		}
	}

	return ReturnValue;
}
```

### 1.3 C# 侧（`Script/UE/Library/TOptionalImplementation.cs:55-72`）

```csharp
public static unsafe T TOptional_GetImplementation<T>(nint InOptional)
{
	var Handle = __TOptional_GetImplementation(InOptional);   // ffi → GetImplementation

	if (Handle != 0)
	{
		…
		return (T)Object;
	}

	return default;                                          // 未 set ⇒ default
}
```

---

## 2. 与审计原文的差异

| 审计原文（`07b:1279-1347`） | HEAD 现状 |
|---|---|
| `Get()` 不检查 `IsSet()`，调用方立刻解箱未初始化内存 | 🟡 **互操作边界已修**：`FRegisterOptional.cpp:130` 先 `IsSet()`；C# 对 0 返回 `default` |
| "返回的指针可能失效" | ❌ 审计已自行撤销（指针恒为 `Data`） |
| 建议 `Get()` 判 `IsSet` 并返回 `nullptr` | 🟡 **helper 内仍未做**；今天唯一调用方自己判了，未来调用方没有契约保护 |

---

## 3. 剩余缺口

| ID | 缺口 | 影响 |
|---|---|---|
| **G1** | `FOptionalHelper::Get()` 对"未 set / `OptionalProperty == nullptr` / `Data == nullptr`"无保护 | Development 下违反 `checkSlow(Data && IsSet(Data))`；Shipping 下返回未构造值地址。**当前唯一调用方安全**，属"未写进接口的隐式前提" |
| **G2** | `IsSet()`/`Reset()` 同样不判 `OptionalProperty`/`Data` 为空 | 与 F-DEL-011（helper 释放后访问）重叠，**本页不处理**，只标注 |
| **G3** | C# `TOptional<T>.Get()` 未 set 时返回 `default`，但无 XML 文档说明 | 使用方可预期，但契约未写明；G7 禁新增注释，只能进维护文档 |
| **G4** | `Get()` 返回 `void*` 被上层立即装箱，生命周期限于调用内 | 当前实现正确；无缓存，无悬垂（F-DEL-011 范畴） |

---

## 4. 方案对比

| 方案 | 内容 | 改动面 | 风险 | 判定 |
|---|---|---|---|---|
| **A. helper 内保护（P1）** | `Get()` 首行：`if (OptionalProperty == nullptr \|\| Data == nullptr \|\| !OptionalProperty->IsSet(Data)) { return nullptr; }` | 1 文件 3 行 | 低：`nullptr` 是既有值域（Q2）；唯一调用方已先判 `IsSet`，行为不变；未来调用方拿 `nullptr` 可判 | ✅ **推荐 P1** |
| **B. 加 `TryGet`/改签名** | `bool TryGet(void*& Out)` | 公开 API（G7 禁） | — | ❌ 不做 |
| **C. 加 `check`/日志** | 断言或打日志 | G7 禁 | — | ❌ 不做 |
| **D. 只补文档** | 维护文档写明"调用前必须 IsSet" | 0 代码 | 现状可用但契约仍隐式 | 🟡 **随 P1 一起做** |

---

## 5. 裁定

1. **P1 = A**：`FOptionalHelper::Get()` 对 `OptionalProperty`/`Data`/`IsSet` 三重判空后返回 `nullptr`；`FRegisterOptional` 无需改动（它已先判 `IsSet`，`Get()` 不会返回 `nullptr`）。
2. **P2 = D**：在维护文档/接口说明里写明"`Get()` 仅在 `IsSet()` 为真时返回有效指针，否则 `nullptr`"；C# 侧 `Get()` 未 set ⇒ `default` 的语义一并写明。
3. **G2 不并入**：`IsSet()`/`Reset()` 的空指针保护属 F-DEL-011，避免与本条混做。
4. 不改 `void*` 返回类型、不加新 API/日志/断言。

---

## 6. P1 验收判据（施工轮照此执行）

| 项 | 判据 |
|---|---|
| 未 set | C++ 探针：构造未 set 的 `FOptionalHelper`（或 C# `new TOptional<FString>()` 后直接取）⇒ `Get() == nullptr`、`IsSet() == false`，Development 构建**不触发** `checkSlow`（探针后移除） |
| 已 set | `Set(FString("x"))` 后 `Get() != nullptr`、装箱值正确；C# `Get()` 返回 `"x"` |
| 回归 | UBT 5.6/5.0 Succeeded；C# 0/0；套件 **1 987/0 同 SHA**；§7 CLEAN / §8 PASS；0 新增日志/断言/公开 API |

---

## 7. 台账

| 台账 | 动作 |
|---|---|
| `07` | §5 T4-5 行标「设计已出（B-176）」；§7 B-176 完成；下一轮 **B-177 = `F-CS1-002` 设计** |
| `01` | `F-DEL-010` 行内追加「B-176 设计已出：互操作边界已修（`IsSet` 守卫）；残余仅 helper 契约；P1 可施工」；**级别维持 P2** |
| `00` / `README` / `INDEX` | 追加轮 B-176 段落 |
| 代码 | **0 改动**；插件树 `4564008d` |

---

## 8. 下一步

- **B-177**：`F-CS1-002`（强句柄泄漏残余）设计裁定。
- 之后 T4 剩余部分修残余复核：`F-ARCH-009` 的 P1 施工排期、`F-DEL-010`/`F-CS1-002` 的 P1 施工可合并为一个小批次。
