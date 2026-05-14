# Git workflow — OutBreakFPS

Solo project, but written as if a teammate might join. Reviewers read commit history and branching strategy. Making both legible signals seniority just as much as code does.

---

## First-time setup (one-time)

```bash
# 1. Install Git LFS
git lfs install

# 2. After cloning, resolve LFS pointers
git lfs pull

# 3. Configure identity for this repo
git config user.name  "Ritvik"
git config user.email "ritvik@kinemeric.com"
```

---

## Branching

`main` is always playable on both PC and VR. If `main` is red, that's a P0.

| Branch | Purpose | Example |
|---|---|---|
| `main` | Latest playable, tagged releases come from here | `main` |
| `feat/<area>-<short-desc>` | New feature or system | `feat/ai-encounter-director` |
| `fix/<area>-<short-desc>` | Bug fix | `fix/combat-double-damage-on-headshot` |
| `perf/<area>-<short-desc>` | Perf-only change, no behavior delta | `perf/vr-reprojection-spikes` |
| `chore/<short-desc>` | Tooling, docs, CI | `chore/update-gitignore` |
| `experiment/<short-desc>` | Throwaway exploration | `experiment/raymarched-fog` |

Merge with `--no-ff` so feature shape stays visible in the log:

```bash
git checkout main
git merge --no-ff feat/ai-encounter-director
```

For solo work, branching still pays — keeps WIP off `main` and gives you a clean revert handle.

### Using GitHub Desktop instead of CLI

GitHub Desktop is fine for most operations. The workflow maps cleanly:

- **New branch** → branch dropdown → "New branch" → name it per the table above.
- **Commit** → fill in the message following Conventional Commits (next section). Desktop wraps the body for you.
- **Push origin** → top bar.
- **Merge** → branch dropdown → "Merge into current branch" → pick source.
- **Tag a release** → CLI only (see "Tags & releases"). Desktop doesn't do tags well.

LFS is transparent in Desktop once `git lfs install` has been run on the machine.

---

## Commit messages — Conventional Commits

```
<type>(<scope>): <subject>

<body — what changed and why, wrapped at ~72 chars>

<footer — e.g., "Closes #12", "BREAKING CHANGE: ...">
```

Types: `feat`, `fix`, `perf`, `refactor`, `docs`, `chore`, `test`, `build`, `ci`.

Examples:

```
feat(ai): add encounter director with cover-aware scoring

Director ticks at 4 Hz and writes target scores to each infected
agent's blackboard. Decouples "how to engage" (BT) from "who and
when" (director). See docs/ARCHITECTURE.md.

Closes #14
```

```
perf(vr): cull off-frustum hit-react Niagara emitters

Reprojection rate on Encounter01 was spiking to 8% during swarms.
Hit-react emitters spawned by off-screen infected were still
allocating particle buffers. Frustum-cull at spawn time.

Reprojection rate: 8% → 1.2% on Index, 5% → 0.8% on Quest 3 (Link).
```

Read `git log --oneline` and you can reconstruct the project's story. That is the goal.

---

## Tags & releases

```bash
git tag -a v0.1.0 -m "Vertical slice — first playable"
git push origin v0.1.0
```

Each tag → a GitHub Release with two artifacts:
- `OutBreakFPS-Win64-PC-v0.1.0.zip`
- `OutBreakFPS-Win64-VR-v0.1.0.zip`

Keeps the README's download links pointing somewhere alive.

---

## Git LFS — operational notes

- Free GitHub gives **1 GB storage / 1 GB bandwidth per month** for LFS. A UE5 portfolio slice eats this fast. Plan ahead:
  - Keep raw source-art (uncompressed PSD/TGA) out of LFS where possible (the `.gitignore` already excludes `SourceArt/**/*.png` and `*.tga`).
  - For releases, attach packaged builds to the GitHub Release rather than committing them.
  - If you bump the LFS limit, GitHub Pro adds a buffer; for serious work, self-host LFS via Backblaze B2 or AWS S3.
- `git lfs ls-files` — see what's tracked.
- `git lfs migrate import --include="*.uasset,*.umap" --everything` — rewrite history to move pre-LFS binaries into LFS. **Do this before publishing**, never after others have cloned.
- `git lfs prune` removes unreferenced LFS objects locally. Safe on a solo repo.

---

## Pre-push checklist

Before every push to `main`:

1. `Development Editor` compiles clean — no new warnings.
2. Editor opens without errors (`Saved/Logs/OutBreakFPS.log`).
3. **PC PIE** — Encounter01 plays to completion.
4. **VR PIE** — Encounter01 plays to completion at 90 FPS, no reprojection spikes during swarm.
5. `Engine → Test Automation` runs `OutBreakFPSTests` to green.
6. `stat unit` still hits targets on both platforms.
7. `git status` — no stray `Saved/`, `Intermediate/`, `.vs/`, `.idea/`. `.gitignore` should catch these; double-check.
8. Commit message follows the convention above.

---

## Useful aliases

Drop into `~/.gitconfig`:

```ini
[alias]
    s        = status -sb
    l        = log --oneline --graph --decorate -20
    la       = log --oneline --graph --decorate --all
    co       = checkout
    cb       = checkout -b
    amend    = commit --amend --no-edit
    undo     = reset --soft HEAD~1
    pristine = !git reset --hard && git clean -fdx
    lfs-ls   = lfs ls-files
```

`git pristine` is dangerous and useful — nukes everything not tracked. Reach for it when UE's `Intermediate/` or `DerivedDataCache/` get weird.

---

## CI (when you're ready)

For a portfolio slice, even "build succeeded locally and produced this artifact" recorded on a GitHub Release is enough. Don't over-invest. If you do want a green checkmark on `main`:

1. Self-hosted Windows runner (GitHub-hosted runners can't build UE — too small).
2. Workflow: on push to a feature branch, headless compile `Development Editor` via `Engine\Build\BatchFiles\RunUAT.bat BuildEditor -project=OutBreakFPS.uproject -platform=Win64`.
3. On tag push: cook & package for both PC and VR configurations, upload `.zip` artifacts to the Release.

The CI itself is a portfolio artifact — a sensible, minimal pipeline reads as more senior than an over-engineered one.
