# Claude Development Guidelines

## Project Overview

EmailEngine = email sync platform. REST API access to email accounts. Supports IMAP/SMTP, Gmail API, Microsoft Graph (Outlook). Real-time webhooks for email events.

## Project Structure

- `server.js` - Main process orchestrator (see Main Process section)
- `/bin` - CLI executable entry point
- `/lib` - Core library modules (account, OAuth, email clients, API routes)
- `/lib/email-client` - Email client implementations (IMAP, Gmail API, Outlook Graph)
- `/lib/api-routes` - REST API route handlers
- `/lib/ui-routes` - Web UI route handlers
- `/lib/lua` - Redis Lua scripts for atomic ops
- `/lib/oauth` - OAuth provider implementations
- `/lib/imapproxy` - IMAP proxy server impl
- `/workers` - Worker thread modules (8 worker types, see Workers section)
- `/test` - Unit + integration tests
- `/config` - TOML config files
- `/views` - Handlebars templates for web UI
- `/static` - Frontend assets (CSS, JS)
- `/translations` - i18n files (7 languages)

### Key Files

- `lib/account.js` - Account class. Manages IMAP/SMTP interactions
- `lib/account/account-state.js` - Account state machine
- `lib/email-client/base-client.js` - Base email client class
- `lib/email-client/gmail-client.js` - Gmail API integration
- `lib/email-client/outlook-client.js` - Microsoft Graph integration
- `lib/oauth2-apps.js` - OAuth2 app configs
- `lib/export.js` - Export class for bulk email export ops
- `lib/api-routes/export-routes.js` - Export REST API endpoints
- `workers/api.js` - REST API worker with Hapi server
- `lib/routes-ui.js` - Admin UI route orchestrator (wires the `lib/ui-routes/*` modules)

## Technology Stack

- **Runtime**: Node.js >=20.x with Worker Threads
- **API Framework**: Hapi.js
- **Database**: Redis (ioredis) + BullMQ for job queues
- **Email**: ImapFlow (IMAP), Nodemailer (SMTP)
- **OAuth2**: Gmail API, Microsoft Graph, Mail.ru

## Development Commands

```
npm start         # Production mode
npm run dev       # Development mode (verbose logging, Redis DB 9)
npm test          # Run full test suite (lint + tests)
npm run format    # Format code with Prettier
npm run format:check  # Check formatting without changes
npm run lint      # Lint with ESLint
npm run swagger   # Generate OpenAPI docs
npm run single    # Single-worker debug mode with Inspector
```

## Testing

- Node.js native test runner + native assert module
- Tests run via Grunt: `npm test` runs `grunt` which runs Node.js test runner
- Tests in `/test` dir
- Redis DB 9 for test isolation
- `npm test` for full suite + lint

## Main Process (server.js)

Main process orchestrates all worker threads. Manages system lifecycle.

**Responsibilities:**
- Spawn + monitor worker threads (health checks every 5s via heartbeats)
- Assign email accounts to IMAP workers via load-balanced round-robin
- Route inter-thread RPC calls with configurable timeout (`EENGINE_TIMEOUT`, default: 10s)
- Manage Redis connection + monitor latency
- Handle license validation (every 20 min, 28-day grace period)
- Collect Prometheus metrics from all workers

**Startup sequence:**
1. Load license from file/env/Redis
2. Init settings (secrets, passwords, service ID)
3. Start API worker (wait for ready)
4. Start all IMAP workers (wait for ready)
5. Assign accounts to IMAP workers
6. Start webhooks, submit, documents workers
7. Start optional SMTP/IMAP proxy servers if enabled

**Account assignment:**
- Initial: Round-robin with load awareness (accounts per worker = ceil(total/workers))
- Reassignment after crashes: Rendezvous hashing for consistent routing
- Failsafe: 10s timeout ensures orphaned accounts get reassigned

