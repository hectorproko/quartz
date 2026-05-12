---
tags:
  - servicemesh
  - Kubernetes
  - istio
---
## Overview 

In this hands-on lab, I implemented a **Zero Trust networking model** inside a Kubernetes cluster using **Istio**, an open-source service mesh. The goal was to prove that Istio can provide enterprise-grade security before migrating production workloads, specifically by encrypting all service-to-service traffic and enforcing identity-based access control.

The environment used Minikube on an AWS EC2 instance. While Minikube is not production-grade, it provided a self-contained Kubernetes cluster sufficient to demonstrate Istio's Zero Trust capabilities end-to-end.

---

## What is Zero Trust?

Zero Trust is a security model built on one core principle: **never trust, always verify**. Unlike traditional perimeter-based security (where everything inside the network is assumed safe), Zero Trust treats every request as potentially hostile, regardless of where it originates. Every service must authenticate itself, and access is only granted based on explicit, verified identity.

In Kubernetes, Istio achieves this through two mechanisms:

- **Mutual TLS (mTLS)** - every service proves its identity with a certificate before a connection is established
- **Authorization Policies** - fine-grained rules that explicitly declare which identities are allowed to talk to which services

---

## Environment Setup

I SSH'd into the lab machine and ran a setup script that launched Minikube with 2 CPUs and 3 GB of memory using the Docker driver.

```bash
#!/bin/bash
echo 'Starting Minikube with sufficient resources for Istio...'
minikube start --driver=docker --cpus=2 --memory=3000 --force
if [ $? -eq 0 ]; then
  echo 'Minikube started successfully!'
  kubectl config use-context minikube
  echo 'Setup complete! You can now proceed with the lab.'
else
  echo 'Error: Minikube failed to start.'
  exit 1
fi
```

**Output:**

```bash
Starting Minikube with sufficient resources for Istio...
* minikube v1.38.1 on Amazon 2023.4.20240429
* Creating docker container (CPUs=2, Memory=3000MB) ...
* Preparing Kubernetes v1.35.1 on Docker 29.2.1 ...
* Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
Minikube started successfully!
Switched to context "minikube".
Setup complete! You can now proceed with the lab.
```

After setup, I confirmed the node was healthy:

