---
tags:
  - "#vault"
  - SecretsManagement
---
## Overview

One of the most critical challenges in modern cloud infrastructure is **secrets management**, safely storing and controlling access to sensitive data like API keys, passwords, and certificates. In this lab, I set up **HashiCorp Vault** backed by **Consul** as a storage backend on an Ubuntu 18.04 AWS EC2 instance. This is a foundational skill for any DevOps or cloud security role.

---

## Why Consul + Vault?

Vault needs somewhere to persist its encrypted data. While it supports many storage backends (S3, PostgreSQL, filesystem, etc.), **Consul** is one of the most popular choices because it offers:

- **High availability** - Consul is a distributed key-value store, meaning Vault can run in HA mode
- **Health checking** - Consul can monitor the health of Vault nodes
- **Native integration** - HashiCorp built both tools to work together seamlessly

Running them together is a common production pattern, so this lab simulates a real-world deployment.

---

## Environment

|Detail|Value|
|---|---|
|OS|Ubuntu 18.04.6 LTS (AWS EC2)|
|Private IP|10.0.1.240|
|Consul Version|1.7.3|
|Vault Version|1.5.0|

---

## Part 1 - Setting Up Consul

### Installation

I downloaded the Consul binary directly from HashiCorp's release server, unzipped it, and moved it to `/usr/bin` so it's available system-wide.

```bash
wget https://releases.hashicorp.com/consul/1.7.3/consul_1.7.3_linux_amd64.zip
sudo apt install unzip -y
unzip consul_1.7.3_linux_amd64.zip
sudo mv consul /usr/bin
```

Running `consul -v` confirmed the installation:

```bash
cloud_user@ip-10-0-1-240:~$ consul -v
Consul v1.7.3
Protocol 2 spoken by default, understands 2 to 3 (agent will automatically use protocol >2 when speaking to compatible agents)
```

### Configuring Consul as a systemd Service

Rather than running Consul manually, I configured it as a **systemd service** so it starts automatically on boot and can be managed like any other system service.

I created `/etc/systemd/system/consul.service` with the following key parameters:

- `-server` - runs Consul in server mode (not just as an agent)
- `-ui` - enables the built-in web UI
- `-bootstrap-expect=1` - tells Consul to expect a single-node cluster (suitable for this lab)
- `-bind=10.0.1.240` - binds Consul to the instance's private IP
- `-config-dir=/etc/consul.d/` - points to the configuration directory

I also created `/etc/consul.d/ui.json` to expose the UI on all interfaces (`0.0.0.0`), making it accessible via browser.

```json
{
  "addresses": {
   "http": "0.0.0.0"
   }
}
```
### Starting and Verifying Consul

```bash
sudo systemctl daemon-reload
sudo systemctl start consul
sudo systemctl enable consul   # persist across reboots
sudo systemctl status consul
```

The status output confirmed Consul was **active (running)** and had successfully elected itself as cluster leader, expected behavior for a single-node setup:

```
[INFO]  agent.server.raft: election won: tally=1
[INFO]  agent.server.raft: entering leader state
[INFO]  agent.server: cluster leadership acquired
[INFO]  agent.server: New leader elected: payload=vault
```

```bash
cloud_user@ip-10-0-1-240:~$ sudo systemctl status consul
● consul.service - Consul
   Loaded: loaded (/etc/systemd/system/consul.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2026-04-10 01:01:45 UTC; 1min 52s ago
     Docs: https://www.consul.io/
 Main PID: 4479 (consul)
    Tasks: 8 (limit: 1134)
   CGroup: /system.slice/consul.service
           └─4479 /usr/bin/consul agent -server -ui -data-dir=/temp/consul -bootstrap-expect=1 -node=vault -bind=10.0.1.240 -config-dir=/etc/consul.d/

Apr 10 01:01:56 ip-10-0-1-240 consul[4479]:     2026-04-10T01:01:56.110Z [WARN]  agent.server.raft: heartbeat timeout reached, starting election: last-leader=
Apr 10 01:01:56 ip-10-0-1-240 consul[4479]:     2026-04-10T01:01:56.111Z [INFO]  agent.server.raft: entering candidate state: node="Node at 10.0.1.240:8300 [Candidate]"
Apr 10 01:01:56 ip-10-0-1-240 consul[4479]:     2026-04-10T01:01:56.115Z [INFO]  agent.server.raft: election won: tally=1
Apr 10 01:01:56 ip-10-0-1-240 consul[4479]:     2026-04-10T01:01:56.115Z [INFO]  agent.server.raft: entering leader state: leader="Node at 10.0.1.240:8300 [Leader]"
Apr 10 01:01:56 ip-10-0-1-240 consul[4479]:     2026-04-10T01:01:56.117Z [INFO]  agent.server: cluster leadership acquired
Apr 10 01:01:56 ip-10-0-1-240 consul[4479]:     2026-04-10T01:01:56.117Z [INFO]  agent.server: New leader elected: payload=vault
Apr 10 01:01:56 ip-10-0-1-240 consul[4479]:     2026-04-10T01:01:56.296Z [INFO]  agent.leader: started routine: routine="CA root pruning"
Apr 10 01:01:56 ip-10-0-1-240 consul[4479]:     2026-04-10T01:01:56.297Z [INFO]  agent.server: member joined, marking health alive: member=vault
Apr 10 01:01:56 ip-10-0-1-240 consul[4479]:     2026-04-10T01:01:56.455Z [INFO]  agent: Synced node info
Apr 10 01:02:08 ip-10-0-1-240 consul[4479]:     2026-04-10T01:02:08.178Z [INFO]  agent: Newer Consul version available: new_version=1.22.6 current_version=1.7.3
lines 1-19/19 (END)
```


