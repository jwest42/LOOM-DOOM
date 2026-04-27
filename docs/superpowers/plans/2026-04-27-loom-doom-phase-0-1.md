# LOOM Phase 0 + Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Integrate **LOOM** (a DOOM-style raycaster FPS) as a new app inside [l0b0tonline](../../../../../l0b0tonline)'s J0IN 0S simulator. Phase 0 stands up the J0IN 0S app shell (clickable icon → window spawns → black canvas). Phase 1 ships a tracer-bullet vertical: one map, two weapons, one enemy, J0IN 0S terminal HUD overlay, all running on a custom WebGL2 raycaster.

**Architecture:** Custom TypeScript raycaster rendering at internal 320×240 via WebGL2, scaled with nearest-neighbor to a J0IN 0S window. Game state in a Zustand slice. HUD as a React component overlaid on the canvas. Code lives in `l0b0tonline/components/loom/` and `l0b0tonline/components/LOOMGame.tsx`.

**Tech Stack:** TypeScript 5.8, React 19.2, Vite 6.2, Tailwind v4, Zustand 5, Web Audio API, WebGL 2 (new dep), Vitest, Playwright. **All except WebGL2 are already in l0b0tonline's package.json.**

**Pre-existing state (read-only context):**
- This LOOM-DOOM repo is a worktree on `claude/awesome-sutherland-aad854`
- Spec lives at `docs/superpowers/specs/2026-04-27-loom-doom-design.md` (this repo)
- Implementation work happens in **`/Users/justinwest/Repos/l0b0tonline/`** (sibling repo, **not** in this LOOM-DOOM worktree)
- l0b0tonline already has: app registry, window manager, Zustand store, Web Audio service, file system, unlocks system. We extend, we don't rebuild.

**Where commands run:** Each task notes its working directory. Most tasks operate in l0b0tonline. The plan and commit messages mention LOOM-DOOM (because the plan is here, by convention). Don't get them mixed.

---

## File structure (post-Phase-1, in l0b0tonline)

```
l0b0tonline/
├── types.ts                                            (modify — add GAME_LOOM enum)
├── components/
│   ├── LOOMGame.tsx                                    (NEW — top-level game component, lazy-loaded)
│   └── loom/                                           (NEW — all LOOM-specific code under here)
│       ├── engine/
│       │   ├── raycaster.ts                            (NEW — Phase 1)
│       │   ├── renderer.ts                             (NEW — Phase 1, WebGL2)
│       │   ├── gameLoop.ts                             (NEW — Phase 1)
│       │   ├── mapLoader.ts                            (NEW — Phase 1)
│       │   ├── collision.ts                            (NEW — Phase 1)
│       │   ├── input.ts                                (NEW — Phase 1, keyboard + pointer-lock)
│       │   └── audioController.ts                      (NEW — Phase 1, wraps soundService)
│       ├── entities/
│       │   ├── player.ts                               (NEW — Phase 1)
│       │   ├── weapons/
│       │   │   ├── IWeapon.ts                          (NEW — Phase 1, interface)
│       │   │   ├── brandedPen.ts                       (NEW — Phase 1)
│       │   │   └── neuralPulse.ts                      (NEW — Phase 1)
│       │   └── enemies/
│       │       └── intern.ts                           (NEW — Phase 1)
│       ├── hud/
│       │   ├── LoomHud.tsx                             (NEW — Phase 1, React overlay)
│       │   └── BootSequence.tsx                        (NEW — Phase 0, J0IN 0S boot animation)
│       ├── store/
│       │   └── cycleStore.ts                           (NEW — Phase 1, Zustand slice)
│       └── types.ts                                    (NEW — LOOM-internal types)
├── features/windows/registry.tsx                       (modify — register GAME_LOOM)
├── store/slices/filesystem.ts                          (modify — add LOOM.EXE)
├── store/slices/unlocks.ts                             (modify — add loom_archive.dat) [path TBD in Task 0.1]
├── data/content/loomLogs.ts                            (modify — add L-907 entry; deferred to Phase 5)
└── public/data/loom/                                   (NEW — game asset directory)
    ├── maps/
    │   └── cyc1_test.json                              (NEW — Phase 1)
    ├── sprites/
    │   ├── intern_walk_a.png                           (NEW — Phase 1, placeholder)
    │   ├── intern_walk_b.png                           (NEW — Phase 1)
    │   ├── intern_attack.png                           (NEW — Phase 1)
    │   ├── intern_death_a.png                          (NEW — Phase 1)
    │   ├── intern_death_b.png                          (NEW — Phase 1)
    │   ├── pen_idle.png                                (NEW — Phase 1, placeholder weapon view)
    │   ├── pen_fire.png                                (NEW — Phase 1)
    │   └── neural_pulse_proj.png                       (NEW — Phase 1, placeholder)
    └── textures/
        └── wall_lobby.png                              (NEW — Phase 1, placeholder)
```

---

# Phase 0 — Integrate LOOM into J0IN 0S

**Phase 0 Definition of Done:** A user opens l0b0tonline in their browser, boots J0IN 0S, locates the **L00M.EXE** file (via terminal command or desktop shortcut — whichever the existing pattern dictates), opens it, and a J0IN 0S window spawns containing a "LOOM TECHNOLOGIES — UNIT BOOTLOADER" boot sequence followed by a black canvas. No browser-console errors.

---

## Task 0.1: Discover l0b0tonline structure + verify dev environment

**Files:**
- Read-only inventory: confirm scout report's findings against actual l0b0tonline state
- No commits in this task

- [ ] **Step 1: Verify l0b0tonline repo + branch**

Run from `/Users/justinwest/Repos/l0b0tonline`:
```bash
cd /Users/justinwest/Repos/l0b0tonline
git status && git branch --show-current
```
Expected: clean working tree (or note any in-progress work). **Note the current branch name.** If on `main` / `master`, create a feature branch:
```bash
git checkout -b loom-game
```

- [ ] **Step 2: Verify dependencies installed and dev server works**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npm install 2>&1 | tail -10
```
Expected: clean install, no errors.

```bash
npm run dev 2>&1 | head -30 &
DEV_PID=$!
sleep 6
echo "--- check process ---"
ps -p $DEV_PID > /dev/null && echo "dev server running"
kill $DEV_PID 2>/dev/null
```
Expected: Vite dev server starts. Note the port it bound to (commonly `5173`, `5174`, or `3000`).

- [ ] **Step 3: Read & confirm key integration files**

Read each of the following and **report the actual line ranges that match the scout's findings**:

```bash
cat /Users/justinwest/Repos/l0b0tonline/types.ts | head -80
```
Expected: a `WindowType` enum or const-object. **Note the exact path/syntax** — is it `enum WindowType { ... }` or `const WindowType = { ... } as const`? Are entries SCREAMING_CASE or camelCase? Note the existing entries (e.g. `GAME_MEMORY`, `GAME_SHIPIT`, etc.).

```bash
sed -n '120,260p' /Users/justinwest/Repos/l0b0tonline/features/windows/registry.tsx
```
Expected: a `WINDOW_REGISTRY` object/Record with entries shaped like `{ lazy: () => import(...), category: 'game', icon: <...>, ... }`. **Note the exact entry shape** for an existing game (e.g. `GAME_MEMORY`).

```bash
sed -n '1,50p' /Users/justinwest/Repos/l0b0tonline/components/WindowManager.tsx
```
Expected: a `WindowProps` interface near top of file. **Note its exact shape** — what props does our LOOMGame component need to accept (`id`, `type`, `data`, `title`, `onClose`, `onSave`, etc.)?

```bash
ls /Users/justinwest/Repos/l0b0tonline/store/slices/
```
Expected: filesystem slice + others. Find the slice that handles **filesystem entries** and the one that handles **unlocks** — note their actual filenames.

```bash
cat /Users/justinwest/Repos/l0b0tonline/data/content/loomLogs.ts
```
Expected: existing LOOM_LOGS array with L-901..L-906 entries. **Note the entry shape** (`id`, `time`, `subject`, `status`, `content`).

```bash
cat /Users/justinwest/Repos/l0b0tonline/services/soundService.ts | head -100
```
Expected: existing soundService. **Note its exported API** — what methods can LOOM call to play sounds?

```bash
ls /Users/justinwest/Repos/l0b0tonline/components/ | grep -i game
ls /Users/justinwest/Repos/l0b0tonline/features/windows/types/
```
Expected: existing game components. Confirm ShipIt and Bardo live at the top-level `components/` path, mini-games at `features/windows/types/`.

```bash
cat /Users/justinwest/Repos/l0b0tonline/package.json | head -50
```
Expected: TypeScript, React, Vite, Zustand, Tailwind, Vitest, Playwright. Note the exact versions for our spec.

- [ ] **Step 4: Open J0IN 0S in a browser and locate the LOOM-icon insertion point**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npm run dev &
DEV_PID=$!
sleep 5
open "http://localhost:5173"
```
*(Adjust port if Step 2 reported a different one.)*

In the browser:
- Boot J0IN 0S to the desktop
- Find an existing game like ShipIt — note how it appears (icon on desktop? terminal command? both?)
- Note the exact mechanism we'll plug into for LOOM

Then kill the dev server:
```bash
kill $DEV_PID 2>/dev/null
```

- [ ] **Step 5: Report findings**

Status: DONE. Provide a summary report with:
- Current branch in l0b0tonline
- Vite dev port
- Exact `WindowType` declaration syntax + sample entries
- Exact `WINDOW_REGISTRY` entry shape (a literal example for `GAME_MEMORY` or similar)
- Exact `WindowProps` interface shape
- Path of filesystem slice and unlocks slice
- Path of soundService and its exported API surface
- Path where heavy games live (`components/SomethingGame.tsx`)
- How games appear in the UI (icon? terminal? both?)
- Anything in scout report that turned out to be different from reality

This task creates no files; the deliverable is the corrected map for Tasks 0.2–0.4.

---

