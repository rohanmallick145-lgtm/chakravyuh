# Abhimanyu's Chakravyuha

A complete, runnable **Godot 4.7** 2D physics platformer. You play **Arjun**, a rolling
RigidBody2D sphere, fighting through seven mythic rounds towards the heart of the
Chakravyuha. Every celestial ring you claim makes you stronger in the air — and
permanently burns away part of your maximum life.

Built with GDScript only. The backgrounds are a layered, screen-space painted sky
(gradient, celestial body, parallax ridges, per-round landmarks, birds and motes),
platforms and characters are drawn procedurally, and every sound is generated as a WAV
by `tools/make_audio.gd` — so there are no missing-asset dependencies.

---

## Run it

1. Open the project folder in **Godot 4.7** (Project Manager → Import → `project.godot`).
2. Press **F5**.
3. The first round opens with a story popup. Press **Space** (or click) to continue.

CLI:

```powershell
godot --path "C:\Users\uchih\OneDrive\Documents\abhimayu's-chakravyu"
```

The project uses Forward+ / D3D12 on Windows. It also boots cleanly headless (tested on
Godot 4.7.2 stable).

---

## Controls

| Input | Action |
| --- | --- |
| **A / D** (or ← →) | Roll left / right (torque-driven, momentum-based) |
| **W** (or ↑ / Space) | Jump. Keep holding in the air to **levitate** |
| **S** (or ↓) | Dive / fast-fall |
| **Space / Enter / Left click** | Advance dialogue, start, restart after victory |

No InputMap setup is required: the player reads physical keys directly, so it works in
any keyboard layout.

---

## The core loop: rings are power *and* poison

Each of the seven rounds hides one celestial ring. Claiming it:

- Permanently lowers **max HP by 1/12** of the base (8.33%).
- Extends **levitation duration** by `+0.35 s`.
- Improves the passive glide (`float_gravity_scale` goes 0.82 → 0.40).

| Rings claimed | Max HP | Levitation |
| --- | --- | --- |
| 0 | 100 | 0.50 s |
| 1 | 91.7 | 0.85 s |
| 3 | 75 | 1.55 s |
| 6 | **50** | 2.60 s |
| 7 | 41.7 | 2.95 s |

That is why **Round 7 begins at exactly 50 / 50 HP** — six rings have already been paid
for. The HUD draws the missing cap as a scarred, cracked section of the health bar.

Rings persist through death. The exit gate stays locked until the round's ring is
claimed; Round 7's gate only opens after the boss dies and drops the seventh ring.

---

## The seven rounds

1. **Drona's Trial** — intro to rolling, spikes, a patroller and an archer.
2. **The Flames of Khandava** — fire geysers, a moving platform, flaming archers.
3. **Shakuni's Traps** — collapsing stone bridges and spike pits.
4. **Bhishma's Arrows** — dense arrow volleys across broken ground.
5. **The Turning Wheel** — chakram spinners, moving platforms, tighter jumps.
6. **Karna's Solar Duel** — triple-burst fire archers and a golden sky.
7. **The Chakravyuha** — pure-black arena (CanvasModulate + PointLight2D), braziers,
   and the living-wheel boss.

**Boss:** the wheel hovers and cycles three attacks (radial ring, aimed burst, spiral).
Touching it normally hurts you. Hit it **while levitating** or at high rolling speed to
shatter one of its six armour pips. Six hits break it; the seventh ring drops.

**Combat:** there is no attack button. Levitation and high-speed contact are the weapon.

---

## Story presentation

- **Round intro:** full pause-screen popup with the speaker portrait and typewriter text.
  Text lives in `scripts/core/dialogue_data.gd` under `round_dialogues`.
- **Checkpoint:** mid-level obelisk triggers a **cinematic slow-motion** popup
  (`Engine.time_scale = 0.22`). Each checkpoint is remembered, so its story never
  repeats after a respawn.
- **Death:** slow-motion "Arjun has fallen" banner, then respawn at the last checkpoint
  (or the round start).
- **Victory:** seventh ring + final gate → closing dialogue → victory screen.

### Portraits

`scripts/ui/dialogue_popup.gd` maps each speaker to a portrait:

- **Elders** — Dronacharya, Shakuni, Bhishma, Vidura, Dhritarashtra — use the supplied
  painting at `assets/portraits/elders.png`.
- **Krishna** — Round 7, the final-ring line and the victory speech — uses
  `assets/portraits/krishna.png`.
- **Karna** — his Round 6 intro — uses `assets/portraits/karna.png`.
- **Younger warriors** — Arjun and Abhimanyu — use
  `assets/portraits/warrior.svg`.

Add a name to `ELDER_SPEAKERS`, `KRISHNA_SPEAKERS`, `KARNA_SPEAKERS` or
`WARRIOR_SPEAKERS` to route a new character. The `DialogueLayer` also still exposes an
`elder_portrait` export and accepts a per-call `{"portrait": texture}` override.

---

## Art & audio

