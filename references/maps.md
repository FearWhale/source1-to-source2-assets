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

## 地形（位移面）

**症状**：导入后在 Hammer 里大片地形不见了，而导入日志里反复出现：

```
Found a displacement missing a needed subkey.
```

这条警告每条对应一个位移面，出现次数等于位移面总数；同时 environment prefab 会明显偏小。

**原因**：反编译出的 VMF 在源数据没有 offset 信息时会省略 `offsets` / `offset_normals` 两个子块（源引擎里它们是可选的），而导入器要求这两个键——**缺了就直接丢弃该位移面**，只留一条警告，不报错。所以整片地形会静默消失。

**修复**：给每个 `dispinfo` 块补上全零的 `offsets` 与 `offset_normals`。行数与列数都取决于该位移的 `power`：网格边长是 `2^power + 1`，每格写一个 `0 0 0`。

```
offset_normals
{
	"row0" "0 0 0 0 0 0 ..."
	...（共 2^power + 1 行）
}
```

**验收**：重新导入后 `missing a needed subkey` 应为 0，且 environment prefab 明显变大（实测一张 Xen 关卡从 4.1 MB 变成 8.5 MB）。

注意这张坑很容易漏掉：**只有含位移面的地图才会触发**。设施关卡那类纯 brush 地图位移面数为 0，导入一切正常；第一次遇到地形关卡才会暴露。所以拿到一张新地图先数一下 VMF 里的 `dispinfo` 数量，非 0 就走一遍上面的检查。

**修完地形一定要重新看一遍材质清单。** `_refs.txt` 是**按实际导入的内容**生成的依赖表：地形被丢弃时，地形用的那些材质也不会出现在清单里。所以修完位移面重新导入后，`_refs.txt` 会变长（实测一张 Xen 关卡从 28 项变成 31 项，多出来的正是三张地形 blend 材质），需要把新增的材质再导入一遍。

表现上这两件事很容易串成一条误判链：地形没了 → 地形材质没进清单 → 表面显示为缺失材质（爆红）→ 看起来像"材质导入失败"，其实根因一直在位移面。**先确认几何完整，再排查材质。**

## 修实体类名

导入把实体放进 prefab（几何相关的在 `environment_prefab`、逻辑实体在 `gameplay_prefab`），但**类名保留源引擎的写法**。目标引擎里不存在的类会在 Hammer 里显示成未知实体，需要改名。

已确认的一例：

| 源类名 | 目标类名 | 说明 |
| --- | --- | --- |
| `light_omni` | `light_omni2` | 目标引擎只有 `light_omni2`，没有 `light_omni` |
| `light_spot` | `light_barn` | 目标引擎同样没有 `light_spot`。**聚光灯对应的是 `light_barn`**，它才是带光圈/挡光板参数的定向灯（`size_params`、`shape`、`soft_x`/`soft_y`）；`light_omni2` 虽然也有 `outer_angle`/`inner_angle`，但那是"半球形光"的参数，拿来替代聚光灯会丢掉锥形 |
| `func_breakable_surf` | `func_breakable` | 目标引擎没有 surf 变体，改到通用可破坏 brush |
| `prop_flare` | `prop_dynamic` | 目标引擎没有该道具类，改到通用动态道具以保留模型 |
| `func_water_analog` | `func_water` | 目标引擎只有 `func_water` |
| `misc_dead_hev`、`prop_hev_charger` | `prop_dynamic` | 本质是模型道具，改到通用动态道具以保留模型 |

判断某盏灯原本属于哪一类，不要只看类名：**带聚光锥角的才是原来的 spot**（导入产物里这类实体有 `outerconeangle`/`innerconeangle` 之类的锥角属性），其余是点光源。目标引擎的灯光类里没有 `light_spot`，改名前先按属性区分，避免把聚光灯错并到点光源上。

**还有一类是"存在但应当改用"的。** 这些类在目标引擎的 FGD 里确实存在，所以单纯的存在性检查不会报出来，但实践中应该换掉：

