---
description: Local development hosting — portless (.localhost URLs) + optional Traefik for container routing
mode: subagent
tools:
  read: true
  write: true
  edit: true
  bash: true
  glob: true
  grep: true
  webfetch: false
---

# Local Hosting

<!-- AI-CONTEXT-START -->

## Quick Reference

- **Primary**: [portless](https://github.com/vercel-labs/portless) — stable `.localhost` URLs, zero-config, worktree-aware
- **Database**: `localdev-helper.sh db` — shared Postgres container management
- **Container routing** (when needed): Traefik for Docker container mesh and trust boundaries
- **Legacy**: `localdev-helper.sh` — dnsmasq + Traefik + mkcert (see [Legacy localdev Stack](#legacy-localdev-stack) below)

**Standard workflow — new project:**

```bash
# Install (one-time)
npm install -g portless

# Run your app (auto-starts proxy, auto-infers name from package.json/git)
portless run next dev
# -> http://myapp.localhost:1355

# Or specify a name explicitly
portless myapp next dev
# -> http://myapp.localhost:1355

# HTTPS (one-time trust, then automatic)
portless trust
portless run next dev
# -> https://myapp.localhost:1355
```

**In package.json** (recommended — works for all team members and AI agents):

```json
{
  "scripts": {
    "dev": "portless run next dev"
  }
}
```

**Why portless over raw `localhost:PORT`?**

| Problem | portless solution |
|---------|-------------------|
| Port conflicts | Automatic port assignment (4000-4999) via `PORT` env var |
| Memorising ports | `myapp.localhost` instead of `localhost:3847` |
| Wrong app on refresh | Named URLs are stable — stop one app, start another, URLs don't collide |
| Agents guess wrong port | `PORTLESS_URL` env var for programmatic discovery |
| Cookie/storage clashes | Each app gets its own origin (`myapp.localhost` vs `api.localhost`) |
| Worktree confusion | Auto-detects worktrees: `fix-ui.myapp.localhost` (see below) |
| Hardcoded ports in config | `PORTLESS_URL` replaces hardcoded `localhost:3000` in CORS/OAuth/env |

<!-- AI-CONTEXT-END -->

## portless

### How it works

```text
Browser request: http://myapp.localhost:1355
        |
        v
  portless proxy (port 1355, auto-started)
    routes by hostname to the correct app port
        |
        v
  Your app on auto-assigned port (4000-4999)
    PORT and HOST env vars set automatically
```

`.localhost` resolves to `127.0.0.1` natively on all modern browsers and OSes (RFC 6761). No DNS configuration, no `/etc/hosts`, no dnsmasq, no resolver files.

### Worktree support (native)

portless auto-detects git worktrees. In a linked worktree, the branch name is prepended as a subdomain:

```bash
# Main worktree (main/master) — no prefix
portless run next dev
# -> http://myapp.localhost:1355

# Linked worktree on branch "fix-ui"
portless run next dev
# -> http://fix-ui.myapp.localhost:1355

# Branch "feature/auth" — uses last segment
portless run next dev
# -> http://auth.myapp.localhost:1355
```

Put `portless run` in `package.json` once — it works in every worktree automatically.

### Subdomains and monorepos

```bash
# API service
portless api.myapp pnpm start
# -> http://api.myapp.localhost:1355

# Docs site
portless docs.myapp next dev
# -> http://docs.myapp.localhost:1355

# Wildcard subdomains (no registration needed)
# tenant1.myapp.localhost:1355 -> myapp
# tenant2.myapp.localhost:1355 -> myapp
```

### HTTPS

```bash
# One-time: generate local CA and trust it
portless trust

# All subsequent runs use HTTPS automatically
portless run next dev
# -> https://myapp.localhost:1355
```

Uses auto-generated local CA with per-hostname TLS certificates (SHA-256). HTTP/2 support included.

### Alias (non-portless services)

Route to services not spawned by portless (e.g., Docker containers):

```bash
# Register a persistent alias
portless alias mydb localhost:5432

# Access via
# http://mydb.localhost:1355
```

### CLI reference

```bash
portless run <command>              # Auto-infer name, run command
portless <name> <command>           # Explicit name, run command
portless alias <name> <host:port>   # Persistent route to external service
portless proxy start                # Start proxy explicitly
portless proxy stop                 # Stop proxy
portless trust                      # Generate and trust local CA (HTTPS)
portless routes                     # List active routes
portless --help                     # Full help
```

### Framework auto-injection

portless sets `PORT` and `HOST` env vars. Most frameworks respect these automatically. For frameworks that ignore `PORT`, portless auto-injects the correct `--port` and `--host` flags:

| Framework | Auto-injected |
|-----------|---------------|
| Next.js | `PORT` env var (native) |
| Express | `PORT` env var (native) |
| Nuxt | `PORT` env var (native) |
| Vite | `--port` and `--host` flags |
| Astro | `--port` and `--host` flags |
| React Router | `--port` flag |
| Angular | `--port` flag |
| Expo | `--port` and `--host` flags |
| React Native | `--port` flag |

### Agent integration

Child processes receive `PORTLESS_URL` containing the public `.localhost` URL:

```bash
# In your app code or test scripts
const url = process.env.PORTLESS_URL; // "http://myapp.localhost:1355"
```

Use this for browser testing agents, Playwright scripts, and any tool that needs to discover the app URL programmatically.

### Loop detection

portless detects forwarding loops (e.g., Vite proxying back through portless) using the `X-Portless-Hops` header. Returns `508 Loop Detected` with an explanation of the fix.

## Architecture — Layered Approach

portless handles developer-facing routing (stable URLs for humans and agents). For projects that also need container isolation or trust boundaries, Traefik operates at a separate infrastructure layer:

```text
Developer browser / AI agent
    |
    v
portless proxy (port 1355)
  routes myapp.localhost → localhost:{auto-port}
    |
    v
Your app on localhost:{port}
    |
    v (only if container isolation is needed)
Docker network / Traefik
  trust boundaries, container-to-container routing
    |
    ├── private-worker container
    └── public-worker container
```

Most projects only need the portless layer. The Traefik layer is for Docker container mesh routing and future trust-boundary enforcement (public vs private worker isolation).

## Database Management

Shared Postgres via Docker, independent of the routing layer. Works with both portless and the legacy localdev stack.

### db

Shared Postgres database management via a `local-postgres` Docker container.

```bash
localdev-helper.sh db start              # Ensure container is running
localdev-helper.sh db stop               # Stop container
localdev-helper.sh db create <dbname>    # Create database
localdev-helper.sh db drop <dbname> -f   # Drop database (requires --force)
localdev-helper.sh db list               # List all databases with URLs
localdev-helper.sh db url <dbname>       # Output connection string
localdev-helper.sh db status             # Container and database status
```

Default configuration (override via environment variables):

| Variable | Default | Purpose |
|----------|---------|---------|
| `LOCALDEV_PG_IMAGE` | `postgres:17-alpine` | Docker image |
| `LOCALDEV_PG_PORT` | `5432` | Host port |
| `LOCALDEV_PG_USER` | `postgres` | Postgres user |
| `LOCALDEV_PG_PASSWORD` | `localdev` | Postgres password |
| `LOCALDEV_PG_DATA` | `~/.local-dev-proxy/pgdata` | Data directory |

Database names with hyphens are auto-converted to underscores for Postgres compatibility (e.g., `myapp-feature-xyz` becomes `myapp_feature_xyz`).

Connection string format: `postgresql://postgres:localdev@localhost:5432/{dbname}`

### Branch-Isolated Databases

Create separate databases per feature branch to avoid schema conflicts:

```bash
# Create branch database
localdev db create myapp-feature-auth
# -> postgresql://postgres:localdev@localhost:5432/myapp_feature_auth

# When branch is merged, clean up
localdev db drop myapp-feature-auth --force
```


## Stack-Specific Guidance

portless auto-injects `PORT` and `HOST` env vars and framework-specific flags. Most frameworks work with zero configuration via `portless run`. Notes for specific stacks:

### Next.js

```bash
# Just works — Next.js respects PORT env var
portless run next dev
# -> http://myapp.localhost:1355
```

**`allowedDevOrigins` (Next.js 15+):** Add your `.localhost` domain to allow cross-origin dev requests:

```typescript
const config: NextConfig = {
  allowedDevOrigins: [
    "myapp.localhost",
    "myapp.localhost:1355",
  ],
};
```

**Stale lock file (Next.js 16+):** Next.js creates `.next/dev/lock` on start. Ungraceful shutdowns leave it behind. Add cleanup to your dev script:

```json
{
  "scripts": {
    "dev": "rm -f .next/dev/lock && portless run next dev"
  }
}
```

### Vite (Vue, React, Svelte)

```bash
# portless auto-injects --port and --host flags
portless run npx vite
```

No manual `--port` configuration needed — portless handles it.

### Turborepo / Monorepo

```bash
# From monorepo root — portless wraps the turbo command
portless run pnpm dev

# Or target a specific app with subdomains
portless api.myapp pnpm --filter api dev
portless web.myapp pnpm --filter web dev
```

For monorepos with `with-env` / `dotenv`, use `PORTLESS_URL` in your root `.env.local`:

```bash
URL="${PORTLESS_URL:-http://localhost:3000}"
NEXT_PUBLIC_URL="${PORTLESS_URL:-http://localhost:3000}"
```

### Ruby on Rails / Django / Go / PHP

```bash
# All respect PORT env var — just works
portless run rails server
portless run python manage.py runserver
portless run go run .
portless run php artisan serve
```

### Docker Compose Projects

For services running in Docker, use `portless alias` to route to their published ports:

```bash
# Start your Docker services
docker compose up -d

# Route portless to the published port
portless alias myapp localhost:3000
# -> http://myapp.localhost:1355 routes to Docker container on port 3000
```

## Troubleshooting

### Port conflicts

```bash
# Check what's using a port
lsof -i :4123

# portless auto-assigns ports (4000-4999), so conflicts are rare
# Use --app-port to force a specific port if needed
portless run --app-port 4123 next dev
```

### `.localhost` not resolving

`.localhost` should resolve to `127.0.0.1` natively (RFC 6761). If it doesn't:

```bash
# Check resolution
ping myapp.localhost

# If it fails, portless can sync /etc/hosts as a fallback
PORTLESS_SYNC_HOSTS=1 portless proxy start
```

### HTTPS certificate issues

```bash
# Re-run trust to regenerate CA
portless trust

# Check if CA is trusted
portless proxy start  # Will warn if CA is not trusted
```

## Prerequisites

```bash
# Node.js (required)
node --version  # v18+ recommended

# Install portless globally
npm install -g portless

# Optional: HTTPS support (one-time)
portless trust

# Optional: shared Postgres for database management
brew install orbstack  # or Docker Desktop
localdev-helper.sh db start
```

---

## Legacy localdev Stack

> The localdev stack (dnsmasq + Traefik + mkcert) is the previous approach. It still works and is needed for Docker container mesh routing. For new projects, use portless instead.

### Why portless replaced localdev for routing

| Issue | localdev | portless |
|-------|----------|----------|
| `.local` TLD conflicts with macOS mDNS | Requires `/etc/hosts` hacks per app | `.localhost` resolves natively (RFC 6761) |
| Setup complexity | `sudo`, dnsmasq, resolver, Traefik, Docker | `npm install -g portless` |
| Worktree support | Manual `localdev branch` per worktree | Auto-detected, zero config |
| Agent integration | None | `PORTLESS_URL` env var |
| Framework auto-config | None — manual port assignment | Auto-injects `--port`/`--host` flags |

### When to still use localdev

- **Database management**: `localdev-helper.sh db` for shared Postgres (independent of routing)
- **Docker container routing**: Traefik for container-to-container communication via Docker networks
- **LocalWP coexistence**: If you use LocalWP for WordPress development alongside other projects
- **Future container isolation**: Trust boundary enforcement between public/private worker containers

### localdev CLI Reference

```bash
localdev-helper.sh init                  # One-time system setup (dnsmasq, resolver, Traefik)
localdev-helper.sh add <name> [port]     # Register app with cert, route, /etc/hosts
localdev-helper.sh rm <name>             # Remove app and all resources
localdev-helper.sh branch <app> <branch> # Create branch subdomain
localdev-helper.sh db <subcommand>       # Database management (see above)
localdev-helper.sh list                  # Dashboard of all local projects
localdev-helper.sh status                # Infrastructure health check
```

### Legacy localhost-helper.sh

```bash
localhost-helper.sh check-port <port>     # Check port availability
localhost-helper.sh find-port [start]     # Find next available port
localhost-helper.sh list-ports            # List common dev ports in use
localhost-helper.sh kill-port <port>      # Kill process on port
localhost-helper.sh generate-cert <domain> # Generate mkcert cert
localhost-helper.sh start-mcp             # Start LocalWP MCP server
```

### LocalWP Coexistence

LocalWP manages WordPress sites with its own DNS entries in `/etc/hosts`. portless uses `.localhost` which doesn't conflict. If using the legacy localdev stack alongside LocalWP, `localdev add` checks for collisions and rejects conflicting `.local` domains.

**LocalWP sites.json**: `~/Library/Application Support/Local/sites.json`

### Docker / OrbStack Integration

The shared Postgres container and any future Traefik container routing use the `local-dev` Docker network:

```bash
docker network create local-dev
```

OrbStack is preferred over Docker Desktop for lower memory footprint. OrbStack provides its own `.orb.local` domains for containers — these are separate from both portless `.localhost` and legacy localdev `.local` domains.

### localdev Architecture (reference)

```text
Browser request: https://myapp.local
        |
        v
  /etc/hosts (127.0.0.1 myapp.local)
    <- REQUIRED for .local in browsers (mDNS intercepts /etc/resolver)
        |
        v
  Traefik (Docker, ports 80/443/8080)
    reads conf.d/*.yml, terminates TLS via mkcert certs
        |
        v
  http://host.docker.internal:{port}
```

**The .local mDNS problem** (why portless is better): macOS reserves `.local` for mDNS (Bonjour). Browsers send `.local` queries to mDNS before consulting `/etc/resolver/local`, so dnsmasq alone is insufficient — every app needs a manual `/etc/hosts` entry. `.localhost` (RFC 6761) resolves to `127.0.0.1` natively, eliminating this entire problem.

### File Locations (legacy)

| Path | Purpose |
|------|---------|
| `~/.local-dev-proxy/` | Traefik config, port registry, Postgres data |
| `~/.local-dev-proxy/traefik.yml` | Traefik static config |
| `~/.local-dev-proxy/docker-compose.yml` | Traefik Docker Compose |
| `~/.local-dev-proxy/conf.d/` | Per-app Traefik route files |
| `~/.local-dev-proxy/ports.json` | Port registry (apps + branches) |
| `~/.local-dev-proxy/pgdata/` | Shared Postgres data directory |
| `~/.local-ssl-certs/` | mkcert certificate and key files |
| `/etc/resolver/local` | macOS resolver for `.local` domains |
