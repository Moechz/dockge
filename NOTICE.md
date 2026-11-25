# NOTICE — third-party components, licences and attributions

This package (`dockgedocker`, TOS 7 Docker application) is an independent packaging by
Moechz of the upstream **Dockge** project. It is not affiliated with, sponsored by or
endorsed by the upstream project or TerraMaster.

## 1. Upstream application — Dockge

- Project: <https://github.com/louislam/dockge>
- Upstream release used: **1.5.0** (the latest release as of 2026-11-25)
- Licence: **MIT License**, copyright © 2023 Louis Lam
- Verified against the upstream repository's `LICENSE` file and the GitHub licence API
  (`spdx_id: MIT`) on 2026-11-25 — this is not copied from older package metadata.
- Relationship: the upstream application is used **unmodified**, inside the upstream
  container image. No upstream source code is redistributed in this package.
- The full MIT text is reproduced in [`LICENSE`](LICENSE), together with the packaging
  notice required by the MIT licence.

## 2. Container image — `louislam/dockge:1.5.0`

- Registry: Docker Hub, repository `louislam/dockge`, fixed tag `1.5.0` (never `:latest`)
- Maintainer: Louis Lam (`louislam`)
- Multi-architecture OCI index digest:
  `sha256:335c6368b880ecc203236ed89e6e5232e0d6578e8ef5920e4a502390451502bf`
- Platform manifests:
  - `linux/amd64`: `sha256:edee39b74f238378ebfe0d9d12b3833f9d1e123aea9dfaf5149ad45c9afe2527`
  - `linux/arm64`: `sha256:6c50c1c501c074cf264c5c83b5a570e2b966cb43eff6d5d76c43b4a8c0855a3c`
- Image build date: 2025-03-30. Base runtime observed in the image: Debian 12.10 with
  Node.js 22.14.0, Yarn 1.22.22, dumb-init and the Docker CLI/Compose plugin.
- The image is pulled from Docker Hub at install time and is **not redistributed** in this
  package. Its licence is the upstream MIT licence; the base distribution and bundled
  third-party components carry their own licences, documented in the image metadata.

## 3. Components inside the image

The image bundles the Dockge application, Node.js, the Debian base userland, Yarn,
dumb-init and the Docker CLI. These components are shipped **inside the image only**;
they are not extracted, modified or redistributed as separate files by this package.
Their copyright notices and licences remain with their respective authors and are
available in the image's package metadata and upstream source tree.

## 4. This packaging

The files authored for this package — `docker-compose.yml`, `config.ini`,
`dockgedocker.lang`, `dockgedocker.svg`, the documentation (`README.md`, `PRIVACY.md`,
`NOTICE.md`, `docs/`) and the build/verification scripts — are released under the
**MIT License** (see `LICENSE`). They contain no upstream source code.

The packaging does not include `TOSAppSelfTestingTool`; the copy kept in the source
repository is TerraMaster's official utility, used unmodified for local validation and
not part of the submitted application archive.

## 5. Trademarks

“Dockge” is the upstream project's name. “TerraMaster”, “TOS” and “TNAS” are trademarks
or registered trademarks of their respective owners. Their use here is descriptive only
and does not imply endorsement. This package is an independent community build.

## 6. Icon

The application icon in `src/dockgedocker.svg` was drawn for this packaging. It does not
reproduce the upstream Dockge logo or any third-party logo.

## 7. Contact

Moechz — moechz@163.com — <https://www.moechz.com/>
