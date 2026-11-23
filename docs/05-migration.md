# 5. Migration

This document specifies how the existing data moves from `hamnet-bbs` to
`new-bbs`, how the cutover is performed, how it is verified, and how it is
undone.

The guiding rule: **the old database is never modified during the migration.**
The migration runs against a copy on a new claim. The old claim stays intact
until the new system has soaked.

## What is being migrated

From the live database on 2026-11-23 (see
[01-current-state.md](01-current-state.md)):

| Old table | Rows | New table |
| --- | --- | --- |
| `db_users` | 7 | `users` |
| `db_boards` | 4 | `boards` |
| `db_messages` | 5 | `messages` |
| `db_bulletins` | 1 | `bulletins` |
| `db_aprs_sents` | 125 | `aprs_sent` |
| `db_aprs_sent_parts` | 141 | `aprs_sent_part` |
| `db_aprs_receiveds` | 667 | `aprs_received` |
| `db_aprs_bulletins` | 5 138 | `aprs_bulletin` |
| `db_activations` | 0 | `activations` |
| `db_tokens` | 0 | `auth_tokens` |
| `db_registrations` | not counted | `registrations` |
| `db_auth_failures` | 0 | `auth_attempts` |
| `db_settings` | 2 | `settings` |
| `db_meta` | 0 | dropped |

Small numbers are not a reason to be casual. The same procedure has to work
when the tables are a thousand times larger, and the verification has to prove
that it did.

## Principles

1. **Lossless by default.** Every row is either carried or explicitly reported
   as dropped, with a reason. No silent coercion.
2. **Deterministic.** The same input produces the same output. Ordering,
   timestamps and identifiers are stable.
3. **Idempotent.** A failed migration can be re-run from the copy without
   cleaning up by hand.
4. **Verifiable.** The migration emits a report with counts, checksums and a
   list of anomalies. Verification is a separate command that reads both files.
5. **Reversible.** Until the new system takes its first write, rollback is a
   service switch. After that, `export-legacy` bounds the loss.

## Field rules

### Timestamps

The old columns store strings like `2026-07-08 14:24 UTC`. Parse with the layout
`2006-01-02 15:04 MST`, require the zone to be UTC, and store RFC 3339 in UTC
(`2026-07-08T14:24:00Z`). A value that does not parse is an error, not a zero
time. The report lists every failure.

`registered` is write-once and is carried as-is. `first_seen` and `last_seen`
keep their meaning. No timestamp is regenerated from "now".

### Callsigns

Upper-case and trim. Validate with the same rule the current application uses.
The primary key means duplicates cannot exist; a case-only duplicate would have
been impossible in SQLite's default binary collation, so no merge step is
needed. Report any callsign that fails validation rather than dropping it.

### Board identifiers

Keep the slug exactly, including `informes-de-error-espa-ol`. It is an
identifier, not a display string; the lost ñ is a historical artifact. Fix the
display name only if the owner wants to. Renaming the slug would break every
message row, saved link and bookmark.

### Message identifiers

Preserve `id` by inserting explicitly and then advancing `sqlite_sequence`. The
current web URLs use message ids; preserving them keeps old links working and
makes the verification a direct comparison. `thread_id` is the root ancestor,
and `depth` is the number of edges from the root, both computed from
`parent_id`.

Anomalies and their handling:

| Anomaly | Handling |
| --- | --- |
| `parent_id` points to a missing row | Import as top-level, record the original parent in the report |
| Parent is on a different board | Import as top-level, record it |
| Cycle in `parent_id` | Refuse the migration and report the cycle; do not guess |
| Depth greater than five | Import with its true depth, flag the thread for sysop review; do not re-parent silently |

The last case should not exist, because the current application caps depth at
five. If it does, it is evidence of a bug or a manual edit and deserves a human
decision, not an automatic one.

### APRS rows

- `db_aprs_sents.passcode` is **not migrated**. The passcode is derived from the
  callsign; storing it per message serves no purpose and widens the blast radius
  of a database leak.
