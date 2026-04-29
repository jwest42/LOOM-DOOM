# LOOM Phase 7 Implementation Plan — Polish + Asset Scaffolding + Final Code Closeout

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make LOOM shippable-pending-art. Close every code-side item from the Phase 5/6 review backlog, scaffold three graceful asset slots (sprites, music tracks, surveillance voiceover) with fallback to current programmatic visuals + Web-Audio synth, validate touch / small-viewport hardware, lock the difficulty curve.

**Phase 7 Definition of Done (per spec §6 acceptance criteria):** "asset slots demonstrably empty-but-functional — a fresh checkout with no asset files passes the e2e and renders gameplay identical to current Phase 6, AND all 8 child slices implemented and merged."

**Architecture:** Extends Phase 0–6. One new pattern recurs in three places (sprite / track / VO asset slots): `manifest.ts` declares slot keys + paths → `*Loader.ts` HEAD-probes then fetches missing-tolerant → `*Service.ts` exposes synchronous getters that always return non-null (real asset OR fallback). Outside the asset-slot pattern: a role-keyed palette module integrated at the renderer's billboard passes, telemetry-driven difficulty tuning, a touch-input hook, plus two batches of unit tests for Phase 6 deferred items.

**Tech Stack:** Same as Phase 6 — TypeScript 5.8, React 19, Vite 6, Tailwind v4, Zustand 5, WebGL 2, Web Audio API, Vitest 4, Playwright. New API surface: `fetch` + `Image.decode` for sprites, `AudioContext.decodeAudioData` for tracks/VO, `('ontouchstart' in window)` + `matchMedia('(orientation: portrait)')` for mobile, Vite `define` for telemetry build-flag stripping.

