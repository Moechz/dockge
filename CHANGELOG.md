# Changelog

All notable user-visible changes to the Dockge TOS application package.

## 1.5.0-3 — 2026-11-25

Remediation of the 1.5.0-2 App Listing Review (x86_64). The version keeps upstream Dockge
`1.5.0` and the same host port `8004`; no functionality is removed.

### Compliance

- **C2 (intellectual property):** `NOTICE.md` rewritten. Upstream licence verified as
  **MIT** (© 2023 Louis Lam) against the upstream `LICENSE` and the GitHub licence API,
  not copied from old metadata. Added the OCI index digest and the amd64/arm64 manifest
  digests, the image build date, an explicit statement of what is and is not redistributed,
  trademark attribution, and a packaging licence that matches `LICENSE` (MIT).
- **C4 (privacy policy):** `PRIVACY.md` rewritten with retention period, user rights
  (access/correct/delete/portability), security measures, deletion channel and contact.
  It also states the real Docker-socket behaviour instead of the earlier incorrect “no
  Docker socket” sentence.
- **C8 (third-party APIs):** explicit disclosure table for the **local Docker daemon**
  (`/var/run/docker.sock`, no network), the **optional update check**
  (`https://dockge.kuma.pet/version`, every 48 h, disableable), **Docker Hub**/mirror
  image pulls, optional remote agents and services referenced by the user's own stacks.
- **C9 / F5 (description):** `descript`, `important`, `release_note` and the `zh-cn`/
  `zh-hk` translations now describe the features Dockge 1.5.0 actually delivers
  (create/edit/start/stop/restart/delete stacks, pull updates, live logs, web terminal,
  interactive editor, `docker run` conversion). Removed the vague “watch” claim. No ASCII
  semicolons in any language value; the rendered `.lang` was checked for truncation and
  the 512-character limits.
- **S7 (sensitive directories):** audit added to the compose header, `README.md` and
  `docs/SOCKET-EXEMPTION.md` §5. The package mounts no `/etc`, `/root` or `/boot` path;
  the upstream image declares only `VOLUME /app/data`; the only host binds are
  `app_data/`, `stacks/` and the socket file.
- **T1 (CVE):** upstream Dockge `1.5.0` is still the latest release (checked GitHub
  releases on 2026-11-25; image built 2025-03-30, Node 22.14.0). The image is used
  unmodified, so a patched base can only arrive from upstream; the pinned tag is kept and
  the follow-up is recorded here. Mitigations already in place: non-root,
  `cap_drop: [ALL]`, `no-new-privileges`, no privileged mode, no host network, one port.
- **Installation check:** the 1.5.0-2 review reported an App Center online-installation
  failure. The package starts in isolated staging (HTTP 200, uid 1000, healthy) and has no
  sensitive mount; the single service, image and port `8004` are unchanged. The healthcheck
  start period was raised to 60 s with 5 retries so a slow first image pull cannot trip the
  installation check.

### Added

- `docs/SOCKET-EXEMPTION.md` — the four-part exemption material for the Docker socket
  (scenario justification, least-privilege table, hardening, reproducible evidence),
  including the S7 sensitive-path audit and the TOS 7.1 socket-group caveat.
- `docs/DESIGN_DECISIONS.md` — decision log D-001…D-008, including the socket exception
  (D-006), the sensitive-path audit (D-007) and the pinned-image/CVE decision (D-008).
- `cap_drop: [ALL]` and `security_opt: ["no-new-privileges:true"]` in the compose service.
- Public `docs/` in the repository (removed from `.gitignore`) so reviewers can read the
  exemption and decision records.

### Changed

- `tosapp.json` / `config.ini`: `help` points to the public `PRIVACY.md`; version bumped to
  `1.5.0-3`; duplicate `Official` key removed from `config.ini`; English and Chinese
  descriptions rewritten.
- `README.md`: repository layout, real feature list, privacy/API summary, sensitive-path
  audit, build/verification, remediation table, data/uninstall instructions and the
  socket risk statement.
- `.lang` re-rendered from `tosapp.json` (23 languages).

## 1.5.0-2 — 2026-11-16

### Fixed

- Resubmission build with the Docker socket exception declared in the spec and in the
  compose file, the socket mounted in YAML long form, and `group_add: ["0"]` for the TOS
  socket group. Published as `dockgedocker_x86_64.tar.gz`.

## 1.5.0-1 — 2026-09-27

### Added

- First release. Dockge (`louislam/dockge:1.5.0`, Docker Hub official image, unmodified)
  packaged as a TOS 7 Docker application, published on port 8004 and running as a non-root
  user (1000:1000).
- Persistent data under `/Volume1/DockerAppData/dockgedocker/`.
