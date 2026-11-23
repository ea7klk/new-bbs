# 3. Options considered

The brief was to investigate a BBS system that can replace hamnet-bbs with SSH,
telnet and web interfaces. This document is the survey, the comparison, and the
recommendation.

## Method

Seven scored candidates were assessed against the requirements in
[02-requirements.md](02-requirements.md). For each, the evidence is the project's
own documentation and source, read on 2026-11-23; the links are at the end.

Scoring is a judgement, not a measurement. Weights reflect this deployment's
priorities: APRS and migration cost matter more than the breadth of classic BBS
features, because the former are used daily and the latter are not used at all.

| Criterion | Weight | What it measures |
| --- | --- | --- |
| Transport | 12 | SSH, telnet and web in the box, without a bolt-on |
| Web interface | 12 | A real browser UI over the same data, not only a terminal in a browser |
| APRS fit | 18 | Extension points that can host APRS without forking the engine |
| Data and identity fit | 14 | Callsign accounts, threaded boards, per-user language |
| Migration cost | 14 | Effort and lossiness of moving the existing rows |
| Operational fit | 10 | Containers, SQLite or simple storage, single-writer model, k3s |
| Project health | 8 | Activity, documentation, release cadence |
| Licence | 4 | Freedom to run and modify without surprises |
| Capability breadth | 8 | FTN, doors, file bases, ANSI art, QWK: what an engine gives for free |

## Candidates

### ENiGMA½

Node.js BBS, BSD-2-Clause, actively developed (last push on the survey date).
It is the closest fit of any engine:

- Telnet, SSH and secure/insecure WebSocket login servers are built in.
- SQLite storage for users, message areas and file bases.
- Messaging is FTN-shaped: conferences contain areas, and messages carry
  `reply_to_message_id`, so a thread can be reconstructed.
- A built-in HTTP server with a REST API under `/_enig/api/v1/` covering message
  bases, file areas, users and system information, plus a documented
  `WebHandlerModule` extension point for custom routes.
- JavaScript mods, an event scheduler, ACS for access control, and FTS4 over
  message subject and body.

What it does not have: APRS, callsign identity, per-user language selection for
arbitrary UI strings, or a browser BBS. The web server serves a terminal client
and an API; a real web UI is something you write as a web handler. Its password
storage is PBKDF2 with per-user parameters, which makes an import of the current
hashes technically conceivable but dependent on its exact encoding.

### Synchronet

The most complete classic BBS package: C/C++ core with a JavaScript engine,
telnet and SSH (cryptlib) and RLogin services, SMB message bases with threading,
FTN networking, doors, and a bundled web interface (`webv4`) with forums, auth
and localisation.

It is also the heaviest option: many services to configure, binary message bases,
a user file rather than a relational schema, and mixed licensing across the
core and third-party components. Writing an APRS integration means writing a
Synchronet JavaScript module and an external service, against an API designed
for doors and message editing rather than modern HTTP integration.

### WWIV

C++17, Apache-2.0, actively maintained. Telnet and SSH access, binary message
bases, a long history. No bundled browser BBS; the project includes `cpp-httplib`
for HTTP endpoints, but not a web interface over boards. No APRS. The message
format is its own, so migration is a conversion project.

### Mystic

Mature Pascal BBS with telnet and SSH, a strong sysop toolchain, and a loyal
following. The maintained 1.12 line is not open source; the public repository is
a clone of 1.10a30. No REST API, no APRS, and a schema that would have to be
reached through its own tooling. A poor fit for a GitOps, API-first deployment.

### Citadel / WebCit

A groupware server with a genuine browser UI (WebCit) and telnet access. It is
not a BBS in the sense this product uses: rooms and mail rather than threaded
boards, no callsign identity, no extension point that maps cleanly to APRS, and
an ageing codebase with a protocol of its own between server and web tier.
WebCit is the best browser UI in the survey and the worst data-model fit.

### LinBPQ / FBB

Ham-specific packet BBS and node software. They speak AX.25, run store-and-forward
traffic, and have telnet and web access in some configurations. They are a
different product category: no community boards, no APRS messaging UX, no modern
API. They would be the right answer to "how do I run a packet BBS on 2 m", not
to this brief.

### Enthral, MagickaBBS, Talisman

Smaller or dormant projects. Enthral has been sunset since 2018. MagickaBBS is a
small C BBS with telnet and SSH and no web tier. Talisman could not be verified
from a public repository during the survey and was therefore not scored. None
changes the conclusion.

### Evolve the current codebase

Not a product, but it is the baseline every product has to beat: keep the Go
domain logic, replace the OpenSSH-per-session transport with an in-process
SSH and telnet server, fold the web app into the same binary, and keep APRS in a
separate service.

It is the only option that preserves every rule already encoded and tested, and
the only one with no user or message migration.

## Comparison

Scores are 1–5 per criterion; the total is the weighted sum out of 500.

| Candidate | Transport | Web | APRS | Data fit | Migration | Ops | Health | Licence | Breadth | Total |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Evolve current code | 5 | 5 | 5 | 5 | 5 | 4 | 4 | 5 | 1 | **450** |
| ENiGMA½ | 5 | 3 | 4 | 3 | 2 | 4 | 5 | 5 | 5 | **378** |
| Synchronet | 5 | 4 | 3 | 4 | 2 | 3 | 4 | 3 | 5 | **360** |
| WWIV | 4 | 1 | 2 | 3 | 2 | 3 | 3 | 4 | 4 | **268** |
| Mystic | 5 | 2 | 1 | 3 | 1 | 2 | 3 | 1 | 5 | **246** |
| Citadel / WebCit | 3 | 5 | 1 | 2 | 1 | 3 | 2 | 2 | 3 | **234** |
| LinBPQ / FBB | 2 | 2 | 3 | 1 | 1 | 2 | 3 | 3 | 4 | **218** |

