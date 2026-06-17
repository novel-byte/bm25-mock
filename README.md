# bm25-mock

A BM25 ranking implementation, in TypeScript.

**Status: not yet implemented.** This repository holds the design and the
intended API. The code is not written yet.

BM25 is the ranking function behind most search systems that predate or avoid
neural retrieval — Elasticsearch, Lucene, Solr, and the search in most
databases. It scores a document against a query by combining three ideas: how
often the query terms appear in the document, how rare those terms are across
the collection, and how long the document is relative to the average.

This is a mock: a small, dependency-free implementation of the scoring
function, with no index, no storage, and no query parser. The point is the
ranking arithmetic and nothing else.

## Why BM25

Term frequency alone ranks a long document above a short one that actually
answers the question. BM25 corrects for that in three ways:

- **Saturation.** The tenth occurrence of a term in a document counts for far
  less than the second. The gain flattens as the count rises, which is what
  `k1` controls.
- **Inverse document frequency.** A term appearing in every document carries
  no information. The rarer a term is, the more it should matter.
- **Length normalisation.** A long document is more likely to contain a term
  by chance, so its score is divided by a length factor that `b` controls.

## The scoring function

For a query with terms \(q_1 \dots q_n\), a document \(D\), and a corpus of
\(N\) documents:

```
IDF(q_i)  = ln( 1 + (N - n(q_i) + 0.5) / (n(q_i) + 0.5) )

score(D)  = Σ IDF(q_i) · ( f(q_i, D) · (k1 + 1) )
                        ─────────────────────────────────────
                        f(q_i, D) + k1 · (1 - b + b · |D| / avgdl)
```

Where `f(q_i, D)` is the raw term frequency of `q_i` in `D`, `n(q_i)` is the
number of documents containing `q_i`, `|D|` is the length of `D` in terms,
and `avgdl` is the average document length across the corpus.

The `+1` inside the logarithm keeps the IDF positive for terms appearing in
more than half the documents, which avoids the negative scores that the
original Robertson formulation can produce.

### The two parameters

| Parameter | Default | Effect |
| --- | --- | --- |
| `k1` | `1.2` | Term frequency saturation. At `0` term frequency is ignored entirely and only IDF matters. Higher values let term frequency keep counting for longer. |
| `b` | `0.75` | Length normalisation. At `0` document length is ignored. At `1` length is normalised fully against the average. |

The defaults are what Robertson and Zaragoza published in 2009 after
experimenting on real collections, and they remain a reasonable starting point
for most collections.

### A worked example

Two documents, and a query for `ranking`:

| | `D1` | `D2` |
| --- | --- | --- |
| Length in terms | 100 | 20 |
| Times it contains `ranking` | 3 | 1 |
| Documents containing `ranking` | 2 of 2 | 2 of 2 |
| Average document length | 60 | 60 |

With `k1 = 1.2`, `b = 0.75` and `N = 2`:

```
IDF(ranking) = ln(1 + (2 - 2 + 0.5) / (2 + 0.5)) = ln(1.2) ≈ 0.1823

D1:  length factor = 1 - 0.75 + 0.75 · (100 / 60) = 1.50
     score = 0.1823 · (3 · 2.2) / (3 + 1.2 · 1.50) ≈ 0.2507

D2:  length factor = 1 - 0.75 + 0.75 · (20 / 60) = 0.50
     score = 0.1823 · (1 · 2.2) / (1 + 1.2 · 0.50) ≈ 0.2507
```

The two documents score the same. `D1` mentions the term three times, but it
is five times longer, and the length normalisation cancels the difference
out. This is the behaviour BM25 was designed to produce, and it is why
tuning `b` towards `0` changes the character of the results rather than just
the scale.

## The intended API

```ts
import { BM25 } from "bm25-mock";

const bm25 = new BM25({
    documents: [
        { id: "d1", text: "..." },
        { id: "d2", text: "..." },
    ],
});

const results = bm25.search("ranking", { limit: 10 });
// [{ id: "d1", score: 0.2507 }, { id: "d2", score: 0.2507 }]
```

The constructor tokenises the documents and records the corpus statistics
needed for IDF and length normalisation. `search` tokenises the query, scores
every document, and returns the highest scoring ones. Documents scoring zero
are omitted.

### Tokenisation

Lowercased, split on non-alphanumeric characters, with a small stop-word
list removed. No stemming and no phrase support: this is a mock of the
scoring function, not of a search engine.

## What this is not

- **Not an index.** There is no inverted index; scoring is a linear pass over
  the documents.
- **Not persistent.** The corpus lives in memory and is rebuilt on every
  construction.
- **Not a search engine.** No query parsing, no ranking beyond BM25, no
  pagination beyond a `limit`.

## Licence

MIT.
