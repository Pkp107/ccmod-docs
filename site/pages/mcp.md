---
title: MCP & AI agents
section: Using CCMOD
order: 40
---

# MCP: let an AI drive CapCut

`ccmod_mcp.py` is a standard **MCP stdio server**. Point Claude Code (or any MCP client) at it and the AI can install effects, change themes, launch CapCut, move the playhead, and call every tool your plugins register.

## Connect Claude Code

```bash
claude mcp add ccmod -- python "C:/path/to/capcut-ccmod/ccmod_mcp.py"
```

Restart the client; the tools appear as `ccmod_*`.

!!! note "Status"
    The server speaks JSON-RPC over stdio (initialize, tools/list, tools/call, ping) and its handshake is covered by tests. It hasn't been exercised by a real MCP client yet, so report any quirks.

## Tools that always work (CapCut need not be running)

| Tool | What it does |
|---|---|
| `ccmod_status` | Is CapCut running and CCMOD attached |
| `ccmod_library` | Effects and transitions (installed or not) with cover paths |
| `ccmod_install` `{kind, key}` | Install an effect or transition (`kind` = `effect` or `transition`) |
| `ccmod_uninstall` `{kind, key}` | Remove one |
| `ccmod_set_cover` `{cover_key, image_path}` | Use an image as a tile cover |
| `ccmod_themes` | List themes + current |
| `ccmod_theme_apply` `{id}` / `ccmod_theme_revert` | Restyle CapCut / go back |
| `ccmod_launch` / `ccmod_stop` | Start CapCut with CCMOD (about a minute) / close it |

## Live tools (CapCut + CCMOD running)

When `ccmod_run.py` is running it starts a **tool server** on `127.0.0.1` (random port, token in `plugin-state/toolserver.json`). The MCP server forwards to it, so you also get:

| Tool | What it does |
|---|---|
| `ccmod__plugins` | Loaded plugins, enabled state, last error |
| `ccmod__playhead` / `ccmod__seek` | Read the playhead and timeline length, move the playhead |
| `ccmod__ui_get`, `ccmod__ui_click` | Read a property of / click a live UI item |
| `ccmod__engine_objects` | List CapCut's live view-model classes |
| `<plugin>__<tool>` | Anything a plugin registered with `ctx.tools.register`, e.g. `blender-bridge__render_3d`, `audio-scrub__...` |

(Dots in internal names become `__` in MCP.)

## Expose your own tool

```python
self.ctx.tools.register(
    "render_3d", "Render a 3D turntable and import it.",
    self.render_3d,
    {"type": "object", "properties": {"kind": {"type": "string"}, "seconds": {"type": "number"}}})
```

Requires the `tools` permission. The tool is listed to every MCP client automatically, as `<your-plugin-id>__render_3d`. Return JSON-serialisable data. An exception becomes a clean error result and never takes the server down.

## The raw HTTP API

```bash
# token and port are in plugin-state/toolserver.json
curl -H "X-CCMOD-Token: <token>" http://127.0.0.1:<port>/tools
curl -H "X-CCMOD-Token: <token>" -d '{"name":"ccmod.seek","arguments":{"seconds":12.5}}' http://127.0.0.1:<port>/call
```

Local only: bound to `127.0.0.1`, every request needs the token, and the token file is readable only by your Windows user.
