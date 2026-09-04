# Azure Resource Group Provisioning — Design Document

**Author:** Daniel
**Date:** 2026-09-03 (revised)
**Status:** Draft for review

**Changes in this revision:** the Terraform state storage plan is finalized — four dedicated subscriptions, one per management group, each holding that environment's state storage account, isolated at both the RBAC layer and the subscription/billing layer. Disaster Recovery has been pulled out of the shared tables and given its own dedicated section (Section 7), consolidating everything DR-specific — management group, identity, approval policy, state storage, and an open question about whether its configuration should mirror Production's automatically.

## 1. Executive Summary

The enterprise currently provisions Azure resource groups across 118–160 subscriptions by running Terraform locally, by hand, with state discarded after every apply. This document proposes replacing that process with a proper Infrastructure-as-Code system: one GitHub repository, Terraform state persisted per subscription, changes made through pull requests, and applies executed by GitHub Actions using OIDC-federated managed identities instead of a human PIM-elevating to run Terraform on their laptop.

The scope of what's managed is intentionally narrow: resource group creation and lifecycle only. Identity and access management is explicitly out of scope — it belongs to the security team's existing process, and this system is not a substitute for it. That boundary keeps the pipeline's identity free of any IAM-shaped permissions at all, which materially simplifies the security case for this project.

## 2. Current State

- Resource groups are created across an estimated 118–160 Azure subscriptions (exact active count unconfirmed).
- Requests arrive through the enterprise ticketing system. There is an API available, but ticket-to-pipeline automation is explicitly out of scope for this phase.
- An engineer PIMs up to an elevated role, then runs Terraform locally: fills in module variables, `terraform init`, `plan`, `apply`.
- Terraform state is **not persisted** — it's discarded after apply. Terraform is being used as a scripting tool, not as a state-tracking system.
- The local process has, at times, also placed one or two role assignments (Owner or Contributor) on the resource group — though this appears to have been applied inconsistently and may have been limited to Sandbox. Regardless of its past scope, role assignment is not carried forward into the new design (see Section 6.3).
- Nothing is tracked after creation. There is no drift detection, no record of what exists, and no supported path to modify or decommission a resource group through the process that created it.
- GitHub Enterprise is the source control platform in use. Service principals are the current auth mechanism for Azure; there is appetite to move to managed identities.
- IAM and Azure Policy are owned and administered by the security team, independent of this process.
- Subscriptions themselves are created manually by a single individual, whose team affiliation isn't firmly established internally. There is no defined handoff between "a subscription exists" and "this process is aware of it."

## 3. Problems With the Current State

1. **No system of record.** Because state is discarded, nothing durable tracks what Terraform created. The only source of truth is the Azure control plane itself.
2. **No drift detection.** A resource group's configuration can be changed by hand in the portal at any time with no way to detect it.
3. **Human PIM elevation as the control.** A person must hold elevated standing (via PIM) to run Terraform locally. This is the correct instinct for a human actor, but it doesn't map cleanly onto an automated pipeline, and it means the "control" is really just "a person with the right role ran a command," with no review step before changes are applied.
4. **No audit trail beyond the ticket.** There's no PR history, no diff, no reviewer sign-off on the actual change made to Azure.
5. **Local execution risk.** Local applies depend on an individual's laptop state, credentials, and Terraform version — not a controlled, repeatable environment.
6. **IaC paradigm broken.** Using Terraform without persisted state discards its core value proposition (plan/diff against known state) and effectively downgrades it to a templating tool for one-off `az` calls.

## 4. Target Architecture

### 4.1 Guiding principles

- **Desired-state IaC, not a scripted transaction.** State is persisted, imported for existing resources, and used to detect drift going forward.
- **Data-driven, not code-generated.** The estate (subscriptions and resource groups) is expressed as data files consumed by one shared root module — not one Terraform file per subscription.
- **Pipeline-executed, human-approved.** GitHub Actions performs applies using federated managed identities; humans approve via required reviewers on protected GitHub Environments, replacing PIM elevation as the control point.
- **Narrow, honest scope.** The system manages resource group creation and lifecycle only. It has no IAM capability whatsoever — that boundary is structural, not procedural, and is enforced by the permissions granted to its identities (Section 6), not merely by convention.

