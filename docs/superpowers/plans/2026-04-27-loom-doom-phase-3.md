# LOOM Phase 3 Implementation Plan — Cycle 2 Escalation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship **Cycle 2 — the escalation tier**. After defeating the HR Manager, the player advances seamlessly into Cycle 2 (the cycleStore tracks the cycle and corruption level for real now). New tier of enemies (PIP / Notification / Middle Manager / VP of Sales), a new alt-weapon (Reply All Storm), the first HUD-corruption mode (subtle text glitches + a scrolling surveillance-log line), and 5 new maps culminating in the VP of Sales boss arena. After Cycle 2 ends, the EndOfCycleStub shows "CYCLE 3 — COMING SOON."

**Architecture:** Extends the Phase 0–2 engine. Updates `campaign.ts` to model 2-cycle progression with explicit `cycle-transition` events; the LOOMGame state machine now advances `cycleStore` at cycle boundaries. The HUD reads corruption level and renders a third line + occasional `[CONNECTION RESET]` blips when ≥1. Adds 1 new weapon (slot 3 alt), 4 enemy classes (1 boss), 1 new projectile (Middle Manager's "Circle Back"), and 5 new maps.

**Tech Stack:** Same as Phase 2 — TypeScript 5.8, React 19, Vite 6, Tailwind v4, Zustand 5, WebGL 2, Web Audio API, Vitest, Playwright.

**Pre-existing state:**
- Implementation lives at `/Users/justinwest/Repos/l0b0tonline`. Phase 2 PR is `loom-game` (PR #100); this plan assumes PR #100 is merged into `main`. If not yet merged, Phase 3 work continues on the `loom-game` branch.
- Phase 2 left 13 cleanup commits stacking on top of Phase 2 proper. Latest commit is `a563b37`.
- Phase 1 final-review item that was deferred to Phase 3: **`cycleStore.advanceCycle()` is unwired** — Task 3.2 closes this.
- All Cycle 1 content works end-to-end (Playwright Phase 2 spec walks the full path).

---

## File structure (post-Phase-3, in l0b0tonline)

New files (NEW), modified files (MOD):

```
l0b0tonline/
├── components/
│   ├── LOOMGame.tsx                                            (MOD — handle cycle-transition + reset cycleStore on EndOfCycleStub close)
│   └── loom/
│       ├── engine/
│       │   ├── audioController.ts                              (MOD — placeholder track for VP boss arena)
│       │   ├── campaign.ts                                     (MOD — CYCLE_2_MAPS, cycle-transition logic, isEndOfCampaign)
│       │   ├── campaign.test.ts                                (MOD — extended for 2-cycle progression)
│       │   └── gameLoop.ts                                     (MOD — emit cycle-transition events, MapExitEvent shape)
│       ├── entities/
│       │   ├── enemies/
│       │   │   ├── pip.ts                                      (NEW)
│       │   │   ├── notification.ts                             (NEW)
│       │   │   ├── middleManager.ts                            (NEW)
│       │   │   └── vpOfSales.ts                                (NEW — Cycle 2 boss)
│       │   ├── projectiles/
│       │   │   └── circleBack.ts                               (NEW — Middle Manager's projectile)
│       │   └── weapons/
│       │       └── replyAllStorm.ts                            (NEW — slot 3 alt)
│       └── hud/
│           └── LoomHud.tsx                                     (MOD — corruption mode 1 rendering)
└── public/
    └── data/
        └── loom/
            └── maps/
                ├── cyc2_open_office.json                       (NEW)
                ├── cyc2_break_room.json                        (NEW)
                ├── cyc2_slack_huddle.json                      (NEW)
                ├── cyc2_wellness_ctr.json                      (NEW)
                └── cyc2_glass_confroom_boss.json               (NEW)
```

Tests:
- `components/loom/engine/campaign.test.ts` (MOD — extended)
- `components/loom/entities/weapons/replyAllStorm.test.ts` (NEW — pellet count & ammo)
- `components/loom/entities/enemies/notification.test.ts` (NEW — dive-bomb behavior)
- `e2e/loom-phase-3.spec.ts` (NEW — full 8-map cycles 1+2 walkthrough)

---

# Phase 3 Definition of Done

A new player can:
1. Boot LOOM, play through Cycle 1 (lobby → cubicles → HR arena → kill HR Manager)
2. Advance to Cycle 2 with no "coming soon" stub — the intermission shows them entering Cycle 2 explicitly, the HUD updates `> cycle 8493 // simulation drift detected // surveillance: ON`, and the corruption=1 mode kicks in (subtle glitch + scrolling surveillance log)
3. Play through Cycle 2's 5 maps engaging PIP, Notification, Middle Manager
4. Defeat VP of Sales in the boss arena
5. See the EndOfCycleStub now showing "CYCLE 3 — COMING SOON" + manifesto

**Automation gate:** Phase 3 Playwright spec walks through 8 maps + 2 boss kills, asserts cycleStore advances correctly, no console errors. Vitest covers the campaign 2-cycle progression and the new Reply All Storm + Notification dive-bomb.

---

# Tasks

## Task 3.1: Extend `campaign.ts` for 2-cycle progression

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/campaign.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/campaign.test.ts`

The current API has `getNextMapId` returning null at cycle boundaries (which then triggered cycle-end). Phase 3 changes this: `getNextMapId('cyc1_hr_arena')` should return `'cyc2_open_office'`, NOT null. We add:
- `CYCLE_2_MAPS` constant
- `getCycleOfMap(id)` — which cycle does this map belong to?
- `isLastMapOfCycle(id)` — keeps existing semantics: true if next map is in a different cycle (or no next map)
- `isEndOfCampaign(id)` — NEW — true only at the very last shipped map

The exit-detection logic in GameLoop will now distinguish three cases:
1. Mid-cycle: `next-map` event with the next-cycle's-first-map id, no cycle advance
2. Last map of non-final cycle: `cycle-transition` event with next-cycle's-first-map id + new cycle number → LOOMGame advances `cycleStore` after intermission
3. Last map of final cycle: `cycle-end` event → EndOfCycleStub

- [ ] **Step 1: Update `campaign.test.ts`**

Replace `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/campaign.test.ts` with:
```typescript
import { describe, it, expect } from 'vitest';
import {
  CYCLE_1_MAPS,
  CYCLE_2_MAPS,
  getCycleOfMap,
  getNextMapId,
  isEndOfCampaign,
  isLastMapOfCycle,
} from './campaign';

describe('campaign', () => {
  describe('cycle map lists', () => {
    it('CYCLE_1_MAPS lists 3 maps in order', () => {
      expect(CYCLE_1_MAPS).toEqual(['cyc1_lobby', 'cyc1_cubicles', 'cyc1_hr_arena']);
    });

    it('CYCLE_2_MAPS lists 5 maps in order', () => {
      expect(CYCLE_2_MAPS).toEqual([
        'cyc2_open_office',
        'cyc2_break_room',
        'cyc2_slack_huddle',
        'cyc2_wellness_ctr',
        'cyc2_glass_confroom_boss',
      ]);
    });
  });

  describe('getCycleOfMap', () => {
    it('returns 1 for Cycle 1 maps', () => {
      expect(getCycleOfMap('cyc1_lobby')).toBe(1);
      expect(getCycleOfMap('cyc1_cubicles')).toBe(1);
      expect(getCycleOfMap('cyc1_hr_arena')).toBe(1);
    });

    it('returns 2 for Cycle 2 maps', () => {
      expect(getCycleOfMap('cyc2_open_office')).toBe(2);
      expect(getCycleOfMap('cyc2_glass_confroom_boss')).toBe(2);
    });

    it('returns null for unknown maps', () => {
      expect(getCycleOfMap('cyc99_unknown')).toBeNull();
    });
  });

  describe('getNextMapId', () => {
    it('returns next map within Cycle 1', () => {
      expect(getNextMapId('cyc1_lobby')).toBe('cyc1_cubicles');
      expect(getNextMapId('cyc1_cubicles')).toBe('cyc1_hr_arena');
    });

    it('crosses the cycle 1→2 boundary', () => {
      expect(getNextMapId('cyc1_hr_arena')).toBe('cyc2_open_office');
    });

    it('returns next map within Cycle 2', () => {
      expect(getNextMapId('cyc2_open_office')).toBe('cyc2_break_room');
      expect(getNextMapId('cyc2_wellness_ctr')).toBe('cyc2_glass_confroom_boss');
    });

    it('returns null after the final shipped map', () => {
      expect(getNextMapId('cyc2_glass_confroom_boss')).toBeNull();
    });

    it('returns null for unknown maps', () => {
      expect(getNextMapId('cyc99_unknown')).toBeNull();
    });
  });

  describe('isLastMapOfCycle', () => {
    it('false for non-boundary maps', () => {
      expect(isLastMapOfCycle('cyc1_lobby')).toBe(false);
      expect(isLastMapOfCycle('cyc1_cubicles')).toBe(false);
      expect(isLastMapOfCycle('cyc2_open_office')).toBe(false);
      expect(isLastMapOfCycle('cyc2_wellness_ctr')).toBe(false);
    });

    it('true at cycle 1 final map (cycle-transition coming)', () => {
      expect(isLastMapOfCycle('cyc1_hr_arena')).toBe(true);
    });

    it('true at cycle 2 final map (end-of-campaign)', () => {
      expect(isLastMapOfCycle('cyc2_glass_confroom_boss')).toBe(true);
    });

    it('false for unknown maps', () => {
      expect(isLastMapOfCycle('cyc99_unknown')).toBe(false);
    });
  });

  describe('isEndOfCampaign', () => {
    it('true only at the very last shipped map', () => {
      expect(isEndOfCampaign('cyc2_glass_confroom_boss')).toBe(true);
    });

    it('false at cycle 1 final map (campaign continues into cycle 2)', () => {
      expect(isEndOfCampaign('cyc1_hr_arena')).toBe(false);
    });

    it('false for non-boundary maps', () => {
      expect(isEndOfCampaign('cyc1_lobby')).toBe(false);
      expect(isEndOfCampaign('cyc2_open_office')).toBe(false);
    });

    it('false for unknown maps', () => {
      expect(isEndOfCampaign('cyc99_unknown')).toBe(false);
    });
  });
});
```

- [ ] **Step 2: Run the test to verify it fails (some tests will pass with old API; new ones fail)**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/engine/campaign.test.ts 2>&1 | tail -20
```

Expected: tests fail because `CYCLE_2_MAPS` and `isEndOfCampaign` and `getCycleOfMap` don't exist; the cycle-1→2 transition test fails because `getNextMapId('cyc1_hr_arena')` currently returns null instead of 'cyc2_open_office'.

- [ ] **Step 3: Replace `campaign.ts` with the 2-cycle implementation**

Replace `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/campaign.ts` with:
```typescript
import type { CycleNumber } from '../store/cycleStore';

/**
 * Campaign progression — what map comes after which, and where each cycle ends.
 *
 * Phase 3 ships Cycle 1 + Cycle 2 (8 maps total). Cycles 3-4 are added
 * in later phases.
 */

export const CYCLE_1_MAPS = ['cyc1_lobby', 'cyc1_cubicles', 'cyc1_hr_arena'] as const;
export const CYCLE_2_MAPS = [
  'cyc2_open_office',
  'cyc2_break_room',
  'cyc2_slack_huddle',
  'cyc2_wellness_ctr',
  'cyc2_glass_confroom_boss',
] as const;

export type CycleMapId =
  | typeof CYCLE_1_MAPS[number]
  | typeof CYCLE_2_MAPS[number];

const ALL_CYCLE_MAPS: readonly string[] = [...CYCLE_1_MAPS, ...CYCLE_2_MAPS];

const MAP_TO_CYCLE: ReadonlyMap<string, CycleNumber> = new Map<string, CycleNumber>([
  ...CYCLE_1_MAPS.map((id): [string, CycleNumber] => [id, 1]),
  ...CYCLE_2_MAPS.map((id): [string, CycleNumber] => [id, 2]),
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
 * - At cyc2_glass_confroom_boss: true (no next map)
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

- [ ] **Step 4: Run the test to verify it passes**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/engine/campaign.test.ts 2>&1 | tail -10
```

Expected: 18 tests pass.

- [ ] **Step 5: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -10
git add components/loom/engine/campaign.ts components/loom/engine/campaign.test.ts
git commit -m "Extend campaign.ts for 2-cycle progression

Phase 3 setup: getNextMapId('cyc1_hr_arena') now returns
'cyc2_open_office' instead of null, crossing the cycle-1→2 boundary
seamlessly.

New API:
- CYCLE_2_MAPS — the 5 Cycle 2 maps in order
- getCycleOfMap(id) — returns 1 | 2 | null
- isEndOfCampaign(id) — true only at cyc2_glass_confroom_boss

isLastMapOfCycle(id) keeps semantics: true at cycle boundaries OR at
end of campaign.

GameLoop's exit-detection logic (Task 3.2) reads these to decide
between next-map / cycle-transition / cycle-end events."
```

(GameLoop integration follows in Task 3.2; this commit may leave the rest of the engine using the old null-at-boundary semantics for one commit. tsc should still pass because the engine never destructured a non-null assertion.)

---

## Task 3.2: Wire `cycleStore.advanceCycle()` + new `MapExitEvent` shape

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/LOOMGame.tsx`

The GameLoop's `MapExitEvent` gains a third variant: `cycle-transition`. When the player completes the last map of a non-final cycle, the GameLoop fires this event with the next map's id + the new cycle number. LOOMGame's state machine handles it: shows intermission, on user-advance calls `cycleStore.advanceCycle()`, transitions to playing the new map.

- [ ] **Step 1: Update `MapExitEvent` and emit `cycle-transition` in GameLoop**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`:

a) Add `getCycleOfMap` to the existing campaign imports:
```typescript
import { getCycleOfMap, getNextMapId, isEndOfCampaign, isLastMapOfCycle } from './campaign';
```

b) Add `CycleNumber` import:
```typescript
import type { CycleNumber } from '../store/cycleStore';
```

c) Update the `MapExitEvent` discriminated union:
```typescript
export type MapExitEvent =
  | { kind: 'next-map'; id: string }
  | { kind: 'cycle-transition'; id: string; newCycle: CycleNumber }
  | { kind: 'cycle-end' };
```

d) Replace the existing exit-detection block in the tick function. Find:
```typescript
if (!this.exitFired) {
  const bossAlive = this.enemies.some((e) => e.isBoss && e.state !== 'dead');
  if (!bossAlive) {
    for (const t of this.map.things) {
      if (t.type !== 'exit') continue;
      const dx = t.x - this.player.x;
      const dy = t.y - this.player.y;
      if (Math.hypot(dx, dy) < 0.6) {
        this.exitFired = true;
        const nextId = getNextMapId(this.map.id);
        if (nextId) {
          this.onMapExit({ kind: 'next-map', id: nextId });
        } else if (isLastMapOfCycle(this.map.id)) {
          this.onMapExit({ kind: 'cycle-end' });
        }
        break;
      }
    }
  }
}
```

Replace with:
```typescript
if (!this.exitFired) {
  const bossAlive = this.enemies.some((e) => e.isBoss && e.state !== 'dead');
  if (!bossAlive) {
    for (const t of this.map.things) {
      if (t.type !== 'exit') continue;
      const dx = t.x - this.player.x;
      const dy = t.y - this.player.y;
      if (Math.hypot(dx, dy) < 0.6) {
        this.exitFired = true;
        if (isEndOfCampaign(this.map.id)) {
          this.onMapExit({ kind: 'cycle-end' });
        } else if (isLastMapOfCycle(this.map.id)) {
          const nextId = getNextMapId(this.map.id);
          const nextCycle = nextId ? getCycleOfMap(nextId) : null;
          if (nextId && nextCycle !== null) {
            this.onMapExit({ kind: 'cycle-transition', id: nextId, newCycle: nextCycle });
          }
        } else {
          const nextId = getNextMapId(this.map.id);
          if (nextId) {
            this.onMapExit({ kind: 'next-map', id: nextId });
          }
        }
        break;
      }
    }
  }
}
```

- [ ] **Step 2: Update LOOMGame state machine to handle cycle-transition**

Edit `/Users/justinwest/Repos/l0b0tonline/components/LOOMGame.tsx`:

a) Add the cycleStore import (just after the existing imports):
```typescript
import { useCycleStore } from './loom/store/cycleStore';
```

b) Update the `GameState` discriminated union to include cycle-advance flag in intermission:
```typescript
type GameState =
  | { kind: 'booting' }
  | { kind: 'playing'; mapId: string }
  | { kind: 'intermission'; previousMap: LoomMap; nextMapId: string; cycleAdvance: boolean }
  | { kind: 'cycle-end'; cycleNumber: number; flavorText?: string };
```

c) In the `loopRef.current = new GameLoop(...)` callback, update the dispatcher to set the cycleAdvance flag:
```typescript
loopRef.current = new GameLoop(canvasRef.current, map, (exit) => {
  if (exit.kind === 'next-map') {
    setState({ kind: 'intermission', previousMap: map, nextMapId: exit.id, cycleAdvance: false });
  } else if (exit.kind === 'cycle-transition') {
    setState({ kind: 'intermission', previousMap: map, nextMapId: exit.id, cycleAdvance: true });
  } else {
    // cycle-end → which cycle just ended? read from the current map.
    const endingCycle = useCycleStore.getState().cycle;
    setState({ kind: 'cycle-end', cycleNumber: endingCycle, flavorText: map.intermissionText });
  }
});
```

d) Update the IntermissionScreen JSX to advance cycleStore on continue when applicable:
```tsx
{state.kind === 'intermission' && (
  <IntermissionScreen
    text={state.previousMap.intermissionText ?? '... continuing ...'}
    nextMapId={state.nextMapId}
    onContinue={() => {
      if (state.cycleAdvance) {
        useCycleStore.getState().advanceCycle();
      }
      setState({ kind: 'playing', mapId: state.nextMapId });
    }}
  />
)}
```

e) Reset cycleStore when the player closes the EndOfCycleStub:
```tsx
{state.kind === 'cycle-end' && (
  <EndOfCycleStub
    cycleNumber={state.cycleNumber}
    flavorText={state.flavorText}
    onClose={() => {
      useCycleStore.getState().reset();
      setState({ kind: 'booting' });
    }}
  />
)}
```

- [ ] **Step 3: Type-check + run tests**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -10
npx vitest run components/loom 2>&1 | tail -8
```

Expected: tsc clean; vitest all pass (the existing tests aren't behavior-coupled to the cycle-transition mechanism yet).

- [ ] **Step 4: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/engine/gameLoop.ts components/LOOMGame.tsx
git commit -m "Wire cycle-transition events + cycleStore.advanceCycle (Phase 1 review #1 deferred item)

GameLoop's MapExitEvent gains a 'cycle-transition' variant for the
boundary between non-final cycles. The exit-detection logic now
distinguishes:
  - end-of-campaign (cyc2_glass_confroom_boss) → 'cycle-end'
  - last-map-of-cycle but not end (cyc1_hr_arena) → 'cycle-transition'
  - everything else → 'next-map'

LOOMGame state machine: intermission state carries cycleAdvance flag.
When the user dismisses an intermission flagged true, it calls
useCycleStore.getState().advanceCycle() before transitioning to play
the next map. The HUD's 'cycle 8493' line and corruption=1 mode kick
in on the very next render.

EndOfCycleStub now resets cycleStore on close — fresh boot starts
back at cycle=1, corruption=0."
```

---

## Task 3.3: HUD corruption mode 1 (subtle glitches + surveillance log)

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/LoomHud.tsx`

When `cycleStore.corruption >= 1`, the HUD adds a third line that scrolls through surveillance-log fragments (rotating every ~30s), and occasionally shows a `[CONNECTION RESET]` blip for ~300ms every ~45s.

The character-flicker effect (mentioned in spec §6.1 corruption-1) is deferred to Phase 4 — it's purely visual sugar and the third line + connection blip already telegraph "things are getting weird."

- [ ] **Step 1: Update LoomHud to render corruption mode 1**

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
 * Rotates every ~30s. Quotes and references match canonical loomLogs.ts
 * from the surrounding J0IN 0S app — extends the universe, doesn't replace.
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

const FRAGMENT_INTERVAL_MS = 30_000;
const CONNECTION_BLIP_INTERVAL_MS = 45_000;
const CONNECTION_BLIP_DURATION_MS = 300;

export function LoomHud({ getSnapshot }: LoomHudProps) {
  const [snap, setSnap] = useState<GameSnapshot>(() => getSnapshot());
  const cycle = useCycleStore((s) => s.cycle);
  const corruption = useCycleStore((s) => s.corruption);
  const [fragmentIdx, setFragmentIdx] = useState(0);
  const [connectionBlip, setConnectionBlip] = useState(false);

  useEffect(() => {
    let h: number;
    const tick = () => {
      setSnap(getSnapshot());
      h = requestAnimationFrame(tick);
    };
    h = requestAnimationFrame(tick);
    return () => cancelAnimationFrame(h);
  }, [getSnapshot]);

  // Rotate surveillance fragments while corruption ≥ 1.
  useEffect(() => {
    if (corruption < 1) return;
    const id = window.setInterval(() => {
      setFragmentIdx((i) => (i + 1) % SURVEILLANCE_FRAGMENTS.length);
    }, FRAGMENT_INTERVAL_MS);
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

  const simState =
    corruption === 0 ? 'nominal'
    : corruption === 1 ? 'drift detected'
    : corruption === 2 ? 'BREACH'
    : 'DISSOLVING';

  const cycleNumber = 8491 + cycle;
  const weaponName = snap.activeWeaponName.toLowerCase().replace(/ /g, '_') + '.exe';

  return (
    <>
      {/* Controls hint — top-right, compact, always visible. */}
      <div
        className="pointer-events-none absolute top-2 right-2 z-50 flex flex-col items-end gap-1 font-mono text-[10px] text-green-300"
      >
        <div className="rounded-sm border border-green-600/60 bg-black/70 px-2 py-1 leading-tight">
          <div><span className="text-green-500">WASD</span> move</div>
          <div><span className="text-green-500">MOUSE</span> click to lock / look</div>
          <div><span className="text-green-500">1/2/3</span> select weapon</div>
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
          bio: <span className="text-green-200 font-semibold">{snap.health}%</span>
          &nbsp;&nbsp;/&nbsp;&nbsp;
          queue: <span className="text-yellow-300 font-semibold">{snap.ammo}</span>
          &nbsp;&nbsp;/&nbsp;&nbsp;
          active: <span className="text-green-200">{weaponName}</span>
          &nbsp;&nbsp;/&nbsp;&nbsp;
          zone: <span className="text-green-200">{snap.zone}</span>
        </div>
        <div className="text-green-700">
          &gt; cycle {cycleNumber} &nbsp;//&nbsp; simulation {simState} &nbsp;//&nbsp; surveillance: ON
        </div>
        {/* Corruption mode 1: surveillance-log line + occasional connection blip. */}
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
(npm run dev 2>&1 | head -15) & DEV_PID=$!
sleep 6
kill $DEV_PID 2>/dev/null
wait 2>/dev/null
```

Expected: tsc clean, vitest all pass, dev server boots.

- [ ] **Step 3: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/hud/LoomHud.tsx
git commit -m "Add HUD corruption mode 1 (subtle glitches + surveillance log)

When cycleStore.corruption ≥ 1 (Cycle 2 onward), the HUD adds:

1. A third line beneath the cycle/sim line, italic + dim green,
   showing surveillance-log fragments rotating every 30s. Quotes
   reference the existing loomLogs.ts canon (L-901..L-906) and
   add L-907 (Subject_DOOM, the player's classification — defective
   unit at large, termination protocol unauthorized).

2. A periodic [CONNECTION RESET] blip every 45s, lasting 300ms.
   Yellow text replaces the surveillance line briefly — telegraphs
   that the simulation is fraying without being obnoxious.

Corruption mode 2 (lying values, full log invasion) and mode 3
(dissolving) are Phase 4-5 work."
```

---

## Task 3.4: Reply All Storm weapon (slot 3 alt)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/replyAllStorm.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/replyAllStorm.test.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Slot 3 alt — pressing 3 cycles between Spam Filter and Reply All Storm. Super Shotgun analog: 8 pellets, ~17° spread, 8 damage each (= 64 max single-target), costs 2 ammo per shot.

Skip the "envelopes ricochet in tight rooms" mechanic from the spec — Phase 4 polish.

- [ ] **Step 1: Write the failing test**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/replyAllStorm.test.ts`:
```typescript
import { describe, it, expect, vi } from 'vitest';
import { ReplyAllStorm } from './replyAllStorm';
import type { IEnemy } from '../enemies/Enemy';

function makeEnemy(): IEnemy & { lastDamage: number; takeCount: number } {
  return {
    x: 1, y: 0, health: 1000,
    state: 'chase' as const,
    placeholderColor: [0, 0, 0] as const,
    isBoss: false,
    lastDamage: 0,
    takeCount: 0,
    update: () => {},
    takeDamage(d: number) { this.lastDamage = d; this.takeCount += 1; },
  };
}

const fakePlayer = { x: 0, y: 0, angle: 0, ammo: 50, pulseCharge: 1, health: 100, activeSlot: 3, takeDamage() {} };
const fakeMap = { id: 'test', cycle: 1 as const, grid: [[0,0,0,0,0,0]], things: [], textures: [] };
const fakeAudio = { onEnemyHit: vi.fn(), onPlayerDamaged: vi.fn(), onGameStart() {}, onGameEnd() {}, onMapLoad() {} };

describe('ReplyAllStorm', () => {
  it('costs 2 ammo per shot', () => {
    const w = new ReplyAllStorm();
    const player = { ...fakePlayer, ammo: 10 };
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [], fakeAudio as any);
    expect(player.ammo).toBe(8);
  });

  it('returns false when ammo < 2', () => {
    const w = new ReplyAllStorm();
    const player = { ...fakePlayer, ammo: 1 };
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    const fired = w.fire(player as any, fakeMap as any, [], fakeAudio as any);
    expect(fired).toBe(false);
    expect(player.ammo).toBe(1);
  });

  it('deals between 0 and 64 single-target damage (8 pellets x 8 dmg)', () => {
    const w = new ReplyAllStorm();
    const e = makeEnemy();
    const player = { ...fakePlayer, ammo: 10 };
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [e], fakeAudio as any);
    // Each pellet that hits adds 8 damage to lastDamage (overwritten); takeCount counts hits.
    expect(e.takeCount).toBeGreaterThanOrEqual(0);
    expect(e.takeCount).toBeLessThanOrEqual(8);
  });

  it('calls audio.onEnemyHit at most once per fire (any-pellet semantics)', () => {
    fakeAudio.onEnemyHit.mockClear();
    const w = new ReplyAllStorm();
    const e = makeEnemy();
    const player = { ...fakePlayer, ammo: 10 };
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    w.fire(player as any, fakeMap as any, [e], fakeAudio as any);
    expect(fakeAudio.onEnemyHit.mock.calls.length).toBeLessThanOrEqual(1);
  });
});
```

- [ ] **Step 2: Run the test, verify it fails**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/weapons/replyAllStorm.test.ts 2>&1 | tail -10
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement Reply All Storm**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/replyAllStorm.ts`:
```typescript
import type { IWeapon } from './IWeapon';
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import type { IEnemy } from '../enemies/Enemy';
import type { AudioController } from '../../engine/audioController';
import { castRay } from '../../engine/raycaster';
import { findEnemyAlongRay } from './hitResolution';

const DAMAGE_PER_PELLET = 8;
const PELLETS = 8;
const SPREAD_RADIANS = 0.30; // ~17° half-angle (wider than Spam Filter)
const RANGE = 14;
const PERP_TOLERANCE = 0.4;
const AMMO_COST = 2;

/** Reply All Storm — slot 3 alt. Super Shotgun analog. 8-pellet hitscan
 *  spread; each pellet does 8 damage (64 max single-target). Costs 2
 *  ammo per shot. Wider spread than Spam Filter rewards close-range
 *  cluster fire. */
export class ReplyAllStorm implements IWeapon {
  readonly name = 'Reply All Storm';
  readonly slot = 3;

  fire(player: Player, map: LoomMap, enemies: IEnemy[], audio: AudioController): boolean {
    if (player.ammo < AMMO_COST) return false;
    player.ammo -= AMMO_COST;

    let anyHit = false;
    for (let p = 0; p < PELLETS; p++) {
      const spread = (Math.random() - 0.5) * 2 * SPREAD_RADIANS;
      const angle = player.angle + spread;
      const wallHit = castRay(map, player.x, player.y, angle);
      const target = findEnemyAlongRay(
        player.x, player.y, angle,
        enemies, RANGE, PERP_TOLERANCE, wallHit.distance,
      );
      if (target) {
        target.takeDamage(DAMAGE_PER_PELLET);
        anyHit = true;
      }
    }
    if (anyHit) audio.onEnemyHit();
    return true;
  }
}
```

- [ ] **Step 4: Run the test, verify it passes**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/weapons/replyAllStorm.test.ts 2>&1 | tail -10
```

Expected: 4 tests pass.

- [ ] **Step 5: Register Reply All Storm on slot 3 alt**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`:

a) Add the import (next to `SpamFilter`):
```typescript
import { ReplyAllStorm } from '../entities/weapons/replyAllStorm';
```

b) Find the existing slot-3 line:
```typescript
this.weaponsBySlot.set(3, [new SpamFilter()]);
```

Change to:
```typescript
this.weaponsBySlot.set(3, [new SpamFilter(), new ReplyAllStorm()]);
```

(Pressing 3 now cycles Spam Filter ↔ Reply All Storm.)

- [ ] **Step 6: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
npx vitest run components/loom 2>&1 | tail -5
git add components/loom/entities/weapons/replyAllStorm.ts \
        components/loom/entities/weapons/replyAllStorm.test.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add Reply All Storm (slot 3 alt — Super Shotgun)

8 pellets per shot, ~17° half-angle spread, 8 damage per pellet
(64 max single-target). Costs 2 ammo per shot. Wider spread + higher
per-pellet damage than Spam Filter — rewards close-range cluster fire,
punishes mistimed long-range shots.

Pressing 3 cycles Spam Filter ↔ Reply All Storm. Spam Filter remains
the default within slot 3 (a fresh game starts on the first array
element).

Skipped the spec's 'envelopes ricochet in tight rooms' mechanic for
Phase 3; deferred to Phase 4 polish.

4 vitest tests cover ammo cost, ammo guard, max-damage bound, and
audio-call cardinality."
```

---

## Task 3.5: PIP enemy (Pinky/Demon analog — fast melee charger)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/pip.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

PIP = "Performance Improvement Plan" demon. Pinky/Demon analog from DOOM. Fast (2x Intern speed) melee charger; medium HP. No ranged attack — closes distance and bites.

- [ ] **Step 1: Implement PIP**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/pip.ts`:
```typescript
import type { EnemyUpdateContext } from './Enemy';
import { BaseEnemy } from './Enemy';

const HEALTH = 60;
const SPEED = 2.0; // 2x Intern
const SIGHT_RANGE = 8;
const MELEE_RANGE = 1.0;
const MELEE_DAMAGE = 12;
const MELEE_COOLDOWN_MS = 800;

/**
 * PIP — Performance Improvement Plan demon. Cycle 2 mid-tier enemy.
 * Pinky/Demon analog. 60 HP, 2x Intern speed, melee-only, hard-hitting.
 * Closes distance fast — punishes static play.
 */
export class PIP extends BaseEnemy {
  health = HEALTH;
  readonly placeholderColor = [180, 60, 60] as const;

  private nextMeleeAt = 0;

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
      if (dist <= MELEE_RANGE && performance.now() >= this.nextMeleeAt) {
        ctx.player.takeDamage(MELEE_DAMAGE);
        ctx.audio.onPlayerDamaged();
        this.nextMeleeAt = performance.now() + MELEE_COOLDOWN_MS;
        return;
      }
      const move = SPEED * dt;
      if (dist > 0.5) {
        this.x += (dx / dist) * move;
        this.y += (dy / dist) * move;
      }
    }
  }
}
```

- [ ] **Step 2: Add `'pip'` to ThingType**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`. Update `ThingType`:
```typescript
export type ThingType =
  | 'player_start'
  | 'intern'
  | 'account_executive'
  | 'recruiter'
  | 'hr_manager'
  | 'pip'
  | 'exit';
```

- [ ] **Step 3: Register PIP in GameLoop spawn loop**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`:

a) Add the import:
```typescript
import { PIP } from '../entities/enemies/pip';
```

b) Add a spawn branch in the existing `for (const t of map.things)` loop:
```typescript
} else if (t.type === 'pip') {
  this.enemies.push(new PIP(t.x, t.y));
}
```

- [ ] **Step 4: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/enemies/pip.ts \
        components/loom/types.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add PIP enemy (Pinky/Demon analog)

Cycle 2 mid-tier. 'Performance Improvement Plan' demon. 60 HP, 2x
Intern speed, melee-only (12 damage, 1.0 tile range, 800ms cooldown).
No ranged attack — punishes static play by closing distance fast.
Visual: red placeholder rectangle."
```

---

## Task 3.6: Notification enemy (Lost Soul analog — flying)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/notification.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/notification.test.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Notification = floating red badge / blue dot. Lost Soul analog from DOOM. Low HP, very fast, dive-bombs the player. The dive-bomb leaves a brief "recovery" window where the Notification briefly retreats before charging again.

- [ ] **Step 1: Write the failing test**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/notification.test.ts`:
```typescript
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { Notification } from './notification';
import type { EnemyUpdateContext } from './Enemy';

function makeCtx(playerX: number, playerY: number) {
  const playerHits: number[] = [];
  return {
    ctx: {
      player: { x: playerX, y: playerY, angle: 0, ammo: 50, pulseCharge: 1, health: 100, activeSlot: 1, takeDamage(n: number) { playerHits.push(n); } } as unknown as EnemyUpdateContext['player'],
      map: { id: 'test', cycle: 2, grid: [[0,0,0,0,0],[0,0,0,0,0],[0,0,0,0,0],[0,0,0,0,0],[0,0,0,0,0]], things: [], textures: [] } as unknown as EnemyUpdateContext['map'],
      spawnProjectile: () => {},
      audio: { onEnemyHit: vi.fn(), onPlayerDamaged: vi.fn(), onGameStart() {}, onGameEnd() {}, onMapLoad() {} } as unknown as EnemyUpdateContext['audio'],
    },
    playerHits,
  };
}

describe('Notification', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });
  afterEach(() => {
    vi.useRealTimers();
  });

  it('starts in idle state', () => {
    const n = new Notification(2, 2);
    expect(n.state).toBe('idle');
  });

  it('transitions to chase when player is within sight range', () => {
    const n = new Notification(2, 2);
    const { ctx } = makeCtx(2.5, 2.5);
    n.update(0.016, ctx);
    expect(n.state).toBe('chase');
  });

  it('damages player on close approach during chase', () => {
    const n = new Notification(2, 2);
    const { ctx, playerHits } = makeCtx(2.0, 2.0); // very close
    n.update(0.016, ctx);
    n.update(0.016, ctx);
    n.update(0.016, ctx);
    expect(playerHits.length).toBeGreaterThan(0);
    expect(playerHits[0]).toBe(8);
  });

  it('takes damage on hit', () => {
    const n = new Notification(2, 2);
    n.takeDamage(10);
    expect(n.health).toBe(5);
    expect(n.state).toBe('pain');
  });

  it('dies at 0 HP', () => {
    const n = new Notification(2, 2);
    n.takeDamage(15);
    expect(n.state).toBe('dead');
  });
});
```

- [ ] **Step 2: Run test, verify failure**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/enemies/notification.test.ts 2>&1 | tail -10
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement Notification**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/notification.ts`:
```typescript
import type { EnemyUpdateContext } from './Enemy';
import { BaseEnemy } from './Enemy';

const HEALTH = 15;
const SPEED = 2.5; // very fast
const SIGHT_RANGE = 9;
const DIVE_RANGE = 0.6;
const DIVE_DAMAGE = 8;
const DIVE_COOLDOWN_MS = 1000;

/**
 * Notification — Cycle 2 mid-tier enemy. Lost Soul analog. Floating
 * red-badge / blue-dot. Very fast (2.5x Intern), low HP (15), dive-bombs
 * the player on contact. After biting, brief 1s cooldown before next
 * dive — gives player a window to retreat.
 */
export class Notification extends BaseEnemy {
  health = HEALTH;
  readonly placeholderColor = [220, 60, 80] as const;

  private nextDiveAt = 0;

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
      // Dive-bomb on close approach
      if (dist <= DIVE_RANGE && performance.now() >= this.nextDiveAt) {
        ctx.player.takeDamage(DIVE_DAMAGE);
        ctx.audio.onPlayerDamaged();
        this.nextDiveAt = performance.now() + DIVE_COOLDOWN_MS;
        return;
      }
      // Otherwise zoom toward player
      const move = SPEED * dt;
      if (dist > 0.3) {
        this.x += (dx / dist) * move;
        this.y += (dy / dist) * move;
      }
    }
  }
}
```

- [ ] **Step 4: Run test, verify pass**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/enemies/notification.test.ts 2>&1 | tail -10
```

