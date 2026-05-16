# Architecture — OutBreakFPS

This document explains *why* the slice is structured the way it is. The README is the elevator pitch; this is the whiteboard for an engineering reviewer.

> **Current state:** the repo is bootstrapped on the **UE 5.3 VR Template** (Blueprint), with `OpenXR`, `OpenXRHandTracking`, and `OpenXREyeTracker` plugins enabled — each plugin explicitly scoped to support `Win64`, `Linux`, and `Android`, making Quest 3 standalone a first-class target rather than a retrofit. The module layout, GAS-based ability system, encounter director, and damage pipeline described below are the **planned C++ architecture being layered onto that foundation**. Each section explains the design intent so reviewers can evaluate the engineering decisions even before the corresponding code lands. Implementation status is tracked via git tags (`v0.0.x-*`, `v0.1.x-*`) and GitHub issues.
>
> **Engine version rationale:** the project targets UE 5.3 specifically because the marketplace assets in the asset pipeline (factory environment packs, common alien enemy) cap their supported version at 5.3. Standardizing on 5.3 avoids per-asset version workarounds and keeps the asset import pipeline consistent. 5.3 still has the Quest-relevant features needed: OpenXR, mobile multi-view, instanced stereo, forward shading, and the eye-tracking plugin for foveated rendering.

---

## Goals & constraints

- **Reviewable in 10 minutes.** A reviewer should be able to open one folder, read one header, and understand a system end-to-end.
- **Idiomatic Unreal.** GAS, Enhanced Input, Common UI, Data Assets, Gameplay Tags, OpenXR. Avoid bespoke frameworks that obscure engineering judgment.
- **One codebase, two platforms.** PC flatscreen and VR share the same gameplay, AI, and combat code. Platform differences are isolated to clearly named modules (`Input/`, `Locomotion/`, `UI/`).
- **C++ where it pays, Blueprint where it shines.** C++ for hot paths, replication-sensitive logic, and authoritative state. Blueprint for designer-tunable parameters, prototyping, and one-shot level scripting.

---

## Module layout

```
Source/
  OutBreakFPS/              # Runtime game module
    Public/
      Gameplay/             # AbilitySystem, Attributes, Inventory
      AI/                   # EncounterDirector, BT tasks, Perception
      Combat/               # DamagePipeline, HitReactions, Weapons
      Input/                # PC + OpenXR input mappings, action remap
      Locomotion/           # Flatscreen walk/sprint, VR snap/smooth
      World/                # GameMode, GameState, world subsystems
      UI/                   # HUD (PC), world-space widgets (VR)
      OutBreakFPS.h
    Private/
      [mirrors Public/]
    OutBreakFPS.Build.cs

  OutBreakFPSEditor/        # Editor-only module
    Public/
      Tools/                # Encounter authoring widget, validators
    OutBreakFPSEditor.Build.cs

  OutBreakFPSTests/         # Automation tests
    Private/
      Gameplay/
      AI/
    OutBreakFPSTests.Build.cs
```

**Why three modules.** Editor-only code (`WITH_EDITOR`-guarded utilities, encounter authoring widgets) lives in `OutBreakFPSEditor` so runtime builds don't carry it. A separate `OutBreakFPSTests` module keeps automation specs from leaking into the shipping module manifest. Habit I carried over from Unity assembly definitions; pays the same dividend.

---

## Dual-platform strategy (PC + VR)

This is the design choice most worth understanding. The same gameplay/AI/combat code runs on both platforms because platform-specific concerns are isolated to three modules:

| Module | What lives there | PC implementation | VR implementation |
|---|---|---|---|
| `Input/` | Enhanced Input action mappings | KB+M + gamepad context | OpenXR controller context (grip, trigger, thumbstick, button overlays) |
| `Locomotion/` | Player movement | First-person camera, `UCharacterMovementComponent` | Snap-turn (default) + smooth-locomotion option, vignetting on acceleration |
| `UI/` | Player-facing HUD | Screen-space UMG | World-space WidgetComponent attached to off-hand wrist |

Everything else — abilities, damage, AI, weapons, encounter pacing — is platform-agnostic. The player pawn is the same `AOutBreakFPSCharacter`; what differs is which subclass of `UPlayerInputComponent` and `UPlayerLocomotionComponent` is attached at spawn, decided by a `UPlatformSubsystem` reading the runtime XR state.

**Why this matters to an AAA reviewer:** the project doesn't fork into two parallel codebases. It demonstrates engineering hygiene around a shared core with platform shims — the same pattern AAA studios use when shipping a flat title with VR support (or vice versa).

---

## Key systems

### Ability & Attribute system

Built on `GameplayAbilitySystem` (GAS).

| Option considered | Rejected because |
|---|---|
| Roll my own component graph | Every gameplay engineer reviewing this would have to re-learn it. GAS is the AAA lingua franca. |
| Pure Blueprint ability tree | Doesn't scale; replication and prediction become guesswork. |
| **GAS with custom `UGameplayAbility` subclasses** ✅ | Standard, replicated, predicted, designer-extensible. |