| 源类名 | 目标类名 | 说明 |
| --- | --- | --- |
| `env_cubemap` | `env_cubemap_box` | 两者都在目标引擎的 FGD 里，但应使用带盒投影的 `env_cubemap_box`。这类映射靠存在性检查发现不了，需要按经验或引擎文档补 |
| `path_particle_rope` | `path_particle_rope_clientside` | 游戏 FGD 里写明了原因：服务端绳索在每次有玩家加入时都会卡顿，要求改用 clientside 版本 |

### @exclude：存在但已被禁用

游戏级 FGD 会用 `@exclude <类名>` 把继承来的类移除，这有两种结果，**必须分开判断**：

| 情况 | 结果 | 例子 |
| --- | --- | --- |
| `@exclude` 之后**又在该 FGD 里重新定义** | 仍然可用（通常是"去掉过时参数后重定义"） | `env_sky` |
| `@exclude` 之后**没有重新定义** | 该游戏里不可用，等同于不存在 | `path_particle_rope`、`info_lighting`、`color_correction`、`env_tonemap_controller`、`fog_volume`、`light_dynamic` 等 |

只看"类名是否在 FGD 里定义过"会漏掉第二种——那些类在基础 FGD 里有定义，存在性检查会判为"有"，但在目标游戏里其实用不了。所以类名核对要做**两遍**：一遍查"有没有定义"，一遍查"有没有被排除且没有重新定义"。

实测两张关卡里踩到的替换与删除：

| 源类名 | 处理 | 说明 |
| --- | --- | --- |
| `path_particle_rope` | 改名 `path_particle_rope_clientside` | 官方指定替代 |
| `info_lighting` | 删除 | 被排除且无替代；引用它的道具会退回按自身原点取光 |
| `color_correction`、`env_tonemap_controller` | 删除 | 被排除且无替代；目标引擎的色彩分级与色调映射走 `post_processing_volume` |
| `fog_volume` | 删除 | 被排除且无替代；雾用 `env_fog_controller` |

**没有对应类的要删掉，不要硬套。** 目标引擎根本不存在这些机制，改名只会造出一个语义错误的新实体：

| 源类名 | 处理 | 原因 |
| --- | --- | --- |
| `info_node`、`info_node_hint`、`info_node_air`、`info_node_air_hint`、`info_node_climb`、`info_node_link` | 删除 | 目标引擎的导航走导航网格，这一整套 AI 节点图都是遗留物（它们定义在 AI/NPC 基础 FGD 里，而目标引擎没有对应的 NPC 体系）。这些实体通常数量很大（实测一张关卡 201 个 + 70 个），留着只会拖慢 Hammer 并淹没有用实体 |
| `func_areaportal`、`func_occluder`、`func_viscluster` | 删除 | 目标引擎的可见性/遮挡系统不同，这些优化实体没有对应物 |
| `npc_*`（头蟹、藤壶、科学家、猎眼、触手、Xen 炮塔等） | 删除 | 目标引擎没有这些 NPC |
| `item_*`（`item_weapon_*`、`item_ammo_*`、`item_grenade_*`、`item_battery`、`item_healthkit`、`item_healthcharger`、`item_suit`、`item_longjump`） | 删除 | 目标引擎没有这类拾取物 |
| `ai_goal_*`、`aiscripted_schedule`、`assault_assaultpoint`、`assault_rallypoint` | 删除 | AI 调度系统不存在 |
| `env_screeneffect`、`env_screenoverlay`、`env_zoom`、`point_viewcontrol`、`point_spotlight`、`point_tesla` | 删除 | 演出/视觉效果类没有对应实体；亮度和视角效果要在新地图里用光照与后处理重做 |
| `env_lensflare`、`shadow_control`、`env_gravity`、`env_cascade_light`、`func_dustmotes`、`trigger_playermovement`、`point_weaponstrip`、`player_speedmod`、`player_loadsaved`、`logic_achievement` | 删除 | 无对应实体 |
| 游戏自制的实体（自定义传送门、分配器、脚本控制器等，例如 `newxog_*`） | 删除 | 只有源游戏才有对应代码 |

删除后再做一次类名核对；两张实测地图（一张设施关卡、一张 Xen 关卡）改完都是 0 缺失。这个映射表是按实测地图逐步补出来的，遇到表里没有的类，先按"目标引擎是否有同功能实体"判断，拿不准就列出来问用户——错误地"硬套"一个形状相似的类（比如把聚光灯改到点光源）比直接删掉更难发现。

