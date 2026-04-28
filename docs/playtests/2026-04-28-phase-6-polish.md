# Phase 6 Polish + Accessibility + Code-Side Audio — Playtest Log

**Date:** 2026-04-28
**l0b0tonline branch:** `loom-game`
**l0b0tonline final commit:** `188a413` (Phase 6 review cleanup)
**LOOM-DOOM plan commit:** `0b40a8d`
**Test file:** `l0b0tonline/e2e/loom-phase-6.spec.ts`
**Test command:** `npm run test:e2e -- e2e/loom-phase-6.spec.ts`

## Result

[X] PASS (automation gate) — manual creative gate items pending

## Phase 6 Definition of Done (per design spec §8)

- [X] **Automation gate** — Phase 6 e2e walks SkillLevelSelect → 18 maps → 4 boss kills → The Hand → C-reveal cinematic → boot return, asserts Termination Letter pickup HUD + cycle 4 transition + reduced-motion mock; ~46s
- [ ] **Manual creative gate** — "an unfamiliar tester can play start to finish without a placeholder asset or jarring difficulty spike"

## Behavior verification

| # | Behavior | Verified by | Result |
|---|---|---|---|
| 1 | All 18 maps + 4 boss kills + The Hand + C-reveal still work (Phase 5 baseline preserved) | Playwright | PASS |
| 2 | SkillLevelSelect renders pre-boot on first LOOMGame mount | Playwright | PASS |
| 3 | Selecting NORMAL → BootSequence → cyc1_lobby starts at 100 HP / 50 ammo (Phase 5 baseline) | Playwright | PASS |
| 4 | Termination Letter pickup at cyc2_glass_confroom_boss (4, 6) → HUD shows `tl: 1` | Playwright | PASS |
| 5 | T-key hint appears in controls panel when player has Termination Letters | Static (LoomHud.tsx review fix I-1) | PASS |
| 6 | matchMedia mock for `prefers-reduced-motion` respected (accessibilityStore.reducedMotion picks up MQ) | Playwright | PASS |
| 7 | C-reveal cinematic NULL_ENTITY title-flip + OK/ACCEPT credits visible after The Hand kill | Playwright | PASS |
| 8 | After cinematic completes, boot returns; cycle reset to 1 | Playwright (HUD `cycle 8492 // nominal`) | PASS |
| 9 | L-907 unlock fires on cycle-end-final completion (unlockFile('loom_l907') in LOOMGame.onComplete) | Static (code review verified) | PASS |
| 10 | LoomInterface filters L-907 out of displayLogs when not unlocked | Static (code review verified) | PASS |
| 11 | Player.applyDebuff uses max-expiry semantics (reapplying shorter no longer shortens active) | Vitest | PASS |
| 12 | TheHand phase-block explicit returns prevent fall-through (defensive) | Vitest | PASS |
| 13 | HandMenu kind never repeats consecutively (anti-repeat over 20 iterations) | Vitest | PASS |
| 14 | SeniorVP + TheAlgorithm unit tests cover sight/attack/cooldown/death | Vitest | PASS |
| 15 | accessibilityStore persists to localStorage (`loom-accessibility` key) | Vitest | PASS |
| 16 | LoomHud gates connection blip / health lie / ammo binary / char break / line fades on reducedMotion | Vitest + manual | PASS (vitest), NEEDS HUMAN (manual visual) |
| 17 | CRevealCinematic skips window-glitch stage + cross-fades title in reducedMotion mode | Vitest | PASS |
| 18 | Skill-level multipliers: Easy 150 HP/×0.7 dmg, Normal 100 HP/×1.0, Hard 75 HP/×1.4 | Vitest | PASS |
| 19 | cycleStore.reset() preserves skillLevel (session-level preference) | Vitest | PASS |
| 20 | Termination Letter 'T' triggers 6-tile-radius room-clear (999 dmg, bosses immune) | Static + e2e pickup | PASS |
| 21 | The Hum hold-fire: holding mouse drains HP every frame from enemies on aim ray | Vitest | PASS |
| 22 | The Algorithm predictive aim: leads moving players by velocity × 0.3 sec | Vitest (smoke) | PASS |
| 23 | 60Hz hum intensity ramps per cycle: 0.4 / 0.6 / 0.8 / 1.0 → soundService gain 0.2 / 0.3 / 0.4 / 0.5 | Vitest (audio mock) | PASS |
| 24 | Q/E keyboard look rotates player.angle when accessibilityStore.keyboardLook=true | Vitest | PASS |
| 25 | SkillLevelSelect has 3 accessibility checkboxes (reduced-motion + keyboard-look) | Vitest | PASS |
| 26 | No console errors throughout | Playwright | PASS |
| 27 | **Termination Letter feels like a meaningful "oh shit" room-clear, not a trivial cheat** | Manual | **NEEDS HUMAN — creative gate** |
| 28 | **Skill-level differences are observable but not punishing on Easy / not bullet-spongy on Hard** | Manual | **NEEDS HUMAN — creative gate** |
| 29 | **60Hz hum ramp builds dread without being uncomfortable** | Manual | **NEEDS HUMAN — creative gate** |
| 30 | **Reduced-motion mode preserves the campaign's emotional payoff (cinematic still lands)** | Manual | **NEEDS HUMAN — creative gate** |
| 31 | The Hum hold-fire is mechanically distinct from clicking — sustained beam feels "song-as-weapon" | Manual | NEEDS HUMAN |
| 32 | The Algorithm's predictive aim makes strafing matter; stationary players still get hit | Manual | NEEDS HUMAN |
| 33 | Q/E keyboard look is responsive enough for combat (turn speed 2.5 rad/sec) | Manual | NEEDS HUMAN |
| 34 | Pressing T with no Termination Letters is a no-op (no audio cue, no UI flash) | Manual | NEEDS HUMAN |
| 35 | After cinematic completes, opening LOOM Surveillance app shows L-907 entry (was hidden before) | Manual | NEEDS HUMAN |
| 36 | The 19-second cinematic with reduced-motion still hits the emotional beats (no jarring abrupt transitions) | Manual | NEEDS HUMAN |