### Reading the scores honestly

The baseline wins because the requirements were derived from it. That is a real
bias, and it is why the next section exists. The score is not an argument that
the current architecture is good; it is an argument that the current *domain* is
the asset and the current *delivery* is the problem. The baseline's scores
assume the delivery is fixed: in-process sessions, one rule implementation, a
telnet listener, and the web folded in. Those are the work.

Sensitivity: if "capability breadth" is weighted 25 and APRS 8 — that is, if
joining the classic BBS scene matters more than the ham-specific product — the
totals become ENiGMA 423, baseline 417, Synchronet 415. The decision genuinely
turns on that question, which is why it is open question 1.

## The decisive findings

1. **No engine carries APRS.** Every option needs an APRS service and an
   integration into the terminal UI. That service is the hard, valuable part,
   and it already exists in Go with tests.
2. **No engine carries callsign identity or the four-language catalogue.** Both
   would be rebuilt as mods or properties, with different semantics.
3. **An engine's value is the feature set this product does not use.** FTN, QWK,
   file bases, doors and ANSI art are explicitly out of scope. Paying migration
   cost for them is a poor trade.
4. **The web story is weaker than it looks.** ENiGMA's web tier is a REST API
   plus extension points: you still write the BBS UI. Synchronet's `webv4` is a
   real UI but is tied to SMB and its JavaScript stack, so the APRS and
   callsign pieces would live outside it.
5. **Adoption replaces one duplication with another.** Today the rules exist in
   the SSH app and the web app. After adoption they would exist in the engine's
   configuration and in the ham-specific mod, with a new boundary between them.
6. **The migration is not justified by volume.** Seven accounts, five messages,
   one bulletin. The cost of a foreign-schema migration is not repaid by scale.

## Recommendation

**Build `new-bbs` as one domain core with three adapters, rather than adopting a
BBS engine.**

Concretely:

- `internal/domain` owns accounts, boards, threads, bulletins, directory, APRS
  composition and every rule.
- `bbsd serve` listens for SSH, telnet and HTTP in one process, so a session is a
  goroutine rather than a process, and every surface calls the same functions.
- `bbsd aprs` is the only process that talks to APRS-IS, sends APRS messages or
  acknowledges them.
- The web UI is the existing design folded into the same binary, with a JSON API
  for scripts and future clients.

This is the only option that satisfies all eight preservation criteria in
[01-current-state.md](01-current-state.md) without a translation layer. It is
also the smallest amount of new code, because the rules are already written and
tested.

The architecture is in [04-architecture.md](04-architecture.md).

## The alternative that is actually competitive

**Adopt ENiGMA½ as the terminal engine, and keep a Go APRS service beside it.**

This is the right choice if the goal is to be a node in the classic BBS scene:
FTN feeds, ANSI art, doors, QWK, a multi-node board that other sysops recognise.
ENiGMA is modern, BSD-licensed, actively developed, container-friendly, and its
REST API gives the web front end a clean boundary.

What that path costs, honestly:

| Work item | Size | Why |
| --- | --- | --- |
| Import accounts | M | Callsign becomes `user_name`; profile fields become `user_property` rows; existing PBKDF2 parameters must be mapped or every user resets |
| Import boards and threads | M | Board becomes an area tag; `parent_id` becomes `reply_to_message_id`; the five-level cap is not an engine rule and must be enforced by the importer or a mod |
| APRS service | L | The existing receiver and ACK worker survive as a service, but the terminal screens have to be rebuilt as ENiGMA menus and art |
| Web BBS | L | Either a web handler module or a separate front end over `/_enig/api/v1/`; ENiGMA does not ship a board UI |
| Identity and email flows | M | Activation, reset, double-confirmed email change and sysop rules become mods or property conventions |
| Localisation | M | The four-language catalogue must be mapped onto ENiGMA's theme/art strings and per-user preference |
| Operations | M | A second runtime, a second schema, a second backup and a second upgrade path |

Estimated at 14–20 weeks of focused work, against 10–12 weeks end to end for the
recommended path, with less control over the result and two runtimes to operate.
The estimate is a judgement, not a measurement; the phase breakdown behind the
recommended figure is in [07-roadmap.md](07-roadmap.md).

**Adopt Synchronet instead** only if the deciding requirement is a bundled web
UI plus FTN and doors, and the operator is willing to run a C/C++ BBS with a
JavaScript module API. Its licensing needs review before redistribution.

## Sources

Read on 2026-11-23:

- ENiGMA½ repository and documentation: <https://github.com/NuSkooler/enigma-bbs>
  — `README.md`, `website/src/content/docs/servers/` (SSH, telnet, WebSocket,
  web server, REST API, web handlers), `website/src/content/docs/messageareas/`,
  `core/database.js`, `core/user_property.js`.
- Synchronet: <https://github.com/SynchronetBBS/sbbs> — repository tree
  (`webv4/`, `src/sbbs3/telnet.c`, `src/sbbs3/websrvr.cpp`, cryptlib patches),
  `docs/`, and <http://synchro.net/copyright.html>.
- WWIV: <https://github.com/wwivbbs/wwiv> — `README.md`, `LICENSE`.
- Mystic: <https://github.com/FIDOSOFT/mysticbbs> — repository description.
- Citadel: <https://github.com/million12/docker-citadel> and the Citadel project
  documentation.
- LinBPQ: <https://github.com/g8bpq/linbpq>, <https://github.com/g8bpq/BPQ32>.
- MagickaBBS: <https://github.com/Cloudxtreme/MagickaBBS>.
- Enthral: <https://github.com/M-griffin/Enthral> (sunset notice).
