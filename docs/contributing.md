# Checks Toolkit — Conventions

## Naming

- Actions: `actions/<kebab-case-name>/action.yml`, following a
  `<type>-<target>` pattern where possible — `type` is the kind of check
  (`lint`, `typecheck`, `audit`, `scan`, `build`) and `target` is
  what it checks: a language/technology (`python`, `node`, `docker`,
  `terraform`, `markdown`) or a repo-wide concern (`secrets`, `code`). The
  name always describes the **concern**, never the specific tool inside —
  that stays swappable without a rename (see "Actions policy" below and
  the "Actions naming convention" row in
  [decisions.md](./decisions.md)). See
  [workflows.md](./workflows.md#actions-overview) for the current list.
- Workflows: `.github/workflows/<name>.yml` — fixed names: `ci.yml`,
  `security.yml`, `docker.yml`, `deploy.yml`
- Runner labels: `self-hosted`, `linux`, `ci` (later: `deploy`)
- Documentation titles: consistently use **"Checks Toolkit"**
  as the product name, followed by an em dash and the document
  name (see the other files in `docs/` for examples)

## Actions policy

- **No random third-party Actions.** Only use Actions published by official/
  verified sources (GitHub itself, Docker, major cloud providers, official
  tool vendors) or Actions built in-house in this repo. Never pin an Action
  from an individual maintainer, however popular — it runs with the calling
  workflow's permissions and secrets, and a compromised maintainer account
  is a real supply-chain risk (see [decisions.md](./decisions.md), row
  "Third-party Actions policy"). If a capability isn't covered by an
  official Action, build a composite action for it here instead of reaching
  for a community one.
- **Every composite action must be standalone-callable.** Define explicit
  inputs and outputs and don't assume it's only ever invoked from `ci.yml`
  or `security.yml`. Someone (including future-you) should be able to
  `uses:` a single action like `detect-changes` on its own, in an unrelated
  workflow, and get a self-contained result — without needing the rest of
  this platform's pipeline around it.

## Security rules for workflows and actions

Anything under `.github/workflows/` and `actions/` runs in other
repositories' pipelines, so these rules are not optional. The reasoning is
in [security.md](./security.md) and [decisions.md](./decisions.md).

- **Never write `${{ ... }}` inside a `run:` script or a `github-script`
  `script:`.** Pass the value through `env:` and read it as a variable. This
  applies to *every* expression, not only the obviously hostile ones: file
  names, branch names, inputs and even outputs of earlier steps can carry
  attacker-controlled text. `actionlint` flags the obvious cases; the rest is
  on the author.
- **Pin every `uses:` to a full commit SHA** with the version in a comment.
  Run `gh api repos/<owner>/<repo>/git/ref/tags/<tag>` to find it (an
  annotated tag has to be dereferenced once more).
- **Every `actions/checkout` sets `persist-credentials: false`.**
- **Default `permissions: contents: read`**; add more only on the single
  job that needs it.
- **Run tools that parse untrusted input in a throwaway container**
  (read-only mount, `--cap-drop ALL`, own user, no secrets) like the existing
  scan actions.
- Run `actionlint` before pushing (`docker run --rm -v "$PWD":/repo -w /repo
  rhysd/actionlint:latest`); it includes shellcheck.

## Pinned versions

Dependabot keeps `uses:` pins current. These are pinned too, but Dependabot
cannot see them, so they are bumped by hand — check them now and then, and
whenever a security advisory for one of the tools appears.

| What | Where | Pinned as |
|---|---|---|
| Trivy | `actions/scan-docker`, `actions/scan-terraform` | container image, version + digest (the two must match) |
| Gitleaks | `actions/scan-secrets` | container image, version + digest |
| Semgrep | `actions/scan-code` | container image, version + digest |
| lychee | `actions/lint-markdown` | container image, version + digest |
| Ruff | `actions/lint-python` | exact pip version |
| mypy | `actions/typecheck-python` | exact pip version |
| pip-audit, Poetry, poetry-plugin-export, pipenv | `actions/audit-python` | exact pip versions |
| codespell, markdownlint-cli2 | `actions/lint-markdown` | exact pip / npm version |
| pnpm | `actions/lint-node`, `typecheck-node`, `audit-node` | exact version, run through `npx` |
| Runner image | `runs-on` in every workflow under `.github/workflows/` | `ubuntu-26.04`, not `ubuntu-latest`; `.github/actionlint.yaml` declares the label for actionlint, which does not know it yet |
| git-cliff | `.github/workflows/release.yml` | container image, version + digest |
| Terraform CLI | `actions/lint-terraform` | `terraform-version` input |
| actionlint | `actions/lint-workflows` | `actionlint-version` and `actionlint-sha256` inputs (change together; the download is verified against the hash) |
| zizmor | `actions/scan-workflows` | `zizmor-version` input (exact pip version) |

To find the digest of a new container image, pull it and read
`docker image inspect <image> --format '{{index .RepoDigests 0}}'`.

## Adding a new ecosystem to detection

1. Add detection signals to `actions/detect-changes` (see
   [workflows.md](./workflows.md) for the current signal overview)
2. Add the capability to the `project.*`/`changed.*` output schema
3. Add the capability → CI mapping to `docs/workflows.md`
4. Add the corresponding job to `ci.yml` (and `security.yml` where
   relevant), conditional on the new `changed.*` flag
5. Test against `ops-lane/checks-toolkit-test`: a PR that only touches
   this new capability must trigger the right jobs, and every other PR
   must keep skipping the new jobs

Important: this all happens centrally in `checks-toolkit`. Consumer
repos never need to change to support a new ecosystem (see the core
principle in [architecture.md](./architecture.md)).

## Versioning

- **Exact tags only**: `v0.1.0`, `v0.2.0`, ..., `v1.0.0`. There is no moving
  `v1` tag. A tag is never moved or deleted once released, so what a consumer
  pins is what it gets. Updating is a deliberate step on the consumer's side;
  Dependabot proposes it and shows the release notes. The reasoning is in
  [decisions.md](./decisions.md), row "Exact tags, no moving major tag".
- **Before 1.0.0** (`v0.x`) the interface can still change between minor
  versions, and every `v0.x` release is marked as a pre-release.
- Consumer repos pin to an exact tag (`@v0.1.0`, with `platform-ref` set to
  the same value), never to `@development`.
- A new version is run against `ops-lane/checks-toolkit-test` first, before
  a real project repo is upgraded.

## Commit messages

Commit subjects follow [Conventional Commits](https://www.conventionalcommits.org):
`type(scope): description`, with type one of `feat`, `fix`, `docs`, `style`,
`refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`. A `!` after the
type or scope marks a breaking change (`feat(api)!: ...`). A security fix is
`fix(security): ...`.

The release notes are generated from these subjects, so the `check-commits`
workflow fails a pull request that has a commit without one. Merge commits
are not checked. The type decides the section a commit lands in (`feat` under
Added, `fix` under Fixed, `fix(security)` under Security, and so on; see
`cliff.toml`).

## Releasing

A release is started by hand from the **Release** workflow
(`.github/workflows/release.yml`, "Run workflow" on the default branch):

1. Make sure everything to be released is merged and its checks are green.
2. Run the workflow with the **version** typed in (`0.2.0`, without the `v`;
   `1.0.0-rc.1` for a release candidate) and **dry-run left on**. Nothing is
   created; the job summary shows the release notes and what `git-cliff`
   would suggest as the version.
3. Read the notes. A wrong note means a wrong commit subject, so fix it before
   releasing (a commit already merged cannot be changed; reword it in the
   next one or write the notes by hand, see below).
4. Run it again with **dry-run switched off**. The workflow validates the
   version again, creates the tag on the exact commit it validated, and
   publishes the GitHub Release with those notes.

What the workflow checks before it publishes: the version is a valid semantic
version, higher than the latest stable one, and its tag does not exist yet;
the run is on the default branch; and every check run on that commit has
passed. It refuses otherwise.

**Hand-written notes.** If `.github/release-notes/v<version>.md` exists, it is
used instead of the generated notes. The first release needs this: the history
is squashed before the repository goes public, so there are no commits to
generate notes from.

**One-time setting:** switch on *immutable releases* in the repository
settings before the first release, so a published release and its tag can
never be changed afterwards. It does not apply retroactively.

## Per-action README

Once an action has real tooling implemented (not just a placeholder skeleton) and is tested, it gets an `actions/<name>/README.md` — copy [actions/TEMPLATE.md](../actions/TEMPLATE.md) and fill it in. Required sections: description, inputs, outputs, a Mermaid diagram of how it works, known issues/limitations, and whether it's in-house or wraps an official third-party Action. Keep it in the action's own folder so it travels with the action if it's ever extracted (see "Actions repo layout" in [decisions.md](./decisions.md)).

## Language

The language of this repository is **English**: documentation, code comments,
commit messages, issues and pull requests. The documentation spell check
(`codespell`) assumes English.

## Keeping documentation up to date

For every architectural decision: record it in
[decisions.md](./decisions.md) (with the reason), then implement.
`decisions.md` is the source of truth for "what has already been decided and
why"; [roadmap.md](./roadmap.md) is the source of truth for what is built,
how well it is verified and what is next. The other files in `docs/` are the
reader-friendly, topic-organized system documentation, and each action
documents itself in its own README.

## Running workflows locally

[`act`](https://github.com/nektos/act) can run a workflow locally before
pushing. For example, to run the self-test workflow against a sample
event payload:

```bash
act push -e event.json --workflows ".github/workflows/self-test.yml"
```

`event.json` is a hand-written GitHub event payload (not checked in).
