<!-- markdownlint-disable MD041 -->
## What and why

<!-- What does this change, and why? Link the issue if there is one. -->

## Checklist

- [ ] I ran `actionlint` on any workflow I changed
- [ ] No `${{ }}` expression inside a `run:` or `script:` (values go through `env:`)
- [ ] Every new `uses:` is pinned to a full commit SHA with the version in a comment
- [ ] Every new `actions/checkout` has `persist-credentials: false`
- [ ] The README of every action I changed is updated, including what was actually verified
- [ ] A decision worth remembering is recorded in `docs/decisions.md`
- [ ] Documentation and changelog are updated if behavior changed
