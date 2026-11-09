---
name: merge-gate-parity
description: Keep local and remote merge gates in agreement — one command that runs everything, a GitHub workflow split by concern, scripted gates with a shared output contract, and the rule that a new gate lands in every file in the same change. Use when adding a linter, a security scan or a custom checker, or when a build is green locally and red on the pull request.
version: 2.0.0
---

# Merge gate parity

A gate is only real if it runs **everywhere the code can be merged**. The failure this prevents is
specific and common: a check added to the local runner and forgotten in CI (green locally, red on the PR),
or added to CI and never run locally (a surprise after lunch).

## When to activate

- Adding a linter, formatter, security scan, or custom checker.
- A PR fails on something that passes locally, or the reverse.
- A gate is slow enough that people skip it.
- A suppression or `--no-verify` appears in a diff.

## Three files, one list

| File | Role |
|---|---|
| `bin/ci` | the entry point a developer runs |
| `config/ci.rb` | the ordered list of steps, with comments |
| `.github/workflows/ci.yml` | the same checks, split into jobs on the PR |

Change the list → change it in all three. A repo with a fourth place (`Makefile`, `package.json`,
`script/test`) has a fourth place to forget.

## The local runner

```ruby
# config/ci.rb — run with bin/ci. The first failing step stops the run.
CI.run do
  step "Setup",                            "bin/setup --skip-server"

  step "Style: Ruby",                      "bin/rubocop"
  step "Style: UI components",             "bin/check-ui-components"
  step "Style: Theme tokens",              "bin/check-theme"

  step "Security: Gem audit",              "bin/bundler-audit"
  step "Security: JS dependency audit",    "npm audit --audit-level=high"
  step "Security: Brakeman code analysis", "bin/brakeman --quiet --no-pager --exit-on-warn --exit-on-error"

  step "Tests: Rails",                     "bin/rails test"
  step "Tests: System",                    "bin/rails test:system"
  # Replanting seeds in the test environment leaves their committed rows behind, and the next
  # `bin/rails test` then fails fixture FK validation on the orphans. Preparing the test database at
  # the end of this step keeps repeated runs idempotent.
  step "Tests: Seeds",                     "env RAILS_ENV=test bin/rails db:seed:replant db:test:prepare"
end
```

Order is not cosmetic: setup first (so a clone can run everything), cheap checks before expensive ones,
and any step that mutates the test database last.

## The workflow, split by concern

```yaml
jobs:
  scan_ruby:    # brakeman + bundler-audit
  scan_js:      # npm audit
  lint:         # rubocop -f github, check-ui-components, check-theme
  test:         # db:test:prepare test        (pgvector/pgvector:pg16 service)
  system-test:  # db:test:prepare test:system (uploads tmp/screenshots on failure)
```

## The gate script contract

Every custom gate behaves the same way, whether it is 20 lines or 300:

```ruby
#!/usr/bin/env ruby
# frozen_string_literal: true

# Header: what each rule is, why it exists (which incident or feature produced it), and the exit
# contract. The header is the documentation for whoever inherits the rule.
#
#   RULE-1  <statement>   (feature/incident that produced it)
#   RULE-2  ...

ROOT = File.expand_path("..", __dir__)
violations = []

# ... detect ...

if violations.empty?
  puts "check-thing: 0 violations"
  exit 0
end

violations.sort.each { |violation| puts violation }   # path:line: message
puts "check-thing: #{violations.size} violations"
exit 1
```

Four properties:

1. **`path:line: message`** — editors, terminals and CI annotations all understand it; a prose summary
   does not.
2. **Exit 0/1, never 2** — a crash and a violation must be distinguishable in a log.
3. **Non-interactive** — no prompts, no pagers, no `--fix` that silently rewrites files in CI.
4. **A count on success** — "0 violations of 30 components" proves the gate actually ran.

### A gate must fail when it covers nothing

```ruby
exit 1 if scanned_files.empty?          # the glob broke; do not report success
```

The most dangerous gate is the one whose glob stopped matching after a rename.

## Authoring a new gate

1. Find a rule that was broken at least twice. A rule invented in advance will be wrong.
2. Write it as a script with the contract above.
3. Run it against the current tree. If it fails on existing code, either fix the code first or narrow the
   rule — never ship a gate that starts red, because the first response will be a suppression.
4. Add it to `config/ci.rb` in the right phase.
5. Add it to `.github/workflows/ci.yml` in the matching job.
6. Document it where contributors look: the repo's agent briefing and the gate table.

## Removing or weakening a gate

A repeatable process, not an inline edit:

1. State what the gate caught in the last quarter. "Nothing" is a real answer and a legitimate reason.
2. If it is being replaced, add the replacement **first** and keep both for one release.
3. Record the removal: which gate, why, what is now responsible for that class of defect.
4. Update all three files in the same change.

A gate deleted quietly inside a feature PR is how a repository loses its guarantees one convenient
exception at a time.

## Security gates are not advisory

```bash
bin/brakeman --quiet --no-pager --exit-on-warn --exit-on-error
bin/bundler-audit
npm audit --audit-level=high
```

`--exit-on-warn` is deliberate: in warn-only mode nobody reads the output, and the first real SQL injection
ships unnoticed. A suppression is legal only with an inline justification:

```ruby
# brakeman:ignore [CommandInjection] Argument is a constant from a whitelist, never user input.
```

Fix findings instead of suppressing them. A suppression is a decision that needs an owner.

## Triage when local and CI disagree

| Symptom | Usual cause |
|---|---|
| Passes locally, fails on the PR | a gate exists in `config/ci.rb` but not in the workflow (or the reverse) |
| Fails only on a fresh clone | a step relies on an artifact from an earlier local run (Vite manifest, compiled assets) |
| Fails in the test job, passes in `bin/ci` | CI prepares the test database; the local run reused a dirty one |
| Passes on a re-run | a test depends on ordering or a shared record |
| Differs between machines | an unpinned tool version (`npx latest`, a floating Ruby) |

Every row is a parity bug, not a flake. A re-run that goes green is a bug that will return.

## References

- [references/gate-catalogue.md](references/gate-catalogue.md) — what each gate catches, and its exact invocation