Attributes are `UAttributeSet` subclasses with `RepNotify` replication.

### Encounter Director

`UEncounterDirector` is a `UWorldSubsystem` that ticks at 4 Hz and re-scores infected agents based on:

- Player cover state
- Recent damage taken/dealt
- Squad cohesion among nearby infected
- Time since last engagement (pacing — enforces lull → swell → ambush rhythm)

Each agent's BT consumes director output via blackboard keys. The BT decides *how*; the director decides *who and when*. Decoupling lets me tune encounter feel without touching agent AI.

### Damage pipeline

Single chokepoint: `UCombatStatics::ApplyDamage(...)`. Melee, projectile, environmental — all funnel through here. Reasons:

1. One place for telemetry/logging.
2. One place to apply damage modifiers (resistance, multipliers, crit).
3. One place to dispatch hit-reaction FX (and to branch on platform — VR gets haptic feedback + low-intensity Niagara; PC gets full screen-shake + camera kick).

The "no camera-jolting effects in VR" rule lives here as a single platform check, not scattered across weapons.

### Locomotion (VR specifically)

VR locomotion is the easiest place to give players motion sickness, so the system is deliberate:

- **Snap turn** default (45°), smooth turn opt-in via menu.
- **Smooth locomotion** with optional comfort vignette that tightens on acceleration.
- **Player capsule** decoupled from HMD position — leaning physically doesn't push the collision capsule (avoids clipping wall geometry with your face).
- **No forced camera rotation, ever.** Cinematic moments are scripted as in-world events, not camera cuts.

These are well-known VR comfort rules; the discipline is in *implementing them as architectural constraints*, not as scattered tweaks.

### UI

`CommonUI` for navigation focus (controller- and VR-friendly), UMG for content. World-space UI in VR is anchored to the off-hand wrist, attached via `UWidgetComponent` with depth testing tuned to not z-fight with held weapons.

---

## Replication / networking

Slice is single-player, but systems are authored as if it weren't:

- Abilities use GAS prediction rather than ad-hoc client-side state.
- Authoritative state lives behind `HasAuthority()` checks even when there's only one player.
- No `Tick`-driven gameplay state — anything important is event-driven.

This costs almost nothing at slice scale and keeps the door open without a rewrite.

---

## Data flow

```
        ┌──────────────┐       ┌──────────────────────────┐
Input → │ Enhanced     │ ───▶  │ APlayerController         │ ──┐
(PC/VR) │ Input        │       │   ↳ Activates GA          │   │
        └──────────────┘       └──────────────────────────┘   │
                                                              ▼
                              ┌─────────────────────────────────┐
                              │ UAbilitySystemComponent          │
                              │   ↳ Cost / cooldown / cues       │
                              │   ↳ Modifies Attributes          │
                              └────────────┬─────────────────────┘
                                           │
                          ┌────────────────┼────────────────┐
                          ▼                ▼                ▼
                  UCombatStatics    HitReact (Niagara     UI (RepNotify
                  (damage pipe)      + platform shim)      + RepNotify)
```

---

## Why these choices over what I'd have done in Unity

A short, non-defensive section. Interviewers will ask anyway.

- **GAS over a ScriptableObject-driven ability system.** GAS is the AAA pattern; the steeper ramp is worth it.
- **`UWorldSubsystem` instead of `DontDestroyOnLoad` singletons.** Explicit lifetimes, injected not globally fetched.
- **Gameplay Tags instead of enums.** Hierarchical, data-driven, designer-editable.
- **Data Assets instead of ScriptableObjects.** Same mental model; trivial transfer.
- **OpenXR instead of vendor-specific SDKs.** Same reasoning Epic uses for first-party VR support: portable across HMDs, no per-headset divergence.

---

## What's intentionally *not* here

Calling these out matters — reviewers should see scope decisions were deliberate.

- Save/load (slice resets each play; not a meaningful demo of anything).
- Localization (would dilute focus).
- Multi-player netcode demo (systems are net-ready, exercising them needs infrastructure that doesn't read in a portfolio).
- Quest standalone build (PC-tethered VR via Link/SteamVR is the target; Quest standalone would need a separate optimization pass and is on the roadmap, not in the slice).
- Live-ops / monetization hooks (irrelevant to the role I'm targeting).

## What's enabled but not exercised yet

The VR Template foundation ships these plugins active; calling them out so reviewers know they're available and intentional:

- **`OpenXRHandTracking`** — hand tracking input is available alongside controllers. Whether the slice uses it for any specific interaction is a design decision tracked in roadmap issues.
- **`OpenXREyeTracker`** — eye tracking enabled so dynamic foveated rendering can be evaluated as a perf strategy (see [PERFORMANCE.md](PERFORMANCE.md)).
