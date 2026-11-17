# Gotchu: project foundation

Date: 2026-11-11 (simplified 2026-11-16)
Status: draft, waiting for the lead's approval

## What this is

Gotchu is a shopping assistant chatbot that a store adds to its website with one script tag. It helps shoppers find products within their budget, add them to the cart and get to checkout. JTV is the first store, but nothing in the code should assume JTV, so we can plug it into other sites later.

This doc only covers the setup work: the repo, a basic working skeleton of the app, and simple rules for how the team works. The goal is that anyone can clone the repo on day one and pick up a task. The real product features come after this.

## What goes into this first PR

- A repo with two parts: `server/` (Python, FastAPI, uv) and `widget/` (TypeScript, built into a single JS file).
- One flow that works end to end, so we know all the pieces connect (see "The first working flow" below).
- Setup steps that work the first time on both Windows and Mac/Linux.
- Short written rules for branches, commits, PRs and reviews.
- CI that runs lint, type checks and tests on every push and PR.

Not in this PR: the real Claude integration, the real JTV integration, deployment, logins, analytics, or any features beyond the one flow. A fake AI and a fake demo store stand in for now, so nobody needs an API key to run or test anything.

## Things to keep in mind

- We don't know what OS everyone is on, so every command has to work in PowerShell and in bash/zsh. No Makefiles or shell scripts as the main way to run things; use `uv run ...` and `npm run ...`. A `.gitattributes` file keeps line endings consistent.
- Versions: Python 3.13, uv 0.10 or newer, Node 24 LTS, npm.
- Don't clone into OneDrive or any synced folder. It causes file lock errors and slow installs. The setup guide will say this.
- The GitHub repo is still named `gotcha` (`github.com/rohitkp-mcgi/gotcha.git`). Docs will use that URL until the lead decides whether to rename it.
- The repo is private on a free account, so GitHub won't block direct pushes to `main`. "No direct pushes" is a team rule for now.
- The `gh` CLI isn't installed, so GitHub settings will be a manual checklist.

## Decisions

Already agreed:

- uv for Python.
- Tasks are split evenly across the team.
- Business logic stays separate from frameworks and outside services, so the code is easy to maintain and grow.
- All of this setup lands as one PR from `chore/1-project-foundation`. After that, PRs should be small.
- One repo with `server/` and `widget/`. The widget renders inside a Shadow DOM so its styles and the store's styles don't clash.
- Claude is the AI behind the assistant.
- GitHub Issues and a GitHub Projects board for tasks.
- Branch names like `feat/12-add-to-cart`, short-lived branches off `main`, at least one review, squash-merge.
- A fake demo store page in the repo for local testing.
- CI runs lint, type checks and tests for both parts.

Suggested, still to confirm:

- CI also runs on pushes to any branch, so this setup branch gets checked before a PR exists.
- Each store gets a small config file (`server/retailers/demo.yaml`, `jtv.yaml`) with its name, colors, budget defaults and allowed origins.
- Widget tools: Preact, Vite, Vitest, ESLint, Prettier. Server tools: FastAPI, Pydantic, pytest, ruff, mypy.
- The server never changes the cart. A store's cart is tied to the shopper's browser session, so the server only suggests "add this to cart" and the widget does it in the browser.

## How the code is organized

Server:

```
server/src/gotchu/
  core/          business logic: products, budget rules, handling a chat message
  adapters/      the outside world: demo store, JTV (stub), fake AI, later Claude
  api/           FastAPI routes
  bootstrap.py   picks which adapters to use and connects them
```

The one rule: `core/` doesn't import FastAPI, the Anthropic SDK, HTTP libraries, or anything store-specific. It defines simple interfaces like "search the catalog" or "ask the AI", and the adapters fill them in. That way we can swap the fake AI for Claude, or the demo store for JTV, without touching the business logic. A lint check (import-linter) fails CI if someone breaks this rule.

The AI never searches the catalog on its own. It asks for a search, the core runs it and applies the budget, then hands the results back. So the budget rules always apply, whichever AI is plugged in.

Chat history is kept in memory for now. That's fine for local development; we'll move it somewhere like Redis when we deploy.

Widget:

```
widget/src/
  loader.ts   reads the script tag and mounts the widget
  ui/         Preact components
  api/        talks to the server
  host/       store-specific actions: add to cart, read cart, go to checkout (demo, jtv stub)
widget/demo/index.html   a fake store page for local testing
```