- Sent rows keep `id`; parts keep their `sent_id` and `number`. The
  `message_id` on parts is the identifier used for ACK matching and is carried
  unchanged.
- `status` and `acked` are carried. A message that was `failed` stays failed;
  the new system does not retry history.
- Received rows keep `id`, `user_callsign`, `at`, `from`, `to`, `text`, `raw`.
- Bulletin rows keep `id` and the existing `dedupe_key`, so identity is stable.
  A **new** nullable `content_key` is computed (sender-normalised, group,
  normalised text with the trailing `{XXXXX` removed, whitespace collapsed).
  It is not unique. It exists so a later policy can decide whether two stations
  relaying the same bulletin should collapse. The 253 exact-text duplicates in
  the current data are recorded in the report, not merged during migration.

### Accounts and auth

- Password hashes are carried byte-for-byte, with the algorithm and parameters
  parsed out. A hash that does not match
  `pbkdf2_sha256$<iterations>$<salt>$<digest>` is an error for that account, not
  a reason to reset everyone.
- All web sessions are invalidated. There are none in the current data, but the
  rule must hold on the day.
- All `db_tokens` rows are invalidated at cutover. There are none today. If
  there were, a reset link minted by the old system should not be honoured by
  the new one; the user requests a new link.
- `db_activations` rows are carried if present, because an unactivated account
  should not have to re-register. The current table is empty, but VE7TCA is
  unactivated, so the flow must still work.
- `db_auth_failures` rows are carried only if their window is still open;
  otherwise they are dropped. Carrying an expired lockout would deny a login for
  no reason.
- `db_registrations` rows are carried with their window start; the meter is
  short-lived and harmless either way.
- The `BBS_SYSOPS` environment variable is not carried; it was a bootstrap, not
  state. The role itself is a column: the importer sets it from the existing
  `is_sysop` value and records an audit row naming the migration as the actor.
  The operator may then re-grant it explicitly with
  `bbsd admin grant-sysop EA7KLK`, which is the supported path for any future
  change.

### Settings

`schema.version` and `migrated.activation` are old-schema bookkeeping and are
not carried. Other `db_settings` rows are carried as key/value pairs. The new
schema has its own `schema_migrations` table.

## Tooling

Three commands, all in `bbsd admin`:

- `migrate --from <old.sqlite> --to <new.sqlite> --dry-run` — reads the old file
  read-only, writes nothing, prints the plan: row counts, anomalies, derived
  values, expected duration.
- `migrate --from ... --to ... --report <file.json>` — performs the migration in
  one transaction per table, writes the report, refuses to leave a partial
  database.
- `verify-migration --old ... --new ... --report <file.json>` — independent
  check: counts, checksums, referential integrity, thread depth, APRS part
  coverage, hash prefixes, timestamp ranges. Exits non-zero on any mismatch.

A fourth command exists for rollback:

- `export-legacy --since <timestamp> --to <dir>` — writes new-system rows that
  did not exist at cutover in the old schema, as CSV or as a SQLite file, so
  they can be loaded into the old system if a rollback happens after new writes.

`migrate` never deletes or modifies the source file. It opens it with
`?mode=ro` and, if the file is in WAL mode, requires that a `.backup` copy was
taken first or takes one itself into a temporary file.

## Cutover

A blue/green cutover on a new claim. The old Deployment and old claim stay in
place, scaled to zero, until the soak ends.

### M0 — Announce and prepare (T−7 days)

- Announce the window and the new telnet address.
- Confirm the current image digest and the old Deployment specs are recorded.
- Confirm the SSH host keys are available outside the image, so fingerprints do
  not change.
- Verify the backup Job has run and the restore has been rehearsed on a copy.

### M1 — Rehearse (T−2 days)

1. Take an online backup of the live database with
   `sqlite3 bbs.sqlite ".backup /tmp/rehearsal.sqlite"`, then
   `PRAGMA integrity_check`.