---

## Part 2 - Setting Up Vault

### Installation

Same approach as Consul, download, unzip, and move to `/usr/bin`.

```bash
wget https://releases.hashicorp.com/vault/1.5.0/vault_1.5.0_linux_amd64.zip
unzip vault_1.5.0_linux_amd64.zip
sudo mv vault /usr/bin
```

### Configuring Vault

I created the Vault configuration file at `/etc/vault/config.hcl`. The two key sections are:

```hcl
storage "consul" {
    address = "10.0.1.240:8500"
    path    = "vault/"
}

listener "tcp" {
    address     = "0.0.0.0:80"
    tls_disable = 1
}

ui = true
```

- **Storage block** - tells Vault to use the local Consul agent as its backend, storing data under the `vault/` prefix
- **Listener block** - Vault listens on port 80 (TLS disabled for this lab; in production, TLS should always be enabled)
- **`ui = true`** - enables the Vault web UI

I then created `/etc/systemd/system/vault.service` following the same systemd pattern as Consul, pointing `ExecStart` at the config file.

### Starting and Verifying Vault

```bash
sudo systemctl daemon-reload
sudo systemctl start vault
sudo systemctl enable vault
sudo systemctl status vault
```

The status output confirmed Vault was **active (running)** with Consul as the storage backend and HA enabled:

```
Storage: consul (HA available)
Version: Vault v1.5.0
Vault server started!
```

```bash
cloud_user@ip-10-0-1-240:~$ sudo systemctl status vault
● vault.service - Vault
   Loaded: loaded (/etc/systemd/system/vault.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2026-04-10 01:23:58 UTC; 17s ago
     Docs: https://www.vault.io/
 Main PID: 4649 (vault)
    Tasks: 7 (limit: 1134)
   CGroup: /system.slice/vault.service
           └─4649 /usr/bin/vault server -config=/etc/vault/config.hcl

Apr 10 01:23:58 ip-10-0-1-240 vault[4649]:               Go Version: go1.14.4
Apr 10 01:23:58 ip-10-0-1-240 vault[4649]:               Listener 1: tcp (addr: "0.0.0.0:80", cluster address: "0.0.0.0:81", max_request_duration: "1m30s", max_request_size: "33554432", t
Apr 10 01:23:58 ip-10-0-1-240 vault[4649]:                Log Level: info
Apr 10 01:23:58 ip-10-0-1-240 vault[4649]:                    Mlock: supported: true, enabled: true
Apr 10 01:23:58 ip-10-0-1-240 vault[4649]:            Recovery Mode: false
Apr 10 01:23:58 ip-10-0-1-240 vault[4649]:                  Storage: consul (HA available)
Apr 10 01:23:58 ip-10-0-1-240 vault[4649]:                  Version: Vault v1.5.0
Apr 10 01:23:58 ip-10-0-1-240 vault[4649]: ==> Vault server started! Log data will stream in below:
Apr 10 01:23:58 ip-10-0-1-240 vault[4649]: 2026-04-10T01:23:58.153Z [INFO]  proxy environment: http_proxy= https_proxy= no_proxy=
Apr 10 01:23:58 ip-10-0-1-240 vault[4649]: 2026-04-10T01:23:58.157Z [WARN]  no `api_addr` value specified in config or in VAULT_API_ADDR; falling back to detection if possible, but this v
lines 1-19/19 (END)
```

---

## Part 3 - Initializing and Unsealing the Vault

### Setting the Vault Address

Before interacting with Vault via CLI, I needed to tell it where the server lives. I used `dig` to get the public FQDN of the EC2 instance:

```bash
dig -x 44.210.116.225
# → ec2-44-210-116-225.compute-1.amazonaws.com
```

Then I exported the address for the current session and persisted it in `.bashrc`:

```bash
export VAULT_ADDR="http://ec2-44-210-116-225.compute-1.amazonaws.com"
echo "export VAULT_ADDR=http://ec2-44-210-116-225.compute-1.amazonaws.com" >> ~/.bashrc
```

