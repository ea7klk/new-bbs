# 6. APRS

APRS is the reason this BBS exists in its current form, and the part most likely
to be broken by a rewrite. This document specifies how it survives: one service,
one queue, one sender, and two surfaces that call the same functions.

The protocol behaviours in this document are copied from the current
implementation and its tests. They are not up for redesign.

## The contract

These behaviours must not change:

| Behaviour | Rule |
| --- | --- |
| Sender SSID | Outbound messages leave from the account's base callsign with SSID 0, whatever the recipient's SSID |
| ACK SSID | An ACK for a message addressed to `CALL-15` is sent from `CALL-15` |
| Splitting | Messages longer than the APRS budget are split into numbered parts |
| Bot commands | Never split, never translated, sent exactly as documented |
| Retention | 200 sent and 500 received per account |
| Reception | Only messages addressed to accounts with `Enable APRS` are stored |
| Message IDs | A sender's trailing `{XXXXX` is stripped from display text and kept in `raw` |
| Bulletins | Global, deduped, categorised, ten days from `last_seen`, never acknowledged |
| ACK pacing | At most one ACK every ten seconds, queue of 64, drops are counted and logged |
| Receiver lifetime | Reconnect hourly |
| Thursday net | `#APRSThursday` is offered only on Thursdays; `U HOTG` leaves, `K HOTG` extends |
| APRSPH | A bot at callsign `APRSPH`, not a group |

## Components

One process, `bbsd aprs`, with five responsibilities. They share one APRS-IS
connection, one identity and one lease.

### Ingest

1. Connect to APRS-IS, log in with the receiver callsign and `filter t/m`.
2. Parse message-format packets. Reject malformed lines with a counted log line,
   not a panic.
3. Classify the addressee:
   - `BLNn` plus a group → bulletin path;
   - a callsign matching an account with `Enable APRS` → inbox path;
   - anything else → dropped, counted.
4. Sanitise the text, store it with the raw packet, and enqueue an ACK when the
   message was accepted for an account. Bulletins are never acknowledged.
5. Reconnect hourly so a stale TCP connection cannot silently stop reception.

### Store

Received messages and bulletins are written by this process only. Reads come
from every surface. Retention runs here:

- per account, keep the newest 500 received rows;
- bulletins: delete rows whose `last_seen` is older than ten days, then enforce
  a hard row cap;
- the sweep runs nightly and is idempotent.

### Outbox

Every outbound action is a row, not a socket write in a handler:

```
queued -> sending -> sent
                  -> failed (with error, bounded retries)
```

- Producers: the terminal send screen, the web send form, the HTTP API, the bot
  menu and the ACK path. All of them append.
- Consumer: the sender. Only the sender updates status.
- The `message_id` is assigned once, when the row is created, so a retry cannot
  produce a second id.
- `sent` is written only after APRS-IS has accepted the line. A caller never
  sees "sent" for a queued message.

### Sender

- One APRS-IS connection for outbound traffic.
- At most one message every ten seconds. The default is the current behaviour;
  it is configurable but not disabled by default.
- Queue cap 64. When it is full, the new item is rejected with a clear error and
  a logged drop count. It is never silently discarded.
- Bounded retries with backoff; a permanent failure is visible in the sent list
  as `failed`.
- Messages are sent from the author's base callsign with SSID 0. ACKs are sent
  from the SSID of the message being acknowledged.

### ACK matcher

When a message-format packet is an ACK, match it to an outbound part by
(message id, sender callsign, addressee). Mark the part acknowledged, and the
parent message acknowledged when every part is acknowledged. An ACK that matches
nothing is counted and logged, not stored as a message.

The matcher must be idempotent: the same ACK arriving twice marks the same rows
and does not create a second state transition.

### Bots and bulletins

- **ANSRVR / #APRSThursday**: `CQ HOTG <text>` to `ANSRVR`; `U HOTG` to leave;
  `K HOTG` to extend. Offered only on Thursdays, evaluated in UTC.
- **APRSPH**: the documented command set, exposed as a submenu so the syntax does
  not have to be remembered. Each command is one message and is never split.
- **Bulletins**: parse `BLNn` and the five-character group, dedupe by content,
  classify by declared group then text keywords, and store globally.

Bulletin classification is the weakest part of the current implementation: 74%
of the 5 138 rows are `general` or `other`. The successor keeps the classifier
behind a domain function so it can be improved without touching a screen, and
adds a monthly report of the unknown groups actually seen.

### Beacons

A login beacon is a position report, not a message. Today both the SSH app and
the web app dial APRS-IS directly on login, whenever `Enable APRS` is true and
the profile has a valid Maidenhead locator. The packet is built from the locator
centre, sent from the base callsign with SSID 0, and carries the comment
`HamNet BBS`. It is not stored in the sent-message history.

