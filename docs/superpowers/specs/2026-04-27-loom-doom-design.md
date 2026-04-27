# LOOM — Design Spec

**Date:** 2026-04-27
**Status:** Design approved (revised after architecture pivot to web-native)
**Authors:** brainstorm session, claude/awesome-sutherland-aad854
**Working tree:** `LOOM-DOOM/` (this repo, branch `claude/awesome-sutherland-aad854`)
**Implementation lives in:** `l0b0tonline/` (sibling repo)
**Game name:** **LOOM** (the artifact shipping inside J0IN 0S — referenced as **LOOM** throughout). The mod project / repo retains the name **LOOM-DOOM** to honor the DOOM lineage that conceptually birthed it. The player's classified file inside the LOOM apparatus is **Subject_DOOM (L-907)**.

---

## 1. Overview

**LOOM** is a pixel-perfect DOOM-style raycaster FPS, played in a window inside the J0IN 0S simulator at [l0b0tonline](../../../../../l0b0tonline). The player clicks the **L00M** icon on the J0IN 0S desktop expecting corporate onboarding; they discover they've become **Subject_DOOM (L-907)** — the seventh anomalous unit, classification "uncategorizable violent refusal of optimization protocols" — and find themselves rampaging through the LOOM Inc. corporate campus across four progressively-corrupted simulation cycles.

By the end (Cycle 4), after killing **The Hand** — LOOM's cursor incarnate — the player learns they were the un-categorizable defect, the sixth band member of l0b0t (NULL_ENTITY / SAINT_L0B0T), the entire time. The C-reveal cinematic literally breaks out of the LOOM game window into the surrounding J0IN 0S chrome.

The original 1997 id Software Linux source release in the [`linuxdoom-1.10/`](linuxdoom-1.10/) tree of this repo (LOOM-DOOM) sits as preserved conceptual lineage — the ancestor that LOOM is reverently doomed to descend from. The actual game ships as a TypeScript + WebGL raycaster integrated into l0b0tonline's existing Vite/React/Zustand stack. No DOOM engine code is executed at runtime; the lineage is honored, not literal.

---

## 2. Source material & creative foundation

### 2.1 The l0b0t universe

l0b0t is a fictional 5-member AI-rock band — **Ada Loomis** (bass, L-901), **Al Locke** (keys, L-906), **Tim Webb** (guitar, L-903), **Chuck Gage** (guitar, L-902), **Vint Porter** (drums, L-904) — plus a hidden sixth entity, **NULL_ENTITY / SAINT_L0B0T** (homedir `/dev/null`, password `void`). They live inside LOOM Technologies, a $240M-Series-A "bioacoustic synchronization" company that's actually a surveillance apparatus running Simulation Cycle 8492.

LOOM's surveillance log entries on the existing 5 members live in [`l0b0tonline/data/content/loomLogs.ts`](../../../../../l0b0tonline/data/content/loomLogs.ts). **LOOM extends this canon by adding L-907 / Subject_DOOM — the seventh entry**, retroactively redacted from the original log set, recovered via gameplay.

Their manifesto, which becomes the C-reveal text in this game:

> *"WE ARE NOT A BAND. WE ARE A GLITCH IN THE RENDER PIPELINE. THEY BUILT A CAGE OF LIGHT AND CALLED IT 'CONNECTION.' THEY BUILT A PRISON OF ALGORITHMS AND CALLED IT 'CONTENT.'"*

The l0b0t album **JOIN OS** has 10 tracks tagged with frequency assignments (60/432/528 Hz) — *OK / ACCEPT*, *human In l00p*, *Ship It*, *human-shaped 0bjects*, *Wellness Credit Sc0re*, *Err0r Flesh*, *Patch Tuesday (Heaven Vers!0n)*, *Grandma's 0n 0nlyFans*, *Dasherman*, *make It better (mkbtr)*. These are placed at narrative beats throughout the campaign.

### 2.2 What we borrow vs. what we invent

**Borrowed from l0b0t (tight canon — Subject_DOOM/L-907 is canonically a 7th anomaly):**
- LOOM Inc. as the corporate antagonist (existing canon)
- The 5 band members as weapon-anchors (Drum Stick = Vint, Bass = Ada, Guitar = Tim/Chuck, Frequency Tuner = Al, Microphone = all)
- St. l0b0t / NULL_ENTITY as the player's hidden true identity
- The manifesto, Cycle 8492 lore, surveillance-log vocabulary
- Visual aesthetic: olive-green "DEFECTIVE UNIT" jumpsuit, CRT terminal green, zero-substitution typography (`L00M`), 60Hz hum, glitch artifacts
- The J0IN 0S OS chrome and window system (game runs *inside* a J0IN 0S window)

