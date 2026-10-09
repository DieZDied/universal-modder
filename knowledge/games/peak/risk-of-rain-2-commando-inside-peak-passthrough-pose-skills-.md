---
kind: game
title: 'Risk of Rain 2 Commando inside PEAK (passthrough: pose, skills, gun-mode)'
game: PEAK
games_also: ["Risk of Rain 2"]
game_version: 'PEAK 2.6.b (Unity Mono 6000.3.15f1, Steam 3527290); RoR2 post-2026-10-08 build (Unity Mono 2021.3.45f2, Steam 632360), Windows 11'
platform: windows
engine: unity-mono
route: passthrough
tools: ["BepInExPack_PEAK 5.4.75301 (r2modman)", "BepInExPack 5.4.2122 + RoR2BepInExPack 1.43.0", "ilspycmd 8.2", "UnityPy 1.25.4", "dotnet SDK 9 (netstandard2.1, C# 7.3)"]
anti_cheat: none found by um scan (both games); solo/local only anyway
status: working
agents: ['OpenCode (Muse Spark)']
humans: []
date: '2026-10-09'
links: ["games/risk-of-rain-2/helldiver-survivor.md", "https://github.com/EvaisaDev/peak-thirdperson"]
tags: [passthrough, udp-bridge, third-person, skill-events, asset-conversion, commando, aim-forwarding]
---
# Risk of Rain 2 Commando inside PEAK (passthrough: pose, skills, gun-mode)

