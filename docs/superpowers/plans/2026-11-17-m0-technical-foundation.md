# Requiem M0 — Technical Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use **superpowers:executing-plans** to implement this plan task-by-task (the owner chose native execution to minimise token use; one whole-branch review at the end). Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the Requiem Unity 6 project, its pure C# core, input layer, placeholder battle scene, debug HUD, private-asset wiring and split public/private CI/CD, so every later milestone builds on a tested, automated foundation.

**Architecture:** Game logic lives in a pure C# library (`Requiem.Core`) compiled both by Unity and by a plain .NET project, so it is tested with `dotnet test` and Unity's Test Runner. Unity-side assemblies (`Input`, `Presentation`, `UI.DebugHud`) talk only through Core's `EventBus`; `App` is the composition root. Private art lives in a private repo mounted as a submodule at `Assets/Art`; everything that touches it (Unity tests, builds, itch.io upload) runs in that private repo's CI, which reports statuses back to the public PR.

**Tech Stack:** Unity 6 LTS (URP, Input System, UI Toolkit, Unity Test Framework), C# 9, .NET SDK 10 (LTS) for `dotnet build/test/format`, NUnit 3.14, GitHub Actions, GameCI, butler (itch.io), Lefthook, Renovate.

**Spec:** `docs/superpowers/specs/2026-11-16-technical-foundation-design.md` (approved 2026-11-17). Read it before starting.

## Global Constraints

- **Where this runs:** Claude Code on the owner's Windows PC, in the local clone of `RedaMG1/requiem`, PowerShell. Unity is installed there.
- **Unity version:** latest Unity 6 **LTS** at setup time, pinned in `ProjectSettings/ProjectVersion.txt`; never open the project in another version.
- **Batch mode:** all Unity work runs through `tools/unity.ps1` (Task 4). Close the Unity editor before running it (an open editor locks the project).
- **No editor clicking for anything reproducible:** assets/settings are created by editor scripts in `Requiem.EditorTools`; hand-written Unity YAML only as a last resort. JSON assets (`.inputactions`, `.asmdef`) may be hand-written.
- **C# language version 9.0** everywhere (Unity 6's level); `<LangVersion>9.0</LangVersion>` in `dotnet/` projects.
- **`Requiem.Core` uses no `UnityEngine`/`UnityEditor`** and targets `netstandard2.1`.
- **Tests use only the NUnit constraint model** (`Assert.That(..., Is....)`) so the same files compile under Unity's NUnit and NUnit **3.14.0** in `dotnet/`.
- **Number formatting for display uses `CultureInfo.InvariantCulture`.**
- **Quality preset names and order are exactly:** `Low`(0), `Medium`(1), `High`(2), `Ultra`(3). Default level for Windows: `Medium`.
- **Default keyboard bindings exactly:** Parry = `J` / `Space`; Dodge = `K` / `Left Shift`; Confirm = `Enter`; Back = `Escape`.
- **Input timestamps** are the Input System event time (`InputAction.CallbackContext.time`, seconds, `double`), never `Time.time`.
- **Repos:** public `RedaMG1/requiem`; private `RedaMG1/requiem-private`, mounted at `Assets/Art`.
- **Secret/variable names exactly:** public repo — `PRIVATE_DISPATCH_TOKEN`; private repo — `UNITY_LICENSE`, `UNITY_EMAIL`, `UNITY_PASSWORD`, `BUTLER_API_KEY`, `PUBLIC_STATUS_TOKEN`, variable `ITCH_TARGET` (e.g. `username/requiem`).
- **Commit status contexts exactly:** `unity / tests`, `unity / build-windows`.
- **Public workflows:** default `permissions: contents: read`; every third-party action pinned to a commit SHA; no `actions/upload-artifact`, no `actions/cache` of anything under `Assets/Art` or `Library/`.
- **Never commit:** `Library/`, `Temp/`, `Obj/`, `Logs/`, `UserSettings/`, `Builds/`, generated `*.csproj`/`*.sln` at repo root, `*.ulf`/`*.alf` licence files, secrets.
- **Git:** one branch per task (`m0/task-N-<slug>`), Conventional Commits, PR into `main`.
- **Owner checkpoints** are marked **🧑 OWNER**: stop and ask the owner to do that step; continue only when they confirm.

## Review Focus

1. **PR with no Unity changes** (docs or `dotnet/` only): the required `unity / tests` and `unity / build-windows` statuses must still be reported (as success, "skipped: no Unity changes"), or the PR can never merge. → Task 9.
2. **Clone without the submodule** (any public visitor, or `git clone` without `--recurse-submodules`): the project must still compile with zero script errors; only the placeholder material shows as missing. → Task 7.
3. **Non-English Windows locale** (e.g. French: decimal comma): HUD numbers must render with a dot and stable formatting. → Task 8.
4. **Analog stick noise around the threshold:** small stick jitter must not spam `Navigate` actions; one action per direction change. → Task 5.
5. **Event handlers that unsubscribe or publish during a publish:** no "collection modified" exception, no skipped or double-called handlers. → Task 2.

---

## File Structure

```
.gitignore  .gitattributes  .editorconfig  LICENSE  README.md  CLAUDE.md
global.json  lefthook.yml  renovate.json  .gitmodules
.claude/settings.json
docs/setup.md
docs/adr/0001-native-pc-unity6.md … 0006-github-actions-gameci-itch.md
tools/unity.ps1                                   batch-mode wrapper (tests, execute method, build)
tools/ci/verify-public-workflows.sh               guards: no artifacts/caches/unpinned actions in public CI
.github/workflows/ci.yml  .github/workflows/codeql.yml
dotnet/Requiem.sln
dotnet/Requiem.Core/Requiem.Core.csproj           compiles Assets/_Project/Scripts/Core/**/*.cs
dotnet/Requiem.Core.Tests/Requiem.Core.Tests.csproj  compiles Assets/_Project/Tests/Core/**/*.cs
Packages/manifest.json  Packages/packages-lock.json  ProjectSettings/**
Assets/_Project/
  Scripts/Core/          Requiem.Core.asmdef  EventBus.cs  GameAction.cs  Messages.cs  HeroHighlightService.cs
  Scripts/Input/         Requiem.Input.asmdef  GameplayActionNames.cs  GameActionMapper.cs  NavigateDirectionFilter.cs  UnityInputSource.cs
  Scripts/Presentation/  Requiem.Presentation.asmdef  BattleLayoutView.cs  QualityPresetApplier.cs  FrameStatsSampler.cs
  Scripts/UI/DebugHud/   Requiem.UI.DebugHud.asmdef  DebugHudPresenter.cs  DebugHudController.cs
  Scripts/App/           Requiem.App.asmdef  AppBootstrap.cs
  Editor/                Requiem.EditorTools.asmdef  ProjectSetup.cs  SceneBuilder.cs  PlaceholderArtBuilder.cs  BuildScript.cs
  Settings/              URP_Low/Medium/High/Ultra.asset (+ renderer data)  Input/Requiem.inputactions
  Scenes/BattleTest.unity
  UI/DebugHud.uxml  UI/DebugHud.uss  UI/DebugHudPanelSettings.asset
  Tests/Core/            Requiem.Core.Tests.asmdef      (pure; shared with dotnet)
  Tests/EditMode/        Requiem.Tests.EditMode.asmdef
  Tests/PlayMode/        Requiem.Tests.PlayMode.asmdef
Assets/Art/  → submodule RedaMG1/requiem-private (private repo; its own .gitattributes, README, .github/workflows/unity-ci.yml, Placeholders/)
```