### 4.2 Repository structure

A single repository, organized by environment, with GitHub Environments providing per-environment identity and approval policy:

```
azure-rg-provisioning/
├── modules/
│   └── resource-group/          # shared root module: resource group only
├── environments/
│   ├── sb/                      # Sandbox — 26 subscriptions
│   │   └── <subscription-id>.yaml
│   ├── np/                      # Non-Prod — 43 subscriptions
│   │   └── <subscription-id>.yaml
│   ├── pr/                      # Production — 43 subscriptions
│   │   └── <subscription-id>.yaml
│   └── dr/                      # Disaster Recovery — 2-3 subscriptions
│       └── <subscription-id>.yaml
├── policy/
│   └── excluded-rg-patterns.yaml
└── .github/workflows/
    └── terraform.yml
```

One repository, one module, four environment folders — not four repositories. This keeps the module logic and the exclusion-pattern list in a single place that can't drift between environments, while GitHub Environments (`sb`, `np`, `pr`, `dr`) still provide independent approval rules and independently scoped Azure identities. Production requires the strictest approval policy of the four; DR's environment, identity, and approval configuration are detailed separately in Section 7 rather than repeated here.

### 4.3 Data-driven configuration

Each subscription is one YAML file. Terraform's root module iterates over these with `for_each`, keyed on a stable identifier (resource group name), never `count` — `count` re-indexes on insertion/removal and is how automated pipelines end up destroying and recreating unrelated resources.

```yaml
# environments/pr/00000000-0000-0000-0000-000000000001.yaml
subscription_id: 00000000-0000-0000-0000-000000000001
subscription_name: "contoso-payments-prod"
resource_groups:
  rg-payments-core:
    location: eastus2
    tags:
      cost_center: "1234"
      owner: "payments-team@contoso.com"
```

No role assignments appear in this schema. The module creates and tags the resource group and nothing else. This is what makes the eventual ticketing integration (explicitly deferred to a later phase) low-risk to add later: it becomes a bot that opens a PR appending a YAML block, not a code generator.

### 4.4 State management

- **Storage location: one dedicated subscription per environment**, not a shared platform subscription. Each of the four management groups (`Sandbox`, `Apps/NonProd`, `Apps/Prod`, `DR`) gets its own subscription holding nothing but that environment's Terraform state infrastructure — no application resources live there. This mirrors the isolation already built into every other part of this design (separate MG, separate identity, separate GitHub Environment) at the one layer where RBAC scoping alone doesn't provide full protection: a subscription can be disabled by Microsoft for a billing or EA enrollment issue, or affected by an organizational change, independent of anything Azure RBAC controls. Four subscriptions means that kind of failure in one environment can't touch the other three's state. It also keeps each environment legible to an auditor — everything under the Production management group, including its own state storage, actually lives under the Production management group.
- **Naming** (adjust to existing conventions): `rg-tfstate-sb` / `rg-tfstate-np` / `rg-tfstate-pr` / `rg-tfstate-dr`, each holding a single storage account (storage account names must be globally unique and 3–24 lowercase alphanumeric characters, e.g. `sttfstatesb001`).
- **One Terraform state file per subscription**, keyed by subscription ID as the blob name within that environment's storage account — not one state for the whole estate, and not one storage account for the whole estate either. A single monolithic state across 160 subscriptions would mean multi-minute plans on every run and an estate-wide blast radius for any state corruption or bad apply.
- **Hard rule:** the applier and reader identities' storage RBAC roles (`Storage Blob Data Contributor` / `Reader`) must be scoped to that environment's specific storage account (or its containing resource group) — never to the subscription. RBAC scope is what actually enforces separation between environments' state; the dedicated subscription is what protects against billing/tenant-level failures. Both matter, and conflating them — for instance, assigning a role at subscription scope "for convenience" — quietly erases the RBAC-level isolation the rest of this design depends on.
- **Locking:** native blob lease locking — no separate lock table needed.
- **Protection:** blob versioning and soft delete enabled on all four state storage accounts before any migration work begins. This is the only rollback path if a bad apply or state corruption occurs, and it costs nothing to turn on early.
- **Bootstrap dependency:** these four subscriptions are themselves new infrastructure and must exist before Phase 0 can begin. Because subscription creation is a manual, single-owner process (Section 10), request all four in one batch immediately rather than sequentially per phase — this is the single most likely thing to stall the project's start if requested piecemeal.

