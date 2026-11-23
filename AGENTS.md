# Project agent instructions

This repository is a **design record**, not a codebase.

## What this means for you

- Do not add implementation: no Go packages, Dockerfiles, Kubernetes manifests,
  SQL migration files, or scripts that perform work. Illustrative fragments
  inside a design document are acceptable only when marked as illustrative.
- Do change prose when it is wrong. A design document that contradicts the live
  system is a defect: correct it and note the evidence.
- Keep the documents consistent with each other. A change to a rule (retention
  cap, ACK behaviour, transport policy) usually touches `01`, `04`, `06` and
  `08` together.
- Mark anything not verified as an assumption. Never present an estimate as a
  measurement.

## Sources of truth, in order

1. The live cluster for facts about the current deployment.
2. `ea7klk/simple-ham-bbs` for current behaviour and its tests.
3. `ea7klk/ham-bbs-web` for the current web surface.
4. This repository for the intended successor.

When the design and the live system disagree, the live system is the fact and
the design is the proposal. Say which one changed.

## Measurement conventions

- Give the date of any live measurement.
- Name the command or query behind a number, or the document it came from.
- Distinguish "measured", "estimated" and "assumed".

## Writing conventions

- British spelling, plain sentences, no marketing voice.
- Prefer a list or a short table over a long paragraph.
- Use `hamnet-bbs` for the deployed system, `new-bbs` for the target.
- Link to the document that owns a topic instead of repeating it.

## GitHub operations

Use `gh` for everything. The account is already authenticated; never ask for a
token. Confirm the repository with `gh repo view ea7klk/new-bbs` before changing
anything, and do not push or change repository settings unless the user asked.

Conventional commits: `docs:`, `fix:`, `chore:`.
