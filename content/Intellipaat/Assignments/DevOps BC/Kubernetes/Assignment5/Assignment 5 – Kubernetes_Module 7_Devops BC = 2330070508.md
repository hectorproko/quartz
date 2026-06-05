---
tags:
  - Kubernetes
---


> [!info] Module 7: Kubernetes Assignment - 5
> **Tasks To Be Performed:** 
> 1. Use the previous deployment 
> 2. Deploy an NGINX deployment of 3 replicas 
> 3. Create an NGINX service of type ClusterIP 
> 4. Create an ingress service/ Apache to Apache service/ NGINX to NGINX service

%%Referencing [[Devops BC = 2330070508#Lecture 8 – Creating An Ingress]]

==Not used==
> [!tip] Ingress:
> Ingress install from [[Hands-On-–-Kubernetes-Installation.pdf]]
> ```bash
> kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml
> ```

%%


### Step 1:
Continuing from [[Assignment 4 – Kubernetes_Module 7_Devops BC = 2330070508|Assignment 4]]
> [!tip] Instances and Their Private IP Addresses
> k-master: `10.0.1.82`
> worker1: `10.0.1.57`
> worker2: `10.0.1.119`
> 

### Step 2:
I use the deployment from [[Assignment 1 – Kubernetes_Module 7_Devops BC = 2330070508#Step 2 NGINX Deployment|Assignment 1 Step 2:  NGINX Deployment]]

### Step 3:
I'll use the `ClusterIP` service we converted from `NodePort` in [[Assignment 4 – Kubernetes_Module 7_Devops BC = 2330070508#Step 2 Changing Service|Assignment 4 Step 2: Changing Service]]

### Step 4: Installing Ingress Controller

Next, I install the Nginx Ingress Controller which allows us to manage Ingress resources and control external traffic into the cluster. Ingress resources are used to configure rules for routing incoming external requests to specific services or pods within the cluster. To do this, I use the official deployment YAML file provided by the project. *GitHub: [Bare metal clusters](https://github.com/kubernetes/ingress-nginx/blob/main/docs/deploy/index.md#bare-metal-clusters)*

This command fetches the deployment YAML file from the official GitHub repository and applies it to the cluster.
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/baremetal/deploy.yaml
```

<br>![[Pasted image 20231113204441.png]]

We can confirm that the Nginx Ingress Controller is successfully running within our Kubernetes cluster as a pod
<br>![[Pasted image 20231113204642.png]]

We have an ingress service configured with NodePort types on the following ports:
- HTTP: 80, mapped to 30426
- HTTPS: 443, mapped to 31013"
<br>![[nodeport ports.png]]
%%
The Public IP that worked was worker2
Need to find 
<br>![[Pasted image 20231113210305.png]]
<br>![[Pasted image 20231113210403.png]]
Seem like this works with the Public IP of the node that runs at least one pod
kubectl get pods -o wide
#question
> [!NOTE] The above theory seem incorrect according to ChatGPT
> With a NodePort service in Kubernetes, you can access the service using the public IP address of any node in the cluster along with the NodePort, regardless of which node the pod is actually running on.
> 
> Enter the public IP address of either worker node followed by the NodePort. For example: `http://<Public_IPv4_of_Node>:30080`.

%%
%%
> [!question]- Issue: curl ClusterIP
> > [!fail]
> > To curl `ClusterIP`'s IP
> 
> > [!done] Solution
> > **Security Group** in EC2 was set to `All TCP` `0-65535` from `0.0.0.0` . Changed it to `All traffic`

%%

I create an Ingress rule by crafting an `ingress.yaml` file with the following content:
Followed [Kubernetes Doc](https://kubernetes.io/docs/concepts/services-networking/ingress/#the-ingress-resource)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /nginx
        pathType: Prefix
        backend:
          service:
            name: my-nginx-deployment
            port:
              number: 80
```

After creating the `ingress.yaml` file, I apply it to my cluster:
```yaml
kubectl apply -f ingress.yaml
```

%%
> [!question]- **Issue:** ingress path NOT working `<IP>/nginx`
> Issue was found looking at logs of the ingress-controller-pod
> Finding pod: `kubectl get pods --all-namespaces`
> Once identified: `kubectl logs ingress-nginx-controller-6f6c84b8-qftv2 -n ingress-nginx`, where `-n` is for namespace
> 
> > [!fail]
> > The log returned error related it to <mark style="background: #FFB8EBA6;">ingress class</mark>
> > ```
> > 7 store.go:431] "Ignoring ingress because of error while validating ingress class" ingress="default/minimal-ingress" error="no object matching key \"nginx-example\" in local store"
> > ```
> > The `ingressClassName:` portion of [[Assignment 5 – Kubernetes_Module 7_Devops BC = 2330070508#^38f1e4|the .yaml]]
> > Before I had a random name from sample file that did not exist later I removed it thinking it will default to something that work (it did not)
> 
> > [!done] Solution
> > Used `kubectl get ingressclass` to get list of `ingressclass`, i had only 1 `nginx`.
> > Updated the ingress.yaml
> > ```yaml
> > ingressClassName: nginx
> > ```
> > Log when working
> > <br>![[success after fixing ingress rule, pod log ingress controller.png]]
> 

When the ingress wasn't working ADDRESS was empty
<br>![[Pasted image 20231114093204.png]]
%%

I verify the ingress was created:

> [!success]
> <br>![[Pasted image 20231114112947.png]]


I test the ingress path `/nginx` using both the public IP addresses of worker nodes 1 and 2, along with the NodePort `30426`.
<br>![[Pasted image 20231114113302.png|500]]
<br>![[Pasted image 20231114113359.png|500]]

I will now create an Apache deployment along with a corresponding ClusterIP service to further test our Ingress.
<br>![[Pasted image 20231114115621.png]]
<br>![[Pasted image 20231114114823.png]]

I have updated `ingress.yaml` to include an additional path `/apache` with the following configuration.
```yaml
      - path: /apache
        pathType: Prefix
        backend:
          service:
            name: my-apache-deployment
            port:
              number: 80
```

I will now apply it again:
```
kubectl apply -f ingress.yaml
```
<br>![[Pasted image 20231114120015.png]]



We can also try with the NodePort for https [[nodeport ports.png|as per]]
80:30426
443:31013

Now, I will test both paths with both NodePorts mapped to port 80 for HTTP and port 443 for HTTPS.
> [!success] 
> Using 80(HTTP) : NodePort `30426`
> <br>![[testingNginxApacheIngressBrowser.gif]]

> [!success] 
> Using 443(HTTPS) : NodePort `31013`
> <br>![[testingNginxApacheIngressBrowserHTTPS.gif]]

> [!attention]
> The ingress rule is configured with `pathType: Prefix`. This means that, for example, both `/nginx` and `/nginx123` will match the rule, but `/123nginx` will not be a match.

%%
> [!NOTE]- Notes: Incase needed
> ```yaml
> ubuntu@ip-10-0-1-82:~$ cat apache-clusterip.yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: my-apache-deployment
> spec:
>   type: ClusterIP
>   ports:
>     - targetPort: 80
>       port: 80
>   selector:
>     app: apache
> ubuntu@ip-10-0-1-82:~$
> ```
> 
> ---
> 
> ```
> ubuntu@ip-10-0-1-82:~$ cat apache-deployment.yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: apache-deployment
>   labels:
>     app: apache
> spec:
>   replicas: 3
>   selector:
>     matchLabels:
>       app: apache
>   template:
>     metadata:
>       labels:
>         app: apache
>     spec:
>       containers:
>       - name: apache
>         image: httpd
>         ports:
>         - containerPort: 80
> ubuntu@ip-10-0-1-82:~$ 
> 
> ```
> ---
> 
> ```
> ubuntu@ip-10-0-1-82:~$ cat ingress.yaml
> apiVersion: networking.k8s.io/v1
> kind: Ingress
> metadata:
>   name: minimal-ingress
>   annotations:
>     nginx.ingress.kubernetes.io/rewrite-target: /
> spec:
>   ingressClassName: nginx
>   rules:
>   - http:
>       paths:
>       - path: /nginx
>         pathType: Prefix
>         backend:
>           service:
>             name: my-nginx-deployment
>             port:
>               number: 80
>       - path: /apache
>         pathType: Prefix
>         backend:
>           service:
>             name: my-apache-deployment
>             port:
>               number: 80
> ubuntu@ip-10-0-1-82:~$
> ```

%%