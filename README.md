# Manifest — Self-Hosted AI Gateway Dashboard

This repository deploys [Manifest](https://manifest.build) via Docker Compose with a Postgres database. It is **infrastructure only** — the application source code lives in the upstream `manifestdotbuild/manifest` container image.

---

## Table of Contents

- [What is Manifest?](#what-is-manifest)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Environment Variables](#environment-variables)
- [Common Commands](#common-commands)
- [Accessing from the LAN](#accessing-from-the-lan)
- [Accessing over Tailscale HTTPS](#accessing-over-tailscale-https)
- [Connecting Local LLM Servers](#connecting-local-llm-servers)
- [Authentication Options](#authentication-options)
- [Email Setup](#email-setup)
- [Security Hardening](#security-hardening)
- [Updating](#updating)
- [Troubleshooting](#troubleshooting)

---

## What is Manifest?

Manifest is a self-hosted AI gateway dashboard. You connect it to LLM providers (OpenAI, Anthropic, Ollama, etc.) and it gives you a unified interface for managing prompts, tracking usage, setting rate limits, and monitoring costs across all your AI workloads.

---

## Quick Start

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (Windows/Mac) or Docker Engine + Compose (Linux)
- Git (optional, for cloning)

### 1. Configure the environment

```bash
cp .env.example .env
```

Edit `.env` and set at minimum:

```bash
# Required — generate with: openssl rand -hex 32
BETTER_AUTH_SECRET=your-64-char-hex-secret-here

# If you want LAN access, set this to your host's IP (see LAN section below)
BETTER_AUTH_URL=http://localhost:2099
```

### 2. Start the stack

```bash
docker compose up -d
```

### 3. Complete setup

Visit `http://localhost:2099` and follow the setup wizard at `/setup` to create the first admin account. **There are no hardcoded credentials.**

---

## Project Structure

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Defines the `manifest` app container and `postgres` database |
| `.env` | **Local secrets and config** — never commit this (it's in `.gitignore`) |
| `.env.example` | Template showing all available options |
| `CLAUDE.md` | Project-specific guidance for Claude Code |
| `.gitignore` | Excludes `.env` and `.claude/` from version control |

### Architecture

- **manifest** — Node.js app container, port `${PORT:-2099}`
- **postgres** — Postgres 16 Alpine for persistent storage
- **Networks** — `internal` (container-to-container) and `frontend` (app ingress)
- **Volume** — `manifest_pgdata` (explicitly pinned name to survive project renames)

The app container runs read-only with a `/tmp` tmpfs, drops all capabilities, and caps logs at 50 MB.

---

## Environment Variables

### Required

| Variable | Description |
|----------|-------------|
| `BETTER_AUTH_SECRET` | Session signing secret. **Generate with `openssl rand -hex 32`.** |

### Recommended

| Variable | Description |
|----------|-------------|
| `MANIFEST_ENCRYPTION_KEY` | Separate at-rest encryption key for stored LLM credentials. Falls back to `BETTER_AUTH_SECRET` if omitted. |
| `BETTER_AUTH_URL` | The base URL the app uses when generating auth callback links, email links, etc. Defaults to `http://localhost:${PORT}`. **This is only for URL generation — it does not enable HTTPS or change what port the container listens on.** |

### Port

| Variable | Description |
|----------|-------------|
| `PORT` | Host port the app binds to. Default: `2099`. Existing installs can set `3001` to stay on the legacy port. |

### Database

| Variable | Description |
|----------|-------------|
| `POSTGRES_PASSWORD` | Postgres password. Default: `manifest` |
| `DATABASE_URL` | Connection string. Must match `POSTGRES_PASSWORD`. Percent-encode special chars (`@` → `%40`, `:` → `%3A`, `/` → `%2F`). |

### LLM Connectivity

| Variable | Description |
|----------|-------------|
| `OLLAMA_HOST` | URL for a local Ollama instance. Default: `http://host.docker.internal:11434` |

The container reaches host LLM servers (Ollama, LM Studio, etc.) via `host.docker.internal:<port>`. Point Manifest at `http://host.docker.internal:<port>/v1` in the dashboard. The self-hosted image auto-allows private and HTTP URLs.

### Email (Optional)

Supports `resend`, `mailgun`, or `sendgrid`:

```bash
EMAIL_PROVIDER=resend
EMAIL_API_KEY=re_your_api_key_here
EMAIL_FROM=noreply@yourdomain.com
```

Without email configured:
- Signup verification is waived (users are created unverified-but-usable)
- Password reset silently no-ops (reset via database admin if needed)
- Threshold alerts still work if configured per-user in the dashboard

### OAuth (Optional)

Providers activate automatically when **both** `CLIENT_ID` and `CLIENT_SECRET` are set:

```bash
GOOGLE_CLIENT_ID=your-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-secret

# GITHUB_CLIENT_ID=
# GITHUB_CLIENT_SECRET=
# DISCORD_CLIENT_ID=
# DISCORD_CLIENT_SECRET=
```

**Important:** Google OAuth requires a **publicly resolvable HTTPS domain**. It does **not** work with `localhost`, `127.0.0.1`, or any private IP like `192.168.x.x`. See the [Authentication Options](#authentication-options) section for details and workarounds.

### Proxy Timeout (Optional)

| Variable | Description |
|----------|-------------|
| `PROVIDER_TIMEOUT_MS` | How long Manifest waits for a response from an upstream LLM provider before aborting. **Default: `180000` (3 minutes).** Increase this if you use slow local models (Ollama, LM Studio) or see timeout errors in the container logs. |

Example values:

```bash
# 5 minutes
PROVIDER_TIMEOUT_MS=300000

# 10 minutes
PROVIDER_TIMEOUT_MS=600000

# 30 minutes
PROVIDER_TIMEOUT_MS=1800000
```

After changing, recreate the container:

```bash
docker compose up -d
```

### Other

| Variable | Description |
|----------|-------------|
| `MANIFEST_MODE` | Defaults to `selfhosted`. Forces self-hosted semantics (no cloud-mode SSRF restrictions). |
| `MANIFEST_TELEMETRY_DISABLED` | Set to `1` to opt out of anonymous usage telemetry. |

---

## Common Commands

```bash
# Start the stack
docker compose up -d

# Stop the stack
docker compose down

# View app logs
docker compose logs -f manifest

# Update to the latest image
docker compose pull && docker compose up -d

# Restart app only
docker compose restart manifest

# Check health status
docker compose ps

# Shell into app container
docker compose exec manifest sh

# Shell into database
docker compose exec postgres psql -U manifest -d manifest
```

---

## Accessing from the LAN

By default, the Docker Compose file binds the app port to **all interfaces**, meaning other devices on your network can reach it once your firewall allows it.

### 1. Find your host's LAN IP

**Windows (PowerShell):**
```powershell
(Get-NetIPAddress -AddressFamily IPv4 | Where-Object { $_.InterfaceAlias -notlike "*Loopback*" -and $_.IPAddress -notlike "169.254.*" }).IPAddress
```

**Windows (Git Bash):**
```bash
ipconfig | grep 'IPv4'
```

**Linux/macOS:**
```bash
ipconfig getifaddr en0   # macOS, Wi-Fi
hostname -I              # Linux
```

Your LAN IP will look like `192.168.x.x` or `10.x.x.x`. Ignore `127.0.0.1` and Docker/WSL virtual interfaces like `172.x.x.x`.

### 2. Update `.env`

```bash
BETTER_AUTH_URL=http://192.168.1.100:2099   # <-- your actual LAN IP
```

> `BETTER_AUTH_URL` only tells Manifest what URL to use in links and redirects. It does **not** change the port the container listens on or enable HTTPS. If you set `https://`, you must also have a reverse proxy terminating TLS in front of the container.

### 3. Recreate the container

```bash
docker compose up -d
```

### 4. Open the Windows Firewall

Other devices cannot connect unless the firewall allows inbound traffic on port `2099`:

```powershell
New-NetFirewallRule -DisplayName "Manifest" -Direction Inbound -LocalPort 2099 -Protocol TCP -Action Allow
```

### 5. Access from other devices

From any device on the same network, open:

```
http://192.168.1.100:2099
```

---

## Accessing over Tailscale HTTPS

The Manifest container serves plain HTTP on port `2099`. To get a stable HTTPS URL for Tailnet access and OAuth callbacks, put Tailscale in front of it.

### What you need

- Tailscale installed and logged in on the Windows host.
- Admin access to your Tailscale policy (or ask your Tailscale admin).
- Know your Tailnet domain: `tailXXXX.ts.net` (found at [login.tailscale.com](https://login.tailscale.com)).

### Step 1: Decide which Tailscale method to use

| Method | URL looks like | When to use |
|--------|---------------|-------------|
| **Tailscale Service** (recommended) | `https://manifest.tailXXXX.ts.net` | You want a dedicated service domain that can move between machines. |
| **Per-device Tailscale Serve** | `https://your-pc.tailXXXX.ts.net` | You want the fastest one-command setup. |

### Step 2: Set the URL in `.env`

After you know the URL Tailscale will give you, set it exactly:

```bash
BETTER_AUTH_URL=https://manifest.tailXXXX.ts.net
```

Replace `tailXXXX.ts.net` with your actual Tailnet domain. **Do not include `:2099`** — Tailscale handles HTTPS on standard port `443`.

Recreate the container:

```bash
docker compose up -d
```

### Method A: Tailscale Service (recommended)

Tailscale Services require the host machine to be a **tagged node**. If your machine is already tagged, skip to **Register the service**.

#### A1. Create a tag in your Tailscale policy

Go to [login.tailscale.com/admin/acls](https://login.tailscale.com/admin/acls) and add a `tagOwners` entry:

```json
{
  "tagOwners": {
    "tag:manifest-host": ["your-email@example.com"]
  }
}
```

Replace `your-email@example.com` with your Tailscale login email. Save the policy. You must be a **Tailscale admin** or have **Network admin** role to edit the policy.

#### A2. Tag your Windows machine

Open PowerShell as **administrator** and run:

```powershell
tailscale up --advertise-tags=tag:manifest-host --force-reauth
```

Follow the re-authentication link. After this, the machine is owned by the tag, not your user. You can still manage it through the admin console.

> **Note:** If you use an auth key to authenticate, you cannot remove tags with `--advertise-tags`. Generate a new auth key with the desired tags instead.

#### A3. Register the service

```powershell
tailscale serve --service=svc:manifest http://localhost:2099
```

Expected output:

```text
This machine is configured as a service proxy for svc:manifest, but approval from an admin is required. Once approved, it will be available in your Tailnet as:

https://manifest.tailXXXX.ts.net/
|-- proxy http://localhost:2099
```

#### A4. Verify the service endpoint

In the Tailscale admin console, the service endpoint **must** be HTTPS on port `443`, not raw TCP on port `2099`.

If the service endpoint is:

```text
tcp:2099
```

that is wrong for a web app. It should be:

```text
443
```

This determines whether other devices access it at `https://manifest.tailXXXX.ts.net` (correct) or raw TCP at `manifest.tailXXXX.ts.net:2099` (wrong for OAuth).

#### A5. Approve the service host

An admin must approve the service host in the Tailscale admin console before traffic routes to this machine.

#### A5. Google OAuth callback URL

Register this in Google Cloud Console:

```text
https://manifest.tailXXXX.ts.net/api/auth/callback/google
```

#### Disable or remove the service

```powershell
# Disable (keeps config)
tailscale serve --service=svc:manifest --https=443 off

# Remove config entirely
tailscale serve clear svc:manifest
```

---

### Method B: Per-device Tailscale Serve

If you do not need a dedicated service domain, proxy directly through the machine name:

```powershell
tailscale serve --bg --https=443 http://localhost:2099
```

Then set:

```bash
BETTER_AUTH_URL=https://your-machine.tailXXXX.ts.net
```

Google OAuth callback:

```text
https://your-machine.tailXXXX.ts.net/api/auth/callback/google
```

To stop:

```powershell
tailscale serve --bg off
```


---

## Connecting Local LLM Servers

Manifest can connect to LLM servers running on the same machine or elsewhere on your network.

### Ollama (same host)

1. Start Ollama and ensure it binds to all interfaces (or `host.docker.internal` is reachable):
   ```bash
   ollama serve
   ```
2. In the Manifest dashboard, add a provider with URL:
   ```
   http://host.docker.internal:11434/v1
   ```

### LM Studio (same host)

1. In LM Studio, start the local server and enable CORS.
2. In Manifest, use:
   ```
   http://host.docker.internal:1234/v1   # default LM Studio port
   ```

### LLM server on another LAN device

Use the actual LAN IP of that device:
```
http://192.168.1.50:11434/v1
```

---

## Authentication Options

### 1. Local Username / Password (Default)

Works out of the box. The setup wizard at `/setup` creates the first admin. Subsequent users can sign up via the login page.

- **Without email configured:** Signup verification is waived. Users are immediately usable.
- **With email configured:** Verification emails are sent; users must verify before logging in.

### 2. Google OAuth

Requires a public HTTPS domain. Google's OAuth policy **rejects all private IP addresses**, including:

- `localhost`
- `127.0.0.1`
- `192.168.x.x`
- `10.x.x.x`
- `172.16-31.x.x`

**To use Google OAuth, you need a publicly reachable HTTPS endpoint.** The Manifest container serves plain HTTP only, so you must put a reverse proxy in front of it.

**Examples:**

- **Tailscale Service (recommended for this repo)** — Run on the Windows host:
  ```powershell
  tailscale serve --service=svc:manifest http://localhost:2099
  # Then approve the service host in Tailscale admin and set:
  # BETTER_AUTH_URL=https://manifest.tailXXXX.ts.net
  ```
  Notice: no port in the URL. Tailscale handles HTTPS on standard port 443 and proxies to local HTTP on 2099.

- **Per-device Tailscale Serve** — Run on the Windows host:
  ```powershell
  tailscale serve --bg --https=443 http://localhost:2099
  # Then set BETTER_AUTH_URL=https://your-machine.tailXXXX.ts.net
  ```

- **ngrok (temporary testing)** —
  ```bash
  ngrok http 2099
  # Then set BETTER_AUTH_URL=https://your-tunnel.ngrok.io
  ```

- **Cloudflare Tunnel or a real reverse proxy** — Point a domain at your machine and terminate TLS there.

**Callback URL format:**
```
${BETTER_AUTH_URL}/api/auth/callback/google
```

### 3. GitHub / Discord OAuth

Same callback pattern. These providers may be more permissive about localhost for testing, but production should still use a real domain.

---

## Email Setup

Email is required only if you want:
- Signup verification emails
- Password reset functionality
- Server-wide threshold alert notifications

**Recommended for self-hosting: Resend**

1. Sign up at [resend.com](https://resend.com)
2. Create an API key
3. Add to `.env`:
   ```bash
   EMAIL_PROVIDER=resend
   EMAIL_API_KEY=re_your_api_key_here
   EMAIL_FROM=noreply@yourdomain.com
   ```
4. Recreate the container: `docker compose up -d`

---

## Security Hardening

Before exposing this instance beyond your local network:

1. **Set a real `BETTER_AUTH_SECRET`** — Generate with `openssl rand -hex 32`.
2. **Set a separate `MANIFEST_ENCRYPTION_KEY`** — So a session leak doesn't also decrypt stored LLM credentials.
3. **Harden the database password** — Set **both** `POSTGRES_PASSWORD` and `DATABASE_URL` with matching values. Percent-encode special characters.
4. **Use a strong admin password** — The setup wizard enforces ≥ 8 characters; use more.
5. **Enable HTTPS** — Use a reverse proxy (Nginx, Caddy, Traefik, or Cloudflare) for TLS termination.
6. **If using email verification** — Set `BETTER_AUTH_URL` to a reachable public URL so verification links resolve correctly.
7. **Review OAuth secrets** — Ensure `.env` is in `.gitignore` and never committed.

---

## Updating

Always pull the latest image and recreate the container:

```bash
docker compose pull manifest && docker compose up -d manifest
```

Database migrations run automatically on startup. A 90-second health-check start period prevents Docker from marking the container unhealthy during cold-pull + migration + cache-warmup.

---

## Troubleshooting

### Container shows as "unhealthy"

The health check needs up to 90 seconds on first boot (image pull + migrations + pricing cache warmup). This is normal. Check logs:

```bash
docker compose logs -f manifest
```

### Cannot access from another device on the network

1. Verify `BETTER_AUTH_URL` uses the host's LAN IP, not `localhost`.
2. Check Windows Firewall has an inbound rule for port `2099`.
3. Verify both devices are on the same network/subnet.
4. Test with `curl http://<host-ip>:2099` from the other device.

### Google OAuth fails with "redirect_uri_mismatch"

Your `BETTER_AUTH_URL` is likely using a private IP (`192.168.x.x`). Google requires a public HTTPS domain. See [Authentication Options](#authentication-options).

### `BETTER_AUTH_URL=https://` is not working

`BETTER_AUTH_URL` is used only for generating links. It does **not** make the container serve HTTPS. The container always serves plain HTTP on the configured port.

| Scenario | Correct `BETTER_AUTH_URL` |
|----------|---------------------------|
| Plain LAN / Tailnet access | `http://192.168.1.100:2099` |
| Behind Tailscale Service | `https://manifest.tailXXXX.ts.net` (no port) |
| Behind per-device Tailscale Serve | `https://your-machine.tailXXXX.ts.net` (no port) |
| Behind ngrok / reverse proxy | `https://your-public-domain.com` |

If you set `https://` without a reverse proxy in front, auth links will fail.

### Tailscale Service troubleshooting

If you see this error:

```text
service hosts must be tagged nodes
```

You skipped **A1–A2**. Go back to [Method A: Tailscale Service](#method-a-tailscale-service-recommended) and create the ACL tag first, then tag your machine, then rerun the `tailscale serve --service=svc:manifest ...` command.

If you see this error:

```text
must be a URL starting with one of the supported schemes: [http https https+insecure unix]
```

use `http://localhost:2099`, not `tcp://localhost:2099`:

```powershell
tailscale serve --service=svc:manifest http://localhost:2099
```

### `ProxyController` timeout errors

If your container logs show:

```text
[ProxyController] Proxy error: The operation was aborted due to timeout
```

the LLM provider request hit the default 3-minute timeout. This is common with slow local models (Ollama, LM Studio). Increase `PROVIDER_TIMEOUT_MS` in `.env`:

```bash
PROVIDER_TIMEOUT_MS=600000   # 10 minutes
```

Then recreate the container:

```bash
docker compose up -d
```

### Database connection errors

Ensure `POSTGRES_PASSWORD` and the password in `DATABASE_URL` match exactly. Special characters in the URL must be percent-encoded.

### "BETTER_AUTH_SECRET must be set in .env"

You haven't set `BETTER_AUTH_SECRET` in your `.env` file. Generate one:

```bash
openssl rand -hex 32
```

---

## Resources

- **Manifest documentation:** https://manifest.build/docs
- **Telemetry details:** https://manifest.build/telemetry
- **Docker Compose reference:** https://docs.docker.com/compose/
- **ngrok (for temporary public URLs):** https://ngrok.com
- **Cloudflare Tunnel:** https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/
