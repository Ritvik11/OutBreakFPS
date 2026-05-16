# Architecture — OutBreak

This document explains *why* the slice is structured the way it is. The README is the elevator pitch; this is the whiteboard for an engineering reviewer.

> **Current state:** the repo is bootstrapped on the **UE 5.3 VR Template** (Blueprint), with `OpenXR`, `OpenXRHandTracking`, and `OpenXREyeTracker` plugins enabled — each plugin explicitly scoped to support `Win64`, `Linux`, and `Android`, making Quest 3 standalone a first-class target. The module layout, GAS-based ability system, encounter director, and damage pipeline described below are the **planned C++ architecture being layered onto that foundation**. Each section explains the design intent so reviewers can evaluate the engineering decisions even before the corresponding code lands. Implementation status is tracked via git tags (`v0.0.x-*`, `v0.1.x-*`) and GitHub issues.
>
> **Engine version rationale:** the project targets UE 5.3 specifically because the marketplace assets in the asset pipeline (factory environment packs, common alien enemy) cap their supported version at 5.3. Standardizing on 5.3 avoids per-asset version workarounds and keeps the asset import pipeline consistent. 5.3 still has the Quest-relevant features needed: OpenXR, mobile multi-view, instanced stereo, forward shading, and the eye-tracking plugin for foveated rendering.

---

## Goals & constraints

- **Quest 3 standalone first.** Mobile renderer, forward shading, baked lighting, ~150 visible draw call budget, 11 ms frame time, 90 FPS comfort threshold. Every system is designed inside that budget on-device, not extrapolated from PCVR.
- **Reviewable in 10 minutes.** A reviewer should be able to open one folder, read one header, and understand a system end-to-end.
- **Idiomatic Unreal.** GAS, Enhanced Input, Common UI, Data Assets, Gameplay Tags, OpenXR. Avoid bespoke frameworks that obscure engineering judgment.
- **C++ where it pays, Blueprint where it shines.** C++ for hot paths, replication-sensitive logic, and authoritative state. Blueprint for designer-tunable parameters, prototyping, and one-shot level scripting.
- **3-player co-op as the multiplayer target.** Listen-server topology; replication-aware design throughout. Player count chosen to fit Quest 3 perf budget for environment + characters + AI + FX simultaneously.

---

## Module layout

```
Source/
  OutBreak/                 # Runtime game module
    Public/
      Gameplay/             # AbilitySystem, Attributes, Inventory
      AI/                   # EncounterDirector, BT tasks, Perception
      Combat/               # DamagePipeline, HitReactions, Weapons
      Input/                # OpenXR input mappings (controllers + hand tracking)
      Locomotion/           # VR snap/smooth turn, decoupled HMD/capsule
      World/                # GameMode, GameState, world subsystems
      UI/                   # World-space wrist UI (OpenXR-attached widgets)
      OutBreak.h
    Private/
      [mirrors Public/]
    OutBreak.Build.cs

  OutBreakEditor/           # Editor-only module
    Public/
      Tools/                # Encounter authoring widget, validators
    OutBreakEditor.Build.cs

  OutBreakTests/            # Automation tests
    Private/
      Gameplay/
      AI/
    OutBreakTests.Build.cs
```

**Why three modules.** Editor-only code (`WITH_EDITOR`-guarded utilities, encounter authoring widgets) lives in `OutBreakEditor` so runtime builds — especially the Android Quest APK — don't carry it. A separate `OutBreakTests` module keeps automation specs from leaking into the shipping module manifest. Habit I carried over from Unity assembly definitions; pays the same dividend.

---

## Quest 3 standalone architecture

The slice runs on Quest 3 standalone (Adreno 740, ~3.5 TFLOPS effective, 90 FPS target, 8 GB shared RAM). Every system below is sized to that constraint.

The core principle: **systems are perf-aware at design time, not retrofitted.** That means baked lighting from day one, forward shading from day one, instanced stereo + mobile multi-view from day one, and aggressive HLOD on every static asset. Adding these as an afterthought once the project is dynamic-lighting-shaped is much harder than starting Quest-first and adding PC fidelity later.

