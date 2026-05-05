---
tags:
  - vault
  - SecretsManagement
---

<!--
From Draft [[Lab Write and Test a HashiCorp Vault Policy]]
-->
## Overview

One of the core responsibilities in securing cloud infrastructure is controlling _who can access what_ and _how_. In this hands-on, I practiced writing, assigning, and validating access policies in **HashiCorp Vault**, an industry-standard secrets management tool. The goal was to demonstrate how fine-grained, path-based permissions work in practice by creating two opposing policies and confirming their behavior with dedicated tokens.

---

## What is HashiCorp Vault?

HashiCorp Vault is an open-source tool designed to securely store and tightly control access to sensitive data such as API keys, passwords, certificates, and tokens. Rather than embedding secrets directly into applications, teams use Vault to centralize secrets management with strong access controls and audit logging.

Vault uses a **policy-based access model**: every token or identity is bound to one or more policies, and those policies define exactly which paths (secret locations) a token can read from, write to, or manage.

---

## Environment

I'm using an AWS EC2 instance (Ubuntu) with [[Installing and Configuring HashiCorp Vault with Consul on Ubuntu|Vault pre-installed]]. The instance was accessible over SSH, and Vault was initialized with a **Shamir secret sharing** setup, meaning the vault required a minimum of **3 out of 5 unseal keys** to become operational after a restart. This is a common production pattern that prevents any single administrator from having full unilateral access to secrets.

|Detail|Value|
|---|---|
|Server OS|Ubuntu (AWS EC2)|
|Vault Version|1.5.0|
|Unseal Threshold|3 of 5 keys|
|Auth Method|Token-based|

---

## Step 1: Unsealing Vault and Authenticating

Before Vault can serve any requests, it must be **unsealed**. I provided 3 of the 5 unseal keys using the `vault operator unseal` command, then authenticated using the root token to gain full administrative access.

```bash
vault operator unseal 
<KEY_1>
vault operator unseal 
<KEY_2>
vault operator unseal 
<KEY_3>

vault login 
<ROOT_TOKEN>
```

> **Why unseal?** Vault starts in a sealed state for security. Even if an attacker gains access to the server, they cannot read any secrets until the vault is explicitly unsealed with the required key shares.

---

## Step 2: Enabling Two KV Secret Engines

I enabled the **KV (key-value) secrets engine** at two separate paths: `secrets-kv-X` and `secrets-kv-Y`. Think of these as two separate vaults within the system,  each holding its own collection of secrets and governed by its own set of permissions.

```bash
vault secrets enable -path=secrets-kv-X kv
vault secrets enable -path=secrets-kv-Y kv
```

> **Why two paths?** The purpose here was to simulate a realistic scenario where different teams or services each have their own secret storage,  and access should be compartmentalized between them.

---

## Step 3: Writing the Policies

I created two HCL policy files that grant **opposite permissions** on the two paths.

### Policy `XY.hcl` - Write to X, Read from Y

```hcl
path "secrets-kv-X/*" {
  capabilities = ["create"]
}

path "secrets-kv-Y/*" {
  capabilities = ["read"]
}
```

A token with policy **XY** can only _write_ secrets to `secrets-kv-X` and only _read_ secrets from `secrets-kv-Y`. Any other action on either path will be denied.

### Policy `YX.hcl` - Write to Y, Read from X

```hcl
path "secrets-kv-Y/*" {
  capabilities = ["create"]
}

path "secrets-kv-X/*" {
  capabilities = ["read"]
}
```

A token with policy **YX** is the mirror image,  it can only _write_ to `secrets-kv-Y` and only _read_ from `secrets-kv-X`.

Both policies were **uploaded** to Vault with:

```bash
vault policy write XY XY.hcl
vault policy write YX YX.hcl
```

> **Why write policies as code?** Storing policies as `.hcl` files allows them to be version-controlled, reviewed, and audited,  an important practice for security compliance and team collaboration.

---

## Step 4: Creating Tokens and Testing Access

I generated two tokens, each bound to one policy, and tested all four possible operations to verify the policies behaved exactly as intended.

```bash
vault token create -policy=XY -format=json | jq
vault token create -policy=YX -format=json | jq
```

### Testing Token XY

