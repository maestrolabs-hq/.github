# Graph Lifecycle Hooks Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish two centrally governed, shell-free Git lifecycle hooks that ask Maestro to durably schedule repository-graph refresh work after commits and before pushes, then canary and hand the immutable revision to every governed consumer.

**Architecture:** `.pre-commit-hooks.yaml` remains the shared manifest. Two distinct IDs call the approved hidden Maestro facade with an explicit lifecycle mode; pre-commit supplies no filenames and Maestro derives repository context from the working directory. Maestro owns the acknowledgement deadline and post-commit warning/pre-push blocking behavior. A tiny Python standard-library test validates the safety-critical manifest fields, and common-fast runs only that static test.

**Tech Stack:** pre-commit manifests, Python 3 standard library (`pathlib`, `re`, `unittest`), existing GitHub Actions common-fast workflow, immutable Git commit revisions.

**Spec:** `../maestro-core/docs/superpowers/specs/2026-09-01-workspace-intelligence-facades.md` (Approved)

## Global Constraints

- Work only in `dot-github`; governance owns every consumer repository edit after handoff.
- Publish exactly `maestro-graph-post-commit` and `maestro-graph-pre-push`; never reuse a memory hook ID.
- Both use `entry: maestro`, `language: system`, `always_run: true`, and `pass_filenames: false`. Post-commit args are exactly `[__hook, graph-refresh, --mode, post-commit]`; pre-push args are exactly `[__hook, graph-refresh, --mode, pre-push]`.
- Do not add a shell wrapper, pipes, redirection, `timeout`, `|| true`, filenames, repository paths, or workspace paths.
- Do not call `maestro memory ingest`. Graph lifecycle signals and memory ingestion are separate protocol domains.
- Do not encode failure policy in the manifest. Maestro warns and exits zero for post-commit enqueue failure; it reports and exits nonzero for pre-push enqueue failure.
- Hook success means durable scheduling acknowledgement only. Neither hook waits for indexing, parsing, merging, embedding, synchronization, or provider completion.
- Do not add graph lifecycle/indexing workflows. CI only validates the static publication contract.
- Write each focused failing test first, observe RED, make the smallest production change, run GREEN, then commit. Tasks 1-2 are one RED/GREEN cycle and produce one commit only after GREEN; never commit the deliberately failing test by itself.
- Consumers pin a full 40-character commit ID, never `main`, a branch, or a movable tag.

---

### Task 1: Specify the shell-free manifest contract

**Files:**

- Create: `.github/tests/test_graph_hook_manifest.py`

**Interfaces:**

- Consumes: the top-level sequence in `.pre-commit-hooks.yaml`.
- Produces: a dependency-free test for the exact IDs, executable, argument vectors, modes, stages, and filename behavior.
- Proves: no memory-ingestion command or shell control token can enter either graph definition.

- [ ] **Step 1: Write the failing standard-library test**

Create `.github/tests/test_graph_hook_manifest.py`:

