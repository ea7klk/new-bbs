# 2. Requirements

These are the requirements the successor is designed against. They are derived
from the current system's contract ([01-current-state.md](01-current-state.md)),
the operational constraints of the cluster, and the owner's stated goals.

`Must` means a release is not acceptable without it. `Should` means it is
expected, and its absence needs a decision. `May` is optional.

## Functional

### Transports

- **Must** accept SSH sessions on the existing port, so no caller has to change
  a saved alias or an SSH config entry.
- **Must** accept telnet sessions, with a caller-visible experience equivalent to
  SSH: same login, same menus, same boards, same APRS.
- **Must** serve a browser interface over HTTPS for the same accounts and data.
- **Should** keep the browser interface usable without JavaScript, with
  progressive enhancement for live updates.
- **May** accept WebSocket terminal sessions later, for browser-based terminals
  such as VTX. This is not a substitute for the browser interface.

### Accounts

- **Must** keep the callsign as the account key and accept existing passwords.
- **Must** keep registration, email activation, password reset and the
  double-confirmed email change, with the same expiry windows.
- **Must** keep the four languages (en, es, fr, de) on every surface.
- **Must** keep the profile fields: full name, email, locator, QTH, rig,
  language, `Enable APRS`.
- **Must** keep sysop administration: promote/demote, disable/enable, delete,
  publish bulletins, manage boards, edit/delete messages.
- **Should** replace the "first claimant of `BBS_SYSOPS`" bootstrap with an
  operator-run grant that is auditable and cannot be raced.
- **Should** record a moderation audit trail (who deleted or edited what, when).
- **May** support APRS-IS passcode possession as a lightweight proof of callsign
  ownership, as an alternative to trust-on-first-use.

### Boards and messages

- **Must** keep boards, threaded messages, the five-level reply cap, the
  500-top-level-message-per-board budget, and the same sanitising rule.
- **Must** apply the same rule on every surface. A reply that is too deep is
  refused identically over SSH, telnet and HTTP.
- **Should** add full-text search across boards and bulletins.
- **Should** add soft delete with an audit record instead of destroying rows.

### APRS

- **Must** keep every behaviour listed under "APRS" in
  [01-current-state.md](01-current-state.md), including SSID and ACK rules and
  the login position beacon.
- **Must** expose APRS on the terminal and the web with the same capabilities.
- **Must** have exactly one process that talks to APRS-IS and exactly one that
  sends acknowledgements.
- **Must** queue outbound messages and never claim "sent" before APRS-IS has
  accepted the packet.
- **Must** keep the retention caps configurable, defaulting to 200 sent and 500
  received per account.
- **Should** expose queue depth, delivery state and APRS-IS connection state to
  the operator.
- **May** add an AX.25/RF gateway later. It is out of scope for this design.

### Operations

- **Must** be deployable by Fleet from GitOps, without manual steps that drift
  correction would revert.
- **Must** run on the existing ReadWriteOnce local-path storage.
- **Must** start without SMTP configured, exactly as the current system does.
- **Must** keep APRS reception running across redeploys of the interactive
  service.
- **Must** expose health and readiness endpoints that distinguish "process
  alive" from "database ready" and "schema current".
- **Should** expose Prometheus metrics and structured logs.

## Non-functional

### Correctness

- One implementation of each domain rule, in one package, called by every
  adapter. A rule must not exist in a handler.
- One writer per aggregate. Interactive writes and APRS writes must not
  interleave into a lost update.
- Every write is transactional and reports failure. A success response means the
  row exists.

### Performance

Targets, measured at the 95th percentile on the deployed node:

- Terminal login prompt within 300 ms at 1 000 accounts and 1 s at 5 000.
- Board list and thread view within 150 ms.
- Web page render within 300 ms for a normal page.
- APRS message accepted into the outbox within 100 ms; on-air within 30 s when
  APRS-IS is healthy.
- No per-connection work proportional to the number of accounts.

The current per-connection startup is the anti-target: it must not survive.

### Security

- Telnet is plaintext. It must be reachable only from HamNet and the local
  network, never from the public internet.
- Passwords are never logged, never emailed, and never stored in a reversible
  form. The PBKDF2 parameters are recorded with the hash.
- Session tokens are stored hashed, with an expiry and a revocation path.
- APRS passcodes are derived from the callsign when needed and never persisted.
- All user-supplied text is sanitised at the boundary and again at render time
  for the terminal.
- CSRF protection on every state-changing web request.
- Rate limits are shared across surfaces and survive a process restart.

### Accessibility

- The web interface keeps the current contrast and keyboard-navigation
  standard. Every colour pair used for body text clears WCAG AA.
- The terminal interface works at 80×24 and 132×24, and degrades to line mode
  for clients that cannot do cursor addressing.

### Operability

- Two Deployments: `bbs` (interactive) and `bbs-aprs` (reception and delivery).
- One migration step, run before the new pods serve traffic, idempotent and
  safe to retry.
- A rollback that does not depend on rebuilding an image.
- A backup of the SQLite file that is consistent (checkpointed or copied with
  the database closed).

## Constraints

- **HamNet is `44.0.0.0/8` behind a tunnel that must not carry the default
  route.** The current host has no `44/8` address yet and the web router matches
  only `bbs.ea7klk.es`. Any HamNet-only surface needs its own router and an
  IP allow-list, per `simple-ham-bbs/HAMNET.md`.
- **Telnet cannot be encrypted.** Its security comes from where it is reachable,
  not from the protocol.
- **APRS-IS is unauthenticated and public.** Anyone can inject packets; anyone
  can spoof a sender. The design must not let inbound traffic cause unbounded
  outbound work.
- **The cluster uses local-path storage.** A ReadWriteOnce claim pins every pod
  that mounts it to one node; there is no shared filesystem.
- **The existing database is SQLite.** Moving to PostgreSQL is a separate
  decision with its own migration, and is not required at the current size.
- **The owner's time is the scarce resource.** The design should minimise the
  amount of new surface that has to be maintained.

## Non-goals

- No FTN (FidoNet-style) message networks, QWK packets, file bases, doors, ANSI
  art packs or multi-node chat. These are what a classic BBS engine buys; they
  are not part of this product.
- No RF/AX.25 transport in this design. APRS-IS over the internet only.
- No multi-tenant operation. One BBS, one callsign namespace.
- No native mobile application. The web interface is responsive; that is all.
- No replacement of the APRS archive (`ea7klk/aprs-archive`) or the ELMER
  project. They are separate systems.

## Success criteria

The replacement is successful when, one week after cutover:

1. Every account that existed before can sign in, and no password was reset.
2. Every message and APRS row is present and attributed correctly.
3. SSH, telnet and web each complete the same scripted journey: log in, read a
   board, post a reply, read APRS inbox, send an APRS message, read bulletins.
4. The APRS service has been restarted at least once without interrupting the
   interactive service, and reception resumed.
5. No rule exists in more than one package, demonstrated by the parity test
   suite running every operation through both adapters.
6. Rollback has been rehearsed on a copy, with a measured restore time.
