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

English only, enforced by a test. No absolute paths anywhere -- in code,
configuration, task runner or workflow. Both rules are mechanical, and both
have caught real regressions.
