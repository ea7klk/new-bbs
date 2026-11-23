# 9. Open questions

These need an owner decision before implementation starts. Each has a
recommendation, so a decision can be made by accepting or rejecting it.

## 1. Is the goal a ham community BBS, or presence in the classic BBS scene?

**Why it matters.** It decides the architecture. A community BBS with APRS is
best served by the core-plus-adapters design. A board that participates in FTN
networks, runs DOS doors and shows ANSI art is best served by ENiGMA½ or
Synchronet, with the APRS service beside it.

**Options.**

- A. Ham community BBS with APRS (recommended).
- B. Classic BBS scene presence, with APRS as an extra.
- C. Both, as separate systems: `new-bbs` for the community, an ENiGMA board for
  the scene, sharing an APRS service.

**Recommendation:** A. The current feature set, the user base and the operator's
investment all point to the community product. C is viable later and does not
require choosing now.

**Impact if wrong:** the whole design. This is the first decision.

## 2. When does HamNet actually become reachable?

**Why it matters.** Telnet is only safe on HamNet or the LAN. The HamNet tunnel
is designed but not implemented: the node has no `44.x` address, and the web
router matches only the public hostname (`simple-ham-bbs/HAMNET.md`).

**Options.**

- A. Build telnet now, expose it only inside the cluster and on the LAN until
  the tunnel exists.
- B. Build telnet and expose it through a HamNet-only router when the tunnel
  lands.
- C. Do not build telnet until the tunnel is real.

**Recommendation:** A now, B when the tunnel lands. The code and the
NetworkPolicy are ready; exposure is a deployment change. Telnet must never be
published on the public internet, whatever the schedule.

**Impact:** without B, telnet reaches only cluster-internal callers, which is
not useful to hams.

## 3. Do callsigns stay unverified?

**Why it matters.** Today anyone can claim any unclaimed callsign, and the first
claimant of a `BBS_SYSOPS` callsign became a sysop. The sysop race is fixed by
ADR-004. The underlying identity question remains.

**Options.**

- A. Keep unverified accounts (recommended). Add the CLI sysop grant and shared
  rate limits.
- B. Require proof of APRS-IS passcode possession at registration. The passcode
  is derived from the callsign, so a caller who can send APRS as that callsign
  can prove it.
- C. Require sysop approval for new accounts.
- D. Require an email address at a known ham domain (fragile, excludes people).

**Recommendation:** A for launch, B as an optional step for accounts that want
to send APRS. B is a natural fit because the BBS already derives the passcode;
it proves control of the callsign to APRS-IS, not legal identity, which is the
right bar.

**Impact:** B adds a registration step and an APRS-IS login test; it also makes
sending-as-someone-else harder.

## 4. Should SSIDs be first-class accounts?

**Why it matters.** Users operate `CALL-7`, `CALL-14`. The current receiver
already strips the SSID when matching an account (`TG5ALY-14` is stored under
`TG5ALY`), so the inbox behaves as if SSIDs were one identity. Sends leave from
SSID 0.

**Options.**

- A. Base callsign is the account; any `CALL-N` addresses that account
  (recommended, and what the data shows today).
- B. Each SSID is its own account with its own settings and history.
- C. Base callsign is the account, but users can register a preferred SSID for
  display and for ACK matching.

**Recommendation:** A, with C considered later if users ask. B doubles the
account surface and fragments history for no clear gain.

**Impact:** B would change the schema key and the APRS matching rules, and would
make the migration more complex.

## 5. How should duplicate bulletins be handled?

**Why it matters.** 5 138 rows contain 4 885 distinct texts. The 253 exact
duplicates come from different senders relaying the same bulletin. Collapsing
them hides relays; keeping them clutters the list.

**Options.**

- A. Keep the current per-sender dedupe; add `content_key` for analysis only
  (recommended).
- B. Collapse by content across senders, keeping a relay count.
- C. Collapse only within a short time window (for example, one hour).

**Recommendation:** A for the migration, then decide with data from
`content_key`. C is the likely long-term answer: it removes the clutter without
hiding that a bulletin was relayed.

