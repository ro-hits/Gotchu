# Gotchu: project foundation

Date: 2026-11-11 (simplified 2026-11-16 and 2026-11-18)

## What this is

Gotchu is a shopping assistant chatbot that a store adds to its site with one script tag. It helps shoppers find products within their budget, add them to the cart and get to checkout. JTV is the first store, but the code shouldn't assume JTV.

This doc covers only the first PR: the repo, a working skeleton and CI. It lands as one PR from `chore/1-project-foundation`. PRs after that stay small.

## Scope

In:

- `server/` (Python 3.13, FastAPI, uv) and `widget/` (TypeScript, built into a single JS file).
- One flow that works end to end against a fake AI and a demo store, so nobody needs an API key.
- CI that runs lint, type checks and tests.
- A README covering setup, commands and team rules.

Out: the real Claude and JTV integrations, deployment, auth and analytics.

## Architecture

Server:

```
server/src/gotchu/
  core/          business logic: products, budget rules, handling a chat message
  adapters/      demo store, fake AI (Claude and JTV come later)
  api/           FastAPI routes
  bootstrap.py   wires the adapters into core
```

- `core/` doesn't import FastAPI, the Anthropic SDK, HTTP libraries or anything store-specific. It defines interfaces like "search the catalog" and "ask the AI", and adapters implement them. import-linter enforces this in CI.
- The AI never searches the catalog itself. It asks for a search, then core runs it, applies the budget and hands back the results. That way the budget rules hold no matter which AI is plugged in.
- The server never changes the cart. A store's cart is tied to the shopper's browser session, so the server only suggests "add to cart" and the widget does it.
- Chat history is kept in memory for now. It moves to something like Redis when we deploy.
- Each store gets a small config file with its name, colors, budget defaults and allowed origins. For now there's only `server/retailers/demo.yaml`.

Widget:

```
widget/src/
  loader.ts   reads the script tag and mounts the widget
  ui/         components
  api/        server client
  host/       store actions: add to cart, read cart, go to checkout (demo only for now)
widget/demo/index.html   fake store page for local testing
```

The widget renders inside a Shadow DOM so its styles and the store's styles don't clash. A store embeds it like this:

```html
<script src=".../gotchu.js" data-retailer="demo" async></script>
```

## The first working flow

1. Open the demo store page.
2. Type "ring under 200".
3. The widget sends the message to `POST /api/v1/chat`.
4. The fake AI asks for a search for rings under 200. Core searches the demo catalog (about 20 items in a JSON file) and filters by price.
5. The reply has a short message and product cards, each with an "Add to cart" button.
6. Clicking "Add to cart" bumps the cart count on the demo page.

There's also a `GET /health` endpoint.

## Repo layout

This is everything the first PR adds at the top level:

```
gotchu/
  server/              app, tests, demo catalog, store config
  widget/              widget, demo page, tests
  docs/                design docs like this one
  .github/workflows/   ci.yml
  README.md
  .gitignore
  .gitattributes       forces LF line endings
```

`.gitattributes` is there because Git for Windows checks files out with CRLF by default, and Prettier and the Linux CI expect LF.

## Setup and commands

The team is on Windows, so the README targets PowerShell. You need Git, Python 3.13, uv 0.10 or newer and Node 24 LTS. Clone outside OneDrive, since the synced Desktop and Documents folders cause file lock errors.

Run the server and the widget in two terminals. The widget's dev server forwards `/api` calls to the Python server.

|            | Server (`cd server`)                              | Widget (`cd widget`) |
| ---------- | --------------------------------------------------- | ---------------------- |
| Install    | `uv sync`                                         | `npm install`        |
| Run        | `uv run fastapi dev src/gotchu/api/app.py`        | `npm run dev`        |
| Test       | `uv run pytest`                                   | `npm test`           |
| Lint       | `uv run ruff check .` and `uv run lint-imports` | `npm run lint`       |
| Format     | `uv run ruff format .`                            | `npm run format`     |
| Type check | `uv run mypy`                                     | `npm run typecheck`  |
| Build      |                                                     | `npm run build`      |

## How we work

This goes in the README.

- Work starts as a GitHub Issue on the project board. The lead splits issues evenly at planning.
- Branch off `main` as `type/<issue>-<short-description>`, for example `feat/12-add-to-cart`. Types are feat, fix, chore, docs, refactor and test.
- Commits and PR titles use the same style, for example `feat(server): add budget filter to product search`. The PR title becomes the squash commit.
- Keep PRs small (under about 400 lines) and put `Closes #<issue>` in the description. Merging needs one approval and green CI. Squash-merge, then delete the branch.
- Never commit `.env` files or API keys.

## CI

One GitHub Actions workflow with two jobs:

- `server`: uv sync, ruff, mypy, lint-imports, pytest.
- `widget`: npm ci, lint, type check, tests, build.

## Done when

1. Someone can make a fresh clone on Windows, follow the README in PowerShell, and get the demo working: type "ring under 200", see products, click "Add to cart" and watch the cart count go up.
2. CI is green on this branch.
3. Importing from `adapters/` inside `core/` fails lint-imports. Try it once, then undo it.
4. There are no secrets in the repo, and nothing needs an API key to run.

## To confirm

- CI runs on pushes to any branch, not just PRs, so this branch gets checked before a PR exists.
- One YAML config per store. `jtv.yaml` gets added when the JTV work starts, not as a stub now.
- Tooling: Preact, Vite, Vitest, ESLint and Prettier for the widget. FastAPI, Pydantic, pytest, ruff and mypy for the server.
- The server never changes the cart (see Architecture).
- The GitHub repo is still named `gotcha`. Rename it, or keep it?
- A private repo on the free plan can't block direct pushes to `main`. Until we move to an org or a paid plan, "no direct pushes" is a team rule.

## Left out (ask if you want any of these)

- `CONTRIBUTING.md`, `CLAUDE.md`, `docs/setup.md`, `docs/architecture.md` and a GitHub settings checklist.
- A PR template, `.editorconfig` and `.env.example`. The `.env.example` can come with the Claude integration.
- JTV stubs in the server and the widget.
