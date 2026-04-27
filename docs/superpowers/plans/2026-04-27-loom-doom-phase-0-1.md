# LOOM-DOOM Phase 0 + Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the build harness for the LOOM-DOOM mod (Phase 0) and ship a tracer-bullet end-to-end vertical (Phase 1) — one map, two weapons (Branded Pen + Slot 0 Neural Pulse), one enemy (Intern), the J0IN 0S terminal HUD shell, and a working `LD_CycleManager` singleton.

**Architecture:** GZDoom 4.x mod packaged as `loom-doom.pk3`. ZScript drives all custom mechanics. Maps authored in SLADE 3, packed as WADs inside the PK3 under `maps/`. Neural Pulse is an Inventory item + `StaticEventHandler` (not a `Weapon`) bound to a custom KEYCONF keybind. Custom HUD inherits `BaseStatusBar` and is registered via MAPINFO. Cycle state lives in a `StaticEventHandler` singleton that survives map transitions.

**Tech Stack:** GZDoom 4.14+, ZScript v4.10, SLADE 3, bash, macOS host (player provides their own `doom2.wad`).

**Pre-existing state:**
- Repo at `/Users/justinwest/Repos/LOOM-DOOM` is a git worktree on branch `claude/awesome-sutherland-aad854`
- `linuxdoom-1.10/`, `ipx/`, `sersrc/`, `sndserv/`, `LICENSE.TXT`, `README.TXT` — preserved 1997 source. **Do not modify these.**
- `.gitignore` exists with `dist/`, `.superpowers/`, `.DS_Store`, etc.
- `docs/superpowers/specs/2026-04-27-loom-doom-design.md` — the design spec (the source of truth this plan implements)

---

## File structure (post-Phase-1)

```
LOOM-DOOM/
├── .gitignore                                 (already exists)
├── README.md                                  (NEW — Task 0.2)
├── CLAUDE.md                                  (NEW — Task 0.2)
├── linuxdoom-1.10/, ipx/, sersrc/, sndserv/   (preserved, untouched)
├── LICENSE.TXT, README.TXT                    (preserved, untouched)
├── docs/superpowers/                          (already exists)
│   ├── specs/2026-04-27-loom-doom-design.md   (already exists)
│   └── plans/2026-04-27-loom-doom-phase-0-1.md (this file)
├── src/
│   ├── GAMEINFO.txt                           (NEW — Task 0.4)
│   ├── MAPINFO.txt                            (NEW — Task 0.4 + extended in 1.2, 1.3, 1.7)
│   ├── KEYCONF.txt                            (NEW — Task 1.6)
│   ├── zscript.txt                            (NEW — Task 1.1, root ZScript include)
│   ├── zscript/
│   │   ├── cycle_manager.zs                   (NEW — Task 1.2)
│   │   ├── ld_player.zs                       (NEW — Task 1.3)
│   │   ├── branded_pen.zs                     (NEW — Task 1.4)
│   │   ├── intern.zs                          (NEW — Task 1.5)
│   │   ├── neural_pulse.zs                    (NEW — Task 1.6)
│   │   └── ld_statusbar.zs                    (NEW — Task 1.7)
│   ├── sprites/
│   │   ├── PENGA0.png                         (NEW — Task 1.4, placeholder)
│   │   ├── PENGB0.png                         (NEW — Task 1.4, placeholder)
│   │   ├── INTRA0.png ... INTRG0.png          (NEW — Task 1.5, placeholders, single-rotation)
│   │   └── NPLSA0.png                         (NEW — Task 1.6, placeholder)
│   └── maps/
│       └── CYC1_TEST.wad                      (NEW — Task 0.3, authored in SLADE)
├── tools/
│   ├── build.sh                               (NEW — Task 0.5)
│   ├── play.sh                                (NEW — Task 0.6)
│   └── validate.sh                            (NEW — Task 0.5, scaffolded only — full impl in later phases)
└── dist/                                      (gitignored, populated by build.sh)
    └── loom-doom.pk3                          (build output)
```

---

# Phase 0 — Build harness

**Phase 0 Definition of Done:** running `./tools/play.sh` opens GZDoom into our empty `CYC1_TEST` map with no console errors. The full pipeline (src → build.sh → dist/loom-doom.pk3 → GZDoom) works end-to-end.

---

## Task 0.1: Verify dev toolchain

**Files:**
- Read-only — no files created in this task

- [ ] **Step 1: Verify GZDoom is installed (or install it)**

Run:
```bash
ls /Applications/GZDoom.app/Contents/MacOS/gzdoom 2>&1
```
Expected: path printed. If "No such file":
```bash
brew install --cask gzdoom
```
Re-verify: the binary should now exist.

- [ ] **Step 2: Confirm GZDoom version**

Run:
```bash
/Applications/GZDoom.app/Contents/MacOS/gzdoom -version 2>&1 | head -5
```
Expected: a version line like `GZDoom v4.14.x` or similar. **Note the version** — we'll pin this in `zscript.txt` Step 1 of Task 1.1.

- [ ] **Step 3: Verify SLADE is installed (or install it)**

Run:
```bash
ls /Applications/SLADE.app 2>&1
```
Expected: path printed. If "No such file":
```bash
brew install --cask slade
```
If the cask doesn't exist on this Homebrew, fall back to: download the macOS build from https://slade.mancubus.net/ and install the .app to `/Applications/`.

- [ ] **Step 4: Verify the player has a copy of doom2.wad somewhere on disk**

Run:
```bash
find ~/Library/Application\ Support ~/Documents ~/Desktop /Applications -iname "doom2.wad" 2>/dev/null | head -5
```
Expected: at least one path printed. If not, the player must supply their own copy. **Save the absolute path** — `play.sh` will reference it.

If no doom2.wad found, ask the user to point us at one before continuing. **Do not proceed without a valid IWAD path.**

- [ ] **Step 5: Verify git worktree state**

Run:
```bash
git status && git branch --show-current
```
Expected: clean working tree, current branch `claude/awesome-sutherland-aad854`.

---

## Task 0.2: Create directory skeleton + project README + CLAUDE.md

**Files:**
- Create: `src/`, `src/zscript/`, `src/sprites/`, `src/maps/`, `tools/`
- Create: `README.md`
- Create: `CLAUDE.md`

- [ ] **Step 1: Create the directory tree**

Run:
```bash
mkdir -p src/zscript src/sprites src/maps tools
ls -la src tools
```
Expected: both directories exist; src/ has zscript, sprites, maps subdirs.

- [ ] **Step 2: Write project-level README.md**

Create `README.md` with this content:
```markdown
# LOOM-DOOM

A thematic GZDoom total conversion of DOOM, reskinned with the satirical
AI-rock-band universe of l0b0t and the corporate-dystopia setting of
LOOM Inc. The original 1997 id Software Linux source release sits in
`linuxdoom-1.10/` as preserved lineage; the actual mod targets GZDoom and
ships as `loom-doom.pk3`.

## Requirements

- GZDoom 4.14+ (`brew install --cask gzdoom`)
- A copy of `doom2.wad` you legally own (we don't redistribute it)
- SLADE 3 if you want to edit maps locally (`brew install --cask slade`)
- macOS / Linux / Windows (build scripts assume bash)

## Build

```bash
./tools/build.sh
./tools/play.sh
```

`build.sh` zips `src/` into `dist/loom-doom.pk3`. `play.sh` launches
GZDoom with the PK3 against your local `doom2.wad`.

## Design

See [docs/superpowers/specs/2026-04-27-loom-doom-design.md](docs/superpowers/specs/2026-04-27-loom-doom-design.md).

## License

The 1997 id Software source under `linuxdoom-1.10/`, `ipx/`, `sersrc/`,
`sndserv/` is GPL — see `LICENSE.TXT`. Original LOOM-DOOM mod assets and
code are GPL-3.0 (matching the 1997 release lineage).
```