```bash
kubectl get nodes
```

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   32s   v1.35.1
```

---

## Part 1 - Deploying the Istio Service Mesh

### Installing Istio

Using `istioctl`, Istio's dedicated bootstrapping CLI that was pre-installed on the machine, I deployed Istio with the `demo` profile, which includes the **control plane** (`istiod`), an ingress gateway for inbound traffic, and an egress gateway for outbound traffic. This profile is well-suited for labs and proof-of-concept deployments; in production you would typically use the `default` profile or a custom-tuned configuration.

```bash
istioctl install --set profile=demo -y
```
<!--
[[Istio#The `--set profile=demo` Flag]]
This is using PKI to the certificae, right now its using self generated ones?
the profile to use in prod is default or a fully custom profile defined in YAML
[[Istio#`istioctl`]]
[[Istio#In Kubernetes Without Istio]] regarding genetic terms ingress, egress gateways
-->

**Output:**

```
✔ Istio core installed ⛵️
✔ Istiod installed 🧠
✔ Egress gateways installed 🛫
✔ Ingress gateways installed 🛬
✔ Installation complete
```

I verified all three Istio components were running in their dedicated namespace:

```bash
kubectl get pods -n istio-system
```

```
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-5bd755b557-mdmph    1/1     Running   0          27s
istio-ingressgateway-6497cf79bc-87l65   1/1     Running   0          27s
istiod-75659b6dd4-cpqkn                 1/1     Running   0          41s
```

^envoyinstancepods

All three pods show `1/1` in the READY column, each component is fully operational.

> [!NOTE]
> The ingress and egress gateway pods are essentially **standalone [[Envoy]] instances**, running as dedicated border guards. Their job is to handle **cluster boundary traffic**.
<!--
[[Istio#How Envoy Shows Up in Your Enforcing Zero Trust Networking in Kubernetes with Istio Service Mesh Lab]]
--> 
---

### Creating the Namespace and Enabling Sidecar Injection

I created a dedicated namespace called `zerotrust` to isolate my workloads. Then I applied the `istio-injection=enabled` label to it.

```bash
kubectl create namespace zerotrust
kubectl label namespace zerotrust istio-injection=enabled
```
<!--[[Labels vs Annotations (kubernetes)#What's actually happening with `istio-injection=enabled`]]
-->
**Why this matters:** This label tells Istio's [[Mutating Admission Webhook|mutating webhook]] to automatically inject an **[[Envoy]] sidecar proxy** into every pod that gets deployed in this namespace. The sidecar is the data plane, it intercepts all inbound and outbound traffic for the pod, enabling mTLS, telemetry, and policy enforcement without any changes to the application code. 
<!--[[Mutating Admission Webhook]]-->
```bash
kubectl get namespace zerotrust --show-labels
```

```
NAME        STATUS   AGE   LABELS
zerotrust   Active   95s   istio-injection=enabled,kubernetes.io/metadata.name=zerotrust
```

---

### Deploying Microservices

I deployed two microservices (`frontend` and `backend`) and a utility `sleep` pod for testing. Each service has its own **Kubernetes Service Account**, this is important because Istio uses service accounts as workload identities for access control.

> [!NOTE]- frontend.yaml
> ```yaml
> [cloud_user@ip-10-0-0-218 lab-files]$ cat frontend.yaml
> apiVersion: v1
> kind: ServiceAccount
> metadata:
>   name: frontend-sa
>   namespace: zerotrust
> ---
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: frontend
>   namespace: zerotrust
>   labels:
>     app: frontend
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: frontend
>   template:
>     metadata:
>       labels:
>         app: frontend
>     spec:
>       serviceAccountName: frontend-sa
>       containers:
>       - name: nginx
>         image: nginx:alpine
>         ports:
>         - containerPort: 80
>         command: ["/bin/sh", "-c"]
>         args:
>         - |
>           echo '<html><body><h1>Frontend Service</h1><p>This is the frontend microservice.</p></body></html>' > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: frontend
>   namespace: zerotrust
>   labels:
>     app: frontend
> spec:
>   ports:
>   - port: 80
>     name: http
>     targetPort: 80
>   selector:
>     app: frontend
> [cloud_user@ip-10-0-0-218 lab-files]$
> ```

> [!NOTE]- backend.yaml
> ```yaml
> [cloud_user@ip-10-0-0-218 lab-files]$ cat backend.yaml
> apiVersion: v1
> kind: ServiceAccount
> metadata:
>   name: backend-sa
>   namespace: zerotrust
> ---
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: backend
>   namespace: zerotrust
>   labels:
>     app: backend
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: backend
>   template:
>     metadata:
>       labels:
>         app: backend
>     spec:
>       serviceAccountName: backend-sa
>       containers:
>       - name: nginx
>         image: nginx:alpine
>         ports:
>         - containerPort: 80
>         command: ["/bin/sh", "-c"]
>         args:
>         - |
>           echo '<html><body><h1>Backend Service</h1><p>This is the backend microservice. Access is controlled by authorization policies.</p></body></html>' > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: backend
>   namespace: zerotrust
>   labels:
>     app: backend
> spec:
>   ports:
>   - port: 80
>     name: http
>     targetPort: 80
>   selector:
>     app: backend
> [cloud_user@ip-10-0-0-218 lab-files]$
> ```

```bash
kubectl apply -f frontend.yaml
kubectl apply -f backend.yaml
```

```
serviceaccount/frontend-sa created
deployment.apps/frontend created
service/frontend created

serviceaccount/backend-sa created
deployment.apps/backend created
service/backend created
```

After the pods started, I checked their status:
```bash
kubectl get pods -n zerotrust
```

```
NAME                        READY   STATUS    RESTARTS   AGE
backend-f6696465c-lkggr     2/2     Running   0          24s
frontend-5ccb44dbcc-44ncn   2/2     Running   0          48s
```

Notice `2/2` in the READY column, each pod is running **two containers**: the application container and the **Envoy sidecar proxy**. This confirms automatic sidecar injection worked correctly.

I verified the sidecar was present by inspecting the frontend pod:

```bash
kubectl describe pod -n zerotrust -l app=frontend | grep -E "Name:|Image:"
```

```
Name:             frontend-5ccb44dbcc-44ncn
    Image:         docker.io/istio/proxyv2:1.29.2
    Image:         docker.io/istio/proxyv2:1.29.2
    Image:         nginx:alpine
