# Checks Toolkit — Workflow interface (technical contract)

This is the platform's API contract. Anything inside it may change freely
internally; anything outside it is a breaking change and requires a new
major version.

## Detection output schema

```json
{
  "project": {
    "python": true,
    "node": false,
    "docker": true,
    "terraform": true,
    "markdown": true,
    "workflows": true
  },
  "changed": {
    "python": false,
    "node": false,
    "docker": false,
    "terraform": true,
    "markdown": true,
    "workflows": true
  },
  "components": [
    { "path": "backend", "type": "python", "changed": false },
    { "path": "frontend", "type": "node", "changed": true },
    { "path": ".", "type": "workflows", "changed": false }
  ]
}
```

`components` has one entry per directory that holds a technology, which is
what makes monorepos work. A possible later extension is a `capabilities`
array per component (e.g. `["lint", "typecheck"]`).

## Detection signals

| Capability | Signals |
|---|---|
| Python | `pyproject.toml`, `requirements.txt`, `setup.py`, `Pipfile`, (weaker: `*.py`) |
| Node | `package.json`, `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock` |
| Docker | `Dockerfile`, `compose.yml`, `docker-compose.yml` |
| Terraform | `*.tf` |
| Markdown | `*.md` |
| Workflows | `.github/workflows/*.yml`, any `action.yml`, `.github/dependabot.yml` |

`workflows` is a single component for the whole repository (`path: "."`), not one per directory, because GitHub only runs workflows from `.github/workflows` of the repository root. It is present when that folder has a workflow file or the repository has a composite `action.yml`.

No weighting in v1: any signal (weak or strong) sets the capability to
`true`. See [decisions.md](./decisions.md), row "`detect-changes` signal
matching (v1)".

## `detect-changes` action interface

