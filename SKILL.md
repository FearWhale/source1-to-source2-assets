---
name: source1-to-source2-assets
description: 把 Source 1 游戏或模组的材质、模型、粒子移植进 Source 2 工坊插件时使用，含三条独立入口与官方导入工具的调用方式、产出结构和已知信息损失。只做全新资产创作时不要使用。
metadata:
  short-description: Source 1 资产移植进 Source 2
---

# Source 1 → Source 2 资产移植

三类资产的管线机制完全不同，先按类型选入口，再读对应的 reference。下文所有 `<name>` 代表实际的项目、游戏、插件或资产名。

## 通用前提

**工具位置**：Source 2 工坊工具的 `game/bin/win64/` 下。

| 工具 | 用途 |
| --- | --- |
| `source1import.exe` | 材质、贴图、粒子、声音、贴花、字体、地图 |
| `cs_mdl_import.exe` | 模型。`source1import` 已移除 MDL 导入器，用它导模型会报 `Unable to find import func for importer "ImportMDLtoVMDL"` |
| `dmxconvert.exe` | DMX / PCF 格式转换，粒子导入器内部会调用 |
| `resourcecompiler.exe` | 把源文件编译成运行期资源 |

源侧还需要 `vpk.exe`（列目录、解包）和 `vtf2tga.exe`（VTF 转 TGA），一般就在原游戏的 `bin/` 下。

**`source1import.exe` 调用形状**

```powershell
'y' | & "<source2-install>/game/bin/win64/source1import.exe" `
  -src1gameinfodir "<s1-game>/<name>" `
  -game <name> -s2addon <name> `
  -usefilelist <batch>.txt
```

- `-src1gameinfodir` 指向含 `gameinfo.txt` 的目录。当它不是目标游戏时，工具会要求确认，从 stdin 喂 `y` 即可继续（别让它停在交互等待上）。
- 文件清单的相对路径基准是 S1 游戏目录；给**绝对路径**时按磁盘上的确切文件处理。粒子导入必须用绝对路径。
- 先做体检再真导：`-showmatchesonly` 只列能转的，`-show_unknown_only` 只列不认识的，`-logwarnings` 把警告落到插件 game 侧的日志文件。
- 批量清单是放在插件 content 目录下的 kv 文本：

```
importfilelist
{
	"file"	"<dir>/*.vmt"
}
```

**产出与编译**

导入器只写**源文件**到 content 侧（`.vmat` / `.tga` / `.vtex` / `.mks` / `.vmdl` / `.dmx` / `.vpcf`）。运行期资源（`*_vmat_c`、`*_vtex_c`、`*_vpcf_c`…）要在工具加载或构建时才生成到 game 侧；没构建过就解析不到，表现就是紫黑格或空引用。

- **默认不写入 game 目录。** 导入只写 content 侧；编译/构建是独立的一步，只有用户明确要求时才执行，产物才落在 game 侧。
- 例外是导入器自己留下的 `source1import*` 日志（用 `-logwarnings` 时落在插件 game 侧）。这属于工具副作用，不是你的资产；清理前先跟用户确认。

## 入口一：材质

三者中最自动的一环，能从 VPK 直接导入。命令、参数映射、产出结构和已知信息损失见 [references/materials.md](references/materials.md)。

## 入口二：模型

走独立工具导入，而且导入产物**必须手工补材质重映射**才可用。产出结构、重映射模板、外部引用处理见 [references/models.md](references/models.md)。

## 入口三：粒子

必须先落成散文件，精灵贴图还要额外补编译入口。见 [references/particles.md](references/particles.md)。

## 通用验收

1. 每次导入看尾部的 `OK: N imported, M failed, K skipped, J unknown`，`failed` 与 `unknown` 逐条追。
2. 材质核对 `.vmat` 里引用的每张贴图是否落地；模型核对材质槽是否解析、外部引用是否齐全；粒子在粒子编辑器里逐个预览。
3. 源侧统计数量与产出数量对不上时，差额就是要手工处理的清单：先按差额定位，再决定补做还是删引用。