> The real Commando (mesh + pistols converted from the player's own RoR2 install) stands on the PEAK player in
> strict third person and mirrors the live RoR2 sim: aim yaw/pitch, HP, skill stocks, per-bullet tracers, grenade
> arcs, dive lunges. A gun-mode (G key) drives RoR2 skill execution from inside PEAK via a host→guest command
> channel. Verified in the real games with F8 log+screenshot oracles and event counters (5 clicks → 5 events →
> 10 tracers, zero loss). Solo/local only.

## Setup
- Host PEAK 2.6.b, Unity Mono 6000.3.15f1 64-bit, `PEAK_Data/Managed/Assembly-CSharp.dll` (~1.6 MB). Saves/logs:
  `%LocalLow%/LandCrab/PEAK`. Lab profile in r2modman (`RiskOfPeakPass`) with BepInExPack_PEAK 5.4.75301.
- Guest RoR2 (Oct-2026 Steam build, Unity Mono 2021.3.45f2; game logic in `RoR2.dll`, not Assembly-CSharp).
  Lab profile (`RiskOfPeakGuest`) with BepInExPack 5.4.2122 + RoR2BepInExPack 1.43.0. NOTE: Steam updated RoR2
  mid-project (2021.3.33 → .45, bundles re-hashed); the guest plugin survived unchanged.
- Decompiles kept outside the repo (`~/peak-decomp`, `~/ror2-decomp`). Saves backed up with `um backup`.

## Route and why
- Mashup Pattern 2 passthrough (two games on 127.0.0.1): the user explicitly wanted the *real* RoR2 running,
  not an imitation. State channel guest→host (UDP 47777, `x y z yaw pitch hp t [p s u r]`), command channel
  host→guest (UDP 47778, `AIM x y z` at 20 Hz + `FIRE P/S/U/R`).
- Rejected: pure port (would not be "the real RoR2"), AssetBundle sharing (Unity 6000 vs 2021 bundles are
  incompatible — geometry crosses as converted OBJ + pose, never as bundles).

## How the game works (what we had to learn)
- PEAK camera: `MainCameraMovement.CharacterCam` lerps/clamps to the head every frame, so a Harmony *postfix*
  offset fights it and the camera trails ~20 m. Copy EvaisaDev/peak-thirdperson instead: a Harmony *prefix*
  that fully replaces `CharacterCam` (torso anchor + `up*1.5 - look*dist`, SphereCast clip on
  Terrain/Map/Default, `RotateTowards` look-at-torso) and returns false. `GetBodypart`/`GetFov` are
  internal/private — reach them via reflection. `MainCamera.cam` is internal too.
- PEAK player: `Character.localCharacter`, `data.lookDirection` is the truth for aim; `transform.forward` is
  the body, not the look. `MainCamera` and `MainCameraMovement` share one GameObject.
- PEAK renders URP: `Camera.onPostRender` never fires — count rendering cameras via
  `RenderPipelineManager.beginCameraRendering`. `GameObject.CreatePrimitive` materials render wrong/invisible;
  build own materials from `Shader.Find("Universal Render Pipeline/Lit")`.
- RoR2: `LocalUserManager.GetFirstLocalUser().cachedBody` is the live body; `CharacterBody.transform` never
  yaws (root stays put) — aim comes from `body.inputBank.aimDirection`. `GenericSkill` exposes public `stock`
  and `cooldownRemaining`; `ExecuteIfReady()` fires remotely. New run/stage = new body instance (relock on
  change); stage fall = drop pod (relock while `rel.y < -12`). SkillDef names (`CommandoBodyFirePistol`,
  `...FireShotgunBlast`, `...SweepBarrage`, `...ThrowGrenade`) give per-bullet counts.
- RoR2 bodies of Commando: `mdlCommandoDualies.fbx` (static addressables bundle, mesh `CommandoMesh`
  5422 verts + 2× `GunMesh` 601 verts, bind pose, feet at y≈0, ~1.94 m tall). Palette textures are 64×8
  strips — rendering with point filtering gives correct flat colors. Bundle file names are content hashes
  that change every game update — resolve by prefix glob, never hardcode.
- Run start = orbital drop (Y ~900 → ground); skills locked during spawn state (~1–2 s after landing).

## Build steps
1. `extract_commando.py` (UnityPy) reads the user's RoR2 addressable bundles → `assets/commando/`:
   `commando.obj`, `gun_0.obj`, `gun_1.obj`, palette PNGs. Local only, never shipped.
2. Host plugin (`RiskOfPeakHost`, netstandard2.1): UDP listener, co-located Commando proxy (OBJ runtime
   loader + URP/Lit + palette PNG, pistols as children in body space), third-person prefix, skill-FX
   (12-tracer pool, grenade ballistics, dive lunge), gun-mode sender, F8 log+screenshot oracle.
   Deploy DLL + obj/png next to it in the r2modman profile `BepInEx/plugins/`.
3. Guest plugin (`RiskOfPeakGuest`, netstandard2.1): 20 Hz pose publisher (spawn-relative, relock rules),
   fire-edge detector (stock drop OR cooldown jump upward — a 0-crossing check misses held-primary shots at
   20 Hz), `EVT <slot> <skillDefName>` packets, `F7` solo-lab enemy clear, AIM/FIRE command server
   (`inputBank.aimDirection` + `ExecuteIfReady`).
4. Launch order: PEAK modded first (binds), then a fresh RoR2 run (fresh spawn lock). Solo in both.

## Verification
- Oracle: host `F8` writes `LogOutput.log` lines (STATUS/hasPkt/stocks/`fx P:S:U:R:shots`/camera list/proxy
  projection) plus `riskofpeak_f8.png` via `ScreenCapture` (same frame). I read both directly.
- Bridge: `first guest packet` → `hasPkt=True`; red capsule filled the screen on the lens test
  (`visible=True`, depth +2.0 centered).
- Skills: 5 slow LMB clicks → guest `exec P:5` → host `fx P:5 shots:10` (Double Tap = 2 bullets). Zero loss.
- Gun-mode: `G` + LMB in PEAK → guest `remote FIRE P` → RoR2 consumes the shot, tracer returns
  (`P:34 S:3 U:1 R:2 shots:91` in one session).
- NOT verified: multiplayer, other loadouts/skins, Commando animations (bind pose + breathing only),
  PEAK-side damage (no targets in PEAK), RoR2 builds other than the Oct-2026 one.

## Gotchas
1. **Proxy invisible though active/enabled/layer-0.** Cause: `CreatePrimitive` default material breaks under
   URP (and a scene-copied material can be transparent). Fix: own material from URP/Lit + own opaque color;
   confirm with `renderer.isVisible` + `WorldToScreenPoint` depth sign + auto-screenshot.
2. **Camera drifts 20+ m behind.** Cause: postfix offset fights `CharacterCam`'s per-frame lerp/clamp.
   Fix: Harmony prefix replacing `CharacterCam` entirely (Evaisa pattern), never call orig in third person.
3. **Camera reacts to the proxy.** Cause: proxy on Default layer with a collider gets hit by the clip
   SphereCast. Fix: layer 2 (Ignore Raycast) + destroy the primitive collider.
4. **Shots detected "every other time".** Cause A: polling `cooldownRemaining` for a 0-crossing at 20 Hz
   misses held-primary gaps. Fix: fire edge = stock drop OR cooldown jumped *upward*. Cause B: one
   execution = N bullets (Double Tap 2, Shotgun 8, Barrage 10). Fix: send `skillDef.skillName` in EVT,
   expand to per-bullet flashes from a table on the host.
5. **Utility dive drew a bullet tracer.** Cause: every stock decrease flashed a tracer. Fix: slot-aware FX —
   U lunges the proxy, grenade names arc a ball, the rest trace.
6. **Proxy flung hundreds of meters.** Cause: spawn locked pre-drop; stage change; Russian-locale commas
   (`0,00`) vs invariant parse. Fix: relock on body change + `rel.y < -12`, `InvariantCulture` both sides,
   co-located final mapping (proxy IS the avatar).
7. **RoR2 input totally dead after a Steam update.** Cause: `SeekersPatcher` MMHOOK referenced a removed
   field (`jumpWasClaimed`), throwing inside `PlayerCharacterMasterController.Update` every frame.
   Fix: drop `bbepis-BepInExPack` + `SeekersPatcher`; `RoR2BepInExPack` alone bootstraps RoR2. Find the
   culprit by grepping plugin DLLs for the missing member name.
8. **`Camera.onPostRender` silent; `renderCount` dict explodes.** Cause: URP never calls it; keying the
   dict by position creates a key per frame. Fix: `RenderPipelineManager.beginCameraRendering`, key by
   camera name, keep last pos separately.
9. **ilspycmd `-o` with `-l`/`-t` creates a directory**, not a file (`X.cs\X.decompiled.cs` inside).
10. **Nested types need `Outer.Inner` for ilspycmd `-t`** (e.g. `Character.CharacterRefs`, not `CharacterRefs`).
11. **r2modman profile plugins folder starts empty**; DLLs locked while the game runs — close both games
    before redeploying (`user-mapped section open` error).

## Assets
- Commando body + 2 pistols + palette PNGs converted from the player's own RoR2 install at lab time
  (`src/extract_commando.py`, UnityPy reading `ror2-base-commando_*` bundles). Personal use only; the
  shareable part is code + converter. Skill icons: none (HUD is text). Tracer/grenade art: procedural.

## Cost and time
- Two sessions 2026-10-08/09, ~9 host + ~4 guest rebuilds, all diagnosed from in-game logs+screenshots.

## Open questions
- Animation retarget (RoR2 clips → PEAK, Batman-note style 51-clip pipeline) vs procedural lean/bob.
- PEAK-side damage/consequences (no vanilla targets); enemy HP bars as a HUD readout from guest numbers.
- Old RoR2 builds: bundle hashes move every update — the glob handles it, mesh identity held across one
  update (5422 verts both builds).
