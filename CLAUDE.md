# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Vibe Unity is a Unity Editor extension (UPM package `com.ricoder.vibe-unity` v2.0.0) that automates Unity development workflows through JSON commands and bash scripts. It enables AI-assisted scene creation, UI element generation, and compilation tracking. Requires Unity 2022.3+.

## Key Commands

```bash
# Validate C# compilation after code changes
./claude-compile-check.sh
./claude-compile-check.sh --include-warnings

# CLI operations (from project root when installed in a Unity project)
./vibe-unity --help
./vibe-unity list-types
./vibe-unity create-scene <name> <path> --type <2D|3D|Empty|DefaultGameObjects|URP|HDRP>
./vibe-unity start-session --scene <name> --canvas-mode <mode> --resolution <WxH>
./vibe-unity show-session
```

There is no automated test suite. Testing is manual via Unity menu items (Tools > Vibe Unity > Test Unity CLI).

## Architecture

All C# code lives in `Editor/` under the `VibeUnity.Editor` namespace (editor-only assembly, depends on TextMeshPro). Every class uses static methods organized by concern:

**Command Processing Pipeline:**
1. JSON files dropped in `.vibe-unity/commands/` by CLI or external tools
2. `VibeUnitySystem` — `FileSystemWatcher` detects new files, queues work to main thread via `EditorApplication.update`
3. `VibeUnityJSONProcessor` — parses JSON, dispatches to action handlers
4. Action handlers (`VibeUnityScenes`, `VibeUnityUI`, `VibeUnityPrimitives`, `VibeUnityGameObjects`) execute Unity API calls
5. Results logged to `.vibe-unity/processed/<filename>.log`, scene auto-saved on success

**Compilation Tracking Pipeline:**
1. `VibeUnityCompilationController` subscribes to Unity's `CompilationPipeline` events
2. Writes real-time status to `.vibe-unity/compilation/current-status.json`
3. Errors persisted to `.vibe-unity/compilation/last-errors.json`
4. `claude-compile-check.sh` reads these status files for external validation (exit codes: 0=success, 1=errors, 2=timeout, 3=script error)

**Other key classes:**
- `VibeUnityCLI` — public API entry points for all operations
- `VibeUnitySceneExporter/Importer` — scene state serialization to/from JSON with gap detection
- `VibeUnitySceneState` — data structures for scene state serialization
- `VibeUnitySettings` + `VibeUnitySettingsWindow` — JSON-based settings with UI (keyboard shortcuts, toggles)
- `VibeUnityMenu` — Unity Editor menu items under Tools > Vibe Unity
- `VibeUnitySetup` — one-time package initialization on import (copies scripts, creates directories)

## Runtime Directory Structure

When installed in a Unity project, the package creates:
```
.vibe-unity/
├── commands/           # File watcher input (JSON command files)
├── processed/          # Execution logs
└── compilation/
    ├── current-status.json   # Real-time compilation state
    ├── last-errors.json      # Latest error details
    └── project-hash.txt      # Project identifier
```

## JSON Command Format

Commands use this structure (see `README_CLAUDE.md` for full reference):
```json
{
  "version": "1.0",
  "description": "Brief description",
  "scene": { "name": "SceneName", "create": true, "path": "Assets/Scenes" },
  "commands": [
    { "action": "add-canvas", "name": "MainCanvas", "renderMode": "ScreenSpaceOverlay" },
    { "action": "add-button", "name": "Btn", "parent": "MainCanvas", "text": "Click" },
    { "action": "add-component", "name": "Obj", "componentType": "Rigidbody", "parameters": [...] }
  ]
}
```

Actions: `create-scene`, `add-canvas`, `add-panel`, `add-button`, `add-text`, `add-scrollview`, `add-cube/sphere/plane/cylinder/capsule`, `add-component`.

## Development Notes

- All Editor code is guarded with `#if UNITY_EDITOR` preprocessor directives
- Component property assignment uses reflection (`VibeUnityGameObjects`)
- The HTTP server (`VibeUnityHttpServer`) is disabled in v2.0.0; the file watcher is the primary integration mechanism
- Bash scripts target WSL and auto-convert paths between WSL and Windows formats
- Keyboard shortcuts are configurable via JSON settings (defaults: Ctrl+Shift+K status, Ctrl+Shift+U recompile, Ctrl+Shift+L clear)