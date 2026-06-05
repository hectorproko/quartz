---

tags:
  - security
  - RHEL10
  - RHEL
  - keylime
linkedin: "False"
quartz: "False"
hardlinked: "True"
---
**Role:** Linux Administrator at a financial services provider  
**Platform:** Red Hat Enterprise Linux 10  
**Environment:** Two RHEL 10 VMs on an isolated internal network,  no internet access

---

## Overview

[[Implementing Post‑Quantum Crypto Policies on RHEL 10|Crypto policies]] and SELinux protect a system while it's running, but neither answers a fundamental question: *has this system been tampered with since it last booted?* That's the problem integrity attestation solves.

[[Keylime]] is an open-source, TPM-based remote attestation system that answers that question continuously. The Workload Node runs a Keylime **agent** that uses a [[TPM (Trusted Platform Module)]] to produce cryptographically signed quotes of its current system state. The Management Node runs a Keylime **verifier** that periodically requests those quotes and validates them against a known-good baseline. If the system state drifts,  a critical binary changes, a bootloader is modified,  the verifier detects it.

In this lab there's no physical TPM available on the cloud VMs, so I used `swtpm` (software TPM) on the Workload Node to simulate one. In a production deployment, hardware TPM is required for genuine security; the software TPM here is a stand-in for learning the workflow.

---

## Environment

| Node            | Role                                                           | Private IP   |
| --------------- | -------------------------------------------------------------- | ------------ |
| Management Node | Keylime verifier, registrar, tenant                            | `10.0.0.63`  |
| Workload Node   | Keylime agent + [[swtpm (software TPM)\|software TPM (swtpm)]] | `10.0.1.224` |

> **Note:** This lab used a fresh pair of VMs. The IPs differ from the earlier sections because the original machines timed out.

---

## Architecture

Before diving in, here's how the Keylime components relate:

```
┌─────────────────────────────────┐    ┌─────────────────────────────────┐
│         Management Node          │    │          Workload Node           │
│                                  │    │                                  │
│  ┌────────────┐                  │    │  ┌─────────────────────────┐    │
│  │  Verifier  │◄─── TPM quotes ──┼────┼──│   Keylime Agent         │    │
│  │  (:8881)   │                  │    │  │   (:9002)               │    │
│  └────────────┘                  │    │  └──────────┬──────────────┘    │
│                                  │    │             │                    │
│  ┌────────────┐                  │    │  ┌──────────▼──────────────┐    │
│  │ Registrar  │◄─ registration ──┼────┼──│   Software TPM (swtpm)  │    │
│  │(:8890/8891)│                  │    │  │   (:2321/:2322)         │    │
│  └────────────┘                  │    │  └─────────────────────────┘    │
│                                  │    │                                  │
│  ┌────────────┐                  │    └─────────────────────────────────┘
│  │   Tenant   │ (CLI tool)       │
│  └────────────┘                  │
└─────────────────────────────────┘
```

- The **registrar** is a database of known agents and their TPM keys.
- The **verifier** continuously polls registered agents for TPM quotes and validates them.
- The **tenant** is the CLI used to provision agents into the verifier and query their status.
- The **agent** runs on each monitored system and responds to quote requests using the TPM.

---

## Part 1 - Management Node: Verifier and Registrar Setup

### Verify Packages

```bash
rpm -q keylime-verifier keylime-tenant keylime-registrar
```

**Output:**
```
keylime-verifier-7.12.1-11.el10_1.4.x86_64
keylime-tenant-7.12.1-11.el10_1.4.x86_64
keylime-registrar-7.12.1-11.el10_1.4.x86_64
```

### Write Drop-In Configuration Files

Keylime uses a drop-in config system under `/etc/keylime/`. Each service has its own `*.conf.d/` directory where you can layer configuration snippets without modifying the base config file. This is the right pattern,  base files stay clean and your customizations are clearly separated.

