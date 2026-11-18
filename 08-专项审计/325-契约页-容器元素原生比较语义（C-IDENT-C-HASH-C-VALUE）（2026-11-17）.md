# 契约页 —— 容器元素的原生比较语义（C-IDENT / C-HASH / C-VALUE / C-SCRATCH）（2026-11-17）

> **来源**：轮 B-289 探针读数（[`08-…/324`](324-轮B-289-F-HLP-028探针读数与分支判定（2026-11-17）.md)）。触发 = `F-HLP-028` 的 B-287 / B-288 **两轮实验、两次回滚**（4 条 ＋ 3 条用例失败 ＋ 一次 AV）。
> **适用范围**：任何"把属性值或容器元素当**原生值**比较 / 取哈希 / 做去重"的代码 —— `FSetHelper`、`FMapHelper`、`FArrayHelper`、`FSetPropertyDescriptor` 一族、动态生成器的去重面，以及将来任何"值缓冲 → 原生暂存"的优化。
> **一句话**：插件的**入参缓冲是"句柄"（引用类型）或"原生值"（值类型）的混合体**；`Set` 负责把前者变成后者，而 `Identical` / `GetValueTypeHash` / 字节比较**各有各的语义边界**，混用会静默判错（本项目已因此报废两轮）。

---

## C-IDENT-1：`FPropertyDescriptor::Identical(A, B)` 的 **B 是句柄**，不是原生值

| 项 | 内容 |
|---|---|
| 事实 | `FObjectPropertyDescriptor::Identical`（`Source/UnrealCSharp/Private/Reflection/Property/ObjectProperty/FObjectPropertyDescriptor.cpp:29-37`）：`A` 走 `Property->GetObjectPropertyValue(A)`（原生），`B` 走 `*static_cast<IManagedHandle*>(B)` → `GetObject`（**句柄**）。`FStrPropertyDescriptor::Identical`（`StringProperty/FStrPropertyDescriptor.cpp:49-58`）同形：`B` 走 `GetString<FString>(*static_cast<IManagedHandle*>(B))`。 |
| 后果 | 把 **native 值传给 B** ⇒ 句柄查表失败 ⇒ **静默判不等**。B-289 实测：对象元素 native↔native **0/3** 命中（同内容字符串同理）。这正是 B-288 的 `SetCompoundAddFalse` / `RemoveTrue` / `ContainsTrue` 三条失败的直接原因。 |
| 规则 | **native↔native 比较一律用引擎谓词**：`Property->Identical(A, B, PortFlags)`（对象：`PropertyObject.cpp:113`；数值：`TProperty_Numeric`）；`Descriptor->Identical` 只用于"**B = C# 入参缓冲**"这一种形态（即现有 `Add`/`Remove`/`Contains` 的线性扫描）。 |

## C-HASH-1：`GetValueTypeHash` 是**原生语义**，不得对入参缓冲调用

| 项 | 内容 |
|---|---|
| 事实 | `FPropertyDescriptor::GetValueTypeHash` = `Property->GetValueTypeHash(Src)`（`Public/Reflection/Property/FPropertyDescriptor.inl:85-93`）；对象 = **指针 hash**（`PropertyObject.cpp:390`：`GetTypeHash(GetPropertyValue(Src))`）；字符串 = **内容 hash**（`StrProperty.cpp.inl:157-160`：`GetTypeHash(FString)`）；数值 = 值 hash（`UnrealType.h:1837-1840`）。 |
| 后果 | 对 C# 入参缓冲取 hash ⇒ ① 引用类型得到"句柄位"的 hash（B-289 实测对象 `0x00003BA1` vs 正确 `0x68428F10`）；② `FString` 的 16 字节结构读**越过 8 字节缓冲**（实测退化为空串 hash `0`，相邻栈内容脏时是越界读）。B-287 的 4 条失败即此。 |
| 规则 | 先 `Descriptor->Set(入参, scratch)` 得到 **native scratch**，再对 scratch 取 hash；**任何情况下不得对入参缓冲直接取 hash**。 |

## C-VALUE-1：引用型元素**没有字节恒等**，只有值等价

| 项 | 内容 |
|---|---|
| 事实 | `FString` 的 `Set` 是**复制构造**：B-289 实测同内容元素与 scratch 的 `Num`/`Max` 相同（`0x14`）而 **Data 指针不同**（`0x012A65C700` vs `0x012A65BA00`）。对象元素才逐字节恒等（句柄 → 同一个 `UObject*`）。 |
| 后果 | `Memcmp` 判等对字符串/含堆缓冲的结构体**恒为假**；把它当"快路径"会退化成"永远插入"（集合膨胀/删除失败）。 |
| 规则 | 元素相等一律走**属性语义**（`Property->Identical`）；字节比较只允许用于"已知是 POD 且无指针成员"的调试用途。 |

## C-SCRATCH-1：scratch 的初始化判定 —— `Set` 可能**不初始化**

| 项 | 内容 |
|---|---|
| 事实 | `FObjectPropertyDescriptor::Set`（`:18-27`）先 `Property->InitializeValue(Dest)` 再 `SetObjectPropertyValue`；而 `FStrPropertyDescriptor::Set`（`:37-47`）**只在 `GetString` 成功时才** `InitializeValue` ＋ `SetPropertyValue` —— 句柄无效时**直接返回、不碰 Dest**。 |
| 后果 | 对"没被 `Set` 初始化过"的 scratch 调 `DestroyValue` ⇒ 在垃圾内存上析构 `TArray` ⇒ **AV**（B-288 run#1 的候选根因之一，本轮未复现）。 |
| 规则 | ① scratch 每次使用前清零；② 只对**确定已初始化**的 scratch 调 `DestroyValue`（实现上建议由 `Set` 的返回状态或"入参句柄有效性"先判断）；③ 探针/试验代码可以只 `Free` 裸内存，但交付代码必须给出配对的析构路径。 |

---

## 证据索引

| 读数 | 值 | 出处 |
|---|---|---|
| 对象：native == elem（字节） | 3/3 | 324 §2.1 |
| 对象：`descNativeEq`（native 喂描述符 `Identical`） | **0/3** | 324 §2.1、§3 |
| 对象：`hashRaw` vs `hashNative` | `0x3BA1` vs `0x68428F10` | 324 §2.1 |
| 整数：三谓词全部成立 | 18/18 | 324 §2.2 |
| 字符串：同内容不同缓冲、hash/Identical 一致 | 47/47 | 324 §2.3 |
| 字符串：`hashRaw` 退化为 0 | 48/48 | 324 §2.3 |

## 版本记录

| 日期 | 变更 |
|---|---|
| 2026-11-17 | 新建：B-289 读数落地为 4 条契约（C-IDENT-1 / C-HASH-1 / C-VALUE-1 / C-SCRATCH-1），用于约束 `F-HLP-028` 施工轮（B-290）与所有"原生值比较"类新代码 |
