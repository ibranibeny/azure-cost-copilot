---
layout: default
title: Install Azure Cost Copilot
---

# Install Azure Cost Copilot

A step-by-step installation of the complete application — Entra ID
registrations, GitHub OIDC trust, Azure infrastructure, runtime configuration,
the first staging deployment and the promotion to production — from this
repository, [`ibranibeny/azure-cost-copilot`](https://github.com/ibranibeny/azure-cost-copilot).

- **Where to run it:** every command below runs in **Azure Cloud Shell (Bash)**
  or in GitHub Actions. Nothing runs on your laptop, and the FastAPI service is
  never started locally — it only runs in Azure Container Apps.
- **Cost:** provisioning creates billable resources (Front Door Premium, a
  container registry, two Container Apps environments, Log Analytics and
  Application Insights). Delete them with the
  [teardown](https://github.com/ibranibeny/azure-cost-copilot/blob/main/docs/workshop-runbook.md#9-teardown)
  when you are done.
- **Background:** the [developer guide](developer-guide.html) explains what each
  component does; the
  [operator runbook](https://github.com/ibranibeny/azure-cost-copilot/blob/main/docs/workshop-runbook.md)
  has the KQL queries, troubleshooting notes and teardown referenced below.

---

## Step 0. Decide which deployment this repository will drive

Every Azure and Entra ID name — resource groups, registry, Front Door profile,
managed identities, app registrations — is fixed in
[`infra/config/`](https://github.com/ibranibeny/azure-cost-copilot/tree/main/infra/config),
and every script **reuses an object that already exists** with that name. The
configuration is currently the one behind the running `GithubDay` deployment.
Choose deliberately:

| Option | When | What you change |
| --- | --- | --- |
| **A. Take over the existing deployment** | This repository replaces `GithubDay` as the source of the running app. | Only the repository identity (Step 2a). The existing resources, app registrations and URLs are reused; this repository gains deployment rights to them. |
| **B. Install a separate instance** | You want an independent copy (another subscription, or side by side). | Repository identity **and** every resource/app name (Step 2b). |

> ⚠️ Running the scripts unchanged against the existing subscription is
> option A, whether you intended it or not.

---

## Step 1. Prerequisites

Open **Azure Cloud Shell** in the Azure portal and choose **Bash**. It ships
with `az`, `gh`, `jq` and `git`. Check them:

```bash
az version --query '"azure-cli"' -o tsv   # needs >= 2.60
gh --version | head -1                    # needs >= 2.55
jq --version
```

Access you need **before** you start:

| Need | Why |
| --- | --- |
| **Owner** (or Contributor + User Access Administrator) on the target subscription | Provisioning creates resources and role assignments. |
| **Application Administrator** (or higher) in the Entra ID tenant | Bootstrap creates app registrations, app roles and app-role assignments. |
| **Admin** on `ibranibeny/azure-cost-copilot` | Bootstrap and governance write variables, environments and rulesets. |
| An **Entra ID security group** of users allowed to read cost data | Its members receive the `Cost.Read` app role. Note its object ID. |
| A **Microsoft Foundry** account with a chat-model deployment | The API calls it for answers. The current configuration uses account `aisdgkwm01` in resource group `lab-ai-demo` (`eastus2`), deployment `gpt-5.4-mini`. |
| Quota for **Container Apps** in the workload region and **Front Door Premium** | Default region is `indonesiacentral`. |

Sign in and clone the repository inside Cloud Shell:

```bash
az login                          # Cloud Shell is usually signed in already
az account set --subscription <subscription-id>
gh auth login                     # GitHub.com → HTTPS → authenticate in the browser
gh auth status                    # scopes must include repo and workflow

git clone https://github.com/ibranibeny/azure-cost-copilot.git
cd azure-cost-copilot
export REPO=ibranibeny/azure-cost-copilot
export WORKSHOP_GROUP_OBJECT_ID=<object-id-of-the-cost-readers-group>
```

---

## Step 2. Point the configuration at this repository

The GitHub OIDC trust is created for `repo:<GITHUB_OWNER>/<GITHUB_REPOSITORY>`.
Today [`shared.env`](https://github.com/ibranibeny/azure-cost-copilot/blob/main/infra/config/shared.env)
says `GithubDay`, so a workflow in **this** repository cannot sign in to Azure
until that changes.

### 2a. Repository identity (both options)

Edit `infra/config/shared.env`:

```text
GITHUB_OWNER=ibranibeny
GITHUB_REPOSITORY=azure-cost-copilot
```

The infrastructure tests read that file, so update their expectations in the
same commit or the `infrastructure` CI check fails:

- `infra/tests/provisioning.bats` — the three `ibranibeny/GithubDay` strings
- `infra/tests/preflight.bats` — `{"name":"GithubDay"}`
- `infra/tests/github-rules.bats` — `GH_REPO="ibranibeny/GithubDay"`
- `frontend/Dockerfile` and `backend/Dockerfile` — the `IMAGE_SOURCE` default
  (cosmetic; the workflows pass the real value)

### 2b. Separate instance only

Also change, in `shared.env`, `staging.env` and `production.env`:

- `AZURE_SUBSCRIPTION_ID`, `AZURE_TENANT_ID`, `WORKSHOP_OWNER`
- `FOUNDRY_RESOURCE_GROUP`, `FOUNDRY_ACCOUNT_NAME`, `FOUNDRY_DEPLOYMENT`,
  `FOUNDRY_LOCATION` — your own Foundry account
- `ACR_NAME` (globally unique, 5–50 alphanumerics), `SHARED_RESOURCE_GROUP`,
  `AFD_PROFILE_NAME` — identical in `staging.env` and `production.env`
- every environment name: `RESOURCE_GROUP`, `ACA_ENVIRONMENT_NAME`, app,
  identity, workspace and Front Door endpoint names
- `VNET_ADDRESS_PREFIX` and subnet prefixes, if they would overlap a network you
  peer with (see the [landing-zone notes](#landing-zone-alignment))

and give the Entra ID objects distinct names when you run bootstrap in Step 4
(`API_APP_DISPLAY_NAME`, `SPA_APP_DISPLAY_NAME`, `TEST_IDENTITY_NAME`).

### 2c. Commit before governance is applied

Push this straight to `main` **now**, while the repository has no rulesets yet,
and create the `staging` branch from it:

```bash
git add infra/config infra/tests frontend/Dockerfile backend/Dockerfile
git commit -m "chore: point infrastructure configuration at azure-cost-copilot"
git push origin main
git push origin main:staging
```

---

## Step 3. Preflight

Check the configuration for drift and missing inputs before anything is created:

```bash
bash infra/scripts/preflight.sh
```

Fix every failure it reports before continuing.

---

## Step 4. Bootstrap Entra ID and GitHub OIDC (the only interactive step)

[`bootstrap-oidc.sh`](https://github.com/ibranibeny/azure-cost-copilot/blob/main/infra/scripts/bootstrap-oidc.sh)
creates the API and SPA app registrations, the `Cost.Read` app role, the
deployment managed identities with their federated credentials, the
subscription role grants for the deployers, and the GitHub repository and
environment variables. It is idempotent and never creates or prints a secret.

Preview it, then run it:

```bash
DRY_RUN=1 bash infra/scripts/bootstrap-oidc.sh     # prints what it would change
bash infra/scripts/bootstrap-oidc.sh
```

For option B, prefix both commands with your own names, for example
`API_APP_DISPLAY_NAME="Cost Copilot API (lab2)" SPA_APP_DISPLAY_NAME="Cost Copilot SPA (lab2)" TEST_IDENTITY_NAME=id-cost-copilot-deploy-test-lab2`.

The Infrastructure workflow also reads the cost-readers group, which bootstrap
does **not** store. Set it yourself:

```bash
gh variable set WORKSHOP_GROUP_OBJECT_ID -R "$REPO" --body "$WORKSHOP_GROUP_OBJECT_ID"
```

Verify the variables landed in this repository:

```bash
gh variable list -R "$REPO"
gh variable list -R "$REPO" --env staging
gh variable list -R "$REPO" --env production
```

The repository level needs `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`,
`AZURE_TEST_CLIENT_ID` and `WORKSHOP_GROUP_OBJECT_ID`; each environment needs
its own `AZURE_CLIENT_ID` plus `ENTRA_API_CLIENT_ID`, `ENTRA_SPA_CLIENT_ID` and
`ACR_NAME`.

---

## Step 5. Apply repository governance

Create the `staging` and `main` rulesets (required CI checks, CodeQL, Copilot
review, no force-push or deletion), keep **merge commits only**, and add
reviewers to the `production` environment:

```bash
GH_REPO="$REPO" PRODUCTION_REVIEWERS=<your-github-login> \
  bash infra/scripts/configure-github.sh
```

Confirm squash and rebase are off — production deployment derives the tested
commit from the merge commit's second parent and refuses otherwise:

```bash
gh api "repos/$REPO" --jq '{allow_merge_commit, allow_squash_merge, allow_rebase_merge}'
```

---

## Step 6. Enable the deployment workflows

They were disabled when this repository was created because it had no Azure
trust yet:

```bash
for wf in infra.yml deploy-staging.yml promote.yml deploy-production.yml; do
  gh workflow enable "$wf" -R "$REPO"
done
gh workflow list -R "$REPO"
```

---

## Step 7. Provision the infrastructure

Provisioning runs in GitHub Actions under OIDC. Run the plans **in order**, and
wait for each to succeed:

```bash
for plan in preflight shared staging; do
  gh workflow run infra.yml -R "$REPO" -f plan="$plan" -f environment=staging
  sleep 10
  gh run watch -R "$REPO" "$(gh run list -R "$REPO" --workflow infra.yml -L1 --json databaseId -q '.[0].databaseId')" --exit-status
done

gh workflow run infra.yml -R "$REPO" -f plan=production -f environment=production
```

The production run waits for a **production reviewer** to approve it in the
Actions tab.

| Plan | Creates |
| --- | --- |
| `shared` | Shared resource group, container registry, Front Door Premium profile |
| `staging` / `production` | Resource group, VNet with a `/23` Container Apps subnet and a private-endpoint subnet, Log Analytics, Application Insights, an internal Container Apps environment, the API and SPA container apps (placeholder image), their managed identities and role grants, Front Door routes over private link, diagnostics and alerts |

---

## Step 8. Register the sign-in redirect URIs

The Front Door host names exist only now. Read them and re-run bootstrap so the
SPA registration accepts sign-in redirects from them:

```bash
. infra/scripts/lib.sh
load_config infra/config/shared.env
load_config infra/config/staging.env
afd_host() {
  az afd endpoint show -g "$SHARED_RESOURCE_GROUP" --profile-name "$AFD_PROFILE_NAME" \
    --endpoint-name "$1" --query hostName -o tsv
}
export STAGING_FRONTDOOR_HOSTNAME="$(afd_host cost-copilot-staging)"
export PRODUCTION_FRONTDOOR_HOSTNAME="$(afd_host cost-copilot-production)"
echo "$STAGING_FRONTDOOR_HOSTNAME  $PRODUCTION_FRONTDOOR_HOSTNAME"

bash infra/scripts/bootstrap-oidc.sh
```

(For option B, use your own endpoint names and the same Entra ID name overrides
as in Step 4.)

---

## Step 9. Configure the container apps (one-time, per environment)

> **Why this step exists:** provisioning creates both apps from a placeholder
> image on port 80, and
> [`deploy.sh`](https://github.com/ibranibeny/azure-cost-copilot/blob/main/infra/scripts/deploy.sh)
> only changes the image. No script sets the target ports (API `8000`, SPA
> `8080`) or the runtime settings, so without this step the first deployment
> never becomes healthy.

Run this block once with `ENV=staging` and once with `ENV=production`, **before**
the first deployment:

```bash
ENV=staging    # then repeat with ENV=production
. infra/scripts/lib.sh
load_config "infra/config/$ENV.env"
load_config infra/config/shared.env

HOST="$(az afd endpoint show -g "$SHARED_RESOURCE_GROUP" --profile-name "$AFD_PROFILE_NAME" \
  --endpoint-name "$AFD_ENDPOINT_NAME" --query hostName -o tsv)"
API_ID="$(gh variable get ENTRA_API_CLIENT_ID -R "$REPO" --env "$ENV")"
SPA_ID="$(gh variable get ENTRA_SPA_CLIENT_ID -R "$REPO" --env "$ENV")"
BACKEND_MI="$(az identity list --query "[?name=='$BACKEND_IDENTITY_NAME'].clientId | [0]" -o tsv)"
AI_CS="$(az monitor app-insights component show -g "$RESOURCE_GROUP" -a "$APP_INSIGHTS_NAME" \
  --query connectionString -o tsv)"
FOUNDRY="https://${FOUNDRY_ACCOUNT_NAME}.openai.azure.com/openai/v1/"

# API: listen port, runtime settings; the connection string is kept as an app secret.
az containerapp ingress update -n "$BACKEND_APP_NAME" -g "$RESOURCE_GROUP" --target-port 8000
az containerapp secret set -n "$BACKEND_APP_NAME" -g "$RESOURCE_GROUP" --secrets appinsights-cs="$AI_CS"
az containerapp update -n "$BACKEND_APP_NAME" -g "$RESOURCE_GROUP" --set-env-vars \
  APP_ENVIRONMENT="$ENV" \
  AZURE_TENANT_ID="$AZURE_TENANT_ID" \
  AZURE_SUBSCRIPTION_ID="$AZURE_SUBSCRIPTION_ID" \
  AZURE_CLIENT_ID="$BACKEND_MI" \
  ENTRA_API_CLIENT_ID="$API_ID" \
  FOUNDRY_ENDPOINT="$FOUNDRY" \
  FOUNDRY_DEPLOYMENT="$FOUNDRY_DEPLOYMENT" \
  ALLOWED_ORIGINS="https://$HOST" \
  APPLICATIONINSIGHTS_CONNECTION_STRING=secretref:appinsights-cs

# SPA: listen port and the public identifiers rendered into /config.js at start-up.
az containerapp ingress update -n "$FRONTEND_APP_NAME" -g "$RESOURCE_GROUP" --target-port 8080
az containerapp secret set -n "$FRONTEND_APP_NAME" -g "$RESOURCE_GROUP" --secrets appinsights-cs="$AI_CS"
az containerapp update -n "$FRONTEND_APP_NAME" -g "$RESOURCE_GROUP" --set-env-vars \
  APP_ENVIRONMENT="$ENV" \
  ENTRA_TENANT_ID="$AZURE_TENANT_ID" \
  ENTRA_SPA_CLIENT_ID="$SPA_ID" \
  ENTRA_API_CLIENT_ID="$API_ID" \
  API_BASE_URL="https://$HOST" \
  APPLICATIONINSIGHTS_CONNECTION_STRING=secretref:appinsights-cs
```

Notes:

- After this step the placeholder revision reports **unhealthy**, because the
  placeholder image listens on port 80. That is expected until Step 10 deploys
  the real images.
- Check the Foundry endpoint host: some accounts expose
  `https://<name>.cognitiveservices.azure.com/` instead. Read it with
  `az cognitiveservices account show -g "$FOUNDRY_RESOURCE_GROUP" -n "$FOUNDRY_ACCOUNT_NAME" --query properties.endpoints`
  and keep the `/openai/v1/` suffix.
- The SPA and the API share the Front Door host, so `API_BASE_URL` and
  `ALLOWED_ORIGINS` are the same origin. If you add a custom domain later, add
  it to both and register it as a redirect URI.
- Everything the SPA receives is a public identifier. Never put a secret in the
  SPA's settings.

---

## Step 10. First deployment to staging

Deployments are triggered by merges into `staging`, so ship any small change
through the normal flow:

```bash
git switch -c chore/first-deploy origin/staging
git commit --allow-empty -m "chore: first staging deployment"
git push -u origin chore/first-deploy
gh pr create -R "$REPO" --base staging --title "First staging deployment" --body "Triggers the first deployment."
```

When the required checks pass and the PR is approved, **merge it with a merge
commit**. `deploy-staging.yml` then builds both images, verifies their
provenance, deploys them by digest behind a health gate, runs `verify.sh`, and
runs the browser and live-API end-to-end suites. Watch it:

```bash
gh run watch -R "$REPO" "$(gh run list -R "$REPO" --workflow deploy-staging.yml -L1 --json databaseId -q '.[0].databaseId')"
```

---

## Step 11. Accept staging, then promote to production

1. Open `https://<STAGING_FRONTDOOR_HOSTNAME>/` and sign in with a member of the
   cost-readers group. The dashboard must show cost data and the chat must
   answer with evidence.
2. A green staging run opens a `staging → main` promotion PR. Review it and merge
   it **as a merge commit**.
3. A production reviewer approves the `production` environment. Production
   deploys the **same image digests** staging tested — it never rebuilds — then
   runs `verify.sh production`, a live smoke check and a telemetry check.

---

## Step 12. Verify

```bash
bash infra/scripts/verify.sh staging
bash infra/scripts/verify.sh production
```

It checks identities and least privilege, digest pinning, Front Door routing,
approved private-link connections, the live endpoints, diagnostics and fresh
telemetry. Then sign in to the production host name as a user in the group.

| Symptom | Where to look |
| --- | --- |
| Revision never becomes healthy | Step 9 not done: wrong target port or missing settings. `az containerapp logs show -n <app> -g <rg>` |
| Sign-in fails with a redirect-URI error | Step 8 not re-run with the host names |
| `403` after sign-in | The user is not in the cost-readers group |
| Costs load, chat returns "explanation not available" | `FOUNDRY_ENDPOINT` / `FOUNDRY_DEPLOYMENT`, or the backend identity's *Cognitive Services OpenAI User* grant |
| `429` from cost endpoints | Cost Management throttling; the API retries and caches — wait and refresh |
| Anything else | [Runbook troubleshooting](https://github.com/ibranibeny/azure-cost-copilot/blob/main/docs/workshop-runbook.md#8-troubleshooting-known-live-verify-gotchas) and its Application Insights KQL |

---

## Landing-zone alignment

The scripts deploy the **application landing zone** part of the
[architecture](developer-guide.html): workload resource groups, per-environment
VNets, Container Apps, Front Door, identities and monitoring. They do **not**
create platform landing-zone services. If this workload goes into an
enterprise landing zone, agree these with the platform team **before Step 7**:

- **Subscription placement** under the right management group (for an
  internet-facing app, typically *Landing zones → Online*), and which Azure
  Policy assignments apply — some may deny the resources the scripts create.
- **Address space** — the defaults `10.20.0.0/16` and `10.30.0.0/16` must not
  overlap the hub or other spokes if the VNets are peered.
- **Egress and DNS** — whether outbound traffic to Cost Management, Foundry and
  Entra ID must go through a hub firewall, and which private DNS zones apply.
- **Central monitoring and security** — whether diagnostics must also flow to a
  central Log Analytics workspace, and Defender for Cloud coverage.
- **Identity governance** — who may create app registrations and grant
  subscription-scope *Cost Management Reader*.

See [Azure landing zones](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/).

---

## Uninstall

Follow the
[runbook teardown](https://github.com/ibranibeny/azure-cost-copilot/blob/main/docs/workshop-runbook.md#9-teardown).
For option A, that removes the running `azurecost.contoso.day` deployment too.
