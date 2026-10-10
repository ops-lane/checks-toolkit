# Security policy

This repository provides reusable GitHub Actions workflows and composite
actions that other repositories run in their own pipelines. A flaw here can
therefore affect everyone who calls them, which is why security reports are
taken seriously.

## Reporting a vulnerability

Please **do not open a public issue** for a security problem.

Use GitHub's private reporting instead: go to the **Security** tab of this
repository and choose **Report a vulnerability**. Only the maintainer can see
the report.

Include what you found, how to reproduce it (a minimal workflow or pull
request is ideal), and what you think the impact is.

## What to expect

This is a personal project with a single maintainer, not a company with a
security team. Reports are handled on a best-effort basis: expect an
acknowledgement and a first assessment, but there is no guaranteed response
time and there is no bug bounty.

Fixes are released as a new version. Please allow time for a fix before
disclosing the issue publicly; the maintainer will credit you in the fix
unless you prefer not to be named.

## Supported versions

The project is pre-1.0 and anything may change between versions. Only the
latest release and the default branch receive fixes.

## What counts as a vulnerability here

In scope — problems in this repository's own workflows and actions, for
example:

- A way for a pull request (including one from a fork) to run commands,
  read secrets or the repository token, or gain write access through these
  workflows or actions (script injection, unsafe use of `pull_request_target`,
  credentials left on disk).
- A way for a pull request to hide or switch off a security check, for
  example by changing the scanner's own configuration.
- Supply-chain weaknesses: an unpinned third-party action, image or package
  that these workflows run.
- Secrets or sensitive data exposed in logs, summaries or pull request
  comments.

Out of scope:

- Vulnerabilities in the tools these workflows run (Trivy, Gitleaks,
  Semgrep, CodeQL, and so on). Report those to the tool's own maintainers.
- Findings that a scanner reports about *your own* code.
- Weaknesses that need an attacker who already has write access to the
  repository.
