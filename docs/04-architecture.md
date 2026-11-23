# 4. Target architecture

The recommended shape is **one domain core, three surfaces, two processes**.
This document specifies that shape: the boundaries, the data model, the
transport details, and the deployment.

## Principles

1. **One implementation of each rule.** A rule lives in `internal/domain` and is
   called by every adapter. If a handler needs to know the reply-depth cap, the
   cap is returned by the domain, not re-declared in the handler.
2. **Thin adapters.** SSH, telnet and HTTP translate bytes and forms into domain
   calls. They do not own policy.
3. **One writer per aggregate.** Interactive tables are written by `bbsd serve`.
   APRS tables are written by `bbsd aprs`, except the outbox, which interactive
   code appends to and only the APRS service updates.
4. **The queue is the boundary.** Anything that leaves the BBS for APRS-IS goes
   through `aprs_outbox`. No handler opens a socket to APRS-IS.
5. **Fail closed and loudly.** A write that did not commit is an error. A
   surface that cannot reach the database is not ready.
6. **Parity is tested, not asserted.** Every domain operation has a test that
   runs it through the terminal adapter and the HTTP adapter and compares the
   resulting rows.

## System shape

```
                 SSH 2222            telnet 2323           HTTPS 443
                    |                    |                    |
                    v                    v                    v
        +-----------------------------------------------------------+
        |                      bbsd serve                           |
        |  internal/term (screen model)      internal/http          |
        |  SSH transport   telnet transport  HTML + JSON + SSE      |
        |                    \                  /                   |
        |                     v                v                    |
        |                  internal/domain  (rules)                 |
        |                           |                               |
        |                  internal/store (SQLite)                  |
        +---------------------------|-------------------------------+
                                    |
                          /var/lib/bbs/bbs.sqlite
                                    |
        +---------------------------|-------------------------------+
        |                      bbsd aprs                            |
        |  receiver -> store -> ACK queue -> sender (rate limited)   |
        |  bots (APRSPH, ANSRVR)   bulletins   retention sweeps      |
        +---------------------------|-------------------------------+
                                    |
                              APRS-IS 14580
```

Two Deployments, one image, one database file. `bbsd serve` is the only process
a caller talks to. `bbsd aprs` is the only process that talks to APRS-IS.

## Process model

### Why one interactive process

Today every SSH connection is a new `bbs_app` process: schema check, seeding,
housekeeping, user scan, then the login prompt. At 10 000 accounts that is 4.6 s
before the prompt (`SCALABILITY.md`). It also means every session has its own
in-memory throttle and its own view of the world.

`bbsd serve` accepts all SSH and telnet connections in one process and gives
each a goroutine. Startup work happens once. The database handle is shared. The
session registry is shared. The throttle is shared.

The cost is blast radius: a panic in one session could take the process down.
That is contained by recovering per session and per request, and by keeping the
process small. It is a better trade than paying O(accounts) work per connection.

### Why APRS stays separate

Because reception must not depend on the interactive service's rollout. That
lesson is already learned and implemented (`bbs-aprs` with
`BBS_APRS_EMBEDDED=false`); the successor keeps it.

`bbsd aprs` holds a lease row in the database with a heartbeat. If the lease is
lost, it stops sending and acknowledging, and reports not-ready. A file lock
remains as a second line of defence on a shared volume. There must never be two
processes acknowledging the same message; a duplicate ACK on the air under the
operator's callsign is a real harm.

### Command surface

One binary, three subcommands:

- `bbsd serve` — SSH, telnet, HTTP. Flags to disable a listener for development.
- `bbsd aprs` — receiver, sender, bots, bulletins, retention.
- `bbsd admin` — `migrate`, `migrate --check`, `grant-sysop`, `revoke-sessions`,
  `export-legacy`, `verify-migration`.

`bbsd admin migrate` is run by an init container, not by the serving pods, so a
rollout cannot start a pod against a half-migrated schema.

## Package layout

