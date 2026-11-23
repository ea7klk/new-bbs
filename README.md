# new-bbs

Design for the successor to **hamnet-bbs**: one BBS serving amateur radio
operators over SSH, telnet and the web, with APRS messaging available from
every surface.

**Status: design only. Nothing in this repository is implemented.** There is no
code, no container, no manifest. This is the specification an implementation
would be written against, and the record of why it looks the way it does.

## The four questions this repository answers

1. **What replaces hamnet-bbs?** — [docs/03-options.md](docs/03-options.md), with
   the recommendation in [docs/04-architecture.md](docs/04-architecture.md).
2. **What does the current system actually do, and what data exists?** —
   [docs/01-current-state.md](docs/01-current-state.md), measured against the
   live deployment on 2026-11-23.
3. **How does existing data move?** — [docs/05-migration.md](docs/05-migration.md).
4. **How do APRS functions stay available in both text and web?** —
   [docs/06-aprs.md](docs/06-aprs.md).

## Recommendation in brief

**No off-the-shelf BBS is a drop-in replacement for hamnet-bbs.** The closest
engines are ENiGMA½ and Synchronet; both are credible terminal BBS packages, and
neither carries APRS, callsign identity, the existing four-language catalogue, or
the current data model. Adopting one means migrating users and threads into a
foreign schema and rebuilding the APRS and web layers on top.

**The recommended target is a re-architecture of the system we already have:**

- one Go domain core that owns every rule and the database;
- one terminal adapter serving **SSH and telnet** from the same screen model;
- one HTTP adapter serving server-rendered **HTML and a JSON API** from the same
  domain calls;
- a **separate APRS service** that is the only process allowed to talk to
  APRS-IS or acknowledge a message.

That keeps APRS, thread semantics, callsign accounts and the existing web design,
while removing the two structural defects that matter: the same rule
implemented three times, and a process per SSH session.

Full reasoning, including the case where an engine *is* the right answer, is in
[docs/03-options.md](docs/03-options.md).

## What was measured

From the live `ham-bbs` namespace on 2026-11-23:

- 7 accounts across 3 languages, 4 boards, 5 messages, 1 bulletin;
- 125 APRS messages sent (141 parts, 3 failed), 667 received, 5 138 bulletin rows;
- SQLite database 3.1 MB with a 4.2 MB WAL;
- three Deployments (`bbs`, `bbs-aprs`, `bbs-web`) and two ReadWriteOnce claims.

Volume is small today. The design is sized for 1 000–5 000 accounts, where the
current per-session startup cost becomes visible (2.5 s at 5 000, measured in
`SCALABILITY.md` in `simple-ham-bbs`).

## Documents

- [01 — Current state](docs/01-current-state.md): what is deployed, what it
  stores, which behaviours are a contract.
- [02 — Requirements](docs/02-requirements.md): what "replace" must mean,
  including the non-goals.
- [03 — Options](docs/03-options.md): the survey, the rubric, the scores, and the
  recommendation with its strongest alternative.
- [04 — Architecture](docs/04-architecture.md): components, data model, process
  boundaries, deployment, security.
- [05 — Migration](docs/05-migration.md): field-by-field mapping, cutover,
  rollback, verification.
- [06 — APRS](docs/06-aprs.md): text and web surfaces, single sender, ACK rules,
  bots, bulletins, failure modes.
- [07 — Roadmap](docs/07-roadmap.md): phases, exit criteria, relative effort.
- [08 — Decisions](docs/08-decisions.md): the ADRs behind the design.
- [09 — Open questions](docs/09-open-questions.md): what the owner must decide
  before implementation starts.

## The decision that comes first

The design assumes the goal is **a ham radio community BBS with APRS**, not
**presence in the classic BBS scene** (FTN networks, ANSI art, doors, file
bases). If the second goal matters more, ENiGMA½ becomes the better answer and
[docs/03-options.md](docs/03-options.md#the-alternative-that-is-actually-competitive)
explains what that path costs. That is question 1 in
[docs/09-open-questions.md](docs/09-open-questions.md).

## Related repositories

- `ea7klk/simple-ham-bbs` — the current canonical SSH BBS and APRS receiver.
- `ea7klk/ham-bbs-web` — the current web front end, sharing the same SQLite file.
- `ea7klk/bbs-api` — the REST API design that `new-bbs` absorbs (ADR-007).

## License

Not chosen yet. See [docs/09-open-questions.md](docs/09-open-questions.md).
