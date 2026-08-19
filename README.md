# DEAD LINE: ZOMBIE FRONTIER

Endless pixel-art zombie shooter / roguelite defence for Android, built in Godot 4.
Fullscreen **16:9 landscape**, side view, hand-generated pixel art, blood, 21 weapons,
6 buildable structures, 28 roguelite upgrades, elites, bosses, wave events and airdrops.

## Run it

1. Open the folder in Godot 4 (`project.godot`), let it import once.
2. Press F5. Main scene is `Main.tscn` (the menu). The run itself is `Game.tscn`.

## Controls

The game is landscape only (`640x360` base viewport, `expand` stretch, sensor
landscape on the phone) - hold the phone sideways and the whole horde is on
screen long before it can reach you.

Touch: drag anywhere on the stick side (bottom-left by default, switchable in
settings) to move - the stick spawns under your thumb. Buttons on the other side:
DASH, NADE, GUN (switch), RLD (reload), BLD (build). Aim is automatic; turn
AUTO FIRE off in settings to get a FIRE button.

Keyboard (editor): WASD move, E dash, Q swap, G grenade, R reload, B build,
1-4 weapon slots, SPACE start wave, ESC pause, CTRL hold fire.

## Structure

    Main.tscn / Game.tscn      the only two scenes - everything else is code
    scripts/Menu.gd            start screen, loadout, settings, how-to-play
    scripts/Game.gd            world, camera, waves, spawning, combat helpers
    scripts/Player.gd          movement, all weapon kinds, upgrades, damage
    scripts/Zombie.gd          AI, armour, elites, status effects, gore
    scripts/Bullet.gd          projectiles with swept (continuous) collision
    scripts/Structure.gd       walls, sandbags, spikes, mines, turrets, tesla
    scripts/Ally.gd            pets (guard dog, war tiger)
    scripts/HUD.gd             stats, boss bar, shop, build bar, pause, game over
    scripts/TouchPad.gd        real multi-touch stick + action buttons
    scripts/Settings.gd        options (audio, graphics, controls, difficulty)
    scripts/Music.gd           soundtrack with crossfades
    scripts/Audio.gd           pooled SFX player
    scripts/Fx.gd              blood, gibs, decals, sparks, explosions, numbers
    scripts/GameData.gd        every balance number in one place
    scripts/SaveManager.gd     records, Cores, unlocks, loadout, settings
    scripts/UI.gd              shared widget factory
    scripts/ArtManifest.gd     sprite table baked in as a constant

## Regenerating assets

    python3 tools/build.py     pixel art  -> art/    (+ scripts/ArtManifest.gd)
    python3 tools/sfx.py       sound fx   -> sfx/
    python3 tools/music.py     soundtrack -> music/  (needs numpy, scipy, ffmpeg)
    python3 tools/check.py     static check of the whole GDScript layer
    python3 tools/lint.py      deep lint: unknown identifiers, type inference

`tools/check.py` verifies cross-script calls, sound/art/settings keys, resource
paths and data tables. `tools/lint.py` walks every function scope and flags
unknown identifiers, forgotten variables, unbalanced brackets, space
indentation and any `:=` type inference that the engine could refuse to infer.
Both must print `ERRORS: 0` before you ship a build - the GitHub workflow runs
them before exporting. (Do NOT gate the build on `godot --check-only --script`:
that mode compiles a script in isolation, without the project autoloads, so it
reports false `Identifier not found: Art/Audio/Save/...` errors.)

## Android APK from GitHub Actions

1. Create a repository on GitHub (private is fine) and push this folder to the
   `main` branch, keeping `.github/workflows/android.yml` in place.
2. Open the repo -> **Actions** tab -> if asked, press
   *I understand my workflows, go ahead and enable them*.
3. The push already started **Build Android APK**. To start it by hand:
   Actions -> *Build Android APK* -> **Run workflow** (you can type another
   engine tag in the `godot_version` field there).
4. Wait for the green check (~8-12 min; most of it is downloading Godot).
5. Click the finished run -> **Artifacts** -> `DeadLine-debug-apk` -> it
   downloads a zip -> inside is `DeadLine.apk`.
6. Copy the APK to the phone, allow *install from unknown sources* and install.

The engine version comes from the `GODOT_VERSION` file (currently
`4.4.1-stable`, which matches `config/features` in `project.godot`), with an
automatic fallback chain if that tag is missing on the release server. The APK is a debug build: signed with a throwaway keystore,
installable on any device, not suitable for Google Play (for Play you need a
release keystore + `--export-release`).