**Key functions:**
- `spawnWorker(type)` - Create worker thread
- `assignAccounts()` - Distribute accounts to IMAP workers
- `call(worker, message)` - RPC with timeout
- `checkWorkerHealth()` - Monitor heartbeats, auto-restart dead workers

## Workers

EmailEngine uses Node.js Worker Threads for isolated execution. Workers talk to main thread (`server.js`) via message passing.

| Worker | File | Count | Purpose |
|--------|------|-------|---------|
| API | `api.js` | 1* | HTTP server for REST API and admin UI (see API Worker section below) |
| IMAP | `imap.js` | 4* | Email sync engine (see IMAP Worker section below) |
| Webhooks | `webhooks.js` | 1* | Webhook delivery processor (see Webhooks section below) |
| Submit | `submit.js` | 1* | Email delivery processor (see Submit Worker section below) |
| Export | `export.js` | 1* | Account data export processor (see Export Worker section below) |
| Documents | `documents.js` | 1 | **Deprecated.** Indexes emails in Elasticsearch (legacy feature) |
| SMTP | `smtp.js` | 1 | Optional SMTP server (see SMTP Server section below) |
| IMAP Proxy | `imap-proxy.js` | 1 | Optional IMAP proxy server (see IMAP Proxy section below) |

*Configurable via environment variables (`EENGINE_WORKERS`, `EENGINE_WORKERS_API`, `EENGINE_WORKERS_WEBHOOKS`, `EENGINE_WORKERS_SUBMIT`, `EENGINE_EXPORT_QC`). Multiple API workers (`EENGINE_WORKERS_API` > 1) require `SO_REUSEPORT` (Linux); on macOS/Windows/Node <23.1 it falls back to a single API worker.

**Worker Lifecycle:**
- Main thread spawns workers at startup. Monitors health via heartbeats (every 10s)
- IMAP workers get account assignments from main thread
- Workers auto-restart on crash. Accounts reassigned to available workers
- BullMQ queues distribute jobs to webhooks, submit, documents workers

### API Worker

API worker (`workers/api.js`) runs Hapi.js HTTP server. Serves REST API (`/v1/*`) + admin web UI (`/admin/*`).

**Server features:**
- REST API with OpenAPI/Swagger docs (`/admin/swagger`)
- Admin dashboard with Handlebars templates
- Server-Sent Events (SSE) for real-time account updates (`/admin/changes`)
- Static file serving, CSRF protection, i18n (7 languages)

**Authentication:**
- **API tokens**: Bearer token via `Authorization` header or `?access_token=` query param
- **Sessions**: Cookie-based (`ee` cookie) for admin UI
- **OAuth2**: Optional OKTA integration (`OKTA_OAUTH2_*` env vars)
- **TOTP**: Optional 2FA for admin login

**Token scopes:** `api`, `metrics`, `smtp`, `imap-proxy`, `*` (all)

**API route categories:**
- `/v1/account/{account}/*` - Account + message ops
- `/v1/token*` - API token mgmt
- `/v1/settings` - Global config
- `/v1/oauth2*` - OAuth2 app mgmt
- `/v1/webhooks*`, `/v1/templates*`, `/v1/gateways*` - Resources

**Configuration:**
- `EENGINE_PORT` / `PORT` - Listen port (default: 3000)
- `EENGINE_HOST` - Bind address (default: 127.0.0.1)
- `EENGINE_WORKERS_API` - Number of API/HTTP workers (default: 1). Values >1 share the port via `SO_REUSEPORT` (Linux only); on macOS/Windows/Node <23.1 EmailEngine falls back to a single worker with a warning
- `EENGINE_MAX_BODY_SIZE` - Max POST body (default: 25MB)
- `EENGINE_TIMEOUT` - Request timeout (default: 10s). Override with `X-EE-Timeout` header
- `EENGINE_API_PROXY` - Enable X-Forwarded-For parsing