### 4.5 CI/CD workflow

- GitHub Actions, matrixed per subscription.
- On pull request: run `plan` for subscriptions whose YAML changed (diff-based job selection) — a full 150-job matrix on every PR will make the system unusable within a month. Any change under `modules/` or `policy/` is treated differently: because it affects every subscription regardless of which YAML files changed, it triggers a plan across all subscriptions (or at minimum a canary subscription per environment), not zero.
- Concurrency group per subscription, so two PRs touching the same subscription can't race.
- On merge to main: `apply`, gated by the target GitHub Environment's required reviewers (`sb` auto-applies or single-approval; `np` similar; `pr` and `dr` require the stricter reviewer group).
- `prevent_destroy = true` on the resource group resource in the shared module, plus a Terraform-managed `CanNotDelete` resource lock on Production and DR resource groups as defense in depth — `prevent_destroy` only stops this pipeline from destroying a resource group; the lock stops deletion through any path, including the portal.
- `terraform validate`/`fmt` and a linter or scanner (`tflint`, `checkov`, or `tfsec`) run against the shared module in CI before merge — a bug in the one module every subscription depends on has enterprise-wide reach.

### 4.6 Resource group classification

Every resource group discovered in the estate falls into exactly one of three buckets:

1. **Managed** — represented in a YAML file, tracked in state, subject to drift detection.
2. **Excluded** — deliberately out of scope, matched by one of two mechanisms:
   - **`managedBy` is non-null.** Azure stamps this on resource groups owned by another Azure service (AKS node RGs, Databricks workspace RGs, Synapse managed RGs, managed applications). This is queryable via Azure Resource Graph and requires no maintained list — it's structurally reliable.
   - **A reviewed name-pattern list** (`policy/excluded-rg-patterns.yaml`) for platform/default resource groups Azure creates without stamping `managedBy`. Starting list, to be validated against the actual estate via Resource Graph before finalizing:
     - `^NetworkWatcherRG$`
     - `^DefaultResourceGroup-.*$`
     - `^LogAnalyticsDefaultResources$`
     - `^cloud-shell-storage-.*$`
     - `^AzureBackupRG_.*$`
     - `^Default-(Storage|SQL|Web|ServiceBus|EventHub)-.*$`
     - `^Default-ActivityLogAlerts$`
     - `^StreamAnalytics-Default-.*$`

     Patterns must be anchored (full-match, not substring) — an unanchored pattern that accidentally matches a real application resource group makes it silently invisible to the system, which is worse than having no exclusion list at all. Changes to this file should require review from the platform team specifically (see CODEOWNERS note in Section 6.3-adjacent CI guidance), since a bad pattern here is a silent failure, not a loud one.
3. **Unclassified** — exists in Azure, matches neither of the above, not yet represented in a YAML file. This is the import backlog during migration and should reach zero at cutover. After cutover, anything appearing here means a resource group was created outside the process — a governance signal the enterprise doesn't currently have. This bucket needs a named owner and a review cadence once the system is live, or it risks becoming a number nobody actually looks at.

## 5. Import Strategy

Goal: bring all existing (non-excluded) resource groups into state, using Terraform's declarative `import` blocks (Terraform ≥ 1.5) rather than the imperative `terraform import` CLI — declarative imports are plannable and reviewable in a PR before anything happens, which matters when touching production resource groups at this scale.

**Process:**

