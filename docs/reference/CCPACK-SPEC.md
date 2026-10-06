# .ccpack format specification

**Spec version: 4** · Normative. The code follows this document; where they
disagree, this document is the bug report.

Every engine claim below cites a line in
`AmazingAuto_out/lua/SadGE.lua` (read from donor `6724227330190873100`, 258
lines) and is tagged ✅ verified / 🟡 inferred / 🔴 unknown.

---

## 1. Why this exists

Every pack failure in this project has been a *contract* failure, not a logic
failure, and every one of them is silent:

| Symptom | Real cause |
|---|---|
| Black frame | uniform declared but unreferenced — compiler drops it, `setTex` fails, pass dies |
| Effect frozen | shader multiplied by `strength` or `time`, neither of which is bound; reads 0 |
| Nothing renders, install "succeeded" | donor is a GESticker package |
| Nothing renders, donor looks fine | donor names its clips `u_src0`/`u_src1`; our names never bind |
| Image upside down | V-flip wrong; no other symptom |

None of these throw. All of them look like "my shader is wrong". The point of a
spec plus `ccmod lint` is to turn each into a message at build time.

---

## 2. Container

A `.ccpack` is a ZIP with a fixed entry allowlist. No directory discovery.

```
pack.json              required   manifest
shader/main.frag       required   fragment stage
shader/main.vert       optional   vertex stage (see §6)
preview.png            optional   card art
README.md              optional   author notes
```

Anything else is rejected. A `.ccpack` is untrusted input — the format exists so
people can download them.

### Safety rules (normative)

| Rule | Limit |
|---|---|
| Total uncompressed size | ≤ 50 MB |
| Per-entry compression ratio | ≤ 100:1 |
| Absolute paths (`/x`, `C:\x`) | rejected |
| Parent traversal (`..`) | rejected |
| Symlink entries | rejected |
| Entries outside the allowlist | rejected |

### Versioning

`pack.json` carries `"spec": 4` — the *format* version, independent of the
pack's own `"version"`. A loader that does not implement a spec version refuses
the pack and names **both** numbers:

```
Pack 'ccmod.foo' declares spec 5; this ccmod implements spec 4.
```

Schema 1 → 2 → 3 drift silently routed old packs down different code paths for
days. Refusing loudly is the whole point of the field.

---

## 3. The uniform contract ✅

This is the part that prevents broken packs.

**Binding is by `type`, not by name.** `SadGE.lua` reads the donor's
`generalEffect.json` and calls `setFloat`/`setTex`/`setInt` using
`uniform.name` as the key and a value chosen by `uniform.type`:

```lua
-- SadGE.lua, checkOnCallback
elseif t == 103 then
    self.mats[index]:setTex(uniform.name, self.input[uniform.inputTextureIndex])
elseif t == 3007 then
    self.mats[index]:setFloat(uniform.name, GEProtocolList[t])
```

So the *name* is arbitrary — it is whatever the donor's `generalEffect.json`
says. The consequence for pack authors is the important part:

> **A pack's shader must use the names its donor declares.** ccmod replaces only
> the fragment shader, never `generalEffect.json`, so the donor owns the names.

### Complete type table ✅

Verified by enumerating every numeric branch in `SadGE.lua`:
`0, 1, 2, 3, 4, 5, 100, 103, 200–299, 1000, 3007`.

| Type | Bound to | When | Usable in a pack |
|---|---|---|---|
| `3007` | `Amaz.Input.frameTimestamp`, normalized 0→1 | every frame | ✅ **yes — the only animated value** |
| `103` | `self.input[inputTextureIndex]` — `#TransitionInput0` / `1` | every frame | ✅ yes — the two clips |
| `1000` | prior pass's render texture, via `inputEffectIndex` | every frame | ✅ multi-pass only; `passId` must be > 1 |
| `200`–`299` | `GEProtocolList[t]`; 200 = width, 201 = height | **once at start** | ✅ yes, but static |
| `100` | `input[0]` | every frame | 🟡 switches to `input[1]` past progress 0.5 only if `considerTemplate`, which is `false` at module scope — so in practice always the outgoing clip |
| `1` | texture `SyncLoad`ed from `xshader/<data[1]>` | once at start | ✅ yes, needs the asset shipped into the donor |
| `2` / `3` / `4` / `5` | int / float / vec2 / vec3 from `data[1..n]` | **once at start** | ✅ yes, but constant — not animatable |
| `0` | no effect | — | ❌ |

