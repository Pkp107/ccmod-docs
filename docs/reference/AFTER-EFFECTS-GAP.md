# After Effects vs CapCut vs CCMOD: what is possible, and what we still have to build

Goal: CCMOD should let someone turn CapCut into a different editing application: new tabs, buttons and icons, whole new looks,
their own engines (3D, particles, compositing), other software wired in (Blender, ffmpeg, ML tools), and at least as much control
as After Effects. This page compares After Effects (AE) with CapCut, decides for every AE feature whether CCMOD can reach it with
what we hold today, and where it cannot, names the missing tool so we can build it.

How to read it: the AE effect list is from my knowledge of AE 2024/2025 (Adobe's help site refused the fetch, 403), and the CapCut
column is from the live editor and general knowledge of CapCut Desktop 8.x; "~" means I am not sure. The route letters are
**predictions**, except where marked "proven".

## 1. What CCMOD holds today

| Lever | Status |
|---|---|
| **Shader effects**: our own GLSL effect packages with sliders, native in CapCut's Effects grid, rendered in preview and export | proven |
| **Transition packages**: our own two-clip shaders in the Transitions grid | proven |
| **Any new UI**: inject QML anywhere in the editor (tabs, panels, buttons, overlays), with clicks calling back into Python | proven (`ctx.ui.add_button`, injected tab/panel) |
| **Restyle CapCut**: recolour and restyle the live UI, reversible (`themes/`) | proven (colours, corner radius); layout moves/hides are possible in the same engine but not yet shown on screen |
| **Call CapCut's own controllers**: seek, tabs, import, 90+ view models | proven for seek/tabs/import |
| **Media import**: put any file into CapCut's media pool from Python | proven (`ctx.media.import_file`) |
| **Run any external program** and take its output back (`ctx.process.which/run`) | built; unit-tested only |
| **Serve CapCut's panels from local data** (our categories in Effects, Transitions; the same route fits Filters, Text, Canvas, ...) | proven for Effects and Transitions |
| **Edit projects as data** (draft JSON: tracks, segments, keyframes, materials) with backups | proven for effect segments; general edits are plain JSON but need a reopen |
| **Hook the renderer** (rewrite shaders, override uniforms per frame) in CapCut's render process | proven in experiments (shader rewrite, uniform override); not packaged as an SDK feature |
| **MCP**: AI agents can drive all of the above | built; handshake tested, not yet with a real client |
| **Plugins** with permissions, a tool server, a Loader to manage them | built |

## 2. Route legend

| Letter | Meaning | Can we do it now? |
|---|---|---|
| **A** | One-pass shader effect (colour maths, warps, procedural generators, wipes) | yes |
| **B** | Multi-pass shader effect (blur, glow, bloom, many keyers). The package format lists several passes; we have not yet shipped one | very likely, needs a first proof |
| **H** | Needs earlier frames or feedback (echo, trails, time displacement, simulation state). Needs a **render agent** in CapCut's render process | not yet; tool to build (see 5) |
| **X** | External computation, result imported back (ML, optical flow, denoise, 3D render, stabilisation analysis) | yes, with a program installed; quality is the program's |
| **K** | Write keyframes, tracks or markers into the project | yes offline; live editing needs a **timeline API** (see 5) |
| **U** | Pure UI/tool: a panel, a button, a data editor | yes |
| **3D** | Needs real 3D (cameras, lights, depth, shadows) | external renderer today (X); live 3D needs a **texture feed** (see 5) |
| **N** | Not reachable (CapCut's engine limits, or Adobe-proprietary) | no, with the reason |

## 3. After Effects effects, category by category

CapCut column: **yes** = CapCut has a comparable built-in, **part** = partly, **no** = nothing comparable.

### 3D Channel (7 effects)
3D Channel Extract, Depth Matte, Depth of Field, EXtractoR, Fog 3D, ID Matte, IDentifier. CapCut: **no** (no depth or ID channels).
Route: **X** by estimating depth with an ML model and importing a depth matte; then Depth of Field / Fog become **A** shaders that read
the matte. EXtractoR/ID Matte need multi-channel EXR: **N** for CapCut's media path.

### Audio (11)
Backwards, Bass & Treble, Delay, Flange & Chorus, High-Low Pass, Modulator, Parametric EQ, Phaser, Reverb, Stereo Mixer, Tone.
CapCut: **part** (EQ-like presets, noise reduction, voice changer, fade, reverse). Route: **X** with ffmpeg audio filters, result imported
and swapped on the timeline. A true live audio effect chain needs a hook in CapCut's audio engine, which we have not looked at.

### Blur & Sharpen (18)
Bilateral, Box, Camera Lens Blur, Camera-Shake Deblur, CC Cross/Radial/Radial Fast/Vector Blur, Channel Blur, Compound Blur, Directional,
Fast Box, Gaussian, Radial, Reduce Interlace Flicker, Sharpen, Smart Blur, Unsharp Mask.
CapCut: **yes** for blur/sharpen families, **no** for compound/vector/channel blur. Route: Directional, Radial, Box, Sharpen, Unsharp,
Compound/Channel Blur **A/B**; large Gaussian and Lens Blur **B**; Camera-Shake Deblur **X** (deconvolution tools).

### Channel (13)
Arithmetic, Blend, Calculations, CC Composite, Channel Combiner, Compound Arithmetic, Invert, Minimax, Remove Color Matting, Set Channels,
Set Matte, Shift Channels, Solid Composite. CapCut: **part** (blend modes, invert via filters). Route: **A** (all are per-pixel maths;
Set Matte needs a second layer as input, which transition-style packages already get as a second texture; for normal effects we need to
prove a second input).

### Color Correction (36)
Auto Color/Contrast/Levels, Black & White, Brightness & Contrast, Broadcast Colors, CC Color Neutralizer/Offset/Kernel/Toner,
Change Color, Change to Color, Channel Mixer, Color Balance (+HLS), Color Link, Color Stabilizer, Colorama, Curves, Equalize, Exposure,
Gamma/Pedestal/Gain, Hue/Saturation, Leave Color, Levels (+Individual), Lumetri Color, Photo Filter, PS Arbitrary Map, Selective Color,
Shadow/Highlight, Tint, Tritone, Vibrance. CapCut: **yes** for most (curves, HSL, colour wheels, LUTs, exposure, vibrance, tint),
**no** for Colorama, Color Stabilizer, Equalize, Arbitrary Map. Route: nearly all **A** (Auto-* need image statistics: **B** with a
reduction pass, or **K** by analysing in Python). LUT-based looks: **A** with a LUT texture shipped in the package.

### Distort (36)
Bezier/Mesh Warp, Bulge, CC Bend It/Bender/Blobbylize/Flo Motion/Griddler/Lens/Page Turn/Power Pin/Ripple Pulse/Slant/Smear/Split/Split 2/Tiler,
Corner Pin, Displacement Map, Liquify, Magnify, Mirror, Offset, Optics Compensation, Polar Coordinates, Reshape, Ripple, Rolling Shutter Repair,
Smear, Spherize, Transform, Turbulent Displace, Twirl, Warp Stabilizer, Wave Warp.
CapCut: **part** (zoom/shake/glitch/fisheye/mirror effects, transform, masks; no mesh warp, liquify, corner pin as an effect).
Route: most are **A** (UV remapping). Mesh/Bezier Warp and Liquify need an interactive tool: **U** plus an **A** shader driven by a
displacement texture. Warp Stabilizer and Rolling Shutter Repair: **X** (analysis) then **K** (keyframes). Corner Pin: **K** with the
tracker output.

### Expression Controls (8)
Angle, Checkbox, Color, Dropdown Menu, Layer, Point, Slider, 3D Point. CapCut: **no**. Route: **U** plus the **expression engine** (see 5).
Our effect sliders already cover Slider/Checkbox/Color-style controls; a dropdown and a point picker are inspector widgets we can inject.

### Generate (25)
4-Color Gradient, Advanced Lightning, Audio Spectrum, Audio Waveform, Beam, CC Glue Gun, CC Light Burst 2.5, CC Light Rays, CC Light Sweep,
CC Threads, Cell Pattern, Checkerboard, Circle, Ellipse, Eyedropper Fill, Fill, Fractal, Gradient Ramp, Grid, Lens Flare, Paint Bucket,
Radio Waves, Scribble, Stroke, Vegas, Write-on. CapCut: **part** (light/flare/glow effects, gradients via backgrounds; no procedural
generators as such). Route: gradients, grid, checkerboard, circle, ellipse, cell pattern, fractal, lens flare, light rays/burst/sweep, beam,
radio waves **A/B**. Audio Spectrum/Waveform: **X/U** (analyse audio in Python, feed values as uniforms, or render a clip). Lightning,
Scribble, Vegas, Write-on, Stroke (path based): **A** with an SDF, or **X** through a vector renderer.

### Immersive Video (11)
VR Blur, VR Chromatic Aberrations, VR Color Gradients, VR Converter, VR De-Noise, VR Digital Glitch, VR Fractal Noise, VR Glow,
VR Plane to Sphere, VR Rotate Sphere, VR Sharpen. CapCut: **no** (no 360 workflow). Route: **A/B** (equirectangular UV maths).

### Keying (11) and Matte (4)
CC Simple Wire Removal, Color Difference Key, Color Key, Color Range, Difference Matte, Extract, Inner/Outer Key, Keylight, Linear Color Key,
Luma Key, Spill Suppressor; Matte Choker, Refine Hard Matte, Refine Soft Matte, Simple Choker.
CapCut: **part** (chroma key, smart cutout, masks). Route: colour/luma/difference/range keys **A**; Keylight-class keying with spill
suppression and matte refinement **B**; AI roto and portrait matting **X** (ML), imported as a matte clip or applied through an alpha
shader.

### Noise & Grain (11)
Add Grain, Dust & Scratches, Fractal Noise, Match Grain, Median, Noise, Noise Alpha, Noise HLS, Noise HLS Auto, Remove Grain, Turbulent Noise.
CapCut: **part** (grain/noise effects, video denoise). Route: Add Grain, Noise*, Fractal/Turbulent Noise **A**; Median **B**;
Dust & Scratches **A/B**; Remove Grain / Match Grain **X** (ffmpeg `nlmeans`, ML denoisers).

### Perspective (9)
Bevel Alpha, Bevel Edges, CC Cylinder, CC Environment, CC Sphere, CC Spotlight, Drop Shadow, Radial Shadow, (3D Camera Tracker).
CapCut: **part** (drop shadow on text/stickers, 3D zoom styles). Route: Bevel, Drop/Radial Shadow **B** (blur plus offset);
CC Cylinder/Sphere/Environment/Spotlight **3D** (needs a real 3D surface: **A** can fake spheres/cylinders with UV maths; true
environment mapping needs the texture feed). Camera tracker: **X** + **K**.

### Simulation (18)
Caustics, Card Dance, CC Ball Action, CC Bubbles, CC Drizzle, CC Hair, CC Mr. Mercury, CC Particle Systems II, CC Particle World,
CC Pixel Polly, CC Rainfall, CC Scatterize, CC Snowfall, CC Star Burst, Foam, Particle Playground, Shatter, Wave World.
CapCut: **part** (snow/rain/particle presets, no simulation). Route: Rain/Snow/Bubbles/Star Burst/Scatterize/Pixel Polly **A** (stateless
particles computed from time and a hash); Card Dance, Ball Action, Shatter **A/3D**; Caustics **A**; Foam, Wave World, Hair, Mr. Mercury,
Particle World/Playground (true state) **H** (render agent keeps state between frames) or **X** (bake in Blender/another simulator and
import).

### Stylize (25)
Brush Strokes, Cartoon, CC Block Load, CC Burn Film, CC Glass, CC HexTile, CC Kaleida, CC Mr. Smoothie, CC Plastic, CC RepeTile,
CC Threshold, CC Threshold RGB, CC Vignette, Color Emboss, Emboss, Find Edges, Glow, Mosaic, Motion Tile, Posterize, Roughen Edges,
Scatter, Strobe Light, Texturize, Threshold. CapCut: **yes** for most looks (glow, mosaic, posterize, vignette, edge, strobe, emboss).
Route: **A** for nearly all; Glow, Cartoon, Brush Strokes **B**; Roughen Edges, Texturize **A** (+ texture in package).

### Text (2) and Time (7)
Numbers, Timecode (**A** with a digit atlas, or **U**). Time: CC Force Motion Blur, CC Wide Time, Echo, Posterize Time, Time Difference,
Time Displacement, Timewarp. CapCut: **part** (speed curves, optical-flow slow motion, motion blur effect, freeze frame).
Route: Posterize Time **A** (quantise the time uniform); Echo, Time Difference, Time Displacement, Wide Time, Force Motion Blur **H**;
Timewarp **X** (RIFE / ffmpeg minterpolate) or the built-in optical flow; CC Wide Time/Force Motion Blur also **K** (render sub-frames).

### Transition (7)
Block Dissolve, Card Wipe, Gradient Wipe, Iris Wipe, Linear Wipe, Radial Wipe, Venetian Blinds. CapCut: **yes**. Route: **A** (transition
packages; proven).

### Utility (6)
Apply Color LUT, Cineon Converter, Color Profile Converter, Grow Bounds, HDR Compander, HDR Highlight Compression.
CapCut: **part** (LUT import). Route: LUT/Cineon **A**; the HDR and colour-profile ones are **N** inside CapCut's own colour pipeline
(we can add tone-mapping shaders, but not change the pipeline's bit depth or colour management).

## 4. After Effects features that are not effects

| AE feature | CapCut | Route and what is needed |
|---|---|---|
| Layers: solids, shape, text, null, adjustment | solids/text/adjustment layers yes; no shape or null layers | Shape layers: **A** (SDF shapes with sliders) or **X** (vector renderer → clip); null/parenting: **K** + **U** (a plugin that moves children from a parent's keyframes) |
| Layer parenting and hierarchy | no | **K**: bake child transforms from the parent; live version needs the **timeline API** |
| Expressions (per-frame scripted properties) | no | **Expression engine** (see 5): bake to keyframes (**K**, works now) or evaluate per frame in the render agent (live) |
| Keyframes with graph editor (bezier) | keyframes with presets, speed curves | **U**: a graph editor panel writing keyframes through **K** |
| 3D layers, cameras, lights, shadows | no (2D with 3D-ish zoom styles) | **3D**: Blender-style external render now; live 3D through the texture feed |
| Shape-layer operators (repeater, trim paths, offset) | no | **A/X** |
| Text animators, per-character animation, range selectors | text animations and templates, no per-character ranges | **K** (generate one text segment per character) or **A** with a glyph atlas texture; **X** via a text renderer |
| Masks and mask modes, mask feather, mask expansion | yes (shapes, brush, feather, keyframes) | covered; extras (mask modes, expansion) **A** |
| Roto Brush / AI matte | smart cutout | **X** (ML segmentation → matte clip) |
| Motion tracking, planar tracker (Mocha), 3D camera tracker | basic tracking | **X** (OpenCV / COLMAP) then **K** keyframes |
| Puppet pin, mesh deform | no | **A** (displacement texture) + **U** (pin editor) |
| Content-Aware Fill | no | **X** (ML inpainting) |
| Layer styles (drop shadow, glow, stroke, bevel) | partial | **A/B** |
| Blend modes (~38) | ~15 | **A**: extra modes as shader effects on an adjustment layer; true per-layer modes need engine access (**H** territory) |
| Precompose / nesting | compound clips | covered |
| Time remapping | speed curves | covered |
| Motion blur (per layer) | effect only | **H/K**: sub-frame accumulation |
| Render queue, templates, Media Encoder | export dialog | **U** + **X**: queue several exports through ffmpeg, presets |
| Scripting (ExtendScript), CEP/UXP panels | none | the SDK: Python plugins, QML panels, tools, MCP |
| Motion Graphics Templates / Essential Graphics | text templates only | **U**: parameter panels backed by shader effects and **K** |
| Dynamic Link, .aep import | no | **N** (Adobe-proprietary); a partial .aep reader is a separate research task |
| 32-bit float, ACES/OCIO colour management, EXR | 8-bit pipeline (as far as we know) | **N** for CapCut's own pipeline; **X** pipelines can work in float outside and import 8-bit/10-bit results |

## 5. The tools we still have to build, in order of leverage

These are what turn the **H**, **K**, **3D** and **U** rows from "predicted" into "real". Each is a general platform feature, not a one-off.

1. **Render agent** (H, 3D, expressions live). A small program injected into CapCut's *render* process (not the UI one). Today we can
   rewrite shaders and override uniforms there; the agent packages that into: (a) per-effect frame history (ring buffer of textures for
   echo/trails/feedback), (b) extra render targets for multi-pass chains, (c) an input for **external textures** (shared memory from
   Blender, a game engine or Python), and (d) per-frame uniform values from a script. This is the single biggest unlock.
2. **Timeline API** (K live). A small, stable Python API over CapCut's timeline objects: list tracks and segments, read/write
   keyframes, add clips/effects/markers at a time, select, split. Today this is draft-file editing plus controller calls; the API
   hides both behind one interface and does safe live edits where the controllers allow them.
3. **Expression engine**. Properties driven by small scripts (`time`, `index`, parent values, audio level) with two backends:
   *bake* (evaluate in Python, write keyframes) and *live* (render agent uniforms).
4. **App integration framework**. Generalise what `blender-bridge` does: declare an external app (how to find it, command templates,
   input and output types), run it, import the result with `ctx.media.import_file`. Blender is *not installed* on this PC, so that plugin
   reports "Blender not found" and does nothing; the same framework serves ffmpeg, ML tools, Krita, Natron, GIMP, anything with a CLI.
5. **UI toolkit**. Ready-made QML components (panels, tabs, sliders, dropdowns, graph editor, node editor, colour pickers) that match
   CapCut's look, plus workspaces (saved layouts) and a docked-panel helper using CapCut's own docking system.
6. **Multi-pass and multi-input effect builder** (B). First prove one multi-pass package (blur then composite), then a Python builder
   that emits multi-pass packages from a description. After that, a **ReShade/shadertoy importer** is mostly translation.
7. **Audio bridge**. Hooks for the audio engine (live effects) or at least an ffmpeg-based offline audio pipeline with replace-on-timeline.
8. **Creator tools**. The "no typing code" editors: effect and transition creators, a node-based shader graph, theme editor with live
   preview, panel designer.

## 6. Honest limits

* Everything that needs CapCut's **own pipeline** to change (bit depth, colour management, true per-layer blend modes in its compositor,
  its audio engine) is only reachable through hooks we have not built, or not at all.
* **Live 3D** inside the preview is only a plan until the render agent exists. Until then 3D is "render elsewhere, import the clip".
* **Adobe formats and services** (Dynamic Link, .aep, Adobe Fonts, Creative Cloud libraries) are out of scope.
* CCMOD never touches the account, Pro/subscription state, the watermark or licensing; that stays true for every item above.
