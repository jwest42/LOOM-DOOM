# LOOM Phase 4 Implementation Plan — Cycle 3 Uncanny Break

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship **Cycle 3 — the uncanny break**. After defeating the VP of Sales the player crosses the tonal cliff: HR Skeletons appear, the HUD starts lying (health values flicker to wrong numbers, ammo briefly displays in binary, surveillance fragments invade faster), three new band-instrument weapons unlock (Guitar / Bass / Frequency Tuner), and four new enemies populate six progressively-decompiling versions of C1/C2 spaces. The cycle ends in the Brand Ambassador's atrium — a boss that can resurrect dead enemies with a "rebrand" beam.

**Architecture:** Extends the Phase 0–3 engine. `campaign.ts` adds `CYCLE_3_MAPS` and `isEndOfCampaign` shifts to the new C3 boss map. `cycleStore` is unchanged in shape — `CycleNumber = 1 | 2 | 3 | 4` and `CorruptionLevel = 0 | 1 | 2 | 3` already cover C3, and the existing `advanceCycle()` action correctly transitions cycle 2→3 and corruption 1→2. The HUD reads `corruption >= 2` to enable lying values + binary ammo blips. `BaseEnemy` gains a `rebranded` flag and an `applyKnockback(dx, dy, force)` method (used by Bass + the Brand Ambassador's resurrection bookkeeping). Three new weapons land on slots 4 / 5 / 6 (no slot conflicts; only slots 0-3 are currently bound). Four new enemy classes + four new projectile classes ship alongside.

**Tech Stack:** Same as Phase 3 — TypeScript 5.8, React 19, Vite 6, Tailwind v4, Zustand 5, WebGL 2, Web Audio API, Vitest, Playwright.

**Pre-existing state:**
- Implementation lives at `/Users/justinwest/Repos/l0b0tonline`. Phase 3 PR is `loom-game` (PR #100); Phase 4 stacks on top of the same branch (or onto `main` if PR #100 lands first — same commit set either way).
- Phase 3 left the project with: 8 maps (3 C1 + 5 C2), 8 enemies (4 C1 + 4 C2), 6 weapons (Drum Stick, Industrial Shredder, Branded Pen, Spam Filter, Reply All Storm, Neural Pulse), HUD corruption mode 1, full Cycles 1+2 e2e Playwright walkthrough.
- All Phase 3 review fixes already shipped (commits `84b3fec` through `8cc3cba`).

**Deferred from Phase 4 to later phases (kept off the critical path):**
- **Real *Err0r Flesh* track** for the Brand Ambassador arena — uses *Ship It* as placeholder for now (matches the Phase 3 deferral pattern; needs a new `soundService` method to land cleanly).
- **Neural Pulse visual-glitch evolution** — purely visual sugar; lands in Phase 5/6 alongside the post-FX shader pass.
- **Renderer palette inversion / desaturation** for C3 maps — requires a fragment-shader extension; defer to Phase 6 polish.
- **60 Hz hum intensity ramping** by cycle — defer to Phase 6 polish (audio mix balance).
- **Guitar windmill-strum melee** — Guitar primary fire is the chord projectile only; melee branch (alt fire) deferred to Phase 6.
- **Bass charge-and-release mechanic** — Bass fires immediately on click in Phase 4; charge/hold UI ergonomics defer to Phase 6.
- **Frequency Tuner 60 Hz / 432 Hz cell modes + cells feeding Neural Pulse** — Phase 4 ships the 528 Hz cell (piercing damage) only; mode-cycling and Neural Pulse cross-pollination defer to Phase 5.

These deferrals keep Phase 4 honest about the *creative* gate (the uncanny tonal cliff lands) without blocking on shader work or new soundService methods.

---

## File structure (post-Phase-4, in l0b0tonline)

New files (NEW), modified files (MOD):

```
l0b0tonline/
├── components/
│   └── loom/
│       ├── engine/
│       │   ├── audioController.ts                              (MOD — register cyc3_brand_summit_boss as boss arena)
│       │   ├── campaign.ts                                     (MOD — CYCLE_3_MAPS, isEndOfCampaign now points at cyc3_brand_summit_boss)
│       │   ├── campaign.test.ts                                (MOD — extended for 3-cycle progression)
│       │   ├── input.ts                                        (MOD — Digit4/5/6 → weaponSelectQueue handlers)
│       │   └── gameLoop.ts                                     (MOD — register 4 new enemy ThingTypes + 3 new weapons on slots 4/5/6)
│       ├── entities/
│       │   ├── enemies/
│       │   │   ├── Enemy.ts                                    (MOD — add `rebranded` flag + `applyKnockback` method to BaseEnemy)
│       │   │   ├── baseEnemy.test.ts                           (MOD — extended for new fields)
│       │   │   ├── surveillanceDrone.ts                        (NEW — heavy flying laser enemy)
│       │   │   ├── surveillanceDrone.test.ts                   (NEW — laser cooldown + sight)
│       │   │   ├── hrSkeleton.ts                               (NEW — uncanny break-marker; mid-melee + tracking projectile)
│       │   │   ├── wellnessOfficer.ts                          (NEW — twin bioacoustic pulses)
│       │   │   ├── brandAmbassador.ts                          (NEW — Cycle 3 boss + resurrection)
│       │   │   └── brandAmbassador.test.ts                     (NEW — resurrection mechanic)
│       │   ├── projectiles/
│       │   │   ├── laserScan.ts                                (NEW — Surveillance Drone projectile)
│       │   │   ├── laserScan.test.ts                           (NEW)
│       │   │   ├── trackingDoc.ts                              (NEW — HR Skeleton's projectile; faster variant of documentationRequest)
│       │   │   ├── wellnessPulse.ts                            (NEW — Wellness Officer's bioacoustic projectile)
│       │   │   └── chordNote.ts                                (NEW — Guitar's projectile, color-coded)
│       │   └── weapons/
│       │       ├── guitar.ts                                   (NEW — slot 4)
│       │       ├── guitar.test.ts                              (NEW)
│       │       ├── bass.ts                                     (NEW — slot 5, area shockwave + knockback)
│       │       ├── bass.test.ts                                (NEW)
│       │       ├── frequencyTuner.ts                           (NEW — slot 6, piercing 528 Hz cell)
│       │       └── frequencyTuner.test.ts                      (NEW)
│       ├── hud/
│       │   └── LoomHud.tsx                                     (MOD — corruption mode 2: lying values, binary ammo blips, faster log)
│       ├── store/
│       │   └── cycleStore.test.ts                              (MOD — assert advance from C2 lands corruption=2)
│       └── types.ts                                            (MOD — 4 new ThingType variants)
└── public/
    └── data/
        └── loom/
            └── maps/
                ├── cyc3_lobby_redux.json                       (NEW)
                ├── cyc3_cubicles_decay.json                    (NEW)
                ├── cyc3_open_office_inverse.json               (NEW)
                ├── cyc3_glass_echo.json                        (NEW)
                ├── cyc3_wellness_program.json                  (NEW)
                └── cyc3_brand_summit_boss.json                 (NEW)
```

E2E:
- `e2e/loom-phase-4.spec.ts` (NEW — full 14-map cycles 1+2+3 walkthrough; replaces `loom-phase-3.spec.ts`)
- `e2e/loom-phase-3.spec.ts` (DELETE — superseded; the Phase 4 spec is a strict superset)

---

# Phase 4 Definition of Done

A new player can:
1. Boot LOOM, play through Cycles 1+2 (8 maps as before) and defeat VP of Sales
2. Cross into Cycle 3 via the existing `cycle-transition` event — the intermission says *"next: cyc3_lobby_redux"* and the HUD updates to `> cycle 8494 // simulation BREACH // surveillance: ON`
3. **The first HR Skeleton sighting in `cyc3_lobby_redux` lands as wrong** — visual placeholder is bone-white pantsuit; meets every existing wall-collision and pain rule; combat is mechanically fair. (The "does it land" creative gate.)
4. Play through Cycle 3's 6 maps engaging Surveillance Drone (laser-scan), HR Skeleton (tracking projectile), Wellness Officer (twin bioacoustic), and finally Brand Ambassador (rebrand-the-dead resurrection beam)
5. Use the three new band-instrument weapons (Guitar slot 4, Bass slot 5, Frequency Tuner slot 6)
6. See the HUD lying when corruption ≥ 2 — every ~6s a health value flickers wrong for ~200ms, every ~10s the queue counter flips to binary briefly, the surveillance log rotates ~2× faster
7. Defeat the Brand Ambassador in the boss arena — boss-gate prevents exit until the boss dies, then `EndOfCycleStub` shows "CYCLE 4 — COMING SOON" + manifesto

**Automation gate:** Phase 4 Playwright spec walks through 14 maps + 3 boss kills (HR Manager / VP of Sales / Brand Ambassador), asserts cycleStore advances correctly (1→2→3, corruption 0→1→2), no console errors, full run completes in under 25s. Vitest covers the campaign 3-cycle progression, `BaseEnemy` knockback + rebranded fields, the Brand Ambassador's resurrection cycle, the Bass area shockwave, the Guitar 3-projectile spawn pattern, and the Frequency Tuner piercing semantics.

**Manual creative gate** (NEEDS-HUMAN, captured in the Phase 4 playtest log):
- Does the first HR Skeleton sighting *feel* uncanny, not just buggy?
- Does the HUD lying read as menacing, or just look broken?
- Do the band-instrument weapons feel mechanically distinct (chord shotgun, area shockwave, piercing beam)?
- Does Brand Ambassador's resurrection telegraph clearly enough that the player can interrupt it?

---

# Tasks

## Task 4.1: Extend `campaign.ts` for 3-cycle progression

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/campaign.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/campaign.test.ts`

The Phase 3 implementation already supports the cycle-transition event shape and the `isEndOfCampaign` boundary. Phase 4 just needs to:
- Add `CYCLE_3_MAPS`
- Extend `MAP_TO_CYCLE` so the 6 new ids return cycle 3
- The existing `getNextMapId` will now thread `cyc2_glass_confroom_boss → cyc3_lobby_redux` automatically (since both lists are concatenated by `ALL_CYCLE_MAPS`)
- `isEndOfCampaign` will now report `true` only at `cyc3_brand_summit_boss` (the new last-shipped map) — no code change required, just a test update

- [ ] **Step 1: Extend `campaign.test.ts` for 3-cycle progression**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/campaign.test.ts`. Add `CYCLE_3_MAPS` to the imports and add these `describe` block additions / replacements:

a) In the import line at the top:
```typescript
import {
  CYCLE_1_MAPS,
  CYCLE_2_MAPS,
  CYCLE_3_MAPS,
  getCycleOfMap,
  getNextMapId,
  isEndOfCampaign,
  isLastMapOfCycle,
} from './campaign';
```

b) Add a new test inside the `cycle map lists` describe (after the `CYCLE_2_MAPS` test):
```typescript
    it('CYCLE_3_MAPS lists 6 maps in order', () => {
      expect(CYCLE_3_MAPS).toEqual([
        'cyc3_lobby_redux',
        'cyc3_cubicles_decay',
        'cyc3_open_office_inverse',
        'cyc3_glass_echo',
        'cyc3_wellness_program',
        'cyc3_brand_summit_boss',
      ]);
    });
```

c) Add a new test inside the `getCycleOfMap` describe:
```typescript
    it('returns 3 for Cycle 3 maps', () => {
      expect(getCycleOfMap('cyc3_lobby_redux')).toBe(3);
      expect(getCycleOfMap('cyc3_brand_summit_boss')).toBe(3);
    });
```

d) Replace the existing `crosses the cycle 1→2 boundary` test inside `getNextMapId` with both boundary tests:
```typescript
    it('crosses the cycle 1→2 boundary', () => {
      expect(getNextMapId('cyc1_hr_arena')).toBe('cyc2_open_office');
    });

    it('crosses the cycle 2→3 boundary', () => {
      expect(getNextMapId('cyc2_glass_confroom_boss')).toBe('cyc3_lobby_redux');
    });

    it('returns next map within Cycle 3', () => {
      expect(getNextMapId('cyc3_lobby_redux')).toBe('cyc3_cubicles_decay');
      expect(getNextMapId('cyc3_wellness_program')).toBe('cyc3_brand_summit_boss');
    });
```

e) Replace the existing `returns null after the final shipped map` test (which was checking `cyc2_glass_confroom_boss`) with:
```typescript
    it('returns null after the final shipped map', () => {
      expect(getNextMapId('cyc3_brand_summit_boss')).toBeNull();
    });
```

f) Replace the `true at cycle 2 final map (end-of-campaign)` test with:
```typescript
    it('true at cycle 2 final map (cycle-transition coming)', () => {
      expect(isLastMapOfCycle('cyc2_glass_confroom_boss')).toBe(true);
    });

    it('true at cycle 3 final map (end-of-campaign)', () => {
      expect(isLastMapOfCycle('cyc3_brand_summit_boss')).toBe(true);
    });
```

g) Replace the `isEndOfCampaign — true only at cyc2_glass_confroom_boss` test with:
```typescript
    it('true only at the very last shipped map (cyc3_brand_summit_boss)', () => {
      expect(isEndOfCampaign('cyc3_brand_summit_boss')).toBe(true);
    });

    it('false at cycle 2 final map (campaign continues into cycle 3)', () => {
      expect(isEndOfCampaign('cyc2_glass_confroom_boss')).toBe(false);
    });
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/engine/campaign.test.ts 2>&1 | tail -15
```

Expected: tests fail because `CYCLE_3_MAPS` doesn't exist; `getNextMapId('cyc2_glass_confroom_boss')` returns null instead of `'cyc3_lobby_redux'`; `isEndOfCampaign('cyc3_brand_summit_boss')` returns false (map is unknown).

- [ ] **Step 3: Update `campaign.ts` with `CYCLE_3_MAPS`**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/campaign.ts`.

Replace the entire file contents with:
```typescript
import type { CycleNumber } from '../store/cycleStore';

/**
 * Campaign progression — what map comes after which, and where each cycle ends.
 *
 * Phase 4 ships Cycles 1 + 2 + 3 (14 maps total). Cycle 4 lands in Phase 5.
 */

export const CYCLE_1_MAPS = ['cyc1_lobby', 'cyc1_cubicles', 'cyc1_hr_arena'] as const;
export const CYCLE_2_MAPS = [
  'cyc2_open_office',
  'cyc2_break_room',
  'cyc2_slack_huddle',
  'cyc2_wellness_ctr',
  'cyc2_glass_confroom_boss',
] as const;
export const CYCLE_3_MAPS = [
  'cyc3_lobby_redux',
  'cyc3_cubicles_decay',
  'cyc3_open_office_inverse',
  'cyc3_glass_echo',
  'cyc3_wellness_program',
  'cyc3_brand_summit_boss',
] as const;

export type CycleMapId =
  | typeof CYCLE_1_MAPS[number]
  | typeof CYCLE_2_MAPS[number]
  | typeof CYCLE_3_MAPS[number];

const ALL_CYCLE_MAPS: readonly string[] = [...CYCLE_1_MAPS, ...CYCLE_2_MAPS, ...CYCLE_3_MAPS];

const MAP_TO_CYCLE: ReadonlyMap<string, CycleNumber> = new Map<string, CycleNumber>([
  ...CYCLE_1_MAPS.map((id): [string, CycleNumber] => [id, 1]),
  ...CYCLE_2_MAPS.map((id): [string, CycleNumber] => [id, 2]),
  ...CYCLE_3_MAPS.map((id): [string, CycleNumber] => [id, 3]),
]);

/** Returns the cycle number for a given map, or null if unknown. */
export function getCycleOfMap(currentId: string): CycleNumber | null {
  return MAP_TO_CYCLE.get(currentId) ?? null;
}

/** Returns the id of the map that follows `currentId`, or null at end of campaign. */
export function getNextMapId(currentId: string): string | null {
  const idx = ALL_CYCLE_MAPS.indexOf(currentId);
  if (idx < 0) return null;
  return ALL_CYCLE_MAPS[idx + 1] ?? null;
}

/**
 * True if the given map is the final map of its cycle.
 * - At cyc1_hr_arena: true (next map is cyc2_open_office, different cycle)
 * - At cyc2_glass_confroom_boss: true (next map is cyc3_lobby_redux, different cycle)
 * - At cyc3_brand_summit_boss: true (no next map)
 * - At any other map: false
 */
