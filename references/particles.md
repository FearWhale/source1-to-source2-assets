# 粒子入口

## 必须先落散文件

粒子导入器内部调用 `dmxconvert.exe` 解析 PCF，而它会把虚拟路径原样传给 `dmxconvert`，后者只认真实文件，于是报：

```
No files match specification "vpk:...<name>.pcf"!
```

正确流程：先用 `vpk.exe x` 把 `.pcf` 解成散文件，再用**绝对路径**调用导入：

```powershell
'y' | & "<source2-install>/game/bin/win64/source1import.exe" `
  -src1gameinfodir "<s1-game>/<name>" -game <name> -s2addon <name> `
  -particleshaders "<path>/particles/<name>.pcf"
```

只解包、仍按相对路径喂入的话，路径会被解析回 VPK，必须给绝对路径。

常用开关：

| 开关 | 作用 |
| --- | --- |
| `-particleshaders` | 列出该文件用到的源着色器 |
| `-rebuildparticleremaptable` | 重建粒子名映射表 |
| `-use_particle_manifest` | 只转清单内的 pcf |
| `-particle_disable_diffuse` | 关闭漫反射光照 |
| `-particle_allow_depth_blend` | 尊重源材质的深度混合开关 |

## 产出

一个 pcf 里的多个粒子系统会各自产出一个 vpcf：

```
particles/<name>/<system>.vpcf
```

导入日志逐条给出每个系统用到的函数及其导入方式（`simple importer` 或自定义导入器），这是判断覆盖面的直接依据。发射器、初始化器、操作器、力场、约束分别对应目标引擎的 `m_Emitters` / `m_Initializers` / `m_Operators` / `m_ForceGenerators` / `m_Constraints`。

## 精灵贴图是三件套

粒子渲染器引用的是**贴图**而不是材质，且贴图需要三个文件才能编译：

| 文件 | 作用 |
| --- | --- |
| `<name>.tga` | 图像本体。精灵图必须保留 alpha（32 位） |
| `<name>.mks` | 分帧描述（有分帧时才有） |
| `<name>.vtex` | 编译入口。**缺它完全不编译，渲染器显示紫黑格** |

`<name>.mks` 形如：

```
packmode flat
sequence 0
frame <name>.tga(0,0,63,63) <name>.tga(0,0,63,63) <name>.tga(0,0,63,63) <name>.tga(0,0,63,63) 1.000000
```

每个 `sequence` 一段，矩形是 `(x0,y0,x1,y1)`，行末是帧速率。整张图就是一帧时不要写 mks，直接让 vtex 指向 tga。

`<name>.vtex` 模板：

```
<!-- dmx encoding keyvalues2_noids 1 format vtex 1 -->
"CDmeVtex"
{
    "m_inputTextureArray" "element_array"
    [
        "CDmeInputTexture"
        {
            "m_name" "string" "InputTexture0"
            "m_fileName" "string" "materials/<dir>/<name>.tga"
            "m_colorSpace" "string" "srgb"
            "m_typeString" "string" "2D"
            "m_imageProcessorArray" "element_array"
            [
                "CDmeImageProcessor"
                {
                    "m_algorithm" "string" "None"
                    "m_stringArg" "string" ""
                    "m_vFloat4Arg" "vector4" "0 0 0 0"
                }
            ]
        }
    ]
    "m_outputTypeString" "string" "2D"
    "m_outputFormat" "string" "DXT5"
    "m_outputClearColor" "vector4" "0 0 0 0"
    "m_nOutputMinDimension" "int" "0"
    "m_nOutputMaxDimension" "int" "0"
    "m_textureOutputChannelArray" "element_array"
    [
        "CDmeTextureOutputChannel"
        {
            "m_inputTextureArray" "string_array" [ "InputTexture0" ]
            "m_srcChannels" "string" "rgba"
            "m_dstChannels" "string" "rgba"
            "m_mipAlgorithm" "CDmeImageProcessor"
            {
                "m_algorithm" "string" "Box"
                "m_stringArg" "string" ""
                "m_vFloat4Arg" "vector4" "0 0 0 0"
            }
            "m_outputColorSpace" "string" "srgb"
        }
    ]
    "m_vClamp" "vector3" "0 0 0"
    "m_bNoLod" "bool" "0"
}
```

`m_fileName` 指向 `.tga`（无分帧）或 `.mks`（有分帧）。源贴图是带 alpha 的 DXT5 时，用 `vtf2tga` 转出 32 位 TGA 再配 vtex；转完检查 TGA 头第 17 字节是 `0x20`（32bpp，带 alpha）。

## 空引用与紫黑格

源 pcf 引用了不存在或已改名的材质时，导入**不会失败**，而是产出空引用并禁用渲染器：

```
_class = "C_OP_RenderSprites"
m_bDisableOperator = true
m_vecTexturesInput = [ { m_hTexture = resource:"" } ]
```

修复路径：

1. 在导入日志里找 `Failed to find VMT for render op: 'materials/<dir>/<name>.vmt'`，确认缺哪个材质。
2. 回源资产找同一张图。常见情况是材质被改名而贴图仍在，此时同名 `<name>.vtf` 还能取到。
3. `vpk.exe x` 解出贴图，`vtf2tga` 转成 32 位 TGA，放进插件的 `materials/<dir>/`。
4. 补 `.vtex`；有分帧再补 `.mks`。
5. 回到 vpcf：删掉 `m_bDisableOperator`，把 `m_hTexture` 指到转好的贴图。

## 渲染器常用字段

字段位置照同类目标 vpcf 的真实写法对齐：

| 字段 | 作用 |
| --- | --- |
| `m_nOutputBlendMode` | 混合模式，源材质是加色时用 `PARTICLE_OUTPUT_BLEND_MODE_ADD` |
| `m_flAnimationRate` | 帧播放速率 |
| `m_nAnimationType` | 如 `ANIMATION_TYPE_FIT_LIFETIME`、`ANIMATION_TYPE_MANUAL_FRAMES` |
| `m_nOrientationType` | 朝向方式 |
| `m_flSelfIllumAmount` | 自发光强度 |

改完在粒子编辑器里预览：贴图是否解析、加色是否过曝、帧动画是否正常。源侧滚动类效果（贴图 UV 动画）在目标引擎里没有直接等价物，需要靠分帧序列或渲染器动画近似。
