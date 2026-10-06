---
title: SDK reference
section: Reference
order: 10
---

# SDK reference

Every plugin gets `self.ctx`. Each namespace needs its permission in `plugin.json`; using one you didn't declare raises `PermissionError`.

## Permissions

| Permission | Allows |
|---|---|
| `ui` | Inject QML (buttons, panels, tabs), read/write UI properties, themes |
| `engine` | Call CapCut controllers, seek the playhead, read engine state |
| `render` | Rewrite shaders / hook the render pipeline |
| `draft` | Read and modify project drafts (with DraftGuard backups) |
| `effects` | Install, list and apply CCMOD effects and packs |
| `network` | Redirect CapCut's network requests / serve catalog rules locally |
| `files` | Read/write inside the plugin's own data folder |
| `builds` | Inspect installed CapCut builds |
| `tools` | Expose tools to MCP clients and the tool server |
| `media` | Import files into CapCut's media pool |
| `process` | Run external programs |

List them any time: `python -m ccmod_sdk permissions`.

## `Plugin`

```python
from ccmod_sdk import Plugin
class P(Plugin):
    def on_load(self): ...
    def on_unload(self): ...
    def on_capcut_start(self): ...
    def on_editor_open(self): ...
    def on_project_open(self, project): ...
```

## ctx.ui

| Call | Returns / does |
|---|---|
| `add_button(label, on_click, where="timeline-toolbar", id=None, icon=None, x=None, y=None, width=110, height=28)` | Adds a button; `on_click()` runs in Python. `where`: `"timeline-toolbar"`, `"panel"`, `"tab-row"` or any live class/objectName. Idempotent per `id`. Returns `False` when the place doesn't exist yet |
| `set_button_label(id, label)` | Change a button's text |
| `remove_button(id)` | Hide it and stop handling clicks |
| `inject_qml(parent_selector, qml)` | Inject QML text under the live item |
| `inject_qml_file(parent_selector, path)` | Same from a file |
| `get_prop(selector, prop)` / `set_prop(selector, prop, value)` | Read/write a property of a live item |
| `click(selector)` | Click a live item by objectName |
| `find_tab_row()` | Address of the editor's top tab row, or `None` |
| `inspect(type="", objectName="", limit=40)` | List live UI items (for theme/plugin authors) |
| `themes()` / `apply_theme(id)` / `revert_theme()` | Theme control |

Button placement: `x`/`y` position the button inside the parent; the default row is the timeline toolbar. Button clicks are counted by the QML and polled every 0.2 s, so a click reaches Python within a fraction of a second.

## ctx.engine

| Call | Meaning |
|---|---|
| `invoke(selector, method, args="")` | Call a method on a live CapCut QObject. Args: `"i:123"` int, `"sl:a|b"` string list, plain text a string; combine with `|` |
| `get_prop` / `set_prop` | Read / write a QObject property |
| `list_objects(limit=0)` | Class names of CapCut's live view-models |
| `describe(selector, depth=5)` | Methods and properties of a class |
| `playhead_seconds()` | Current playhead |
| `seek(seconds)` | Move the playhead |
| `timeline_duration_seconds()` | Timeline length |

!!! note "Arguments"
    Pointer or object arguments (such as `QQuickItem*`) aren't supported by `invoke`. For those, inject a QML snippet that calls the view-model from inside the scene (this is how `ctx.media` imports files).

## ctx.effects

| Call | Meaning |
|---|---|
| `library()` | Names of the built-in effect library |
| `installed()` | Installed donorless effects as tile dicts |
| `install(name)` | Build an effect package from the library |
| `grid_items()` / `transition_items()` | Installed effects / transitions as CCMOD grid items |
| `install_borrowed(state_dir)` | Place packages under the stock ids the grid borrowed |
| `apply(project, name)` | Write the effect into a project's draft |
| `set_param(key, value)` | Drive the selected effect's slider live (no reload) |

## ctx.network

| Call | Meaning |
|---|---|
| `set_grid(items, name="CCMOD", extras_name="CCMOD Extras", main_count=8, transitions=None)` | Publish the CCMOD category for effects (and transitions) |
| `add_category(category_id, name, position=0, key=None)`, `add_tiles(items, effect_type=7, mode="append")`, `serve_category(category_id, items, effect_type=7)` | Lower-level catalog rules |
| `redirect(pattern, local_file)` / `rules()` / `clear()` | Answer CapCut's requests from local files (experimental) |
| `catalog_state_dir` | Where the proxy records borrowed ids |

## ctx.media, ctx.process, ctx.tools

See [External apps](external-apps.html) and [MCP](mcp.html). Summary:

```python
ctx.media.import_file(path); ctx.media.import_files([paths])
ctx.process.which(name, extra_dirs, env_var); ctx.process.run(cmd, timeout, cwd, env); ctx.process.run_async(cmd, on_done)
ctx.tools.register(name, description, fn, schema=None); ctx.tools.unregister(name)
```

## ctx.storage

Private to your plugin; paths that escape the folder raise.

| Call | Meaning |
|---|---|
| `read_json(name, default=None)`, `write_json(name, data)` | JSON files |
| `path(name)` | `Path` inside your folder (you create subfolders yourself) |
| `folder` | Your folder |

## ctx.draft, ctx.builds, ctx.hooks

| Call | Meaning |
|---|---|
| `draft.save(project)`, `restore(project)`, `format_of(project)` | Protect a project draft from CapCut's format bumps |
| `builds.installed()`, `builds.preferred()` | CapCut builds on this PC |
| `hooks.resolve(name, build)` | An engine function's RVA, found by byte signature so it survives updates |

## ctx.events and ctx.log

`ctx.events.on(event, fn)`, `ctx.events.emit(event, ...)` is an in-process event bus between plugins. `ctx.log` is a standard `logging.Logger`.

## Backends

`FridaBackend` attaches to the running CapCut and talks to `ccmod_inject.dll` (Qt metaobject reflection). `FakeBackend` records calls and keeps in-memory properties for tests.

## Python-level helpers (no CapCut needed)

```python
from ccmod_sdk import themes, transitions, blender
themes.discover(); themes.validate(theme); transitions.build(key); blender.find_blender()
```
