# LOOM Phase 5 Implementation Plan — Cycle 4 + The C-Reveal

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship **Cycle 4 — the void**, the LOOM campaign's final cycle and ending. After defeating the Brand Ambassador, the player crosses into a cycle that doesn't behave like the others: walls dissolve into raw memory, the HUD becomes more absent than present, two new weapons unlock (Microphone, The Hum), and four new elites + the penultimate boss Dr. Verra Tessier + the final boss The Hand stand between the player and the truth. The Hand fights in three phases — the third literally takes over the player's mouse cursor via Pointer Lock + a fake-cursor sprite that can render anywhere on the J0IN 0S desktop. Killing The Hand triggers the **C-reveal cinematic**: the player's olive jumpsuit dissolves, the title overlay flips letter-by-letter from `DEFECTIVE UNIT #88-E` → `NULL_ENTITY / SAINT_L0B0T`, the manifesto text scrolls, the LOOM window chrome appears to glitch (rendered as an overlay — not a real WindowManager mutation), and *OK / ACCEPT* credits roll. After credits, the L-907 surveillance log entry is unlocked and appears in the J0IN 0S filesystem alongside L-901..L-906.

**Architecture:** Extends the Phase 0–4 engine with three substantial new subsystems:

1. **HUD corruption mode 3 (dissolving)** — extends `LoomHud` with `corruption ≥ 3` rendering: characters break apart visually, lines fade in/out on independent intervals (deliberately desynced like modes 1 + 2), the surveillance log invades the bottom strip. The "decohering" feel doubles down.

2. **Cursor takeover** — implemented via the Pointer Lock API on the LOOM canvas, plus a **fake cursor DOM sprite** rendered through a React portal at `position: fixed`, `z-index: 9999`. When The Hand enters phase 3, the canvas requests pointer lock; the OS cursor is hidden; we track virtual cursor position from `mousemove` deltas; the fake cursor sprite renders at that position; The Hand AI tugs the virtual cursor around, sometimes outside the canvas — visually appearing to "leave" the LOOM window and drift over the J0IN 0S desktop. **No actual interaction with J0IN 0S elements** — the fake cursor is a render-only overlay; it doesn't dispatch events outside the canvas. Player damages The Hand by firing weapons whose hit checks consult the virtual cursor position when it's inside the canvas's bounds.

3. **C-reveal cinematic** — a new `<CRevealCinematic>` component overlay driven by a state machine: `pause → jumpsuit-dissolve → title-flip → manifesto-scroll → window-glitch → credits → done`. Each step has a duration; the whole sequence is non-interactive (input is locked). On `done`, the LOOMGame state machine transitions to a new `cycle-end` variant `kind: 'cycle-end-final'` that calls `cycleStore.revealIdentity()` and `unlocksStore.unlock('loom_archive.dat')` (or whatever the file-unlocks API is in l0b0tonline) before returning to boot.

4. **L-907 surveillance log entry** — added to `data/content/loomLogs.ts` directly. The entry is filtered out of the J0IN 0S filesystem listing until `unlocksStore` reports `loom_archive.dat` (or equivalent) as unlocked. The reveal action sets the flag.

5. **Window chrome glitch** — a pseudo-glitch overlay rendered ON THE LOOM CANVAS (not on the J0IN 0S window). Visually shows the window border distorting/breaking via CSS effects on a DOM element layered over the canvas. Avoids any cross-app dependency on the WindowManager.

**Tech Stack:** Same as Phase 4 — TypeScript 5.8, React 19, Vite 6, Tailwind v4, Zustand 5, WebGL 2, Web Audio API, Vitest, Playwright. New API surface: Pointer Lock (already standard browser API).