- **Painted sky:** `scripts/fx/sky_background.gd` draws a full-screen layered backdrop on
  its own `CanvasLayer` at layer `-10`: a banded gradient, a haloed sun/moon, drifting
  clouds, parallax ridges, round-specific landmarks (gurukul, burning trees, dice tower,
  broken chariot, wheel shrine, sun chariot), stars, birds and drifting motes. The layer
  sits behind the world, so Round 7's `CanvasModulate` blacks out the arena without
  erasing the HUD or double-darkening the sky. The backdrop simply paints pure black in
  Round 7, leaving the braziers and PointLight2D to reveal the arena.
- **Atmosphere:** `shaders/vignette.gdshader` adds a warm vignette and a whisper of
  canvas grain on `CanvasLayer` 5, above the world and below the HUD.
- **Enemies:** patrollers and archers draw the supplied painting at
  `assets/enemies/asura.png` through `scripts/enemies/enemy_art.gd`. Its white
  background has been cut out to alpha, so the sprite sits cleanly over the painted
  world; archers keep their drawn aim-bow and flame casters stay warm-tinted.
- **Sound:** all 18 effects and the 16-second seamless drone live in `assets/audio/`.
  `Sfx` (`scripts/core/sfx.gd`) loads them once, plays one-shots through a 12-player
  round-robin pool on the `SFX` bus, and loops the ambience on the `Music` bus. The drone
  is retuned per round (lower and louder in Round 7). Regenerate the set with:

  ```powershell
  godot --headless --path . --script res://tools/make_audio.gd
  ```

  The bus layout is `default_bus_layout.tres`.

---

## Project layout

```
project.godot                  autoloads: GameState, Sfx; main scene: scenes/game.tscn
default_bus_layout.tres        Master / Music / SFX buses
scenes/
  game.tscn                    shell: level container, vignette, HUD, dialogue, fade
  levels/level_01..07.tscn     tiny scenes; all share scripts/levels/level.gd
  player/arjun.tscn            RigidBody2D + visual + orbit halo + camera
  objects/                     ring, exit portal, checkpoint
  ui/                          HUD and dialogue popup
scripts/
  core/
    game_state.gd              autoload: rings, HP cap, levitation, persistence
    sfx.gd                     autoload: one-shot pool + looping ambience
    game_manager.gd            round loading, death/respawn, transitions, victory
    dialogue_data.gd           all story text (round_dialogues, checkpoints, victory)
    level_db.gd                all 7 level layouts and palettes
  levels/level.gd              builds geometry, hazards, enemies, pickups from LevelDB
  player/arjun.gd              movement, jump/levitate, damage, death
  enemies/                     patroller, archer, projectile, boss, enemy_art.gd
  objects/                     hazard, spinner, moving/crumble platform, ring, etc.
  fx/                          painted sky, bursts, radial light factory, brazier
  ui/                          hud.gd, dialogue_popup.gd
shaders/vignette.gdshader      warm vignette + canvas grain
assets/portraits/              elders.png, krishna.png, karna.png, warrior.svg
assets/enemies/asura.png       supplied enemy painting (cut out to alpha)
assets/audio/                  generated SFX + ambient WAVs
tools/make_audio.gd            regenerates every WAV
tests/smoke_test.tscn|gd       headless end-to-end smoke test
```

### Tuning

- **HP cost per ring:** `HP_LOSS_PER_RING` in `game_state.gd`.
- **Levitation:** `BASE_LEVITATION`, `LEVITATION_PER_RING`, `float_gravity_scale()`.
- **Movement feel:** exported values at the top of `player/arjun.gd`
  (`max_speed`, `jump_velocity`, `roll_torque`, `levitation_rise`, …).
- **Levels:** `scripts/core/level_db.gd`. Platform `Rect2`s use **top-left**
  coordinates; enemy `"pos"` is the body centre; archer/checkpoint/exit `"pos"` is the
  ground point. Add a round by adding a dictionary and a `level_08.tscn` clone.

---

## Testing

Headless end-to-end test (boots the real game scene and drives it):

```powershell
godot --headless --path "C:\Users\uchih\OneDrive\Documents\abhimayu's-chakravyu" res://tests/smoke_test.tscn
```

It verifies: round load, intro popup, audio streams, painted sky, painted enemy sprite,
elder/younger/Krishna portraits, rolling, checkpoint
slow-motion, ring pickup and HP-cap reduction, gate unlocking, death/respawn with ring
persistence, round transition, the 50/50 HP start of Round 7, dark mode, boss
spawn/defeat, seventh ring, and the victory screen. Exit code is 0 when everything
passes.

To start directly in a later round while tuning, change `GameState.begin_round(index)`
in `game_manager.gd` or temporarily claim rounds in `GameState`.

---

## Notes

- The seven level scenes are intentionally minimal; geometry is generated at runtime from
  `LevelDB`, so editing one dictionary reskins/rearranges the whole round.
- The dark arena uses `CanvasModulate` (pure black) plus `PointLight2D` and additive
  glow. The player carries a small light that only switches on in dark mode.
- The game is keyboard-first; mouse click only advances dialogue.
- The supplied elder painting is wired to the elder speakers rather than treated as a
  generic placeholder, so Dronacharya, Shakuni, Bhishma and Krishna all speak from it.
