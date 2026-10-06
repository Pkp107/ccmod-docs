---
title: Troubleshooting
section: Project
order: 5
---

# Troubleshooting

| Symptom | Fix |
|---|---|
| `no export` / `ccmod_invoke` errors | The DLL got reverted to a smaller build. Close CapCut, restore `ccmod_inject.dll.bak2` over `ccmod_inject.dll` |
| CapCut exits right after launch | Normal sometimes; `ccmod_run.py` retries. If it persists, close all CapCut processes in Task Manager and run it again |
| CapCut hangs or "insufficient disk space" | `%TEMP%` is full of Frida leftovers (~43 MB per attach). Run `python scratch/_clean_frida_tmp.py` |
| No **ccmod** category in Effects | Effects aren't installed (`python scratch/ccfx.py list`), or the panel was cached. Restart `ccmod_run.py`: it clears CapCut's saved catalog replies |
| Tiles appear but dragging does nothing / wrong effect | Wait for the editor to be fully open (`on_editor_open` installs the borrowed-id packages) and try again |
| Effect renders unchanged, black or passthrough | Shader compile error, or a CapCut build newer than 8.9.1. Check `python ccversion.py` |
| Animated effect looks still | `iTime` only advances during playback |
| "Subscription Service" screen on opening a project | CapCut's own server-side check; ccmod doesn't touch it. Closing the dialog can abort the open |
| Button doesn't appear | The place doesn't exist yet (`add_button` returns `False`). Call it in `on_editor_open`, which runs when the editor exists |
| Button click does nothing | An exception in your handler is logged (`ctx.log`); watch the terminal |
| Theme has no effect | The editor must be open; the theme re-applies on the next editor open. Try `ccmod_api.py theme-apply <id>` |
| Hid something with a theme rule | Revert: Loader Themes, **Revert**, or `python ccmod_api.py theme-revert` |
| Plugin missing from the list | `python -m ccmod_sdk list` shows why it was skipped (manifest error, wrong CapCut version) |
| `PermissionError` in a plugin | Add the namespace's permission to `plugin.json` |
| `FileNotFoundError: template transition package not cached` | Apply any stock transition once in CapCut, then build again |
| Blender render fails | Blender isn't found. Install it or set `ccmod_blender` to `blender.exe` |
| An auto-update removed 8.9.1 | `python ccversion.py restore 8.9.1` (needs a prior `backup`); `ccmod_run.py` also tries to restore it |

## Collecting information for a bug report

```bash
python ccversion.py --json
python -m ccmod_sdk list
python -m pytest -q
```

Plus the last 50 lines of the `ccmod_run.py` terminal. Open an issue at <https://github.com/Pkp107/capcut-ccmod/issues>.