```python
"""Lock graph lifecycle hooks to the reviewed shell-free facade."""

from pathlib import Path
import re
import unittest


ROOT = Path(__file__).resolve().parents[2]
MANIFEST = ROOT / ".pre-commit-hooks.yaml"
EXPECTED = {
    "maestro-graph-post-commit": {
        "args": ["__hook", "graph-refresh", "--mode", "post-commit"],
        "stages": ["post-commit"],
    },
    "maestro-graph-pre-push": {
        "args": ["__hook", "graph-refresh", "--mode", "pre-push"],
        "stages": ["pre-push"],
    },
}


def scalar(block: str, key: str) -> str:
    match = re.search(rf"(?m)^  {re.escape(key)}: ([^\n]+)$", block)
    if match is None:
        raise AssertionError(f"missing {key}")
    return match.group(1).strip()


def inline_list(block: str, key: str) -> list[str]:
    value = scalar(block, key)
    if not (value.startswith("[") and value.endswith("]")):
        raise AssertionError(f"{key} must be an inline list")
    return [item.strip() for item in value[1:-1].split(",") if item.strip()]


def hook_blocks(text: str) -> dict[str, str]:
    blocks: dict[str, str] = {}
    pattern = r"(?ms)^- id: ([^\n]+)\n(.*?)(?=^- id: |\Z)"
    for match in re.finditer(pattern, text):
        blocks[match.group(1).strip()] = match.group(2)
    return blocks


class GraphHookManifestTest(unittest.TestCase):
    def test_graph_hooks_are_exact_shell_free_facades(self) -> None:
        blocks = hook_blocks(MANIFEST.read_text(encoding="utf-8"))
        self.assertEqual(EXPECTED.keys(), EXPECTED.keys() & blocks.keys())

        for hook_id, expected in EXPECTED.items():
            with self.subTest(hook_id=hook_id):
                block = blocks[hook_id]
                self.assertEqual("maestro", scalar(block, "entry"))
                self.assertEqual("system", scalar(block, "language"))
                self.assertEqual("true", scalar(block, "always_run"))
                self.assertEqual("false", scalar(block, "pass_filenames"))
                self.assertEqual(expected["args"], inline_list(block, "args"))
                self.assertEqual(expected["stages"], inline_list(block, "stages"))
                command = " ".join(["maestro", *inline_list(block, "args")])
                self.assertNotIn("memory ingest", command)
                self.assertIsNone(re.search(r"(?:\|\||&&|[|;<>])", command))


if __name__ == "__main__":
    unittest.main()
```

The parser deliberately supports only the repository's two-space mappings and inline lists. Do not add PyYAML for two entries.

- [ ] **Step 2: Run RED**

```bash
python3 -m unittest .github/tests/test_graph_hook_manifest.py -v
```

Expected: `test_graph_hooks_are_exact_shell_free_facades` fails because both expected IDs are absent.

- [ ] **Step 3: Keep RED uncommitted and continue directly to Task 2**

Do not commit a deliberately failing repository state. Task 2 adds the manifest,
runs the complete GREEN checks, and commits the test and implementation
together as one reviewable RED/GREEN change.

---

### Task 2: Publish the hooks and validate the manifest in common-fast

**Files:**

- Modify: `.pre-commit-hooks.yaml`
- Modify: `.github/workflows/common-fast.yml`

**Interfaces:**

- Consumes: `maestro __hook graph-refresh --mode post-commit|pre-push`, implemented and behavior-tested by the approved core runtime plan.
- Produces: one post-commit notification and one pre-push idempotent backstop.
- Preserves: every existing quality hook and workflow job.

- [ ] **Step 1: Add the minimal definitions**

Append:

```yaml
- id: maestro-graph-post-commit
  name: Maestro graph refresh (post-commit notification)
  entry: maestro
  language: system
  args: [__hook, graph-refresh, --mode, post-commit]
  always_run: true
  pass_filenames: false
  stages: [post-commit]

- id: maestro-graph-pre-push
  name: Maestro graph refresh (pre-push backstop)
  entry: maestro
  language: system
  args: [__hook, graph-refresh, --mode, pre-push]
  always_run: true
  pass_filenames: false
  stages: [pre-push]
```

A single executable plus an argument vector is shell-free. `always_run` is required because the signal concerns repository state; `pass_filenames: false` prevents changed paths from being appended.

- [ ] **Step 2: Run focused GREEN**

```bash
python3 -m unittest .github/tests/test_graph_hook_manifest.py -v
```

Expected: one test passes.

- [ ] **Step 3: Wire the static test into common-fast**

Add this job to `.github/workflows/common-fast.yml` after `brief`:

```yaml
  hook-manifest:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - name: Shared hook manifest matches its reviewed facades
        run: python3 -m unittest .github/tests/test_graph_hook_manifest.py -v
```