If a run goes red, open it and read the failing step: `Static checks` means the
linters found something, `Validate every GDScript parses` means a script has a
syntax error, `Resolve & download Godot` means the version tag does not exist.

## What is new in 1.2.1

- **CI unpack step rewritten.** The old step took the first `*.zip` in the repo
  root, assumed the project was directly inside it and exited 1 when it was
  not (that is the `Error: Process completed with exit code 1` right after
  `Unpacking ...`). It now unpacks *every* zip in the repo, expands nested
  archives (a repo zip that contains the project zip), picks the project with
  the highest `config/version` - so an old zip left in the repo can never win -
  and copies it into the root with `cp -a` (no more non-portable `cp -n`
  warning).
- `GODOT_VERSION` pinned to `4.4.1-stable` to match `config/features`.

## What is new in 1.2

- **Landscape 16:9.** Base viewport is now `640x360` with `canvas_items` /
  `expand` stretch, so the picture fills any phone from 16:9 to 21:9 without
  black bars and without cropping the HUD. The camera looks ahead in the
  direction the survivor faces.
- **No more shooting at nothing.** Auto-aim range is clamped to what is
  actually visible (`min(weapon range, half screen + 10 px)`), so the survivor
  never opens fire at a zombie that is off screen.
- **Weapon holding rewritten.** The gun now hangs on a real hand pivot
  (`arm` node): it rotates to the aim angle, mirrors with `scale.y = -1` when
  facing left instead of flipping upside down, gets recoil kickback, walk bob
  and a melee swing arc. The muzzle flash and the bullet spawn point sit
  exactly on the barrel tip.
- **Real economy.** Every gun has a magazine *and* a reserve pool
  (`reserve`, `ammo_price`, `unlock`). You buy the gun once and then pay for
  ammo; you only pay for the rounds that are missing. Guns unlock by wave, so
  wave-1 cash can never buy a railgun. Weapons can be sold back for a refund
  (`SELL_REFUND`), the last shooting weapon can never be dropped, and an empty
  gun auto-swaps to one that can still fire.
- **New shop.** Two-column scrollable market with price, DPS, unlock wave,
  owned/locked state, BUY / AMMO / SELL per line, plus REFILL ALL, MEDKIT
  (priced by missing HP) and GRENADE buttons. Every deny shows a reason banner
  instead of silently doing nothing.
- **Location fixed.** The background bands no longer drift, slide or "drive
  away": each band is sized to the widest possible view, wrapped around the
  camera every frame with `fposmod`, and re-anchored to the camera instead of
  to the level.
- **HUD relaid out for landscape:** compact top status bar (HP, XP, score,
  wave, cash, kills, combo, event), centred weapon strip at the bottom,
  thumbstick bottom-left, action cluster bottom-right, free middle. HUD taps
  can no longer be stolen by the stick.
- **New soundtrack** (menu / battle / boss), regenerated from scratch with
  `tools/music.py`.

## What is new in 1.1.1

- Fixed the `HUD.gd` parse error `Cannot infer the type of "id" variable`:
  every `:=` inference on a value coming from a dynamically typed object was
  removed, and cross-object references (`game`, `player`, `hud`, `fx`, bullets,
  structures, allies) are now dynamically typed on purpose, so the analyser can
  never refuse a member it cannot see through a `Node` type hint.
- Added `tools/lint.py` (scope-aware linter) and wired both linters into the
  GitHub workflow.
- The workflow can build straight from the project zip committed to the repo
  root (GitHub web upload allows only 100 files per commit).

## What is new in 1.1

- Damage registration fixed: bullets use swept collision (no tunnelling),
  hitscan sorts targets by distance, melee sweeps an arc, headshots do x1.5.
- Start screen: menu, animated background, loadout ("equip your gear"),
  settings, how to play, profile records.
- Soundtrack: procedural industrial metal - menu / battle / boss with crossfade.
- Full settings: volumes, camera zoom, gore level, shake, quality, damage
  numbers, FPS, stick side, auto fire, vibration, three difficulties.
- More scale: Riot Cop and Hazmat zombies, elite mutations, seven wave events,
  airdrops, M249 SAW, Gauss Rifle, SWAT and Medic suits, Cores meta progression.
- Better feel: combo multiplier, score, hitstop, shockwaves, shell casings,
  acid pools, off-screen threat arrows, boss health bar, vibration.
- Real multi-touch controls (hold the stick and press buttons at the same time).
