# source1-to-source2-assets

A Codex skill for porting **materials, models, and particles** from a Source 1 game or mod into a Source 2 workshop addon.

It is not a tutorial on authoring assets. It records the failure modes of this specific migration path: the official toolchain splits the three asset types across different tools, and every step can produce output that reports success yet resolves to an empty reference when opened.

[中文](README_CN.md)

## Why this exists

There is no single entry point for Source 1 → Source 2 asset porting:

| Type | Tool | The trap |
| --- | --- | --- |
| Materials | `source1import.exe` | Reflection masks stored in normal-map alpha are lost; proxy-driven animated self-illumination is dropped; custom shaders degrade to warnings |
| Models | `cs_mdl_import.exe` | The output has **no material remap**, so materials fail to resolve; break-piece references surface as errors |
| Particles | `source1import.exe` | Cannot import straight from a VPK; sprite textures need the `tga + mks + vtex` trio or the renderer shows a missing-texture checkerboard |

These notes come from an actual port, not from documentation: the material losses were found by auditing the generated `.vmat` files, the model issue appeared as a wall of errors in the model editor, and the particle issue was traced through the importer's internal `dmxconvert` invocation.

## Three entry points

The skill splits by asset type into three independent entry points, each with its own detail document read on demand:

```text
SKILL.md                      Router: prerequisites, tool inventory, invocation shape, acceptance checks
references/materials.md       Entry 1: parameter mapping, output layout, four known losses
references/models.md          Entry 2: output layout, MaterialGroupList remap template, external references
references/particles.md       Entry 3: loose-file requirement, texture trio, empty-reference repair, renderer fields
```

## What it covers

Material parameter mapping (`$basetexture` / `$bumpmap` / `$phongexponent` / `$envmapmask` / `$selfillum` and friends onto target shader slots), the four kinds of files a model import produces and what each one means, how to judge the particle importer's coverage from its log, plus three templates you can apply directly:

- the model `MaterialGroupList` / `DefaultMaterialGroup` remap block
- the particle `.vtex` texture compile entry
- the particle `.mks` sprite-sheet descriptor

## Install

The repository root is the skill directory. Clone it into your platform's skills folder.

### Codex

```bash
git clone <repo-url> "$HOME/.codex/skills/source1-to-source2-assets"
```

Windows PowerShell:

```powershell
git clone <repo-url> "$env:USERPROFILE\.codex\skills\source1-to-source2-assets"
```

If `CODEX_HOME` is set, place it under `$CODEX_HOME/skills/` instead.

### Other platforms

Copy the whole directory into that platform's skills folder:

| Platform | Target |
| --- | --- |
| Claude Code | plugin directory or `skills/` |
| Cursor | plugin directory or `skills/` |
| Other | the platform's skill/rule directory |

### Merging into a skill collection

If you already use a skill collection repository, copy this directory under its `skills/` folder without changing the internal layout of `SKILL.md` and `references/`.

## Usage

Describe the task and the skill is selected automatically, for example:

> Port these Source 1 materials and models into my Source 2 addon

> This particle shows a missing-texture checkerboard after import — find out why

> The model imported, but its materials don't show up in the editor

Or invoke it explicitly: `$source1-to-source2-assets`

## Requirements

- **Source 2 workshop tools**: the skill relies on `source1import.exe`, `cs_mdl_import.exe`, `dmxconvert.exe`, and `resourcecompiler.exe`, located under the toolchain's `game/bin/win64/`.
- **Source-side tools**: `vpk.exe` (listing, extraction) and `vtf2tga.exe` (texture conversion), usually under the original game's `bin/`.
- Importing runs locally and needs no network access. Where files get written, and whether the original game directory may be touched, is up to your own authorization boundaries.

## Conventions

Every `<name>` in the skill stands for a concrete game, addon, or asset name; `<source2-install>` and `<s1-game>` stand for the relevant install roots. Nothing is bound to a specific project — substitute your own.

## Limitations

The pipeline is lossy. Each loss and its workaround is documented, but one-to-one fidelity is not guaranteed:

- Materials: mask textures, texture animation, and some blend modes need manual rework.
- Models: facial deformation has no direct equivalent on the target engine and usually has to be rebuilt or dropped.
- Particles: source-side texture scrolling has no direct equivalent and must be approximated with frame sequences or renderer animation.
- Converting everything is expensive; convert in batches based on what the map actually needs.

## Asset rights

Porting redistributes someone else's game assets. Local use and public distribution — especially uploading to a workshop or publishing a repository — have very different requirements. Confirm you have the right to redistribute before publishing. This skill documents a technical process and ships no game assets.
