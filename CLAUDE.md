# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Overview

Snapmaker Orca is a 3D slicer forked from OrcaSlicer (itself a Bambu Studio fork),
C++17 with wxWidgets for the GUI and CMake as the build system. On the
`feature/cad-primitives` branch it embeds a **parametric, sketch-first CAD "Design"
tab**: draw 2D sketches on a plane, constrain them, turn them into solids
(extrude/revolve/sweep/loft, fillet/chamfer/shell/draft/hole/thread, pattern,
boolean), and commit straight to the plater — without leaving the slicer.

The CAD model is a **recipe, not a mesh**: every action is a feature in an ordered
tree replayed from the start on each change, so editing a dimension set twenty steps
ago rebuilds everything downstream. The recipe is persisted inside the 3MF.

### Design-tab layout

| Area | Location | Role |
|---|---|---|
| Kernel | `src/libslic3r/CAD/` | `CadDocument` feature tree + `recompute()`, OCCT B-rep. OCCT is already linked for STEP import; the CAD module adds `TKFillet`/`TKOffset`. |
| Solver | `src/libslic3r/slvs/` | Vendored SolveSpace `libslvs` 2D constraint solver (GPL-3.0). |
| GUI | `src/slic3r/GUI/CAD/` | `DesignPanel`, `DesignCanvas`, `DesignSketchTool`, `SketchInlineEditor`, `DesignOffer`, `McpControl`. |
| MCP control | `src/slic3r/GUI/McpControl.cpp` | Machine-controllable surface: JSON-RPC over a unix socket, off unless `SNAPORCA_MCP` is set. Facade over the same `CadDocument` the GUI drives — never a parallel engine. |
| Tool offer | `docs/ux/tool_atlas.json` → `src/slic3r/GUI/CAD/DesignOffer.hpp` | Object-driven offer menu: point at geometry, the geometry offers the verbs that apply. `DesignOffer.hpp` is **generated** by `docs/ux/gen_offer_table.py`; the map lives once in `tool_atlas.json`. |

### Forks and remotes

- This repo is the Snapmaker base. Its twin (same CAD, mainline OrcaSlicer base) is
  `tommasobbianchi/Orca-Cad`, maintained in parallel at ~17-identical/8-diverging file
  parity across the shared CAD files.
- Push to the `fork` remote only. **Never push to the upstream OrcaSlicer remotes.**

## Build Commands

### Linux
```bash
./build_linux.sh -u     # install system dependencies
./build_linux.sh -dsi   # build deps + slicer + AppImage
./build_linux.sh -d     # dependencies only
./build_linux.sh -s     # slicer only
./build_linux.sh -j N   # limit to N cores
./build_linux.sh -b     # debug build
```

### macOS
```bash
./build_release_macos.sh          # everything
./build_release_macos.sh -d       # deps only
./build_release_macos.sh -s       # slicer only
./build_release_macos.sh -a arm64 # architecture
```

### Windows
```bash
build_release_vs2022.bat          # everything
build_release_vs2022.bat deps     # deps only
build_release_vs2022.bat slicer   # slicer only
```

### Build system
- CMake ≥ 3.13. Primary build dir `build/`, deps in `deps/build/`.
- The official CI (`build_linux.sh -ur`, `-dr`, `-isr`) builds the **pinned** deps
  first; do not substitute apt-installed OCCT/OpenCV/paho for the pinned ones.

## Testing

```bash
cd build && ctest                        # all suites
./tests/libslic3r/libslic3r_tests        # kernel suite, incl. [CadDocument]
```

Kernel code is Catch2-tested (`tests/libslic3r/`); the GUI layer (`src/slic3r/GUI/CAD`)
has no unit tests — its coverage is the headless rig (`scripts/CAD/`) driven over the
MCP socket. Before committing, the offer-table check must pass:

```bash
python3 docs/ux/gen_offer_table.py --check
```

## Development conventions

- **C++17**, PascalCase classes, snake_case functions/locals; 4-space indent,
  140-column limit (`.clang-format`).
- Kernel changes land in `libslic3r/CAD` and are exercised headlessly (Catch2); GUI
  changes in `slic3r/GUI/CAD` are verified on the rig or a display, not trusted from a
  clean build.
- The offer table is generated from `docs/ux/tool_atlas.json` — edit the atlas, never
  hand-edit `DesignOffer.hpp`. Row order is ratified; changing an index breaks every
  user's muscle memory.
- Mockups and point-in-time plans/prototypes live in `sandboxes/`, not in `docs/`.
  `docs/` holds only the high-level design and the key technical decisions
  (`design_tab.md`, `cad_ux_guidelines.md`, `cad_dependency_weight.md`,
  `design_tab_upstream_portability.md`, `GAP_ANALYSIS_vs_ONSHAPE.md`).
- `deps/` and `deps_src/` are vendored snapshots — do not modify without mirroring the
  upstream tag.
