# TODO

Findings from an adversarial review, ordered by what they cost if left alone.

This repository is executed by every other repository in the organisation on
every push and every pull request. Its blast radius is the estate.

---

## ~~P0~~ FIXED -- the repository everything executes was the one nothing guarded

### 1. ~~No CI runs here~~ FIXED

All ten workflows are `on: workflow_call` only. There is no caller. So the
repository whose `main` becomes arbitrary code in every sibling's CI is the
only one that never runs gitleaks, never runs zizmor on its own workflows, and
never runs the prose gate it defines.

`common-fast.yml` runs `zizmor .github/workflows` -- on the **caller's**
workflows. Never on the shared ones being audited from.

Compounding it: `mature-discipline` requires zero approving reviews, and
`baseline.txt` audits only `deletion`, `non_fast_forward`, `pull_request`. So a
PR into this repo needs no approval and has no required contexts.

Chain: self-merge here becomes code execution in all three siblings' CI on
their next PR, including `rust-release.yml` with `contents: write`,
`id-token: write`, `attestations: write` -- the ability to sign attestations
for arbitrary artefacts.

**Fix:** add `.github/workflows/ci.yml` calling `common-fast.yml` on push and
pull_request. Add a repository ruleset on `main` naming those contexts as
required. This is the single highest-leverage change in the estate.

### 2. ~~Dependabot is excluded from the repo holding every action pin~~ FIXED

`maestro-governance/baseline.txt` scopes `dependabot.yml` to three repos. This
one is not among them, and has no such file.

Every consumer's CI toolchain is pinned **here**: `actions/checkout`,
`taiki-e/install-action`, `googleapis/release-please-action`,
`actions/create-github-app-token`, `ossf/scorecard-action`,
`github/codeql-action`. SHA pinning is required org-wide, which is correct and
means the pins never move on their own. Without Dependabot they never move at
all, and a CVE in a pinned action is invisible in the one place it is
guaranteed to be running.

**Fix:** add `.github/dependabot.yml` with the `github-actions` ecosystem, and
add this repo to the baseline's scope for that file.

---

## P1 -- a control that cannot fail is not a control

### 3. `ts-arch` runs depcruise with no failing rule

`ts-fast.yml:148`

```yaml
run: npx depcruise --config .dependency-cruiser.js src
```

`depcruise` exits 0 unless a rule has `severity: error`. Nothing here ships a
`.dependency-cruiser.js`, so the first TypeScript repo to adopt this either
fails on a missing config or passes with a config someone wrote in a hurry.

The estate coined "a gate that cannot fail is decoration." This is one.

**Fix:** ship a default config in `.shared-policy/` with at least
`no-circular: error`, and have the workflow fall back to it.

### 4. Zero required approvals plus a one-name CODEOWNERS

`GOVERNANCE.md:38` describes review; `CODEOWNERS` names one person; the ruleset
requires zero approvals. With six contributors this reviews nothing -- and the
CODEOWNERS entry gives the false impression that it does.

**Fix:** decide. Either require one approval and accept that the solo
maintainer cannot self-merge, or delete CODEOWNERS and stop implying review.

---

## P2

### 5. Local and CI gitleaks are different engines under a comment claiming parity

`.pre-commit-config.yaml` (in each repo) runs one gitleaks; `common-fast.yml:36`
runs another. ADR-0002 says local hooks mirror the fast tier. Two engines, two
rule sets, one claim of equivalence -- so a secret can pass locally and fail in
CI, which is the specific surprise that ADR exists to prevent.

**Fix:** pin both to the same version, or say plainly that local is a fast
approximation.

### 6. `SECURITY.md` hardcodes one repository

`SECURITY.md:5-7` tells a reporter to file against a named repo regardless of
which repo they found the issue in. This is the org-wide policy file; it is
inherited by every repo including ones that do not exist yet.

**Fix:** say "the affected repository."

### 7. `CONTRIBUTING.md` points at a file this repo does not have

`CONTRIBUTING.md:12` tells a contributor to run the hooks. There is no
`.pre-commit-config.yaml` here. A contributor to this repo -- the highest-risk
repo in the estate -- has no local gate at all. Fixed by #1.
