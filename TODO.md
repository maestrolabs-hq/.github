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

### 2a. This repository has no required status check and no repository ruleset

```text
$ gh api repos/maestrolabs-hq/.github/rulesets --jq '.[]|"\(.name)\t\(.source_type)"'
commits-are-conventional   Organization
floor-no-destruction       Organization
floor-release-tags         Organization
mature-discipline          Organization
visibility-is-frozen       Organization
```

Nothing repository-sourced, so no `required_status_checks` rule at all. The
organisation `pull_request` rule carries `required_approving_review_count: 0`.
A pull request with every check red merges on one click.

`GOVERNANCE.md:22` states *"| Required status checks | per repository, naming
the fast-tier contexts |"* and `GOVERNANCE.md:34` states *"They are required
contexts, so a red check blocks the merge rather than inviting a judgement
call."* Neither is true here.

`ci.yml`'s own header explains the stakes: this repository's `main` becomes
arbitrary code in three siblings' CI on their next push, including
`rust-release.yml` with `contents: write`, `id-token: write` and
`attestations: write`.

**Fix:** create a repository ruleset on `main` requiring the eight contexts
this repo produces: `fast / dependency-review`, `fast / secrets-scan`,
`fast / prose`, `fast / brief`, `fast / markdown`, `fast / toml`,
`fast / no-absolute-paths`, `fast / actions-security`.

### 2b. `.pre-commit-hooks.yaml` has no callers, and the baseline says otherwise

```text
$ grep -rn "maestrolabs-hq/.github" --include=".pre-commit-config.yaml" .
NO REFERENCES FOUND
```

All three siblings inline their hooks with `repo: local`.
`maestro-governance/baseline.txt:96` asserts the opposite -- *"What could be
centralised has been. The hook definitions live in maestrolabs-hq/.github as
.pre-commit-hooks.yaml"* -- and that sentence is the stated reason
`.pre-commit-config.yaml` is hash-pinned for only two of three repos.

The three copies have already diverged, and the shared file carries the exact
defect the siblings' comments warn about: `.pre-commit-hooks.yaml:12` uses
`types: [rust]` with no `always_run`.

**Fix:** either have the siblings reference this file by URL and revision, or
delete it and correct `baseline.txt`.

### 2c. Scorecard has never completed a run

The README shows a Scorecard badge. `common-heavy.yml` defines the job, and no
run of it has ever completed, so the badge reports a score for a workflow that
has not executed here.

**Fix:** run it once and confirm the SARIF upload works, or remove the badge
until it does.

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

### 6a. `CODEOWNERS` names five paths that do not exist and omits all five that do

Every pattern in the file matches nothing in the repository, and the five
directories that do exist are unowned. Combined with zero required approvals
(#4), the review story is decorative twice over.

**Fix:** rewrite against the actual tree.

### 6b. `dependabot.yml` is outside the baseline's scope and has diverged

Related to #2: the file now exists but the baseline entry does not name this
repository, so the copy here is unwatched and already differs from the
siblings'.

**Fix:** add `dot-github` to the entry's scope.

### 6c. Five pinned tool versions are invisible to Dependabot

`gitleaks 8.30.1`, `zizmor 1.30.0`, `markdownlint-cli2 0.18.1`,
`similarity-rs`/`similarity-ts` and `taplo-cli` are pinned inside `run:` blocks
and `with: tool:` arguments, which no ecosystem scans. A CVE in any of them is
invisible.

**Fix:** move the versions to a place Dependabot reads, or record that they are
reviewed by hand and when.

### 6d. The `pending code_scanning_default_setup_query_suite` entry is stale and wrong

The baseline's pending entry names a key whose value no longer matches what the
API returns.

**Fix:** re-derive the key and value from a live read.

### 6e. Defects in the open pull request (#21)

- `ts-heavy.yml:70-71` -- an orphaned two-line brief left behind by the
  `cross-platform` move now sits above `osv-scan`, describing a job that moved
  to `ts-fast.yml`. Delete it.
- `rust-fast.yml` lacks `defaults: run: shell: bash`, which `python-fast.yml:37`
  and `ts-fast.yml:41` both set. The pull request added a `windows-latest` leg
  to it, so the next `run:` script added there gets pwsh.
- `CONTRIBUTING.md:46` says the gate refuses "a drive letter"; it refuses one
  only when the first segment is the Windows user root. A drive letter
  followed by any other directory passes, as do the other home-bearing
  environment variables. (Described rather than quoted: the gate scans every
  tracked file, so spelling out its trigger here would fail it.)
- `rust-heavy.yml` `wsl-toolchain` runs `just setup` and `just check` without
  `just install`, so `prek`, `cargo-deny`, `cargo-machete` and `similarity-rs`
  are all absent. The job cannot pass.

### 7. `CONTRIBUTING.md` points at a file this repo does not have

`CONTRIBUTING.md:12` tells a contributor to run the hooks. There is no
`.pre-commit-config.yaml` here. A contributor to this repo -- the highest-risk
repo in the estate -- has no local gate at all. Fixed by #1.
