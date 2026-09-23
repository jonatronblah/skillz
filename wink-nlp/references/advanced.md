# Advanced: pipelines, custom entities, BM25, similarity, embeddings

## Processing pipeline

Stages run in this order when enabled: `tokenization` (always on) → `sbd` → `negation` → `sentiment` → `ner` → `pos` → `cer`.

The default instance runs all stages. Pass a custom pipe as the 2nd constructor arg:

```js
const nlp = winkNLP(model, ['sbd', 'pos']);
// ...
doc.pipeConfig();   // -> ['sbd', 'pos'] — inspect what's active
```

Interplay rules (what breaks when you trim):

- No `sbd` → the entire text is treated as one sentence.
- `sentiment` depends on `negation` — include both if you want sentiment.
- `lemma` accuracy drops without `pos` in the pipe.
- No `ner` / `cer` → `doc.entities()` / `doc.customEntities()` come back empty.
- No `pos` → `its.pos` and `its.lemma` lose accuracy; `contextualVectors({ lemma: true })` throws.

## Custom entities (CER)

Define your own entity patterns and detect them with the same API surface as built-in entities:

```js
const patterns = [
  // '[|DET]' = optional, alternatives inside brackets, no spaces inside [..]
  { name: 'nounPhrase', patterns: ['[|DET] [|ADJ] [NOUN|PROPN]'] },

  // Mark only a sub-span of the match: 0-based; negative indexes count from the end
  { name: 'adjectiveAnimalPair', patterns: ['[ADJ] [cats|dogs]'], mark: [0, 0] },

  // Reuse built-in entity types (UPPERCASE)
  { name: 'Qty', patterns: ['CARDINAL'] },
];

nlp.learnCustomEntities(patterns, { matchValue: false, usePOS: true, useEntity: true });

const doc = nlp.readDoc('The grumpy cats slept.');
doc.customEntities().out(its.detail);
// -> [{ value: 'grumpy cats', type: 'adjectiveAnimalPair' }, { value: 'The grumpy cats', type: 'nounPhrase' }] (illustrative)
```

Rules:

- Call `learnCustomEntities()` **before** `readDoc()` — it configures how subsequent documents are processed.
- Matching is greedy with longest-preference. Match precedence: entity type → token value → POS.
- Pattern syntax: space-separated token matchers. `[a|b]` = alternatives (no spaces inside the brackets). POS tags and built-in entity types are UPPERCASE (`NOUN`, `DET`, `CARDINAL`). Anything else matches the token value literally. `^` is the escape character (`^^` matches a literal `^`).
- Config defaults: `{ matchValue: false, usePOS: true, useEntity: true }`. (The docs prose around `matchValue` is self-contradictory; these defaults are the verified ones.)

## BM25 vectorizer (search / ranking)

```js
const BM25Vectorizer = require('wink-nlp/utilities/bm25-vectorizer');
const bm25 = BM25Vectorizer();   // config: { k1: 1.2, b: 0.75, k: 1, norm: 'none' } — norm: 'none' | 'l1' | 'l2'

// Learn from tokenized documents — one learn() call per document
corpus.forEach((text) => bm25.learn(nlp.readDoc(text).tokens().out()));

// Inspect / export
bm25.out(its.terms);             // vocabulary
bm25.out(its.idf);               // idf per term
bm25.out(its.docTermMatrix);     // doc × term matrix
bm25.out(its.docBOWArray);       // per-doc bag of words
bm25.out(its.modelJSON);         // serialized model — persist this

// Vectorize a query (tokens as produced by wink-nlp)
bm25.vectorOf(nlp.readDoc(query).tokens().out());
bm25.bowOf(tokens, false);       // processOOV=false → OOV tokens dropped

// Restore a saved model
const bm25Restored = BM25Vectorizer();
bm25Restored.loadModel(savedJson);

// Access learned document #n
bm25.doc(n);
```

## Similarity utilities

```js
const similarity = require('wink-nlp/utilities/similarity.js');

similarity.bow.cosine(bowA, bowB);        // cosine between bag-of-words objects
similarity.set.tversky(setA, setB);       // α=β=0.5 by default → Sørensen–Dice
similarity.set.tversky(setA, setB, 1, 1); // α=β=1 → Jaccard
similarity.set.oo(setA, setB);            // set overlap measure
similarity.vector.cosine(vA, vB);         // v2.1+ — for token/doc vectors
```

All similarity scores are in the 0–1 range. There is no dedicated Sørensen–Dice function — derive it from `tversky` with the default parameters as shown.

## Word embeddings (optional, server-side)

```bash
npm install wink-embeddings-sg-100d --save
```

```js
const winkNLP = require('wink-nlp');
const model = require('wink-eng-lite-web-model');
const vectors = require('wink-embeddings-sg-100d');

const nlp = winkNLP(model, ['sbd'], vectors);   // embeddings = 3rd constructor arg
```

Facts:

- 100-dimensional vectors, 350K+ words. ~110MB download / ~310MB installed. Requires wink-nlp ≥ 2.1.0.
- **Too heavy for the browser** — this is a Node-side dependency. For browser embeddings use `doc.contextualVectors()` instead (see `browser-guide.md`).

Usage once wired:

```js
token.out(its.vector);                          // 100-dim vector for one token

// Averaged embedding for a sentence/document — drop stop words first
doc.tokens()
  .filter((t) => t.out(its.type) === 'word' && !t.out(its.stopWordFlag))
  .out(its.value, as.vector);

// Compare two averaged vectors
similarity.vector.cosine(vecA, vecB);

// Contextual vectors: learn corpus-local vectors (v2.0+).
// Throws if word vectors aren't loaded; with lemma:true requires 'pos' in the pipe.
doc.contextualVectors();
```

## Utility import paths

| Module | Import |
|---|---|
| BM25 vectorizer | `require('wink-nlp/utilities/bm25-vectorizer')` |
| Similarity | `require('wink-nlp/utilities/similarity.js')` |
| Model install helper (legacy Node-only model) | `require('wink-nlp/models/install')` — only for `wink-eng-lite-model`; avoid |

## Reference links

- Pipeline: <https://winkjs.org/wink-nlp/processing-pipeline.html>
- Custom entities: <https://winkjs.org/wink-nlp/custom-entities.html> · <https://winkjs.org/wink-nlp/learn-custom-entities.html>
- BM25: <https://winkjs.org/wink-nlp/bm25-vectorizer.html>
- Similarity: <https://winkjs.org/wink-nlp/similarity.html>
- Language models: <https://winkjs.org/wink-nlp/language-models.html>
- Changelog: <https://github.com/winkjs/wink-nlp/blob/master/CHANGELOG.md>
