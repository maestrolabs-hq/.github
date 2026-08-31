# MaestroLabs

Agent orchestration, and the machinery that keeps it honest.

## Repositories

| Repository | What it is |
| --- | --- |
| [maestro-core](https://github.com/maestrolabs-hq/maestro-core) | the orchestrator agents work under: it delegates all the work and does none of it, while staying accountable for all of it |
| [maestro-pi-config](https://github.com/maestrolabs-hq/maestro-pi-config) | every post-install change to a pi installation, captured so a machine can be rebuilt |
| [maestro-governance](https://github.com/maestrolabs-hq/maestro-governance) | what the repositories should look like, and the drift from it |

## How the code is kept

Rust, no application dependencies, and gates that are allowed to fail.

Every repository runs the same set: formatting, `clippy` with pedantic lints as
errors, tests, unused-dependency and advisory checks, copy-paste detection
against a reasoned allowlist, module size, and an English-only rule. They run
locally as hooks and again in CI, where they cannot be skipped, and `main`
takes changes only through a pull request with those checks green.

The rule behind all of it: **a gate that cannot fail is not a gate.** Each one
here has been proved by injecting the fault it is supposed to catch and
watching it fail. Several were written after a green build hid a real bug, and
the commit messages say which.
