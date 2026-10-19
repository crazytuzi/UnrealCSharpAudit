# 轮 B-44 探针：compound 批量枚举协议**受阻于 weak handle**（回滚，不施工）

> **来源**：维护者指定"开始 补 compound 句柄数组协议"（承接 B-42/B-43 的值类型批量）。**结论**：🟥 **受阻，回滚** —— 协议本身能产出正确句柄，但 **`descriptor->Get` 创建的是 weak handle**，批量期间会被 GC 回收，C# 读到 null。**插件工作树已还原到 `87dac201`（0 改动、无提交）**。
> **口径**：G7 / [`08-…/66`](66-待决策页-三问（Q1四路线·Q2口径例外·Q3失败判据）（2026-10-15）.md) Q2。**本报告只登记证据与设计选项，不产生插件改动**。

---

## 0. 结论摘要

| 项 | 读数 |
|---|---|
| **尝试的方案** | 把三个 bulk 函数统一为"按 `descriptor->Get(Src, (void**)slot)` 写槽位"（槽宽 = `GetBufferSize()`，compound = 8），C# 侧 `typeof(T).IsValueType` 决定读字节还是读句柄 |
| **句柄正确性** | 🟢 **正确**：`TArray<FString>` 取第 0 个元素，bulk 句柄 = direct 句柄（`bulk=54470 direct=54470`） |
| **批量可用性** | 🔴 **失败**：chunk=4 时首次 `foreach` 即 `NullReferenceException`（`Value.ToString()`，wrapper 为 null）；chunk=1 时 `TArray<FString>` 20×10 000 跑完（104 ms），`TMap<FString,FString>` 批量崩溃 |
| **套件** | 🔴 失败态：CSV 只到 **1 695 行**（正常 1 963），`B44-bulk` 运行在 `TestCompoundEnumeratorPerf` 处中断（栈顶 `Value.ToString()`） |
| **根因** | `FStrPropertyDescriptor::Get(..., FMember)` 用 `Class->NewObject(true)` ⇒ `HandleData.Alloc(Object, weak=true)`；**批量一次创建 N 个 wrapper，期间 GC 可回收先前 wrapper**；C# 拿到句柄时对象已不在 |
| **为什么逐元素没事** | 每次只创建一个 wrapper，native 返回后**立即** `GetObject` 并由调用方强引用；窗口极小 |
| **处置** | **回滚**（7 个文件 `git checkout` 回 `87dac201`）＋ **登记设计选项**（见 §3） |

---

## 1. 证据链

### 1.1 句柄值正确（排除"写错槽位/类型"）

临时探针（已随回滚移除）：

```csharp
var Handle = HandleData.GetHandle(Array);
var Buffer = new byte[8];
// fixed + TArray_GetElementsImplementation(Handle, 0, 1, Buffer, 8)
var Bulk = BitConverter.ToInt64(Buffer, 0);
var Direct = TArrayImplementation.TArray_GetCompoundImplementation<FString>(Handle, 0);
// 输出：[B44PERF] bulk=54470 direct=54470
```

### 1.2 批量越大越容易崩（GC 窗口）

| chunk | `TArray<FString>` 20×10 000 | `TMap<FString,FString>` 20×10 000 |
|---|---|---|
| 4 | 🔴 首次 foreach 即 NRE | ——（未跑到） |
| 1 | 🟢 104 ms（≈ 逐元素基线 107 ms） | 🔴 崩溃 |

> chunk=1 时数组跑完但 map 仍崩（map 的 chunk 仍是 256）⇒ 与"批量创建数量"直接相关，而非容器逻辑。

### 1.3 崩溃栈（`B44-bulk`）

```
System.NullReferenceException
   at UUnitTestSubsystem.TestCompoundEnumeratorPerf() ... line 92   // Sum += Value.ToString().Length
```

`Value` 是 `(T)HandleData.GetObject(handle)` 的返回值；句柄存在但 `GCHandle.Target` 已为 null ⇒ 对象被回收。

---

## 2. 根因