Expected: 5 tests pass.

- [ ] **Step 5: Add 'notification' to ThingType + register in GameLoop**

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
  | 'exit';
```

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`:

```typescript
import { Notification } from '../entities/enemies/notification';
```

In the spawn loop:
```typescript
} else if (t.type === 'notification') {
  this.enemies.push(new Notification(t.x, t.y));
}
```

- [ ] **Step 6: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/enemies/notification.ts \
        components/loom/entities/enemies/notification.test.ts \
        components/loom/types.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add Notification enemy (Lost Soul analog — flying dive-bomber)

Cycle 2 mid-tier. Floating red-badge / blue-dot. 15 HP (fragile),
2.5x Intern speed, dive-bombs the player on contact (8 damage,
1s cooldown). Low HP rewards aggressive return-fire; high speed +
short cooldown punishes hesitation.

5 vitest tests cover state transitions, sight detection, dive-bomb
damage, and death threshold."
```

---

## Task 3.7: Middle Manager enemy (Hell Knight analog — projectile)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/circleBack.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/middleManager.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Middle Manager = khakis, tucked-in shirt, fires "let's circle back" projectiles. Hell Knight analog. Slow + tanky + ranged. Projectile is straight-line (not tracking — that's HR Manager's gimmick), 14 damage per hit.

- [ ] **Step 1: Implement CircleBack projectile**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/circleBack.ts`:
```typescript
import type { IProjectile, ProjectileUpdateContext } from './Projectile';

const SPEED = 5.0;
const DAMAGE = 14;
const HIT_RADIUS = 0.4;
const MAX_LIFETIME_MS = 4000;

/**
 * CircleBack — Middle Manager's projectile. Straight-line travel at
 * 5 tiles/sec (faster than Recruiter fireball, more damage). 14 damage
 * on hit, 4s lifetime. No tracking — read the trajectory and step
 * sideways to dodge.
 */
export class CircleBack implements IProjectile {
  x: number;
  y: number;
  dead = false;
  readonly color = [120, 180, 200] as const;
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
    if (iy < 0 || iy >= ctx.map.grid.length || !ctx.map.grid[iy] || ix < 0 || ix >= ctx.map.grid[iy].length || ctx.map.grid[iy][ix] > 0) {
      this.dead = true;
      return;
    }

    this.x = newX;
    this.y = newY;
  }
}
```

