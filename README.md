# owncloud-docker

[![License: Apache-2.0](https://img.shields.io/github/license/owncloud-docker/.github)](https://github.com/owncloud-docker/.github/blob/master/LICENSE)
[![ownCloud OSPO](https://img.shields.io/badge/OSPO-ownCloud-blue)](https://kiteworks.com/opensource)

Organisation-wide files and documentation for the
[`owncloud-docker`](https://github.com/owncloud-docker) organisation.

This repository holds no source code. It provides the organisation-wide default
[community health files](#community-health-defaults) and shared documentation for
the ownCloud Docker image repositories.

- [Docker Image Lifecycle](docs/IMAGE-LIFECYCLE.md) — how the ownCloud Docker
  images are built, tagged, scanned, kept up to date, and published.

## Community health defaults

The files in this repository act as **organisation-wide defaults**: GitHub
applies them to every repository in the `owncloud-docker` organisation that does
not supply its own copy.

- [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md)
- [CONTRIBUTING.md](CONTRIBUTING.md)
- [SECURITY.md](SECURITY.md)
- [SUPPORT.md](SUPPORT.md)

## Community & Support

- [ownCloud Website](https://owncloud.com)
- [Community Discussions](https://github.com/orgs/owncloud/discussions)
- [Matrix Chat](https://app.element.io/#/room/#owncloud:matrix.org)
- [Documentation](https://doc.owncloud.com)
- [Enterprise Support](https://owncloud.com/contact-us/)
- [OSPO Home](https://kiteworks.com/opensource)

See [SUPPORT.md](SUPPORT.md) for the full list of support channels.

## Contributing

We welcome contributions! Please read the [Contributing Guidelines](CONTRIBUTING.md)
and our [Code of Conduct](CODE_OF_CONDUCT.md) before getting started. To
contribute to a specific image, open a PR in that image's own repository.

- **Rebase Early, Rebase Often!** We use a rebase workflow — rebase on the target
  branch before submitting a PR.
- **Signed commits**: All commits **must** be PGP/GPG signed and carry a DCO
  `Signed-off-by` line (`git commit -S -s`).
- **Conventional Commits**: PR titles must follow the
  [Conventional Commits](https://www.conventionalcommits.org/) format.

## Security

**Do not open a public GitHub issue for security vulnerabilities.**

Report vulnerabilities at **<https://security.owncloud.com>** — see [SECURITY.md](SECURITY.md).

Bug bounty: [YesWeHack ownCloud Program](https://yeswehack.com/programs/owncloud-bug-bounty-program)

## About the ownCloud OSPO

The [Kiteworks Open Source Program Office](https://kiteworks.com/opensource), operating under
the [ownCloud](https://owncloud.com) brand, launched on May 5, 2026, to steward the open source
ecosystem around ownCloud's products. The OSPO ensures transparent governance, license compliance,
community health, and sustainable collaboration between the open source community and
[Kiteworks](https://www.kiteworks.com), which acquired ownCloud in 2023.

- **OSPO Home**: <https://kiteworks.com/opensource>
- **GitHub**: <https://github.com/owncloud>
- **ownCloud**: <https://owncloud.com>

For questions about the OSPO or licensing, contact ospo@kiteworks.com.

## License

This repository is licensed under the **Apache License 2.0** — the license the
OSPO is adopting across the ecosystem. See the [LICENSE](LICENSE) file for details.
