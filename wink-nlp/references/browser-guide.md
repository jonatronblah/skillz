# Running wink-nlp in the browser

Everything here targets `wink-nlp@^2.4.0` + `wink-eng-lite-web-model@^1.8.1`. Official guide: <https://winkjs.org/wink-nlp/wink-nlp-in-browsers.html>.

## What works client-side

- The **full feature set** runs in the browser: tokenization, SBD, NER, POS, lemma/stem, sentiment, custom entities, BM25, contextual vectors. No server round-trips, no WASM, no native code.
- Performance: ~650,000 tokens/sec with the full pipeline on an M1-class machine, same in browser and Node. The docs pitch it as running smoothly on a low-end smartphone's browser.

## What is NOT officially supported

- **CDN `<script>` includes** (unpkg/jsDelivr): there is no official browser-global or ESM build. Do not reach for a CDN URL — bundle the npm packages.
- **ESM `import` from a CDN**: undocumented.
- **Web workers**: not documented officially. It is plain JS, so running the bundle in a worker works in practice (community pattern) — just don't expect docs support.

All official examples use CommonJS `require()`. Any bundler with CJS interop handles it: Browserify (the documented path), webpack, esbuild, Vite, Parcel.

## Recipe — Browserify (verbatim from the official guide)

```bash
npm install wink-nlp wink-eng-lite-web-model --save
npm install -g browserify
```

`token-counter.js`:

```js
const winkNLP = require('wink-nlp');
const model = require('wink-eng-lite-web-model');
const nlp = winkNLP(model);
const its = nlp.its;

const text = 'Its quarterly profits jumped 76% to $1.13 billion for the three months to December, from $639million of previous year.';
const doc = nlp.readDoc(text);

doc.entities().each((e) => e.markup());
document.getElementById('result').innerHTML = doc.out(its.markedUpText);
```

```bash
browserify token-counter.js -o bundle.js
```

```html
<div id="result"></div>
<script src="bundle.js" charset="utf-8"></script>
```

## Recipe — webpack / esbuild / Vite

- **webpack**: zero config; `require('wink-nlp')` works out of the box.
- **esbuild**: `esbuild app.js --bundle --outfile=bundle.js`
- **Vite / TypeScript**: set `"esModuleInterop": true` and `"allowSyntheticDefaultImports": true` in `tsconfig.json`, then `import winkNLP from 'wink-nlp'`. Vite pre-bundles CJS dependencies automatically.

## Serving the bundle

- The model adds ~3.5MB to the bundle — **under 1MB gzipped**. Always serve with gzip/brotli compression.
- The model never changes for a given version: cache aggressively (long `max-age`, ideally a content-hashed filename so deploys bust the cache naturally).
- If NLP isn't needed on first paint, split the NLP bundle into a lazy-loaded chunk so the initial payload stays lean.

## Browser compatibility

Pure JavaScript; the practical floor is the MDN [`atob()` compatibility table](https://developer.mozilla.org/en-US/docs/Web/API/atob#browser_compatibility) (the model decodes base64 payloads) — effectively every browser still receiving updates.

## Rendering results in the DOM

The markup API bridges analysis → HTML. Official pattern (trusted input):

```js
const doc = nlp.readDoc(text);
doc.entities().each((e) => {
  if (e.out(its.type) === 'MONEY') e.markup('<mark class="money">', '</mark>');
});
document.getElementById('result').innerHTML = doc.out(its.markedUpText);
```

⚠️ **XSS caution** (our guidance, not from the official docs): wink-nlp does **not** HTML-escape token values. With untrusted input, injecting `markedUpText` via `innerHTML` lets raw text like `<script>` or `onerror=` through. Prefer building DOM nodes with `textContent`:

```js
// Safe render for untrusted input: extract values, let the DOM escape them
const list = document.createElement('ul');
doc.entities().each((e) => {
  const li = document.createElement('li');
  li.textContent = `${e.out()} (${e.out(its.type)})`;  // safe insertion
  list.appendChild(li);
});
document.getElementById('result').replaceChildren(list);
```

## Embeddings in the browser

- The full embeddings package `wink-embeddings-sg-100d` is ~110MB downloaded (~310MB installed) — **never ship it to browsers**.
- `doc.contextualVectors()` (v2.0+) exists precisely for "pure browser-side applications": it computes corpus-local vectors from your own corpus instead of using pretrained vectors. Caveats: it throws if word vectors aren't loaded, and with `lemma: true` it requires `pos` in the pipeline (behavior as of v2.2.2 — verify against current docs before relying on it).
- Once you have vectors (any source), compare them with `similarity.vector.cosine(vA, vB)` — see `advanced.md`.

## Performance tips

- Trim the pipeline to the task, e.g. `winkNLP(model, ['sbd'])` when you only need sentences/tokens — fewer stages, less work. Check what's active with `doc.pipeConfig()`.
- Create the `nlp` instance once and reuse it across `readDoc()` calls; only `learnCustomEntities()` mutates instance-level state.
- For large documents on the main thread, consider a web worker (see above) to keep the UI responsive.
