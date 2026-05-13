# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **Docker Compose deployment repository** for [Manifest](https://manifest.build) (`manifestdotbuild/manifest:latest`), a self-hosted AI gateway dashboard. It does **not** contain application source code — only the infrastructure to run the pre-built container image with a Postgres database.

## Stack Architecture

- **manifest** — The main application container (Node.js, port `${PORT:-2099}`).
- **postgres** — Postgres 16 Alpine for persistent data.
- **Networks** — `internal` (container-to-container only) and `frontend` (app ingress).
- **Volume** — `manifest_pgdata` (explicitly pinned name to survive project renames).

The app container is run as read-only with a `/tmp` tmpfs, capability drops, and a 50 MB log cap.

## Common Commands

| Task | Command |
|------|---------|
| Start stack | `docker compose up -d` |
| Stop stack | `docker compose down` |
| View logs | `docker compose logs -f manifest` |
| Update image | `docker compose pull && docker compose up -d` |
| Restart app only | `docker compose restart manifest` |
| Check health | `docker compose ps` |
| Shell into app | `docker compose exec manifest sh` |
| Shell into DB | `docker compose exec postgres psql -U manifest -d manifest` |

## Configuration

All runtime configuration lives in `.env` (copied from `.env.example`).

**Required:**
- `BETTER_AUTH_SECRET` — Session signing secret (`openssl rand -hex 32`).

**Recommended:**
- `MANIFEST_ENCRYPTION_KEY` — Separate at-rest encryption key for stored LLM credentials. If omitted, falls back to `BETTER_AUTH_SECRET`.

**Port:**
- Default is `2099`. Existing installs can set `PORT=3001` in `.env` to stay on the legacy port without editing `docker-compose.yml`.

**Database:**
- Default password is `manifest`. To harden, set **both** `POSTGRES_PASSWORD` and `DATABASE_URL` with matching values. Percent-encode special characters in the URL (`@` → `%40`, `:` → `%3A`, `/` → `%2F`).

**LLM Connectivity:**
- The container reaches host LLM servers (Ollama, LM Studio, etc.) via `host.docker.internal:<port>`.
- Point Manifest at `http://host.docker.internal:<port>/v1`. The self-hosted image auto-allows private and HTTP URLs.
- Override with `OLLAMA_HOST` in `.env` if needed.

**Email (optional):**
- Supports `resend`, `mailgun`, or `sendgrid`. Configured server-wide in `.env`; per-user dashboard configs take precedence.
- Without email: signup verification is waived, password reset silently no-ops.

**OAuth (optional):**
- Providers activate automatically when both `CLIENT_ID` and `CLIENT_SECRET` are set.
- Callback URL format: `${BETTER_AUTH_URL}/api/auth/callback/<provider>`.

## First-Time Setup

1. `cp .env.example .env`
2. Generate and set `BETTER_AUTH_SECRET`.
3. `docker compose up -d`
4. Visit `http://localhost:2099` and create the first admin account via the setup wizard at `/setup`. No hardcoded credentials exist.

## Security Notes Before Exposing Beyond localhost

- Set a real `BETTER_AUTH_SECRET`.
- Harden `POSTGRES_PASSWORD` and `DATABASE_URL`.
- Ensure the admin password is strong (≥ 8 characters enforced by wizard).
- If enabling email verification, set `BETTER_AUTH_URL` to a reachable public URL.
- The compose binds the port to all interfaces by default (`${PORT:-2099}:${PORT:-2099}`). If exposing on a LAN, update `BETTER_AUTH_URL` to match the reachable host.

## Updating

Always pull the latest image and recreate the container:

```bash
docker compose pull manifest && docker compose up -d manifest
```

Database migrations run automatically on startup. A 90-second health-check start period prevents Docker from marking the container unhealthy during cold-pull + migration + cache-warmup.
