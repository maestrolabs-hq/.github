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

---

## Round 2 -- the shared workflows, executed rather than read

### 7. `npx <tool>` resolves four shipped tools to the wrong npm package

`ts-fast.yml:181` runs `npx depcruise --config .dependency-cruiser.js src`. The
`depcruise` binary is shipped by `dependency-cruiser`; a package literally named
`depcruise` also exists. Run exactly as CI runs it, with the tool absent from
`package.json`:

```text
$ CI=true npx depcruise --version </dev/null
npm warn exec The following package was not found and will be installed: depcruise@1.0.0
This is a placeholder published to prevent dependency confusion.
EXIT=0
```

npm 11 auto-installs without prompting in a non-TTY. Four of six bare tool
names resolve wrongly:

| call site | intended package | what `npx` fetches |
| --- | --- | --- |
| `ts-fast.yml:59`, `:74` | `@biomejs/biome` | an unrelated env-var manager |
| `ts-fast.yml:91` | `typescript` | a deprecated 2016 release |
| `ts-fast.yml:181` | `dependency-cruiser` | a placeholder that **exits 0** |
| `ts-heavy.yml:134` | `@stryker-mutator/core` | a deprecated v0 |

The same four names are in `.pre-commit-hooks.yaml:74,81,88,96`, which this
repository publishes as the organisation's shared hook definitions.
`common-fast.yml:171` shows the author knows the correct pattern -- name and
version both pinned -- and no other `npx` call follows it.

**Why it matters:** `npm ci` runs first, so a consumer who declares the real
dependency is protected. One who does not gets an arbitrary registry package
executed in CI, and for `ts-arch` the architecture gate then reports success.
Anyone who publishes to those four names gets code execution in every consumer.

**Fix:** `npm exec --no -- <tool>`, which refuses to fetch and turns a silent
registry install into a red build.

### 8. Every Scorecard run in the estate reads zero files

`.gitattributes:77-84`, byte-identical in all four repositories, marks
`.github/`, `tests/`, `docs/` and `justfile` as `export-ignore`. GitHub's
archive endpoint honours that, and `common-heavy.yml:24-28` calls
`ossf/scorecard-action` without `file_mode`, whose default is `archive`.

```text
$ curl -L .../maestro-governance/tarball/ | tar tz | wc -l
20                                   # git ls-tree: 40
```

For `.github` itself the tarball carries 15 of 35 files and **not one of the
eleven reusable workflows**. Same version, same repository, only the mode
differing:

```text
--file-mode archive          --file-mode git
aggregate: 4.6               aggregate: 6.8
 -1 Dangerous-Workflow         10 Dangerous-Workflow
  0 Dependency-Update-Tool     10 Dependency-Update-Tool
 -1 Pinned-Dependencies        10 Pinned-Dependencies
 -1 Token-Permissions          10 Token-Permissions
```

A control run on an unrelated repository scores identically in both modes, so
this is these repositories' `export-ignore`, not an upstream bug. Live CI
matches archive mode exactly.

**Why it matters:** the estate's only supply-chain scorecard evaluates the three
checks that would validate its central claims -- every action pinned, every
token least-privilege, no dangerous workflow patterns -- against an empty file
list, drops them as inconclusive, publishes a false 4.6 with
`publish_results: true`, and files a permanent High alert saying no dependency
update tool is configured in repositories whose `dependabot.yml` is a tracked
baseline item.

**Fix:** `file_mode: git`. Separately reconsider `tests/` and `docs/` under
`export-ignore`: a release tarball without tests is not verifiable.

### 9. The prose gate exits 0 when its rules file is missing

`common-fast.yml:91` -- `done < .shared-policy/prose-rules.txt`, under
`set -uo pipefail` with no `set -e`:

```text
=== rules file PRESENT, violation present ===   exit=1
=== rules file MISSING ===                      No banned wording.  exit=0
=== rules file present but EMPTY ===            No banned wording.  exit=0
```

The redirect fails, the loop body never runs, `status` stays 0, and the job
prints success. This gate runs in all four repositories and exists, per its own
comment, because *"one repository carried the rule, a sibling named a sink
implementation in three files, and nothing looked."*

**Fix:** assert the file exists and is non-empty before the loop.

### 10. `actions-security` runs zizmor offline, so a lying version comment passes

zizmor is invoked without network access, so the audit that would notice a
pinned SHA whose version comment does not match never runs. The pinning policy
also cannot see the `$/` local reference form this repository's own `ci.yml`
uses, and accepts any ref for the organisation.

**Fix:** give the job the token zizmor needs for its online audits, and extend
the policy to the local reference form.

### 11. The release pipeline has never executed and holds the estate's only write permissions

`rust-release.yml` -- 139 lines, `contents: write`, `id-token: write`,
`attestations: write` -- has zero callers and zero runs. No repository has a
tag. Its CycloneDX SBOM describes one crate of a workspace, chosen by
filesystem order, and it builds a Linux-only binary for repositories whose
headline claim is three platforms.

Meanwhile all three siblings' `CHANGELOG.md:8` states *"This file is generated
by release-please"*, and `baseline.txt:124` treats their identity as evidence
the pipeline holds them in step. Nothing generates them; they are identical
because none has been touched.

**Fix:** wire it up with the App token it needs, or delete the workflow, the six
release-please files and the four baseline lines, and rewrite the CHANGELOG
headers.

### 12. `common / dependency-review` is a required context that has never reviewed a dependency

Required in all three Rust repositories. The job log, verbatim:
`Dependency review did not detect any vulnerable packages`. It reads the pull
request's dependency manifest diff; on a Rust repository with no manifest
change it has nothing to look at, which is every run so far.

**Fix:** keep it, and stop counting it as coverage.

### 13. Smaller, verified

- Five of ten reusable workflows have no caller; PR #21 adds ninety lines to
  three of them.
- `rust-coverage.yml` has no callers and is duplicated inside `rust-heavy.yml`.
- Two phantom workflows are registered on GitHub and cannot be opened.
- `ts-heavy.yml` documents its coverage job as gated; it enforces nothing.
- `taplo` silently falls back to default formatting when the shared config is
  missing.
- `python-fast.yml` declares an input no step reads.
- Two shared hooks run Cargo in every repository regardless of language.
- `prose-rules.txt` contains non-ASCII in a file governed by the estate's
  English-only rule.
- ADR-0005 names an enforcing tool for Python and TypeScript; neither exists.
- This repository calls the shared workflow under a job name none of the
  required contexts match.
- The WSL job in PR #21 is the only unverified remote code execution in the
  estate's CI.
