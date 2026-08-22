# dock-codex

Run OpenAI Codex inside Docker with a host directory mounted as `/workspace`.

## Usage

```sh
npx dock-codex
dock-codex ~/src/my-project
dock-codex . --full-auto
```

Arguments after the directory are passed to `codex`. Use `.` before Codex
flags when mounting the current directory.

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
- `--mount <host:guest>`: repeatable extra host bind mount. Relative host paths
  resolve from the current directory; guest paths must be absolute.

Examples:

```sh
dock-codex --docker-file ./Dockerfile.dev --docker-context . --rebuild .
dock-codex --guest-mount dist --guest-mount packages/app/node_modules .
dock-codex --mount ~/.ssh:/workspace/.ssh --mount ../shared:/shared .
```

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

The packaged Dockerfile starts from `node:24-slim` and installs CA certificates,
Git, OpenSSH, and Codex directly. When `.dock-codex-init.sh` exists, the Docker
build runs it with Bash as root after installing Codex. `CODEX_VERSION` is
available to the script and defaults to `latest`. Without an init script, the
customization step is skipped entirely.

For full control, `Dockerfile.dock-codex` remains available as an advanced
override.

Add `.dock-codex` to both `.gitignore` and `.dockerignore`:

```gitignore
.dock-codex/
```

## Notes

- The image is built on first run.
- The container runs as the current host UID/GID.
- Codex state is stored in `.dock-codex/` inside the mounted directory.
- Host Codex configuration and OpenAI-related environment variables are not
  forwarded automatically.
