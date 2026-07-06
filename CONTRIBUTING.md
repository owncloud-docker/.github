# Contributing

Thank you for your interest in contributing to the ownCloud Docker projects!

Please read the full contributing guidelines at:
**<https://owncloud.com/contribute/>**

## About this repository

This is the **organisation `.github` repository** for the
[`owncloud-docker`](https://github.com/owncloud-docker) organisation. The
community health files it contains (`CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`,
`SECURITY.md`, `SUPPORT.md`) act as **organisation-wide defaults**: GitHub
applies them to every repository in the org that does not provide its own copy.
The repository itself holds no source code — only these defaults and shared
documentation such as [`docs/IMAGE-LIFECYCLE.md`](docs/IMAGE-LIFECYCLE.md).

To contribute to a specific image, open a PR in that image's repository
(for example [`owncloud-docker/server`](https://github.com/owncloud-docker/server)
or [`owncloud-docker/ocis`](https://github.com/owncloud-docker/ocis)).

## Pull requests

- **Rebase Early, Rebase Often!** We use a rebase workflow. Rebase on the target
  branch before submitting a PR; do not create merge commits.
- **Signed commits**: All commits **must** be PGP/GPG signed. See
  [GitHub's signing guide](https://docs.github.com/en/authentication/managing-commit-signature-verification).
- **DCO Sign-off**: Every commit must carry a `Signed-off-by` line:
  ```
  git commit -S -s -m "your commit message"
  ```
- **Conventional Commits**: PR titles must follow the
  [Conventional Commits](https://www.conventionalcommits.org/) format.
- **GitHub Actions Policy**: Workflows may only use actions that are (a) owned by
  `owncloud`, (b) created by GitHub (`actions/*`), or (c) verified in the GitHub
  Marketplace. Pin all actions to their full commit SHA.