```

Two images are present: `nginx:alpine` (the app) and `istio/proxyv2` (the Envoy sidecar). Sidecar injection confirmed.

> [!NOTE]- sleep.yaml
> ```yaml
> [cloud_user@ip-10-0-0-218 lab-files]$ cat sleep.yaml
> apiVersion: v1
> kind: ServiceAccount
> metadata:
>   name: sleep-sa
>   namespace: zerotrust
> ---
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: sleep
>   namespace: zerotrust
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: sleep
>   template:
>     metadata:
>       labels:
>         app: sleep
>     spec:
>       serviceAccountName: sleep-sa
>       containers:
>       - name: sleep
>         image: curlimages/curl:latest
>         command: ["/bin/sleep", "infinity"]
> [cloud_user@ip-10-0-0-218 lab-files]$
> ```

```bash
kubectl apply -f sleep.yaml
```

```
serviceaccount/sleep-sa created
deployment.apps/sleep created
```

Finally, I ran a **quick connectivity test** to confirm the mesh was passing traffic:

```bash
kubectl exec -n zerotrust deploy/sleep -- curl -s http://backend
```
<!--
**`kubectl exec -n zerotrust deploy/sleep --`** You're telling Kubernetes: _"open a shell session inside the running container of the `sleep` deployment, in the `zerotrust` namespace."_ Everything after `--` is the command that runs _inside_ that container.

**`curl -s http://backend`** Inside the sleep pod, you're making an HTTP GET request to a service named `backend`. The `-s` flag just means "silent" — suppress curl's progress output. Kubernetes DNS resolves `backend` to the full service address `backend.zerotrust.svc.cluster.local` automatically.
-->`
```html
<html><body><h1>Backend Service</h1><p>This is the backend microservice. Access is controlled by authorization policies.</p></body></html>
```

✅ The service mesh is up and routing traffic successfully.

>It succeeds because at this point in the lab, **no restriction policies exist yet**. We haven't applied `PeerAuthentication` or any `AuthorizationPolicy` yet.

---

## Part 2 - Enforcing Mutual TLS

### What is mTLS?

[[Mutual TLS (two-way)]]

In a service mesh, Istio issues certificates automatically to every sidecar, making mTLS seamless to configure.

### PeerAuthentication Policy

I applied a `PeerAuthentication` resource set to `STRICT` mode. This tells Istio that every service in the `zerotrust` namespace must use mTLS, plaintext connections are rejected outright. **PeerAuthentication** handles the server side (what connections to accept).

```bash
kubectl apply -f peer-authentication.yaml
```

> [!NOTE]- peer-authentication.yaml
> ```yaml
> apiVersion: security.istio.io/v1beta1
> kind: PeerAuthentication
> metadata:
>   name: default
>   namespace: zerotrust
> spec:
>   mtls:
>     mode: STRICT
> ```
> 
> `name: default` - The name `default` is special, Istio treats it as the namespace-wide policy, applying to every workload in the namespace *(`zerotrust`)* automatically
   `mtls.mode: STRICT` - **Server-side enforcement** reject any connection that doesn't present a valid mTLS certificate|
> 

```
peerauthentication.security.istio.io/default created
```

```bash
kubectl get peerauthentication -n zerotrust
```

```
NAME      MODE     AGE
default   STRICT   29s
```

### DestinationRule

I also applied a `DestinationRule` to configure the _client side_, telling Envoy sidecars to use `ISTIO_MUTUAL` TLS when sending traffic to any service in the namespace. **DestinationRule** handles the client side (how to initiate connections).

```bash
kubectl apply -f destination-rule.yaml
```

> [!NOTE]- destination-rule.yaml 
> ```yaml
> apiVersion: networking.istio.io/v1beta1
> kind: DestinationRule
> metadata:
>   name: default
>   namespace: zerotrust
> spec:
>   host: "*.zerotrust.svc.cluster.local"
>   trafficPolicy:
>     tls:
>       mode: ISTIO_MUTUAL
> ```
>
> `host: "*.zerotrust.svc.cluster.local"` - it's telling the client-side Envoy _"when you're about to call any service matching this pattern, use these settings"_. The `*` wildcard covers all services in the namespace
   `trafficPolicy.tls.mode: ISTIO_MUTUAL` - use Istio-managed certificates when initiating the connection
<!--
```
*.zerotrust.svc.cluster.local
│  │         │   │
│  │         │   └── cluster.local   → the default DNS domain for the entire cluster
│  │         └────── svc             → indicates this is a Service (not a pod or node)
│  └──────────────── zerotrust       → the namespace
└─────────────────── *               → any service name
```
-->

```bash
kubectl get destinationrule -n zerotrust
```

```
NAME      HOST                            AGE
default   *.zerotrust.svc.cluster.local   52s
```

### Verifying mTLS Blocks Plaintext Traffic

To prove that STRICT mTLS actually rejects non-mesh clients, I created a separate namespace **without** sidecar injection and tried to reach the backend from there:

```bash
kubectl create namespace no-mesh
kubectl run curl-test --image=curlimages/curl:latest -n no-mesh -- sleep infinity
kubectl wait --for=condition=ready pod/curl-test -n no-mesh --timeout=60s