**Key files:**
- `workers/api.js` - Hapi server setup and middleware
- `lib/routes-ui.js` - Admin UI route orchestrator (wires the `lib/ui-routes/*` route modules)
- `lib/api-routes/*.js` - REST API route modules
- `lib/tokens.js` - Token validation + CRUD

### IMAP Worker

IMAP worker (`workers/imap.js`) manages all email account connections + sync. Each worker handles many accounts via `ConnectionHandler` class.

**Connection types:**
- **IMAP**: Native IMAP via ImapFlow lib with IDLE for real-time sync
- **Gmail API**: OAuth2-based. Uses Pub/Sub for notifications (10-min polling fallback)
- **Outlook API**: Microsoft Graph with subscription webhooks (3-day auto-renewal)

**Synchronization:**
- IMAP: Persistent IDLE connection for real-time change detection
- Full mailbox sync on connect, then 15-min periodic resync
- UID tracking with UIDValidity validation (full resync if changed)
- Exponential backoff reconnect (2s initial, 30s max)

**Operations supported:**
- Message: `listMessages`, `getMessage`, `getText`, `getRawMessage`, `getAttachment`
- Message actions: `updateMessage`, `moveMessage`, `deleteMessage`, `uploadMessage`
- Mailbox: `listMailboxes`, `createMailbox`, `modifyMailbox`, `deleteMailbox`
- Account: `pause`, `resume`, `delete`, `getQuota` (IMAP only)

**Error handling:**
- Auth failures tracked. Auto-disable after threshold (4-hour window)
- Transient errors (timeout, DNS) → reconnect with backoff
- Permanent errors (5xx) fail immediately
- Excessive reconnect detection (>20/min triggers warning)

**Key files:**
- `workers/imap.js` - Worker thread with ConnectionHandler class
- `lib/email-client/imap-client.js` - IMAP impl
- `lib/email-client/gmail-client.js` - Gmail API impl
- `lib/email-client/outlook-client.js` - Outlook/Graph impl
- `lib/email-client/base-client.js` - Shared client logic

**Limitations:**
- Gmail/Outlook: `getQuota` not supported
- Gmail: No IDLE equivalent (polling fallback)
- Outlook: `uploadMessage` only works for drafts

### Webhooks

Webhooks system (`workers/webhooks.js`, `lib/webhooks.js`) delivers real-time HTTP POST notifications when email events fire. Uses BullMQ queue for reliable delivery with retries.

**Supported events:**
- Message events: `messageNew`, `messageDeleted`, `messageUpdated`, `messageSent`, `messageDeliveryError`, `messageFailed`, `messageBounce`, `messageComplaint`
- Mailbox events: `mailboxNew`, `mailboxDeleted`, `mailboxReset`
- Account events: `accountAdded`, `accountInitialized`, `accountDeleted`, `authenticationError`, `authenticationSuccess`, `connectError`
- Tracking events: `trackOpen`, `trackClick`, `listUnsubscribe`, `listSubscribe`
- Export events: `exportCompleted`, `exportFailed`

**Configuration levels:**
1. Global: `webhooksEnabled`, `webhooks` (URL), `webhookEvents` (whitelist)
2. Per-account: `webhooks` URL overrides global
3. Custom routes: Multiple URLs with JS filter/transform functions

**Delivery details:**
- Retries: 10 attempts with exponential backoff (start 5s)
- Auth: Basic auth via URL credentials, custom headers, or HMAC-SHA256 signature
- Signature header: `X-EE-Wh-Signature` (HMAC-SHA256 of body using service secret)
- Concurrency: Configurable via `EENGINE_NOTIFY_QC` (default: 1)

**Custom routes** (`lib/webhooks.js`):
- `fn` - JS filter function returning boolean (include/exclude event)
- `map` - JS transform function to modify payload pre-delivery
- Functions run in sandboxed SubScript env (30s timeout, 1MB max)

**Key files:**
- `workers/webhooks.js` - BullMQ worker processing webhook queue
- `lib/webhooks.js` - WebhooksHandler class for CRUD + payload formatting
- `lib/email-client/notification-handler.js` - Event emission to webhook queue

