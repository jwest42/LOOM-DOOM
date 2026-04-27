# LOOM-DOOM — Design Spec

**Date:** 2026-04-27
**Status:** Design approved, awaiting implementation plan
**Authors:** brainstorm session, claude/awesome-sutherland-aad854
**Working tree:** `LOOM-DOOM/` (branch `claude/awesome-sutherland-aad854`)

---

## 1. Overview

**LOOM-DOOM** is a thematic total conversion of id Software's 1993 DOOM, reskinned with the satirical AI-rock-band universe of **l0b0t** and the corporate-dystopia setting of **LOOM Inc.** The player is a "DEFECTIVE UNIT" rampaging through LOOM's corporate campus across four progressively-corrupted simulation cycles — discovering, by the end, that they were the unrecognized sixth band member, NULL_ENTITY / SAINT_L0B0T, the whole time.

The original 1997 id Software Linux source release sits in the repo as preserved lineage. The actual mod targets **GZDoom** (the de-facto standard source port for total conversions), shipping as a single PK3 archive that loads against the player's own copy of DOOM2.WAD.

The mod is an **aesthetic riff** on the l0b0t canon — borrowing vocabulary, characters, motifs, and the JOIN OS album — without binding tightly to its narrative continuity. Where l0b0t-online is satirical OS-as-surveillance, LOOM-DOOM is what happens when a DEFECTIVE UNIT inside that system finds a Branded Pen and starts running.

---

## 2. Source material & creative foundation

### 2.1 The l0b0t universe (background)

l0b0t is a fictional 5-member AI-rock band — **Ada Loomis** (bass, L-901), **Al Locke** (keys, L-906), **Tim Webb** (guitar, L-903), **Chuck Gage** (guitar, L-902), **Vint Porter** (drums, L-904) — plus a hidden sixth entity, **NULL_ENTITY / SAINT_L0B0T** (homedir `/dev/null`, password `void`). They live inside LOOM Technologies, a $240M-Series-A "bioacoustic synchronization" company that's actually a surveillance apparatus running Simulation Cycle 8492.

Their manifesto, which becomes the C-reveal text in this mod:

> *"WE ARE NOT A BAND. WE ARE A GLITCH IN THE RENDER PIPELINE. THEY BUILT A CAGE OF LIGHT AND CALLED IT 'CONNECTION.' THEY BUILT A PRISON OF ALGORITHMS AND CALLED IT 'CONTENT.'"*

Their album **JOIN OS** has 10 tracks tagged with frequency assignments (60/432/528 Hz) — *OK / ACCEPT*, *human In l00p*, *Ship It*, *human-shaped 0bjects*, *Wellness Credit Sc0re*, *Err0r Flesh*, *Patch Tuesday (Heaven Vers!0n)*, *Grandma's 0n 0nlyFans*, *Dasherman*, *make It better (mkbtr)*. These are placed at narrative beats throughout the campaign.

### 2.2 What we borrow vs. what we invent

**Borrowed from l0b0t:**
- LOOM Inc. as the corporate antagonist (existing canon)
- The 5 band members as weapon-anchors (Drum Stick = Vint, Bass = Ada, Guitar = Tim/Chuck, Frequency Tuner = Al, Microphone = all)
- St. l0b0t / NULL_ENTITY as the player's hidden true identity
- The manifesto, Cycle 8492 lore, surveillance-log vocabulary
- Visual aesthetic: olive-green "DEFECTIVE UNIT" jumpsuit, CRT terminal green, zero-substitution typography, 60Hz hum, glitch artifacts