This job does not install pre-commit or Maestro, invoke a lifecycle hook, enqueue work, or index a graph. It is not a lifecycle CI counterpart.

- [ ] **Step 4: Run GREEN for every changed surface**

```bash
python3 -m unittest discover -s .github/tests -p 'test_*.py' -v
npx --yes markdownlint-cli2@0.18.1 --config .markdownlint.jsonc "docs/**/*.md"
git diff --check
```

Expected: the test passes and both lint/whitespace commands are silent.

- [ ] **Step 5: Commit publication**

```bash
git add .github/tests/test_graph_hook_manifest.py .pre-commit-hooks.yaml .github/workflows/common-fast.yml
git commit -m "feat: publish graph lifecycle hooks"
git rev-parse HEAD | tee /tmp/maestro-graph-hooks-candidate.sha
test "$(wc -c < /tmp/maestro-graph-hooks-candidate.sha)" -eq 41
git show --stat --oneline "$(tr -d '\n' < /tmp/maestro-graph-hooks-candidate.sha)"
```

The saved object ID, not a tag or branch, is the publication candidate.

---

### Task 3: Canary the immutable revision in the `.github` repository

**Files:**

- Create: `.pre-commit-config.yaml`
- Create: `docs/governance/hook-install-evidence.txt`

**Interfaces:**

- Consumes: Task 2's 40-character candidate and an installed Maestro release containing the facade.
- Produces: the organisation `.github` repository's self-canary with all four governance-required Git stages installed and both lifecycle hooks selected.
- Produces: a reviewed install-evidence contract recording the exact candidate, configuration digest, installer version, path kind, validation result, and exact generated-byte digest for every installed stage without persisting an absolute path.
- Does not produce: graph CI work, a mutable pin, or a lifecycle wrapper.

- [ ] **Step 1: Establish RED**

```bash
test -f .pre-commit-config.yaml
```

Expected: exit 1 because this repository has no local pre-commit configuration.

- [ ] **Step 2: Verify and insert the exact revision**

```bash
revision=$(tr -d '\n' < /tmp/maestro-graph-hooks-candidate.sha)
test "${#revision}" -eq 40
git cat-file -e "${revision}^{commit}"
printf '%s\n' "$revision"
```

Create `.pre-commit-config.yaml` directly from the verified revision:

```bash
python3 - "$revision" <<'PY'
from pathlib import Path
import sys

revision = sys.argv[1]
Path(".pre-commit-config.yaml").write_text(
    "# Canary centrally published graph lifecycle hooks in the repository that owns them.\n"
    "default_install_hook_types: [pre-commit, pre-merge-commit, post-commit, pre-push]\n"
    "\n"
    "repos:\n"
    "  - repo: https://github.com/maestrolabs-hq/.github\n"
    f"    rev: {revision}\n"
    "    hooks:\n"
    "      - id: maestro-graph-post-commit\n"
    "      - id: maestro-graph-pre-push\n",
    encoding="utf-8",
)
PY
```

Prove the resulting pin is the exact candidate commit:

```bash
grep -Eq '^    rev: [0-9a-f]{40}$' .pre-commit-config.yaml
grep -Fx "    rev: $revision" .pre-commit-config.yaml
```

- [ ] **Step 3: Install stages and exercise CLI-owned behavior**

```bash
prek install --install-hooks
prek validate-config .pre-commit-config.yaml
prek run maestro-graph-post-commit --hook-stage post-commit -v
prek run maestro-graph-pre-push --hook-stage pre-push -v
```

Expected: both resolve from the pinned commit, receive no filenames, and return after durable acknowledgement without awaiting indexing.

From `maestro-core`, run the existing runtime-plan test:

```bash
cargo test -p maestro-cli --test hook_mode -- post_commit_warns_but_pre_push_blocks_when_commit_fails --exact
```

Expected: injected enqueue failure makes post-commit print `warning` and exit zero, while pre-push prints the durable-enqueue error and exits nonzero. The fixture also proves neither mode waits for indexing or invokes memory ingestion. Do not reproduce this policy in the manifest.

