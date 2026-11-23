# 1. Current state

Measured against the live deployment on **2026-11-23**. Figures marked
"measured" come from `kubectl` against the `ham-bbs` namespace; figures marked
"source" come from the repository named beside them.

## What `hamnet-bbs` is today

The deployed system is three Deployments in the `ham-bbs` namespace — two of
them running the same image — plus a shared SQLite file on a ReadWriteOnce
claim.

| Component | Role | Transport |
| --- | --- | --- |
| `bbs` | Canonical BBS: accounts, boards, bulletins, APRS send, sysop | OpenSSH on 2222 |
| `bbs-aprs` | APRS receiver, ACK worker, bulletin collector | outbound APRS-IS only |
| `bbs-web` | Server-rendered web front end over the same database | HTTP 8080 behind Traefik |

Both `bbs` and `bbs-aprs` run `ghcr.io/ea7klk/simple-ham-bbs` at the same digest.
`bbs-web` runs `ghcr.io/ea7klk/ham-bbs-web`. The two claims are 1 GiB
(`/config/ssh`, the host keys) and 2 GiB (`/var/lib/bbs`, the shared database),
`retained-local-path`, ReadWriteOnce, so every pod that mounts them is pinned to
one node.

### The SSH path is a process per caller

`sshd_config` sets `ForceCommand /usr/local/bin/bbs_session.sh`, which `exec`s
`bbs_app`. So every SSH connection is a new `bbs_app` process with its own
database handle, its own startup work and its own copy of any in-memory state.

That is the origin of the scalability finding in `simple-ham-bbs/SCALABILITY.md`:
at 10 000 accounts the login prompt takes about 4.6 s, almost all of it
per-connection housekeeping, and at 5 000 it is 2.5 s. The measurement is from a
generated database; the mechanism is visible in the deployment.

There is no telnet listener anywhere.

### The web path is a second implementation

`ham-bbs-web` opens the same SQLite file and reimplements the rules in its own
handlers: reply depth, sanitising, sysop bootstrap, token formats, the
registration meter, APRS message sections and the bot command strings. Its own
`AGENTS.md` says the SSH BBS is canonical and the web app must mirror it. That
mirroring is the cost the design has to remove.

### The APRS path is already split, correctly

`bbs-aprs` sets `BBS_SSH_ENABLED=false` and `BBS_APRS_EMBEDDED=false` and runs
`bbs_app aprs-supervisor`. A file lock on `$BBS_DATA_DIR/.aprs-reception.lock`
ensures only one receiver runs. This split exists because a redeploy of the SSH
container used to interrupt reception; the successor must preserve it.

## Data inventory

Measured row counts in the live `bbs.sqlite`:

| Table | Rows | Notes |
| --- | --- | --- |
| `db_users` | 7 | 3 languages: en (4), es (2), de (1); 1 unactivated |
| `db_boards` | 4 | `general`, `ure-sevilla`, two bug-report boards |
| `db_messages` | 5 | all top-level today; no replies |
| `db_bulletins` | 1 | sysop bulletin |
| `db_aprs_sents` | 125 | 122 `sent`, 3 `failed` |
| `db_aprs_sent_parts` | 141 | 113 single-part, 10 two-part, 2 four-part |
| `db_aprs_receiveds` | 667 | DN9RZ is at the 500-row cap |
| `db_aprs_bulletins` | 5 138 | 4 885 distinct texts; 74% `general` or `other` |
| `db_tokens` | 0 | no outstanding reset or email-change tokens |
| `db_auth_failures` | 0 | throttle state empty |
| `db_activations` | 0 | no pending activation codes |
| `db_registrations` | 0 | per-address registration meter |
| `db_settings` | 2 | `schema.version=2`, `migrated.activation=1` |
| `db_meta` | 0 | empty |

The database file is 3.1 MB with a 4.2 MB WAL. There is no performance reason to
change the storage engine at this size; the reason to consider PostgreSQL later
is write concurrency, not volume.

### What the data shape says

- **Volume is not the problem.** Seven accounts and five messages. The design
  should be judged on correctness, operability and the cost of change, not on
  throughput.
- **Retention caps are real.** DN9RZ holds exactly 500 received rows, which is
  the documented cap. The migration must not silently drop or double them.
- **Timestamps are strings**, formatted `2026-07-08 14:24 UTC`. They parse with
  the layout `2006-01-02 15:04 MST` and must become explicit UTC instants.
- **Board IDs are slugs, and one is lossy.** `informes-de-error-espa-ol` was
  produced by stripping the ñ from "Español". Keep the ID; fix only the display
  name. Renaming IDs would break links and message rows.
- **Bulletin dedupe is per sender, group and text, not per content.** There are
  253 exact-text duplicates across senders. A content-only key would collapse
  them, but it would also merge two stations relaying the same bulletin, which
  may be undesirable. Decide deliberately (see 06).
- **The bulletin taxonomy is not working.** 2 105 rows are `general` and 1 701
  are `other`; together 74% of everything received. The successor should either
  improve classification or stop presenting categories as if they were useful.

## Behaviours that are a contract

