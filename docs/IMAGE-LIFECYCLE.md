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

Two **extension images** are deployed alongside oCIS rather than instead of it:

- **`owncloud/ocis-workflows`** — oCIS Workflows, an AI-powered file workflow automation
  extension (built from source). Unlike the images above it carries **two deployables**:
  the Go backend sidecar, which is the container's entrypoint, and the Vue web extension
  for ownCloud Web. Both come from a single upstream checkout so they always ship in
  lockstep. The web assets are baked in at `/web/apps/workflows` but are **not served by
  this image** — an oCIS deployment copies that path into its `WEB_ASSET_APPS_PATH`.
- **`owncloud/web-extensions`** — ownCloud Web Extensions, a collection of community and
  supplementary extensions for the ownCloud Web frontend (built from source). Unlike every
  other image described here it is **one Docker Hub repository holding many unrelated
  images**: one tag per extension per release. Each carries no binary at all — an nginx
  runtime serving that one extension's built assets on port 8080. See the upstream
  [`owncloud/web-extensions`](https://github.com/owncloud/web-extensions) repository for
  how an extension is wired into an oCIS deployment.

Three **supporting base images** exist only to build `owncloud/server`:

- **`owncloud/ubuntu`** — Ubuntu OS layer + shared tooling (gomplate, wait-for, retry).
- **`owncloud/php`** — Apache + PHP on top of `owncloud/ubuntu`.
- **`owncloud/base`** — ownCloud runtime, entrypoint and `occ` dispatcher on top of `owncloud/php`.

### Repository topology

Each image is maintained in its **own GitHub repository** under the
[`owncloud-docker`](https://github.com/owncloud-docker) organisation
(`server`, `ocis`, `ocis-workflows`, `web-extensions`, `base`, `php`, `ubuntu`). There is
no monorepo.

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

owncloud/ocis            (independent, multi-stage build from source)
owncloud/ocis-workflows  (independent, multi-stage build from source; deployed
                          alongside owncloud/ocis as an extension)
owncloud/web-extensions  (independent, multi-stage build from source; one image tag
                          per ownCloud Web extension release)
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

Used by `ubuntu`, `php`, `base`, `server`. A single job:

1. Starts an ephemeral local registry (`registry:3` service on port 5000).
2. Builds both architectures in one `docker/build-push-action` run (BuildKit / buildx)
   and pushes the result to the local registry as `127.0.0.1:5000/image:temp`.
3. Runs the Trivy scan (§4) and the smoke test (§6) against that temporary image.
4. On `master`, logs in to Docker Hub and re-runs the build pushing the final tags.

### 2b. Native per-arch build + manifest merge — `docker-build-native.yml`

Used by `ocis` (both the release `main.yml` and the `rolling.yml` workflow),
`ocis-workflows` and `web-extensions`, because building these applications from source
cross-platform is expensive. A matrix runs the build **natively** on each architecture
(`amd64` on `ubuntu-latest`, `arm64` on `ubuntu-24.04-arm`):

1. Each arch builds, loads the image locally, scans (Trivy) and smoke-tests it.
2. When the caller passes `push: true` — every caller derives that from
   `github.event_name != 'pull_request'`, not from a branch comparison — each arch pushes
   **by digest** (no tag) to Docker Hub and uploads its digest as an artifact. Each
   caller's own `on: push` is still branch-filtered, so the non-default-ref publish paths
   are tag pushes (`ocis`, `ocis-workflows`) and `workflow_dispatch`.
3. A final `merge` job — also gated on `push`, so it does not run on PRs — assembles a
   **multi-arch manifest** from the per-arch digests with `docker buildx imagetools
   create` and applies all tags. A break in the manifest merge or the digest artifacts is
   therefore invisible on PRs.

### 2c. oCIS three-stage from-source build

The oCIS image (`ocis/v8/Dockerfile.multiarch`) is built entirely from source:

- **`node-builder`** (Alpine) — clones the oCIS git repo at the requested ref, builds
  the IDP frontend (`pnpm build`) and pulls web assets (both are `//go:embed`-ed).
- **`go-builder`** (golang-alpine) — compiles the binary with CGO + libvips via the
  upstream `release-linux-docker-${TARGETARCH}` Makefile target.
- **runtime** (Alpine) — copies only the binary; runs `apk upgrade` so OS packages are
  at the latest Alpine patch level at build time.

### 2d. oCIS Workflows three-stage from-source build

`ocis-workflows/Dockerfile.multiarch` follows the same shape as §2c, but the frontend is
shipped as static assets rather than embedded in the binary:

- **`frontend-builder`** (node-alpine) — clones the upstream repo at `GIT_REF` (pinned to
  `GIT_SHA` when set) and builds the Vue web extension with `pnpm build`.
- **`go-builder`** (golang-alpine) — compiles the backend with `CGO_ENABLED=0` for
  `TARGETARCH`. No CGO, so no libvips-style native dependencies.
- **runtime** (Alpine) — runs `apk upgrade`, adds a non-root user (uid 1000), and copies
  in **both** the binary (`/usr/local/bin/app`, the entrypoint) and the built frontend
  assets (`/web/apps/workflows`).

Because the Go stage compiles the shipped binary, the Go **standard library** linked into
it is whatever the pinned `golang:*-alpine` digest provides. A stale digest therefore
surfaces as `stdlib` findings against `/usr/local/bin/app` in the Trivy gate (§4); the fix
is bumping the digest, not adding a `.trivyignore` entry. This applies equally to `ocis`.

### 2e. Web Extensions two-stage from-source build

`web-extensions/Dockerfile.multiarch` builds **one** extension per image, selected by
build arg, and ships static assets only — there is no compiled binary and no Go stage:

- **`builder`** (node-alpine) — clones the upstream monorepo at `GIT_REF` (a release tag
  such as `cast-v0.4.1`) and builds that single package with
  `pnpm --filter ./packages/web-app-${PACKAGE} build`.
- **runtime** (`nginxinc/nginx-unprivileged:alpine`) — runs `apk upgrade`, empties the
  default document root, and copies the package's `dist/` to
  `/usr/share/nginx/html/${PACKAGE}`, served on port 8080 as the base image's own
  non-root `nginx` user (uid 101).

The upstream repo's `docker/Dockerfile` base (`owncloudops/nginx`) is deliberately *not*
reused: it has never published an arm64 variant, which would break the arm64 leg of
`docker-build-native.yml` (§2b).

**Which releases get built is not checked into this repo.** GitHub Actions cannot trigger
off a tag pushed in a *different* repository, so a `prepare` job polls instead:
`scripts/resolve-pending-releases.sh` lists upstream tags matching `<package>-v<version>`,
drops anything listed in `scripts/ignored-releases.txt`, diffs the rest against the tags
already on Docker Hub, and emits what remains as the `build` job's matrix — one leg per
pending release, `fail-fast: false`, and an empty matrix skips `build` entirely. This is
the only image in the organisation whose **set of matrix legs** is derived from live
upstream state rather than from a version list in the repository: `ocis-workflows` and
`ocis-rolling` also resolve a ref from upstream, but they always build exactly one image.

One consequence for PRs: because the matrix comes from upstream state and not from the
diff, a PR that changes `Dockerfile.multiarch` builds nothing at all unless a release
happens to be pending, so these images have no per-PR build gate (§6).

The Trivy gate (§4) sees the nginx runtime and its Alpine packages, but **not** the
extension's JavaScript dependencies: Vite bundles them into `dist/`, and no
`package.json` or lockfile is copied into the image, so there is nothing for Trivy's
JS analyser to resolve. This is the JS equivalent of the Go `stdlib` nuance in §2d, minus
the detection — a CVE in a bundled npm dependency is invisible to the scan and is fixed
upstream in the extension's own dependency tree.

### Build arguments (how the application version is selected)

| Image | Arg(s) | Meaning |
|-------|--------|---------|
| `server` | `TARBALL_URL` | URL of the `owncloud-complete-*.tar.bz2` release tarball, injected from the workflow matrix. No version is pinned inside the Dockerfile. |
| `ocis` | `VERSION`, `GIT_REF`, `GIT_SHA`, `REVISION` | Git tag (`v${VERSION}`) or branch (`GIT_REF=master`) to clone; `GIT_SHA` pins a branch build to an exact commit (used by rolling builds); `REVISION` is embedded in OCI labels. |
| `ocis-workflows` | `GIT_REF`, `GIT_SHA`, `VERSION`, `REVISION` | Same scheme as `ocis`, but only `GIT_REF=main` is used today — upstream has no release tags yet, so a `prepare` job resolves `main` HEAD and passes it as `GIT_SHA`/`REVISION`. That reference also busts the clone layer's cache, so the rolling build never serves a stale checkout. |
| `web-extensions` | `GIT_REF`, `PACKAGE`, `VERSION`, `REVISION` | `GIT_REF` is the upstream release tag to clone (`<package>-v<version>`); `PACKAGE` selects which `packages/web-app-<PACKAGE>` to build, so one Dockerfile produces every extension image; `VERSION`/`REVISION` are embedded in OCI labels. All four come from the polled matrix (§2e), never from a checked-in list. The `web-app-` prefix is assumed, not checked: a release tagged for a non-`web-app-` package (upstream also has `packages/ai-llm-proxy`) would be matched by the resolver and then fail the build, and `ignored-releases.txt` is the only way to exclude it. |

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
| `owncloud/ocis-workflows` | `latest`, `<YYYYMMDD>`, `sha-<short>` | **Rolling only, no version tags.** Upstream cuts no semver releases yet, so there is no version matrix; `latest` tracks upstream `main`. Pin the `sha-`/date tag in anything you care about. Once upstream starts tagging releases, this gains a version matrix following `owncloud/ocis`. |
| `owncloud/web-extensions` | `vim-nav-0.1.0`, `vim-nav-latest` | **One repository, many extensions.** Every tag is prefixed with the extension name: an immutable `<extension>-<version>` plus a floating `<extension>-latest`. No date tag — a given release is built once, so the version tag is already immutable. This pipeline publishes **no** global `latest` (no extension is "the" image). **Most tags in this repository predate this pipeline** (census as of 2026-09-21): of 98 tags only the 12 pushed on 2026-09-18 came from it (6 extensions × version + `-latest`). The other 86 — including 18 of the 24 `<extension>-latest` tags, e.g. `cast-latest` (2026-06-15) — were pushed by the upstream repo's own, now-superseded automation, and one global `latest` dates from 2024. Two consequences: those legacy tags are **`amd64`-only** (no arm64 manifest, unlike everything this pipeline builds), and since the resolver treats "tag exists on Docker Hub" as "already published", **their immutable `<extension>-<version>` tags will never be rebuilt or Trivy-scanned here** (§2e, §5c). A legacy `<extension>-latest` is fixed by that extension's next release, which re-points it at a freshly scanned multi-arch image; a legacy version tag never is. `<extension>-latest` is applied per matrix leg with no ordering, so a release for an older but still-maintained line, packaged after a newer one, takes the floating tag — pin `<extension>-<version>` where that matters. |
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
  The exception is `web-extensions`, where a PR only builds what the release poll finds
  pending, so in the steady state a `Dockerfile.multiarch` change is neither built nor
  scanned before it merges (§2e).

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

- `server`, `ocis`, `base`, `php` and `ubuntu` have a `.renovaterc.json` extending the
  shared preset **`github>owncloud-ops/renovate-presets:docker`** (which in turn extends
  `:base`), so update policy is centrally governed for them. The two extension-image
  repos, `ocis-workflows` and `web-extensions`, have **no** Renovate config: their base
  digests are bumped by weekly Dependabot PRs instead (§5c), which do **not** auto-merge
  and so wait for a human.
- The preset **auto-merges** digest/pin updates for a curated allowlist of images —
  including `owncloud/*`, `ubuntu`, `alpine`, `golang` and others — because these are
  usually security patches. There is no fixed time-of-day schedule or open-PR cap for
  Docker updates in the preset; digest-bump PRs are raised as upstream digests change
  and merge automatically once CI (build → scan → smoke-test) passes.
- Merging a digest-bump PR triggers a full build → scan → smoke-test → publish cycle,
  so the security fix in the new base layer flows straight to Docker Hub through the
  normal gated pipeline. **Not so for `web-extensions`:** the push to `main` runs its
  workflow, but the release poll finds nothing pending and the build is skipped, so a
  merged base-image bump publishes nothing. The bumped digest only reaches Docker Hub
  with that extension's next release (§5c).
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
- **oCIS, oCIS Workflows and Web Extensions runtimes** (final Alpine stage — Alpine nginx
  for Web Extensions) run `apk upgrade --no-cache`, so they pick up Alpine patch releases
  on every build directly. For `web-extensions` "every build" is narrower than it sounds:
  see the caveat in §5c.

Combined with the **scheduled rebuilds** below, this means OS security patches land as
`owncloud/ubuntu` / base-digest rebuilds flow through the chain and, for the Alpine-based
images, on every rebuild directly.

### 5c. Scheduled rebuilds — closing the CVE window automatically

The pinning above is only useful if images are actually **rebuilt** so fixes land.
Two schedules guarantee that:

| Schedule | Cron / cadence | Scope | Effect |
|----------|----------------|-------|--------|
| **Weekly rebuild** | `0 0 * * 0` (Sun 00:00 UTC) | all image repos except `web-extensions` (`main.yml`) | Rebuilds against current base digests, re-scans with the latest Trivy DB, and re-publishes. Refreshes packages per the per-image mechanism in §5b (`owncloud/ubuntu` `apt-get upgrade`; oCIS and oCIS Workflows `apk upgrade`). Picks up OS/base CVE fixes without any manual bump. For `ocis-workflows` this is also how upstream application changes land, since it has no release matrix. |
| **Renovate (digests)** | continuous; auto-merge on green CI | all repos w/ `.renovaterc.json` | Raises digest/pin bump PRs as upstream digests change; the allowlisted ones auto-merge once CI passes, rebuilding through the gated pipeline. No fixed schedule or open-PR cap in the preset. |
| **Dependabot (Actions)** | weekly, Sun 22:00 UTC, ≤5 open PRs | repos w/ `.github/dependabot.yml` | Bumps GitHub Actions SHA pins (see §5e). |
| **Dependabot (Docker)** | weekly, Sun 22:00 UTC, ≤5 open PRs | `ocis-workflows`, `web-extensions` (the two repos with a `docker` ecosystem and no Renovate config) | Bumps the base-image digests in `Dockerfile.multiarch` — the job Renovate does for the other repos (§5a), but human-merged, so a base CVE fix waits for a reviewer. |
| **oCIS rolling** | `0 2 * * *` (daily 02:00 UTC) | `ocis/rolling.yml` | Rebuilds `owncloud/ocis-rolling` from oCIS `master` HEAD (pinned to the resolved commit SHA), so upstream fixes on `master` are testable next day. |
| **Web Extensions release poll** | `0 */6 * * *` (every 6 h) | `web-extensions/main.yml` | A *release* trigger, not a rebuild schedule: each run re-polls upstream tags against Docker Hub (§2e), so a new extension release is packaged within 6 hours. **Caveat for the security team:** when nothing is pending the matrix is empty and `build` is skipped, so already-published extension tags are never rebuilt on a schedule. Base-image CVE fixes therefore reach an extension only when that extension cuts its next release — a manual `workflow_dispatch` does not help, since it resolves the same empty matrix. |

Net effect: a HIGH/CRITICAL CVE fixed in a base image is picked up as soon as its
digest-bump PR auto-merges, and **at most one weekly cycle** later even if no digest
changed; the Trivy gate ensures the rebuilt image is verified clean before it ships. The
one exception is `web-extensions`, which has no rebuild-only schedule — see its row above.

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
- **`ocis-workflows`** — nothing to bump: the pipeline resolves upstream `main` HEAD on
  every run, so the published image follows upstream automatically. This changes to the
  deliberate `ocis` model once upstream cuts semver releases.
- **`web-extensions`** — nothing to bump either, for the opposite reason: the `prepare`
  job discovers each new upstream *release tag* by polling (§2e), so packaging follows
  upstream automatically. The only manual control is
  `web-extensions/scripts/ignored-releases.txt`, a curated list of upstream tags that
  must never be built (bootstrap-era releases nobody wants packaged). Nothing is added
  to or expires from it automatically.

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
  - `ocis-workflows`: on any non-PR event (`main` + weekly schedule).
  - `web-extensions`: on any non-PR event (`main`, the 6-hourly release poll, or a manual
    `workflow_dispatch` — see the branch caveat in §2b), and only
    for releases that are not on Docker Hub yet — a run with nothing pending pushes
    nothing at all.
  - Pull requests **build, scan and smoke-test but never push** — except in
    `web-extensions`, where a PR builds only what the poll happens to find pending, so a
    PR can be green without any image having been built or scanned (§2e).
- **The smoke test gates the push.** Before publishing, the freshly built image is run
  and validated:
  - `server`: poll `http://localhost:8080/status.php` for HTTP 200 (≈60 s window) and
    assert the reported `.versionstring` equals the tag.
  - `ocis`: poll `https://localhost:9200/status.php` (with `OCIS_INSECURE=true`) and
    assert `.productversion`.
  - `ocis-workflows`: start `app server` and poll `http://localhost:9109/healthz` for
    HTTP 200. There is no version to assert against, as the build has no version input.
  - `web-extensions`: poll `http://localhost:8080/<package>/manifest.json` for HTTP 200 —
    every extension ships a `public/manifest.json` that Vite copies verbatim into `dist/`,
    so one probe works for every extension with no per-extension special-casing. No
    version assertion: the built assets expose no version endpoint.
  - supporting images: a one-shot command check inside the container —
    `ubuntu` asserts `VERSION_ID` from `/etc/os-release`, `php` runs
    `php --version | grep -qF 'PHP <ver>'`, and `base` runs
    `php -r "echo 'OK';" | grep -q OK`.
- **Authentication:** `DOCKERHUB_USERNAME` (org variable) + `DOCKERHUB_TOKEN` (secret).
- **Docker Hub description sync:** on publish, the reusable `docker-hub-desc.yml`
  workflow pushes each repo's `README.md` as the Docker Hub image description, so the
  published description is always the reviewed, in-repo README. In `web-extensions` that
  job declares `needs: build`, so it is skipped along with `build` whenever no release is
  pending: a README fix merged there updates Docker Hub only when the next release
  happens to be built in the same run.

---

## 7. Security control summary

| Concern | Control | Mechanism | Cadence | Where enforced |
|---------|---------|-----------|---------|----------------|
| Base-OS CVEs | Digest-pinned bases, auto-bumped; auto-merged except in the two Dependabot repos | Renovate (`owncloud-ops/renovate-presets:docker`); Dependabot in `ocis-workflows` and `web-extensions`, which have no Renovate config | Continuous, auto-merge on green CI (Renovate); weekly, human-merged (Dependabot) | `.renovaterc.json` in `server`, `ocis`, `base`, `php`, `ubuntu`; `.github/dependabot.yml` elsewhere |
| OS-package CVEs | Refresh packages at build time (per-image) | `apt-get upgrade` in `owncloud/ubuntu` (cascades to php/base/server); `apk upgrade` in the oCIS, oCIS Workflows and Web Extensions runtimes | Every build / base-digest bump | `ubuntu`, `ocis`, `ocis-workflows`, `web-extensions` `Dockerfile.multiarch` (for `web-extensions` only when a release is pending — §5c) |
| Go stdlib CVEs in from-source images | Digest-pin the compiling toolchain, auto-bumped | `golang:*-alpine` bumped by Renovate (`ocis`) or Dependabot (`ocis-workflows`); Trivy flags `stdlib` against the shipped binary | Continuous, auto-merge on green CI (`ocis`); weekly, human-merged (`ocis-workflows`) | `ocis`, `ocis-workflows` `Dockerfile.multiarch` |
| Unpatched images | Rebuild even with no code change | Scheduled workflow rebuilds | Weekly (all except `web-extensions`, which rebuilds only on a new upstream release — §5c) + daily (ocis-rolling) | `main.yml` / `rolling.yml` |
| Shipping a vulnerable image | Block publish on HIGH/CRITICAL | Trivy scan, `exit-code: 1`, `ignore-unfixed` | Every build incl. PRs | shared `docker-build*.yml` |
| Accepted/unfixable CVEs | Documented, scoped exceptions | `.trivyignore` w/ justification | Reviewed each maintenance | per-repo/per-version files |
| Reproducibility / supply chain | Immutable pins | SHA256 base digests; full-SHA Action pins | Continuous | Dockerfiles + workflows |
| CI supply chain | Trusted, pinned Actions | Dependabot bumps; OSPO allowlist | Weekly | `.github/dependabot.yml`, OSPO policy |
| Broken release | Runtime verification before push | Smoke test (status endpoint + version assert) | Every build | shared `docker-build*.yml` |
| Credential leakage | Build secrets, not layers | `--mount=type=secret` (Freexian mirror) | Every build | `php/v22.04/Dockerfile.multiarch` |

---

## 8. References

**Image repositories**

- [`owncloud-docker/server`](https://github.com/owncloud-docker/server)
- [`owncloud-docker/ocis`](https://github.com/owncloud-docker/ocis)
- [`owncloud-docker/ocis-workflows`](https://github.com/owncloud-docker/ocis-workflows)
- [`owncloud-docker/web-extensions`](https://github.com/owncloud-docker/web-extensions)
- [`owncloud-docker/base`](https://github.com/owncloud-docker/base)
- [`owncloud-docker/php`](https://github.com/owncloud-docker/php)
- [`owncloud-docker/ubuntu`](https://github.com/owncloud-docker/ubuntu)

**Shared CI (in the `ubuntu` repo)**

- `.github/workflows/docker-build.yml` — cross-compile build, scan, smoke test, publish
- `.github/workflows/docker-build-native.yml` — per-arch build + manifest merge
  (oCIS, oCIS Workflows, Web Extensions)
- `.github/workflows/docker-hub-desc.yml` — Docker Hub description sync

**Dependency automation**

- `.renovaterc.json` in `server`, `ocis`, `base`, `php`, `ubuntu` → preset [`owncloud-ops/renovate-presets`](https://github.com/owncloud-ops/renovate-presets)
- `.github/dependabot.yml` (GitHub Actions updates everywhere; **also** the Docker
  base-image updates in `ocis-workflows` and `web-extensions`, which have no Renovate
  config — see §5a)

**Vulnerability exceptions**

- `.trivyignore` (repo root) and per-version files, e.g. `server/v22.04/<version>/.trivyignore`, `ocis/v8/.trivyignore`