Standalone-callable composite action (see
[contributing.md](./contributing.md#actions-policy)) — works whether
called from `ci.yml`/`security.yml` or on its own.

**Inputs** (both optional):

| Input | Default | Purpose |
|---|---|---|
| `base-ref` | Derived from `github.event_name`/`github.event` (PR base / previous push commit) | What to diff against |
| `head-ref` | `github.sha` | What to diff |

**Outputs**:

| Output | Type | Description |
|---|---|---|
| `result` | JSON string | Matches the [Detection output schema](#detection-output-schema) above in full — read it with `fromJSON(steps.<id>.outputs.result)` |

A single JSON output (rather than one flattened output per capability) so
adding a new ecosystem never changes the action's interface — see
[decisions.md](./decisions.md), row "`detect-changes` output shape".

## Capability → CI mapping

```mermaid
flowchart LR
    Python --> Ruff
    Python --> mypy
    Python --> CodeQL_py[CodeQL]
    Python --> PipAudit[pip-audit]
    Python --> Semgrep_py[Semgrep]
    Node --> ESLint
    Node --> Typecheck
    Node --> CodeQL_js[CodeQL]
    Node --> NpmAudit["npm/pnpm audit"]
    Node --> Semgrep_js[Semgrep]
    Docker --> Build[docker build]
    Docker --> Trivy
    Docker --> Semgrep_docker["Semgrep (Dockerfile lint)"]
    Terraform --> Fmt["terraform fmt -check"]
    Terraform --> Validate[terraform validate]
    Terraform --> TrivyConfig["trivy config (misconfig)"]
    Markdown --> Markdownlint
    Workflows --> Actionlint[actionlint]
    Workflows --> Zizmor[zizmor]
```

`pip-audit` and `npm`/`pnpm audit` are each capability's native SCA tool
(yarn is not supported yet). Docker's dependency exposure is covered by
Trivy's image scan, so it gets no separate SCA step. Terraform has no
dependency audit: OSV-Scanner, the planned fallback, cannot scan Terraform
at all — see [decisions.md](./decisions.md), row "`audit-terraform` /
OSV-Scanner for Terraform dropped". The platform does not run a
consumer's own tests — see row "No code-level tests in the platform".

**Semgrep** runs alongside CodeQL on Python/Node and on Docker (its
Dockerfile rules, inside `scan-code`) — see [decisions.md](./decisions.md), row
"Semgrep adoption". **Terraform misconfiguration** is covered by Trivy's
`trivy config` rather than a dedicated tool — see row "Terraform
misconfiguration scanning".

## Actions overview

Every action below lives at `actions/<name>/action.yml` in this repo,
named per the `<type>-<target>` convention in
[contributing.md](./contributing.md#naming). `ci.yml` calls the `lint-*`,
`typecheck-*` and `build-*` actions, `security.yml` the `audit-*` and
`scan-*` ones. A few are standalone and not part of either workflow; they are
marked as such below, and this repository runs them from its own workflows. Every action listed here is implemented and has its own
README in its folder; see each README for what has been verified and what
has not.

| Action | Type | Target | Tools inside | Implemented | Build approach |
|---|---|---|---|---|---|
| `detect-changes` | — | — | See [Detection output schema](#detection-output-schema) above | ✅ | Built in-house (see decisions.md) |
| `ci-gate` | — | — | Evaluates job results, builds the per-step result table (summary + PR comment), enforces the verdict; used by both the `CI gate` and the `Security gate` | ✅ | Built in-house — see decisions.md |
| `lint-python` | lint | python | Ruff | ✅ | Own CLI call — `pip install ruff` (or `uvx ruff`) + `ruff check`, no Action needed |
| `typecheck-python` | typecheck | python | mypy | ✅ | Own CLI call — pinned `pip install mypy`, project dependencies deliberately not installed. See [its README](../actions/typecheck-python/README.md) |
| `audit-python` | audit | python | pip-audit | ✅ | Own CLI call — `pip install pip-audit` + `pip-audit`, no Action needed |
| `lint-node` | lint | node | ESLint (the project's own) | ✅ | Own CLI call — dependencies installed with `--ignore-scripts`, then the project's ESLint. See [its README](../actions/lint-node/README.md) |
| `typecheck-node` | typecheck | node | tsc (the project's own) | ✅ | Own CLI call — dependencies installed with `--ignore-scripts`, then `tsc --noEmit`. See [its README](../actions/typecheck-node/README.md) |
| `audit-node` | audit | node | npm audit / pnpm audit (matching detected lockfile) | ✅ | Own CLI call — the package manager itself, lockfile only, nothing installed. See [its README](../actions/audit-node/README.md) |
| `build-docker` | build | docker | `docker build` | ✅ | Own CLI call — Docker is preinstalled on GitHub-hosted runners. See [its README](../actions/build-docker/README.md) |
| `scan-docker` | scan | docker | Trivy image scan | ✅ | Official Trivy container image, pinned by digest. See [its README](../actions/scan-docker/README.md) |
| `lint-terraform` | lint | terraform | `terraform fmt -check` (done) + `terraform validate` (not yet) | ✅ | Own CLI call — `hashicorp/setup-terraform@v4` (official, HashiCorp) just to install a pinned CLI version, then plain `terraform` commands. See [its README](../actions/lint-terraform/README.md) |
| `scan-terraform` | scan | terraform | Trivy config (misconfig) | ✅ | Official Trivy container image, pinned by digest. See [its README](../actions/scan-terraform/README.md) |
| `lint-markdown` | lint | markdown | markdownlint-cli2 + codespell + lychee | ✅ | Own CLI calls — `npx markdownlint-cli2`, `pip install codespell`, lychee via its Docker image; no Action needed. See [its README](../actions/lint-markdown/README.md) |
| `scan-secrets` | scan | secrets | Gitleaks (always-on) | ✅ | Official Gitleaks container image, pinned by digest. See [its README](../actions/scan-secrets/README.md) |
| `lint-pr-title` | lint | pull request title | A regular expression over the title (Conventional Commits) | ✅ | Own CLI call. Used by `check-pr-title.yml` on this repository's own pull requests, not by `ci.yml`. See [its README](../actions/lint-pr-title/README.md) |
| `lint-commits` | lint | commits | A regular expression over `git log` (Conventional Commits) | ✅ | Own CLI call. Used by `check-commits.yml` on this repository's own pull requests, not by `ci.yml`. See [its README](../actions/lint-commits/README.md) |
| `lint-workflows` | lint | workflows | actionlint | ✅ implemented, not yet run on GitHub | Own CLI call — pinned release tarball, SHA-256 verified before use; no third-party Action. See [its README](../actions/lint-workflows/README.md) |
| `scan-workflows` | scan | workflows | zizmor (offline) | ✅ implemented, not yet run on GitHub | Own CLI call — `pip install zizmor==<pinned>`; no third-party Action. Scans workflows, composite actions and `dependabot.yml`. See [its README](../actions/scan-workflows/README.md) |
| `lint-action-docs` | lint | action docs | A short Python script (PyYAML) | ✅ | Own script. Used by `check-repo.yml` on this repository, not by `ci.yml`. See [its README](../actions/lint-action-docs/README.md) |
| `lint-codeowners` | lint | codeowners | GitHub's codeowners errors API | ✅ | Own CLI call (`gh api`). Used by `check-repo.yml`, not by `ci.yml`. See [its README](../actions/lint-codeowners/README.md) |
| `lint-shell` | lint | shell | ShellCheck | ✅ | Official ShellCheck container image, pinned by digest. Checks `*.sh` files and the `run:` blocks of composite actions. Used by `check-repo.yml`, not by `ci.yml`. See [its README](../actions/lint-shell/README.md) |
| `lint-yaml` | lint | yaml | yamllint | ✅ | Own CLI call — `pip install yamllint==<pinned>`. Used by `check-repo.yml`, not by `ci.yml`. See [its README](../actions/lint-yaml/README.md) |
| `audit-dependencies` | audit | dependencies | GitHub's dependency review | ✅ | Wraps the official `actions/dependency-review-action` (GitHub), pinned by SHA. Used by `dependency-review.yml`, not by `security.yml`. See [its README](../actions/audit-dependencies/README.md) |
| `scan-licenses` | scan | licenses | Trivy (license scanner) | ✅ | Official Trivy container image, pinned by digest. Standalone, not part of `security.yml`. See [its README](../actions/scan-licenses/README.md) |
| `build-sbom` | build | sbom | GitHub's dependency graph, or Trivy | ✅ | Own CLI call (`gh api`) or the official Trivy container image. Used by `release.yml`. See [its README](../actions/build-sbom/README.md) |
| `scan-code` | scan | code | CodeQL + Semgrep, one combined run across whichever languages are present — see [decisions.md](./decisions.md), row "CodeQL/Semgrep bundling" | ✅ | `github/codeql-action` (GitHub itself, pinned to a commit) + the official Semgrep container image, pinned by digest. See [its README](../actions/scan-code/README.md) |

The tools themselves run as official container images pinned by version and digest (not as third-party Actions) wherever that is possible — see [decisions.md](./decisions.md), row "Security tools run as pinned container images, not third-party Actions".

## `check-workflows.yml` (the platform checks itself)

`.github/workflows/check-workflows.yml` runs `lint-workflows` and
`scan-workflows` against this repo's own `.github/workflows`, `actions/`
and `dependabot.yml`. It is deliberately **not** part of `ci.yml`: if
`ci.yml` has a syntax error it never starts, so it cannot report its own
mistake. The checker therefore has its own trigger (a `paths:` filter on
`pull_request` and `push` to `development`, plus `workflow_dispatch`) and
does not use `detect-changes`. Only `actionlint` blocks; `zizmor` reports
until its known findings are resolved (see
[decisions.md](./decisions.md), row "Workflow checks").

`ci.yml` also offers the same two checks to consumer repositories through
the `workflows` capability of `detect-changes`: when a pull request changes a
workflow, an `action.yml` or `dependabot.yml`, the `workflows` job runs
`lint-workflows` (blocking) and `scan-workflows` (report only, unless
`fail-on-findings` is on). Not yet run on GitHub — see
[roadmap.md](./roadmap.md).

## This repository's other workflows

Besides `ci.yml`, `security.yml` and `check-workflows.yml`, the repository
runs a few workflows only on itself. None of them is part of the interface a
consumer calls.

| Workflow | What it does |
|---|---|
| `check-repo.yml` | `lint-action-docs`, `lint-codeowners`, `lint-shell` and `lint-yaml` on the repository's own files |
| `check-commits.yml` | `lint-commits`: every commit follows Conventional Commits |
| `check-pr-title.yml` | `lint-pr-title`: the pull request title follows Conventional Commits |
| `dependency-review.yml` | `audit-dependencies`: a pull request may not add a vulnerable dependency |
| `labeler.yml` | Labels a pull request from the paths it changes (`.github/labeler.yml`) |
| `scorecard.yml` | OpenSSF Scorecard, weekly and on `development`; results go to code scanning |
| `stale.yml` | Marks and closes issues and pull requests that have gone quiet |
| `release.yml` | Publishes a version with its release files and attestations; see [security.md](./security.md#verifying-a-release) |

## Workflow interface for consumer repos

```yaml
on:
  pull_request:
    # labeled/unlabeled: so adding or removing the skip-ci label starts a
    # new run. Without them the label only takes effect on the next push.
    types: [opened, synchronize, reopened, labeled, unlabeled]

permissions:
  contents: read

jobs:
  ci:
    permissions:
      contents: read
      actions: read          # the gate's result table reads the run's jobs
      pull-requests: write   # ... and posts/updates the PR comment
    uses: ops-lane/checks-toolkit/.github/workflows/ci.yml@v0.1.0
    with:
      platform-ref: v0.1.0   # must match the @ref above, see below

  security:
    permissions:
      contents: read
      actions: read
      pull-requests: write
    uses: ops-lane/checks-toolkit/.github/workflows/security.yml@v0.1.0
    with:
      platform-ref: v0.1.0
```

Things every caller has to know:

- **`platform-ref` is required and must equal the `@ref` in `uses:`.** A reusable workflow can't see which ref of itself it was called with, so it needs the value to check out its own actions (see `decisions.md`, row "Calling `ci.yml`'s actions from an external repo").
- **`platform-read-token`** is an optional secret. It is only needed if `checks-toolkit` is private (for example a private copy): a token with read access to it, because the caller's default token has none. Not needed for the public repository, so the example above leaves it out.
- **Permissions** above are the minimum, granted on the calling job rather than the whole workflow (least privilege; `zizmor` flags the broad form): a called workflow can't get more than its caller grants, and the gate jobs request `actions: read` and `pull-requests: write` — a caller granting less has its whole run stopped at startup. The scans themselves only need `contents: read`.
- **Optional inputs.** `ci.yml`: `fail-on-findings` (fail on zizmor findings and on broken Markdown links). `security.yml`: `fail-on-severity` (default `high`; critical and high findings block, lower ones are reported), `fail-on-findings` (make `audit-python` fail, since pip-audit has no severity) and `codeql` (default `false`: CodeQL only works on a public repository or one with GitHub Advanced Security, so switch it on yourself; Semgrep always runs). Leaked secrets always block.
- **Two required checks.** Make `CI gate` and `Security gate` the required checks, not the individual jobs.
- **Trigger only on `pull_request`, `push` or `workflow_dispatch`.** Both workflows refuse to run on `pull_request_target` (they would run a fork's code with write access and secrets) — see [security.md](./security.md#public-repository-and-fork-pull-requests).

## Checks vs. outputs (important)

Workflow outputs (`workflow.outputs`) are not the same as GitHub checks.
Mergify evaluates PRs based on check runs/statuses, not on
`workflow.outputs`. The contract is therefore:

Consumer repositories get exactly two required checks:

```mermaid
flowchart TD
    CI["CI gate → success / failure"]
    Security["Security gate → success / failure"]
```

Internally, these workflows may contain as many jobs as needed:

```mermaid
flowchart TD
    CIw[CI] --> Detect
    CIw --> Python
    CIw --> Node
    CIw --> Docker
    CIw --> Terraform
    CIw --> Markdown
    CIw --> CIGate[CI gate]

    Secw[Security] --> Secrets["secrets (Gitleaks, always)"]
    Secw --> Code["code (CodeQL + Semgrep)"]
    Secw --> Deps["python / node audit"]
    Secw --> Images["docker (build + Trivy)"]
    Secw --> TF["terraform (Trivy config)"]
```

Mergify's required checks: only the two gate jobs (`CI gate`, `Security
gate`), never the individual sub-jobs — otherwise a skipped job would block
the merge. The gate jobs always run, even when everything else was skipped.

## Skip-CI label

A `skip-ci` PR label lets a maintainer deliberately override CI entirely,
including security (Gitleaks, CodeQL, Trivy, dependency scans). It is a
deliberate, full override — not a selective skip.

Design rules:

- The label always wins over change detection: `skip-ci` > detected changes.
- Both workflows check the pull request labels in their `detect` job. When
  the label is set, every other job is skipped through its `if:`; the gate
  job still runs. The gate is the required check, so it always starts and
  always finishes: a required check that never starts would stay "pending"
  forever and block the merge, which is the problem this design avoids.
- The skip reason is shown visibly in the gate's job summary and PR comment
  ("skipped via skip-ci label … a green check here does NOT mean the change
  was tested"), so a green check is never mistaken for "tested".
- Only the `pull_request` event can skip. A push or a manual run never does.
- The consumer's `pull_request` trigger must list the `labeled` and
  `unlabeled` types (see the snippet above), otherwise adding the label does
  not start a new run until the next push.
- Because `skip-ci` also skips security, only maintainers may set the
  label — this prevents contributors from bypassing security checks
  themselves. Setting that restriction is a repository/Mergify setting; the
  workflow cannot enforce it.
- Mergify can also use this label directly as a merge condition,
  separate from the individual required checks, as an extra explicit
  override.

```mermaid
flowchart TD
    PR[Pull request] --> Label{"skip-ci label set?"}
    Label -- yes --> NoOp["All jobs skipped, the gate runs<br/>and passes (reason logged)"]
    Label -- no --> Detect[detect-changes] --> Normal[Normal conditional execution]
```

## Mergify configuration

```yaml
# in a consumer repo's .mergify.yml
extends: ops-lane/checks-toolkit
```

```mermaid
flowchart LR
    Consumer["Consumer repo<br/>.mergify.yml"] -->|extends| Platform["checks-toolkit<br/>Mergify config"]
```

The actual `pull_request_rules`/`queue_rules` live centrally in
`checks-toolkit` (Phase 4, see [roadmap.md](./roadmap.md)). Still to verify before
implementation: whether Mergify's shared/extended configuration feature
still works as expected under the current plan.
