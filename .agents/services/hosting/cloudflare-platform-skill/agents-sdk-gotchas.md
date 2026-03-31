# Agents SDK Gotchas & Best Practices

## Security and auth

- Auth **before** `conn.accept()` — unauthenticated connections can read broadcasts.
- Secrets in env bindings only; never in agent/connection state.
- Don't trust client-controlled headers without validation.

```ts
// ❌ conn.accept(); then check auth later
// ✅ Validate first, accept only on success
async onConnect(conn: Connection, ctx: ConnectionContext) {
  const token = (ctx.request.headers.get("Authorization") ?? "").replace("Bearer ", "");
  if (!token || !await this.validateToken(token)) { conn.close(4001, "Unauthorized"); return; }
  conn.accept();
}
```

## State discipline

- Always `setState()` — direct `this.state` mutation skips sync to clients and persistence.
- Keep state serializable and small; move large data to SQL storage.
- Don't store functions, class instances, or circular references in state.

```ts
// ❌ this.state.count++
// ✅ this.setState({ ...this.state, count: this.state.count + 1 })
```

### Connection state vs agent state

- `conn.setState()` — per-connection metadata (userId, session token); lost on disconnect.
- `this.setState()` — shared agent state; persisted and broadcast to all connections.
- Don't put user-specific data in agent state or shared data in connection state.

## SQL

- Initialize schema in `onStart()` — tables don't exist until created.
- Always parameterize — tagged template literals auto-escape; string interpolation does not.

```ts
// ❌ this.sql`...WHERE id = '${userId}'`  (SQL injection)
// ✅ this.sql`...WHERE id = ${userId}`     (parameterized)
```

## Routing and entry point

- `routeAgentRequest()` in the Worker `fetch` handler is required to route requests to agent DOs. Missing it = "Agent not found" errors.

```ts
// ❌ export default { fetch(req, env) { return new Response("ok"); } }
// ✅ export default { fetch(req, env, ctx) { return routeAgentRequest(req, env) ?? new Response("Not found", { status: 404 }); } }
```

## WebSocket lifecycle

- Call `conn.accept()` promptly — delayed accept causes client-side timeout.
- Handle `onClose`/`onError` for cleanup; don't assume connections persist across hibernation.
- Connection state survives hibernation; in-memory variables do not.

## Scheduling

- 1 alarm per DO — `setAlarm()` overwrites any existing alarm.
- Alarm handlers have a 15-minute wall-clock limit; retries use exponential backoff (max 6 attempts).
- Use `schedule()` for cron-like patterns; use `setAlarm()` for one-shot delayed work.

## AI and performance

- Use AI Gateway for caching, streaming, and rate limiting.
- Wrap model calls in `try/catch` with fallbacks for quota/timeout/provider errors.
- Batch `setState()` calls; reduce write frequency; limit broadcast fan-out with backpressure.

```ts
try { return await this.env.AI.run(model, { prompt }); }
catch { return { error: "Unavailable" }; }
```

## Runtime limits

| Resource | Limit |
|----------|-------|
| CPU | 30s/request (up to 5 min via `limits.cpu_ms`) |
| Memory | 128 MB/instance |
| WebSocket connections | 32,768/DO (practical limit lower) |
| Alarms | 1 per DO |
| SQL storage | Shares DO 10 GB quota |

## Migration

- `new_sqlite_classes` must be set when the class is **first created** — cannot add SQLite to an existing deployed class.
- `deleted_classes` destroys all data permanently; no rollback.
- Test migrations in staging with `--dry-run`.

```toml
[[migrations]]
tag = "v1"
new_sqlite_classes = ["MyAgent"]
```

## Common failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| "Agent not found" | Missing DO binding or `routeAgentRequest()` | Check `wrangler.toml` bindings and Worker entry point |
| State not syncing | Direct `this.state` mutation | Use `setState()` |
| Connect timeout | Delayed `conn.accept()` | Call `conn.accept()` immediately in `onConnect` |
| SQL errors on start | Missing `onStart()` schema init | Create tables in `onStart()` |

## Debugging

```bash
npx wrangler dev          # Local development
npx wrangler tail         # Stream remote logs
```