### Submit Worker

Submit worker (`workers/submit.js`) processes queued outbound emails via BullMQ. Delivers via SMTP or provider APIs (Gmail, Outlook). All email sending in EmailEngine = async.

**How it works:**
1. API/SMTP server queues message to Redis (content) + BullMQ (job metadata)
2. Submit worker picks up job from queue
3. Loads account, calls `submitMessage()` in base-client.js
4. Sends via SMTP or OAuth2 API based on account config
5. Fires webhook events for success/failure

**Retry logic:**
- Default: 10 attempts (`deliveryAttempts` setting)
- Backoff: Exponential start 5s (`5s, 10s, 20s, 40s...`)
- Retries on transient errors (< 500 status code)
- No retry on permanent 5xx errors (message rejected)

**Webhook events:**
- `messageSent` - Message accepted by SMTP server
- `messageDeliveryError` - Retryable error (includes `nextAttempt`)
- `messageFailed` - All retries exhausted, delivery failed

**Configuration:**
- `EENGINE_SUBMIT_QC` - Concurrency per worker (default: 1)
- `EENGINE_SUBMIT_DELAY` - Rate limit (e.g., `1s` = 1 msg/sec)
- `deliveryAttempts` setting - Default retry count (default: 10)

**Post-delivery actions:**
- Uploads to Sent folder (if IMAP account, not Gmail)
- Sets `\Answered` flag on replied messages
- Sets `$Forwarded` flag on forwarded messages
- Updates gateway delivery stats (if using gateway)

**Key files:**
- `workers/submit.js` - BullMQ worker impl
- `lib/email-client/base-client.js` - `queueMessage()` + `submitMessage()` logic
- `lib/outbox.js` - Queue inspection API

### Export Worker

Export worker (`workers/export.js`) processes bulk email export jobs via BullMQ. Extracts messages from accounts. Writes to compressed NDJSON files with optional encryption.

**How it works:**
1. API creates export job with date range + folder filters
2. Worker indexes matching messages from specified folders
3. Fetches message content in batches (parallel for API accounts, sequential for IMAP)
4. Writes to gzip-compressed NDJSON file (optionally encrypted)
5. Fires webhook events on completion or failure

**Export phases:**
- `pending` - Job queued, waiting for worker
- `indexing` - Scanning folders for matching messages
- `exporting` - Fetching + writing message content
- `complete` - Export finished

**Error handling and recovery:**
- **Transient errors** (network timeouts, 5xx): Retry with exponential backoff
- **Skippable errors** (message not found, 404): Skip message, increment counter
- **Account validation**: Checks every 60s if account still exists
**Retry configuration:**
- IMAP messages: 3 retries with 2s base delay (exponential backoff)
- API batch requests: 5 retries for rate limits (429) with 5s base delay
- Folder indexing: 3 retries with 1s base delay

**Webhook events:**
- `exportCompleted` - Export done with stats (messages exported, skipped, bytes)
- `exportFailed` - Export failed with error details + phase info

**Configuration:**
- `EENGINE_EXPORT_QC` - Concurrency per worker (default: 1)
- `EENGINE_EXPORT_TIMEOUT` - Operation timeout (default: 5 minutes)
- `EENGINE_EXPORT_PATH` - Export file directory (default: OS temp dir)
- `EENGINE_EXPORT_MAX_AGE` / `exportMaxAge` setting - Export file retention in ms (default: 24 hours)
- `exportMaxConcurrent` setting - Per-account concurrent limit (default: 3)
- `exportMaxGlobalConcurrent` setting - Global concurrent limit (default: 10)
- `exportMaxMessageSize` setting - Max attachment size (default: 25MB)

