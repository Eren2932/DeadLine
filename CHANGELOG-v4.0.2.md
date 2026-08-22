# DeadLine v4.0.2 — the black screen, the cause, and the gate

## What actually happened in v4.0.0

`scripts/Game.gd` declared the same member **twice**:

```
line 48:  var _beams: Array = []      # new: searchlight cones (Polygon2D)
line 61:  var _beams: Array = []      # old: laser beam pool  (Line2D)
```

GDScript refuses to compile a class with a repeated member. `Game.gd` therefore
never compiled, `Game.tscn` loaded as a plain `Node2D` with no script, and the
screen showed nothing but `default_clear_color` — the exact uniform
`rgb(12, 10, 15)` on the screenshot. No player, no HUD, no error the player
could see. Nothing was wrong with the belt, the art or the export.

Two of my own tools ran green on that build: `check.py` and `lint.py` never
looked for duplicate declarations, and the pipeline never ran the game — it only
packaged it. A broken script is not a broken export, so CI was happy.

## Fixes

**1. The bug.** The searchlight list is gone from `Game.gd`; it lives in
`EnvCity.gd` as `beams`, in its own class. `_beams` in `Game.gd` is the laser
pool again, declared once.

**2. `lint.py` now fails the build on any duplicated `var` / `const` / `func` /
`signal`, and on any `Autoload.member` that the autoload does not declare.**
Verified against the broken file:

```
! Game.gd: var '_beams' declared 2 times (lines 44, 58)
  - GDScript will not compile this class
```

**3. CI runs the game before it packages it.** New blocking step boots
`Main.tscn` and `Game.tscn` headless for 240 frames each and fails on
`SCRIPT ERROR`, `Parse Error`, `Failed to instantiate`, `Invalid call`,
`already exists in`, … Logs are uploaded as the `boot-logs` artifact either way.
This build could not have shipped with that step in place.

## Architecture, so scenery can never take the game down again

**The new autoload is gone.** v4.0.0 introduced `World` as an autoload. An
autoload that fails takes down every script that mentions it — that is the whole
game for one background. All of the geometry now lives in `GameData.gd`, the
autoload that has worked since v1: `GameData.BELT_TOP`, `belt_h()`,
`depth_of()`, `scale_at()`, `move_vec()`, `spawn_y()`, `cam_y()`.
`GameData.BELT_ENABLED = false` still returns the flat v3 band exactly.

**The city is loaded at runtime, and last.** `scripts/EnvCity.gd` has no
`class_name` and is not preloaded. `Game.gd` reaches it with
`load()` + null check, from a `call_deferred` at the very end of `_ready` —
after the player, the HUD and the camera already exist. Three guards: script
null → flat v3 city; no `build()` → flat v3 city; zero bands found → flat v3
city. A runtime error inside `build()` kills that deferred call and nothing else.

**The sky is a `ColorRect`, painted first.** Themed, no file, nothing to import,
nothing to miss. The arena is a night sky from frame one even if every PNG were
missing. A black screen has no path left.

## Space (unchanged from the v4 design)

| | |
|---|---|
| `HORIZON_Y` -34 | fence foot |
| `BELT_TOP` -22 | **depth 0** — far curb, drawn |
| `BELT_BOT` 72 | **depth 1** — near curb, drawn |
| `BELT_H` 94 | was 78 |

Scale 0.88 → 1.10 with depth, depth movement at 0.62 speed, camera travels 45%
of your depth, spawns biased to the far side (0.34 ± 0.46).

## Art

`props_near` grew 150 → 200 px, so the near apron reaches the bottom of the
frame on any phone aspect ratio instead of relying on the camera to hide it.
All four themes rebaked: 223 KB for the whole environment.

## Still not verified

There is no Godot binary in my sandbox and no network to fetch one, so I have
never compiled this project — that is exactly the hole that let v4.0.0 through.
The CI smoke test is the fix for that: from this build on, the pipeline compiles
and boots the game, and a black-screen APK cannot be produced.
