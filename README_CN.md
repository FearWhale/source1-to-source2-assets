# source1-to-source2-assets

一个给 Codex 用的技能：把 Source 1 的**材质、模型、粒子**移植进 Source 2 工坊插件。

它不是"教你怎么写资产"，而是记录这条移植路上那些**不报错但结果不可用**的坑——官方工具链把三类资产拆在不同工具里，每一步都有一批看起来成功、实际打开是空引用的产物。

[English](README.md)

## 为什么需要这个技能

Source 1 → Source 2 的资产移植没有统一入口：

| 类型 | 导入工具 | 最容易踩的坑 |
| --- | --- | --- |
| 材质 | `source1import.exe` | 法线 alpha 里的反射遮罩丢失；带代理的动画自发光不转；自定义着色器只留警告 |
| 模型 | `cs_mdl_import.exe` | 导入产物**没有材质重映射**，不手补就是粉黑格；断裂件引用会整片报错 |
| 粒子 | `source1import.exe` | 不能从 VPK 直接导；精灵贴图必须是 `tga + mks + vtex` 三件套，缺一个就紫黑格 |

这些结论来自一次真实的移植过程，不是从文档抄的：材质那三条是逐张 `.vmat` 核对出来的，模型那条是在模型编辑器里看到整片报错才定位到的，粒子那条是追着导入器内部调用 `dmxconvert` 的报错找出来的。

## 三个入口

技能按资产类型分三条独立入口，各自带一份细节文档，按需读取：

```text
SKILL.md                      路由器：通用前提、工具清单、调用形状、验收标准
references/materials.md       入口一：参数映射表、产出结构、四条已知信息损失
references/models.md          入口二：产出结构、MaterialGroupList 重映射模板、外部引用处理
references/particles.md       入口三：散文件要求、贴图三件套、空引用修复路径、渲染器字段
```

## 覆盖内容

材质的参数映射（`$basetexture` / `$bumpmap` / `$phongexponent` / `$envmapmask` / `$selfillum` 等到目标着色器槽位）、模型导入产出的四类文件及其含义、粒子导入器的函数覆盖判断方式，以及三份可直接套用的模板：

- 模型的 `MaterialGroupList` / `DefaultMaterialGroup` 重映射片段
- 粒子的 `.vtex` 贴图编译入口
- 粒子的 `.mks` 分帧描述

## 安装

本仓库目录本身就是一个技能目录，克隆后放进对应平台的 skills 目录即可。

### Codex

```bash
git clone <repo-url> "$HOME/.codex/skills/source1-to-source2-assets"
```

Windows PowerShell：

```powershell
git clone <repo-url> "$env:USERPROFILE\.codex\skills\source1-to-source2-assets"
```

若设置了 `CODEX_HOME`，放进 `$CODEX_HOME/skills/`。

### 其他平台

把整个目录放到该平台的 skills 目录下：

| 平台 | 目标位置 |
| --- | --- |
| Claude Code | 插件目录或 `skills/` |
| Cursor | 插件目录或 `skills/` |
| 其他 | 对应平台的技能/规则目录 |

### 并入技能合集

如果已经在用技能合集仓库，把本目录整体拷进合集仓库的 `skills/` 下即可，`SKILL.md` 与 `references/` 的相对结构不要改。

## 使用

安装后直接描述任务即可触发，例如：

> 把这些 Source 1 的材质和模型转进我的 Source 2 插件里

> 这个粒子导入后是紫黑格，帮我查一下

> 模型导进来了，但材质在编辑器里不显示

也可以显式调用：`$source1-to-source2-assets`

## 前置条件

- **Source 2 工坊工具**：技能依赖其中的 `source1import.exe`、`cs_mdl_import.exe`、`dmxconvert.exe`、`resourcecompiler.exe`，位于工具的 `game/bin/win64/` 下。
- **源侧工具**：`vpk.exe`（列目录、解包）与 `vtf2tga.exe`（贴图转 TGA），通常在原游戏的 `bin/` 下。
- 导入本身是本地操作，不需要联网。
- **默认不写入 game 目录。** 产生运行期资源的编译/构建步骤只在你明确要求时才执行；原游戏目录同理。

## 命名约定

技能内所有 `<name>` 都代表实际的游戏、插件或资产名，`<source2-install>`、`<s1-game>` 代表对应的安装根目录。文中不绑定任何具体项目，替换成自己的即可。

## 已知限制

这条链路是**有损**的，技能里逐条记了损失点和补救方式，但不保证能一比一还原：

- 材质：遮罩贴图、贴图动画、部分混合模式需要手工重做。
- 模型：面部形变（flex）在目标引擎里没有一一对应，通常要重建或放弃。
- 粒子：源侧的贴图滚动类效果没有直接等价物，只能用分帧或渲染器动画近似。
- 全量转换成本很高，建议按地图实际需要分批转。

## 版权提醒

移植的是他人的游戏资产。自己本地使用与公开发布（尤其是上传到创意工坊或仓库分发）在授权要求上完全不同，发布前请自行确认你是否有权分发这些资产。本技能只描述技术流程，不附带任何游戏资产。
