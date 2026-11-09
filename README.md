# Merge gate parity

Keep local and remote merge gates in agreement — one command that runs everything, a GitHub workflow split by concern, scripted gates with a shared output contract, and the rule that a new gate lands in every file in the same change. Use when adding a linter, a security scan or a custom checker, or when a build is green locally and red on the pull request.

One skill, one repository. Part of a twelve-skill pack extracted from a production Rails 8 +
Inertia + React helpdesk monolith — the opinionated layer no public Rails or shadcn skill can know.
Version `2.0.0` tracks the source constitution.

## Install

```jsonc
// skills-lock.json
{
  "version": 1,
  "skills": {
    "merge-gate-parity": {
      "source": "mahardikalgw/ruby-merge-gate-parity",
      "sourceType": "github",
      "skillPath": "SKILL.md"
    }
  }
}
```

Or clone and symlink:

```bash
git clone git@github.com:mahardikalgw/ruby-merge-gate-parity.git
ln -s "$PWD/ruby-merge-gate-parity" .claude/skills/merge-gate-parity
```

## When to activate

- Adding a linter, formatter, security scan, or custom checker.
- A PR fails on something that passes locally, or the reverse.
- A gate is slow enough that people skip it.
- A suppression or `--no-verify` appears in a diff.

## What's inside

```
ruby-merge-gate-parity/
  SKILL.md
  bin/check-skill
  references/gate-catalogue.md
```

`SKILL.md` is a router (kept under 250 lines) — the depth lives in `references/`, and the
copy-in code lives in `assets/`.

## Validate

```bash
./bin/check-skill
```

Checks frontmatter, the activation section, link resolution, orphaned references, script
executability, leftover placeholders, and — when the source app is checked out next to this
repository — that vendored scripts still match it byte for byte. Runs in CI on every push.

## Related skills

- [`flat-ui-contract`](https://github.com/mahardikalgw/ruby-flat-ui-contract) — INV-1…INV-9 and THEME-1…3, with both gate scripts vendored
- [`speckit-feature-lifecycle`](https://github.com/mahardikalgw/ruby-speckit-feature-lifecycle) — specify → clarify → plan → tasks → implement → analyze → converge

## Licence

MIT — see [LICENSE](LICENSE).
