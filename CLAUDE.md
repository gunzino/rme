# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Remere's Map Editor (RME) 3.8.0 — a C++20 wxWidgets/OpenGL map editor for OpenTibia servers (OTBM map format). GPLv3. No unit-test framework exists; `test/` is a prebuilt runnable copy of the app (exe + DLLs + data), not a test suite.

## Building (Windows, Visual Studio)

Build via the MSBuild solution `vcproj/RME.sln`, configuration **Release | x64** (or Debug | x64). Open it in Visual Studio 2022, or build from the command line:

```powershell
& "C:\Program Files\Microsoft Visual Studio\2022\Community\MSBuild\Current\Bin\MSBuild.exe" `
    vcproj\RME.sln /p:Configuration=Release /p:Platform=x64 /m
```

Output: `vcproj/x64/Release/RME_x64.exe`, with runtime DLLs deployed next to it automatically. The exe locates `data/clients.xml` by probing its own directory and `../`, `../../` etc., so it picks up the repo `data/` directory.

Dependencies come from vcpkg (`C:\vcpkg`, `VCPKG_ROOT` is set) with user-wide MSBuild integration (`vcpkg integrate install`) providing includes/libs — the project file has no explicit dependency paths except the wxWidgets `setup.h` directories (`$(VCPKG_ROOT)\installed\x64-windows\lib\mswu` for Release, `debug\lib\mswud` for Debug), which the integration does not add on its own. Required packages (triplet `x64-windows`): `wxwidgets boost-date-time boost-system boost-filesystem boost-iostreams fmt nlohmann-json freeglut zlib asio`. The Win32/x86 configurations are not usable (no x86 vcpkg packages installed).

Notes baked into the project that must not be lost when touching it: x64 toolset is v143, `/utf-8` is required (fmt refuses to compile without it), `FMT_SHARED` because vcpkg's fmt is a DLL.

There is no lint step and no test suite. Adding/removing a source file requires editing `vcproj/Project/RME.vcxproj` (and `source/CMakeLists.txt`, which other platforms build from).

## Code conventions

- Tabs for indentation, K&R braces, LF line endings (`.editorconfig`).
- Every `.cpp` starts with `#include "main.h"` — it's the umbrella/precompiled header (wx, asio, fmt, nlohmann/json, `definitions.h`). The MSBuild build uses it as a real PCH (generated via `mkpch.cpp`).
- Use the `newd` macro instead of `new` (defined in `main.h`; enables leak tracking under `DEBUG_MEM`).
- No `dynamic_cast` — the codebase uses manual RTTI: `Brush::isGround()/asGround()` etc. When adding a Brush/Item subclass, add the matching `isXxx()`/`asXxx()` pair.
- wxString↔std::string via the `nstr()`/`wxstr()` macros from `definitions.h`.
- Feature flags live in `source/definitions.h` (`OTGZ_SUPPORT` is off by default, `_USE_UPDATER_`, `_USE_PROCESS_COM`, version macros).

## Architecture

### Global singletons (plain extern globals)

| Global | Header | Role |
|---|---|---|
| `g_gui` | `source/gui.h` | GUI god-object: editors/tabs, palettes, current brush, `GraphicManager gfx`, copybuffer. Nearly everything routes through it. |
| `g_settings` | `source/settings.h` | Config store, keys in the `Config::` enum |
| `g_items` | `source/items.h` | Item database from `items.otb` + `items.xml` |
| `g_creatures` | `source/creatures.h` | Creature database |
| `g_materials` | `source/materials.h` | Tilesets loaded from material XMLs |
| `g_brushes` | `source/brush.h` | Brush registry (name → `Brush*`) + autoborders |

Bootstrap: `Application::OnInit()` (`source/application.cpp`) — discovers `data/`, loads settings, loads `data/clients.xml` (`ClientVersion::loadVersions()`), creates `MainFrame`. A full client version is loaded/unloaded in exactly one place: `GUI::LoadVersion`/`UnloadVersion` (`source/gui.cpp`) — sprites → `items.otb` → `items.xml` → `creatures.xml` → `materials.xml` → extensions → `g_brushes.init()`.