I needed to configure each service to:
- Listen on all interfaces (so the **Workload Node** can reach them)
- Limit worker processes (reduces shared-memory conflicts on small VMs)
- Disable EK certificate verification (required when using a software TPM, which has no real EK cert in NVRAM)

```bash
# Verifier - listen on all interfaces
sudo tee /etc/keylime/verifier.conf.d/00-listen.conf >/dev/null <<'EOF'
[verifier]
ip = "0.0.0.0"
EOF

# Registrar - listen on all interfaces
sudo tee /etc/keylime/registrar.conf.d/00-listen.conf >/dev/null <<'EOF'
[registrar]
ip = "0.0.0.0"
EOF

# Verifier - limit workers
sudo tee /etc/keylime/verifier.conf.d/10-workers.conf >/dev/null <<'EOF'
[verifier]
num_workers = 1
EOF

# Registrar - limit workers
sudo tee /etc/keylime/registrar.conf.d/10-workers.conf >/dev/null <<'EOF'
[registrar]
num_workers = 1
EOF

# Verifier - disable EK cert check (software TPM has no real EK cert)
sudo tee /etc/keylime/verifier.conf.d/20-no-ek-check.conf >/dev/null <<'EOF'
[verifier]
require_ek_cert = false
EOF

# Tenant - disable EK cert check
sudo tee /etc/keylime/tenant.conf.d/10-no-ek.conf >/dev/null <<'EOF'
[tenant]
require_ek_cert = false
EOF
```
<!--
**`ip = "0.0.0.0"` — Is that too relaxed?**
In a production environment, yes, it would be. Binding to `0.0.0.0` means the verifier and registrar accept connections on _any_ network interface the machine has.
-->
<!--
**`num_workers = 1` — What exactly is a worker, and does it limit agents?**

Your instinct on the verifier is right. A verifier worker is a process whose job is to go out and request TPM quotes from agents. So if you have 5 workers, 5 agents can be actively quoted at the same time in parallel.

For the **registrar**, the workers serve a different purpose — they handle _incoming_ registration requests from agents. So `num_workers = 1` on the registrar means only 1 registration request is processed at a time. It does not cap how many agents can be registered total — it just means if 10 agents tried to register simultaneously, they'd queue up instead of being handled in parallel.

The reason both are set to `1` here is purely about the small VM size. These cloud instances are lightweight, and Keylime's workers use shared memory. Running multiple workers on a small instance causes shared-memory allocation conflicts — which is also why you had to keep cleaning `/tmp/pymp-*` between service starts.
-->
<!--
**Registration is not permanent state.**

The registrar's database is **ephemeral by default**. When the Keylime agent starts up, one of the first things it does is contact the registrar and say "here I am, here's my EK, here's my AK (Attestation Key), here's my UUID." The registrar stores that information in memory/a local database so the verifier knows how to find and challenge that agent.

But if the agent restarts — or in this lab, if `swtpm` restarts — the TPM state can change. A new EK or AK may be generated, so the old registration entry is stale. The agent re-registers to give the registrar its fresh keys.
-->
### Start the Verifier and Registrar

A key operational note: starting the **verifier** creates temporary shared-memory files under `/tmp/pymp-*`. If the **registrar** starts before those are cleaned up, it can fail. The fix is to clean `/tmp/pymp-*` between service starts:

```bash
sudo rm -rf /tmp/pymp-*
sudo systemctl daemon-reload
sudo systemctl enable --now keylime_verifier
```

Waiting for the verifier to initialize, then cleaning temp files before starting the registrar:

```bash
sleep 5
sudo rm -rf /tmp/pymp-*
sudo systemctl enable --now keylime_registrar
```

**Verifier status:**
```
● keylime_verifier.service - The Keylime verifier
     Active: active (running) since Wed 2026-05-13 15:33:18 UTC; 1min 14s ago
   Main PID: 7809 (keylime_verifie)
...
keylime_verifier[7819]: 2026-05-13 15:33:28 - keylime.verifier - INFO - Starting verifier process 0
```

