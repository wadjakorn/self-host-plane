# Plane (self-hosted) + MCP server

A local [Plane](https://plane.so) project-management instance running via Docker Compose, plus
Plane's official MCP server packaged as its own Compose service so AI agents (Claude Code, etc.)
can read and manage projects, work items, cycles, and labels.

```
plane.so/
├── plane-selfhost/        # Plane app stack (Community edition)
│   ├── setup.sh           # official installer/manager (download, start, stop, upgrade, backup)
│   └── plane-app/
│       ├── docker-compose.yaml
│       └── plane.env       # Plane config (ports, URLs, secrets)
└── plane-mcp/             # Plane MCP server (HTTP transport)
    ├── docker-compose.yml
    ├── Dockerfile
    └── .env                # MCP config (instance URL, port)
```

## Prerequisites

- A Docker engine. This setup was built on **Colima** (`colima start`) with the Docker CLI.
- `docker-compose` (standalone v2) on `PATH`. *Note:* on this machine the `docker compose`
  plugin subcommand is not wired up — use the standalone `docker-compose` binary.
- For local agent registration: [`uv`](https://docs.astral.sh/uv/) and the Claude Code CLI
  (only needed if you re-register the MCP server with Claude).

## 1. Run Plane

```bash
cd plane-selfhost
./setup.sh start
```

`setup.sh` is the official manager. It is menu-driven, but also takes a verb:

| Command              | Action                          |
| -------------------- | ------------------------------- |
| `./setup.sh install` | Download compose + pull images  |
| `./setup.sh start`   | Start all services              |
| `./setup.sh stop`    | Stop all services               |
| `./setup.sh restart` | Restart                         |
| `./setup.sh upgrade` | Upgrade to latest release       |
| `./setup.sh logs api`| Tail one service's logs         |
| `./setup.sh backup`  | Back up DB, MinIO, RabbitMQ, Redis |

### Access

| From                 | URL                       |
| -------------------- | ------------------------- |
| This host            | http://localhost          |
| LAN / other devices  | http://192.168.1.37       |
| Tailscale            | http://100.76.154.32      |
| Instance admin       | http://localhost/god-mode |

First visit → create an account. The app login (`/`) and the instance-admin panel
(`/god-mode`) are **separate** account systems.

### Configuration

Edit `plane-selfhost/plane-app/plane.env`, then `./setup.sh restart`. Key vars:

- `APP_DOMAIN` — drives `WEB_URL`; set to the host/IP you reach Plane on (`192.168.1.37`).
- `LISTEN_HTTP_PORT` — published HTTP port (default `80`).
- `CORS_ALLOWED_ORIGINS` — comma-separated allowed origins.

Backup of the original env: `plane-app/plane.env.orig`.

## 2. Run the MCP server

```bash
cd plane-mcp
docker-compose up -d --build
```

This builds a small image (`pip install plane-mcp-server`) and runs it in **HTTP mode**,
listening on port **8211**. It reaches the Plane API over the Plane stack's internal Docker
network (`http://proxy`), so it does not depend on the host IP.

| Command                      | Action                  |
| ---------------------------- | ----------------------- |
| `docker-compose up -d --build` | Start / rebuild       |
| `docker-compose down`        | Stop                    |
| `docker-compose logs -f`     | Logs                    |
| `docker-compose restart`     | Restart                 |

Configure via `plane-mcp/.env`:

- `PLANE_BASE_URL` — Plane API base as seen from the container (default `http://proxy`).
- `MCP_PORT` — host port to publish (default `8211`).
- The `PLANE_OAUTH_PROVIDER_*` values are dummies required only so HTTP mode boots; the
  endpoint actually used (`/http/api-key/mcp`) authenticates per request via headers. Leave them.

### Endpoint

```
http://localhost:8211/http/api-key/mcp
```

Auth is per request, via headers:

- `Authorization: Bearer <PLANE_API_KEY>`
- `X-Workspace-slug: <workspace-slug>`

Generate a token in Plane: **Workspace Settings → API Tokens → Add** (shown once, copy it).
The workspace slug is the segment in your URL: `http://192.168.1.37/<slug>/...`.

## 3. Connect Claude Code

Registered globally (user scope, all projects):

```bash
claude mcp add --transport http plane http://localhost:8211/http/api-key/mcp -s user \
  --header "Authorization: Bearer <PLANE_API_KEY>" \
  --header "X-Workspace-slug: <slug>"

claude mcp get plane      # should report: Status: ✓ Connected
claude mcp remove plane -s user   # to undo
```

Your credentials live in the Claude config (`~/.claude.json`), **not** in the Compose files —
so the MCP server image/compose is credential-less and safe to commit or share.

New Claude Code sessions expose `mcp__plane__*` tools (e.g. `list_projects`).

## Troubleshooting

- **Can't log in after registering** — the app login (`/`) and `/god-mode` are different account
  systems. A "Admin user does not exist" message means you're on `/god-mode` but no instance
  admin has been created. Also double-check for typos in the email you registered with.
- **MCP `Connection failed`** — confirm the container is up (`docker ps | grep plane-mcp`) and
  the Plane stack is running; the MCP container must share the `plane-app_default` network.
- **Network name mismatch** — `docker network ls | grep plane`; update the `name:` under
  `networks:` in `plane-mcp/docker-compose.yml` if it differs.
- **Changed host IP** — update `APP_DOMAIN`/`CORS_ALLOWED_ORIGINS` in `plane.env` and restart.

See `AGENTS.md` for the agent-oriented operational reference.
