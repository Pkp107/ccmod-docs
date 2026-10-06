# Authoring a ccmod pack

Companion to [CCPACK-SPEC.md](CCPACK-SPEC.md). The spec is normative; this is
the practical guide. Everything here is what you need to produce a valid pack
without reading ccmod's source.

If you only read one thing: **`ccmod lint` enforces every rule below.** Run it
before you hand the pack over. `ccmod pack` refuses to build a pack that fails.

---

## What a pack is

```
my-pack/
├── pack.json
└── shader/
    ├── main.frag       required
    └── main.vert       optional — ccmod supplies a default
```

```
ccmod pack my-pack/ -o my-pack.ccpack
```

Transitions only. Single-clip effects have no working loader in CapCut — not a
limitation of ccmod, there is no path to them at all. `kind: "effect"` fails
lint.

## pack.json

```json
{
  "spec": 4,
  "id": "yourname.my-pack",
  "name": "My Pack",
  "kind": "transition",
  "version": "1.0.0",
  "author": "yourname",
  "description": "One line on what it looks like.",
  "requires": { "clips": 2, "progress": true, "passes": 1 }
}
```

`requires` declares intent; ccmod derives the uniform contract from it. You do
not write engine types by hand, which is deliberate — it is how the format stops
you declaring one that does not exist.

---

## The legal uniform set

**This list is closed.** Anything not on it is never bound, reads 0, and your
effect silently renders as nothing.

| Uniform | Declare as | What it gives you |
|---|---|---|
| `progress` | `uniform float progress;` | 0 → 1 across the transition. **The only animated value.** |
| `inputImageTexture` | `uniform sampler2D` | outgoing clip |
| `inputImageTexture2` | `uniform sampler2D` | incoming clip |
| `texCoord` | `varying vec2 texCoord;` | UV from the vertex stage |

### Never use these ❌

| | Why |
|---|---|
| `time` | The engine type behind it (302) has **no implementation at all** — there is no branch for it in the engine's Lua. Not "sometimes zero"; never bound. |
| `strength` | Not an engine concept. Reads 0, so `x * strength` is `0`. This silently turned a flash transition into a plain hard cut. |
| `u_src0`, `u_src1`, `u_progress` | A different donor family's names. They bind on those donors and never on ours. |
| `iTime`, `u_Time`, `u_albedo` | Shadertoy / scene-script conventions. Not bound here. |

**Want a tunable?** Use a `const`. There is no mechanism for a live parameter:

```glsl
const float ZOOM = 0.5;      // yes
uniform float zoom;          // no - never bound, reads 0
```

---

## The rules

### 1. Flip V when you sample

`texCoord` is top-origin; the textures are bottom-origin.

```glsl
vec2 uv = vec2(texCoord.x, 1.0 - texCoord.y);
vec4 a = texture2D(inputImageTexture, uv);
```

This is what `slide-down` does, and `slide-down` is the only pack confirmed to
render. Sampling unflipped renders upside down **and has no other symptom** —
the colours, the timing and the motion all look right.

**Multi-pass exception.** Prior-pass output (engine type 1000) has the opposite
origin to a direct texture (type 103). If you mix them in one shader, flip the
prior-pass source only:

```glsl
// direct texture: flip
vec2 uv = vec2(texCoord.x, 1.0 - texCoord.y);
vec4 direct = texture2D(inputImageTexture, uv);

// prior-pass output in the same shader: already top-origin, do not flip again
vec4 prior = texture2D(priorPassTexture, texCoord);
```

Do not flatten this into one rule. It has been documented backwards in **both**
directions and cost a rebuild of four packs each time.

### 2. Declaring an unused uniform is fine; reading an unbound one is fatal

`slide-down` declares `time` and `strength`, uses neither, and renders. So a
stray declaration will not kill your pack — `ccmod lint` warns rather than
errors.

Reading one *is* fatal, because it reads 0:

