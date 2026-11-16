# Requiem — Technical Foundation (M0) Design

Date: 2026-11-16
Status: Draft, awaiting owner approval

## 1. Intent and why

**Working title:** *Requiem* (repo `requiem`). A codename; the final title may change before release.

An original 3D turn-based RPG, inspired by the *style of combat* of Clair Obscur: Expedition 33 (turn-based actions plus real-time dodge / parry / timed attacks). Setting, characters, story, art and music are original. The story and art direction come later; this document covers only the **technical foundation** every later milestone builds on.

The owner is a solo developer (experienced programmer, aiming for a DevOps career, guitarist and composer). Budget is $0 for now, with no deadline. The project is also meant to be a showcase of clean engineering and DevOps practice.

The game must run **in the browser on desktop and on phones** (iOS Safari and Android Chrome included), in 3D.

## 2. Goals, non-goals, anti-goals

### Goals of M0
1. A repository with the full toolchain wired: dev server, type checking, lint, format, unit tests, end-to-end tests, build.
2. A Babylon.js 3D test scene that runs smoothly on desktop and on a mid-range phone.
3. A Solid UI overlay on top of the canvas, proving the UI ↔ game communication path.
4. A unified input layer (keyboard, mouse/touch, gamepad) that produces timestamped game actions — the base for parry/dodge timing later.
5. CI/CD: every pull request is checked and gets a preview deploy; `main` deploys to production on Cloudflare.
6. Architecture rules enforced by tooling, not by memory.
7. Project context documented (README, CLAUDE.md, architecture decision records) so future sessions and contributors start informed.

### Non-goals (later milestones)
- Combat rules, turn order, AP, damage, timing judgement (M1).
- Real art, models, animations, VFX; the asset pipeline beyond a placeholder (later).
- Music and the audio pipeline beyond a stub (later; owner composes).
- Story, dialogue, exploration, save system.
- Backend, accounts, leaderboards, app-store packaging (Capacitor), installable PWA / offline mode.
- Error monitoring service (Sentry) — added once there is something to monitor.

### Anti-goals (would make M0 a failure even if it "works")
- A workflow that depends on a GUI editor or manual steps to build or deploy.
- Combat/game logic coupled to Babylon (must stay testable without a GPU).
- Babylon imported from the package root (pulls megabytes into the bundle).
- Mismatched versions across `@babylonjs/*` packages.
- Secrets, API tokens or private assets committed to the public repo.
- A scene that runs on desktop but stutters or overheats a mid-range phone.

## 3. Constraints

- **Cost:** $0. Only free tiers.
- **Repo:** public on GitHub. Music, art source files and story material are **not** stored in the repo (they will live in private storage, Cloudflare R2, when they exist).
- **Platforms:** latest two versions of Chrome, Edge, Firefox, Safari (macOS and iOS), Chrome on Android. Rendering baseline is **WebGL2**; WebGPU is optional and opt-in.
- **Phone target:** the owner's reference phone is an **iPhone 15 Pro** (iOS Safari). Steady 60 fps in the M0 test scene, landscape orientation. Because this phone is high-end, also check desktop Chrome with 4× CPU throttling as a stand-in for mid-range devices; a real mid-range Android test is added later.
- **Infrastructure:** one-time account/project setup (GitHub repo settings, Cloudflare project, deploy token) is done **manually** in M0 following a written checklist (`docs/setup.md`). Every build and deploy after that is fully automatic. Managing this setup as code (Terraform) is its own milestone, **M0.5**, right after M0.
- **Security:** static site served over HTTPS with a strict Content-Security-Policy; no `unsafe-eval`.

## 4. Approaches considered

### 4.1 Engine — chosen: **Babylon.js 9.x**
| Option | Why not chosen / why chosen |
|---|---|
| **Babylon.js** ✅ | Full game engine (animation, audio, physics, GUI, glTF, inspector); written in TypeScript; strict backward-compatibility policy; WebGL2 + WebGPU; works under strict CSP. Best balance of features, stability and phone performance for a solo developer. |
| PlayCanvas | Very light and mobile-first, but fewer built-in high-end features; its editor is cloud-based (private projects paid). Strong second choice. |
| Three.js | Smallest core and biggest ecosystem, but a renderer rather than an engine (more code to write ourselves = more bugs), and releases often break APIs. |
| Godot 4 / Unity 6 web export | 8–10+ MB WebAssembly downloads, iOS memory risk, editor-first workflow, need `wasm-unsafe-eval` in CSP. Poor fit for phone browsers. |

### 4.2 Lint and format — chosen: **ESLint + typescript-eslint (strict, type-checked) + Prettier**
| Option | Trade-off |
|---|---|
| **ESLint + typescript-eslint + Prettier** ✅ | Most complete type-aware rules (e.g. `no-floating-promises` catches forgotten `await` in async game sequences); enforces import boundaries; huge plugin ecosystem (incl. `eslint-plugin-solid`). Slower than Biome, irrelevant at this size. |
| Biome | Single fast tool, simpler config, but fewer type-aware rules. Rejected because "fewest bugs" is the priority. |

