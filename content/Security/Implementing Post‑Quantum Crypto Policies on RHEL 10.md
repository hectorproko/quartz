---
tags:
  - RHEL10
  - security
  - RHEL
quartz: "True"
linkedin: "False"
hardlinked: "True"
---
**Role:** Linux Administrator at a financial services provider  
**Platform:** Red Hat Enterprise Linux 10  
**Environment:** Two RHEL 10 VMs on an isolated network (no internet access)

---

## Overview

Quantum computing is advancing fast enough that organizations preparing for long-term data security need to start acting now. The threat model is simple: adversaries can harvest encrypted traffic today and decrypt it later once quantum computers become capable. To get ahead of this, RHEL 10 ships with a preview crypto policy module (`TEST-PQ`) that layers post-quantum key exchange and signature algorithms on top of the existing `DEFAULT` policy.

In this lab I enabled that policy on the Management Node, then validated that both SSH and TLS continued to work normally,  because enabling stronger crypto that breaks your admin tooling is not a win.

---

## Environment

| Node | Role | Private IP |
|---|---|---|
| Management Node | Crypto policy changes, TLS server | `10.0.0.64` |
| Workload Node | TLS client, SSH target | `10.0.1.110` |

---

## Step 1 - Verify the Environment

Before changing anything on a production-adjacent system, I confirmed all required packages were present and that SELinux was in enforcing mode. There's no point tuning crypto if the system isn't in a known-good baseline state.

```bash
rpm -q ansible-core rhel-system-roles crypto-policies-scripts \
    policycoreutils-python-utils sshpass \
    keylime-verifier keylime-tenant keylime-registrar
```

**Output:**
```
ansible-core-2.16.14-1.el10.noarch
rhel-system-roles-1.108.6-0.1.el10.noarch
crypto-policies-scripts-20250905-2.gitc7eb7b2.el10_1.1.noarch
policycoreutils-python-utils-3.9-1.el10.noarch
sshpass-1.09-9.el10_0.x86_64
keylime-verifier-7.12.1-11.el10_1.4.x86_64
keylime-tenant-7.12.1-11.el10_1.4.x86_64
keylime-registrar-7.12.1-11.el10_1.4.x86_64
```

I then confirmed the **PQ module** was available and SELinux was enforcing:

```bash
ls /usr/share/crypto-policies/policies/modules/ | grep -i pq
getenforce
```
<!--RHEL (and derivatives like CentOS Stream, Fedora) that's the standard installation path for the `crypto-policies` package.-->
**Output:**
```
NO-PQ.pmod
TEST-PQ.pmod

Enforcing
```

Both checks passed,  `TEST-PQ.pmod` is included in the base `crypto-policies` package, and SELinux is enforcing.

---

## Step 2 - Inspect the Current Crypto Policy

```bash
update-crypto-policies --show
ls -1 /usr/share/crypto-policies/policies/modules | sed 's/\.pmod$//' | head -n 30
```
<!--[[ls command]]
Lists the contents of that directory. The `-1` flag forces one item per line (instead of columns), which makes it easier to process with the next command.
`| sed 's/\.pmod$//'`
Takes each line and strips the `.pmod` file extension. `sed` is a stream editor — it reads line by line and applies a substitution.
-->
**Output:**
```
DEFAULT

AD-SUPPORT-LEGACY
AD-SUPPORT
ECDHE-ONLY
NO-ENFORCE-EMS
NO-PQ
OSPP
TEST-PQ
```

The system was running the `DEFAULT` policy. `TEST-PQ` was listed and ready to apply. Note that `NO-PQ` is also available,  it explicitly *disables* post-quantum algorithms, which is useful in environments where PQ algorithms cause compatibility issues with legacy endpoints.

---

## Step 3 - Enable the Post-Quantum Crypto Profile

The `update-crypto-policies` tool applies **system-wide** cryptographic policy by modifying configuration for [[openssl|OpenSSL]], [[GnuTLS]], [[NSS (Network Security Services)|NSS]], and OpenSSH in one shot. Adding `:TEST-PQ` layers the **PQ module** on top of `DEFAULT` without replacing it.

```bash
sudo update-crypto-policies --set DEFAULT:TEST-PQ
```

**Output (abbreviated):**
```
ExperimentalValueWarning: `group` value `P521-MLKEM1024` is experimental and might go away in the future
ExperimentalValueWarning: `group` value `MLKEM1024` is experimental and might go away in the future
ExperimentalValueWarning: `group` value `X25519-MLKEM512` is experimental and might go away in the future
ExperimentalValueWarning: `sign` value `MLDSA87-P384` is experimental and might go away in the future
ExperimentalValueWarning: `sign` value `MLDSA44-ED25519` is experimental and might go away in the future
...
Setting system policy to DEFAULT:TEST-PQ
Note: System-wide crypto policies are applied on application start-up.
It is recommended to restart the system for the change of policies
to fully take place.
```

