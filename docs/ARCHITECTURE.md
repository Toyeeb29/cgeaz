# Architecture — stage flow and identity boundaries

An assessor should be able to answer two questions from this page: **what runs
in what order**, and **which badge is allowed to do what**.

## Stage flow

```
                    ┌─────────────────────────────────────────┐
                    │  01-foundation  (state: 01-foundation)  │
                    │  mg-grc → sandbox → subscription        │
                    │  policies + initiative                  │
                    │  id-grc-remediation-*                   │
                    │  law-grc-sandbox, evidence RG           │
                    └───────────────┬─────────────────────────┘
                                    │ outputs (remote state)
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
   02-activation          03-evidence-store         06-enforcement
   discover pricings      Cosmos + WORM             modify public-blob
   enable only the gap    collector Function        dry-run default
              │                     │
              │                     ▼
              │            04-reporting
              │            reporter Function
              │            POA&M + SAR → WORM
              ▼
        Defender assessments ──► collector ──► Cosmos ──► reports
                                      ▲
                                      │ nightly / on-demand
```

Downstream stages **read outputs**. They do not mutate another stage’s resources.
That is the blast-radius contract.

| Stage | State key | What it owns |
|---|---|---|
| 01-foundation | `01-foundation.tfstate` | Hierarchy, baseline policies (incl. HTTPS Deny), remediation identity, log workspace, evidence RG |
| 02-activation | `02-activation.tfstate` | Defender plan coverage; CSF 2.0 assignment |
| 03-evidence-store | `03-evidence-store.tfstate` | Cosmos, WORM `reports`, collector app + its roles |
| 04-reporting | `04-reporting.tfstate` | Reporter app + Cosmos **read** + blob **write** |
| 06-enforcement | `06-enforcement.tfstate` | `cge-fix-public-blob` + Storage Account Contributor when mode ≠ audit |

## Identity boundaries (principal / role / scope)

Same sentence as Lab 1: **who + what + where**.

```
You (user) ──► Storage Blob Data Contributor @ rg-grc-tfstate
               (Terraform state data plane; Owner is not enough)

grc-auditors ──► Reader @ rg-grc-sandbox-dev only

id-grc-remediation-dev
    ──► Monitoring Contributor @ mg-grc-sandbox
    ──► Storage Account Contributor @ mg-grc-sandbox   (only when remediation_mode ≠ audit)
    Never Owner. Never Contributor. Filter Activity Log by this caller.

Collector Function (system-assigned)
    ──► Security Reader @ subscription     (read Defender, change nothing)
    ──► Cosmos Data Contributor @ evidence account
    Cannot write reports.

Reporter Function (system-assigned, separate app)
    ──► Cosmos Data Reader @ evidence account
    ──► Storage Blob Data Contributor @ evidence storage (WORM reports)
    No Security Reader. No Cosmos write.

GitHub Actions (OIDC app, your fork only)
    ──► Contributor @ mg-grc  (plan / refresh; still cannot grant RBAC)
    ──► Storage Blob Data Contributor @ rg-grc-tfstate
    Federation subject names this fork (immutable owner/repo IDs after Jul 2026).
```

If a job needs both “write evidence” and “write the narrative,” it is the wrong
design. Split the Function App; that **is** the identity boundary.

## Evidence plane

```
Defender assessments API
        │  collector (managed identity)
        ▼
Cosmos  grc / assessments   (runId, collectedAt, upsert on deterministic id)
        grc / frameworks    (CSF 2.0 seeded as data)
        grc / mappings      (crosswalk — collect once)
        │
        │  reporter (different identity)
        ▼
Blob    reports/   WORM (unlocked)   poam/YYYY/MM/   sar/YYYY/MM/
```

A report number is a stored query, not a live API call at generate time.

## Enforcement ladder

`var.remediation_mode`: **audit** → **dry-run** → **enforce**.

| Mode | Policy effect | Assignment | Who remediates |
|---|---|---|---|
| audit | Audit | DoNotEnforce | nobody — observe only |
| dry-run (default) | Modify | DoNotEnforce | you create the task |
| enforce | Modify | Default | platform, still as the remediation identity |

De-escalating Deny → Audit on `public_blob_policy_effect` is the same idea: a
reviewed one-line apply, not a portal toggle.

## Candidate-added control

`cge-deny-http-storage` lives in stage 01, in the same initiative, reference
`deny-http-storage`. Blast radius: **creates** of storage with
`supportsHttpsTrafficOnly=false` are rejected. It does not modify existing
accounts (no Modify/DINE). Rollback: set `https_policy_effect=Disabled` or
`Audit` and apply. Proof: `RequestDisallowedByPolicy` on `--https-only false`.
