# agents.md — .github (organisation meta-repository)

## Repository Overview

This is the **organisation `.github` repository** for the
[`owncloud-docker`](https://github.com/owncloud-docker) GitHub organisation. It
holds no source code and builds no image. It serves two purposes:

1. **Organisation-wide community health defaults** — the health files at the repo
   root (`CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`, `SECURITY.md`, `SUPPORT.md`)
   are applied by GitHub to **every** repository in the org that does not supply
   its own copy.
2. **Shared documentation** — cross-repo reference docs under `docs/`.

- **Classification:** Organisation meta-repository (no build)
- **Activity Status:** Active
- **License:** Apache-2.0

## Architecture & Key Paths

- `README.md` — organisation landing page
- `LICENSE` — Apache-2.0
- `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`, `SECURITY.md`, `SUPPORT.md` —
  org-wide default community health files
- `docs/IMAGE-LIFECYCLE.md` — how the ownCloud Docker images are built, tagged,
  scanned, kept up to date, and published (the authoritative lifecycle document)

There is **no build system, CI workflow, or `CHANGELOG.md`** in this repository.

## How the org-wide defaults work

GitHub's community-health-file fallback resolves in this order for any repo:

1. The repository's own file, if present.
2. Otherwise, the file in this `.github` repo (root, `.github/`, or `docs/`).

So a repo without its own `SECURITY.md` inherits the one here. Repos that need
repo-specific wording (for example the image repos, which each ship their own
tailored `CONTRIBUTING.md`) override the default by committing their own copy.

## Development Conventions

- Keep the health files generic — they are the fallback for the whole org.
- Conventional-Commit PR titles.
- No build or lint toolchain; changes are documentation only.

## OSPO Policy Constraints

### GitHub Actions
- If workflows are ever added, **only** use actions owned by `owncloud`, created
  by GitHub (`actions/*`), verified on the GitHub Marketplace, or verified by the
  ownCloud Maintainers, pinned to a full commit SHA.

### Git Workflow
- **Rebase policy**: Always rebase; never create merge commits.
- **Signed commits**: All commits **must** be PGP/GPG signed (`git commit -S`).
- **DCO sign-off**: Every commit needs a `Signed-off-by` line (`git commit -s`).
- **Conventional Commits**: PR titles must follow
  [Conventional Commits](https://www.conventionalcommits.org/).

## Context for AI Agents

- This is a meta-repo: editing a health file here changes the default for every
  `owncloud-docker` repository that lacks its own copy.
- Do not add a build system or `CHANGELOG.md` — neither belongs here.
- `docs/IMAGE-LIFECYCLE.md` is the canonical description of the image build,
  scan, and publish pipeline; keep links to it stable.