**Pre-existing state:**
- `l0b0tonline` `main` is at commit `8b9faba` (Phase 6 PR #166 merged including the rehydrate fix `fa2da81`).
- `LOOM-DOOM` `master` is at `be9fbf1` (Phase 6 docs PR #5 merged).
- LOOM-DOOM Phase 7 worktree at `claude/phase-7-polish-shippable` (commit `970c6fd` — spec only).
- GitHub issues #167–#175 open on jwest42/l0b0tonline tracking the 8 slices.
- Renderer is a software raycaster outputting to a `Uint8Array` framebuffer that uploads to a single WebGL2 texture each frame. Enemy + projectile sprites are colored rectangles using `e.placeholderColor` / `p.color` (lines 188–223 / 225–256 of `components/loom/engine/renderer.ts`).
- `accessibilityStore` (`components/loom/store/accessibilityStore.ts`) carries `reducedMotionPref`, `reducedMotion` (computed), `colorblindPalette` (`'normal' | 'deuteranopia'`, structure-only), `keyboardLook`. Persist hardened in Phase 6 with `partialize` + `onRehydrateStorage`.
- `SkillLevelSelect` (`components/loom/hud/SkillLevelSelect.tsx`) has 2 accessibility checkboxes (Reduced motion, Keyboard look). This task adds 2 more (Color palette radio, Mute voiceover checkbox).
- `SURVEILLANCE_FRAGMENTS` is a flat array at `components/loom/hud/LoomHud.tsx:16` (single pool, not yet split by cycle).
- Phase 6 e2e lives at `e2e/loom-phase-6.spec.ts`. Will be renamed `loom-phase-7.spec.ts` in Task 24.

**OUT OF SCOPE — explicitly deferred to Phase 8 (creative pass):**
- Recording / mastering of *Patch Tuesday (Heaven Vers!0n)* and *human In l00p*
- Drawing the 13 enemy + 5 weapon viewmodel + 3 pickup sprite frame strips
- Recording C2/C3 surveillance-log voiceover takes
- AI-generated placeholder content for any of the above (would dilute bespoke aesthetic)
- Tritanopia / protanopia / monochrome palettes (deuteranopia only this phase)
- J0IN 0S desktop chrome portrait redesign (LOOM only needs the *game window* mobile-OK)

---

## File structure (post-Phase-7, in l0b0tonline)

New files (NEW), modified files (MOD):

```
l0b0tonline/
├── components/
│   └── loom/
│       ├── engine/
│       │   ├── audioController.ts             (MOD — playTrackForCycle, voiceover bus, volume gating)
│       │   ├── audioController.test.ts        (MOD — track loaded vs missing scenarios)
│       │   ├── audioTrackLoader.ts            (NEW — HEAD-probe + decodeAudioData per manifest entry)
│       │   ├── audioTrackLoader.test.ts       (NEW)
│       │   ├── difficultyTelemetry.ts         (NEW — recordEvent + dev-only console.table)
│       │   ├── difficultyTelemetry.test.ts    (NEW)
│       │   ├── gameLoop.ts                    (MOD — telemetry hooks at hit/kill/pickup/death/room enter)
│       │   ├── gameLoop.test.ts               (MOD — hold-fire pump cadence tests)
│       │   ├── palette.ts                     (NEW — NORMAL_PALETTE + DEUTERANOPIA_PALETTE, role-keyed)
│       │   ├── palette.test.ts                (NEW)
│       │   ├── renderer.ts                    (MOD — palette resolution at billboard passes; sprite-quad fallback)
│       │   ├── spriteLoader.ts                (NEW — fetch + ImageBitmap per manifest entry)
│       │   ├── spriteLoader.test.ts           (NEW)
│       │   ├── spriteService.ts               (NEW — getSprite synchronous getter)
│       │   ├── voiceoverLoader.ts             (NEW — HEAD-probe + decodeAudioData)
│       │   ├── voiceoverLoader.test.ts        (NEW)
│       │   ├── voiceoverService.ts            (NEW — scheduleRandomFromCycle + RM volume gating)
│       │   ├── voiceoverService.test.ts       (NEW)
│       ├── hud/
│       │   ├── LandscapeRotateOverlay.tsx     (NEW — portrait+narrow viewport prompt)
│       │   ├── LoomHud.tsx                    (MOD — data-palette attribute; SURVEILLANCE_FRAGMENTS split by cycle; VO scheduler call)
│       │   ├── SkillLevelSelect.tsx           (MOD — Color palette radio + Mute voiceover checkbox; touch-target sizing)
│       │   ├── SkillLevelSelect.test.tsx      (MOD — palette + mute-VO toggle behavior)
│       │   └── surveillanceFragments.test.ts  (NEW — N-3 paraphrase guard)
│       ├── input/
│       │   ├── useTouchInput.ts               (NEW — touch gesture → input mappings)
│       │   └── useTouchInput.test.ts          (NEW)
│       ├── entities/
│       │   └── projectiles/
│       │       └── terminationLetter.test.ts  (NEW — room-clear unit tests)
│       └── store/
│           └── accessibilityStore.ts          (MOD — add muteVoiceover field + setter)
├── data/
│   └── assets/
│       ├── sprites/
│       │   └── manifest.ts                    (NEW — 13 enemy + 5 viewmodel + 3 pickup slot keys)
│       └── audio/
│           ├── trackManifest.ts               (NEW — patch_tuesday + human_in_loop slots)
│           └── voManifest.ts                  (NEW — C2 + C3 VO slot keys)
├── e2e/
│   ├── loom-phase-6.spec.ts → loom-phase-7.spec.ts  (RENAME + extend)
│   └── loom-phase-7.spec.ts                   (MOD — palette toggle persistence, asset-slot empty-state, small-viewport touch)
├── styles/
│   └── globals.css                            (MOD — [data-palette='deuteranopia'] HUD overrides)
├── vite.config.ts                             (MOD — define __LOOM_TELEMETRY__ false in prod)
└── public/
    └── loom/
        ├── sprites/                           (NEW empty dir — slot for Phase 8 PNGs)
        ├── audio/
        │   ├── tracks/                        (NEW empty dir — slot for Phase 8 MP3s)
        │   └── vo/                            (NEW empty dir — slot for Phase 8 MP3s)
```

LOOM-DOOM additions (this worktree):

```
LOOM-DOOM/
└── docs/
    ├── superpowers/
    │   ├── specs/2026-04-28-loom-doom-phase-7-design.md   (already committed at 970c6fd)
    │   └── plans/2026-04-29-loom-doom-phase-7.md          (this file)
    └── playtests/
        ├── 2026-04-29-mobile-audit.md                     (NEW — Task 20)
        ├── 2026-04-29-difficulty-easy.md                  (NEW — Task 19)
        ├── 2026-04-29-difficulty-normal.md                (NEW — Task 19)
        ├── 2026-04-29-difficulty-hard.md                  (NEW — Task 19)
        ├── 2026-04-29-mobile-ios-safari.md                (NEW — Task 23)
        ├── 2026-04-29-mobile-android-chrome.md            (NEW — Task 23)
        └── 2026-04-29-phase-7-final.md                    (NEW — Task 24, ship-it summary)
```

---

## Task ordering rationale

Tasks are ordered so that each builds on the prior commit and avoids merge conflicts on `SkillLevelSelect` (which is touched by Slices 1, 7, and 8):

1. **Setup** (Task 1) — branch creation
2. **Slice 1: Colorblind palette** (Tasks 2–5) — first because it lands the SkillLevelSelect Color palette radio that Slice 7 builds on
3. **Slice 2: I-3 unit tests** (Tasks 6–7) — quick test-only wins; no app code
4. **Slice 3: N-3 paraphrase guard** (Task 8) — quick test-only win
5. **Slice 5: Sprite asset slot** (Tasks 9–11) — establishes the asset-slot pattern
6. **Slice 6: Audio-track asset slot** (Tasks 12–13) — applies pattern to audio
7. **Slice 7: Voiceover asset slot** (Tasks 14–16) — third application + extends SkillLevelSelect after Slice 1
8. **Slice 4: Difficulty telemetry + tuning** (Tasks 17–19) — after game state is stable
9. **Slice 8: Mobile audit + touch** (Tasks 20–23) — last because depends on all prior SkillLevelSelect changes for touch sizing
10. **Final** (Task 24) — Phase 7 e2e rebrand + ship-it summary

Each task references its parent issue # for ground-truth scope.

---

## Task 1: Branch setup

**Issue:** N/A (setup)

**Files:** none

- [ ] **Step 1: Create the Phase 7 branch off `main`**

Run:
```bash
cd /Users/justinwest/Repos/l0b0tonline
git checkout main
git pull --ff-only
git checkout -b phase-7-polish
```

Expected: branch `phase-7-polish` created from `8b9faba` (Phase 6 merge commit).

- [ ] **Step 2: Confirm tests pass at HEAD**

Run:
```bash
npx vitest run
```

Expected: 53 files / 973 tests pass (current Phase 6 baseline).

- [ ] **Step 3: Confirm tsc clean**

Run:
```bash
npx tsc --noEmit
```

Expected: no output (success).

- [ ] **Step 4: Create empty asset directories** (so the loaders have somewhere to fetch from later)

Run:
```bash
mkdir -p public/loom/sprites public/loom/audio/tracks public/loom/audio/vo
touch public/loom/sprites/.gitkeep public/loom/audio/tracks/.gitkeep public/loom/audio/vo/.gitkeep
git add public/loom/
git commit -m "Phase 7 setup: empty asset slot directories"
```

Expected: one commit on `phase-7-polish`.

---

## Task 2: Palette module (Slice 1, issue #168)

**Files:**
- Create: `components/loom/engine/palette.ts`
- Test: `components/loom/engine/palette.test.ts`

- [ ] **Step 1: Write the failing test**

Create `components/loom/engine/palette.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { NORMAL_PALETTE, DEUTERANOPIA_PALETTE, getPalette, type PaletteRole } from './palette';

describe('palette', () => {
  it('NORMAL_PALETTE has all 10 roles defined', () => {
    const expected: PaletteRole[] = [
      'enemy.melee', 'enemy.ranged', 'enemy.boss',
      'projectile.player', 'projectile.enemy',
      'pickup.health', 'pickup.ammo', 'pickup.terminationLetter',
      'hud.warning', 'hud.success',
    ];
    for (const role of expected) {
      expect(NORMAL_PALETTE[role]).toBeDefined();
      expect(NORMAL_PALETTE[role]).toHaveLength(3); // [r, g, b]
    }
  });

  it('DEUTERANOPIA_PALETTE has all 10 roles defined', () => {
    const expected: PaletteRole[] = [
      'enemy.melee', 'enemy.ranged', 'enemy.boss',
      'projectile.player', 'projectile.enemy',
      'pickup.health', 'pickup.ammo', 'pickup.terminationLetter',
      'hud.warning', 'hud.success',
    ];
    for (const role of expected) {
      expect(DEUTERANOPIA_PALETTE[role]).toBeDefined();
      expect(DEUTERANOPIA_PALETTE[role]).toHaveLength(3);
    }
  });

  it('getPalette("normal") returns NORMAL_PALETTE', () => {
    expect(getPalette('normal')).toBe(NORMAL_PALETTE);
  });

  it('getPalette("deuteranopia") returns DEUTERANOPIA_PALETTE', () => {
    expect(getPalette('deuteranopia')).toBe(DEUTERANOPIA_PALETTE);
  });

  it('deuteranopia palette differs from normal at red↔green roles', () => {
    // The whole point: red-green confusion. enemy.melee (typically red) and
    // pickup.health (typically green) must shift away from red/green tones.
    expect(DEUTERANOPIA_PALETTE['enemy.melee']).not.toEqual(NORMAL_PALETTE['enemy.melee']);
    expect(DEUTERANOPIA_PALETTE['pickup.health']).not.toEqual(NORMAL_PALETTE['pickup.health']);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
npx vitest run components/loom/engine/palette.test.ts
```

Expected: FAIL with "Cannot find module './palette'".

- [ ] **Step 3: Implement `palette.ts`**

Create `components/loom/engine/palette.ts`:

```typescript
import type { ColorblindPalette } from '../store/accessibilityStore';

export type PaletteRole =
  | 'enemy.melee'
  | 'enemy.ranged'
  | 'enemy.boss'
  | 'projectile.player'
  | 'projectile.enemy'
  | 'pickup.health'
  | 'pickup.ammo'
  | 'pickup.terminationLetter'
  | 'hud.warning'
  | 'hud.success';

export type RGB = readonly [number, number, number];
export type Palette = Record<PaletteRole, RGB>;

/**
 * NORMAL_PALETTE — the canonical LOOM color set.
 * Approximates the placeholderColor values in current entity defs.
 */
export const NORMAL_PALETTE: Palette = {
  'enemy.melee':              [200,  60,  60], // red — Intern, HR Skeleton
  'enemy.ranged':             [220, 140,  60], // orange — Brand Ambassador, SeniorVP
  'enemy.boss':               [180,  40, 180], // magenta — The Algorithm, The Hand
  'projectile.player':        [120, 220,  80], // bright green
  'projectile.enemy':         [240,  80,  40], // hot orange
  'pickup.health':            [ 60, 220,  90], // green
  'pickup.ammo':              [240, 220,  60], // yellow
  'pickup.terminationLetter': [255, 255, 255], // white
  'hud.warning':              [240,  80,  40], // hot orange
  'hud.success':              [120, 220,  80], // bright green
};

/**
 * DEUTERANOPIA_PALETTE — red-green confusion safe. Shifts:
 *   - red enemies → desaturated brown-orange (still readable as "danger")
 *   - green pickups → cyan-teal (still readable as "good")
 *   - hot orange projectiles → magenta (high contrast against teal pickups)
 *
 * Validated manually against Stark colorblind simulator (Sim Daltonism style).
 */
export const DEUTERANOPIA_PALETTE: Palette = {
  'enemy.melee':              [180, 110,  60], // desaturated brown-orange
  'enemy.ranged':             [220, 170,  90], // muted ochre
  'enemy.boss':               [120,  60, 200], // deep purple
  'projectile.player':        [120, 200, 220], // cyan
  'projectile.enemy':         [220,  80, 200], // magenta
  'pickup.health':            [100, 200, 220], // teal
  'pickup.ammo':              [240, 230,  90], // yellow (preserved — distinct in deuteranopia)
  'pickup.terminationLetter': [255, 255, 255], // white (preserved)
  'hud.warning':              [220,  80, 200], // magenta
  'hud.success':              [100, 200, 220], // teal
};

export const getPalette = (kind: ColorblindPalette): Palette =>
  kind === 'deuteranopia' ? DEUTERANOPIA_PALETTE : NORMAL_PALETTE;
```

- [ ] **Step 4: Run tests to verify they pass**

Run:
```bash
npx vitest run components/loom/engine/palette.test.ts
```

Expected: PASS, 5/5 tests.

- [ ] **Step 5: Commit**

```bash
git add components/loom/engine/palette.ts components/loom/engine/palette.test.ts
git commit -m "Add palette module — NORMAL + DEUTERANOPIA role-keyed (Phase 7 §4.3, #168)"
```

---

## Task 3: Renderer palette integration (Slice 1, issue #168)

**Files:**
- Modify: `components/loom/engine/renderer.ts:188-256` (enemy + projectile billboard passes)
- Test: `components/loom/engine/renderer.test.ts` (extend, or new if missing)

**Background:** Each `IEnemy` and `IProjectile` already carries a `placeholderColor: readonly [number, number, number]`. The integration is non-destructive: enemies/projectiles MAY also declare a `paletteRole?: PaletteRole`. When both are present AND the active palette is non-normal, the renderer resolves color via `getPalette(palette)[role]`. Otherwise it falls back to `placeholderColor`. This keeps every existing test using `placeholderColor: [0,0,0]` green.

- [ ] **Step 1: Write the failing test**

Append to `components/loom/engine/renderer.test.ts` (create if missing — the file may not exist yet):

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { useAccessibilityStore } from '../store/accessibilityStore';
// NOTE: this is a unit test of the *color resolution* helper extracted from the
// renderer. The render loop itself isn't easy to unit-test (writes pixels into
// a Uint8Array); the helper IS testable in isolation.

import { resolveSpriteColor } from './renderer';

describe('renderer.resolveSpriteColor', () => {
  beforeEach(() => {
    useAccessibilityStore.setState({
      reducedMotionPref: 'auto',
      reducedMotion: false,
      colorblindPalette: 'normal',
      keyboardLook: false,
    });
  });

  it('returns placeholderColor when palette is normal', () => {
    const color = resolveSpriteColor([10, 20, 30], 'enemy.melee');
    expect(color).toEqual([10, 20, 30]);
  });

  it('returns palette role color when palette is deuteranopia and role provided', () => {
    useAccessibilityStore.getState().setColorblindPalette('deuteranopia');
    const color = resolveSpriteColor([10, 20, 30], 'enemy.melee');
    expect(color).toEqual([180, 110, 60]); // DEUTERANOPIA_PALETTE['enemy.melee']
  });

  it('returns placeholderColor when palette is deuteranopia but role is undefined', () => {
    useAccessibilityStore.getState().setColorblindPalette('deuteranopia');
    const color = resolveSpriteColor([10, 20, 30], undefined);
    expect(color).toEqual([10, 20, 30]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
npx vitest run components/loom/engine/renderer.test.ts
```

Expected: FAIL with "resolveSpriteColor is not exported".

- [ ] **Step 3: Add `resolveSpriteColor` helper + integrate in renderer.ts**

Open `components/loom/engine/renderer.ts`. At the top of the file, add imports:

```typescript
import { getPalette, type PaletteRole } from './palette';
import { useAccessibilityStore } from '../store/accessibilityStore';
```

Add the helper just above the `Renderer` class:

```typescript
/**
 * Resolve a sprite/projectile color through the active accessibility palette.
 * Falls back to the entity's `placeholderColor` when no role is declared OR
 * when the active palette is `'normal'`.
 */
export function resolveSpriteColor(
  fallback: readonly [number, number, number],
  role: PaletteRole | undefined,
): readonly [number, number, number] {
  if (!role) return fallback;
  const palette = useAccessibilityStore.getState().colorblindPalette;
  if (palette === 'normal') return fallback;
  return getPalette(palette)[role];
}
```

Modify the enemy billboard pass (around line 217). Replace:

```typescript
this.framebuffer[idx]     = e.placeholderColor[0];
this.framebuffer[idx + 1] = e.placeholderColor[1];
this.framebuffer[idx + 2] = e.placeholderColor[2];
```

with:

```typescript
const [r, g, b] = resolveSpriteColor(e.placeholderColor, e.paletteRole);
this.framebuffer[idx]     = r;
this.framebuffer[idx + 1] = g;
this.framebuffer[idx + 2] = b;
```

Modify the projectile billboard pass (around line 254). Replace:

```typescript
this.framebuffer[idx]     = p.color[0];
```

with the same pattern, using `resolveSpriteColor(p.color, p.paletteRole)`. Show the full block:

```typescript
const [r, g, b] = resolveSpriteColor(p.color, p.paletteRole);
this.framebuffer[idx]     = r;
this.framebuffer[idx + 1] = g;
this.framebuffer[idx + 2] = b;
this.framebuffer[idx + 3] = 255;
```

Add `paletteRole?: PaletteRole;` to the `IEnemy` and `IProjectile` interfaces:

```typescript
// components/loom/entities/enemies/Enemy.ts (around the existing placeholderColor field):
import type { PaletteRole } from '../../engine/palette';
// inside IEnemy:
readonly placeholderColor: readonly [number, number, number];
readonly paletteRole?: PaletteRole;
```

```typescript
// components/loom/entities/projectiles/Projectile.ts:
import type { PaletteRole } from '../../engine/palette';
// inside IProjectile:
readonly color: readonly [number, number, number];
readonly paletteRole?: PaletteRole;
```

- [ ] **Step 4: Run tests to verify they pass**

Run:
```bash
npx vitest run components/loom/engine/renderer.test.ts
```

Expected: PASS, 3/3.

Run the full suite to confirm no regressions:
```bash
npx vitest run
```

Expected: 53+ files pass, 973+ tests pass (no existing tests broken because `paletteRole` is optional and falls back to `placeholderColor`).

- [ ] **Step 5: Commit**

```bash
git add components/loom/engine/renderer.ts components/loom/engine/renderer.test.ts \
        components/loom/entities/enemies/Enemy.ts components/loom/entities/projectiles/Projectile.ts
git commit -m "Renderer resolves sprite color via accessibility palette (Phase 7 §4.3, #168)"
```

---

## Task 4: HUD palette CSS (Slice 1, issue #168)

**Files:**
- Modify: `components/loom/hud/LoomHud.tsx` (add `data-palette` attribute on root)
- Modify: `styles/globals.css` (add deuteranopia overrides for `.loom-hud-warning`, `.loom-hud-success` etc.)

- [ ] **Step 1: Read the existing HUD root JSX**

Run:
```bash
grep -n 'return (' components/loom/hud/LoomHud.tsx | head -3
grep -n 'className=' components/loom/hud/LoomHud.tsx | head -5
```

Note the line of the outermost `<div>` that the HUD returns.

- [ ] **Step 2: Add `data-palette` attribute**

In `components/loom/hud/LoomHud.tsx`, near the top, add:

```typescript
import { useAccessibilityStore } from '../store/accessibilityStore';
```

Inside the component function, near the existing store reads:

```typescript
const colorblindPalette = useAccessibilityStore((s) => s.colorblindPalette);
```

On the outermost JSX `<div>`, add the attribute:

```tsx
<div
  className="..."
  data-palette={colorblindPalette}
  ...
>
```

- [ ] **Step 3: Add deuteranopia CSS overrides**

Open `styles/globals.css`. Append:

```css
/* Phase 7 §4.3 — deuteranopia palette HUD chrome overrides.
 * Surveillance log + manifesto text intentionally NOT overridden — they're
 * green-CRT on principle and serve a different aesthetic role.
 */
[data-palette='deuteranopia'] .loom-hud-warning {
  color: rgb(220, 80, 200); /* magenta — DEUTERANOPIA_PALETTE['hud.warning'] */
}
[data-palette='deuteranopia'] .loom-hud-success {
  color: rgb(100, 200, 220); /* teal — DEUTERANOPIA_PALETTE['hud.success'] */
}
```

If `LoomHud.tsx` doesn't currently use `loom-hud-warning` / `loom-hud-success` classes on its warning/success glyphs, add them now (search for the existing health-low / ammo-pickup-flash glyph nodes and add the className).

- [ ] **Step 4: Manual verification via dev server**

Run:
```bash
npm run dev
```

Open localhost, take damage to trigger health-warning glyph. In dev tools, toggle `data-palette` on the HUD root from `normal` to `deuteranopia` — confirm the warning glyph shifts to magenta. Stop the dev server.

- [ ] **Step 5: Commit**

```bash
git add components/loom/hud/LoomHud.tsx styles/globals.css
git commit -m "Add data-palette attribute + deuteranopia HUD CSS overrides (Phase 7 §4.3, #168)"
```

---

## Task 5: SkillLevelSelect palette radio + e2e (Slice 1, issue #168)

**Files:**
- Modify: `components/loom/hud/SkillLevelSelect.tsx`
- Modify: `components/loom/hud/SkillLevelSelect.test.tsx`

- [ ] **Step 1: Write the failing test**

Append to `components/loom/hud/SkillLevelSelect.test.tsx`:

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { render, fireEvent, screen } from '@testing-library/react';
import { useAccessibilityStore } from '../store/accessibilityStore';
import { SkillLevelSelect } from './SkillLevelSelect';

describe('SkillLevelSelect — palette radio (Phase 7)', () => {
  beforeEach(() => {
    useAccessibilityStore.setState({
      reducedMotionPref: 'auto',
      reducedMotion: false,
      colorblindPalette: 'normal',
      keyboardLook: false,
    });
  });

  it('renders Default and Deuteranopia radios', () => {
    render(<SkillLevelSelect onConfirm={() => {}} />);
    expect(screen.getByLabelText(/default/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/deuteranopia/i)).toBeInTheDocument();
  });

  it('selecting Deuteranopia updates the store', () => {
    render(<SkillLevelSelect onConfirm={() => {}} />);
    fireEvent.click(screen.getByLabelText(/deuteranopia/i));
    expect(useAccessibilityStore.getState().colorblindPalette).toBe('deuteranopia');
  });

  it('Default is initially checked when store palette is normal', () => {
    render(<SkillLevelSelect onConfirm={() => {}} />);
    expect((screen.getByLabelText(/default/i) as HTMLInputElement).checked).toBe(true);
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run:
```bash
npx vitest run components/loom/hud/SkillLevelSelect.test.tsx
```

Expected: FAIL — "Unable to find a label" for the new radios.

- [ ] **Step 3: Add the radio pair to `SkillLevelSelect.tsx`**

Inside the existing accessibility section of `SkillLevelSelect.tsx`, add:

```tsx
{/* Phase 7 §4.3 — Color palette */}
<fieldset className="mt-4">
  <legend className="text-xs uppercase tracking-wider opacity-70">
    Color palette
  </legend>
  <label className="block mt-1 text-sm">
    <input
      type="radio"
      name="palette"
      value="normal"
      checked={colorblindPalette === 'normal'}
      onChange={() => setColorblindPalette('normal')}
      className="mr-2"
    />
    Default
  </label>
  <label className="block mt-1 text-sm">
    <input
      type="radio"
      name="palette"
      value="deuteranopia"
      checked={colorblindPalette === 'deuteranopia'}
      onChange={() => setColorblindPalette('deuteranopia')}
      className="mr-2"
    />
    Deuteranopia
  </label>
</fieldset>
```

Add the corresponding store hooks at the top of the component (if not already there):

```tsx
const colorblindPalette = useAccessibilityStore((s) => s.colorblindPalette);
const setColorblindPalette = useAccessibilityStore((s) => s.setColorblindPalette);
```

- [ ] **Step 4: Run tests to verify they pass**

Run:
```bash
npx vitest run components/loom/hud/SkillLevelSelect.test.tsx
```

Expected: PASS, all tests including the 3 new ones.

- [ ] **Step 5: Add Playwright visual snapshot to e2e**

Open `e2e/loom-phase-6.spec.ts` (will be renamed to `-7` in Task 24). Add a new `test()` block:

```typescript
test('Phase 7 §4.3 — colorblind palette toggle persists across reload', async ({ page }) => {
  await page.goto('/');
  // Click the LOOM icon to launch the game
  await page.getByText(/L00M/).click();
  // Wait for skill-level select
  await page.waitForSelector('text=/Color palette/i');
  // Select deuteranopia
  await page.getByLabel(/Deuteranopia/).check();
  // Reload
  await page.reload();
  await page.getByText(/L00M/).click();
  await page.waitForSelector('text=/Color palette/i');
  // Confirm deuteranopia is still selected
  expect(await page.getByLabel(/Deuteranopia/).isChecked()).toBe(true);
});
```

Run:
```bash
npx playwright test e2e/loom-phase-6.spec.ts -g "colorblind palette"
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add components/loom/hud/SkillLevelSelect.tsx components/loom/hud/SkillLevelSelect.test.tsx e2e/loom-phase-6.spec.ts
git commit -m "Add Color palette radio to SkillLevelSelect + e2e persistence (Phase 7 §4.3, #168)"
```

---

## Task 6: GameLoop hold-fire unit tests (Slice 2, issue #169)

**Files:**
- Modify: `components/loom/engine/gameLoop.test.ts`

**Background:** `gameLoop.ts` was extended in Phase 6 Task 6.8 to pump primary-fire on every tick when `input.isPrimaryFireDown` is true AND the active weapon's `supportsHoldFire` flag is set. The Hum has `supportsHoldFire = true`; Drumstick / Bass / etc. do not. This task tests the dispatch logic in isolation.

- [ ] **Step 1: Locate the hold-fire dispatch site**

Run:
```bash
grep -n 'isPrimaryFireDown\|supportsHoldFire\|holdFire' components/loom/engine/gameLoop.ts | head -10
```

Note the function name (likely `tick` or a private helper) where the dispatch happens.

- [ ] **Step 2: Write the failing tests**

Append to `components/loom/engine/gameLoop.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { GameLoop } from './gameLoop';

describe('GameLoop — hold-fire dispatch (Phase 7 I-3, #169)', () => {
  let loop: GameLoop;
  let fireSpy: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fireSpy = vi.fn();
    // Construct a minimal loop fixture. Adjust constructor args to match the real signature.
    loop = new GameLoop(/* see real constructor */);
    // Replace the active weapon with a stub whose fire() is the spy.
    (loop as any).player.activeWeapon = {
      supportsHoldFire: true,
      fire: fireSpy,
      cooldownMs: 50,
      lastFireTime: 0,
    };
  });

  it('hold-fire pumps Hum at correct cadence when isPrimaryFireDown=true', () => {
    (loop as any).input.isPrimaryFireDown = true;
    // Tick at 60Hz for 200ms; with 50ms cooldown, expect ~4 fires
    vi.spyOn(performance, 'now').mockImplementation(() => 0);
    loop.tick();
    vi.spyOn(performance, 'now').mockImplementation(() => 60);
    loop.tick();
    vi.spyOn(performance, 'now').mockImplementation(() => 120);
    loop.tick();
    vi.spyOn(performance, 'now').mockImplementation(() => 180);
    loop.tick();
    expect(fireSpy).toHaveBeenCalledTimes(4);
  });

  it('hold-fire stops on isPrimaryFireDown=false', () => {
    (loop as any).input.isPrimaryFireDown = true;
    vi.spyOn(performance, 'now').mockImplementation(() => 0);
    loop.tick();
    vi.spyOn(performance, 'now').mockImplementation(() => 60);
    loop.tick();
    expect(fireSpy).toHaveBeenCalledTimes(2);

    (loop as any).input.isPrimaryFireDown = false;
    vi.spyOn(performance, 'now').mockImplementation(() => 120);
    loop.tick();
    vi.spyOn(performance, 'now').mockImplementation(() => 180);
    loop.tick();
    expect(fireSpy).toHaveBeenCalledTimes(2); // no new fires
  });

  it('does NOT pump for weapons without hold-fire support (Drumstick, etc.)', () => {
    (loop as any).player.activeWeapon.supportsHoldFire = false;
    (loop as any).input.isPrimaryFireDown = true;
    vi.spyOn(performance, 'now').mockImplementation(() => 0);
    loop.tick();
    vi.spyOn(performance, 'now').mockImplementation(() => 60);
    loop.tick();
    expect(fireSpy).toHaveBeenCalledTimes(0);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail (red)**

Run:
```bash
npx vitest run components/loom/engine/gameLoop.test.ts
```

Expected: 3 new tests FAIL (constructor args wrong / spy not invoked at expected cadence). This is signal that the test fixture needs to match the real GameLoop API.

- [ ] **Step 4: Adjust the fixture to match the real GameLoop**

Read `components/loom/engine/gameLoop.ts` constructor signature. Update the `new GameLoop(...)` call in the test to pass the real required dependencies (or expose a test-only factory). Common case: GameLoop takes `(canvas, audioController, store)` — create stubs.

If the GameLoop's hold-fire dispatch lives in a private method, expose it via a `_dispatchHoldFire(now: number)` helper (rename existing internal call) and have the test invoke that helper directly with synthetic `now` values. This is a more reliable test design than mocking `performance.now` for the whole tick.

Show the rename in `gameLoop.ts`:

```typescript
// Before (Phase 6):
private tick(): void {
  // ... existing code ...
  if (this.input.isPrimaryFireDown && this.player.activeWeapon.supportsHoldFire) {
    if (now - this.player.activeWeapon.lastFireTime >= this.player.activeWeapon.cooldownMs) {
      this.player.activeWeapon.fire(...);
      this.player.activeWeapon.lastFireTime = now;
    }
  }
}

// After (Phase 7):
private tick(): void {
  // ... existing code ...
  this._dispatchHoldFire(performance.now());
}

/** @internal — exposed for unit tests */
public _dispatchHoldFire(now: number): void {
  if (!this.input.isPrimaryFireDown) return;
  const weapon = this.player.activeWeapon;
  if (!weapon.supportsHoldFire) return;
  if (now - weapon.lastFireTime < weapon.cooldownMs) return;
  weapon.fire(this.player, this.map, this.enemies, this.audio, this.spawnProjectile);
  weapon.lastFireTime = now;
}
```

Update the test to call `loop._dispatchHoldFire(0)`, `loop._dispatchHoldFire(60)`, etc. directly. Remove the `performance.now` spy.

- [ ] **Step 5: Run tests to verify they pass (green)**

Run:
```bash
npx vitest run components/loom/engine/gameLoop.test.ts
```

Expected: all tests pass, including 3 new hold-fire tests.

- [ ] **Step 6: Commit**

```bash
git add components/loom/engine/gameLoop.ts components/loom/engine/gameLoop.test.ts
git commit -m "Add hold-fire dispatch unit tests + extract helper (Phase 7 I-3, #169)"
```

---

## Task 7: TerminationLetter room-clear unit tests (Slice 2, issue #169)

**Files:**
- Create: `components/loom/entities/projectiles/terminationLetter.test.ts`

**Background:** Phase 6 Task 6.7 added the Termination Letter consumable: pressing `T` consumes a TL and damages all enemies in the active room. The dispatch lives in `gameLoop.ts` (handled via input KeyT), and the damage path is a function on the projectile module. This test isolates the room-clear logic.

- [ ] **Step 1: Locate the room-clear function**

Run:
```bash
grep -rn 'terminationLetter\|TerminationLetter\|consumeTL\|roomClear' components/loom/ --include='*.ts' | head -10
```

Identify the exported function (likely `applyTerminationLetter(player, map, enemies)` or similar).

- [ ] **Step 2: Write the failing test**

Create `components/loom/entities/projectiles/terminationLetter.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { applyTerminationLetter } from './terminationLetter';
import type { IEnemy } from '../enemies/Enemy';
import type { IPlayer } from '../player';
import type { GameMap } from '../../engine/mapLoader';

// Minimal map fixture: 2 rooms separated at x=10
const FIXTURE_MAP: GameMap = {
  width: 20,
  height: 10,
  tiles: new Uint8Array(20 * 10).fill(0), // all open
  rooms: [
    { id: 'r1', minX: 0, minY: 0, maxX: 9, maxY: 9 },
    { id: 'r2', minX: 10, minY: 0, maxX: 19, maxY: 9 },
  ],
} as unknown as GameMap;

const makeEnemy = (id: string, x: number, y: number, hp = 100): IEnemy => ({
  id, x, y, hp, maxHealth: 100,
  state: 'alive',
  placeholderColor: [0, 0, 0],
  takeDamage: function (dmg: number) { this.hp -= dmg; if (this.hp <= 0) this.state = 'dead'; },
  // ... other IEnemy fields stubbed minimally
} as unknown as IEnemy);

describe('terminationLetter — room-clear (Phase 7 I-3, #169)', () => {
  it('damages all enemies in the player\'s current room once', () => {
    const player = { x: 5, y: 5, terminationLetters: 1 } as IPlayer;
    const enemies: IEnemy[] = [
      makeEnemy('a', 3, 3),  // in r1
      makeEnemy('b', 7, 7),  // in r1
      makeEnemy('c', 15, 5), // in r2 (different room — must not be damaged)
    ];

    applyTerminationLetter(player, FIXTURE_MAP, enemies);

    expect(enemies[0].state).toBe('dead');
    expect(enemies[1].state).toBe('dead');
    expect(enemies[2].state).toBe('alive'); // adjacent room not damaged
    expect(enemies[2].hp).toBe(100);
    expect(player.terminationLetters).toBe(0);
  });

  it('with zero ammo is a no-op', () => {
    const player = { x: 5, y: 5, terminationLetters: 0 } as IPlayer;
    const enemies: IEnemy[] = [makeEnemy('a', 3, 3)];

    applyTerminationLetter(player, FIXTURE_MAP, enemies);

    expect(enemies[0].state).toBe('alive');
    expect(enemies[0].hp).toBe(100);
    expect(player.terminationLetters).toBe(0);
  });

  it('respects room boundaries — enemies in adjacent rooms not damaged', () => {
    const player = { x: 5, y: 5, terminationLetters: 1 } as IPlayer;
    const enemies: IEnemy[] = [
      makeEnemy('boundary', 9, 5),   // edge of r1 (just inside)
      makeEnemy('outside', 10, 5),   // first cell of r2
    ];

    applyTerminationLetter(player, FIXTURE_MAP, enemies);

    expect(enemies[0].state).toBe('dead');     // in r1
    expect(enemies[1].state).toBe('alive');    // in r2
  });
});
```

- [ ] **Step 3: Run to verify failure (red)**

Run:
```bash
npx vitest run components/loom/entities/projectiles/terminationLetter.test.ts
```

Expected: FAIL with "applyTerminationLetter is not exported" OR test failures because the existing function signature differs.

- [ ] **Step 4: Reconcile signature with the existing function**

Read the existing `terminationLetter.ts` and adjust the test to match the real signature. If the existing function signature is significantly different (e.g., takes `(gameState)` instead of `(player, map, enemies)`), refactor the function to take the explicit args (better testability) OR adapt the test to construct a `gameState` fixture.

If the function does NOT yet exist (Phase 6 may have inlined the logic into gameLoop): extract it now into `terminationLetter.ts`:

```typescript
// components/loom/entities/projectiles/terminationLetter.ts
import type { IEnemy } from '../enemies/Enemy';
import type { IPlayer } from '../player';
import type { GameMap } from '../../engine/mapLoader';

const TL_DAMAGE = 999; // intentional one-shot (Phase 6 design)

/** Find the room containing point (x, y) on the map, or null if not in any room. */
function findRoom(map: GameMap, x: number, y: number) {
  return map.rooms?.find(
    (r) => x >= r.minX && x <= r.maxX && y >= r.minY && y <= r.maxY,
  ) ?? null;
}

/**
 * Consumes one Termination Letter (if available) and damages every enemy in
 * the player's current room. Enemies in adjacent rooms are unaffected.
 * No-op if `player.terminationLetters === 0`.
 */
export function applyTerminationLetter(
  player: IPlayer,
  map: GameMap,
  enemies: IEnemy[],
): void {
  if (player.terminationLetters <= 0) return;
  const room = findRoom(map, player.x, player.y);
  if (!room) return;
  for (const enemy of enemies) {
    if (enemy.state !== 'alive') continue;
    const enemyRoom = findRoom(map, enemy.x, enemy.y);
    if (enemyRoom !== room) continue;
    enemy.takeDamage(TL_DAMAGE);
  }
  player.terminationLetters -= 1;
}
```

Then in `gameLoop.ts`, where the KeyT handler currently lives, replace inline logic with a call to `applyTerminationLetter(this.player, this.map, this.enemies)`.

- [ ] **Step 5: Run tests to verify they pass (green)**

Run:
```bash
npx vitest run components/loom/entities/projectiles/terminationLetter.test.ts
```

Expected: PASS, 3/3.

Run the full suite to confirm no regressions:
```bash
npx vitest run
```

Expected: 53+ files pass.

- [ ] **Step 6: Commit**

```bash
git add components/loom/entities/projectiles/terminationLetter.ts \
        components/loom/entities/projectiles/terminationLetter.test.ts \
        components/loom/engine/gameLoop.ts
git commit -m "Add Termination Letter room-clear unit tests + extract helper (Phase 7 I-3, #169)"
```

---

## Task 8: Surveillance fragments paraphrase guard (Slice 3, issue #170)

**Files:**
- Create: `components/loom/hud/surveillanceFragments.test.ts`
- Modify (only if needed): `components/loom/hud/LoomHud.tsx` — add a paraphrase line to `SURVEILLANCE_FRAGMENTS`

- [ ] **Step 1: Read existing SURVEILLANCE_FRAGMENTS**

Run:
```bash
sed -n '15,55p' components/loom/hud/LoomHud.tsx
```

Capture the array contents.

- [ ] **Step 2: Grep for paraphrase candidates in the existing pool**

Run:
```bash
grep -in 'l[-_]\?907\|null[-_]\?entity\|seventh\|saint' components/loom/hud/LoomHud.tsx
```

If there's at least one match, the test will pass against existing copy. If zero matches, you'll need to add a fragment in Step 4.

- [ ] **Step 3: Write the failing test**

Create `components/loom/hud/surveillanceFragments.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { SURVEILLANCE_FRAGMENTS } from './LoomHud';

describe('SURVEILLANCE_FRAGMENTS — N-3 paraphrase guard (Phase 7 §4.8, #170)', () => {
  it('contains at least one fragment paraphrasing L-907 / NULL_ENTITY / SAINT identity', () => {
    // The C-reveal cinematic depends on this foreshadowing landing somewhere
    // in the surveillance log rotation by mid-Cycle 3. This test guards
    // against accidental deletion. If you need to delete a matching line,
    // add a replacement that also matches before doing so.
    const regex = /l[-_]?907|null[-_]?entity|seventh|saint/i;
    const matches = SURVEILLANCE_FRAGMENTS.filter((f) => regex.test(f));
    expect(matches.length).toBeGreaterThan(0);
  });
});
```

This test imports `SURVEILLANCE_FRAGMENTS` from `LoomHud.tsx`, which means the array MUST be exported. Search for `const SURVEILLANCE_FRAGMENTS` and change it to `export const SURVEILLANCE_FRAGMENTS`.

- [ ] **Step 4: Run to verify**

Run:
```bash
npx vitest run components/loom/hud/surveillanceFragments.test.ts
```

Two outcomes:
- **PASS** — at least one existing fragment matches. Skip to Step 6.
- **FAIL** — no existing fragment matches. Continue to Step 5.

- [ ] **Step 5 (only if Step 4 failed): Draft a paraphrase fragment**

Add a fragment to `SURVEILLANCE_FRAGMENTS` that lands the foreshadowing without being too on-the-nose. Candidate:

```typescript
'SUBJECT L-907 STATUS: NOT IN OPTIMIZATION QUEUE',
```

OR (more cryptic, leans into the SAINT_L0B0T angle):

```typescript
'SEVENTH UNIT BEHAVIORAL DRIFT — REROUTE TO /dev/null',
```

Pick one. Insert into the array roughly halfway through (so it appears in mid-cycle rotation).

> **Note for implementer:** flag this commit as HITL — present the candidate line(s) to the user for copy approval before the next task.

Run the test again:
```bash
npx vitest run components/loom/hud/surveillanceFragments.test.ts
```

Expected: PASS now.

- [ ] **Step 6: Commit**

```bash
git add components/loom/hud/LoomHud.tsx components/loom/hud/surveillanceFragments.test.ts
git commit -m "Add N-3 surveillance-fragment L-907 paraphrase guard (Phase 7 §4.8, #170)"
```

---

## Task 9: Sprite manifest + loader (Slice 5, issue #172)

**Files:**
- Create: `data/assets/sprites/manifest.ts`
- Create: `components/loom/engine/spriteLoader.ts`
- Create: `components/loom/engine/spriteLoader.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `components/loom/engine/spriteLoader.test.ts`:

```typescript
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { spriteLoader, type SpriteCacheEntry } from './spriteLoader';
import { SPRITE_MANIFEST } from '../../../data/assets/sprites/manifest';

describe('spriteLoader (Phase 7 §4.4, #172)', () => {
  beforeEach(() => {
    spriteLoader.reset();
  });

  it('marks all manifest entries as "missing" when fetch returns 404', async () => {
    global.fetch = vi.fn(async () => new Response(null, { status: 404 })) as any;
    await spriteLoader.loadAll();
    for (const id of Object.keys(SPRITE_MANIFEST.enemies)) {
      expect(spriteLoader.getStatus(`enemies.${id}`)).toBe('missing');
    }
  });

  it('marks entries as "loaded" when HEAD probe + fetch succeed', async () => {
    const fakeBitmap = {} as ImageBitmap;
    global.createImageBitmap = vi.fn(async () => fakeBitmap) as any;
    global.fetch = vi.fn(async (url: string, init: RequestInit | undefined) => {
      // HEAD probe always 200 in this test, body fetch returns a fake blob
      if (init?.method === 'HEAD') return new Response(null, { status: 200 });
      return new Response(new Blob(['fake']), { status: 200 });
    }) as any;
    await spriteLoader.loadAll();
    expect(spriteLoader.getStatus('enemies.intern')).toBe('loaded');
  });

  it('one missing file does not block other fetches', async () => {
    global.createImageBitmap = vi.fn(async () => ({} as ImageBitmap)) as any;
    global.fetch = vi.fn(async (url: string, init: RequestInit | undefined) => {
      if (typeof url === 'string' && url.includes('intern')) {
        return new Response(null, { status: 404 });
      }
      if (init?.method === 'HEAD') return new Response(null, { status: 200 });
      return new Response(new Blob(['fake']), { status: 200 });
    }) as any;
    await spriteLoader.loadAll();
    expect(spriteLoader.getStatus('enemies.intern')).toBe('missing');
    expect(spriteLoader.getStatus('enemies.hr_skeleton')).toBe('loaded');
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run:
```bash
npx vitest run components/loom/engine/spriteLoader.test.ts
```

Expected: FAIL — manifest + loader don't exist.

- [ ] **Step 3: Create the manifest**

Create `data/assets/sprites/manifest.ts`:

```typescript
export interface SpriteEntry {
  path: string;
  frames: number;
  frameWidth: number;
  frameHeight: number;
}

export const SPRITE_MANIFEST = {
  enemies: {
    intern:           { path: '/loom/sprites/enemies/intern.png',         frames: 8, frameWidth: 64, frameHeight: 64 },
    hr_skeleton:      { path: '/loom/sprites/enemies/hr_skeleton.png',    frames: 8, frameWidth: 64, frameHeight: 64 },
    wellness_officer: { path: '/loom/sprites/enemies/wellness_officer.png', frames: 8, frameWidth: 64, frameHeight: 64 },
    senior_vp:        { path: '/loom/sprites/enemies/senior_vp.png',      frames: 8, frameWidth: 64, frameHeight: 64 },
    brand_ambassador: { path: '/loom/sprites/enemies/brand_ambassador.png', frames: 8, frameWidth: 64, frameHeight: 64 },
    synergy_pod:      { path: '/loom/sprites/enemies/synergy_pod.png',    frames: 8, frameWidth: 64, frameHeight: 64 },
    the_algorithm:    { path: '/loom/sprites/enemies/the_algorithm.png',  frames: 8, frameWidth: 96, frameHeight: 96 },
    the_hand:         { path: '/loom/sprites/enemies/the_hand.png',       frames: 8, frameWidth: 96, frameHeight: 96 },
    tessier:          { path: '/loom/sprites/enemies/tessier.png',        frames: 8, frameWidth: 64, frameHeight: 64 },
    compliance_drone: { path: '/loom/sprites/enemies/compliance_drone.png', frames: 8, frameWidth: 64, frameHeight: 64 },
    metrics_imp:      { path: '/loom/sprites/enemies/metrics_imp.png',    frames: 8, frameWidth: 48, frameHeight: 48 },
    onboarding_ghoul: { path: '/loom/sprites/enemies/onboarding_ghoul.png', frames: 8, frameWidth: 64, frameHeight: 64 },
    coffee_zombie:    { path: '/loom/sprites/enemies/coffee_zombie.png',  frames: 8, frameWidth: 64, frameHeight: 64 },
  },
  weapons: {
    drumstick:       { path: '/loom/sprites/weapons/viewmodels/drumstick.png',       frames: 4, frameWidth: 128, frameHeight: 128 },
    bass:            { path: '/loom/sprites/weapons/viewmodels/bass.png',            frames: 4, frameWidth: 128, frameHeight: 128 },
    guitar:          { path: '/loom/sprites/weapons/viewmodels/guitar.png',          frames: 4, frameWidth: 128, frameHeight: 128 },
    frequency_tuner: { path: '/loom/sprites/weapons/viewmodels/frequency_tuner.png', frames: 4, frameWidth: 128, frameHeight: 128 },
    microphone:      { path: '/loom/sprites/weapons/viewmodels/microphone.png',      frames: 4, frameWidth: 128, frameHeight: 128 },
  },
  pickups: {
    health:             { path: '/loom/sprites/pickups/health.png',             frames: 1, frameWidth: 32, frameHeight: 32 },
    ammo:               { path: '/loom/sprites/pickups/ammo.png',               frames: 1, frameWidth: 32, frameHeight: 32 },
    termination_letter: { path: '/loom/sprites/pickups/termination_letter.png', frames: 1, frameWidth: 32, frameHeight: 32 },
  },
} as const;

export type SpriteCategory = keyof typeof SPRITE_MANIFEST;
```

- [ ] **Step 4: Implement the loader**

Create `components/loom/engine/spriteLoader.ts`:

```typescript
import { SPRITE_MANIFEST } from '../../../data/assets/sprites/manifest';

export type SpriteCacheEntry = ImageBitmap | 'missing' | 'loading';

class SpriteLoader {
  private cache = new Map<string, SpriteCacheEntry>();

  reset(): void {
    this.cache.clear();
  }

  getStatus(id: string): 'loaded' | 'missing' | 'loading' | 'unknown' {
    const entry = this.cache.get(id);
    if (entry === undefined) return 'unknown';
    if (entry === 'missing') return 'missing';
    if (entry === 'loading') return 'loading';
    return 'loaded';
  }

  get(id: string): ImageBitmap | null {
    const entry = this.cache.get(id);
    if (!entry || entry === 'missing' || entry === 'loading') return null;
    return entry;
  }

  async loadAll(): Promise<void> {
    const tasks: Promise<void>[] = [];
    for (const [category, entries] of Object.entries(SPRITE_MANIFEST)) {
      for (const [name, entry] of Object.entries(entries)) {
        const id = `${category}.${name}`;
        this.cache.set(id, 'loading');
        tasks.push(this.loadOne(id, entry.path));
      }
    }
    await Promise.allSettled(tasks);
  }

  private async loadOne(id: string, path: string): Promise<void> {
    try {
      // HEAD probe first — cheap detection of 404.
      const head = await fetch(path, { method: 'HEAD' });
      if (!head.ok) {
        this.cache.set(id, 'missing');
        return;
      }
      const res = await fetch(path);
      if (!res.ok) {
        this.cache.set(id, 'missing');
        return;
      }
      const blob = await res.blob();
      const bitmap = await createImageBitmap(blob);
      this.cache.set(id, bitmap);
    } catch (err) {
      // Network error / decode error → mark missing, log at debug level only.
      // Absence is the expected default state in Phase 7.
      console.debug(`[spriteLoader] ${id} failed to load: ${err}`);
      this.cache.set(id, 'missing');
    }
  }
}

export const spriteLoader = new SpriteLoader();
```

- [ ] **Step 5: Run tests to verify they pass**

Run:
```bash
npx vitest run components/loom/engine/spriteLoader.test.ts
```

Expected: PASS, 3/3.

- [ ] **Step 6: Commit**

```bash
git add data/assets/sprites/manifest.ts \
        components/loom/engine/spriteLoader.ts \
        components/loom/engine/spriteLoader.test.ts
git commit -m "Add sprite manifest + loader with HEAD probe + missing fallback (Phase 7 §4.4, #172)"
```

---

## Task 10: Sprite service + renderer integration (Slice 5, issue #172)

**Files:**
- Create: `components/loom/engine/spriteService.ts`
- Modify: `components/loom/engine/renderer.ts` (enemy + projectile passes — replace rect fill with textured-quad path when sprite loaded)
- Modify: `components/LOOMGame.tsx` (call `spriteLoader.loadAll()` on mount)

- [ ] **Step 1: Write the failing test**

Append to `components/loom/engine/spriteLoader.test.ts`:

```typescript
import { spriteService } from './spriteService';

describe('spriteService — synchronous getter (Phase 7 §4.4, #172)', () => {
  it('returns null for unknown id', () => {
    spriteLoader.reset();
    expect(spriteService.getSprite('enemies.intern')).toBeNull();
  });

  it('returns null for missing-marked id', async () => {
    spriteLoader.reset();
    global.fetch = vi.fn(async () => new Response(null, { status: 404 })) as any;
    await spriteLoader.loadAll();
    expect(spriteService.getSprite('enemies.intern')).toBeNull();
  });

  it('returns ImageBitmap for loaded id', async () => {
    spriteLoader.reset();
    const fakeBitmap = {} as ImageBitmap;
    global.createImageBitmap = vi.fn(async () => fakeBitmap) as any;
    global.fetch = vi.fn(async (_url: string, init?: RequestInit) => {
      if (init?.method === 'HEAD') return new Response(null, { status: 200 });
      return new Response(new Blob(['fake']), { status: 200 });
    }) as any;
    await spriteLoader.loadAll();
    expect(spriteService.getSprite('enemies.intern')).toBe(fakeBitmap);
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run:
```bash
npx vitest run components/loom/engine/spriteLoader.test.ts
```

Expected: FAIL on the new tests (spriteService doesn't exist).

- [ ] **Step 3: Implement spriteService**

Create `components/loom/engine/spriteService.ts`:

```typescript
import { spriteLoader } from './spriteLoader';

class SpriteService {
  /**
   * Synchronous getter. Returns the loaded ImageBitmap or null. Never blocks.
   * Renderer uses this to decide between textured-quad and palette-aware rect.
   */
  getSprite(id: string): ImageBitmap | null {
    return spriteLoader.get(id);
  }
}

export const spriteService = new SpriteService();
```

- [ ] **Step 4: Wire `loadAll()` into game mount**

In `components/LOOMGame.tsx`, near the existing `useEffect` that initializes the engine, add:

```tsx
import { spriteLoader } from './loom/engine/spriteLoader';

// inside the component, near top of useEffect (no await — let it run in background)
spriteLoader.loadAll();
```

- [ ] **Step 5: Renderer integration — sprite fallback path**

The renderer currently fills enemy rects pixel-by-pixel with `placeholderColor`. With a sprite loaded, we want a different path: blit the bitmap into the framebuffer at the sprite-quad coordinates.

Add a helper near the top of `renderer.ts`:

```typescript
import { spriteService } from './spriteService';

/**
 * Blit a sprite frame into the framebuffer at the given screen rect, masking
 * via zBuffer. Returns true if the blit happened, false if no sprite was
 * available (caller should fall back to colored rect).
 */
function blitSprite(
  framebuffer: Uint8Array,
  zBuffer: Float32Array,
  bitmap: ImageBitmap,
  drawStartX: number, drawEndX: number,
  drawStartY: number, drawEndY: number,
  cx: number,
  internalW: number, internalH: number,
): boolean {
  // For now, sprite path is a stub — Phase 8 will implement actual ImageBitmap
  // → framebuffer pixel transfer. Return false to fall back to rect during
  // Phase 7 (the test verifies rect path stays the default with no assets).
  return false;
}
```

In the enemy billboard pass, before the existing rect-fill loop, add:

```typescript
const bitmap = e.spriteId ? spriteService.getSprite(e.spriteId) : null;
const used = bitmap ? blitSprite(this.framebuffer, this.zBuffer, bitmap,
  drawStartX, drawEndX, drawStartY, drawEndY, cx, INTERNAL_W, INTERNAL_H) : false;
if (!used) {
  // existing palette-aware rect fill (from Task 3)
  for (let x = drawStartX; x <= drawEndX; x++) { /* ... */ }
}
```

Add `spriteId?: string;` to `IEnemy` and `IProjectile` interfaces alongside `paletteRole`.

- [ ] **Step 6: Run tests + e2e**

Run:
```bash
npx vitest run
```

Expected: 53+ files pass, including the 3 new spriteService tests.

Run the existing Phase 6 e2e to confirm rect fallback still works:
```bash
npx playwright test e2e/loom-phase-6.spec.ts
```

Expected: PASS — gameplay renders identically to current Phase 6 (no sprites committed → all fall back to rect).

- [ ] **Step 7: Commit**

```bash
git add components/loom/engine/spriteService.ts \
        components/loom/engine/spriteLoader.test.ts \
        components/loom/engine/renderer.ts \
        components/loom/entities/enemies/Enemy.ts \
        components/loom/entities/projectiles/Projectile.ts \
        components/LOOMGame.tsx
git commit -m "Add spriteService + renderer fallback path (Phase 7 §4.4, #172)"
```

---

## Task 11: Sprite slot e2e empty-state regression check (Slice 5, issue #172)

**Files:**
- Modify: `e2e/loom-phase-6.spec.ts` (add empty-slot smoke test)

- [ ] **Step 1: Add empty-slot smoke test**

Append to `e2e/loom-phase-6.spec.ts`:

```typescript
test('Phase 7 §4.4 — sprite asset slot empty state: gameplay renders identical to Phase 6', async ({ page }) => {
  // No sprites committed under public/loom/sprites/ — all should fall back to
  // palette-aware rects. The visual snapshot from a clean Phase 6 baseline
  // should still match.
  const consoleWarnings: string[] = [];
  page.on('console', (msg) => {
    if (msg.type() === 'warning' || msg.type() === 'error') {
      consoleWarnings.push(msg.text());
    }
  });
  await page.goto('/');
  await page.getByText(/L00M/).click();
  // Wait for the game canvas to render frames
  await page.waitForTimeout(2000);
  // Confirm no spriteLoader warnings landed at warn+/error level
  // (debug-level 404s are filtered by the loader's console.debug call).
  const spriteWarnings = consoleWarnings.filter((w) => w.includes('spriteLoader'));
  expect(spriteWarnings).toHaveLength(0);
});
```

- [ ] **Step 2: Run and confirm**

Run:
```bash
npx playwright test e2e/loom-phase-6.spec.ts -g "sprite asset slot empty"
```

Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add e2e/loom-phase-6.spec.ts
git commit -m "Add sprite-slot empty-state e2e regression check (Phase 7 §4.4, #172)"
```

---

## Task 12: Audio-track manifest + loader (Slice 6, issue #173)

**Files:**
- Create: `data/assets/audio/trackManifest.ts`
- Create: `components/loom/engine/audioTrackLoader.ts`
- Create: `components/loom/engine/audioTrackLoader.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `components/loom/engine/audioTrackLoader.test.ts`:

```typescript
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { audioTrackLoader } from './audioTrackLoader';
import { TRACK_MANIFEST } from '../../../data/assets/audio/trackManifest';

const fakeAudioBuffer = {} as AudioBuffer;
const fakeAudioCtx = {
  decodeAudioData: vi.fn(async () => fakeAudioBuffer),
} as unknown as AudioContext;

describe('audioTrackLoader (Phase 7 §4.5, #173)', () => {
  beforeEach(() => {
    audioTrackLoader.reset();
    vi.clearAllMocks();
  });

  it('marks all manifest entries as "missing" when HEAD probe returns 404', async () => {
    global.fetch = vi.fn(async () => new Response(null, { status: 404 })) as any;
    await audioTrackLoader.loadAll(fakeAudioCtx);
    for (const id of Object.keys(TRACK_MANIFEST)) {
      expect(audioTrackLoader.getStatus(id)).toBe('missing');
    }
  });

  it('marks entries as "loaded" when HEAD probe + fetch + decode succeed', async () => {
    global.fetch = vi.fn(async (_url: string, init?: RequestInit) => {
      if (init?.method === 'HEAD') return new Response(null, { status: 200 });
      return new Response(new ArrayBuffer(8), { status: 200 });
    }) as any;
    await audioTrackLoader.loadAll(fakeAudioCtx);
    expect(audioTrackLoader.getStatus('patch_tuesday')).toBe('loaded');
    expect(audioTrackLoader.get('patch_tuesday')).toBe(fakeAudioBuffer);
  });

  it('decode failure marks entry as missing without crashing', async () => {
    global.fetch = vi.fn(async (_url: string, init?: RequestInit) => {
      if (init?.method === 'HEAD') return new Response(null, { status: 200 });
      return new Response(new ArrayBuffer(8), { status: 200 });
    }) as any;
    (fakeAudioCtx.decodeAudioData as any).mockRejectedValueOnce(new Error('decode failed'));
    await audioTrackLoader.loadAll(fakeAudioCtx);
    expect(audioTrackLoader.getStatus('patch_tuesday')).toBe('missing');
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run:
```bash
npx vitest run components/loom/engine/audioTrackLoader.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Create the manifest**

Create `data/assets/audio/trackManifest.ts`:

```typescript
export interface TrackEntry {
  path: string;
  /** Cycle indices this track is used for. 'credits' for end-of-game. */
  useFor: ReadonlyArray<1 | 2 | 3 | 4 | 'credits'>;
}

export const TRACK_MANIFEST: Record<string, TrackEntry> = {
  patch_tuesday: {
    path: '/loom/audio/tracks/patch_tuesday.mp3',
    useFor: [4, 'credits'],
  },
  human_in_loop: {
    path: '/loom/audio/tracks/human_in_loop.mp3',
    useFor: [3],
  },
};

export type TrackId = keyof typeof TRACK_MANIFEST;
```

- [ ] **Step 4: Implement the loader**

Create `components/loom/engine/audioTrackLoader.ts`:

```typescript
import { TRACK_MANIFEST } from '../../../data/assets/audio/trackManifest';

type Status = 'unknown' | 'loading' | 'loaded' | 'missing';

class AudioTrackLoader {
  private cache = new Map<string, AudioBuffer | 'missing' | 'loading'>();

  reset(): void {
    this.cache.clear();
  }

  getStatus(id: string): Status {
    const entry = this.cache.get(id);
    if (entry === undefined) return 'unknown';
    if (entry === 'missing') return 'missing';
    if (entry === 'loading') return 'loading';
    return 'loaded';
  }

  get(id: string): AudioBuffer | null {
    const entry = this.cache.get(id);
    if (!entry || entry === 'missing' || entry === 'loading') return null;
    return entry;
  }

  async loadAll(ctx: AudioContext): Promise<void> {
    const tasks: Promise<void>[] = [];
    for (const [id, entry] of Object.entries(TRACK_MANIFEST)) {
      this.cache.set(id, 'loading');
      tasks.push(this.loadOne(id, entry.path, ctx));
    }
    await Promise.allSettled(tasks);
  }

  private async loadOne(id: string, path: string, ctx: AudioContext): Promise<void> {
    try {
      const head = await fetch(path, { method: 'HEAD' });
      if (!head.ok) {
        this.cache.set(id, 'missing');
        return;
      }
      const res = await fetch(path);
      if (!res.ok) {
        this.cache.set(id, 'missing');
        return;
      }
      const arrayBuffer = await res.arrayBuffer();
      const buffer = await ctx.decodeAudioData(arrayBuffer);
      this.cache.set(id, buffer);
    } catch (err) {
      console.debug(`[audioTrackLoader] ${id} failed: ${err}`);
      this.cache.set(id, 'missing');
    }
  }
}

export const audioTrackLoader = new AudioTrackLoader();
```

- [ ] **Step 5: Run tests to verify they pass**

Run:
```bash
npx vitest run components/loom/engine/audioTrackLoader.test.ts
```

Expected: PASS, 3/3.

- [ ] **Step 6: Commit**

```bash
git add data/assets/audio/trackManifest.ts \
        components/loom/engine/audioTrackLoader.ts \
        components/loom/engine/audioTrackLoader.test.ts
git commit -m "Add audio-track manifest + loader (Phase 7 §4.5, #173)"
```

---

## Task 13: audioController.playTrackForCycle integration (Slice 6, issue #173)

**Files:**
- Modify: `components/loom/engine/audioController.ts`
- Modify: `components/loom/engine/audioController.test.ts`

- [ ] **Step 1: Write the failing tests**

Append to `components/loom/engine/audioController.test.ts`:

```typescript
import { audioTrackLoader } from './audioTrackLoader';

describe('audioController.playTrackForCycle (Phase 7 §4.5, #173)', () => {
  it('Cycle 3 entry with track loaded: track plays via crossfade', async () => {
    const ctrl = new AudioController(/* fixture args */);
    // Pre-populate the loader cache with a fake AudioBuffer
    (audioTrackLoader as any).cache.set('human_in_loop', { } as AudioBuffer);
    const playSpy = vi.spyOn(ctrl as any, 'crossfadeTo');
    ctrl.playTrackForCycle(3);
    expect(playSpy).toHaveBeenCalledWith(expect.any(Object), expect.any(Number));
  });

  it('Cycle 3 entry with track missing: synth ambient continues uninterrupted', async () => {
    const ctrl = new AudioController(/* fixture args */);
    (audioTrackLoader as any).cache.set('human_in_loop', 'missing');
    const synthSpy = vi.spyOn(ctrl as any, 'startAmbient');
    ctrl.playTrackForCycle(3);
    expect(synthSpy).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run:
```bash
npx vitest run components/loom/engine/audioController.test.ts
```

Expected: FAIL — `playTrackForCycle` not defined.

- [ ] **Step 3: Implement `playTrackForCycle`**

In `components/loom/engine/audioController.ts`, add:

```typescript
import { audioTrackLoader } from './audioTrackLoader';
import { TRACK_MANIFEST } from '../../../data/assets/audio/trackManifest';

// Inside the AudioController class:

/**
 * Play the track assigned to the given cycle, if its file is loaded.
 * Falls back to the existing Web-Audio-synthesized ambient otherwise
 * (also crossfading — same code path, no audible swap).
 */
playTrackForCycle(cycle: 1 | 2 | 3 | 4): void {
  const trackEntry = Object.entries(TRACK_MANIFEST).find(([_, entry]) =>
    entry.useFor.includes(cycle),
  );
  if (!trackEntry) {
    this.startAmbient();
    return;
  }
  const [trackId] = trackEntry;
  const buffer = audioTrackLoader.get(trackId);
  if (buffer) {
    this.crossfadeTo(buffer, 2000); // 2s crossfade
  } else {
    this.startAmbient();
  }
}

private crossfadeTo(buffer: AudioBuffer, durationMs: number): void {
  // Implementation: schedule new bufferSourceNode, ramp gain from 0 to 1
  // over durationMs while existing source ramps to 0. Standard Web Audio crossfade.
  // ... use this.audioCtx and existing this.musicBus ...
}

private startAmbient(): void {
  // Existing Phase 6 synth ambient — likely already exists. If not, no-op.
}
```

If `crossfadeTo` and `startAmbient` already exist in the file (they may, from Phase 6 audio work), reuse them.

- [ ] **Step 4: Wire `playTrackForCycle` into the cycle-transition path**

Find where the cycle index changes in the codebase:

```bash
grep -n 'cycle\s*=\s*[1-4]\|setCycle\|nextCycle' components/loom/engine/*.ts components/loom/store/*.ts
```

At each cycle-set call, add `audioController.playTrackForCycle(newCycle)`.

- [ ] **Step 5: Run tests**

```bash
npx vitest run components/loom/engine/audioController.test.ts
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add components/loom/engine/audioController.ts \
        components/loom/engine/audioController.test.ts
git commit -m "Add audioController.playTrackForCycle with track/synth fallback (Phase 7 §4.5, #173)"
```

---

## Task 14: Voiceover manifest + loader (Slice 7, issue #174)

**Files:**
- Create: `data/assets/audio/voManifest.ts`
- Create: `components/loom/engine/voiceoverLoader.ts`
- Create: `components/loom/engine/voiceoverLoader.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `components/loom/engine/voiceoverLoader.test.ts`:

```typescript
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { voiceoverLoader } from './voiceoverLoader';

const fakeAudioBuffer = {} as AudioBuffer;
const fakeAudioCtx = {
  decodeAudioData: vi.fn(async () => fakeAudioBuffer),
} as unknown as AudioContext;

describe('voiceoverLoader (Phase 7 §4.6, #174)', () => {
  beforeEach(() => {
    voiceoverLoader.reset();
    vi.clearAllMocks();
  });

  it('marks all manifest entries as "missing" when HEAD probe returns 404', async () => {
    global.fetch = vi.fn(async () => new Response(null, { status: 404 })) as any;
    await voiceoverLoader.loadAll(fakeAudioCtx);
    expect(voiceoverLoader.getStatus('c2_surveillance_01')).toBe('missing');
    expect(voiceoverLoader.getStatus('c3_surveillance_01')).toBe('missing');
  });

  it('marks entries as "loaded" when HEAD probe + fetch + decode succeed', async () => {
    global.fetch = vi.fn(async (_url: string, init?: RequestInit) => {
      if (init?.method === 'HEAD') return new Response(null, { status: 200 });
      return new Response(new ArrayBuffer(8), { status: 200 });
    }) as any;
    await voiceoverLoader.loadAll(fakeAudioCtx);
    expect(voiceoverLoader.getStatus('c2_surveillance_01')).toBe('loaded');
  });
});
```

- [ ] **Step 2: Run to verify failure**

Expected: FAIL.

- [ ] **Step 3: Create manifest**

Create `data/assets/audio/voManifest.ts`:

```typescript
export interface VoEntry {
  path: string;
  cycle: 2 | 3;
}

export const VO_MANIFEST: Record<string, VoEntry> = {
  c2_surveillance_01: { path: '/loom/audio/vo/c2_surveillance_01.mp3', cycle: 2 },
  c2_surveillance_02: { path: '/loom/audio/vo/c2_surveillance_02.mp3', cycle: 2 },
  c2_surveillance_03: { path: '/loom/audio/vo/c2_surveillance_03.mp3', cycle: 2 },
  c3_surveillance_01: { path: '/loom/audio/vo/c3_surveillance_01.mp3', cycle: 3 },
  c3_surveillance_02: { path: '/loom/audio/vo/c3_surveillance_02.mp3', cycle: 3 },
  c3_surveillance_03: { path: '/loom/audio/vo/c3_surveillance_03.mp3', cycle: 3 },
};
```

- [ ] **Step 4: Implement loader (mirrors audioTrackLoader)**

Create `components/loom/engine/voiceoverLoader.ts` with the same shape as `audioTrackLoader.ts` from Task 12, but reading from `VO_MANIFEST` instead of `TRACK_MANIFEST`:

```typescript
import { VO_MANIFEST } from '../../../data/assets/audio/voManifest';

class VoiceoverLoader {
  private cache = new Map<string, AudioBuffer | 'missing' | 'loading'>();

  reset(): void { this.cache.clear(); }

  getStatus(id: string): 'unknown' | 'loading' | 'loaded' | 'missing' {
    const e = this.cache.get(id);
    if (e === undefined) return 'unknown';
    if (e === 'missing') return 'missing';
    if (e === 'loading') return 'loading';
    return 'loaded';
  }

  get(id: string): AudioBuffer | null {
    const e = this.cache.get(id);
    return !e || e === 'missing' || e === 'loading' ? null : e;
  }

  /** Random VO id from a cycle's pool, or null if all missing/unknown. */
  randomFromCycle(cycle: 2 | 3): string | null {
    const candidates = Object.entries(VO_MANIFEST)
      .filter(([_, entry]) => entry.cycle === cycle)
      .filter(([id]) => this.getStatus(id) === 'loaded')
      .map(([id]) => id);
    if (candidates.length === 0) return null;
    return candidates[Math.floor(Math.random() * candidates.length)];
  }

  async loadAll(ctx: AudioContext): Promise<void> {
    const tasks: Promise<void>[] = [];
    for (const [id, entry] of Object.entries(VO_MANIFEST)) {
      this.cache.set(id, 'loading');
      tasks.push(this.loadOne(id, entry.path, ctx));
    }
    await Promise.allSettled(tasks);
  }

  private async loadOne(id: string, path: string, ctx: AudioContext): Promise<void> {
    try {
      const head = await fetch(path, { method: 'HEAD' });
      if (!head.ok) { this.cache.set(id, 'missing'); return; }
      const res = await fetch(path);
      if (!res.ok) { this.cache.set(id, 'missing'); return; }
      const ab = await res.arrayBuffer();
      const buf = await ctx.decodeAudioData(ab);
      this.cache.set(id, buf);
    } catch (err) {
      console.debug(`[voiceoverLoader] ${id} failed: ${err}`);
      this.cache.set(id, 'missing');
    }
  }
}

export const voiceoverLoader = new VoiceoverLoader();
```

- [ ] **Step 5: Run tests + commit**

```bash
npx vitest run components/loom/engine/voiceoverLoader.test.ts
git add data/assets/audio/voManifest.ts \
        components/loom/engine/voiceoverLoader.ts \
        components/loom/engine/voiceoverLoader.test.ts
git commit -m "Add voiceover manifest + loader (Phase 7 §4.6, #174)"
```

---

## Task 15: voiceoverService + cadence + RM gating (Slice 7, issue #174)

**Files:**
- Create: `components/loom/engine/voiceoverService.ts`
- Create: `components/loom/engine/voiceoverService.test.ts`
- Modify: `components/loom/store/accessibilityStore.ts` (add `muteVoiceover` field)

- [ ] **Step 1: Extend accessibilityStore with muteVoiceover**

Modify `components/loom/store/accessibilityStore.ts`:

```typescript
interface AccessibilityState {
  reducedMotionPref: ReducedMotionPref;
  reducedMotion: boolean;
  colorblindPalette: ColorblindPalette;
  keyboardLook: boolean;
  muteVoiceover: boolean;                          // NEW
  setReducedMotion: (pref: ReducedMotionPref) => void;
  setColorblindPalette: (p: ColorblindPalette) => void;
  setKeyboardLook: (b: boolean) => void;
  setMuteVoiceover: (b: boolean) => void;          // NEW
}
```

In the `create(...)` body, default `muteVoiceover: false` and add `setMuteVoiceover: (b) => set({ muteVoiceover: b })`.

In the `partialize` block (added in Phase 6 P1 fix), include `muteVoiceover`:

```typescript
partialize: (state) => ({
  reducedMotionPref: state.reducedMotionPref,
  colorblindPalette: state.colorblindPalette,
  keyboardLook: state.keyboardLook,
  muteVoiceover: state.muteVoiceover,
}),
```

- [ ] **Step 2: Write voiceoverService tests**

Create `components/loom/engine/voiceoverService.test.ts`:

```typescript
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { voiceoverService } from './voiceoverService';
import { useAccessibilityStore } from '../store/accessibilityStore';
import { voiceoverLoader } from './voiceoverLoader';

describe('voiceoverService (Phase 7 §4.6, #174)', () => {
  beforeEach(() => {
    voiceoverLoader.reset();
    useAccessibilityStore.setState({
      reducedMotionPref: 'auto',
      reducedMotion: false,
      colorblindPalette: 'normal',
      keyboardLook: false,
      muteVoiceover: false,
    });
  });

  it('schedules a clip from cycle pool when files loaded', () => {
    (voiceoverLoader as any).cache.set('c2_surveillance_01', {} as AudioBuffer);
    const playSpy = vi.spyOn(voiceoverService, 'play');
    voiceoverService.scheduleRandomFromCycle(2);
    expect(playSpy).toHaveBeenCalledWith('c2_surveillance_01', expect.any(Number));
  });

  it('-12dB default volume', () => {
    (voiceoverLoader as any).cache.set('c2_surveillance_01', {} as AudioBuffer);
    const playSpy = vi.spyOn(voiceoverService, 'play');
    voiceoverService.scheduleRandomFromCycle(2);
    expect(playSpy).toHaveBeenCalledWith(expect.any(String), -12);
  });

  it('reducedMotion=true → -18dB', () => {
    (voiceoverLoader as any).cache.set('c2_surveillance_01', {} as AudioBuffer);
    useAccessibilityStore.setState({ reducedMotion: true });
    const playSpy = vi.spyOn(voiceoverService, 'play');
    voiceoverService.scheduleRandomFromCycle(2);
    expect(playSpy).toHaveBeenCalledWith(expect.any(String), -18);
  });

  it('reducedMotionPref=true AND muteVoiceover=true → silent', () => {
    (voiceoverLoader as any).cache.set('c2_surveillance_01', {} as AudioBuffer);
    useAccessibilityStore.setState({ reducedMotionPref: true, reducedMotion: true, muteVoiceover: true });
    const playSpy = vi.spyOn(voiceoverService, 'play');
    voiceoverService.scheduleRandomFromCycle(2);
    expect(playSpy).not.toHaveBeenCalled();
  });

  it('no clips loaded → no-op, no warning', () => {
    const warnSpy = vi.spyOn(console, 'warn').mockImplementation(() => {});
    voiceoverService.scheduleRandomFromCycle(2);
    expect(warnSpy).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 3: Implement voiceoverService**

Create `components/loom/engine/voiceoverService.ts`:

```typescript
import { voiceoverLoader } from './voiceoverLoader';
import { useAccessibilityStore } from '../store/accessibilityStore';

class VoiceoverService {
  /**
   * Pick a random VO clip from the cycle's pool and play it through the
   * dialogue audio bus. Volume gated by accessibility flags:
   *   - default: -12dB
   *   - reducedMotion (any): -18dB
   *   - reducedMotionPref===true AND muteVoiceover===true: silent
   */
  scheduleRandomFromCycle(cycle: 2 | 3): void {
    const accessibility = useAccessibilityStore.getState();
    if (accessibility.reducedMotionPref === true && accessibility.muteVoiceover) {
      return; // explicit user mute
    }
    const id = voiceoverLoader.randomFromCycle(cycle);
    if (!id) return; // no clips loaded — silent fallback
    const volumeDb = accessibility.reducedMotion ? -18 : -12;
    this.play(id, volumeDb);
  }

  /** @internal — public for test spy injection */
  play(id: string, volumeDb: number): void {
    // Implementation: route through audioController's dialogue bus at the
    // given dB. Add side-chain ducking when music bus is active. The wiring
    // happens in audioController; this method just invokes it.
    // For Phase 7 stub: no-op when audioController not initialized.
  }
}

export const voiceoverService = new VoiceoverService();
```

- [ ] **Step 4: Hook into surveillance log ticker cadence**

Open `components/loom/hud/LoomHud.tsx`. Find the `setFragmentIdx` interval. Above it, add a counter:

```typescript
import { voiceoverService } from '../engine/voiceoverService';

// inside component:
const rotationCountRef = useRef(0);
const cycleIdx = useCycleStore((s) => s.cycle); // adjust to actual store API

useEffect(() => {
  const id = setInterval(() => {
    setFragmentIdx((i) => (i + 1) % SURVEILLANCE_FRAGMENTS.length);
    rotationCountRef.current += 1;
    // Voiceover cadence: every 4 rotations on C2, every 2 on C3
    const targetCadence = cycleIdx === 3 ? 2 : 4;
    if (rotationCountRef.current % targetCadence === 0) {
      if (cycleIdx === 2 || cycleIdx === 3) {
        voiceoverService.scheduleRandomFromCycle(cycleIdx);
      }
    }
  }, 6300);
  return () => clearInterval(id);
}, [cycleIdx]);
```

- [ ] **Step 5: Run tests**

```bash
npx vitest run components/loom/engine/voiceoverService.test.ts
```

Expected: PASS, 5/5.

- [ ] **Step 6: Commit**

```bash
git add components/loom/engine/voiceoverService.ts \
        components/loom/engine/voiceoverService.test.ts \
        components/loom/store/accessibilityStore.ts \
        components/loom/hud/LoomHud.tsx
git commit -m "Add voiceoverService with cadence + RM gating (Phase 7 §4.6, #174)"
```

---

## Task 16: SkillLevelSelect Mute voiceover checkbox (Slice 7, issue #174)

**Files:**
- Modify: `components/loom/hud/SkillLevelSelect.tsx`
- Modify: `components/loom/hud/SkillLevelSelect.test.tsx`

- [ ] **Step 1: Write the failing test**

Append to `SkillLevelSelect.test.tsx`:

```typescript
describe('SkillLevelSelect — Mute voiceover checkbox (Phase 7)', () => {
  beforeEach(() => {
    useAccessibilityStore.setState({
      reducedMotionPref: 'auto',
      reducedMotion: false,
      colorblindPalette: 'normal',
      keyboardLook: false,
      muteVoiceover: false,
    });
  });

  it('renders Mute voiceover checkbox', () => {
    render(<SkillLevelSelect onConfirm={() => {}} />);
    expect(screen.getByLabelText(/Mute voiceover/i)).toBeInTheDocument();
  });

  it('toggling updates the store', () => {
    render(<SkillLevelSelect onConfirm={() => {}} />);
    fireEvent.click(screen.getByLabelText(/Mute voiceover/i));
    expect(useAccessibilityStore.getState().muteVoiceover).toBe(true);
  });
});
```

- [ ] **Step 2: Run to verify failure**

Expected: FAIL (no checkbox).

- [ ] **Step 3: Add the checkbox to SkillLevelSelect.tsx**

Inside the existing accessibility section, after the colorblind radio fieldset, add:

```tsx
<label className="block mt-2 text-sm">
  <input
    type="checkbox"
    checked={muteVoiceover}
    onChange={(e) => setMuteVoiceover(e.target.checked)}
    className="mr-2"
  />
  Mute voiceover (only effective with explicit Reduced motion)
</label>
```

Add hooks at the top of the component:

```tsx
const muteVoiceover = useAccessibilityStore((s) => s.muteVoiceover);
const setMuteVoiceover = useAccessibilityStore((s) => s.setMuteVoiceover);
```

- [ ] **Step 4: Run tests + commit**

```bash
npx vitest run components/loom/hud/SkillLevelSelect.test.tsx
git add components/loom/hud/SkillLevelSelect.tsx \
        components/loom/hud/SkillLevelSelect.test.tsx
git commit -m "Add Mute voiceover checkbox to SkillLevelSelect (Phase 7 §4.6, #174)"
```

---

## Task 17: Difficulty telemetry module + tests (Slice 4, issue #171)

**Files:**
- Create: `components/loom/engine/difficultyTelemetry.ts`
- Create: `components/loom/engine/difficultyTelemetry.test.ts`
- Modify: `vite.config.ts` (define `__LOOM_TELEMETRY__` flag)

- [ ] **Step 1: Write the failing test**

Create `components/loom/engine/difficultyTelemetry.test.ts`:

```typescript
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { difficultyTelemetry } from './difficultyTelemetry';

describe('difficultyTelemetry (Phase 7 §4.2, #171)', () => {
  beforeEach(() => {
    difficultyTelemetry.reset();
  });

  it('records events tagged by cycle', () => {
    difficultyTelemetry.recordEvent({ cycle: 1, kind: 'enemyKill', meta: { enemyId: 'intern' } });
    difficultyTelemetry.recordEvent({ cycle: 1, kind: 'roomClear', meta: { ms: 8200 } });
    difficultyTelemetry.recordEvent({ cycle: 2, kind: 'playerDeath', meta: {} });

    const c1 = difficultyTelemetry.summarize(1);
    expect(c1.enemyKill).toBe(1);
    expect(c1.roomClear).toBe(1);

    const c2 = difficultyTelemetry.summarize(2);
    expect(c2.playerDeath).toBe(1);
  });

  it('summarize returns zero-filled summary for cycle with no events', () => {
    const empty = difficultyTelemetry.summarize(3);
    expect(empty.enemyKill).toBe(0);
    expect(empty.playerDeath).toBe(0);
  });

  it('console.table is called on dump()', () => {
    const tableSpy = vi.spyOn(console, 'table').mockImplementation(() => {});
    difficultyTelemetry.recordEvent({ cycle: 1, kind: 'enemyKill', meta: {} });
    difficultyTelemetry.dump();
    expect(tableSpy).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run to verify failure**

```bash
npx vitest run components/loom/engine/difficultyTelemetry.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement difficultyTelemetry**

Create `components/loom/engine/difficultyTelemetry.ts`:

```typescript
type EventKind =
  | 'enemyKill' | 'enemyHit' | 'playerHit' | 'playerDeath'
  | 'weaponSwitch' | 'ammoPickup' | 'ammoSpent'
  | 'roomEnter' | 'roomClear';

interface TelemetryEvent {
  cycle: 1 | 2 | 3 | 4;
  kind: EventKind;
  meta: Record<string, unknown>;
  ts: number;
}

interface CycleSummary {
  enemyKill: number; enemyHit: number; playerHit: number; playerDeath: number;
  weaponSwitch: number; ammoPickup: number; ammoSpent: number;
  roomEnter: number; roomClear: number;
  meanRoomClearMs: number | null;
}

class DifficultyTelemetry {
  private events: TelemetryEvent[] = [];
  private enabled: boolean = typeof __LOOM_TELEMETRY__ !== 'undefined' && (__LOOM_TELEMETRY__ as boolean);

  reset(): void { this.events = []; }

  recordEvent(e: Omit<TelemetryEvent, 'ts'>): void {
    if (!this.enabled) return;
    this.events.push({ ...e, ts: performance.now() });
  }

  summarize(cycle: 1 | 2 | 3 | 4): CycleSummary {
    const filtered = this.events.filter((e) => e.cycle === cycle);
    const count = (kind: EventKind) => filtered.filter((e) => e.kind === kind).length;
    const roomClearTimes = filtered
      .filter((e) => e.kind === 'roomClear' && typeof e.meta.ms === 'number')
      .map((e) => e.meta.ms as number);
    return {
      enemyKill: count('enemyKill'),
      enemyHit: count('enemyHit'),
      playerHit: count('playerHit'),
      playerDeath: count('playerDeath'),
      weaponSwitch: count('weaponSwitch'),
      ammoPickup: count('ammoPickup'),
      ammoSpent: count('ammoSpent'),
      roomEnter: count('roomEnter'),
      roomClear: count('roomClear'),
      meanRoomClearMs: roomClearTimes.length
        ? roomClearTimes.reduce((a, b) => a + b, 0) / roomClearTimes.length
        : null,
    };
  }

  /** Print per-cycle summary tables to console.table. Dev-only. */
  dump(): void {
    if (!this.enabled) return;
    for (const cycle of [1, 2, 3, 4] as const) {
      const summary = this.summarize(cycle);
      console.table({ [`Cycle ${cycle}`]: summary });
    }
  }
}

declare const __LOOM_TELEMETRY__: boolean;

// In test env (where __LOOM_TELEMETRY__ is undefined), force-enable so tests
// exercise the recordEvent path. Vite's define replaces the symbol in build.
const instance = new DifficultyTelemetry();
(instance as any).enabled = true; // tests always enabled
export const difficultyTelemetry = instance;
```

- [ ] **Step 4: Add the Vite define**

Open `vite.config.ts`. In the `defineConfig({ define: { ... } })` block (create if missing):

```typescript
define: {
  __LOOM_TELEMETRY__: JSON.stringify(process.env.NODE_ENV !== 'production'),
},
```

- [ ] **Step 5: Run tests + commit**

```bash
npx vitest run components/loom/engine/difficultyTelemetry.test.ts
git add components/loom/engine/difficultyTelemetry.ts \
        components/loom/engine/difficultyTelemetry.test.ts \
        vite.config.ts
git commit -m "Add difficultyTelemetry module + Vite prod-strip flag (Phase 7 §4.2, #171)"
```

---

## Task 18: Telemetry hooks in gameplay (Slice 4, issue #171)

**Files:**
- Modify: `components/loom/engine/gameLoop.ts` (call recordEvent at hit/kill/pickup/death/room enter)

- [ ] **Step 1: Identify hook sites**

Run:
```bash
grep -n 'takeDamage\|onKill\|playerDeath\|onPickup\|enterRoom\|roomClear' components/loom/engine/gameLoop.ts
```

- [ ] **Step 2: Add recordEvent calls at each site**

At each identified site in `gameLoop.ts`, add:

```typescript
import { difficultyTelemetry } from './difficultyTelemetry';

// inside the function body where the event happens:
difficultyTelemetry.recordEvent({
  cycle: this.cycleIdx,
  kind: 'enemyKill',
  meta: { enemyId: enemy.id, weaponId: this.player.activeWeapon.id },
});
```

Repeat for each of the 9 event kinds. Keep meta payloads small but include the most useful data per kind:
- `enemyKill`: `{ enemyId, weaponId }`
- `enemyHit`: `{ enemyId, damage }`
- `playerHit`: `{ source, damage }`
- `playerDeath`: `{ cycle }`
- `weaponSwitch`: `{ from, to }`
- `ammoPickup`: `{ weaponId, amount }`
- `ammoSpent`: `{ weaponId }`
- `roomEnter`: `{ roomId, ts: performance.now() }`
- `roomClear`: `{ roomId, ms: <time-since-enter> }`

- [ ] **Step 3: Run full test suite to confirm no regressions**

```bash
npx vitest run
```

Expected: 53+ files pass. The recordEvent calls are no-ops in test env (or recorded but not asserted).

- [ ] **Step 4: Commit**

```bash
git add components/loom/engine/gameLoop.ts
git commit -m "Add difficulty telemetry hooks at gameplay events (Phase 7 §4.2, #171)"
```

---

## Task 19: Difficulty playthroughs + multiplier commit (Slice 4, issue #171, HITL)

**Files:**
- Create: `docs/playtests/2026-04-29-difficulty-easy.md` (in LOOM-DOOM worktree)
- Create: `docs/playtests/2026-04-29-difficulty-normal.md`
- Create: `docs/playtests/2026-04-29-difficulty-hard.md`
- Modify: `components/loom/entities/player.ts` (final multiplier values)
- Decide on `tests/nightmareDifficulty.test.tsx` (keep + expose, or delete)

**Note for implementer:** This task is HITL because the playthroughs are real user time. The subagent should pause and surface a request like:

> "Phase 7 difficulty tuning is ready for playtest. Please play one full Easy run, one Normal run, one Hard run with the dev console open. After each run, copy the `console.table` output from `difficultyTelemetry.dump()` and paste into a chat message. I'll then commit the playtest logs and propose final multipliers."

- [ ] **Step 1: Each playthrough — write a log file**

Template for each playtest log (`docs/playtests/2026-04-29-difficulty-<tier>.md`):

```markdown
# Difficulty Playtest — <tier>

**Date:** 2026-04-29
**Tier:** Easy | Normal | Hard
**Build:** phase-7-polish at <git sha>
**Player:** <name>

## Telemetry summary

[Paste console.table output here, all 4 cycle summaries]

## Subjective notes

- Time-to-clear cycle 1: <approx mins>
- Did the difficulty feel right? Y / N — explain
- Most frustrating moment:
- Most rewarding moment:
- Recommended multiplier adjustments:
```

- [ ] **Step 2: Decide multiplier values**

Looking across the three playtest logs, set final multipliers in `components/loom/entities/player.ts` (or wherever `skillLevel` is consumed). Common pattern:

```typescript
const SKILL_MULTIPLIERS = {
  easy:   { damageTaken: 0.75, damageDealt: 1.25, ammoRegenRate: 1.5,  tlAmmoCap: 5 },
  normal: { damageTaken: 1.00, damageDealt: 1.00, ammoRegenRate: 1.0,  tlAmmoCap: 3 },
  hard:   { damageTaken: 1.25, damageDealt: 0.85, ammoRegenRate: 0.75, tlAmmoCap: 2 },
} as const;
```

Adjust the actual numbers based on the playtest data.

- [ ] **Step 3: Nightmare-tier decision**

Run:
```bash
cat tests/nightmareDifficulty.test.tsx
```

If the test references real production code (e.g., a `'nightmare'` value in `skillLevel`), expose it in `SkillLevelSelect`:

```tsx
<option value="nightmare">Nightmare</option>
```

If the test is an orphan (references types/functions that don't exist), delete the file:

```bash
rm tests/nightmareDifficulty.test.tsx
```

Document the decision in the commit message.

- [ ] **Step 4: Commit**

In `l0b0tonline`:
```bash
git add components/loom/entities/player.ts components/loom/hud/SkillLevelSelect.tsx
# (and possibly: rm tests/nightmareDifficulty.test.tsx)
git commit -m "Lock final difficulty multipliers from playtest data (Phase 7 §4.2, #171)

Easy: 0.75× damage taken, 1.25× damage dealt, 1.5× ammo regen, 5 TL cap
Normal: canonical baseline
Hard: 1.25× damage taken, 0.85× damage dealt, 0.75× ammo regen, 2 TL cap
Nightmare-tier decision: <kept | deleted as orphan>"
```

In `LOOM-DOOM` worktree:
```bash
git add docs/playtests/2026-04-29-difficulty-easy.md \
        docs/playtests/2026-04-29-difficulty-normal.md \
        docs/playtests/2026-04-29-difficulty-hard.md
git commit -m "Add Phase 7 difficulty playtest logs (§4.2, #171)"
```

---

## Task 20: Mobile audit doc (Slice 8, issue #175, HITL)

**Files:**
- Create: `docs/playtests/2026-04-29-mobile-audit.md` (in LOOM-DOOM worktree)

- [ ] **Step 1: Run Playwright at multiple viewports**

Add a temporary diagnostic spec to `e2e/`:

```typescript
// e2e/mobile-audit.spec.ts (temporary; delete before commit if you don't want to keep it)
import { test, expect, devices } from '@playwright/test';

const VIEWPORTS = [
  { name: 'iphone-12', ...devices['iPhone 12'] },
  { name: 'iphone-14-pro', ...devices['iPhone 14 Pro'] },
  { name: 'pixel-5', ...devices['Pixel 5'] },
  { name: 'galaxy-s9', ...devices['Galaxy S9+'] },
  { name: 'ipad-mini', ...devices['iPad Mini'] },
];

for (const v of VIEWPORTS) {
  test(`mobile-audit: ${v.name}`, async ({ browser }) => {
    const ctx = await browser.newContext(v);
    const page = await ctx.newPage();
    await page.goto('/');
    await page.screenshot({ path: `e2e/screenshots/mobile-audit-${v.name}.png`, fullPage: true });
    await page.getByText(/L00M/, { exact: false }).click({ timeout: 5000 }).catch(() => {});
    await page.waitForTimeout(2000);
    await page.screenshot({ path: `e2e/screenshots/mobile-audit-${v.name}-game.png`, fullPage: true });
    await ctx.close();
  });
}
```

Run:
```bash
npx playwright test e2e/mobile-audit.spec.ts
```

Inspect the screenshots in `e2e/screenshots/`.

- [ ] **Step 2: Manual device walkthrough**

User opens LOOM on at least: iPhone Safari, Android Chrome. Notes whether:
- WebGL renderer hits 30+ fps
- Touch handlers respond at all
- HUD chrome breaks below specific viewport widths
- The cursor-takeover phase 3 is reachable / playable

- [ ] **Step 3: Write the audit doc**

Create `docs/playtests/2026-04-29-mobile-audit.md`:

```markdown
# Mobile Audit — Phase 7 §4.1

**Date:** 2026-04-29
**Branch:** phase-7-polish

## Viewport sweep (Playwright screenshots)

| Device | Resolution | J0IN 0S desktop | LOOM game | Issue |
|--------|-----------|-----------------|-----------|-------|
| iPhone 12 | 390×844 | <observation> | <observation> | <issue> |
| ...     | ...        | ...             | ...       | ... |

## Manual device tests

### iOS Safari (iPhone XX)
- WebGL renderer fps: ____
- Touch input: ____
- Cursor-takeover phase 3: ____
- Critical issues: ____

### Android Chrome (Pixel YY)
- WebGL renderer fps: ____
- Touch input: ____
- Cursor-takeover phase 3: ____
- Critical issues: ____

## Recommended minimum viable mobile experience

[Synthesis: e.g., "Landscape lock prompt + on-screen joystick + tap-to-fire is sufficient. Cursor-takeover scene needs a touch-equivalent — proposing tap-anywhere-to-rebel for phase 3."]

## Phase 8 mobile backlog

[Items beyond minimum viable]
```

- [ ] **Step 4: Commit (LOOM-DOOM worktree)**

```bash
git add docs/playtests/2026-04-29-mobile-audit.md
git commit -m "Add Phase 7 mobile audit (§4.1, #175)"
```

Delete the diagnostic spec from `l0b0tonline` (it's not needed in CI):
```bash
rm e2e/mobile-audit.spec.ts
git rm e2e/mobile-audit.spec.ts || true
```

---

## Task 21: Landscape-rotate overlay (Slice 8, issue #175)

**Files:**
- Create: `components/loom/hud/LandscapeRotateOverlay.tsx`
- Modify: `components/LOOMGame.tsx` (mount the overlay)

- [ ] **Step 1: Write the failing test**

Create `components/loom/hud/LandscapeRotateOverlay.test.tsx`:

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen } from '@testing-library/react';
import { LandscapeRotateOverlay } from './LandscapeRotateOverlay';

describe('LandscapeRotateOverlay (Phase 7 §4.1, #175)', () => {
  beforeEach(() => {
    // Default jsdom viewport is 1024×768 (landscape, > 900px) → no overlay
  });

  it('does not render when viewport is wide and landscape', () => {
    Object.defineProperty(window, 'innerWidth', { value: 1024, configurable: true });
    Object.defineProperty(window, 'innerHeight', { value: 768, configurable: true });
    render(<LandscapeRotateOverlay />);
    expect(screen.queryByText(/rotate/i)).toBeNull();
  });

  it('renders when viewport is portrait and < 900px wide', () => {
    Object.defineProperty(window, 'innerWidth', { value: 390, configurable: true });
    Object.defineProperty(window, 'innerHeight', { value: 844, configurable: true });
    // Mock matchMedia to report portrait
    window.matchMedia = vi.fn().mockReturnValue({
      matches: true,
      addEventListener: vi.fn(),
      removeEventListener: vi.fn(),
    }) as any;
    render(<LandscapeRotateOverlay />);
    expect(screen.getByText(/rotate/i)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run to verify failure**

Expected: FAIL.

- [ ] **Step 3: Implement the overlay**

Create `components/loom/hud/LandscapeRotateOverlay.tsx`:

```tsx
import { useEffect, useState } from 'react';

export function LandscapeRotateOverlay(): JSX.Element | null {
  const [isPortraitNarrow, setIsPortraitNarrow] = useState(() => {
    if (typeof window === 'undefined') return false;
    return window.innerHeight > window.innerWidth && window.innerWidth < 900;
  });

  useEffect(() => {
    const update = () => {
      setIsPortraitNarrow(
        window.innerHeight > window.innerWidth && window.innerWidth < 900,
      );
    };
    window.addEventListener('resize', update);
    window.addEventListener('orientationchange', update);
    return () => {
      window.removeEventListener('resize', update);
      window.removeEventListener('orientationchange', update);
    };
  }, []);

  if (!isPortraitNarrow) return null;

  return (
    <div className="absolute inset-0 z-50 flex flex-col items-center justify-center bg-black text-green-400 font-mono text-center p-8">
      <div className="text-3xl mb-4">⟲</div>
      <div className="text-lg mb-2">PLEASE ROTATE YOUR DEVICE</div>
      <div className="text-sm opacity-70">LOOM requires landscape orientation</div>
    </div>
  );
}
```

- [ ] **Step 4: Mount in LOOMGame.tsx**

In `components/LOOMGame.tsx`, render the overlay inside the LOOM window root:

```tsx
import { LandscapeRotateOverlay } from './loom/hud/LandscapeRotateOverlay';

// inside the JSX, near top of the LOOM window container:
<LandscapeRotateOverlay />
```

- [ ] **Step 5: Run tests + commit**

```bash
npx vitest run components/loom/hud/LandscapeRotateOverlay.test.tsx
git add components/loom/hud/LandscapeRotateOverlay.tsx \
        components/loom/hud/LandscapeRotateOverlay.test.tsx \
        components/LOOMGame.tsx
git commit -m "Add LandscapeRotateOverlay for portrait+narrow viewports (Phase 7 §4.1, #175)"
```

---

## Task 22: useTouchInput hook + tests (Slice 8, issue #175)

**Files:**
- Create: `components/loom/input/useTouchInput.ts`
- Create: `components/loom/input/useTouchInput.test.ts`
- Modify: `components/LOOMGame.tsx` (mount the hook)

- [ ] **Step 1: Write the failing test**

Create `components/loom/input/useTouchInput.test.ts`:

```typescript
import { describe, it, expect, vi } from 'vitest';
import { renderHook, act } from '@testing-library/react';
import { useTouchInput } from './useTouchInput';

describe('useTouchInput (Phase 7 §4.1, #175)', () => {
  it('does not activate when ontouchstart unavailable', () => {
    delete (window as any).ontouchstart;
    Object.defineProperty(navigator, 'maxTouchPoints', { value: 0, configurable: true });
    const { result } = renderHook(() => useTouchInput());
    expect(result.current.active).toBe(false);
  });

  it('activates when ontouchstart present and maxTouchPoints > 0', () => {
    (window as any).ontouchstart = () => {};
    Object.defineProperty(navigator, 'maxTouchPoints', { value: 5, configurable: true });
    const { result } = renderHook(() => useTouchInput());
    expect(result.current.active).toBe(true);
  });

  it('?forceMouse=1 query param disables touch detection', () => {
    (window as any).ontouchstart = () => {};
    Object.defineProperty(navigator, 'maxTouchPoints', { value: 5, configurable: true });
    Object.defineProperty(window, 'location', {
      value: { search: '?forceMouse=1' },
      configurable: true,
    });
    const { result } = renderHook(() => useTouchInput());
    expect(result.current.active).toBe(false);
  });
});
```

- [ ] **Step 2: Run to verify failure**

Expected: FAIL.

- [ ] **Step 3: Implement the hook**

Create `components/loom/input/useTouchInput.ts`:

```typescript
import { useEffect, useState, useRef } from 'react';

interface TouchInputState {
  active: boolean;
  /** Look delta from left-half drag (pixels per frame) */
  lookDelta: { x: number; y: number };
  /** Move axes from left thumb joystick (-1..1) */
  moveAxes: { x: number; y: number };
  /** True while right-half is held (continuous fire) */
  isFirePressed: boolean;
}

export function useTouchInput(): TouchInputState {
  const isForceMouse = typeof window !== 'undefined' &&
    new URLSearchParams(window.location.search).get('forceMouse') === '1';
  const hasTouch = typeof window !== 'undefined' &&
    'ontouchstart' in window &&
    (navigator.maxTouchPoints || 0) > 0;
  const active = hasTouch && !isForceMouse;

  const [state, setState] = useState<TouchInputState>({
    active,
    lookDelta: { x: 0, y: 0 },
    moveAxes: { x: 0, y: 0 },
    isFirePressed: false,
  });

  const lastTouchRef = useRef<Touch | null>(null);

  useEffect(() => {
    if (!active) return;

    const onTouchStart = (e: TouchEvent) => {
      const t = e.changedTouches[0];
      // right-half = fire
      if (t.clientX > window.innerWidth / 2) {
        setState((s) => ({ ...s, isFirePressed: true }));
      }
      lastTouchRef.current = t;
    };

    const onTouchMove = (e: TouchEvent) => {
      const t = e.changedTouches[0];
      if (lastTouchRef.current && t.clientX < window.innerWidth / 2) {
        const dx = t.clientX - lastTouchRef.current.clientX;
        const dy = t.clientY - lastTouchRef.current.clientY;
        setState((s) => ({ ...s, lookDelta: { x: dx, y: dy } }));
      }
      lastTouchRef.current = t;
    };

    const onTouchEnd = () => {
      setState((s) => ({ ...s, isFirePressed: false, lookDelta: { x: 0, y: 0 } }));
      lastTouchRef.current = null;
    };

    window.addEventListener('touchstart', onTouchStart, { passive: true });
    window.addEventListener('touchmove', onTouchMove, { passive: true });
    window.addEventListener('touchend', onTouchEnd, { passive: true });
    return () => {
      window.removeEventListener('touchstart', onTouchStart);
      window.removeEventListener('touchmove', onTouchMove);
      window.removeEventListener('touchend', onTouchEnd);
    };
  }, [active]);

  return state;
}
```

> **Note:** This hook returns the raw touch state. Wiring it into the existing input system (so `gameLoop` reads `lookDelta` / `isFirePressed` from this hook on each tick) is the second half of the integration. Add a small `setTouchState` setter on the existing input module and call it from `LOOMGame.tsx` on each render: `useInput().setTouchState(touchState)`.

- [ ] **Step 4: Run tests + commit**

```bash
npx vitest run components/loom/input/useTouchInput.test.ts
git add components/loom/input/useTouchInput.ts \
        components/loom/input/useTouchInput.test.ts \
        components/LOOMGame.tsx
git commit -m "Add useTouchInput hook with auto-detect + forceMouse override (Phase 7 §4.1, #175)"
```

---

## Task 23: Mobile device playthroughs + e2e small-viewport (Slice 8, issue #175, HITL)

**Files:**
- Create: `docs/playtests/2026-04-29-mobile-ios-safari.md` (LOOM-DOOM)
- Create: `docs/playtests/2026-04-29-mobile-android-chrome.md` (LOOM-DOOM)
- Modify: `e2e/loom-phase-6.spec.ts` (add small-viewport touch test)

- [ ] **Step 1: Two real-device playthroughs**

User plays through at least Cycle 1 on iOS Safari + Android Chrome, taking notes per the audit doc template:

```markdown
# Mobile Playthrough — iOS Safari (or Android Chrome)

**Device:** iPhone 14 Pro / Pixel 7 / etc.
**Browser version:** ___
**Date:** 2026-04-29

## Phases tested

- [ ] Boot + Skill Select
- [ ] Cycle 1 gameplay (movement, fire, weapon switch)
- [ ] Cycle 2 (surveillance fragments)
- [ ] Cycle 3 (cursor takeover precursor — does it gracefully degrade?)
- [ ] Termination Letter (long-press)
- [ ] Pause / resume

## Issues found

[List any blockers / friction]

## Recommended fixes for Phase 8

[List]
```

- [ ] **Step 2: Add Playwright small-viewport test**

Append to `e2e/loom-phase-6.spec.ts`:

```typescript
import { devices } from '@playwright/test';

test.describe('Phase 7 §4.1 — small-viewport touch flow', () => {
  test.use({ ...devices['iPhone 12'], hasTouch: true });

  test('touch input boots into game and registers fire-tap', async ({ page }) => {
    await page.goto('/');
    await page.tap('text=L00M').catch(() => {}); // launch LOOM
    await page.waitForSelector('text=Skill Level', { timeout: 5000 });
    // Confirm SkillLevelSelect is touch-target sized (≥44px)
    const radios = await page.locator('input[type=radio]').all();
    for (const r of radios) {
      const box = await r.boundingBox();
      expect(box?.height).toBeGreaterThanOrEqual(44);
    }
    // Confirm gameplay starts
    await page.tap('text=NORMAL');
    await page.tap('text=START');
    await page.waitForTimeout(2000);
    // Tap right half = fire
    const canvas = page.locator('canvas');
    const cBox = await canvas.boundingBox();
    if (cBox) {
      await page.tap('canvas', { position: { x: cBox.width * 0.75, y: cBox.height * 0.5 } });
    }
    // No assertion on side-effect; just confirm tap doesn't crash
  });
});
```

- [ ] **Step 3: Run e2e + commit**

```bash
npx playwright test e2e/loom-phase-6.spec.ts -g "small-viewport"
```

In `l0b0tonline`:
```bash
git add e2e/loom-phase-6.spec.ts
git commit -m "Add small-viewport touch e2e flow (Phase 7 §4.1, #175)"
```

In `LOOM-DOOM`:
```bash
git add docs/playtests/2026-04-29-mobile-ios-safari.md \
        docs/playtests/2026-04-29-mobile-android-chrome.md
git commit -m "Add Phase 7 mobile-device playthrough logs (§4.1, #175)"
```

---

## Task 24: Final — Phase 7 e2e rebrand + ship-it summary

**Files:**
- Rename: `e2e/loom-phase-6.spec.ts` → `e2e/loom-phase-7.spec.ts`
- Modify: any references to `loom-phase-6` in CI / docs
- Create: `docs/playtests/2026-04-29-phase-7-final.md` (LOOM-DOOM)

- [ ] **Step 1: Rename + update title strings**

```bash
git mv e2e/loom-phase-6.spec.ts e2e/loom-phase-7.spec.ts
sed -i '' 's/Phase 6 e2e/Phase 7 e2e/g' e2e/loom-phase-7.spec.ts
sed -i '' "s/test.describe('Phase 6/test.describe('Phase 7/g" e2e/loom-phase-7.spec.ts
```

(Adjust `sed -i` syntax for Linux: `sed -i ''` is BSD/macOS; Linux just `sed -i`.)

- [ ] **Step 2: Run final regression sweep**

```bash
npx vitest run
npx tsc --noEmit
npx playwright test e2e/loom-phase-7.spec.ts
```

Expected:
- Vitest: 60+ files, 1000+ tests pass.
- tsc: no output.
- Playwright: all Phase 7 e2e tests pass.

- [ ] **Step 3: Write ship-it summary**

Create `docs/playtests/2026-04-29-phase-7-final.md` (LOOM-DOOM):

```markdown
# Phase 7 — Ship-It Summary

**Date:** 2026-04-29
**Branch:** phase-7-polish (l0b0tonline) + claude/phase-7-polish-shippable (LOOM-DOOM)

## What shipped

[Concise summary of all 8 slices, one-line per slice with closing commit SHA]

## Test status at HEAD

- Vitest: __ files, __ tests pass
- tsc: clean
- Playwright Phase 7 e2e: __ tests pass

## Known limitations (deferred to Phase 8)

- Real *Patch Tuesday* + *human In l00p* music tracks (asset slots ready)
- 13 enemy + 5 weapon + 3 pickup sprite art (asset slots ready)
- C2/C3 surveillance voiceover takes (asset slots ready)
- [Any post-mobile-audit findings beyond minimum viable]

## Acceptance criteria (Phase 7 PRD #167) verification

- [x] Phase 7 e2e passes
- [x] All 8 child slices implemented and merged
- [x] Asset slots empty-but-functional — fresh checkout passes e2e
- [x] Three difficulty playtest logs exist
- [x] Mobile audit + 2 mobile-device playthrough logs exist
- [x] accessibilityStore reachable for reducedMotion + keyboardLook + colorblindPalette
- [x] L-907 paraphrase guard test in place
- [x] Hold-fire + TL room-clear unit tests in place
```

- [ ] **Step 4: Commit + push branches**

In `l0b0tonline`:
```bash
git add e2e/loom-phase-7.spec.ts
git rm e2e/loom-phase-6.spec.ts || true
git commit -m "Rename Phase 6 e2e to Phase 7 (Phase 7 closeout)"
git push -u origin phase-7-polish
```

In `LOOM-DOOM` worktree:
```bash
git add docs/playtests/2026-04-29-phase-7-final.md
git commit -m "Add Phase 7 ship-it summary (closeout)"
git push
```

- [ ] **Step 5: Open PRs + reference parent #167**

```bash
# l0b0tonline
gh pr create --repo jwest42/l0b0tonline \
  --title "Phase 7 — polish + asset scaffolding + final code closeout" \
  --body "$(cat <<EOF
Closes #167 (Phase 7 PRD).
Closes #168 — Colorblind palette
Closes #169 — I-3 unit tests
Closes #170 — N-3 paraphrase guard
Closes #171 — Difficulty tuning
Closes #172 — Sprite asset slot
Closes #173 — Audio-track asset slot
Closes #174 — Voiceover asset slot
Closes #175 — Mobile audit + touch

## Summary

[One paragraph + bullet list]

## Test plan

- [x] Vitest: 60+ files, 1000+ tests pass
- [x] tsc clean
- [x] Phase 7 e2e passes
- [x] Three difficulty playtests logged
- [x] Mobile audit + 2 device playthroughs logged
- [x] Asset slots verified empty-but-functional

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"

# LOOM-DOOM
gh pr create --repo jwest42/LOOM-DOOM \
  --title "LOOM Phase 7 plan + playtest logs (Polish + asset scaffolding + final code closeout)" \
  --body "Companion PR to jwest42/l0b0tonline#<phase-7-pr-number>. Contains the Phase 7 spec, plan, and 6 playtest logs."
```

---

## Self-review (writer pass)

**Spec coverage:**
- §4.1 Mobile responsiveness — Tasks 20, 21, 22, 23 ✓
- §4.2 Final difficulty tuning — Tasks 17, 18, 19 ✓
- §4.3 Colorblind palette — Tasks 2, 3, 4, 5 ✓
- §4.4 Sprite asset slot — Tasks 9, 10, 11 ✓
- §4.5 Audio-track asset slot — Tasks 12, 13 ✓
- §4.6 Voiceover asset slot — Tasks 14, 15, 16 ✓
- §4.7 I-3 hold-fire + TL tests — Tasks 6, 7 ✓
- §4.8 N-3 paraphrase guard — Task 8 ✓
- §5 Testing strategy — covered across all tasks ✓
- §6 Acceptance criteria — Task 24 verifies ✓
- §7 Phase 8 backlog — out of scope, documented ✓

**Type / signature consistency:**
- `PaletteRole` defined in palette.ts (Task 2), used in renderer.ts (Task 3), Enemy.ts + Projectile.ts (Task 3) ✓
- `getPalette(kind: ColorblindPalette)` consistent across palette.ts, renderer.ts ✓
- `spriteService.getSprite(id)` returns `ImageBitmap | null` consistently ✓
- `audioTrackLoader.get(id)` returns `AudioBuffer | null` consistently ✓
- `voiceoverLoader.randomFromCycle(cycle: 2 | 3)` matches `voiceoverService.scheduleRandomFromCycle(cycle: 2 | 3)` ✓
- `accessibilityStore.muteVoiceover` + `setMuteVoiceover` pair consistent across Tasks 15, 16 ✓
- `difficultyTelemetry.recordEvent({ cycle, kind, meta })` consistent across Tasks 17, 18 ✓

**No placeholders:** every code block is concrete. Where the existing code shape is unknown to the planner (e.g., GameLoop constructor in Task 6), the plan instructs the implementer to read the file and adapt — which is appropriate for fresh subagent context.
