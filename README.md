<h1 align="center">tfidf.mq</h1>

> [!WARNING]
> This project is currently under active development. APIs and features may change without notice.

> [!NOTE]
> This module requires mq v0.10.1 or later, the same as [ds.mq](https://github.com/harehare/ds.mq). The heap relies on `slice` cutting an unshared array in place, which is available from that version; older versions may not work or copy the heap on every `pop`.

Full-text ranking for [mq](https://github.com/harehare/mq), as a module: BM25 search over a set of documents and TF-IDF keyword extraction. Documents are plain strings, so it works on Markdown sections, headings, list items or anything else you can pull out with a query.

## Installation

`tfidf.mq` imports [ds.mq](https://github.com/harehare/ds.mq) for its heap, which is included as a git submodule (pinned to `v0.1.0`). Clone with it:

```sh
git clone --recurse-submodules https://github.com/harehare/tfidf.mq
```

If you already cloned without it, run `git submodule update --init`.

Use it by adding both directories to the module path:

```sh
mq -L tfidf.mq -L tfidf.mq/vendor/ds.mq -I null 'import "tfidf" | tfidf::tokenize("Hello, World!")'
```

## Usage

```sh
mq -L . -L vendor/ds.mq -I null '
  import "tfidf"
  | let docs = ["the cat sat on the mat", "the dog chased the cat", "dogs and cats are pets", "quantum field theory"]
  | let index = tfidf::build(docs)
  | tfidf::search(index, "cat", 2)
'
# [[0.677110356331052, 1], [0.6197551261477159, 0]]
```

Results are `[score, document index]` pairs, best first. To rank the sections of a document, split it into strings first and map the indexes back to them.

| Function | Description |
| --- | --- |
| `build(docs)` | index over an array of strings |
| `build_with(docs, tokenizer)` | same, with your own `fn(text): [tokens];` |
| `search(index, query, k)` | the best `k` documents for a query string |
| `search_tokens(index, tokens, k)` | same, for a query that is already split |
| `keywords(index, doc, k)` | the `k` terms that best set document `doc` apart, as `[score, term]` |
| `idf(index, term)` | BM25 inverse document frequency of a term |
| `tokenize(text)` | lowercase words, split on anything that is not a letter or a digit |

Scoring is BM25 with `k1 = 1.5` and `b = 0.75`. Documents sharing no term with the query are left out, and ties go to the earlier document.

## Notes

- There is no stemming or stop-word list: `cats` does not match `cat`. Filter terms in a custom tokenizer if you need that.
- `tokenize` splits on whitespace and punctuation, so Japanese or Chinese text stays one token per run. Pass a tokenizer that splits it (for example into character n-grams) to `build_with` and `search_tokens`.
- `build` costs O(total tokens). `search` scans every document, O(documents x query terms), plus O(m log k) to pick the best `k` of `m` matches.

## Development

```sh
git submodule update --init
mq-test -L vendor/ds.mq tfidf_tests.mq
mq-bench -L vendor/ds.mq -L . tfidf_bench.mq
```

One of the goals of this module is also to check mq for performance problems, so it comes with a benchmark in `tfidf_bench.mq`.