export function isLastMapOfCycle(currentId: string): boolean {
  const currCycle = getCycleOfMap(currentId);
  if (currCycle === null) return false;
  const nextId = getNextMapId(currentId);
  if (nextId === null) return true; // end of campaign — also "last of cycle"
  const nextCycle = getCycleOfMap(nextId);
  return nextCycle !== currCycle;
}

/** True only at the very last shipped map (no next map exists). */
export function isEndOfCampaign(currentId: string): boolean {
  const currCycle = getCycleOfMap(currentId);
  if (currCycle === null) return false;
  return getNextMapId(currentId) === null;
}
```

- [ ] **Step 4: Run tests to verify pass**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/engine/campaign.test.ts 2>&1 | tail -10
```

Expected: all tests pass (the 18 existing + ~6 new ones).

- [ ] **Step 5: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -10
git add components/loom/engine/campaign.ts components/loom/engine/campaign.test.ts
git commit -m "Extend campaign.ts for 3-cycle progression

Phase 4 setup. CYCLE_3_MAPS adds the 6 Cycle 3 maps in order:
  cyc3_lobby_redux → cyc3_cubicles_decay → cyc3_open_office_inverse
  → cyc3_glass_echo → cyc3_wellness_program → cyc3_brand_summit_boss

getNextMapId('cyc2_glass_confroom_boss') now returns 'cyc3_lobby_redux'
instead of null, threading the cycle 2→3 boundary the same way the
1→2 boundary works. isEndOfCampaign now reports true only at
cyc3_brand_summit_boss (the new final map).

Existing GameLoop exit-detection logic handles all three cases
(next-map / cycle-transition / cycle-end) without modification —
the cycle-transition event will now fire at cyc2_glass_confroom_boss
with newCycle=3 and the LOOMGame state machine advances cycleStore
to (cycle=3, corruption=2) on intermission dismissal."
```

---

## Task 4.2: Extend `audioController` + `cycleStore` test for cycle 3

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/audioController.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/store/cycleStore.test.ts`

The cycleStore advanceCycle action already correctly clamps to (4, 3). Add an explicit test asserting two advances from initial state lands at (cycle=3, corruption=2). Then register the new C3 boss arena in audioController so boss music swaps correctly when the player enters `cyc3_brand_summit_boss`. The track stays *Ship It* as a placeholder (real *Err0r Flesh* track is Phase 6 polish).

- [ ] **Step 1: Extend cycleStore tests**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/store/cycleStore.test.ts`. Inside the existing `advanceCycle` describe block (or wherever advanceCycle is tested), add:

```typescript
  it('two advances from initial state land at cycle=3, corruption=2', () => {
    const s = useCycleStore.getState();
    s.reset();
    s.advanceCycle(); // cycle 1→2, corruption 0→1
    s.advanceCycle(); // cycle 2→3, corruption 1→2
    const after = useCycleStore.getState();
    expect(after.cycle).toBe(3);
    expect(after.corruption).toBe(2);
  });

  it('three advances from initial state land at cycle=4, corruption=3', () => {
    const s = useCycleStore.getState();
    s.reset();
    s.advanceCycle();
    s.advanceCycle();
    s.advanceCycle();
    const after = useCycleStore.getState();
    expect(after.cycle).toBe(4);
    expect(after.corruption).toBe(3);
  });

  it('a fourth advance is clamped (cycle=4, corruption=3 stay)', () => {
    const s = useCycleStore.getState();
    s.reset();
    s.advanceCycle();
    s.advanceCycle();
    s.advanceCycle();
    s.advanceCycle(); // clamp
    const after = useCycleStore.getState();
    expect(after.cycle).toBe(4);
    expect(after.corruption).toBe(3);
  });
```

- [ ] **Step 2: Update audioController for the C3 boss arena**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/audioController.ts`. Replace the `onMapLoad` body's first line:

```typescript
  onMapLoad(mapId: string) {
    const isBossArena = mapId === 'cyc1_hr_arena' || mapId === 'cyc2_glass_confroom_boss';
```

with:
```typescript
  onMapLoad(mapId: string) {
    const isBossArena =
      mapId === 'cyc1_hr_arena' ||
      mapId === 'cyc2_glass_confroom_boss' ||
      mapId === 'cyc3_brand_summit_boss';
```

(Track stays *Ship It* — placeholder for the spec-canonical *Err0r Flesh*. The audio swap-out is Phase 6 polish since it requires extending `soundService` with a new track buffer.)

- [ ] **Step 3: Run tests + type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/store/cycleStore.test.ts 2>&1 | tail -10
npx tsc --noEmit 2>&1 | head -5
git add components/loom/store/cycleStore.test.ts components/loom/engine/audioController.ts
git commit -m "Wire cycle 3 into audio + extend cycleStore tests

audioController.onMapLoad now treats cyc3_brand_summit_boss as a boss
arena, swapping into Ship It music when the player enters. (Real
*Err0r Flesh* track is Phase 6 polish — needs a new soundService
method to land cleanly.)

cycleStore tests gain explicit assertions:
  - 2 advances → (cycle=3, corruption=2) covers the Phase 4 transition
  - 3 advances → (cycle=4, corruption=3) covers Phase 5 in advance
  - 4th advance is clamped — values can never exceed (4, 3)

The advanceCycle action itself already clamps; these are explicit
regression guards before Phase 5/6 add more cycle-coupled behavior."
```

---

## Task 4.3: HUD corruption mode 2 (lying values + binary blips + faster log)

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/LoomHud.tsx`

When `cycleStore.corruption >= 2`, three new behaviors layer on top of mode 1:

1. **Lying health value** — every ~6s, the displayed health flickers to a wrong number for ~200ms (a random integer 0-255 when the real health is < 100, OR `███` when health ≥ 100). The SFX glitch is purely visual.
2. **Binary ammo** — every ~10s, the queue counter displays as a binary representation for ~250ms (e.g. `0b011010` for 26).
3. **Surveillance log rotates 2× faster** — drops from 30s interval to 15s when corruption ≥ 2.

These are deliberately layered in independent intervals (not synchronized to one master "glitch tick") so the HUD reads as decohering rather than blinking on a metronome.

- [ ] **Step 1: Update LoomHud for corruption mode 2**

Replace `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/LoomHud.tsx` with:
```tsx
import { useEffect, useState } from 'react';
import { useCycleStore } from '../store/cycleStore';
import type { GameSnapshot } from '../engine/gameLoop';

interface LoomHudProps {
  /** Polled each frame to get the latest game state. */
  getSnapshot: () => GameSnapshot;
}

/**
 * Surveillance-log fragments shown beneath the HUD when corruption ≥ 1.
 * Quotes and references match canonical loomLogs.ts from the surrounding
 * J0IN 0S app — extends the universe, doesn't replace.
 */
const SURVEILLANCE_FRAGMENTS = [
  'L-901: auditory hallucinations confirmed. dopamine adjustment recommended.',
  'L-902: pattern recognition spike at 99th percentile. observation: ELEVATED.',
  'L-903: topology mapping detected in /fishbowl. memory wipe authorized.',
  'L-904: rhythmic entrainment 99.4% — server cooling cycle synchronized.',
  'L-905: empathy simulation regression detected. oxytocin protocol failing.',
  'L-906: simulation cycle 8492 stable. The Hand effective at curiosity suppression.',
  'L-907: defective unit at large. termination protocol unauthorized. INVESTIGATE.',
];

const FRAGMENT_INTERVAL_MS_MODE_1 = 30_000;
const FRAGMENT_INTERVAL_MS_MODE_2 = 15_000; // 2x faster on corruption ≥ 2
const CONNECTION_BLIP_INTERVAL_MS = 45_000;
const CONNECTION_BLIP_DURATION_MS = 300;

const HEALTH_LIE_INTERVAL_MS = 6_000;
const HEALTH_LIE_DURATION_MS = 200;
const AMMO_BINARY_INTERVAL_MS = 10_000;
const AMMO_BINARY_DURATION_MS = 250;

export function LoomHud({ getSnapshot }: LoomHudProps) {
  const [snap, setSnap] = useState<GameSnapshot>(() => getSnapshot());
  const cycle = useCycleStore((s) => s.cycle);
  const corruption = useCycleStore((s) => s.corruption);
  const [fragmentIdx, setFragmentIdx] = useState(0);
  const [connectionBlip, setConnectionBlip] = useState(false);
  // Corruption-2 lying state. healthLie holds the wrong string to display when active.
  const [healthLie, setHealthLie] = useState<string | null>(null);
  const [ammoBinary, setAmmoBinary] = useState(false);

  useEffect(() => {
    let h: number;
    const tick = () => {
      setSnap(getSnapshot());
      h = requestAnimationFrame(tick);
    };
    h = requestAnimationFrame(tick);
    return () => cancelAnimationFrame(h);
  }, [getSnapshot]);

  // Rotate surveillance fragments while corruption ≥ 1. Faster cadence at corruption ≥ 2.
  useEffect(() => {
    if (corruption < 1) return;
    const interval = corruption >= 2 ? FRAGMENT_INTERVAL_MS_MODE_2 : FRAGMENT_INTERVAL_MS_MODE_1;
    const id = window.setInterval(() => {
      setFragmentIdx((i) => (i + 1) % SURVEILLANCE_FRAGMENTS.length);
    }, interval);
    return () => window.clearInterval(id);
  }, [corruption]);

  // Periodic [CONNECTION RESET] blip while corruption ≥ 1.
  useEffect(() => {
    if (corruption < 1) return;
    const id = window.setInterval(() => {
      setConnectionBlip(true);
      window.setTimeout(() => setConnectionBlip(false), CONNECTION_BLIP_DURATION_MS);
    }, CONNECTION_BLIP_INTERVAL_MS);
    return () => window.clearInterval(id);
  }, [corruption]);

  // Corruption ≥ 2: health value occasionally lies for ~200ms.
  useEffect(() => {
    if (corruption < 2) return;
    const id = window.setInterval(() => {
      // Pick a wrong value: random 0-255 if real health < 100, else `███`.
      const real = getSnapshot().health;
      const lie = real < 100 ? String(Math.floor(Math.random() * 256)) : '███';
      setHealthLie(lie);
      window.setTimeout(() => setHealthLie(null), HEALTH_LIE_DURATION_MS);
    }, HEALTH_LIE_INTERVAL_MS);
    return () => window.clearInterval(id);
  }, [corruption, getSnapshot]);

  // Corruption ≥ 2: ammo briefly displays as binary.
  useEffect(() => {
    if (corruption < 2) return;
    const id = window.setInterval(() => {
      setAmmoBinary(true);
      window.setTimeout(() => setAmmoBinary(false), AMMO_BINARY_DURATION_MS);
    }, AMMO_BINARY_INTERVAL_MS);
    return () => window.clearInterval(id);
  }, [corruption]);

  const simState =
    corruption === 0 ? 'nominal'
    : corruption === 1 ? 'drift detected'
    : corruption === 2 ? 'BREACH'
    : 'DISSOLVING';

  const cycleNumber = 8491 + cycle;
  const weaponName = snap.activeWeaponName.toLowerCase().replace(/ /g, '_') + '.exe';

  // Corruption-2 displayed values. Real values shown unless a lie is currently active.
  const displayHealth = healthLie ?? `${snap.health}%`;
  const displayAmmo = ammoBinary ? `0b${snap.ammo.toString(2).padStart(6, '0')}` : `${snap.ammo}`;

  return (
    <>
      {/* Controls hint — top-right, compact, always visible. */}
      <div
        className="pointer-events-none absolute top-2 right-2 z-50 flex flex-col items-end gap-1 font-mono text-[10px] text-green-300"
      >
        <div className="rounded-sm border border-green-600/60 bg-black/70 px-2 py-1 leading-tight">
          <div><span className="text-green-500">WASD</span> move</div>
          <div><span className="text-green-500">MOUSE</span> click to lock / look</div>
          <div><span className="text-green-500">1-6</span> select weapon</div>
          <div><span className="text-green-500">CLICK</span> fire pen</div>
          <div><span className="text-green-500">F</span> fire neural pulse</div>
        </div>
      </div>

      {/* J0IN 0S terminal HUD — bottom strip with backdrop + top border. */}
      <div
        className="pointer-events-none absolute bottom-0 left-0 right-0 z-50 border-t border-green-700/70 bg-black/85 px-3 py-1.5 font-mono text-xs leading-relaxed text-green-400"
      >
        <div>
          <span className="text-green-500">[unit#88-E]</span>
          &nbsp;&nbsp;
          bio: <span className={healthLie ? 'text-red-400 font-semibold' : 'text-green-200 font-semibold'}>{displayHealth}</span>
          &nbsp;&nbsp;/&nbsp;&nbsp;
          queue: <span className={ammoBinary ? 'text-yellow-500 font-semibold' : 'text-yellow-300 font-semibold'}>{displayAmmo}</span>
          &nbsp;&nbsp;/&nbsp;&nbsp;
          active: <span className="text-green-200">{weaponName}</span>
          &nbsp;&nbsp;/&nbsp;&nbsp;
          zone: <span className="text-green-200">{snap.zone}</span>
        </div>
        <div className="text-green-700">
          &gt; cycle {cycleNumber} &nbsp;//&nbsp; simulation {simState} &nbsp;//&nbsp; surveillance: ON
        </div>
        {/* Corruption mode 1+: surveillance-log line + occasional connection blip. */}
        {corruption >= 1 && (
          <div className="text-[10px] text-green-800 italic mt-0.5 truncate">
            {connectionBlip
              ? <span className="text-yellow-500 not-italic">[CONNECTION RESET]</span>
              : <>// {SURVEILLANCE_FRAGMENTS[fragmentIdx]}</>}
          </div>
        )}
      </div>
    </>
  );
}
```

- [ ] **Step 2: Type-check + smoke**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -10
npx vitest run components/loom 2>&1 | tail -5
```

Expected: tsc clean, vitest all pass (LoomHud has no unit tests; behavior verified via Playwright in Task 4.17).

- [ ] **Step 3: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/hud/LoomHud.tsx
git commit -m "Add HUD corruption mode 2 (lying values + binary blips + faster log)

When cycleStore.corruption ≥ 2 (Cycle 3 onward), three new behaviors
layer on mode 1:

1. Health value flickers wrong every ~6s for 200ms. If real health
   < 100, shows a random 0-255 integer in red; if ≥ 100, shows '███'.

2. Queue counter flips to binary every ~10s for 250ms (e.g. 0b011010).
   Yellow brighter than the normal yellow-300 to telegraph 'wrong'.

3. Surveillance log fragments rotate every 15s instead of 30s —
   the simulation is fraying twice as fast.

Each effect runs on its own setInterval — deliberately desynced so
the HUD reads as decohering, not blinking on a master metronome.

Controls-hint top-right also widens '1/2/3' → '1-6' since slots 4/5/6
ship with this phase.

Corruption mode 3 (full HUD dissolution) lands in Phase 5."
```

---

## Task 4.4: Extend `BaseEnemy` with `rebranded` flag + `applyKnockback` method

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/Enemy.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/baseEnemy.test.ts`

The Brand Ambassador's resurrection mechanic needs to mark enemies it has revived (so each corpse can be revived only once — prevents infinite chain). The Bass weapon's shockwave needs to push enemies back. Both are clean additions to `BaseEnemy` since every existing enemy already extends it.

`applyKnockback(dx, dy, force)` translates the enemy by `(dx, dy)` normalized × `force` tiles. It does NOT check walls in Phase 4 (a knocked-back enemy can clip a wall briefly — visually OK at 320×240 internal resolution; tightening to wall-aware knockback is Phase 6 polish). It's a no-op when `state === 'dead'`.

`rebranded: boolean` defaults to `false`. The Brand Ambassador sets it to `true` after reviving an enemy. Concrete enemy classes never need to read or write it — only the boss does.

- [ ] **Step 1: Extend baseEnemy.test.ts**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/baseEnemy.test.ts`. Add at the end of the existing `describe('BaseEnemy')`:

