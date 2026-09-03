# Contributing

## The shape of a change

`main` takes changes only through a pull request, with the checks green.
Direct pushes are refused by the branch ruleset, for the maintainer too.

```text
git switch -c <topic>
just check          # the same gates CI will run
git push -u origin <topic>
gh pr create --fill
gh pr merge --squash --delete-branch
```

## Before writing code

Write the failing test first, and watch it fail for the reason you expect.
A test that passes the moment you write it has proved nothing: it may be
testing the implementation you just wrote rather than the behaviour you
wanted.

## Gates

`just check` runs everything CI does. If a gate blocks something you believe
is correct, say so in the pull request rather than working around it -- a gate
that gets bypassed quietly is worse than no gate, because it still reports
green.

Two gates have an allowlist rather than a threshold: accepted code duplication
and retired vocabulary. Adding an entry is allowed and expected; adding one
without the reason is not, and a second test fails when an entry stops being
true so the list cannot rot into excuses.

## Prose

English only, enforced by a test. No path anchored to one machine anywhere --
in code, configuration, task runner or workflow -- enforced by the
`no-absolute-paths` gate in the fast tier.

That second gate is newer than the sentence describing it. The rule was
written down in three repositories and checked by nothing for as long as it
had been quoted in reviews, which is how a documented gate rots: believed, and
never run.

What it refuses is a path that names a machine -- a home directory, a drive
letter, a user profile. What it allows, deliberately, is `/usr`, `/opt`,
`/etc`, `/var` and `/tmp`: those name a platform convention rather than a
machine, and a gate that fails on `#!/bin/sh` is one people learn to skip.
In a test, use a synthetic root such as `/somewhere`.
