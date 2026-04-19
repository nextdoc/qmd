# Clojure support

QMD treats Clojure as a first-class indexed language. `.clj`, `.cljs`, and
`.cljc` files are picked up by all the usual commands (`search`, `query`,
`vsearch`, `get`, MCP tooling) with no extra flags, and AST-aware chunking
is available via `--chunk-strategy auto`.

## What AST-aware chunking does for Clojure

With `--chunk-strategy auto`, chunk boundaries align with Clojure's
top-level semantic forms rather than arbitrary token counts. Boundaries
and their scores (higher = stronger preference):

| Form                                                                               | Capture  | Score |
|------------------------------------------------------------------------------------|----------|-------|
| `(ns ...)`                                                                         | `@mod`   | 100   |
| `(defprotocol ...)`, `(defrecord ...)`, `(deftype ...)`, `(definterface ...)`      | `@class` | 100   |
| `(defn ...)`, `(defn- ...)`, `(defmacro ...)`, `(defmulti ...)`, `(defmethod ...)` | `@func`  | 90    |

Scores match the rest of qmd's scale (markdown h1 = 100, h2 = 90, etc.)
so `findBestCutoff` in `src/store.ts` treats them uniformly with the
other supported languages.

## How it's wired in

- **Grammar source**: [sogaiu/tree-sitter-clojure](https://github.com/sogaiu/tree-sitter-clojure), CC0-1.0.
- **Vendored WASM**: `assets/grammars/tree-sitter-clojure.wasm`. The
  source commit and rebuild recipe are recorded in
  [`assets/grammars/CREDITS.md`](../assets/grammars/CREDITS.md).
- **Integration point**: `src/ast.ts`. Five maps drive language support
  (`SupportedLanguage`, `EXTENSION_MAP`, `GRAMMAR_MAP`,
  `LANGUAGE_QUERIES`, `SCORE_MAP`) and `GRAMMAR_MAP` accepts either
  `{ pkg, wasm }` (npm-resolved, as used for TS/JS/Python/Go/Rust) or
  `{ vendored }` (loaded relative to `assets/grammars/`).

## Trade-offs

### Vendored WASM instead of npm dependency

sogaiu does not publish their grammar to npm (the `tree-sitter-clojure`
name is squatted by a stale 2019 fork). Realistic alternatives would
have been a git dependency plus a `postinstall` that runs
`tree-sitter-cli build --wasm` on every install — which forces
`tree-sitter-cli` + wasi-sdk onto every end-user machine.

Vendoring avoids that tax at the cost of manual updates: to refresh the
grammar, follow the recipe in `assets/grammars/CREDITS.md` and commit
the new `.wasm`. The grammar is slow-moving (still 0.0.13, substantive
changes in 2025 have been README-only), so an as-needed manual refresh
is reasonable.

Grammar is CC0-1.0, so bundling into MIT-licensed qmd imposes no
license obligations.

### Chunk boundaries exclude bare `def` and `defonce`

Top-level `def`s are common enough in Clojure that treating each as a
chunk boundary would fragment files into many small chunks with weak
semantic signal. Only function-, macro-, and type-level forms get
boundaries. If per-`def` chunking becomes desirable, it's a one-line
addition to the `LANGUAGE_QUERIES.clojure` pattern in `src/ast.ts`.

### No namespaced-form matching

The query matches unqualified forms only — `(clojure.core/defn ...)`
won't register as a `@func` boundary. This is an intentional edge-case
choice; namespaced `def*` calls at the top level are rare in practice.

### Query-local helper captures are filtered

The Clojure query uses the `#match?` and `#eq?` predicates to discriminate
between `defn`/`defmacro`/etc., which requires a helper capture on the
head symbol (`@_h`). Tree-sitter's convention is that capture names
beginning with `_` are for predicate use and are not emitted as results,
but `web-tree-sitter` returns them regardless — so `src/ast.ts` filters
them explicitly in the capture loop. Other language queries don't use
predicates and so don't hit this path.

## Limitations

- **No symbol-level search.** You can full-text- or vector-search
  Clojure content, but you can't ask qmd for "all `defmethod`s on `:foo`"
  or navigate by symbol.
- **Embedding model is not Clojure-specialized.** Vector search uses
  embeddinggemma, which treats source code as plain text. Works well
  enough because Clojure names tend to be descriptive, but it wasn't
  trained on S-expressions specifically.
- **Reader conditionals and tagged literals are not specially handled.**
  The grammar parses them correctly, but `#?(:clj ... :cljs ...)` forms
  aren't split into per-platform chunks.

## Usage

```sh
qmd collection add ~/code/my-clj-project --name myclj --chunk-strategy auto
qmd embed
qmd query "how is authentication handled"
qmd search "defn authenticate"
```

`qmd status` lists `clojure` among the available AST languages when the
vendored grammar has loaded correctly.
