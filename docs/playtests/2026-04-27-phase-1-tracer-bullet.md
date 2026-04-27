# Phase 1 Tracer Bullet — Playtest Log

**Date:** 2026-04-27
**l0b0tonline commit:** aa772fe
**LOOM-DOOM plan commit:** 0d333b0
**Test file:** `l0b0tonline/e2e/loom-phase-1.spec.ts`
**Test command:** `npm run test:e2e -- e2e/loom-phase-1.spec.ts`

## Result

[X] PASS / [ ] FAIL

## Behavior verification

| # | Behavior | Verified by | Result |
|---|---|---|---|
| 1 | LOOM window opens from J0IN 0S desktop | Playwright | PASS |
| 2 | Boot sequence plays ("LOOM TECHNOLOGIES" → "ACCESS_GRANTED") | Playwright | PASS |
| 3 | Map renders (raycaster paints walls) | Manual | NEEDS HUMAN |
| 4 | Pointer lock works on canvas click | Playwright (best-effort) | N/A — headless Playwright does not honour requestPointerLock; game loop continues ticking after click (verified) |
| 5 | WASD movement works | Manual | NEEDS HUMAN |
| 6 | Mouse-look works | Manual | NEEDS HUMAN |
| 7 | Branded Pen fires (queue ammo decrements on click) | Manual / Playwright probe | NEEDS HUMAN — canvas-click in headless fires the input handler but pointer lock is not active so fire may not register; manual sanity check required |
| 8 | Branded Pen damages Interns (3 hits at 10 dmg = 30 dmg, 20 HP enemy → kill in 2 hits) | Manual | NEEDS HUMAN |
| 9 | Neural Pulse fires (F key consumes pulseCharge) | Manual | NEEDS HUMAN — keyboard injection to locked canvas is unreliable headless; manual sanity check required |
| 10 | Neural Pulse damages Interns (4 hits at 5 dmg = 20 dmg → kill in 4 hits) | Manual | NEEDS HUMAN |
| 11 | Interns chase player when within 8 tiles | Manual | NEEDS HUMAN |
| 12 | Interns die / disappear when HP reaches 0 | Manual | NEEDS HUMAN |
| 13 | LoomHud overlay renders bottom-left | Playwright | PASS |
| 14 | HUD shows correct starting state (bio: 100%, queue: 50, branded_pen.exe, cycle 8492, simulation nominal, surveillance: ON) | Playwright | PASS |
| 15 | Window closeable cleanly (no console errors) | Playwright | PASS |

## Automated coverage

Items marked "Playwright" are verified by `e2e/loom-phase-1.spec.ts`. Specifically:

- **LOOM window opens** — `__STORE__.getState().windows.openWindow({ type: 'GAME_LOOM', ... })` injection; title "loom.exe" appears.
- **Boot sequence** — `getByText('LOOM TECHNOLOGIES')` and `getByText('ACCESS_GRANTED')` both visible within 8s.
- **Canvas presence** — `canvas` locator is visible after boot.
- **WebGL2 context** — `canvas.getContext('webgl2') !== null` evaluated in-page returns `true`.
- **GameLoop is ticking** — `window.__LOOM_GAME__.getSnapshot().frameCount` sampled twice 600ms apart; second value is strictly greater than first (typically ~36 frames apart at 60 fps).
- **Game loop survives canvas click** — `frameCount` still increasing after a `canvas.click({ force: true })`.
- **HUD DOM text presence** — All six HUD text fragments verified: `[unit#88-E]`, `bio:`, `100%`, `queue:`, `branded_pen.exe`, `cycle 8492`, `simulation nominal`, `surveillance: ON`.
- **No console errors** — filtered for known-benign browser noise (chrome-extension, ERR_FAILED, favicon).

### Test-only affordances added in this commit

Two small additions were made to production source files purely to enable e2e probing:

1. **`GameSnapshot.frameCount`** — A monotonically-incrementing counter incremented each `requestAnimationFrame` tick. Included in `getSnapshot()` return value. Not used by the HUD or any user-facing code.
2. **`window.__LOOM_GAME__`** — Set to `loopRef.current` in `LOOMGame.tsx` immediately after `loopRef.current.start()`, and deleted in the cleanup return. Both are no-ops in production (an extra global ref that nothing reads).

## Items requiring human eyes

Several Phase 1 behaviors involve canvas-pixel rendering or input-driven movement that headless Playwright cannot observe reliably:

- Walls actually rendered as a 3D corridor (not just "canvas exists and has WebGL2")
- Player movement responding to WASD keyboard input
- Mouse-look turning the camera
- Sprites visible as the 3 Interns
- Intern AI (walking toward player when within 8 tiles)
- Damage feedback (sprites taking damage / disappearing when killed)
- Branded Pen ammo decrement visible in HUD on mouse click
- Neural Pulse pulseCharge decrement on F key

To verify these, a human should:

1. `cd /Users/justinwest/Repos/l0b0tonline && npm run dev`
2. Open http://localhost:3001 (or whatever port Vite picks)
3. Boot J0IN 0S, run loom.exe
4. After the boot sequence: click the canvas to lock pointer, walk around with WASD, look around with mouse.
5. Aim at a visible Intern and click to fire the Branded Pen — confirm queue ammo decrements in the HUD.
6. Press F to fire the Neural Pulse — confirm pulseCharge drops and the Intern takes damage.
7. Kill an Intern (2 Pen shots or 4 Neural Pulse hits) — confirm it disappears.
8. Close the window cleanly — confirm no errors in the browser console.

## Notes / observations

- Pointer lock is explicitly unsupported in headless Playwright (Chromium rejects `requestPointerLock` without a real display). The test clicks the canvas with `{ force: true }` and verifies only that the game loop keeps running post-click — not that pointer lock was granted. Row 4 is marked N/A rather than FAIL for this reason.
- WebGL2 is available in Playwright's headless Chromium. The renderer initialised and the context probe returned `true`.
- Test runtime: 21.6s (includes full J0IN OS boot + Playwright browser startup + 8s boot wait + 600ms liveness sample + 200ms post-click sample).
- `cycle 8492` is derived from `8491 + cycle` where the initial `cycle` from `cycleStore` is `1`. This is stable as long as `initialState.cycle` remains `1`.

## Phase 1 Definition of Done

[X] Automated DoD items pass (HUD, GameLoop ticking, WebGL2, no crashes).
[ ] Manual DoD items pass (visual rendering, input, AI, combat) — pending user sanity check.