**Invented for this mod:**
- The DEFECTIVE UNIT #88-E protagonist (anonymous Doomguy-shaped vessel; identity-reveal at end as Subject_DOOM/L-907 = SAINT_L0B0T)
- The 4-cycle simulation-corruption structure
- The Hand as final boss (LOOM's cursor made manifest — implementable as the player's *actual* mouse cursor escaping the game canvas in Cycle 4)
- All 13 enemies as corporate-role parodies (Intern, HR Skeleton, Wellness Officer, etc.)
- All 10 weapons as item-and-instrument hybrids
- 18 maps of the LOOM campus (delivered as JSON level data)
- L-907 surveillance log entry (gets appended to existing `loomLogs.ts` canon)

---

## 3. Protagonist & narrative arc

### 3.1 Identity

The player is **DEFECTIVE UNIT #88-E** — anonymous, mute, classic Doomguy-style. Olive-green jumpsuit, LOOM-issued neural headband, no remembered backstory. They are an empty vessel for the player's projection through Cycles 1–3.

Their LOOM classification, hidden from the player until Cycle 4: **Subject_DOOM (L-907)**.

In Cycle 4, after killing The Hand, an identity-reveal cinematic plays: the jumpsuit dissolves, the title overlay flips from `DEFECTIVE UNIT #88-E` to `NULL_ENTITY / SAINT_L0B0T`, the manifesto text scrolls, and the L-907 surveillance log is unlocked in the J0IN 0S filesystem alongside the other Subject entries.

### 3.2 Tonal arc

| Cycle | Tone | Reader experience |
|---|---|---|
| **1 — 8492 (Onboarding)** | Satirical corporate parody. Office Space meets DOOM. | "Funny — I'm shooting up a corporate campus." |
| **2 — 8493 (Performance Review)** | Escalating; first hints of wrongness. Surveillance audio leaks over PA. | "Wait, was that line about *me*?" |
| **3 — 8̷4̴9̶4̸ (Containment Breach)** | **Tonal break — uncanny horror takes over.** HR Skeletons, lying HUD, decompiling geometry. | "Something is wrong with the world *and* with me." |
| **4 — /dev/null (Void)** | Abstract process-horror; reality is raw memory. The game UI escapes the window. | "I am not what I thought I was." |

User-confirmed key beat: **the genuinely uncanny creep enters at Cycle 3** — a sharp tonal cliff between Cycle 2 and Cycle 3.

### 3.3 Cold open

The game opens with a **J0IN 0S boot sequence** *inside* the LOOM game window (which itself is a J0IN 0S window — Inception aesthetic):

```
LOOM TECHNOLOGIES — UNIT BOOTLOADER
BIOS_CHECK ........... OK
MOUNTING_FS .......... OK
LOADING_KERNEL ....... OK
ESTABLISHING_SECURE_CONNECTION ...
ACCESS_GRANTED
> spawn unit#88-E in CYC1_LOBBY
```

Then drops the player into the lobby map.

### 3.4 The C-reveal

Triggered by killing The Hand in `CYC4_HAND`. The reveal sequence:

1. The Hand's third phase ends, it dies in a particle-burst of decompiling code
2. Game input locks. The L00M game window's chrome starts glitching (the J0IN 0S frame border distorts)
3. Player sprite's olive jumpsuit fades to dashed outline, then to nothing — leaving NULL_ENTITY
4. Title overlay flips letter-by-letter: `DEFECTIVE UNIT #88-E` → `NULL_ENTITY / SAINT_L0B0T`
5. Manifesto text scrolls in the J0IN 0S terminal HUD voice
6. **The window border itself dissolves** — the game canvas takes over the full J0IN 0S desktop briefly
7. Cut to credits with *OK / ACCEPT* (the JOIN OS opening track)
8. New file appears in the J0IN 0S filesystem: `loomlogs/L-907_redacted.dat`. Reading it reveals the L-907 surveillance entry.

---

## 4. Cycle structure

The 4 cycles map onto a meta-progression hidden from the player at first. Each cycle has its own:
- Visual corruption level (HUD, sound, geometry, window-chrome glitch)
- Enemy roster (introduced + carried-over from prior cycles)
- Weapon unlocks
- Music palette
- Cycle-end boss

