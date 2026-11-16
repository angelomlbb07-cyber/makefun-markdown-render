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

## Final measurements (2026-11-16)

- Practice set: **720/720** cases byte-identical to marked 18.1.3, verified
  through the kit driver (`main.ts` + `parse.js`).
- Speed (interleaved A/B, same driver, same machine): baseline marked
  ~710–750 ms, this fork ~460–515 ms — **~1.44–1.60x** depending on machine
  load; the ratio is consistently above the ~1.4x top-20 threshold and below
  the ~2x top-3 threshold. The remaining time is core parsing machinery
  (inline tokeniser loop, list recursion, emphasis matching, masking), with
  no further safe surgical win found; a from-scratch rewrite was already
  tried twice (v1 hand-rolled 76/720, v2 structure rewrite 714/720) and is
  not a credible route to 2x.
- Determinism: two driver runs produce byte-identical `html.json`.
- Fuzz: 100,000 random snippets (doc substrings, token soup, spec
  mutations) — 0 mismatches vs baseline.
