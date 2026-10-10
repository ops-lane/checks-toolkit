# Checks Toolkit — Roadmap and status

This page is the honest answer to "what works today, how well is it
verified, and what is next". For the reasoning behind decisions see
[decisions.md](./decisions.md); for how the pieces fit see
[architecture.md](./architecture.md).

The project is **pre-1.0**: the interface (inputs, outputs, check names) can
still change between versions.

## Phases

| Phase | Goal | State |
|---|---|---|
| 1. Foundation | Change detection, the `ci.yml` workflow, naming, the gate | Done |
| 2. CI | Lint, typecheck and build per capability | Done |
| 3. Security | Secrets, SAST, dependency, image and Terraform scans, severity policy | Done, except scheduled full scans |
| 4. Merge gate | Central Mergify configuration with `CI gate` and `Security gate` as the only required checks | Next |
| 5. Deployment | Registry, environments, OIDC, separate deploy runners | Later |
| 6. Maturity | Version signalling for consumers, metrics (changelog and release automation exist: see below) | Partly done |

## What exists

| Capability | CI (`ci.yml`) | Security (`security.yml`) |
|---|---|---|
| Python | Ruff, mypy | pip-audit |
| Node | ESLint, `tsc` | `npm audit`, `pnpm audit` |
| Docker | `docker build` | Trivy image scan |
| Terraform | `terraform fmt -check` | Trivy misconfiguration scan |
| Markdown | markdownlint, codespell, lychee | — |
| Whole repository | — | Gitleaks (always), Semgrep, CodeQL (off by default, switch on with `codeql: true`) |
| This repository's own workflows | actionlint, zizmor (`check-workflows.yml`, not part of `ci.yml`) | — |
| This repository's own commits | Conventional Commits on every pull request (`check-commits.yml`) | — |

Plus the building blocks around them: `detect-changes` (which technologies
are present, what changed, per component), `ci-gate` (one verdict and a
result table per workflow), the `skip-ci` label, and the severity policy
(`fail-on-severity`, default `high`). See [workflows.md](./workflows.md) and
[security.md](./security.md).

## How well is it verified?

| Level | What it covers |
|---|---|
| Run on GitHub | Everything in `ci.yml` and `security.yml` except CodeQL. A test consumer (`checks-toolkit-test`) calls both workflows over 20+ scenarios, each with an expected result: the right check fails for the right reason, the others pass, a hostile folder name is treated as text, a secret added and removed in a later commit is found without its value being printed, the `skip-ci` label skips everything and its removal restores the checks, and the two gate comments stay separate. The platform's self-test covers the Markdown matrix, the full-history secret scan and the new-branch fallback |
| Not yet seen | A pull request from a fork (needs a public repository), a caller that triggers on `pull_request_target`, and `fail-on-findings: true` on `audit-python` |
| Not run at all | **CodeQL.** It is off by default because it needs a public repository or GitHub Advanced Security; only its result mapping was tested. Semgrep, the other half of `scan-code`, has run locally |

Each action's own README says exactly what was verified for it. Closing the
"static only" and "not run at all" rows is the next piece of work.

## Releases

Releases are exact tags, never moved, each with a GitHub Release. They are
published by the `Release` workflow, started by hand with a typed version; the
notes are generated from Conventional Commit subjects by git-cliff. See
[contributing.md](./contributing.md#releasing). The pipeline itself has not
been run yet: a dry run is the first step.

## Next

1. Run both workflows end to end on a test consumer repository, including a
   deliberately failing change per tool and the `skip-ci` label.
2. First real CodeQL run, once the repository is public: switch it on with `codeql: true`.
3. Phase 4: central Mergify configuration.
4. Scheduled (nightly) full scans: CodeQL, dependencies and container images
   across the whole repository instead of per pull request.
5. Offer `lint-workflows` and `scan-workflows` to consumer repositories through `ci.yml` (needs a `detect-changes` signal for workflows), and switch `zizmor` from report-only to blocking now that this repository's own baseline is clean.
6. Self-hosted runners (see [runners.md](./runners.md)). The platform runs on
   GitHub-hosted runners today; self-hosted is a later, deliberate step with
   its own isolation requirements.

## Not planned for now

Kubernetes, multi-cloud deployment, automatic production deployment, policy
engines, automatic code remediation, support for every language. The full
list is in [architecture.md](./architecture.md#non-goals).

## Known limitations

- **yarn is not supported** by the Node actions (they warn and skip).
- **CodeQL needs a public repository** (or GitHub Advanced Security on a
  private one), so it is off by default and has to be switched on with
  `codeql: true`.
- **A repository cannot allow-list its own secret-scanner false positives.**
  The scanned repository's Gitleaks configuration is deliberately ignored, so
  a pull request cannot hide its own findings. A per-repository allowlist
  would have to be read from the base branch.
- **`audit-python` has no severity.** pip-audit does not report one, so it is
  all-or-nothing and report-only by default.
- **No Terraform dependency audit.** OSV-Scanner cannot scan Terraform.
- **Linting and Docker builds execute code from the pull request** (an ESLint
  config file, a Dockerfile's `RUN` steps). That is acceptable on ephemeral
  GitHub-hosted runners; it must be revisited before running pull requests
  on self-hosted runners.
- **Docker images are built twice** (once in each workflow), because
  separate workflows cannot share a local image.
- **No per-component failure policy.** A blocking check blocks for every
  component of that type.
- **The root component over-matches in monorepos.** A change inside a nested
  component can also mark a root-level component of the same type as changed.
  It over-triggers and never under-triggers.
- **`check-workflows.yml` can be skipped by a pull request that changes it.** GitHub runs the workflow file from the pull request, so only branch protection with required checks closes this; branch protection is not available yet on the free plan for a private repository.
- **`terraform validate` is not implemented**; it needs provider downloads and
  its own supply-chain design.