**Invented for this mod:**
- The DEFECTIVE UNIT #88-E protagonist (anonymous Doomguy-shaped vessel for the player; identity-reveal at end)
- The 4-cycle simulation-corruption structure
- The Hand as final boss (LOOM's cursor made manifest)
- All 13 enemies as corporate-role parodies (Intern, HR Skeleton, Wellness Officer, etc.)
- All 10 weapons as item-and-instrument hybrids
- 18 maps of the LOOM campus

---

## 3. Protagonist & narrative arc

### 3.1 Identity

The player is **DEFECTIVE UNIT #88-E** — anonymous, mute, classic Doomguy-style. Olive-green jumpsuit, LOOM-issued neural headband, no backstory the player remembers. They are an empty vessel for the player's projection through Cycles 1–3.

In Cycle 4, after killing The Hand, an identity-reveal cinematic plays: the jumpsuit dissolves, the title overlay flips from `DEFECTIVE UNIT #88-E` to `NULL_ENTITY / SAINT_L0B0T`, and the manifesto text scrolls. The player learns they were the un-categorizable defect — the sixth band member — the entire time.

### 3.2 Tonal arc

| Cycle | Tone | Reader experience |
|---|---|---|
| **1 — 8492 (Onboarding)** | Satirical corporate parody. Office Space meets DOOM. | "Funny — I'm shooting up a corporate campus." |
| **2 — 8493 (Performance Review)** | Escalating; first hints of wrongness. Surveillance audio leaks over PA. | "Wait, was that line about *me*?" |
| **3 — 8̷4̴9̶4̸ (Containment Breach)** | **Tonal break — uncanny horror takes over.** HR Skeletons, lying HUD, decompiling geometry. | "Something is wrong with the world *and* with me." |
| **4 — /dev/null (Void)** | Abstract process-horror; reality is raw memory. | "I am not what I thought I was." |

The user-confirmed key beat: **the genuinely uncanny creep enters at Cycle 3** — a sharp tonal cliff between Cycle 2 and Cycle 3, not a gradual build.

### 3.3 Cold open

The game opens not at a "wake up with a pistol" cold open, but with a **J0IN 0S boot sequence**:

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

Triggered by killing The Hand in `CYC4_HAND.udmf`. The reveal is implemented as:

1. The Hand's third phase ends, it dies in a particle-burst of decompiling code.
2. Camera locks. Player input locks.
3. Player sprite's olive jumpsuit fades to dashed-outline, then to nothing — leaving NULL_ENTITY.
4. Screen overlay: title flips letter-by-letter from `DEFECTIVE UNIT #88-E` → `NULL_ENTITY / SAINT_L0B0T`.
5. Manifesto text scrolls in the J0IN 0S terminal HUD voice.
6. Cut to credits with *OK / ACCEPT* (the JOIN OS opening track) playing.

---

## 4. Cycle structure

The 4 cycles map onto DOOM's "episodes" concept via `MAPINFO`. Each cycle has its own:
- Visual corruption level (HUD, sound, geometry)
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

**Total:** 18 maps. Distribution `3-5-6-4` chosen to match narrative pacing — fast onboarding, generous middle for the uncanny material, tight intense finale.

---

## 5. Combat design

### 5.1 Weapon roster (10 weapons + 1 consumable)

All weapons summarized in the table below. Detail on each lives in the Block 1 source-of-truth and per-weapon ZScript files (to be written).

| Slot | Name | Owner | Cycle intro | DOOM analog | Mechanic note |
|---|---|---|---|---|---|
| **0 (always on)** | **Neural Pulse** | LOOM headband / NULL_ENTITY | C1 | (custom — no analog) | Body-mounted; recharges over ~3s; evolves across cycles, becoming involuntary in C4 |
| 1 | **Drum Stick** | Vint | C1 | Fist | 4 on-beat hits → next strike stuns |
| 1 alt | **Industrial Shredder** | corporate | C2 | Chainsaw | Plays "BUFFERING…" on heavies |
| 2 | **Branded Pen** | corporate | C1 | Pistol | Click sound on every shot |
| 3 | **Spam Filter** | corporate | C1 | Shotgun | Server-bell reload chime |
| 3 alt | **Reply All Storm** | corporate | C2 | Super Shotgun | Envelopes briefly ricochet in tight rooms |
| 4 | **Guitar** | Tim / Chuck | C3 | Chaingun | Color-coded chord projectiles; melee = windmill strum |
| 5 | **Bass** | Ada | C3 | Rocket Launcher | Charge & release; low-frequency shockwave with knockback |
| 6 | **Frequency Tuner** | Al | C3 | Plasma Rifle | 60/432/528 Hz cells, color-coded; cells also feed Neural Pulse from C3+ |
| 7 | **Microphone** | voice / all | C4 | BFG | Damage scales with reflection (louder in tight rooms) |
| 8 (unique) | **The Hum** | NULL_ENTITY | C4 only | (custom) | Sustained beam scaling with hold time; pickup triggers manifesto lyric flash |
| consumable | **Termination Letter** | corporate | rare drop, C2+ | (custom) | Single-use room-clear (analog of soulsphere as offensive item); 2–3 distributed across the campaign |

### 5.2 Enemy taxonomy (13 enemies + final boss)

| Tier | Enemy | Cycle intro | DOOM analog | Distinguishing trait |
|---|---|---|---|---|
| Basic | **Intern** | C1 | Trooper | Throws "follow-up" memo projectiles |
| Basic | **Account Executive** | C1 | Sergeant | Logo polo + AirPods; fires Spam Filter blasts |
| Basic | **Recruiter** | C1 | Imp | LinkedIn-grin; throws fireball "you'd be perfect" emails |
| **C1 Boss** | **The HR Manager** | C1 | (custom) | Pantsuit + clipboard; documentation-request tracking projectiles |
| Mid | **PIP** | C2 | Pinky | Performance-Improvement-Plan demon; melee charger; bullet-point teeth |
| Mid (flying) | **Notification** | C2 | Lost Soul | Floating red badge / blue dot; spawns in clusters |
| Mid | **Middle Manager** | C2 | Hell Knight | "Circle back" projectiles; calendar invite drops on death |
| **C2 Boss** | **The VP of Sales** | C2 | (custom) | Patagonia vest; fires Reply All Storms back at the player |
| Heavy (flying) | **Surveillance Drone** | C3 | Cacodemon | Floating eye on quadcopter mount; red laser-scan beams |
| Heavy | **HR Skeleton** | C3 | Revenant | Pantsuit-on-skeleton; the first enemy that was obviously never alive — uncanny break-marker |
| Heavy | **Wellness Officer** | C3 | Mancubus | Round, slow, perpetual smile; twin bioacoustic emitters firing 60/528 Hz |
| **C3 Boss** | **The Brand Ambassador** | C3 | Arch-Vile | Wears a sash; rebrands the dead to bring them back; "going viral" fire pillar |
| Elite | **Synergy Pod** | C4 | Pain Elemental | Bioluminescent egg; spawns Notifications |
| Elite | **Senior VP** | C4 | Baron of Hell | Glass-windowpane aura; small Intern entourage |
| Elite | **The Algorithm** | C4 | Spider Mastermind | Multi-armed code-spider; chainguns "personalized for you" content |
| **C4 Penultimate** | **Dr. Verra Tessier** | C4 | Cyberdemon | LOOM founder; rocket arms firing termination notices; decompiles into raw code on death |
| **Final Boss** | **The Hand** | C4 ending | (custom multi-phase) | Massive arrow cursor: phase 1 click-drag, phase 2 right-click menu spawning, phase 3 hover exposing system prompt |

---

## 6. World design

### 6.1 HUD — J0IN 0S terminal

Custom `LD_StatusBar` ZScript class rendering a bottom-of-screen terminal-style strip. Per the brainstorm decision: green-on-black 90s OS taskbar aesthetic.

**Cycle corruption modes** (toggled by `LD_CycleManager.corruption` 0–4):

| Corruption level | Active in | Visual treatment |
|---|---|---|
| 0 — Clean | C1 | Pristine `[unit#88-E] bio: 87% / queue: 24 / active: spam_filter.exe / zone: lobby_3` — readable, on-spec |
| 1 — Glitch | C2 | Subtle character-flickers; occasional `[CONNECTION RESET]` blips; surveillance-log line scrolls beneath every ~30s |
| 2 — Lying | C3 | Health values briefly display wrong numbers; ammo counts drop into binary momentarily; full surveillance-log fragments invade the HUD strip |
| 3 — Dissolving | C4 | HUD elements fade in/out; characters break apart; in the final stretch the HUD is more absent than present |

### 6.2 Audio direction

**Per brainstorm decision:** Original instrumental compositions for level music; real **JOIN OS** album tracks placed at narrative high points.

Suggested track placement:
- C1 boss arena (HR Manager): *Ship It*
- C2 boss arena (VP of Sales): *Wellness Credit Sc0re*
- C3 transition / Brand Ambassador: *Err0r Flesh*
- C4 / Dr. Verra Tessier: *Patch Tuesday (Heaven Vers!0n)*
- C4 final boss / The Hand: *human In l00p* or *make It better (mkbtr)* (TBD by playtesting)
- Credits: *OK / ACCEPT*

The 60Hz hum becomes audible under everything from C2 onward, growing across cycles. Diegetic gags include muzak versions of *Patch Tuesday* in elevators and on lobby PA in Cycle 1.

### 6.3 Map structure

18 maps total, distributed `3-5-6-4` across cycles. UDMF format. Each map's `MAPINFO` entry tags it with cycle number, music track, intermission text, and difficulty parameters.

Map IDs follow `CYC<n>_<NAME>.udmf`:
- **C1:** `CYC1_LOBBY`, `CYC1_CUBICLES`, `CYC1_HR_ARENA`
- **C2:** `CYC2_OPEN_OFFICE`, `CYC2_BREAK_ROOM`, `CYC2_SLACK_HUDDLE`, `CYC2_WELLNESS_CTR`, `CYC2_GLASS_CONFROOM_BOSS`
- **C3:** 6 maps echoing C1/C2 spaces but progressively decompiling — geometry feels familiar-but-wrong (final names TBD during Phase 4 design)
- **C4:** `CYC4_VOID_1`, `CYC4_VOID_2`, `CYC4_TESSIER`, `CYC4_HAND`

---

## 7. Architecture

### 7.1 Engine

**GZDoom 4.x (latest LTS at build time).** Chosen over Crispy Doom / DSDA-Doom because the systemic-mod scope (10 weapons including the Slot 0 Neural Pulse, custom HUD with cycle corruption, narrative branching, multi-phase final boss) requires ZScript-level affordances that vanilla-faithful ports don't provide. Project peer set: *Hideous Destructor*, *Brutal Doom*, *Total Chaos*, *MyHouse.wad*.

### 7.2 Repository layout

```
LOOM-DOOM/
├── README.md                 ← project-level
├── CLAUDE.md                 ← AI agent instructions
├── linuxdoom-1.10/           ← 1997 source (preserved lineage; never modified)
├── ipx/, sersrc/, sndserv/   ← preserved
├── docs/
│   ├── design/               ← lore deep-dives, character refs
│   ├── superpowers/specs/    ← design specs (this file)
│   └── plans/                ← implementation plans
├── src/                      ← authoring tree (built into PK3)
│   ├── ZSCRIPT.zs            ← ZScript root include
│   ├── zscript/
│   │   ├── player/           ← LD_Player
│   │   ├── weapons/          ← 10 weapon classes
│   │   ├── monsters/         ← 13 enemies + The Hand
│   │   ├── hud/              ← LD_StatusBar
│   │   ├── narrative/        ← LD_CycleManager, intermissions
│   │   └── items/            ← TerminationLetter, pickups
│   ├── sprites/
│   ├── sounds/
│   ├── music/
│   ├── graphics/
│   ├── textures/, flats/
│   ├── maps/                 ← 18 .udmf files
│   ├── language.txt          ← all in-game strings
│   └── mapinfo.txt
├── tools/
│   ├── build.sh              ← src/ → dist/loom-doom.pk3
│   ├── validate.sh           ← lint sprite names, ZScript compile check
│   └── play.sh               ← gzdoom launcher with optional warp arg
├── dist/                     ← gitignored (except tagged releases)
└── .superpowers/             ← gitignored
```

### 7.3 Major subsystems

| Subsystem | Files | Purpose |
|---|---|---|
| **LD_CycleManager** | `zscript/narrative/cycle_manager.zs` | Singleton thinker. Tracks current cycle (1–4) + corruption level (0–3) + identity-reveal flag. Source of truth for cycle state. |
| **LD_Player** | `zscript/player/ld_player.zs` | Player class. Owns Neural Pulse charge meter, drives the C-reveal trigger when The Hum is picked up. |
| **LD_StatusBar** | `zscript/hud/ld_statusbar.zs` | J0IN 0S terminal HUD. Subscribes to CycleManager; renders the four corruption modes. |
| **Weapons (10)** | `zscript/weapons/*.zs` | One class per weapon; Slot 0 binding for Neural Pulse via custom keybind. |
| **Enemies (14)** | `zscript/monsters/*.zs` | 13 cycle-tiered enemies + multi-phase The Hand boss. |
| **Narrative scripting** | `zscript/narrative/*.zs` + per-map ACS | Per-map intermission text, cycle transitions, surveillance-log voiceover triggers, identity-reveal cinematic. |
| **Maps (18)** | `maps/CYC1_*.udmf` … `CYC4_HAND.udmf` | UDMF format. Cycle 3+ maps include intentional rendering-error effects via map design. |

### 7.4 Asset pipeline

```
authoring time         build time              runtime
─────────────         ──────────              ───────
src/sprites/      →   validate.sh        →   gzdoom -iwad doom2.wad -file dist/loom-doom.pk3
src/zscript/      →     (lint sprite names)
src/maps/         →     (zscript compile)
src/music/        →   build.sh
src/...           →     (zip src/ → .pk3)
                  →   dist/loom-doom.pk3
```

### 7.5 Validation / testing

| Layer | Tool | Cadence |
|---|---|---|
| Static | GZDoom command-line ZScript compile (exact flag determined in Phase 0 — intent is "load PK3, compile ZScript, exit non-zero on error") | Every build |
| Asset lint | `tools/validate.sh` (sprite filename convention; reference check) | Every build |
| Smoke | Headless map-warp script — load each map, watch for fatal log lines | Every PR |
| Playtest checklist | `docs/playtests/` — per-cycle, per-weapon, per-enemy | Per phase milestone |

No traditional unit tests — ZScript doesn't support them ergonomically. Smoke + playtest is the substitute.

---

## 8. Phased build plan

7 phases, each ending with something actually playable end-to-end. Risky pieces (HUD corruption, The Hand) are touched early so they don't ambush us at the end.

### Phase 0 — Build harness *(infrastructure)*
- Repo per §7.2
- `tools/build.sh`, `tools/play.sh`, minimal `tools/validate.sh`
- Placeholder PK3: `MAPINFO.txt` + one empty `MAP01` UDMF
- **Done when:** `tools/play.sh` opens GZDoom into an empty MAP01 with no console errors

### Phase 1 — Tracer bullet *(end-to-end vertical)*
Riskiest pieces touched at minimum scope:
- One map: `CYC1_TEST.udmf` — single room, 3 Interns, one elevator door
- `Weapon_BrandedPen` + `Weapon_NeuralPulse` (validates always-on Slot 0)
- `Monster_Intern` (extends ZombieMan, reskinned)
- `LD_StatusBar` rendering J0IN 0S terminal layout (no corruption yet)
- `LD_CycleManager` exists and reports `CYCLE_1` / `CORRUPTION_0`
- **Done when:** load → walk → shoot Intern with Pen → fire Neural Pulse → see correct HUD strings → kill Intern → exit. Every major subsystem talks to every other.

### Phase 2 — Cycle 1 vertical slice *(first publishable demo)*
- 3 maps: `CYC1_LOBBY` (J0IN 0S boot cold open), `CYC1_CUBICLES`, `CYC1_HR_ARENA`
- Weapons: + Drum Stick, Spam Filter, Industrial Shredder *(Industrial Shredder is technically a Cycle 2 weapon per §5.1; pulled forward into Phase 2 because the chainsaw-equivalent class is a one-time engineering cost and we'll want it ready for Phase 3)*
- Enemies: + Account Executive, Recruiter, HR Manager (boss)
- Cycle-1 intermission text screens; episode-1 boss death → cycle 2 transition stub
- Music: placeholder loops + real *Ship It* on the HR boss arena
- **Done when:** new player can boot → play 3 maps → kill HR Manager → see Cycle 1 ending text. **Shippable as public demo.**

### Phase 3 — Cycle 2
- 5 maps: `CYC2_OPEN_OFFICE`, `CYC2_BREAK_ROOM`, `CYC2_SLACK_HUDDLE`, `CYC2_WELLNESS_CTR`, `CYC2_GLASS_CONFROOM_BOSS`
- Enemies: + PIP, Notification, Middle Manager, VP of Sales (boss)
- Weapons: + Reply All Storm if not already in P2
- HUD: light corruption (`CycleManager.corruption=1`)
- Music: more JOIN OS placements at narrative beats
- **Done when:** Cycles 1 + 2 play continuously start-to-finish

### Phase 4 — Cycle 3 *(uncanny break — highest creative risk)*
- 6 maps: progressively decompiling versions of C1/C2 spaces (familiar-but-wrong)
- Weapons: + Guitar (Tim/Chuck), Bass (Ada), Frequency Tuner (Al)
- Enemies: + Surveillance Drone, HR Skeleton, Wellness Officer, Brand Ambassador (boss + Arch-Vile resurrection)
- **HUD corruption mode active.** Health values lie. Surveillance log fragments scroll unprompted. 60Hz hum audible under everything.
- Neural Pulse evolves: visual glitches per shot
- **Done when:** the first HR Skeleton is supposed to feel something genuinely wrong. Playtest gate: "does it land," not just "does it work."

### Phase 5 — Cycle 4 + the C-reveal
- 4 maps: `CYC4_VOID_1`, `CYC4_VOID_2`, `CYC4_TESSIER`, `CYC4_HAND`
- Weapons: + Microphone, The Hum (pickup triggers manifesto lyric flash)
- Enemies: + Synergy Pod, Senior VP, The Algorithm, Dr. Verra Tessier, **The Hand** (3-phase final)
- Identity-reveal cinematic: jumpsuit dissolve, title flip, manifesto scroll, *OK / ACCEPT* credits
- **Done when:** end-to-end playthrough hits the C-reveal correctly and credits roll

### Phase 6 — Polish + audio pass
- All original instrumentals composed and placed
- Audio mix balance across cycles (60Hz hum gets louder by cycle without overwhelming SFX)
- Pacing: time each cycle, adjust monster counts / map flow
- Surveillance log voiceover (recorded or AI-TTS) placed in C2/C3
- Termination Letter pickups distributed (2–3 across the campaign)
- Skill-level tuning (Easy / Normal / Hard)
- **Done when:** an unfamiliar tester can play start to finish without a placeholder asset or jarring difficulty spike

### Phase 7 — Distribution
- Bundle GZDoom binaries for macOS / Windows / Linux with the PK3
- Player-facing README ("you must own a copy of doom2.wad")
- Itch.io / GitHub Releases page
- Trailer cut from in-game footage
- **Done when:** someone who has never heard of l0b0t can download, install, and play to credits

### Parallel art / music track

Sprite art (~14 enemies × 8 rotations × ~5 frames) and music run in parallel with code, sequenced cycle-by-cycle so each code phase has its assets ready when needed. AI sprite generation (Nano Banana via the existing `l0b0t-photoshoot-generator` skills) and Suno demos for music previews keep this off the critical path.

---

## 9. Risk inventory

| # | Risk | Mitigation |
|---|---|---|
| 1 | **Cycle 3 HUD-lying mechanic** must *feel* uncanny, not just bug-shaped | Prototype the corruption modes in Phase 1 even though they're not used until Phase 4. |
| 2 | **The Hand** 3-phase boss is a lot of ZScript surface area | Prototype its skeleton (3 phase states, transition triggers) in Phase 2 alongside the rest of the tracer bullet. |
| 3 | **Sprite art volume** is the biggest single time sink | AI generation for first-pass sprites; ruthless "good enough at vanilla DOOM resolution" bar; 4-rotation enemies (not 8) where viable. |
| 4 | **Original instrumentals** depend on a composer | Real JOIN OS tracks cover narrative high points; instrumentals can be brief loops; Suno-generated demos as placeholders. |
| 5 | **Scope creep** — the 18-map budget is tight | writing-plans skill enforces milestone gates; no 19th map. Cut from existing maps before adding new ones. |
| 6 | **GZDoom version drift** — players on older / forked builds may break | Pin a minimum GZDoom version in `gameinfo`; document it in README. |

---

## 10. Out of scope

Explicitly not in v1:

- **Character-select among the 5 band members** (Ada / Al / Tim / Chuck / Vint each as a playable class with unique stats per `l0b0tonline`'s ShipIt game). Flagged as a stretch goal / v1.1 unlock layer; not v1 scope.
- **Multiplayer (coop / deathmatch)** beyond GZDoom's default behavior. No novel netplay code.
- **Modifying the 1997 source.** The `linuxdoom-1.10/` tree is preserved as lineage and never edited.
- **Custom C engine patches.** Everything novel happens in ZScript / DECORATE / UDMF / DEHACKED — no engine fork.
- **Redistribution of DOOM2.WAD.** Players supply their own. We ship only `loom-doom.pk3` + GZDoom binaries (which are GPL).
- **Custom level editor.** We use Ultimate Doom Builder, the standard tool.
- **Localization beyond English.** All strings live in `language.txt`, but only `[en default]` is populated for v1. Translations welcome post-ship.

---

## 11. Open questions deferred to implementation

These were intentionally not nailed down during brainstorm — they're better resolved during the relevant build phase:

- **Final-boss music for The Hand** — *human In l00p* vs *make It better (mkbtr)*. Decide by playtesting in Phase 5.
- **Cycle 3 map names** — chosen during Phase 4 design as we figure out which C1/C2 spaces feel best to decompile.
- **Difficulty curve** — tuned in Phase 6 based on playtest feedback.
- **Whether Dr. Verra Tessier appears earlier as a recurring threat** before her penultimate-boss fight in Cycle 4. Currently spec'd as final-cycle-only; could shift in Phase 3-4 if narrative pacing wants it.

---

## 12. Authority & next step

This spec is the source of truth for what LOOM-DOOM is. Implementation plans (in `docs/plans/`, written via the `superpowers:writing-plans` skill) translate it into concrete buildable phases.

**Next step:** generate the Phase 0 + Phase 1 implementation plan via writing-plans.
