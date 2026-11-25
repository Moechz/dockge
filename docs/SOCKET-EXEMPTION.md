# Docker socket exemption — Dockge (`dockgedocker`)

> Submitted with the review package. This application is a Docker control panel, so the
> socket is its core function rather than an optional privilege. The exception is
> declared, minimal and reproducible; the application still runs as a non-root user.
> Related decision: [`DESIGN_DECISIONS.md`](DESIGN_DECISIONS.md) D-006.

## 0. One sentence

Dockge (`louislam/dockge:1.5.0`) manages Docker Compose stacks, so every stack operation
is a Docker Engine API call over `/var/run/docker.sock`. This package mounts **only that
socket file**, adds `group_add: ["0"]` because the TOS socket is `root:root 0660`, runs the
container as **uid 1000**, drops **all capabilities**, and does **not** use `privileged`,
host networking, `cap_add`, `/var/run`, the Docker data root, or any `/etc`, `/root` or
`/boot` path.

## 1. Why the socket is unavoidable

- Dockge's whole feature set — list stacks, create/edit, start/stop/restart/delete,
  `compose up/down`, pull/update images, live logs and the interactive container terminal
  — is implemented as calls to the Docker Engine API. With the official image the only
  transport for that API is the Unix socket.
- The store rule requires the upstream image to be used **unmodified**. Removing the
  socket would leave a UI that opens but cannot do anything: a functional slice of the
  upstream application, and a fork of its documented deployment.
- Alternatives considered and rejected:
  1. **Remote Docker API over TCP/TLS** — TOS DockerEngine listens on the Unix socket only;
     enabling TCP would expose the daemon on the network, a larger risk than this mount.
  2. **Socket-proxy sidecar** — the proxy itself must mount the same socket, so the
     privilege surface is unchanged (and one more container/image is added).
  3. **A non-Docker runtime** — Dockge is a Docker Compose manager by definition; it is
     not applicable.

## 2. Least-privilege implementation

| Item | Value in this package | Why |
|---|---|---|
| User / group | `user: "1000:1000"` | Non-root; the upstream image works as uid 1000 when its data and stacks directories are mounted (measured, see §4) |
| Capabilities | `cap_drop: [ALL]` | The app needs no Linux capability; Docker API access is mediated by the daemon |
| Privilege escalation | `security_opt: ["no-new-privileges:true"]` | Blocks setuid/setgid escalation inside the container |
| Privileged mode | none | Forbidden by the platform and unnecessary |
| Host network | none | Single published port 8004 → container 5001 |
| `cap_add` / devices | none | No device or capability is requested |
| Socket | bind mount of `/var/run/docker.sock` **file only** | Mounting `/var/run` would expose pid files, locks and other runtime state |
| Supplementary group | `group_add: ["0"]` | The TOS socket is `srw-rw---- root:root 0660`; a uid-1000 process needs the socket's own gid to open it |
| Host writable paths | only `/Volume1/DockerAppData/dockgedocker/app_data` and `.../stacks` | The app's own data; same model as other Docker applications |

The container is therefore **not** root, **not** privileged, has **no** added capability,
and has exactly one host path outside its own data: the socket file.

## 3. Security hardening

- Login is mandatory: Dockge asks for an administrator account on first start; there is no
  default password and no anonymous API.
- Only one host port is published. The UI is HTTP by default; put it behind a TLS reverse
  proxy if it must be reached from outside the LAN.
- No port-forwarding proxy or Docker API bridge is included: the container cannot be used
  as a remote Docker API endpoint by third parties.
- The host socket is mounted with YAML **long syntax** (`type: bind` / `source` / `target`).
  The official installer chowns bind-mount sources it can read as `src:dst`; with the short
  form `- /var/run/docker.sock:/var/run/docker.sock` the host socket was measured to change
  from `root:root 0660` to `1000:0 660` after installation — that would hand Docker to every
  uid-1000 container on the NAS. The long form is still a fully explicit bind mount but is
  not a chown target. The build gate (`tools/verify.py`) refuses any other spelling.
- `group_add: ["0"]` is not redundant: `/var/run` is a tmpfs and the socket returns to
  `root:root 0660` after a NAS reboot, so the supplementary group must be in the container
  definition.
