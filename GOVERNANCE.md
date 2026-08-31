# Governance

How decisions are made here, and what enforces them.

## Who decides

One maintainer. This is not a foundation and pretending otherwise would be
theatre. What follows describes how that maintainer is constrained, because a
single decision-maker without constraints is how estates rot.

## What is enforced by the platform

Nothing on this list depends on anyone remembering it. Organisation rulesets
apply to every repository, **including ones that do not exist yet**, and carry
no bypass actors -- the owner is subject to them like anyone else.

| Control | Scope |
| --- | --- |
| No force-push, no branch deletion | every repository |
| Version tags immutable | every repository, `refs/tags/v*` |
| Pull request required, squash only | every `tier=mature` repository |
| Required status checks | per repository, naming the fast-tier contexts |
| Actions pinned to a full SHA | organisation-wide |
| `GITHUB_TOKEN` read-only by default | organisation-wide |
| Two-factor authentication, secure methods | every member |

See [maestro-governance ADR-0004](https://github.com/maestrolabs-hq/maestro-governance/blob/main/docs/adr/0004-what-the-platform-enforces.md)
for what each control is for.

## How a change lands

1. A branch, and a pull request. Direct pushes to a default branch are
   refused by the platform, for the maintainer too.
2. The fast-tier checks pass. They are required contexts, so a red check
   blocks the merge rather than inviting a judgement call.
3. Squash merge. Rebase merging rewrites commits without re-signing them.

Zero approving reviews are required, because a single maintainer cannot
approve their own pull request and a nonzero count would deadlock every merge.
The gate is the checks, not a rubber stamp.

## How a decision is recorded

Anything that would otherwise have to be re-derived becomes an ADR in the
repository it governs. An ADR states what was decided, what it costs, and what
remains unresolved. A decision that lives only in a commit message is a
decision the next reader will make differently.

## Drift

`maestro-governance` holds the desired state of every repository and
organisation setting in `baseline.txt`, and audits it weekly. Drift opens a
tracking issue that closes itself when the estate matches again.

The audit reads with a token that cannot write. It reports; it does not
correct.

## Adding a repository

New repositories inherit the organisation floor, the `tier` custom property
and the security configuration on creation. What they still need: an entry in
`baseline.txt`, a repository ruleset naming their required contexts, and the
shared configuration files the baseline tracks.