## Task 0.2: Add `WindowType.GAME_LOOM` and register in `WINDOW_REGISTRY`

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/types.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/features/windows/registry.tsx`

- [ ] **Step 1: Add `GAME_LOOM` to `WindowType`**

Open `/Users/justinwest/Repos/l0b0tonline/types.ts` and find the `WindowType` declaration (path/line confirmed in Task 0.1).

If it's a TypeScript enum:
```typescript
export enum WindowType {
  // …existing entries…
  GAME_LOOM = 'GAME_LOOM',
}
```

If it's a const-object pattern (the alternate Task 0.1 might find):
```typescript
export const WindowType = {
  // …existing entries…
  GAME_LOOM: 'GAME_LOOM',
} as const;
```

**Match the existing convention exactly** — don't change style. Add `GAME_LOOM` next to other game entries (alphabetically or grouped, whichever the file does).

- [ ] **Step 2: Register `GAME_LOOM` in `WINDOW_REGISTRY`**

Open `/Users/justinwest/Repos/l0b0tonline/features/windows/registry.tsx` and find the `WINDOW_REGISTRY` declaration. Per Task 0.1's findings, **heavy games use Pattern B** (`category.kind: 'custom'` + a `render` function wrapping in `GameErrorBoundary`), and the import uses a named-export wrapper.

Add the import at the top of the file (next to existing icon imports):
```typescript
import { Gamepad2 } from 'lucide-react';  // already imported — confirm before adding
```

Add this entry to `WINDOW_REGISTRY`, modeled exactly on the `GAME_SHIP` (ShipItGame) entry that Task 0.1 quoted:
```typescript
[WindowType.GAME_LOOM]: {
  component: lazy(() => import('../../components/LOOMGame').then(m => ({ default: m.LOOMGame }))),
  category: {
    kind: 'custom',
    render: (Component, props) => (
      <GameErrorBoundary name="LOOMGame">
        <Component onClose={props.onClose ?? (() => {})} />
      </GameErrorBoundary>
    ),
  },
  icon: <Gamepad2 size={14} />,
},
```

This:
- Uses `lazy()` (already imported in this file — verify before assuming)
- Uses `.then(m => ({ default: m.LOOMGame }))` because LOOMGame is a **named export** (matching ShipItGame / BardoGame pattern)
- Uses `category.kind: 'custom'` and a `render` function so we control the close-button wiring (Pattern B from Task 0.1)
- Wraps in `GameErrorBoundary` so a runtime error doesn't crash the whole J0IN 0S

- [ ] **Step 3: Type-check**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -20
```
Expected: no errors related to our changes. If TS complains about the missing `LOOMGame` component — that's expected (we create it in Task 0.3). The lazy import is statically typed but only resolved at runtime.

If the lazy import fails type-checking, temporarily comment out the registry entry and we'll re-enable it after Task 0.3.

- [ ] **Step 4: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add types.ts features/windows/registry.tsx
git commit -m "Add WindowType.GAME_LOOM and register lazy LOOMGame component

Reserves the registry slot for the LOOM raycaster FPS, which will be
implemented as a J0IN 0S app starting in the next task."
```

---

## Task 0.3: Create `LOOMGame.tsx` skeleton with boot sequence + black canvas

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/LOOMGame.tsx`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/BootSequence.tsx`

- [ ] **Step 1: Create the BootSequence React component**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/BootSequence.tsx`:
```tsx
import { useEffect, useState } from 'react';

const BOOT_LINES = [
  'LOOM TECHNOLOGIES — UNIT BOOTLOADER',
  '',
  'BIOS_CHECK ............................... OK',
  'MOUNTING_FS .............................. OK',
  'LOADING_KERNEL ........................... OK',
  'ESTABLISHING_SECURE_CONNECTION ...........',
  'ACCESS_GRANTED',
  '',
  '> spawn unit#88-E in CYC1_LOBBY',
];

const LINE_DELAY_MS = 280;

interface BootSequenceProps {
  onComplete: () => void;
}

export function BootSequence({ onComplete }: BootSequenceProps) {
  const [visibleLines, setVisibleLines] = useState(0);

  useEffect(() => {
    if (visibleLines >= BOOT_LINES.length) {
      const t = setTimeout(onComplete, 600);
      return () => clearTimeout(t);
    }
    const t = setTimeout(() => setVisibleLines((n) => n + 1), LINE_DELAY_MS);
    return () => clearTimeout(t);
  }, [visibleLines, onComplete]);

  return (
    <div className="absolute inset-0 flex items-center justify-center bg-black font-mono text-sm leading-relaxed text-green-400">
      <div className="flex flex-col items-start">
        {BOOT_LINES.slice(0, visibleLines).map((line, i) => (
          <div key={i}>{line || ' '}</div>
        ))}
        {visibleLines < BOOT_LINES.length && (
          <div className="ml-1 inline-block h-4 w-2 animate-pulse bg-green-400" />
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Create the LOOMGame top-level component**

Create `/Users/justinwest/Repos/l0b0tonline/components/LOOMGame.tsx`:
```tsx
import { useRef, useState } from 'react';
import { BootSequence } from './loom/hud/BootSequence';

// Heavy games (BardoGame, ShipItGame) take only `onClose` as their prop.
// The registry's Pattern B render function feeds it from WindowProps.onClose.
interface LoomGameProps {
  onClose: () => void;
}

export function LOOMGame(_props: LoomGameProps) {
  const canvasRef = useRef<HTMLCanvasElement | null>(null);
  const [booted, setBooted] = useState(false);

  return (
    <div className="relative h-full w-full overflow-hidden bg-black">
      <canvas
        ref={canvasRef}
        className="h-full w-full"
        style={{ imageRendering: 'pixelated' }}
      />
      {!booted && <BootSequence onComplete={() => setBooted(true)} />}
    </div>
  );
}
```

Notes:
- **Named export** `export function LOOMGame` matches the BardoGame / ShipItGame convention. The registry entry handles the `.then(m => ({ default: m.LOOMGame }))` wrapper.
- Component takes only `onClose: () => void` — that's the prop shape Pattern B's render function supplies.
- The `imageRendering: 'pixelated'` style ensures the canvas's eventual nearest-neighbor scaling looks crisp.

- [ ] **Step 3: Type-check**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -20
```
Expected: no errors. If the registry entry from Task 0.2 was commented out, re-enable it now and verify the lazy-import resolves cleanly.

- [ ] **Step 4: Verify the component imports resolve**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npm run dev &
DEV_PID=$!
sleep 5
```

Open http://localhost:5173 in a browser, boot J0IN 0S, look for any browser-console errors. If `LOOMGame` fails to load, check the import path in `registry.tsx`.

Kill the dev server: `kill $DEV_PID`.

- [ ] **Step 5: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/LOOMGame.tsx components/loom/hud/BootSequence.tsx
git commit -m "Add LOOMGame skeleton with J0IN 0S boot sequence

Phase 0 placeholder. Window opens, boot text scrolls, ends on black
canvas. The actual game loop arrives in Phase 1."
```

---

## Task 0.4: Add `loom.exe` filesystem entry + `loom_archive.dat` unlock setup

