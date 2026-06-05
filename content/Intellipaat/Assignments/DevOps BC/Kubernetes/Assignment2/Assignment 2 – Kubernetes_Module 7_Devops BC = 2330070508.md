---
tags:
  - Kubernetes
---


> [!info] Module 7: Kubernetes Assignment - 2
> **Tasks To Be Performed:** 
> 1. Use the previous deployment 
> 2. Create a service of type NodePort for NGINX deployment 
> 3. Check the NodePort service on a browser to verify


### Step 1:
Previous deployment [[Assignment 1 – Kubernetes_Module 7_Devops BC = 2330070508|Assignment 1 – Kubernetes]]
> [!tip] Instances and Their Private IP Addresses
> k-master: `10.0.1.82`
> worker1: `10.0.1.57`
> worker2: `10.0.1.119`
> 
### Step 2: Create NodePort Service
To allow external traffic to reach my Nginx deployment, I will create a NodePort service. This is done by defining a service configuration in a YAML I named `nginx-nodeport.yaml`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-deployment
spec:
  type: NodePort
  ports:
    - targetPort: 80
      port: 80
      nodePort: 30008
  selector:
    app: nginx
```
> The `type: NodePort` specification in the service configuration ensures that the service is accessible through a specific port allocated on each node in the cluster. By applying the `nginx-nodeport.yaml` file, the service will be configured to allow access to the Nginx server via the allocated `nodePort` of 30008 on all the nodes.
> 


> [!tip]
> The `selector` field in this service configuration is set to `app: nginx`. This matches the label of the pods created by the [[Assignment 1 – Kubernetes_Module 7_Devops BC = 2330070508#^5f259a|nginx-deployment.yaml (Assignment 1)]] . It is important to note that this service will route traffic specifically to the [[Pasted image 20231113173344.png|pods]] created by the `nginx-deployment.yaml` file and not to those created by the one-line `kubectl create deployment` command, unless they were also labeled with `app: nginx`.
>
> ```
> spec:
>   replicas: 3
>   selector:
>     matchLabels:
>       app: nginx
> ```



Once I've created this file, I apply it using the command:
```bash
kubectl apply -f nginx-nodeport.yaml
```

I check to ensure the service is running with:
```bash
kubectl get svc
```

<br>![[nodePort service created.png]]
> We've confirmed that the NodePort service is actively routing traffic. The defined NodePort of `30008` is set up to forward to the internal port `80` of the Nginx pods


### Step 3: Verification

I retrieve the public IP address of one of the worker nodes in the cluster to test the Nginx service externally. By entering `http://<Worker_Node_Public_IP>:30008` into a web browser, I can access the Nginx landing page served through the NodePort, verifying that the service is correctly exposed.

> [!success]
> <br>![[Pasted image 20231113182315.png]]