In `new-bbs`:

- the login path enqueues `kind=beacon`; it never dials;
- the sender emits it like any other outbound item, with the same connection and
  the same passcode derivation;
- a beacon is skipped, with a log line, when `Enable APRS` is false, the locator
  is missing or invalid, or the callsign fails validation;
- a per-account minimum interval (default five minutes) prevents a login storm
  from becoming a beacon storm. This is a deliberate tightening of today's
  every-login behaviour, recorded as ADR-011.

Beacons are counted and logged but are not part of the 200-message sent history,
because they are not messages and including them would evict real history.

## Data model

```
aprs_sent(id, user_callsign, at, from, to, text, status, acked, message_id)
aprs_sent_part(id, sent_id, number, text, status, detail, message_id, acked)
aprs_received(id, user_callsign, at, from, to, text, raw)
aprs_bulletin(id, dedupe_key, content_key, first_seen, last_seen, seen_count,
              from, slot, group, category, text, raw)
aprs_outbox(id, created_at, kind, callsign, target, payload, status, attempts,
            next_attempt_at, sent_at, message_id, last_error)
aprs_lease(name, holder, heartbeat_at, expires_at)
```

Indexes: `(user_callsign, at)` on both history tables; `(category, last_seen)`
on bulletins; `(status, next_attempt_at)` on the outbox; unique `dedupe_key` on
bulletins.

There is no `passcode` column. The passcode is derived when a connection needs
it, and the derived value is never written down.

## Text surface

The terminal menu keeps the current shape:

```
APRS
  1  Send message
  2  Received messages
  3  Sent messages and acknowledgements
  4  APRS bulletins
  5  Nets and bots
       1  #APRSThursday check-in
       2  APRSPH commands
  6  Settings
```

Screen behaviour:

- **Send**: recipient, text, a live character count and a preview of how the
  message will be split. The screen states that the message leaves as
  `CALL-0`. Validation is the domain's, not the screen's: an invalid callsign or
  an empty body is refused before a queue row is written.
- **Received**: newest first, paginated, sender and relative age; detail shows
  the text and the raw packet. The list refreshes on entry; there is no live
  push to a terminal.
- **Sent**: each message with its parts and their status. A queued message says
  "queued", not "sent". A failed message shows the reason.
- **Bulletins**: category list with counts, then items, then detail. Open to any
  logged-in user, not gated on `Enable APRS`.
- **Nets and bots**: the Thursday entry is hidden or refused outside Thursday
  (UTC), with the reason shown. APRSPH commands are presented as labelled
  actions; the command text is never translated.
- **Settings**: `Enable APRS` on/off, with a plain explanation of what it
  controls: receiving and sending, not reading bulletins.

All labels come from the same four-language catalogue as the rest of the
terminal. Command strings and protocol text do not.

## Web surface

The web mirrors the terminal, not the other way round.

| Page | Route | Notes |
| --- | --- | --- |
| APRS home | `/aprs` | Inbox count, queue state, APRS-IS state, shortcuts |
| Inbox | `/aprs/inbox` | Paginated, live updates via SSE |
| Sent | `/aprs/sent` | Status per message and per part, ACK state |
| Send | `/aprs/send` | Same validation and split preview as the terminal |
| Bulletins | `/aprs/bulletins` | Category list, items, detail |
| Nets and bots | `/aprs/nets` | Thursday gate, APRSPH actions |
| Settings | `/aprs/settings` | `Enable APRS`, with the same explanation |

The JSON API exposes the same operations under `/api/v1/aprs/`. The contract is
in `api/openapi.yaml` and is exercised by the web UI itself, so it cannot drift.

Server-sent events at `/api/v1/events` carry `aprs.received`, `aprs.acked` and
`aprs.bulletin` events. A browser that loses the stream reconnects with
`Last-Event-ID`; missed events are re-read from the database, so the UI never
depends on the stream for correctness.

The web never opens a socket to APRS-IS. Sending from the web appends to
`aprs_outbox` exactly as the terminal does.

## Parity matrix

Each row is one domain function, called by both adapters.

| Capability | Terminal | Web | Domain function |
| --- | --- | --- | --- |
| Compose and validate a message | Send screen | Send form | `aprs.Compose` |
| Split into parts | Preview | Preview | `aprs.Split` |
| Enqueue | Send | Send | `aprs.Enqueue` |
| Queue state | Sent list | Sent list + SSE | `aprs.QueueState` |
| Inbox | Received list | Inbox + SSE | `aprs.Inbox` |
| History retention | implicit | implicit | `aprs.Trim` (service only) |
| ACK state | Sent detail | Sent detail | `aprs.AckState` |
| Toggle `Enable APRS` | Settings | Settings | `accounts.SetAPRS` |
| Login beacon (automatic) | On login | On login | `aprs.EnqueueBeacon` |
| Bulletins by category | Menu | Page | `bulletins.List` |
| Thursday check-in | Net menu | Net page | `bots.HOTG` |
| APRSPH commands | Bot menu | Bot page | `bots.APRSPH` |