|Action|Path|Expected|Result|
|---|---|---|---|
|`vault kv put`|`secrets-kv-X`|✅ Allowed|Success|
|`vault kv put`|`secrets-kv-Y`|❌ Denied|403 Permission Denied|
|`vault kv get`|`secrets-kv-X`|❌ Denied|403 Permission Denied|
|`vault kv get`|`secrets-kv-Y`|✅ Allowed|Success (no value yet)|
#### Testing Token XY (`s.FY8qmFVDB3fra0DcD69NxIeq`)

```bash
# Login with XY token
vault login
# Token: s.FY8qmFVDB3fra0DcD69NxIeq

# PUT to secrets-kv-X → ALLOWED
vault kv put secrets-kv-X/my-kv-secret username=password
Success! Data written to: secrets-kv-X/my-kv-secret

# PUT to secrets-kv-Y → DENIED
vault kv put secrets-kv-Y/my-kv-secret username=password
Error writing data to secrets-kv-Y/my-kv-secret: Error making API request.
URL: PUT http://ec2-3-235-196-12.compute-1.amazonaws.com/v1/secrets-kv-Y/my-kv-secret
Code: 403. Errors:
* 1 error occurred:
        * permission denied

# GET from secrets-kv-X → DENIED
vault kv get secrets-kv-X/my-kv-secret
Error reading secrets-kv-X/my-kv-secret: Error making API request.
URL: GET http://ec2-3-235-196-12.compute-1.amazonaws.com/v1/secrets-kv-X/my-kv-secret
Code: 403. Errors:
* 1 error occurred:
        * permission denied

# GET from secrets-kv-Y → ALLOWED (no value written yet)
vault kv get secrets-kv-Y/my-kv-secret
No value found at secrets-kv-Y/my-kv-secret
```

### Testing Token YX

|Action|Path|Expected|Result|
|---|---|---|---|
|`vault kv put`|`secrets-kv-X`|❌ Denied|403 Permission Denied|
|`vault kv put`|`secrets-kv-Y`|✅ Allowed|Success|
|`vault kv get`|`secrets-kv-X`|✅ Allowed|Returns `username=password`|
|`vault kv get`|`secrets-kv-Y`|❌ Denied|403 Permission Denied|

#### Testing Token YX (`s.yXjfxiDvBIn1IsOYS6Xpk4R2`)

```bash
# Login with YX token
vault login
# Token: s.yXjfxiDvBIn1IsOYS6Xpk4R2

# PUT to secrets-kv-X → DENIED
vault kv put secrets-kv-X/my-kv-secret username=password
Error writing data to secrets-kv-X/my-kv-secret: Error making API request.
URL: PUT http://ec2-3-235-196-12.compute-1.amazonaws.com/v1/secrets-kv-X/my-kv-secret
Code: 403. Errors:
* 1 error occurred:
        * permission denied

# PUT to secrets-kv-Y → ALLOWED
vault kv put secrets-kv-Y/my-kv-secret username=password
Success! Data written to: secrets-kv-Y/my-kv-secret

# GET from secrets-kv-X → ALLOWED
vault kv get secrets-kv-X/my-kv-secret
====== Data ======
Key         Value
---         -----
username    password

# GET from secrets-kv-Y → DENIED
vault kv get secrets-kv-Y/my-kv-secret
Error reading secrets-kv-Y/my-kv-secret: Error making API request.
URL: GET http://ec2-3-235-196-12.compute-1.amazonaws.com/v1/secrets-kv-Y/my-kv-secret
Code: 403. Errors:
* 1 error occurred:
        * permission denied
```

Every test produced the expected result, confirming that Vault's policy engine enforces access precisely at the path level.

---

## Key Takeaways

**Least privilege in action.** Each token could only do exactly what its policy defined,  nothing more. This is the principle of least privilege applied directly to secrets management.

**Path-based access control is powerful.** Vault treats secret paths like file system paths, allowing you to grant or restrict access at any level of granularity ,  from an entire secrets engine down to a specific named secret.

**Policies are declarative and auditable.** Writing policies as `.hcl` files makes access control transparent, reviewable, and easy to reproduce a must-have in any production or compliance-driven environment.

**Unsealing is a deliberate security gate.** The Shamir unseal process ensures no single person can activate Vault alone, which is an important safeguard for high-security environments.

---

## Skills Demonstrated

- HashiCorp Vault administration (unseal, login, secrets engine management)
- Writing and uploading HCL access policies
- Token-based authentication and authorization
- Validating access controls through hands-on testing
- Secrets management fundamentals (KV engine, path-based permissions)