This is also the reason a flatscreen PC build is a *follow-up* milestone rather than a concurrent target. PC inheriting the discipline of mobile constraints is a much smaller engineering lift than the other direction.

---

## Key systems

### Ability & Attribute system

Built on `GameplayAbilitySystem` (GAS).

| Option considered | Rejected because |
|---|---|
| Roll my own component graph | Every gameplay engineer reviewing this would have to re-learn it. GAS is the AAA lingua franca. |
| Pure Blueprint ability tree | Doesn't scale; replication and prediction become guesswork for 3-player co-op. |
| **GAS with custom `UGameplayAbility` subclasses** ✅ | Standard, replicated, predicted, designer-extensible. Networking model is mature. |

Attributes are `UAttributeSet` subclasses with `RepNotify` replication.

### Encounter Director

`UEncounterDirector` is a `UWorldSubsystem` that drives wave-based combat encounters. The slice ships three escalating waves:

| Wave | Composition | Design intent |
|---|---|---|
| 1 | Up to 10 visible minors, no elite | Train the team on baseline alien behavior. Build confidence and reveal level geometry. |
| 2 | 1 elite + ~7 minors | Paradigm shift moment — the elite arrives and forces re-prioritization. Reveals the gameplay surprise. |
| 3 | 1 elite + 1-3 minors with explicit tactical coordination | Final test. By dropping enemy count and adding coordination AI, the difficulty shifts from numbers to intelligence. |

Per-frame load is managed via two systems:
- **AI dormancy.** Minors beyond ~25m from any player have their BTs ticked at 1 Hz instead of 60 Hz. They still exist and render, but aren't decision-making.
- **Distance-based animation budgeting.** `AnimationBudgeting` plugin enabled with a 2 ms target — animation eval degrades gracefully for distant agents.

The director itself ticks at 4 Hz, re-scoring agents based on player cover state, recent damage taken/dealt, squad cohesion, and time since last engagement (enforcing lull → swell → ambush pacing). Each agent's BT consumes director output via blackboard keys. The BT decides *how*; the director decides *who and when*.

Wave compositions live in a `UEncounterDataAsset` (Blueprint-editable) so designers tune pacing without touching code. Wave state is replicated server → clients via `RepNotify` so all 3 co-op players see the same wave at the same time.

### Damage pipeline

Single chokepoint: `UCombatStatics::ApplyDamage(...)`. Melee, projectile, environmental — all funnel through here. Reasons:

1. One place for telemetry/logging.
2. One place to apply damage modifiers (resistance, multipliers, crit).
3. One place to dispatch hit-reaction FX — physical-material-aware Niagara hit reactions plus HMD-safe haptic feedback. No camera-jolting effects, ever (causes nausea in VR).

The "no camera-jolt in VR" rule is implemented as an architectural constraint at the damage pipeline, not scattered across weapon classes. Any weapon, any damage source, any AI attack — they all go through the same chokepoint and inherit the same comfort rules.

### Locomotion (VR comfort discipline)

VR locomotion is the easiest place to give players motion sickness, so the system is deliberate:

- **Snap turn** default (45°), smooth turn opt-in via menu.
- **Smooth locomotion** with optional comfort vignette that tightens on acceleration.
- **Player capsule** decoupled from HMD position — leaning physically doesn't push the collision capsule (avoids clipping wall geometry with your face).
- **No forced camera rotation, ever.** Cinematic moments are scripted as in-world events (a character on a screen pointing, environmental triggers), not camera cuts.

These are well-known VR comfort rules; the discipline is in *implementing them as architectural constraints* on `APlayerController` and `UCharacterMovementComponent` subclasses, not as scattered tweaks.

### UI

