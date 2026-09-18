# 模型入口

## 导入

```powershell
& "<source2-install>/game/bin/win64/cs_mdl_import.exe" -game <name> `
  -i "<path-to-loose-source-models>" `
  -o "<source2-install>/content/<addon-root>" `
  -lods -l <batch>.txt
```

要点：

- `source1import.exe` 不含 MDL 导入器，模型必须走这个工具。
- `-i` 是输入根、`-o` 是输出的 content 根，两者是替换关系。把源模型落成散文件再逐条喂入最稳（实测可行），不要指望它解析 VPK 虚拟路径。
- 默认只导 LOD0，需要全部 LOD 时加 `-lods`。
- `-skipcommondmxwrite` 跳过动画 DMX 的写出。
- 可以一次喂多条文件规格，适合批量。

## 产出结构

```
models/<dir>/<name>.vmdl
models/<dir>/<name>_refs.txt
models/<dir>/<name>_refs/mesh/<name>_..._lod0.dmx
models/<dir>/<name>_refs/mesh/meshinfo.txt
models/<dir>/<name>_refs/phys/<name>_phy.dmx
```

- `_refs.txt` 是该模型需要的材质清单，本身就是 `importfilelist` 格式，可以直接喂给 `source1import.exe -usefilelist`。
- 网格 DMX 里记录的是**裸材质名**（如 `<name>.vmat`，不带目录），所以模型不会自动找到材质。
- `.phy` 会被转成 `_phy.dmx` 并接进 `PhysicsShapeList`；碰撞不对时先确认源物理是否完整。

## 必做：补材质重映射

导入出的 vmdl 没有 `MaterialGroupList` 节点，需要手工加在 `rootNode.children` 里，把裸名指到真实路径：

```
{
	_class = "MaterialGroupList"
	children = 
	[
		{
			_class = "DefaultMaterialGroup"
			remaps = 
			[
				{
					from = "<name>.vmat"
					to = "materials/<dir>/<name>.vmat"
				},
			]
			use_global_default = false
			global_default_material = ""
		},
	]
},
```

一个模型有多个材质槽就写多条 remap。改完检查花括号配平，kv3 头不要动。

## 外部引用会变成错误项

导入出的 vmdl 会把源模型的断裂件、部分动画写成对外部资产的引用，例如：

```
_class = "BreakPieceExternal"
model = "models/<dir>/<name>_gib_01.vmdl"
```

这些目标模型没导入时，模型编辑器里会整片报错。两种处理：

- 需要这个行为：把外部模型一并导入，并各自补上材质重映射（它们的 `_refs.txt` 会列出所需材质，往往只依赖主模型的材质）。
- 不需要：把整段节点删掉。

## 批量转换

单个或小批量模型按上面的命令跑即可；上百个模型时，下面几条是实测必须处理的。

**解包**

- `vpk.exe x` **不会自动创建子目录**：路径里的目录不存在时会报 `Unable to create '...'` 然后静默跳过，最后你得到 0 个文件。先按文件清单建好目录树再解。
- 一次传几百个文件规格会超出命令行长度（`文件名或扩展名太长`）。按每批 60–80 个文件分块调用。
- 源模型可能分散在**多个挂载的包**里：模组自己的包、以及它挂载的基础游戏包。只查模组自己的包会漏掉一批（实测 140 个引用里有 11 个来自基础游戏包），这些漏掉的模型就是地图里的 error 模型。

**转换**

- `cs_mdl_import -l <清单>` 在几百个模型的清单上会**中途崩溃**（`0xC0000005` 访问冲突）且不回退，崩溃点之后的模型全部没转。
- 稳妥做法是**逐个模型调用**：先跳过目标 `.vmdl` 已存在的，再逐个跑，把失败记进清单后继续。实测吞吐约每秒 2 个模型。
- 少数模型即使单独跑也稳定崩溃，这不是参数问题，只能回退到 DCC 路线（把模型反编译后经 Blender 导出 DMX/FBX）。

**材质清单**

- 每个模型的 `<name>_refs.txt` 列出它需要的材质。其中的裸名字（`foo.vmt`）要自己解析成完整路径：先用 `vpk.exe l` 建一张「基名 → 完整路径」索引再查表。
- 批量导入时跳过目标 `.vmat` 已存在的条目，能省掉大量重复工作。

## 重映射自动化

模型上到几百个时，逐个人工补 `MaterialGroupList` 不现实，按下面的流程脚本化：

1. 从每个模型的网格 DMX 里提取材质名。**注意它通常是路径而不是基名**：形如 `models\props_office\computer_devices01.vmat`，也就是 S1 材质路径去掉开头的 `materials/`。少数模型只存基名。
2. 用插件里**已经导入的 `.vmat` 文件**建一张「基名 → 相对路径」索引，解析出每条 remap 的 `to` 目标。
3. 给每个模型注入 `MaterialGroupList`：**`from` 必须原样匹配网格里的名字**（统一成正斜杠写法，例如 `models/props_office/computer_devices01.vmat`），`to` 才是解析出来的材质路径。

三个必须记住的点：

- **`use_global_default` 必须留 `false`。** 它对应模型编辑器里的 "Globally Replace All Materials In Model"：勾上之后所有材质槽都塌缩到那一个材质，逐条材质 remap 全部失效，模型看上去就是"材质不对"。它既不能当缺失材质的兜底（缺 remap 时编译仍会整体失败并报 `referencing missing material '<名称>.vmat'`），也不该在已有逐条 remap 时开启。生成 vmdl 时写 `use_global_default = false`、`global_default_material = ""`。
- **用基名当 `from` 会静默失效，并被全局替换掩盖。** 网格里存的是路径、remap 的 `from` 写的是基名时，这条 remap 根本不匹配；如果这时 `use_global_default` 还是 `true`，所有材质都会落到那个全局材质上，表面上"能编译、有材质"，直到关掉全局替换才集中暴露成 `referencing missing material`。所以校验必须按**精确名字**比对（两边都统一成正斜杠再比），按基名比对会给出假绿。
- **带 skin 的模型，导入器会自己生成 `MaterialGroupList`。** 里面是按皮肤分组的 `MaterialGroup`（`name = "1"`、`"2"`…），`remaps` 全是空的（`remaps = [  ]`），而且**只有命名组、没有默认组**。命名组只在显式选择皮肤时生效，而道具默认不带皮肤，材质就解析不到——编译报 `Mesh '...' referencing missing material ...` 并整体失败。处理分两步：(1) 把命名组的空 `remaps` 填上；(2) 再补一个 `DefaultMaterialGroup`（remaps 取命名组的并集，同名 `from` 以第一个组为准），保留命名组以维持皮肤。判断"已有 remap"要看有没有真实的 `to = "materials/..."`，不能只看有没有 `MaterialGroupList` 这个字符串。
- **材质解析范围要覆盖所有挂载的 VPK。** 模组通常还会挂载它的基础游戏包，模型可能引用基础游戏的材质；只索引模组自己的包会漏。另外**网格 DMX 里引用的材质可能比模型 `_refs.txt` 多**，因此收尾需要一轮"收集全部未解析名 → 解析 → 导入 → 再修正"。
- **收尾做一次机械校验。** 对每个模型：网格 DMX 里的材质名集合必须是 remap `from` 集合的子集，并且每条 remap 的 `to` 都指向存在的文件。这两项任一不满足编译就会失败；只看"有没有 remap"会漏掉这两种情况。

收尾时把解析不到的先指向一个已知材质，等补导入之后做**第二遍修正**：把 `to` 仍指向那个兜底材质、而 `from` 现在已经能解析的条目改掉。源数据里本来就有的坏引用（不存在的名字）保持兜底即可。

按这套流程跑下来，材质解析可以做到 100%，剩下的只会是源数据里本就不存在的名字。

## 编译期的结构问题

材质之外，导入产物还有几类会让编译直接失败的结构问题，都需要在收尾时一并处理：

- **bodygroup 的 choice 没有名字。** 导入器生成的 `BodyGroupList` 里，`BodyGroupChoice` 只有 `meshes` 字段、**没有 `name`**，编译器报：

  ```
  Invalid empty body group choice name in bodygroup '<名称>'. Non-empty choice names are required.
  ```

  给每个 choice 补一个非空名字即可（按组内序号 `choice_0`、`choice_1`…）。**空 choice（`meshes = [  ]`，对应源模型的 blank）同样需要名字**，不能省略。

这类错误在错误面板里往往只显示模型名、材质名或节点名的一部分。定位时不要靠截图，按"挨个模型扫一遍结构"的方式校验更可靠：确认每个模型都有材质组、每条 remap 的 `to` 存在、每个 bodygroup choice 都有名字。

## 其他注意

- 角色类模型还要查骨骼、附件点（`AttachmentList`）、hitbox 和动画序列；面部形变（flex）在目标引擎里没有一一对应，通常要重建或放弃。
- 大模型集合按地图实际需要转，不要全量导入——全量导入后失败项和体积都难管理。
- 网格输入格式的选择：目标引擎的模型文档接受 DMX / FBX / OBJ。若手上只有另一种中间格式（例如旧式 SMD 或 QC 反编译产物），要么换一条能直接产出上述格式的工具链，要么先进 DCC 转一道。
