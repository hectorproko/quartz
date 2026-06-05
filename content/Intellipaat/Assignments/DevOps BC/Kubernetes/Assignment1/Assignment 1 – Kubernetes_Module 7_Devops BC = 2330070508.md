---
tags:
  - Kubernetes
---



> [!info] Module 7: Kubernetes Assignment - 1
> **Tasks To Be Performed:** 
> 1. Deploy a Kubernetes cluster for 3 nodes 
> 2. Create a NGINX deployment of 3 replicas


> [!tip] Instances and Their Private IP Addresses
> k-master: `10.0.1.82`
> worker1: `10.0.1.57`
> worker2: `10.0.1.119`
> 


### Step 1: Deploying Kubernetes
 <br>![[Installing Kubernetes (Kubeadm)]]

---
### Step 2:  NGINX Deployment

I will create a YAML file named `nginx-deployment.yaml` to define my Nginx deployment in Kubernetes.
> [Kubernetes Docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

^5f259a

I initiate the deployment by applying the file to my Kubernetes cluster with the following command:
```bash
kubectl apply -f nginx-deployment.yaml
```

To verify that the deployment was successful and to view the deployed resources, I use the command:
```bash
kubectl get deploy
```

We can also see the number of pods running is 3
```bash
kubectl get pods
```

> [!success]
> <br>![[Pasted image 20231113173344.png]]

Alternatively, instead of using a `.yaml` file, I can quickly create a deployment directly from the command line. The following one-liner command sets up a deployment named `nginx-deployment-1` using the `nginx` image and specifies that there should be three replicas:
```bash
kubectl create deployment nginx-deployment-1 --image=nginx --replicas=3
```
<br>![[Pasted image 20231113173555.png]]