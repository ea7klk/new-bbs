# 7. Roadmap

The phases below are sequenced so that the riskiest thing — the data migration —
happens after the new system can already serve a real session, and so that the
old system keeps running until the new one has proven itself.

Effort is expressed in focused engineering weeks. It is a judgement, not a
measurement, and it assumes one engineer familiar with the codebase. End to end
the phases total **10–12 weeks**, of which **9–11 weeks** bring the system to
first cutover; Phase 6 is the decommission that follows the soak.

## Phase 0 — Decide (1 week)

**Deliverable:** this repository, plus a decision on open questions 1–3.

**Exit criteria:**

- The owner has chosen between the recommended architecture and the ENiGMA½
  alternative.
- The telnet exposure policy is agreed (HamNet/LAN only, or not at all).
- The identity policy is agreed (unverified callsigns remain acceptable, or a
  verification step is added).

**Why first:** the choice between "core plus adapters" and "engine plus mods"
changes every later phase. Building both is waste.

## Phase 1 — Walking skeleton (2–3 weeks)

**Deliverable:** one binary that serves SSH, telnet and HTTP from one process,
against the new schema, with accounts, boards, messages and the migration
command.

**Scope:**

- `internal/domain`, `internal/store`, numbered migrations.
- In-process SSH with host keys and pty handling.
- Telnet negotiation and the shared screen model.
- HTTP login, board list, thread view, post and reply.
- `bbsd admin migrate` and `verify-migration`.
- Parity test harness for the first three operations.

**Exit criteria:**

- An existing account logs in over all three transports with its existing
  password.
- A message posted over telnet is visible over SSH and the web.
- The reply-depth cap is enforced identically on all three.
- A copy of the live database migrates and verifies.
- The login prompt is under 300 ms with 1 000 generated accounts.

**What is deliberately missing:** APRS, bulletins, sysop tools, email, i18n,
soft delete, search. They come next, on a system that already works.

## Phase 2 — Accounts, mail and sysop (2 weeks)

**Deliverable:** feature parity for everything that is not APRS.

**Scope:**

- Registration, activation, reset, double-confirmed email change.
- Shared rate limits across transports.
- Sysop user administration and the CLI sysop grant.
- Bulletins and the station directory.
- The four-language catalogue wired into terminal and web.
- Soft delete and the audit log.

**Exit criteria:**

- The account flows work with and without SMTP, exactly as today.
- A sysop can disable an account and that account cannot log in on any
  transport.
- Every visible string has four translations, enforced by a test.
- A deleted message is hidden but recoverable, and the audit row names the actor.

## Phase 3 — Web consolidation (1–2 weeks)

**Deliverable:** `ham-bbs-web` retired; the web surface served by `bbsd serve`.

**Scope:**

- Port the existing templates and stylesheet.
- Sessions in the database, CSRF, security headers.
- `/api/v1` for every page operation; SSE for live updates.
- Redirects for any changed route.

**Exit criteria:**

- The existing web pages are reproduced with the same visual design.
- Every page action has an API equivalent, exercised by the UI.
- There is no second implementation of any rule: a review of `internal/http`
  finds no policy code.
- Old URLs either work or redirect.

## Phase 4 — APRS service (2 weeks)

**Deliverable:** `bbsd aprs` with ingest, outbox, sender, ACK matcher, bots,
bulletins and retention; the terminal and web APRS surfaces.

**Scope:**

- Port the receiver, ACK worker, bots and bulletin classifier into
  `internal/aprs`.
- Outbox schema and state machine; one sender; lease.
- Terminal screens: send, inbox, sent, bulletins, nets, settings.
- Web pages and API endpoints, with the parity matrix from
  [06-aprs.md](06-aprs.md).
- Fake APRS-IS test server; golden-packet tests; load test.

**Exit criteria:**

- Sends leave with SSID 0 and ACKs with the addressed SSID, verified against a
  fake server.
- The queue cap, the ten-second pacing and the drop accounting are tested.
- A second process cannot acquire the lease while the first is healthy.
- Rendering the inbox over SSH, telnet and web produces the same content.
- Nothing in `internal/http` or `internal/term` opens a socket to APRS-IS.

## Phase 5 — Rehearsal and cutover (1 week, plus soak)

**Deliverable:** the live system running on `new-bbs`.

**Scope:**

- Execute [05-migration.md](05-migration.md) M1–M4.
- Run the smoke matrix on all three surfaces.
- Watch the metrics for one hour, then seven days.

**Exit criteria:**

- `verify-migration` exits 0; the report is archived.
- All twelve smoke actions pass on all three transports.
- SSH host key fingerprints are unchanged.
- Rollback has been rehearsed, including the restore time.
- After seven days: no rollback, no data-loss report, no unexplained error rate.

## Phase 6 — Decommission (1 week)

**Deliverable:** the old system retired without losing anything.

**Scope:**

- Archive `simple-ham-bbs` and `ham-bbs-web` with a pointer to `new-bbs`.
- Remove the old Fleet bundles and the `bbs-web` GitRepo.
- Archive the old claim; delete it after the archive is verified.
- Update DNS, README, and the operator runbook.
- Keep the SSH host and port unchanged so no caller has to act.

**Exit criteria:**

- The old Deployments no longer exist.
- The old claim exists only as an archive.
- A new caller can follow the documentation and connect over SSH, telnet (on
  HamNet) and the web.

## Order of risk

| Risk | Phase where it is retired |
| --- | --- |
| A rule cannot be shared between surfaces | Phase 1 (parity harness) |
| Telnet cannot be made safe to expose | Phase 1 (NetworkPolicy and external test) |
| The migration loses or mangles data | Phase 1 (copy of live data) and Phase 5 |
| APRS regresses | Phase 4 (golden packets, fake server) and Phase 5 |
| The web loses its design or accessibility | Phase 3 (side-by-side comparison) |
| Rollback does not work | Phase 5 (rehearsed restore) |

## What could stretch the schedule

- A decision to add callsign verification (new product surface, new mail flows).
- A decision to support SSIDs as first-class accounts.
- A move to PostgreSQL before cutover.
- Any attempt to keep `ham-bbs-web` alive as a second writer during the soak.
  Do not do this: it reintroduces the exact defect the design removes.