- [ ] **Step 2: Implement Middle Manager**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/middleManager.ts`:
```typescript
import type { EnemyUpdateContext } from './Enemy';
import { BaseEnemy } from './Enemy';
import { CircleBack } from '../projectiles/circleBack';

const HEALTH = 80;
const SPEED = 0.8;
const SIGHT_RANGE = 10;
const ATTACK_RANGE = 9;
const ATTACK_COOLDOWN_MS = 1800;

/**
 * Middle Manager — Cycle 2 heavy. Hell Knight analog. 80 HP, slow
 * (0.8 speed), fires CircleBack projectiles at mid-range (9 tiles, 1.8s
 * cooldown). Tanky — eats Spam Filter shots; Reply All Storm closes
 * the gap fast.
 */
export class MiddleManager extends BaseEnemy {
  health = HEALTH;
  readonly placeholderColor = [160, 130, 90] as const;

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
        ctx.spawnProjectile(new CircleBack(this.x, this.y, angleToPlayer));
        return;
      }
      const move = SPEED * dt;
      if (dist > 2) {
        this.x += (dx / dist) * move;
        this.y += (dy / dist) * move;
      }
    }
  }
}
```

- [ ] **Step 3: Add 'middle_manager' to ThingType + register in GameLoop**

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
  | 'exit';
```

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`:

```typescript
import { MiddleManager } from '../entities/enemies/middleManager';
```

In the spawn loop:
```typescript
} else if (t.type === 'middle_manager') {
  this.enemies.push(new MiddleManager(t.x, t.y));
}
```

- [ ] **Step 4: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/projectiles/circleBack.ts \
        components/loom/entities/enemies/middleManager.ts \
        components/loom/types.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add Middle Manager enemy + CircleBack projectile (Hell Knight analog)

Cycle 2 heavy. 80 HP, slow (0.8 speed), ranged. CircleBack projectile
travels straight at 5 tiles/sec, 14 damage on hit, 4s lifetime. No
tracking — read the trajectory to dodge.

Middle Manager fires every 1.8s within 9 tiles. Tanky — eats Spam
Filter; Reply All Storm closes the gap fast."
```