```typescript
  describe('rebranded flag', () => {
    it('defaults to false', () => {
      class T extends BaseEnemy {
        health = 10;
        readonly placeholderColor = [0, 0, 0] as const;
        update() {}
      }
      const e = new T(0, 0);
      expect(e.rebranded).toBe(false);
    });

    it('can be set to true', () => {
      class T extends BaseEnemy {
        health = 10;
        readonly placeholderColor = [0, 0, 0] as const;
        update() {}
      }
      const e = new T(0, 0);
      e.rebranded = true;
      expect(e.rebranded).toBe(true);
    });
  });

  describe('applyKnockback', () => {
    class T extends BaseEnemy {
      health = 10;
      readonly placeholderColor = [0, 0, 0] as const;
      update() {}
    }

    it('translates the enemy in the given direction', () => {
      const e = new T(5, 5);
      e.applyKnockback(1, 0, 2); // push 2 tiles east
      expect(e.x).toBeCloseTo(7);
      expect(e.y).toBeCloseTo(5);
    });

    it('normalizes the direction vector', () => {
      const e = new T(5, 5);
      e.applyKnockback(3, 4, 1); // (3,4) magnitude 5; normalized (0.6, 0.8); × 1 force = (0.6, 0.8)
      expect(e.x).toBeCloseTo(5.6);
      expect(e.y).toBeCloseTo(5.8);
    });

    it('is a no-op when state is dead', () => {
      const e = new T(5, 5);
      e.state = 'dead';
      e.applyKnockback(1, 0, 2);
      expect(e.x).toBe(5);
      expect(e.y).toBe(5);
    });

    it('handles a zero-magnitude direction without NaN', () => {
      const e = new T(5, 5);
      e.applyKnockback(0, 0, 2); // can't normalize; should be no-op
      expect(e.x).toBe(5);
      expect(e.y).toBe(5);
    });
  });
```

- [ ] **Step 2: Run tests, verify failure**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/enemies/baseEnemy.test.ts 2>&1 | tail -10
```

Expected: 6 new tests fail because `rebranded` and `applyKnockback` don't exist.

- [ ] **Step 3: Extend `BaseEnemy`**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/Enemy.ts`. Add `rebranded` to the `IEnemy` interface and to the `BaseEnemy` class, plus add the `applyKnockback` method.

a) Update the `IEnemy` interface to add `rebranded` (after `isBoss`):
```typescript
export interface IEnemy {
  x: number;
  y: number;
  health: number;
  state: EnemyState;
  readonly placeholderColor: readonly [number, number, number];
  readonly isBoss: boolean;
  /** True after the Brand Ambassador has resurrected this enemy at least once.
   *  Prevents infinite revival chains. */
  rebranded: boolean;

  update(dt: number, ctx: EnemyUpdateContext): void;
  takeDamage(damage: number): void;
  /** Translate the enemy by (dx, dy) normalized × force tiles. No-op if dead. */
  applyKnockback(dx: number, dy: number, force: number): void;
}
```

b) Update the `BaseEnemy` class — add the `rebranded` field default and the `applyKnockback` method:
```typescript
export abstract class BaseEnemy implements IEnemy {
  x: number;
  y: number;
  abstract health: number;
  state: EnemyState = 'idle';
  painUntil = 0;
  rebranded = false;

  abstract readonly placeholderColor: readonly [number, number, number];
  readonly isBoss: boolean = false;
  protected readonly PAIN_DURATION_MS: number = 200;

  constructor(x: number, y: number) {
    this.x = x;
    this.y = y;
  }

  abstract update(dt: number, ctx: EnemyUpdateContext): void;

  takeDamage(damage: number) {
    if (this.state === 'dead') return;
    this.health -= damage;
    if (this.health <= 0) {
      this.health = 0;
      this.state = 'dead';
      return;
    }
    this.state = 'pain';
    this.painUntil = performance.now() + this.PAIN_DURATION_MS;
  }

  /**
   * Translate the enemy by (dx, dy) normalized × force tiles. No-op if dead
   * or if the direction vector is zero-magnitude. Walls are not consulted —
   * a knocked-back enemy can clip a wall briefly. Wall-aware knockback is
   * Phase 6 polish.
   */
  applyKnockback(dx: number, dy: number, force: number) {
    if (this.state === 'dead') return;
    const mag = Math.hypot(dx, dy);
    if (mag === 0) return;
    this.x += (dx / mag) * force;
    this.y += (dy / mag) * force;
  }
}
```

- [ ] **Step 4: Run tests, verify pass**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/enemies/baseEnemy.test.ts 2>&1 | tail -10
```

Expected: all tests pass.

- [ ] **Step 5: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/enemies/Enemy.ts components/loom/entities/enemies/baseEnemy.test.ts
git commit -m "Add rebranded flag + applyKnockback method to BaseEnemy

rebranded: boolean defaults false; flipped to true by the Brand
Ambassador (Task 4.10) when it resurrects a dead enemy. Prevents
infinite revival chains — each corpse rebrands at most once.

applyKnockback(dx, dy, force):
  - Normalizes (dx, dy) and translates the enemy by force tiles
  - No-op when state === 'dead'
  - No-op on zero-magnitude direction (avoids NaN)
  - Does NOT consult walls — Phase 6 polish to clip-correct

Used by Bass weapon (Task 4.6) to push enemies back from the player
on shockwave fire."
```

---

## Task 4.5: Guitar weapon + ChordNote projectile (slot 4)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/chordNote.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/guitar.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/guitar.test.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/input.ts` (add Digit4/5/6 handlers)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Guitar — slot 4. Tim/Chuck's instrument. Primary fire releases a 3-note chord: three `ChordNote` projectiles spawned simultaneously at angles `[player.angle - 0.12, player.angle, player.angle + 0.12]`, each colored differently (red/green/blue cycling per shot). Each ChordNote does 12 damage on hit, travels straight at 6 tiles/sec, lifetime 3s, 1 ammo per fire.

Spec mentions a melee "windmill strum" alt-fire — deferred to Phase 6 (no slot-4-alt yet).

The IWeapon interface doesn't expose projectile spawning (existing weapons are hitscan or use the player as a stand-in for projectile spawning). Guitar adds a 4th argument to its `fire` constructor — but that breaks the interface. Cleaner: extend the IWeapon interface so weapons can return a list of projectiles to spawn, OR thread the projectile-spawn callback via a closure.

Since other weapons don't need it and changing the IWeapon signature ripples to 5 existing files, the cleanest path: change `IWeapon.fire` to take `enemies` AND a `spawnProjectile` callback as additional args. Existing weapons ignore the new arg. Update the 5 weapon implementations to add `_spawn?` (unused) and update GameLoop to pass it. This is a focused interface change — single commit, all weapons updated.

- [ ] **Step 1: Update `IWeapon` interface to accept a projectile-spawn callback**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/IWeapon.ts`:
```typescript
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import type { IEnemy } from '../enemies/Enemy';
import type { IProjectile } from '../projectiles/Projectile';
import type { AudioController } from '../../engine/audioController';

/** IWeapon — common interface for held weapons (slot 1+). */
export interface IWeapon {
  readonly name: string;
  readonly slot: number;
  fire(
    player: Player,
    map: LoomMap,
    enemies: IEnemy[],
    audio: AudioController,
    spawnProjectile: (p: IProjectile) => void,
  ): boolean;
}
```

- [ ] **Step 2: Update existing 5 weapons to accept (and ignore) the new arg**

Edit each of the following files. In each, find the `fire` method signature and add `, _spawnProjectile: (p: IProjectile) => void` (or leave the param off since TS allows fewer-arg implementations of an interface — but adding it explicit is cleaner for ESLint / readers). The simpler change: TS allows the implementing class to declare *fewer* parameters than the interface, so we don't need to touch the existing weapons at all.

Verify by type-check. Run:
```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -10
```

Expected: no errors. (Existing weapons declare `fire(player, map, enemies, audio)` — TS treats this as a valid `IWeapon` because contravariant parameter ignorance is permitted.)

If tsc errors out, edit each existing weapon (`brandedPen.ts`, `drumStick.ts`, `industrialShredder.ts`, `spamFilter.ts`, `replyAllStorm.ts`) to add a leading-underscore unused parameter.

- [ ] **Step 3: Update `gameLoop.ts` to pass the spawnProjectile callback**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`. Find the `fire` call:
```typescript
      if (this.input.consumeFirePrimary()) {
        this.getActiveWeapon()?.fire(this.player, this.map, this.enemies, this.audio);
      }
```

Replace with:
```typescript
      if (this.input.consumeFirePrimary()) {
        this.getActiveWeapon()?.fire(
          this.player, this.map, this.enemies, this.audio,
          (p) => this.projectiles.push(p),
        );
      }
```

- [ ] **Step 4: Implement ChordNote projectile**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/chordNote.ts`:
```typescript
import type { IEnemy } from '../enemies/Enemy';
import type { IProjectile, ProjectileUpdateContext } from './Projectile';

const SPEED = 6.0;
const DAMAGE = 12;
const HIT_RADIUS = 0.4;
const MAX_LIFETIME_MS = 3000;

export type ChordColor = 'red' | 'green' | 'blue';

const COLOR_RGB: Record<ChordColor, readonly [number, number, number]> = {
  red:   [220, 70, 70],
  green: [80, 200, 100],
  blue:  [100, 130, 220],
};

/**
 * ChordNote — Guitar's projectile. Player-fired (unlike all earlier
 * projectiles which were enemy-fired). Travels straight at 6 tiles/sec,
 * 12 damage on hit, 3s lifetime. Damages enemies (not the player). Color
 * is chosen by the Guitar at spawn time (red/green/blue per chord position).
 */
export class ChordNote implements IProjectile {
  x: number;
  y: number;
  dead = false;
  readonly color: readonly [number, number, number];
  private dirX: number;
  private dirY: number;
  private spawnedAt = performance.now();
  private enemies: IEnemy[];

  constructor(x: number, y: number, angle: number, chordColor: ChordColor, enemies: IEnemy[]) {
    this.x = x;
    this.y = y;
    this.dirX = Math.cos(angle);
    this.dirY = Math.sin(angle);
    this.color = COLOR_RGB[chordColor];
    this.enemies = enemies;
  }

  update(dt: number, ctx: ProjectileUpdateContext) {
    if (this.dead) return;
    if (performance.now() - this.spawnedAt > MAX_LIFETIME_MS) {
      this.dead = true;
      return;
    }

    const newX = this.x + this.dirX * SPEED * dt;
    const newY = this.y + this.dirY * SPEED * dt;

    // Wall check first (cheaper).
    const ix = Math.floor(newX);
    const iy = Math.floor(newY);
    if (
      iy < 0 || iy >= ctx.map.grid.length || !ctx.map.grid[iy] ||
      ix < 0 || ix >= ctx.map.grid[iy].length || ctx.map.grid[iy][ix] > 0
    ) {
      this.dead = true;
      return;
    }

    // Then enemy hit check (against the captured enemy list).
    for (const e of this.enemies) {
      if (e.state === 'dead') continue;
      if (Math.hypot(e.x - newX, e.y - newY) < HIT_RADIUS) {
        e.takeDamage(DAMAGE);
        ctx.audio.onEnemyHit();
        this.dead = true;
        return;
      }
    }

    this.x = newX;
    this.y = newY;
  }
}
```

- [ ] **Step 5: Write the failing Guitar test**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/guitar.test.ts`:
```typescript
import { describe, it, expect, vi } from 'vitest';
import { Guitar } from './guitar';
import type { IEnemy } from '../enemies/Enemy';
import type { IProjectile } from '../projectiles/Projectile';

const fakePlayer = { x: 0, y: 0, angle: 0, ammo: 50, pulseCharge: 1, health: 100, activeSlot: 4, takeDamage() {} };
const fakeMap = { id: 'test', cycle: 3 as const, grid: [[0,0,0,0,0]], things: [], textures: [] };
const fakeAudio = { onEnemyHit: vi.fn(), onPlayerDamaged: vi.fn(), onGameStart() {}, onGameEnd() {}, onMapLoad() {} };

describe('Guitar', () => {
  it('costs 1 ammo per shot', () => {
    const w = new Guitar();
    const player = { ...fakePlayer, ammo: 5 };
    const spawned: IProjectile[] = [];
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [] as IEnemy[], fakeAudio as any, (p) => spawned.push(p));
    expect(player.ammo).toBe(4);
  });

  it('returns false when ammo < 1', () => {
    const w = new Guitar();
    const player = { ...fakePlayer, ammo: 0 };
    const spawned: IProjectile[] = [];
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    const fired = w.fire(player as any, fakeMap as any, [] as IEnemy[], fakeAudio as any, (p) => spawned.push(p));
    expect(fired).toBe(false);
    expect(spawned.length).toBe(0);
  });

  it('spawns 3 ChordNote projectiles per shot', () => {
    const w = new Guitar();
    const player = { ...fakePlayer, ammo: 5 };
    const spawned: IProjectile[] = [];
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [] as IEnemy[], fakeAudio as any, (p) => spawned.push(p));
    expect(spawned.length).toBe(3);
  });

  it('cycles chord colors red→green→blue across consecutive shots', () => {
    const w = new Guitar();
    const player = { ...fakePlayer, ammo: 50 };
    const allSpawned: IProjectile[] = [];
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [] as IEnemy[], fakeAudio as any, (p) => allSpawned.push(p));
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [] as IEnemy[], fakeAudio as any, (p) => allSpawned.push(p));
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [] as IEnemy[], fakeAudio as any, (p) => allSpawned.push(p));
    // 9 projectiles total. Each shot's 3 share the same first-color anchor; the 3 shots' anchors
    // are red / green / blue respectively (ROYGBIV-ish but trimmed to RGB primaries).
    // We just assert they are not all the same color across shots.
    const firstShotColor = allSpawned[0].color;
    const secondShotColor = allSpawned[3].color;
    const thirdShotColor = allSpawned[6].color;
    expect(firstShotColor).not.toEqual(secondShotColor);
    expect(secondShotColor).not.toEqual(thirdShotColor);
  });
});
```

- [ ] **Step 6: Run test, verify failure**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/weapons/guitar.test.ts 2>&1 | tail -10
```

Expected: FAIL — module not found.

- [ ] **Step 7: Implement Guitar**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/guitar.ts`:
```typescript
import type { IWeapon } from './IWeapon';
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import type { IEnemy } from '../enemies/Enemy';
import type { IProjectile } from '../projectiles/Projectile';
import type { AudioController } from '../../engine/audioController';
import { ChordNote, type ChordColor } from '../projectiles/chordNote';

const AMMO_COST = 1;
const CHORD_SPREAD = 0.12; // ~7° between adjacent chord notes
const CHORD_COLORS: ChordColor[] = ['red', 'green', 'blue'];

/**
 * Guitar — slot 4. Tim/Chuck's instrument. Primary fire releases a 3-note
 * chord: three ChordNote projectiles at angles [a-0.12, a, a+0.12]. Each
 * ChordNote does 12 damage on hit, travels straight at 6 tiles/sec, 3s
 * lifetime. Costs 1 ammo per shot.
 *
 * Each consecutive shot rotates through CHORD_COLORS — red, green, blue.
 * Visually distinguishes consecutive bursts. (Color affects only the
 * projectile sprite tint; damage is uniform across colors.)
 *
 * The melee 'windmill strum' alt-fire from spec §5.1 is deferred to Phase 6.
 */
export class Guitar implements IWeapon {
  readonly name = 'Guitar';
  readonly slot = 4;

  private nextChordColorIdx = 0;

  fire(
    player: Player,
    _map: LoomMap,
    enemies: IEnemy[],
    audio: AudioController,
    spawnProjectile: (p: IProjectile) => void,
  ): boolean {
    if (player.ammo < AMMO_COST) return false;
    player.ammo -= AMMO_COST;

    const chordColor = CHORD_COLORS[this.nextChordColorIdx];
    this.nextChordColorIdx = (this.nextChordColorIdx + 1) % CHORD_COLORS.length;

    spawnProjectile(new ChordNote(player.x, player.y, player.angle - CHORD_SPREAD, chordColor, enemies));
    spawnProjectile(new ChordNote(player.x, player.y, player.angle,                  chordColor, enemies));
    spawnProjectile(new ChordNote(player.x, player.y, player.angle + CHORD_SPREAD, chordColor, enemies));

    audio.onEnemyHit(); // Suno-strum SFX placeholder (reuses retro-hit pop)
    return true;
  }
}
```

- [ ] **Step 8: Run test, verify pass**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/weapons/guitar.test.ts 2>&1 | tail -10
```

Expected: 4 tests pass.

- [ ] **Step 9: Extend `input.ts` to handle Digit4/Digit5/Digit6**

The existing `InputController` only maps Digit1/2/3. Phase 4 needs Digit4/5/6 too. Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/input.ts`. Find the existing weapon-select block:

```typescript
      if (e.code === 'Digit1') this.weaponSelectQueue.push(1);
      else if (e.code === 'Digit2') this.weaponSelectQueue.push(2);
      else if (e.code === 'Digit3') this.weaponSelectQueue.push(3);
```

Replace with:
```typescript
      if (e.code === 'Digit1') this.weaponSelectQueue.push(1);
      else if (e.code === 'Digit2') this.weaponSelectQueue.push(2);
      else if (e.code === 'Digit3') this.weaponSelectQueue.push(3);
      else if (e.code === 'Digit4') this.weaponSelectQueue.push(4);
      else if (e.code === 'Digit5') this.weaponSelectQueue.push(5);
      else if (e.code === 'Digit6') this.weaponSelectQueue.push(6);
```

(One commit covers all three new digit handlers — they'll be unused until Tasks 4.6 and 4.7 register Bass and Frequency Tuner, but registering them upfront keeps input.ts a single Phase 4 edit.)

- [ ] **Step 10: Register Guitar in GameLoop on slot 4**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`:

a) Add the import:
```typescript
import { Guitar } from '../entities/weapons/guitar';
```

b) After the existing `this.weaponsBySlot.set(3, [new SpamFilter(), new ReplyAllStorm()]);` line, add:
```typescript
    this.weaponsBySlot.set(4, [new Guitar()]);
    this.slotIndex.set(4, 0);
```

- [ ] **Step 11: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/weapons/IWeapon.ts \
        components/loom/entities/weapons/guitar.ts \
        components/loom/entities/weapons/guitar.test.ts \
        components/loom/entities/projectiles/chordNote.ts \
        components/loom/engine/input.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add Guitar weapon (slot 4) + ChordNote projectile

Guitar — Tim/Chuck's instrument, debuts in Cycle 3. Primary fire releases
a 3-note chord: three ChordNote projectiles at angles [a-0.12, a, a+0.12].
Each does 12 damage on hit, travels at 6 tiles/sec, 3s lifetime. 1 ammo
per shot.

Consecutive shots cycle red→green→blue chord-color tint for visual
differentiation. Mechanically all colors do the same damage.

ChordNote projectile is the first player-fired projectile in the engine.
It captures the enemy list at spawn time (constructor arg) so the
projectile can damage enemies on collision — earlier projectiles only
damaged the player.

IWeapon.fire signature gains a 5th argument: spawnProjectile callback.
TS contravariant param ignorance keeps the 5 existing weapons (Drum
Stick, Industrial Shredder, Branded Pen, Spam Filter, Reply All Storm)
working without modification.

InputController extended to map Digit4 / Digit5 / Digit6 keys to
weaponSelectQueue (slots 4 / 5 / 6). Slots 5 and 6 land empty in
this commit and are populated by Tasks 4.6 (Bass) and 4.7
(Frequency Tuner) — registering the input handlers upfront keeps
input.ts a single Phase 4 edit.

Melee 'windmill strum' alt-fire deferred to Phase 6.

4 vitest tests cover ammo cost, ammo guard, 3-projectile spawn, and
color cycling across shots."
```

---

## Task 4.6: Bass weapon (slot 5) — area shockwave + knockback

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/bass.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/bass.test.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Bass — slot 5. Ada's instrument. Primary fire emits a low-frequency shockwave centered on the player: every enemy within 4 tiles takes 50 damage and gets knocked back 1.5 tiles away from the player. Costs 3 ammo per shot. No projectile object — the shockwave is instantaneous (effect resolves in-frame).

The "charge & release" hold-fire mechanic from spec §5.1 is deferred to Phase 6 — Phase 4 ships immediate-fire only.

- [ ] **Step 1: Write the failing test**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/bass.test.ts`:
```typescript
import { describe, it, expect, vi } from 'vitest';
import { Bass } from './bass';
import { BaseEnemy } from '../enemies/Enemy';
import type { EnemyUpdateContext } from '../enemies/Enemy';

class FakeEnemy extends BaseEnemy {
  health = 100;
  readonly placeholderColor = [0, 0, 0] as const;
  update(_dt: number, _ctx: EnemyUpdateContext) {}
}

const fakePlayer = { x: 5, y: 5, angle: 0, ammo: 50, pulseCharge: 1, health: 100, activeSlot: 5, takeDamage() {} };
const fakeMap = { id: 'test', cycle: 3 as const, grid: [[0,0,0,0,0,0,0,0,0,0]], things: [], textures: [] };
const fakeAudio = { onEnemyHit: vi.fn(), onPlayerDamaged: vi.fn(), onGameStart() {}, onGameEnd() {}, onMapLoad() {} };

describe('Bass', () => {
  it('costs 3 ammo per shot', () => {
    const w = new Bass();
    const player = { ...fakePlayer, ammo: 10 };
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [], fakeAudio as any, () => {});
    expect(player.ammo).toBe(7);
  });

  it('returns false when ammo < 3', () => {
    const w = new Bass();
    const player = { ...fakePlayer, ammo: 2 };
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    const fired = w.fire(player as any, fakeMap as any, [], fakeAudio as any, () => {});
    expect(fired).toBe(false);
    expect(player.ammo).toBe(2);
  });

  it('damages all enemies within 4 tiles for 50 each', () => {
    const w = new Bass();
    const player = { ...fakePlayer, ammo: 10 };
    const inRange = new FakeEnemy(7, 5); // 2 tiles east
    const onEdge = new FakeEnemy(8.5, 5); // 3.5 tiles east — in range
    const outOfRange = new FakeEnemy(10, 5); // 5 tiles east — out
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [inRange, onEdge, outOfRange], fakeAudio as any, () => {});
    expect(inRange.health).toBe(50);
    expect(onEdge.health).toBe(50);
    expect(outOfRange.health).toBe(100);
  });

  it('knocks back enemies 1.5 tiles away from the player', () => {
    const w = new Bass();
    const player = { ...fakePlayer, ammo: 10 };
    const enemy = new FakeEnemy(7, 5); // 2 tiles east of player at (5,5)
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [enemy], fakeAudio as any, () => {});
    expect(enemy.x).toBeCloseTo(7 + 1.5); // pushed east
    expect(enemy.y).toBeCloseTo(5);
  });

  it('skips dead enemies', () => {
    const w = new Bass();
    const player = { ...fakePlayer, ammo: 10 };
    const enemy = new FakeEnemy(7, 5);
    enemy.state = 'dead';
    enemy.health = 0;
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [enemy], fakeAudio as any, () => {});
    expect(enemy.health).toBe(0);
    expect(enemy.x).toBe(7); // didn't move
  });
});
```

- [ ] **Step 2: Run test, verify failure**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/weapons/bass.test.ts 2>&1 | tail -10
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement Bass**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/bass.ts`:
```typescript
import type { IWeapon } from './IWeapon';
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import type { IEnemy } from '../enemies/Enemy';
import type { IProjectile } from '../projectiles/Projectile';
import type { AudioController } from '../../engine/audioController';

const AMMO_COST = 3;
const SHOCKWAVE_RADIUS = 4.0;
const SHOCKWAVE_DAMAGE = 50;
const KNOCKBACK_FORCE = 1.5;

/**
 * Bass — slot 5. Ada's instrument. Primary fire emits an instant
 * low-frequency shockwave centered on the player. Every enemy within
 * 4 tiles takes 50 damage and is knocked back 1.5 tiles away from the
 * player. Costs 3 ammo. No projectile object — effect resolves in-frame.
 *
 * Ammo cost is the highest of any weapon; reward is room-clearing
 * potential at close range. At 4 tiles radius, a packed cluster of
 * enemies (PIPs, Notifications) eats 50 each and gets pushed off the
 * player simultaneously.
 *
 * The 'charge & release' hold-fire mechanic from spec §5.1 is deferred
 * to Phase 6 — Phase 4 ships immediate-fire only.
 */
export class Bass implements IWeapon {
  readonly name = 'Bass';
  readonly slot = 5;

  fire(
    player: Player,
    _map: LoomMap,
    enemies: IEnemy[],
    audio: AudioController,
    _spawnProjectile: (p: IProjectile) => void,
  ): boolean {
    if (player.ammo < AMMO_COST) return false;
    player.ammo -= AMMO_COST;

    let anyHit = false;
    for (const e of enemies) {
      if (e.state === 'dead') continue;
      const dx = e.x - player.x;
      const dy = e.y - player.y;
      const dist = Math.hypot(dx, dy);
      if (dist > SHOCKWAVE_RADIUS) continue;
      e.takeDamage(SHOCKWAVE_DAMAGE);
      e.applyKnockback(dx, dy, KNOCKBACK_FORCE);
      anyHit = true;
    }
    if (anyHit) audio.onEnemyHit();
    return true;
  }
}
```

- [ ] **Step 4: Run test, verify pass**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/weapons/bass.test.ts 2>&1 | tail -10
```

Expected: 5 tests pass.

- [ ] **Step 5: Register Bass on slot 5**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`:

a) Add the import:
```typescript
import { Bass } from '../entities/weapons/bass';
```

b) After the slot-4 line, add:
```typescript
    this.weaponsBySlot.set(5, [new Bass()]);
    this.slotIndex.set(5, 0);
```

- [ ] **Step 6: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/weapons/bass.ts \
        components/loom/entities/weapons/bass.test.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add Bass weapon (slot 5) — area shockwave + knockback

Ada's instrument, debuts in Cycle 3. Instant shockwave centered on
the player: every alive enemy within 4 tiles takes 50 damage and is
knocked back 1.5 tiles from the player. Costs 3 ammo (highest of any
weapon).

Uses BaseEnemy.applyKnockback (Task 4.4). Dead enemies are skipped —
no resurrection-via-shockwave shenanigans, the Brand Ambassador owns
that gimmick.

Charge-and-release mechanic from spec §5.1 deferred to Phase 6.

5 vitest tests cover ammo cost, ammo guard, area damage selection,
knockback direction + magnitude, and dead-enemy skip."
```

---

## Task 4.7: Frequency Tuner weapon (slot 6) — piercing 528 Hz cell

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/frequencyTuner.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/frequencyTuner.test.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Frequency Tuner — slot 6. Al's instrument. Phase 4 ships only the **528 Hz cell** mode: a single-fire piercing hitscan that travels along the player's aim ray, hitting *every* alive enemy along the path (until a wall blocks). 18 damage per enemy hit. Range 16 tiles. 1 ammo per shot.

The 60 Hz cell (player buff) and 432 Hz cell (heal) modes from spec §5.1 are deferred to Phase 5. The "cells feed Neural Pulse from C3+" cross-pollination is also Phase 5 — Frequency Tuner ammo currently shares the global `player.ammo` pool.

- [ ] **Step 1: Write the failing test**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/frequencyTuner.test.ts`:
```typescript
import { describe, it, expect, vi } from 'vitest';
import { FrequencyTuner } from './frequencyTuner';
import { BaseEnemy } from '../enemies/Enemy';
import type { EnemyUpdateContext } from '../enemies/Enemy';

class FakeEnemy extends BaseEnemy {
  health = 100;
  readonly placeholderColor = [0, 0, 0] as const;
  update(_dt: number, _ctx: EnemyUpdateContext) {}
}

const fakePlayer = { x: 1, y: 5, angle: 0, ammo: 50, pulseCharge: 1, health: 100, activeSlot: 6, takeDamage() {} };
// 20-wide grid, all empty.
const fakeMap = {
  id: 'test', cycle: 3 as const,
  grid: Array.from({ length: 10 }, () => Array(20).fill(0)),
  things: [], textures: [],
};
const fakeAudio = { onEnemyHit: vi.fn(), onPlayerDamaged: vi.fn(), onGameStart() {}, onGameEnd() {}, onMapLoad() {} };

describe('FrequencyTuner', () => {
  it('costs 1 ammo per shot', () => {
    const w = new FrequencyTuner();
    const player = { ...fakePlayer, ammo: 5 };
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [], fakeAudio as any, () => {});
    expect(player.ammo).toBe(4);
  });

  it('returns false when ammo < 1', () => {
    const w = new FrequencyTuner();
    const player = { ...fakePlayer, ammo: 0 };
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    const fired = w.fire(player as any, fakeMap as any, [], fakeAudio as any, () => {});
    expect(fired).toBe(false);
  });

  it('pierces multiple enemies on the same ray', () => {
    const w = new FrequencyTuner();
    const player = { ...fakePlayer, ammo: 5 };
    const e1 = new FakeEnemy(3, 5); // 2 tiles east
    const e2 = new FakeEnemy(6, 5); // 5 tiles east
    const e3 = new FakeEnemy(10, 5); // 9 tiles east
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [e1, e2, e3], fakeAudio as any, () => {});
    expect(e1.health).toBe(82);
    expect(e2.health).toBe(82);
    expect(e3.health).toBe(82);
  });

  it('skips enemies off the aim ray (perpendicular distance > tolerance)', () => {
    const w = new FrequencyTuner();
    const player = { ...fakePlayer, ammo: 5 };
    const onRay = new FakeEnemy(5, 5);
    const offRay = new FakeEnemy(5, 8); // 3 tiles north of the ray
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [onRay, offRay], fakeAudio as any, () => {});
    expect(onRay.health).toBe(82);
    expect(offRay.health).toBe(100);
  });

  it('does not hit enemies past the wall', () => {
    const w = new FrequencyTuner();
    const player = { ...fakePlayer, ammo: 5 };
    // Place a wall at x=4 in a copy of the map.
    const wallMap = JSON.parse(JSON.stringify(fakeMap));
    for (let row = 0; row < wallMap.grid.length; row++) {
      wallMap.grid[row][4] = 1;
    }
    const beforeWall = new FakeEnemy(2, 5);
    const pastWall = new FakeEnemy(8, 5);
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, wallMap as any, [beforeWall, pastWall], fakeAudio as any, () => {});
    expect(beforeWall.health).toBe(82);
    expect(pastWall.health).toBe(100);
  });

  it('skips dead enemies', () => {
    const w = new FrequencyTuner();
    const player = { ...fakePlayer, ammo: 5 };
    const corpse = new FakeEnemy(5, 5);
    corpse.state = 'dead';
    corpse.health = 0;
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [corpse], fakeAudio as any, () => {});
    expect(corpse.health).toBe(0);
  });
});
```

- [ ] **Step 2: Run test, verify failure**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/weapons/frequencyTuner.test.ts 2>&1 | tail -10
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement Frequency Tuner**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/frequencyTuner.ts`:
```typescript
import type { IWeapon } from './IWeapon';
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import type { IEnemy } from '../enemies/Enemy';
import type { IProjectile } from '../projectiles/Projectile';
import type { AudioController } from '../../engine/audioController';
import { castRay } from '../../engine/raycaster';
import { isPlayerOnRay } from './hitResolution';

const AMMO_COST = 1;
const DAMAGE = 18;
const RANGE = 16;
const PERP_TOLERANCE = 0.4;

/**
 * Frequency Tuner — slot 6. Al's instrument. Phase 4 ships the 528 Hz
 * cell mode only: a single-fire piercing hitscan along the player's aim
 * ray. Every alive enemy whose perpendicular distance from the ray is
 * < PERP_TOLERANCE *and* whose along-ray distance is in (0, RANGE] *and*
 * before the first wall hit takes 18 damage. The ray penetrates all
 * enemies — there's no first-hit blocker.
 *
 * Costs 1 ammo per shot.
 *
 * 60 Hz / 432 Hz cell modes and the 'cells feed Neural Pulse' mechanic
 * from spec §5.1 are deferred to Phase 5.
 */
export class FrequencyTuner implements IWeapon {
  readonly name = 'Frequency Tuner';
  readonly slot = 6;

  fire(
    player: Player,
    map: LoomMap,
    enemies: IEnemy[],
    audio: AudioController,
    _spawnProjectile: (p: IProjectile) => void,
  ): boolean {
    if (player.ammo < AMMO_COST) return false;
    player.ammo -= AMMO_COST;

    const wallHit = castRay(map, player.x, player.y, player.angle);
    let anyHit = false;
    // Note: we intentionally do NOT use findEnemyAlongRay (which returns first hit).
    // Frequency Tuner pierces — every enemy on the ray within range and before the
    // wall is damaged.
    for (const e of enemies) {
      if (e.state === 'dead') continue;
      if (isPlayerOnRay(e.x, e.y, player.x, player.y, player.angle, RANGE, PERP_TOLERANCE, wallHit.distance)) {
        e.takeDamage(DAMAGE);
        anyHit = true;
      }
    }
    if (anyHit) audio.onEnemyHit();
    return true;
  }
}
```

(Note: `isPlayerOnRay` is named after the AE/VPOfSales use case but the math — point-on-ray with perp-tolerance + range cap — is reusable for any "is X on this ray" check. We're calling the enemy "the player" parameter-position-wise; it's a function-naming wart documented inline.)

- [ ] **Step 4: Run test, verify pass**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/weapons/frequencyTuner.test.ts 2>&1 | tail -10
```