These are the rules users and other systems already rely on. The successor must
keep them unless an ADR says otherwise.

### Accounts and identity

- The callsign is the account key. Case is normalised to upper case.
- A callsign proves nothing. Anyone can claim an unclaimed one; accounts are
  unverified pseudonyms. This is accepted, and it is also finding H1 in
  `REVIEW.md`: the first claimant of a `BBS_SYSOPS` callsign becomes a sysop.
- Passwords are `pbkdf2_sha256$<iterations>$<salt>$<digest>` at 200 000
  iterations.
- Registration requires full name, email, language (en/es/fr/de), optional
  Maidenhead locator, and a BBS password.
- With SMTP configured, registration is completed by a four-character email
  code; password reset links last 2 hours; an email change needs confirmation
  from both the old and the new address within 24 hours.
- Five failed passwords from one address in fifteen minutes lock that address
  out for fifteen minutes. Five registrations per address per hour.
- `BBS_SYSOPS` is a bootstrap grant only, honoured while no active sysop exists.
- Sysops can promote, disable, delete, publish bulletins and manage boards; they
  cannot delete or disable themselves, and the last active sysop cannot be
  removed.

### Boards and messages

- Boards are ordered; new installs get `General`.
- Messages are threaded with `parent_id`; replies nest at most five deep.
- Only top-level messages count toward the 500-message per-board budget.
- Text is sanitised on write and again on read: ESC, C0/C1 controls, DEL,
  bidi/zero-width characters removed, line breaks normalised, blank runs capped.
- Sanitising exists because stored terminal escapes could reach another user's
  terminal (finding H3).

### APRS

- Sends leave from the base callsign with SSID 0, whatever the recipient's SSID.
- Acknowledgements follow the message: an ACK for a message addressed to
  `CALL-15` goes out from `CALL-15`.
- Long messages are split into numbered parts; bot commands are never split.
- The last 200 sent and 500 received messages are retained per user.
- Received messages addressed to accounts with `Enable APRS` are stored;
  message IDs appended by senders are stripped from display text but kept raw.
- Bulletins are global, deduped, categorised, kept ten days from `last_seen`,
  and are never acknowledged.
- `#APRSThursday` is `CQ HOTG` to `ANSRVR`, offered only on Thursdays;
  `Leave` sends `U HOTG`, `Extend` sends `K HOTG`.
- `APRSPH` is a bot, not a group; its commands are sent exactly as documented,
  in one message, and are never translated.
- Acknowledgements are queued, at most one every ten seconds, with a queue of 64
  and dropped sends logged rather than retried without bound.
- On login, if `Enable APRS` is true and a valid Maidenhead locator is set, a
  position beacon is sent from the base callsign with SSID 0 and the comment
  `HamNet BBS`. Beacons are not stored in the sent-message history.
- The receiver exits hourly so the APRS-IS connection is refreshed.

### Surface behaviour

- The terminal is 132×24, with a best-effort resize request and a 132×24
  assumption when the client refuses.
- Cursor keys and Enter drive menus; forms use Tab, Save/Cancel, F2 to save.
- Scripted sessions can type menu numbers and field values.
- Every user-visible string exists in English, Spanish, French and German.

## Defects the successor must not inherit

| Defect | Evidence | Consequence |
| --- | --- | --- |
| Process per SSH session | `sshd_config` `ForceCommand` | O(users) startup per connection; 4.6 s at 10 000 |
| Rules implemented twice | `ham-bbs-web/AGENTS.md` invariant 0 | Every rule change is two changes, and they drift |
| Two writers on one SQLite file | SSH app and web app both write, and both run schema backfills | Write serialisation holds only while every path uses `_txlock=immediate`; two applications own one schema |
| Rate limits in two places | SQLite in SSH, memory in web | A caller gets a different budget per surface |
| APRS send path in both apps | `handlers_aprs.go` and `aprs.go` both dial APRS-IS | Two senders and two rate limiters for messages, and a third path for login beacons, all outside the APRS service |
| Passcode persisted per message | `db_aprs_sents.passcode` | A derived value stored as if it were a secret |
| Sysop bootstrap race | H1 in `REVIEW.md` | First claimant of a listed callsign becomes sysop |
| No telnet | deployment inventory | Callers without SSH clients cannot connect |
| No audit trail for moderation | hard deletes in sysop handlers | Deleting a message destroys the evidence |
| No full-text search | schema has no FTS table | Finding an old thread means paging |

## What "replace" has to preserve

A successor is acceptable only if all of the following remain true after
cutover:

1. The seven existing accounts can sign in with their existing passwords.
2. Every board, message, bulletin and APRS row is present, with its author and
   timestamp, and thread relationships are intact.
3. The same callsign, over SSH, reaches a BBS that looks and behaves like the
   current one.
4. Telnet reaches the same boards and the same APRS functions.
5. The web reaches the same data, with no second copy of any rule.
6. APRS sends still leave with the right SSID, ACKs still match, and no message
   is acknowledged twice.
7. The four languages still work.
8. A single `kubectl` rollout can be rolled back without data loss.

The migration plan in [05-migration.md](05-migration.md) is written against this
list.
