# CCMOD platform: buttons anywhere, plugins, tools, MCP

The goal: a user (or an AI) can add a new button or tool to CapCut with a few lines of Python. This is where each piece stands.
Status words: **live** = verified in the running CapCut, **tested** = unit-tested with the fake backend only, **plan** = not built.

## What a plugin can do today

| Need | API | Status |
|---|---|---|
| A button that runs Python when pressed | `ctx.ui.add_button(label, on_click, where=...)` | **live** (real click reached the handler, 6 Oct) |
| Change or remove it later | `ctx.ui.set_button_label(id, text)`, `ctx.ui.remove_button(id)` | **live** (remove), label **tested** |
| Any QML (panels, tabs, overlays) anywhere | `ctx.ui.inject_qml(selector, qml)` + `get_prop/set_prop` | **live** |
| Read playhead / seek / call CapCut's own controllers | `ctx.engine.playhead_seconds()`, `seek()`, `invoke()` | **live** |
| Add effects and transitions to CapCut's real panels | `ctx.effects.*`, `ctx.network.set_grid(...)` | **live** |
| Expose a tool to MCP clients | `ctx.tools.register(name, description, fn, schema)` | **tested** |
| Persist plugin settings | `ctx.storage.read_json/write_json` | **tested** |

`where` for buttons: `"timeline-toolbar"`, `"panel"`, `"tab-row"` (joins Media/Effects/Transitions), or any live QQuickItem class or
objectName. Buttons are idempotent, so call `add_button` from `on_editor_open`.

## MCP for CapCut

`ccmod_mcp.py` is an MCP server over stdio with no dependencies beyond Python.

```
claude mcp add ccmod -- python "C:\path\to\Capcut custom\ccmod_mcp.py"
```

* Always available (CapCut can be closed): status, library, install/uninstall effect or transition, set cover, launch, stop.
* While CapCut runs under `ccmod_run.py` (it starts a local tool server on 127.0.0.1, token-protected): playhead, seek, read/click UI
  items, list plugins, and every tool a plugin registered. Those appear in the client's tool list as `<plugin>__<tool>`.

Status: the protocol and the live-tool forwarding are **tested**; the MCP stdio handshake was checked by hand. A real MCP client
connecting to a running CapCut is **not** verified yet.

## The example ideas, honestly

**Audio scrubbing (hear the audio as you drag the playhead).** Plugin `audio-scrub`, built on the primitives above:
it polls `ctx.engine.playhead_seconds()`, tells scrubbing from normal playback by playhead speed, decodes the project's audio once
with ffmpeg and plays a 90 ms faded grain at the playhead (`winsound`). A Scrub button on the timeline toolbar turns it on.
Status: grain cutting and ffmpeg decode are **tested**; the button path is **live**; hearing it while scrubbing in CapCut is **not yet
verified**. Known limits: it needs a media file with a real path (the 0216 test project is a compound clip, so use the `set_source`
tool); it plays one source, not the mixed timeline.

**Render with ReShade 2D effects.** CCMOD effects are GLSL packages that CapCut itself renders and exports, so a ReShade shader that
works as screen-space passes can become a CCMOD effect and then appears in preview and in the export with no extra step. The format
supports several passes. The missing piece is a converter from ReShade FX (HLSL-like, with its own pass and texture syntax) to our
GLSL package, plus mapping ReShade uniforms to CapCut sliders. Status: **plan**. Shaders that need depth, or screen history from
previous frames, will not port.

**New tools in the UI.** A tool is a button plus Python code, or a tool registered for MCP, or both. The pieces exist; the missing
piece is polish: a Create page in the Loader for effects and transitions, and more named button places (preview area, inspector).

## More levers found by recon (6 Oct, live CapCut 8.9.1)

The live editor exposes about 400 custom QML/controller objects (93 view models) and 150 layout containers. All items below are
**recon only**: the object or data exists, but nothing here has been built or proven yet.

| Lever | Evidence | What a plugin could do |
|---|---|---|
| Keyboard shortcuts | `EditorShortCut`, `ExclusiveEditorShortcut`, `MultiCameraShortCut` QML types | Inject a QML `Shortcut`, same click-counter pattern as buttons: bind any key to Python |
| Right-click menus | `LVMenu`, `LVMenuItem`, `LVPopupMenu`, `MainTimeLineTrackHeaderMenuItem` (menus are model-driven lists, see the toolbar note) | Add items to clip, track and panel context menus |
| Selected-clip inspector | `DraftInspectorTransition`, `DraftInspectorVideo`, `DraftInspectorProjectConfig` (+ their view models) | Add our own sections or sliders to the inspector (e.g. a duration control for CCMOD transitions) |
| Preview overlays | `PlayerInteractiveArea`, `PlayerEffectControlArea`, `PlayerSafeArea`, `PlayerRuler` | On-canvas guides, grids, handles, color picker |
| Native dockable panels | `DockWidgets::*` (KDDockWidgets: `DockWidgetInstantiator`, `DockWidgetContainer`) | A CCMOD panel that docks and resizes like CapCut's own, instead of a floating injected panel |
| Settings dialog sections | `PanelSettingTabListModel`, `PanelSettingTabViewControl` | A CCMOD section inside CapCut's Settings |
| Editor state as events | `MainTimeLineCursorViewModel`, `MainTimeLineCanvasViewModel`, `MainTimeLineSegment{Video,Effect,Transition}ViewModel`, `ProjectConfigViewModel` | Polling-based `on_playhead`, `on_selection`, `on_edit` events for plugins |
| Export and prerender | `ExportButton`, `PrerenderViewModel`, `MainTimeLinePrerenderIndicator` | Run steps before or after an export (backup, post-process, notify) |
| CapCut's own test hook | `AutoTestHelper` | Probe it with `describe()`: it may offer built-in automation calls |
| Other panels via the retag route | The panel replies for filters, stickers, text, canvas and others use the same shape as effects (default category plus inline tiles; effect_type 1, 10, 11, 13, 27, ...) | CCMOD categories in Filters, Text effects, Canvas and more, like Effects and Transitions today |
| Global post-process | Shader compile and uniform calls are reachable in the render process | One shader pass over the final frame (a real ReShade-style chain), independent of clips |
| Projects as data | Drafts are plain JSON files | Generate projects, bulk-edit, beat-sync cuts, auto-subtitles from outside CapCut |
| Theming CapCut | Colors live in QML properties of the objects above | Recolor CapCut's UI by walking the live items (no theme file exists to edit) |

Suggested order by value over effort: shortcuts, context menus, inspector sections (fixes the 2.0 s transition duration), then
the other panels through the retag route.

## Rules that stay fixed

CCMOD never touches the account, Pro/subscription state, the watermark or licensing. Plugins only get the permissions their
`plugin.json` declares (`ui`, `engine`, `files`, `tools`, ...); a plugin that uses a namespace it did not declare is refused.
