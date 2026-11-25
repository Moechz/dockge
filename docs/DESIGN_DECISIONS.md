# DESIGN_DECISIONS — Dockge TOS package (`dockgedocker`)

> Numbered decision log. Read the “why” before changing a decision. This file is part of
> the public repository and is submitted with the review package where relevant.

## D-001 Application form: TOS 7 Docker application (four-file archive)

**Decision:** `application_type: docker`; the archive contains exactly `config.ini`,
`dockgedocker.lang`, `dockgedocker.svg` and `docker-compose.yml`.

**Consequences:**
- Lifecycle, image and logs are owned by Docker Engine and App Center. Docker applications
  do not use TNAS shared folders, and App Center does not upgrade them in place: a new
  version is a new submission; user data is kept unless “delete data” is chosen.

## D-002 Absolute host paths below `/Volume1/DockerAppData/dockgedocker/`

**Decision:** every bind mount uses an absolute `/Volume1/...` host path; `/Volume*`
wildcards and `./` relative mounts are forbidden.

**Consequences:**
- The platform does not expand `/Volume*`, so a wildcard would create a literal directory.
- Relative mounts are not chowned by the installer and a uid-1000 container cannot write
  into them. Only absolute `/Volume1/...` sources are created and owned by the app user.
- The stacks directory is mounted at the **same absolute path** inside the container because
  Dockge requires “Left Stacks Path === Right Stacks Path” when it invokes the Docker CLI;
  a relative `./data` volume in a user stack then resolves identically for Dockge and the
  daemon.

## D-003 Non-root container

**Decision:** `user: "1000:1000"`, `cap_drop: [ALL]`, `security_opt: ["no-new-privileges:true"]`.

**Consequences:**
- The upstream image has no `user:` and its default entrypoint can run as uid 1000 when
  `/app/data` and the stacks directory are mounted. Measured: without the stacks mount the
  image exits with `EACCES mkdir /opt/stacks`; with both mounts it serves HTTP 200 and
  reports healthy as uid 1000.
- No capability is needed: the app only serves HTTP and talks to the Docker API through the
  socket, which is permission-checked by the daemon rather than by Linux capabilities.

## D-004 No comments inside compose sequence blocks

**Decision:** keep comments above `ports:` / `volumes:` and never between `-` items.

**Consequences:**
- The official installer parses the compose file line by line; a comment inside a sequence
  makes it skip its permission sync. Mounted directories would stay `root:root` and the
  non-root container could not write into them. This is a real, hard-to-see failure mode,
  so the build's `tools/verify.py` and the maintainers' review both check it.

## D-005 Port `8004` → `5001`

**Decision:** publish one host port, `8004`, to container port `5001`.

**Consequences:** the port is in the TOS recommended 8000–19999 range and is not reserved.
`config.ini.path`, the compose `ports` entry and `x-app-meta.web.port` are kept identical.
Port collisions with other store applications are expected in a large catalogue and are not
treated as a defect (RUNBOOK §12); the user decides whether to install two apps that share a
port.

## D-006 Docker socket + `group_add: ["0"]` (declared exception)

**Decision:** mount `/var/run/docker.sock` as a YAML long-form bind mount and add
`group_add: ["0"]`. The full four-part justification is
[`SOCKET-EXEMPTION.md`](SOCKET-EXEMPTION.md).

**Why it cannot be removed:** Dockge is a Docker control panel. Listing stacks, starting and
stopping them, `compose up/down`, pulling images, logs and the terminal are all Docker Engine
API calls. Without the socket the container starts and serves a UI that cannot perform any
action — a functional slice of the upstream application, which the store rules forbid.

**Why `group_add: ["0"]`:** on the documented TOS socket (`root:root`, mode 0660) a uid-1000
process cannot open the socket without the socket's own gid. Measured: without it, the
container gets `permission denied ... dial unix /var/run/docker.sock`; with it, `docker ps`
and `curl --unix-socket` succeed while `id` still shows uid 1000.

**Why long syntax:** the official installer chowns every bind-mount source it parses as
`src:dst`. With the short form `- /var/run/docker.sock:/var/run/docker.sock` the host socket
was measured to become `1000:0 660` after installation, giving Docker access to every
uid-1000 container on the NAS. The long form is an explicit bind mount but is not a chown
target; the build gate refuses the short form and any directory mount.

**Boundary (must not be expanded):** only the socket file; never `/var/run`, never the
Docker data root; no `privileged`, no `cap_add`, no host network; the container stays
`1000:1000` with all capabilities dropped. See `SOCKET-EXEMPTION.md` §6 for the TOS 7.1
socket-group caveat.

## D-007 Sensitive-path audit (review code S7)

**Decision:** the package mounts no `/etc`, `/root` or `/boot` path and does not declare
them as volumes.

**Evidence:**
- The compose file has exactly three binds: `app_data/`, `stacks/` and the socket file.
- `docker image inspect louislam/dockge:1.5.0` reports `Config.Volumes = {"/app/data": {}}`.
- The package does not build a Dockerfile, so it cannot `COPY` or `VOLUME` those paths.
- `cap_drop: [ALL]` and `no-new-privileges` further bound what the container can do.

**Consequences:** the only host privilege is the declared socket exception; the audit is
kept in `SOCKET-EXEMPTION.md` §5 and in the README so reviewers can re-check it without
reading the compose file.

## D-008 Pinned upstream image and CVE follow-up

**Decision:** keep `image: louislam/dockge:1.5.0` (the latest upstream release as of
2026-11-25) and do not fork the image.

**Consequences:** the image is used unmodified. Upstream has not published a newer release,
so a patched base can only arrive from upstream; the package will be rebuilt when it does.
Mitigations already applied: non-root, all capabilities dropped, no-new-privileges, no
privileged mode, no host network, one published port, socket-only exception. The T1 status
is recorded in `CHANGELOG.md` and the submission notes.
