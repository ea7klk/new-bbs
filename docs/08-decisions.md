# 8. Decisions

Architecture decision records for the choices that shape `new-bbs`. Each records
the context, the decision, the consequences, and the alternative that was
rejected.

---

## ADR-001 — No wholesale BBS engine adoption

**Status:** proposed

**Context.** ENiGMA½ and Synchronet are credible terminal BBS packages with SSH,
telnet and web access. Neither carries APRS, callsign identity, the four-language
catalogue, or the current data model. Adopting one means importing users and
threads into a foreign schema and rebuilding the ham-specific layer as a mod.

**Decision.** Build `new-bbs` as one domain core with SSH, telnet and HTTP
adapters, reusing the existing rules and APRS code. Treat engine adoption as a
documented alternative, not the default.

**Consequences.**

- Every existing rule survives without translation.
- The team owns the terminal UX and the BBS engine, not only the ham layer.
- Classic BBS features (FTN, doors, file bases, ANSI art) are not available.
- The decision is reversible: if scene presence becomes the priority, the
  ENiGMA path in `03-options.md` remains open, and the APRS service would survive
  the move.

**Rejected alternative.** ENiGMA½ as the terminal engine. It scores well on
transport and ecosystem health, but its web tier is an API plus extension
points, not a BBS UI, and the APRS screens, identity rules and localisation all
have to be rebuilt anyway.

---

## ADR-002 — One interactive process, APRS separate

**Status:** proposed

**Context.** Today every SSH session is a process, paying O(accounts) startup
work. APRS reception was already separated because redeploying the SSH container
interrupted it.

**Decision.** `bbsd serve` handles all SSH and telnet sessions and HTTP requests
in one process. `bbsd aprs` handles APRS-IS, the outbox and acknowledgements in a
second.

**Consequences.**

- Startup work happens once; a session is a goroutine.
- A crash in the interactive process affects every session; mitigated by
  per-session recover and a small process.
- APRS survives interactive redeploys.
- Two Deployments, two schemas' worth of responsibility, one database.

**Rejected alternative.** One process for everything. Simpler to deploy, but it
re-couples APRS availability to SSH rollouts and makes a web deploy interrupt
reception.

**Rejected alternative.** A process per surface (SSH, telnet, web). More blast
radius containment, but three writers and cross-process session state at a scale
that does not need it. It remains the scaling path in ADR-003.

---

## ADR-003 — SQLite now, PostgreSQL only on evidence

**Status:** proposed

**Context.** The live database is 3.1 MB with seven accounts. The measured limit
is write concurrency, not size: SQLite permits one writer, and the current design
already queues writers with `_txlock=immediate` and a busy timeout.

**Decision.** Keep SQLite in WAL mode, with one write pool of one connection and
a separate read pool. Revisit when measured write latency or busy retries cross
the target in `02-requirements.md`, or when multiple nodes are required.

**Consequences.**

- No database server to operate; backup is a file.
- A single node is a hard constraint while the claim is ReadWriteOnce.
- The store layer must not leak SQLite-specific SQL into the domain, so a later
  move is a store change rather than a rewrite.

**Rejected alternative.** PostgreSQL now. It solves a problem that is not
present at seven accounts, and adds an operator surface, a backup story and a
migration before the first cutover.

---

## ADR-004 — The callsign stays the identity, and the sysop grant moves to the CLI

**Status:** proposed

**Context.** Accounts are unverified pseudonyms. Anyone can claim an unclaimed
callsign. `BBS_SYSOPS` grants the role to the first claimant of a listed
callsign, which is a race and a privilege-escalation path (finding H1).

**Decision.** Keep the callsign as the account key. Replace the bootstrap with
`bbsd admin grant-sysop <callsign>`, which writes an audit row. Keep an optional
development token, clearly marked unsafe.

**Consequences.**

- No window in which a caller can become a sysop by being first.
- The operator must run one command after cutover (or the importer sets the flag
  from `is_sysop` and records the migration as the actor).
- Callsign ownership remains unverified. That is a product decision, recorded as
  open question 3.

**Rejected alternative.** Email-domain or document verification at registration.
It adds a gate that the community has not asked for and would exclude users
without a callsign-linked address.

---

## ADR-005 — APRS protocol behaviour is frozen

**Status:** proposed

**Context.** SSID rules, ACK addressing, splitting and bot command strings are
hard-won and visible on the air. A rewrite that "improves" them breaks other
stations.

**Decision.** Port the existing behaviour exactly. Any change to a protocol
behaviour requires its own ADR, a test that demonstrates the old behaviour, and
a reason that outweighs interoperability.

**Consequences.**

- The new APRS service is a port, not a redesign.
- Known oddities (base SSID 0 for sends, ACK from the addressed SSID) stay.
- Improvements are limited to observability, queueing and error reporting.