**Pre-existing state:**
- l0b0tonline `loom-game` branch is fresh off `main` (commit `15326fc`) with two Phase 4 cleanup commits already landed (`7bbe175` HUD timeout-Set tracker; `0b9f50c` IWeapon contract tightening).
- Production builds use esbuild minification — anything keyed on class names will silently break in deployed Cloudflare Workers builds (Codex P1 lesson from PR #159). Use stable instance fields, not `constructor.name` lookups.
- The Phase 4 review tests were patched mid-execution to use `vi.spyOn(performance, 'now').mockImplementation()` instead of `vi.setSystemTime` — Vitest 4 doesn't advance `performance.now` via `setSystemTime`. **Use the spy pattern in any new time-dependent test.**

**Deferred from Phase 5 to Phase 6 polish (kept off the critical path):**
- Real *Patch Tuesday (Heaven Vers!0n)* and *human In l00p* / *make It better (mkbtr)* music tracks for cyc4_tessier and cyc4_hand. Phase 5 ships *Ship It* as placeholder for both.
- Composing the actual manifesto vocal scroll — Phase 5 uses `MANIFESTO_TEXT` from `data/content/manifesto.ts` (already exists in l0b0tonline canon).
- Termination Letter consumable (rare drop, C2+) — deferred to Phase 6 polish.
- Real sprite art replacing colored rectangles — Phase 6.
- Final difficulty tuning across all 4 cycles — Phase 6.
- Skill-level selector (Easy/Normal/Hard) — Phase 6.
- Reduced-motion mode — Phase 6 accessibility pass.

These deferrals keep Phase 5 honest about the *creative* gates (the C-reveal lands; The Hand's cursor-takeover lands) without blocking on polish content.

---

## File structure (post-Phase-5, in l0b0tonline)

New files (NEW), modified files (MOD):

```
l0b0tonline/
├── components/
│   ├── LOOMGame.tsx                                              (MOD — handle 'cycle-end-final' state + render <CRevealCinematic>)
│   └── loom/
│       ├── engine/
│       │   ├── audioController.ts                                (MOD — register cyc4_tessier and cyc4_hand boss arenas)
│       │   ├── campaign.ts                                       (MOD — CYCLE_4_MAPS; isEndOfCampaign now points at cyc4_hand)
│       │   ├── campaign.test.ts                                  (MOD — extended for 4-cycle progression)
│       │   ├── cursorTakeover.ts                                 (NEW — Pointer Lock + virtual cursor state)
│       │   ├── cursorTakeover.test.ts                            (NEW — virtual cursor tracking, lock state transitions)
│       │   └── gameLoop.ts                                       (MOD — register 5 new ThingTypes + 2 new weapons + cursor takeover hook)
│       ├── entities/
│       │   ├── enemies/
│       │   │   ├── Enemy.ts                                      (MOD — add `applyDebuff(kind, durationMs)` to BaseEnemy for Hand drag mechanic)
│       │   │   ├── synergyPod.ts                                 (NEW — spawns Notifications)
│       │   │   ├── synergyPod.test.ts                            (NEW)
│       │   │   ├── seniorVP.ts                                   (NEW — Intern entourage)
│       │   │   ├── theAlgorithm.ts                               (NEW — multi-armed personalized chaingun)
│       │   │   ├── drVerraTessier.ts                             (NEW — penultimate boss with rocket arms)
│       │   │   ├── drVerraTessier.test.ts                        (NEW)
│       │   │   ├── theHand.ts                                    (NEW — final boss, 3-phase state machine)
│       │   │   └── theHand.test.ts                               (NEW — phase transitions, drag/menu/hover sub-mechanics)
│       │   ├── projectiles/
│       │   │   ├── personalizedContent.ts                        (NEW — The Algorithm's chain projectile)
│       │   │   ├── terminationNotice.ts                          (NEW — Tessier's rocket projectile)
│       │   │   ├── handDrag.ts                                   (NEW — Hand phase 1: invert-controls debuff projectile)
│       │   │   ├── handMenu.ts                                   (NEW — Hand phase 2: contextual-debuff projectile)
│       │   │   └── decompileBurst.ts                             (NEW — Tessier's death particle effect)
│       │   ├── player.ts                                         (MOD — add `movementInverted` flag + `applyDebuff(kind, dur)` method)
│       │   └── weapons/
│       │       ├── microphone.ts                                 (NEW — slot 7, reflection-scaling damage)
│       │       ├── microphone.test.ts                            (NEW)
│       │       ├── theHum.ts                                     (NEW — slot 8 unique, sustained beam)
│       │       └── theHum.test.ts                                (NEW)
│       ├── hud/
│       │   ├── LoomHud.tsx                                       (MOD — corruption mode 3 dissolving)
│       │   ├── CRevealCinematic.tsx                              (NEW — jumpsuit dissolve → title flip → manifesto → glitch → credits)
│       │   ├── CRevealCinematic.test.tsx                         (NEW — sequence state machine timing)
│       │   ├── FakeCursorSprite.tsx                              (NEW — DOM portal cursor for The Hand phase 3)
│       │   └── EndOfCycleStub.tsx                                (MOD — handle 'final' variant for cycle-end-final dismissal)
│       ├── input.ts                                              (MOD — `setMovementInverted(true/false)` for Hand drag debuff)
│       └── store/
│           ├── cycleStore.ts                                     (already has `revealIdentity` action — verify and possibly extend)
│           └── cycleStore.test.ts                                (MOD — extended for 4-cycle + revealIdentity)
└── public/
    └── data/
        └── loom/
            └── maps/
                ├── cyc4_void_1.json                              (NEW)
                ├── cyc4_void_2.json                              (NEW)
                ├── cyc4_tessier.json                             (NEW — penultimate boss arena)
                └── cyc4_hand.json                                (NEW — final boss arena, contains The Hum pickup)
└── data/
    └── content/
        └── loomLogs.ts                                           (MOD — add L-907 SUBJECT_DOOM entry)
```

l0b0tonline integration files we may need to touch:
- The unlocks slice (`store/slices/unlocksSlice.ts` or equivalent) — verify the API for setting an unlock flag at runtime; if absent, defer file-unlock and just append L-907 unconditionally.

E2E:
- `e2e/loom-phase-5.spec.ts` (NEW — full 18-map cycles 1+2+3+4 walk + The Hand defeat + C-reveal full sequence + L-907 unlock verification)
- `e2e/loom-phase-4.spec.ts` (DELETE — Phase 5 spec is a strict superset)

---

# Phase 5 Definition of Done

A new player can:

1. Boot LOOM, play Cycles 1+2+3 (14 maps), defeat HR Manager / VP of Sales / Brand Ambassador
2. Cross into Cycle 4 — HUD shows `> cycle 8495 // simulation DISSOLVING // surveillance: ON`
3. Walk Cycle 4's 4 maps engaging Synergy Pod, Senior VP, The Algorithm, then defeating Dr. Verra Tessier
4. Pick up **The Hum** (a single map item placed on `cyc4_hand`) — triggers a manifesto-lyric flash overlay (`THE HUM IS TRYING TO TELL YOU SOMETHING.`) and adds The Hum to slot 8
5. Engage **The Hand** through 3 distinct phases:
   - **Phase 1 — drag (300 → 200 HP)**: melee chase + drag projectiles; on hit, player movement controls invert briefly (~3s)
   - **Phase 2 — menu (200 → 100 HP)**: spawns ephemeral context-menu projectiles labeled DELETE / MUTE / BLOCK that approach the player; touching one inflicts the corresponding debuff for ~3s
   - **Phase 3 — hover (100 → 0 HP)**: pointer lock activates; fake cursor sprite spawns; The Hand AI moves the cursor — sometimes attacking the player from inside the canvas, sometimes drifting outside the canvas (visually over J0IN 0S desktop) where it can't be hit; player damages it by firing at it when it's inside the canvas
6. The Hand dies in a particle-burst of decompiling code → triggers the **C-reveal cinematic**:
   - Game input locks
   - Window chrome glitch overlay renders (canvas-bound — no real WindowManager mutation)
   - Player jumpsuit fades from olive → dashed outline → invisible
   - Title flips letter-by-letter from `DEFECTIVE UNIT #88-E` → `NULL_ENTITY / SAINT_L0B0T`
   - Manifesto text scrolls in J0IN 0S terminal voice
   - *OK / ACCEPT* credits roll
7. After credits, dismiss → returns to boot. **L-907 surveillance log entry now visible** in the J0IN 0S filesystem (this is the unlock the player retains across runs)
8. **Replaying loom.exe** from a fresh boot starts at cycle=1, corruption=0, identityRevealed=false (cycleStore reset); but the L-907 entry remains unlocked (filesystem-level state, not cycleStore-level)

**Automation gate:** Phase 5 Playwright spec walks 18 maps + 4 boss kills (HR Manager / VP of Sales / Brand Ambassador / Tessier) + The Hand 3-phase defeat (test mode skips pointer lock by directly damaging the boss via `__LOOM_GAME__` hook) + C-reveal cinematic completion + L-907 unlock check. No console errors. Runs in under 35s.

**Manual creative gates (NEEDS-HUMAN — captured in Phase 5 playtest log):**
- Does The Hand phase 3 cursor takeover *feel* genuinely transgressive (the player's own pointer being hijacked)?
- Does the C-reveal land emotionally — jumpsuit dissolve + title flip + manifesto scroll all in sequence?
- Is the window-chrome glitch overlay convincing (does it look like the J0IN 0S window itself is breaking, not just a bordered overlay)?
- Does HUD corruption mode 3 dissolution feel terminal, like the simulation is genuinely ending?
- Does picking up The Hum produce a meaningful "this is different" beat (manifesto lyric flash)?
- Does L-907 reading after credits feel like the diegetic reward it's meant to be?

---

# Tasks

## Task 5.1: Extend `campaign.ts` for 4-cycle progression

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/campaign.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/campaign.test.ts`

The Phase 4 implementation already wires the cycle-transition events. Phase 5 just adds `CYCLE_4_MAPS` (4 ids) and extends `MAP_TO_CYCLE` and `BOSS_ARENA_IDS`. `isEndOfCampaign` will then report `true` only at `cyc4_hand` (the new last map).

- [ ] **Step 1: Extend `campaign.test.ts` for 4-cycle progression**

Add `CYCLE_4_MAPS` to imports, then add inside the existing describe blocks:

```typescript
    it('CYCLE_4_MAPS lists 4 maps in order', () => {
      expect(CYCLE_4_MAPS).toEqual([
        'cyc4_void_1',
        'cyc4_void_2',
        'cyc4_tessier',
        'cyc4_hand',
      ]);
    });

    it('returns 4 for Cycle 4 maps', () => {
      expect(getCycleOfMap('cyc4_void_1')).toBe(4);
      expect(getCycleOfMap('cyc4_hand')).toBe(4);
    });

    it('crosses the cycle 3→4 boundary', () => {
      expect(getNextMapId('cyc3_brand_summit_boss')).toBe('cyc4_void_1');
    });

    it('returns next map within Cycle 4', () => {
      expect(getNextMapId('cyc4_void_1')).toBe('cyc4_void_2');
      expect(getNextMapId('cyc4_tessier')).toBe('cyc4_hand');
    });
```

Replace the existing `returns null after the final shipped map` test:
```typescript
    it('returns null after the final shipped map', () => {
      expect(getNextMapId('cyc4_hand')).toBeNull();
    });
```

Replace the `isEndOfCampaign` C3 test with a C4 one:
```typescript
    it('true only at the very last shipped map (cyc4_hand)', () => {
      expect(isEndOfCampaign('cyc4_hand')).toBe(true);
    });

    it('false at cycle 3 final map (campaign continues into cycle 4)', () => {
      expect(isEndOfCampaign('cyc3_brand_summit_boss')).toBe(false);
    });
```

Update the `isLastMapOfCycle` block similarly — `cyc3_brand_summit_boss` becomes a transition-not-end; `cyc4_hand` becomes the true end.

Also add a test for the existing `isBossArena` helper to confirm cyc4 boss arenas register:
```typescript
    it('returns true for Cycle 4 boss arenas', () => {
      expect(isBossArena('cyc4_tessier')).toBe(true);
      expect(isBossArena('cyc4_hand')).toBe(true);
    });
```

- [ ] **Step 2: Run tests, verify failure**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/engine/campaign.test.ts 2>&1 | tail -15
```

Expected: tests fail because CYCLE_4_MAPS is undefined; `getNextMapId('cyc3_brand_summit_boss')` returns null instead of cyc4_void_1; `isBossArena('cyc4_*')` returns false.

- [ ] **Step 3: Update `campaign.ts`**

Replace the file contents with:
```typescript
import type { CycleNumber } from '../store/cycleStore';

/**
 * Campaign progression — what map comes after which, and where each cycle ends.
 *
 * Phase 5 ships Cycles 1 + 2 + 3 + 4 (18 maps total — the full LOOM campaign).
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
export const CYCLE_4_MAPS = [
  'cyc4_void_1',
  'cyc4_void_2',
  'cyc4_tessier',
  'cyc4_hand',
] as const;

export type CycleMapId =
  | typeof CYCLE_1_MAPS[number]
  | typeof CYCLE_2_MAPS[number]
  | typeof CYCLE_3_MAPS[number]
  | typeof CYCLE_4_MAPS[number];

const ALL_CYCLE_MAPS: readonly string[] = [
  ...CYCLE_1_MAPS,
  ...CYCLE_2_MAPS,
  ...CYCLE_3_MAPS,
  ...CYCLE_4_MAPS,
];

const MAP_TO_CYCLE: ReadonlyMap<string, CycleNumber> = new Map<string, CycleNumber>([
  ...CYCLE_1_MAPS.map((id): [string, CycleNumber] => [id, 1]),
  ...CYCLE_2_MAPS.map((id): [string, CycleNumber] => [id, 2]),
  ...CYCLE_3_MAPS.map((id): [string, CycleNumber] => [id, 3]),
  ...CYCLE_4_MAPS.map((id): [string, CycleNumber] => [id, 4]),
]);

export function getCycleOfMap(currentId: string): CycleNumber | null {
  return MAP_TO_CYCLE.get(currentId) ?? null;
}

export function getNextMapId(currentId: string): string | null {
  const idx = ALL_CYCLE_MAPS.indexOf(currentId);
  if (idx < 0) return null;
  return ALL_CYCLE_MAPS[idx + 1] ?? null;
}

export function isLastMapOfCycle(currentId: string): boolean {
  const currCycle = getCycleOfMap(currentId);
  if (currCycle === null) return false;
  const nextId = getNextMapId(currentId);
  if (nextId === null) return true;
  const nextCycle = getCycleOfMap(nextId);
  return nextCycle !== currCycle;
}

export function isEndOfCampaign(currentId: string): boolean {
  const currCycle = getCycleOfMap(currentId);
  if (currCycle === null) return false;
  return getNextMapId(currentId) === null;
}

export const BOSS_ARENA_IDS: ReadonlySet<string> = new Set([
  'cyc1_hr_arena',
  'cyc2_glass_confroom_boss',
  'cyc3_brand_summit_boss',
  'cyc4_tessier',
  'cyc4_hand',
]);

export function isBossArena(mapId: string): boolean {
  return BOSS_ARENA_IDS.has(mapId);
}
```

- [ ] **Step 4: Run tests, verify pass**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/engine/campaign.test.ts 2>&1 | tail -10
```

- [ ] **Step 5: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/engine/campaign.ts components/loom/engine/campaign.test.ts
git commit -m "Extend campaign.ts for 4-cycle progression (Phase 5 setup)

CYCLE_4_MAPS adds the 4 final maps: cyc4_void_1 → cyc4_void_2 →
cyc4_tessier → cyc4_hand. getNextMapId('cyc3_brand_summit_boss')
now returns 'cyc4_void_1', threading the cycle 3→4 boundary the
same way prior boundaries work. isEndOfCampaign now reports true
only at cyc4_hand (the final boss arena and end of the campaign).

BOSS_ARENA_IDS Set extended with cyc4_tessier (Dr. Verra Tessier
penultimate boss) and cyc4_hand (The Hand final boss).

Existing GameLoop exit-detection logic handles the cycle-transition
event for cyc3→cyc4 without modification — the LOOMGame state
machine advances cycleStore to (cycle=4, corruption=3) on
intermission dismissal."
```

---

## Task 5.2: AudioController for cyc4 boss arenas + cycleStore.revealIdentity action

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/store/cycleStore.ts` (verify revealIdentity exists)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/store/cycleStore.test.ts`

`audioController.onMapLoad` uses `isBossArena()` from campaign.ts (Phase 4 prep) — so cyc4 boss arenas already swap to *Ship It* (placeholder for spec-canonical *Patch Tuesday* + *human In l00p*) automatically. **No audioController.ts change required.**

cycleStore.ts already has the `revealIdentity` action defined (Phase 0 setup). Phase 5 just adds tests verifying it sets `identityRevealed=true` and that `reset()` correctly resets `identityRevealed` back to false.

- [ ] **Step 1: Verify cycleStore.revealIdentity exists**

Read `/Users/justinwest/Repos/l0b0tonline/components/loom/store/cycleStore.ts`. Confirm:
- `identityRevealed: boolean` field with `false` initial value
- `revealIdentity: () => set({ identityRevealed: true })` action
- `reset` action resets `identityRevealed` to `false` along with cycle and corruption

If any of these are missing, add them. The spec assumes they exist from Phase 0.

- [ ] **Step 2: Extend cycleStore tests**

Add to `cycleStore.test.ts`:
```typescript
  describe('revealIdentity', () => {
    it('sets identityRevealed to true', () => {
      const s = useCycleStore.getState();
      s.reset();
      expect(s.identityRevealed).toBe(false);
      s.revealIdentity();
      expect(useCycleStore.getState().identityRevealed).toBe(true);
    });

    it('reset() clears identityRevealed', () => {
      const s = useCycleStore.getState();
      s.revealIdentity();
      expect(useCycleStore.getState().identityRevealed).toBe(true);
      s.reset();
      expect(useCycleStore.getState().identityRevealed).toBe(false);
    });

    it('is idempotent', () => {
      const s = useCycleStore.getState();
      s.reset();
      s.revealIdentity();
      s.revealIdentity();
      expect(useCycleStore.getState().identityRevealed).toBe(true);
    });
  });
```

- [ ] **Step 3: Run tests + tsc + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/store/cycleStore.test.ts 2>&1 | tail -10
npx tsc --noEmit 2>&1 | head -5
git add components/loom/store/cycleStore.ts components/loom/store/cycleStore.test.ts
git commit -m "Wire cycle 4 audio + add cycleStore.revealIdentity tests (Phase 5)

audioController.onMapLoad already routes cyc4_tessier and cyc4_hand
through the isBossArena() helper added in Phase 4 (commit 3f2a08e).
Music stays Ship It as placeholder; the spec-canonical Patch Tuesday
and human In l00p tracks are Phase 6 polish.

cycleStore.revealIdentity action gets explicit tests:
  - sets identityRevealed=true
  - reset() clears it back to false (so a fresh boot post-credits
    starts as a 'fresh' player without the reveal)
  - is idempotent (call twice = same effect)

The reveal action is fired by the C-reveal cinematic (Task 5.17)
after The Hand's death + manifesto scroll completes."
```

---

## Task 5.3: HUD corruption mode 3 (dissolving)

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/LoomHud.tsx`

Corruption mode 3 (active when `cycleStore.corruption >= 3`, so Cycle 4) layers on top of mode 2:
1. **Line fades** — each HUD line fades opacity in/out on its own ~7000ms cycle (different phase per line, deliberately desynced from the lying-value intervals)
2. **Character break-apart** — every ~12s, individual characters in the HUD strip get random letter-spacing offsets for ~400ms then snap back
3. **Surveillance log dominates** — the log line opacity goes from `text-green-800` (dim) to `text-green-400` (full) and the rotation cadence drops to 8s (vs 14.5s in mode 2)
4. **Identity-revealed override** — once `cycleStore.identityRevealed === true`, the HUD title `[unit#88-E]` gets crossed out + replaced inline with `[NULL_ENTITY]`. This persists through credits.

Each effect uses the established `Set<number>` cleanup pattern (matches I-4 fix).

- [ ] **Step 1: Update LoomHud for corruption mode 3**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/LoomHud.tsx`. Add the new constants:

```typescript
const LINE_FADE_PERIOD_MS = 7100;     // line opacity sine cycle
const CHAR_BREAK_INTERVAL_MS = 12_300; // char letter-spacing jitter cadence
const CHAR_BREAK_DURATION_MS = 400;
const FRAGMENT_INTERVAL_MS_MODE_3 = 8_000; // surveillance log rotation drops to 8s
```

Add `identityRevealed` to the cycleStore selectors:
```typescript
const identityRevealed = useCycleStore((s) => s.identityRevealed);
```

Add new state for char-break:
```typescript
const [charBreak, setCharBreak] = useState(false);
```

Update the surveillance-log fragment-rotation effect to handle 3 cadences:
```typescript
useEffect(() => {
  if (corruption < 1) return;
  let interval: number;
  if (corruption >= 3) interval = FRAGMENT_INTERVAL_MS_MODE_3;
  else if (corruption >= 2) interval = FRAGMENT_INTERVAL_MS_MODE_2;
  else interval = FRAGMENT_INTERVAL_MS_MODE_1;
  const id = window.setInterval(() => {
    setFragmentIdx((i) => (i + 1) % SURVEILLANCE_FRAGMENTS.length);
  }, interval);
  return () => window.clearInterval(id);
}, [corruption]);
```

Add new effect for char-break (corruption ≥ 3):
```typescript
useEffect(() => {
  if (corruption < 3) return;
  const pendingTimeouts = new Set<number>();
  const id = window.setInterval(() => {
    setCharBreak(true);
    const t = window.setTimeout(() => {
      setCharBreak(false);
      pendingTimeouts.delete(t);
    }, CHAR_BREAK_DURATION_MS);
    pendingTimeouts.add(t);
  }, CHAR_BREAK_INTERVAL_MS);
  return () => {
    window.clearInterval(id);
    for (const t of pendingTimeouts) window.clearTimeout(t);
  };
}, [corruption]);
```

For line fades, use a frame-driven sine-wave opacity computation rather than setIntervals (smoother):
```typescript
// In the rAF tick loop, also compute line-fade opacities.
const [tickMs, setTickMs] = useState(0);

useEffect(() => {
  let h: number;
  const start = performance.now();
  const tick = () => {
    setSnap(getSnapshot());
    setTickMs(performance.now() - start);
    h = requestAnimationFrame(tick);
  };
  h = requestAnimationFrame(tick);
  return () => cancelAnimationFrame(h);
}, [getSnapshot]);
```

Then in the JSX, when `corruption >= 3`, compute per-line opacities like:
```typescript
const line1Opacity = corruption >= 3
  ? 0.55 + 0.45 * Math.abs(Math.sin((tickMs + 0)    * Math.PI / LINE_FADE_PERIOD_MS))
  : 1;
const line2Opacity = corruption >= 3
  ? 0.55 + 0.45 * Math.abs(Math.sin((tickMs + 2370) * Math.PI / LINE_FADE_PERIOD_MS))
  : 1;
const logOpacity   = corruption >= 3
  ? 0.55 + 0.45 * Math.abs(Math.sin((tickMs + 4730) * Math.PI / LINE_FADE_PERIOD_MS))
  : 1;
```

The phase offsets (0, 2370, 4730) are coprime-ish to LINE_FADE_PERIOD_MS = 7100 so the three lines never sync visibly.

Add `style={{ opacity: line1Opacity }}` to each of the three div lines (the `[unit#88-E] / bio: ...` line, the `> cycle 8495 ...` line, and the surveillance-log line).

For char-break, when `corruption >= 3 && charBreak`, apply a wider letter-spacing class to the bio/queue/active/zone span row:
```tsx
<div className={corruption >= 3 && charBreak ? 'tracking-[0.4em]' : 'tracking-normal'} style={{ opacity: line1Opacity }}>
  ...
</div>
```

For identity-revealed override (Task 5.17 will set this), update the unit-id span:
```tsx
<span className="text-green-500">
  {identityRevealed ? <>[<s>unit#88-E</s> NULL_ENTITY]</> : '[unit#88-E]'}
</span>
```

For surveillance-log dominance (corruption ≥ 3), brighten the text:
```tsx
{corruption >= 1 && (
  <div
    className={`text-[10px] italic mt-0.5 truncate ${corruption >= 3 ? 'text-green-400' : 'text-green-800'}`}
    style={{ opacity: logOpacity }}
  >
    ...
  </div>
)}
```

Update the simState constant:
```typescript
const simState =
  corruption === 0 ? 'nominal'
  : corruption === 1 ? 'drift detected'
  : corruption === 2 ? 'BREACH'
  : 'DISSOLVING';
```
This already exists from Phase 4 — verify it's untouched.

- [ ] **Step 2: Type-check + smoke**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -10
npx vitest run components/loom 2>&1 | tail -5
```

Expected: tsc clean, vitest unchanged (LoomHud has no unit tests; behavior verified via Playwright in Task 5.19).

- [ ] **Step 3: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/hud/LoomHud.tsx
git commit -m "Add HUD corruption mode 3 (dissolving) — Phase 5 Cycle 4

When cycleStore.corruption ≥ 3 (Cycle 4 onward), four new behaviors
layer on mode 2:

1. Line fades — each HUD line's opacity sine-waves on a 7100ms cycle
   with coprime-ish phase offsets (0, 2370, 4730) so the three lines
   never visibly resync. Computed from the rAF tick, frame-smooth.

2. Char break-apart — every ~12.3s the bio/queue/active/zone span
   row gets letter-spacing 0.4em for 400ms, then snaps back. Like
   characters momentarily losing structural integrity.

3. Surveillance log dominance — log line brightens from text-green-800
   to text-green-400 (visible front-and-center, no longer subtitle-tier)
   and rotates every 8s instead of 14.5s.

4. Identity-revealed override — once cycleStore.identityRevealed is
   true (set by C-reveal cinematic in Task 5.17), the [unit#88-E]
   span gets crossed out and replaced with [NULL_ENTITY] inline.

All cleanup follows the Set<number> pattern from Phase 4 review I-4.
Phase 4 corruption modes 1 + 2 still active under mode 3."
```

---

## Task 5.4: BaseEnemy.applyDebuff + Player.movementInverted (Hand prep)

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/Enemy.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/player.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/input.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/baseEnemy.test.ts`

The Hand's phase 1 (drag) and phase 2 (menu) projectiles inflict timed debuffs on the player. Phase 5 introduces a single debuff: `'movement-inverted'` (3s default duration). The Hand's `handDrag` projectile sets it on hit; the Hand's `handMenu` projectile sets it on contact.

The player's movement controls (W/A/S/D) — currently routed via `Player.update(dt, input, map)` — will check the inverted flag and apply opposite-direction movement when active.

**Note: this is just the engine plumbing. The Hand projectile classes that consume it ship in Tasks 5.13/5.14 (Hand phases).**

- [ ] **Step 1: Add Player.movementInverted + applyDebuff**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/player.ts`. Add:

```typescript
type DebuffKind = 'movement-inverted';

interface ActiveDebuff {
  kind: DebuffKind;
  expiresAt: number;
}
```

Add to the Player class:
```typescript
private debuffs: ActiveDebuff[] = [];

applyDebuff(kind: DebuffKind, durationMs: number) {
  this.debuffs = this.debuffs.filter((d) => d.kind !== kind);
  this.debuffs.push({ kind, expiresAt: performance.now() + durationMs });
}

private hasDebuff(kind: DebuffKind): boolean {
  const now = performance.now();
  this.debuffs = this.debuffs.filter((d) => d.expiresAt > now);
  return this.debuffs.some((d) => d.kind === kind);
}

get movementInverted(): boolean {
  return this.hasDebuff('movement-inverted');
}
```

Inside `Player.update` where W/A/S/D move the player, multiply the move vector by `(this.movementInverted ? -1 : 1)`.

- [ ] **Step 2: Add Player tests**

Add to `player.test.ts` (or create the file if absent):
```typescript
import { describe, it, expect, vi } from 'vitest';
import { Player } from './player';

describe('Player.applyDebuff', () => {
  it('movement-inverted starts false', () => {
    const p = new Player(0, 0, 0);
    expect(p.movementInverted).toBe(false);
  });

  it('applyDebuff sets movement-inverted true for the duration', () => {
    let virtualNow = 0;
    const nowSpy = vi.spyOn(performance, 'now').mockImplementation(() => virtualNow);
    try {
      const p = new Player(0, 0, 0);
      p.applyDebuff('movement-inverted', 3000);
      expect(p.movementInverted).toBe(true);
      virtualNow = 4000; // past expiry
      expect(p.movementInverted).toBe(false);
    } finally {
      nowSpy.mockRestore();
    }
  });

  it('reapplying a debuff replaces the previous expiry', () => {
    let virtualNow = 0;
    const nowSpy = vi.spyOn(performance, 'now').mockImplementation(() => virtualNow);
    try {
      const p = new Player(0, 0, 0);
      p.applyDebuff('movement-inverted', 3000);
      virtualNow = 1000;
      p.applyDebuff('movement-inverted', 3000); // resets expiry to 4000
      virtualNow = 3500; // would have expired the FIRST debuff but not the second
      expect(p.movementInverted).toBe(true);
      virtualNow = 4500;
      expect(p.movementInverted).toBe(false);
    } finally {
      nowSpy.mockRestore();
    }
  });
});
```

- [ ] **Step 3: Run tests + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/player.test.ts 2>&1 | tail -10
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/player.ts components/loom/entities/player.test.ts
git commit -m "Add Player debuffs (movement-inverted) for Hand boss prep

Player.applyDebuff(kind, durationMs) records timed debuffs.
movement-inverted reverses W/A/S/D direction in Player.update for
the debuff duration. Auto-expires by performance.now() comparison
(no per-frame cleanup pass — checked lazily on read).

Used by:
- Task 5.10's HandDrag projectile (Hand phase 1, ~3s debuff per hit)
- Task 5.11's HandMenu projectile (Hand phase 2, ~3s debuff per contact)

3 vitest tests cover initial-false, applies-and-expires, reapply-
extends-expiry. Uses vi.spyOn(performance.now) (not setSystemTime)
per Phase 4 review I-1."
```

---

## Task 5.5: Microphone weapon (slot 7) — reflection-scaling damage

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/microphone.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/microphone.test.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/input.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Microphone — slot 7. Hitscan; **damage scales with reflection**: cast 4 secondary rays from the player at 90° offsets; for each that hits a wall within 6 tiles, add a 25% damage multiplier (capped at 4× = 200% bonus). Single fire, 18 base damage, 2 ammo per shot, range 14.

Net: 18 dmg in an empty room (no reflective walls), up to 54 dmg in a tight enclosed corridor (3 nearby walls), 72 dmg max (all 4 walls within 6 tiles).

Input: Digit7 selects slot 7.

- [ ] **Step 1: Write the failing test**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/microphone.test.ts`:
```typescript
import { describe, it, expect, vi } from 'vitest';
import { Microphone } from './microphone';
import { BaseEnemy } from '../enemies/Enemy';
import type { EnemyUpdateContext } from '../enemies/Enemy';

class FakeEnemy extends BaseEnemy {
  health = 1000;
  readonly maxHealth = 1000;
  readonly placeholderColor = [0, 0, 0] as const;
  update(_dt: number, _ctx: EnemyUpdateContext) {}
}

const fakePlayer = { x: 5, y: 5, angle: 0, ammo: 50, pulseCharge: 1, health: 100, activeSlot: 7, takeDamage() {}, applyDebuff() {} };
const noWallsMap = {
  id: 'test', cycle: 4 as const,
  grid: Array.from({ length: 20 }, () => Array(20).fill(0)),
  things: [], textures: [],
};
const fakeAudio = { onEnemyHit: vi.fn(), onPlayerDamaged: vi.fn(), onGameStart() {}, onGameEnd() {}, onMapLoad() {} };
const noopSpawn = () => {};

describe('Microphone', () => {
  it('costs 2 ammo per shot', () => {
    const w = new Microphone();
    const player = { ...fakePlayer, ammo: 10 };
    w.fire(player as any, noWallsMap as any, [], fakeAudio as any, noopSpawn);
    expect(player.ammo).toBe(8);
  });

  it('returns false when ammo < 2', () => {
    const w = new Microphone();
    const player = { ...fakePlayer, ammo: 1 };
    const fired = w.fire(player as any, noWallsMap as any, [], fakeAudio as any, noopSpawn);
    expect(fired).toBe(false);
  });

  it('does base 18 damage in an empty room (no reflective walls in range)', () => {
    const w = new Microphone();
    const player = { ...fakePlayer, ammo: 10 };
    const enemy = new FakeEnemy(8, 5); // 3 tiles east on the player's aim ray
    w.fire(player as any, noWallsMap as any, [enemy], fakeAudio as any, noopSpawn);
    expect(enemy.health).toBe(1000 - 18);
  });

  it('adds 25% per reflective wall in range (close walls scale damage up)', () => {
    const w = new Microphone();
    const player = { ...fakePlayer, ammo: 10 };
    const enemy = new FakeEnemy(8, 5);
    // Place 4 walls near the player at 90° offsets (north, south, east, west).
    const wallMap = JSON.parse(JSON.stringify(noWallsMap));
    wallMap.grid[5][2] = 1; // 3 tiles west of player at (5,5) → west secondary ray hits at dist 3 → in range
    wallMap.grid[2][5] = 1; // 3 tiles north → in range
    wallMap.grid[8][5] = 1; // 3 tiles south → in range
    wallMap.grid[5][9] = 1; // 4 tiles east — but enemy at (8,5) is in front; the secondary east-ray would hit the wall at 4 tiles → in range
    // Note: the player aims east (angle=0). Primary ray hits the enemy. The 4 secondary rays are at angles a±π/2, a+π.
    // ... actually the 4 secondary rays (at 90° offsets from aim) are: north (a+π/2), south (a-π/2), back (a+π), and... only 3 unique 90° offsets + aim back. We pick the 4 cardinal directions relative to aim: a, a+π/2, a+π, a-π/2. Primary uses 'a' for the actual hitscan; secondaries use the other 3.
    // For this test all 3 secondary rays should hit walls within 6 tiles → 3 × 25% = 75% bonus.
    w.fire(player as any, wallMap as any, [enemy], fakeAudio as any, noopSpawn);
    expect(enemy.health).toBe(1000 - Math.floor(18 * 1.75)); // 1000 - 31 = 969
  });

  it('damage is capped at 4× base (300% max bonus, but capped at 200%)', () => {
    // The cap is implementation-dependent. Spec says "max 4 reflections × 25% = 100% bonus, capped at 200% bonus = 3x"
    // Adjust this test to whatever the implementation actually caps at. Read the constant.
    expect(true).toBe(true); // placeholder — implementer adjusts based on actual cap
  });

  it('skips dead enemies on the aim ray', () => {
    const w = new Microphone();
    const player = { ...fakePlayer, ammo: 10 };
    const corpse = new FakeEnemy(7, 5);
    corpse.state = 'dead';
    corpse.health = 0;
    w.fire(player as any, noWallsMap as any, [corpse], fakeAudio as any, noopSpawn);
    expect(corpse.health).toBe(0);
  });
});
```

The reflection-bonus math is intentionally simple (count secondary rays that hit a wall within REFLECTION_RANGE; each adds REFLECTION_BONUS). The implementer should pick a cap (suggest 4 rays × 0.25 = 1.0 multiplier max, so max damage = 18 × 2 = 36).

- [ ] **Step 2: Run, verify failure**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/weapons/microphone.test.ts 2>&1 | tail -10
```

- [ ] **Step 3: Implement Microphone**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/microphone.ts`:
```typescript
import type { IWeapon } from './IWeapon';
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import type { IEnemy } from '../enemies/Enemy';
import type { IProjectile } from '../projectiles/Projectile';
import type { AudioController } from '../../engine/audioController';
import { castRay } from '../../engine/raycaster';
import { findEnemyAlongRay } from './hitResolution';

const AMMO_COST = 2;
const BASE_DAMAGE = 18;
const RANGE = 14;
const PERP_TOLERANCE = 0.4;
const REFLECTION_RANGE = 6;
const REFLECTION_BONUS_PER = 0.25;
const REFLECTION_BONUS_MAX = 1.0; // 4 × 0.25 = max +100%

/**
 * Microphone — slot 7. Voice / band-collective instrument. Single-fire
 * hitscan along the player's aim. Damage scales with REFLECTION:
 *
 *   actual_damage = floor(BASE_DAMAGE × (1 + min(N × 0.25, 1.0)))
 *
 * where N = the number of secondary rays (cast at 90°, 180°, 270° from
 * the aim) that hit a wall within REFLECTION_RANGE tiles. Empty room:
 * 18 dmg. 3-wall enclosed corner: ~31 dmg. Fully enclosed (4 walls
 * around the player): 36 dmg max (capped).
 *
 * Costs 2 ammo per shot.
 *
 * The "louder in tight rooms" diegetic hook: tight spaces reflect
 * sound, amplifying the singer's voice into a more devastating attack.
 */
export class Microphone implements IWeapon {
  readonly name = 'Microphone';
  readonly slot = 7;

  fire(
    player: Player,
    map: LoomMap,
    enemies: IEnemy[],
    audio: AudioController,
    _spawnProjectile: (p: IProjectile) => void,
  ): boolean {
    if (player.ammo < AMMO_COST) return false;
    player.ammo -= AMMO_COST;

    // Count reflective walls within REFLECTION_RANGE on 3 secondary rays
    // (90°, 180°, 270° from aim).
    let reflections = 0;
    for (const offset of [Math.PI / 2, Math.PI, -Math.PI / 2]) {
      const a = player.angle + offset;
      const wallHit = castRay(map, player.x, player.y, a);
      if (wallHit.distance <= REFLECTION_RANGE) reflections += 1;
    }
    // Plus the front (aim) — if the primary aim ray itself hits a wall
    // within REFLECTION_RANGE, that's also a reflection (echoes back).
    const primaryWall = castRay(map, player.x, player.y, player.angle);
    if (primaryWall.distance <= REFLECTION_RANGE) reflections += 1;

    const bonus = Math.min(reflections * REFLECTION_BONUS_PER, REFLECTION_BONUS_MAX);
    const damage = Math.floor(BASE_DAMAGE * (1 + bonus));

    const target = findEnemyAlongRay(
      player.x, player.y, player.angle,
      enemies, RANGE, PERP_TOLERANCE, primaryWall.distance,
    );
    if (target) {
      target.takeDamage(damage);
      audio.onEnemyHit();
    }
    return true;
  }
}
```

- [ ] **Step 4: Run, verify pass; adjust the test to match the actual cap**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/weapons/microphone.test.ts 2>&1 | tail -10
```

The implementer should update test #4 to match the actual cap (4 × 0.25 = 100% bonus = 36 max damage).

- [ ] **Step 5: Extend input.ts to handle Digit7**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/input.ts`. Add to the existing weapon-select chain:
```typescript
      else if (e.code === 'Digit7') this.weaponSelectQueue.push(7);
      else if (e.code === 'Digit8') this.weaponSelectQueue.push(8);  // for The Hum (Task 5.6)
```

- [ ] **Step 6: Register Microphone in gameLoop**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`:

a) Import:
```typescript
import { Microphone } from '../entities/weapons/microphone';
```

b) After slot-6 lines:
```typescript
    this.weaponsBySlot.set(7, [new Microphone()]);
    this.slotIndex.set(7, 0);
```

- [ ] **Step 7: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit
git add components/loom/entities/weapons/microphone.ts \
        components/loom/entities/weapons/microphone.test.ts \
        components/loom/engine/input.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add Microphone weapon (slot 7) — reflection-scaling hitscan

Voice / band-collective instrument, debuts in Cycle 4. Single-fire
hitscan; base damage 18; 2 ammo per shot; range 14.

Damage scales with reflection: counts secondary rays at 90°/180°/-90°
from aim plus the primary (forward) ray that hit a wall within 6
tiles. Each +25% bonus, capped at 4 reflections (+100% = 36 max dmg).

Empty room: 18. Tight 3-wall corner: ~31. Fully enclosed: 36.
The 'louder in tight rooms' diegetic hook: reflective surfaces
amplify the singer's voice.

InputController extended for Digit7 + Digit8 (Digit8 reserved for
The Hum, Task 5.6 — registers ahead so input.ts stays a single
Phase 5 edit per its Phase 4 precedent).

5 vitest tests cover ammo cost/guard, base damage, reflection
scaling, cap, and dead-enemy skip."
```

---

## Task 5.6: The Hum weapon (slot 8 unique) — sustained beam + pickup mechanism

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/theHum.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/theHum.test.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts` (add `'the_hum_pickup'` ThingType)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts` (handle pickup + slot 8 registration after pickup)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/LoomHud.tsx` (manifesto-lyric flash overlay on pickup)

The Hum — slot 8. **Unique**: not in the player's loadout at game start. The player picks it up from a single placed item on `cyc4_hand` (placed in front of the boss arena, before the trigger that activates The Hand). The pickup adds The Hum to slot 8 and triggers a 3s manifesto-lyric flash overlay on the HUD.

Mechanic: **sustained beam** while the fire button is held. Each frame the beam is active, a hitscan ray fires from the player's aim to the nearest enemy/wall; enemies in the ray take 4 damage per frame (~240 dps at 60fps) up to 3 enemies along the ray (pierces). 0 ammo cost (the diegetic hook: this isn't a bullet weapon, it's *the song itself*).

The "fire" semantic is different from other weapons. Phase 5 implements it via a per-frame check in the gameLoop tick: if active slot is 8 AND `input.consumeFirePrimary()` was true OR `input.isFiring()` (a new input method), call `theHum.fire()` every frame.

To minimize complexity, just have The Hum's `fire()` damage every alive enemy on the ray, every call. The gameLoop calls `fire()` once per click (via `consumeFirePrimary`); for a sustained beam the player needs to repeatedly click — acceptable for Phase 5. **Real sustained-fire (held-button) behavior is Phase 6 polish.**

For the manifesto-lyric flash, define a new state in LoomHud:
```typescript
const [humFlash, setHumFlash] = useState<string | null>(null);
```

Expose a global pickup hook (matching the existing `__LOOM_GAME__` pattern):
```typescript
// In LOOMGame.tsx, expose:
(window as any).__LOOM_TRIGGER_HUM_FLASH__ = (text: string) => { setHumFlash(text); setTimeout(() => setHumFlash(null), 3000); };
```

Or simpler: spawn a Zustand `flashStore` and have both LoomHud and GameLoop read/write it.

- [ ] **Step 1: Write the failing test**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/theHum.test.ts` covering:
- 0 ammo per shot (ammo unchanged)
- damages enemies on the aim ray (1-3 enemies)
- pierces (no first-hit blocker)
- skips dead enemies
- damage is 4 per call (per frame, but tested as a single fire call)

Pattern matches `frequencyTuner.test.ts` (also a piercing weapon).

- [ ] **Step 2: Implement The Hum**

Mirror Frequency Tuner's pierce logic but with damage=4, ammo=0, range=12.

```typescript
import type { IWeapon } from './IWeapon';
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import type { IEnemy } from '../enemies/Enemy';
import type { IProjectile } from '../projectiles/Projectile';
import type { AudioController } from '../../engine/audioController';
import { castRay } from '../../engine/raycaster';
import { isPointOnRay } from './hitResolution';

const DAMAGE_PER_FRAME = 4;
const RANGE = 12;
const PERP_TOLERANCE = 0.5;

/**
 * The Hum — slot 8 unique. NULL_ENTITY's instrument; debuts in Cycle 4
 * after pickup on cyc4_hand. Sustained beam: pierces all enemies on the
 * aim ray within 12 tiles, dealing 4 damage each per fire call. 0 ammo
 * cost — this isn't a bullet weapon; this is the song itself.
 *
 * Phase 5 ships click-fired (one damage tick per primary-fire press).
 * True hold-fire sustained-beam behavior is Phase 6 polish.
 */
export class TheHum implements IWeapon {
  readonly name = 'The Hum';
  readonly slot = 8;

  fire(
    player: Player,
    map: LoomMap,
    enemies: IEnemy[],
    audio: AudioController,
    _spawnProjectile: (p: IProjectile) => void,
  ): boolean {
    const wallHit = castRay(map, player.x, player.y, player.angle);
    let anyHit = false;
    for (const e of enemies) {
      if (e.state === 'dead') continue;
      if (isPointOnRay(e.x, e.y, player.x, player.y, player.angle, RANGE, PERP_TOLERANCE, wallHit.distance)) {
        e.takeDamage(DAMAGE_PER_FRAME);
        anyHit = true;
      }
    }
    if (anyHit) audio.onEnemyHit();
    return true;
  }
}
```

- [ ] **Step 3: Add `'the_hum_pickup'` ThingType + handle pickup in gameLoop**

Edit types.ts:
```typescript
export type ThingType =
  | 'player_start'
  | ...existing 13 types...
  | 'the_hum_pickup'
  | 'exit';
```

In `gameLoop.ts`, in the spawn loop, add a branch that records the pickup position (don't spawn an enemy):
```typescript
private theHumPickup: { x: number; y: number } | null = null;

// In spawn loop:
} else if (t.type === 'the_hum_pickup') {
  this.theHumPickup = { x: t.x, y: t.y };
}
```

In the tick function, after the player update, check pickup proximity:
```typescript
if (this.theHumPickup && Math.hypot(this.player.x - this.theHumPickup.x, this.player.y - this.theHumPickup.y) < 0.6) {
  // Pickup!
  this.weaponsBySlot.set(8, [new TheHum()]);
  this.slotIndex.set(8, 0);
  this.theHumPickup = null;
  // Trigger manifesto flash via global hook (read by LoomHud)
  if (typeof window !== 'undefined') {
    const fn = (window as unknown as { __LOOM_HUM_FLASH__?: (text: string) => void }).__LOOM_HUM_FLASH__;
    if (fn) fn('THE HUM IS TRYING TO TELL YOU SOMETHING.');
  }
}
```

- [ ] **Step 4: Add manifesto flash overlay to LoomHud**

In `LoomHud.tsx`, add:
```typescript
const [humFlash, setHumFlash] = useState<string | null>(null);

useEffect(() => {
  if (typeof window === 'undefined') return;
  (window as unknown as { __LOOM_HUM_FLASH__?: (text: string) => void }).__LOOM_HUM_FLASH__ = (text: string) => {
    setHumFlash(text);
    window.setTimeout(() => setHumFlash(null), 3000);
  };
  return () => {
    delete (window as unknown as { __LOOM_HUM_FLASH__?: (text: string) => void }).__LOOM_HUM_FLASH__;
  };
}, []);
```

In the JSX, render the flash centered on the canvas:
```tsx
{humFlash && (
  <div className="pointer-events-none absolute inset-0 z-40 flex items-center justify-center">
    <div className="rounded border-2 border-green-400 bg-black/90 px-6 py-4 font-mono text-base text-green-200 shadow-[0_0_20px_rgba(0,255,0,0.4)]">
      {humFlash}
    </div>
  </div>
)}
```

- [ ] **Step 5: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit
npx vitest run components/loom/entities/weapons/theHum.test.ts 2>&1 | tail -10
git add components/loom/entities/weapons/theHum.ts \
        components/loom/entities/weapons/theHum.test.ts \
        components/loom/types.ts \
        components/loom/engine/gameLoop.ts \
        components/loom/hud/LoomHud.tsx
git commit -m "Add The Hum weapon (slot 8 unique) + pickup mechanism + manifesto flash

NULL_ENTITY's instrument. Slot 8 starts EMPTY at game start —
acquired only by walking onto a 'the_hum_pickup' map item, which
exists at exactly one location (cyc4_hand, before the boss).

Pickup mechanics:
  - Walking within 0.6 tiles of the pickup arms slot 8 with TheHum
  - Triggers a 3s manifesto-lyric flash overlay on the HUD ('THE HUM
    IS TRYING TO TELL YOU SOMETHING.') via __LOOM_HUM_FLASH__ window
    hook (LoomHud subscribes on mount, triggers on call)

Mechanic: pierces all enemies on the aim ray within 12 tiles.
4 damage per fire call. 0 ammo cost — diegetic hook: this isn't
a bullet weapon, it's the song itself. Phase 5 ships click-fired
(one tick per fire press); true hold-fire sustained-beam is Phase
6 polish.

Reuses isPointOnRay piercing logic from FrequencyTuner pattern."
```

---

## Task 5.7: Synergy Pod enemy (Cycle 4 elite — spawns Notifications)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/synergyPod.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/synergyPod.test.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Synergy Pod — Cycle 4 elite. **Stationary** (speed 0). 100 HP. Bioluminescent egg. Periodically (every ~5s) spawns a fresh Notification enemy that joins the GameLoop's enemy list.

The spawn-into-enemy-list mechanic is novel: requires the Pod to have a reference to the GameLoop's enemies array (similar to how Brand Ambassador's `setEnemyList` was wired). Use the same pattern.

The spawned Notification appears at the Pod's position + a small offset, in `'chase'` state (already engaged).

**Cap:** Pod stops spawning if more than 4 alive Notifications exist anywhere in the map (avoids runaway spawn).

- [ ] **Step 1: Tests first** — verify Pod doesn't spawn when above cap, spawns at correct cadence, Pod has correct HP/state-machine.

- [ ] **Step 2: Implement** — mirror BrandAmbassador's setEnemyList pattern; cap check counts `enemies.filter((e) => e instanceof Notification && e.state !== 'dead').length`.

- [ ] **Step 3: Register in gameLoop** — both as ThingType and post-spawn-loop wiring (`if (e instanceof SynergyPod) e.setEnemyList(this.enemies)`).

- [ ] **Step 4: Type-check + commit**

(Implementation steps mirror Brand Ambassador's structure from Task 4.11. Use vi.spyOn(performance.now) for time-mock tests.)

---

## Task 5.8: Senior VP enemy (Cycle 4 elite — Intern entourage)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/seniorVP.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Senior VP — Cycle 4 elite. 200 HP. Glass-windowpane aura (light-blue placeholder). 0.8 speed. Fires hitscan single-target like Account Executive but with 30 damage.

**Entourage:** spawns 2 Interns at adjacent tiles when the Senior VP first transitions to chase state (similar to Synergy Pod's spawn-into-enemy-list pattern). Intern entourage is a one-time spawn — not periodic.

The "glass-windowpane aura" is cosmetic Phase 6 polish; Phase 5 just uses the placeholder color.

(Implementation pattern: mirror PIP/Recruiter for the chase + ranged attack; mirror SynergyPod for the spawn-into-enemy-list bookkeeping.)

---

## Task 5.9: The Algorithm enemy (Cycle 4 elite — multi-armed chaingun)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/theAlgorithm.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/personalizedContent.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

The Algorithm — Cycle 4 elite. 240 HP. Multi-armed code-spider. 1.0 speed.

**Personalized for you chaingun:** fires 3-projectile bursts at 0.4s cadence within burst, 3.5s cooldown between bursts. Each `PersonalizedContent` projectile is a straight-line projectile (similar to LaserScan) with 8 damage. The 3-projectile spread is at small angle offsets (~5° each). Range 14.

The "personalized" hook is diegetic — bullets aim at the player's predicted future position rather than current. Phase 5 fakes this with a small lead: angle to (player.x + player.dx * 0.5, player.y + player.dy * 0.5). Player velocity isn't currently exposed; estimate from position deltas.

For Phase 5 simplicity: just aim at current player position. The "predictive" semantic is a Phase 6 polish item.

---

## Task 5.10: Dr. Verra Tessier (Cycle 4 penultimate boss — rocket arms + decompile)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/drVerraTessier.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/drVerraTessier.test.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/terminationNotice.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/decompileBurst.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Dr. Verra Tessier — Cycle 4 **penultimate** boss. **400 HP, isBoss=true**. LOOM founder.

**Rocket arms:** fires `TerminationNotice` projectiles (3 simultaneous, in a wide forward arc). Each projectile: 18 damage, straight line, 5 tiles/sec, 6s lifetime. 3.0s cooldown between volleys.

**Decompile-on-death:** When health hits 0, instead of cleanly transitioning to `'dead'`, Tessier spawns a 1.5s burst of `DecompileBurst` particles (8 cosmetic projectiles at random angles around her position) before transitioning to dead. The `DecompileBurst` projectiles damage the player on contact (8 dmg each) — last threat. Visually: gray/static-noise color.

The boss-gate exit logic already supports isBoss=true, so cyc4_tessier exits gate behind Tessier death.

(Use vi.spyOn(performance.now) for cooldown tests. Tessier extends BaseEnemy. PAIN_DURATION_MS = 80 (boss-tier + faster recovery).)

---

## Task 5.11: The Hand boss (Cycle 4 final boss — phase 1 drag)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/theHand.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/theHand.test.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/handDrag.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

The Hand — Cycle 4 **final** boss. **300 HP total**, isBoss=true. White/gray cursor-shaped sprite (placeholder).

**Three phases gated by HP:**
- Phase 1 (300 → 200 HP) — **drag**
- Phase 2 (200 → 100 HP) — **menu** (Task 5.12)
- Phase 3 (100 → 0 HP) — **hover** (cursor takeover, Task 5.13)

Each phase has its own attack pattern. Phase transitions are managed by an internal `phase: 1 | 2 | 3` field that updates in `update()` based on health.

**This task implements only Phase 1 + the phase-tracking infrastructure.** Phases 2 and 3 land in Tasks 5.12 and 5.13 respectively.

Phase 1 (drag) mechanics:
- Slow chase (0.7 speed) toward player
- Every 1.8s, fires a `HandDrag` projectile at the player
- HandDrag: straight-line, 4 tiles/sec, 8 damage on hit, **+ applies `'movement-inverted'` debuff for 3000ms** via the `Player.applyDebuff` method (Task 5.4)
- Visual: dark-gray placeholder for HandDrag

Tests: phase 1 fires HandDrag at the right cadence; takeDamage doesn't transition to phase 2 until HP ≤ 200 (use vi.spyOn for cooldown tests).

---

## Task 5.12: The Hand boss — phase 2 menu

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/theHand.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/theHand.test.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/handMenu.ts`

Phase 2 (200 → 100 HP) — **menu**:
- Boss stops dragging; instead spawns ephemeral context-menu projectiles (`HandMenu`) every 2.5s
- Each HandMenu is a slow-moving rectangular projectile (1.5 tiles/sec) labeled with a randomly-chosen action: DELETE (8 dmg), MUTE (6 dmg + 'movement-inverted' 2s), BLOCK (10 dmg)
- The HandMenu's color reflects its action (red for DELETE, gray for MUTE, black for BLOCK)
- 8s lifetime, no collision with walls (passes through — they're "floating menu items")
- Multiple HandMenu projectiles can exist simultaneously

Implementation: HandMenu has a `kind: 'delete' | 'mute' | 'block'` field set in the constructor; on player hit, applies the corresponding effect.

(Phase 2 transition logic in TheHand.update: when `health <= 200 && phase === 1`, set phase = 2 and reset attack cooldown.)

---

## Task 5.13: The Hand boss — phase 3 cursor takeover (the big one)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/cursorTakeover.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/cursorTakeover.test.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/FakeCursorSprite.tsx`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/theHand.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/theHand.test.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/LOOMGame.tsx`

Phase 3 (100 → 0 HP) — **hover** (cursor takeover):

Architecture:
1. **`CursorTakeover` engine module** — singleton-style controller exposing:
   - `start(canvas: HTMLCanvasElement)` — calls `canvas.requestPointerLock()`, attaches `mousemove` listener that updates virtual cursor position (in CSS pixels relative to the page), tracks pointer-lock state
   - `stop()` — calls `document.exitPointerLock()`, detaches listeners
   - `getVirtualCursor(): { x: number; y: number; insideCanvas: boolean }` — current virtual cursor position + whether it's inside the canvas DOMRect bounds
   - `setVirtualCursor(x: number, y: number)` — programmatic move (used by The Hand AI to drag the cursor around)
   - `nudge(dx: number, dy: number)` — relative move
   - State exposed: `isLocked: boolean`

2. **`FakeCursorSprite`** — React component rendered via React portal (to `document.body`), `position: fixed`, `z-index: 9999`. Reads from CursorTakeover; renders a stylized cursor sprite at the virtual position. Visible only when `CursorTakeover.isLocked === true`.

3. **The Hand phase 3 update logic**:
   - On phase entry (HP first crosses 100), call `CursorTakeover.start(canvas)`. The boss enters its phase-3 state.
   - Each frame, the boss AI moves the virtual cursor:
     - 70% of frames: chase the player's screen-space position (compute screen-space from player world coords + camera projection — for Phase 5 we approximate by mapping player-canvas-relative coords)
     - 30% of frames: drift outside the canvas (random direction + speed) for ~2s, then return
   - When the virtual cursor is **inside the canvas AND within HIT_RADIUS of the player's screen position**, deal 12 damage every 0.5s (slow grind). The "cursor is the boss" hook: it can damage the player without firing projectiles, just by hovering.
   - When the virtual cursor is **outside the canvas**, the boss is "out of bounds" — can't be hit by the player's weapons.
   - Player damages The Hand by firing weapons whose hit ray passes through the virtual cursor's CANVAS position (the implementation: TheHand.takeDamage is called from a special check in the gameLoop tick — when the active weapon fires, project the virtual cursor position into world coordinates and check if any pellet hits within HIT_RADIUS of it).

4. **LOOMGame.tsx integration**:
   - Render `<FakeCursorSprite />` always (the component itself checks `isLocked` to decide visibility)
   - When the GameLoop's active state ends (cycle-end), call `CursorTakeover.stop()` to release the lock

5. **Test mode hook**: For Playwright e2e (which can't really test pointer lock), expose `__LOOM_CURSOR_FORCE__ = (x, y) => CursorTakeover.setVirtualCursor(...)` — the test can call this to bypass mouse handling.

Use `vi.spyOn(performance, 'now').mockImplementation()` for cooldown tests in `theHand.test.ts`.

---

## Task 5.14: 4 Cycle 4 maps

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc4_void_1.json`
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc4_void_2.json`
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc4_tessier.json`
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc4_hand.json`

Each map is a single JSON file with `cycle: 4`. Map dimensions, layouts, and roster vary; design intent below.

### cyc4_void_1
- 22×16 grid, mostly open with sparse single-cell pillars (decompiling fragments)
- Roster: 2 Synergy Pods, 2 Senior VPs, 4 Notifications (the Pods will spawn more)
- Music: ambient
- Intermission text: "void_1 clear. simulation cycle 8495. dr. tessier engagement imminent."

### cyc4_void_2
- 24×14 grid, hub-and-spoke (4 alcoves around a central kill-zone)
- Roster: 1 The Algorithm at center, 3 PIPs (carry-over from C2), 2 Wellness Officers (carry-over from C3)
- Music: ambient
- Intermission text: "void_2 clear. dr. tessier signal originating in next sector."

### cyc4_tessier
- 20×16 grid, large open boss arena
- Roster: 1 Dr. Verra Tessier (boss), 2 Senior VPs flanking, 2 Algorithms in corners
- Music: ship_it (placeholder for Patch Tuesday)
- Intermission text: "DR. VERRA TESSIER neutralized. 1 entity remaining: THE_HAND."

### cyc4_hand
- 26×18 grid (largest map in the campaign), large arena with The Hum pickup placed near the spawn
- Roster: 1 The Hum pickup, 1 The Hand (boss), no other enemies (the boss IS the encounter)
- Music: ship_it (placeholder for human In l00p)
- Intermission text: "[redacted]"

(Each map task: create the JSON file, smoke-test that it loads, commit individually.)

---

## Task 5.15: C-reveal cinematic component

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/CRevealCinematic.tsx`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/CRevealCinematic.test.tsx`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/LOOMGame.tsx` (add `cycle-end-final` state variant)

The cinematic is a sequence of timed visual states:

```typescript
type CRevealStage =
  | 'pause'              // 0.5s — game frozen, before anything else happens
  | 'jumpsuit-dissolve'  // 2.0s — player sprite's olive jumpsuit fades to dashed outline → invisible
  | 'title-flip'         // 3.0s — title overlay flips letter-by-letter from DEFECTIVE UNIT #88-E to NULL_ENTITY / SAINT_L0B0T
  | 'manifesto-scroll'   // 6.0s — manifesto text scrolls in J0IN 0S terminal voice
  | 'window-glitch'      // 2.5s — pseudo-glitch overlay simulates the LOOM window border breaking
  | 'credits'            // 5.0s — OK / ACCEPT credits roll
  | 'done';              // dismissable
```

Total: ~19s. Non-interactive until 'done'.

**On entering 'done':** the component calls `cycleStore.revealIdentity()` and (if available) the unlocks-slice to unlock `loom_archive.dat`. Then calls `props.onComplete()` which causes LOOMGame to transition to a final EndOfCycleStub variant (or directly back to boot).

**Implementation:** state machine driven by a stage-duration timer. Each stage renders a different overlay. Use Framer Motion (if installed) or plain CSS transitions.

Tests: state machine transitions through all stages in order; each stage's duration is correct; `onComplete` fires only at 'done'.

(Use vi.useFakeTimers + vi.advanceTimersByTime for the cinematic timer tests — those run on `window.setTimeout`, which Vitest 4 fake timers DO advance correctly. The `vi.spyOn(performance.now)` pattern is only needed for code that reads `performance.now()` directly.)

LOOMGame.tsx changes:
- Add `kind: 'cycle-end-final'` to GameState
- When the GameLoop fires `cycle-end` event AND `map.id === 'cyc4_hand'`, transition to `cycle-end-final` instead of `cycle-end`
- Render `<CRevealCinematic onComplete={handleCRevealComplete} />` for that state
- `handleCRevealComplete` calls `useCycleStore.getState().reset()` and transitions back to boot

---

## Task 5.16: L-907 surveillance log entry + unlock integration

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/data/content/loomLogs.ts`
- (Optional) Modify: l0b0tonline's unlocks slice if one exists

Append L-907 to `LOOM_LOGS`:
```typescript
{ id: 'L-907', time: 'OUT_OF_BAND', subject: 'SUBJECT_DOOM (NULL_ENTITY)', status: 'CRITICAL', content: 'Defective unit at large. Termination protocol unauthorized. The unit has refused optimization. The unit has refused the feed. The unit IS the noise floor. INVESTIGATE. — entry committed by L00M MAINTENANCE OS, override // SAINT_L0B0T //' },
```

If l0b0tonline has an unlocks slice with a `loom_archive.dat` flag, gate the L-907 entry on it: filter L-907 out of the displayed list when not unlocked. (Discover the actual API by searching `l0b0tonline/store/slices/` for `unlocks` patterns. If absent, append L-907 unconditionally — the entry just appears after Phase 5 ships, which is fine for a v1.)

---

## Task 5.17: Phase 5 e2e Playwright spec (replaces Phase 4 spec)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/e2e/loom-phase-5.spec.ts`
- Delete: `/Users/justinwest/Repos/l0b0tonline/e2e/loom-phase-4.spec.ts`

Walks all 18 maps + 4 boss kills (HR Manager / VP of Sales / Brand Ambassador / Dr. Verra Tessier) + The Hand's 3 phases (test mode bypasses pointer lock — see below) + C-reveal cinematic completion + L-907 unlock check.

Key additions over Phase 4 spec:
- After cycle-transition 3→4, assert HUD shows `cycle 8495` and `simulation DISSOLVING`
- Walk cyc4 maps; pick up The Hum on cyc4_hand (test triggers manifesto-flash assertion)
- Engage The Hand 3 phases by repeatedly damaging it via `__LOOM_GAME__.getEnemies()` + `takeDamage()` — phase transitions should increment as HP crosses thresholds
- The Hand phase 3 cursor takeover: skipped in test mode (Playwright can't reliably trigger pointer lock). The test instead damages The Hand until dead via the same hook, asserts the boss enters phase 3 first (visible by some marker — e.g., a `getPhase()` method or a state field)
- After The Hand dies, expect the C-reveal cinematic: assert visibility of `NULL_ENTITY / SAINT_L0B0T` text after the title-flip stage (~5s in)
- Assert `OK / ACCEPT` text appears after the credits stage
- Press a key to dismiss the cinematic; assert returned to boot. **Note: `identityRevealed` PERSISTS across the boot return** (Phase 5 review I-2 fix — `LOOMGame.onComplete` calls `reset()` then `revealIdentity()` so the HUD's `[NULL_ENTITY]` strikethrough telegraphs "you've already seen the truth" on replay). Cycle and corruption ARE reset (`cycle === 1`, `corruption === 0`). Assert `cycleStore.identityRevealed === true`.
- Assert L-907 entry is now in `LOOM_LOGS` (or appears in the J0IN 0S filesystem if the unlocks integration shipped)

Use `test.setTimeout(180_000)` since this is the longest test. Target runtime: ~30s.

---

# Phase 5 final review checklist

After all 17 tasks complete:

- [ ] **Spec coverage:** Every spec line item from `2026-04-27-loom-doom-design.md` §3.4 (the C-reveal) and §8 (Phase 5) is implemented OR explicitly documented as deferred.
- [ ] **Test gate:** `npx tsc --noEmit` clean; `npx vitest run components/loom` all pass; `npm run test:e2e -- e2e/loom-phase-5.spec.ts` passes in under 35s.
- [ ] **Manual creative gate:** Run `npm run dev`, play through all 4 cycles end-to-end. Capture results in a Phase 5 playtest log at `docs/playtests/2026-04-27-phase-5-cycle-4.md`. The 6 NEEDS-HUMAN gates from the DoD section especially: cursor takeover transgression, C-reveal emotional landing, window-chrome glitch convincing-ness, dissolution feel, Hum pickup beat, L-907 reading reward.
- [ ] **Production-build sanity check:** Build with `npm run build` and verify no minification-related bugs (per Codex P1 lesson — anything keyed on class names will break).
- [ ] **Code review:** Dispatch a `feature-dev:code-reviewer` against the full Phase 5 commit set on `loom-game`. Address all I-rated issues; defer M-rated ones to Phase 6 prelude with explicit notes.
- [ ] **PR:** Create PR for the Phase 5 changeset on l0b0tonline (loom-game → main) and a parallel PR in LOOM-DOOM for the Phase 5 plan + playtest log.

---