> [!NOTE]
> The `VAULT_ADDR` environment variable tells the **Vault CLI where to send its commands**.
> 
> When you run something like `vault operator unseal` or `vault login`, the CLI needs to know which server to talk to over HTTP. Without it, the CLI doesn't know where to direct its requests.
> 
> We could use **localhost** `http://127.0.0.1` or **private IP** `http://10.0.1.240` for local CLI use, and it would work fine. Using the public FQDN is just more realistic, it's the same address an external application or user would use to reach Vault, so it validates that the service is reachable from the outside as well.

<!--
In this lab specifically:
- Vault is listening on **port 80** of the EC2 instance
- The CLI is running **on the same machine**, but it still communicates with Vault over HTTP like any client would
- So you point it at the public FQDN: `http://ec2-44-210-116-225.compute-1.amazonaws.com`

[[#Configuring Vault]]
-->

### Initializing Vault

Running `vault operator init` is a **one-time operation** that bootstraps the Vault. It generates:

- **5 Unseal Keys** - used to reconstruct the master key
- **1 Initial Root Token** - used for initial authentication

```
cloud_user@ip-10-0-1-240:~$ vault operator init
Unseal Key 1:MpqLbT/dhFjFrpeTn65KQPZPUXw8AxYseUkcQu9sNitq
Unseal Key 2: 4fGHhFO7fuCWX/cFwhu8uf/PGG+D+d8Jm3iKVTmc1jFy
Unseal Key 3: D25ufHi2jLMPj5SISF1qS+J+Cu/Dk/ccfwtCfpWUslkX
Unseal Key 4: rNLXA3FzIRUN1PCoYx/5o/jnD5eMAf3aqQw3VBijwmud
Unseal Key 5: kSYyEjYYKf7H9fOztOzZqOzWVARzkRzqr05paGJbRhng

Initial Root Token: s.rE3Wkkx71PQDtvUkPMdta0i7

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 3 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```

 > [!attention] Important 👀 
 > 
> These keys must be stored securely and separately. Vault does **not** store the master key, if you lose enough key shares, access to the Vault is permanently lost.

### Unsealing Vault

By default, Vault starts in a **sealed** state, it holds encrypted data but cannot access it until the master key is reconstructed. This requires providing **3 out of 5 keys** (the threshold), a security model known as [Shamir's Secret Sharing](https://en.wikipedia.org/wiki/Shamir%27s_secret_sharing).

I ran `vault operator unseal` three times, providing a different key each time. After the third key, the `Sealed` status flipped to `false`:

```
Sealed             false
HA Mode            standby
```

```bash
cloud_user@ip-10-0-1-240:~$ vault operator unseal
Unseal Key (will be hidden):
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    1/3
Unseal Nonce       e0cbb9a6-fd18-d55c-fe04-552efb89056f
Version            1.5.0
HA Enabled         true
cloud_user@ip-10-0-1-240:~$ vault operator unseal
Unseal Key (will be hidden):
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    2/3
Unseal Nonce       e0cbb9a6-fd18-d55c-fe04-552efb89056f
Version            1.5.0
HA Enabled         true
cloud_user@ip-10-0-1-240:~$ vault operator unseal
Unseal Key (will be hidden):
Key                    Value
---                    -----
Seal Type              shamir
Initialized            true
Sealed                 false
Total Shares           5
Threshold              3
Version                1.5.0
Cluster Name           vault-cluster-7f4aa55f
Cluster ID             18aeefac-e31f-6d67-7f02-f49f02fbccbd
HA Enabled             true
HA Cluster             n/a     #<<<
HA Mode                standby #<<<
Active Node Address    <none>
```
### Logging In

```bash
vault login s.rE3Wkkx71PQDtvUkPMdta0i7
```

```
Success! You are now authenticated.
token_duration       ∞
token_policies       ["root"]
```

✅ Vault is now initialized, unsealed, and ready to serve secrets.

Logging in with the root token gives us administrative access to perform tasks such as creating secret engines, defining policies, and setting up auth methods. That said, the root token is not meant for everyday use, once Vault is unsealed, applications can interact with it using their own scoped tokens and permissions without any root involvement.

---

## Key Takeaways

- **Consul as a Vault backend** is a production-ready pattern that enables high availability and native service discovery.
- **systemd integration** ensures both services are resilient, they'll restart automatically on failure or reboot.
- **Vault's seal/unseal mechanism** is a deliberate security design, it protects data at rest even if the underlying storage is compromised.
- **The root token should be tightly controlled** and ideally revoked once admin policies are established. In production, you'd create narrowly-scoped tokens and policies instead of operating as root.

---

## What's Next

[[Writing and Testing HashiCorp Vault Policies]]

<!--
With Vault up and running, the natural next steps would be:

- Enabling secrets engines (KV, AWS, database credentials)
- Creating access policies and non-root tokens
- Configuring audit logging
- Enabling TLS on the Vault listener
- Setting up Vault agent for application-level secret injection

-->