kubectl exec -n no-mesh curl-test -- curl -s -m 5 http://backend.zerotrust
```

```
command terminated with exit code 56
```

Exit code 56 means the connection was forcibly reset, the Envoy sidecar on the backend rejected the plaintext request because the client had no mTLS certificate. This is exactly the behavior we want: **no certificate, no access**.
<!--
### What Happens Step by Step

1. `curl-test` initiates a plain **HTTP** connection to `backend.zerotrust` (no TLS, no cert)
2. The request crosses the namespace boundary — Kubernetes networking allows this by default, namespaces are not network firewalls
3. The request arrives at the **backend's Envoy sidecar**
4. Envoy checks: _"Does this connection have a valid mTLS certificate?"_
5. It doesn't — so Envoy **resets the connection**
-->
---

## Part 3 - Identity-Based Authorization Policies

### The Zero Trust Authorization Model

With mTLS in place, all traffic is encrypted and authenticated. Now I needed to add **authorization**, controlling _which_ authenticated services are allowed to talk to each other.

Istio uses an important pattern here: when you create an `ALLOW` policy for a workload, Istio automatically applies an **implicit deny-all** for any traffic that doesn't match. You never need a separate "deny everything else" rule, Istio's default behavior handles it.

### Applying the Policy

I applied an `AuthorizationPolicy` that explicitly allows only the `frontend-sa` service account to access the `backend` service:

```bash
kubectl apply -f authz-allow-frontend.yaml
```
<!--
- so the policies are applied to the service account not the resource itself? service account is the identity of the resource?
- The Service Account IS the identity of the workload
  Istio isn't saying "allow the frontend _pod_" or "allow traffic from IP `10.x.x.x`" — it's saying **"allow any workload presenting a certificate that was issued to `frontend-sa`."**
-->

> [!NOTE]- authz-allow-frontend.yaml
> ```yaml
> apiVersion: security.istio.io/v1beta1
> kind: AuthorizationPolicy
> metadata:
>   name: allow-frontend
>   namespace: zerotrust
> spec:
>   selector:
>     matchLabels:
>       app: backend
>   action: ALLOW
>   rules:
>   - from:
>     - source:
>         principals:
>         - cluster.local/ns/zerotrust/sa/frontend-sa
> ```

```bash
kubectl get authorizationpolicy -n zerotrust
```

```
NAME             ACTION   AGE
allow-frontend   ALLOW    54s
```

### Testing Access Control

**Frontend → Backend (should succeed):**

```bash
kubectl exec -n zerotrust deploy/frontend -- curl -s http://backend
```

```html
<html><body><h1>Backend Service</h1><p>This is the backend microservice. Access is controlled by authorization policies.</p></body></html>
```

✅ Frontend has `frontend-sa` - matches the policy, access granted.

**Sleep → Backend (should be denied):**

```bash
kubectl exec -n zerotrust deploy/sleep -- curl -s -o /dev/null -w "%{http_code}" http://backend
```

```
403
```

❌ Sleep uses `sleep-sa` - not in the ALLOW policy, implicitly denied. No separate deny rule needed.

### Confirming Identity Mapping

```bash
kubectl get pods -n zerotrust -o custom-columns=NAME:.metadata.name,SERVICE_ACCOUNT:.spec.serviceAccountName
```

```
NAME                        SERVICE_ACCOUNT
backend-f6696465c-lkggr     backend-sa
frontend-5ccb44dbcc-44ncn   frontend-sa
sleep-86d948cc48-b8h4z      sleep-sa
```

This table makes the identity model clear, each workload has a distinct service account, and Istio uses these identities (not IP addresses or network segments) as the basis for access decisions.

---

## Summary

|Security Layer|Mechanism|Result|
|---|---|---|
|Encryption|mTLS via `PeerAuthentication` (STRICT)|All traffic encrypted; plaintext rejected|
|Authentication|Istio-issued certificates per sidecar|Every service proves its identity|
|Authorization|`AuthorizationPolicy` (ALLOW + implicit deny)|Only explicitly allowed identities can connect|

### Key Takeaways

- **Sidecar injection** is the foundation of the data plane, it transparently intercepts all pod traffic without application changes.
- **STRICT mTLS** means a pod without an Istio certificate (i.e., outside the mesh) simply cannot connect, regardless of network access.
- **Authorization Policies use identity, not IP**, this is the core of Zero Trust. A service's location in the network is irrelevant; what matters is _who_ it is.
- **Implicit deny-all** removes the risk of misconfigured permissive defaults. Once an ALLOW policy exists, unlisted traffic is automatically blocked.

This pattern, mesh enrollment, mTLS enforcement, identity-based authorization, is the production-ready Zero Trust blueprint for Kubernetes workloads.