Generate the reviewed evidence from the exact installed bytes. The evidence is
lossless for governance purposes: every required stage is named separately and
bound to its generated bytes by SHA-256; no successful or failed observation is
collapsed into prose, and no machine path is persisted.

```bash
candidate="$revision" python3 - <<'PY'
from hashlib import sha256
from pathlib import Path
import os
import subprocess

stages = ["pre-commit", "pre-merge-commit", "post-commit", "pre-push"]
hooks = Path(subprocess.check_output(
    ["git", "rev-parse", "--git-path", "hooks"], text=True
).strip())
lines = [
    "schema=1",
    "repository=.github",
    f"candidate_revision={os.environ['candidate']}",
    f"configuration_sha256={sha256(Path('.pre-commit-config.yaml').read_bytes()).hexdigest()}",
    f"installer={subprocess.check_output(['prek', '--version'], text=True).strip()}",
    "hooks_path_kind=git-managed",
    "validation_command=prek validate-config .pre-commit-config.yaml",
    "validation_exit=0",
]
for stage in stages:
    path = hooks / stage
    if not path.is_file() or not os.access(path, os.X_OK):
        raise SystemExit(f"missing executable {stage} hook")
    data = path.read_bytes()
    lines.extend([
        f"hook.{stage}.executable=true",
        f"hook.{stage}.type={stage}",
        f"hook.{stage}.sha256={sha256(data).hexdigest()}",
    ])
Path("docs/governance").mkdir(parents=True, exist_ok=True)
Path("docs/governance/hook-install-evidence.txt").write_text(
    "\n".join(lines) + "\n", encoding="utf-8"
)
PY
```

The per-stage digest binds the exact generated bytes without copying the
machine-specific executable path embedded by the installer. Governance
validates this schema and pins the merged evidence blob; malformed, missing,
duplicate, non-executable, or mismatched stage records are drift.

- [ ] **Step 4: Commit the canary pin and evidence**

```bash
git add .pre-commit-config.yaml docs/governance/hook-install-evidence.txt
git commit -m "chore: canary graph lifecycle hooks"
```

Retain the earlier candidate ID in the config. The later canary commit consumes, but does not redefine, the immutable hooks.

- [ ] **Step 5: Observe before handoff**

Record five normal commits and five normal pushes in the rollout issue. Every hook must return within the CLI's hard acknowledgement deadline, none may wait for provider work, and no memory-ingestion operation may appear. An unexpected block or deadline breach stops rollout. Fix Maestro or publish and re-canary a new commit; never move the reviewed revision.

---

### Task 4: Publish the reviewed canary handoff

**Files:**

- Modify: `.github/tests/test_graph_hook_manifest.py`
- Create: `docs/governance/graph-hook-handoff.md`

**Interfaces:**

- Consumes: the immutable candidate revision and reviewed `docs/governance/hook-install-evidence.txt` from Task 3.
- Produces one handoff document containing the exact 40-character candidate revision, both hook IDs and argument vectors, canary commit, install-evidence SHA-256, observation window, and rollback rule.
- Does not edit or enumerate consumer repositories; `maestro-governance` owns remote fleet membership and rollout.

- [ ] **Step 1: Extend the test to require a complete handoff**

Add a test that reads `docs/governance/graph-hook-handoff.md`, extracts one lowercase 40-character candidate revision, and requires both hook IDs, both exact argument vectors, a canary commit, the install-evidence path and 64-character SHA-256, and the statement that lifecycle signals have no CI counterpart. Recompute the evidence digest in the test. Reject `main`, branch names, tags, abbreviated revisions, digest mismatch, and workspace membership claims.

- [ ] **Step 2: Run RED**

Run: `python3 -m unittest .github.tests.test_graph_hook_manifest -v`

Expected: the handoff test fails because `docs/governance/graph-hook-handoff.md` does not exist.

