# STATUS — LOOM-DOOM

**Parked** · reached ~Phase 7 ("shippable pending art") · last reviewed 2026-07 (Decision 5)

## What this repo is

LOOM-DOOM is a fork of [id Software's DOOM](https://github.com/id-Software/DOOM). The original
1997 C source is preserved untouched (`linuxdoom-1.10/`, `ipx/`, `sersrc/`, `sndserv/`,
[`README.TXT`](README.TXT)). It is **not** the LOOM game — it is LOOM's **design, lore, and
planning home**. Everything LOOM-specific lives under [`docs/`](docs/):

- [`docs/superpowers/specs/`](docs/superpowers/specs/) — per-phase design specs
- [`docs/superpowers/plans/`](docs/superpowers/plans/) — per-phase implementation plans
- [`docs/playtests/`](docs/playtests/) — per-phase playtest logs

The **playable game** — a browser DOOM-like tied to the l0b0t band universe — is implemented in
the sibling `l0b0tonline` repo (TypeScript / React / WebGL). GitHub issues on this repo were
enabled 2026-07-07 to serve as the task tracker.

## Status: parked

Development followed a disciplined, phased tracer-bullet process (Phases 0–7): each phase has a
design spec and implementation plan recorded here, with playtest logs through Phase 6. Phase 7's
[spec](docs/superpowers/specs/2026-04-28-loom-doom-phase-7-design.md) and
[plan](docs/superpowers/plans/2026-04-29-loom-doom-phase-7.md) are committed; its code lands in
`l0b0tonline`.

Phase 7 is the polish + closeout phase whose goal is *shippable-pending-art*. It introduces
**graceful art-slot scaffolding**: sprite, music-track, and surveillance-voiceover slots that
HEAD-probe for assets and fall back to the current programmatic visuals + Web-Audio synth when a
file is absent. A fresh checkout with no asset files still runs — so when bespoke art arrives it
drops into the ready slots with zero code changes.

This repo is **parked here**: no active feature development, art-slot scaffolding in place,
awaiting the creative (art + audio) pass. Parking applies to this repo only.

## Future work — the Phase 8 audio bridge

The most concrete band↔game bridge, carried forward from the 2026-07 review, is to record two
canonical LOOM tracks as **actual l0b0t songs** and drop them into the ready audio slots:

- ***Patch Tuesday (Heaven Vers!0n)*** — Cycle 4 boss fight + C-reveal credits
- ***human In l00p*** — Cycle 3 ambient

Today these play through synth placeholders (see
[Phase 7 spec §7](docs/superpowers/specs/2026-04-28-loom-doom-phase-7-design.md), which holds the
full deferred backlog). This becomes actionable once the l0b0t music pipeline is between albums —
cross-reference the l0b0t fleet plan (`jwest42/l0b0t` → `plans/album-fleet-plan.md`).