A parity test invokes each capability through the terminal command layer and the
HTTP handler, then compares the resulting rows and the returned view model. If
the two surfaces diverge, the test fails before a user finds out.

## Limits and failure modes

| Condition | Behaviour |
| --- | --- |
| APRS-IS unreachable | Outbox grows; sends display "queued"; alert at depth > 16 or oldest > 5 minutes |
| Queue full (64) | New sends are refused with a visible error; drop is counted |
| Send rate exceeded | Queued, delivered on the next ten-second slot |
| ACK for an unknown message | Counted and logged; no row created |
| Duplicate inbound message | Stored as received; ACK deduped within a short window so retries are still acknowledged |
| Duplicate ACK | Idempotent; no second state transition |
| Malformed packet | Counted, logged at debug with a redacted line; never crashes the receiver |
| Lease lost | Stop sending and acknowledging; readiness false; replacement takes over |
| Clock jump | Timestamps use the application clock in UTC; the node runs NTP; relative ages are rendered from stored instants |
| Login storm | Beacons are paced per account; message sends are paced globally; neither can be driven faster by repeated logins |
| Disk full | Writes fail; outbox stalls; readiness false; alert |

## Security and abuse

APRS-IS is unauthenticated. Anyone can send a packet claiming any sender, and
the BBS derives a passcode from a callsign it does not own. These are properties
of APRS, not defects in the BBS; the design's job is to bound the damage.

- **No privilege from an APRS claim.** An inbound message never changes a role,
  an email address or a password. The only effect of an inbound packet is a row
  in an inbox, a bulletin row, or an ACK.
- **Outbound is bounded.** Queue cap, one send per ten seconds, ACK budget, and
  a hard limit on retries. A flood of inbound packets cannot become a flood of
  outbound connections.
- **Text is untrusted.** Sanitise on write and on terminal render. A bulletin is
  third-party content rendered into other users' terminals; it gets the same
  treatment as any other user text.
- **No secrets in APRS.** Passcodes are derived, never stored or logged. Message
  bodies are not written to logs at info level.
- **Callsign validation.** A message is only sent if the callsign passes the
  same validator used at registration.

## Testing

- **Codec tests** with golden packets: message, ACK, bulletin, multipart,
  trailing message id, malformed lines, non-ASCII.
- **Split property**: for every length up to the maximum, joining the parts
  reproduces the input, and the part count matches the preview.
- **ACK matching**: correct part, correct SSID, wrong id, duplicate ACK, ACK for
  another station.
- **Queue**: one send per window under a burst; cap enforced; statuses correct.
- **Lease**: two processes, only one sends; takeover after expiry.
- **Fake APRS-IS**: a local TCP server that accepts the login, records outbound
  lines and can be told to disconnect. No test touches the real APRS-IS.
- **Parity**: every row of the matrix, through both adapters.
- **Load**: 1 000 inbound messages a minute for ten minutes, with the outbox
  bounded and memory flat.

## Observability

Metrics: `aprs_connected`, `aprs_packets_in_total{kind}`,
`aprs_packets_dropped_total{reason}`, `aprs_outbox_depth`, `aprs_outbox_oldest_age_seconds`,
`aprs_sent_total{status}`, `aprs_ack_sent_total`, `aprs_ack_matched_total`,
`aprs_bulletins_stored_total`, `aprs_bulletin_unknown_group_total`.

Alerts: APRS-IS disconnected for more than five minutes; outbox oldest older
than five minutes; outbox depth above 32; lease held by nobody while the
deployment is ready; failed-send rate above 10% over fifteen minutes.

## Open questions

1. **Over-the-air character set.** APRS message text is conventionally ASCII.
   Should non-ASCII input be transliterated, replaced, or refused? The current
   behaviour should be measured before deciding.
2. **Cross-sender bulletin dedupe.** The live data has 253 exact-text
   duplicates from different senders. Collapsing them hides relays; keeping them
   clutters the list. A `content_key` is computed so the policy can be chosen
   later without another migration.
3. **Accounts and SSIDs.** Accounts are base callsigns, but users operate
   `CALL-7`, `CALL-14`. Should an account be allowed to claim SSIDs, and should
   inbox matching treat `CALL` and `CALL-14` as one identity or two?
4. **ACK for disabled accounts.** Today an inbound message for a disabled
   account is not accepted, so it is not acknowledged. Confirm that is intended.
5. **AX.25 / RF.** Not in scope. If an RF igate is added later, it becomes
   another ingest source into the same store, not another APRS implementation.
