# LOOM — Phase 7 Design Spec (Polish + Asset Scaffolding + Final Code Closeout)

**Date:** 2026-04-28
**Status:** Draft — pending user approval
**Authors:** brainstorm session, claude/phase-7-polish-shippable
**Working tree:** `LOOM-DOOM/` (branch `claude/phase-7-polish-shippable`)
**Implementation lives in:** `l0b0tonline/` (sibling repo, branch off `main`)

---

## 1. Goal

Make LOOM **shippable-pending-art**. Close every code-side item left in the Phase 5/6 review backlog, wire asset-loading scaffolding for the four creative artifacts that still need human authorship (two music tracks, sprite art, surveillance voiceover), validate the game on touch / small-viewport hardware, and lock the difficulty curve. After Phase 7, the only thing standing between LOOM and a public demo is recording / drawing the four assets and dropping them into known slots.

**Non-goals:** Generating finished assets (music / sprites / voiceover) with AI or otherwise. We do *not* dilute the bespoke aesthetic by shipping placeholder AI art. The asset slots remain intentionally empty; the game falls back to the current programmatic visuals + Web-Audio-synthesized soundtrack until real assets land.

---

## 2. Scope summary

| # | Item | Carry-from | Owner |
|---|------|-----------|-------|
| 7.1 | Mobile responsiveness investigation + minimum-viable touch controls | Phase 6 backlog | code |
| 7.2 | Final difficulty tuning (Easy / Normal / Hard balance pass) | Phase 6 backlog | code |
| 7.3 | Colorblind palette UI toggle + palette-aware enemy/projectile color shifts | Phase 6 backlog (struct-only in `accessibilityStore`) | code |
| 7.4 | Sprite manifest + loader with programmatic-rect fallback | Phase 6 backlog (asset scaffolding) | code |
| 7.5 | Audio track registry for *Patch Tuesday* + *human In l00p* with synth fallback | Phase 6 backlog (asset scaffolding) | code |
| 7.6 | Voiceover slot system for C2/C3 surveillance log audio | Phase 6 backlog (asset scaffolding) | code |
| 7.7 | I-3: GameLoop hold-fire dispatch + Termination Letter room-clear unit tests | Phase 6 review carry-over | code |
| 7.8 | N-3: Verify L-907 paraphrase rendered in HUD `SURVEILLANCE_FRAGMENTS` | Phase 6 review carry-over | code |

**Out of scope for Phase 7** (explicitly punted to Phase 8 = creative pass):
- Recording / mastering of *Patch Tuesday (Heaven Vers!0n)* and *human In l00p*
- Drawing / animating real sprite frames for the 13 enemies + 5 weapon viewmodels
- Recording the C2/C3 surveillance-log voiceover takes
- Any AI-generated placeholder content for the four asset slots above

---

## 3. Architectural shape

Phase 7 introduces **one new pattern** that recurs across 7.4/7.5/7.6: the *graceful asset slot*. All three follow the same shape:

```
data/assets/<kind>/manifest.ts   — declares the slot keys, expected file paths, fallback strategy
components/loom/engine/<kind>Loader.ts — fetches files at boot; missing files = silent fallback
components/loom/engine/<kind>Service.ts — getter API used by gameplay code; never blocks on load
```

The renderer / audio path queries the service synchronously. If the slot is loaded, it returns the asset; if it's still loading or absent, it returns the fallback (programmatic rect, synth oscillator, silent buffer). **No gameplay code branches on `if (loaded)` — the service abstraction guarantees a non-null return value always.**

This pattern matters because: (a) it lets us ship Phase 7 without any asset files committed to the repo, (b) when assets arrive in Phase 8 they drop into `public/loom/sprites/`, `public/loom/audio/tracks/`, `public/loom/audio/vo/` with zero code changes, and (c) the fallback path stays exercised in tests, so a missing file never silently breaks the game.

Outside the asset-slot pattern, Phase 7 is conventional polish: store extension for colorblind palette, a touch-input layer, telemetry-driven difficulty multipliers, and two batches of new unit tests.

---

## 4. Detailed design

### 4.1 Mobile responsiveness + minimum-viable touch controls

**Problem.** LOOM was authored desktop-first: pointer-lock cursor capture, WASD + mouse aiming, Q/E look (Phase 6.5), keyboard `T` for Termination Letter, `Tab` for HUD. On mobile / touch devices: pointer lock is unsupported on iOS Safari, no keyboard, viewport is portrait by default, and the J0IN 0S desktop chrome was sized for laptop+ widths.