```
cmd/bbsd/                 wiring only
internal/domain/          accounts, boards, threads, bulletins, directory, policy
internal/store/           SQLite, migrations, repositories, transactions
internal/term/            screen model, input, rendering, layout
internal/term/ssh/        SSH listener and pty handling
internal/term/telnet/     IAC negotiation and the telnet stream
internal/http/            router, sessions, templates, JSON API, SSE
internal/aprs/            packet codec, APRS-IS client, outbox, ACK matcher, bots
internal/mail/            SMTP and the four message flows
internal/i18n/            catalogue loading and formatting
internal/observability/   logs, metrics, health
api/openapi.yaml          the HTTP contract
```

A rule that appears in `internal/http` or `internal/term` and not in
`internal/domain` is a defect in review. This is the invariant that replaces
`ham-bbs-web`'s "mirror the SSH BBS" instruction.

## Data model

The schema is new, but every column has a source in the current database. The
mapping is in [05-migration.md](05-migration.md).

### Identity and access

- `users` — callsign as primary key (upper case), profile fields, language,
  `enable_aprs`, password hash and algorithm, role flags, `first_seen`,
  `last_seen`, `registered`, `activated`.
- `sessions` — hashed token, callsign, surface (`web`), created, expires,
  last seen, revoked. Terminal sessions are not persisted; they end with the
  connection.
- `auth_tokens` — hashed token, purpose (`reset`, `email_confirm_old`,
  `email_confirm_new`), callsign, target email, change id, expiry, consumed.
- `activations` — callsign, code hash, expiry, attempts.
- `auth_attempts` — callsign, source address, occurred at. Indexed for the
  fifteen-minute window.
- `registrations` — source address, window start, count.
- `audit_log` — actor, action, target, detail, at. Written by sysop actions.

The password hash keeps the current format
`pbkdf2_sha256$<iterations>$<salt>$<digest>` and gains an `algo` column so a
future Argon2id hash can be introduced without a flag day. Verification accepts
both; a successful login with an old algorithm rehashes in place.

### Content

- `boards` — id (slug, primary key), position, name, description, created.
- `messages` — id, board id, `thread_id` (root message), `parent_id`,
  `depth` (1-based), position, author callsign, subject, body, created,
  edited at/by, deleted at/by.
- `bulletins` — id, position, title, body, updated, author.

`thread_id` and `depth` are denormalised so the reply cap is a column check
rather than a recursive walk, and so a thread can be fetched with one indexed
query. The cap remains five. The importer computes both from `parent_id` and
refuses to invent a depth: a row deeper than five is reported, not silently
re-parented.

Deletion is soft. A sysop delete sets `deleted_at`/`deleted_by` and hides the
row; it does not destroy it. This is a deliberate change from the current hard
delete, and it is what makes the audit trail meaningful.

### APRS

- `aprs_sent` — author, at, from, to, text, status, acked, message id.
- `aprs_sent_part` — sent id, number, text, status, detail, message id, acked.
- `aprs_received` — account, at, from, to, text, raw.
- `aprs_bulletin` — dedupe key (unique), first seen, last seen, seen count,
  sender, slot, group, category, text, raw.
- `aprs_outbox` — id, created, kind (`message`, `ack`, `bot`, `beacon`),
  payload, status, attempts, next attempt, sent at, message id, last error.
- `aprs_lease` — name, holder, heartbeat, expires.

`aprs_outbox` is the only table written by both processes: interactive code
appends, the APRS service consumes and updates. The state machine is
`queued -> sending -> sent | failed`, and the transition to `sent` is the only
place a message id is finalised.

Indexes that matter: `aprs_received(user_callsign, at)`, `aprs_sent(user_callsign,
at)`, `aprs_bulletin(category, last_seen)`, `aprs_outbox(status, next_attempt)`,
`messages(board_id, position)`, `messages(thread_id, position)`,
`users(email COLLATE NOCASE)`.

### Search

Phase 2 adds an FTS5 table over message subject and body, kept in step by
triggers, with the same sanitiser applied before indexing. Search is one domain
function; the terminal and the web both call it.