2. Create a staging namespace and a new claim; copy the backup in.
3. Run `migrate --dry-run`, then `migrate`, then `verify-migration`.
4. Run the smoke matrix (below) against a staging `bbsd serve`.
5. Record the migration duration and the verification output. If duration is
   over ten minutes, adjust the window.

### M2 — Freeze (T−0)

1. Scale `bbs`, `bbs-aprs` and `bbs-web` to zero. This stops APRS reception;
   the gap is the cutover window and is announced.
2. Take the final backup from the closed database (`.backup` is safe, but a
   closed database is simpler and faster).
3. Run `PRAGMA integrity_check` and record the SHA-256 of the backup.
4. Copy the backup to the new claim.

### M3 — Migrate and start

1. Run the migration Job against the new claim.
2. Run `verify-migration`; require exit 0 before continuing.
3. Deploy the new workloads as `bbs-next` and `bbs-aprs-next`, both mounting the
   new claim. The old Deployments stay at zero replicas and keep the old claim.
4. Start `bbsd aprs`; confirm readiness and that it holds the lease.
5. Start `bbsd serve`; confirm SSH, telnet and HTTP readiness.
6. Copy the SSH host keys into the new pods and confirm the fingerprint matches
   the old one.
7. Switch the `bbs` Service selector to `bbs-next` (and the metrics selector to
   `bbs-aprs-next`). The Service names and ports do not change, so callers see
   no difference.

Renaming `bbs-next` back to `bbs` happens after the soak, not during the
cutover. Until then the old Deployment is one selector change away, and the
Traefik routes keep pointing at the same Service, so callers keep using the same
host and port.

### M4 — Verify (T+0 to T+1 hour)

Run the smoke matrix on all three surfaces. Compare the verification report with
the M1 rehearsal. Watch the error rate, the APRS outbox and the database busy
counter for one hour.

### M5 — Soak (7 days)

- Keep the old claim and the old image digest.
- Keep `export-legacy` available.
- Keep the old Deployments at zero replicas, not deleted.
- After seven days with no rollback, take a final archive of the old claim.

### M6 — Decommission

- Archive `simple-ham-bbs` and `ham-bbs-web` (or leave them read-only) and note
  the successor in their READMEs.
- Delete the old claim after the archive is verified.
- Remove the old Fleet bundles and the `bbs-web` GitRepo.
- Update DNS and documentation; keep the old SSH alias working (same host and
  port).

## Smoke matrix

Every cell is a scripted, repeatable action, not a human impression.

| Action | SSH | Telnet | Web |
| --- | --- | --- | --- |
| Log in with an existing password | required | required | required |
| Wrong password is refused and counted | required | required | required |
| Board list matches | required | required | required |
| Open a message and see its author and timestamp | required | required | required |
| Post a top-level message | required | required | required |
| Reply five deep is accepted; sixth is refused | required | required | required |
| Read APRS inbox | required | required | required |
| Send an APRS message; it appears as queued | required | required | required |
| Read bulletins by category | required | required | required |
| Switch language and see translated UI | required | required | required |
| Sysop can disable and re-enable an account | required | required | required |

## Verification queries

These run inside `verify-migration` and are also useful by hand.

- Row counts per table equal, except for documented drops.
- `messages`: no `parent_id` without a parent; `thread_id` not null; `depth`
  equals the computed depth for every row.
- `aprs_sent_part`: every `sent_id` exists; the number of parts per sent matches
  the old data.
- `users`: count equal; every `password_hash` starts with `pbkdf2_sha256$`; the
  language distribution matches; `registered` is not blank.
- `boards`: the four ids match exactly.
- `aprs_bulletin`: count equal; `dedupe_key` set equal; `content_key` non-null.
- Timestamps: minimum and maximum of each timestamp column equal the old values
  after conversion.
- Checksum: for each table, hash a deterministic concatenation of its rows in
  primary-key order, excluding derived columns; compare old and new.

The checksum is the backstop. If a count matches but a body was truncated, the
checksum catches it.

## Rollback

