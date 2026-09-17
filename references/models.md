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

## 其他注意

- 角色类模型还要查骨骼、附件点（`AttachmentList`）、hitbox 和动画序列；面部形变（flex）在目标引擎里没有一一对应，通常要重建或放弃。
- 大模型集合按地图实际需要转，不要全量导入——全量导入后失败项和体积都难管理。
- 网格输入格式的选择：目标引擎的模型文档接受 DMX / FBX / OBJ。若手上只有另一种中间格式（例如旧式 SMD 或 QC 反编译产物），要么换一条能直接产出上述格式的工具链，要么先进 DCC 转一道。