**Investigation deliverable.** A short audit document (`docs/playtests/2026-04-XX-mobile-audit.md`) covering: which iOS / Android browsers can run the WebGL renderer at >=30fps, where the J0IN 0S window chrome breaks below 768px, and what a *minimum-viable* mobile experience looks like. Audit is a code-side observation pass (Playwright at multiple viewport sizes + manual device-frame screenshots), **not** a design-from-scratch effort.

**Code deliverables.**
- **Landscape lock prompt.** When the game window opens on a viewport with `height > width` and `width < 900px`, show a `Please rotate your device` overlay inside the LOOM window (not on J0IN 0S chrome). Pure CSS + a `window.matchMedia('(orientation: portrait)')` listener. The overlay disappears when the user rotates.
- **Touch input layer.** A `useTouchInput()` React hook that wraps the existing input system. Adds:
  - Left-half-of-canvas drag → look (replaces mouse-look)
  - Right-half-of-canvas tap → primary fire (replaces left-click)
  - Two-finger tap → switch weapon (Replaces wheel scroll)
  - Hold right-half → continuous fire (replaces hold left-click — this also makes the Hum hold-fire from 6.8 work on touch)
  - Long-press left-half → Termination Letter (replaces `T` key)
  - On-screen `MOVE` virtual joystick (left thumb) for WASD analog movement
  - Auto-detected via `('ontouchstart' in window)` — no UI toggle, falls back gracefully on hybrid laptops with touchscreens
- **Skill-level select touch sizing.** The Phase 6 `SkillLevelSelect` and the new `Phase 7.3` colorblind toggle become touch-target-sized (≥44px) when the touch-input layer is active.

