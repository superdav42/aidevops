---
description: Telegram bot integration — grammY SDK (TypeScript, MIT), BotFather setup, long polling + webhooks, inline keyboards, commands, reactions, forum topics, files, payments, group privacy mode, access control, runner dispatch, Matterbridge bridging
mode: subagent
tools:
  read: true
  write: false
  edit: false
  bash: true
  glob: false
  grep: false
  webfetch: false
  task: false
---

# Telegram Bot Integration

<!-- AI-CONTEXT-START -->

## Quick Reference

- **Type**: Cloud-based messaging platform — centralized, server-side message storage, phone number required
- **Bot SDK**: [grammY](https://grammy.dev/) (TypeScript-first, MIT, 3.4k stars)
- **npm Package**: `grammy`
- **Bot API Version**: 9.5 (latest supported by grammY)
- **Runtimes**: Node.js, Deno, Bun, browser (web bundle)
- **Bot API Docs**: https://core.telegram.org/bots/api
- **grammY Docs**: https://grammy.dev/
- **grammY Repo**: [github.com/grammyjs/grammY](https://github.com/grammyjs/grammY)
- **Telegram Bot FAQ**: https://core.telegram.org/bots/faq

**Key characteristics**: Telegram is a cloud-based messenger with 900M+ users. Bots are first-class citizens with a rich API (inline keyboards, payments, mini apps, forum topics, reactions). grammY is the recommended TypeScript SDK — written from scratch in TypeScript with middleware architecture, comprehensive plugin ecosystem, and always-current Bot API support.

**When to use Telegram vs other protocols**:

| Criterion | Telegram | SimpleX | Matrix | XMTP |
|-----------|----------|---------|--------|------|
| User identifiers | Phone number + username | None | `@user:server` | Wallet/DID |
| Encryption | Server-side (cloud); optional E2E (Secret Chats, not for bots) | Double ratchet (E2E) | Megolm (optional E2E) | MLS + post-quantum |
| Bot ecosystem | Mature, first-class (Bot API, payments, mini apps) | WebSocket JSON API | `matrix-bot-sdk` | `@xmtp/agent-sdk` |
| Group scalability | 200K members per supergroup | Experimental (1000+) | Production-grade | Growing |
| Best for | Large audience reach, rich bot UX, payments | Maximum privacy | Team collaboration, bridges | Web3/agent messaging |

<!-- AI-CONTEXT-END -->

## Architecture

```text
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ Telegram Client   │     │ Telegram Servers  │     │ Bot Process      │
│ (iOS, Android,    │     │ (cloud, MTProto)  │     │ (Node.js/Deno/   │
│  Desktop, Web)    │     │                  │     │  Bun + grammY)   │
│                  │────▶│ Store messages    │────▶│                  │
│ User sends:      │     │ Route to bot      │     │ 1. Receive update│
│ "/status server" │◀────│ Deliver response  │◀────│ 2. Middleware    │
│                  │     │                  │     │ 3. Dispatch      │
│ Bot response     │     │                  │     │ 4. Reply         │
└──────────────────┘     └──────────────────┘     └──────────────────┘
                                                        │
                                                        ▼
                                                  ┌──────────────────┐
                                                  │ aidevops Runner  │
                                                  │ runner-helper.sh │
                                                  │ → AI session     │
                                                  │ → response       │
                                                  └──────────────────┘
```

**Update delivery modes**:

- **Long polling** (`bot.start()`): Bot asks Telegram for updates; Telegram holds connection open until updates arrive. Simpler — no domain or SSL needed. Best for VPS, local dev, always-on servers.
- **Webhooks** (`webhookCallback(bot, "<framework>")`): Bot provides a public HTTPS URL; Telegram pushes updates to it. Best for serverless (Cloudflare Workers, Vercel, Deno Deploy, AWS Lambda).

## Bot Setup

### BotFather Registration

1. Open Telegram, talk to [@BotFather](https://t.me/BotFather)
2. Send `/newbot`
3. Provide a **display name** and **username** (must end in `bot`, e.g., `MyDevOpsBot`)
4. BotFather returns an authentication token: `110201543:AAHdqTcvCH1vGWJxfSeofSAs0K5PALDsaw`
5. Store token securely: `aidevops secret set TELEGRAM_BOT_TOKEN`

**Other BotFather commands**: `/setname`, `/setdescription`, `/setabouttext`, `/setuserpic`, `/setcommands`, `/setprivacy`, `/setinline`, `/setjoingroups`.

### Token Management

- Token format: `<bot_id>:<random_string>`
- Store via gopass or `~/.config/aidevops/credentials.sh` (600 permissions)
- Regenerate via BotFather `/token` if compromised
- Never commit tokens to source control

### Minimal Bot (Node.js)

```bash
npm init -y && npm i grammy
```

```typescript
import { Bot } from "grammy";

const bot = new Bot(process.env.TELEGRAM_BOT_TOKEN!);

bot.command("start", (ctx) => ctx.reply("Welcome! Up and running."));
bot.on("message", (ctx) => ctx.reply("Got another message!"));

bot.start();
```

### Minimal Bot (Deno)

```typescript
import { Bot } from "https://deno.land/x/grammy@v1.41.0/mod.ts";

const bot = new Bot(Deno.env.get("TELEGRAM_BOT_TOKEN")!);

bot.command("start", (ctx) => ctx.reply("Welcome!"));
bot.start();
// Run: deno -IN bot.ts
```

## Middleware Architecture

grammY uses a middleware stack (similar to Koa/Express). Middleware functions process updates top-to-bottom; calling `await next()` passes control downstream.

```typescript
import { Bot, Context, NextFunction } from "grammy";

// Custom middleware: response time logging
async function responseTime(ctx: Context, next: NextFunction): Promise<void> {
  const before = Date.now();
  await next(); // invoke downstream middleware
  const after = Date.now();
  console.log(`Response time: ${after - before} ms`);
}

const bot = new Bot(token);
bot.use(responseTime);
bot.command("ping", (ctx) => ctx.reply("pong"));
bot.start();
```

**Key rules**:

- Always `await next()` — forgetting `await` causes wrong execution order and unhandled rejections
- Not calling `next()` stops propagation — downstream middleware is skipped
- Registration order matters — first registered = first executed
- `Bot` extends `Composer` — `bot.use()`, `bot.on()`, `bot.command()`, `bot.hears()` are all middleware registration

### Composer Pattern

```typescript
import { Composer } from "grammy";

const adminModule = new Composer();
adminModule.command("ban", (ctx) => { /* ... */ });
adminModule.command("unban", (ctx) => { /* ... */ });

bot.use(adminModule); // mount as a unit
```

## Bot API Features

### Commands

```typescript
bot.command("start", (ctx) => ctx.reply("Hello!"));
bot.command("help", (ctx) => ctx.reply("Available commands: /start, /help, /status"));
bot.command("status", (ctx) => ctx.reply("All systems operational."));
```

- Commands are `/keyword` format, up to 32 chars, Latin + numbers + underscores
- Register with BotFather via `/setcommands` for autocomplete
- Command scopes: show different commands to different users/groups via `setMyCommands` with `BotCommandScope`

### Inline Keyboards (Buttons Below Messages)

```typescript
import { InlineKeyboard } from "grammy";

const keyboard = new InlineKeyboard()
  .text("Approve", "approve_123")
  .text("Reject", "reject_123").row()
  .url("View PR", "https://github.com/org/repo/pull/123");

await ctx.reply("PR #123 ready for review:", { reply_markup: keyboard });

// Handle button clicks
bot.callbackQuery("approve_123", async (ctx) => {
  await ctx.answerCallbackQuery({ text: "Approved!" });
  await ctx.editMessageText("PR #123: Approved by " + ctx.from.first_name);
});
```

- Methods: `.text()`, `.url()`, `.switchInline()`, `.switchInlineCurrent()`, `.row()`
- Always call `ctx.answerCallbackQuery()` to dismiss the loading animation

### Custom Keyboards (Replace System Keyboard)

```typescript
import { Keyboard } from "grammy";

const keyboard = new Keyboard()
  .text("Deploy staging").text("Deploy production").row()
  .text("View logs").text("Cancel")
  .resized().oneTime();

await ctx.reply("Choose action:", { reply_markup: keyboard });
```

- Options: `.resized()`, `.oneTime()`, `.persistent()`, `.placeholder()`
- Sends regular text messages when pressed — handle via `bot.hears()`
- Remove with `{ reply_markup: { remove_keyboard: true } }`

### Reactions

```typescript
// Send reactions
bot.command("start", (ctx) => ctx.react("👍"));

// Listen for reaction changes
bot.reaction("🎉", (ctx) => ctx.reply("Celebration!"));
bot.reaction(["👍", "👎"], (ctx) => {
  const { emojiAdded, emojiRemoved } = ctx.reactions();
  // Process reaction changes
});
```

- Requires `allowed_updates: ["message_reaction", "message_reaction_count"]`
- Bot must be admin to receive reaction updates in groups
- Supports emoji, custom emoji, and paid (star) reactions

### File Handling

```typescript
// Receiving files
bot.on("message:document", async (ctx) => {
  const file = await ctx.getFile(); // temporary URL, valid ~1 hour
  console.log(file.file_path);
});

// Sending files
import { InputFile } from "grammy";

await ctx.replyWithPhoto(new InputFile("/path/to/photo.jpg"));           // upload
await ctx.replyWithPhoto("https://example.com/photo.jpg");               // by URL
await ctx.replyWithDocument(new InputFile(buffer, "report.pdf"));        // from buffer
await ctx.replyWithPhoto(existingFileId);                                 // by file_id
```

- Download limit: 20 MB (standard Bot API); up to 2 GB with local Bot API server
- Upload limit: 50 MB (standard); up to 2 GB with local Bot API server
- `InputFile` accepts: file paths, streams, Buffers, async iterators, URLs, Blobs
- Files plugin (`@grammyjs/files`) adds `file.download()` and `file.getUrl()`

### Voice Messages

```typescript
// Receive voice messages
bot.on("message:voice", async (ctx) => {
  const voice = ctx.msg.voice;
  const file = await ctx.getFile();
  // Download and process (e.g., speech-to-text)
});

// Send voice messages
await ctx.replyWithVoice(new InputFile("/path/to/audio.ogg"));
```

- Also: `message:audio`, `message:video_note` for other media types
- Integration with speech-to-text: see `tools/voice/speech-to-speech.md`

### Forum Topics

Telegram supergroups can enable "Topics" (forum mode) for organized discussions:

```typescript
// Create a topic
const topic = await ctx.api.createForumTopic(chatId, "Bug Reports", {
  icon_color: 0xFF0000,
});

// Send to a specific topic
await ctx.api.sendMessage(chatId, "New bug filed", {
  message_thread_id: topic.message_thread_id,
});

// Close/reopen topics
await ctx.api.closeForumTopic(chatId, topic.message_thread_id);
await ctx.api.reopenForumTopic(chatId, topic.message_thread_id);
```

- Methods: `createForumTopic`, `editForumTopic`, `closeForumTopic`, `reopenForumTopic`, `deleteForumTopic`
- Messages in topic-enabled groups include `message_thread_id`

### Payments

```typescript
// Send invoice (Telegram Stars for digital goods)
await ctx.replyWithInvoice(
  "Premium Analysis",                    // title
  "Detailed code review report",         // description
  "premium_analysis_001",                // payload
  "XTR",                                 // currency (Telegram Stars)
  [{ label: "Analysis", amount: 100 }],  // prices
);

// Handle pre-checkout
bot.on("pre_checkout_query", (ctx) => ctx.answerPreCheckoutQuery(true));

// Handle successful payment
bot.on("message:successful_payment", (ctx) => {
  ctx.reply("Payment received! Starting analysis...");
});
```

- **Digital goods**: Must use Telegram Stars (`XTR` currency)
- **Physical goods**: Standard payment providers (Stripe, etc.)
- Flow: `sendInvoice` → user pays → `pre_checkout_query` → `answerPreCheckoutQuery` → `successful_payment`
- Subscription plans and paid media (photos/videos behind paywall) supported

### Inline Queries

Users type `@botusername query` in any chat to get results:

```typescript
bot.on("inline_query", async (ctx) => {
  const results = [
    {
      type: "article",
      id: "1",
      title: "Result",
      input_message_content: { message_text: "Hello from inline!" },
    },
  ];
  await ctx.answerInlineQuery(results);
});
```

- Enable via BotFather `/setinline`
- Results appear as a dropdown above the keyboard

### Mini Apps (Web Apps)

Full custom HTML/JS interfaces inside Telegram:

- Launched via keyboard button, inline button, or menu button
- Access to user data, theme, haptic feedback
- Can send data back to the bot
- Useful for complex UIs (dashboards, forms, games)

## Plugin Ecosystem

### Key Plugins

| Plugin | Package | Purpose |
|--------|---------|---------|
| Conversations | `@grammyjs/conversations` | Multi-step dialogs and conversational flows |
| Menu | `@grammyjs/menu` | Dynamic button menus with navigation |
| Router | `@grammyjs/router` | Route messages to different handlers |
| Runner | `@grammyjs/runner` | Concurrent long polling at scale |
| Session | (built-in) | Persistent user data (memory, file, Redis, etc.) |
| Auto-retry | `@grammyjs/auto-retry` | Handle rate limiting (429 errors) automatically |
| Throttler | `@grammyjs/transformer-throttler` | Slow outgoing API calls to avoid flood limits |
| Rate Limiter | `@grammyjs/ratelimiter` | Rate-limit incoming user requests |
| Files | `@grammyjs/files` | Easy file downloading |
| Parse Mode | `@grammyjs/parse-mode` | Simplify HTML/Markdown formatting |
| i18n | `@grammyjs/i18n` | Internationalization |
| Chat Members | `@grammyjs/chat-members` | Track user join/leave events |
| Commands | `@grammyjs/commands` | Advanced command management with scopes |
| Hydrate | `@grammyjs/hydrate` | Call methods on API return objects |

### Plugin Types

1. **Middleware plugins**: Installed via `bot.use()` — handle incoming updates
2. **Transformer plugins**: Installed via `bot.api.config.use()` — transform outgoing API calls

## Group Privacy Mode

### How It Works

- **Enabled by default** for all bots
- In privacy mode, bots in groups only receive:
  - Commands explicitly addressed to them (`/command@this_bot`)
  - Replies to the bot's own messages
  - Service messages (member joins/leaves, title changes, etc.)
  - Inline messages sent via the bot
- Bots added as **group administrators** always receive **all messages**
- Privacy mode can be toggled via BotFather `/setprivacy`

### Recommendation

Keep privacy mode enabled unless the bot genuinely needs to see all messages. Use force reply or explicit commands instead of disabling privacy mode. Users can see the bot's privacy setting in the group members list.

## Access Control

### User Allowlist Middleware

```typescript
const ALLOWED_USERS = new Set([
  123456789,  // admin user ID
  987654321,  // dev user ID
]);

bot.use(async (ctx, next) => {
  if (!ctx.from || !ALLOWED_USERS.has(ctx.from.id)) {
    return ctx.reply("Unauthorized. Contact admin for access.");
  }
  return next();
});
```

### Admin-Only Commands

```typescript
const adminCommands = new Composer<Context>();

adminCommands.use(async (ctx, next) => {
  if (!ctx.chat || ctx.chat.type === "private") return next();
  const member = await ctx.getChatMember(ctx.from!.id);
  if (["administrator", "creator"].includes(member.status)) {
    return next();
  }
  return ctx.reply("This command requires admin privileges.");
});

adminCommands.command("deploy", (ctx) => { /* ... */ });
adminCommands.command("restart", (ctx) => { /* ... */ });

bot.use(adminCommands);
```

### Chat/Group Restriction

```typescript
const ALLOWED_CHATS = new Set([
  -1001234567890,  // ops group
  -1009876543210,  // dev group
]);

bot.use(async (ctx, next) => {
  if (ctx.chat && !ALLOWED_CHATS.has(ctx.chat.id)) {
    return; // silently ignore messages from unknown chats
  }
  return next();
});
```

### Rate Limiting

```typescript
import { limit } from "@grammyjs/ratelimiter";

bot.use(limit({
  timeFrame: 2000,   // 2 seconds
  limit: 3,          // max 3 messages per timeFrame
  onLimitExceeded: (ctx) => ctx.reply("Too many requests. Please wait."),
}));
```

### Command Scopes

Show different commands to different users:

```typescript
// Admin commands visible only to admins
await bot.api.setMyCommands(
  [{ command: "deploy", description: "Deploy to production" }],
  { scope: { type: "all_chat_administrators" } },
);

// Public commands visible to everyone
await bot.api.setMyCommands(
  [{ command: "status", description: "Check system status" }],
  { scope: { type: "default" } },
);
```

## Deployment

### Long Polling (VPS / Always-On)

```typescript
import { Bot } from "grammy";
import { run } from "@grammyjs/runner";

const bot = new Bot(process.env.TELEGRAM_BOT_TOKEN!);
// ... register handlers ...

// Simple (sequential)
bot.start();

// Concurrent (recommended for production)
const runner = run(bot);
runner.isRunning(); // true
// runner.stop(); to gracefully shut down
```

### Webhooks (Serverless)

```typescript
import { Bot, webhookCallback } from "grammy";
import express from "express";

const bot = new Bot(process.env.TELEGRAM_BOT_TOKEN!);
// ... register handlers ...

const app = express();
app.use(express.json());
app.post("/webhook", webhookCallback(bot, "express"));
app.listen(3000);
```

**Supported frameworks**: express, fastify, koa, oak, hono, next-js, sveltekit, cloudflare, aws-lambda, azure, bun, vercel, std/http (Deno.serve).

### Docker

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
CMD ["node", "dist/bot.js"]
```

### Process Management

```bash
# PM2 (recommended for VPS)
npm i -g pm2
pm2 start dist/bot.js --name telegram-bot
pm2 save && pm2 startup

# Systemd
# See matterbridge.md for systemd unit template pattern
```

### Local Bot API Server

For higher file limits (2 GB upload/download) and custom webhook ports:

- Source: https://github.com/tdlib/telegram-bot-api
- Enables: unlimited file downloads, 2 GB uploads, HTTP webhooks, any port, up to 100K webhook connections

## Privacy and Security Assessment

### Server-Side Message Storage

**Telegram is not E2E encrypted by default.** All regular chats (including bot conversations) are stored on Telegram's servers using client-server encryption (MTProto). Telegram has access to all non-Secret-Chat message content.

| Aspect | Status |
|--------|--------|
| Regular chats | Server-side encrypted (Telegram has access) |
| Secret Chats | E2E encrypted (not available for bots) |
| Bot messages | Server-side encrypted (Telegram has access) |
| Group chats | Server-side encrypted (Telegram has access) |
| Phone number | Required for account creation |
| Username | Optional but public |
| Metadata | Visible to Telegram servers (sender, recipient, timestamp, IP) |
| Push notifications | Via FCM/APNs — metadata exposed to Apple/Google |
| Data location | Distributed across multiple jurisdictions |
| AI training | No published policy on using chat data for AI training |

### Implications for aidevops

- **Do not send secrets, credentials, or sensitive code through Telegram bots** — Telegram servers can read all bot messages
- Bot tokens grant full control of the bot — treat as high-value credentials
- Telegram can comply with law enforcement requests for message content
- For sensitive communications, use SimpleX (E2E, no identifiers) or Matrix (optional E2E)
- Telegram is appropriate for: notifications, status updates, non-sensitive command dispatch, public-facing bot UX

### Bot Security Model

1. **Treat all inbound messages as untrusted input** — sanitize before passing to AI models or shell commands
2. **Prompt injection defense**: Scan inbound messages with `prompt-guard-helper.sh scan` before dispatching to runners
3. **Credential isolation**: Never expose secrets in bot responses or error messages
4. **User allowlist**: Restrict bot access to known user IDs (see Access Control above)
5. **Chat restriction**: Limit bot to specific groups/channels
6. **Rate limiting**: Prevent abuse with `@grammyjs/ratelimiter`
7. **Command sandboxing**: Bot commands dispatched to runners should run in restricted environments

## Integration with aidevops

### Runner Dispatch Pattern

```typescript
import { Bot, Context } from "grammy";
import { exec } from "child_process";
import { promisify } from "util";

const execAsync = promisify(exec);
const bot = new Bot(process.env.TELEGRAM_BOT_TOKEN!);

// Dispatch to aidevops runner
async function dispatchToRunner(
  runner: string,
  prompt: string,
): Promise<string> {
  const { stdout } = await execAsync(
    `runner-helper.sh dispatch "${runner}" "${prompt}"`,
    { timeout: 600_000 },
  );
  return stdout.trim();
}

// Map commands to runners
bot.command("review", async (ctx) => {
  const prompt = ctx.match || "Review latest changes";
  await ctx.reply("Dispatching to code reviewer...");
  await ctx.api.sendChatAction(ctx.chat.id, "typing");
  const result = await dispatchToRunner("code-reviewer", prompt);
  await ctx.reply(result, { parse_mode: "Markdown" });
});

bot.command("status", async (ctx) => {
  await ctx.api.sendChatAction(ctx.chat.id, "typing");
  const result = await dispatchToRunner("ops-monitor", "System status report");
  await ctx.reply(result);
});

bot.start();
```

### Potential Architecture with Entity System

```text
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ Telegram Chat     │     │ Telegram Bot      │     │ aidevops Runner  │
│                  │     │ (grammY + Node)   │     │                  │
│ User sends:      │────▶│ 1. Parse command  │────▶│ runner-helper.sh │
│ /review auth.ts  │     │ 2. Check perms    │     │ → AI session     │
│                  │◀────│ 3. Resolve entity │◀────│ → response       │
│ AI response      │     │ 4. Load context   │     │                  │
│                  │     │ 5. Dispatch       │     │                  │
└──────────────────┘     └──────────────────┘     └──────────────────┘
                                │
                                ▼
                    ┌──────────────────────┐
                    │ memory.db (shared)   │
                    │ ├── entities         │
                    │ ├── entity_channels  │
                    │ ├── interactions     │
                    │ └── conversations    │
                    └──────────────────────┘
```

Entity resolution would map Telegram user IDs to entities via `entity-helper.sh`, following the same pattern as the Matrix bot integration.

### Use Cases

| Scenario | Value |
|----------|-------|
| Deployment notifications | Push deploy status, CI results to team groups |
| Code review dispatch | `/review <file>` triggers AI code review via runner |
| System monitoring | `/status` returns infrastructure health from ops runner |
| Task management | `/task <description>` creates TODO entries |
| Approval workflows | Inline keyboards for deploy approvals (approve/reject buttons) |
| Forum-based ops | Use Telegram Topics for organized ops channels (bugs, deploys, alerts) |

## Matterbridge Integration

Telegram is natively supported by [Matterbridge](https://github.com/42wim/matterbridge) — no adapter needed.

### Configuration

```toml
[telegram]
  [telegram.main]
  Token="YOUR_BOT_TOKEN"
  # Optional: use HTML parse mode for richer formatting
  # MessageFormat="HTMLNick"

[[gateway]]
name="ops-bridge"
enable=true

  [[gateway.inout]]
  account="telegram.main"
  channel="-1001234567890"  # Supergroup chat ID (negative)

  [[gateway.inout]]
  account="matrix.home"
  channel="#ops:example.com"

  [[gateway.inout]]
  account="discord.myserver"
  channel="ops"
```

### Getting the Chat ID

- Add [@userinfobot](https://t.me/userinfobot) to the group, or
- Forward a group message to [@userinfobot](https://t.me/userinfobot)
- Supergroup IDs are negative (e.g., `-1001234567890`)

### Key Notes

- Matterbridge uses the Telegram Bot API directly — the bot must be a member of the group
- File/image bridging is supported
- Nick format customizable via `RemoteNickFormat`
- See `services/communications/matterbridge.md` for full configuration reference

### Privacy Gradient

Users who need maximum privacy use SimpleX or Matrix directly. Users who prefer convenience and reach use Telegram. Matterbridge bridges messages transparently between platforms, allowing each user to choose their preferred client.

**Warning**: Bridging to Telegram exposes messages to Telegram's servers. E2E encryption from SimpleX or Matrix is broken at the bridge boundary. See `tools/security/opsec.md` for implications.

## grammY vs Alternatives

| Aspect | grammY | Telegraf | node-telegram-bot-api |
|--------|--------|----------|-----------------------|
| TypeScript | Written from scratch in TS | Migrated from JS; complex types | Minimal/no types |
| Bot API version | Always latest (9.5) | Often lags behind | Lags behind |
| Documentation | Comprehensive website + guides | Generated API reference only | Basic README |
| Architecture | Middleware stack, Composer | Middleware (similar) | Single EventEmitter |
| Plugin ecosystem | Large, coordinated releases | Smaller | None |
| Multi-runtime | Node, Deno, Bun, browser | Node only | Node only |
| Maintenance | Actively maintained, 80+ contributors | Active but slower | Largely unmaintained |

**Recommendation**: Use grammY for new projects. Telegraf only if maintaining an existing Telegraf bot.

## Limitations

### No E2E Encryption for Bots

Bots cannot use Secret Chats. All bot messages are server-side encrypted only. Telegram has access to all bot conversation content.

### File Size Limits

Standard Bot API: 20 MB download, 50 MB upload. Self-hosted Bot API server raises this to 2 GB.

### Rate Limits

- ~30 messages/second to different chats
- ~20 messages/minute to the same group
- Bulk notifications: ~30 messages/second max
- Use `@grammyjs/auto-retry` and `@grammyjs/transformer-throttler` to handle automatically

### Phone Number Requirement

All Telegram users (including those interacting with bots) must have a phone number. This is a privacy concern — phone numbers are personally identifiable.

### Platform Dependency

Telegram is a centralized service operated by Telegram FZE (Dubai). Service availability, API terms, and data handling are at Telegram's discretion. There is no self-hosted alternative for the messaging platform itself (only the Bot API server can be self-hosted).

### Group Privacy Mode Constraints

In privacy mode (default), bots in groups only see commands and replies to the bot. Disabling privacy mode requires re-adding the bot to the group. Making the bot an admin bypasses privacy mode but grants additional permissions.

## Related

- `services/communications/matterbridge.md` — Multi-platform chat bridge (native Telegram support)
- `services/communications/matrix-bot.md` — Matrix bot for runner dispatch (E2E capable)
- `services/communications/simplex.md` — SimpleX Chat (zero-knowledge, E2E)
- `services/communications/xmtp.md` — XMTP (Web3 messaging, agent SDK)
- `tools/security/opsec.md` — Platform trust matrix, metadata warnings
- `tools/voice/speech-to-speech.md` — Voice message transcription
- `tools/ai-assistants/headless-dispatch.md` — Headless AI dispatch patterns
- grammY Docs: https://grammy.dev/
- grammY Plugins: https://grammy.dev/plugins/
- Telegram Bot API: https://core.telegram.org/bots/api
- Telegram Bot FAQ: https://core.telegram.org/bots/faq