**Registrar status:**
```
● keylime_registrar.service - The Keylime registrar service
     Active: active (running) since Wed 2026-05-13 15:35:13 UTC; 32s ago
   Main PID: 7901 (keylime_registr)
...
keylime_registrar[7901]: 2026-05-13 15:35:14 - keylime.web - INFO - Listening on HTTPS...
```

Confirming all three ports are open:

```bash
ss -tlnp | grep -E '8881|8890|8891'
```

**Output:**
```
LISTEN 0      128          0.0.0.0:8881      0.0.0.0:*
LISTEN 0      128          0.0.0.0:8891      0.0.0.0:*
LISTEN 0      128          0.0.0.0:8890      0.0.0.0:*
```

Verifier (`:8881`) and registrar (`:8890`, `:8891`) are listening.

---

## Part 2 - Workload Node: Software TPM and Agent Setup

### Verify Packages

```bash
rpm -q keylime-agent-rust swtpm swtpm-tools tpm2-tools
```

**Output:**
```
keylime-agent-rust-0.2.7-2.el10.x86_64
swtpm-0.9.0-5.el10.x86_64
swtpm-tools-0.9.0-5.el10.x86_64
tpm2-tools-5.7-4.el10.x86_64
```

### Set SELinux to Permissive

SELinux in enforcing mode blocks the socket communication that `swtpm` and the Keylime agent need. For this lab I set it to permissive on the **Workload Node**:

```bash
sudo setenforce 0
```

> In production, the correct solution is to write an [[Writing a Custom SELinux Policy|SELinux policy]] module that permits the specific interactions needed, rather than going permissive.

### Initialize the Software TPM

```bash
sudo mkdir -p /var/lib/swtpm
sudo chmod 700 /var/lib/swtpm
sudo swtpm_setup --tpm2 --tpmstate /var/lib/swtpm --overwrite
```

**Output:**
```
Starting vTPM manufacturing as root:root @ Wed 13 May 2026 03:40:48 PM UTC
TPM is listening on Unix socket.
Successfully activated PCR banks sha256 among sha1,sha256,sha384,sha512.
Successfully authored TPM state.
Ending vTPM manufacturing @ Wed 13 May 2026 03:40:48 PM UTC
```

Note: I initialized without `--createek`. The Keylime agent creates its own [[EK (Endorsement Key)]], pre-creating one causes handle conflicts that prevent the agent from starting.

Starting the software TPM and verifying it's listening:

```bash
sudo /usr/bin/swtpm socket --tpm2 \
    --tpmstate dir=/var/lib/swtpm \
    --ctrl type=tcp,port=2322 \
    --server type=tcp,port=2321 \
    --flags startup-clear &

sleep 2
ss -tlnp | grep -E '2321|2322'
```

**Output:**
```
LISTEN 0      1          127.0.0.1:2322      0.0.0.0:*
LISTEN 0      1          127.0.0.1:2321      0.0.0.0:*
```

Testing TPM connectivity:

```bash
export TPM2TOOLS_TCTI="swtpm:port=2321"
tpm2_pcrread sha256:0
```

**Output:**
```
  sha256:
    0 : 0x0000000000000000000000000000000000000000000000000000000000000000
```

All-zero PCR values are expected for a fresh software TPM, the Platform Configuration Registers haven't been extended by any measurements yet.

### Configure the Keylime Agent