**Out of scope.** Re-laying out the J0IN 0S desktop for portrait. (J0IN 0S itself is desktop-first by design; LOOM only needs the *game window* to work. The user opens LOOM, the game window goes fullscreen-within-J0IN-0S, and that's the contract.)

### 4.2 Final difficulty tuning

**Problem.** The Phase 6 `SkillLevelSelect` exists and the multipliers are wired through `Player.skillLevel`, but the actual numbers (`Easy: 1.25× damage taken, 0.75× damage dealt`, etc. — placeholder defaults from Task 6.6) have not been validated against actual play. We also have an existing `tests/nightmareDifficulty.test.tsx` from earlier work, suggesting a fourth tier was scaffolded.

**Telemetry hooks.** Add a lightweight `difficultyTelemetry.ts` module that tracks per-cycle stats during a playthrough: time-to-clear, deaths, weapon-switch frequency, ammo waste %, room-clear time. **Console-only** — no network egress. Used in dev to profile balance, stripped in prod via Vite `define` flag.

**Tuning pass deliverable.**
- A 60-minute playthrough on each of `easy / normal / hard` with telemetry on; record numbers in `docs/playtests/2026-04-XX-difficulty-pass.md`.
- Use the data to set final multipliers in `Player.skillLevel`. Likely directional moves: Easy stays accommodating but increases Termination Letter ammo cap; Normal becomes the canonical experience; Hard tightens room-clear times and reduces healing.
- **Nightmare** tier: confirm whether `nightmareDifficulty.test.tsx` represents a planned 4th tier; if yes, scope it to Phase 7 (just expose in the SkillLevelSelect); if it's a scaffold from earlier work that wasn't kept, delete the orphan test.

**Acceptance.** Three tier-specific full-playthrough recordings (or detailed playtest logs) exist. Multipliers are committed to `Player.skillLevel`. The `SkillLevelSelect` still defaults to Normal.

### 4.3 Colorblind palette UI + palette-aware color shifts

**Current state (Phase 6).** `accessibilityStore.colorblindPalette` exists with values `'normal' | 'deuteranopia'` but no UI toggle and no consumers. The store rehydration was hardened in the Phase 6 Codex P1 fix (commit `fa2da81`).

**Phase 7 deliverables.**
- **Palette module.** `components/loom/render/palette.ts` exporting two constant palettes: `NORMAL_PALETTE` (current colors) and `DEUTERANOPIA_PALETTE` (red↔orange/blue contrast pair, validated with a deuteranopia simulator). Each palette is keyed by *role* (`enemy.melee`, `enemy.ranged`, `enemy.boss`, `projectile.player`, `projectile.enemy`, `pickup.health`, `pickup.ammo`, `pickup.terminationLetter`, `hud.warning`, `hud.success`).
- **Renderer integration.** Three sprite-rendering call-sites switch from hardcoded hex to `getPalette()[role]`: `WebGL2Renderer.drawEnemy`, `WebGL2Renderer.drawProjectile`, `WebGL2Renderer.drawPickup`. The `getPalette()` call resolves once per frame from the store; cheap.
- **HUD integration.** `LoomHud` health/ammo/warning glyphs already use Tailwind utility classes; we add a `data-palette={palette}` attribute on the HUD root + a small CSS rule block in `globals.css` that overrides those colors under `[data-palette='deuteranopia']`. Surveillance log + manifesto text *do not* change — they're pure text on green CRT background and serve a different aesthetic role.
- **UI toggle.** Extend the existing Phase 6 `SkillLevelSelect` accessibility checkboxes (currently *Reduced motion* + *Keyboard look*) with a third radio-pair for *Color palette* (`Default` / `Deuteranopia`). The store update wires through the existing `setColorblindPalette` action.
- **Tests.** A unit test that sets palette = deuteranopia and reads back rendered enemy color; a visual-regression Playwright snapshot at the difficulty-select screen (toggle on / off).

**Out of scope.** Tritanopia / protanopia / monochrome palettes. (Deuteranopia is the most common form; nailing one well is a stronger ship than three half-done.)

### 4.4 Asset slot — Sprite manifest + loader

**Slot design.**
```
public/loom/sprites/
├── enemies/
│   ├── intern.png       # 64×64, transparent PNG, idle/walk/death frames stacked vertically
│   ├── hr_skeleton.png
│   └── …                # 13 enemies total
├── weapons/
│   └── viewmodels/
│       ├── drumstick.png
│       ├── bass.png
│       └── …            # 5 weapon viewmodels
└── pickups/
    ├── health.png
    ├── ammo.png
    └── termination_letter.png
```

**Manifest** (`data/assets/sprites/manifest.ts`):
```typescript
export const SPRITE_MANIFEST = {
  enemies: {
    intern: { path: '/loom/sprites/enemies/intern.png', frames: 8, frameWidth: 64, frameHeight: 64 },
    // … one entry per enemy
  },
  weapons: { /* … */ },
  pickups: { /* … */ },
} as const;
```

**Loader** (`components/loom/engine/spriteLoader.ts`): on game-window mount, iterate the manifest and `fetch`-then-decode each entry into an `ImageBitmap`. **Each fetch is independent** — one missing file does not block others. Loader updates a `Map<string, ImageBitmap | 'missing'>` keyed by sprite ID.

**Service** (`components/loom/engine/spriteService.ts`):
```typescript
function getSprite(id: string): ImageBitmap | null {
  const entry = spriteCache.get(id);
  return entry === 'missing' || entry === undefined ? null : entry;
}
```

**Renderer fallback.** `WebGL2Renderer.drawEnemy` checks `getSprite(enemy.spriteId)`; if `null`, draws the existing programmatic colored rect (which already takes its color from `palette[role]` from §4.3 — so the fallback is now palette-aware too). When the sprite IS present, the rect is replaced with a textured quad using the sprite's frame for the enemy's current animation state.

**Tests.** `spriteLoader.test.ts` mocks `fetch` to return 200 / 404 mixes and asserts the cache state. A Playwright run with **no sprite files present** (current state) MUST still pass the full Phase 6 e2e — confirms the fallback path is the default and gameplay is unchanged when assets are missing.

### 4.5 Asset slot — Audio track registry

**Slot design.**
```
public/loom/audio/tracks/
├── patch_tuesday.mp3        # *Patch Tuesday (Heaven Vers!0n)* — Cycle 4 boss-fight + C-reveal credits
├── human_in_loop.mp3        # *human In l00p* — Cycle 3 ambient
└── …
```

**Track registry** (`data/assets/audio/trackManifest.ts`):
```typescript
export const TRACK_MANIFEST = {
  patch_tuesday: { path: '/loom/audio/tracks/patch_tuesday.mp3', useFor: ['cycle4', 'credits'] },
  human_in_loop: { path: '/loom/audio/tracks/human_in_loop.mp3', useFor: ['cycle3'] },
};
```

**Loader** (`components/loom/engine/audioTrackLoader.ts`): on first audio-context resume, attempt to fetch+decode each track. **HEAD request first** — cheap probe to detect 404 without paying the decode cost on missing files.

**Service hook.** `audioController.playTrackForCycle(cycle: number)` queries the registry. If the cycle's intended track is loaded, swap in via the existing audio mixer with a 2s crossfade; if missing, the existing Web-Audio-synthesized ambient (current Phase 6 behavior) keeps playing — same crossfade, just to itself with no audible swap.

**Tests.** `audioTrackLoader.test.ts` mocks fetch + AudioContext.decodeAudioData. Existing `audioController.test.ts` extends with two cases: "Cycle 3 entry with Track loaded" (verify crossfade), "Cycle 3 entry with Track missing" (verify synth ambient continues uninterrupted).

### 4.6 Asset slot — Surveillance voiceover

**Slot design.**
```
public/loom/audio/vo/
├── c2_surveillance_01.mp3   # Cycle 2 PA loops — short, bureaucratic
├── c2_surveillance_02.mp3
├── c3_surveillance_01.mp3   # Cycle 3 PA loops — distorted, accusatory
└── c3_surveillance_02.mp3
```

**Manifest + loader pattern identical to §4.5.**

**Service.** A new `voiceoverService.scheduleRandomFromCycle(cycle: 2 | 3)` API. The existing surveillance-log HUD ticker (Phase 5) already rotates text fragments on a coprime-interval schedule. We add an audio companion: every N text rotations (configurable, default 4 for C2 / 2 for C3 — voiceover gets denser as paranoia rises), the service picks a random VO clip from that cycle's pool and plays it through the dialogue audio bus at -12dB.

**Reduced-motion respect.** When `accessibilityStore.reducedMotion === true`, voiceover plays at -18dB (quieter, less startling). When `reducedMotionPref === true` (explicit) AND the user has explicitly muted voiceover via a *new* checkbox in `SkillLevelSelect`, voiceover is silent. (Reduced motion alone does not silence — the surveillance theme is the campaign's emotional payload and full silence would hurt; lowering volume is the compromise.)

**Tests.** `voiceoverService.test.ts` covers rotation cadence, silence under both flags, missing-file fallback (silent, no errors logged at WARN+).

### 4.7 Deferred I-3 — GameLoop hold-fire + TL room-clear unit tests

**Carried from Phase 6 review.** Phase 6 added:
- Hum hold-fire dispatch in `gameLoop.ts` (Task 6.8) — pumps continuous primary-fire when `input.isPrimaryFireDown` AND the active weapon supports hold-fire.
- Termination Letter room-clear (Task 6.7) — pressing `T` consumes a TL and triggers a damage burst against all enemies in the active room.

Both ship without dedicated unit tests in Phase 6. Phase 7 closes the gap:
- `gameLoop.test.ts` extends with: "hold-fire pumps Hum at correct cadence when isPrimaryFireDown=true", "hold-fire stops on isPrimaryFireDown=false", "hold-fire does NOT pump for weapons without hold-fire (Drumstick, etc.)".
- `terminationLetter.test.ts` (new): "consuming TL with enemies in current room damages all of them once", "TL with zero ammo is a no-op", "TL respects room boundaries — enemies in adjacent rooms not damaged".

### 4.8 Deferred N-3 — L-907 paraphrase HUD verification

**Carried from Phase 6 review.** During Phase 6, we wanted to confirm the HUD `SURVEILLANCE_FRAGMENTS` text rotation includes a paraphrase / hint of L-907's identity by mid-Cycle-3, ahead of the Cycle 4 reveal. This was deferred because the rotation is intent-driven, not assertion-tested.

**Phase 7 deliverable.** A unit test on the `SURVEILLANCE_FRAGMENTS` array: assert that *some* fragment in the C3 pool contains a substring matching `/l[-_]?907|null[-_]?entity|seventh|saint/i` (case-insensitive). This is a guard against accidental deletion of the foreshadowing without forcing a specific exact line. The existing C-reveal narrative requires this hint to land; the test makes that requirement permanent.

If no current fragment matches, write one. (User-facing copy decision; will draft and request review in the implementation phase.)

---

## 5. Testing strategy

| Layer | Coverage |
|------|----------|
| Vitest unit | `palette.test.ts`, `spriteLoader.test.ts`, `audioTrackLoader.test.ts`, `voiceoverService.test.ts`, `gameLoop.test.ts` extension, `terminationLetter.test.ts`, `surveillanceFragments.test.ts`, `difficultyTelemetry.test.ts`, `touchInput.test.ts` |
| Vitest integration | `accessibilityStore` extension test for `setColorblindPalette` + render-side palette switch via mocked WebGL canvas |
| Playwright e2e | Phase 6 `loom-phase-6.spec.ts` rebrands to `loom-phase-7.spec.ts`. Adds: small-viewport touch-control flow (Playwright's `hasTouch` viewport), colorblind toggle persistence across reload, asset-loader graceful-fallback (assets absent in test env — current state, must still pass) |
| Manual playtest | Three full playthroughs (Easy / Normal / Hard) on desktop, one full playthrough on iOS Safari + Android Chrome with touch input, log to `docs/playtests/2026-04-XX-phase-7-*.md` |

**TDD discipline.** Each task in the implementation plan starts with a failing test, followed by the minimal code to pass. Same rhythm as Phases 4–6.

---

## 6. Acceptance criteria

Phase 7 is complete when *all* of these are true:

1. The Phase 6 e2e (rebranded as Phase 7 e2e) passes on the new branch.
2. All eight items in §2 are implemented and merged.
3. **Asset slots are demonstrably empty-but-functional**: a fresh checkout of `main` with no asset files added passes the e2e and renders gameplay identical to current Phase 6.
4. Three playtest logs exist documenting Easy / Normal / Hard balance pass.
5. One mobile-audit log exists, plus at least one mobile-device playthrough log.
6. `accessibilityStore` covers `reducedMotion`, `keyboardLook`, `colorblindPalette` — all three reachable from in-game UI.
7. The L-907 paraphrase test guards the surveillance-fragment foreshadowing.
8. The TL room-clear and Hum hold-fire mechanics each have dedicated unit tests.

---

## 7. Phase 8 backlog (intentionally deferred)

These are the four assets the Phase 7 scaffolding is preparing slots for:

1. **Patch Tuesday (Heaven Vers!0n)** — full track, ~3:30, Cycle 4 boss + credits.
2. **human In l00p** — full track, ~3:00, Cycle 3 ambient.
3. **13 enemy sprites + 5 weapon viewmodels + 3 pickup sprites** — pixel-art, transparent PNG with frame strips per the manifest in §4.4.
4. **C2/C3 surveillance voiceover** — ~6 short PA-loop clips, dry recording → light corp-comms processing for C2, heavy distortion + cut-up for C3.

Plus any post-mobile-audit findings that exceeded "minimum viable" scope (e.g., a custom mobile chrome theme for J0IN 0S, on-screen weapon-wheel selector, etc.).

---

## 8. Risks + mitigations

| Risk | Mitigation |
|------|-----------|
| Touch-input layer hides bugs in pointer-lock path on hybrid devices | Auto-detect on `('ontouchstart' in window) && navigator.maxTouchPoints > 0`; provide a debug query-string `?forceMouse=1` to override |
| Asset-loader 404s spam the dev console | Loader uses `fetch(path, { method: 'HEAD' })` first; only logs 404 at `debug` level (not `warn`) since absence is the expected default state |
| Difficulty tuning numbers feel arbitrary without telemetry data | Telemetry hook lands first in the plan; tuning pass uses real numbers, not vibes |
| Colorblind palette breaks the green-CRT aesthetic | Palette swap touches sprite roles only — text and HUD frame chrome stay green-on-black; the deuteranopia palette redistributes color use within the role-keyed slots |
| Voiceover timing collides with Patch Tuesday backing track at C-reveal | Voiceover bus is -12dB by default and ducks further when a track plays in the music bus (simple sidechain via the existing audio mixer's bus-priority field) |

---

## 9. Out-of-scope reminder

Phase 7 does NOT:
- Generate or commit real sprite art, music tracks, or voiceover audio
- Add new enemies, weapons, levels, or narrative content
- Refactor the WebGL2 renderer beyond palette + sprite-quad integration
- Touch the J0IN 0S desktop chrome layout outside the LOOM window itself
- Address the Phase 6 N-3 "verify the paraphrase exists" by hand-writing copy that doesn't already pass — if no current fragment matches, the implementation plan adds a draft line and flags it for user review

---

## 10. Definition of done

When PRs land:
- l0b0tonline: a single PR off `main` titled `Phase 7 — polish + asset scaffolding + final code closeout` with all eight item commits.
- LOOM-DOOM: a single PR off `master` containing this spec, the Phase 7 implementation plan, and the three+ playtest logs.

Both PRs reference each other in their bodies. Codex review on l0b0tonline; LOOM-DOOM PR is docs-only.