## Identity and authentication

### Callsign login

The callsign is the account key, as today. It is not proof of identity; the
current system treats accounts as unverified pseudonyms, and the design keeps
that, but stops making it worse:

- `BBS_SYSOPS` bootstrap is replaced by `bbsd admin grant-sysop <callsign>`,
  run by the operator against the live database. It writes an audit row. There
  is no window in which the first caller to type a listed callsign becomes a
  sysop.
- An optional `BBS_BOOTSTRAP_TOKEN` may exist for development, but it is
  documented as unsafe for production and logs a warning when used.

### Transport authentication

- **SSH**: host keys from a Secret (preferred, so they survive a claim change)
  or the existing `/config/ssh` volume. Callers authenticate with public key, or
  with a transport password when `BBS_SSH_PASSWORD_AUTH=true`.
  The SSH username is ignored; the BBS asks for the callsign. On HamNet only,
  `BBS_SSH_NO_AUTH=true` may disable transport authentication, and the
  NetworkPolicy must be verified before that is enabled.
- **Telnet**: no transport authentication. The BBS login is the only gate. This
  is why telnet is not reachable from the public internet.
- **Web**: callsign and password form; session cookie `HttpOnly`, `Secure`,
  `SameSite=Lax`; CSRF token on every state-changing form.

### Rate limits

One table, one rule, all surfaces: five failures for a (callsign, source) pair
in fifteen minutes locks that pair for fifteen minutes. Five registrations per
address per hour. The counters are rows, not process memory, so an SSH attempt
counts against a web attempt and a restart does not clear the budget.

## Terminal adapter

### SSH

`golang.org/x/crypto/ssh` with a server config that offers only `pty-req`,
`window-change` and `shell`. No forwarding, no tunnels, no X11, no agent
forwarding. One session per connection; idle timeout and a maximum concurrent
session count.

The host keys are stable across restarts and stored outside the image.

### Telnet

A minimal, correct negotiation:

- refuse `LINEMODE`;
- negotiate `ECHO`, `SUPPRESS-GO-AHEAD`, `TERMINAL-TYPE` and `NAWS`;
- strip IAC sequences from input, escape a literal `0xFF` in output;
- use CRLF line endings;
- keep the stream 8-bit clean for UTF-8.

The negotiation completes before the screen model starts, so both transports
hand the same `io.ReadWriteCloser` and the same initial size to `internal/term`.

### Screen model

Bubble Tea remains the renderer, because the existing screens and tests are
built on it. The differences from today:

- the layout adapts to the reported size, with 80×24 as the minimum and 132×24
  as the preferred size; the app no longer assumes it can resize the caller's
  emulator;
- a line-mode fallback renders numbered menus and prompts for clients that
  report `TERM=dumb` or fail the cursor-addressing probe;
- colours are chosen from the terminal's reported capability, not assumed.

Both transports share every screen. There is no second terminal implementation.

### WebSocket terminal (optional)

If a browser terminal is wanted later, it plugs into the same `internal/term`
stream and needs no new rules. It is not a substitute for the web interface.

## HTTP adapter

Two consumers, one router:

- **HTML** — server-rendered templates, the existing "Packet Manual structure,
  Spectrum palette" design, progressive enhancement. No build toolchain, no SPA.
- **JSON** — `/api/v1`, Bearer tokens, the contract in `api/openapi.yaml`.

Server-sent events at `/api/v1/events` push new APRS messages, ACK changes and
new board posts to open pages. Polling remains a fallback. SSE is chosen over
WebSocket because the traffic is one-way and the proxy story is simpler.

The API starts from the contract described in `ea7klk/bbs-api`, including the
APRS sections already sketched there. `new-bbs` becomes the owner of that
contract (ADR-007). The API is a first-class surface, not a side effect of the
templates: if a page can do it, the API can do it.

## APRS service

Fully specified in [06-aprs.md](06-aprs.md). The architectural points:

- one APRS-IS connection, reconnected hourly;
- one outbound queue, one rate limiter (one message every ten seconds), one ACK
  matcher;
