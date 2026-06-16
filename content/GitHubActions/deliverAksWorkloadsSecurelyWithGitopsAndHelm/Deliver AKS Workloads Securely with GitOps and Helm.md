---

tags:
  - azure
  - Kubernetes
  - GitOps
  - ArgoCD
  - githubactions
  - ci/cd
  - security
  - helm
  - AKS
linkedin: "False"
quartz: "False"
refactored: "False"
pluralsight: "True"
hands-on: "True"
completed: "True"
hardlinked: "True"
---
%%so github actions build and image out of the repo when there are code changes, argpcd uses image (image tag updated in manifest) and deploys it in the cluster, via manifest that were generated with helm templates

GitHub Actions builds an image from the code and pushes it to the container registry. It then updates the image tag in the values file in Git. ArgoCD detects that change, uses Helm to generate the Kubernetes manifests with the new tag, and applies them to the cluster. Kubernetes then pulls the image from the registry and runs i
%%
## Overview 

In this hands-on lab I took on the role of a DevSecOps engineer responsible for securely delivering workloads to an Azure Kubernetes Service (AKS) cluster using GitOps principles and Helm. The goal was to build a fully automated CI/CD pipeline that builds container images, tests them in a staging environment using automated security scanning, and promotes approved releases to production - all without hardcoded credentials.

The key technologies used were:

- **[[ArgoCD]]** for GitOps-driven continuous delivery to the AKS cluster 
- **[[Helm]]** for packaging and managing Kubernetes application releases 
- **GitHub Actions** for CI/CD automation
- **Workload Identity Federation** to authenticate GitHub Actions with Azure without storing long-lived secrets %%[[federated identity|federated user]]%%
- **OWASP ZAP** for automated security scanning against the staging environment

---

## Architecture

The pipeline follows this general flow:

```
Code Push to main
       │
       ▼
GitHub Actions: build-and-push
  └─ Build container images via Azure Container Registry (ACR)
       │
       ▼
GitHub Actions: release-and-test-staging
  └─ Update Helm values, ArgoCD syncs, ZAP security scan, GitHub Issues created
       │
       ▼
Manual Approval Gate (Required Reviewer)
       │
       ▼
GitHub Actions: release-production
  └─ Update Helm values, ArgoCD syncs, Live in Production
```

---

## Step 1 - %%Fork and %%Configure the GitHub Repository