- [ ] **Step 3: Write CLAUDE.md**

Create `CLAUDE.md` with this content:
```markdown
# CLAUDE.md

Instructions for AI agents working in this repo.

## Project context

LOOM-DOOM is a GZDoom total conversion. The 1997 DOOM source under
`linuxdoom-1.10/` (and the IPX/serial/sound-server subdirs) is preserved
lineage — **never modify it**. All actual mod code lives in `src/`.

## Source of truth

The design spec at `docs/superpowers/specs/2026-04-27-loom-doom-design.md`
is the canonical reference for what LOOM-DOOM is. When in doubt, read
that. Implementation plans live in `docs/superpowers/plans/`.

## Conventions

- ZScript files: lowercase_snake_case.zs under `src/zscript/`
- Sprite filenames: 4-character base + frame letter + rotation digit
  (e.g. `PENGA0.png`, `INTRA0.png`). Use rotation `0` for non-rotating
  single-frame sprites.
- Maps live as `.wad` files under `src/maps/`. Author with SLADE 3 or
  Ultimate Doom Builder.
- Commit per task. Frequent commits. Don't batch unrelated changes.

## Build & run

- `./tools/build.sh` produces `dist/loom-doom.pk3`
- `./tools/play.sh` launches GZDoom with the PK3 (optional arg: map name to warp to)

## Out of scope

- Modifying the 1997 source lineage
- Custom C engine patches (everything lives in ZScript / DECORATE / UDMF)
- Redistributing doom2.wad
```

- [ ] **Step 4: Commit**

```bash
git add README.md CLAUDE.md
git commit -m "Add project README and CLAUDE.md"
```

---

## Task 0.3: Author placeholder UDMF map (CYC1_TEST.wad) in SLADE

**Files:**
- Create: `src/maps/CYC1_TEST.wad`

This task is GUI-driven (SLADE is a desktop app). If running headless / via subagent, request the user perform this step themselves and confirm before continuing.

- [ ] **Step 1: Open SLADE and create a new empty WAD**

Launch `/Applications/SLADE.app`. Menu: `File → New → WAD Archive`.

- [ ] **Step 2: Add a new UDMF map**

Menu: `Archive → New → Map`. In the dialog:
- Map name: `CYC1_TEST`
- Format: `UDMF`
- Game configuration: `Doom 2 (UDMF)`
- Click `Create Map`

This opens SLADE's built-in map editor.

- [ ] **Step 3: Draw a 256×256 room**

In the map editor:
- Press `D` for Drawing mode
- Click 4 corners to form a square sector (suggest coordinates `(-128,-128)`, `(128,-128)`, `(128,128)`, `(-128,128)`)
- Press `Enter` to close the sector

The room is created with default floor/ceiling textures (likely STARTAN3 walls, FLOOR4_8 floor).

- [ ] **Step 4: Place player 1 start and 3 placeholder things (will become Interns later)**

