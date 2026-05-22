# Multi-Team Collaboration on Azure Data Factory

This document outlines how multiple teams can work on the same Azure Data Factory project simultaneously — with clear access boundaries, safe code contribution workflows, and production safeguards.

---

## Scenario

Your core data engineering team is actively working on an ADF project. A new team joins mid-project to collaborate or extend existing work. The goal is to integrate the new team without disrupting existing pipelines, exposing sensitive resources, or compromising production stability.

---

## Step 1 — Define Access Requirements Before Granting Anything

Before adding any new member or team to the project, answer these three questions:

- What do they need to **read**? (view pipelines, monitor runs, inspect datasets)
- What do they need to **build**? (create pipelines, author dataflows, configure datasets)
- What should they **never touch**? (production environment, shared Linked Services, Key Vault secrets, triggers)

The answers determine the exact role and scope assigned. Do not grant broad access as a convenience — scope every permission deliberately.

---

## Step 2 — Git Repository Access (Code Level)

ADF is connected to a Git repository in Azure DevOps. The new team is added to the repository with the **Contributor** role, which allows them to push to feature branches and raise Pull Requests, but not merge directly to the collaboration branch (`main`).

**Branch naming convention for the new team:**

```
feature/teamb-<descriptive-name>
```

**Workflow:**

1. New team works exclusively on their own feature branches
2. When work is ready, they raise a Pull Request targeting `main`
3. A senior engineer from the core team reviews the PR
4. Only upon approval does the branch merge into `main`
5. Publishing to live ADF happens only from `main`

This ensures all new contributions are reviewed and validated before they affect the shared environment.

---

## Step 3 — ADF Resource Access via Azure RBAC

In the Azure Portal, navigate to the ADF resource, open **Access Control (IAM)**, and assign the appropriate role to the new team.

| Role | Permitted Actions |
|---|---|
| Reader | View pipelines, datasets, and run history. No modifications allowed. |
| Data Factory Contributor | Create and edit pipelines, datasets, and dataflows. Cannot delete resources or manage access. |
| Owner | Full control. Reserved for core team leads only. |

**Standard starting point for a new collaborating team: Data Factory Contributor on the Development ADF instance only.**

Production ADF access is not granted until the collaboration is stable and the team's work has been validated across lower environments.

---

## Step 4 — Branch Protection Policies

Branch policies on `main` act as the final enforcement layer. Configure the following in Azure DevOps under **Repos > Branches > Branch Policies**:

- **Minimum reviewers:** At least one reviewer required, and that reviewer must be from the core team
- **Build validation:** The CI pipeline must pass before any PR is eligible for merge
- **No direct pushes:** Direct commits to `main` are disabled for all contributors

These policies ensure that even a team member with Contributor access cannot introduce untested or unreviewed code into the shared collaboration branch.

---

## Step 5 — ADF Folder Structure and Naming Conventions

Inside ADF, all resources are organised into dedicated folders per team. This provides visual separation in the ADF authoring UI and prevents accidental edits to another team's work.

**Folder structure:**

```
/ Core Team
    core_ingest_sales
    core_transform_orders
    core_load_warehouse

/ Team B
    teamb_ingest_finance
    teamb_transform_invoices
    teamb_load_reporting
```

**Rules:**

- All pipelines, datasets, and dataflows created by the new team must reside in their designated folder
- All resource names must carry the team's prefix (e.g. `teamb_`)
- Cross-folder modifications require explicit sign-off from the owning team

---

## Step 6 — Linked Services and Key Vault Access

**Shared Linked Services** (e.g. Azure SQL connection, ADLS Gen2 connection) are owned and maintained exclusively by the core platform team. The new team may reference these Linked Services in their datasets but may not edit or delete them.

**Key Vault access** is scoped to individual secrets, not the entire vault. If the new team requires credentials for their own data sources, a dedicated secret is created for them in Key Vault and access is granted only to that specific secret via an access policy or RBAC role assignment.

The new team is never granted vault-wide access.

---

## Step 7 — Environment Access Boundaries

| Environment | Core Team | New Collaborating Team |
|---|---|---|
| Development ADF | Full access | Data Factory Contributor |
| Staging ADF | Full access | Read only (for validation) |
| Production ADF | Controlled via release pipeline | No direct access |

The new team develops and tests in the Development environment. Promotion to Staging and Production is handled by the core team through the established Azure DevOps release pipeline, following the standard approval and gate process.

---

## Access Model Summary

```
New Team Joins
      |
      |-- Azure DevOps Repo        Contributor role, feature branches only
      |-- ADF Azure Portal         Data Factory Contributor, Development only
      |-- Key Vault                Scoped to specific secrets only
      |-- Main Branch              Protected; requires core team PR approval
      |-- ADF Folder Structure     Dedicated folder and naming prefix assigned
      |-- Staging / Production     No direct access; core team deploys on their behalf
```

---

## Core Principle

Grant the minimum access required to perform the work. Protect the collaboration branch with mandatory PR reviews from the core team. Never grant production access until the partnership is proven stable across lower environments.
