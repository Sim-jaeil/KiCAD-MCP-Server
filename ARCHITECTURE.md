# KiCAD-MCP-Server Architecture

This document describes how the fork is wired end-to-end so future agents can add tools safely.

## 1) Runtime Entry Point and Process Model

- Node entrypoint is `src/index.ts:19` (`main()`), invoked unconditionally at `src/index.ts:112`.
- Build/runtime mapping is defined in `package.json:6` (`main: dist/index.js`), `package.json:8` (`build: tsc`), and `package.json:10` (`start: node dist/index.js`).
- `main()` loads config via `loadConfig()` from `src/config.ts:43` and resolves Python bridge path to `python/kicad_interface.py` at `src/index.ts:29`.
- The active server class is `KiCADMcpServer` in `src/server.ts:175`.
- Legacy file `src/kicad-server.ts` is present but not used by `src/index.ts`; treat it as historical/reference code unless explicitly reactivated.

## 2) MCP Server Startup Flow

1. `src/index.ts:32` constructs `KiCADMcpServer`.
2. Constructor in `src/server.ts:198` initializes MCP server + stdio transport and calls `registerAll()` (`src/server.ts:225`).
3. `registerAll()` registers router tools first (`src/server.ts:235`) and then direct tool/resource/prompt modules (`src/server.ts:238-263`).
4. `start()` in `src/server.ts:452` resolves Python executable (`findPythonExecutable()`), validates prerequisites, then spawns `python/kicad_interface.py` (`src/server.ts:471`).
5. Requests are sent to Python over stdin/stdout JSON lines through `callKicadScript()` + queue (`src/server.ts:550-719`).

## 3) Tool Registration and Router Pattern

### Direct tool registration

- Direct MCP tools are registered in per-domain modules (example: `src/tools/project.ts:8`), each wrapping `callKicadScript("<command>", args)`.
- This keeps tool schemas in TypeScript while business logic executes in Python command handlers.

### Router/discovery layer

- Router endpoints are defined in `src/tools/router.ts`:
  - `list_tool_categories` (`src/tools/router.ts:47`)
  - `get_category_tools` (`src/tools/router.ts:82`)
  - `execute_tool` (`src/tools/router.ts:130`)
  - `search_tools` (`src/tools/router.ts:223`)
- Registry metadata lives in `src/tools/registry.ts`:
  - category definitions `toolCategories` (`src/tools/registry.ts:26`)
  - always-visible direct set `directToolNames` (`src/tools/registry.ts:123`)
  - lookup/search helpers (`src/tools/registry.ts:174`, `src/tools/registry.ts:181`, `src/tools/registry.ts:227`).

### Important current-state detail

- `router.ts` supports handler-based routed execution via `registerToolHandler()` (`src/tools/router.ts:29`), but current tool modules do not call it.
- Therefore `execute_tool` usually goes through fallback path (`src/tools/router.ts:157-164`) and calls Python command names directly.

## 4) How To Add a New Tool and/or Category

### Add a new tool to an existing category

1. Add MCP tool schema/wrapper in the right TS module (pattern in `src/tools/project.ts:10-97`).
2. Add the command mapping in Python command map (`python/kicad_interface.py:306-410`).
3. Implement the Python handler in an existing/new command class under `python/commands/`.
4. Add tool name to category in `src/tools/registry.ts:26` list for discoverability.
5. Ensure module registration is called from `registerAll()` in `src/server.ts:238-251`.

### Add a new category

1. Add a new `ToolCategory` object in `src/tools/registry.ts:26`.
2. Add tools to that category's `tools` array.
3. Rebuild; `initializeRegistry()` (`src/tools/registry.ts:159`) auto-populates maps.
4. Verify with router tools:
   - `list_tool_categories`
   - `get_category_tools`
   - `search_tools`
   - `execute_tool`

### Optional routed-handler migration (recommended)

- For true routed execution (without fallback), each tool module should register a handler via `registerToolHandler()` (`src/tools/router.ts:29`) during tool registration.

## 5) Python Backend Architecture

