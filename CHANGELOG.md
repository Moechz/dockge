# Changelog

All notable user-visible changes to the Dockge TOS application package.

## 1.5.0-1 — 2026-09-27

### Added
- First release. Dockge (`louislam/dockge:1.5.0`, Docker Hub official image, unmodified)
  packaged as a TOS 7 Docker application, published on port 8004 and running
  as a non-root user (1000:1000).
- Persistent data is kept under `/Volume1/DockerAppData/dockgedocker/`.

### Notes
- This application is a Docker control panel, so it mounts
  `/var/run/docker.sock` - a deliberate, documented exception rather than an
  oversight: without the socket every action in the UI fails. It still runs as
  an unprivileged user (`1000:1000`), adds no capability, uses no privileged
  mode and no host networking, and mounts only the socket file itself. The
  socket is mounted in YAML long form so that the platform installer does not
  chown the host socket to the application user.
- Persistent data: `/Volume1/DockerAppData/dockgedocker/app_data` (settings,
  users, the SQLite database) and `/Volume1/DockerAppData/dockgedocker/stacks`
  (your compose files). Neither is visible in the TOS file manager - use
  SSH/SFTP or the built-in editor.