### Explicitly unusable ❌

| Name / type | Why |
|---|---|
| type `302` (`time`) | **Zero occurrences in `SadGE.lua`.** Not merely undriven — there is no branch for it. Unimplemented. |
| type `300` / `301` | `GEProtocolList[300]`/`[301]` are populated with `1/width`, `1/height` at init, but the only branch that reads them is `t >= 200 and t < 300`, which **excludes 300**. Populated and never bound. ⚠️ Newly found while writing this spec; earlier docs implied they worked. |
| `strength` (any type) | Not an engine concept. As a type-3 float it is set once from `data[1]`, and donors do not declare it, so it reads 0. Multiplying by it zeroes the effect. |
| per-frame floats by name | `checkOnCallback` refreshes a type-3 float only if `GEProtocolList[uniform.name]` exists. The sole string key is `"frame"`, set to 0 at init and never updated. So no float is animatable by name. |

### The names our reference donors use ✅

All 13 currently targetable donors declare exactly:

| Purpose | Name | Type | Extra |
|---|---|---|---|
| Outgoing clip | `inputImageTexture` | 103 | `inputTextureIndex: 0` |
| Incoming clip | `inputImageTexture2` | 103 | `inputTextureIndex: 1` |
| Progress | `progress` | 3007 | — |

Two donors (`Comparison`, `Shake`) declare `u_src0`/`u_src1` and a type-3
`u_progress` instead. They are excluded from the picker because no pack written
against the names above can bind to them. See §8.

### Varying

`texCoord`, supplied by the pack's own vertex shader (§6).

---

## 4. Manifest

```json
{
  "spec": 4,
  "id": "ccmod.slide-down",
  "name": "Slide Down",
  "kind": "transition",
  "version": "1.0.0",
  "author": "ccmod",
  "description": "The incoming clip slides down from the top edge.",
  "requires": {
    "clips": 2,
    "progress": true,
    "passes": 1
  },
  "disabled": false,
  "disabled_reason": ""
}
```

The manifest has two jobs and keeps them visibly separate:

1. **ccmod metadata** — `id`, `name`, `version`, `author`, `description`,
   `kind`, `disabled`, `disabled_reason`
2. **Engine intent** — `requires`, from which the uniform contract is derived

### Option B chosen: declare intent, derive the contract

Two designs were on the table:

- **A** — the manifest carries a literal `generalEffect` object, written verbatim.
  Transparent, but the author must get the engine format right by hand, and
  every author repeats the same three uniform declarations.
- **B** — the manifest declares intent (`clips: 2`, `progress: true`) and ccmod
  derives the uniform set.

**B is chosen.** An author cannot mistype `inputTextureIndex` or declare type
302 if they never write types at all, and if the engine format shifts we correct
one function rather than every published pack.

**The tradeoff, stated plainly:** B only pays off fully if ccmod also *writes*
`generalEffect.json` into the donor, which would let it normalise names and make
`Comparison`/`Shake` usable. ccmod does **not** do that today — it replaces the
fragment shader only. So the derived contract is currently used to *validate*
against the donor rather than to overwrite it. Writing `generalEffect.json` is
specified as the next step and is 🔴 untested: `main.scene` and the donor's Lua
are built around the pass list already in that file, and replacing it on a
multi-pass donor may break the pipeline in ways only a render test can reveal.

### `requires`

| Field | Meaning | Derived contract |
|---|---|---|
| `clips` | 1 or 2 | `inputImageTexture` (+ `inputImageTexture2` if 2), type 103 |
| `progress` | bool | `progress`, type 3007 |
| `passes` | int ≥ 1 | > 1 additionally permits type 1000 |