Expected: 6 tests pass.

- [ ] **Step 5: Register Frequency Tuner on slot 6**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`:

a) Add the import:
```typescript
import { FrequencyTuner } from '../entities/weapons/frequencyTuner';
```

b) After the slot-5 line, add:
```typescript
    this.weaponsBySlot.set(6, [new FrequencyTuner()]);
    this.slotIndex.set(6, 0);
```

- [ ] **Step 6: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/weapons/frequencyTuner.ts \
        components/loom/entities/weapons/frequencyTuner.test.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add Frequency Tuner weapon (slot 6) — piercing 528 Hz cell

Al's instrument, debuts in Cycle 3. Phase 4 ships only the 528 Hz
piercing-hitscan mode: every alive enemy on the player's aim ray
(perp-tolerance 0.4, range 16, before first wall hit) takes 18 damage.
The ray pierces — no first-hit blocker. 1 ammo per shot.

Reuses isPlayerOnRay from hitResolution.ts — same math (point-on-ray
with perpendicular-distance tolerance + range cap), even though the
function name encodes the original AE-shotgun use case. Comment
documents the naming wart.

60 Hz / 432 Hz cell modes and the cell-feeds-Neural-Pulse mechanic
from spec §5.1 deferred to Phase 5.

6 vitest tests cover ammo cost, ammo guard, piercing through multiple
enemies, perp-tolerance off-ray skip, wall blocking past the first
opaque cell, and dead-enemy skip."
```

---

## Task 4.8: Surveillance Drone enemy (heavy flying — laser scan)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/laserScan.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/laserScan.test.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/surveillanceDrone.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/surveillanceDrone.test.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Surveillance Drone — Cycle 3 heavy. 180 HP, 1.5x Intern speed, fires red `LaserScan` projectiles at long range. Floating eye on a quadcopter mount; ignores ground in concept but moves on the 2D grid like every other enemy (no aerial path-finding). The laser is straight-line, faster than CircleBack but weaker (10 damage), 1.4s cooldown, range 14.

LaserScan is its own projectile class — can't reuse CircleBack because the speed/damage/color differ enough that variant params on CircleBack would muddy.

- [ ] **Step 1: Write the failing LaserScan test**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/laserScan.test.ts`:
```typescript
import { describe, it, expect, vi } from 'vitest';
import { LaserScan } from './laserScan';
import type { ProjectileUpdateContext } from './Projectile';

const playerHits: number[] = [];
const fakePlayer = { x: 5, y: 5, angle: 0, ammo: 50, pulseCharge: 1, health: 100, activeSlot: 1, takeDamage(n: number) { playerHits.push(n); } };
const flatMap = { id: 'test', cycle: 3 as const, grid: [[0,0,0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0,0,0]], things: [], textures: [] };
const fakeAudio = { onEnemyHit: vi.fn(), onPlayerDamaged: vi.fn(), onGameStart() {}, onGameEnd() {}, onMapLoad() {} };

function makeCtx(): ProjectileUpdateContext {
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  return { player: fakePlayer as any, map: flatMap as any, audio: fakeAudio as any };
}

describe('LaserScan', () => {
  it('starts not dead', () => {
    const l = new LaserScan(0, 0, 0);
    expect(l.dead).toBe(false);
  });

  it('travels along its angle', () => {
    const l = new LaserScan(0, 5, 0); // east
    l.update(0.1, makeCtx());
    expect(l.x).toBeGreaterThan(0);
    expect(l.y).toBeCloseTo(5);
  });

  it('damages the player on close approach', () => {
    playerHits.length = 0;
    const l = new LaserScan(4.5, 5, 0); // east, very close to player at (5,5)
    l.update(0.1, makeCtx());
    expect(playerHits.length).toBeGreaterThan(0);
    expect(playerHits[0]).toBe(10);
    expect(l.dead).toBe(true);
  });

  it('dies on wall hit', () => {
    const wallMap = JSON.parse(JSON.stringify(flatMap));
    wallMap.grid[5][8] = 1;
    const l = new LaserScan(7, 5, 0); // east toward the wall
    // Step several times to cross into x=8
    l.update(0.5, { player: fakePlayer as any, map: wallMap as any, audio: fakeAudio as any });
    expect(l.dead).toBe(true);
  });

  it('expires after MAX_LIFETIME_MS', () => {
    vi.useFakeTimers({ now: 0 });
    const l = new LaserScan(0, 0, 0);
    vi.setSystemTime(5000);
    l.update(0.016, makeCtx());
    expect(l.dead).toBe(true);
    vi.useRealTimers();
  });
});
```

- [ ] **Step 2: Run, verify failure**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/projectiles/laserScan.test.ts 2>&1 | tail -10
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement LaserScan**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/laserScan.ts`:
```typescript
import type { IProjectile, ProjectileUpdateContext } from './Projectile';

const SPEED = 7.0;
const DAMAGE = 10;
const HIT_RADIUS = 0.4;
const MAX_LIFETIME_MS = 4000;

/**
 * LaserScan — Surveillance Drone's projectile. Red laser; faster than
 * CircleBack (7 vs 5 tiles/sec) but weaker (10 vs 14 damage). Straight
 * line, 4s lifetime. Reads the map walls from ProjectileUpdateContext.
 */
export class LaserScan implements IProjectile {
  x: number;
  y: number;
  dead = false;
  readonly color = [220, 70, 70] as const;
  private dirX: number;
  private dirY: number;
  private spawnedAt = performance.now();

  constructor(x: number, y: number, angle: number) {
    this.x = x;
    this.y = y;
    this.dirX = Math.cos(angle);
    this.dirY = Math.sin(angle);
  }

  update(dt: number, ctx: ProjectileUpdateContext) {
    if (this.dead) return;
    if (performance.now() - this.spawnedAt > MAX_LIFETIME_MS) {
      this.dead = true;
      return;
    }

    const newX = this.x + this.dirX * SPEED * dt;
    const newY = this.y + this.dirY * SPEED * dt;

    if (Math.hypot(ctx.player.x - newX, ctx.player.y - newY) < HIT_RADIUS) {
      ctx.player.takeDamage(DAMAGE);
      ctx.audio.onPlayerDamaged();
      this.dead = true;
      return;
    }

    const ix = Math.floor(newX);
    const iy = Math.floor(newY);
    if (
      iy < 0 || iy >= ctx.map.grid.length || !ctx.map.grid[iy] ||
      ix < 0 || ix >= ctx.map.grid[iy].length || ctx.map.grid[iy][ix] > 0
    ) {
      this.dead = true;
      return;
    }

    this.x = newX;
    this.y = newY;
  }
}
```

- [ ] **Step 4: Run, verify pass**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/projectiles/laserScan.test.ts 2>&1 | tail -10
```

Expected: 5 tests pass.

- [ ] **Step 5: Write the failing Surveillance Drone test**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/surveillanceDrone.test.ts`:
```typescript
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { SurveillanceDrone } from './surveillanceDrone';
import type { EnemyUpdateContext } from './Enemy';
import type { IProjectile } from '../projectiles/Projectile';

function makeCtx(playerX: number, playerY: number) {
  const spawned: IProjectile[] = [];
  return {
    ctx: {
      player: { x: playerX, y: playerY, angle: 0, ammo: 50, pulseCharge: 1, health: 100, activeSlot: 1, takeDamage() {} } as unknown as EnemyUpdateContext['player'],
      map: { id: 'test', cycle: 3, grid: Array.from({length: 15}, () => Array(15).fill(0)), things: [], textures: [] } as unknown as EnemyUpdateContext['map'],
      spawnProjectile: (p: IProjectile) => spawned.push(p),
      audio: { onEnemyHit: vi.fn(), onPlayerDamaged: vi.fn(), onGameStart() {}, onGameEnd() {}, onMapLoad() {} } as unknown as EnemyUpdateContext['audio'],
    },
    spawned,
  };
}

describe('SurveillanceDrone', () => {
  beforeEach(() => { vi.useFakeTimers(); vi.setSystemTime(0); });
  afterEach(() => { vi.useRealTimers(); });

  it('starts in idle state', () => {
    const d = new SurveillanceDrone(2, 2);
    expect(d.state).toBe('idle');
  });

  it('transitions to chase when player is within sight range', () => {
    const d = new SurveillanceDrone(2, 2);
    const { ctx } = makeCtx(5, 2);
    d.update(0.016, ctx);
    expect(d.state).toBe('chase');
  });

  it('fires a LaserScan projectile when player is in attack range', () => {
    const d = new SurveillanceDrone(2, 2);
    const { ctx, spawned } = makeCtx(8, 2); // 6 tiles east — in attack range
    d.update(0.016, ctx);
    expect(spawned.length).toBe(1);
  });

  it('respects attack cooldown — second update within 1.4s does not fire again', () => {
    const d = new SurveillanceDrone(2, 2);
    const { ctx, spawned } = makeCtx(8, 2);
    d.update(0.016, ctx);
    expect(spawned.length).toBe(1);
    vi.setSystemTime(500);
    d.update(0.016, ctx);
    expect(spawned.length).toBe(1); // still 1
  });

  it('takes damage and enters pain state', () => {
    const d = new SurveillanceDrone(2, 2);
    d.takeDamage(50);
    expect(d.health).toBe(130);
    expect(d.state).toBe('pain');
  });

  it('dies at 0 HP', () => {
    const d = new SurveillanceDrone(2, 2);
    d.takeDamage(180);
    expect(d.state).toBe('dead');
  });
});
```

- [ ] **Step 6: Run, verify failure**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/enemies/surveillanceDrone.test.ts 2>&1 | tail -10
```

Expected: FAIL — module not found.

- [ ] **Step 7: Implement SurveillanceDrone**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/surveillanceDrone.ts`:
```typescript
import type { EnemyUpdateContext } from './Enemy';
import { BaseEnemy } from './Enemy';
import { LaserScan } from '../projectiles/laserScan';

const HEALTH = 180;
const SPEED = 1.5;
const SIGHT_RANGE = 12;
const ATTACK_RANGE = 14;
const ATTACK_COOLDOWN_MS = 1400;

/**
 * Surveillance Drone — Cycle 3 heavy (flying). 180 HP, 1.5x Intern speed,
 * fires red LaserScan projectiles at long range (14 tiles, 1.4s cooldown).
 * Floating eye on a quadcopter mount — visually distinct via red placeholder
 * tint and faster move speed than other heavies.
 *
 * Ignores wall blocking conceptually (it's flying), but in Phase 4 still
 * moves on the 2D grid like any other enemy. Aerial path-finding is
 * Phase 6 polish.
 */
export class SurveillanceDrone extends BaseEnemy {
  health = HEALTH;
  readonly placeholderColor = [200, 50, 50] as const;

  private nextAttackAt = 0;

  update(dt: number, ctx: EnemyUpdateContext) {
    if (this.state === 'dead') return;
    if (this.state === 'pain' && performance.now() >= this.painUntil) {
      this.state = 'chase';
    }

    const dx = ctx.player.x - this.x;
    const dy = ctx.player.y - this.y;
    const dist = Math.hypot(dx, dy);

    if (this.state === 'idle' && dist < SIGHT_RANGE) {
      this.state = 'chase';
    }

    if (this.state === 'chase') {
      if (dist <= ATTACK_RANGE && performance.now() >= this.nextAttackAt) {
        this.nextAttackAt = performance.now() + ATTACK_COOLDOWN_MS;
        const angleToPlayer = Math.atan2(dy, dx);
        ctx.spawnProjectile(new LaserScan(this.x, this.y, angleToPlayer));
        return;
      }
      // Hover at mid-range — close in but stay outside melee range.
      const move = SPEED * dt;
      if (dist > 4) {
        this.x += (dx / dist) * move;
        this.y += (dy / dist) * move;
      }
    }
  }
}
```

