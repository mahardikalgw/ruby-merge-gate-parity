# The gate catalogue

| Gate | Catches | Invocation |
|---|---|---|
| RuboCop | style drift, dead code, non-Omakase idioms | `bin/rubocop -f github` |
| UI component contract | raw elements, borders, shadows, off-scale type and corners, orphaned components, layout drift | `bin/check-ui-components` |
| Theme contract | literal colours, unmapped tokens, contrast below threshold | `bin/check-theme` |
| Bundler audit | a gem with a known CVE | `bin/bundler-audit` |
| npm audit | a JS dependency with a known CVE | `npm audit --audit-level=high` |
| Brakeman | injection, mass assignment, unsafe redirects, XSS | `bin/brakeman --quiet --no-pager --exit-on-warn --exit-on-error` |
| Rails tests | unit, integration, query, service, mailer, job | `bin/rails test` |
| System tests | the critical journey through a browser | `bin/rails test:system` |
| Seeds | the demo dataset loads cleanly in the test environment | `env RAILS_ENV=test bin/rails db:seed:replant db:test:prepare` |
| Type check | the Inertia prop contract on the client | `npm run check` (`tsc`) |

`npm run check` belongs in the `lint` job. A prop renamed in Ruby and not in `types.ts` is a runtime
`undefined`, which no test catches unless it reads that field.

## What each custom gate should refuse

**UI component contract**

```
INV-1  <button>, <input>, <select>, <textarea> outside components/ui
INV-2  outline-none; a text field that paints a focus ring
INV-3  a components/ui file with no import site
INV-4  a page outside the sanctioned layouts; a second nav+header implementation
INV-5  shadow-* / ring-*
INV-6  border utilities (except border-0, border-collapse, border-separate, border-transparent, border-spacing)
INV-7  text-[…], text-3xl+, inline fontSize
INV-8  rounded-xs|xl|2xl|3xl|4xl, rounded-[…]
INV-9  no tabular-nums in the stylesheet, or a proportional-nums opt-out
```

**Theme contract**

```
THEME-1  a :root colour not mapped in @theme inline, or a mapping to an undeclared role
THEME-2  hex / rgb() / hsl() / bg-white / bg-black / Tailwind palette class outside the palette block
THEME-3  a recorded pairing below its WCAG threshold
```

**Skills pack (this repository's own gate)**

```
SK-1  frontmatter name matches the directory; description long enough to route on
SK-2  SKILL.md <= 250 lines and declares its activation
SK-3  every relative link resolves
SK-4  no orphaned file under references/ or assets/
SK-5  everything under scripts/ is executable
SK-6  vendored scripts match the app they were copied from
SK-7  no absolute author paths, no placeholder markers, no unfilled braces
```

## Gate ordering inside one job

Cheap and deterministic first, so a typo does not wait four minutes for a browser:

```
rubocop -> check-ui-components -> check-theme -> tsc -> rails test -> system test -> seeds
```

Within a single `bin/ci` run the order is the same, with `bin/setup` first so a fresh clone can run
everything without a manual step.

## Timing budget

| Gate | Target | When it exceeds |
|---|---|---|
| RuboCop | < 10 s | a cache miss; fix the cache key, not the patience |
| Custom checkers | < 3 s | the glob is walking `node_modules` |
| Rails tests | < 90 s | a test is hitting the network or building a world in a loop |
| System tests | < 5 min | a test is asserting things that belong in an integration test |
| Seeds | < 30 s | a per-row query in a loop |

A gate that takes minutes gets skipped locally, and a gate skipped locally is a red PR. Optimise or delete,
never tolerate.

## Pre-push hook

```bash
#!/usr/bin/env sh
# .git/hooks/pre-push
set -e
bin/ci
```

Ninety seconds to know the PR will be green. A hook that runs a **different** set of checks than CI is
worse than none, because it teaches the team to trust a green that means nothing.