### 4.3 UI — chosen: **Solid (HTML/CSS overlay) for menus and HUD + Babylon for in-world UI**
| Option | Trade-off |
|---|---|
| **Solid + HTML/CSS overlay** ✅ | Fine-grained reactivity: only the changed text/bar updates, no virtual-DOM diffing stealing frame time on phones; ~7 KB; components scale to inventory/skill trees; sharp text, responsive CSS, accessible. Owner knows React, so JSX/components are familiar. |
| Preact | Equally viable, React API; slightly more runtime work per update. |
| Vanilla TypeScript | Zero size, but unmaintainable as the RPG UI grows. |
| Babylon GUI for everything | Blurry text on phones, weaker responsive layout, no accessibility. Kept **only** for UI attached to 3D objects (enemy HP bars, damage numbers, parry rings). |

### 4.4 Hosting — chosen: **Cloudflare** (static hosting with PR previews; R2 for large assets later)
| Option | Trade-off |
|---|---|
| **Cloudflare** ✅ | Unlimited free static bandwidth, PR preview deploys, custom headers, R2 object storage (10 GB free, no egress fees), commercial use allowed, strong DevOps learning value (DNS, WAF, Workers, Terraform). 25 MB per-file limit (big files go to R2). |
| Netlify | Good DX, but limited free monthly usage. |
| Vercel | Free tier forbids commercial use. |
| GitHub Pages | No custom security headers, no PR previews. |

### 4.5 Repository — chosen: **public code, private assets**
Public gives free CodeQL, secret scanning, unlimited Actions minutes and portfolio value. Valuable creative material stays out of the repo.

**License:** all rights reserved (no open-source license). The code is visible on GitHub but may not be reused without permission. A `LICENSE` file states this; the owner may switch to an open license later. The README states contributions are not accepted for now.

## 5. Decided technical design

### 5.1 Stack
| Area | Choice |
|---|---|
| Runtime | Node.js current LTS, version pinned in `.nvmrc` and `package.json#engines` |
| Package manager | pnpm (lockfile committed; dependency install scripts blocked by default, allow-listed explicitly) |
| Language | TypeScript, `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` |
| Build / dev server | Vite |
| Engine | `@babylonjs/core` (+ `@babylonjs/loaders`, `@babylonjs/inspector` dev-only), exact pinned version, all `@babylonjs/*` on the same version |
| UI | Solid (`solid-js`, `vite-plugin-solid`) |
| Lint / format | ESLint flat config, typescript-eslint `strictTypeChecked` + `stylisticTypeChecked`, `eslint-plugin-solid`; Prettier |
| Unit tests | Vitest (Babylon `NullEngine` for headless scene tests where needed) |
| E2E tests | Playwright (desktop + phone viewport projects) |
| Git hooks | Lefthook: format + lint + typecheck staged files on commit |
| Commits | Conventional Commits |
| CI/CD | GitHub Actions |
| Dependency updates | Renovate (Babylon packages grouped into one PR) |
| Security scanning | CodeQL, GitHub secret scanning + push protection, `pnpm audit` in CI |
| Hosting | Cloudflare static hosting with PR previews; production on `main` |

### 5.2 Source layout and dependency rules
```
src/
  main.ts       bootstraps engine, game, UI
  core/         game loop, clock, event bus, state machines        (no Babylon, no Solid)
  combat/       (empty in M0) pure combat rules                     (no Babylon, no Solid)
  input/        unified input → timestamped actions                 (no Babylon, no Solid)
  render/       Babylon engine, scenes, cameras, debug tools        (may import core)
  ui/           Solid components (HUD, menus)                       (may import core; never render/)
  audio/        stub in M0
  data/         game data (characters, skills, attack timing tracks) — empty/sample in M0
tests/          unit + e2e
tools/          scripts (asset/audio pipelines later)
public/         static files (icons, self-hosted Babylon decoders later)
docs/adr/       architecture decision records
docs/superpowers/specs/  design docs
```
Rules, enforced by ESLint `no-restricted-imports` per folder:
- `core/`, `combat/`, `input/`, `data/` must not import `@babylonjs/*` or `solid-js`.
- `ui/` must not import `render/`; UI and rendering talk only through `core/` (event bus / shared state).
- Babylon is imported by deep module path only (`@babylonjs/core/...`), never from the package root.

### 5.3 M0 test scene
- Babylon `Engine` on WebGL2 by default; `?renderer=webgpu` URL flag tries WebGPU and falls back to WebGL2.
- A ground plane, one directional light + ambient/hemispheric light, a camera framing a battle layout: 3 placeholder heroes (capsules) and 1 placeholder enemy, gently idle-animated.
- Handles resize, device orientation changes, and pauses rendering when the tab is hidden.
- **Resolution scaling:** on phones, render at a fraction of the device pixel ratio (start at ~0.75) to protect frame rate and battery; adjustable from the debug overlay.
- Babylon Inspector available in dev builds only (keyboard shortcut), excluded from production bundles.

