# Performance — OutBreakFPS

Targets and how they're measured. Numbers without methodology are vibes; this page is the methodology.

---

## PC target

| Metric | Target | Rationale |
|---|---|---|
| Frame time @ 1440p, RTX 3070 | < 16.6 ms | Locked 60 FPS during gameplay, not just at the splash |
| GPU time | < 12 ms | Leaves headroom on lower-end cards |
| CPU game thread | < 4 ms | More than that on a single encounter and the architecture is wrong |
| Draw calls | < 4,000 | Pragmatic ceiling for Nanite + traditional mix |
| Persistent memory | < 4 GB | Console-class budget; not arbitrary |
| Cold load to playable | < 8 s | Below the threshold where players alt-tab |

## VR target

VR runs on a tighter budget because dropped frames cause nausea, not just complaints.

| Metric | Target (Quest 3 via Link / Index) | Rationale |
|---|---|---|
| Frame time | < 11.1 ms (90 Hz) | Below that, reprojection kicks in |
| GPU time | < 8 ms | Stereo rendering doubles cost; headroom matters more |
| CPU game thread | < 4 ms | Same budget as PC; VR is GPU-bound first |
| Reprojection rate | < 5% | Above this, comfort degrades visibly |
| Motion-to-photon latency | < 20 ms | Comfort threshold for most users |
| Draw calls | < 2,500 | Stereo doubles, so we halve the budget |

---

## How to reproduce

```
# Editor console (~)
stat unit                # frame timings
stat scenerendering      # draw calls, primitives
stat memory              # heap usage
stat gpu                 # GPU timings per pass

# VR-specific
stat vr                  # reprojection rate, late update timing
stat xr                  # OpenXR layer breakdown
```

Full traces:
```
Engine\Binaries\Win64\UnrealInsights.exe
```

For packaged build:
```
OutBreakFPS.exe -trace=cpu,gpu,frame,bookmark -tracehost=127.0.0.1
```

All measurements taken on Encounter01, starting from the same checkpoint, with the same input sequence. Methodology matters more than the absolute numbers — reviewers should be able to repeat the capture.

---

## Captures

Insights traces and `stat unit` screenshots live in `docs/media/perf/`. Commit them; they're the receipts.

| Date | Build | Scene | Platform | GT (ms) | RT (ms) | GPU (ms) | Repro % | Notes |
|---|---|---|---|---|---|---|---|---|
| YYYY-MM-DD | v0.1.0 | Encounter01 mid | PC 1440p | — | — | — | — | Baseline |
| YYYY-MM-DD | v0.1.0 | Encounter01 mid | Quest 3 (Link) | — | — | — | — | Baseline |

---

## Optimization log

Every meaningful perf win, the technique, the delta. Reviewers love these — closest thing to seeing you think.

| Change | Technique | Before → After | Platform | Where |
|---|---|---|---|---|
| Hit-react MID pooling | Material instance reuse | 2.1 → 1.7 ms GT | Both | `UCombatStatics::ApplyHitFx` |
| Off-frustum Niagara cull | Frustum check at spawn | 8% → 1.2% repro | VR | `UHitReactComponent::Spawn` |
| | | | | |

Empty rows are fine — they show this section will keep growing. An honest empty log is more credible than fabricated numbers.

---

## VR-specific notes

- **Forward shading** evaluated and rejected for this slice — Lumen + GPU instancing keeps us in budget and reads better visually. Documented the experiment so reviewers don't think it was skipped.
- **Fixed Foveated Rendering** (FFR) enabled on Quest builds, off on PC tethered VR (Index/Reverb have higher pixel density tolerance).
- **Instanced Stereo Rendering** on by default — non-negotiable for VR perf.
- **Mobile preview** not used for Quest standalone builds; targeting Link/PCVR only keeps the scope honest. Standalone Quest 3 native would need a separate optimization pass and is called out in the roadmap.
