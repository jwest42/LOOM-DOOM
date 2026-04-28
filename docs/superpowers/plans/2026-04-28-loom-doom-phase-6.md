# LOOM Phase 6 Implementation Plan — Polish + Accessibility + Code-Side Audio

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Polish the now-narratively-complete LOOM campaign for ship readiness. Address all Phase 5 review carry-overs, add accessibility hooks (reduced-motion, keyboard mouse-look, colorblind palette), introduce skill-level tuning + the Termination Letter consumable, finish the weapon-polish items deferred from earlier phases, and lay groundwork for ship-day audio work without blocking on actual content composition.

**Phase 6 Definition of Done (per design spec §8):** "an unfamiliar tester can play start to finish without a placeholder asset or jarring difficulty spike."

**Architecture:** Extends Phase 0–5. No new subsystems. Surgical additions and refactors:
- Accessibility flags on a new `accessibilityStore` (reduced-motion, colorblind palette, keyboard mouse-look)
- Skill-level on the existing `cycleStore` (or a new dedicated slice if cleaner) — `'easy' | 'normal' | 'hard'`, affects player HP / enemy damage multipliers / ammo regen
- Termination Letter consumable as a new `IItem`-shaped pickup in the existing thing-spawn pipeline
- Audio: a single new `AudioController.setHumIntensity(level)` method called from `onMapLoad` (60Hz hum amplitude ramps by cycle)
- A handful of code refactors and missing-test additions for Phase 5 review carry-overs

**Tech Stack:** Same as Phase 5 — TypeScript 5.8, React 19, Vite 6, Tailwind v4, Zustand 5, WebGL 2, Web Audio API, Vitest, Playwright. New API surface: `window.matchMedia('(prefers-reduced-motion: reduce)')` + `localStorage` for accessibility persistence.