- Uninstalling (`docker compose down` plus deleting the container) does not modify the host
  socket's owner or mode, and leaves the data directories intact unless the user asks to
  delete them.

## 4. Reproducible evidence

The following commands are read-only with respect to the host (they run a short-lived
container and remove it); they are for reviewers who want to re-check the claims.

```bash
# 1) Host socket permissions on a factory TOS install.
stat -c '%n %U:%G %a' /var/run/docker.sock
# expected: /var/run/docker.sock root:root 660

# 2) Without the socket's gid, a uid-1000 container cannot reach the daemon.
docker run --rm --user 1000:1000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --entrypoint sh louislam/dockge:1.5.0 \
  -c 'id; curl -s --unix-socket /var/run/docker.sock http://localhost/_ping'
# id shows uid=1000 ... groups=1000
# curl fails: permission denied ... dial unix /var/run/docker.sock

# 3) With the socket's own gid the API works and the identity is still non-root.
docker run --rm --user 1000:1000 --group-add 0 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --entrypoint sh louislam/dockge:1.5.0 \
  -c 'id; curl -s --unix-socket /var/run/docker.sock http://localhost/_ping; echo'
# id shows uid=1000(node) gid=1000(node) groups=1000(node),0(root)
# curl prints OK
```

Additional facts checked for this submission:

- `docker image inspect louislam/dockge:1.5.0` → `Config.Volumes` contains **only**
  `/app/data`; there is no `/etc`, `/root` or `/boot` volume.
- The packaged compose file mounts only `app_data/`, `stacks/` and the socket; the data
  directories are under `/Volume1/DockerAppData/dockgedocker/`.
- Isolated staging of the built package (temporary data directory and port; no production
  data touched) starts the container as uid 1000, `healthcheck = healthy`, and serves
  `HTTP 200` on the app port. Its `docker inspect` mount list contains no sensitive system
  path.
- The official `TOSAppSelfTestingTool -c` passes for both architecture packages.

## 5. Sensitive-path audit (review code S7)

This section answers the S7 finding explicitly.

| Check | Result |
|---|---|
| `/etc`, `/root`, `/boot` mounted by the package | **No.** The compose file has three bind mounts: the app data directory, the stacks directory and `/var/run/docker.sock`. |
| Those paths declared as `VOLUME` in the image | **No.** Image config: `Volumes: {"/app/data": {}}` only. |
| `COPY` of those paths in a Dockerfile inside this package | **No.** This package only ships a compose file, metadata, translations and an icon; it does not build or modify the image. |
| Other host paths writable by the container | **No.** The container writes only into its two mounted app directories (and its own `/tmp`, discarded with the container). |
| Extra capabilities / privileges | **No.** `cap_drop: [ALL]`, `no-new-privileges`, non-root, no `privileged`, no `cap_add`, no host network. |

**Boundary of the exception:** the socket is the *only* host-level privilege. It is not a
general-purpose privilege escalation: it is the API the application exists to call, and it
is bounded by what the Docker daemon permits. The package does not mount `/var/run`, does
not mount the Docker data root, does not add a proxy, and does not change the host socket's
permissions.

## 6. Known operational caveat (gid variants)

The documented exception assumes the TOS socket group is **gid 0** (`root:root 0660`), which
is the factory configuration and the one measured for this package. Some TOS 7.1 systems
run DockerEngine with `"group": "admin"` in `daemon.json`; the socket then becomes
`root:admin 0660` and a container that only has gid 0 cannot open it. This is a host
configuration difference, not a package privilege request. Do **not** silently add the
`admin` gid: that would broaden the container's group membership beyond the declared
exception. Instead, align the DockerEngine socket group with the documented `root:root`
layout (remove the `"group"` override and restart DockerEngine), or explicitly decide to
grant the matching gid and update this document and `PRIVACY.md` at the same time.

If the socket cannot be opened, Dockge still starts and its UI reports a Docker connection
error; the package's own healthcheck (HTTP on the web port) stays green because the web
server is healthy. That is intentional: a host-side socket-permission problem must not put
the container into a restart loop.