**Nothing in a pack names a donor.** The donor is chosen by the user at install
and recorded in `installed.json`.

---

## 5. Fragment shader rules

Enforced by `ccmod lint`:

1. `precision` declared before any other statement
2. GLES 2 only: `varying` not `in`/`out`, `texture2D()` not `texture()`, no `#version`
3. Writes `gl_FragColor`; `gl_FragColor.a` set to `1.0` on every return path
4. Loop bounds are compile-time constants
5. Every declared uniform is in the contract derived from `requires`
6. No uniform from the unusable list (§3) is **read**

### Unreferenced uniforms — graded, not fatal ⚠️

An earlier version of this rule said any unreferenced uniform is dropped by the
compiler, its bind fails, and the pass renders black — and called it the top
failure mode.

**`slide-down` disproves that for scalars.** It declares `time` and `strength`,
references neither, and renders. Since `slide-down` is normative (§7), the rule
loses. `ccmod lint` grades instead:

| Case | Level | Why |
|---|---|---|
| unusable uniform **read** | ERROR | reads 0; `flash_cut` scaled its flash by `strength` and rendered a plain hard cut |
| unusable uniform **declared only** | WARN | harmless — `slide-down` does exactly this — but misleading |
| unreferenced `sampler2D` | WARN | the failing call would be `setTex`; suspected cause of dead passes, 🔴 unproven |
| unreferenced scalar | INFO | proven harmless by `slide-down` |

### V-flip — conditional, and wrong in both directions historically

- Donor uniformly type 103 → **flip**: `vec2(texCoord.x, 1.0 - texCoord.y)`
- Mixing type 1000 (prior-pass output) with type 103 → the two have opposite
  origin; flip the type-1000 source only

`slide-down`, the only pack confirmed to render, flips
(`slide_down.frag:20`). An earlier version of `AUTHORING.md` claimed the
opposite and four packs were written against it.

---

## 6. Vertex shader

Packs ship `shader/main.vert`. If omitted, ccmod supplies the reference vertex
stage, which passes `texCoord` straight through:

```glsl
attribute vec4 position;
attribute vec2 inputTextureCoordinate;
varying vec2 texCoord;

void main() {
    gl_Position = position;
    texCoord = inputTextureCoordinate;
}
```

🟡 Inferred from the donor `.vert` files alongside each `.frag`; not
independently confirmed against the engine's attribute binding.

---

## 7. Reference packs

| Pack | Role |
|---|---|
| `slide-down` | **Normative example.** The only pack confirmed to render in CapCut. Every rule here is satisfied by it; if a rule and this pack disagree, the rule is wrong. |
| `smoke-green` | **Diagnostic.** Solid green, correct plumbing, no shader logic. If it does not render on a donor, the donor or the pipeline is at fault and no shader question about that donor is meaningful. |

Both are fixtures in the test suite.

---

## 8. Donor compatibility (checked at install, not build)

A pack's derived contract must be satisfied by the donor's
`generalEffect.json`: the same uniform **names**, with the same **types**.

`ccmod validate-donor <effect_id> <pack>` reports this, naming exactly which
uniforms differ. This is the check that would have caught `Comparison` before an
install rendered nothing.

Additionally the donor must be `DonorFormat.GE_PROTOCOL` — it ships
`xshader/generalEffect.json` and `SadGE.lua`. GESticker packages ship the
protocol and declare the right uniforms but never render, so they are excluded
by format regardless of contract. See `docs/formats.md`.

---

## 9. Unresolved 🔴

- Whether writing a generated `generalEffect.json` into a donor works (§4)
- Whether the reference vertex stage matches the engine's attribute binding (§6)
- Whether donor `Blur` (`6916426617455645186`) renders — it ships the protocol
  under an `AmazingFeature/` wrapper rather than `AmazingAuto_out/`
- What types `0` and the 202–299 range do in practice; only 200/201 are
  populated by `GEProtocolList.init`