**Repository**: [Lab-Deliver-AKS-Workloads-Securely-with-GitOps-and-Helm](https://github.com/hectorproko/Lab-Deliver-AKS-Workloads-Securely-with-GitOps-and-Helm/)

%%### Why

The lab application lives in a Pluralsight-owned repository. Forking it into my own GitHub account gives me a personal copy that I can modify, connect to my Azure environment, and run GitHub Actions against. The repository also needs GitHub environments configured to enforce deployment gates and control which credentials are available at each stage.

### Fork the Repository

1. Sign into GitHub with your personal account.
2. Navigate to the lab repository: `https://github.com/pluralsight-cloud/Lab-Deliver-AKS-Workloads-Securely-with-GitOps-and-Helm`
3. Click **Fork**, confirm the owner and repository name, and click **Create Fork**.

> **Note:** If you've run this lab before, delete the existing fork first before creating a new one to avoid naming conflicts.

 %%

### Enable Issues

GitHub Issues will later be used by the ZAP security scanner to automatically report vulnerabilities found in the staging environment.

1. In the forked repository, go to **Settings**.
2. Under **Features**, enable the **Issues** checkbox.

### Create the Staging Environment

GitHub Environments let you scope secrets and enforce rules per deployment target.

1. Go to **Settings - Environments - New environment**.
2. Name it `staging` and click **Configure environment**.

### Create the Production Environment

The production environment adds a required reviewer gate - no deployment reaches production without a human explicitly approving it.

1. Go back to **Environments - New environment**.
2. Name it `production` and click **Configure environment**.
3. Enable **Required reviewers** and add your GitHub username.
4. Click **Save protection rules**.

> **Output:** Two environments are now visible under Settings - Environments: `staging` with no gate and `production` with a required reviewer.

![[Pasted image 20260511144528.png]] ![[Pasted image 20260511144642.png]]

---

## Step 2 - Install and Configure ArgoCD

### Why

ArgoCD is the GitOps engine of this pipeline. Rather than having the CI/CD pipeline push manifests directly to Kubernetes, ArgoCD continuously watches the Git repository and pulls the desired state into the cluster. This model means the cluster always reflects what is in Git - making rollbacks as simple as reverting a commit and keeping an auditable trail of every change.

### Install ArgoCD on the AKS Cluster
%%ArgoCD is deployed **as workloads inside the AKS cluster itself**, running as pods in the `argocd` namespace.%%
1. In the Azure Portal, open **Cloud Shell** and select **Bash**.
    
    ![[Pasted image 20260511145026.png]]
    
2. Set variables for the resource group and AKS cluster name, then retrieve cluster credentials:
    
    ```bash
    RG=$(az group list --query [].name --output tsv)
    AKS=$(az aks list --resource-group $RG --query [].name --output tsv)
    az aks get-credentials --resource-group $RG --name $AKS
    ```
    
    **Example output:** %%[[Context (kubernetes)]]%%
    
    ```
    Merged "aks-xtj4c4iaele7a" as current context in /home/cloud/.kube/config
    ```
    
3. Create the `argocd` namespace and deploy ArgoCD from the official stable manifests:
    
    ```bash
    kubectl create namespace argocd
    kubectl create -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```
    
    **Example output (truncated):**
    
    ```
    namespace/argocd created
    customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
    customresourcedefinition.apiextensions.k8s.io/applicationsets.argoproj.io created
    serviceaccount/argocd-application-controller created
    serviceaccount/argocd-server created
    ...
    deployment.apps/argocd-server created
    statefulset.apps/argocd-application-controller created
    networkpolicy.networking.k8s.io/argocd-server-network-policy created
    ```
    
    Wait for all resources to finish creating before proceeding.
    
    ![[Pasted image 20260511145419.png]]
    

### Create ArgoCD Applications
%%
ArgoCD uses a custom resource called an `Application` that tells it which **Git repository to watch**, which **Helm values file to use**, and which Kubernetes **namespace to deploy into**.
%%
Exactly. The `Application` object is how you register a target with ArgoCD. It tells ArgoCD three things:

- **What to watch** - the Git repository and path (`repoURL`, `path`, `targetRevision`)
- **How to render it** - Helm in this case, with a specific values file
- **Where to deploy it** - the destination cluster and namespace

Once that object exists, ArgoCD takes over. It continuously compares what's in Git against what's running in the cluster, and with `automated` sync enabled it will automatically apply any drift it detects. That's the pull model, you never push to the cluster directly, you push to Git and ArgoCD pulls it in

---

1. Export your forked repository URL and the Azure Container Registry login server as environment variables:
    
    ```bash
    export REPO_URL="https://github.com/<your-github-username>/<your-repo-name>"
    export IMAGE_REGISTRY=$(az acr list --query [].loginServer -o tsv)
    ```
    
2. Create the ArgoCD Application for the **staging** environment:
    
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: simple-grocery-store-staging
      namespace: argocd
    spec:
      project: default
      destination:
        server: https://kubernetes.default.svc
        namespace: staging
      source:
        repoURL: ${REPO_URL}
        targetRevision: HEAD
        path: charts/simple-grocery-store
        helm:
          valueFiles:
            - values/staging.yaml
          values: |
            global:
              imageRegistry: ${IMAGE_REGISTRY}/
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
    EOF
    ```
    
    **Example output:**
    
    ```
    application.argoproj.io/simple-grocery-store-staging created
    ```
    
3. Create the ArgoCD Application for the **production** environment:
    
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: simple-grocery-store-production
      namespace: argocd
    spec:
      project: default
      destination:
        server: https://kubernetes.default.svc
        namespace: production
      source:
        repoURL: ${REPO_URL}
        targetRevision: HEAD
        path: charts/simple-grocery-store
        helm:
          valueFiles:
            - values/production.yaml
          values: |
            global:
              imageRegistry: ${IMAGE_REGISTRY}/
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
    EOF
    ```
%%
No, there are three differences between the two:
- `metadata.name` - `simple-grocery-store-staging` vs `simple-grocery-store-production`
- `spec.destination.namespace` - `staging` vs `production`
- `spec.source.helm.valueFiles` - `values/staging.yaml` vs `values/production.yaml`
  
we are giving the helm values via file and variable
[[Helm]]
The Helm chart is the template - it contains all the Kubernetes manifests (Deployments, Services, ConfigMaps, etc.) needed to run the grocery store, but with placeholders instead of hardcoded values. Think of it like a blueprint that doesn't know yet which environment it's being built for.

Your Ansible analogy is solid. The chart is like the role - reusable, parameterized, self-contained. The values file is like the inventory/vars file that tells the role how to behave in a specific environment.

The one thing to add to your mental model: in this lab you didn't write the chart, you consumed it. The grocery store chart was already in the repo. Your job was to wire up the values files and let ArgoCD + Helm do the rendering. That's actually the more common real-world pattern - teams maintain one chart and promote releases across environments purely by swapping values.
%%

**Example output:**

```
application.argoproj.io/simple-grocery-store-production created
```
    
4. Confirm both applications were registered:
    
    ```bash
    kubectl get application --namespace argocd
    ```
    
    **Example output:**
    
    ```
    NAME                              SYNC STATUS   HEALTH STATUS
    simple-grocery-store-production   OutOfSync     Progressing
    simple-grocery-store-staging      OutOfSync     Progressing
    ```
    
    > **Note:** The `OutOfSync` status is expected at this point. The applications are registered, but the CI/CD pipeline hasn't pushed any container images to the registry yet, so there's nothing to deploy.
    

---

## Step 3 - Configure CI/CD with Workload Identity Federation

### Why

A naive approach to connecting GitHub Actions to Azure would be to create a service principal, generate a client secret, and store it as a GitHub secret. The problem is that secrets can leak, expire unexpectedly, or need manual rotation. Workload Identity Federation solves this by establishing a trust relationship between GitHub's [[OpenID Connect (OIDC)|OIDC]] provider and Azure Active Directory. GitHub Actions receives a short-lived OIDC token per job - no stored passwords, no long-lived credentials.
%%**You're not turning workloads into federated users.** You're giving GitHub Actions the ability to _prove its identity_ to Azure without needing a password. The "federation" part means Azure agrees to trust GitHub's word when GitHub says "this job is running from repo X, branch Y, environment Z." Azure then issues a short-lived access token based on that trust — no account, no password involved.

---

**On your second question — yes, exactly.** Without Workload Identity Federation, the classic approach is:

1. Create a **Service Principal** in Azure (think: a non-human account representing your app or pipeline)
2. Generate a **client secret** for it (basically a password)
3. Copy that secret into GitHub as a repository secret
4. GitHub Actions uses it on every run to authenticate%%
### Create Federated Credentials

1. In the Azure Portal, minimize Cloud Shell and navigate to the **Managed Identity** resource in your resource group.
    
    ![[Pasted image 20260511150848.png]]
    
2. Go to **Settings - Federated credentials - Add Credential**.
    
3. Select **GitHub Actions deploying Azure resources** as the scenario.
    
4. Create the first federated credential for the **staging** environment:
 
    |Field|Value|
    |---|---|
    |Organization|Your GitHub username|
    |Repository|Your forked repository name|
    |Entity|Environment|
    |Environment|`staging`|
    |Name|e.g. `<username>-AKS-GitOps-Helm-Staging`|
    
    ![[Pasted image 20260609122244.png]]
    
5. Click **Add**, then repeat to create a second credential for **production**:
    
    |Field|Value|
    |---|---|
    |Organization|Your GitHub username|
    |Repository|Your forked repository name|
    |Entity|Environment|
    |Environment|`production`|
    |Name|e.g. `<username>-AKS-GitOps-Helm-Prod`|
    
6. Create a third credential for the **main** branch (used by the build job to push images to ACR):
    
    |Field|Value|
    |---|---|
    |Organization|Your GitHub username|
    |Repository|Your forked repository name|
    |Entity|Branch|
    |Branch|`main`|
    |Name|e.g. `<username>-AKS-GitOps-Helm-Main`|
    
    > **Note:** In a real production setup you would use separate managed identities per environment, each scoped with least-privilege permissions. This lab uses a single identity with Contributor for simplicity.
    
    ![[Pasted image 20260609122648.png]]
    
7. Go to **Settings - Properties** and copy the **Tenant ID**.
    
8. Go to the **Overview** page of the managed identity and copy the **Client ID** and **Subscription ID**.
    
    ![[Pasted image 20260511152503.png]]
    

### Create GitHub Secrets

These three values need to be available to GitHub Actions workflows at runtime. They are stored as encrypted repository secrets.

1. In your forked repository, go to **Settings - Secrets and variables - Actions - New repository secret**.
    
2. Create the following three secrets:
    
    |Secret Name|Value|
    |---|---|
    |`AZURE_CLIENT_ID`|Client ID of the User-assigned managed identity|
    |`AZURE_SUBSCRIPTION_ID`|Azure Subscription ID|
    |`AZURE_TENANT_ID`|Azure Active Directory Tenant ID|
    
    > **Why "Actions" and not the other options?**
    > 
    > - **Actions** - secrets for GitHub Actions workflows (this is what we need)
    > - **Dependabot** - secrets for dependency update checks only
    > - **Codespaces** - environment variables for cloud dev environments
    > - **Agents** - related to self-hosted runners, not credential storage
    
    ![[Pasted image 20260511154720.png]]
    

### Set Up the GitHub Actions Workflow

The workflow file defines three jobs that run in sequence:

- **`build-and-push`** - checks out the code and builds three container images (frontend, cart-service, product-service) into ACR, tagging each with the GitHub run number.
- **`release-and-test-staging`** - updates the Helm values file for staging with the new image tag, commits and pushes to Git (which triggers ArgoCD to sync), polls until the deployment is live, retrieves the load balancer IP, and runs an OWASP ZAP baseline security scan. Any findings are automatically filed as GitHub Issues.
- **`release-production`** - waits for manual approval, then repeats the Helm values update for the production values file.

1. Go to the **Actions** tab in the repository.
    
2. Click **set up a workflow yourself**.
    
    ![[Pasted image 20260511155340.png]]
    
3. Replace the default content with the following workflow:
    
    ```yaml
    name: Build and Release
    on:
      push:
        branches: [ main ]
        paths-ignore:
          - 'charts/simple-grocery-store/values/**'
    
    jobs:
      build-and-push:
        permissions:
          id-token: write   # Required to fetch an OIDC token
          contents: read    # Read access to repository contents
        runs-on: ubuntu-latest
        steps:
        - name: Checkout code
          uses: actions/checkout@v4
    
        - name: Azure login
          uses: azure/login@v2
          with:
            client-id: ${{ secrets.AZURE_CLIENT_ID }}
            tenant-id: ${{ secrets.AZURE_TENANT_ID }}
            subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    
        - name: Azure CLI script
          uses: azure/cli@v2
          with:
            azcliversion: latest
            inlineScript: |
              RG=$(az group list --query [].name --output tsv)
              ACR=$(az acr list --resource-group $RG --query [].name --output tsv)
              cd frontend
              az acr build --registry "$ACR" --image "frontend:${GITHUB_RUN_NUMBER}" .
              cd ../services/cart-service
              az acr build --registry "$ACR" --image "cart-service:${GITHUB_RUN_NUMBER}" .
              cd ../product-service
              az acr build --registry "$ACR" --image "product-service:${GITHUB_RUN_NUMBER}" .
    
      release-and-test-staging:
        needs: build-and-push
        permissions:
          id-token: write    # Required to fetch an OIDC token
          contents: write    # Write access to push Helm value updates
          issues: write      # Write access to create ZAP scan issues
        runs-on: ubuntu-latest
        environment: staging
        env:
          POLL_INTERVAL: 60   # Seconds between deployment checks
          MAX_ATTEMPTS: 20    # Max polling attempts before timeout
        steps:
        - name: Azure login
          uses: azure/login@v2
          with:
            client-id: ${{ secrets.AZURE_CLIENT_ID }}
            tenant-id: ${{ secrets.AZURE_TENANT_ID }}
            subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    
        - name: Kubectl tool installer
          uses: Azure/setup-kubectl@v4.0.0
    
        - name: Configure AKS Credentials
          run: |
            RG=$(az group list --query [].name --output tsv)
            AKS=$(az aks list --resource-group $RG --query [].name --output tsv)
            az aks get-credentials --resource-group $RG --name $AKS --overwrite-existing
    
        - name: Checkout code
          uses: actions/checkout@v4
    
        - name: Update Helm Release tags
          run: |
            sed -E "/frontend:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/staging.yaml
            sed -E "/cartService:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/staging.yaml
            sed -E "/productService:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/staging.yaml
            git config --global user.name "GitHub Action"
            git config --global user.email "action@github.com"
            git add charts/simple-grocery-store/values/staging.yaml
            git commit -m "Update HelmRelease image tags to ${GITHUB_RUN_NUMBER}"
            git fetch origin
            git rebase origin/main
            git push
    
        - name: Wait for Image Tag to Match Run Number
          run: |
            TARGET_TAG="${{ github.run_number }}"
            echo "Polling for Deployment spec tag to match '$TARGET_TAG'..."
            attempt=0
            while [ $attempt -lt ${{ env.MAX_ATTEMPTS }} ]; do
              FULL_IMAGE_SPEC=$(kubectl get deployment frontend-deployment -n staging \
                -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
              CURRENT_TAG="${FULL_IMAGE_SPEC##*:}"
              if [ -z "$FULL_IMAGE_SPEC" ]; then
                echo "Attempt $attempt/${{ env.MAX_ATTEMPTS }}: Deployment not retrievable. Retrying..."
              elif [ "$CURRENT_TAG" = "$TARGET_TAG" ]; then
                echo "Match found: Image tag is '$CURRENT_TAG'."
                exit 0
              else
                echo "Attempt $attempt/${{ env.MAX_ATTEMPTS }}: Current tag '$CURRENT_TAG'. Retrying in ${{ env.POLL_INTERVAL }}s..."
              fi
              attempt=$((attempt + 1))
              sleep ${{ env.POLL_INTERVAL }}
            done
            echo "Timeout: Tag did not match after $(( ${{ env.POLL_INTERVAL }} * ${{ env.MAX_ATTEMPTS }} )) seconds."
            exit 1
    
        - name: Retrieve Load Balancer IP
          id: get-ip
          run: |
            IP=$(kubectl get service frontend-service -n staging \
              -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
            if [[ -z "$IP" ]]; then
              echo "Error: IP not available."
              exit 1
            fi
            echo "IP retrieved: $IP"
            echo "ip=$IP" >> $GITHUB_OUTPUT
            echo "target_url=http://$IP" >> $GITHUB_OUTPUT
    
        - name: ZAP Scan
          if: steps.get-ip.outputs.ip != ''
          uses: zaproxy/action-baseline@v0.14.0
          with:
            token: ${{ secrets.GITHUB_TOKEN }}
            target: ${{ steps.get-ip.outputs.target_url }}
    
      release-production:
        needs: release-and-test-staging
        permissions:
          id-token: write
          contents: write
        runs-on: ubuntu-latest
        environment: production
        steps:
        - name: Azure login
          uses: azure/login@v2
          with:
            client-id: ${{ secrets.AZURE_CLIENT_ID }}
            tenant-id: ${{ secrets.AZURE_TENANT_ID }}
            subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    
        - name: Kubectl tool installer
          uses: Azure/setup-kubectl@v4.0.0
    
        - name: Configure AKS Credentials
          run: |
            RG=$(az group list --query [].name --output tsv)
            AKS=$(az aks list --resource-group $RG --query [].name --output tsv)
            az aks get-credentials --resource-group $RG --name $AKS --overwrite-existing
    
        - name: Checkout code
          uses: actions/checkout@v4
    
        - name: Update Helm Release tags
          run: |
            sed -E "/frontend:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/production.yaml
            sed -E "/cartService:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/production.yaml
            sed -E "/productService:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/production.yaml
            git config --global user.name "GitHub Action"
            git config --global user.email "action@github.com"
            git add charts/simple-grocery-store/values/production.yaml
            git commit -m "Update HelmRelease image tags to ${GITHUB_RUN_NUMBER}"
            git fetch origin
            git rebase origin/main
            git push
    ```
    
    > **What this workflow does end-to-end:** The `build-and-push` job authenticates with Azure using a short-lived OIDC token and builds all three service images into ACR, tagging each with the run number. The `release-and-test-staging` job then updates the Helm values file with the new tag, commits it back to Git, and waits for ArgoCD to detect the change and roll out the new pods. Once the deployment is live, ZAP performs a baseline security scan and files any findings as GitHub Issues. The `release-production` job is identical in structure but targets the production values file and is gated behind a required manual approval.
    
4. Click **Commit changes**, add a commit message such as `Add CI/CD workflow for build and release process`, and confirm.
    
    ![[Pasted image 20260511155455.png|500]]
    
After the workflow runs for the first time, the built images are visible in Azure Container Registry tagged with the GitHub run number.

![[Pasted image 20260611131537.png]]


---

## Step 4 - Review Issues and Release to Production

### Why

This final stage closes the loop on the entire pipeline. The staging environment has been running the new build and has been scanned for security issues by OWASP ZAP. Any findings are surfaced as GitHub Issues, giving the team a chance to review them before promoting to production. The production environment is protected by a required reviewer gate - a human must explicitly approve the deployment before the `release-production` job is allowed to run. This combination of automated security testing and manual approval is what makes the pipeline suitable for workloads that need a controlled, auditable release process.

### Trigger the Workflow

The GitHub Actions workflow only fires on a push to `main` that does **not** touch the `charts/simple-grocery-store/values/` folder (that path is excluded to prevent the automated Helm tag commit from re-triggering the pipeline in a loop). To kick off a new run without a real code change, the easiest approach is to make a small, harmless edit to `README.md` and commit it directly to `main`.

1. In the repository, open `README.md` and make a minor edit (e.g., add a blank line or update a description).
2. Commit the change directly to `main`.

This fires the **Build and Release** workflow.

### Watch the Workflow Progress

1. Go to the **Actions** tab. You should see the new run queued or already running.
    
    The run progresses through three stages visible in the workflow graph:
    
    - **`build-and-push`** - builds the three container images into ACR (~2 minutes)
        
    - **`release-and-test-staging`** - updates staging Helm values, waits for ArgoCD to sync, then runs the ZAP scan (~3-20 minutes depending on polling)
        
    - **`release-production`** - waits for manual approval before running
        
    
    > **Important:** The `release-and-test-staging` job polls every 60 seconds up to 20 times while waiting for ArgoCD to sync the new image tag. The Actions tab will show it as "In progress" for several minutes - this is normal. Do not cancel the run or assume it has failed.
    
    **Example output - workflow in progress (build complete, staging deploying):**
    
    ![[Pasted image 20260609123343.png]]
    
2. Once `release-and-test-staging` completes, go to the **Issues** tab.
    
    The ZAP baseline scan automatically files any security findings it detects against the staging environment as GitHub Issues. Review each one - in a real environment these would be triaged and assigned. For this lab, reviewing them is sufficient before proceeding to the production approval.
    
    ![[Pasted image 20260609152202.png]] ![[Pasted image 20260609152255.png]]
    
3. Return to the **Actions** tab and select the latest workflow run. You will see a yellow banner:
    
    ```
    hectorproko requested your review to deploy to production
    ```
    
    The `release-production` job shows status **"production waiting for review"**.
    
    ![[Pasted image 20260609123933.png]]
    

### Approve the Production Deployment

4. Click **Review deployments**.
    
5. In the dialog, check the box next to **production**.
    
6. Leave an optional comment, for example: `We are good to go`.
    
7. Click **Approve and deploy**.
    
    **Example - Review deployments dialog:**
    
    ![[Pasted image 20260609124032.png]]
    

The `release-production` job is now unblocked and begins running.

![[Pasted image 20260609125335.png]]

Once the production job completes, the application is accessible via the frontend service's load balancer IP.

```
kubectl get svc --all-namespaces
```

```
NAMESPACE           NAME                                      TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                
production          frontend-service                          LoadBalancer   10.0.237.121   168.62.4.68      80:30126/TCP
staging             frontend-service                          LoadBalancer   10.0.141.37    52.190.138.121   80:31044/TCP 
```

![[Pasted image 20260611133422.png]]

---

## Key Concepts Reinforced

**GitOps pull model** - ArgoCD pulls desired state from Git rather than having pipelines push directly to Kubernetes. The cluster is always a reflection of the repository.

**Workload Identity Federation** - GitHub Actions authenticates to Azure using a short-lived OIDC token tied to the specific environment or branch. No service principal secrets are stored anywhere.

**Environment gates** - The `production` GitHub Environment requires a named reviewer to approve before the deployment job runs, providing a human checkpoint between staging and production.

**Automated security testing in CI** - The ZAP baseline scan runs automatically after every staging deployment. Security findings surface as GitHub Issues, creating a traceable record tied directly to the code change that introduced them.

**Helm as a release mechanism** - Rather than managing raw Kubernetes manifests, Helm charts allow values (like image tags) to be environment-specific while keeping the chart template shared. ArgoCD uses these value files as the source of truth for each environment's configuration.
<!--
%%
### SHould be put Helm
%%
## Overview

In this hands-on lab I took on the role of a DevSecOps engineer responsible for securely delivering workloads to an Azure Kubernetes Service (AKS) cluster using GitOps principles and Helm. The goal was to build a fully automated CI/CD pipeline that builds container images, tests them in a staging environment using automated security scanning, and promotes approved releases to production — all without hardcoded credentials.

The key technologies used were:

- **ArgoCD** for GitOps-driven continuous delivery to the AKS cluster
- **Helm** for packaging and managing Kubernetes application releases
- **GitHub Actions** for CI/CD automation
- **Workload Identity Federation** to authenticate GitHub Actions with Azure without storing long-lived secrets
- **OWASP ZAP** for automated security scanning against the staging environment

---

## Architecture

The pipeline follows this general flow:

```
Code Push to main
       │
       ▼
GitHub Actions: build-and-push
  └─ Build container images via Azure Container Registry (ACR)
       │
       ▼
GitHub Actions: release-and-test-staging
  └─ Update Helm values → ArgoCD syncs → ZAP security scan → GitHub Issues created
       │
       ▼
Manual Approval Gate (Required Reviewer)
       │
       ▼
GitHub Actions: release-production
  └─ Update Helm values → ArgoCD syncs → Live in Production
```

---

## Step 1 — Fork and Configure the GitHub Repository

### Why

The lab application lives in a Pluralsight-owned repository. Forking it into my own GitHub account gives me a personal copy that I can modify, connect to my Azure environment, and run GitHub Actions against. The repository also needs GitHub environments configured to enforce deployment gates and control which credentials are available at each stage.

### Fork the Repository

1. Sign into GitHub with your personal account.
2. Navigate to the lab repository: `https://github.com/pluralsight-cloud/Lab-Deliver-AKS-Workloads-Securely-with-GitOps-and-Helm`
3. Click **Fork**, confirm the owner and repository name, and click **Create Fork**.

> **Note:** If you've run this lab before, delete the existing fork first before creating a new one to avoid naming conflicts.

### Enable Issues

GitHub Issues will later be used by the ZAP security scanner to automatically report vulnerabilities found in the staging environment.

1. In the forked repository, go to **Settings**.
2. Under **Features**, enable the **Issues** checkbox.

### Create the Staging Environment

GitHub Environments let you scope secrets and enforce rules per deployment target.

1. Go to **Settings → Environments → New environment**.
2. Name it `staging` and click **Configure environment**.

### Create the Production Environment

The production environment adds a required reviewer gate — no deployment reaches production without a human explicitly approving it.

1. Go back to **Environments → New environment**.
2. Name it `production` and click **Configure environment**.
3. Enable **Required reviewers** and add your GitHub username.
4. Click **Save protection rules**.

> **Output:** Two environments are now visible under Settings → Environments — `staging` with no gate and `production` with a required reviewer.

![[Pasted image 20260511144528.png]] ![[Pasted image 20260511144642.png]]

---

## Step 2 — Install and Configure ArgoCD

### Why

ArgoCD is the GitOps engine of this pipeline. Rather than having the CI/CD pipeline push manifests directly to Kubernetes, ArgoCD continuously watches the Git repository and pulls the desired state into the cluster. This model means the cluster always reflects what is in Git — making rollbacks as simple as reverting a commit and keeping an auditable trail of every change.

### Install ArgoCD on the AKS Cluster

1. In the Azure Portal, open **Cloud Shell** and select **Bash**.
    
    ![[Pasted image 20260511145026.png]]
    
2. Set variables for the resource group and AKS cluster name, then retrieve cluster credentials:
    
    ```bash
    RG=$(az group list --query [].name --output tsv)
    AKS=$(az aks list --resource-group $RG --query [].name --output tsv)
    az aks get-credentials --resource-group $RG --name $AKS
    ```
    
    **Example output:**
    
    ```
    Merged "aks-xtj4c4iaele7a" as current context in /home/cloud/.kube/config
    ```
    
3. Create the `argocd` namespace and deploy ArgoCD from the official stable manifests:
    
    ```bash
    kubectl create namespace argocd
    kubectl create -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```
    
    **Example output (truncated):**
    
    ```
    namespace/argocd created
    customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
    customresourcedefinition.apiextensions.k8s.io/applicationsets.argoproj.io created
    serviceaccount/argocd-application-controller created
    serviceaccount/argocd-server created
    ...
    deployment.apps/argocd-server created
    statefulset.apps/argocd-application-controller created
    networkpolicy.networking.k8s.io/argocd-server-network-policy created
    ```
    
    Wait for all resources to finish creating before proceeding.
    
    ![[Pasted image 20260511145419.png]]
    

### Create ArgoCD Applications

ArgoCD uses a custom resource called an `Application` that tells it which Git repository to watch, which Helm values file to use, and which Kubernetes namespace to deploy into.

1. Export your forked repository URL and the Azure Container Registry login server as environment variables:
    
    ```bash
    export REPO_URL="https://github.com/<your-github-username>/<your-repo-name>"
    export IMAGE_REGISTRY=$(az acr list --query [].loginServer -o tsv)
    ```
    
2. Create the ArgoCD Application for the **staging** environment:
    
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: simple-grocery-store-staging
      namespace: argocd
    spec:
      project: default
      destination:
        server: https://kubernetes.default.svc
        namespace: staging
      source:
        repoURL: ${REPO_URL}
        targetRevision: HEAD
        path: charts/simple-grocery-store
        helm:
          valueFiles:
            - values/staging.yaml
          values: |
            global:
              imageRegistry: ${IMAGE_REGISTRY}/
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
    EOF
    ```
    
    **Example output:**
    
    ```
    application.argoproj.io/simple-grocery-store-staging created
    ```
    
3. Create the ArgoCD Application for the **production** environment:
    
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: simple-grocery-store-production
      namespace: argocd
    spec:
      project: default
      destination:
        server: https://kubernetes.default.svc
        namespace: production
      source:
        repoURL: ${REPO_URL}
        targetRevision: HEAD
        path: charts/simple-grocery-store
        helm:
          valueFiles:
            - values/production.yaml
          values: |
            global:
              imageRegistry: ${IMAGE_REGISTRY}/
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
    EOF
    ```
    
    **Example output:**
    
    ```
    application.argoproj.io/simple-grocery-store-production created
    ```
    
4. Confirm both applications were registered:
    
    ```bash
    kubectl get application --namespace argocd
    ```
    
    **Example output:**
    
    ```
    NAME                              SYNC STATUS   HEALTH STATUS
    simple-grocery-store-production   OutOfSync     Progressing
    simple-grocery-store-staging      OutOfSync     Progressing
    ```
    
    > **Note:** The `OutOfSync` status is expected at this point. The applications are registered, but the CI/CD pipeline hasn't pushed any container images to the registry yet, so there's nothing to deploy.
    

---

## Step 3 — Configure CI/CD with Workload Identity Federation

### Why

A naive approach to connecting GitHub Actions to Azure would be to create a service principal, generate a client secret, and store it as a GitHub secret. The problem is that secrets can leak, expire unexpectedly, or need manual rotation. Workload Identity Federation solves this by establishing a trust relationship between GitHub's OIDC provider and Azure Active Directory. GitHub Actions receives a short-lived OIDC token per job — no stored passwords, no long-lived credentials.

### Create Federated Credentials

1. In the Azure Portal, minimize Cloud Shell and navigate to the **Managed Identity** resource in your resource group.
    
    ![[Pasted image 20260511150848.png]]
    
2. Go to **Settings → Federated credentials → Add Credential**.
    
3. Select **GitHub Actions deploying Azure resources** as the scenario.
    
4. Create the first federated credential for the **staging** environment:
    
    |Field|Value|
    |---|---|
    |Organization|Your GitHub username|
    |Repository|Your forked repository name|
    |Entity|Environment|
    |Environment|`staging`|
    |Name|e.g. `<username>-AKS-GitOps-Helm-Staging`|
    
    ![[Pasted image 20260609122244.png]]
    
5. Click **Add**, then repeat to create a second credential for **production**:
    
    |Field|Value|
    |---|---|
    |Organization|Your GitHub username|
    |Repository|Your forked repository name|
    |Entity|Environment|
    |Environment|`production`|
    |Name|e.g. `<username>-AKS-GitOps-Helm-Prod`|
    
6. Create a third credential for the **main** branch (used by the build job to push images to ACR):
    
    |Field|Value|
    |---|---|
    |Organization|Your GitHub username|
    |Repository|Your forked repository name|
    |Entity|Branch|
    |Branch|`main`|
    |Name|e.g. `<username>-AKS-GitOps-Helm-Main`|
    
    > **Note:** In a real production setup you would use separate managed identities per environment, each scoped with least-privilege permissions. This lab uses a single identity with Contributor for simplicity.
    
    ![[Pasted image 20260609122648.png]]
    
7. Go to **Settings → Properties** and copy the **Tenant ID**.
    
8. Go to the **Overview** page of the managed identity and copy the **Client ID** and **Subscription ID**.
    
    ![[Pasted image 20260511152503.png]]
    

### Create GitHub Secrets

These three values need to be available to GitHub Actions workflows at runtime. They are stored as encrypted repository secrets.

1. In your forked repository, go to **Settings → Secrets and variables → Actions → New repository secret**.
    
2. Create the following three secrets:
    
    |Secret Name|Value|
    |---|---|
    |`AZURE_CLIENT_ID`|Client ID of the User-assigned managed identity|
    |`AZURE_SUBSCRIPTION_ID`|Azure Subscription ID|
    |`AZURE_TENANT_ID`|Azure Active Directory Tenant ID|
    
    > **Why "Actions" and not the other options?**
    > 
    > - **Actions** — secrets for GitHub Actions workflows (this is what we need)
    > - **Dependabot** — secrets for dependency update checks only
    > - **Codespaces** — environment variables for cloud dev environments
    > - **Agents** — related to self-hosted runners, not credential storage
    
    ![[Pasted image 20260511154720.png]]
    

### Set Up the GitHub Actions Workflow

The workflow file defines three jobs that run in sequence:

- **`build-and-push`** — checks out the code and builds three container images (frontend, cart-service, product-service) into ACR, tagging each with the GitHub run number.
- **`release-and-test-staging`** — updates the Helm values file for staging with the new image tag, commits and pushes to Git (which triggers ArgoCD to sync), polls until the deployment is live, retrieves the load balancer IP, and runs an OWASP ZAP baseline security scan. Any findings are automatically filed as GitHub Issues.
- **`release-production`** — waits for manual approval, then repeats the Helm values update for the production values file.

1. Go to the **Actions** tab in the repository.
    
2. Click **set up a workflow yourself**.
    
    ![[Pasted image 20260511155340.png]]
    
3. Replace the default content with the following workflow:
    
    ```yaml
    name: Build and Release
    on:
      push:
        branches: [ main ]
        paths-ignore:
          - 'charts/simple-grocery-store/values/**'
    
    jobs:
      build-and-push:
        permissions:
          id-token: write   # Required to fetch an OIDC token
          contents: read    # Read access to repository contents
        runs-on: ubuntu-latest
        steps:
        - name: Checkout code
          uses: actions/checkout@v4
    
        - name: Azure login
          uses: azure/login@v2
          with:
            client-id: ${{ secrets.AZURE_CLIENT_ID }}
            tenant-id: ${{ secrets.AZURE_TENANT_ID }}
            subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    
        - name: Azure CLI script
          uses: azure/cli@v2
          with:
            azcliversion: latest
            inlineScript: |
              RG=$(az group list --query [].name --output tsv)
              ACR=$(az acr list --resource-group $RG --query [].name --output tsv)
              cd frontend
              az acr build --registry "$ACR" --image "frontend:${GITHUB_RUN_NUMBER}" .
              cd ../services/cart-service
              az acr build --registry "$ACR" --image "cart-service:${GITHUB_RUN_NUMBER}" .
              cd ../product-service
              az acr build --registry "$ACR" --image "product-service:${GITHUB_RUN_NUMBER}" .
    
      release-and-test-staging:
        needs: build-and-push
        permissions:
          id-token: write    # Required to fetch an OIDC token
          contents: write    # Write access to push Helm value updates
          issues: write      # Write access to create ZAP scan issues
        runs-on: ubuntu-latest
        environment: staging
        env:
          POLL_INTERVAL: 60   # Seconds between deployment checks
          MAX_ATTEMPTS: 20    # Max polling attempts before timeout
        steps:
        - name: Azure login
          uses: azure/login@v2
          with:
            client-id: ${{ secrets.AZURE_CLIENT_ID }}
            tenant-id: ${{ secrets.AZURE_TENANT_ID }}
            subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    
        - name: Kubectl tool installer
          uses: Azure/setup-kubectl@v4.0.0
    
        - name: Configure AKS Credentials
          run: |
            RG=$(az group list --query [].name --output tsv)
            AKS=$(az aks list --resource-group $RG --query [].name --output tsv)
            az aks get-credentials --resource-group $RG --name $AKS --overwrite-existing
    
        - name: Checkout code
          uses: actions/checkout@v4
    
        - name: Update Helm Release tags
          run: |
            sed -E "/frontend:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/staging.yaml
            sed -E "/cartService:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/staging.yaml
            sed -E "/productService:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/staging.yaml
            git config --global user.name "GitHub Action"
            git config --global user.email "action@github.com"
            git add charts/simple-grocery-store/values/staging.yaml
            git commit -m "Update HelmRelease image tags to ${GITHUB_RUN_NUMBER}"
            git fetch origin
            git rebase origin/main
            git push
    
        - name: Wait for Image Tag to Match Run Number
          run: |
            TARGET_TAG="${{ github.run_number }}"
            echo "Polling for Deployment spec tag to match '$TARGET_TAG'..."
            attempt=0
            while [ $attempt -lt ${{ env.MAX_ATTEMPTS }} ]; do
              FULL_IMAGE_SPEC=$(kubectl get deployment frontend-deployment -n staging \
                -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
              CURRENT_TAG="${FULL_IMAGE_SPEC##*:}"
              if [ -z "$FULL_IMAGE_SPEC" ]; then
                echo "Attempt $attempt/${{ env.MAX_ATTEMPTS }}: Deployment not retrievable. Retrying..."
              elif [ "$CURRENT_TAG" = "$TARGET_TAG" ]; then
                echo "Match found: Image tag is '$CURRENT_TAG'."
                exit 0
              else
                echo "Attempt $attempt/${{ env.MAX_ATTEMPTS }}: Current tag '$CURRENT_TAG'. Retrying in ${{ env.POLL_INTERVAL }}s..."
              fi
              attempt=$((attempt + 1))
              sleep ${{ env.POLL_INTERVAL }}
            done
            echo "Timeout: Tag did not match after $(( ${{ env.POLL_INTERVAL }} * ${{ env.MAX_ATTEMPTS }} )) seconds."
            exit 1
    
        - name: Retrieve Load Balancer IP
          id: get-ip
          run: |
            IP=$(kubectl get service frontend-service -n staging \
              -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
            if [[ -z "$IP" ]]; then
              echo "Error: IP not available."
              exit 1
            fi
            echo "IP retrieved: $IP"
            echo "ip=$IP" >> $GITHUB_OUTPUT
            echo "target_url=http://$IP" >> $GITHUB_OUTPUT
    
        - name: ZAP Scan
          if: steps.get-ip.outputs.ip != ''
          uses: zaproxy/action-baseline@v0.14.0
          with:
            token: ${{ secrets.GITHUB_TOKEN }}
            target: ${{ steps.get-ip.outputs.target_url }}
    
      release-production:
        needs: release-and-test-staging
        permissions:
          id-token: write
          contents: write
        runs-on: ubuntu-latest
        environment: production
        steps:
        - name: Azure login
          uses: azure/login@v2
          with:
            client-id: ${{ secrets.AZURE_CLIENT_ID }}
            tenant-id: ${{ secrets.AZURE_TENANT_ID }}
            subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    
        - name: Kubectl tool installer
          uses: Azure/setup-kubectl@v4.0.0
    
        - name: Configure AKS Credentials
          run: |
            RG=$(az group list --query [].name --output tsv)
            AKS=$(az aks list --resource-group $RG --query [].name --output tsv)
            az aks get-credentials --resource-group $RG --name $AKS --overwrite-existing
    
        - name: Checkout code
          uses: actions/checkout@v4
    
        - name: Update Helm Release tags
          run: |
            sed -E "/frontend:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/production.yaml
            sed -E "/cartService:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/production.yaml
            sed -E "/productService:/,/tag:/ s/(tag: ).*/\1${GITHUB_RUN_NUMBER}/" -i charts/simple-grocery-store/values/production.yaml
            git config --global user.name "GitHub Action"
            git config --global user.email "action@github.com"
            git add charts/simple-grocery-store/values/production.yaml
            git commit -m "Update HelmRelease image tags to ${GITHUB_RUN_NUMBER}"
            git fetch origin
            git rebase origin/main
            git push
    ```
    
    > **What this workflow does end-to-end:** The `build-and-push` job authenticates with Azure using a short-lived OIDC token and builds all three service images into ACR, tagging each with the run number. The `release-and-test-staging` job then updates the Helm values file with the new tag, commits it back to Git, and waits for ArgoCD to detect the change and roll out the new pods. Once the deployment is live, ZAP performs a baseline security scan and files any findings as GitHub Issues. The `release-production` job is identical in structure but targets the production values file and is gated behind a required manual approval.
    
4. Click **Commit changes**, add a commit message such as `Add CI/CD workflow for build and release process`, and confirm.
    
    ![[Pasted image 20260511155455.png]]
    

---

## Step 4 — Review Issues and Release to Production

### Why

This final stage closes the loop on the entire pipeline. The staging environment has been running the new build and has been scanned for security issues by OWASP ZAP. Any findings are surfaced as GitHub Issues, giving the team a chance to review them before promoting to production. The production environment is protected by a required reviewer gate — a human must explicitly approve the deployment before the `release-production` job is allowed to run. This combination of automated security testing and manual approval is what makes the pipeline suitable for workloads that need a controlled, auditable release process.

### Trigger the Workflow

The GitHub Actions workflow only fires on a push to `main` that does **not** touch the `charts/simple-grocery-store/values/` folder (that path is excluded to prevent the automated Helm tag commit from re-triggering the pipeline in a loop). To kick off a new run without a real code change, the easiest approach is to make a small, harmless edit to `README.md` and commit it directly to `main`.

1. In the repository, open `README.md` and make a minor edit (e.g., add a blank line or update a description).
2. Commit the change directly to `main`.

This fires the **Build and Release** workflow.

### Watch the Workflow Progress

1. Go to the **Actions** tab. You should see the new run queued or already running.

   The run progresses through three stages visible in the workflow graph:

   - **`build-and-push`** — builds the three container images into ACR (~2 minutes)
   - **`release-and-test-staging`** — updates staging Helm values, waits for ArgoCD to sync, then runs the ZAP scan (~3–20 minutes depending on polling)
   - **`release-production`** — waits for manual approval before running

   > ⚠️ **Important:** The `release-and-test-staging` job polls every 60 seconds up to 20 times while waiting for ArgoCD to sync the new image tag. The Actions tab will show it as "In progress" for several minutes — this is normal. Do not cancel the run or assume it has failed.

   **Example output — workflow in progress (build complete, staging deploying):**
   ![[Pasted image 20260609123343.png]]
   

2. Once `release-and-test-staging` completes, go to the **Issues** tab.

   The ZAP baseline scan automatically files any security findings it detects against the staging environment as GitHub Issues. Review each one — in a real environment these would be triaged and assigned. For this lab, reviewing them is sufficient before proceeding to the production approval.
   
   ![[Pasted image 20260609152202.png]]
   
   ![[Pasted image 20260609152255.png]]

3. Return to the **Actions** tab and select the latest workflow run. You will see a yellow banner:

   ```
   hectorproko requested your review to deploy to production
   ```

   The `release-production` job shows status **"production waiting for review"**.
   
   ![[Pasted image 20260609123933.png]]
### Approve the Production Deployment

4. Click **Review deployments**.
5. In the dialog, check the box next to **production**.
6. Leave an optional comment — for example: `We are good to go`.
7. Click **Approve and deploy**.

   **Example — Review deployments dialog:**

  ![[Pasted image 20260609124032.png]]
The `release-production` job is now unblocked and begins running.

![[Pasted image 20260609125335.png]]


%%[[redoing it reusing repo]] has some troubleshooting done%%

---

## Key Concepts Reinforced

**GitOps pull model** — ArgoCD pulls desired state from Git rather than having pipelines push directly to Kubernetes. The cluster is always a reflection of the repository.

**Workload Identity Federation** — GitHub Actions authenticates to Azure using a short-lived OIDC token tied to the specific environment or branch. No service principal secrets are stored anywhere.

**Environment gates** — The `production` GitHub Environment requires a named reviewer to approve before the deployment job runs, providing a human checkpoint between staging and production.

**Automated security testing in CI** — The ZAP baseline scan runs automatically after every staging deployment. Security findings surface as GitHub Issues, creating a traceable record tied directly to the code change that introduced them.

**Helm as a release mechanism** — Rather than managing raw Kubernetes manifests, Helm charts allow values (like image tags) to be environment-specific while keeping the chart template shared. ArgoCD uses these value files as the source of truth for each environment's configuration.

---