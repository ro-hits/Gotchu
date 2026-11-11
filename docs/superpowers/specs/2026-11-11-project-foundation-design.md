# Gotchu — Project Foundation Design

**Date:** 2026-11-11
**Status:** Draft, awaiting project lead approval
**Scope:** The repo setup, architecture skeleton, and written ways of working that let the team start building Gotchu in parallel. This is *not* the product build.

---

## 1. Intent

Gotchu is an AI shopping concierge that a retailer embeds on their site with one script tag. It helps shoppers find products, stay within a budget, get curated picks, add items to the cart, and reach checkout. JTV (jewelry) is the first retailer; the product must stay retailer-agnostic so it can plug into other sites.

The project lead wants the team to be able to clone the repo and start working on day one, with:

- a codebase whose **architecture keeps business logic separate from frameworks and integrations**, so it scales and stays easy to maintain;
- **written rules for how the team works**: branching, commits, PRs, reviews, how tasks are picked up;
- **setup instructions that work the first time**, with no tribal knowledge.

## 2. Goals, non-goals, anti-goals

**Goals**
- A monorepo with a Python server and a TypeScript widget, both runnable locally with a handful of documented commands.
- A *walking skeleton*: one real request travels widget → server API → core → adapters and back, proving every layer and the setup instructions.
- The architecture's dependency rule is **enforced by tooling in CI**, not just described in prose.
- Complete written guidance: README, setup guide, contributing guide, architecture guide, decision records, AI-assistant guide, GitHub setup checklist for the lead.
- CI on every PR: lint, format check, type check, import-boundary check, tests, widget build.