`CommonUI` for navigation focus (controller- and hand-tracking-friendly), UMG for content. World-space UI is anchored to the off-hand wrist via a `UWidgetComponent`, with depth testing tuned to not z-fight with held weapons. No screen-space HUD in VR — players would see it through reality and find it uncomfortable.

---

## Replication / networking (3-player co-op)

- **Listen-server topology.** One client hosts; the other two join. Chosen because dedicated servers add infrastructure cost (Linux VM hosting) that doesn't add portfolio value at slice scale. Listen-server demonstrates the same engineering tradeoffs (replication, authority, prediction) without ops overhead.
- Abilities use GAS prediction rather than ad-hoc client-side state. Movement uses standard `UCharacterMovementComponent` replication.
- Authoritative state lives behind `HasAuthority()` checks. AI ticks server-side; pose data replicates to clients.
- Per-actor `NetCullDistanceSquared` tuned aggressively for enemies — distant AI doesn't need to replicate pose updates at full rate. Saves bandwidth, especially during heavy waves.
- No `Tick`-driven gameplay state — anything important is event-driven so the dormancy system can pause AI BTs without breaking game logic.

---

## Data flow

```
        ┌──────────────┐       ┌──────────────────────────┐
Input → │ Enhanced     │ ───▶  │ APlayerController         │ ──┐
(OpenXR)│ Input        │       │   ↳ Activates GA          │   │
        └──────────────┘       └──────────────────────────┘   │
                                                              ▼
                              ┌─────────────────────────────────┐
                              │ UAbilitySystemComponent          │
                              │   ↳ Cost / cooldown / cues       │
                              │   ↳ Modifies Attributes          │
                              │   ↳ Predicted client-side        │
                              └────────────┬─────────────────────┘
                                           │
                          ┌────────────────┼────────────────┐
                          ▼                ▼                ▼
                  UCombatStatics    HitReact (Niagara     UI (wrist
                  (damage pipe)      + haptic feedback)    world-space)
```

---

## Why these choices over what I'd have done in Unity

A short, non-defensive section. Interviewers will ask anyway.

- **GAS over a ScriptableObject-driven ability system.** GAS is the AAA pattern; the steeper ramp is worth it. Replication and prediction come for free.
- **`UWorldSubsystem` instead of `DontDestroyOnLoad` singletons.** Explicit lifetimes, injected not globally fetched.
- **Gameplay Tags instead of enums.** Hierarchical, data-driven, designer-editable.
- **Data Assets instead of ScriptableObjects.** Same mental model; trivial transfer.
- **OpenXR instead of vendor-specific SDKs (e.g., Meta XR Plugin).** Vendor-neutral. Same reasoning Epic uses for first-party VR support: portable across HMDs, no per-headset divergence. The cost is losing some Meta-specific features (advanced foveation profiles, scene anchors); for a vertical slice those tradeoffs land on the right side.

---

## What's intentionally *not* here

Calling these out matters — reviewers should see scope decisions were deliberate.

- **Save/load** (slice resets each play; not a meaningful demo of anything).
- **Localization** (would dilute focus).
- **Flatscreen PC build** (planned follow-up milestone — the Quest 3 build is the discipline-forcing target; PC support layers on after it ships cleanly).
- **PCVR build via Link / SteamVR** (planned follow-up — same gameplay, higher-fidelity rendering on PC-tethered headsets).
- **Quest 2 / Pro support** (Quest 3 only; older devices would need a separate optimization pass and the slice is already constrained enough).
- **Live-ops / monetization hooks** (irrelevant to the role I'm targeting).

## What's enabled but not exercised yet

The VR Template foundation ships these plugins active; calling them out so reviewers know they're available and intentional:

- **`OpenXRHandTracking`** — hand tracking input available alongside controllers. Whether the slice uses it for specific interactions (e.g., menu navigation, environmental interaction) is a design decision tracked in roadmap issues.
- **`OpenXREyeTracker`** — eye tracking enabled so dynamic foveated rendering can be evaluated as a perf strategy on Quest 3 (see [PERFORMANCE.md](PERFORMANCE.md)).
