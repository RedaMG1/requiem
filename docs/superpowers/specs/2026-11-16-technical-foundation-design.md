# Requiem — Technical Foundation (M0) Design

Date: 2026-11-17 (revision 2 — replaces the first draft, which targeted a web game built with Babylon.js)
Status: Draft, awaiting owner approval

## 1. Intent and why

**Working title:** *Requiem* (repo `requiem`). A codename; the final title may change before release.

An original 3D turn-based RPG, inspired by the *style of combat* of Clair Obscur: Expedition 33: turn-based actions combined with real-time dodge, parry and timed attacks. Setting, characters, story, art and music are original.

- **Platform:** a native, downloadable **PC game (Windows first)**, eventually sold on Steam. A native mobile version (iOS/Android app) may come later; nothing in M0 may block it.
- **Visual direction:** **cel-shaded anime** (in the family of Persona and Genshin Impact). Details belong to a separate art direction design.
- **Hardware stance:** neither the developer nor the players should need an expensive PC.
- **Game scope context** (designed separately later): an overworld map or hub connecting zones, released in acts; the first real target is Act 1, about 2–3 hours of play.

The owner is a solo developer: an experienced programmer aiming for a DevOps career, and a guitarist and composer. Budget is $0 for now, with no deadline. The project is also meant to showcase clean engineering and DevOps practice.

This document covers only the **technical foundation (M0)** every later milestone builds on.

## 2. Goals, non-goals, anti-goals

### Goals of M0
1. A Unity 6 LTS project (URP) with a pinned version, a clear folder structure, and assembly boundaries that enforce the architecture.
2. A pure C# core library (no Unity references) with tests that run both inside Unity and with plain `dotnet test`.
3. A test scene with a placeholder battle layout and quality presets, running smoothly on modest hardware.
4. An input layer (keyboard, mouse, gamepad) producing timestamped game actions — the base for parry/dodge timing.
5. A UI Toolkit debug HUD proving the UI → core → scene path.
6. The private assets repository wired in as a submodule, proven by one placeholder asset that appears in local and CI builds.
7. CI/CD: every pull request runs code checks, tests and a Windows build; `main` additionally uploads the build to a private itch.io page.
8. Project context documented (README, CLAUDE.md, setup checklist, architecture decision records).

### Non-goals (later milestones)
- Combat rules, turn order, AP, damage, timing judgement (M1).
- Real art, toon shader, characters, animations, VFX (art direction design).
- Audio pipeline and the Unity-audio-vs-FMOD decision (audio design).
- Story, exploration, save system, settings menu, key rebinding UI.
- Steam release, Mac/Linux/Steam Deck builds, mobile builds.
- Managing GitHub/itch.io configuration as code with Terraform (milestone **M0.5**, right after M0).

### Anti-goals (would make M0 a failure even if it "works")
- Any build, test or release step that only works by clicking in the Unity editor.
- Core/game logic that references `UnityEngine` (it must stay testable without Unity).
- Third-party (Asset Store) assets, the owner's music, or art source files committed to the public repo.
- Secrets or Unity license files committed anywhere.
- Unity version drift (the project silently upgraded by opening it in another Unity version).
- Committing Unity's generated folders (`Library/`, `Temp/`, `Logs/`, `UserSettings/`, generated `.csproj`/`.sln`).
- A test scene that only runs well on a high-end PC.

## 3. Constraints

- **Cost:** $0. Unity Personal (free under its revenue threshold), GitHub Free, itch.io (free).
- **Repos:** public code repo `RedaMG1/requiem` (all rights reserved); private assets repo `RedaMG1/requiem-private`.
- **Storage limits:** GitHub Free includes 10 GiB of Git LFS storage and 10 GiB of LFS bandwidth per month, shared across the account. CI must cache LFS files to protect the bandwidth quota.
- **Target platform:** Windows 10/11, 64-bit.
- **Player minimum spec target:** GTX 1050-class GPU or recent integrated graphics (Intel Iris Xe / Radeon 680M class), 8 GB RAM, at the Low preset.
- **Development machine:** the owner's PC (good specs, but nothing may *require* high-end hardware).
- **Mobile later:** the render pipeline and architecture must not rule out a native mobile build.

## 4. Approaches considered

### 4.1 Platform — chosen: **native PC** (previously: web)
The first draft targeted the browser (Babylon.js) for instant play from a link. The owner changed direction: a "real" PC game, eventually sold on Steam, with a higher graphics ceiling. Native also allows a native mobile app later, with much better performance than mobile web. Trade-off: no instant play from a link (itch.io downloads cover playtesting).

