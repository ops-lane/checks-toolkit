# `lint-node`

Lints Node code with the project's own [ESLint](https://eslint.org/). A blocking check: the step fails on ESLint **errors**; warnings are counted but do not fail it. Findings are listed in the job summary.

**Origin:** In-house composite action. It installs the project's dependencies and runs the ESLint (and configuration) the project itself ships — no Action wrapped, no ESLint version imposed by the platform.

## Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `working-directory` | No | `.` | Directory to run ESLint in — one component's `path` from `detect-changes`' `components[]`. |

## Outputs

This action produces no outputs; it succeeds or fails, and writes its findings to the job summary.

## How it works

```mermaid
flowchart TD
    Anode_modules present? -- "no" --> B["Install without lifecycle scripts"]
    A -- "yes" --> C
    B --> CESLint in node_modules?
    C -- "no" --> D["Warn, summary says skipped, step succeeds"]
    C -- "yes" --> E["eslint . --format json"]
    E --> FExit code
    F -- "0" --> G["Step succeeds"]
    F -- "1: errors" --> H["Errors in the summary, fail the step"]
    F -- "2: ESLint could not run" --> I["Error in the summary, fail the step"]
```

The install uses `npm ci --ignore-scripts` (or `pnpm install --frozen-lockfile --ignore-scripts`, pnpm pinned and run through `npx`). If `node_modules` already exists in the directory (another step of the same job installed it), nothing is installed again.

## Usage

```yaml
- uses: ./actions/lint-node
  with:
    working-directory: frontend
```

## Known issues / Limitations

- **Skipped without ESLint.** A project that does not have ESLint as a dependency is not linted; the step succeeds with a warning and a "Skipped" note in the summary, never silently.
- **yarn is not supported yet.** With only a `yarn.lock` the dependencies are not installed (warning), so ESLint is usually not found and the step skips.
- **The ESLint config is code from the pull request** and is executed by ESLint. Lifecycle scripts are not run, but this is still why the job belongs on an ephemeral runner. See [security.md](../../docs/security.md#how-the-scans-run).
- **Verified locally** with a real ESLint project (errors, clean, no ESLint); not yet run on a GitHub runner.

## Related documentation

- [decisions.md](../../docs/decisions.md) — "Node actions install dependencies without lifecycle scripts; yarn is unsupported"
- [workflows.md](../../docs/workflows.md) — actions overview and capability → CI mapping
