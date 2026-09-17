# dock-codex

Run OpenAI Codex inside Docker with a host directory mounted as `/workspace`.

## Usage

```sh
npx dock-codex
dock-codex ~/src/my-project
dock-codex . --full-auto
dock-codex . remote-control start
dock-codex . remote-control pair
```

Arguments after the directory are passed to `codex`. Use `.` before Codex
flags when mounting the current directory. Each workspace reuses a named Docker
container, and each invocation runs Codex inside it with `docker exec`.

## Options

- `--image <name>`: Docker image tag. Defaults to
  `dock-codex-(dirname):latest`.
- `--rebuild`: rebuild before running.
- `--docker-file <path>`: Dockerfile for `docker build`. Defaults to
  the packaged `Dockerfile.dock-codex`. Pass this option to use a custom
  Dockerfile.
- `--docker-context <path>`: build context for `docker build`. Defaults to the
  mounted directory.
- `--guest-mount <path>`: repeatable workspace-relative path hidden by a
  guest-only volume. Defaults to `node_modules`.
- `--docker-run <argument>`: repeatable argument passed to `docker run` when the
  container is created. Use `--docker-run=--option=value` when the argument
  starts with `--`.

Examples:

```sh
dock-codex --docker-file ./Dockerfile.dev --docker-context . --rebuild .
dock-codex --guest-mount dist --guest-mount packages/app/node_modules .
dock-codex --docker-run=--network=host .
dock-codex --docker-run=--cap-add=NET_ADMIN --docker-run=--device=/dev/net/tun .
dock-codex --docker-run=--mount=type=bind,src=/host/path,dst=/container/path .
```

Container options apply when Docker creates the container. Use `--rebuild` or
remove the existing workspace container to change these options.

## Authentication

Codex keeps its configuration and login state in `.dock-codex/` in the
mounted project. On first use, sign in interactively. In a headless container,
device-code authentication is the most convenient option:

```sh
dock-codex . login --device-auth
```

For API-key authentication, pass the key only to the login command:

```sh
printenv OPENAI_API_KEY | dock-codex . login --with-api-key
```

Remote Control requires ChatGPT authentication. Start and pair it in the same
persistent workspace container:

```sh
dock-codex . login --device-auth
dock-codex . remote-control start
dock-codex . remote-control pair
```

## Project Setup

Codex is installed by the packaged Dockerfile. Optionally add
`.dock-codex-init.sh` to the project root to provision additional tools in the
project's cached image:

```sh
#!/bin/bash
set -euo pipefail

apt-get update
apt-get install -y --no-install-recommends jq
rm -rf /var/lib/apt/lists/*
```

The packaged Dockerfile starts from `debian:bookworm-slim` and installs CA
certificates, Git, OpenSSH, procps, and the standalone Codex release from
OpenAI's installer. It does not include Node.js. Install Node.js in
`.dock-codex-init.sh` if your project needs it.
When `.dock-codex-init.sh` exists, the Docker build runs it with Bash as root
after installing Codex. `CODEX_VERSION` is available to the script and defaults
to `latest`. Without an init script, the customization step is skipped entirely.

For full control, `Dockerfile.dock-codex` remains available as an advanced
override.

Add `.dock-codex` to both `.gitignore` and `.dockerignore`:

```gitignore
.dock-codex/
```

## Notes

- The image is built on first run.
- Each workspace uses a named `dock-codex-(dirname)` container.
- The container stays alive with `sleep infinity`; Codex runs through
  `docker exec`.
- Containers are not removed automatically. Use `docker rm -f <name>` when a
  workspace container is no longer needed.
- The container runs as the current host UID/GID.
- Codex state is stored in `.dock-codex/` inside the mounted directory.
- Host Codex configuration and OpenAI-related environment variables are not
  forwarded automatically.
