---
title: Install
section: Getting started
order: 10
---

# Install

## Requirements

| Need | Details |
|---|---|
| Windows 10/11 | ccmod injects into the Windows build of CapCut Desktop |
| CapCut **8.9.1.3802** | Newer builds currently render our shader packages as passthrough. ccmod can keep and launch the pinned build for you (see below) |
| Python 3.10+ | `python --version` |
| Frida | `pip install frida frida-tools`, the injector ccmod uses to attach to CapCut |
| .NET 8 SDK (optional) | only to build the Loader app yourself |

## 1. Get the code

```bash
git clone https://github.com/Pkp107/capcut-ccmod.git
cd capcut-ccmod
pip install frida frida-tools markdown pytest
```

## 2. Check the injection DLL

The full build of `ccmod_inject.dll` exports `ccmod_invoke`. Confirm you have it:

```bash
python -c "print(b'ccmod_invoke' in open('ccmod_inject.dll','rb').read())"
```

It must print `True`. If not, you have the wrong DLL; see [Troubleshooting](troubleshooting.html).

## 3. Make sure the right CapCut build is installed

CapCut auto-updates, and builds after 8.9.1 break our effect packages. Check what you have:

```bash
python ccversion.py
```

It lists installed builds under `%LOCALAPPDATA%\CapCut\Apps\` and which one ccmod will launch. `ccmod_run.py` also restores the pinned build from ccmod's backup if an auto-update removed it. Always start CapCut through ccmod, never the auto-updating stub.

!!! note "Sign-in"
    CapCut may show a subscription or sign-in interstitial when opening a project. That is CapCut's own behaviour and ccmod does not touch it.

## 4. Run it

```bash
python ccmod_run.py
```

This one command:

1. checks the pinned CapCut build,
2. clears CapCut's cached catalog replies and starts the local catalog proxy,
3. loads your plugins once so their grid rules exist before CapCut asks for the Effects panel,
4. launches CapCut (retrying, because a launch right after a close often exits),
5. attaches the SDK to the live editor and keeps running: every time an editor opens, plugins get `on_editor_open`.

Leave the terminal open. `Ctrl+C` stops ccmod (CapCut stays open but loses the ccmod extras at next launch).

To start ccmod with Windows: `python ccmod_run.py autostart`.

## 5. Install some effects

ccmod ships a library of effects and transitions. Install them once:

```bash
python scratch/ccfx.py install easedown easegraph chromatic vignette pixelate duotone mirror scanlines shake pulse wave strobe rgbdrift
python -m ccmod_sdk.transitions install
```

Or do it from the [Loader](loader.html), which is the friendlier way.

## 6. (Optional) Build the Loader

```bash
cd bootstrapper/ccmodloader
dotnet build -c Release
```

The exe lands in `bin/Release/net8.0-windows/ccmodloader.exe`. Pin a shortcut to it.

## Verify

```bash
python -m ccmod_sdk list         # your plugins and why any were skipped
python -m pytest                 # the test suite (no CapCut needed)
```

Next: [Quick start](quickstart.html).