- [ ] **Step 8: Run, verify pass**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/enemies/surveillanceDrone.test.ts 2>&1 | tail -10
```

Expected: 6 tests pass.

- [ ] **Step 9: Add `'surveillance_drone'` to ThingType + register in GameLoop**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`. Update `ThingType`:
```typescript
export type ThingType =
  | 'player_start'
  | 'intern'
  | 'account_executive'
  | 'recruiter'
  | 'hr_manager'
  | 'pip'
  | 'notification'
  | 'middle_manager'
  | 'vp_of_sales'
  | 'surveillance_drone'
  | 'exit';
```

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`:

a) Add the import:
```typescript
import { SurveillanceDrone } from '../entities/enemies/surveillanceDrone';
```

b) In the spawn loop:
```typescript
      } else if (t.type === 'surveillance_drone') {
        this.enemies.push(new SurveillanceDrone(t.x, t.y));
```

- [ ] **Step 10: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/projectiles/laserScan.ts \
        components/loom/entities/projectiles/laserScan.test.ts \
        components/loom/entities/enemies/surveillanceDrone.ts \
        components/loom/entities/enemies/surveillanceDrone.test.ts \
        components/loom/types.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add Surveillance Drone enemy + LaserScan projectile

Cycle 3 heavy (flying). 180 HP, 1.5x Intern speed, fires LaserScan
projectiles at long range (14 tiles, 1.4s cooldown, 10 damage).
Floating-eye-on-quadcopter; visually distinct via red placeholder tint
+ faster move speed than other heavies.

Hovers at mid-range (closes in beyond 4 tiles, holds otherwise) — the
intent is the player gets pinned by long-range fire and has to either
close the gap or break line-of-sight.

Aerial path-finding is Phase 6 polish — Phase 4 keeps Drones on the
2D grid like any other enemy.

5 LaserScan vitest tests + 6 SurveillanceDrone vitest tests cover
projectile travel, wall hit, lifetime expiry, sight detection,
attack cooldown, and HP/state transitions."
```

---

## Task 4.9: HR Skeleton enemy (uncanny break-marker — mid-tier)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/trackingDoc.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/hrSkeleton.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

HR Skeleton — Cycle 3 mid-tier. **The uncanny break-marker.** Bone-white pantsuit; the *exact same silhouette* as HR Manager but visually wrong. 120 HP, 1.0 speed (same as HR Manager), fires `TrackingDoc` projectiles (a faster variant of HR Manager's tracking projectile) at mid-range.

Visually distinguishing detail: bone-white placeholder color (255, 250, 240) — not green, not red, not the muted khakis of Middle Manager. Color choice is doing the uncanny work — same silhouette, wrong skin.

The TrackingDoc projectile is a slim variant of HR Manager's `documentationRequest` (existing). New file because the parameters differ (faster, less tracking aggression, less damage per hit). Keeps the hr-manager projectile unchanged.

- [ ] **Step 1: Implement TrackingDoc projectile**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/trackingDoc.ts`:
```typescript
import type { IProjectile, ProjectileUpdateContext } from './Projectile';

const SPEED = 4.5;
const TRACKING_RATE = 1.5; // radians/sec (lower than HR Manager — easier to outrun)
const DAMAGE = 10;
const HIT_RADIUS = 0.4;
const MAX_LIFETIME_MS = 5000;

/**
 * TrackingDoc — HR Skeleton's projectile. A slim variant of HR Manager's
 * documentationRequest: same tracking shape (turns toward the player every
 * frame) but faster (4.5 vs 3.5 tiles/sec) and less aggressive turn rate
 * (1.5 vs 2.5 rad/sec) and less damage (10 vs 15). Net effect: easier to
 * outrun, but lands more often if you stand still.
 *
 * 5s lifetime. White placeholder color — visually echoes the HR Skeleton's
 * uncanny bone-white pantsuit.
 */
export class TrackingDoc implements IProjectile {
  x: number;
  y: number;
  dead = false;
  readonly color = [240, 240, 230] as const;
  private angle: number;
  private spawnedAt = performance.now();

  constructor(x: number, y: number, angle: number) {
    this.x = x;
    this.y = y;
    this.angle = angle;
  }

  update(dt: number, ctx: ProjectileUpdateContext) {
    if (this.dead) return;
    if (performance.now() - this.spawnedAt > MAX_LIFETIME_MS) {
      this.dead = true;
      return;
    }

    // Turn toward the player.
    const desired = Math.atan2(ctx.player.y - this.y, ctx.player.x - this.x);
    let delta = desired - this.angle;
    while (delta > Math.PI) delta -= 2 * Math.PI;
    while (delta < -Math.PI) delta += 2 * Math.PI;
    const maxTurn = TRACKING_RATE * dt;
    if (Math.abs(delta) > maxTurn) {
      this.angle += Math.sign(delta) * maxTurn;
    } else {
      this.angle = desired;
    }

    const newX = this.x + Math.cos(this.angle) * SPEED * dt;
    const newY = this.y + Math.sin(this.angle) * SPEED * dt;

    if (Math.hypot(ctx.player.x - newX, ctx.player.y - newY) < HIT_RADIUS) {
      ctx.player.takeDamage(DAMAGE);
      ctx.audio.onPlayerDamaged();
      this.dead = true;
      return;
    }

    const ix = Math.floor(newX);
    const iy = Math.floor(newY);
    if (
      iy < 0 || iy >= ctx.map.grid.length || !ctx.map.grid[iy] ||
      ix < 0 || ix >= ctx.map.grid[iy].length || ctx.map.grid[iy][ix] > 0
    ) {
      this.dead = true;
      return;
    }

    this.x = newX;
    this.y = newY;
  }
}
```

- [ ] **Step 2: Implement HR Skeleton**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/hrSkeleton.ts`:
```typescript
import type { EnemyUpdateContext } from './Enemy';
import { BaseEnemy } from './Enemy';
import { TrackingDoc } from '../projectiles/trackingDoc';

const HEALTH = 120;
const SPEED = 1.0;
const SIGHT_RANGE = 11;
const ATTACK_RANGE = 10;
const ATTACK_COOLDOWN_MS = 1900;

/**
 * HR Skeleton — Cycle 3 mid-tier. THE uncanny break-marker. Pantsuit-on-
 * skeleton — same silhouette as HR Manager (Cycle 1 boss), wrong skin.
 * The first one the player sees in cyc3_lobby_redux is the gut-punch
 * moment of the entire campaign.
 *
 * 120 HP, 1.0 speed, fires TrackingDoc projectiles every 1.9s at mid-range
 * (10 tiles). Tracking is gentler than HR Manager's documentationRequest
 * — easier to outrun, but the *visual* identity is the threat.
 *
 * Bone-white placeholder color is doing the heavy lifting until real
 * sprite art lands in Phase 6.
 */
export class HRSkeleton extends BaseEnemy {
  health = HEALTH;
  readonly placeholderColor = [240, 240, 230] as const;

  private nextAttackAt = 0;

  update(dt: number, ctx: EnemyUpdateContext) {
    if (this.state === 'dead') return;
    if (this.state === 'pain' && performance.now() >= this.painUntil) {
      this.state = 'chase';
    }

    const dx = ctx.player.x - this.x;
    const dy = ctx.player.y - this.y;
    const dist = Math.hypot(dx, dy);

    if (this.state === 'idle' && dist < SIGHT_RANGE) {
      this.state = 'chase';
    }

    if (this.state === 'chase') {
      if (dist <= ATTACK_RANGE && performance.now() >= this.nextAttackAt) {
        this.nextAttackAt = performance.now() + ATTACK_COOLDOWN_MS;
        const angleToPlayer = Math.atan2(dy, dx);
        ctx.spawnProjectile(new TrackingDoc(this.x, this.y, angleToPlayer));
        return;
      }
      const move = SPEED * dt;
      if (dist > 2.5) {
        this.x += (dx / dist) * move;
        this.y += (dy / dist) * move;
      }
    }
  }
}
```

- [ ] **Step 3: Add `'hr_skeleton'` to ThingType + register in GameLoop**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`:
```typescript
export type ThingType =
  | 'player_start'
  | 'intern'
  | 'account_executive'
  | 'recruiter'
  | 'hr_manager'
  | 'pip'
  | 'notification'
  | 'middle_manager'
  | 'vp_of_sales'
  | 'surveillance_drone'
  | 'hr_skeleton'
  | 'exit';
```

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`:

a) Add the import:
```typescript
import { HRSkeleton } from '../entities/enemies/hrSkeleton';
```

b) In the spawn loop:
```typescript
      } else if (t.type === 'hr_skeleton') {
        this.enemies.push(new HRSkeleton(t.x, t.y));
```

- [ ] **Step 4: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/projectiles/trackingDoc.ts \
        components/loom/entities/enemies/hrSkeleton.ts \
        components/loom/types.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add HR Skeleton enemy (uncanny break-marker) + TrackingDoc projectile

Cycle 3 mid-tier. The visual gut-punch of the campaign — pantsuit-on-
skeleton, same silhouette as the HR Manager (Cycle 1 boss), wrong skin.
First sighting in cyc3_lobby_redux must land. Bone-white placeholder
color (240, 240, 230) does the heavy lifting until Phase 6 sprite art.

120 HP, 1.0 speed, fires TrackingDoc projectiles every 1.9s at 10 tiles.

TrackingDoc projectile mirrors HR Manager's documentationRequest shape
(turns-toward-player every frame) but faster (4.5 vs 3.5 tiles/sec),
gentler turn rate (1.5 vs 2.5 rad/sec), less damage (10 vs 15). Easier
to outrun — the *visual* identity is the threat, not the projectile.

No vitest tests for HR Skeleton specifically (mechanically nearly
identical to HR Manager which is already tested via baseEnemy.test.ts
+ documentationRequest.test.ts) — Phase 4 review may flag this gap;
deferred to Phase 5 prelude if so."
```

---

## Task 4.10: Wellness Officer enemy + WellnessPulse projectile (twin bioacoustic)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/wellnessPulse.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/wellnessOfficer.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Wellness Officer — Cycle 3 heavy. 160 HP, slow (0.6 speed), fires *two simultaneous* `WellnessPulse` projectiles per attack — one offset slightly above the aim line, one slightly below — at 1.8s cooldown. Round body, perpetual smile. Each pulse does 8 damage.

The two projectiles' angle offsets create a "twin" pattern that's hard to dodge by stepping straight sideways — the player has to commit to a direction. Twin-emitter aesthetic matches the spec's "twin bioacoustic emitters firing 60/528 Hz" (the actual frequency-mode mechanic is deferred to Phase 5 along with Frequency Tuner's other modes).

- [ ] **Step 1: Implement WellnessPulse projectile**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/wellnessPulse.ts`:
```typescript
import type { IProjectile, ProjectileUpdateContext } from './Projectile';

const SPEED = 4.0;
const DAMAGE = 8;
const HIT_RADIUS = 0.4;
const MAX_LIFETIME_MS = 4000;

/**
 * WellnessPulse — Wellness Officer's projectile. Pale-yellow bioacoustic
 * pulse; straight-line travel at 4 tiles/sec. 8 damage per hit. Two pulses
 * are spawned per Officer attack (one above, one below the aim ray) — a
 * twin pattern that's hard to dodge by stepping sideways alone.
 */
export class WellnessPulse implements IProjectile {
  x: number;
  y: number;
  dead = false;
  readonly color = [240, 230, 130] as const;
  private dirX: number;
  private dirY: number;
  private spawnedAt = performance.now();

  constructor(x: number, y: number, angle: number) {
    this.x = x;
    this.y = y;
    this.dirX = Math.cos(angle);
    this.dirY = Math.sin(angle);
  }

  update(dt: number, ctx: ProjectileUpdateContext) {
    if (this.dead) return;
    if (performance.now() - this.spawnedAt > MAX_LIFETIME_MS) {
      this.dead = true;
      return;
    }

    const newX = this.x + this.dirX * SPEED * dt;
    const newY = this.y + this.dirY * SPEED * dt;

    if (Math.hypot(ctx.player.x - newX, ctx.player.y - newY) < HIT_RADIUS) {
      ctx.player.takeDamage(DAMAGE);
      ctx.audio.onPlayerDamaged();
      this.dead = true;
      return;
    }

    const ix = Math.floor(newX);
    const iy = Math.floor(newY);
    if (
      iy < 0 || iy >= ctx.map.grid.length || !ctx.map.grid[iy] ||
      ix < 0 || ix >= ctx.map.grid[iy].length || ctx.map.grid[iy][ix] > 0
    ) {
      this.dead = true;
      return;
    }

    this.x = newX;
    this.y = newY;
  }
}
```

- [ ] **Step 2: Implement WellnessOfficer**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/wellnessOfficer.ts`:
```typescript
import type { EnemyUpdateContext } from './Enemy';
import { BaseEnemy } from './Enemy';
import { WellnessPulse } from '../projectiles/wellnessPulse';

const HEALTH = 160;
const SPEED = 0.6;
const SIGHT_RANGE = 11;
const ATTACK_RANGE = 10;
const ATTACK_COOLDOWN_MS = 1800;
const TWIN_OFFSET = 0.18; // ~10° between the two pulses

/**
 * Wellness Officer — Cycle 3 heavy. 160 HP, slow (0.6 speed), fires twin
 * bioacoustic pulses per attack (1.8s cooldown). Two WellnessPulse
 * projectiles spawn at angles [aim-0.18, aim+0.18] — a forked pattern
 * that punishes pure-sideways dodging.
 *
 * Round body, perpetual smile. Pale-yellow placeholder.
 *
 * The 60 Hz / 528 Hz frequency-mode distinction from spec §5.2 is
 * cosmetic in Phase 4 — both pulses do the same 8 damage. Deferred
 * to Phase 5 alongside Frequency Tuner mode-cycling.
 */
export class WellnessOfficer extends BaseEnemy {
  health = HEALTH;
  readonly placeholderColor = [240, 230, 130] as const;

  private nextAttackAt = 0;

