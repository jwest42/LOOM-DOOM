# LOOM Phase 2 Implementation Plan — Cycle 1 Vertical Slice

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the **Cycle 1 vertical slice** — the first publishable demo. New player can boot LOOM, play through 3 hand-authored maps (lobby → cubicles → HR arena), kill the HR Manager boss, see Cycle 1 ending text, hit a "Cycle 2 coming soon" transition stub.

**Architecture:** Builds on the Phase 0+1 tracer-bullet engine. Adds: a polymorphic `IEnemy` base, weapon-switching, map progression with exit triggers, an intermission screen, a basic projectile system, and the `soundService` integration. New content: 3 enemies (Account Executive, Recruiter, HR Manager), 3 weapons (Drum Stick, Industrial Shredder, Spam Filter), 3 maps, the *Ship It* JOIN OS track on the HR arena.

**Tech Stack:** Same as Phase 1 — TypeScript 5.8, React 19, Vite 6, Tailwind v4, Zustand 5, WebGL2, Web Audio API (via `services/soundService.ts`), Vitest, Playwright. No new dependencies.

**Pre-existing state:**
- Implementation lives at `/Users/justinwest/Repos/l0b0tonline` on branch `loom-game`. Phase 0+1 was merged via PR #98; the HUD-fix follow-up is PR #99 (assume merged before this plan starts; if not, rebase).
- Phase 1 left these files in place: `components/LOOMGame.tsx`, `components/loom/{engine,entities,hud,store,types.ts}`, `public/data/loom/maps/cyc1_test.json`, `public/data/loom/sprites/intern_idle.png`, two Playwright specs.
- Phase 1 final-review punch list still has a few items the spec said should land at Phase 2 setup: **I8 (extract Enemy base from Intern)** and **I1 (stabilize `getSnapshot` identity)**. They're Tasks 2.1 and 2.2 here.
- **Where commands run:** Most tasks operate in `/Users/justinwest/Repos/l0b0tonline`. Plan + playtest log commits go in this LOOM-DOOM worktree. Each task notes its working directory.

---

## File structure (post-Phase-2, in l0b0tonline)

New files (NEW), modified files (MOD):

```
l0b0tonline/
├── components/
│   ├── LOOMGame.tsx                                            (MOD — game-state machine)
│   └── loom/
│       ├── engine/
│       │   ├── audioController.ts                              (NEW — wraps soundService)
│       │   ├── campaign.ts                                     (NEW — map sequence + progression)
│       │   ├── gameLoop.ts                                     (MOD — weapon registry, exit detection, audio hooks)
│       │   ├── input.ts                                        (MOD — number-key weapon-select)
│       │   └── renderer.ts                                     (MOD — IEnemy[] not Intern[])
│       ├── entities/
│       │   ├── enemies/
│       │   │   ├── Enemy.ts                                    (NEW — IEnemy interface + EnemyState)
│       │   │   ├── intern.ts                                   (MOD — implement IEnemy + add melee attack)
│       │   │   ├── accountExecutive.ts                         (NEW)
│       │   │   ├── recruiter.ts                                (NEW)
│       │   │   └── hrManager.ts                                (NEW)
│       │   ├── projectiles/
│       │   │   ├── Projectile.ts                               (NEW — base projectile interface)
│       │   │   ├── recruiterFireball.ts                        (NEW)
│       │   │   └── documentationRequest.ts                     (NEW — HR Manager's tracking projectile)
│       │   ├── weapons/
│       │   │   ├── IWeapon.ts                                  (MOD — IEnemy[] not Intern[])
│       │   │   ├── brandedPen.ts                               (MOD — IEnemy[])
│       │   │   ├── neuralPulse.ts                              (MOD — IEnemy[])
│       │   │   ├── drumStick.ts                                (NEW — slot 1 melee with rhythm combo)
│       │   │   ├── industrialShredder.ts                       (NEW — slot 1 alt melee)
│       │   │   └── spamFilter.ts                               (NEW — slot 3 spread shotgun)
│       │   └── player.ts                                       (MOD — exit-trigger detection, take-damage helper)
│       └── hud/
│           ├── IntermissionScreen.tsx                          (NEW — between-map text + continue)
│           ├── LoomHud.tsx                                     (MOD — controls hint adds 1/2/3 keys)
│           └── EndOfCycleStub.tsx                              (NEW — "Cycle 2 coming soon" placeholder)
└── public/
    └── data/
        └── loom/
            ├── maps/
            │   ├── cyc1_lobby.json                             (NEW — Phase 2 map 1, J0IN 0S boot)
            │   ├── cyc1_cubicles.json                          (NEW — Phase 2 map 2)
            │   └── cyc1_hr_arena.json                          (NEW — Phase 2 map 3, HR Manager boss)
            └── sprites/
                ├── account_executive_idle.png                  (NEW — placeholder)
                ├── recruiter_idle.png                          (NEW — placeholder)
                ├── hr_manager_idle.png                         (NEW — placeholder)
                └── recruiter_fireball.png                      (NEW — placeholder)
```

Tests:
- `components/loom/engine/campaign.test.ts` (NEW)
- `components/loom/entities/enemies/intern.test.ts` (NEW — minimal IEnemy conformance + melee attack)
- `e2e/loom-phase-2.spec.ts` (NEW — Phase 2 DoD smoke)

---

# Phase 2 Definition of Done

A new player opens l0b0tonline, runs `loom.exe`, watches the boot sequence, and:
1. Spawns into `cyc1_lobby` (the first real map) with WASD + mouse-look + Branded Pen + Neural Pulse working
2. Encounters Interns, Account Executives, Recruiters; can switch weapons via 1/2/3 keys; can kill enemies; takes damage from melee + projectiles
3. Reaches an exit tile, sees an intermission screen with the lobby's intermission text, presses any key to continue
4. Spawns into `cyc1_cubicles`; same loop; reaches exit; intermission
5. Spawns into `cyc1_hr_arena`; *Ship It* music starts; fights the HR Manager boss alongside lower-tier enemies; kills the boss
6. Sees the Cycle 2 transition stub (a `EndOfCycleStub` screen with "CYCLE 1 COMPLETE" + "CYCLE 2 — COMING SOON" + the manifesto opening lyric)

**Automation gate:** Playwright spec verifies map progression (cyc1_lobby → cyc1_cubicles transition) without crashes; vitest covers Enemy interface conformance, campaign progression logic, weapon-switching state machine. Manual sanity items: visual map design quality, music timing on HR arena, boss combat balance.

---

# Tasks

## Task 2.1: Extract `IEnemy` interface from `Intern`

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/Enemy.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/intern.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/IWeapon.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/brandedPen.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/neuralPulse.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/renderer.ts`

This is a refactor — Phase 1 left `IWeapon.fire` and the renderer parameterized on the concrete `Intern` class. Phase 2 adds 3 more enemy types so we extract a base interface now (cheap with 1 type, expensive with 4).

- [ ] **Step 1: Create the `IEnemy` interface**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/Enemy.ts`:
```typescript
import type { Player } from '../player';
import type { LoomMap } from '../../types';

/** Common enemy lifecycle states. */
export type EnemyState = 'idle' | 'chase' | 'pain' | 'dead';

/**
 * IEnemy — common interface for all LOOM enemies.
 * Implementations live under entities/enemies/<name>.ts.
 *
 * The renderer billboards them; weapons resolve hits against them; the
 * GameLoop ticks them each frame.
 */
export interface IEnemy {
  x: number;
  y: number;
  health: number;
  state: EnemyState;
  /** Sprite filename (relative to /data/loom/sprites/, no extension). */
  readonly spriteName: string;
  /** Sprite color tint as RGB triplet (Phase 2: still flat-rect placeholder). */
  readonly placeholderColor: readonly [number, number, number];

  update(dt: number, player: Player, map: LoomMap): void;
  takeDamage(damage: number): void;
}
```

- [ ] **Step 2: Update `Intern` to implement `IEnemy`**

Replace `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/intern.ts` with:
```typescript
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import type { IEnemy, EnemyState } from './Enemy';

const HEALTH = 20;
const SPEED = 1.0;
const SIGHT_RANGE = 8;
const MELEE_RANGE = 1.0;
const MELEE_DAMAGE = 5;
const MELEE_COOLDOWN_MS = 800;

export class Intern implements IEnemy {
  x: number;
  y: number;
  health = HEALTH;
  state: EnemyState = 'idle';
  painUntil = 0;
  private nextMeleeAt = 0;

  readonly spriteName = 'intern_idle';
  readonly placeholderColor = [100, 140, 80] as const;

  constructor(x: number, y: number) {
    this.x = x;
    this.y = y;
  }

  update(dt: number, player: Player, _map: LoomMap) {
    if (this.state === 'dead') return;

    if (this.state === 'pain' && performance.now() >= this.painUntil) {
      this.state = 'chase';
    }

    const dx = player.x - this.x;
    const dy = player.y - this.y;
    const dist = Math.hypot(dx, dy);

    if (this.state === 'idle' && dist < SIGHT_RANGE) {
      this.state = 'chase';
    }

    if (this.state === 'chase') {
      // Melee attack when close (Phase 1 gap: Interns previously couldn't damage the player)
      if (dist <= MELEE_RANGE && performance.now() >= this.nextMeleeAt) {
        player.takeDamage(MELEE_DAMAGE);
        this.nextMeleeAt = performance.now() + MELEE_COOLDOWN_MS;
        return;
      }
      // Otherwise chase
      const move = SPEED * dt;
      if (dist > 0.5) {
        this.x += (dx / dist) * move;
        this.y += (dy / dist) * move;
      }
    }
  }

  takeDamage(damage: number) {
    if (this.state === 'dead') return;
    this.health -= damage;
    if (this.health <= 0) {
      this.health = 0;
      this.state = 'dead';
      return;
    }
    this.state = 'pain';
    this.painUntil = performance.now() + 200;
  }
}
```