### Map data model

`BaseMap` (`source/basemap.h`) stores tiles in a 16-way tree of `QTreeNode`s (`source/map_region.h`, splits x/y 4 bits per level); leaves hold `Floor*[16]` (one per z), each `Floor` = 16×16 `TileLocation`s. A `TileLocation` is the *persistent* per-coordinate slot; the `Tile*` inside it is replaceable — this is what makes undo a symmetric tile swap. `Tile` (`source/tile.h`) holds ground item, item stack, creature, spawn, house id, flags. `Map : BaseMap` (`source/map.h`) adds metadata plus `Towns`, `Houses`, `Spawns`, `Waypoints` (houses/spawns are saved as separate XML sidecar files next to the `.otbm`).

### Undo/actions — how to mutate the map

All map mutations must go through the action queue (`source/action.h`), never by editing live tiles directly:

```cpp
BatchAction* batch = editor.createBatch(ACTION_DRAW);
Action* action = editor.createAction(batch);
Tile* newtile = tile->deepCopy(map);   // mutate the COPY
// ... modify newtile ...
action->addChange(newd Change(newtile));
editor.addBatch(batch);                 // commits, enables undo, feeds live-sync
```

`Action::commit()` swaps the new tile in and keeps the old one in the `Change` — undo is the reverse swap. `Selection` (`source/selection.h`) wraps the same machinery. The `DirtyList` produced by actions drives live-collaboration broadcasting (`Editor::BroadcastNodes`).

### Brushes and materials

Abstract `Brush` (`source/brush.h`) with `draw/undraw/canDraw`; subclasses per file (`ground_brush`, `wall_brush`, `doodad_brush`, `carpet_brush`, `table_brush`, `raw_brush`, `creature_brush`, house/waypoint/spawn/flag/door/eraser). Per-client-version XMLs under `data/<version>/` (`materials.xml` includes `borders.xml`, `grounds.xml`, `walls.xml`, `doodads.xml`, `tilesets.xml`) are parsed by `Materials` into `g_brushes` and `Tileset`s, which the palettes (`source/palette_*.cpp`) render. `brush_tables.cpp` holds static border/wall lookup tables.

### IO

- `source/filehandle.h` — OTBM binary node-format primitives (also used by live networking via the memory variants).
- `source/iomap_otbm.cpp` — the OTBM loader/saver; houses/spawns via pugixml (vendored in `source/ext/`).
- `source/client_version.cpp` + `data/clients.xml` — registry of supported client/OTB/map versions; each client version maps to a `data/<version>/` folder and a user-configured path to the client's `Tibia.dat`/`Tibia.spr`.

### Rendering & GUI

`MapCanvas : wxGLCanvas` (`source/map_display.cpp`) handles input; `MapDrawer` (`source/map_drawer.cpp`) does the immediate-mode OpenGL pass; `GraphicManager` (`source/graphics.cpp`) lazily loads sprites from the client's `.dat`/`.spr` (descriptor: `.otfi` file, template in `tools/Tibia.otfi`). Editor icons are byte arrays embedded in `source/pngfiles.cpp`, regenerated from `brushes/*.png` by `tools/convert_png2cpp.py`. Menus are built from `data/menubar.xml` (`source/main_menubar.cpp`). Dialogs follow the `*_window.cpp` naming convention. GUI state must only be touched on the main thread; live-network results are marshalled back via wx events.

### Live collaboration

`LiveServer`/`LiveClient`/`LivePeer` over standalone asio (`source/live_*.cpp`, `source/net_connection.cpp`) exchange serialized map nodes and remote actions; protocol pinned by `__LIVE_NET_VERSION__` in `definitions.h` — bump it when changing `live_packets.h`.
