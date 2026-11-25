# Privacy Policy — Dockge for TerraMaster TOS (`dockgedocker`)

**Last updated:** 25 November 2026
**Data controller / contact:** Moechz — moechz@163.com
**Covered product:** the TOS application `dockgedocker` and its packaging (upstream Dockge
`1.5.0`, image `louislam/dockge:1.5.0`, used unmodified).

## 1. Summary

- The app runs **entirely on your NAS**. The developer does not receive, collect or have
  access to your data.
- No telemetry, no analytics, no advertising SDK and no account operated by the developer.
- The app talks to the **local Docker daemon** through `/var/run/docker.sock` (the declared
  socket exception), to an **optional update-check endpoint** (`dockge.kuma.pet`), and to
  **Docker Hub** when you pull images. These are the only outbound paths built into the
  app; everything else is a service you configure yourself.
- All data the app stores stays under `/Volume1/DockerAppData/dockgedocker/` on your NAS.

## 2. What data is processed, and where it is stored

Dockge is a web UI for Docker Compose stacks. Its own state and the files it manages are
kept in two bind mounts:

| Container path | Host path on the NAS | Contents |
|---|---|---|
| `/app/data` | `/Volume1/DockerAppData/dockgedocker/app_data` | SQLite database `dockge.db` (settings, user accounts with bcrypt password hashes, stack metadata, JWT signing secret, notification settings), `db-config.json`, and the SQLite `-wal`/`-shm` files while running |
| `/Volume1/DockerAppData/dockgedocker/stacks` | `/Volume1/DockerAppData/dockgedocker/stacks` | The compose files you create or scan, one directory per stack, plus any `.env` files, TLS keys and other files you put there |

The app also reads live data from the Docker daemon while you use it: container names,
images, states, logs and terminal output. That data is displayed in your browser and is
not copied into a developer-operated system. Container stdout/stderr is held by the Docker
logging driver on your NAS (`docker logs dockgedocker`), not by the developer.

Web-server access logs are not written by this package. Any reverse proxy or browser you
add may keep its own logs under your control.

## 3. Retention period

- **Application data** (`app_data/`, `stacks/`): kept until **you** delete it. There is no
  automatic expiry and no upload. Uninstalling the app keeps the folders unless you tick
  “delete data” in the App Center.
- **Container logs**: controlled by the Docker daemon's logging configuration on your NAS
  (default `json-file` without an explicit cap); rotate or prune them with Docker's own
  settings if you need a retention limit.
- **Browser session**: a JWT cookie/local storage entry, cleared when you log out or clear
  browser data.
- **Email to the developer**: if you write to `moechz@163.com`, the message is kept only as
  long as needed to answer it, then deleted.

## 4. Your rights and the deletion channel

Because the developer holds no copy of your data, you exercise these rights directly on
the NAS:

| Right | How to exercise it |
|---|---|
| Access / export | Copy `/Volume1/DockerAppData/dockgedocker/` — it is the complete state. The SQLite database opens with any SQLite client. |
| Correct | Change settings, users and stacks in Dockge's own web UI. |
| Delete | Uninstall the app with “delete data”, or stop the container and remove `/Volume1/DockerAppData/dockgedocker/` over SSH/SFTP. |
| Restrict / object | Do not start the app; remove its data directory. |
| Portability | The database and compose files are standard SQLite/YAML files; copy them to another NAS. |
| Assistance / complaint | Email **moechz@163.com** with the subject “Dockge data request”; a reply follows within 30 days. |

If you expose Dockge to other people, **you** are the controller of the accounts and stacks
you create; the developer is not a processor for them.

## 5. Security measures

- The container runs as **uid/gid 1000:1000**, never root, with `cap_drop: [ALL]` and
  `no-new-privileges:true`; there is no `privileged`, no `cap_add` and no host network.
- Only one host port is published: **8004** (container port 5001).
- Dockge requires you to create an administrator account on first start. Passwords are
  stored as bcrypt hashes; sessions use a JWT secret generated in the database. There is no
  default password.
- **Docker socket exception:** the container mounts `/var/run/docker.sock` because the app
  is a Docker control panel. Anyone who can log in to Dockge can control every container on
  this NAS. Set a strong password, keep the UI on your LAN or behind a TLS reverse proxy,
  and do not expose port 8004 to the internet. The full justification and the least-
  privilege boundary are in [`docs/SOCKET-EXEMPTION.md`](docs/SOCKET-EXEMPTION.md).
- The host socket is mounted in YAML long form, so the platform installer does not chown
  it; installing or removing the app does not change the host socket's owner or mode.
- The image tag is pinned (`louislam/dockge:1.5.0`); the package is rebuilt when upstream
  publishes a security update. Upstream 1.5.0 is the latest release as of 2026-11-25; the
  CVE follow-up is recorded in [`CHANGELOG.md`](CHANGELOG.md).
- Backups of `app_data/` and `stacks/` contain password hashes, JWT secrets, `.env` files
  and possibly TLS keys — protect them like any other credential store.

## 6. Third-party sharing and API disclosure (C8)

The developer sells, rents or shares nothing. The app itself contacts only the endpoints
below. “Data sent” describes what leaves the NAS, not what is displayed locally.

| Service | Endpoint / trigger | Data sent | Destination / region | Can it be disabled? |
|---|---|---|---|---|
| **Local Docker daemon** | Unix socket `/var/run/docker.sock` (no network) | Compose file contents, stack and container names, image names, start/stop/pull commands, log and terminal streams | Your own NAS only | No — it is the app's core function |
| **Update check** | `https://dockge.kuma.pet/version`, HTTPS GET at startup and every 48 hours | The HTTP request itself (source IP, User-Agent, TLS metadata); no stack, account or file data. The returned version string is compared locally | The Dockge maintainer's service (`kuma.pet`, outside your LAN, typically US-hosted) | Yes — Settings → disable “Check Update” |
| **Docker Hub** | `registry-1.docker.io` (or the registry mirror configured in your Docker daemon) when you pull/update images | Image name/tag and an anonymous pull request; source IP and User-Agent | Docker Hub (US) or your configured mirror | Only by not pulling images; the app does not pull in the background |
| **Remote Dockge agents** (optional, only if you add one) | The `ws://`/`wss://` or `http(s)://` URL you enter under Settings → Agents | Stack metadata, commands, logs and terminal I/O needed for that agent | The host you choose | Yes — do not add an agent |
| **Services referenced by your own stacks** | Whatever URLs/images your compose files contain, when you deploy them | Determined by your compose file | The provider you choose | Yes — edit your stack |

Additional notes:

- The app does **not** call GitHub's API in the background. The “Check Update On GitHub”
  link only opens a browser page when you click it.
- There is no third-party analytics, crash reporter, advertising or tracking pixel.
- Docker Hub and `kuma.pet` receive the NAS's public IP as part of any HTTPS request; that
  is inherent to the connection and is not linked by the app to your account.

## 7. Children

The app is a self-hosted utility for administrators and is not directed at children.

## 8. Changes

Material changes are published in this file and reflected in the “Last updated” date. The
version history is in [`CHANGELOG.md`](CHANGELOG.md).

## 9. Contact

Moechz — **moechz@163.com** — <https://www.moechz.com/>
Public policy URL: <https://github.com/Moechz/dockge/blob/main/PRIVACY.md>
