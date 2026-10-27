# 轮 B-140：`F-HLP-008` 收口 —— `TArray.Remove` 单遍扫描 ＋ 单次 Shrink（2026-10-28）

> **来源**：B-139 立项 ①（[`08-…/173`](173-轮B-139-T3第九次选片与台账收尾（2026-10-28）.md) §3）。
> **一句话结论**：`FArrayHelper::Remove` 从"`Find` 重扫 ＋ 每次 `RemoveAt(..., true)`"改为**单遍索引扫描 ＋ 每次 `RemoveAt(..., false)` ＋ 结尾一次 `Shrink`**，并删掉未使用局部 `TArray<int32> x;`；微基准 **93 ms → 5 ms（−94.6%）**；套件 **1 987 / 0 同 SHA `D520F5AD…`**；产物 **16 155 / 0**；§8 **PASS**。

---

## 1. 量化（先做）

**探针**（临时，已移除）：`TestArray()` 末尾追加两段计时 —— ① `TArray<int>` 10 000 元素（值 = `Index % 200`）＋ 200 轮 `Remove(Round)`（共删 10 000 个元素）→ `B140REMOVE`；② 20 000 次 `new FName("Foo_" + i % 100)` ＋ `ToString()` → `B140NAME`。

| 读数 | 改前 | 改后 |
|---|---|---|
| `B140REMOVE`（10 000 删除） | **93 ms** | **5 ms（−94.6%）** |
| 小规模对照（仅删 200 个） | 7 ms | ——（规模放大后 O(n²)+realloc 才显形） |
| `B140NAME`（20 000 次 FName 往返） | 13–36 ms（波动） | 35 ms（同量级） |

> **`F-PROP2-009 (04b)` 量化结论**：`FName` 往返 **≈0.65–1.8 µs/次**（20 000 次 13–36 ms），**corpus 无消费者、无热路径**，且波动大于差异 ⇒ 按 `F-GEN-016`/`F-MACRO-003` 先例 **降级 P3、本体不施工**（表内级别已改）。

---

## 2. 施工

`Source/UnrealCSharp/Private/Reflection/Container/FArrayHelper.cpp`（**+15/−7**）：

```cpp
int32 FArrayHelper::Remove(const void* InValue) const
{
	auto RemovedNum = 0;

	auto ScriptArrayHelper = CreateHelperFormInnerProperty();

	for (auto Index = 0; Index < Num();)
	{
		if (InnerPropertyDescriptor->Identical(ScriptArrayHelper.GetRawPtr(Index), InValue))
		{
			RemoveAt(Index, 1, false);

			++RemovedNum;
		}
		else
		{
			++Index;
		}
	}

	if (RemovedNum > 0)
	{
		ScriptArray->Shrink(InnerPropertyDescriptor->GetSize(), __STDCPP_DEFAULT_NEW_ALIGNMENT__);
	}

	return RemovedNum;
}
```

- 删除死代码 `TArray<int32> x;`（原 `:389`，未使用局部）。
- `RemoveSingle` **未改**：它只删 1 个元素、只 shrink 1 次，本来 O(n)。
- `GetRawPtr` 每次读 `ScriptArray->GetData()`（`UnrealType.h:175188` 的 `WithScriptArray`）⇒ `RemoveAt` 内的 `Remove`（memmove）后指针仍有效，单实例 helper 可复用。

---

## 3. 判据

| 项 | 读数 |
|---|---|
| 微基准 A/B | **93 → 5 ms**（−94.6%，同一探针、同一会话形态、两次运行） |
| 套件 | **1 987 行 / 0 失败**，SHA **`D520F5AD71AE6B1D9534DA27FAF22FB1C7AEAC110C025F30FFB1B77BFB0C54C6`** —— 与 B-132/B-138 交付态**逐字节相同** |
| 日志 | assert/fatal **0**；`LogUnrealCSharp: Error` **3**（既有判据） |
| 产物 | **16 155 / 0 差异 / `E8FAABB8…`** |
| UBT | **Succeeded**（42.52 s） |
| §8 | `b30-style-lint` **PASS**（1 文件 / AddedMax **92** / Trailing 0）；`b29` **CLEAN** |
| 探针清理 | `grep B140` = **0** |

---

## 4. 边界

1. **容量语义**：改前每删一个元素 shrink 到最小（`Num`）；改后结尾 shrink 一次 ⇒ 中间容量更宽松、结尾与原语义一致（`Shrink` 在 `ArrayNum != ArrayMax` 时才 `ResizeTo`）⇒ 属改善，不改变可观察的 `Num`/元素顺序。
2. **单线程微基准**：未测多线程；`FArrayHelper` 的并发前提沿用既有口径（无锁）。
3. `Remove` 的复杂度从 O(n²) 降到 O(n)（`Identical` 每元素一次）。

---

## 5. 台账

| 台账 | 动作 |
|---|---|
| `F-HLP-008` | 🟢 **已修** —— 迁出 `01` → [`05 §86`](../09-问题清单/05-已完成（P2与P3）.md) |
| `F-PROP2-009 (04b)` | 🟡 **降级 P2 → P3、本体不施工**（量化见 §1；级别单元格已改，事实与编号保留） |
| `01` 计数 | **828 → 827 行**、**497 → 496 条 `| F-` 行**（P2 150 → 148、P3 318 → 319） |

---

## 6. 补丁

`Saved/Retest/fix-B140-array-remove.patch`：**1 018 B** / SHA256 `7D2C1D05DC6FEE2562C357BD1FF653F8093BED405F63BAD48F7CDF6B32D74A9C` / LF / 无 BOM / **1 hunk** / `git apply --reverse --check` = 0。

---

## 7. 提交

| 项 | 内容 |
|---|---|
| 提交 | `08ea0bcf40dfa26463e1621282140989c5f84410`（父 `83d1537c3fce9697c8d2b203141110a1783a91a5`） |
| 作者 / 时间 | crazytuzi `<liuxiangcode@qq.com>` / **2026-10-28 03:55:29 +0800** |
| subject | `FTypeBridge Optional FieldPath && FArrayHelper Remove Shrink` |
| 规模 | **6 文件 `+106/−22`** |
| 等价性 | 提交 blob `7fdf61b5690a84f5c0e111e8f68ddf9e8627be8c` **与交付工作区 blob 完全相同** ⇒ **零调整**（本轮无排版/命名返工）；交付补丁在提交树上 `git apply --reverse --check` = 0 |
| 提交树复跑 | UBT **Succeeded**（`Target is up to date`，10.14 s）；套件 **1 987 / 0 / `D520F5AD…`**；产物 **16 155 / 0 / `E8FAABB8…`**；日志 assert/fatal 0、`LogUnrealCSharp: Error` 3（既有） |