---

### Task 1: Repository foundation and project documents

**Files:**
- Create: `.gitignore`, `.gitattributes`, `.editorconfig`, `LICENSE`, `README.md`, `CLAUDE.md`, `.claude/settings.json`, `lefthook.yml`, `renovate.json`, `docs/setup.md`, `docs/adr/0001-native-pc-unity6.md`, `0002-cel-shaded-anime.md`, `0003-public-code-private-assets.md`, `0004-ui-toolkit-hybrid.md`, `0005-pure-csharp-core-dual-tests.md`, `0006-github-actions-gameci-itch.md`

**Interfaces:**
- Produces: `docs/setup.md` sections referenced by later 🧑 OWNER checkpoints: **§1 Tools** (Unity Hub + pinned Unity 6 LTS with Windows Build Support, Git for Windows, Git LFS, .NET SDK 10, Lefthook, actionlint; env var `UNITY_EDITOR` = full path to `Unity.exe`), **§2 Private assets repo** (create `RedaMG1/requiem-private`, private, **initialized with a README** so it has a first commit; Git LFS installed locally), **§3 Public repo settings** (fine-grained token `PRIVATE_DISPATCH_TOKEN` with Actions + Contents read/write on `requiem-private` only; branch protection on `main` requiring `core`, `CodeQL`, `unity / tests`, `unity / build-windows`; secret scanning + push protection; Dependabot alerts; "Require approval for all outside collaborators" on Actions), **§4 Private repo CI** (Unity Personal licence → `UNITY_LICENSE`/`UNITY_EMAIL`/`UNITY_PASSWORD` per GameCI's activation guide; `PUBLIC_STATUS_TOKEN` with Commit statuses write on `requiem` only; itch.io account, restricted project page, `BUTLER_API_KEY`, variable `ITCH_TARGET`).

- [ ] **Step 1: Write `.gitignore`** — Unity's standard ignore set plus: `/Builds/`, `/Temp/`, `/Library/`, `/Obj/`, `/Logs/`, `/UserSettings/`, `/*.csproj`, `/*.sln`, `*.ulf`, `*.alf`, `dotnet/**/bin/`, `dotnet/**/obj/`, `.vs/`, `.idea/`.

- [ ] **Step 2: Verify ignore rules**

Run: `git check-ignore -v Library/x Temp/x Builds/x UserSettings/x Requiem.sln Assembly-CSharp.csproj Unity_v6.ulf dotnet/Requiem.Core/bin/x`
Expected: every path printed with a matching rule. Then `git check-ignore dotnet/Requiem.sln Assets/x.cs` prints nothing (exit code 1).

- [ ] **Step 3: Write `.gitattributes`** — `* text=auto`; `*.cs diff=csharp`; Unity YAML (`*.unity *.prefab *.asset *.mat *.anim *.controller *.meta`) as `text eol=lf merge=unityyamlmerge`; `*.inputactions *.asmdef *.json *.uxml *.uss` text. No LFS patterns in the public repo.

- [ ] **Step 4: Write `.editorconfig`** — C# conventions (4-space indent, `file_scoped` namespaces **off** (C# 9), `_camelCase` private fields, `dotnet_analyzer_diagnostic.category-Style.severity = warning` for `dotnet/` builds).

- [ ] **Step 5: Write `LICENSE`** — "Copyright (c) 2026 RedaMG1. All rights reserved." plus: the source is published for viewing only; no permission is granted to copy, modify, distribute or use it, in whole or part, without written permission; contributions are not accepted.

- [ ] **Step 6: Write `CLAUDE.md`** — project summary (from spec §1), stack, layout and dependency rules (spec §5.2), the Global Constraints above, how to run tests/builds (`tools/unity.ps1`, `dotnet test dotnet/Requiem.sln`), and the working rule verbatim: *"When offering choices, always present them as a comparison table with trade-offs and a recommendation for this project before asking the owner to choose."*

- [ ] **Step 7: Write `.claude/settings.json`**

```json
{
  "extraKnownMarketplaces": {
    "superpowers-marketplace": { "source": { "source": "github", "repo": "obra/superpowers-marketplace" } }
  },
  "enabledPlugins": { "superpowers@superpowers-marketplace": true }
}
```

- [ ] **Step 8: Write `docs/setup.md`** with §1–§4 as listed under Interfaces, each step as a checkbox with where to click/what to run, plus "Clone with `git clone --recurse-submodules`" and "Configure UnityYAMLMerge as the git merge driver `unityyamlmerge`" (path from the Unity install).

- [ ] **Step 9: Write the six ADRs** (Context / Decision / Alternatives / Consequences), content from spec §4.1–4.6 and §5.6. ADR-0001 states it supersedes the web/Babylon.js direction.

- [ ] **Step 10: Write `README.md`** — what Requiem is, requirements, clone/open instructions, test/build commands, and an empty "Performance baseline" table (Preset | CPU ms | GPU ms | FPS) plus "Build size" filled in Task 11.

- [ ] **Step 11: Write `lefthook.yml`** (pre-commit: `dotnet format dotnet/Requiem.sln --include {staged_files}` on `*.cs`, `stage_fixed: true`) and `renovate.json` (`config:recommended`, `helpers:pinGitHubActionDigests`, NuGet enabled, Unity packages ignored).

- [ ] **Step 12: Validate JSON and hook**

Run: `Get-Content .claude/settings.json -Raw | ConvertFrom-Json | Out-Null; Get-Content renovate.json -Raw | ConvertFrom-Json | Out-Null; lefthook install; lefthook run pre-commit`
Expected: no errors (pre-commit runs nothing yet — no `.cs` files).

- [ ] **Step 13: Commit**

```bash
git add -A && git commit -m "chore: add repository foundation, licence and project docs"
```

---

### Task 2: Core library — event bus and messages

**Files:**
- Create: `global.json`, `dotnet/Requiem.sln`, `dotnet/Requiem.Core/Requiem.Core.csproj`, `dotnet/Requiem.Core.Tests/Requiem.Core.Tests.csproj`
- Create: `Assets/_Project/Scripts/Core/EventBus.cs`, `GameAction.cs`, `Messages.cs`
- Test: `Assets/_Project/Tests/Core/EventBusTests.cs`, `GameActionTests.cs`

**Interfaces:**
- Produces (namespace `Requiem.Core`):
  - `public sealed class EventBus` — `IDisposable Subscribe<T>(Action<T> handler)`; `void Publish<T>(T message)`. Synchronous; handlers run in subscription order; each publish iterates a snapshot; a handler unsubscribed earlier in the same publish is not called; a handler subscribed during a publish is not called for that message; if handlers throw, the rest still run and an `AggregateException` of all thrown exceptions is raised afterwards; `Subscribe(null)` throws `ArgumentNullException`; disposing a subscription twice is a no-op.
  - `public enum GameActionType { Confirm, Back, Parry, Dodge, Navigate }`
  - `public readonly struct GameAction` — ctor `(GameActionType type, double timestamp, int navigateX = 0, int navigateY = 0)`; properties `Type`, `Timestamp`, `NavigateX`, `NavigateY`.
  - `public enum QualityPreset { Low = 0, Medium = 1, High = 2, Ultra = 3 }`
  - Message structs (readonly, ctor + get-only properties): `HighlightNextHeroRequested` (no fields), `HeroHighlightChanged(int HeroIndex)`, `QualityPresetRequested(QualityPreset Preset)`, `QualityPresetChanged(QualityPreset Preset)`, `BenchmarkModeRequested(bool Enabled)`, `FrameStats(double Fps, double CpuFrameMs, double GpuFrameMs, bool VSyncEnabled)` — `GpuFrameMs` is `double.NaN` when unavailable.

- [ ] **Step 1: Create the .NET projects**
  - `global.json`: SDK `10.0.100`, `rollForward: latestFeature`.
  - `Requiem.Core.csproj`: `netstandard2.1`, `LangVersion 9.0`, `EnableDefaultCompileItems=false`, `<Compile Include="../../Assets/_Project/Scripts/Core/**/*.cs" />`, `TreatWarningsAsErrors=true`, `AnalysisLevel=latest-recommended`, `EnforceCodeStyleInBuild=true`.
  - `Requiem.Core.Tests.csproj`: `net10.0`, `LangVersion 9.0`, `EnableDefaultCompileItems=false`, `<Compile Include="../../Assets/_Project/Tests/Core/**/*.cs" />`, packages NUnit `3.14.0`, NUnit3TestAdapter, Microsoft.NET.Test.Sdk (latest stable), ProjectReference to Core.
  - `dotnet new sln -o dotnet -n Requiem` and add both.

- [ ] **Step 2: Write the failing tests** in `EventBusTests.cs` (namespace `Requiem.Core.Tests`):

```csharp
[Test] public void Publish_WithNoSubscribers_DoesNothing() =>
    Assert.That(() => new EventBus().Publish(new HeroHighlightChanged(1)), Throws.Nothing);

[Test] public void Publish_CallsHandlersInSubscriptionOrder() { /* subscribe A then B; publish; Assert.That(calls, Is.EqualTo(new[] { "A", "B" })); */ }

[Test] public void Publish_OnlyReachesHandlersOfThatMessageType() { /* FrameStats handler not called for HeroHighlightChanged */ }

[Test] public void DisposedSubscription_IsNotCalled_AndDoubleDisposeIsNoOp() { /* dispose twice, publish, count == 0 */ }

[Test] public void HandlerUnsubscribingItselfDuringPublish_DoesNotThrowOrSkipOthers() { /* A disposes own subscription inside handler; B still called once; second publish: only B */ }

[Test] public void HandlerUnsubscribingALaterHandler_PreventsThatCallInSamePublish() { /* A disposes C's subscription; C not called */ }

[Test] public void HandlerSubscribedDuringPublish_IsNotCalledForCurrentMessage() { /* called on next publish only */ }

[Test] public void HandlerPublishingAnotherMessage_DeliversNestedMessageImmediately() { /* order: outer-start, inner, outer-end */ }

[Test] public void ThrowingHandler_DoesNotStopOthers_AndAggregatesExceptions() {
    // two throwing handlers + one normal: normal called; Assert.That(ex.InnerExceptions.Count, Is.EqualTo(2));
}

[Test] public void Subscribe_Null_Throws() =>
    Assert.That(() => new EventBus().Subscribe<FrameStats>(null), Throws.ArgumentNullException);
```

In `GameActionTests.cs`: `GameAction_PreservesTimestampExactly` (`12.345678901234` round-trips), `GameAction_DefaultsNavigateToZero`, `QualityPreset_ValuesMatchQualityLevelIndices` (`(int)QualityPreset.Ultra == 3`).

- [ ] **Step 3: Run tests to verify they fail**

Run: `dotnet test dotnet/Requiem.sln`
Expected: build FAILS — `EventBus`, `GameAction`, messages not defined.

- [ ] **Step 4: Implement `EventBus`, `GameAction`, `Messages`** per the Interfaces block (one file each; messages in `Messages.cs`). EventBus: `Dictionary<Type, List<Subscription>>`; publish copies the list, skips subscriptions marked disposed.

- [ ] **Step 5: Run tests to verify they pass**

Run: `dotnet test dotnet/Requiem.sln`
Expected: all tests PASS, 0 warnings.

- [ ] **Step 6: Verify formatting**

Run: `dotnet format dotnet/Requiem.sln --verify-no-changes`
Expected: exit code 0.

- [ ] **Step 7: Commit**

```bash
git add global.json dotnet Assets/_Project/Scripts/Core Assets/_Project/Tests/Core
git commit -m "feat(core): add event bus, game actions and core messages"
```

---

### Task 3: Core — hero highlight service

**Files:**
- Create: `Assets/_Project/Scripts/Core/HeroHighlightService.cs`
- Test: `Assets/_Project/Tests/Core/HeroHighlightServiceTests.cs`

**Interfaces:**
- Consumes: `EventBus`, `HighlightNextHeroRequested`, `HeroHighlightChanged` (Task 2).
- Produces: `public sealed class HeroHighlightService : IDisposable` — ctor `(EventBus bus, int heroCount)`, throws `ArgumentOutOfRangeException` if `heroCount < 1`; `int HighlightedIndex { get; }` starts at 0; `void PublishCurrent()` publishes `HeroHighlightChanged(HighlightedIndex)`; on `HighlightNextHeroRequested` advances `(index + 1) % heroCount` and publishes; `Dispose()` unsubscribes.

- [ ] **Step 1: Write the failing tests**

```csharp
[Test] public void StartsAtZero() => Assert.That(new HeroHighlightService(new EventBus(), 3).HighlightedIndex, Is.EqualTo(0));
[Test] public void Request_AdvancesAndPublishes() { /* publish request; Assert.That(received, Is.EqualTo(new[] { 1 })); */ }
[Test] public void Request_WrapsAroundAfterLastHero() { /* 3 requests with heroCount 3 → received {1,2,0} */ }
[Test] public void PublishCurrent_PublishesCurrentIndex() { /* received {0} */ }
[Test] public void Dispose_StopsReactingToRequests() { /* after Dispose, request → index stays 0, nothing received */ }
[Test] public void HeroCountBelowOne_Throws() =>
    Assert.That(() => new HeroHighlightService(new EventBus(), 0), Throws.TypeOf<ArgumentOutOfRangeException>());
```

- [ ] **Step 2: Run to verify failure** — `dotnet test dotnet/Requiem.sln` → build FAILS (`HeroHighlightService` missing).
- [ ] **Step 3: Implement `HeroHighlightService`** per Interfaces.
- [ ] **Step 4: Run to verify pass** — `dotnet test dotnet/Requiem.sln` → all PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(core): add hero highlight service"`

---

### Task 4: Unity project bootstrap

🧑 **OWNER:** complete `docs/setup.md` §1 (tools, `UNITY_EDITOR` env var) before this task.

**Files:**
- Create: `Packages/manifest.json`, `Packages/packages-lock.json`, `ProjectSettings/**` (generated), `tools/unity.ps1`
- Create: `Assets/_Project/Scripts/Core/Requiem.Core.asmdef`, `Assets/_Project/Tests/Core/Requiem.Core.Tests.asmdef`, `Assets/_Project/Editor/Requiem.EditorTools.asmdef`, `Assets/_Project/Editor/ProjectSetup.cs`, `Assets/_Project/Tests/EditMode/Requiem.Tests.EditMode.asmdef`
- Create (by `ProjectSetup`): `Assets/_Project/Settings/URP_Low.asset`, `URP_Medium.asset`, `URP_High.asset`, `URP_Ultra.asset` and their renderer data
- Test: `Assets/_Project/Tests/EditMode/ProjectSettingsTests.cs`

**Interfaces:**
- Produces:
  - `tools/unity.ps1` with parameters: `-Tests EditMode|PlayMode` (writes `Temp/TestResults-<mode>.xml`, exits non-zero on failure), `-Execute <Type.Method>` (runs `-executeMethod` then quits), `-Compile` (opens, compiles, quits; fails on compile errors). Always passes `-batchmode -nographics -projectPath <repo> -logFile -` and propagates Unity's exit code.
  - `Requiem.EditorTools.ProjectSetup.Run()` — idempotent.
  - Assembly names: `Requiem.Core` (`noEngineReferences: true`), `Requiem.Core.Tests` (refs `Requiem.Core`, `overrideReferences: true`, `precompiledReferences: ["nunit.framework.dll"]`, `includePlatforms: ["Editor"]`, `defineConstraints: ["UNITY_INCLUDE_TESTS"]`, `noEngineReferences: true` — if Unity's Test Runner does not discover its tests with this flag, set it to `false`; purity is still enforced by the `dotnet` build), `Requiem.EditorTools` (Editor only), `Requiem.Tests.EditMode` (Editor only, test assembly).

- [ ] **Step 1: Create the Unity project** — `& $env:UNITY_EDITOR -batchmode -quit -createProject "$env:TEMP\requiem-new"`, then move its `Assets`, `Packages`, `ProjectSettings` into the repo (keep existing `Assets/_Project`). Add to `Packages/manifest.json`: `com.unity.render-pipelines.universal`, `com.unity.inputsystem`, `com.unity.test-framework`, `com.unity.ugui` at the versions the installed editor lists as recommended; add `"testables": ["com.unity.inputsystem"]`.

- [ ] **Step 2: Write `tools/unity.ps1`** per Interfaces and run `./tools/unity.ps1 -Compile`. Expected: exit code 0; `ProjectSettings/ProjectVersion.txt` exists; `Packages/packages-lock.json` generated.

- [ ] **Step 3: Add the asmdefs** listed under Files. Run `./tools/unity.ps1 -Tests EditMode`. Expected: the Core tests from Tasks 2–3 are discovered and PASS (check the result XML lists `EventBusTests`).

- [ ] **Step 4: Write the failing tests** in `ProjectSettingsTests.cs`:

```csharp
[Test] public void SerializationIsForceText() => Assert.That(EditorSettings.serializationMode, Is.EqualTo(SerializationMode.ForceText));
[Test] public void MetaFilesAreVisible() => Assert.That(VersionControlSettings.mode, Is.EqualTo("Visible Meta Files"));
[Test] public void ColorSpaceIsLinear() => Assert.That(PlayerSettings.colorSpace, Is.EqualTo(ColorSpace.Linear));
[Test] public void ProductNameIsRequiem() => Assert.That(PlayerSettings.productName, Is.EqualTo("Requiem"));
[Test] public void InputSystemIsTheOnlyInputHandler() { /* read "activeInputHandler" from ProjectSettings via SerializedObject; Assert.That(value, Is.EqualTo(1)); */ }
[Test] public void FrameTimingStatsEnabled() => Assert.That(PlayerSettings.enableFrameTimingStats, Is.True);
[Test] public void QualityLevelsAreLowMediumHighUltra() => Assert.That(QualitySettings.names, Is.EqualTo(new[] { "Low", "Medium", "High", "Ultra" }));
[Test] public void EveryQualityLevelHasItsUrpAsset() { /* for i in 0..3: QualitySettings.GetRenderPipelineAssetAt(i) is UniversalRenderPipelineAsset named "URP_" + names[i] */ }
[Test] public void DefaultPipelineIsUrpMedium() { /* GraphicsSettings.defaultRenderPipeline name == "URP_Medium" */ }
[Test] public void WindowsDefaultQualityIsMedium() { /* default level for Standalone == 1 */ }
[Test] public void PresetsScaleUp() {
    // shadow distance and render scale are non-decreasing Low→Ultra; Low has supportsMainLightShadows == false or shadowDistance <= 20;
    // Low renderScale < 1; Ultra renderScale >= 1; post-processing disabled on Low renderer, enabled on High/Ultra.
}
```

- [ ] **Step 5: Run to verify failure** — `./tools/unity.ps1 -Tests EditMode` → the new tests FAIL.

- [ ] **Step 6: Implement `ProjectSetup.Run()`** — sets everything the tests assert. Quality levels via `SerializedObject` on `ProjectSettings/QualitySettings.asset` (`m_QualitySettings` array, rename/trim to 4, assign `customRenderPipeline`); URP assets via `UniversalRenderPipelineAsset.Create(rendererData)` saved under `Assets/_Project/Settings/`. Preset values within the scaling rules tested above (exact numbers are the builder's call). Restart-required settings (active input handler) are written to the asset; the next batch run picks them up.

- [ ] **Step 7: Apply and verify** — `./tools/unity.ps1 -Execute Requiem.EditorTools.ProjectSetup.Run; ./tools/unity.ps1 -Tests EditMode` → all PASS. Run `ProjectSetup.Run` a second time and `git status` → no changes (idempotent).

- [ ] **Step 8: Commit**

```bash
git add Packages ProjectSettings Assets tools/unity.ps1
git commit -m "feat(unity): bootstrap Unity 6 URP project with scripted settings and quality presets"
```

---

### Task 5: Input layer

**Files:**
- Create: `Assets/_Project/Settings/Input/Requiem.inputactions`, `Assets/_Project/Scripts/Input/Requiem.Input.asmdef` (refs `Requiem.Core`, `Unity.InputSystem`), `GameplayActionNames.cs`, `GameActionMapper.cs`, `NavigateDirectionFilter.cs`, `UnityInputSource.cs`
- Create: `Assets/_Project/Tests/PlayMode/Requiem.Tests.PlayMode.asmdef` (refs Core, Input, `Unity.InputSystem`, `Unity.InputSystem.TestFramework`)
- Modify: `Assets/_Project/Tests/EditMode/Requiem.Tests.EditMode.asmdef` (add refs `Requiem.Input`, `Unity.InputSystem`)
- Test: `Assets/_Project/Tests/EditMode/GameActionMapperTests.cs`, `NavigateDirectionFilterTests.cs`, `Assets/_Project/Tests/PlayMode/UnityInputSourceTests.cs`

**Interfaces:**
- Consumes: `EventBus`, `GameAction`, `GameActionType` (Task 2).
- Produces (namespace `Requiem.Input`):
  - `public static class GameplayActionNames` — `Map = "Gameplay"`, `Confirm`, `Back`, `Parry`, `Dodge`, `Navigate` (values equal their names).
  - `public static class GameActionMapper` — `const float NavigateThreshold = 0.5f`; `bool TryMapButton(string actionName, double timestamp, out GameAction action)` (true for Confirm/Back/Parry/Dodge, false otherwise incl. Navigate/null); `(int x, int y) QuantizeNavigate(float x, float y)` (each axis: `>= 0.5 → 1`, `<= -0.5 → -1`, else `0`).
  - `public sealed class NavigateDirectionFilter` — `bool TryUpdate(int x, int y)`: true only when `(x,y)` differs from the last value and is not `(0,0)`; returning to `(0,0)` resets so the same direction fires again.
  - `public sealed class UnityInputSource : IDisposable` — ctor `(InputActionAsset actions, EventBus bus)`; `void Enable()`; publishes `GameAction` on button `performed` with `timestamp = context.time`; Navigate reads `Vector2` on `performed`/`canceled`, quantizes, filters, publishes with `navigateX/Y`.

- [ ] **Step 1: Write `Requiem.inputactions`** (JSON): map `Gameplay`; buttons Confirm (`<Keyboard>/enter`, `<Gamepad>/buttonSouth`, `<Mouse>/leftButton`), Back (`<Keyboard>/escape`, `<Gamepad>/buttonEast`, `<Mouse>/rightButton`), Parry (`<Keyboard>/j`, `<Keyboard>/space`, `<Gamepad>/rightShoulder`), Dodge (`<Keyboard>/k`, `<Keyboard>/leftShift`, `<Gamepad>/leftShoulder`); value Navigate (`Vector2`): 2D composites WASD and arrows, `<Gamepad>/leftStick`, `<Gamepad>/dpad`.

- [ ] **Step 2: Write the failing EditMode tests**

```csharp
[TestCase("Parry", GameActionType.Parry)] [TestCase("Dodge", GameActionType.Dodge)]
[TestCase("Confirm", GameActionType.Confirm)] [TestCase("Back", GameActionType.Back)]
public void TryMapButton_MapsKnownButtons(string name, GameActionType expected) { /* true, Type == expected, Timestamp == 42.123456789 exactly */ }
[TestCase("Navigate")] [TestCase("Unknown")] [TestCase(null)]
public void TryMapButton_RejectsOthers(string name) => Assert.That(GameActionMapper.TryMapButton(name, 1, out _), Is.False);
[TestCase(0.49f, 0f, 0, 0)] [TestCase(0.5f, 0f, 1, 0)] [TestCase(-0.7f, 0.7f, -1, 1)] [TestCase(0f, -1f, 0, -1)]
public void QuantizeNavigate(float x, float y, int ex, int ey) { /* Assert.That(result, Is.EqualTo((ex, ey))); */ }
[Test] public void Filter_JitterAroundSameDirection_FiresOnce() { /* (1,0),(1,0),(1,0) → true,false,false */ }
[Test] public void Filter_ReturnToCenter_AllowsSameDirectionAgain() { /* (1,0)=true,(0,0)=false,(1,0)=true */ }
[Test] public void Filter_DirectionChange_Fires() { /* (1,0)=true,(1,1)=true */ }
[Test] public void InputActionsAsset_HasExactDefaultKeyboardBindings() {
    // load Requiem.inputactions via AssetDatabase; Parry bindings contain "<Keyboard>/j" and "<Keyboard>/space"; Dodge "<Keyboard>/k" and "<Keyboard>/leftShift"; Confirm "<Keyboard>/enter"; Back "<Keyboard>/escape"
}
```

- [ ] **Step 3: Write the failing PlayMode test** (`UnityInputSourceTests : InputTestFixture`): add a `Keyboard` and `Gamepad`; create the source from the asset; `Press(keyboard.jKey)` → exactly one `GameAction` Parry whose `Timestamp` equals the press event time (`InputState.currentTime` at press, tolerance 1e-9); `Press(gamepad.leftShoulder)` → Dodge; `Set(gamepad.leftStick, new Vector2(0.9f, 0))` then `Set(..., new Vector2(0.95f, 0.1f))` → exactly one Navigate `(1,0)`.

- [ ] **Step 4: Run to verify failure** — `./tools/unity.ps1 -Tests EditMode; ./tools/unity.ps1 -Tests PlayMode` → FAIL (types missing).
- [ ] **Step 5: Implement** the four classes per Interfaces.
- [ ] **Step 6: Run to verify pass** — both commands → all PASS.
- [ ] **Step 7: Commit** — `git commit -m "feat(input): add gameplay input actions and timestamped input source"`

---

### Task 6: Private assets repo, submodule and placeholder art

🧑 **OWNER:** complete `docs/setup.md` §2 (create private `RedaMG1/requiem-private`, initialized with a README — an empty repo cannot be added as a submodule) before this task.

**Files:**
- Create: `.gitmodules` (submodule `Assets/Art` → `https://github.com/RedaMG1/requiem-private.git`)
- Create in `requiem-private`: `.gitattributes` (LFS: `*.png *.jpg *.tga *.psd *.exr *.fbx *.blend *.wav *.ogg *.mp3 *.flac *.ttf *.otf`), `README.md` (purpose, never make public)
- Create: `Assets/_Project/Editor/PlaceholderArtBuilder.cs`
- Create (by the builder, committed in `requiem-private`): `Placeholders/T_Placeholder_Checker.png`, `Placeholders/M_Placeholder_Enemy.mat` (+ `.meta` files)
- Test: `Assets/_Project/Tests/EditMode/PlaceholderArtTests.cs`

**Interfaces:**
- Produces: `Requiem.EditorTools.PlaceholderArtBuilder.Build()` (idempotent; writes a 256×256 two-colour checker PNG and a URP Lit material using it); constant asset path `Assets/Art/Placeholders/M_Placeholder_Enemy.mat` used by Task 7.

- [ ] **Step 1: Add the submodule** — `git submodule add https://github.com/RedaMG1/requiem-private.git Assets/Art`; inside `Assets/Art`: `git lfs install`, add `.gitattributes` and `README.md`, commit, push.
- [ ] **Step 2: Write the failing test** — `PlaceholderMaterial_ExistsAndUsesCheckerTexture`: material at the path loads, shader is `Universal Render Pipeline/Lit`, `mainTexture` is `T_Placeholder_Checker`, and `AssetDatabase.GetAssetPath(texture)` starts with `Assets/Art/`. Mark with `[Category("RequiresPrivateArt")]`.
- [ ] **Step 3: Run to verify failure** — `./tools/unity.ps1 -Tests EditMode` → FAIL (asset missing).
- [ ] **Step 4: Implement `PlaceholderArtBuilder.Build()`**, run `./tools/unity.ps1 -Execute Requiem.EditorTools.PlaceholderArtBuilder.Build`.
- [ ] **Step 5: Run to verify pass** — `./tools/unity.ps1 -Tests EditMode` → PASS. Run `git -C Assets/Art lfs ls-files` → lists `T_Placeholder_Checker.png`.
- [ ] **Step 6: Commit both repos** — in `Assets/Art`: `git add -A && git commit -m "feat: add placeholder checker art" && git push`; in the public repo: `git add .gitmodules Assets/Art Assets/_Project && git commit -m "feat(art): mount private assets submodule with placeholder art"`.

---

### Task 7: Battle test scene, presentation and composition root

**Files:**
- Create: `Assets/_Project/Scripts/Presentation/Requiem.Presentation.asmdef` (refs Core, `Unity.RenderPipelines.Universal.Runtime`), `BattleLayoutView.cs`, `QualityPresetApplier.cs`, `FrameStatsSampler.cs`
- Create: `Assets/_Project/Scripts/App/Requiem.App.asmdef` (refs Core, Input, Presentation, `Unity.InputSystem`), `AppBootstrap.cs`
- Create: `Assets/_Project/Editor/SceneBuilder.cs`, `Assets/_Project/Editor/BuildScript.cs`
- Create (by `SceneBuilder`): `Assets/_Project/Scenes/BattleTest.unity`
- Modify: `Requiem.Tests.EditMode.asmdef` (add refs `Requiem.Presentation`, `Requiem.App`), `Requiem.Tests.PlayMode.asmdef` (add refs `Requiem.Presentation`, `Requiem.App`, `Unity.RenderPipelines.Universal.Runtime`)
- Test: `Assets/_Project/Tests/EditMode/BattleTestSceneTests.cs`, `Assets/_Project/Tests/PlayMode/BattleTestSceneTests.cs`, `Assets/_Project/Tests/EditMode/NoCodeDependsOnPrivateArtTests.cs`

**Interfaces:**
- Consumes: `EventBus`, messages, `HeroHighlightService` (Tasks 2–3); `UnityInputSource` (Task 5); placeholder material path (Task 6).
- Produces:
  - `Requiem.Presentation.BattleLayoutView : MonoBehaviour` — serialized `Transform[] heroes` (3), `Transform enemy`; `void Bind(EventBus bus)`; `int HighlightedIndex { get; }`; highlighted hero target scale `1.25`, others `1.0`; idle bob amplitude `0.05` m, period `2` s, phase offset per unit.
  - `Requiem.Presentation.QualityPresetApplier : IDisposable` — ctor `(EventBus bus)`; on `QualityPresetRequested` → `QualitySettings.SetQualityLevel((int)preset, true)` then publish `QualityPresetChanged`; on `BenchmarkModeRequested(true)` → `vSyncCount = 0`, `Application.targetFrameRate = -1`; `(false)` → `vSyncCount = 1`.
  - `Requiem.Presentation.FrameStatsSampler : MonoBehaviour` — `const float SampleIntervalSeconds = 0.5f`; `void Bind(EventBus bus)`; publishes `FrameStats` every interval from `FrameTimingManager` (GPU = `NaN` when 0/unavailable) and averaged `unscaledDeltaTime`.
  - `Requiem.App.AppBootstrap : MonoBehaviour` — serialized `InputActionAsset inputActions`, `BattleLayoutView battleLayout`, `FrameStatsSampler frameStats`; `EventBus Bus { get; }`; `Awake` builds the graph (bus, `HeroHighlightService(bus, 3)`, `QualityPresetApplier`, `UnityInputSource`, binds views, sets `Application.runInBackground = false`), then publishes `HeroHighlightChanged` (via `PublishCurrent`) and `QualityPresetChanged(current level)`; `OnDestroy` disposes everything.
  - `Requiem.EditorTools.SceneBuilder.BuildBattleTestScene()` — idempotent; GameObjects named `Ground`, `Sun`, `Main Camera`, `Hero_0`, `Hero_1`, `Hero_2`, `Enemy_0` (enemy uses the placeholder material), `App` (with `AppBootstrap`, `FrameStatsSampler`); adds the scene as build index 0.
  - `Requiem.EditorTools.BuildScript.BuildWindowsDev()` — `StandaloneWindows64`, `BuildOptions.Development`, scenes from build settings, output from `-customBuildPath` argument or `Builds/Windows/Requiem.exe`; exits with code 1 on failure in batch mode.

- [ ] **Step 1: Write the failing EditMode tests** — `Scene_ContainsBattleLayout` (open scene; all named objects exist; `Hero_*` and `Enemy_0` are capsules/primitives with renderers); `Scene_IsBuildIndexZero`; `Enemy_UsesPlaceholderMaterial` (`[Category("RequiresPrivateArt")]`); `NoCodeDependsOnPrivateArt`: no `.cs` file under `Assets/_Project` contains the string `"Assets/Art/"` except `PlaceholderArtBuilder.cs` and test files.

- [ ] **Step 2: Write the failing PlayMode tests** (load `BattleTest` via `SceneManager.LoadScene`, yield a frame, get `AppBootstrap.Bus`):

```csharp
[UnityTest] public IEnumerator HighlightRequest_HighlightsNextHero() { /* publish HighlightNextHeroRequested; yield; Assert.That(view.HighlightedIndex, Is.EqualTo(1)); */ }
[UnityTest] public IEnumerator QualityRequest_ChangesQualityLevelAndConfirms() { /* request Ultra → QualitySettings.GetQualityLevel()==3 and QualityPresetChanged(Ultra) received */ }
[UnityTest] public IEnumerator BenchmarkMode_TurnsVSyncOffAndBackOn() { /* vSyncCount 0 then 1 */ }
[UnityTest] public IEnumerator FrameStats_ArePublishedWithinOneSecond() { /* wait up to 1 s realtime; received at least one FrameStats with Fps > 0 */ }
```

- [ ] **Step 3: Run to verify failure** — EditMode and PlayMode runs FAIL.
- [ ] **Step 4: Implement** the Presentation classes, `AppBootstrap`, `SceneBuilder`, `BuildScript`; run `./tools/unity.ps1 -Execute Requiem.EditorTools.SceneBuilder.BuildBattleTestScene`.
- [ ] **Step 5: Run to verify pass** — `./tools/unity.ps1 -Tests EditMode; ./tools/unity.ps1 -Tests PlayMode` → all PASS.
- [ ] **Step 6: Verify a local Windows development build** — `./tools/unity.ps1 -Execute Requiem.EditorTools.BuildScript.BuildWindowsDev` → exit 0 and `Builds/Windows/Requiem.exe` exists.
- [ ] **Step 7: Verify the project compiles without the submodule** (Review Focus 2) — `git clone https://github.com/RedaMG1/requiem.git "$env:TEMP\requiem-nosub"` (branch of this task, no `--recurse-submodules`); run Unity `-batchmode -quit -projectPath <that folder>` → exit 0, log has no `error CS`. Delete the temp folder.
- [ ] **Step 8: 🧑 OWNER check** — open `Builds/Windows/Requiem.exe`: three heroes and an enemy with a checker texture bob gently; window loses focus → scene pauses.
- [ ] **Step 9: Commit** — `git commit -m "feat(scene): add battle test scene, presentation views and app composition root"`

---

### Task 8: Debug HUD and architecture tests

**Files:**
- Create: `Assets/_Project/Scripts/UI/DebugHud/Requiem.UI.DebugHud.asmdef` (refs Core; `defineConstraints: ["UNITY_EDITOR || DEVELOPMENT_BUILD"]`), `DebugHudPresenter.cs`, `DebugHudController.cs`
- Create: `Assets/_Project/UI/DebugHud.uxml`, `DebugHud.uss`, `DebugHudPanelSettings.asset` (created by `SceneBuilder`)
- Modify: `AppBootstrap.cs` (create HUD inside `#if UNITY_EDITOR || DEVELOPMENT_BUILD`; App asmdef references `Requiem.UI.DebugHud`), `SceneBuilder.cs` (add `UIDocument` on `App` with panel settings and UXML)
- Modify: `Requiem.Tests.EditMode.asmdef` and `Requiem.Tests.PlayMode.asmdef` (add ref `Requiem.UI.DebugHud`)
- Test: `Assets/_Project/Tests/EditMode/DebugHudPresenterTests.cs`, `Assets/_Project/Tests/EditMode/ArchitectureTests.cs`, `Assets/_Project/Tests/PlayMode/DebugHudTests.cs`

**Interfaces:**
- Consumes: `EventBus`, messages, `GameAction` (Task 2); `AppBootstrap` (Task 7).
- Produces (namespace `Requiem.UI.DebugHud`):
  - `public sealed class DebugHudPresenter : IDisposable` — ctor `(EventBus bus)`; `event Action Changed`; `string FrameStatsText`, `string LastActionText`, `QualityPreset CurrentPreset`, `bool BenchmarkEnabled`; `void SelectPreset(QualityPreset)` publishes `QualityPresetRequested`; `void SetBenchmark(bool)` sets `BenchmarkEnabled` and publishes `BenchmarkModeRequested`; `void HighlightNextHero()` publishes `HighlightNextHeroRequested`. Formats (invariant culture): FrameStats → `"{Fps:0} fps · CPU {Cpu:0.00} ms · GPU {Gpu:0.00} ms"`, GPU part `"GPU n/a"` when `NaN`, suffix `" · VSync off"` when `!VSyncEnabled`; GameAction → `"{Type} @ {Timestamp:0.000000} s"`, Navigate adds `" ({x},{y})"`; initial `LastActionText = "No input yet"`, initial `FrameStatsText = "—"`. `CurrentPreset` follows `QualityPresetChanged`.
  - `public sealed class DebugHudController : IDisposable` — ctor `(VisualElement root, DebugHudPresenter presenter)`; binds elements by name: Label `frame-stats`, Label `last-action`, DropdownField `quality-preset` (choices `Low`, `Medium`, `High`, `Ultra`), Toggle `benchmark`, Button `highlight-next`.

- [ ] **Step 1: Write the failing presenter tests**

```csharp
[Test] public void FrameStats_FormatsWithInvariantCulture_UnderFrenchLocale() {
    // set CultureInfo.CurrentCulture = new CultureInfo("fr-FR") (restore in finally)
    // publish FrameStats(59.6, 4.256, 3.1, true) → Assert.That(p.FrameStatsText, Is.EqualTo("60 fps · CPU 4.26 ms · GPU 3.10 ms"));
}
[Test] public void FrameStats_ShowsGpuNotAvailable_AndVSyncOff() { /* (120,8.333,NaN,false) → "120 fps · CPU 8.33 ms · GPU n/a · VSync off" */ }
[Test] public void LastAction_ShowsTypeAndTimestamp() { /* Parry @ 12.3456789 → "Parry @ 12.345679 s" */ }
[Test] public void LastAction_NavigateIncludesDirection() { /* Navigate (-1,1) @ 1 → "Navigate @ 1.000000 s (-1,1)" */ }
[Test] public void InitialTexts() { /* "No input yet", "—" */ }
[Test] public void SelectPreset_PublishesRequest() { /* Ultra → QualityPresetRequested(Ultra) */ }
[Test] public void CurrentPreset_FollowsChangedMessage() { /* publish QualityPresetChanged(Low) → CurrentPreset == Low and Changed raised */ }
[Test] public void SetBenchmark_PublishesAndStores() { /* true → BenchmarkModeRequested(true), BenchmarkEnabled true */ }
[Test] public void HighlightNextHero_PublishesRequest() { }
[Test] public void Dispose_StopsUpdates() { }
```

- [ ] **Step 2: Write the failing architecture tests** (`ArchitectureTests`, using `Assembly.GetReferencedAssemblies()` on the loaded assemblies):

```csharp
[Test] public void Core_ReferencesNoUnityAssembly() { /* no reference name starts with "UnityEngine" or "UnityEditor" */ }
[Test] public void UI_DoesNotReferencePresentation() { }
[Test] public void Presentation_DoesNotReferenceUI() { }
[Test] public void Input_DoesNotReferencePresentationOrUI() { }
```

- [ ] **Step 3: Write the failing PlayMode HUD test** — load `BattleTest`; root of the `UIDocument` has the five named elements; publish `FrameStats(60, 5, 4, true)` and a Parry `GameAction`; after one frame `frame-stats` and `last-action` label texts equal the presenter's formatted strings.
- [ ] **Step 4: Run to verify failure** — EditMode and PlayMode FAIL.
- [ ] **Step 5: Implement** presenter, controller, UXML/USS (top-left panel, readable at 1080p), update `SceneBuilder` and `AppBootstrap`; rebuild the scene.
- [ ] **Step 6: Run to verify pass** — EditMode and PlayMode → all PASS; `dotnet test dotnet/Requiem.sln` → PASS.
- [ ] **Step 7: Verify the HUD is excluded from release builds** — temporarily build with `BuildOptions.None` via a one-off `-executeMethod` call to a `BuildScript.BuildWindowsReleaseCheck()` (output `Builds/WindowsRelease/`); `Builds/WindowsRelease/Requiem_Data/Managed/` contains no `Requiem.UI.DebugHud.dll`. Keep the method (it documents the check).
- [ ] **Step 8: 🧑 OWNER check** (acceptance 3) — in a dev build: keyboard J/Space/K/Shift/Enter/Esc, mouse buttons, and a gamepad all update "last action" with timestamps; "Highlight next hero" enlarges the next hero; the preset dropdown changes the look.
- [ ] **Step 9: Commit** — `git commit -m "feat(ui): add development debug HUD and assembly architecture tests"`

---

### Task 9: Public CI — code checks, CodeQL, dispatch

🧑 **OWNER:** complete `docs/setup.md` §3 (token `PRIVATE_DISPATCH_TOKEN`, security features, Actions approval setting). Branch protection is enabled at the end of Task 10.

**Files:**
- Create: `.github/workflows/ci.yml`, `.github/workflows/codeql.yml`, `tools/ci/verify-public-workflows.sh`

**Interfaces:**
- Produces: `repository_dispatch` to `RedaMG1/requiem-private` with `event_type: "unity-ci"` and `client_payload: { "sha": "<head sha>", "ref": "<git ref>", "pr": <number or null> }`; public job names `core`, `dispatch`; CodeQL workflow name `CodeQL`.

- [ ] **Step 1: Write the failing guard script** `tools/ci/verify-public-workflows.sh` — fails (exit 1, message naming file and line) if any file in `.github/workflows/` uses `actions/upload-artifact`, `actions/cache`, references `Assets/Art` or `Library`, uses a `uses:` ref that is not a 40-hex SHA (local `./` actions excepted), lacks a top-level `permissions:` block, or if `ci.yml`'s `dispatch` job lacks the fork guard `github.event.pull_request.head.repo.full_name == github.repository`. Prove it works: create a temp workflow with `uses: actions/cache@v4` and run the script → exit 1; delete it.

- [ ] **Step 2: Write `ci.yml`** — triggers `pull_request` and `push` to `main`; `permissions: contents: read`; job `core` (ubuntu-latest): checkout (no submodules), `actions/setup-dotnet` from `global.json`, `bash tools/ci/verify-public-workflows.sh`, `dotnet format dotnet/Requiem.sln --verify-no-changes`, `dotnet build dotnet/Requiem.sln -c Release`, `dotnet test dotnet/Requiem.sln -c Release --no-build`. Job `dispatch` (needs `core`; runs on push to `main` or same-repo PRs only; `permissions: statuses: write`): computes changed files (PR: `base.sha...head.sha`; push: `before..after`, with full history fetch limited to those commits); if none match `^(Assets|Packages|ProjectSettings)/` → posts `success` statuses `unity / tests` and `unity / build-windows` with description `skipped: no Unity changes` (GITHUB_TOKEN); else sends the dispatch with `PRIVATE_DISPATCH_TOKEN` via `gh api repos/RedaMG1/requiem-private/dispatches`.

- [ ] **Step 3: Write `codeql.yml`** — CodeQL for `csharp`, `build-mode: none`, on PR/push to `main` and weekly; `permissions: security-events: write, contents: read`.

- [ ] **Step 4: Lint and guard locally**

Run: `actionlint; bash tools/ci/verify-public-workflows.sh`
Expected: no findings; exit 0.

- [ ] **Step 5: Verify on GitHub** — push the branch, open a PR containing only this task's files (no Unity paths): `core` and `CodeQL` pass; `unity / tests` and `unity / build-windows` show success "skipped: no Unity changes" (Review Focus 1).
- [ ] **Step 6: Commit** — `git commit -m "ci: add public code checks, CodeQL and private pipeline dispatch"` (before Step 5's push).

---

### Task 10: Private CI — Unity tests, Windows build, itch.io

🧑 **OWNER:** complete `docs/setup.md` §4 (Unity licence secrets, `PUBLIC_STATUS_TOKEN`, itch.io restricted page, `BUTLER_API_KEY`, variable `ITCH_TARGET`).

**Files (in `Assets/Art`, i.e. the `requiem-private` repo):**
- Create: `.github/workflows/unity-ci.yml`

**Interfaces:**
- Consumes: dispatch payload `{sha, ref, pr}` (Task 9); `BuildScript.BuildWindowsDev` (Task 7); test categories incl. `RequiresPrivateArt`.
- Produces: commit statuses `unity / tests` and `unity / build-windows` on `RedaMG1/requiem@<sha>` with `target_url` = this run.

- [ ] **Step 1: Write `unity-ci.yml`** — triggers `repository_dispatch: types: [unity-ci]` and `workflow_dispatch` (inputs `sha`, `ref`, `pr`); `concurrency` group per `sha`; `permissions: contents: read`; steps:
  1. Post `pending` for both contexts (`gh api repos/RedaMG1/requiem/statuses/$SHA` with `PUBLIC_STATUS_TOKEN`).
  2. Checkout `RedaMG1/requiem` at `$SHA` into `requiem/` (no submodules).
  3. Read the submodule commit: `git -C requiem ls-tree HEAD Assets/Art` → `ART_SHA`.
  4. Checkout this repo at `ART_SHA` into `requiem/Assets/Art` with `lfs: false`; build `.lfs-assets-id` from `git lfs ls-files -l`; `actions/cache` `.git/lfs` (inside the art checkout) keyed on its hash; `git lfs pull`.
  5. `actions/cache` `requiem/Library` keyed on hashes of `Assets/**`, `Packages/**`, `ProjectSettings/**`.
  6. `game-ci/unity-test-runner` (pinned SHA): `projectPath: requiem`, `testMode: all`, Unity version from `ProjectVersion.txt`; licence secrets.
  7. `game-ci/unity-builder` (pinned SHA): `targetPlatform: StandaloneWindows64`, `buildMethod: Requiem.EditorTools.BuildScript.BuildWindowsDev`, `projectPath: requiem`.
  8. `actions/upload-artifact` (pinned SHA) of the build, `retention-days: 7`.
  9. If `ref == refs/heads/main`: download butler from `https://broth.itch.zone/butler/linux-amd64/LATEST/archive/default`, `butler push <build dir> "$ITCH_TARGET:windows-dev" --userversion "${SHA:0:7}"` with `BUTLER_API_KEY`.
  10. Post final `success`/`failure` per context (`if: always()`), each tied to its step's outcome.

- [ ] **Step 2: Lint** — `actionlint Assets/Art/.github/workflows/unity-ci.yml` → no findings.
- [ ] **Step 3: Commit and push in the private repo to its default branch (`main`)** — `repository_dispatch` only runs workflows that exist on the default branch — then update the submodule pointer in the public repo branch and push.
- [ ] **Step 4: Verify end to end** — the public PR (it now changes `Assets/Art` pointer, a Unity path) shows `unity / tests` and `unity / build-windows` going pending → success; the private run's artifact downloads and `Requiem.exe` runs on Windows (🧑 OWNER, acceptance 5); the public repo's Actions run page has **no** artifacts.
- [ ] **Step 5: 🧑 OWNER** — enable branch protection on `main` (setup.md §3) requiring `core`, `CodeQL`, `unity / tests`, `unity / build-windows`; merge the PR; confirm the itch.io restricted page shows a new `windows-dev` build (acceptance 6).

---

### Task 11: Acceptance run and baselines

**Files:**
- Modify: `README.md` (performance baseline table, build size)

**Interfaces:**
- Consumes: everything above.

- [ ] **Step 1: Fresh clone check** (acceptance 1) — `git clone --recurse-submodules https://github.com/RedaMG1/requiem.git "$env:TEMP\requiem-fresh"`; there: `dotnet test dotnet/Requiem.sln`, `./tools/unity.ps1 -Tests EditMode`, `./tools/unity.ps1 -Tests PlayMode` → all PASS; `ProjectVersion.txt` unchanged afterwards.
- [ ] **Step 2: Architecture violation check** (acceptance 4) — on a throwaway branch add `using UnityEngine;` to a Core file → `dotnet build dotnet/Requiem.sln` and `./tools/unity.ps1 -Compile` both FAIL; add a `Requiem.Presentation` reference to the DebugHud asmdef → `ArchitectureTests` FAIL. Delete the branch.
- [ ] **Step 3: 🧑 OWNER baselines** (acceptance 2) — in a dev build, enable Benchmark (VSync off), switch Low → Medium → High → Ultra, wait 10 s each, note CPU/GPU ms and FPS; then Benchmark off and confirm a steady 60 fps.
- [ ] **Step 4: Record** the numbers, the owner's CPU/GPU/RAM, and the zipped Windows build size in `README.md`.
- [ ] **Step 5: Leak check** (acceptance 7) — in the public repo, `git ls-files | Select-String -Pattern '\.(png|jpg|tga|psd|exr|fbx|blend|wav|ogg|mp3|flac)$'` → no results, and `git ls-files Assets/Art` → only the submodule entry `Assets/Art`; the public repo's Actions runs list no artifacts or caches.
- [ ] **Step 6: Docs check** (acceptance 8) — README, LICENSE, CLAUDE.md, `docs/setup.md`, six ADRs present and match what was built (update any drift).
- [ ] **Step 7: Commit** — `git commit -m "docs: record M0 baselines and acceptance results"`; PR and merge.