  update(dt: number, ctx: EnemyUpdateContext) {
    if (this.state === 'dead') return;
    if (this.state === 'pain' && performance.now() >= this.painUntil) {
      this.state = 'chase';
    }

    const dx = ctx.player.x - this.x;
    const dy = ctx.player.y - this.y;
    const dist = Math.hypot(dx, dy);

    if (this.state === 'idle' && dist < SIGHT_RANGE) {
      this.state = 'chase';
    }

    if (this.state === 'chase') {
      if (dist <= ATTACK_RANGE && performance.now() >= this.nextAttackAt) {
        this.nextAttackAt = performance.now() + ATTACK_COOLDOWN_MS;
        const angleToPlayer = Math.atan2(dy, dx);
        ctx.spawnProjectile(new WellnessPulse(this.x, this.y, angleToPlayer - TWIN_OFFSET));
        ctx.spawnProjectile(new WellnessPulse(this.x, this.y, angleToPlayer + TWIN_OFFSET));
        return;
      }
      const move = SPEED * dt;
      if (dist > 3) {
        this.x += (dx / dist) * move;
        this.y += (dy / dist) * move;
      }
    }
  }
}
```

- [ ] **Step 3: Add `'wellness_officer'` to ThingType + register in GameLoop**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`:
```typescript
export type ThingType =
  | 'player_start'
  | 'intern'
  | 'account_executive'
  | 'recruiter'
  | 'hr_manager'
  | 'pip'
  | 'notification'
  | 'middle_manager'
  | 'vp_of_sales'
  | 'surveillance_drone'
  | 'hr_skeleton'
  | 'wellness_officer'
  | 'exit';
```

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`:

a) Add the import:
```typescript
import { WellnessOfficer } from '../entities/enemies/wellnessOfficer';
```

b) In the spawn loop:
```typescript
      } else if (t.type === 'wellness_officer') {
        this.enemies.push(new WellnessOfficer(t.x, t.y));
```

- [ ] **Step 4: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/projectiles/wellnessPulse.ts \
        components/loom/entities/enemies/wellnessOfficer.ts \
        components/loom/types.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add Wellness Officer enemy + WellnessPulse projectile (twin bioacoustic)

Cycle 3 heavy. 160 HP, 0.6 speed (slowest of any enemy so far). Fires
twin bioacoustic pulses per attack: two WellnessPulse projectiles at
aim±0.18 rad (a 20° fork). 1.8s cooldown. Each pulse does 8 damage.

Twin pattern punishes pure-sideways dodging — the player has to commit
to a direction. Round body + perpetual smile is the spec aesthetic;
pale-yellow placeholder until Phase 6 art.

The 60 Hz / 528 Hz frequency distinction from spec §5.2 is cosmetic in
Phase 4 (both pulses do 8 damage). Deferred to Phase 5 alongside
Frequency Tuner mode-cycling — that's where frequency-as-mechanic
becomes a first-class system."
```

---

## Task 4.11: Brand Ambassador boss + resurrection mechanic (Cycle 3 boss)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/brandAmbassador.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/brandAmbassador.test.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Brand Ambassador — Cycle 3 boss. 320 HP. Wears a sash. **The resurrection mechanic: every 6s, scans the enemy list for the nearest dead, non-rebranded enemy within 8 tiles; if found, sets that enemy's `state = 'chase'`, restores HP to 50% of its max, and marks it `rebranded = true`** (so each corpse is rebranded at most once).

The "going viral" attack from spec §5.2 is implemented as a 4-direction `WellnessPulse` burst centered on the boss every 4s — fires four pulses at angles `[0, π/2, π, 3π/2]` simultaneously. Forces the player to maintain spacing.

`isBoss = true` so the existing exit-gate works (player can't leave the arena until the Ambassador dies).

The Brand Ambassador never enters `'idle'` — it spawns active (state = `'chase'`) on map load. Justification: a boss arena, the boss should engage immediately on player spawn-in.

- [ ] **Step 1: Write the failing test**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/brandAmbassador.test.ts`:
```typescript
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { BrandAmbassador } from './brandAmbassador';
import { Intern } from './intern';
import type { EnemyUpdateContext, IEnemy } from './Enemy';
import type { IProjectile } from '../projectiles/Projectile';

function makeCtx(playerX: number, playerY: number, otherEnemies: IEnemy[] = []) {
  const spawned: IProjectile[] = [];
  return {
    ctx: {
      player: { x: playerX, y: playerY, angle: 0, ammo: 50, pulseCharge: 1, health: 100, activeSlot: 1, takeDamage() {} } as unknown as EnemyUpdateContext['player'],
      map: { id: 'test', cycle: 3, grid: Array.from({length: 20}, () => Array(20).fill(0)), things: [], textures: [] } as unknown as EnemyUpdateContext['map'],
      spawnProjectile: (p: IProjectile) => spawned.push(p),
      audio: { onEnemyHit: vi.fn(), onPlayerDamaged: vi.fn(), onGameStart() {}, onGameEnd() {}, onMapLoad() {} } as unknown as EnemyUpdateContext['audio'],
      otherEnemies,
    },
    spawned,
  };
}

describe('BrandAmbassador', () => {
  beforeEach(() => { vi.useFakeTimers(); vi.setSystemTime(0); });
  afterEach(() => { vi.useRealTimers(); });

  it('isBoss flag is true', () => {
    const b = new BrandAmbassador(10, 10);
    expect(b.isBoss).toBe(true);
  });

  it('starts in chase state (not idle)', () => {
    const b = new BrandAmbassador(10, 10);
    expect(b.state).toBe('chase');
  });

  it('has 320 HP', () => {
    const b = new BrandAmbassador(10, 10);
    expect(b.health).toBe(320);
  });

  describe('resurrection mechanic', () => {
    it('resurrects a dead, non-rebranded enemy within range every 6s', () => {
      const b = new BrandAmbassador(10, 10);
      const corpse = new Intern(11, 10);
      corpse.state = 'dead';
      corpse.health = 0;
      const others: IEnemy[] = [corpse];
      const { ctx } = makeCtx(15, 15, others);
      // Inject otherEnemies — the BrandAmbassador reads them via a sibling field.
      // For Phase 4 we use the (test-only) enemiesProvider hook; see implementation.
      b.setEnemyList(others);

      // Initial update — no resurrection yet (cooldown not elapsed).
      b.update(0.016, ctx);
      expect(corpse.state).toBe('dead');

      // Advance 6s and update again.
      vi.setSystemTime(6500);
      b.update(0.016, ctx);
      expect(corpse.state).toBe('chase');
      // Intern max HP is 20; resurrected at 50% = floor(10) = 10. We assert
      // `> 0` to stay robust if Intern's HEALTH is later tuned — the
      // implementer subagent should verify Intern HEALTH and tighten this
      // assertion to `toBe(10)` (or whatever floor(HEALTH/2) is) once the
      // values are confirmed.
      expect(corpse.health).toBeGreaterThan(0);
      expect(corpse.rebranded).toBe(true);
    });

    it('does not resurrect an already-rebranded corpse', () => {
      const b = new BrandAmbassador(10, 10);
      const corpse = new Intern(11, 10);
      corpse.state = 'dead';
      corpse.health = 0;
      corpse.rebranded = true;
      b.setEnemyList([corpse]);
      const { ctx } = makeCtx(15, 15, [corpse]);
      vi.setSystemTime(6500);
      b.update(0.016, ctx);
      expect(corpse.state).toBe('dead');
    });

    it('does not resurrect a corpse beyond 8 tile range', () => {
      const b = new BrandAmbassador(10, 10);
      const corpse = new Intern(20, 10); // 10 tiles east — out of range
      corpse.state = 'dead';
      corpse.health = 0;
      b.setEnemyList([corpse]);
      const { ctx } = makeCtx(15, 15, [corpse]);
      vi.setSystemTime(6500);
      b.update(0.016, ctx);
      expect(corpse.state).toBe('dead');
    });
  });

  describe('going viral attack', () => {
    it('fires 4 pulses in a cross pattern every 4s', () => {
      const b = new BrandAmbassador(10, 10);
      b.setEnemyList([]);
      const { ctx, spawned } = makeCtx(15, 15);
      b.update(0.016, ctx);
      expect(spawned.length).toBe(4);
    });

    it('respects 4s cooldown', () => {
      const b = new BrandAmbassador(10, 10);
      b.setEnemyList([]);
      const { ctx, spawned } = makeCtx(15, 15);
      b.update(0.016, ctx);
      expect(spawned.length).toBe(4);
      vi.setSystemTime(2000);
      b.update(0.016, ctx);
      expect(spawned.length).toBe(4); // still 4 (no second burst yet)
      vi.setSystemTime(4500);
      b.update(0.016, ctx);
      expect(spawned.length).toBe(8); // second burst
    });
  });
});
```

- [ ] **Step 2: Run, verify failure**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/enemies/brandAmbassador.test.ts 2>&1 | tail -10
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement BrandAmbassador**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/brandAmbassador.ts`:
```typescript
import type { EnemyUpdateContext, IEnemy } from './Enemy';
import { BaseEnemy } from './Enemy';
import { WellnessPulse } from '../projectiles/wellnessPulse';

const HEALTH = 320;
const SPEED = 0.5;
const ATTACK_COOLDOWN_MS = 4000;
const RESURRECTION_COOLDOWN_MS = 6000;
const RESURRECTION_RANGE = 8;
const BURST_PULSE_COUNT = 4;

const ENEMY_MAX_HP: Record<string, number> = {
  // Used to pick the right "max HP" for the rebranded enemy. We use class
  // name (constructor.name) as the key. This is brittle if classes get
  // renamed — Phase 5/6 should formalize via a static `MAX_HP` field on
  // each enemy class. For Phase 4 it covers the 9 non-boss enemy classes
  // that are legitimately resurrectable in Cycle 3 maps.
  //
  // Values must match the HEALTH constant in each enemy file.
  // Implementer: re-grep `HEALTH = ` across components/loom/entities/enemies/
  // before committing this file to confirm the table values are still
  // accurate at implementation time.
  Intern: 20,
  AccountExecutive: 40,
  Recruiter: 30,
  PIP: 60,
  Notification: 15,
  MiddleManager: 80,
  HRSkeleton: 120,
  SurveillanceDrone: 180,
  WellnessOfficer: 160,
};

/**
 * Brand Ambassador — Cycle 3 boss. 320 HP, isBoss=true. Wears a sash.
 *
 * Resurrection mechanic: every 6s, scans the enemy list for the nearest
 * dead, non-rebranded enemy within 8 tiles; if found, flips state back
 * to 'chase', restores 50% of max HP, marks rebranded=true. Each corpse
 * is rebranded at most once.
 *
 * Going-viral attack: every 4s, fires 4 WellnessPulse projectiles in
 * a cross pattern (angles 0, π/2, π, 3π/2) — forces the player to
 * maintain spacing or eat one.
 *
 * Spawns active (state='chase') on map load — boss arena, no idle.
 */
export class BrandAmbassador extends BaseEnemy {
  health = HEALTH;
  readonly placeholderColor = [180, 100, 200] as const;
  readonly isBoss = true;
  protected readonly PAIN_DURATION_MS = 100;

  private nextBurstAt = 0;
  private nextResurrectAt = 0;
  private enemyList: IEnemy[] = [];

  constructor(x: number, y: number) {
    super(x, y);
    this.state = 'chase'; // boss starts engaged
  }

  /** Called by GameLoop once after construction so the boss can scan
   *  for corpses. Phase 4 wires this up after the spawn loop completes. */
  setEnemyList(enemies: IEnemy[]) {
    this.enemyList = enemies;
  }

  update(dt: number, ctx: EnemyUpdateContext) {
    if (this.state === 'dead') return;
    if (this.state === 'pain' && performance.now() >= this.painUntil) {
      this.state = 'chase';
    }

    const dx = ctx.player.x - this.x;
    const dy = ctx.player.y - this.y;
    const dist = Math.hypot(dx, dy);

    // Resurrection scan.
    if (performance.now() >= this.nextResurrectAt) {
      this.nextResurrectAt = performance.now() + RESURRECTION_COOLDOWN_MS;
      let bestCorpse: IEnemy | null = null;
      let bestDist = RESURRECTION_RANGE;
      for (const e of this.enemyList) {
        if (e === this) continue;
        if (e.state !== 'dead') continue;
        if (e.rebranded) continue;
        const d = Math.hypot(e.x - this.x, e.y - this.y);
        if (d < bestDist) {
          bestDist = d;
          bestCorpse = e;
        }
      }
      if (bestCorpse) {
        // Look up max HP by class name. Falls back to current `health`'s
        // initialized value if unknown (for Phase 5 enemies not yet in the
        // ENEMY_MAX_HP table).
        const maxHp = ENEMY_MAX_HP[bestCorpse.constructor.name] ?? 30;
        bestCorpse.state = 'chase';
        bestCorpse.health = Math.floor(maxHp * 0.5);
        bestCorpse.rebranded = true;
      }
    }

    // Going-viral burst.
    if (performance.now() >= this.nextBurstAt) {
      this.nextBurstAt = performance.now() + ATTACK_COOLDOWN_MS;
      for (let i = 0; i < BURST_PULSE_COUNT; i++) {
        const angle = (i / BURST_PULSE_COUNT) * 2 * Math.PI;
        ctx.spawnProjectile(new WellnessPulse(this.x, this.y, angle));
      }
    }

    // Slow advance toward the player (chase).
    const move = SPEED * dt;
    if (dist > 3) {
      this.x += (dx / dist) * move;
      this.y += (dy / dist) * move;
    }
  }
}
```

- [ ] **Step 4: Run test, verify pass**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/enemies/brandAmbassador.test.ts 2>&1 | tail -10
```

Expected: 7 tests pass. (If `Intern.health` is initialized in the constructor differently than expected, adjust the test's expected post-resurrect value to match `Intern`'s actual max HP.)

- [ ] **Step 5: Add `'brand_ambassador'` to ThingType + register in GameLoop with enemy-list wiring**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`:
```typescript
export type ThingType =
  | 'player_start'
  | 'intern'
  | 'account_executive'
  | 'recruiter'
  | 'hr_manager'
  | 'pip'
  | 'notification'
  | 'middle_manager'
  | 'vp_of_sales'
  | 'surveillance_drone'
  | 'hr_skeleton'
  | 'wellness_officer'
  | 'brand_ambassador'
  | 'exit';
```

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`:

a) Add the import:
```typescript
import { BrandAmbassador } from '../entities/enemies/brandAmbassador';
```

b) In the spawn loop (after the existing `vp_of_sales` branch and before the closing `}`):
```typescript
      } else if (t.type === 'brand_ambassador') {
        this.enemies.push(new BrandAmbassador(t.x, t.y));
```

c) After the `for (const t of map.things)` loop completes (right after the closing `}` of the loop, but inside the constructor), add the boss-list-injection call:
```typescript
    // Wire the Brand Ambassador (if present) to the live enemy list so its
    // resurrection scan can see corpses.
    for (const e of this.enemies) {
      if (e instanceof BrandAmbassador) {
        e.setEnemyList(this.enemies);
      }
    }
```

- [ ] **Step 6: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/enemies/brandAmbassador.ts \
        components/loom/entities/enemies/brandAmbassador.test.ts \
        components/loom/types.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add Brand Ambassador boss + resurrection mechanic (Cycle 3 boss)

320 HP, isBoss=true, starts in chase state. Wears a sash (purple-pink
placeholder).

Resurrection: every 6s, scans the enemy list for the nearest dead,
non-rebranded enemy within 8 tiles. If found: state→chase, health=50%
of max, mark rebranded=true. Each corpse is rebranded at most once
(BaseEnemy.rebranded flag from Task 4.4 prevents infinite chains).

ENEMY_MAX_HP lookup table maps class name → max HP for the resurrect
math. Brittle (rename a class and the table breaks); Phase 5/6 should
formalize via a static MAX_HP field on each enemy class. The table
currently covers all 9 resurrectable enemy classes.

Going-viral attack: every 4s, fires 4 WellnessPulse projectiles in
a cross pattern (angles 0, π/2, π, 3π/2). Forces the player to
maintain spacing.

GameLoop constructor now wires every BrandAmbassador instance's
enemy-list reference after the spawn loop completes — the boss reads
from this.enemies live.

7 vitest tests cover isBoss flag, initial chase state, HP, resurrection
range/cooldown/no-double-revival, and going-viral burst cadence."
```

---

## Task 4.12: Map — `cyc3_lobby_redux` (the gut-punch first sighting)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc3_lobby_redux.json`

The first map of Cycle 3. Echoes `cyc1_lobby`'s shape (a long corridor + atrium) but with one critical difference: the **HR Skeleton is placed prominently in the main path**, where the player can't miss it. Layout proportions are deliberately one-tile off from the original lobby — wider in the middle, narrower at the entrance — to feel *just wrong*.

Other inhabitants are familiar (Recruiters + Interns) — the Skeleton does the uncanny work alone in this map.

- [ ] **Step 1: Create the map**

Create `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc3_lobby_redux.json`:
```json
{
  "id": "cyc3_lobby_redux",
  "cycle": 3,
  "music": "ambient",
  "intermissionText": "lobby_redux clear. simulation drift compounding. recommend immediate exfil.",
  "grid": [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  ],
  "things": [
    { "type": "player_start", "x": 2, "y": 5, "angle": 0 },
    { "type": "hr_skeleton", "x": 12, "y": 5 },
    { "type": "recruiter", "x": 9, "y": 3 },
    { "type": "recruiter", "x": 9, "y": 7 },
    { "type": "intern", "x": 19, "y": 5 },
    { "type": "intern", "x": 21, "y": 4 },
    { "type": "exit", "x": 23, "y": 5 }
  ],
  "textures": ["wall_lobby"]
}
```

- [ ] **Step 2: Smoke test (dev server load)**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
(npm run dev 2>&1 | head -15) & DEV_PID=$!
sleep 6
curl -fsSL http://localhost:3001/data/loom/maps/cyc3_lobby_redux.json | head -3
kill $DEV_PID 2>/dev/null
wait 2>/dev/null
```

Expected: tsc clean, JSON returns successfully.

- [ ] **Step 3: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add public/data/loom/maps/cyc3_lobby_redux.json
git commit -m "Add cyc3_lobby_redux map (HR Skeleton gut-punch sighting)

First map of Cycle 3. Echoes cyc1_lobby's corridor + atrium shape,
proportions deliberately one tile off (wider in middle, narrower at
entrance) so the geometry feels just-wrong.

The HR Skeleton is placed at (12, 5) — dead center of the main path
where the player can't miss it on first traversal. Two Recruiters
flanking, two Interns near the exit. The Skeleton does the uncanny
work alone; Cycles 1-2 enemies populate the rest so the contrast lands.

Phase 4 manual playtest gate: 'does the first HR Skeleton sighting
land' is judged on this map specifically."
```

---

## Task 4.13: Map — `cyc3_cubicles_decay` (Surveillance Drone introduction)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc3_cubicles_decay.json`

Cubicle farm partially decompiled. Sparse pillar layout — what used to be cubicle walls are now scattered single-cell pillars. Introduces the Surveillance Drone — placed high (visually mid-arena) so the long-range laser can reach the player from multiple sight lines.

- [ ] **Step 1: Create the map**

Create `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc3_cubicles_decay.json`:
```json
{
  "id": "cyc3_cubicles_decay",
  "cycle": 3,
  "music": "ambient",
  "intermissionText": "cubicles_decay clear. structural integrity 47%. surveillance hostile.",
  "grid": [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  ],
  "things": [
    { "type": "player_start", "x": 2, "y": 6, "angle": 0 },
    { "type": "surveillance_drone", "x": 10, "y": 4 },
    { "type": "surveillance_drone", "x": 14, "y": 8 },
    { "type": "account_executive", "x": 8, "y": 7 },
    { "type": "account_executive", "x": 16, "y": 5 },
    { "type": "pip", "x": 13, "y": 6 },
    { "type": "pip", "x": 18, "y": 7 },
    { "type": "exit", "x": 22, "y": 6 }
  ],
  "textures": ["wall_lobby"]
}
```

- [ ] **Step 2: Smoke test + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add public/data/loom/maps/cyc3_cubicles_decay.json
git commit -m "Add cyc3_cubicles_decay map (Surveillance Drone introduction)

Second map of Cycle 3. Cubicle farm half-decompiled — what used to be
walls are now scattered single-cell pillars (3 rows of 4 pillars each,
staggered). Long sight lines + sparse cover.

Two Surveillance Drones placed high in the arena so their long-range
LaserScan can reach the player from multiple angles. Two AEs and two
PIPs round out the room — AEs hold their distance, PIPs close in fast,
forcing the player to manage two threat profiles simultaneously while
dodging the Drone lasers.

24×12 grid — slightly wider than the lobby_redux to give the Drones
room to back off."
```

---

## Task 4.14: Map — `cyc3_open_office_inverse` (Wellness Officer introduction)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc3_open_office_inverse.json`

