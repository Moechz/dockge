# dockge-for-tos

> TerraMaster TOS 7 App Center — **Docker application** package for
> [Dockge](https://github.com/louislam/dockge), the self-hosted Docker Compose
> stack manager.

## What this is

A four-file TOS Docker application archive (`dockgedocker.tar.gz`) that installs the
official Dockge image on TOS 7 through the App Center:

- `config.ini` — TOS application metadata (`application_type: docker`, id `dockgedocker`)
- `dockgedocker.lang` — 23-language description file (TOS requires 14)
- `dockgedocker.svg` — application icon
- `docker-compose.yml` — one service, image `louislam/dockge:1.5.0`, non-root,
  capability-dropped, one published port, healthcheck, `x-app-meta`

The image is pulled from Docker Hub at install time and used **unmodified**. This
repository contains the packaging, documentation and build/verification scripts; it does
not contain Dockge source code or the container image.

| | |
|---|---|
| Container image | `louislam/dockge:1.5.0` (Docker Hub) — OCI index `sha256:335c6368…1502bf` |
| Upstream release | Dockge **1.5.0** — the latest release as of 2026-11-25 |
| Open | `http://<NAS-IP>:8004/` (container port 5001) |
| Runs as | non-root, `user: "1000:1000"`, `cap_drop: [ALL]`, `no-new-privileges` |
| Persistent data | `/Volume1/DockerAppData/dockgedocker/` (`app_data/`, `stacks/`) |
| Logs | stdout/stderr → `docker logs dockgedocker` |
| Login | set up the administrator account on first start; no default password |

## Features (what actually ships)

The package delivers the upstream Dockge 1.5.0 features, without additions or removals:

- create, edit, start, stop, restart and delete Docker Compose stacks;
- pull updated images for a stack (`compose pull` + `up`);
- live logs and an interactive web terminal for a stack's containers;
- an interactive editor for `compose.yaml`;
- convert a `docker run …` command into a compose file;
- keep compose files as ordinary files in a directory you can also reach over SSH/SFTP.

It is **not** a general NAS control panel: it manages Docker containers through the local
Docker Engine, not files, users, shares or the TOS system.

## Repository layout

```
.
├── src/                       # the four files that go into the .tar.gz
│   ├── config.ini
│   ├── dockgedocker.lang
│   ├── dockgedocker.svg
│   └── docker-compose.yml
├── scripts/
│   ├── build.sh               # one architecture
│   └── build-dual.sh          # x86_64 + aarch64, official -c for both
├── tools/verify.py            # review-standards gate run by build.sh
├── test/install-and-verify.sh # on-device E2E (not needed for normal use)
├── docs/
│   ├── SOCKET-EXEMPTION.md    # four-part justification for docker.sock
│   └── DESIGN_DECISIONS.md    # decision log, including D-006 / D-007
├── PRIVACY.md                 # privacy policy and third-party API table
├── NOTICE.md                  # licences, image digests, trademarks
├── CHANGELOG.md
└── out/                       # build products (generated, not committed)
```

## Install, first start, upgrade

1. Install **DockerEngine** first if it is not already installed. The App Center declares
   the dependency and will offer to install it.
2. Install **Dockge** from the App Center (or validate the archive with
   `TOSAppSelfTestingTool -c` and install it with `-i` on a test NAS).
3. Open `http://<NAS-IP>:8004/`. On first start Dockge asks you to create an administrator
   account. There is no default login.
4. Create a stack in the UI, or drop a directory containing `compose.yaml` into
   `/Volume1/DockerAppData/dockgedocker/stacks/` and use **Scan Stacks Folder**.

Docker applications are not upgraded in place by the App Center: a new version is a new
package/submission. Uninstalling and reinstalling keeps `app_data/` and `stacks/` unless
you explicitly choose to delete the data. Back them up before a reinstall.

## Data, backup and uninstall

Everything the application owns is in two bind mounts:

| Host path | Contents |
|---|---|
| `/Volume1/DockerAppData/dockgedocker/app_data/` | SQLite database (settings, users with bcrypt hashes, stack metadata, JWT secret), `db-config.json`, SQLite WAL/SHM while running |
| `/Volume1/DockerAppData/dockgedocker/stacks/` | your compose files, one directory per stack, `.env` files and anything else you put there |

Neither directory is visible in the TOS file manager (Docker applications do not use TNAS
shared folders). Use SSH/SFTP, or Dockge's built-in editor, to inspect or back them up.

- **Backup:** copy both directories (or snapshot the app data root) before uninstalling
  with data removal, before changing the database backend, and before major stack changes.
- **Restore:** stop the container, put the directories back, start it again.
- **Uninstall:** in the App Center, uninstall and tick “delete data” only if you want the
  data gone; otherwise the two directories remain and a reinstall picks them up.
- **Manual deletion:** stop the container and remove
  `/Volume1/DockerAppData/dockgedocker/` over SSH/SFTP. The host Docker socket and other
  applications are not touched.

Do not put relative volume paths (`./data`) in your own stacks: they resolve inside the
stacks directory. Use absolute `/Volume1/...` host paths so data lands where you expect it.

## Security model and the Docker socket exception

Dockge **is** a Docker control panel, so it needs `/var/run/docker.sock`. This is a
declared exception to the “no host privileges” rule, not a silent default:

- the container runs as **uid/gid 1000:1000**, never root;
- all Linux capabilities are dropped (`cap_drop: [ALL]`) and privilege escalation is blocked
  (`no-new-privileges`);
- no `privileged`, no `cap_add`, no host network, no devices;
- **only the socket file** is mounted — never `/var/run`, never the Docker data root;
- `group_add: ["0"]` is present only because the TOS socket is `root:root 0660`; a uid-1000
  process cannot open it otherwise;
- the socket is written as a YAML long-form bind mount so the platform installer does not
  chown the host socket to uid 1000 (measured failure mode with the short form).

The full justification, the least-privilege table, the reproducible checks and the TOS 7.1
socket-group caveat are in [`docs/SOCKET-EXEMPTION.md`](docs/SOCKET-EXEMPTION.md); the
decision record is D-006 in [`docs/DESIGN_DECISIONS.md`](docs/DESIGN_DECISIONS.md).

### Sensitive-path audit (review code S7)

- The package mounts **no** `/etc`, `/root` or `/boot` path.
- The upstream image declares **only** `VOLUME /app/data`; this repository does not build or
  modify the image, so it cannot `COPY` or `VOLUME` a sensitive path.
- The only host bind mounts are `app_data/`, `stacks/` and the socket file.
- `cap_drop: [ALL]` and `no-new-privileges` bound the container further.

> **Risk you accept by installing this app:** anyone who can log in to Dockge can control
> every container on this NAS. Set a strong administrator password, keep port 8004 on the
> LAN or behind a TLS reverse proxy, and do not expose it to the internet.

## Privacy and third-party services

The app runs entirely on your NAS; the developer operates no telemetry and receives no
data. Full details, retention, user rights, security measures and the deletion channel are
in [`PRIVACY.md`](PRIVACY.md).

| Service | Endpoint / trigger | Data sent | Destination | Disable |
|---|---|---|---|---|
| Local Docker daemon | `/var/run/docker.sock` (no network) | compose contents, stack/container names, commands, logs, terminal I/O | this NAS | no — core function |
| Update check | `https://dockge.kuma.pet/version`, startup + every 48 h | HTTP request metadata only (IP, User-Agent); no stack/account data | Dockge maintainer's service, outside the LAN | Settings → “Check Update” |
| Docker Hub | `registry-1.docker.io` or your mirror, when you pull images | image name/tag, anonymous pull request, IP/UA | Docker Hub (US) or your mirror | do not pull |
| Remote agent (optional) | the `ws(s)://` URL you add under Agents | stack metadata, commands, logs, terminal I/O | the host you choose | do not add an agent |
| Your own stacks | URLs/images in your compose files | whatever your stack sends | the provider you choose | edit the stack |

There is no advertising, analytics, crash reporter or tracking pixel. The “Check Update On
GitHub” link only opens a browser page when clicked; the app does not call GitHub's API in
the background.

## Build and verification

```bash
# both architectures, with the official TOSAppSelfTestingTool -c on each package
bash scripts/build-dual.sh 1.5.0-3

# one architecture (writes out/dockgedocker.tar.gz)
bash scripts/build.sh 1.5.0-3 x86_64
```

The build stages the four files, substitutes `@@VERSION@@`/`@@PLATFORM@@`, strips CRLF/BOM,
runs `tools/verify.py` (archive layout, metadata, 23-language coverage, version consistency,
Docker-Hub-only fixed tag, reserved/recommended port, non-root, no privileged/host-network/
cap_add, healthcheck/TZ/restart, data paths, the socket long-form gate, no literal secrets,
SVG limits), then packs a deterministic tar.gz and writes its `.sha256`.

Runtime verification by the packager uses an isolated staging instance (temporary data
directory and port; production `/Volume1/DockerAppData/dockgedocker` is never touched):
the container must become `healthy` as uid 1000, answer `HTTP 200` on its web port, and
show no `/etc`, `/root` or `/boot` mount in `docker inspect`. The on-device E2E script
`test/install-and-verify.sh` additionally asserts the host socket's owner/mode before and
after installation; it is for maintainers and is **not** run as part of a normal build.

## Review remediation — 1.5.0-3

This version responds to the 1.5.0-2 review. The full item-by-item record is in
[`CHANGELOG.md`](CHANGELOG.md); summary:

| Code | Finding | Change |
|---|---|---|
| C2 | Licence/attribution incomplete | `NOTICE.md` rewritten: MIT verified against upstream `LICENSE` + GitHub licence API (© 2023 Louis Lam), image index/arch digests, build date, what is and is not redistributed, packaging licence aligned to MIT in `LICENSE`, trademarks |
| C4 | Privacy policy incomplete | `PRIVACY.md` rewritten: retention, user rights, security measures, deletion channel, third-party sharing, contact |
| C8 | Third-party APIs not disclosed | explicit table for the local Docker daemon, optional `dockge.kuma.pet` update check, Docker Hub/mirror pulls, optional agents and user stacks |
| C9 / F5 | Description inconsistent / declared function not usable | `descript`/`important`/`release_note` rewritten to the actual 1.5.0 feature set in all 23 languages, no ASCII semicolons, `help` now points to the public privacy policy |
| S7 | Sensitive directories mounted | audit added to compose comments, README and `docs/SOCKET-EXEMPTION.md` §5: no `/etc`, `/root` or `/boot` mount, image volume is only `/app/data` |
| T1 | High/critical CVE | upstream 1.5.0 is still the latest release; the image is unmodified and cannot be patched by this package, so the pinned tag is kept and the follow-up is documented; mitigations are non-root, `cap_drop: [ALL]`, no-new-privileges, no privileged/host network, single port, socket-only exception |

The 1.5.0-2 review also reported an App Center online-installation check failure. The
package starts successfully in isolated staging (HTTP 200, uid 1000, healthcheck healthy,
no sensitive mount); the compose keeps the same single service and port `8004` (port
conflicts are not treated as a package defect, see RUNBOOK §12). The healthcheck start
period was raised to 60 s with 5 retries so a slow first pull cannot fail an installation
check.

## Limitations

- Dockge manages **Docker Compose stacks only**; it is not a general NAS control panel.
- The stacks folder is not browseable in the TOS file manager; use SSH/SFTP or Dockge's
  editor, and back it up before removing data.
- Relative volume paths in your own stacks resolve inside the stacks folder; use absolute
  `/Volume1/...` paths.
- The web UI is HTTP by default. Put it behind a TLS reverse proxy for remote access.
- The socket exception gives the app control of all containers on the NAS (see the risk
  note above).
- T1: upstream 1.5.0 (image built 2025-03-30, Node 22.14.0) is the latest release; a patched
  base can only come from upstream. See `CHANGELOG.md`.

## Licence and attributions

| Component | Licence | Notes |
|---|---|---|
| Dockge | MIT (© 2023 Louis Lam) | upstream application, used unmodified inside the image |
| Image `louislam/dockge:1.5.0` | MIT + bundled component licences | pulled from Docker Hub at install time |
| This packaging | MIT (see `LICENSE`) | compose file, metadata, translations, icon, documentation |

“Dockge” is the upstream project's name; “TerraMaster”, “TOS” and “TNAS” are trademarks of
their respective owners. This is an independent community package by Moechz and is not
affiliated with or endorsed by the upstream project or TerraMaster. Full details:
[`NOTICE.md`](NOTICE.md).

## Support

- Issues and questions: <https://github.com/Moechz/dockge/issues>
- Email: moechz@163.com
- Privacy: [`PRIVACY.md`](PRIVACY.md) · Notices: [`NOTICE.md`](NOTICE.md) ·
  Socket exemption: [`docs/SOCKET-EXEMPTION.md`](docs/SOCKET-EXEMPTION.md)