1. compound 的 `Get` 走 `Class->NewObject(true)`（`FStrPropertyDescriptor.cpp` 等），`ObjectBridge.NewObject(handle, bIsWeak=1)` ⇒ `HandleData.Alloc(Object, weak)`。
2. weak GCHandle **不保活**对象；批量循环里每创建一个 wrapper 都可能触发 GC，**先创建的会被回收**。
3. C# 在 native 返回后才逐个 `GetObject`，此时前面的 wrapper 已经 null。
4. 逐元素路径每次只创建一个并立即物化，故不受影响（`Array[0]` 实测 `first=X`）。

---

## 3. 解除阻断的选项（均需维护者裁定，本轮不做）

| 选项 | 内容 | 代价 / 风险 |
|---|---|---|
| **A. strong-handle bulk 通路** | 给 descriptor 增加"强句柄"取值方式（`NewObject(false)`）或在 bulk 内重新分配强句柄；C# 读完显式释放 | 需要**定义释放责任**（谁在何时 `Free`），涉及 `FClassReflection::NewObject` / string·struct·object descriptor / 句柄生命周期 —— 属新设计 |
| **B. `GC.TryStartNoGCRegion` 包住批量** | C# 在"native 批量 + 句柄物化成 `object[]`"期间禁止 GC；物化完成后结束区域，再 `yield` | 全局 GC 抑制（时长 = 一个 chunk）；预算失败要兜底；**不能跨 `yield`** ⇒ 每块先物化一个 `object[]`；行为面大 |
| **C. 保持逐元素**（现状） | 不做 compound 批量 | 正确性优先；compound `foreach` 仍是 2/3/4 次 P/Invoke/元素 |

**建议**：先按 **C** 收口（本报告即登记），若将来 compound 遍历成为量化热点，再按 **A** 立专项（strong-handle 所有权设计）。

---

## 4. 回滚与验证

| 项 | 读数 |
|---|---|
| 回滚 | 7 个文件 `git checkout --`（`TArray.cs`/`TMap.cs`/`TSet.cs`/`TArrayImplementation.cs`/`FRegisterArray.cpp`/`FRegisterMap.cpp`/`FRegisterSet.cpp`）；插件 `git status` 仅剩动工前既有的 `Source/SourceCodeGenerator` 指针 |
| 插件 HEAD | **`87dac201`**（B-43 交付，**无新提交**） |
| 复跑 | `-game` 套件 **1 963 / 0**，CSV `150779588C88EE00DC0F5270B010FD9B0E25063A0FD14194A1A3F7D11E2B3870`（与 B-37…B-43 **逐字节相同**） |
| 探针 | `grep B44PERF` = **0**；`TestCompoundEnumeratorPerf` 已移除 |

---

## 5. 台账回写

| 台账 | 动作 |
|---|---|
| [`09-…/01`](../09-问题清单/01-全局问题总表.md) | `F-PERF-025`：compound 边界从"未做"改述为 **"受阻（weak handle）"**，附证据与三个选项 |
| [`09-…/00`](../09-问题清单/00-推荐优先修复清单.md) | §0.0 新增"轮 B-44"注（探针轮、0 改动、受阻登记） |
| [`09-…/05`](../09-问题清单/05-已完成（P2与P3）.md) | 新增 **§44 轮 B-44 探针** |
| [`README.md`](../README.md) / [`INDEX.md`](../INDEX.md) | 轮次行 ＋ 报告条目 |

---

## 6. 复用价值

1. **"批量句柄"必须区分 weak/strong**：句柄值正确 ≠ 可用；weak GCHandle 在批量创建期间会被回收。任何"批量返回托管对象句柄"的协议，先问"谁保活"。
2. **chunk=1 是最小判别实验**：批量崩、逐个不崩 ⇒ 窗口/保活问题，而不是编码或类型问题。
3. **探针失败也要留完整读数**：本报告保留 `bulk=54470 direct=54470`、chunk=4/1 对照、崩溃栈与套件行数（1 695 vs 1 963），使"回滚"是可复核的决定而不是猜测。
