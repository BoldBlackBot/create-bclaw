# Rename bclaw → dispatch

**Date:** 2026-08-15
**Status:** Proposed

## Goal

Rename the product from **bclaw** to **dispatch** across the generator repo: npm package, GitHub repo, homepage, CLI identity, generator rename token, and the bundled `template/` snapshot. This is Phase 1 of the product rename — the code sweep lands first; the GitHub repo and npm package renames follow after this PR merges (copy leads, services catch up).

## Motivation

The name is changing at the product level (site copy precedent: "Name your dispatch agent", `/setup-dispatch`, `/manage-dispatch`, `/teardown-dispatch`). The new name is not short for anything — the "BusinessClaw" etymology is retired. `dispatch` reads as a normal English word and works as a scoped package suffix, a repo suffix, and a token inside generated IAM/SSM/KMS identifiers.

## Technical Details

### Replacement map (condensed)

| Old | New |
|---|---|
| npm package `@boldblackai/create-bclaw` | `@boldblackai/create-dispatch` (stays scoped) |
| npm-init shorthand `npm init @boldblackai/bclaw` | `npm init @boldblackai/dispatch` (npm prepends `create-` itself; the old line resolved to `@boldblackai/create-bclaw`) |
| GitHub repo `boldblackai/create-bclaw` | `boldblackai/create-dispatch` (renamed on GitHub **after** this PR merges; redirect covers old links) |
| homepage `https://bclaw.sh` | `https://dispatch.boldblack.ai` |
| bin `create-bclaw` | `create-dispatch` |
| generator rename token `RENAME_FROM = "bclaw"` | `RENAME_FROM = "dispatch"` |
| template SSM namespace `/bclaw/` | `/dispatch/` |
| template skills `setup-bclaw` / `manage-bclaw` / `teardown-bclaw` | `setup-dispatch` / `manage-dispatch` / `teardown-dispatch` (dirs, `SKILL.md` frontmatter names, cross-references) |
| git init author fallback `create-bclaw@local` | `create-dispatch@local` |

Region token handling (`us-east-1`) is untouched.

The template must keep the token **lowercase and standalone** (rename-model constraint — a literal substring replace is the whole transform). The one capitalized sentence start ("Bclaw uses a two-role model") is normalized to lowercase as part of the sweep. `template/` contained no pre-existing English-word `dispatch` occurrences, so the new token cannot collide with prose.

The golden test is updated first (TDD): it now generates with `name=dispatch` expecting `template/` byte-for-byte, and its residual grep hunts `dispatch`.

### Verification

- Golden test (17 tests): `pnpm test` — byte-for-byte template fidelity, both token renames, residual greps.
- Sweep gate: `grep -rniE "bclaw" .` → zero hits outside historical RFCs (dated records keep their titles).

## Migration Notes

- **Existing deployed agents are NOT affected.** The rename changes what the generator emits for *new* generations; running deployments keep their generated names, SSM namespaces, and IAM scopes untouched.
- **npm:** the old package `@boldblackai/create-bclaw` remains installable and will be deprecated with a pointer to `@boldblackai/create-dispatch`. Users should switch to `npx @boldblackai/create-dispatch <name>`. Note: the new name's first publish must be manual (trusted publishing cannot do first publishes, npm/cli#8544); trusted-publisher registration follows, then `npm deprecate` of the old name.
- **GitHub:** renaming `boldblackai/create-bclaw` → `boldblackai/create-dispatch` leaves a redirect on the old URL, so existing clones and links keep working (local clones should update their remote URL at their leisure).
- **Homepage:** `https://bclaw.sh` → `https://dispatch.boldblack.ai` (DNS/site cutover follows the repo rename).

## Implementation Checklist

- [x] Golden test swept to the new token (RED first, then GREEN)
- [x] `src/generate.ts` — `RENAME_FROM = "dispatch"`; `src/cli.ts` — banner, help, prompt default, git-init identity, commit message; npm-init shorthand corrected
- [x] `template/` — token sweep in contents; `git mv` for policy files and the three skill directories
- [x] `package.json` — name, bin, version 1.1.0, repository URL, homepage; lockfile regenerated
- [x] `tag-on-merge.yml` idempotence guard → `@boldblackai/create-dispatch@${VERSION}`
- [x] `README.md` rewritten (naming, install command, links, etymology removed)
- [x] `CHANGELOG.md` — 1.1.0 entry
- [x] `AGENTS.md` self-references + `journal:create-dispatch:` corkboard namespace
- [ ] (post-merge) GitHub repo renamed to `boldblackai/create-dispatch`
- [ ] (post-merge) manual first publish of `@boldblackai/create-dispatch`, trusted-publisher registration, deprecate `@boldblackai/create-bclaw`
- [ ] (post-merge) homepage DNS cutover to `dispatch.boldblack.ai`
