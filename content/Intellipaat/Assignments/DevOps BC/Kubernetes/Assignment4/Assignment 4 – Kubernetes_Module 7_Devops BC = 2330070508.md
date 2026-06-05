---
tags:
  - Kubernetes
---


> [!info] Module 7: Kubernetes Assignment - 4
> **Tasks To Be Performed:** 
> 1. Use the previous deployment 
> 2. Change the service type to ClusterIP

%%Referencing [[zoom video#assignment 4]]%%

### Step 1:
Continuing from [[Assignment 3 – Kubernetes_Module 7_Devops BC = 2330070508|Assignment 3]]
> [!tip] nstances and Their Private IP Addresses
> k-master: `10.0.1.82`
> worker1: `10.0.1.57`
> worker2: `10.0.1.119`
> 

### Step 2: Changing Service
Currently, I have a NodePort service set up for my deployment. I'll modify this service to be a ClusterIP type, which is only accessible within the Kubernetes cluster.
<br>![[nodePort service created.png]]
   
I use the `kubectl edit` command to modify the service configuration directly:
```bash
kubectl edit service
```

This command opens the service's configuration in an editor. Within the editor, I perform the following changes:
1. Change the `type` field from `NodePort` to `ClusterIP`.
2. Set the `nodePort` field to `null` or remove it entirely, as ClusterIP services do not require an external port.

Before:
<br>![[Pasted image 20231113192859.png]]
After:
<br>![[Pasted image 20231113192816.png|222]]

I verify the change by inspecting the list of services with the command `kubectl get svc`. In the output, I confirm that my service, `my-nginx-service`, now shows `ClusterIP` in the `TYPE` column. %%[[nodePort service created.png|previous]]%%

> [!success]
> <br>![[Pasted image 20231113193027.png]]

To quickly test the ClusterIP service, I use the `curl` command to send a request to the service's IP address from within the cluster (master node) :

> [!done] Successfully returns the webpage
> <br>![[Pasted image 20231114091431.png]]