**注意别把导入器自己生成的辅助实体当成遗留物删掉。** prefab 里的实体不一定都来自源地图：导入器也会生成目标引擎原生的辅助实体。典型例子是 `path_node_generic`——它名字像导航节点，实际是**编辑器专用的路径挂点**（FGD 里标着 `editor_only = true`，作为 `path_track` / `path_simple` 这类路径实体的 `path_node_class`），删掉路径就断了。

区分方法很简单：**拿类名去源 VMF 里查**。源 VMF 里没有、prefab 里却有的，就是导入器生成的，别动；源 VMF 里有、prefab 里也有的，才是需要判断去留的源实体。

**类名匹配要区分大小写地写正则。** 自制类里常有大写（例如 `newLight_Point`），用 `[a-z_0-9]+` 之类的模式去抓类名会**静默漏掉**它们——删除逻辑漏掉它们、类名核对也会漏掉它们，于是给出"全部合规"的假结论。抽类名一律用 `[A-Za-z_0-9]+`。

## 只保留影响场景的实体

如果目标是拿导入结果当"可重建的场景底子"而不是完整复刻玩法，可以按**保留白名单**清理：能渲染或能被看到听到的留下，只驱动行为的删掉。实测两张关卡按这个口径清完，实体种类从 56/65 降到 34/40。

**保留**

| 类别 | 类名 |
| --- | --- |
| 灯光 | `light_omni2`、`light_barn`、`light_rect`、`light_environment`、`light_dynamic`、`info_lighting`，以及自制灯光映射后的结果 |
| 天空与反射 | `env_sky`、`sky_camera`、`env_cubemap_box` |
| 雾与后处理 | `env_fog_controller`、`fog_volume`、`post_processing_volume`、`env_tonemap_controller`、`color_correction` |
| 模型 | `prop_static`、`prop_dynamic`、`prop_dynamic_override`、`prop_physics`、`prop_physics_override`、`prop_ragdoll`、`prop_door_rotating` |
| 可见几何与机关 | `func_brush`、`func_breakable`、`func_water`、`func_door`、`func_door_rotating`、`func_rotating`、`func_tracktrain`、`func_useableladder`、`func_clip_vphysics`、`func_button`、`momentary_rot_button` |
| 特效 | `info_particle_system`、`env_particle_glow`、`env_explosion`、`env_physexplosion`、`env_spark`、`path_particle_rope`、`gibshooter`、`env_fade`、`env_wind`、`water_lod_control` |
| 环境音 | `ambient_generic`、`env_soundscape`、`env_soundscape_proxy` |
| 出生点与移动路径 | `info_player_start`，以及与 `func_tracktrain` 配套的 `path_track` / `path_node_generic` |

**删除**：`trigger_*`、`logic_*`、`math_*`、`filter_*`、`ai_*`、`npc_*`、`assault_*`、`point_*`（除已归类的特效）、`scripted_sequence`、`info_target`、`info_landmark`、`info_hint`、`info_ladder_dismount`、`info_teleport_destination`、`env_shake`、`env_hudhint`、`env_message` 等纯驱动类。

两个容易误删的：`path_node_generic` 看着像导航节点，其实是移动平台路径的挂点；`worldspawn` 是地图根元素，每个 prefab 都有一个，必须保留。清理前后都用类名核对一遍，确认没有把这两类顺手删掉。

## 自制灯光类

游戏经常带一套自制的动态灯光实体，它们不在目标引擎的类表里，但语义上就是灯，应该按灯映射而不是删除：

| 源类名 | 目标类名 | 说明 |
| --- | --- | --- |
| `newLight_Point` | `light_omni2` | 点光源 |
| `newLight_Spot` | `light_barn` | 聚光 |
| `newLight_Dir` | `light_environment` | 平行光 |
| `newLights_settings` | 删除 | 按时间改亮度/颜色的灯光脚本控制器，属逻辑，无对应实体 |

自制灯光往往带 god rays、曝光、色调映射这类目标引擎没有的参数，映射后只剩"这里有一盏灯"，动态变化的部分需要重做。

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
