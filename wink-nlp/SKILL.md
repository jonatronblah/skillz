---
name: wink-nlp
description: Use wink-nlp for natural language processing in JavaScript/TypeScript — tokenization, sentence boundaries, named entities, POS tags, lemmas/stems, sentiment, custom entity patterns, BM25 search, and word embeddings — in Node.js or fully in the browser (bundled with wink-eng-lite-web-model). WHEN: wink-nlp, winkjs, wink-nlp in browser, client-side NLP, tokenize text JavaScript, extract entities JS, sentiment analysis JavaScript, BM25 JavaScript, NLP without Python.
---

# wink-nlp

[wink-nlp](https://winkjs.org/wink-nlp/) is a fast, lossless NLP engine for JavaScript/TypeScript: tokenization, sentence boundary detection (SBD), named entity recognition (NER), POS tagging, lemmas/stems, sentiment, custom entity recognition (CER), BM25 vectorizing, and optional word embeddings. It is pure JavaScript (no WASM, no native code), MIT-licensed, has zero runtime dependencies, ships TypeScript definitions, and runs at the same speed in Node.js and the browser (~650k tokens/sec, full pipeline, M1-class laptop).

**Pin these or newer:** `wink-nlp@^2.4.0` and `wink-eng-lite-web-model@^1.8.1`.

## Setup

```bash
npm install wink-nlp wink-eng-lite-web-model --save
```

> Always use **`wink-eng-lite-web-model`** — not `wink-eng-lite-model`. The plain `-model` package is Node-12/14-era and server-side only. The `-web-model` package is the current one and works everywhere: Node ≥ 16 and browsers.

## Quick start

```js
const winkNLP = require('wink-nlp');
const model = require('wink-eng-lite-web-model');
const nlp = winkNLP(model);
const its = nlp.its;  // property selectors
const as = nlp.as;    // collection reducers

const doc = nlp.readDoc('Hello   World🌎! How are you?');

doc.out();                                  // full original text — lossless
doc.sentences().out();                      // ['Hello   World🌎!', 'How are you?']
doc.entities().out(its.detail);             // [{ value: '🌎', type: 'EMOJI' }]
doc.tokens().out();                         // ['Hello','World','🌎','!','How','are','you','?']
doc.tokens().out(its.type, as.freqTable);   // [['word',5],['punctuation',2],['emoji',1]]
```

## Mental model (learn once, apply everywhere)

1. `nlp.readDoc(text)` → a **Document**. Always lossless: `doc.out()` with no args returns the original text, whitespace included.
2. The document exposes **collections**: `doc.sentences()`, `doc.entities()`, `doc.customEntities()`, `doc.tokens()`. Collections chain downward, e.g. `doc.sentences().itemAt(0).tokens()`.
3. Extract with `.out(property, reducer)`:
   - `its.*` selects a property (`its.value`, `its.type`, `its.pos`, `its.lemma`, `its.stem`, `its.sentiment`, `its.detail`, …)
   - `as.*` reduces the output (`as.array` default, `as.set`, `as.bow`, `as.freqTable`, `as.bigrams`, `as.text`, `as.vector`, …)
   - On a collection → array of results; on a single item → a single value.
4. Annotate in place with `item.markup()`, then read the result via `out(its.markedUpText)`.

## Running in the browser

wink-nlp runs fully client-side — no server calls. The official path is **npm + a JS bundler** (Browserify is the documented one; webpack/esbuild/Vite work fine). There is **no official CDN or ESM build**: `import` from a CDN URL is undocumented territory — bundle the npm packages instead.

```bash
npm install wink-nlp wink-eng-lite-web-model --save
npm install -g browserify        # or use webpack / esbuild / Vite
browserify app.js -o bundle.js
```

```html
<div id="result"></div>
<script src="bundle.js" charset="utf-8"></script>
```

```js
// app.js — identical code to Node
const winkNLP = require('wink-nlp');
const model = require('wink-eng-lite-web-model');
const nlp = winkNLP(model);
const its = nlp.its;

const doc = nlp.readDoc('Its quarterly profits jumped 76% to $1.13 billion for the three months to December, from $639million of previous year.');
doc.entities().each((e) => e.markup());
document.getElementById('result').innerHTML = doc.out(its.markedUpText);
```

Browser facts that matter:

- **Bundle size:** the model is ~3.5MB uncompressed, **under 1MB gzipped** — always serve with gzip/brotli and cache aggressively.
- **Pure JS, no WASM.** Supported browsers track the MDN `atob()` compatibility table (the model decodes base64). Runs on low-end smartphone browsers.
- All official examples are CommonJS `require()`. Bundlers handle interop; for TypeScript set `"esModuleInterop": true` (+ `allowSyntheticDefaultImports`) and use default imports.
- For embeddings client-side, use `doc.contextualVectors()` rather than shipping the ~110MB `wink-embeddings-sg-100d` package to browsers.
- Web workers aren't officially documented, but nothing stops you from running the bundle in one to keep the UI responsive on large texts.

Full walkthrough (bundler configs, serving/caching, DOM rendering, XSS caution): read `references/browser-guide.md`.

## Gotchas

- The entry method is **`nlp.readDoc(text)`** — not `read()`.
- **No string output formats exist** (`out('html')`, `out('brat')`, `out('md')` are not things). Use `its.markedUpText` after `.markup()`, or compose output from `its.*` / `as.*`.
- `its.vector` requires embeddings to be loaded; `doc.contextualVectors()` throws without them (and with `lemma: true` unless `pos` is in the pipeline).
- A trimmed pipeline changes behavior: no `sbd` → the whole text is one sentence; `sentiment` relies on `negation`; `lemma` accuracy drops without `pos`; without `ner` / `cer` the corresponding collections are empty.
- `nlp.learnCustomEntities(patterns, config)` must be called **before** `readDoc()`.
- English models only for NER/POS/lemma/sentiment. The tokenizer itself is language-agnostic and lossless — multilingual text still tokenizes correctly.
- `collection.itemAt(n)` is 0-based and returns `undefined` when out of range.
- `collection.map()` requires wink-nlp ≥ 2.4.0.
- The official `its`/`as` docs page lags the source; the tables in `references/api-reference.md` reflect `src/its.js` and `src/as.js` at v2.4.0.

## References

Load on demand — do not read these up front:

| File | Read when… |
|---|---|
| `references/api-reference.md` | You need the full Document/Collection/Item API, the complete `its` / `as` tables, markup details, or TypeScript setup |
| `references/browser-guide.md` | Building the browser bundle, choosing bundler config, serving/caching advice, rendering results in the DOM |
| `references/advanced.md` | Custom pipelines, `learnCustomEntities()` pattern syntax, BM25 vectorizer, similarity utilities, word embeddings |

Official docs: [getting started](https://winkjs.org/wink-nlp/getting-started.html) · [API](https://winkjs.org/wink-nlp/document.html) · [in browsers](https://winkjs.org/wink-nlp/wink-nlp-in-browsers.html) · [examples](https://winkjs.org/examples.html) · [changelog](https://github.com/winkjs/wink-nlp/blob/master/CHANGELOG.md)
