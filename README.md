# OutBreakFPS

> A single-player outbreak survival FPS vertical slice built in Unreal Engine 5, playable on flatscreen PC **and** in VR from one shared codebase. Built to demonstrate gameplay programming, AI, and dual-platform engineering for AAA roles.

<!-- Replace with a 5–8 second looping GIF of the strongest moment. -->
![Hero GIF](docs/media/hero.gif)

**[▶ 90-second showcase](https://youtu.be/REPLACE)** &nbsp;•&nbsp; **[⬇ Playable build — PC (Win64)](https://github.com/Ritvik11/OutBreakFPS/releases)** &nbsp;•&nbsp; **[⬇ Playable build — VR (Quest / SteamVR)](https://github.com/Ritvik11/OutBreakFPS/releases)** &nbsp;•&nbsp; **[📄 Technical breakdown](docs/ARCHITECTURE.md)**

---

## TL;DR

| | |
|---|---|
| **Engine** | Unreal Engine 5.7 |
| **Languages** | Blueprint (current foundation), C++ (gameplay-critical systems — in progress) |
| **Platforms** | Windows (flatscreen) + OpenXR (Quest 3, Index, Reverb G2) |
| **XR features** | Controller + hand tracking + eye tracking (foveation-ready), all via OpenXR |
| **Scope** | Solo project, in progress |
| **Target — PC** | 60 FPS at 1440p, RTX 3070-class |
| **Target — VR** | 90 FPS at native HMD resolution, RTX 3070-class |
| **Source control** | Git + Git LFS, Conventional Commits, structured branching |
| **What I owned** | 100% — design, programming, integration, lighting, audio cues, build pipeline |

> **Status:** the repo is set up on the UE 5.7 VR Template (Blueprint) with `OpenXR`, `OpenXRHandTracking`, and `OpenXREyeTracker` plugins enabled. C++ gameplay systems described below are being layered onto this foundation. Tagged milestones (`v0.0.1-scaffold` → `v0.0.2-project` → …) track the build-up so the project's evolution is visible in git history.

---

## What this slice will demonstrate

The plan, paired with what's already in the repo. Each item links to the architecture rationale; code references will turn into clickable paths as systems land.

- **Dual-platform gameplay architecture.** One shared gameplay/AI/combat codebase with platform-specific input (Enhanced Input — KB+M, gamepad, OpenXR controllers, hand tracking) and locomotion (flatscreen first-person vs. VR snap-turn + smooth-locomotion). Foundation is the UE 5.7 VR Template; C++ shim layer in progress. Rationale: [docs/ARCHITECTURE.md § Dual-platform strategy](docs/ARCHITECTURE.md#dual-platform-strategy-pc--vr).
- **Gameplay C++ + GAS.** Player and infected AI to be built on `UAbilitySystemComponent` with data-driven attributes, replicated and prediction-aware even in single-player. Rationale: [docs/ARCHITECTURE.md § Ability & Attribute system](docs/ARCHITECTURE.md#ability--attribute-system).
- **AI behavior.** Behavior Tree + utility scoring hybrid. A `UEncounterDirector` world subsystem drives infected pacing (lulls, swells, ambushes) based on player cover state and tempo of recent combat. Rationale: [docs/ARCHITECTURE.md § Encounter Director](docs/ARCHITECTURE.md#encounter-director).
- **Combat pipeline.** Single `UCombatStatics::ApplyDamage` chokepoint feeds physical-material-aware Niagara hit reactions, screen-shake on PC, and HMD-safe haptic feedback on VR (no camera-jolting effects in VR — comfort first). Rationale: [docs/ARCHITECTURE.md § Damage pipeline](docs/ARCHITECTURE.md#damage-pipeline).
- **Performance discipline for VR.** Lumen tuned for VR's tighter budgets, Nanite where it pays, instanced static meshes for infected swarms, eye-tracked dynamic foveation via the enabled `OpenXREyeTracker` plugin. Methodology: [docs/PERFORMANCE.md](docs/PERFORMANCE.md).
- **Engineering hygiene.** Architecture documented ([docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)), git workflow legible ([docs/GIT_WORKFLOW.md](docs/GIT_WORKFLOW.md)), perf targets stated and measured as systems come online.

---

## Visual tour

| Infected ambush | VR weapon handling | Encounter pacing |
|---|---|---|
| ![](docs/media/ambush.gif) | ![](docs/media/vr-weapon.gif) | ![](docs/media/pacing.jpg) |
| Squad-coordinated flanking driven by EncounterDirector | Two-handed grip with physics-based recoil, OpenXR | Lumen + emissive material work, tuned for both display targets |

---

## How to run it

### Play the build
Grab the latest tagged release from the [Releases page](https://github.com/Ritvik11/OutBreakFPS/releases). Two artifacts per release:
- `OutBreakFPS-Win64-PC-vX.Y.Z.zip` — flatscreen build
- `OutBreakFPS-Win64-VR-vX.Y.Z.zip` — OpenXR build (SteamVR / Quest Link)

No installer. Unzip, run `OutBreakFPS.exe`.

### Build from source

**Prerequisites:** Unreal Engine 5.7.x, Visual Studio 2022 with the *Game development with C++* workload, Git, Git LFS.

```bash
git lfs install
git clone https://github.com/Ritvik11/OutBreakFPS.git
cd OutBreakFPS
git lfs pull
```

Double-click `OutBreakFPS.uproject` to open in the editor. Once C++ systems land, the project will need Visual Studio: right-click the `.uproject` → **Generate Visual Studio project files** → open the generated `OutBreakFPS.sln` → **Development Editor** → F5. First-time DDC build ≈ 10–15 minutes.

OpenXR, hand tracking, and eye tracking plugins are already enabled in the project. For VR Preview: **Play dropdown → VR Preview** with your headset connected.

---

## Architecture at a glance

```
┌─────────────────────────────────────────────────────────────┐
│  OutBreakFPS (Game Module)                                  │
│                                                             │
│  Gameplay/        Player, abilities, attributes, items      │
│  AI/              Infected behaviors, encounter director    │
│  Combat/          Damage pipeline, hit reactions, weapons   │
│  Input/           Enhanced Input — PC + OpenXR mappings     │
│  Locomotion/      Platform-aware movement (flat / VR)       │
│  World/           Streaming, gameplay tags, level scripts   │
│  UI/              HUD (PC) + world-space UI (VR)            │
│                                                             │
└─────────────┬───────────────────────────────┬───────────────┘
              │                               │
              ▼                               ▼
   ┌──────────────────────┐         ┌──────────────────────┐
   │ OutBreakFPSEditor    │         │ OutBreakFPSTests     │
   │ (Editor utilities,   │         │ (Automation specs,   │
   │  encounter authoring)│         │  perf assertions)    │
   └──────────────────────┘         └──────────────────────┘
```

Full rationale — module boundaries, GAS vs hand-rolled, PC/VR code sharing strategy, replication choices — lives in **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)**.

---

## Performance

Stated up front and measured. Methodology lives in [`docs/PERFORMANCE.md`](docs/PERFORMANCE.md).

### PC target (RTX 3070, 1440p)

| Metric | Target | Measured |
|---|---|---|
| Frame time | < 16.6 ms | [fill in] |
| GPU time | < 12 ms | [fill in] |
| Persistent memory | < 4 GB | [fill in] |

### VR target (RTX 3070, Quest 3 via Link / Index)

| Metric | Target | Measured |
|---|---|---|
| Frame time | < 11.1 ms (90 Hz) | [fill in] |
| GPU time | < 8 ms | [fill in] |
| Reprojection rate | < 5% | [fill in] |
| Motion-to-photon | < 20 ms | [fill in] |

VR targets are tighter than PC by design — comfort is non-negotiable. Dropped frames in VR cause nausea; dropped frames on PC cause complaints. The same project respects both budgets.

---

## A note on my background

I spent ~10 years building production systems in Unity/C# (most recently as a principal engineer), including substantial XR work. This slice exists to show those engineering instincts transfer cleanly to Unreal — and that the dual-platform discipline I bring to XR work is a meaningful asset to AAA studios, not a distraction from them.

A short retrospective on the Unity → UE transition: **[docs/unity-to-unreal-notes.md](docs/unity-to-unreal-notes.md)**.

---

## Roadmap

This is a portfolio slice, not a shipping product. Future-work items are tracked as GitHub issues with the `future-work` label.

---

## License

Code: MIT (see [LICENSE](LICENSE)).
Art assets: see [docs/ASSET_ATTRIBUTION.md](docs/ASSET_ATTRIBUTION.md). Some assets come from Epic's free samples, Quixel/Fab, or licensed marketplace packs and are **not** redistributable under the project's MIT license.

---

## Contact

Ritvik — [ritvik@kinemeric.com](mailto:ritvik@kinemeric.com) — [LinkedIn](https://linkedin.com/in/REPLACE) — [Portfolio](https://REPLACE)