**API endpoints:**
- `POST /v1/account/{account}/export` - Create export job
- `GET /v1/account/{account}/export/{exportId}` - Get export status
- `GET /v1/account/{account}/export/{exportId}/download` - Download completed export
- `DELETE /v1/account/{account}/export/{exportId}` - Cancel/delete export
- `GET /v1/account/{account}/exports` - List exports with pagination

**Key files:**
- `workers/export.js` - BullMQ worker impl
- `lib/export.js` - Export class with CRUD + queue ops
- `lib/api-routes/export-routes.js` - REST API endpoints

### SMTP Server

SMTP server (`workers/smtp.js`) = built-in Message Submission Agent (MSA). Lets legacy apps send emails through EmailEngine via standard SMTP. Messages queued for async delivery via Submit worker.

**How it works:**
1. Client connects, optionally authenticates via SMTP AUTH
2. Client sends message with MAIL FROM, RCPT TO, DATA commands
3. Server queues message for delivery through associated EmailEngine account
4. Returns queue ID + scheduled send time

**Authentication methods:**
- With auth enabled (`smtpServerAuthEnabled`):
  - Username: Account ID
  - Password: Global password (`smtpServerPassword`) or 64-char hex token with `smtp` scope
- No auth: Specify account via `X-EE-Account` header in message

**Configuration** (settings or env vars):
- `smtpServerEnabled` / `EENGINE_SMTP_ENABLED` - Enable server
- `smtpServerPort` / `EENGINE_SMTP_PORT` - Listen port (default: 2525)
- `smtpServerHost` / `EENGINE_SMTP_HOST` - Bind address (default: 127.0.0.1)
- `smtpServerAuthEnabled` - Require SMTP auth
- `smtpServerPassword` / `EENGINE_SMTP_SECRET` - Global password (encrypted)
- `smtpServerTLSEnabled` - Enable TLS
- `smtpServerProxy` / `EENGINE_SMTP_PROXY` - Enable PROXY protocol (HAProxy)

**Special headers** (stripped before sending):
- `X-EE-Account` - Specify sending account (when auth disabled)
- `X-EE-Idempotency-Key` - Prevent duplicate submissions

**Limitations:**
- Max message size: 25MB (configurable via `EENGINE_MAX_SMTP_MESSAGE_SIZE`)
- Async delivery only (queued, not sent immediately)
- Account must have valid SMTP or OAuth2 creds

### IMAP Proxy

IMAP proxy (`lib/imapproxy/`) lets standard IMAP clients access EmailEngine-managed accounts. Abstracts OAuth2 complexity. Legacy clients can connect to Gmail, Microsoft 365, other OAuth2-only providers.

**How it works:**
1. Client connects, authenticates with account ID + password/token
2. Proxy validates creds, opens connection to real mail server
3. After auth, all IMAP commands pass through to backend

**Authentication methods:**
- Global password: Configure `imapProxyServerPassword` setting
- Access tokens: 64-char hex token with `imap-proxy` or `*` scope

**Configuration** (settings or env vars):
- `imapProxyServerEnabled` / `EENGINE_IMAP_PROXY_ENABLED` - Enable proxy
- `imapProxyServerPort` / `EENGINE_IMAP_PROXY_PORT` - Listen port (default: 2993)
- `imapProxyServerHost` / `EENGINE_IMAP_PROXY_HOST` - Bind address
- `imapProxyServerTLSEnabled` - Enable TLS
- `imapProxyServerProxy` - Enable PROXY protocol (HAProxy)

**Key files:**
- `lib/imapproxy/imap-server.js` - Main proxy server + auth logic
- `lib/imapproxy/imap-core/` - IMAP protocol impl (RFC 3501)

**Limitations:**
- No support for API-only accounts (e.g., Mail.ru API mode)
- Requires IMAP support on email provider

## Authentication Design

- **Passkey (WebAuthn) login = standalone auth method.** Bypasses TOTP intentionally. When user authenticates via passkey, no extra factor required. By design - passkeys treated as single sufficient factor.

## Architecture Notes