## How to verify the manual items

```bash
cd /Users/justinwest/Repos/l0b0tonline
npm run dev
```

Open the dev URL, boot J0IN 0S, run `loom.exe`. SkillLevelSelect appears first — pick Normal (or Easy / Hard to verify multipliers feel right). Play through Cycles 1-4 (~30-45 minutes), engaging:

- The 3 Termination Letter pickups (cyc2_glass_confroom_boss, cyc3_wellness_program, cyc4_void_2)
- The 60Hz hum ramp listening cue across cycles (subtle in C1, dominant in C4)
- The Hum hold-fire on cyc4_hand (after the pickup)
- The cursor takeover phase 3 of The Hand (with reduced-motion off — try with reduced-motion on too to verify the cinematic still lands)

After defeating The Hand and watching the cinematic, return to J0IN 0S desktop. Open LOOM Surveillance app — verify L-907 (SUBJECT_DOOM / NULL_ENTITY) entry now appears alongside L-901..L-906.

To test accessibility:
1. Pre-boot SkillLevelSelect screen has 2 checkboxes (reduced motion + keyboard look). Toggle them, then start a run.
2. Reduced-motion: HUD glitch effects should be subdued (no yellow flashes, no char break, no line fades). Cinematic should skip the window-glitch stage and cross-fade the title instead of letter-by-letter.
3. Keyboard look: Q rotates left, E rotates right. Mouse-look continues to work additively.

## Phase 6 commit summary (11 commits on `loom-game`)

| Task / Item | SHA | Description |
|---|---|---|
| 6.1 | `3a67a39` | TheHand M-3 explicit returns + M-5 HandMenu anti-repeat |
| 6.2 | `bfee35d` | SeniorVP + TheAlgorithm unit tests (M-6 coverage gap) |
| 6.3 | `cdcf5f6` | Player.applyDebuff max-expiry semantics (M-1) |
| 6.4 | `1287812` | accessibilityStore + reduced-motion gates |
| 6.5 | `fb613ae` | Q/E keyboard look + L-907 unlock pipeline |
| 6.6 | `78b31a7` | Easy/Normal/Hard skill-level selector |
| 6.7 | `e96a2dc` | Termination Letter consumable |
| 6.8 | `01e76e8` | Hum hold-fire + Algorithm predictive aim |
| 6.9 | `12b6a2a` | 60Hz hum intensity ramp by cycle |
| 6.10 | `576afca` | Phase 6 e2e Playwright spec (replaces Phase 5) |
| Final review fixes | `188a413` | T-hint + accessibility checkboxes + audio nit + test cleanup |

