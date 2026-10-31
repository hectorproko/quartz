---
title: "Salesforce CI/CD with Azure DevOps (Part 4): Environments and Parity"
tags:
  - salesforce
  - ci/cd
portfolio description: Mapping DEV, QA and PROD to free orgs, pinning the API version, checking that the targets are clean, and running the first validation deploy by hand, which failed for a parity reason.
---
---

Part of the series [[01-overview|Salesforce CI/CD with Azure DevOps]].

The Deloitte role talks about **maintaining environment parity**. Before automating anything, I wanted to understand what parity means for Salesforce and test it by hand.

> [!abstract] Environment parity, in one sentence Every environment gets **the same source from git**, deployed **the same way** (by the pipeline), so that what passed in QA is what reaches PROD. Nobody clicks changes directly into QA or PROD.

---

## 1. The environment map

|Env|What I used|Typical enterprise setup|How it is kept in line with PROD|
|---|---|---|---|
|DEV|Scratch org from `project-scratch-def.json`|Scratch orgs or Developer sandboxes|Rebuilt from code, so it is disposable|
|QA|Developer Edition org #2|Partial or Full Copy sandbox|Enterprise: refreshed from PROD periodically. Mine: deployed only by the pipeline|
|PROD|Developer Edition org #1 (also the Dev Hub)|Production org|Changes only through the pipeline, after approval|

**An honest limitation:** real sandboxes can be **refreshed** from production, which resets them to match. Developer Edition orgs cannot be refreshed. So in my setup, the "only the pipeline deploys" rule is the **whole** parity control. Part 7 shows what happens when that rule is broken.

---

## 2. Check API versions across environments

Each org runs a Salesforce release, which supports a certain **API version**. My project was generated with `sourceApiVersion: 67.0`.

```bash
for o in dev qa prod; do
  echo "== $o"
  sf org display --target-org $o | grep -iE "api version|instance url"
done
```

```
== dev
│ Api Version     │ 68.0 │
== qa
│ Api Version     │ 68.0 │
== prod
│ Api Version     │ 68.0 │
```

(`grep` keeps only those two lines, so the access token in the full output never gets printed.)

**Why this matters in the real world:** Salesforce upgrades **sandboxes about a month before production** every release (the "preview window"). For a few weeks, QA can run a newer release than PROD, and something that passes in QA can fail in PROD. Pinning `sourceApiVersion` to a version **every environment supports** is one way teams protect against that.

I had already seen the mismatch in my own output earlier:

```
Retrieving v67.0 metadata ... using the v68.0 SOAP API
```

The org talked API 68, but the project asked for files in the 67.0 format. Deploying an older format to a newer org is safe; the other direction breaks. All three orgs reported 68.0, so I pinned the project to it:

```bash
sed -i 's/"sourceApiVersion": "67.0"/"sourceApiVersion": "68.0"/' sfdx-project.json
git commit -am "chore: pin sourceApiVersion to 68.0 to match all environments"
git push
```

The commit went onto the feature branch, so the open PR !1 picked it up automatically.

---

## 3. Confirm QA and PROD are clean starting points

Before the first automated deploy, I wanted to know what was already in the targets. Three things could cause surprises:

1. **`Expense__c` already existing**, so the first deploy would update it instead of creating it.
2. **Existing Apex in the org.** A production deploy with `RunLocalTests` runs **all** local tests in the org and checks coverage across **all** of its Apex, including code I did not write. New orgs sometimes come with sample code.
3. **Different installed packages** in QA and PROD. That is configuration drift.

```bash
for o in qa prod; do
  echo "== $o"
  echo -n "Expense__c present: ";      sf org list metadata --metadata-type CustomObject --target-org $o | grep -ci "expense__c"
  echo -n "Unmanaged Apex classes: ";  sf org list metadata --metadata-type ApexClass   --target-org $o | grep -c "unmanaged"
  echo -n "Unmanaged Apex triggers: "; sf org list metadata --metadata-type ApexTrigger --target-org $o | grep -c "unmanaged"
  echo "Installed packages:";          sf package installed list --target-org $o
done
```

