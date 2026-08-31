# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Downloads official Microsoft Windows patches and integrates them into an official Windows ISO to produce an updated install image. The patch download URLs/hashes are stored as `.meta4` (metalink4 XML) files that are regenerated automatically from the Microsoft Update Catalog.

The workflow is Windows-only: put one official `.iso` in the repo root, run `Start.cmd` as Administrator, and the script extracts the ISO, determines build/arch/language, downloads patches via `aria2`, and delegates integration to `W10UI.cmd`.

## Commands

There is no build/lint/test suite — this is a batch/PowerShell project. The two relevant entry points:

```bash
# Full pipeline (Windows, elevated): ISO in repo root required
Start.cmd

# Regenerate .meta4 files from Microsoft Update Catalog (PowerShell)
powershell -ExecutionPolicy Bypass -File Update-Meta4.ps1
```

`Update-Meta4.ps1` parameters:

```bash
powershell -ExecutionPolicy Bypass -File Update-Meta4.ps1 -Build 26100 -Arch x64
powershell -ExecutionPolicy Bypass -File Update-Meta4.ps1 -Build 26100,22621 -Arch x64,arm64
powershell -ExecutionPolicy Bypass -File Update-Meta4.ps1 -TestMode   # dry run, no writes
powershell -ExecutionPolicy Bypass -File Update-Meta4.ps1 -OutputDir <dir>
```

If a run is interrupted by catalog errors, restore the previous state with `git -C <repo> checkout -- Scripts/` (the script prints this hint itself).

## Architecture

Three top-level components plus the data files:

- **`Start.cmd`** — orchestrator. Detects the ISO in the root, extracts it with the bundled `7z`, reads `sources\install.wim`/`install.esd` with `dism.exe` to determine `build`/`arch`/`lang`/server flag, normalizes the build number (see below), then downloads the matching `.meta4` via `aria2` and calls `W10UI.cmd`. Bilingual (zh-CN / English) output selected by OS UI language.
- **`W10UI.cmd`** (~4100 lines) — the integration engine, a fork of `abbodi1406/BatUtil` W10UI (`uiv=v10.62`). Mounts the image with DISM, installs updates (LCU/SSU/.NET/DU), handles `boot.wim`/`winre.wim`, rebuilds and creates the ISO. This fork adds the custom options `ltscfix`, `nosuggapp`, `nosuggtip`, `norestorage`, `nogamebar`, `oobebypass` (applied in the `:cuinstall` region around lines 1758–1816) and the `apply26h2` enablement-package handling.
- **`Update-Meta4.ps1`** — regenerates `Scripts/*.meta4` by scraping the Microsoft Update Catalog (search → supersedence chain → download links) cross-validated against MS Update History pages (for the README build numbers). It filters out Preview-channel updates (title `-notmatch 'Preview'`) so only mainline GA updates are selected. Run locally to test, or automatically by GitHub Actions.
- **`Scripts/*.meta4`** — metalink4 XML files; one per `build × arch`, listing download URL + SHA-1 per update file. These are the data, not code. They are auto-committed by CI.

### Build-number mapping and filename conventions

`Start.cmd` normalizes detected OS build numbers to a canonical "family" key before looking up a meta4 file:

- `19042`–`19045` → `19041`; `20349` → `20348`; `22631` → `22621`; `26200`–`26300` → `26100`
- `26100` + server product type → `script_server_26100_<arch>.meta4` instead of `script_26100_<arch>.meta4`

Meta4 file naming: `Scripts/script_<build>_<arch>.meta4` (arch ∈ `x64`, `x86`, `arm64`), with `script_server_` prefix for server, and `.NET` updates split into `Scripts/netfx4.8/` and `Scripts/netfx4.8.1/` subdirectories as `script_<netfxver>_<build>_<arch>.meta4`. The `.NET` files carry a `<language>` tag per entry; the main files do not. `28000` has no `x86` variant.

### Configuration

`W10UI.ini` overrides the `W10UI.cmd` defaults (the script reads it at runtime; deleting it restores the in-script defaults). Custom flags of note:

- `ltscfix=1` — provisions VP9 extensions for LTSC 2021/2024 (uses `bin\Microsoft.VP9VideoExtensions_*`).
- `apply26h2=1` — integrates the Windows 11 26H2 enablement package (KB5121794); default `0` keeps 25H2.
- `SkipKB=5054156,5121794` — comma-separated KB numbers deleted from `patch/` after download, so they are excluded from integration. Default excludes the 25H2 (KB5054156) and 26H2 (KB5121794) enablement packages, keeping the 24H2 baseline. Clear it to let `apply26h2` switch 25H2/26H2.
- `netfx481=1` — include .NET Framework 4.8.1 support.

## Conventions and gotchas

- **Line endings are enforced** by `.gitattributes`: `*.cmd`/`*.bat`/`*.ps1` must be CRLF (`eol=crlf`), `*.sh` LF. Do not commit these files with LF endings — batch scripts break.
- **Two READMEs in sync**: `README.md` (English) and `README_cn.md` (Chinese). `Update-Meta4.ps1` updates both (date + build versions) via regex, and the CI workflow commits both. Keep any change mirrored in both.
- **`.gitattributes` `export-ignore`**: `Update-Meta4.ps1` and `.github/` are excluded from release archives (the source tarball users download).
- **Bundled binaries** in `bin/` (7z, aria2, wimlib, oscdimg, PSFExtractor) are the build tools — `Start.cmd` picks `bin64\` variants on amd64/arm64 hosts. These are third-party and should not be edited.
- **CI** (`.github/workflows/update-meta4.yml`) runs `Update-Meta4.ps1` on a schedule and on pushes touching `Scripts/`, commits meta4 + README changes, creates a versioned tag/release (`vYYYY.M[b|d][_N]`), and triggers the downstream `adavak/win_iso_build` repo.
- `Update-Meta4.ps1` preserves "baseline"/checkpoint LCUs and language-specific `.NET` files across runs; the sweep logic that drops stale superseded MSUs is subtle — read the comments in the `# 3. Preserve old MSUs` block before touching it.
