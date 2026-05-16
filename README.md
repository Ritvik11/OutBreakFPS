# OutBreak

> A co-op outbreak survival VR FPS vertical slice built for **Meta Quest 3 standalone** in Unreal Engine 5.3. Built to demonstrate gameplay programming, AI, and standalone VR engineering for AAA roles.

> Repository: `OutBreakFPS` · Project file: `OutBreak.uproject` (the repo name predates the project rename — both refer to the same slice).

<!-- Replace with a 5–8 second looping GIF of the strongest moment, captured on-device on Quest 3. -->
![Hero GIF](docs/media/hero.gif)

**[▶ 90-second showcase](https://youtu.be/REPLACE)** &nbsp;•&nbsp; **[⬇ Quest 3 build (APK)](https://github.com/Ritvik11/OutBreakFPS/releases)** &nbsp;•&nbsp; **[📄 Technical breakdown](docs/ARCHITECTURE.md)**

---

## TL;DR

| | |
|---|---|
| **Engine** | Unreal Engine 5.3 |
| **Platform** | Meta Quest 3 standalone (Android via OpenXR) |
| **Languages** | Blueprint (current foundation), C++ (gameplay-critical systems — in progress) |
| **XR features** | Controllers + hand tracking + eye tracking (foveation-ready), all via OpenXR |
| **Multiplayer** | 3-player co-op |
| **Performance target** | 90 FPS native, on-device |
| **Scope** | Solo project, in progress |
| **Source control** | Git + Git LFS, Conventional Commits, structured branching |
| **What I owned** | 100% — design, programming, integration, lighting, audio cues, build pipeline |

> **Status:** the repo is bootstrapped on the UE 5.3 VR Template (Blueprint) with `OpenXR`, `OpenXRHandTracking`, and `OpenXREyeTracker` plugins enabled — each plugin explicitly scoped to support `Win64`, `Linux`, and `Android` so Quest 3 standalone is a first-class target. C++ gameplay systems described below are being layered onto this foundation. Tagged milestones (`v0.0.1-scaffold` → `v0.0.2-project` → `v0.1.0-ue53-migration` → …) track the build-up so the project's evolution is visible in git history.

> **Why Quest-only:** Starting with the tighter platform forces engineering discipline (mobile renderer, ~150 draw call budget, 11 ms frame time, baked lighting only). A flatscreen PC build is a planned follow-up milestone once the Quest slice is locked — adding PC last *inherits* the discipline of mobile constraints rather than fighting against them.

---

## What this slice will demonstrate

The plan, paired with what's in the repo. Each item links to the architecture rationale; code references will turn into clickable paths as systems land.

- **Standalone VR gameplay architecture.** Mobile-renderer-friendly Blueprint foundation with a C++ shim layer for performance-critical paths. Enhanced Input wired to OpenXR controllers + hand tracking. VR locomotion using snap-turn by default with smooth-turn opt-in, physically decoupled HMD/capsule for comfort. Rationale: [docs/ARCHITECTURE.md § VR locomotion & comfort](docs/ARCHITECTURE.md#locomotion-vr-specifically).
- **Gameplay C++ + GAS.** Player and infected AI built on `UAbilitySystemComponent` with data-driven attributes, replicated and prediction-aware to support 3-player co-op cleanly. Rationale: [docs/ARCHITECTURE.md § Ability & Attribute system](docs/ARCHITECTURE.md#ability--attribute-system).
- **Wave-based encounter design.** Three escalating waves: light commons (Wave 1), commons + elite introduction (Wave 2), elite + coordinated minor under tactical AI (Wave 3). Spawn caps tuned for Quest 3 perf budget; coordination behaviors authored as Behavior Tree compositions, not hand-rolled. Rationale: [docs/ARCHITECTURE.md § Encounter Director](docs/ARCHITECTURE.md#encounter-director).
- **Combat pipeline.** Single `UCombatStatics::ApplyDamage` chokepoint feeds physical-material-aware Niagara hit reactions and HMD-safe haptic feedback (no camera-jolting effects in VR — comfort first). Rationale: [docs/ARCHITECTURE.md § Damage pipeline](docs/ARCHITECTURE.md#damage-pipeline).
- **Performance discipline for Quest 3.** Forward shading, mobile multi-view, instanced stereo, baked lightmaps, aggressive HLOD on environment, eye-tracked dynamic foveation via `OpenXREyeTracker`. Targets stated and measured on-device, not extrapolated from PCVR captures. Methodology: [docs/PERFORMANCE.md](docs/PERFORMANCE.md).
- **Engineering hygiene.** Architecture documented ([docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)), git workflow legible ([docs/GIT_WORKFLOW.md](docs/GIT_WORKFLOW.md)), perf targets stated and measured as systems come online.

---

## Visual tour

<!-- Captures from on-device Quest 3 gameplay. -->

| Infected ambush | VR weapon handling | Encounter pacing |
|---|---|---|
| ![](docs/media/ambush.gif) | ![](docs/media/vr-weapon.gif) | ![](docs/media/pacing.jpg) |
| Squad-coordinated flanking driven by EncounterDirector | Two-handed grip with physics-based recoil via OpenXR | Lighting + emissive material work tuned for the Quest 3 forward renderer |

---

## How to run it

### Play the build (Quest 3)

Grab the latest tagged release from the [Releases page](https://github.com/Ritvik11/OutBreakFPS/releases). Each release ships one artifact:

- `OutBreak-Quest3-vX.Y.Z.apk` — standalone Quest 3 build

To install on your Quest 3:

1. Enable Developer Mode on your Quest 3 via the Meta Horizon mobile app.
2. Connect the headset to your PC via USB-C and accept the "Allow USB Debugging?" prompt inside the headset.
3. Sideload the APK:
   ```bash
   adb install -r OutBreak-Quest3-vX.Y.Z.apk
   ```
4. In the headset: **Library → Unknown Sources → OutBreak → Launch**.

### Build from source

**Prerequisites:** Unreal Engine 5.3.x, OpenJDK 17, Android SDK + NDK (installed via UE's Turnkey: `RunUAT.bat Turnkey -command=InstallSdk -platform=Android -BestAvailable`), Git, Git LFS.

```bash
git lfs install
git clone https://github.com/Ritvik11/OutBreakFPS.git
cd OutBreakFPS
git lfs pull
```

Double-click `OutBreak.uproject` to open in UE 5.3. Once C++ systems land, the project will need Visual Studio: right-click the `.uproject` → **Generate Visual Studio project files** → open the generated `OutBreak.sln` → **Development Editor** → F5. First-time DDC build ≈ 10–15 minutes.

To package a Quest 3 APK: **Platforms menu → Android (ASTC) → Package Project**.

OpenXR, hand tracking, and eye tracking plugins are already enabled.

---

## Architecture at a glance

```
┌─────────────────────────────────────────────────────────────┐
│  OutBreak (Game Module)                                     │
│                                                             │
│  Gameplay/        Player, abilities, attributes, items      │
│  AI/              Infected behaviors, encounter director    │
│  Combat/          Damage pipeline, hit reactions, weapons   │
│  Input/           Enhanced Input — OpenXR controllers,      │
│                   hand tracking, eye tracking               │
│  Locomotion/      VR snap-turn + smooth-locomotion,         │
│                   decoupled HMD/capsule for comfort         │
│  World/           Streaming, gameplay tags, level scripts   │
│  UI/              World-space wrist UI (OpenXR-attached)    │
│                                                             │
└─────────────┬───────────────────────────────┬───────────────┘
              │                               │
              ▼                               ▼
   ┌──────────────────────┐         ┌──────────────────────┐
   │ OutBreakEditor       │         │ OutBreakTests        │
   │ (Editor utilities,   │         │ (Automation specs,   │
   │  encounter authoring)│         │  perf assertions)    │
   └──────────────────────┘         └──────────────────────┘
```

Full rationale — module boundaries, GAS vs. hand-rolled, replication choices for 3-player co-op, encounter director design — lives in **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)**.

---

## Performance

Targets stated up front and measured on-device. Methodology lives in [`docs/PERFORMANCE.md`](docs/PERFORMANCE.md).

### Quest 3 standalone target

| Metric | Target | Measured |
|---|---|---|
| Frame time | < 11.1 ms (90 Hz) | [fill in] |
| GPU time | < 8 ms | [fill in] |
| CPU game thread | < 4 ms | [fill in] |
| Reprojection rate | < 5% | [fill in] |
| Draw calls (visible) | < 200 | [fill in] |
| APK size | < 1 GB | [fill in] |

VR perf is non-negotiable. Dropped frames cause nausea, not just complaints. Every system is designed to fit the 11 ms / 8 ms budget on-device, including network replication and AI tick costs across all 3 co-op players.

---

## A note on my background

I spent ~10 years building production systems in Unity/C# (most recently as a principal engineer), including substantial XR work. This slice exists to show those engineering instincts transfer cleanly to Unreal — particularly the discipline of shipping inside a tight perf budget on standalone VR hardware.

A short retrospective on the Unity → UE transition: **[docs/unity-to-unreal-notes.md](docs/unity-to-unreal-notes.md)**.

---

## Roadmap

This is a portfolio slice, not a shipping product. After the Quest 3 standalone build lands cleanly, the next planned milestones are:

- **PC flatscreen build** — same gameplay, same codebase, flatscreen input + camera. Inherits the perf discipline of the Quest target rather than fighting it.
- **PCVR build (SteamVR / Quest Link)** — higher-fidelity rendering on PC-tethered headsets, sharing the standalone gameplay layer.

Future-work items are tracked as GitHub issues with the `future-work` label.

---

## License

Code: MIT (see [LICENSE](LICENSE)).
Art assets: see [docs/ASSET_ATTRIBUTION.md](docs/ASSET_ATTRIBUTION.md). Some assets come from Epic's free samples, Quixel/Fab, or licensed marketplace packs and are **not** redistributable under the project's MIT license.

---

## Contact

Ritvik — [ritvik@kinemeric.com](mailto:ritvik@kinemeric.com) — [LinkedIn](https://linkedin.com/in/REPLACE) — [Portfolio](https://REPLACE)
