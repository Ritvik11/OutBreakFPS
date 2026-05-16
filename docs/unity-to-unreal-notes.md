# Unity → Unreal: a principal engineer's transition notes

Honest, short, non-defensive. For an interviewer who wants to know if I'll be productive in UE on day one.

---

## What translated immediately

| Unity concept | Unreal equivalent | Notes |
|---|---|---|
| `MonoBehaviour` | `UActorComponent` / `AActor` | Component composition mental model is identical. |
| `ScriptableObject` | `UDataAsset` / `UPrimaryDataAsset` | Same role: data containers decoupled from scene instances. |
| `Update()` | `Tick(DeltaTime)` | Same hazards. Avoid both when an event will do. |
| Prefab | Blueprint Class | Both are reusable assemblies with default values. |
| `[SerializeField]` | `UPROPERTY(EditAnywhere)` | Same intent, richer metadata in UE (`Category`, `meta=(ClampMin=...)`). |
| `OnEnable / OnDisable` | `BeginPlay / EndPlay` | Lifecycle hooks; same shape. |
| Layers + Tags | Gameplay Tags + Collision Channels | UE's split is cleaner — gameplay tags are hierarchical and data-driven. |
| Coroutines | Timers, Latent actions, native C++ coroutines (5.4+) | Multiple options; I default to Timers for short delays, Latent BP nodes for designer-readable async. |
| `Resources.Load` | `LoadObject<T>` / async `FStreamableManager` | Soft references via `TSoftObjectPtr` are the idiomatic move. |
| Assembly definitions | Modules (`*.Build.cs`) | Same goal: split compile units, reduce iteration time. |
| Unity XR Toolkit | OpenXR + `IXRTrackingSystem` | XR Toolkit's abstractions map almost 1:1 onto UE's OpenXR plugin. The XR work I've done in Unity transfers especially cleanly. |

---

## What I had to rewire

- **No GameObject root.** `AActor` *is* the thing; you don't add components to a generic container. Took a day to stop reaching for an analog.
- **Headers and reflection.** `UCLASS` / `UPROPERTY` / `UFUNCTION` macros do what C# attributes do — but the build tool reads them at compile time. Forgetting `UPROPERTY()` on a `UObject*` member is the most expensive mistake a Unity dev can make: GC will eat your pointer. I lint for it.
- **Blueprint as a first-class collaborator.** In Unity I'd rarely give designers visual scripting. In UE the *right* answer is to expose tunable surfaces as `BlueprintNativeEvent` / `BlueprintImplementableEvent` hooks. The C++/BP boundary is where most engineering judgment lives.
- **Replication is the floor, not an add-on.** Even in single-player, GAS' replication-aware design pushed me toward better state architecture than my default Unity habits.
- **Iteration loop.** Hot reload is unreliable; Live Coding is better but not free. My workflow shifted to "compile less often, test more deliberately." This is a feature, not a bug.

---

## XR specifically

I've shipped real XR work in Unity (Quest, Vision Pro, WebXR). UE's OpenXR plugin is mature and the patterns I rely on transferred:

- One input action graph for all controllers (UE's Enhanced Input + OpenXR action sets are functionally Unity's Input System + XR Toolkit actions).
- Comfort discipline (no forced camera rotation, decoupled HMD/capsule, snap turn default) is engine-agnostic — these are human-comfort rules, not Unity rules.
- Performance budgets are the same shape. VR is GPU-bound first on both engines; the optimizations look similar (instanced stereo, fixed foveation where supported, aggressive culling on swarms).

The reason OutBreak targets Quest 3 standalone first isn't novelty — it's that working inside the tighter perf budget is the same engineering muscle that AAA studios exercise when targeting console hardware: shipping inside hard constraints rather than burning headroom. The discipline transfers cleanly.

---

## What I still prefer about Unity

Including this because not naming it would read as insincere.

- **Iteration time.** C# recompile + domain reload is faster than C++ + UBT, full stop.
- **Package ecosystem clarity.** UPM is more legible than `*.uplugin` plus Fab plus loose `Plugins/` folders.
- **Editor scripting API surface.** Easier to extend the Unity editor in 50 lines of C# than the UE editor in 200 lines of C++ + Slate. UE Editor Utility Widgets close some of the gap.

I name these because (a) they're true and (b) being able to compare engines honestly is the skill, not pretending one is strictly better.

---

## Where my Unity experience makes me *better* in UE

The paragraph I want a hiring manager to read.

- **I treat GC and pointer lifecycles seriously.** A decade of Unity-style `null` checks on cross-scene references made me paranoid about `WeakObjectPtr` vs raw `UObject*` in the right places.
- **I refuse to put gameplay logic on Tick.** Unity taught me the cost; UE rewards the discipline even more because Tick is more expensive per actor.
- **I structure data first, behavior second.** Years of ScriptableObject-driven design translates directly to `UPrimaryDataAsset`-driven UE projects, which is the AAA-standard approach.
- **I've shipped XR.** Comfort, on-device perf, and shipping inside tight standalone-VR constraints aren't theoretical for me — they're the discipline of my last several years.
- **I know where the bodies are buried in component composition.** The Unity/UE component models rhyme; so do their failure modes. I've seen them in production and don't have to learn them again.