Agent configuration also uses drop-in files, plus a systemd override to inject the TPM environment variables at service start:
<!--
drop-in files mean Instead of editing one big config file directly, you create a folder and drop small files into it
-->
```bash
sudo mkdir -p /etc/keylime/agent.conf.d
sudo mkdir -p /etc/systemd/system/keylime_agent.service.d

# Fixed UUID for this agent
sudo tee /etc/keylime/agent.conf.d/00-uuid.conf >/dev/null <<'EOF'
[agent]
uuid = "d432fbb3-d2f1-4a97-9ef7-75bd81c00000"
EOF

# Point the agent at the Management Node registrar
sudo tee /etc/keylime/agent.conf.d/10-registrar.conf >/dev/null <<'EOF'
[agent]
registrar_ip = "10.0.0.63"
EOF

# Listen on all interfaces
sudo tee /etc/keylime/agent.conf.d/20-listen.conf >/dev/null <<'EOF'
[agent]
ip = "0.0.0.0"
EOF

# Inject TCTI variables so the agent uses swtpm instead of a hardware TPM
sudo tee /etc/systemd/system/keylime_agent.service.d/10-swtpm.conf >/dev/null <<'EOF'
[Service]
Environment="TCTI=swtpm:port=2321"
Environment="TPM2TOOLS_TCTI=swtpm:port=2321"
Environment="TSS2_TCTI=swtpm:port=2321"
EOF
```


<!--
### Troubleshooting: Agent Failed on First Start

On my first attempt I left the registrar IP as the literal placeholder text `MANAGEMENT_PRIVATE_IP` instead of the actual IP, the agent failed immediately with `status=1/FAILURE`. After correcting the config to `10.0.0.63` and reloading:

```bash
sudo tee /etc/keylime/agent.conf.d/10-registrar.conf >/dev/null <<'EOF'
[agent]
registrar_ip = "10.0.0.63"
EOF

sudo systemctl daemon-reload
sudo systemctl start keylime_agent
sudo systemctl --no-pager status keylime_agent
```

**Output:**
```
● keylime_agent.service - The Keylime compute agent
     Active: active (running) since Wed 2026-05-13 15:48:55 UTC; 32s ago
...
keylime_agent[6403]:  INFO  keylime_agent              > Agent UUID: d432fbb3-d2f1-4a97-9ef7-75bd81c00000
keylime_agent[6403]:  INFO  keylime::registrar_client  > Requesting agent registration
keylime_agent[6403]:  INFO  keylime_agent              > SUCCESS: Agent d432fbb3... registered
keylime_agent[6403]:  INFO  keylime::registrar_client  > Requesting agent activation
keylime_agent[6403]:  INFO  keylime_agent              > SUCCESS: Agent d432fbb3... activated
keylime_agent[6403]:  INFO  keylime_agent              > Listening on https://0.0.0.0:9002
```

The agent registered with the Management Node's registrar and activated successfully.

Confirming port `:9002` is open:

```bash
ss -tlnp | grep 9002
```
```
LISTEN 0      1024         0.0.0.0:9002      0.0.0.0:*
```
-->
---

## Part 3 - Trust Establishment: Copy the CA Certificate

Keylime uses [[Mutual TLS (two-way)|mutual TLS (mTLS)]] between the verifier and the agent. The verifier generates its own CA when it first starts; the agent needs to trust that CA to accept connections from the verifier.

From the **Management Node**, I copied the verifier's [[CA certificate]] to the **Workload Node**:
<!--
the Keylime verifier generates its own self-signed CA on first startup
**Self-signed** means the CA signed its own certificate — there's no higher authority vouching for it.
The verifier acts as its own CA
[[Self-signed  CA certificate]]
-->

```bash
sudo scp /var/lib/keylime/cv_ca/cacert.crt cloud_user@10.0.1.224:/tmp/
```
```
cacert.crt                         100% 1432     1.0MB/s   00:00
```

On the **Workload Node**, I placed the cert where the agent expects it:

```bash
sudo mkdir -p /var/lib/keylime/cv_ca
sudo cp /tmp/cacert.crt /var/lib/keylime/cv_ca/
sudo chown -R keylime:keylime /var/lib/keylime/cv_ca
sudo systemctl restart keylime_agent
```

After restarting, the agent re-registered and re-activated cleanly:
```
INFO  keylime_agent  > SUCCESS: Agent d432fbb3... registered
INFO  keylime_agent  > SUCCESS: Agent d432fbb3... activated
INFO  keylime_agent  > Listening on https://0.0.0.0:9002
```

---

