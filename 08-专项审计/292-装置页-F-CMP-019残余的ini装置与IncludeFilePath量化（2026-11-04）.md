# 装置页：`F-CMP-019` 残余的 ini 装置与 `IncludeFilePath` 量化（2026-11-04）

> **性质**：装置 ＋ 量化（**0 行入库**）。对象 = `F-CMP-019` 残余（B 组：绝对 `#include` 半边）。
> **结论**：`DefaultUnrealCSharpEditorSetting.ini` 装置**可用**；全量导出生成了 **1 912** 个 `.inl`（既有 631 是 5.7 会话遗留的陈旧子集）；候选补丁（`IncludeFilePath` ＋ 两级回退）实测 **1 781/1 912 文件变更（93%）**、**0 个空 include**、构建 Succeeded；回退后 **1 912/1 912 逐字节回到基线**。⇒ 决策 #3 所需的改动面与风险点已量化。

---

## 1. 装置（ini 门控 ＋ 强制 UHT）

| 项 | 值 |
|---|---|
| 配置文件 | `Config/DefaultUnrealCSharpEditorSetting.ini`（本工程原本**没有**该文件 ⇒ 导出器在 `File.Exists` 处直接返回） |
| 段/键 | `[/Script/UnrealCSharpCore.UnrealCSharpEditorSetting]` ＋ `bEnableExport=True`；`ExportModule` **留空** ⇒ 导出全部模块（B-218 语义） |
| 触发方式 | 改 ini **不会**让 UBT 重算规则（B-218 §4.2 已证）⇒ 用 **touch 一个 UCLASS 头**（`Source/UnrealCSharpTest/Public/TestBase/TestBaseSubsystem.h`）强制 UHT 重跑；日志出现 `Running Internal UnrealHeaderTool …` |
| 导出结果 | `Plugins/UnrealCSharp/Intermediate/Build/Win64/UnrealEditor/Inc/UnrealCSharp/UHT/*.inl`：**631（陈旧）→ 1 912** |

---

## 2. 基线（原代码，绝对 include）

- 1 912 个文件中 **1 790** 个的首个 include 为绝对路径；全库 `#include "D:\…"` 行 **3 271**。
- 样例：`Actor.binding.inl:10-12` 均形如 `#include "D:\file\UnrealEngine\5.6\Engine\Source\Runtime\Engine\Classes\…"`。

---

## 3. 候选补丁（`IncludeFilePath` ＋ 回退链）

```csharp
private string GetHeaderFile(UhtClass classObj)
{
	var IncludeFilePath = classObj.HeaderFile.IncludeFilePath;

	if (IncludeFilePath.Length == 0)
	{
		IncludeFilePath = classObj.HeaderFile.ModuleRelativeFilePath;
	}

	if (IncludeFilePath.Length == 0)
	{
		IncludeFilePath = classObj.HeaderFile.FilePath;
	}

	return IncludeFilePath;
}
```

| 读数 | 值 |
|---|---|
| 文件变更 | **1 781 / 1 912**（131 未变 = `.header.inl` 列表文件 ＋ 陈旧产物） |
| 首个 include 形态 | **相对 1 903** ／ 绝对 9 ／ **空 0** |
| 全库 `#include "D:\…"` 行 | 3 271 → **11** |
| 构建 | **Succeeded**（60.22 s；`.inl` 无编译期消费者，故不受影响） |
| 样例 | `#include "GameFramework/Actor.h"`、`#include "Components/ActorComponent.h"` |

**剩余 11 行绝对 include 全是陈旧产物**：它们指向 **`D:\file\UnrealEngine\5.7`**（`BlueprintSpringMathLibrary` / `MaterialExpressionAggregate` / `SizeBoxComponent` / `WidgetNavigation` 等），即 5.7 会话生成、**当前 5.6 UHT 不再生成这些类** ⇒ 文件未被重写，与补丁无关。**新观察（登记）**：导出器不清理"不再导出"的类的旧 `.inl`。

---

## 4. 回退与还原

1. 回滚 `Source/SourceCodeGenerator/SourceCodeGenerator.cs` → 重建 UBT 插件 → touch 同一个头 → 构建 ⇒ **1 912/1 912 与基线逐字节相同**（`changed=0`，绝对行回到 3 271）。
2. 删除 ini → 最终构建 **Succeeded**（54.31 s）；`Config/` 恢复原状；插件树与子模块 **0 改动**（仅既有 `SourceCodeGenerator` 指针状态）。
3. **构件层变化（非入库）**：`Intermediate` 下的 `.inl` 由 631 变为 **1 912**（全量导出的一次性产物，均无编译期消费者）。

---

## 5. 对决策 #3 的含义

- **改动面**：1 781 个 `.inl`（93%），均为构件目录内的生成产物，不入库。
- **安全性**：回退链兜底 ⇒ **0 个空 include**；构建不受影响（无消费者）。
- **两个附带条件**：① 落地前需清理陈旧产物（否则 11 行仍指旧引擎路径）；② 生成必须由**装有导出器的 UBT**跑一次（本装置已证明可行，1 912 文件约 25–60 s）。
- **建议执行顺序**（若批准）：清陈旧 `.inl` → 应用补丁 → 全量生成 → 快照 diff（应以 1 781 变更为准）→ UBT 5.6/5.0 构建 ＋ 套件 ×2。

---

## 6. 台账

| 位置 | 变化 |
|---|---|
| `01` | `F-CMP-019` 行补 B-257 量化数据（**级别 P2 / 行数不变**） |
| `06` | B 组该行补"装置可用；改动面 1 781/1 912；0 空 include"（**B 2 / 总 7 不变**） |
| `07` | 补 B-257 行 |
| 代码 | **0 入库**（补丁已回滚，ini 已删除） |