1. **Discover.** Run an Azure Resource Graph query across all subscriptions to enumerate every resource group, its `managedBy` field, tags, and location.
2. **Classify.** Apply the `managedBy` check and the pattern list to sort results into Excluded vs. candidate-for-import.
3. **Generate.** For each candidate, generate the YAML data entry and the corresponding `import` block from the Resource Graph output — this is a data-generation step, not hand-written HCL.
4. **Review and land in waves.** Import in batches by environment, starting with Sandbox, validating that `terraform plan` reaches a clean no-op (or an intentional, reviewed diff) before moving to the next environment. Production and DR are imported last, once the process has been proven.
5. **Expect friction.** This phase will surface real inconsistencies: resource groups with policy-applied tags that create perpetual diffs, exclusion-pattern gaps, region mismatches. Budget this as the majority of the project's effort — the pipeline itself is comparatively quick to build.

Role assignments are never imported, generated, or reconciled by this process — see Section 6.3.

## 6. Identity & Privilege Model

### 6.1 Authentication: OIDC federation, no stored secrets

GitHub Actions authenticates to Azure via OIDC federated credentials on user-assigned managed identities — one identity per GitHub Environment (`sb`, `np`, `pr`, `dr`), each trusting only that environment's federated subject (`repo:<org>/<repo>:environment:<env>`). No client secrets are stored or rotated. This directly satisfies the goal of moving off service principals.

Each environment splits identity by function:

- A **Reader-scoped identity**, standing, used for all `plan` runs on pull requests.
- A separate **applier identity**, reachable only through the protected GitHub Environment with required reviewers, used solely for `apply`. The human approval that PIM currently provides moves to this merge-time review gate — arguably better logged than a PIM activation, since it's tied to a specific reviewed diff.

### 6.2 Authorization: a custom role, scoped per environment's management group

There is no Azure built-in role limited to resource-group CRUD — `Contributor` and `Owner` both grant full rights over everything created inside the resource group, and `Owner` additionally grants IAM rights. Since this system's job is now exclusively "create, read, update, delete the resource group shell," a purpose-built custom role is the right fit and is easy to justify to security precisely because of how little it can do:

```json
{
  "Name": "Resource Group CRUD",
  "IsCustom": true,
  "Description": "Create, read, update, and delete resource groups only. No rights over resources inside a resource group, and no IAM/role-assignment rights of any kind.",
  "Actions": [
    "Microsoft.Resources/subscriptions/resourceGroups/read",
    "Microsoft.Resources/subscriptions/resourceGroups/write",
    "Microsoft.Resources/subscriptions/resourceGroups/delete"
  ],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/providers/Microsoft.Management/managementGroups/<tenant-root-or-scope>"
  ]
}
```

Tags on the resource group are included in the `write` action's payload, so no separate tag permission is required. This role is assigned once per environment, at that environment's management group node:

| Environment | Management group scope | Nesting |
|---|---|---|
| SB | `Sandbox` | top-level, flat |
| NP | `Apps` → `NonProd` | second-level |
| PR | `Apps` → `Prod` | second-level |

DR's scope and identity are detailed in Section 7 rather than in this table, since Section 7 consolidates everything DR-specific in one place.

Azure RBAC inherits down through management group hierarchy, so assigning the role at each of these nodes covers every subscription nested beneath it without per-subscription assignment. The applier identity holds this role and nothing else. The Reader identity used for PR-time plans can hold the built-in `Reader` role at the same scopes.

### 6.3 IAM and role assignments: explicitly not this system's job

IAM and Azure Policy are owned by the security team. This system does not create, modify, or track role assignments on any resource group, in any environment — including Sandbox, where the previous process may have done so inconsistently. That responsibility is not carried forward. No identity used by this pipeline holds any `Microsoft.Authorization/*` permission, which removes an entire category of risk (self-elevation, over-broad standing access) that would otherwise need to be negotiated with security.

**Open item:** once a resource group is created, something still needs to grant the requester access to it, or the ticket resolves to an unusable empty resource group. Confirm whether the existing ticket already carries a separate task routed to security for access provisioning, independent of resource-group creation. If it does, this system requires no changes to accommodate that — it's already handled elsewhere. If it doesn't, a lightweight handoff (even a manual note on the ticket, consistent with how ticketing automation is deferred elsewhere in this document) needs to be defined so this gap doesn't fall silently between "resource group exists" and "someone can use it."