## Part 4 - Provision and Observe Live Attestation

Back on the **Management Node**, I first verified the agent appeared in the registrar:

```bash
sudo keylime_tenant -c reglist
```

**Output:**
```
Agent list from Registrar (127.0.0.1:8891) retrieved:
{"uuids": ["d432fbb3-d2f1-4a97-9ef7-75bd81c00000"]}
```

The agent is registered. Now I provisioned it into the verifier for active attestation:

```bash
sudo rm -rf /tmp/pymp-*
sudo systemctl restart keylime_verifier
sleep 5

sudo keylime_tenant --command add \
    --targethost 10.0.1.224 \
    --uuid d432fbb3-d2f1-4a97-9ef7-75bd81c00000 \
    --cert default
```

**Key lines from the output:**
```
keylime.tenant - WARNING - DANGER: EK cert checking is disabled... (expected for swtpm)
keylime.tenant - INFO - Quote from Agent d432fbb3... (10.0.1.224:9002) validated
keylime.tenant - INFO - Agent d432fbb3... (10.0.1.224:9002) added to Verifier (127.0.0.1:8881) after 0 tries
```

The verifier validated the TPM quote from the agent and added it to active monitoring.

### Watching Attestation in Action

Checking the [[attestation]] status immediately after provisioning:

```bash
sudo keylime_tenant -c cvstatus --uuid d432fbb3-d2f1-4a97-9ef7-75bd81c00000
```

**Relevant fields from the JSON output:**
```json
{
  "operational_state": "Get Quote",
  "hash_alg": "sha256",
  "enc_alg": "rsa",
  "sign_alg": "rsassa",
  "attestation_count": 74,
  "last_received_quote": 1778688402,
  "last_successful_attestation": 1778688402
}
```

^11c119

After waiting 10 seconds and rechecking:

```bash
sleep 10
sudo keylime_tenant -c cvstatus --uuid d432fbb3-d2f1-4a97-9ef7-75bd81c00000
```

```json
{
  "operational_state": "Get Quote",
  "attestation_count": 123,
  ...
}
```

The `attestation_count` increased from 74 to 123 in 10 seconds, roughly two successful attestations per second. `operational_state: "Get Quote"` means the verifier is actively polling the agent and all quotes are passing validation.
<!--
The count of **74** came from the **first** `cvstatus` check you ran right after provisioning the agent: 
[[#^11c119]]
-->

Finally, confirming in the verifier logs:

```bash
sudo journalctl -u keylime_verifier -n 20 --no-pager
```

**Key log entry:**
```
keylime_verifier[8136]: 2026-05-13 16:04:11 - keylime.verifier - INFO - POST returning 200 response for adding agent id: d432fbb3-d2f1-4a97-9ef7-75bd81c00000
```

The `200` response confirms the verifier accepted the agent and attestation is running cleanly.

---

## Key Takeaways

- Keylime provides continuous remote integrity attestation using TPM quotes, it doesn't just check a system once, it polls it repeatedly, so any change to the measured state is detected quickly.
- The **registrar** is a directory of known agents. The **verifier** is what actually does continuous monitoring. An agent must register before the verifier can poll it.
- Use drop-in config directories (`/etc/keylime/verifier.conf.d/`, etc.) for all customizations. Never edit the base config files directly, drop-ins survive package updates cleanly.
- Clean `/tmp/pymp-*` between starting the verifier and the registrar to avoid shared-memory conflicts.
- Do not pre-create the EK during `swtpm_setup`. Let the Keylime agent create its own EK to avoid TPM handle conflicts that prevent startup.
- The `TCTI` environment variables (`TCTI`, `TPM2TOOLS_TCTI`, `TSS2_TCTI`) must all point to the software TPM socket, injecting them via a systemd drop-in is the cleanest way to ensure the agent always finds the right TPM interface.
- A rising `attestation_count` with `operational_state: "Get Quote"` is the confirmation that the full pipeline, agent → registrar → verifier → tenant, is working end to end.