**Files:**
- Modify: `/Users/justinwest/Repos/l0b0tonline/data/filesystem/nodes.ts` *(this is where the actual filesystem tree lives — NOT in `store/slices/`, per Task 0.1's correction)*
- (Possibly) Modify: `/Users/justinwest/Repos/l0b0tonline/store/slices/unlocksSlice.ts` *(only if we need to pre-register `loom_archive.dat`; per Task 0.1, unlocks are a `string[]` array — files only get added when unlocked, no pre-registration needed)*

- [ ] **Step 1: Add `loom.exe` to the filesystem nodes**

Open `/Users/justinwest/Repos/l0b0tonline/data/filesystem/nodes.ts`. Find existing EXE entries (e.g. `ship_it.exe`, `bardo.exe`) — match their exact shape and convention (lowercase filename + `type: 'EXE'`):

```typescript
"loom.exe": {
  name: "loom.exe",
  type: 'EXE',
  target: WindowType.GAME_LOOM,
  content: "INITIALIZING LOOM PROTOCOL..."
},
```

Insert it in the same parent directory/object where existing executables live (likely the desktop or root node — match the existing pattern for ship_it.exe / bardo.exe).

- [ ] **Step 2: Verify unlocks slice — no changes needed for Phase 0**

Per Task 0.1: `unlockedFiles` is a `string[]` array, and files are only added when unlocked (no pre-registration). The `loom_archive.dat` file gets pushed onto `unlockedFiles[]` after the C-reveal in Phase 5 — **no Phase 0 work required here**. Skip to Step 3.

If you want to verify, peek at `store/slices/unlocksSlice.ts`:
```bash
cd /Users/justinwest/Repos/l0b0tonline
head -40 store/slices/unlocksSlice.ts
```
Confirm the slice exposes `unlockFile(filename: string): boolean` and `isFileUnlocked(filename: string): boolean`. We use these in Phase 5; no edits required now.

- [ ] **Step 3: Type-check**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -20
```
Expected: no errors.

- [ ] **Step 4: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add store/slices/
git commit -m "Add LOOM.EXE filesystem entry and loom_archive unlock flag

LOOM.EXE appears on the J0IN 0S filesystem from boot. Clicking/running
it spawns the LOOM game window. The unlock flag will be set true after
the C-reveal in Phase 5."
```

---

## Task 0.5: Phase 0 DoD smoke test

**Files:**
- No code changes; this is a manual playtest gate

- [ ] **Step 1: Boot l0b0tonline locally**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npm run dev
```
Open http://localhost:5173 in a browser.

- [ ] **Step 2: Play through Phase 0 DoD**

Walk through this checklist. Each must pass:

| # | Behavior | Expected |
|---|---|---|
| 1 | l0b0tonline boots cleanly | no console errors during initial page load |
| 2 | J0IN 0S desktop reachable | log in / boot through to the desktop screen |
| 3 | LOOM.EXE visible | appears in filesystem (terminal `ls /desktop` or graphical view, depending on the J0IN 0S pattern) |
| 4 | LOOM.EXE openable | running it (terminal command or click) spawns a window |
| 5 | Window chrome correct | window title reads "L00M.EXE", close/drag/resize work normally |
| 6 | Boot sequence plays | green-on-black "LOOM TECHNOLOGIES — UNIT BOOTLOADER" lines scroll in over ~3s |
| 7 | Boot completes | all lines visible, then the boot-text disappears |
| 8 | Black canvas remains | window contents settle to a black canvas (game loop comes Phase 1) |
| 9 | Window closeable | close button kills the window cleanly, no console errors |
| 10 | Re-openable | running LOOM.EXE again works the same way |

If any fail, return to the relevant Task and fix.

- [ ] **Step 3: Save Phase 0 playtest log**

Create `/Users/justinwest/Repos/LOOM-DOOM/.claude/worktrees/awesome-sutherland-aad854/docs/playtests/2026-04-27-phase-0-dod.md`:
```markdown
# Phase 0 DoD — Playtest Log

**Date:** 2026-04-27
**Tester:** [your name / "agent" / "claude"]
**l0b0tonline commit:** [paste current git short hash from l0b0tonline]
**LOOM-DOOM plan commit:** [paste current git short hash from LOOM-DOOM]

## Checklist

| # | Behavior | Result |
|---|---|---|
| 1 | l0b0tonline boots cleanly | PASS / FAIL |
| 2 | J0IN 0S desktop reachable | PASS / FAIL |
| 3 | LOOM.EXE visible | PASS / FAIL |
| 4 | LOOM.EXE openable | PASS / FAIL |
| 5 | Window chrome correct | PASS / FAIL |
| 6 | Boot sequence plays | PASS / FAIL |
| 7 | Boot completes | PASS / FAIL |
| 8 | Black canvas remains | PASS / FAIL |
| 9 | Window closeable | PASS / FAIL |
| 10 | Re-openable | PASS / FAIL |

## Notes / known issues

[Anything weird, edge cases, etc.]

## Phase 0 Definition of Done

[X] All 10 checklist items pass — Phase 0 complete.
```

Replace PASS / FAIL with actual results.

- [ ] **Step 4: Commit playtest log**

```bash
cd /Users/justinwest/Repos/LOOM-DOOM/.claude/worktrees/awesome-sutherland-aad854
mkdir -p docs/playtests
git add docs/playtests/2026-04-27-phase-0-dod.md
git commit -m "Add Phase 0 DoD playtest log"
```

---

# Phase 1 — Tracer bullet

**Phase 1 Definition of Done:** Player opens LOOM, walks around a single room with WASD + mouse-look, fires the Branded Pen, fires the Neural Pulse with F, kills 3 Interns. The J0IN 0S terminal HUD overlays the canvas showing health/ammo/cycle state. CycleStore reports `cycle=1, corruption=0` throughout. **Every major subsystem talks to every other.**

---

## Task 1.1: Create `cycleStore` Zustand slice

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/store/cycleStore.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/store/cycleStore.test.ts`

- [ ] **Step 1: Write the failing test**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/store/cycleStore.test.ts`:
```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { useCycleStore } from './cycleStore';

describe('cycleStore', () => {
  beforeEach(() => {
    useCycleStore.getState().reset();
  });

  it('initializes with cycle=1, corruption=0, identityRevealed=false', () => {
    const s = useCycleStore.getState();
    expect(s.cycle).toBe(1);
    expect(s.corruption).toBe(0);
    expect(s.identityRevealed).toBe(false);
  });

  it('advanceCycle increments cycle and corruption', () => {
    const s = useCycleStore.getState();
    s.advanceCycle();
    expect(useCycleStore.getState().cycle).toBe(2);
    expect(useCycleStore.getState().corruption).toBe(1);
  });

  it('advanceCycle clamps at cycle=4, corruption=3', () => {
    const s = useCycleStore.getState();
    s.advanceCycle();
    s.advanceCycle();
    s.advanceCycle();
    s.advanceCycle();   // would be cycle=5 if unclamped
    s.advanceCycle();
    expect(useCycleStore.getState().cycle).toBe(4);
    expect(useCycleStore.getState().corruption).toBe(3);
  });

  it('revealIdentity flips identityRevealed', () => {
    useCycleStore.getState().revealIdentity();
    expect(useCycleStore.getState().identityRevealed).toBe(true);
  });

  it('reset returns to initial state', () => {
    const s = useCycleStore.getState();
    s.advanceCycle();
    s.revealIdentity();
    s.reset();
    expect(useCycleStore.getState().cycle).toBe(1);
    expect(useCycleStore.getState().corruption).toBe(0);
    expect(useCycleStore.getState().identityRevealed).toBe(false);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/store/cycleStore.test.ts
```
Expected: FAIL — `Cannot find module './cycleStore'`.

- [ ] **Step 3: Implement `cycleStore`**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/store/cycleStore.ts`:
```typescript
import { create } from 'zustand';

export type CycleNumber = 1 | 2 | 3 | 4;
export type CorruptionLevel = 0 | 1 | 2 | 3;

interface CycleState {
  cycle: CycleNumber;
  corruption: CorruptionLevel;
  identityRevealed: boolean;

  advanceCycle: () => void;
  revealIdentity: () => void;
  reset: () => void;
}

const initialState = {
  cycle: 1 as CycleNumber,
  corruption: 0 as CorruptionLevel,
  identityRevealed: false,
};

export const useCycleStore = create<CycleState>((set) => ({
  ...initialState,

  advanceCycle: () =>
    set((s) => ({
      cycle: Math.min(4, s.cycle + 1) as CycleNumber,
      corruption: Math.min(3, s.corruption + 1) as CorruptionLevel,
    })),

  revealIdentity: () => set({ identityRevealed: true }),

  reset: () => set(initialState),
}));
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/store/cycleStore.test.ts
```
Expected: PASS — all 5 tests green.

- [ ] **Step 5: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/store/cycleStore.ts components/loom/store/cycleStore.test.ts
git commit -m "Add LOOM cycleStore Zustand slice + tests

Tracks current cycle (1-4) and corruption level (0-3) across the four
simulation cycles. The HUD subscribes to this; identity-reveal trigger
flips the identityRevealed flag in Phase 5."
```

---

## Task 1.2: Hand-author `cyc1_test.json` map + map types

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc1_test.json`

- [ ] **Step 1: Define LOOM-internal types**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/types.ts`:
```typescript
// LOOM-internal type definitions.

import type { CycleNumber } from './store/cycleStore';

/** A cell value in a map's grid. 0 = empty/walkable, ≥1 = wall (texture index + 1). */
export type GridCell = number;

/** An entity placed in the map: enemy spawn, player start, item, exit, etc. */
export type ThingType =
  | 'player_start'
  | 'intern'
  | 'exit'
  // future: more enemy types, items, etc.
  ;

export interface MapThing {
  type: ThingType;
  /** Tile-coordinate position (not pixel). */
  x: number;
  y: number;
  /** Facing in radians, 0 = east. Optional for static items. */
  angle?: number;
}

export interface LoomMap {
  id: string;             // "cyc1_test"
  cycle: CycleNumber;
  music?: string;         // track key for soundService; optional in Phase 1
  /** Wall grid. grid[y][x] gives the cell at (x, y). 0 = walkable, n = wall texture index n-1. */
  grid: GridCell[][];
  /** Things placed in the map (enemies, player start, items). */
  things: MapThing[];
  /** Texture name lookup. textures[n-1] is the texture for grid value n. */
  textures: string[];
  /** Optional intermission text shown after the map ends. */
  intermissionText?: string;
}

export interface PlayerState {
  /** World-space position (in tile units, can be fractional). */
  x: number;
  y: number;
  /** Facing in radians, 0 = east. */
  angle: number;
  health: number;
  ammo: number;
  /** Currently held weapon's slot. Slot 0 = neural pulse (always available). */
  activeSlot: number;
  /** Neural pulse charge meter, 0..1. */
  pulseCharge: number;
}
```

- [ ] **Step 2: Hand-author the placeholder map**

Create `/Users/justinwest/Repos/l0b0tonline/public/data/loom/maps/cyc1_test.json`:
```json
{
  "id": "cyc1_test",
  "cycle": 1,
  "grid": [
    [1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1]
  ],
  "things": [
    { "type": "player_start", "x": 4, "y": 4, "angle": 0 },
    { "type": "intern", "x": 1.5, "y": 1.5 },
    { "type": "intern", "x": 6.5, "y": 1.5 },
    { "type": "intern", "x": 1.5, "y": 6.5 }
  ],
  "textures": ["wall_lobby"],
  "intermissionText": "TRACER BULLET COMPLETE."
}
```

This is an 8×8 tile map with walls around the border, a 6×6 walkable interior, player spawned in the middle, 3 Interns at three corners.

- [ ] **Step 3: Verify the JSON parses**

```bash
cd /Users/justinwest/Repos/l0b0tonline
node -e "console.log('OK', JSON.parse(require('fs').readFileSync('public/data/loom/maps/cyc1_test.json','utf8')).id)"
```
Expected: `OK cyc1_test`.

- [ ] **Step 4: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/types.ts public/data/loom/maps/cyc1_test.json
git commit -m "Add LOOM map types and the cyc1_test placeholder map

8x8 grid, walls around the border, 6x6 walkable interior, player +
3 Interns. Used for the Phase 1 tracer bullet."
```

---

## Task 1.3: Implement raycaster math + WebGL2 renderer

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/raycaster.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/raycaster.test.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/renderer.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/mapLoader.ts`

- [ ] **Step 1: Write raycaster tests**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/raycaster.test.ts`:
```typescript
import { describe, it, expect } from 'vitest';
import { castRay } from './raycaster';
import type { LoomMap } from '../types';

const SIMPLE_MAP: LoomMap = {
  id: 'test',
  cycle: 1,
  grid: [
    [1, 1, 1, 1, 1],
    [1, 0, 0, 0, 1],
    [1, 0, 0, 0, 1],
    [1, 0, 0, 0, 1],
    [1, 1, 1, 1, 1],
  ],
  things: [],
  textures: ['wall'],
};

describe('castRay', () => {
  it('hits the east wall when looking east from center', () => {
    // Player at (2.5, 2.5), looking east (angle=0). Map is 5 wide, walls at x=4.
    const hit = castRay(SIMPLE_MAP, 2.5, 2.5, 0);
    expect(hit.distance).toBeGreaterThan(1.4);
    expect(hit.distance).toBeLessThan(1.6);
    expect(hit.tileX).toBe(4);
    expect(hit.tileY).toBe(2);
  });

  it('hits the north wall when looking north', () => {
    // angle = -π/2 = looking up (-y). Wall at y=0.
    const hit = castRay(SIMPLE_MAP, 2.5, 2.5, -Math.PI / 2);
    expect(hit.distance).toBeGreaterThan(2.4);
    expect(hit.distance).toBeLessThan(2.6);
    expect(hit.tileY).toBe(0);
  });

  it('returns side flag for wall orientation (NS vs EW)', () => {
    const hitEW = castRay(SIMPLE_MAP, 2.5, 2.5, 0); // east — hits a NS wall
    const hitNS = castRay(SIMPLE_MAP, 2.5, 2.5, -Math.PI / 2); // north — hits an EW wall
    expect(hitEW.side).not.toBe(hitNS.side);
  });

  it('reports the hit textureIndex from the grid value', () => {
    const hit = castRay(SIMPLE_MAP, 2.5, 2.5, 0);
    expect(hit.textureIndex).toBe(0); // grid value was 1 → texture index 0
  });
});
```

- [ ] **Step 2: Run tests, verify they fail**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/engine/raycaster.test.ts
```
Expected: FAIL — `Cannot find module './raycaster'`.

- [ ] **Step 3: Implement the raycaster**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/raycaster.ts`:
```typescript
// DDA-style raycaster for a 2D grid map. Casts a single ray from a position
// at a given angle and returns the first wall-cell hit.
//
// Reference: Lode Vandevenne's classic raycaster tutorial
// (https://lodev.org/cgtutor/raycasting.html). The math is unchanged from
// 1992-era game programming — we just typed it in TypeScript.

import type { LoomMap } from '../types';

export interface RayHit {
  /** Euclidean distance from origin to wall hit (in tile units). */
  distance: number;
  /** Grid cell of the wall that was hit. */
  tileX: number;
  tileY: number;
  /** 0 = hit a NS wall (vertical edge); 1 = hit an EW wall (horizontal edge). */
  side: 0 | 1;
  /** Texture index for the wall (grid cell value minus 1). */
  textureIndex: number;
  /** Where on the wall slice we hit, 0..1 along the wall's tangent. Used for texture mapping. */
  wallX: number;
}

/** Maximum tile-distance to march before giving up. Should exceed the largest map. */
const MAX_DISTANCE = 64;

export function castRay(
  map: LoomMap,
  originX: number,
  originY: number,
  angle: number,
): RayHit {
  const dirX = Math.cos(angle);
  const dirY = Math.sin(angle);

  let mapX = Math.floor(originX);
  let mapY = Math.floor(originY);

  // Length of ray from one x or y side to next x or y side
  const deltaDistX = Math.abs(1 / dirX);
  const deltaDistY = Math.abs(1 / dirY);

  // Step direction and initial ray distance to nearest grid edge
  const stepX = dirX < 0 ? -1 : 1;
  const stepY = dirY < 0 ? -1 : 1;

  let sideDistX =
    dirX < 0 ? (originX - mapX) * deltaDistX : (mapX + 1 - originX) * deltaDistX;
  let sideDistY =
    dirY < 0 ? (originY - mapY) * deltaDistY : (mapY + 1 - originY) * deltaDistY;

  let side: 0 | 1 = 0;
  let distance = 0;

  while (distance < MAX_DISTANCE) {
    if (sideDistX < sideDistY) {
      sideDistX += deltaDistX;
      mapX += stepX;
      side = 0;
    } else {
      sideDistY += deltaDistY;
      mapY += stepY;
      side = 1;
    }

    if (mapY < 0 || mapY >= map.grid.length) break;
    const row = map.grid[mapY];
    if (!row || mapX < 0 || mapX >= row.length) break;
    const cell = row[mapX];

    if (cell > 0) {
      // Hit a wall; compute perpendicular distance (avoids fish-eye)
      distance =
        side === 0
          ? (mapX - originX + (1 - stepX) / 2) / dirX
          : (mapY - originY + (1 - stepY) / 2) / dirY;
      // Where on the wall did we hit, in [0,1)?
      const wallXraw =
        side === 0
          ? originY + distance * dirY
          : originX + distance * dirX;
      const wallX = wallXraw - Math.floor(wallXraw);

      return {
        distance,
        tileX: mapX,
        tileY: mapY,
        side,
        textureIndex: cell - 1,
        wallX,
      };
    }
  }

  // No hit — return a sentinel "infinite" hit
  return {
    distance: MAX_DISTANCE,
    tileX: mapX,
    tileY: mapY,
    side,
    textureIndex: -1,
    wallX: 0,
  };
}
```

- [ ] **Step 4: Run tests, verify pass**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx vitest run components/loom/engine/raycaster.test.ts
```
Expected: PASS — all 4 tests green. If any fail, reread the math and the test expectations carefully.

- [ ] **Step 5: Implement the WebGL2 renderer (minimal — flat-color walls for now)**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/renderer.ts`:
```typescript
// WebGL2 renderer. Phase 1 is intentionally simple: it renders flat-shaded
// walls (no textures yet) at internal 320x240 and scales to the canvas with
// nearest-neighbor.
//
// Phase 4+ adds textures, sprites, post-processing (CRT/glitch). For now,
// shade each column based on side (NS vs EW) and distance, output to canvas.

import type { LoomMap } from '../types';
import { castRay } from './raycaster';

const INTERNAL_W = 320;
const INTERNAL_H = 240;
const FOV = Math.PI / 3; // 60° field of view

const VS = `#version 300 es
in vec2 a_pos;
in vec2 a_uv;
out vec2 v_uv;
void main() {
  v_uv = a_uv;
  gl_Position = vec4(a_pos, 0.0, 1.0);
}`;

const FS = `#version 300 es
precision highp float;
in vec2 v_uv;
out vec4 outColor;
uniform sampler2D u_tex;
void main() {
  outColor = texture(u_tex, v_uv);
}`;

export class Renderer {
  private gl: WebGL2RenderingContext;
  private texture: WebGLTexture;
  private framebuffer: Uint8ClampedArray;

  constructor(canvas: HTMLCanvasElement) {
    const gl = canvas.getContext('webgl2');
    if (!gl) throw new Error('WebGL2 not supported');
    this.gl = gl;

    // Compile shaders
    const vs = compileShader(gl, gl.VERTEX_SHADER, VS);
    const fs = compileShader(gl, gl.FRAGMENT_SHADER, FS);
    const program = linkProgram(gl, vs, fs);
    gl.useProgram(program);

    // Fullscreen quad
    const quadBuf = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, quadBuf);
    gl.bufferData(
      gl.ARRAY_BUFFER,
      new Float32Array([
        // x,    y,    u,   v
        -1, -1, 0, 0,
         1, -1, 1, 0,
        -1,  1, 0, 1,
        -1,  1, 0, 1,
         1, -1, 1, 0,
         1,  1, 1, 1,
      ]),
      gl.STATIC_DRAW,
    );

    const aPos = gl.getAttribLocation(program, 'a_pos');
    const aUv  = gl.getAttribLocation(program, 'a_uv');
    gl.enableVertexAttribArray(aPos);
    gl.vertexAttribPointer(aPos, 2, gl.FLOAT, false, 16, 0);
    gl.enableVertexAttribArray(aUv);
    gl.vertexAttribPointer(aUv, 2, gl.FLOAT, false, 16, 8);

    // Texture for the internal-resolution framebuffer
    this.texture = gl.createTexture()!;
    gl.bindTexture(gl.TEXTURE_2D, this.texture);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
    gl.texImage2D(
      gl.TEXTURE_2D, 0, gl.RGBA, INTERNAL_W, INTERNAL_H, 0,
      gl.RGBA, gl.UNSIGNED_BYTE, null,
    );

    this.framebuffer = new Uint8ClampedArray(INTERNAL_W * INTERNAL_H * 4);
  }

  render(map: LoomMap, playerX: number, playerY: number, playerAngle: number) {
    // Clear framebuffer (sky on top, floor on bottom)
    for (let y = 0; y < INTERNAL_H; y++) {
      for (let x = 0; x < INTERNAL_W; x++) {
        const idx = (y * INTERNAL_W + x) * 4;
        if (y < INTERNAL_H / 2) {
          // Ceiling: dark gray-green
          this.framebuffer[idx]     = 18;
          this.framebuffer[idx + 1] = 28;
          this.framebuffer[idx + 2] = 18;
        } else {
          // Floor: slightly lighter
          this.framebuffer[idx]     = 28;
          this.framebuffer[idx + 1] = 28;
          this.framebuffer[idx + 2] = 28;
        }
        this.framebuffer[idx + 3] = 255;
      }
    }

    // Cast one ray per output column
    for (let col = 0; col < INTERNAL_W; col++) {
      const cameraX = (2 * col) / INTERNAL_W - 1;          // -1..1
      const rayAngle = playerAngle + Math.atan(cameraX * Math.tan(FOV / 2));
      const hit = castRay(map, playerX, playerY, rayAngle);

      // Project distance to wall slice height. Use perpendicular distance
      // (already computed in raycaster) so vertical lines stay vertical.
      const perpDist = hit.distance * Math.cos(rayAngle - playerAngle);
      const sliceHeight = Math.min(INTERNAL_H, INTERNAL_H / Math.max(perpDist, 0.05));
      const drawStart = Math.max(0, Math.floor((INTERNAL_H - sliceHeight) / 2));
      const drawEnd = Math.min(INTERNAL_H, Math.floor((INTERNAL_H + sliceHeight) / 2));

      // Side-based shading: EW walls slightly darker than NS walls
      const sideDarken = hit.side === 1 ? 0.7 : 1.0;
      // Distance-based shading
      const distFog = Math.max(0.25, 1 - perpDist / 12);
      const shade = sideDarken * distFog;

      // Wall color: green-tinted for placeholder
      const r = Math.floor(120 * shade);
      const g = Math.floor(180 * shade);
      const b = Math.floor(120 * shade);

      for (let y = drawStart; y < drawEnd; y++) {
        const idx = (y * INTERNAL_W + col) * 4;
        this.framebuffer[idx]     = r;
        this.framebuffer[idx + 1] = g;
        this.framebuffer[idx + 2] = b;
        this.framebuffer[idx + 3] = 255;
      }
    }

    // Upload framebuffer to GPU and draw
    const gl = this.gl;
    gl.bindTexture(gl.TEXTURE_2D, this.texture);
    gl.texSubImage2D(
      gl.TEXTURE_2D, 0, 0, 0, INTERNAL_W, INTERNAL_H,
      gl.RGBA, gl.UNSIGNED_BYTE, this.framebuffer,
    );
    gl.viewport(0, 0, gl.drawingBufferWidth, gl.drawingBufferHeight);
    gl.clearColor(0, 0, 0, 1);
    gl.clear(gl.COLOR_BUFFER_BIT);
    gl.drawArrays(gl.TRIANGLES, 0, 6);
  }
}

function compileShader(gl: WebGL2RenderingContext, type: number, src: string): WebGLShader {
  const shader = gl.createShader(type);
  if (!shader) throw new Error('createShader failed');
  gl.shaderSource(shader, src);
  gl.compileShader(shader);
  if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
    const log = gl.getShaderInfoLog(shader);
    throw new Error(`shader compile failed: ${log}`);
  }
  return shader;
}

function linkProgram(gl: WebGL2RenderingContext, vs: WebGLShader, fs: WebGLShader): WebGLProgram {
  const program = gl.createProgram();
  if (!program) throw new Error('createProgram failed');
  gl.attachShader(program, vs);
  gl.attachShader(program, fs);
  gl.linkProgram(program);
  if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
    const log = gl.getProgramInfoLog(program);
    throw new Error(`program link failed: ${log}`);
  }
  return program;
}
```

- [ ] **Step 6: Implement the map loader**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/mapLoader.ts`:
```typescript
import type { LoomMap } from '../types';

/**
 * Load a LOOM map from /public/data/loom/maps/<id>.json.
 * Throws if the fetch fails or the JSON doesn't validate against our shape.
 */
export async function loadMap(id: string): Promise<LoomMap> {
  const res = await fetch(`/data/loom/maps/${id}.json`);
  if (!res.ok) {
    throw new Error(`map fetch failed: ${id} (${res.status})`);
  }
  const data = (await res.json()) as unknown;
  if (!isLoomMap(data)) {
    throw new Error(`map ${id} failed schema validation`);
  }
  return data;
}

function isLoomMap(data: unknown): data is LoomMap {
  if (typeof data !== 'object' || data === null) return false;
  const m = data as Record<string, unknown>;
  return (
    typeof m.id === 'string' &&
    typeof m.cycle === 'number' &&
    Array.isArray(m.grid) &&
    Array.isArray(m.things) &&
    Array.isArray(m.textures)
  );
}
```

- [ ] **Step 7: Type-check**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -20
```
Expected: no errors.

- [ ] **Step 8: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/engine/raycaster.ts components/loom/engine/raycaster.test.ts components/loom/engine/renderer.ts components/loom/engine/mapLoader.ts
git commit -m "Add LOOM raycaster (DDA), WebGL2 renderer, map loader

Phase 1 minimum viable rendering: per-column ray casting, flat-shaded
walls at internal 320x240, scaled to canvas with nearest-neighbor.
Texture sampling and sprites come in later tasks."
```

---

## Task 1.4: Game loop + player input + wire renderer into LOOMGame

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/input.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/player.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/LOOMGame.tsx`

- [ ] **Step 1: Implement input controller**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/input.ts`:
```typescript
/**
 * Keyboard + pointer-lock mouse-look input. Owned by the game loop; reads
 * its current state synchronously each tick.
 */
export class InputController {
  private keys = new Set<string>();
  private mouseDeltaX = 0;
  /** Pending mouse-fire flag (consumed each tick). */
  private firePrimary = false;
  private firePulse = false;

  constructor(canvas: HTMLCanvasElement) {
    canvas.tabIndex = 0;

    const onKeyDown = (e: KeyboardEvent) => {
      this.keys.add(e.code);
      if (e.code === 'KeyF') this.firePulse = true;
    };
    const onKeyUp = (e: KeyboardEvent) => this.keys.delete(e.code);
    const onMouseMove = (e: MouseEvent) => {
      if (document.pointerLockElement === canvas) {
        this.mouseDeltaX += e.movementX;
      }
    };
    const onClick = () => {
      if (document.pointerLockElement === canvas) {
        this.firePrimary = true;
      } else {
        canvas.requestPointerLock();
      }
    };

    canvas.addEventListener('keydown', onKeyDown);
    canvas.addEventListener('keyup', onKeyUp);
    canvas.addEventListener('mousemove', onMouseMove);
    canvas.addEventListener('click', onClick);

    // Cleanup hook (caller should manage lifecycle)
    this._cleanup = () => {
      canvas.removeEventListener('keydown', onKeyDown);
      canvas.removeEventListener('keyup', onKeyUp);
      canvas.removeEventListener('mousemove', onMouseMove);
      canvas.removeEventListener('click', onClick);
    };
  }

  private _cleanup: () => void;

  isDown(code: string): boolean {
    return this.keys.has(code);
  }

  /** Consume mouse delta — call once per tick and act on it. */
  consumeMouseDelta(): number {
    const dx = this.mouseDeltaX;
    this.mouseDeltaX = 0;
    return dx;
  }

  consumeFirePrimary(): boolean {
    const f = this.firePrimary;
    this.firePrimary = false;
    return f;
  }

  consumeFirePulse(): boolean {
    const f = this.firePulse;
    this.firePulse = false;
    return f;
  }

  destroy() {
    this._cleanup();
  }
}
```

- [ ] **Step 2: Implement Player**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/player.ts`:
```typescript
import type { LoomMap, PlayerState } from '../types';
import type { InputController } from '../engine/input';

const MOVE_SPEED = 3.0;          // tiles per second
const TURN_SPEED = 0.0025;       // radians per pixel of mouse movement
const PULSE_RECHARGE_RATE = 1 / 3; // full charge in 3s

export class Player implements PlayerState {
  x: number;
  y: number;
  angle: number;
  health = 100;
  ammo = 50;
  activeSlot = 2; // Branded Pen by default
  pulseCharge = 1;

  constructor(spawnX: number, spawnY: number, spawnAngle: number) {
    this.x = spawnX;
    this.y = spawnY;
    this.angle = spawnAngle;
  }

  update(dt: number, input: InputController, map: LoomMap) {
    // Mouse look
    const dx = input.consumeMouseDelta();
    this.angle += dx * TURN_SPEED;

    // Movement (WASD)
    let mx = 0;
    let my = 0;
    if (input.isDown('KeyW')) { mx += Math.cos(this.angle); my += Math.sin(this.angle); }
    if (input.isDown('KeyS')) { mx -= Math.cos(this.angle); my -= Math.sin(this.angle); }
    if (input.isDown('KeyA')) { mx += Math.cos(this.angle - Math.PI / 2); my += Math.sin(this.angle - Math.PI / 2); }
    if (input.isDown('KeyD')) { mx += Math.cos(this.angle + Math.PI / 2); my += Math.sin(this.angle + Math.PI / 2); }

    // Normalize diagonals
    const len = Math.hypot(mx, my);
    if (len > 0) {
      mx /= len;
      my /= len;
    }

    const newX = this.x + mx * MOVE_SPEED * dt;
    const newY = this.y + my * MOVE_SPEED * dt;

    // Simple grid-cell collision: don't move into a wall cell. Check X and Y
    // independently for wall-sliding behavior.
    if (!isWallAt(map, newX, this.y)) this.x = newX;
    if (!isWallAt(map, this.x, newY)) this.y = newY;

    // Recharge pulse
    this.pulseCharge = Math.min(1, this.pulseCharge + PULSE_RECHARGE_RATE * dt);
  }
}

function isWallAt(map: LoomMap, x: number, y: number): boolean {
  const ix = Math.floor(x);
  const iy = Math.floor(y);
  if (iy < 0 || iy >= map.grid.length) return true;
  const row = map.grid[iy];
  if (!row || ix < 0 || ix >= row.length) return true;
  return row[ix] > 0;
}
```

- [ ] **Step 3: Implement the game loop**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`:
```typescript
import type { LoomMap } from '../types';
import { Player } from '../entities/player';
import { InputController } from './input';
import { Renderer } from './renderer';

export class GameLoop {
  private rafHandle: number | null = null;
  private lastTimeMs = 0;
  private renderer: Renderer;
  private input: InputController;
  private player: Player;
  private map: LoomMap;

  constructor(canvas: HTMLCanvasElement, map: LoomMap) {
    this.map = map;
    this.renderer = new Renderer(canvas);
    this.input = new InputController(canvas);

    const spawn = map.things.find((t) => t.type === 'player_start');
    if (!spawn) throw new Error('map has no player_start');
    this.player = new Player(spawn.x, spawn.y, spawn.angle ?? 0);
  }

  start() {
    this.lastTimeMs = performance.now();
    const tick = (nowMs: number) => {
      const dt = Math.min(0.1, (nowMs - this.lastTimeMs) / 1000);
      this.lastTimeMs = nowMs;
      this.player.update(dt, this.input, this.map);
      this.renderer.render(this.map, this.player.x, this.player.y, this.player.angle);
      this.rafHandle = requestAnimationFrame(tick);
    };
    this.rafHandle = requestAnimationFrame(tick);
  }

  stop() {
    if (this.rafHandle !== null) {
      cancelAnimationFrame(this.rafHandle);
      this.rafHandle = null;
    }
    this.input.destroy();
  }

  /** Exposed for HUD and weapon code in later tasks. */
  getPlayer(): Player {
    return this.player;
  }
}
```

- [ ] **Step 4: Wire into LOOMGame**

Edit `/Users/justinwest/Repos/l0b0tonline/components/LOOMGame.tsx`. Replace the file's entire content with:
```tsx
import { useEffect, useRef, useState } from 'react';
import { BootSequence } from './loom/hud/BootSequence';
import { loadMap } from './loom/engine/mapLoader';
import { GameLoop } from './loom/engine/gameLoop';

interface LoomGameProps {
  id: string;
  onClose: () => void;
}

export default function LOOMGame(_props: LoomGameProps) {
  const canvasRef = useRef<HTMLCanvasElement | null>(null);
  const [booted, setBooted] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (!booted || !canvasRef.current) return;
    let loop: GameLoop | null = null;
    let cancelled = false;
    (async () => {
      try {
        const map = await loadMap('cyc1_test');
        if (cancelled || !canvasRef.current) return;
        loop = new GameLoop(canvasRef.current, map);
        loop.start();
      } catch (e) {
        setError(e instanceof Error ? e.message : String(e));
      }
    })();
    return () => {
      cancelled = true;
      loop?.stop();
    };
  }, [booted]);

  return (
    <div className="relative h-full w-full overflow-hidden bg-black">
      <canvas
        ref={canvasRef}
        className="h-full w-full"
        style={{ imageRendering: 'pixelated' }}
        width={640}
        height={480}
      />
      {!booted && <BootSequence onComplete={() => setBooted(true)} />}
      {error && (
        <div className="absolute inset-0 flex items-center justify-center bg-black/80 font-mono text-sm text-red-400">
          ERROR: {error}
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 5: Type-check + manual smoke test**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -20
```
Expected: clean.

```bash
npm run dev
```
Open http://localhost:5173, boot J0IN 0S, run LOOM.EXE. After the boot sequence:
- The black canvas becomes a rendered first-person view of the placeholder room (greenish walls)
- Click the canvas — pointer lock engages
- WASD moves you around
- Mouse turns you
- Walls block movement
- No console errors

Quit when done.

- [ ] **Step 6: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/engine/input.ts components/loom/engine/gameLoop.ts components/loom/entities/player.ts components/LOOMGame.tsx
git commit -m "Add LOOM game loop, player movement, and wire renderer into LOOMGame

Player can now WASD-walk a single placeholder room with mouse-look via
pointer lock. Grid-cell collision keeps them inside walls. Frame loop
runs requestAnimationFrame at internal 320x240 scaled to canvas."
```

---

## Task 1.5: Branded Pen weapon (placeholder sprite + hitscan logic)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/IWeapon.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/brandedPen.ts`

- [ ] **Step 1: Write the IWeapon interface**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/IWeapon.ts`:
```typescript
import type { Player } from '../player';
import type { LoomMap } from '../../types';

/**
 * IWeapon — common interface for held weapons (slot 1+).
 * Slot 0 (Neural Pulse) is structurally different; it's not an IWeapon.
 */
export interface IWeapon {
  /** Display name for the HUD. */
  readonly name: string;
  /** Slot number (1-7). */
  readonly slot: number;
  /** Try to fire. Returns true if a shot fired (for ammo decrement, sound, etc.). */
  fire(player: Player, map: LoomMap): boolean;
}
```

- [ ] **Step 2: Implement Branded Pen**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/brandedPen.ts`:
```typescript
import type { IWeapon } from './IWeapon';
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import { castRay } from '../../engine/raycaster';

const DAMAGE = 10;
const RANGE = 16;

/**
 * Branded Pen — slot 2. Hitscan single-shot. Damages the first enemy or wall
 * the ray hits within RANGE tiles. For Phase 1 we only damage walls (a no-op);
 * Task 1.7 wires up enemy-hit logic.
 */
export class BrandedPen implements IWeapon {
  readonly name = 'Branded Pen';
  readonly slot = 2;

  fire(player: Player, map: LoomMap): boolean {
    if (player.ammo <= 0) return false;
    player.ammo -= 1;

    const hit = castRay(map, player.x, player.y, player.angle);
    if (hit.distance <= RANGE) {
      // Phase 1.5 + 1.7: when enemies exist, this is where we'd find the
      // closest enemy along the ray and damage it. For now, the wall hit
      // proves the ray-resolution path works.
      void DAMAGE;
    }

    return true;
  }
}
```

(The `DAMAGE` constant + `void DAMAGE` looks awkward but signals intent: this is the value Task 1.7 will use when the enemy-hit logic is wired in. Removing it entirely would lose the design intent.)

- [ ] **Step 3: Type-check**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npx tsc --noEmit 2>&1 | head -10
```
Expected: clean.

- [ ] **Step 4: Wire firing into the game loop**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`. Add Branded Pen handling — change the `start()` method's tick function to also handle weapon firing:

Find this block:
```typescript
this.player.update(dt, this.input, this.map);
this.renderer.render(this.map, this.player.x, this.player.y, this.player.angle);
```

Replace it with:
```typescript
this.player.update(dt, this.input, this.map);
if (this.input.consumeFirePrimary()) {
  this.brandedPen.fire(this.player, this.map);
}
this.renderer.render(this.map, this.player.x, this.player.y, this.player.angle);
```

Add the import and instance at the top of `GameLoop`:
```typescript
import { BrandedPen } from '../entities/weapons/brandedPen';
// …existing imports

export class GameLoop {
  // …existing fields
  private brandedPen = new BrandedPen();
  // …
}
```

- [ ] **Step 5: Manual smoke test**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npm run dev
```
Open the game. Click the canvas to enter pointer lock. Click again to fire — confirm the player's ammo drops in the console (we'll wire the HUD in Task 1.8). Add a temporary log if needed:
```typescript
console.log('ammo', this.player.ammo);
```
Remove the log before committing.

- [ ] **Step 6: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/entities/weapons/IWeapon.ts components/loom/entities/weapons/brandedPen.ts components/loom/engine/gameLoop.ts
git commit -m "Add Branded Pen weapon + IWeapon interface; wire fire to mouse click

Hitscan ray of damage and range constants in place. Enemy-hit resolution
arrives in Task 1.7 once the Intern actor exists."
```

---

## Task 1.6: Neural Pulse always-on weapon (F key)

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/neuralPulse.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

- [ ] **Step 1: Implement Neural Pulse**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/neuralPulse.ts`:
```typescript
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import { castRay } from '../../engine/raycaster';

const DAMAGE = 5;
const RANGE = 12;
const COST = 1; // full charge consumed per shot

/**
 * Neural Pulse — slot 0, always available. Body-mounted; fires from the
 * headband regardless of which weapon is currently held. Recharges over ~3s
 * (managed in Player.update). No ammo, just charge.
 */
export class NeuralPulse {
  readonly name = 'Neural Pulse';
  readonly slot = 0;

  fire(player: Player, map: LoomMap): boolean {
    if (player.pulseCharge < COST) return false;
    player.pulseCharge -= COST;

    const hit = castRay(map, player.x, player.y, player.angle);
    if (hit.distance <= RANGE) {
      // Same TODO as Branded Pen: enemy-hit wiring in Task 1.7.
      void DAMAGE;
    }

    return true;
  }
}
```

- [ ] **Step 2: Wire firing into the game loop**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`. Add a `NeuralPulse` instance and consume the F-fire flag:

Add the import:
```typescript
import { NeuralPulse } from '../entities/weapons/neuralPulse';
```

Add the field:
```typescript
private neuralPulse = new NeuralPulse();
```

Update the tick logic:
```typescript
this.player.update(dt, this.input, this.map);
if (this.input.consumeFirePrimary()) {
  this.brandedPen.fire(this.player, this.map);
}
if (this.input.consumeFirePulse()) {
  this.neuralPulse.fire(this.player, this.map);
}
this.renderer.render(this.map, this.player.x, this.player.y, this.player.angle);
```

- [ ] **Step 3: Manual smoke test**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npm run dev
```
Open the game. After pointer lock, press F. Confirm `player.pulseCharge` drops to 0 (temporary console.log if needed) and recharges back to 1 over ~3s. Remove debug logs.

- [ ] **Step 4: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/entities/weapons/neuralPulse.ts components/loom/engine/gameLoop.ts
git commit -m "Add Neural Pulse always-on weapon, bound to F key

Slot 0; charge meter consumed per shot, recharges over 3s. The body-
mounted headband framing comes through visually in Phase 4 when the
HUD shows pulse charge."
```

---

## Task 1.7: Intern enemy + AI + sprite rendering

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/intern.ts`
- Create: `/Users/justinwest/Repos/l0b0tonline/public/data/loom/sprites/intern_idle.png` (placeholder)
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/renderer.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/brandedPen.ts`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/neuralPulse.ts`

- [ ] **Step 1: Create a placeholder Intern sprite**

The simplest thing that works: a 32×64 PNG with a green-on-dark filled rectangle and a small dot for the "head." If you have access to an image tool:
- Open any image editor / paint program
- Create 32×64 transparent PNG
- Fill with a dark olive green (#5e6b3a)
- Add a slightly lighter rectangle for the head/body
- Save as `/Users/justinwest/Repos/l0b0tonline/public/data/loom/sprites/intern_idle.png`

Or generate via `node` + `sharp` / `pngjs` if available — but a 32×64 placeholder rendered in any tool works. **Just commit one PNG; we're not asking the engineer to make sprite animations yet — those come in Task 1.8 / later phases.**

- [ ] **Step 2: Implement the Intern**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/enemies/intern.ts`:
```typescript
import type { Player } from '../player';
import type { LoomMap } from '../../types';

const HEALTH = 20;
const SPEED = 1.0;        // tiles per second
const SIGHT_RANGE = 8;    // tiles

export type EnemyState = 'idle' | 'chase' | 'pain' | 'dead';

export class Intern {
  x: number;
  y: number;
  health = HEALTH;
  state: EnemyState = 'idle';

  constructor(x: number, y: number) {
    this.x = x;
    this.y = y;
  }

  update(dt: number, player: Player, _map: LoomMap) {
    if (this.state === 'dead') return;

    const dx = player.x - this.x;
    const dy = player.y - this.y;
    const dist = Math.hypot(dx, dy);

    if (this.state === 'idle' && dist < SIGHT_RANGE) {
      this.state = 'chase';
    }

    if (this.state === 'chase') {
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
    // Auto-recover after a brief moment (in real game, animate pain frames)
    setTimeout(() => {
      if (this.state === 'pain') this.state = 'chase';
    }, 200);
  }
}
```

- [ ] **Step 3: Add enemy hit-detection helper**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/entities/weapons/brandedPen.ts`. Replace `fire` with a version that takes the enemy list and resolves hits:
```typescript
import type { IWeapon } from './IWeapon';
import type { Player } from '../player';
import type { LoomMap } from '../../types';
import type { Intern } from '../enemies/intern';
import { castRay } from '../../engine/raycaster';

const DAMAGE = 10;
const RANGE = 16;

export class BrandedPen implements IWeapon {
  readonly name = 'Branded Pen';
  readonly slot = 2;

  fire(player: Player, map: LoomMap, enemies: Intern[]): boolean {
    if (player.ammo <= 0) return false;
    player.ammo -= 1;

    const wallHit = castRay(map, player.x, player.y, player.angle);

    // Find first live enemy along the ray, closer than the wall hit.
    const dirX = Math.cos(player.angle);
    const dirY = Math.sin(player.angle);
    let closest: { enemy: Intern; dist: number } | null = null;
    for (const e of enemies) {
      if (e.state === 'dead') continue;
      const ex = e.x - player.x;
      const ey = e.y - player.y;
      const along = ex * dirX + ey * dirY;
      if (along <= 0 || along > RANGE) continue;
      const perp = Math.abs(ex * dirY - ey * dirX);
      if (perp > 0.4) continue; // off-axis miss tolerance
      if (along > wallHit.distance) continue;
      if (!closest || along < closest.dist) closest = { enemy: e, dist: along };
    }

    if (closest) closest.enemy.takeDamage(DAMAGE);
    return true;
  }
}
```

Apply the same pattern to `neuralPulse.ts` (with `DAMAGE = 5`, `RANGE = 12`):
```typescript
// near the top:
import type { Intern } from '../enemies/intern';

// fire signature:
fire(player: Player, map: LoomMap, enemies: Intern[]): boolean {
  if (player.pulseCharge < 1) return false;
  player.pulseCharge -= 1;

  const wallHit = castRay(map, player.x, player.y, player.angle);
  const dirX = Math.cos(player.angle);
  const dirY = Math.sin(player.angle);
  let closest: { enemy: Intern; dist: number } | null = null;
  for (const e of enemies) {
    if (e.state === 'dead') continue;
    const ex = e.x - player.x;
    const ey = e.y - player.y;
    const along = ex * dirX + ey * dirY;
    if (along <= 0 || along > 12) continue;
    const perp = Math.abs(ex * dirY - ey * dirX);
    if (perp > 0.4) continue;
    if (along > wallHit.distance) continue;
    if (!closest || along < closest.dist) closest = { enemy: e, dist: along };
  }
  if (closest) closest.enemy.takeDamage(5);
  return true;
}
```

- [ ] **Step 4: Spawn Interns + tick them in the game loop**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`. Add Intern handling:

```typescript
// imports
import { Intern } from '../entities/enemies/intern';

// add field:
private enemies: Intern[] = [];

// in constructor, after `this.player = ...`:
for (const t of map.things) {
  if (t.type === 'intern') {
    this.enemies.push(new Intern(t.x, t.y));
  }
}

// tick — replace the relevant block:
this.player.update(dt, this.input, this.map);
for (const e of this.enemies) e.update(dt, this.player, this.map);
if (this.input.consumeFirePrimary()) this.brandedPen.fire(this.player, this.map, this.enemies);
if (this.input.consumeFirePulse())   this.neuralPulse.fire(this.player, this.map, this.enemies);
this.renderer.render(this.map, this.player.x, this.player.y, this.player.angle, this.enemies);
```

- [ ] **Step 5: Add sprite billboarding to the renderer**

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/renderer.ts`. Update `render` signature and add sprite drawing.

Replace the `render` method's signature with:
```typescript
render(map: LoomMap, playerX: number, playerY: number, playerAngle: number, enemies: Intern[]) {
```

Add the import at the top: `import type { Intern } from '../entities/enemies/intern';`

Track per-column wall depth so we can z-test sprites against walls. Replace the wall loop with one that also writes a `zBuffer`:

Modify the wall loop section to maintain a `zBuffer: number[]` of length `INTERNAL_W`. Initialize before the loop:
```typescript
const zBuffer = new Array<number>(INTERNAL_W);
```

Inside the per-column loop, after computing `perpDist`, add `zBuffer[col] = perpDist;` before the wall draw.

Then after the wall loop, draw sprites:
```typescript
// Sprite billboarding pass — draw enemies as colored rectangles for Phase 1
// (textured sprites added later). Sort back-to-front for correct overlap.
const livingEnemies = enemies.filter((e) => e.state !== 'dead');
const sortedEnemies = livingEnemies
  .map((e) => ({ e, dist: Math.hypot(e.x - playerX, e.y - playerY) }))
  .sort((a, b) => b.dist - a.dist);

for (const { e, dist } of sortedEnemies) {
  // Vector from player to enemy
  const ex = e.x - playerX;
  const ey = e.y - playerY;
  // Project into camera space: rotate by -playerAngle
  const cosA = Math.cos(-playerAngle);
  const sinA = Math.sin(-playerAngle);
  const cx = ex * cosA - ey * sinA;
  const cy = ex * sinA + ey * cosA;
  if (cx <= 0.1) continue; // behind player

  const screenX = Math.floor((INTERNAL_W / 2) * (1 + cy / cx / Math.tan(FOV / 2)));
  const spriteHeight = Math.min(INTERNAL_H, INTERNAL_H / cx);
  const drawStartY = Math.floor((INTERNAL_H - spriteHeight) / 2);
  const drawEndY = Math.floor((INTERNAL_H + spriteHeight) / 2);
  const spriteWidth = Math.floor(spriteHeight * 0.5); // Intern is taller than wide
  const drawStartX = Math.max(0, screenX - Math.floor(spriteWidth / 2));
  const drawEndX = Math.min(INTERNAL_W - 1, screenX + Math.floor(spriteWidth / 2));

  for (let x = drawStartX; x <= drawEndX; x++) {
    if (x < 0 || x >= INTERNAL_W) continue;
    if (zBuffer[x] !== undefined && cx > zBuffer[x]) continue; // behind a wall
    for (let y = Math.max(0, drawStartY); y < Math.min(INTERNAL_H, drawEndY); y++) {
      const idx = (y * INTERNAL_W + x) * 4;
      this.framebuffer[idx]     = 100;
      this.framebuffer[idx + 1] = 140;
      this.framebuffer[idx + 2] = 80;
      this.framebuffer[idx + 3] = 255;
    }
  }
}
```

This is intentionally crude — colored rectangles where each Intern is. Real sprite textures arrive in a later phase; for Phase 1, "I can see and shoot a thing in 3D space" is the goal.

- [ ] **Step 6: Manual smoke test**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npm run dev
```
Open the game. After boot:
- 3 olive-green rectangles visible — the 3 Interns
- They walk toward the player when within 8 tiles
- Click fire — Branded Pen kills Interns (3 hits each at 10 damage; HP 20)
- Press F — Neural Pulse weakens them
- Each killed Intern stops moving and disappears (well, its rect doesn't draw because state==='dead')

If the rays don't hit reliably: the perpendicular tolerance constant `0.4` may need tuning. Adjust and retest.

- [ ] **Step 7: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/entities/enemies/intern.ts components/loom/entities/weapons/brandedPen.ts components/loom/entities/weapons/neuralPulse.ts components/loom/engine/gameLoop.ts components/loom/engine/renderer.ts public/data/loom/sprites/intern_idle.png
git commit -m "Add Intern enemy + sprite billboarding + weapon hit resolution

Interns chase the player when within 8 tiles, take damage from Branded
Pen (10 dmg) and Neural Pulse (5 dmg), die at 0 HP. Renderer draws
billboarded sprite columns z-tested against the wall depth buffer."
```

---

## Task 1.8: `LoomHud` React overlay

**Files:**
- Create: `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/LoomHud.tsx`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/LOOMGame.tsx`
- Modify: `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`

- [ ] **Step 1: Expose player state to React**

The HUD needs to read player state every frame. Simplest pattern: `GameLoop` exposes a `getSnapshot()` method that React polls via `useSyncExternalStore`, and we wire the cycleStore in directly.

Edit `/Users/justinwest/Repos/l0b0tonline/components/loom/engine/gameLoop.ts`. Add a snapshot method:
```typescript
// Add an interface near the top of the file:
export interface GameSnapshot {
  health: number;
  ammo: number;
  pulseCharge: number;
  activeSlot: number;
  zone: string;
}

// Then in GameLoop:
getSnapshot(): GameSnapshot {
  return {
    health: this.player.health,
    ammo: this.player.ammo,
    pulseCharge: this.player.pulseCharge,
    activeSlot: this.player.activeSlot,
    zone: this.map.id,
  };
}
```

- [ ] **Step 2: Build the React HUD**

Create `/Users/justinwest/Repos/l0b0tonline/components/loom/hud/LoomHud.tsx`:
```tsx
import { useEffect, useState } from 'react';
import { useCycleStore } from '../store/cycleStore';
import type { GameSnapshot } from '../engine/gameLoop';

interface LoomHudProps {
  /** Polled each frame to get the latest game state. */
  getSnapshot: () => GameSnapshot;
}

const WEAPON_NAMES: Record<number, string> = {
  0: 'neural_pulse.exe',
  1: 'drum_stick.exe',
  2: 'branded_pen.exe',
  3: 'spam_filter.exe',
};

export function LoomHud({ getSnapshot }: LoomHudProps) {
  const [snap, setSnap] = useState<GameSnapshot>(() => getSnapshot());
  const cycle = useCycleStore((s) => s.cycle);
  const corruption = useCycleStore((s) => s.corruption);

  useEffect(() => {
    let h: number;
    const tick = () => {
      setSnap(getSnapshot());
      h = requestAnimationFrame(tick);
    };
    h = requestAnimationFrame(tick);
    return () => cancelAnimationFrame(h);
  }, [getSnapshot]);

  const simState =
    corruption === 0 ? 'nominal'
    : corruption === 1 ? 'drift detected'
    : corruption === 2 ? 'BREACH'
    : 'DISSOLVING';

  const cycleNumber = 8491 + cycle;
  const weaponName = WEAPON_NAMES[snap.activeSlot] ?? 'unknown.exe';

  return (
    <div className="pointer-events-none absolute bottom-0 left-0 right-0 z-10 px-3 pb-2 font-mono text-[11px] leading-relaxed text-green-400">
      <div>
        [unit#88-E] &nbsp;
        bio: <span className="text-green-300">{snap.health}%</span> &nbsp;/&nbsp;
        queue: <span className="text-yellow-300">{snap.ammo}</span> &nbsp;/&nbsp;
        active: <span className="text-green-300">{weaponName}</span> &nbsp;/&nbsp;
        zone: <span className="text-green-300">{snap.zone}</span>
      </div>
      <div className="text-green-700">
        &gt; cycle {cycleNumber} &nbsp;//&nbsp; simulation {simState} &nbsp;//&nbsp; surveillance: ON
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Wire HUD into LOOMGame**

Edit `/Users/justinwest/Repos/l0b0tonline/components/LOOMGame.tsx`. Add the HUD overlay. Replace the file with:

```tsx
import { useEffect, useRef, useState } from 'react';
import { BootSequence } from './loom/hud/BootSequence';
import { LoomHud } from './loom/hud/LoomHud';
import { loadMap } from './loom/engine/mapLoader';
import { GameLoop } from './loom/engine/gameLoop';

interface LoomGameProps {
  id: string;
  onClose: () => void;
}

export default function LOOMGame(_props: LoomGameProps) {
  const canvasRef = useRef<HTMLCanvasElement | null>(null);
  const loopRef = useRef<GameLoop | null>(null);
  const [booted, setBooted] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [loopReady, setLoopReady] = useState(false);

  useEffect(() => {
    if (!booted || !canvasRef.current) return;
    let cancelled = false;
    (async () => {
      try {
        const map = await loadMap('cyc1_test');
        if (cancelled || !canvasRef.current) return;
        loopRef.current = new GameLoop(canvasRef.current, map);
        loopRef.current.start();
        setLoopReady(true);
      } catch (e) {
        setError(e instanceof Error ? e.message : String(e));
      }
    })();
    return () => {
      cancelled = true;
      loopRef.current?.stop();
      loopRef.current = null;
      setLoopReady(false);
    };
  }, [booted]);

  return (
    <div className="relative h-full w-full overflow-hidden bg-black">
      <canvas
        ref={canvasRef}
        className="h-full w-full"
        style={{ imageRendering: 'pixelated' }}
        width={640}
        height={480}
      />
      {!booted && <BootSequence onComplete={() => setBooted(true)} />}
      {loopReady && loopRef.current && (
        <LoomHud getSnapshot={() => loopRef.current!.getSnapshot()} />
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

- [ ] **Step 4: Manual smoke test**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npm run dev
```
Open the game. After boot:
- The bottom-left of the LOOM window shows two lines of green text:
  - Line 1: `[unit#88-E] bio: 100% / queue: 50 / active: branded_pen.exe / zone: cyc1_test`
  - Line 2: `> cycle 8492 // simulation nominal // surveillance: ON`
- Fire the Pen — `queue:` decrements
- Take damage by waiting for an Intern to attack you (or hardcode `player.health -= 1` in `update` temporarily) — `bio:` decrements
- Press F — pulse charge (not displayed yet — that's a Phase 4 enhancement) consumes silently

Quit when done.

- [ ] **Step 5: Commit**

```bash
cd /Users/justinwest/Repos/l0b0tonline
git add components/loom/hud/LoomHud.tsx components/loom/engine/gameLoop.ts components/LOOMGame.tsx
git commit -m "Add LoomHud J0IN 0S terminal HUD overlay (clean / corruption=0)

React component overlaid on the WebGL canvas; subscribes to cycleStore
and polls the GameLoop snapshot each rAF. Renders the spec'd two-line
terminal layout with bio/queue/active/zone + cycle/sim-state/surveillance."
```

---

## Task 1.9: Phase 1 integration playtest + final commit

**Files:**
- Create: `/Users/justinwest/Repos/LOOM-DOOM/.claude/worktrees/awesome-sutherland-aad854/docs/playtests/2026-04-27-phase-1-tracer-bullet.md`

- [ ] **Step 1: Run the full integration test**

```bash
cd /Users/justinwest/Repos/l0b0tonline
npm run dev
```
Open http://localhost:5173 in a browser. Boot J0IN 0S, run LOOM.EXE.

Walk the **playtest checklist:**

| # | Behavior | Expected |
|---|---|---|
| 1 | LOOM window opens | from J0IN 0S desktop |
| 2 | Boot sequence plays | green-on-black scrolling boot text, ends after ~3s |
| 3 | Map renders | walls visible in 3D, greenish flat-shaded |
| 4 | Pointer lock works | clicking the canvas locks mouse, esc unlocks |
| 5 | WASD movement | player moves forward/back/strafe; collision with walls |
| 6 | Mouse-look | turning works smoothly |
| 7 | Branded Pen fires | mouse click drops `queue:` by 1 |
| 8 | Branded Pen damages | hitting an Intern reduces their health (verify via 3 hits to kill) |
| 9 | Neural Pulse fires | F key consumes pulse charge (verify with debug log if needed) |
| 10 | Neural Pulse damages | F key hitting an Intern reduces their health (verify via 4 hits to kill) |
| 11 | Interns chase | when player is within 8 tiles, Interns walk toward you |
| 12 | Interns die | reaching 0 HP, they stop drawing (state='dead') |
| 13 | HUD renders | bottom-left text shows bio/queue/active/zone + cycle line |
| 14 | HUD updates real-time | bio drops on damage, queue drops on Pen fire |
| 15 | Window closeable | close button kills the game cleanly, no console errors |

If ALL 15 pass — Phase 1 complete.

- [ ] **Step 2: Save Phase 1 playtest log**

Create `/Users/justinwest/Repos/LOOM-DOOM/.claude/worktrees/awesome-sutherland-aad854/docs/playtests/2026-04-27-phase-1-tracer-bullet.md`:
```markdown
# Phase 1 Tracer Bullet — Playtest Log

**Date:** 2026-04-27
**Tester:** [your name / "agent" / "claude"]
**l0b0tonline commit:** [paste current git short hash from l0b0tonline]

## Checklist

| # | Behavior | Result |
|---|---|---|
| 1 | LOOM window opens | PASS / FAIL |
| 2 | Boot sequence plays | PASS / FAIL |
| 3 | Map renders | PASS / FAIL |
| 4 | Pointer lock works | PASS / FAIL |
| 5 | WASD movement | PASS / FAIL |
| 6 | Mouse-look | PASS / FAIL |
| 7 | Branded Pen fires | PASS / FAIL |
| 8 | Branded Pen damages | PASS / FAIL |
| 9 | Neural Pulse fires | PASS / FAIL |
| 10 | Neural Pulse damages | PASS / FAIL |
| 11 | Interns chase | PASS / FAIL |
| 12 | Interns die | PASS / FAIL |
| 13 | HUD renders | PASS / FAIL |
| 14 | HUD updates real-time | PASS / FAIL |
| 15 | Window closeable | PASS / FAIL |

## Notes / known issues

[Anything that worked weirdly, sprite offset issues, etc.]

## Phase 1 Definition of Done

[X] All 15 checklist items pass — tracer bullet complete.
```

Replace PASS / FAIL with actual results.

- [ ] **Step 3: Commit playtest log**

```bash
cd /Users/justinwest/Repos/LOOM-DOOM/.claude/worktrees/awesome-sutherland-aad854
mkdir -p docs/playtests
git add docs/playtests/2026-04-27-phase-1-tracer-bullet.md
git commit -m "Add Phase 1 tracer-bullet playtest log (DoD met)"
```

- [ ] **Step 4: Verify branch state**

```bash
cd /Users/justinwest/Repos/LOOM-DOOM/.claude/worktrees/awesome-sutherland-aad854
git log --oneline | head -10
cd /Users/justinwest/Repos/l0b0tonline
git log --oneline | head -15
```
Both repos: clean trees, expected commits visible.

---

# Self-review

Spec coverage check (against `docs/superpowers/specs/2026-04-27-loom-doom-design.md`):

| Spec section | Phase 0+1 coverage |
|---|---|
| §3.1 Identity (DEFECTIVE UNIT #88-E) | ✅ HUD shows `[unit#88-E]`; LOOMGame component renders with that framing |
| §3.4 C-reveal | ❌ Out of scope — Phase 5 |
| §4 Cycle structure | ✅ partial — cycle 1 active; CycleStore Zustand slice in place |
| §5.1 Weapon roster | ✅ partial — Neural Pulse (Slot 0) + Branded Pen (Slot 2) |
| §5.2 Enemy taxonomy | ✅ partial — Intern (Cycle 1 basic) |
| §6.1 HUD J0IN 0S terminal | ✅ clean (corruption=0) mode; corruption modes deferred to Phase 3+ |
| §6.2 Audio | ❌ Phase 6 — placeholder only, no real audio |
| §6.3 Maps | ✅ partial — 1 placeholder map; rest in Phase 2+ |
| §7 Architecture | ✅ Repo layout, raycaster, renderer, game loop, HUD all done per spec |
| §7.4 Render pipeline | ✅ flat-shaded walls + sprite billboarding via z-buffer; textures + post-FX in later phases |
| §7.5 Validation | ✅ Vitest tests for cycleStore + raycaster; type-check clean; manual playtest log |
| §8 Phase 0 DoD | ✅ Task 0.5 |
| §8 Phase 1 DoD | ✅ Task 1.9 |
| §9 Risk inventory | n/a — meta |
| §10 Out of scope | ✅ enforced; no character-select / multiplayer / etc. |

Placeholder scan: `void DAMAGE` markers in Tasks 1.5–1.6 are intentional (signal Task 1.7 wires them up); replaced with real usage in 1.7. No "TODO" / "TBD" left at the end of Phase 1. ✅

Type consistency: class/interface names verified across tasks:
- `CycleStore` / `useCycleStore` — Task 1.1 + reference in Task 1.8 (consistent)
- `LoomMap` / `MapThing` / `PlayerState` — Task 1.2 (defined) + used in 1.3, 1.4, 1.5, 1.6, 1.7
- `IWeapon` / `BrandedPen` / `NeuralPulse` — Task 1.5 + 1.6 + reference in 1.7
- `Intern` — Task 1.7 + referenced in 1.5, 1.6 weapon updates
- `GameLoop` / `GameSnapshot` — Task 1.4 (initial) + extended in 1.7, 1.8
- `LoomHud` — Task 1.8

All references match. ✅

---

# Done condition for this plan

Phase 1 complete when Task 1.9 Step 1's 15-item playtest checklist all pass. At that point:
- LOOM is a clickable, runnable app inside J0IN 0S at l0b0tonline
- Players can navigate, fight one enemy type, see the J0IN 0S terminal HUD
- The full subsystem map (raycaster, renderer, game loop, player, weapons, enemies, HUD, cycle store) is wired together end-to-end
- The mod is genuinely playable in the browser — Path 3 is validated as the right call

The next plan, when this one is done: `2026-XX-XX-loom-phase-2.md` covering Cycle 1 maps + remaining Cycle 1 weapons (Drum Stick, Spam Filter, Industrial Shredder) + remaining Cycle 1 enemies (Account Executive, Recruiter, HR Manager boss) + cycle-1 → cycle-2 transition.
