# markdown-render-speed-02 entry — a marked fork

`parse.js` is marked **18.1.3** (MIT, see `LICENSE-marked` — Copyright (c)
2018+ MarkedJS, Copyright (c) 2011-2018 Christopher Jeffrey), optimised
surgically for speed while keeping its output byte-for-byte identical.
`main.ts` is the kit's driver, unchanged: it reads `cases.json`, renders each
case alone with `parse()` from `parse.js`, and writes `html.json`.

Optimisations (each verified 720/720 on the practice set, reverted if it broke
a case or gained nothing):

- Inline tokeniser: first-character dispatch — every inline rule is anchored,
  so a rule whose first character cannot match is never attempted.
- Block tokeniser: same first-character dispatch for fences / heading / hr /
  blockquote / list / html / def; a delimiter-line check before the table
  regex; a cheap `\n {0,3}[=-]` pre-test before the setext-heading regex.
- `list()`: per-line first-character dispatch replacing six regex tests per
  line, `indexOf('\n')` instead of `split('\n', 1)`, hand-rolled first-
  non-space scans, tab expansion only when the leading whitespace holds a tab,
  and an hr-only guard before the hr regex.
- Emphasis closer scan (`beginEmStrongDelCloserScan`): an empty frame when
  the string holds no `*`, `_` or `~`, instead of a char-by-char scan.
- `emStrong`: surrogate-pair-safe replacements for spread-based code-point
  counts (`[...s].length`, `[...s].at(-1)`); delimiter runs are ASCII.
- `codespan`: newline replacement only when the span holds a newline.
- `#maskSrc`: the punctuation and block-skip replaces are skipped when the
  string cannot contain what they match (no backtick/backslash, no `[`/backtick/`<`).
- Entry point calls `_Lexer.lex` + `_Parser.parse` directly, bypassing the
  `Marked` wrapper.

Measured on the kit practice set (720 cases, same driver, interleaved A/B):
baseline marked ~723 ms, this fork ~503 ms — **~1.44x**. Deterministic: two
runs produce identical `html.json`.

Source of the fork (TypeScript, before bundling) lives in the build
workspace; `parse.js` here is the esbuild bundle of it.
