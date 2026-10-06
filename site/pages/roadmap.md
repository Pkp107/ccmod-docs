---
title: Roadmap
section: Project
order: 10
---

# Roadmap and what's possible

The goal: **give plugin authors enough control that CapCut can look and behave like a different editor**: a 3D tab with its own engine, new tools and icons, other software wired in, and at least as much control as After Effects.

The full, category-by-category comparison is on the [After Effects gap analysis](after-effects.html) page. This page is the short version.

## Possible today

| Capability | How |
|---|---|
| Single-pass shader effects (colour, warps, glitch, procedural, vignette, ...) | [Making effects](making-effects.html) |
| Two-clip transitions | [Making transitions](making-transitions.html) |
| New buttons, tabs, panels, any QML | `ctx.ui` |
| Restyle CapCut | [Themes](making-themes.html) |
| External programs described in JSON, results imported, commands exposed to AI | [External apps](external-apps.html) |
| AI agents drive CapCut | [MCP](mcp.html) |

## Needs a tool we haven't built yet

| Missing tool | Unlocks | Route letter |
|---|---|---|
| **Multi-pass shaders** (format supports it; none shipped) | Blur, glow, bloom | B |
| **Render agent** (a hook in CapCut's render process) | Echo, trails, feedback, live textures from other programs | H |
| **Timeline API** (read/write tracks, clips, keyframes live) | Expressions, scripting, automation, rigs | K |
| **Texture feed** | Live 3D and particle engines in the preview | 3D |
| **Expression engine** | After Effects-style property expressions | K |
| **Effect and transition creator** in the Loader | Make effects without hand-writing code | U |

Route letters are explained in the [gap analysis](after-effects.html#2-route-legend).

## Not verified yet

MCP with a real client; audio scrub behaviour; `ctx.process` against a real app; the Blender bridge end to end; layout-changing themes; multi-pass shaders; the transitions extras tab.

## Help wanted

- Try the unverified items above and report what you see.
- Effects and transitions (see [Making effects](making-effects.html)).
- Themes, especially full-layout ones (an After Effects-style workspace is the dream).
- Support for CapCut builds newer than 8.9.1.