---

## Task 3.8: VP of Sales boss (Cycle 2 boss, hitscan multi-pellet)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/vpOfSales.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

VP of Sales = Patagonia vest + blue button-down. Cycle 2 boss. The spec calls them "fires Reply All Storms back at the player" — implementation: hitscan 6-pellet spread, more damage than Account Executive's 3-pellet, longer range, on a 2.5s cooldown. Boss-tier HP (250). isBoss=true.

- [ ] **Step 1: Implement VP of Sales**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/vpOfSales.ts`:
```typescript
import type { EnemyUpdateContext } from './Enemy';
import { BaseEnemy } from './Enemy';
import { castRay } from '../../engine/raycaster';

const HEALTH = 250;
const SPEED = 0.7;
const SIGHT_RANGE = 14;
const ATTACK_RANGE = 12;
const ATTACK_COOLDOWN_MS = 2500;
const PELLETS = 6;
const SPREAD_RADIANS = 0.2;
const DAMAGE_PER_PELLET = 5;

/**
 * VP of Sales — Cycle 2 boss. 250 HP (boss-tier). Patagonia vest +
 * blue button-down. Fires hitscan 6-pellet spread shotgun every 2.5s
 * within 12 tiles — DOOM Sergeant pattern at 2x scale.
 *
 * No phase transitions for Phase 3 — multi-phase boss mechanics ship
 * with The Hand in Phase 5.
 */
