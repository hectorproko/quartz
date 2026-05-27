---
title: Persisting data in Kubernetes
tags:
  - Kubernetes
  - EKS
  - AWS
  - EBS
  - ConfigMaps
  - "#PersistentVolumes"
hardlinked: "True"
quartz: "True"
linkedin: "True"
---
*~~(original [Project 23](https://github.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/blob/main/Project23_Steps.md))~~*

## Overview

One of the fundamental challenges in containerized environments is data persistence. By default, any data written inside a container is ephemeral, it disappears the moment the pod is terminated or restarted. In this project, I explore multiple strategies for persisting data in a Kubernetes cluster running on AWS EKS, progressing from a raw EBS volume attachment all the way to dynamic provisioning with PersistentVolumeClaims and configuration management with ConfigMaps.

---

## Prerequisites

This project builds on top of an existing EKS cluster. The cluster setup follows the same steps from [[Project 22 Deploying applications into a Kubernetes cluster|Deploying Applications into a Kubernetes Cluster]]. To confirm the cluster is up and reachable: 

```bash
kubectl cluster-info
```

**Output:**

```
Kubernetes control plane is running at https://D236EDB340398719073CDF558090376C.gr7.us-east-1.eks.amazonaws.com
CoreDNS is running at https://D236EDB340398719073CDF558090376C.gr7.us-east-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

---

## Part 1 - Deploying Nginx Without a Volume (Baseline)

Before introducing any persistence layer, I deploy a basic Nginx deployment to understand the default behavior and inspect the configuration that will be referenced later.

```bash
kubectl apply -f nginx-pod.yaml
```

`nginx-pod.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Verifying the pods are running:

```bash
kubectl get pods
```

**Output:**

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6fdcffd8fc-j2jtt   1/1     Running   0          22s
nginx-deployment-6fdcffd8fc-l75gk   1/1     Running   0          22s
nginx-deployment-6fdcffd8fc-zxk9p   1/1     Running   0          22s
```

### Inspecting the Nginx Configuration

I exec into one of the pods to read the default Nginx config and identify the **document root**, the directory Nginx uses to serve content. This path will be used later as the mount point for our volume.

```bash
kubectl exec -it nginx-deployment-6fdcffd8fc-j2jtt -- bash
cat /etc/nginx/conf.d/default.conf
```

**Output (relevant section):**

```nginx
location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
}
```

The document root is `/usr/share/nginx/html`. Any volume we mount there will provide persistence for the web content served by Nginx.

---

## Part 2 - Attaching an EBS Volume Directly

### Finding the Worker Node's Availability Zone

Since an EBS volume can only be mounted by an EC2 instance in the **same Availability Zone**, the first step is to find which AZ the pod's node resides in.

```bash
kubectl get pods -o wide
```

**Output:**

```
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE
nginx-deployment-6fdcffd8fc-j2jtt   1/1     Running   0          18m   192.168.154.43   ip-192-168-138-68.ec2.internal
nginx-deployment-6fdcffd8fc-l75gk   1/1     Running   0          18m   192.168.246.1    ip-192-168-229-214.ec2.internal
nginx-deployment-6fdcffd8fc-zxk9p   1/1     Running   0          18m   192.168.190.59   ip-192-168-138-68.ec2.internal
```

Using `kubectl describe node` on the nodes revealed that:

- `ip-192-168-138-68.ec2.internal` is in `us-east-1a`
- `ip-192-168-229-214.ec2.internal` is in `us-east-1b`

I chose `us-east-1a` for creating the EBS volume.

### Creating the EBS Volume in AWS

In the AWS Console, I navigated to **EC2 → Elastic Block Store → Volumes → Create Volume**, making sure to select the `us-east-1a` Availability Zone to match the node.

![Example Image](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/createvolume.png)
### Updating the Deployment to Use the EBS Volume

With the EBS volume created, I updated the deployment manifest to reference it using the `awsElasticBlockStore` volume type. The `volumeID` is the ID of the EBS volume just created.

```yaml
# nginx-pod.yaml (updated)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-volume
          mountPath: /usr/share/nginx/
      volumes:
      - name: nginx-volume
        # This AWS EBS volume must already exist.
        awsElasticBlockStore:
          volumeID: "vol-07b537651bbe68be0"
          fsType: ext4
```

The `volumeMounts` section answers the question: _where inside the container should this volume be mounted?_ Here it is mounted at `/usr/share/nginx/`, which means all data written to that path will be stored on the EBS volume and survive pod restarts.

Applying the updated manifest:

```bash
kubectl apply -f nginx-pod.yaml
```

**Output:**

```
deployment.apps/nginx-deployment configured
```

Confirming the pod is running:

```bash
kubectl get pods
```

**Output:**

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5844d76665-gq748   1/1     Running   0          7s
```

> **Note:** EBS only supports a single EC2 instance mounting a volume at a time (ReadWriteOnce). This approach also requires manually creating and managing the EBS volume, which does not scale well. The next section addresses this with dynamic provisioning.

---

## Part 3 - Managing Volumes Dynamically with PVs and PVCs

Manually creating EBS volumes and hard-coding their IDs is not ideal for production. Kubernetes provides **PersistentVolumes (PVs)** and **PersistentVolumeClaims (PVCs)** to abstract storage management and support dynamic provisioning.

- A **PersistentVolume (PV)** is a piece of storage provisioned in the cluster.
- A **PersistentVolumeClaim (PVC)** is a request for storage made by a pod. It binds to an available PV that satisfies its requirements.
- A **StorageClass** defines the type of storage to use and enables dynamic provisioning, so PVs are created automatically when a PVC requests them.

In EKS, the default StorageClass is `gp2`, backed by AWS EBS:

```bash
kubectl get storageclass
```
<!--
The `gp2` StorageClass was **automatically created by EKS when the cluster was provisioned** — you didn't create it manually at any point in this lab. It's part of the default EKS installation.
-->
**Output:**

```
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  21m
```

### Creating the PersistentVolumeClaim

`nginx-volume-claim.yml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-volume-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: gp2
```

```bash
kubectl apply -f nginx-volume-claim.yml
```

**Output:**

```
persistentvolumeclaim/nginx-volume-claim created
```

Checking the PVC status:

```bash
kubectl get pvc
```

**Output:**

```
NAME                 STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nginx-volume-claim   Pending                                      gp2            23s
```

The `Pending` status is expected at this stage, the `gp2` StorageClass uses `WaitForFirstConsumer` binding mode, meaning it will not provision the PV until a pod actually requests it.

### Troubleshooting: Pods Stuck in Pending

After deploying the Nginx deployment that references this PVC, the pods remained in `Pending` state. I used the following steps to diagnose the issue.

**Step 1 - Check the Deployment:**

```bash
kubectl get deployments
```

**Output:**

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   0/1     1            0           6m28s
```

**Step 2 - Check the Pod status:**

```bash
kubectl get pods
```

**Output:**

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7676f8b4d9-m5v2t   0/1     Pending   0          4m58s
```

**Step 3 - Describe the Pod to find the root cause:**

```bash
kubectl describe pod nginx-deployment-7676f8b4d9-m5v2t
```

**Output (Events section):**

```
❌ Warning  FailedScheduling  55s (x5 over 5m23s)  default-scheduler  no nodes available to schedule pods
```

**Root Cause:** There were no worker nodes available in the cluster to schedule the pod. I provisioned the required additional nodes, after which the deployment became healthy.

```bash
kubectl get deployments
```

**Output:**

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           29m
```

### Verifying PV Creation

Once a pod was scheduled and used the PVC, the `gp2` StorageClass dynamically provisioned a PV:

```bash
kubectl get pv
```

**Output:**

```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   REASON   AGE
pvc-aa96611c-aba1-42c4-b079-243af9ae7212   2Gi        RWO            Delete           Bound    default/nginx-volume-claim   gp2                     4m54s
```

The PV is now `Bound` to our `nginx-volume-claim` PVC. The corresponding EBS volume is also visible in the AWS Console:

![Example Image](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/createvolume2.png)
### Deploying Nginx with the PVC

I updated the deployment manifest to mount the PVC:

```yaml
# nginxdeployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-volume-claim
          mountPath: "/tmp/dare"
      volumes:
      - name: nginx-volume-claim
        persistentVolumeClaim:
          claimName: nginx-volume-claim
```

The PVC `nginx-volume-claim` is mounted inside the container at `/tmp/dare`. Any data written to that path will be stored on the EBS volume and persisted across pod restarts.

```bash
kubectl apply -f nginxdeployment.yml
```

### Exposing Nginx with a LoadBalancer Service

Using the same service definition from [[Project 22 Deploying applications into a Kubernetes cluster|Deploying applications into a Kubernetes cluster]]: 

```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    tier: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

```bash
kubectl apply -f nginx-service.yaml
```

Checking the service:

```bash
kubectl get svc
```

**Output:**

```
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                               PORT(S)        AGE
kubernetes      ClusterIP      10.100.0.1      <none>                                                                    443/TCP        4h47m
nginx-service   LoadBalancer   10.100.54.132   a69047daa62674e7392375d178cb0a9b-1787659259.us-east-1.elb.amazonaws.com   80:31491/TCP   14s
```

Accessing the service via the external load balancer URL confirms Nginx is serving traffic:

```bash
lynx a69047daa62674e7392375d178cb0a9b-1787659259.us-east-1.elb.amazonaws.com
```

![Example Image](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/welcomenginx.png)

---

## Part 4 - Using ConfigMaps for Configuration Persistence

While PVs and PVCs handle persistent data storage, **ConfigMaps** serve a different purpose: keeping configuration files, such as the Nginx `index.html`, from being lost when a pod is replaced. ConfigMaps are not suitable for large data storage, but they are ideal for managing configuration that needs to survive pod restarts.

### Step 1 - Strip the PVC from the Deployment

To start fresh for this demonstration, I removed the `volumeMounts` and `volumes` sections from the deployment:

```bash
kubectl apply -f nginxdeployment.yml
```

**Output:**

```
deployment.apps/nginx-deployment configured
```

Getting the service's external IP to confirm Nginx is accessible:

```bash
kubectl get service
```

**Output:**

```
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)        AGE
kubernetes      ClusterIP      10.100.0.1       <none>                                                                   443/TCP        5h31m
nginx-service   LoadBalancer   10.100.182.155   aab8c1f0d166c4dfba6efab2c8126f6f-785035398.us-east-1.elb.amazonaws.com   80:30460/TCP   8m50s
```

![Example Image](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/welcomenginx3.png)
<!--
The actual point of showing it was just to confirm that Nginx still works normally (via the LoadBalancer) _before_ introducing the ConfigMap
-->
### Step 2 - Copy the Default index.html

I exec into the running pod and read the default `index.html` content to use it as the base for the **ConfigMap**:

```bash
kubectl exec -it nginx-deployment-5b98d885c7-xdn7h -- bash
cat /usr/share/nginx/html/index.html
```

**Output:**

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and working.</p>
...
</body>
</html>
```

### Step 3 - Create the ConfigMap Manifest

I embed the HTML content directly into a ConfigMap manifest. The `index-file` key holds the full HTML that will be injected as a file into the pod's filesystem.

`nginx-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: website-index-file
data:
  # file to be mounted inside a volume
  index-file: |
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    ...
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    ...
    </body>
    </html>
```

```bash
kubectl apply -f nginx-configmap.yaml
```

**Output:**

```
configmap/website-index-file created
```

### Step 4 - Mount the ConfigMap into the Deployment

I update the deployment to mount the ConfigMap as a volume at `/usr/share/nginx/html`, replacing the ephemeral directory with one backed by the ConfigMap:

`nginx-pod-with-cm.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
          - name: config
            mountPath: /usr/share/nginx/html
            readOnly: true
      volumes:
      - name: config
        configMap:
          name: website-index-file
          items:
          - key: index-file
            path: index.html
```

```bash
kubectl apply -f nginx-pod-with-cm.yaml
```

Verifying the pod is running:

```bash
kubectl get pods
```

**Output:**

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7dcdfbd66f-m527b   1/1     Running   0          2m3s
```

Exec into the pod to confirm the index.html is now managed by the ConfigMap:

```bash
kubectl exec -it nginx-deployment-7dcdfbd66f-m527b -- bash
ls -ltr /usr/share/nginx/html
```

**Output:**

```
total 0
lrwxrwxrwx 1 root root 17 Aug 11 21:29 index.html -> ..data/index.html
```

The `index.html` is now a **symlink** pointing to the ConfigMap's data directory. It is no longer ephemeral.

### Step 5 - Live-Update the ConfigMap

One of the key benefits of ConfigMaps is that changes propagate to running pods **without requiring a restart**. I edit the ConfigMap directly to update the heading:

```bash
kubectl edit cm website-index-file
```

**Output:**

```
configmap/website-index-file edited
```

I updated the `<h1>` tag to read **"USING ConfigMAP!"**.

![Example Image](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/configmap.png)

Checking the service to get the load balancer URL:

```bash
kubectl get service
```

**Output:**

```
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)        AGE
kubernetes      ClusterIP      10.100.0.1       <none>                                                                   443/TCP        5h54m
nginx-service   LoadBalancer   10.100.182.155   aab8c1f0d166c4dfba6efab2c8126f6f-785035398.us-east-1.elb.amazonaws.com   80:30460/TCP   32m
```

Without restarting the pod, the page now reflects the updated ConfigMap content:

![Example Image](https://raw.githubusercontent.com/hectorproko/PERSISTING-DATA-IN-KUBERNETES/main/images/usingconfigmap.png)
*Notice the USING ConfigMap!*

> **Key Takeaway:** Updating a ConfigMap's data will propagate to all pods consuming it automatically, whether those pods are restarted or not.

---

## Summary

|Approach|Use Case|Persistence|Dynamic Provisioning|
|---|---|---|---|
|**awsElasticBlockStore** (direct)|Simple single-pod stateful workloads|✅ Yes|❌ No - manual EBS creation required|
|**PersistentVolumeClaim (PVC)**|Scalable stateful applications|✅ Yes|✅ Yes - EBS provisioned automatically via `gp2` StorageClass|
|**ConfigMap**|Configuration files (not data storage)|✅ Yes (survives pod restarts)|N/A - stored in etcd|

Each approach has its place. Direct EBS attachment is simple but brittle. PVCs with a StorageClass are the production-grade solution for stateful data. ConfigMaps solve the narrower problem of keeping configuration files consistent and updatable without rebuilding or restarting pods.

<!--
### Post

Just completed refactoring and cleaning up a previous **Persisting Data in Kubernetes** on EKS documentation.

This project tackles one of the most important challenges in Kubernetes: data persistence. 

I also explore mounting volumes, common troubleshooting (like pods stuck in Pending), and using **ConfigMaps** to persist and dynamically update configuration without rebuilding images.

[https://hectorproko.github.io/quartz/darey.io/Kubernetes/Project23/Project23-Persisting-data-in-Kubernetes](https://hectorproko.github.io/quartz/darey.io/Kubernetes/Project23/Project23-Persisting-data-in-Kubernetes)


#Kubernetes #EKS #AWS #DevOps #CloudNative #PersistentVolumes #Storage #Containerization
-->
