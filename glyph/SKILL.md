---
name: glyph
description: "Use glyph (pmndrs/glyph) for GPU text typography in JavaScript/TypeScript - font loading and offline/runtime baking to .font.glb, HarfBuzz/Wasm shaping, paragraph layout, and batched Slug / MTSDF / bitmap glyph rendering in Three.js, React Three Fiber, Vue/TresJS, TypeGPU, or a custom renderer via the GlyphConfig contract. WHEN: glyph, pmndrs/glyph, @pmndrs/glyph, GlyphConfig, TextGroup, glyph.shape, useMsdf, useBitmap, useSlug, GlyphProvider, GPU text rendering JavaScript, MSDF text, MTSDF text, Slug text, PMNDRS_font, font.glb, text shaping JavaScript, Three.js text, typography rendering."
---

# glyph (pmndrs/glyph)

Renderer-neutral, raster-independent text system for JavaScript/WebGPU: shapes modern Unicode text once (HarfBuzz via HarfRust Wasm), lays it out in application-controlled regions (JS paragraph engine), and renders batched glyph data through interchangeable **Slug**, **MTSDF** (MSDF module's only V0 encoding), or **bitmap** techniques. Three.js, React Three Fiber, Vue/TresJS, and TypeGPU consume the same public core through separate integrations.

**Maturity: pre-alpha `0.1.0`.** ESM-only, published to npm (canary tag from main; `latest` only from version tags). The public API is not frozen — the merged-v0 surface (`/v0`, `/raster/*`, `/react` v0 paths) is being deleted in favor of the v1 `FontLoader → TextGroup → Text` surface. Trust the guide imports in `references/` over older reports that mention `loadFont`, `createFontLibrary`, or `createGlyphEngine` — those are removed.

## Install and peers

```bash
npm install @pmndrs/glyph three        # Three path
npm install @pmndrs/glyph three @react-three/fiber react   # R3F path
```

Peer floors: Three.js `0.185.0`, React `19`, R3F `>=9.7.0 <10 || >=10.0.0-alpha.4 <11`, TypeGPU `0.12.5` (+ `@typegpu/gl` 0.12.4, `@typegpu/three` 0.12.1). TypeGPU/Three/React are optional peers per entry point. Node `^20.19.0 || >=22.12.0` for `/bake` + CLI.

## Mental model

```
font source ──bake──► PMNDRS_font GLB (.font.glb) ──load/validate──► FontFace / Font
Font ──HarfRust Wasm shape──► glyph IDs + UTF-16 clusters + positions (SoA)
    ──JS paragraph engine──► bidi, itemization, line breaks, ellipsis, layout
    ──raster (bitmap | MTSDF | Slug)──► fixed-stride GPU records + textures
    ──Codec / CommandBufferView / DisplayList──► renderer decode → draw
```

- **TypeScript owns** the root runtime, config, font/raster loading, adapters. **Rust/Wasm owns** Unicode analysis, bidi, shaping, line composition, positioning, queries, command-buffer packing. TS never shapes or lays out paragraphs itself.
- **You own** box resolution, constraints, node transforms/clipping, scene attachment, and render-pass submission. glyph never owns canvas/context/scene/`GPUDevice`/render passes.
- Measurement never consults raster artifacts; raster/paint changes rebuild draw batches without invalidating paragraphs.

## Quick start (imperative Three.js)

```ts
import { glyph, msdf, bitmap } from '@pmndrs/glyph';
import { ThreeConfig } from '@pmndrs/glyph/three';

await glyph.init();                                  // idempotent; failure is fatal for the module
const three = glyph.handle('main', ThreeConfig);     // handle owns one anonymous root

const Inter = glyph.fontFace('/fonts/Inter.font.glb', {
  family: 'Inter',
  format: [msdf, bitmap({ strikes: [16, 32] })],     // strikes are static tuples
});
await Inter.load();                                  // all formats in parallel; or Inter.msdf.load()

const label = three.createText({
  font: Inter.msdf,
  text: 'Hello Glyph',
  style: { fontSize: 32 },
  layout: { wrap: 'word' },
});
scene.add(label);

glyph.shape();                                       // one flush of every dirty root
renderer.render(scene, camera);                      // app-owned renderer/camera

label.dispose(); three.dispose(); Inter.dispose();   // text → handle → font order
```

`glyph.fontFace(url, { family, format })` is the **only** font declaration surface. There is no `loadFont` / `FontLibrary`.

## Core vocabulary

| Term | Meaning |
|---|---|
| `FontFace` | One declaration from a source; members `load()`, `isLoaded()`, `dispose()`, per-format selections `Inter.bitmap` / `.msdf` / `.slug` with their own `.load()` / `.clone()` |
| `Text` / `TextGroup` | Three `Object3D` subclasses (also `<Text>`/`<TextGroup>` in React/Vue); `TextGroup` batches descendants and shares a planner; child `renderOrder` = paragraph rank |
| `GlyphConfig` | `defineGlyphConfig({ schema, fonts, encode, resolve, renderer, root, commands? })` — the renderer-neutral contract every integration (including Three) implements |
| Codec | Selected by `encode()`; owns programs, buffer lanes, batching, ordering, transform mode. Never put batching/order in `decode()` |
| `CommandBufferView` | Handed to `renderer.decode(view)`; phases `resources`, `buffers`, `patches`, `retirements`, `displayList`. **Expires when `decode()` returns** — copy scalars, never store it |
| Slug / MTSDF / bitmap strike | Raster techniques: Slug = analytic curves for large/zoomed text (40 B/glyph); MTSDF = linear RGBA8 multi-channel + true distance in alpha; bitmap = native device-pixel strikes declared in ppem |
| detached glyph slice | `Text.breakApart()` → frozen `[Glyphs, Decorations | undefined]` — a renderer-owned *copy* of one committed paragraph |

## API map

- Root runtime: `glyph.init()`, `glyph.handle(name, config)`, `glyph.shape()`, `glyph.fontFace(...)`. Handle: `createText()`, `createTextGroup()`, named roots via `handle('hud')`-style terminal siblings (live names unique, reusable after dispose).
- Text: `set({...})`, `measure()` → `ParagraphLayoutSummary`, `glyphs()`, `withGlyphs(cb)` (sync, borrowed), `breakApart()`, `caretAt(x,y)`, `selectionRects(start,end)`, `visible`, `renderOrder`, `boundingBox`. `text` + `spans` are authored together (assigning `text` without `spans` clears replaced ranges; the old offset helpers were removed).
- React (`@pmndrs/glyph/react`): `GlyphProvider`, `<Text>`, `<TextGroup>`, `useFont` / `useBitmap` / `useMsdf` / `useSlug` (each `.preload()` / `.clear()`).
- Vue (`@pmndrs/glyph/vue`): same components; composables return `{ font, error, ready }`; standalone `preloadFont` / `clearFont` (+ per-format leaves).
- Custom renderer: `defineGlyphSchema` → `encode`/Codec → `resolve`/`resourceLease` → `renderer.decode` → `root.create` via `/core`.
- Details in `references/api.md`; per-framework recipes in `references/integrations.md`; architecture and constraints in `references/architecture.md`.

## Error model

- Font/entry errors **throw** at the operation: malformed payloads throw at read/decode; creating text with an unloaded font selection throws; rejected loads are evicted so an explicit retry can start fresh.
- Render/frame errors are **retained, not thrown**: `Text.error` / `group.error` → `onError` during traversal (traversal never throws); explicit `glyph.shape()` throws at the call site. Renderer failure discards the candidate draw, keeps the last accepted draw live, and does **not** retry unchanged frames.
- `/three` re-raises frame rejections as `TextFrameError` with a discriminated `rejection`: caller-actionable statuses `styleRangeInvalid`, `styleSplitsCluster`, `styleNestingInvalid`, `styleRootInvalid`, `fontStackMissing`, `fontMetricsMissing`; `invalidRequest` = internal invariant (reports nothing).
- `glyph.init()` failure is fatal for that module lifetime — only a page/module replacement retries.

## Pitfalls

1. Order matters: `await glyph.init()` → `glyph.handle()` → loaded font selection → `createText`. `GPUDevice` only before physical resources; never needed for shaping.
2. Dispose **text → group → handle → font → FontFace**. `TextGroup.dispose()` does not dispose children (live children can move to another group). Disposal is idempotent; FinalizationRegistry is a leak net, not a lifecycle.
3. Per-frame: transform/visibility/camera changes never enter Wasm — only content/style/geometry do. Don't call `shape()` for matrix-only updates.
4. In custom renderers: `context.create(...)` exactly once inside `root.create`; never store `CommandBufferView` or its sequences past `decode()`; never deep-compare payloads (equal `RasterResourceId` ⇒ equal bytes); leases are keyed by `RasterResourceId`, not filenames.
5. React `<Text>`/`<TextGroup>` never accept handle/root props — selection is `GlyphProvider`'s (remount to change). Nested `<Text>` accepts only `children`, `font`, `style`, `paint`, `material`; rich text is nested `<Text>` or `txt`/`span` (a raw `spans` field is rejected).
6. Vue provider props `font-faces` / `handle` are immutable after mount (changing throws — remount); one glyph root per `<TresCanvas>`; composables must run inside `<TresCanvas>`.
7. Three: one root per `THREE.Scene` (use distinct named roots); `TextGroup` is not a `groupOrder` boundary (put a real `Group` above it if you need one).

## Stability map

- **Stable-ish**: root runtime, `/three`, `/shaders/tsl`, `/react`, `/core` construction helpers, `/bake` + CLI, raster packages.
- **Experimental**: `/three/typegpu`, `/shaders/typegpu`, direct `/typegpu` (proof-of-concept).
- **Not shipped**: `transformGlyphs`/`clearGlyphTransforms`, public drop-cap/contour authoring, balanced wrapping, hyphenation, vertical writing, color emoji, CJK paging (post-v1).
- Open gates to respect: mixed-bidi resize perf, p95 width gate, MTSDF pinch artifacts, R3F v10 cross-entry cost. Do not make numeric speed/size claims without the project's benchmark harness.

## When writing code for this repo

Conventions from `docs/engineering/code-style.md`: 120-col Oxfmt (semicolons, single quotes, trailing commas); discriminated unions for lifecycle/result states; keep boundary data `unknown` (a cast is not validation); renderer integrations implement public `GlyphConfig` — no second core API; no sleeps/retries/timers as correctness mechanisms in tests; goldens regenerate only with an explained change. Package source changes require a concept + `source_digest` refresh or CI fails.