**CI governance note:** changes to `modules/` or `policy/excluded-rg-patterns.yaml` should require review from the platform team via CODEOWNERS, regardless of which environment folder happens to be touched in the same pull request — these files affect every environment at once, unlike a routine change to a single subscription's YAML.

## 7. Disaster Recovery Environment

DR is treated as a full fourth environment throughout this design, on equal footing with Sandbox, Non-Prod, and Production, even though it currently spans only two or three subscriptions. The same reasoning that keeps every other environment's blast radius contained applies here regardless of scale — isolation is worth the modest extra overhead.

- **Management group:** `DR`, top-level and flat, sitting alongside `Sandbox` and `Apps` in the tenant's management group hierarchy.
- **Repository:** `environments/dr/`, using the same shared module and YAML schema as every other environment (Sections 4.2–4.3).
- **Identity:** its own OIDC-federated reader and applier identities, trusting only the `dr` GitHub Environment's federated subject, holding the same narrow `Resource Group CRUD` custom role (Section 6.2) assigned at the `DR` management group scope.
- **Approval policy:** DR does not merge its GitHub Environment with PR's — it has its own `dr` environment and its own identity, so a compromised or misconfigured DR identity still can't reach PR subscriptions. What it does share is PR's *policy*: the same required-reviewer group and branch protection rules are applied to the `dr` environment, so DR gets production-grade scrutiny without inheriting production's actual access.
- **State storage:** its own dedicated subscription (`rg-tfstate-dr`, per Section 4.4), isolated from the other three environments' state at both the RBAC layer and the subscription/billing layer — the same treatment as SB, NP, and PR, sized for a much smaller environment but not architecturally simplified because of that.

**Open question, worth resolving before Phase 4:** if DR exists to mirror Production for failover, hand-authoring its YAML independently of PR's invites the two to drift apart — exactly the kind of divergence that could make a failover not behave as expected when it's actually needed. Worth deciding whether DR's subscription and resource-group data should be derived or templated from PR's data rather than maintained as a fully separate set of files. This doesn't need to be settled immediately, but it's cheaper to decide before DR's YAML already exists independently (Migration Plan, Phase 4) than to retrofit a derivation mechanism afterward.

## 8. Migration Plan

| Phase | Scope | Key activities |
|---|---|---|
| 0 | Foundation | Request all four state-storage subscriptions (SB, NP, PR, DR) in a single batch immediately, given the manual, single-owner subscription-creation process (Section 10). Once available: storage accounts + resource groups (versioning + soft delete on), repo scaffold, shared module, custom role defined and assigned at each of the four management group scopes, exclusion-pattern list drafted and validated against a Resource Graph sweep |
| 1 | Sandbox (26 subs) | Discover, classify, import, validate clean plans. Prove the pipeline end to end at lowest risk. |
| 2 | Non-Prod (43 subs) | Same process, informed by Phase 1 friction points |
| 3 | Production (43 subs) | Same process; stricter review |
| 4 | DR (2-3 subs) | Migrated using the configuration in Section 7; resolve the PR-mirroring question before this phase begins |
| 5 | Cutover | Local/PIM-based process retired; all new requests go through PR review against the repo |

## 9. Scope Boundary: Platform Repo vs. Workload Repos

This document covers resource-group provisioning only — a narrow, uniform-schema operation well suited to the data-driven YAML pattern above. If the enterprise later wants to track everything inside those resource groups (VMs, storage accounts, databases, networking) in Terraform as well, that should live in a **separate repository** (or set of repositories), not be folded into this one.

The two workloads have fundamentally different shapes: this repo's schema is narrow and uniform, its change frequency is low, and it has one clear owner. Resource-level Terraform has a heterogeneous schema per resource type, much higher change frequency, and — depending on organizational decisions not yet made — potentially many owners (a central platform team, or each application team owning its own resources). Separating repositories gives each workload its own CODEOWNERS, its own branch protection, and critically, its own OIDC-federated identity boundary: a workload repo's identity is scoped to that repo/environment and cannot reach this repo's state storage or its resource-group-creation identity, even accidentally.