- **"Unmanaged"** means code that belongs to the org itself. **"Installed"** means code that came from a package.
- `grep -c` counts matching lines, so `0` means none.

|Check|QA|PROD|
|---|---|---|
|`Expense__c`|absent|absent|
|Unmanaged Apex classes|0|0|
|Unmanaged Apex triggers|0|0|
|Installed packages|`devedapp` 0.9.0.2|`devedapp` 0.9.0.2|

Both targets were clean and matched each other. The only Apex in PROD was 4 classes from the preinstalled `devedapp` package, marked **installed**, so their tests do not count against my deploys. My DEV scratch org probably did not have that package. That is acceptable drift, because nothing in my code depends on it.

### A lesson hiding in the output

On the first run, the PROD lines printed this:

```
== prod
Expense__c present: Error (1):

0
Unmanaged Apex classes: Error (1):

0
```

The `sf` command **failed** (an intermittent error, which a rerun with `--json` confirmed was transient), but `grep -c` still printed `0`, which looks like a clean result.

**Why:** in a pipe, the shell only reports the exit code of the **last** command. `grep` succeeded, so the failure of `sf` was hidden.

**What I did about it:** every script step in the pipeline starts with `set -euo pipefail`, so a failure anywhere in a pipe fails the whole step. This is one of the most common ways a CI job ends up green when it should be red.

---

## 4. A manual validation deploy to QA

Before automating, I ran by hand **exactly what the PR pipeline would run**. If the automated version failed later, I would know whether the problem was the pipeline or the deploy itself.

```bash
sf project deploy validate \
  --source-dir force-app \
  --target-org qa \
  --test-level RunLocalTests \
  --junit --coverage-formatters cobertura \
  --results-dir test-results \
  --wait 30
```

|Part|What it does|
|---|---|
|`validate`|Sends everything to QA, compiles it, runs the tests, then **rolls it all back**. Nothing changes in QA. This is a **check-only deploy**|
|`--source-dir force-app`|Deploys the whole project. QA and PROD have no source tracking, so pipelines always deploy from a path or a manifest|
|`--test-level RunLocalTests`|Runs every test not from a managed package. The safest level, and what production uses by default|
|`--junit --coverage-formatters cobertura`|Produces the two file formats Azure DevOps reads to show test results and coverage|

### It failed

```
 ✔ Deploying Metadata 9.03s
   ▸ Components: 12/12 (100%)
 ✘ Running Tests 2.37s
   ▸ Failed: 4/4 (100%)

Test Failures [4]
• ExpenseServiceTest.doesNotFlagAtThreshold
  message: System.DmlException: Operation failed due to fields being inaccessible on Sobject Expense__c
• ExpenseServiceTest.flagsExpenseOverThreshold
  message: System.DmlException: Operation failed due to fields being inaccessible on Sobject Expense__c
• ExpenseServiceTest.handlesBulkInsert
  message: System.QueryException: No such column 'Requires_Approval__c' on entity 'Expense__c'.
• ExpenseServiceTest.reflagsOnUpdate
  message: System.DmlException: Operation failed due to fields being inaccessible on Sobject Expense__c
```

The deploy itself worked (12 of 12 components). All 4 tests failed. The same tests passed 100% in DEV.

### Reading the clues

1. **"fields being inaccessible"** is about **field-level security (FLS)**, the per-field access settings. The fields exist; the test user just could not see them.
2. **"No such column 'Requires_Approval__c'"** is the same problem in a query. When a user cannot see a field, a query that respects field access treats it as if it does not exist.
3. **Which test got furthest?** `handlesBulkInsert` got past its insert and failed at the query. The other three failed on the insert. The difference: the bulk test only sets `Amount__c`, the others also set `Category__c`. `Amount__c` is required, and required fields are always accessible. The other fields are not.