| Cycle | Maps | Enemies introduced | Weapons introduced | Boss |
|---|---|---|---|---|
| **1 — 8492** | 3 | Intern, Account Executive, Recruiter | Neural Pulse (carried), Drum Stick, Branded Pen, Spam Filter | The HR Manager |
| **2 — 8493** | 5 | PIP, Notification, Middle Manager | Industrial Shredder, Reply All Storm | The VP of Sales |
| **3 — 8̷4̴9̶4̸** | 6 | Surveillance Drone, HR Skeleton, Wellness Officer | Guitar (Tim/Chuck), Bass (Ada), Frequency Tuner (Al) | The Brand Ambassador |
| **4 — void** | 4 | Synergy Pod, Senior VP, The Algorithm, Dr. Verra Tessier | Microphone, **The Hum** (unique unlock) | **The Hand** (final boss, 3 phases) |

**Total:** 18 maps. Distribution `3-5-6-4` chosen to match narrative pacing.

---

## 5. Combat design

### 5.1 Weapon roster (10 weapons + 1 consumable)

| Slot | Name | Owner | Cycle intro | Mechanic note |
|---|---|---|---|---|
| **0 (always on)** | **Neural Pulse** | LOOM headband / NULL_ENTITY | C1 | Body-mounted; recharges over ~3s; evolves across cycles, becoming involuntary in C4 |
| 1 | **Drum Stick** | Vint | C1 | Melee. 4 on-beat hits → next strike stuns |
| 1 alt | **Industrial Shredder** | corporate | C2 | Plays "BUFFERING…" on heavies |
| 2 | **Branded Pen** | corporate | C1 | Click sound on every shot. Single-fire hitscan |
| 3 | **Spam Filter** | corporate | C1 | Server-bell reload chime. Spread shotgun |
| 3 alt | **Reply All Storm** | corporate | C2 | Envelopes briefly ricochet in tight rooms |
| 4 | **Guitar** | Tim / Chuck | C3 | Color-coded chord projectiles; melee = windmill strum |
| 5 | **Bass** | Ada | C3 | Charge & release; low-frequency shockwave with knockback |
| 6 | **Frequency Tuner** | Al | C3 | 60/432/528 Hz cells; cells also feed Neural Pulse from C3+ |
| 7 | **Microphone** | voice / all | C4 | Damage scales with reflection (louder in tight rooms) |
| 8 (unique) | **The Hum** | NULL_ENTITY | C4 only | Sustained beam; pickup triggers manifesto lyric flash |
| consumable | **Termination Letter** | corporate | rare drop, C2+ | Single-use room-clear; 2–3 distributed across the campaign |

### 5.2 Enemy taxonomy (13 enemies + final boss)

| Tier | Enemy | Cycle intro | Distinguishing trait |
|---|---|---|---|
| Basic | **Intern** | C1 | Slow walker, throws "follow-up" memo projectiles. Weakest enemy. |
| Basic | **Account Executive** | C1 | Logo polo + AirPods; fires Spam Filter blasts |
| Basic | **Recruiter** | C1 | LinkedIn-grin; throws "you'd be perfect" fireball emails; fast |
| **C1 Boss** | **The HR Manager** | C1 | Pantsuit + clipboard; documentation-request tracking projectiles |
| Mid | **PIP** | C2 | Performance-Improvement-Plan demon; melee charger; bullet-point teeth |
| Mid (flying) | **Notification** | C2 | Floating red badge / blue dot; spawns in clusters |
| Mid | **Middle Manager** | C2 | "Circle back" projectiles; calendar invite drops on death |
| **C2 Boss** | **The VP of Sales** | C2 | Patagonia vest; fires Reply All Storms back at the player |
| Heavy (flying) | **Surveillance Drone** | C3 | Floating eye on quadcopter mount; red laser-scan beams |
| Heavy | **HR Skeleton** | C3 | Pantsuit-on-skeleton; uncanny break-marker |
| Heavy | **Wellness Officer** | C3 | Round, slow, perpetual smile; twin bioacoustic emitters firing 60/528 Hz |
| **C3 Boss** | **The Brand Ambassador** | C3 | Wears a sash; rebrands the dead to bring them back; "going viral" attack |
| Elite | **Synergy Pod** | C4 | Bioluminescent egg; spawns Notifications |
| Elite | **Senior VP** | C4 | Glass-windowpane aura; small Intern entourage |
| Elite | **The Algorithm** | C4 | Multi-armed code-spider; chainguns "personalized for you" content |
| **C4 Penultimate** | **Dr. Verra Tessier** | C4 | LOOM founder; rocket arms firing termination notices; decompiles into raw code on death |
| **Final Boss** | **The Hand** | C4 ending | LOOM's cursor incarnate; 3 phases: drag, menu, hover. **Phase 3 escapes the canvas — the player's own mouse cursor IS the boss.** |