Total: 11 commits since the merge of PR #164 (`7da1209`).

## Test status

- TSC clean
- **Vitest 971/971** across 53 test files
- **Playwright Phase 6 e2e** — PASS in ~46s (full 18-map walkthrough + 4 boss kills + The Hand + C-reveal + Termination Letter pickup + SkillLevelSelect interaction + matchMedia mock + console-error gate)

## Phase 6 review carry-overs (deferred to Phase 7 prelude)

Per the Phase 6 final code review (Critical: 0, Important: 4 — 2 fixed, 2 deferred; Minor: 5 — 2 fixed, 3 deferred):

**Important issues fixed in this PR (commit `188a413`):**
- I-1: T-key hint added to controls panel when player has Termination Letters (was invisible mechanic before)
- I-2: Accessibility settings UI shipped (2 checkboxes in SkillLevelSelect) — closes the chicken-and-egg accessibility regression where the keyboardLook target audience couldn't reach the toggle

**Important issues deferred to Phase 7 prelude:**
- I-3: GameLoop hold-fire dispatch + TL room-clear unit tests. Currently exercised only e2e/integration. Phase 7 prelude should add `gameLoop.test.ts` covering the new dispatch logic.
- I-4: E2E L-907 unlock pipeline assertion. The cinematic completes and the unlock fires (verified statically), but the e2e doesn't open LOOM Surveillance app post-cinematic to verify L-907 actually appears. Phase 7 prelude should add this assertion.

**Minor issues deferred to Phase 7 prelude:**
- N-3: L-907 paraphrase in `LoomHud.tsx` SURVEILLANCE_FRAGMENTS rotation appears in corruption modes 1-3 regardless of unlock state. May be intentional foreshadowing — verify with user before either gating or leaving as-is.
- N-4: accessibilityStore tests don't verify persist middleware roundtrip (set → reload → rehydrate). Low priority; cycleStore tests skip this too.
- N-5: soundService.setAmbientGain always glides via setTargetAtTime — no immediate-mode for mute scenarios. Phase 7 audio mix balance work will replace with proper per-track automation.

## Phase 6 plan-documented Phase 7 deferrals (still pending)

These were explicitly marked OUT OF SCOPE for Phase 6 (kept off the critical path):

- **Real *Patch Tuesday (Heaven Vers!0n)* and *human In l00p* music tracks** — requires Suno composition + WAV import to `services/soundService.ts`. Phase 7 ship task.
- **Real sprite art** replacing colored rectangles — requires AI sprite generation pipeline. Spec §8 risk #4: "the biggest single time sink." Phase 7.
- **Surveillance log voiceover** for C2/C3 — requires audio recording / AI-TTS. Phase 7.
- **Mobile responsiveness investigation** — requires device-specific testing. Phase 7 stretch.
- **Final difficulty tuning** based on playtester data — requires real playtester sessions. Phase 7.
- **colorblindPalette toggle UI + palette-aware enemy/projectile color shifts** — Phase 7 UI work (the store field exists; Phase 6 just stubbed it).

## Phase 7 prelude — recommended cleanup batch

Before Phase 7 ship work, address the deferred items above:

1. **N-3** — verify L-907 fragment rotation intent (single-line decision)
2. **I-3** — add `gameLoop.test.ts` covering Hum hold-fire dispatch + TL room-clear (~50 lines)
3. **I-4** — extend Phase 6 e2e with post-cinematic LOOM Surveillance app open + L-907 visibility assertion (~10 lines)

Estimated: ~1 hour total. Land before Phase 7's bigger creative work begins.

## What's still placeholder content

For ship-readiness assessment:
- 4 maps' wall colors are still placeholder texture indices (`["wall_lobby"]` for all 18 maps)
- Enemy sprites are colored rectangles, not real sprites
- Music: Cycle 1+2+3 use 'ambient' (placeholder); Cycle 2+3+4 boss arenas use 'ship_it' (placeholder for Patch Tuesday + human In l00p + Err0r Flesh)
- Manifesto text uses canonical `MANIFESTO_TEXT` from l0b0tonline (works as-is)

Phase 6 ships the *mechanical* polish; Phase 7 ships the *content* polish.