- bots and bulletin collection in the same process, because they share the
  connection and the identity;
- the lease prevents a second process from acting.

## Deployment

### Workloads

| Object | Purpose |
| --- | --- |
| `Deployment/bbs` | `bbsd serve`: SSH 2222, telnet 2323, HTTP 8080, metrics 9090 |
| `Deployment/bbs-aprs` | `bbsd aprs`: no inbound ports, egress to APRS-IS |
| `Service/bbs` | ClusterIP for 2222, 2323, 8080 |
| `Service/bbs-metrics` | ClusterIP for 9090, scraped by the existing stack |
| `IngressRoute/bbs-ssh` or `hostPort` | TCP 2222 to the Service, preserving the current public port |
| `PersistentVolumeClaim/data` | One claim, ReadWriteOnce, mounted at `/var/lib/bbs` |
| `Secret/bbs-ssh-host-keys` | The three SSH host keys, carried over so fingerprints do not change |
| `IngressRoute/bbs-web` | `bbs.ea7klk.es` over HTTPS |
| `IngressRoute/bbs-hamnet` | HTTP router for the HamNet address, allow-listed to `44.0.0.0/8` |
| `NetworkPolicy/bbs` | SSH from anywhere; telnet from HamNet and the cluster only; HTTP from Traefik |
| `Job` or init container | `bbsd admin migrate` before the new pods serve |

The existing namespace, Secrets (`bbs-env`, `bbs-smtp`) and Traefik entrypoints
are reused. The namespace stays `ham-bbs`; renaming it would churn DNS,
certificates, Secrets and the Fleet bundles for no functional gain.

### SSH and HTTP exposure

SSH keeps port 2222. Today it is published with `hostPort: 2222`, which binds
every node address; that is how the HamNet address will reach it without a
change (`HAMNET.md`). The design prefers a Traefik TCP router to the ClusterIP
because it keeps the pod off the host network, but `hostPort` is acceptable if
the node firewall and the NetworkPolicy are verified. Either way, the caller's
host and port do not change.

HTTP is exposed only through Traefik: an HTTPS router for `bbs.ea7klk.es`, and
an HTTP-only router for the HamNet address with an `IPAllowList` of
`44.0.0.0/8`, as described in `HAMNET.md`. The public host never serves plain
HTTP.

### Telnet exposure

Telnet must never be reachable from the public internet. Two mechanisms, both
required:

1. **NetworkPolicy** allowing port 2323 only from `44.0.0.0/8`, the cluster
   CIDR and the operator's LAN.
2. **A HamNet-only path**: either a Traefik TCP router bound to the HamNet
   address, or a `hostPort` on 2323 with a node firewall rule that permits only
   `44.0.0.0/8`. Prefer the router. Verify with an external connection attempt
   from a non-HamNet address, not by reading the manifest.

NetworkPolicy enforcement depends on the CNI. The verification is an actual
connection from outside, not an assumption.

### Migration job

The init container runs `bbsd admin migrate` with the data claim mounted. It
takes an exclusive database lock, applies numbered migrations, records the
version, and exits. `bbsd serve` refuses to start if the schema version is not
current; `bbsd aprs` does the same.

For the cutover this is belt and braces: the new pods mount a **new claim**, so
the old pods never see the new schema (see
[05-migration.md](05-migration.md)). For ordinary upgrades after cutover, a
migration must be backward-compatible with the previous release, or it must
follow an expand/contract sequence across two releases. A migration that
drops or renames a column cannot be rolled back by reverting an image.

### Resources

| Workload | Request | Limit |
| --- | --- | --- |
| `bbs` | 100 m CPU, 128 MiB | 1 CPU, 512 MiB |
| `bbs-aprs` | 25 m CPU, 64 MiB | 250 m CPU, 256 MiB |

The values are starting points. The metrics in `SCALABILITY.md` suggest memory
is dominated by the page cache, not the process.

### Probes and observability

