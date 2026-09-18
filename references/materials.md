# 材质入口

## 导入

```powershell
'y' | & "<source2-install>/game/bin/win64/source1import.exe" `
  -src1gameinfodir "<s1-game>/<name>" -game <name> -s2addon <name> `
  -usefilelist <batch>.txt -logwarnings
```

材质与贴图能从 VPK 直接读，不需要先解包。用 `vpk.exe l <dir>.vpk` 出清单，筛出 `.vmt` 后按目录分批写成 `-usefilelist` 的 kv 文件；一批几十到几百个，便于定位失败项。

## 产出

每个 `<name>.vmt` 产出一个 `.vmat` 加若干源贴图：

| 产出 | 说明 |
| --- | --- |
| `<name>.vmat` | kv3，`Layer0.Shader` 由源着色器决定 |
| `<name>_d_color.tga` | 基础色 |
| `<name>_n_normal.tga` | 法线 |
| `<name>_rough.tga` | 粗糙度，由 `$phongexponent` 一类参数推算 |
| `<name>_d_selfillum.tga` | 自发光遮罩 |
| `<name>_n_normal.txt` | 贴图侧车设置，**必须与 tga 同目录保留** |

侧车里常见 `"legacy_source1_inverted_normal" "1"`，表示源法线需要翻转绿通道，编译时读它。删掉侧车会导致法线方向错误。

## 参数映射

`VertexLitGeneric` 一类的源着色器会落到对应的 legacy 着色器上（具体名字看产出 `.vmat` 的 `Layer0.Shader`），其余多数走通用 PBR 着色器。原始参数会保留在 `.vmat` 的 `legacy_import` 段里备查。常见对应：

| 源参数 | 目标 |
| --- | --- |
| `$basetexture` | `TextureColor` |
| `$bumpmap` | `TextureNormal` |
| `$phongexponent` / `$phongboost` | `TextureRoughness` |
| `$envmapmask` / `$normalmapalphaenvmapmask` | `TextureMetalness`，或折算成 `g_flCubeMapScalar` |
| `$selfillum` | `TextureSelfIllumMask` + `F_SELF_ILLUM` |
| `$envmap` | 立方图反射开关与强度 |
| `$translucent` / `$alphatest` / `$nocull` / `$additive` | 混合模式、Alpha 测试、双面 |

## 已知信息损失

转换是有损的，先认清这几类再决定要不要手补：

1. **法线 alpha 里的反射遮罩会丢。** 源材质用 `$normalmapalphaenvmapmask` 时，导出的法线是 24 位 TGA，alpha 不保留，遮罩被折算成均匀的 `g_flCubeMapScalar`，整块反射会变得一样亮。要还原得自己补一张遮罩贴图并接回材质。
2. **带代理的动画自发光不转。** `$emissiveBlend*` 配合 `Sine`、滚动之类的代理只留在 `legacy_import`，目标材质只有静态自发光。
3. **`$detail` 混合、四向混合、自定义着色器、着色器专用的 `.raw` 贴图没有对应实现**，会落进警告列表。这类只能按目标引擎的通用 PBR 着色器重写。
4. 源材质引用了不存在或已被改名的贴图时，导入不报错，但材质指向空资源，表现为紫黑格——靠 `-logwarnings` 的结果筛。
5. `materials/dev/` 整个目录被列进导入黑名单，无法转换，会报 `Removing blacklisted file from import`。这类材质只能手写或改指其他材质。

## 手补材质

按上面的损失清单分类处理：

- 只缺遮罩或自发光：补贴图并在 `.vmat` 里接上对应槽位。
- 着色器不支持：按通用 PBR 着色器重写，`legacy_import` 段保留原始参数作对照。
- 源贴图缺失：先从 `vpk.exe x` 解出来单独转换，再回填路径。

手写 `.vmat` 时遵循目标引擎的 kv3 结构，别改动文件头的 encoding / format 版本标识。
