# 地图入口

Source 1 的地图必须先还原成 `.vmf` 才能导入，因为编译器只接受 VMF 作为地图输入。

## 管线

```
<name>.bsp ──BSPSource 反编译──> <name>.vmf ──补顶层块──> source1import ──> <name>.vmap + prefab
```

## 第一步：反编译 BSP

用 BSPSource（Windows 整合包自带 Java 运行时，不需要额外装 JDK）：

```powershell
& "<bspsrc>/bin/java.exe" -m info.ata4.bspsrc.app/info.ata4.bspsrc.app.src.BspSourceLauncher `
  -o "<out>/maps/<name>.vmf" "<src>/maps/<name>.bsp"
```

- 百 MB 级地图约 1–2 秒，产物就是完整 brushwork。
- 它会在 BSP 旁边写一份同名 `.log`，里面有 `BSP version` 和识别出的游戏名，用来确认版本是否被正确识别。
- 不要拿网格导出工具代替这一步：那些工具产出的是三角网格，而 VMF 只接受凸实体（brush）。要得到可编辑的地图几何，必须从 BSP 反编译出真正的 brush。

## 第二步：补顶层块

BSPSource 输出的 VMF **直接从 `world` 开头**，缺少编译器要求的顶层键，会报：

```
CVMFtoVMAP: Missing a required top-level key.
Failed to construct map document from provided vmf data.
```

在文件开头补上标准三块即可（`mapversion` 填地图自己的版本号）：

```
versioninfo
{
	"editorversion" "400"
	"editorbuild" "8075"
	"mapversion" "1"
	"formatversion" "100"
	"prefab" "0"
}
visgroups
{
}
viewsettings
{
	"bSnapToGrid" "1"
	"bShowGrid" "1"
	"nGridSpacing" "64"
	"bShow3DGrid" "0"
}
```

## 第三步：导入

```powershell
'y' | & "<source2-install>/game/bin/win64/source1import.exe" `
  -src1gameinfodir "<s1-game-mod>" `
  -src1contentdir "<s1-content>" `
  -game <name> -s2addon <name> `
  -skipdeps "maps/<name>.vmf"
