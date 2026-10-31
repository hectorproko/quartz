---
title: "Salesforce CI/CD with Azure DevOps (Part 8): Lessons and the Enterprise Version"
tags:
  - salesforce
  - azure-devops
portfolio description: What I would keep, what I would change in an enterprise SAFe delivery setup, the honest limits of this project, and what I would build next.
---

---

Part of the series [[01-overview|Salesforce CI/CD with Azure DevOps]].

## The ten things I will remember

1. **Git is the source of truth, not the org.** Every change to QA and PROD came from a commit, through a PR, through the pipeline.
2. **Environment parity is a process, not a setting.** Same source, same template, same deploy command, different inputs. The only thing that broke parity was a manual change.
3. **Validate against the real target before approving.** Check-only deploys plus tests against QA and PROD caught every problem before it changed an org.
4. **A pipeline is only a gate if branch policies make it one.** And a policy that says "enabled" can still be bound to the wrong pipeline.
5. **Salesforce tests are a deployment gate.** 75% coverage is the floor; assertions are what matter.
6. **Profiles are a trap; permission sets deploy cleanly.** And deploying a permission set does not assign it.
7. **Salesforce deploys never delete.** Rollbacks need destructive changes, planned in advance.
8. **Secrets go where they belong:** keys in secure files, config in variable groups, one key per environment, a negative test to prove isolation, and Open access off so first use needs approval.
9. **Verify by asking the system**, and find runs by commit, not by "latest".
10. **Shift left.** A 30-second local test run beats a pipeline round trip. CI is the safety net.

---

## How each failure maps to a control

|Failure|Control that caught or would prevent it|
|---|---|
|Tests passed in DEV, failed in QA (field access from profiles)|Manual validation against QA before automating, and tests that run with the permission set|
|`grep -c` printed `0` after `sf` failed|`set -euo pipefail` in every pipeline script|
|PR validation never triggered|Checking the policy binding with the `az` CLI|
|Tests not updated with the code|Required build validation policy on the PR|
|Drift in QA|"Pipeline only" rule; in an enterprise, restricted admin access plus drift detection|
|Rollback that did not delete|PROD validation of the rollback PR, then a destructive changes manifest|

---

## What changes in an enterprise SAFe setup

The Deloitte role works across **Salesforce, MuleSoft and Informatica workstreams** inside a **SAFe** delivery structure. My project is one workstream, one developer, free orgs. Here is what I would do differently at that scale.

### Environments

|Mine|Enterprise|
|---|---|
|2 Developer Edition orgs|A sandbox chain: Developer sandboxes, an integration sandbox, QA/UAT (Partial Copy), Staging (Full Copy), Production|
|No refresh possible|**Scheduled sandbox refreshes** from production, aligned to the release calendar, followed by a redeploy of everything not yet released|
|Parity enforced only by a rule|Parity enforced by **access control** (no admin rights outside the pipeline's integration user) and **automated drift detection**|
|One org per stage|Many orgs per stage, often one per team or Agile Release Train (ART)|

### Pipeline

- **Delta deployments.** I deploy all of `force-app` every time. At scale that is slow and risky. Teams generate a manifest of **only what changed** between two commits (for example with the `sfdx-git-delta` plugin) and deploy that.
- **Test level.** I used `RunLocalTests` everywhere. Large orgs often use `RunSpecifiedTests` on PRs for speed, and `RunLocalTests` for production.
- **Quick deploy into the release window.** Validate against production before the window, then `sf project deploy quick` during it. My pipeline validates on the release PR, but the PROD stage still does a full deploy with tests.
- **Static code analysis** on every PR (Salesforce Code Analyzer, PMD) before tests run.
- **A dedicated integration user** for JWT, not a person's account.
- **Destructive changes as a release artifact**, reviewed with the release, and removed from the repo after the release ships so it does not run again.
- **Pipeline changes released separately** from application changes.
- **Agents:** a pool of self-hosted agents inside the company network or Azure, with only approved tooling, rather than one laptop.

### Release process

|Mine|SAFe enterprise|
|---|---|
|I approve my own PROD deploys|Release manager or Change Advisory Board approves; self-approval off; 2+ reviewers|
|Approval comment in Azure DevOps|Approval linked to a **change ticket** (for example ServiceNow) through the approvals API|
|Basic process: Issues|**Agile or Scrum process**: Epics, Features, User Stories in sprints, rolled up to the ART's Program Increment (PI)|
|Release whenever I want|**Sprint-cadence releases** on a release train calendar, with business-hours and exclusive-lock checks on production|
|`v1.x.y` tags on one repo|Coordinated release versions across workstreams, with release notes generated from linked work items|

### Cross-workstream coordination

In a SAFe program with Salesforce, MuleSoft and Informatica:

- A Salesforce change (a new field) often needs a **matching MuleSoft API change** and an **Informatica mapping change**. Those have to ship in the right order, or integrations break.
- Each workstream has its own pipeline, but releases are **coordinated** on the same train, often with a release orchestration stage or runbook that sequences them.
- **Environment synchronization** means more than one org: the QA Salesforce org must talk to the QA MuleSoft environment and the QA Informatica environment, all at compatible versions.

This project does not touch MuleSoft or Informatica. But the same ideas apply: same source and same pipeline per environment, validate against the real target, gate production, and make rollbacks a planned part of every release.

---

## Honest limits of this project

- **Personal, solo, and guided.** I followed a step by step build and wrote these pages to understand it. Nothing here ran in an enterprise.
- **Free Developer Edition orgs**, not sandboxes. No refresh, few licenses, small limits.
- **I approved my own changes** in every gate, because I was the only person.
- **Full deploys of the whole project** on every run, which is fine for 15 components and not for thousands.
- **The destructive manifest is still in the repo** after the rollback. A later release should remove it.
- **The GitHub Actions version** of the same pipeline (my original stretch goal) is not built yet.

---

## What I would build next

1. **Mirror the pipeline in GitHub Actions**, to compare the two platforms side by side (environments and required reviewers, secrets, reusable workflows instead of templates).
2. **Scheduled drift detection:** a nightly job that retrieves QA and PROD metadata and fails if it differs from the branch that should be deployed there.
3. **Delta deployments** with a generated manifest of changed components.
4. **Quick deploy for PROD:** reuse the validation ID from the release PR inside the approved deploy, so the release window does not rerun tests.
5. **Static analysis** as the first PR check.

---

Back to the start: [[01-overview|Part 1: Overview and architecture]]