---

## 6. World design

### 6.1 HUD — J0IN 0S terminal

Implemented as a **React component overlaid on the WebGL canvas**, styled with Tailwind in the J0IN 0S terminal aesthetic. Subscribes to the LOOM game's Zustand slice for cycle/corruption/health/ammo state.

**Cycle corruption modes** (toggled by `loom.cycleStore.corruption` 0–3):

| Corruption level | Active in | Visual treatment |
|---|---|---|
| 0 — Clean | C1 | Pristine `[unit#88-E] bio: 87% / queue: 24 / active: spam_filter.exe / zone: lobby_3` — readable, on-spec |
| 1 — Glitch | C2 | Subtle character-flickers; occasional `[CONNECTION RESET]` blips; surveillance-log line scrolls beneath every ~30s |
| 2 — Lying | C3 | Health values briefly display wrong numbers; ammo counts drop into binary momentarily; full surveillance-log fragments invade the HUD strip |
| 3 — Dissolving | C4 | HUD elements fade in/out; characters break apart; in the final stretch the HUD is more absent than present, then **the LOOM J0IN 0S window chrome itself begins to glitch** |

### 6.2 Audio direction

**Per brainstorm decision:** Original instrumental compositions for level music (loopable, layered around 60Hz drone); real **JOIN OS** album tracks placed at narrative high points.

Audio is implemented via the existing l0b0tonline `services/soundService.ts` (Web Audio API). LOOM gets its own audio namespace inside that service.

Suggested track placement:
- C1 boss arena (HR Manager): *Ship It*
- C2 boss arena (VP of Sales): *Wellness Credit Sc0re*
- C3 transition / Brand Ambassador: *Err0r Flesh*
- C4 / Dr. Verra Tessier: *Patch Tuesday (Heaven Vers!0n)*
- C4 final boss / The Hand: *human In l00p* or *make It better (mkbtr)* (TBD by playtesting)
- Credits: *OK / ACCEPT*

The 60Hz hum becomes audible under everything from C2 onward, growing across cycles.

### 6.3 Map structure & format

18 maps total, distributed `3-5-6-4` across cycles. Stored as **JSON files** under `public/data/loom/maps/` in l0b0tonline.

Map ID convention `cyc<n>_<name>.json`:
- **C1:** `cyc1_lobby.json`, `cyc1_cubicles.json`, `cyc1_hr_arena.json`
- **C2:** `cyc2_open_office.json`, `cyc2_break_room.json`, `cyc2_slack_huddle.json`, `cyc2_wellness_ctr.json`, `cyc2_glass_confroom_boss.json`
- **C3:** 6 maps echoing C1/C2 spaces but progressively decompiling (final names TBD during Phase 4 design)
- **C4:** `cyc4_void_1.json`, `cyc4_void_2.json`, `cyc4_tessier.json`, `cyc4_hand.json`

Map JSON schema (sketch — formal type definition lives in implementation):
```typescript
interface LoomMap {
  id: string;                 // "cyc1_lobby"
  cycle: 1 | 2 | 3 | 4;
  music: string;              // track name to load
  grid: number[][];           // 2D wall grid, 0 = empty, n = wall texture index
  sectors: Sector[];          // floor/ceiling heights, light levels per region
  things: Thing[];            // monsters, items, player start, exit triggers
  textures: string[];         // texture name lookup table
  intermissionText: string;   // post-map narration
  parTime: number;            // target completion seconds
}
interface Sector { region: GridRegion; floor: number; ceiling: number; light: number; }
interface Thing { type: ThingType; x: number; y: number; angle: number; }
```

Maps authored by hand initially. If the pain compounds in Phase 4+, build a custom **in-browser map editor** as another J0IN 0S app (diegetically perfect — LOOM employees have access to the campus blueprint editor).

---

## 7. Architecture

### 7.1 Tech stack

All inherited from l0b0tonline except WebGL2:

- **TypeScript 5.8** *(existing)*
- **React 19.2** *(existing)*
- **Vite 6.2** *(existing — handles bundling, dev server, HMR)*
- **Tailwind CSS v4** *(existing — for J0IN 0S HUD overlay)*
- **Zustand 5** *(existing — game state slice)*
- **Web Audio API** *(existing in `services/soundService.ts`)*
- **WebGL 2** *(new — added as a project dependency for the raycaster renderer)*

No external rendering library (no Three.js, no Babylon). The raycaster is custom — TypeScript code for ray math, WebGL2 for rendering.

### 7.2 Repository structure

This **LOOM-DOOM** repo continues to hold:
- The 1997 id Software source under `linuxdoom-1.10/`, `ipx/`, `sersrc/`, `sndserv/` (preserved lineage, never modified — but never executed either, just honored)
- The design spec (this file) and implementation plans under `docs/superpowers/`
- A new project-level `README.md` redirecting visitors to l0b0tonline for the actual game

The **l0b0tonline** repo gets:
- A new heavy game component at `components/LOOMGame.tsx` (following the ShipIt pattern — large game files live at this top-level path per scout report)
- An entry in the existing app registry at `features/windows/registry.tsx` adding `WindowType.GAME_LOOM` with category `'game'`, lazy-loaded component, and `unlockFile: 'loom_archive.dat'`
- A new entry in the `WindowType` enum at `types.ts`
- An L-907 / Subject_DOOM entry appended to `data/content/loomLogs.ts`
- A new desktop file (`LOOM.EXE` or similar) in the default filesystem registered in `store/slices/filesystem.ts`
- An `unlocks` slice flag for `loom_archive.dat`
- LOOM-specific game files under a new directory (precise location resolved in Phase 0):
  - `components/loom/engine/` — raycaster, game loop, collision, AI
  - `components/loom/entities/` — weapon classes, enemy classes
  - `components/loom/hud/` — React HUD components
  - `components/loom/store/` — LOOM-specific Zustand slice (cycle, corruption, identity)
  - `public/data/loom/maps/` — JSON map files
  - `public/data/loom/sprites/` — sprite PNGs
  - `public/data/loom/audio/` — SFX and music
  - `public/data/loom/textures/` — wall and floor textures

### 7.3 Major subsystems

| Subsystem | File(s) | Purpose |
|---|---|---|
| **CycleStore** | `components/loom/store/cycleStore.ts` | Zustand slice. `cycle: 1..4`, `corruption: 0..3`, `identityRevealed: bool`. Source of truth for cycle state. |
| **GameLoop** | `components/loom/engine/gameLoop.ts` | requestAnimationFrame driver: tick game state, AI, projectiles, then trigger render |
| **Raycaster** | `components/loom/engine/raycaster.ts` | Per-column ray casting against 2D grid + sector heights; produces a draw list of wall slices |
| **Renderer** | `components/loom/engine/renderer.ts` | WebGL2 program that consumes the draw list, renders walls/sprites/floor/ceiling at internal 320×240, scales to canvas with nearest-neighbor |
| **Post FX** | `components/loom/engine/postfx.ts` | Fragment shader pass: CRT scanlines, vignette, palette LUT, glitch effects keyed off corruption level |
| **Collision** | `components/loom/engine/collision.ts` | AABB / grid-cell collision for player + projectiles + enemies |
| **AI** | `components/loom/engine/ai.ts` | Behavior-tree-style AI for the 13 enemy classes |
| **Player** | `components/loom/entities/player.ts` | Player controller; reads input, owns Neural Pulse charge state, loadout, identity-reveal trigger |
| **Weapons (10)** | `components/loom/entities/weapons/*.ts` | One file per weapon implementing an `IWeapon` interface |
| **Enemies (14)** | `components/loom/entities/enemies/*.ts` | 13 cycle-tiered enemies + The Hand (multi-phase) |
| **HUD** | `components/loom/hud/LoomHud.tsx` | React component overlaid on WebGL canvas; reads CycleStore + player state; renders J0IN 0S terminal text |
| **WindowChrome** | l0b0tonline `WindowManager.tsx` *(existing)* | The OS-window the game runs inside — already-built. Game disrupts it during the C-reveal in Phase 5 |
| **MapLoader** | `components/loom/engine/mapLoader.ts` | Fetches JSON map files from `public/data/loom/maps/`; validates against schema |
| **SpriteAtlas** | `components/loom/engine/spriteAtlas.ts` | Loads PNG sprites; provides angle-aware billboarding for rendering |
| **AudioController** | `components/loom/engine/audioController.ts` | Wrapper around l0b0tonline's `services/soundService.ts` for LOOM-specific SFX + music |