**Non-goals (for this foundation)**
- Real Claude integration (the first feature issue, not part of the foundation).
- Real JTV integration (needs research into JTV's catalog and cart; a stub adapter marks the spot).
- Deployment, hosting, production infrastructure, auth, analytics.
- Product features beyond what the walking skeleton needs (curation, full budget logic, checkout flow).

**Anti-goals (would be a failure even if it "works")**
- The core importing FastAPI, Anthropic SDK, HTTP clients, or retailer-specific code.
- Setup steps that only work on the lead's machine, or instructions that don't run verbatim.
- The widget's styles leaking into the retailer page, or the retailer page's styles breaking the widget.
- Any API key or secret reaching the browser or the git history.
- Long-lived branches or giant PRs becoming the norm.

## 3. Constraints

- **Team OS is unknown.** Every documented command must work in both PowerShell (Windows) and bash/zsh (macOS/Linux). No Makefiles, no shell-only scripts as required entry points. Entry points are `uv run …` (server) and `npm run …` (widget). Add `.gitattributes` to normalize line endings (LF in repo).
- **No API key needed** for first run, tests, or CI. The AI adapter defaults to a deterministic fake.
- **Toolchain versions:** Python 3.13 (pinned via `.python-version`), uv ≥ 0.10, Node 24 LTS (pinned via `.nvmrc` and `engines`), npm (ships with Node; pnpm not assumed).
- **The lead's working copy lives in OneDrive.** The setup guide must recommend cloning outside synced folders and list OneDrive lock / slow-IO errors in troubleshooting.
- **The git remote is still `github.com/rohitkp-mcgi/gotcha.git`** while the project is named "gotchu". Docs use the real URL; renaming the GitHub repo is the lead's decision (listed in the GitHub checklist).
- **`gh` CLI is not installed** on the lead's machine. GitHub-side settings are delivered as a manual checklist, not automated.

## 4. Decisions

### 4.1 Decided by the project lead
- Python side uses **uv**.
- Tasks are **divided equally** across the team.
- The **architecture separates business logic** from the rest of the code; it must be **scalable and easy to maintain**.
- The foundation lands as **one bootstrap PR** from branch `chore/1-project-foundation`. Claude commits as it goes and pushes the branch; the lead opens and merges the PR. This is a one-time exception to the "small PRs" rule, and CI running on that PR is part of proving the setup works.
- The GitHub repo is **private on a free personal account**, so GitHub will not enforce branch protection (see §7.2 and §8).

### 4.2 Proposed by Claude and approved by the lead in conversation ("this sounds good")
- One repo, two parts: `server/` (Python, FastAPI, uv) and `widget/` (TypeScript, built to a single JS file, rendered inside a Shadow DOM).
- Claude as the AI behind the concierge.
- GitHub Issues + a GitHub Projects board for tasks.
- Branch naming `type/<issue#>-<short-description>` with types `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `spike`.
- Short-lived branches off `main`; PR with ≥ 1 review; squash-merge.
- Commit messages in the same `type: description` style.
- A fake "demo retailer" page in the repo for local development.
- CI on every PR covering lint, type checks, and tests for both parts.

### 4.3 Claude's calls (not yet reviewed by the lead — check these)
- **CI runs on pushes to any branch and on PRs**, so the bootstrap branch gets a CI run before a PR exists.
- **Per-retailer config files and prompts as files** (not hardcoded): branding, concierge name/tone, budget defaults, feature toggles, allowed origins.
- **Widget stack:** Preact (small, React-like, familiar to most devs), Vite in library mode (single IIFE bundle), Vitest, ESLint, Prettier.
- **Server stack:** FastAPI, Pydantic + pydantic-settings, PyYAML, pytest, ruff (lint + format), mypy, import-linter.
- **Fake AI adapter by default**; the real Claude adapter is the first feature issue.
- **Browser-side retailer adapter**: cart actions run in the shopper's browser because the retailer's cart is tied to their session/cookies. The server never changes the cart; it *proposes* actions (e.g. `add_to_cart`) that the widget carries out.
- **Cart snapshot per message**: the widget reads the cart through its host adapter and sends a snapshot with every chat message, so the core can enforce a whole-cart budget without owning the cart.
- **Core-run tool loop for the AI**: the AI adapter never calls the catalog itself. The core runs the loop: ask the AI → if it requests a tool (e.g. `search_products`), the core executes it through the ports and business rules → feed results back → repeat until a final reply (capped step count). This keeps business rules in the core and lets the AI see real products before it writes the reply — the shape the real Claude integration will need.
- **Stateless server** with sessions behind a storage port (in-memory now, Redis later).
- **pre-commit** hooks (installed via `uvx pre-commit install`) running ruff and the widget linters.
- **Ownership by area, not by person**; the guidelines don't assume a headcount.
- **Local dev uses two terminals** (server, widget) with Vite proxying `/api` to the server.

## 5. Architecture

### 5.1 The core-and-adapters rule (server)

```
            ┌──────────────────────────────────────────┐
  HTTP ───► │ api/          FastAPI routes, schemas    │
            └───────────────┬──────────────────────────┘
                            │ calls use cases
            ┌───────────────▼──────────────────────────┐
            │ core/         business logic             │
            │   models      Product, Cart, Budget, …   │
            │   rules       pure functions             │
            │   use_cases   HandleMessage, …           │
            │   ports       interfaces (Protocols)     │
            └───────────────▲──────────────────────────┘
                            │ implement ports
            ┌───────────────┴──────────────────────────┐
            │ adapters/     retailers/demo, retailers/ │
            │               jtv (stub), ai/fake,       │
            │               ai/claude (later),         │
            │               storage/memory             │
            └──────────────────────────────────────────┘
   bootstrap.py — the only place that picks adapters and wires them in
```

**Dependency rule:** `core` imports nothing from `api`, `adapters`, `bootstrap`, or any framework/SDK (FastAPI, Starlette, Anthropic, httpx, etc.). `adapters` may import `core`. `api` may import `core` and `bootstrap`. Enforced by **import-linter** contracts in `pyproject.toml`, run in CI and pre-commit. The done criteria include proving the check fails on a deliberate violation.

**Ports** are Python `Protocol`s in `core/ports/`:
- `RetailerCatalog` — search products, get product by id.
- `AIAssistant` — given the conversation, context, and the list of tools the core offers, returns *either* one or more tool requests *or* a final reply (text + product ids to show + proposed actions). It never executes tools itself.
- `SessionStore` — load/save conversation state by session id.

**The tool loop lives in the core** (`HandleMessage`): call `AIAssistant` → execute any requested tool through ports + rules (e.g. `search_products` = `RetailerCatalog.search` then budget filtering) → append results → call again, up to a fixed step cap → return the final reply. Tools are defined in the core, so business rules (budget, curation) always apply no matter which AI is plugged in.

**Cart and budget:** the server does not own or change the cart. Each chat request carries a cart snapshot (`items[{product_id, qty, unit_price}]`, `currency`) read by the widget's host adapter. The core's `Cart` model is a read-only view of that snapshot. Budget rules support both a per-item price cap ("under 200") and an optional whole-cart budget for the session; the walking skeleton only exercises the per-item cap.

### 5.2 Retailer config (server)

`server/retailers/<retailer_id>.yaml` holds per-retailer settings: display name, concierge name and tone, brand colors, budget defaults, feature toggles, allowed CORS origins, which catalog adapter to use. Prompt templates live as text files under `server/prompts/`. A typed settings model validates config at startup — bad config fails fast. Secrets come from environment variables (`.env`, gitignored; `.env.example` committed), never from retailer YAML.

### 5.3 Widget

- **Embed contract:** `<script src="…/gotchu.js" data-retailer="demo" async></script>`. The loader reads `data-*` attributes, fetches the retailer's public config (`GET /api/v1/retailers/{id}/config`), and mounts the UI in a Shadow DOM.
- **Layers:** `src/core/` (state, message protocol types — no DOM or Preact imports), `src/ui/` (Preact components), `src/api/` (server client), `src/host/` (browser-side retailer adapters). Same dependency idea as the server, enforced with ESLint `no-restricted-imports`.
- **Host adapter interface:** `addToCart(productId, qty)`, `getCart()`, `goToCheckout()`. `demo` implementation drives the demo page's fake cart; `jtv` is a stub with a TODO pointing at the research issue.
- **Session id** generated by the widget, kept in `sessionStorage` (wrapped in try/catch; works without it).

### 5.4 Walking skeleton (the one slice that must work end to end)

1. Developer opens the demo retailer page (`widget/demo/index.html` served by Vite).
2. Types "ring under 200" in the widget.
3. Widget → `POST /api/v1/chat` `{retailer_id, session_id, message, cart}` (cart snapshot from the demo host adapter).
4. `api` → `core.use_cases.HandleMessage` runs the tool loop:
   - `AIAssistant` (fake, deterministic keyword parser) requests `search_products(query="ring", max_price=200)`;
   - core executes it: `RetailerCatalog` (demo: searches a JSON fixture of ~20 jewelry items) → budget rule filters by price;
   - results go back to the fake AI, which returns a final reply naming the matches;
   - `SessionStore` saves the turn.
5. Response: reply text + product cards + an `add_to_cart` action per product.
6. Widget renders cards; clicking "Add to cart" calls the demo host adapter, and the demo page's cart counter increments.

Also `GET /health`. API is versioned under `/api/v1`.

## 6. Repo layout

```
gotchu/
├── server/
│   ├── pyproject.toml  uv.lock  .python-version
│   ├── src/gotchu/
│   │   ├── core/        models/ rules/ use_cases/ ports/
│   │   ├── adapters/    retailers/{demo,jtv}/ ai/fake.py storage/memory.py
│   │   ├── api/         app.py routes/ schemas/
│   │   ├── settings.py
│   │   └── bootstrap.py
│   ├── retailers/       demo.yaml  jtv.yaml
│   ├── prompts/
│   └── tests/           unit/ (core, no I/O)  integration/ (API via TestClient)
├── widget/
│   ├── package.json  package-lock.json  vite.config.ts  tsconfig.json  .nvmrc
│   ├── src/             loader.ts  core/ ui/ api/ host/{demo,jtv}.ts
│   ├── demo/            index.html  (fake retailer storefront)
│   └── tests/
├── docs/
│   ├── setup.md  architecture.md  github-setup.md
│   ├── adr/             0000-template.md  0001-core-and-adapters.md  …
│   └── superpowers/specs/
├── .github/
│   ├── workflows/ci.yml
│   ├── pull_request_template.md
│   └── ISSUE_TEMPLATE/  feature.yml  bug.yml  spike.yml
├── README.md  CONTRIBUTING.md  CLAUDE.md
├── .editorconfig  .gitattributes  .gitignore  .pre-commit-config.yaml
└── .env.example
```

## 7. Ways of working (content of CONTRIBUTING.md)

### 7.1 Tasks
- Every piece of work starts as a **GitHub Issue** using a template (feature / bug / spike).
- **Board columns:** Backlog → Ready → In progress → In review → Done.
- **Labels:** `area:server`, `area:widget`, `area:ai`, `area:retailer`, `area:infra`, `area:docs`; `type:feat|fix|chore|spike`; `priority:high|medium|low`.
- **Even split:** at planning, the lead balances issues across people. People **assign themselves** before starting and pull from *Ready*. Rotate across areas so no one becomes the only person who knows a part. An owners table in CONTRIBUTING.md has area rows and blank name columns for the lead to fill in.

### 7.2 Branches
- `main` is always green. No direct pushes. **While the repo is private on a free plan, GitHub can't enforce this, so it's a team rule.** CONTRIBUTING.md says so plainly, and CI still runs on every PR so a red build is visible before merge. Once the repo is on a plan that supports it, the lead turns on enforcement using `docs/github-setup.md`.
- Format: `type/<issue#>-<short-kebab-description>`, lowercase, ≤ 50 characters.
  Examples: `feat/12-add-to-cart`, `fix/31-budget-rounding`, `docs/4-setup-guide`, `spike/18-jtv-cart-api`.
- Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `spike`.
- Every branch has an issue. Branch from latest `main`; keep it short-lived (aim: merged within ~3 days); rebase or merge `main` in if it drifts.
- `spike/*` branches are throwaway and never merged; findings go into the issue or an ADR.

### 7.3 Commits
- Conventional Commits: `type(scope): imperative summary` (≤ 72 chars). Scopes: `server`, `widget`, `docs`, `ci`, `repo`.
  Example: `feat(server): add budget filter to product search`.
- Small, focused commits. Never commit secrets, `.env`, or generated bundles.

### 7.4 Pull requests
- Open a **draft PR early**; mark ready when CI is green.
- **PR title** follows the commit format (it becomes the squash commit message). Body uses the template and includes `Closes #<issue>`.
- Keep PRs small (aim < 400 changed lines; split otherwise).
- **Merge requires:** ≥ 1 approving review, CI green, conversations resolved. **Squash-merge**; delete the branch after.
- The author merges after approval.

### 7.5 Reviews
- First review within **1 working day**.
- Reviewer checks: does it respect the dependency rule, are there tests for new core logic, are docs/config updated, is anything retailer-specific leaking into `core`?
- Comments prefixed `nit:` are optional; everything else must be addressed or discussed.

### 7.6 Definition of done
- Code merged to `main` via an approved PR with green CI.
- New core logic has unit tests; new endpoints have an integration test.
- Docs updated if behavior, setup, or architecture changed; ADR written if an architectural decision was made.
- Issue closed and board updated.

### 7.7 Architecture decisions
- Significant decisions get a short ADR in `docs/adr/NNNN-title.md` (context, decision, consequences). Seed ADRs record the foundational decisions in §4.

## 8. Documents to produce

| File | Audience | Must contain |
|---|---|---|
| `README.md` | Everyone | What Gotchu is, 5-minute quickstart (clone → run server → run widget → open demo), links to other docs. |
| `docs/setup.md` | New devs | Prerequisites + install for Windows (PowerShell) and macOS/Linux: Git, uv, Node 24. Step-by-step first run. Pre-commit install. Troubleshooting (incl. OneDrive, ports in use, uv/npm cache). |
| `CONTRIBUTING.md` | Everyone | §7 in full, with copy-pasteable command examples. |
| `docs/architecture.md` | Devs | §5 diagram and dependency rule, request flow, "how to add a retailer", "how to add an AI provider", where config/prompts live. |
| `docs/adr/` | Devs | Template + seed ADRs. |
| `docs/github-setup.md` | Lead | Manual checklist, in two parts. **Works on the current free private repo:** invite collaborators, squash-merge only, auto-delete head branches, labels, Projects board, optional repo rename `gotcha` → `gotchu` (and updating local remotes after). **Needs a plan that supports it:** options to get enforcement (create a GitHub organization and move the repo to it on the Team plan, or upgrade the personal account to Pro), then the exact branch protection / ruleset settings for `main` (require PR, 1 approval, required CI jobs `server` and `widget`, block force pushes, linear history). |
| `CLAUDE.md` | AI assistants | Short summary of conventions, commands, and the dependency rule, pointing to the docs above. |
| `.github/*` | Everyone | PR template with checklist, issue templates, CI workflow. |

### 8.1 Commands the docs must document (all cross-shell)

| Purpose | Server (`cd server`) | Widget (`cd widget`) |
|---|---|---|
| Install | `uv sync` | `npm install` |
| Run dev | `uv run fastapi dev src/gotchu/api/app.py` (or equivalent) | `npm run dev` (serves demo page, proxies `/api`) |
| Test | `uv run pytest` | `npm test` |
| Lint | `uv run ruff check .` | `npm run lint` |
| Format | `uv run ruff format .` | `npm run format` |
| Type check | `uv run mypy` | `npm run typecheck` |
| Import boundaries | `uv run lint-imports` | (part of `npm run lint`) |
| Everything CI runs | one documented command | `npm run check` |
| Build | — | `npm run build` |

## 9. CI

GitHub Actions, triggered on pull requests and on pushes to any branch. Two jobs:
- **server:** `astral-sh/setup-uv`, `uv sync --locked`, ruff check, ruff format --check, mypy, lint-imports, pytest.
- **widget:** `actions/setup-node` (Node 24, npm cache), `npm ci`, lint, typecheck, test, build.

Job names are stable so branch protection can require them.

## 10. Left to the builder
- Exact ruff/ESLint rule sets and mypy strictness (start strict on `core`, pragmatic elsewhere).
- Internal file structure within each layer; demo catalog contents; demo page look.
- Whether the "run everything" server command uses a small task runner (e.g. poethepoet) or a documented sequence.
- Exact wording and count of seed ADRs.
- Issue template fields.

## 11. Done criteria
1. From a **fresh clone outside OneDrive**, following `README.md` and `docs/setup.md` **verbatim in PowerShell**, a developer gets the walking skeleton working: type "ring under 200", see matching product cards, click "Add to cart", see the demo cart count increase.
2. Every server and widget check passes locally, and the CI workflow runs green on GitHub for the pushed `chore/1-project-foundation` branch.
3. A deliberate `core → adapters` import makes `lint-imports` fail (verified, then reverted).
4. No secrets in the repo; first run needs no API key.
5. All files in §8 exist and are internally consistent (commands in docs match what actually runs).