### The root cause

When I created the fields in the scratch org UI, Salesforce automatically gave the **Admin profile** access to them. Those were exactly the profile files I **deleted** in Part 3. In QA, the fields arrived with no profile granting access, and my permission set existed but was not assigned to anyone.

**Why it passed in DEV:** the scratch org had access settings that QA did not. **That is an environment parity failure**, caught by a validation instead of by a failed production deploy.

> [!note] Something I could not fully explain Apex normally runs in **system mode** and ignores field-level security, so I did not expect FLS to block test DML. These newer orgs seem to enforce it. I did not track down which org setting or release update causes it. The fix below is the right one either way.

### The fix: tests run with the permission set

I did not add the profiles back. I made the tests assign `Expense_Access` to the running user:

```apex
private static User userWithExpenseAccess() {
    Id psId = [SELECT Id FROM PermissionSet WHERE Name = 'Expense_Access'].Id;
    Id me = UserInfo.getUserId();
    if ([SELECT COUNT() FROM PermissionSetAssignment
         WHERE AssigneeId = :me AND PermissionSetId = :psId] == 0) {
        insert new PermissionSetAssignment(AssigneeId = me, PermissionSetId = psId);
    }
    return new User(Id = me);
}

@isTest
static void flagsExpenseOverThreshold() {
    System.runAs(userWithExpenseAccess()) {
        Expense__c exp = new Expense__c(Amount__c = 600, Category__c = 'Travel');
        Test.startTest();
        insert exp;
        Test.stopTest();
        exp = [SELECT Requires_Approval__c FROM Expense__c WHERE Id = :exp.Id];
        Assert.isTrue(exp.Requires_Approval__c, 'Expenses over 500 should require approval');
    }
}
```

- The assignment only exists **inside the test**. Test data is rolled back, so nothing sticks in the org.
- The `COUNT()` check avoids a duplicate-assignment error in an org where it is already assigned.
- It reuses my own user instead of creating one, because Developer Edition orgs have very few user licenses.
- **`System.runAs`** starts a new user context, so the assignment takes effect.
- **A side benefit:** the tests now also prove the permission set grants what the feature needs. If someone removes a field from it later, tests fail.

### Validation, take two

```
 ✔ Deploying Metadata 5.78s
   ▸ Components: 12/12 (100%)
 ✔ Running Tests 4.41s
   ▸ Successful: 4/4 (100%)
 Status: Succeeded

Successfully validated the deployment (0Afbm00000lb3ogCAA).
Run "sf project deploy quick --job-id 0Afbm00000lb3ogCAA" to execute this deploy
```

```bash
git commit -am "fix(test): run tests with Expense_Access permission set so they pass without profile FLS"
git push
```

**Proof that validation changes nothing:**

```bash
sf org list metadata --metadata-type CustomObject --target-org qa --json | grep -c '"fullName": "Expense__c"'
# 0
```

![[Pasted image 20261028130011.png]] _QA > Setup > Deployment Status. Salesforce keeps its own record of every deploy and validation, separate from the pipeline._

### Quick deploy

The output offered a **quick deploy** ID. A successful validation can be deployed for real within **10 days, without running the tests again**:

```bash
sf project deploy quick --job-id <ID> --target-org qa
```

I did not run it, because I wanted the first real QA deploy to come from the pipeline. But this is how enterprise teams keep release windows short: **validate before the window, quick deploy during it.**

---

## What I learned

- **Parity breaks in places you do not see.** The failure was not in my code; it was in access settings that existed in DEV and not in QA.
- **Validate before you automate.** I debugged a Salesforce problem by hand, not a Salesforce problem plus a YAML problem at the same time.
- **`set -euo pipefail`**, always, in pipeline scripts.
- **Check-only deploys** prove a release will work in the target without changing it, and **quick deploy** turns that proof into a short release window.

Next: [[05-headless-auth-jwt|Part 5: Headless auth with JWT]]