A store adds the widget like this:

```html
<script src=".../gotchu.js" data-retailer="demo" async></script>
```

## The first working flow

This is the one thing that has to work end to end:

1. Open the demo store page.
2. Type "ring under 200" in the chat.
3. The widget sends the message to `POST /api/v1/chat`.
4. The fake AI asks for a search for rings under 200. The core searches the demo catalog (about 20 items in a JSON file) and filters by price.
5. The reply comes back with a short message and product cards, each with an "Add to cart" button.
6. Clicking "Add to cart" bumps the cart count on the demo page.

There's also a `GET /health` endpoint.

## Repo layout

```
gotchu/
  server/      Python app, tests, store config files
  widget/      TypeScript widget, demo page, tests
  docs/        setup.md, architecture.md, github-setup.md
  .github/     CI workflow and PR template
  README.md  CONTRIBUTING.md  CLAUDE.md
  .gitignore  .gitattributes  .editorconfig  .env.example
```

## How we work (this goes in CONTRIBUTING.md)

**Tasks.** Every piece of work starts as a GitHub Issue. The board has Backlog, Ready, In progress, In review and Done. The lead splits issues evenly at planning; people assign themselves and pick from Ready. Try to work across different areas so no one ends up the only person who knows a part of the code.

**Branches.** Branch off the latest `main` and name it `type/<issue#>-<short-description>`, for example `feat/12-add-to-cart` or `fix/31-budget-rounding`. Types are feat, fix, chore, docs, refactor, test and spike. Spike branches are for quick experiments and don't get merged. Aim to get a branch merged within a few days.

**Commits.** Same style, for example `feat(server): add budget filter to product search`. Keep them small. Never commit `.env` files or API keys.

**Pull requests.** Use the same format for the PR title, since it becomes the squash commit message. Put `Closes #<issue>` in the description. Keep PRs small, ideally under 400 changed lines. To merge you need one approval and green CI. The author merges with squash and deletes the branch.

**Reviews.** Try to review within a day. Mostly check that business logic stays in `core/`, new logic has tests, and docs are updated if something changed.

If you make a bigger design decision, add a short note about it in `docs/architecture.md`.

## Docs we'll write

- `README.md`: what Gotchu is, and a quick start (clone, run the server, run the widget, open the demo).
- `docs/setup.md`: installing Git, uv and Node on Windows and Mac/Linux, the first run, and fixes for common problems (OneDrive, port already in use, cache issues).
- `CONTRIBUTING.md`: the "How we work" section above, with example commands.
- `docs/architecture.md`: how the code is organized, how a request flows through it, and how to add a new store or AI provider.
- `docs/github-setup.md`: a checklist for the lead. First, what you can do now on the free plan: invite people, allow squash-merge only, auto-delete branches, labels, the project board, and optionally renaming the repo. Second, what it takes to actually block pushes to `main` (a paid plan or a GitHub organization), with the exact settings to turn on.
- `CLAUDE.md`: a short summary of the commands and rules for AI assistants.

## Commands

All of these work in PowerShell and bash. Run the server and the widget in two terminals; the widget's dev server forwards `/api` calls to the Python server.

| | Server (`cd server`) | Widget (`cd widget`) |
|---|---|---|
| Install | `uv sync` | `npm install` |
| Run | `uv run fastapi dev src/gotchu/api/app.py` | `npm run dev` |
| Test | `uv run pytest` | `npm test` |
| Lint | `uv run ruff check .` and `uv run lint-imports` | `npm run lint` |
| Format | `uv run ruff format .` | `npm run format` |
| Type check | `uv run mypy` | `npm run typecheck` |
| Build | | `npm run build` |

## CI

GitHub Actions, with two jobs (`server` and `widget`) on every push and PR:

- `server`: install with uv, then ruff, mypy, lint-imports and pytest.
- `widget`: install with npm, then lint, type check, tests and build.

## Done when

1. Someone can clone the repo (outside OneDrive), follow the README and setup guide exactly in PowerShell, and get the demo working: type "ring under 200", see products, click "Add to cart", see the cart count go up.
2. All checks pass locally, and CI is green on GitHub for this branch.
3. If `core/` imports something from `adapters/`, the import check fails. (Try it once, then undo it.)
4. No secrets in the repo, and nothing needs an API key to run.
5. The docs match what actually runs.
