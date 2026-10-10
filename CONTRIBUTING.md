# Contributing

Thanks for taking an interest. This is a small project with a single
maintainer: replies may take a while, but issues and pull requests are
welcome.

The conventions are in [docs/contributing.md](docs/contributing.md). The
short version:

- **Open an issue first** for anything bigger than a small fix, so the
  direction can be agreed before you spend time on it.
- **Security problems are not public issues.** See [SECURITY.md](SECURITY.md).
- **English** for everything: code comments, docs, commits, issues.
- **These workflows run in other people's pipelines**, so changes to
  `.github/workflows/` and `actions/` follow stricter rules: no
  `${{ }}` inside scripts (pass values through `env:`), every `uses:` pinned
  to a commit SHA, `persist-credentials: false` on checkouts, least-privilege
  permissions. They are spelled out in
  [docs/contributing.md](docs/contributing.md#security-rules-for-workflows-and-actions).
- **Lint your workflow changes** with `actionlint` before pushing.
- **Record decisions** in [docs/decisions.md](docs/decisions.md) and update
  the README of any action you change.