**Pre-existing state:**
- l0b0tonline `loom-game` branch is fresh off `main` (commit `7da1209` — Phase 5 PR #164 merged).
- LOOM-DOOM worktree `claude/awesome-sutherland-aad854` is on master (Phase 5 docs PR #4 merged).
- All Phase 5 review fixes already shipped (commits `463d4c0` and `6d14818`); all Codex P1+P2 comments addressed.

**OUT OF SCOPE — explicitly deferred to Phase 7 or external work** (kept off the critical path):
- **Real *Patch Tuesday (Heaven Vers!0n)* and *human In l00p* music tracks** — requires Suno composition + WAV import to `services/soundService.ts`. Phase 7 ship task.
- **Real sprite art** replacing colored rectangles — requires AI sprite generation pipeline (Nano Banana / l0b0t-photoshoot-generator). Spec §8 risk #4 calls this "the biggest single time sink." Phase 7.
- **Surveillance log voiceover** for C2/C3 — requires audio recording / AI-TTS. Phase 7.
- **Mobile responsiveness investigation** — requires device-specific testing. Spec §8 calls it stretch. Phase 7.
- **Final difficulty tuning** based on playtester data — requires real playtester sessions. Phase 7.

These deferrals keep Phase 6 honest about what's *codeable* without blocking on content/asset/recording work.

---

## File structure (post-Phase-6, in l0b0tonline)

New files (NEW), modified files (MOD):

```
l0b0tonline/
├── components/
│   ├── LOOMGame.tsx                                              (MOD — render <SkillLevelSelect /> on first boot)
│   └── loom/
│       ├── engine/
│       │   ├── audioController.ts                                (MOD — setHumIntensity by cycle)
│       │   ├── gameLoop.ts                                       (MOD — register termination_letter pickup, apply skillLevel multipliers)
│       │   ├── input.ts                                          (MOD — Q/E keyboard mouse-look fallback)
│       ├── entities/
│       │   ├── enemies/
│       │   │   ├── seniorVP.test.ts                              (NEW — closes M-6 coverage gap)
│       │   │   ├── theAlgorithm.test.ts                          (NEW — closes M-6 coverage gap)
│       │   │   ├── theHand.ts                                    (MOD — explicit returns between phase blocks per M-3)
│       │   │   ├── theHand.test.ts                               (MOD — extended for HandMenu anti-repeat per M-5)
│       │   │   ├── theAlgorithm.ts                               (MOD — predictive aim per spec §5.2 deferred item)
│       │   ├── projectiles/
│       │   │   ├── handMenu.ts                                   (MOD — kind selection w/o immediate repeat per M-5)
│       │   │   ├── handMenu.test.ts                              (NEW — test for anti-repeat)
│       │   │   └── terminationLetter.ts                          (NEW — area-clear consumable pickup)
│       │   └── weapons/
│       │       ├── theHum.ts                                     (MOD — true hold-fire sustained beam)
│       │       ├── theHum.test.ts                                (MOD — extended for hold-fire timing)
│       │       └── terminationLetterItem.ts                      (NEW — Termination Letter inventory entry)
│       ├── hud/
│       │   ├── LoomHud.tsx                                       (MOD — accessibility-aware glitch toggling)
│       │   ├── CRevealCinematic.tsx                              (MOD — accessibility reduced-motion path)
│       │   ├── SkillLevelSelect.tsx                              (NEW — pre-boot skill picker)
│       │   └── SkillLevelSelect.test.tsx                         (NEW)
│       └── store/
│           ├── accessibilityStore.ts                             (NEW — reducedMotion, colorblind, keyboardLook)
│           ├── accessibilityStore.test.ts                        (NEW)
│           ├── cycleStore.ts                                     (MOD — skillLevel field + setSkillLevel action)
│           └── cycleStore.test.ts                                (MOD — extended)
├── data/
│   └── content/
│       └── loomLogs.ts                                           (MOD — gate L-907 via unlocks slice OR document; see Task 6.5)
└── public/
    └── data/
        └── loom/
            └── maps/
                ├── cyc2_glass_confroom_boss.json                 (MOD — add 1 termination_letter pickup)
                ├── cyc3_wellness_program.json                    (MOD — add 1 termination_letter pickup)
                └── cyc4_void_2.json                              (MOD — add 1 termination_letter pickup)
```

E2E:
- `e2e/loom-phase-6.spec.ts` (NEW — extends Phase 5 spec with skill-level + Termination Letter + accessibility flag verification)
- `e2e/loom-phase-5.spec.ts` (DELETE — Phase 6 spec is a strict superset)

---

# Phase 6 Definition of Done (DoD)

A new player can:

1. Boot J0IN 0S, click loom.exe — sees a SkillLevelSelect screen offering Easy / Normal / Hard
2. Picks a skill level — game starts with appropriate HP/damage/ammo multipliers
3. Plays through Cycles 1+2+3+4 normally (all 18 maps)
4. Picks up Termination Letter consumables (1 per cycle from C2 onward = 3 total) — uses them via a new keybind to clear all enemies in a 6-tile radius
5. If they have OS-level prefers-reduced-motion enabled, glitch effects (HUD char-break, line fades, window-glitch overlay) are disabled — the cinematic still plays but without the visually aggressive overlays
6. If they enable colorblind mode (toggle in J0IN 0S settings... actually we'll skip the toggle UI — just default to NORMAL palette for v1 and let Phase 7 add the toggle), enemy/projectile colors shift toward a deuteranopia-safe palette — **scoped down**: Phase 6 only ADDS the palette code path; the toggle UI lands later
7. If they prefer keyboard look, they can press Q/E to turn left/right (instead of mouse-look)
8. The 60Hz hum audibly intensifies as cycles progress (audio mix code-driven; uses existing soundService noise channel)
9. The Hum (slot 8) sustains while fire button held (true hold-fire)
10. The Algorithm aims with a small predictive lead based on player velocity
11. After defeating The Hand: cinematic plays, L-907 entry appears in J0IN 0S filesystem (gated per Task 6.5 decision)

**Automation gate:** Phase 6 Playwright spec extends the Phase 5 spec with:
- Skill level selection on first boot (test selects 'normal' to mirror Phase 5 baseline)
- Termination Letter pickup test on cyc2_glass_confroom_boss
- Reduced-motion class on body when prefers-reduced-motion media query is mocked
- The Hum hold-fire damage scaling
- Same 18-map walkthrough + boss kills + C-reveal as Phase 5

**Manual creative gates** (NEEDS-HUMAN, captured in Phase 6 playtest log):
- Termination Letter feels like a meaningful "oh shit" room-clear, not a trivial cheat
- Skill-level differences are observable but not punishing on Easy / not bullet-spongy on Hard
- 60Hz hum ramp builds dread without being uncomfortable
- Reduced-motion mode preserves the campaign's emotional payoff (cinematic still lands)

---

# Tasks

## Task 6.1: Phase 5 review carry-overs M-3, M-5 — TheHand explicit returns + HandMenu anti-repeat

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/theHand.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/handMenu.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/theHand.test.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/handMenu.test.ts`

**M-3 (TheHand explicit returns):** Defensive code-clarity fix — ensure phase-1/phase-2/phase-3 blocks in `update()` end with explicit `return;` so a future maintainer adding shared post-phase code can't accidentally fall through.

**M-5 (HandMenu anti-repeat):** The Hand's phase-2 menu projectiles currently pick `kind: 'delete' | 'mute' | 'block'` uniformly randomly. Adjacent same-kind picks feel like a frustration spike. Add a `lastKind` field on TheHand; when picking the next kind, exclude `lastKind` from the candidate set.

- [ ] **Step 1: Apply M-3 explicit returns**

In `theHand.ts`, find `update()`. Each phase block (the `if (this.phase === 1)`, `if (this.phase === 2)`, `if (this.phase === 3)` blocks) currently ends with a movement step. Add `return;` at the end of each block so phase-2 logic never runs after phase-1 movement (defensive; can't happen today but cheap insurance).

```typescript
    if (this.phase === 1) {
      // ... phase 1 logic ...
      return;  // <- defensive
    }

    if (this.phase === 2) {
      // ... phase 2 logic ...
      return;  // <- defensive
    }

    if (this.phase === 3) {
      // ... phase 3 logic ...
      return;  // <- defensive (already returns in places, but adds clarity)
    }
```

- [ ] **Step 2: Apply M-5 anti-repeat to HandMenu kind selection**

Add `lastKind: HandMenuKind | null = null` field to TheHand. Update the phase-2 spawn logic:

```typescript
const HAND_MENU_KINDS = ['delete', 'mute', 'block'] as const;
// ... in phase 2 spawn:
const candidates = HAND_MENU_KINDS.filter((k) => k !== this.lastKind);
const kind: HandMenuKind = candidates[Math.floor(Math.random() * candidates.length)];
this.lastKind = kind;
```

(First call: `lastKind === null`, so all 3 candidates are eligible. Subsequent calls: 2 candidates, never repeating.)

- [ ] **Step 3: Tests**

Add to `theHand.test.ts` in the phase-2 describe block:

```typescript
it('phase 2 never picks the same kind twice in a row', () => {
  const h = new TheHand(13, 13);
  h.takeDamage(100); // → phase 2
  const observed: string[] = [];
  let virtualNow = 100; // past pain
  const nowSpy = vi.spyOn(performance, 'now').mockImplementation(() => virtualNow);
  try {
    const { ctx, spawned } = makeCtx(20, 13);
    for (let i = 0; i < 20; i++) {
      h.update(0.016, ctx);
      virtualNow += 2600; // past 2.5s cooldown
    }
    for (const p of spawned) {
      const m = p as { kind?: string };
      if (m.kind) observed.push(m.kind);
    }
    for (let i = 1; i < observed.length; i++) {
      expect(observed[i]).not.toEqual(observed[i - 1]);
    }
  } finally {
    nowSpy.mockRestore();
  }
});
```

Create `handMenu.test.ts` with basic projectile-behavior tests if it doesn't exist (kind sets damage/color/debuff correctly).

- [ ] **Step 4: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
npx vitest run components/loom/entities/enemies/theHand.test.ts components/loom/entities/projectiles/handMenu.test.ts 2>&1 | tail -10
git add components/loom/entities/enemies/theHand.ts \
        components/loom/entities/enemies/theHand.test.ts \
        components/loom/entities/projectiles/handMenu.ts \
        components/loom/entities/projectiles/handMenu.test.ts
git commit -m "TheHand M-3 + M-5 polish (Phase 5 review carry-overs)

M-3: explicit return; at end of each phase block in TheHand.update()
to prevent any future regression where shared post-phase code could
unintentionally execute for the wrong phase. Defensive only — the
existing logic was correct.

M-5: HandMenu kind selection now excludes the last picked kind from
the candidate set, so adjacent shots never repeat. First call has no
lastKind so all 3 are candidates; subsequent calls pick from 2.
Frustration-mitigation per Phase 5 review.

Tests: 20-iteration assertion that no two consecutive HandMenu kinds
match. handMenu.test.ts gets a smoke-test for kind→damage/color
mapping.
"
```

---

## Task 6.2: Phase 5 review carry-over M-6 — SeniorVP + TheAlgorithm unit tests

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/seniorVP.test.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/theAlgorithm.test.ts`

Closes the test-coverage gap from Phase 5 (Synergy Pod has 8 tests; Senior VP and The Algorithm have 0 each, only e2e coverage).

5–6 tests each, using the established `vi.spyOn(performance, 'now')` pattern. Mirror SynergyPod's structure.

**SeniorVP tests:** isBoss=false / health=200, idle/chase transitions, hitscan damage on direct line of sight, 2-Intern entourage spawns on first chase (one-shot), no entourage spawns on subsequent chase resets, dies cleanly.

**TheAlgorithm tests:** isBoss=false / health=240, idle/chase transitions, fires 3 PersonalizedContent projectiles in burst at attack range, respects 3.5s cooldown, projectile spread is ±5°, dies cleanly.

- [ ] **Step 1: Write SeniorVP tests** (mirror SynergyPod pattern + standard chase/attack tests)

- [ ] **Step 2: Write TheAlgorithm tests** (similar)

- [ ] **Step 3: Type-check + run + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/entities/enemies/seniorVP.test.ts components/loom/entities/enemies/theAlgorithm.test.ts 2>&1 | tail -10
git add components/loom/entities/enemies/seniorVP.test.ts \
        components/loom/entities/enemies/theAlgorithm.test.ts
git commit -m "Add SeniorVP + TheAlgorithm unit tests (Phase 5 review M-6)

SynergyPod had 8 tests; SeniorVP and TheAlgorithm had 0 each
(carry-over deferral from Phase 5 task 5.8 + 5.9).

SeniorVP: 6 tests covering isBoss=false, idle→chase via sight,
hitscan damage, 2-Intern entourage one-shot spawn, no double-spawn
on subsequent chase resets, death.

TheAlgorithm: 5 tests covering isBoss=false, idle→chase, 3-projectile
burst, ±5° spread, 3.5s cooldown, death.

Both use vi.spyOn(performance.now) per Phase 4 review I-1 lesson."
```

---

## Task 6.3: Phase 5 review carry-over M-1 — debuff stacking semantics

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/player.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/player.test.ts`

Currently `Player.applyDebuff` REPLACES expiry on reapply — so a 3000ms HandDrag debuff (Hand phase 1) followed at +1000ms by a 2000ms HandMenu mute (Hand phase 2) results in a 3000ms total (extends only by setting new expiry to t+2000=3000ms). That's actually correct EXTEND-behavior. M-1's concern: the spec might want the LONGER remaining duration to win, so a fresh shorter debuff doesn't shorten the active one.

**Decision:** new debuff expiry = `max(existingExpiresAt, now + newDurationMs)`. So a 3s debuff + 2s reapply at +1s = expiry = `max(3000, 1000+2000) = 3000ms` (no shortening). A 3s debuff + 4s reapply at +1s = `max(3000, 1000+4000) = 5000ms` (extends).

- [ ] **Step 1: Update Player.applyDebuff**

```typescript
applyDebuff(kind: DebuffKind, durationMs: number): void {
  const existing = this.debuffs.find((d) => d.kind === kind);
  const newExpiresAt = performance.now() + durationMs;
  if (existing) {
    existing.expiresAt = Math.max(existing.expiresAt, newExpiresAt);
  } else {
    this.debuffs.push({ kind, expiresAt: newExpiresAt });
  }
}
```

- [ ] **Step 2: Update tests**

The existing "reapplying a debuff replaces the previous expiry" test asserts replace-semantics. Change to "reapplying a debuff with a shorter duration does not shorten the active one":

```typescript
it('reapplying a shorter debuff does NOT shorten the active expiry', () => {
  let virtualNow = 0;
  const nowSpy = vi.spyOn(performance, 'now').mockImplementation(() => virtualNow);
  try {
    const p = new Player(0, 0, 0);
    p.applyDebuff('movement-inverted', 3000); // expires at 3000
    virtualNow = 1000;
    p.applyDebuff('movement-inverted', 1500); // would expire at 2500; but original 3000 wins
    virtualNow = 2999;
    expect(p.movementInverted).toBe(true);
    virtualNow = 3001;
    expect(p.movementInverted).toBe(false);
  } finally {
    nowSpy.mockRestore();
  }
});

it('reapplying a longer debuff extends the active expiry', () => {
  let virtualNow = 0;
  const nowSpy = vi.spyOn(performance, 'now').mockImplementation(() => virtualNow);
  try {
    const p = new Player(0, 0, 0);
    p.applyDebuff('movement-inverted', 1000);
    virtualNow = 500;
    p.applyDebuff('movement-inverted', 2000); // expires at 2500 — wins over 1000
    virtualNow = 1500;
    expect(p.movementInverted).toBe(true); // would have expired at 1000 under replace
    virtualNow = 2501;
    expect(p.movementInverted).toBe(false);
  } finally {
    nowSpy.mockRestore();
  }
});
```

- [ ] **Step 3: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/entities/player.ts components/loom/entities/player.test.ts
git commit -m "Player.applyDebuff uses max-expiry (Phase 5 review M-1)

Reapplying a debuff with a shorter remaining duration could previously
shorten the active expiry — e.g., a 3s HandDrag followed at +1s by a
2s HandMenu mute would expire at +3s under the old replace-semantics
(2s < 2s remaining of original means net same, but a 1s reapply at
+2s of a 3s debuff would have shortened from 3s → 3s, depending on
ordering).

New semantics: existingExpiresAt = max(existing, now + newDuration).
A reapply can extend but never shorten. Aligns with player intuition
that 'getting a fresh debuff shouldn't make the existing one go away
faster.'

Two tests updated/added to lock in max-expiry behavior."
```

---

## Task 6.4: Accessibility — accessibilityStore + reduced-motion detection

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/store/accessibilityStore.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/store/accessibilityStore.test.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/LoomHud.tsx` (gate glitch effects on reducedMotion)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/CRevealCinematic.tsx` (reduced-motion path skips window-glitch stage)

A new Zustand slice `accessibilityStore` exposes:
- `reducedMotion: boolean` — auto-detected from `window.matchMedia('(prefers-reduced-motion: reduce)')` on init; user override possible via `setReducedMotion(true|false|'auto')`
- `colorblindPalette: 'normal' | 'deuteranopia'` — defaults `'normal'`; structure-only for now (Phase 6 doesn't ship the toggle UI; Phase 7 does)
- `keyboardLook: boolean` — Q/E turn-left/right fallback for users who can't use mouse-look

Persist via `localStorage` so the preference survives page refresh.

LoomHud reads `reducedMotion`; if true, skips:
- Line fade sine waves (mode 3)
- Char break-apart (mode 3)
- Connection blip (mode 1+) — stays text-only without the yellow flash; the surveillance log line still rotates

CRevealCinematic reads `reducedMotion`; if true:
- Skips the `window-glitch` stage (transitions directly from manifesto-scroll to credits)
- Title-flip stage uses a CSS fade instead of letter-by-letter flip

- [ ] **Step 1: Implement accessibilityStore**

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

type ReducedMotionPref = 'auto' | true | false;

interface AccessibilityState {
  reducedMotionPref: ReducedMotionPref;
  reducedMotion: boolean; // computed from pref + media query
  colorblindPalette: 'normal' | 'deuteranopia';
  keyboardLook: boolean;
  setReducedMotion: (pref: ReducedMotionPref) => void;
  setColorblindPalette: (p: 'normal' | 'deuteranopia') => void;
  setKeyboardLook: (b: boolean) => void;
}

const detectMQ = (): boolean => {
  if (typeof window === 'undefined' || typeof window.matchMedia !== 'function') return false;
  return window.matchMedia('(prefers-reduced-motion: reduce)').matches;
};

export const useAccessibilityStore = create<AccessibilityState>()(
  persist(
    (set) => ({
      reducedMotionPref: 'auto',
      reducedMotion: detectMQ(),
      colorblindPalette: 'normal',
      keyboardLook: false,
      setReducedMotion: (pref) => set(() => ({
        reducedMotionPref: pref,
        reducedMotion: pref === 'auto' ? detectMQ() : pref,
      })),
      setColorblindPalette: (p) => set({ colorblindPalette: p }),
      setKeyboardLook: (b) => set({ keyboardLook: b }),
    }),
    { name: 'loom-accessibility' },
  ),
);
```

- [ ] **Step 2: Tests**

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { useAccessibilityStore } from './accessibilityStore';

describe('accessibilityStore', () => {
  beforeEach(() => {
    useAccessibilityStore.setState({
      reducedMotionPref: 'auto',
      reducedMotion: false,
      colorblindPalette: 'normal',
      keyboardLook: false,
    });
  });

  it('default: reducedMotionPref is auto, reducedMotion based on MQ', () => {
    expect(useAccessibilityStore.getState().reducedMotionPref).toBe('auto');
  });

  it('setReducedMotion(true) overrides MQ', () => {
    useAccessibilityStore.getState().setReducedMotion(true);
    expect(useAccessibilityStore.getState().reducedMotion).toBe(true);
  });

  it('setReducedMotion(false) overrides MQ to false', () => {
    useAccessibilityStore.getState().setReducedMotion(false);
    expect(useAccessibilityStore.getState().reducedMotion).toBe(false);
  });

  it('setColorblindPalette toggles', () => {
    useAccessibilityStore.getState().setColorblindPalette('deuteranopia');
    expect(useAccessibilityStore.getState().colorblindPalette).toBe('deuteranopia');
  });

  it('setKeyboardLook toggles', () => {
    useAccessibilityStore.getState().setKeyboardLook(true);
    expect(useAccessibilityStore.getState().keyboardLook).toBe(true);
  });
});
```

- [ ] **Step 3: LoomHud integration**

Read `useAccessibilityStore((s) => s.reducedMotion)`. Wrap the line-fade opacity computation, char-break effect, and connection blip in `if (!reducedMotion)` guards. (Don't disable the surveillance log itself — just the visually aggressive effects.)

- [ ] **Step 4: CRevealCinematic integration**

Read `reducedMotion`. If true:
- Skip `window-glitch` stage entirely (advance straight from `manifesto-scroll` to `credits`)
- Replace title-flip's letter-by-letter rendering with a 2-second cross-fade

- [ ] **Step 5: Commit**

```bash
git add components/loom/store/accessibilityStore.ts \
        components/loom/store/accessibilityStore.test.ts \
        components/loom/hud/LoomHud.tsx \
        components/loom/hud/CRevealCinematic.tsx
git commit -m "Add accessibilityStore + reduced-motion gates (Phase 6)

New Zustand slice with persist middleware (loom-accessibility key in
localStorage):
  - reducedMotionPref: 'auto' | true | false (auto detects via MQ)
  - colorblindPalette: 'normal' | 'deuteranopia' (structure only;
    toggle UI lands Phase 7)
  - keyboardLook: bool (Q/E turn-left/right fallback; consumer in
    Task 6.5 input.ts edit)

LoomHud: line fades, char break-apart, connection blip yellow flash
all gated on !reducedMotion. Surveillance log itself still rotates.

CRevealCinematic: window-glitch stage skipped; title-flip uses
cross-fade instead of letter-by-letter when reducedMotion=true. The
manifesto + credits stages still render — the campaign's emotional
payoff is preserved without the visually aggressive overlays.

5 vitest tests cover store API: defaults, set methods, persist."
```

---

## Task 6.5: Accessibility + L-907 visibility — keyboard look + L-907 unlock decision

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/input.ts` (Q/E keyboard look)
- Modify: `/Users/justinwest/Repos/l0b0tonline/data/content/loomLogs.ts` (L-907 visibility decision)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/LoomInterface.tsx` (filter L-907 if not unlocked)

Two micro-tasks bundled because both are small.

**Keyboard look (Q/E):** When `accessibilityStore.keyboardLook === true`, Q-down rotates player.angle by `-MOUSE_SENSITIVITY * dt` per tick; E-down rotates by `+MOUSE_SENSITIVITY * dt`. Mouse-look continues to work (additive). This is for users who can't use a mouse OR want a deterministic way to test.

**L-907 visibility (M-2 finalized):** Per Codex P2-2 fix recommendation in PR #4, gate L-907 via a persistent unlocks slice. Discover/use the existing l0b0tonline `unlocksSlice` (search `store/slices/`). If absent, document "intentionally always-visible" with a code comment. The C-reveal cinematic's `onComplete` callback (Task 5.15) should fire the unlock action when present.

- [ ] **Step 1: Q/E keyboard look in input.ts**

Add a getter on InputController:
```typescript
isDown(code: string): boolean {
  return this.keys.has(code);
}
```
(verify if it already exists; if so, no change needed)

In `Player.update`, after the existing mouse-look block, read `accessibilityStore.getState().keyboardLook`. If true:
```typescript
if (input.isDown('KeyQ')) this.angle -= TURN_SPEED * dt;
if (input.isDown('KeyE')) this.angle += TURN_SPEED * dt;
```

`TURN_SPEED` constant ~2.5 rad/sec.

- [ ] **Step 2: L-907 unlock gating**

Search for an existing unlocks slice:
```bash
grep -rn "unlocks" /Users/justinwest/Repos/l0b0tonline/store/slices/ /Users/justinwest/Repos/l0b0tonline/components/ 2>&1 | head -10
```

If an `unlocksSlice` exists with persist middleware, add a `'loom_l907'` flag:
- The C-reveal cinematic's `onComplete` (LOOMGame.tsx) calls `useStore.getState().unlock('loom_l907')`
- In `LoomInterface.tsx` (where `LOOM_LOGS` is consumed), filter out L-907 unless the flag is set.

If no unlocks slice exists, take the **document-and-keep-visible** path: add a top-of-file comment to `loomLogs.ts`:

```typescript
/**
 * LOOM research logs.
 *
 * L-907 (SUBJECT_DOOM / NULL_ENTITY) is INTENTIONALLY always-visible
 * in v1. The narrative framing: this entry leaked from L00M MAINTENANCE
 * OS as an admin override, so the player can encounter it before
 * defeating The Hand and complete the C-reveal — it's a foreshadowing
 * artifact, not a post-game reward.
 *
 * If a future phase adds an unlocks-slice, gate L-907 on a
 * 'loom_l907' flag set by CRevealCinematic.onComplete.
 */
```

- [ ] **Step 3: Type-check + commit**

```bash
git add components/loom/engine/input.ts \
        components/loom/entities/player.ts \
        data/content/loomLogs.ts \
        components/LoomInterface.tsx \
        # ... whatever else changed
git commit -m "Add Q/E keyboard look + finalize L-907 visibility (Phase 6 M-2 + a11y)

Q/E keyboard look:
  - When accessibilityStore.keyboardLook=true, Q-down rotates left,
    E-down rotates right (additive with mouse-look)
  - 2.5 rad/sec turn speed (matches mouse-look feel at moderate
    sensitivity)
  - Lets users without a mouse play; also useful for deterministic
    Playwright e2e turn tests

L-907 visibility (Codex M-2 fix):
  - Search for existing unlocks-slice path: [resolve as the implementer
    finds]
  - If unlocks-slice exists: gate L-907 on 'loom_l907' flag set by
    C-reveal cinematic onComplete; filter in LoomInterface
  - If no unlocks-slice: top-of-file comment documents 'intentionally
    always-visible' as the v1 narrative framing (admin-leak from L00M
    MAINTENANCE OS); future phase can add the gate

Resolves Phase 5 review M-2 + Codex PR #4 P2-2."
```

(The implementer subagent will resolve the unlocks-slice question by searching the codebase and pick the appropriate path.)

---

## Task 6.6: Skill-level selector + cycleStore.skillLevel

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/store/cycleStore.ts` (add skillLevel field + setSkillLevel action)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/store/cycleStore.test.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/SkillLevelSelect.tsx`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/SkillLevelSelect.test.tsx`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/LOOMGame.tsx` (render SkillLevelSelect on first boot per session)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts` (apply skillLevel multipliers)

`SkillLevel = 'easy' | 'normal' | 'hard'`, default 'normal'. cycleStore gains the field; `reset()` does NOT clear skillLevel (it's a session-level preference, not run state).

Effects:
- **Easy:** player starts with 150 HP (vs 100), 75 ammo (vs 50); enemy damage ×0.7
- **Normal:** baseline (no multipliers)
- **Hard:** player starts with 75 HP, 50 ammo; enemy damage ×1.4; pickup HP/ammo cut in half

Multipliers applied in `Player` constructor (HP/ammo) and in enemy `takeDamage`/`update` paths (damage scaling — read from cycleStore in the damage application sites).

`SkillLevelSelect` is a simple React component shown when `state.kind === 'booting'` AND user hasn't picked yet this session. After picking, transitions to BootSequence.

- [ ] **Step 1: Add skillLevel to cycleStore**

```typescript
export type SkillLevel = 'easy' | 'normal' | 'hard';

interface CycleState {
  // ... existing fields
  skillLevel: SkillLevel;
  setSkillLevel: (s: SkillLevel) => void;
}

// In create() initial state:
skillLevel: 'normal',
setSkillLevel: (s) => set({ skillLevel: s }),

// reset() should NOT clear skillLevel:
reset: () => set((s) => ({ ...initialState, skillLevel: s.skillLevel })),
```

- [ ] **Step 2: Tests for cycleStore**

```typescript
it('skillLevel default is normal', () => {
  const s = useCycleStore.getState();
  s.reset();
  expect(s.skillLevel).toBe('normal');
});

it('setSkillLevel updates', () => {
  const s = useCycleStore.getState();
  s.setSkillLevel('hard');
  expect(useCycleStore.getState().skillLevel).toBe('hard');
});

it('reset() preserves skillLevel', () => {
  const s = useCycleStore.getState();
  s.setSkillLevel('easy');
  s.reset();
  expect(useCycleStore.getState().skillLevel).toBe('easy');
});
```

- [ ] **Step 3: Apply multipliers in Player + GameLoop**

Player constructor reads cycleStore.skillLevel and adjusts initial HP/ammo:
```typescript
constructor(...) {
  // ...
  const skill = useCycleStore.getState().skillLevel;
  if (skill === 'easy') { this.health = 150; this.ammo = 75; }
  else if (skill === 'hard') { this.health = 75; this.ammo = 50; }
  // 'normal' uses defaults
}
```

For enemy damage: simplest implementation is to add a `getDamageMultiplier(): number` helper:
```typescript
function enemyDamageMultiplier(): number {
  const s = useCycleStore.getState().skillLevel;
  return s === 'easy' ? 0.7 : s === 'hard' ? 1.4 : 1;
}
```

Then in each enemy's damage-application site (where `ctx.player.takeDamage(N)` is called), wrap: `ctx.player.takeDamage(Math.round(N * enemyDamageMultiplier()))`. Or — simpler: have `Player.takeDamage` apply the multiplier internally:
```typescript
takeDamage(damage: number) {
  const skill = useCycleStore.getState().skillLevel;
  const mult = skill === 'easy' ? 0.7 : skill === 'hard' ? 1.4 : 1;
  this.health = Math.max(0, this.health - Math.round(damage * mult));
}
```

The Player.takeDamage approach is cleanest — single source of truth.

- [ ] **Step 4: SkillLevelSelect component**

```tsx
import { useState } from 'react';
import { useCycleStore } from '../store/cycleStore';

interface SkillLevelSelectProps {
  onSelect: () => void;
}

export function SkillLevelSelect({ onSelect }: SkillLevelSelectProps) {
  const setSkillLevel = useCycleStore((s) => s.setSkillLevel);
  const [hovered, setHovered] = useState<'easy' | 'normal' | 'hard' | null>('normal');
  const pick = (s: 'easy' | 'normal' | 'hard') => {
    setSkillLevel(s);
    onSelect();
  };
  return (
    <div className="absolute inset-0 z-50 flex flex-col items-center justify-center bg-black font-mono text-green-300">
      <div className="mb-4 text-2xl">SELECT BIO_PROFILE</div>
      <div className="flex flex-col gap-2 text-base">
        <button onClick={() => pick('easy')} onMouseEnter={() => setHovered('easy')} className="border border-green-700 px-6 py-2 hover:bg-green-900/30">EASY <span className="text-green-700">— bio: 150% / queue: 150% / hostile dmg ×0.7</span></button>
        <button onClick={() => pick('normal')} onMouseEnter={() => setHovered('normal')} className="border border-green-500 px-6 py-2 hover:bg-green-900/30">NORMAL <span className="text-green-700">— spec baseline</span></button>
        <button onClick={() => pick('hard')} onMouseEnter={() => setHovered('hard')} className="border border-red-700 px-6 py-2 hover:bg-red-900/30 text-red-300">HARD <span className="text-red-800">— bio: 75% / hostile dmg ×1.4 / pickups halved</span></button>
      </div>
      <div className="mt-6 text-xs text-green-700">selection persists for this session</div>
    </div>
  );
}
```

- [ ] **Step 5: LOOMGame integration**

Add a `skillSelected: boolean` state (per-mount; resets on game close-and-reopen). `state.kind === 'booting'` and `!skillSelected` → render SkillLevelSelect. After pick: setSkillSelected(true); BootSequence renders next.

- [ ] **Step 6: Commit**

```bash
git add components/loom/store/cycleStore.ts \
        components/loom/store/cycleStore.test.ts \
        components/loom/hud/SkillLevelSelect.tsx \
        components/loom/hud/SkillLevelSelect.test.tsx \
        components/LOOMGame.tsx \
        components/loom/entities/player.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add Easy/Normal/Hard skill levels + SkillLevelSelect (Phase 6)

cycleStore.skillLevel: 'easy' | 'normal' | 'hard' (default normal).
reset() preserves skillLevel — it's a session-level preference,
not run state.

Multipliers applied in Player.takeDamage:
  - Easy: 150 HP / 75 ammo start; enemy damage ×0.7
  - Normal: 100 HP / 50 ammo (baseline)
  - Hard: 75 HP / 50 ammo start; enemy damage ×1.4

SkillLevelSelect component renders on first boot per LOOMGame mount
(before BootSequence). After pick, transitions normally."
```

---

## Task 6.7: Termination Letter consumable

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/terminationLetter.ts` (the consumable item, not a projectile despite the name — see naming note below)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts` (add `'termination_letter'` ThingType)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts` (pickup tracking, room-clear effect, key-bind 'T' to use)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/input.ts` (KeyT consumes one Termination Letter from inventory)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/player.ts` (terminationLetters: number)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/LoomHud.tsx` (display TL count)
- Modify: 3 map files (cyc2_glass_confroom_boss, cyc3_wellness_program, cyc4_void_2) to add TL pickups

(Naming note: the file lives in `projectiles/` for path consistency with other ThingType-paired items — but it's a static pickup, not a projectile.)

**Termination Letter mechanic:**
- Picked up by walking within 0.6 tiles of a `'termination_letter'` ThingType (1 per cycle from C2 onward; 3 total per playthrough)
- Adds 1 to `Player.terminationLetters` count
- Pressing 'T' (when count > 0) consumes one and triggers a 6-tile-radius damage-everything pulse: every alive enemy within 6 tiles of the player takes 999 damage (one-shot kill, even bosses... actually, NOT bosses — gate the room-clear on `!isBoss`. Bosses are immune to TL.)
- HUD shows "TL: N" next to ammo/health

- [ ] **Step 1: TerminationLetter pickup data + tests**

(Static-thing — no projectile interface needed; just position tracking in GameLoop similar to TheHum pickup pattern.)

- [ ] **Step 2: Player.terminationLetters + Player methods**

Add field, increment on pickup, expose `useTerminationLetter(): boolean` (returns true if count > 0 and decrements; false if 0).

- [ ] **Step 3: GameLoop pickup proximity + 'T' keybind**

Mirror The Hum pickup pattern but tracking multiple positions (array of pickup `{x, y, used: boolean}`). On 'T' keypress + count > 0: iterate enemies within 6 tiles; if `!isBoss && state !== 'dead'`, call takeDamage(999).

- [ ] **Step 4: HUD display**

Append "tl: N" to the bio/queue/active line when N > 0.

- [ ] **Step 5: Add pickups to 3 maps**

`cyc2_glass_confroom_boss.json`: add `{ "type": "termination_letter", "x": 4, "y": 6 }` (away from VP boss spawn — early-room pickup).
`cyc3_wellness_program.json`: add `{ "type": "termination_letter", "x": 6, "y": 7 }`.
`cyc4_void_2.json`: add `{ "type": "termination_letter", "x": 5, "y": 7 }`.

- [ ] **Step 6: Commit**

```bash
git commit -m "Add Termination Letter consumable (Phase 6)

Rare-drop room-clear consumable per design spec §5.1. 3 instances
distributed: cyc2_glass_confroom_boss, cyc3_wellness_program,
cyc4_void_2 — one per cycle from C2 onward.

Mechanic:
  - Walk within 0.6 tiles of a 'termination_letter' pickup → +1 to
    Player.terminationLetters
  - Press 'T' (when count > 0) → consume 1 + room-clear: every alive
    enemy within 6 tiles takes 999 damage. Bosses immune (isBoss=true).

HUD shows 'tl: N' inline with bio/queue/active when count > 0.

Diegetically: a Termination Letter handed in the right cubicle clears
the room of corporate hostiles. Bosses are too entrenched in the
hierarchy to be terminated by a single letter."
```

---

## Task 6.8: True hold-fire The Hum + predictive aim The Algorithm

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/theHum.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/theHum.test.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts` (call theHum.fire on every frame while button held, not just on press)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/input.ts` (expose `isPrimaryFireDown(): boolean` accessor)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/theAlgorithm.ts` (predictive aim)

**The Hum hold-fire:** Currently fires once per click. Spec calls for sustained beam. Add `isPrimaryFireDown()` to InputController that returns the current mouse-button state (separate from the click-edge `consumeFirePrimary()`). In the GameLoop tick, if active weapon is The Hum AND mouse-button is currently held: call `theHum.fire(...)` every frame. Other weapons keep using consumeFirePrimary (single-shot semantics).

**The Algorithm predictive aim:** Track player position deltas frame-over-frame (estimate velocity). When firing, aim at `(player.x + estimatedDx * LEAD_FACTOR, player.y + estimatedDy * LEAD_FACTOR)` instead of current position. `LEAD_FACTOR ≈ 0.3` seconds (so the projectile arrives roughly where the player will be in 0.3s). For fast-moving players the spread misses; for stationary ones the burst still lands.

- [ ] **Step 1: Hum hold-fire**

```typescript
// in InputController:
isPrimaryFireDown(): boolean {
  return this.primaryFireHeld;
}

// the existing fireDown/firePressed mechanics — add primaryFireHeld
// updated on mousedown/mouseup events.
```

In gameLoop tick:
```typescript
const activeWeapon = this.player.getActiveWeapon();
if (activeWeapon instanceof TheHum && this.input.isPrimaryFireDown()) {
  activeWeapon.fire(this.player, this.map, this.enemies, this.audio, ...);
}
```

(Other weapons keep using `consumeFirePrimary()` for single-shot edge-triggering.)

- [ ] **Step 2: Algorithm predictive aim**

```typescript
private prevPlayerX: number | null = null;
private prevPlayerY: number | null = null;

// in update(), before firing:
if (this.prevPlayerX !== null && this.prevPlayerY !== null) {
  const vx = (ctx.player.x - this.prevPlayerX) / dt;
  const vy = (ctx.player.y - this.prevPlayerY) / dt;
  const leadX = ctx.player.x + vx * LEAD_FACTOR;
  const leadY = ctx.player.y + vy * LEAD_FACTOR;
  const angleToPlayer = Math.atan2(leadY - this.y, leadX - this.x);
  // ... fire 3 projectiles at angleToPlayer ± spread
}
this.prevPlayerX = ctx.player.x;
this.prevPlayerY = ctx.player.y;
```

- [ ] **Step 3: Tests**

Hum: extend test that asserts hold-fire ticks accumulate damage over multiple `fire()` calls (already covered — the existing pierce test asserts each call deals damage). Add a test confirming the gameLoop calls fire on consecutive frames when isPrimaryFireDown is true (mock the input + assert fire-call count over N frames).

Algorithm: extend test asserting that with `prevPlayerX/Y === null` (first frame), aim is at current position. With deltas, aim leads.

- [ ] **Step 4: Commit**

```bash
git commit -m "Hum hold-fire + Algorithm predictive aim (Phase 6 deferrals)

The Hum (slot 8) — true sustained beam:
  - InputController exposes isPrimaryFireDown() (mouse-button held
    state, separate from consumeFirePrimary edge-trigger)
  - GameLoop tick: if active weapon is TheHum AND mouse held, call
    fire() every frame. Other weapons keep consumeFirePrimary
    single-shot semantics.
  - Result: holding the fire button now drains 4 dmg/frame from every
    alive enemy on the aim ray, ~240 dps at 60fps as spec'd.

The Algorithm — predictive aim per spec §5.2:
  - Tracks player position frame-over-frame; estimates (vx, vy)
    via dt-normalized delta
  - LEAD_FACTOR=0.3 sec: aim shifts to player + velocity × 0.3
  - First frame (no prev pos): aim at current position (no lead)
  - Strafing players harder to hit; stationary players still get
    the full 3-projectile burst.

Both items resolve Phase 5 plan-documented deferrals."
```

---

## Task 6.9: 60Hz hum intensity ramp by cycle (audio code-side)

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/audioController.ts` (setHumIntensity + onMapLoad call)
- Modify: `/Users/justinwest/Repos/l0b0tonline/services/soundService.ts` (verify hum noise channel exists or add small wrapper)

The 60Hz hum is part of soundService's existing ambient noise channel. Phase 6 adds a per-cycle intensity ramp — Cycle 1 = 0.4 (subtle), Cycle 2 = 0.6, Cycle 3 = 0.8, Cycle 4 = 1.0 (full).

The audioController already calls `onMapLoad(mapId)` on each transition. Add a helper that reads `getCycleOfMap(mapId)` and calls a new `soundService.setHumIntensity(level: 0..1)` (or similar — adapt to whatever the existing API exposes).

If soundService doesn't have a hum channel knob, the simplest hack: tune the ambient gain via `gainNode.gain.value` on the existing ambient node. Phase 7 can replace with proper per-track mix automation.

- [ ] **Step 1: Search soundService for ambient/hum methods**

```bash
grep -n "ambient\|gain\|hum\|noise" /Users/justinwest/Repos/l0b0tonline/services/soundService.ts | head -20
```

Identify the API surface. If a `setAmbientGain(0..1)` method exists, use it. If not, add one or extend an existing method.

- [ ] **Step 2: AudioController.setHumIntensity**

```typescript
setHumIntensity(level: number) {
  // Map cycle-derived 0..1 to the ambient gain knob (whatever soundService exposes).
  soundService.setAmbientGain?.(level * 0.5); // Cap at 0.5 — full hum might be obnoxious; tune in Phase 7.
}

onMapLoad(mapId: string) {
  // ... existing isBossArena logic
  const cycle = getCycleOfMap(mapId) ?? 1;
  this.setHumIntensity(0.4 + 0.2 * (cycle - 1)); // 0.4, 0.6, 0.8, 1.0
}
```

- [ ] **Step 3: Commit**

```bash
git commit -m "60Hz hum intensity ramps per cycle (Phase 6 audio code-side)

audioController.setHumIntensity(level) takes a 0..1 cycle-mapped
amplitude. onMapLoad reads getCycleOfMap and applies the ramp:
  - Cycle 1: 0.4 (subtle)
  - Cycle 2: 0.6
  - Cycle 3: 0.8
  - Cycle 4: 1.0 (full)

Wraps soundService's ambient gain knob (or adds a simple wrapper if
absent). Phase 7 replaces with proper per-track mix automation when
the real Patch Tuesday + human In l00p tracks land.

Spec §6.2: 'The 60Hz hum becomes audible under everything from C2
onward, growing across cycles.' This implementation grows it across
all 4 cycles starting subtle in C1; tune the start-point during Phase
7 audio mix balance work."
```

---

## Task 6.10: Phase 6 e2e Playwright spec (replaces Phase 5 spec)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/e2e/loom-phase-6.spec.ts`
- Delete: `/Users/justinwest/Repos/l0b0tonline/e2e/loom-phase-5.spec.ts`

Phase 6 spec extends Phase 5 with:
- SkillLevelSelect: select 'normal' (mirrors Phase 5 baseline)
- Termination Letter pickup test in cyc2_glass_confroom_boss (teleport to pickup; verify TL count = 1; press 'T'; verify enemies in radius dead)
- Reduced-motion media query mock; verify body has `data-reduced-motion="true"` or equivalent class
- The Hum hold-fire test (single-press fire + hold-fire = different damage totals over N frames)

Same 18-map walkthrough + boss kills + C-reveal as Phase 5.

- [ ] **Step 1: Read Phase 5 spec for hooks + boot pattern**

- [ ] **Step 2: Write Phase 6 spec** with the 4 new assertions interleaved at the appropriate cycle points

- [ ] **Step 3: Delete Phase 5 spec**

- [ ] **Step 4: Run + commit**

```bash
git commit -m "Add Phase 6 e2e Playwright spec (replaces Phase 5)

Extends Phase 5 spec with:
  - SkillLevelSelect: pick 'normal' before boot to mirror Phase 5
    baseline behavior
  - Termination Letter pickup on cyc2_glass_confroom_boss; press 'T';
    verify enemies cleared (except boss VP of Sales)
  - Reduced-motion media query mocked; assert body class reflects state
  - The Hum hold-fire: enemy HP delta over N frames > single-press
    delta

Phase 5 spec deleted — Phase 6 is a strict superset.

Runtime: ~50s (Phase 5 baseline ~46s + the 4 new assertions add ~4s).
test.setTimeout(180_000) provides ample headroom."
```

---

# Phase 6 final review checklist

After all 10 tasks complete:
- [ ] Spec coverage: every Phase 6 spec line item from `2026-04-27-loom-doom-design.md` §8 is implemented OR explicitly documented as deferred to Phase 7 (real music, real sprite art, surveillance log voiceover, mobile responsiveness, final difficulty tuning).
- [ ] Phase 5 review carry-overs M-1, M-3, M-5, M-6 all resolved.
- [ ] Test gate: tsc clean; vitest 850+/850+ pass; Phase 6 e2e PASSES.
- [ ] Manual creative gate: Phase 6 playtest log captures the 4 NEEDS-HUMAN items + verifies prior-phase regressions absent.
- [ ] Code review: dispatch a `feature-dev:code-reviewer` against the full Phase 6 commit set.
- [ ] PR: create PR for the Phase 6 changeset on l0b0tonline + a parallel PR in LOOM-DOOM for the plan + playtest log.