The handoff between the two should not depend on this repo's Terraform state. A workload repo should look up the resource group it deploys into via a `data "azurerm_resource_group"` block (by name) or a workflow input, not a `terraform_remote_state` reference — the latter would require granting workload pipelines read access to this repo's state storage, which defeats the purpose of the separation. The only shared contract between the two should be a documented resource-group naming convention; changing that convention later becomes a breaking change for every downstream workload repo, so it's worth getting right early.

Whether the workload layer ends up as one central repo or one per application team is an open organizational question, not a technical one, and is deliberately left unresolved here so it doesn't block shipping the resource-group system described in this document.

## 10. Subscription Onboarding & Key-Person Risk

Subscriptions are currently created manually by a single individual whose team affiliation isn't firmly established within engineering. This sits upstream of everything in this document: the pipeline assumes a subscription already exists and is correctly placed under the right management group. If that placement is inconsistent or if that individual is unavailable, there is currently no defined moment at which a new subscription "enters" this system — someone has to notice it exists and manually add its YAML file. This project now also depends on that same individual for the four new state-storage subscriptions in Section 4.4, which is additional exposure to the same risk, not separate from it — see the Phase 0 note in Section 8 recommending all four be requested together, as early as possible.

Fully automating subscription creation (an Azure "subscription vending" pattern) is a separate initiative and out of scope here, consistent with how ticketing automation is treated elsewhere in this document. What this project should still produce is a short onboarding checklist — what must be true about a subscription before its YAML file is added to this repository (correct management group placement per Section 6.2 and Section 7, required tags, budget in place). That doesn't fix the underlying key-person risk, but it gives the system a defined, checkable entry point instead of an implicit one, and it surfaces a misplaced subscription before it causes an identity-scoping failure at apply time.

## 11. Explicitly Out of Scope (This Phase)

- **Identity and access management.** Role assignments, PIM, and all IAM mechanics belong to the security team's existing process. This system has no permissions in that space at all (Section 6.3).
- **Resources inside resource groups.** VMs, storage accounts, databases, networking, and everything else a workload might need is deliberately excluded from this repository (Section 9).
- **Subscription creation/vending.** Remains a manual, single-owner process for now; only a lightweight onboarding checklist is proposed here (Section 10).
- **Ticketing system automation.** The intake remains manual (a person opens a PR from the ticket) for this phase. The YAML-per-subscription structure is chosen specifically so this can be added later as a bot that opens PRs, without redesigning the core system.
- **Full policy-as-code enforcement** (e.g., OPA/Conftest or Azure Policy assertions specific to this pipeline) — Azure Policy already sits with security and is expected to be the enforcement backstop; this pipeline should surface policy denials cleanly rather than duplicate that validation.
- **Scheduled/automatic drift detection cadence** — to be defined once the estate is fully imported; a nightly or weekly `plan`-only run against all environments is the likely shape.

## 12. Risks

- **Import phase reveals more inconsistency than expected**, extending the timeline. Mitigated by phasing (Sandbox first) and budgeting import as the majority of project effort rather than the pipeline build.
- **Exclusion pattern gaps** could either hide real resource groups from management (if a pattern is too broad) or flood the "unclassified" bucket with noise (if too narrow). Mitigated by anchoring all patterns, validating against a full Resource Graph sweep before relying on the list, and requiring platform-team review on changes to that file specifically.
- **The post-creation access handoff (Section 6.3) is unresolved.** If the ticket doesn't already route access provisioning to security independently, requesters could receive resource groups they can't yet use, with no defined process to close that gap.
- **Subscription-creation key-person risk (Section 10) sits upstream of this entire system**, and this project now depends on that same process four additional times for state-storage subscriptions. Only made visible, not fixed, by an onboarding checklist and the recommendation to batch all four requests immediately.
- **DR configuration drifting from Production (Section 7)** if its YAML is maintained independently rather than derived from PR's data — worth resolving before Phase 4, not after.
- **Repo ownership after go-live.** A desired-state system with no clear long-term owner decays back toward the current situation. Recommend naming an owning team before cutover, not after.
