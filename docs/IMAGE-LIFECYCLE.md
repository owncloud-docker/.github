# ownCloud Docker Image Lifecycle

How the official ownCloud container images are **built, tagged, scanned, kept up to
date, and published**. This is the authoritative concept document for the
`owncloud-docker` organisation. It is written primarily for the security team: the
[Keeping images up to date](#5-keeping-images-up-to-date) and
[Security control summary](#7-security-control-summary) sections describe how
dependency and CVE fixes reach the published images, and on what cadence.

> This document describes the *concept and process*. The single source of truth for
> exact versions, digests and CVE exceptions is always the code in the repositories
> linked under [References](#8-references).

---

## 1. Overview & scope

The organisation publishes several Docker images to Docker Hub. Two are the "final"
images consumed by end users:

- **`owncloud/server`** — ownCloud Classic (PHP application, packaged from a release tarball).
- **`owncloud/ocis`** — ownCloud Infinite Scale (Go application, built from source).

Three **supporting base images** exist only to build `owncloud/server`, plus a
frontend image:

- **`owncloud/ubuntu`** — Ubuntu OS layer + shared tooling (gomplate, wait-for, retry).
- **`owncloud/php`** — Apache + PHP on top of `owncloud/ubuntu`.
- **`owncloud/base`** — ownCloud runtime, entrypoint and `occ` dispatcher on top of `owncloud/php`.
- **`owncloud/web`** — ownCloud Web frontend (built independently, nginx-based).

### Repository topology

Each image is maintained in its **own GitHub repository** under the
[`owncloud-docker`](https://github.com/owncloud-docker) organisation
(`server`, `ocis`, `base`, `php`, `ubuntu`, `web`). There is no monorepo.

The **`ubuntu` repository is the CI hub**: it hosts the *reusable* GitHub Actions
workflows that every other repository calls
(`docker-build.yml`, `docker-build-native.yml`, `docker-hub-desc.yml`,
`lint-editorconfig.yml`, `lint-pr-title.yml`). A change to the build, scan, tag or
publish process is made once in `ubuntu` and takes effect for all images.

### Image inheritance chain

```
docker.io/ubuntu:<ver>@sha256:…      (upstream, digest-pinned)
        │
        ▼
owncloud/ubuntu    ──►  owncloud/php  ──►  owncloud/base  ──►  owncloud/server
                                                                (release tarball)

owncloud/ocis      (independent, multi-stage build from source)
owncloud/web       (independent, multi-stage, nginx)
```

A rebuild of a lower layer (e.g. `owncloud/ubuntu` after an Ubuntu base bump) flows
upward: `php` → `base` → `server` are rebuilt against the new digest via Renovate PRs
and the weekly schedule (see §5).

---

## 2. Build

All images are built by **GitHub Actions**, multi-architecture for
**`linux/amd64` and `linux/arm64`**. Each repository has a `main.yml` workflow that
calls the shared reusable workflow in the `ubuntu` repo. Two build strategies exist:

### 2a. Cross-compile build — `docker-build.yml`

Used by `ubuntu`, `php`, `base`, `server`, `web`. A single job:

1. Starts an ephemeral local registry (`registry:3` service on port 5000).
2. Builds both architectures in one `docker/build-push-action` run (BuildKit / buildx)
   and pushes the result to the local registry as `127.0.0.1:5000/image:temp`.
3. Runs the Trivy scan (§4) and the smoke test (§6) against that temporary image.
4. On `master`, logs in to Docker Hub and re-runs the build pushing the final tags.

### 2b. Native per-arch build + manifest merge — `docker-build-native.yml`

Used by `ocis` (both the release `main.yml` and the `rolling.yml` workflow), because
building oCIS from source cross-platform is expensive. A matrix runs the build
**natively** on each architecture (`amd64` on `ubuntu-latest`, `arm64` on
`ubuntu-24.04-arm`):

1. Each arch builds, loads the image locally, scans (Trivy) and smoke-tests it.
2. On `master`, each arch pushes **by digest** (no tag) to Docker Hub and uploads its
   digest as an artifact.
3. A final `merge` job assembles a **multi-arch manifest** from the per-arch digests
   with `docker buildx imagetools create` and applies all tags.

### 2c. oCIS three-stage from-source build

The oCIS image (`ocis/v8/Dockerfile.multiarch`) is built entirely from source:

- **`node-builder`** (Alpine) — clones the oCIS git repo at the requested ref, builds
  the IDP frontend (`pnpm build`) and pulls web assets (both are `//go:embed`-ed).
- **`go-builder`** (golang-alpine) — compiles the binary with CGO + libvips via the
  upstream `release-linux-docker-${TARGETARCH}` Makefile target.
- **runtime** (Alpine) — copies only the binary; runs `apk upgrade` so OS packages are
  at the latest Alpine patch level at build time.

### Build arguments (how the application version is selected)

| Image | Arg(s) | Meaning |
|-------|--------|---------|
| `server` | `TARBALL_URL` | URL of the `owncloud-complete-*.tar.bz2` release tarball, injected from the workflow matrix. No version is pinned inside the Dockerfile. |
| `ocis` | `VERSION`, `GIT_REF`, `GIT_SHA`, `REVISION` | Git tag (`v${VERSION}`) or branch (`GIT_REF=master`) to clone; `GIT_SHA` pins a branch build to an exact commit (used by rolling builds); `REVISION` is embedded in OCI labels. |
| `web` | `VERSION` | Release tag whose `web.tar.gz` is downloaded and **SHA256-verified** during build. |

---

## 3. Tagging

Tags combine **human-friendly floating tags** with **immutable date/commit tags** so
consumers can either track a line of updates or pin an exact build.

| Image | Example tags | Notes |
|-------|--------------|-------|
| `owncloud/server` | `10.16.3`, `10.16`, `10`, `latest`, `10.16.3-<YYYYMMDD>` | Floating major/minor/`latest` plus an immutable date tag (`build-date-tag: true`). |
| `owncloud/server` (RC) | `11.0.0-rc1`, `11.0.0-rc1-<YYYYMMDD>` | Version + immutable date tag, but **no floating `latest`/major/minor tags**. |
| `owncloud/ocis` | `8.0.5`, `8.0`, `8`, `8.0.5-<YYYYMMDD>` | Floating + immutable date tag. |
| `owncloud/ocis` (RC) | `8.1.0-rc.2`, `8.1.0-rc.2-<YYYYMMDD>` | Version + immutable date tag, but **no floating `latest`/major/minor tags**. |
| `owncloud/ocis-rolling` | `latest`, `<YYYYMMDD>`, `sha-<short>` | Daily build of oCIS `master` (unstable, testing only). |
| `owncloud/web` | `12.3.3`, `12.3`, `12`, `latest` | Published on git tags only. |
| `owncloud/{ubuntu,php,base}` | `22.04`, `24.04`, `22.04-<YYYYMMDD>` | Ubuntu-release-based tags + immutable date tag. |

Immutable date/`sha-` tags exist specifically so a deployment can pin the exact bytes
it was validated against while `latest`/minor tags keep receiving security rebuilds.

---

## 4. Scanning

Every build — on **pull request, push, and every scheduled rebuild** — is scanned with
**Trivy** (`aquasecurity/trivy-action`) *before* it can be published:

```yaml
severity:       HIGH,CRITICAL
ignore-unfixed: true
skip-files:     /usr/bin/gomplate,/usr/bin/wait-for
exit-code:      1            # a finding fails the job → nothing is pushed
trivyignores:   <per-repo/per-version .trivyignore>
```

Key properties for the security team:

- **The scan gates publication.** `exit-code: 1` means an unresolved HIGH/CRITICAL
  finding fails the workflow, so a vulnerable image is never pushed to Docker Hub.
- **`ignore-unfixed: true`** — only CVEs with an available upstream fix fail the build;
  vulnerabilities with no fix yet do not block releases (they are picked up
  automatically once a fix lands and the next scheduled rebuild runs).
- **PRs are scanned too**, so regressions are caught before merge, not just at publish.

### Accepted-CVE exception process (`.trivyignore`)

When a HIGH/CRITICAL finding is a false positive or cannot be fixed (e.g. the fix lives
in an upstream dependency ownCloud does not control yet), the CVE is added to a
`.trivyignore` file **with a justification comment**. Exceptions are scoped as tightly
as possible:

- Repository-wide: `<repo>/.trivyignore`
- Per application version: e.g. `server/v22.04/10.16.3/.trivyignore`, `ocis/v8/.trivyignore`

Each entry documents *why* it is accepted, for example:

```
# vulnerability is affecting windows only
CVE-2024-51736

# fix requires ownCloud to update bundled aws-sdk-php in files_primary_s3
GHSA-27qh-8cxx-2cr5
```

Reviewing and pruning these files (removing entries once the fix ships) is part of
regular maintenance — a stale ignore silently suppresses a now-fixable CVE.

---

## 5. Keeping images up to date

> **This is the section the security team asked for.** It explains how dependency and
> CVE fixes reach published images, largely **without manual intervention**, and why
> the window between an upstream fix and a rebuilt image is short and bounded.

There are three categories of dependency, each with its own update mechanism:

### 5a. Base images (the OS layer) — digest pinning + Renovate

Every `FROM` is pinned to an **immutable SHA256 digest**, not a floating tag:

```dockerfile
FROM docker.io/ubuntu:24.04@sha256:786a8b558f7be160c6c8c4a54f9a57274f3b4fb1491cf65…
FROM owncloud/base:24.04@sha256:084cd10e781442c7081a68066f0ec7c2b9a70756ba59ca0e…
```

Digest pinning makes every build **reproducible** and prevents a silently-changed
upstream tag from entering an image unreviewed. To stay current, **Renovate** opens
PRs that bump these digests:

- Every repository has a `.renovaterc.json` extending the shared preset
  **`github>owncloud-ops/renovate-presets:docker`** (which in turn extends
  `:base`), so update policy is centrally governed.
- The preset **auto-merges** digest/pin updates for a curated allowlist of images —
  including `owncloud/*`, `ubuntu`, `alpine`, `golang` and others — because these are
  usually security patches. There is no fixed time-of-day schedule or open-PR cap for
  Docker updates in the preset; digest-bump PRs are raised as upstream digests change
  and merge automatically once CI (build → scan → smoke-test) passes.
- Merging a digest-bump PR triggers a full build → scan → smoke-test → publish cycle,
  so the security fix in the new base layer flows straight to Docker Hub through the
  normal gated pipeline.
- Because the images form a chain, a base bump (e.g. `owncloud/ubuntu`) cascades:
  Renovate then bumps the pinned `owncloud/php` digest, then `owncloud/base`, then
  `owncloud/server`.

The `ubuntu` repo additionally disables **major/minor** Ubuntu bumps in its Renovate
config (only digest/patch updates are automatic) so an OS-release jump is always a
deliberate, human decision.

### 5b. OS packages inside the image — build-time upgrade

OS packages are refreshed to the latest patch level available at build time. The
mechanism differs per image, so where the upgrade happens matters:

- **`owncloud/ubuntu`** (the base of the Ubuntu chain) runs an explicit
  `apt-get update && apt-get upgrade`. The images built on top of it (`php`, `base`,
  `server`) do **not** run `apt-get upgrade` themselves — they inherit the upgraded,
  digest-pinned `owncloud/ubuntu` layer and their own `apt-get update` + install
  pulls the current versions of the packages they add. So Ubuntu-side OS patches enter
  the chain via an `owncloud/ubuntu` rebuild (a digest bump or the weekly schedule),
  which then cascades upward.
- **oCIS runtime** (final Alpine stage) runs `apk upgrade --no-cache`, so it picks up
  Alpine patch releases on every build directly.
- **`web`** does **not** run `apk upgrade`; its final stage is the digest-pinned
  `nginx:alpine` image, so its OS-package currency depends on that pinned digest being
  bumped by Renovate (§5a), not on a build-time upgrade.

Combined with the **scheduled rebuilds** below, this means OS security patches land as
`owncloud/ubuntu` / base-digest rebuilds flow through the chain and, for oCIS, on every
rebuild directly.

### 5c. Scheduled rebuilds — closing the CVE window automatically

The pinning above is only useful if images are actually **rebuilt** so fixes land.
Two schedules guarantee that:

| Schedule | Cron / cadence | Scope | Effect |
|----------|----------------|-------|--------|
| **Weekly rebuild** | `0 0 * * 0` (Sun 00:00 UTC) | all image repos (`main.yml`) | Rebuilds against current base digests, re-scans with the latest Trivy DB, and re-publishes. Refreshes packages per the per-image mechanism in §5b (`owncloud/ubuntu` `apt-get upgrade`; oCIS `apk upgrade`). Picks up OS/base CVE fixes without any manual bump. |
| **Renovate (digests)** | continuous; auto-merge on green CI | all repos w/ `.renovaterc.json` | Raises digest/pin bump PRs as upstream digests change; the allowlisted ones auto-merge once CI passes, rebuilding through the gated pipeline. No fixed schedule or open-PR cap in the preset. |
| **Dependabot (Actions)** | weekly, Sun 22:00 UTC, ≤5 open PRs | repos w/ `.github/dependabot.yml` | Bumps GitHub Actions SHA pins (see §5e). |
| **oCIS rolling** | `0 2 * * *` (daily 02:00 UTC) | `ocis/rolling.yml` | Rebuilds `owncloud/ocis-rolling` from oCIS `master` HEAD (pinned to the resolved commit SHA), so upstream fixes on `master` are testable next day. |

Net effect: a HIGH/CRITICAL CVE fixed in a base image is picked up as soon as its
digest-bump PR auto-merges, and **at most one weekly cycle** later even if no digest
changed; the Trivy gate ensures the rebuilt image is verified clean before it ships.

### 5d. Pinned tooling binaries — Renovate datasource hints

Third-party binaries baked into `owncloud/ubuntu` (gomplate, wait-for, retry) are
version-pinned via `ENV` and annotated so Renovate can update them:

```dockerfile
# renovate: datasource=github-releases depName=hairyhenderson/gomplate
ENV GOMPLATE_VERSION="v5.1.0"
```

### 5e. GitHub Actions (the CI supply chain) — SHA pinning + Dependabot

The build pipeline itself is a dependency. Every `uses:` is pinned to a **full commit
SHA** (never a moving tag):

```yaml
uses: aquasecurity/trivy-action@ed142fd0673e97e23eac54620cfb913e5ce36c25 # v0.36.0
```

**Dependabot** (weekly) opens PRs to bump these pins. Per the org OSPO policy, only
actions owned by `owncloud`, authored by GitHub (`actions/*`), or Marketplace-verified
may be used — third-party unverified actions are not permitted.

### 5f. Application version updates

The application (not OS) versions are updated deliberately, not automatically:

- **`server`** — bump the `TARBALL_URL`/`version` entry in `server/.github/workflows/main.yml`.
- **`ocis`** — bump the release matrix (git tag) in `ocis/.github/workflows/main.yml`;
  the rolling image already tracks `master` daily.
- **`web`** — publish on the corresponding git tag; the tarball is SHA256-verified at build.

### 5g. Extended-support note — PHP 7.4

The `v22.04` PHP image (for ownCloud 10.x) uses PHP 7.4, which is past upstream Ubuntu
support. It pulls PHP packages from the **Freexian** extended-support Debian mirror,
authenticated via a Docker **build secret** (`--mount=type=secret`) so mirror
credentials never persist in the image. This keeps PHP 7.4 receiving security patches.

---

## 6. Publishing

- **Registry:** Docker Hub, under the `owncloud/` namespace.
- **When images are pushed:**
  - `server`, `base`, `php`, `ubuntu`: on push to `master` (and the weekly schedule).
  - `ocis`: on any non-PR event (`master` + rolling schedule).
  - `ocis-rolling`: always, on the daily schedule.
  - `web`: on git tags only.
  - Pull requests **build, scan and smoke-test but never push**.
- **The smoke test gates the push.** Before publishing, the freshly built image is run
  and validated:
  - `server`: poll `http://localhost:8080/status.php` for HTTP 200 (≈60 s window) and
    assert the reported `.versionstring` equals the tag.
  - `ocis`: poll `https://localhost:9200/status.php` (with `OCIS_INSECURE=true`) and
    assert `.productversion`.
  - supporting images: a one-shot command check inside the container —
    `ubuntu` asserts `VERSION_ID` from `/etc/os-release`, `php` runs
    `php --version | grep -qF 'PHP <ver>'`, `base` runs `php -r "echo 'OK';" | grep -q OK`,
    and `web` runs `nginx -t`.
- **Authentication:** `DOCKERHUB_USERNAME` (org variable) + `DOCKERHUB_TOKEN` (secret).
- **Docker Hub description sync:** on publish, the reusable `docker-hub-desc.yml`
  workflow pushes each repo's `README.md` as the Docker Hub image description, so the
  published description is always the reviewed, in-repo README.

---

## 7. Security control summary

| Concern | Control | Mechanism | Cadence | Where enforced |
|---------|---------|-----------|---------|----------------|
| Base-OS CVEs | Digest-pinned bases, auto-bumped & auto-merged | Renovate (`owncloud-ops/renovate-presets:docker`) | Continuous; auto-merge on green CI | every image repo `.renovaterc.json` |
| OS-package CVEs | Refresh packages at build time (per-image) | `apt-get upgrade` in `owncloud/ubuntu` (cascades to php/base/server); `apk upgrade` in oCIS runtime; `web` relies on pinned `nginx:alpine` digest bumps | Every build / base-digest bump | `ubuntu`, `ocis` `Dockerfile.multiarch`; §5a for `web` |
| Unpatched images | Rebuild even with no code change | Scheduled workflow rebuilds | Weekly (all) + daily (ocis-rolling) | `main.yml` / `rolling.yml` |
| Shipping a vulnerable image | Block publish on HIGH/CRITICAL | Trivy scan, `exit-code: 1`, `ignore-unfixed` | Every build incl. PRs | shared `docker-build*.yml` |
| Accepted/unfixable CVEs | Documented, scoped exceptions | `.trivyignore` w/ justification | Reviewed each maintenance | per-repo/per-version files |
| Reproducibility / supply chain | Immutable pins | SHA256 base digests; full-SHA Action pins | Continuous | Dockerfiles + workflows |
| CI supply chain | Trusted, pinned Actions | Dependabot bumps; OSPO allowlist | Weekly | `.github/dependabot.yml`, OSPO policy |
| Broken release | Runtime verification before push | Smoke test (status endpoint + version assert) | Every build | shared `docker-build*.yml` |
| Tarball integrity (web) | Checksum verification | SHA256 verify at build | Every build | `web/Dockerfile.multiarch` |
| Credential leakage | Build secrets, not layers | `--mount=type=secret` (Freexian mirror) | Every build | `php/v22.04/Dockerfile.multiarch` |

---

## 8. References

**Image repositories**

- [`owncloud-docker/server`](https://github.com/owncloud-docker/server)
- [`owncloud-docker/ocis`](https://github.com/owncloud-docker/ocis)
- [`owncloud-docker/base`](https://github.com/owncloud-docker/base)
- [`owncloud-docker/php`](https://github.com/owncloud-docker/php)
- [`owncloud-docker/ubuntu`](https://github.com/owncloud-docker/ubuntu)
- [`owncloud-docker/web`](https://github.com/owncloud-docker/web)

**Shared CI (in the `ubuntu` repo)**

- `.github/workflows/docker-build.yml` — cross-compile build, scan, smoke test, publish
- `.github/workflows/docker-build-native.yml` — per-arch build + manifest merge (oCIS)
- `.github/workflows/docker-hub-desc.yml` — Docker Hub description sync

**Dependency automation**

- `.renovaterc.json` in each repo → preset [`owncloud-ops/renovate-presets`](https://github.com/owncloud-ops/renovate-presets)
- `.github/dependabot.yml` (GitHub Actions updates)

**Vulnerability exceptions**

- `.trivyignore` (repo root) and per-version files, e.g. `server/v22.04/<version>/.trivyignore`, `ocis/v8/.trivyignore`