**Impact:** the bulletin list size and the meaning of `seen_count`.

## 6. What is the rollback deadline?

**Why it matters.** Rollback before the first new write is a service switch.
After it, the old database is missing new rows. `export-legacy` can copy them
back, but it is extra work that must be built and rehearsed.

**Options.**

- A. 24-hour decision deadline; accept the loss of the delta after that
  (recommended if the community is small and tolerant).
- B. 7-day deadline with `export-legacy` built and rehearsed (recommended if
  posts matter).
- C. No rollback after cutover; the old system is deleted immediately.

**Recommendation:** B. The soak is seven days; the export command is a small
amount of work compared with losing a week of posts, and it makes the rollback
claim honest.

**Impact:** one more command in Phase 5, and a clear operational rule.

## 7. Does the web interface keep the current visual design?

**Why it matters.** `ham-bbs-web` has a distinctive "Packet Manual structure,
Spectrum palette" design with WCAG AA contrast. It is an asset.

**Options.**

- A. Keep it, port it as-is (recommended).
- B. Redesign for a more conventional community-forum look.
- C. Keep it for authenticated pages, simplify the public pages.

**Recommendation:** A. It differentiates the product, it is already accessible,
and a redesign is not on the critical path.

**Impact:** Phase 3 effort and the front-end tests.

## 8. Which repository owns the API contract?

**Why it matters.** `bbs-api` currently owns an OpenAPI design with no
implementation. `new-bbs` will implement it.

**Options.**

- A. Move the contract to `new-bbs` and archive `bbs-api` (recommended,
  ADR-007).
- B. Keep `bbs-api` as the contract owner and consume it as a dependency.
- C. Drop the formal contract and document the API in prose.

**Recommendation:** A. One repository, one commit, no drift.

**Impact:** a one-time move and an archive note; no ongoing cost.

## 9. What licence should `new-bbs` carry?

**Why it matters.** The repository is public. Without a licence, default
copyright applies and others cannot reuse it. The sibling repositories are MIT.

**Options.**

- A. MIT, matching the other repositories (recommended).
- B. Apache-2.0, with an explicit patent grant.
- C. No licence, all rights reserved.

**Recommendation:** A, unless the operator wants the patent grant. The code is a
personal ham project; MIT is consistent with the rest of the org.

**Impact:** none technical; it is a legal choice for the owner.

## 10. Is `ham-bbs-web` kept alive during the soak?

**Why it matters.** It is tempting to keep the old web app running against the
old database "just in case". It would write to a database the new system no
longer serves, and it would reintroduce the two-writer defect.

**Options.**

- A. Scale it to zero at cutover and keep it only as a rollback artifact
  (recommended).
- B. Keep it running against the old database for read-only reference.
- C. Keep it running in parallel against the new database.

**Recommendation:** A. B is acceptable only if the old database is opened
read-only and the app is known to honour that. C is unsafe.

**Impact:** the integrity of the migration and the credibility of "one writer".

## 11. Do telnet callers get self-registration?

**Why it matters.** Registration requires an email address and, with SMTP, an
activation code. Telnet has neither a browser nor a secure channel.

**Options.**

- A. Telnet is login-only; registration and password reset are web/SSH
  (recommended).
- B. Telnet registration with an emailed code typed into the terminal.
- C. Telnet registration with sysop approval.

**Recommendation:** A. It keeps the registration surface on channels where the
email flow works and reduces abuse from a plaintext transport.

**Impact:** one menu branch and a clear message on the telnet login screen.

## 12. When is the maintenance window?

**Why it matters.** APRS reception stops during the freeze. The window should
avoid known nets and the Thursday APRS activity.

**Options.**

- A. A weekday morning UTC, announced seven days ahead (recommended).
- B. A Sunday night UTC.
- C. No fixed window; migrate when the queue is empty.

**Recommendation:** A. The APRS gap is minutes, but the announcement is what
prevents "the BBS is down" messages.

**Impact:** scheduling only.