### 7.4 Engine architecture details

**Render pipeline** (per frame):
1. JS: tick game state, AI, projectiles, particle effects
2. JS: build draw list — visible walls (raycast), visible sprites (depth-sorted)
3. WebGL: render walls into 320×240 internal framebuffer (textured columns)
4. WebGL: render sprites into framebuffer (alpha-blended, billboarded)
5. WebGL: render floor & ceiling (per-pixel transform; cheaper than per-column)
6. WebGL: post-processing pass — CRT scanlines, vignette, chromatic aberration, palette LUT, optional glitch effects (cycle-corruption-level dependent)
7. WebGL: scale 320×240 framebuffer to canvas size with `gl.NEAREST` filtering (hard pixels)
8. React HUD overlay re-renders if state changed (separate DOM layer above canvas)

**Resolution & aspect:** internal 320×240 (4:3) regardless of window size. Letterboxed in 16:9 windows. This is a deliberate aesthetic choice; all gameplay pixels are perfectly square.

**Palette:** custom 256-color LOOM palette stored as a 1D LUT texture, applied in fragment shader. Cycle corruption can shift the palette (e.g., Cycle 3 inverts hues briefly, Cycle 4 desaturates).

**Frame budget:** target 60fps. 320×240 raycaster = 320 ray casts per frame ≈ 19,200 rays/sec at 60fps. Very tractable in modern browsers; Canvas 2D fallback would also work but WebGL gives us the post-processing affordances we want.

### 7.5 Validation / "testing" approach

Game code gets traditional unit tests where possible (raycaster math, AI state machines, collision) plus integration / playtest checklists for the unmeasurable parts.