### 5.4 M0 UI overlay (Solid)
- Debug panel: FPS, frame time, draw calls, active renderer (WebGL2/WebGPU), resolution scale.
- One sample interactive control (e.g. "highlight next hero") that sends an event through `core/` and is reflected in the 3D scene — proves the UI → core → render path.
- Layout works in landscape on phone and desktop; respects phone safe areas (notches).

### 5.5 Input layer
- Normalizes keyboard, pointer/touch and gamepad into game actions: `confirm`, `back`, `parry`, `dodge`, `navigate`.
- Every action carries a high-resolution timestamp taken at the input event, so later timing judgement does not depend on frame rate.
- Default bindings: parry = J / Space / a gamepad face button, dodge = K / Shift / another gamepad face button (exact mapping is the builder's call), confirm = Enter / tap.
- *Builder's call, recorded:* touch in M0 = **temporary on-screen Parry and Dodge buttons** in the debug overlay (bottom corners, thumb-sized, respecting safe areas) plus **tap on the scene = confirm**. No gestures yet; the final touch layout is designed in M1.
- In M0, actions are shown in the debug overlay (last action + timestamp) to prove it works.

### 5.6 CI/CD pipeline
On every pull request:
`install (pnpm, cached) → typecheck → lint → unit tests → build → bundle size check → Playwright smoke (desktop + phone viewport) → preview deploy`
On push to `main`: same checks, then production deploy.
Smoke test: the page loads, a canvas renders, no console errors, the overlay is visible, a key press shows up as an action.

### 5.7 Performance budget
- M0 measures a baseline (JS size gzipped, first load time, fps on a phone) and then locks the bundle-size budget at baseline + 10%; CI fails above it.
- Rule of thumb for later: < ~150 draw calls, < ~200k triangles on screen, compressed textures (KTX2), first download < ~20 MB.

### 5.8 Security
- Security headers served by the host: strict CSP (`script-src 'self' 'wasm-unsafe-eval'`, `worker-src 'self' blob:`, never `'unsafe-eval'` for JavaScript), HSTS (no preload while on the free host subdomain), `X-Content-Type-Options: nosniff`, `Referrer-Policy`, restrictive `Permissions-Policy` (allow gamepad, fullscreen).
  - *Builder's call, recorded:* `'wasm-unsafe-eval'` and `blob:` workers are allowed because Babylon's KTX2/mesh decoders (later) run WebAssembly in workers. Arbitrary JavaScript eval stays banned. (Godot/Unity were rejected for download size and iOS memory, not for this directive.)
  - The Playwright smoke test runs against the built site with the production CSP, so a CSP violation fails CI.
- No secrets in the repo; deploy credentials live in GitHub Actions secrets.
- Source maps not served publicly.

### 5.9 Developer workflow
- Dev builds include an on-screen console (Eruda, dev-only, excluded from production) so errors can be read directly on the iPhone; Safari Web Inspector via a Mac is optional.
- `pnpm dev` serves over HTTPS on the local network so a phone on the same Wi-Fi can open it (WebGPU and some APIs require a secure origin).
- Scripts: `dev`, `build`, `preview`, `typecheck`, `lint`, `format`, `test`, `e2e`.
- Trunk-based: short-lived branches, pull requests into `main`.

### 5.10 Documentation delivered in M0
- `README.md` — what the project is, how to run it, how to test on a phone.
- `CLAUDE.md` — project context and working rules for AI sessions, including: always present options as a comparison with a recommendation for this project before asking the owner to choose.
- `docs/adr/` — ADR-001 Babylon.js, ADR-002 ESLint + typescript-eslint, ADR-003 Solid + Babylon in-world UI, ADR-004 Cloudflare, ADR-005 public code / private assets.
- `.claude/settings.json` — declares the Superpowers plugin marketplace and plugin so new sessions on this repo get it.

## 6. Done when (M0 acceptance)
1. A fresh clone runs with `pnpm install && pnpm dev`; all scripts pass.
2. The test scene runs at a steady 60 fps on desktop and on the owner's reference phone(s) in landscape — checked manually by the owner using the overlay's FPS readout; the reading is recorded in the README as the baseline.
3. Overlay shows live stats; the sample control visibly changes the scene; input actions appear with timestamps from keyboard, touch and gamepad.
4. Breaking an architecture rule (e.g. importing Babylon in `core/`) fails lint.
5. A pull request runs the full pipeline and posts a working preview link; merging to `main` updates production.
6. Security headers are present on the deployed site.
7. README, CLAUDE.md and the five ADRs exist.

## 7. Left to the builder
- Exact versions (latest stable at setup time, then pinned).
- Exact folder file names inside each module, event bus implementation details.
- Gamepad button mapping details, debug overlay styling.
- Whether to use Cloudflare Pages or Workers static assets, as long as PR previews and custom headers work.
- *Recorded:* production uses the free Cloudflare subdomain (`*.pages.dev` / `*.workers.dev`); a custom domain costs money and waits until the budget changes.
- Exact CSP directives beyond the rules above.