```

三条硬性要求：

1. **输入只能是 `.vmf`。** `.bsp` 按后缀被拉黑，放在哪个目录都一样：

   ```
   Removing blacklisted file from import
   *** Found no files matching specifications
   ```

2. **`-src1contentdir` 必须是一棵独立于 game 目录的 content 树。** 把两个参数指向同一个目录（有些 mod 没有独立 content 树）会让输出路径算成空字符串：

   ```
   Failed to write map document to specified file: 
   ```

   建一棵只放源文件的树（例如把 VMF 放到 `<s1-content>/maps/<name>.vmf`）即可。

3. **VMF 必须位于那棵 content 树的 `maps/` 下**，文件规范的相对路径基准就是这棵树。若同一文件在 game 目录里也有一份，会被优先匹配到 game 那份，同样导致上面的空路径错误——别留重复副本。

工具实际认识、但没写进帮助文本的地图开关：`-usebsp`、`-usebsp_onlybsp`、`-usebsp_onlyvmf`、`-usebsp_nomergeinstances`、`-skipdeps`。`-skipdeps` 只转地图本身，依赖另走材质与模型入口。

## 产出

```
maps/<name>.vmap                                      主文件
maps/<name>_refs.txt                                  依赖清单（importfilelist 格式）
maps/prefabs/<name>/<name>_environment_prefab.vmap     几何（brushwork，通常最大）
maps/prefabs/<name>/<name>_gameplay.vmap               实体与逻辑
maps/prefabs/<name>/<name>_audio_prefab.vmap
maps/prefabs/<name>/<name>_lighting_prefab.vmap
maps/prefabs/<name>/<name>_cubemaps_prefab.vmap
maps/prefabs/<name>/<name>_nav_prefab.vmap
```

主 `.vmap` 的文件头是 `<!-- dmx encoding keyvalues2 ... format vmap ... -->`，体积很小，真正的内容在 prefab 里——看到主文件只有几十 KB 不代表失败。

`_refs.txt` 与模型入口的 `_refs.txt` 是同一个机制：`materials/...vmt` 走材质入口批量导入，`models/...mdl` 走模型入口（`cs_mdl_import` 加材质重映射）。

## 补依赖

用 `-skipdeps` 导入时只产出地图本身，依赖要从 `_refs.txt` 分流：

```
importfilelist
{
	"file"	"materials/...vmt"
	"file"	"models/...mdl"
	"file"	"postprocess/...vpost"
}
```

- **材质**：抽出 `.vmt` 条目一次性喂给 `source1import`。天空盒材质通常转不过去（S2 的天空盒要在地图里单独配置），这不算失败。
- 解析材质路径时要把**所有挂载的 VPK** 纳入索引：模组通常还会挂载基础游戏的包，地图与模型都可能引用那里的材质。
- `materials/tools/*`（nodraw、clip、trigger 之类）由引擎自带，不必导入；它们在依赖清单里"缺失"是正常的。
- **模型**：抽出 `.mdl` 条目，走模型入口的批量流程（解包 → 逐个转换 → 补材质 → 注入 remap）。一个中等规模的关卡实测约四百个模型。
- 漏掉的材质在 Hammer 里表现为粉黑格或错误引用。先确认 `_refs.txt` 是否已全部导入，再怀疑转换本身。

体量参考：一个中等规模关卡补齐依赖后，content 侧约 2 GB 量级（数百个 `.vmat`、上千张贴图与网格），构建时间要按这个量级预期。

## 修实体类名

导入把实体放进 prefab（几何相关的在 `environment_prefab`、逻辑实体在 `gameplay_prefab`），但**类名保留源引擎的写法**。目标引擎里不存在的类会在 Hammer 里显示成未知实体，需要改名。

已确认的一例：

| 源类名 | 目标类名 | 说明 |
| --- | --- | --- |
| `light_omni` | `light_omni2` | 目标引擎只有 `light_omni2`，没有 `light_omni` |
| `light_spot` | `light_omni2` | 目标引擎同样没有 `light_spot`；聚光由灯自身的角度参数控制，实体属性里已带锥角数据 |
| `func_breakable_surf` | `func_breakable` | 目标引擎没有 surf 变体，改到通用可破坏 brush |
| `prop_flare` | `prop_dynamic` | 目标引擎没有该道具类，改到通用动态道具以保留模型 |

**没有对应类的要删掉，不要硬套。** 目标引擎根本不存在这些机制，改名只会造出一个语义错误的新实体：

| 源类名 | 处理 | 原因 |
| --- | --- | --- |
| `info_node_link` | 删除 | 目标引擎的导航是数据驱动的，没有导航点连线实体 |
| `func_areaportal` | 删除 | 目标引擎的遮挡剔除机制不同，没有 areaportal |
| `env_lensflare`、`shadow_control` | 删除 | 无对应实体 |
| `npc_*`（`npc_headcrab`、`npc_barnacle`、`npc_human_scientist` 等） | 删除 | 目标引擎没有这些 NPC |
| `item_*`（`item_weapon_*`、`item_ammo_*`、`item_battery`、`item_healthkit`、`item_healthcharger`、`item_suit`） | 删除 | 目标引擎没有这类拾取物 |
| 游戏自制的实体（如自定义传送门、分配器） | 删除 | 只有源游戏才有对应代码 |

**先自查一遍类名**，不要等 Hammer 报错：

1. 从 prefab 里取出所有 `"classname" "string" "<类名>"`。注意 prefab 通常是**二进制 DMX**，用可读串提取即可。
2. 与目标引擎的 FGD 类清单比对。取值文件是 `game/core/*.fgd`、`game/csgo/*.fgd`、`game/csgo_core/*.fgd` 这类；**要排除导入工具自带的 FGD**（例如 `game/csgo/import_scripts/` 下的那一套），那是源引擎的 FGD，会把 `light_omni` 这类本该改名的类判成"存在"。
3. 匹配类名时注意 FGD 允许**跨行声明**（`= prop_static` 之后换行才是 `:`），只匹配同一行的写法会漏掉大量类。

**改法是转文本再转回来**，别直接编辑二进制：

```powershell
# 二进制 -> 文本
dmxconvert -i <prefab> -ie binary -o <prefab_text> -oe keyvalues2
# 改完再转回去（先备份原文件）
dmxconvert -i <prefab_text> -ie keyvalues2 -o <prefab> -oe binary
```

替换时只改 classname 的值（KV3 里是 `"classname" "string" "light_omni"` 三段），别误伤同名的资源路径。改完用同一套方法复查一遍类名归零。

**删除实体时要连同元素的类型名一起去掉。** DMX 的数组元素写成 `"<类型名>" { ... }`——类型名是紧跟在花括号前面的一行独立字符串。只删 `{...}` 会留下孤立的类型名，`dmxconvert` 转回二进制时会报：

```
<文件>(行号) : Expecting '{', didn't find it!
```

所以删除范围要覆盖「类型名 + 块 + 尾随逗号」。删完先转回二进制验证一次，再从生成的二进制里重新抽一遍类名做终检。

顺带一提：`dmxconvert` 解析失败时会往**当前工作目录**写一份 `dmxconvert_*.mdmp` 崩溃转储，那是失败产物，不是资产。

## 验收与边界

- 在 Hammer 里打开主 `.vmap`：brushwork 在 environment prefab，实体在 gameplay prefab。
- 标准实体会映射（灯光、`prop_static`、各类 trigger 等）；**源游戏的自定义实体在目标引擎里不存在**，会以未知实体的形式留下，需要重做或删除。
- 光照不会跟随：S1 的烘焙 lightmap 不参与转换，需要在新地图里重新打光。
- 天空盒、3D 天空盒、导航网格都要在新地图里重新配置。
- 缺失的模型与材质在 Hammer 里表现为错误引用或粉黑格，属依赖未导入，按 `_refs.txt` 补齐即可。