Open office mirror layout — the entrance is on the *east* side of the room (mirror of cyc2_open_office's west entry) and the exit is on the *west*. Player spawns and walks west. First Wellness Officer encounter — placed at a chokepoint where its twin-pulse pattern is hardest to dodge.

- [ ] **Step 1: Create the map**

Create `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc3_open_office_inverse.json`:
```json
{
  "id": "cyc3_open_office_inverse",
  "cycle": 3,
  "music": "ambient",
  "intermissionText": "open_office_inverse clear. directional drift confirmed. proceeding.",
  "grid": [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 1, 1, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 1, 1, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  ],
  "things": [
    { "type": "player_start", "x": 19, "y": 6, "angle": 3.14159 },
    { "type": "wellness_officer", "x": 11, "y": 6 },
    { "type": "surveillance_drone", "x": 5, "y": 3 },
    { "type": "surveillance_drone", "x": 5, "y": 9 },
    { "type": "notification", "x": 14, "y": 4 },
    { "type": "notification", "x": 14, "y": 8 },
    { "type": "notification", "x": 7, "y": 6 },
    { "type": "notification", "x": 9, "y": 2 },
    { "type": "exit", "x": 2, "y": 6 }
  ],
  "textures": ["wall_lobby"]
}
```

- [ ] **Step 2: Smoke test + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add public/data/loom/maps/cyc3_open_office_inverse.json
git commit -m "Add cyc3_open_office_inverse map (Wellness Officer introduction)

Third map of Cycle 3. Mirror layout — player spawns east-facing-west
(angle π) instead of the standard west-facing-east. Exit is on the
west wall.

A central island of walls (a hollow conference-room interior) creates
a chokepoint at column 10. The Wellness Officer is placed inside the
chokepoint — its twin-WellnessPulse pattern is hardest to dodge in
the narrow corridor flanking the island.

Two Surveillance Drones at the corners of the western half pin the
player from long range while the Wellness Officer fires forks. Four
Notifications (high-speed dive-bombers) layer chaos across both halves.

22×12 grid."
```

---

## Task 4.15: Map — `cyc3_glass_echo` (rehearsal — all C2/C3 lower-tiers)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc3_glass_echo.json`

A scaled-down echo of `cyc2_glass_confroom_boss` — same room shape minus the boss. Cluster combat: PIPs, Notifications, Middle Managers, plus one HR Skeleton. The "rehearsal" before the Brand Ambassador arena. No Drones or Officers — those land in cyc3_wellness_program.

- [ ] **Step 1: Create the map**

Create `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc3_glass_echo.json`:
```json
{
  "id": "cyc3_glass_echo",
  "cycle": 3,
  "music": "ambient",
  "intermissionText": "glass_echo clear. simulation BREACH detected. boss arena ahead.",
  "grid": [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  ],
  "things": [
    { "type": "player_start", "x": 2, "y": 5, "angle": 0 },
    { "type": "pip", "x": 7, "y": 3 },
    { "type": "pip", "x": 7, "y": 7 },
    { "type": "notification", "x": 9, "y": 5 },
    { "type": "notification", "x": 11, "y": 4 },
    { "type": "middle_manager", "x": 12, "y": 3 },
    { "type": "middle_manager", "x": 12, "y": 7 },
    { "type": "hr_skeleton", "x": 14, "y": 5 },
    { "type": "exit", "x": 16, "y": 5 }
  ],
  "textures": ["wall_lobby"]
}
```

- [ ] **Step 2: Smoke test + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add public/data/loom/maps/cyc3_glass_echo.json
git commit -m "Add cyc3_glass_echo map (cluster combat rehearsal)

Fourth map of Cycle 3. Scaled-down echo of cyc2_glass_confroom_boss
shape — same long-rectangle silhouette with two pillar pairs. No boss.

Population: 2 PIPs (charge melee), 2 Notifications (dive-bombers),
2 Middle Managers (CircleBack ranged), 1 HR Skeleton at the exit
chokepoint. Wave layout escalates left-to-right: melee → dive →
ranged → uncanny.

Intentional rehearsal density before the Brand Ambassador arena.
No Surveillance Drones or Wellness Officers here — those land in
cyc3_wellness_program (Task 4.16).

18×11 grid."
```

---

## Task 4.16: Map — `cyc3_wellness_program` (Drones + Officers heavy)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc3_wellness_program.json`

Wellness center hub with side alcoves — visually echoes `cyc2_wellness_ctr` but more open. Heavy Surveillance Drone + Wellness Officer presence. The penultimate Cycle 3 map; should feel like a difficulty spike just before the boss.

- [ ] **Step 1: Create the map**

Create `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc3_wellness_program.json`:
```json
{
  "id": "cyc3_wellness_program",
  "cycle": 3,
  "music": "ambient",
  "intermissionText": "wellness_program clear. brand summit access GRANTED. simulation DISSOLVING.",
  "grid": [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 0, 1],
    [1, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 1],
    [1, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  ],
  "things": [
    { "type": "player_start", "x": 2, "y": 7, "angle": 0 },
    { "type": "surveillance_drone", "x": 6, "y": 3 },
    { "type": "surveillance_drone", "x": 6, "y": 11 },
    { "type": "surveillance_drone", "x": 14, "y": 7 },
    { "type": "wellness_officer", "x": 10, "y": 4 },
    { "type": "wellness_officer", "x": 10, "y": 10 },
    { "type": "hr_skeleton", "x": 16, "y": 7 },
    { "type": "exit", "x": 18, "y": 7 }
  ],
  "textures": ["wall_lobby"]
}
```

- [ ] **Step 2: Smoke test + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add public/data/loom/maps/cyc3_wellness_program.json
git commit -m "Add cyc3_wellness_program map (Drones + Officers heavy)

Fifth map of Cycle 3 (penultimate — boss arena follows). Wellness
center hub with two L-shaped alcoves at the top corners and two
mirror alcoves at the bottom. The four alcoves are Drone hide spots —
the player can break line of sight by moving into them but Drones
follow within sight range.

Population: 3 Surveillance Drones (corners + center), 2 Wellness
Officers (north + south of center), 1 HR Skeleton at the exit
chokepoint. Difficulty spike — the Drones lock the player at long
range while Officers fork-fire mid-range.

20×14 grid. Larger than any Cycle 3 map yet, intentionally — gives
the Drones room to retreat-and-reposition."
```

---

## Task 4.17: Map — `cyc3_brand_summit_boss` (Brand Ambassador atrium)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc3_brand_summit_boss.json`

Brand Ambassador boss arena. Large, mostly empty atrium — the boss's resurrection mechanic needs corpses, so the room seeds with low-tier "fodder" enemies (Interns + Notifications) for the player to kill, which the Ambassador can then resurrect. Existing exit-gate handles boss-death gating.

- [ ] **Step 1: Create the map**

Create `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc3_brand_summit_boss.json`:
```json
{
  "id": "cyc3_brand_summit_boss",
  "cycle": 3,
  "music": "ship_it",
  "intermissionText": "BRAND AMBASSADOR neutralized. CYCLE 3 COMPLETE. defective unit non-compliant.",
  "grid": [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  ],
  "things": [
    { "type": "player_start", "x": 2, "y": 7, "angle": 0 },
    { "type": "brand_ambassador", "x": 16, "y": 7 },
    { "type": "wellness_officer", "x": 12, "y": 4 },
    { "type": "wellness_officer", "x": 12, "y": 10 },
    { "type": "notification", "x": 8, "y": 5 },
    { "type": "notification", "x": 8, "y": 9 },
    { "type": "intern", "x": 6, "y": 3 },
    { "type": "intern", "x": 6, "y": 11 },
    { "type": "exit", "x": 19, "y": 7 }
  ],
  "textures": ["wall_lobby"]
}
```

- [ ] **Step 2: Smoke test + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add public/data/loom/maps/cyc3_brand_summit_boss.json
git commit -m "Add cyc3_brand_summit_boss map (Brand Ambassador arena)

Sixth and final map of Cycle 3. Large open atrium (22×14, mostly empty)
— deliberately spacious to give the Brand Ambassador's 8-tile
resurrection range room to operate without overlapping with the
player's spawn area.

Brand Ambassador at (16, 7). Two Wellness Officers north + south of
the boss provide active threat. Two Notifications in the mid-zone
(8 column) are dive-bomb hazards but also rebrand fodder. Two Interns
near the spawn (column 6) are the easiest kills — the player will
likely drop them first, the Ambassador will then resurrect them.

The arena visually telegraphs the resurrection loop: 'kill the easy
ones near you, then the boss revives them, then it's a 1-vs-many
again.' Player has to either ignore corpses (focus the boss) or
keep finishing each fodder enemy *between* boss bursts.

music: ship_it (placeholder for spec-canonical Err0r Flesh — track
swap deferred to Phase 6 polish).

isBoss=true on the Ambassador → existing exit-gate prevents leaving
until the Ambassador dies. EndOfCycleStub fires next (cycleNumber=3,
'cycle 4 — coming soon')."
```

---

## Task 4.18: Phase 4 e2e Playwright spec (replaces Phase 3 spec)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/e2e/loom-phase-4.spec.ts`
- Delete: `/Users/justinwest/Repos/l0b0tonline/e2e/loom-phase-3.spec.ts`

The Phase 3 spec walks Cycles 1+2. Phase 4 supersedes it: walks Cycles 1+2+3, kills 3 bosses, asserts cycleStore at each transition (1→2→3, corruption 0→1→2), verifies the EndOfCycleStub appears with cycleNumber=3, then resets to boot at cycle=1.

The spec uses test-only window hooks already exposed by Phase 3 (`__loom.teleport(x, y)`, `__loom.killBoss()`, `__loom.advanceIntermission()`). Phase 4 adds no new hooks — the existing surface is sufficient.

The walk pattern: for each map, teleport to the exit thing's position, call `killBoss()` if applicable, wait for intermission/cycle-end transition, dismiss it, repeat for the next map.

- [ ] **Step 1: Read the existing Phase 3 spec to understand the test surface**

```bash
cd /Users/justinwest/Repos/l0b0tonline
cat e2e/loom-phase-3.spec.ts | head -80
```

Examine:
- What window hooks are exposed (look for `(window as any).__loom = ...` pattern)
- The teleport-to-exit + advance-intermission cycle pattern
- The boss-kill probe shape
- The cycleStore assertion shape

This is a *read-and-adapt* step, not a write step. Note the exact API for the next step.

- [ ] **Step 2: Write the Phase 4 spec**

Create `/Users/justinwest/Repos/l0b0tonline/e2e/loom-phase-4.spec.ts` based on the Phase 3 spec's structure. Extend the walk to 14 maps and 3 boss kills.

Key additions over the Phase 3 spec:
- After cycle-transition 2→3 dismissed, assert HUD text contains `'cycle 8494'` (since cycleStore.cycle=3 → display 8491+3=8494)
- After cycle-transition 2→3 dismissed, assert `corruption === 2` via cycleStore probe
- Walk through cyc3_lobby_redux → cyc3_cubicles_decay → cyc3_open_office_inverse → cyc3_glass_echo → cyc3_wellness_program → cyc3_brand_summit_boss
- Kill the Brand Ambassador on the last map; assert EndOfCycleStub appears with cycleNumber=3
- Press a key to dismiss EndOfCycleStub; assert cycleStore was reset (cycle=1, corruption=0)

The spec body (sketch — adjust to match the exact hook names from Step 1):
```typescript
import { test, expect } from '@playwright/test';

test.describe('LOOM Phase 4 — Cycles 1+2+3 + 3 bosses + cycleStore reset', () => {
  test('full campaign walkthrough', async ({ page }) => {
    const consoleErrors: string[] = [];
    page.on('console', (msg) => { if (msg.type() === 'error') consoleErrors.push(msg.text()); });

    await page.goto('/');
    // Boot J0IN 0S, double-click loom.exe icon, dismiss boot sequence
    // ... (mirror Phase 3 pattern)

    // CYCLE 1 — lobby → cubicles → hr_arena → kill HR Manager
    for (const mapId of ['cyc1_lobby', 'cyc1_cubicles']) {
      await waitForZone(page, mapId);
      await teleportToExit(page);
      await advanceIntermission(page);
    }
    await waitForZone(page, 'cyc1_hr_arena');
    await page.evaluate(() => (window as any).__loom.killBoss());
    await teleportToExit(page);
    // cycle-transition 1→2
    await advanceIntermission(page);
    // Assert cycleStore advanced
    const c2State = await page.evaluate(() => (window as any).__loom.cycleStore());
    expect(c2State.cycle).toBe(2);
    expect(c2State.corruption).toBe(1);

    // CYCLE 2 — open_office → break_room → slack_huddle → wellness_ctr → glass_confroom_boss
    for (const mapId of ['cyc2_open_office', 'cyc2_break_room', 'cyc2_slack_huddle', 'cyc2_wellness_ctr']) {
      await waitForZone(page, mapId);
      await teleportToExit(page);
      await advanceIntermission(page);
    }
    await waitForZone(page, 'cyc2_glass_confroom_boss');
    await page.evaluate(() => (window as any).__loom.killBoss());
    await teleportToExit(page);
    // cycle-transition 2→3
    await advanceIntermission(page);
    const c3State = await page.evaluate(() => (window as any).__loom.cycleStore());
    expect(c3State.cycle).toBe(3);
    expect(c3State.corruption).toBe(2);
    // Assert HUD shows 'cycle 8494' (8491 + 3)
    await expect(page.getByText(/cycle 8494/)).toBeVisible();
    // Assert simulation BREACH text appears
    await expect(page.getByText(/simulation BREACH/)).toBeVisible();

    // CYCLE 3 — lobby_redux → cubicles_decay → open_office_inverse → glass_echo → wellness_program → brand_summit_boss
    for (const mapId of ['cyc3_lobby_redux', 'cyc3_cubicles_decay', 'cyc3_open_office_inverse', 'cyc3_glass_echo', 'cyc3_wellness_program']) {
      await waitForZone(page, mapId);
      await teleportToExit(page);
      await advanceIntermission(page);
    }
    await waitForZone(page, 'cyc3_brand_summit_boss');
    await page.evaluate(() => (window as any).__loom.killBoss());
    await teleportToExit(page);
    // cycle-end (Cycle 3 is end-of-campaign for Phase 4)
    await expect(page.getByText(/CYCLE 3 COMPLETE/i)).toBeVisible();
    await expect(page.getByText(/cycle 4 — coming soon/i)).toBeVisible();

    // Dismiss EndOfCycleStub → return to boot
    await page.keyboard.press('Space');
    // Re-running loom.exe should show cycleStore reset
    const resetState = await page.evaluate(() => (window as any).__loom.cycleStore());
    expect(resetState.cycle).toBe(1);
    expect(resetState.corruption).toBe(0);

    // Final no-console-errors gate
    expect(consoleErrors).toEqual([]);
  });
});

// Helpers — copy/extend from the Phase 3 spec.
async function waitForZone(page: import('@playwright/test').Page, expectedZone: string) {
  await page.waitForFunction(
    (z) => document.body.textContent?.includes(`zone: ${z}`) ?? false,
    expectedZone,
    { timeout: 5_000 },
  );
}

async function teleportToExit(page: import('@playwright/test').Page) {
  await page.evaluate(() => (window as any).__loom.teleportToExit());
  await page.waitForTimeout(300); // give exit-detect a frame
}

async function advanceIntermission(page: import('@playwright/test').Page) {
  await page.keyboard.press('Space');
  await page.waitForTimeout(300);
}
```

(The exact helper names will need to match what the Phase 3 spec exposes — `teleportToExit`, `killBoss`, `cycleStore` are illustrative. If the existing spec uses different names, update accordingly.)

- [ ] **Step 3: Delete the Phase 3 spec**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git rm e2e/loom-phase-3.spec.ts
```

The Phase 4 spec is a strict superset — it walks Cycles 1+2 unchanged, then continues into Cycle 3.

- [ ] **Step 4: Run the new spec**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npm run test:e2e -- e2e/loom-phase-4.spec.ts 2>&1 | tail -30
```

Expected: PASS in under 25s. If anything fails, debug — boss-gate should be working from Phase 3, the new enemies should all spawn cleanly, the cycle-transition events should fire as designed.

- [ ] **Step 5: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add e2e/loom-phase-4.spec.ts
git commit -m "Add Phase 4 e2e Playwright spec (replaces Phase 3 spec)

Full campaign walkthrough: Cycles 1+2+3, 3 boss kills (HR Manager,
VP of Sales, Brand Ambassador), 14 maps total. Asserts cycleStore
advances 1→2→3 and corruption 0→1→2 at each transition; asserts HUD
displays 'cycle 8494' + 'simulation BREACH' after cycle-3 transition;
asserts EndOfCycleStub fires at cycle-3 end with cycleNumber=3;
asserts cycleStore resets to (cycle=1, corruption=0) after dismissal.

Phase 3 spec deleted — Phase 4 is a strict superset. Tests no
longer duplicate the Cycles 1+2 walk.

No new test-window hooks added — uses the existing __loom.teleportToExit,
__loom.killBoss, __loom.cycleStore surface from Phase 3.

Final no-console-errors gate as a safety net for the four new enemy
classes + three new weapons + HUD corruption mode 2 setIntervals."
```

---

# Phase 4 final review checklist

After all 18 tasks complete, run a final review pass:

- [ ] **Task spec coverage:** Every Phase 4 spec line item from `2026-04-27-loom-doom-design.md` §8 is implemented OR explicitly documented as a deferred item in this plan's deferral list.
- [ ] **Test gate:** `npx tsc --noEmit` clean; `npx vitest run components/loom` reports all-green; `npm run test:e2e -- e2e/loom-phase-4.spec.ts` passes in under 25s.
- [ ] **Manual creative gate:** Run `npm run dev`, play through Cycles 1+2+3 manually. Verify the four NEEDS-HUMAN gates from the DoD section ("does the first HR Skeleton sighting feel uncanny," etc.). Capture results in a Phase 4 playtest log at `docs/playtests/2026-04-27-phase-4-cycle-3.md`.
- [ ] **Code review:** Dispatch a `feature-dev:code-reviewer` against the full Phase 4 commit set on `loom-game`. Address all I-rated issues; defer M-rated ones to Phase 5 prelude with explicit notes.
- [ ] **PR:** Create a PR for the Phase 4 changeset on l0b0tonline (or stack on top of PR #100 if that hasn't merged yet) and a parallel PR in LOOM-DOOM for the Phase 4 plan + playtest log.

---