The warnings are expected,  [[MLKEM (ML-KEM, formerly Kyber)]] and [[MLDSA (ML-DSA, formerly Dilithium)]] are the NIST-standardized post-quantum algorithms, and RHEL 10 marks them experimental while the ecosystem stabilizes. The policy was set successfully.

Confirming the active policy:

```bash
update-crypto-policies --show
```
```
DEFAULT:TEST-PQ
```

---

## Step 4 - Restart SSH and Verify It Still Works

Policy changes apply at application startup, so the SSH daemon needs a restart to pick them up:

```bash
sudo systemctl restart sshd
sudo systemctl --no-pager status sshd
```
<!--[[systemctl#Options]]-->
**Output:**
```
● sshd.service - OpenSSH server daemon
     Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-05-13 14:20:52 UTC; 14s ago
   Main PID: 7925 (sshd)
...
May 13 14:20:52 ip-10-0-0-64.ec2.internal sshd[7925]: Server listening on 0.0.0.0 port 22.
May 13 14:20:52 ip-10-0-0-64.ec2.internal sshd[7925]: Server listening on :: port 22.
May 13 14:20:52 ip-10-0-0-64.ec2.internal systemd[1]: Started sshd.service - OpenSSH server daemon.
```

SSH is active and listening. Next, I tested an actual connection from the Management Node to the Workload Node to confirm the new policy doesn't break normal admin workflows:

```bash
ssh cloud_user@10.0.1.110 'echo "connection works"'
```
```
connection works
```

SSH works under `DEFAULT:TEST-PQ`. ✅

---

## Step 5 - TLS Handshake Test

SSH uses its own key exchange negotiation, but I also wanted to verify that TLS 1.3 still completed a handshake under the new policy. I created a temporary self-signed certificate and spun up a [[TLS server]] on port `8443`.

### Why I needed `sudo -i` first

My first attempt used `sudo nohup ... &`, which sent the process to the background before `sudo` could prompt for a password,  causing Linux to suspend it. The fix was to elevate to root first, then launch the server:

```bash
sudo -i
nohup openssl s_server -accept 8443 -cert /tmp/lab-tls.crt \
    -key /tmp/lab-tls.key -www -tls1_3 \
    > /tmp/openssl-s_server.log 2>&1 &

ss -tlnp | grep 8443
```

^2e6914
<!--
[[nohup (no hang up)|nohup]], [[openssl]]
-->
**Output:**
```
LISTEN 0      4096               *:8443            *:*    users:(("openssl",pid=8188,fd=3))
```

Server is now listening on the **Management Node** on port `8443`. From the **Workload Node**, I ran a TLS client handshake back to the Management Node:

```bash
openssl s_client -connect 10.0.0.64:8443 -tls1_3 -brief </dev/null
```
<!--
`openssl s_client` just initiates a raw TLS handshake — there's no password involved at all.

The `</dev/null` at the end is just telling it "send no data, just do the handshake and exit immediately." Without it, the command would sit there waiting for you to type something.

No credentials, no authentication — it's the equivalent of knocking on a door to confirm someone answers, not actually going inside. The only thing being tested is whether the two machines can successfully negotiate a TLS 1.3 connection under the new crypto policy.
-->
**Output:**
```
Connecting to 10.0.0.64
Can't use SSL_get_servername
depth=0 CN=mgmt.lab.local
verify error:num=18:self-signed certificate
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Ciphersuite: TLS_AES_256_GCM_SHA384
Peer certificate: CN=mgmt.lab.local
Hash used: SHA256
Signature type: RSA-PSS
Verification error: self-signed certificate
Server Temp Key: X25519, 253 bits
DONE
```

`CONNECTION ESTABLISHED` with `Protocol version: TLSv1.3` confirms TLS works correctly. The self-signed certificate warning is expected since we didn't use a trusted CA. After the test, I stopped the server and confirmed port `8443` was released:

```bash
pkill -f "openssl s_server -accept 8443" || true
ss -tlnp | grep 8443 || echo "TLS server stopped"
```
```
TLS server stopped
```

---

## Key Takeaways

- RHEL 10's `crypto-policies` framework makes it possible to enable post-quantum algorithm support system-wide in a single command, with no per-application tuning.
- The `TEST-PQ` module adds ML-KEM (key exchange) and ML-DSA (signatures) on top of the standard `DEFAULT` policy,  it does not replace classical algorithms, so compatibility is preserved.
- `ExperimentalValueWarning` messages are expected and safe to proceed past; they reflect the evolving standardization status of these algorithms.
- Always restart affected services (like `sshd`) after a policy change and verify connectivity before calling it done.
- Using `sudo nohup ... &` in an interactive terminal requires authentication first,  elevate with `sudo -i` before running background services to avoid job suspension.
