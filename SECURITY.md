# Security policy

## Reporting

Report a vulnerability privately through
[GitHub Security Advisories](https://github.com/maestrolabs-hq/maestro-core/security/advisories/new)
on the affected repository. Do not open a public issue.

Expect an acknowledgement within a week. These are personal projects, not a
funded product, and the response time reflects that honestly rather than
promising a service level nobody is on call for.

## What is in scope

Anything that lets code or configuration reach a machine without passing the
gates: a supply-chain path through the toolchain, a workflow that can be made
to run untrusted input with credentials, or a captured configuration that
leaks a secret.

## What is deliberately not covered

Credentials never enter these repositories. `auth.json`, GitHub tokens and
provider keys are excluded from capture by design, so a leak of this
repository leaks configuration, not access. Reports of that exclusion working
as intended are not findings.
