# CGE-AZ GRC pipeline

An automated GRC engineering pipeline on Azure: discover coverage, enable the
gap, store evidence we own, report only from that store, and remediate through a
named least-privilege identity. Humans authorize escalation. The repo is the
change path; the portal is an exception the drift detectors are watching.

This README is how to **deploy the pipeline from an empty subscription**. Course
lab write-ups stay in `labs/` (gotchas and waits). Control catalog:
[`docs/CONTROLS.md`](docs/CONTROLS.md). Stage flow and identity boundaries:
[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md). Why non-obvious choices were
made: [below](#why-these-choices).

**Candidate-added control:** `cge-deny-http-storage` — storage must require
HTTPS (Deny). Proven with a failed create (`https-deny-evidence.txt` locally).
Mapped in CONTROLS.md. Distinct from the starter’s public-blob deny (who can
see the files vs how bits travel).

## Prerequisites

- Azure subscription you own (free account is enough) and `az login`
- Terraform >= 1.9, Azure CLI ~2.90, Python 3.11+, `zip` or 7-Zip
- Providers registered: `Microsoft.Management`, `OperationalInsights`,
  `Security`, `DocumentDB`, `Web`, `Storage`, `Insights`, `PolicyInsights`
- Consumption (Y1) quota in at least one region — run
  `./labs/00-setup/probe-quota.sh`. This subscription: **centralus** and
  **westus3** had quota; most other US regions did not. Stages 03/04 default
  `functions_location` to `centralus`.
- Git Bash on Windows: `export MSYS_NO_PATHCONV=1` before any argument that
  starts with `/` (resource IDs, `terraform import`).

Cost guardrails: free-account spending protection, $200 credit, and a $10/month
budget with 80% actual + 100% forecast alerts (`labs/01-sandbox/create-budget.sh`).

## Deploy order (empty subscription)

Each `stages/*` directory is its **own Terraform root** with **its own state
file**. Do not combine them. Compose through outputs / remote state only.

Export once per shell:

```bash
export MSYS_NO_PATHCONV=1
export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export TF_VAR_owner_email=you@example.com   # becomes the RG owner tag / POA&M owner
```

### 1. Hierarchy and budget (once, by CLI)

Create `mg-grc` → `mg-grc-sandbox` → this subscription, tagged
`rg-grc-sandbox-dev`, Reader for `grc-auditors` at **RG scope only**, and the
$10 budget. Commands: `labs/01-sandbox/README.md`.

### 2. Remote state (chicken-and-egg)

```bash
cd labs/03-foundation && ./bootstrap.sh
```

Creates versioned, private state storage and writes `backend.hcl` (gitignored —
per-learner). Grants you **Storage Blob Data Contributor** on that RG (data
plane; Owner is not enough). Wait 1–3 minutes if `terraform init` returns 403.

```bash
export TF_VAR_state_storage_account=$(grep storage_account_name backend.hcl | cut -d'"' -f2)
```

### 3. Foundation — import what exists, then apply

```bash
cd ../../stages/01-foundation
terraform init -backend-config=../../labs/03-foundation/backend.hcl
# import mg-grc, mg-grc-sandbox, rg-grc-sandbox-dev, law-grc-sandbox (see labs/03-foundation)
terraform apply
```

Creates the evidence RG, custom policies + initiative (including
**HTTPS-only Deny**), assignment on `mg-grc-sandbox`, and
`id-grc-remediation-dev` (Monitoring Contributor only).

### 4. Activation (discover, then enable)

```bash
cd ../02-activation
terraform init -backend-config=../../labs/03-foundation/backend.hcl
# import StorageAccounts pricing + nist-csf-20 if you enabled them by hand
terraform apply
```

`terraform output` `current_plan_tiers` / `activation_needed` is the coverage
inventory. Only plans that are not already Standard are created.

### 5. Evidence store + collector

```bash
cd ../03-evidence-store
terraform init -backend-config=../../labs/03-foundation/backend.hcl
terraform apply   # Cosmos can take several minutes; default location eastus2
```

Zip **contents** of `functions/collect_assessments` (`host.json` at archive
root) and `az functionapp deployment source config-zip` onto
`collector_function_app` in `rg-grc-evidence-dev`. Seed frameworks:

```bash
export COSMOS_ENDPOINT=$(terraform output -raw cosmos_endpoint)
python labs/04-evidence/seed_frameworks.py
```

Trigger `collect_now`; nightly timer is 05:00 UTC.

### 6. Reports (read the store only)

```bash
cd ../04-reporting
terraform init -backend-config=../../labs/03-foundation/backend.hcl
terraform apply
```

Zip-deploy `functions/reports`. Trigger `poam_now` and `sar_now` **once per
calendar day** (WORM + dated paths; same-day overwrite is blocked). Timers:
POA&M daily 06:00 UTC, SAR weekly Monday 07:00 UTC.

Reproduce a SAR number in Cosmos Data Explorer:

```sql
SELECT VALUE COUNT(1) FROM c WHERE c.runId = "<runId>" AND c.status = "Unhealthy"
```

### 7. Enforcement (leave dry-run)

```bash
cd ../06-enforcement
terraform init -backend-config=../../labs/03-foundation/backend.hcl
terraform apply   # remediation_mode defaults to dry-run
```

Modify policy is assigned `DoNotEnforce`. You create the remediation task
(human gate). Caller on the fix is `id-grc-remediation-dev`, not a person.

### 8. CI (your fork only)

`labs/06-loop/arm-your-fork.sh <your-github-user>` federates OIDC to **your**
fork. Add the five printed **variables** (not secrets). Never arm
`GRCEngClub/cgeaz` — on a public repo, `pull_request` subjects match PRs from
any fork.

Forks created after 15 July 2026 emit **immutable** OIDC subjects
(`repo:OWNER@id/REPO@id:pull_request`). Federated credentials must match that
`sub` exactly. `backend.hcl` is gitignored; CI init needs those values from
variables or an equivalent `-backend-config`.

## Layout

```
stages/     one root module per stage, separate state, outputs as the contract
functions/  collector and reporters — separate Function Apps (identity boundary)
policy/     conftest/OPA: public blob, HTTPS, shared keys, identity, broad roles
docs/       CONTROLS.md, ARCHITECTURE.md, this deploy path
labs/       course labs (waits, CLI potholes)
.github/    compliance-gate (PR) and drift-detection (nightly plan)
```

## Why these choices

| Choice | Why |
|---|---|
| Separate state per stage | Blast-radius walls. A reporting mistake must not rewrite foundation. |
| Discovery then activation | Enabling “everything Standard” blindly fights reality and can destroy the plan you just created (read-your-own-writes). |
| Functions in `centralus` | Free-account Y1 quota is regional and zero in most US regions. Probe first. |
| Cosmos default `eastus2` | East US often lacks Cosmos capacity on new subscriptions. |
| Collector ≠ reporter | SoD: the identity that writes evidence cannot author reports. |
| Remediation ≠ Owner/Contributor | One named identity; roles are a whitelist of the job. Activity Log caller is the audit story. |
| Deny on public blob **and** HTTPS | Two PR.DS failures. Escalation is a reviewed variable, not a portal click. |
| Reports read Cosmos only | A number you cannot replay tomorrow is not evidence. |
| WORM unlocked | Tamper-evident while `terraform destroy` at course-end still works. |
| Enforcement default dry-run | Automation proposes; a human creates the remediation task. |
| Upstream CI unarmed | Public `pull_request` OIDC would hand every fork a path to the course subscription. |

## Verify before you call it live

```bash
./self-check.sh
```

Plus: WORM delete → `BlobImmutableDueToPolicy`; HTTPS create with
`--https-only false` → `RequestDisallowedByPolicy` / `cge-deny-http-storage`;
a SAR `findings` count equals a Cosmos query on that `runId`;
`terraform output remediation_mode` is `dry-run`.

## Teardown (reverse)

`terraform destroy` in **06 → 04 → 03 → 01**, then
`az security pricing create --name StorageAccounts --tier Free`, then
`az ad app delete --id <AZURE_CLIENT_ID>` if you armed a fork.

---

Control mappings: [`docs/CONTROLS.md`](docs/CONTROLS.md) · Rubric: [`docs/RUBRIC.md`](docs/RUBRIC.md)
