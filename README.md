# dockge-for-tos

> TerraMaster TOS 7 App Center — **Docker application** package for
> [Dockge](https://github.com/louislam/dockge).

## What this is

A four-file TOS Docker application archive (`dockgedocker.tar.gz`) that installs
Dockge on TOS 7 through the App Center:

- `config.ini` — TOS application metadata (`application_type: docker`, id `dockgedocker`)
- `dockgedocker.lang` — 23-language superset (TOS requires 14)
- `dockgedocker.svg` — application icon (1584 bytes, 5 SVG elements)
- `docker-compose.yml` — one service, image `louislam/dockge:1.5.0`

## How it works

| | |
|---|---|
| Container image | `louislam/dockge:1.5.0` (Docker Hub, used unmodified) |
| Open | `http://<NAS-IP>:8004/` (container port 5001) |
| Runs as | non-root, `user: "1000:1000"` |
| Persistent data | `/app/data` → `/Volume1/DockerAppData/dockgedocker/app_data`, `/Volume1/DockerAppData/dockgedocker/stacks` → `/Volume1/DockerAppData/dockgedocker/stacks` |
| Logs | `docker logs dockgedocker` (stdout/stderr) |

The container needs no Docker-external state at all: everything it owns is in
`app_data/` (its SQLite database with settings, users and stack metadata) and
in `stacks/` (the compose files it manages). Both are bind mounts, so
restarting or reinstalling the application keeps them. Neither directory is
visible in the TOS file manager (Docker applications have no shared folder),
so use SSH/SFTP, or Dockge's built-in editor, to drop files in or take a
backup.


## Design highlights

- **Non-root.** `user: "1000:1000"`; verified on real hardware (the container reports uid 1000).
- **Fixed image tag, never `:latest`**, Docker Hub only, no `privileged`, no `network_mode: host`, and no `cap_add`. This application does mount the Docker socket, in YAML long form - a deliberate, documented exception (compose comments + the package decision record D-006): it is a Docker control panel, so the socket is its entire function. It still runs unprivileged and mounts only the socket file itself. The long form matters: the installer chowns mount sources it can read as `src:dst`, and doing that to the host socket would hand Docker to every uid-1000 container on the NAS.
- **Health check** probing the service on container port 5001.
- **Deterministic build**: GNU tar with uid/gid 0 and zeroed timestamps, gzip without a timestamp, plus the review-standards self-check above.
- **Stacks directory: container path == host path.** Dockge spells out a hard requirement - "Left Stacks Path === Right Stacks Path" - because it hands the compose file path to the Docker CLI. The stacks folder is therefore mounted at its own absolute host path (`/Volume1/DockerAppData/dockgedocker/stacks` inside and outside the container), so a relative volume like `./data` in a stack resolves to the same directory for the daemon as for Dockge.
- **Measured non-root fit.** The upstream image carries no `user:`, so the probe ran it as uid 1000: it exits with `EACCES mkdir /opt/stacks` when the stacks directory is not mounted. With both directories mounted (and therefore owned by the application user) the unmodified image serves HTTP 200 and reports `healthy` as uid 1000 - no entrypoint wrapper needed.
- **Socket exception, documented.** Only `/var/run/docker.sock` is mounted; `group_add: ["0"]` exists solely because the TOS socket is `root:root 0660`. The rationale, the least-privilege analysis and the reproducible evidence are in `docs/SOCKET-EXEMPTION.md` and decision D-006; the build gate refuses the socket for every application that does not declare this exception.

## Runtime file manifest

Everything the application writes on the NAS lives inside its own data
directory; the container-internal `/tmp` is discarded with the container. The
table lists what is created and how it behaves. Nothing else is written.

| Path | Purpose | Created when | Growth bound | Lifecycle |
|---|---|---|---|---|
| `app_data/dockge.db` | settings, users, stack metadata and the JWT secret (SQLite) | first start | grows slowly with the number of stacks and stored passwords | never overwritten; the application owns it - back it up |
| `app_data/dockge.db-shm` | SQLite shared-memory index (created while the service runs) | first start | bounded, fixed size | transient; recreated automatically |
| `app_data/dockge.db-wal` | SQLite write-ahead log (created while the service runs) | first start | bounded; checkpointed into dockge.db automatically | transient; recreated automatically |
| `app_data/db-config.json` | which database backend Dockge uses (SQLite by default) | first start | static | never overwritten; safe to delete (falls back to SQLite) |
| `stacks/` | your compose files - one directory per stack, plus .env files and anything you create there | when you create the first stack in the UI | bounded by your own stacks | user owned; never touched by the package - back it up |
| `/Volume1/DockerAppData/dockgedocker/app_data` | Dockge application data (everything the container writes outside /tmp) | on first start / on demand | bounded by your own data | persistent; kept on uninstall unless you ask to delete it |
| `/Volume1/DockerAppData/dockgedocker/stacks` | Dockge application data (everything the container writes outside /tmp) | on first start / on demand | bounded by your own data | persistent; kept on uninstall unless you ask to delete it |

## Build

```bash
scripts/build.sh                  # -> out/dockgedocker.tar.gz + .sha256
scripts/build.sh 1.5.0-2   # packaging iteration bump
scripts/build.sh 1.5.0-1 aarch64
```

The build stages the four files, substitutes the version/platform placeholders,
removes CRLF/BOM/AppleDouble, then runs
`tools/verify.py`, a review-standards self-check (archive layout, JSON validity,
required fields, 14+ language coverage, version consistency, Docker-Hub-only
images, fixed tag, reserved/recommended host ports, non-root user, per-service
healthcheck/TZ/restart, data path below the application data root, never the
`/Volume*` wildcard, comment-free sequence blocks, `x-app-meta` position, no
literal secrets and the SVG size/element limits).

## Local verification

```bash
./TOSAppSelfTestingTool -c out/dockgedocker.tar.gz   # official spec check
bash test/install-and-verify.sh                  # on-device E2E (needs root)
```

> Docker applications cannot be side-loaded through the App Center's manual
> install page (it accepts `.deb` only); the official self-testing tool's `-i`
> mode or `docker compose` are the only ways to install one for testing.

## Store submission checklist

1. Public GitHub repo (code + README only).
2. Release **tag `v1.5.0-1`**, assets `dockgedocker.tar.gz` +
   `dockgedocker.tar.gz.sha256` (Docker asset naming: `<app_id>.tar.gz`, **no
   platform suffix**).
3. Developer platform → Add Application: ID `dockgedocker`, package type Docker,
   repository URL, architecture `x86_64`.
4. Version Management → Add Version `1.5.0-1` → automated validation → review.

## License / branding

Dockge is distributed by its upstream authors under **MIT**. The container image is the official image from Docker Hub (`louislam/dockge:1.5.0`) and is used unmodified; no upstream source code is redistributed here. This packaging is an independent community submission by Moechz and is not affiliated with the upstream project or TerraMaster.

## Limitations

Dockge manages **Docker** containers only, and only what the TOS DockerEngine daemon can do; it is not a general NAS control panel.
The stacks folder (`/Volume1/DockerAppData/dockgedocker/stacks`) is not browseable in the TOS file manager - reach it over SSH/SFTP or through Dockge's editor. Back it up before removing the application with data.
Stacks that use **relative** volume paths are written relative to the compose file, that is, into the stacks folder itself; use absolute `/Volume1/...` host paths in your own stacks to keep data where you expect it.
The container terminal runs as the container's own user, not as the NAS administrator.

---

Packaged for TOS 7 by Moechz. Upstream image used unmodified from Docker Hub.