- Main bridge is `python/kicad_interface.py`.
- It selects backend mode:
  - IPC preferred when available (`python/kicad_interface.py:115-133`)
  - SWIG fallback using `pcbnew` import (`python/kicad_interface.py:136-145`).
- Command dispatch table binds MCP command strings to methods (`python/kicad_interface.py:306-410`).
- Domain logic is split into `python/commands/*.py` (project, board, routing, DRC, export, library, schematic, etc.).

## 6) DRC Timeout Locations (Exact)

### TypeScript request-level timeout policy

- Default command timeout is `30000ms` in `callKicadScript()` (`src/server.ts:561`).
- Long-running command list includes `run_drc`, `export_gerber`, `export_pdf`, `export_3d` (`src/server.ts:562-567`).
- Those commands are raised to `600000ms` (`10 minutes`) (`src/server.ts:569`).
- Request timeout is enforced in queue processor via `setTimeout()` (`src/server.ts:675-698`).

### Python subprocess timeout policy for DRC

- `run_drc()` executes `kicad-cli pcb drc ...` with `timeout=600` (`python/commands/design_rules.py:246`).
- Optional text report subprocess also uses `timeout=600` (`python/commands/design_rules.py:334`).
- Timeout exception handling returns explicit DRC timeout failure (`python/commands/design_rules.py:354-360`).

## 7) `kicad-skip` Usage and Safety Assessment

### Observed usage in this repository

- Literal `kicad-skip` appears in docs/README architectural notes (`README.md:64`, `README.md:71`, `README.md:73`).
- A related implementation comment appears in schematic delete handler: "no skip writes" (`python/kicad_interface.py:712`).
- No literal `kicad-skip` token is used in TypeScript/Python executable code paths.

### Read/write safety assessment

- Schematic operations include direct text manipulation of `.kicad_sch` content (`python/kicad_interface.py:733-799` shown in delete flow).
- Safety measures present:
  - avoids deleting inside `lib_symbols` by range skip (`python/kicad_interface.py:736-761`)
  - deletes matched symbol blocks from back-to-front (`python/kicad_interface.py:794-798`).
- Residual risk:
  - text/S-expression editing is structurally fragile if format assumptions change.
  - any future write-path expansion should keep parser-based edits and add regression tests with real schematic fixtures.

## 8) Platform-Specific and macOS Notes

### Node-side Python discovery

- Platform flags are computed in `findPythonExecutable()` (`src/server.ts:47-49`).
- macOS search includes KiCad app bundle Python across versions `3.9..3.13` and common app paths (`src/server.ts:93-113`).
- macOS Homebrew fallback paths are checked (`src/server.ts:116-121`).

### Python-side platform handling

- Platform and environment are logged early (`python/kicad_interface.py:34-37`).
- macOS troubleshooting guidance is emitted on import failure (`python/kicad_interface.py:161-167`).
- DRC CLI finder includes macOS paths (`python/commands/design_rules.py:394-401`).

### Operational implication

- macOS reliability depends on matching KiCad-bundled Python/pcbnew ABI or a correctly configured alternate interpreter.
- When backend import fails, server exits early with actionable diagnostics (`python/kicad_interface.py:178-184`).

## 9) Build, TypeScript, and Validation Context

- Compiler settings are strict and NodeNext (`tsconfig.json:4-8`).
- Build artifact path is `dist/` (`tsconfig.json:8`, `package.json:6`).
- Server prerequisite check requires build artifact `dist/index.js` (`src/server.ts:357-364`).

## 10) Practical Extension Checklist

1. Implement Python command handler in `python/commands/` and wire in `python/kicad_interface.py` map.
2. Add/update TS tool schema wrapper under `src/tools/` and ensure registration in `src/server.ts`.
3. Add registry metadata (`src/tools/registry.ts`) so router discovery stays accurate.
4. For heavy jobs, classify timeout behavior (default vs long-running list in `src/server.ts:562-569`).
5. Verify on at least one target platform (macOS pathing is a common failure mode).
