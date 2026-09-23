# wink-nlp API reference

Complete API surface for wink-nlp 2.4.x. Compiled from the official docs plus the library source (`src/its.js`, `src/as.js` at v2.4.0) — the official its/as helper page lags behind the source, so treat these tables as current.

## Instantiation

```js
const winkNLP = require('wink-nlp');
const model = require('wink-eng-lite-web-model');

// Default: full pipeline — tokenization, sbd, negation, sentiment, ner, pos, cer
const nlp = winkNLP(model);

// Custom pipeline — 2nd arg is an array of stage names
const nlp = winkNLP(model, ['sbd', 'pos']);

// With word embeddings — 3rd arg (see advanced.md)
const nlp = winkNLP(model, ['sbd'], vectors);

const its = nlp.its;  // property selectors
const as = nlp.as;    // collection reducers
```

## Document

| API | Returns | Notes |
|---|---|---|
| `nlp.readDoc(text)` | `Document` | Entry point. Lossless — original text is always recoverable. |
| `nlp.learnCustomEntities(patterns, config)` | `number` | Must run **before** `readDoc()`. See `advanced.md`. |
| `doc.sentences()` | Collection | |
| `doc.entities()` | Collection | |
| `doc.customEntities()` | Collection | |
| `doc.tokens()` | Collection | |
| `doc.out([its.x][, as.y])` | string / array | No args → detokenized original text, whitespace included. |
| `doc.pipeConfig()` | array | Active pipeline stages. |

## Collection (sentences / entities / customEntities / tokens)

| API | Returns | Notes |
|---|---|---|
| `.itemAt(n)` | Item \| `undefined` | 0-based index. |
| `.filter(cb)` | Collection | New collection; `cb` receives each item. |
| `.each(cb)` | — | Iterate items. |
| `.map(cb)` | array | v2.4.0+. e.g. `doc.tokens().map(t => ({ token: t.out(), pos: t.out(its.pos) }))` |
| `.length()` | number | |
| `.out([its.x][, as.y])` | array | Default: `its.value` as `as.array`. |

## Item (sentence / entity / token)

| API | Returns | Notes |
|---|---|---|
| `.out([its.x])` | value | Default `its.value`. |
| `.index()` | number | Index within the parent document. |
| `.markup(beginTag?, endTag?)` | — | Defaults `<mark>` / `</mark>`. Any tag pair works — HTML, ANSI colors, symbols. |
| `.parentDocument()` | Document | |
| `.parentSentence()` | Item \| `undefined` | Token and entity. |
| `.parentEntity()` | Item \| `undefined` | Token only. |

### Chaining

Collections chain downward across levels:

```js
// Sentence 0 → its first entity → that entity's tokens
doc.sentences().itemAt(0).entities().itemAt(0).tokens().out();
// -> ['July', '20', ',', '1969']

// Content words of sentence 3
doc.sentences().itemAt(3).tokens()
  .filter((t) => t.out(its.type) === 'word' && !t.out(its.stopWordFlag))
  .out();
// -> ['story', 'movie', 'spans']
```

## `its` — property selectors

| Helper | Applies to | Returns |
|---|---|---|
| `its.value` (default) | doc, sentence, entity, token | string |
| `its.normal` | doc, sentence, entity, token | lowercased, whitespace collapsed to single spaces, British→American spelling |
| `its.type` | entity, token | entity type (`DATE`, `MONEY`, `EMOJI`, …) or token type (`word`, `punctuation`, `emoji`, `number`, …) |
| `its.detail` | entity | `{ value, type }` |
| `its.span` | doc, sentence, entity | `[firstTokenIdx, lastTokenIdx]` |
| `its.pos` | token | UPOS tag: `NOUN`, `VERB`, `AUX`, `PROPN`, … |
| `its.lemma` | token | lemma (accuracy depends on `pos` being in the pipeline) |
| `its.stem` | token | Porter Stemmer v2 |
| `its.shape` | token | `Xxxxx`-style shape; digits → `d`; runs trimmed after 4 identical chars |
| `its.case` | token | `lowerCase` \| `upperCase` \| `titleCase` \| `other` |
| `its.stopWordFlag` | token | boolean |
| `its.negationFlag` | token | boolean |
| `its.abbrevFlag` | token | boolean |
| `its.contractionFlag` | token | boolean |
| `its.precedingSpaces` | token | string (this is how the tokenizer stays lossless) |
| `its.prefix` | token | string |
| `its.suffix` | token | string |
| `its.uniqueId` | token | number |
| `its.sentiment` | doc, sentence | number in [-1, +1] |
| `its.readabilityStats` | doc | object — Flesch reading ease (`fres`), complex words, reading time, sentiment |
| `its.sentenceWiseImportance` | doc | `[{ index, importance }]`, importance 0–1 (v1.14+) |
| `its.markedUpText` | doc, sentence | string with markup after `.markup()` calls |
| `its.vector` | token | 100-dim vector — **requires embeddings loaded** |

BM25-only helpers (`its.terms`, `its.docTermMatrix`, `its.docBOWArray`, `its.bow`, `its.idf`, `its.tf`, `its.modelJSON`) are covered in `advanced.md`.

> These do **not** exist — common hallucinations: `its.index` (use the `.index()` method), `its.hash`, `its.markup` (use `its.markedUpText`).

## `as` — collection reducers

| Reducer | Output |
|---|---|
| `as.array` (default) | array of values |
| `as.set` | `Set` of values |
| `as.unique` | array of unique values |
| `as.bow` | bag-of-words object `{ value: count }` |
| `as.freqTable` | `[[value, count], …]` sorted by frequency |
| `as.bigrams` | array of bigram pairs |
| `as.text` | plain-text output of the collection |
| `as.markedUpText` | marked-up text output (after `.markup()` calls) |
| `as.vector` | v2.0+. Averages token vectors into one embedding. Combine with `.filter()` to drop stop words first. Requires embeddings. |

> `as.text`, `as.markedUpText`, `as.vector` are missing from the official its/as docs page but exist in `src/as.js` at v2.4.0.

## Markup

```js
// Highlight every number with the default <mark> tags
doc.tokens().each((t) => {
  if (t.out(its.type) === 'number') t.markup();
});
doc.out(its.markedUpText);

// Custom tags — HTML classes, ANSI codes, or plain symbols
entity.markup('<span class="money">', '</span>');
```

Read the result with `out(its.markedUpText)` on the document, a sentence, or a sentence collection. Markup is the bridge to HTML rendering — see `browser-guide.md` for DOM usage and the XSS caution.

## TypeScript

Both packages ship types. In `tsconfig.json`:

```json
{ "compilerOptions": { "esModuleInterop": true, "allowSyntheticDefaultImports": true } }
```

```ts
import winkNLP from 'wink-nlp';
import model from 'wink-eng-lite-web-model';

const nlp = winkNLP(model);
const doc = nlp.readDoc('Hello world!');
```

## Language support

The official models are English (`wink-eng-lite-web-model`; legacy `wink-eng-lite-model`). The tokenizer is language-agnostic and lossless — e.g. `"¡Hola! नमस्कार! Hi! Bonjour chéri"` tokenizes fine — but model-driven features (NER, POS, lemma, sentiment) are English-only. Third-party models can plug in via the model interface.