Rollback is a first-class path, not an apology.

**Before the new system takes its first write:** switch the Service selector
back to the old Deployment and the old claim. The old database is untouched.
This takes minutes and is the expected path for a failed verification.

**After the first write:** the old claim does not contain the new rows. Two
options, in order of preference:

1. Run `export-legacy --since <cutover>` and load the delta into the old
   database, then switch back. This preserves posts made during the soak.
2. Accept the loss of the delta, bounded by the rollback decision deadline.

Set the deadline explicitly: 24 hours after cutover, the decision is either
"continue" or "export and roll back". After seven days, the old claim is
archived and rollback is no longer on the table.

## Risks

| Risk | Mitigation |
| --- | --- |
| Copying a WAL database without checkpointing loses recent rows | Always use `.backup` or `VACUUM INTO`; never `cp` the main file alone |
| New claim has the wrong ownership | Init container sets ownership and mode; the app refuses to start if it cannot write |
| SSH host key changes and clients see a MITM warning | Copy the host keys before starting; verify the fingerprint |
| Telnet is exposed publicly by a hostPort | NetworkPolicy plus an external connection test; prefer a HamNet-only router |
| APRS reception gap during freeze | Schedule outside known nets; keep the window under ten minutes |
| Old web URLs 404 | Preserve message ids and board slugs; add redirects for changed routes |
| A message body is truncated by a column limit | Compare lengths and checksums; the new schema has no shorter limits |
| An unactivated account cannot activate | Carry activation rows; test the flow with a staging account |
| Rollback after new writes loses posts | `export-legacy` plus a 24-hour decision deadline |

## Appendix: if ENiGMA½ is adopted

This is the mapping for the alternative in
[03-options.md](03-options.md#the-alternative-that-is-actually-competitive). It
is a sketch, not a specification; it exists so the cost is concrete.

- **Accounts** — `db_users.callsign` becomes `user.user_name`. Profile fields
  become `user_property` rows: `real_name`, `location` (QTH), `email_address`,
  plus custom properties for language, Maidenhead locator, rig and
  `enable_aprs`. `db_users.is_sysop` becomes membership of a sysop group.
- **Passwords** — ENiGMA stores `pw_pbkdf2_salt`, `pw_pbkdf2_dk` and
  `pw_hash_params`. The current hashes are PBKDF2-SHA256 at 200 000 iterations,
  so an import is theoretically possible by setting the parameters to match.
  It depends on the exact encoding of salt and derived key. Treat it as a
  research spike with a forced reset as the fallback: never migrate a hash
  format you have not verified against a known password.
- **Boards** — each board becomes an area under a `local` conference. Area tags
  must be snake_case: `informes-de-error-espa-ol` becomes
  `informes_de_error_espa_ol`. The display name is preserved.
- **Messages** — insert into `message` with `area_tag`, `from_user_name`,
  `to_user_name = 'All'`, `subject`, `message`, `reply_to_message_id` and
  `modified_timestamp`. ENiGMA assigns its own autoincrement ids, so old message
  URLs break unless a `legacy_id` is written to `message_meta` and a redirect
  route is added. The five-level reply cap is not an ENiGMA rule; the importer
  must validate it and a mod must enforce it afterwards.
- **Bulletins** — ENiGMA bulletins are normally art files, not database rows.
  A dynamic bulletin list needs a mod or a dedicated area.
- **APRS** — not present. The existing Go receiver and ACK worker become a
  service; the terminal screens become a mod with its own menu and art. The web
  surface is a web handler or an external front end over `/_enig/api/v1/`.
- **Localisation** — ENiGMA themes carry their own strings. Four languages means
  four theme/art variants or a mod that resolves strings per user. The current
  `translations.json` does not map one-to-one.
- **Operations** — a second SQLite database (ENiGMA's), a second schema, a
  second backup, and a Node.js runtime beside the Go APRS service.

The mapping is feasible. It is simply more work than evolving the schema we
already own, and it leaves two runtimes to operate.