- [ ] **Step 3: Generate the handoff from reviewed repository bytes**

Read the literal candidate revision from the canary's `.pre-commit-config.yaml`, verify it resolves to the hook publication commit, and capture the canary commit:

```bash
candidate="$(python3 - <<'PY'
from pathlib import Path
lines = Path('.pre-commit-config.yaml').read_text(encoding='utf-8').splitlines()
for index, line in enumerate(lines):
    if line.strip() == 'repo: https://github.com/maestrolabs-hq/.github':
        print(lines[index + 1].split(':', 1)[1].strip())
        break
else:
    raise SystemExit('central hook repository entry missing')
PY
)"
test "${#candidate}" -eq 40
git cat-file -e "${candidate}^{commit}"
canary="$(git rev-parse HEAD)"
python3 - "$candidate" "$canary" <<'PY'
from pathlib import Path
import sys
candidate, canary = sys.argv[1:]
evidence = Path('docs/governance/hook-install-evidence.txt').read_bytes()
evidence_sha256 = __import__('hashlib').sha256(evidence).hexdigest()
Path('docs/governance').mkdir(parents=True, exist_ok=True)
Path('docs/governance/graph-hook-handoff.md').write_text(f'''# Graph lifecycle hook handoff

Candidate revision: `{candidate}`
Canary commit: `{canary}`
Install evidence: `docs/governance/hook-install-evidence.txt`
Install evidence SHA-256: `{evidence_sha256}`
Hooks: `maestro-graph-post-commit` and `maestro-graph-pre-push`
Arguments: `[__hook, graph-refresh, --mode, post-commit]` and `[__hook, graph-refresh, --mode, pre-push]`
Observation: five normal commits and five normal pushes completed within the acknowledgement deadline.
Lifecycle signals have no CI counterpart and perform no synchronous indexing.
Rollback: remove the consumer selections; never mutate or delete the candidate revision.
''', encoding='utf-8')
PY
```

- [ ] **Step 4: Run GREEN**

Run:

```bash
python3 -m unittest .github.tests.test_graph_hook_manifest -v
git diff --check
```

Expected: the manifest and handoff contracts pass, and the document contains repository-derived immutable revisions rather than placeholders.

- [ ] **Step 5: Commit the handoff**

```bash
git add .github/tests/test_graph_hook_manifest.py docs/governance/graph-hook-handoff.md
git commit -m "docs: publish graph hook canary handoff"
```

The governance rollout consumes this reviewed document; dot-github makes no consumer configuration change.

---

## Final Verification

In `dot-github`:

```bash
python3 -m unittest discover -s .github/tests -p 'test_*.py' -v
npx --yes markdownlint-cli2@0.18.1 --config .markdownlint.jsonc "docs/**/*.md"
git diff --check
git status --short
```

Confirm the approved core plan's `hook_mode` test evidence proves deadline-bounded acknowledgement, post-commit warning, pre-push blocking, no filenames, no indexing wait, and no memory ingestion. Hand `docs/governance/graph-hook-handoff.md` to the governance plan; do not modify consumers here.

## Rollback

Never mutate or delete the candidate revision. On canary failure, remove the two IDs from the consumer config, reinstall remaining stages, and commit that rollback. Fix the CLI or manifest on a new commit, repeat Tasks 1–3, and distribute the new object ID only after the complete canary window.

## Completion Evidence

- The smallest manifest test passes using only Python's standard library.
- The manifest has distinct shell-free IDs, explicit modes/stages, `always_run: true`, and `pass_filenames: false`.
- Static CI validates publication without invoking Maestro or indexing.
- The `.github` canary pins and installs the immutable candidate and publishes reviewed evidence for all four generated stages.
- Core tests prove CLI ownership of warning/blocking semantics and asynchronous acknowledgement.
- The handoff carries the full candidate SHA and canary evidence for governance to consume without treating local manifest membership as remote fleet authority.
