# `typecheck-node`

Type-checks TypeScript with the project's own `tsc --noEmit`. A blocking check: the step fails on any type error (or an unusable `tsconfig.json`), with the output in the job summary.

**Origin:** In-house composite action. It installs the project's dependencies and runs the TypeScript version the project itself ships.

## Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `working-directory` | No | `.` | Directory to run tsc in — one component's `path` from `detect-changes`' `components[]`. |

## Outputs

This action produces no outputs; it succeeds or fails, and writes its findings to the job summary.

## How it works

```mermaid
flowchart TD
    Anode_modules present? -- "no" --> B["Install without lifecycle scripts"]
    A -- "yes" --> C
    B --> Ctsconfig.json and tsc present?
    C -- "no" --> D["Notice, summary says skipped, step succeeds"]
    C -- "yes" --> E["tsc --noEmit"]
    E --> FExit code
    F -- "0" --> G["Step succeeds"]
    F -- "non-zero" --> H["Errors in the summary, fail the step"]
```

The install uses `npm ci --ignore-scripts` (or `pnpm install --frozen-lockfile --ignore-scripts`, pnpm pinned and run through `npx`). If `node_modules` already exists in the directory (another step of the same job installed it), nothing is installed again.

## Usage

```yaml
- uses: ./actions/typecheck-node
  with:
    working-directory: frontend
```

## Known issues / Limitations

- **Skipped without TypeScript.** A project without `tsconfig.json` or without TypeScript as a dependency is not checked; the step succeeds with a notice and a "Skipped" note in the summary.
- **yarn is not supported yet** (see `lint-node`).
- **Verified locally** with a real TypeScript project (error, no TypeScript); not yet run on a GitHub runner.

## Related documentation

- [decisions.md](../../docs/decisions.md) — "Node actions install dependencies without lifecycle scripts; yarn is unsupported"
- [workflows.md](../../docs/workflows.md) — actions overview and capability → CI mapping