- `/healthz` — process is alive. No database access.
- `/readyz` — database open, schema current, and, for `bbsd aprs`, the lease is
  held and the last APRS-IS connect is within the expected window.
- Metrics: session count by transport, login outcomes, request latency, DB busy
  retries, outbox depth and age, APRS-IS connection state, ACKs sent and
  matched, bulletins stored, drops.
- Logs: structured JSON to stdout; no passwords, tokens or message bodies at
  info level.

### Backup

A nightly Job runs `sqlite3 /var/lib/bbs/bbs.sqlite ".backup /backup/bbs-<date>.sqlite"`,
verifies `PRAGMA integrity_check`, and copies off the node with retention. The
current deployment has no visible BBS backup; this is a gap the successor must
close before cutover, not after.

## Security model

| Threat | Control |
| --- | --- |
| Credential stuffing | Shared per-(callsign, address) throttle, surfaced identically on all transports |
| Terminal escape injection | One sanitiser on write and on terminal render; tested against stored sequences |
| Stored XSS in the web UI | Contextual escaping by `html/template`; no `template.HTML` on user data |
| CSRF | Token per session, checked on every non-GET request |
| Session theft | HttpOnly, Secure, SameSite=Lax, short expiry, revocation on password change |
| Telnet eavesdropping | Not exposed publicly; documented as plaintext; HamNet only |
| APRS spoofing | Accepted as inherent; no privilege is granted by an APRS claim |
| Outbound APRS storm | Queue cap, one send per ten seconds, ACK queue budget, drop-and-log |
| SQL injection | Parameterised queries only; no string-built SQL |
| Secret leakage | Secrets from `bbs-env`/`bbs-smtp`; never logged; no passcode stored |
| Dependency risk | `govulncheck` in CI; images pinned by digest |
| Privilege escalation in the pod | Non-root, read-only root filesystem, no added capabilities |

The sysop bootstrap race (H1) is closed by the CLI grant. Callsign ownership is
still unverified; that is a product decision recorded in
[09-open-questions.md](09-open-questions.md).

## Failure modes

| Condition | Behaviour |
| --- | --- |
| SQLite busy beyond the timeout | Request fails with a retryable error; no partial write; page shows a retry message |
| Disk full | Writes fail loudly; readiness false; alert fires |
| APRS-IS unreachable | Outbox grows; sends show "queued"; alert at depth or age threshold; no false "sent" |
| Lease lost | APRS service stops sending/acking and reports not-ready; the replacement acquires the lease |
| Duplicate inbound packet | Stored as received, as today; only ACK emission is deduped within a short window |
| Duplicate outbound delivery | `aprs_outbox` state transition is idempotent; a message id is assigned once |
| Web SSE disconnect | Client reconnects with `Last-Event-ID`; missed events are re-read from the database |
| Migration failure | Init container exits non-zero; new pods do not start; the old deployment keeps running |
| Clock skew | All timestamps are UTC from the application; the node runs NTP; the UI renders relative ages |

## Scaling path

- **Up to ~5 000 accounts**: this design, one interactive process.
- **5 000–20 000, or visible write contention**: split `serve` into `bbs-ssh`
  and `bbs-web` — same packages, two Deployments. Sessions and throttles are
  already in the database, so this is a deployment change, not a rewrite.
- **Beyond that, or any need for multiple nodes**: PostgreSQL, multiple
  replicas, and the APRS lease becomes a proper leader election. The domain
  layer does not change; the store does.

The trigger is measured write latency and busy retries, not account count.

## What is reused

| Source | Reused |
| --- | --- |
| `simple-ham-bbs` | Domain rules, APRS codec and receiver, ACK logic, bots, bulletin classification, sanitiser, mail flows, translations |
| `ham-bbs-web` | Templates, stylesheet, page structure, account flows, accessibility work |
| `bbs-api` | OpenAPI contract and API design decisions |
| `simple-ham-bbs/SCALABILITY.md` | The evidence for in-process sessions and indexed lookups |
| `simple-ham-bbs/REVIEW.md` | The security findings the design must not reintroduce |
