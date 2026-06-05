---
tags:
  - Kubernetes
---


> [!info] Module 7: Kubernetes Assignment - 3
> **Tasks To Be Performed:** 
> 1. Use the previous deployment 
> 2. Change the replicas to 5 for the deployment

%%Referencing [[zoom video#Assignment 3]]%%

### Step 1:
Continuing from [[Assignment 2 – Kubernetes_Module 7_Devops BC = 2330070508|Assignment 2 – Kubernetes]]

> [!tip] Instances and Their Private IP Addresses
> k-master: `10.0.1.82`
> worker1: `10.0.1.57`
> worker2: `10.0.1.119`
> 

### Step 2: Change Replicas
At the moment our deployments have 3 replicas each. 
<br>![[Pasted image 20231113173555.png]]
   
I run the command `kubectl edit deploy` which opens the deployment configuration in a command-line editor. Within this editor, which operates similarly to Vim, I modify the number of `replicas` to scale the deployment.
<br>![[Pasted image 20231113190534.png]]
<br>![[Pasted image 20231113190649.png|260]]
After editing, I save the changes using the editor's save and exit commands `:wq`

After saving the updated deployment configuration, I confirm the changes by checking the status of deployments 

> [!success]
> <br>![[Pasted image 20231113191119.png]]