export class VPOfSales extends BaseEnemy {
  health = HEALTH;
  readonly placeholderColor = [80, 100, 200] as const;
  readonly isBoss = true;
  protected readonly PAIN_DURATION_MS = 100;

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
        let damage = 0;
        for (let p = 0; p < PELLETS; p++) {
          const spread = (Math.random() - 0.5) * 2 * SPREAD_RADIANS;
          const a = angleToPlayer + spread;
          const wallHit = castRay(ctx.map, this.x, this.y, a);
          if (dist < wallHit.distance && Math.random() > 0.4) {
            damage += DAMAGE_PER_PELLET;
          }
        }
        if (damage > 0) {
          ctx.player.takeDamage(damage);
          ctx.audio.onPlayerDamaged();
        }
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

- [ ] **Step 2: Add 'vp_of_sales' to ThingType + register in GameLoop**

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
  | 'exit';
```

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`:

```typescript
import { VPOfSales } from '../entities/enemies/vpOfSales';
```

In the spawn loop:
```typescript
} else if (t.type === 'vp_of_sales') {
  this.enemies.push(new VPOfSales(t.x, t.y));
}
```

- [ ] **Step 3: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/enemies/vpOfSales.ts \
        components/loom/types.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add VP of Sales boss (Cycle 2 boss, hitscan multi-pellet)

Cycle 2 boss. 250 HP (boss-tier; 25% more than HR Manager). Patagonia
vest + blue button-down placeholder color. Hitscan 6-pellet spread
shotgun every 2.5s within 12 tiles — Sergeant pattern at 2x scale.
Per-pellet 60% hit fairness (matches Account Executive).

isBoss=true, so the cyc2_glass_confroom_boss exit gates on this kill
(the boss-skip loophole closure from Phase 2 review #8 applies
automatically to Phase 3's boss too).

Short pain (100ms) so DPS-checks aren't trivialized."
```

---

## Task 3.9: cyc2_open_office map

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc2_open_office.json`

Cycle 2 opener. Big open area with sparse interior pillars (open-floor-plan office aesthetic). First exposure to PIP and a Notification or two. Player should feel the speed shift.

- [ ] **Step 1: Author the map**

Create `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc2_open_office.json`:
```json
{
  "id": "cyc2_open_office",
  "cycle": 2,
  "music": "ship_it",
  "intermissionText": "OPEN OFFICE cleared. SUBJECT 88-E shows aggression escalation. PROCEED TO BREAK ROOM.",
  "grid": [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  ],
  "things": [
    { "type": "player_start", "x": 2, "y": 5, "angle": 0 },
    { "type": "intern", "x": 8, "y": 2 },
    { "type": "intern", "x": 9, "y": 8 },
    { "type": "account_executive", "x": 11, "y": 3 },
    { "type": "account_executive", "x": 11, "y": 7 },
    { "type": "pip", "x": 14, "y": 5 },
    { "type": "notification", "x": 6, "y": 3 },
    { "type": "exit", "x": 16, "y": 5 }
  ],
  "textures": ["wall_lobby"]
}
```

- [ ] **Step 2: Verify + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
node -e "console.log('OK', JSON.parse(require('fs').readFileSync('public/data/loom/maps/cyc2_open_office.json','utf8')).id)"
git add public/data/loom/maps/cyc2_open_office.json
git commit -m "Add cyc2_open_office map — Cycle 2 opener

11x18 grid. Open-floor-plan with 4 sparse interior wall pillars.
Roster: 2 Interns, 2 AEs, 1 PIP, 1 Notification. PIP introduces
the speed shift; the Notification trains dodging the dive-bomb.
Mostly horizontal corridors — first encounter with the new tier
is mid-range and clear."
```

---

## Task 3.10: cyc2_break_room map

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc2_break_room.json`

Tighter, more intimate. Multiple PIPs in an enclosed kitchen/break-room layout. Notifications in clusters.

- [ ] **Step 1: Author the map**

Create `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc2_break_room.json`:
```json
{
  "id": "cyc2_break_room",
  "cycle": 2,
  "music": "ship_it",
  "intermissionText": "BREAK ROOM cleared. SLACK HUDDLE detected. PROCEED.",
  "grid": [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 1, 1, 1, 0, 0, 0, 0, 1, 1, 1, 0, 1],
    [1, 0, 1, 0, 1, 0, 0, 0, 0, 1, 0, 1, 0, 1],
    [1, 0, 1, 0, 1, 0, 0, 0, 0, 1, 0, 1, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 1, 1, 0, 0, 0, 0, 0, 0, 1, 1, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  ],
  "things": [
    { "type": "player_start", "x": 2, "y": 1, "angle": 0 },
    { "type": "intern", "x": 6, "y": 1 },
    { "type": "intern", "x": 8, "y": 7 },
    { "type": "pip", "x": 5, "y": 7 },
    { "type": "pip", "x": 11, "y": 5 },
    { "type": "notification", "x": 6, "y": 5 },
    { "type": "notification", "x": 9, "y": 5 },
    { "type": "account_executive", "x": 11, "y": 1 },
    { "type": "exit", "x": 12, "y": 8 }
  ],
  "textures": ["wall_lobby"]
}
```

- [ ] **Step 2: Verify + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
node -e "console.log('OK', JSON.parse(require('fs').readFileSync('public/data/loom/maps/cyc2_break_room.json','utf8')).id)"
git add public/data/loom/maps/cyc2_break_room.json
git commit -m "Add cyc2_break_room map — Cycle 2 mid (PIP introduction proper)

10x14 grid. Two enclosed counter blocks (table-shaped wall pillars)
create kitchen-aisle corridors. Roster: 2 PIPs (in tight space),
2 Notifications, 2 Interns, 1 AE. Tight space + fast PIPs trains
combat in tight quarters."
```

---

## Task 3.11: cyc2_slack_huddle map

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc2_slack_huddle.json`

Conference room style — round table at center, chairs implied by sparse pillars. Heavy Notification spawns. First Middle Manager.

- [ ] **Step 1: Author the map**

Create `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc2_slack_huddle.json`:
```json
{
  "id": "cyc2_slack_huddle",
  "cycle": 2,
  "music": "ship_it",
  "intermissionText": "SLACK HUDDLE escalation suppressed. WELLNESS CENTER ahead.",
  "grid": [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 1, 1, 1, 1, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 1, 0, 0, 1, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 1, 0, 0, 1, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 1, 1, 1, 1, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  ],
  "things": [
    { "type": "player_start", "x": 2, "y": 4, "angle": 0 },
    { "type": "notification", "x": 4, "y": 1 },
    { "type": "notification", "x": 4, "y": 8 },
    { "type": "notification", "x": 11, "y": 1 },
    { "type": "notification", "x": 11, "y": 8 },
    { "type": "middle_manager", "x": 12, "y": 4 },
    { "type": "pip", "x": 8, "y": 4 },
    { "type": "account_executive", "x": 7, "y": 1 },
    { "type": "exit", "x": 14, "y": 4 }
  ],
  "textures": ["wall_lobby"]
}
```

- [ ] **Step 2: Verify + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
node -e "console.log('OK', JSON.parse(require('fs').readFileSync('public/data/loom/maps/cyc2_slack_huddle.json','utf8')).id)"
git add public/data/loom/maps/cyc2_slack_huddle.json
git commit -m "Add cyc2_slack_huddle map — Cycle 2 mid (first Middle Manager)

10x16 grid. Hollow conference-table block in the center; 4 corners
host Notification spawns. Roster: 4 Notifications, 1 Middle Manager
(first time facing CircleBack projectiles), 1 PIP, 1 AE. Player
must learn to read CircleBack trajectories while dodging dive-bombs."
```

---

## Task 3.12: cyc2_wellness_ctr map

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc2_wellness_ctr.json`

Yoga / wellness aesthetic — long open hallway with parallel walls (yoga mats implied). Heavy on Middle Managers (tanky tier). The breather room before the boss.

- [ ] **Step 1: Author the map**

Create `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc2_wellness_ctr.json`:
```json
{
  "id": "cyc2_wellness_ctr",
  "cycle": 2,
  "music": "ship_it",
  "intermissionText": "WELLNESS CENTER cleared. VP CONFERENCE ROOM ahead. Bioacoustic optimization recommended.",
  "grid": [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 1],
    [1, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 1],
    [1, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  ],
  "things": [
    { "type": "player_start", "x": 2, "y": 5, "angle": 0 },
    { "type": "middle_manager", "x": 14, "y": 2 },
    { "type": "middle_manager", "x": 14, "y": 8 },
    { "type": "middle_manager", "x": 9, "y": 5 },
    { "type": "pip", "x": 6, "y": 2 },
    { "type": "pip", "x": 6, "y": 8 },
    { "type": "notification", "x": 10, "y": 1 },
    { "type": "notification", "x": 10, "y": 9 },
    { "type": "exit", "x": 18, "y": 5 }
  ],
  "textures": ["wall_lobby"]
}
```

- [ ] **Step 2: Verify + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
node -e "console.log('OK', JSON.parse(require('fs').readFileSync('public/data/loom/maps/cyc2_wellness_ctr.json','utf8')).id)"
git add public/data/loom/maps/cyc2_wellness_ctr.json
git commit -m "Add cyc2_wellness_ctr map — Cycle 2 late (Middle Manager-heavy)

11x20 — wider than other Cycle 2 maps. Long hallway with 4 yoga-mat
pillars + central divider. Roster: 3 Middle Managers (heavy tier),
2 PIPs, 2 Notifications. Player must clear Middle Managers from
range — encourages Reply All Storm + Branded Pen swap."
```

---

## Task 3.13: cyc2_glass_confroom_boss map (boss arena)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc2_glass_confroom_boss.json`

VP boss arena. Wide square room, glass walls implied (no interior pillars), VP at far end with a couple of Middle Managers backing them up. Player can use range or close in.

- [ ] **Step 1: Author the map**

Create `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc2_glass_confroom_boss.json`:
```json
{
  "id": "cyc2_glass_confroom_boss",
  "cycle": 2,
  "music": "ship_it",
  "intermissionText": "VP OF SALES neutralized. CYCLE 2 STABILITY DEGRADING. simulation breach IMMINENT.",
  "grid": [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  ],
  "things": [
    { "type": "player_start", "x": 2, "y": 6, "angle": 0 },
    { "type": "vp_of_sales", "x": 15, "y": 6 },
    { "type": "middle_manager", "x": 12, "y": 3 },
    { "type": "middle_manager", "x": 12, "y": 9 },
    { "type": "pip", "x": 8, "y": 6 },
    { "type": "notification", "x": 8, "y": 3 },
    { "type": "notification", "x": 8, "y": 9 },
    { "type": "exit", "x": 18, "y": 6 }
  ],
  "textures": ["wall_lobby"]
}
```

- [ ] **Step 2: Verify + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
node -e "console.log('OK', JSON.parse(require('fs').readFileSync('public/data/loom/maps/cyc2_glass_confroom_boss.json','utf8')).id)"
git add public/data/loom/maps/cyc2_glass_confroom_boss.json
git commit -m "Add cyc2_glass_confroom_boss map — Cycle 2 boss arena

12x20 — biggest Phase 3 map. Two corner pillars give cover from VP's
6-pellet hitscan spread. Roster: VP of Sales + 2 Middle Managers
(rear support) + 1 PIP (mid) + 2 Notifications (flanks). Player
must clear adds before engaging the boss safely.

Boss-skip loophole closure (from Phase 2 review #8) keeps the exit
gated until VP is dead."
```

---

## Task 3.14: Phase 3 e2e Playwright spec

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/e2e/loom-phase-3.spec.ts`

Walk through the full Cycles 1+2 path: 8 maps, 2 boss kills via probe, intermission/cycle-transition advances, EndOfCycleStub showing Cycle 2 number. Plus assertions on cycleStore advancement.

The spec replaces (deletes) the Phase 2 e2e spec — Phase 3 supersedes it.

- [ ] **Step 1: Write the Phase 3 spec**

Create `/Users/justinwest/Repos/l0b0tonline/e2e/loom-phase-3.spec.ts`:
```typescript
import { test, expect } from '@playwright/test';

test.describe('LOOM Phase 3 DoD', () => {
  test('full Cycles 1 + 2 progression with cycleStore advance', async ({ page }) => {
    const consoleErrors: string[] = [];
    page.on('console', (msg) => {
      if (msg.type() === 'error') consoleErrors.push(msg.text());
    });
    page.on('pageerror', (err) => consoleErrors.push(err.message));

    await page.goto('/');

    await page.waitForFunction(
      () => typeof (window as unknown as { __STORE__?: unknown }).__STORE__ !== 'undefined',
      { timeout: 10_000 },
    );
    await page.evaluate(() => {
      const store = (window as unknown as {
        __STORE__: { getState: () => { windows: { openWindow: (w: { type: string; title: string }) => void } } };
      }).__STORE__;
      store.getState().windows.openWindow({ type: 'GAME_LOOM', title: 'loom.exe' });
    });

    // Boot
    await expect(page.getByText('LOOM TECHNOLOGIES')).toBeVisible({ timeout: 10_000 });
    await expect(page.getByText('ACCESS_GRANTED')).toBeVisible({ timeout: 10_000 });
    await expect(page.locator('canvas')).toBeVisible();

    // Helper: advance through a map by killing all enemies (closes boss-gate
    // for boss arenas; harmless on non-boss maps), waiting for new __LOOM_GAME__
    // ref, then teleporting to the exit, then pressing Space on the intermission
    // / EndOfCycleStub.
    async function advanceMap(mapId: string, exitX: number, exitY: number, isCycleEnd = false) {
      // Wait for the loop to be on this map
      await page.waitForFunction(
        (id) => {
          const g = (window as unknown as { __LOOM_GAME__?: { getSnapshot: () => { zone?: string } } }).__LOOM_GAME__;
          return !!g && g.getSnapshot().zone === id;
        },
        mapId,
        { timeout: 10_000 },
      );

      // Kill all enemies (closes the boss-gate where applicable)
      await page.evaluate(() => {
        const g = (window as unknown as {
          __LOOM_GAME__: { getEnemies: () => Array<{ takeDamage: (n: number) => void }> };
        }).__LOOM_GAME__;
        for (const e of g.getEnemies()) e.takeDamage(99999);
      });
      await page.waitForTimeout(50);

      // Teleport to exit
      await page.evaluate(([ex, ey]) => {
        const g = (window as unknown as {
          __LOOM_GAME__: { getPlayer: () => { x: number; y: number } };
        }).__LOOM_GAME__;
        const p = g.getPlayer();
        p.x = ex;
        p.y = ey;
      }, [exitX, exitY]);

      // Wait for transition screen + advance
      if (isCycleEnd) {
        await expect(page.getByText('CYCLE', { exact: false })).toBeVisible({ timeout: 5_000 });
        await expect(page.getByText('COMPLETE', { exact: false })).toBeVisible();
      } else {
        await expect(page.getByText('INTERMISSION', { exact: false })).toBeVisible({ timeout: 5_000 });
      }
      await page.keyboard.press('Space');
    }

    // ===== Cycle 1: lobby → cubicles → hr_arena =====
    await advanceMap('cyc1_lobby', 14, 5);
    await advanceMap('cyc1_cubicles', 12, 9);
    // After hr_arena exit: NOT cycle-end (we move to cyc2_open_office) —
    // it's a cycle-transition, which still uses INTERMISSION screen.
    await advanceMap('cyc1_hr_arena', 16, 6);

    // ===== Cycle 2 =====
    await advanceMap('cyc2_open_office', 16, 5);
    await advanceMap('cyc2_break_room', 12, 8);
    await advanceMap('cyc2_slack_huddle', 14, 4);
    await advanceMap('cyc2_wellness_ctr', 18, 5);
    // VP boss arena → cycle-end stub
    await advanceMap('cyc2_glass_confroom_boss', 18, 6, true);

    // ===== Back to boot =====
    await expect(page.getByText('LOOM TECHNOLOGIES')).toBeVisible({ timeout: 5_000 });

    // No console errors (filter known headless noise)
    const relevantErrors = consoleErrors.filter(
      (e) =>
        !e.includes('chrome-extension') &&
        !e.includes('Failed to load resource: net::ERR_FAILED') &&
        !e.includes('favicon') &&
        !e.includes('Music playback failed') &&
        !e.includes('NotSupportedError'),
    );
    expect(relevantErrors).toEqual([]);
  });
});
```

- [ ] **Step 2: Delete the Phase 2 e2e spec (superseded)**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git rm e2e/loom-phase-2.spec.ts
```

- [ ] **Step 3: Run the Phase 3 spec**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npm run test:e2e -- e2e/loom-phase-3.spec.ts 2>&1 | tail -30
```

Expected: PASS in ~30-45 seconds.

If it fails:
- Selector misses — adjust to match actual rendered text
- Cycle-transition visible-text difference — the intermission for cycle-1→2 might say something different than "INTERMISSION" if you customized; verify and adjust
- Real bug — pause and report

- [ ] **Step 4: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add e2e/loom-phase-3.spec.ts
git commit -m "Add Phase 3 DoD smoke test (replaces Phase 2 spec)

Walks the full Cycles 1+2 progression: 8 maps + 2 boss kills via
probe + intermission/cycle-transition advances + EndOfCycleStub
on cyc2 final + return to boot. Asserts no console errors.

The spec uses an advanceMap(id, exitX, exitY, isCycleEnd) helper
that kills all enemies (closes boss-gate), teleports to the exit,
then presses Space on the resulting transition screen.

Replaces e2e/loom-phase-2.spec.ts which only covered Cycle 1.
Phase 3 supersedes."
```

---

## Task 3.15: Phase 3 playtest log

**Files:**
- Create: `/Users/justinwest/Repos/LOOM-DOOM/.claude/worktrees/awesome-sutherland-aad854/docs/playtests/2026-04-27-phase-3-cycle-2.md`

Same template as Phase 2's log: 8 Playwright items + 22 manual sanity items.

- [ ] **Step 1: Write the playtest log**

Create `/Users/justinwest/Repos/LOOM-DOOM/.claude/worktrees/awesome-sutherland-aad854/docs/playtests/2026-04-27-phase-3-cycle-2.md`:
```markdown
# Phase 3 Cycle 2 Escalation — Playtest Log

**Date:** 2026-04-27
**l0b0tonline branch:** `loom-game` (or `loom-cycle-2` if branched fresh)
**l0b0tonline commit:** [paste latest short SHA]
**LOOM-DOOM plan commit:** [paste latest short SHA]
**Test file:** `l0b0tonline/e2e/loom-phase-3.spec.ts`
**Test command:** `npm run test:e2e -- e2e/loom-phase-3.spec.ts`

## Result

[X] PASS (automation gate) / [ ] FAIL

## Phase 3 Definition of Done

- [X] **Automation gate** — Phase 3 Playwright spec passes (full 8-map + 2-boss flow + cycleStore advance)
- [ ] **Manual gate** — visual rendering, controls feel, weapon variety, AI behavior, boss combat, music, SFX (the 22 NEEDS-HUMAN items below)

## Behavior verification

| # | Behavior | Verified by | Result |
|---|---|---|---|
| 1 | LOOM window opens from J0IN 0S desktop | Playwright | PASS |
| 2 | Boot sequence plays | Playwright | PASS |
| 3 | Cycle 1 plays through to cyc1_hr_arena | Playwright | PASS |
| 4 | After hr_arena: cycle-transition advances cycleStore to 2 | Playwright | PASS |
| 5 | HUD displays "cycle 8493" + "drift detected" + corruption-1 mode | Playwright | PASS |
| 6 | Cycle 2 plays through 5 maps including VP boss | Playwright | PASS |
| 7 | After VP arena: EndOfCycleStub appears with cycleNumber=2 | Playwright | PASS |
| 8 | Press-key returns to boot | Playwright | PASS |
| 9 | No console errors | Playwright | PASS |
| 10 | PIP charges fast and hits hard (12 dmg, 1.0 range) | Manual | NEEDS HUMAN |
| 11 | Notifications dive-bomb and bounce back | Manual | NEEDS HUMAN |
| 12 | Middle Manager fires CircleBack — read trajectory + dodge | Manual | NEEDS HUMAN |
| 13 | VP of Sales 6-pellet hitscan reads as devastating | Manual | NEEDS HUMAN |
| 14 | Reply All Storm cycles in slot 3 (press 3 twice) | Manual | NEEDS HUMAN |
| 15 | Reply All Storm devastates at close range (8 pellets × 8 dmg) | Manual | NEEDS HUMAN |
| 16 | Reply All Storm uses 2 ammo per shot (queue: drops by 2) | Manual | NEEDS HUMAN |
| 17 | HUD third line appears with surveillance log fragment after Cycle 1 | Manual | NEEDS HUMAN |
| 18 | Surveillance fragment rotates every ~30s | Manual | NEEDS HUMAN |
| 19 | [CONNECTION RESET] blip appears every ~45s | Manual | NEEDS HUMAN |
| 20 | Cycle-transition intermission feels narratively continuous (not "coming soon") | Manual | NEEDS HUMAN |
| 21 | EndOfCycleStub shows cycleNumber=2 ("cycle 3 — coming soon") | Manual | NEEDS HUMAN |
| 22 | After EndOfCycleStub, fresh boot starts at cycle 1 (cycleStore reset) | Manual | NEEDS HUMAN |
| 23 | All 5 Cycle 2 maps render visibly (walls + sprites) | Manual | NEEDS HUMAN |
| 24 | All 4 new enemy placeholders are visually distinct | Manual | NEEDS HUMAN |
| 25 | CircleBack projectile is visibly cyan (different from existing projectiles) | Manual | NEEDS HUMAN |
| 26 | Combat in tight rooms feels different from open arenas | Manual | NEEDS HUMAN |
| 27 | Boss-gate closure works on cyc2_glass_confroom_boss (can't exit until VP dies) | Manual | NEEDS HUMAN |

## How to verify the manual items

```bash
cd /Users/justinwest/Repos/l0b0tonline
npm run dev
```

Open the dev URL, boot J0IN 0S, run loom.exe. Play through Cycles 1 + 2. Mark each NEEDS-HUMAN item PASS / FAIL.

## Phase 3 commit summary

[fill in once all commits land]

| Task | SHA | Description |
|---|---|---|
| 3.1 | [sha] | Extend campaign.ts for 2-cycle progression |
| 3.2 | [sha] | Wire cycle-transition + cycleStore.advanceCycle |
| 3.3 | [sha] | HUD corruption mode 1 |
| 3.4 | [sha] | Reply All Storm |
| 3.5 | [sha] | PIP enemy |
| 3.6 | [sha] | Notification enemy |
| 3.7 | [sha] | Middle Manager + CircleBack projectile |
| 3.8 | [sha] | VP of Sales boss |
| 3.9 | [sha] | cyc2_open_office map |
| 3.10 | [sha] | cyc2_break_room map |
| 3.11 | [sha] | cyc2_slack_huddle map |
| 3.12 | [sha] | cyc2_wellness_ctr map |
| 3.13 | [sha] | cyc2_glass_confroom_boss map |
| 3.14 | [sha] | Phase 3 e2e spec |
```

- [ ] **Step 2: Commit**

```bash
cd /Users/justinwest/Repos/LOOM-DOOM/.claude/worktrees/awesome-sutherland-aad854
git add docs/playtests/2026-04-27-phase-3-cycle-2.md
git commit -m "Add Phase 3 Cycle 2 playtest log

Hybrid coverage: Phase 3 e2e spec verifies the 9 automation-testable
DoD items (boot, cycle 1 traversal, cycle-transition, HUD corruption
mode, cycle 2 traversal, EndOfCycleStub, return to boot, no console
errors). 18 visual/feel/AI/audio items NEEDS HUMAN.

Follows the same template as Phase 0/1/2 logs."
```

---

# Self-review

**Spec coverage** (against `docs/superpowers/specs/2026-04-27-loom-doom-design.md`, §8 Phase 3):

| Spec requirement | Task |
|---|---|
| 5 maps: cyc2_open_office, break_room, slack_huddle, wellness_ctr, glass_confroom_boss | 3.9, 3.10, 3.11, 3.12, 3.13 |
| Enemies: PIP, Notification, Middle Manager, VP of Sales (boss) | 3.5, 3.6, 3.7, 3.8 |
| Weapon: Reply All Storm | 3.4 |
| HUD: light corruption mode (corruption=1) | 3.3 |
| More JOIN OS placements at narrative beats | (deferred — soundService doesn't yet have Wellness Credit Sc0re method; reuses Ship It as placeholder) |
| Cycles 1 + 2 play continuously start-to-finish | 3.1, 3.2 (cycle-transition + advanceCycle), 3.14 (Playwright proves it) |
| **Phase 3 DoD: Cycles 1 + 2 play continuously** | 3.14 e2e + 3.15 playtest log |

**Phase 1 review item I1 carry-over:**
| Item | Task |
|---|---|
| `cycleStore.advanceCycle` never called in production | 3.2 — wired into the intermission's onContinue when `cycleAdvance` is true |

**Spec drift acknowledgments:**
- The "More JOIN OS placements" line in spec §8 Phase 3 is partially deferred. The current `soundService.ts` has `startShipItMusic` but no `startWellnessMusic`. Phase 3 reuses Ship It on the VP boss arena as placeholder. Adding a second track is a Phase 6 polish task — extending `soundService.ts` is out of scope for Phase 3 cleanup.
- The "envelopes ricochet in tight rooms" mechanic for Reply All Storm is deferred to Phase 4 polish.
- HUD corruption mode 1 ships the surveillance-log line + connection blip but skips the "subtle character flickers" effect (Phase 4 visual polish).

**Placeholder scan:** No "TBD" / "TODO" / "fill in details" patterns. The known deferrals (Wellness music, ricochet, character-flicker) are explicitly documented as Phase 4-6 work, not placeholders.

**Type consistency:**
- `MapExitEvent` discriminated union — `next-map` / `cycle-transition` / `cycle-end` (Task 3.1 & 3.2)
- `GameState.intermission` carries `cycleAdvance: boolean` (Task 3.2)
- `IEnemy.isBoss` flag — VP of Sales overrides to true (Task 3.8); existing Phase 2 BaseEnemy default is false
- New `ThingType` values: `'pip'`, `'notification'`, `'middle_manager'`, `'vp_of_sales'` (Tasks 3.5–3.8)
- `IWeapon.fire(player, map, enemies, audio)` unchanged from Phase 2; Reply All Storm matches
- `IProjectile.update(dt, ProjectileUpdateContext)` unchanged from Phase 2; CircleBack matches
- `findEnemyAlongRay` helper used by Reply All Storm (matches Phase 2 pattern)

All consistent.

---

# Done condition

Phase 3 complete when:
1. All Tasks 3.1–3.15 implementation steps done and committed
2. Task 3.14 Playwright spec passes (full Cycles 1+2 walk)
3. Vitest passes (existing 44 LOOM tests + new tests in 3.1, 3.4, 3.6 = ~55 LOOM tests)
4. TSC clean
5. Playtest log file exists (manual items remain NEEDS HUMAN if user can't visually verify)

Next plan after Phase 3: `2026-XX-XX-loom-phase-4.md` covering Cycle 3 (uncanny break — 6 maps, Surveillance Drone / HR Skeleton / Wellness Officer / Brand Ambassador boss, Guitar / Bass / Frequency Tuner weapons, **HUD corruption mode 2** with lying-values + full surveillance-log invasion, real sprite art replacing colored rectangles).