- **Multi-threaded**: 8 worker types (API, IMAP, webhooks, submit, export, documents, SMTP server, IMAP proxy)
- **Redis-backed**: Primary data store. Lua scripts for atomic ops
- **Encrypted**: All credentials encrypted at rest (AES-256-GCM)
- **State machine**: Account states (init, connecting, syncing, connected, authenticationError, connectError, unset)

## Environment Variables

**Core:**
- `EENGINE_REDIS` / `REDIS_URL` - Redis connection URI (default: `redis://127.0.0.1:6379/8`)
- `EENGINE_PORT` / `PORT` - API server port (default: 3000)
- `EENGINE_HOST` - API server bind address (default: 127.0.0.1)
- `EENGINE_TIMEOUT` - Command timeout in ms (default: 10000)
- `EENGINE_LOG_LEVEL` - Log level (default: trace)

**Workers:**
- `EENGINE_WORKERS` - IMAP worker count (default: 4)
- `EENGINE_WORKERS_API` - API/HTTP worker count (default: 1; values >1 need `SO_REUSEPORT`/Linux, otherwise falls back to 1)
- `EENGINE_WORKERS_WEBHOOKS` - Webhook worker count (default: 1)
- `EENGINE_WORKERS_SUBMIT` - Submit worker count (default: 1)
- `EENGINE_EXPORT_QC` - Export concurrency per worker (default: 1)
- `EENGINE_EXPORT_TIMEOUT` - Export operation timeout (default: 5 min)
- `EENGINE_NOTIFY_QC` - Webhook concurrency per worker (default: 1)

**Prepared configuration** (applied on startup):
- `EENGINE_SETTINGS` - JSON settings object
- `EENGINE_PREPARED_TOKEN` - Base64url msgpack-encoded API token
- `EENGINE_PREPARED_PASSWORD` - Base64url PBKDF2 password hash
- `EENGINE_PREPARED_LICENSE` - License key

## Code Style Rules

- Never use emojis in code or documentation, only printable ASCII characters
- Use a single hyphen-minus (`-`) as a dash in UI copy and user-facing strings. Never use double hyphens (`--`), em dashes, or en dashes.
- When composing git commit messages do not include Claude as co-contributor
- For commits that do not change runtime behavior (docs, comments, CI/workflow tweaks, formatting), append `[skip ci]` to the commit message to avoid triggering the GitHub Actions workflows. Exception: do not add `[skip ci]` to commits using a `fix:` or `feat:` prefix - those must run so the release action is triggered.
- After making code changes:
  1. Run `/simplify` to review changed code for reuse, quality, and efficiency
  2. Run `npm run format` and `npm run lint`
  3. Run `/security-review` to check for security issues before commit
- After pushing, check GitHub Actions runs for the push (e.g. `gh run list --branch <branch>`) + report status. If run fails for strange/unrelated reason (checkout "account suspended", HTTP 403, auth/infra errors unrelated to change), check https://www.githubstatus.com/ for active GitHub incident before assuming code caused it.
- Avoid circuit breaker pattern unless absolutely necessary. EmailEngine processes many independent accounts through shared workers. Single failing account can trip circuit breaker, block all others. Prefer per-account error handling (retry with backoff, error state tracking) over global circuit breakers.
- Never suppress or swallow unhandled rejections/exceptions at global handler level. If error reaches global `unhandledRejection` or `uncaughtException` handler, worker must die -- last line of defense. Correct fix: handle error at source so it never bubbles to global handler. Means proper try/catch, .catch(), or error event handlers at actual call site. If unhandled rejection comes from dependency (e.g. ImapFlow), fix in dependency itself.

## Dependencies We Maintain

- **ImapFlow** (`../imapflow`): ImapFlow IMAP client library maintained by us. Local dev copy at `../imapflow` relative to project root. When bugs or unhandled promise rejections originate in ImapFlow, fix directly in ImapFlow source rather than work around in EmailEngine.