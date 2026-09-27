# Azure Cost Copilot

Chat with your Azure spend. Azure Cost Copilot pairs a **Microsoft Foundry
chatbot** with an **Azure Cost Analysis dashboard**: ask a plain‑English
question ("why did compute jump last week?") and get an answer grounded in the
same cost figures the charts render, with the evidence attached.

## What it is

- **Backend** — a FastAPI service that validates Entra ID access tokens
  (`Cost.Read` app role), queries Azure **Cost Management**, and calls a Foundry
  model to answer questions grounded in that data. Its answers cite the exact
  metrics they used.
- **Frontend** — a React (Vite) single‑page app: an MSAL sign‑in, a cost
  dashboard (ECharts), and a chat panel. It reads back a correlation id for every
  request so a trace can be followed end to end.

## Architecture

```
            Browser (Entra ID sign-in, MSAL)
                     │  https
              Azure Front Door (Premium)
        /api/*, /health/*  │        │  /*
                 ┌─────────┘        └─────────┐
      backend Container App            frontend Container App
      (FastAPI, user-assigned MI)      (nginx-served SPA)
             │                                  
   Cost Management  ── Foundry (configured model, eastus2)
             └── Application Insights / Azure Monitor / Log Analytics
```

- **Front Door Premium** fronts **two Azure Container Apps environments**,
  **staging** and **production**, both in **`indonesiacentral`**, reached over
  **private link** (the apps have internal ingress only). Routing is by longest
  path prefix: `/api/*` and `/health/*` → backend, everything else → the SPA.
- Each app runs as its own **user‑assigned managed identity** (least privilege:
  only the backend holds *Cost Management Reader* and *Cognitive Services OpenAI
  User*). Users authenticate with **Entra ID**; the backend enforces the
  `Cost.Read` role on every request.
- **Application Insights / Azure Monitor / Log Analytics** capture telemetry,
  and diagnostic settings + alerts are provisioned per environment.
- The **Foundry** model deployment lives in **`eastus2`**.
  The current configuration selects `gpt-5.4-mini`; the model is not chosen by
  the browser.

For the implementation-level request path, operational boundaries and an
editable landing-zone-aware draw.io view, see the
[developer guide](docs/developer-guide.md) and
[architecture diagram](docs/architecture.drawio).
The [static documentation landing page](docs/index.html) is prepared for a
GitHub Pages branch source rooted at `docs/`. It links to the existing
authenticated dashboard; it does not host cost data or run the API.

## Repository layout

```
backend/     FastAPI service (uv-managed, src/cost_copilot), Dockerfile
frontend/    React + Vite SPA, Playwright config, nginx image
infra/        config/*.env, scripts/*.sh (provision, deploy, verify, governance), tests/*.bats
tests/e2e/    Playwright end-to-end suites (dashboard, auth redirect, live-api)
.github/      CI, CodeQL, and the staging/production/promote/infra workflows
docs/         operator runbook and design/plan notes
```

## Development and verification

**Backend** (Python 3.13, [uv](https://docs.astral.sh/uv/)): the API is
validated by the existing CI job and an approved Azure environment. Do not
run FastAPI on a laptop where IT policy disallows it. The commands below are
for authorized development environments only:

```bash
cd backend
uv sync                 # install the locked dependency set
uv run ruff check .     # lint
uv run pytest           # unit + contract tests (integration is opt-in: -m integration)
uv run uvicorn cost_copilot.main:app --reload   # serve on :8000
```

Copy `.env.example` to `backend/.env` and fill in the tenant, subscription,
`ENTRA_API_CLIENT_ID`, and `FOUNDRY_ENDPOINT` before serving.

**Frontend** (Node ≥ 22.12; CI installs via `npm ci` and runs the tool binaries
directly):

```bash
cd frontend
npm ci
npm run dev             # Vite dev server
npm run lint && npm test && npm run build
```

For UI work without Entra sign‑in or a backend, use the **fixture preview
harness** — `frontend/preview.html` renders the dashboard against fixtures
(`vite` serves it; it is never emitted by `vite build`).

**Both together** (containers, only where local container execution is
authorized):

```bash
docker compose up --build   # API on :8000, web on :8080
```

## CI/CD flow

```
feature PR ──▶ staging ──▶ (auto) staging deploy + E2E ──▶ promotion PR ──▶ main ──▶ production deploy
   CI + CodeQL + Copilot review        verify.sh + Playwright      human approval    verify + smoke + telemetry
```

Every PR runs four CI checks (`frontend`, `backend`, `infrastructure`,
`container-build`), CodeQL, and a Copilot review. Confirm the corresponding
rulesets and required approvals in the destination repository: workflow files
alone do not enforce branch protection. Merging to `staging`
auto‑deploys and runs the E2E suites; a green deploy opens a `staging → main`
**merge** PR. Merging that (with production reviewer approval) promotes the
**exact images** staging tested — production never rebuilds; it reads the tested
commit from the merge's second parent. All Azure access is **OIDC only** — no
client secrets or registry passwords anywhere.

## Installing it

See **[docs/installation.md](docs/installation.md)** for a step-by-step
installation from Azure Cloud Shell into this repository, including the
one-time container app configuration that no script performs.

## Operating it

See **[docs/workshop-runbook.md](docs/workshop-runbook.md)** for the end‑to‑end
operator guide: prerequisites, the one‑time OIDC bootstrap, provisioning,
governance, the ship flow, verification, Application Insights KQL,
troubleshooting, and teardown.