**Rejected alternative.** Redesign around a single sender identity. Simpler, and
wrong: senders compare ACKs against the station they addressed.

---

## ADR-006 — Telnet is HamNet and LAN only

**Status:** proposed

**Context.** Telnet is plaintext and has no transport authentication. The
deployment is on a public host with a HamNet tunnel that carries `44.0.0.0/8`.

**Decision.** Telnet is never reachable from the public internet. Enforce it
with a NetworkPolicy and a HamNet-only ingress path; verify with a connection
attempt from outside, not by reading the manifest.

**Consequences.**

- Callers on HamNet get the classic experience; internet callers use SSH or the
  web.
- The operator must maintain the network policy and test it after changes.
- If the CNI does not enforce the policy for hostPort traffic, a node firewall
  rule is required as well.

**Rejected alternative.** Telnet on the public internet with a warning. It would
put passwords on the wire for anyone to read.

---

## ADR-007 — The HTTP contract lives in `new-bbs`

**Status:** proposed

**Context.** `ea7klk/bbs-api` contains an OpenAPI design with no implementation.
Keeping the contract in a separate repository from its only consumer creates a
coordination step with no benefit.

**Decision.** Move the contract to `api/openapi.yaml` in `new-bbs`. Archive
`bbs-api` with a pointer, preserving its design documents as inputs.

**Consequences.**

- The contract and the handlers change in one commit.
- The reference frontend described in `bbs-api` becomes the `new-bbs` web UI.
- History is preserved by linking the archived repository.

**Rejected alternative.** Keep `bbs-api` as the contract owner. It was reasonable
while there was no implementation; it is overhead once there is one.

---

## ADR-008 — Shared domain, separate rendering

**Status:** proposed

**Context.** The current web app reimplements the SSH BBS's rules because the two
are separate programs with separate rendering models.

**Decision.** Share the domain, not the view. The terminal renders a screen model;
HTTP renders templates and JSON. Both call the same domain functions and return
the same view models where the content is the same.

**Consequences.**

- A rule change is one change.
- A UI change is still two, because a terminal and a browser are different
  media. That is acceptable; a rule change is not.
- Parity tests compare effects, not pixels.

**Rejected alternative.** One renderer for both (for example, ANSI in a browser
terminal). It gives the web a terminal, not a web interface.

---

## ADR-009 — Soft delete and an audit log

**Status:** proposed

**Context.** Sysop deletes currently destroy rows. There is no record of who
deleted what.

**Decision.** Deletes set `deleted_at` and `deleted_by`. Sysop actions write an
`audit_log` row. Queries filter deleted rows by default; a sysop view can include
them.

**Consequences.**

- Moderation is reversible and attributable.
- Every query must respect the soft-delete filter; a forgotten filter is a data
  leak. This is enforced by the repository layer, not by handlers.
- Storage grows; a retention policy for deleted rows is a later decision.

**Rejected alternative.** Hard delete with an external log. The log and the data
can diverge, and a restore becomes guesswork.

---

## ADR-010 — Invalidate sessions and tokens at cutover

**Status:** proposed

**Context.** A session or reset token minted by the old system may outlive the
migration. Honouring it in the new system assumes the two implementations agree
on token format and revocation.

**Decision.** Invalidate all web sessions and all `db_tokens` rows at cutover.
Keep activation codes for unactivated accounts, because re-registration is a
worse outcome than carrying a short-lived code.

**Consequences.**

- Every web user signs in once after cutover. There are no web sessions to lose
  today, but the rule holds later.
- A pending password-reset link stops working; the user requests a new one.
- An unactivated account can still activate.

**Rejected alternative.** Carry sessions and tokens. It preserves convenience at
the cost of trusting a format across two implementations, for a user base that
can re-authenticate in seconds.

---

## ADR-011 — Pace login beacons per account

**Status:** proposed

**Context.** Today every successful login sends an APRS position beacon when
`Enable APRS` is on and a locator exists. Both the SSH app and the web app dial
APRS-IS to do it. A caller who reconnects repeatedly can put a beacon on the air
for each connection, and nothing bounds it.

**Decision.** Keep the login trigger, but enqueue the beacon and require at
least five minutes between beacons for the same account. The interval is
configurable; zero disables the pacing and restores today's behaviour.

**Consequences.**

- Repeated logins no longer produce repeated beacons.
- A user who moves and reconnects within five minutes will not update their
  position until the interval expires; the next login after that will.
- The beacon path goes through the outbox, so it is subject to the same
  connection, rate limiting and observability as everything else.

**Rejected alternative.** Keep every-login beacons. It is simpler and closer to
today, but it gives any caller a way to generate unbounded outbound traffic
under their own callsign, and it puts the beacon outside the single sender.