### 4.2 Engine — chosen: **Unity 6 (LTS)**
| Option | Why not chosen / why chosen |
|---|---|
| **Unity 6** ✅ | Best anime pipeline (VRoid → UniVRM, free toon shaders such as lilToon and Unity-Chan Toon Shader), strong cinematic tools (Timeline, Cinemachine), mature animation, huge asset/tutorial ecosystem, proven PC-plus-mobile path (Genshin Impact). Runs on modest hardware with URP. Downsides: proprietary licence with a pricing controversy in 2023 (reversed), messier scene merges, CI needs licence activation (solved by GameCI). |
| Godot 4 | Free/open source, very light, cleanest Git workflow. Close second; less mature anime pipeline and cinematic tooling. |
| Unreal Engine 5 | Best realism (Clair Obscur's engine), MetaHuman. Rejected: needs a strong PC, steep learning curve, heavy CI, and realism was dropped in favour of cel-shaded anime. |

### 4.3 Visual style (high level) — chosen: **cel-shaded anime**
Over stylized realism: cheaper to produce solo, runs on modest PCs and later on phones, unifies assets from different sources, ages well. Detailed choices (toon shader, outlines, palette) go to the art direction design.

### 4.4 Asset storage — chosen: **public code repo + private assets repo (Git submodule with Git LFS)**
| Option | Trade-off |
|---|---|
| Everything in the public repo | Violates Asset Store licences (no redistribution); publishes the owner's music and art. Rejected. |
| **Public code + private assets submodule** ✅ | Respects licences, protects creative work, keeps full version history, keeps free CI/security tools for the public code. CI needs a read-only token to fetch the assets. |
| Assets in Cloudflare R2 synced by script | Custom tooling, fragile with Unity's `.meta` files, history only with bucket versioning. |
| Everything private | Loses free security scanning, portfolio value, and unlimited CI minutes. |

When the 10 GiB LFS quota gets tight, asset storage moves to a cheaper backend (a later DevOps milestone).

### 4.4b Where the Unity CI runs — chosen: **in the private repo**
| Option | Trade-off |
|---|---|
| **Unity jobs in `requiem-private`, reporting status to the public PR** ✅ | Private builds and caches, works when the owner's PC is off, free (2,000 min/month, enough for solo use), strong DevOps showcase (cross-repo pipeline). |
| Self-hosted runner on the owner's PC | Free, unlimited, fastest; only safe on a private repo; unavailable when the PC is off. Added later on top of the chosen option. |
| Public repo without artifacts or private caches | Fragile (one mistake leaks assets), slow, burns the LFS bandwidth quota. Rejected. |

### 4.4c Who implements — chosen: **Claude Code running on the owner's Windows PC**
| Option | Trade-off |
|---|---|
| **Claude Code on the owner's PC** ✅ | Unity is installed there, so editor assets can be generated by editor scripts in batch mode, and tests/builds can be run and checked directly. Owner checks visual and fps results and does one-time account setup. |
| A cloud session without Unity, owner doing editor steps by hand | Slower, many manual steps, more back-and-forth. |
| Owner builds, Claude advises | Most effort for the owner. |

### 4.5 UI — chosen: **UI Toolkit for menus and HUD; world-space uGUI or meshes for UI attached to 3D objects**
| Option | Trade-off |
|---|---|
| **UI Toolkit** ✅ (menus, HUD) | Layout files plus CSS-like stylesheets — familiar to a web developer; scales to inventories and skill trees; Unity's recommended system for new projects. Less mature for UI attached to 3D objects. |
| uGUI ✅ (in-world only) | Mature world-space canvases for enemy HP bars and damage numbers; messier for large menu systems. |

### 4.6 Repository visibility and licence
Public code (free CodeQL, secret scanning, unlimited Actions minutes on standard runners, portfolio value). **All rights reserved:** a `LICENSE` file states that the code is visible but may not be reused without permission. Contributions are not accepted for now.

## 5. Decided technical design

### 5.1 Stack
| Area | Choice |
|---|---|
| Engine | Unity 6, latest LTS at setup time, exact version pinned (`ProjectSettings/ProjectVersion.txt`) |
| Render pipeline | Universal Render Pipeline (URP) |
| Input | Unity Input System package |
| UI | UI Toolkit (menus/HUD); uGUI world-space later for in-world UI |
| Language | C# (Unity's supported language version) |
| Core library | Pure C# assembly with no engine references, also compiled by an SDK-style `.csproj` targeting `netstandard2.1` so it builds and tests without Unity |
| Tests | Unity Test Framework (NUnit) for EditMode/PlayMode; plain `dotnet test` (NUnit) for the core library |
| Code quality | `.editorconfig`, .NET Roslyn analyzers (warnings as errors in the core library), `dotnet format --verify-no-changes` in CI |
| Git hooks | Lefthook: `dotnet format` on staged `.cs` files |
| Commits | Conventional Commits |
| Version control | Git; Git LFS in the assets repo; Unity set to "Visible Meta Files" and "Force Text" serialization; UnityYAMLMerge documented as the merge tool for scenes and prefabs |
| CI/CD | GitHub Actions + GameCI (Unity test runner and builder) |
| Playtest distribution | itch.io (restricted page), uploaded by `butler` from CI |
| Dependency updates | Renovate for GitHub Actions and NuGet (core test project); Unity packages updated manually on a schedule |
| Security scanning | CodeQL for C# (no-build mode), GitHub secret scanning with push protection, Dependabot alerts |
| IDE | Owner's choice (Rider, free for non-commercial use, or Visual Studio Community) |

### 5.2 Repository layout
```
requiem/                         (public)
  Assets/
    _Project/
      Scripts/
        Core/          Requiem.Core.asmdef          pure C#: clock, event bus, state machines; later combat rules
        Input/         Requiem.Input.asmdef         Input System → core actions
        Presentation/  Requiem.Presentation.asmdef  scene, camera, debug wiring
        UI/            Requiem.UI.asmdef            UI Toolkit code
        App/           Requiem.App.asmdef           composition root: creates Core services and wires Input, Presentation and UI together
      Scenes/
      Settings/        URP assets, quality presets
      UI/              UXML layouts and USS stylesheets
      Tests/
        EditMode/      Requiem.Tests.EditMode.asmdef
        PlayMode/      Requiem.Tests.PlayMode.asmdef
    Art/               ← Git submodule: requiem-private (private, LFS) — art, audio, third-party assets, with their .meta files
  Packages/            manifest.json, packages-lock.json
  ProjectSettings/
  dotnet/
    Requiem.Core.csproj          compiles Assets/_Project/Scripts/Core/**/*.cs
    Requiem.Core.Tests.csproj    NUnit tests for the core library
  docs/  adr/  superpowers/specs/  setup.md
  .claude/settings.json          declares the Superpowers plugin for sessions on this repo
```

**Dependency rules (enforced by assembly definitions, so violations are compile errors):**
- `Requiem.Core` references nothing and has engine references disabled (`noEngineReferences`). The `dotnet` build of the same files proves it.
- `Input`, `Presentation` and `UI` may reference `Core`.
- `UI` and `Presentation` must not reference each other; they communicate only through `Core` (events and shared state).
- `App` is the only assembly that references all of them; it builds the object graph at startup (composition root). *Builder's call, recorded.*
- Unity tests live under `Assets/` (Unity only compiles test assemblies inside `Assets/`). *Builder's call, recorded.*

The project must still open without the submodule (missing art shows as missing references); only the owner and CI have the assets.

### 5.3 Test scene
- URP, a ground plane, a directional light with ambient light, a camera framing a battle layout: 3 placeholder heroes (capsules) and 1 placeholder enemy, gently idle-animated.
- One placeholder material or texture comes from the private assets submodule, proving the asset path end to end.
- **Quality presets:** Low (no real-time shadows or post-processing, reduced render scale), Medium, High (shadows, bloom). Selectable from the debug HUD.
- Target 60 fps with VSync; the scene pauses when the window loses focus (builder's call whether to pause or throttle).

### 5.4 Input layer
- An Input System actions asset with a `Gameplay` map: `Confirm`, `Back`, `Parry`, `Dodge`, `Navigate`; bindings for keyboard, mouse and gamepad.
- The input layer converts these into core actions carrying the **Input System event timestamp** (not the frame time), so later timing judgement does not depend on frame rate.
- Default keyboard bindings: Parry = J / Space, Dodge = K / Shift, Confirm = Enter, Back = Esc. Gamepad mapping is the builder's call.

### 5.5 Debug HUD (UI Toolkit)
- Shows FPS, frame time, quality preset (with a selector), last input action and its timestamp.
- One sample control (e.g. "highlight next hero") sends an event through `Core`, and the scene reacts — proving UI → Core → Presentation.
- Included in Development builds only (compiled out of release builds).

### 5.6 CI/CD
CI is split so that **nothing containing private content ever runs, is cached, or is stored in the public repo** (on a public repo, anyone signed in can download workflow artifacts, and fork pull requests can restore base-branch caches).

**Public repo `requiem` — fast code checks only (no Unity, no assets):**
1. `core` job (Ubuntu): `dotnet format --verify-no-changes`, build the core library with analyzers as errors, `dotnet test`.
2. CodeQL for C#.
3. `dispatch` job: for pull requests from branches of this repo and pushes to `main`, sends a `repository_dispatch` event (commit SHA, PR number) to `requiem-private`. Fork pull requests get no secrets, so they run only jobs 1–2.

**Private repo `requiem-private` — holds the assets and runs everything that needs them:**
4. Triggered by the dispatch: checks out the public repo at the given SHA, places its own content at `Assets/Art`, restores private caches (LFS objects, `Library/`).
5. `unity-tests` (GameCI test runner, EditMode + PlayMode) and `build-windows` (GameCI builder, `StandaloneWindows64`, Mono backend; IL2CPP evaluated later). The build is stored as a workflow artifact **in the private repo** (short retention).
6. Reports a commit status back to the public pull request (e.g. `unity / tests`, `unity / build-windows`), so the PR shows pass/fail and branch protection can require it.
7. On `main`: `butler push` to a restricted itch.io page (channel `windows-dev`, Development build with the debug HUD), versioned from the commit.
8. Only runs Unity jobs when Unity-relevant paths changed, to stay within the private repo's 2,000 free Actions minutes per month.

**Later:** a self-hosted runner on the owner's PC, attached to `requiem-private` only (never to the public repo), for native Windows/IL2CPP builds and unlimited minutes.

**Workflow security:** default `permissions: contents: read`; third-party actions pinned to commit SHAs (Renovate keeps them updated); fine-grained, least-privilege tokens; nothing private in public logs.

**Secrets needed** (created manually, see `docs/setup.md`):
- In `requiem`: `PRIVATE_DISPATCH_TOKEN` (fine-grained, can trigger workflows on `requiem-private` only).
- In `requiem-private`: Unity Personal licence credentials as required by GameCI, `BUTLER_API_KEY`, `PUBLIC_STATUS_TOKEN` (fine-grained, commit statuses write on `requiem` only).

### 5.7 Performance and size
- M0 records a baseline in the README: frame time at Low/Medium/High on the owner's PC, and Windows build size.
- *Builder's call, recorded:* frame times are measured with VSync off (a benchmark toggle in the debug HUD), using CPU and GPU frame timings, so the numbers show real headroom rather than a capped 60 fps.
- The Low preset must stay comfortably within a 60 fps budget on the owner's PC with headroom, as a proxy for the minimum spec; a real low-end test happens before the first public playtest.

### 5.8 Developer workflow
- **Implementer:** Claude Code on the owner's Windows PC, in the local clone. Editor-made assets (URP and quality assets, the test scene, the Input Actions asset, project settings) are created by editor scripts run with `Unity -batchmode -executeMethod ...` wherever possible; hand-writing Unity YAML is a last resort. The owner performs the one-time setup in `docs/setup.md` and the Windows checks in acceptance items 2, 3 and 5.
- Unity Hub with the pinned Unity version; Git for Windows with Git LFS.
- Clone with `git clone --recurse-submodules`.
- Short-lived branches, pull requests into `main`, Conventional Commits.

### 5.9 Documentation delivered in M0
- `README.md` — what the project is, how to open and run it, baselines.
- `LICENSE` — all rights reserved.
- `CLAUDE.md` — project context and working rules for AI sessions, including: always present options as a comparison with a recommendation for this project before asking the owner to choose.
- `docs/setup.md` — the one-time manual setup checklist: create the private assets repo and LFS, create the read-only token, Unity licence secrets, itch.io account and API key, branch protection, enabling security features.
- `docs/adr/` — ADR-001 Native PC with Unity 6 (supersedes the web/Babylon.js direction), ADR-002 Cel-shaded anime direction, ADR-003 Public code + private assets submodule, ADR-004 UI Toolkit + world-space hybrid, ADR-005 Pure C# core with dual test runners, ADR-006 GitHub Actions + GameCI + itch.io.
- `.claude/settings.json` — declares the Superpowers plugin marketplace and plugin.

## 6. Done when (M0 acceptance)
1. A fresh `git clone --recurse-submodules` opens in the pinned Unity version without errors; core tests pass with `dotnet test` and in Unity's Test Runner.
2. The test scene runs at a steady 60 fps on the owner's PC; Low/Medium/High frame times are recorded in the README.
3. Keyboard, mouse and gamepad actions appear in the HUD with timestamps; the sample control visibly changes the scene through `Core`.
4. Adding a `UnityEngine` reference to `Core`, or a `UI` → `Presentation` reference, fails compilation.
5. A pull request shows the public checks and the Unity statuses reported from `requiem-private`; the Windows build artifact (in the private repo) downloads and runs on Windows.
6. A merge to `main` updates the restricted itch.io page.
7. The placeholder asset from the private submodule appears in local and CI builds; the public repo contains no third-party or private assets, and no public workflow run has access to private content (no artifacts, caches or logs containing it).
8. README, LICENSE, CLAUDE.md, setup checklist and the six ADRs exist.

## 7. Left to the builder
- Exact Unity, package and tool versions (latest stable at setup, then pinned).
- File names inside each module; the event bus implementation.
- Gamepad mapping; HUD styling; exact quality preset values.
- CI caching details and job structure, as long as the behaviour above holds.
- itch.io page details (title, restricted access settings).