This adds:
- `IEnemy` conformance (the `spriteName` + `placeholderColor` fields are new)
- A working melee attack (Phase 1 gap fix — previously Interns couldn't damage the player)
- A `nextMeleeAt` cooldown so Interns don't tick-spam-damage every frame

`player.takeDamage()` is referenced and added in Task 2.3.

- [ ] **Step 3: Update `IWeapon` interface**

Replace `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/IWeapon.ts` with:
```typescript
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import type { IEnemy } from '../enemies/Enemy';

/** IWeapon — common interface for held weapons (slot 1+). */
export interface IWeapon {
  readonly name: string;
  readonly slot: number;
  fire(player: Player, map: LoomMap, enemies: IEnemy[]): boolean;
}
```

- [ ] **Step 4: Update `BrandedPen` and `NeuralPulse` signatures**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/brandedPen.ts`. Find the import line for `Intern` and replace:
```typescript
import type { Intern } from '../enemies/intern';
```
with:
```typescript
import type { IEnemy } from '../enemies/Enemy';
```

Then change the `fire` signature:
```typescript
fire(player: Player, map: LoomMap, enemies: IEnemy[]): boolean {
```

Find the `closest` typed declaration `let closest: { enemy: Intern; dist: number } | null = null;` and change `Intern` → `IEnemy`.

Apply the same swap to `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/neuralPulse.ts`.

- [ ] **Step 5: Update `GameLoop` and `Renderer` signatures**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`. Replace:
```typescript
import { Intern } from '../entities/enemies/intern';
```
with:
```typescript
import { Intern } from '../entities/enemies/intern';
import type { IEnemy } from '../entities/enemies/Enemy';
```

Change `private enemies: Intern[] = [];` to `private enemies: IEnemy[] = [];`. The `for (const t of map.things)` loop that constructs Interns stays; we just store them as `IEnemy`.

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/renderer.ts`. Replace `import type { Intern } from '../entities/enemies/intern';` with:
```typescript
import type { IEnemy } from '../entities/enemies/Enemy';
```

Change the `render` method's `enemies: Intern[]` parameter to `enemies: IEnemy[]`. Update the sprite-billboarding loop's color line — replace the hard-coded `(100, 140, 80)` with `e.placeholderColor[0]` etc.:
```typescript
this.framebuffer[idx]     = e.placeholderColor[0];
this.framebuffer[idx + 1] = e.placeholderColor[1];
this.framebuffer[idx + 2] = e.placeholderColor[2];
```

- [ ] **Step 6: Type-check, run tests**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -10
npx vitest run components/loom 2>&1 | tail -8
```
Expected: tsc clean; vitest 9/9 still passing (no new tests yet — Task 2.2 adds them).

- [ ] **Step 7: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/entities/enemies/Enemy.ts \
        components/loom/entities/enemies/intern.ts \
        components/loom/entities/weapons/IWeapon.ts \
        components/loom/entities/weapons/brandedPen.ts \
        components/loom/entities/weapons/neuralPulse.ts \
        components/loom/engine/gameLoop.ts \
        components/loom/engine/renderer.ts
git commit -m "Extract IEnemy interface from Intern + add melee attack

Refactors weapons + renderer + game loop to operate on IEnemy[] instead
of Intern[]. Phase 1 left these parameterized on the concrete class
because Intern was the only enemy. Phase 2 adds 3 more — cheap to
extract now, expensive after.

Also adds a working melee attack to Intern (Phase 1 gap — Interns
chased but couldn't damage the player). Cooldown 800ms; damage 5;
range 1.0 tiles. Player.takeDamage() is added in Task 2.3."
```

---

## Task 2.2: Stabilize `getSnapshot` identity + add `Player.takeDamage`

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/LOOMGame.tsx`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/player.ts`

This pulls in two small Phase 1 review items that block Task 2.1's compile (the `player.takeDamage()` call) and prevent unnecessary effect re-runs in `LoomHud`.

- [ ] **Step 1: Add `takeDamage` to `Player`**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/player.ts`. Add a method (anywhere among the public methods):
```typescript
takeDamage(damage: number) {
  this.health = Math.max(0, this.health - damage);
}
```

(No death state for now — health 0 just means "very low"; the gameLoop will surface a `playerDead` state in a later task. For Phase 2 DoD, it's fine if the player is invincible at 0 HP — the manual playtest just verifies that taking damage decrements the bio reading.)

- [ ] **Step 2: Stabilize `getSnapshot` callback identity**

Edit `/Users/justinwest/Repos/l0b0tonline/components/LOOMGame.tsx`. Add `useCallback` to the imports:
```typescript
import { useCallback, useEffect, useRef, useState } from 'react';
```

Just before the `return` statement, add:
```typescript
const getSnapshot = useCallback(() => loopRef.current!.getSnapshot(), []);
```

Then change the `<LoomHud getSnapshot={() => loopRef.current!.getSnapshot()} />` line to:
```typescript
{loopReady && loopRef.current && <LoomHud getSnapshot={getSnapshot} />}
```

(The empty deps array is correct — `loopRef.current` is read at call time, not capture time, and the ref itself is stable across renders.)

- [ ] **Step 3: Type-check + run tests**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -10
npx vitest run components/loom 2>&1 | tail -5
npm run test:e2e -- e2e/loom-phase-1.spec.ts 2>&1 | tail -10
```
Expected: tsc clean, vitest 9/9, Playwright Phase 1 spec PASS.

- [ ] **Step 4: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/entities/player.ts components/LOOMGame.tsx
git commit -m "Add Player.takeDamage + stabilize getSnapshot identity

Player.takeDamage(n) clamps health at 0; called from Intern's new
melee attack and from upcoming projectile hits.

Wraps LOOMGame's getSnapshot prop in useCallback so its identity is
stable across LOOMGame re-renders, preventing unnecessary effect
re-runs in LoomHud. (Per Phase 1 final review item I1.)"
```

---

## Task 2.3: Weapon-switching system (1/2/3 keys + slot cycling)

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/input.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/player.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/LoomHud.tsx`

DOOM-style weapon switching: number key N selects slot N; pressing N again cycles between weapons in that slot. Phase 2 has slot 1 (Drum Stick / Industrial Shredder), slot 2 (Branded Pen), slot 3 (Spam Filter — only one weapon for now). Slot 0 (Neural Pulse) is the always-on F-key weapon and isn't part of this switching.

- [ ] **Step 1: Add weapon-select tracking to `InputController`**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/input.ts`. Add a private field:
```typescript
private weaponSelectQueue: number[] = [];
```

In the `onKeyDown` callback, just after the existing `if (e.code === 'KeyF') this.firePulse = true;` line, add:
```typescript
if (e.code === 'Digit1') this.weaponSelectQueue.push(1);
else if (e.code === 'Digit2') this.weaponSelectQueue.push(2);
else if (e.code === 'Digit3') this.weaponSelectQueue.push(3);
```

Add a public method:
```typescript
consumeWeaponSelect(): number | null {
  return this.weaponSelectQueue.shift() ?? null;
}
```

- [ ] **Step 2: Add weapon registry + switching to `GameLoop`**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`. Replace the `private brandedPen` field with a slot-keyed weapon registry. After the existing imports, the relevant section becomes:

```typescript
// (existing imports + BrandedPen + NeuralPulse + Intern + Renderer + InputController + Player)

import type { IWeapon } from '../entities/weapons/IWeapon';

// Inside the GameLoop class:
private neuralPulse = new NeuralPulse();

/** Weapons keyed by slot. Each slot holds an array (DOOM-style — pressing
 *  the slot key cycles within the slot). Phase 2: slot 1 has Drum Stick +
 *  Industrial Shredder; slot 2 has Branded Pen; slot 3 has Spam Filter. */
private weaponsBySlot: Map<number, IWeapon[]> = new Map();

/** Index of the currently-active weapon within the active slot (per slot,
 *  tracks the last-used choice when switching back). */
private slotIndex: Map<number, number> = new Map();
```

In the constructor, after `this.player = new Player(...)`, add:
```typescript
// Default loadout. Drum Stick + Industrial Shredder added in their
// own tasks; for now slot 1 is empty until Task 2.8.
this.weaponsBySlot.set(2, [new BrandedPen()]);
this.slotIndex.set(1, 0);
this.slotIndex.set(2, 0);
this.slotIndex.set(3, 0);
this.player.activeSlot = 2; // Branded Pen at start
```

Add a private method:
```typescript
private trySelectSlot(slot: number) {
  const weapons = this.weaponsBySlot.get(slot);
  if (!weapons || weapons.length === 0) return; // empty slot — no-op
  if (this.player.activeSlot === slot) {
    // Cycle within slot
    const i = (this.slotIndex.get(slot) ?? 0);
    this.slotIndex.set(slot, (i + 1) % weapons.length);
  } else {
    // Switch to slot, keep its remembered index
    this.player.activeSlot = slot;
  }
}

private getActiveWeapon(): IWeapon | null {
  const slot = this.player.activeSlot;
  const weapons = this.weaponsBySlot.get(slot);
  if (!weapons || weapons.length === 0) return null;
  const i = this.slotIndex.get(slot) ?? 0;
  return weapons[i];
}
```

In the tick function, replace the `if (this.input.consumeFirePrimary()) { this.brandedPen.fire(this.player, this.map, this.enemies); }` with:
```typescript
const select = this.input.consumeWeaponSelect();
if (select !== null) this.trySelectSlot(select);

if (this.input.consumeFirePrimary()) {
  this.getActiveWeapon()?.fire(this.player, this.map, this.enemies);
}
```

(Remove the `private brandedPen = new BrandedPen();` field — it's now in the registry.)

- [ ] **Step 3: Update `Player` to expose `activeSlot` mutably**

The `activeSlot` field already exists; it's already mutable. No change needed beyond verifying it.

- [ ] **Step 4: Update HUD controls hint**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/LoomHud.tsx`. In the controls badge JSX, add:
```jsx
<div><span className="text-green-500">1/2/3</span> select weapon</div>
```
between the `MOUSE` and `CLICK` lines.

- [ ] **Step 5: Type-check + run tests + smoke**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -10
npx vitest run components/loom 2>&1 | tail -5
(npm run dev 2>&1 | head -15) & DEV_PID=$!
sleep 6
kill $DEV_PID 2>/dev/null
wait 2>/dev/null
```
Expected: tsc clean, vitest 9/9, dev server boots.

- [ ] **Step 6: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/engine/input.ts \
        components/loom/engine/gameLoop.ts \
        components/loom/hud/LoomHud.tsx
git commit -m "Add DOOM-style weapon switching (1/2/3 keys + slot cycling)

InputController queues number-key presses; GameLoop maintains a
weaponsBySlot map and a per-slot activeIndex. Pressing a slot key
either switches to that slot or cycles within it (DOOM behavior).

Slot 2 starts with Branded Pen; slots 1 and 3 are empty until their
weapons are added in Tasks 2.8-2.10. The HUD's controls badge now
hints 1/2/3 select weapon."
```

---

## Task 2.4: Map exit triggers + campaign progression

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/campaign.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/campaign.test.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

The placeholder `cyc1_test` map has a single `exit` thing type that's never been wired up. Phase 2 needs map progression: walking onto an exit thing transitions to the next map in the campaign, or shows the cycle-end stub if it's the last map.

- [ ] **Step 1: Write `campaign.test.ts`**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/campaign.test.ts`:
```typescript
import { describe, it, expect } from 'vitest';
import { CYCLE_1_MAPS, getNextMapId, isLastMapOfCycle } from './campaign';

describe('campaign', () => {
  it('CYCLE_1_MAPS lists 3 maps in order', () => {
    expect(CYCLE_1_MAPS).toEqual(['cyc1_lobby', 'cyc1_cubicles', 'cyc1_hr_arena']);
  });

  it('getNextMapId returns the next map in cycle 1', () => {
    expect(getNextMapId('cyc1_lobby')).toBe('cyc1_cubicles');
    expect(getNextMapId('cyc1_cubicles')).toBe('cyc1_hr_arena');
  });

  it('getNextMapId returns null after the final cycle 1 map', () => {
    expect(getNextMapId('cyc1_hr_arena')).toBeNull();
  });

  it('isLastMapOfCycle detects the final map', () => {
    expect(isLastMapOfCycle('cyc1_lobby')).toBe(false);
    expect(isLastMapOfCycle('cyc1_cubicles')).toBe(false);
    expect(isLastMapOfCycle('cyc1_hr_arena')).toBe(true);
  });

  it('returns null for unknown map ids', () => {
    expect(getNextMapId('cyc99_unknown')).toBeNull();
    expect(isLastMapOfCycle('cyc99_unknown')).toBe(false);
  });
});
```

- [ ] **Step 2: Run test, verify it fails**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/engine/campaign.test.ts 2>&1 | tail -10
```
Expected: FAIL — `Cannot find module './campaign'`.

- [ ] **Step 3: Implement `campaign.ts`**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/campaign.ts`:
```typescript
/**
 * Campaign progression — what map comes after which, and where each cycle ends.
 *
 * Phase 2 ships Cycle 1 only (3 maps). Cycles 2-4 are added in later phases.
 */

export const CYCLE_1_MAPS = ['cyc1_lobby', 'cyc1_cubicles', 'cyc1_hr_arena'] as const;

export type CycleMapId = typeof CYCLE_1_MAPS[number];

const ALL_CYCLE_MAPS: readonly string[] = [...CYCLE_1_MAPS];
// Phase 3+: append CYCLE_2_MAPS, CYCLE_3_MAPS, CYCLE_4_MAPS here.

const CYCLE_BOUNDARIES: ReadonlyMap<string, true> = new Map([
  [CYCLE_1_MAPS[CYCLE_1_MAPS.length - 1], true],
  // Phase 3+: also push the last map of cycle 2, 3, 4.
]);

/** Returns the id of the map that follows `currentId`, or null at cycle end. */
export function getNextMapId(currentId: string): string | null {
  const idx = ALL_CYCLE_MAPS.indexOf(currentId);
  if (idx < 0) return null;
  if (CYCLE_BOUNDARIES.has(currentId)) return null;
  return ALL_CYCLE_MAPS[idx + 1] ?? null;
}

/** True if the given map id is the last of its cycle (cycle-end transition). */
export function isLastMapOfCycle(currentId: string): boolean {
  return CYCLE_BOUNDARIES.has(currentId);
}
```

- [ ] **Step 4: Run test, verify it passes**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/engine/campaign.test.ts 2>&1 | tail -10
```
Expected: 5/5 PASS.

- [ ] **Step 5: Wire exit detection into the game loop**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`. Add at the top:
```typescript
import { getNextMapId, isLastMapOfCycle } from './campaign';
```

Add an "exit reached" callback hook, plumbed through the constructor:
```typescript
// New constructor parameter:
constructor(
  canvas: HTMLCanvasElement,
  map: LoomMap,
  private onMapExit: (next: { kind: 'next-map'; id: string } | { kind: 'cycle-end' }) => void,
) {
  // ... existing constructor body ...
}
```

Add a private flag to prevent re-firing the exit on every frame:
```typescript
private exitFired = false;
```

In the tick function, after `for (const e of this.enemies) e.update(...)`, add:
```typescript
// Exit-tile detection: if the player is within 0.6 tiles of any exit thing, fire onMapExit once.
if (!this.exitFired) {
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
```

The `LOOMGame.tsx` integration follows in Task 2.5 (intermission state machine).

- [ ] **Step 6: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -10
git add components/loom/engine/campaign.ts \
        components/loom/engine/campaign.test.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add campaign progression + exit-tile detection

components/loom/engine/campaign.ts holds CYCLE_1_MAPS sequence and
getNextMapId/isLastMapOfCycle helpers (5 vitest tests).

GameLoop now takes an onMapExit callback; each tick checks if the
player is near an exit thing and fires the callback exactly once.
LOOMGame's state machine handling lands in Task 2.5."
```

(TSC will surface that `LOOMGame.tsx` doesn't yet pass the new constructor arg — that's fine, Task 2.5 fixes it. If you want a clean intermediate state, temporarily make the constructor parameter optional with `onMapExit: (...) => void = () => {}` and remove the default in Task 2.5.)

---

## Task 2.5: Intermission screen + game-state machine

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/IntermissionScreen.tsx`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/EndOfCycleStub.tsx`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/LOOMGame.tsx`

Wires `LOOMGame` into a state machine: `boot → playing → intermission → playing → ... → cycle-end-stub → playing-cyc2 (someday)`. Renders the appropriate React element for each state.

- [ ] **Step 1: Create `IntermissionScreen` component**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/IntermissionScreen.tsx`:
```tsx
import { useEffect } from 'react';

interface IntermissionScreenProps {
  /** Text shown to the player (from the previous map's intermissionText). */
  text: string;
  /** Map id being entered next, displayed as a header. */
  nextMapId: string;
  /** Called when the player presses any key. */
  onContinue: () => void;
}

export function IntermissionScreen({ text, nextMapId, onContinue }: IntermissionScreenProps) {
  useEffect(() => {
    const handler = (_e: KeyboardEvent) => onContinue();
    window.addEventListener('keydown', handler);
    return () => window.removeEventListener('keydown', handler);
  }, [onContinue]);

  return (
    <div className="absolute inset-0 z-50 flex flex-col items-center justify-center bg-black font-mono text-green-400">
      <div className="mb-6 text-xs uppercase tracking-widest text-green-700">
        ── INTERMISSION ──
      </div>
      <div className="mb-8 max-w-md px-4 text-center text-sm leading-relaxed">
        {text}
      </div>
      <div className="mb-2 text-xs text-green-600">next:</div>
      <div className="mb-8 text-base font-bold">{nextMapId}</div>
      <div className="text-xs text-green-700 animate-pulse">[ press any key to continue ]</div>
    </div>
  );
}
```

- [ ] **Step 2: Create `EndOfCycleStub` component**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/EndOfCycleStub.tsx`:
```tsx
import { useEffect } from 'react';

interface EndOfCycleStubProps {
  cycleNumber: number;
  onClose: () => void;
}

export function EndOfCycleStub({ cycleNumber, onClose }: EndOfCycleStubProps) {
  useEffect(() => {
    const handler = (_e: KeyboardEvent) => onClose();
    window.addEventListener('keydown', handler);
    return () => window.removeEventListener('keydown', handler);
  }, [onClose]);

  return (
    <div className="absolute inset-0 z-50 flex flex-col items-center justify-center bg-black px-6 text-center font-mono text-green-400">
      <div className="mb-8 text-xs uppercase tracking-widest text-green-700">
        ── END OF CYCLE {cycleNumber} ──
      </div>
      <div className="mb-4 text-2xl font-bold">CYCLE {cycleNumber} COMPLETE</div>
      <div className="mb-8 text-sm text-green-600">
        ─── ─── ─── ─── ─── ─── ───
      </div>
      <div className="mb-4 max-w-md text-sm leading-relaxed">
        WE ARE NOT A BAND.<br />
        WE ARE A GLITCH IN THE RENDER PIPELINE.
      </div>
      <div className="mb-12 max-w-md text-xs text-green-600 leading-relaxed">
        cycle {cycleNumber + 1} ── coming soon
      </div>
      <div className="text-xs text-green-700 animate-pulse">[ press any key ]</div>
    </div>
  );
}
```

- [ ] **Step 3: Update `LOOMGame` state machine**

Replace the entire content of `/Users/justinwest/Repos/l0b0tonline/components/LOOMGame.tsx` with:
```tsx
import { useCallback, useEffect, useRef, useState } from 'react';
import { BootSequence } from './loom/hud/BootSequence';
import { LoomHud } from './loom/hud/LoomHud';
import { IntermissionScreen } from './loom/hud/IntermissionScreen';
import { EndOfCycleStub } from './loom/hud/EndOfCycleStub';
import { loadMap } from './loom/engine/mapLoader';
import { GameLoop } from './loom/engine/gameLoop';
import type { LoomMap } from './loom/types';

interface LoomGameProps {
  onClose: () => void;
}

type GameState =
  | { kind: 'booting' }
  | { kind: 'playing'; mapId: string }
  | { kind: 'intermission'; previousMap: LoomMap; nextMapId: string }
  | { kind: 'cycle-end'; cycleNumber: number };

export function LOOMGame(_props: LoomGameProps) {
  const canvasRef = useRef<HTMLCanvasElement | null>(null);
  const loopRef = useRef<GameLoop | null>(null);
  const currentMapRef = useRef<LoomMap | null>(null);
  const [state, setState] = useState<GameState>({ kind: 'booting' });
  const [error, setError] = useState<string | null>(null);

  // Start playing cyc1_lobby once the boot sequence completes.
  const onBootComplete = useCallback(() => {
    setState({ kind: 'playing', mapId: 'cyc1_lobby' });
  }, []);

  // Effect: when we enter a 'playing' state, load the map and run the loop.
  useEffect(() => {
    if (state.kind !== 'playing' || !canvasRef.current) return;
    let cancelled = false;
    (async () => {
      try {
        const map = await loadMap(state.mapId);
        if (cancelled || !canvasRef.current) return;
        currentMapRef.current = map;
        loopRef.current = new GameLoop(canvasRef.current, map, (exit) => {
          if (exit.kind === 'next-map') {
            setState({ kind: 'intermission', previousMap: map, nextMapId: exit.id });
          } else {
            setState({ kind: 'cycle-end', cycleNumber: 1 });
          }
        });
        loopRef.current.start();
        canvasRef.current.focus();
        if (typeof window !== 'undefined') {
          (window as unknown as { __LOOM_GAME__?: GameLoop }).__LOOM_GAME__ = loopRef.current;
        }
      } catch (e) {
        setError(e instanceof Error ? e.message : String(e));
      }
    })();
    return () => {
      cancelled = true;
      loopRef.current?.stop();
      loopRef.current = null;
      if (typeof window !== 'undefined') {
        delete (window as unknown as { __LOOM_GAME__?: GameLoop }).__LOOM_GAME__;
      }
    };
  }, [state]);

  const getSnapshot = useCallback(() => loopRef.current!.getSnapshot(), []);

  return (
    <div className="relative h-full w-full overflow-hidden bg-black">
      <canvas
        ref={canvasRef}
        className="h-full w-full"
        style={{ imageRendering: 'pixelated' }}
        width={640}
        height={480}
      />

      {state.kind === 'booting' && <BootSequence onComplete={onBootComplete} />}

      {state.kind === 'playing' && loopRef.current && (
        <LoomHud getSnapshot={getSnapshot} />
      )}

      {state.kind === 'intermission' && (
        <IntermissionScreen
          text={state.previousMap.intermissionText ?? '... continuing ...'}
          nextMapId={state.nextMapId}
          onContinue={() => setState({ kind: 'playing', mapId: state.nextMapId })}
        />
      )}

      {state.kind === 'cycle-end' && (
        <EndOfCycleStub
          cycleNumber={state.cycleNumber}
          onClose={() => setState({ kind: 'booting' })}
        />
      )}

      {error && (
        <div className="absolute inset-0 flex items-center justify-center bg-black/80 font-mono text-sm text-red-400">
          ERROR: {error}
        </div>
      )}
    </div>
  );
}
```

Notes:
- `state` discriminated union drives rendering — only one screen shows at a time.
- `cyc1_lobby` is the bootable starting map (replaces the previous `cyc1_test`). The placeholder `cyc1_test.json` stays on disk but is no longer the starting point.
- The cycle-end stub's "press any key" loops back to `booting` so the player can replay (a satisfying "you completed Cycle 1" moment).

- [ ] **Step 4: Type-check + run e2e to confirm Phase 1 spec still passes**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -10
npm run test:e2e -- e2e/loom-phase-1.spec.ts 2>&1 | tail -10
```

Expected: tsc clean. The Phase 1 spec might break here — it loaded `cyc1_test` directly via `__STORE__.openWindow` and our new state machine expects to spawn into `cyc1_lobby`. If the spec breaks, **temporarily skip the Phase 1 spec** (we'll re-introduce a Phase 2 spec in Task 2.18 that exercises the proper boot-into-cyc1_lobby flow):

```bash
mv e2e/loom-phase-1.spec.ts e2e/loom-phase-1.spec.ts.skip
```

Document the skip in the commit. The Phase 2 spec replaces it.

- [ ] **Step 5: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/hud/IntermissionScreen.tsx \
        components/loom/hud/EndOfCycleStub.tsx \
        components/LOOMGame.tsx \
        e2e/loom-phase-1.spec.ts.skip
git rm e2e/loom-phase-1.spec.ts
git commit -m "Add LOOMGame state machine + IntermissionScreen + EndOfCycleStub

LOOMGame now drives a GameState union: booting -> playing -> intermission
-> playing -> ... -> cycle-end. The GameLoop's onMapExit callback drives
state transitions. Press-any-key advances intermission/cycle-end screens.

Cycle 1 starts at cyc1_lobby (not the cyc1_test placeholder). The cycle-
end stub displays the manifesto opening lyric + 'cycle 2 coming soon'.

Phase 1 e2e spec skipped (renamed to .skip) — it bypassed the boot flow
that the new state machine enforces. Phase 2 spec lands in Task 2.18."
```

---

## Task 2.6: Drum Stick weapon (slot 1, melee with rhythm combo)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/drumStick.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Drum Stick is a melee weapon: short range, no ammo, but fast. It has a rhythm-combo gimmick: 4 hits within a 1.2-second window stuns the next hit.

- [ ] **Step 1: Implement Drum Stick**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/drumStick.ts`:
```typescript
import type { IWeapon } from './IWeapon';
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import type { IEnemy } from '../enemies/Enemy';

const DAMAGE = 8;
const STUN_BONUS_DAMAGE = 16;
const RANGE = 1.5;
const COMBO_WINDOW_MS = 1200;
const COMBO_THRESHOLD = 4;

export class DrumStick implements IWeapon {
  readonly name = 'Drum Stick';
  readonly slot = 1;

  private comboHits = 0;
  private comboLastAt = 0;

  fire(player: Player, _map: LoomMap, enemies: IEnemy[]): boolean {
    const now = performance.now();
    if (now - this.comboLastAt > COMBO_WINDOW_MS) {
      this.comboHits = 0;
    }

    // Find the closest live enemy within RANGE in front of the player.
    const dirX = Math.cos(player.angle);
    const dirY = Math.sin(player.angle);
    let closest: { enemy: IEnemy; dist: number } | null = null;
    for (const e of enemies) {
      if (e.state === 'dead') continue;
      const ex = e.x - player.x;
      const ey = e.y - player.y;
      const along = ex * dirX + ey * dirY;
      if (along <= 0 || along > RANGE) continue;
      const perp = Math.abs(ex * dirY - ey * dirX);
      if (perp > 0.6) continue;
      if (!closest || along < closest.dist) closest = { enemy: e, dist: along };
    }

    this.comboHits += 1;
    this.comboLastAt = now;
    const isStunHit = this.comboHits >= COMBO_THRESHOLD;
    if (isStunHit) {
      this.comboHits = 0; // reset combo
    }

    if (closest) {
      closest.enemy.takeDamage(isStunHit ? STUN_BONUS_DAMAGE : DAMAGE);
    }
    return true;
  }
}
```

- [ ] **Step 2: Register Drum Stick on slot 1**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`. Add the import:
```typescript
import { DrumStick } from '../entities/weapons/drumStick';
```

In the constructor, change `this.weaponsBySlot.set(2, [new BrandedPen()]);` to:
```typescript
this.weaponsBySlot.set(1, [new DrumStick()]);
this.weaponsBySlot.set(2, [new BrandedPen()]);
```

(Industrial Shredder gets pushed onto slot 1 in Task 2.7.)

- [ ] **Step 3: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/weapons/drumStick.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add Drum Stick weapon (slot 1, melee with rhythm combo)

Vint's drumstick. Short range (1.5 tiles), no ammo, 8 damage per hit.
4 hits within a 1.2s window primes a stun-bonus on the next hit
(2x damage, combo resets). Locked-in melee, encourages closing
distance with low-tier enemies."
```

---

## Task 2.7: Industrial Shredder weapon (slot 1 alt — drag-in melee)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/industrialShredder.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Spec calls Industrial Shredder a "Cycle 2 alt" but per the Phase 1 plan it gets pulled forward to Phase 2 since the chainsaw-equivalent class is a one-time engineering cost. Slot 1 alt — pressing 1 again cycles from Drum Stick to Shredder.

- [ ] **Step 1: Implement Industrial Shredder**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/industrialShredder.ts`:
```typescript
import type { IWeapon } from './IWeapon';
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import type { IEnemy } from '../enemies/Enemy';

const DAMAGE = 4;
const RANGE = 2.0;
const PULL_DISTANCE = 0.3; // tile units pulled toward player per hit

/** Industrial Shredder — slot 1 alt. Higher range than the Drum Stick,
 *  lower per-hit damage, but pulls the target slightly toward the player
 *  ("buffering..." sound effect comes in Phase 6 audio polish). */
export class IndustrialShredder implements IWeapon {
  readonly name = 'Industrial Shredder';
  readonly slot = 1;

  fire(player: Player, _map: LoomMap, enemies: IEnemy[]): boolean {
    const dirX = Math.cos(player.angle);
    const dirY = Math.sin(player.angle);
    let closest: { enemy: IEnemy; dist: number } | null = null;
    for (const e of enemies) {
      if (e.state === 'dead') continue;
      const ex = e.x - player.x;
      const ey = e.y - player.y;
      const along = ex * dirX + ey * dirY;
      if (along <= 0 || along > RANGE) continue;
      const perp = Math.abs(ex * dirY - ey * dirX);
      if (perp > 0.7) continue;
      if (!closest || along < closest.dist) closest = { enemy: e, dist: along };
    }

    if (closest) {
      closest.enemy.takeDamage(DAMAGE);
      // Pull toward player (drag-in mechanic)
      const dx = player.x - closest.enemy.x;
      const dy = player.y - closest.enemy.y;
      const dist = Math.hypot(dx, dy);
      if (dist > 0.3) {
        closest.enemy.x += (dx / dist) * PULL_DISTANCE;
        closest.enemy.y += (dy / dist) * PULL_DISTANCE;
      }
    }
    return true;
  }
}
```

- [ ] **Step 2: Register Industrial Shredder on slot 1 alt**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`. Add the import:
```typescript
import { IndustrialShredder } from '../entities/weapons/industrialShredder';
```

Change `this.weaponsBySlot.set(1, [new DrumStick()]);` to:
```typescript
this.weaponsBySlot.set(1, [new DrumStick(), new IndustrialShredder()]);
```

- [ ] **Step 3: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/weapons/industrialShredder.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add Industrial Shredder (slot 1 alt — drag-in melee)

Pressing '1' cycles between Drum Stick and Industrial Shredder.
Shredder has lower damage (4) but longer range (2.0) and a drag-in
effect that pulls hit enemies 0.3 tiles toward the player. Useful
for chaining hits."
```

---

## Task 2.8: Spam Filter weapon (slot 3 — spread shotgun)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/spamFilter.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Slot 3, shotgun-style. Fires 5 hitscan rays in a cone with random angular spread; each ray independently resolves against the closest enemy.

- [ ] **Step 1: Implement Spam Filter**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/spamFilter.ts`:
```typescript
import type { IWeapon } from './IWeapon';
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import type { IEnemy } from '../enemies/Enemy';
import { castRay } from '../../engine/raycaster';

const DAMAGE_PER_PELLET = 5;
const PELLETS = 5;
const SPREAD_RADIANS = 0.18; // ~10 degrees half-angle
const RANGE = 12;
const AMMO_COST = 1;

export class SpamFilter implements IWeapon {
  readonly name = 'Spam Filter';
  readonly slot = 3;

  fire(player: Player, map: LoomMap, enemies: IEnemy[]): boolean {
    if (player.ammo < AMMO_COST) return false;
    player.ammo -= AMMO_COST;

    for (let p = 0; p < PELLETS; p++) {
      const spread = (Math.random() - 0.5) * 2 * SPREAD_RADIANS;
      const angle = player.angle + spread;
      const dirX = Math.cos(angle);
      const dirY = Math.sin(angle);
      const wallHit = castRay(map, player.x, player.y, angle);

      let closest: { enemy: IEnemy; dist: number } | null = null;
      for (const e of enemies) {
        if (e.state === 'dead') continue;
        const ex = e.x - player.x;
        const ey = e.y - player.y;
        const along = ex * dirX + ey * dirY;
        if (along <= 0 || along > RANGE) continue;
        const perp = Math.abs(ex * dirY - ey * dirX);
        if (perp > 0.4) continue;
        if (along > wallHit.distance) continue;
        if (!closest || along < closest.dist) closest = { enemy: e, dist: along };
      }
      if (closest) closest.enemy.takeDamage(DAMAGE_PER_PELLET);
    }
    return true;
  }
}
```

- [ ] **Step 2: Register Spam Filter on slot 3**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`. Add the import:
```typescript
import { SpamFilter } from '../entities/weapons/spamFilter';
```

After the slot-2 setter, add:
```typescript
this.weaponsBySlot.set(3, [new SpamFilter()]);
```

- [ ] **Step 3: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/weapons/spamFilter.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add Spam Filter weapon (slot 3 — spread shotgun)

5 pellets per shot, ~10° half-angle spread, 5 damage per pellet
(25 max single-target). Ammo cost: 1 per shot from the shared pool.
Each pellet is an independent hitscan ray resolved against the
closest enemy along its trajectory."
```

---

## Task 2.9: Projectile system + `Projectile` base interface

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/Projectile.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/renderer.ts`

Phase 1 weapons were all hitscan. Phase 2 adds enemies that fire actual projectiles (Recruiter's fireball, HR Manager's tracking projectile). We need a system: projectiles tick each frame, move, check collision with the player, get drawn as billboarded sprites.

- [ ] **Step 1: Create the Projectile interface**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/Projectile.ts`:
```typescript
import type { Player } from '../player';
import type { LoomMap } from '../../types';

export interface IProjectile {
  x: number;
  y: number;
  /** True once the projectile has hit something or expired. */
  dead: boolean;
  /** Sprite color tint for placeholder rendering. */
  readonly color: readonly [number, number, number];
  /** Update one frame; check collisions; mark dead if needed. */
  update(dt: number, player: Player, map: LoomMap): void;
}
```

- [ ] **Step 2: Add projectile tracking to `GameLoop`**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`. Add the import:
```typescript
import type { IProjectile } from '../entities/projectiles/Projectile';
```

Add the field:
```typescript
private projectiles: IProjectile[] = [];
```

Add a public method (used by enemies in Tasks 2.11 and 2.13):
```typescript
spawnProjectile(p: IProjectile) {
  this.projectiles.push(p);
}
```

Add a getter for renderer access:
```typescript
getProjectiles(): IProjectile[] {
  return this.projectiles;
}
```

In the tick function, after the enemies loop, add:
```typescript
for (const p of this.projectiles) p.update(dt, this.player, this.map);
this.projectiles = this.projectiles.filter((p) => !p.dead);
```

Update the `renderer.render(...)` call to pass projectiles:
```typescript
this.renderer.render(this.map, this.player.x, this.player.y, this.player.angle, this.enemies, this.projectiles);
```

We also need enemies to spawn projectiles into the loop. Pass a reference to the GameLoop's `spawnProjectile` into the enemy update — change the IEnemy.update signature to:

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/Enemy.ts` — replace its content with:
```typescript
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import type { IProjectile } from '../projectiles/Projectile';

export type EnemyState = 'idle' | 'chase' | 'pain' | 'dead';

/**
 * Context passed to enemy update — gives the enemy access to spawn
 * projectiles back into the game world (via the GameLoop's queue).
 */
export interface EnemyUpdateContext {
  player: Player;
  map: LoomMap;
  spawnProjectile: (p: IProjectile) => void;
}

export interface IEnemy {
  x: number;
  y: number;
  health: number;
  state: EnemyState;
  readonly spriteName: string;
  readonly placeholderColor: readonly [number, number, number];

  update(dt: number, ctx: EnemyUpdateContext): void;
  takeDamage(damage: number): void;
}
```

This signature change ripples back to `Intern`. Update `intern.ts`'s `update` signature:
```typescript
update(dt: number, ctx: EnemyUpdateContext) {
  if (this.state === 'dead') return;
  // ... use ctx.player and ctx.map instead of the old player/map params ...
}
```
Replace every `player.` with `ctx.player.` and every `_map` with `ctx.map`.

In `gameLoop.ts`, the enemy update call becomes:
```typescript
const enemyCtx: EnemyUpdateContext = {
  player: this.player,
  map: this.map,
  spawnProjectile: (p) => this.projectiles.push(p),
};
for (const e of this.enemies) e.update(dt, enemyCtx);
```

Add the import: `import type { EnemyUpdateContext } from '../entities/enemies/Enemy';`

- [ ] **Step 3: Add projectile rendering to `Renderer`**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/renderer.ts`. Add the import:
```typescript
import type { IProjectile } from '../entities/projectiles/Projectile';
```

Update the `render` signature:
```typescript
render(
  map: LoomMap,
  playerX: number,
  playerY: number,
  playerAngle: number,
  enemies: IEnemy[],
  projectiles: IProjectile[],
) {
```

After the existing sprite-billboarding loop for enemies, add an analogous loop for projectiles. Reuse the camera-space-projection math; render projectiles as smaller squares (~50% the size of an enemy):
```typescript
// Projectile billboarding pass
const livingProjectiles = projectiles.filter((p) => !p.dead);
const sortedProjectiles = livingProjectiles
  .map((p) => ({ p, dist: Math.hypot(p.x - playerX, p.y - playerY) }))
  .sort((a, b) => b.dist - a.dist);

for (const { p } of sortedProjectiles) {
  const ex = p.x - playerX;
  const ey = p.y - playerY;
  const cosA = Math.cos(-playerAngle);
  const sinA = Math.sin(-playerAngle);
  const cx = ex * cosA - ey * sinA;
  const cy = ex * sinA + ey * cosA;
  if (cx <= 0.1) continue;

  const screenX = Math.floor((INTERNAL_W / 2) * (1 + cy / cx / Math.tan(FOV / 2)));
  const projectileHeight = Math.min(INTERNAL_H, INTERNAL_H / cx) * 0.4;
  const drawStartY = Math.floor((INTERNAL_H - projectileHeight) / 2);
  const drawEndY = Math.floor((INTERNAL_H + projectileHeight) / 2);
  const projectileWidth = Math.floor(projectileHeight);
  const drawStartX = Math.max(0, screenX - Math.floor(projectileWidth / 2));
  const drawEndX = Math.min(INTERNAL_W - 1, screenX + Math.floor(projectileWidth / 2));

  for (let x = drawStartX; x <= drawEndX; x++) {
    if (x < 0 || x >= INTERNAL_W) continue;
    if (zBuffer[x] !== undefined && cx > zBuffer[x]) continue;
    for (let y = Math.max(0, drawStartY); y < Math.min(INTERNAL_H, drawEndY); y++) {
      const idx = (y * INTERNAL_W + x) * 4;
      this.framebuffer[idx]     = p.color[0];
      this.framebuffer[idx + 1] = p.color[1];
      this.framebuffer[idx + 2] = p.color[2];
      this.framebuffer[idx + 3] = 255;
    }
  }
}
```

- [ ] **Step 4: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -10
git add components/loom/entities/projectiles/Projectile.ts \
        components/loom/entities/enemies/Enemy.ts \
        components/loom/entities/enemies/intern.ts \
        components/loom/engine/gameLoop.ts \
        components/loom/engine/renderer.ts
git commit -m "Add projectile system + rendering

New IProjectile interface; GameLoop tracks a projectile array, ticks
each frame, sweeps dead ones. Enemies receive an EnemyUpdateContext
with spawnProjectile so they can fire into the world.

Renderer billboards projectiles after enemies (z-tested against the
wall buffer like sprites). Projectiles render at ~40% sprite-height
to feel smaller than enemies. First projectile actor lands in
Task 2.11 (Recruiter fireball)."
```

---

## Task 2.10: Account Executive enemy (slot-3 spread shotgun blasts)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/accountExecutive.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/sprites/account_executive_idle.png`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

DOOM-style Sergeant analog: slow ranged attack, fires hitscan blasts. Higher HP than Intern, more damage per shot.

- [ ] **Step 1: Generate placeholder sprite**

```bash
cd /Users/justinwest/Repos/l0b0tonline
node -e "
const fs = require('fs');
const { PNG } = require('pngjs');
const png = new PNG({ width: 32, height: 64 });
for (let y = 0; y < 64; y++) {
  for (let x = 0; x < 32; x++) {
    const idx = (32 * y + x) << 2;
    const inBody = x >= 4 && x < 28 && y >= 4 && y < 60;
    const inHead = x >= 10 && x < 22 && y >= 8 && y < 22;
    if (inHead) { png.data[idx]=200; png.data[idx+1]=180; png.data[idx+2]=60; png.data[idx+3]=255; }
    else if (inBody) { png.data[idx]=120; png.data[idx+1]=90; png.data[idx+2]=20; png.data[idx+3]=255; }
    else { png.data[idx+3]=0; }
  }
}
png.pack().pipe(fs.createWriteStream('public/data/loom/sprites/account_executive_idle.png')).on('finish', () => console.log('OK'));
"
```

- [ ] **Step 2: Add `'account_executive'` to `ThingType`**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`. Update:
```typescript
export type ThingType =
  | 'player_start'
  | 'intern'
  | 'account_executive'
  | 'exit';
```

- [ ] **Step 3: Implement Account Executive**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/accountExecutive.ts`:
```typescript
import type { IEnemy, EnemyState, EnemyUpdateContext } from './Enemy';
import { castRay } from '../../engine/raycaster';

const HEALTH = 40;
const SPEED = 0.7;
const SIGHT_RANGE = 10;
const ATTACK_RANGE = 7;
const ATTACK_COOLDOWN_MS = 1500;
const PELLETS = 3;
const SPREAD_RADIANS = 0.15;
const DAMAGE_PER_PELLET = 4;

export class AccountExecutive implements IEnemy {
  x: number;
  y: number;
  health = HEALTH;
  state: EnemyState = 'idle';
  painUntil = 0;
  private nextAttackAt = 0;

  readonly spriteName = 'account_executive_idle';
  readonly placeholderColor = [200, 180, 60] as const;

  constructor(x: number, y: number) {
    this.x = x;
    this.y = y;
  }

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
      // Ranged attack when in range and on cooldown
      if (dist <= ATTACK_RANGE && performance.now() >= this.nextAttackAt) {
        this.nextAttackAt = performance.now() + ATTACK_COOLDOWN_MS;
        // Hitscan spread shotgun toward player. No projectiles for AE — instant
        // damage like a real shotgun blast.
        const angleToPlayer = Math.atan2(dy, dx);
        let didHit = false;
        for (let p = 0; p < PELLETS; p++) {
          const spread = (Math.random() - 0.5) * 2 * SPREAD_RADIANS;
          const a = angleToPlayer + spread;
          const wallHit = castRay(ctx.map, this.x, this.y, a);
          if (dist < wallHit.distance) {
            // Pellet reaches player — small per-pellet random miss for fairness
            if (Math.random() > 0.4) didHit = true;
          }
        }
        if (didHit) ctx.player.takeDamage(DAMAGE_PER_PELLET);
        return;
      }

      // Otherwise close distance
      const move = SPEED * dt;
      if (dist > 1.5) {
        this.x += (dx / dist) * move;
        this.y += (dy / dist) * move;
      }
    }
  }

  takeDamage(damage: number) {
    if (this.state === 'dead') return;
    this.health -= damage;
    if (this.health <= 0) {
      this.health = 0;
      this.state = 'dead';
      return;
    }
    this.state = 'pain';
    this.painUntil = performance.now() + 200;
  }
}
```

- [ ] **Step 4: Register `account_executive` thing in `GameLoop` constructor**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`. Add the import:
```typescript
import { AccountExecutive } from '../entities/enemies/accountExecutive';
```

In the `for (const t of map.things)` loop in the constructor, add a branch:
```typescript
} else if (t.type === 'account_executive') {
  this.enemies.push(new AccountExecutive(t.x, t.y));
}
```

- [ ] **Step 5: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/enemies/accountExecutive.ts \
        components/loom/types.ts \
        components/loom/engine/gameLoop.ts \
        public/data/loom/sprites/account_executive_idle.png
git commit -m "Add Account Executive enemy (Sergeant analog)

40 HP; ~1.5x range and 2x damage potential vs Intern. Fires hitscan
3-pellet spread shotgun bursts on a 1.5s cooldown when within 7 tiles.
Each pellet has a random fairness miss (~60% hit per pellet).

Visual: yellow placeholder rectangle. Real sprite art is Phase 4-5."
```

---

## Task 2.11: Recruiter enemy (Imp analog — fireball)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/recruiter.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/recruiterFireball.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/sprites/recruiter_idle.png`
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/sprites/recruiter_fireball.png`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Imp analog: fast melee + projectile attack. Recruiter throws a fireball "you'd be perfect for this role" email at range, scratches in melee.

- [ ] **Step 1: Generate placeholder sprites**

```bash
cd /Users/justinwest/Repos/l0b0tonline
node -e "
const fs = require('fs');
const { PNG } = require('pngjs');

// Recruiter — purple/magenta humanoid
const r = new PNG({ width: 32, height: 64 });
for (let y = 0; y < 64; y++) {
  for (let x = 0; x < 32; x++) {
    const idx = (32 * y + x) << 2;
    const inBody = x >= 4 && x < 28 && y >= 4 && y < 60;
    const inHead = x >= 10 && x < 22 && y >= 8 && y < 22;
    if (inHead) { r.data[idx]=220; r.data[idx+1]=180; r.data[idx+2]=220; r.data[idx+3]=255; }
    else if (inBody) { r.data[idx]=140; r.data[idx+1]=60; r.data[idx+2]=160; r.data[idx+3]=255; }
    else { r.data[idx+3]=0; }
  }
}
r.pack().pipe(fs.createWriteStream('public/data/loom/sprites/recruiter_idle.png')).on('finish', () => {
  // Recruiter fireball — orange-yellow blob
  const f = new PNG({ width: 16, height: 16 });
  for (let y = 0; y < 16; y++) {
    for (let x = 0; x < 16; x++) {
      const idx = (16 * y + x) << 2;
      const cx = 8, cy = 8, r2 = (x-cx)*(x-cx) + (y-cy)*(y-cy);
      if (r2 < 36) { f.data[idx]=255; f.data[idx+1]=180; f.data[idx+2]=60; f.data[idx+3]=255; }
      else if (r2 < 56) { f.data[idx]=200; f.data[idx+1]=120; f.data[idx+2]=40; f.data[idx+3]=255; }
      else { f.data[idx+3]=0; }
    }
  }
  f.pack().pipe(fs.createWriteStream('public/data/loom/sprites/recruiter_fireball.png')).on('finish', () => console.log('OK'));
});
"
```

- [ ] **Step 2: Implement Recruiter fireball projectile**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/recruiterFireball.ts`:
```typescript
import type { IProjectile } from './Projectile';
import type { Player } from '../player';
import type { LoomMap } from '../../types';

const SPEED = 4.0;
const DAMAGE = 8;
const HIT_RADIUS = 0.4;
const MAX_LIFETIME_MS = 5000;

export class RecruiterFireball implements IProjectile {
  x: number;
  y: number;
  dead = false;
  readonly color = [255, 180, 60] as const;
  private dirX: number;
  private dirY: number;
  private spawnedAt = performance.now();

  constructor(x: number, y: number, angle: number) {
    this.x = x;
    this.y = y;
    this.dirX = Math.cos(angle);
    this.dirY = Math.sin(angle);
  }

  update(dt: number, player: Player, map: LoomMap) {
    if (this.dead) return;
    if (performance.now() - this.spawnedAt > MAX_LIFETIME_MS) {
      this.dead = true;
      return;
    }

    const newX = this.x + this.dirX * SPEED * dt;
    const newY = this.y + this.dirY * SPEED * dt;

    // Hit player check
    const dx = player.x - newX;
    const dy = player.y - newY;
    if (Math.hypot(dx, dy) < HIT_RADIUS) {
      player.takeDamage(DAMAGE);
      this.dead = true;
      return;
    }

    // Hit wall check
    const ix = Math.floor(newX);
    const iy = Math.floor(newY);
    if (iy < 0 || iy >= map.grid.length || !map.grid[iy] || ix < 0 || ix >= map.grid[iy].length || map.grid[iy][ix] > 0) {
      this.dead = true;
      return;
    }

    this.x = newX;
    this.y = newY;
  }
}
```

- [ ] **Step 3: Implement Recruiter enemy**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/recruiter.ts`:
```typescript
import type { IEnemy, EnemyState, EnemyUpdateContext } from './Enemy';
import { RecruiterFireball } from '../projectiles/recruiterFireball';

const HEALTH = 30;
const SPEED = 1.5; // faster than Intern
const SIGHT_RANGE = 9;
const FIREBALL_RANGE = 8;
const FIREBALL_COOLDOWN_MS = 1800;
const MELEE_RANGE = 1.0;
const MELEE_DAMAGE = 6;
const MELEE_COOLDOWN_MS = 700;

export class Recruiter implements IEnemy {
  x: number;
  y: number;
  health = HEALTH;
  state: EnemyState = 'idle';
  painUntil = 0;
  private nextFireballAt = 0;
  private nextMeleeAt = 0;

  readonly spriteName = 'recruiter_idle';
  readonly placeholderColor = [180, 80, 200] as const;

  constructor(x: number, y: number) {
    this.x = x;
    this.y = y;
  }

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
      // Melee in close range
      if (dist <= MELEE_RANGE && performance.now() >= this.nextMeleeAt) {
        ctx.player.takeDamage(MELEE_DAMAGE);
        this.nextMeleeAt = performance.now() + MELEE_COOLDOWN_MS;
        return;
      }
      // Fireball at mid-range
      if (dist <= FIREBALL_RANGE && performance.now() >= this.nextFireballAt) {
        this.nextFireballAt = performance.now() + FIREBALL_COOLDOWN_MS;
        const angleToPlayer = Math.atan2(dy, dx);
        ctx.spawnProjectile(new RecruiterFireball(this.x, this.y, angleToPlayer));
        return;
      }
      // Otherwise chase
      const move = SPEED * dt;
      if (dist > 0.5) {
        this.x += (dx / dist) * move;
        this.y += (dy / dist) * move;
      }
    }
  }

  takeDamage(damage: number) {
    if (this.state === 'dead') return;
    this.health -= damage;
    if (this.health <= 0) {
      this.health = 0;
      this.state = 'dead';
      return;
    }
    this.state = 'pain';
    this.painUntil = performance.now() + 200;
  }
}
```

- [ ] **Step 4: Add `'recruiter'` to ThingType + register in GameLoop**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts` — add `'recruiter'` to `ThingType`.

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts` — add the import:
```typescript
import { Recruiter } from '../entities/enemies/recruiter';
```

In the constructor's `for (const t of map.things)` loop, add a branch:
```typescript
} else if (t.type === 'recruiter') {
  this.enemies.push(new Recruiter(t.x, t.y));
}
```

- [ ] **Step 5: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/enemies/recruiter.ts \
        components/loom/entities/projectiles/recruiterFireball.ts \
        components/loom/types.ts \
        components/loom/engine/gameLoop.ts \
        public/data/loom/sprites/recruiter_idle.png \
        public/data/loom/sprites/recruiter_fireball.png
git commit -m "Add Recruiter enemy + fireball projectile (Imp analog)

30 HP, 1.5x Intern speed. Fast melee scratch (6 damage, 700ms CD)
in close range; fireball email projectile at mid-range (4.0 tiles/sec
travel speed, 8 damage on hit, 5s lifetime). Fireball check is full-
trajectory (movement + wall + player collision per tick).

Recruiter sprite: magenta/purple placeholder. Fireball sprite: orange
blob, 16x16, billboarded smaller than enemies."
```

---

## Task 2.12: HR Manager boss (high HP + tracking projectile)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/hrManager.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/documentationRequest.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/sprites/hr_manager_idle.png`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

Cycle 1 boss. High HP (200, ~10x basic), fires "documentation request" projectiles that weakly track the player (correct course toward player at low rate). No phase transitions for Phase 2 — multi-phase bosses arrive with The Hand in Phase 5.

- [ ] **Step 1: Generate placeholder sprite**

```bash
cd /Users/justinwest/Repos/l0b0tonline
node -e "
const fs = require('fs');
const { PNG } = require('pngjs');
const png = new PNG({ width: 48, height: 96 });
for (let y = 0; y < 96; y++) {
  for (let x = 0; x < 48; x++) {
    const idx = (48 * y + x) << 2;
    const inBody = x >= 6 && x < 42 && y >= 6 && y < 90;
    const inHead = x >= 16 && x < 32 && y >= 12 && y < 32;
    const inClipboard = x >= 4 && x < 16 && y >= 40 && y < 60;
    if (inHead) { png.data[idx]=220; png.data[idx+1]=200; png.data[idx+2]=220; png.data[idx+3]=255; }
    else if (inClipboard) { png.data[idx]=255; png.data[idx+1]=255; png.data[idx+2]=240; png.data[idx+3]=255; }
    else if (inBody) { png.data[idx]=80; png.data[idx+1]=70; png.data[idx+2]=140; png.data[idx+3]=255; }
    else { png.data[idx+3]=0; }
  }
}
png.pack().pipe(fs.createWriteStream('public/data/loom/sprites/hr_manager_idle.png')).on('finish', () => console.log('OK'));
"
```

- [ ] **Step 2: Implement Documentation Request tracking projectile**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/projectiles/documentationRequest.ts`:
```typescript
import type { IProjectile } from './Projectile';
import type { Player } from '../player';
import type { LoomMap } from '../../types';

const SPEED = 2.8; // slower than Recruiter fireball — must dodge
const DAMAGE = 12;
const HIT_RADIUS = 0.4;
const MAX_LIFETIME_MS = 8000;
const TRACKING_RATE = 0.8; // radians per second of course-correction toward player

export class DocumentationRequest implements IProjectile {
  x: number;
  y: number;
  dead = false;
  readonly color = [255, 255, 240] as const;
  private dirAngle: number;
  private spawnedAt = performance.now();

  constructor(x: number, y: number, initialAngle: number) {
    this.x = x;
    this.y = y;
    this.dirAngle = initialAngle;
  }

  update(dt: number, player: Player, map: LoomMap) {
    if (this.dead) return;
    if (performance.now() - this.spawnedAt > MAX_LIFETIME_MS) {
      this.dead = true;
      return;
    }

    // Slowly turn toward player
    const desired = Math.atan2(player.y - this.y, player.x - this.x);
    let diff = desired - this.dirAngle;
    while (diff > Math.PI) diff -= 2 * Math.PI;
    while (diff < -Math.PI) diff += 2 * Math.PI;
    const maxTurn = TRACKING_RATE * dt;
    if (Math.abs(diff) <= maxTurn) {
      this.dirAngle = desired;
    } else {
      this.dirAngle += Math.sign(diff) * maxTurn;
    }

    const dirX = Math.cos(this.dirAngle);
    const dirY = Math.sin(this.dirAngle);
    const newX = this.x + dirX * SPEED * dt;
    const newY = this.y + dirY * SPEED * dt;

    // Hit player
    if (Math.hypot(player.x - newX, player.y - newY) < HIT_RADIUS) {
      player.takeDamage(DAMAGE);
      this.dead = true;
      return;
    }

    // Hit wall
    const ix = Math.floor(newX);
    const iy = Math.floor(newY);
    if (iy < 0 || iy >= map.grid.length || !map.grid[iy] || ix < 0 || ix >= map.grid[iy].length || map.grid[iy][ix] > 0) {
      this.dead = true;
      return;
    }

    this.x = newX;
    this.y = newY;
  }
}
```

- [ ] **Step 3: Implement HR Manager**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/hrManager.ts`:
```typescript
import type { IEnemy, EnemyState, EnemyUpdateContext } from './Enemy';
import { DocumentationRequest } from '../projectiles/documentationRequest';

const HEALTH = 200;
const SPEED = 0.6;
const SIGHT_RANGE = 16;
const ATTACK_RANGE = 12;
const ATTACK_COOLDOWN_MS = 2200;

export class HRManager implements IEnemy {
  x: number;
  y: number;
  health = HEALTH;
  state: EnemyState = 'idle';
  painUntil = 0;
  private nextAttackAt = 0;

  readonly spriteName = 'hr_manager_idle';
  readonly placeholderColor = [120, 100, 200] as const;

  constructor(x: number, y: number) {
    this.x = x;
    this.y = y;
  }

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
        ctx.spawnProjectile(new DocumentationRequest(this.x, this.y, angleToPlayer));
        return;
      }
      const move = SPEED * dt;
      if (dist > 3) {
        this.x += (dx / dist) * move;
        this.y += (dy / dist) * move;
      }
    }
  }

  takeDamage(damage: number) {
    if (this.state === 'dead') return;
    this.health -= damage;
    if (this.health <= 0) {
      this.health = 0;
      this.state = 'dead';
      return;
    }
    this.state = 'pain';
    this.painUntil = performance.now() + 100; // short pain — boss
  }
}
```

- [ ] **Step 4: Add `'hr_manager'` to ThingType + register in GameLoop**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts` — add `'hr_manager'` to `ThingType`.

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts` — add the import:
```typescript
import { HRManager } from '../entities/enemies/hrManager';
```

In the spawn loop, add:
```typescript
} else if (t.type === 'hr_manager') {
  this.enemies.push(new HRManager(t.x, t.y));
}
```

- [ ] **Step 5: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -5
git add components/loom/entities/enemies/hrManager.ts \
        components/loom/entities/projectiles/documentationRequest.ts \
        components/loom/types.ts \
        components/loom/engine/gameLoop.ts \
        public/data/loom/sprites/hr_manager_idle.png
git commit -m "Add HR Manager boss + tracking documentation request

200 HP (10x basic enemy). Fires DocumentationRequest projectiles every
2.2s when within 12 tiles. Projectiles travel at 2.8 tiles/sec and
weakly track the player (0.8 rad/sec course correction) — dodgeable
but punishing if you stand still.

No phase transitions for Phase 2 — multi-phase boss mechanics ship
with The Hand in Phase 5. Boss has short pain animation (100ms) so
DPS-checks aren't trivialized."
```

---

## Task 2.13: cyc1_lobby map (Cycle 1 opener)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc1_lobby.json`

The first real map. Open lobby aesthetic — wider than the test map, with reception-desk-implying interior walls. 5 Interns, 2 Account Executives, 1 Recruiter. Exit at the far end.

- [ ] **Step 1: Author cyc1_lobby.json**

Create `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc1_lobby.json`:
```json
{
  "id": "cyc1_lobby",
  "cycle": 1,
  "music": "ship_it",
  "intermissionText": "ENTRY LOG: SUBJECT 88-E breached LOBBY containment. PROCEED TO CUBICLE FLOOR.",
  "grid": [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 1, 1, 0, 0, 1],
    [1, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  ],
  "things": [
    { "type": "player_start", "x": 2, "y": 5, "angle": 0 },
    { "type": "intern", "x": 6, "y": 2 },
    { "type": "intern", "x": 9, "y": 2 },
    { "type": "intern", "x": 5, "y": 9 },
    { "type": "intern", "x": 11, "y": 6 },
    { "type": "intern", "x": 8, "y": 5 },
    { "type": "account_executive", "x": 12, "y": 2 },
    { "type": "account_executive", "x": 12, "y": 9 },
    { "type": "recruiter", "x": 7, "y": 8 },
    { "type": "exit", "x": 14, "y": 5 }
  ],
  "textures": ["wall_lobby"]
}
```

11 rows × 16 cols. Player spawns left-center facing east. Exit on the far right wall. Two interior wall pillars partition the space into something feeling like a foyer.

- [ ] **Step 2: Verify JSON parses + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
node -e "console.log('OK', JSON.parse(require('fs').readFileSync('public/data/loom/maps/cyc1_lobby.json','utf8')).id)"
git add public/data/loom/maps/cyc1_lobby.json
git commit -m "Add cyc1_lobby map — Cycle 1 opener

11x16 grid. Player spawns left-center facing east; exit far right.
Roster: 5 Interns, 2 Account Executives, 1 Recruiter. Two interior
wall pillars partition the lobby into a foyer-shape.

intermissionText narrates the breach into the cubicle floor."
```

---

## Task 2.14: cyc1_cubicles map

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc1_cubicles.json`

Cubicle farm: a regular grid of small enclosures with corridors between. More enemies, tighter spaces, encourages weapon-switching.

- [ ] **Step 1: Author cyc1_cubicles.json**

Create `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc1_cubicles.json`:
```json
{
  "id": "cyc1_cubicles",
  "cycle": 1,
  "music": "ship_it",
  "intermissionText": "CUBICLE FLOOR cleared. HR DIRECTOR file unlocked. PROCEED TO HR ARENA.",
  "grid": [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 1, 0, 1],
    [1, 0, 1, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 1, 0, 1],
    [1, 0, 1, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 1, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  ],
  "things": [
    { "type": "player_start", "x": 2, "y": 1, "angle": 0 },
    { "type": "intern", "x": 6, "y": 1 },
    { "type": "intern", "x": 11, "y": 4 },
    { "type": "intern", "x": 4, "y": 7 },
    { "type": "account_executive", "x": 8, "y": 4 },
    { "type": "account_executive", "x": 11, "y": 7 },
    { "type": "account_executive", "x": 6, "y": 9 },
    { "type": "recruiter", "x": 10, "y": 1 },
    { "type": "recruiter", "x": 4, "y": 9 },
    { "type": "exit", "x": 12, "y": 9 }
  ],
  "textures": ["wall_lobby"]
}
```

- [ ] **Step 2: Verify + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
node -e "console.log('OK', JSON.parse(require('fs').readFileSync('public/data/loom/maps/cyc1_cubicles.json','utf8')).id)"
git add public/data/loom/maps/cyc1_cubicles.json
git commit -m "Add cyc1_cubicles map — Cycle 1 mid

11x14 grid; cubicle pattern of L-shaped wall blocks creating 3 rows
of fake cubicle enclosures with corridors between. Roster: 3 Interns,
3 Account Executives, 2 Recruiters. Tighter spaces than the lobby —
favors close-quarters weapons."
```

---

## Task 2.15: cyc1_hr_arena map (boss arena)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc1_hr_arena.json`

Boss arena: large open space, HR Manager + 2 Account Executives + 3 Interns. Exit only matters after HR Manager dies (the GameLoop's `getNextMapId` will return null so it triggers the cycle-end stub). For Phase 2 simplicity: exit thing is always present; player can technically walk past the boss without killing them (we accept this as a Phase 2 fairness loophole and tighten later).

- [ ] **Step 1: Author cyc1_hr_arena.json**

Create `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc1_hr_arena.json`:
```json
{
  "id": "cyc1_hr_arena",
  "cycle": 1,
  "music": "ship_it",
  "intermissionText": "HR DIRECTOR neutralized. CYCLE 1 CONTAINMENT BREACH RESOLVED.",
  "grid": [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
  ],
  "things": [
    { "type": "player_start", "x": 2, "y": 6, "angle": 0 },
    { "type": "hr_manager", "x": 13, "y": 6 },
    { "type": "account_executive", "x": 9, "y": 3 },
    { "type": "account_executive", "x": 9, "y": 9 },
    { "type": "intern", "x": 6, "y": 3 },
    { "type": "intern", "x": 6, "y": 9 },
    { "type": "intern", "x": 12, "y": 6 },
    { "type": "exit", "x": 16, "y": 6 }
  ],
  "textures": ["wall_lobby"]
}
```

12 rows × 18 cols — biggest map. Two pillars provide cover. HR Manager is in the back-right; player spawns left-center facing east.

- [ ] **Step 2: Verify + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
node -e "console.log('OK', JSON.parse(require('fs').readFileSync('public/data/loom/maps/cyc1_hr_arena.json','utf8')).id)"
git add public/data/loom/maps/cyc1_hr_arena.json
git commit -m "Add cyc1_hr_arena map — Cycle 1 boss arena

12x18 — biggest Phase 2 map. Two interior pillars give cover from
HR Manager's tracking projectiles. Roster: 1 HR Manager, 2 Account
Executives, 3 Interns. Exit at far right; reaching it (with or
without killing the boss) triggers the EndOfCycleStub.

Phase 2 fairness loophole: player can technically skip the boss by
walking past. Accepted for now; tightened in Phase 3 setup."
```

---

## Task 2.16: Audio integration (Ship It on HR arena + retro SFX hooks)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/audioController.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/intern.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/accountExecutive.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/recruiter.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/hrManager.ts`

Wraps `services/soundService.ts` for LOOM-specific calls. Per scout report, soundService exposes `startShipItMusic()`, `stopShipItMusic()`, `startAmbient()`, `stopAmbient()`, `playRetroHit()`, `playError()` etc. We don't add new sounds — we wire LOOM events to existing ones.

- [ ] **Step 1: Create audioController.ts**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/audioController.ts`:
```typescript
import { soundService } from '../../../services/soundService';

/**
 * Thin wrapper around l0b0tonline's existing soundService for LOOM-specific
 * audio events. Keeps SFX/music decisions in one place rather than scattered
 * across enemies/weapons/game-loop.
 *
 * Phase 2: maps soundService primitives to LOOM events. Real LOOM-specific
 * soundbanks (60Hz hum bed, weapon-specific click/recoil) come Phase 6.
 */
export class AudioController {
  /** Called when LOOM session starts (after boot). */
  onGameStart() {
    void soundService.resumeContext();
    soundService.startAmbient();
  }

  /** Called when LOOM session ends (window close / cleanup). */
  onGameEnd() {
    soundService.stopAmbient(true);
    soundService.stopShipItMusic();
  }

  /** Called when a new map loads. mapId controls boss-music behavior. */
  onMapLoad(mapId: string) {
    if (mapId === 'cyc1_hr_arena') {
      soundService.stopAmbient();
      soundService.startShipItMusic();
    } else {
      soundService.stopShipItMusic();
      soundService.startAmbient();
    }
  }

  /** Called when an enemy takes damage from any weapon. */
  onEnemyHit() {
    soundService.playRetroHit();
  }

  /** Called when the player takes damage. */
  onPlayerDamaged() {
    soundService.playError();
  }
}
```

- [ ] **Step 2: Wire AudioController into GameLoop**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`. Add the import:
```typescript
import { AudioController } from './audioController';
```

Add the field:
```typescript
private audio = new AudioController();
```

In the constructor, after `this.player = new Player(...)`:
```typescript
this.audio.onGameStart();
this.audio.onMapLoad(map.id);
```

In `stop()`, before any other cleanup:
```typescript
this.audio.onGameEnd();
```

Pass the audio controller into the enemy update context:
```typescript
const enemyCtx: EnemyUpdateContext = {
  player: this.player,
  map: this.map,
  spawnProjectile: (p) => this.projectiles.push(p),
  audio: this.audio,
};
```

Update the `EnemyUpdateContext` type in `Enemy.ts` accordingly:
```typescript
import type { AudioController } from '../../engine/audioController';

export interface EnemyUpdateContext {
  player: Player;
  map: LoomMap;
  spawnProjectile: (p: IProjectile) => void;
  audio: AudioController;
}
```

- [ ] **Step 3: Hook player-damage SFX in each enemy**

In each enemy file (`intern.ts`, `accountExecutive.ts`, `recruiter.ts`, `hrManager.ts`), find the spot where `ctx.player.takeDamage(...)` is called and immediately after it add:
```typescript
ctx.audio.onPlayerDamaged();
```

For Account Executive, this means after `ctx.player.takeDamage(DAMAGE_PER_PELLET);`. For Intern after the melee `ctx.player.takeDamage(MELEE_DAMAGE);`. For Recruiter after both melee and (the projectile handles its own damage so no audio call needed there — projectile damage is fire-and-forget). For HR Manager, no direct player-damage call — only via DocumentationRequest.

For Recruiter and HR Manager projectile damage: edit `recruiterFireball.ts` and `documentationRequest.ts` — when the projectile hits the player and calls `player.takeDamage(...)`, we'd need access to the audio controller. The cleanest way: have IProjectile.update receive an extended context similar to enemies. For Phase 2 simplicity, **skip projectile-damage SFX** — the player-damage hit sound only plays for hitscan attacks. (This is a known gap; track for Phase 3.)

- [ ] **Step 4: Hook enemy-hit SFX in each weapon**

The cleanest place is inside each weapon's `fire` method, after `closest.enemy.takeDamage(...)` calls. But weapons don't have access to AudioController either. Options:
1. Plumb audio through IWeapon.fire(player, map, enemies, audio)
2. Make a global module-scope event bus

For Phase 2 simplicity: **option 1**. Update IWeapon:
```typescript
import type { AudioController } from '../../engine/audioController';

export interface IWeapon {
  readonly name: string;
  readonly slot: number;
  fire(player: Player, map: LoomMap, enemies: IEnemy[], audio: AudioController): boolean;
}
```

Update each existing weapon (`brandedPen.ts`, `neuralPulse.ts`, `drumStick.ts`, `industrialShredder.ts`, `spamFilter.ts`) — change the signature:
```typescript
fire(player: Player, map: LoomMap, enemies: IEnemy[], audio: AudioController): boolean {
```

After every `.takeDamage(...)` call in the weapon body, add:
```typescript
audio.onEnemyHit();
```

For Spam Filter (multiple pellets), call `audio.onEnemyHit()` once outside the pellet loop, only if any pellet hit (track a bool).

Update the `getActiveWeapon()?.fire(...)` call site in `gameLoop.ts`:
```typescript
this.getActiveWeapon()?.fire(this.player, this.map, this.enemies, this.audio);
```
And the Neural Pulse fire:
```typescript
if (this.input.consumeFirePulse()) {
  this.neuralPulse.fire(this.player, this.map, this.enemies, this.audio);
}
```

- [ ] **Step 5: Type-check + commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -10
git add components/loom/engine/audioController.ts \
        components/loom/engine/gameLoop.ts \
        components/loom/entities/enemies/Enemy.ts \
        components/loom/entities/enemies/intern.ts \
        components/loom/entities/enemies/accountExecutive.ts \
        components/loom/entities/enemies/recruiter.ts \
        components/loom/entities/enemies/hrManager.ts \
        components/loom/entities/weapons/IWeapon.ts \
        components/loom/entities/weapons/brandedPen.ts \
        components/loom/entities/weapons/neuralPulse.ts \
        components/loom/entities/weapons/drumStick.ts \
        components/loom/entities/weapons/industrialShredder.ts \
        components/loom/entities/weapons/spamFilter.ts
git commit -m "Wire LOOM audio: Ship It on HR arena + retro SFX hooks

components/loom/engine/audioController.ts wraps soundService with LOOM
events: onGameStart/End, onMapLoad (boss music swap on cyc1_hr_arena),
onEnemyHit, onPlayerDamaged.

EnemyUpdateContext and IWeapon.fire now receive the AudioController so
gameplay events trigger sounds. Hitscan damage plays playRetroHit on
hit; melee/AE shotgun damage plays playError when player is hit.

Projectile-damage SFX is a known gap (projectiles update with a simpler
context); tracking for Phase 3."
```

---

## Task 2.17: Phase 2 e2e Playwright spec

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/e2e/loom-phase-2.spec.ts`

Replaces the skipped Phase 1 spec. Verifies the boot → cyc1_lobby → exit-trigger → intermission → cyc1_cubicles transition (the new state-machine flow).

- [ ] **Step 1: Write the Phase 2 spec**

Create `/Users/justinwest/Repos/l0b0tonline/e2e/loom-phase-2.spec.ts`:
```typescript
import { test, expect } from '@playwright/test';

test.describe('LOOM Phase 2 DoD', () => {
  test('boots → cyc1_lobby → HUD renders → progressing through map flows', async ({ page }) => {
    const consoleErrors: string[] = [];
    page.on('console', (msg) => {
      if (msg.type() === 'error') consoleErrors.push(msg.text());
    });
    page.on('pageerror', (err) => consoleErrors.push(err.message));

    await page.goto('/');

    // Boot J0IN 0S to desktop, then open LOOM via the store affordance.
    await page.waitForFunction(
      () => typeof (window as any).__STORE__ !== 'undefined',
      { timeout: 10_000 },
    );
    await page.evaluate(() => {
      (window as any).__STORE__.getState().windows.openWindow({
        type: 'GAME_LOOM',
        title: 'loom.exe',
      });
    });

    // Boot sequence runs
    await expect(page.getByText('LOOM TECHNOLOGIES')).toBeVisible({ timeout: 10_000 });
    await expect(page.getByText('ACCESS_GRANTED')).toBeVisible({ timeout: 10_000 });

    // After boot, HUD shows up with cyc1_lobby zone
    await expect(page.locator('canvas')).toBeVisible();
    await expect(page.getByText(/zone:.*cyc1_lobby/)).toBeVisible({ timeout: 10_000 });
    await expect(page.getByText('cycle 8492')).toBeVisible();

    // GameLoop ticking — frameCount increases over time
    await page.waitForFunction(
      () => {
        const g = (window as any).__LOOM_GAME__;
        return g && g.getSnapshot && typeof g.getSnapshot().frameCount === 'number';
      },
      { timeout: 5_000 },
    );
    const f1 = await page.evaluate(() => (window as any).__LOOM_GAME__.getSnapshot().frameCount);
    await page.waitForTimeout(500);
    const f2 = await page.evaluate(() => (window as any).__LOOM_GAME__.getSnapshot().frameCount);
    expect(f2).toBeGreaterThan(f1);

    // Probe: teleport player to the exit thing and watch state advance to intermission
    await page.evaluate(() => {
      const g = (window as any).__LOOM_GAME__;
      const player = g.getPlayer();
      // cyc1_lobby exit is at (14, 5)
      player.x = 14;
      player.y = 5;
    });

    await expect(page.getByText('next:')).toBeVisible({ timeout: 5_000 });
    await expect(page.getByText('cyc1_cubicles')).toBeVisible();

    // Press a key to advance from intermission
    await page.keyboard.press('Space');

    // Confirm we're now in cyc1_cubicles
    await expect(page.getByText(/zone:.*cyc1_cubicles/)).toBeVisible({ timeout: 5_000 });

    // Filter known noise (chrome-extension errors etc.)
    const relevantErrors = consoleErrors.filter(
      (e) =>
        !e.includes('chrome-extension') &&
        !e.includes('Failed to load resource: net::ERR_FAILED') &&
        !e.includes('favicon'),
    );
    expect(relevantErrors).toEqual([]);
  });
});
```

- [ ] **Step 2: Run spec, debug if needed**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npm run test:e2e -- e2e/loom-phase-2.spec.ts 2>&1 | tail -20
```

Expected: PASS. If selectors or timing fail, iterate. If a real bug surfaces (e.g., transition doesn't fire), report BLOCKED with details.

- [ ] **Step 3: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add e2e/loom-phase-2.spec.ts
git rm e2e/loom-phase-1.spec.ts.skip
git commit -m "Add Phase 2 DoD smoke test (replaces skipped Phase 1 spec)

Verifies the new state-machine flow: boot -> cyc1_lobby -> HUD renders
with correct zone -> GameLoop tickable -> teleport-to-exit triggers
intermission -> press-key advances to cyc1_cubicles. No console errors.

Removes the skipped Phase 1 spec — Phase 2 spec subsumes its coverage
plus tests the new map-progression and state-machine layers."
```

---

## Task 2.18: Phase 2 integration playtest log

**Files:**
- Create: `/Users/justinwest/Repos/LOOM-DOOM/.claude/worktrees/awesome-sutherland-aad854/docs/playtests/2026-04-27-phase-2-cycle-1.md`

Following the Phase 0/1 pattern: capture which DoD items the Playwright spec covers and which need human eyes.

- [ ] **Step 1: Write the playtest log**

Create `/Users/justinwest/Repos/LOOM-DOOM/.claude/worktrees/awesome-sutherland-aad854/docs/playtests/2026-04-27-phase-2-cycle-1.md`:
```markdown
# Phase 2 Cycle 1 Vertical Slice — Playtest Log

**Date:** 2026-04-27
**l0b0tonline commit:** [paste latest short SHA from loom-game]
**LOOM-DOOM plan commit:** [paste latest short SHA]
**Test file:** `l0b0tonline/e2e/loom-phase-2.spec.ts`
**Test command:** `npm run test:e2e -- e2e/loom-phase-2.spec.ts`

## Result

[X] PASS / [ ] FAIL

## Behavior verification

| # | Behavior | Verified by | Result |
|---|---|---|---|
| 1 | LOOM window opens from J0IN 0S desktop | Playwright | PASS / FAIL / N/A |
| 2 | Boot sequence plays | Playwright | PASS / FAIL / N/A |
| 3 | Player spawns into cyc1_lobby | Playwright | PASS / FAIL / N/A |
| 4 | HUD renders with cyc1_lobby zone | Playwright | PASS / FAIL / N/A |
| 5 | GameLoop ticks (frameCount increases) | Playwright | PASS / FAIL / N/A |
| 6 | Walking onto exit triggers intermission | Playwright probe | PASS / FAIL / N/A |
| 7 | Press-key advances intermission to cyc1_cubicles | Playwright | PASS / FAIL / N/A |
| 8 | No console errors | Playwright | PASS / FAIL / N/A |
| 9 | Walls render as a 3D corridor | Manual | NEEDS HUMAN |
| 10 | WASD walks; mouse-look turns | Manual | NEEDS HUMAN |
| 11 | Number keys 1/2/3 switch weapons | Manual | NEEDS HUMAN |
| 12 | Press-1-twice cycles Drum Stick ↔ Industrial Shredder | Manual | NEEDS HUMAN |
| 13 | Branded Pen fires on click; queue decrements | Manual | NEEDS HUMAN |
| 14 | Spam Filter sprays 5-pellet shotgun | Manual | NEEDS HUMAN |
| 15 | Drum Stick / Industrial Shredder feel different | Manual | NEEDS HUMAN |
| 16 | Neural Pulse (F) fires from headband | Manual | NEEDS HUMAN |
| 17 | Interns chase + melee damage player | Manual | NEEDS HUMAN |
| 18 | Account Executives fire shotgun blasts at distance | Manual | NEEDS HUMAN |
| 19 | Recruiters throw fireballs (visible orange projectile) | Manual | NEEDS HUMAN |
| 20 | HR Manager fires tracking projectiles (white) | Manual | NEEDS HUMAN |
| 21 | HR Manager dies after enough damage (200 HP) | Manual | NEEDS HUMAN |
| 22 | After cyc1_hr_arena exit: EndOfCycleStub appears with manifesto text | Manual | NEEDS HUMAN |
| 23 | Press-key on EndOfCycleStub returns to boot | Manual | NEEDS HUMAN |
| 24 | Ship It music plays on HR arena (different from lobby ambient) | Manual | NEEDS HUMAN |
| 25 | retro hit / damage SFX trigger on combat | Manual | NEEDS HUMAN |

## Phase 2 Definition of Done

[ ] Automated DoD items pass (Playwright)
[ ] Manual DoD items verified by a human in a browser

## Notes / observations

[Anything weird, edge cases, balance issues, sprite oddness etc.]
```

Mark Playwright items based on actual e2e run. Manual items remain "NEEDS HUMAN" until verified.

- [ ] **Step 2: Commit**

```bash
cd /Users/justinwest/Repos/LOOM-DOOM/.claude/worktrees/awesome-sutherland-aad854
git add docs/playtests/2026-04-27-phase-2-cycle-1.md
git commit -m "Add Phase 2 Cycle 1 playtest log

Covers 25-item checklist: 8 items automated via Playwright (boot,
lobby spawn, HUD zone, GameLoop tick, exit trigger, intermission
advance, no console errors), 17 items NEEDS HUMAN (visual rendering,
controls feel, weapon variety, AI behavior, boss combat, music
transition, SFX)."
```

---

# Self-review

**Spec coverage** (against `docs/superpowers/specs/2026-04-27-loom-doom-design.md`, §8 Phase 2):

| Spec requirement | Task |
|---|---|
| 3 maps: cyc1_lobby, cyc1_cubicles, cyc1_hr_arena | 2.13, 2.14, 2.15 |
| Weapons: Drum Stick, Industrial Shredder, Branded Pen, Spam Filter | 2.6, 2.7, (Pen from Phase 1), 2.8 |
| Enemies: Intern, Account Executive, Recruiter, HR Manager (boss) | (Intern from Phase 1, melee added in 2.1), 2.10, 2.11, 2.12 |
| Cycle-1 intermission text screens | 2.5 |
| Episode-1 boss death → cycle 2 transition stub | 2.5 (EndOfCycleStub) + 2.4 (campaign isLastMapOfCycle) |
| One real JOIN OS track: Ship It on the HR boss arena | 2.16 |
| Cycle 1 plays continuously start-to-finish | 2.5 (state machine) |
| Phase 2 DoD: shippable as public demo | 2.17 (Playwright) + 2.18 (playtest log) |

**Phase 1 review punch list pickups:**
| Item | Task |
|---|---|
| I8 — Extract Enemy interface | 2.1 |
| I1 — Stabilize getSnapshot identity | 2.2 |
| Phase 1 gap: Interns can't damage player | 2.1 (melee added) |
| Other Phase 1 review I-items (I4, I5, I6, I7, M1-M12) | Deferred to Phase 3 setup or Phase 6 polish |

**Placeholder scan:** No "TBD", "TODO", "fill in details" patterns in the plan. The "real LOOM-specific soundbanks come Phase 6" comment in audioController is a forward-reference, not a placeholder. The Phase 2 fairness loophole (player can skip HR Manager by walking past) is documented as accepted scope.

**Type consistency:**
- `IEnemy`, `EnemyState`, `EnemyUpdateContext` — defined in 2.1, expanded in 2.9 (audio added in 2.16)
- `IWeapon.fire(player, map, enemies, audio)` — final signature settled by 2.16
- `IProjectile` — defined in 2.9, used in 2.11, 2.12
- `weaponsBySlot: Map<number, IWeapon[]>`, `slotIndex: Map<number, number>` — defined in 2.3, populated in 2.6/2.7/2.8
- `ThingType = 'player_start' | 'intern' | 'account_executive' | 'recruiter' | 'hr_manager' | 'exit'` — final after Tasks 2.10-2.12
- `GameSnapshot` — Phase 1 type, untouched by Phase 2 (no new HUD fields needed since pulse-charge UI is Phase 4)
- `LOOMGame` `GameState` discriminated union — defined in 2.5

All consistent across tasks.

---

# Done condition

Phase 2 complete when:
1. All Tasks 2.1–2.18 implementation steps done and committed
2. Task 2.17 Playwright spec passes (full boot → lobby → exit-trigger → intermission → cubicles)
3. Vitest passes (existing 9 LOOM tests + new campaign tests = 14 LOOM tests; full suite is whatever l0b0tonline's main app has + LOOM)
4. TSC clean
5. Playtest log file exists (manual items can remain NEEDS HUMAN if user can't visually verify in this environment)

Next plan after Phase 2: `2026-XX-XX-loom-phase-3.md` covering Cycle 2 (5 maps, escalation tier — PIP, Notification, Middle Manager, VP of Sales boss, Reply All Storm weapon, light HUD corruption mode).