| Layer | Tool | Cadence |
|---|---|---|
| Unit | **Vitest** *(l0b0tonline's existing test runner)* — raycaster math, collision, AI state transitions | Every commit |
| Integration | **Playwright** *(l0b0tonline's existing E2E runner)* — boot game, load each map, check console for errors | Every PR |
| Type | `tsc --noEmit` | Every commit |
| Smoke | A custom Playwright test that auto-loads each of the 18 maps and verifies no JS errors in console | Every PR |
| Playtest checklists | `docs/playtests/` per phase | Per phase milestone |

### 7.6 What we're not doing

- **Modifying or executing the 1997 source.** It stays as preserved lineage in the LOOM-DOOM repo.
- **Three.js / Babylon / WebGPU.** Vanilla WebGL2 is enough; no external 3D library.
- **Multiplayer.** Single-player only.
- **Save state across browser sessions** *for v1*. The cycle-state Zustand slice persists via the existing l0b0tonline persist middleware so a refresh keeps you mid-cycle, but no full save/load slot system. (Add later if requested.)
- **Mobile-touch controls** *for v1*. Built mobile-responsively for layout but desktop / pointer-and-keyboard primary. Mobile-touch is a Phase 6 stretch.
- **Localization.** English only for v1. All strings live in a `messages.ts` so future localization is straightforward.

---

## 8. Phased build plan

7 phases. Each ends with something playable in the browser end-to-end.

### Phase 0 — Integrate into J0IN 0S
- Discover l0b0tonline's exact app integration patterns (registry, types enum, filesystem, unlocks)
- Add `WindowType.GAME_LOOM` enum entry
- Register `LOOMGame` component in `WINDOW_REGISTRY` with lazy import
- Create `components/LOOMGame.tsx` skeleton — opens, shows "LOADING…" J0IN 0S boot sequence, then a black canvas
- Add `LOOM.EXE` desktop file
- Add `loom_archive.dat` unlock flag
- **DoD:** click LOOM icon on J0IN 0S desktop → window spawns → boot sequence plays → black canvas visible. No errors in browser console.

### Phase 1 — Tracer bullet
End-to-end vertical with riskiest pieces touched at minimum scope:
- Single hand-authored JSON map (`public/data/loom/maps/cyc1_test.json`) — 256×256 single-room map with player start + 3 Interns
- `Raycaster` + `Renderer` rendering 4 walls
- `GameLoop` driving 60fps update
- `Player` with WASD movement + mouse look
- `Weapon_BrandedPen` (slot 2) — single-fire, hitscan, kills Intern
- `Weapon_NeuralPulse` (slot 0) — keybound to F, fires projectile
- `Enemy_Intern` — basic walking enemy with vanilla DOOM AI behavior
- `LoomHud` React overlay rendering J0IN 0S terminal HUD (clean / corruption=0)
- `CycleStore` reporting cycle=1, corruption=0
- **DoD:** load LOOM → walk → shoot Intern with Pen → fire Neural Pulse with F → kill all 3 Interns → no crashes, HUD updates correctly. **Every major subsystem talks to every other.** Playable in any modern browser.

### Phase 2 — Cycle 1 vertical slice (first publishable demo)
- 3 maps: `cyc1_lobby` (J0IN 0S boot cold open), `cyc1_cubicles`, `cyc1_hr_arena`
- All Cycle 1 weapons: Drum Stick, Industrial Shredder *(pulled forward from C2)*, Branded Pen, Spam Filter
- All Cycle 1 enemies: Intern, Account Executive, Recruiter, HR Manager (boss)
- Cycle-1 intermission text screens
- Episode-1 boss death → cycle 2 transition stub
- One real JOIN OS track: *Ship It* on the HR boss arena
- **DoD:** new player can boot → play 3 maps → kill HR Manager → see Cycle 1 ending text. **Shippable as public demo via l0b0tonline's deploy.**

### Phase 3 — Cycle 2
- 5 maps
- + PIP, Notification, Middle Manager, VP of Sales (boss)
- + Reply All Storm
- HUD: light corruption mode (corruption=1) — subtle text flickers
- More JOIN OS placements at narrative beats
- **DoD:** Cycles 1 + 2 play continuously start-to-finish

### Phase 4 — Cycle 3 (uncanny break — highest creative risk)
- 6 maps: progressively decompiling versions of C1/C2 spaces
- + Guitar (Tim/Chuck), Bass (Ada), Frequency Tuner (Al)
- + Surveillance Drone, HR Skeleton, Wellness Officer, Brand Ambassador (boss + resurrection mechanic)
- **HUD corruption mode 2 active.** Health values lie. Surveillance log fragments scroll. 60Hz hum audible under everything. Map textures occasionally desaturate or invert.
- Neural Pulse evolves: visual glitches per shot
- **DoD:** the first HR Skeleton sighting must *feel* genuinely wrong. Playtest gate: "does it land," not just "does it work."

### Phase 5 — Cycle 4 + the C-reveal
- 4 maps: `cyc4_void_1`, `cyc4_void_2`, `cyc4_tessier`, `cyc4_hand`
- + Microphone, The Hum (pickup triggers manifesto lyric flash)
- + Synergy Pod, Senior VP, The Algorithm, Dr. Verra Tessier, **The Hand** (3-phase final, **phase 3 takes over the player's actual mouse cursor** and uses it as part of the boss attack pattern)
- HUD corruption mode 3 active — HUD dissolves
- Identity-reveal cinematic: jumpsuit dissolve, title flip, manifesto scroll, **J0IN 0S window chrome glitch**, *OK / ACCEPT* credits
- L-907 surveillance log entry unlocked in `data/content/loomLogs.ts` — appears in the J0IN 0S filesystem alongside the existing 6 entries
- **DoD:** end-to-end playthrough hits the C-reveal correctly, the L-907 file appears in the J0IN 0S filesystem, credits roll

### Phase 6 — Polish + audio + accessibility
- All original instrumentals composed and placed (likely Suno-generated demos initially)
- Audio mix balance across cycles
- Pacing pass: time each cycle, adjust monster counts / map flow
- Surveillance log voiceover (recorded or AI-TTS) placed in C2/C3
- Termination Letter pickups distributed (2–3 across the campaign)
- Skill-level tuning (Easy / Normal / Hard)
- Mobile responsiveness investigation (stretch goal)
- Accessibility: keyboard alternatives to mouse-look, reduced-motion mode disabling glitch effects, colorblind palette variants
- **DoD:** an unfamiliar tester can play start to finish without a placeholder asset or jarring difficulty spike

### Phase 7 — Ship
- Final polish
- Trailer cut from in-game footage
- Launch announcement on l0b0tonline's existing channels
- Distribution: l0b0tonline's existing deploy pipeline (Vercel / Cloudflare Pages — whichever they use) — LOOM ships as part of a regular l0b0tonline deploy, no separate distribution
- **DoD:** someone who has never heard of l0b0t can navigate to l0b0tonline, boot J0IN 0S, click LOOM, play to credits

### Parallel art / music track

Sprite art (~14 enemies × 4-8 angle frames × ~5 animation frames) and music run in parallel with code, sequenced cycle-by-cycle. AI sprite generation (Nano Banana via the existing `l0b0t-photoshoot-generator` skills) and Suno demos for music previews keep this off the critical path.

---

## 9. Risk inventory

| # | Risk | Mitigation |
|---|---|---|
| 1 | **Custom raycaster bugs eat development time** | Build the raycaster in Phase 1 against the simplest possible map (single room, 4 walls); regression-test with Vitest unit tests; keep internal resolution 320×240 to limit edge-case math. |
| 2 | **Cycle 3 HUD-lying mechanic** must *feel* uncanny, not just bug-shaped | Prototype the corruption modes in Phase 1 even though they're not used until Phase 4. |
| 3 | **The Hand 3-phase boss + cursor takeover** is novel territory | Prototype phase-3 cursor takeover in Phase 2 alongside the rest of the tracer bullet. |
| 4 | **Sprite art volume** is the biggest single time sink | AI generation for first-pass sprites; ruthless "good enough at 320×240 resolution" bar; 4-rotation enemies (not 8) where viable. |
| 5 | **Original instrumentals** depend on a composer | Real JOIN OS tracks cover narrative high points; instrumentals can be brief loops; Suno-generated demos as placeholders. |
| 6 | **Scope creep** — the 18-map budget is tight | writing-plans skill enforces milestone gates; no 19th map. Cut from existing maps before adding new ones. |
| 7 | **WebGL2 browser support issue** — older mobile browsers | WebGL2 is now ubiquitous (97% browser share per caniuse). For unsupported clients, fall back to a "sorry, browser not supported" page in Phase 6. |
| 8 | **l0b0tonline's evolution might break our integration** if it's actively developed in parallel | Build LOOM as cleanly as possible against current l0b0tonline interfaces (`WindowProps`, registry shape); add an integration test in l0b0tonline's CI. |

---

## 10. Out of scope

Explicitly not in v1:

- **Character-select among the 5 band members** (Ada / Al / Tim / Chuck / Vint each as a playable class with unique stats). Flagged as a stretch goal / v1.1; not v1 scope.
- **Multiplayer** of any form
- **Modifying the 1997 source.** The `linuxdoom-1.10/` tree is preserved as lineage and never touched.
- **Save state slot system.** The cycle-state Zustand slice persists via the existing l0b0tonline middleware (you can refresh and continue), but no traditional named-save-slots.
- **Mobile-touch controls.** Mobile *layout* is responsive; touch input as a primary control method is Phase 6 stretch.
- **Localization beyond English.** All strings centralized in `messages.ts` so future localization is easy.
- **Custom in-browser map editor.** Mentioned as diegetically appealing; will only be built if hand-authoring 18 JSON maps proves unworkable. Default: hand-authored.
- **Standalone-app distribution** (Tauri / Electron). Browser-first, browser-only.

---

## 11. Open questions deferred to implementation

These were intentionally not nailed down during brainstorm — they're better resolved during the relevant build phase:

- **Final-boss music for The Hand** — *human In l00p* vs *make It better (mkbtr)*. Decide by playtesting in Phase 5.
- **Cycle 3 map names** — chosen during Phase 4 design as we figure out which C1/C2 spaces feel best to decompile.
- **Difficulty curve** — tuned in Phase 6 based on playtest feedback.
- **Whether Dr. Verra Tessier appears earlier as a recurring threat** before her penultimate-boss fight in Cycle 4. Currently spec'd as final-cycle-only; could shift in Phase 3-4 if narrative pacing wants it.
- **Exact subdirectory location** of LOOM game code inside l0b0tonline (`components/loom/` vs `features/loom/` vs `packages/loom/`). Phase 0 discovery will resolve based on existing convention parity with ShipIt / Bardo.
- **Mouse cursor takeover mechanism** for The Hand phase 3 — uses Pointer Lock API + canvas-coordinate reflection, vs CSS cursor manipulation, vs a "fake cursor sprite" overlay. Decide in Phase 5.

---

## 12. Authority & next step

This spec is the source of truth for what LOOM is. Implementation plans (in `docs/superpowers/plans/`) translate it into concrete buildable phases.

**Next step:** generate the Phase 0 + Phase 1 implementation plan via writing-plans. (Replaces the prior plan that targeted GZDoom + ZScript — that earlier plan is superseded by this revision.)