Press `T` for Things mode:
- Place a Player 1 Start (Thing type 1) near the center, facing east (angle 0)
- Place 3 generic monsters (any default monster — they'll be replaced via DECORATE-style swap later) at three corners

Save the map (`Ctrl+S` / `Cmd+S`).

- [ ] **Step 5: Save the WAD**

Menu: `File → Save As`. Save as `src/maps/CYC1_TEST.wad` in the project repo.

Verify on disk:
```bash
ls -la src/maps/CYC1_TEST.wad
```
Expected: file exists, size >100 bytes.

- [ ] **Step 6: Confirm the WAD parses**

Reopen it in SLADE: `File → Open` → `src/maps/CYC1_TEST.wad`. The map should load and display the room. If errors are shown, return to Step 3.

- [ ] **Step 7: Commit**

```bash
git add src/maps/CYC1_TEST.wad
git commit -m "Add CYC1_TEST.wad placeholder map for tracer bullet"
```

---

## Task 0.4: Write GAMEINFO and MAPINFO

**Files:**
- Create: `src/GAMEINFO.txt`
- Create: `src/MAPINFO.txt`

- [ ] **Step 1: Write GAMEINFO.txt**

GZDoom looks for this lump in the PK3 root and reads it before everything else. It tells GZDoom which IWAD this mod targets.

Create `src/GAMEINFO.txt`:
```
IWAD = "doom2.wad"
LOAD = ""
NOSPRITERENAME = "true"
STARTUPTITLE = "LOOM-DOOM"
```

- [ ] **Step 2: Write MAPINFO.txt**

Create `src/MAPINFO.txt`:
```
gameinfo
{
    titlepage = "TITLEPIC"
    creditpage = "CREDIT"
    intermissionmusic = "$MUSIC_DM2INT"
}

map CYC1_TEST "Tracer Bullet (CYC1_TEST)"
{
    next = "CYC1_TEST"
    sky1 = "SKY1"
    music = "$MUSIC_RUNNIN"
    par = 60
}

clearepisodes
episode CYC1_TEST
{
    name = "Cycle 1 - Tracer Bullet"
    key = "1"
}
```

This defines a single map (`CYC1_TEST`), wires it into a one-episode game, and gives it placeholder stock DOOM textures (TITLEPIC, SKY1) and music. We'll replace these in later phases.

- [ ] **Step 3: Verify file contents**

Run:
```bash
cat src/GAMEINFO.txt src/MAPINFO.txt
```
Expected: both files print, no syntax errors visible. (Real validation comes in Task 0.6 when we launch GZDoom.)

- [ ] **Step 4: Commit**

```bash
git add src/GAMEINFO.txt src/MAPINFO.txt
git commit -m "Add GAMEINFO + MAPINFO with single tracer-bullet map definition"
```

---

## Task 0.5: Write tools/build.sh and scaffold validate.sh

**Files:**
- Create: `tools/build.sh`
- Create: `tools/validate.sh` (scaffold only)

- [ ] **Step 1: Write tools/build.sh**

Create `tools/build.sh`:
```bash
#!/usr/bin/env bash
# build.sh — package src/ into dist/loom-doom.pk3
#
# A PK3 is just a ZIP with a different extension. GZDoom reads it
# transparently. We zip from inside src/ so the archive root contains
# GAMEINFO.txt etc. directly (not nested under "src/").

set -euo pipefail

REPO_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
SRC_DIR="$REPO_ROOT/src"
DIST_DIR="$REPO_ROOT/dist"
PK3_PATH="$DIST_DIR/loom-doom.pk3"

if [[ ! -d "$SRC_DIR" ]]; then
    echo "FATAL: $SRC_DIR does not exist" >&2
    exit 1
fi

mkdir -p "$DIST_DIR"
rm -f "$PK3_PATH"

cd "$SRC_DIR"
# Exclude macOS junk and editor backup files. -r recurses; -X strips extra attrs.
zip -r -X -q "$PK3_PATH" . \
    -x ".DS_Store" "*/.DS_Store" "*~" "*.swp"

echo "Built: $PK3_PATH ($(du -h "$PK3_PATH" | cut -f1))"
```

- [ ] **Step 2: Make build.sh executable**

```bash
chmod +x tools/build.sh
ls -l tools/build.sh
```
Expected: permissions show `-rwxr-xr-x` (or similar with execute bit).

- [ ] **Step 3: Run build.sh and verify output**

```bash
./tools/build.sh
```
Expected stdout: `Built: /path/to/dist/loom-doom.pk3 (XK)`.

Verify:
```bash
ls -la dist/
unzip -l dist/loom-doom.pk3 | head -20
```
Expected: `loom-doom.pk3` exists in `dist/`. The archive listing shows `GAMEINFO.txt`, `MAPINFO.txt`, `maps/CYC1_TEST.wad`, and the empty `zscript/` and `sprites/` directories (no zscript.txt yet).

- [ ] **Step 4: Write tools/validate.sh scaffold**

Create `tools/validate.sh`:
```bash
#!/usr/bin/env bash
# validate.sh — lint sprite filenames + asset reference checks
#
# Phase 0+1 scaffold: this script is intentionally minimal. It will be
# extended in later phases as we accumulate sprite-naming conventions
# and ZScript class references that need cross-checking.

set -euo pipefail

REPO_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
SRC_DIR="$REPO_ROOT/src"

errors=0

# Check 1: sprite filenames follow 4-char + frame + rotation pattern (when sprites exist)
if compgen -G "$SRC_DIR/sprites/*.png" > /dev/null; then
    while IFS= read -r f; do
        bn="$(basename "$f" .png)"
        # Match XXXXY[0-8] (8 chars total: 4 base + 1 frame letter + 1 rotation digit) OR
        #       XXXXYZW[0-8] (mirrored frames) — for now, just check 6-char baseline
        if [[ ! "$bn" =~ ^[A-Z0-9]{4}[A-Z][0-8]$ ]] && [[ ! "$bn" =~ ^[A-Z0-9]{4}[A-Z][0-8][A-Z][0-8]$ ]]; then
            echo "ERROR: sprite filename does not match DOOM naming convention: $bn" >&2
            errors=$((errors + 1))
        fi
    done < <(find "$SRC_DIR/sprites" -name "*.png" -type f)
fi

if (( errors > 0 )); then
    echo "Validation failed: $errors error(s)" >&2
    exit 1
fi

echo "Validation passed."
```

- [ ] **Step 5: Make validate.sh executable and run it**

```bash
chmod +x tools/validate.sh
./tools/validate.sh
```
Expected: `Validation passed.` (no sprites yet, so nothing to validate).

- [ ] **Step 6: Commit**

```bash
git add tools/build.sh tools/validate.sh
git commit -m "Add build.sh (src/ -> dist/loom-doom.pk3) and validate.sh scaffold"
```

---

## Task 0.6: Write tools/play.sh and verify Phase 0 DoD

**Files:**
- Create: `tools/play.sh`

- [ ] **Step 1: Write tools/play.sh**

Create `tools/play.sh`:
```bash
#!/usr/bin/env bash
# play.sh — launch GZDoom with our mod
#
# Usage:
#   ./tools/play.sh                    # boot to title screen
#   ./tools/play.sh CYC1_TEST          # warp directly to a map
#
# Requires:
#   - GZDoom installed (default: /Applications/GZDoom.app)
#   - doom2.wad at $LOOM_DOOM_IWAD or one of the fallback paths below
#   - dist/loom-doom.pk3 (run ./tools/build.sh first if missing)

set -euo pipefail

REPO_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
PK3_PATH="$REPO_ROOT/dist/loom-doom.pk3"

GZDOOM_BIN="${GZDOOM_BIN:-/Applications/GZDoom.app/Contents/MacOS/gzdoom}"

# IWAD resolution: env var first, then check common locations
IWAD="${LOOM_DOOM_IWAD:-}"
if [[ -z "$IWAD" ]]; then
    for candidate in \
        "$HOME/doom2.wad" \
        "$HOME/Documents/doom2.wad" \
        "$HOME/Desktop/doom2.wad" \
        "$HOME/Library/Application Support/gzdoom/doom2.wad" \
        "$REPO_ROOT/doom2.wad"; do
        if [[ -f "$candidate" ]]; then
            IWAD="$candidate"
            break
        fi
    done
fi

if [[ -z "$IWAD" || ! -f "$IWAD" ]]; then
    echo "FATAL: doom2.wad not found." >&2
    echo "Set LOOM_DOOM_IWAD to its path, e.g.:" >&2
    echo "  export LOOM_DOOM_IWAD=/path/to/doom2.wad" >&2
    exit 1
fi

if [[ ! -x "$GZDOOM_BIN" ]]; then
    echo "FATAL: gzdoom binary not found at $GZDOOM_BIN" >&2
    echo "Install: brew install --cask gzdoom" >&2
    echo "Or set GZDOOM_BIN to override the path." >&2
    exit 1
fi

if [[ ! -f "$PK3_PATH" ]]; then
    echo "loom-doom.pk3 not built yet — running build.sh first…" >&2
    "$REPO_ROOT/tools/build.sh"
fi

# Build the GZDoom command
ARGS=(
    -iwad "$IWAD"
    -file "$PK3_PATH"
    -nostartup
)

# If a map argument was passed, append a +map console command to warp directly
if [[ $# -gt 0 ]]; then
    MAP_NAME="$1"
    ARGS+=(+map "$MAP_NAME")
    echo "Launching with warp to map: $MAP_NAME"
fi

echo "Running: $GZDOOM_BIN ${ARGS[*]}"
exec "$GZDOOM_BIN" "${ARGS[@]}"
```

- [ ] **Step 2: Make play.sh executable**

```bash
chmod +x tools/play.sh
```

- [ ] **Step 3: Run play.sh — expect GZDoom title screen**

```bash
./tools/play.sh
```
Expected: GZDoom window opens, shows the standard DOOM 2 title screen. No fatal errors in the console (the GZDoom log will print to terminal alongside).

If it fails:
- "doom2.wad not found": set `LOOM_DOOM_IWAD` env var to the actual path
- "Bad MAPINFO" or similar: re-check `src/MAPINFO.txt` syntax (Task 0.4)
- ZScript-related error: there shouldn't be any yet — we have no zscript.txt; if you see ZScript errors, examine the log

Quit GZDoom (`Cmd+Q` or close the window).

- [ ] **Step 4: Run play.sh with map warp arg — Phase 0 DoD**

```bash
./tools/play.sh CYC1_TEST
```
Expected: GZDoom launches and warps directly into the empty CYC1_TEST room. You can walk around the placeholder room. No errors.

**This is the Phase 0 Definition of Done — the build pipeline works end-to-end.**

Quit GZDoom.

- [ ] **Step 5: Commit**

```bash
git add tools/play.sh
git commit -m "Add play.sh (Phase 0 done — empty CYC1_TEST loads cleanly)"
```

---

# Phase 1 — Tracer bullet

**Phase 1 Definition of Done:** Player loads into `CYC1_TEST`, sees the J0IN 0S terminal HUD with cycle indicator, has Branded Pen + always-on Neural Pulse, can shoot 3 Interns, kill them all. Console shows `LD_CycleManager: registered, cycle=1, corruption=0` at startup. **All major subsystems are wired together.**

---

## Task 1.1: ZSCRIPT root file + LANGUAGE stub

**Files:**
- Create: `src/zscript.txt`

- [ ] **Step 1: Write zscript.txt root file**

Create `src/zscript.txt`:
```
version "4.10"

// Includes added per-task as new ZScript classes are introduced.
// Order matters when classes reference each other; cycle_manager is
// included first because both ld_player and ld_statusbar look it up.

#include "zscript/cycle_manager.zs"
#include "zscript/ld_player.zs"
#include "zscript/branded_pen.zs"
#include "zscript/intern.zs"
#include "zscript/neural_pulse.zs"
#include "zscript/ld_statusbar.zs"
```

Note: we declare all the includes up front, even though the referenced files don't all exist yet. That's intentional — we'll make ZScript empty stubs as we go. Adding includes one-at-a-time would mean editing zscript.txt 6 times.

- [ ] **Step 2: Create empty stub files for every include**

Run:
```bash
cd src
touch zscript/cycle_manager.zs zscript/ld_player.zs zscript/branded_pen.zs \
      zscript/intern.zs zscript/neural_pulse.zs zscript/ld_statusbar.zs
ls zscript/
cd ..
```
Expected: 6 empty .zs files exist.

- [ ] **Step 3: Verify the build still succeeds with empty ZScript files**

```bash
./tools/build.sh
```
Expected: `Built: ...` no errors.

- [ ] **Step 4: Verify GZDoom launches with empty ZScript files**

```bash
./tools/play.sh CYC1_TEST
```
Expected: GZDoom launches into CYC1_TEST. **Watch the console output for any ZScript parse errors.** Empty files should parse cleanly.

If you see "ZSCRIPT must include version directive" or similar — it means GZDoom isn't finding `zscript.txt`. Verify it lives at the PK3 root (not under `src/`):
```bash
unzip -l dist/loom-doom.pk3 | grep zscript
```
Expected: `zscript.txt` and `zscript/cycle_manager.zs`, etc., listed.

Quit GZDoom.

- [ ] **Step 5: Commit**

```bash
git add src/zscript.txt src/zscript/
git commit -m "Scaffold ZScript root with empty stubs for Phase 1 classes"
```

---

## Task 1.2: LD_CycleManager StaticEventHandler

**Files:**
- Modify: `src/zscript/cycle_manager.zs`
- Modify: `src/MAPINFO.txt`

- [ ] **Step 1: Write the LD_CycleManager class**

Replace `src/zscript/cycle_manager.zs` content with:
```
// LD_CycleManager — global singleton tracking cycle state across maps.
//
// StaticEventHandler is GZDoom's idiom for game-wide state that survives
// map transitions but is reset on game restart. (NOT save-game-persisted.)
// Instances of this class are looked up via StaticEventHandler.Find().
//
// Cycle: 1..4  — which simulation cycle the player is in
// Corruption: 0..3  — visual / mechanical corruption level for this cycle
//   0 = clean (Cycle 1)
//   1 = glitch (Cycle 2 — subtle)
//   2 = lying (Cycle 3 — uncanny break)
//   3 = dissolving (Cycle 4 — void)
// Identity revealed: bool — true after player picks up The Hum (Phase 5).

class LD_CycleManager : StaticEventHandler
{
    int currentCycle;
    int corruptionLevel;
    bool identityRevealed;

    override void OnRegister()
    {
        currentCycle = 1;
        corruptionLevel = 0;
        identityRevealed = false;
        Console.Printf("LD_CycleManager: registered, cycle=%d, corruption=%d", currentCycle, corruptionLevel);
    }

    // Convenience accessor used by other systems.
    static clearscope LD_CycleManager Get()
    {
        return LD_CycleManager(StaticEventHandler.Find("LD_CycleManager"));
    }
}
```

- [ ] **Step 2: Register LD_CycleManager in MAPINFO**

Edit `src/MAPINFO.txt` and add the `AddEventHandlers` line inside the `gameinfo` block. Replace:
```
gameinfo
{
    titlepage = "TITLEPIC"
    creditpage = "CREDIT"
    intermissionmusic = "$MUSIC_DM2INT"
}
```
with:
```
gameinfo
{
    titlepage = "TITLEPIC"
    creditpage = "CREDIT"
    intermissionmusic = "$MUSIC_DM2INT"
    AddEventHandlers = "LD_CycleManager"
}
```

- [ ] **Step 3: Build and run**

```bash
./tools/build.sh && ./tools/play.sh CYC1_TEST
```

- [ ] **Step 4: Verify CycleManager registered**

In the GZDoom console (toggle with `~`), run:
```
logfile gzdoom.log
```
(or just visually scroll the launch console).

Look for the line: `LD_CycleManager: registered, cycle=1, corruption=0`

Expected: that exact string (with our values) appears once during startup.

If not present:
- Did you save MAPINFO.txt? (Re-run `./tools/build.sh` if you forgot.)
- ZScript parse error in `cycle_manager.zs`? Check the console output for compile errors.
- Class name mismatch? `AddEventHandlers` value must exactly match the class name `LD_CycleManager`.

Quit GZDoom.

- [ ] **Step 5: Commit**

```bash
git add src/zscript/cycle_manager.zs src/MAPINFO.txt
git commit -m "Add LD_CycleManager StaticEventHandler with cycle/corruption state"
```

---

## Task 1.3: LD_Player class (extends DoomPlayer)

**Files:**
- Modify: `src/zscript/ld_player.zs`
- Modify: `src/MAPINFO.txt`

- [ ] **Step 1: Write the LD_Player class**

Replace `src/zscript/ld_player.zs` content with:
```
// LD_Player — the player class for LOOM-DOOM. Extends DoomPlayer.
//
// In Phase 1 this is a thin wrapper. Future phases attach:
//   - Neural Pulse charge meter
//   - Identity-reveal trigger logic
//   - Drum Stick rhythm meter
//   - Cycle-aware sound + visual feedback

class LD_Player : DoomPlayer
{
    Default
    {
        Player.DisplayName "Defective Unit";
        Player.StartItem "BrandedPen";
        Player.StartItem "Clip", 50;
    }
}
```

`Player.StartItem "BrandedPen"` references the class we'll define in Task 1.4. GZDoom resolves this at runtime — defining LD_Player before BrandedPen is fine.

- [ ] **Step 2: Register LD_Player as the player class in MAPINFO**

Edit `src/MAPINFO.txt`. Inside the `gameinfo` block (already added in 1.2), add the `playerclasses` line:
```
gameinfo
{
    titlepage = "TITLEPIC"
    creditpage = "CREDIT"
    intermissionmusic = "$MUSIC_DM2INT"
    AddEventHandlers = "LD_CycleManager"
    playerclasses = "LD_Player"
}
```

- [ ] **Step 3: Build and run**

```bash
./tools/build.sh && ./tools/play.sh CYC1_TEST
```

Expected behavior in-game: player spawns. Pulling up the inventory or trying to fire will fail because `BrandedPen` doesn't exist yet — but the player class itself should load. Look for ZScript errors in the console.

If you see "Unknown actor 'BrandedPen'" — that's expected at this point. We'll fix it in Task 1.4. The class itself should still load.

If you see "Unknown player class 'LD_Player'" — fix the MAPINFO `playerclasses` line.

- [ ] **Step 4: Verify in console**

Open GZDoom console (`~`) and run:
```
puke 0
classlist | grep LD_Player
```
Expected: `LD_Player (Player class)` appears.

Quit GZDoom.

- [ ] **Step 5: Commit**

```bash
git add src/zscript/ld_player.zs src/MAPINFO.txt
git commit -m "Add LD_Player class extending DoomPlayer; wire as default player class"
```

---

## Task 1.4: Branded Pen weapon (placeholder sprites + ZScript class)

**Files:**
- Create: `src/sprites/PENGA0.png` (placeholder)
- Create: `src/sprites/PENGB0.png` (placeholder)
- Modify: `src/zscript/branded_pen.zs`

- [ ] **Step 1: Generate placeholder weapon sprites**

We need 2 placeholder PNGs for the Branded Pen — one for the idle pose (PENGA0), one for the firing pose (PENGB0). Both should be ~60×40 px, transparent background, with a visible "PEN" indicator so we know it's our placeholder and not vanilla DOOM's pistol.

If `python3` and Pillow are available, use this script. Save it as `tools/_gen_placeholder_sprites.py` (the leading underscore signals it's a one-off helper):
```python
#!/usr/bin/env python3
# One-off helper to generate Phase 1 placeholder sprites.
# Run from repo root: python3 tools/_gen_placeholder_sprites.py
from PIL import Image, ImageDraw, ImageFont
from pathlib import Path

OUT = Path(__file__).resolve().parent.parent / "src" / "sprites"
OUT.mkdir(parents=True, exist_ok=True)

def make(name: str, w: int, h: int, text: str, bg=(50, 50, 30, 255), fg=(212, 165, 116, 255)):
    img = Image.new("RGBA", (w, h), bg)
    d = ImageDraw.Draw(img)
    d.rectangle([(0, 0), (w-1, h-1)], outline=fg, width=2)
    d.text((4, h // 2 - 6), text, fill=fg)
    img.save(OUT / f"{name}.png")
    print(f"wrote {OUT / (name + '.png')}")

make("PENGA0", 80, 60, "PEN A")    # Branded Pen idle
make("PENGB0", 80, 60, "PEN B!")   # Branded Pen firing
```

Run:
```bash
python3 tools/_gen_placeholder_sprites.py
ls src/sprites/
```
Expected: `PENGA0.png` and `PENGB0.png` exist.

If python3 / Pillow isn't available: open SLADE → New Entry → Image → 80×60 → fill with a dark color → save as `src/sprites/PENGA0.png`. Repeat for PENGB0.

- [ ] **Step 2: Write the BrandedPen weapon class**

Replace `src/zscript/branded_pen.zs` content with:
```
// Branded Pen — Slot 2 weapon. Functionally a vanilla Pistol, reskinned.
// Click sound on every shot; LOOM-logo placeholder sprite for now.

class BrandedPen : Pistol replaces Pistol
{
    Default
    {
        Weapon.SlotNumber 2;
        Weapon.AmmoType "Clip";
        Weapon.AmmoUse 1;
        Weapon.AmmoGive 20;
        Tag "Branded Pen";
        Inventory.PickupMessage "You got a Branded Pen.";
    }

    States
    {
    Ready:
        PENG A 1 A_WeaponReady;
        Loop;
    Deselect:
        PENG A 1 A_Lower;
        Loop;
    Select:
        PENG A 1 A_Raise;
        Loop;
    Fire:
        PENG A 4;
        PENG B 6 A_FirePistol;
        PENG B 4 A_ReFire;
        Goto Ready;
    Flash:
        PENG B 7 Bright A_Light1;
        Goto LightDone;
    Spawn:
        PENG A -1;
        Stop;
    }
}
```

A few things to note:
- `replaces Pistol` means anywhere a Pistol would spawn (in maps, drops), our BrandedPen replaces it. This is the fastest way to swap a vanilla weapon.
- `PENG A 1` references the sprite filename `PENGA0.png` (4-char base + frame letter A + the sprite-system implicit rotation 0 for non-rotating).
- The `Spawn` state with `PENG A -1` is the world-spawn state when the weapon is dropped on the ground. Players see the same sprite as the first-person view — fine for placeholder.
- `A_FirePistol` is the inherited Pistol firing action — single shot, hitscan damage. We'll reskin behavior beyond name/sprite in later phases if needed.

- [ ] **Step 3: Build and run**

```bash
./tools/build.sh && ./tools/play.sh CYC1_TEST
```

- [ ] **Step 4: Verify Branded Pen works**

In-game:
- Player should spawn with the Branded Pen visible (the placeholder PNG should appear at the bottom-center of the screen as the first-person weapon view)
- Press fire (default `Ctrl` or left mouse) — should fire and the PEN B sprite briefly appears
- The sprite may appear in the wrong position (offset) — that's expected, we'll fix offsets in Task 1.4 Step 5

If the weapon is invisible or the screen has no weapon at all:
- Sprite filename mismatch: verify `PENGA0.png` exists in PK3 — `unzip -l dist/loom-doom.pk3 | grep PENG`
- ZScript error: check the console at startup
- Pistol replacement not picked up: check that `Player.StartItem "BrandedPen"` in `LD_Player` is correct

- [ ] **Step 5: Set sprite offsets in SLADE**

Sprite offsets determine how the weapon image is positioned relative to the screen anchor. Without correct offsets, the weapon hangs off the bottom of the screen or floats too high.

Open `src/sprites/PENGA0.png` and `src/sprites/PENGB0.png` in SLADE (right-click → Open in tab). Use the Graphic Offsets tool:
- For a weapon HUD sprite, click `Offsets` → `Set Type → Weapon (Centered)` → Save.
- This stamps the offset into the PNG via grAb chunk.

Re-run `./tools/build.sh && ./tools/play.sh CYC1_TEST` and verify the weapon now appears at a normal weapon-view position.

(If you can't get SLADE to set offsets: the placeholder works visually even if offset, and we'll tune it during real-art passes in later phases. Don't block on perfect offsets here.)

- [ ] **Step 6: Commit**

```bash
git add src/sprites/PENGA0.png src/sprites/PENGB0.png src/zscript/branded_pen.zs tools/_gen_placeholder_sprites.py
git commit -m "Add Branded Pen weapon (replaces Pistol) with placeholder sprites"
```

---

## Task 1.5: Intern enemy (placeholder sprites + ZScript class)

**Files:**
- Create: `src/sprites/INTRA0.png` through `src/sprites/INTRG0.png` (placeholders, 7 frames)
- Create: `src/sprites/INTRH0.png`, `src/sprites/INTRI0.png` (death frames)
- Modify: `src/zscript/intern.zs`
- Modify: `tools/_gen_placeholder_sprites.py`
- Modify: `src/maps/CYC1_TEST.wad` (replace placeholder monsters with Intern)

- [ ] **Step 1: Extend the placeholder-sprite generator**

Edit `tools/_gen_placeholder_sprites.py`. Replace the bottom (`make("PENG…")` lines) with:
```python
make("PENGA0", 80, 60, "PEN A")
make("PENGB0", 80, 60, "PEN B!")

# Intern enemy — single-rotation placeholder. Frames A-D = walking, E-G = attack, H-I = death.
INT_BG = (40, 60, 40, 255)
INT_FG = (180, 200, 160, 255)
for letter, label in [
    ("A", "INT 1"), ("B", "INT 2"), ("C", "INT 3"), ("D", "INT 4"),
    ("E", "ATK 1"), ("F", "ATK 2"), ("G", "ATK 3"),
    ("H", "DIE 1"), ("I", "DIE 2"),
]:
    make(f"INTR{letter}0", 50, 80, label, bg=INT_BG, fg=INT_FG)

# Neural Pulse projectile — tiny purple square.
NPLS_BG = (40, 0, 60, 255)
NPLS_FG = (192, 132, 252, 255)
make("NPLSA0", 16, 16, "*", bg=NPLS_BG, fg=NPLS_FG)
```

Run it:
```bash
python3 tools/_gen_placeholder_sprites.py
ls src/sprites/
```
Expected: 9 INTR* + 2 PENG* + 1 NPLSA0 = 12 PNGs total.

- [ ] **Step 2: Write the Monster_Intern class**

Replace `src/zscript/intern.zs` content with:
```
// Intern — basic Cycle 1 enemy. Reskinned ZombieMan: same AI/stats,
// new sprite + sound + name.
//
// LOOM lore: throws "I'll send a follow-up" memo projectiles. For Phase 1
// the projectile behavior is inherited from ZombieMan (hitscan); we'll
// swap to actual memo-projectiles in a later cycle pass.

class Intern : ZombieMan replaces ZombieMan
{
    Default
    {
        Health 20;
        Speed 8;
        Tag "Intern";
        Obituary "%o was sidelined by an Intern.";
    }

    States
    {
    Spawn:
        INTR AB 10 A_Look;
        Loop;
    See:
        INTR AABBCCDD 4 A_Chase;
        Loop;
    Missile:
        INTR E 10 A_FaceTarget;
        INTR F 8 A_PosAttack;
        INTR E 8;
        Goto See;
    Pain:
        INTR G 3;
        INTR G 3 A_Pain;
        Goto See;
    Death:
        INTR H 5;
        INTR I 5 A_Scream;
        INTR I 5 A_NoBlocking;
        INTR I -1;
        Stop;
    Raise:
        INTR I 5;
        INTR H 5;
        Goto See;
    }
}
```

- [ ] **Step 3: Update the placeholder map to spawn Interns**

Open `src/maps/CYC1_TEST.wad` in SLADE. Open the map editor. In Things mode (`T`):
- Replace the 3 placeholder monster things with Thing Type **3004** (Trooper / ZombieMan — replaced via DECORATE-style swap by our Intern class)

Or equivalently, since we used `replaces ZombieMan` in the ZScript, anything you placed as a ZombieMan in Task 0.3 will already become an Intern at runtime. If your placed things were ZombieMen already, you don't need to re-edit the map.

If you placed different monsters (Imps, etc.), edit them now to be ZombieMen so our `replaces` rule kicks in.

Save the map (`Cmd+S`) and the WAD (`File → Save`).

- [ ] **Step 4: Build and run**

```bash
./tools/build.sh && ./tools/play.sh CYC1_TEST
```

- [ ] **Step 5: Verify Interns work**

In-game:
- 3 Interns should be visible in the room (placeholder green-rectangle sprites)
- They should walk toward the player
- They should fire at the player
- Player should be able to kill them with the Branded Pen — death animation plays (frames H, I)
- Intern obituary should show "X was sidelined by an Intern" if you die to one

If Interns appear as the original ZombieMan sprites:
- The `replaces ZombieMan` declaration was lost — re-check `intern.zs`
- The map's Thing IDs don't reference 3004 — re-check via SLADE

If Interns appear invisible or at wrong scale:
- Sprite offsets — open each INTR PNG in SLADE and `Offsets → Set Type → Monster`. Save.

Quit GZDoom.

- [ ] **Step 6: Commit**

```bash
git add src/zscript/intern.zs src/sprites/INTR*.png src/maps/CYC1_TEST.wad tools/_gen_placeholder_sprites.py
git commit -m "Add Intern enemy (replaces ZombieMan) with placeholder sprites; spawn 3 in test map"
```

---

## Task 1.6: Neural Pulse — always-on Slot 0 weapon

**Files:**
- Create: `src/KEYCONF.txt`
- Create: `src/sprites/NPLSA0.png` (already created in 1.5 Step 1; if missing, re-run sprite generator)
- Modify: `src/zscript/neural_pulse.zs`
- Modify: `src/MAPINFO.txt`
- Modify: `src/zscript/ld_player.zs`

The Neural Pulse is **not** a Weapon class. It's a body-mounted ability tied to a custom keybind. The pattern: a custom KEYCONF binding sends a `netevent` to ZScript; an EventHandler catches it and spawns a projectile from the player's position.

- [ ] **Step 1: Write KEYCONF.txt**

Create `src/KEYCONF.txt`:
```
addkeysection "LOOM-DOOM Controls" LoomDoom

addmenukey "Fire Neural Pulse" lddoom_neuralpulse

alias lddoom_neuralpulse "netevent NeuralPulseFire"

defaultbind F "lddoom_neuralpulse"
```

This:
- Adds a "LOOM-DOOM Controls" section to the controls menu
- Adds a "Fire Neural Pulse" entry that the player can rebind
- Registers a console alias `lddoom_neuralpulse` that issues a `netevent` named `NeuralPulseFire`
- Default-binds the F key to the alias

- [ ] **Step 2: Write the NeuralPulseProjectile actor + the handler**

Replace `src/zscript/neural_pulse.zs` content with:
```
// NeuralPulseProjectile — the projectile fired from the player's headband.
// Phase 1: low damage, slow recharge, single-target. Cycle evolution
// (more damage, glitch visuals, involuntary firing in C4) lives in
// later phases.

class NeuralPulseProjectile : Actor
{
    Default
    {
        Radius 6;
        Height 8;
        Speed 30;
        Damage 5;
        Projectile;
        +RANDOMIZE;
        RenderStyle "Add";
        Alpha 0.85;
        SeeSound "weapons/neuralpulse_fire";
    }

    States
    {
    Spawn:
        NPLS A 4 Bright;
        Loop;
    Death:
        NPLS A 4 Bright;
        Stop;
    }
}

// NeuralPulseHandler — listens for the "NeuralPulseFire" netevent
// (issued by KEYCONF alias when the player presses the bound key) and
// fires a NeuralPulseProjectile from the active player's position.
//
// Phase 1: no recharge meter, no charge state — just fires immediately.
// Phase 2 will add a CycleManager-aware charge timer.

class NeuralPulseHandler : EventHandler
{
    override void NetworkProcess(ConsoleEvent e)
    {
        if (e.Name != "NeuralPulseFire")
            return;

        if (e.Player < 0 || e.Player >= MAXPLAYERS)
            return;

        let p = players[e.Player];
        if (!p || !p.mo)
            return;

        // Spawn the projectile slightly in front of and above the player,
        // aimed where they're looking. CMF_AIMDIRECTION uses the player's
        // current view direction. Offsets are in map units.
        p.mo.A_SpawnProjectile("NeuralPulseProjectile", 32 /*z offset*/, 0 /*x offset*/, 0 /*angle*/, CMF_AIMDIRECTION);
    }
}
```

`EventHandler` (not `StaticEventHandler`) is correct here because we want per-map registration that can be serialized into save games — though we don't take advantage of that yet, it's the right base class for input event handlers per the GZDoom wiki.

- [ ] **Step 3: Register NeuralPulseHandler in MAPINFO**

Edit `src/MAPINFO.txt`. Append `NeuralPulseHandler` to the `AddEventHandlers` list:
```
gameinfo
{
    titlepage = "TITLEPIC"
    creditpage = "CREDIT"
    intermissionmusic = "$MUSIC_DM2INT"
    AddEventHandlers = "LD_CycleManager", "NeuralPulseHandler"
    playerclasses = "LD_Player"
}
```

- [ ] **Step 4: Build and run**

```bash
./tools/build.sh && ./tools/play.sh CYC1_TEST
```

- [ ] **Step 5: Verify Neural Pulse fires**

In-game:
- Press the **F** key (default Neural Pulse bind)
- A small purple projectile (placeholder sprite) should fly from the player's view direction
- It should travel until it hits a wall or an Intern
- Hitting an Intern should damage them (Intern has 20 HP, projectile does 5 damage — 4 hits to kill)

If F does nothing:
- Check the GZDoom console for KEYCONF parse errors
- Try opening the controls menu (`Esc → Customize Controls`) — there should be a "LOOM-DOOM Controls" section with "Fire Neural Pulse" entry. If missing, KEYCONF.txt didn't load (re-check filename / location).
- In console, manually run: `netevent NeuralPulseFire` — this should fire a pulse without going through KEYCONF. If THIS works but F doesn't, it's a KEYCONF binding issue.

If F triggers but no projectile appears:
- ZScript error in `neural_pulse.zs` — check console
- Sprite missing — `unzip -l dist/loom-doom.pk3 | grep NPLS`
- The sound `weapons/neuralpulse_fire` doesn't exist — that's a soft warning, not a fatal error; ignore it for Phase 1 (we'll add real sounds in later phases)

Quit GZDoom.

- [ ] **Step 6: Commit**

```bash
git add src/KEYCONF.txt src/zscript/neural_pulse.zs src/MAPINFO.txt
git commit -m "Add Neural Pulse Slot-0 weapon (KEYCONF bind + projectile + handler)"
```

---

## Task 1.7: LD_StatusBar — J0IN 0S terminal HUD

**Files:**
- Modify: `src/zscript/ld_statusbar.zs`
- Modify: `src/MAPINFO.txt`

- [ ] **Step 1: Write the LD_StatusBar class**

Replace `src/zscript/ld_statusbar.zs` content with:
```
// LD_StatusBar — J0IN 0S terminal HUD.
//
// Phase 1: clean (corruption=0) mode only — green-on-black text strip
// at the bottom of the screen showing unit ID, biometric (health), queue
// (ammo), active weapon, and zone (current map name) plus a cycle line.
//
// Cycle 2-4 corruption modes (text glitching, lying values, dissolving)
// will be added in Phase 3+ as the LD_CycleManager.corruptionLevel
// evolves.

class LD_StatusBar : BaseStatusBar
{
    HUDFont mHUDFont;

    override void Init()
    {
        Super.Init();
        // Use the default DOOM small font (HU_FONT) for now. Real font
        // lives in a later phase.
        Font fnt = Font.GetFont("HU_FONT");
        mHUDFont = HUDFont.Create(fnt, 1, Mono_CellLeft, 1, 1);
        SetSize(0, 320, 200);  // 0 = no fixed status bar height; HUD fills
    }

    override void Draw(int state, double TicFrac)
    {
        Super.Draw(state, TicFrac);

        if (state == HUD_None || state == HUD_AltHUD)
            return;

        BeginHUD(1.0, false, 320, 200);

        let mgr = LD_CycleManager.Get();
        int cycle      = mgr ? mgr.currentCycle    : 1;
        int corruption = mgr ? mgr.corruptionLevel : 0;

        let pmo = CPlayer.mo;
        if (!pmo) { return; }

        int health = pmo.Health;
        int ammo   = 0;
        String activeWeapon = "neural_pulse.exe";

        let curWeapon = CPlayer.ReadyWeapon;
        if (curWeapon)
        {
            activeWeapon = curWeapon.GetTag().MakeLower() .. ".exe";
            let amType   = curWeapon.AmmoType1;
            if (amType)
            {
                let am = pmo.FindInventory(amType);
                if (am) { ammo = am.Amount; }
            }
        }

        String zone = level.MapName.MakeLower();

        // Layout: two lines, anchored bottom-left.
        // Line 1: [unit#88-E] bio: 87% / queue: 24 / active: spam_filter.exe / zone: cyc1_test
        // Line 2: > cycle 8492 // simulation nominal // surveillance: ON
        String line1 = String.Format(
            "[unit#88-E]  bio: %d%%  /  queue: %d  /  active: %s  /  zone: %s",
            health, ammo, activeWeapon, zone
        );

        String simState = (corruption == 0) ? "nominal" :
                          (corruption == 1) ? "drift detected" :
                          (corruption == 2) ? "BREACH" : "DISSOLVING";
        String line2 = String.Format(
            "> cycle %d  //  simulation %s  //  surveillance: ON",
            8491 + cycle, simState
        );

        // Anchor at bottom-left, with small padding.
        DrawString(mHUDFont, line1, (4, -18), DI_SCREEN_LEFT_BOTTOM, Font.CR_GREEN);
        DrawString(mHUDFont, line2, (4, -8),  DI_SCREEN_LEFT_BOTTOM, Font.CR_DARKGREEN);
    }
}
```

A few notes on the implementation choices:
- We use `BeginHUD(1.0, false, 320, 200)` to declare the HUD's logical resolution as 320×200 (DOOM native). GZDoom auto-scales for higher resolutions.
- `DI_SCREEN_LEFT_BOTTOM` anchors coordinates relative to the screen's lower-left corner. Negative Y values offset upward from the bottom edge.
- Cycle number is displayed as `8491 + cycle` to match the spec's "Cycle 8492 / 8493 / etc." flavor text. (cycle=1 → "8492".)
- We're using DOOM's `HU_FONT` for now. The real terminal-green pixelated font goes in Phase 6 polish.

- [ ] **Step 2: Register LD_StatusBar in MAPINFO**

Edit `src/MAPINFO.txt`. Inside the `gameinfo` block, add `statusbarclass`:
```
gameinfo
{
    titlepage = "TITLEPIC"
    creditpage = "CREDIT"
    intermissionmusic = "$MUSIC_DM2INT"
    AddEventHandlers = "LD_CycleManager", "NeuralPulseHandler"
    playerclasses = "LD_Player"
    statusbarclass = "LD_StatusBar"
}
```

- [ ] **Step 3: Build and run**

```bash
./tools/build.sh && ./tools/play.sh CYC1_TEST
```

- [ ] **Step 4: Verify the HUD renders**

In-game:
- The bottom-left of the screen should show two lines of green text:
  - Line 1: `[unit#88-E]  bio: 100%  /  queue: 50  /  active: branded pen.exe  /  zone: cyc1_test`
  - Line 2: `> cycle 8492  //  simulation nominal  //  surveillance: ON`
- Take damage from an Intern: `bio:` value should drop in real-time
- Fire the Branded Pen: `queue:` value should drop with each shot

If the HUD doesn't appear:
- Check `statusbarclass` spelling in MAPINFO.txt (matches class name exactly)
- ZScript error in `ld_statusbar.zs` — check console
- The HUD might be obscured by the default DOOM status bar — try `+screenblocks 11` in console (full HUD overlay mode)

If text appears but looks wrong:
- Coordinate system off — adjust the `(4, -18)` and `(4, -8)` offsets
- Wrong font — try `Font.GetFont("CONFONT")` instead of `HU_FONT`

Quit GZDoom.

- [ ] **Step 5: Commit**

```bash
git add src/zscript/ld_statusbar.zs src/MAPINFO.txt
git commit -m "Add LD_StatusBar (J0IN 0S terminal HUD, clean mode for Phase 1)"
```

---

## Task 1.8: Phase 1 integration playtest + final commit

**Files:**
- Create: `docs/playtests/2026-04-27-phase-1-tracer-bullet.md`

- [ ] **Step 1: Run the full integration test**

```bash
./tools/build.sh && ./tools/play.sh CYC1_TEST
```

Walk through this **playtest checklist** and verify each item works:

| # | Behavior | Expected |
|---|---|---|
| 1 | Game launches | GZDoom window opens, no fatal errors in console |
| 2 | CycleManager registers | Console shows `LD_CycleManager: registered, cycle=1, corruption=0` |
| 3 | Map loads | Player spawns inside CYC1_TEST room |
| 4 | Player class | Press `~` console: `printinv` shows player has Clip 50 + BrandedPen |
| 5 | Branded Pen visible | Weapon sprite (placeholder PEN) shows at bottom-center |
| 6 | Branded Pen fires | Click fire (Ctrl/LMB) — bullet trace fires, Clip count drops |
| 7 | Interns spawn | 3 visible in room, walk-toward / fire-at player |
| 8 | Interns die | 1-2 Branded Pen shots kills one |
| 9 | Neural Pulse fires | Press `F` — purple projectile flies from player view |
| 10 | Neural Pulse damages | Pulse hitting an Intern reduces their HP |
| 11 | Custom HUD renders | Bottom-left green text shows unit/bio/queue/active/zone + cycle line |
| 12 | HUD updates | Take damage → bio drops; fire pen → queue drops |
| 13 | All 3 Interns killable | All 3 die, no errors after combat ends |

If ALL 13 pass — Phase 1 is complete.

If any fail — go back to the relevant Task and fix.

- [ ] **Step 2: Save playtest result**

Create `docs/playtests/2026-04-27-phase-1-tracer-bullet.md` with this content:
```markdown
# Phase 1 Tracer Bullet — Playtest Log

**Date:** 2026-04-27
**Tester:** [your name / "agent" / "claude"]
**Build:** loom-doom.pk3 commit [paste current git short hash]

## Checklist

| # | Behavior | Result |
|---|---|---|
| 1 | Game launches without fatal errors | PASS / FAIL |
| 2 | CycleManager registers (console line visible) | PASS / FAIL |
| 3 | Map loads | PASS / FAIL |
| 4 | Player has Clip 50 + BrandedPen | PASS / FAIL |
| 5 | Branded Pen sprite visible | PASS / FAIL |
| 6 | Branded Pen fires + Clip drops | PASS / FAIL |
| 7 | 3 Interns spawn | PASS / FAIL |
| 8 | Interns die from Branded Pen | PASS / FAIL |
| 9 | Neural Pulse fires from F key | PASS / FAIL |
| 10 | Neural Pulse damages Interns | PASS / FAIL |
| 11 | Custom HUD renders bottom-left | PASS / FAIL |
| 12 | HUD updates in real time | PASS / FAIL |
| 13 | All 3 Interns killable | PASS / FAIL |

## Notes / known issues

[Anything that worked weirdly, sprite offset issues, etc.]

## Phase 1 Definition of Done

[X] All 13 checklist items pass — tracer bullet complete.
```

Replace `PASS / FAIL` with actual results from the playtest. Keep this committed as historical record.

- [ ] **Step 3: Commit playtest log**

```bash
mkdir -p docs/playtests
git add docs/playtests/2026-04-27-phase-1-tracer-bullet.md
git commit -m "Add Phase 1 tracer-bullet playtest log (DoD met)"
```

- [ ] **Step 4: Verify branch state**

```bash
git log --oneline | head -20
git status
```
Expected: clean tree, ~14-16 commits since the spec commit, all Phase-0/Phase-1 task commits visible.

---

# Self-review

Spec coverage check (against `docs/superpowers/specs/2026-04-27-loom-doom-design.md`):

| Spec section | Phase 0+1 coverage |
|---|---|
| §3.1 Identity (DEFECTIVE UNIT #88-E) | ✅ LD_Player class established; HUD shows `[unit#88-E]` |
| §3.4 C-reveal | ❌ Out of scope for Phase 1 — Phase 5 work |
| §4 Cycle structure | ✅ partial — cycle 1 active; LD_CycleManager singleton in place |
| §5.1 Weapon roster | ✅ partial — Neural Pulse (Slot 0) + Branded Pen (Slot 2) implemented |
| §5.2 Enemy taxonomy | ✅ partial — Intern (Cycle 1 basic) implemented |
| §6.1 HUD J0IN 0S terminal | ✅ clean (corruption=0) mode; corruption modes deferred to Phase 3+ |
| §6.2 Audio | ❌ Phase 6 — placeholder only, no real audio |
| §6.3 Maps | ✅ partial — 1 placeholder map; rest in Phase 2+ |
| §7 Architecture | ✅ Repo layout, build pipeline, validate scaffold all done per spec |
| §8 Phased build plan, Phase 0 | ✅ DoD met (Task 0.6 Step 4) |
| §8 Phased build plan, Phase 1 | ✅ DoD met (Task 1.8 Step 1) |
| §9 Risk inventory | n/a — meta |

Placeholder scan: searched the plan for "TBD" / "TODO" / "implement later" — none found. All steps have actual content. ✅

Type consistency: class names verified across tasks:
- `LD_CycleManager` — Task 1.2 + lookup in 1.7 (consistent)
- `LD_Player` — Task 1.3 + reference in MAPINFO (consistent)
- `BrandedPen` — Task 1.4 + reference in `LD_Player.StartItem` in 1.3 (consistent — note: 1.3 references it before it exists in 1.4, this is fine because GZDoom resolves at runtime)
- `Intern` — Task 1.5, no other references needed (it's auto-spawned via `replaces ZombieMan`)
- `NeuralPulseProjectile` + `NeuralPulseHandler` — Task 1.6, handler registered in MAPINFO (consistent)
- `LD_StatusBar` — Task 1.7, registered in MAPINFO statusbarclass (consistent)

All type and method references match. ✅

---

# Done condition for this plan

Phase 1 complete when Task 1.8 Step 1's 13-item playtest checklist all pass. At that point, `loom-doom.pk3` is publishable as a "tracer bullet" demo and the project is unblocked for Phase 2 (the full Cycle 1 vertical slice).

The next plan, when this one is done: `2026-XX-XX-loom-doom-phase-2.md` covering Cycle 1 maps + remaining Cycle 1 weapons + remaining Cycle 1 enemies + the HR Manager boss.