```glsl
flash = pow(f, 3.0) * strength;   // -> 0. No flash. Renders as a hard cut.
```

An earlier version of this guide told authors to add a "dummy multiply" to
reference every declared uniform. **Do not do that.** It is exactly how three
packs got silently zeroed.

### 3. GLES 2 only

- `precision highp float;` as the first statement
- `varying`, not `in`/`out`
- `texture2D()`, not `texture()`
- write `gl_FragColor`; no `#version`
- loop bounds must be compile-time constants
- set `gl_FragColor.a = 1.0`

### 4. Drive everything from `progress`

It is the only value that changes per frame. If you want jitter or noise that
evolves, seed it from `progress`:

```glsl
float seed = band + floor(progress * 24.0);   // evolves
float seed = band + floor(time * 24.0);       // frozen: time is never bound
```

---

## The vertex shader

Optional. Omit it and ccmod supplies this:

```glsl
attribute vec4 position;
attribute vec2 inputTextureCoordinate;
varying vec2 texCoord;

void main() {
    gl_Position = position;
    texCoord = inputTextureCoordinate;
}
```

---

## Worked example — the reference pack

This is `custom_transitions/slide_down/shader/main.frag`, copied verbatim. It is
the only pack confirmed to render in CapCut.

```glsl
precision highp float;
varying vec2 texCoord;

uniform float progress;
uniform float time;
uniform float strength;
uniform sampler2D inputImageTexture;
uniform sampler2D inputImageTexture2;

void main() {
    float boundary = 1.0 - progress;

    vec2 sampleUv = vec2(texCoord.x, 1.0 - texCoord.y);

    if (texCoord.y > boundary) {
        vec2 incUv = vec2(texCoord.x, 1.0 - (texCoord.y - boundary));
        gl_FragColor = texture2D(inputImageTexture2, incUv);
    } else {
        gl_FragColor = texture2D(inputImageTexture, sampleUv);
    }
    gl_FragColor.a = 1.0;
}
```

Note it declares `time` and `strength` and reads neither — the warning you will
get for that is informational, and this pack renders.

## Diagnostic pack

`custom_transitions/_smoke` renders solid green with no shader logic.

**Install it first when something does not work.** If green fills the
transition, the pipeline is fine and the problem is your shader. If it does not,
your shader is irrelevant — the donor or the install is at fault, and no amount
of shader debugging will help.

---

## Pre-flight checklist

Exactly what `ccmod lint` checks. Run it; do not check by hand.

```
ccmod lint my-pack/
```

- [ ] `pack.json` has `"spec": 4`, and `id`, `name`, `kind`, `version`
- [ ] `id` is namespaced and lowercase: `yourname.my-pack`
- [ ] `kind` is `"transition"`
- [ ] Every uniform declared is in the legal set
- [ ] No `time`, `strength`, `u_src0`, `iTime`, `u_Time` is **read**
- [ ] Every uniform the `requires` implies is declared
- [ ] `varying vec2 texCoord;` declared
- [ ] `precision` first, `varying` not `in`/`out`, `texture2D()` not `texture()`
- [ ] No `#version`
- [ ] `gl_FragColor.a` set to `1.0`
- [ ] Loop bounds are constants
- [ ] Sampling flips V (single-pass)
- [ ] Offset sampling is clamped, or you accept edge/wrap behaviour

Then check the donor before installing:

```
ccmod validate-donor <effect_id> my-pack/
```

which reports whether that specific donor declares the names your shader uses,
and names what it offers instead if not.

---

## What is still unverified 🔴

Be aware when authoring:

- Only `slide-down` and `smoke-green` have been seen to render. Every other pack
  in this repo is code that looks right.
- Whether an unreferenced **sampler** is safe is unknown. Unreferenced scalars
  are proven safe; samplers are not. Sample what you declare.
- The default vertex shader above is inferred from donor `.vert` files, not
  confirmed against the engine's attribute binding.
